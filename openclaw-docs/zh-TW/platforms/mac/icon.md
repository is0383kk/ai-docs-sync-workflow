---
read_when:
    - 變更選單列圖示行為
summary: macOS 上 OpenClaw 的選單列圖示狀態與動畫
title: 選單列圖示
x-i18n:
    generated_at: "2026-07-26T07:47:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8a38f1253f0c376ef2ce6c0ae339b67084c472c764964bcc7ad21e10133e2b47
    source_path: platforms/mac/icon.md
    workflow: 16
---

# 選單列圖示狀態

範圍：macOS 應用程式（`apps/macos`）。繪製：`CritterIconRenderer.makeIcon(...)`。動畫／狀態連接：`CritterStatusLabel` + `CritterStatusLabel+Behavior.swift`。

## 狀態

| 狀態                  | 觸發條件                                  | 視覺效果                                                                                              |
| --------------------- | ----------------------------------------- | --------------------------------------------------------------------------------------------------- |
| 閒置                  | 預設                                      | 一般眨眼／擺動動畫；睜眼時保留光亮反光                                                                  |
| 暫停                  | `isPaused=true`                        | 觸角下垂（「休息中」）且眼睛睜開；無動作                                                               |
| 睡眠                  | 閘道中斷連線／未設定                      | 觸角下垂，眼睛閉合成 `⌣ ⌣` 眼瞼；無動作                                                  |
| 慶祝                  | 訊息已傳送（`sendCelebrationTick`）          | 眼睛閃現開心的 `∩ ∩` 弧線約 0.9 秒，並踢一下腿                                            |
| 語音喚醒（大耳朵）    | 聽到喚醒詞                                | 觸角豎直並伸得更高（`earScale=1.9`）；靜默後恢復                                                    |
| 工作中                | `isWorking=true` 或作用中的 `IconState` | 腿部以更快速度擺動（`legWiggle` 最高至 `1.0`），並有小幅水平位移；疊加於閒置擺動效果 |

當工作階段有作用中的作業或工具時，可在同一個小動物圖示上方顯示工具活動徽章（SF Symbol 圓形徽章，例如執行使用 `chevron.left.slash.chevron.right`）。該徽章來自 `IconState`/`ActivityKind`；完整狀態模型請參閱[選單列](/zh-TW/platforms/mac/menu-bar)。

## 語音喚醒耳朵

- 觸發：`AppStateStore.shared.triggerVoiceEars(ttl: nil)`，由語音喚醒擷取流水線（`VoiceWakeRuntime`）及語音喚醒偵錯／測試工具（`VoiceWakeTester`、`VoiceWakeOverlayController`）呼叫。
- 停止：`stopVoiceEars()`，在擷取完成時呼叫。
- 完成前的靜默時間窗：一般為 `2.0s`；如果只聽到觸發詞，且之後沒有其他語音（`VoiceWakeRuntime.silenceWindow` / `triggerOnlySilenceWindow`），則為 `5.0s`。
- 增強期間，閒置眨眼／擺動／腿部／耳朵計時器會暫停（`earBoostActive` 會控制 `CritterStatusLabel+Behavior` 中的動畫工作）。

## 形狀與大小

- 畫布：18x18pt 範本影像，繪製至 36x36px 點陣圖後備儲存區（2x），讓圖示在 Retina 顯示器上保持清晰。
- 耳朵縮放比例預設為 `1.0`；語音增強會設定 `earScale=1.9`，但不會變更整體框架。
- `antennaDroop`（0-1）會將觸角向下折疊，形成暫停與睡眠姿勢。
- 腿部疾走使用 `legWiggle`，最高至 `1.0`，並伴隨小幅水平抖動。

## 行為注意事項

- 沒有可控制耳朵或工作狀態的外部命令列介面／代理切換；兩者均由應用程式訊號（`AppState.setWorking`、`AppState.triggerVoiceEars`）在內部驅動，以避免意外反覆切換。
- 任何新的 TTL 都應保持短暫（遠低於 10 秒），讓圖示在作業卡住時也能快速回復基準狀態。

## 相關內容

- [選單列](/zh-TW/platforms/mac/menu-bar)
- [macOS 應用程式](/zh-TW/platforms/macos)
