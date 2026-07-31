---
read_when:
    - 你正在构建或重构消息渠道插件的接收路径
    - 你需要共享的入站上下文构建、会话记录或准备好的回复分发
    - 你正在将旧的渠道轮次辅助函数迁移到入口/消息 API
summary: 渠道插件的入站事件辅助函数：上下文构建、共享运行器编排、会话记录和预备回复分发
title: 渠道入站 API
x-i18n:
    generated_at: "2026-07-26T05:57:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 854408ca42cfe1e1b48e4fd223b176f438e1db28deb9a5aa33eea8238127d9df
    source_path: plugins/sdk-channel-inbound.md
    workflow: 16
---

渠道接收路径遵循同一流程：

```text
平台事件 -> 入站事实/上下文 -> 智能体回复 -> 消息投递
```

使用 `openclaw/plugin-sdk/channel-inbound` 进行入站事件规范化、
格式化、根目录处理和编排。使用
`openclaw/plugin-sdk/channel-outbound` 处理原生发送、回执、持久化
投递和实时预览行为。

## 核心辅助函数

```ts
import {
  buildChannelInboundEventContext,
  runChannelInboundEvent,
  dispatchChannelInboundReply,
} from "openclaw/plugin-sdk/channel-inbound";
```

- `buildChannelInboundEventContext(...)`：将规范化的渠道事实
  投射到提示词/会话上下文中。通过 `channelContext` 传递渠道所有的发送者/聊天元数据，
  插件钩子会将其视为 `ctx.channelContext`。
  从此子路径扩展 `PluginHookChannelSenderContext` 或 `PluginHookChannelChatContext`
  以添加渠道特有字段。
- `runChannelInboundEvent(...)`：针对一个入站平台事件运行摄取、分类、预检、解析、
  记录、分发和最终处理。
- `dispatchChannelInboundReply(...)`：使用投递适配器记录并分发一个已
  组装的入站回复。

对于仅含媒体的入站事件，请将消息正文和命令文本留空，并为每个原生附件
传递一个 `ChannelInboundMediaInput` 事实。当环境
历史记录行或其他纯文本载体必须描述这些事实时，请使用
`formatMediaPlaceholderText(media)`。它会依次根据 `kind`、MIME
类型以及路径或 URL 扩展名对每个事实进行分类；尚未下载的原生附件仍应
各自提供一个仅含类型的事实。不要使用格式化程序合成
主要入站正文。

使用 `toInboundMediaFacts(...)` 规范化插件所有的附件记录，然后
通过上下文的 `media` 字段传递生成的有序数组：

```ts
const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

数组位置即附件标识。每个事实的 `transcribed`、`messageId` 和
`workspaceDir` 取代旧版并行索引/工作区字段。
`MediaPath`、`MediaPaths`、`MediaUrl`、`MediaUrls`、`MediaType`、`MediaTypes`、
`MediaTranscribedIndexes`、`MediaWorkspaceDir` 和 `MediaStaged` 上下文字段，
以及 `buildChannelInboundMediaPayload(...)`，仅作为已弃用的
兼容项保留。新插件不应构造或读取它们。

已获得注入插件运行时对象的内置/原生渠道
可以调用 `runtime.channel.inbound.*` 下的相同辅助函数，而无需
直接导入此子路径：

```ts
await runtime.channel.inbound.run({
  channel: "demo",
  accountId,
  raw: platformEvent,
  adapter: {
    ingest: normalizePlatformEvent,
    resolveTurn: resolveInboundReply,
  },
});
```

为将平台投递保留在投递适配器中的兼容
分发器组装 `dispatchChannelInboundReply(...)` 输入。新的发送
路径应改用 `channel-outbound` 中的消息适配器和持久化消息辅助函数。

## 投递结算契约

`ChannelInboundTurnPlan.delivery` 负责每个逻辑回复
载荷的原生发送。核心负责出站钩子顺序，并在适配器选择启用时
负责终结态 `message_sent` 观测。请将这些职责分开，以免
一个载荷产生重复的终结态事件。

投递结果字段含义如下：

| 字段                    | 契约                                                                                                                                                                                                                     |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `content`                | 原生格式化或最终处理后，提供商接受的逻辑载荷可见文本。省略时，终结态观测将使用准备好的载荷文本。仅媒体发送可将其省略。                             |
| `messageIds` / `receipt` | 可见发送的实际提供商标识。优先使用 `MessageReceipt`；核心使用其主要提供商 ID 作为 `message_sent`。                                                                                            |
| `visibleReplySent`       | 仅当提供商未生成任何可见预览或最终消息时，才将其设为 `false`。核心不会为该结果发出成功的 `message_sent`。                                                                          |
| `finalization`           | 同一逻辑载荷延迟原生结算的 Promise，例如关闭或编辑原位流式卡片。在终结态观测和 `onDelivered` 之前，其解析后的字段会覆盖即时结果。 |

当核心应为此适配器的非持久化发送发出规范的插件和内部
`message_sent` 事件时，将投递适配器的 `observeMessageSent` 选项设为 `true`。
不要从 `deliver` 返回此选项，也不要
在插件中重复发出这些事件。持久化发送已通过
共享出站所有者发出，不会重复。

每个逻辑载荷返回一个结果。`finalization` 并非第二次发送，
不得重新运行 `reply_payload_sending` 或 `message_sending`。一旦
`deliver` 返回，核心就会观测最终处理 Promise 的拒绝，
以防其成为未处理拒绝；回复分发结算后，核心仍会等待原始 Promise。
随后，它针对每个载荷最多发出一次终结态观测，
其中包含最终确定的内容和提供商 ID。若存在 `onDelivered`，
它将在该观测后接收结算结果。

原生投递失败时，拒绝 `deliver` 或 `finalization`。如果没有尝试
提供商发送，则从 `openclaw/plugin-sdk/error-runtime`
抛出 `PlatformMessageNotDispatchedError`；核心会抑制错误的 `message_sent`
事件。如果原生发送已经可见，但之后的操作失败，
请在错误中保留可见部分：

```ts
import { createChannelPartialDeliveryError } from "openclaw/plugin-sdk/channel-inbound";

throw createChannelPartialDeliveryError(cause, {
  visibleReplySent: true,
  content: finalizedVisibleText,
  receipt,
});
```

核心会使用该提供商可见的内容和标识发出失败的终结态观测，
同时保持投递失败状态，避免调用方将部分
成功误认为完全成功的发送。在任何
预览、草稿、附件或最终消息变得可见后，都不要报告 `visibleReplySent: false`。

注册 `reply_payload_sending` 或 `message_sending` 时，这些钩子
必须在创建任何提供商可见内容之前完成结算，因为任一钩子
都可以改写或取消逻辑载荷。过早的原生预览会泄露
改写前的内容，或留下已取消的草稿。请缓冲预览内容，
直到被接受的载荷到达 `deliver`；若任一钩子已
注册，较早启动预览的兼容分发器必须抑制该过早预览。
新预览路径请使用
[渠道出站 API](/zh-CN/plugins/sdk-channel-outbound) 中可最终处理的实时预览辅助函数。

## 迁移

`runtime.channel.turn.*` 运行时别名已移除。请使用：

- `runtime.channel.inbound.run(...)` 处理原始入站事件。
- `runtime.channel.inbound.dispatchReply(...)` 处理已组装的回复上下文。
- `runtime.channel.inbound.buildContext(...)` 处理入站上下文载荷。
- `runtime.channel.inbound.runPreparedReply(...)` 已弃用，仅用于
  已自行组装分发闭包、由渠道所有的预处理分发路径。

新的插件代码不应引入以 `turn` 命名的渠道 API。请将模型或
智能体轮次术语保留在智能体/提供商代码中；渠道插件应使用入站、
消息、投递和回复等术语。
