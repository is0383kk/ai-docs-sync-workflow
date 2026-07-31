---
read_when:
    - 配對或重新連線 Android 節點
    - 偵錯 Android 閘道探索或驗證問題
    - 從遠端 Mac 鏡像或控制 Android 裝置
    - 驗證各用戶端之間的聊天記錄一致性
summary: Android 應用程式（節點）：連線操作手冊 + 連線／聊天／OpenClaw／語音／Canvas 命令介面
title: Android 應用程式
x-i18n:
    generated_at: "2026-07-26T07:55:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a134a678e26924abc24dd107c3feaad9d09e83e3829eef73514c7ef078d578f1
    source_path: platforms/android.md
    workflow: 16
---

<Note>
官方 Android 應用程式可從 [Google Play](https://play.google.com/store/apps/details?id=ai.openclaw.app&hl=en_IN) 取得，也可在支援的 [GitHub Releases](https://github.com/openclaw/openclaw/releases) 中取得已簽署的獨立 APK。它是配套節點，需要執行中的 OpenClaw 閘道。原始碼：[apps/android](https://github.com/openclaw/openclaw/tree/main/apps/android)（[建置說明](https://github.com/openclaw/openclaw/blob/main/apps/android/README.md)）。
</Note>

## 支援概況

- 角色：配套節點應用程式（Android 不託管閘道）。
- 需要閘道：是（透過 WSL2 在 macOS、Linux 或 Windows 上執行）。
- 安裝：[Google Play](https://play.google.com/store/apps/details?id=ai.openclaw.app&hl=en_IN) 或從支援的 [GitHub Release](https://github.com/openclaw/openclaw/releases) 取得 `OpenClaw-Android.apk`；請先參閱閘道的[入門指南](/zh-TW/start/getting-started)，再參閱[配對](/zh-TW/channels/pairing)。
- 閘道：[操作手冊](/zh-TW/gateway) + [設定](/zh-TW/gateway/configuration)。
  - 通訊協定：[閘道通訊協定](/zh-TW/gateway/protocol)（節點 + 控制平面）。
- 當操作者連線具有 `operator.admin`，且閘道支援 `openclaw.chat` 時，**Settings → OpenClaw** 會開啟專用的閘道設定助理。其設定對話會與一般聊天分開，在本機遮蔽機密回覆，且只有在你點選 **Open Chat** 後才會移至聊天。

系統控制（launchd/systemd）位於閘道主機上，請參閱[閘道](/zh-TW/gateway)。

## 同時使用多個閘道工作階段

每個閘道只需配對一次，然後開啟 **Settings → Gateway**。勾號表示目前聚焦的閘道，而每個開關則控制非聚焦閘道的操作者工作階段是否保持連線。當應用程式位於前景時，已啟用的閘道會各自重新連線，因此切換焦點不會中斷其他連線。只有聚焦的閘道能擁有 Android 節點工作階段和裝置功能；這可防止多個閘道同時向同一支手機發出相機、位置、螢幕或通知命令。應用程式離開前景後，Android 可能會暫停次要連線。

## Wear OS 配套應用程式

Wear OS 配套應用程式使用已配對 Android 手機經過驗證的閘道連線；手錶絕不會接收或儲存閘道認證資訊。它可以選取代理程式和工作階段、讀取限定範圍的逐字記錄、傳送文字或語音輸入的回覆、中止進行中的執行、在所選工作階段內啟動即時 Talk，以及連線或中斷已配對手機的閘道。它也提供本機回覆通知、深色或淺色外觀，以及選用的自動語音朗讀回覆功能。代理程式與閘道控制會協商功能，以支援手機與手錶錯開更新。即時 Talk 會透過暫時的 Wear OS Data Layer 通道串流麥克風與播放音訊，並在所選手機、閘道連線或音訊通道中斷時停止。

## 從 Google Play 以外的來源安裝

一般正式版與修正版 GitHub Releases 包含通用的 `OpenClaw-Android.apk` 和 `OpenClaw-Android-SHA256SUMS.txt`。APK 由發行標籤建置、使用 OpenClaw Android 發行金鑰簽署，並附有 GitHub Actions 來源證明。

選擇同時列出這兩項資產的[發行版本](https://github.com/openclaw/openclaw/releases)，然後下載並驗證該確切標籤，再進行側載：

```bash
release_tag=vYYYY.M.PATCH
gh release download "$release_tag" \
  --repo openclaw/openclaw \
  --pattern OpenClaw-Android.apk \
  --pattern OpenClaw-Android-SHA256SUMS.txt
sha256sum --check OpenClaw-Android-SHA256SUMS.txt
gh attestation verify OpenClaw-Android.apk \
  --repo openclaw/openclaw \
  --signer-workflow openclaw/openclaw/.github/workflows/android-release.yml \
  --source-ref "refs/tags/${release_tag}" \
  --deny-self-hosted-runners
```

<Warning>
Google Play 與獨立 APK 安裝使用不同的更新管道，且可能具有不同的簽署身分。切換管道前，Android 可能會要求解除安裝現有應用程式，這會移除其本機應用程式資料。一般更新請固定使用同一個管道。
</Warning>

## 從遠端 Mac 鏡像並控制 Android

[scrcpy](https://github.com/Genymobile/scrcpy) 會將 Android 螢幕鏡像至 macOS 視窗，並透過 Android Debug Bridge（ADB）轉送鍵盤與游標輸入。這是操作者端的工作流程，與 OpenClaw 節點連線分開。當 Android 裝置與 Mac 位於不同位置，但共用私人 Tailscale 網路時，此功能相當實用。

### 開始之前

- 在 Android 裝置與 Mac 上安裝 Tailscale，並將兩者連線至同一個 tailnet。
- 在 Android 上啟用 **Developer options** 和 **USB debugging**。Android 16 將 **Wireless
  debugging** 放在 **Settings > System > Developer options** 下。請參閱 [Android 開發人員
  選項](https://developer.android.com/studio/debug/dev-options)。
- 在 Mac 上安裝 scrcpy 和 ADB：

  ```bash
  brew install scrcpy
  brew install --cask android-platform-tools
  ```

- 第一次連線時，請確保 Android 裝置可供操作。每台 Mac 必須先由 Android 核准其 ADB
  金鑰，才能控制裝置。

### 啟用透過 TCP 的 ADB

進行初始設定時，請使用 USB 將 Android 裝置連接至受信任的電腦，並核准其偵錯提示。然後執行：

```bash
adb devices
adb tcpip 5555
```

現在可以中斷 USB 連線。如果裝置重新啟動或重設偵錯後，連接埠 5555 停止監聽，請重複此本機設定步驟。Android 11 及更新版本也可以使用 **Wireless debugging > Pair device with pairing code** 和 `adb pair` 建立初始信任。

### 僅允許控制端 Mac

使用限制性授權規則的 tailnet，必須明確允許控制端 Mac 連線至 Android 裝置的 TCP 連接埠 5555。請在 tailnet 原則中新增範圍有限的規則，並將範例位址替換為兩台裝置的穩定 Tailscale IP：

```json5
{
  grants: [
    {
      src: ["<remote-mac-tailnet-ip>"],
      dst: ["<android-tailnet-ip>"],
      ip: ["tcp:5555"],
    },
  ],
}
```

如需主機別名與其他選取器，請參閱 [Tailscale 授權規則](https://tailscale.com/docs/reference/syntax/grants)。請勿向公用網際網路開放此連接埠，也不要使用 Funnel 將其公開：經授權的 ADB 用戶端對裝置具有廣泛的控制權。

### 連線並開始鏡像

在遠端 Mac 上：

```bash
adb connect <android-tailnet-ip>:5555
adb devices
scrcpy --serial <android-tailnet-ip>:5555
```

此 Mac 第一次執行 `adb connect` 時，Android 上會顯示授權對話框。請解鎖裝置、確認金鑰指紋，且僅在信任該 Mac 時選取 **Always allow from this computer**。成功的 `adb devices` 項目會以 `device` 結尾；`unauthorized` 表示尚未核准裝置上的提示。

scrcpy 視窗開啟後，可以直接使用，或透過 macOS 螢幕自動化工具（例如 [Peekaboo](https://peekaboo.sh/)）操作它。scrcpy 負責傳輸顯示畫面與輸入；Tailscale 僅提供私人網路路徑。

### 疑難排解

- `Connection timed out`：確認 tailnet 已授權 TCP 5555。成功的 `tailscale ping` 僅能證明對等裝置可達，無法證明原則允許此 TCP 連接埠。請從 Mac 使用
  `nc -vz <android-tailnet-ip> 5555` 測試。
- `unauthorized`：解鎖 Android 並核准遠端 Mac 的 ADB 金鑰，或在 **Wireless debugging > Paired devices** 下移除過時的工作站，然後重新配對。
- `Connection refused`：重新從本機連線，並再次執行 `adb tcpip 5555`。
- 列出多個裝置：保留明確的 `--serial <android-tailnet-ip>:5555` 引數。

完成後，關閉 scrcpy 並中斷 ADB 連線：

```bash
adb disconnect <android-tailnet-ip>:5555
```

## 連線操作手冊

Android 節點應用程式 ⇄（mDNS/NSD + WebSocket）⇄ **閘道**

Android 會直接連線至閘道 WebSocket，並使用裝置配對（`role: node`）。

對於 Tailscale 或公用主機，Android 需要安全端點：

- 建議：使用具有 `https://<magicdns>` / `wss://<magicdns>` 的 Tailscale Serve / Funnel
- 也支援：任何其他具有真正 TLS 端點的 `wss://` 閘道 URL
- 私人 LAN 位址 / `.local` 主機仍支援明文 `ws://`，此外也支援 `localhost`、`127.0.0.1` 和 Android 模擬器橋接器（`10.0.2.2`）；非回送位址設定會自動使用受限的操作者存取權限

### 先決條件

- 閘道正在另一台機器上執行（或可透過 SSH 連線）。
- Android 裝置／模擬器可以連線至閘道 WebSocket：
  - 位於使用 mDNS/NSD 的同一個 LAN，**或**
  - 位於使用廣域 Bonjour / 單點傳播 DNS-SD 的同一個 Tailscale tailnet（請參閱下文），**或**
  - 手動指定閘道主機／連接埠（備援方式）
- Tailnet／公用行動裝置配對**不會**使用原始 tailnet IP 的 `ws://` 端點。請改用 Tailscale Serve 或其他 `wss://` URL。
- 閘道機器上（或透過 SSH）必須提供 `openclaw` 命令列介面，才能核准配對要求。

### 1. 啟動閘道

```bash
openclaw gateway --port 18789 --verbose
```

確認記錄中出現類似以下內容：

- `listening on ws://0.0.0.0:18789`

若要透過 Tailscale 從遠端 Android 存取，建議使用 Serve/Funnel，而不是直接繫結至原始 tailnet：

```bash
openclaw gateway --tailscale serve
```

這會為 Android 提供安全的 `wss://` / `https://` 端點。除非你另外終止 TLS，否則僅設定 `gateway.bind: "tailnet"` 不足以完成首次遠端 Android 配對。

### 2. 驗證探索功能（選用）

從閘道機器執行：

```bash
dns-sd -B _openclaw-gw._tcp local.
```

更多偵錯說明：[Bonjour](/zh-TW/gateway/bonjour)。

如果也設定了廣域探索網域，請與以下結果比較：

```bash
openclaw gateway discover --json
```

此命令會一次顯示 `local.` 和已設定的廣域網域，並使用解析後的服務端點，而非僅使用 TXT 提示。

#### 透過單點傳播 DNS-SD 進行跨網路探索

Android NSD/mDNS 探索無法跨越網路。如果 Android 節點與閘道位於不同網路，但透過 Tailscale 連線，請改用廣域 Bonjour／單點傳播 DNS-SD。對於 tailnet／公用 Android 配對，只有探索功能並不足夠，探索到的路由仍需要安全端點（`wss://` 或 Tailscale Serve）：

1. 在閘道主機上設定 DNS-SD 區域（例如 `openclaw.internal.`），並發布 `_openclaw-gw._tcp` 記錄。
2. 為所選網域設定 Tailscale 分割 DNS，並將其指向該 DNS 伺服器。

詳細資訊與 CoreDNS 設定範例：[Bonjour](/zh-TW/gateway/bonjour)。

### 3. 從 Android 連線

在 Android 應用程式中：

- 應用程式透過**前景服務**（常駐通知）維持閘道連線。
- 開啟 **Connect** 分頁。
- 使用 **Setup Code** 或 **Manual** 模式。
- 如果探索功能遭封鎖，請在 **Advanced controls** 中手動指定主機／連接埠。對於私人 LAN 主機，`ws://` 仍可運作。對於 Tailscale／公用主機，請啟用 TLS，並使用 `wss://` / Tailscale Serve 端點。

首次成功配對後，Android 會在啟動時自動重新連線至目前使用中的已配對閘道（對透過探索找到的閘道採盡力而為方式，且該閘道必須可在網路上被探索到）。

官方設定碼會將 Android 連接為節點，並預設透過 `wss://` 授予完整的閘道操作員
存取權。明文、非迴送的 `ws://` 設定會自動使用受限存取權，
以確保持有者權杖的安全性。**Settings → Gateway** 會顯示 **Full** 或 **Limited** 存取權。若為受限連線，請設定
`wss://` 或 Tailscale Serve，在 Control UI 中或使用
`openclaw qr` 產生新的完整存取權設定碼，接著在該頁面掃描或貼上設定碼並重新連線。想使用精簡設定檔的操作員
可在 Control UI 中選取 **Limited access**，或執行
`openclaw qr --limited`。

### 管理已配對的閘道

應用程式會保存每個已配對閘道的登錄資訊，因此你可以讓操作員工作階段保持連線，並切換焦點而無須重新配對：

- **Settings → Gateway** 會列出已配對的閘道，並標示目前聚焦的閘道。點選項目即可將焦點切換至該閘道；其他已啟用的操作員工作階段仍會保持連線。
- 每個開關控制應用程式位於前景時，對應的非聚焦閘道是否保持連線。聚焦的閘道會維持啟用，並擁有手機的節點連線與裝置功能。
- 配對多個閘道時，**Connect** 分頁會顯示快速切換器。
- 認證資訊、裝置權杖、TLS 信任、聊天記錄與排入佇列的離線訊息會按閘道分別儲存。切換焦點絕不會混用不同閘道之間的狀態，離線時排入佇列的訊息也只會傳送至其原本指定的閘道。
- **Forget** 會移除閘道的登錄項目，以及其認證資訊、裝置權杖、TLS 固定資訊與快取聊天內容。

### 存活狀態信標

通過驗證的節點工作階段連線後，以及應用程式移至背景但前景服務仍保持連線時，Android 會使用 `event: "node.presence.alive"` 呼叫 `node.event`。只有在得知通過驗證的節點裝置身分後，閘道才會將此資訊記錄為已配對節點／裝置中繼資料上的 `lastSeenAtMs`/`lastSeenReason`。

只有當閘道回應包含 `handled: true` 時，應用程式才會將信標計為已成功記錄。較舊的閘道可能會使用 `{ "ok": true }` 確認 `node.event`；此回應具相容性，但不會計為持久的最後出現時間更新。

### 4. 核准配對（命令列介面）

在閘道機器上：

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

配對詳細資訊：[配對](/zh-TW/channels/pairing)。

選用：若 Android 節點一律從嚴格控管的子網路連線，你可以使用明確的 CIDR 或確切 IP，選擇啟用首次節點自動核准：

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

此功能預設停用。它僅適用於未要求任何範圍的全新 `role: node` 配對。操作員／瀏覽器配對，以及任何角色、範圍、中繼資料或公開金鑰變更，仍需要手動核准。

### 5. 驗證節點已連線

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

### 6. 聊天與記錄

Android 的 Chat 分頁支援選取工作階段（預設為 `main`，也可選取其他現有工作階段）：

- 記錄：`chat.history`（已針對顯示進行正規化——會移除行內指令標籤、純文字工具呼叫 XML 承載內容（`<tool_call>`、`<function_call>`、`<tool_calls>`、`<function_calls>` 及其截斷變體），以及洩漏的 ASCII／全形模型控制權杖；會省略僅含靜默權杖的助理資料列，例如內容完全等於 `NO_REPLY` / `no_reply` 的資料列；過大的資料列可能會替換為預留位置）
- 傳送：`chat.send`
- 持久傳送：每次傳送（文字、選取的圖片及語音備忘）都會在嘗試任何網路操作前，記錄至各閘道專屬的裝置端寄件匣，因此即使應用程式終止，也不會遺失已提交的輸入。離線時排入佇列的傳送內容會在重新連線後依序送出，並使用穩定的等冪性金鑰；只有當該輪對話出現在標準 `chat.history` 中時，傳送內容才會從佇列移除——僅有確認回覆不會被視為已送達的證明。結果不明確時（確認回覆遺失、應用程式在傳送途中遭終止，或閘道在逐字稿寫入前重新啟動），會顯示為可見資料列，並提供明確的 **Retry**／**Delete**，而不會自動重新傳送。斜線命令絕不會在重新連線後自動重播，而是停留等待明確重試。佇列設有上限（每個閘道 50 則訊息及 48 MB 的附件位元組），未傳送的資料列會在 48 小時後到期。從未提交的編輯器草稿無法跨處理程序持久保存。
- 推播更新（盡力而為）：`chat.subscribe` -> `event:"chat"`
- 聆聽：長按助理訊息並選擇 **Listen** 即可聆聽；音訊會透過閘道 `tts.speak`，使用已設定的 TTS 提供者鏈進行算繪；若閘道無法算繪音訊，則使用裝置端系統 TTS。切換工作階段、建立新聊天、應用程式進入背景或關閉聊天時，播放都會停止。

### 7. 畫布與相機

#### 閘道畫布主機（建議用於網頁內容）

若要讓節點顯示代理程式可直接在磁碟上編輯的實際 HTML/CSS/JS，請將節點指向閘道畫布主機。

<Note>
節點會從閘道 HTTP 伺服器載入畫布（與 `gateway.port` 使用相同連接埠，預設為 `18789`）。
</Note>

1. 在閘道主機上建立 `~/.openclaw/workspace/canvas/index.html`。
2. 將節點導覽至該位置（LAN）：

```bash
openclaw nodes invoke --node "<Android Node>" --command canvas.navigate --params '{"url":"http://<gateway-hostname>.local:18789/__openclaw__/canvas/"}'
```

Tailnet（選用）：若兩部裝置都位於 Tailscale 上，請使用 MagicDNS 名稱或 tailnet IP 取代 `.local`，例如 `http://<gateway-magicdns>:18789/__openclaw__/canvas/`。

此伺服器會將即時重新載入用戶端注入 HTML，並在檔案變更時重新載入。閘道也會提供 `/__openclaw__/a2ui/`，但 Android 應用程式會將遠端 A2UI 頁面視為僅供算繪。具動作能力的 A2UI 命令會使用隨附、由應用程式擁有的 A2UI 頁面。

畫布命令（僅限前景）：

- `canvas.eval`、`canvas.snapshot`、`canvas.navigate`（使用 `{"url":""}` 或 `{"url":"/"}` 返回預設架構）。`canvas.snapshot` 會傳回 `{ format, base64 }`（預設為 `format="jpeg"`）。
- A2UI：`canvas.a2ui.push`、`canvas.a2ui.reset`（`canvas.a2ui.pushJSONL` 舊版別名）。這些命令會使用隨附、由應用程式擁有的 A2UI 頁面進行具動作能力的算繪。

相機命令（僅限前景；受權限控管）：`camera.snap`（jpg）、`camera.clip`（mp4）。參數與命令列介面輔助程式請參閱[相機節點](/zh-TW/nodes/camera)。

### 8. 語音與擴充的 Android 命令介面

- Android 的殼層導覽項目為 **Home**、**Chat** 和 **Settings**。語音輸入
  位於 Chat 編輯器中；沒有獨立的 Voice 分頁。
- 點選編輯器的麥克風，即可使用裝置端語音辨識並將
  逐字稿插入草稿。長按麥克風可錄製語音備忘
  附件。當語音辨識無法使用、缺少權限、發生忙碌／網路失敗或未偵測到語音時，UI 會回報狀況，
  而不會直接捨棄
  此次嘗試。
- 從 Chat 波形啟動持續 **Talk**。聽寫、語音備忘
  錄製和 Talk 是互斥的麥克風路徑。
- Talk Mode 會在開始擷取前，將現有的前景服務從 `connectedDevice` 提升至 `connectedDevice|microphone`，並在 Talk Mode 停止時將其降級。節點服務使用 `CHANGE_NETWORK_STATE` 宣告 `FOREGROUND_SERVICE_CONNECTED_DEVICE`；Android 14+ 還需要 `FOREGROUND_SERVICE_MICROPHONE` 宣告、`RECORD_AUDIO` 執行階段授權，以及執行階段的麥克風服務類型。
- Android Talk 預設使用原生語音辨識、閘道聊天，以及透過已設定閘道 Talk 提供者使用的 `talk.speak`。只有在 `talk.speak` 無法使用時，才會使用本機系統 TTS。
- 只有當 `talk.realtime.mode` 為 `realtime` 且 `talk.realtime.transport` 為 `gateway-relay` 時，Android Talk 才會使用即時閘道轉送。
- Android 不會通告 `voiceWake` 功能。請使用 Chat 聽寫、
  語音備忘或 Talk 進行語音輸入。
- 其他 Android 命令系列（可用性取決於裝置、權限和使用者設定）：
  - `device.status`、`device.info`、`device.permissions`、`device.health`
  - 僅當 **Settings > Phone Capabilities > Installed Apps** 啟用時，才提供 `device.apps`；預設會列出啟動器中可見的應用程式（傳入 `includeNonLaunchable` 可取得完整清單）。
  - `notifications.list`、`notifications.actions`（請參閱下方的[通知轉送](#notification-forwarding)）
  - `photos.latest`
  - `contacts.search`、`contacts.add`
  - `calendar.events`、`calendar.add`
  - `callLog.search`
  - `sms.search`
  - `motion.activity`、`motion.pedometer`

### 9. 工作區檔案（唯讀）

Home 概覽包含一張 **Files** 卡片，可透過唯讀的 `agents.workspace.list` / `agents.workspace.get` 閘道 RPC 瀏覽作用中代理程式的工作區：支援逐層瀏覽目錄、預覽文字與圖片，以及透過 Android 分享面板匯出。此功能不提供寫入操作，且預覽大小受閘道限制。

## 審查命令核准

具備 `operator.admin` 的操作員連線，或閘道明確指定的已配對
`operator.approvals` 連線，可以在 **Settings -> Approvals** 下審查
待處理的執行要求。應用程式會先載入
閘道已清理的核准記錄，之後才啟用按鈕；畫面會顯示任何
安全性警告，以及該要求所提供的確切決策選項，並將
核准 ID 與擁有者種類提交回閘道。

核准狀態會與 Control UI 及支援的聊天介面共用。
最先提交的答案生效；即使其他介面先回答，Android 仍會顯示該標準結果。
若解析回應遺失或閘道
中斷連線，應用程式會讓該動作維持鎖定，並再次讀取核准內容，
之後才會再次提供決策選項。

早於統一核准方法的閘道會回復使用已發布的
執行專用方法。待處理審查仍可運作，但保留的終端狀態
與更豐富的跨介面結果需要更新版閘道。

## 回答代理程式問題

對於具備 `operator.questions`（或 `operator.admin`）的操作員連線，
Chat 會將待處理的閘道問題顯示為原生卡片。卡片支援單選與
多選選項、選項說明、自由文字 **Other** 回答，以及
到期倒數計時。重新連線時會從閘道重新載入待處理問題。當此裝置回答、
其他介面先回答，或問題到期或遭取消時，
卡片會鎖定。

## 助理進入點

Android 支援從系統助理觸發器（Google Assistant）啟動 OpenClaw。按住主畫面按鈕（或其他 `ACTION_ASSIST` 觸發器）會開啟應用程式；說出 "Hey Google, ask OpenClaw `<prompt>`" 會符合應用程式宣告的 App Actions 查詢模式，並將提示詞帶入聊天編輯器，而不會自動傳送。

此功能使用在應用程式資訊清單中宣告的 Android **App Actions**（`shortcuts.xml` 功能）。不需要進行閘道端設定——助理意圖完全由 Android 應用程式處理。

<Note>
App Actions 的可用性取決於裝置、Google Play Services 版本，以及使用者是否已將 OpenClaw 設為預設助理應用程式。
</Note>

## 通知轉送

Android 可將裝置通知作為 `node.event` 項目轉送至閘道。此功能是在應用程式的 Settings 面板中，**於裝置上**進行設定，而不是在閘道／`openclaw.json` 設定中進行。

| 設定                        | 說明                                                                                                                                                                                              |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 轉送通知事件                | 主開關。預設關閉；必須先授予 Notification Listener Access。                                                                                                                                       |
| 套件篩選器                  | **允許清單**（僅轉送列出的套件 ID）或**封鎖清單**（預設：除列出的 ID 外，轉送所有套件）。在封鎖清單模式下，一律排除 OpenClaw 自有套件，以防止轉送迴圈。 |
| 勿擾時段                    | 會抑制轉送的本機 HH:mm 開始／結束時間範圍。預設停用；啟用後預設為 `22:00`-`07:00`。                                                                                |
| 每分鐘事件上限              | 每部裝置的轉送通知速率限制。預設為 20。                                                                                                                                                            |
| 路由工作階段金鑰            | 選用。將轉送的通知事件固定導入特定工作階段，而非裝置的預設通知路由。                                                                                                                              |

<Note>
通知轉送需要 Android Notification Listener 權限。應用程式會在設定期間提示授予此權限。
</Note>

WhatsApp、WhatsApp Business、Telegram、Telegram X、Discord 和 Signal 通知一律排除。其訊息已由原生 OpenClaw 頻道工作階段管理；若將 Android 通知作為獨立的節點事件轉送，可能會導致回覆經由錯誤的對話傳送。

## 相關內容

- [iOS 應用程式](/zh-TW/platforms/ios)
- [節點](/zh-TW/nodes)
- [Android 節點疑難排解](/zh-TW/nodes/troubleshooting)
