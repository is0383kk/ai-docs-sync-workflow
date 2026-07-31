---
read_when:
    - 你需要精確的欄位層級設定語意或預設值
    - 你正在驗證頻道、模型、閘道或工具設定區塊
summary: 核心 OpenClaw 鍵值、預設值及專用子系統參考連結的閘道設定參考資料
title: 設定參考資料
x-i18n:
    generated_at: "2026-07-26T07:51:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7135554fda444fd1b8c072af5768c53a165f7be2dcd12a7909fc7fd4bd864428
    source_path: gateway/configuration-reference.md
    workflow: 16
---

`~/.openclaw/openclaw.json` 的欄位級參考：鍵、預設值，以及更深入的子系統頁面連結。如需以任務為導向的設定指引，請參閱[設定](/zh-TW/gateway/configuration)。由頻道和外掛擁有的命令目錄，以及深度記憶體／QMD 調整項目，位於各自的頁面，而非此處。

設定格式為 **JSON5**（允許註解與尾隨逗號）。所有欄位皆為選填；省略時，OpenClaw 會使用安全的預設值。

程式碼實際狀態優先於本頁內容：

- `openclaw config schema` 會列印用於驗證和 Control UI 的即時 JSON Schema，其中已合併內建／外掛／頻道的中繼資料。
- 代理程式在編輯設定前，應針對一個路徑範圍明確的結構描述節點，呼叫 `gateway` 工具動作 `config.schema.lookup`。
- `pnpm config:docs:check`／`pnpm config:docs:gen` 會依目前的結構描述介面，驗證本文件的基準雜湊。

結構描述 `uiHints` 也會為每個路徑包含一個已解析的 `advanced` 布林值。
Control UI 使用此值優先顯示常用欄位，並依各
區段摺疊進階欄位；搜尋仍涵蓋兩個層級。層級中繼資料僅用於呈現。
新增鍵時，請在葉節點宣告其層級，或讓它繼承最近的
祖先宣告。沒有任何已宣告祖先的路徑預設為進階層級。

專門的深入參考資料：

- [記憶體設定參考](/zh-TW/reference/memory-config)，涵蓋 `memory.search.*`、`memory.qmd.*`、`memory.citations`，以及 `plugins.entries.memory-core.config.dreaming` 下的夢境整理設定。
- [斜線命令](/zh-TW/tools/slash-commands)，涵蓋目前的內建與隨附命令目錄。
- 各頻道／外掛的擁有者頁面，涵蓋頻道特有的命令介面。

---

## 頻道

各頻道的設定鍵位於[設定 - 頻道](/zh-TW/gateway/config-channels)：Slack、Discord、Telegram、WhatsApp、Matrix、iMessage 和其他隨附頻道的 `channels.*`（驗證、存取控制、多帳號、提及門檻）。

## 代理程式預設值、多代理程式、工作階段與訊息

請參閱[設定 - 代理程式](/zh-TW/gateway/config-agents)，了解：

- `agents.defaults.*`（工作區、模型、思考、心跳偵測、記憶體、媒體、Skills、沙箱）
- `multiAgent.*`（多代理程式路由與繫結）
- `session.*`（工作階段生命週期、壓縮、修剪）
- `messages.*`（訊息傳遞、TTS、Markdown 轉譯）
- `talk.*`（Talk 模式）
  - `talk.consultThinkingLevel`：覆寫 Control UI Talk 即時諮詢背後完整 OpenClaw 代理程式執行的思考層級
  - `talk.consultFastMode`：Control UI Talk 即時諮詢的一次性快速模式覆寫
  - `talk.speechLocale`：Android、iOS 和 macOS 上 Talk 語音辨識的選用 BCP 47 語言環境 ID
  - `talk.silenceTimeoutMs`：未設定時，Talk 會在傳送轉錄文字前保留平台預設的暫停時間範圍（`700 ms on macOS and Android, 900 ms on iOS`）
  - `talk.realtime.consultRouting`：針對略過 `openclaw_agent_consult` 的已完成即時 Talk 轉錄文字，使用閘道轉送備援

## 工具與自訂供應商

工具政策、實驗性切換開關、由供應商支援的工具設定，以及自訂
供應商／基底 URL 設定，位於
[設定 - 工具與自訂供應商](/zh-TW/gateway/config-tools)。

## 模型

供應商定義、模型允許清單和自訂供應商設定位於
[設定 - 工具與自訂供應商](/zh-TW/gateway/config-tools#custom-providers-and-base-urls)。
`models` 根節點也負責全域模型目錄行為。

```json5
{
  models: {
    // 選用。預設值：true。變更後需要重新啟動閘道。
    pricing: { enabled: false },
  },
}
```

- `models.mode`：供應商目錄行為（`merge` 或 `replace`）。
- `models.providers`：以供應商 ID 為鍵的自訂供應商對應表。
- `models.providers.*.localService`：供本機模型伺服器使用的選用隨需程序管理員。OpenClaw 會探測已設定的健康狀態端點、在需要時啟動絕對路徑的 `command`、等待就緒，然後傳送模型要求。請參閱[本機模型服務](/zh-TW/gateway/local-model-services)。
- `models.pricing.enabled`：控制背景定價啟動程序；該程序會在附屬程序和頻道進入閘道就緒路徑後啟動。設為 `false` 時，閘道會略過擷取 OpenRouter 和 LiteLLM 定價目錄；已設定的 `models.providers.*.models[].cost` 值仍可用於本機成本估算。

## MCP

由 OpenClaw 管理的 MCP 伺服器定義位於 `mcp.servers` 下，並由內嵌式 OpenClaw 和其他執行階段配接器使用。`openclaw mcp list`、`show`、`set` 和 `unset` 命令會管理此區塊，而不會在編輯設定時連線至目標伺服器。

```json5
{
  mcp: {
    servers: {
      docs: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-fetch"],
      },
      remote: {
        url: "https://example.com/mcp",
        transport: "streamable-http", // streamable-http | sse
        requestTimeoutMs: 20000,
        connectionTimeoutMs: 5000,
        supportsParallelToolCalls: true,
        headers: {
          Authorization: "Bearer ${MCP_REMOTE_TOKEN}",
        },
        auth: "oauth",
        oauth: {
          scope: "docs.read",
        },
        sslVerify: true,
        clientCert: "/path/to/client.crt",
        clientKey: "/path/to/client.key",
        toolFilter: {
          include: ["search_*"],
          exclude: ["admin_*"],
        },
        // 選用的 Codex app-server 投影控制項。
        codex: {
          agents: ["main"],
          defaultToolsApprovalMode: "approve", // auto | prompt | approve
        },
      },
    },
  },
}
```

- `mcp.servers`：具名的 stdio 或遠端 MCP 伺服器定義，供會公開已設定 MCP 工具的執行階段使用。
  遠端項目使用 `transport: "streamable-http"` 或 `transport: "sse"`；
  `type: "http"` 是命令列介面原生別名，`openclaw mcp set` 和
  `openclaw doctor --fix` 會將其正規化為標準的 `transport` 欄位。
- `mcp.servers.<name>.enabled`：設為 `false`，即可保留已儲存的伺服器定義，
  同時將其排除於內嵌式 OpenClaw MCP 探索和工具投影之外。
- `mcp.servers.<name>.requestTimeoutMs`：各伺服器的 MCP 要求逾時時間，單位為毫秒。
- `mcp.servers.<name>.connectionTimeoutMs`：各伺服器的連線逾時時間，單位為毫秒。
- `mcp.servers.<name>.supportsParallelToolCalls`：供可選擇是否發出平行 MCP 工具呼叫的
  配接器使用的選用並行提示。
- `mcp.servers.<name>.auth`：針對需要 OAuth 的 HTTP MCP 伺服器設為 `"oauth"`。
  執行 `openclaw mcp login <name>`，將權杖儲存在 OpenClaw 狀態下。
- `mcp.servers.<name>.oauth`：選用的 OAuth 範圍、重新導向 URL 和用戶端
  中繼資料 URL 覆寫。
- `mcp.servers.<name>.sslVerify`、`clientCert`、`clientKey`：供私人端點與相互 TLS 使用的 HTTP TLS 控制項。
- `mcp.servers.<name>.toolFilter`：選用的各伺服器工具選擇。`include`
  會將探索到的 MCP 工具限制為符合的名稱；`exclude` 會隱藏符合的
  名稱。項目可以是精確的 MCP 工具名稱，或簡單的 `*` glob。具有
  資源或提示詞的伺服器也會產生公用程式工具名稱（`resources_list`、
  `resources_read`、`prompts_list`、`prompts_get`），這些名稱使用
  相同的篩選器。
- `mcp.servers.<name>.codex`：選用的 Codex app-server 投影控制項。
  此區塊僅供 Codex app-server 執行緒使用的 OpenClaw 中繼資料；不會
  影響 ACP 工作階段、一般 Codex 控制程式設定或其他執行階段配接器。
  非空白的 `codex.agents` 會將伺服器限制於列出的 OpenClaw 代理程式 ID。
  空的、空白的或無效的限定範圍代理程式清單會被設定驗證拒絕，
  並由執行階段投影路徑省略，而不會成為全域設定。
  `codex.defaultToolsApprovalMode` 會為該伺服器發出 Codex 原生的
  `default_tools_approval_mode`。OpenClaw 會先移除 `codex`
  區塊，再將原生 `mcp_servers` 設定傳遞給 Codex。省略此區塊可讓
  伺服器繼續投影至每個 Codex app-server 代理程式，並使用 Codex
  預設的 MCP 核准行為。
- 工作階段範圍的隨附 MCP 執行階段使用內建的 10 分鐘閒置 TTL。
  單次內嵌式執行會要求在執行結束時清理；TTL 則是長期工作階段和未來呼叫端的最後保障。
- `mcp.*` 下的變更會透過處置快取的工作階段 MCP 執行階段即時套用。
  下一次探索／使用工具時，會依新設定重新建立它們，因此已移除的
  `mcp.servers` 項目會立即回收，而不必等待閒置 TTL。
- 執行階段探索也會遵循 MCP 工具清單變更通知，捨棄
  該工作階段的快取目錄。宣告資源或提示詞的伺服器會取得用於列出／讀取資源，以及列出／擷取
  提示詞的公用程式工具。重複的工具呼叫失敗會讓受影響的伺服器暫停片刻，
  之後才會再次嘗試呼叫。

如需了解執行階段行為，請參閱 [MCP](/zh-TW/cli/mcp#openclaw-as-an-mcp-client-registry) 和
[命令列介面後端](/zh-TW/gateway/cli-backends#bundle-mcp-overlays)。

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
      allowUploadedArchives: false,
    },
    workshop: {
      allowSymlinkTargetWrites: false,
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // 或純文字字串
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`：僅適用於隨附 Skills 的選用允許清單（不影響受管理／工作區 Skills）。
- `load.extraDirs`：額外的共用 Skill 根目錄（優先順序最低）。
- `load.allowSymlinkTargets`：當 Skill 符號連結位於其已設定來源根目錄之外時，
  允許其解析至的受信任真實目標根目錄。
- `workshop.allowSymlinkTargetWrites`：允許 Skill Workshop 套用作業透過
  已受信任的符號連結目標寫入（預設值：false）。
- `install.preferBrew`：設為 true 時，若 `brew` 可用，
  會先偏好 Homebrew 安裝程式，再回退至其他安裝程式種類。
- `install.nodeManager`：`metadata.openclaw.install`
  規格的節點安裝程式偏好設定（`npm` | `pnpm` | `yarn` | `bun`）。
- `install.allowUploadedArchives`：允許受信任的 `operator.admin` 閘道
  用戶端安裝透過 `skills.upload.*` 暫存的私人 zip 封存檔
  （預設值：false）。這只會啟用上傳封存檔的路徑；一般 ClawHub
  安裝不需要此設定。
- `entries.<skillKey>.enabled: false` 會停用 Skill，即使它已隨附／安裝亦然。
- `entries.<skillKey>.apiKey`：供宣告主要環境變數的 Skills 使用的便利設定（純文字字串或 SecretRef 物件）。
- `limits.maxCandidatesPerRoot`、`limits.maxSkillsLoadedPerSource`、`limits.maxSkillsInPrompt`、`limits.maxSkillsPromptChars`、`limits.maxSkillFileBytes`：限制 Skill 探索和面向模型的 Skills 提示詞。
- Skill Workshop 自主性／核准設定（`workshop.autonomous.enabled`、`workshop.approvalPolicy`、`workshop.maxPending`、`workshop.maxSkillBytes`）記載於 [Skills 設定](/zh-TW/tools/skills-config)。

---

## 外掛

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-plugin"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- 從 `~/.openclaw/extensions` 和 `<workspace>/.openclaw/extensions` 下的套件或套裝目錄載入，並載入 `plugins.load.paths` 中列出的檔案或目錄。
- 將獨立外掛檔案放在 `plugins.load.paths` 中；自動探索的擴充功能根目錄會忽略頂層的 `.js`、`.mjs` 和 `.ts` 檔案，因此這些根目錄中的輔助指令碼不會阻止啟動。
- 探索功能接受原生 OpenClaw 外掛，以及相容的 Codex 套裝和 Claude 套裝，包括沒有資訊清單、採用 Claude 預設配置的套裝。
- **變更設定需要重新啟動閘道。**
- `allow`：選用的允許清單（只載入列出的外掛）。以 `deny` 為準。
- `plugins.entries.<id>.apiKey`：外掛層級的 API 金鑰便利欄位（外掛支援時）。
- `plugins.entries.<id>.env`：外掛範圍的環境變數對應表。
- `plugins.entries.<id>.hooks.allowPromptInjection`：當 `false` 時，核心會封鎖會修改提示的鉤子，例如 `before_prompt_build`。適用於原生外掛鉤子及支援的套裝所提供的鉤子目錄。
- `plugins.entries.<id>.hooks.allowConversationAccess`：當 `true` 時，受信任的非隨附外掛可從型別化鉤子讀取原始對話內容，例如 `llm_input`、`llm_output`、`before_model_resolve`、`before_agent_reply`、`before_agent_run`、`before_agent_finalize` 和 `agent_end`。
- `plugins.entries.<id>.subagent.allowModelOverride`：明確信任此外掛，允許其為背景子代理程式執行要求每次執行的 `provider` 和 `model` 覆寫。
- `plugins.entries.<id>.subagent.allowedModels`：供受信任子代理程式覆寫使用的標準 `provider/model` 目標選用允許清單。只有在有意允許任何模型時，才使用 `"*"`。
- `plugins.entries.<id>.llm.allowModelOverride`：明確信任此外掛，允許其為 `api.runtime.llm.complete` 要求模型覆寫。
- `plugins.entries.<id>.llm.allowedModels`：供受信任外掛 LLM 補全覆寫使用的標準 `provider/model` 目標選用允許清單。只有在有意允許任何模型時，才使用 `"*"`。
- `plugins.entries.<id>.llm.allowAgentIdOverride`：明確信任此外掛，允許其針對非預設代理程式 ID 執行 `api.runtime.llm.complete`。
- `plugins.entries.<id>.config`：外掛定義的設定物件（可用時由原生 OpenClaw 外掛結構描述驗證）。
- 頻道外掛的帳號／執行階段設定位於 `channels.<id>` 下，且應由所屬外掛資訊清單中的 `channelConfigs` 中繼資料描述，而不是由中央 OpenClaw 選項登錄檔描述。

### Codex 控制框架外掛設定

隨附的 `codex` 外掛擁有位於
`plugins.entries.codex.config` 下的原生 Codex app-server 控制框架設定。完整設定
介面請參閱 [Codex 控制框架參考資料](/zh-TW/plugins/codex-harness-reference)，執行階段模型請參閱 [Codex 控制框架](/zh-TW/plugins/codex-harness)。

`codexPlugins` 僅適用於選取原生 Codex 控制框架的工作階段。
它不會為 OpenClaw 提供者執行、ACP
對話繫結或任何非 Codex 控制框架啟用 Codex 外掛。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
                allow_destructive_actions: false,
              },
            },
          },
        },
      },
    },
  },
}
```

- `plugins.entries.codex.config.codexPlugins.enabled`：為 Codex 控制框架啟用原生 Codex
  外掛／應用程式支援。預設值：`false`。
- `plugins.entries.codex.config.codexPlugins.allow_all_plugins`：在每個新的原生 Codex 對話串中，公開目前已驗證 Codex 帳號所連接且可存取的所有應用程式。預設值：`false`。
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions`：
  已設定外掛應用程式引導要求的預設破壞性動作政策。
  使用 `true` 可在不提示的情況下接受安全的 Codex 核准結構描述，使用 `false`
  可拒絕這些要求，使用 `"auto"` 可透過 OpenClaw
  外掛核准轉送 Codex 所需的核准，或使用 `"ask"`，在沒有持久核准的情況下，針對每個外掛寫入／破壞性動作進行提示。
  `"ask"` 模式會清除受影響應用程式的持久 Codex
  個別工具核准覆寫，並在 Codex 對話串開始前，為該應用程式選取人工
  核准審查者。
  預設值：`true`。
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.enabled`：當全域 `codexPlugins.enabled` 也為 true 時，啟用已設定的外掛項目。
  明確項目的預設值：`true`。
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.marketplaceName`：
  穩定的市集識別資訊，每個已解析項目都必須與 `pluginName` 一併提供。支援 `"openai-curated"` 和 `"workspace-directory"`。缺少任一識別欄位的項目會被忽略。
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.pluginName`：穩定的
  Codex 外掛識別資訊，必須與 `marketplaceName` 一併提供。
  `workspace-directory` 項目必須使用 `plugin/list` 傳回的完整市集限定
  `summary.id`，例如
  `"example-plugin@workspace-directory"`。
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.allow_destructive_actions`：
  個別外掛的破壞性動作覆寫。省略時，會使用全域
  `allow_destructive_actions` 值。個別外掛值接受相同的
  `true`、`false`、`"auto"` 或 `"ask"` 政策。

每個獲准且使用 `"ask"` 的外掛應用程式，都會將該應用程式的核准要求
轉送給人工審查者。其他應用程式及非應用程式的對話串核准會保留其
已設定的審查者，因此混合外掛政策不會繼承 `"ask"` 行為。

`codexPlugins.enabled` 是全域啟用指令。由移轉寫入的明確外掛
項目，是持久的精選安裝與修復資格集合。手動設定的 `workspace-directory` 項目必須已
安裝並啟用，且其所屬應用程式必須可存取；OpenClaw
不會安裝或驗證這些項目。如果 Codex 拒絕明確的工作區
目錄要求，已啟用的工作區項目會以
`marketplace_missing` 關閉失敗，而預設目錄中的精選項目仍然
可用。不支援 `plugins["*"]`，也沒有 `install` 開關；本機
`marketplacePath` 值刻意不作為設定欄位，因為它們
因主機而異。如需 app-server 版本與
就緒要求，請參閱[原生 Codex 外掛](/zh-TW/plugins/codex-native-plugins)。

`app/list` 就緒檢查會快取一小時，並在過期時
以非同步方式重新整理。Codex 對話串應用程式設定會在建立 Codex 控制框架
工作階段時運算，而非每一輪都運算；變更原生外掛設定後，請使用 `/new`、`/reset`，或重新啟動閘道。

`codexPlugins.allow_all_plugins` 會將目前可存取的每個帳號
應用程式快照到每個新的原生 Codex 對話串中。它不會安裝外掛或應用程式，且
無法存取的應用程式仍會遭到排除。帳號應用程式使用全域
`codexPlugins.allow_destructive_actions` 政策。同一應用程式同時出現在兩種途徑時，以明確外掛項目
為準。如果無法讀取 `app/list`，帳號範圍的公開會關閉失敗。

- `plugins.entries.firecrawl.config.webFetch`：Firecrawl 網頁擷取提供者設定。
  - `apiKey`：用於提高限制的選用 Firecrawl API 金鑰（接受 SecretRef）。後備使用 `plugins.entries.firecrawl.config.webSearch.apiKey` 或 `FIRECRAWL_API_KEY` 環境變數。
  - `baseUrl`：Firecrawl API 基底 URL（預設值：`https://api.firecrawl.dev`；自行託管的覆寫必須指向私人／內部端點）。
  - `onlyMainContent`：僅擷取頁面的主要內容（預設值：`true`）。
  - `maxAgeMs`：快取時間上限（毫秒）（預設值：`172800000`／2 天）。
  - `timeoutSeconds`：擷取要求逾時秒數（預設值：`60`）。
- `plugins.entries.xai.config.xSearch`：xAI X Search（Grok 網頁搜尋）設定。
  - `enabled`：啟用 X Search 提供者。
  - `model`：用於搜尋的 Grok 模型（例如 `"grok-4.3"`）。
- `plugins.entries.memory-core.config.dreaming`：記憶夢境整理設定。階段和閾值請參閱[夢境整理](/zh-TW/concepts/dreaming)。
  - `enabled`：夢境整理主開關（預設值為 `false`）。
  - `frequency`：每次完整夢境整理掃描的排程頻率（預設為 `"0 3 * * *"`）。
  - `model`：選用的夢境日記子代理程式模型覆寫。需要 `plugins.entries.memory-core.subagent.allowModelOverride: true`；搭配 `allowedModels` 以限制目標。模型無法使用的錯誤會使用工作階段預設模型重試一次；信任或允許清單失敗不會無提示地後備。
  - 階段政策和閾值是實作細節（不是面向使用者的設定鍵）。
- 完整記憶設定位於[記憶設定參考資料](/zh-TW/reference/memory-config)：
  - `memory.search.*`
  - `agents.entries.*.memory.search.*`，用於個別代理程式覆寫
  - `memory.backend`
  - `memory.citations`
  - `memory.qmd.*`
  - `plugins.entries.memory-core.config.dreaming`
- 已啟用的 Claude 套裝外掛也可以從 `settings.json` 提供內嵌的 OpenClaw 預設值；OpenClaw 會將這些值套用為經過清理的代理程式設定，而不是原始 OpenClaw 設定修補。
- `plugins.slots.memory`：選取作用中的記憶外掛 ID，或選取 `"none"` 以停用記憶外掛。
- `plugins.slots.contextEngine`：選取作用中的情境引擎外掛 ID；除非你安裝並選取其他引擎，否則預設為 `"legacy"`。

請參閱[外掛](/zh-TW/tools/plugin)。

---

## 瀏覽器

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // 僅針對受信任的私人網路存取選擇啟用
      // allowPrivateNetwork: true, // 舊版別名
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    tabCleanup: {
      enabled: true,
      idleMinutes: 120,
      maxTabsPerSession: 8,
      sweepMinutes: 5,
    },
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: { driver: "existing-session", attachOnly: true, color: "#00AA00" },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
    color: "#FF4500",
    // headless: false,
    // noSandbox: false,
    // extraArgs: [],
    // executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    // attachOnly: false,
  },
}
```

- `evaluateEnabled: false` 會停用 `act:evaluate` 和 `wait --fn`。
- `tabCleanup` 控制盡力而為的定期清理，針對閒置一段時間或工作階段超出其上限後所追蹤的主要代理程式
  分頁。追蹤僅適用於由瀏覽器工具 `action: "open"` 建立的
  分頁；使用者開啟或擁有權未知的分頁絕不會被接管。停用 `tabCleanup` 不會停用明確的工作階段生命週期清理。
- 使用穩定原生 CDP 目標和瀏覽器身分在主機本機開啟的項目，
  會儲存在共用 SQLite 狀態中，並在閘道重新啟動後仍符合
  `/new` 和工作階段生命週期清理的資格。提供給原生工具使用的 CDP 目標在重新啟動後，
  也仍符合閒置和上限清理的資格。Chrome MCP 使用
  程序本機目標控制代碼，因此冷啟動的現有工作階段記錄會等待
  生命週期清理，而不會冒險對重新啟動後無法歸屬的
  活動執行閒置清理。OpenClaw 會先驗證設定檔和瀏覽器執行個體，
  再將其關閉。Chrome MCP 自動連線、缺少 `/json/version` 瀏覽器
  身分，以及無法解析的原生目標都會完全保留在程序本機，
  因此重新啟動後不會自動關閉。較舊且未追蹤的分頁需要
  手動關閉。暫時性失敗會維持待處理狀態，以供稍後重試。請參閱
  [分頁清理擁有權](/zh-TW/tools/browser#tab-cleanup-ownership)。
- 未設定 `ssrfPolicy.dangerouslyAllowPrivateNetwork` 時會停用，因此瀏覽器導覽預設維持嚴格模式。
- 只有在你有意信任私人網路瀏覽器導覽時，才設定 `ssrfPolicy.dangerouslyAllowPrivateNetwork: true`。
- 在嚴格模式下，遠端 CDP 設定檔端點（`profiles.*.cdpUrl`）在連線能力／探索檢查期間，也會受到相同的私人網路封鎖。
- `ssrfPolicy.allowPrivateNetwork` 仍支援作為舊版別名。
- 在嚴格模式下，使用 `ssrfPolicy.hostnameAllowlist` 和 `ssrfPolicy.allowedHostnames` 設定明確例外。
- 遠端設定檔僅能附加（已停用啟動／停止／重設）。
- `profiles.*.cdpUrl` 接受 `http://`、`https://`、`ws://` 和 `wss://`。
  若要讓 OpenClaw 探索 `/json/version`，請使用 HTTP(S)；若你的供應商
  提供直接的 DevTools WebSocket URL，請使用 WS(S)。
- 如果可透過回送介面連線至外部管理的 CDP 服務，請設定該
  設定檔的 `attachOnly: true`；否則 OpenClaw 會將回送連接埠視為
  本機受管理的瀏覽器設定檔，並可能回報本機連接埠擁有權錯誤。
- `existing-session` 設定檔使用 Chrome MCP 而非 CDP，並可在
  所選主機上或透過已連線的瀏覽器節點進行附加。
- `existing-session` 設定檔可設定 `userDataDir`，以指定特定的
  Chromium 架構瀏覽器設定檔，例如 Brave 或 Edge。
- 當 Chrome 已在 DevTools HTTP(S) 探索端點或直接 WS(S) 端點
  後方執行時，`existing-session` 設定檔可設定 `cdpUrl`。在該
  模式下，OpenClaw 會將端點傳給 Chrome MCP，而非使用自動連線；
  `userDataDir` 會被 Chrome MCP 啟動引數忽略。
- `existing-session` 設定檔維持目前的 Chrome MCP 路由限制：
  使用快照／參照驅動的動作，而非 CSS 選取器指定目標、單一檔案上傳
  鉤子、不支援對話方塊逾時覆寫、不支援 `wait --load networkidle`，也不支援
  `responsebody`、PDF 匯出、下載攔截或批次動作。
- 本機受管理的 `openclaw` 設定檔會自動指派 `cdpPort` 和 `cdpUrl`；只有遠端 CDP 設定檔或現有工作階段端點
  附加才需明確設定 `cdpUrl`。
- 本機受管理的設定檔可設定 `executablePath`，以針對該設定檔覆寫全域
  `browser.executablePath`。使用此功能可讓一個設定檔在
  Chrome 中執行，另一個則在 Brave 中執行。
- 自動偵測順序：若預設瀏覽器採用 Chromium 架構則優先使用 → Chrome → Brave → Edge → Chromium → Chrome Canary。
- `browser.executablePath` 和 `browser.profiles.<name>.executablePath` 都
  接受 `~` 和 `~/...`，並會在啟動 Chromium 前將其展開為你的作業系統家目錄。
  `existing-session` 設定檔上的個別設定檔 `userDataDir` 也會展開波浪號。
- 控制服務：僅限回送介面（連接埠衍生自 `gateway.port`，預設為 `18791`）。
- `extraArgs` 會在啟動本機 Chromium 時附加額外的啟動旗標（例如
  `--disable-gpu`、視窗大小或偵錯旗標）。

---

## 使用者介面

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // 表情符號、短文字、圖片 URL 或資料 URI
    },
    prefs: {
      theme: "claw", // claw | knot | dash | custom
      themeMode: "system", // light | dark | system
      locale: "en",
      chatShowThinking: true,
      chatShowToolCalls: true,
      chatPersistCommentary: true, // 執行後在控制使用者介面中保留評論；不會將其傳送至頻道
      chatSendShortcut: "enter", // enter | modifier-enter
      chatFollowUpMode: "steer", // steer | queue；省略以使用伺服器佇列模式
      showAdvancedSettings: false, // 展開設定中的每個進階群組
    },
  },
}
```

- `seamColor`：原生應用程式使用者介面框架的強調色（對話模式氣泡色調等）。
- `assistant`：控制使用者介面身分覆寫。若未設定，則使用作用中代理程式身分。
- `prefs`：跨裝置的操作員偏好設定。這是標準的主要位置，因此代理程式可
  透過核准閘門變更這些設定，且每個控制使用者介面用戶端都能保持
  同步；瀏覽器會將值鏡像至本機儲存空間，以便立即啟動，並在
  無法寫入設定時（檢視者範圍、離線）保留裝置本機副本。
  `chatPersistCommentary` 預設為 `true`。將其設定為 `false`，會讓即時
  評論在執行期間保持可見，但在完成時移除，並防止新的
  Codex 評論進入永久逐字稿鏡像。訊息頻道
  傳送則維持獨立且不變。
  `showAdvancedSettings` 預設為 `false`；設定搜尋可能會暫時
  開啟一個相符的進階群組，而不變更此偏好設定。
  僅影響呈現的偏好設定，例如文字縮放比例、聊天寬度和即時
  側邊欄活動，會保留在瀏覽器本機並於設定中進行配置。
  已連線的用戶端會即時套用伺服器端變更：閘道會在每次永久儲存設定後，
  廣播僅含雜湊的 `config.changed` 事件，而
  用戶端會重新整理其快照（當本機設定草稿有未儲存的編輯時略過）。
  重新連線的用戶端會在連線時進行協調。

---

## 閘道

```json5
{
  gateway: {
    mode: "local", // local | remote
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token", // none | token | password | trusted-proxy
      token: "your-token",
      // password: "your-password", // 或 OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // 適用於 mode=trusted-proxy；請參閱 /gateway/trusted-proxy-auth
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,
      },
    },
    tailscale: {
      mode: "off", // off | serve | funnel
      resetOnExit: false,
    },
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      // root: "dist/control-ui",
      // toolTitles: false, // 選擇啟用工具呼叫的 AI 用途標題（會耗用公用模型權杖）
      // embedSandbox: "scripts", // strict | scripts | trusted
      // allowExternalEmbedUrls: false, // 危險：允許絕對外部 http(s) 嵌入 URL
      // allowedOrigins: ["https://control.example.com"], // 非回送介面的控制使用者介面必須設定
      // dangerouslyAllowHostHeaderOriginFallback: false, // 危險的 Host 標頭來源後援模式
    },
    terminal: {
      enabled: false,
      // shell: "/bin/zsh",
    },
    remote: {
      url: "ws://127.0.0.1:18789",
      transport: "ssh", // ssh | direct
      token: "your-token",
      // password: "your-password",
    },
    trustedProxies: ["10.0.0.1"],
    // 選用。預設為 false。
    allowRealIpFallback: false,
    nodes: {
      pairing: {
        // 選用。預設未設定／停用。
        autoApproveCidrs: ["192.168.1.0/24", "fd00:1234:5678::/64"],
        // 經 SSH 驗證的自動核准。預設：啟用（true）。
        // 設定為 false 只會停用 SSH 驗證；這不會影響
        // 上方的 autoApproveCidrs。若只允許手動節點配對，請設為 false 並且
        // 取消設定 autoApproveCidrs。傳入物件以調整：{ user, identity,
        // timeoutMs, cidrs }。
        sshVerify: true,
      },
      commands: {
        allow: ["canvas.navigate"],
        deny: ["system.run"],
      },
    },
    tools: {
      // 額外的 /tools/invoke HTTP 拒絕項目
      deny: ["browser"],
      // 為擁有者／管理員呼叫端從預設 HTTP 拒絕清單中移除工具
      allow: ["gateway"],
    },
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000,
        },
      },
    },
  },
}
```

<Accordion title="閘道欄位詳細資訊">

- `mode`: `local`（執行閘道）或 `remote`（連線至遠端閘道）。除非 `local`，否則閘道會拒絕啟動。
- `port`: WS + HTTP 共用的單一多工連接埠。優先順序：`--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`。
- `bind`: `auto`、`loopback`（預設）、`lan`（`0.0.0.0`）、`tailnet`（可用時使用 Tailscale IPv4，否則使用回送介面），或 `custom`（一個 IPv4 位址）。已解析的 `tailnet` 位址，以及 `127.0.0.1` 或 `0.0.0.0` 以外的任何 `custom` 位址，都要求同主機用戶端在相同連接埠上使用 `127.0.0.1`；若任一監聽器無法繫結，啟動便會失敗。非回送介面的公開範圍仍僅限所選介面。
- **舊版繫結別名**：請使用 `gateway.bind` 中的繫結模式值（`auto`、`loopback`、`lan`、`tailnet`、`custom`），而非主機別名（`0.0.0.0`、`127.0.0.1`、`localhost`、`::`、`::1`）。
- **Docker 注意事項**：預設的 `loopback` 繫結會在容器內監聽 `127.0.0.1`。使用 Docker 橋接網路（`-p 18789:18789`）時，流量會抵達 `eth0`，因此無法連上閘道。請使用 `--network host`，或設定 `bind: "lan"`（或搭配 `customBindHost: "0.0.0.0"` 使用 `bind: "custom"`）以監聽所有介面。
- **驗證**：預設為必要。非回送介面的繫結需要閘道驗證。實務上，這表示需要共用權杖／密碼，或搭配 `gateway.auth.mode: "trusted-proxy"` 的身分感知反向 Proxy。初始設定精靈預設會產生權杖。
- 若同時設定 `gateway.auth.token` 與 `gateway.auth.password`（包括 SecretRef），請明確將 `gateway.auth.mode` 設為 `token` 或 `password`。若兩者皆已設定但未設定模式，啟動及服務安裝／修復流程都會失敗。
- `gateway.auth.mode: "none"`: 明確的無驗證模式。僅限受信任的本機回送介面設定使用；初始設定提示刻意不提供此選項。
- `gateway.auth.mode: "trusted-proxy"`: 將瀏覽器／使用者驗證委派給身分感知反向 Proxy，並信任來自 `gateway.trustedProxies` 的身分標頭（請參閱[受信任 Proxy 驗證](/zh-TW/gateway/trusted-proxy-auth)）。此模式預設要求 Proxy 來源為**非回送介面**；同主機的回送介面反向 Proxy 必須明確設定 `gateway.auth.trustedProxy.allowLoopback = true`。內部同主機呼叫端可以使用 `gateway.auth.password` 作為本機直接備援；`gateway.auth.token` 仍與受信任 Proxy 模式互斥。
- `gateway.auth.allowTailscale`: 當 `true` 時，Tailscale Serve 身分標頭可滿足 Control UI/WebSocket 驗證（透過 `tailscale whois` 驗證）。HTTP API 端點**不會**使用該 Tailscale 標頭驗證；而是依循閘道的一般 HTTP 驗證模式。此無權杖流程假設閘道主機值得信任。當 `tailscale.mode = "serve"` 時，預設為 `true`。
- `gateway.auth.rateLimit`: 選用的驗證失敗限制器。依用戶端 IP 及驗證範圍套用（共用祕密與裝置權杖會分別追蹤）。遭封鎖的嘗試會傳回 `429` + `Retry-After`。
  - 在非同步 Tailscale Serve Control UI 路徑上，對同一 `{scope, clientIp}` 的失敗嘗試會在寫入失敗結果前依序處理。因此，來自同一用戶端的並行錯誤嘗試可能會在第二個請求時觸發限制器，而非兩者競爭通過並僅視為一般不相符。
  - `gateway.auth.rateLimit.exemptLoopback` 預設為 `true`；若你刻意也要限制 localhost 流量的速率（用於測試設定或嚴格的 Proxy 部署），請設定 `false`。
- 來自瀏覽器來源的 WS 驗證嘗試一律受到節流，且停用回送介面豁免（縱深防禦，以防瀏覽器式 localhost 暴力破解）。
- 在回送介面上，這些瀏覽器來源的鎖定會依正規化後的 `Origin`
  值分開處理，因此某個 localhost 來源反覆失敗，不會自動
  鎖定另一個來源。
- `tailscale.mode`: `serve`（僅限 tailnet，回送介面繫結）或 `funnel`（公開，需要驗證）。
- `tailscale.serviceName`: Serve 模式的選用 Tailscale Service 名稱，例如
  `svc:openclaw`。設定後，OpenClaw 會將其傳遞給 `tailscale serve
--service`，讓 Control UI 可透過具名 Service 公開，而非
  裝置主機名稱。此值必須使用 Tailscale 的 `svc:<dns-label>`
  Service 名稱格式；啟動時會回報推導出的 Service URL。
- `tailscale.preserveFunnel`: 當 `true` 且 `tailscale.mode = "serve"` 時，OpenClaw
  會在啟動時重新套用 Serve 前檢查 `tailscale funnel status`，若外部設定的 Funnel 路由已涵蓋閘道連接埠，
  便會略過套用。
  預設為 `false`。
- `controlUi.allowedOrigins`: 閘道 WebSocket 連線的明確瀏覽器來源允許清單。公開的非回送瀏覽器來源必須設定。從回送介面、RFC1918／連結本機、`.local`、`.ts.net` 或 Tailscale CGNAT 主機載入的私有同源 LAN／Tailnet UI，無須啟用 Host 標頭備援即可接受。
- `controlUi.toolTitles`: 選擇啟用 Control UI 聊天中由 AI 產生的工具呼叫用途標題。預設：`false`（工具呈現維持完全確定性，不會進行背景模型呼叫）。啟用後，`chat.toolTitles` 方法會透過標準公用模型路由為複雜呼叫加上標籤——使用代理程式的 `utilityModel`（這是操作者的決策，可能會像每個公用任務一樣，將有限的工具引數傳送至所選提供者），或工作階段提供者宣告的小型模型預設值（OpenAI → `gpt-5.6-luna`、Anthropic → `claude-haiku-4-5`）——並將結果快取至每個代理程式的狀態資料庫，因此重複檢視絕不會再次計費。`utilityModel: \"\"` 會如同其他所有公用任務一樣停用標題；標題絕不會退回使用主要模型。
- `controlUi.dangerouslyAllowHostHeaderOriginFallback`: 危險模式，會針對刻意依賴 Host 標頭來源原則的部署啟用 Host 標頭來源備援。
- `terminal.enabled`: 選擇啟用管理員範圍的操作者終端。預設：`false`。終端會在所選代理程式工作區中啟動主機 PTY、繼承閘道程序環境，且具有 `sandbox.mode: "all"` 的代理程式會遭拒絕。僅限受信任的操作者部署啟用；變更此設定會重新啟動閘道，並更新 Control UI 的內容安全性原則。
- `terminal.shell`: 選用的 Shell 執行檔。未設定時，OpenClaw 在 Unix 上使用 `$SHELL`，在 Windows 上使用 `%ComSpec%`。
- `terminal.detachedSessionTimeoutSeconds`: 終端工作階段在連線中斷（頁面重新載入、筆電休眠）後可存續多久，期間仍可透過 `terminal.attach` 重新連接，並重播其近期輸出。預設：`300`。設定 `0` 可在連線中斷時立即終止工作階段。已中斷連接的工作階段仍會繼續執行其命令，因此在共用或公開主機上請縮短此時間。
- `remote.transport`: `ssh`（預設）或 `direct`（ws/wss）。對於 `direct`，公開主機上的 `remote.url` 必須為 `wss://`；純文字 `ws://` 僅接受回送介面、LAN、連結本機、`.local`、`.ts.net` 及 Tailscale CGNAT 主機。
- `remote.remotePort`: 遠端 SSH 主機上的閘道連接埠。預設為 `18789`；當本機通道連接埠與遠端閘道連接埠不同時使用此設定。
- `remote.tlsFingerprint`: 遠端 `wss://` 閘道的預期 SHA-256 憑證指紋。macOS App 會將其同時套用至操作者／控制與伴隨節點連線。若未明確設定值，macOS 僅會在一般系統信任驗證成功後，記錄首次使用的固定指紋。
- `remote.sshHostKeyPolicy`: macOS SSH 通道主機金鑰原則。`strict` 是預設值，且要求金鑰已受信任。`openssh` 是針對受管理別名之有效 OpenSSH 設定的明確選擇啟用；使用前請檢查相符的使用者與系統 SSH 設定。除非再次明確選擇啟用，否則 macOS App 與 `configure-remote` 在變更目標時會將此原則重設為 `strict`。
- `gateway.remote.token` / `.password` 是遠端用戶端認證資訊欄位。它們本身不會設定閘道驗證。
- `gateway.push.apns.relay.baseUrl`: 由轉送支援的 iOS 組建將註冊發佈至閘道後所使用之外部 APNs 轉送服務的基礎 HTTPS URL。公開的 App Store 組建使用託管的 OpenClaw 轉送服務。自訂轉送 URL 必須對應至刻意分離的 iOS 組建／部署路徑，且該路徑的轉送 URL 指向該轉送服務。
- `gateway.push.apns.relay.timeoutMs`: 閘道至轉送服務的傳送逾時，以毫秒為單位。預設為 `10000`。
- 由轉送支援的註冊會委派給特定閘道身分。已配對的 iOS App 會擷取 `gateway.identity.get`、將該身分納入轉送註冊，並將註冊範圍的傳送授權轉送給閘道。其他閘道無法重複使用該已儲存的註冊。
- `OPENCLAW_APNS_RELAY_BASE_URL` / `OPENCLAW_APNS_RELAY_TIMEOUT_MS`: 上述轉送設定的暫時性環境變數覆寫。
- `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true`: 僅供開發使用的逃生機制，用於回送介面的 HTTP 轉送 URL。正式環境的轉送 URL 應維持使用 HTTPS。
- `OPENCLAW_HANDSHAKE_TIMEOUT_MS`: 內建驗證前閘道 WebSocket 交握逾時的選用環境變數覆寫。
- `channels.<provider>.healthMonitor.enabled`: 在維持全域監控器啟用的情況下，依頻道選擇停用健康狀態監控器重新啟動。
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: 多帳號頻道的個別帳號覆寫。設定後，其優先順序高於頻道層級覆寫。
- 僅當 `gateway.auth.*` 未設定時，本機閘道呼叫路徑才可使用 `gateway.remote.*` 作為備援。
- 若透過 SecretRef 明確設定 `gateway.auth.token` / `gateway.auth.password` 但無法解析，解析會以封閉方式失敗（不允許遠端備援掩蓋問題）。
- `trustedProxies`: 終止 TLS 或注入轉送用戶端標頭的反向 Proxy IP。僅列出你控制的 Proxy。回送介面項目對同主機 Proxy／本機偵測設定仍然有效（例如 Tailscale Serve 或本機反向 Proxy），但它們**不會**使回送介面請求符合 `gateway.auth.mode: "trusted-proxy"` 的資格。
- `allowRealIpFallback`: 當 `true` 時，若缺少 `X-Forwarded-For`，閘道會接受 `X-Real-IP`。預設為 `false`，以採取封閉式失敗行為。
- `gateway.nodes.pairing.autoApproveCidrs`: 選用的 CIDR/IP 允許清單，用於自動核准首次且未要求任何範圍的節點裝置配對。未設定時停用。此設定不會自動核准操作者／瀏覽器／Control UI／WebChat 配對，也不會自動核准角色、範圍、中繼資料或公開金鑰升級。
- `gateway.nodes.pairing.sshVerify`: 首次節點裝置配對的 SSH 驗證自動核准（預設：啟用）。閘道會透過 SSH 連回配對主機（BatchMode、嚴格主機金鑰），且僅在 `openclaw node identity` 裝置金鑰完全相符時核准。資格門檻與 `autoApproveCidrs` 相同；除非 `cidrs` 覆寫，否則探測僅限私有／CGNAT 來源位址。設定 `false` 可停用，或使用 `{ user, identity, timeoutMs, cidrs }` 調整。請參閱[節點配對](/zh-TW/gateway/pairing#ssh-verified-device-auto-approval-default)。
- `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny`：在配對及平台允許清單評估後，針對已宣告的節點命令進行全域允許／拒絕調整。使用 `commands.allow` 選擇啟用危險的節點命令，例如 `camera.snap`、`camera.clip`、`screen.record`、`health.summary`、`sms.search` 及 `sms.send`；即使平台預設值或明確允許原本會包含某項命令，`commands.deny` 仍會將其移除。iOS 健康權限、Android SMS 權限及閘道命令授權彼此獨立。節點變更其宣告的命令清單後，請拒絕該裝置的配對並重新核准，使閘道儲存更新後的命令快照。
- `gateway.tools.deny`：額外針對 HTTP `POST /tools/invoke` 封鎖的工具名稱（擴充預設拒絕清單）。
- `gateway.tools.allow`：從預設 HTTP 拒絕清單中移除工具名稱，適用於
  擁有者／管理員呼叫端。這不會將帶有身分資訊的 `operator.write`
  呼叫端提升為擁有者／管理員存取權；即使已加入允許清單，非擁有者呼叫端仍
  無法使用 `cron`、`gateway` 及 `nodes`。

</Accordion>

### OpenAI 相容端點

- 管理 HTTP RPC：預設關閉，如同 `admin-http-rpc` 外掛。啟用此外掛以註冊 `POST /api/v1/admin/rpc`。請參閱[管理 HTTP RPC](/zh-TW/plugins/admin-http-rpc)。
- Chat Completions：預設停用。使用 `gateway.http.endpoints.chatCompletions.enabled: true` 啟用。
- Responses API：`gateway.http.endpoints.responses.enabled`。
- Responses URL 輸入強化：
  - `gateway.http.endpoints.responses.maxUrlParts`
  - `gateway.http.endpoints.responses.files.urlAllowlist`
  - `gateway.http.endpoints.responses.images.urlAllowlist`
    空白允許清單視為未設定；使用 `gateway.http.endpoints.responses.files.allowUrl=false`
    和／或 `gateway.http.endpoints.responses.images.allowUrl=false` 停用 URL 擷取。
- 選用的回應強化標頭：
  - `gateway.http.securityHeaders.strictTransportSecurity`（僅為你控制的 HTTPS 來源設定；請參閱[受信任的 Proxy 驗證](/zh-TW/gateway/trusted-proxy-auth#tls-termination-and-hsts)）

### 多執行個體隔離

使用不同的連接埠和狀態目錄，在一台主機上執行多個閘道：

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json \
OPENCLAW_STATE_DIR=~/.openclaw-a \
openclaw gateway --port 19001
```

便利旗標：`--dev`（使用 `~/.openclaw-dev` + 連接埠 `19001`）、`--profile <name>`（使用 `~/.openclaw-<name>`）。

請參閱[多個閘道](/zh-TW/gateway/multiple-gateways)。

### `gateway.tls`

```json5
{
  gateway: {
    tls: {
      enabled: false,
      autoGenerate: false,
      certPath: "/etc/openclaw/tls/server.crt",
      keyPath: "/etc/openclaw/tls/server.key",
      caPath: "/etc/openclaw/tls/ca-bundle.crt",
    },
  },
}
```

- `enabled`：在閘道接聽器啟用 TLS 終止（HTTPS/WSS）（預設：`false`）。
- `autoGenerate`：未設定明確檔案時，自動產生本機自簽憑證／金鑰組；僅供本機／開發用途。
- `certPath`：TLS 憑證檔案的檔案系統路徑。
- `keyPath`：TLS 私密金鑰檔案的檔案系統路徑；請限制其權限。
- `caPath`：用於用戶端驗證或自訂信任鏈的選用 CA 套件組合路徑。

### `gateway.reload`

```json5
{
  gateway: {
    reload: {
      mode: "hybrid", // off | restart | hot | hybrid
      debounceMs: 500,
      deferralTimeoutMs: 300000,
    },
  },
}
```

- `mode`：控制如何在執行階段套用設定編輯。
  - `"off"`：忽略即時編輯；變更需要明確重新啟動。
  - `"restart"`：設定變更時，一律重新啟動閘道程序。
  - `"hot"`：不重新啟動，直接在程序內套用變更。
  - `"hybrid"`（預設）：先嘗試熱重新載入；必要時退回重新啟動。
- `debounceMs`：套用設定變更前的去彈跳時間範圍，以毫秒為單位（非負整數；預設：`300`）。
- `deferralTimeoutMs`：在強制重新啟動或頻道熱重新載入之前，等待進行中作業的選用最長時間，以毫秒為單位。省略此值即使用預設的有限等待時間（`300000`）；設為 `0` 則無限期等待，並定期記錄仍在等待中的警告。

---

## 雲端工作節點環境

雲端工作節點為選擇性啟用。如果缺少 `cloudWorkers`，或 `profiles` 為空白，OpenClaw 不接受建立任何新的工作節點。先前建立的永久記錄仍會進行協調並保持可見；現有的閘道／節點投影不受影響。

每個工作節點提供者都必須從受信任的佈建輸出傳回 SSH `hostKey`，且內容必須恰為 `algorithm base64`，不得包含主機名稱或註解。啟動程序會將該金鑰寫入隔離的 `known_hosts` 檔案、使用 `StrictHostKeyChecking=yes`，並在提供者遺漏該金鑰時，於開啟連線前失敗。沒有首次使用即信任的退回機制。

通道會按需設定，而非作為佈建的一部分。啟動後，閘道會將工作節點本機的 Unix 通訊端反向轉送至其迴路 WebSocket 端點。該通訊端位於隨機配置、僅擁有者可存取的遠端目錄中；與迴路 TCP 連接埠不同，多使用者工作節點上的其他帳號無法存取它，也不會與其他環境的連接埠衝突。只有在通道擁有者仍為目前擁有者時，才會執行 SSH 保持連線與設有上限的重新連線退避。停止通道時，會先阻止重新連線，再關閉 SSH 程序。

控制流量與工作區傳輸使用不同的 SSH 連線。兩者會重複使用相同的已解析身分與隔離且已固定的 `known_hosts` 檔案，但工作區傳輸不會與長時間執行的通道共用 SSH 連線多工，因此 rsync 無法阻塞控制流量。

### Crabbox 設定檔

內建的 `crabbox` 提供者會透過本機 Crabbox 命令列介面佈建具備 SSH 功能的租用資源。內層 `settings.provider` 用於選取 Crabbox 後端；它與外層 OpenClaw 提供者 ID 分開。

```json5
{
  cloudWorkers: {
    profiles: {
      production: {
        provider: "crabbox",
        install: "bundle", // 預設；僅針對已發布的閘道版本使用 "npm"。
        settings: {
          provider: "aws",
          class: "standard",
          ttl: "24h",
          idleTimeout: "60m",
          // 選用的絕對路徑。預設：同層的 ../crabbox/bin/crabbox，接著是 PATH。
          binary: "/usr/local/bin/crabbox",
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `settings.provider`（必要）：透過 `--provider` 傳遞的 Crabbox 後端。請使用檢查輸出中包含 SSH 端點的後端；`aws` 會選取直接 AWS 後端。
- `settings.class`（必要）：傳遞至 `--class` 的 Crabbox 機器類別。
- `settings.ttl` 和 `settings.idleTimeout`（必要）：傳遞至 `--ttl` 和 `--idle-timeout` 的正值 Go 持續時間字串。這些提供者端的故障保護機制不同於下方 OpenClaw 儲存的 `lifetime` 原則。
- `settings.binary`：選用的 Crabbox 可執行檔絕對路徑。若未設定，OpenClaw 會先檢查同層的 Crabbox 簽出，接著檢查 `PATH` 上的可執行項目，最後叫用 `crabbox`，讓缺少命令列介面時仍顯示明確的提供者錯誤。

未知的設定會遭拒絕。Crabbox 認證資訊與後端特定的帳號設定仍由 Crabbox 擁有；請勿將其放入 `settings`。OpenClaw 僅叫用本機命令列介面，且此外掛不會發出任何提供者網路呼叫。佈建一律傳遞 `--keep=true`；OpenClaw 擁有外部生命週期，並使用 `crabbox stop` 銷毀租用資源。

<Note>
  OpenClaw 會透過提供者擁有的密鑰解析器解析 Crabbox 租用資源本機的 `sshKey` 路徑，並固定使用 `crabbox inspect --json` 傳回的權威 `sshHostKey`。AWS 准入也需要 `providerMetadata.instanceProfileAttached`。請安裝 Crabbox 0.38.1 或更新版本，以使用此封閉式檢查合約。
</Note>

### 靜態 SSH 開發設定檔

```json5
{
  cloudWorkers: {
    profiles: {
      development: {
        provider: "static-ssh",
        settings: {
          host: "worker.example.test",
          port: 22,
          user: "openclaw",
          hostKey: "ssh-ed25519 <base64-public-host-key>",
          keyRef: {
            source: "env",
            provider: "default",
            id: "OPENCLAW_WORKER_SSH_KEY",
          },
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `profiles`：具名工作節點設定檔，其 ID 必須為非空白且已去除首尾空白。每個設定檔會選取由外掛註冊的提供者。
- `provider`：非空白的工作節點提供者 ID。範例使用內建的 `crabbox` 提供者和 QA Lab 的 `static-ssh` 提供者。
- `install`：工作節點安裝方式。`"bundle"`（預設）會傳輸閘道已安裝建置的內容雜湊套件，並支援已發布、開發中和尚未發布的版本。`"npm"` 是未修改之封裝發布版本的選擇性最佳化；它會從公開 npm 登錄安裝 `openclaw@<exact gateway version>`，且絕不安裝 `latest`。
- 設定後會自動選取內建的提供者外掛，但明確停用和 `plugins.allow` 仍然適用。設定允許清單時，請包含提供者 ID（例如 `crabbox`）。外部提供者外掛也必須安裝並明確啟用。
- `settings`：由提供者擁有且大小受限的 JSON。所選外掛會定義並驗證其索引鍵；對於包含密鑰的值，請使用 [SecretRef 物件](/zh-TW/gateway/secrets)。靜態 SSH 提供者需要 `host`、`user`、`hostKey` 和 `keyRef`；`port` 預設為 `22`。`hostKey` 必須是從已知主機或其他受信任管道取得的一行 OpenSSH 公開主機金鑰（`algorithm base64`），不得有選項前綴。
- `lifetime.idleTimeoutMinutes`：儲存供後續閒置回收原則使用的正整數分鐘數。
- `lifetime.maxLifetimeMinutes`：儲存供後續生命週期原則使用的正整數分鐘數。

工作節點必須已安裝支援的 Node 執行階段（22.22.3+、24.15+ 或 25.9+），並包含可安全重設 WAL 的 SQLite。選擇性啟用的 `"npm"` 方法也需要 `npm`，以及對公開 npm 登錄的輸出 HTTPS 存取權。網路工具鏈設定屬於提供者原則；啟動程序會回報可採取行動的錯誤，而不會自行安裝工具鏈。

此基礎功能會安裝並驗證閘道建置，並提供通道啟動／停止生命週期，但不會啟動一般的 OpenClaw 命令列介面。獨立式工作節點進入點和迴圈會在下一個雲端工作節點里程碑中推出。

每筆永久環境記錄都會在建立時的設定檔快照中保留已驗證的提供者設定、已解析的安裝方式和生命週期原則。變更或移除具名設定檔會影響新建立的項目；只要擁有該項目的外掛仍然可用，現有記錄就會繼續使用該快照進行生命週期協調。

生命週期值在第一版雲端工作節點版本中僅為資料；自動執行會隨後續生命週期工作推出。設定檔變更需要重新啟動閘道。

<Warning>
  `static-ssh` 提供者是原始碼樹中的 QA Lab 開發測試工具，且不包含於封裝發行版本中。在其共用主機上執行的工作節點可讀取不相關的主機資料，因此請勿將此提供者用作正式環境的隔離邊界。
  其操作者必須提供預期的 `hostKey`；OpenClaw 不會從第一次連線得知或接受金鑰。
  銷毀其租用資源只會釋放 OpenClaw 的邏輯記錄；不會停止或清理主機。
</Warning>

---

## 鉤子

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: true,
    allowedSessionKeyPrefixes: ["hook:", "hook:gmail:"],
    allowedAgentIds: ["hooks", "main"],
    presets: ["gmail"],
    transformsDir: "~/.openclaw/hooks/transforms",
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        agentId: "hooks",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "寄件者：{{messages[0].from}}\n主旨：{{messages[0].subject}}\n{{messages[0].snippet}}",
        deliver: true,
        channel: "last",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

驗證：`Authorization: Bearer <token>` 或 `x-openclaw-token: <token>`。
查詢字串中的鉤子權杖會遭拒絕。

驗證與安全注意事項：

- `hooks.enabled=true` 必須有非空白的 `hooks.token`。
- `hooks.token` 應與作用中的閘道共用密鑰驗證（`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` 或 `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`）不同；啟動時若偵測到重複使用，會記錄非致命的安全警告。
- `openclaw security audit` 會將鉤子／閘道驗證重複使用標示為重大發現，包括僅在稽核時提供的閘道密碼驗證（`--auth password --password <password>`）。執行 `openclaw doctor --fix` 以輪替已保存且重複使用的 `hooks.token`，然後更新外部鉤子傳送端，使其使用新的鉤子權杖。
- `hooks.path` 不可為 `/`；請使用專用子路徑，例如 `/hooks`。
- 若為 `hooks.allowRequestSessionKey=true`，請限制 `hooks.allowedSessionKeyPrefixes`（例如 `["hook:"]`）。
- 若對應或預設集使用範本化的 `sessionKey`，請設定 `hooks.allowedSessionKeyPrefixes` 和 `hooks.allowRequestSessionKey=true`。靜態對應鍵不需要此項選擇性啟用。

**端點：**

- `POST /hooks/wake` → `{ text, mode?: "now"|"next-heartbeat" }`
- `POST /hooks/agent` → `{ message, name?, agentId?, sessionKey?, wakeMode?, deliver?, channel?, to?, model?, thinking?, timeoutSeconds? }`
  - 只有當 `hooks.allowRequestSessionKey=true` 時，才接受來自要求承載資料的 `sessionKey`（預設：`false`）。
- `POST /hooks/<name>` → 透過 `hooks.mappings` 解析
  - 由範本呈現的對應 `sessionKey` 值會視為外部提供，因此也需要 `hooks.allowRequestSessionKey=true`。

<Accordion title="對應詳細資料">

- `match.path` 會比對 `/hooks` 之後的子路徑（例如 `/hooks/gmail` → `gmail`）。
- `match.source` 會比對一般路徑的承載資料欄位。
- 類似 `{{messages[0].subject}}` 的範本會從承載資料讀取內容。
- `transform` 可指向傳回鉤子動作的 JS/TS 模組。
  - `transform.module` 必須是相對路徑，且須保持在 `hooks.transformsDir` 內（系統會拒絕絕對路徑與路徑周遊）。
  - 請將 `hooks.transformsDir` 保留在 `~/.openclaw/hooks/transforms` 下；系統會拒絕工作區 Skills 目錄。若 `openclaw doctor` 回報此路徑無效，請將轉換模組移至鉤子轉換目錄，或移除 `hooks.transformsDir`。
- `agentId` 會路由至特定代理程式；未知 ID 會回復使用預設代理程式。
- `allowedAgentIds`：限制實際代理程式路由，包括省略 `agentId` 時的預設代理程式路徑（`*` 或省略 = 全部允許，`[]` = 全部拒絕）。
- `defaultSessionKey`：供未明確指定 `sessionKey` 的鉤子代理程式執行使用的選用固定工作階段金鑰。
- `allowRequestSessionKey`：允許 `/hooks/agent` 呼叫端及由範本驅動的對應工作階段金鑰設定 `sessionKey`（預設：`false`）。
- `allowedSessionKeyPrefixes`：明確 `sessionKey` 值（要求 + 對應）的選用前綴允許清單，例如 `["hook:"]`。任何對應或預設集使用範本化的 `sessionKey` 時，此設定即為必要。
- `deliver: true` 會將最終回覆傳送至頻道；`channel` 預設為 `last`。
- `model` 會覆寫此次鉤子執行使用的 LLM（若已設定模型目錄，該模型必須獲准使用）。

</Accordion>

### Gmail 整合

- 內建 Gmail 預設集使用 `sessionKey: "hook:gmail:{{messages[0].id}}"`。
- 此逐訊息金鑰會隔離對話內容，而非工具或工作區存取。若沒有設定 `agentId` 的自訂對應，預設集會使用預設代理程式。
- 對於不受信任的收件匣，請將 Gmail 路由至專用的讀取代理程式，並透過[個別代理程式沙箱與工具原則](/zh-TW/tools/multi-agent-sandbox-tools)限制該代理程式。若讀取代理程式必須通知主要代理程式，請使用 [`tools.agentToAgent`](/zh-TW/gateway/config-tools#toolsagenttoagent) 限制交接。建議採用的威脅模型與模型等級請參閱[提示詞注入](/zh-TW/gateway/security#prompt-injection)。
- 若保留此逐訊息路由，請設定 `hooks.allowRequestSessionKey: true`，並限制 `hooks.allowedSessionKeyPrefixes` 以符合 Gmail 命名空間，例如 `["hook:", "hook:gmail:"]`。
- 若需要 `hooks.allowRequestSessionKey: false`，請以靜態 `sessionKey` 覆寫預設集，而非使用範本化的預設值。

```json5
{
  hooks: {
    gmail: {
      account: "openclaw@gmail.com",
      topic: "projects/<project-id>/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,
      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

- 設定後，閘道會在啟動時自動啟動 `gog gmail watch serve`。設定 `OPENCLAW_SKIP_GMAIL_WATCHER=1` 即可停用。
- 請勿在閘道旁另外執行 `gog gmail watch serve`。

---

## Canvas 外掛主機

```json5
{
  plugins: {
    entries: {
      canvas: {
        config: {
          host: {
            root: "~/.openclaw/workspace/canvas",
            liveReload: true,
            // enabled: false, // 或 OPENCLAW_SKIP_CANVAS_HOST=1
          },
        },
      },
    },
  },
}
```

- 透過閘道連接埠下的 HTTP 提供代理程式可編輯的 HTML/CSS/JS 與 A2UI：
  - `http://<gateway-host>:<gateway.port>/__openclaw__/canvas/`
  - `http://<gateway-host>:<gateway.port>/__openclaw__/a2ui/`
- 僅限本機：保留 `gateway.bind: "loopback"`（預設）。
- 非迴路位址繫結：Canvas 路由與其他閘道 HTTP 介面一樣，需要閘道驗證（權杖／密碼／受信任的 Proxy）。
- 節點 WebView 通常不會傳送驗證標頭；節點完成配對並連線後，閘道會公告供 Canvas/A2UI 存取使用、限定於該節點的能力 URL。
- 能力 URL 會繫結至作用中的節點 WS 工作階段，並會迅速到期。不使用以 IP 為基礎的回復機制。
- 將即時重新載入用戶端注入所提供的 HTML。
- 空白時會自動建立初始 `index.html`。
- 也會在 `/__openclaw__/a2ui/` 提供 A2UI。
- 變更後必須重新啟動閘道。
- 大型目錄或發生 `EMFILE` 錯誤時，請停用即時重新載入。

---

## 探索

### mDNS (Bonjour)

```json5
{
  discovery: {
    mdns: {
      mode: "minimal", // minimal | full | off
    },
  },
}
```

- `minimal`（預設）：從 TXT 記錄省略 `cliPath` + `sshPort`。
- `full`：包含 `cliPath` + `sshPort`；區域網路多點傳播公告仍須啟用隨附的 `bonjour` 外掛。
- `off`：不變更外掛啟用狀態，但抑制區域網路多點傳播公告。
- 隨附的 `bonjour` 外掛會在 macOS 主機上自動啟動；在 Linux、Windows 和容器化閘道部署上則須選擇性啟用。
- 若系統主機名稱是有效的 DNS 標籤，主機名稱預設使用該名稱，否則回復為 `openclaw`。可使用 `OPENCLAW_MDNS_HOSTNAME` 覆寫。
- `OPENCLAW_DISABLE_BONJOUR=1` 會完全停用 mDNS 公告，並覆寫 `discovery.mdns.mode`。

### 廣域 (DNS-SD)

```json5
{
  discovery: {
    wideArea: { enabled: true },
  },
}
```

在 `~/.openclaw/dns/` 下寫入單點傳播 DNS-SD 區域。若要進行跨網路探索，請搭配 DNS 伺服器（建議使用 CoreDNS）+ Tailscale 分割 DNS。

設定：`openclaw dns setup --apply`。

---

## 環境

### `env`（內嵌環境變數）

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

- 只有在程序環境中缺少該鍵時，才會套用內嵌環境變數。
- `.env` 檔案：CWD `.env` + `~/.openclaw/.env`（兩者都不會覆寫現有變數）。
- `shellEnv`：從你的登入 Shell 設定檔匯入缺少的預期鍵。
- 完整優先順序請參閱[環境](/zh-TW/help/environment)。

### 環境變數替換

使用 `${VAR_NAME}` 在任何設定字串中參照環境變數：

```json5
{
  gateway: {
    auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" },
  },
}
```

- 只會比對大寫名稱：`[A-Z_][A-Z0-9_]*`。
- 缺少或空白的變數會在載入設定時擲回錯誤。
- 使用 `$${VAR}` 跳脫，以表示字面值 `${VAR}`。
- 可搭配 `$include` 使用。

---

## 密鑰

密鑰參照是附加功能：純文字值仍可使用。

### `SecretRef`

使用以下單一物件形式：

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

驗證：

- `provider` 模式：`^[a-z][a-z0-9_-]{0,63}$`
- `source: "env"` ID 模式：`^[A-Z][A-Z0-9_]{0,127}$`
- `source: "file"` ID：絕對 JSON 指標（例如 `"/providers/openai/apiKey"`）
- `source: "exec"` ID 模式：`^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$`（支援 AWS 形式的 `secret#json_key` 選擇器）
- `source: "exec"` ID 不得包含 `.` 或 `..` 斜線分隔路徑區段（例如系統會拒絕 `a/../b`）

### 支援的認證資訊介面

- 標準矩陣：[SecretRef 認證資訊介面](/zh-TW/reference/secretref-credential-surface)
- `secrets apply` 以支援的 `openclaw.json` 認證資訊路徑為目標。
- `auth-profiles.json` 參照會納入執行階段解析與稽核涵蓋範圍。

### 密鑰提供者設定

```json5
{
  secrets: {
    providers: {
      default: { source: "env" }, // 選用的明確環境提供者
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",
        timeoutMs: 5000,
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        passEnv: ["PATH", "VAULT_ADDR"],
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

注意事項：

- `file` 提供者支援 `mode: "json"` 和 `mode: "singleValue"`（在 singleValue 模式中，`id` 必須是 `"value"`）。
- 無法驗證 Windows ACL 時，檔案和執行提供者路徑會採取封閉式失敗。僅針對無法驗證的受信任路徑設定 `allowInsecurePath: true`。
- `exec` 提供者需要絕對 `command` 路徑，並透過 stdin/stdout 使用通訊協定承載資料。
- 預設會拒絕符號連結命令路徑。設定 `allowSymlinkCommand: true` 可允許符號連結路徑，同時驗證解析後的目標路徑。
- 若已設定 `trustedDirs`，受信任目錄檢查會套用至解析後的目標路徑。
- `exec` 子程序環境預設為最小化；請使用 `passEnv` 明確傳遞必要變數。
- 密鑰參照會在啟用時解析至記憶體內快照，之後要求路徑只會讀取該快照。
- 啟用期間會套用作用中介面篩選：已啟用介面上未解析的參照會導致啟動／重新載入失敗，非作用中介面則會略過並提供診斷資訊。

---

## 驗證儲存空間

```json5
{
  auth: {
    profiles: {
      "anthropic:default": { provider: "anthropic", mode: "api_key" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai:personal": { provider: "openai", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:default", "anthropic:work"],
      openai: ["openai:personal"],
    },
  },
}
```

- 每個代理程式的設定檔儲存於 `<agentDir>/auth-profiles.json`。
- `auth-profiles.json` 支援靜態認證資訊模式的值層級參照（`api_key` 使用 `keyRef`，`token` 使用 `tokenRef`）。
- 舊版扁平 `auth-profiles.json` 對應表（例如 `{ "provider": { "apiKey": "..." } }`）不是執行階段格式；`openclaw doctor --fix` 會將其重寫為標準 `provider:default` API 金鑰設定檔，並建立 `.legacy-flat.*.bak` 備份。
- OAuth 模式設定檔（`auth.profiles.<id>.mode = "oauth"`）不支援由 SecretRef 支援的驗證設定檔認證資訊。
- 靜態執行階段認證資訊來自記憶體中已解析的快照；發現舊版靜態 `auth.json` 項目時會將其清除。
- 舊版 OAuth 從 `~/.openclaw/credentials/oauth.json` 匯入。
- 請參閱 [OAuth](/zh-TW/concepts/oauth)。
- 機密資料的執行階段行為與 `audit/configure/apply` 工具：[機密資料管理](/zh-TW/gateway/secrets)。

---

## 稽核

```json5
{
  audit: {
    enabled: true,
    messages: "off", // off | direct | all
  },
}
```

閘道會將代理程式執行與工具動作的**僅含中繼資料**稽核事件記錄到共用狀態資料庫中。訊息生命週期中繼資料需另行選擇啟用。此帳本會儲存身分、時間資訊、工具名稱及正規化結果，但絕不儲存提示詞、訊息本文、工具引數、結果或原始錯誤文字。訊息資料列不會儲存原始平台帳號、對話、訊息和目標 ID。執行／工具工作階段金鑰仍可用於建立關聯，而其中本身可能包含平台帳號或對等端 ID。記錄會在 30 天後到期，且帳本上限為 100,000 筆資料列。使用 [`openclaw audit`](/zh-TW/cli/audit) 或[`audit.activity.list`](/zh-TW/gateway/protocol#audit-ledger-rpc) 閘道 RPC 查詢。完整資料模型、隱私語意及涵蓋範圍限制，請參閱[稽核歷程記錄](/zh-TW/gateway/audit)。

- `enabled`：記錄新的稽核事件（預設值：`true`）。帳本預設為啟用，因為只有在事件發生後才啟用的稽核軌跡無法解釋該事件。設定 `false` 後，閘道重新啟動時便會停止插入新事件；現有記錄在到期前仍可讀取。重新啟用後會從該時間點恢復記錄，不會回填中間的空缺。
- `messages`：訊息中繼資料範圍（預設值：`"off"`）。`"direct"` 僅記錄已知的直接對話。`"all"` 也會記錄群組、頻道及未知的對話種類。兩種模式都不包含內容，並會在可建立關聯時，以安裝環境本機的金鑰式假名取代原始識別碼。這些假名是關聯輔助工具，而非匿名化機制；狀態資料庫會儲存衍生金鑰，但 RPC 與命令列介面匯出內容不會包含該金鑰。

執行中的閘道會在啟動時擷取 `audit.enabled` 與 `audit.messages`；變更任一設定後請重新啟動。訊息涵蓋範圍目前包括抵達核心分派的已接受傳入訊息，以及每個抵達共用持久傳遞機制的原始邏輯傳出回覆承載資料所對應的一筆終止資料列。尚未涵蓋繞過這些共用邊界的外掛本機路徑與直接傳送路徑。具容量限制的背景寫入器採盡力而為方式，並非無損的法規遵循封存系統。

---

## 記錄

```json5
{
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty", // pretty | compact | json
    redactSensitive: "tools", // off | tools
    redactPatterns: ["\\bTOKEN\\b\\s*[=:]\\s*([\"']?)([^\\s\"']+)\\1"],
  },
}
```

- 預設記錄檔：`/tmp/openclaw/openclaw-YYYY-MM-DD.log`；具名設定檔使用 `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`。
- 設定 `logging.file` 以使用固定路徑。
- 當 `--verbose` 時，`consoleLevel` 會提升為 `debug`。
- `maxFileBytes`：輪替前使用中記錄檔的大小上限，以位元組為單位（正整數；預設值：`104857600` = 100 MB）。OpenClaw 會在使用中檔案旁保留最多五個編號封存檔。
- `redactSensitive`／`redactPatterns`：盡力遮罩主控台輸出、檔案記錄、OTLP 記錄資料，以及持久保存的工作階段逐字稿文字。`redactSensitive: "off"` 只會停用這項一般記錄／逐字稿政策；UI、工具及診斷安全介面仍會在輸出前遮蔽機密資料。

---

## 診斷

```json5
{
  diagnostics: {
    enabled: true,
    flags: ["telegram.*"],

    otel: {
      enabled: false,
      endpoint: "https://otel-collector.example.com:4318",
      tracesEndpoint: "https://traces.example.com/v1/traces",
      metricsEndpoint: "https://metrics.example.com/v1/metrics",
      logsEndpoint: "https://logs.example.com/v1/logs",
      protocol: "http/protobuf", // http/protobuf | grpc
      headers: { "x-tenant-id": "my-org" },
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: false,
      logsExporter: "otlp",
      sampleRate: 1.0,
      flushIntervalMs: 5000,
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },

    cacheTrace: {
      enabled: false,
      filePath: "~/.openclaw/logs/cache-trace.jsonl",
      includeMessages: true,
      includePrompt: true,
      includeSystem: true,
    },
  },
}
```

- `enabled`：檢測輸出的總開關（預設值：`true`）。
- `flags`：啟用目標式記錄輸出的旗標字串陣列（支援 `"telegram.*"` 或 `"*"` 等萬用字元）。
- `otel.enabled`：啟用 OpenTelemetry 匯出管線（預設值：`false`）。完整設定、訊號目錄及隱私模型，請參閱 [OpenTelemetry 匯出](/zh-TW/gateway/opentelemetry)。
- `otel.endpoint`：OTel 匯出的收集器 URL。
- `otel.tracesEndpoint`／`otel.metricsEndpoint`／`otel.logsEndpoint`：選用的訊號專用 OTLP 端點。設定後，只會針對該訊號覆寫 `otel.endpoint`。
- `otel.protocol`：`"http/protobuf"`（預設）或 `"grpc"`。
- `otel.headers`：隨 OTel 匯出要求傳送的額外 HTTP／gRPC 中繼資料標頭。
- `otel.serviceName`：資源屬性的服務名稱。
- `otel.traces`／`otel.metrics`／`otel.logs`：啟用追蹤、指標或記錄匯出。
- `otel.logsExporter`：記錄匯出目的地：`"otlp"`（預設）、每一行標準輸出輸出一個 JSON 物件的 `"stdout"`，或 `"both"`。
- `otel.sampleRate`：追蹤取樣率 `0`-`1`。
- `otel.flushIntervalMs`：定期遙測排清間隔，以毫秒為單位。
- `otel.captureContent`：選擇啟用 OTEL 範圍屬性的原始內容擷取。預設關閉。布林值 `true` 會擷取非系統訊息／工具內容；物件形式可讓你明確啟用 `inputMessages`、`outputMessages`、`toolInputs`、`toolOutputs`、`systemPrompt` 及 `toolDefinitions`。
- `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`：適用於最新實驗性 GenAI 推論範圍形狀的環境開關，包括 `{gen_ai.operation.name} {gen_ai.request.model}` 範圍名稱、`CLIENT` 範圍種類，以及以 `gen_ai.provider.name` 取代舊版 `gen_ai.system`。為維持相容性，範圍預設會保留 `openclaw.model.call` 與 `gen_ai.system`；GenAI 指標使用有界的語意屬性。
- `OPENCLAW_OTEL_PRELOADED=1`：適用於已註冊全域 OpenTelemetry SDK 之主機的環境開關。OpenClaw 接著會略過外掛所擁有之 SDK 的啟動／關閉程序，同時讓診斷接聽程式保持作用。
- `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`、`OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` 及 `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`：當對應的設定鍵未設定時，所使用的訊號專用端點環境變數。
- `cacheTrace.enabled`：記錄嵌入式執行的快取追蹤快照（預設值：`false`）。
- `cacheTrace.filePath`：快取追蹤 JSONL 的輸出路徑（預設值：`$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl`）。
- `cacheTrace.includeMessages`／`includePrompt`／`includeSystem`：控制快取追蹤輸出中包含的內容（全部預設為：`true`）。

---

## 更新

```json5
{
  update: {
    channel: "stable", // stable | extended-stable | beta | dev
    checkOnStart: true,

    auto: {
      enabled: false,
    },
  },
}
```

- `channel`：發行頻道 — `"stable"`、`"extended-stable"`、`"beta"` 或 `"dev"`。延伸穩定版僅適用於套件：前景命令負責安裝，而閘道可發出唯讀更新提示。
- `checkOnStart`：閘道啟動時檢查 npm 更新（預設值：`true`）。已儲存的延伸穩定版選擇使用相同的唯讀提示及 24 小時提示排程。
- `auto.enabled`：為穩定版與測試版套件安裝啟用背景自動更新（預設值：`false`）。延伸穩定版絕不會自動套用更新。

---

## ACP

```json5
{
  acp: {
    enabled: true,
    dispatch: { enabled: true },
    backend: "acpx",
    fallbacks: ["acpx-secondary"],
    defaultAgent: "main",
    allowedAgents: ["main", "ops"],
    stream: {
      repeatSuppression: true,
      deliveryMode: "live", // live | final_only
    },
  },
}
```

- `enabled`：全域 ACP 功能閘門（預設值：`true`；設定 `false` 可隱藏 ACP 分派與衍生操作介面）。
- `dispatch.enabled`：ACP 工作階段回合分派的獨立閘門（預設值：`true`）。設定 `false` 可在封鎖執行的同時保留 ACP 命令。
- `backend`：預設 ACP 執行階段後端 ID（必須符合已註冊的 ACP 執行階段外掛）。
  請先安裝後端外掛；如果已設定 `plugins.allow`，請加入後端外掛 ID（例如 `acpx`），否則 ACP 後端將不會載入。
- `fallbacks`：當主要後端在產生任何輸出前，因疑似暫時性錯誤（無法使用、受到速率限制、配額耗盡或過載）而提早失敗時，依序嘗試的備援 ACP 後端 ID 清單。每個項目都必須符合已註冊的 ACP 執行階段外掛後端。
- `defaultAgent`：衍生操作未指定明確目標時使用的備援 ACP 目標代理程式 ID。
- `allowedAgents`：ACP 執行階段工作階段所允許之代理程式 ID 的允許清單；空白表示沒有額外限制。
- `stream.repeatSuppression`：抑制每個回合中重複的狀態／工具行（預設值：`true`）。
- `stream.deliveryMode`：`"live"` 會以增量方式串流；`"final_only"` 會緩衝至回合終止事件。
- `stream.tagVisibility`：標籤名稱對應至串流事件布林可見度覆寫值的記錄。
- `runtime.installCommand`：啟動 ACP 執行階段環境時要執行的選用安裝命令。

---

## 精靈

命令列介面引導式設定流程（`onboard`、`configure`、`doctor`）的行為與中繼資料：

```json5
{
  wizard: {
    accessMode: "full",
    appRecommendations: true,
    lastRunAt: "2026-01-01T00:00:00.000Z",
    lastRunVersion: "2026.1.4",
    lastRunCommit: "abc1234",
    lastRunCommand: "configure",
    lastRunMode: "local",
    securityAcknowledgedAt: "2026-01-01T00:00:00.000Z",
  },
}
```

- `wizard.accessMode`：在引導式初始設定開始時選擇的探索同意設定。`"full"`（建議）讓設定程序自動尋找 AI 應用程式、金鑰與本機執行環境；`"guarded"` 則讓設定程序在尋找前詢問一次，並改為提供手動設定選項。

- `wizard.appRecommendations` 預設為 `true`。將其設為 `false`，可在引導式或傳統初始設定期間停用已安裝應用程式建議，並封鎖閘道的 `device.apps` 存取。節點主機仍須另外啟用預設關閉的已安裝應用程式共用旗標，才會公告該命令。

---

## 身分

請參閱[代理程式預設值](/zh-TW/gateway/config-agents#agent-defaults)下的 `agents.entries` 身分欄位。

---

## 橋接器（舊版，已移除）

目前的建置版本已不再包含 TCP 橋接器。節點會透過閘道 WebSocket 連線。`bridge.*` 金鑰已不再屬於設定結構描述的一部分（移除前驗證會失敗；`openclaw doctor --fix` 可移除未知金鑰）。

<Accordion title="舊版橋接器設定（歷史參考）">

```json
{
  "bridge": {
    "enabled": true,
    "port": 18790,
    "bind": "tailnet",
    "tls": {
      "enabled": true,
      "autoGenerate": true
    }
  }
}
```

</Accordion>

---

## 排程

```json5
{
  cron: {
    enabled: true,
    webhook: "https://example.invalid/legacy", // 已儲存 notify:true 工作的已淘汰備援
    webhookToken: "replace-with-dedicated-token", // 用於外送網路鉤子驗證的選用持有人權杖
    sessionRetention: "24h", // 持續時間字串或 false
  },
}
```

- `sessionRetention`：在清除 SQLite 工作階段資料列前，要保留已完成隔離排程執行工作階段的時間長度。也會控制已封存之已刪除排程文字記錄的清理。預設值：`24h`；設為 `false` 即可停用。
- 執行歷程會自動為每項工作保留最新的 2000 筆終止狀態資料列。遺失的資料列仍保有其 24 小時清理期限。
- `webhookToken`：用於排程網路鉤子 POST 傳送（`delivery.mode = "webhook"`）的持有人權杖；若省略，則不會傳送驗證標頭。
- `webhook`：已淘汰的舊版備援網路鉤子 URL（http/https），供 `openclaw doctor --fix` 遷移仍具有 `notify: true` 的已儲存工作；執行階段傳送使用每項工作的 `delivery.mode="webhook"` 加上 `delivery.to`，或在保留公告傳送時使用 `delivery.completionDestination`。

### `cron.failureAlert`

```json5
{
  cron: {
    failureAlert: {
      enabled: false,
      after: 3,
      cooldownMs: 3600000,
      includeSkipped: false,
      mode: "announce",
      accountId: "main",
    },
  },
}
```

- `enabled`：啟用排程工作的失敗警示（預設值：`false`）。
- `after`：觸發警示前的連續失敗次數（正整數，最小值：`1`）。
- `cooldownMs`：同一工作重複警示之間的最短毫秒數（非負整數）。
- `includeSkipped`：是否將連續略過的執行計入警示閾值（預設值：`false`）。略過的執行會分開追蹤，不影響執行錯誤的退避機制。
- `mode`：傳送模式——`"announce"` 透過頻道訊息傳送；`"webhook"` 發布至已設定的網路鉤子。
- `accountId`：選用的帳號或頻道 ID，用於限定警示傳送範圍。

### `cron.failureDestination`

```json5
{
  cron: {
    failureDestination: {
      mode: "announce",
      channel: "last",
      to: "channel:C1234567890",
      accountId: "main",
    },
  },
}
```

- 所有工作的排程失敗通知預設目的地。
- `mode`：`"announce"` 或 `"webhook"`；若有足夠的目標資料，則預設為 `"announce"`。
- `channel`：公告傳送的頻道覆寫。`"last"` 會重複使用最後已知的傳送頻道。
- `to`：明確的公告目標或網路鉤子 URL。網路鉤子模式必填。
- `accountId`：選用的傳送帳號覆寫。
- 每項工作的 `delivery.failureDestination` 會覆寫此全域預設值。
- 若全域與每項工作皆未設定失敗目的地，已透過 `announce` 傳送的工作會在失敗時退回使用該主要公告目標。
- `delivery.failureDestination` 僅支援 `sessionTarget="isolated"` 工作，除非該工作的主要 `delivery.mode` 為 `"webhook"`。

請參閱[排程工作](/zh-TW/automation/cron-jobs)。隔離的排程執行會以[背景工作](/zh-TW/automation/tasks)追蹤。

## 媒體模型範本變數

在 `tools.media.models[].args` 中展開的範本預留位置：

| 變數                        | 說明                                              |
| --------------------------- | ------------------------------------------------- |
| `{{Body}}`                  | 完整的傳入訊息本文                                |
| `{{RawBody}}`               | 原始本文（不含歷程／傳送者包裝）                  |
| `{{BodyStripped}}`          | 已移除群組提及的本文                              |
| `{{From}}`                  | 傳送者識別碼                                      |
| `{{To}}`                    | 目的地識別碼                                      |
| `{{MessageSid}}`            | 頻道訊息 ID                                       |
| `{{SessionId}}`             | 目前工作階段 UUID                                 |
| `{{IsNewSession}}`          | 建立新工作階段時的 `"true"`             |
| `{{AttachmentUrl}}`         | 目前附件 URL 或供應商參照                         |
| `{{AttachmentPath}}`        | 目前附件的本機路徑                                |
| `{{AttachmentContentType}}` | 目前附件的 MIME 內容類型                          |
| `{{AttachmentDir}}`         | 包含 `AttachmentPath` 的目錄                    |
| `{{AttachmentIndex}}`       | 從零開始的來源事實索引                            |
| `{{Transcript}}`            | 音訊逐字稿                                        |
| `{{Prompt}}`                | 已解析的命令列介面項目媒體提示                    |
| `{{MaxChars}}`              | 已解析的命令列介面項目最大輸出字元數              |
| `{{ChatType}}`              | `"direct"` 或 `"group"`          |
| `{{GroupSubject}}`          | 群組主旨（盡力而為）                              |
| `{{GroupMembers}}`          | 群組成員預覽（盡力而為）                          |
| `{{SenderName}}`            | 傳送者顯示名稱（盡力而為）                        |
| `{{SenderE164}}`            | 傳送者電話號碼（盡力而為）                        |
| `{{Provider}}`              | 供應商提示（whatsapp、telegram、discord 等）       |

舊版的 `{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}` 與 `{{MediaDir}}`
名稱在外掛 SDK 相容性期間仍可使用，但已淘汰。
新的設定應使用 `Attachment*` 變數。

---

## 設定引入（`$include`）

將設定拆分至多個檔案：

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: {
    $include: ["./clients/mueller.json5", "./clients/schmidt.json5"],
  },
}
```

**合併行為：**

- 單一檔案：取代包含它的物件。
- 檔案陣列：依序深度合併（後者覆寫前者）。
- 同層金鑰：在引入後合併（覆寫引入的值）。
- 巢狀引入：深度最多 10 層。
- 路徑：相對於執行引入的檔案解析，但必須維持在最上層設定目錄（`openclaw.json` 的 `dirname`）內。只有在解析結果仍位於該界線內時，才允許絕對路徑／`../` 形式。設定 `OPENCLAW_INCLUDE_ROOTS`（絕對路徑）可允許設定目錄外的其他根目錄。
- 限制：解析前後的路徑皆不得包含空位元組，且必須嚴格少於 4096 個字元；每個引入檔案的上限為 2 MB。
- 若 OpenClaw 所擁有的寫入只變更由單一檔案引入支援的一個最上層區段，會直接寫入該引入檔案。例如，`plugins install` 會更新 `plugins.json5` 中的 `plugins: { $include: "./plugins.json5" }`，並保持 `openclaw.json` 不變。
- 對於 OpenClaw 所擁有的寫入，根層引入、引入陣列及具有同層覆寫的引入皆為唯讀；這些寫入會以失敗關閉方式處理，而不會將設定扁平化。
- 錯誤：針對檔案遺失、剖析錯誤、循環引入、無效路徑格式及長度過長提供清楚訊息。

---

## 相關內容

- [設定](/zh-TW/gateway/configuration)
- [設定範例](/zh-TW/gateway/configuration-examples)
- [Doctor](/zh-TW/gateway/doctor)
