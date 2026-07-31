---
read_when:
    - 你想讓 OpenClaw 代理程式加入 Zoom 會議
    - 你正在為 Zoom 會議的回傳音訊設定 Chrome、BlackHole 或 SoX
summary: Zoom 會議外掛：以 Chrome 瀏覽器訪客身分加入會議
title: Zoom 會議外掛
x-i18n:
    generated_at: "2026-07-26T07:30:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d91e57cccb163f634c6eaee71dd3832fc7b9e783fc5cd02601572b302d0d25e8
    source_path: plugins/zoom-meetings.md
    workflow: 16
---

`zoom-meetings` 外掛會透過 OpenClaw Chrome 設定檔中的 Zoom Web App，以訪客身分加入 Zoom 會議連結。它接受 `zoom.us/j/...` 下的會議連結，以及 `example.zoom.us/j/...` 等帳號子網域。它不會建立會議、撥號加入、使用 Zoom Meeting SDK，也不會擷取音訊／視訊錄影。

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
      "zoom-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

如果不想啟用此外掛，請執行 `openclaw plugins disable zoom-meetings`。

```bash
openclaw zoommeetings setup
openclaw zoommeetings join 'https://zoom.us/j/1234567890'
```

使用 `chromeNode.node` 可在已配對的 macOS 節點上執行 Chrome、BlackHole 和 SoX。該節點必須允許 `zoommeetings.chrome` 和 `browser.proxy`。

## 模式

| 模式         | 行為                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | 即時轉錄會諮詢已設定的 OpenClaw 代理；並由 TTS 回覆。 |
| `bidi`       | 即時語音模型會直接聆聽並回覆。                        |
| `transcribe` | 僅觀察加入，並提供即時字幕轉錄快照。                   |

每種模式都會在獲准加入後啟用 Zoom 即時字幕，讓 OpenClaw 可以
保存會議筆記。`transcript` 動作仍然只會針對 `transcribe`
工作階段傳回有界的即時緩衝區。離開時，OpenClaw 會將持久轉錄稿及其衍生摘要
儲存在共用狀態資料庫中；可使用 [`openclaw transcripts`](/zh-TW/cli/transcripts)
列出或匯出它們。

自動筆記預設為啟用。將 `transcripts.enabled: false` 設為停用，
即可全域停用持久筆記；明確指定的 `transcribe` 模式仍只會公開
其有界的即時尾端內容。

## 訪客加入限制

瀏覽器介面卡會選擇 **Join from browser**、填入訪客名稱、關閉攝影機、依所選模式設定麥克風，然後按一下 **Join**。Zoom Web App 會在 `app.zoom.us` 下執行；此外掛會在導覽前，授予該來源麥克風及揚聲器選擇權限。通話中狀態會使用 Zoom 的 Leave 控制項。等候室、登入、密碼、CAPTCHA 和裝置權限狀態會傳回明確的手動操作原因。

Zoom 主持人和帳號政策可能會停用瀏覽器加入、要求驗證身分或電子郵件、顯示 CAPTCHA，或要求主持人核准加入。請在 OpenClaw Chrome 設定檔中完成該步驟，然後重試狀態或語音。此外掛不會繞過 Zoom 政策。

Zoom Web App 已使用官方 Zoom 測試會議完成實際驗證，涵蓋應用程式插頁、iframe 訪客名稱輸入、加入前的麥克風與攝影機控制、加入、瀏覽器與 macOS 媒體權限、通話中偵測、啟用即時字幕，以及偵測主持人結束會議。等候室和驗證狀態取決於主持人政策；若沒有穩定的 DOM 識別碼，則會保留文字後援機制。

## 工具與閘道介面

`zoom_meetings` 代理工具支援 `join`、`leave`、`status`、`transcript` 和 `speak`。閘道方法使用 `zoommeetings.*` 前綴。節點命令為 `zoommeetings.chrome`。

## 相關內容

- [會議外掛概覽](/zh-TW/plugins/meeting-plugins)
