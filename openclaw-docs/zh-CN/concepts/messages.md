---
read_when:
    - 说明入站消息如何转化为回复
    - 阐明会话、排队模式或流式传输行为
    - 记录推理可见性及其使用影响
summary: 消息流、会话、排队和推理可见性
title: 消息
x-i18n:
    generated_at: "2026-07-26T05:46:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e42bed834e9a57fb8a248c8654b75ea9977928582f68a83859cf6c16ed0b6bf5
    source_path: concepts/messages.md
    workflow: 16
---

入站消息依次经过路由、去重/防抖、智能体运行和出站投递：

```text
入站消息
  -> 路由/绑定 -> 会话键
  -> 去重 + 防抖
  -> 队列（如果已有运行处于活动状态）
  -> 智能体运行（流式传输 + 工具）
  -> 出站回复（渠道限制 + 分块）
```

主要配置项：

- `messages.*` 用于前缀、排队、入站防抖和群组行为。
- `agents.defaults.*` 用于分块流式传输、分块和静默回复默认值。
- 渠道覆盖项（`channels.telegram.*`、`channels.whatsapp.*` 等），用于设置各渠道的限制和流式传输开关。

完整模式请参阅[配置](/zh-CN/gateway/configuration)。

## 入站去重

渠道在重新连接后可能会再次投递同一条消息。OpenClaw 维护一个内存缓存，其键由智能体范围、渠道路由（渠道 + 对端 + 账户 + 线程）和消息 ID 组成，因此再次投递的消息不会触发第二次智能体运行。缓存条目会在 20 分钟后或跟踪条目数达到 5000 条时过期，以先发生者为准。

## 入站防抖

通过 `messages.inbound`，可将来自同一发送者的快速连续文本消息合并为一个智能体轮次。防抖范围限定为每个渠道 + 对话，并使用最新消息进行回复线程关联和 ID 设置。

```json5
{
  messages: {
    inbound: {
      debounceMs: 2000,
      byChannel: {
        discord: 1500,
        slack: 1500,
        whatsapp: 5000,
      },
    },
  },
}
```

- 防抖仅适用于纯文本消息；媒体/附件会立即触发刷新。
- 控制命令（停止/中止/状态等）会绕过防抖，以便立即分派。
- 默认禁用：`messages.inbound.debounceMs` 没有内置默认值，因此只有在你进行设置后（全局或按渠道），防抖才会启用。
- iMessage 遵循相同的通用防抖策略。`imsg` 0.13.1 及更高版本会在 OpenClaw 收到消息前合并 Apple URL 预览产生的拆分发送，因此无需 iMessage 专用的防抖设置。

## 会话和设备

会话归 Gateway 网关所有，而非客户端。

- 直接聊天会归并到智能体的主会话键。
- 群组/频道拥有各自的会话键。
- 会话存储和对话记录位于 Gateway 网关主机上。

多个设备/渠道可以映射到同一会话，但历史记录不会完整同步回每个客户端。对于长对话，请使用一个主要设备，以避免上下文出现分歧。Control UI 和 TUI 始终显示由 Gateway 网关支持的会话对话记录，因此它们是事实来源。

详情：[会话管理](/zh-CN/concepts/session)。

## 提示词正文和历史上下文

渠道插件会在入站上下文中填充多个文本字段，按优先级从高到低排列：

| 字段             | 用途                                                                                                     |
| ----------------- | ----------------------------------------------------------------------------------------------------------- |
| `BodyForAgent`    | 当前轮次面向模型的文本。未设置时回退到 `CommandBody` / `RawBody` / `Body`。        |
| `BodyForCommands` | 用于解析指令/命令的干净文本。未设置时回退到 `CommandBody` / `RawBody` / `Body`。 |
| `CommandBody`     | 旧版中间正文；优先使用 `BodyForCommands`。                                                         |
| `RawBody`         | `CommandBody` 的已弃用别名。                                                                         |
| `Body`            | 旧版提示词正文；可能包含渠道信封和历史记录包装。                                     |

当渠道提供历史记录时，会使用以下内容包装：

- `[Chat messages since your last reply - for context]`
- `[Current message - respond to this]`

对于非直接聊天（群组/频道/房间），当前消息正文会以发送者标签作为前缀，与历史记录条目使用的样式一致。指令剥离仅应用于当前消息部分，因此历史记录保持完整。包装历史记录的渠道应将 `BodyForCommands`（或旧版 `CommandBody` / `RawBody`）设置为原始消息文本，并将 `Body` 保留为组合提示词。

历史记录缓冲区仅包含待处理内容：它们包含未触发运行的群组消息（例如受提及门控的消息），并排除会话对话记录中已有的消息。在组装提示词时，结构化历史记录、回复、转发和渠道元数据会呈现为不受信任的用户角色上下文块。

使用 `messages.groupChat.historyLimit`（全局默认值）或 `channels.slack.historyLimit` 和 `channels.telegram.accounts.<id>.historyLimit` 等按渠道覆盖项配置历史记录大小（将 `0` 设置为禁用）。

## 工具结果元数据

工具结果的 `content` 是模型可见的结果；`details` 是用于 UI 呈现、诊断、媒体投递和插件的运行时元数据。

- 在提供商重放之前以及压缩输入之前，会移除 `toolResult.details`。
- 持久化的会话对话记录仅保留有界的 `details`；过大的元数据会被替换为标记了 `persistedDetailsTruncated: true` 的紧凑摘要。
- 插件和工具应将模型必须读取的文本放入 `content`，而不能只放入 `details`。

## 排队和后续消息

当已有运行处于活动状态时，入站消息默认会引导进入该运行。`messages.queue` 控制模式：

| 模式              | 行为                                            |
| ----------------- | --------------------------------------------------- |
| `steer`（默认） | 将新提示词注入活动运行。          |
| `followup`        | 在活动运行完成后运行该消息。      |
| `collect`         | 将兼容的消息合并到稍后的一个轮次中。      |
| `interrupt`       | 中止活动运行，然后启动最新提示词。 |

对于 Steer、后续消息和收集批处理，队列使用内置的 500ms 防抖。`messages.queue.cap` 默认为 20 条排队消息，`messages.queue.drop` 默认为 `summarize`（也可使用 `old` 和 `new`）。通过 `messages.queue.byChannel` 和 `messages.queue.debounceMsByChannel` 配置按渠道覆盖项。

详情：[命令队列](/zh-CN/concepts/queue)和[Steering queue](/zh-CN/concepts/queue-steering)。

## 渠道运行所有权

在消息进入会话队列之前，渠道插件可以保持顺序、对输入进行防抖，并施加传输背压。它们不应在智能体轮次本身外部施加单独的超时。一旦消息被路由到某个会话，会话、工具和运行时生命周期就会管理长时间运行的工作，从而让所有渠道以一致的方式报告慢速轮次并从中恢复。

## 流式传输、分块和批处理

分块流式传输会在模型生成文本块时发送部分回复；分块会遵守渠道文本限制，并避免拆分围栏代码。

- `agents.defaults.blockStreamingDefault`（`on|off`，默认 `off`）
- `agents.defaults.blockStreamingBreak`（`text_end|message_end`）
- `agents.defaults.blockStreamingChunk`（`minChars|maxChars|breakPreference`）
- `agents.defaults.blockStreamingCoalesce`（基于空闲时间的批处理）
- `agents.defaults.humanDelay`（分块回复之间类似人类的停顿）
- 渠道覆盖项：内置渠道上的 `*.streaming.block.enabled` 和 `*.streaming.block.coalesce`；过时的扁平键由 `openclaw doctor --fix` 迁移。除非明确启用，否则所有渠道（包括 Telegram）的分块流式传输均处于关闭状态。QQ Bot 是例外：它没有 `streaming.block` 键，并且会流式传输分块回复，除非 `channels.qqbot.streaming.mode` 为 `"off"`。

详情：[流式传输和分块](/zh-CN/concepts/streaming)。

## 推理可见性和 Token

- `/reasoning on|off|stream` 控制可见性。
- 当模型生成推理内容时，该内容仍计入 Token 用量。
- Telegram 支持将推理内容流式传输到临时草稿气泡中，该气泡会在最终投递后删除；如需持久化推理输出，请使用 `/reasoning on`。

详情：[思考 + 推理指令](/zh-CN/tools/thinking)和 [Token 用量](/zh-CN/reference/token-use)。

## 前缀、线程关联和回复

- 出站前缀位于 `channels.<channel>.responsePrefix` 和 `channels.<channel>.accounts.<id>.responsePrefix`。账户值优先。当这些规范字段未设置时，Doctor 会将全局回退值复制到已配置的渠道块中；`messages.responsePrefix` 仍作为隐式渠道和自定义渠道的回退值。
- 通过 `replyToMode` 和各渠道默认值实现回复线程关联。

详情：[配置](/zh-CN/gateway/config-agents#messages)和渠道文档。

## 静默回复

静默 Token `NO_REPLY`（不区分大小写，因此 `no_reply` 也匹配）表示“不要投递用户可见的回复”。当一个轮次还有待处理的工具媒体（例如生成的 TTS 音频）时，OpenClaw 会移除静默文本，但仍会投递媒体附件。

静默策略根据对话类型确定：

- 直接对话永远不会收到 `NO_REPLY` 提示词指导。如果直接运行意外返回了单独的静默 Token，OpenClaw 会将其抑制，而不是重写或投递。
- 群组/频道默认允许静默。在 `message_tool` 可见回复模式下，静默意味着模型不会调用 `message(action=send)`。
- 内部编排默认允许静默。

默认值位于 `agents.defaults.silentReply` 下；`surfaces.<id>.silentReply` 可以按界面覆盖群组/内部策略。

OpenClaw 还会对非直接聊天中的通用内部运行器故障使用静默回复，因此群组/频道不会看到 Gateway 网关错误样板文本。具有面向用户的恢复说明的分类故障（例如缺少身份验证、速率限制或过载通知）仍可投递。默认情况下，直接聊天会显示简短的故障说明；只有启用 `/verbose full` 时才会显示原始运行器详情。

单独的静默回复会在所有界面上被丢弃，因此父会话会保持安静，而不会将哨兵文本重写为回退闲聊。

## 相关内容

- [消息生命周期重构](/zh-CN/concepts/message-lifecycle-refactor) - 持久发送和接收设计目标
- [流式传输](/zh-CN/concepts/streaming) - 实时消息投递
- [重试](/zh-CN/concepts/retry) - 消息投递重试行为
- [队列](/zh-CN/concepts/queue) - 消息处理队列
- [渠道](/zh-CN/channels) - 消息平台集成
