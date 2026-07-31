---
read_when:
    - 設定 `tools.*` 原則、允許清單或實驗性功能
    - 註冊自訂提供者或覆寫基礎 URL
    - 設定與 OpenAI 相容的自行託管端點
sidebarTitle: Tools and custom providers
summary: 工具設定（政策、實驗性切換選項、由供應商支援的工具）及自訂供應商／基礎 URL 設定
title: 設定 — 工具與自訂提供者
x-i18n:
    generated_at: "2026-07-26T07:18:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2010a2e48e8f4c8d0049e5c707bb8286e291a92312baac94301a7b5a674583c1
    source_path: gateway/config-tools.md
    workflow: 16
---

`tools.*` 設定鍵與自訂供應商／基礎 URL 設定。關於代理程式、頻道及其他頂層設定鍵，請參閱[設定參考](/zh-TW/gateway/configuration-reference)。

## 工具

### 工具設定檔

`tools.profile` 會在 `tools.allow`/`tools.deny` 之前設定基礎允許清單：

<Note>
若未設定，本機新手引導預設會將新的本機設定設為 `tools.profile: "coding"`（會保留現有的明確設定檔）。
</Note>

| 設定檔     | 包含                                                                                                                                                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | 僅限 `session_status`                                                                                                                                                                                                                                   |
| `coding`    | `group:fs`、`group:runtime`、`group:web`、`group:sessions`、`group:memory`、`cron`、`get_goal`、`create_goal`、`update_goal`、`update_plan`、`ask_user`、`skill_workshop`、`image`、`image_generate`、`music_generate`、`video_generate`                |
| `messaging` | `group:messaging`、`sessions`、`sessions_list`、`sessions_history`、`sessions_search`、`conversations_list`、`conversations_send`、`conversations_turn`、`sessions_send`、`sessions_spawn`、`sessions_yield`、`subagents`、`session_status`、`ask_user` |
| `full`      | 無限制（等同未設定）                                                                                                                                                                                                                          |

`coding` 與 `messaging` 也會隱含允許 `bundle-mcp`（已設定的 MCP 伺服器）。

### 工具群組

| 群組              | 工具                                                                                                                                                                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec`、`process`、`code_execution`（接受 `bash` 作為 `exec` 的別名）                                                                                                                                                                        |
| `group:fs`         | `read`、`write`、`edit`、`apply_patch`                                                                                                                                                                                                                 |
| `group:sessions`   | `sessions`、`sessions_list`、`sessions_history`、`sessions_search`、`conversations_list`、`conversations_send`、`conversations_turn`、`sessions_send`、`sessions_spawn`、`sessions_yield`、`subagents`、`session_status`、`spawn_task`、`dismiss_task` |
| `group:memory`     | `memory_search`、`memory_get`                                                                                                                                                                                                                          |
| `group:web`        | `web_search`、`x_search`、`web_fetch`                                                                                                                                                                                                                  |
| `group:ui`         | `browser`、`screen`、`terminal`、`canvas`、`show_widget`                                                                                                                                                                                               |
| `group:automation` | `heartbeat_respond`、`cron`、`gateway`                                                                                                                                                                                                                 |
| `group:messaging`  | `message`                                                                                                                                                                                                                                              |
| `group:nodes`      | `nodes`、`computer`                                                                                                                                                                                                                                    |
| `group:agents`     | `agents_list`、`get_goal`、`create_goal`、`update_goal`、`update_plan`、`ask_user`、`skill_workshop`                                                                                                                                                   |
| `group:media`      | `image`、`image_generate`、`music_generate`、`video_generate`、`tts`                                                                                                                                                                                   |
| `group:openclaw`   | 上述所有內建工具，但不包括 `read`/`write`/`edit`/`apply_patch`/`exec`/`process`/`canvas`（不包括外掛工具）                                                                                                                                  |
| `group:plugins`    | 由已載入外掛擁有的工具，包括透過 `bundle-mcp` 公開的已設定 MCP 伺服器                                                                                                                                                           |

`spawn_task` 可讓程式設計代理程式提出已確認的後續工作，而不會立即開始執行。控制介面會將標題與摘要顯示為可操作的晶片；由閘道支援的終端介面則會顯示功能相同的互動式提示。接受任一提示都會建立新的受管理工作樹工作階段，並將完整提示傳送至該處，同時目前回合會繼續進行。`dismiss_task` 會使用 `spawn_task` 傳回的暫時性 `task_id`，撤回仍在等待處理的建議。

只有在啟動操作的操作者介面能接收並處理閘道工作建議事件時，才會提供這些工具。頻道工作階段與本機／嵌入式終端介面工作階段不會接收這些事件；頻道傳輸必須先支援可攜式的具型別工作動作，才能安全地公開此流程。建議僅存在於處理程序本機，並會在閘道重新啟動時消失。這兩項工具仍保留在 `coding` 設定檔與 `group:sessions` 中，因此當介面支援時，一般的 `tools.allow` 與 `tools.deny` 政策會自動設定它們。

### 沙箱工具政策內的 MCP 與外掛工具

已設定的 MCP 伺服器會以 `bundle-mcp` 外掛 ID 之下、由外掛擁有的工具形式公開。一般工具設定檔可以允許這些工具，但對沙箱工作階段而言，`tools.sandbox.tools` 是額外的閘門。若沙箱模式為 `"all"` 或 `"non-main"`，而你希望 MCP／外掛工具可見，請在沙箱工具允許清單中加入下列其中一個項目：

- `bundle-mcp`：來自 `mcp.servers`、由 OpenClaw 管理的 MCP 伺服器
- 特定原生外掛的外掛 ID
- `group:plugins`：所有已載入且由外掛擁有的工具
- 確切的 MCP 伺服器工具名稱或伺服器萬用字元，例如 `outlook__send_mail` 或 `outlook__*`，適用於你只需要一個伺服器時

伺服器萬用字元使用對供應商安全的 MCP 伺服器前綴，不一定是原始的 `mcp.servers` 鍵。非 `[A-Za-z0-9_-]` 字元會變成 `-`，名稱若不是以字母開頭，會加上 `mcp-` 前綴，而過長或重複的前綴可能會遭截短或附加後綴；例如，`mcp.servers["Outlook Graph"]` 會使用類似 `outlook-graph__*` 的萬用字元。

```json5
{
  agents: { defaults: { sandbox: { mode: "all" } } },
  mcp: {
    servers: {
      outlook: { command: "node", args: ["./outlook-mcp.js"] },
    },
  },
  tools: {
    sandbox: {
      tools: {
        alsoAllow: ["web_search", "web_fetch", "memory_search", "memory_get", "bundle-mcp"],
      },
    },
  },
}
```

若沒有該沙箱層項目，MCP 伺服器仍可成功載入，但其工具會在供應商請求之前遭到篩除。使用 `openclaw doctor` 可偵測 `mcp.servers` 中由 OpenClaw 管理之伺服器的這種情況。從隨附外掛資訊清單或 Claude `.mcp.json` 載入的 MCP 伺服器會使用相同的沙箱閘門，但此診斷目前尚不會列舉這些來源；若它們的工具在沙箱回合中消失，請使用相同的允許清單項目。

### `tools.codeMode`

`tools.codeMode` 會啟用通用的 OpenClaw 程式碼模式介面。當在具有工具的執行中啟用時，
一般 OpenClaw 工具會移至沙箱內的 `tools.*`
目錄橋接器後方，而 MCP 工具則可透過產生的 `MCP`
命名空間使用。模型通常會看到 `exec` 與 `wait`；像 `computer`
這類結構化結果無法通過僅限 JSON 的橋接器之工具，則會維持直接提供。

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

也接受簡寫形式：

```json5
{
  tools: { codeMode: true },
}
```

在程式碼模式中，MCP 宣告會透過唯讀虛擬 API 檔案介面公開。
客體程式碼可以呼叫 `API.list("mcp")` 與
`API.read("mcp/<server>.d.ts")`，先檢查 TypeScript 風格的簽章，再
呼叫 `MCP.<server>.<tool>()`。關於執行階段合約、限制與偵錯步驟，請參閱[程式碼模式](/zh-TW/tools/code-mode)。

### `tools.allow` / `tools.deny`

全域工具允許／拒絕政策（拒絕優先）。不區分大小寫，支援 `*` 萬用字元。即使 Docker 沙箱已關閉也會套用。

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

`write` 與 `apply_patch` 是不同的工具 ID。`allow: ["write"]` 也會為相容模型啟用 `apply_patch`，但 `deny: ["write"]` 不會拒絕 `apply_patch`。若要封鎖所有檔案變更，請拒絕 `group:fs`，或明確列出每個會進行變更的工具：

```json5
{
  tools: { deny: ["write", "edit", "apply_patch"] },
}
```

<Note>
`allow` 與 `alsoAllow` 無法在同一個範圍（`tools`、`tools.byProvider.<id>`、`agents.entries.*.tools`）內同時設定——設定驗證會拒絕此情況。請將 `alsoAllow` 項目合併至 `allow`，或移除 `allow`，改用 `profile` + `alsoAllow`。
</Note>

### `tools.byProvider`

針對特定供應商或模型進一步限制工具。順序：基礎設定檔 → 供應商設定檔 → 允許／拒絕。

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.toolsBySender`

限制目前回合原始請求者可使用的工具。這是在頻道存取控制之上的縱深防禦；傳送者值必須來自頻道配接器，而非訊息文字。它不會驗證模型提示詞中的其他內容；請參閱[限定請求者範圍的控制與提示詞情境](/zh-TW/gateway/security#requester-scoped-controls-and-prompt-context)。

```json5
{
  tools: {
    toolsBySender: {
      "channel:discord:1234567890123": { alsoAllow: ["group:fs"] },
      "id:guest-user-id": { deny: ["group:runtime", "group:fs"] },
      "*": { deny: ["exec", "process", "write", "edit", "apply_patch"] },
    },
  },
}
```

鍵使用明確的前置詞：`channel:<channelId>:<senderId>`、`id:<senderId>`、`e164:<phone>`、`username:<handle>`、`name:<displayName>` 或 `"*"`。頻道 ID 是標準 OpenClaw ID；例如 `teams` 的別名會正規化為 `msteams`。舊版無前置詞的鍵只會被接受為 `id:`。比對順序依次為頻道 + ID、ID、e164、使用者名稱、名稱，最後是萬用字元。

當每個代理程式的 `agents.entries.*.tools.toolsBySender` 符合時，會覆寫全域傳送者比對，即使 `{}` 原則為空亦然。

### `tools.elevated`

控制沙箱外的提升權限 exec 存取：

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123", "987654321098765432"],
      },
    },
  },
}
```

- 每個代理程式的覆寫（`agents.entries.*.tools.elevated`）只能進一步限制。
- `/elevated on|off|ask|full` 會依工作階段儲存狀態；行內指示詞僅套用至單一訊息。
- 提升權限的 `exec` 會略過沙箱隔離，並使用已設定的逸出路徑（預設為 `gateway`；當 exec 目標為 `node` 時則為 `node`）。

### `tools.exec`

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      approvalRunningNoticeMs: 10000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      commandHighlighting: false,
      applyPatch: {
        enabled: true,
        allowModels: ["gpt-5.6-sol"],
      },
    },
  },
}
```

除 `applyPatch.allowModels` 外，顯示的值皆為預設值（預設為空白／未設定，表示任何相容模型皆可使用 `apply_patch`）。當需要核准的 exec 執行時間過長時，`approvalRunningNoticeMs` 會發出執行中通知；`0` 則會停用此通知。

### `tools.loopDetection`

工具迴圈安全檢查**預設為停用**。設定 `enabled: true` 以啟用偵測。設定可在 `tools.loopDetection` 中進行全域定義，並可由每個代理程式的 `agents.entries.*.tools.loopDetection` 覆寫。

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
    },
  },
}
```

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // or BRAVE_API_KEY env (Brave provider)
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // optional; omit for auto-detect
        maxChars: 20000,
        maxCharsCap: 20000,
        maxResponseBytes: 750000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true,
        userAgent: "custom-ua",
      },
    },
  },
}
```

除 `provider` 和 `userAgent` 外，顯示的值皆為預設值。`maxResponseBytes` 會限制在 32000–10000000；`maxChars` 會限制為 `maxCharsCap`（提高 `maxCharsCap` 可允許更大的回應）。

### `tools.media`

設定傳入媒體理解（圖片／音訊／影片）：

```json5
{
  tools: {
    media: {
      concurrency: 2,
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          capabilities: ["audio"],
        },
        { provider: "ollama", model: "gemma4:26b", capabilities: ["image"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["video"] },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-mini-transcribe" },
      image: { enabled: true, preferredModel: "ollama/gemma4:26b" },
      video: { enabled: true },
    },
  },
}
```

`tools.media.models` 是唯一設定的模型清單。每個項目都會宣告其處理的能力。選用的 `preferredModel` 選擇器接受 `provider/model`、模型 ID、用於供應商預設項目的 `provider:<id>`，或 `cli:command`；符合的項目會移至該能力備援順序的最前方。針對各能力的提示詞、限制、請求設定、範圍、附件原則和音訊逐字稿回顯，對已設定與自動偵測的模型皆維持預設值；模型項目可覆寫模型專屬欄位。

<AccordionGroup>
  <Accordion title="媒體模型項目欄位">
    **供應商項目**（`type: "provider"` 或省略）：

    - `provider`：API 供應商 ID（`openai`、`anthropic`、`google`/`gemini`、`groq` 等）
    - `model`：模型 ID 覆寫
    - `profile` / `preferredProfile`：`auth-profiles.json` 設定檔選擇

    **命令列介面項目**（`type: "cli"`）：

    - `command`：要執行的可執行檔
    - `args`：樣板化引數（支援 `{{AttachmentPath}}`、`{{AttachmentUrl}}`、`{{AttachmentContentType}}`、`{{AttachmentDir}}`、`{{AttachmentIndex}}`、`{{Prompt}}`、`{{MaxChars}}` 等；`openclaw doctor --fix` 會將已棄用的 `{input}` 預留位置遷移為 `{{AttachmentPath}}`）。較舊的 `{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}` 和 `{{MediaDir}}` 別名在相容期內仍可使用，但已棄用。

    **共用欄位：**

    - `capabilities`：包含 `image`、`audio` 和 `video` 中一個或多個項目的清單。
    - `prompt`、`maxChars`、`maxBytes`、`timeoutSeconds`、`language`：各項目覆寫。
    - 當代理程式呼叫明確的 `image` 工具時，符合的圖片模型 `timeoutSeconds` 項目也會套用。對於圖片理解，此逾時套用至請求本身，不會因先前的準備工作而縮短。
    - 失敗時會退回下一個項目。

    供應商驗證遵循標準順序：`auth-profiles.json` → 環境變數 → `models.providers.*.apiKey`。

  </Accordion>
</AccordionGroup>

### `tools.agentToAgent`

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `tools.sessions`

控制工作階段工具（`sessions_list`、`sessions_history`、`sessions_send`）可將哪些工作階段設為目標。

預設：`tree`（目前工作階段 + 由其產生的工作階段，例如子代理程式，以及同一代理程式的環境感知
受監看群組工作階段）。

```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",
    },
  },
}
```

<AccordionGroup>
  <Accordion title="可見性範圍">
    - `self`：僅限目前的工作階段鍵。
    - `tree`：目前工作階段 + 由目前工作階段產生的工作階段（子代理程式）。對讀取作業而言，也包含目前工作階段透過環境群組感知所監看的同代理程式群組工作階段。
    - `agent`：屬於目前代理程式 ID 的任何工作階段（如果你在同一代理程式 ID 下執行按傳送者區分的工作階段，可能包含其他使用者）。
    - `all`：任何工作階段。跨代理程式指定目標仍需要 `tools.agentToAgent`。
    - 沙箱限制：當目前工作階段位於沙箱中且 `agents.defaults.sandbox.sessionToolsVisibility="spawned"`（預設值）時，即使 `tools.sessions.visibility="all"`，可見性也會強制設為 `tree`。
    - 當不是 `all` 時，`sessions_list` 會包含精簡的 `visibility` 欄位，
      說明有效模式，並警告目前範圍外的某些工作階段可能會
      被省略。

  </Accordion>
</AccordionGroup>

使用預設的 `session.dmScope: "main"` 時，群組中的人類活動會讓該同代理程式群組
工作階段在環境感知下對代理程式的主要工作階段可見。在多使用者設定中，`"main"` 還會讓
多位使用者共用一個私訊工作階段，因此每位被路由至該處的使用者都能讀取環境感知下受監看的群組，
包括透過工作階段記憶體 `memory_search`。若要隔離私訊，請使用按對等端區分的 `dmScope`，或設定
`tools.sessions.visibility: "self"` 以選擇停用環境感知下受監看工作階段的讀取。

### `tools.sessions_spawn`

控制 `sessions_spawn` 的行內附件支援。

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // opt-in: set true to allow inline file attachments
        maxTotalBytes: 5242880, // 5 MB total across all files
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 MB per file
        retainOnSessionKeep: false, // keep attachments when cleanup="keep"
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="附件注意事項">
    - 附件需要 `enabled: true`。
    - 子代理程式附件會具現化至子工作區的 `.openclaw/attachments/<uuid>/`，並附有 `.manifest.json`。
    - ACP 附件僅限圖片，並會在通過相同的檔案數量、單一檔案位元組數及總位元組數限制後，以行內方式轉送至 ACP 執行階段。
    - 附件內容會自動從逐字稿持久化資料中遮蔽。
    - Base64 輸入會經過嚴格的字母表／填補檢查，以及解碼前大小防護。
    - 子代理程式附件的檔案權限為：目錄使用 `0700`，檔案使用 `0600`。
    - 子代理程式清理遵循 `cleanup` 原則：`delete` 一律移除附件；`keep` 僅在 `retainOnSessionKeep: true` 時保留附件。

  </Accordion>
</AccordionGroup>

<a id="toolsexperimental"></a>

### `tools.experimental`

實驗性內建工具旗標。預設關閉，除非符合嚴格代理式 GPT-5 自動啟用規則。

```json5
{
  tools: {
    experimental: {
      planTool: true, // enable experimental update_plan
    },
  },
}
```

- `planTool`：啟用結構化的 `update_plan` 工具，用於追蹤非瑣碎的多步驟工作。
- 預設：`false`，除非 `agents.defaults.embeddedAgent.executionContract`（或每個代理程式的覆寫）在針對 GPT-5 系列模型 ID 的 `openai` 供應商執行中設為 `"strict-agentic"`（這也涵蓋 OpenAI Codex 命令列介面的執行，因為 Codex 驗證／模型路由位於 `openai` 供應商下）。設定 `true` 可在該範圍之外強制啟用工具，或設定 `false`，即使是嚴格代理式 GPT-5 執行也維持關閉。
- 啟用後，系統提示詞也會加入使用指南，讓模型只在實質工作中使用此工具，並且最多只保留一個步驟為 `in_progress`。

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        allowAgents: ["research"],
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 8,
        runTimeoutSeconds: 900,
        announceTimeoutMs: 120000,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model`：所產生子代理程式的預設模型。若省略，子代理程式會繼承呼叫者的模型。
- `allowAgents`：當請求代理程式未設定自己的 `subagents.allowAgents` 時，`sessions_spawn` 已設定目標代理程式 ID 的預設允許清單（`["*"]` = 任何已設定的目標；預設：僅限同一代理程式）。若代理程式設定已刪除，其過時項目會遭 `sessions_spawn` 拒絕，並從 `agents_list` 中省略；執行 `openclaw doctor --fix` 以清除這些項目。
- `maxConcurrent`：子代理程式同時執行數上限。預設值：`8`。
- `runTimeoutSeconds`：呼叫者未傳入自己的覆寫值時，`sessions_spawn` 的逾時時間（秒）。預設值：`0`（不逾時）；上方顯示的 `900` 是常見的選用值，而非內建預設值。
- `announceTimeoutMs`：閘道 `agent` 公告傳遞嘗試的每次呼叫逾時時間（毫秒）。預設值：`120000`。暫時性重試可能使公告的總等待時間超過單次設定的逾時時間。
- `archiveAfterMinutes`：子代理程式工作階段完成後，經過多少分鐘自動封存。預設值：`60`；`0` 會停用自動封存。
- 每個子代理程式的工具原則：`tools.subagents.tools.allow` / `tools.subagents.tools.deny`。

---

## 自訂提供者與基底 URL

提供者外掛會發布自己的模型目錄資料列。透過設定中的 `models.providers` 或 `~/.openclaw/agents/<agentId>/agent/models.json` 新增自訂提供者。

設定自訂／本機提供者的 `baseUrl`，同時也是針對模型 HTTP 請求的精確網路信任決策：OpenClaw 允許該 `scheme://host:port` 的確切來源通過受防護的擷取路徑，無須新增個別設定選項，也不會信任其他私人來源。

```json5
{
  models: {
    mode: "merge", // 合併（預設）| 取代
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions", // openai-completions | openai-responses | anthropic-messages | google-generative-ai | 等
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            contextTokens: 96000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="驗證與合併優先順序">
    - 自訂驗證需求請使用 `authHeader: true` + `headers`。
    - 使用 `OPENCLAW_AGENT_DIR` 覆寫代理程式設定根目錄。
    - 相符提供者 ID 的合併優先順序：
      - 非空白的代理程式 `models.json` `baseUrl` 值優先。
      - 只有當該提供者在目前的設定／驗證設定檔情境中不由 SecretRef 管理時，非空白的代理程式 `apiKey` 值才會優先。
      - 由 SecretRef 管理的提供者 `apiKey` 值會從來源標記重新整理（環境變數參照使用 `ENV_VAR_NAME`，檔案／執行參照使用 `secretref-managed`），而不會保存解析後的祕密。
      - 由 SecretRef 管理的提供者標頭值會從來源標記重新整理（環境變數參照使用 `secretref-env:ENV_VAR_NAME`，檔案／執行參照使用 `secretref-managed`）。
      - 空白或缺少的代理程式 `apiKey`/`baseUrl` 會回復使用設定中的 `models.providers`。
      - 相符模型的 `contextWindow`/`maxTokens`：若明確設定值存在且有效（正有限數），則以該值優先；否則使用隱含／產生的目錄值。
      - 相符模型的 `contextTokens` 遵循相同的「明確值優先，否則使用隱含值」規則；可使用此值限制有效情境，而不變更原生模型中繼資料。
      - 提供者外掛目錄會以產生的外掛自有目錄分片形式，儲存在代理程式的外掛狀態下。
      - 若要讓設定完全重寫 `models.json`，並略過合併外掛自有的目錄分片，請使用 `models.mode: "replace"`。
      - 標記保存以來源為準：標記是從有效的來源設定快照（解析前）寫入，而非從解析後的執行階段祕密值寫入。

  </Accordion>
</AccordionGroup>

### 提供者欄位詳細資料

<AccordionGroup>
  <Accordion title="頂層目錄">
    - `models.mode`：提供者目錄行為（`merge` 或 `replace`）。
    - `models.providers`：以提供者 ID 為鍵的自訂提供者對應。
      - 安全編輯：使用 `openclaw config set models.providers.<id> '<json>' --strict-json --merge` 或 `openclaw config set models.providers.<id>.models '<json-array>' --strict-json --merge` 進行附加式更新。除非傳入 `--replace`，否則 `config set` 會拒絕破壞性取代。

  </Accordion>
  <Accordion title="提供者連線與驗證">
    - `models.providers.*.api`：請求配接器（`openai-completions`、`openai-responses`、`openai-chatgpt-responses`、`anthropic-messages`、`google-generative-ai`、`google-vertex`、`github-copilot`、`bedrock-converse-stream`、`ollama`、`azure-openai-responses`）。對於 MLX、vLLM、SGLang 等自架的 `/v1/chat/completions` 後端，以及大多數與 OpenAI 相容的本機伺服器，請使用 `openai-completions`。具有 `baseUrl` 但沒有 `api` 的自訂提供者，預設使用 `openai-completions`；只有後端支援 `/v1/responses` 時才設定 `openai-responses`。
    - `models.providers.*.apiKey`：提供者認證資訊（建議使用 SecretRef／環境變數替換）。
    - `models.providers.*.auth`：驗證策略（`api-key`、`token`、`oauth`、`aws-sdk`）。
    - `models.providers.*.contextWindow`：當模型項目未設定 `contextWindow` 時，此提供者下模型的預設原生情境視窗。
    - `models.providers.*.contextTokens`：當模型項目未設定 `contextTokens` 時，此提供者下模型的預設有效執行階段情境上限。
    - `models.providers.*.maxTokens`：當模型項目未設定 `maxTokens` 時，此提供者下模型的預設輸出權杖上限。
    - `models.providers.*.timeoutSeconds`：選用的各提供者模型 HTTP 請求逾時時間（秒），包括連線、標頭、本文及整體請求中止處理。
    - `models.providers.*.injectNumCtxForOpenAICompat`：對於 Ollama + `openai-completions`，將 `options.num_ctx` 注入請求（預設值：`true`）。
    - `models.providers.*.authHeader`：需要時，強制透過 `Authorization` 標頭傳輸認證資訊。
    - `models.providers.*.baseUrl`：上游 API 基底 URL。
    - `models.providers.*.headers`：供 Proxy／租用戶路由使用的額外靜態標頭。

  </Accordion>
  <Accordion title="請求傳輸覆寫">
    `models.providers.*.request`：模型提供者 HTTP 請求的傳輸覆寫。

    - `request.headers`：額外標頭（與提供者預設值合併）。值接受 SecretRef。
    - `request.auth`：驗證策略覆寫。模式：`"provider-default"`（使用提供者的內建驗證）、`"authorization-bearer"`（搭配 `token`）、`"header"`（搭配 `headerName`、`value`，以及選用的 `prefix`）。
    - `request.proxy`：HTTP Proxy 覆寫。模式：`"env-proxy"`（使用 `HTTP_PROXY`/`HTTPS_PROXY` 環境變數）、`"explicit-proxy"`（搭配 `url`）。兩種模式都接受選用的 `tls` 子物件。
    - `request.tls`：直接連線的 TLS 覆寫。欄位：`ca`、`cert`、`key`、`passphrase`（皆接受 SecretRef）、`serverName`、`insecureSkipVerify`。
    - `request.allowPrivateNetwork`：當 `true` 時，允許模型提供者 HTTP 請求透過提供者 HTTP 擷取防護存取私人、CGNAT 或類似範圍。自訂／本機提供者的基底 URL 已信任確切設定的來源，但中繼資料／連結本機來源除外；若未明確選用，這些來源仍會遭封鎖。將此值設定為 `false`，可選擇退出確切來源信任。WebSocket 對標頭／TLS 使用相同的 `request`，但不受該擷取 SSRF 閘門限制。預設值為 `false`。

  </Accordion>
  <Accordion title="模型目錄項目">
    - `models.providers.*.models`：明確的提供者模型目錄項目。
    - `models.providers.*.models.*.input`：模型輸入模態。純文字模型使用 `["text"]`，原生圖片／視覺模型使用 `["text", "image"]`。只有所選模型標示為支援圖片時，圖片附件才會注入代理程式回合。
    - `models.providers.*.models.*.contextWindow`：原生模型情境視窗中繼資料。這會覆寫該模型的提供者層級 `contextWindow`。
    - `models.providers.*.models.*.contextTokens`：選用的執行階段情境上限。這會覆寫提供者層級的 `contextTokens`；若要讓有效情境預算小於模型原生的 `contextWindow`，請使用此值；當兩個值不同時，`openclaw models list` 會同時顯示兩者。

    #### 自訂提供者能力宣告

    提供者目錄擁有內建及目錄已知模型路由的 `compat`。請勿將這些旗標複製到設定中：只要設定的 `api` 與 `baseUrl` 仍識別該路由，OpenClaw 就會使用目錄資料列。`openclaw doctor --fix` 會移除相符的舊版覆寫，並回報有差異的值以供審查。

    對於真正的自訂提供者、自訂模型，或路由至不同端點的目錄模型，仍支援 `compat` 區塊。僅設定已針對該端點驗證的能力：

    | 自訂路由鍵 | 執行階段合約 |
    | --- | --- |
    | `supportsStore` | 接受 OpenAI `store` 請求欄位。 |
    | `supportsPromptCacheKey` | 接受 OpenAI 提示快取／工作階段親和性鍵。 |
    | `supportsDeveloperRole` | 接受 `developer` 訊息，而非要求 `system`。 |
    | `supportsReasoningEffort` | 接受推理強度控制。 |
    | `supportsTemperature` | 接受此模型與配接器的 `temperature`。 |
    | `supportsUsageInStreaming` | 在串流回應中輸出用量中繼資料。 |
    | `supportsTools` | 支援結構化工具／函式呼叫。設定 `false` 可停用工具。 |
    | `supportsStrictMode` | 接受嚴格工具結構描述。 |
    | `requiresStringContent` | 要求純字串的 Chat Completions 訊息內容。 |
    | `strictMessageKeys` | 要求傳出訊息僅包含接受的鍵。 |
    | `visibleReasoningDetailTypes` | 指定可安全顯示於文字記錄中的推理詳細資料區塊類型。 |
    | `supportedReasoningEfforts` | 列出端點接受的推理標籤。 |
    | `reasoningEffortMap` | 將 OpenClaw 思考標籤對應至端點特定標籤。 |
    | `maxTokensField` | 選取 `max_tokens` 或 `max_completion_tokens`。 |
    | `thinkingFormat` | 選取端點的推理承載資料方言。 |
    | `requiresToolResultName` | 要求工具結果訊息包含工具名稱。 |
    | `requiresAssistantAfterToolResult` | 要求工具結果後接一則助理訊息。 |
    | `requiresThinkingAsText` | 將推理以文字而非結構化內容重播。 |
    | `requiresReasoningContentOnAssistantMessages` | 重播期間保留 DeepSeek 樣式的 `reasoning_content`。 |
    | `toolSchemaProfile` | 選取提供者定義的工具結構描述正規化設定檔。 |
    | `unsupportedToolSchemaKeywords` | 移除端點拒絕的具名 JSON Schema 關鍵字。 |
    | `toolCallArgumentsEncoding` | 選取端點的工具呼叫引數編碼。 |
    | `requiresOpenAiAnthropicToolPayload` | 將 OpenAI 格式的工具呼叫轉換為 Anthropic 系列承載資料。 |

  </Accordion>
  <Accordion title="Amazon Bedrock 探索">
    - `plugins.entries.amazon-bedrock.config.discovery`：Bedrock 自動探索設定的根節點。
    - `plugins.entries.amazon-bedrock.config.discovery.enabled`：開啟／關閉隱含探索。
    - `plugins.entries.amazon-bedrock.config.discovery.region`：用於探索的 AWS 區域。
    - `plugins.entries.amazon-bedrock.config.discovery.providerFilter`：用於定向探索的選用提供者 ID 篩選器。
    - `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`：探索重新整理的輪詢間隔。
    - `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`：已探索模型的備援上下文視窗。
    - `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`：已探索模型的備援最大輸出權杖數。

  </Accordion>
</AccordionGroup>

互動式自訂提供者的初始設定會根據已知的視覺模型 ID 模式推斷是否支援影像輸入，包括 GPT-4o/GPT-4.1/GPT-5+、`o1`/`o3`/`o4` 推理系列、Claude、Gemini、任何以 `-vl` 結尾的 ID（Qwen-VL 及類似模型），以及 LLaVA、Pixtral、InternVL、Mllama、MiniCPM-V 和 GLM-4V 等具名系列；對於已知的純文字系列（Llama、DeepSeek、Mistral/Mixtral、Kimi/Moonshot、Codestral、Devstral、Phi、QwQ、CodeLlama，以及不含 vl/vision 後綴的單純 Qwen ID），則會略過額外問題。未知模型 ID 仍會詢問是否支援影像。非互動式初始設定使用相同的推斷方式；傳入 `--custom-image-input` 可強制使用支援影像的中繼資料，或傳入 `--custom-text-input` 可強制使用純文字中繼資料。

### 提供者範例

<AccordionGroup>
  <Accordion title="Cerebras（GLM 4.7 / GPT OSS）">
    官方外部 `cerebras` 提供者外掛可透過 `openclaw onboard --auth-choice cerebras-api-key` 完成此設定。僅在覆寫預設值時使用明確的提供者設定。

    ```json5
    {
      env: { CEREBRAS_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: {
            primary: "cerebras/zai-glm-4.7",
            fallbacks: ["cerebras/gpt-oss-120b"],
          },
          models: {
            "cerebras/zai-glm-4.7": { alias: "GLM 4.7 (Cerebras)" },
            "cerebras/gpt-oss-120b": { alias: "GPT OSS 120B (Cerebras)" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          cerebras: {
            baseUrl: "https://api.cerebras.ai/v1",
            apiKey: "${CEREBRAS_API_KEY}",
            api: "openai-completions",
            models: [
              { id: "zai-glm-4.7", name: "GLM 4.7 (Cerebras)" },
              { id: "gpt-oss-120b", name: "GPT OSS 120B (Cerebras)" },
            ],
          },
        },
      },
    }
    ```

    Cerebras 使用 `cerebras/zai-glm-4.7`；直接使用 Z.AI 則用 `zai/glm-4.7`。

  </Accordion>
  <Accordion title="Kimi Coding">
    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: { "kimi/kimi-for-coding": { alias: "Kimi Code" } },
        },
      },
    }
    ```

    內建且與 Anthropic 相容的提供者。捷徑：`openclaw onboard --auth-choice kimi-code-api-key`。

  </Accordion>
  <Accordion title="本機模型（LM Studio）">
    請參閱[本機模型](/zh-TW/gateway/local-models)。簡而言之：在效能強大的硬體上，透過 LM Studio Responses API 執行大型本機模型；保留合併的託管模型作為備援。
  </Accordion>
  <Accordion title="MiniMax M3（直接連線）">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "Minimax" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          minimax: {
            baseUrl: "https://api.minimax.io/anthropic",
            apiKey: "${MINIMAX_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.12, cacheWrite: 0 },
                contextWindow: 1000000,
                maxTokens: 131072,
              },
            ],
          },
        },
      },
    }
    ```

    設定 `MINIMAX_API_KEY`。捷徑：`openclaw onboard --auth-choice minimax-global-api` 或 `openclaw onboard --auth-choice minimax-cn-api`。模型目錄預設為 M3，亦包含 M2.7 變體。在與 Anthropic 相容的串流路徑上，除非你自行明確設定 `thinking`，否則 OpenClaw 預設會停用 MiniMax M2.x 的思考功能；MiniMax-M3（及 M3.x）預設會維持提供者省略／自適應的思考路徑。`/fast on` 或 `params.fastMode: true` 會將 `MiniMax-M2.7` 改寫為 `MiniMax-M2.7-highspeed`。

  </Accordion>
  <Accordion title="Moonshot AI（Kimi）">
    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: { "moonshot/kimi-k2.6": { alias: "Kimi K2.6" } },
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
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
            ],
          },
        },
      },
    }
    ```

    中國端點使用：`baseUrl: "https://api.moonshot.cn/v1"` 或 `openclaw onboard --auth-choice moonshot-api-key-cn`。

    Moonshot 原生端點會在共用的 `openai-completions` 傳輸上宣告串流用量相容性，而 OpenClaw 會根據端點功能判斷，而非僅依據內建提供者 ID。

  </Accordion>
  <Accordion title="OpenCode">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "opencode/claude-opus-4-6" },
          models: { "opencode/claude-opus-4-6": { alias: "Opus" } },
        },
      },
    }
    ```

    設定 `OPENCODE_API_KEY`（或 `OPENCODE_ZEN_API_KEY`）。Zen 目錄使用 `opencode/...` 參照，Go 目錄則使用 `opencode-go/...` 參照。捷徑：`openclaw onboard --auth-choice opencode-zen` 或 `openclaw onboard --auth-choice opencode-go`。

  </Accordion>
  <Accordion title="Synthetic（與 Anthropic 相容）">
    ```json5
    {
      env: { SYNTHETIC_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
          models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          synthetic: {
            baseUrl: "https://api.synthetic.new/anthropic",
            apiKey: "${SYNTHETIC_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "hf:MiniMaxAI/MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```

    基底 URL 應省略 `/v1`（Anthropic 用戶端會附加該部分）。捷徑：`openclaw onboard --auth-choice synthetic-api-key`。

  </Accordion>
  <Accordion title="Z.AI（GLM-4.7）">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-4.7" },
          models: { "zai/glm-4.7": {} },
        },
      },
    }
    ```

    設定 `ZAI_API_KEY`。模型參照使用標準的 `zai/*` 提供者 ID。捷徑：`openclaw onboard --auth-choice zai-api-key`。

    - 一般端點：`https://api.z.ai/api/paas/v4`
    - 程式設計端點：`https://api.z.ai/api/coding/paas/v4`
    - 預設的 `zai-api-key` 驗證選項會探測你的金鑰，並自動偵測其所屬端點（若偵測結果不明確，則改為提示你選擇，預設為 Global）。另亦提供專用的 CN 與 Coding-Plan 驗證選項，供你明確選擇。
    - 對於一般端點，請定義自訂提供者並覆寫基底 URL。

  </Accordion>
</AccordionGroup>

---

## 相關內容

- [設定 — 代理程式](/zh-TW/gateway/config-agents)
- [設定 — 頻道](/zh-TW/gateway/config-channels)
- [設定參考](/zh-TW/gateway/configuration-reference) — 其他頂層鍵
- [工具與外掛](/zh-TW/tools)
