---
read_when:
    - 你需要一份适合初学者的 TUI 使用指南
    - 你需要完整的 TUI 功能、命令和快捷键列表
summary: 终端界面（TUI）：连接到 Gateway 网关，或以嵌入模式在本地运行
title: TUI
x-i18n:
    generated_at: "2026-07-26T06:33:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dc4dc5e2a408b5097b3615283b5a4590e8b55bccb15c26d8e38ab2c84b902f4a
    source_path: web/tui.md
    workflow: 16
---

## 快速开始

### Gateway 网关模式

1. 启动 Gateway 网关。

```bash
openclaw gateway
```

2. 打开 TUI。

```bash
openclaw tui
```

3. 输入消息并按 Enter 键。

远程 Gateway 网关：

```bash
openclaw tui --url ws://<host>:<port> --token <gateway-token>
```

如果 Gateway 网关使用密码身份验证，请使用 `--password`。

### 本地模式

不使用 Gateway 网关运行 TUI：

```bash
openclaw chat
# 或
openclaw tui --local
```

- `openclaw chat` 和 `openclaw terminal` 是 `openclaw tui --local` 的别名。
- `--local` 不能与 `--url`、`--token` 或 `--password` 结合使用。
- 本地模式直接使用嵌入式智能体运行时。大多数本地工具都可用，但仅限 Gateway 网关的功能不可用。
- 直接运行 `openclaw`（不带子命令）会自动选择目标：未配置的安装会运行推理新手引导；配置无效时会打开经典 Doctor 指引；如果已配置的 Gateway 网关可访问，则会以 Gateway 网关模式打开此 TUI shell；否则，如果已配置本地模型，则会以本地模式打开。

## 界面内容

- 页眉：连接 URL、当前智能体、当前会话。
- 聊天记录：用户消息、助手回复、系统通知、工具卡片。
- 状态行：连接/运行状态（正在连接、正在运行、正在流式传输、空闲、错误）。
- 页脚：智能体 + 会话 + 模型 + 目标状态 + 思考/快速/详细/追踪/推理 + token 计数 + 交付。
- 输入区：带自动补全功能的文本编辑器。

## 心智模型：智能体 + 会话

- 智能体由唯一的 slug 标识（例如 `main`、`research`）。Gateway 网关会提供该列表。
- 会话属于当前智能体。
- 会话键存储为 `agent:<agentId>:<sessionKey>`。
  - 如果输入 `/session main`，TUI 会将其展开为 `agent:<currentAgent>:main`。
  - 如果输入 `/session agent:other:main`，则会显式切换到该智能体会话。
- 会话范围：
  - `per-sender`（默认）：每个智能体拥有多个会话。
  - `global`：TUI 始终使用 `global` 会话（选择器可能为空）。
- 当前智能体和会话始终显示在页脚中。
- 如果会话具有[目标](/zh-CN/tools/goal)，页脚会显示其简要状态：
  `Pursuing goal`、`Goal paused (/goal resume)`、`Goal blocked (/goal resume)` 或 `Goal achieved`。
- 在不带 `--session` 启动时，如果上次所选会话仍然存在，Gateway 网关模式的 TUI 会恢复同一 Gateway 网关、智能体和会话范围下上次选择的会话。传入 `--session`、`/session`、`/new` 或 `/reset` 时仍会显式指定会话。

## 发送与交付

- 消息始终发送到 Gateway 网关（本地模式下则发送到嵌入式运行时）；将助手回复交付回聊天提供商是一个独立步骤，默认关闭。
- TUI 与 WebChat 一样属于内部来源界面，而不是通用出站渠道。对于要求使用 `tools.message` 才能显示回复的 harness，可以使用不带目标的 `message.send` 来完成当前 TUI 轮次；向显式提供商交付仍使用常规配置的渠道，并且绝不会回退到 `lastChannel`。
- 交付设置在启动时即固定用于整个 TUI 会话：使用 `openclaw tui --deliver` 启动即可开启。不存在可在会话期间切换此设置的 `/deliver` 斜杠命令或 Settings 开关；如需更改，请重启 TUI。

## 选择器与叠加面板

- 模型选择器：列出可用模型并设置会话覆盖值。
- 智能体选择器：选择其他智能体。
- 会话选择器：最多显示当前智能体过去 7 天内更新的 50 个会话。使用 `/session <key>` 可跳转到更早的已知会话。
- Settings（`/settings`）：切换工具输出展开状态和思考内容可见性。此面板不控制交付。

## 键盘快捷键

- Enter：发送消息
- Esc：中止正在进行的运行
- Ctrl+C：清除输入（按两次退出）
- Ctrl+D：退出
- Ctrl+L：模型选择器
- Ctrl+G：智能体选择器
- Ctrl+P：会话选择器
- Ctrl+O：切换工具输出展开状态
- Ctrl+T：切换思考内容可见性（重新加载历史记录）

## 斜杠命令

核心：

- `/help`
- `/status`（转发到 Gateway 网关；显示会话/模型摘要）
- `/gateway-status`（别名 `/gwstatus`；直接显示 Gateway 网关连接状态）
- `/agent <id>`（或 `/agents`）
- `/session <key>`（或 `/sessions`）
- `/model <provider/model>`（或 `/models`）

会话控制：

- `/think <off|minimal|low|medium|high>`（根据模型，较高层级可能会添加 `xhigh`/`max` 等级别）
- `/fast <status|auto|on|off>`
- `/verbose <on|full|off>`
- `/trace <on|off>`
- `/reasoning <on|off|stream>`
- `/usage <off|tokens|full|reset>`（`reset`/`inherit`/`clear`/`default` 会清除会话覆盖值）
- `/goal [status] | /goal start <objective> | /goal edit <objective> | /goal pause|resume|complete|block|clear`
- `/elevated <on|off|ask|full>`（别名：`/elev`）
- `/activation <mention|always>`
- `/queue <steer|followup|collect|interrupt> [debounce:<duration>] [cap:<n>] [drop:<summarize|old|new>]`
- `/queue default`（或 `/queue reset`）会清除会话覆盖值

会话生命周期：

- `/new`（使用新键生成全新的隔离会话；不会影响旧会话上的其他 TUI 客户端）
- `/reset`（原位重置当前会话键）
- `/abort`（中止正在进行的运行）
- `/settings`
- `/exit`（或 `/quit`）

仅限本地模式：

- `/auth [provider]` 会在 TUI 内打开提供商身份验证/登录流程。

本地模式在嵌入式运行时内实现相同的队列模式。运行过程中的
提示遵循会话的 `/queue` 策略：当运行时能够接受时，`steer` 会注入提示，
`followup` 会等待单独的轮次，`collect` 会合并待处理的提示，
而 `interrupt` 会先停止当前运行，再开始新的运行。
显式 `/steer <message>` 仅限 Gateway 网关使用；在本地模式下，请使用 `/queue steer` 并发送
普通消息。

OpenClaw：

- `/openclaw [request]` 从常规智能体 TUI 返回 [OpenClaw](#openclaw-setup-and-repair-helper) 设置/修复聊天，并可选择转发一个请求。

其他 Gateway 网关斜杠命令（例如 `/context`）会转发到 Gateway 网关，并显示为系统输出。请参阅[斜杠命令](/zh-CN/tools/slash-commands)。

## 本地 shell 命令

- 在行首添加 `!`，可在 TUI 主机上运行本地 shell 命令。
- TUI 每个会话会提示一次是否允许本地执行；拒绝后，`!` 将在该会话中保持禁用。
- 命令会在 TUI 工作目录中的全新非交互式 shell 内运行（不会持久保留 `cd`/环境）。
- 本地 shell 命令的环境中会包含 `OPENCLAW_SHELL=tui-local`。
- 单独的 `!` 会作为普通消息发送；前导空格不会触发本地执行。

## OpenClaw 设置和修复助手

OpenClaw 是最高权限层级的设置/修复助手，在已配置的默认模型通过实时推理检查后以 `openclaw setup` 提供。如果推理不可用，交互式调用会返回推理新手引导，自动化调用则会失败并给出修复指引。它在与 `openclaw tui --local` 相同的本地 TUI shell 内运行，底层由一个 AI 智能体提供支持，并且该智能体仅能执行 OpenClaw 类型化且需要审批的操作：

```bash
openclaw setup                       # 以交互方式启动
openclaw setup -m "status"           # 运行一个请求后退出
openclaw setup -m "set default model openai/gpt-5.2" --yes   # 应用配置写入
```

- 持久配置写入需要审批：可以交互式确认，也可以传入 `--yes`。
- `--json` 会以 JSON 格式输出启动概览，而不是开始聊天。
- 在 OpenClaw 内部，`open-tui` 请求（例如要求与常规智能体对话）会退出 OpenClaw 并打开常规智能体 TUI；可在那里使用 `/openclaw` 返回。

如果当前配置已通过验证，并且需要嵌入式智能体在同一台计算机上检查配置、与文档进行比较并协助修复偏差，而不依赖正在运行的 Gateway 网关，请使用本地模式。

如果 `openclaw config validate` 已经失败，请先运行 `openclaw configure` 或 `openclaw doctor --fix`；`openclaw chat` 仍然需要能够加载的配置才能启动。

典型流程：

1. 启动本地模式：

```bash
openclaw chat
```

2. 告诉智能体需要检查的内容，例如：

```text
将我的 Gateway 网关身份验证配置与文档进行比较，并建议最小改动的修复方案。
```

3. 使用本地 shell 命令获取确切证据并进行验证：

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

4. 使用 `openclaw config set` 或 `openclaw configure` 应用小范围更改，然后重新运行 `!openclaw config validate`。
5. 如果 Doctor 建议自动迁移或修复，请先审查，再运行 `!openclaw doctor --fix`。

提示：

- 优先使用 `openclaw config set` 或 `openclaw configure`，而不是手动编辑 `openclaw.json`。
- `openclaw docs "<query>"` 会从同一台计算机搜索实时文档索引。
- 需要结构化 schema 以及 SecretRef/可解析性错误时，`openclaw config validate --json` 很有用。

## 工具输出

- 工具调用会显示为包含参数和结果的卡片。
- Ctrl+O 可在折叠视图和展开视图之间切换。
- 工具运行时，部分更新会流式传输到同一张卡片中。

## 终端颜色

- TUI 会让助手正文使用终端的默认前景色，以确保深色和浅色终端均保持清晰可读。
- 如果终端使用浅色背景且自动检测不正确，请在启动 `openclaw tui` 前设置 `OPENCLAW_THEME=light`。
- 如果要强制使用原始深色配色，请设置 `OPENCLAW_THEME=dark`。

## 历史记录与流式传输

- 连接时，TUI 会加载最新历史记录（默认为 200 条消息）。
- 流式响应会原位更新，直至完成。
- TUI 还会监听智能体工具事件，以提供信息更丰富的工具卡片。

## 连接详情

- TUI 使用客户端 ID `openclaw-tui`，并以粗粒度的 `ui` 客户端模式连接（Control UI 和 WebChat 对 Gateway 网关策略使用相同模式）。
- 重新连接时会显示系统消息；事件缺口会在日志中显示。

## 选项

- `--local`：针对本地嵌入式智能体运行时运行
- `--url <url>`：Gateway 网关 WebSocket URL（默认为配置中的 `gateway.remote.url`，或环回地址上的 `ws://127.0.0.1:<port>`）
- `--token <token>`：Gateway 网关令牌（如需要）
- `--password <password>`：Gateway 网关密码（如需要）
- `--tls-fingerprint <sha256>`：固定 `wss://` Gateway 网关的预期 TLS 证书指纹
- `--session <key>`：会话键（默认值：`main`；范围为全局时使用 `global`）
- `--deliver`：将助手回复递送给提供商（默认关闭）
- `--thinking <level>`：覆盖发送时的思考级别
- `--message <text>`：连接后发送初始消息
- `--timeout-ms <ms>`：智能体超时时间（毫秒，默认为 `agents.defaults.timeoutSeconds`）
- `--history-limit <n>`：要加载的历史记录条目数（默认值为 `200`）

<Warning>
设置 `--url` 后，TUI 不会回退使用配置或环境凭据。请显式传递 `--token` 或 `--password`；如果目标使用固定证书，还需传递 `--tls-fingerprint`。缺少显式凭据会导致错误。在本地模式下，请勿传递 `--url`、`--token`、`--password` 或 `--tls-fingerprint`。
</Warning>

## 故障排查

发送消息后没有输出：

- 在 TUI 中运行 `/status`，确认 Gateway 网关已连接且处于空闲/忙碌状态。
- 检查 Gateway 网关日志：`openclaw logs --follow`。
- 确认智能体可以运行：`openclaw status` 和 `openclaw models status`。
- 如果希望消息出现在聊天渠道中，请确认启动 TUI 时使用了 `--deliver`（如未重启，之后无法启用此选项）。

## 连接故障排查

- `disconnected`：确保 Gateway 网关正在运行，并且你的 `--url/--token/--password` 正确。
- 选择器中没有智能体：检查 `openclaw agents list` 和你的路由配置。
- 会话选择器为空：你可能处于全局范围，或尚无任何会话。

## 相关内容

- [Control UI](/zh-CN/web/control-ui) — 基于 Web 的控制界面
- [配置](/zh-CN/cli/config) — 检查、验证和编辑 `openclaw.json`
- [Doctor](/zh-CN/cli/doctor) — 引导式修复和迁移检查
- [CLI 参考](/zh-CN/cli) — 完整的 CLI 命令参考
