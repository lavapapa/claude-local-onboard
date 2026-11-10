# 位置与出口信号

## 先分清“位置”是什么

Anthropic 对 Claude 消费产品说明：平台会用请求的公网 IP 和其他信号推断国家/地区级的大致位置，以执行服务条款、防止滥用并决定地区可用功能；这类安全用途无法在隐私面板关闭。用户启用本地化搜索等功能时，Claude 也可能根据 IP 推断城市/地区来提供相关结果，这类产品功能用法可在隐私面板关闭。Anthropic 目前表示不会从消费者取得 GPS、Wi-Fi 或 Bluetooth 推断出的精确位置。部署流程应尊重用户真实所在地区和服务条款，不把网络配置当作规避地区资格的工具。

网页浏览器和 macOS 应用在用户授权后可调用地理位置 API。网页 Geolocation API 会请求站点权限；macOS Core Location 可综合 Wi-Fi、蜂窝与 GPS 等信号提供位置。它们与服务端从公网 IP 推断大致位置是两条不同路径。拒绝精确位置授权不会隐藏公网 IP，也不会阻止服务根据出口 IP 判断大致地区。

Anthropic 不公开 Claude/Claude Code 位置与风控判断的完整信号权重。本文区分官方说明、协议上可观察的网络路径和社区猜测；字体、设备标识、时区等可以构成客户端环境信息或一致性线索，现有官方资料不足以证明某个特定字段被 Claude 用作地理定位或封禁依据。

## 各客户端的可能信号

| 使用面 | 服务端可见的网络来源 | 本地环境或协议侧线索 | 需要确认的边界 |
|---|---|---|---|
| Claude.ai / CCBaga 浏览器 | 网站请求的 IPv4/IPv6 源地址、连接域名的 DNS 解析路径；出口 IP 数据库给出的国家/城市、ASN、网络类型可能不完全一致 | 浏览器语言、时区、平台、屏幕与渲染特征；网站获准后可请求 Geolocation；WebRTC ICE/STUN 在允许直连 UDP 时可能揭示候选地址 | 系统代理设置不保证覆盖浏览器每种协议或独立浏览器辅助进程。分别检查浏览器实际出口、DNS、IPv6、WebRTC 和地理位置授权 |
| Claude Desktop | 桌面应用发出的服务连接来源 IP；内嵌网页与桌面原生请求可能采用不同网络路径 | macOS 语言、时区、区域设置；macOS 应用具备在用户授权后请求 Core Location 的平台能力，Anthropic 当前说明其消费者产品不收集 GPS、Wi-Fi 或 Bluetooth 推断出的精确位置；内嵌浏览器会暴露自身请求元数据 | 依据当前桌面版文档确认其是否请求定位和代理行为，不能从 Safari/Chrome 的成功结果推断桌面应用相同 |
| Claude Code CLI | CLI 主请求、认证、更新及其子程序各自建立的网络连接 | 进程继承到的环境变量、用户与系统 locale/timezone、当前配置、启动方式；不同库/子进程可能采用不同代理逻辑 | Claude Code 文档支持启动前设置 HTTP_PROXY/HTTPS_PROXY 和 NO_PROXY，并说明不支持 SOCKS 代理。代理变量是客户端配置，不是操作系统的出站强制策略 |
| Claude Code 中的工具和子进程 | curl、Git、包管理器、浏览器、MCP、由任务触发的辅助程序可能直接访问其目标站点 | 各工具自身的代理设置、系统 resolver、IPv4/IPv6、UDP/QUIC、应用内 DoH | 分别覆盖进程树和网络协议；从 Claude 主进程的代理变量推不出所有子程序都会遵守 |

出口 IP 是线上服务识别请求来源时最直接的一类信号。共享或数据中心出口可能被 IP 情报服务归类为 hosting/VPN/proxy，地理数据库之间也可能出现城市、国家或 ASN 差异。Anthropic 未公开这些分类在其决策中的权重；因此把“家宽才可用”“VPS 必封”或某个 IP 评分当成规则，都超出了公开证据。

时区、语言和地区设置会让设备环境呈现不同的区域信息。它们可以与公网出口形成明显差异，但本身不等同于 GPS 定位。TLS/浏览器指纹可以识别客户端实现或帮助关联访问，不能单独证明用户的物理位置。报告中要分开写“网络出口观测”“浏览器环境观测”“系统本地信息”和“未知服务端权重”。

## 为什么系统代理经常不足

macOS 系统代理是应用可采用的代理设置。不同应用和网络库会自行决定是否读取它，子进程也可能不继承 GUI 的配置。浏览器常能使用系统代理，终端 CLI、某个原生网络请求、后台更新器、MCP 服务器和辅助程序则需要各自核实。

Claude Code 官方网络文档要求在启动前设置 HTTP(S) 代理环境变量，列出 NO_PROXY 行为，并明确表示不支持 SOCKS。代理变量可让遵循这些变量的 Claude Code 网络请求使用代理；环境变量不会强制任意子进程使用代理，也不会拦截自行直连的 IPv4、IPv6 或 UDP。修改 shell 变量后，已经运行的进程也不会自动获得新值。

TUN 在系统网络层接管更多流量，适合建立整机固定出口和统一网络策略；它仍需正确处理路由、DNS、IPv4/IPv6、本地网段、故障阻断及系统批准。TUN 路由配置也不能改变进程本身读取到的语言、时区或本地权限状态。配置与实测需覆盖网络层和本地环境层。

## 家宽、VPS 与远程 Mac 的成本取舍

| 方案 | 可能的收益 | 成本与限制 |
|---|---|---|
| 稳定家宽/ISP 出口 | 出口类型可能被识别为 ISP/住宅网络；长期固定可减少地址变化 | 价格、运营商限制、地理位置和带宽因供应商而异；住宅标记不代表服务必然接受或完全没有其他风险 |
| 海外 VPS | 资源易部署、可固定 IP、适合远程开发和网关服务 | 地址通常属于云/Hosting 网络；具体声誉由 IP 段、运营商与历史共同影响；增加系统维护、账单和远程运维 |
| 托管/租用 Mac mini | 应用和系统运行在远端机器，可减少本地设备与网络路径之间的差异 | 机器、带宽、存储和管理费用更高；“Mac 在海外”不自动意味着住宅出口，也不保证特定账号或服务结果 |
| 本地 Mac + Mihomo TUN | 复用现有设备与订阅，以一条稳定出口和系统级路由完成软件加固，成本与组件较少 | 整机网络受影响，固定节点稳定性重要；需要照顾本地网络、SSH、远程控制和日常访问 |

这些路线解决的问题不同：住宅/ISP 节点着重出口类型，VPS 着重可控的远端计算和网络，远程 Mac 着重把应用运行位置移到远端。本项目的核心是用软件构建可检查、可回退的网络边界：固定出口、最小直连白名单、DNS/IPv6/UDP/WebRTC 检查、Claude 与浏览器全进程覆盖、故障时阻断、旧身份数据隔离，同时保留正常开发与日常网络能力。采用何种路线取决于真实所在地、服务可用性、设备移动情况和用户能接受的成本。

## 引导与判断

Agent 应先说明每类信号能观察什么、不能据此推断什么，再问用户本次目标、设备是否固定、真实所在地、现有订阅与网络、日常必须保留的远程服务。根据回答推荐 TUN 或把专用代理列作需要验证的试用方案。

没有公开证据的风控细节应标为未知，不将社交媒体帖子、仓库营销文案或单个检测页的结果陈述为 Anthropic 的内部规则。对服务地区资格、账号注册或付费资格有疑问时，查阅 Anthropic 当时的支持地区与账户政策；网络配置不能替代资格确认。

## 资料来源

- [Anthropic：Claude 是否使用我的位置](https://privacy.anthropic.com/en/articles/11186740-does-claude-use-my-location)
- [Anthropic：Claude Code 企业网络配置](https://code.claude.com/docs/en/corporate-proxy)
- [Anthropic：Claude Code 安装与使用条件](https://docs.anthropic.com/en/docs/claude-code/getting-started)
- [Anthropic：Web Search 如何使用位置](https://support.anthropic.com/en/articles/10684626-enabling-and-using-web-search)
- [MDN：Geolocation API 与权限](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation_API)
- [Apple：Core Location 定位来源](https://developer.apple.com/documentation/CoreLocation/getting-the-current-location-of-a-device)
- [Mullvad：WebRTC 说明与检测方式](https://mullvad.net/en/help/webrtc)
- [BrowserLeaks：WebRTC 测试](https://browserleaks.com/webrtc)

每次实际接入时重新检查官方政策和客户端网络文档。IP 归属库和第三方测试服务的观测结果随时间变化，应记录工具与日期。
