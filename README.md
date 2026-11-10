# Claude Local Onboard

一个面向 macOS 的 Codex / Claude Agent Skill，帮助用户逐步搭建 Claude Code 本机使用环境：备份并整理旧配置、安装和配置 CCBaga、设置 Mihomo 网络出口、完成登录前网络检查及 CLI 验收，并准备恢复说明。

## 核心目标

1. 给你清理出来一套“相对”彻底干净的环境
2. 防范 claude code 的各种已知检测手段 + 尽量减少“手滑”误操作的失误暴露ip的风险
3. 减少不必要的限制的前提下，让你的agent （比如codex）自动给你打造一套高度可用的运行环境。

## 用法

把下面这段提示词发给你的 Agent（例如 Codex）：

> 安装 skill：https://github.com/lavapapa/claude-local-onboard 并用该 skill 帮我搭建起 Claude Code 使用环境。

Skill 会引导 Agent 分阶段盘点现状、备份旧数据、配置环境、验收网络与 CLI，并在关键步骤说明影响和恢复方式。

## 许可

本项目采用 MIT License，详见 [LICENSE](LICENSE)。
