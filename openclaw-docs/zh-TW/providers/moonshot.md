---
read_when:
    - 你想設定 Moonshot Kimi K3/K2（Moonshot Open Platform）還是 Kimi Coding
    - 你需要瞭解各自獨立的端點、金鑰與模型參照。
    - 你需要任一供應商可直接複製貼上的設定
summary: 設定 Moonshot Kimi 模型與 Kimi Coding（分開的供應商與金鑰）
title: 月之暗面 AI
x-i18n:
    generated_at: "2026-07-26T08:46:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 213379bf88fec26b052184a920e112f0887d6485601bfb47f590cf37ef983e58
    source_path: providers/moonshot.md
    workflow: 16
---

Moonshot 透過相容於 OpenAI 的端點提供 Kimi API。為 Kimi K3 選取
`moonshot/kimi-k3`，保留新手引導預設值
`moonshot/kimi-k2.6`，或為 Kimi Coding 使用 `kimi/kimi-for-coding`。

<Warning>
Moonshot 與 Kimi Coding 是**不同的供應商**，各自以獨立的外部外掛提供。金鑰不可互換、端點不同，模型參照也不同（`moonshot/...` 與 `kimi/...`）。
</Warning>

## 內建模型目錄

[//]: # "moonshot-kimi-k2-ids:start"

| 模型參照                            | 名稱                     | 推理       | 輸入          | 上下文    | 最大輸出   |
| ----------------------------------- | ------------------------ | ---------- | ------------- | --------- | ---------- |
| `moonshot/kimi-k2.6`                | Kimi K2.6                | 否         | 文字、圖片    | 262,144   | 262,144    |
| `moonshot/kimi-k3`                  | Kimi K3                  | 永遠最大   | 文字、圖片    | 1,048,576 | 1,048,576  |
| `moonshot/kimi-k2.7-code`           | Kimi K2.7 Code           | 永遠開啟   | 文字、圖片    | 262,144   | 262,144    |
| `moonshot/kimi-k2.7-code-highspeed` | Kimi K2.7 Code HighSpeed | 永遠開啟   | 文字、圖片    | 262,144   | 262,144    |
| `moonshot/kimi-k2.5`                | Kimi K2.5                | 否         | 文字、圖片    | 262,144   | 262,144    |

[//]: # "moonshot-kimi-k2-ids:end"

目錄成本估算使用 Moonshot 公布的隨用隨付費率。在做出成本
決策前，請查看 [Kimi K3](https://platform.kimi.ai/docs/pricing/chat-k3)、
[Kimi K2.7 Code](https://platform.kimi.ai/docs/pricing/chat-k27-code)、
[Kimi K2.6](https://platform.kimi.ai/docs/pricing/chat-k26) 和
[Kimi K2.5](https://platform.kimi.ai/docs/pricing/chat-k25) 的即時供應商頁面。

Kimi K3 永遠以 `reasoning_effort: "max"` 進行推理。OpenClaw 僅公開
`/think max`，省略僅限 K2 的 `thinking` 欄位，並移除 K3
固定為供應商預設值的取樣覆寫（`temperature`、`top_p`、`n`、`presence_penalty` 和
`frequency_penalty`）。Kimi K2.7 Code 也永遠使用原生思考，但要求同時省略
`thinking` 和 `reasoning_effort`；HighSpeed 變體使用相同的合約。
Kimi K2.6 仍是新手引導的預設值。
請參閱 Moonshot 的 [Kimi K3 快速入門](https://platform.kimi.ai/docs/guide/kimi-k3-quickstart)。

## 開始使用

Moonshot 與 Kimi Coding 都是外部外掛——請先安裝其中一個，再進行
新手引導。

<Tabs>
  <Tab title="Moonshot API">
    **最適合：**透過 Moonshot Open Platform 使用 Kimi K3 與 K2 模型。

    <Steps>
      <Step title="安裝外掛">
        ```bash
        openclaw plugins install @openclaw/moonshot-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="選擇端點區域">
        | 驗證選項               | 端點                           | 區域          |
        | ---------------------- | ------------------------------ | ------------- |
        | `moonshot-api-key`     | `https://api.moonshot.ai/v1`   | 國際          |
        | `moonshot-api-key-cn`  | `https://api.moonshot.cn/v1`   | 中國          |
      </Step>
      <Step title="執行新手引導">
        ```bash
        openclaw onboard --auth-choice moonshot-api-key
        ```

        或使用中國端點：

        ```bash
        openclaw onboard --auth-choice moonshot-api-key-cn
        ```
      </Step>
      <Step title="將 Kimi K3 設為預設模型">
        新手引導會保留 Kimi K2.6 作為初始預設值。想使用 Kimi K3 時，
        請明確切換：

        ```bash
        openclaw models set moonshot/kimi-k3
        ```
      </Step>
      <Step title="確認模型可用">
        ```bash
        openclaw models list --provider moonshot
        ```
      </Step>
      <Step title="執行即時冒煙測試">
        若想在不影響一般工作階段的情況下驗證模型存取與成本
        追蹤，請使用隔離的狀態目錄：

        ```bash
        OPENCLAW_CONFIG_PATH=/tmp/openclaw-kimi/openclaw.json \
        OPENCLAW_STATE_DIR=/tmp/openclaw-kimi \
        openclaw agent --local \
          --session-id live-kimi-cost \
          --message 'Reply exactly: KIMI_LIVE_OK' \
          --thinking max \
          --json
        ```

        JSON 回應應回報 `provider: "moonshot"` 和
        `model: "kimi-k3"`。當 Moonshot 傳回用量中繼資料時，助理逐字稿項目會在
        `usage.cost` 下儲存正規化的 Token 用量與估算成本。
      </Step>
    </Steps>

    ### 設定範例

    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: {
            // moonshot-kimi-k2-aliases:start
            "moonshot/kimi-k2.6": { alias: "Kimi K2.6" },
            "moonshot/kimi-k3": { alias: "Kimi K3" },
            "moonshot/kimi-k2.7-code": { alias: "Kimi K2.7 Code" },
            "moonshot/kimi-k2.7-code-highspeed": { alias: "Kimi K2.7 Code HighSpeed" },
            "moonshot/kimi-k2.5": { alias: "Kimi K2.5" },
            // moonshot-kimi-k2-aliases:end
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              // moonshot-kimi-k2-models:start
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k3",
                name: "Kimi K3",
                reasoning: true,
                thinkingLevelMap: {
                  off: null,
                  minimal: null,
                  low: null,
                  medium: null,
                  high: null,
                  xhigh: "max",
                  max: "max",
                },
                input: ["text", "image"],
                cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 0 },
                contextWindow: 1048576,
                maxTokens: 1048576,
              },
              {
                id: "kimi-k2.7-code",
                name: "Kimi K2.7 Code",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.19, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.7-code-highspeed",
                name: "Kimi K2.7 Code HighSpeed",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 1.9, output: 8, cacheRead: 0.38, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.5",
                name: "Kimi K2.5",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.6, output: 3, cacheRead: 0.1, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              // moonshot-kimi-k2-models:end
            ],
          },
        },
      },
    }
    ```

  </Tab>

  <Tab title="Kimi Coding">
    **最適合：**透過 Kimi Coding 端點執行以程式碼為主的工作。

    <Note>
    Kimi Coding 使用的 API 金鑰與供應商前綴（`kimi/...`）不同於 Moonshot（`moonshot/...`）。目前的參照包括使用 256K 上下文的 `kimi/k3`、使用 1M 等級的 `kimi/k3[1m]`、`kimi/kimi-for-coding` 和 `kimi/kimi-for-coding-highspeed`。舊版參照 `kimi/kimi-code` 和 `kimi/k2p5` 仍可接受，並會正規化為 `kimi/kimi-for-coding`。
    </Note>

    此程式設計服務同時接受相容於 OpenAI 的
    `https://api.kimi.com/coding/v1` 用戶端與相容於 Anthropic 的
    `https://api.kimi.com/coding/` 用戶端。此外掛使用 Anthropic Messages。
    請在 [Kimi Code Console](https://www.kimi.com/code/console) 建立會員金鑰；目前的會員
    定價請參閱 [Kimi 定價頁面](https://www.kimi.com/membership/pricing)。

    <Steps>
      <Step title="安裝外掛">
        ```bash
        openclaw plugins install @openclaw/kimi-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="執行新手引導">
        ```bash
        openclaw onboard --auth-choice kimi-code-api-key
        ```
      </Step>
      <Step title="設定預設模型">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "kimi/kimi-for-coding" },
            },
          },
        }
        ```
      </Step>
      <Step title="確認模型可用">
        ```bash
        openclaw models list --provider kimi
        ```
      </Step>
    </Steps>

    Kimi Code K3 預設在 `max` 使用深度思考。`/think off` 會傳送
    `thinking.type: "disabled"`；`/think max` 會傳送 K3 的自適應思考
    請求，並使用最大推理強度。過時的較低思考等級會解析為受支援的
    `max` 等級。1M 模型需要 Allegretto 或更高等級的 Kimi
    會員資格；Moderato 請使用 `kimi/k3`。

    如需目前方案的可用情況，請參閱官方 [Kimi Code 模型表](https://www.kimi.com/code/docs/en/kimi-code/models.html)。

    ### 設定範例

    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: {
            "kimi/kimi-for-coding": { alias: "Kimi" },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

## Kimi 網頁搜尋

Moonshot 外掛也會將 **Kimi** 註冊為 `web_search` 供應商，其後端使用 Moonshot 網頁搜尋。

<Steps>
  <Step title="執行互動式網頁搜尋設定">
    ```bash
    openclaw configure --section web
    ```

    在網頁搜尋區段中選擇 **Kimi**，以儲存
    `plugins.entries.moonshot.config.webSearch.*`。

  </Step>
  <Step title="設定網頁搜尋區域與模型">
    互動式設定會提示以下項目：

    | 設定                | 選項                                                                 |
    | ------------------- | -------------------------------------------------------------------- |
    | API 區域            | `https://api.moonshot.ai/v1`（國際）或 `https://api.moonshot.cn/v1`（中國） |
    | 網頁搜尋模型        | 預設為 `kimi-k2.6`                                             |

  </Step>
</Steps>

設定位於 `plugins.entries.moonshot.config.webSearch` 下：

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // or use KIMI_API_KEY / MOONSHOT_API_KEY
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.6",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

## 進階設定

<AccordionGroup>
  <Accordion title="原生思考模式">
    Moonshot API Kimi K3 永遠以最大推理強度進行推理。OpenClaw 僅公開
    `/think max`、傳送 `reasoning_effort: "max"`，並忽略過時的較低等級或
    `off` 設定。

    Kimi Code K3 提供 `/think off|max`。其 Anthropic 相容端點
    會接收 `thinking.type: "disabled"` 以關閉思考，或使用自適應思考並透過
    `output_config.effort: "max"` 設為最大值。這同時適用於 `kimi/k3` 與
    `kimi/k3[1m]`。
    Moonshot API K3 支援 `auto`、`none`、`required` 及固定工具選擇，
    因此 OpenClaw 會保留請求的 `tool_choice`。對於多輪工具使用，
    OpenClaw 會保留 Moonshot 重播合約所要求的助理推理內容。

    Kimi K2.7 Code 一律使用原生思考。Moonshot 要求用戶端針對此模型
    省略 `thinking` 欄位，因此 OpenClaw 僅提供 `on`，並
    忽略過時的 `off` 設定。K2.7 也會固定 `temperature`、`top_p`、`n`、
    `presence_penalty` 及 `frequency_penalty`；OpenClaw 會省略這些欄位已設定的
    覆寫值。

    其他 Moonshot Kimi 模型支援二元原生思考：

    - `thinking: { type: "enabled" }`
    - `thinking: { type: "disabled" }`

    透過 `agents.defaults.models.<provider/model>.params` 為各模型進行設定：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "disabled" },
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw 會為這些模型對應執行階段的 `/think` 等級：

    | `/think` 等級       | Moonshot 行為          |
    | -------------------- | -------------------------- |
    | `/think off`         | `thinking.type=disabled`   |
    | 任何非關閉等級    | `thinking.type=enabled`    |

    <Warning>
    啟用 Moonshot K2 思考時，`tool_choice` 必須是 `auto` 或 `none`。固定工具選擇（`type: "tool"` 或 `type: "function"`）會改為強制將思考恢復成 `disabled`，以確保請求的工具仍會執行；`tool_choice: "required"` 則會改為正規化成 `auto`。Kimi K2.7 Code 無法停用思考，因此其不相容的 `tool_choice` 會正規化成 `auto`。Kimi K3 使用獨立的推理強度合約，並保留支援的工具選擇。
    </Warning>

    Kimi K2.6 也接受選用的 `thinking.keep` 欄位，用於控制
    多輪對話中 `reasoning_content` 的保留方式。將其設為 `"all"`，即可跨輪保留完整
    推理；省略此欄位（或維持為 `null`）則會使用伺服器
    的預設策略。OpenClaw 僅會為
    `moonshot/kimi-k2.6` 轉送 `thinking.keep`，並從其他模型中移除此欄位。Kimi K2.7 Code
    預設會保留完整推理歷程，而 OpenClaw 會省略整個
    `thinking` 欄位。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "enabled", keep: "all" },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="工具呼叫 ID 清理">
    Moonshot Kimi 提供形如 `functions.<name>:<index>` 的原生 tool_call ID。OpenClaw 會保留每個原生 Kimi ID 的第一次出現，並將後續重複項目改寫為確定性的 OpenAI 風格 `call_*` ID。相符的工具結果會使用相同 ID 重新對應，因此無須移除 Kimi 的第一個原生 ID，也能確保重播時的 ID 唯一。此行為已整合至隨附的 Moonshot 提供者，並非使用者可設定的選項。
  </Accordion>

  <Accordion title="串流用量相容性">
    原生 Moonshot 端點（`https://api.moonshot.ai/v1` 與
    `https://api.moonshot.cn/v1`）宣告支援串流用量相容性。
    OpenClaw 依據端點主機而非提供者 ID 判斷，因此指向相同原生
    Moonshot 主機的自訂提供者 ID 會繼承相同的串流用量行為。

    使用目錄中的 K2.6 定價時，包含輸入、輸出與
    快取讀取權杖的串流用量，也會轉換為本機預估美元成本，供
    `/status`、`/usage full`、`/usage cost` 及由逐字稿支援的工作階段
    計費使用。

  </Accordion>

  <Accordion title="端點與模型參照參考">
    | 提供者   | 模型參照前綴 | 端點                      | 驗證環境變數        |
    | ---------- | ---------------- | ------------------------------ | ------------------- |
    | Moonshot   | `moonshot/`      | `https://api.moonshot.ai/v1`  | `MOONSHOT_API_KEY`  |
    | Moonshot CN| `moonshot/`      | `https://api.moonshot.cn/v1`  | `MOONSHOT_API_KEY`  |
    | Kimi Coding| `kimi/`          | Kimi Coding 端點           | `KIMI_API_KEY`      |
    | 網頁搜尋 | 不適用              | 與 Moonshot API 區域相同    | `KIMI_API_KEY` 或 `MOONSHOT_API_KEY` |

    - Kimi 網頁搜尋使用 `KIMI_API_KEY` 或 `MOONSHOT_API_KEY`，並預設採用 `https://api.moonshot.ai/v1` 搭配模型 `kimi-k2.6`。
    - 如有需要，請在 `models.providers` 中覆寫定價與情境中繼資料。
    - 如果 Moonshot 為某個模型發布不同的情境限制，請相應調整 `contextWindow`。

  </Accordion>
</AccordionGroup>

## 相關內容

<CardGroup cols={2}>
  <Card title="模型選擇" href="/zh-TW/concepts/model-providers" icon="layers">
    選擇提供者、模型參照及容錯移轉行為。
  </Card>
  <Card title="網頁搜尋" href="/zh-TW/tools/web" icon="magnifying-glass">
    設定包括 Kimi 在內的網頁搜尋提供者。
  </Card>
  <Card title="設定參考" href="/zh-TW/gateway/configuration-reference" icon="gear">
    提供者、模型與外掛的完整設定結構描述。
  </Card>
  <Card title="Moonshot 開放平台" href="https://platform.moonshot.ai" icon="globe">
    Moonshot API 金鑰管理與文件。
  </Card>
</CardGroup>
