---
read_when:
    - 你希望 OpenClaw 從已完成的對話中學習可重複使用的程序
    - 你正在決定是否啟用自主 Skills 提案
    - 你需要瞭解自我學習的安全性、成本、適用資格或疑難排解方式
sidebarTitle: Self-learning
summary: 讓 OpenClaw 根據修正內容與已完成的重要工作提出可重複使用的 Skills 建議
title: 自我學習
x-i18n:
    generated_at: "2026-07-26T08:46:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b10618c1a64441bdf0ba58f03e02972bdf2b1d59643a78358910594f8139ccb8
    source_path: tools/self-learning.md
    workflow: 16
---

自我學習可讓 OpenClaw 將對話中的實用證據轉換為待處理的
[Skill Workshop](/zh-TW/tools/skill-workshop) 提案。它不會訓練模型
權重、編輯作用中的技能，或在未告知的情況下變更代理程式行為。每項學到的
程序都會維持待處理狀態，直到操作人員審查並套用為止。

自我學習**預設為停用**。只有在額外的
背景模型執行與對話記錄審查適合你的工作區時，才啟用此功能。

## 啟用自我學習

在 Control UI 中，開啟 **Plugins → Workshop**，然後開啟 **Self-learning**。此
變更會立即生效；當另一個設定寫入程式已更新
檔案時，Control UI 會重新整理設定快照並重試切換，而無須重新載入
頁面或閘道。

使用命令列介面：

```bash
openclaw config set skills.workshop.autonomous.enabled true --strict-json
```

或編輯 `~/.openclaw/openclaw.json`：

```json5
{
  skills: {
    workshop: {
      autonomous: {
        enabled: true,
      },
    },
  },
}
```

使用以下命令再次停用：

```bash
openclaw config set skills.workshop.autonomous.enabled false --strict-json
```

停用自我學習時，使用者要求的技能建立、`/learn` 和手動 Skill Workshop 操作
仍可繼續運作。

## 手動審查過去的工作階段

手動歷程記錄審查是自主擷取的保守替代方案。
在 Control UI 中開啟 **Plugins → Workshop**，然後選取 **Find skill ideas**。
這不會變更 `skills.workshop.autonomous.enabled`。

每次掃描：

- 從最新且尚未審查的工作階段開始，並逐步往回處理；
- 最多審查 20 個至少包含六個模型回合的實質工作階段；
- 略過排程、心跳偵測、鉤子、子代理程式、ACP、外掛擁有及內部審查
  工作階段；
- 在將對話記錄套件傳送給所選代理程式已設定的模型前，遮蔽已辨識的密鑰並限制其大小；
- 採用與自主經驗審查相同的高標準；以及
- 最多可建立或修訂三個待處理提案，絕不會處理作用中的技能。

Workshop 會報告累計工作階段數、日期涵蓋範圍及找到的構想。
選取 **Scan earlier work** 以處理下一個較舊的區段。當游標抵達
符合資格之歷程記錄的開頭時，動作會變更為 **Scan new work**。
OpenClaw 只會將游標與涵蓋範圍中繼資料保留在共用狀態資料庫中；
不會建立第二份對話記錄封存。

只有當 OpenClaw 能證明工作階段的擁有權並排除
外部鉤子內容時，才會掃描工作階段。升級後，目前升級前的對話記錄可在
本機分類，但會略過缺少每次執行來源資訊且已輪替的升級前對話記錄。新的對話記錄會在輪替後保留此來源資訊。

手動掃描仍會產生模型供應商費用，並將符合資格的對話
內容傳送給已設定的供應商。只有當該審查符合
工作區的隱私與資料處理要求時，才使用此功能。

## OpenClaw 可以學習的內容

自我學習有兩條保守路徑：

1. **直接指示與更正。** OpenClaw 會偵測持久有效的語句，
   例如「從現在開始」、「下次」，以及對失敗方法的更正。
   啟用自我學習後，它可將這些訊號轉換為待處理提案，
   而無須等待另一個提示。此確定性路徑可將相關
   指示分組為最多三個提案、以可寫入的工作區技能為目標，
   或修訂其自身相關的待處理提案。它也會在失敗的回合後執行，
   因為它擷取的是使用者的指示，而不是判斷是否完成。
2. **經驗審查。** 在成功且內容充實的前景回合後，
   OpenClaw 可審查已完成的工作，找出可重複使用的復原技巧或
   能減少未來至少兩次模型或工具往返的穩定程序。

適合的候選項目包括：

- 在工具或模型重複失敗後的可靠復原方法；
- 可避免重複發生錯誤的非顯而易見順序限制；
- 需要重複探索的穩定多步驟工作流程；或
- 可避免未來多次呼叫的可重複使用預先檢查。

對於例行的成功工作、一次性要求、個人事實、簡單偏好、暫時性
環境失敗、一般性建議、缺乏依據的否定主張及密鑰，審查程式應放棄提出提案。

## 經驗審查的執行時機

經驗審查會刻意延遲且受到限制：

- 前景回合必須成功完成。
- 目前回合必須包含至少十次模型反覆運算。
- 排程、心跳偵測、記憶體、溢位、鉤子、子代理程式及審查工作階段
  會被排除。
- 前景執行必須已解析供應商和模型，而且實際上
  必須能存取 `skill_workshop`。
- OpenClaw 會在完成後等待 30 秒。同一工作階段中較晚完成的前景作業
  會重新開始該靜默期。
- 若任何代理程式或回覆執行仍在進行中，審查會再等待 30 秒。
- 同一時間只會執行一項經驗審查。
- 延遲審查是處理程序本機的閘道工作。閘道必須在
  閒置期間持續執行；單次本機與命令列介面支援的執行階段無法保留
  足夠的軌跡與工具可用性內容來排程審查。

前景答案絕不會因學習而延遲。失敗或不符合資格的
回合不會啟動經驗審查，但停用自主功能時，仍可將使用者的直接更正
作為建議提供。

## 審查程式會收到的內容

背景審查程式只會收到目前回合，從其最近的
使用者訊息開始。呈現後的軌跡上限為 60,000 個字元；
必要時，OpenClaw 會保留第一則訊息與最新證據，並
標記中間省略的部分。

審查程式會重複使用已解析的供應商和模型。當該身分可用時，它會重複使用前景
認證設定檔，並停用模型後援。因此，此
審查會在已設定的供應商上啟動一次額外的模型執行。
當該執行檢查或草擬提案時，可能會提出多次供應商要求。供應商的定價與資料處理條款
會如同前景回合一樣適用。

開始前，OpenClaw 會重新載入目前的執行階段設定，並重新檢查
原始對話的有效沙箱與工具原則。若執行位於
沙箱中、原則已不再允許 `skill_workshop`，或缺少必要的執行階段事實，
審查會採取封閉式失敗且不建立任何內容。

<Warning>
  啟用自我學習會允許將符合資格的對話內容，包括目前回合中的工具
  輸入與結果，傳送給所選模型
  供應商進行一次額外審查。若該審查會違反資料處理要求，
  請勿在該工作區中啟用此功能。
</Warning>

## 提案安全性

審查程式會在隔離的工作階段中執行，且工具
介面受到刻意限制：

- 它只能列出或檢查 Workshop 提案，以及建立或修訂一個
  待處理提案。
- 它無法更新作用中的技能、套用提案、拒絕提案、隔離
  提案、傳送訊息或使用一般代理程式工具。
- 模型重試會共用一個異動額度，因此一次審查最多只能建立或
  修訂一個提案。
- 受審查的軌跡會被視為不受信任的證據，而不是背景代理程式的
  指示。
- Skill Workshop 會掃描提案內容，並在寫入提案狀態前拒絕
  已辨識的常值認證資訊。

一般 Workshop 限制仍然適用，包括 `maxPending`、`maxSkillBytes`、
支援檔案限制、掃描程式檢查及僅限工作區的寫入。即使設定
`approvalPolicy: "auto"`，也不會授予背景審查程式存取
生命週期動作的權限。

## 審查學到的提案

自我學習會產生與手動使用 Workshop 相同的待處理提案。
套用前請先檢查：

```bash
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

修訂、拒絕或隔離實用但尚未就緒的提案：

```bash
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md
openclaw skills workshop reject <proposal-id> --reason "Too specific"
openclaw skills workshop quarantine <proposal-id> --reason "Needs security review"
```

套用是唯一會寫入作用中 `SKILL.md` 的操作。完整的生命週期與儲存
模型請參閱
[Skill Workshop](/zh-TW/tools/skill-workshop)。

## 設定

| 設定                                       | 預設值   | 自我學習效果                                                                                                                      |
| ------------------------------------------ | -------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `skills.workshop.autonomous.enabled`       | `false`  | 啟用直接更正擷取與延遲經驗審查。                                                                                                  |
| `skills.workshop.approvalPolicy`           | `"auto"` | 控制一般代理程式所啟動生命週期動作的核准提示；不會擴大背景審查程式的權限。                                                        |
| `skills.workshop.maxPending`               | `50`     | 限制每個工作區的待處理與已隔離提案數量。                                                                                          |
| `skills.workshop.maxSkillBytes`            | `40000`  | 限制提案本文大小（位元組）。                                                                                                      |
| `skills.workshop.allowSymlinkTargetWrites` | `false`  | 僅影響套用行為；自我學習本身會寫入提案狀態，而非作用中技能目標。                                                                  |

如需完整的結構描述、範圍及相關技能設定，請參閱
[Skills 設定](/zh-TW/tools/skills-config#workshop-skills-workshop)。

## 疑難排解

### 長回合後未出現提案

請檢查以下所有項目：

1. 作用中閘道設定中的 `skills.workshop.autonomous.enabled` 為 `true`。
2. 回合已成功，且最近的使用者訊息之後至少包含十次模型反覆運算。
3. 對話是一般前景執行，而不是排程、記憶體、
   鉤子或子代理程式執行。
4. 原始執行可存取 `skill_workshop`，且不在沙箱中。
5. 系統維持閒置的時間足以執行延遲審查。
6. 長時間執行的閘道處理程序在整個閒置期間都維持作用中；
   單次本機命令不會等待延遲審查。

符合資格的審查仍可能不產生提案。當證據未達到可重複使用程序的標準時，
放棄提出提案是預期結果。

### Doctor 回報 Workshop 工具已隱藏

啟用自我學習時，`openclaw doctor` 會檢查預設
代理程式的有效工具原則是否允許 `skill_workshop`。依照回報的
`tools.allow` 或 `tools.alsoAllow` 變更操作，或停用自我學習。

### 出現過多低價值提案

停用自我學習，並繼續使用 `/learn` 或明確的 Workshop 要求：

```bash
openclaw config set skills.workshop.autonomous.enabled false --strict-json
```

停用此功能後，待處理提案仍可供審查。停用
自我學習不會套用、拒絕或刪除這些提案。

## 相關內容

- [Skill 工作坊](/zh-TW/tools/skill-workshop)，用於提案審查、核准及
  儲存
- [建立 Skills](/zh-TW/tools/creating-skills)，用於手動編寫的 Skills 及
  `SKILL.md` 結構
- [Skills 設定](/zh-TW/tools/skills-config)，涵蓋所有 `skills.*` 設定
- [Skills 命令列介面](/zh-TW/cli/skills)，用於工作坊及策展人命令
