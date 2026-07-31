---
read_when:
    - 瞭解代理程式首次執行時會發生什麼事
    - 說明啟動程序檔案的位置
    - 偵錯新手引導身分設定
sidebarTitle: Bootstrapping
summary: 用於建立工作區與身分檔案初始內容的代理啟動流程
title: 代理程式啟動設定
x-i18n:
    generated_at: "2026-07-26T08:43:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: efb47e1a6a86d68aef1aa1662fe9c5def9a4e5b45649b84aeb9060bfcba21a5d
    source_path: start/bootstrapping.md
    workflow: 16
---

啟動初始化是首次執行時的必要流程，會為新的代理工作區建立初始內容，並引導代理選擇身分。此流程只會在完成初始設定後、代理第一次正式互動時執行一次。

## 執行內容

首次針對全新工作區執行時（預設為 `~/.openclaw/workspace`），
OpenClaw 會：

- 建立 `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md` 和 `BOOTSTRAP.md` 的初始內容。
- 讓代理依序完成最多三個步驟的誕生流程：詢問你想如何稱呼它、用一句簡短文字描述其靈魂／氛圍，並詢問你想要最精簡的建議外掛組合，還是最大程度的便利性。
- 將雙方確認的身分儲存兩次：寫入 `IDENTITY.md` 和 `SOUL.md`（代理讀取的自我資訊），並透過 `openclaw agents set-identity` 儲存（頻道和使用者介面顯示的資訊）。
- 直接讀取初始設定期間已儲存的應用程式建議，不再重新掃描。官方外掛使用 `openclaw plugins install <id>`；第三方 ClawHub Skills 則仍須明確選擇加入。處理完選擇後，代理會確認已儲存的提議，確保不再詢問。
- 工作區看起來已完成設定後，刪除 `BOOTSTRAP.md`，因此此流程只會執行一次。

只要 `SOUL.md`、`IDENTITY.md` 或 `USER.md` 已不同於其初始範本，或存在 `memory/` 資料夾，工作區就會視為已完成設定。

<Note>
`BOOTSTRAP.md` 涵蓋完整的身分對話。內容請參閱
[BOOTSTRAP.md 範本](/zh-TW/reference/templates/BOOTSTRAP)。
</Note>

## 內嵌與本機模型執行

對於內嵌或本機模型執行，OpenClaw 不會將 `BOOTSTRAP.md` 放入具特殊權限的系統內容中。在主要互動式首次執行期間，仍會透過使用者提示傳入檔案內容，因此無法可靠呼叫 `read` 工具的模型也能完成此流程。如果目前的執行無法安全存取工作區，代理會收到一則簡短的受限啟動初始化說明，而不是一般問候語。

## 略過啟動初始化

若要在已預先建立初始內容的工作區略過此流程，請執行：

```bash
openclaw onboard --skip-bootstrap
```

## 執行位置

啟動初始化一律在閘道主機上執行。如果 macOS 應用程式連線至遠端閘道，工作區及其啟動初始化檔案會位於該遠端機器上，而不是 Mac 上。

<Note>
當閘道在另一部機器上執行時，請在閘道主機上編輯工作區檔案（例如 `user@gateway-host:~/.openclaw/workspace`）。
</Note>

## 相關文件

- macOS 應用程式初始設定：[初始設定](/zh-TW/start/onboarding)
- 工作區配置：[代理工作區](/zh-TW/concepts/agent-workspace)
- 範本內容：[BOOTSTRAP.md 範本](/zh-TW/reference/templates/BOOTSTRAP)
