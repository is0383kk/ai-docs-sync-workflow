---
read_when:
    - 你想要瞭解工作階段的路由與隔離機制
    - 你想要為多使用者設定配置私訊範圍
    - 你正在偵錯每日或閒置工作階段重設問題
summary: OpenClaw 如何管理對話工作階段
title: 工作階段管理
x-i18n:
    generated_at: "2026-07-26T08:22:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de85fe5a623bdbc6d5564d822b39e9077a582b0816b62ab30d2f7245bd097000
    source_path: concepts/session.md
    workflow: 16
---

OpenClaw 會根據每則傳入訊息的來源，將其路由至一個**工作階段**：
例如私訊、群組聊天、排程工作等。所有工作階段狀態皆由
**閘道**擁有；UI 用戶端會向閘道查詢工作階段資料。

關於個人代理程式的預設方式——所有私訊管道共用一個持續延伸的對話，
群組活動與背景工作也會流入其中——請參閱
[主要工作階段](/zh-TW/concepts/main-session)。

## 訊息如何路由

| 來源          | 行為                  |
| --------------- | ------------------------- |
| 私訊 | 預設共用工作階段 |
| 群組聊天     | 每個群組各自隔離        |
| 聊天室／頻道  | 每個聊天室各自隔離         |
| 排程工作       | 每次執行使用全新工作階段     |
| 網路鉤子        | 每個鉤子各自隔離         |

## 私訊隔離

預設情況下，所有私訊會共用一個工作階段以保持連貫性，這適合
單一使用者的設定。

<Warning>
若有多人可傳訊息給你的代理程式，請啟用私訊隔離。若未啟用，
所有使用者都會共用相同的對話上下文，因此 Alice 的私人訊息可能會
顯示給 Bob。
</Warning>

```json5
{
  session: {
    dmScope: "per-channel-peer", // 依頻道 + 傳送者隔離
  },
}
```

`session.dmScope` 選項：

| 值                      | 行為                                                 |
| -------------------------- | -------------------------------------------------------- |
| `main`（預設）           | 所有私訊共用[主要工作階段](/zh-TW/concepts/main-session) |
| `per-peer`                 | 跨頻道依傳送者隔離                       |
| `per-channel-peer`         | 依頻道 + 傳送者隔離（建議）                |
| `per-account-channel-peer` | 依帳號 + 頻道 + 傳送者隔離                    |

<Tip>
若同一人透過多個頻道與你聯絡，請使用
`session.identityLinks` 將其身分對應至一個標準對等端 ID，讓
這些身分共用一個工作階段。
</Tip>

### 停靠已連結的頻道

停靠命令會將目前私聊工作階段的回覆路由移至另一個
已連結的頻道，而不啟動新的工作階段。範例、設定與
疑難排解請參閱[頻道停靠](/zh-TW/concepts/channel-docking)。

使用 `openclaw security audit` 驗證你的設定。

## 無痕工作階段

無痕工作階段僅能從控制 UI 的 **New thread** 畫面使用。開始討論串前開啟 **Incognito**，即可將其工作階段項目、逐字記錄與壓縮狀態保留在處理程序記憶體中，而非儲存至磁碟。閘道重新啟動後，該討論串即會消失；它不會執行 OpenClaw 的自動記憶體清除，且在你重設或刪除時不會建立逐字記錄封存。由 Codex 支援的執行也會以暫時模式啟動其控制框架討論串，因此 Codex 不會寫入任何 rollout 或本機工作階段狀態檔案；其他模型供應商使用 HTTP API，不會在 OpenClaw 中保留本機供應商逐字記錄。

`incognito-` 區段保留供儀表板、子代理程式及隱藏的內部工作階段金鑰使用；`openclaw doctor --fix` 會重新命名任何發生衝突的舊版持久性金鑰。

無痕模式不會限制代理程式的一般工具。明確要求儲存資訊，或任何由工具驅動的檔案寫入，仍可能在無痕工作階段儲存區之外持久保存資料。你設定的模型供應商仍會處理你傳送的訊息，診斷記錄維持不變，而且 OpenClaw 仍會記錄不含內容的稽核中繼資料，例如 HMAC 參照。

在多使用者閘道上，無痕討論串僅對具有管理員範圍的連線可見，且絕不會透過其他工作階段的代理程式工作階段工具或逐字記錄搜尋顯示。這可避免資料被儲存，並防止其他由閘道中介的使用者存取，但無法防範閘道擁有者或處理程序操作員，因為他們隨時都能觀察即時工作階段。

## 跨對話記憶

不同逐字記錄分別控制各對話的本機歷程。對於個人或完全信任的代理程式，`memory.search.rememberAcrossConversations: true`
會在該代理程式的其他私人
對話間新增選用的擷取步驟；它不會合併這些對話的逐字記錄。

私人直接對話與明確建立且持久的 UI 對話，可以彼此提供相關
上下文。群組與頻道在雙向上皆維持隔離：
其逐字記錄不會作為私人回憶來源，而這些
對話中的回覆也不會收到私人逐字記錄上下文。目前的
對話也會排除，因為其歷程已載入。

此設定不會變更工作階段金鑰、私訊範圍、路由、傳遞或
`tools.sessions.visibility`。`MEMORY.md` 與
`memory/*.md` 中的共用工作區記憶也會維持既有行為。目前的記憶供應商
必須支援受保護的私人逐字記錄回憶；Lossless Claw 等
上下文引擎維持獨立，並可與其並行執行。設定
與執行階段詳情請參閱[主動記憶](/zh-TW/concepts/active-memory#remember-across-conversations)。

## 工作階段生命週期

工作階段會持續重複使用，直到你手動重設，或選擇採用自動重設原則：

- **不自動重設**（預設 `mode: "none"`）- 工作階段維持相同的
  `sessionId`；隨著對話增長，由壓縮管理作用中的上下文。
- **每日重設**（`mode: "daily"`）- 選擇在閘道主機上設定的本機
  小時（`session.reset.atHour`，預設 `4`，0-23）建立新工作階段。每日
  時效性取決於目前的 `sessionId` 何時開始，而非後續的
  中繼資料寫入時間。
- **閒置重設**（`mode: "idle"`）- 選擇在閒置 `session.reset.idleMinutes`
  後建立新工作階段。閒置時效性取決於最後一次真實的使用者／頻道
  互動，因此心跳偵測、排程與 exec 系統事件不會讓
  工作階段保持作用中。
- **手動重設** - 在聊天中輸入 `/new` 或 `/reset`。`/new <model>` 也會
  切換模型。

同時設定每日重設與閒置重設時，先到期者優先。
心跳偵測、排程、exec 與其他系統事件回合可能會寫入工作階段中繼資料，
但這些寫入不會延長每日或閒置重設的時效性。重設使
工作階段輪替時，舊工作階段佇列中的系統事件通知會遭
捨棄，避免過時的背景更新附加在新工作階段第一個提示詞之前。

具有供應商所擁有且作用中之命令列介面工作階段的工作階段，也遵循相同的不自動重設
預設值。若這些工作階段應依計時器到期，請使用 `/reset` 或明確設定 `session.reset`。

先全域選擇採用自動重設，再依聊天類型或頻道覆寫：

```json5
{
  session: {
    reset: { mode: "daily", atHour: 4 },
    resetByType: {
      group: { mode: "idle", idleMinutes: 120 },
      thread: { mode: "daily", atHour: 6 },
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 10080 },
    },
  },
}
```

`resetByType` 支援 `direct`、`group` 與 `thread`。Doctor 會將舊版 `dm` 項目遷移至 `direct`，並將 `session.idleMinutes` 遷移至 `session.reset.idleMinutes`；結構描述會拒絕這兩種已淘汰形式。

## 狀態儲存位置

- **執行階段工作階段資料列：** `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- **已封存的逐字記錄檔案：** `~/.openclaw/agents/<agentId>/sessions/`
- **舊版資料列遷移來源：** `~/.openclaw/agents/<agentId>/sessions/sessions.json`

每個代理程式的 SQLite 資料庫中的工作階段資料列會分別保存生命週期
時間戳記：

- `sessionStartedAt`：目前的 `sessionId` 開始時間；每日重設會使用此值。
- `lastInteractionAt`：最後一次會延長閒置期限的使用者／頻道互動。
- `updatedAt`：最後一次儲存區資料列異動；適合用於列出與修剪，但並非
  每日／閒置重設時效性的權威依據。

從較舊的安裝版本遷移期間，閘道啟動與 `openclaw doctor
--fix` 會自動將舊版 `sessions.json` 資料列和熱逐字記錄 JSONL 歷程匯入
SQLite。缺少 `sessionStartedAt` 的資料列會在可用時，從
舊版逐字記錄 JSONL 工作階段標頭解析。若較舊的資料列也
缺少 `lastInteractionAt`，閒置時效性會回退至該工作階段的開始時間，
而非後續的簿記寫入時間。若要取得明確的
檢查或驗證證據，請使用 `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` 與 [Doctor 遷移
順序](/zh-TW/cli/doctor#session-sqlite-migration)。

## 工作階段維護

OpenClaw 透過 `session.maintenance` 限制工作階段儲存空間隨時間增長，以下為
預設值：

```json5
{
  session: {
    maintenance: {
      mode: "enforce", // "enforce" 會套用清理；"warn" 僅回報
      pruneAfter: "30d",
      maxEntries: 500,
    },
  },
}
```

對於正式環境規模的 `maxEntries` 限制，閘道執行階段寫入會使用小型
高水位緩衝區，並分批清理回設定的上限。
閘道啟動期間，工作階段儲存區讀取不會修剪或限制項目，
因此啟動與隔離的排程工作階段不必負擔完整儲存區清理的成本。
`openclaw sessions cleanup --enforce` 會立即套用上限。

閘道模型執行探測工作階段預設為短期存在。符合
`agent:*:explicit:model-run-<uuid>` 的資料列使用固定的 `24h` 保留期，但清理作業
受壓力條件控制：僅在達到工作階段項目
維護／上限壓力時，才會移除過時的探測資料列，並且會在較廣泛的過時項目
存留時間截止與項目上限之前執行。一般的直接、群組、討論串、排程、鉤子、心跳偵測、
ACP 與子代理程式工作階段不會沿用此 24h 保留期。

維護作業會保留持久的外部對話指標，包括群組
工作階段與討論串範圍的聊天工作階段，同時仍允許合成的排程、
鉤子、心跳偵測、ACP 與子代理程式項目隨時間淘汰。

已封存的工作階段由使用者收存，不受任何自動維護
路徑影響，包括依存留時間修剪、項目上限、模型執行清理與磁碟預算
逐出。它們會保持封存，直到你取消封存或明確
刪除。

若你先前使用私訊隔離，後來再將 `session.dmScope` 恢復為
`main`，請使用
`openclaw sessions cleanup --dry-run --fix-dm-scope` 預覽過時的對等端金鑰私訊資料列。套用相同的旗標
會淘汰這些舊的直接私訊資料列，並將其逐字記錄保留為已刪除的
封存。

使用 `openclaw sessions cleanup --dry-run` 預覽任何維護執行。

## 檢查工作階段

| 命令                    | 顯示內容                                           |
| -------------------------- | ----------------------------------------------- |
| `openclaw status`          | 工作階段儲存區路徑與近期活動          |
| `openclaw sessions --json` | 所有工作階段（使用 `--active <minutes>` 篩選） |
| 聊天中的 `/status`          | 上下文用量、模型與切換選項               |
| `/context list`            | 系統提示詞中的內容                    |

## 延伸閱讀

- [工作階段搜尋](/zh-TW/concepts/session-search) - 跨過往逐字記錄進行全文回憶
- [工作階段修剪](/zh-TW/concepts/session-pruning) - 修剪工具結果
- [壓縮](/zh-TW/concepts/compaction) - 摘要長篇對話
- [工作階段工具](/zh-TW/concepts/session-tool) - 用於跨工作階段作業的代理程式工具
- [工作階段管理深入解析](/zh-TW/reference/session-management-compaction) -
  儲存區結構描述、逐字記錄、傳送原則、來源中繼資料與進階設定
- [多代理程式](/zh-TW/concepts/multi-agent) - 跨代理程式的路由與工作階段隔離
- [背景工作](/zh-TW/automation/tasks) - 分離的工作如何建立帶有工作階段參照的工作記錄
- [頻道路由](/zh-TW/channels/channel-routing) - 傳入訊息如何路由至工作階段

## 相關內容

- [工作階段修剪](/zh-TW/concepts/session-pruning)
- [工作階段工具](/zh-TW/concepts/session-tool)
- [命令佇列](/zh-TW/concepts/queue)
