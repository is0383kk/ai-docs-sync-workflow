---
read_when:
    - 你想使用 Grok 進行 web_search
    - 你想要使用 xAI OAuth 或 XAI_API_KEY 進行網頁搜尋
summary: 透過 xAI 網路依據回應進行 Grok 網路搜尋
title: Grok 搜尋
x-i18n:
    generated_at: "2026-07-26T08:50:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6e39edd660d0ffe8be066ae81317810da691a7dbd8c59a74222a59145cff5c77
    source_path: tools/grok-search.md
    workflow: 16
---

OpenClaw 支援將 Grok 作為 `web_search` 提供者，使用 xAI 以網路資料為依據的
回應，產生由即時搜尋結果與引用來源支援的 AI 綜合答案。

Grok 網路搜尋會優先使用現有的 xAI OAuth 登入（若有）。
若不存在 OAuth 設定檔，同一組 xAI API 金鑰也會支援內建的
`x_search` 工具，以搜尋 X（前稱 Twitter）貼文，以及 `code_execution`
工具。將金鑰儲存在 `plugins.entries.xai.config.webSearch.apiKey` 也能
讓 OpenClaw 將其重複用作內建 xAI 模型提供者的備援。

若要取得貼文層級的 X 指標（轉發、回覆、書籤、檢視次數），請使用
[`x_search`](/zh-TW/tools/web#x_search)，並提供確切的貼文 URL 或狀態 ID，
而非寬泛的搜尋查詢。

## 初始設定與配置

在 `openclaw onboard` 或 `openclaw configure --section
web` 期間選擇 **Grok**，可讓 OpenClaw 重複使用現有的 xAI OAuth 設定檔，而不會提示
輸入個別的網路搜尋金鑰。若沒有 OAuth，則會改用 xAI API 金鑰設定。

接著，OpenClaw 會提供後續步驟，讓你使用相同的 xAI
認證資訊啟用 `x_search`。此後續步驟：

- 僅在你為 `web_search` 選擇 Grok 後才會顯示
- 不是個別的頂層網路搜尋提供者選項
- 可選擇在同一流程中設定 `x_search` 模型

略過此步驟，即可稍後在設定中啟用或變更 `x_search`。

## 登入或取得 API 金鑰

<Steps>
  <Step title="使用 xAI OAuth">
    如果你已在初始設定或模型驗證期間登入 xAI，請選擇
    Grok 作為 `web_search` 提供者。不需要個別的 API 金鑰：

    ```bash
    openclaw onboard --auth-choice xai-oauth
    openclaw config set tools.web.search.provider grok
    ```

  </Step>
  <Step title="使用 API 金鑰備援">
    當 OAuth 無法使用，或你刻意想使用由金鑰支援的網路搜尋設定時，
    請從 [xAI](https://console.x.ai/) 取得 API 金鑰。
  </Step>
  <Step title="儲存金鑰">
    在閘道環境中設定 `XAI_API_KEY`，或透過以下方式配置：

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
      xai: {
        config: {
          webSearch: {
            apiKey: "xai-...", // optional if xAI OAuth or XAI_API_KEY is available
            baseUrl: "https://api.x.ai/v1", // optional Responses API proxy/base URL override
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "grok",
      },
    },
  },
}
```

**其他認證資訊選項：** 閘道環境中的 `openclaw models auth login --provider xai
--method oauth`、`XAI_API_KEY`，或
`plugins.entries.xai.config.webSearch.apiKey`。若是閘道安裝，請將環境
變數放在 `~/.openclaw/.env` 中。

## 運作方式

Grok 使用 xAI 以網路資料為依據的回應來綜合答案，並附上行內
引用，類似於 Gemini 的 Google 搜尋依據方法。

## 支援的參數

Grok 搜尋支援 `query`。為了共用 `web_search`
相容性，也接受 `count`，但 Grok 一律傳回一個附帶引用的綜合答案，
而不是包含 N 筆結果的清單。不支援提供者專屬篩選條件。

Grok 預設逾時時間為 60 秒，因為 xAI Responses 以網路資料為依據的
搜尋執行時間可能超過共用的 `web_search` 預設值。可使用
`tools.web.search.timeoutSeconds` 覆寫此設定。

## 基礎 URL 覆寫

設定 `plugins.entries.xai.config.webSearch.baseUrl`，即可透過營運者代理伺服器或與 xAI 相容的 Responses
端點路由 Grok 網路搜尋。OpenClaw 會在移除結尾斜線後，
向 `<baseUrl>/responses` 傳送 POST 請求。除非已設定 `plugins.entries.xai.config.xSearch.baseUrl`，
否則 `x_search` 會備援至相同的 `webSearch.baseUrl`。

## 相關內容

- [網路搜尋概覽](/zh-TW/tools/web) -- 所有提供者與自動偵測
- [網路搜尋中的 x_search](/zh-TW/tools/web#x_search) -- 透過 xAI 進行第一級 X 搜尋
- [Gemini 搜尋](/zh-TW/tools/gemini-search) -- 透過 Google 依據功能產生 AI 綜合答案
