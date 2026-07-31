---
read_when:
    - 你想要透過 LiteLLM Proxy 路由 OpenClaw
    - 你需要透過 LiteLLM 進行成本追蹤、記錄或模型路由管理
summary: 透過 LiteLLM Proxy 執行 OpenClaw，以統一存取模型並追蹤成本
title: LiteLLM
x-i18n:
    generated_at: "2026-07-26T08:46:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 22451f0eefcf991a602409701fc752f97600a67752c67304137c7f17f3dd1a16
    source_path: providers/litellm.md
    workflow: 16
---

[LiteLLM](https://litellm.ai) 是一個開放原始碼的 LLM 閘道，提供統一 API，可連接 100 多個模型供應商。透過 LiteLLM 路由 OpenClaw，無須變更 OpenClaw 設定，即可集中追蹤成本、記錄日誌、使用具支出上限的虛擬金鑰，以及在後端間容錯移轉。

## 快速開始

<Tabs>
  <Tab title="初始設定（建議）">
    ```bash
    openclaw onboard --auth-choice litellm-api-key
    ```

    若要透過遠端代理伺服器進行非互動式設定，請明確傳入代理伺服器 URL：

    ```bash
    openclaw onboard --non-interactive --accept-risk --auth-choice litellm-api-key \
      --litellm-api-key "$LITELLM_API_KEY" --custom-base-url "https://litellm.example/v1"
    ```

  </Tab>

  <Tab title="手動設定">
    <Steps>
      <Step title="啟動 LiteLLM 代理伺服器">
        ```bash
        pip install 'litellm[proxy]'
        litellm --model claude-opus-4-6
        ```
      </Step>
      <Step title="將 OpenClaw 指向 LiteLLM">
        ```bash
        export LITELLM_API_KEY="your-litellm-key"
        openclaw
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

## 設定

```json5
{
  models: {
    providers: {
      litellm: {
        baseUrl: "http://localhost:4000",
        apiKey: "${LITELLM_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "claude-opus-4-6",
            name: "Claude Opus 4.6",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 200000,
            maxTokens: 64000,
          },
          {
            id: "gpt-4o",
            name: "GPT-4o",
            reasoning: false,
            input: ["text", "image"],
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "litellm/claude-opus-4-6" },
    },
  },
}
```

初始設定寫入的預設模型是 `litellm/claude-opus-4-6`。

## 圖片生成

LiteLLM 可透過與 OpenAI 相容的 `/images/generations` 和 `/images/edits` 路由，為 `image_generate` 工具提供後端支援。預設圖片模型是 `gpt-image-2`；若要設定不同的模型，請在 `agents.defaults.mediaModels.image` 下設定：

```json5
{
  models: {
    providers: {
      litellm: {
        baseUrl: "http://localhost:4000",
        apiKey: "${LITELLM_API_KEY}",
      },
    },
  },
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "litellm/gpt-image-2",
        timeoutMs: 180_000,
      },
    },
  },
}
```

本機迴路 LiteLLM URL（`http://localhost:4000`、`127.0.0.1`、`::1`、`host.docker.internal`）無須全域私人網路覆寫即可運作。若代理伺服器託管於區域網路，請設定 `models.providers.litellm.request.allowPrivateNetwork: true`，因為 API 金鑰會傳送至該主機。

## 進階

<AccordionGroup>
  <Accordion title="虛擬金鑰">
    為 OpenClaw 建立具支出上限的專用金鑰：

    ```bash
    curl -X POST "http://localhost:4000/key/generate" \
      -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
      -H "Content-Type: application/json" \
      -d '{
        "key_alias": "openclaw",
        "max_budget": 50.00,
        "budget_duration": "monthly"
      }'
    ```

    將產生的金鑰用作 `LITELLM_API_KEY`。

  </Accordion>

  <Accordion title="模型路由">
    LiteLLM 可將模型請求路由至不同的後端。請在你的 LiteLLM `config.yaml` 中設定：

    ```yaml
    model_list:
      - model_name: claude-opus-4-6
        litellm_params:
          model: claude-opus-4-6
          api_key: os.environ/ANTHROPIC_API_KEY

      - model_name: gpt-4o
        litellm_params:
          model: gpt-4o
          api_key: os.environ/OPENAI_API_KEY
    ```

    OpenClaw 會持續請求 `claude-opus-4-6`；LiteLLM 負責處理路由。

  </Accordion>

  <Accordion title="檢視用量">
    ```bash
    # 金鑰資訊
    curl "http://localhost:4000/key/info" \
      -H "Authorization: Bearer sk-litellm-key"

    # 支出日誌
    curl "http://localhost:4000/spend/logs" \
      -H "Authorization: Bearer $LITELLM_MASTER_KEY"
    ```

  </Accordion>

  <Accordion title="代理伺服器行為注意事項">
    - LiteLLM 預設在 `http://localhost:4000` 上執行。
    - OpenClaw 透過 LiteLLM 代理伺服器形式且與 OpenAI 相容的 `/v1` 端點連線。
    - 透過已設定的 LiteLLM 基礎 URL 時，不會套用僅限原生 OpenAI 的請求格式調整：
      不使用 `service_tier`、不使用 Responses `store`、不使用提示快取提示，也不調整 OpenAI 推理強度
      承載資料的格式。
    - 隱藏的 OpenClaw 歸屬標頭（`originator`、`version`、`User-Agent`）只會傳送至
      經驗證的原生 OpenAI 端點，因此不會注入自訂 LiteLLM 基礎 URL。
  </Accordion>
</AccordionGroup>

<Note>
如需一般供應商設定及容錯移轉行為的資訊，請參閱[模型供應商](/zh-TW/concepts/model-providers)。
</Note>

## 相關內容

<CardGroup cols={2}>
  <Card title="LiteLLM 文件" href="https://docs.litellm.ai" icon="book">
    LiteLLM 官方文件與 API 參考資料。
  </Card>
  <Card title="模型選擇" href="/zh-TW/concepts/model-providers" icon="layers">
    所有供應商、模型參照及容錯移轉行為的概覽。
  </Card>
  <Card title="設定" href="/zh-TW/gateway/configuration" icon="gear">
    完整設定參考資料。
  </Card>
  <Card title="模型" href="/zh-TW/concepts/models" icon="brain">
    如何選擇及設定模型。
  </Card>
</CardGroup>
