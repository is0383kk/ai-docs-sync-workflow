---
read_when:
    - 你想要使用一個受管理的金鑰來存取多個模型供應商
    - 你需要在 OpenClaw 中使用 ClawRouter 模型探索或配額回報功能
summary: 透過 ClawRouter 路由認證資訊範圍限定的模型，並顯示受管理的配額
title: ClawRouter
x-i18n:
    generated_at: "2026-07-26T08:38:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 929a93e8d1d003e21f792d0fdab9542553ffab374f59d4d0505819b0f719591f
    source_path: providers/clawrouter.md
    workflow: 16
---

ClawRouter 為 OpenClaw 提供一個受原則範圍限制的金鑰，用於多個上游模型
供應商。內建的 `clawrouter` 外掛只會探索該金鑰允許的模型，
依各模型宣告的通訊協定進行路由，並在 OpenClaw 的用量介面上回報
該金鑰的預算與彙總用量。

上游認證資訊與供應商特定的轉送處理都保留在 ClawRouter 中，因此
你完全不需要在 OpenClaw 主機上安裝或驗證各個上游供應商外掛。
此外掛隨 OpenClaw 內建提供（`enabledByDefault: true`）；
你只需要取得核發的 ClawRouter 認證資訊。

| 屬性          | 值                                       |
| ------------- | ---------------------------------------- |
| 供應商        | `clawrouter`                       |
| 外掛          | 內建（包含在 OpenClaw 中）               |
| 驗證          | `CLAWROUTER_API_KEY`                       |
| 預設 URL      | `https://clawrouter.openclaw.ai`                       |
| 模型目錄      | 透過 `/v1/catalog` 限定認證資訊範圍 |
| 配額          | 透過 `/v1/usage` 取得每月預算與用量 |

## 開始使用

<Steps>
  <Step title="取得限定範圍的認證資訊">
    向 ClawRouter 管理員索取認證資訊，其原則應包含
    你可使用的供應商、模型及每月預算。認證資訊核發時
    只會顯示一次。
  </Step>
  <Step title="設定 OpenClaw">
    ```bash
    export CLAWROUTER_API_KEY="..."
    openclaw onboard --auth-choice clawrouter-api-key
    openclaw plugins enable clawrouter
    ```

    `clawrouter` 已內建且預設啟用。如果你的設定包含
    `plugins.allow`，請先將 `clawrouter` 加入該清單，再啟用。此外，
    若是自訂部署，請將 `models.providers.clawrouter.baseUrl` 設為
    ClawRouter 來源；預設值為 `https://clawrouter.openclaw.ai`。

  </Step>
  <Step title="列出已授權的模型">
    ```bash
    openclaw models list --all --provider clawrouter
    ```

    請完全依照傳回內容使用模型參照。這些參照會保留上游
    命名空間，例如 `clawrouter/openai/gpt-5.5`、
    `clawrouter/anthropic/claude-sonnet-4-6` 或
    `clawrouter/google/gemini-3.5-flash`。如果已設定 `agents.defaults.modelPolicy.allow`，
    請將每個選定的 ClawRouter 參照加入其中。

  </Step>
  <Step title="選取模型">
    ```bash
    openclaw models set clawrouter/<provider>/<model>
    ```

    你也可以使用 `openclaw agent --model clawrouter/<provider>/<model> --message "..."`
    為單次執行選取傳回的模型。

  </Step>
</Steps>

## 受管理的非互動式部署

將代理伺服器金鑰保留在工作負載的密鑰注入機制中，並只在
`openclaw.json` 儲存 SecretRef。標準的受管理欄位如下：

| 用途          | 設定或環境欄位                                                           |
| ------------- | ------------------------------------------------------------------------ |
| 路由器來源    | `models.providers.clawrouter.baseUrl`                                                       |
| 認證資訊      | `models.providers.clawrouter.apiKey` -> 環境變數 SecretRef                                 |
| 密鑰值        | 閘道程序環境中的 `CLAWROUTER_API_KEY`                                      |
| 預設模型      | `agents.defaults.model.primary` -> `clawrouter/<provider>/<model>`                                 |
| 工作負載標籤  | `models.providers.clawrouter.headers.X-ClawRouter-Project-Id`（選用）                                               |

例如，部署控制器可以管理以下 JSON5 修補檔：

```json5
{
  plugins: {
    entries: { clawrouter: { enabled: true } },
  },
  models: {
    providers: {
      clawrouter: {
        baseUrl: "https://clawrouter.internal.example",
        apiKey: {
          source: "env",
          provider: "default",
          id: "CLAWROUTER_API_KEY",
        },
        headers: {
          "X-ClawRouter-Project-Id": "fakeco",
        },
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "clawrouter/openai/gpt-5.5" },
    },
  },
}
```

如果部署設定了 `plugins.allow`，請保留其現有項目並加入
`clawrouter`。不使用互動式精靈即可驗證並套用：

```bash
openclaw config patch --file ./clawrouter.patch.json5 --dry-run --json
openclaw config patch --file ./clawrouter.patch.json5
```

試執行會解析 SecretRef，但絕不會輸出其值。若要輪替
認證資訊，請更新提供 `CLAWROUTER_API_KEY` 的外部 Secret，並
重新啟動閘道工作負載，以載入新的程序環境。
設定檔與模型參照不需變更。

對於從原始碼建置的獨立 Docker 閘道，ClawRouter 已包含在
根執行階段中。只需選取需要獨立封裝的頻道外掛，
例如 `OPENCLAW_EXTENSIONS=clickclack`、`slack` 或 `msteams`；請參閱
[包含指定外掛的原始碼建置映像](/zh-TW/install/docker#source-built-images-with-selected-plugins)。
封存／設備型部署必須透過自身的成品流水線封裝相同的已合併原始碼，
而不是使用 OCI 映像。

## 就緒狀態與即時驗證

下列檢查驗證的是不同邊界；不可互相替代：

```bash
# 僅檢查 ClawRouter 程序健康狀態；不會使用認證資訊或上游模型。
curl -fsS https://clawrouter.internal.example/v1/health

# 僅檢查 OpenClaw 閘道啟動就緒狀態；不會呼叫模型。
curl -fsS http://127.0.0.1:18789/readyz

# 探索限定認證資訊範圍的目錄。
openclaw models list --all --provider clawrouter --json

# 透過已設定的 ClawRouter 供應商執行最小的實際推論探測。
openclaw models status --probe --probe-provider clawrouter --probe-max-tokens 8 --json

# 使用明確授權模型參照的工作負載金絲雀測試。
openclaw agent --agent main \
  --model clawrouter/openai/gpt-5.5 \
  --message "請完全照此回覆：CLAWROUTER_CANARY_OK" \
  --json
```

請使用限定範圍目錄所傳回的模型，而不要直接照抄範例
模型。成功的 `/readyz` 回應表示閘道可以處理
要求；這不代表 ClawRouter、其認證資訊或上游
供應商已就緒。模型探測與代理程式金絲雀測試才是推論驗證。

若要進行即時診斷，請執行金絲雀測試並檢查閘道的標準記錄。
現有僅包含中繼資料的模型傳輸診斷會輸出如下格式的行：

```text
[model-fetch] 開始 provider=clawrouter api=openai-responses model=openai/gpt-5.5 method=POST url=https://clawrouter.internal.example/v1/responses
[model-fetch] 回應 provider=clawrouter api=openai-responses model=openai/gpt-5.5 status=200
```

當這些識別碼可用時，外掛會傳送有長度限制的 `X-ClawRouter-Client`、
`X-ClawRouter-Agent-Id` 和 `X-ClawRouter-Session-Id` 標頭。它也會
將模型呼叫的診斷 `callId`（`<run-id>:model:<n>`）對應至
`X-Request-ID`，讓 OpenClaw 模型呼叫事件可與 ClawRouter
僅包含中繼資料的稽核軌跡進行關聯。在 128 字元要求 ID 預算內的值
會完全相同。較長的值會保留 `:model:<n>` 後綴與確定性
雜湊，使不同呼叫仍維持在長度限制內並可供關聯。靜態部署中繼資料
（例如 `X-ClawRouter-Project-Id`）可在供應商的 `headers` 對應表中設定。
代理程式與工作階段歸屬標頭會保留各自獨立的 256 字元
限制。若自動要求 ID 包含 ClawRouter ASCII
識別碼集合以外的字元，則會使用相同的確定性限長格式。
明確設定的標頭（包括 `X-Request-ID` 的任何大小寫變體）優先於
自動值。傳輸診斷只記錄路由與回應
中繼資料；不會記錄認證資訊、要求 ID、提示或完成內容。
ClawRouter 自身的稽核事件會提供選定的上游供應商與
內容保留狀態。

## 模型探索

`GET /v1/catalog` 會傳回 `{ providers: [...] }`，其中每個供應商項目
都會列出其自身的 `models[]`（包含上游 ID、功能與定價），以及其
支援的要求路由。OpenClaw 不會附帶另一份固定的
ClawRouter 模型清單。符合下列條件時，目錄模型會公布為 OpenClaw 模型：

- 認證資訊的原則授權其供應商；
- 目錄模型公布受支援的 LLM 功能（`llm.responses`、
  `llm.chat`、`llm.messages`，或具有相符串流
  路由的 `llm.stream`）；且
- 供應商為下列其中一種傳輸方式公開相符的路由。

將模型加入受支援的 ClawRouter 供應商不需要發布新版 OpenClaw：
下一次目錄重新整理（依認證資訊範圍快取 60 秒）就會探索到
該模型。需要新線路通訊協定的模型，則必須先由外掛提供支援。

## 通訊協定與供應商外掛

ClawRouter 管理上游認證資訊；其目錄會告知 OpenClaw 應使用哪種
傳輸方式，因此你完全不需要安裝每家上游公司的驗證外掛。

| 目錄功能／路由                                           | OpenClaw 傳輸方式       |
| -------------------------------------------------------- | ----------------------- |
| `llm.responses`（OpenAI 相容供應商）                  | `openai-responses`      |
| `llm.chat`（OpenAI 相容供應商）                  | `openai-completions`      |
| `llm.messages` + `anthropic.messages` 路由             | `anthropic-messages`      |
| `llm.stream` + 串流 `google.generate_content` 路由        | `google-generative-ai`      |

此外掛也會為這些系列套用相符的重播與工具結構描述原則
（OpenAI/DeepSeek/Gemini/Perplexity 工具結構描述相容性；原生
Anthropic 與 Google Gemini 重播原則）。Perplexity 模型會套用嚴格的
結構描述改寫：移除 `patternProperties` 與 `additionalProperties`，且
每個物件結構描述都會宣告 `properties`，因為 Perplexity 會拒絕
缺少這些內容的工具結構描述。若目錄供應商只公開
不受支援的要求格式，則會刻意不將其公布為 OpenClaw
文字模型。請在 ClawRouter 中將這些供應商正規化為
其中一種受支援的合約，而不要傳送不相容的承載內容。

## 配額與用量

ClawRouter 的 `/v1/usage` 回應會提供給一般的 OpenClaw 供應商用量
介面：要求、權杖及支出總計；如果金鑰設有限額，也會提供
每月預算期間。未計量的金鑰仍會顯示彙總用量，但不會顯示
百分比期間。

配額查詢使用與模型探索相同的限定範圍金鑰。配額
查詢失敗不會阻止模型執行。

使用以下命令檢查即時快照：

```bash
openclaw status --usage
openclaw models status
```

相同的供應商快照也可供聊天中的 `/status` 與 OpenClaw
用量介面使用。預算適用於整個原則，因此使用
相同 ClawRouter 原則的其他用戶端所發出的要求，可能會改變剩餘百分比。

## 疑難排解

| 症狀                                     | 檢查                                                                                                                                             |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| 沒有 ClawRouter 模型                     | 確認外掛已啟用且獲 `plugins.allow` 允許，接著檢查認證資訊是否有效，且至少授權一個已就緒的供應商。                                               |
| 找不到已設定的 ClawRouter 模型           | 檢查其 `/v1/catalog` 功能與路由支援。系統會刻意篩除不受支援的傳輸合約。                                                                       |
| 模型覆寫遭原則拒絕                       | 將確切的目錄參照或 `clawrouter/*` 加入 `agents.defaults.modelPolicy.allow`。                                                                                   |
| 目錄或用量傳回 `401` 或 `403` | 重新核發 ClawRouter 認證資訊或調整其範圍；OpenClaw 不會改用上游供應商金鑰作為後援。                                                               |
| 探索後模型呼叫失敗                       | 檢查 ClawRouter 中的供應商連線與上游健康狀態，待其就緒狀態恢復後再重試。                                                                           |
| 用量有總計但沒有百分比                   | 此原則未計量；請在 ClawRouter 中新增每月預算，以顯示百分比期間。                                                                                   |

## 安全性行為

- 目錄探索的範圍限定於已設定的 proxy key，並依各認證資訊範圍快取（代理程式目錄、工作區目錄、驗證設定檔 ID 及基底 URL）。
- proxy key 僅在分派請求時附加；不會儲存在模型中繼資料中。
- 自動歸屬與請求關聯值會先去除前後空白並拒絕控制字元，再進行分派。歸屬值上限為 256 個字元；請求 ID 上限為 128 個字元。
- 模型傳輸診斷僅包含中繼資料，絕不包含 proxy key 或模型內容。
- 原生 Anthropic 與 Gemini 模型 ID 僅在分派時改寫為其上游 ID。
- 不受支援或未獲授權的目錄資料列會以封閉方式失敗，且無法選取。

## 相關內容

<CardGroup cols={2}>
  <Card title="模型供應商" href="/zh-TW/concepts/model-providers" icon="layers">
    供應商設定與模型選擇。
  </Card>
  <Card title="用量追蹤" href="/zh-TW/concepts/usage-tracking" icon="chart-line">
    OpenClaw 用量與狀態介面。
  </Card>
</CardGroup>
