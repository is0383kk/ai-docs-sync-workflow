---
read_when:
    - 你想使用 Gemini 進行 web_search
    - 你需要 `GEMINI_API_KEY` 或 `models.providers.google.apiKey`
    - 你想使用 Google 搜尋依據功能
summary: 使用 Google 搜尋依據的 Gemini 網頁搜尋
title: Gemini 搜尋
x-i18n:
    generated_at: "2026-07-26T08:41:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4c7cb55fb185adfda01ab6b3c6434ab6e3ee31162733c752d4c81328bce9a6cd
    source_path: tools/gemini-search.md
    workflow: 16
---

OpenClaw 支援內建
[Google 搜尋基礎檢索](https://ai.google.dev/gemini-api/docs/grounding)的 Gemini 模型，
可根據即時 Google 搜尋結果傳回由 AI 統整並附有
引用的回答。

## 取得 API 金鑰

<Steps>
  <Step title="建立金鑰">
    前往 [Google AI Studio](https://aistudio.google.com/apikey) 並建立
    API 金鑰。
  </Step>
  <Step title="儲存金鑰">
    在閘道環境中設定 `GEMINI_API_KEY`、重複使用
    `models.providers.google.apiKey`，或透過以下命令設定專用的網路搜尋金鑰：

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
      google: {
        config: {
          webSearch: {
            apiKey: "AIza...", // 若已設定 GEMINI_API_KEY 或 models.providers.google.apiKey，則為選用
            baseUrl: "https://generativelanguage.googleapis.com/v1beta", // 選用；回退至 models.providers.google.baseUrl
            model: "gemini-2.5-flash", // 預設值
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "gemini",
      },
    },
  },
}
```

**認證資訊優先順序：**Gemini 網路搜尋會依序優先使用
`plugins.entries.google.config.webSearch.apiKey`、`GEMINI_API_KEY`，
再使用 `models.providers.google.apiKey`。對於基礎 URL，專用的
`plugins.entries.google.config.webSearch.baseUrl` 優先於
`models.providers.google.baseUrl`。

若為閘道安裝，請將環境金鑰放在 `~/.openclaw/.env` 中。

## 運作方式

傳統搜尋提供者會傳回連結與摘要清單，而 Gemini 則不同，它使用 Google 搜尋基礎檢索來產生
由 AI 統整並附有行內引用的回答。結果同時包含統整後的回答與來源
URL。

- Gemini 基礎檢索所提供的引用 URL，會透過 OpenClaw 具 SSRF 防護的
  擷取路徑發出 HEAD 請求，自動將 Google
  重新導向 URL 解析為直接 URL（跟隨重新導向、驗證 http/https）。
- 重新導向解析會使用嚴格的 SSRF 預設值，因此會封鎖重新導向至
  私有／內部目標的情況。

## 支援的參數

Gemini 搜尋支援 `query`、`freshness`、`date_after` 和 `date_before`。

系統接受 `count` 以相容共用的 `web_search`，但 Gemini 基礎檢索
仍會傳回一個附有引用的統整回答，而不是包含 N 筆結果的
清單。

`freshness` 接受 `day`、`week`、`month`、`year`，以及共用的快速選項
`pd`、`pw`、`pm` 和 `py`。`day`/`pd` 會在 Gemini
查詢中加入近期性指示，而非使用固定的 24 小時範圍。`week`、`month`、`year` 和明確的
`date_after`/`date_before` 範圍會設定 Gemini Google 搜尋基礎檢索的
`timeRangeFilter`。不支援 `country`、`language` 和 `domain_filter`。

## 模型選擇

預設模型為 `gemini-2.5-flash`（快速且具成本效益）。任何支援基礎檢索的 Gemini
模型皆可透過
`plugins.entries.google.config.webSearch.model` 使用。

## 基礎 URL 覆寫

當 Gemini 網路搜尋必須透過營運者 Proxy 或自訂的 Gemini 相容端點進行路由時，請設定
`plugins.entries.google.config.webSearch.baseUrl`。若未設定，Gemini 網路搜尋會重複使用 `models.providers.google.baseUrl`。單純的
`https://generativelanguage.googleapis.com` 值會正規化為
`https://generativelanguage.googleapis.com/v1beta`；自訂 Proxy 路徑在移除尾端斜線後，會依提供的內容保留。

## 相關內容

- [網路搜尋概覽](/zh-TW/tools/web) -- 所有提供者與自動偵測
- [Brave Search](/zh-TW/tools/brave-search) -- 包含摘要的結構化結果
- [Perplexity Search](/zh-TW/tools/perplexity-search) -- 結構化結果 + 內容擷取
