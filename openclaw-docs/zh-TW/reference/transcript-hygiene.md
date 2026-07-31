---
read_when:
    - 你正在偵錯與逐字稿結構有關的提供者請求遭拒問題
    - 你正在變更對話記錄清理或工具呼叫修復邏輯
    - 你正在調查不同供應商之間的工具呼叫 ID 不相符問題
summary: 參考：供應商專用的逐字稿清理與修復規則
title: 對話記錄整理原則
x-i18n:
    generated_at: "2026-07-26T07:56:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 33d978772062cb2a81eb358bb5c62bd1261b433ffdc8acdbaa6679b121fbbf62
    source_path: reference/transcript-hygiene.md
    workflow: 16
---

OpenClaw 會在執行前（建構模型上下文時）對轉錄內容套用**供應商專屬修正**。其中大多數是為滿足嚴格的供應商要求而進行的**記憶體內**調整。另一個工作階段檔案修復階段也可能在載入工作階段前重寫儲存的 JSONL，但僅限格式錯誤的行，或不是有效持久記錄的已保存對話輪次。
已傳送的助理回覆會保留在磁碟上；供應商專屬的助理預填內容移除只會在建構外送承載資料時進行。

進行修復時，原始檔案會在不可分割取代前寫入暫時的
`*.bak-<pid>-<ts>` 同層檔案，並於取代成功後移除。只有在清理本身失敗時才會保留備份，
此時會回報其路徑。

範圍包括：

- 僅供執行階段使用的提示上下文不會進入使用者可見的轉錄輪次
- 工具呼叫 ID 清理
- 工具呼叫輸入驗證
- 工具結果配對修復
- 輪次驗證／排序
- 思緒簽章清理
- 思考簽章清理
- 圖片承載資料清理
- 供應商重播前的空白文字區塊清理
- 供應商重播前的不完整、僅推理長度終止輪次清理
- 使用者輸入來源標記（用於跨工作階段路由的提示）
- Bedrock Converse 重播的空白助理錯誤輪次修復

如需轉錄儲存詳細資訊，請參閱
[工作階段管理深入解析](/zh-TW/reference/session-management-compaction)。

---

## 全域規則：執行階段上下文不是使用者轉錄內容

執行階段／系統上下文可以加入某一輪的模型提示，但並非終端使用者撰寫的內容。OpenClaw 會為閘道回覆、佇列中的後續訊息、ACP、命令列介面和嵌入式 OpenClaw 執行保留獨立的轉錄用提示本文。儲存的可見使用者輪次會使用該轉錄本文，而非經執行階段資訊擴充的提示。

對於已保存執行階段包裝內容的舊版工作階段，閘道歷程介面會先套用顯示投影，再將訊息傳回 WebChat、終端介面、REST 或 SSE 用戶端。

---

## 執行位置

所有轉錄內容維護都集中在嵌入式執行器中：

- 原則選擇：`src/agents/transcript-policy.ts`
  （`resolveTranscriptPolicy`，依據 `provider`、`modelApi` 和 `modelId`）
- 清理／修復套用：`sanitizeSessionHistory`，位於
  `src/agents/embedded-agent-runner/replay-history.ts`

除轉錄內容維護外，也會在載入前修復工作階段檔案（如有需要）：

- `repairSessionFileIfNeeded`，位於 `src/agents/session-file-repair.ts`
- 由 `src/agents/embedded-agent-runner/run/attempt.ts` 和
  `src/agents/embedded-agent-runner/compact.ts` 呼叫

---

## 全域規則：圖片清理

一律會清理圖片承載資料，以避免因大小限制而遭供應商拒絕（縮小／重新壓縮過大的 base64 圖片）。這也有助於控制具備視覺能力的模型因圖片產生的權杖壓力：較低的最大尺寸可減少權杖用量，較高的尺寸則可保留細節。

實作：

- `sanitizeSessionMessagesImages`，位於
  `src/agents/embedded-agent-helpers/images.ts`
- `sanitizeContentBlocksImages`，位於 `src/agents/tool-images.ts`
- 圖片邊長上限可透過 `agents.defaults.imageMaxDimensionPx` 設定
  （預設值：`1200`）
- 此階段巡覽重播內容時，也會移除空白文字區塊。
  因此變成空白的助理輪次會從重播副本中捨棄；變成空白的使用者
  和工具結果輪次則會收到非空白的內容省略預留位置。

---

## 全域規則：格式錯誤的工具呼叫

在建構模型上下文前，會捨棄同時缺少 `input` 和 `arguments` 的助理工具呼叫區塊。這可防止供應商因部分保存的工具呼叫而拒絕要求（例如在速率限制失敗後）。

實作：

- `sanitizeToolCallInputs`，位於 `src/agents/session-transcript-repair.ts`
- 套用於 `sanitizeSessionHistory`
  （`src/agents/embedded-agent-runner/replay-history.ts`）

---

## 全域規則：工具結果配對

在改寫供應商專屬的呼叫 ID 前，會將每個助理輪次中的工具結果與工具呼叫出現項配對。供應商產生的 ID 可能在後續輪次中重複，因此與重複呼叫相鄰的結果仍會歸屬於該次出現項。只有在恰好有一個尚未解析的出現項可擁有結果時，才會移動錯置的結果；有歧義的多餘結果會被捨棄，而缺少結果的出現項會收到合成的錯誤結果。

實作：`sanitizeToolUseResultPairing`，位於
`src/agents/session-transcript-repair.ts`

---

## 全域規則：不完整或無聲的僅推理輪次

發生下列任一事件後，如果助理輪次只包含思考或經遮蔽的思考內容，該輪次就會從記憶體內的重播副本中省略：

- 供應商輸出限制使輪次以不完整的推理狀態結束。
- 無聲回覆清理移除了該輪次唯一可見的 `NO_REPLY` 文字。

無聲回覆清理可防止嚴格的供應商重建對話時，隱藏推理合併至後續的助理工具使用輪次。

空白的長度終止輪次不會變更，包含可見文字、工具呼叫或未知內容區塊的長度終止輪次也不會變更。包含工具呼叫或未知內容區塊的無聲回覆輪次同樣不會變更。不會重寫儲存的轉錄內容。

實作：`normalizeAssistantReplayContent`，位於
`src/agents/embedded-agent-runner/replay-history.ts`

---

## 全域規則：跨工作階段輸入來源

當代理程式透過 `sessions_send` 將提示傳送至另一個工作階段時
（包括代理程式間的回覆／公告步驟），OpenClaw 會以 `message.provenance.kind = "inter_session"` 保存所建立的使用者輪次。

OpenClaw 也會在路由的提示文字前加上同一輪次的 `[Inter-session message] ... isUser=false`
標記，讓目前的模型呼叫能夠區分外部工作階段輸出與外部終端使用者指示。此標記會在可用時包含來源工作階段、頻道和工具。為了供應商相容性，轉錄內容仍會使用 `role: "user"`，但可見文字和來源中繼資料都會將該輪標記為跨工作階段資料。

重建上下文時，OpenClaw 會將相同標記套用至只有來源中繼資料的舊版已保存跨工作階段使用者輪次。

---

## 供應商矩陣（目前行為）

**OpenAI／OpenAI Codex**

- 僅清理圖片。
- 對 OpenAI Responses／Codex 轉錄內容，捨棄孤立的推理簽章（後方沒有內容區塊的獨立推理項目），並在模型路由切換後捨棄可重播的 OpenAI 推理。
- 保留可重播的 OpenAI Responses 推理項目承載資料，包括摘要為空白的加密項目，以便手動／WebSocket 重播時，必要的 `rs_*` 狀態能與助理輸出項目保持配對。
- 原生 ChatGPT Codex Responses 會遵循 Codex 線路一致性，在不含先前項目 ID 的情況下重播先前的 Responses 推理／訊息／函式承載資料，同時保留工作階段 `prompt_cache_key`。
- OpenAI Responses 系列重播會保留標準的 `call_*|fc_*` 同模型推理配對，但會在 pi-ai 承載資料轉換前，以確定性方式正規化格式錯誤或過長的 `call_id`／函式呼叫項目 ID。
- 工具結果配對修復可能會移動實際相符的輸出，並為缺少結果的工具呼叫合成 Codex 樣式的 `aborted` 輸出。
- 不進行輪次驗證或重新排序；不移除思緒簽章。

**OpenAI 相容的 Chat Completions**

- 重播前會移除歷史助理思考／推理區塊，讓本機和代理式 OpenAI 相容伺服器不會收到先前輪次的推理欄位，例如 `reasoning` 或 `reasoning_content`。
- 目前同一輪次中的工具呼叫延續內容，會讓助理推理區塊保持附加於工具呼叫，直到工具結果完成重播為止。
- 具有 `reasoning: true` 的自訂／自行託管模型項目會保留重播的推理中繼資料。
- 當線路通訊協定需要重播推理中繼資料時，供應商所擁有的例外可選擇不套用此規則。

**Google（Generative AI／Gemini CLI／Antigravity）**

- 工具呼叫 ID 清理：嚴格限制為英數字元。
- 工具結果配對修復及合成工具結果。
- 輪次驗證（Gemini 樣式的輪次交替）。
- Google 輪次排序修正（如果歷程以助理開始，則在前方加入極短的使用者啟動訊息）。
- Antigravity Claude：正規化思考簽章；捨棄未簽署的思考區塊。

**Anthropic／Minimax（Anthropic 相容）**

- 工具結果配對修復及合成工具結果。
- 輪次驗證（合併連續的使用者輪次，以滿足嚴格的交替要求）。
- 啟用思考時，會從外送的 Anthropic Messages 承載資料中移除尾端的助理預填輪次，包括 Cloudflare AI 閘道路由。
- 當工作階段已壓縮時，會在供應商重播前移除壓縮前的助理思考簽章。思考簽章在產生時以密碼學方式繫結至對話前綴；壓縮後前綴會變更（摘要內容取代原始內容），因此重播原始簽章會導致 Anthropic 以 “Invalid signature in thinking block” 拒絕要求。思考文字會保留為未簽署區塊，再由下方規則處理。
- 重播簽章缺少、空白或僅含空格的思考區塊，會在供應商轉換前移除。如果這使助理輪次變成空白，OpenClaw 會使用非空白的推理省略文字維持輪次結構。
- 必須移除的舊版僅思考助理輪次，會替換為非空白的推理省略文字，避免供應商轉接器捨棄該重播輪次。

**Amazon Bedrock（Converse API）**

- 重播前會將空白的助理串流錯誤輪次修復為非空白的備援文字區塊。Bedrock Converse 會拒絕帶有 `content: []` 的助理訊息，因此內容為空白且具有 `stopReason:
"error"` 的已保存助理輪次，也會在載入前於磁碟上修復。
- 只包含空白文字區塊的助理串流錯誤輪次，會從記憶體內的重播副本中捨棄，而不是重播無效的空白區塊。
- 當工作階段已壓縮時，會在 Converse 重播前移除壓縮前的助理思考簽章，原因與上方 Anthropic 相同。
- 重播簽章缺少、空白或僅含空格的 Claude 思考區塊，會在 Converse 重播前移除。如果這使助理輪次變成空白，OpenClaw 會使用非空白的推理省略文字維持輪次結構。
- 必須移除的舊版僅思考助理輪次，會替換為非空白的推理省略文字，使 Converse 重播能維持嚴格的輪次結構。
- 重播時會篩除 OpenClaw 傳送鏡像和閘道注入的助理輪次。
- 透過全域規則套用圖片清理。

**Mistral（包括依模型 ID 偵測）**

- 工具呼叫 ID 清理：strict9（英數字元，長度為 9）。

**OpenRouter Gemini**

- 思緒簽章清理：移除非 base64 的 `thought_signature` 值
  （保留 base64）。

**OpenRouter Anthropic**

- 啟用推理時，會從已驗證的 OpenRouter OpenAI 相容 Anthropic 模型承載資料中移除尾端的助理預填輪次，以符合直接 Anthropic 和 Cloudflare Anthropic 的重播行為。

**其他所有項目**

- 僅清理圖片。

---

## 歷史行為（2026.1.22 之前）

在 2026.1.22 版本之前，OpenClaw 會套用多層轉錄內容維護：

- 一個 **transcript-sanitize 擴充功能**會在每次建構情境時執行，並且可以：
  - 修復工具使用與結果的配對。
  - 清理工具呼叫 ID（包括保留
    `_`/`-` 的非嚴格模式）。
- 執行器也會執行供應商特定的清理，因而
  重複處理相同工作。
- 供應商原則之外還會發生其他變更，包括
  在持久化前從助理文字中移除 `<final>` 標籤、捨棄
  空白的助理錯誤輪次，以及裁切工具
  呼叫後的助理內容。

這種複雜性造成跨供應商的迴歸問題（尤其是
`openai-responses` `call_id|fc_id` 配對）。2026.1.22 的清理移除了
該擴充功能、將邏輯集中到執行器，並使 OpenAI 除了影像清理之外
**完全不做任何變更**。

## 相關內容

- [工作階段管理](/zh-TW/concepts/session)
- [工作階段修剪](/zh-TW/concepts/session-pruning)
