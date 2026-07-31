---
read_when:
    - 你想要將 Bedrock Mantle 託管的 OSS 模型與 OpenClaw 搭配使用
    - 你需要用於 GPT-OSS、Qwen、Kimi 或 GLM 的 Mantle OpenAI 相容端點
    - 你想透過 Amazon Bedrock Mantle 使用 Claude Opus 5、Sonnet 5 或 Mythos 5
summary: 搭配 OpenClaw 使用 Amazon Bedrock Mantle 的 OpenAI 相容模型與 Claude Messages 模型
title: Amazon Bedrock Mantle
x-i18n:
    generated_at: "2026-07-26T07:31:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3d2b49120560c4466aff217c3101fab057dd87c1c501f1b8eb94d74f62bd1037
    source_path: providers/bedrock-mantle.md
    workflow: 16
---

OpenClaw 內建一個隨附的 **Amazon Bedrock Mantle** 提供者，可連線至
與 OpenAI 相容的 Mantle 端點。Mantle 透過由 Bedrock 基礎架構支援的標準
`/v1/chat/completions` 介面，託管開放原始碼及
第三方模型（GPT-OSS、Qwen、Kimi、GLM 及類似模型）。Mantle 也會
透過 Anthropic Messages 路由提供 Anthropic Claude 模型。

| 屬性           | 值                                                                                     |
| -------------- | -------------------------------------------------------------------------------------- |
| 提供者 ID      | `amazon-bedrock-mantle`                                                                |
| API            | 探索到的 OSS 模型使用 `openai-completions`，Claude 模型使用 `anthropic-messages` |
| 驗證           | 明確設定 `AWS_BEARER_TOKEN_BEDROCK`，或透過 IAM 認證資訊鏈產生持有人權杖    |
| 預設區域       | `us-east-1`（可使用 `AWS_REGION` 或 `AWS_DEFAULT_REGION` 覆寫）                       |

## 開始使用

選擇偏好的驗證方式，並依照設定步驟操作。

<Tabs>
  <Tab title="明確設定持有人權杖">
    **最適合：** 已有 Mantle 持有人權杖的環境。

    <Steps>
      <Step title="在閘道主機上設定持有人權杖">
        ```bash
        export AWS_BEARER_TOKEN_BEDROCK="..."
        ```

        也可以設定區域（預設為 `us-east-1`）：

        ```bash
        export AWS_REGION="us-west-2"
        ```
      </Step>
      <Step title="確認已探索到模型">
        ```bash
        openclaw models list
        ```

        探索到的模型會顯示在 `amazon-bedrock-mantle` 提供者下。除非你想覆寫預設值，
        否則不需要額外設定。
      </Step>
    </Steps>

  </Tab>

  <Tab title="IAM 認證資訊">
    **最適合：** 使用與 AWS SDK 相容的認證資訊（共用設定、SSO、Web 身分、執行個體或工作角色）。

    <Steps>
      <Step title="在閘道主機上設定 AWS 認證資訊">
        任何與 AWS SDK 相容的驗證來源皆可使用：

        ```bash
        export AWS_PROFILE="default"
        export AWS_REGION="us-west-2"
        ```
      </Step>
      <Step title="確認已探索到模型">
        ```bash
        openclaw models list
        ```

        OpenClaw 會自動從認證資訊鏈產生 Mantle 持有人權杖。
      </Step>
    </Steps>

    <Tip>
    未設定 `AWS_BEARER_TOKEN_BEDROCK` 時，OpenClaw 會從 AWS 預設認證資訊鏈為你簽發持有人權杖，包括共用認證資訊／設定檔、SSO、Web 身分，以及執行個體或工作角色。
    </Tip>

  </Tab>
</Tabs>

## 自動探索模型

設定 `AWS_BEARER_TOKEN_BEDROCK` 時，OpenClaw 會直接使用它。否則，
OpenClaw 會嘗試從 AWS 預設認證資訊鏈產生 Mantle 持有人權杖，
接著查詢該區域的 `/v1/models` 端點，
探索可用的 Mantle 模型。

| 行為              | 詳細資訊                                                                             |
| ----------------- | ------------------------------------------------------------------------------------ |
| 探索快取          | 每個區域的結果會快取 1 小時；擷取失敗時會傳回最後一次快取的結果 |
| IAM 權杖重新整理  | 每 2 小時一次，依區域分別快取                                                       |

若要讓 Mantle 外掛保持啟用，但停用自動探索和 IAM
持有人權杖產生功能，請停用外掛所擁有的探索切換設定：

```bash
openclaw config set plugins.entries.amazon-bedrock-mantle.config.discovery.enabled false
```

<Note>
此持有人權杖與標準 [Amazon Bedrock](/zh-TW/providers/bedrock) 提供者所使用的 `AWS_BEARER_TOKEN_BEDROCK` 相同。
</Note>

### 支援的區域

`us-east-1`、`us-east-2`、`us-west-2`、`ap-northeast-1`、
`ap-south-1`、`ap-southeast-3`、`eu-central-1`、`eu-west-1`、`eu-west-2`、
`eu-south-1`、`eu-north-1`、`sa-east-1`。

## 手動設定

若你偏好使用明確設定，而非自動探索：

```json5
{
  models: {
    providers: {
      "amazon-bedrock-mantle": {
        baseUrl: "https://bedrock-mantle.us-east-1.api.aws/v1",
        api: "openai-completions",
        auth: "api-key",
        apiKey: "env:AWS_BEARER_TOKEN_BEDROCK",
        models: [
          {
            id: "gpt-oss-120b",
            name: "GPT-OSS 120B",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32000,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

明確指定且非空的 `models` 清單具有最高優先權，會取代每一個
探索到的項目，包括下方的 Claude 項目。省略 `models` 可保留
自動產生的 Mantle 目錄；或者納入你想使用的完整 Claude 模型項目。

## 進階設定

<AccordionGroup>
  <Accordion title="推理支援">
    系統會根據模型 ID 是否包含 `thinking`、`reasoner`、
    `reasoning`、`deepseek.r`、`gpt-oss-120b` 或
    `gpt-oss-safeguard-120b` 等模式來推斷推理支援。探索期間，OpenClaw 會自動為
    相符的模型設定 `reasoning: true`。
  </Accordion>

  <Accordion title="端點無法使用">
    如果 Mantle 端點無法使用、未傳回任何模型，或無法解析持有人權杖，
    探索會傳回空結果，並略過隱含提供者。OpenClaw 不會報錯；
    其他已設定的提供者會繼續正常運作。
  </Accordion>

  <Accordion title="透過 Anthropic Messages 路由使用 Claude">
    當模型清單由自動探索管理時，OpenClaw 會在成功查詢後附加五個 Claude
    模型，不受 `/v1/models` 傳回內容影響：
    `amazon-bedrock-mantle/anthropic.claude-opus-5`（Claude Opus 5）、
    `amazon-bedrock-mantle/anthropic.claude-sonnet-5`（Claude Sonnet 5）、
    `amazon-bedrock-mantle/anthropic.claude-opus-4-7`（Claude Opus 4.7）及
    `amazon-bedrock-mantle/anthropic.claude-mythos-5`（Claude Mythos 5），另加
    `amazon-bedrock-mantle/anthropic.claude-mythos-preview`（Claude Mythos
    預覽版）。它們使用 `anthropic-messages` API 介面，並透過
    相同、以持有人權杖驗證且與 Anthropic 相容的端點
    （`<mantle-base>/anthropic`）進行串流，因此 AWS 持有人權杖不會被視為
    Anthropic API 金鑰。

    Claude Opus 5 提供 1,000,000 個權杖的內容視窗、128,000 個權杖的
    輸出上限、圖片輸入及 `$5/$25` 輸入／輸出定價。自適應
    思考預設為 `high`；`/think off` 會停用思考，
    而 `/think xhigh|max` 會使用模型原生的投入程度。OpenClaw 會省略
    呼叫端選擇的取樣參數。

    Claude Sonnet 5 一律使用自適應思考，預設投入程度為 `high`。
    `/think off` 和 `/think minimal` 會對應至 `low`，因為 Mantle
    路由無法停用思考。OpenClaw 也會省略 Sonnet 5 請求中的自訂溫度。

    Claude Mythos 5 僅限部分使用者存取。它提供 1,000,000 個權杖的內容
    視窗及 128,000 個權杖的輸出上限，一律使用自適應思考，將
    `/think off` 和 `/think minimal` 對應至 `low`，並省略呼叫端選擇的
    取樣參數。

    Claude Mythos 預覽版一律要求推理；未設定 `/think` 等級時，
    預設採用 `high` 投入程度（將 `xhigh`/`max` 下調至
    `high`，並將 `minimal` 上調至 `low`）。Mantle 上的 Opus 4.7
    會在模型未提供推理的情況下進行串流，而 OpenClaw 會省略其
    `temperature` 參數，因為 Opus 4.7 在此路由上不接受取樣覆寫；
    Mythos 預覽版則照常接受 `temperature` 覆寫。

    非空的明確 `models.providers["amazon-bedrock-mantle"].models`
    清單會取代完整的探索目錄。若要使用這些內建 Claude 項目，
    請省略該清單。

  </Accordion>

  <Accordion title="與 Amazon Bedrock 提供者的關係">
    Bedrock Mantle 與標準 [Amazon Bedrock](/zh-TW/providers/bedrock) 提供者
    是不同的提供者。Mantle 的 OSS 目錄使用與 OpenAI 相容的
    `/v1` 介面，而標準 Bedrock 提供者則使用原生 Bedrock Converse API。

    兩個提供者都會在 `AWS_BEARER_TOKEN_BEDROCK` 認證資訊存在時共用它。

  </Accordion>
</AccordionGroup>

## 相關內容

<CardGroup cols={2}>
  <Card title="Amazon Bedrock" href="/zh-TW/providers/bedrock" icon="cloud">
    適用於 Anthropic Claude、Titan 及其他模型的原生 Bedrock 提供者。
  </Card>
  <Card title="模型選擇" href="/zh-TW/concepts/model-providers" icon="layers">
    選擇提供者、模型參照及容錯移轉行為。
  </Card>
  <Card title="OAuth 與驗證" href="/zh-TW/gateway/authentication" icon="key">
    驗證詳細資訊及認證資訊重複使用規則。
  </Card>
  <Card title="疑難排解" href="/zh-TW/help/troubleshooting" icon="wrench">
    常見問題及其解決方式。
  </Card>
</CardGroup>
