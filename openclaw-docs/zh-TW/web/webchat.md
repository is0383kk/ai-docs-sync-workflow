---
read_when:
    - 偵錯或設定 WebChat 存取權限
summary: 供聊天 UI 使用的迴路 WebChat 靜態主機與閘道 WebSocket 用法
title: WebChat
x-i18n:
    generated_at: "2026-07-26T08:43:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 19c301af1eb1b28650849cdd90924805dd0f5189516693505d9b75f62197007f
    source_path: web/webchat.md
    workflow: 16
---

狀態：macOS/iOS SwiftUI 聊天介面會直接與閘道 WebSocket 通訊。不使用內嵌瀏覽器，也不使用本機靜態伺服器。

## 這是什麼

- 閘道的原生聊天介面。
- 使用與其他頻道相同的工作階段和路由規則。
- 確定性路由：回覆一律傳回 WebChat。
- 歷程記錄一律從閘道擷取（不監看本機檔案）。如果無法連線至閘道，WebChat 會變成唯讀。

## 快速開始

1. 啟動閘道。
2. 開啟 WebChat 介面（macOS/iOS 應用程式）或 Control UI 聊天分頁。
3. 確認已設定有效的閘道驗證路徑（即使在迴路介面上，預設仍使用共用密鑰）。

## 運作方式

- 介面會連線至閘道 WebSocket，並使用 `chat.history`、`chat.send`、`chat.inject` 和 `chat.message.get` RPC 方法。
- `chat.history` 會受到限制以維持穩定性：閘道可能截斷過長的文字欄位、省略大量中繼資料，並將過大的項目替換為 `[chat.history omitted: message too large]`。API 用戶端可在個別請求中傳送 `maxChars`，以針對單次呼叫覆寫預設限制。
- 當可見的助理訊息在 `chat.history` 中遭到截斷時，Control UI 可開啟側邊閱讀器，並透過 `chat.message.get` 視需要擷取完整且經顯示正規化的項目，而不增加預設的歷程記錄承載內容。`chat.message.get` 使用與 `chat.history` 相同的逐字稿分支和顯示規則，但會透過 `messageId` 指定單一項目；當完整內容已無法傳回時，則會如實傳回無法取得的原因。
- `chat.history` 會針對僅附加內容的工作階段檔案沿用作用中的逐字稿分支，因此 WebChat 不會呈現已捨棄的重寫分支和已被取代的提示詞副本。
- 壓縮項目會顯示為「已壓縮的歷程記錄」分隔線，說明壓縮後的逐字稿會保留為檢查點，並提供開啟工作階段檢查點的操作（在權限允許時，可建立分支或還原）。
- Control UI 會記住 `chat.history` 傳回的後端閘道 `sessionId`，並將其納入後續的 `chat.send` 呼叫，因此除非使用者啟動或重設工作階段，否則重新連線和重新整理頁面都會繼續使用同一個已儲存的對話。
- 前景傳送也會將已呈現歷程記錄中顯示分支的葉節點納入 `expectedLeafEntryId`；如果另一個用戶端已先切換分支，Control UI 會暫存訊息以供檢查，並重新整理逐字稿，而不會將訊息發佈至新分支。重新連線及還原寄件匣後的重播，則會在核對目前歷程記錄後刻意省略此前置條件。
- `chat.send` 接受冪等性金鑰（Control UI 使用執行 ID）；閘道會對重複使用相同金鑰的請求進行去重，因此針對相同工作階段、訊息及附件重試或重複提交仍在處理中的請求時，不會建立第二次執行。
- 回覆特定訊息（按一下滑鼠右鍵 → Reply）時，會在 `chat.send` 上將目標的逐字稿 ID 以 `replyToId` 傳送。閘道會從工作階段歷程記錄解析該訊息，並補入 Discord 回覆所使用、與頻道無關的相同回覆情境中繼資料：代理程式會看到 `has_reply_context`，以及包含傳送者標籤和內文的不受信任「目前使用者訊息的回覆目標」區塊。（依據直接 WebChat 工作階段現有的位元組穩定提示詞政策，WebChat 提示詞仍會隱藏 `reply_to_id` 等短暫對話 ID。）沒有持久化逐字稿 ID 的回覆目標（例如等待傳送的訊息）會改為在訊息內文中加入行內引文。
- 工作區啟動檔案和待處理的 `BOOTSTRAP.md` 指示，會透過代理程式系統提示詞的 `# Project Context` 區段提供，而不會複製到 WebChat 使用者訊息中。如果啟動內容遭到截斷，系統提示詞會改為收到簡短的「啟動情境通知」；詳細計數和設定旋鈕則保留在診斷介面上。
- `chat.history` 的顯示正規化會移除：僅供執行階段使用的 OpenClaw 情境、傳入信封包裝、`[[reply_to_current]]`、`[[reply_to:<id>]]` 和 `[[audio_as_voice]]` 等行內傳遞指示標籤、純文字工具呼叫 XML 承載內容（`<tool_call>`、`<function_call>`、`<tool_calls>`、`<function_calls>`，包括遭截斷的區塊），以及外洩的 ASCII／全形模型控制權杖。如果助理項目的所有可見文字僅包含靜默權杖 `NO_REPLY`（不區分大小寫），則會省略該項目。
- 標示為推理的回覆承載內容（`isReasoning: true`）不會納入 WebChat 助理內容、逐字稿重播文字及音訊內容區塊，因此僅含思考內容的承載資料不會顯示為可見的助理訊息或可播放的音訊。
- `chat.inject` 會直接將助理註記附加至逐字稿，並廣播至介面（不執行代理程式）。
- 中止的執行可讓部分助理輸出繼續顯示在介面中。當有緩衝輸出時，閘道會將該部分文字持久化至逐字稿歷程記錄，並以中止中繼資料標記該項目。

### 逐字稿與傳遞模型

WebChat 有兩條獨立的資料路徑：

- SQLite 逐字稿資料列是持久化的模型／執行階段逐字稿。對於一般代理程式執行，內嵌的 OpenClaw 執行階段會透過工作階段存取器，持久化模型可見的 `user`、`assistant` 和 `toolResult` 訊息。WebChat 不會將任意傳遞、狀態或輔助文字寫入該逐字稿。
- 閘道 `ReplyPayload` 事件是即時傳遞投影：已針對 WebChat／頻道顯示、區塊串流、指示標籤、媒體嵌入、TTS／音訊旗標及介面備援行為進行正規化。它們本身並非標準的工作階段記錄。
- 需要透過 `tools.message` 顯示可見回覆的測試框架，仍會將 WebChat 用作目前執行的內部來源回覆接收端。來自該作用中 WebChat 執行且未指定目標的 `message.send`，會投影至同一個聊天並鏡像至工作階段逐字稿；WebChat 不會因此成為可重複使用的對外頻道，也永遠不會繼承 `lastChannel`。
- 只有當閘道在一般內嵌代理程式回合以外擁有顯示訊息時，WebChat 才會插入助理逐字稿項目：`chat.inject`、非代理程式命令回覆、中止後的部分輸出，以及由 WebChat 管理的媒體逐字稿補充內容。
- 如果執行期間出現即時助理文字，但在重新載入歷程記錄後消失，請依序檢查：SQLite 逐字稿是否包含助理文字、`chat.history` 顯示投影是否將其移除，然後檢查 Control UI 的樂觀尾端合併是否以持久化快照取代本機傳遞狀態。

一般代理程式執行的最終答案應具備持久性，因為內嵌執行階段會寫入助理 `message_end`。任何將已傳遞最終承載內容鏡像至逐字稿的備援機制，都必須先避免重複內嵌執行階段已寫入的助理回合。

## Control UI 代理程式工具面板

- Control UI 的 `/agents` Tools 面板具有由 `tools.effective(sessionKey=...)` 支援的「目前可用」檢視：這是由伺服器產生、唯讀的目前工作階段工具清單投影，其中包括核心、外掛、頻道所擁有，以及已探索到的 MCP 伺服器工具。
- 另一個獨立的設定編輯檢視（由 `tools.catalog` 支援）涵蓋設定檔、各代理程式覆寫和目錄語意。
- 執行階段可用性以工作階段為範圍。在同一個代理程式上切換工作階段，可能會變更「目前可用」清單。如果已設定的 MCP 伺服器自上次探索後尚未連線或已發生變更，面板會顯示通知，而不會從讀取路徑默默啟動 MCP 傳輸。
- 設定編輯器不代表執行階段一定可用；實際存取權仍遵循政策優先順序（`allow`/`deny`、各代理程式及提供者／頻道覆寫）。

## 遠端使用

- 遠端模式會透過 SSH/Tailscale 建立閘道 WebSocket 通道。
- 不需要執行獨立的 WebChat 伺服器。

## 設定參考（WebChat）

完整設定：[設定](/zh-TW/gateway/configuration)

WebChat 沒有持久化的設定區段。閘道使用內建的 `chat.history` 顯示限制；API 用戶端可在個別請求中傳送 `maxChars`，以針對單次呼叫覆寫該限制。舊版 `channels.webchat` 和 `gateway.webchat` 設定已停用；請執行 `openclaw doctor --fix` 將其移除。

相關全域選項：

- `gateway.port`、`gateway.bind`：WebSocket 主機／連接埠。
- `gateway.auth.mode`、`gateway.auth.token`、`gateway.auth.password`：
  共用密鑰 WebSocket 驗證。
- `gateway.auth.allowTailscale`：啟用時，瀏覽器 Control UI 聊天分頁可使用 Tailscale
  Serve 身分識別標頭。
- `gateway.auth.mode: "trusted-proxy"`：供位於具身分感知能力的**非迴路** Proxy 來源後方之瀏覽器用戶端使用的反向 Proxy 驗證（請參閱[受信任 Proxy 驗證](/zh-TW/gateway/trusted-proxy-auth)）。
- `gateway.remote.url`、`gateway.remote.token`、`gateway.remote.password`：遠端閘道目標。
- `session.*`：工作階段儲存空間和主要金鑰預設值。

## 相關內容

- [Control UI](/zh-TW/web/control-ui)
- [儀表板](/zh-TW/web/dashboard)
