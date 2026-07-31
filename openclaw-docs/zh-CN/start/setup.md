---
read_when:
    - 设置新机器
    - 你希望在不破坏个人设置的前提下使用“最新、最强”的版本
summary: OpenClaw 高级设置和开发工作流
title: 设置
x-i18n:
    generated_at: "2026-07-26T06:24:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c40d6d2bf2814465f3cc49c65d4c1498671420af728ce8012d13af3fba67025a
    source_path: start/setup.md
    workflow: 16
---

<Note>
如果你是首次设置，请从[入门指南](/zh-CN/start/getting-started)开始。
有关新手引导的详细信息，请参阅[新手引导（CLI）](/zh-CN/start/wizard)。
</Note>

## 简而言之

根据你希望更新的频率以及是否想自行运行 Gateway 网关，选择设置工作流：

- **定制内容放在仓库之外：** 将你的配置和工作区保存在 `~/.openclaw/openclaw.json` 和 `~/.openclaw/workspace/` 中，这样仓库更新就不会触及它们。
- **稳定工作流（推荐大多数用户使用）：** 安装 macOS 应用，并让它运行内置的 Gateway 网关。
- **前沿工作流（开发）：** 通过 `pnpm gateway:watch` 自行运行 Gateway 网关，然后让 macOS 应用以 Local 模式连接。

## 前置要求（从源码运行）

- 推荐 Node 24.15+（仍支持 Node 22 LTS，当前为 `22.22.3+`）
- 从源码检出时需要 `pnpm`。在开发模式下，OpenClaw 从
  `extensions/*` pnpm 工作区软件包加载内置插件，因此根目录的 `npm install`
  不会准备完整的源码树。
- Docker（可选；仅用于容器化设置/E2E——参阅 [Docker](/zh-CN/install/docker)）

## 定制策略（避免更新造成破坏）

如果你既希望“100% 为我量身定制”，又希望轻松更新，请将自定义内容保存在：

- **配置：** `~/.openclaw/openclaw.json`（类似 JSON/JSON5）
- **工作区：** `~/.openclaw/workspace`（Skills、提示词、记忆；建议设为私有 Git 仓库）

只需初始化一次配置和工作区文件夹，无需运行完整的新手引导向导：

```bash
openclaw setup --baseline
```

还没有全局安装？改为从此仓库运行：

```bash
pnpm openclaw setup --baseline
```

（不带 `--baseline` 的单独 `openclaw setup` 是 `openclaw onboard` 的别名，会运行完整的交互式向导。）

## 从此仓库运行 Gateway 网关

完成 `pnpm build` 后，可以直接运行打包的 CLI：

```bash
node openclaw.mjs gateway --port 18789 --verbose
```

## 稳定工作流（优先使用 macOS 应用）

1. 安装并启动 **OpenClaw.app**（菜单栏）。
2. 完成新手引导/权限检查清单（TCC 提示）。
3. 确保 Gateway 网关为 **Local** 且正在运行（由应用管理）。
4. 连接渠道（例如 WhatsApp）：

```bash
openclaw channels login
```

5. 完整性检查：

```bash
openclaw health
```

如果你的构建版本不提供新手引导：

- 运行 `openclaw setup`，然后运行 `openclaw channels login`，再手动启动 Gateway 网关（`openclaw gateway`）。

## 前沿工作流（在终端中运行 Gateway 网关）

目标：开发 TypeScript Gateway 网关、使用热重载，并保持 macOS 应用 UI 与之连接。

### 0)（可选）也从源码运行 macOS 应用

如果你还希望 macOS 应用使用前沿版本：

```bash
./scripts/restart-mac.sh
```

### 1) 启动开发版 Gateway 网关

```bash
pnpm install
# 仅首次运行时需要（或重置本地 OpenClaw 配置/工作区后）
pnpm openclaw setup
pnpm gateway:watch
```

`gateway:watch` 会在一个命名的 tmux 会话
（`openclaw-gateway-watch-main`）中启动或重启 Gateway 网关监视进程，并从交互式
终端自动连接。非交互式 shell 会保持分离状态并输出
`tmux attach -t openclaw-gateway-watch-main`；使用
`OPENCLAW_GATEWAY_WATCH_ATTACH=0 pnpm gateway:watch` 可让交互式运行保持分离状态，
或使用 `pnpm gateway:watch:raw` 进入前台监视模式。监视器在接管活动配置文件中
已配置的端口或默认端口之前，会停止该配置文件已安装的 Gateway 网关服务，
防止服务管理器替换源码进程。该服务仍保持安装状态；监视结束后运行
`pnpm openclaw gateway start`。启动失败后 tmux 窗格仍然可用，
因此其他终端或智能体可以连接或捕获其日志。监视器会在相关源码、配置和
内置插件元数据发生变化时重新加载。如果受监视的 Gateway 网关在启动期间退出，
`gateway:watch` 会运行一次
`openclaw doctor --fix --non-interactive` 并重试；设置
`OPENCLAW_GATEWAY_WATCH_AUTO_DOCTOR=0` 可禁用这一仅限开发环境的修复流程。
`pnpm gateway:watch` 不会重新构建 `dist/control-ui`，因此在 `ui/` 发生变化后应重新运行 `pnpm ui:build`，或在开发 Control UI 时使用 `pnpm ui:dev`。

### 2) 将 macOS 应用指向正在运行的 Gateway 网关

在 **OpenClaw.app** 中：

- Connection Mode: **Local**
  应用将连接到已配置端口上正在运行的 Gateway 网关。

### 3) 验证

- 应用内的 Gateway 网关状态应显示 **"Using existing gateway …"**
- 或者通过 CLI：

```bash
openclaw health
```

### 常见陷阱

- **端口错误：** Gateway 网关 WebSocket 默认使用 `ws://127.0.0.1:18789`；确保应用和 CLI 使用相同端口。
- **状态存储位置：**
  - 渠道/提供商状态：`~/.openclaw/credentials/`
  - 模型身份验证配置文件：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
  - 会话和记录：`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
  - 旧版/归档会话工件：`~/.openclaw/agents/<agentId>/sessions/`
  - 日志：`/tmp/openclaw/`

## 凭据存储映射

调试身份验证或决定要备份哪些内容时，请使用此映射：

- **WhatsApp**：`~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
- **Telegram Bot 令牌**：配置/环境变量或 `channels.telegram.tokenFile`（仅限普通文件；拒绝符号链接）
- **Discord Bot 令牌**：配置/环境变量或 SecretRef（环境变量/文件/exec 提供商）
- **Slack 令牌**：配置/环境变量（`channels.slack.*`）
- **配对允许列表**：
  - `~/.openclaw/credentials/<channel>-allowFrom.json`（默认账户）
  - `~/.openclaw/credentials/<channel>-<accountId>-allowFrom.json`（非默认账户）
- **模型身份验证配置文件**：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- **基于文件的机密载荷（可选）**：`~/.openclaw/secrets.json`
- **旧版 OAuth 导入**：`~/.openclaw/credentials/oauth.json`
  更多详情：[安全](/zh-CN/gateway/security#credential-storage-map)。

## 更新（不破坏你的设置）

- 将 `~/.openclaw/workspace` 和 `~/.openclaw/` 作为“你的内容”保存；不要将个人提示词/配置放入 `openclaw` 仓库。
- 更新源码：`git pull` + `pnpm install` + 继续使用 `pnpm gateway:watch`。

## Linux（systemd 用户服务）

Linux 安装使用 systemd **用户**服务。默认情况下，systemd 会在用户
注销或空闲时停止用户服务，从而终止 Gateway 网关。新手引导会尝试为你启用
持久驻留（可能提示使用 sudo）。如果仍未启用，请运行：

```bash
sudo loginctl enable-linger $USER
```

对于始终在线或多用户服务器，请考虑使用**系统**服务，而不是
用户服务（无需持久驻留）。有关 systemd 的说明，请参阅 [Gateway 网关运行手册](/zh-CN/gateway)。

## 相关文档

- [Gateway 网关运行手册](/zh-CN/gateway)（标志、监管、端口）
- [Gateway 配置](/zh-CN/gateway/configuration)（配置架构 + 示例）
- [Discord](/zh-CN/channels/discord) 和 [Telegram](/zh-CN/channels/telegram)（回复标签 + replyToMode 设置）
- [OpenClaw 助手设置](/zh-CN/start/openclaw)
- [macOS 应用](/zh-CN/platforms/macos)（Gateway 网关生命周期）
