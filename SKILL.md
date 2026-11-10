---
name: claude-local-onboard
description: 在 macOS 上逐步建立 Claude Code 本地接入环境：盘点与备份旧数据、CCBaga 独立指纹浏览器、Mihomo TUN 固定出口、系统语言与时区一致性、登录前网络验收、CLI 安装与登录后验收。
---

# Claude Local Onboard

本 Skill 引导 Agent 一步步与用户完成 macOS 上的 Claude Code 接入。核心方法是使用软件建立可观察、可回退的网络边界与干净工作环境：默认推荐 Mihomo TUN、固定出口和白名单直连，配合必选的 CCBaga 独立指纹浏览器，并把系统语言、时区与地区显示调到用户实际工作的地区。

## 核心目标

1. 给你清理出来一套“相对”彻底干净的环境
2. 防范 claude code 的各种已知检测手段 + 尽量减少“手滑”误操作的失误暴露 IP 的风险
3. 减少不必要的限制的前提下，让你的 agent（比如 codex）自动给你打造一套高度可用的运行环境。

这里的“防范检测”指降低意外直连、DNS/IPv6/WebRTC 暴露、出口切换和本地环境与出口地区不一致等已知风险；不承诺改变平台的资格或风控判断。

## 工作原则

1. 先备份，再清理；网络、时区与启动配置的每次改动都留有原值和回退步骤。
2. 固定单一出口，代理故障时阻断未授权公网流量，不回退到 DIRECT。
3. 浏览器、CLI 与子进程走同一条受控路径；启动器只负责方便，系统级出站控制负责兜底。
4. 每个阶段以可观察结果验收，不通过就停在原地修复，并输出脱敏后的检查摘要。

## 为什么代理还不够

Claude 服务端可根据请求的公网 IP 和其他信号推断大致国家/地区。浏览器、桌面端、CLI 和它们的子进程还可能使用不同的网络栈、DNS、IPv4/IPv6、WebRTC/UDP 与本地环境设置，因此“网页能打开”或“系统代理已启用”都不能证明所有请求走同一出口。Anthropic 说明消费者产品会用 IP 和其他信号作国家/地区级位置推断；不公开完整的信号权重。

浏览器可能暴露请求出口、DNS、IPv6 与 WebRTC 候选地址，也可能在用户授权后使用地理位置 API；Claude Code 需要按官方文档配置 HTTP(S) 代理，且不支持 SOCKS。时区、语言等属于本地环境信息，与出口 IP 是两类信号，任何一层的配置都不会自动改变另一层。详见 [位置与出口信号](references/location-and-egress-signals.md)。

本项目以软件加固为核心：固定稳定出口、限定直连白名单、覆盖 Claude Code 与 CCBaga 全进程、代理故障时阻断，并检查 DNS、IPv6、UDP/WebRTC、系统时区与出口地区是否一致、普通工作流是否仍可用。家宽、VPS、远程 Mac mini 各有成本与限制；它们都不能单独保证账号或服务结果。

## Agent 逐步引导流程

Agent 应按阶段推进。每一阶段先说明目的、需要用户提供的信息和可能影响；完成后报告观察结果与未完成项，再引出下一阶段。一次只处理当前阶段所需的选择，避免把整套配置问题集中抛给用户。需要用户亲自完成的订阅提供、系统授权、登录、验证码和付费，明确指出操作位置与原因；其余可逆的盘点和资料准备继续自行推进。

### 1. 确认目标与设备现状

先询问用户希望使用 Claude 的方式、所在地与期望的出口地区、设备是否经常移动、当前网络/代理状态、日常必须保留的访问（如校园网、局域网、SSH、UU、Tailscale/tailcat、远程开发），以及旧 Claude 数据的处理意愿。随后只读盘点 macOS 版本与芯片、Claude Code、CCBaga、Mihomo、浏览器、网络服务、代理端口、当前出口，以及系统语言、时区、地区显示和定位权限的当前值。只记录脱敏摘要。

同时记录安装当天各工具的版本号、下载来源和签名方式，便于日后复现；版本不写死，但来源要能说清。详见 [目标盘点与数据边界](references/preflight-and-data-boundary.md)。

向用户解释两种网络选择：固定、稳定的 Mac 优先 TUN；设备经常移动时可评估专用进程代理，但本流程历史上只验证过 TUN。确认用户理解 TUN 对整机网络的影响，以及节点切换尤其跨国切换应尽量少。需要订阅节点时，在此阶段确认用户是否已有订阅。

若用户明确不愿上整机 TUN，走 [最小可用路径](references/minimal-path.md)，并说明它只覆盖读取代理变量的程序，浏览器辅助进程、UDP 与直连二进制仍可能绕过。

### 2. 备份并清理旧 Claude 环境

先确认当前执行者。如果流程正在 Claude Code 自身中运行，清理其配置或重装 CLI 会中断流程；由 Codex 或用户终端执行这些步骤，本 Agent 先完成只读盘点和命令准备。再检查正在运行的 Claude Code/Desktop 进程与未完成任务；说明 CCBaga 身份切换、清理或重装可能结束哪些进程，安排在用户选择的时机执行。确认要保留的配置、凭据相关状态和历史数据后，默认先做完整备份并确认备份可读取，再从原目录清理旧 Claude 数据；旧会话迁移不作为推荐路径。用户明确不保留时，按已确认范围清除。

优先评估可审阅源码的第三方清理工具。区分活动配置、可恢复身份归档和可删除构建/缓存文件；特别保护用户要保留的 CCBaga 账号归档、浏览器 profile 与恢复材料。确认清理工具的目标目录、备份策略、删除语义和日志内容后再执行。新身份不复用旧会话或日常浏览器站点数据。具体范围见 [目标盘点与数据边界](references/preflight-and-data-boundary.md)。

### 3. 配置出口与日常网络白名单

默认使用 [Mihomo TUN 与固定出口](references/mihomo-tun-fixed-egress.md)。由用户提供订阅后，查阅部署当时的 Mihomo 官方文档和发行版，评估节点稳定性、延迟、丢包、出口一致性与目标服务连通性，选定一个固定节点。仅用户确认的国内目标及指定 IP 直连，其余公网流量经固定出口。变更前确认执行者自身是否经这条代理联网；如果会受影响，提前告诉用户当前 Agent 可能掉线，并把原配置和手动恢复路径准备好。启用 TUN、修改 DNS 或系统路由前，还要准备用户可理解的回退方式。

将系统代理、CLI、CCBaga 和客户端控制面统一到一个代理入口。历史使用中用户偏好 7890；迁移时先核对当前客户端的监听与设置行为，再统一采用单一端口。为用户常用的局域网、校园网、SSH、UU、Tailscale/tailcat 和 Simulation 路径明确规则，避免网络加固拖慢或中断这些工作流。网络恢复优先重连原节点；故障时保持未授权公网流量阻断。优先采用 Mihomo 已有健康探测与重连能力；需要新增常驻恢复服务时先说明依赖、权限与故障行为。

节点可用后立即做一次出口观测，把“出口 IP 的国家/地区、ASN 与网络类型”记入本阶段摘要；后面的时区与语言设置以这个地区为参照。

若用户选择专用进程代理，先读取 [专用进程代理评估](references/dedicated-process-proxy.md)。将其作为次选试用架构，建议在 Pro 账号上观察长期稳定性后再考虑升级订阅。

### 4. 安装 CCBaga 与 Claude Code，统一本机环境

CCBaga 是必选项；它本身就是独立指纹浏览器，并兼有 Claude 身份管理与多账号切换。查阅 [CCBaga 与浏览器](references/ccbaga-and-browser.md)，确认部署当天的支持平台、安装来源与最新稳定版，记录版本号与发布日期。建立干净工作身份和专用浏览器 profile。

再按 Anthropic 官方文档安装当前最新稳定版 Claude Code，确定唯一 CLI 安装与启动入口。

然后处理本机环境一致性：系统语言、时区、地区显示、定位权限应与用户实际工作地区相符。时区优先在 CCBaga profile 内设置；需要改 macOS 系统时区时，按 [系统语言、时区与地区](references/macos-timezone-and-locale.md) 记录原值、设置新值、验证并保留回退步骤。区域设置、输入法与日期格式按同样方式处理；定位权限只授予确有需要的应用。改动会同时影响日历、提醒、日志时间戳、定时任务触发时刻与 Git 提交时间，执行前说明这些影响。

为用户设置一个清楚、稳定的日常启动入口，并在启动前提示当前受控网络状态与固定节点。入口状态无法确认时先停下来显示可操作的恢复提示；操作系统级出站控制仍须覆盖直接调用二进制、SSH、IDE 和子进程，启动器本身不承担防旁路职责。不要依赖用户每次手动记住代理变量、端口或节点。

### 5. 登录前完成环境验收

先用 CCBaga 的空白工作 profile 和已安装但未登录的 CLI 验收固定出口、DNS、IPv4/IPv6、WebRTC/UDP、进程覆盖、入口旁路、代理故障阻断、系统时区与出口地区的一致性，以及用户指定的日常网络。检查工具见[网络检查工具目录](references/network-check-tools.md)。存在必需项不通过时，先修复网络环境，再让用户登录。检查细节见 [登录与 CLI 验收](references/login-and-cli-acceptance.md)。

### 6. 引导用户登录并验收 CLI

环境预验收通过后，指导用户打开 CCBaga 的独立浏览器，由用户本人完成 Claude 登录、订阅付费、验证码、系统授权和钥匙串确认。随后从约定 CLI 入口登录并发起实际请求，确认浏览器与 CLI 使用预期身份，出口保持选定节点。认证材料不进入聊天、脚本参数或日志。具体步骤见 [登录与 CLI 验收](references/login-and-cli-acceptance.md)。

记录每一项为“已配置、已观测、故障时已验证、未覆盖、不通过”之一。有必需项“不通过”时，回到对应阶段修复；有必需项未覆盖时，交付状态写为未完成。

### 7. 输出验收证据，收尾并讨论可选扩展

每个阶段收尾时给用户一段脱敏摘要，至少覆盖：出口 IP 的国家/地区与 ASN 类型、DNS 解析位置、IPv4/IPv6 状态、WebRTC/UDP 观测结果、系统时区与语言是否与出口地区一致、代理故障时的阻断行为。摘要只放结论和观测方式，不放完整 IP、订阅地址、令牌或账号信息；字段与记录格式见 [登录与 CLI 验收](references/login-and-cli-acceptance.md)。

阅读 [加固与恢复](references/hardening-and-recovery.md)，整理软件版本与来源、脱敏节点标识、配置位置、旧数据备份位置、恢复方法、验收证据和未完成项。

Claude Code 更新或重装后，重新检查统一启动入口、直接二进制路径与系统级网络阻断是否仍然生效；只在实际更新/重装发生时做这项回归验收。

只有用户提出需求后，才评估 Paseo、T3 Code、Orca 等远程接入方案或多账号并行。远程 Agent 若在这台 Mac 上运行，仍须遵循同一网络出口和故障阻断策略。

## 渐进式参考文档

| 文档 | 何时读取 |
|---|---|
| [位置与出口信号](references/location-and-egress-signals.md) | 解释 IP、浏览器、桌面端、CLI 和本地环境各自能暴露什么 |
| [目标盘点与数据边界](references/preflight-and-data-boundary.md) | 盘点、备份或清理旧 Claude 环境 |
| [Mihomo TUN 与固定出口](references/mihomo-tun-fixed-egress.md) | 设置推荐架构、白名单、固定节点及日常网络恢复 |
| [CCBaga 与浏览器](references/ccbaga-and-browser.md) | 安装 CCBaga 并建立身份和独立浏览器 profile |
| [系统语言、时区与地区](references/macos-timezone-and-locale.md) | 核对、修改并回退 macOS 语言、时区、地区与定位权限 |
| [最小可用路径](references/minimal-path.md) | 用户不愿上整机 TUN 时，只做固定出口与 CLI 代理变量 |
| [专用进程代理评估](references/dedicated-process-proxy.md) | 评估适用于移动设备的代理试用方案 |
| [网络检查工具目录](references/network-check-tools.md) | 选择第三方出口、DNS、IP 质量及 WebRTC 检查工具 |
| [登录与 CLI 验收](references/login-and-cli-acceptance.md) | 引导用户完成认证、验收各入口并输出脱敏证据 |
| [加固与恢复](references/hardening-and-recovery.md) | 设定最小必要保护、故障处理与交付记录 |

## 当前资料入口

- [Anthropic 位置使用说明](https://privacy.anthropic.com/en/articles/11186740-does-claude-use-my-location)
- [Claude Code 安装文档](https://code.claude.com/docs/en/setup)
- [Claude Code 网络代理文档](https://code.claude.com/docs/en/corporate-proxy)
- [Mihomo 官方文档](https://wiki.metacubex.one/)
- [CCBaga 项目](https://github.com/liseami/CCBaga)
- macOS 时区与语言：`man systemsetup`、系统设置 → 通用 → 日期与时间

部署时重新核对服务地区政策、官方文档、软件兼容性和发行版本。
