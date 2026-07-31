---
read_when:
    - 你正在建置或重構訊息通道外掛的接收路徑
    - 你需要共用的傳入內容脈絡建構、工作階段記錄或預先準備的回覆分派
    - 你正在將舊版頻道回合輔助函式遷移至 inbound/message API
summary: 頻道外掛的傳入事件輔助工具：情境建構、共用執行器協調、工作階段記錄，以及預備回覆分派
title: 頻道傳入 API
x-i18n:
    generated_at: "2026-07-26T07:29:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 854408ca42cfe1e1b48e4fd223b176f438e1db28deb9a5aa33eea8238127d9df
    source_path: plugins/sdk-channel-inbound.md
    workflow: 16
---

Channel 接收路徑遵循單一流程：

```text
平台事件 -> 入站事實／情境 -> 代理程式回覆 -> 訊息傳遞
```

使用 `openclaw/plugin-sdk/channel-inbound` 處理入站事件正規化、
格式化、根目錄與協調作業。使用
`openclaw/plugin-sdk/channel-outbound` 處理原生傳送、收據、持久化
傳遞與即時預覽行為。

## 核心輔助函式

```ts
import {
  buildChannelInboundEventContext,
  runChannelInboundEvent,
  dispatchChannelInboundReply,
} from "openclaw/plugin-sdk/channel-inbound";
```

- `buildChannelInboundEventContext(...)`：將正規化的 Channel 事實
  投影至提示詞／工作階段情境。透過 `channelContext` 傳遞
  Channel 所擁有的傳送者／聊天中繼資料，外掛鉤子會將其視為 `ctx.channelContext`。
  從此子路徑擴充 `PluginHookChannelSenderContext` 或 `PluginHookChannelChatContext`
  以加入 Channel 特定欄位。
- `runChannelInboundEvent(...)`：針對一個入站平台事件執行擷取、分類、預檢、解析、
  記錄、分派及完成作業。
- `dispatchChannelInboundReply(...)`：使用傳遞配接器記錄並分派已
  組裝完成的入站回覆。

對於僅含媒體的入站事件，請將訊息本文與命令文字留空，並
為每個原生附件傳入一項 `ChannelInboundMediaInput` 事實。當環境
歷史記錄行或其他純文字載體必須描述這些事實時，請使用
`formatMediaPlaceholderText(media)`。它會依序根據 `kind`、MIME
類型，再根據路徑或 URL 副檔名分類每項事實；尚未下載的原生附件仍應
各自提供一項僅含類型的事實。請勿使用格式化工具合成
主要入站本文。

使用 `toInboundMediaFacts(...)` 正規化外掛所擁有的附件記錄，然後
透過情境的 `media` 欄位傳入產生的有序陣列：

```ts
const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

陣列位置就是附件身分。每項事實的 `transcribed`、`messageId` 和
`workspaceDir` 會取代舊版平行索引／工作區欄位。
`MediaPath`、`MediaPaths`、`MediaUrl`、`MediaUrls`、`MediaType`、`MediaTypes`、
`MediaTranscribedIndexes`、`MediaWorkspaceDir` 和 `MediaStaged` 情境欄位，
以及 `buildChannelInboundMediaPayload(...)`，僅作為已棄用的
相容性功能保留。新的外掛不應建構或讀取這些欄位。

已接收注入之外掛執行階段物件的內建／原生 Channel，
可以改為呼叫 `runtime.channel.inbound.*` 下的相同輔助函式，而不必
直接匯入此子路徑：

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

為將平台傳遞保留在傳遞配接器中的相容性
分派器組裝 `dispatchChannelInboundReply(...)` 輸入。新的傳送
路徑應改用 `channel-outbound` 中的訊息配接器與持久化訊息輔助函式。

## 傳遞結算合約

`ChannelInboundTurnPlan.delivery` 負責每個邏輯回覆
承載內容的原生傳送。核心負責出站鉤子的執行順序，以及在配接器選擇加入時，
終端 `message_sent` 觀察。請將這些責任分開，以免
一項承載內容產生重複的終端事件。

傳遞結果欄位具有下列含義：

| 欄位                    | 合約                                                                                                                                                                                                                     |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `content`                | 邏輯承載內容經原生格式化或完成處理後，提供者所接受的可見文字。省略此欄位時，終端觀察會使用已準備的承載內容文字。僅媒體傳送可省略此欄位。                             |
| `messageIds` / `receipt` | 可見傳送的實際提供者身分。優先使用 `MessageReceipt`；核心會使用其主要提供者 ID 作為 `message_sent`。                                                                                            |
| `visibleReplySent`       | 僅當提供者未產生任何可見預覽或最終訊息時，才設為 `false`。核心不會為該結果發出成功的 `message_sent`。                                                                          |
| `finalization`           | 同一邏輯承載內容延遲完成原生結算的 Promise，例如關閉或編輯原地串流資訊卡。其解析後的欄位會在終端觀察與 `onDelivered` 之前覆寫立即結果。 |

當核心應為此配接器的非持久化傳送發出標準外掛及內部
`message_sent` 事件時，請將傳遞配接器的 `observeMessageSent` 選項設為 `true`。
請勿從 `deliver` 傳回此選項，也不要同時在
外掛中發出這些事件。持久化傳送已由共用出站擁有者發出，
不會重複發出。

每個邏輯承載內容傳回一項結果。`finalization` 並非第二次傳送，
不得重新執行 `reply_payload_sending` 或 `message_sending`。一旦
`deliver` 傳回，核心就會觀察完成處理 Promise 的拒絕，
以免它成為未處理的拒絕；在回覆分派完成後，核心仍會等待原始 Promise。
接著，它會使用完成處理後的內容與提供者 ID，為每項承載內容發出至多一次終端觀察。
若有 `onDelivered`，它會在該觀察之後接收
已結算的結果。

原生傳遞失敗時，拒絕 `deliver` 或 `finalization`。若未嘗試任何提供者
傳送，請從 `openclaw/plugin-sdk/error-runtime` 擲回 `PlatformMessageNotDispatchedError`；
核心會抑制錯誤的 `message_sent`
事件。若原生傳送在後續作業失敗前已經可見，
請在錯誤中保留可見的子集：

```ts
import { createChannelPartialDeliveryError } from "openclaw/plugin-sdk/channel-inbound";

throw createChannelPartialDeliveryError(cause, {
  visibleReplySent: true,
  content: finalizedVisibleText,
  receipt,
});
```

核心會使用該提供者可見的內容與
身分發出失敗的終端觀察，並維持傳遞失敗狀態，避免呼叫端將部分
成功誤認為完整傳送。任何預覽、草稿、附件或最終訊息
變得可見後，都不得回報 `visibleReplySent: false`。

註冊 `reply_payload_sending` 或 `message_sending` 時，這些鉤子
必須在建立任何提供者可見內容之前完成，因為任一鉤子
都可能改寫或取消邏輯承載內容。過早顯示原生預覽會洩漏
改寫前的內容，或遺留已取消的草稿。請緩衝預覽內容，
直到接受的承載內容抵達 `deliver`；若任一鉤子已註冊，
較早啟動預覽的相容性分派器必須抑制該過早預覽。新的預覽路徑請使用
[Channel 出站 API](/zh-TW/plugins/sdk-channel-outbound) 中可完成處理的即時預覽輔助函式。

## 移轉

`runtime.channel.turn.*` 執行階段別名已移除。請使用：

- `runtime.channel.inbound.run(...)` 用於原始入站事件。
- `runtime.channel.inbound.dispatchReply(...)` 用於已組裝的回覆情境。
- `runtime.channel.inbound.buildContext(...)` 用於入站情境承載內容。
- `runtime.channel.inbound.runPreparedReply(...)` 已棄用，僅適用於
  已自行組裝分派閉包、由 Channel 所擁有的已準備分派路徑。

新的外掛程式碼不應引入以 `turn` 命名的 Channel API。模型或
代理程式回合詞彙應保留在代理程式／提供者程式碼內；Channel 外掛應使用入站、
訊息、傳遞與回覆等詞彙。
