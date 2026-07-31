---
read_when:
    - 編輯 IPC 合約或選單列應用程式 IPC
summary: OpenClaw 應用程式、閘道節點傳輸與 PeekabooBridge 的 macOS IPC 架構
title: macOS IPC
x-i18n:
    generated_at: "2026-07-26T08:32:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39e11af2bb9348d1c1f6e4fe6be95e825d23d5c1aa66e32dae713a89afb12b4f
    source_path: platforms/mac/xpc.md
    workflow: 16
---

# OpenClaw macOS IPC 架構

本機 Unix socket 將節點主機服務連接至 macOS App，以處理執行核准和 `system.run`。另提供 `openclaw-mac` 偵錯命令列介面（`apps/macos/Sources/OpenClawMacCLI`），用於探索／連線檢查；代理程式動作仍透過閘道 WebSocket 和 `node.invoke` 傳遞。由節點支援的 `computer.act` 路徑會在程序內執行嵌入式 Peekaboo 自動化；獨立的 Peekaboo 用戶端則使用 PeekabooBridge。

## 目標

- 由單一 GUI App 執行個體負責所有面向 TCC 的工作（通知、螢幕錄製、麥克風、語音、AppleScript）。
- 精簡的自動化介面：閘道 + 節點命令、程序內 `computer.act`，以及供獨立 UI 自動化用戶端使用的 PeekabooBridge。
- 可預期的權限：始終使用相同的已簽署套件 ID，並由 launchd 啟動，讓 TCC 授權得以保留。

## 運作方式

### 閘道 + 節點傳輸

- App 會執行閘道（本機模式），並以節點身分連線至該閘道。
- 代理程式動作透過 `node.invoke` 執行（例如 `system.run`、`system.notify`、`canvas.*`）。
- 節點命令包括 `canvas.*`、`camera.snap`、`camera.clip`、`screen.snapshot`、`screen.record`、`computer.act`、`system.run` 和 `system.notify`。
- 節點會回報 `permissions` 對應表，讓代理程式得知螢幕、相機、麥克風、語音、自動化或輔助使用的存取權限是否可用。

### 節點服務 + App IPC

- 無頭節點主機服務會連線至閘道 WebSocket。
- `system.run` 請求會透過本機 Unix socket（`ExecApprovalsSocket.swift`）轉送至 macOS App。
- App 會在 UI 情境中執行命令，視需要顯示提示，並傳回輸出。

圖表（SCI）：

```text
代理程式 -> 閘道 -> 節點服務（WS）
                         |  IPC（UDS + 權杖 + HMAC + TTL）
                         v
                  Mac App（UI + TCC + system.run）
```

### PeekabooBridge（UI 自動化）

- 內建的代理程式 `computer` 工具**不會**使用此 socket。已配對的 macOS 節點會在 App 程序中使用嵌入式 Peekaboo 服務完成 `computer.act`。
- UI 自動化使用另一個 UNIX socket（`~/Library/Application Support/OpenClaw/<socket>`）和 PeekabooBridge JSON 通訊協定。
- 主機偏好順序（用戶端）：Peekaboo.app -> Claude.app -> OpenClaw.app -> 本機執行。
- 安全性：橋接主機必須具備允許清單中的 TeamID（隨附的 `PeekabooBridgeHostCoordinator` 會允許固定團隊及 App 自身的簽署團隊）；僅限 DEBUG、允許相同 UID 的逃生機制由 `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` 防護（Peekaboo 慣例）。
- 詳細資訊請參閱：[PeekabooBridge 使用方式](/zh-TW/platforms/mac/peekaboo)。

## 操作流程

- 重新啟動／重新建置：`scripts/restart-mac.sh` 會終止現有執行個體、透過 Swift 重新建置、重新封裝並重新啟動。它會自動偵測可用的簽署身分，若找不到則退回 `--no-sign`；傳入 `--sign` 可強制要求簽署（若沒有可用金鑰則失敗），或傳入 `--no-sign` 強制使用未簽署路徑。在已簽署路徑中，環境內設定的 `SIGN_IDENTITY` 會被取消設定，使 `scripts/codesign-mac-app.sh` 自身的身分自動偵測功能選取憑證。
- 單一執行個體：App 會檢查 `NSWorkspace.runningApplications` 是否存在重複的套件 ID；若找到多個執行個體便會結束（`MenuBar.swift` 中的 `isDuplicateInstance()`）。

## 強化注意事項

- 對所有特殊權限介面，建議要求 TeamID 相符。
- PeekabooBridge：`PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1`（僅限 DEBUG）可能允許相同 UID 的呼叫端，以供本機開發使用。
- 所有通訊皆僅限本機；不會公開任何網路 socket。
- TCC 提示僅由 GUI App 套件發出；請在重新建置時維持已簽署套件 ID 的穩定。
- 執行核准 socket 強化：檔案模式 `0600`、共用權杖、對等端 UID 檢查（`getpeereid`）、HMAC-SHA256 挑戰／回應，以及請求的短 TTL。

## 相關內容

- [macOS App](/zh-TW/platforms/macos)
- [macOS IPC 流程（執行核准）](/zh-TW/tools/exec-approvals-advanced#macos-ipc-flow)
