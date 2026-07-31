---
read_when:
    - 正在處理遙測／隱私權控制功能
    - 關於收集哪些資料的問題
summary: ClawHub 命令列介面所收集的安裝遙測資料，以及如何選擇退出。
x-i18n:
    generated_at: "2026-07-26T07:45:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a02bb1c76fea3105255235f6314ade73f260f692d6eb1b41f8001dc84db6ded7
    source_path: clawhub/telemetry.md
    workflow: 16
---

# 遙測

ClawHub 使用最低限度的命令列介面遙測資料，計算彙總的 skill 與外掛安裝次數。

## 收集遙測資料的時機

僅在下列情況下傳送遙測資料：

- 你已在命令列介面中登入。
- 你執行 `clawhub install <slug>`，或完成經過驗證的
  `openclaw plugins install clawhub:<package>` 安裝。
- 遙測功能**未停用**（請參閱下方的「如何停用」）。

如果你尚未登入，則不會回報任何資料。

## 我們收集的資料

當 skill 或外掛安裝完成，且其本機安裝記錄已持久化後，命令列介面會以盡力而為的方式傳送一筆安裝事件。

事件包含：

- 已安裝 skill 的 slug 或外掛的標準套件名稱。
- `version`：已安裝的版本（若已知）。

### 我們_不會_收集的資料

- 不收集資料夾路徑或衍生自資料夾的識別碼。
- 不收集檔案內容。
- 不收集每次執行的記錄、提示詞或其他命令列介面輸出。

## 安裝次數

對於 skill，ClawHub 會維護：

- `installsAllTime`：曾回報至少一次該 skill 命令列介面安裝的唯一使用者。
- `installsCurrent`：曾回報安裝且未刪除其
  遙測資料的唯一使用者。

對於外掛，ClawHub 會計算每位使用者針對每個套件所回報的第一次成功安裝。重複安裝與更新會重新整理記錄的版本，但不會增加彙總安裝次數。

## 透明度與使用者控制

所有人都只能看到**彙總的安裝計數器**。

刪除你的帳號時，也會刪除你的遙測資料，並從安裝計數器中移除其貢獻。

## 如何停用遙測功能

設定環境變數：

```bash
export CLAWHUB_DISABLE_TELEMETRY=1
```

設定此變數後，命令列介面將不會傳送安裝遙測資料。
