---
read_when:
    - 說明傳入訊息如何轉化為回覆
    - 釐清工作階段、佇列模式或串流行為
    - 記錄推理可見性與使用量影響
summary: 訊息流程、工作階段、佇列處理與推理可見性
title: 訊息
x-i18n:
    generated_at: "2026-07-26T07:17:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e42bed834e9a57fb8a248c8654b75ea9977928582f68a83859cf6c16ed0b6bf5
    source_path: concepts/messages.md
    workflow: 16
---

輸入訊息會經過路由、去重複／防彈跳、代理程式執行，以及輸出傳遞：

```text
輸入訊息
  -> 路由／繫結 -> 工作階段金鑰
  -> 去重複 + 防彈跳
  -> 佇列（若已有執行中的作業）
  -> 代理程式執行（串流 + 工具）
  -> 輸出回覆（頻道限制 + 分塊）
```

主要設定介面：

- `messages.*` 用於前綴、佇列、輸入防彈跳及群組行為。
- `agents.defaults.*` 用於區塊串流、分塊及靜默回覆預設值。
- 頻道覆寫（`channels.telegram.*`、`channels.whatsapp.*` 等）用於各頻道的上限與串流切換。

完整結構描述請參閱[設定](/zh-TW/gateway/configuration)。

## 輸入去重複

頻道可能會在重新連線後重新傳遞相同訊息。OpenClaw 會保留一個記憶體內快取，並以代理程式範圍、頻道路由（頻道 + 對等端 + 帳號 + 討論串）及訊息 ID 作為索引鍵，因此重新傳遞的訊息不會觸發第二次代理程式執行。快取項目會在 20 分鐘後或追蹤到 5000 個項目時到期，以先發生者為準。

## 輸入防彈跳

來自相同傳送者、快速連續送出的文字訊息，可透過 `messages.inbound` 批次合併為一次代理程式輪次。防彈跳以每個頻道 + 對話為範圍，並使用最新訊息進行回覆討論串關聯與 ID 指派。

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

- 防彈跳僅套用於純文字訊息；媒體／附件會立即送出。
- 控制命令（停止／中止／狀態等）會略過防彈跳，以便立即分派。
- 預設停用：`messages.inbound.debounceMs` 沒有內建預設值，因此只有在你設定後（全域或各頻道）才會啟用防彈跳。
- iMessage 遵循相同的一般防彈跳政策。`imsg` 0.13.1 及更新版本會在 OpenClaw 收到訊息前，先合併 Apple 網址預覽造成的分次傳送，因此不需要 iMessage 專用的防彈跳設定。

## 工作階段與裝置

工作階段由閘道擁有，而非用戶端。

- 直接聊天會合併至代理程式的主要工作階段金鑰。
- 群組／頻道會取得各自的工作階段金鑰。
- 工作階段儲存區與文字記錄位於閘道主機上。

多個裝置／頻道可以對應至相同工作階段，但歷程不會完整同步回每個用戶端。長時間對話請使用一部主要裝置，以避免情境分歧。控制介面與終端介面一律顯示由閘道支援的工作階段文字記錄，因此它們是唯一可信來源。

詳細資訊：[工作階段管理](/zh-TW/concepts/session)。

## 提示詞本文與歷程情境

頻道外掛會在輸入情境中填入數個文字欄位，以下依優先順序由高至低排列：

| 欄位              | 用途                                                                                                        |
| ----------------- | ----------------------------------------------------------------------------------------------------------- |
| `BodyForAgent`    | 目前輪次提供給模型的文字。未設定時會退回使用 `CommandBody`／`RawBody`／`Body`。        |
| `BodyForCommands` | 用於剖析指令／命令的乾淨文字。未設定時會退回使用 `CommandBody`／`RawBody`／`Body`。 |
| `CommandBody`     | 舊版中介本文；建議使用 `BodyForCommands`。                                                         |
| `RawBody`         | `CommandBody` 的已棄用別名。                                                                         |
| `Body`            | 舊版提示詞本文；可能包含頻道封裝與歷程包裝。                                     |

當頻道提供歷程時，會使用以下內容包裝：

- `[Chat messages since your last reply - for context]`
- `[Current message - respond to this]`

對於非直接聊天（群組／頻道／聊天室），目前訊息本文會加上傳送者標籤作為前綴，格式與歷程項目相同。移除指令僅套用於目前訊息區段，因此歷程會保持完整。包裝歷程的頻道應將 `BodyForCommands`（或舊版 `CommandBody`／`RawBody`）設為原始訊息文字，並將 `Body` 保留為合併後的提示詞。

歷程緩衝區僅包含待處理內容：其中包含未觸發執行的群組訊息（例如受提及條件限制的訊息），並排除已存在於工作階段文字記錄中的訊息。組合提示詞時，結構化歷程、回覆、轉寄及頻道中繼資料會呈現為不受信任的使用者角色情境區塊。

使用 `messages.groupChat.historyLimit`（全域預設值）或各頻道覆寫（例如 `channels.slack.historyLimit` 與 `channels.telegram.accounts.<id>.historyLimit`）設定歷程大小（將 `0` 設為停用）。

## 工具結果中繼資料

工具結果的 `content` 是模型可見的結果；`details` 則是用於介面呈現、診斷、媒體傳遞及外掛的執行階段中繼資料。

- `toolResult.details` 會在重新提交給供應商及輸入壓縮前移除。
- 持久化的工作階段文字記錄僅保留受限大小的 `details`；過大的中繼資料會替換為標記了 `persistedDetailsTruncated: true` 的精簡摘要。
- 外掛與工具應將模型必須讀取的文字放入 `content`，而非僅放入 `details`。

## 佇列與後續訊息

當已有作業正在執行時，輸入訊息預設會引導至該作業。`messages.queue` 控制模式：

| 模式              | 行為                                                |
| ----------------- | --------------------------------------------------- |
| `steer`（預設） | 將新提示詞注入執行中的作業。          |
| `followup`        | 在執行中的作業完成後執行該訊息。      |
| `collect`         | 將相容訊息批次合併至稍後的一次輪次。      |
| `interrupt`       | 中止執行中的作業，然後開始執行最新提示詞。 |

佇列針對引導、後續及收集批次處理使用內建的 500ms 防彈跳。`messages.queue.cap` 預設為 20 則佇列訊息，而 `messages.queue.drop` 預設為 `summarize`（亦可使用 `old` 與 `new`）。透過 `messages.queue.byChannel` 與 `messages.queue.debounceMsByChannel` 設定各頻道覆寫。

詳細資訊：[命令佇列](/zh-TW/concepts/queue)與[引導佇列](/zh-TW/concepts/queue-steering)。

## 頻道執行所有權

頻道外掛可在訊息進入工作階段佇列前維持順序、對輸入套用防彈跳，並施加傳輸背壓。它們不應對代理程式輪次本身額外設定逾時。訊息路由至工作階段後，長時間執行作業會由工作階段、工具及執行階段生命週期管理，讓所有頻道都能以一致方式回報緩慢輪次並從中復原。

## 串流、分塊與批次處理

區塊串流會隨模型產生文字區塊而傳送部分回覆；分塊會遵守頻道文字限制，並避免拆分圍欄程式碼。

- `agents.defaults.blockStreamingDefault`（`on|off`，預設為 `off`）
- `agents.defaults.blockStreamingBreak`（`text_end|message_end`）
- `agents.defaults.blockStreamingChunk`（`minChars|maxChars|breakPreference`）
- `agents.defaults.blockStreamingCoalesce`（依閒置狀態批次處理）
- `agents.defaults.humanDelay`（區塊回覆之間模擬真人的停頓）
- 頻道覆寫：內建頻道上的 `*.streaming.block.enabled` 與 `*.streaming.block.coalesce`；過時的扁平金鑰會由 `openclaw doctor --fix` 遷移。所有頻道（包括 Telegram）的區塊串流皆為停用狀態，除非明確啟用。QQ Bot 是例外：它沒有 `streaming.block` 金鑰，且除非 `channels.qqbot.streaming.mode` 為 `"off"`，否則會以串流方式傳送區塊回覆。

詳細資訊：[串流 + 分塊](/zh-TW/concepts/streaming)。

## 推理可見性與權杖

- `/reasoning on|off|stream` 控制可見性。
- 模型產生的推理內容仍會計入權杖用量。
- Telegram 支援將推理內容串流至暫時性的草稿泡泡，並在最終傳遞後刪除；若要持久輸出推理內容，請使用 `/reasoning on`。

詳細資訊：[思考 + 推理指令](/zh-TW/tools/thinking)與[權杖用量](/zh-TW/reference/token-use)。

## 前綴、討論串與回覆

- 輸出前綴位於 `channels.<channel>.responsePrefix` 與 `channels.<channel>.accounts.<id>.responsePrefix`。帳號值優先。當這些標準欄位未設定時，Doctor 會將全域備援值複製到已設定的頻道區塊；`messages.responsePrefix` 仍作為隱含及自訂頻道的備援值。
- 透過 `replyToMode` 及各頻道預設值建立回覆討論串。

詳細資訊：[設定](/zh-TW/gateway/config-agents#messages)及頻道文件。

## 靜默回覆

靜默權杖 `NO_REPLY`（不區分大小寫，因此 `no_reply` 也符合）表示「不要傳遞使用者可見的回覆」。當某次輪次同時有待處理的工具媒體（例如產生的 TTS 音訊）時，OpenClaw 會移除靜默文字，但仍會傳遞媒體附件。

靜默政策依對話類型決定：

- 直接對話永遠不會收到 `NO_REPLY` 提示詞指引。如果直接對話的執行意外傳回單獨的靜默權杖，OpenClaw 會予以抑制，而不會改寫或傳遞。
- 群組／頻道預設允許靜默。在 `message_tool` 可見回覆模式中，靜默表示模型不會呼叫 `message(action=send)`。
- 內部協調流程預設允許靜默。

預設值位於 `agents.defaults.silentReply`；`surfaces.<id>.silentReply` 可依各介面覆寫群組／內部政策。

OpenClaw 也會在非直接聊天中，針對一般內部執行器失敗使用靜默回覆，因此群組／頻道不會看到閘道錯誤的制式文字。具有使用者可見復原說明的已分類失敗，例如缺少驗證、速率限制或過載通知，仍可傳遞。直接聊天預設會顯示精簡的失敗說明；只有啟用 `/verbose full` 時才會顯示原始執行器詳細資料。

所有介面都會捨棄單獨的靜默回覆，因此父工作階段會保持安靜，而不會將哨兵文字改寫成備援對話。

## 相關內容

- [訊息生命週期重構](/zh-TW/concepts/message-lifecycle-refactor) - 以持久可靠的傳送與接收設計為目標
- [串流](/zh-TW/concepts/streaming) - 即時訊息傳遞
- [重試](/zh-TW/concepts/retry) - 訊息傳遞重試行為
- [佇列](/zh-TW/concepts/queue) - 訊息處理佇列
- [頻道](/zh-TW/channels) - 訊息平台整合
