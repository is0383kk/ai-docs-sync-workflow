---
read_when:
    - 配對或重新連接 iOS 節點
    - 啟用或疑難排解直接連線的 Apple Watch 節點
    - 從原始碼執行 iOS 應用程式
    - 偵錯閘道探索或畫布命令
summary: iOS 節點應用程式：連線至閘道、配對、畫布與疑難排解
title: iOS 應用程式
x-i18n:
    generated_at: "2026-07-26T07:47:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2b01a63fa1e2c445f7fb35843536f7f5918e94bfe885dac19c852d7d52d86342
    source_path: platforms/ios.md
    workflow: 16
---

可用性：針對特定版本啟用時，iPhone App 建置版本會透過 Apple 管道發布。本機開發建置版本也可以從原始碼執行。

## 功能

- 透過 WebSocket 連線至閘道（區域網路或 tailnet）。
- 提供節點功能：Canvas、螢幕快照、相機擷取、位置、對話模式、語音喚醒，以及可選用的健康摘要。
- 接收 `node.invoke` 命令並回報節點狀態事件。
- 從代理程式介面（檔案）以唯讀方式瀏覽所選代理程式的工作區：逐層瀏覽目錄、檢視具語法醒目提示的文字預覽與圖片預覽，以及透過分享表單匯出。不支援寫入操作；閘道會限制預覽大小。
- 為每個已配對的閘道保留近期聊天工作階段與文字記錄的小型唯讀離線快取：冷啟動時會立即顯示最後已知的文字記錄，並在閘道回應後重新整理；中斷連線時仍可瀏覽近期聊天；重設／忘記操作會清除受保護的本機快取。
- 將中斷連線期間傳送的文字訊息排入各閘道專屬的持久寄件匣（最多 50 則）：已排入佇列的訊息泡泡會顯示在文字記錄中；重新連線後依序送出，並以冪等方式重試；在標準歷程確認訊息已傳送前會持續保留；先採用退避機制重試，之後才顯示重試／刪除操作；離線超過 48 小時後會過期而不再傳送；重設／忘記操作會連同快取一起清除佇列。
- 聊天是唯一的文字與語音介面。聊天操作可在不離開聊天的情況下開啟完整的工作階段畫面，並可顯示或隱藏助理的推理與工具活動。點一下麥克風可進行草稿聽寫；開啟其選單可錄製語音訊息；也可使用行內的對話控制項進行即時語音互動。聆聽或說話時，對話控制項會依即時麥克風或播放音量顯示動畫。
- 當操作員連線具有 `operator.admin`，且閘道支援 `openclaw.chat` 時，**Settings -> OpenClaw** 會開啟專用的閘道設定助理。其設定對話會與一般聊天分開、在本機遮蔽含有祕密資訊的回覆，而且只有在你點選 **Open Chat** 後才會移至聊天。
- 可按需朗讀助理訊息：在聊天中長按訊息並選擇 **Listen**。App 會使用已設定的 TTS 供應商播放閘道支援的 `tts.speak` 音訊片段；若閘道音訊無法使用或播放，則改用裝置端語音。切換工作階段或 App 進入背景時會停止播放。

## 需求

- 閘道需在另一台裝置上執行（macOS、Linux，或透過 WSL2 執行的 Windows）。
- 網路路徑：
  - 透過 Bonjour 使用相同區域網路，**或**
  - 透過單播 DNS-SD 使用 tailnet（範例網域：`openclaw.internal.`），**或**
  - 手動指定主機／連接埠（備援方式）。

## 快速開始（配對與連線）

首次啟動時，App 會逐步顯示簡短的配對說明和
權限頁面（通知、相機、麥克風、照片、聯絡人、
行事曆、提醒事項、位置）。所有授權均為選用，之後可在
**Settings** -> **Permissions** 或 iOS「設定」App 中變更。

1. 啟動已驗證身分且具有手機可連線路徑的閘道。建議使用 Tailscale
   Serve 作為遠端路徑：

```bash
openclaw gateway --port 18789 --tailscale serve
```

對於受信任的相同區域網路設定，請改用已驗證身分的 `gateway.bind: "lan"`。
預設的迴路位址繫結無法由手機存取。如果閘道
尚未設定，請先執行 `openclaw onboard`，讓設定碼
建立程序可使用權杖或密碼驗證路徑。

2. 開啟[控制介面](/zh-TW/web/control-ui)，選取 **Nodes**，然後在
   **Devices** 頁面上按一下 **Pair mobile device**。建議使用完整存取權，
   且預設會選取此選項；只有在你想排除閘道管理控制功能時，
   才選擇 Limited access，然後按一下 **Create setup code**。

3. 在 iOS App 中開啟 **Settings** -> **Gateway**，掃描 QR Code（或貼上
   設定碼），然後連線。

   如果設定碼同時包含區域網路和 Tailscale Serve 路徑，App
   會依序探測，並儲存第一個可連線的端點。

   已配對的閘道會保留在 **Gateways** 清單中。勾號表示
   目前聚焦的閘道；使用其他列上的閃電控制項，可同時維持其
   操作員工作階段連線。切換焦點不會中斷其他已啟用閘道的
   連線。只有聚焦的閘道會接收具備 iPhone 功能的節點工作階段，
   因此相機、螢幕、位置及其他裝置命令永遠只有一個明確擁有者。
   App 進入背景後，iOS 可能會暫停這些前景連線。

4. 官方 App 會自動連線。如果 **Pending approval** 顯示
   要求，請先檢閱其角色和範圍再核准。

   **Settings → Gateway** 會顯示已儲存的操作員連線具有
   **Full** 或 **Limited** 存取權。為確保持有人權杖安全性，明文區域網路
   `ws://` 設定會自動限制存取權。如果存取權受限，請設定 `wss://`
   或 Tailscale Serve，從控制介面或 `openclaw qr` 掃描新的完整存取設定碼，
   然後重新連線以啟用設定與升級。

控制介面按鈕需要已有具備 `operator.admin` 的配對工作階段。
如需終端機備援方式，請在 iOS App 中選擇探索到的閘道（或啟用
Manual Host 並輸入主機／連接埠），然後在閘道主機上核准要求：

```bash
openclaw devices list
openclaw devices approve <requestId>
```

如果 App 使用已變更的驗證詳細資料（角色／範圍／公開金鑰）重試配對，先前待處理的要求會被取代，並建立新的 `requestId`。核准前請再次執行 `openclaw devices list`。

選用：如果 iOS 節點一律從嚴格控管的子網路連線，你可以使用明確的 CIDR 或確切 IP，選擇啟用首次節點自動核准：

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

此功能預設為停用。它僅適用於未要求任何範圍的新 `role: node` 配對。操作員／瀏覽器配對，以及任何角色、範圍、中繼資料或公開金鑰變更，仍需手動核准。

5. 驗證連線：

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

## 健康摘要

iOS 節點可傳回目前
日曆日的選用唯讀 HealthKit 彙總資料。iOS 裝置同意與明確的閘道命令授權是
彼此獨立的關卡。請參閱 [HealthKit 摘要](/zh-TW/platforms/ios-healthkit)，瞭解
設定、叫用、承載資料欄位、隱私權行為與疑難排解。

預設情況下，Apple Watch 隨附 App 會繼續使用現有的 iPhone 中繼，
不需要另行與閘道配對。請在 Apple 的 Watch App 中將 Apple Watch 與 iPhone 配對，
從 **Watch app -> My Watch -> Available
Apps** 安裝 OpenClaw，然後分別在兩台裝置上開啟一次 OpenClaw。

## 檢閱命令核准

具有 `operator.admin` 的操作員連線，或由閘道明確指定的已配對
`operator.approvals` 連線，可以在 iPhone 上檢閱
待處理的執行要求。核准卡片會顯示閘道已清理的
命令預覽、警告、主機內容、到期時間，以及該要求提供的
決策選項。已配對的 Apple Watch 會透過現有的 iPhone 中繼接收相同的
檢閱者安全提示，並提供精簡的僅允許一次／拒絕決策選項。直接 Apple Watch 閘道模式不會傳送
核准提示。

核准狀態會與控制介面及支援的聊天介面共用。
第一個提交的答案會生效。當其他介面解決要求後、收到遠端
已解決通知後，以及解決確認可能遺失時，iPhone 和 Apple Watch 都會擷取閘道的標準
終止記錄。在該回讀確認
要求是否仍處於待處理狀態之前，操作會維持不可用。

核准擁有權會繫結至所選的閘道。切換閘道無法
將舊提示套用至替代連線。早於統一核准方法的閘道
會退回使用已發布的執行專用方法；
保留的終止狀態和更豐富的跨介面結果需要更新後的
閘道。

## 回答代理程式問題

對於具有 `operator.questions`（或 `operator.admin`）的操作員連線，
聊天會將待處理的閘道問題顯示為原生卡片。卡片支援單選和
多選選項、選項說明、自由文字 **Other** 答案，以及
到期倒數計時。重新連線時會從閘道重新載入待處理問題。當此裝置回答、
其他介面先回答，或問題到期或取消時，卡片
會鎖定。

## 選用的直接 Apple Watch 節點

直接模式會為 Apple Watch 提供自己的簽署節點身分和閘道連線。
當 OpenClaw 處於使用中時，即使已配對的 iPhone 無法使用，
支援的節點命令仍可透過 Apple Watch 的 Wi-Fi 或行動網路運作。

需求：

- iPhone 已使用 `operator.admin` 範圍連線至閘道。
- 設定碼會公布具有 watchOS 信任憑證的 `wss://` 閘道端點；
  Apple Watch 會輪詢對應的 `https://` 來源。不支援純 HTTP，
  以及自我簽署或僅以指紋建立的信任。端點設定請參閱[閘道擁有的
  配對](/zh-TW/gateway/pairing)。Apple Watch 無法獨立連線至迴路位址、僅限 iPhone
  和僅限 tailnet 的路徑。
- 使用行動網路需要支援行動網路且已啟用服務的 Apple Watch。
- OpenClaw 在 Apple Watch 上處於使用中。Apple 不允許一般 watchOS App
  持續保持通用 WebSocket/TCP 連線，因此直接節點會使用短週期 HTTPS
  輪詢，並在 App 返回前景時重新連線。請參閱 Apple 的
  [watchOS 低階網路指引](https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchOS)。

設定：

1. 在 iPhone 上開啟 **Settings -> Apple Watch**。
2. 點選 **Enable Direct Gateway Connection**。
3. 在短效設定碼到期前，於 Apple Watch 上開啟 OpenClaw。
4. 使用 `openclaw nodes status` 驗證獨立的 Apple Watch 列。

設定碼包含短效、僅限節點使用的啟動認證資訊；在到期前應將其
視同密碼。它絕不包含 iPhone 儲存的閘道
密碼或權杖。配對後，Apple Watch 會儲存自己的裝置權杖，並
刪除啟動認證資訊。直接模式僅涵蓋下列命令。
聊天、對話、核准和現有的 `watch.*` 通知流程仍為
iPhone 中繼功能，且仍需要已配對的 iPhone。

直接 watchOS 節點命令：

| 介面          | 命令                           | 備註                                                       |
| ------------- | ------------------------------ | ---------------------------------------------------------- |
| 裝置          | `device.info`、`device.status` | Apple Watch 身分、電池、溫度、儲存空間和網路。             |
| 通知          | `system.notify`                | App 處於使用中時可用；需要 Apple Watch 權限。               |

watchOS 不向第三方 App 提供 WebKit，因此直接 Apple Watch 節點
不會公布 Canvas 命令。

## 官方建置版本的中繼支援推播

官方發布的 iOS 建置版本使用外部推播中繼，而不會將原始 APNs 權杖發布至閘道。公開發布管道的官方 App Store 建置版本使用託管於 `https://ios-push-relay.openclaw.ai` 的中繼；此基底 URL 會硬式編碼於 App Store 發布版本中，且不會讀取任何覆寫值。

自訂中繼部署需要刻意使用獨立的 iOS 建置／部署路徑，且其中繼 URL 必須與閘道中繼 URL 相符。App Store 發布管道絕不接受自訂中繼 URL。如果你使用自訂中繼建置版本，請設定相符的閘道中繼 URL：

```json5
{
  gateway: {
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
        },
      },
    },
  },
}
```

流程運作方式：

- iOS App 使用 App Attest 和 StoreKit App 交易 JWS 向中繼服務註冊。
- 中繼服務會傳回不透明的中繼控制代碼，以及註冊範圍限定的傳送授權。
- iOS App 會擷取已配對的閘道身分（`gateway.identity.get`），並在中繼服務註冊時納入該身分，讓以中繼服務為基礎的註冊委派給該特定閘道。
- App 會透過 `push.apns.register`，將該以中繼服務為基礎的註冊轉送至已配對的閘道。
- 閘道會將儲存的中繼控制代碼用於 `push.test`、背景喚醒與喚醒提示。
- 如果 App 之後連線至不同的閘道，或連線至使用不同中繼服務基底 URL 的組建，它會重新整理中繼服務註冊，而不會重複使用舊的繫結。

此路徑中閘道**不**需要：不需要整個部署共用的中繼服務權杖，也不需要用於官方 App Store 中繼傳送的直接 APNs 金鑰。

預期的操作流程：

1. 安裝官方 iOS App。
2. 選用：只有在刻意使用獨立的自訂中繼服務組建時，才需在閘道上設定 `gateway.push.apns.relay.baseUrl`。
3. 將 App 與閘道配對，並等待其完成連線。
4. App 會在取得 APNs 權杖、操作員工作階段已連線且中繼服務註冊成功後，發布 `push.apns.register`。
5. 之後，`push.test`、重新連線喚醒與喚醒提示便可使用儲存的中繼服務註冊。

## 背景存活信標

當 iOS 因靜默推播、背景重新整理或重大位置變更事件喚醒 App 時，App 會嘗試短暫重新連線至節點，接著使用 `event: "node.presence.alive"` 呼叫 `node.event`。只有在得知已驗證的節點裝置身分後，閘道才會將此資訊記錄為已配對節點／裝置中繼資料上的 `lastSeenAtMs`/`lastSeenReason`。

只有當閘道回應包含 `handled: true` 時，App 才會將背景喚醒視為已成功記錄。較舊的閘道可能會以 `{ "ok": true }` 確認 `node.event`；此回應具有相容性，但不會計為持久的最後上線時間更新。

相容性注意事項：

- `OPENCLAW_APNS_RELAY_BASE_URL` 仍可作為閘道的暫時環境變數覆寫使用（`gateway.push.apns.relay.baseUrl` 是設定優先的路徑）。
- App Store 發行組建的推播模式會硬編碼託管中繼服務主機，且絕不讀取中繼服務 URL 覆寫；組建階段的環境變數 `OPENCLAW_PUSH_RELAY_BASE_URL` 僅影響本機／沙箱 iOS 組建模式。

## 驗證與信任流程

中繼服務的存在，是為了強制執行兩項官方 iOS 組建使用閘道直接連線 APNs 時無法提供的限制：

- 只有透過 Apple 發布的正版 OpenClaw iOS 組建才能使用託管中繼服務。
- 閘道只能為與該特定閘道配對的 iOS 裝置傳送以中繼服務為基礎的推播。

逐段流程：

1. `iOS app -> gateway`：App 會透過一般閘道驗證流程與閘道配對，取得已驗證的節點工作階段與已驗證的操作員工作階段。操作員工作階段會呼叫 `gateway.identity.get`。
2. `iOS app -> relay`：App 會透過 HTTPS 呼叫中繼服務註冊端點，並提供 App Attest 證明與 StoreKit App 交易 JWS。中繼服務會驗證套件組合 ID、App Attest 證明與 Apple 發布證明，並要求使用官方／正式環境發布路徑。這會阻止本機 Xcode／開發組建使用託管中繼服務，因為本機組建無法滿足官方 Apple 發布證明。
3. `gateway identity delegation`：在向中繼服務註冊之前，App 會從 `gateway.identity.get` 擷取已配對的閘道身分，並將其納入中繼服務註冊承載資料。中繼服務會傳回中繼控制代碼，以及委派給該閘道身分的註冊範圍限定傳送授權。
4. `gateway -> relay`：閘道會儲存來自 `push.apns.register` 的中繼控制代碼與傳送授權。進行 `push.test`、重新連線喚醒及喚醒提示時，閘道會使用自己的裝置身分簽署傳送要求；中繼服務會根據註冊時委派的閘道身分，同時驗證儲存的傳送授權與閘道簽章。即使其他閘道以某種方式取得該控制代碼，也無法重複使用這項儲存的註冊。
5. `relay -> APNs`：中繼服務持有正式環境 APNs 認證資訊，以及官方組建的原始 APNs 權杖。對於以中繼服務為基礎的官方組建，閘道絕不儲存原始 APNs 權杖；中繼服務會代表已配對的閘道，將最終推播傳送至 APNs。

建立此設計的原因：避免將正式環境 APNs 認證資訊放入使用者的閘道、避免在閘道上儲存官方組建的原始 APNs 權杖、僅允許官方 OpenClaw iOS 組建使用託管中繼服務，並防止某個閘道向屬於其他閘道的 iOS 裝置傳送喚醒推播。

本機／手動組建仍使用直接 APNs。如果你在不使用中繼服務的情況下測試這些組建，閘道仍需直接 APNs 認證資訊：

```bash
export OPENCLAW_APNS_TEAM_ID="TEAMID"
export OPENCLAW_APNS_KEY_ID="KEYID"
export OPENCLAW_APNS_PRIVATE_KEY_P8="$(cat /path/to/AuthKey_KEYID.p8)"
```

這些是閘道主機執行階段環境變數，不是 Fastlane 設定。`apps/ios/fastlane/.env` 僅儲存 App Store Connect 驗證資訊，例如 `APP_STORE_CONNECT_KEY_ID` 和 `APP_STORE_CONNECT_ISSUER_ID`；它不會為本機 iOS 組建設定直接 APNs 傳送。

建議的閘道主機儲存方式，與 `~/.openclaw/credentials/` 下的其他供應商認證資訊一致：

```bash
mkdir -p ~/.openclaw/credentials/apns
chmod 700 ~/.openclaw/credentials/apns
mv /path/to/AuthKey_KEYID.p8 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
chmod 600 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
export OPENCLAW_APNS_PRIVATE_KEY_PATH="$HOME/.openclaw/credentials/apns/AuthKey_KEYID.p8"
```

請勿提交 `.p8` 檔案，也不要將其放在存放庫簽出目錄下。

## 探索路徑

### Bonjour（區域網路）

iOS App 會在 `local.` 上瀏覽 `_openclaw-gw._tcp`，並在設定後瀏覽相同的廣域 DNS-SD 探索網域。同一區域網路中的閘道會自動透過 `local.` 顯示；跨網路探索則可使用設定的廣域網域，無須變更信標類型。

### Tailnet（跨網路）

如果 mDNS 遭到封鎖，請使用單點傳播 DNS-SD 區域（選擇一個網域，例如：`openclaw.internal.`）和 Tailscale 分割 DNS。CoreDNS 範例請參閱 [Bonjour](/zh-TW/gateway/bonjour)。

### 手動主機／連接埠

在 Settings 中啟用 **Manual Host**，並輸入閘道主機與連接埠（預設為 `18789`）。

## 多個閘道

App 會保留所有已配對閘道的登錄清單，因此你可以在它們之間切換，而無須再次配對：

- **Settings -> Gateway** 會顯示 **Paired Gateways** 清單，並標示作用中的閘道。點一下項目即可切換；App 會中斷目前的工作階段，並重新連線至選取的閘道。配對超過一個閘道時，連線列旁會顯示快速切換選單。
- 認證資訊、TLS 信任決策、各閘道偏好設定及快取的聊天記錄都會按閘道分別儲存。切換閘道絕不會混用不同閘道的狀態，推播註冊也會跟隨作用中的閘道。
- 滑動已配對的閘道（或使用其內容選單）以 **Forget** 該閘道，這會移除其認證資訊、裝置權杖、TLS 固定憑證及快取的聊天內容。
- 若要切換至探索到的閘道，該閘道必須可在網路上看見；手動閘道則會依儲存的主機與連接埠重新連線。

## 畫布 + A2UI

iOS 節點會呈現 WKWebView 畫布。使用 `node.invoke` 操作：

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.navigate --params '{"url":"http://<gateway-host>:18789/__openclaw__/canvas/"}'
```

注意事項：

- 閘道畫布主機會透過閘道 HTTP 伺服器提供 `/__openclaw__/canvas/` 和 `/__openclaw__/a2ui/`（與 `gateway.port` 使用相同連接埠，預設為 `18789`）。
- iOS 節點會保留內建框架作為已連線時的預設檢視。`canvas.a2ui.push` 和 `canvas.a2ui.reset` 會使用隨附且由 App 擁有的 A2UI 頁面。
- 遠端閘道 A2UI 頁面在 iOS 上僅供呈現；只有隨附且由 App 擁有的頁面所發出的原生 A2UI 按鈕動作才會被接受。
- 使用 `canvas.navigate` 和 `{"url":""}` 返回內建框架。

## 與 Computer Use 的關係

iOS App 是行動節點介面，而不是 Codex Computer Use 後端。Codex Computer Use 和 `cua-driver mcp` 會透過 MCP 工具控制本機 macOS 桌面；iOS App 則透過 OpenClaw 節點命令公開 iPhone 功能，例如 `canvas.*`、`camera.*`、`screen.*`、`location.*` 和 `talk.*`。

代理仍可透過 OpenClaw 呼叫節點命令來操作 iOS App，但這些呼叫會通過閘道節點通訊協定，並受 iOS 前景／背景限制約束。本機桌面控制請使用 [Codex Computer Use](/zh-TW/plugins/codex-computer-use)；iOS 節點功能則請參閱本頁。

### 畫布求值／快照

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.eval --params '{"javaScript":"(() => { const {ctx} = window.__openclaw; ctx.clearRect(0,0,innerWidth,innerHeight); ctx.lineWidth=6; ctx.strokeStyle=\"#ff2d55\"; ctx.beginPath(); ctx.moveTo(40,40); ctx.lineTo(innerWidth-40, innerHeight-40); ctx.stroke(); return \"ok\"; })()"}'
```

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.snapshot --params '{"maxWidth":900,"format":"jpeg"}'
```

## 語音喚醒 + 對話模式

- 語音喚醒與對話模式可在 Settings 中使用。
- 當 `talk.realtime.transport` 為 `webrtc` 時，OpenAI 即時對話會使用由用戶端管理的 WebRTC；明確的 `gateway-relay` 設定仍由閘道管理。請參閱[對話模式](/zh-TW/nodes/talk)。
- 支援對話的 iOS 節點會公告 `talk` 功能，並可宣告 `talk.ptt.start`、`talk.ptt.stop`、`talk.ptt.cancel` 和 `talk.ptt.once`；閘道預設允許受信任且支援對話的節點使用這些按住說話命令。
- iOS 可能會暫停背景音訊；App 未處於作用中狀態時，語音功能僅能盡力提供。

## 常見錯誤

- `NODE_BACKGROUND_UNAVAILABLE`：將 iOS App 帶至前景（畫布／相機／螢幕命令需要 App 位於前景）。
- `A2UI_HOST_UNAVAILABLE`：App WebView 無法存取隨附的 A2UI 頁面；讓 App 保持在前景並停留於 Screen 分頁，然後重試。
- 配對提示從未出現：執行 `openclaw devices list` 並手動核准。
- Watch 未顯示 iPhone 狀態：確認 iPhone 在 `watch.status` 中回報 `watchPaired: true`
  和 `watchAppInstalled: true`。如果配對狀態為 false，請在 Apple 的 Watch App 中配對
  Watch。如果安裝狀態為 false，請從 **My Watch -> Available Apps** 安裝隨附 App。
  完成任一變更後，在 Watch 上開啟 OpenClaw 一次；即時可連線性仍要求兩個 App
  都在執行，而佇列中的更新稍後仍可在背景抵達。
- 重新安裝後無法重新連線：鑰匙圈中的配對權杖已遭清除；請重新配對節點。

## 相關文件

- [配對](/zh-TW/channels/pairing)
- [探索](/zh-TW/gateway/discovery)
- [Bonjour](/zh-TW/gateway/bonjour)
