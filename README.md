# Claude Local Onboard

> 让 Codex 等 Agent 替你搭好 Claude Code 的本机环境：干净的工作身份、单一固定出口、网络与本地时间的一致性、可验收、可回退。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![macOS](https://img.shields.io/badge/platform-macOS-blue)
![Agent Skill](https://img.shields.io/badge/format-Agent%20Skill-brightgreen)

**目前只支持 macOS。** 流程依赖 Mihomo TUN、macOS 网络扩展授权与 CCBaga 独立指纹浏览器，Linux 与 Windows 用户在本仓库拿不到可用路径。

## 这是什么

一个面向 **Codex / Claude Agent** 的流程 Skill。仓库里没有现成脚本，实际配置由 Agent 在你确认后逐步执行：

- 旧环境盘点与完整备份
- CCBaga 独立指纹浏览器 + 工作身份隔离
- Mihomo TUN 固定出口 + 日常网络白名单
- 系统语言、时区、地区显示与定位权限核对
- 登录前网络 / DNS / IPv6 / WebRTC 验收
- Claude Code CLI 安装与统一启动入口
- 登录后验收、脱敏证据摘要与恢复说明

每个阶段都有验收门槛和回退方法：网关、DNS、路由改动前先记录原配置，节点与规则变更后重新核对实际出口。

## 适用边界

- 结果取决于 Anthropic 当时的资格判断与风控，本项目只处理本机环境
- 登录、验证码、付费与系统授权由你在本机亲自完成
- 所有二进制从官方来源获取；订阅与账号凭据由你保管

## 核心原则

1. 先备份，再清理，每一步可回退
2. 固定单一出口，代理故障时阻断未授权公网流量
3. 浏览器与 CLI 走同一条受控路径，减少旁路
4. 不指导伪造时区/位置，尊重真实所在地
5. 每阶段都有验收，不通过不进入下一步

## 快速开始

把下面这段直接发给你的 Agent（Codex、Claude Code 等）：

```
安装 skill：
https://github.com/lavapapa/claude-local-onboard

并用该 skill 帮我搭建起 Claude Code 使用环境。
```

Agent 会先问清使用方式、设备是否经常移动、当前网络与订阅、必须保留的访问（校园网、局域网、SSH、UU、Tailscale/tailcat、远程开发）、旧数据处理意愿，再按阶段执行。第 2 阶段的清理与 CLI 重装建议在 Codex 或独立终端里做，避免 Claude Code 在运行中重装自己。

## 前置条件

- macOS
- 可用的 Mihomo 订阅，或你能提供节点
- 愿意让 Agent 操作本机网络与 Claude 相关配置（会先说明影响并做备份）
- 一次网络切换窗口中不依赖这台 Mac 远程办公

## 仓库结构

- `SKILL.md` — 主流程：阶段划分、决策点、验收标准与证据要求
- `references/` — 按需读取的参考文档
  - `location-and-egress-signals.md` — 位置与出口信号分析
  - `preflight-and-data-boundary.md` — 目标盘点、备份与清理边界
  - `mihomo-tun-fixed-egress.md` — TUN 固定出口与白名单配置
  - `ccbaga-and-browser.md` — CCBaga 安装、身份与浏览器 profile
  - `macos-timezone-and-locale.md` — 系统语言、时区、地区与定位权限
  - `network-check-tools.md` — 出口、DNS、IP 质量与 WebRTC 检查工具
  - `login-and-cli-acceptance.md` — 登录前预验收与登录后 CLI 验收
  - `hardening-and-recovery.md` — 最小加固、恢复卡片与交付摘要
  - `dedicated-process-proxy.md` — 常用移动设备时的进程代理试用方案
  - `minimal-path.md` — 不上整机 TUN 的最小可用路径
- `.github/ISSUE_TEMPLATE/` — 问题与经验反馈模板
- `CONTRIBUTING.md` — 贡献方式
- `LICENSE`

想直接读流程就从 `SKILL.md` 开始；只关心某一步就跳到对应参考文档。

## 已知限制

- 只在 macOS + Mihomo TUN 路径上验证过，其他组合需要重新验收
- TUN 影响整机网络，节点切换（尤其跨国）应尽量少，切换后需重新核对出口
- 设备经常移动时，TUN 可能不是最优解，Skill 会提示专用进程代理作为次选
- 第三方 IP 情报与泄露检测站点只代表当次观测，不能当作平台内部规则
- 软件版本刻意不固定，安装当天的版本、来源与日期会记入交付摘要

## 使用反馈

遇到检验不通过、工具已失效或某一步在你机器上表现不同，欢迎按 Issue 模板提交，附上 macOS 版本、工具版本和脱敏后的观测结果。请勿粘贴订阅地址、令牌、Cookie、验证码或完整 IP。

## 许可

MIT License，详见 [LICENSE](LICENSE)。

---

**重要**：本项目聚焦“减少意外暴露 + 建立可检查的网络边界”，不鼓励、也不提供规避地区限制的方法。使用前请自行确认符合 Anthropic 的服务条款与当地法律。
