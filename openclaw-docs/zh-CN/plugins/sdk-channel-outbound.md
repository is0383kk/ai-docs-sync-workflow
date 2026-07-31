---
read_when:
    - 你正在构建或重构消息渠道插件的发送路径
    - 你需要可靠的最终回复投递、回执、实时预览定稿或接收确认策略
    - 你正在从频道消息或旧版回复分发辅助函数迁移
summary: 渠道插件的出站消息生命周期 API：适配器、回执、持久化发送、实时预览和回复管线辅助函数
title: 渠道出站 API
x-i18n:
    generated_at: "2026-07-26T06:23:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8edeca81d2e9261f33be1d538153caaea87caedb90dfccac33dd227c924501f1
    source_path: plugins/sdk-channel-outbound.md
    workflow: 16
---

渠道插件通过
`openclaw/plugin-sdk/channel-outbound` 暴露出站消息行为。使用
`openclaw/plugin-sdk/channel-inbound` 进行接收、上下文和分派
编排。

核心负责队列处理、持久性、持久化的**入口监控和排空**
（`createChannelIngressMonitor`、`createChannelIngressDrain` 和
`openChannelIngressDrain`）、通用重试策略、轮次接管生命周期
（`turnAdoptionLifecycle` / `bindIngressLifecycleToReplyOptions`）、钩子、
回执以及共享的 `message` 工具。插件负责原生
发送、编辑和删除调用、目标规范化、平台线程处理、选定的
引用、通知标志、账户状态、入口检查和有效载荷
编码、通道键、不可重试谓词、可选的取代
授权以及平台特定的副作用。

## 持久化入口监控

当渠道必须在分派前持久化已接受的
传输事件时，使用 `createChannelIngressMonitor(...)`。它将渠道入口队列和排空机制
与共享的准入、轮询、清理、投递和关闭生命周期组合起来。
仅当传输层拥有实质不同的准入或泵送契约时，才使用较低层级的
`createChannelIngressDrain(...)`。

必需选项如下：

| 选项                           | 契约                                                                                                                                                                                                                                                                                                         |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `queue`                          | 一个 `ChannelIngressQueue`，或打开账户作用域队列的惰性工厂。                                                                                                                                                                                                                                  |
| `inspect(raw, context)`          | 返回稳定的 `eventId` 和序列化的 `laneKey`，对于忽略的事件则返回 `null`。认领时的事实必须与持久化的 ID 和通道匹配。                                                                                                                                                                    |
| `payload`                        | 提供有效载荷版本以及正文序列化/反序列化。对标准的 `{ version, rawEvent }` 字符串信封使用 `storage: "raw-event"`，或为现有的渠道特定形态提供自定义编码/解码回调。`createClaimError` 对无效版本或身份变更进行分类。 |
| `deliver(raw, lifecycle, claim)` | 分派一个已解码事件并接收完整的接管生命周期。它可以返回 `completed`、`deferred`、`failed-retryable`，或不返回任何内容。                                                                                                                                                                |
| `pollIntervalMs`                 | 在监控运行期间调度恢复/排空轮询。                                                                                                                                                                                                                                                     |
| `retention`                      | 提供清理频率以及已完成/失败项的 TTL 和条目上限。                                                                                                                                                                                                                                              |

监控会串行化准入，确保追加退避不会颠倒通道内的顺序。
默认的有界追加延迟为 `0`、`100` 和 `300` ms；重试耗尽时会拒绝
传输回调，而不是分派一个尚未持久化的事件。
认领时，它会解码带版本的有效载荷，重新运行 `inspect`，并在投递前
拒绝 ID 或通道不匹配的情况。

`deliver` 接收 `onAdopted`、`onDeferred`、`onAdoptionFinalizing`、
`onAbandoned` 和 `abortSignal`。在没有显式移交的情况下返回，会将
终止且未分派的事件标记为已接管。`admission` 始终为 `exclusive`。
延迟移交会保持认领，而关闭或中止会使未接管的
工作保持可重试状态。监控独立跟踪投递和认领结算，
因为接管可能会在渠道的投递 Promise
返回前将行标记为墓碑。

可选设置包括自定义追加延迟、用于
高级排空顺序、并发和重试策略的 `drain` 选项块、外部 `abortSignal`、
时钟、泵送错误报告、已停止错误工厂以及准入策略。
返回的监控暴露 `admit`、`start`、`pause`、`stop`、`waitForIdle`、
`isRunning` 和 `isStopped`。`stop` 首先结算已接受的准入，然后
中止并释放排空器，等待泵和活跃投递完成，并
再次释放以消除惰性创建竞态。

将传输特定的脱敏、原始信封验证、不可重试
分类和持久化有效载荷形态保留在插件中。Webhook 传输
应仅在 `admit` 完成后确认；不可重放的传输应
暴露持久化追加耗尽错误，而不是静默分派。

## 适配器

大多数插件会定义一个 `message` 适配器：

```ts
import {
  defineChannelMessageAdapter,
  createMessageReceiptFromOutboundResults,
} from "openclaw/plugin-sdk/channel-outbound";

export const demoMessageAdapter = defineChannelMessageAdapter({
  id: "demo",
  durableFinal: {
    capabilities: {
      text: true,
      replyTo: true,
      thread: true,
      messageSendingHooks: true,
    },
  },
  send: {
    text: async ({ cfg, to, text, accountId, replyToId, threadId, signal }) => {
      const sent = await sendDemoMessage({
        cfg,
        to,
        text,
        accountId: accountId ?? undefined,
        replyToId: replyToId ?? undefined,
        threadId: threadId == null ? undefined : String(threadId),
        signal,
      });

      return {
        receipt: createMessageReceiptFromOutboundResults({
          results: [{ channel: "demo", messageId: sent.id, conversationId: to }],
          kind: "text",
          threadId: threadId == null ? undefined : String(threadId),
          replyToId: replyToId ?? undefined,
        }),
      };
    },
  },
});
```

仅声明原生传输实际保留的能力。使用此子路径导出的
契约辅助函数覆盖每项声明的发送、回执、实时预览和接收确认能力。

## 出站回显抑制

当平台可能将插件自身的出站消息重新投递为入站消息时，使用渠道、账户、会话以及稳定的平台消息或来源身份调用 `recordOutboundMessageIdentity(...)`。共享入站轮次路径会在会话记录或智能体分派前，于有界的 30 秒窗口内丢弃匹配身份；可以在发送前预留来源身份，或在移除渠道路由时刷新它，以消除投递竞态。`isRecentOutboundMessageIdentity(...)` 为渠道诊断和测试暴露相同的查询。不要为同一稳定身份维护并行的渠道本地 TTL 缓存。

## 纯文本清理

当出站适配器需要将支持的 HTML 格式标签转换为
轻量级文本标记时，使用 `sanitizeForPlainText(...)`。默认情况下保留
现有聊天风格的粗体和删除线标记。仅当渠道会将结果重新解析为 Markdown 时，
才传递 `{ style: "markdown" }`：

```ts
import { sanitizeForPlainText } from "openclaw/plugin-sdk/channel-outbound";

const chatText = sanitizeForPlainText(text);
const markdownText = sanitizeForPlainText(text, { style: "markdown" });
```

Markdown 样式使用 `**bold**` 和 `~~strikethrough~~`；斜体和行内
代码在两种样式中都保留 `_italic_` 和反引号标记。应在
渠道边界选择样式，而不是在清理后重写标记文本。

## 投递证据

`MessageReceipt` 记录渠道适配器返回的结果。具体的
平台消息标识符表明平台发送路径已接受
消息；它们并不能证明接收方设备已显示或读取该消息。
没有平台消息标识符的回执仅属于本地回执元数据。
具有已读回执或设备投递状态的渠道应通过独立的
渠道特定路径跟踪这些事实。

如果渠道适配器能够证明重试失败不会造成
接收方可见消息重复，并且尚未开始任何可完成最终处理的调用，则从
`openclaw/plugin-sdk/error-runtime` 抛出
`new PlatformMessageNotDispatchedError("...", { cause: error })`。这样核心便可清除过时的发送尝试
证据，并安全地重试已排队的意图。只有拥有最终
分派边界的适配器才能做出此断言。最终处理/发送调用一旦开始或返回
不明确的结果，绝不能使用该标记；错误标记可能导致
消息重复。

## 现有出站适配器

如果渠道已有兼容的 `outbound` 适配器，请基于它派生
消息适配器，而不要重复发送代码：

```ts
import { createChannelMessageAdapterFromOutbound } from "openclaw/plugin-sdk/channel-outbound";

export const messageAdapter = createChannelMessageAdapterFromOutbound({
  id: "demo",
  outbound,
  durableFinal: {
    capabilities: {
      text: true,
      media: true,
    },
  },
});
```

## 持久化发送

运行时发送辅助函数也位于 `channel-outbound`：

- `sendDurableMessageBatch(...)`
- `withDurableMessageSendContext(...)`
- `deliverInboundReplyWithMessageSendContext(...)`
- 草稿流式传输/进度辅助函数，例如 `resolveChannelDraftStreamingChunking(...)`

`sendDurableMessageBatch(...)` 返回一个显式结果：

| 结果          | 含义                                                                                 |
| ---------------- | --------------------------------------------------------------------------------------- |
| `sent`           | 平台发送路径至少接受了一条可见的平台消息            |
| `suppressed`     | 不应将任何平台消息视为缺失                                        |
| `partial_failed` | 在后续有效载荷或副作用失败前，至少有一条平台消息已被接受 |
| `failed`         | 未生成平台回执                                                        |

当批次混合已发送、已抑制和失败的
有效载荷时，使用 `payloadOutcomes`。不要根据空的旧版
直接投递结果推断钩子取消。

## 延迟投递准入

当已解析的账户无法安全接受核心管理的出站或延迟投递时，
使用 `message.durableFinal.admitDeferredDelivery(...)`。核心会在实时出站工作前同步调用
此钩子，包括跳过队列持久化的路径，并在重放恢复的意图前再次调用。
上下文包含 `cfg`、`channel`、`to`、`accountId`，以及值为 `live` 或
`recovery` 的 `phase`。

返回 `{ status: "allowed" }` 以继续。当投递不得
持久化、直接发送或重放时，返回
`{ status: "permanent_rejection", reason }`。实时拒绝会在创建队列、
消息钩子或平台工作之前失败。恢复拒绝会将
排队记录标记为失败，并跳过协调和重放。省略该钩子
即表示允许。

该钩子用于执行同步准入决策，而非发送路径。只读取
已加载的配置或运行时状态；不要执行网络、文件系统或
其他异步 I/O。契约测试应通过
`openclaw/plugin-sdk/channel-outbound` 中的 `ChannelMessageDurableFinalAdapter`
覆盖两个阶段和两种结果变体。

## 兼容性分派

通过 `channel-inbound` 中的 `dispatchChannelInboundReply(...)`
组装入站回复分派。将平台投递保留在投递适配器中；使用
`channel-outbound` 处理消息适配器、持久化发送、回执、实时
预览和回复流水线选项。
