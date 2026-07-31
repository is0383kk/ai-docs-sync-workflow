---
read_when:
    - 你希望在 API 供應商發生故障時有可靠的備援機制
    - 你正在執行本機 AI 命令列介面，並想要重複使用它們
    - 你想瞭解用於命令列介面後端工具存取的 MCP 回環橋接器
summary: 命令列介面後端：本機 AI 命令列介面備援，可選用 MCP 工具橋接器
title: 命令列介面後端
x-i18n:
    generated_at: "2026-07-26T07:51:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ce0427c587bf2a1e0a2ff24b5e76952eecae059e6f900af777b897b2d8d4210
    source_path: gateway/cli-backends.md
    workflow: 16
---

OpenClaw 可在 API 提供者停機、受到速率限制或行為異常時，執行本機 AI 命令列介面，作為僅文字的備援。其設計刻意採取保守方式：

- 不會直接注入 OpenClaw 工具，但具備 `bundleMcp: true` 的後端可以透過迴路 MCP 橋接器接收閘道工具。
- 針對支援 JSONL 串流的命令列介面使用 JSONL 串流。
- 支援工作階段，因此後續輪次能保持連貫。
- 若命令列介面接受圖片路徑，圖片便會傳遞至其中。

將其用作提供「永遠可用」文字回應的安全網，而非主要路徑。如需具備 ACP 工作階段控制、背景工作、討論串／對話繫結，以及持久外部程式設計工作階段的完整執行框架，請改用 [ACP 代理程式](/zh-TW/tools/acp-agents)；命令列介面後端並非 ACP。

<Tip>
  正在建置新的後端外掛嗎？請參閱[命令列介面後端外掛](/zh-TW/plugins/cli-backend-plugins)。本頁說明如何設定及操作已註冊的後端。
</Tip>

## 快速開始

內建的 Anthropic 外掛會註冊預設的 `claude-cli` 後端，因此只要已安裝並登入 Claude Code，無須其他設定即可運作：

```bash
openclaw agent --agent main --message "hi" --model claude-cli/claude-sonnet-4-6
```

若未設定明確的代理程式清單，`main` 是預設代理程式 ID；否則請替換為你自己的代理程式 ID。

閘道服務的 `PATH` 中必須包含該命令列介面。如果部署需要
非標準的可執行檔路徑或引數，請改在
[命令列介面後端外掛](/zh-TW/plugins/cli-backend-plugins)中註冊該轉接器，而不要將啟動
機制放入 `openclaw.json`。

當模型選擇或模型範圍的 `agentRuntime.id` 參照某個後端時，OpenClaw 會自動載入擁有該後端的內建外掛。

## 作為備援使用

將命令列介面後端加入備援清單，使其僅在主要模型失敗時執行：

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["claude-cli/claude-sonnet-4-6"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "claude-cli/claude-sonnet-4-6": {},
      },
    },
  },
}
```

主要提供者因驗證、速率限制或逾時而失敗時，已設定的備援仍符合使用資格，即使它們不在 `agents.defaults.modelPolicy.allow` 中亦然。只有當使用者也應能透過 `/model`、工作階段覆寫或 `--model` 直接選取命令列介面後端模型時，才將該模型加入此原則。`agents.defaults.models` 僅管理各模型的別名、參數及中繼資料。

## 設定

使用者透過模型與執行階段原則選擇已註冊的後端。請維持
模型參照的標準形式，並按模型選取命令列介面執行階段：

```json5
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-5",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
      },
    },
  },
}
```

認證資訊仍存放在 OpenClaw 驗證設定檔或所屬外掛的設定中。
命令、argv、環境、剖析、工作階段、圖片及監控機制皆為透過 `api.registerCliBackend(...)` 註冊的外掛程式碼。

## 運作方式

1. 依提供者前綴選取後端（`claude-cli/...`）。
2. 使用相同的 OpenClaw 提示詞與工作區內容建構系統提示詞。
3. 使用工作階段 ID（若支援）執行命令列介面，以保持歷程一致。內建的 `claude-cli` 後端會為每個 OpenClaw 工作階段維持一個 Claude stdio 處理程序，並透過 stream-json stdin 傳送後續輪次。
4. 剖析輸出（JSON 或純文字）並傳回最終文字。
5. 按後端持久保存工作階段 ID，使後續輪次重複使用同一個命令列介面工作階段。

## 逾時與長時間執行的工作

命令列介面後端具有兩項互相獨立的限制：

- `agents.defaults.timeoutSeconds` 限制整個代理程式輪次。一般閘道輪次會繼承 48 小時的預設值；`0` 會讓輪次預算不受限制。已儲存的覆寫值（例如 `600`）會取代該預設值。
- 命令列介面無輸出監控會停止持續無輸出的子處理程序。每個後端外掛分別管理全新／恢復設定檔，即使整體輪次預算不受限制，監控仍會保持作用。

移除較短的整體逾時覆寫即可恢復 48 小時的預設值，或設定明確的預算，例如 12 小時：

```bash
# 恢復 48 小時的預設值：
openclaw config unset agents.defaults.timeoutSeconds

# 或選擇明確的 12 小時限制：
openclaw config set agents.defaults.timeoutSeconds 43200
```

在命令列介面內啟動的背景工作仍屬於該命令列介面子處理程序的一部分。如果父輪次達到整體限制，OpenClaw 會一併停止子處理程序及其命令列介面內部的背景工作。若要執行持久的長時間工作，請使用已分離的 OpenClaw [子代理程式](/zh-TW/tools/subagents)或 [ACP 代理程式](/zh-TW/tools/acp-agents)；已分離的子代理程式預設沒有執行逾時。

`openclaw agent` 命令也有自己的請求截止時間。其 600 秒備援預設值適用於該次命令叫用，而非一般閘道輪次；請參閱 [`openclaw agent`](/zh-TW/cli/agent)。

### Claude 命令列介面細節

內建的 `claude-cli` 後端偏好使用 Claude Code 的原生技能解析器。當目前的技能快照至少有一項具備實體化路徑的已選技能時，OpenClaw 會透過 `--plugin-dir` 傳入暫時的 Claude Code 外掛，並從附加的系統提示詞中省略重複的 OpenClaw 技能目錄。若沒有實體化的外掛技能，OpenClaw 會保留提示詞目錄作為備援。技能環境變數／API 金鑰覆寫仍會套用至該次執行的子處理程序環境。

Claude 命令列介面有自己的非互動式權限模式；OpenClaw 會將其對應至現有的執行原則，而非新增 Claude 專用設定。對於由 OpenClaw 管理的 Claude 即時工作階段，有效的執行原則具有最終決定權：YOLO（`tools.exec.mode: "full"`）通常會使用 `--permission-mode bypassPermissions` 啟動 Claude，而限制性原則則會使用 `--permission-mode default` 啟動。以 root 執行的閘道也會使用 `default`，因為 Claude Code 會拒絕 root 使用略過模式。各代理程式的 `agents.entries.*.tools.exec` 設定會覆寫該代理程式的全域 `tools.exec`。Anthropic 外掛會正規化 Claude 的權限旗標，使其符合有效原則與主機限制。

在限制性原則下，Claude 使用其原生或擴充工具（其自身的 Bash、WebFetch 或 Claude in Chrome 瀏覽器工具）前，會透過 stdio 向 OpenClaw 詢問。當有效執行詢問設定為 `on-miss` 或 `always` 時，OpenClaw 會將每項請求轉送至工作階段的頻道，要求互動式核准：**允許一次**會允許單次呼叫，**一律允許**會在剩餘的 Claude 即時工作階段中允許該工具名稱（僅存於記憶體中，絕不持久保存），而**拒絕**、逾時或無法連線的核准路由都會拒絕呼叫。永不提示的原則會維持原有行為：`security: "deny"` 會拒絕每項請求，而安全性低於完整等級（執行模式為 `allowlist`）時的詢問 `off` 會直接拒絕，不進行詢問。

### Claude 瀏覽器工具與 1Password 登入

Claude Code 可以透過 [Claude in Chrome 擴充功能](https://code.claude.com/docs/en/chrome)操作 Chrome 瀏覽器，包括使用 [1Password for Claude](/zh-TW/gateway/1password#browser-sign-in-with-1password-for-claude)自動填入認證資訊。內建後端不會啟用此功能；請註冊一個[命令列介面後端外掛](/zh-TW/plugins/cli-backend-plugins)，將 `--chrome` 附加至 `claude-stream-json` 方言後端的啟動引數。OpenClaw 會在一般執行中保留已設定的 `--chrome`，而在使用受限工具原則的執行中（例如附帶問題）一律強制使用 `--no-chrome`。Chrome 視窗、擴充功能及任何 1Password 核准提示都位於閘道主機上，因此必須有人在該機器前核准使用認證資訊。

該後端也會將 OpenClaw 的 `/think` 等級對應至 Claude Code 原生的 `--effort` 旗標：`minimal`/`low` -> `low`、`medium` -> `medium`，而 `high`/`xhigh`/`max` 則直接傳遞。這可讓訂閱支援的 Claude 命令列介面與 API 金鑰路由維持相同的 Fable 5 支援投入程度。`adaptive` 會移除已設定的 `--effort` 旗標且不提供替代項，因此 Claude Code 會根據自身的環境、設定及模型預設值解析有效投入程度。其他命令列介面後端必須由其所屬外掛宣告等效的 argv 對應器，`/think` 才會影響產生的命令列介面。

OpenClaw 能使用 `claude-cli` 前，Claude Code 本身必須已在同一部主機登入：

```bash
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

Docker 安裝需要在持久保存的容器主目錄內安裝並登入 Claude Code，而不能只在主機上進行；請參閱 [Docker 中的 Claude 命令列介面後端](/zh-TW/install/docker#claude-cli-backend-in-docker)。

閘道服務必須能在 `PATH` 中解析 `claude`。若使用非標準路徑，
請註冊一個小型包裝後端外掛。

## 工作階段

- 如果命令列介面支援工作階段，請使用 `{sessionId}` 預留位置設定 `sessionArgs`（例如 `["--session-id", "{sessionId}"]`）。
- 如果命令列介面使用具有不同旗標的恢復子命令，請設定 `resumeArgs`（恢復時取代 `args`），並可選擇設定 `resumeOutput` 以進行非 JSON 恢復。
- `sessionMode`：
  - `always`：一律傳送工作階段 ID（若未儲存則使用新的 UUID）。
  - `existing`：僅在先前已儲存工作階段 ID 時傳送。
  - `none`：永不傳送工作階段 ID。
- `claude-cli` 預設為 `liveSession: "claude-stdio"`、`output: "jsonl"` 及 `input: "stdin"`，因此即時 Claude 處理程序處於作用中時，後續輪次會重複使用該程序，包括省略傳輸欄位的自訂設定。如果閘道重新啟動或閒置處理程序結束，OpenClaw 會從已儲存的 Claude 工作階段 ID 恢復。恢復前會確認已儲存的工作階段 ID 對應到可讀取的專案逐字稿；若缺少逐字稿，則會清除繫結（記錄為 `reason=transcript-missing`），而不會在 `--resume` 下悄悄啟動新的工作階段。
- Claude 即時工作階段會維持有界限的 JSONL 輸出防護：每輪 8 MiB 及 20,000 行原始 JSONL。
- 已儲存的命令列介面工作階段是由提供者管理的連續性。預設停用自動重設；`/reset` 及明確的每日或閒置 `session.reset` 原則仍會中斷工作階段。
- 新的命令列介面工作階段通常只會根據 OpenClaw 的壓縮摘要及壓縮後尾端重新植入內容。若要復原在壓縮前失效的短工作階段，後端可以透過 `reseedFromRawTranscriptWhenUncompacted: true` 選擇加入。原始逐字稿重新植入仍有界限，且僅限於安全的失效情況，例如缺少命令列介面逐字稿、孤立的工具使用尾端、訊息原則／系統提示詞／cwd／MCP 變更，或工作階段已過期的重試；驗證設定檔或認證資訊週期變更絕不會重新植入原始逐字稿歷程。

序列化：`serialize: true` 會維持同一通道中的執行順序（大多數命令列介面會在單一提供者通道上序列化）。當選取的驗證身分變更時，OpenClaw 也會停止重複使用已儲存的命令列介面工作階段，包括驗證設定檔 ID、靜態 API 金鑰、靜態權杖或命令列介面公開的 OAuth 帳戶身分有所變更；僅 OAuth 存取／重新整理權杖輪替不會中斷工作階段。如果命令列介面沒有穩定的 OAuth 帳戶 ID，OpenClaw 會讓該命令列介面自行強制執行其恢復權限。

## claude-cli 工作階段的備援前置內容

當 `claude-cli` 嘗試容錯移轉至 [`agents.defaults.model.fallbacks`](/zh-TW/concepts/model-failover) 中的非命令列介面候選項時，OpenClaw 會使用從 Claude Code 本機 JSONL 逐字稿擷取的情境前文來初始化下一次嘗試（位於 `~/.claude/projects/` 下，依工作區作為索引鍵）。若沒有此初始化內容，備援供應商會冷啟動，因為 OpenClaw 自身的工作階段逐字稿對 `claude-cli` 執行而言是空的。

- 前文會優先採用最新的 `/compact` 摘要或 `compact_boundary` 標記，接著附加邊界之後最近的對話輪次，直到達到字元預算為止。邊界之前的對話輪次會被捨棄，因為摘要已經涵蓋它們。
- 工具區塊會合併成精簡的 `(tool call: name)` 和 `(tool result: …)` 提示，以如實維持提示詞預算；過大的摘要會被截斷並標示為 `(truncated)`。
- 同一供應商從 `claude-cli` 到 `claude-cli` 的備援會依賴 Claude 自身的 `--resume`，並略過此前文。
- 初始化內容會重複使用現有的 Claude 工作階段檔案路徑驗證，因此無法讀取任意路徑。

## 圖片

外掛作者使用 `imageArg` 宣告圖片路徑支援：

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw 會將 base64 圖片寫入暫存檔。如果已設定 `imageArg`，這些路徑會作為命令列介面引數傳遞；若未設定，OpenClaw 會將檔案路徑附加至提示詞（路徑注入），這適用於會從純文字路徑自動載入本機檔案的命令列介面。

## 輸入與輸出

- `output: "text"`（預設）將 stdout 視為最終回應。
- `output: "json"` 會嘗試剖析 JSON，並擷取文字及工作階段 ID。
- `output: "jsonl"` 會剖析 JSONL 串流，並在存在時擷取最終代理程式訊息及工作階段識別碼。
- 對於 Gemini 命令列介面的 JSON 輸出，當 `usage` 缺少或為空時，OpenClaw 會從 `response` 讀取回覆文字，並從 `stats` 讀取用量。內建的 Gemini 命令列介面配接器使用 `stream-json`。

輸入模式：

- `input: "arg"`（預設）將提示詞作為最後一個命令列介面引數傳遞。
- `input: "stdin"` 透過 stdin 傳送提示詞。
- 如果提示詞非常長且已設定 `maxPromptArgChars`，則改用 stdin。

## 外掛擁有的預設值

命令列介面後端預設值是外掛介面的一部分：

- 外掛使用 `api.registerCliBackend(...)` 註冊這些預設值。
- 後端 `id` 會成為模型參照中的供應商前綴。
- 命令、argv、環境、剖析器、工作階段及監控程式行為都保留在外掛程式碼中。
- 後端專屬的正規化會透過選用的 `normalizeConfig` 掛鉤維持由外掛擁有。

Anthropic 擁有 `claude-cli`，Google 擁有 `google-gemini-cli`。OpenAI Codex 代理程式執行會透過 `openai/*` 使用 Codex 應用程式伺服器控管機制；OpenClaw 不再註冊內建的 `codex-cli` 後端。

內建的 Anthropic 外掛會為 `claude-cli` 註冊：

| 索引鍵                   | 值                                                                                                                                                                                                         |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`             | `claude`                                                                                                                                                                                                      |
| `args`                | `-p --output-format stream-json --include-partial-messages --verbose --setting-sources user --allowedTools mcp__openclaw__* --disallowedTools ScheduleWakeup,CronCreate,Bash(run_in_background:true),Monitor` |
| `output`              | `jsonl`                                                                                                                                                                                                       |
| `input`               | `stdin`                                                                                                                                                                                                       |
| `modelArg`            | `--model`                                                                                                                                                                                                     |
| `sessionArgs`         | `["--session-id", "{sessionId}"]`                                                                                                                                                                             |
| `sessionMode`         | `always`                                                                                                                                                                                                      |
| `imageArg`            | `@`                                                                                                                                                                                                           |
| `imagePathScope`      | `workspace`                                                                                                                                                                                                   |
| `systemPromptFileArg` | `--append-system-prompt-file`                                                                                                                                                                                 |
| `systemPromptMode`    | `append`                                                                                                                                                                                                      |

內建的 Google 外掛會為 `google-gemini-cli` 註冊：

| 索引鍵                       | 值                                                                                  |
| ------------------------- | -------------------------------------------------------------------------------------- |
| `command`                 | `gemini`                                                                               |
| `args`                    | `--skip-trust --approval-mode auto_edit --output-format stream-json --prompt {prompt}` |
| `resumeArgs`              | 相同，但包含 `--resume {sessionId}`                                                      |
| `output` / `resumeOutput` | `jsonl`                                                                                |
| `jsonlDialect`            | `gemini-stream-json`                                                                   |
| `imageArg`                | `@`                                                                                    |
| `imagePathScope`          | `workspace`                                                                            |
| `modelArg`                | `--model`                                                                              |
| `sessionMode`             | `existing`                                                                             |
| `sessionIdFields`         | `["session_id", "sessionId"]`                                                          |

先決條件：本機必須已安裝 Gemini 命令列介面，且在 `PATH` 中可作為 `gemini` 使用（`brew install gemini-cli` 或 `npm install -g @google/gemini-cli`）。

Gemini 命令列介面輸出注意事項：

- 預設的 `stream-json` 剖析器會讀取助理的 `message` 事件、工具事件、最終 `result` 用量，以及致命的 Gemini 錯誤事件。
- 當 `usage` 不存在或為空時，用量會退回使用 `stats`；`stats.cached` 會正規化為 OpenClaw `cacheRead`，而如果缺少 `stats.input`，輸入權杖數會從 `stats.input_tokens - stats.cached` 推導。

## 文字轉換覆疊

需要小型提示詞／訊息相容性轉接層的外掛，可以宣告雙向文字轉換，而不必取代供應商或命令列介面後端：

```typescript
api.registerTextTransforms({
  input: [{ from: /red basket/g, to: "blue basket" }],
  output: [{ from: /blue basket/g, to: "red basket" }],
});
```

`input` 會重寫傳遞給命令列介面的系統提示詞及使用者提示詞。`output` 會在 OpenClaw 處理自身的控制標記與頻道傳遞之前，重寫串流中的助理文字及剖析後的最終文字；對於由供應商支援的模型呼叫，它也會在串流修復後、工具執行前，還原結構化工具呼叫引數中的字串值。原始供應商 JSON 片段會保持不變；取用端應使用結構化的部分、結束或結果承載資料。

對於會發出供應商專屬 JSONL 事件的命令列介面，請在該後端的設定中設定 `jsonlDialect`：Claude Code 相容串流使用 `claude-stream-json`，Gemini 命令列介面的 `stream-json` 事件使用 `gemini-stream-json`。

## 原生壓縮的所有權

某些命令列介面後端會執行自行壓縮其逐字稿的代理程式，因此 OpenClaw 不得對它們執行防護摘要器，否則會與後端自身的壓縮互相衝突，並可能導致該輪次完全失敗。

`claude-cli` 沒有控管機制端點（Claude Code 會在內部壓縮），因此它會宣告 `ownsNativeCompaction: true`，而 OpenClaw 的壓縮路徑會原封不動地傳回工作階段項目。OpenClaw 會透過 Claude Code 記載的 [`CLAUDE_CODE_AUTO_COMPACT_WINDOW`](https://code.claude.com/docs/en/env-vars) 傳遞此次執行的有效情境預算，使原生自動壓縮與已設定的 Anthropic `contextTokens` 限制保持一致。Codex 等原生控管機制工作階段則會繼續路由至其控管機制的壓縮端點。

```typescript
api.registerCliBackend({ id: "my-cli", ownsNativeCompaction: true /* ... */ });
```

只有在後端真正擁有壓縮時，才能宣告 `ownsNativeCompaction`：它必須可靠地將自身逐字稿限制在情境視窗附近，並保存可恢復的工作階段（例如 `--resume` / `--session-id`），否則延後的工作階段可能持續超出預算。

## 內建 MCP 覆疊

命令列介面後端不會直接接收 OpenClaw 工具呼叫，但後端可以透過 `bundleMcp: true` 選擇加入產生的 MCP 設定覆疊。目前的內建行為：

- `claude-cli`：產生嚴格的 MCP 設定檔。
- `google-gemini-cli`：產生 Gemini 系統設定檔。

啟用內建 MCP 時，OpenClaw 會：

- 產生一個迴路 HTTP MCP 伺服器，向命令列介面處理程序公開閘道工具，並使用僅在目前執行嘗試期間有效的每次執行情境授權（`OPENCLAW_MCP_TOKEN`）進行驗證；
- 將工具存取權繫結至閘道所選的工作階段、帳號及頻道情境，而非信任子處理程序標頭；
- 載入目前工作區已啟用的內建 MCP 伺服器，並將它們與任何現有的後端 MCP 設定／系統設定結構合併；
- 使用擁有該後端之外掛所定義的整合模式，重寫啟動設定。

受限的執行（例如使用 `toolsAllow` 的排程工作）需要由後端擁有的精確轉譯。隨附的 `claude-cli` 後端會停用 Claude 的原生工具，以及使用者、專案和本機自訂項目，包括鉤子、外掛、代理程式、Skills 和 `CLAUDE.md`。接著，它會透過受授權範圍限制的 MCP 伺服器公開每個允許的 OpenClaw 工具。如此可讓檔案系統、程序、執行、核准和沙箱政策保留在 OpenClaw 內，而不會將權限擴大到 Claude 的原生工具或自訂程序。同一份 MCP 清單會在 Claude 產生的設定中強制執行，閘道也會在列出及執行工具時再次強制執行。核心在核發授權之前，會拒絕任何指定原始允許清單以外 MCP 權限的後端轉譯。沒有精確轉譯的後端仍會以封閉方式失敗。

如果未啟用任何 MCP 伺服器，當後端選擇加入隨附 MCP 時，OpenClaw 仍會注入嚴格設定，讓背景執行維持隔離。

工作階段範圍的隨附 MCP 執行環境會快取，以便在工作階段內重複使用，並在閒置 10 分鐘後清除。驗證探測、slug 產生和主動記憶回想等一次性嵌入式執行，會要求在執行結束時清理，避免 stdio 子程序及 Streamable HTTP/SSE 串流在執行結束後繼續存在。

對於 `claude-cli`，相容且已選取或排序的 OpenClaw OAuth/權杖設定檔會轉送至該 Claude 子程序。如此可讓每個代理程式的設定檔成為該回合的權威來源，同時在沒有相容設定檔時保留 Claude 的原生主機登入。

## 重新植入歷程上限

從先前的 OpenClaw 文字記錄植入新的命令列介面工作階段時（例如在 `session_expired` 重試之後），系統會限制算繪後的 `<conversation_history>` 區塊，避免重新植入提示詞急遽膨脹。預設值為 12,288 個字元（約 3,000 個權杖）。

Claude 命令列介面後端則會依解析後的 Claude 上下文視窗調整此上限：較大的上下文視窗可取得較大的先前歷程片段，最高不超過固定上限；其他命令列介面後端維持保守的預設值。此上限只會控制重新植入提示詞中的先前歷程區塊。

## 限制

- OpenClaw 不會將工具呼叫注入命令列介面後端通訊協定。後端只有在選擇加入 `bundleMcp: true` 時，才能看到閘道工具。
- 串流處理因後端而異：部分後端會串流 JSONL，其他後端則會緩衝至程序結束。
- 結構化輸出取決於命令列介面本身的 JSON 格式。

## 疑難排解

| 症狀                  | 修正方式                                                                                             |
| --------------------- | ---------------------------------------------------------------------------------------------------- |
| 找不到命令列介面      | 將命令列介面加入閘道服務的 `PATH`，或更新擁有該功能之外掛所註冊的命令。                  |
| 模型名稱錯誤          | 更新外掛的 `modelAliases` 對應。                                                                 |
| 工作階段無法延續      | 檢查外掛的 `sessionArgs` 和 `sessionMode`。                                                |
| 圖片遭到忽略          | 檢查外掛的 `imageArg`，以及命令列介面對檔案路徑的支援。                                     |

## 相關內容

- [閘道操作手冊](/zh-TW/gateway)
- [本機模型](/zh-TW/gateway/local-models)
