---
read_when:
    - 你想要從 Hermes 或其他代理系統遷移至 OpenClaw
    - 你正在新增由外掛擁有的遷移提供者
summary: '`openclaw migrate` 的命令列介面參考（從其他代理系統匯入狀態）'
title: 遷移
x-i18n:
    generated_at: "2026-07-26T08:19:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f492535019f8a69706ff918462ba74cf5d26e733d2e4e9493b3c76bd77f2584d
    source_path: cli/migrate.md
    workflow: 16
---

# `openclaw migrate`

透過外掛擁有的遷移提供者，從另一個代理系統匯入狀態。內建提供者涵蓋 Claude、Codex 命令列介面和 [Hermes](/zh-TW/install/migrating-hermes)；外掛可註冊其他提供者。

<Tip>
如需面向使用者的逐步指南，請參閱[從 Claude 遷移](/zh-TW/install/migrating-claude)和[從 Hermes 遷移](/zh-TW/install/migrating-hermes)。[遷移中心](/zh-TW/install/migrating)列出所有路徑。
</Tip>

## 命令

```bash
openclaw migrate list
openclaw migrate claude --dry-run
openclaw migrate codex --dry-run
openclaw migrate codex --skill gog-vault77-google-workspace
openclaw migrate codex --plugin google-calendar --dry-run
openclaw migrate codex --plugin google-calendar --verify-plugin-apps --dry-run
openclaw migrate hermes --dry-run
openclaw migrate hermes
openclaw migrate apply codex --yes --skill gog-vault77-google-workspace
openclaw migrate apply codex --yes --plugin google-calendar
openclaw migrate apply codex --yes
openclaw migrate apply claude --yes
openclaw migrate apply hermes --yes
openclaw migrate apply hermes --include-secrets --yes
openclaw onboard --flow import
openclaw onboard --import-from claude --import-source ~/.claude
openclaw onboard --import-from hermes --import-source ~/.hermes
```

執行 `openclaw migrate <provider>` 且未使用其他旗標時，會建立計畫、顯示預覽，並在套用前（於 TTY 中）提示確認。`openclaw migrate plan <provider>` 和 `openclaw migrate apply <provider>` 會使用相同旗標，將預覽和套用拆分為不同的子命令。

<ParamField path="<provider>" type="string">
  已註冊遷移提供者的名稱，例如 `hermes`。執行 `openclaw migrate list` 以查看已安裝的提供者。
</ParamField>
<ParamField path="--dry-run" type="boolean">
  建立計畫並結束，而不變更狀態。
</ParamField>
<ParamField path="--from <path>" type="string">
  覆寫來源狀態目錄。Hermes 會依循 `$HERMES_HOME` 和作用中的設定檔，接著使用平台預設值（`~/.hermes` 或 `%LOCALAPPDATA%\hermes`）。Codex 預設為 `~/.codex`（或 `$CODEX_HOME`），Claude 預設為 `~/.claude`。
</ParamField>
<ParamField path="--include-secrets" type="boolean">
  不提示即匯入支援的認證資訊。互動式套用會先詢問是否匯入偵測到的驗證認證資訊，預設選取「是」；非互動式 `--yes` 需要 `--include-secrets` 才會匯入。
</ParamField>
<ParamField path="--no-auth-credentials" type="boolean">
  略過驗證認證資訊匯入，包括互動式提示。
</ParamField>
<ParamField path="--overwrite" type="boolean">
  當計畫回報衝突時，允許套用程序取代現有目標。
</ParamField>
<ParamField path="--yes" type="boolean">
  略過確認提示。非互動模式下為必要項目。
</ParamField>
<ParamField path="--skill <name>" type="string">
  依 Skill 名稱或項目 ID 選取一個 Skill 複製項目。重複使用此旗標可遷移多個 Skill。省略時，互動式 Codex 遷移會顯示核取方塊選擇器，非互動式遷移則會保留所有已規劃的 Skill。
</ParamField>
<ParamField path="--plugin <name>" type="string">
  依外掛名稱或項目 ID 選取一個 Codex 外掛安裝項目。重複使用此旗標可遷移多個 Codex 外掛。省略時，互動式 Codex 遷移會顯示原生 Codex 外掛核取方塊選擇器，非互動式遷移則會保留所有已規劃的外掛。僅適用於由 Codex 應用程式伺服器清單發現、從來源安裝的 `openai-curated` Codex 外掛。
</ParamField>
<ParamField path="--verify-plugin-apps" type="boolean">
  僅限 Codex。在規劃原生外掛啟用前，強制重新遍歷來源 Codex 應用程式伺服器的 `app/list`。預設關閉，以保持遷移規劃快速。
</ParamField>
<ParamField path="--backup-output <path>" type="string">
  遷移前的備份封存檔路徑或目錄。會原樣傳遞給 `openclaw backup create`。
</ParamField>
<ParamField path="--no-backup" type="boolean">
  略過套用前備份。當本機 OpenClaw 狀態存在時，需要搭配 `--force`。
</ParamField>
<ParamField path="--force" type="boolean">
  當套用程序原本會拒絕略過備份時，必須與 `--no-backup` 一併使用。
</ParamField>
<ParamField path="--json" type="boolean">
  以 JSON 輸出計畫或套用結果。搭配 `--json` 且未使用 `--yes` 時，套用程序會輸出計畫，且不會變更狀態。
</ParamField>

## 安全模型

`openclaw migrate` 採預覽優先。

<AccordionGroup>
  <Accordion title="套用前預覽">
    提供者會在任何變更發生前傳回逐項計畫，包括衝突、略過的項目和敏感項目。JSON 計畫、套用輸出和遷移報告會遮蔽巢狀、看似機密的鍵，例如 API 金鑰、權杖、授權標頭、Cookie 和密碼。

    除非設定 `--yes`，否則 `openclaw migrate apply <provider>` 會預覽計畫，並在變更狀態前提示確認。在非互動模式下，套用需要 `--yes`。

  </Accordion>
  <Accordion title="備份">
    套用程序會在執行遷移前建立並驗證 OpenClaw 備份。如果本機尚不存在 OpenClaw 狀態，則略過備份步驟並繼續遷移。若要在狀態存在時略過備份，請同時傳入 `--no-backup` 和 `--force`。
  </Accordion>
  <Accordion title="衝突">
    當計畫存在衝突時，套用程序會拒絕繼續。檢閱計畫後，如果確實要取代現有目標，請使用 `--overwrite` 重新執行。提供者仍可能在遷移報告目錄中，針對遭覆寫的檔案建立項目層級的備份。
  </Accordion>
  <Accordion title="機密資訊">
    互動式套用會詢問是否匯入偵測到的驗證認證資訊，預設選取「是」。使用 `--no-auth-credentials` 可略過，或搭配 `--yes` 使用 `--include-secrets`，以進行無人值守的認證資訊匯入。
  </Accordion>
</AccordionGroup>

## Claude 提供者

內建 Claude 提供者預設會在 `~/.claude` 偵測 Claude Code 狀態。使用 `--from <path>` 可匯入指定的 Claude Code 主目錄或專案根目錄。

<Tip>
如需面向使用者的逐步指南，請參閱[從 Claude 遷移](/zh-TW/install/migrating-claude)。
</Tip>

### Claude 匯入的內容

- 來自 `~/.claude/projects/*/memory` 和使用者設定之 `autoMemoryDirectory` 的 Claude Code 自動記憶 Markdown，會複製到 `memory/imports/claude-code/` 下，以供索引式回憶。
- 將專案 `CLAUDE.md` 和 `.claude/CLAUDE.md` 匯入 OpenClaw 代理工作區（`AGENTS.md`）。
- 將使用者 `~/.claude/CLAUDE.md` 附加至工作區 `USER.md`。
- 來自專案 `.mcp.json`、Claude Code `~/.claude.json`（包括其各專案項目）和 Claude Desktop `claude_desktop_config.json` 的 MCP 伺服器定義。
- 包含 `SKILL.md` 的 Claude Skill 目錄（使用者 `~/.claude/skills` 和專案 `.claude/skills`）。
- Claude 命令 Markdown 檔案（使用者 `~/.claude/commands` 和專案 `.claude/commands`）會轉換為僅能手動叫用的 OpenClaw Skill。

### 封存與需手動檢閱的狀態

Claude 鉤子、權限、環境預設值、專案 `CLAUDE.local.md`、`.claude/rules`、使用者與專案 `agents/` 目錄，以及專案歷程記錄（`~/.claude` 下的 `projects`、`cache`、`plans`）會保留在遷移報告中，或回報為需手動檢閱的項目。OpenClaw 不會執行鉤子、複製廣泛的允許清單，也不會自動匯入 OAuth／Desktop 認證資訊狀態。

## Codex 提供者

內建 Codex 提供者預設會在 `~/.codex` 偵測 Codex 命令列介面狀態；若已設定該環境變數，則會在 `CODEX_HOME` 偵測。使用 `--from <path>` 可盤點指定的 Codex 主目錄。

移轉至 OpenClaw Codex 執行框架，且想刻意提升實用的個人 Codex 命令列介面資產時，請使用此提供者。本機 Codex 應用程式伺服器會使用每個代理各自的 `CODEX_HOME` 啟動，因此預設不會讀取你的個人 `~/.codex`。一般程序的 `HOME` 仍會繼承，因此 Codex 可看到共用的 `$HOME/.agents/*` Skill／外掛市集項目，子程序也能找到使用者主目錄中的設定和權杖。

在互動式終端中執行 `openclaw migrate codex`，會先預覽完整計畫，再於最終套用確認前開啟核取方塊選擇器。系統會先提示選取 Skill 複製項目。使用 `Toggle all on` 或 `Toggle all off` 可進行大量選取。按 Space 可切換資料列，按 Enter 則可啟用醒目顯示的資料列並繼續。已規劃的 Skill 預設勾選，有衝突的 Skill 預設不勾選，而 `Skip for now` 會略過本次執行的 Skill 複製，但仍會繼續進行外掛選取。若可遷移從來源安裝且經整理的 Codex 外掛，並且未提供 `--plugin`，遷移程序接著會依外掛名稱提示啟用原生 Codex 外掛。除非目標 OpenClaw Codex 外掛設定中已有該外掛，否則外掛項目預設勾選。現有的目標外掛預設不勾選，並顯示類似 `conflict: plugin exists` 的衝突提示；選擇 `Toggle all off` 可在本次執行中不遷移任何原生 Codex 外掛，或選擇 `Skip for now` 在套用前停止。

若要執行指令碼化或精確的作業，請明確選取一或多個 Skill 或外掛：

```bash
openclaw migrate codex --dry-run --skill gog-vault77-google-workspace
openclaw migrate apply codex --yes --skill gog-vault77-google-workspace
openclaw migrate codex --dry-run --plugin google-calendar
openclaw migrate apply codex --yes --plugin google-calendar
```

### Codex 匯入的內容

- 來自 `$CODEX_HOME/memories` 的整合 Codex `MEMORY.md` 和 `memory_summary.md`，會複製到 `memory/imports/codex/` 下，以供索引式回憶。不會匯入原始 rollout 記憶。
- `$CODEX_HOME/skills` 下的 Codex 命令列介面 Skill 目錄，但不包括 Codex 的 `.system` 快取。
- `$HOME/.agents/skills` 下的個人 AgentSkills，會複製到目前的 OpenClaw 代理工作區，以供各代理個別擁有。
- 透過 Codex 應用程式伺服器 `plugin/list` 發現、從來源安裝的 `openai-curated` Codex 外掛。規劃程序會讀取每個已啟用且已安裝外掛的 `plugin/read`。

應用程式支援的外掛遷移有額外的門檻：

- 應用程式支援的外掛要求來源 Codex 應用程式伺服器帳戶必須是 ChatGPT 訂閱帳戶。非 ChatGPT 帳戶或缺少帳戶的回應會以 `codex_subscription_required` 略過。
- 依預設，遷移程序不會呼叫來源 `app/list`，因此通過帳戶門檻的應用程式支援外掛會在未驗證來源應用程式可存取性的情況下進行規劃，而帳戶查詢的傳輸失敗會以 `codex_account_unavailable` 略過。
- 傳入 `--verify-plugin-apps` 可強制建立全新的來源 `app/list` 快照，並要求每個所擁有的應用程式都存在、已啟用且可存取，才會規劃原生啟用。在此模式下，帳戶查詢的傳輸失敗會轉而進行來源應用程式清單驗證。快照只會在目前程序的記憶體中保留；絕不會寫入遷移輸出或目標設定。

已停用的外掛、無法讀取的外掛詳細資料、受訂閱限制的來源帳戶，以及（設定 `--verify-plugin-apps` 時）不存在、已停用或無法存取的應用程式，會成為附有型別化原因、需手動處理的略過項目，而不會成為目標設定項目。套用程序會為每個選取且符合資格的外掛呼叫應用程式伺服器 `plugin/install`，即使目標應用程式伺服器已回報該外掛已安裝並啟用亦同。遷移後的 Codex 外掛僅能用於選取原生 Codex 執行框架的工作階段；它們不會提供給 OpenClaw 提供者執行、ACP 對話繫結或其他執行框架使用。

### 需手動檢閱的 Codex 狀態

Codex `config.toml`、原生 `hooks/hooks.json`、非策展市集、並非透過原始碼安裝之策展外掛的快取外掛套件，以及未通過來源訂閱閘門的原始碼安裝外掛，都不會自動啟用。設定 `--verify-plugin-apps` 時，也會略過未通過來源應用程式清單閘門的外掛。所有這些項目都會複製至移轉報告或在其中列出，以供手動審查。

對於已移轉、透過原始碼安裝的策展外掛，套用以下寫入：

- `plugins.entries.codex.enabled: true`
- `plugins.entries.codex.config.codexPlugins.enabled: true`
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions: true`
- 每個選定的外掛各有一個明確的外掛項目，包含 `marketplaceName: "openai-curated"` 和 `pluginName`

移轉絕不會寫入 `plugins["*"]`，也絕不會儲存本機市集快取路徑。

略過的外掛不會寫入目標設定。來源端訂閱失敗會以具型別的原因列於手動處理項目：`codex_subscription_required`、`codex_account_unavailable`、`plugin_disabled` 或 `plugin_read_unavailable`。使用 `--verify-plugin-apps` 時，來源應用程式清單失敗也可能顯示為 `app_inaccessible`、`app_disabled`、`app_missing` 或 `app_inventory_unavailable`。目標端需要驗證的安裝會在受影響的外掛項目中列出 `status: "skipped"`、`reason: "auth_required"` 與經過清理的應用程式識別碼；其明確設定項目會以停用狀態寫入，直到你重新授權並啟用它們。其他安裝失敗則是項目範圍的 `error` 結果。

如果規劃期間無法取得 Codex 應用程式伺服器外掛清單，移轉會改用快取套件諮詢項目，而不會使整個移轉失敗。

## Hermes 提供者

內建的 Hermes 提供者會依循 `$HERMES_HOME` 與作用中的設定檔，接著使用平台預設值（`~/.hermes` 或 `%LOCALAPPDATA%\hermes`）。使用 `--from <path>` 覆寫探索行為。

### Hermes 匯入的內容

- 來自 `config.yaml` 的預設模型設定。
- 來自 `model`、`providers` 和 `custom_providers` 的已設定模型提供者與自訂 OpenAI 相容端點。
- 來自 `mcp_servers` 或 `mcp.servers` 的 MCP 伺服器定義。精確的 OpenClaw 對應涵蓋預設的可串流 HTTP 路由、OAuth 範圍、布林值 TLS 驗證、分離的用戶端憑證／金鑰路徑，以及 Hermes 原生／資源／提示詞工具政策。不支援且僅限 Hermes 的執行階段或認證資訊欄位會列出以供手動審查。
- 將 `SOUL.md` 和 `AGENTS.md` 匯入 OpenClaw 代理程式工作區。
- 將 `memories/MEMORY.md` 和 `memories/USER.md` 附加至工作區記憶檔案。
  僅限記憶的介面（新手設定記憶頁面與 Control UI Memory
  匯入頁面）則會將這些檔案複製到 `memory/imports/hermes/` 下，
  以供建立索引後回憶，而不會變更現有的工作區記憶。
- OpenClaw 檔案記憶的記憶設定預設值，以及 Honcho 等外部記憶提供者的封存或手動審查項目。
- 位於 `skills/` 下任何位置且包含 `SKILL.md` 檔案的 Skills；巢狀 Skills 會攤平至工作區 Skills 目錄。
- 來自 `skills.config` 的各 Skills 設定值。
- 在接受互動式認證資訊移轉時，或設定 `--include-secrets` 時，匯入目前的 Hermes OpenAI Codex OAuth 認證資訊與 OpenCode OpenAI OAuth 認證資訊。請勿讓 Hermes 與 OpenClaw 使用同一個匯入的重新整理授權。
- 在接受互動式認證資訊移轉時，或設定 `--include-secrets` 時，匯入 Hermes `.env` 與 OpenCode `auth.json` 中支援的 API 金鑰和權杖。

### 支援的 `.env` 金鑰

`AI_GATEWAY_API_KEY`、`ALIBABA_API_KEY`、`ANTHROPIC_API_KEY`、`ARCEEAI_API_KEY`、`CEREBRAS_API_KEY`、`CHUTES_API_KEY`、`CLOUDFLARE_AI_GATEWAY_API_KEY`、`COPILOT_GITHUB_TOKEN`、`DASHSCOPE_API_KEY`、`DEEPINFRA_API_KEY`、`DEEPSEEK_API_KEY`、`FIREWORKS_API_KEY`、`GEMINI_API_KEY`、`GH_TOKEN`、`GITHUB_TOKEN`、`GLM_API_KEY`、`GOOGLE_API_KEY`、`GROQ_API_KEY`、`HF_TOKEN`、`HUGGINGFACE_HUB_TOKEN`、`KILOCODE_API_KEY`、`KIMICODE_API_KEY`、`KIMI_API_KEY`、`KIMI_CODING_API_KEY`、`MINIMAX_API_KEY`、`MINIMAX_CODING_API_KEY`、`MISTRAL_API_KEY`、`MODELSTUDIO_API_KEY`、`MOONSHOT_API_KEY`、`NVIDIA_API_KEY`、`OPENAI_API_KEY`、`OPENCODE_API_KEY`、`OPENCODE_GO_API_KEY`、`OPENCODE_ZEN_API_KEY`、`OPENROUTER_API_KEY`、`QIANFAN_API_KEY`、`QWEN_API_KEY`、`TOGETHER_API_KEY`、`VENICE_API_KEY`、`XAI_API_KEY`、`XIAOMI_API_KEY`、`ZAI_API_KEY`、`Z_AI_API_KEY`。

### 僅封存狀態

OpenClaw 無法安全解讀的 Hermes 狀態會複製到移轉報告中供手動審查，但不會載入即時 OpenClaw 設定或認證資訊。這包括 `plugins/`、`sessions/`、`logs/`、`cron/`、`mcp-tokens/`、`plans/`、`workspace/`、`skins/`、`kanban/`、配對／平台狀態、閘道路由／程序狀態，以及偵測到的 Hermes SQLite 資料庫。

### 套用後

```bash
openclaw doctor
```

## 外掛合約

移轉來源是外掛。外掛會在 `openclaw.plugin.json` 中宣告其提供者 ID：

```json
{
  "contracts": {
    "migrationProviders": ["hermes"]
  }
}
```

執行階段中，外掛會呼叫 `api.registerMigrationProvider(...)`。提供者會實作 `detect`、`plan` 和 `apply`。核心負責命令列介面協調、備份政策、提示、JSON 輸出及衝突預檢。核心會將審查後的計畫傳入 `apply(ctx, plan)`；僅為相容性，提供者可在缺少該引數時重建計畫。移轉項目可為新手設定必須延後至已暫存的本機資料持久發布後，才能執行的外部啟用效果設定 `applyPhase: "after-promotion"`。這些提供者必須宣告 `deferredApply: { retrySafe: true }`，並確保每個延後效果都能在程序中斷後安全重播；新手設定會拒絕未宣告的延後效果。具冪等性的無操作應傳回含有 `deferredCompletion: true` 的非變更項目，讓復原程序可將其記錄為完成。獨立的 `openclaw migrate` 仍會透過其一般、具備備份支援的流程套用完整計畫。

提供者外掛可使用 `openclaw/plugin-sdk/migration` 建構項目與統計摘要，並使用 `openclaw/plugin-sdk/migration-runtime` 進行可感知衝突的檔案複製、僅封存報告複製、快取設定執行階段包裝，以及產生移轉報告。

## 新手設定整合

當提供者偵測到已知來源時，新手設定可以提供移轉選項。`openclaw onboard --flow import` 和 `openclaw setup --wizard --import-from hermes` 都會使用相同的外掛移轉提供者，且仍會在套用前顯示預覽。與獨立移轉不同，全新目標的新手設定路徑會暫存本機成品與匯入的認證資訊、在暫存區內驗證或修復匯入的推論設定，然後先提升工作區和代理程式狀態，再提交設定。模式為 `0600` 的提升日誌可讓下一次執行完成或回復中斷的發布，包括任何延後的外部啟用，而不會重播匯入的本機資料。

<Note>
新手設定匯入需要全新的 OpenClaw 設定。如果已有本機狀態，請先重設設定、認證資訊、工作階段與工作區。針對現有設定的備份後覆寫或合併匯入功能由功能閘門控管。
</Note>

## 相關內容

- [從 Hermes 移轉](/zh-TW/install/migrating-hermes)：面向使用者的操作說明。
- [從 Claude 移轉](/zh-TW/install/migrating-claude)：面向使用者的操作說明。
- [移轉](/zh-TW/install/migrating)：將 OpenClaw 移至新機器。
- [Doctor](/zh-TW/gateway/doctor)：套用移轉後的健康狀態檢查。
- [外掛](/zh-TW/tools/plugin)：外掛安裝與註冊。
