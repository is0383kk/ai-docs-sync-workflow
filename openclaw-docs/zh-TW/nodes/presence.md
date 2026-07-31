---
read_when:
    - 你希望 OpenClaw 識別使用中的 Mac
    - 你正在偵錯最後輸入活動或作用中節點選擇
    - 你想瞭解節點連線通知的路由方式
summary: 偵測你最近使用的 Mac，並將節點警示傳送到該處
title: 主動電腦存在狀態
x-i18n:
    generated_at: "2026-07-26T07:24:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3f1d1d0e98b1f3b7478cf80696dc693677b57897b07260cce30938e9187c314
    source_path: nodes/presence.md
    workflow: 16
---

主動電腦狀態會告知閘道，哪個已連線的 macOS 節點最近接收到
實體滑鼠或鍵盤輸入。OpenClaw 使用此訊號將其中一台 Mac 標記為 `active`、
為代理程式提供穩定的作用中節點提示，並將節點連線警示路由到你最可能正在使用的電腦。

這與[系統狀態](/zh-TW/concepts/presence)不同，後者是閘道用戶端的即時
名冊；也與持久的 `node.presence.alive` 信標不同，後者會
記錄行動節點上次喚醒的時間，但不會將其視為已連線。

## 需求

- OpenClaw macOS 應用程式已配對，並以節點模式連線。
- 已啟用 **Settings -> Permissions -> Active computer detection**。此功能預設為關閉。
- 已授予簽署過的 OpenClaw 應用程式 **Accessibility** 權限。
- 若要接收連線警示，還需授予 **Notifications** 權限，且
  Mac 節點須公開 `system.notify`。

活動回報目前由原生 macOS 節點實作。iOS、
Android、watchOS 和無頭節點主機可以回報連線或背景
最後出現狀態，但不會競爭主動電腦的指定資格。

## 檢查主動電腦

1. 在 macOS 應用程式中開啟 **Settings -> Permissions**，啟用
   **Active computer detection**，並在 macOS System Settings 中授予 **Accessibility**。
2. 確認 Mac 節點已連線：

   ```bash
   openclaw nodes status --connected
   ```

3. 在該 Mac 上移動滑鼠或按下按鍵，然後執行：

   ```bash
   openclaw nodes status
   openclaw nodes describe --node <node-id-or-name>
   ```

最新且符合資格的 Mac 會標記為 `active`。狀態輸出會顯示其距離上次輸入的
時間；`describe` 會公開 `active`、`lastActiveAtMs` 和 `presenceUpdatedAtMs`。
活動會刻意合併，因此在近期回報後再次輸入，顯示內容最多可能需要約 15
秒才會反映。

## 活動如何成為狀態資訊

macOS 回報器每兩秒取樣一次 HID 系統的閒置時鐘。
節點連線就緒時會回報一次，之後回報較新的實體
活動，頻率不會超過每 15 秒一次。閒置期間，每三分鐘傳送一次
保持連線訊息。閒置時間上限為 30 天，以避免非常舊的取樣
隨時間向前漂移，並錯誤成為最新電腦。

停用 **Active computer detection** 會停止取樣，並透過目前的節點連線
傳送經驗證的清除事件。閘道會立即移除
該 Mac 保留的活動時間戳記，並重新計算主動電腦；
其他節點功能與進行中的工作仍保持連線。若已連線的
閘道版本早於此清除動作，Mac 節點會重新連線一次，讓中斷連線時的
清理程序移除保留的活動資訊。

只有在以下條件全都成立時，閘道才會接受活動：

- 事件屬於該節點 ID 目前已驗證的連線；
- 節點具備有效的 `accessibility: true` 權限；
- 承載內容包含有界整數 `idleSeconds` 值。

閘道會從自身觀察時間減去 `idleSeconds`，以推導出
`lastActiveAtMs`。它絕不信任節點提供的系統時鐘時間戳記。在
已連線且符合資格的 Mac 中，最新的 `lastActiveAtMs` 勝出；若時間相同，則採用最近一次的
狀態更新。

狀態資訊僅限於程序本機，並與連線綁定。中斷目前
工作階段的連線、以使用相同節點 ID 的另一個工作階段取代它，或撤銷
Accessibility，都會清除該節點的活動狀態，並重新計算主動 Mac。

## 隱私權與模型上下文

活動分享預設為關閉，且與用於 UI 自動化的 Accessibility 授權
彼此獨立。OpenClaw 傳送的是閒置時間，而非輸入內容。它不會傳送按鍵值、
滑鼠座標、應用程式名稱、視窗標題或原始輸入事件。
macOS 回報器會讀取硬體 HID 狀態，因此合成的電腦控制
事件不會讓自動化 Mac 看起來像是你實際使用的電腦。

持續活動不會建立面向模型的系統事件。動態
執行階段行只包含已驗證的節點 ID：

```text
active_node=<node-id>
```

確切時間戳記和由節點控制的顯示名稱不會進入提示詞，以
避免提示詞注入和快取頻繁變動。代理程式需要目前詳細資訊時，
可以改用 `nodes` 工具讀取 `node.list` 或 `node.describe`。

## 連線警示的路由方式

節點在核准後完成首次成功的閘道交握後，
OpenClaw 會等待 750 毫秒，讓正在連線的 Mac 提交第一個
活動取樣。接著，它會嘗試向活動最新且支援通知的已連線 Mac
傳送通知。

- 若主要傳送成功，其他 Mac 都不會收到警示。
- 若沒有可用的主動 Mac，或主要傳送失敗，OpenClaw 會等待五
  秒，然後嘗試所有其他公開 `system.notify` 的已連線 Mac。
- 之後的重新連線不會發出通知。閘道會將成功連線
  記錄在配對中繼資料中，因此閘道重新啟動時，不會對每個
  先前已連線的節點重播警示。

警示會綁定至已驗證的節點身分。同一節點的替代工作階段
會接手其待處理的首次連線警示；若傳送時該節點
已不再連線，警示便會取消。

## 疑難排解

| 症狀                                   | 檢查                                                                                                                                                                |
| ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 沒有任何列標記為 `active`                 | 確認已啟用主動電腦偵測、原生 macOS 節點已連線，且 `openclaw nodes describe --node <id>` 顯示 `permissions.accessibility: true`。   |
| 錯誤的 Mac 仍維持主動狀態              | 實際操作該 Mac、等待合併時間窗結束，然後重新執行 `openclaw nodes status`。合成的電腦控制動作不算在內。                        |
| 上次輸入資料消失                | 檢查 Mac 是否中斷連線、其節點工作階段是否遭取代，或 Accessibility 是否遭撤銷。每種情況都會刻意清除活動資訊。                       |
| 警示出現在多台 Mac 上         | 主要傳送無法使用或失敗，因此執行了延遲後援。確認主動 Mac 已連線、允許通知，且公開 `system.notify`。 |
| 代理程式未提及主動 Mac | 活動變更後開始新的對話輪次。執行階段提示穩定且精簡；如需目前的確切中繼資料，請使用 `nodes` 工具。                                    |

如需 TCC 復原資訊，請參閱 [macOS 權限](/zh-TW/platforms/mac/permissions)。如需節點
連線與命令失敗的相關資訊，請參閱[節點疑難排解](/zh-TW/nodes/troubleshooting)。

## 相關內容

- [節點](/zh-TW/nodes)
- [節點命令列介面](/zh-TW/cli/nodes)
- [系統狀態](/zh-TW/concepts/presence)
- [閘道通訊協定](/zh-TW/gateway/protocol#presence)
- [macOS 應用程式](/zh-TW/platforms/macos)
