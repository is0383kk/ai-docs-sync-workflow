---
read_when:
    - 你想要不使用 API 金鑰進行網頁搜尋
    - 你想使用 Parallel 的付費搜尋 API
    - 你需要依大型語言模型上下文效率排序的密集摘錄
summary: 平行搜尋 -- 針對 LLM 最佳化的網路來源密集摘錄
title: 平行搜尋
x-i18n:
    generated_at: "2026-07-26T08:41:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eff693f286015b287bbdacf44f11ff6f07f2f7d2605ef6f09259e7402b40515e
    source_path: tools/parallel-search.md
    workflow: 16
---

Parallel 外掛提供兩個 [Parallel](https://parallel.ai/) `web_search`
提供者，兩者都會從專為 AI 代理程式建立的網頁索引傳回經過排序、針對 LLM 最佳化的摘錄：

| 提供者                 | id              | 驗證                                                                                       |
| ---------------------- | --------------- | ------------------------------------------------------------------------------------------ |
| Parallel Search（免費） | `parallel-free` | 無——Parallel 的免費 [Search MCP](https://docs.parallel.ai/integrations/mcp/search-mcp) |
| Parallel Search        | `parallel`      | `PARALLEL_API_KEY`——付費 Search API，具有更高的速率限制與目標調校功能             |

將 `tools.web.search.provider` 設為 `parallel-free` 或 `parallel`，以明確選取
其中之一；兩者都不會自動偵測。

<Note>
  直接使用 OpenAI Responses 的模型（`api: "openai-responses"`、提供者
  `openai`、官方 API 基礎 URL）會在 `tools.web.search.provider` 未設定、為空值、`"auto"`
  或 `"openai"` 時，自動使用 OpenAI 託管的原生網頁搜尋，
  因此預設會略過 Parallel。若要改為透過 Parallel 路由，請將
  `tools.web.search.provider` 設為 `parallel-free` 或 `parallel`。
  請參閱[網頁搜尋概覽](/zh-TW/tools/web)。
</Note>

## 安裝外掛

```bash
openclaw plugins install @openclaw/parallel-plugin
openclaw gateway restart
```

## API 金鑰（付費提供者）

`parallel-free` 不需要金鑰，但仍必須明確選取。付費
`parallel` 提供者需要 API 金鑰：

<Steps>
  <Step title="建立帳戶">
    在 [platform.parallel.ai](https://platform.parallel.ai) 註冊，並
    從你的儀表板產生 API 金鑰。
  </Step>
  <Step title="儲存金鑰">
    在閘道環境中設定 `PARALLEL_API_KEY`，或透過下列指令設定：

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## 設定

```json5
{
  plugins: {
    entries: {
      parallel: {
        config: {
          webSearch: {
            apiKey: "par-...", // 若已設定 PARALLEL_API_KEY，則為選用
            baseUrl: "https://api.parallel.ai", // 選用；OpenClaw 會附加 /v1/search
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        // 免費 Search MCP 使用 "parallel-free"，此處所示的
        // 付費 API 支援提供者則使用 "parallel"。
        provider: "parallel",
      },
    },
  },
}
```

**環境變數替代方式：**在閘道環境中設定 `PARALLEL_API_KEY`。
若是閘道安裝，請將它放入 `~/.openclaw/.env`。

## 基礎 URL 覆寫

僅適用於付費 `parallel` 提供者；`parallel-free` 一律使用
`https://search.parallel.ai/mcp`，並忽略此設定。

設定 `plugins.entries.parallel.config.webSearch.baseUrl`，以透過相容的 Proxy 或替代端點路由付費
請求（例如 Cloudflare AI Gateway）。OpenClaw 會在裸主機名稱前加上
`https://` 以進行正規化，並附加 `/v1/search`，除非路徑已以該字串結尾。解析後的
端點是搜尋快取鍵的一部分，因此不同端點的結果絕不會共用。

## 工具參數

兩個提供者都公開 Parallel 的原生搜尋格式，讓模型填入
自然語言目標及數個簡短的關鍵字查詢——這是 Parallel 為取得
最佳結果而[建議](https://docs.parallel.ai/search/best-practices)的搭配方式。

<ParamField path="objective" type="string" required>
基礎問題或目標的自然語言描述（最多 5000
個字元）。內容應可獨立理解。
</ParamField>

<ParamField path="search_queries" type="string[]" required>
簡潔的關鍵字搜尋查詢，每項 3-6 個字詞（1-5 項，每項最多 200 個字元）。
為獲得最佳結果，請提供 2-3 個多樣化的查詢。
</ParamField>

<ParamField path="count" type="number">
要傳回的結果數（1-40）。
</ParamField>

<ParamField path="session_id" type="string">
上一個結果之 `sessionId` 中的選用 Parallel 工作階段 ID。在同一任務的
後續搜尋中傳入此 ID，讓 Parallel 將相關呼叫分組，並改善後續結果。
`parallel` 的上限為 1000 個字元；免費的
`parallel-free` Search MCP 將其限制為 100 個字元。超過上限的 ID 會被捨棄
（付費），或改為產生新的 ID（免費）。
</ParamField>

<ParamField path="client_model" type="string">
發出呼叫之模型的選用識別碼（例如 `claude-opus-4-7`、
`gpt-5.6-sol`），最多 100 個字元。讓 Parallel 能依據你的
模型能力調整預設設定。請傳入目前使用中模型的完整 slug；不要縮短為
系列別名。
</ParamField>

## 注意事項

- Parallel 會依照對 LLM 推理的實用性來排序並壓縮結果，而非供人類
  點閱；每個結果會包含密集的摘錄，而非完整頁面
  內容。
- 結果摘錄會以 `excerpts` 陣列傳回，也會合併至
  `description`，以相容於通用的 `web_search` 合約。
- 兩個提供者都會傳回 `session_id`；OpenClaw 會在
  工具承載資料中將其公開為 `sessionId`，讓呼叫端能將後續搜尋分組。由
  Parallel 產生的工作階段 ID（並非由呼叫端提供）會從快取項目中排除，
  因為查詢相同但互不相關的任務不應繼承該 ID。
- Parallel 傳回的 `searchId`、`warnings` 和 `usage` 會在
  存在時原樣傳遞。
- OpenClaw 一律會將解析後的結果數量以
  `advanced_settings.max_results`（`parallel`）轉送至 Parallel，或在 Parallel 傳回固定大小的回應
  （`parallel-free`）後，於用戶端套用 `count`。
  呼叫端的 `count` 引數優先，其次為 `tools.web.search.maxResults`，否則使用
  OpenClaw 的通用 `web_search` 預設值（5）——Parallel 自有 API 的預設值
  為 10。
- 結果預設會快取 15 分鐘（`cacheTtlMinutes`）。
- 當呼叫端未提供時，`parallel-free` 會透過其 MCP 交握，為每次呼叫
  產生新的 `session_id`；在此情況下，`parallel` 會使其保持未設定。

## 相關內容

- [網頁搜尋概覽](/zh-TW/tools/web)——所有提供者與自動偵測
- [Exa 搜尋](/zh-TW/tools/exa-search)——具備內容擷取功能的神經搜尋
- [Perplexity Search](/zh-TW/tools/perplexity-search)——支援網域篩選的結構化結果
