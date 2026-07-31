---
read_when:
    - 重構頻道傳送或接收行為
    - 變更頻道傳入、回覆分派、傳出佇列、預覽串流或外掛 SDK 訊息 API
    - 設計需要持久化傳送、回執、預覽、編輯或重試的新頻道外掛
summary: 持久化訊息接收／傳送生命週期的狀態：已發布的內容、相較原始設計的變更，以及仍待處理的事項
title: 訊息生命週期重構
x-i18n:
    generated_at: "2026-07-26T08:30:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d21eda70b8be0de78677f4ff6d7547317112731d9e86a5bef58eac0268899818
    source_path: concepts/message-lifecycle-refactor.md
    workflow: 16
---

<Note>
此頁面最初是一份前瞻性設計提案。該設計的核心後來已在 `src/channels/message/*` 與公開的
`openclaw/plugin-sdk/channel-outbound` / `channel-inbound` 子路徑中推出。若要使用目前的 API，請參閱[頻道輸出 API](/zh-TW/plugins/sdk-channel-outbound) 與
[頻道輸入 API](/zh-TW/plugins/sdk-channel-inbound)。此頁面追蹤已推出的內容、實作與原始草案的差異，以及仍待處理的項目。
</Note>

## 為何進行這次重構

頻道堆疊由多項局部修正逐漸形成：依成熟度提供不同的輸入輔助函式（簡易配接器使用
`runtime.channel.inbound.run`，功能豐富的配接器使用
`runtime.channel.inbound.runPreparedReply`）、舊版回覆分派輔助函式（`dispatchInboundReplyWithBase`、`recordInboundSessionAndDispatchReply`）、
頻道專屬的預覽串流，以及附加至既有回覆承載資料路徑的最終傳遞持久性。這種架構產生了過多公開概念，也導致傳遞語意可能在太多位置出現偏差。

迫使這次重新設計的可靠性缺口：

```text
Telegram 輪詢更新已確認
  -> 助理的最終文字已存在
  -> 程序在 sendMessage 成功前重新啟動
  -> 最終回覆遺失
```

目標不變條件：一旦核心決定應存在一則使用者可見的輸出訊息，就必須在嘗試呼叫平台前持久化傳送意圖，並在成功後提交平台收據。如此預設可提供至少一次的復原能力。只有配接器能證明原生冪等性，或在重播前能根據平台狀態核對傳送後結果不明的嘗試時，才具有恰好一次的行為。

## 已推出的內容

內部領域位於 `src/channels/message/*`：

| 檔案                        | 負責範圍                                                                                                               |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `types.ts`                  | 配接器、傳送情境、收據及持久意圖的型別合約                                                  |
| `send.ts`                   | `withDurableMessageSendContext` / `sendDurableMessageBatch` — 持久傳送情境                             |
| `receive.ts`                | `createMessageReceiveContext` — 輸入確認原則狀態機                                                   |
| `live.ts`                   | 即時預覽狀態，以及原地完成或改用備援方式的邏輯                                                        |
| `state.ts`                  | `classifyDurableSendRecoveryState` — 中斷後的復原分類                                    |
| `receipt.ts`                | 將平台傳送結果正規化為 `MessageReceipt`                                                             |
| `capabilities.ts`           | 從承載資料推導持久最終傳遞所需的能力                                                         |
| `contracts.ts`              | 驗證所宣告配接器能力的合約證明                                                      |
| `adapter.ts`                | `defineChannelMessageAdapter`                                                                                      |
| `outbound-bridge.ts`        | `createChannelMessageAdapterFromOutbound` — 包裝舊版 `sendText`/`sendMedia`/`sendPayload`/`sendPoll` 函式 |
| `ingress-queue.ts`          | `createChannelIngressQueue` — 持久輸入事件佇列                                                          |
| `durable-receive.ts`        | `createDurableInboundReceiveJournal` — 用於輸入去重的接受／待處理／完成／釋放日誌                  |
| `inbound-reply-dispatch.ts` | `dispatchChannelInboundReply` 與沿用舊名稱的包裝函式                                                            |
| `reply-pipeline.ts`         | `createChannelReplyPipeline`、回覆前置字串及輸入狀態回呼輔助函式                                             |

公開介面：`openclaw/plugin-sdk/channel-outbound`（傳送／收據／持久／即時／回覆流水線輔助函式）與 `openclaw/plugin-sdk/channel-inbound`（輸入情境、`runChannelInboundEvent`、`dispatchChannelInboundReply`）。如需配接器範例、目前的型別名稱及遷移注意事項，請參閱這些頁面；API 架構應以這些頁面為準，而非下方草案。

### 傳送情境

`withDurableMessageSendContext` 為頻道程式碼提供圍繞單一輸出訊息的 `render`、`previewUpdate`、
`send`、`edit`、`delete`、`commit` 與 `fail` 步驟。`sendDurableMessageBatch` 是一般情況的包裝函式：先算繪、傳送，再於 `sent`/`suppressed` 時提交，或在發生錯誤時標記失敗。

`sendDurableMessageBatch` 會傳回下列其中一種可辨識聯集結果：

| 狀態           | 意義                                                                          |
| ---------------- | -------------------------------------------------------------------------------- |
| `sent`           | 至少已傳遞一則使用者可見的平台訊息                              |
| `suppressed`     | 不應將任何平台訊息視為遺漏（被鉤子取消、試執行等） |
| `partial_failed` | 在後續承載資料或副作用失敗前，至少已傳遞一則訊息      |
| `failed`         | 未產生平台收據                                                 |

持久性為 `required`、`best_effort` 或 `disabled` 之一（`src/channels/message/types.ts` 中的 `MessageDurabilityPolicy`）。當無法寫入持久意圖時，`required` 會採取封閉失敗；當持久化無法使用時，`best_effort` 會改為直接傳送；`disabled` 則保留重構前的直接傳送行為。舊版相容性輔助函式預設使用
`disabled`，不會僅因頻道具有通用輸出配接器就推斷使用 `required`。

仍具風險的邊界：平台呼叫成功之後、收據提交之前。若程序在此時終止，除非配接器宣告 `reconcileUnknownSend`，否則核心無法判斷平台訊息是否存在。該鉤子會將中斷的傳送分類為 `sent`、`not_sent` 或 `unresolved`；只有 `not_sent` 允許重播。不具核對機制的頻道會退回 `unknown_after_send` 狀態（`src/channels/message/state.ts`、`src/infra/outbound/delivery-queue-recovery.ts`），而且只有當重複的可見訊息是該頻道可接受且已記錄於文件中的取捨時，才可選擇至少一次重播。

### 接收情境

`createMessageReceiveContext` 會追蹤每個輸入事件的確認／否定確認狀態，並提供冪等的 `ack()` 與明確的 `nack(error)`。確認原則（`ChannelMessageReceiveAckPolicy`）為下列其中之一：

| 原則                 | 確認時機                                                                                     |
| ---------------------- | --------------------------------------------------------------------------------------------- |
| `after_receive_record` | 核心已持久化足夠的輸入中繼資料，可對重新傳遞進行去重／路由                           |
| `after_agent_dispatch` | 代理程式執行已分派                                                             |
| `after_durable_send`   | 此輪次的持久輸出傳送已提交                                             |
| `manual`               | 呼叫端明確控制確認時機（未宣告原則的配接器預設使用此項） |

Telegram 輪詢會使用此機制持久化安全完成的更新水位標記（`extensions/telegram/src/bot-update-tracker.ts` 中的 `safeCompletedUpdateId`）：
grammY 仍會觀察進入中介軟體鏈的每個更新，但 OpenClaw 只會讓持久化的重新啟動水位標記越過已完成分派的更新，因此失敗或仍在待處理的更新會在重新啟動後重播。Telegram 上游的 `getUpdates` 位移量仍由 grammY 負責；目前尚未建置可在此水位標記之外控制平台層級重新傳遞的完全持久化輪詢來源（請參閱「待解問題」）。

### 即時預覽

`src/channels/message/live.ts` 將預覽／編輯／完成塑造成單一生命週期：
`createLiveMessageState`、`markLiveMessagePreviewUpdated`、
`markLiveMessageFinalized`、`markLiveMessageCancelled` 與
`deliverFinalizableLivePreviewAdapter`（從草稿建立最終編輯內容、套用該內容，並在無法編輯或編輯失敗時改用一般傳送）。
`LiveMessageState.phase` 是 `idle | previewing | finalizing | finalized |
cancelled`；`canFinalizeInPlace` 控制預覽是否可透過編輯成為最終訊息，而不需重新傳送。

### 持久收據

`MessageReceipt`（`src/channels/message/types.ts`）會將單次邏輯傳送產生的一個或多個平台訊息 ID 正規化為 `platformMessageIds`，以及各部分的 `parts`（種類、索引、討論串 ID、回覆目標 ID）。系統會保留主要 ID，以供討論串處理及後續編輯使用。這讓多部分傳遞（文字加媒體、分段文字、卡片備援）可在重新啟動後重播及去重。

### 縮減公開 SDK

此次重構已吸收或棄用：`reply-runtime`、`reply-dispatch-runtime`、
`reply-reference`、`reply-chunking`、作為公開 API 提供的 `reply-payload` 輔助函式、`inbound-reply-dispatch`、`channel-reply-pipeline`，以及舊輸出門面的大多數公開用法。`src/plugin-sdk/channel-message.ts` 現在是指向 `channel-outbound` /
`channel-inbound` 的 `@deprecated` 重新匯出彙整模組；`channel.turn` 執行階段別名已移除，舊版 `/plugins/sdk-channel-turn` 文件頁面會重新導向至
[頻道輸入 API](/zh-TW/plugins/sdk-channel-inbound)。新的外掛程式碼應直接以 `channel-outbound` 與 `channel-inbound` 為目標。

## 實作與原始設計的差異

下方的設計草案從未完全依照描述推出。保留此紀錄是為了確保歷史正確性；請勿將這些型別名稱視為目前的 API。

- **沒有 `MessageOrigin` / `shouldDropOpenClawEcho`。** 原始計畫要求在閘道失敗訊息上加上 `source: "openclaw"` 來源標籤，並提供共用述詞，在執行 `allowBots` 授權之前，丟棄共用聊天室中帶有標籤、由機器人撰寫的回音訊息。程式碼庫中不存在該型別與述詞。`allowBots` 本身是實際存在的頻道層級設定鍵（Slack、Discord、Google Chat 等），但原本要用來保護它的來源標記機制從未建置。在已啟用機器人的聊天室中抑制閘道失敗回音，仍是尚未解決的缺口，而非已推出的保證。
- **沒有統一的 `core.messages.receive/send/live/state` 命名空間。** 已推出的函式直接位於 `src/channels/message/*`（`withDurableMessageSendContext`、`createMessageReceiveContext`、`createLiveMessageState`、`classifyDurableSendRecoveryState`），而不是置於 `core.messages.*` 門面之後。
- **沒有通用的 `ChannelMessage` / `MessageTarget` / `MessageRelation` 正規化訊息型別。** 核心仍會透過傳送配接器傳遞具體的回覆承載資料（`ReplyPayload`）與頻道專屬情境，而非使用含有 `kind: "reply" |
"followup" | "broadcast" | "system"` 關聯的單一平台中立訊息架構。
- **確認原則名稱與草案不同。** 已推出：
  `after_receive_record | after_agent_dispatch | after_durable_send | manual`。
  原始草案使用含有網路鉤子逾時原因欄位的 `immediate | after-record | after-durable-send |
manual`；該架構並未建置。
- **`DurableFinalDeliveryRequirementMap` 能力鍵取代了草案中的
  `MessageCapabilities` 物件。** 能力採用扁平布林旗標（`text`、
  `media`、`poll`、`payload`、`silent`、`replyTo`、`thread`、`nativeQuote`、
  `messageSendingHooks`、`batch`、`reconcileUnknownSend`、`afterSendSuccess`、
  `afterCommit`），並透過 `verifyDurableFinalCapabilityProofs` 驗證，而非採用巢狀的 `text.chunking` / `attachments.voice` 式結構。

## 具體遷移風險（仍然適用）

這些頻道專屬副作用早於此次重構，且必須透過新的傳送路徑繼續運作。它們並非假設情境：每一項目前都已實作，並承擔關鍵功能。

- **iMessage**（`extensions/imessage/src/monitor/echo-cache.ts`、
  `persisted-echo-cache.ts`）：監視器會在成功傳送後，將已傳送的訊息記錄至回音
  快取。具持久性的最終傳送仍必須填入該快取，否則 OpenClaw 可能會將自己的回覆
  重新擷取為使用者傳入訊息。
- **Tlon**（`extensions/tlon/src/monitor/index.ts`）：附加選用的模型
  簽章，並在群組回覆後記錄參與過的討論串。持久性
  傳遞不得略過這些效果。
- **Discord 與其他已準備的分派器**已自行處理直接傳遞與
  預覽行為。在某個頻道的已準備分派器明確透過傳送情境路由最終訊息之前，
  該頻道並未實現端對端持久性；請勿假設僅靠通用轉接器便已涵蓋。
- **Telegram 靜默備援傳遞**在進行分塊／備援
  投影後，必須傳遞完整的投影酬載陣列，而不只是第一個酬載。
- **LINE、Zalo、Nostr** 及類似的輔助路徑可能具有回覆權杖
  處理、媒體代理、已傳送訊息快取，或僅限回呼的目標。
  在傳送轉接器能呈現這些語意，且有測試涵蓋之前，它們仍由頻道自行處理傳遞。
- **直接私訊輔助程式**可能具有唯一正確傳輸目標的回覆回呼。
  通用對外傳送不得從原始平台欄位猜測目標並略過該回呼。

## 失敗分類

轉接器會將傳輸失敗分類為 `DeliveryFailureKind` 形式的封閉
類別（暫時性、速率限制、驗證、權限、找不到、無效
酬載、衝突、已取消、未知）。核心政策：

- 重試暫時性與速率限制失敗。
- 除非存在呈現備援，否則不要重試無效酬載失敗。
- 在設定變更之前，不要重試驗證或權限失敗。
- 發生找不到錯誤時，若頻道宣告此做法安全，允許即時最終處理從編輯
  改為重新傳送。
- 發生衝突時，使用收據／冪等性狀態來判斷訊息
  是否已存在。
- 若平台呼叫可能已成功，但在提交收據前發生任何錯誤，
  除非轉接器能證明平台操作並未發生，否則該錯誤會成為 `unknown_after_send`。

## 開放問題

- Telegram 最終是否應以完全持久性的輪詢來源取代 grammY（`1.43.0`）
  輪詢執行器；該來源不只控制 OpenClaw 持久化的重新啟動水位標記
  （`safeCompletedUpdateId`），也控制平台層級的重新傳遞。
- 即時預覽狀態應與最終傳送意圖存放在同一筆記錄中，
  還是存放在同層的即時狀態儲存區中。
- 共用且已啟用機器人的聊天室中，因閘道失敗而進行的回音抑制，
  是否需要原先規劃的來源標記機制、較簡單的個別頻道
  合約，或是不在範圍內。
- 哪些頻道原生支援來源／中繼資料，可用於跨機器人的回音
  抑制；哪些頻道則需要持久化的對外傳送登錄檔。

## 相關內容

- [訊息](/zh-TW/concepts/messages)
- [串流與分塊](/zh-TW/concepts/streaming)
- [進度草稿](/zh-TW/concepts/progress-drafts)
- [重試政策](/zh-TW/concepts/retry)
- [頻道對外傳送 API](/zh-TW/plugins/sdk-channel-outbound)
- [頻道傳入 API](/zh-TW/plugins/sdk-channel-inbound)
