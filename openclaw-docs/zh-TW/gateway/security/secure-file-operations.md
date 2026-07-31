---
read_when:
    - 變更檔案存取、封存檔解壓縮、工作區儲存空間或外掛檔案系統輔助工具
summary: OpenClaw 如何安全地處理本機檔案存取，以及為何預設停用選用的 fs-safe Python 輔助工具
title: 安全的檔案操作
x-i18n:
    generated_at: "2026-07-26T07:53:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5c8edf36ddbb8c8bc1edc52ecdf481affe5395d1779c679a40439167dfe70299
    source_path: gateway/security/secure-file-operations.md
    workflow: 16
---

OpenClaw 使用 [`@openclaw/fs-safe`](https://github.com/openclaw/fs-safe) 執行涉及安全性的本機檔案操作：限制於根目錄範圍內的讀取／寫入、原子取代、封存檔解壓縮、暫存工作區、JSON 狀態，以及機密檔案處理。

它是供受信任的 OpenClaw 程式碼在接收不受信任的路徑名稱時使用的**程式庫防護機制**，並非沙箱。主機檔案系統權限、作業系統使用者、容器，以及代理程式／工具政策，仍會決定實際的影響範圍。

## 預設：不使用 Python 輔助程式

OpenClaw 預設將 fs-safe POSIX Python 輔助程式設為**關閉**：

- 除非操作人員選擇啟用，否則閘道不應啟動持續運作的 Python 附屬程序；
- 大多數安裝環境不需要額外強化父目錄異動操作；
- 停用 Python 可讓桌面、Docker、CI 與封裝應用程式環境中的執行階段行為保持可預期。

OpenClaw 只會變更_預設值_。明確設定一律優先：

```bash
# OpenClaw 預設行為：僅使用 Node 的 fs-safe 後備機制。
OPENCLAW_FS_SAFE_PYTHON_MODE=off

# 在輔助程式可用時選擇啟用，無法使用時則改用後備機制。
OPENCLAW_FS_SAFE_PYTHON_MODE=auto

# 若輔助程式無法啟動，則採取失敗關閉。
OPENCLAW_FS_SAFE_PYTHON_MODE=require

# 選用的明確直譯器路徑。
OPENCLAW_FS_SAFE_PYTHON=/usr/bin/python3
```

一般的 fs-safe 環境變數名稱也可使用：`FS_SAFE_PYTHON_MODE` 和 `FS_SAFE_PYTHON`。

當輔助程式是安全防護策略的一部分時，請使用 `require`（而非 `auto`）；如果輔助程式無法啟動，`auto` 會無提示地改用僅限 Node 的行為。

## 不使用 Python 時仍具備的保護

關閉輔助程式後，OpenClaw 仍會獲得 fs-safe 僅限 Node 的防護機制：

- 拒絕相對路徑逸出（`..`）、絕對路徑，以及在只允許單純名稱的位置出現路徑分隔符號；
- 透過受信任的根目錄控制代碼解析操作，而非使用臨時拼湊的 `path.resolve(...).startsWith(...)` 檢查；
- 對要求此政策的 API 拒絕符號連結與硬連結模式；
- 當 API 傳回或取用檔案內容時，會以身分檢查開啟檔案；
- 透過同層暫存檔案加上重新命名，以原子方式寫入狀態／設定檔；
- 對讀取與封存檔解壓縮強制執行位元組限制；
- 在 API 要求時，對機密資訊與狀態檔案套用私密檔案模式。

這涵蓋 OpenClaw 的一般威脅模型：在單一受信任的操作人員邊界內，由受信任的閘道程式碼處理來自不受信任模型／外掛／頻道的路徑輸入。

## Python 增加的保護

在 POSIX 上，選用的輔助程式會維持一個持續運作的 Python 程序，並針對父目錄異動操作使用相對於檔案描述元的檔案系統操作：重新命名、移除、建立目錄、取得狀態／列出內容，以及部分寫入路徑。

這能縮小相同 UID 的競爭條件時間窗口，避免其他程序在驗證與異動之間替換父目錄；對於不受信任的本機程序可修改 OpenClaw 操作之相同目錄的主機而言，這是一項縱深防禦措施。

如果你的部署環境具有此風險，且能保證 Python 存在，請設定：

```bash
OPENCLAW_FS_SAFE_PYTHON_MODE=require
```

## 外掛與核心指南

- 當路徑來自訊息、模型輸出、設定或外掛輸入時，提供給外掛的檔案存取應透過 `openclaw/plugin-sdk/*` 輔助函式進行，而非直接使用 `fs`。
- 核心程式碼應使用 `src/infra/*` 下的 fs-safe 包裝函式，確保 OpenClaw 的程序政策能一致套用。
- 封存檔解壓縮應使用 fs-safe 封存檔輔助函式，並明確設定大小、項目數量、連結及目的地限制。
- 機密資訊應使用 OpenClaw 機密資訊輔助函式，或 fs-safe 的機密資訊／私密狀態輔助函式；請勿自行在 `fs.writeFile` 周圍實作模式檢查。
- 如需隔離具敵意的本機使用者，請勿僅依賴 fs-safe。請在不同的作業系統使用者／主機下執行個別閘道，或使用沙箱隔離。

相關內容：[安全性](/zh-TW/gateway/security)、[沙箱隔離](/zh-TW/gateway/sandboxing)、[執行核准](/zh-TW/tools/exec-approvals)、[機密資訊](/zh-TW/gateway/secrets)。
