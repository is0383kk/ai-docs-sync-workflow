---
read_when:
    - 你想要代理程式透過聊天建立或更新 Skill
    - 你需要審查、套用、拒絕或隔離已產生的技能草稿
    - 你正在設定 Skill Workshop 的核准、自主權、儲存空間或限制
    - 你想了解自我學習提案會在哪裡接受審查
sidebarTitle: Skill Workshop
summary: 透過 Skill Workshop 審查建立及更新工作區 Skills
title: Skill 工作坊
x-i18n:
    generated_at: "2026-07-26T07:38:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2c2590f2a1bcad3b22ef8504eac7b3a44611c3fedc0df3832660f8926ce04252
    source_path: tools/skill-workshop.md
    workflow: 16
---

Skill Workshop 是 OpenClaw 用於建立及更新工作區
Skills 的受控管途徑。代理程式與操作者絕不會透過此途徑直接寫入 `SKILL.md`
——他們會建立一份**提案**（包含內容、目標繫結、掃描器狀態、雜湊及回復中繼資料的待處理草稿），
只有套用後才會成為上線的 Skill。

Skill Workshop 只會寫入工作區 Skills。它絕不會變更內建、
外掛、ClawHub、額外根目錄、受管理、個人代理程式或系統 Skills。

## 運作方式

- **提案優先：**產生的內容會儲存為 `PROPOSAL.md`，而非
  `SKILL.md`。
- **套用是唯一的上線寫入操作：**建立、更新及修訂絕不會變更
  使用中的 Skills。
- **限於工作區：**建立操作以工作區的 `skills/` 根目錄為目標；只有可寫入的工作區 Skills
  才能更新。
- **不覆寫：**如果目標 Skill 已存在，建立操作會失敗。
- **雜湊繫結：**更新提案會繫結目前的目標雜湊；若上線 Skill 在套用前發生變更，
  提案會進入 `stale`。
- **掃描器管制：**套用前會重新執行安全性掃描器。
- **可復原：**套用操作會先寫入回復中繼資料，再變更上線檔案。
- **介面一致：**聊天、命令列介面及閘道都會呼叫相同的服務。

## 生命週期

```text
建立／更新 -> 待處理
修訂       -> 待處理
套用       -> 已套用
拒絕       -> 已拒絕
隔離       -> 已隔離
目標變更   -> 已過時
```

只有 `pending` 提案可以修訂、套用、拒絕或隔離。

## 生命週期整理

閘道會在共用狀態資料庫中追蹤 Skill 的彙總使用情況。它每天會審查一次由
Skill Workshop 建立並套用的 Skills。超過 30 天未使用的 Skill 會變成
`stale`；90 天後會變成 `archived`，且不再納入新的代理程式 Skill 快照。
已封存的 Skill 檔案在磁碟上維持不變。手動編寫的 Skills 絕不會被整理；
只有透過 Skill Workshop 提案建立的 Skills 才會進入生命週期整理。

已釘選的 Skills 不會進行生命週期轉換。過時的 Skill 在再次使用，且下一次掃描完成後，
會回到 `active`。已封存的 Skills 只能透過明確的還原操作恢復：

生命週期轉換與還原會套用至新工作階段；執行中的工作階段會保留目前的
Skill 快照。

```bash
openclaw skills curator status
openclaw skills curator pin <skill>
openclaw skills curator unpin <skill>
openclaw skills curator restore <skill>
```

所有整理器命令都接受 `--json`。狀態也會將確定性的重疊
候選項目僅列為建議；它絕不會合併 Skills 或呼叫模型。

## 聊天

向代理程式提出所需的 Skill；它會呼叫 `skill_workshop` 並傳回
提案 ID。

### 從近期工作中學習

使用 `/learn`，將目前對話或指定來源轉換成一份
遵循標準指引的 Skill 提案：

```text
/learn
/learn docs/runbook.md and https://example.com/guide; focus on recovery
```

未提供要求時，`/learn` 會要求代理程式從目前對話中提煉出
可重複使用的工作流程。提供要求時，代理程式會將路徑、URL、貼上的
筆記及對話參照視為來源，同時遵循重點、範圍及命名要求。它會使用現有工具
收集來源，再以 `action: "create"` 呼叫 `skill_workshop`。

產生的提案會維持 `pending`；`/learn` 絕不會套用它。
請透過一般核准流程或使用 `openclaw skills workshop` 審查並套用提案。

建立：

```text
建立一個名為 morning-catchup 的 Skill，用來執行我的週一收件匣例行工作。
```

更新現有的工作區 Skill：

```text
更新 trip-planning，讓它也能在預訂前檢查座位圖。
```

反覆調整待處理的提案：

```text
顯示 morning-catchup 提案。
修訂它，讓它也能標記任何註明為緊急的項目。
套用 morning-catchup 提案。
```

代理程式發起的 `apply`、`reject` 及 `quarantine`
預設不會顯示額外的核准提示。將 `skills.workshop.approvalPolicy` 設為 `"pending"`，
即可要求操作者在執行這些動作前核准。

需要核准時，提示會指出提案 ID 及目標 Skill，並顯示提案說明、
支援檔案數量及本文大小。核准要求的時間會受到限制，以便在代理程式工具監控器
逾時前完成。如果提示到期前沒有收到決定，生命週期動作便不會執行：
提案會維持待處理且不變。請稍後在 Skill Workshop UI 中決定，或執行
`openclaw skills workshop apply|reject|quarantine <proposal-id>`。代理程式不應循環重試已到期的生命週期動作。

## 命令列介面

```bash
# 建立
openclaw skills workshop propose-create \
  --name morning-catchup \
  --description "每日收件匣追蹤：分類、封存、提出重點、起草、規劃" \
  --proposal ./PROPOSAL.md

# 更新現有的工作區 Skill
openclaw skills workshop propose-update trip-planning --proposal ./PROPOSAL.md

# 列出並檢查
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>

# 核准前修訂
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md

# 結案
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "重複"
openclaw skills workshop quarantine <proposal-id> --reason "需要安全性審查"
```

每個子命令都接受 `--agent <id>`（目標工作區；預設依目前工作目錄推斷，
其次使用預設代理程式）及 `--json`（結構化輸出）。
`propose-create`、`propose-update` 及 `revise` 也接受
`--goal <text>` 和 `--evidence <text>`，以便將提案脈絡連同
`--proposal` 一併記錄。

## 提案內容

在待處理期間，提案會以 `PROPOSAL.md` 儲存，並包含僅供提案使用的
frontmatter：

```markdown
---
name: "morning-catchup"
description: "每日收件匣追蹤：分類、封存、提出重點、起草、規劃"
status: proposal
version: "v1"
date: "2026-05-30T00:00:00.000Z"
---
```

套用時，Skill Workshop 會寫入使用中的 `SKILL.md`，並移除
僅供提案使用的欄位：`status`、提案 `version` 及提案 `date`。

## 支援檔案

當提議的 Skill 需要放置在 `PROPOSAL.md` 旁的檔案時，
請使用 `--proposal-dir`：

```bash
openclaw skills workshop propose-create \
  --name weekly-update \
  --description "週五總結：統計資料、重點、下週前三項要務" \
  --proposal-dir ./weekly-update-proposal
```

該目錄必須包含 `PROPOSAL.md`。支援檔案必須位於
`assets/`、`examples/`、`references/`、`scripts/`
或 `templates/` 下。Skill Workshop 會掃描這些檔案、計算雜湊，
並將它們與提案一併儲存；只有套用時才會將它們寫入上線
`SKILL.md` 旁。

會遭拒絕的支援檔案路徑包括：絕對路徑、隱藏的路徑區段、路徑
遍歷、重疊路徑、可執行檔、非 UTF-8 文字、Null 位元組，
以及標準支援資料夾以外的路徑。

## 代理程式工具

模型會使用 `skill_workshop`，其中有一個必要的 `action`：
`create | update | revise | list | inspect | apply | reject | quarantine`。
其他參數依動作而定：

| 參數                       | 使用動作                                             | 備註                                                                 |
| -------------------------- | ---------------------------------------------------- | -------------------------------------------------------------------- |
| `name`         | `create`、`inspect`、`revise` | `create` 的必要參數；否則依名稱解析待處理提案              |
| `description`         | `create`、`update`、`revise` | 上限 160 位元組                                                       |
| `skill_name`         | `update`                                   | 現有 Skill 名稱或索引鍵                                              |
| `proposal_content`         | `create`、`update`、`revise` | 儲存為 `PROPOSAL.md`；受 `skills.workshop.maxSkillBytes` 限制               |
| `support_files`         | `create`、`update`、`revise` | `{ path, content }` 的陣列                                            |
| `goal`、`evidence` | `create`、`update`、`revise` | 自由文字脈絡                                                         |
| `proposal_id`         | `inspect`、`revise`、`apply`、`reject`、`quarantine` | 目標提案                                              |
| `reason`         | `apply`、`reject`、`quarantine` | 選用                                                                 |
| `query`、`status`、`limit` | `list` | 篩選／分頁；`limit` 上限 50，預設 20                     |

代理程式必須使用 `skill_workshop` 來處理產生的 Skill 工作。它們不得
透過 `write`、`edit`、`exec`、Shell
命令或直接檔案系統操作建立或變更提案檔案。

<Note>
`skill_workshop` 是內建代理程式工具，且包含在
`tools.profile: "coding"` 中。如果較嚴格的原則將其隱藏，請將
`skill_workshop` 加入使用中的 `tools.allow` 清單；若範圍使用的設定檔沒有
明確的 `tools.allow`，則使用 `tools.alsoAllow: ["skill_workshop"]`。
沙箱執行不會建構主機端的 Skill Workshop 工具，因此請從一般主機端
代理程式工作階段或命令列介面執行提案審查動作。
</Note>

## 建議的 Skills

當互動回合結束時，包括失敗的回合，OpenClaw 會偵測「下次」、「記得要」等
可長期沿用的指示及回應式修正。在下一個回合中，代理程式會提議透過
`skill_workshop` 儲存最近偵測到的工作流程；由使用者決定是否建立
提案。這項內建建議功能本身不會建立或變更 Skill。啟用
`skills.workshop.autonomous.enabled`，即可改為直接建立待處理提案。在 Control
UI 中，Workshop 分頁會在頁面標題顯示相同的設定，作為 **Self-learning**
切換開關，並在空白提案看板上顯示為啟用按鈕。

### 掃描過往工作階段

Control UI 無須啟用自主自我學習，即可審查較早的工作。
開啟 **Plugins → Workshop**，然後選取 **Find skill ideas**。掃描會從
最新且符合資格的工作階段開始，並審查有限範圍內具實質內容的工作。
它會略過排程、心跳偵測、Hook、子代理程式、ACP、外掛擁有及內部審查
工作階段，以及模型回合少於六次的對話。

審查器會使用所選代理程式設定的模型，並接收經過機密遮蔽且大小受限的
逐字稿組合。它採用與經驗審查相同的保守門檻：具體的復原模式，或能夠
減少至少兩次未來模型或工具呼叫的穩定程序。例行工作及一次性事實
不應產生提案。

單次掃描最多可以建立或修訂三份待處理提案。它無法套用、
拒絕、隔離或編輯上線 Skill。Workshop 會顯示累計涵蓋範圍，
例如 **已審查 20 個工作階段 · 6 月 18 日至今天 · 找到 2 個構想**。選取
**Scan earlier work**，即可從持久保存的最早工作階段游標繼續。可用歷史記錄
全部掃描完畢後，該動作會變成 **Scan new work**。

歷史審查一律手動進行，即使
`skills.workshop.autonomous.enabled` 為 `false`。每次點擊都會啟動一次模型執行，
因此適用供應商的定價與資料處理條款。游標與涵蓋範圍計數
會儲存在共用的 OpenClaw 狀態資料庫中；逐字稿內容不會複製
到掃描狀態中。

啟用自主擷取後，OpenClaw 也能在成功完成重大工作，且整個代理系統進入閒置狀態後，
進行保守審查。該隔離審查最多可以建立或修訂一項待處理提案。它無法更新使用中的 skill，
也無法套用、拒絕或隔離提案，即使 `approvalPolicy` 為 `"auto"`。

如需啟用方式、資格條件、隱私權與成本詳細資訊、提案門檻和疑難排解，
請參閱[自我學習](/zh-TW/tools/self-learning)。

## 核准與自主性

```json5
{
  skills: {
    workshop: {
      autonomous: {
        enabled: false,
      },
      allowSymlinkTargetWrites: false,
      approvalPolicy: "auto",
      maxPending: 50,
      maxSkillBytes: 40000,
    },
  },
}
```

| 設定                    | 預設值  | 效果                                                                                                                                                              |
| -------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `autonomous.enabled`       | `false`  | 根據明確的修正建立待處理提案；經過閒置延遲後，也會根據已完成且具備可重複使用的復原方法或能顯著節省往返作業的重大工作建立提案。   |
| `allowSymlinkTargetWrites` | `false`  | 允許套用操作透過工作區 skill 符號連結寫入，而其實際目標必須列於 `skills.load.allowSymlinkTargets`。                                                 |
| `approvalPolicy`           | `"auto"` | `"auto"` 會略過代理所發起之 `apply`、`reject` 或 `quarantine` 的額外提示（代理仍須呼叫該動作）。`"pending"` 則需要核准。 |
| `maxPending`               | `50`     | 限制每個工作區的待處理與已隔離提案數量（1-200）。                                                                                                       |
| `maxSkillBytes`            | `40000`  | 限制提案本文的位元組大小（1024-200000）。                                                                                                                     |

自主擷取會辨識預期性規則（例如“從現在開始”）及反應性
修正（例如“那不是我要求的”）。它會依主題將新指示分組，每回合最多產生
三項提案；將詞彙相符項目導向現有且可寫入的工作區 skill；若另一項修正指向相同的 skill，
則修訂自己建立的待處理提案。

對於沒有明確修正的成功重大工作，所選模型會在隔離執行中判斷已完成的軌跡
是否達到保守的提案門檻。前景模型在回覆前不會收到學習提示。背景審查器會保留
前景執行作為提案來源，無法存取一般代理工具，也無法做出生命週期
決策。只有當前景執行階段同時回報其確切解析後的模型，
以及 `skill_workshop` 確實可用時，審查才會開始。因此，限制性或未知的工具政策
會採取封閉式失敗，且不建立提案。

如需完整的自主審查行為與安全
模型，請參閱[自我學習](/zh-TW/tools/self-learning)。

無論 `maxSkillBytes` 為何，
提案說明一律以 160 位元組為上限。

## 閘道方法

| 方法                             | 範圍            |
| ---------------------------------- | ---------------- |
| `skills.proposals.list`            | `operator.read`  |
| `skills.proposals.inspect`         | `operator.read`  |
| `skills.proposals.historyStatus`   | `operator.read`  |
| `skills.proposals.historyScan`     | `operator.admin` |
| `skills.proposals.create`          | `operator.admin` |
| `skills.proposals.update`          | `operator.admin` |
| `skills.proposals.revise`          | `operator.admin` |
| `skills.proposals.requestRevision` | `operator.admin` |
| `skills.proposals.apply`           | `operator.admin` |
| `skills.proposals.reject`          | `operator.admin` |
| `skills.proposals.quarantine`      | `operator.admin` |
| `skills.curator.status`            | `operator.read`  |
| `skills.curator.pin`               | `operator.admin` |
| `skills.curator.unpin`             | `operator.admin` |
| `skills.curator.restore`           | `operator.admin` |

`requestRevision` 僅適用於閘道（沒有命令列介面或代理工具的對應功能）：它會將
自由文字修訂指示轉送至所屬代理的聊天工作階段，而不是直接取代
`PROPOSAL.md`，供要求代理進行修訂，而非提交全新字面內容的使用者介面使用。

`historyStatus` 與 `historyScan` 是 Control UI 支援方法。`historyScan`
接受 `direction: "older" | "newer"`；它一律將結果保留為待處理
提案。

## 儲存空間

```text
<OPENCLAW_STATE_DIR>/skill-workshop/
  proposals.json
  proposals/<proposal-id>/
    proposal.json
    PROPOSAL.md
    rollback.json
    assets/
    examples/
    references/
    scripts/
    templates/
```

預設狀態目錄：`~/.openclaw`。

- `proposal.json`：標準提案記錄。
- `proposals.json`：快速清單索引，可從提案資料夾重建。
- `PROPOSAL.md`：待處理的 skill 提案。
- `rollback.json`：在套用變更至使用中檔案之前寫入的復原中繼資料。

## 限制

| 限制                           | 值                                                                |
| ------------------------------- | -------------------------------------------------------------------- |
| 說明                     | 160 位元組                                                            |
| 提案本文                   | `skills.workshop.maxSkillBytes`（預設 40,000；硬性上限 1 MiB） |
| 支援檔案                   | 每項提案 64 個                                                      |
| 支援檔案大小               | 每個 256 KiB，合計 2 MiB                                            |
| 待處理 + 已隔離提案 | 每個工作區 `skills.workshop.maxPending`（預設 50）              |

## 疑難排解

| 問題                                        | 解決方式                                                                                                                                                                                                  |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Skill proposal description is too large`      | 將 `description` 縮短至 160 位元組以下。                                                                                                                                                                 |
| `Skill proposal content is too large`          | 縮短提案本文或提高 `skills.workshop.maxSkillBytes`。                                                                                                                                         |
| `Target skill changed after proposal creation` | 依照目前目標修訂提案，或建立新提案。                                                                                                                                   |
| `Proposal scan failed`                         | 檢查掃描器發現的問題，然後修訂或隔離提案。                                                                                                                                           |
| `untrusted symlink target`                     | 設定 `skills.load.allowSymlinkTargets`，並且僅針對刻意共用的 skill 根目錄啟用 `skills.workshop.allowSymlinkTargetWrites`。                                                                  |
| `Support file paths must be under one of...`   | 將支援檔案移至 `assets/`、`examples/`、`references/`、`scripts/` 或 `templates/` 之下。                                                                                                                |
| 提案未顯示在清單中                 | 檢查所選的 `--agent` 工作區與 `OPENCLAW_STATE_DIR`。                                                                                                                                            |
| 代理無法呼叫 `skill_workshop`             | 檢查目前的工具政策與執行模式。`coding` 包含該工具；限制性的 `tools.allow` 政策必須明確列出它，而沙箱執行必須使用一般主機端代理工作階段或命令列介面。 |

### 工具政策診斷

啟用自主擷取時，`openclaw doctor` 會針對預設代理執行
`core/doctor/skill-workshop-tool-policy` 檢查。若政策隱藏
`skill_workshop`，警告會指出第一個排除它的設定層，
以及需要進行的確切 `allow` 或 `alsoAllow` 變更。較舊的操作手冊可能仍使用
`openclaw plugins inspect skill-workshop`；該命令現在會說明 Skill
Workshop 為內建功能，並在適用時輸出相同的政策提示。

## 相關內容

- [Skills](/zh-TW/tools/skills)：載入順序、優先順序與可見性
- [自我學習](/zh-TW/tools/self-learning)：保守的執行後 skill 提案
- [建立 skill](/zh-TW/tools/creating-skills)：手寫 `SKILL.md`
  的基本概念
- [Skills 設定](/zh-TW/tools/skills-config)：完整的 `skills.workshop` 結構描述
- [Skills 命令列介面](/zh-TW/cli/skills)：`openclaw skills` 命令
