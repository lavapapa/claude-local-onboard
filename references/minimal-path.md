# 最小可用路径

## 适用与代价

这条路径给不想上整机 TUN 的用户：只把系统代理和 Claude Code 的代理变量固定到同一个节点，再做基础观测。它保留日常网络原样，改动小、回退快。

代价要提前说清：系统代理是“应用愿意读才生效”的设置。浏览器辅助进程、不读代理变量的原生网络请求、UDP/QUIC、由 Finder 或 IDE 直接拉起的二进制、SSH 与会自行清理环境的子进程都可能绕过它；代理进程退出时，流量通常直接回落到直连。因此这条路径的验收结论只能写“已配置/已观测”，不能写成“全进程覆盖”或“故障时阻断”。

如果用户之后需要真正可阻断的边界，回到 [Mihomo TUN 与固定出口](mihomo-tun-fixed-egress.md) 升级。

## 步骤

### 1. 固定节点与端口

选定一个长期稳定的节点，确认订阅允许固定使用，并让客户端持续监听一个本地端口（历史设备上用户偏好 `7890`）。记录客户端版本、配置路径和节点标识的脱敏值。端口被占用或客户端会自动改端口时，先修好再继续。

### 2. 记录原状

```bash
networksetup -listallnetworkservices
networksetup -getwebproxy "Wi-Fi"
networksetup -getsecurewebproxy "Wi-Fi"
networksetup -getproxybypassdomains "Wi-Fi"
scutil --proxy
```

把网络服务名、原代理状态与绕过列表记入恢复材料。多网络服务（Wi-Fi、USB 网卡、Thunderbolt 网桥）要逐项记录，只改当前使用的那一个。

### 3. 设置系统代理

```bash
sudo networksetup -setwebproxy "Wi-Fi" 127.0.0.1 7890
sudo networksetup -setsecurewebproxy "Wi-Fi" 127.0.0.1 7890
sudo networksetup -setproxybypassdomains "Wi-Fi" "*.local" "169.254/16" "127.0.0.1" "localhost"
sudo networksetup -setwebproxystate "Wi-Fi" on
sudo networksetup -setsecurewebproxystate "Wi-Fi" on
```

绕过列表按用户必须保留的局域网、校园网、SSH 与 Tailscale/tailcat 目标调整。确认客户端本身不依赖被改动的代理才能联网；如果执行者正走这条链路，先准备好手动恢复命令。

### 4. 设置 CLI 代理变量

在 `~/.zshrc`（或用户实际使用的 shell 启动文件）中加入带标记的一段，便于整体回退：

```bash
# >>> claude-local-onboard >>>
export HTTPS_PROXY="http://127.0.0.1:7890"
export HTTP_PROXY="http://127.0.0.1:7890"
export NO_PROXY="localhost,127.0.0.1,::1,*.local,192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,*.ts.net"
# <<< claude-local-onboard <<<
```

Claude Code 官方支持 HTTP(S) 代理变量，不支持 SOCKS；不要写 `ALL_PROXY=socks5://` 代替。变量只对新开的 shell 生效，已运行的进程需要重启。GUI 应用若要读取，需按 macOS 的登录环境方式设置或走系统代理。

### 5. 基础观测

```bash
curl -s https://api.ipify.org; echo          # 出口 IPv4
curl -s -6 https://api64.ipify.org; echo     # 预期失败或与出口一致
scutil --dns | head -40                      # 系统 resolver
env | grep -i proxy                          # CLI 变量
date; readlink /etc/localtime                # 本地时间与时区
```

再在 CCBaga 浏览器里用 [网络检查工具目录](network-check-tools.md) 中的页面核对出口、DNS 与 WebRTC；按 [系统语言、时区与地区](macos-timezone-and-locale.md) 核对本地环境。记录观测时间、工具与结论。

### 6. 回退

```bash
sudo networksetup -setwebproxystate "Wi-Fi" off
sudo networksetup -setsecurewebproxystate "Wi-Fi" off
sudo networksetup -setproxybypassdomains "Wi-Fi" <原始列表>
```

删除 shell 启动文件里标记块，重开终端。恢复后重新确认直连可用。

## 交付时写清的三件事

1. 覆盖范围：哪些程序读了系统代理或环境变量，哪些没有验证。
2. 故障行为：代理停止时的实际表现（多数情况下回落直连），以及用户应如何发现。
3. 升级条件：出现需要进程级覆盖、UDP 阻断或旁路防护的需求时，改走 TUN。
