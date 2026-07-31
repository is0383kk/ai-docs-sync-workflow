---
read_when:
    - 你想讓 OpenClaw 代理程式加入 Google Meet 通話
    - 你想要 OpenClaw 代理程式建立新的 Google Meet 通話
    - 你正在將 Chrome、Chrome 節點或 Twilio 設定為 Google Meet 傳輸方式
summary: Google Meet 外掛：透過 Chrome 或 Twilio 加入明確指定的 Meet URL，並使用代理程式回話預設值
title: Google Meet 外掛
x-i18n:
    generated_at: "2026-07-26T07:48:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8a611e283fe900984a29b563969936a641c7af430b05933eb03b98dc93b5d0c8
    source_path: plugins/google-meet.md
    workflow: 16
---

`google-meet` 外掛會代表 OpenClaw 代理程式加入明確指定的 Meet URL。其範圍刻意保持精簡：

- 它只會加入 `https://meet.google.com/...` URL；絕不會使用自行找到的電話號碼撥入會議。
- `googlemeet create` 可透過 Google Meet API（或瀏覽器備援方式）建立新的 Meet URL，並預設加入該會議。
- Chrome 參與方式會使用已登入的 Chrome 設定檔，也可選擇在已配對的節點上執行。Twilio 參與方式會透過[語音通話外掛](/zh-TW/plugins/voice-call)撥打電話號碼並輸入 PIN/DTMF；它無法直接撥打 Meet URL。
- `mode: "agent"`（預設）會使用即時服務供應商轉錄參與者的語音，將其傳送給已設定的 OpenClaw 代理程式，並使用一般的 OpenClaw TTS 說出回答。`mode: "bidi"` 可讓即時語音模型直接回答。`mode: "transcribe"` 會以僅觀察模式加入，不提供語音回應。
- 外掛加入通話時不會自動播放同意聲明。
- 命令列介面命令是 `googlemeet`；`meet` 保留給更廣泛的代理程式電話會議工作流程。

## 快速開始

安裝外掛與本機音訊相依套件，然後設定即時服務供應商金鑰。在 `agent` 模式下，OpenAI 是預設的轉錄服務供應商；Google Gemini Live 可作為 `bidi` 模式的語音服務供應商：

```bash
openclaw plugins install npm:@openclaw/google-meet
brew install blackhole-2ch sox
export OPENAI_API_KEY=sk-...
# 僅當 bidi 模式的 realtime.voiceProvider 為 "google" 時才需要
export GEMINI_API_KEY=...
```

`blackhole-2ch` 會安裝供 Chrome 路由音訊的 `BlackHole 2ch` 虛擬音訊裝置。Homebrew 安裝程式完成後必須重新啟動，macOS 才會顯示該裝置：

```bash
sudo reboot
```

重新啟動後，確認這兩個元件：

```bash
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

外掛安裝後預設會啟用。只有需要自訂時才新增項目：

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        config: {},
      },
    },
  },
}
```

如果不希望啟用此外掛，請執行 `openclaw plugins disable google-meet`。

檢查設定，然後加入：

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

`setup` 的輸出可供代理程式讀取，並會因模式與傳輸方式而異：它會回報 Chrome 設定檔、節點固定設定；針對即時 Chrome 加入，還會回報 BlackHole/SoX 音訊橋接與延遲開場白檢查。僅觀察加入會略過即時處理的必要條件：

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
```

設定 Twilio 委派後，`setup` 也會回報 `voice-call`、Twilio 認證資訊與公開網路鉤子是否已準備就緒。在代理程式加入之前，應將任何 `ok: false` 檢查視為該傳輸方式／模式的阻擋問題。使用 `--json` 取得機器可讀輸出，並使用 `--transport chrome|chrome-node|twilio` 事先檢查特定傳輸方式：

```bash
openclaw googlemeet setup --transport twilio
```

或讓代理程式透過 `google_meet` 工具加入：

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

在非 macOS 的閘道主機上，`google_meet` 仍可用於成品、行事曆、設定、轉錄、Twilio 與 `chrome-node` 動作，但本機 Chrome 語音回應（搭配 `mode: "agent"` 或 `"bidi"` 的 `transport: "chrome"`）會在抵達音訊橋接前遭到阻擋，因為該路徑目前仰賴 macOS `BlackHole 2ch`。請改用 `mode: "transcribe"`、Twilio 撥入或 macOS `chrome-node` 主機。

### 建立會議

```bash
openclaw googlemeet create --transport chrome-node --mode agent
openclaw googlemeet create --no-join
```

`create` 有兩種路徑，並會在結果的 `source` 欄位中回報：

- **`api`**：設定 Google Meet OAuth 認證資訊時使用。結果具確定性；不依賴瀏覽器 UI 狀態。
- **`browser`**：未設定 OAuth 認證資訊時使用。OpenClaw 會在固定的 Chrome 節點上開啟 `https://meet.google.com/new`，並等待 Google 重新導向至真正的會議代碼 URL；該節點上的 OpenClaw Chrome 設定檔必須已登入 Google。加入與建立都會先重複使用現有的 Meet 分頁（或處理中的 `.../new`／Google 帳戶提示分頁），再開啟新分頁；分頁比對會忽略 `authuser` 這類無害的查詢字串。

`create` 預設會加入，並傳回 `joined: true` 與加入工作階段。傳入 `--no-join`（命令列介面）或 `"join": false`（工具）可只建立 URL。

對於透過 API 建立的會議室，請設定明確的存取原則，而非沿用 Google 帳戶的預設值：

```bash
openclaw googlemeet create --access-type OPEN --transport chrome-node --mode agent
```

| `--access-type` | 無須敲門即可加入的人員                                              |
| --------------- | ------------------------------------------------------------------- |
| `OPEN`          | 任何持有 Meet URL 的人                                              |
| `TRUSTED`       | 主辦者組織中的受信任使用者、受邀的外部使用者，以及撥入使用者 |
| `RESTRICTED`    | 僅限受邀者                                                        |

此設定僅適用於透過 API 建立的會議室，因此必須設定 OAuth。如果你在此選項推出前已完成驗證，請先將 `meetings.space.settings` 範圍新增至 OAuth 同意畫面，再重新執行 `openclaw googlemeet auth login --json`。

如果瀏覽器備援方式遇到 Google 登入或 Meet 權限阻擋，工具會傳回 `manualActionRequired: true`，其中包含 `manualActionReason`、`manualActionMessage`，以及 `browser.nodeId`/`browser.targetId`/`browserUrl`。請回報該訊息，並停止開啟新的 Meet 分頁，直到操作者完成瀏覽器步驟。

### 僅觀察加入

將 `"mode": "transcribe"` 設為略過雙工即時橋接（不需要 BlackHole/SoX，也不提供語音回應）。轉錄模式的 Chrome 加入也會略過 OpenClaw 的麥克風／攝影機權限授予及 Meet **Use microphone** 路徑；如果 Meet 顯示音訊選擇過場畫面，自動化會先嘗試 **Continue without microphone**。受管理的 Chrome 傳輸方式會在所有模式中安裝盡力而為的 Meet 字幕觀察器，以便在不變更即時代理程式諮詢路徑的情況下提供持久化筆記。`googlemeet status --json` 與 `googlemeet doctor` 會回報 `captioning`、`captionsEnabledAttempted`、`transcriptLines`、`lastCaptionAt`、`lastCaptionSpeaker`、`lastCaptionText`，以及 `recentTranscript` 尾端內容。

若要讀取有界限的工作階段逐字稿，請讀取確切追蹤的 Meet 分頁：

```bash
openclaw googlemeet transcript <session-id>
openclaw googlemeet transcript <session-id> --since <next-index> --json
```

觀察器最多會在 Meet 頁面中保留 2,000 行已完成的字幕。可見的漸進式文字會持續保留在狀態健康情形的尾端內容中，直到字幕列完成，因此儲存 `nextIndex` 不會漏掉後續的文字擴充；離開時會先完成可見列，再建立快照。超過上限時，`droppedLines` 會回報從開頭遺失的行數。有界限的 `googlemeet transcript` 尾端內容仍只保留最近結束的四個工作階段，並會隨閘道重設。另外，OpenClaw 會在整場會議期間將已完成的字幕列附加至共用狀態資料庫，並在離開時寫入衍生摘要。使用 [`openclaw transcripts`](/zh-TW/cli/transcripts) 檢查或匯出這些持久化筆記。

自動筆記預設為啟用。將 `transcripts.enabled: false` 設為
停用全域持久化筆記；明確的 `transcribe` 模式仍只會公開
其有界限的即時尾端內容。Twilio 加入沒有瀏覽器字幕串流，因此
不會由此路徑擷取。

若要執行是／否聆聽探測：

```bash
openclaw googlemeet test-listen <meet-url> --transport chrome-node
```

它會以轉錄模式加入，等待新的字幕／逐字稿活動，並傳回 `listenVerified`、`listenTimedOut`、手動動作欄位，以及目前的字幕健康情形。

### 即時工作階段健康情形

在語音回應工作階段期間，`google_meet` 狀態會回報 Chrome／音訊橋接的健康情形：`inCall`、`manualActionRequired`、`providerConnected`、`realtimeReady`、`audioInputActive`、`audioOutputActive`、上次輸入／輸出時間戳記、位元組計數器，以及橋接關閉狀態。受管理的 Chrome 工作階段只會在健康情形回報 `inCall: true` 後說出開場白／測試片語；否則會回報 `speechReady: false` 並阻擋語音嘗試，而不是無聲地不執行任何動作。

本機 Chrome 會透過已登入的 OpenClaw 瀏覽器設定檔加入，且麥克風／喇叭路徑需要 `BlackHole 2ch`。單一 BlackHole 裝置足以進行第一次冒煙測試，但可能產生回音；請使用不同的虛擬裝置或 Loopback 類型的音訊圖，以取得清晰的雙工音訊。

## 本機閘道 + Parallels Chrome

如果只是要在 macOS VM 中提供 Chrome，並不需要在 VM 內執行完整的閘道或設定模型 API 金鑰。請在本機執行閘道與代理程式；在 VM 中執行節點主機。

| 執行位置           | 內容                                                                                            |
| -------------------- | ----------------------------------------------------------------------------------------------- |
| 閘道主機         | OpenClaw 閘道、代理程式工作區、模型／API 金鑰、即時服務供應商、Google Meet 外掛設定 |
| Parallels macOS VM   | OpenClaw 命令列介面／節點主機、Chrome、SoX、BlackHole 2ch、已登入 Google 的 Chrome 設定檔        |
| VM 中不需要 | 閘道服務、代理程式設定、模型服務供應商設定                                             |

安裝 VM 相依套件、重新啟動並確認：

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

在 VM 中安裝外掛（預設會啟用），然後啟動節點主機：

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw node run --host <gateway-host> --port 18789 --display-name parallels-macos
```

如果 `<gateway-host>` 是未使用 TLS 的 LAN IP，請針對該受信任的私人網路選擇允許：

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

安裝為 LaunchAgent 時請使用相同旗標（這是處理程序環境；如果安裝命令中有提供，就會儲存在 LaunchAgent 環境中，而不是 `openclaw.json` 設定）：

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install --host <gateway-lan-ip> --port 18789 --display-name parallels-macos --force
openclaw node restart
```

從閘道主機核准節點，然後確認它同時公告 `googlemeet.chrome` 與瀏覽器功能／`browser.proxy`：

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

將 Meet 路由至該節點：

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["googlemeet.chrome", "browser.proxy"] },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          chrome: {
            guestName: "OpenClaw Agent",
            autoJoin: true,
            reuseExistingTab: true,
          },
          chromeNode: {
            node: "parallels-macos",
          },
        },
      },
    },
  },
}
```

現在可從閘道主機正常加入：

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

若要使用單一命令執行冒煙測試，建立或重複使用工作階段、說出已知片語並輸出工作階段健康情形：

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij
```

在即時加入期間，瀏覽器自動化會填入訪客名稱、按一下 Join/Ask to join，並在 Meet 首次執行的 “Use microphone” 提示出現時接受該提示（若為僅觀察加入和僅透過瀏覽器建立會議，則選擇 “Continue without microphone”）。如果設定檔已登出、Meet 正在等待主持人准入、Chrome 需要麥克風／攝影機權限，或 Meet 卡在尚未處理的提示上，結果會回報 `manualActionRequired: true`，並包含 `manualActionReason` 和 `manualActionMessage`。停止重試，回報該訊息以及 `browserUrl`/`browserTitle`，並僅在手動操作完成後重試。

如果省略 `chromeNode.node`，OpenClaw 僅會在恰好有一個已連線節點同時宣告 `googlemeet.chrome` 和瀏覽器控制能力時自動選取；如果連線了多個具備能力的節點，請指定 `chromeNode.node`（節點 ID、顯示名稱或遠端 IP）。

### 常見失敗檢查

| 症狀                                                  | 修正方式                                                                                                                                                                                                                                                                                   |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Configured Google Meet node ... is not usable: offline` | 已知指定的節點，但目前無法使用。回報設定阻礙；除非使用者要求，否則不要無聲地改用其他傳輸方式。                                                                                                                                                      |
| `No connected Google Meet-capable node`                  | 在虛擬機器中安裝 `npm:@openclaw/google-meet`、執行 `openclaw plugins enable browser`、啟動 `openclaw node run`，並核准配對。如果已明確停用 Google Meet，也請啟用它。確認 `gateway.nodes.commands.allow` 包含 `googlemeet.chrome` 和 `browser.proxy`。 |
| `BlackHole 2ch audio device not found`                   | 在受檢查的主機上安裝 `blackhole-2ch`，然後重新啟動。                                                                                                                                                                                                                         |
| `BlackHole 2ch audio device not found on the node`       | 在虛擬機器中安裝 `blackhole-2ch`，然後重新啟動虛擬機器。                                                                                                                                                                                                                                  |
| Chrome 開啟但無法加入                             | 在虛擬機器中的瀏覽器設定檔登入，或保持設定 `chrome.guestName`。訪客自動加入會透過節點瀏覽器代理使用 OpenClaw 瀏覽器自動化；將節點的 `browser.defaultProfile`（或具名的現有工作階段設定檔）指向要使用的設定檔。                   |
| 重複的 Meet 分頁                                      | 保持設定 `chrome.reuseExistingTab: true`。OpenClaw 會啟用具有相同 URL 的現有分頁；在開啟另一個分頁前，建立流程也會重複使用進行中的 `.../new` 或 Google 帳戶提示分頁。                                                                                        |
| 沒有音訊                                                 | 透過 OpenClaw 使用的虛擬音訊路徑路由 Meet 麥克風／喇叭；使用不同的虛擬裝置或 Loopback 類型的路由，以獲得清晰的雙向音訊。                                                                                                                                |

## 安裝注意事項

Chrome 回傳語音的預設方式使用兩項 OpenClaw 未綁定或重新散布的外部工具；請透過 Homebrew 將它們安裝為主機相依套件：

- `sox`：命令列音訊公用程式。外掛會針對預設的 24 kHz PCM16 音訊橋接器發出明確的 CoreAudio 裝置命令。
- `blackhole-2ch`：提供 `BlackHole 2ch` 裝置的 macOS 虛擬音訊驅動程式，Chrome/Meet 會透過此裝置路由。

SoX 採用 `LGPL-2.0-only AND GPL-2.0-only` 授權；BlackHole 採用 GPL-3.0。如果你建置的安裝程式或設備將 BlackHole 與 OpenClaw 綁定，請檢視 BlackHole 的上游授權，或向 Existential Audio 取得個別授權。

## 傳輸方式

| 傳輸方式     | 使用時機                                                                                     |
| ------------- | -------------------------------------------------------------------------------------------- |
| `chrome`      | Chrome／音訊位於閘道主機上                                                        |
| `chrome-node` | Chrome／音訊位於已配對的節點上（例如 Parallels macOS 虛擬機器）                        |
| `twilio`      | 無法使用 Chrome 參與時，透過語音通話外掛使用電話撥入備援方式 |

### Chrome

透過 OpenClaw 瀏覽器控制開啟 Meet URL，並以已登入的 OpenClaw 瀏覽器設定檔加入。在 macOS 上，外掛會在啟動前檢查 `BlackHole 2ch`，若有設定，則在開啟 Chrome 前執行音訊橋接器健康狀態／啟動命令。對於本機 Chrome，請使用 `browser.defaultProfile` 選擇設定檔；`chrome.browserProfile` 則會傳遞給 `chrome-node` 主機。

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome-node
```

Chrome 麥克風／喇叭音訊會透過本機 OpenClaw 音訊橋接器路由。如果未安裝 `BlackHole 2ch`，加入會因設定錯誤而失敗，而不是在沒有音訊路徑的情況下加入。

### Twilio

委派給[語音通話外掛](/zh-TW/plugins/voice-call)的嚴格撥號方案。它不會剖析 Meet 頁面以取得電話號碼；Google Meet 必須為會議提供電話撥入號碼和 PIN。

請在閘道主機上啟用語音通話，而不是在 Chrome 節點上：

```json5
{
  plugins: {
    allow: ["google-meet", "voice-call", "google"],
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          // 或者，如果 Twilio 應為預設值，請設定為 "twilio"
        },
      },
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          inboundPolicy: "allowlist",
          realtime: {
            enabled: true,
            provider: "google",
            instructions: "以 OpenClaw 代理程式身分加入這場 Google Meet。回答請簡短。",
            toolPolicy: "safe-read-only",
            providers: {
              google: {
                silenceDurationMs: 500,
                startSensitivity: "high",
              },
            },
          },
        },
      },
      google: {
        enabled: true,
      },
    },
  },
}
```

透過環境提供 Twilio 認證資訊，以避免將密鑰存放在 `openclaw.json` 中：

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
export GEMINI_API_KEY=...
```

如果 OpenAI 是即時語音供應商，請改用 `realtime.provider: "openai"` 搭配 `OPENAI_API_KEY`。

啟用 `voice-call` 後，請重新啟動或重新載入閘道；外掛設定變更在重新載入前不會生效。驗證：

```bash
openclaw config validate
openclaw plugins list | grep -E 'google-meet|voice-call'
openclaw googlemeet setup
```

完成 Twilio 委派連接後，`googlemeet setup` 會包含 `twilio-voice-call-plugin`、`twilio-voice-call-credentials` 和 `twilio-voice-call-webhook` 檢查。

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

若要使用自訂序列，請使用 `--dtmf-sequence`，並以開頭的 `w` 或逗號在輸入 PIN 前暫停：

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

## OAuth 與預檢

建立 Meet 連結時 OAuth 為選用項目，因為 `googlemeet create` 可以改用瀏覽器自動化。若要使用官方 API 建立、解析空間或進行 Meet Media API 預檢，請設定 OAuth。Chrome/Chrome-node 加入一律不依賴 OAuth；無論如何，它們都會使用已登入的 Chrome 設定檔、BlackHole/SoX，以及（對於 `chrome-node`）已連線的節點。

### 建立 Google 認證資訊

在 Google Cloud Console 中：

<Steps>
<Step title="建立或選取專案">
</Step>
<Step title="啟用 Google Meet REST API">
</Step>
<Step title="設定 OAuth 同意畫面">
對 Google Workspace 組織而言，Internal 最簡單。External 適用於個人／測試設定；應用程式處於 Testing 狀態期間，請將每個要授權的 Google 帳戶新增為測試使用者。
</Step>
<Step title="新增要求的範圍">
- `https://www.googleapis.com/auth/meetings.space.created`
- `https://www.googleapis.com/auth/meetings.space.readonly`
- `https://www.googleapis.com/auth/meetings.space.settings`
- `https://www.googleapis.com/auth/meetings.conference.media.readonly`
- `https://www.googleapis.com/auth/calendar.events.readonly`（日曆查詢）
- `https://www.googleapis.com/auth/drive.meet.readonly`（逐字稿／智慧筆記文件本文匯出）

</Step>
<Step title="建立 OAuth 用戶端 ID">
應用程式類型為 **Web application**。已授權的重新導向 URI：

```text
http://localhost:8085/oauth2callback
```

</Step>
<Step title="複製用戶端 ID 和用戶端密鑰">
</Step>
</Steps>

`spaces.create` 需要 `meetings.space.created`。`meetings.space.readonly` 會將 Meet URL／代碼解析為空間。`meetings.space.settings` 可讓 OpenClaw 在透過 API 建立會議室時傳遞 `SpaceConfig` 設定，例如 `accessType`。`meetings.conference.media.readonly` 用於 Meet Media API 預檢和媒體作業；若要實際使用 Media API，Google 可能會要求加入 Developer Preview。只有 `--today`/`--event` 日曆查詢需要 `calendar.events.readonly`。只有 `--include-doc-bodies` 匯出需要 `drive.meet.readonly`。如果你只需要透過瀏覽器進行 Chrome 加入，請完全略過 OAuth。

### 核發重新整理權杖

設定 `oauth.clientId`，並視需要設定 `oauth.clientSecret`（或將它們作為環境變數傳遞），然後執行：

```bash
openclaw googlemeet auth login --json
```

這會使用 `http://localhost:8085/oauth2callback` 上的 localhost 回呼執行 PKCE 流程，並輸出包含重新整理權杖的 `oauth` 設定區塊。如果瀏覽器無法連線至本機回呼，請新增 `--manual` 以使用複製／貼上流程：

```bash
OPENCLAW_GOOGLE_MEET_CLIENT_ID="your-client-id" \
OPENCLAW_GOOGLE_MEET_CLIENT_SECRET="your-client-secret" \
openclaw googlemeet auth login --json --manual
```

JSON 輸出：

```json
{
  "oauth": {
    "clientId": "your-client-id",
    "clientSecret": "your-client-secret",
    "refreshToken": "refresh-token",
    "accessToken": "access-token",
    "expiresAt": 1770000000000
  },
  "scope": "..."
}
```

將 `oauth` 物件儲存在外掛設定下：

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          oauth: {
            clientId: "your-client-id",
            clientSecret: "your-client-secret",
            refreshToken: "refresh-token",
          },
        },
      },
    },
  },
}
```

如果你不希望在設定中存放重新整理權杖，請優先使用環境變數；系統會先解析設定，再以環境作為備援。如果你是在支援會議建立、日曆查詢或文件本文匯出之前完成驗證，請重新執行 `openclaw googlemeet auth login --json`，使重新整理權杖涵蓋目前的範圍集合。

### 使用 doctor 驗證 OAuth

```bash
openclaw googlemeet doctor --oauth --json
```

這會檢查 OAuth 設定是否存在，以及重新整理權杖能否核發存取權杖，而不載入 Chrome 執行階段，也不需要已連線的節點。報告只包含狀態欄位（`ok`、`configured`、`tokenSource`、`expiresAt`、檢查訊息），且絕不會印出存取權杖、重新整理權杖或用戶端密鑰。

| 檢查                 | 意義                                                                             |
| -------------------- | -------------------------------------------------------------------------------- |
| `oauth-config`       | 存在 `oauth.clientId` 加上 `oauth.refreshToken`，或快取的存取權杖 |
| `oauth-token`        | 快取的存取權杖仍然有效，或重新整理權杖已核發新的存取權杖    |
| `meet-spaces-get`    | 選用的 `--meeting` 檢查已解析現有的 Meet 空間                       |
| `meet-spaces-create` | 選用的 `--create-space` 檢查已建立新的 Meet 空間                         |

使用會產生副作用的建立檢查，證明 Meet API 已啟用並具有 `spaces.create` 範圍：

```bash
openclaw googlemeet doctor --oauth --create-space --json
```

證明對現有空間具有讀取權限：

```bash
openclaw googlemeet doctor --oauth --meeting https://meet.google.com/abc-defg-hij --json
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
```

這些檢查傳回 `403`，通常表示 Meet REST API 已停用、重新整理權杖缺少必要範圍，或 Google 帳戶無法存取該空間。重新整理權杖錯誤表示需要重新執行 `openclaw googlemeet auth login --json`，並儲存新的 `oauth` 區塊。

瀏覽器備援不需要 OAuth；其 Google 驗證來自所選節點上已登入的 Chrome 設定檔，而不是 OpenClaw 設定。

以下環境變數可作為備援：

- `OPENCLAW_GOOGLE_MEET_CLIENT_ID` 或 `GOOGLE_MEET_CLIENT_ID`
- `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET` 或 `GOOGLE_MEET_CLIENT_SECRET`
- `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` 或 `GOOGLE_MEET_REFRESH_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN` 或 `GOOGLE_MEET_ACCESS_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` 或 `GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT`
- `OPENCLAW_GOOGLE_MEET_DEFAULT_MEETING` 或 `GOOGLE_MEET_DEFAULT_MEETING`
- `OPENCLAW_GOOGLE_MEET_PREVIEW_ACK` 或 `GOOGLE_MEET_PREVIEW_ACK`

### 解析、預先檢查及讀取成品

```bash
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet preflight --meeting https://meet.google.com/abc-defg-hij
```

Meet 建立會議記錄後：

```bash
openclaw googlemeet artifacts --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet attendance --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet export --meeting https://meet.google.com/abc-defg-hij --output ./meet-export
```

使用 `--meeting` 時，`artifacts` 和 `attendance` 預設使用最新的會議記錄；傳入 `--all-conference-records` 可處理每一筆保留的記錄。

行事曆查詢會先從 Google Calendar 解析會議 URL，再讀取成品（需要包含 Calendar 事件唯讀範圍的重新整理權杖）：

```bash
openclaw googlemeet latest --today
openclaw googlemeet calendar-events --today --json
openclaw googlemeet artifacts --event "Weekly sync"
openclaw googlemeet attendance --today --format csv --output attendance.csv
```

`--today` 會在今天的 `primary` 行事曆中搜尋含有 Meet 連結的事件；`--event <query>` 會搜尋相符的事件文字；`--calendar <id>` 會指定非主要行事曆。`calendar-events` 會預覽相符事件，並標示 `latest`/`artifacts`/`attendance`/`export` 將選擇哪一個事件。

如果已知道會議記錄 ID，可直接指定：

```bash
openclaw googlemeet latest --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 --json
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 --json
```

關閉由 API 建立的空間：

```bash
openclaw googlemeet end-active-conference https://meet.google.com/abc-defg-hij
```

這會呼叫 `spaces.endActiveConference`，且需要使用 `meetings.space.created` 範圍的 OAuth，才能操作已授權帳戶可管理的空間。可接受 Meet URL、會議代碼或 `spaces/{id}`，並先將其解析為 API 空間資源。這與 `googlemeet leave` 不同：`leave` 會停止 OpenClaw 的本機／工作階段參與；`end-active-conference` 則要求 Google Meet 結束該空間的進行中會議。

寫入可讀報告：

```bash
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 \
  --format markdown --output meet-artifacts.md
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 \
  --format csv --output meet-attendance.csv
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --zip --output meet-export
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --dry-run
```

當 Google 提供相關資料時，`artifacts` 會傳回會議記錄中繼資料，以及參與者、錄影、逐字稿、結構化逐字稿項目和智慧筆記資源中繼資料。`--no-transcript-entries` 會略過大型會議的項目查詢。`attendance` 會將參與者展開為參與者工作階段資料列，包含首次／最後出現時間、工作階段總時長、遲到／提早離開旗標，並依已登入使用者或顯示名稱合併重複的參與者資源；`--no-merge-duplicates` 會將原始資源分開保留，`--late-after-minutes`/`--early-before-minutes` 則可調整門檻。

`export` 會寫入一個包含 `summary.md`、`attendance.csv`、`transcript.md`、`artifacts.json`、`attendance.json` 和 `manifest.json` 的資料夾。`manifest.json` 會記錄選定的輸入、匯出選項、會議記錄、輸出檔案、數量、權杖來源、使用的任何 Calendar 事件，以及部分擷取警告。`--zip` 也會在資料夾旁寫入可攜式封存檔。`--include-doc-bodies` 會透過 Drive `files.export` 匯出連結的逐字稿／智慧筆記 Google Docs 文字（需要 Drive Meet 唯讀範圍）；若沒有此選項，匯出內容只會包含 Meet 中繼資料和結構化逐字稿項目。部分成品失敗（智慧筆記列出、逐字稿項目或文件本文錯誤）時，會將警告保留在摘要／資訊清單中，而不是讓整個匯出失敗。`--dry-run` 會擷取相同資料並印出資訊清單 JSON，但不建立資料夾或 ZIP。

代理程式會透過 `google_meet` 工具使用相同動作（`export`、搭配 `accessType` 的 `create`、`end_active_conference`、`test_listen`）；請參閱[工具](#tool)。

### 即時冒煙測試

```bash
OPENCLAW_LIVE_TEST=1 \
OPENCLAW_GOOGLE_MEET_LIVE_MEETING=https://meet.google.com/abc-defg-hij \
pnpm test:live -- extensions/google-meet/google-meet.live.test.ts
```

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
openclaw googlemeet test-listen https://meet.google.com/abc-defg-hij --transport chrome-node --timeout-ms 30000
```

| 變數                                                                                                                  | 用途                                                                |
| ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `OPENCLAW_LIVE_TEST=1`                                                                                                    | 啟用受防護的即時測試                                             |
| `OPENCLAW_GOOGLE_MEET_LIVE_MEETING`                                                                                       | 保留的 Meet URL、代碼或 `spaces/{id}`                              |
| `OPENCLAW_GOOGLE_MEET_CLIENT_ID` / `GOOGLE_MEET_CLIENT_ID`                                                                | OAuth 用戶端 ID                                                        |
| `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` / `GOOGLE_MEET_REFRESH_TOKEN`                                                        | 重新整理權杖                                                          |
| `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET`、`OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN`、`OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` | 選用；不含 `OPENCLAW_` 前綴的相同備援名稱也可使用 |

基本成品／出席狀況冒煙測試需要 `meetings.space.readonly` 和 `meetings.conference.media.readonly`。行事曆查詢需要 `calendar.events.readonly`。Drive 文件本文匯出需要 `drive.meet.readonly`。

### 建立範例

```bash
openclaw googlemeet create
```

印出新會議 URI、來源和加入工作階段。使用 OAuth 時會使用 Meet API；若沒有 OAuth，則使用固定 Chrome 節點的已登入設定檔。瀏覽器備援 JSON：

```json
{
  "source": "browser",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

如果瀏覽器備援先遇到 Google 登入或 Meet 權限阻擋，`google_meet` 會傳回結構化詳細資料，而不是純文字字串：

```json
{
  "source": "browser",
  "error": "google-login-required: 請在 OpenClaw 瀏覽器設定檔中登入 Google，然後重試建立會議。",
  "manualActionRequired": true,
  "manualActionReason": "google-login-required",
  "manualActionMessage": "請在 OpenClaw 瀏覽器設定檔中登入 Google，然後重試建立會議。",
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1",
    "browserUrl": "https://accounts.google.com/signin",
    "browserTitle": "Sign in - Google Accounts"
  }
}
```

API 建立 JSON：

```json
{
  "source": "api",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "space": {
    "name": "spaces/abc-defg-hij",
    "meetingCode": "abc-defg-hij",
    "meetingUri": "https://meet.google.com/abc-defg-hij"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

建立時預設會加入，但 Chrome／Chrome 節點仍需要已登入的 Google 設定檔，才能透過瀏覽器加入；若已登出，OpenClaw 會回報 `manualActionRequired: true` 或瀏覽器備援錯誤，並要求操作者完成 Google 登入後再重試。

只有在確認你的 Cloud 專案、OAuth 主體及會議參與者均已加入適用於 Meet 媒體 API 的 Google Workspace Developer Preview Program 後，才能設定 `preview.enrollmentAcknowledged: true`。

## 設定

一般 Chrome 代理程式路徑只需要啟用外掛、BlackHole、SoX、即時供應商金鑰，以及已設定的 OpenClaw TTS 供應商：

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {},
      },
    },
  },
}
```

### 預設值

| 鍵                                | 預設值                                   | 備註                                                                                                                                                                                                              |
| --------------------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `defaultTransport`                | `"chrome"`                       |                                                                                                                                                                                                                   |
| `defaultMode`                | `"agent"`                       | 接受 `"realtime"` 作為 `"agent"` 的舊版別名；新的呼叫端應使用 `"agent"`                                                                                                                   |
| `chromeNode.node`                | 未設定                                   | `chrome-node` 的節點 ID／名稱／IP；可能連接多個具備能力的節點時為必填                                                                                                                                        |
| `chrome.launch`                | `true`                       | 啟動 Chrome 以加入；僅在重複使用已開啟的工作階段時設定 `false`                                                                                                                                        |
| `chrome.audioBackend`                | `"blackhole-2ch"`                       |                                                                                                                                                                                                                   |
| `chrome.guestName`                | `"OpenClaw Agent"`                       | 顯示於已登出的 Meet 訪客畫面                                                                                                                                                                                      |
| `chrome.autoJoin`                | `true`                       | 在 `chrome-node` 上盡力填入訪客名稱並點擊 Join Now                                                                                                                                                           |
| `chrome.reuseExistingTab`                | `true`                       | 啟用現有的 Meet 分頁，而非開啟重複分頁                                                                                                                                                                           |
| `chrome.waitForInCallMs`                | `20000`                       | 在回話開場白觸發前，等待 Meet 分頁回報已進入通話                                                                                                                                                                 |
| `chrome.audioFormat`                | `"pcm16-24khz"`                       | 命令配對音訊格式；`"g711-ulaw-8khz"` 僅適用於輸出電話音訊的舊版／自訂命令配對                                                                                                                                     |
| `chrome.audioBufferBytes`                | `4096`                       | 產生命令配對音訊命令所使用的 SoX 處理緩衝區（為 SoX 預設 8192 位元組緩衝區的一半，以降低管線延遲）；值會限制為至少 17 位元組                                                                                       |
| `chrome.audioInputCommand`                | 產生的 SoX 命令                          | 從 CoreAudio `BlackHole 2ch` 讀取，並以 `chrome.audioFormat` 寫入音訊                                                                                                                                             |
| `chrome.audioOutputCommand`                | 產生的 SoX 命令                          | 以 `chrome.audioFormat` 讀取音訊，並寫入 CoreAudio `BlackHole 2ch`                                                                                                                                                |
| `chrome.bargeInInputCommand`                | 未設定                                   | 選用的本機麥克風命令，寫入帶正負號的 16 位元小端序單聲道 PCM，以便在助理播放期間偵測人聲插話；適用於由閘道託管的命令配對橋接器                                                                                     |
| `chrome.bargeInRmsThreshold`                | `650`                       | 視為人聲中斷的 RMS 音量                                                                                                                                                                                           |
| `chrome.bargeInPeakThreshold`                | `2500`                       | 視為人聲中斷的峰值音量                                                                                                                                                                                           |
| `chrome.bargeInCooldownMs`                | `900`                       | 重複清除中斷之間的最短延遲                                                                                                                                                                                       |
| `mode`（每次請求）    | `"agent"`                       | 回話模式；請參閱[代理與雙向模式](#agent-and-bidi-modes)表格                                                                                                                                                       |
| `realtime.provider`                | `"openai"`                       | 當下方範圍限定欄位未設定時使用的相容性備援                                                                                                                                                                       |
| `realtime.transcriptionProvider`                | `"openai"`                       | `agent` 模式用於即時轉錄的供應商 ID                                                                                                                                                                   |
| `realtime.voiceProvider`                | 未設定                                   | `bidi` 模式用於直接即時語音的供應商 ID；設為 `"google"` 即可使用 Gemini Live，同時讓代理模式的轉錄繼續使用 OpenAI。搭配 `realtime.model` 以選擇特定的 Gemini Live 模型。                         |
| `realtime.toolPolicy`                | `"safe-read-only"`                       | 請參閱[代理與雙向模式](#agent-and-bidi-modes)                                                                                                                                                                     |
| `realtime.instructions`                | 簡短的口語回覆指示                       | 告知模型簡短發言，並使用 `openclaw_agent_consult` 提供更深入的回答                                                                                                                                                      |
| `realtime.introMessage`                | `"Say exactly: I'm here and listening."`                       | 即時橋接器連線時朗讀一次；設為 `""` 可靜默加入                                                                                                                                                      |
| `realtime.agentId`                | `"main"`                       | `openclaw_agent_consult` 使用的 OpenClaw 代理 ID                                                                                                                                                                        |
| `voiceCall.enabled`                | `true`                       | 將 Twilio PSTN 通話、DTMF 與開場問候委派給 Voice Call 外掛                                                                                                                                                        |
| `voiceCall.dtmfDelayMs`                | `12000`                       | 透過 Twilio 播放由 PIN 衍生的 DTMF 序列前的初始等待時間                                                                                                                                                           |
| `voiceCall.postDtmfSpeechDelayMs`                | `5000`                       | Voice Call 啟動 Twilio 通話端後，要求即時開場問候前的延遲                                                                                                                                                         |

`chrome.audioBridgeCommand` 與 `chrome.audioBridgeHealthCommand` 可讓外部橋接器取代 `chrome.audioInputCommand`/`chrome.audioOutputCommand`，完整接管本機音訊路徑；關於哪些模式可以使用它們的限制，請參閱[備註](#notes)。

針對舊版 `realtime.provider: "google"` 結構提供 `openclaw doctor --fix` 遷移：當 `realtime.voiceProvider: "google"` 與 `realtime.transcriptionProvider: "openai"` 尚未設定時，會將該意圖移至這兩個欄位。

### 選用覆寫

```json5
{
  defaults: {
    meeting: "https://meet.google.com/abc-defg-hij",
  },
  browser: {
    defaultProfile: "openclaw",
  },
  chrome: {
    guestName: "OpenClaw Agent",
    waitForInCallMs: 30000,
    bargeInInputCommand: [
      "sox",
      "-q",
      "-t",
      "coreaudio",
      "External Microphone",
      "-r",
      "24000",
      "-c",
      "1",
      "-b",
      "16",
      "-e",
      "signed-integer",
      "-t",
      "raw",
      "-",
    ],
  },
  chromeNode: {
    node: "parallels-macos",
  },
  defaultMode: "agent",
  realtime: {
    provider: "openai",
    transcriptionProvider: "openai",
    voiceProvider: "google",
    model: "gemini-3.1-flash-live-preview",
    agentId: "jay",
    toolPolicy: "owner",
    introMessage: "Say exactly: I'm here.",
    providers: {
      google: {
        speakerVoice: "Kore",
      },
    },
  },
}
```

代理模式的聆聽與發言皆使用 ElevenLabs：

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        modelId: "eleven_v3",
        speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
      },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        config: {
          realtime: {
            transcriptionProvider: "elevenlabs",
            providers: {
              elevenlabs: {
                modelId: "scribe_v2_realtime",
                audioFormat: "ulaw_8000",
                sampleRate: 8000,
                commitStrategy: "vad",
              },
            },
          },
        },
      },
    },
  },
}
```

持續使用的 Meet 語音來自 `tts.providers.elevenlabs.speakerVoiceId`。啟用 TTS 模型覆寫時，代理回覆也可以使用每次回覆的 `[[tts:speakerVoiceId=... model=eleven_v3]]` 指令，但設定是會議的確定性預設值。加入時，記錄會顯示 `transcriptionProvider=elevenlabs`，而每次口語回覆都會記錄 `provider=elevenlabs model=eleven_v3 speakerVoiceId=<voiceId>`。

僅限 Twilio 的設定：

```json5
{
  defaultTransport: "twilio",
  twilio: {
    defaultDialInNumber: "+15551234567",
    defaultPin: "123456",
  },
  voiceCall: {
    gatewayUrl: "ws://127.0.0.1:18789",
  },
}
```

使用 `voiceCall.enabled: true`（預設值）與 Twilio 傳輸時，Voice Call 會先送出 DTMF 序列，再開啟即時媒體串流，接著將已儲存的開場文字用作初始的即時問候。如果未啟用 `voice-call`，Google Meet 仍可驗證並記錄撥號計畫，但無法撥打 Twilio 通話。

將 `voiceCall.gatewayUrl` 保持未設定，以使用本機受信任的閘道執行階段；這會在整個呼叫期間保留發起呼叫的代理程式。已設定的閘道 URL 仍是明確的 WebSocket 目標，且無法驗證外掛來源；非預設代理程式的加入會採取失敗關閉，而不會默默使用其他代理程式。需要依代理程式路由時，請在同一個閘道程序中執行 Google Meet 和 Voice Call。

## 工具

代理程式使用 `google_meet` 工具：

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

| `action`                | 用途                                                                                              |
| ----------------------- | ------------------------------------------------------------------------------------------------- |
| `join`                  | 加入明確指定的 Meet URL                                                                           |
| `create`                | 建立空間（預設也會加入）；支援 `accessType`/`entryPointAccess`                              |
| `status`                | 列出作用中的工作階段，或依 `sessionId` 檢查其中一個                                         |
| `setup_status`          | 執行與 `googlemeet setup` 相同的檢查                                                              |
| `resolve_space`         | 透過 `spaces.get` 解析 URL／代碼／`spaces/{id}`                                        |
| `preflight`             | 驗證 OAuth 與會議解析的必要條件                                                                   |
| `latest`                | 尋找會議的最新會議記錄                                                                            |
| `calendar_events`       | 預覽含有 Meet 連結的 Calendar 活動                                                                |
| `artifacts`             | 列出會議記錄，以及參與者／錄製內容／逐字稿／智慧筆記中繼資料                                      |
| `attendance`            | 列出參與者與參與者工作階段                                                                        |
| `export`                | 寫入成品／出席記錄／逐字稿／資訊清單套件；設定 `"dryRun": true` 可只產生資訊清單                 |
| `recover_current_tab`   | 聚焦／檢查現有的 Meet 分頁，而不開啟新分頁                                                        |
| `transcript`            | 讀取有界限的字幕逐字稿；`sinceIndex` 會從前一個 `nextIndex` 繼續                      |
| `leave`                 | 結束工作階段（Chrome 會按一下 Leave；只關閉其開啟的分頁；Twilio 會掛斷）                           |
| `end_active_conference` | 結束由 API 管理之空間目前進行中的 Google Meet 會議                                                 |
| `speak`                 | 在提供 `sessionId` 和 `message` 後，讓即時代理程式立即說話                         |
| `test_speech`           | 建立／重複使用工作階段、觸發已知語句，並回傳 Chrome 健康狀態                                       |
| `test_listen`           | 建立／重複使用僅觀察工作階段，並等待字幕／逐字稿出現變動                                           |

`test_speech` 一律強制使用 `mode: "agent"` 或 `"bidi"`；若要求以 `mode: "transcribe"` 執行則會失敗，因為僅觀察工作階段無法輸出語音。`speechOutputVerified` 要求即時輸出位元組必須是新的，且在該輸出期間，橋接器的麥克風擷取路徑也必須傳回新的非靜音音訊。重複使用之工作階段的舊輸出或迴路訊號不計入，僅輸出端位元組增加也不再回報為已驗證語音。

對於 Chrome 傳輸，`leave` 會在按下 Meet 的 Leave 通話按鈕後，讓重複使用且由使用者擁有的分頁保持開啟。由 OpenClaw 開啟的分頁會在離開後關閉。

Chrome 在閘道主機上執行時，請使用 `transport: "chrome"`；在已配對的節點上執行時，請使用 `transport: "chrome-node"`。在這兩種情況下，模型供應商與 `openclaw_agent_consult` 都在閘道主機上執行，因此模型認證資訊會留在該處。代理程式模式的記錄會在橋接器啟動時包含解析後的轉錄供應商／模型，並在每次合成回覆後包含 TTS 供應商／模型／語音／輸出格式／取樣率。原始的 `mode: "realtime"` 仍可作為 `mode: "agent"` 的舊版相容別名，但已不再顯示於工具的 `mode` 列舉中。

使用 API 支援的房間及明確存取原則來 `create`：

```json
{
  "action": "create",
  "transport": "chrome-node",
  "mode": "agent",
  "accessType": "OPEN"
}
```

結束已知房間中作用中的會議：

```json
{
  "action": "end_active_conference",
  "meeting": "https://meet.google.com/abc-defg-hij"
}
```

在宣稱會議可用前，先進行聆聽驗證：

```json
{
  "action": "test_listen",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "timeoutMs": 30000
}
```

依需求說話：

```json
{
  "action": "speak",
  "sessionId": "meet_...",
  "message": "Say exactly: I'm here and listening."
}
```

若可用，`status` 會包含 Chrome 健康狀態：

| 欄位                                                                  | 含義                                                                                                                   |
| --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `inCall`                                                              | Chrome 看起來已進入 Meet 通話                                                                                          |
| `micMuted`                                                            | 盡力判斷的 Meet 麥克風狀態                                                                                            |
| `manualActionRequired` / `manualActionReason` / `manualActionMessage` | 語音功能運作前，瀏覽器設定檔需要手動登入、由 Meet 主持人准入、授予權限，或修復瀏覽器控制                              |
| `speechReady` / `speechBlockedReason` / `speechBlockedMessage`        | 目前是否允許受管理的 Chrome 語音；`speechReady: false` 表示 OpenClaw 未傳送介紹／測試語句                              |
| `providerConnected` / `realtimeReady`                                 | 即時語音橋接器狀態                                                                                                     |
| `lastInputAt` / `lastOutputAt`                                        | 最近一次從橋接器接收／傳送至橋接器的音訊                                                                               |
| `audioOutputRouted` / `audioOutputDeviceLabel`                        | Meet 分頁的媒體輸出是否已主動路由至橋接器的 BlackHole 裝置                                                            |
| `lastOutputLoopbackAt` / `outputLoopbackSignalBytes`                  | 波形指紋已在 BlackHole 麥克風擷取路徑上建立關聯的新輸出                                                               |
| `lastOutputLoopbackCorrelation`                                       | 將擷取訊號與目前助理輸出世代建立關聯的相關性分數                                                                       |
| `outputGeneration` / `verifiedOutputGeneration`                       | 單調遞增 ID；相等表示目前輸出而非較舊語句已通過迴路驗證                                                               |
| `lastOutputLoopbackRms` / `lastOutputLoopbackPeak`                    | 最新已驗證迴路擷取區塊的音訊能量診斷                                                                                   |
| `lastSuppressedInputAt` / `suppressedInputBytes`                      | 助理播放期間忽略的迴路輸入                                                                                             |

## 代理程式與雙向模式

| 模式    | 由誰決定回答                  | 語音輸出路徑                         | 適用情境                                              |
| ------- | ----------------------------- | ------------------------------------ | ----------------------------------------------------- |
| `agent` | 已設定的 OpenClaw 代理程式     | 一般 OpenClaw TTS 執行階段           | 你想要「我的代理程式正在會議中」的行為                |
| `bidi`  | 即時語音模型                  | 即時語音供應商的音訊回應             | 你想要延遲最低的對話語音迴路                          |

`agent` 模式：即時轉錄供應商會聽取會議音訊，參與者的最終逐字稿會路由至已設定的 OpenClaw 代理程式，並透過一般 OpenClaw TTS 說出回答。相近的最終逐字稿片段會在諮詢前合併，因此單一口語輪次不會產生數個過時的局部回答；當佇列中的助理音訊仍在播放時，會抑制即時輸入，且在諮詢前會忽略近期類似助理語音的逐字稿回音，以免 BlackHole 迴路使代理程式回應自己的語音。

`bidi` 模式：即時語音模型會直接回答，並可呼叫 `openclaw_agent_consult` 以進行更深入的推理、取得目前資訊，或使用一般 OpenClaw 工具。諮詢工具會在幕後使用近期的會議逐字稿脈絡執行一般 OpenClaw 代理程式，並回傳簡潔的口語回答；在 `agent` 模式中，OpenClaw 會將該回答直接傳送至 TTS；在 `bidi` 模式中，即時語音模型可以將其說出。它使用與 Voice Call 相同的共用諮詢機制。

諮詢預設會針對 `main` 代理程式執行；設定 `realtime.agentId`，可將 Meet 路線指向專用的代理程式工作區、模型預設值、工具原則、記憶與工作階段歷史。代理程式模式的諮詢會使用每場會議專屬的 `agent:<id>:subagent:google-meet:<session>` 工作階段金鑰，因此後續問題可保留會議脈絡，同時繼承一般代理程式原則。當代理程式在代理程式模式下呼叫 `google_meet` 時，諮詢者工作階段會先分叉呼叫者目前的逐字稿，再回答參與者的發言；Meet 工作階段會保持分離，因此會議中的後續問題不會直接變更呼叫者的逐字稿。

`realtime.toolPolicy` 控制諮詢執行：

| 原則             | 行為                                                                                                                             |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `safe-read-only` | 公開諮詢工具；將一般代理程式限制為 `read`、`web_search`、`web_fetch`、`x_search`、`memory_search`、`memory_get` |
| `owner`          | 公開諮詢工具；讓一般代理程式使用其正常工具原則                                                                                    |
| `none`           | 不向即時語音模型公開諮詢工具                                                                                                     |

諮詢工作階段金鑰的範圍限定於每個 Meet 工作階段，因此在同一場會議期間，後續諮詢呼叫會重複使用先前的諮詢脈絡。

在 Chrome 完全加入後，強制執行口語就緒檢查：

```bash
openclaw googlemeet speak meet_... "Say exactly: I'm here and listening."
```

完整的加入並說話冒煙測試：

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "Say exactly: I'm here and listening."
```

## 即時測試檢查清單

將會議交給無人看管的代理程式前：

```bash
openclaw googlemeet setup
openclaw nodes status
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "Say exactly: Google Meet speech test complete."
```

預期的 Chrome-node 狀態：

- `googlemeet setup` 全部顯示綠色，且在 Chrome 節點為預設傳輸方式或已固定某個節點時，包含 `chrome-node-connected`。
- `nodes status` 顯示所選節點已連線，並同時公告 `googlemeet.chrome` 與 `browser.proxy`。
- Meet 分頁成功加入，且 `test-speech` 傳回包含 `inCall: true` 的 Chrome 健康狀態。

對於 Parallels macOS VM 等遠端 Chrome 主機，更新閘道或 VM 後，最簡短且安全的檢查方式如下：

```bash
openclaw googlemeet setup
openclaw nodes status --connected
openclaw nodes invoke \
  --node parallels-macos \
  --command googlemeet.chrome \
  --params '{"action":"setup"}'
```

這可證明閘道外掛已載入、VM 節點已使用目前的權杖連線，且在代理程式開啟實際會議分頁之前，Meet 音訊橋接器已可用。

若要執行 Twilio 煙霧測試，請使用提供電話撥入詳細資訊的會議：

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

預期的 Twilio 狀態：

- `googlemeet setup` 包含顯示綠色的 `twilio-voice-call-plugin`、`twilio-voice-call-credentials` 和 `twilio-voice-call-webhook` 檢查。
- 重新載入閘道後，命令列介面中可使用 `voicecall`。
- 傳回的工作階段包含 `transport: "twilio"` 和一個 `twilio.voiceCallId`。
- `openclaw logs --follow` 顯示先提供 DTMF TwiML，再提供即時 TwiML，接著建立已將初始問候語排入佇列的即時橋接器。
- `googlemeet leave <sessionId>` 會掛斷委派的語音通話。

## 疑難排解

### 代理程式看不到 Google Meet 工具

確認外掛已啟用並重新載入閘道；執行中的代理程式只能看到目前閘道程序所註冊的外掛工具：

```bash
openclaw plugins list | grep google-meet
openclaw googlemeet setup
```

在非 macOS 的閘道主機上，`google_meet` 仍會顯示，但本機 Chrome 的回話動作會在到達音訊橋接器之前遭到封鎖。請使用 `mode: "transcribe"`、Twilio 撥入，或 macOS `chrome-node` 主機，而不要使用預設的本機 Chrome 代理程式路徑。

### 沒有已連線且支援 Google Meet 的節點

在節點主機上：

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw plugins enable browser
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

在閘道主機上：

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

節點必須已連線，並列出 `googlemeet.chrome` 和 `browser.proxy`；閘道設定必須同時允許兩者：

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["browser.proxy", "googlemeet.chrome"] },
    },
  },
}
```

如果 `googlemeet setup` 無法通過 `chrome-node-connected`，或閘道記錄回報 `gateway token mismatch`，請使用目前的閘道權杖重新安裝或重新啟動節點：

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install \
  --host <gateway-lan-ip> \
  --port 18789 \
  --display-name parallels-macos \
  --force
```

接著重新載入節點服務，並再次執行：

```bash
openclaw googlemeet setup
openclaw nodes status --connected
```

### 瀏覽器已開啟，但代理程式無法加入

針對僅觀察加入執行 `googlemeet test-listen`，或針對即時加入執行 `googlemeet test-speech`，接著檢查傳回的 Chrome 健康狀態。如果任一者回報 `manualActionRequired: true`，請向操作人員顯示 `manualActionMessage`，並停止重試，直到瀏覽器動作完成。

常見的手動動作：登入 Chrome 設定檔；由 Meet 主持人帳戶允許訪客加入；在原生提示出現時授予 Chrome 麥克風／攝影機權限；關閉或修復卡住的 Meet 權限對話框。

不要只因 Meet 詢問「Do you want people to hear you in the meeting?」就回報「未登入」；這是 Meet 的音訊選擇過渡畫面。若瀏覽器自動化可用，OpenClaw 會按一下 **Use microphone**，並繼續等待實際的會議狀態；若是僅建立的瀏覽器備援方式，則可能改為按一下 **Continue without microphone**，因為產生 URL 不需要即時音訊路徑。

### 建立會議失敗

設定 OAuth 時，`googlemeet create` 會使用 Meet API `spaces.create`，否則會使用固定的 Chrome 節點瀏覽器。請確認：

- **API 建立**：`oauth.clientId` 和 `oauth.refreshToken`（或相符的 `OPENCLAW_GOOGLE_MEET_*` 環境變數）均存在，且重新整理權杖是在新增建立功能支援後產生；較舊的權杖可能缺少 `meetings.space.created`，因此請重新執行 `openclaw googlemeet auth login --json`。
- **瀏覽器備援方式**：`defaultTransport: "chrome-node"` 和 `chromeNode.node` 指向具備 `browser.proxy` 和 `googlemeet.chrome` 的已連線節點；該節點上的 OpenClaw Chrome 設定檔已登入且可開啟 `https://meet.google.com/new`。
- **瀏覽器備援重試**：開啟新分頁前，重複使用現有的 `.../new` 或 Google 帳戶提示分頁；請重試工具呼叫，而不是手動再開啟另一個分頁。
- **手動動作**：如果工具傳回 `manualActionRequired: true`，請使用 `browser.nodeId`、`browser.targetId`、`browserUrl` 和 `manualActionMessage` 引導操作人員；不要循環重試。
- **音訊選擇過渡畫面**：如果 Meet 顯示「Do you want people to hear you in the meeting?」，請讓分頁保持開啟。OpenClaw 應按一下 **Use microphone** 或（僅建立時）**Continue without microphone**，並繼續等待產生的 URL；如果無法執行，錯誤應提及 `meet-audio-choice-required`，而非 `google-login-required`。

### 代理程式已加入但沒有說話

```bash
openclaw googlemeet setup
openclaw googlemeet doctor
```

將 `mode: "agent"` 用於 STT -> OpenClaw 代理程式 -> TTS 路徑，將 `mode: "bidi"` 用於直接即時語音備援。`mode: "transcribe"` 刻意不啟動回話橋接器。若要進行僅觀察偵錯，請在參與者說話後執行 `openclaw googlemeet status --json <session-id>`，並檢查 `captioning`、`transcriptLines`、`lastCaptionText`。如果 `inCall` 為 true，但 `transcriptLines` 維持 `0`，則可能是 Meet 字幕已停用、安裝觀察器後沒有人說話、Meet 使用者介面已變更，或該會議語言／帳戶無法使用即時字幕。

`googlemeet test-speech` 一律檢查即時路徑，並回報該次叫用是否觀察到橋接器輸出位元組。如果 `speechOutputVerified` 為 false 且 `speechOutputTimedOut` 為 true，即時提供者可能已接受語句，但 OpenClaw 未觀察到新的輸出位元組抵達 Chrome 音訊橋接器。

另請確認：閘道主機上有可用的即時提供者金鑰（`OPENAI_API_KEY` 或 `GEMINI_API_KEY`）；Chrome 主機上可看到 `BlackHole 2ch`；該處存在 `sox`；Meet 麥克風／喇叭已透過虛擬音訊路徑路由（對於本機 Chrome 即時加入，`doctor` 應顯示 `meet output routed: yes`）。

`googlemeet doctor [session-id]` 會輸出工作階段、節點、通話中狀態、手動動作原因、即時提供者連線、`realtimeReady`、音訊輸入／輸出活動、最後音訊時間戳記、位元組計數器和瀏覽器 URL。使用 `googlemeet status [session-id] --json` 取得原始 JSON，並使用 `googlemeet doctor --oauth`（加上 `--meeting` 或 `--create-space`）在不暴露權杖的情況下驗證 OAuth 重新整理。

如果代理程式逾時且 Meet 分頁已開啟，請在不開啟另一個分頁的情況下檢查：

```bash
openclaw googlemeet recover-tab
openclaw googlemeet recover-tab https://meet.google.com/abc-defg-hij
```

對應的工具動作是 `recover_current_tab`：它會針對所選傳輸方式（`chrome` 使用本機瀏覽器控制，`chrome-node` 使用已設定的節點）聚焦並檢查現有的 Meet 分頁，而不開啟新的分頁或工作階段，並回報目前的阻礙（登入、允許加入、權限、音訊選擇狀態）。命令列介面命令會與已設定的閘道通訊，因此閘道必須正在執行；`chrome-node` 還要求節點已連線。

### Twilio 設定檢查失敗

當 `voice-call` 未獲允許或未啟用時，`twilio-voice-call-plugin` 會失敗：請將其加入 `plugins.allow`、啟用 `plugins.entries.voice-call`，然後重新載入閘道。

當 Twilio 後端缺少帳戶 SID、驗證權杖或來電號碼時，`twilio-voice-call-credentials` 會失敗：

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
```

當 `voice-call` 沒有公開的網路鉤子對外端點，或 `publicUrl` 指向迴路／私人網路空間時，`twilio-voice-call-webhook` 會失敗。不要將 `localhost`、`127.0.0.1`、`0.0.0.0`、`10.x`、`172.16.x`-`172.31.x`、`192.168.x`、`169.254.x`、`fc00::/7` 或 `fd00::/8` 用作 `publicUrl`；電信業者回呼無法連線到這些位址。請將 `plugins.entries.voice-call.config.publicUrl` 設為公開 URL，或設定通道／Tailscale 對外端點：

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          fromNumber: "+15550001234",
          publicUrl: "https://voice.example.com/voice/webhook",
        },
      },
    },
  },
}
```

進行本機開發時，請使用通道或 Tailscale 對外端點，而不是私人主機 URL：

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tunnel: { provider: "ngrok" },
          // 或
          tailscale: { mode: "funnel", path: "/voice/webhook" },
        },
      },
    },
  },
}
```

重新啟動或重新載入閘道，接著執行：

```bash
openclaw googlemeet setup --transport twilio
openclaw voicecall setup
openclaw voicecall smoke
```

`voicecall smoke` 預設只檢查就緒狀態。針對特定號碼執行模擬測試：

```bash
openclaw voicecall smoke --to "+15555550123"
```

只有在刻意撥出實際通話時，才加入 `--yes`：

```bash
openclaw voicecall smoke --to "+15555550123" --yes
```

### Twilio 通話已開始，但始終未進入會議

確認 Meet 活動提供電話撥入詳細資訊，並傳入確切的撥入號碼及 PIN，或自訂 DTMF 序列：

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

在 `--dtmf-sequence` 中使用開頭的 `w` 或逗號，以便在輸入 PIN 前暫停。

如果通話已建立，但 Meet 參與者名單始終未顯示撥入參與者：

- `openclaw googlemeet doctor <session-id>`：確認委派的 Twilio 通話 ID、DTMF 是否已排入佇列，以及是否已要求播放介紹問候語。
- `openclaw voicecall status --call-id <id>`：確認通話仍在進行中。
- `openclaw voicecall tail`：確認 Twilio 網路鉤子正抵達閘道。
- `openclaw logs --follow`：尋找 Twilio Meet 流程序列：Google Meet 委派加入動作，Voice Call 儲存並提供連線前 DTMF TwiML，Voice Call 為 Twilio 通話提供即時 TwiML，接著 Google Meet 使用 `voicecall.speak` 要求播放介紹語音。
- 重新執行 `openclaw googlemeet setup --transport twilio`；設定檢查必須顯示綠色，但這不代表會議 PIN 序列一定正確。
- 確認撥入號碼與 PIN 屬於同一份 Meet 邀請及同一區域。
- 如果 Meet 接聽速度較慢，或在傳送連線前 DTMF 後，通話逐字稿仍顯示 PIN 提示，請將 `voiceCall.dtmfDelayMs` 從預設的 12 秒調高。
- 如果參與者已加入但你沒有聽到問候語，請檢查 `openclaw logs --follow` 中 DTMF 後的 `voicecall.speak` 要求，以及媒體串流 TTS 播放或 Twilio `<Say>` 備援。如果逐字稿仍顯示「enter the meeting PIN」，表示電話端尚未加入 Meet 會議室，因此參與者不會聽到語音。

如果網路鉤子未送達，請先偵錯語音通話外掛：供應商必須能連線至 `plugins.entries.voice-call.config.publicUrl` 或已設定的通道。請參閱[語音通話疑難排解](/zh-TW/plugins/voice-call#troubleshooting)。

## 注意事項

Google Meet 的官方媒體 API 以接收為主，因此若要在通話中說話，仍需要參與者路徑。此外掛會明確保留此界線：Chrome 負責瀏覽器參與及本機音訊路由；Twilio 負責電話撥入參與。

Chrome 回話模式需要 `BlackHole 2ch`，以及下列其中一項：

- `chrome.audioInputCommand` 加上 `chrome.audioOutputCommand`：OpenClaw 擁有橋接器，並在這些命令與所選供應商之間透過 `chrome.audioFormat` 傳送音訊。`agent` 模式使用即時轉錄加上一般 TTS；`bidi` 模式使用即時語音供應商。預設路徑為搭配 `chrome.audioBufferBytes: 4096` 的 24 kHz PCM16；8 kHz G.711 mu-law 仍可供舊版命令配對使用。
- `chrome.audioBridgeCommand`：外部橋接命令擁有整個本機音訊路徑，並且必須在啟動或驗證其常駐程式後結束。僅適用於 `bidi`，因為 `agent` 模式需要直接存取命令配對以進行 TTS。

使用命令配對式 Chrome 橋接器時，`chrome.bargeInInputCommand` 可以監聽另一支本機麥克風，並在人類開始說話時清除助理播放內容。如此一來，即使共用的 BlackHole 回送輸入在助理播放期間暫時受到抑制，也能讓人類語音優先於助理輸出。與 `chrome.audioInputCommand`/`chrome.audioOutputCommand` 相同，這是由操作人員設定的本機命令：請使用明確且受信任的命令路徑或引數清單，絕不可使用來自不受信任位置的指令碼。

若要獲得清晰的雙工音訊，請透過不同的虛擬裝置或 Loopback 類型的虛擬裝置圖來路由 Meet 輸出與 Meet 麥克風；單一共用的 BlackHole 裝置可能會將其他參與者的聲音回音至通話中。

`googlemeet speak` 會觸發 Chrome 工作階段的作用中回話音訊橋接器；`googlemeet leave` 會將其停止（若為透過語音通話委派的 Twilio 工作階段，也會掛斷底層通話）。若為 API 管理的空間，請使用 `googlemeet end-active-conference` 一併關閉作用中的 Google Meet 會議。

## 相關內容

- [會議外掛概覽](/zh-TW/plugins/meeting-plugins)
- [語音通話外掛](/zh-TW/plugins/voice-call)
- [對話模式](/zh-TW/nodes/talk)
- [建置外掛](/zh-TW/plugins/building-plugins)
