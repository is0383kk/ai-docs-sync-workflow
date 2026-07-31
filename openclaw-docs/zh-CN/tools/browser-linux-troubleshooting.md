---
read_when: Browser control fails on Linux, especially with snap Chromium
summary: 修复 Linux 上 OpenClaw 浏览器控制的 Chrome/Brave/Edge/Chromium CDP 启动问题
title: 浏览器故障排查
x-i18n:
    generated_at: "2026-07-26T07:03:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5db2da2d43129862f0c005213df828f6eae81f5561e57d41795ea90787822a
    source_path: tools/browser-linux-troubleshooting.md
    workflow: 16
---

## 问题：无法在端口 18800 上启动 Chrome CDP

```json
{ "error": "错误：无法为配置文件 \"openclaw\" 在端口 18800 上启动 Chrome CDP。" }
```

### 根本原因

在 Ubuntu 和大多数 Linux 发行版上，`apt install chromium` 安装的是 snap
包装程序，而不是真正的浏览器：

```text
注意，选择 'chromium-browser' 而非 'chromium'
chromium-browser 已经是最新版本 (2:1snap1-0ubuntu2)。
```

Snap 的 AppArmor 限制会干扰 OpenClaw 启动和监控
浏览器进程的方式。

其他常见的 Linux 启动失败原因：

- `The profile appears to be in use by another Chromium process`：托管配置文件目录中存在过期的
  `Singleton*` 锁文件。当锁指向已终止或
  位于其他主机上的进程时，OpenClaw 会移除这些锁并重试一次。
- `Missing X server or $DISPLAY`：在没有桌面会话的主机上明确请求了可见浏览器。
  在 Linux 上，如果 `DISPLAY` 和 `WAYLAND_DISPLAY` 均未设置，
  本地托管配置文件会回退到无头模式。如果设置了 `OPENCLAW_BROWSER_HEADLESS=0`、`browser.headless: false`
  或 `browser.profiles.<name>.headless: false`，请移除该有头模式覆盖项，设置
  `OPENCLAW_BROWSER_HEADLESS=1`，启动 `Xvfb`，运行
  `openclaw browser start --headless` 以执行一次性托管启动，或在
  真正的桌面会话中运行 OpenClaw。

### 解决方案 1：安装 Google Chrome（推荐）

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install -y  # 如果出现依赖错误
```

更新 `~/.openclaw/openclaw.json`：

```json
{
  "browser": {
    "enabled": true,
    "executablePath": "/usr/bin/google-chrome-stable",
    "headless": true,
    "noSandbox": true
  }
}
```

### 解决方案 2：以仅附加模式使用 snap Chromium

如果必须保留 snap Chromium，请将 OpenClaw 配置为附加到
手动启动的浏览器，而不是由其启动浏览器：

```json
{
  "browser": {
    "enabled": true,
    "attachOnly": true,
    "headless": true,
    "noSandbox": true
  }
}
```

手动启动 Chromium：

```bash
chromium-browser --headless --no-sandbox --disable-gpu \
  --remote-debugging-port=18800 \
  --user-data-dir=$HOME/.openclaw/browser/openclaw/user-data \
  about:blank &
```

也可以选择通过 systemd 用户服务自动启动：

```ini
# ~/.config/systemd/user/openclaw-browser.service
[Unit]
Description=OpenClaw 浏览器 (Chrome CDP)
After=network.target

[Service]
ExecStart=/snap/bin/chromium --headless --no-sandbox --disable-gpu --remote-debugging-port=18800 --user-data-dir=%h/.openclaw/browser/openclaw/user-data about:blank
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now openclaw-browser.service
```

### 验证浏览器是否正常工作

```bash
curl -s http://127.0.0.1:18791/ | jq '{running, pid, chosenBrowser}'
curl -s -X POST http://127.0.0.1:18791/start
curl -s http://127.0.0.1:18791/tabs
```

### 配置参考

| 选项                      | 说明                                                          | 默认值                                                            |
| --------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `browser.enabled`           | 启用浏览器控制                                               | `true`                                                             |
| `browser.executablePath`    | 基于 Chromium 的浏览器二进制文件路径（Chrome/Brave/Edge/Chromium） | 自动检测（如果操作系统默认浏览器基于 Chromium，则优先使用它） |
| `browser.headless`          | 在无 GUI 的情况下运行                                                      | `false`                                                            |
| `OPENCLAW_BROWSER_HEADLESS` | 本地托管浏览器无头模式的按进程覆盖项         | 未设置                                                              |
| `browser.noSandbox`         | 添加 `--no-sandbox` 标志（某些 Linux 环境需要）               | `false`                                                            |
| `browser.attachOnly`        | 不启动浏览器；仅附加到现有浏览器              | `false`                                                            |

在 Raspberry Pi、较旧的 VPS 主机或速度较慢的存储设备上，如果 Chrome 公开其 CDP HTTP
端点或进入就绪状态所需的时间超过托管浏览器的截止时间，请使用通过
`attachOnly` 手动启动的浏览器。

### 问题：未找到 profile="user" 的 Chrome 标签页

你正在使用 `user`（`existing-session` / Chrome MCP）配置文件，但没有
可供附加的已打开标签页。

解决方法：

1. 改用托管浏览器：
   `openclaw browser --browser-profile openclaw start`（或设置
   `browser.defaultProfile: "openclaw"`）。
2. 保持本地 Chrome 运行并至少打开一个标签页，然后使用
   `--browser-profile user` 重试。

注意：

- `user` 仅适用于主机。在 Linux 服务器、容器或远程主机上，请优先使用
  CDP 配置文件。
- `user` 和其他 `existing-session` 配置文件具有当前 Chrome MCP
  的相同限制：仅支持由引用驱动的操作、每次上传一个文件、不支持对话框 `timeoutMs`
  覆盖项、不支持 `wait --load networkidle`，也不支持 `responsebody`、PDF 导出、
  下载拦截或批量操作。
- 本地 `openclaw` 驱动程序配置文件会自动分配 `cdpPort`/`cdpUrl`；仅为远程 CDP
  手动设置这些值。
- 远程 CDP 配置文件接受 `http://`、`https://`、`ws://` 和 `wss://`。
  使用 HTTP(S) 进行 `/json/version` 发现；当浏览器
  服务提供直接的 DevTools 套接字 URL 时，则使用 WS(S)。

## 相关内容

- [浏览器](/zh-CN/tools/browser)
- [浏览器登录](/zh-CN/tools/browser-login)
- [浏览器 WSL2 故障排除](/zh-CN/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
