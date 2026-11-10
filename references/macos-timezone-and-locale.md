# 系统语言、时区与地区

## 这一层解决什么

出口 IP 决定服务端看到的网络来源，系统语言、时区、地区与定位权限决定设备本地呈现的环境信息。两类信号由不同配置产生：改时区不会改变出口 IP，换节点也不会改变系统读到的时区。接入时按用户实际工作的地区把两者调到一致，并记录改动前后的值。

时区设置会连带影响日历与提醒的显示、`cron`/`launchd` 的触发时刻、日志时间戳、备份时间与 Git 提交时间。执行前向用户说明这些影响，改动集中在一次可回退的操作里完成。

## 盘点当前状态

只读检查，不需要管理员权限：

```bash
readlink /etc/localtime                 # 当前时区对应的 zoneinfo 路径
date                                    # 当前时间、时区缩写与偏移
defaults read -g AppleLocale            # 地区设置
defaults read -g AppleLanguages         # 语言优先级
sudo systemsetup -gettimezone           # 时区名称（需管理员）
sudo systemsetup -getusingnetworktime   # 是否用网络时间源校时
```

图形界面路径：系统设置 → 通用 → 日期与时间。确认“根据当前位置自动设置时区”是否开启——开启时系统会按定位服务改写时区，手工设置的值可能被覆盖。

应用侧快速核对，可用来确认新设置确实被进程读到：

```bash
node -e 'console.log(Intl.DateTimeFormat().resolvedOptions().timeZone)'
```

## 设置时区

先记录原值，再决定目标时区。目标时区按用户实际工作地区确定，与固定出口所在地区保持一致；跨地区时区（如 `.5` 小时偏移）本身不是问题，关键是偏移与地区自洽。

停用自动时区后手工设置：

1. 系统设置 → 通用 → 日期与时间 → 时区，关闭“根据当前位置自动设置时区”。
2. 选择目标城市，或使用命令行：

```bash
sudo systemsetup -listtimezones | grep -i shanghai     # 确认可用名称
sudo systemsetup -settimezone Asia/Shanghai            # 写入目标时区
```

`systemsetup` 在部分 macOS 版本要求终端具备完全磁盘访问权限，或在系统设置中授权；命令输出被拒绝时，改用界面操作并记录原因。修改时区后无需重启，注销并重新登录可让所有常驻应用重新读取。

## 设置语言、地区与定位权限

- 语言与地区：系统设置 → 通用 → 语言与地区。调整语言后需注销重新登录，部分应用要重启。命令行 `defaults write -g AppleLanguages` 可用，但会与系统设置界面产生两份来源，优先用界面。
- 日期、数字与度量格式跟随“地区”；单独改格式只影响显示，不改变时区。
- 定位权限：系统设置 → 隐私与安全性 → 定位服务。只为确有需要的应用开启，逐一核对 Claude 相关应用、浏览器与 CCBaga 的授权状态。CCBaga 的身份 profile 内也有语言、时区与地区设置，优先在 profile 级完成，避免影响整机其他工作。

## 验证与回退

验证：

```bash
date
readlink /etc/localtime
sudo systemsetup -gettimezone
defaults read -g AppleLocale
```

再在 CCBaga 浏览器与 CLI 各确认一次：浏览器内 `Intl.DateTimeFormat().resolvedOptions().timeZone`、Claude 界面显示的时间、`date` 的输出三者一致，并与固定出口地区相符。

回退：把记录下来的原时区名写回 `sudo systemsetup -settimezone <原时区>`，重新开启自动时区（如原本开启），并重新登录。原值与修改时间一并写入交付摘要。

## 记录格式

在验收摘要中单列一行“本地环境”，写清时区名称、UTC 偏移、地区、语言、定位权限状态，以及是否与出口地区一致。只记录结论与观测方式，不记录与账号相关的信息。
