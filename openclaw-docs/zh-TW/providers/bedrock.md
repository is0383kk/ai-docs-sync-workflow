---
read_when:
    - 你想要搭配 OpenClaw 使用 Amazon Bedrock 模型
    - 你需要設定 AWS 認證資訊／區域，才能呼叫模型
summary: 搭配 OpenClaw 使用 Amazon Bedrock（Converse API）模型
title: Amazon Bedrock
x-i18n:
    generated_at: "2026-07-26T07:53:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9cbc9534c0d06e0d5642b8d167c633c16880908812b97adbbf9c6bd6c5511603
    source_path: providers/bedrock.md
    workflow: 16
---

OpenClaw 可透過其 **Bedrock Converse** 串流供應商使用 **Amazon Bedrock** 模型。Bedrock 驗證使用 **AWS SDK 預設認證資訊鏈**，而非 API 金鑰。

| 屬性     | 值                                                          |
| -------- | ----------------------------------------------------------- |
| 供應商   | `amazon-bedrock`                                            |
| API      | `bedrock-converse-stream`                                   |
| 驗證     | AWS 認證資訊（環境變數、共用設定或執行個體角色）            |
| 區域     | `AWS_REGION` 或 `AWS_DEFAULT_REGION`（預設：`us-east-1`） |

## 開始使用

選擇偏好的驗證方式，並依照設定步驟操作。

<Tabs>
  <Tab title="存取金鑰／環境變數">
    **最適合：**開發人員電腦、CI，或由你直接管理 AWS 認證資訊的主機。

    <Steps>
      <Step title="在閘道主機上設定 AWS 認證資訊">
        ```bash
        export AWS_ACCESS_KEY_ID="EXAMPLE_AWS_ACCESS_KEY_ID"
        export AWS_SECRET_ACCESS_KEY="..."
        export AWS_REGION="us-east-1"
        # 選用：
        export AWS_SESSION_TOKEN="..."
        export AWS_PROFILE="your-profile"
        # 選用（Bedrock API 金鑰／Bearer 權杖）：
        export AWS_BEARER_TOKEN_BEDROCK="..."
        ```
      </Step>
      <Step title="將 Bedrock 供應商和模型加入設定">
        不需要 `apiKey`。請使用 `auth: "aws-sdk"` 設定供應商：

        ```json5
        {
          models: {
            providers: {
              "amazon-bedrock": {
                baseUrl: "https://bedrock-runtime.us-east-1.amazonaws.com",
                api: "bedrock-converse-stream",
                auth: "aws-sdk",
                models: [
                  {
                    id: "us.anthropic.claude-opus-4-6-v1",
                    name: "Claude Opus 4.6 (Bedrock)",
                    reasoning: true,
                    input: ["text", "image"],
                    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                    contextWindow: 200000,
                    maxTokens: 8192,
                  },
                ],
              },
            },
          },
          agents: {
            defaults: {
              model: { primary: "amazon-bedrock/us.anthropic.claude-opus-4-6-v1" },
            },
          },
        }
        ```
      </Step>
      <Step title="確認模型可用">
        ```bash
        openclaw models list
        ```
      </Step>
    </Steps>

    <Tip>
    使用環境標記驗證（`AWS_ACCESS_KEY_ID`、`AWS_PROFILE` 或 `AWS_BEARER_TOKEN_BEDROCK`）時，OpenClaw 會自動啟用隱含的 Bedrock 供應商以探索模型，不需要額外設定。
    </Tip>

  </Tab>

  <Tab title="EC2 執行個體角色（IMDS）">
    **最適合：**已附加 IAM 角色，並使用執行個體中繼資料服務進行驗證的 EC2 執行個體。

    <Steps>
      <Step title="明確啟用探索">
        使用 IMDS 時，OpenClaw 無法只透過環境標記偵測 AWS 驗證，因此你必須選擇啟用：

        ```bash
        openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
        openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1
        ```
      </Step>
      <Step title="選擇性加入自動模式所需的環境標記">
        如果也想讓環境標記自動偵測路徑運作（例如用於 `openclaw status` 介面）：

        ```bash
        export AWS_PROFILE=default
        export AWS_REGION=us-east-1
        ```

        你**不需要**虛假的 API 金鑰。
      </Step>
      <Step title="確認已探索到模型">
        ```bash
        openclaw models list
        ```
      </Step>
    </Steps>

    <Warning>
    附加至 EC2 執行個體的 IAM 角色必須具備下列權限：

    - `bedrock:InvokeModel`
    - `bedrock:InvokeModelWithResponseStream`
    - `bedrock:ListFoundationModels`（用於自動探索）
    - `bedrock:ListInferenceProfiles`（用於推論設定檔探索）

    或附加受管理的政策 `AmazonBedrockFullAccess`。
    </Warning>

    <Note>
    只有在你特別需要自動模式或狀態介面使用環境標記時，才需要 `AWS_PROFILE=default`。實際的 Bedrock 執行階段驗證路徑使用 AWS SDK 預設鏈，因此即使沒有環境標記，IMDS 執行個體角色驗證仍可運作。
    </Note>

  </Tab>
</Tabs>

## 自動模型探索

OpenClaw 可自動探索支援**串流**和**文字輸出**的 Bedrock 模型。探索使用 `bedrock:ListFoundationModels` 和
`bedrock:ListInferenceProfiles`，且結果會快取（預設：1 小時）。

隱含供應商的啟用方式：

- 如果 `plugins.entries.amazon-bedrock.config.discovery.enabled` 為 `true`，
  即使沒有 AWS 環境標記，OpenClaw 仍會嘗試探索。
- 如果未設定 `plugins.entries.amazon-bedrock.config.discovery.enabled`，
  OpenClaw 只有在看到下列任一 AWS 驗證標記時，才會自動加入
  隱含的 Bedrock 供應商：
  `AWS_BEARER_TOKEN_BEDROCK`、`AWS_ACCESS_KEY_ID` +
  `AWS_SECRET_ACCESS_KEY`，或 `AWS_PROFILE`。
- 實際的 Bedrock 執行階段驗證路徑仍使用 AWS SDK 預設鏈，因此即使探索需要透過 `enabled: true` 選擇啟用，
  共用設定、SSO 和 IMDS 執行個體角色驗證仍可運作。

<Note>
對於明確的 `models.providers["amazon-bedrock"]` 項目，OpenClaw 仍可透過 `AWS_BEARER_TOKEN_BEDROCK` 等 AWS 環境標記提早解析 Bedrock 環境標記驗證，而不強制載入完整的執行階段驗證。實際的模型呼叫驗證路徑仍使用 AWS SDK 預設鏈。
</Note>

<AccordionGroup>
  <Accordion title="探索設定選項">
    設定選項位於 `plugins.entries.amazon-bedrock.config.discovery`：

    ```json5
    {
      plugins: {
        entries: {
          "amazon-bedrock": {
            config: {
              discovery: {
                enabled: true,
                region: "us-east-1",
                providerFilter: ["anthropic", "amazon"],
                refreshInterval: 3600,
                defaultContextWindow: 32000,
                defaultMaxTokens: 4096,
              },
            },
          },
        },
      },
    }
    ```

    | 選項 | 預設值 | 說明 |
    | ---- | ------ | ---- |
    | `enabled` | 自動 | 在自動模式下，OpenClaw 只有在看到支援的 AWS 環境標記時，才會啟用隱含的 Bedrock 供應商。設為 `true` 可強制探索。 |
    | `region` | `AWS_REGION` / `AWS_DEFAULT_REGION` / `us-east-1` | 探索 API 呼叫所使用的 AWS 區域。 |
    | `providerFilter` |（全部）| 比對 Bedrock 供應商名稱（例如 `anthropic`、`amazon`）。 |
    | `refreshInterval` | `3600` | 快取持續時間（秒）。設為 `0` 可停用快取。 |
    | `defaultContextWindow` | `32000` | 用於沒有已知權杖限制之已探索模型的內容窗口（如果知道模型限制，請覆寫此值）。 |
    | `defaultMaxTokens` | `4096` | 用於沒有已知權杖限制之已探索模型的最大輸出權杖數（如果知道模型限制，請覆寫此值）。 |

  </Accordion>

  <Accordion title="內容窗口和最大權杖限制">
    Bedrock 的 `ListFoundationModels` 和 `GetFoundationModel` API 不會傳回
    權杖限制中繼資料，只會傳回模型 ID、名稱、模態和生命週期
    狀態。OpenClaw 隨附常見 Bedrock 模型（Claude、Nova、Llama、Mistral、DeepSeek
    等）的已知內容窗口與輸出限制查詢表，讓工作階段管理、壓縮門檻和
    內容溢位偵測可針對這些模型正確運作。

    不在表格中的已探索模型會回退至 `defaultContextWindow`
    和 `defaultMaxTokens`。如果你使用的模型缺少正確限制，
    請以明確的
    `models.providers["amazon-bedrock"].models` 項目覆寫。

  </Accordion>
</AccordionGroup>

## 快速設定（AWS 路徑）

本逐步指南會建立 IAM 角色、附加 Bedrock 權限、建立執行個體設定檔關聯，
並在 EC2 主機上啟用 OpenClaw 探索。

```bash
# 1. 建立 IAM 角色和執行個體設定檔
aws iam create-role --role-name EC2-Bedrock-Access \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy --role-name EC2-Bedrock-Access \
  --policy-arn arn:aws:iam::aws:policy/AmazonBedrockFullAccess

aws iam create-instance-profile --instance-profile-name EC2-Bedrock-Access
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-Bedrock-Access \
  --role-name EC2-Bedrock-Access

# 2. 附加至你的 EC2 執行個體
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxx \
  --iam-instance-profile Name=EC2-Bedrock-Access

# 3. 在 EC2 執行個體上明確啟用探索
openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1

# 4. 選用：若要在不明確啟用的情況下使用自動模式，請加入環境標記
echo 'export AWS_PROFILE=default' >> ~/.bashrc
echo 'export AWS_REGION=us-east-1' >> ~/.bashrc
source ~/.bashrc

# 5. 確認已探索到模型
openclaw models list
```

## 進階設定

<AccordionGroup>
  <Accordion title="推論設定檔">
    OpenClaw 會在探索基礎模型的同時探索**區域和全域推論設定檔**。當設定檔對應至已知的基礎模型時，
    該設定檔會繼承模型的能力（內容窗口、最大權杖數、
    推理、視覺），並自動注入正確的 Bedrock 要求區域。
    這表示跨區域 Claude 設定檔不需要手動覆寫
    供應商即可運作。全域跨區域設定檔（`global.*`）在
    `openclaw models list` 中會列於最前面，因為它們通常能提供更好的容量
    和自動容錯移轉。

    推論設定檔 ID 的格式類似 `us.anthropic.claude-opus-4-6-v1`（區域）
    或 `anthropic.claude-opus-4-6-v1`（全域）。如果支援該設定檔的模型已存在於
    探索結果中，設定檔會繼承其完整能力集合；
    否則會套用安全的預設值。

    不需要額外設定。只要已啟用探索，且 IAM
    主體具備 `bedrock:ListInferenceProfiles`，設定檔就會與
    基礎模型一起出現在 `openclaw models list` 中。

  </Accordion>

  <Accordion title="服務層級">
    部分 Bedrock 模型支援 `service_tier` 參數，以針對成本
    或延遲進行最佳化。可用的層級如下：

    | 層級 | 說明 |
    |------|------|
    | `default` | 標準 Bedrock 層級 |
    | `flex` | 適用於可容忍較長延遲之工作負載的折扣處理 |
    | `priority` | 適用於延遲敏感工作負載的優先處理 |
    | `reserved` | 適用於穩態工作負載的保留容量 |

    請透過 `agents.defaults.params` 為 Bedrock 模型要求設定
    `serviceTier`（或 `service_tier`），或在
    `agents.defaults.models["<model-key>"].params` 中針對個別模型設定：

    ```json5
    {
      agents: {
        defaults: {
          params: {
            serviceTier: "flex", // 套用至所有模型
          },
          models: {
            "amazon-bedrock/mistral.mistral-large-3-675b-instruct": {
              params: {
                serviceTier: "priority", // 個別模型覆寫
              },
            },
          },
        },
      },
    }
    ```

    有效值為 `default`、`flex`、`priority` 和 `reserved`。Claude
    Fable 5、Opus 5 和 Sonnet 5 僅支援 `default` 層級；若為這些模型要求使用
    `flex`、`priority` 或 `reserved`，OpenClaw 會發出警告並
    忽略該要求。對於其他模型，並非每個模型都支援所有層級——不支援的層級
    會傳回 Bedrock 驗證錯誤，而且錯誤訊息可能會
    造成誤導（例如顯示「提供的模型識別碼無效」，
    而不是指出問題出在層級）。如果看到此錯誤，請確認
    該模型是否支援要求的層級。

  </Accordion>

  <Accordion title="Claude Opus 5、4.8 和 4.7 的溫度">
    Bedrock 會拒絕 Claude Opus 5、Opus 4.8 和 Opus 4.7 的
    `temperature` 參數。OpenClaw 會自動針對任何相符的 Bedrock
    參照省略 `temperature`，包括基礎模型 ID、具名推論設定檔、底層模型透過
    `bedrock:GetInferenceProfile` 解析為 Opus 5/4.8/4.7 的應用程式
    推論設定檔，以及可選擇加上區域前綴的點分隔
    `opus-4.7`/`opus-4.8` 變體（`us.`、`eu.`、`ap.`、`apac.`、`au.`、`jp.`、
    `global.`）。不需要任何設定開關，且此省略同時套用於
    要求選項物件和 `inferenceConfig` 承載內容欄位。
  </Accordion>

  <Accordion title="Claude Opus 5">
    在 Messages API 的 Bedrock
    端點上使用 `amazon-bedrock/anthropic.claude-opus-5`，或當區域／全域推論設定檔（例如
    `global.anthropic.claude-opus-5`）出現在 Bedrock 探索結果中時使用該設定檔。
    OpenClaw 會套用 1,000,000 個權杖的上下文視窗、128,000 個權杖的輸出
    上限、圖片輸入、提示快取、拒絕安全串流，以及原生
    `xhigh`/`max` 力度等級。

    自適應思考預設為 `high`。`/think off` 會停用思考，而
    `/think xhigh|max` 會讓自適應思考保持啟用。OpenClaw 會省略自訂
    取樣參數和不受支援的非預設服務層級。

  </Accordion>

  <Accordion title="Claude Fable 5">
    在 `us-east-1` 中使用 `amazon-bedrock/anthropic.claude-fable-5`，或使用
    `us.anthropic.claude-fable-5` 等區域推論 ID。
    OpenClaw 會套用 Fable 的 1M 上下文視窗、128K 輸出上限、永遠啟用的
    自適應思考，以及支援的力度對應。`/think off` 和
    `/think minimal` 會對應至 `low`；溫度和強制工具選擇控制項
    會被省略，與 Opus 4.7/4.8 路徑一致。串流輸出會保留
    至 Bedrock 傳回終止狀態，以免串流途中發生拒絕時
    洩漏部分文字。

    Fable 必須先在 AWS 中明確選擇加入 `provider_data_share` 資料保留機制，
    才能使用。提示與完成內容會與 Anthropic 分享，並
    最多保留 30 天，以供信任與安全用途。啟用模型前，請先檢閱並設定
    [Bedrock 資料保留](https://docs.aws.amazon.com/bedrock/latest/userguide/data-retention.html)。

  </Accordion>

  <Accordion title="Claude Mythos 5">
    只有獲得所需有限存取核准的帳戶，才能透過 Bedrock 使用
    Claude Mythos 5。OpenClaw 可辨識基礎模型
    `anthropic.claude-mythos-5`，以及
    `us.anthropic.claude-mythos-5` 等區域或全域推論設定檔。

    OpenClaw 會套用 1,000,000 個權杖的上下文視窗、128,000 個權杖的輸出
    上限、圖片輸入、提示快取、拒絕安全串流，以及原生
    力度等級。自適應思考永遠啟用：`/think off` 和
    `/think minimal` 會對應至 `low`，而 `xhigh` 和 `max` 仍可使用。
    自訂取樣和強制工具選擇值會被省略。

  </Accordion>

  <Accordion title="Claude Sonnet 5">
    AWS 文件說明 Sonnet 5 同時支援
    [`bedrock-runtime` 和 `bedrock-mantle` 端點](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5.html)。
    OpenClaw 可辨識 Bedrock 基礎模型
    `anthropic.claude-sonnet-5`，以及
    `us.anthropic.claude-sonnet-5` 等區域或全域推論設定檔。它會套用 1,000,000 個權杖的上下文
    視窗、128,000 個權杖的輸出上限、圖片輸入、原生力度等級、
    提示快取和拒絕安全串流。

    Bedrock 會讓 Sonnet 5 的自適應思考保持啟用。OpenClaw 預設使用
    `high`；由於此路徑無法停用思考，`/think off` 和 `/think minimal`
    會對應至 `low`。啟用自適應思考時，
    自訂溫度和強制工具選擇值會被省略。

  </Accordion>

  <Accordion title="防護機制">
    你可以將 [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
    套用至所有 Bedrock 模型叫用，方法是在
    `amazon-bedrock` 外掛設定中加入 `guardrail` 物件。防護機制可讓你強制執行內容篩選、
    主題拒絕、字詞篩選、敏感資訊篩選和情境
    依據檢查。

    ```json5
    {
      plugins: {
        entries: {
          "amazon-bedrock": {
            config: {
              guardrail: {
                guardrailIdentifier: "abc123", // 防護機制 ID 或完整 ARN
                guardrailVersion: "1", // 版本號碼或 "DRAFT"
                streamProcessingMode: "sync", // 選用："sync" 或 "async"
                trace: "enabled", // 選用："enabled"、"disabled" 或 "enabled_full"
              },
            },
          },
        },
      },
    }
    ```

    `guardrailIdentifier` 和 `guardrailVersion` 為必填。

    | 選項 | 說明 |
    | ------ | ----------- |
    | `guardrailIdentifier` | 防護機制 ID（例如 `abc123`）或完整 ARN（例如 `arn:aws:bedrock:us-east-1:123456789012:guardrail/abc123`）。 |
    | `guardrailVersion` | 已發布的版本號碼，或使用 `"DRAFT"` 代表工作草稿。 |
    | `streamProcessingMode` | 在串流期間進行防護機制評估時使用 `"sync"` 或 `"async"`。若省略，Bedrock 會使用其預設值。 |
    | `trace` | 偵錯時使用 `"enabled"` 或 `"enabled_full"`；正式環境請省略或設為 `"disabled"`。 |

    <Warning>
    閘道使用的 IAM 主體除了標準叫用權限外，還必須具備 `bedrock:ApplyGuardrail` 權限。
    </Warning>

  </Accordion>

  <Accordion title="記憶搜尋的嵌入">
    Bedrock 也可以作為
    [記憶搜尋](/zh-TW/concepts/memory-search)的嵌入提供者。這與推論提供者分開設定——請將 `memory.search.provider` 設為 `"bedrock"`：

    ```json5
    {
      memory: {
        search: {
          provider: "bedrock",
          model: "amazon.titan-embed-text-v2:0", // 預設值
        },
      },
    }
    ```

    Bedrock 嵌入使用與推論相同的 AWS SDK 認證資訊鏈（執行個體
    角色、SSO、存取金鑰、共用設定和 Web 身分）。不需要
    API 金鑰。

    支援的嵌入模型包括 Amazon Titan Embed（v1、v2）、Amazon Nova
    Embed、Cohere Embed（v3、v4）和 TwelveLabs Marengo。完整模型清單和維度選項請參閱
    [記憶設定參考——Bedrock](/zh-TW/reference/memory-config#bedrock-embedding-config)。

  </Accordion>

  <Accordion title="注意事項與限制">
    - Bedrock 要求你的 AWS 帳戶／區域已啟用**模型存取權**。
    - 自動探索需要 `bedrock:ListFoundationModels` 和
      `bedrock:ListInferenceProfiles` 權限。
    - 如果依賴自動模式，請在閘道主機上設定其中一個受支援的 AWS 驗證環境標記。
      如果偏好使用不含環境標記的 IMDS／共用設定驗證，請設定
      `plugins.entries.amazon-bedrock.config.discovery.enabled: true`。
    - OpenClaw 會依下列順序顯示認證資訊來源：`AWS_BEARER_TOKEN_BEDROCK`，
      接著是 `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`，再來是 `AWS_PROFILE`，最後是
      預設 AWS SDK 鏈。
    - 推理支援依模型而異；請查閱 Bedrock 模型卡以瞭解
      目前的功能。
    - 如果偏好受管理的金鑰流程，也可以在 Bedrock 前方部署與 OpenAI 相容的
      Proxy，並改為將其設定成 OpenAI 提供者。
  </Accordion>
</AccordionGroup>

## 相關內容

<CardGroup cols={2}>
  <Card title="模型選擇" href="/zh-TW/concepts/model-providers" icon="layers">
    選擇提供者、模型參照和容錯移轉行為。
  </Card>
  <Card title="記憶搜尋" href="/zh-TW/concepts/memory-search" icon="magnifying-glass">
    用於記憶搜尋設定的 Bedrock 嵌入。
  </Card>
  <Card title="記憶設定參考" href="/zh-TW/reference/memory-config#bedrock-embedding-config" icon="database">
    完整的 Bedrock 嵌入模型清單和維度選項。
  </Card>
  <Card title="疑難排解" href="/zh-TW/help/troubleshooting" icon="wrench">
    一般疑難排解和常見問題。
  </Card>
</CardGroup>
