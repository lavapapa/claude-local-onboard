# 环境盘点、备份与旧数据清理

## 1. 阶段 1 只读盘点清单

动手修改任何配置前，先向用户确认：

1. **使用形态**：日常在固定桌面 Mac 还是移动笔记本使用；期望固定的出口国家/地区（须在 Anthropic 官方支持列表内，例如美西加利福尼亚）。
2. **日常直连需求**：哪些访问绝不能被代理阻断或绕远（如局域网 NAS/打印机、内网/校园网域名、SSH 跳板、Tailscale/ZeroTier/WireGuard 等 Mesh VPN、UU 远程/RustDesk 等远程桌面）。
3. **本地环境偏好**：希望仅在指纹浏览器 Profile 与 CLI 环境变量层对齐目标出口时区/语言，还是连同 macOS 系统全局时区一并修改。
4. **旧数据处理意愿**：完整归档备份后清理（默认推荐，新环境不导入旧会话），还是保留特定非认证配置（如个人 `CLAUDE.md`、自定义 Skills）。

随后执行无副作用的只读盘点，输出**脱敏摘要**（遮蔽完整公网 IP、邮箱、Token、订阅链接）：

```bash
sw_vers                                         # macOS 版本
uname -m                                        # 芯片架构（arm64 或 x86_64，本流程均支持）
which -a claude 2>/dev/null                     # 现有 CLI 入口
ls -la ~/.local/share/claude/versions 2>/dev/null
pgrep -fl -i "claude|clash|mihomo|flclash|verge|adspower|ccbaga" 2>/dev/null
networksetup -listallnetworkservices
scutil --proxy
readlink /etc/localtime && date                 # 当前时区与偏移
defaults read -g AppleLocale 2>/dev/null
defaults read -g AppleLanguages 2>/dev/null
```

同时记录当天拟安装或复用工具的名称、版本号与下载来源。

## 2. 执行者自检（防自杀中断）

进入清理或重装 CLI 前，必须确认**当前正在执行任务的 Agent 是谁**：

- 如果当前 Agent 正运行在待清理的本机 Claude Code 会话中，直接删除 `~/.claude`、注销凭据或替换 CLI 入口会导致当前会话中断。此时应先生成完整备份与清理命令，交由外部 Agent（如 Codex）或用户在独立终端中执行。
- 检查是否有其他未保存工作的 Claude Code 终端或桌面进程，说明清理与切换可能结束的进程范围，由用户确认执行窗口。

## 3. Claude Code 本机数据与凭据路径全表

不要仅靠删除 `claude` 二进制来“清理环境”。在 macOS 上逐项核查以下路径：

| 类别 | 常见路径 / 存储位置 | 包含内容与风险说明 |
|---|---|---|
| 默认主配置与会话目录 | `~/.claude/` | `settings.json`、`projects/`、`sessions/`、`history.jsonl`、插件与缓存、旧认证残留 |
| 全局状态文件 | `~/.claude.json`（及 `~/.claude.json.backup*`） | 账号标识、设备/首次引导状态、最近项目记录、MCP 与特性开关 |
| 自定义配置目录 | 环境变量 `CLAUDE_CONFIG_DIR` 指向的目录，或 `~/.config/claude*` | 多账号或外部编排器托管的独立配置与状态 |
| 原生安装器文件 | `~/.local/bin/claude`、`~/.local/share/claude/`、`~/.local/state/claude/` | 启动入口与历史版本二进制、运行态日志 |
| Homebrew / npm 残留 | `/opt/homebrew/bin/claude`、`/usr/local/bin/claude`、全局 `node_modules/@anthropic-ai/claude-code` | 多渠道混装容易导致 `PATH` 命中未受沙盒保护的旧入口 |
| macOS 应用支持与缓存 | `~/Library/Application Support/Claude/`、`~/Library/Caches/claude*`、`~/Library/Preferences/com.anthropic.*` | 桌面端或辅助组件留下的本地状态与缓存 |
| macOS 钥匙串（Keychain） | 登录钥匙串中名称匹配 `Claude Code-credentials*` 或 `Claude Safe Storage` 的条目 | OAuth 令牌与加密凭据（多账号时可能带哈希后缀） |
| Shell 启动脚本 | `~/.zshrc`、`~/.zprofile`、`~/.bashrc` | 旧的 `ANTHROPIC_*`、`CLAUDE_*`、代理变量或 alias |

区分三类对象：
1. **活动配置与状态**：备份后从原位置移走或清除，新环境从干净状态重新生成；
2. **需要保护的独立资产**：用户现有的指纹浏览器其他 Profile、代理客户端订阅配置、个人通用开发密钥，严禁误删；
3. **可丢弃缓存**：编译缓存与临时日志，无需迁回。

## 4. 备份与清理标准步骤

1. **建立受限权限备份目录并归档**（备份位置不得放在 `~/.claude` 内）：
   ```bash
   BACKUP_DIR="$HOME/.local/backups/claude-onboard-$(date +%Y%m%d-%H%M%S)"
   mkdir -p "$BACKUP_DIR" && chmod 700 "$BACKUP_DIR"

   for p in "$HOME/.claude" "$HOME/.claude.json" "$HOME/.local/share/claude" "$HOME/.local/state/claude"; do
     [ -e "$p" ] && cp -a "$p" "$BACKUP_DIR/"
   done
   ```
2. **校验备份可读且完整**：
   ```bash
   ls -la "$BACKUP_DIR"
   ```
   确认关键文件体积与权限正常后，再执行任何删除动作。
3. **核查钥匙串凭据名称（不回显密文）**：
   ```bash
   security dump-keychain | grep -E '"svce"<blob>="Claude' || true
   ```
   若用户确认清除旧登录态，按查到的精确服务名由用户确认后删除（如 `security delete-generic-password -s "Claude Code-credentials"`），避免盲目按宽泛前缀删掉用户仍需保留的其他账号凭据。
4. **按确认范围清理原路径**：
   移除旧的 `~/.claude`、`~/.claude.json` 及多余安装渠道残留（保留单一官方安装方式），清理 Shell 启动文件里冲突的旧 `ANTHROPIC_BASE_URL` / `ANTHROPIC_API_KEY` 等遗留变量（除非用户明确使用第三方网关，但那不属于官方账号接入场景）。
5. **需要保留的非会话资产审阅迁回**：
   新环境初始化后，仅按需拷回纯文本指令或个人技能（如审阅过的 `CLAUDE.md`），不迁回 `history.jsonl`、`projects/`、`.claude.json` 或未知 Hook。

## 5. 可选第三方清理工具审阅原则

如果用户希望参考开源清理脚本（例如 [claude-env-cleanup](https://github.com/pnedfff/claude-env-cleanup)）：

- 仅将其视为辅助对照清单，执行前必须先完成第 4 步的独立备份；
- 审阅当前版本源码：确认其扫描路径、是否带 `--dry-run` 预览、是否会触碰浏览器其他 Profile 或钥匙串、日志是否打印敏感字段；
- 忽略任何第三方仓库中关于“养号”“规避风控”的主观说法，只复用其目录清理能力。
