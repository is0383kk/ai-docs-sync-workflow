---
read_when:
    - 調整代理程式預設值（模型、思考、工作區、心跳偵測、媒體、Skills）
    - 設定多代理路由與繫結
    - 調整工作階段、訊息傳遞與對話模式行為
summary: 代理程式預設值、多代理程式路由、工作階段、訊息與對話設定
title: 設定 — 代理程式
x-i18n:
    generated_at: "2026-07-26T08:23:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7a161d65b02e3333c15a2d998421419ee37d36be4d02ebb3a86e66282df06adb
    source_path: gateway/config-agents.md
    workflow: 16
---

`agents.*`、`multiAgent.*`、`session.*`、
`messages.*` 和 `talk.*` 下的代理程式範圍設定鍵。關於頻道、工具、閘道執行階段及其他
頂層鍵，請參閱[設定參考](/zh-TW/gateway/configuration-reference)。

## 代理程式預設值

### `agents.defaults.workspace`

預設值：若已設定則為 `OPENCLAW_WORKSPACE_DIR`，否則為 `~/.openclaw/workspace`（若 `OPENCLAW_PROFILE` 設為非預設設定檔，則為 `~/.openclaw/workspace-<profile>`）。

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

明確的 `agents.defaults.workspace` 值優先於
`OPENCLAW_WORKSPACE_DIR`。若不想將路徑寫入設定，可使用環境變數讓預設代理程式
指向已掛載的工作區。

### `agents.defaults.repoRoot`

顯示於系統提示詞「Runtime」行中的選用儲存庫根目錄。若未設定，OpenClaw 會從工作區開始向上逐層自動偵測。

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

供未設定
`agents.entries.*.skills` 的代理程式使用的選用預設 Skill 允許清單。

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" }, // 繼承 github、weather
      { id: "docs", skills: ["docs-search"] }, // 取代預設值
      { id: "locked-down", skills: [] }, // 不使用 Skill
    ],
  },
}
```

- 若預設不限制 Skill，請省略 `agents.defaults.skills`。
- 若要繼承預設值，請省略 `agents.entries.*.skills`。
- 若不使用任何 Skill，請設定 `agents.entries.*.skills: []`。
- 非空的 `agents.entries.*.skills` 清單是該代理程式的最終集合；
  不會與預設值合併。

### `agents.defaults.skipBootstrap`

停用自動建立工作區啟動檔案（`AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`BOOTSTRAP.md`）。

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.skipOptionalBootstrapFiles`

略過建立所選的選用工作區檔案，但仍會寫入必要的啟動檔案（`AGENTS.md`、`TOOLS.md`、`BOOTSTRAP.md`）。有效值：`SOUL.md`、`USER.md` 和 `IDENTITY.md`（可接受 `HEARTBEAT.md`，但因心跳偵測情境已移至排程監控暫存區，所以不會產生任何作用）。

```json5
{
  agents: {
    defaults: {
      skipOptionalBootstrapFiles: ["SOUL.md", "USER.md"],
    },
  },
}
```

### `agents.defaults.contextInjection`

控制何時將工作區啟動檔案注入系統提示詞。預設值：`"always"`。

- `"continuation-skip"`：安全的接續回合（在助理完成回覆後）會略過重新注入工作區啟動內容，以減少提示詞大小。心跳偵測執行及壓縮後的重試仍會重建情境。
- `"never"`：在每個回合停用工作區啟動內容與情境檔案注入。僅供完全自行管理提示詞生命週期的代理程式使用（自訂情境引擎、自行建構情境的原生執行階段，或不使用啟動內容的特殊工作流程）。心跳偵測與壓縮復原回合也會略過注入。

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

每個代理程式的覆寫值：`agents.entries.*.contextInjection`。省略的值會繼承
`agents.defaults.contextInjection`。

### `agents.defaults.bootstrapMaxChars`

每個工作區啟動檔案在截斷前的字元數上限。預設值：`20000`。

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

每個代理程式的覆寫值：`agents.entries.*.bootstrapMaxChars`。省略的值會繼承
`agents.defaults.bootstrapMaxChars`。

### `agents.defaults.bootstrapTotalMaxChars`

所有工作區啟動檔案合計注入的字元數上限。預設值：`60000`。

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 60000 } },
}
```

每個代理程式的覆寫值：`agents.entries.*.bootstrapTotalMaxChars`。省略的值會
繼承 `agents.defaults.bootstrapTotalMaxChars`。

### 每個代理程式的啟動設定檔覆寫值

當某個代理程式需要與共用預設值不同的提示詞
注入行為時，請使用每個代理程式的啟動設定檔覆寫值。省略的欄位會繼承自
`agents.defaults`。

```json5
{
  agents: {
    defaults: {
      contextInjection: "continuation-skip",
      bootstrapMaxChars: 20000,
      bootstrapTotalMaxChars: 60000,
    },
    list: [
      {
        id: "strict-worker",
        contextInjection: "always",
        bootstrapMaxChars: 50000,
        bootstrapTotalMaxChars: 300000,
      },
    ],
  },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

控制啟動內容遭截斷時，代理程式可見的系統提示詞通知。
預設值：`"always"`。

- `"off"`：永不將截斷通知文字注入系統提示詞。
- `"once"`：每種不重複的截斷特徵僅注入一次簡短通知。
- `"always"`：存在截斷時，每次執行都注入簡短通知（建議）。

詳細的原始／已注入計數與設定調校欄位會保留在情境／狀態報告及記錄等
診斷資訊中；一般 WebChat 使用者／執行階段情境僅會收到簡短的復原通知。

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "always" } }, // off | once | always
}
```

### 情境預算權責對照表

OpenClaw 有多個高容量提示詞／情境預算，這些預算刻意依子系統
分開管理，而非全部透過單一通用
控制項處理。

| 預算                                                         | 涵蓋範圍                                                                                                                                                          |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agents.defaults.bootstrapMaxChars` / `bootstrapTotalMaxChars` | 一般工作區啟動內容注入                                                                                                                            |
| `agents.defaults.startupContext.*`                             | 一次性的重設／啟動模型執行前置內容，包括近期每日的 `memory/*.md` 檔案。純聊天 `/new` 和 `/reset` 會直接確認，而不叫用模型 |
| `skills.limits.*`                                              | 注入系統提示詞的精簡 Skills 清單                                                                                                         |
| `agents.defaults.contextLimits.*`                              | 有界限的執行階段摘錄及注入的執行階段自有區塊                                                                                                      |
| `memory.qmd.limits.*`                                          | 已建立索引的記憶搜尋片段與注入大小                                                                                                              |

對應的每個代理程式覆寫值：

- `agents.entries.*.skillsLimits.maxSkillsPromptChars`
- `agents.entries.*.contextInjection`
- `agents.entries.*.bootstrapMaxChars`
- `agents.entries.*.bootstrapTotalMaxChars`
- `agents.entries.*.contextLimits.*`

#### `agents.defaults.startupContext`

控制在重設／啟動模型執行時，於第一個回合注入的啟動前置內容。
純聊天 `/new` 和 `/reset` 命令會確認重設，但不叫用
模型，因此不會載入此前置內容。

```json5
{
  agents: {
    defaults: {
      startupContext: {
        enabled: true,
        applyOn: ["new", "reset"],
        dailyMemoryDays: 2,
        maxFileBytes: 16384,
        maxFileChars: 1200,
        maxTotalChars: 2800,
      },
    },
  },
}
```

#### `agents.defaults.contextLimits`

有界限執行階段情境介面的共用預設值。

```json5
{
  agents: {
    defaults: {
      contextLimits: {
        memoryGetMaxChars: 12000,
        postCompactionMaxChars: 1800,
      },
    },
  },
}
```

- `memoryGetMaxChars`：加入截斷
  中繼資料與接續通知前，`memory_get` 摘錄的預設上限。
- 當 `memory_get` 省略 `lines` 時，OpenClaw 會使用內建的 120 行範圍，
  然後套用 `memoryGetMaxChars`。
- 即時工具結果會使用依模型情境自動調整的上限：低於 100K
  個權杖時為 `16000` 個字元，達 100K+ 個權杖時為 `32000` 個字元，達 200K+ 個權杖時則為 `64000` 個字元。
- `postCompactionMaxChars`：壓縮後重新整理注入期間使用的 AGENTS.md 摘錄上限。

#### `agents.entries.*.contextLimits`

共用 `contextLimits` 控制項的每個代理程式覆寫值。省略的欄位會繼承
自 `agents.defaults.contextLimits`。

```json5
{
  agents: {
    defaults: {
      contextLimits: { memoryGetMaxChars: 12000 },
    },
    list: [
      {
        id: "tiny-local",
        contextLimits: {
          memoryGetMaxChars: 6000,
        },
      },
    ],
  },
}
```

#### `skills.limits.maxSkillsPromptChars`

注入系統提示詞之精簡 Skills 清單的全域上限。這
不會影響依需求讀取 `SKILL.md` 檔案。

```json5
{
  skills: { limits: { maxSkillsPromptChars: 18000 } },
}
```

#### `agents.entries.*.skillsLimits.maxSkillsPromptChars`

Skills 提示詞預算的每個代理程式覆寫值。

```json5
{
  agents: {
    list: [{ id: "tiny-local", skillsLimits: { maxSkillsPromptChars: 6000 } }],
  },
}
```

### `agents.defaults.imageMaxDimensionPx`

在呼叫提供者前，逐字記錄／工具圖片區塊中圖片最長邊的像素大小上限。
預設值：`1200`。

較低的值通常可減少大量使用螢幕截圖之執行作業的視覺權杖用量及要求承載內容大小。
較高的值則可保留更多視覺細節。

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.imageQuality`

針對從檔案路徑、URL 和媒體參照載入的圖片，設定圖片工具的壓縮／細節偏好。
預設值：`auto`。

OpenClaw 會依所選圖片模型調整縮放層級。例如 Claude Opus 4.8、OpenAI GPT-5.6 Sol、Qwen VL 及託管的 Llama 4 視覺模型，可以使用比舊版／預設高細節視覺路徑更大的圖片；而多圖片回合在 `auto` 模式下會進行更積極的壓縮，以控制權杖與延遲成本。

值：

- `auto`：依模型限制及圖片數量調整。
- `efficient`：優先使用較小的圖片，以降低權杖與位元組用量。
- `balanced`：使用標準的折衷縮放層級。
- `high`：為螢幕截圖、圖表及文件圖片保留更多細節。

```json5
{
  agents: { defaults: { imageQuality: "auto" } },
}
```

### `agents.defaults.userTimezone`

系統提示詞情境所使用的時區（非訊息時間戳記）。未設定時使用主機時區。

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

系統提示詞中的時間格式。預設值：`auto`（作業系統偏好設定）。

```json5
{
  agents: { defaults: { timeFormat: "auto" } }, // auto | 12 | 24
}
```

### `agents.defaults.model`

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
      utilityModel: "openai/gpt-5.4-mini",
      imageModel: {
        primary: "openrouter/qwen/qwen-2.5-vl-72b-instruct:free",
        fallbacks: ["openrouter/google/gemini-2.0-flash-vision:free"],
      },
      mediaModels: {
        image: {
          primary: "openai/gpt-image-2",
          fallbacks: ["google/gemini-3.1-flash-image"],
        },
        video: {
          primary: "qwen/wan2.6-t2v",
          fallbacks: ["qwen/wan2.6-i2v"],
        },
      },
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      params: { cacheRetention: "long" }, // 全域預設提供者參數
      pdfMaxMb: 10,
      pdfMaxPages: 20,
      thinkingDefault: "low",
      verboseDefault: "off",
      toolProgressDetail: "explain",
      reasoningDefault: "off",
      elevatedDefault: "on",
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      contextTokens: 200000,
      maxConcurrent: 4,
    },
  },
}
```

- `model`：接受字串（`"provider/model"`）或物件（`{ primary, fallbacks }`）。
  - 字串形式只設定主要模型。
  - 物件形式設定主要模型及依序排列的容錯移轉模型。
- `utilityModel`：選用的 `provider/model` 參照或別名，用於簡短的內部工作。目前用於產生 Control UI 工作階段標題、Telegram 私訊主題標題、Discord 自動討論串標題，以及[進度草稿旁白](/zh-TW/concepts/progress-drafts#narrated-status)。未設定時，若主要供應商宣告了小型模型預設值，OpenClaw 會採用該值（OpenAI → `gpt-5.6-luna`、Anthropic → `claude-haiku-4-5`）；否則標題工作會使用代理程式的主要模型，旁白則維持關閉。若個別公用模型無法準備或完成產生的標題，OpenClaw 會改用主要模型重試該標題一次。對於儀表板標題，自動公用模型推導及一般後援會使用有效的工作階段供應商與驗證設定檔；明確指定的公用模型則保留其設定的供應商／驗證。設定 `utilityModel: ""` 可略過替代公用模型路徑；儀表板標題仍會直接使用一般工作階段模型產生。`agents.entries.*.utilityModel` 會覆寫預設值，而作業專用的模型覆寫優先於兩者。公用工作會進行個別的模型呼叫，並將工作專用內容傳送至所選模型供應商。儀表板標題產生最多會傳送第一則非命令訊息的前 1,000 個字元；旁白會傳送傳入的要求，以及精簡且經過遮蔽處理的工具摘要。請選擇符合成本與資料處理要求的供應商。
- `imageModel`：接受字串（`"provider/model"`）或物件（`{ primary, fallbacks }`）。
  - 當作用中模型無法接受圖片時，`image` 工具路徑會將其作為視覺模型設定。具備原生視覺能力的模型則會直接接收已載入的圖片位元組。
  - 當所選／預設模型無法接受圖片輸入時，也會用於後援路由。
  - 建議使用明確的 `provider/model` 參照。為了相容性，也接受不含前綴的 ID；若不含前綴的 ID 唯一符合 `models.providers.*.models` 中已設定且支援圖片的項目，OpenClaw 會為其補上該供應商。若符合多個已設定項目，則必須明確加上供應商前綴。
- `mediaModels.image`：接受字串（`"provider/model"`）或物件（`{ primary, fallbacks }`）。
  - 供共用圖片產生功能及任何未來會產生圖片的工具／外掛介面使用。
  - 常見值：用於 Gemini 原生圖片產生的 `google/gemini-3.1-flash-image`、用於 fal 的 `fal/fal-ai/flux/dev`、用於 OpenAI Images 的 `openai/gpt-image-2`，或用於 OpenAI 透明背景 PNG／WebP 輸出的 `openai/gpt-image-1.5`。
  - 若直接選取供應商／模型，也請設定相符的供應商驗證（例如 `google/*` 使用 `GEMINI_API_KEY` 或 `GOOGLE_API_KEY`，`openai/gpt-image-2`／`openai/gpt-image-1.5` 使用 `OPENAI_API_KEY` 或 OpenAI Codex OAuth，`fal/*` 使用 `FAL_KEY`）。
  - 若省略，`image_generate` 仍可推斷具備驗證資訊的供應商預設值。它會先嘗試目前的預設供應商，再依供應商 ID 順序嘗試其餘已註冊的圖片產生供應商。
- `mediaModels.music`：接受字串（`"provider/model"`）或物件（`{ primary, fallbacks }`）。
  - 供共用音樂產生功能及內建的 `music_generate` 工具使用。
  - 常見值：`google/lyria-3-clip-preview`、`google/lyria-3-pro-preview` 或 `minimax/music-2.6`。
  - 若省略，`music_generate` 仍可推斷具備驗證資訊的供應商預設值。它會先嘗試目前的預設供應商，再依供應商 ID 順序嘗試其餘已註冊的音樂產生供應商。
  - 若直接選取供應商／模型，也請設定相符的供應商驗證／API 金鑰。
- `mediaModels.video`：接受字串（`"provider/model"`）或物件（`{ primary, fallbacks }`）。
  - 供共用影片產生功能及內建的 `video_generate` 工具使用。
  - 常見值：`qwen/wan2.6-t2v`、`qwen/wan2.6-i2v`、`qwen/wan2.6-r2v`、`qwen/wan2.6-r2v-flash` 或 `qwen/wan2.7-r2v`。
  - 若省略，`video_generate` 仍可推斷具備驗證資訊的供應商預設值。它會先嘗試目前的預設供應商，再依供應商 ID 順序嘗試其餘已註冊的影片產生供應商。
  - 若直接選取供應商／模型，也請設定相符的供應商驗證／API 金鑰。
  - 官方 Qwen 影片產生外掛最多支援 1 部輸出影片、1 張輸入圖片、4 部輸入影片、10 秒長度，以及供應商層級的 `size`、`aspectRatio`、`resolution`、`audio` 和 `watermark` 選項。
- `pdfModel`：接受字串（`"provider/model"`）或物件（`{ primary, fallbacks }`）。
  - 供 `pdf` 工具用於模型路由。
  - 若省略，PDF 工具會先後援至 `imageModel`，再後援至解析後的工作階段／預設模型。
- `pdfMaxMb`：呼叫時未傳入 `maxBytesMb` 的情況下，`pdf` 工具的預設 PDF 大小上限。
- `pdfMaxPages`：`pdf` 工具在擷取後援模式中考量的預設最大頁數。
- `verboseDefault`：代理程式的預設詳細程度。值：`"off"`、`"on"`、`"full"`。預設值：`"off"`。
- `toolProgressDetail`：`/verbose` 工具摘要及進度草稿工具行的詳細資料模式。值：`"explain"`（預設，精簡的人類可讀標籤）或 `"raw"`（可用時附加原始命令／詳細資料）。每個代理程式的 `agents.entries.*.toolProgressDetail` 會覆寫此預設值。
- `reasoningDefault`：代理程式的預設推理可見性。值：`"off"`、`"on"`、`"stream"`。每個代理程式的 `agents.entries.*.reasoningDefault` 會覆寫此預設值。僅在擁有者、已授權的傳送者或操作員管理員閘道情境下，且未設定每則訊息或工作階段的推理覆寫時，才會套用已設定的推理預設值。
- `elevatedDefault`：代理程式的預設提升輸出層級。值：`"off"`、`"on"`、`"ask"`、`"full"`。預設值：`"on"`。
- `model.primary`：格式為 `provider/model`（例如用於 Codex OAuth 存取的 `openai/gpt-5.6-sol`）。若省略供應商，OpenClaw 會先嘗試別名，接著尋找該確切模型 ID 唯一符合的已設定供應商，最後才後援至已設定的預設供應商（此為已棄用的相容性行為，因此建議明確使用 `provider/model`）。若該供應商已不再提供已設定的預設模型，OpenClaw 會後援至第一個已設定的供應商／模型，而不會顯示已移除供應商的過時預設值。
- `contextTokens`：選用的代理程式全域上限。它可以降低較大型模型的有效預算，但無法將模型提高至超過其已設定或已探索的 `contextTokens`。若要讓某個直接 OpenAI 模型使用其更大的原生視窗，請為該模型設定 `models.providers.openai.models[].contextWindow` 和 `contextTokens`；請參閱 [OpenAI 上下文視窗預設值](/zh-TW/providers/openai#context-window-defaults-and-long-context-opt-in)。
- `models`：已設定的別名及每個模型的設定。每個項目可包含 `alias`（捷徑）和 `params`（供應商專用，例如 `temperature`、`maxTokens`、`cacheRetention`、`context1m`、`responsesServerCompaction`、`responsesCompactThreshold`、OpenRouter `provider` 路由、`chat_template_kwargs`、`extra_body`／`extraBody`）。新增項目不會限制模型覆寫。
  - 使用 `provider/*` 項目（例如 `"openai/*": {}` 或 `"vllm/*": {}`），即可顯示所選供應商探索到的所有模型，而不必手動列出每個模型 ID。
  - 若該供應商所有動態探索到的模型都應使用相同執行階段，請在 `provider/*` 項目中新增 `agentRuntime`。精確的 `provider/model` 執行階段原則仍優先於萬用字元。
  - 安全的中繼資料編輯：使用 `openclaw config set agents.defaults.models '<json>' --strict-json --merge` 新增項目。除非傳入 `--replace`，否則 `config set` 會拒絕可能移除現有項目的取代操作。
- `modelPolicy.allow`：明確的覆寫允許清單。接受別名、精確的 `provider/model` 參照，以及 `openai/*` 或 `clawrouter/anthropic/*` 等尾端前綴萬用字元。省略此項或使用 `[]`，即可允許任何模型。`agents.entries.*.modelPolicy.allow` 會取代該代理程式的預設原則；明確的空清單會讓該代理程式允許任何模型。
  - 限定供應商範圍的設定／初始設定流程，會將所選供應商的模型合併至此對應表，並保留已設定的其他不相關供應商。
  - 對於直接 OpenAI Responses 模型，會自動啟用伺服器端壓縮。使用 `params.responsesServerCompaction: false` 可停止注入 `context_management`，或使用 `params.responsesCompactThreshold` 覆寫閾值。請參閱 [OpenAI 伺服器端壓縮](/zh-TW/providers/openai#advanced-configuration)。
- `params`：套用至所有模型的全域預設供應商參數。在 `agents.defaults.params` 設定（例如 `{ cacheRetention: "long" }`）。
- `params` 合併優先順序（設定）：`agents.defaults.params`（全域基底）會由 `agents.defaults.models["provider/model"].params`（每個模型）覆寫，接著 `agents.entries.*.params`（相符的代理程式 ID）會依鍵覆寫。詳情請參閱[提示快取](/zh-TW/reference/prompt-caching)。
- `models.providers.openrouter.params.provider`：OpenRouter 全域的預設供應商路由原則。OpenClaw 會將此項轉送至 OpenRouter 要求的 `provider` 物件；每個模型的 `agents.defaults.models["openrouter/<model>"].params.provider` 及代理程式參數會依鍵覆寫。請參閱 [OpenRouter 供應商路由](/zh-TW/providers/openrouter#advanced-configuration)。
- `params.extra_body`／`params.extraBody`：進階透傳 JSON，會合併至 OpenAI 相容代理伺服器的 `api: "openai-completions"` 要求本文。若與產生的要求鍵衝突，額外本文優先；非原生 completions 路由之後仍會移除僅限 OpenAI 的 `store`。
- `params.chat_template_kwargs`：vLLM／OpenAI 相容的聊天範本引數，會合併至頂層 `api: "openai-completions"` 要求本文。對於關閉思考的 `vllm/nemotron-3-*`，隨附的 vLLM 外掛會自動傳送 `enable_thinking: false` 和 `force_nonempty_content: true`；明確的 `chat_template_kwargs` 會覆寫產生的預設值，而 `extra_body.chat_template_kwargs` 仍具有最終優先權。已設定的 vLLM Qwen 和 Nemotron 思考模型會提供二元 `/think` 選項（`off`、`on`），而非多層級的投入程度階梯。
- `compat.thinkingFormat`：OpenAI 相容的思考承載資料樣式。Together 樣式的 `reasoning.enabled` 請使用 `"together"`，Qwen 樣式的頂層 `enable_thinking` 請使用 `"qwen"`，而在支援要求層級聊天範本 kwargs 的 Qwen 系列後端（例如 vLLM）上，`chat_template_kwargs.enable_thinking` 請使用 `"qwen-chat-template"`。OpenClaw 會將停用思考對應至 `false`，將啟用思考對應至 `true`；已設定的 vLLM Qwen 模型則會針對這些格式提供二元 `/think` 選項。
- `compat.supportedReasoningEfforts`：各模型的 OpenAI 相容推理強度清單。對於確實接受 `"xhigh"` 的自訂端點，請將其納入；之後 OpenClaw 會在命令選單、閘道工作階段資料列、工作階段修補驗證、代理程式命令列介面驗證，以及該已設定供應商／模型的 `llm-task` 驗證中公開 `/think xhigh`。當後端需要使用供應商特定值來表示標準層級時，請使用 `compat.reasoningEffortMap`。
- `params.preserveThinking`：僅限 Z.AI 的保留思考內容選用設定。啟用且思考功能開啟時，OpenClaw 會傳送 `thinking.clear_thinking: false`，並重播先前的 `reasoning_content`；請參閱 [Z.AI 思考與保留思考內容](/zh-TW/providers/zai#advanced-configuration)。
- `localService`：供本機／自行託管模型伺服器使用的選用供應商層級程序管理器。當所選模型屬於該供應商時，OpenClaw 會探測 `healthUrl`（或 `baseUrl + "/models"`）；若端點無法使用，便以 `args` 啟動 `command`，等待最多 `readyTimeoutMs`，然後傳送模型請求。`command` 必須是絕對路徑。`idleStopMs: 0` 會讓程序持續執行，直到 OpenClaw 結束；正值則會在經過該毫秒數的閒置時間後，停止由 OpenClaw 啟動的程序。請參閱[本機模型服務](/zh-TW/gateway/local-model-services)。
- 執行階段原則應設定於供應商或模型，而不是 `agents.defaults`。供應商通用規則請使用 `models.providers.<provider>.agentRuntime`，模型特定規則則請使用 `agents.defaults.models["provider/model"].agentRuntime`／`agents.entries.*.models["provider/model"].agentRuntime`。僅有供應商／模型前綴絕不會選取執行框架。當 runtime 未設定或為 `auto` 時，只有在路由恰好是官方 HTTPS Platform Responses 或 ChatGPT Responses，且沒有自行設定的請求覆寫時，OpenAI 才可能隱含選取 Codex。請參閱 [OpenAI 隱含代理程式執行階段](/zh-TW/providers/openai#implicit-agent-runtime)。
- 會變更這些欄位的設定寫入工具（例如 `/models set`、`/models set-image`，以及新增／移除備援模型的命令）會儲存標準物件形式，並盡可能保留現有的備援清單。
- `maxConcurrent`：跨工作階段同時執行的代理程式數量上限（每個工作階段內仍會依序執行）。預設值：`4`。

### 執行階段政策

```json5
{
  models: {
    providers: {
      openai: {
        agentRuntime: { id: "codex" },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
        "vllm/*": {
          agentRuntime: { id: "openclaw" },
        },
      },
    },
  },
}
```

- `id`：`"auto"`、`"openclaw"`、已註冊的外掛執行框架 ID，或支援的命令列介面後端別名。隨附的 Codex 外掛會註冊 `codex`；隨附的 Anthropic 外掛則提供 `claude-cli` 命令列介面後端。
- `id: "auto"` 允許已註冊的外掛執行框架接管宣告或以其他方式符合其支援契約的有效路由，且沒有任何執行框架相符時使用 OpenClaw。明確指定的外掛執行階段（例如 `id: "codex"`）會要求該執行框架及相容的有效路由；任一者無法使用或執行失敗時，皆會採取封閉式失敗。
- `id: "pi"` 僅作為 `openclaw` 的已淘汰別名接受，以保留 v2026.5.22 及更早版本已發布的設定。新設定應使用 `openclaw`。
- 執行階段優先順序依序為精確模型政策（`agents.entries.*.models["provider/model"]`、`agents.defaults.models["provider/model"]` 或 `models.providers.<provider>.models[]`）、`agents.entries.*` / `agents.defaults.models["provider/*"]`，最後是 `models.providers.<provider>.agentRuntime` 的整個供應商政策。
- 整體代理程式執行階段鍵屬於舊版。執行階段選擇會忽略 `agents.defaults.agentRuntime`、`agents.entries.*.agentRuntime`、工作階段執行階段固定值及 `OPENCLAW_AGENT_RUNTIME`。請執行 `openclaw doctor --fix` 移除過時值。
- 符合資格、精確且未編寫要求覆寫的官方 HTTPS OpenAI Responses/ChatGPT 路由，可隱含使用 Codex 執行框架。供應商／模型 `agentRuntime.id: "codex"` 會將 Codex 設為封閉式失敗要求，但不會使不相容的路由變得相容。
- 對於 Claude 命令列介面部署，建議使用 `model: "anthropic/claude-opus-5"` 搭配模型範圍的 `agentRuntime.id: "claude-cli"`。舊版 `claude-cli/<model>` 參照仍可基於相容性運作，但新設定應維持供應商／模型選擇的標準形式，並將執行後端置於供應商／模型執行階段政策中。
- 這只會控制文字代理程式回合的執行。媒體生成、視覺、PDF、音樂、影片和 TTS 仍使用各自的供應商／模型設定。

**內建別名簡寫**（僅在模型位於 `agents.defaults.models` 時套用）：

| 別名                | 模型                            |
| ------------------- | ------------------------------- |
| `opus`              | `anthropic/claude-opus-5`       |
| `sonnet`            | `anthropic/claude-sonnet-5`     |
| `gpt`               | `openai/gpt-5.4`                |
| `gpt-mini`          | `openai/gpt-5.4-mini`           |
| `gpt-nano`          | `openai/gpt-5.4-nano`           |
| `gemini`            | `google/gemini-3.1-pro-preview` |
| `gemini-flash`      | `google/gemini-3-flash-preview` |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite`  |

你設定的別名一律優先於預設值。

除非設定 `--thinking off` 或自行定義 `agents.defaults.models["zai/<model>"].params.thinking`，否則 Z.AI GLM-4.x 模型會自動啟用思考模式。
Z.AI 模型預設啟用 `tool_stream`，以串流傳輸工具呼叫。將 `agents.defaults.models["zai/<model>"].params.tool_stream` 設為 `false` 即可停用。
Anthropic Claude Opus 4.8 在 OpenClaw 中預設關閉思考；明確啟用調適型思考時，Anthropic 供應商所擁有的努力程度預設值為 `high`。若未設定明確的思考層級，Claude 4.6 模型預設為 `adaptive`。

### 命令列介面後端選擇

命令列介面配接器機制由外掛註冊，而非在代理程式
預設值下設定。如上所示，使用模型範圍的 `agentRuntime.id`
選擇已註冊的命令列介面後端。操作方式請參閱[命令列介面後端](/zh-TW/gateway/cli-backends)，
命令、工作階段、影像和剖析器註冊方式則請參閱[建置命令列介面後端外掛](/zh-TW/plugins/cli-backend-plugins)。

### `agents.defaults.promptOverlays`

依模型系列套用至 OpenClaw 所組合提示詞介面的供應商無關提示詞覆疊。GPT-5 系列模型 ID 在 OpenClaw／供應商路由間接收共用行為契約；`personality` 僅控制友善的互動風格層。原生 Codex 應用程式伺服器路由會保留 Codex 所擁有的基礎／模型指示，而非使用此 OpenClaw GPT-5 覆疊，且 OpenClaw 會為原生討論串停用 Codex 的內建個性。

```json5
{
  agents: {
    defaults: {
      promptOverlays: {
        gpt5: {
          personality: "friendly", // friendly | on | off
        },
      },
    },
  },
}
```

- `"friendly"`（預設）和 `"on"` 會啟用友善的互動風格層。
- `"off"` 僅停用友善層；帶標記的 GPT-5 行為契約仍會啟用。
- 未設定此共用設定時，仍會讀取舊版 `plugins.entries.openai.config.personality`。

### `agents.defaults.heartbeat`

定期執行心跳偵測。

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m disables
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        includeSystemPromptSection: true, // default: true; false omits the Heartbeat section from the system prompt
        lightContext: false, // default: false; true skips workspace bootstrap files for heartbeat runs
        isolatedSession: false, // default: false; true runs each heartbeat in a fresh session (no conversation history)
        skipWhenBusy: false, // default: false; true also waits for this agent's subagent/nested lanes
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (default) | block
        target: "none", // default: none | options: last | whatsapp | telegram | discord | ...
        prompt: "Follow the heartbeat monitor scratch context...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
        timeoutSeconds: 45,
      },
    },
  },
}
```

- `every`：持續時間字串（ms/s/m/h）。預設值：`30m`（API 金鑰驗證）或 `1h`（OAuth 驗證）。設為 `0m` 即可停用。
- 執行頻率會寫入系統所擁有的排程監控資料列。執行 `openclaw doctor --fix` 以具現化遺失或過時的資料列。如果排程已停用，排定的心跳偵測不會執行，且閘道會記錄啟動警告。
- `includeSystemPromptSection`：設為 false 時，從系統提示詞中省略心跳偵測區段。預設值：`true`。
- `suppressToolErrorWarnings`：設為 true 時，在心跳偵測執行期間抑制工具錯誤警告承載資料。
- `timeoutSeconds`：心跳偵測代理程式回合在中止前允許的最長秒數。保持未設定時，若已設定 `agents.defaults.timeoutSeconds`，則使用該值；否則使用心跳偵測頻率，且上限為 600 秒。
- `directPolicy`：直接／私訊傳遞政策。`allow`（預設）允許直接目標傳遞。`block` 會抑制直接目標傳遞並發出 `reason=dm-blocked`。
- `lightContext`：設為 true 時，心跳偵測執行會使用輕量啟動內容並略過工作區啟動檔案。無論如何，心跳偵測執行器都會注入監控暫存內容。
- `isolatedSession`：設為 true 時，每次心跳偵測都會在沒有先前對話歷程的新工作階段中執行。隔離模式與排程 `sessionTarget: "isolated"` 相同。將每次心跳偵測的權杖成本從約 100K 降至約 2-5K 個權杖。
- `skipWhenBusy`：設為 true 時，心跳偵測執行會在該代理程式的額外忙碌通道上延後：其自身以工作階段鍵控的子代理程式或巢狀命令工作。即使沒有此旗標，排程通道一律會延後心跳偵測。
- 個別代理程式：設定 `agents.entries.*.heartbeat`。只要有任何代理程式定義 `heartbeat`，就**只有那些代理程式**會執行心跳偵測。
- 心跳偵測會執行完整的代理程式回合——間隔越短，消耗的權杖越多。

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // default | safeguard
        provider: "my-provider", // id of a registered compaction provider plugin (optional)
        thinkingLevel: "low", // optional compaction-only thinking override
        timeoutSeconds: 180,
        keepRecentTokens: 50000,
        recentTurnsPreserve: 3,
        identifierPolicy: "strict", // strict | off
        qualityGuard: { enabled: true, maxRetries: 1 },
        midTurnPrecheck: { enabled: false }, // optional tool-loop pressure check
        postIndexSync: "async", // off | async | await
        postCompactionSections: ["Session Startup", "Red Lines"],
        model: "openrouter/anthropic/claude-sonnet-4-6", // optional compaction-only model override
        truncateAfterCompaction: true, // rotate to a smaller successor JSONL after compaction
        maxActiveTranscriptBytes: "20mb", // optional preflight local compaction trigger
        notifyUser: true, // notices when compaction starts/completes and on memory-flush degradation (default: false)
        memoryFlush: {
          enabled: true,
          model: "ollama/qwen3:8b", // optional memory-flush-only model override
          softThresholdTokens: 6000,
          forceFlushTranscriptBytes: "2mb",
        },
      },
    },
  },
}
```

- `mode`: `default` 或 `safeguard`（針對長篇歷史記錄進行分塊摘要）。請參閱[壓縮](/zh-TW/concepts/compaction)。
- `provider`: 已註冊壓縮提供者外掛的 ID。設定後，會呼叫提供者的 `summarize()`，而非使用內建的 LLM 摘要。失敗時會回復使用內建功能。設定提供者會強制使用 `mode: "safeguard"`。請參閱[壓縮](/zh-TW/concepts/compaction)。
- `thinkingLevel`: 選用的思考層級，僅用於內嵌 OpenClaw 壓縮摘要（`off`、`minimal`、`low`、`medium`、`high`、`xhigh`、`adaptive`、`max` 或 `ultra`）。它會覆寫工作階段目前的思考層級，並限制在所選壓縮模型／執行階段支援的範圍內。保持未設定即可繼承工作階段層級。原生 Codex app-server 壓縮會忽略此設定，因為原生壓縮要求沒有個別操作的思考覆寫；若已設定，OpenClaw 會記錄警告。
- `timeoutSeconds`: OpenClaw 中止單次壓縮操作前允許的最長秒數。預設值：`180`。
- `keepRecentTokens`: 保留最新逐字對話記錄尾端的代理程式截斷點預算。明確設定時，手動 `/compact` 會遵循此值；否則手動壓縮是硬性檢查點。
- `recentTurnsPreserve`: 在防護摘要之外逐字保留的最新使用者／助理對話輪數。預設值：`3`。
- `identifierPolicy`: `strict`（預設）或 `off`。`strict` 會在壓縮摘要期間前置內建的不透明識別碼保留指引。
- `qualityGuard`: 防護摘要針對格式錯誤輸出的重試檢查。在防護模式中預設啟用；設為 `enabled: false` 可略過稽核。
- `midTurnPrecheck`: 選用的工具迴圈壓力檢查。當設為 `enabled: true` 時，OpenClaw 會在附加工具結果之後、下次呼叫模型之前檢查上下文壓力。如果上下文已無法容納，它會在提交提示詞前中止目前嘗試，並重用現有的預先檢查復原路徑，以截斷工具結果或進行壓縮後重試。適用於 `default` 和 `safeguard` 壓縮模式。預設：停用。
- `postIndexSync`: 壓縮後的工作階段記憶重新索引模式。預設值：`"async"`。使用 `"await"` 可取得最高即時性，使用 `"async"` 可降低壓縮延遲，只有在工作階段記憶同步由其他位置處理時才使用 `"off"`。
- `postCompactionSections`: 壓縮後要重新注入的選用 AGENTS.md H2/H3 區段名稱。保持未設定或使用 `[]` 即可停用。
- `model`: 僅供壓縮摘要使用的選用 `provider/model-id` 或來自 `agents.defaults.models` 的純別名。純別名會在分派前解析；發生衝突時，已設定的字面模型 ID 優先。當主要工作階段應維持使用某個模型，但壓縮摘要應在另一個模型上執行時，請使用此設定；未設定時，壓縮會使用工作階段的主要模型。
- `truncateAfterCompaction`: 壓縮後輪替使用中的工作階段對話記錄，使後續對話輪次僅載入摘要與未摘要的尾端，同時封存先前的完整對話記錄。避免長時間執行的工作階段中，使用中的對話記錄無限制增長。預設值：`false`。
- `maxActiveTranscriptBytes`: 選用的位元組門檻值（`number` 或類似 `"20mb"` 的字串）；當對話記錄歷史超過門檻時，會在執行前觸發一般本機壓縮。需要 `truncateAfterCompaction`，成功壓縮後才能輪替至較小的後繼對話記錄。未設定或為 `0` 時停用。
- `notifyUser`: 設為 `true` 時，會向使用者傳送簡短的上下文維護通知：壓縮開始與完成時（例如「正在壓縮上下文……」和「壓縮完成」），以及壓縮前記憶清除已用盡、因此回覆以降級狀態繼續時（例如「記憶維護暫時失敗；繼續你的回覆。」）。預設停用，以保持這些通知靜默。
- `memoryFlush`: 自動壓縮前的靜默代理式對話輪次，用於儲存持久記憶。當此維護對話輪次應持續使用本機模型時，請將 `model` 設為確切的提供者／模型，例如 `ollama/qwen3:8b`；此覆寫不會繼承使用中工作階段的備援鏈。即使權杖計數器已過時，`forceFlushTranscriptBytes` 仍會在對話記錄大小達到門檻時強制清除。工作區唯讀時會略過。

自訂壓縮指示由程式碼擁有。請實作具有 `summarize()` 的壓縮提供者
外掛，以自訂摘要建構；若壓縮後的上下文必須注入後續
模型提示詞，請使用 `before_prompt_build`。Doctor 會移除已淘汰的指示欄位，並指向這些
接合面。

### `agents.defaults.contextPruning`

在傳送至 LLM 前，從記憶體內上下文中修剪**舊工具結果**。**不會**修改磁碟上的工作階段歷史記錄。預設停用；設定 `mode: "cache-ttl"` 即可啟用。

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // off（預設）| cache-ttl
      },
    },
  },
}
```

<Accordion title="cache-ttl 模式行為">

- `mode: "cache-ttl"` 會啟用修剪階段。
- 修剪會先對過大的工具結果進行軟修剪，然後視需要硬性清除較舊的工具結果。

**軟修剪**會保留開頭與結尾，並在中間插入 `...`。

**硬性清除**會以預留位置取代整個工具結果。

注意事項：

- 影像區塊絕不會被修剪／清除。
- 比例以字元為基礎（近似值），並非精確的權杖數。
- 會保留最新的助理訊息。

</Accordion>

如需行為詳細資訊，請參閱[工作階段修剪](/zh-TW/concepts/session-pruning)。

### 區塊串流

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200, breakPreference: "paragraph" },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off（預設）| natural | custom（使用 minMs/maxMs）
    },
  },
}
```

- 非 Telegram 頻道需要明確設定 `*.streaming.block.enabled: true`，才能啟用區塊回覆。QQ Bot 是例外：它沒有 `streaming.block` 鍵，且除非 `channels.qqbot.streaming.mode` 為 `"off"`，否則會串流區塊回覆。
- 頻道覆寫：`channels.<channel>.streaming.block.coalesce`（以及各帳號的變體）。Discord、Google Chat、Mattermost、MS Teams、Signal 和 Slack 預設為 `minChars: 1500` / `idleMs: 1000`。
- `blockStreamingChunk.breakPreference`: 偏好的區塊邊界（`"paragraph" | "newline" | "sentence"`）。
- `humanDelay`: 區塊回覆之間的隨機暫停。預設值：`off`。`natural` = 800-2500ms。`custom` 使用 `minMs`/`maxMs`（任何未設定的界限都會回復使用自然範圍）。各代理程式覆寫：`agents.entries.*.humanDelay`。

如需行為與分塊詳細資訊，請參閱[串流](/zh-TW/concepts/streaming)。

### 輸入中指示器

```json5
{
  agents: {
    defaults: {
      typingMode: "instant", // never | instant | thinking | message
      typingIntervalSeconds: 6,
    },
  },
}
```

- 預設值：直接聊天／提及為 `instant`，未提及的群組聊天為 `message`。
- `typingIntervalSeconds` 預設值：`6`。
- 各代理程式覆寫：`agents.entries.*.typingMode`。

請參閱[輸入中指示器](/zh-TW/concepts/typing-indicators)。

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

內嵌代理程式的選用沙箱功能。完整指南請參閱[沙箱](/zh-TW/gateway/sandboxing)。

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off（預設）| non-main | all
        backend: "docker", // docker（預設）| ssh | openshell
        scope: "agent", // session | agent（預設）| shared
        workspaceAccess: "none", // none（預設）| ro | rw
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          containerPrefix: "openclaw-sbx-",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git curl jq",
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          gpus: "all",
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
          dns: ["1.1.1.1", "8.8.8.8"],
          extraHosts: ["internal.service:10.0.0.5"],
          binds: ["/home/user/source:/source:rw"],
        },
        ssh: {
          target: "user@gateway-host:22",
          command: "ssh",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // 也支援 SecretRefs／行內內容：
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
        browser: {
          enabled: false,
          image: "openclaw-sandbox-browser:bookworm-slim",
          network: "openclaw-sandbox-browser",
          cdpPort: 9222,
          cdpSourceRange: "172.21.0.1/32",
          vncPort: 5900,
          noVncPort: 6080,
          headless: false,
          enableNoVnc: true,
          allowHostControl: false,
          autoStart: true,
          autoStartTimeoutMs: 12000,
        },
        prune: {
          idleHours: 24,
          maxAgeDays: 7,
        },
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        allow: [
          "exec",
          "process",
          "read",
          "write",
          "edit",
          "apply_patch",
          "sessions_list",
          "sessions_history",
          "sessions_send",
          "sessions_spawn",
          "session_status",
        ],
        deny: ["browser", "canvas", "nodes", "cron", "discord", "gateway"],
      },
    },
  },
}
```

以上顯示的預設值（`off`/`docker`/`agent`/`none`/`bookworm-slim` 映像檔/`none` 網路等）是 OpenClaw 的實際預設值，不只是示意值。

<Accordion title="沙箱詳細資訊">

**後端：**

- `docker`: 本機 Docker 執行階段（預設）
- `ssh`: 通用的 SSH 後端遠端執行階段
- `openshell`: OpenShell 執行階段

選取 `backend: "openshell"` 時，執行階段專屬設定會移至
`plugins.entries.openshell.config`。

**SSH 後端設定：**

- `target`：採用 `user@host[:port]` 格式的 SSH 目標
- `command`：SSH 用戶端命令（預設：`ssh`）
- `workspaceRoot`：用於各範圍工作區的絕對遠端根目錄（預設：`/tmp/openclaw-sandboxes`）
- `identityFile` / `certificateFile` / `knownHostsFile`：傳遞給 OpenSSH 的現有本機檔案
- `identityData` / `certificateData` / `knownHostsData`：OpenClaw 在執行階段具現化為暫存檔案的內嵌內容或 SecretRef
- `strictHostKeyChecking` / `updateHostKeys`：OpenSSH 主機金鑰原則控制項（兩者預設皆為 `true`）

**SSH 驗證優先順序：**

- `identityData` 優先於 `identityFile`
- `certificateData` 優先於 `certificateFile`
- `knownHostsData` 優先於 `knownHostsFile`
- 以 SecretRef 為後端的 `*Data` 值會在沙箱工作階段啟動前，從作用中的密鑰執行階段快照解析

**SSH 後端行為：**

- 在建立或重新建立後，僅植入遠端工作區一次
- 之後以遠端 SSH 工作區為標準來源
- 透過 SSH 路由 `exec`、檔案工具和媒體路徑
- 不會自動將遠端變更同步回主機
- 不支援沙箱瀏覽器容器

**工作區存取：**

- `none`：位於 `~/.openclaw/sandboxes` 下的各範圍沙箱工作區（預設）
- `ro`：沙箱工作區位於 `/workspace`，代理程式工作區以唯讀方式掛載於 `/agent`
- `rw`：代理程式工作區以讀寫方式掛載於 `/workspace`

**範圍：**

- `session`：每個工作階段各有一個容器和工作區
- `agent`：每個代理程式各有一個容器和工作區（預設）
- `shared`：共用容器和工作區（工作階段之間不隔離）

**OpenShell 外掛設定：**

```json5
{
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          mode: "mirror", // 鏡像（預設）| 遠端
          command: "openshell",
          from: "openclaw",
          remoteWorkspaceDir: "/sandbox",
          remoteAgentWorkspaceDir: "/agent",
          gateway: "lab", // 選用
          gatewayEndpoint: "https://lab.example", // 選用
          policy: "strict", // 選用的 OpenShell 原則 ID
          providers: ["openai"], // 選用
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**OpenShell 模式：**

- `mirror`：執行前從本機植入遠端，執行後同步回本機；本機工作區維持為標準來源
- `remote`：建立沙箱時僅植入遠端一次，之後以遠端工作區為標準來源

在 `remote` 模式下，植入步驟完成後，在 OpenClaw 外部進行的主機本機編輯不會自動同步至沙箱。
傳輸方式是透過 SSH 進入 OpenShell 沙箱，但沙箱生命週期和選用的鏡像同步由此外掛負責。

**`setupCommand`** 會在容器建立後執行一次（透過 `sh -lc`）。需要網路輸出、可寫入的根目錄及 root 使用者。

**容器預設為 `network: "none"`**——若代理程式需要對外存取，請設為 `"bridge"`（或自訂橋接網路）。
`"host"` 會遭到封鎖。除非明確設定
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`（緊急解鎖），否則 `"container:<id>"` 預設會遭到封鎖。
作用中 OpenClaw 沙箱內的 Codex app-server 回合，會使用相同的輸出設定進行其原生程式碼模式網路存取。

**傳入附件**會暫存至作用中工作區的 `media/inbound/*`。

**`docker.binds`** 會掛載其他主機目錄；全域和各代理程式的繫結會合併。

**沙箱瀏覽器**（`sandbox.browser.enabled`，預設為 `false`）：容器中的 Chromium + CDP。noVNC URL 會注入系統提示詞。在 `openclaw.json` 中不需要 `browser.enabled`。
noVNC 觀察者存取預設使用 VNC 驗證，且 OpenClaw 會產生短效權杖 URL（而非在共用 URL 中公開密碼）。

- `allowHostControl: false`（預設）會阻止沙箱工作階段以主機瀏覽器為目標。
- `network` 預設為 `openclaw-sandbox-browser`（專用橋接網路）。只有在明確需要全域橋接連線時，才設為 `bridge`。`"host"` 在此處也會遭到封鎖。
- `cdpSourceRange` 可選擇在容器邊界將 CDP 傳入流量限制於某個 CIDR 範圍（例如 `172.21.0.1/32`）。
- `sandbox.browser.binds` 僅將其他主機目錄掛載至沙箱瀏覽器容器。設定後（包括 `[]`），它會取代瀏覽器容器的 `docker.binds`。
- 沙箱瀏覽器容器的 Chromium 一律以 `--no-sandbox --disable-setuid-sandbox` 啟動（容器不具備 Chrome 自身沙箱所需的核心原語）；沒有可切換此行為的設定。
- 啟動預設值定義於 `scripts/sandbox-browser-entrypoint.sh`，並針對容器主機調校：
  - `--remote-debugging-address=127.0.0.1`
  - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
  - `--user-data-dir=${HOME}/.chrome`
  - `--no-first-run`
  - `--no-default-browser-check`
  - `--disable-dev-shm-usage`
  - `--disable-background-networking`
  - `--disable-breakpad`
  - `--disable-crash-reporter`
  - `--no-zygote`
  - `--metrics-recording-only`
  - `--password-store=basic`
  - `--use-mock-keychain`
  - `--disable-3d-apis`、`--disable-gpu` 和 `--disable-software-rasterizer`
    預設啟用；若使用 WebGL/3D 時需要，可透過
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` 停用。
  - `--disable-extensions`（預設啟用）；若工作流程依賴擴充功能，
    `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` 可重新啟用擴充功能。
  - 預設為 `--renderer-process-limit=2`；可透過
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>` 變更，將 `0` 設為使用 Chromium 的
    預設處理程序上限。
  - 僅在啟用 `headless` 時使用 `--headless=new`。
  - 這些預設值是容器映像基準；若要變更容器預設值，請使用具備自訂
    進入點的自訂瀏覽器映像。

</Accordion>

瀏覽器沙箱和 `sandbox.docker.binds` 僅支援 Docker。

建置映像（從原始碼簽出）：

```bash
scripts/sandbox-setup.sh           # 主要沙箱映像
scripts/sandbox-browser-setup.sh   # 選用的瀏覽器映像
```

若使用不含原始碼簽出的 npm 安裝，請參閱[沙箱 § 映像與設定](/zh-TW/gateway/sandboxing#images-and-setup)，以取得內嵌的 `docker build` 命令。

### `agents.entries`（各代理程式覆寫）

使用 `agents.entries.*.tts` 可讓代理程式擁有自己的 TTS 提供者、語音、模型、
風格或自動 TTS 模式。代理程式區塊會深度合併至全域
`tts` 之上，因此共用認證資訊可集中存放，而個別
代理程式只需覆寫其所需的語音或提供者欄位。作用中代理程式的
覆寫會套用至自動語音回覆、`/tts audio`、`/tts status` 和
`tts` 代理程式工具。如需提供者範例和優先順序，請參閱[文字轉語音](/zh-TW/tools/tts#per-agent-voice-overrides)。

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        name: "Main Agent",
        workspace: "~/.openclaw/workspace",
        agentDir: "~/.openclaw/agents/main/agent",
        model: "anthropic/claude-opus-4-6", // 或 { primary, fallbacks }
        utilityModel: "openai/gpt-5.4-mini",
        thinkingDefault: "high", // 各代理程式的思考層級覆寫
        reasoningDefault: "on", // 各代理程式的推理可見性覆寫
        fastModeDefault: false, // 各代理程式的快速模式覆寫
        params: { cacheRetention: "none" }, // 依鍵覆寫相符的 defaults.models 參數
        tts: {
          providers: {
            elevenlabs: { speakerVoiceId: "EXAVITQu4vr4xnSDxMaL" },
          },
        },
        skills: ["docs-search"], // 設定後取代 agents.defaults.skills
        identity: {
          name: "Samantha",
          theme: "helpful sloth",
          emoji: "🦥",
          avatar: "avatars/samantha.png",
        },
        groupChat: { mentionPatterns: ["@openclaw"] },
        sandbox: { mode: "off" },
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent", // persistent | oneshot
            cwd: "/workspace/openclaw",
          },
        },
        subagents: { allowAgents: ["*"] },
        tools: {
          profile: "coding",
          allow: ["browser"],
          deny: ["canvas"],
          elevated: { enabled: true },
        },
      },
    ],
  },
}
```

- `id`：穩定的代理程式 ID（必填）。
- `default`：設定多個時，第一個優先（會記錄警告）。若未設定，清單中的第一個項目為預設值。
- `model`：字串形式會設定嚴格的各代理程式主要模型，且不使用模型備援；物件形式的 `{ primary }` 也同樣嚴格，除非加入 `fallbacks`。使用 `{ primary, fallbacks: [...] }` 讓該代理程式選擇啟用備援，或使用 `{ primary, fallbacks: [] }` 明確指定嚴格行為。僅覆寫 `primary` 的排程工作仍會繼承預設備援，除非設定 `fallbacks: []`。
- `utilityModel`：選用的各代理程式覆寫，用於產生工作階段和討論串標題等短期內部工作。依序備援至 `agents.defaults.utilityModel`，再備援至有效工作階段提供者宣告的小型模型預設值。儀表板標題會使用有效的一般工作階段模型重試一次。空字串會略過此代理程式的替代公用工具路徑，但不會停用儀表板標題產生功能。
- `params`：各代理程式的串流參數，會合併並覆寫 `agents.defaults.models` 中選取的模型項目。使用此設定可指定 `cacheRetention`、`temperature` 或 `maxTokens` 等代理程式專屬覆寫，而不必複製整個模型目錄。
- `tts`：選用的各代理程式文字轉語音覆寫。此區塊會深度合併並覆寫 `tts`，因此請將共用的提供者認證資訊和備援政策保留在 `tts` 中，並僅在此設定角色專屬值，例如提供者、語音、模型、風格或自動模式。
- `skills`：選用的各代理程式 Skill 允許清單。若省略，代理程式會在已設定時繼承 `agents.defaults.skills`；明確指定的清單會取代預設值而非與其合併，而 `[]` 表示不使用任何 Skill。
- `thinkingDefault`：選用的各代理程式預設思考層級（`off | minimal | low | medium | high | xhigh | adaptive | max`）。未設定各訊息或工作階段覆寫時，會覆寫此代理程式的 `agents.defaults.thinkingDefault`。所選的提供者／模型設定檔會控制哪些值有效；對 Google Gemini 而言，`adaptive` 會保留由提供者管理的動態思考（Gemini 3/3.1 會省略 `thinkingLevel`，Gemini 2.5 則使用 `thinkingBudget: -1`）。
- `reasoningDefault`：選用的各代理程式預設推理可見性（`on | off | stream`）。未設定各訊息或工作階段推理覆寫時，會覆寫此代理程式的 `agents.defaults.reasoningDefault`。
- `fastModeDefault`：選用的各代理程式快速模式預設值（`"auto" | true | false`）。未設定各訊息或工作階段快速模式覆寫時套用。
- `models`：選用的各代理程式模型目錄／執行階段覆寫，以完整的 `provider/model` ID 作為索引鍵。使用 `models["provider/model"].agentRuntime` 設定各代理程式的執行階段例外。
- `runtime`：選用的各代理程式執行階段描述元。當代理程式應預設使用 ACP 測試框架工作階段時，請使用 `type: "acp"` 搭配 `runtime.acp` 預設值（`agent`、`backend`、`mode`、`cwd`）。
- `identity.avatar`：工作區相對路徑、`http(s)` URL 或 `data:` URI。
- 本機工作區相對的 `identity.avatar` 圖片檔案上限為 2 MB。`http(s)` URL 和 `data:` URI 不受本機檔案大小限制檢查。
- `identity` 會衍生預設值：`ackReaction` 衍生自 `emoji`，`mentionPatterns` 衍生自 `name`/`emoji`。
- `subagents.allowAgents`：為明確的 `sessions_spawn.agentId` 目標所設定的代理程式 ID 允許清單（`["*"]` = 任意已設定的目標；預設：僅限同一代理程式）。若應允許以自身為目標的 `agentId` 呼叫，請納入請求者 ID。代理程式設定已刪除的過時項目會遭 `sessions_spawn` 拒絕，且不會出現在 `agents_list` 中；執行 `openclaw doctor --fix` 以清理這些項目，若該目標應在繼承預設值的同時仍可被產生，則新增最基本的 `agents.entries.*` 項目。
- 沙箱繼承防護：若請求者工作階段位於沙箱中，`sessions_spawn` 會拒絕將在未使用沙箱的情況下執行的目標。
- `subagents.requireAgentId`：設為 true 時，阻擋省略 `agentId` 的 `sessions_spawn` 呼叫（強制明確選取設定檔；預設：false）。
- `subagents.maxConcurrent`：子代理程式執行期間可同時執行的子代理程式數量上限。預設值：`8`。
- `subagents.maxChildrenPerAgent`：單一代理程式工作階段可產生的作用中子代理程式數量上限。預設值：`5`。
- `subagents.maxSpawnDepth`：產生子代理程式的最大巢狀深度（`1`-`5`）。預設值：`1`（不允許巢狀）。
- `subagents.archiveAfterMinutes`：已完成的子代理程式狀態在封存前的保留時間。預設值：`60`。

---

## 多代理程式路由

在單一閘道內執行多個隔離的代理程式。請參閱[多代理程式](/zh-TW/concepts/multi-agent)。

```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
    ],
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
  ],
}
```

### 繫結比對欄位

- `type`（選用）：`route` 用於一般路由（缺少類型時預設為路由），`acp` 用於持久性 ACP 對話繫結。
- `match.channel`（必填）
- `match.accountId`（選用；`*` = 任意帳號；省略 = 預設帳號）
- `match.peer`（選用；`{ kind: direct|group|channel, id }`）
- `match.guildId` / `match.teamId`（選用；依頻道而異）
- `acp`（選用；僅限 `type: "acp"`）：`{ mode, label, cwd, backend }`

**確定性比對順序：**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId`（精確比對，不含對等方／公會／團隊）
5. `match.accountId: "*"`（頻道範圍）
6. 預設代理程式

在每個層級中，第一個相符的 `bindings` 項目優先。

對於 `type: "acp"` 項目，OpenClaw 會依精確的對話身分（`match.channel` + 帳號 + `match.peer.id`）解析，不使用上述路由繫結層級順序。

### 各代理程式存取設定檔

<Accordion title="完整存取權（無沙箱）">

```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="唯讀工具 + 工作區">

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "ro" },
        tools: {
          allow: [
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="無檔案系統存取權（僅限訊息傳遞）">

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "none" },
        tools: {
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "whatsapp",
            "telegram",
            "slack",
            "discord",
            "gateway",
          ],
          deny: [
            "read",
            "write",
            "edit",
            "apply_patch",
            "exec",
            "process",
            "browser",
            "canvas",
            "nodes",
            "cron",
            "gateway",
            "image",
          ],
        },
      },
    ],
  },
}
```

</Accordion>

如需優先順序的詳細資訊，請參閱[多代理程式沙箱與工具](/zh-TW/tools/multi-agent-sandbox-tools)。

---

## 工作階段

```json5
{
  session: {
    scope: "per-sender",
    dmScope: "main", // main | per-peer | per-channel-peer | per-account-channel-peer
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"],
    },
    reset: {
      mode: "daily", // daily | idle
      atHour: 4,
      idleMinutes: 60,
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      direct: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 },
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 30 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    maintenance: {
      mode: "enforce", // enforce（預設）| warn
      pruneAfter: "30d",
      maxEntries: 500,
      resetArchiveRetention: "30d", // 持續時間或 false
      maxDiskBytes: "500mb", // 選用的硬性預算
      highWaterBytes: "400mb", // 選用的清理目標
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // 預設閒置自動取消聚焦時數（`0` 表示停用）
      maxAgeHours: 0, // 預設硬性最長時數（`0` 表示停用）
    },
    sharing: {
      readOnly: true,
      suggest: true,
      drafts: true,
    },
    mainKey: "main", // 舊版（執行階段一律使用 "main"）
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="工作階段欄位詳細資訊">

- **`scope`**：群組聊天情境的基礎工作階段分組策略。
  - `per-sender`（預設）：每位傳送者在頻道情境中都有隔離的工作階段。
  - `global`：頻道情境中的所有參與者共用單一工作階段（僅在需要共用情境時使用）。
- **`dmScope`**：私訊的分組方式。
  - `main`：所有私訊共用主要工作階段。
  - `per-peer`：跨頻道依傳送者 ID 隔離。
  - `per-channel-peer`：依頻道 + 傳送者隔離（建議用於多使用者收件匣）。
  - `per-account-channel-peer`：依帳號 + 頻道 + 傳送者隔離（建議用於多帳號）。
- **`identityLinks`**：將標準 ID 對應至帶有提供者前綴的對等方，以便跨頻道共用工作階段。`/dock_discord` 等停駐命令會使用相同的對應，將作用中工作階段的回覆路由切換至另一個已連結的頻道對等方；請參閱[頻道停駐](/zh-TW/concepts/channel-docking)。
- **`reset`**：主要重設原則。`none` 會停用自動重設，且為預設值；改由壓縮限制作用中情境。`daily` 會在當地時間 `atHour` 重設；`idle` 會在 `idleMinutes` 後重設。若兩者皆有設定，則以先到期者為準。`/new` 和 `/reset` 在每種模式下仍可使用。每日重設的新鮮度使用工作階段資料列的 `sessionStartedAt`；閒置重設的新鮮度使用 `lastInteractionAt`。心跳偵測、排程喚醒、執行通知和閘道簿記等背景／系統事件寫入可更新 `updatedAt`，但不會讓每日／閒置工作階段維持新鮮。
  - **`resetByType`**：依類型覆寫（`direct`、`group`、`thread`）。Doctor 會將舊版 `dm` 項目遷移至 `direct`；結構描述會拒絕 `dm`。
- **`resetByChannel`**：依提供者／頻道 ID 作為索引鍵的各頻道重設覆寫。當工作階段的頻道有相符項目時，該項目會完全優先於該工作階段的 `resetByType`/`reset`。僅在某個頻道需要與類型層級原則不同的重設行為時使用。
- **`mainKey`**：舊版欄位。執行階段一律使用 `"main"` 作為主要直接聊天分組。
- **`sendPolicy`**：依 `channel`、`chatType`（`direct|group|channel`，含舊版 `dm` 別名）、`keyPrefix` 或 `rawKeyPrefix` 比對。第一個拒絕項目優先。
- **`maintenance`**：工作階段儲存區清理 + 保留控制。
  - `mode`：`enforce` 會套用清理且為預設值；`warn` 僅發出警告。
  - `pruneAfter`：過期項目的存留時間截止值（預設為 `30d`）。
  - `maxEntries`：SQLite 工作階段項目的最大數量（預設為 `500`）。對於生產環境規模的上限，執行階段寫入會以小幅高水位緩衝區進行批次清理；`openclaw sessions cleanup --enforce` 會立即套用上限。
  - 短期閘道模型執行探查工作階段使用固定的 `24h` 保留期，但清理受壓力條件限制：只有在達到工作階段項目維護／上限壓力時，才會移除過期且嚴格的模型執行探查資料列。只有符合 `agent:*:explicit:model-run-<uuid>` 的嚴格明確探查索引鍵符合資格；一般直接、群組、討論串、排程、鉤子、心跳偵測、ACP 和子代理程式工作階段不會繼承此 24 小時保留期。模型執行清理會先於範圍更廣的 `pruneAfter` 過期項目清理和 `maxEntries` 上限執行。
  - 目前的結構描述會拒絕舊版 `rotateBytes`；`openclaw doctor --fix` 會從較舊的設定中移除它。
  - `resetArchiveRetention`：針對已重設／刪除逐字稿封存檔的依存留時間保留原則。根據預設，封存檔會保留至因磁碟預算而遭到逐出；設定持續時間即可選擇依實際時間刪除，或設定 `false` 以明確停用。
  - `maxDiskBytes`：選用的工作階段目錄磁碟預算。在 `warn` 模式下會記錄警告；在 `enforce` 模式下會優先移除最舊的成品／工作階段。
  - `highWaterBytes`：預算清理後的選用目標。預設為 `maxDiskBytes` 的 `80%`。
- **`threadBindings`**：討論串繫結工作階段功能的全域預設值。
  - `enabled`：支援的頻道討論串繫結總開關
  - `idleHours`：預設的閒置自動取消聚焦時數（`0` 會停用；提供者可覆寫）
  - `maxAgeHours`：預設的絕對最長存留時數（`0` 會停用；提供者可覆寫）
  - `spawnSessions`：從 `sessions_spawn` 和 ACP 討論串衍生建立討論串繫結工作階段的預設閘門。啟用討論串繫結時預設為 `true`；提供者／帳號可覆寫。
  - `defaultSpawnContext`：討論串繫結衍生的預設原生子代理程式情境（`"fork"` 或 `"isolated"`）。預設為 `"fork"`。
- **`sharing`**：控制擁有者和 `operator.admin` 連線可選取哪些各工作階段協作模式。每個旗標預設為 `true`；將其中一個設為 `false`，會從 Control UI 移除該選項，並使建立時可見性或 `session.visibility.set` 拒絕該選項。除非 Control UI 將新工作階段建立為草稿，否則新工作階段會以 `shared` 啟動。
  - `readOnly`：允許 `read-only`，非成員可觀看，但無法傳送、引導、中止、核准或變更工作階段狀態。
  - `suggest`：允許 `suggest`。在此階段，它會強制執行與 `read-only` 相同的准入行為；建議佇列是後續功能。
  - `drafts`：允許 `draft`，這會對非管理員、非擁有者隱藏工作階段，使其不會出現在工作階段清單和事件廣播中。

成員資格與可見性變更會以系統附註寫入工作階段逐字稿。這些控制項用於協調共用同一代理程式的操作人員；它們不是租戶之間的安全界線。工作需要隔離時，請使用不同的閘道或代理程式。

</Accordion>

---

## 訊息

```json5
{
  messages: {
    responsePrefix: "🦞", // 或 "auto"
    ackReaction: "👀",
    ackReactionScope: "group-mentions", // group-mentions | group-all | direct | all | off | none
    queue: {
      mode: "steer", // steer（預設）| followup | collect | interrupt
      debounceMs: 500,
      cap: 20,
      drop: "summarize", // old | new | summarize（預設）
      byChannel: {
        whatsapp: "followup",
        telegram: "followup",
      },
    },
    inbound: {
      debounceMs: 2000, // 0 會停用
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

### 回覆前綴

各頻道／帳號覆寫：`channels.<channel>.responsePrefix`、`channels.<channel>.accounts.<id>.responsePrefix`。

解析順序（最明確者優先）：帳號 → 頻道 → 全域。`""` 會停用並停止級聯。`"auto"` 會衍生 `[{identity.name}]`。

**範本變數：**

| 變數          | 說明            | 範例                     |
| ----------------- | ---------------------- | --------------------------- |
| `{model}`         | 簡短模型名稱       | `claude-opus-4-6`           |
| `{modelFull}`     | 完整模型識別碼  | `anthropic/claude-opus-4-6` |
| `{provider}`      | 提供者名稱          | `anthropic`                 |
| `{thinkingLevel}` | 目前的思考層級 | `high`、`low`、`off`        |
| `{identity.name}` | 代理程式身分名稱    | （與 `"auto"` 相同）          |

變數不區分大小寫。`{think}` 是 `{thinkingLevel}` 的別名。

### 確認反應

- 預設為作用中代理程式的 `identity.emoji`，否則為 `"👀"`。設定 `""` 即可停用。
- 各頻道覆寫：`channels.<channel>.ackReaction`、`channels.<channel>.accounts.<id>.ackReaction`。
- 解析順序：帳號 → 頻道 → `messages.ackReaction` → 身分後援。
- 範圍：`group-mentions`（預設）、`group-all`、`direct`、`all` 或 `off`/`none`（完全停用確認反應）。
- `messages.statusReactions.enabled`：在 Slack、Discord、Signal、Telegram 和 WhatsApp 上啟用生命週期狀態反應。
  在 Discord 上，未設定時，只要確認反應處於作用中，狀態反應就會保持啟用。
  在 Slack、Signal、Telegram 和 WhatsApp 上，請明確將其設為 `true` 以啟用生命週期狀態反應。
  Slack 預設使用其原生助理討論串狀態和輪替載入訊息來顯示進度，同時讓已設定的確認反應保持不變。

### 佇列

- `mode`：工作階段執行期間收到傳入訊息時所使用的佇列策略。預設：`"steer"`。
  - `steer`：將新提示注入作用中的執行。
  - `followup`：在作用中執行完成後執行新提示。
  - `collect`：將相容的訊息分批，稍後一起執行。
  - `interrupt`：在啟動最新提示前中止作用中的執行。
- `debounceMs`：分派已排入佇列／已引導訊息前的延遲。預設：`500`。
- `cap`：套用捨棄原則前的最大排隊訊息數。預設：`20`。
- `drop`：超過上限時的策略。`"summarize"`（預設）會捨棄最舊的項目，但保留精簡摘要；`"old"` 會捨棄最舊的項目且不保留摘要；`"new"` 會拒絕最新項目。
- `byChannel`：以提供者 ID 作為索引鍵的各頻道 `mode` 覆寫。
- `debounceMsByChannel`：以提供者 ID 作為索引鍵的各頻道 `debounceMs` 覆寫。

### 傳入防彈跳

將來自同一傳送者且快速連續送達的純文字訊息批次合併為單一代理程式輪次。媒體／附件會立即排清。控制命令會略過防彈跳。預設 `debounceMs`：`2000`。

### 其他訊息索引鍵

- `channels.whatsapp.responsePrefix`：傳出 WhatsApp 回覆前綴。僅當此標準值未設定時，Doctor 才會將已淘汰的傳入 `messagePrefix` 值移至此處。
- `messages.visibleReplies`：控制直接、群組和頻道對話中的可見來源回覆（`"message_tool"` 需要 `message(action=send)` 才能產生可見輸出；`"automatic"` 會如先前一樣發布一般回覆）。
- `messages.usageTemplate` / `messages.responseUsage`：自訂 `/usage` 頁尾範本和預設的每次回覆使用模式（`off | tokens | full`，另含 `on` 作為 `tokens` 的舊版別名）。
- `messages.groupChat.mentionPatterns` / `historyLimit`：群組訊息提及觸發條件和歷程記錄視窗大小。
- `messages.suppressToolErrors`：設為 `true` 時，隱藏向使用者顯示的 `⚠️` 工具錯誤警告（代理程式仍會在情境中看到錯誤並可重試）。預設：`false`。

### TTS（文字轉語音）

```json5
{
  tts: {
    auto: "off", // off（預設）| always | inbound | tagged
    mode: "final", // final | all
    provider: "elevenlabs",
    summaryModel: "openai/gpt-5.4-mini",
    modelOverrides: { enabled: true },
    maxTextLength: 4000,
    timeoutMs: 30000,
    providers: {
      elevenlabs: {
        apiKey: "example-elevenlabs-api-key",
        baseUrl: "https://api.elevenlabs.io",
        speakerVoiceId: "voice_id",
        modelId: "eleven_multilingual_v2",
        seed: 42,
        applyTextNormalization: "auto",
        languageCode: "en",
        voiceSettings: {
          stability: 0.5,
          similarityBoost: 0.75,
          style: 0.0,
          useSpeakerBoost: true,
          speed: 1.0,
        },
      },
      microsoft: {
        speakerVoice: "en-US-MichelleNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
      },
      openai: {
        apiKey: "example-openai-api-key",
        baseUrl: "https://api.openai.com/v1",
        model: "gpt-4o-mini-tts",
        speakerVoice: "coral",
      },
    },
  },
}
```

全域偏好設定路徑屬於機器狀態（預設為
`~/.openclaw/settings/tts.json`；可使用 `OPENCLAW_TTS_PREFS` 覆寫）。進階
多代理程式設定可設定 `agents.entries.<id>.tts.prefsPath`，為每個代理程式使用不同的
偏好設定儲存區。

- `auto` 控制預設的自動 TTS 模式：`off`、`always`、`inbound` 或 `tagged`。`/tts on|off` 可覆寫本機偏好設定，而 `/tts status` 會顯示實際生效的狀態。
- `summaryModel` 會覆寫自動摘要所使用的 `agents.defaults.model.primary`。
- `modelOverrides` 預設為啟用（`enabled !== false`）；`modelOverrides.allowProvider` 則須選擇性啟用。
- API 金鑰會回退至 `ELEVENLABS_API_KEY`/`XI_API_KEY` 和 `OPENAI_API_KEY`。
- 內建語音供應商由外掛管理。若已設定 `plugins.allow`，請加入你想使用的每個 TTS 供應商外掛，例如用於 Edge TTS 的 `microsoft`。舊版 `edge` 供應商 ID 可作為 `microsoft` 的別名使用。
- `providers.openai.baseUrl` 會覆寫 OpenAI TTS 端點。解析順序依次為設定、`OPENAI_TTS_BASE_URL`，最後是 `https://api.openai.com/v1`。
- 當 `providers.openai.baseUrl` 指向非 OpenAI 端點時，OpenClaw 會將其視為與 OpenAI 相容的 TTS 伺服器，並放寬模型與語音驗證。

---

## 對話

對話模式的預設值（macOS/iOS/Android 與瀏覽器控制介面）。

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        speakerVoiceId: "elevenlabs_voice_id",
        voiceAliases: {
          Clawd: "EXAVITQu4vr4xnSDxMaL",
          Roger: "CwhRBWXzGAHq8TQ4Fs17",
        },
        modelId: "eleven_multilingual_v2",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
      mlx: {
        modelId: "mlx-community/Soprano-80M-bf16",
      },
      system: {},
    },
    consultThinkingLevel: "low",
    consultFastMode: true,
    speechLocale: "ru-RU",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
    realtime: {
      provider: "openai",
      providers: {
        openai: {
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
      instructions: "以溫暖的語氣說話，並保持回答簡短。",
      mode: "realtime", // realtime | stt-tts | transcription
      transport: "webrtc", // webrtc | provider-websocket | gateway-relay | managed-room
      vadThreshold: 0.5,
      silenceDurationMs: 500,
      prefixPaddingMs: 300,
      reasoningEffort: "medium",
      brain: "agent-consult", // agent-consult | direct-tools | none
    },
  },
}
```

- 設定多個對話供應商時，`talk.provider` 必須與 `talk.providers` 中的某個鍵相符。
- 舊版扁平化對話鍵（`talk.voiceId`、`talk.voiceAliases`、`talk.modelId`、`talk.outputFormat`、`talk.apiKey`）僅供相容性使用。執行 `openclaw doctor --fix`，將持久化設定改寫為 `talk.providers.<provider>`。
- 語音 ID 會回退至 `ELEVENLABS_VOICE_ID` 或 `SAG_VOICE_ID`（macOS 對話用戶端的行為）。
- `providers.*.apiKey` 接受純文字字串或 SecretRef 物件。
- 僅在未設定對話 API 金鑰時，才會套用 `ELEVENLABS_API_KEY` 回退。
- `providers.*.voiceAliases` 可讓對話指令使用易於理解的名稱。
- `providers.mlx.modelId` 選取 macOS 本機 MLX 輔助程式使用的 Hugging Face 儲存庫。若省略，macOS 會使用 `mlx-community/Soprano-80M-bf16`。
- macOS MLX 播放會透過內建的 `openclaw-mlx-tts` 輔助程式執行（若存在），或使用 `PATH` 上的可執行檔；`OPENCLAW_MLX_TTS_BIN` 可覆寫開發環境中的輔助程式路徑。
- `consultThinkingLevel` 控制介面對話即時 `openclaw_agent_consult` 呼叫背後完整 OpenClaw 代理程式執行的思考層級。保持未設定即可保留一般工作階段／模型行為。
- `consultFastMode` 可為控制介面對話即時諮詢設定單次快速模式覆寫，而不變更工作階段的一般快速模式設定。
- `speechLocale` 設定 Android、iOS 與 macOS 對話語音辨識所使用的 BCP 47 地區設定 ID。Android 也會使用其語言部分引導即時輸入轉錄。保持未設定即可使用裝置預設值。
- `silenceTimeoutMs` 控制對話模式在使用者停止說話後，要等待多久才傳送逐字稿。未設定時會保留平台的預設暫停時間範圍（`700 ms on macOS and Android, 900 ms on iOS`）。
- `realtime.instructions` 會將供應商端系統指示附加至 OpenClaw 內建的即時提示詞，因此可設定語音風格，而不會遺失預設的 `openclaw_agent_consult` 指引。
- `realtime.vadThreshold` 設定供應商的語音活動閾值，範圍從 `0`（最敏感）到 `1`（最不敏感）。未設定時會保留供應商預設值。
- `realtime.silenceDurationMs` 設定供應商提交即時使用者回合前的正整數靜音時間範圍。未設定時會保留供應商預設值。
- `realtime.prefixPaddingMs` 設定偵測到語音開始前所保留音訊的非負整數時間量。未設定時會保留供應商預設值。
- `realtime.reasoningEffort` 設定即時工作階段的供應商專屬推理層級。未設定時會保留供應商預設值。
- `realtime.consultRouting`：當即時供應商產生不含 `openclaw_agent_consult` 的最終使用者逐字稿時，`"provider-direct"`（預設）會保留供應商的直接回覆。`"force-agent-consult"` 則改為透過 OpenClaw 路由已定稿的要求。

---

## 相關內容

- [設定參考](/zh-TW/gateway/configuration-reference) — 所有其他設定鍵
- [設定](/zh-TW/gateway/configuration) — 常見工作與快速設定
- [設定範例](/zh-TW/gateway/configuration-examples)
