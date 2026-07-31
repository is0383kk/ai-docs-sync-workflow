---
read_when:
    - 你需要偵錯工作階段 ID、文字記錄事件或工作階段資料列欄位
    - 你正在變更自動壓縮行為，或新增「壓縮前」的整理作業
    - 你想要實作記憶清空或靜默系統回合
summary: 深入解析：工作階段儲存區與逐字稿、生命週期，以及（自動）壓縮的內部機制
title: 工作階段管理深入解析
x-i18n:
    generated_at: "2026-07-26T08:47:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ae02d49245768831abd17e1c2e5adacfa1a36673cef2a8a7a06a5300392b104
    source_path: reference/session-management-compaction.md
    workflow: 16
---

單一**閘道程序**端對端擁有工作階段狀態。UI（macOS App、Web 控制 UI、終端介面）會向閘道查詢工作階段清單與權杖計數。在遠端模式下，工作階段檔案位於遠端主機，因此檢查本機 Mac 的檔案不會反映閘道實際使用的內容。

請先參閱概覽文件：[工作階段管理](/zh-TW/concepts/session)、[壓縮](/zh-TW/concepts/compaction)、[記憶概覽](/zh-TW/concepts/memory)、[記憶搜尋](/zh-TW/concepts/memory-search)、[工作階段修剪](/zh-TW/concepts/session-pruning)、[逐字稿衛生](/zh-TW/reference/transcript-hygiene)；完整設定參考請見[代理程式設定](/zh-TW/gateway/config-agents)。

## 兩個持久化層

1. **工作階段資料列（每個代理程式的 SQLite）** - 鍵值對應表 `sessionKey -> SessionEntry`。由閘道擁有的可變執行階段狀態。追蹤中繼資料：目前的工作階段 ID、最後活動時間、切換設定、權杖計數器。
2. **逐字稿事件（每個代理程式的 SQLite）** - 僅附加、樹狀結構（項目包含 `id` + `parentId`）。儲存對話、工具呼叫及壓縮摘要；為後續輪次重建模型上下文。壓縮檢查點是已壓縮後繼逐字稿上的中繼資料，新一次壓縮不會寫入第二份 `.checkpoint.*.jsonl`。

較舊的安裝可能仍在代理程式的 `sessions/`
目錄下保有 `sessions.json` 檔案。請將這些檔案視為舊版工作階段資料列的遷移輸入，或明確的
離線維護目標。閘道啟動和 `openclaw doctor --fix` 會自動將
使用中的舊版資料列與逐字稿歷程匯入每個代理程式的 SQLite 儲存區。
需要明確的檢查或驗證證據時，請執行 `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents`，再依循
[Doctor 遷移程序](/zh-TW/cli/doctor#session-sqlite-migration)。若舊版逐字稿
成品封存後遷移失敗，請使用該程序中的 Doctor 復原模式。
復原會使用遷移資訊清單，只還原受影響的已封存支援
成品，並在要求時準備經過清理的 GitHub Issue 報告，而且不會
讓使用中的執行階段再次讀取 JSONL 檔案。

除非介面需要任意歷史存取，否則閘道歷程讀取器會避免將整份逐字稿具體化。第一頁歷程、嵌入式聊天歷程、重新啟動復原，以及權杖／用量檢查，都使用 SQLite 的有界尾端讀取。完整逐字稿掃描會透過非同步逐字稿索引進行，並由並行讀取器共用。

## 磁碟上的位置

在閘道主機上，每個代理程式的位置如下（透過 `src/config/sessions.ts` 解析）：

- 執行階段工作階段資料列儲存區：`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- 執行階段逐字稿資料列：`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- 舊版／封存逐字稿成品：`~/.openclaw/agents/<agentId>/sessions/`
- 舊版資料列遷移輸入：`~/.openclaw/agents/<agentId>/sessions/sessions.json`

## 儲存區維護與磁碟控制

`session.maintenance` 控制 SQLite 工作階段資料列、SQLite 逐字稿資料列、封存成品及軌跡附屬檔案的自動維護：

| 鍵                      | 預設值                | 備註                                                                                         |
| ----------------------- | --------------------- | ------------------------------------------------------------------------------------------- |
| `mode`                  | `"enforce"`           | 或 `"warn"`（僅報告，不進行變更）                                                      |
| `pruneAfter`            | `"30d"`               | 過時項目的存留時間上限                                                                      |
| `maxEntries`            | `500`                 | 工作階段項目數量上限                                                                      |
| `resetArchiveRetention` | 保留（無存留時間上限）  | `*.reset.*`/`*.deleted.*` 逐字稿封存的存留時間上限；指定持續時間即可啟用刪除 |
| `maxDiskBytes`          | `10gb`                | 每個代理程式的工作階段磁碟預算；`false` 可停用                                            |
| `highWaterBytes`        | `maxDiskBytes` 的 80% | 預算清理後的目標值                                                                 |

重設會推進使用中的 `sessionKey -> sessionId` 對應，但保留先前的 SQLite 工作階段、逐字稿、軌跡及搜尋資料列。該歷程在相同工作階段鍵下仍可搜尋；一般的項目與工作階段清單只會顯示新的使用中對應。保留的重設歷程受磁碟預算限制，而非 `resetArchiveRetention`；後者只會讓封存成品隨時間淘汰。明確刪除則不同：它會先寫入並驗證壓縮逐字稿封存（若可使用 zstd，則為 `*.jsonl.deleted.<timestamp>.zst`），之後才移除已刪除工作階段的資料列。

`maxDiskBytes` 的強制執行使用實體位元組：每個代理程式的 SQLite 主檔案、其 `-wal` 檔案，以及代理程式工作階段目錄中納入計算的檔案。它絕不估算資料列 JSON 大小，也不會從總量中扣除邏輯資料列大小。

閘道模型執行探測工作階段（鍵符合 `agent:*:explicit:model-run-<uuid>`）具有獨立且固定的 `24h` 保留期。此修剪由壓力觸發：只有在達到工作階段項目維護／上限壓力時才執行，且只會在全域過時項目清理／上限步驟之前執行。其他明確工作階段不使用此保留期。

當實體用量總和超過 `maxDiskBytes` 時，`mode: "enforce"` 會先回收可透過檢查點釋放的資料庫空間，再移除最舊的保留重設／刪除封存。若用量仍高於 `highWaterBytes`，它會依 `sessions.updated_at` 走訪歷史 SQLite 工作階段，從最舊者開始。所謂歷史工作階段，是指其工作階段 ID 未被使用中工作階段項目、路由目標或已准入／執行中的工作參照。對每個清理目標，清理程序會先寫入壓縮封存、執行 fsync 並讀回，之後寫入交易才會移除工作階段資料列及其逐字稿、軌跡、使用中狀態、索引與 FTS 投影。這包括含有軌跡事件但不含逐字稿事件的工作階段。清理程序會在刪除時重新檢查路由、項目及准入參照，在處理每個封存或工作階段目標後重新測量實體用量，並在達到 `highWaterBytes` 時停止。

已提交的寫入與刪除會先落到 WAL。清理程序會為其建立檢查點，使 WAL 可立即縮小，接著使用增量式 vacuum，從主檔案歸還符合條件的閒置尾端頁面；尚無法回收的頁面會留在主檔案中，因此在下一次實體測量時仍會納入計算。`mode: "warn"` 會回報目前的實體超額用量，不會建立檢查點、寫入封存或刪除資料列。

視需要執行維護：

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

維護會保留群組工作階段和限定討論串範圍的聊天工作階段等持久外部對話指標，但合成的執行階段項目（排程、鉤子、心跳偵測、ACP、子代理程式）一旦超過設定的存留時間、數量或磁碟預算，仍可能遭到移除。隔離的排程執行使用獨立的 `cron.sessionRetention` 控制，與模型執行探測的保留期無關。

一般閘道寫入會經過工作階段存取器，後者透過執行階段寫入器路徑，將每個代理程式的 SQLite 變更序列化。執行階段程式碼應優先使用 `src/config/sessions/session-accessor.ts` 中的存取器輔助函式；舊版 `sessions.json` 輔助函式則是遷移與離線維護工具。閘道可連線時，非試執行的 `openclaw sessions cleanup` 與 `openclaw agents delete` 會將儲存區變更委派給閘道，使清理加入相同的寫入器佇列；`--store <path>` 是所選舊版儲存區的明確離線修復路徑，且一律維持在本機執行（`--dry-run` 亦同）。`maxEntries` 清理會針對正式環境規模的儲存區分批進行，因此儲存區可能會短暫超過設定上限，直到下一次高水位清理將其重寫至上限以下。閘道啟動期間，讀取絕不會修剪項目或套用數量上限；只有寫入或 `openclaw sessions cleanup --enforce` 才會這麼做，後者還會立即套用上限，並修剪未被參照的舊版逐字稿、檢查點及軌跡成品，即使未設定磁碟預算亦然。

OpenClaw 在閘道寫入期間不再自動建立 `sessions.json.bak.*` 輪替備份。目前的結構描述會拒絕舊版 `session.maintenance.rotateBytes` 鍵，而 `openclaw doctor --fix` 會從舊設定中移除該鍵。

逐字稿變更會對 SQLite 逐字稿目標使用工作階段寫入佇列：

工作階段寫入鎖定使用固定的正式環境預設值。對應的
`OPENCLAW_SESSION_WRITE_LOCK_*` 環境變數仍可用於
程序層級的診斷與緊急覆寫。

### 切換至 SQLite 後降級

執行較舊的檔案型 OpenClaw 版本前，請還原已封存的舊版逐字稿成品：

```bash
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

遷移會保留舊版 `sessions.json` 檔案，以供支援與
回復使用，但已匯入 SQLite 的使用中逐字稿 JSONL 檔案會
重新命名並移入 `session-sqlite-import-archive/`。較舊的檔案型執行階段會依循
`sessions.json` 中的 `sessionFile` 路徑，因此啟動前
必須先還原這些成品。還原程序會使用遷移資訊清單，只移動有記錄且
原始路徑已不存在的封存成品，並保留 SQLite 資料庫以便之後向前復原。

切換至 SQLite 後建立的工作階段僅存在於 SQLite 中，不會出現在
較舊的檔案型執行階段中。若降級後再次升級，請重新執行 Doctor
檢查與驗證程序，讓 OpenClaw 能在匯入前驗證已還原的舊版
成品。

## 排程工作階段與執行記錄

隔離的排程執行會建立自己的工作階段項目／逐字稿，並使用專屬保留設定：

- `cron.sessionRetention`（預設為 `"24h"`）會從儲存區修剪舊的隔離排程執行工作階段；`false` 可停用此功能。
- 執行歷程會為每個排程工作保留最新的 2000 個終止資料列。遺失的資料列仍維持其 24 小時清理時間窗。

當排程強制建立新的隔離執行工作階段時，會在寫入新資料列之前清理先前的 `cron:<jobId>` 工作階段項目：它會保留安全的偏好設定（思考／快速／詳細／推理設定、標籤、顯示名稱）及使用者明確選取的模型／驗證覆寫，但會捨棄環境對話上下文（頻道／群組路由、傳送／佇列政策、權限提升、來源、ACP 執行階段繫結），讓全新的隔離執行無法繼承較舊執行中的過時傳送設定或執行階段權限。

## 工作階段鍵（`sessionKey`）

`sessionKey` 用於識別你所在的對話分區（路由 + 隔離）。標準規則：[/concepts/session](/zh-TW/concepts/session)。

| 模式                         | 範例                                                        |
| ---------------------------- | ----------------------------------------------------------- |
| 主要／直接聊天（每個代理程式） | `agent:<agentId>:<mainKey>`（預設為 `main`）                |
| 群組                         | `agent:<agentId>:<channel>:group:<id>`                      |
| 房間／頻道（Discord/Slack）  | `agent:<agentId>:<channel>:channel:<id>` 或 `...:room:<id>` |
| 排程                         | `cron:<job.id>`                                             |
| 網路鉤子                     | `hook:<uuid>`（除非已覆寫）                           |

## 工作階段 ID（`sessionId`）

每個 `sessionKey` 都指向目前的 `sessionId`（延續對話的 SQLite 逐字稿識別碼）。判定邏輯位於 `src/auto-reply/reply/session.ts` 的 `initSessionState()` 中。

- **重設**（`/new`、`/reset`）會為該 `sessionKey` 建立新的 `sessionId`。
- 預設為**不自動重設**。目前的 `sessionId` 會繼續使用，而壓縮會將作用中模型的上下文維持在範圍內。
- **每日重設**（`session.reset.mode: "daily"`）會在經過設定的本機小時界線後，於下一則訊息建立新的 `sessionId`（`session.reset.atHour`，預設為 `4`）。
- **閒置到期**（`session.reset.mode: "idle"` 搭配 `session.reset.idleMinutes`，或舊版 `session.idleMinutes`）會在閒置時段過後收到訊息時建立新的 `sessionId`。若同時設定每日與閒置重設，則以最先到期者為準。
- **控制介面重新連線續接**會保留目前可見的工作階段，供重新連線後傳送一次訊息；前提是閘道從操作員介面用戶端收到相符的 `sessionId`。這是一次性訊號；一般的過期傳送仍會建立新的 `sessionId`。
- **系統事件**（心跳偵測、排程喚醒、執行通知、閘道簿記）可能會變更工作階段資料列，但絕不會延長每日／閒置重設的新鮮度。重設輪替會在建立全新提示詞之前，捨棄上一個工作階段中排入佇列的系統事件通知。
- **父層分支政策**會在建立討論串或子代理分支時，使用 OpenClaw 的作用中分支。若該分支過大（超過固定的內部上限，目前為 100K 個權杖），OpenClaw 會以隔離的上下文啟動子項，而不會失敗或繼承無法使用的歷程記錄。大小計算會自動進行且無法設定；舊版 `session.parentForkMaxTokens` 設定會由 `openclaw doctor --fix` 移除。
- **操作員分支**：`sessions.create { parentSessionKey, fork: true }` 會建立新的工作階段，其逐字記錄從父層目前的狀態分支而來（使用與產生子代理相同的分支機制，包括上述大小上限）。父層有作用中的執行時會拒絕建立分支；除非明確傳入模型選擇，否則會繼承父層的模型選擇；並使用全新的權杖計數器將子項標記為 `forkedFromParent`。

## 工作階段儲存區結構描述

執行階段儲存區會將 `SessionEntry` 值保存在各代理專屬的 SQLite 中。值的型別是在 `src/config/sessions.ts` 中定義的 `SessionEntry`。主要欄位（非完整清單）：

- `sessionId`：用於定址 SQLite 逐字記錄資料列的目前逐字記錄 ID
- `sessionStartedAt`：目前 `sessionId` 的開始時間戳記；每日重設的新鮮度會使用此值。舊版資料列可能會從 JSONL 工作階段標頭衍生此值。
- `lastInteractionAt`：上次真實使用者／頻道互動的時間戳記；閒置重設的新鮮度會使用此值，因此心跳偵測、排程和執行事件不會讓工作階段保持作用中。缺少此欄位的舊版資料列會改用復原的工作階段開始時間。
- `updatedAt`：上次變更儲存區資料列的時間戳記，用於清單、修剪和簿記，而不是每日／閒置新鮮度的權威依據。
- `archivedAt`：選用的封存時間戳記。已封存的工作階段會連同完整逐字記錄保留在儲存區中，且不會出現在一般的作用中清單內。
- `pinnedAt`：選用的釘選時間戳記。作用中的已釘選工作階段會排序在未釘選工作階段之前；封存工作階段會清除其釘選狀態。
- Codex 討論串互通：兩個欄位都遵循 Codex 的討論串管理形式——線路上的 `archived`/`pinned` 布林值一律衍生自時間戳記，並由伺服器端加註，符合 Codex `threads.archived_at` 語意和駝峰式序列化。OpenClaw 時間戳記使用 Epoch 毫秒，而 Codex 使用 Epoch 秒，因此橋接器會在 `codex` 外掛接縫處進行轉換。Codex 尚無釘選 API（只有 `thread/archive`/`thread/unarchive`）；在該 API 出現前，釘選狀態會留在 OpenClaw 端，之後相符的形式便能讓已繫結的工作階段自動往返同步釘選狀態。
- Codex 監督功能只會列出未封存的原生討論串。只有在操作員明確確認沒有其他 Codex 程序擁有該討論串後，才能透過原生 `thread/archive` 封存閘道本機的 `idle` 或活動狀態不明的 `notLoaded` 討論串；外掛會先重新讀取程序本機狀態，之後該討論串便會從目錄中消失。該讀取作業無法證明其他 App Server 程序未使用此討論串。OpenClaw 拒絕封存作用中和錯誤資料列，而且在節點橋接器能夠擁有完整的串流討論串生命週期之前，無法封存配對節點。在原生 Codex 用戶端中解除封存後，討論串即可再次出現。
- `lastReadAt` / `markedUnreadAt`：由 `sessions.patch { unread }` 在伺服器端加註的讀取狀態時間戳記——`unread: false` 會記錄一次讀取（設定 `lastReadAt`、清除 `markedUnreadAt`）；`unread: true` 會將工作階段標記為未讀，直到下次讀取。工作階段資料列會公開衍生的 `unread` 布林值：明確標記為未讀，或讀取時間早於最新活動。從未標記為已讀的工作階段會維持 `unread: false`，因此現有安裝不會在升級後突然顯示未讀。
- `lastActivityAt`：上次完成且應計為值得標示未讀之活動的代理執行時間戳記（使用者、頻道及排程執行）。心跳偵測和內部事件回合，以及中繼資料修補，都不會更新此值；`updatedAt` 不是活動訊號。
- `sessionFile`：為遷移／封存相容性保留的舊版標記；作用中執行階段使用 SQLite 身分
- `chatType`：`direct | group | room`
- `provider`、`subject`、`room`、`space`、`displayName`：群組／頻道標籤中繼資料
- 切換項目：`thinkingLevel`、`verboseLevel`、`reasoningLevel`、`elevatedLevel`、`sendPolicy`（各工作階段覆寫）
- 模型選擇：`providerOverride`、`modelOverride`、`authProfileOverride`
- 權杖計數器（盡力提供／取決於供應商）：`inputTokens`、`outputTokens`、`totalTokens`、`contextTokens`
- `compactionCount`：此工作階段索引鍵完成自動壓縮的次數
- `memoryFlushAt` / `memoryFlushCompactionCount`：上次壓縮前記憶體排清的時間戳記和壓縮次數

閘道是權威來源：工作階段執行時，它可能會重寫或重新載入項目。對於舊版以檔案為基礎的安裝，請使用
`openclaw doctor --session-sqlite import --session-sqlite-all-agents` 遷移，而不要編輯
`sessions.json` 並預期執行階段會繼續讀取該檔案。

## 逐字記錄事件結構

逐字記錄由 OpenClaw 工作階段存取器管理，並透過以身分為基礎的輔助函式公開給執行階段程式碼。事件串流僅能附加：

- 第一個項目：工作階段標頭——`type: "session"`、`id`、`cwd`、`timestamp`，以及選用的 `parentSession`。
- 接著：包含 `id` + `parentId` 的項目（樹狀結構）。

值得注意的項目型別：

- `message`：使用者／助理／工具結果訊息
- `custom_message`：由擴充功能注入且_會_進入模型上下文的訊息（當 `display: true` 時會顯示在終端介面中；當 `display: false` 時會完全隱藏）
- `custom`：由擴充功能提供且_不會_進入模型上下文的狀態（用於在重新載入後保存擴充功能狀態）
- `compaction`：包含 `firstKeptEntryId` 和 `tokensBefore` 的持久化壓縮摘要
- `branch_summary`：導覽樹狀分支時保存的摘要

OpenClaw 刻意不會「修正」逐字記錄；閘道使用 `SessionManager` 讀寫逐字記錄。

## 上下文視窗與追蹤的權杖

這是兩個不同的概念：

1. **模型上下文視窗**：每個模型的硬性上限（模型可見的權杖）。此值來自模型目錄，也可透過設定覆寫。
2. **工作階段儲存區計數器**：寫入工作階段資料列的滾動統計資料（用於 `/status` 和儀表板）。`contextTokens` 是執行階段估計／回報值——請勿將其視為嚴格保證。

如需更多限制相關資訊，請參閱：[/reference/token-use](/zh-TW/reference/token-use)。

## 壓縮：這是什麼

壓縮會將較舊的對話彙整成逐字記錄中持久保存的 `compaction` 項目，並完整保留近期訊息。壓縮後，後續回合會看到壓縮摘要，以及 `firstKeptEntryId` 之後的訊息。壓縮是**持久性的**，與工作階段修剪不同——請參閱 [/concepts/session-pruning](/zh-TW/concepts/session-pruning)。

內嵌式 OpenClaw 壓縮預設會繼承工作階段的思考層級。設定 `agents.defaults.compaction.thinkingLevel` 即可為摘要呼叫使用不同層級；執行階段會將其限制在每個實際壓縮模型或備援模型支援的範圍內。原生 Codex App Server 壓縮會自行管理其壓縮要求，無法接受個別壓縮的思考層級覆寫，因此 OpenClaw 會顯示警告，並將該設定交由 Codex 處理。

壓縮後重新注入 AGENTS.md 區段仍須透過 `agents.defaults.compaction.postCompactionSections` 明確啟用。外掛可透過 `before_prompt_build` 新增其他提示詞上下文。

### 區塊界線與工具配對

將較長的逐字記錄分割成壓縮區塊時，OpenClaw 會將助理工具呼叫與其相符的 `toolResult` 項目保持配對：

- 如果依權杖占比分割會使界線落在工具呼叫及其結果之間，OpenClaw 會將界線移到助理工具呼叫訊息，而不會拆散兩者。
- 如果尾端工具結果區塊會讓區塊超過目標，OpenClaw 會保留該待處理工具區塊，並完整保留未摘要的尾端內容。
- 已中止／發生錯誤的工具呼叫區塊不會讓待處理分割持續保持開啟。

## 自動壓縮的發生時機

內嵌式 OpenClaw 代理中有兩個觸發條件：

1. **溢位復原**：模型傳回上下文溢位錯誤（`request_too_large`、`context length exceeded`、`input exceeds the maximum number of tokens`、`input token count exceeds the maximum number of input tokens`、`input is too long for the model`、`ollama error: context length exceeded`，以及其他供應商特有的變體）——先壓縮，再重試。當供應商回報嘗試使用的權杖數時，OpenClaw 會將觀察到的計數轉送至溢位復原壓縮；如果供應商確認溢位但未公開可剖析的計數，OpenClaw 會將略微超出預算的最小合成計數傳給壓縮引擎和診斷功能。如果溢位復原仍失敗，OpenClaw 會顯示明確指引並保留目前的工作階段對應，而不是無提示地輪替為新的工作階段 ID——請重試該訊息、執行 `/compact`，或執行 `/new`。
2. **臨界值維護**：成功完成一個回合後，當目前上下文超過模型視窗減去 OpenClaw 為提示詞及下一次模型輸出內建的預留空間時。

這兩個觸發條件之外還會執行另外兩項防護：

- **執行前本機壓縮**：設定 `agents.defaults.compaction.maxActiveTranscriptBytes`（位元組數，或如 `"20mb"` 的字串），即可在作用中逐字稿達到該大小後，於開啟下一次執行前觸發本機壓縮。這是用來控制本機重新開啟成本的大小防護，而非原始封存機制——一般的語意壓縮仍會執行，且需要 `truncateAfterCompaction`，才能讓壓縮後的摘要成為新的後繼逐字稿。
- **回合中預先檢查**：設定 `agents.defaults.compaction.midTurnPrecheck.enabled: true`（預設為 `false`）以新增工具迴圈防護。附加工具結果後、下一次模型呼叫前，OpenClaw 會使用與回合開始時相同的執行前預算邏輯估算提示詞壓力。如果內容已無法容納，防護不會就地壓縮——它會發出結構化的回合中預先檢查訊號、停止目前的提示詞提交，並讓外層執行迴圈使用既有的復原路徑（若截斷過大的工具結果已足夠，便進行截斷；否則觸發已設定的壓縮模式並重試）。同時適用於 `default` 與 `safeguard` 壓縮模式，包括由提供者支援的防護壓縮。此機制獨立於 `maxActiveTranscriptBytes`：位元組大小防護會在回合開啟前執行，而回合中預先檢查則稍後在附加新工具結果後執行。

## 壓縮設定

```json5
{
  agents: {
    defaults: {
      compaction: {
        enabled: true,
        keepRecentTokens: 20000,
      },
    },
  },
}
```

OpenClaw 會對嵌入式執行強制套用內建保留額度，並依作用中模型的內容視窗設定上限，使其無法耗盡整個提示詞預算。這可避免內容視窗較小的本機模型從第一個權杖起就進入壓縮，同時為記憶體清除等多回合例行維護保留足夠空間。

手動 `/compact` 會遵循明確的 `agents.defaults.compaction.keepRecentTokens`，並保留執行階段的近期尾端切分點。若未明確指定保留預算，手動壓縮會成為硬性檢查點，而重建的內容會從新摘要開始。

啟用 `truncateAfterCompaction` 時，OpenClaw 會在壓縮後將作用中逐字稿輪替為壓縮後的後繼逐字稿。分支／還原檢查點動作會使用該壓縮後的後繼逐字稿；舊版壓縮前檢查點檔案只要仍被參照，就仍可讀取。

## 可插拔壓縮提供者

外掛會透過外掛 API 上的 `registerCompactionProvider()` 註冊壓縮提供者。當 `agents.defaults.compaction.provider` 設為已註冊的提供者 ID 時，防護擴充功能會將摘要工作委派給該提供者，而非使用內建的 `summarizeInStages` 流水線。

- `provider`：已註冊壓縮提供者外掛的 ID。保留未設定即可使用預設的 LLM 摘要。設定 `provider` 會強制使用 `mode: "safeguard"`。
- 提供者會收到與內建路徑相同的壓縮指示及識別碼保留政策，而防護機制仍會在提供者輸出後保留近期回合及分割回合的後綴內容。
- 內建防護摘要會將先前摘要與新訊息重新提煉，而非逐字保留完整的舊摘要。
- 防護模式預設會啟用摘要品質稽核；設定 `qualityGuard.enabled: false` 可略過輸出格式錯誤時的重試行為。
- 如果提供者失敗或傳回空白結果，OpenClaw 會自動改用內建 LLM 摘要。呼叫端明確觸發的中止／逾時訊號會重新擲出而不會被忽略，因此取消要求一定會受到遵循。

來源：`src/plugins/compaction-provider.ts`、`src/agents/agent-hooks/compaction-safeguard.ts`。

## 使用者可見介面

- 任何聊天工作階段中的 `/status`
- `openclaw status`（命令列介面）
- `openclaw sessions` / `openclaw sessions --json`
- 閘道記錄（`pnpm gateway:watch` 或 `openclaw logs --follow`）：`embedded run auto-compaction start` + `complete`
- 詳細模式：`🧹 Auto-compaction complete` 加上壓縮次數

## 靜默例行維護（`NO_REPLY`）

OpenClaw 支援背景工作的「靜默」回合，讓使用者不會看到中間輸出。

- 助理以完全一致的靜默權杖 `NO_REPLY` / `no_reply` 作為輸出開頭，表示「不要向使用者傳送回覆」。OpenClaw 會在傳送層移除／抑制此內容。
- 完全一致的靜默權杖抑制不區分大小寫：當整個承載內容只有靜默權杖時，`NO_REPLY` 和 `no_reply` 都會生效。
- 自 `2026.1.10` 起，如果部分區塊以 `NO_REPLY` 開頭，OpenClaw 也會抑制草稿／輸入中串流，避免靜默作業在回合進行期間洩漏部分輸出。
- 這只適用於真正的背景／不傳送回合——不能用它來便宜處理一般可執行的使用者要求。

## 壓縮前記憶體清除

自動壓縮發生前，OpenClaw 可執行靜默的代理式回合，將持久狀態寫入磁碟（例如代理工作區中的 `memory/YYYY-MM-DD.md`），避免壓縮清除關鍵內容。它會監控工作階段的內容使用量；一旦超過低於壓縮門檻的軟性門檻，就會使用完全一致的靜默權杖 `NO_REPLY` / `no_reply` 傳送靜默的「立即寫入記憶」指示，讓使用者不會看到任何內容。

設定（`agents.defaults.compaction.memoryFlush`），完整參考資料請見 [/gateway/config-agents](/zh-TW/gateway/config-agents#agentsdefaultscompaction)：

| 鍵                          | 預設值           | 備註                                                                                                                                   |
| --------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                   | `true`           |                                                                                                                                        |
| `model`                     | 未設定           | 僅針對清除回合精確覆寫提供者／模型，例如 `ollama/qwen3:8b`                                                                            |
| `softThresholdTokens`       | `4000`           | 低於壓縮門檻且會觸發清除的差距                                                                                                         |
| `forceFlushTranscriptBytes` | 未設定（停用）   | 即使權杖計數器已過時，逐字稿檔案達到此位元組大小（或如 `"2mb"` 的字串）時仍強制清除；`0` 會停用 |

備註：

- 內建提示詞和系統提示詞包含 `NO_REPLY` 提示，用於抑制傳送。
- 設定 `model` 時，清除回合會使用該模型，而不繼承作用中工作階段的備援鏈，因此僅限本機的例行維護失敗時，不會無聲地改用付費對話模型。
- 每個壓縮週期只會執行一次清除（於工作階段資料列中追蹤）。
- 清除只會對嵌入式 OpenClaw 工作階段執行；命令列介面後端和心跳偵測回合會略過。
- 工作階段工作區為唯讀時（`workspaceAccess: "ro"` 或 `"none"`），會略過清除。
- 工作區檔案配置和寫入模式請參閱[記憶體](/zh-TW/concepts/memory)。

OpenClaw 在擴充功能 API 中公開 `session_before_compact` 鉤點，但上述清除邏輯位於閘道端（`src/auto-reply/reply/memory-flush.ts`、`src/auto-reply/reply/agent-runner-memory.ts`），而非該鉤點上。

## 疑難排解檢查清單

- **工作階段金鑰錯誤？** 請先參閱[/concepts/session](/zh-TW/concepts/session)，並確認 `/status` 中的 `sessionKey`。
- **儲存區與逐字稿不符？** 請透過 `openclaw status` 確認閘道主機和儲存區路徑。
- **壓縮過於頻繁？** 請檢查模型的內容視窗（太小會迫使系統頻繁壓縮）及工具結果膨脹情況（調整工作階段修剪）。
- **在小型本機模型上，每個提示詞似乎都會溢位？** 請確認提供者回報正確的模型內容視窗。只有在已知該視窗大小時，OpenClaw 才能限制有效保留額度。
- **靜默回合發生洩漏？** 請確認回覆以完全一致的靜默權杖 `NO_REPLY` 開頭（不區分大小寫），且使用的組建包含串流抑制修正（`2026.1.10`+）。

## 相關內容

- [工作階段管理](/zh-TW/concepts/session)
- [工作階段修剪](/zh-TW/concepts/session-pruning)
- [內容引擎](/zh-TW/concepts/context-engine)
- [代理設定參考](/zh-TW/gateway/config-agents)
