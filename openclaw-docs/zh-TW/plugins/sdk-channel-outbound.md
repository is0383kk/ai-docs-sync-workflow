---
read_when:
    - 你正在建置或重構訊息傳遞頻道外掛的傳送路徑
    - 你需要持久可靠的最終回覆傳送、回執、即時預覽定稿或接收確認政策
    - 你正在從 channel-message 或舊版回覆分派輔助函式進行遷移
summary: 頻道外掛的外送訊息生命週期 API：轉接器、回執、持久傳送、即時預覽與回覆流水線輔助工具
title: 頻道傳出 API
x-i18n:
    generated_at: "2026-07-26T08:00:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8edeca81d2e9261f33be1d538153caaea87caedb90dfccac33dd227c924501f1
    source_path: plugins/sdk-channel-outbound.md
    workflow: 16
---

Channel 外掛會從
`openclaw/plugin-sdk/channel-outbound` 公開出站訊息行為。使用
`openclaw/plugin-sdk/channel-inbound` 進行接收／上下文／分派
協調。

核心負責佇列、持久性、持久化的**輸入監控與排空**
（`createChannelIngressMonitor`、`createChannelIngressDrain` 和
`openChannelIngressDrain`）、通用重試原則、回合採用生命週期
（`turnAdoptionLifecycle` / `bindIngressLifecycleToReplyOptions`）、掛鉤、
收據，以及共用的 `message` 工具。外掛負責原生
傳送／編輯／刪除呼叫、目標正規化、平台討論串、選定的
引用、通知旗標、帳號狀態、輸入檢查與酬載
編碼、通道鍵、不可重試判定條件、可選的取代
授權，以及平台特定的副作用。

## 持久化輸入監控

當 Channel 必須在分派前持久保存已接受的
傳輸事件時，請使用 `createChannelIngressMonitor(...)`。它會將 Channel 輸入佇列與排空機制
和共用的准入、輪詢、修剪、遞送及關閉生命週期組合在一起。
只有當傳輸層擁有實質不同的准入或泵送契約時，才使用較低階的 `createChannelIngressDrain(...)`。

必要選項如下：

| 選項                           | 契約                                                                                                                                                                                                                                                                                                         |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `queue`                          | 一個 `ChannelIngressQueue`，或開啟帳號範圍佇列的延遲工廠。                                                                                                                                                                                                                                  |
| `inspect(raw, context)`          | 傳回穩定的 `eventId` 和序列化的 `laneKey`，若事件應忽略則傳回 `null`。宣告時的事實必須與持久保存的 ID 和通道相符。                                                                                                                                                                    |
| `payload`                        | 提供酬載版本以及主體序列化／反序列化。標準 `{ version, rawEvent }` 字串封套請使用 `storage: "raw-event"`，既有的 Channel 特定格式則提供自訂編碼／解碼回呼。`createClaimError` 會分類無效版本或變更的身分。 |
| `deliver(raw, lifecycle, claim)` | 分派一個已解碼事件，並接收完整的採用生命週期。它可以傳回 `completed`、`deferred`、`failed-retryable`，或不傳回任何內容。                                                                                                                                                                |
| `pollIntervalMs`                 | 在監控執行期間排程復原／排空輪詢。                                                                                                                                                                                                                                                     |
| `retention`                      | 提供修剪頻率，以及已完成／失敗項目的 TTL 與數量上限。                                                                                                                                                                                                                                              |

監控會將准入作業序列化，因此附加操作的退避不會顛倒通道內的順序。預設的
有界附加延遲為 `0`、`100` 和 `300` ms；耗盡後會拒絕
傳輸回呼，而不是分派尚未持久化的事件。
在宣告時，它會解碼有版本的酬載、重新執行 `inspect`，並在遞送前
拒絕 ID 或通道不符的項目。

`deliver` 會接收 `onAdopted`、`onDeferred`、`onAdoptionFinalizing`、
`onAbandoned` 和 `abortSignal`。若未明確交接便傳回，會將終止且未分派的事件標記為已採用。
`admission` 一律為 `exclusive`。延後交接會保持宣告，而關閉或中止則會讓未採用的
工作保持可重試。監控會分別追蹤遞送與宣告結算，
因為採用可能會在 Channel 的遞送 Promise 傳回前，先為資料列建立墓碑。

可選設定包括自訂附加延遲、用於進階排空排序／並行處理／重試原則的 `drain` 選項區塊、外部 `abortSignal`、
時鐘、泵送錯誤回報、已停止錯誤工廠，以及准入原則。
傳回的監控會公開 `admit`、`start`、`pause`、`stop`、`waitForIdle`、
`isRunning` 和 `isStopped`。`stop` 會先結算已接受的准入作業，接著
中止並釋放排空機制，等待泵送與進行中的遞送完成，然後
再次釋放，以處理延遲建立的競爭條件。

傳輸特定的遮蔽、原始封套驗證、不可重試
分類，以及持久化酬載格式，應保留在外掛中。網路鉤子傳輸
應只在 `admit` 解析完成後確認；不可重播的傳輸應
公開持久附加耗盡錯誤，而非無聲地分派。

## 轉接器

大多數外掛會定義一個 `message` 轉接器：

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

只宣告原生傳輸實際保留的能力。使用
此子路徑匯出的契約輔助函式，涵蓋每項已宣告的傳送、收據、即時預覽及接收確認能力。

## 出站回音抑制

當平台可能將外掛自己的出站訊息重新遞送為輸入時，請使用 Channel、帳號、對話，以及穩定的平台訊息或來源身分呼叫 `recordOutboundMessageIdentity(...)`。共用輸入回合路徑會在工作階段記錄或代理程式分派前，於有界的 30 秒時段內丟棄相符的身分；可在傳送前保留來源身分，或在移除 Channel 路由時重新整理，以封閉遞送競爭條件。`isRecentOutboundMessageIdentity(...)` 會為 Channel 診斷與測試公開相同的查詢。請勿針對同一穩定身分維護平行的 Channel 本機 TTL 快取。

## 純文字清理

當出站轉接器需要將支援的 HTML 格式標籤轉換成
輕量文字標記時，請使用 `sanitizeForPlainText(...)`。預設會保留
現有聊天樣式的粗體與刪除線標記。只有在
Channel 會將結果重新剖析為 Markdown 時，才傳入 `{ style: "markdown" }`：

```ts
import { sanitizeForPlainText } from "openclaw/plugin-sdk/channel-outbound";

const chatText = sanitizeForPlainText(text);
const markdownText = sanitizeForPlainText(text, { style: "markdown" });
```

Markdown 樣式使用 `**bold**` 和 `~~strikethrough~~`；在兩種樣式中，斜體與行內
程式碼都會保留 `_italic_` 和反引號標記。請在
Channel 邊界選擇樣式，而非在清理後重寫標記文字。

## 遞送證據

`MessageReceipt` 會記錄 Channel 轉接器傳回的結果。具體的
平台訊息識別碼表示平台傳送路徑已接受
該訊息；這無法證明收件者的裝置已顯示或讀取訊息。
沒有平台訊息識別碼的收據只是本機收據中繼資料。
具備讀取回條或裝置遞送狀態的 Channel，應透過
獨立的 Channel 特定路徑追蹤這些事實。

如果 Channel 轉接器能證明重試失敗不可能導致
收件者可見的重複傳送，且尚未開始任何可完成最終處理的呼叫，請從
`openclaw/plugin-sdk/error-runtime` 擲回
`new PlatformMessageNotDispatchedError("...", { cause: error })`。核心便可清除過期的傳送嘗試
證據，並安全地重試已排入佇列的意圖。只有擁有
最終分派邊界的轉接器可以做出此斷言。最終處理／傳送呼叫開始後，或其傳回
模稜兩可的結果時，絕不可使用此標記；錯誤標記可能會
造成訊息重複。

## 既有出站轉接器

如果 Channel 已有相容的 `outbound` 轉接器，請從中衍生
訊息轉接器，而不是重複傳送程式碼：

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

## 持久化傳送

執行階段傳送輔助函式也位於 `channel-outbound`：

- `sendDurableMessageBatch(...)`
- `withDurableMessageSendContext(...)`
- `deliverInboundReplyWithMessageSendContext(...)`
- 草稿串流／進度輔助函式，例如 `resolveChannelDraftStreamingChunking(...)`

`sendDurableMessageBatch(...)` 會傳回一個明確結果：

| 結果          | 意義                                                                                 |
| ---------------- | --------------------------------------------------------------------------------------- |
| `sent`           | 平台傳送路徑已接受至少一則可見的平台訊息            |
| `suppressed`     | 不應將任何平台訊息視為遺失                                        |
| `partial_failed` | 在後續酬載或副作用失敗前，平台已接受至少一則平台訊息 |
| `failed`         | 未產生平台收據                                                        |

當批次混合已傳送、已抑制和失敗的
酬載時，請使用 `payloadOutcomes`。請勿從空白的舊版
直接遞送結果推斷掛鉤取消。

## 延後遞送准入

當已解析的帳號無法安全接受由核心管理的出站或延後遞送時，
請使用 `message.durableFinal.admitDeferredDelivery(...)`。核心會在即時出站工作前同步呼叫
此掛鉤，包括略過佇列持久化的路徑，並在重播已復原的意圖前再次呼叫。
上下文包含 `cfg`、`channel`、`to`、`accountId`，以及值為 `live` 或
`recovery` 的 `phase`。

傳回 `{ status: "allowed" }` 以繼續。當遞送不得
持久保存、直接傳送或重播時，請傳回
`{ status: "permanent_rejection", reason }`。即時拒絕會在建立佇列、
訊息掛鉤或平台作業前失敗。復原拒絕會將
佇列記錄標記為失敗，並略過對帳與重播。省略此掛鉤
即表示允許。

此掛鉤是同步的准入決策，而不是傳送路徑。僅讀取
已載入的設定或執行階段狀態；請勿執行網路、檔案系統或
其他非同步 I/O。契約測試應透過 `openclaw/plugin-sdk/channel-outbound` 中的
`ChannelMessageDurableFinalAdapter`，涵蓋兩個階段與兩種結果變體。

## 相容性分派

透過 `channel-inbound` 中的 `dispatchChannelInboundReply(...)`
組合入站回覆分派。將平台傳遞保留在傳遞配接器中；將
`channel-outbound` 用於訊息配接器、持久傳送、回條、即時
預覽及回覆流水線選項。
