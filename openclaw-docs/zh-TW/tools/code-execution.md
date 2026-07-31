---
read_when:
    - 你想要啟用或設定 `code_execution`
    - 你想要在沒有本機 Shell 存取權的情況下進行遠端分析
    - 你想要將 x_search 或 web_search 與遠端 Python 分析結合使用
summary: code_execution：使用 xAI 執行沙箱化的遠端 Python 分析
title: 程式碼執行
x-i18n:
    generated_at: "2026-07-26T07:35:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1ab391daed9154f113535e6d241c45d5c08c22abdc012148a9f0f2ae5ec548b3
    source_path: tools/code-execution.md
    workflow: 16
---

`code_execution` 會在 xAI 的 Responses API 上執行沙箱化的遠端 Python 分析
（`https://api.x.ai/v1/responses`，與 `x_search` 使用相同的端點）。它由隨附的 `xai` 外掛依據 `tools` 合約註冊。

<Warning>
  `code_execution` 會在 xAI 的伺服器上執行。xAI 對每 1,000 次工具呼叫收取 $5，
  另加模型的輸入與輸出權杖費用。
</Warning>

| 屬性               | 值                                                                                |
| ------------------ | --------------------------------------------------------------------------------- |
| 工具名稱           | `code_execution`                                                                  |
| 提供者外掛         | `xai`（隨附，`enabledByDefault: true`）                                         |
| 驗證               | xAI 驗證設定檔、`XAI_API_KEY` 或 `plugins.entries.xai.config.webSearch.apiKey` |
| 預設模型           | `grok-4.3`                                                                        |
| 預設逾時           | 30 秒                                                                             |
| 預設 `maxTurns` | 未設定（xAI 會套用其內部限制）                                                   |

可用於計算、表格化、快速統計與圖表式分析，
包括分析由 `x_search` 或 `web_search` 傳回的資料。它無法存取
本機檔案、你的 shell、你的儲存庫或已配對裝置，且不會在呼叫之間
保留狀態，因此應將每次呼叫視為暫時性分析，而非
筆記本工作階段。若要取得最新的 X 資料，請先執行 [`x_search`](/zh-TW/tools/web#x_search)，
再將結果輸入其中。

若要在本機執行，請改用 [`exec`](/zh-TW/tools/exec)。

## 設定

<Steps>
  <Step title="提供 xAI 認證資訊">
    OAuth 需要符合資格的 SuperGrok 或 X Premium 訂閱
    （採用裝置代碼驗證，因此可從遠端主機使用，無須
    localhost 回呼）：

    ```bash
    openclaw models auth login --provider xai --method oauth
    ```

    全新安裝時，也可以在初始設定中選擇相同選項：

    ```bash
    openclaw onboard --install-daemon --auth-choice xai-oauth
    ```

    或使用 API 金鑰：

    ```bash
    openclaw models auth login --provider xai --method api-key
    export XAI_API_KEY=xai-...
    ```

    或透過設定：

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              webSearch: {
                apiKey: "xai-...",
              },
            },
          },
        },
      },
    }
    ```

    這三種方式都能同時支援 `x_search` 與 Grok `web_search`。

  </Step>

  <Step title="啟用並調整 code_execution">
    省略 `enabled` 時，只有在使用中
    模型的提供者為 `xai` 且可解析 xAI 認證資訊時，才會公開 `code_execution`。
    如果使用中模型已有已知的非 xAI 提供者，請將
    `plugins.entries.xai.config.codeExecution.enabled` 設為 `true`，以選擇啟用
    跨提供者使用。若缺少使用中模型的提供者或無法解析，
    此工具會維持隱藏。將 `enabled` 設為 `false`，即可對所有
    提供者停用此工具。無論如何都必須提供 xAI 認證資訊。

    使用相同區塊覆寫模型、回合上限或逾時：

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              codeExecution: {
                enabled: true, // 已知的非 xAI 模型提供者必須設定
                model: "grok-4.3", // 覆寫預設的 xAI 程式碼執行模型
                maxTurns: 2,            // 內部工具回合數的選用上限
                timeoutSeconds: 30,     // 請求逾時（預設：30）
              },
            },
          },
        },
      },
    }
    ```

  </Step>

  <Step title="重新啟動閘道">
    ```bash
    openclaw gateway restart
    ```

    當 xAI 外掛重新註冊，且通過上述提供者、啟用狀態及驗證檢查後，
    `code_execution` 就會出現在代理程式的工具清單中。

  </Step>
</Steps>

## 使用方式

請明確說明分析目的；此工具只接受一個 `task` 參數，
因此請在單一提示中傳送完整要求與任何內嵌資料：

```text
使用 code_execution 計算這些數字的 7 天移動平均值：...
```

```text
使用 x_search 尋找本週提及 OpenClaw 的貼文，然後使用 code_execution 按日期統計數量。
```

```text
使用 web_search 蒐集最新的 AI 基準測試數據，然後使用 code_execution 比較百分比變化。
```

## 錯誤

若未提供驗證，此工具會傳回結構化 JSON 錯誤（而非擲回
例外），讓代理程式能自行修正：

```json
{
  "error": "missing_xai_api_key",
  "message": "code_execution 需要 xAI 認證資訊。請執行 `openclaw onboard --auth-choice xai-oauth` 以使用 Grok 登入、執行 `openclaw onboard --auth-choice xai-api-key`、在閘道環境中設定 `XAI_API_KEY`，或設定 `plugins.entries.xai.config.webSearch.apiKey`。",
  "docs": "https://docs.openclaw.ai/tools/code-execution"
}
```

## 相關內容

<CardGroup cols={2}>
  <Card title="Exec 工具" href="/zh-TW/tools/exec" icon="terminal">
    在你的機器或已配對節點上執行本機 shell。
  </Card>
  <Card title="Exec 核准" href="/zh-TW/tools/exec-approvals" icon="shield">
    shell 執行的允許／拒絕政策。
  </Card>
  <Card title="Web 工具" href="/zh-TW/tools/web" icon="globe">
    `web_search`、`x_search` 和 `web_fetch`。
  </Card>
  <Card title="xAI 提供者" href="/zh-TW/providers/xai" icon="microchip">
    Grok 模型、Web／X 搜尋及程式碼執行設定。
  </Card>
</CardGroup>
