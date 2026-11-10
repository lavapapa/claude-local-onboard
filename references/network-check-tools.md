# 网络检查工具目录

本目录用于复用现有第三方工具，不另建一套独立测试 Skill，也不把网页绿灯视为全链路保证。检查前先说明第三方网站会收到公网 IP、请求时间、浏览器信息等数据；用户选择可接受的工具后再打开。页面只代表该浏览器、该时刻、该站点测试路径的观测结果。

## 工具与适用范围

| 工具 | 可复用的观测 | 使用边界 |
|---|---|---|
| [Mullvad Connection Check](https://mullvad.net/en/check) | 浏览器公网出口、DNS 泄露和 WebRTC 检查 | 由独立 VPN 供应商运营；不同节点/浏览器应分别检查。黄色/无法判定不等于通过。 |
| [BrowserLeaks WebRTC](https://browserleaks.com/webrtc) | WebRTC/STUN 候选和浏览器可见的远端、本地地址 | 适合检查 CCBaga 内的浏览器 profile；只覆盖网页测试触发的 WebRTC 流量。 |
| [BrowserLeaks 总览](https://browserleaks.com/) | DNS、IPv6、WebRTC、浏览器可见网络信息等 | 分页检查各自覆盖的协议；不能替代操作系统级路由检查。 |
| [IPLeak](https://ipleak.net/) | 公网 IP、DNS、IPv4/IPv6 与浏览器侧泄露观测 | 旧标签页/缓存可能保留旧结果；在连接稳定后的新标签页重复检查。 |
| [IPinfo Privacy Detection](https://ipinfo.io/developers/privacy-standard-api) | 对已观测公网 IP 评估 VPN、代理、Tor、relay 或 hosting 标记 | 部分字段需付费/API token；结果依赖其数据与分类。只提交公网出口 IP，不把密钥放入命令历史。 |
| [dns-leak-detector-cli](https://github.com/ricco020/dns-leak-detector-cli) | macOS 命令行发起系统 DNS 查询、查看出口 IPv4/IPv6、检测 DNS resolver 并输出 JSON | 项目较新且维护/方法学需部署时审阅；会访问 ipify、ip-api 与 bash.ws。它补充系统解析观测，不检查 WebRTC、UDP 全覆盖或 Claude 进程树。避免高频运行，公共检测端点有限流。 |
| [AnixOps Speedtest](https://github.com/AnixOps/AnixOps-speedtest) | 读取 Mihomo/Clash 配置或订阅，批量比较节点延迟、速度、丢包、出口与部分服务可达性 | 适合评估候选节点，不证明当前整机路由或 Claude 进程安全。订阅 URL 含凭据时不要放入共享命令历史；先审阅工具如何处理本地配置、下载核心和生成报告。 |
| CCBaga 内置环境检查 | CCBaga 展示的 IP、地区、语言、时区及浏览器环境 | 与独立外部观测交叉核验；项目对部分泄露检查独立性有局限说明，部署时看当前代码/文档。 |

如果某工具停止维护、改为要求安装未知扩展或超出用户愿意披露的数据范围，部署时替换为仍可用的同类服务，并记录工具名称和检查日期。

## 覆盖缺口

- 网页出口检查不能证明 Claude Code 子进程、原生网络调用或系统后台请求遵循相同路由。
- WebRTC 页面通常通过 STUN 检查浏览器 ICE 候选，不能代表任意 UDP 服务、游戏、QUIC 或所有目标端口。
- DNS 测试仅观测测试域名触发的解析。应用内 DoH、缓存、系统 resolver 和其他域名策略可能不同。
- IP 信誉/代理分类是第三方数据库判断，可能误报或漏报；节点质量还需结合目标服务连接稳定性、延迟和丢包。
- Mihomo 面板显示代理节点在线，仅说明控制面/节点状态，不能证明 Mac 的所有进程都通过该节点。

因此，网络验收还要结合 Mihomo 路由/连接观测、CLI 请求结果、进程树与启动路径、故障时阻断、以及普通开发工作流。所需证据范围见 [登录与 CLI 验收](login-and-cli-acceptance.md)。不要实现与现有工具重复的复杂测试脚本；只有多个工具缺少同一项必要覆盖时，才设计最小自动化补充。

节点预筛可先用 AnixOps Speedtest 这类能直接读取 Mihomo 配置的现成 CLI；部署前审查项目维护状态、凭据处理方式与节点测试的出网范围。DNS 系统级交叉检查可评估 dns-leak-detector-cli，但需把它视为新兴第三方候选，先审阅源码和其外部 API 依赖。浏览器侧继续使用 BrowserLeaks、Mullvad Check 或 IPLeak。这里记录候选工具，不表示本次已经安装或运行。

## 进一步资料

- [Mullvad WebRTC 说明](https://mullvad.net/en/help/webrtc)
- [Mullvad DNS 检查说明](https://mullvad.net/en/help/dns-leaks)
- [IPinfo Privacy Detection API](https://ipinfo.io/developers/privacy-standard-api)
