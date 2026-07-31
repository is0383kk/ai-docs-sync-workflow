---
read_when:
    - 你想讓 OpenClaw 代理加入視訊會議
    - 你正在 Google Meet、Microsoft Teams 會議與 Zoom 會議外掛之間做選擇
    - 你需要共用 Chrome、BlackHole、SoX 或會議模式設定
summary: 選擇並設定參與 Google Meet、Microsoft Teams 或 Zoom 會議的方式
title: 會議外掛
x-i18n:
    generated_at: "2026-07-26T07:48:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f41488de018402e3d5cfd01fa5351cdb6107412477d5d54e2d9e186e0fc8ee94
    source_path: plugins/meeting-plugins.md
    workflow: 16
---

OpenClaw 為 Google Meet、Microsoft Teams 會議和 Zoom 分別提供不同的外掛。三者都可透過 Chrome 加入、使用相同的參與模式，並可在閘道主機或配對節點上執行 Chrome。它們的平台 URL、安裝模式和額外功能各不相同。

這些外掛用於參與會議。它們與 [Microsoft Teams 頻道](/zh-TW/channels/msteams)等訊息頻道以及[語音通話外掛](/zh-TW/plugins/voice-call)彼此獨立。

## 選擇外掛

| 平台        | 外掛                                      | 接受的會議連結                                                                                      | 安裝                                    | 參與方式                                      | 平台特定功能                                                                                |
| --------------- | ------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | ----------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Google Meet     | [`google-meet`](/zh-TW/plugins/google-meet)       | `meet.google.com/...`                                                                                       | 從 npm 或 ClawHub 安裝；預設啟用 | 本機 Chrome、配對節點上的 Chrome，或 Twilio 電話撥入 | 可透過 Meet API 或已登入的瀏覽器建立會議；可使用 OAuth 讀取支援的 Meet 成果資料 |
| Microsoft Teams | [`teams-meetings`](/zh-TW/plugins/teams-meetings) | `teams.microsoft.com/l/meetup-join/...` 下的工作連結和 `teams.live.com/meet/...` 下的消費者連結 | 已內含；預設啟用                    | 本機 Chrome 或配對節點上的 Chrome                  | 以訪客身分加入工作和消費者會議                                                                     |
| Zoom            | [`zoom-meetings`](/zh-TW/plugins/zoom-meetings)   | `zoom.us/j/...` 和 `example.zoom.us/j/...` 等帳戶子網域                                      | 已內含；預設啟用                    | 本機 Chrome 或配對節點上的 Chrome                  | 透過 Zoom Web App 以訪客身分加入                                                                           |

需要建立會議、取得 Google API 成果資料或使用 Twilio 電話路徑時，請選擇 Google Meet。若要在相應平台上直接透過瀏覽器以訪客身分參與，請選擇 Teams 或 Zoom。Teams 和 Zoom 外掛不會建立會議、撥入、呼叫廠商 API，也不會擷取音訊／視訊錄影。

## 選擇模式

這三個外掛共用相同的模式：

| 模式         | 行為                                                                                              | 音訊需求                                      |
| ------------ | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `agent`      | 即時轉錄會傳送至設定的 OpenClaw 代理程式；一般 OpenClaw TTS 會朗讀回覆。  | Chrome 語音回傳需要 BlackHole 和 SoX 橋接。 |
| `bidi`       | 即時語音模型會直接聆聽並回覆。                                                  | Chrome 語音回傳需要 BlackHole 和 SoX 橋接。 |
| `transcribe` | 以僅觀察模式加入，並在平台提供字幕時公開有界的即時字幕逐字稿。 | 不使用 BlackHole 或 SoX 語音回傳橋接。                   |

當代理程式只需要會議文字時，請使用 `transcribe`。若要使用一般 OpenClaw 推理和工具，請使用 `agent`。當低延遲直接語音比透過一般代理程式處理每輪對話更重要時，請使用 `bidi`。

有界的即時逐字稿僅在 `transcribe` 模式下保持可用。在全部
三種模式中，透過瀏覽器加入也會將已完成的字幕列和衍生的
摘要持久保存至共用狀態資料庫。離開會議時會完成可見
字幕並寫入摘要；使用 [`openclaw transcripts`](/zh-TW/cli/transcripts)
列出、檢查或匯出該摘要。這個持久筆記路徑不會變更即時
代理程式諮詢逐字稿，也不會建立音訊／視訊錄影。

自動筆記預設為開啟。設定 `transcripts.enabled: false` 可在全域停用
持久筆記。明確選取的 `transcribe` 工作階段會保留其
有界的即時字幕尾端，但不會寫入持久資料列。字幕是否可用
仍取決於會議平台、帳戶、語言和主持人政策。

## 準備 Chrome 和音訊

Chrome 可在閘道主機或配對節點上執行。遠端 Chrome 節點必須允許 `browser.proxy` 以及平台命令：

| 外掛          | 節點命令           |
| --------------- | ---------------------- |
| Google Meet     | `googlemeet.chrome`    |
| Microsoft Teams | `teamsmeetings.chrome` |
| Zoom            | `zoommeetings.chrome`  |

若要透過 Chrome 使用 `agent` 或 `bidi` 模式，請在 macOS 上執行 Chrome，並在同一台主機上安裝共用音訊相依套件：

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

當 Chrome 在配對節點上執行時，閘道主機仍持有 OpenClaw 代理程式和模型認證資訊。為 `agent` 模式設定即時轉錄供應商和 OpenClaw TTS，或為 `bidi` 模式設定即時語音供應商。平台指南包含供應商和音訊命令選項。

## 安裝或停用外掛

Google Meet 必須另外安裝；安裝後預設啟用。Teams 會議和 Zoom 已內含於 OpenClaw，且預設啟用：

```bash
# 僅限 Google Meet
openclaw plugins install npm:@openclaw/google-meet
```

停用任何你不使用的會議外掛：

```bash
openclaw plugins disable google-meet
openclaw plugins disable teams-meetings
openclaw plugins disable zoom-meetings
```

如果你的外掛管理路徑不會自動重新啟動閘道，請重新啟動閘道。接著，在加入前執行平台設定檢查。

## 驗證並加入

| 平台        | 設定檢查                    | 加入命令                                                                  |
| --------------- | ------------------------------ | ----------------------------------------------------------------------------- |
| Google Meet     | `openclaw googlemeet setup`    | `openclaw googlemeet join 'https://meet.google.com/abc-defg-hij'`             |
| Microsoft Teams | `openclaw teamsmeetings setup` | `openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'` |
| Zoom            | `openclaw zoommeetings setup`  | `openclaw zoommeetings join 'https://zoom.us/j/1234567890'`                   |

任何設定檢查失敗，都應視為該傳輸方式和模式的阻礙因素。若要進行僅觀察的冒煙測試，請選取 `transcribe` 模式，並先確認狀態回報正在通話中的工作階段，再預期出現字幕文字。

對於語音回傳冒煙測試，驗證語音所需的不只是播放命令接受位元組。共用命令配對橋接會將目前輸出產生的有界波形指紋，與從 BlackHole 麥克風擷取路徑返回的音訊相互關聯；若只有輸出位元組計數器增加，或存在不相關的參與者音訊，Google Meet、Teams 和 Zoom 不會回報 `speechOutputVerified: true`。

## 處理平台政策提示

瀏覽器自動化會處理一般的訪客名稱、加入前攝影機和麥克風、加入、通話中和離開控制項。它不會繞過平台或主辦者政策。

- Google Meet 可能要求登入 Google、由主持人准入，或進行瀏覽器權限決定。
- Microsoft Teams 可能要求租用戶登入、電子郵件驗證或由主辦者准入。
- Zoom 可能要求身分驗證、電子郵件驗證、密碼、完成 CAPTCHA 或由主持人准入；帳戶也可能停用瀏覽器加入功能。

當加入或狀態結果回報 `manualActionRequired` 時，請先在同一個 OpenClaw Chrome 設定檔中完成所回報的步驟，再重試。反覆開啟新分頁無法解除帳戶、租用戶、大廳或 CAPTCHA 限制。

只能加入操作人員獲授權可新增代理程式的會議。若當地政策或同意規則要求揭露自動參與、轉錄或合成語音，請告知參與者。

## Discord 語音聊天

[Discord 語音頻道](/zh-TW/channels/discord#voice-channels)無須瀏覽器會議自動化，即可提供原生的純音訊即時對話。OpenClaw 可加入語音頻道、聆聽、透過 OpenClaw 代理程式或即時語音模型處理對話輪次，並說出回覆。即使人們在同一個 Discord 頻道中使用視訊，它也不會傳送或接收攝影機視訊或螢幕分享，因此 Discord 語音是相關的即時對話介面，而不是第四個瀏覽器會議外掛。

## 平台指南

- [Google Meet 外掛](/zh-TW/plugins/google-meet)
- [Microsoft Teams 會議外掛](/zh-TW/plugins/teams-meetings)
- [Zoom 會議外掛](/zh-TW/plugins/zoom-meetings)
- [管理外掛](/zh-TW/plugins/manage-plugins)
- [瀏覽器控制](/zh-TW/tools/browser)
