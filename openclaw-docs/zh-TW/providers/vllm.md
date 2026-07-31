---
read_when:
    - 你想要讓 OpenClaw 連接至本機 vLLM 伺服器執行
    - 你想要使用自有模型的 OpenAI 相容 `/v1` 端點
summary: 使用 vLLM（與 OpenAI 相容的本機伺服器）執行 OpenClaw
title: vLLM
x-i18n:
    generated_at: "2026-07-26T07:33:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98d1044c0a82efb6c9937e961d765d0cfcea8664cbaa043168921b457756512c
    source_path: providers/vllm.md
    workflow: 16
---

vLLM 透過 **OpenAI 相容** HTTP API 提供開放原始碼（以及部分自訂）模型。OpenClaw 使用 `openai-completions` API 連線，並可在你透過 `VLLM_API_KEY` 選擇啟用時**自動探索**模型。

| 屬性             | 值                                         |
| ---------------- | ------------------------------------------ |
| 提供者 ID        | `vllm`                         |
| API              | `openai-completions`（OpenAI 相容）          |
| 驗證             | `VLLM_API_KEY` 環境變數                |
| 預設基礎 URL     | `http://127.0.0.1:8000/v1`                         |
| 串流用量         | 支援（`stream_options.include_usage`）                 |

## 開始使用

<Steps>
  <Step title="使用 OpenAI 相容伺服器啟動 vLLM">
    你的基礎 URL 必須公開 `/v1` 端點（`/v1/models`、`/v1/chat/completions`）。vLLM 通常執行於：

    ```text
    http://127.0.0.1:8000/v1
    ```

  </Step>
  <Step title="設定 API 金鑰環境變數">
    如果你的伺服器不強制驗證，任何非空值皆可使用：

    ```bash
    export VLLM_API_KEY="vllm-local"
    ```

  </Step>
  <Step title="選取模型">
    請替換為你的其中一個 vLLM 模型 ID：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "vllm/your-model-id" },
        },
      },
    }
    ```

  </Step>
  <Step title="確認模型可用">
    ```bash
    openclaw models list --provider vllm
    ```
  </Step>
</Steps>

<Tip>
若要進行非互動式設定（CI、指令碼），請直接傳入基礎 URL、金鑰和模型：

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice vllm \
  --custom-base-url "http://127.0.0.1:8000/v1" \
  --custom-api-key "vllm-local" \
  --custom-model-id "your-model-id"
```

</Tip>

## 模型探索（隱含提供者）

設定 `VLLM_API_KEY`（或存在驗證設定檔）且**未**定義 `models.providers.vllm` 時，OpenClaw 會查詢 `GET http://127.0.0.1:8000/v1/models`，並將傳回的 ID 轉換為模型項目。

<Note>
如果你明確設定 `models.providers.vllm`，OpenClaw 只會使用你宣告的模型。將 `"vllm/*": {}` 新增至 `agents.defaults.models`，可讓 OpenClaw 同時查詢該已設定提供者的 `/models` 端點，並納入所有公布的 vLLM 模型。
</Note>

## 明確設定

當 vLLM 在不同主機或連接埠上執行、你想固定 `contextWindow`/`maxTokens`、伺服器要求真正的 API 金鑰，或你連線至受信任的回送、區域網路或 Tailscale 端點時，請進行明確設定：

```json5
{
  models: {
    providers: {
      vllm: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "${VLLM_API_KEY}",
        api: "openai-completions",
        timeoutSeconds: 300, // 選用：延長速度較慢之本機模型的請求逾時時間
        models: [
          {
            id: "your-model-id",
            name: "Local vLLM Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

若要讓提供者保持動態而不列出每個模型，請在可見模型目錄中新增萬用字元：

```json5
{
  agents: {
    defaults: {
      models: {
        "vllm/*": {},
      },
    },
  },
}
```

## 進階設定

<AccordionGroup>
  <Accordion title="代理式行為">
    vLLM 會被視為代理式的 OpenAI 相容 `/v1` 後端，而非原生 OpenAI 端點：

    | 行為                                    | 是否套用？                       |
    | --------------------------------------- | -------------------------------- |
    | 原生 OpenAI 請求塑形                    | 否                               |
    | `service_tier`                      | 不傳送                           |
    | Responses `store`            | 不傳送                           |
    | 提示詞快取提示                          | 不傳送                           |
    | OpenAI 推理相容承載資料塑形             | 不套用                           |
    | 隱藏的 OpenClaw 歸屬標頭                | 不會插入自訂基礎 URL             |

  </Accordion>

  <Accordion title="Qwen 思考控制">
    對於 Qwen 模型，若伺服器預期 Qwen 聊天範本關鍵字引數，請在模型列設定 `compat.thinkingFormat: "qwen-chat-template"`。這些模型會公開二元 `/think` 設定檔（`off`、`on`），因為 Qwen 聊天範本的思考功能是開關旗標，而非 OpenAI 式的投入程度階梯。

    ```json5
    {
      models: {
        providers: {
          vllm: {
            models: [
              {
                id: "Qwen/Qwen3-8B",
                name: "Qwen3 8B",
                reasoning: true,
                compat: { thinkingFormat: "qwen-chat-template" },
              },
            ],
          },
        },
      },
    }
    ```

    OpenClaw 會將 `/think off` 對應至：

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "preserve_thinking": true
      }
    }
    ```

    非 `off` 的思考層級會傳送 `enable_thinking: true`。如果你的端點改為預期 DashScope 式頂層旗標，請使用 `compat.thinkingFormat: "qwen"`，在請求根層級傳送 `enable_thinking`。

  </Accordion>

  <Accordion title="Nemotron 3 思考控制">
    對於關閉思考功能的 `vllm/nemotron-3-*` 模型，隨附的外掛會傳送：

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "force_nonempty_content": true
      }
    }
    ```

    若要自訂這些值，請在模型參數下設定 `chat_template_kwargs`。如果你也設定 `params.extra_body.chat_template_kwargs`，會以該值為準，因為 `extra_body` 是最後套用的請求本文覆寫。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/nemotron-3-super": {
              params: {
                chat_template_kwargs: {
                  enable_thinking: false,
                  force_nonempty_content: true,
                },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Qwen 工具呼叫顯示為文字">
    請先確認 vLLM 已使用適合該模型的正確工具呼叫剖析器與聊天範本啟動。vLLM 文件為 Qwen2.5 模型記載 `hermes`，並為 Qwen3-Coder 模型記載 `qwen3_xml`。

    症狀：Skills／工具從未執行、助理輸出 `{"name":"read","arguments":...}` 等原始 JSON/XML，或 OpenClaw 傳送 `tool_choice: "auto"` 時，vLLM 傳回空的 `tool_calls` 陣列。

    部分 Qwen/vLLM 組合只會在請求使用 `tool_choice: "required"` 時傳回結構化工具呼叫。請使用 `params.extra_body` 針對各模型強制啟用：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/Qwen-Qwen2.5-Coder-32B-Instruct": {
              params: {
                extra_body: {
                  tool_choice: "required",
                },
              },
            },
          },
        },
      },
    }
    ```

    請將模型 ID 替換為 `openclaw models list --provider vllm` 中的確切 ID，或從命令列介面套用相同覆寫：

    ```bash
    openclaw config set agents.defaults.models '{"vllm/Qwen-Qwen2.5-Coder-32B-Instruct":{"params":{"extra_body":{"tool_choice":"required"}}}}' --strict-json --merge
    ```

    這是選擇性啟用的因應措施：它會強制每個提供工具的回合進行工具呼叫，因此只能用於可接受此行為的專用模型項目。請勿將其設為所有 vLLM 模型的全域預設值，也不要將其與會把任意助理文字轉換為可執行工具呼叫的代理搭配使用。

  </Accordion>

  <Accordion title="自訂基礎 URL">
    如果你的 vLLM 伺服器在非預設主機或連接埠上執行，請在明確的提供者設定中設定 `baseUrl`：

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:9000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [
              {
                id: "my-custom-model",
                name: "Remote vLLM Model",
                reasoning: false,
                input: ["text"],
                contextWindow: 64000,
                maxTokens: 4096,
              },
            ],
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## 疑難排解

<AccordionGroup>
  <Accordion title="首次回應緩慢或遠端伺服器逾時">
    對於大型本機模型、遠端區域網路主機或 tailnet 連線，請設定提供者範圍的請求逾時：

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:8000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [{ id: "your-model-id", name: "Local vLLM Model" }],
          },
        },
      },
    }
    ```

    `timeoutSeconds` 僅套用於 vLLM 模型 HTTP 請求：連線設定、回應標頭、本文串流，以及受保護擷取作業的整體中止。它也會將此提供者的 LLM 閒置／串流監控逾時上限提高至隱含的約 120s 預設值以上。請優先採用此設定，而非提高控制整個代理執行過程的 `agents.defaults.timeoutSeconds`。

  </Accordion>

  <Accordion title="無法連線至伺服器">
    請檢查 vLLM 伺服器是否正在執行且可供存取：

    ```bash
    curl http://127.0.0.1:8000/v1/models
    ```

    如果出現連線錯誤，請確認主機、連接埠，以及 vLLM 是否以 OpenAI 相容伺服器模式啟動。對於回送、區域網路和 Tailscale 端點上的受保護模型請求，OpenClaw 會信任設定之 `models.providers.vllm.baseUrl` 的確切來源。若未明確選擇啟用，中繼資料／連結本機來源仍會遭到封鎖。只有當 vLLM 請求必須連線至其他私人來源時，才設定 `models.providers.vllm.request.allowPrivateNetwork: true`；若要停用確切來源信任，則設定 `false`。

  </Accordion>

  <Accordion title="請求發生驗證錯誤">
    如果請求因驗證錯誤而失敗，請設定符合伺服器設定的真正 `VLLM_API_KEY`，或在 `models.providers.vllm` 下明確設定提供者。

    <Tip>
    如果你的 vLLM 伺服器不強制驗證，`VLLM_API_KEY` 的任何非空值都可作為 OpenClaw 的選擇啟用訊號。
    </Tip>

  </Accordion>

  <Accordion title="未探索到模型">
    自動探索要求設定 `VLLM_API_KEY`。如果你已定義 `models.providers.vllm`，除非 `agents.defaults.models` 包含 `"vllm/*": {}`，否則 OpenClaw 只會使用你宣告的模型。
  </Accordion>

  <Accordion title="工具呈現為原始文字">
    如果 Qwen 模型輸出 JSON/XML 工具語法，而非執行 Skill：

    - 使用適合該模型的正確剖析器／範本啟動 vLLM。
    - 使用 `openclaw models list --provider vllm` 確認確切的模型 ID。
    - 只有在 `tool_choice: "auto"` 仍傳回空白或純文字工具呼叫時，才新增專用的各模型 `params.extra_body.tool_choice: "required"` 覆寫。

  </Accordion>
</AccordionGroup>

<Warning>
更多協助：[疑難排解](/zh-TW/help/troubleshooting)與[常見問題](/zh-TW/help/faq)。
</Warning>

## 相關內容

<CardGroup cols={2}>
  <Card title="模型選取" href="/zh-TW/concepts/model-providers" icon="layers">
    選擇提供者、模型參照和容錯移轉行為。
  </Card>
  <Card title="OpenAI" href="/zh-TW/providers/openai" icon="bolt">
    原生 OpenAI 提供者與 OpenAI 相容路由行為。
  </Card>
  <Card title="OAuth 與驗證" href="/zh-TW/gateway/authentication" icon="key">
    驗證詳細資料與認證資訊重複使用規則。
  </Card>
  <Card title="疑難排解" href="/zh-TW/help/troubleshooting" icon="wrench">
    常見問題及其解決方式。
  </Card>
</CardGroup>
