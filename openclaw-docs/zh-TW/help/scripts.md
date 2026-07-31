---
read_when:
    - 從儲存庫執行指令碼
    - 新增或變更 ./scripts 下的指令碼
summary: 儲存庫指令碼：用途、範圍與安全注意事項
title: 指令碼
x-i18n:
    generated_at: "2026-07-26T07:44:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 323069190ea6647101ee7120e06f6b2a018833d0904a11787fa1b610f5b3d9e1
    source_path: help/scripts.md
    workflow: 16
---

`scripts/` 包含本機工作流程與維運任務的輔助指令碼。當任務明確與某個指令碼相關時，請使用這些指令碼；否則優先使用命令列介面。

## 慣例

- 除非文件或發布檢查清單中有提及，否則指令碼均為**選用**。
- 若有命令列介面可用，請優先使用（範例：`openclaw models status --check`）。
- 請假設指令碼僅適用於特定主機；在新機器上執行前，請先閱讀其內容。

## 認證監控指令碼

一般模型認證請參閱[身分驗證](/zh-TW/gateway/authentication)。以下指令碼是一套獨立的選用系統，用於在遠端／無頭主機上監控 **Claude Code 命令列介面訂閱權杖**，並透過手機重新進行身分驗證：

- `scripts/setup-auth-system.sh` - 一次性設定：檢查目前的認證狀態、協助產生長效的 `claude setup-token`，並輸出 systemd／Termux 安裝步驟。
- `scripts/claude-auth-status.sh [full|json|simple]` - 檢查 Claude Code 與 OpenClaw 的認證狀態。
- `scripts/auth-monitor.sh` - 輪詢狀態，並在權杖即將到期時傳送通知（透過 OpenClaw send 和／或 ntfy.sh）。環境變數：`WARN_HOURS`（預設為 `2`）、`NOTIFY_PHONE`、`NOTIFY_NTFY`。使用隨附的 `scripts/systemd/openclaw-auth-monitor.{service,timer}` 依排程執行（每 30 分鐘一次）。
- `scripts/mobile-reauth.sh` - 重新執行 `claude setup-token`，並輸出可在手機上開啟的 URL，供透過 Termux 使用 SSH 時使用。
- `scripts/termux-quick-auth.sh`、`scripts/termux-auth-widget.sh`、`scripts/termux-sync-widget.sh` - Termux:Widget 指令碼：透過 SSH 連線至主機、顯示狀態快顯通知，並在認證到期時開啟重新驗證主控台／操作說明。

## GitHub 讀取輔助工具

若希望 `gh` 對儲存庫範圍的讀取呼叫使用 GitHub App 安裝權杖，同時讓一般的 `gh` 保持使用你的個人登入來執行寫入操作，請使用 `scripts/gh-read`。

必要環境變數：

- `OPENCLAW_GH_READ_APP_ID`
- `OPENCLAW_GH_READ_PRIVATE_KEY_FILE`

選用環境變數：

- `OPENCLAW_GH_READ_INSTALLATION_ID`：用於略過依儲存庫查找安裝項目
- `OPENCLAW_GH_READ_PERMISSIONS`：以逗號分隔的覆寫值，用於指定要請求的讀取權限子集

儲存庫解析順序：

- `gh ... -R owner/repo`
- `GH_REPO`
- `git remote origin`

範例：

- `scripts/gh-read pr view 123`
- `scripts/gh-read run list -R openclaw/openclaw`
- `scripts/gh-read api repos/openclaw/openclaw/pulls/123`

## 新增指令碼時

- 讓指令碼保持用途明確，並提供文件說明。
- 在相關文件中加入簡短條目（若文件不存在，請建立一份）。

## 相關內容

- [測試](/zh-TW/help/testing)
- [即時測試](/zh-TW/help/testing-live)
