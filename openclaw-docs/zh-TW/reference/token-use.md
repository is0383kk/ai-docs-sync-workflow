---
read_when:
    - 說明權杖用量、費用或上下文視窗
    - 偵錯上下文增長或壓縮行為
summary: OpenClaw 如何建構提示詞上下文，以及回報權杖用量與成本
title: Token 使用量與費用
x-i18n:
    generated_at: "2026-07-26T07:34:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6624bceb0bcbca769c9d569389b73b82f1ea73133e09f0ae9859833196d85911
    source_path: reference/token-use.md
    workflow: 16
---

OpenClaw 追蹤的是 **Token**，而不是字元。Token 因模型而異，但對英文文字而言，大多數
OpenAI 風格模型平均每個 Token 約為 4 個字元。

## 系統提示詞的建構方式

OpenClaw 會在每次執行時組合自己的系統提示詞。其中包括：

- 工具清單 + 簡短說明
- Skills 清單（僅中繼資料；指示會視需要透過 `read` 載入）。原生
  Codex 輪次會將精簡的 Skills 區塊作為限定於該輪次的協作
  開發者指示；其他執行框架則會將其置於一般提示詞介面中。
  受 `skills.limits.maxSkillsPromptChars` 限制，並可在
  `agents.entries.*.skillsLimits.maxSkillsPromptChars` 為每個代理程式選擇性覆寫。
- 自我更新指示
- 工作區 + 啟動檔案（`AGENTS.md`、`SOUL.md`、`TOOLS.md`、
  `IDENTITY.md`、`USER.md`、`HEARTBEAT.md`、新增時的 `BOOTSTRAP.md`，以及
  存在時的 `MEMORY.md`）。大型注入檔案會由
  `agents.defaults.bootstrapMaxChars` 截斷（預設：`20000`）；啟動注入總量
  受 `agents.defaults.bootstrapTotalMaxChars` 限制（預設：
  `60000`）。
  - 當該工作區可使用記憶工具時，原生 Codex 輪次不會貼上原始
    `MEMORY.md`；而是會在限定於該輪次的協作開發者指示中取得簡短的記憶指標，
    並視需要使用記憶工具。如果工具遭停用、記憶搜尋不可用，或
    使用中的工作區與代理程式記憶工作區不同，`MEMORY.md`
    會回退至一般的有限輪次上下文路徑。
  - 小寫的根目錄 `memory.md` 永遠不會被注入。它是
    `openclaw doctor --fix` 的舊版修復輸入，後者會將其遷移至 `MEMORY.md`。
  - `memory/*.md` 每日檔案不屬於一般啟動提示詞的一部分；
    在一般輪次中，它們仍會透過記憶工具按需取用。重設／啟動
    模型執行可在第一個輪次前置一次性的啟動上下文區塊，其中包含近期的
    每日記憶，並由
    `agents.defaults.startupContext` 控制。純聊天的 `/new` 和 `/reset`
    會直接確認，而不會叫用模型。
  - 壓縮後的 `AGENTS.md` 摘錄需要明確選擇啟用
    `agents.defaults.compaction.postCompactionSections`；外掛可透過
    `before_prompt_build` 新增其他上下文。
- 時間（UTC + 使用者時區）
- 回覆標籤 + 心跳偵測行為
- 執行階段中繼資料（主機／作業系統／模型／思考）

完整細目請參閱[系統提示詞](/zh-TW/concepts/system-prompt)。

記錄認證資訊或驗證程式碼片段時，請使用
[機密預留位置慣例](/zh-TW/reference/secret-placeholder-conventions)，以
避免僅修改文件時出現機密掃描器的誤判。

## 上下文視窗會計入哪些內容

模型收到的所有內容都會計入上下文限制：

- 系統提示詞（以上所有章節）
- 對話記錄（使用者 + 助理訊息）
- 工具呼叫與工具結果
- 附件／逐字稿（圖片、音訊、檔案）
- 壓縮摘要與修剪產物
- 供應商包裝內容或安全標頭（不可見，但仍會計入）

執行階段負載較高的介面在
`agents.defaults.contextLimits` 下有各自明確的限制（每個代理程式的覆寫位於
`agents.entries.*.contextLimits`）：

| 鍵                       | 用途                                                                     |
| ------------------------ | ------------------------------------------------------------------------ |
| `memoryGetMaxChars`      | `memory_get` 在截斷前傳回的最大字元數。                            |
| `postCompactionMaxChars` | 壓縮後重新整理期間從 `AGENTS.md` 保留的最大字元數。 |

這些是有界限的執行階段摘錄與由執行階段擁有的注入區塊，
與啟動限制、啟動上下文限制及 Skills 提示詞限制分開。

OpenClaw 會根據有效的模型上下文視窗推導即時工具結果限制：
低於 100K Token 時為 `16000` 個字元，100K+ Token 時為
`32000` 個字元，200K+ Token 時為 `64000` 個字元。
執行階段的上下文占比防護也會將單一工具結果限制為上下文視窗的 30%。

當大型供應商視窗會顯著改變成本或延遲時，不會自動啟用。例如，直接使用的
OpenAI GPT-5.5 與 GPT-5.6 模型公布的總視窗為 `1050000` Token，
但 OpenClaw 預設將其作用中執行階段預算設為 `272000` Token。
選擇啟用的 `922000` 輸入預算會保留完整的 `128000` 輸出額度，
而一旦輸入超過 `272000` Token，OpenAI 就會對整個要求套用較高的長上下文定價。
請參閱 [OpenAI 上下文視窗預設值](/zh-TW/providers/openai#context-window-defaults-and-long-context-opt-in)。

對於圖片，OpenClaw 會先縮小逐字稿／工具圖片承載內容，再呼叫
供應商。可使用 `agents.defaults.imageMaxDimensionPx` 調整（預設：
`1200`）：

- 較低的值可減少視覺 Token 用量與承載內容大小。
- 較高的值可為大量使用 OCR／UI 的螢幕截圖保留更多視覺細節。

如需實用的細目（依各注入檔案、工具、Skills 與系統
提示詞大小），請使用 `/context list` 或 `/context detail`。請參閱
[上下文](/zh-TW/concepts/context)。

## 如何查看目前的 Token 用量

在聊天中：

- `/status` -> 包含豐富表情符號的狀態卡，顯示工作階段模型、上下文用量、
  上次回應的輸入／輸出 Token，以及為使用中的模型設定本機定價時的預估成本。
- `/usage off|tokens|full` -> 在每則
  回覆附加個別回應的用量頁尾。每個工作階段會持續保留（儲存為 `responseUsage`）。
  - `/usage reset`（別名：`inherit`、`clear`、`default`）會清除
    工作階段覆寫，使其重新繼承已設定的預設值。
  - `/usage tokens` 顯示輪次的 Token／快取詳細資料。
  - `/usage full` 顯示精簡的模型／上下文／成本詳細資料；只有在 OpenClaw
    擁有使用中模型的用量中繼資料與本機定價時，才會顯示預估成本。自訂
    `messages.usageTemplate` 版面配置可包含 Token／快取欄位。
- `/usage cost` -> OpenClaw 工作階段記錄中的本機成本摘要。

其他介面：

- **終端介面／Web 終端介面：**支援 `/status` 和 `/usage`。
- **命令列介面：**`openclaw status --usage` 和 `openclaw channels list` 會顯示
  正規化的供應商配額視窗（`X% left`，不是個別回應的成本）。
  目前支援用量視窗的供應商：Claude (Anthropic)、ClawRouter、Copilot
  (GitHub)、DeepSeek、Gemini (Google Gemini CLI)、MiniMax、OpenAI、Xiaomi、
  Xiaomi Token Plan 和 z.ai。

用量介面會先正規化常見的供應商原生欄位別名，再進行
顯示。對於 OpenAI 系列的 Responses 流量，這同時包括
`input_tokens`/`output_tokens` 和 `prompt_tokens`/`completion_tokens`，因此
傳輸層特有的欄位名稱不會改變 `/status`、`/usage` 或工作階段
摘要。Gemini CLI 用量也會正規化：預設的 `stream-json`
剖析器會讀取助理的 `message` 事件，而 `stats.cached` 會對應至
`cacheRead`；當命令列介面省略明確的
`stats.input` 欄位時，會使用 `stats.input_tokens - stats.cached`。舊版 JSON 覆寫仍會從
`response` 讀取回覆文字。

對於原生 OpenAI 系列 Responses 流量，WebSocket／SSE 用量別名會以
相同方式正規化；當 `total_tokens` 遺失或為 `0` 時，
總計會回退為正規化後的輸入加輸出。

當目前的工作階段快照內容稀疏時，`/status` 和 `session_status`
可以從最近的逐字稿用量記錄中復原 Token／快取計數器，以及使用中的執行階段模型標籤。
現有的非零即時值仍優先於逐字稿回退值；當儲存的總計遺失或較小時，
較大的提示詞導向逐字稿總計可優先採用。

供應商配額視窗的用量驗證會優先使用供應商特定的掛鉤；
如果供應商沒有掛鉤（或掛鉤未解析出權杖），OpenClaw
會回退至驗證設定檔、環境或設定中相符的 OAuth／API 金鑰認證資訊。

助理逐字稿項目會保存相同的正規化用量結構，包括
當使用中的模型已設定定價且供應商傳回用量中繼資料時的 `usage.cost`。
如此一來，即使即時執行階段狀態已不存在，`/usage cost` 和
以逐字稿為基礎的工作階段狀態仍有穩定的資料來源。

OpenClaw 會將供應商用量計算與目前的上下文
快照分開。供應商 `usage.total` 可包含已快取輸入、輸出，以及
多次工具迴圈模型呼叫，因此適合用於成本與遙測，但可能會高估
即時上下文視窗。上下文顯示與診斷會使用
最新的提示詞快照（`promptTokens`；若沒有提示詞快照，則使用最後一次模型呼叫）來計算
`context.used`。

## 成本估算（顯示時）

成本會根據你的模型定價設定進行估算：

```text
models.providers.<provider>.models[].cost
```

這些是 `input`、`output`、`cacheRead` 和
`cacheWrite` 的**每 1M Token 美元價格**。如果缺少定價，`/usage full` 會省略成本；
當你需要在每則回覆中顯示 Token／快取詳細資料時，請使用
`/usage tokens` 或自訂的 `messages.usageTemplate`。成本顯示並不限於 API 金鑰
驗證：當 `aws-sdk` 等非 API 金鑰供應商的已設定模型項目包含本機定價，且供應商
傳回用量中繼資料時，也可顯示預估成本。

在附屬處理程序和頻道抵達閘道就緒路徑後，OpenClaw 會針對尚未具備本機定價的
已設定模型參照，啟動選用的背景定價啟動程序。該啟動程序會擷取遠端 OpenRouter 與
LiteLLM 定價目錄。在離線或受限網路中，設定 `models.pricing.enabled: false` 即可略過這些
目錄擷取；明確設定的 `models.providers.*.models[].cost` 項目仍會驅動本機成本估算。

## 快取 TTL 與修剪的影響

供應商提示詞快取只會在快取 TTL 視窗內套用。OpenClaw
可選擇執行**快取 TTL 修剪**：快取 TTL 到期後修剪工作階段，
接著重設快取視窗，讓後續要求重複使用新快取的上下文，而非重新快取
完整記錄。這可在工作階段閒置時間超過 TTL 時降低快取寫入成本。

請在[閘道設定](/zh-TW/gateway/configuration)中進行設定，並參閱
[工作階段修剪](/zh-TW/concepts/session-pruning)了解行為詳情。

心跳偵測可讓快取在閒置期間保持**暖機**。如果你的模型快取
TTL 是 `1h`，將心跳偵測間隔設定為略短於該值（例如 `55m`），可
避免重新快取完整提示詞，進而降低快取寫入成本。

在多代理程式設定中，你可以共用一份模型設定，並使用
`agents.entries.*.params.cacheRetention` 為每個代理程式調整快取行為。

如需各項設定的完整指南，請參閱[提示詞快取](/zh-TW/reference/prompt-caching)。

對於 Anthropic API 定價，快取讀取的費用顯著低於輸入
Token，而快取寫入則會以較高的乘數計費。如需最新費率與 TTL 乘數，
請參閱 Anthropic 的提示詞快取定價：
[https://docs.anthropic.com/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/docs/build-with-claude/prompt-caching)

### 範例：使用心跳偵測讓 1h 快取保持暖機

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
    heartbeat:
      every: "55m"
```

### 範例：搭配每個代理程式快取策略的混合流量

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long" # 多數代理程式的預設基準
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m" # 讓長期快取保持暖機，以供深度工作階段使用
    - id: "alerts"
      params:
        cacheRetention: "none" # 避免為突發通知寫入快取
```

`agents.entries.*.params` 會合併至所選模型的 `params` 之上，因此你
可以只覆寫 `cacheRetention`，並原樣繼承其他模型預設值。

### Anthropic 1M 上下文

OpenClaw 會為支援正式發布的 Claude 4.x 模型（例如 Opus 4.8、Opus 4.7、Opus
4.6 和 Sonnet 4.6）配置 Anthropic 的 1M 上下文視窗。這些模型不需要
`params.context1m: true`。

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        alias: opus
```

較舊的設定可以保留 `context1m: true`，但 OpenClaw 不再針對此設定傳送
Anthropic 已停用的 `context-1m-2025-08-07` Beta 標頭，也不會將不支援的舊版 Claude 模型
擴充至 1M。

要求：認證資訊必須具備使用長上下文的資格。否則，
Anthropic 會針對該請求回傳供應商端的速率限制錯誤。

如果你使用 OAuth／訂閱權杖（`sk-ant-oat-*`）向 Anthropic 進行驗證，
OpenClaw 會保留 OAuth 所需的 Anthropic Beta 標頭，同時移除舊設定中仍存在的
已停用 `context-1m-*` Beta。

## 降低權杖壓力的提示

- 使用 `/compact` 摘要長時間的工作階段。
- 在工作流程中精簡大型工具輸出。
- 針對大量使用螢幕截圖的工作階段，降低 `agents.defaults.imageMaxDimensionPx`。
- 保持技能說明簡短（技能清單會注入提示詞）。
- 進行冗長的探索性工作時，優先使用較小的模型。

如需確切的技能清單額外負擔計算公式，請參閱 [Skills](/zh-TW/tools/skills)。

## 相關內容

- [API 使用量與費用](/zh-TW/reference/api-usage-costs)
- [提示詞快取](/zh-TW/reference/prompt-caching)
- [用量追蹤](/zh-TW/concepts/usage-tracking)
