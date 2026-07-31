---
read_when:
    - 打包 OpenClaw.app
    - 调试 macOS Gateway 网关 launchd 服务
    - 安装适用于 macOS 的 Gateway 网关 CLI
summary: macOS 上的 Gateway 网关运行时（外部 launchd 服务）
title: macOS 上的 Gateway 网关
x-i18n:
    generated_at: "2026-07-26T06:19:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 30c1ae14d8f8eaab73d0e2b725292d7411c2c8b5e0e0c32ad13989c01340d054
    source_path: platforms/mac/bundled-gateway.md
    workflow: 16
---

OpenClaw.app 不内置 Node 或 Gateway 网关运行时。macOS 应用
要求在外部安装 `openclaw` CLI，不会将 Gateway 网关作为
子进程启动，而是管理每用户的 launchd 服务，以保持 Gateway 网关
运行（或连接到已在本地运行的 Gateway 网关）。

## 自动设置

在新 Mac 上进行新手引导时，选择 **This Mac**。应用会在 Gateway 网关向导之前运行其
已签名的内置安装程序脚本：该脚本会在 `~/.openclaw` 下安装
用户空间 Node 运行时及匹配的 `openclaw` CLI，
然后安装并启动每用户的 launchd 服务。此流程无需
Terminal、Homebrew 或管理员权限。

应用仅内置安装程序脚本，不内置 Node 或 Gateway 网关载荷；
设置时需要互联网连接，以下载运行时和匹配的
OpenClaw 软件包。

## 手动恢复

手动安装推荐使用 Node 24.15+；Node 22.22.3+ 也可以使用。全局安装
`openclaw`：

```bash
npm install -g openclaw@<version>
```

自动设置失败后，使用 **Retry setup**。如果仍然失败，
请使用上述命令手动安装 CLI，然后在新手引导中选择 **Check again**。

## Launchd（将 Gateway 网关作为 LaunchAgent）

标签：`ai.openclaw.gateway`（默认配置文件），或命名配置文件使用
`ai.openclaw.<profile>`。

Plist 位置（每用户）：`~/Library/LaunchAgents/ai.openclaw.gateway.plist`
（或 `ai.openclaw.<profile>.plist`）。

在 Local 模式下，macOS 应用负责为默认配置文件安装和更新 LaunchAgent。
CLI 也可以直接安装它：`openclaw gateway install`
（通过 `OPENCLAW_PROFILE` 环境变量选择命名配置文件）。

行为：

- “OpenClaw Active”用于启用或禁用 LaunchAgent。
- 退出应用**不会**停止 Gateway 网关（launchd 会使其保持运行）。
- 如果配置的端口上已有 Gateway 网关在运行，应用会连接到该网关，
  而不是启动新的网关。

日志：

- launchd 标准输出：`~/Library/Logs/openclaw/gateway.log`（配置文件使用
  `gateway-<profile>.log`）
- launchd 标准错误：已抑制
- 如果主机因重复出现 `EADDRINUSE` 或快速重启而陷入循环，请检查
  是否存在重复的 `ai.openclaw.gateway` / `ai.openclaw.node` LaunchAgent，以及
  [Gateway 网关故障排除](/zh-CN/gateway/troubleshooting#macos-launchd-supervisor-loop-with-duplicate-gatewaynode-launchagents)
  中的 launchd 标记解决方法。

## 版本兼容性

macOS 应用会将 Gateway 网关版本与自身版本进行核对。现有 CLI 缺失或
不兼容时，新手引导会自动运行托管设置。使用 **Retry setup** 可重新安装，
修复外部 CLI 后则使用 **Check again**。

## macOS 上的状态目录

将 OpenClaw 状态保存在本地且不同步的磁盘上。避免使用 iCloud Drive 和其他
云同步文件夹；同步延迟和文件锁可能会影响会话、
凭据和 Gateway 网关状态。

仅在需要覆盖默认设置时，将 `OPENCLAW_STATE_DIR` 设置为本地路径。
`openclaw doctor` 会针对常见的云同步状态路径发出警告，并建议
迁回本地存储。请参阅
[环境变量](/zh-CN/help/environment#path-related-env-vars)和
[Doctor](/zh-CN/gateway/doctor)。

## 调试应用连接

在源码检出目录中使用 macOS 调试 CLI，以运行与应用所用相同的 Gateway 网关
WebSocket 握手和设备发现逻辑：

```bash
cd apps/macos
swift run openclaw-mac connect --json
swift run openclaw-mac discover --timeout 3000 --json
```

`connect` 接受 `--url`、`--token`、`--timeout`、`--probe` 和 `--json`
（以及客户端身份覆盖选项；使用 `--help` 运行可查看完整列表）。
`discover` 接受 `--timeout`、`--json` 和 `--include-local`。需要
区分 CLI 设备发现问题与应用端连接问题时，请将设备发现输出与
`openclaw gateway discover --json` 进行比较。

## 冒烟检查

```bash
openclaw --version

OPENCLAW_SKIP_CHANNELS=1 \
OPENCLAW_SKIP_CANVAS_HOST=1 \
openclaw gateway --port 18999 --bind loopback
```

然后：

```bash
openclaw gateway call health --url ws://127.0.0.1:18999 --timeout 3000
```

## 相关内容

- [macOS 应用](/zh-CN/platforms/macos)
- [Gateway 网关运行手册](/zh-CN/gateway)
