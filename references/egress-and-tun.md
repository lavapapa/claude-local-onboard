# 固定出口、Mihomo TUN 与直连白名单

## 1. 客户端选型（Apple Silicon 与 Intel 均适用）

本流程不绑定单一代理客户端，只要底层支持 **Mihomo（Clash Meta）规则语义**、能提供稳定的本地 HTTP(S) 混合监听端口与固定策略组即可：

- **图形客户端（推荐大多数用户）**：FlClash、Clash Verge Rev、Mihomo Party、Stash 等。优点是自带订阅拉取、持久覆写脚本/配置（Override/Merge）以及 macOS 系统扩展（System Extension）或特权 Helper 管理 TUN，适合日常查看实时连接与规则命中。
- **裸 `mihomo` 命令行核心**：适合熟悉终端运维的用户。注意裸核心不包含 macOS Network Extension 图形授权界面，开启 TUN 需要通过 `sudo` 或给二进制赋予 `root:wheel` + `chmod +s` 创建 `utun` 接口，订阅需通过配置文件中的 `proxy-providers` 管理。

> 如果执行配置的 Agent 本身正通过当前代理联网，修改 TUN、DNS 或端口可能导致当前回合短暂断连。动手前必须先备份原客户端配置，并把一行手动恢复命令展示给用户。

## 2. 路由与端口的核心设计要求

1. **统一本地代理入口端口**：系统代理（若开启）、`sandbox-exec` 启动器环境变量和指纹浏览器统一指向同一个本地混合端口（常见默认如 `127.0.0.1:7890`；若现有客户端使用其他端口，先核实并统一，避免多端口分叉）。
2. **锁定单一固定出口节点**：建立名为 `FIXED-EXIT`（或同类清晰名称）的 `select` 策略组，手动选定一个经过延迟、丢包和出口地区核验的稳定节点。**不要对 Claude 流量使用 `url-test` / `fallback` 自动跨区切换，更不允许节点故障时回退到 `DIRECT`。**
3. **控制接口安全**：`external-controller` 仅监听 `127.0.0.1:<ctrl-port>`（如 `127.0.0.1:9090`），并配置强随机 `secret`，防止本机其他网页或未授权进程无认证篡改节点；随后在 `sandbox-exec` 层还会对该端口做二次封禁。
4. **进程识别与显式直连白名单**：开启 `find-process-mode: always`。需要低延迟直连的日常工具（局域网、内网/校园网域名、SSH、Tailscale/ZeroTier 等 Mesh VPN、UU 远程/RustDesk 等远程桌面）显式写为 `DIRECT`，其余公网流量统一走 `FIXED-EXIT`。
   - **明确认知**：被列入 `DIRECT` 白名单的应用（如远程桌面）在对外通信时，远端服务器看到的是本机真实公网 IP，这是保障低延迟的设计结果，不会把流量混入受 `sandbox-exec` 约束的 Claude 进程。
5. **DNS 与 IPv6 一致性**：
   - 开启 TUN 的 `dns-hijack` 与 Mihomo 内置 `dns`，远端 DoH/DoT 解析强制走 `FIXED-EXIT` 代理链路，避免向本地运营商 DNS 泄露目标域名查询；
   - 若订阅节点不支持可靠的代理 IPv6，在客户端关闭公网 IPv6 出口或确保沙盒内直连 IPv6 被拒绝，防止双栈环境下 IPv4 走代理、IPv6 绕回本机直连。

## 3. 最小参考配置 / 持久覆写骨架

不要直接手改会被订阅刷新覆盖的生成文件；应将以下关键设置放入客户端的**持久覆写（Merge / Script / Override）层**：

```yaml
mixed-port: 7890
allow-lan: false
bind-address: 127.0.0.1
mode: rule
log-level: info
ipv6: false
find-process-mode: always

external-controller: 127.0.0.1:9090
secret: "<生成一段随机强口令>"

tun:
  enable: true
  stack: mixed
  auto-route: true
  auto-detect-interface: true
  strict-route: true
  dns-hijack:
    - any:53
    - tcp://any:53

dns:
  enable: true
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  default-nameserver:
    - 223.5.5.5
  nameserver:
    - https://cloudflare-dns.com/dns-query#FIXED-EXIT
    - https://dns.google/dns-query#FIXED-EXIT
  proxy-server-nameserver:
    - 223.5.5.5
    - 119.29.29.29

proxy-groups:
  - name: FIXED-EXIT
    type: select
    proxies:
      - "<选定的单一稳定出口节点>"

rules:
  # 1. 本机与局域网、Mesh VPN 保留地址
  - IP-CIDR,127.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,10.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,172.16.0.0/12,DIRECT,no-resolve
  - IP-CIDR,192.168.0.0/16,DIRECT,no-resolve
  - IP-CIDR,100.64.0.0/10,DIRECT,no-resolve

  # 2. 按需保留的远程桌面 / Mesh VPN 进程直连示例（按本机实际安装路径增删）
  # - PROCESS-PATH-REGEX,^/Applications/Tailscale\.app/.*,DIRECT
  # - PROCESS-PATH-REGEX,^/Applications/UURemote\.app/.*,DIRECT

  # 3. Claude 与关联域名显式锁定固定出口
  - DOMAIN-SUFFIX,anthropic.com,FIXED-EXIT
  - DOMAIN-SUFFIX,claude.ai,FIXED-EXIT
  - DOMAIN-SUFFIX,claude.com,FIXED-EXIT

  # 4. 用户确认需直连的特定内网或国内域名/GeoSite（按需添加）
  # - DOMAIN-SUFFIX,example.edu.cn,DIRECT
  # - GEOSITE,cn,DIRECT
  # - GEOIP,CN,DIRECT,no-resolve

  # 5. 兜底全部走固定出口，禁止兜底 DIRECT
  - MATCH,FIXED-EXIT
```

> 若用户选择**模式 B（不开整机 TUN，仅本地代理端口 + `sandbox-exec`）**，将上方 `tun.enable` 设为 `false`，保留 `mixed-port: 7890`、`external-controller`、`dns` 与 `FIXED-EXIT` 规则即可。

## 4. 节点筛选与连通性核验

导入订阅后，通过客户端面板或带认证的本地控制 API 测试候选节点，不引入来历不明的第三方测速脚本：

```bash
# 通过本地代理端口观测实际公网出口与延迟（7890 替换为实际端口）
curl -fsS -x http://127.0.0.1:7890 --max-time 10 https://www.cloudflare.com/cdn-cgi/trace | grep -E "^(ip|loc|warp)="
curl -fsS -x http://127.0.0.1:7890 -o /dev/null -w "HTTP %{http_code} | connect=%{time_connect}s total=%{time_total}s\n" https://api.anthropic.com
```

记录出口所在的**国家/地区、城市（如美西洛杉矶）与 ASN 类型**（不把完整 IP 明文写入公开交付文档），后续阶段 4 的浏览器 Profile 与时区设置以该地区为准。

## 5. 断网回退步骤

若启用 TUN 后出现本机断网或 DNS 异常：

1. 在代理客户端界面关闭 **TUN 模式** 与 **系统代理**（若使用裸 `mihomo`，停止对应进程）；
2. 核对默认路由与系统 DNS 是否恢复：
   ```bash
   route -n get default
   scutil --dns | head -n 30
   ```
3. 如客户端残留了手动系统代理或自定义 DNS，用 `networksetup -setwebproxystate "<服务名>" off`、`networksetup -setsecurewebproxystate "<服务名>" off`、`networksetup -setdnsservers "<服务名>" Empty` 一键清空。
