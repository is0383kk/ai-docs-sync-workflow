---
read_when:
    - 你想讓代理程式向使用者提出結構化問題
    - 你正在回答或偵錯 ask_user 提示詞
    - 你需要 `ask_user` 結構描述、逾時或頻道行為
summary: ask_user 如何暫停代理程式回合，以取得結構化的人工作決策
title: 詢問使用者
x-i18n:
    generated_at: "2026-07-26T08:08:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 32556314a34c26054c3aabfdd8ecc474cf85196e5cc71adb833face596edbd24
    source_path: tools/ask-user.md
    workflow: 16
---

`ask_user` 可讓代理程式向使用者提出一至三個結構化問題，並
等待回答。此工具適用於真正應由使用者做出的決定，
而非例行確認，或代理程式可從請求、
程式碼或合理預設值推導出的資訊。

此工具僅在主要工作階段中可用。子代理程式和其他非主要
執行不會取得此工具。

## 回答問題

你可以從任何支援的對話介面回答：

- 網頁版 Control UI 會將問題面板直接固定在編輯器上方。對於
  包含多個問題的提示，面板會一次顯示一個問題，並透過簡短的步驟導覽
  逐題前進。處理完成後，面板會關閉，而聊天中
  僅保留精簡的回答摘要。
- Telegram、Discord 和 Slack 會針對單選、
  單一問題的提示顯示原生按鈕。
- 純文字回覆適用於任何頻道。請以數字、選項標籤
  或你自己的答案回覆。

OpenClaw 一律啟用自由文字的 **其他** 回答。代理程式不得在編寫的選項清單中加入
`Other` 選項。

## 平台行為

所有支援的對話介面都可進行回答。網頁版 Control UI 使用
固定式步驟導覽，展開時會取代編輯器；收合後，完整編輯器會恢復顯示於
精簡問題列下方。iOS、macOS 和 Android 會顯示
行內卡片；多個問題會維持堆疊，這是刻意採用、便於觸控操作的
呈現方式。每個平台都會將問題與回答摘要保留在目前聊天的
時間軸中，不會定時移除，而且所有平台都提供 **略過**。

無法使用原生按鈕的提示（包括多問題和
多選提示）在頻道中會降級為易讀的文字。Control UI
則會保留完整的結構化步驟導覽。

## 逾時與未回答

預設逾時時間為 900 秒。`timeoutSeconds` 會限制在
30 至 3600 秒的範圍內。

如果問題在收到回答前到期或遭到取消，工具會
傳回 `status: "no_answer"`。接著代理程式會依最佳判斷繼續。
中止代理程式執行會取消其待處理的閘道問題。

## 工具結構描述

```ts
{
  questions: Array<{
    id: string; // 唯一的 snake_case 回答鍵
    header: string; // 簡短標籤；截斷為 12 個字元
    question: string; // 一個句子
    options: Array<{
      label: string;
      description?: string;
    }>; // 2-4 個選項
    multiSelect?: boolean;
  }>; // 1-3 個問題
  timeoutSeconds?: number; // 整數；預設為 900，限制在 30-3600
}
```

使用 `multiSelect: true` 時，使用者可選擇多個選項。每個問題的回答
值都會以陣列傳回。

已回答結果範例：

```json
{
  "status": "answered",
  "answers": {
    "answers": {
      "deploy_target": ["Staging (Recommended)"]
    }
  }
}
```

## 模型指引

面向模型的合約會指示代理程式：

- 僅在確實因應由使用者決定的事項而受阻時提問；
- 優先只問一個問題，且不得超過三個；
- 將建議選項放在第一個，並在其標籤後加上 `(Recommended)`；
- 省略自行編寫的 `Other` 選項，因為系統會自動加入自由文字輸入；
- 在 `no_answer` 後依最佳判斷繼續。

代理程式不應使用 `ask_user` 詢問是否可以繼續，或確認
自己的計畫。
