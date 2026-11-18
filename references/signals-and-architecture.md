# 信号模型与网络架构选择

## 1. 客户端与服务端可见的信号边界

Anthropic 公开说明消费者产品会根据请求的公网 IP 及相关信号推断国家/地区级位置，用于服务条款执行与滥用防范；目前说明不收集来自 GPS、Wi-Fi 或蓝牙的精确物理定位，但未公开风控或资格判定的内部信号权重。

因此，本机环境治理需要把**可观察的工程事实**与**未知的平台内部规则**严格分开：

| 使用面 | 服务端或外部可见的网络信号 | 本地环境与协议侧信号 | 常见旁路风险 |
|---|---|---|---|
| 独立指纹浏览器（Web 登录/账单） | HTTP/HTTPS/HTTP3 请求源 IPv4/IPv6、目标域名背后的 DNS 解析器出口 | 浏览器 `Accept-Language`、`Intl` 时区、系统平台与渲染特征、WebRTC ICE/STUN 候选地址、站点定位权限 | 系统代理未必覆盖 UDP/QUIC 或浏览器辅助进程；WebRTC 未禁用非代理 UDP 时可能暴露网卡或真实公网地址 |
| Claude Code CLI 主进程 | API、OAuth 认证、遥测与自动更新连接的源 IP | 进程继承的 `TZ`、`LANG`/`LC_ALL`、系统时区与区域配置、生效配置目录 | 官方仅支持 `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY`，不支持 `SOCKS`；环境变量只是客户端意愿，不是内核强制边界 |
| CLI 触发的工具与子进程 | `curl`、`git`、`node`、`python`、包管理器、MCP 服务各自发起的出站连接 | 子进程自身的解析器调用（如 `scutil --dns`）、双栈直连、应用内 DoH、自行清空代理变量 | 父进程设置了代理变量不代表子进程遵守；单次命令读取网卡或环境变量进入对话也会造成文本侧暴露 |

> Claude Desktop 不在本 Skill 的支持范围内（桌面端内嵌网页与原生进程的网络路径与 CLI 不同，此处不作覆盖承诺）。

## 2. 为什么“只开系统代理”或“只开 TUN”都不够

- **只设系统代理或环境变量**：属于“自愿遵守”。不读代理变量的二进制、原生 Socket 调用、UDP/QUIC、IDE/SSH 直接拉起的进程会直接绕过；代理退出时往往静默回退直连。
- **只开整机 TUN，不上进程沙盒**：正常工作时能接管整机路由与浏览器流量，但**一旦代理客户端崩溃、退出或重载失败，`utun` 虚拟网卡与路由随之消失，macOS 会立即回退到物理网卡直连（fail-open）**，无法满足“代理故障时阻断未授权公网流量”的要求。
- **双层组合（代理/TUN + `sandbox-exec`）**：
  1. **代理/TUN 层**负责“流量从哪个固定出口出去、DNS 在哪里解析、哪些日常流量允许直连”；
  2. **macOS `sandbox-exec`（Seatbelt）层**在内核态强制限制 Claude Code 及其全部子进程**只能连接 `localhost` 回环端口与本地 Unix Socket**，禁止一切公网 IPv4/IPv6/UDP 直连。哪怕代理客户端彻底退出，受限进程也只会连接失败（`EPERM` / 连接拒绝），绝不会把数据包发往物理网卡。

## 3. 三种可选网络架构对比

部署阶段 1 向用户说明以下三种模式，根据设备固定/移动形态与接受度选定其一：

| 维度 | 模式 A：TUN 固定出口 + `sandbox-exec`（默认推荐） | 模式 B：本地代理端口 + `sandbox-exec`（移动设备/轻量推荐） | 模式 C：仅系统代理 + CLI 代理变量（最小兜底） |
|---|---|---|---|
| 适用场景 | 长期固定办公的 Mac（Apple Silicon / Intel） | 经常跨网络移动、不希望整机流量被 TUN 接管的笔记本 | 临时试用、拒绝启用 TUN 与自定义沙盒入口 |
| Claude CLI 全进程防旁路 | 内核级强制（`sandbox-exec`） | 内核级强制（`sandbox-exec`） | 无强制（仅覆盖读取代理变量的程序） |
| 代理停止时阻断直连（fail-closed） | 支持（内核拒绝非回环出站） | 支持（CLI 走沙盒、指纹浏览器绑定本地代理端口） | 不支持（代理退出可能回落直连） |
| 浏览器全流量与 UDP 覆盖 | 整机 TUN + 浏览器禁用非代理 UDP 双重兜底 | 依赖指纹浏览器的代理绑定配置与禁用非代理 UDP | 仅覆盖遵循系统代理的请求 |
| 对整机日常网络的影响 | 影响整机，需为局域网/SSH/远程桌面配白名单 | 不接管整机路由，日常应用保持原网络 | 影响读取 macOS 系统代理的应用 |
| 最高可标记验收结论 | `故障时已验证` | `故障时已验证`（限受控入口与绑定代理的浏览器） | 最高仅可标 `已配置` / `已观测` |

### 多代理客户端并行的注意事项

若用户机器已有日常翻墙用的 Clash/Mihomo，想另起一套独立实例专供 Claude：两套客户端若同时开启 TUN 或争用系统代理、DNS 和监听端口会互相覆盖。只有在明确固定其中一套为本地纯端口监听（不开 TUN、不改系统代理、使用独立混合端口与控制端口）并配合 `sandbox-exec` 指向该专用端口时，并行方案才可成立；否则优先复用单一客户端并通过规则组锁定固定节点。

## 4. 出口节点类型的客观取舍

| 出口路线 | 常见特征 | 现实限制 |
|---|---|---|
| 静态住宅 / 原生 ISP 节点 | 在第三方 IP 库中更常显示为 `isp`，欺诈/代理标记较少 | 带宽与延迟波动较大，价格较高；标签干净不代表免疫平台风控 |
| 海外 VPS 自建出口 | 独享 IP、稳定性可控 | ASN 通常归类为 `hosting` / 云厂商，部分网段可能有历史滥用记录 |
| 商业订阅中的固定专线节点 | 延迟与吞吐较好，接入成本低 | 多人共用出口；需确认供应商允许长期固定单节点且出口 IP 不频繁跳变 |

向用户解释节点选择时，只依据可测量的稳定性、延迟、丢包、出口地区一致性和 IP 库观测结果，不把“非家宽必封”或某个第三方评分说成平台内部规则。

## 5. 模式 C（仅代理变量 + 系统代理）的最小执行与回退

仅当用户明确拒绝模式 A 与模式 B 时采用，并提前告知其不具备进程防旁路与故障阻断保证。

### 记录原状与设置系统代理

```bash
networksetup -listallnetworkservices
networksetup -getwebproxy "Wi-Fi"
networksetup -getsecurewebproxy "Wi-Fi"
networksetup -getproxybypassdomains "Wi-Fi"
scutil --proxy

# 将 "Wi-Fi" 替换为当前活动网络服务名，7890 替换为实际本地混合代理端口
sudo networksetup -setwebproxy "Wi-Fi" 127.0.0.1 7890
sudo networksetup -setsecurewebproxy "Wi-Fi" 127.0.0.1 7890
sudo networksetup -setproxybypassdomains "Wi-Fi" "*.local" "169.254/16" "127.0.0.1" "localhost"
sudo networksetup -setwebproxystate "Wi-Fi" on
sudo networksetup -setsecurewebproxystate "Wi-Fi" on
```

### 配置 Claude Code 与终端代理变量

优先写入 `~/.claude/settings.json` 的 `env` 字段（对 IDE 或非登录 Shell 拉起的会话同样生效），同时可在 `~/.zshrc` 留带标记的配置块：

```json
{
  "env": {
    "HTTPS_PROXY": "http://127.0.0.1:7890",
    "HTTP_PROXY": "http://127.0.0.1:7890",
    "NO_PROXY": "localhost,127.0.0.1,::1"
  }
}
```

```bash
# >>> claude-local-onboard >>>
export HTTPS_PROXY="http://127.0.0.1:7890"
export HTTP_PROXY="http://127.0.0.1:7890"
export NO_PROXY="localhost,127.0.0.1,::1,*.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
# <<< claude-local-onboard <<<
```

> Claude Code 不支持 `SOCKS` 代理，切勿用 `ALL_PROXY=socks5://...` 代替 HTTP(S) 代理。

### 回退命令

```bash
sudo networksetup -setwebproxystate "Wi-Fi" off
sudo networksetup -setsecurewebproxystate "Wi-Fi" off
sudo networksetup -setproxybypassdomains "Wi-Fi" <原绕过列表>
```

再移除 `~/.claude/settings.json` 中的代理字段与 `~/.zshrc` 中的标记块，重开终端即可恢复。

## 参考资料

- [Anthropic：Claude 如何使用位置信息](https://privacy.claude.com/en/articles/11186740-does-claude-use-my-location)
- [Anthropic：Claude Code 企业代理与网络配置](https://code.claude.com/docs/en/corporate-proxy)
- [Anthropic：Claude Code 快速开始](https://code.claude.com/docs/en/getting-started)
- [Anthropic：Web Search 位置说明](https://support.claude.com/en/articles/10684626-enabling-and-using-web-search)
