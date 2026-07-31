---
read_when:
    - 你想了解哪些功能可能會呼叫付費 API
    - 你需要稽核金鑰、成本和用量可見性
    - 你正在說明 `/status` 或 `/usage` 的費用報告功能
summary: 稽核哪些項目可能產生費用、使用了哪些金鑰，以及如何檢視用量
title: API 使用量與費用
x-i18n:
    generated_at: "2026-07-26T08:42:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 22caad8b8fa168739563223b3663a04adceeef7e83576a53dc9cdf885a35750d
    source_path: reference/api-usage-costs.md
    workflow: 16
---

OpenClaw 中可呼叫付費供應商 API 的功能對照、各功能讀取認證資訊的位置，以及所產生費用的顯示位置。

## 費用顯示位置

**`/status`**（每個工作階段的快照）

- 顯示目前工作階段的模型、情境用量，以及上一則回應的權杖數。
- 當 OpenClaw 具有用量中繼資料及作用中模型的本機定價時，會加入上一則回覆的**估算費用**，包括具有明確定價且不使用 API 金鑰的供應商，例如 Bedrock `aws-sdk` 模型。
- 如果即時工作階段快照的資料不足，`/status` 會從最新的逐字稿用量項目復原權杖／快取計數器和作用中模型標籤。現有的非零即時值優先於逐字稿資料；當儲存的總數缺漏或較小時，與提示詞大小相當的逐字稿總數仍可優先採用。

**`/usage`**（每則訊息的頁尾）

- `/usage full` 會在每則回覆後附加用量頁尾；若已設定本機定價且有可用的用量中繼資料，也會包含**估算費用**。
- `/usage tokens` 僅顯示權杖數。訂閱型 OAuth／權杖及命令列介面執行階段只會顯示權杖數，除非它們提供相容的用量中繼資料及明確的本機價格。
- `/usage cost` 會輸出本機費用摘要；`/usage off` 會停用頁尾。
- Gemini 命令列介面注意事項：`stream-json` 和舊版 `json` 輸出都會在 `stats` 下提供用量。OpenClaw 會將 `stats.cached` 正規化為 `cacheRead`，並在需要時從 `stats.input_tokens - stats.cached` 推導輸入權杖數。

**Control UI → Usage**（跨工作階段分析）

- 顯示所選日期範圍內由逐字稿推導的權杖總數及估算費用總額，並依供應商、模型、代理程式、頻道和權杖類型細分。
- 比較以所選範圍結束日期為終點的較短日曆期間。缺少的日期會計為零用量日曆日，不會略過這些日期以建立較密集的期間。
- 直接標示每日圖表的刻度。`√` 徽章表示正在使用平方根壓縮，讓低用量日期仍保持可見。
- 這些總數描述可用的本機工作階段歷程，並非供應商發票或終身計費帳本。部分項目缺少定價時，UI 會發出警告。

**命令列介面用量期間**（供應商配額，而非每則訊息的費用）

- `openclaw status --usage` 和 `openclaw channels list` 會將供應商的**用量期間**顯示為 `X% left`。
- 目前支援用量期間的供應商：Anthropic、ClawRouter、DeepSeek、GitHub Copilot、Gemini 命令列介面、MiniMax、OpenAI（涵蓋 ChatGPT／Codex OAuth／權杖驗證）、Xiaomi 和 z.ai。完整的供應商／旗標清單請參閱[模型命令列介面](/zh-TW/cli/models)和[頻道命令列介面](/zh-TW/cli/channels)。
- MiniMax 的原始 `usage_percent`／`usagePercent` 欄位回報剩餘配額，因此 OpenClaw 會將其反轉；若存在以計數為基礎的欄位，則以其為準。如果回應包含 `model_remains` 陣列，OpenClaw 會選擇聊天模型項目、在必要時從時間戳記推導期間標籤，並在方案標籤中納入模型名稱。
- 若有供應商專屬鉤子，會從中取得用量驗證；否則 OpenClaw 會改為使用驗證設定檔、環境或設定中相符的 OAuth／API 金鑰認證資訊。

詳細範例請參閱[權杖使用量與費用](/zh-TW/reference/token-use)。

<Note>
Anthropic 已確認，除非發布新政策，否則重複使用 Claude 命令列介面（包括 `claude -p`）是受認可的整合模式。Anthropic 不提供每則訊息的金額估算，因此 `/usage full` 無法顯示 Claude 命令列介面用量的費用。
</Note>

## 金鑰的探索方式

- **驗證設定檔**：每個代理程式各自儲存於 `auth-profiles.json`。
- **環境變數**：例如 `OPENAI_API_KEY`、`BRAVE_API_KEY`、`FIRECRAWL_API_KEY`。
- **設定**：`models.providers.*.apiKey`、`plugins.entries.*.config.webSearch.apiKey`、`plugins.entries.firecrawl.config.webFetch.apiKey`、`memory.search.*`、`talk.providers.*.apiKey`。
- **Skills**：`skills.entries.<name>.apiKey`，可將金鑰匯出至 Skill 程序的環境。

## 可能使用金鑰並產生費用的功能

### 核心模型回應（聊天與工具）

每則回覆或工具呼叫都會在目前的模型供應商上執行。這是用量和費用的主要來源，包括在 OpenClaw 本機 UI 之外計費的訂閱型託管方案：OpenAI Codex、Alibaba Cloud Model Studio Coding Plan、MiniMax Coding Plan、Z.AI／GLM Coding Plan，以及已啟用 Extra Usage 的 Anthropic Claude 登入途徑。

定價設定請參閱[模型](/zh-TW/providers/models)，顯示方式請參閱[權杖使用量與費用](/zh-TW/reference/token-use)。

### 媒體理解（音訊／圖片／影片）

在回覆管線執行前，可透過供應商 API 摘要或轉錄傳入的媒體。各外掛會分別登錄所支援的供應商，且清單會隨新增外掛而變更；目前的清單與設定請參閱[媒體理解](/zh-TW/nodes/media-understanding)。

### 圖片與影片生成

`image_generate` 和 `video_generate` 會路由至任何可用且已驗證的供應商。若其 `agents.defaults.mediaModels` 項目尚未設定，兩者皆可推斷以驗證資訊為基礎的預設供應商。

目前的供應商清單請參閱[圖片生成](/zh-TW/tools/image-generation)和[影片生成](/zh-TW/tools/video-generation)。

### 記憶嵌入與語意搜尋

當 `memory.search.provider` 指定遠端配接器（例如 `openai`、`gemini`、`voyage`、`mistral`、`deepinfra`、`github-copilot`、`amazon-bedrock`）時，語意記憶搜尋會使用嵌入 API。`memory.search.provider = "lmstudio"` 或 `"ollama"` 會針對本機／自行託管的伺服器執行，通常不會產生託管服務費用。`memory.search.provider = "local"` 會將所有作業保留在裝置上，不使用 API。選用的 `memory.search.fallback` 供應商可在本機嵌入失敗時接手處理。

請參閱[記憶](/zh-TW/concepts/memory)。

### 網頁搜尋工具

`web_search` 可能會依所選供應商產生用量費用。每個供應商會先從環境變數讀取金鑰，再從 `plugins.entries.<id>.config.webSearch.apiKey` 讀取：

| 供應商               | 環境變數                                                                                                                                                             |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Brave Search           | `BRAVE_API_KEY`                                                                                                                                                        |
| DuckDuckGo             | 不需要金鑰；非官方、以 HTML 為基礎、不收費                                                                                                                           |
| Exa                    | `EXA_API_KEY`                                                                                                                                                          |
| Firecrawl              | `FIRECRAWL_API_KEY`                                                                                                                                                    |
| Gemini (Google Search) | `GEMINI_API_KEY`                                                                                                                                                       |
| Grok (xAI)             | xAI OAuth 設定檔或 `XAI_API_KEY`                                                                                                                                     |
| Kimi (Moonshot)        | `KIMI_API_KEY` 或 `MOONSHOT_API_KEY`                                                                                                                                   |
| MiniMax Search         | `MINIMAX_CODE_PLAN_KEY`、`MINIMAX_CODING_API_KEY`、`MINIMAX_OAUTH_TOKEN` 或 `MINIMAX_API_KEY`                                                                         |
| Ollama Web Search      | 可連線且已登入的本機主機不需要金鑰；直接使用 `https://ollama.com` 搜尋時會使用 `OLLAMA_API_KEY`；受驗證保護的主機會重複使用一般 Ollama 供應商的持有人驗證 |
| Parallel               | `PARALLEL_API_KEY`                                                                                                                                                     |
| Perplexity Search API  | `PERPLEXITY_API_KEY` 或 `OPENROUTER_API_KEY`                                                                                                                           |
| SearXNG                | `SEARXNG_BASE_URL`；不需要金鑰／自行託管，不會產生託管服務費用                                                                                                            |
| Tavily                 | `TAVILY_API_KEY`                                                                                                                                                       |

舊版 `tools.web.search.*` 設定路徑仍可透過相容性轉接層載入，但已不再是建議使用的介面。

**Brave Search 免費額度**：每個方案每月都包含 $5 的循環免費額度。Search 方案每 1,000 次要求收費 $5，因此該額度每月可免費涵蓋 1,000 次要求。請在 Brave 儀表板中設定用量上限，以避免非預期費用。

請參閱[網頁工具](/zh-TW/tools/web)。

### 網頁擷取工具（Firecrawl）

`web_fetch` 可透過免金鑰入門存取呼叫 Firecrawl；加入 `FIRECRAWL_API_KEY`（或 `plugins.entries.firecrawl.config.webFetch.apiKey`）可取得更高限額。如果未設定 Firecrawl，此工具會改用直接擷取及隨附的 `web-readability` 外掛（不使用付費 API）。停用 `plugins.entries.web-readability.enabled` 可略過本機 Readability 擷取。

請參閱[網頁工具](/zh-TW/tools/web)。

### 供應商用量快照（狀態／健康情況）

`openclaw status --usage` 和 `openclaw models status --json` 會呼叫供應商用量端點，以顯示配額期間或驗證健康情況。呼叫量很低，但仍會存取供應商 API。

請參閱[模型命令列介面](/zh-TW/cli/models)。

### 壓縮防護摘要

壓縮防護可使用目前的模型摘要工作階段歷程，執行時會呼叫供應商 API。

請參閱[工作階段管理與壓縮](/zh-TW/reference/session-management-compaction)。

### 模型掃描／探測

`openclaw models scan` 可探測 OpenRouter 模型，並在啟用探測時使用 `OPENROUTER_API_KEY`。

請參閱[模型命令列介面](/zh-TW/cli/models)。

### 對話（語音）

設定完成後，對話模式可呼叫 ElevenLabs：`ELEVENLABS_API_KEY` 或 `talk.providers.elevenlabs.apiKey`。

請參閱[對話模式](/zh-TW/nodes/talk)。

### Skills（第三方 API）

Skills 可將 `apiKey` 儲存在 `skills.entries.<name>.apiKey` 中。如果 Skill 使用該金鑰存取外部 API，費用會依該 Skill 的供應商計算。

請參閱[Skills](/zh-TW/tools/skills)。

## 相關內容

- [權杖使用量與費用](/zh-TW/reference/token-use)
- [提示詞快取](/zh-TW/reference/prompt-caching)
- [用量追蹤](/zh-TW/concepts/usage-tracking)
