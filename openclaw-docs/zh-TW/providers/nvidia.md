---
read_when:
    - 你想在 OpenClaw 中免費使用開放模型
    - 你需要設定 NVIDIA_API_KEY
    - 你想透過 NVIDIA 使用 Nemotron 3 Ultra
summary: 在 OpenClaw 中使用 NVIDIA 的 OpenAI 相容 API
title: NVIDIA
x-i18n:
    generated_at: "2026-07-26T07:54:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b5ac7bcc19400a661b2f2861a1dd4d2306c94e445783929e342e9184003314e9
    source_path: providers/nvidia.md
    workflow: 16
---

NVIDIA 透過位於
`https://integrate.api.nvidia.com/v1` 的 OpenAI 相容 API 免費提供開放模型，並使用來自
[build.nvidia.com](https://build.nvidia.com/settings/api-keys) 的 API 金鑰進行驗證。OpenClaw
預設將 NVIDIA 提供者設為 Nemotron 3 Ultra，這是 NVIDIA 具備總計 550B／啟用 55B
參數的推理模型，適合長上下文的代理式工作。

## 開始使用

<Steps>
  <Step title="取得 API 金鑰">
    在 [build.nvidia.com](https://build.nvidia.com/settings/api-keys) 建立 API 金鑰。
  </Step>
  <Step title="匯出金鑰並執行新手設定">
    ```bash
    export NVIDIA_API_KEY="nvapi-..."
    openclaw onboard --auth-choice nvidia-api-key
    ```
  </Step>
  <Step title="設定 NVIDIA 模型">
    ```bash
    openclaw models set nvidia/nvidia/nemotron-3-ultra-550b-a55b
    ```
  </Step>
</Steps>

若要進行非互動式設定，請直接傳入金鑰：

```bash
openclaw onboard --auth-choice nvidia-api-key --nvidia-api-key "nvapi-..."
```

<Warning>
`--nvidia-api-key` 會將金鑰寫入 Shell 歷史記錄與 `ps` 輸出。請盡可能優先使用
`NVIDIA_API_KEY` 環境變數。
</Warning>

## 設定範例

```json5
{
  env: { NVIDIA_API_KEY: "nvapi-..." },
  models: {
    providers: {
      nvidia: {
        baseUrl: "https://integrate.api.nvidia.com/v1",
        api: "openai-completions",
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "nvidia/nvidia/nemotron-3-ultra-550b-a55b" },
    },
  },
}
```

## 精選目錄

設定 NVIDIA API 金鑰後，設定與模型選擇路徑會從
`https://assets.ngc.nvidia.com/products/api-catalog/featured-models.json` 擷取
NVIDIA 的公開精選模型目錄，並將結果快取 24 小時（前 32 個項目，匯入為自由文字輸入
資料列）。因此，build.nvidia.com 的新精選模型無須等待 OpenClaw 發布新版本，即會出現在設定與
模型選擇介面中。即時資訊提供可用時，第一個傳回的模型會成為
NVIDIA 設定期間的預選選項。

擷取作業對 `assets.ngc.nvidia.com` 使用固定的 HTTPS 主機原則。若未
設定 NVIDIA API 金鑰，或資訊提供無法使用或格式錯誤，
OpenClaw 會改用下方的內建目錄與內建預設值。

## Nemotron 3 Ultra

Nemotron 3 Ultra 是 OpenClaw 中的預設 NVIDIA 模型。NVIDIA 的
[`nvidia/nemotron-3-ultra-550b-a55b`](https://build.nvidia.com/nvidia/nemotron-3-ultra-550b-a55b)
建置頁面將其列為可用的免費端點，並標示 1M-token 的上下文規格。

內建的 Ultra 資料列預設傳送
`chat_template_kwargs: { enable_thinking: false, force_nonempty_content: true }`，
讓一般聊天輸出保留在可見答案中，而不會暴露推理文字。

若要使用功能最強的 NVIDIA 預設模型，請使用 Ultra。若想使用較小的 Nemotron 3 選項，
請繼續選用 Super；如果 NVIDIA 目錄中託管的第三方模型在上下文、延遲或行為方面
更符合需求，也可以選用其中之一。

## 內建備援目錄

可選的內建資料列是 NVIDIA 精選模型目錄的快照。已棄用的
相容性資料列仍可透過精確參照解析，但不會顯示在模型
選擇器中。

| 模型參照                                   | 名稱                  | 上下文    | 最大輸出   |
| ------------------------------------------ | --------------------- | --------- | ---------- |
| `nvidia/nvidia/nemotron-3-ultra-550b-a55b` | Nemotron 3 Ultra 550B | 1,048,576 | 8,192      |
| `nvidia/nvidia/nemotron-3-super-120b-a12b` | Nemotron 3 Super 120B | 1,000,000 | 8,192      |
| `nvidia/z-ai/glm-5.2`                      | GLM 5.2               | 202,752   | 8,192      |
| `nvidia/moonshotai/kimi-k2.6`              | Kimi K2.6             | 262,144   | 8,192      |
| `nvidia/minimaxai/minimax-m3`              | Minimax M3            | 196,608   | 8,192      |
| `nvidia/deepseek-ai/deepseek-v4-pro`       | DeepSeek V4 Pro       | 262,144   | 16,384     |
| `nvidia/qwen/qwen3.5-397b-a17b`            | Qwen3.5 397B A17B     | 262,144   | 16,384     |

完整的相容性目錄也保留這些已發布參照，以供現有
設定使用：`nvidia/moonshotai/kimi-k2.5`、`nvidia/z-ai/glm-5.1`、
`nvidia/minimaxai/minimax-m2.5`、`nvidia/z-ai/glm5` 及
`nvidia/minimaxai/minimax-m2.7`。這些參照仍可透過精確參照使用，但
絕不會出現在新手設定或模型選擇器中。

## 進階設定

<AccordionGroup>
  <Accordion title="自動啟用行為">
    設定 `NVIDIA_API_KEY` 環境變數，或在新手設定期間儲存過金鑰時，
    提供者會自動啟用。除了金鑰以外，不需要明確設定提供者。
  </Accordion>

  <Accordion title="目錄與定價">
    設定 NVIDIA 驗證後，OpenClaw 會優先使用 NVIDIA 的公開精選模型目錄，
    並將其快取 24 小時。內建的可選備援是 NVIDIA 精選模型目錄的
    靜態快照；已棄用且僅供精確參照使用的相容性資料列不會顯示在模型選擇器中。原始碼中的費用預設為 `0`，
    因為 NVIDIA 目前為列出的模型提供免費 API 存取。
  </Accordion>

  <Accordion title="OpenAI 相容端點">
    OpenClaw 使用 `openai-completions` 轉接器，透過標準
    `/v1` 聊天補全路由與 NVIDIA 通訊。任何 OpenAI 相容工具搭配 NVIDIA 基礎 URL
    應可直接使用。
  </Accordion>

  <Accordion title="Nemotron 3 Ultra 推理參數">
    NVIDIA 的 Ultra 範例請求使用 `chat_template_kwargs.enable_thinking`
    與 `reasoning_budget` 來輸出推理內容。OpenClaw 的內建 Ultra 資料列
    預設停用範本思考，以供一般聊天使用。若需要
    選擇啟用 NVIDIA 推理輸出，或強制設定其他 NVIDIA 專用的請求
    欄位，請設定各模型的參數，並將提供者專用覆寫限制在
    NVIDIA 模型範圍內：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "nvidia/nvidia/nemotron-3-ultra-550b-a55b": {
              params: {
                chat_template_kwargs: { enable_thinking: true },
                extra_body: { reasoning_budget: 16384 },
              },
            },
          },
        },
      },
    }
    ```

    `params.chat_template_kwargs` 會合併至請求上已有的任何 `chat_template_kwargs`，
    而不會取代整個物件。
    `params.extra_body` 是最終的 OpenAI 相容請求主體覆寫，
    並會覆寫衝突的承載資料鍵，因此僅應用於 NVIDIA
    為所選端點記載的欄位。

  </Accordion>

  <Accordion title="緩慢的自訂提供者回應">
    部分由 NVIDIA 託管的自訂模型，可能需要比預設約 120s
    模型閒置監視器更長的時間，才會發出第一個回應區塊。對於自訂
    NVIDIA 提供者項目，請提高提供者逾時，而不是整個
    代理執行階段的逾時；`timeoutSeconds` 涵蓋提供者的 HTTP 請求，並
    提高該提供者的閒置／串流監視器上限：

    ```json5
    {
      models: {
        providers: {
          "custom-integrate-api-nvidia-com": {
            baseUrl: "https://integrate.api.nvidia.com/v1",
            api: "openai-completions",
            apiKey: "NVIDIA_API_KEY",
            timeoutSeconds: 300,
          },
        },
      },
      agents: {
        defaults: {
          models: {
            "custom-integrate-api-nvidia-com/meta/llama-3.1-70b-instruct": {
              params: { thinking: "off" },
            },
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

<Tip>
NVIDIA 模型目前可免費使用。請查看
[build.nvidia.com](https://build.nvidia.com/) 以取得最新的可用性與
速率限制詳細資訊。
</Tip>

## 相關內容

<CardGroup cols={2}>
  <Card title="模型選擇" href="/zh-TW/concepts/model-providers" icon="layers">
    選擇提供者、模型參照及容錯移轉行為。
  </Card>
  <Card title="設定參考" href="/zh-TW/gateway/configuration-reference" icon="gear">
    代理、模型及提供者的完整設定參考。
  </Card>
</CardGroup>
