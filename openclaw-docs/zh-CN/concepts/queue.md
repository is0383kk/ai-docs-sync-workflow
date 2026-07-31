---
read_when:
    - 更改自动回复执行方式或并发机制
    - 解释 `/queue` 模式或消息引导行为
summary: 自动回复队列模式、默认值和按会话覆盖设置
title: 命令队列
x-i18n:
    generated_at: "2026-07-26T06:46:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 69b40f67146226b0315492b27fc9d2218cace8bbd1eaff6514f7efb33b69d763
    source_path: concepts/queue.md
    workflow: 16
---

OpenClaw 通过一个小型进程内队列串行处理入站自动回复运行（所有渠道），防止多个智能体运行发生冲突，同时仍允许跨会话安全并行执行。

## 原因

- 自动回复运行的开销可能很大（LLM 调用），而且当多条入站消息在相近时间到达时，可能会发生冲突。
- 串行处理可避免争用共享资源（会话文件、日志、CLI 标准输入），并降低触发上游速率限制的可能性。

## 工作原理

- 支持分通道的 FIFO 队列会按可配置的并发上限处理每个通道（未配置的通道默认为 1；`main` 默认为 4，`subagent` 默认为 8）。
- `runEmbeddedAgent` 按**会话键**（通道 `session:<key>`）入队，以确保每个会话仅有一个活跃运行。
- 随后，每个会话运行都会进入一个**全局通道**（默认为 `main`），因此总体并行度受 `agents.defaults.maxConcurrent` 限制。
- 启用详细日志后，如果已入队的运行在启动前等待超过约 2 秒，系统会输出一条简短通知。
- 输入状态指示仍会在入队时立即触发（如果渠道支持），因此运行等待轮到自己期间，用户体验保持不变。

## 默认值

未设置时，所有入站渠道界面都使用：

- `mode: "steer"`
- `debounceMs: 500`
- `cap: 20`
- `drop: "summarize"`

默认使用同轮 Steering。运行过程中到达的提示词会在运行可接受 Steering 时注入活跃运行时，因此不会启动第二个会话运行。如果活跃运行无法接受 Steering，OpenClaw 会等待活跃运行结束后再开始处理该提示词。

## 队列模式

`/queue` 控制会话已有活跃运行时，普通入站消息的处理方式：

- `steer`：将消息注入活跃运行时。OpenClaw 会在**当前助手轮次完成其工具调用后**、下一次 LLM 调用前，传递所有待处理的 Steering 消息；Codex app-server 会收到一个批量的 `turn/steer`。如果运行未在主动流式传输，或 Steering 不可用，OpenClaw 会等待活跃运行结束后再开始处理该提示词。
- `followup`：不进行 Steering。将每条消息排入队列，待当前运行结束后，在后续智能体轮次中处理。
- `collect`：不进行 Steering。在静默窗口结束后，将已排队消息合并为**单个**后续轮次。如果消息面向不同的渠道/线程，则会分别处理以保留路由。
- `interrupt`：中止该会话的活跃运行，然后处理最新消息。

有关特定运行时的时序和依赖项行为，请参阅 [Steering queue](/zh-CN/concepts/queue-steering)。有关显式 `/steer <message>` 命令，请参阅 [Steer](/zh-CN/tools/steer)。

通过 `messages.queue` 进行全局或按渠道配置：

```json5
{
  messages: {
    queue: {
      mode: "steer",
      debounceMs: 500,
      cap: 20,
      drop: "summarize",
      byChannel: { discord: "collect" },
    },
  },
}
```

## 队列选项

选项适用于已排队的传递。`debounceMs` 还会在 `steer` 模式下设置 Codex Steering 静默窗口：

- `debounceMs`：处理已排队的后续消息或 collect 批次前的静默窗口；在 Codex `steer` 模式下，指发送批量 `turn/steer` 前的静默窗口。无单位数字按毫秒计算；`/queue` 选项接受单位 `ms`、`s`、`m`、`h` 和 `d`。
- `cap`：每个会话最多可排队的消息数。低于 `1` 的值会被忽略。
- `drop: "summarize"`（默认）：按需丢弃最早的队列条目，保留精简摘要，并将其作为合成的后续提示词注入。
- `drop: "old"`：按需丢弃最早的队列条目，不保留摘要。
- `drop: "new"`：队列已满时拒绝最新消息。

默认值：`debounceMs: 500`、`cap: 20`、`drop: summarize`。

## Steer 和流式传输

当渠道流式传输为 `partial` 或 `block` 时，在活跃运行到达运行时边界的过程中，Steering 可能表现为多条简短的可见回复：

- `partial`：预览可能提前结束，接受 Steering 后会开始新的预览。
- `block`：草稿大小的内容块可能产生同样的连续显示效果。
- 未使用流式传输时，如果运行时无法接受同轮 Steering，则会在活跃运行结束后回退为后续处理。

`steer` 不会中止正在执行的工具。当最新消息应中止当前运行时，请使用 `/queue interrupt`。

## 优先级

选择模式时，OpenClaw 按以下顺序解析：

1. 内联或已存储的每会话 `/queue` 覆盖项。
2. `messages.queue.byChannel.<channel>`。
3. `messages.queue.mode`。
4. 默认值 `steer`。

对于选项，内联或已存储的 `/queue` 选项优先于配置。随后按顺序应用渠道特定的防抖设置（`messages.queue.debounceMsByChannel`）、插件防抖默认值、全局 `messages.queue` 选项和内置默认值。`cap` 和 `drop` 是全局/会话选项，而不是按渠道配置的键。

## 每会话覆盖项

- 将 `/queue <steer|followup|collect|interrupt>` 作为独立命令发送，以存储当前会话的队列模式。
- 选项可以组合使用：`/queue collect debounce:0.5s cap:25 drop:summarize`
- `/queue default` 或 `/queue reset` 会清除会话覆盖项。

## 已排队轮次的取消

当提示词停留在 followup/collect 队列中时（例如另一个轮次处于活跃状态时，TUI 或
网页聊天的 `chat.send` 到达），Gateway 网关会为该客户端 `runId` 保留一个
**由 Gateway 网关拥有的取消标识**，直至已排队内容运行或被丢弃。该标识会随折叠到
溢出摘要中的内容一同保留。

- 携带特定 `runId` 的 `chat.abort` 会在该轮次仍处于
  队列中时将其取消，前提是请求者已获得授权（所有权规则与活跃运行相同）。
- 对于未提供 `runId` 的会话，`chat.abort` 会**先取消已授权的排队轮次，
  再中止已授权的活跃运行**。此顺序可防止队列处理将任务提升到仅停止了一半的会话中。
- 在不按请求者检查的情况下清除整个会话队列，并不是
  多所有者会话的停止路径。
- 对于 `sessions.list`，排队等待不会投影为活跃智能体运行，
  也不具有活跃运行的超时语义；只有活跃阶段具有这些语义。

由 Gateway 网关支持的客户端（包括 `openclaw tui`）会转发运行过程中的提示词，并
由 Gateway 网关应用队列模式。Esc/`/stop` 使用会话范围的中止，
因此即使本地句柄丢失，也不会让仍在队列中的提示词继续运行。

`openclaw chat` 和 `openclaw tui --local` 会在
嵌入式运行时中应用相同的四种模式。当运行时接受 Steering 时，本地 `steer` 会将内容注入活跃的嵌入式运行，
否则该内容会变为后续处理；`followup` 和
`collect` 仍作为本地待处理工作；`interrupt` 会先中止活跃的本地运行，
再开始处理最新消息。显式 `/steer <message>` 命令
不是本地模式命令。

## 范围和保证

- 适用于使用 Gateway 网关回复流水线的所有入站渠道中的自动回复智能体运行（WhatsApp 网页版、Telegram、Slack、Discord、Signal、iMessage、网页聊天等）。
- 默认通道（`main`）在进程范围内由入站处理和主 Heartbeat 共用；设置 `agents.defaults.maxConcurrent` 可允许多个会话并行运行。
- 还可能存在其他通道（例如 `cron`、`cron-nested`、`nested`、`subagent`），使后台任务能够并行运行，而不阻塞入站回复。隔离的定时任务智能体轮次会占用一个 `cron` 槽位，而其内部智能体执行使用 `cron-nested`。共享的非定时任务 `nested` 流程会保留各自的通道行为。这些分离运行会作为[后台任务](/zh-CN/automation/tasks)进行跟踪。
- 每会话通道可确保同一时间只有一个智能体运行操作给定会话。
- 没有外部依赖项或后台工作线程；仅使用 TypeScript + Promise。

## 故障排查

- 如果命令似乎卡住，请启用详细日志并查找“queued for ...ms”行，以确认队列正在处理。
- 对于已接受轮次但随后停止输出进度的 Codex app-server 运行，Codex 适配器会将其中断，使活跃会话通道能够释放，而不是等待外层运行超时。
- 启用诊断后，如果会话在超过内置警告阈值后仍处于 `processing`，并且未观察到回复、工具、状态、内容块或 ACP 进度，则会根据当前活动进行分类：
  - 有近期进度的活跃工作会记录为 `session.long_running`。拥有所有者的静默模型调用也会保持 `session.long_running` 状态，直至达到内置中止阈值，以免过早将速度较慢或非流式传输的提供商报告为停滞。
  - 没有近期进度的活跃工作会记录为 `session.stalled`；拥有所有者的模型调用、受阻的工具调用和停滞的嵌入式运行会在达到或超过中止阈值时切换为 `session.stalled`。没有所有者的过期模型/工具活动不会被隐藏为长时间运行。
  - `session.stuck` 专用于可恢复的过期会话记录，包括存在没有所有者的过期模型/工具活动的空闲排队会话。
  - `session.stuck` 始终会触发可释放受影响会话通道的恢复。在超过中止阈值后，`session.stalled` 分类（受阻的工具调用、停滞的模型调用或停滞的嵌入式运行）也可以触发活跃中止恢复，因此两种分类都可以解除队列卡滞，而不仅是 `session.stuck`。
  - 当会话保持不变时，重复的 `session.stuck` 和 `session.long_running` 警告日志行会按指数退避；无论此退避如何，每次 Heartbeat 周期仍会执行恢复尝试。

## 相关内容

- [会话管理](/zh-CN/concepts/session)
- [Steering queue](/zh-CN/concepts/queue-steering)
- [Steer](/zh-CN/tools/steer)
- [重试策略](/zh-CN/concepts/retry)
