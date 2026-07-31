---
read_when:
    - 你想要將 Brave Search 用於 `web_search`
    - 你需要 BRAVE_API_KEY 或方案詳細資訊
summary: 用於 web_search 的 Brave Search API 設定
title: Brave 搜尋
x-i18n:
    generated_at: "2026-07-26T08:40:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52168db93abb564eda5868584261e0530ce3cff57c3463a2fc1eded351df30f2
    source_path: tools/brave-search.md
    workflow: 16
---

OpenClaw 支援 Brave Search API 作為 `web_search` 提供者。

## 取得 API 金鑰

1. 在 [https://brave.com/search/api/](https://brave.com/search/api/) 建立 Brave Search API 帳號
2. 在儀表板中選擇 **Search** 方案，並產生 API 金鑰。
3. 將金鑰儲存在設定中，或在閘道環境中設定 `BRAVE_API_KEY`。

## 設定範例

```json5
{
  plugins: {
    entries: {
      brave: {
        config: {
          webSearch: {
            apiKey: "BRAVE_API_KEY_HERE",
            mode: "web", // 或 "llm-context"
            baseUrl: "https://api.search.brave.com", // 選用的 Proxy／基底 URL 覆寫值
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "brave",
        maxResults: 5,
        timeoutSeconds: 30,
      },
    },
  },
}
```

Brave 提供者專屬的搜尋設定位於 `plugins.entries.brave.config.webSearch.*`；這是標準設定路徑。

`webSearch.mode` 控制 Brave 傳輸模式：

- `web`（預設）：一般 Brave 網頁搜尋，包含標題、URL 和摘要
- `llm-context`：Brave LLM Context API，提供預先擷取的文字區塊和來源，以便建立依據

`webSearch.baseUrl` 可將 Brave 請求指向受信任且與 Brave 相容的 Proxy
或閘道。OpenClaw 會將 `/res/v1/web/search` 或 `/res/v1/llm/context` 附加至
設定的基底 URL，並將基底 URL 納入快取鍵。公開
端點必須使用 `https://`；只有受信任的迴路
或私人網路 Proxy 主機才可使用 `http://`。

## 工具參數

<ParamField path="query" type="string" required>
搜尋查詢。
</ParamField>

<ParamField path="count" type="number" default="5">
要傳回的結果數量（1–10）。
</ParamField>

<ParamField path="country" type="string">
2 個字母的 ISO 國家代碼（例如 `US`、`DE`）。
</ParamField>

<ParamField path="language" type="string">
搜尋結果的 ISO 639-1 語言代碼（例如 `en`、`de`、`fr`）。
</ParamField>

<ParamField path="search_lang" type="string">
Brave 搜尋語言代碼（例如 `en`、`en-gb`、`zh-hans`）。
</ParamField>

<ParamField path="ui_lang" type="string">
UI 元素的 ISO 語言代碼。
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
時間篩選條件 — `day` 代表 24 小時。
</ParamField>

<ParamField path="date_after" type="string">
只傳回在此日期之後發布的結果（`YYYY-MM-DD`）。
</ParamField>

<ParamField path="date_before" type="string">
只傳回在此日期之前發布的結果（`YYYY-MM-DD`）。
</ParamField>

**範例：**

```javascript
// 依國家和語言搜尋
await web_search({
  query: "renewable energy",
  country: "DE",
  language: "de",
});

// 最近的結果（過去一週）
await web_search({
  query: "AI news",
  freshness: "week",
});

// 依日期範圍搜尋
await web_search({
  query: "AI developments",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});
```

## 注意事項

- OpenClaw 使用 Brave **Search** 方案。如果你使用舊版訂閱（例如原始的 Free 方案，每月可查詢 2,000 次），該訂閱仍然有效，但不包含 LLM Context 或更高速率限制等較新功能。
- 每個 Brave 方案均包含**每月 \$5 的免費點數**（定期更新）。Search 方案每 1,000 個請求收費 \$5，因此該點數可支付每月 1,000 次查詢。請在 Brave 儀表板中設定用量限制，以免產生非預期費用。如需目前的方案資訊，請參閱 [Brave API 入口網站](https://brave.com/search/api/)。
- Search 方案包含 LLM Context 端點和 AI 推論權利。若要儲存結果以訓練或調校模型，必須使用明確授予儲存權利的方案。請參閱 Brave [服務條款](https://api-dashboard.search.brave.com/terms-of-service)。
- `llm-context` 模式會傳回有來源依據的項目，而非一般網頁搜尋的摘要格式。
- `llm-context` 模式支援 `freshness` 和有界的 `date_after` + `date_before` 範圍。它不支援 `ui_lang`；未搭配 `date_after` 的 `date_before` 會遭拒絕，因為 Brave 要求自訂時間範圍必須同時包含開始和結束日期。
- `ui_lang` 必須包含 `en-US` 之類的地區子標籤。
- 結果預設會快取 15 分鐘（可透過 `cacheTtlMinutes` 設定）。
- 自訂 `webSearch.baseUrl` 值會納入 Brave 快取識別資訊，因此
  Proxy 專屬的回應不會發生衝突。
- 疑難排解時，啟用 `brave.http` 診斷旗標，即可記錄 Brave 請求 URL／查詢參數、回應狀態／計時，以及搜尋快取命中／未命中／寫入事件。此旗標絕不會記錄 API 金鑰或回應本文，但搜尋查詢可能包含敏感資訊。

## 相關內容

- [網頁搜尋概覽](/zh-TW/tools/web) -- 所有提供者與自動偵測
- [Perplexity Search](/zh-TW/tools/perplexity-search) -- 支援網域篩選的結構化結果
- [Exa Search](/zh-TW/tools/exa-search) -- 具備內容擷取功能的神經搜尋
