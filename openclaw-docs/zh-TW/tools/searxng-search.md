---
read_when:
    - 你想要自託管的網頁搜尋供應商
    - 你想使用 SearXNG 進行網頁搜尋
    - 你需要注重隱私或實體隔離的搜尋選項
summary: SearXNG 網頁搜尋——自架、免金鑰的中繼搜尋供應商
title: SearXNG 搜尋
x-i18n:
    generated_at: "2026-07-26T08:42:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cae8de9f8e2c8dd9cec615adb48da5c1fd7654bffe96c7afc1acea3effbcf1fc
    source_path: tools/searxng-search.md
    workflow: 16
---

OpenClaw 支援 [SearXNG](https://docs.searxng.org/) 作為**自行託管、
免金鑰**的 `web_search` 提供者。SearXNG 是一款開放原始碼的中繼搜尋引擎，
可彙整來自 Google、Bing、DuckDuckGo 及其他來源的結果。

優點：

- **免費且無限制** -- 不需要 API 金鑰或商業訂閱
- **隱私／實體隔離** -- 查詢永遠不會離開你的網路
- **隨處可用** -- 不受商業搜尋 API 的區域限制

## 設定

<Steps>
  <Step title="安裝外掛">
    ```bash
    openclaw plugins install @openclaw/searxng-plugin
    ```
  </Step>
  <Step title="執行 SearXNG 執行個體">
    ```bash
    docker run -d -p 8888:8080 searxng/searxng
    ```

    或使用你有權存取的任何現有 SearXNG 部署。請參閱
    [SearXNG 文件](https://docs.searxng.org/)以瞭解正式環境設定。

  </Step>
  <Step title="設定">
    ```bash
    openclaw configure --section web
    # 選取 "searxng" 作為提供者
    ```

    或設定環境變數，讓自動偵測找到它：

    ```bash
    export SEARXNG_BASE_URL="http://localhost:8888"
    ```

  </Step>
</Steps>

## 設定

```json5
{
  tools: {
    web: {
      search: {
        provider: "searxng",
      },
    },
  },
}
```

SearXNG 執行個體的外掛層級設定：

```json5
{
  plugins: {
    entries: {
      searxng: {
        config: {
          webSearch: {
            baseUrl: "http://localhost:8888",
            categories: "general,news", // 選用
            language: "en", // 選用
          },
        },
      },
    },
  },
}
```

`baseUrl` 也接受 SecretRef 物件（例如 `{ source: "env", id: "SEARXNG_BASE_URL" }`）。

## 環境變數

設定 `SEARXNG_BASE_URL` 作為組態的替代方式：

```bash
export SEARXNG_BASE_URL="http://localhost:8888"
```

解析順序：已設定的 `baseUrl` 字串，接著是
`baseUrl` 上的內嵌環境變數 SecretRef，最後是 `SEARXNG_BASE_URL`。
如果所有組態路徑皆未設定、`SEARXNG_BASE_URL` 存在，且未明確選擇提供者，
自動偵測會選取 SearXNG。

## 外掛組態參考

| 欄位        | 說明                                                        |
| ------------ | ------------------------------------------------------------------ |
| `baseUrl`    | 你的 SearXNG 執行個體的基底 URL（必填）                       |
| `categories` | 以逗號分隔的類別，例如 `general`、`news` 或 `science` |
| `language`   | 結果的語言代碼，例如 `en`、`de` 或 `fr`              |

`web_search` 工具呼叫也接受 `count`（1-10 筆結果）、`categories`
和 `language` 作為每次呼叫的覆寫值。

## 注意事項

- **JSON API** -- 使用 SearXNG 原生的 `format=json` 端點，而非擷取 HTML
- **圖片結果 URL** -- 當 SearXNG 傳回直接圖片 URL 時，圖片類別的結果會包含
  `img_src`
- **不需要 API 金鑰** -- 可直接搭配任何 SearXNG 執行個體使用
- **基底 URL 驗證** -- `baseUrl` 必須是有效的 `http://` 或 `https://`
  URL
- **網路防護** -- `http://` 基底 URL 必須指向受信任的私人或
  回送主機（公開主機必須使用 `https://`）；解析至私人／內部位址的 `https://`
  基底 URL 會獲得相同的自行託管允許，
  而解析至公開位址的 `https://` 基底 URL 仍會維持嚴格的 SSRF 防護
- **自動偵測順序** -- SearXNG 需要已設定的 `baseUrl`（在已具備必要認證資訊的
  提供者中，順序為 200）。DuckDuckGo 或 Ollama Web Search 等免金鑰提供者
  絕不會隱含勝出自動偵測；只有明確選擇 `provider` 時才會啟用
- **自行託管** -- 你可控制執行個體、查詢及上游搜尋引擎
- **類別**未設定時，預設為 `general`
- **類別備援** -- 如果非 `general` 類別的請求成功，
  但傳回零筆結果，OpenClaw 會先使用 `general` 重試相同查詢一次，
  再傳回空白結果集
- **結果快取** -- 相同的查詢（相同的查詢內容、數量、類別、
  語言及基底 URL）會在處理程序內以較短的 TTL 快取
- **版本需求** -- 此外掛宣告 `minHostVersion: >=2026.6.9`

<Tip>
  若要讓 SearXNG JSON API 正常運作，請確保你的 SearXNG 執行個體已在
  `settings.yml` 的 `search.formats` 下啟用 `json` 格式。
</Tip>

## 相關內容

- [網頁搜尋概覽](/zh-TW/tools/web) -- 所有提供者及自動偵測
- [DuckDuckGo 搜尋](/tools/duckduckgo-search) -- 另一個免金鑰提供者
- [Brave 搜尋](/zh-TW/tools/brave-search) -- 提供免費方案的結構化結果
