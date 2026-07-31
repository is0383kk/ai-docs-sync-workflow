---
read_when:
    - 編輯系統提示文字、工具清單或時間／心跳偵測區段
    - 變更工作區啟動程序或 Skills 注入行為
summary: OpenClaw 系統提示包含的內容及其組合方式
title: 系統提示詞
x-i18n:
    generated_at: "2026-07-26T07:40:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 669fbc6f21a82a2c3c067d2ff3a6365acb3316460a85f2db165b7ad49ce79f70
    source_path: concepts/system-prompt.md
    workflow: 16
---

OpenClaw 會為每次代理程式執行建構自己的系統提示；不存在執行階段預設提示。

組裝分為三層：

- `buildAgentSystemPrompt` 會從明確的輸入轉譯提示。它維持為純轉譯器，不會直接讀取全域設定。
- `resolveAgentSystemPromptConfig` 會為特定代理程式解析由設定支援的提示調整項目（擁有者顯示、TTS 提示、模型別名、記憶引用模式、子代理程式委派模式）。
- 執行階段配接器（內嵌、命令列介面、命令／匯出預覽、壓縮）會收集即時資訊（工具、沙箱狀態、頻道功能、情境檔案、供應商提示貢獻），並呼叫已設定的提示外觀介面。

這能讓匯出／偵錯提示介面與即時執行保持一致，而不會將每個執行階段細節都塞入單一龐大的建構器。

供應商外掛可以提供可感知快取的指引，而不取代由 OpenClaw 擁有的提示。供應商執行階段可以：

- 取代三個具名核心區段之一：`interaction_style`、`tool_call_style`、`execution_bias`
- 在提示快取邊界上方注入**穩定前綴**
- 在提示快取邊界下方注入**動態後綴**

使用供應商擁有的貢獻進行模型系列專屬調校。僅將舊版 `before_prompt_build` 掛鉤保留給相容性或真正的全域提示變更。

隨附的 OpenAI/Codex GPT-5 系列覆寫層（`resolveGpt5SystemPromptContribution`）會使用此機制：一份 `stablePrefix` 行為契約（執行政策、工具紀律、輸出契約、完成契約），以及可選用的 `interaction_style` 覆寫，以提供更友善的語氣。它適用於透過 OpenAI 或 Codex 外掛路由的任何 `gpt-5*` 模型 ID，並由 `agents.defaults.promptOverlays.gpt5.personality`（`"friendly"`/`"on"` 或 `"off"`）控制。

## 結構

提示相當精簡，包含固定區段：

- **工具**：結構化工具為真實資料來源的提醒，以及執行階段工具使用指引。啟用實驗性 `update_plan` 工具（`tools.experimental.planTool`）時，其自身的工具說明會補充：僅將它用於非瑣碎的多步驟工作、最多僅讓一個步驟處於 `in_progress`，並在簡單的單步驟工作中略過它。
- **執行傾向**：針對可採取行動的要求在目前回合內執行、持續到完成或遭到阻擋、從不理想的工具結果中復原、即時檢查可變狀態，並在完成前驗證。
- **安全性**：避免追求權力的行為或繞過監督的簡短防護提醒。
- **Skills**（可用時）：告訴模型如何視需要載入 Skills 指示。
- **OpenClaw 控制**：設定／重新啟動工作應優先使用 `gateway` 工具；不要杜撰命令列介面命令。
- **OpenClaw 自我更新**：使用 `config.schema.lookup` 安全檢查設定、使用 `config.patch` 修補、使用 `config.apply` 取代完整設定，並且僅在使用者明確要求時執行 `update.run`。面向代理程式的 `gateway` 工具會拒絕重寫 `tools.exec.mode`。
- **工作區**：工作目錄（`agents.defaults.workspace`）。
- **文件**：本機文件／原始碼路徑，以及應在何時閱讀。
- **工作區檔案（已注入）**：註明下方已包含啟動載入檔案。
- **沙箱**（啟用時）：沙箱化執行階段、沙箱路徑、提高權限執行的可用性。
- **目前日期與時間**：僅含時區（快取穩定；即時時鐘來自 `session_status`）。
- **助理輸出指示**：精簡的附件、語音訊息與回覆標籤語法。
- **心跳偵測**：為預設代理程式啟用心跳偵測時的心跳偵測提示與確認行為。
- **執行階段**：主機、作業系統、節點、模型、儲存庫根目錄（偵測到時）、思考層級（單行）。
- **推理**：目前的可見性層級，以及 `/reasoning` 切換提示。

大量穩定內容（包括**專案情境**）會保留在內部提示快取邊界上方。每回合易變動的區段（Control UI 內嵌指引、**傳訊**、**語音**、**群組聊天情境**、**反應**、**心跳偵測**、**執行階段**）會附加在該邊界下方，讓具有前綴快取的本機後端能跨頻道回合重複使用穩定的工作區前綴。當接受的結構描述已包含該執行階段細節時，工具說明應避免嵌入目前的頻道名稱。

工具也包含長時間執行工作的指引：

- 使用排程處理未來的後續工作（`check back later`、提醒、週期性工作），而不是使用 `exec` 睡眠迴圈、`yieldMs` 延遲技巧或重複 `process` 輪詢
- 僅將 `exec` / `process` 用於立即啟動並持續在背景執行的命令
- 啟用自動完成喚醒時，只需啟動命令一次，並依賴推送式喚醒路徑
- 使用 `process` 取得執行中命令的記錄、狀態、輸入或進行介入
- 對於較大型的工作，優先使用 `sessions_spawn`；子代理程式完成採用推送機制，並會自動向要求者發出通知
- 不要為了等待完成而在迴圈中輪詢 `subagents list` / `sessions_list`

`agents.defaults.subagents.delegationMode`（預設為 `"suggest"`）可以強化此行為。`"prefer"` 會加入專用的**子代理程式委派**區段，指示主要代理程式扮演回應迅速的協調者，並將任何比直接回覆更複雜的工作透過 `sessions_spawn` 推送出去。這僅作用於提示；工具政策仍會控制 `sessions_spawn` 是否可用。

系統提示中的安全防護準則屬於建議，而非強制執行。請使用工具政策、執行核准、沙箱及頻道允許清單進行強制控管；依設計，操作人員可以停用提示防護準則。

在具有原生核准卡片／按鈕的頻道中，提示會告知代理程式優先依賴該 UI，並僅在工具結果表示聊天核准不可用，或手動核准是唯一途徑時，才包含手動 `/approve` 命令。

## 提示模式

OpenClaw 會為子代理程式轉譯較小的系統提示。執行階段會為每次執行設定一個 `promptMode`（並非面向使用者的設定）：

- `full`（預設）：上述所有區段。
- `minimal`：用於子代理程式；省略記憶提示區段（隨附為**記憶回想**）、**OpenClaw 自我更新**、**模型別名**、**使用者身分**、**助理輸出指示**、**傳訊**、**靜默回覆**及**心跳偵測**。工具、**安全性**、**Skills**（若有提供）、工作區、沙箱、目前日期與時間（已知時）、執行階段及注入的情境仍然可用。
- `none`：僅傳回基本身分行。

在 `promptMode=minimal` 下，額外注入的提示會標示為**子代理程式情境**，而非**群組聊天情境**。

對於頻道自動回覆執行，當直接、群組或僅訊息工具情境已擁有可見回覆契約時，OpenClaw 會省略一般的**靜默回覆**區段。只有舊版自動群組／頻道模式會顯示 `NO_REPLY`；直接聊天和僅訊息工具的回覆會略過靜默權杖指引。

## 提示快照

OpenClaw 會在 `test/fixtures/agents/prompt-snapshots/codex-runtime-happy-path/` 下保留已提交的提示快照，用於 Codex 執行階段的理想路徑。它們會轉譯所選的應用程式伺服器執行緒／回合參數，以及為 Telegram 直接聊天、Discord 群組和心跳偵測回合重建的模型綁定提示層堆疊：固定的 Codex `gpt-5.5` 模型提示固定內容、Codex 理想路徑權限開發者文字、OpenClaw 開發者指示、OpenClaw 提供時的回合範圍協作模式指示、使用者回合輸入，以及動態工具規格的參照。

使用 `pnpm prompt:snapshots:sync-codex-model` 重新整理固定的 Codex 模型提示固定內容。依預設，它會依序尋找 `$CODEX_HOME/models_cache.json`、`~/.codex/models_cache.json`，接著尋找維護者簽出慣例 `~/code/codex/codex-rs/models-manager/models.json`；若都不存在，它會結束且不變更已提交的固定內容。傳入 `--catalog <path>`，即可從特定的 `models_cache.json` 或 `models.json` 檔案重新整理。

這些快照並非逐位元組的原始 OpenAI 要求擷取。OpenClaw 傳送執行緒與回合參數後，Codex 可以加入由執行階段擁有的工作區情境（`AGENTS.md`、環境情境、記憶、應用程式／外掛指示、內建的預設協作模式指示）。

使用 `pnpm prompt:snapshots:gen` 重新產生；使用 `pnpm prompt:snapshots:check` 驗證偏移。CI 會連同額外邊界分片一起執行偏移檢查，因此提示變更與快照更新會在同一個 PR 中一併提交。

## 工作區啟動載入注入

啟動載入檔案會從作用中的工作區解析，並路由到符合其生命週期的提示介面：

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`（僅限全新工作區）
- `MEMORY.md`（存在時）

在原生 Codex 測試框架上，OpenClaw 會避免在每個使用者回合重複穩定的工作區檔案。Codex 會透過自身的專案文件探索載入 `AGENTS.md`。`TOOLS.md` 會以繼承的 Codex 開發者指示轉送。`SOUL.md`、`IDENTITY.md` 和 `USER.md` 會以回合範圍的協作開發者指示轉送，因此原生 Codex 子代理程式不會繼承它們。`HEARTBEAT.md` 內容不會直接注入；當檔案存在且非空白時，心跳偵測回合會取得一則指向該檔案的協作模式註記。`MEMORY.md` 內容也不會貼入每個原生 Codex 回合：當工作區可使用記憶工具時，Codex 回合會取得一則簡短的工作區記憶註記，將模型導向 `memory_search` 或 `memory_get`。如果工具已停用、記憶搜尋不可用，或作用中工作區與代理程式記憶工作區不同，`MEMORY.md` 會回復為一般的有限回合情境路徑。`BOOTSTRAP.md` 會維持一般的回合情境角色。

在非 Codex 測試框架上，啟動載入檔案會依其現有門檻組合進 OpenClaw 提示。當預設代理程式停用心跳偵測，或 `agents.defaults.heartbeat.includeSystemPromptSection` 為 false 時，一般執行會省略 `HEARTBEAT.md`。注入的檔案應保持精簡，尤其是非 Codex 的 `MEMORY.md`：它應維持為經整理的長期摘要，詳細的每日筆記則放在 `memory/*.md`，並可視需要透過 `memory_search` / `memory_get` 擷取。過大的非 Codex `MEMORY.md` 檔案會增加提示用量，且可能依下列啟動載入檔案限制而僅注入部分內容。

<Note>
`memory/*.md` 每日檔案**不屬於**一般啟動載入的專案情境。在一般回合中，會視需要透過 `memory_search` / `memory_get` 存取，因此除非模型明確讀取，否則不會占用情境視窗。單純的 `/new` 與 `/reset` 回合屬於例外：執行階段可為該第一個回合預先加入最近的每日記憶，作為一次性的啟動情境區塊。
</Note>

大型檔案會截斷並附上標記：

| 限制                                         | 設定鍵                                             | 預設值   |
| -------------------------------------------- | -------------------------------------------------- | -------- |
| 每個檔案的字元上限                           | `agents.defaults.bootstrapMaxChars`                | 20000    |
| 所有檔案合計                                 | `agents.defaults.bootstrapTotalMaxChars`           | 60000    |
| 截斷警告（`off`\|`once`\|`always`） | `agents.defaults.bootstrapPromptTruncationWarning` | `always` |

缺少的檔案會注入簡短的檔案遺失標記。詳細的原始／注入計數則保留在診斷資訊中，例如 `/context`、`/status`、doctor 和日誌。

對於記憶檔案，截斷並不代表資料遺失：磁碟上的檔案會保持完整。在原生 Codex 上，若記憶工具可用，會依需要透過記憶工具讀取 `MEMORY.md`；否則會使用有界限的提示詞後備機制。在其他執行框架中，模型只會看到縮短後的注入副本，直到它直接讀取或搜尋記憶為止。如果 `MEMORY.md` 反覆遭到截斷，請將其提煉成更短且持久的摘要、把詳細歷程移至 `memory/*.md`，或刻意提高啟動限制。

子代理程式工作階段只會注入 `AGENTS.md` 和 `TOOLS.md`（其他啟動檔案會被篩除，以縮小子代理程式的情境）。

內部鉤子可透過 `agent:bootstrap` 事件攔截此步驟，以修改或取代注入的啟動檔案（例如將 `SOUL.md` 替換為其他角色設定）。

若要讓語氣不那麼制式，請從 [SOUL.md 個性指南](/zh-TW/concepts/soul)開始。

若要檢查每個注入檔案的占用量（原始與注入、截斷、工具結構描述額外負擔），請使用 `/context list` 或 `/context detail`。請參閱[情境](/zh-TW/concepts/context)。

## 時間處理

只有在已知使用者時區時，才會顯示**目前日期與時間**區段，而且只包含**時區**（不含動態時鐘或時間格式），以維持提示詞快取穩定。

當代理程式需要目前時間時，請使用 `session_status`；其狀態卡會包含時間戳記行。同一工具亦可選擇設定每個工作階段的模型覆寫（`model=default` 可將其清除）。

設定方式：

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat`（`auto` | `12` | `24`）

如需完整的行為詳細資訊，請參閱[時區](/zh-TW/concepts/timezone)和[日期與時間](/zh-TW/date-time)。

## Skills

當有符合資格的 Skills 時，OpenClaw 會注入精簡的 `<available_skills>` 清單（`formatSkillsForPrompt`），其中包含每個 Skill 的**檔案路徑**和由內容衍生的 `<version>sha256:...</version>` 標記。提示詞會指示模型使用 `read` 載入所列位置（工作區、受管理或隨附）的 SKILL.md，並在某項 Skill 的 `<version>` 與前一回合不同時重新讀取該 Skill。如果沒有符合資格的 Skills，則會省略 Skills 區段。

原生 Codex 回合會以回合範圍的協作開發者指示接收此清單，而不是每回合的使用者輸入；但保留確切排程提示詞的輕量排程回合除外。其他執行框架則維持一般提示詞區段。

該位置可指向巢狀 Skill，例如 `skills/personal/foo/SKILL.md`。巢狀結構僅供組織使用；提示詞會使用 `SKILL.md` frontmatter 中的扁平 Skill 名稱。

資格判定包括 Skill 中繼資料閘門、執行階段環境／設定檢查，以及設定 `agents.defaults.skills` 或 `agents.entries.*.skills` 時生效的代理程式 Skill 允許清單。只有在所屬外掛啟用時，外掛隨附的 Skills 才符合資格，讓工具外掛能提供更深入的操作指南，而不必將所有指引嵌入每項工具說明中。

```xml
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
    <version>sha256:...</version>
  </skill>
</available_skills>
```

這能縮小基礎提示詞，同時仍可針對性地使用 Skill。大小限制由 Skills 子系統負責，與一般執行階段的讀取／注入大小限制分開：

| 範圍     | Skills 提示詞預算                                 | 執行階段摘錄預算             |
| --------- | ---------------------------------------------------- | ---------------------------------- |
| 全域    | `skills.limits.maxSkillsPromptChars`                 | `agents.defaults.contextLimits.*`  |
| 每個代理程式 | `agents.entries.*.skillsLimits.maxSkillsPromptChars` | `agents.entries.*.contextLimits.*` |

執行階段摘錄預算涵蓋 `memory_get`、即時工具結果，以及壓縮後的 `AGENTS.md` 重新整理。

## 文件

當本機文件可用時，**文件**區段會指向本機文件（Git 簽出中的 `docs/` 或隨附於 npm 套件的文件）；否則會後備至 [https://docs.openclaw.ai](https://docs.openclaw.ai)。它也會列出 OpenClaw 原始碼的位置：Git 簽出會顯示本機原始碼根目錄；套件安裝則會提供 GitHub 原始碼 URL，並指示在文件不完整或過時時於該處檢閱原始碼。

在模型理解 OpenClaw 的運作方式（記憶／每日筆記、工作階段、工具、閘道、設定、命令、專案情境）之前，提示詞會將文件定位為 OpenClaw 自身知識的權威來源，並告知模型應將 `AGENTS.md`、專案情境、工作區／設定檔／記憶筆記和 `memory_search` 視為指示情境或使用者記憶，而不是 OpenClaw 的設計／實作知識。如果文件未提及或已過時，模型應加以說明並檢查原始碼。提示詞也會告知模型，若可行應自行執行 `openclaw status`，只有在無權存取時才詢問使用者。

特別針對設定，提示詞會引導代理程式使用 `gateway` 工具動作 `config.schema.lookup`，取得精確到欄位層級的文件與限制，再參閱 `docs/gateway/configuration.md` 和 `docs/gateway/configuration-reference.md` 以取得更廣泛的指引。

## 相關內容

- [代理程式執行階段](/zh-TW/concepts/agent)
- [代理程式工作區](/zh-TW/concepts/agent-workspace)
- [情境引擎](/zh-TW/concepts/context-engine)
