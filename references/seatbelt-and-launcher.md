# `sandbox-exec` 进程沙盒与受控启动链

这是本 Skill 实现**防进程旁路**与**代理故障时严格阻断（fail-closed）**的核心层。

单靠环境变量或整机 TUN 无法防止两件事：一是子进程绕过代理变量发起原生 Socket / UDP 直连，二是代理客户端崩溃退出后 `utun` 网卡消失导致整机瞬间回落物理网卡直连。通过 macOS 内置的 Seatbelt（`/usr/bin/sandbox-exec`）约束 Claude Code 进程树，可以从内核态彻底封死非回环公网出站。

## 1. 工作原理

1. **默认拒绝主动出站**：`(deny network-outbound)` 拦截进程及所有子进程（`bash`、`curl`、`git`、`node`、`python`、MCP 服务等）向任意公网 IPv4、IPv6、TCP、UDP 发起的直连请求，内核直接返回 `EPERM (Operation not permitted)`。
2. **仅放行本机回环与本地 IPC**：`(allow network-outbound (remote ip "localhost:*") (remote unix-socket))` 允许连接本机代理监听端口（如 `127.0.0.1:7890`）、本地开发服务器、SSH Agent 与 Unix Socket。
3. **封禁代理控制面与敏感配置**：在回环放行之后追加 `(deny network-outbound (remote ip "localhost:9090"))`（按实际控制端口调整），并禁止沙盒内进程读写 `.sb` 策略文件与代理客户端配置目录，防止 Agent 被提示词诱导查询订阅节点或切换代理为 `DIRECT`。
4. **屏蔽本机 DNS 配置读取**：禁止访问 `com.apple.SystemConfiguration.DNSConfiguration`，防止沙盒内执行 `scutil --dns` 直接读出运营商本地 DNS 服务器地址（正常的经 `localhost:7890` 代理请求不受影响）。

## 2. 部署 Seatbelt 策略文件 `outbound.sb`

创建配置目录 `~/.config/claude-network/` 并写入策略文件（配合 `sandbox-exec -D HOME="$HOME"` 参数化主目录，无需硬编码用户名）：

```bash
mkdir -p "$HOME/.config/claude-network" "$HOME/.local/bin"
chmod 700 "$HOME/.config/claude-network"
```

`~/.config/claude-network/outbound.sb` 参考内容（将 `9090` 替换为实际 Mihomo 控制面端口）：

```scheme
(version 1)
(allow default)

; 1. 默认拒绝一切主动对外网络连接（覆盖 IPv4 / IPv6 / TCP / UDP）
(deny network-outbound)

; 2. 仅允许连接本机回环端口（本地代理、本地开发服务）与本地 Unix Socket
(allow network-outbound (remote ip "localhost:*") (remote unix-socket))

; 3. 封禁 Mihomo/Clash 本地管理控制端口，防止沙盒内读取或篡改代理策略
(deny network-outbound (remote ip "localhost:9090"))

; 4. 保护沙盒策略与启动脚本自身不被沙盒内进程篡改或删除
(deny file-write*
  (subpath (string-append (param "HOME") "/.config/claude-network"))
  (literal (string-append (param "HOME") "/.local/bin/claude"))
  (literal (string-append (param "HOME") "/.local/bin/claude-network-run")))

; 5. 阻止沙盒内进程读取常见代理客户端配置与订阅文件
(deny file-read* file-write*
  (subpath (string-append (param "HOME") "/.config/clash"))
  (subpath (string-append (param "HOME") "/.config/mihomo"))
  (subpath (string-append (param "HOME") "/Library/Application Support/com.follow.clash"))
  (subpath (string-append (param "HOME") "/Library/Application Support/io.github.clash-verge-rev.clash-verge-rev"))
  (subpath (string-append (param "HOME") "/Library/Application Support/mihomo-party")))

; 6. 阻止通过系统配置服务读取物理网络本地 DNS 解析器地址
(deny mach-lookup (global-name "com.apple.SystemConfiguration.DNSConfiguration"))
```

## 3. 部署通用沙盒执行器 `claude-network-run`

`~/.local/bin/claude-network-run` 负责注入统一代理变量、在进入目标程序前做一次 `CONNECT api.anthropic.com:443` 连通性预检，并在 `sandbox-exec` 约束下 `exec` 目标程序。代理不可用时立即停止并给出清晰恢复提示：

```bash
cat > "$HOME/.local/bin/claude-network-run" <<'EOF'
#!/usr/bin/env python3
import ctypes
import os
import socket
import sys
import time
from pathlib import Path

PROXY_PORT = int(os.environ.get("CLAUDE_PROXY_PORT", "7890"))
PROXY_URL = f"http://127.0.0.1:{PROXY_PORT}"
SB_PROFILE = str(Path.home() / ".config" / "claude-network" / "outbound.sb")


def already_sandboxed() -> bool:
    try:
        lib = ctypes.CDLL("/usr/lib/libsandbox.1.dylib")
        return lib.sandbox_check(os.getpid(), b"network-outbound", 0) == 1
    except Exception:
        return False


def check_proxy_ready() -> bool:
    req = b"CONNECT api.anthropic.com:443 HTTP/1.1\r\nHost: api.anthropic.com:443\r\n\r\n"
    for attempt in range(3):
        try:
            with socket.create_connection(("127.0.0.1", PROXY_PORT), timeout=2.5) as s:
                s.sendall(req)
                line = s.recv(256).split(b"\r\n", 1)[0]
                if line.startswith((b"HTTP/1.1 200", b"HTTP/1.0 200")):
                    return True
        except OSError:
            pass
        if attempt < 2:
            time.sleep(0.2)
    return False


def main() -> None:
    if len(sys.argv) < 2:
        sys.exit("用法: claude-network-run <命令> [参数...]")

    for k in ("ALL_PROXY", "all_proxy"):
        os.environ.pop(k, None)
    for k in ("HTTP_PROXY", "HTTPS_PROXY", "WS_PROXY", "WSS_PROXY"):
        os.environ[k] = os.environ[k.lower()] = PROXY_URL
    os.environ["NO_PROXY"] = os.environ["no_proxy"] = "localhost,127.0.0.1,::1"
    os.environ["NODE_USE_ENV_PROXY"] = "1"

    if not check_proxy_ready():
        sys.exit(
            f"[claude-network] 本地固定代理 (127.0.0.1:{PROXY_PORT}) 不可用或无法连通 api.anthropic.com:443，已阻止启动。\n"
            "请先启动代理客户端并确认固定出口节点正常后重试。"
        )

    target = sys.argv[1:]
    if already_sandboxed():
        os.execvp(target[0], target)

    cmd = [
        "/usr/bin/sandbox-exec",
        "-D", f"HOME={Path.home()}",
        "-f", SB_PROFILE,
        *target,
    ]
    os.execv(cmd[0], cmd)


if __name__ == "__main__":
    main()
EOF
chmod 755 "$HOME/.local/bin/claude-network-run"
```

> 如需在 CLI 层统一时区与语言（例如美西出口），可在上述脚本环境注入处同步设置 `os.environ.setdefault("TZ", "America/Los_Angeles")` 与 `os.environ.setdefault("LANG", "en_US.UTF-8")`。

## 4. 安装 Claude Code 并接管 `~/.local/bin/claude` 启动入口

1. **使用官方原生安装脚本安装最新稳定版**：
   ```bash
   curl -fsSL https://claude.ai/install.sh | bash
   ```
   官方原生安装器会将具体版本二进制存放在 `~/.local/share/claude/versions/<x.y.z>`，并在 `~/.local/bin/claude` 生成符号链接。确认无 Homebrew/npm 等多份残留争用 `PATH`（确保 `~/.local/bin` 位于 `PATH` 前列）。

2. **将 `~/.local/bin/claude` 替换为受保护启动包装器**：
   包装器动态选取 `~/.local/share/claude/versions/` 中语义化版本号最高的可执行文件，交由 `claude-network-run` 套上 `sandbox-exec` 启动：

   ```bash
   rm -f "$HOME/.local/bin/claude"
   cat > "$HOME/.local/bin/claude" <<'EOF'
   #!/usr/bin/env python3
   import os
   import re
   import sys
   from pathlib import Path

   versions_dir = Path.home() / ".local" / "share" / "claude" / "versions"
   candidates = [
       p for p in (versions_dir.iterdir() if versions_dir.is_dir() else [])
       if re.fullmatch(r"\d+\.\d+\.\d+", p.name) and p.is_file() and os.access(p, os.X_OK)
   ]
   if not candidates:
       sys.exit("未找到 Claude Code 版本二进制（~/.local/share/claude/versions/），请先运行官方安装器。")

   latest = str(max(candidates, key=lambda p: tuple(map(int, p.name.split(".")))))
   runner = str(Path.home() / ".local" / "bin" / "claude-network-run")
   os.execv(runner, [runner, latest, *sys.argv[1:]])
   EOF
   chmod 755 "$HOME/.local/bin/claude"
   ```

3. **为什么这样设计更稳**：
   - 终端、IDE（VS Code / Cursor / Zed）、SSH 或外部 Agent 编排器（如 Orca、Paseo）只要调用 `claude`，都会自动经过预检与 `sandbox-exec`；
   - Claude 在会话内再次调用 `claude` 子命令时，`already_sandboxed()` 通过 `libsandbox.1.dylib` 检测到内核已施加 `network-outbound` 限制，不会因重复套用 `sandbox-exec` 报错；
   - 日常自动更新在 `versions/` 下落盘新版本时会自动被选中；若未来手动重跑官方 `install.sh` 覆盖了 `~/.local/bin/claude`，按本节恢复包装器即可。

## 5. 快速自检与回退

部署完成后立即做一次无登录验证：

```bash
which -a claude
claude --version

# 验证沙盒内直连公网 IPv4 被内核 EPERM 拒绝（预期输出 BLOCKED_EPERM）
claude-network-run /usr/bin/python3 -c '
import errno, socket, sys
s = socket.socket()
s.settimeout(2)
try:
    s.connect(("1.1.1.1", 443))
    sys.exit("LEAK_CONNECTED")
except OSError as e:
    print("BLOCKED_EPERM" if e.errno == errno.EPERM else f"OTHER_ERR:{e}")
'
```

**回退方法**：若需恢复无包装的原生入口，将 `~/.local/bin/claude` 重新软链回 `~/.local/share/claude/versions/<最新版本>`，并保留或移除 `~/.config/claude-network/` 与 `~/.local/bin/claude-network-run`。
