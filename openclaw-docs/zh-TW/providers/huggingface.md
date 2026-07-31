---
read_when:
    - 你想要搭配 OpenClaw 使用 Hugging Face Inference
    - 你需要 HF 權杖環境變數或命令列介面驗證選項
summary: Hugging Face 推論設定（驗證 + 模型選擇）
title: Hugging Face（推論）
x-i18n:
    generated_at: "2026-07-26T07:54:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 92c400b78c5ad2cc724ad4029560dccc5bc2006fdeae400fc6b58998e727e17c
    source_path: providers/huggingface.md
    workflow: 16
---

[Hugging Face 推論供應商](https://huggingface.co/docs/inference-providers)透過單一權杖，為許多託管模型（DeepSeek、Llama 等）提供與 OpenAI 相容的聊天補全路由器。OpenClaw **僅使用聊天補全端點**；若需文字轉圖片、嵌入或語音功能，請直接使用 [HF 推論用戶端](https://huggingface.co/docs/api-inference/quicktour)。

| 屬性         | 值                                                                                                                          |
| ------------ | --------------------------------------------------------------------------------------------------------------------------- |
| 供應商 ID    | `huggingface`                                                                                                          |
| 外掛         | 內建（預設啟用，無須安裝）                                                                                                  |
| 驗證環境變數 | `HUGGINGFACE_HUB_TOKEN` 或 `HF_TOKEN`（細粒度權杖）                                                                       |
| API          | 與 OpenAI 相容（`https://router.huggingface.co/v1`）                                                                                         |
| 計費         | 單一 HF 權杖；[定價](https://huggingface.co/docs/inference-providers/pricing)依循各供應商費率，並提供免費方案                 |

## 開始使用

<Steps>
  <Step title="建立細粒度權杖">
    前往 [Hugging Face Settings Tokens](https://huggingface.co/settings/tokens/new?ownUserPermissions=inference.serverless.write&tokenType=fineGrained)，建立新的細粒度權杖。

    <Warning>
    權杖必須啟用 **Make calls to Inference Providers** 權限，否則 API 請求會遭拒絕。
    </Warning>

  </Step>
  <Step title="執行初始設定">
    在供應商下拉式選單中選擇 **Hugging Face**，然後在提示出現時輸入你的 API 金鑰：

    ```bash
    openclaw onboard --auth-choice huggingface-api-key
    ```

  </Step>
  <Step title="選取預設模型">
    在 **Default Hugging Face model** 下拉式選單中選取模型。權杖有效時，清單會從 Inference API 載入；否則 OpenClaw 會顯示下方的內建目錄。你的選擇會儲存為 `agents.defaults.model.primary`：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
        },
      },
    }
    ```

  </Step>
  <Step title="確認模型可用">
    ```bash
    openclaw models list --provider huggingface
    ```
  </Step>
</Steps>

### 非互動式設定

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice huggingface-api-key \
  --huggingface-api-key "$HF_TOKEN"
```

將 `huggingface/deepseek-ai/DeepSeek-R1` 設為預設模型。

## 模型 ID

模型參照使用 `huggingface/<org>/<model>` 格式（Hub 樣式 ID）。OpenClaw 的內建目錄如下：

| 模型          | 參照（加上 `huggingface/` 前綴） |
| ------------- | ------------------------------------- |
| DeepSeek R1   | `deepseek-ai/DeepSeek-R1`                    |
| DeepSeek V3.1 | `deepseek-ai/DeepSeek-V3.1`                    |
| GPT-OSS 120B  | `openai/gpt-oss-120b`                    |

<Tip>
權杖有效時，OpenClaw 也會在初始設定及閘道啟動時，透過 **GET** `https://router.huggingface.co/v1/models` 探索任何其他模型，因此你的目錄能包含遠多於上述三個模型的項目。你可以在任何模型 ID 後附加 `:fastest` 或 `:cheapest`；HF 的路由器會將其路由至相符的推論供應商。請在 [Inference Provider settings](https://hf.co/settings/inference-providers)中設定預設供應商順序。
</Tip>

## 進階設定

<AccordionGroup>
  <Accordion title="模型探索與初始設定下拉式選單">
    OpenClaw 使用以下方式探索模型：

    ```bash
    GET https://router.huggingface.co/v1/models
    Authorization: Bearer $HUGGINGFACE_HUB_TOKEN   # or $HF_TOKEN
    ```

    回應採用 OpenAI 樣式：`{ "object": "list", "data": [ { "id": "Qwen/Qwen3-8B", "owned_by": "Qwen", ... }, ... ] }`。

    設定金鑰後（透過初始設定、`HUGGINGFACE_HUB_TOKEN` 或 `HF_TOKEN`），互動式設定期間的 **Default Hugging Face model** 下拉式選單會由此端點填入。閘道啟動時會重複相同呼叫以重新整理目錄。探索到的模型會與上述內建目錄合併（ID 相符時，內建目錄會用於提供內容窗口和成本等中繼資料）。如果請求失敗、未傳回資料或未設定金鑰，OpenClaw 只會退回使用內建目錄。

    若要停用探索而不移除供應商：

    ```bash
    openclaw config set plugins.entries.huggingface.config.discovery.enabled false
    ```

  </Accordion>

  <Accordion title="模型名稱、別名與原則後綴">
    - **API 提供的名稱：**探索到的模型會在 API 的 `name`、`title` 或 `display_name` 存在時使用該值；否則 OpenClaw 會從模型 ID 衍生名稱（例如 `deepseek-ai/DeepSeek-R1` 會變成 “DeepSeek R1”）。
    - **覆寫顯示名稱：**在設定中為各模型設定自訂標籤：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1 (fast)" },
            "huggingface/deepseek-ai/DeepSeek-R1:cheapest": { alias: "DeepSeek R1 (cheap)" },
          },
        },
      },
    }
    ```

    - **原則後綴：**`:fastest` 和 `:cheapest` 是 HF 路由器的慣例，並非 OpenClaw 會改寫的內容：後綴會原封不動地作為模型 ID 的一部分傳送，而 HF 路由器會選取相符的推論供應商。如果你希望每個後綴都有不同的別名，請將各變體分別新增為 `models.providers.huggingface.models` 下的獨立項目（或新增至 `model.primary`）。
    - **設定合併：**設定合併時會保留 `models.providers.huggingface.models` 中的現有項目（例如 `models.json` 中的項目），因此你在其中設定的任何自訂 `name`、`alias` 或模型選項都會在重新啟動後保留。

  </Accordion>

  <Accordion title="環境與常駐程式設定">
    如果閘道以常駐程式（launchd/systemd）方式執行，請確認該程序能存取 `HUGGINGFACE_HUB_TOKEN` 或 `HF_TOKEN`（例如透過 `~/.openclaw/.env` 或 `env.shellEnv`）。

    <Note>
    OpenClaw 同時接受 `HUGGINGFACE_HUB_TOKEN` 和 `HF_TOKEN`。如果兩者皆已設定，則以 `HUGGINGFACE_HUB_TOKEN` 為優先。
    </Note>

  </Accordion>

  <Accordion title="設定：使用 DeepSeek R1 並提供備援">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "huggingface/deepseek-ai/DeepSeek-R1",
            fallbacks: ["huggingface/openai/gpt-oss-120b"],
          },
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1" },
            "huggingface/openai/gpt-oss-120b": { alias: "GPT-OSS 120B" },
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="設定：使用 DeepSeek 的最便宜和最快變體">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1" },
            "huggingface/deepseek-ai/DeepSeek-R1:cheapest": { alias: "DeepSeek R1 (cheapest)" },
            "huggingface/deepseek-ai/DeepSeek-R1:fastest": { alias: "DeepSeek R1 (fastest)" },
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="設定：使用 DeepSeek + GPT-OSS 並設定別名">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "huggingface/deepseek-ai/DeepSeek-V3.1",
            fallbacks: ["huggingface/openai/gpt-oss-120b"],
          },
          models: {
            "huggingface/deepseek-ai/DeepSeek-V3.1": { alias: "DeepSeek V3.1" },
            "huggingface/openai/gpt-oss-120b": { alias: "GPT-OSS 120B" },
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## 相關內容

<CardGroup cols={2}>
  <Card title="模型選取" href="/zh-TW/concepts/model-providers" icon="layers">
    所有供應商、模型參照及容錯移轉行為的概覽。
  </Card>
  <Card title="模型選取" href="/zh-TW/concepts/models" icon="brain">
    如何選擇及設定模型。
  </Card>
  <Card title="推論供應商文件" href="https://huggingface.co/docs/inference-providers" icon="book">
    Hugging Face 推論供應商官方文件。
  </Card>
  <Card title="設定" href="/zh-TW/gateway/configuration" icon="gear">
    完整設定參考。
  </Card>
</CardGroup>
