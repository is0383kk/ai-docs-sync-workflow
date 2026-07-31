---
read_when:
    - 你想了解「上下文」在 OpenClaw 中的意義
    - 你正在偵錯模型為何「知道」某件事（或忘記了它）
    - 你想要降低上下文負擔（/context、/status、/compact）
summary: 情境：模型看到的內容、其建構方式，以及如何檢視情境
title: 上下文
x-i18n:
    generated_at: "2026-07-26T08:21:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1eb3d342a601a447487640587f746cc80a133ede338a880741f53c3e01f20ed1
    source_path: concepts/context.md
    workflow: 16
---

「情境」是 **OpenClaw 在一次執行中傳送給模型的所有內容**。它受限於模型的**情境視窗**（權杖限制）。

初學者心智模型：

- **系統提示詞**（由 OpenClaw 建置）：規則、工具、Skills 清單、時間／執行階段，以及注入的工作區檔案。
- **對話記錄**：你在此工作階段中的訊息 + 助理的訊息。
- **工具呼叫／結果 + 附件**：命令輸出、檔案讀取內容、圖片／音訊等。

情境與「記憶」_並不相同_：記憶可以儲存在磁碟上並於稍後重新載入；情境則是模型目前視窗內的內容。

## 快速開始（檢查情境）

- `/status` → 快速查看「我的視窗用了多少？」及工作階段設定。
- `/context list` → 查看注入了哪些內容及概略大小（各檔案 + 總計）。
- `/context detail` → 更深入的明細：各檔案、各工具結構描述、各 Skill 項目的大小、系統提示詞大小，以及可壓縮的逐字記錄訊息數量。
- `/context map` → 目前工作階段中受追蹤情境來源的 WinDirStat 風格矩形式樹狀圖。
- `/usage tokens` → 在一般回覆後附加每次回覆的使用量頁尾。
- `/compact` → 將較舊的記錄摘要為精簡項目，以釋放視窗空間。

另請參閱：[斜線命令](/zh-TW/tools/slash-commands)、[權杖用量與成本](/zh-TW/reference/token-use)、[壓縮](/zh-TW/concepts/compaction)。

## 輸出範例

數值會依模型、供應商、工具政策及工作區內容而異。

### `/context list`

```text
🧠 情境明細
工作區：<workspaceDir>
啟動程序上限／檔案：12,000 個字元
沙箱：mode=non-main sandboxed=false
系統提示詞（執行）：38,412 個字元（約 9,603 個權杖）（專案情境 23,901 個字元（約 5,976 個權杖））

注入的工作區檔案：
- AGENTS.md：正常 | 原始 1,742 個字元（約 436 個權杖）| 注入 1,742 個字元（約 436 個權杖）
- SOUL.md：正常 | 原始 912 個字元（約 228 個權杖）| 注入 912 個字元（約 228 個權杖）
- TOOLS.md：已截斷 | 原始 54,210 個字元（約 13,553 個權杖）| 注入 20,962 個字元（約 5,241 個權杖）
- IDENTITY.md：正常 | 原始 211 個字元（約 53 個權杖）| 注入 211 個字元（約 53 個權杖）
- USER.md：正常 | 原始 388 個字元（約 97 個權杖）| 注入 388 個字元（約 97 個權杖）
- HEARTBEAT.md：缺少 | 原始 0 | 注入 0
- BOOTSTRAP.md：正常 | 原始 0 個字元（約 0 個權杖）| 注入 0 個字元（約 0 個權杖）

Skills 清單（系統提示詞文字）：2,184 個字元（約 546 個權杖）（12 個 Skills）
工具：read、edit、write、exec、process、browser、message、sessions_send、…
工具清單（系統提示詞文字）：1,032 個字元（約 258 個權杖）
工具結構描述（JSON）：31,988 個字元（約 7,997 個權杖）（會計入情境；不會顯示為文字）
工具：（同上）

工作階段權杖（已快取）：總計 14,250 / ctx=32,000
```

### `/context detail`

```text
🧠 情境明細（詳細）
…
最大的 Skills（提示詞項目大小）：
- frontend-design：412 個字元（約 103 個權杖）
- oracle：401 個字元（約 101 個權杖）
…（另有 10 個 Skills）

最大的工具（結構描述大小）：
- browser：9,812 個字元（約 2,453 個權杖）
- exec：6,240 個字元（約 1,560 個權杖）
…（另有 N 個工具）
```

### `/context map`

傳送一張由最新快取的執行報告與工作階段逐字記錄產生的圖片。在工作階段中的一般訊息產生執行報告之前，`/context map` 會傳回無法使用的訊息，而非算繪估計值。矩形面積與受追蹤的提示詞字元數成正比：

- 對話逐字記錄（使用者訊息、助理回覆、工具結果、壓縮摘要），以及只會傳至模型的每回合執行階段情境與掛鉤提示詞增補內容
- 注入的工作區檔案
- 基礎系統提示詞文字
- Skill 提示詞項目
- 工具 JSON 結構描述

對話群組會隨工作階段進行而增長，因此此圖每回合都會改變；壓縮後，它會縮減為摘要圖塊。

沒有快取執行報告時，`/context list`、`/context detail` 和 `/context json` 仍可檢查隨需產生的估計值。

## 哪些內容會計入情境視窗

模型收到的所有內容都會計入，包括：

- 系統提示詞（所有區段）。
- 對話記錄。
- 工具呼叫 + 工具結果。
- 附件／逐字記錄（圖片／音訊／檔案）。
- 壓縮摘要與修剪成品。
- 供應商「包裝器」或隱藏標頭（不可見，但仍會計入）。

## OpenClaw 如何建置系統提示詞

系統提示詞**由 OpenClaw 擁有**，並在每次執行時重新建置。它包括：

- 工具清單 + 簡短說明。
- Skills 清單（僅中繼資料；請見下文）。
- 工作區位置。
- 時間（UTC + 已設定時轉換後的使用者時間）。
- 執行階段中繼資料（主機／作業系統／模型／思考）。
- 在**專案情境**下所注入的工作區啟動程序檔案。

完整明細：[系統提示詞](/zh-TW/concepts/system-prompt)。

## 注入的工作區檔案（專案情境）

OpenClaw 預設會注入一組固定的工作區檔案（若存在）：

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`（僅限首次執行）

大型檔案會依各檔案使用 `agents.defaults.bootstrapMaxChars` 截斷（預設為 `20000` 個字元）。OpenClaw 也會透過 `agents.defaults.bootstrapTotalMaxChars`，對所有檔案強制套用啟動程序注入總上限（預設為 `60000` 個字元）。`/context` 會顯示**原始與注入後**的大小，以及是否發生截斷。

發生截斷時，執行階段可在專案情境下的提示詞中注入警告區塊。可透過 `agents.defaults.bootstrapPromptTruncationWarning`（`off`、`once`、`always`；預設為 `always`）進行設定。

## Skills：注入與隨需載入

系統提示詞包含精簡的 **Skills 清單**（名稱 + 說明 + 位置）。此清單確實會產生額外負擔。

預設不會包含 Skill 指示。模型應該**只在需要時** `read` Skill 的 `SKILL.md`。

## 工具：有兩種成本

工具會以兩種方式影響情境：

1. 系統提示詞中的**工具清單文字**（你看到的「工具」）。
2. **工具結構描述**（JSON）。這些內容會傳送給模型，使其能夠呼叫工具。即使你看不到其純文字內容，它們仍會計入情境。

`/context detail` 會列出最大工具結構描述的明細，讓你瞭解哪些內容占比最高。

## 命令、指令與「行內捷徑」

斜線命令由閘道處理。它們有幾種不同的行為：

- **獨立命令**：只有 `/...` 的訊息會作為命令執行。
- **指令**：`/think`、`/fast`、`/verbose`、`/trace`、`/reasoning`、`/elevated`、`/exec`、`/model`、`/queue` 會在模型看到訊息前遭到移除。
  - 僅包含指令的訊息會保存工作階段設定。
  - 一般訊息中的行內指令會作為每則訊息的提示。
- **行內捷徑**（僅限允許清單中的傳送者）：一般訊息內的特定 `/...` 權杖可立即執行（例如：「hey /status」），並會在模型看到其餘文字前遭到移除。

詳細資訊：[斜線命令](/zh-TW/tools/slash-commands)。

## 工作階段、壓縮與修剪（會保存哪些內容）

訊息之間會保存哪些內容，取決於所用機制：

- **一般記錄**會保留在工作階段逐字記錄中，直到依政策壓縮／修剪為止。
- **壓縮**會將摘要保存至逐字記錄中，並原樣保留近期訊息。
- **修剪**會從_記憶體內_提示詞中移除舊工具結果，以釋放情境視窗空間，但不會重寫工作階段逐字記錄——仍可在磁碟上檢查完整記錄。

文件：[工作階段](/zh-TW/concepts/session)、[壓縮](/zh-TW/concepts/compaction)、[工作階段修剪](/zh-TW/concepts/session-pruning)。

OpenClaw 預設使用內建的 `legacy` 情境引擎進行組裝與
壓縮。如果你安裝了提供 `kind: "context-engine"` 的外掛，並
透過 `plugins.slots.contextEngine` 選取該外掛，OpenClaw 便會將情境
組裝、`/compact` 及相關子代理程式情境生命週期掛鉤委派給該
引擎。`ownsCompaction: false` 不會自動退回舊版
引擎；作用中的引擎仍必須正確實作 `compact()`。請參閱
[情境引擎](/zh-TW/concepts/context-engine)，瞭解完整的
可插拔介面、生命週期掛鉤及設定。

## `/context` 實際回報的內容

如有可用資料，`/context` 會優先使用最新的**執行時建置**系統提示詞報告：

- `System prompt (run)` = 從上次嵌入式（可使用工具的）執行中擷取，並保存於工作階段儲存區。
- `System prompt (estimate)` = 沒有執行報告時（或透過不會產生報告的命令列介面後端執行時）即時計算。

無論是哪種方式，它都會回報大小和主要來源；**不會**傾印完整的系統提示詞或工具結構描述。在詳細模式下，它也會使用壓縮所採用的相同真實對話訊息述詞，比較工作階段逐字記錄，讓高提示詞／快取用量與可壓縮的對話記錄更容易區分。

## 相關內容

<CardGroup cols={2}>
  <Card title="情境引擎" href="/zh-TW/concepts/context-engine" icon="puzzle-piece">
    透過外掛自訂情境注入。
  </Card>
  <Card title="壓縮" href="/zh-TW/concepts/compaction" icon="compress">
    摘要長篇對話，使其保持在模型視窗範圍內。
  </Card>
  <Card title="系統提示詞" href="/zh-TW/concepts/system-prompt" icon="message-lines">
    系統提示詞的建置方式，以及每回合注入的內容。
  </Card>
  <Card title="代理程式迴圈" href="/zh-TW/concepts/agent-loop" icon="arrows-rotate">
    從收到訊息到最終回覆的完整代理程式執行週期。
  </Card>
</CardGroup>
