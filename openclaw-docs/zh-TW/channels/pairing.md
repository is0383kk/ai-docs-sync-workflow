---
read_when:
    - 設定私訊存取控制
    - 配對新的 iOS/Android 節點
    - 審查 OpenClaw 的安全態勢
summary: 配對概覽：核准哪些人可以私訊你，以及哪些節點可以加入
title: 配對
x-i18n:
    generated_at: "2026-07-26T07:43:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dc874d660509f59bc26795c8b3ce13f5d238cd61154c717637f5d545b995fb08
    source_path: channels/pairing.md
    workflow: 16
---

「配對」是 OpenClaw 明確的存取核准步驟。
它用於兩個地方：

1. **私訊配對**（允許誰與機器人交談）
2. **節點配對**（允許哪些裝置／節點加入閘道網路）

安全性背景資訊：[安全性](/zh-TW/gateway/security)

## 1) 私訊配對（傳入聊天存取權）

當頻道設定了私訊政策 `pairing` 時，未知傳送者會收到一組短代碼，而且在你核准之前，其訊息**不會被處理**。

預設私訊政策記載於：[安全性](/zh-TW/gateway/security)

只有在有效的私訊允許清單包含 `"*"` 時，`dmPolicy: "open"` 才是公開的。
設定與驗證要求公開開放設定必須包含該萬用字元。如果現有
狀態包含具有具體 `allowFrom` 項目的 `open`，執行階段仍只允許
這些傳送者，而配對儲存區中的核准不會擴大 `open` 存取權。

配對代碼：

- 8 個字元、大寫、不含容易混淆的字元（`0O1I`）。
- **1 小時後到期**。機器人只會在建立新要求時傳送配對訊息（每位傳送者大約每小時一次）。
- 待處理的私訊配對要求上限為**每個頻道帳號 3 個**；在其中一個到期或獲得核准之前，系統會忽略其他要求。

### 從控制介面核准

開啟 **Settings → Channels → DM access requests**。佇列會彙整所有已設定且私訊政策為 `pairing` 的
頻道帳號之待處理要求。
依頻道或帳號篩選、檢查傳送者 ID 與中繼資料，然後選擇
**Approve**。

核准只會授予私訊存取權，不會授予群組存取權。若支援，
核准對話方塊也會提供下列明確選項：

- **核准後通知要求者**
- **同時將此傳送者設為第一位命令擁有者**，僅在尚無命令
  擁有者且控制介面工作階段具有 `operator.admin` 時顯示

選擇 **Dismiss** 可移除待處理要求而不予核准。忽略
並非永久封鎖；傳送者稍後仍可再次要求存取權。

### 從命令列介面核准

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

加入 `--notify`，即可透過相同頻道通知要求者。多帳號頻道
接受 `--account <id>`。

與控制介面的明確核取方塊不同，當尚未設定命令擁有者時，命令列介面會自動啟用
`commands.ownerAllowFrom`，使用如 `telegram:123456789` 的項目。
這會為首次設定提供一位明確擁有者，以處理具特殊權限的命令和執行核准提示。擁有者存在後，後續
配對核准只會授予私訊存取權，不會新增更多擁有者。

<Note>
WhatsApp 的登入 QR Code 會將 WhatsApp 帳號連結至 OpenClaw。私訊存取要求
會核准向該帳號傳送訊息的人員。這是兩個不同的流程。
</Note>

支援的頻道（任何宣告配對功能的已安裝頻道外掛；如 `openclaw-weixin` 等外部外掛可增加更多頻道）：`discord`、`feishu`、`googlechat`、`imessage`、`irc`、`line`、`matrix`、`mattermost`、`msteams`、`nextcloud-talk`、`nostr`、`signal`、`slack`、`sms`、`synology-chat`、`telegram`、`twitch`、`whatsapp`、`zalo`、`zalouser`。

### 可重複使用的傳送者群組

當相同的受信任傳送者集合應套用至
多個訊息頻道，或同時套用至私訊與群組允許清單時，請使用頂層 `accessGroups`。

靜態群組使用 `type: "message.senders"`，並在頻道允許清單中以
`accessGroup:<name>` 參照：

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
        whatsapp: ["+15551234567"],
      },
    },
  },
  channels: {
    telegram: { dmPolicy: "allowlist", allowFrom: ["accessGroup:operators"] },
    whatsapp: { groupPolicy: "allowlist", groupAllowFrom: ["accessGroup:operators"] },
  },
}
```

存取群組的詳細說明請參閱：[存取群組](/zh-TW/channels/access-groups)

### 狀態儲存位置

儲存在 `~/.openclaw/state/openclaw.sqlite` 的共用 SQLite 狀態資料庫中：

- `channel_pairing_requests` 中的待處理要求
- `channel_pairing_allow_entries` 中已核准的傳送者

帳號範圍行為：

- 每個要求與已核准傳送者都以頻道和帳號為索引鍵
- 執行階段只讀取標準 SQLite 資料列；不會合併舊版檔案

舊版閘道會在 `~/.openclaw/credentials/` 下寫入 `<channel>-pairing.json` 和
`<channel>-<accountId>-allowFrom.json`。
啟動移轉程序與 `openclaw doctor --fix` 會將這些檔案匯入 SQLite，並在成功匯入後
移除各來源檔案。請將 SQLite 資料庫視為敏感資料，因為這些資料列會管制你助理的存取權。

<Note>
配對允許清單儲存區用於私訊存取權。群組授權則另行處理。
核准私訊配對代碼不會自動允許該傳送者在群組中執行
命令或控制機器人。第一位擁有者的初始建立是 `commands.ownerAllowFrom` 中獨立的設定
狀態，而群組聊天傳遞仍遵循
頻道的群組允許清單（例如 `groupAllowFrom`、`groups`，或依頻道而定的個別群組
或個別主題覆寫設定）。
</Note>

## 2) 節點裝置配對（iOS／Android／macOS／無頭節點）

節點會以具有 `role: node` 的**裝置**身分連線至閘道。閘道
會建立必須核准的裝置配對要求。

### 從控制介面配對（建議）

使用已連線且具有 `operator.admin` 存取權的控制介面工作階段：

1. 開啟控制介面並前往 **Settings → Devices**。
2. 在 **Devices** 頁面上，按一下 **Pair mobile device**。
3. 保留 **Full access (recommended)**，或選取 **Limited access** 以排除
   閘道管理控制功能。
4. 按一下 **Create setup code**。
5. 在手機上開啟 OpenClaw 應用程式 → **Settings** → **Gateway**。
6. 掃描 QR Code 或貼上設定代碼，然後連線。

當官方 OpenClaw iOS 與 Android 應用程式的
設定代碼中繼資料相符時，系統會自動核准。如果 **Pending approval** 顯示要求（例如
非官方用戶端或中繼資料不相符），請在核准前檢查其角色和
範圍。

當目前的控制介面工作階段沒有
管理員存取權時，此按鈕會停用。此時請在閘道主機上使用下方的命令列介面核准流程。

### 透過 Telegram 配對

如果你使用 `device-pair` 外掛，可以完全透過 Telegram 完成首次裝置配對：

1. 在 Telegram 中傳訊息給你的機器人：`/pair`
2. 機器人會回覆兩則訊息：一則操作說明訊息，以及另一則獨立的**設定代碼**訊息（方便在 Telegram 中複製／貼上）。
3. 在手機上開啟 OpenClaw iOS 應用程式 → Settings → Gateway。
4. 掃描 QR Code（`/pair qr`），或貼上設定代碼並連線。
5. 官方行動應用程式會自動連線。如果 `/pair pending` 顯示
   要求，請在核准前檢查其角色和範圍。

設定代碼是以 base64 編碼的 JSON 承載資料，其中包含：

- `url`：閘道 WebSocket URL（`ws://...` 或 `wss://...`）
- `urls`：若可用，行動應用程式可依序嘗試的 LAN／Tailnet 路由
- `bootstrapToken`：用於初始配對交握的一次性啟動權杖；閘道會在 10 分鐘後使其到期

配對完成後，執行 `/pair cleanup` 以使未使用的設定代碼失效。

該啟動權杖帶有內建的配對啟動設定檔：

- 安全的 `wss://` 設定（或同一主機的迴路）預設為 `node`，並具有完整的
  原生行動裝置 `operator` 存取權
- 轉交的 `node` 權杖會維持 `scopes: []`
- 預設轉交的 `operator` 權杖包含 `operator.admin`、
  `operator.approvals`、`operator.read`、`operator.talk.secrets` 和
  `operator.write`
- 控制介面的 **Limited access** 和 `openclaw qr --limited` 會省略
  `operator.admin`，同時保留其他操作員範圍
- 純文字 LAN `ws://` 設定會自動使用相同的受限設定檔；
  設定 `wss://` 或 Tailscale Serve，並產生新代碼以取得完整存取權
- 後續的權杖輪替／撤銷仍同時受到裝置已核准的
  角色合約與呼叫端工作階段操作員範圍的限制

設定代碼有效期間，請像保護密碼一樣保護它。

iOS 與 Android 的 **Settings → Gateway** 頁面會顯示 **Full** 或 **Limited**
存取權。若要升級受限的手機，請先設定安全的 `wss://` 或
Tailscale Serve 路由，接著產生新的完整存取權設定代碼、在該設定頁面中掃描或貼上
該代碼，然後重新連線。

對於 Tailscale、公開或其他遠端行動裝置配對，請使用 Tailscale Serve／Funnel
或其他 `wss://` 閘道 URL。純文字 `ws://` 設定代碼只接受
迴路、私人 LAN 位址、`.local` Bonjour 主機與 Android
模擬器主機。非迴路純文字路由會獲得受限存取權。Tailnet
CGNAT 位址、`.ts.net` 名稱與公開主機仍會在
發行 QR Code／設定代碼前採取失敗即關閉策略。

對於 `gateway.bind=lan` 設定 URL，OpenClaw 會偵測將作用中閘道迴路連接埠
設為 Proxy 的持續性 Tailscale Serve HTTPS 根目錄，並與 LAN 路由一同公布。
設定命令只會為 `lan` 加入此備援路徑；`custom` 和 `tailnet` 會保留其明確公布的路由。
iOS 應用程式會依序探測公布的路由，並儲存第一個可連線的
端點。

### 核准節點裝置

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

當明確核准因執行核准的已配對裝置工作階段僅以配對範圍開啟而遭拒時，
命令列介面會使用 `operator.admin` 重試相同要求。這讓現有具管理員能力的已配對裝置
可以復原新的控制介面／瀏覽器配對，而不必手動編輯配對儲存區。
閘道仍會驗證重試的連線；無法使用 `operator.admin` 進行驗證的
權杖仍會遭到封鎖。

如果同一裝置使用不同的驗證詳細資料（例如不同的
角色／範圍／公開金鑰）重試，先前的待處理要求會被取代，並建立新的
`requestId`。

<Note>
已配對的裝置不會在未告知的情況下取得更廣泛的存取權。如果裝置重新連線並要求更多範圍或更廣泛的角色，OpenClaw 會維持現有核准不變，並建立新的待處理升級要求。核准前，請使用 `openclaw devices list` 比較目前已核准的存取權與新要求的存取權。
</Note>

### 選用的受信任 CIDR 節點自動核准

裝置配對預設仍須手動進行。對於受到嚴格控管的節點網路，
你可以使用明確的 CIDR 或確切 IP，選擇加入首次節點自動核准功能：

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

這只適用於未要求
範圍的新 `role: node` 配對要求。操作員、瀏覽器、控制介面與 WebChat 用戶端仍需手動
核准。角色、範圍、中繼資料與公開金鑰的變更仍需手動
核准。

### 節點配對狀態儲存區

儲存在 `~/.openclaw/state/openclaw.sqlite` 的共用 SQLite 狀態資料庫中：

- 待處理的裝置配對要求（短期；5 分鐘後到期）
- 已配對的裝置與權杖

舊版閘道將此狀態保存在 `~/.openclaw/devices/*.json`；這些檔案會在閘道啟動時
匯入 SQLite，並以 `.migrated` 後綴封存。

### 注意事項

- `node.pair.*` API（命令列介面：`openclaw nodes pending|approve|reject|remove|rename`）管理
  儲存在相同配對裝置記錄中的節點功能核准。WS 節點
  仍需要裝置配對；請參閱[節點配對](/zh-TW/gateway/pairing)。
- 配對記錄是已核准角色的持久性事實來源。有效的
  裝置權杖仍受限於該組已核准角色；核准角色之外的
  零散權杖項目不會建立新的存取權限。

## 相關文件

- 安全性模型與提示詞注入：[安全性](/zh-TW/gateway/security)
- 安全更新（執行 doctor）：[更新](/zh-TW/install/updating)
- 頻道設定：
  - Telegram：[Telegram](/zh-TW/channels/telegram)
  - WhatsApp：[WhatsApp](/zh-TW/channels/whatsapp)
  - Signal：[Signal](/zh-TW/channels/signal)
  - iMessage：[iMessage](/zh-TW/channels/imessage)
  - Discord：[Discord](/zh-TW/channels/discord)
  - Slack：[Slack](/zh-TW/channels/slack)
