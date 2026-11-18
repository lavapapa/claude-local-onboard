---
name: claude-local-onboard
description: 在 macOS（Apple Silicon 与 Intel）上搭建、验收并长期维护防泄漏的 Claude Code 本地环境：盘点备份与清理旧数据、Mihomo 固定出口与直连白名单、独立指纹浏览器与时区/语言对齐、macOS sandbox-exec 进程级强制沙盒与受控启动入口、受控故障阻断测试及长期防漂移巡检。
---

# Claude Local Onboard

本 Skill 引导 Agent 在 macOS（Apple Silicon 与 Intel 通用）上逐步与用户建立一套干净、可验收、可回退的 Claude Code 本地运行环境，并提供后续日常维护规范。

## 核心架构与工作原则

本方案采用**双层防泄漏 + 隔离浏览环境**设计，不绑定单一商业软件：

1. **路由与固定出口层（Mihomo 系客户端 / TUN）**：提供统一本地 HTTP(S) 混合代理端口（如 `127.0.0.1:7890`），锁定单一固定出口节点（禁止自动跨区漂移或回退 `DIRECT`），接管 DNS 解析，并为必需的局域网、SSH、Mesh VPN、远程桌面配置显式直连白名单。
2. **进程级强制阻断层（macOS 原生 `sandbox-exec` + 受控启动入口）**：核心防旁路设计。通过 Seatbelt 策略在内核态禁止 Claude Code 及其整棵子进程树发起任何非回环公网连接（IPv4 / IPv6 / TCP / UDP 直连一律返回 `EPERM`），封禁代理控制端口，并在启动前做代理 `CONNECT` 连通性预检。即使代理客户端退出或 TUN 失效，受限进程也严格保持 **fail-closed（阻断不直连）**。
3. **独立指纹浏览器与环境对齐层**：使用 AdsPower、MoreLogin、CCBaga 或独立 Chromium Profile 隔离网页登录环境，与 CLI 共用同一固定出口，并按用户确认的层级将时区与语言对齐到目标出口地区（例如美西加利福尼亚 `America/Los_Angeles`）。

执行铁律：
- **先备份再改动**：清理旧数据、修改网络/代理或调整系统时区前，必须先留存可读备份与原值回退命令。
- **逐阶段渐进确认**：每阶段先说明目的、影响与需要用户提供的输入，一次只推进当前阶段决策，不一次性抛出一堆问题。
- **敏感操作由用户本人执行**：订阅导入、系统权限批准、账号登录、验证码与付款均由用户在本机界面亲自完成，凭据、Cookie、Token 与订阅地址绝不进入聊天或日志。
- **以实测证据验收**：单次网页绿灯或配置文件写入不能代表通过；必须验证到真实进程沙盒状态与代理停机阻断行为，输出脱敏摘要。

## 渐进式引导流程（首次接入 7 阶段）

若用户已完成首次接入、仅需日常巡检或排查配置漂移，直接跳读 [接入后的长期维护](references/ongoing-maintenance.md)。

### 阶段 1：盘点现状与确定方案

先向用户确认：期望固定的出口国家/地区、设备属于固定办公还是经常移动、日常必须保留直连的网络路径（如局域网、内网域名、SSH、Tailscale 等 Mesh VPN、UU 等远程桌面）、时区对齐偏好（仅工作环境局部对齐还是连同 macOS 全局时区一并修改），以及旧 Claude 数据的处理意愿。

随后执行无副作用的只读盘点（macOS 版本与架构、现有 CLI 入口与进程、代理客户端、浏览器、当前网络服务与时区语言），输出脱敏摘要，并向用户介绍三种网络模式供选定（默认推荐 **模式 A：TUN 固定出口 + `sandbox-exec`**；移动笔记本可选 **模式 B：本地代理端口 + `sandbox-exec`**）。详见 [信号模型与网络架构选择](references/signals-and-architecture.md) 与 [环境盘点、备份与旧数据清理](references/preflight-and-cleanup.md)。

### 阶段 2：备份并清理旧 Claude 环境

先确认当前执行者：若当前 Agent 正运行在本机待清理的 Claude Code 自身会话内，切勿自删配置或重装二进制；改为先准备备份与清理命令交由外部 Agent（如 Codex）或用户独立终端执行。

对旧配置、状态文件与会话做完整归档备份并校验可读后，按确认范围清理 `~/.claude`、`~/.claude.json`、多余安装入口及用户确认清除的钥匙串凭据。新身份默认不导入旧会话历史。具体路径全表与步骤见 [环境盘点、备份与旧数据清理](references/preflight-and-cleanup.md)。

### 阶段 3：配置固定出口与直连白名单

协助用户在 Mihomo 系图形客户端（FlClash、Clash Verge Rev、Mihomo Party、Stash 等）或裸 `mihomo` 核心中完成持久覆写配置：统一本地代理监听端口、给控制接口设置监听与认证 Secret、锁定单一 `FIXED-EXIT` 出口节点、配置远程代理 DNS 解析及必需的直连白名单。

若当前执行 Agent 正经由待改代理联网，变更前必须先提示可能断连并把手动恢复步骤交给用户。节点连通后记录出口所在国家/城市与 ASN 类型（脱敏）。详见 [固定出口、Mihomo TUN 与直连白名单](references/egress-and-tun.md)。

### 阶段 4：准备独立指纹浏览器与对齐本地环境

协助用户准备独立指纹浏览器环境（可选 AdsPower、MoreLogin、CCBaga 或独立 Chromium Profile），将其代理绑定到本地统一端口，关闭非代理 WebRTC UDP 与地理位置权限。

按用户确认的方案将浏览器 Profile、CLI 环境变量（及可选的 macOS 系统全局时区/语言）与固定出口所在地区对齐（如美西出口设为 `America/Los_Angeles`、`en-US`），记录原值与回退命令。详见 [独立指纹浏览器与本地时区/语言一致性](references/browser-and-locale.md)。

### 阶段 5：安装 Claude Code 并部署 `sandbox-exec` 受控启动链

通过官方原生安装器安装最新稳定版 Claude Code，部署 `~/.config/claude-network/outbound.sb`（Seatbelt 出站限制策略）、`~/.local/bin/claude-network-run`（代理 `CONNECT` 预检 + 防重入检测 + `sandbox-exec` 包装）以及 `~/.local/bin/claude` 统一启动入口。

确认 `PATH` 唯一命中受保护入口，且无登录下执行 `claude --version` 与直连 `EPERM` 快速自检通过。详见 [`sandbox-exec` 进程沙盒与受控启动链](references/seatbelt-and-launcher.md)。

### 阶段 6：登录前预验收与受控故障阻断测试

在未登录任何 Claude 账号的状态下，使用独立指纹浏览器空白环境与受控 CLI 入口逐项实测：公网出口一致性、DNS 与 IPv6 防泄漏、浏览器 WebRTC/UDP、沙盒内直连 TCP/UDP 与代理管理端口 `EPERM` 拒绝、受控停止代理时的 fail-closed 阻断行为，以及日常直连白名单工作流。

任何必需项“不通过”时立即停在当前阶段修复，严禁带病进入账号登录。探针命令与检查项见 [全链路验收、故障阻断测试与交付恢复卡](references/acceptance-and-delivery.md)。

### 阶段 7：引导用户登录、完成 CLI 验收与交付恢复卡

引导用户本人在独立指纹浏览器中完成账号登录与授权，再从统一入口 `claude` 完成 CLI 认证并发起一次真实请求，同时用 `libsandbox` 对运行中的真实 Claude 进程验证 `network-outbound` 受限生效。

最后输出脱敏验收摘要表、组件与文件路径清单及一键恢复卡片。仅在用户主动提出时，才进一步讨论 Orca、Paseo 等外部编排接入或多账号隔离方案。详见 [全链路验收、故障阻断测试与交付恢复卡](references/acceptance-and-delivery.md) 与 [接入后的长期维护](references/ongoing-maintenance.md)。

## 参考文档导航

| 文档 | 何时读取 |
|---|---|
| [信号模型与网络架构选择](references/signals-and-architecture.md) | 阶段 1 解释各层暴露面，对比模式 A / B / C 网络方案 |
| [环境盘点、备份与旧数据清理](references/preflight-and-cleanup.md) | 阶段 1–2 执行只读盘点、完整归档备份与旧环境清理 |
| [固定出口、Mihomo TUN 与直连白名单](references/egress-and-tun.md) | 阶段 3 配置固定节点、DNS 防泄漏、持久覆写规则与白名单 |
| [`sandbox-exec` 进程沙盒与受控启动链](references/seatbelt-and-launcher.md) | 阶段 5 部署 Seatbelt 出站策略、代理预检与统一 CLI 包装入口 |
| [独立指纹浏览器与本地时区/语言一致性](references/browser-and-locale.md) | 阶段 4 选择指纹浏览器方案，对齐时区、语言与定位权限 |
| [全链路验收、故障阻断测试与交付恢复卡](references/acceptance-and-delivery.md) | 阶段 6–7 运行内核阻断探针、停代理测试、输出脱敏摘要与恢复卡 |
| [接入后的长期维护](references/ongoing-maintenance.md) | 接入完成后日常巡检、升级核验、工具输出边界与多账号维护 |

## 官方资料入口

- [Anthropic：位置信息说明](https://privacy.claude.com/en/articles/11186740-does-claude-use-my-location)
- [Claude Code 安装文档](https://code.claude.com/docs/en/setup)
- [Claude Code 企业代理与网络配置](https://code.claude.com/docs/en/corporate-proxy)
- [Mihomo 官方文档](https://wiki.metacubex.one/)
