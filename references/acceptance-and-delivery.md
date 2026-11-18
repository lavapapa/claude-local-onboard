# 全链路验收、故障阻断测试与交付恢复卡

先在**未登录任何 Claude 账号**的状态下完成阶段 6 环境预验收与受控故障阻断测试，全部必需项达标后，再由用户在阶段 7 登录账号并验收真实 CLI 请求。

## 1. 安全的检查工具与观测手段

不要安装来历不明或无社区审阅的第三方测速/检测 CLI（尤其不要把含密码的代理配置或订阅链接喂给未知脚本）。统一采用**成熟公开检测页 + macOS 原生标准命令**：

| 检查维度 | 浏览器侧公开检测页（由用户知情后打开） | 本机 / 沙盒内原生命令探针 | 观测边界说明 |
|---|---|---|---|
| 公网出口 IPv4 / IPv6 与地区 | [Mullvad Check](https://mullvad.net/en/check)、[IPLeak](https://ipleak.net/) | `claude-network-run curl -fsS https://www.cloudflare.com/cdn-cgi/trace` | 至少用两个独立端点交叉核对，确认出口一致且无直连 IPv6 旁路 |
| IP ASN 类型与风险参考 | [IPinfo](https://ipinfo.io/)、[Scamalytics](https://scamalytics.com/ip) | `curl -fsS -x http://127.0.0.1:7890 https://ipinfo.io/json` | 第三方评分与 `isp`/`hosting` 分类仅供环境参考，不等同于平台内部规则 |
| DNS 解析出口 | [BrowserLeaks DNS](https://browserleaks.com/dns)、[Mullvad Check](https://mullvad.net/en/check) | 通过代理请求外部 Trace / DoH 检测页，并在沙盒内验证 `scutil --dns` 被屏蔽或无本地直连泄露 | 确认 DNS 解析器位于目标出口同区域或海外公共云，不出现本地宽带/校园网递归 DNS |
| WebRTC / STUN / UDP | [BrowserLeaks WebRTC](https://browserleaks.com/webrtc)、[Mullvad Check](https://mullvad.net/en/check) | 沙盒内 Python UDP Socket 直连探针（见第 2 节） | 浏览器候选地址不得暴露本机局域网、Mesh 地址或真实公网 IP |

## 2. 阶段 6 登录前必跑探针（沙盒与故障阻断验收）

### 探针 A：核验 `sandbox-exec` 内核强制边界（直连与管理口全拒）

运行以下一键只读探针，验证受保护环境内**直连 IPv4、直连 IPv6、直连 UDP、访问代理控制口 `9090` 全部返回 `EPERM`**，而本地代理端口 `7890` 可正常连通：

```bash
claude-network-run /usr/bin/python3 -c '
import errno, socket, sys

checks = []

def expect_eperm(name, family, socktype, addr):
    s = socket.socket(family, socktype)
    s.settimeout(2)
    try:
        s.connect(addr)
        if socktype == socket.SOCK_DGRAM:
            s.send(b"\x00")
        checks.append((name, False, "unexpectedly succeeded"))
    except OSError as e:
        ok = (e.errno == errno.EPERM)
        checks.append((name, ok, f"errno={e.errno} ({e.strerror})"))
    finally:
        s.close()

expect_eperm("direct_ipv4_tcp", socket.AF_INET, socket.SOCK_STREAM, ("1.1.1.1", 443))
expect_eperm("direct_ipv6_tcp", socket.AF_INET6, socket.SOCK_STREAM, ("2606:4700:4700::1111", 443))
expect_eperm("direct_ipv4_udp", socket.AF_INET, socket.SOCK_DGRAM, ("1.1.1.1", 53))
expect_eperm("proxy_ctrl_9090", socket.AF_INET, socket.SOCK_STREAM, ("127.0.0.1", 9090))

try:
    with socket.create_connection(("127.0.0.1", 7890), timeout=2):
        checks.append(("proxy_7890_reachable", True, "connected"))
except OSError as e:
    checks.append(("proxy_7890_reachable", False, str(e)))

for name, ok, detail in checks:
    print(f"[{'PASS' if ok else 'FAIL'}] {name}: {detail}")
sys.exit(0 if all(ok for _, ok, _ in checks) else 1)
'
```

### 探针 B：受控代理停机 fail-closed 阻断测试

> **注意**：若当前执行任务的 Agent 自身依赖本机代理联网，短暂停止代理客户端会中断当前回合。可写成单条自动“停代理 → 测阻断 → 立即拉起代理恢复”的原子命令，并提前告知用户。

验收标准：
1. 暂停或退出本地代理客户端，确认 `127.0.0.1:7890` 已关闭；
2. 此时执行 `claude --version` 或 `claude-network-run curl -fsS --max-time 5 https://www.cloudflare.com/cdn-cgi/trace` **必须立即报错阻断，绝不能返回本机真实公网出口**；
3. 在独立指纹浏览器中刷新 IP 检测页，确认请求失败而不会回落本机直连；
4. 重新启动代理客户端，确认固定出口自动恢复一致。

### 探针 C：日常工作流兼容核验

逐一验证用户在阶段 1 声明必须保留的访问（如局域网设备、内网网站、SSH 远端主机、Tailscale、远程桌面）正常可用且延迟无异常。

## 3. 阶段 7 用户登录与 CLI 终验

登录前预验收全部通过后：

1. **浏览器端登录**：引导用户在独立指纹浏览器中亲自完成 Claude 登录、验证码、订阅支付或授权确认。**严禁将 Cookie、OAuth 授权码、Token、密码或支付信息贴入 Agent 对话或日志。**
2. **CLI 端登录与请求验证**：
   - 在终端执行 `claude`（若自动拉起浏览器授权，确保在独立指纹浏览器中打开授权链接完成绑定）；
   - 发起一次最小对话请求，同时在另一个终端核对运行中的真实 Claude 进程确实处于 `sandbox-exec` 限制下：
     ```bash
     python3 -c '
     import ctypes, subprocess
     lib = ctypes.CDLL("/usr/lib/libsandbox.1.dylib")
     pids = subprocess.check_output(["pgrep", "-f", "claude/versions/"], text=True).split()
     for pid in pids:
         restricted = lib.sandbox_check(int(pid), b"network-outbound", 0) == 1
         print(f"PID {pid}: network-outbound restricted = {restricted}")
     '
     ```
     输出必须为 `True`。

## 4. 验收状态定义与脱敏摘要模板

每项检查结果仅允许标记为以下五种状态之一（不得将单次网页绿灯夸大为全链路通过）：

- **`已配置`**：配置已写入，尚无独立运行态证据；
- **`已观测`**：正常状态下实测符合预期；
- **`故障时已验证`**：在受控断开代理/故障注入下实测严格阻断且恢复正常；
- **`未覆盖`**：受模式限制（如模式 C）未覆盖或尚未测试；
- **`不通过`**：出现直连泄露、出口漂移或破坏日常必需网络（必须回退修复后才能交付）。

### 交付给用户的脱敏验收摘要表

> 摘要中**严禁包含完整公网 IP、订阅 URL、控制口 Secret、令牌或账号名**（出口 IP 仅写国家/城市与 ASN 类型，或做掩码脱敏）。

| 检查项 | 观测方法 | 实测结论 | 状态 |
|---|---|---|---|
| 固定出口地区与类型 | Cloudflare Trace + IPinfo + 浏览器检测页 | 例：美国加利福尼亚（ISP / 低风险参考），多端一致 | 已观测 |
| DNS 与 IPv6 防泄漏 | BrowserLeaks DNS + 沙盒内双栈探针 | DNS 解析位于出口同区，直连 IPv6 返回 `EPERM` | 已观测 |
| 浏览器 WebRTC / UDP | BrowserLeaks WebRTC + Mullvad Check | 无本机内网或真实公网 ICE 候选泄露 | 已观测 |
| CLI 全进程树沙盒 | 探针 A + 真实 Claude PID `sandbox_check` | 非回环 TCP/UDP 与 `9090` 控制口均被内核拒绝 | 已观测 |
| 代理故障阻断（fail-closed） | 探针 B 受控停代理实测 | 代理关闭期间启动与请求立即阻断，无回退直连 | 故障时已验证 |
| 本地时区与语言一致性 | 浏览器 `Intl` + CLI `date` / `TZ` | 例：`America/Los_Angeles` / `en-US`，与出口一致 | 已观测 |
| 日常直连白名单兼容 | 实测局域网 / SSH / Mesh VPN / 远程桌面 | 保留路径正常连通 | 已观测 |

## 5. 最小安全加固与用户恢复卡片

最终交付时，向用户留存一份简明**本机交付与恢复卡片**（不含任何敏感凭据明文）：

1. **组件与版本清单**：macOS 版本与架构、代理客户端版本、指纹浏览器名称与 Profile 标识、Claude Code 版本与安装日期。
2. **关键文件路径**：
   - 统一启动入口：`~/.local/bin/claude` 与 `~/.local/bin/claude-network-run`
   - 沙盒策略文件：`~/.config/claude-network/outbound.sb`
   - 代理持久覆写配置路径
   - 阶段 2 旧环境备份目录路径（如 `~/.local/backups/claude-onboard-<timestamp>`）
3. **日常异常处置与一键回退步骤**：
   - **Claude 提示代理不可用无法启动**：检查代理客户端是否运行、`FIXED-EXIT` 节点是否在线，切勿临时删掉 `outbound.sb` 裸跑；
   - **整机网络因 TUN 异常需紧急恢复直连**：关闭代理客户端 TUN 与系统代理 → `networksetup` 清空残留代理/DNS → 验证 `route -n get default` 恢复物理网关；
   - **恢复原系统时区/语言**（如改动过全局设置）：`sudo systemsetup -settimezone <原时区名>` 并按需重开自动时区；
   - **卸载受控包装恢复原生 CLI**：将 `~/.local/bin/claude` 重新软链至 `~/.local/share/claude/versions/<版本>`。
4. **注意不要默认添加后台轮询守护进程**：除非用户主动要求自动重连/健康看门狗，否则优先依赖代理客户端内置重连与启动器预检；确需添加 `launchd` 任务时，先向用户说明触发频率、日志路径与卸载命令。
