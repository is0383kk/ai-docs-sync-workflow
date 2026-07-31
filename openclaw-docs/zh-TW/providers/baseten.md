---
read_when:
    - 你想在 OpenClaw 中執行 Thinking Machines Lab 的 Inkling
    - 你想要一個適用於 Baseten 託管模型且與 OpenAI 相容的 API
summary: Inkling 與託管模型 API 的 Baseten 設定
title: Baseten
x-i18n:
    generated_at: "2026-07-26T08:45:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2ccc3b5cf64b01859f9f022d7bc15a69a1cb42c87d4f914c118276c1151020de
    source_path: providers/baseten.md
    workflow: 16
---

[Baseten 模型 API](https://docs.baseten.co/inference/model-apis/overview)提供託管且相容於 OpenAI 的前沿模型存取方式。官方外部外掛使用經過驗證的探索機制，因此 OpenClaw 會採用你的 Baseten 帳戶所啟用的完整模型集。其離線備援包含建置此 OpenClaw 版本時可用的所有模型 API。

| 屬性            | 值                                                       |
| --------------- | -------------------------------------------------------- |
| 提供者 ID       | `baseten`                                       |
| 外掛            | 官方外部套件（`@openclaw/baseten-provider`）                       |
| 驗證環境變數    | `BASETEN_API_KEY`                                       |
| 初始設定旗標    | `--auth-choice baseten-api-key`                                       |
| 直接命令列旗標  | `--baseten-api-key <key>`                                       |
| API             | 相容於 OpenAI（`openai-completions`）                      |
| 基礎 URL        | `https://inference.baseten.co/v1`                                       |
| 預設模型        | `baseten/thinkingmachines/inkling`                                       |

## 安裝外掛

```bash
openclaw plugins install @openclaw/baseten-provider
openclaw gateway restart
```

## 開始使用

<Steps>
  <Step title="建立 Baseten 帳戶與 API 金鑰">
    Baseten 的 Basic 方案沒有每月平台費用；模型 API 呼叫按用量計價。請在 [Baseten API 金鑰設定](https://app.baseten.co/settings/api_keys)中建立金鑰，並在[定價頁面](https://www.baseten.co/pricing)查看目前費率。
  </Step>
  <Step title="執行初始設定">
    <CodeGroup>

```bash Onboarding
openclaw onboard --auth-choice baseten-api-key
```

```bash Direct flag
openclaw onboard --non-interactive \
  --auth-choice baseten-api-key \
  --baseten-api-key "$BASETEN_API_KEY"
```

```bash Env only
export BASETEN_API_KEY=...
```

    </CodeGroup>

  </Step>
  <Step title="驗證即時目錄">
    ```bash
    openclaw models list --provider baseten
    ```

    若有可用的驗證資訊，外掛會請求 `GET /v1/models`，並列出該帳戶所傳回的所有模型。若沒有驗證資訊，則會維持離線並使用隨附的備援目錄。

  </Step>
</Steps>

## Inkling

[Thinking Machines Lab 的 Inkling](https://thinkingmachines.ai/news/introducing-inkling/)是預設模型。在 OpenClaw 中，它支援文字與圖片輸入、工具呼叫、結構化工具結構描述、可設定的推理強度、1.048M 個權杖的上下文視窗，以及最多 32k 個輸出權杖：

```json5
{
  agents: {
    defaults: {
      model: { primary: "baseten/thinkingmachines/inkling" },
    },
  },
}
```

使用 `/model baseten/thinkingmachines/inkling` 切換現有聊天的模型。

## 隨附的備援目錄

經過驗證的即時目錄具有最高權威。探索成功前，以下項目可讓設定與模型選擇功能維持可用：

| 模型參照                                           | 輸入        | 上下文 | 最大輸出 |
| -------------------------------------------------- | ----------- | ------: | ---------: |
| `baseten/deepseek-ai/DeepSeek-V4-Pro`              | 文字        |    262k |       262k |
| `baseten/zai-org/GLM-4.7`                          | 文字        |    200k |       200k |
| `baseten/zai-org/GLM-5`                            | 文字        |    202k |       202k |
| `baseten/zai-org/GLM-5.1`                          | 文字        |    202k |       202k |
| `baseten/zai-org/GLM-5.2`                          | 文字        |    202k |       202k |
| `baseten/thinkingmachines/inkling`                 | 文字、圖片 |  1.048M |        32k |
| `baseten/moonshotai/Kimi-K2.5`                     | 文字、圖片 |    262k |       262k |
| `baseten/moonshotai/Kimi-K2.6`                     | 文字、圖片 |    262k |       262k |
| `baseten/moonshotai/Kimi-K2.7-Code`                | 文字、圖片 |    262k |       262k |
| `baseten/nvidia/Nemotron-120B-A12B`                | 文字        |    202k |       202k |
| `baseten/nvidia/NVIDIA-Nemotron-3-Ultra-550B-A55B` | 文字        |    202k |       202k |
| `baseten/openai/gpt-oss-120b`                      | 文字        |    128k |       128k |

所有隨附模型皆支援工具呼叫與推理。OpenClaw 會將其思考層級對應至具備原生 `reasoning_effort` 的模型。Baseten 的選擇性啟用 GLM、Kimi 與 Nemotron 模型預設會關閉思考功能；多數模型提供關閉／開啟二元控制，而 GLM 5.2 則提供關閉、高與最大。OpenClaw 會透過 Baseten 的 `chat_template_args.enable_thinking` 控制傳送這些選項；對於 GLM 5.2，還會使用經驗證的頂層 `reasoning_effort` 參數。

<Note>
Baseten 可以獨立於 OpenClaw 版本新增、移除或變更模型 API。外掛會透過經過驗證的 API 重新整理模型 ID、上下文限制、輸出限制，以及輸入、快取輸入與輸出定價，同時保留各模型專屬的 OpenClaw 傳輸原則。
</Note>

## 手動設定

多數設定只需要 API 金鑰。若要明確指定提供者：

```json5
{
  env: { BASETEN_API_KEY: "..." },
  agents: {
    defaults: {
      model: { primary: "baseten/thinkingmachines/inkling" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      baseten: {
        baseUrl: "https://inference.baseten.co/v1",
        apiKey: "${BASETEN_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "thinkingmachines/inkling",
            name: "Inkling",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 1048000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<Note>
如果閘道以常駐程式執行（launchd、systemd、Docker），請確認該程序可以使用 `BASETEN_API_KEY`。僅在互動式 shell 中匯出的金鑰，已在執行中的受管理服務無法看見。
</Note>

## 相關內容

<CardGroup cols={2}>
  <Card title="模型提供者" href="/zh-TW/concepts/model-providers" icon="layers">
    選擇提供者、模型參照與容錯移轉行為。
  </Card>
  <Card title="思考模式" href="/zh-TW/tools/thinking" icon="brain">
    選取 OpenClaw 推理強度層級。
  </Card>
  <Card title="模型命令列介面" href="/zh-TW/cli/models" icon="terminal">
    列出、檢查及選取探索到的模型。
  </Card>
  <Card title="模型常見問題" href="/zh-TW/help/faq-models" icon="circle-question">
    驗證設定檔與模型選擇疑難排解。
  </Card>
</CardGroup>
