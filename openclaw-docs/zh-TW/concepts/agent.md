---
read_when:
    - 變更代理程式執行階段、工作區啟動程序或工作階段行為
summary: 代理程式執行階段、工作區合約與工作階段啟動程序
title: 代理程式執行階段
x-i18n:
    generated_at: "2026-07-26T08:29:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4d3dd9c0c65e4ccd791a2a6131f1b7457c8cfee6da71502d93c355280e094390
    source_path: concepts/agent.md
    workflow: 16
---

OpenClaw 內建一個**嵌入式代理程式執行階段**：包含內建的代理程式迴圈、工具
接線與提示詞組裝，與將對話輪次委派給外部
框架程序不同。每個已設定的代理程式（若要執行多個代理程式，請參閱[多代理程式路由](/zh-TW/concepts/multi-agent)）
都有自己的工作區、啟動檔案與工作階段
儲存區。本頁說明該執行階段合約：工作區必須
包含哪些內容、會注入哪些檔案，以及工作階段如何依據這些內容啟動。

## 工作區（必要）

每個代理程式都使用單一工作區目錄（`agents.defaults.workspace`，或每個代理程式使用
`agents.entries.*.workspace`）作為工具與內容脈絡的**唯一**
工作目錄（`cwd`）。

建議：使用 `openclaw setup` 建立缺少的 `~/.openclaw/openclaw.json`，並初始化工作區檔案。

完整的工作區配置與備份指南：[代理程式工作區](/zh-TW/concepts/agent-workspace)

若已啟用 `agents.defaults.sandbox`，非主要工作階段可使用
`agents.defaults.sandbox.workspaceRoot` 下的個別工作階段工作區覆寫此設定（請參閱
[閘道設定](/zh-TW/gateway/configuration)）。

## 啟動檔案（注入）

在工作區內，OpenClaw 預期存在以下可由使用者編輯的檔案：

| 檔案           | 用途                                              |
| -------------- | ---------------------------------------------------- |
| `AGENTS.md`    | 操作指示與「記憶」                    |
| `SOUL.md`      | 人格、界線、語氣                            |
| `TOOLS.md`     | 使用者維護的工具附註與慣例           |
| `IDENTITY.md`  | 代理程式名稱／風格／表情符號                                |
| `USER.md`      | 使用者個人資料與偏好的稱呼方式                     |
| `HEARTBEAT.md` | 心跳偵測專用指示                      |
| `BOOTSTRAP.md` | 一次性的首次執行儀式（完成後刪除） |
| `MEMORY.md`    | 根層級長期記憶檔案（若存在）               |

在新工作階段的第一個對話輪次中，OpenClaw 會將這些檔案的內容注入系統提示詞的專案內容脈絡。只有當 `MEMORY.md` 存在於工作區根目錄時，才會注入該檔案。

空白檔案會被略過。大型檔案會經過修剪與截斷，並附上標記，以保持提示詞精簡（若要取得完整內容，請讀取檔案）。缺少檔案時（`MEMORY.md` 除外），則會改為注入一行「缺少檔案」標記；`openclaw setup` 會為其建立安全的預設範本。

只有在**全新的工作區**（不存在其他啟動檔案）中，才會建立 `BOOTSTRAP.md`。在其尚待處理期間，OpenClaw 會將它保留在專案內容脈絡中，並在系統提示詞中加入初始儀式的啟動指引，而不會將其複製到使用者訊息中。若在完成儀式後將它刪除，之後重新啟動時不會再次建立。

觀察到工作區後，OpenClaw 會將其設定狀態與
證明儲存在共用 SQLite 資料庫
`~/.openclaw/state/openclaw.sqlite` 中。若最近已證明的工作區
消失或遭到清除，啟動程序會拒絕無聲地重新植入 `BOOTSTRAP.md`；
請還原工作區，或使用完整的初始設定重設，以便同時清除工作區及其
資料庫狀態。

較舊版本使用工作區 JSON 與 `.attested` 側載檔案。執行階段
不會讀取這些檔案。請執行 `openclaw doctor --fix` 以驗證這些檔案、將其
狀態匯入 SQLite，並在確認匯入的資料列後移除各來源檔案。

若要完全停用啟動檔案建立功能（適用於已預先植入內容的工作區），請設定：

```json5
{ agents: { defaults: { skipBootstrap: true } } }
```

## 內建工具

核心工具（讀取／執行／編輯／寫入及相關系統工具）一律可用，
但受工具政策限制。對 OpenAI 模型而言，`apply_patch` 預設開啟，並受
`tools.exec.applyPatch`（`enabled`、`workspaceOnly`、`allowModels`）管控。`TOOLS.md` **不會**控制存在哪些工具；它是
關於你希望如何使用這些工具的指引。

## Skills

OpenClaw 會從以下位置載入 Skills（依優先順序由高至低）：

- 工作區：`<workspace>/skills`
- 專案代理程式 Skills：`<workspace>/.agents/skills`
- 個人代理程式 Skills：`~/.agents/skills`
- 受管理／本機：`~/.openclaw/skills`
- 隨附（隨安裝項目提供）
- 額外的 Skill 資料夾：`skills.load.extraDirs`

Skill 根目錄可以包含分組資料夾，例如
`<workspace>/skills/personal/foo/SKILL.md`；該 Skill 仍會以其
扁平化的 frontmatter 名稱公開，例如 `foo`。

Skills 可由設定／環境變數管控（請參閱[閘道設定](/zh-TW/gateway/configuration)中的 `skills`）。

## 執行階段界線

嵌入式代理程式執行階段由 OpenClaw 擁有：模型探索、工具接線、
提示詞組裝、工作階段管理與頻道傳遞共用一個整合式
執行階段介面。

## 工作階段

工作階段資料列儲存在每個代理程式的 SQLite 資料庫中：

- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`

對話記錄 JSONL 檔案仍可存放在
`~/.openclaw/agents/<agentId>/sessions/` 下，作為舊版遷移輸入、已刪除或
重設的封存、匯入、匯出與支援成品。作用中的代理程式歷程會
連同工作階段資料列儲存在 SQLite 中。工作階段 ID 固定不變，並由
OpenClaw 選定。OpenClaw 不會讀取其他工具的工作階段資料夾。

## 串流期間的引導

執行期間收到的輸入提示詞預設會被引導至目前的執行。
引導會在**目前的助理對話輪次完成其
工具呼叫後**、下一次 LLM 呼叫前傳遞，且不再略過
目前助理訊息中其餘的工具呼叫。

`/queue steer` 是預設的作用中執行行為。`/queue followup` 與
`/queue collect` 會讓訊息等候後續對話輪次，而不進行引導。
`/queue interrupt` 則會中止作用中的執行。如需瞭解佇列與邊界行為，請參閱[佇列](/zh-TW/concepts/queue)
和[引導佇列](/zh-TW/concepts/queue-steering)。

區塊串流會在助理區塊完成後立即傳送；此功能
**預設關閉**（`agents.defaults.blockStreamingDefault: "off"`）。
可透過 `agents.defaults.blockStreamingBreak` 調整邊界（`text_end` 與 `message_end`；預設為 `text_end`）。
使用 `agents.defaults.blockStreamingChunk` 控制軟性區塊分段（預設為
800-1200 個字元；優先在段落分隔處分段，其次是換行，最後才是句子）。
使用 `agents.defaults.blockStreamingCoalesce` 合併串流區塊，以減少
單行訊息洗版（傳送前根據閒置狀態合併）。非 Telegram 頻道需要
明確設定 `*.streaming.block.enabled: true` 才能啟用區塊回覆（QQ Bot
則會串流區塊回覆，除非 `channels.qqbot.streaming.mode` 為 `"off"`）。
詳細工具摘要會在工具啟動時發出（不進行防抖）；Control UI
會在可用時透過代理程式事件串流工具輸出。
更多詳細資訊：[串流與分段](/zh-TW/concepts/streaming)。

## 模型參照

設定中的模型參照（例如 `agents.defaults.model` 和 `agents.defaults.models`）會在**第一個** `/` 處分割解析。

- 設定模型時，請使用 `provider/model`。
- 若模型 ID 本身包含 `/`（OpenRouter 風格），請包含供應商前綴（例如：`openrouter/moonshotai/kimi-k2`）。
- 若省略供應商，OpenClaw 會先嘗試別名，接著尋找與該模型 ID 完全相符且唯一的
  已設定供應商，最後才回退至
  已設定的預設供應商。若該供應商不再提供
  已設定的預設模型，OpenClaw 會回退至第一個已設定的
  供應商／模型，而不會顯示已失效的已移除供應商預設值。

## 設定（最低需求）

至少設定：

- `agents.defaults.workspace`
- `channels.whatsapp.allowFrom`（強烈建議）

## 相關內容

- [代理程式工作區](/zh-TW/concepts/agent-workspace)
- [多代理程式路由](/zh-TW/concepts/multi-agent)
- [工作階段管理](/zh-TW/concepts/session)
- [群組聊天](/zh-TW/channels/group-messages)
