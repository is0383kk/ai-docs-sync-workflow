---
read_when:
    - 你想要使用 Exa 進行 web_search
    - 你需要 EXA_API_KEY
    - 你想要神經搜尋或內容擷取
summary: Exa AI 搜尋——具備內容擷取功能的神經與關鍵字搜尋
title: Exa 搜尋
x-i18n:
    generated_at: "2026-07-26T08:49:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3ddfd6fb471f92e705facf5a2d02361c1a343b9032fa8e0a7b135af634df65b7
    source_path: tools/exa-search.md
    workflow: 16
---

[Exa AI](https://exa.ai/) 是一個 `web_search` 提供者，具備神經、關鍵字及
混合搜尋模式，並內建內容擷取功能（重點、文字、
摘要）。

## 安裝外掛

```bash
openclaw plugins install @openclaw/exa-plugin
openclaw gateway restart
```

## 取得 API 金鑰

<Steps>
  <Step title="建立帳號">
    在 [exa.ai](https://exa.ai/) 註冊，並從你的
    儀表板產生 API 金鑰。
  </Step>
  <Step title="儲存金鑰">
    在閘道環境中設定 `EXA_API_KEY`，或透過以下方式設定：

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
      exa: {
        config: {
          webSearch: {
            apiKey: "exa-...", // 若已設定 EXA_API_KEY，則為選用
            baseUrl: "https://api.exa.ai", // 選用；OpenClaw 會附加 /search
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "exa",
      },
    },
  },
}
```

**環境變數替代方案：**在閘道環境中設定 `EXA_API_KEY`。若是
閘道安裝，請將其放入 `~/.openclaw/.env`。請參閱
[環境變數](/zh-TW/help/faq#env-vars-and-env-loading)。

## 覆寫基底 URL

設定 `plugins.entries.exa.config.webSearch.baseUrl`，即可透過相容的 Proxy 或替代端點路由 Exa 搜尋
請求。OpenClaw 會在裸主機名稱前加上 `https://` 以進行正規化，並附加 `/search`，除非
路徑已以此結尾。解析後的端點是搜尋
快取鍵的一部分，因此絕不會共用來自不同端點的結果。

## 工具參數

<ParamField path="query" type="string" required>
搜尋查詢。
</ParamField>

<ParamField path="count" type="number" default="5">
要傳回的結果數量（1-100，受 Exa 搜尋類型限制）。
</ParamField>

<ParamField path="type" type="'auto' | 'neural' | 'fast' | 'deep' | 'deep-reasoning' | 'instant'">
搜尋模式。
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
時間篩選條件。不可與 `date_after`/`date_before` 結合使用。
</ParamField>

<ParamField path="date_after" type="string">
此日期之後的結果（`YYYY-MM-DD`）。
</ParamField>

<ParamField path="date_before" type="string">
此日期之前的結果（`YYYY-MM-DD`）。
</ParamField>

<ParamField path="contents" type="object">
內容擷取選項（見下文）。
</ParamField>

### 內容擷取

傳入 `contents` 物件，以控制結果中擷取的內容：

```javascript
await web_search({
  query: "transformer architecture explained",
  type: "neural",
  contents: {
    text: true, // 完整頁面文字
    highlights: { numSentences: 3 }, // 關鍵句子
    summary: true, // AI 摘要
  },
});
```

| 內容選項 | 類型                                                                  | 說明            |
| --------------- | --------------------------------------------------------------------- | ---------------------- |
| `text`          | `boolean \| { maxCharacters }`                                        | 擷取完整頁面文字 |
| `highlights`    | `boolean \| { maxCharacters, query, numSentences, highlightsPerUrl }` | 擷取關鍵句子  |
| `summary`       | `boolean \| { query }`                                                | AI 產生的摘要   |

若省略 `contents`，Exa 預設使用 `{ highlights: true }`，因此結果
會包含關鍵句子摘錄。結果說明會依序採用重點、
摘要及完整文字，以最先可用者為準。結果
也會保留 Exa API 回應中原始的 `highlightScores` 與 `summary` 欄位（若有）。

### 搜尋模式

| 模式             | 說明                       |
| ---------------- | --------------------------------- |
| `auto`           | Exa 選擇最佳模式（預設） |
| `neural`         | 語意／基於含義的搜尋     |
| `fast`           | 快速關鍵字搜尋              |
| `deep`           | 全面深入搜尋              |
| `deep-reasoning` | 具推理功能的深入搜尋        |
| `instant`        | 最快取得結果                   |

## 注意事項

- `count` 最多接受 100，受 Exa 搜尋類型限制。
- 結果預設快取 15 分鐘。設定共用的
  `tools.web.search.cacheTtlMinutes`（分鐘）與
  `tools.web.search.timeoutSeconds`（預設 30 秒），即可變更所有 `web_search` 提供者（包括 Exa）的快取與
  請求逾時設定。

## 相關內容

- [網頁搜尋概覽](/zh-TW/tools/web) -- 所有提供者及自動偵測
- [Brave Search](/zh-TW/tools/brave-search) -- 提供國家／語言篩選的結構化結果
- [Perplexity Search](/zh-TW/tools/perplexity-search) -- 提供網域篩選的結構化結果
