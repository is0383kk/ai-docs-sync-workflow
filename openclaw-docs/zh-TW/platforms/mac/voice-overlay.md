---
read_when:
    - 調整語音覆疊行為
summary: 喚醒詞與按住說話重疊時的語音浮層生命週期
title: 語音疊加層
x-i18n:
    generated_at: "2026-07-26T08:01:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eef571c3e8d41a97779537b1b373fab25b08f63575b50e5019f6c5fbcb782c52
    source_path: platforms/mac/voice-overlay.md
    workflow: 16
---

# 語音浮層生命週期（macOS）

對象：macOS 應用程式貢獻者。目標：當喚醒詞與按住說話重疊時，讓語音浮層維持可預期的行為。

## 行為

- 如果浮層已因喚醒詞而顯示，且使用者按下快速鍵，快速鍵工作階段會接管現有文字，而非將其重設。按住快速鍵期間，浮層會持續顯示。放開時：若有去除前後空白的文字則傳送，否則關閉。
- 僅使用喚醒詞時，仍會在偵測到靜音後自動傳送；按住說話則會在放開時立即傳送。

## 實作

- `VoiceSessionCoordinator`（`apps/macos/Sources/OpenClaw/VoiceSessionCoordinator.swift`）是作用中語音工作階段的唯一擁有者。它是 `@MainActor @Observable` 單例，而非 actor。API：`startSession`、`updatePartial`、`finalize`、`sendNow`、`dismiss`、`updateLevel`、`snapshot`。每個工作階段都帶有一個 `UUID` 權杖；使用過期或不相符權杖的呼叫會被捨棄。
- `VoiceWakeOverlayController`（`VoiceWakeOverlayController+Session.swift`）會呈現浮層，並透過工作階段權杖，經由協調器將使用者動作（`requestSend`、`dismiss`）傳回。它本身絕不擁有工作階段狀態。
- 按住說話（`VoicePushToTalk.begin()`）會將任何可見的浮層文字接管為 `adoptedPrefix`（透過 `VoiceSessionCoordinator.shared.snapshot()`），因此在喚醒浮層顯示時按下快速鍵，會保留文字並附加新的語音內容。放開時，它最多會等待 1.5 秒以取得最終逐字稿，之後才退回使用目前文字。
- 在 `dismiss` 時，浮層會呼叫 `VoiceSessionCoordinator.overlayDidDismiss`，進而觸發 `VoiceWakeRuntime.refresh(state:)`，使手動按 X 關閉、空白文字關閉，以及傳送後關閉，都會恢復喚醒詞聆聽。
- 統一傳送路徑：如果去除前後空白的文字為空，則關閉；否則 `sendNow` 會播放一次傳送提示音，透過 `VoiceWakeForwarder` 轉送，然後關閉。

## 記錄

語音子系統為 `ai.openclaw`；每個元件都會記錄在各自的類別下：

| 類別                    | 元件                                            |
| ----------------------- | ----------------------------------------------- |
| `voicewake.coordinator` | `VoiceSessionCoordinator`                       |
| `voicewake.overlay`     | `VoiceWakeOverlayController`/`VoiceWakeOverlay` |
| `voicewake.ptt`         | 按住說話快速鍵與擷取                            |
| `voicewake.runtime`     | 喚醒詞執行階段                                  |
| `voicewake.chime`       | 提示音播放                                      |
| `voicewake.sync`        | 全域設定同步                                    |
| `voicewake.forward`     | 逐字稿轉送                                      |
| `voicewake.meter`       | 麥克風音量監控                                  |

## 偵錯檢查清單

- 重現浮層停滯問題時串流記錄：

  ```bash
  sudo log stream --predicate 'subsystem == "ai.openclaw" AND category CONTAINS "voicewake"' --level info --style compact
  ```

- 確認只有一個作用中的工作階段權杖；過期的回呼會由協調器捨棄。
- 確認放開按住說話按鍵時，一律會使用作用中的權杖呼叫 `end()`；如果文字為空，應直接關閉，而不會播放提示音或傳送。

## 相關內容

- [macOS 應用程式](/zh-TW/platforms/macos)
- [語音喚醒（macOS）](/zh-TW/platforms/mac/voicewake)
- [對話模式](/zh-TW/nodes/talk)
