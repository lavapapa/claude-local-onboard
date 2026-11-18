# 独立指纹浏览器与本地时区/语言一致性

## 1. 为什么需要独立指纹浏览器

Claude 网页登录、OAuth 授权与订阅账单都在浏览器内完成。日常主力浏览器通常积累了大量历史 Cookie、多地区登录态、扩展插件以及可能走直连 UDP 的 WebRTC/QUIC 配置。

将 Claude 网页操作隔离到一个**专用指纹浏览器环境（或完全独立的浏览器 Profile）**，目的有三点：
1. 与日常浏览器的 Cookie、缓存、账号状态彻底隔离；
2. 强制浏览器与 Claude Code CLI 走**同一个本地代理入口与固定出口节点**；
3. 在浏览器 Profile 内收敛 WebRTC、QUIC、语言、时区与地理位置权限，减少意外泄露本机网络或环境差异。

## 2. 浏览器方案选型（全系 Mac 通用）

不绑定单一软件，用户可根据芯片架构（Apple Silicon / Intel）与使用习惯任选其一：

| 方案 | 适用人群与优势 | 配置要点与注意事项 |
|---|---|---|
| **成熟指纹浏览器**（如 AdsPower、MoreLogin 等） | 同时支持 Apple Silicon 与 Intel Mac；Profile 级代理绑定、WebRTC 策略、时区与语言面板成熟，支持多身份隔离 | 代理类型设为本地 `HTTP`（如 `127.0.0.1:7890`，与 CLI 共用同一固定节点，不要单独配另一条节点）；WebRTC 设为禁用或仅代理转发；地理位置设为禁止或询问且不授权 |
| **CCBaga**（[liseami/CCBaga](https://github.com/liseami/CCBaga)） | 专为 Claude 设计的开源独立指纹浏览器 + 身份管理工具 | 1) 官方 Release 目前主要提供 macOS `arm64` 包（Intel Mac 需自行源码构建或改选其他指纹浏览器）；<br>2) 身份切换功能默认只操作 `~/.claude` 与 `~/.claude.json`（不感知自定义 `CLAUDE_CONFIG_DIR`），且切换时会终止运行中的 Claude 进程；<br>3) 若使用软链共享 Skills/Plugins，切勿取消勾选同步项以免移走软链；慎用一键清理钥匙串以免按前缀误删多账号凭据；<br>4) 其安装/重装功能若检测到自定义 `~/.local/bin/claude` 启动器可能跳过或覆盖，操作后需复查启动器 |
| **独立 Chromium / Chrome Profile**（零第三方依赖） | 不想安装指纹浏览器软件时的原生轻量方案，任何 Mac 均可直接运行 | 必须指定全新的 `--user-data-dir`，配合强制代理、禁用非代理 WebRTC UDP 与禁用 QUIC 参数启动（见下方示例），切勿混入日常 Chrome Profile |

### 原生独立 Chromium / Chrome 启动脚本参考（可选兜底）

若采用原生独立 Profile，可准备一个专用启动脚本（也可套在 `claude-network-run` 下运行）：

```bash
#!/bin/zsh
PROFILE_DIR="$HOME/.local/share/claude-browser-profile"
mkdir -p "$PROFILE_DIR"

TZ="America/Los_Angeles" LANG="en_US.UTF-8" \
open -na "Google Chrome" --args \
  --user-data-dir="$PROFILE_DIR" \
  --proxy-server="http://127.0.0.1:7890" \
  --host-resolver-rules="MAP * ~NOTFOUND , EXCLUDE 127.0.0.1 , EXCLUDE localhost" \
  --force-webrtc-ip-handling-policy=disable_non_proxied_udp \
  --enforce-webrtc-ip-permission-check \
  --disable-quic \
  --lang=en-US \
  --no-first-run
```

## 3. 时区、语言与定位权限对齐策略

公网出口 IP 所在的地区（服务端视角）与本机暴露的时区/语言（客户端视角）是两条独立信号：换代理节点不会自动改变系统时区，改时区也不会改变出口 IP。两者不一致（例如美西出口配东八区时区与中文环境）属于明显的可观察差异。

执行时先向用户说明两种对齐层级，由用户自主选择：

### 层级一：仅在工作环境内局部对齐（不影响整机日历与日常软件）

1. **指纹浏览器 Profile 层**：在 AdsPower / CCBaga 等 Profile 设置中开启“基于 IP 自动匹配时区/语言”，或手动指定与固定出口一致的时区与语言（如美西加利福尼亚出口设为 `America/Los_Angeles`、`en-US`）。
2. **Claude Code CLI 进程层**：在 `~/.local/bin/claude-network-run` 或生效的 `settings.json`（`env` 字段）中写入：
   ```json
   {
     "env": {
       "TZ": "America/Los_Angeles",
       "LANG": "en_US.UTF-8",
       "LC_ALL": "en_US.UTF-8"
     }
   }
   ```

### 层级二：连同 macOS 系统全局时区与语言一并对齐

若用户希望整机系统时区、语言与固定出口地区完全一致，先提醒这会同时影响系统日历、提醒事项、日志时间戳、定时任务触发时刻与 Git 提交时间，记录原值后再修改：

1. **先备份当前系统设置原值**：
   ```bash
   readlink /etc/localtime
   date
   defaults read -g AppleLocale
   defaults read -g AppleLanguages
   sudo systemsetup -gettimezone
   ```
2. **关闭自动时区并设置为目标出口时区（以美国加利福尼亚 `America/Los_Angeles` 为例）**：
   - 图形界面：`系统设置 → 通用 → 日期与时间`，关闭 **“根据当前位置自动设置时区”**（否则定位服务会把手工设置的时区改回去）。
   - 命令行写入：
     ```bash
     sudo systemsetup -listtimezones | grep -i los_angeles
     sudo systemsetup -settimezone America/Los_Angeles
     ```
     （部分 macOS 版本若提示 `systemsetup` 需要终端具备“完全磁盘访问权限”，可直接在系统设置界面将最近城市设为 `Los Angeles`。）
3. **语言与地区**：
   `系统设置 → 通用 → 语言与地区`，按需将首选语言设为 `English (US)`、地区设为 `United States`（修改后注销重新登录对全部常驻应用生效）。
4. **定位服务权限核查**：
   打开 `系统设置 → 隐私与安全性 → 定位服务`，确认指纹浏览器与终端未被授予不必要的物理定位权限；网页端保持拒绝 Geolocation 授权请求。

## 4. 一致性验证与回退

### 验证命令

```bash
# 核对 CLI 受控环境内读取到的时区与语言
claude-network-run /bin/sh -c 'date; echo "TZ=$TZ LANG=$LANG"'

# 如安装了 Node.js，核对运行时解析出的 IANA 时区名
claude-network-run node -e 'console.log(Intl.DateTimeFormat().resolvedOptions().timeZone)'
```

同时在独立指纹浏览器控制台运行 `Intl.DateTimeFormat().resolvedOptions().timeZone` 与 `navigator.languages`，确认浏览器与 CLI 时区、语言一致且与固定出口地区匹配。

### 回退方法

- **局部对齐回退**：移除 `claude-network-run` 或 `settings.json` 中的 `TZ`/`LANG` 字段即可。
- **系统全局回退**：执行 `sudo systemsetup -settimezone <备份的原时区名>`，并在 `系统设置 → 通用 → 日期与时间 / 语言与地区` 中恢复原自动时区开关与原语言顺序。
