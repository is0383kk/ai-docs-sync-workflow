---
read_when:
    - 你想讓 OpenClaw 代理程式加入 Microsoft Teams 會議
    - 你正在設定 Chrome、BlackHole 或 SoX，以便在 Teams 會議中進行語音回傳
summary: Microsoft Teams 會議外掛：以 Chrome 瀏覽器訪客身分加入工作或消費者會議
title: Microsoft Teams 會議外掛
x-i18n:
    generated_at: "2026-07-26T07:30:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6f84e58d478185d026dd79a02a8500af48f51689ef6865d56badb0e27c6d2814
    source_path: plugins/teams-meetings.md
    workflow: 16
---

`teams-meetings` 外掛會在 OpenClaw Chrome 設定檔中以來賓身分加入 Microsoft Teams 連結。它接受 `teams.microsoft.com/l/meetup-join/...` 下的工作連結，以及 `teams.live.com/meet/...` 下的消費者連結。它不會建立會議、撥入、呼叫 Microsoft Graph，也不會擷取音訊／視訊錄製內容。

## 設定

語音回覆使用與 [Google Meet 外掛](/zh-TW/plugins/google-meet)相同的本機音訊必要條件：macOS、`BlackHole 2ch` 虛擬音訊裝置，以及 SoX。

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

此外掛預設已包含並啟用。只有在需要自訂時才新增項目，然後檢查設定：

```json5
{
  plugins: {
    entries: {
      "teams-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

如果不希望啟用此外掛，請執行 `openclaw plugins disable teams-meetings`。

```bash
openclaw teamsmeetings setup
openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'
```

使用 `chromeNode.node`，可在已配對的 macOS 節點上執行 Chrome、BlackHole 和 SoX。該節點必須允許 `teamsmeetings.chrome` 和 `browser.proxy`。

## 模式

| 模式         | 行為                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | 即時轉錄會諮詢已設定的 OpenClaw 代理程式；並由 TTS 回覆。 |
| `bidi`       | 即時語音模型會直接聆聽並回覆。                        |
| `transcribe` | 僅觀察加入，並提供即時字幕轉錄快照。                   |

每種模式都會在准許加入後啟用 Teams 即時字幕，讓 OpenClaw 能夠
保存標示發言者的筆記。`transcript` 動作仍只會針對
`transcribe` 工作階段傳回有界的即時緩衝區。離開時，OpenClaw 會將
持久轉錄與衍生摘要儲存在共用狀態資料庫中；可使用 [`openclaw transcripts`](/zh-TW/cli/transcripts)
列出或匯出它們。

自動筆記預設為啟用。將 `transcripts.enabled: false` 設為
停用全域持久筆記；明確的 `transcribe` 模式仍只會公開
其有界的即時尾端內容。

## 來賓加入限制

瀏覽器介接器會關閉應用程式插頁、填入來賓名稱、關閉攝影機、依所選模式設定麥克風，並點擊加入按鈕。通話中狀態會使用掛斷控制項；大廳、租用戶登入及裝置權限狀態會傳回明確的手動操作原因。支援消費者會議啟動器重新導向，以及 Chrome 顯示的 `BlackHole 2ch (Virtual)` 標籤。

Teams 租用戶原則可能要求登入、電子郵件驗證或由召集人准許加入。請在 OpenClaw Chrome 設定檔中完成該步驟，然後重試狀態或語音操作。此外掛不會繞過租用戶原則。

消費者版 Teams 網頁用戶端已針對應用程式插頁、來賓名稱輸入、加入前的麥克風／攝影機切換、加入、大廳准許、媒體權限、通話中偵測、即時字幕、BlackHole 輸入／輸出路由、離開及通話後偵測完成即時驗證。工作租用戶可能套用不同的登入、電子郵件驗證、准許加入及離開確認原則；請在 OpenClaw Chrome 設定檔中完成任何回報的手動操作。

## 工具與閘道介面

`teams_meetings` 代理程式工具支援 `join`、`leave`、`status`、`transcript` 和 `speak`。閘道方法使用 `teamsmeetings.*` 前綴。節點命令為 `teamsmeetings.chrome`。

## 相關內容

- [會議外掛概覽](/zh-TW/plugins/meeting-plugins)
- [Microsoft Teams 頻道](/zh-TW/channels/msteams)
