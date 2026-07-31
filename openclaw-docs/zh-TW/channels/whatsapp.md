---
read_when:
    - 處理 WhatsApp／網頁頻道行為或收件匣路由設定
summary: WhatsApp 頻道支援、存取控制、傳遞行為與操作管理
title: WhatsApp
x-i18n:
    generated_at: "2026-07-26T07:44:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7489b37f91775868d0694daea8a0958ee000d1411674d1800bb1e77df5961e68
    source_path: channels/whatsapp.md
    workflow: 16
---

狀態：已可用於正式環境，透過 WhatsApp Web（Baileys）運作。閘道負責管理已連結的工作階段；沒有獨立的 Twilio WhatsApp 頻道。

## 安裝

`openclaw onboard` 和 `openclaw channels add --channel whatsapp` 會在你首次選取外掛時提示安裝；如果缺少外掛，`openclaw channels login --channel whatsapp` 也會提供相同的安裝流程。開發版簽出會使用本機外掛路徑；穩定版／測試版安裝會先從 ClawHub 安裝 `@openclaw/whatsapp`，失敗時再改用 npm。WhatsApp 執行階段不隨 OpenClaw 核心 npm 套件提供，因此其執行階段相依套件會保留在外部外掛中。手動安裝：

```bash
openclaw plugins install clawhub:@openclaw/whatsapp
```

僅將純 npm 套件（`@openclaw/whatsapp`）用於登錄檔備援；只有需要可重現安裝時，才固定使用確切版本。

<CardGroup cols={3}>
  <Card title="配對" icon="link" href="/zh-TW/channels/pairing">
    未知傳送者的預設私訊原則為配對。
  </Card>
  <Card title="頻道疑難排解" icon="wrench" href="/zh-TW/channels/troubleshooting">
    跨頻道診斷與修復操作手冊。
  </Card>
  <Card title="閘道設定" icon="settings" href="/zh-TW/gateway/configuration">
    完整的頻道設定模式與範例。
  </Card>
</CardGroup>

## 快速設定

<Steps>
  <Step title="設定存取原則">

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

  </Step>

  <Step title="連結 WhatsApp（QR 碼）">

```bash
openclaw channels login --channel whatsapp
```

    登入僅支援 QR 碼。在遠端或無頭主機上，開始登入前，請確保有可靠的方法能將即時 QR 碼傳送到手機；終端機顯示的 QR 碼、螢幕截圖或聊天附件都可能在傳輸途中過期。

    若要指定帳號：

```bash
openclaw channels login --channel whatsapp --account work
```

    若要在登入前附加現有／自訂的驗證目錄：

```bash
openclaw channels add --channel whatsapp --account work --auth-dir /path/to/wa-auth
openclaw channels login --channel whatsapp --account work
```

  </Step>

  <Step title="啟動閘道">

```bash
openclaw gateway
```

  </Step>

  <Step title="核准第一個私訊存取要求（配對模式）">

    開啟 **設定 → 頻道 → 私訊存取要求**，找到 WhatsApp 帳號，
    然後核准傳送者。如果偏好使用命令列介面：

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

    私訊存取要求會在 1 小時後過期；每個
    帳號最多可有 3 個待處理要求。此核准與用於連結
    帳號本身的 WhatsApp 登入 QR 碼分開處理。

  </Step>
</Steps>

<Note>
建議使用獨立的 WhatsApp 號碼（設定與中繼資料已針對此方式最佳化），但也完整支援個人號碼／自我聊天設定。
</Note>

## 部署模式

<AccordionGroup>
  <Accordion title="專用號碼（建議）">
    - 為 OpenClaw 使用獨立的 WhatsApp 身分
    - 更清楚的私訊允許清單與路由界線
    - 降低自我聊天混淆的機率

    ```json5
    {
      channels: {
        whatsapp: {
          dmPolicy: "allowlist",
          allowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="個人號碼備援">
    新手引導支援個人號碼模式，並寫入適合自我聊天的基準設定：`dmPolicy: "allowlist"`、`allowFrom`（包含你自己的號碼）、`selfChatMode: true`。執行階段的自我聊天防護會依據已連結的自身號碼加上 `allowFrom` 判定。
  </Accordion>
</AccordionGroup>

## 執行階段模型

- 閘道負責管理 WhatsApp 通訊端與重新連線迴圈。
- 看門狗會分別追蹤兩個訊號：原始 WhatsApp Web 傳輸活動與應用程式訊息活動。安靜但仍保持連線的工作階段，不會只因最近沒有收到訊息就重新啟動；只有在固定的內部時間範圍內（使用者無法設定）未收到傳輸框架，或應用程式訊息靜默時間超過一般訊息逾時的 4 倍時，才會強制重新連線。最近仍有活動的工作階段剛完成重新連線時，第一個時間範圍會使用較短的一般訊息逾時，而不是 4 倍時間範圍。在該次重新連線初期，OpenClaw 可以自動回覆 Baileys 提前傳遞的離線訊息，範圍受限於傳入訊息 ID 的去重生命週期；初始啟動仍會保留較短的過時記錄防護。
- 傳出訊息時，目標帳號必須有作用中的 WhatsApp 接聽器；否則傳送會立即失敗。
- 傳送群組訊息時，如果 `@+<digits>` 和 `@<digits>` 權杖（位於文字與媒體說明文字中）符合目前的參與者中繼資料，則會附加原生提及中繼資料，包括以 LID 為基礎的群組。
- 狀態與廣播聊天（`@status`、`@broadcast`）會被忽略。
- 直接聊天使用私訊工作階段規則（`session.dmScope`；預設的 `main` 會將私訊合併至代理程式的主要工作階段）。群組工作階段會依每個 JID 隔離（`agent:<agentId>:whatsapp:group:<jid>`）。
- WhatsApp 頻道／電子報可透過其原生 `@newsletter` JID 指定為明確的傳出目標，並使用頻道工作階段中繼資料（`agent:<agentId>:whatsapp:channel:<jid>`），而非私訊語意。
- WhatsApp Web 傳輸會採用閘道主機上的標準 Proxy 環境變數（`HTTPS_PROXY`、`HTTP_PROXY`、`NO_PROXY` 及其小寫變體）。請優先使用主機層級的 Proxy 設定，而非各頻道設定。

## 使用 MeowCaller 致電目前的要求者（實驗性）

外掛可在源自 WhatsApp 的代理程式回合中公開 `whatsapp_call`。它使用 [MeowCaller](https://github.com/purpshell/meowcaller) 向目前已授權的要求者撥打 WhatsApp 語音通話，並在對方接聽後播放 OpenClaw TTS 訊息。此工具沒有目的地號碼參數，因此提示詞無法將通話重新導向。預設停用。

<Warning>
MeowCaller 為實驗性功能，沒有已加標籤的版本，且使用另外配對的 whatsmeow 已連結裝置工作階段，無法重複使用外掛的 Baileys 認證資訊。配對會在同一個 WhatsApp 帳號中新增另一部已連結裝置；請使用 OpenClaw 所使用的身分進行掃描。個人號碼／自我聊天模式無法撥打給自己；請使用專用 OpenClaw 號碼撥打你的個人號碼。
</Warning>

<Steps>
  <Step title="啟用實驗性通話">

    將 `actions.calls: true` 新增至 WhatsApp 頻道設定，然後重新啟動閘道：

```json
{
  "channels": {
    "whatsapp": {
      "actions": {
        "calls": true
      }
    }
  }
}
```

    若未設定或為 `false`，OpenClaw 不會公開 `whatsapp_call` 工具。

  </Step>

  <Step title="安裝已審查的 MeowCaller 命令列介面">

    轉接器預期閘道主機的 `PATH` 中有一個 `meowcaller` 可執行檔。在 [MeowCaller PR #7](https://github.com/purpshell/meowcaller/pull/7) 合併前，請建置已審查的分支：

```bash
git clone --branch feat/send-only-notify https://github.com/steipete/meowcaller.git
cd meowcaller
git checkout 752050471fc2bf7a8cdfbf7dbd3cd4e865d85d3f
mkdir -p "$HOME/.local/bin"
go build -o "$HOME/.local/bin/meowcaller" ./cmd/meowcaller
```

    確保閘道服務的 `PATH` 中包含 `$HOME/.local/bin`。此修訂版具有明確的 `pair` 與僅傳送的 `notify` 命令；`notify` 不會開啟麥克風、喇叭、視訊裝置或診斷擷取。請勿改用上游範例命令列介面的 `play` 命令。

  </Step>

  <Step title="配對 MeowCaller 已連結裝置">

    要求 WhatsApp 代理程式檢查通話設定（`whatsapp_call` 狀態動作會回報帳號特定的狀態目錄與配對命令）。針對預設帳號：

```bash
state_dir="$HOME/.openclaw/credentials/whatsapp-calls/default"
mkdir -p "$state_dir"
chmod 700 "$state_dir"
meowcaller pair --store "$state_dir/wa-voip.db"
```

    以互動方式執行此命令，從 **WhatsApp > Linked devices** 掃描 QR 碼，並等待 `MeowCaller linked device ready`。請將 `wa-voip.db` 保密，這是 MeowCaller 工作階段。非預設帳號會從狀態動作取得各自的儲存區路徑；在 Windows 上，請執行其 PowerShell 命令。

  </Step>

  <Step title="設定 TTS 並從 WhatsApp 撥打電話">

    設定支援電話語音的 [TTS 供應商](/zh-TW/tools/tts)，重新啟動閘道，然後傳送類似 `Call me and say the build finished.` 的要求。此工具會從受信任的傳入內容解析傳送者、合成暫時的私人 WAV 檔案、在有限的通話時間範圍內執行 MeowCaller，並在之後刪除音訊檔案。OpenClaw 會明確傳遞帳號的儲存區、等待接聽／播放／掛斷後的零退出狀態，並將逾時或非零退出視為工具呼叫失敗。

  </Step>
</Steps>

限制：僅支援一對一傳出語音通話、不支援任意目的地號碼、不與聊天連線共用驗證、個人號碼／自我聊天模式無法撥打給自己、合成音訊上限為 60 秒、除了 MeowCaller 完成接聽／播放／掛斷外，沒有手機端可聽性回執；OpenClaw 會在有限的 115–175 秒時間範圍後停止隨附程序（涵蓋 MeowCaller 的連線、接聽、播放及關閉階段）。

## 核准提示

WhatsApp 可將執行與外掛核准提示呈現為 `👍`/`👎` 回應，並由頂層核准轉送設定控制：

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session",
    },
    plugin: {
      enabled: true,
      mode: "targets",
      targets: [{ channel: "whatsapp", to: "+15551234567" }],
    },
  },
}
```

`approvals.exec` 與 `approvals.plugin` 彼此獨立；啟用 WhatsApp 頻道只會連結傳輸層，除非啟用相符的核准類別並將其路由至該處，否則不會傳送任何內容。工作階段模式只會對源自 WhatsApp 的核准傳送原生表情符號核准。目標模式會對明確目標使用共用轉送流水線，不會建立個別的核准者私訊扇出。

WhatsApp 核准回應需要在 `allowFrom`（或 `"*"`）中明確指定核准者。`defaultTo` 設定一般預設訊息目標，而不是核准者清單。手動 `/approve` 命令在解析核准前，仍會經過一般的 WhatsApp 傳送者授權路徑。

## 問題回應

若 `ask_user` 提示包含一個非機密的單選問題，且有一至四個選項，WhatsApp 會在選項標籤旁顯示 `1️⃣` 到 `4️⃣`。使用相符數字回應已送達的提示即可作答。OpenClaw 會透過閘道將數字對應至標準選項；過期或重複的點選會被忽略。多問題、多選及自由文字提示仍只能以文字回覆。一般 WhatsApp 私訊／群組准入規則會授權做出回應的傳送者。

## 外掛掛鉤與隱私權

傳入的 WhatsApp 訊息可能含有個人內容、電話號碼、群組識別碼、傳送者名稱及工作階段關聯欄位。除非你選擇加入，否則 WhatsApp 不會將傳入的 `message_received` 掛鉤承載資料廣播給外掛：

```json5
{
  channels: {
    whatsapp: {
      pluginHooks: {
        messageReceived: true,
      },
    },
  },
}
```

在 `channels.whatsapp.accounts.<id>.pluginHooks.messageReceived` 下將選擇加入的範圍限制為單一帳號。只有當你信任外掛能存取傳入的 WhatsApp 內容與識別碼時，才啟用此功能。

## 存取控制與啟用

<Tabs>
  <Tab title="私訊原則">
    `channels.whatsapp.dmPolicy`：

    | 值 | 行為 |
    | --- | --- |
    | `pairing`（預設） | 未知傳送者要求配對；由擁有者核准 |
    | `allowlist` | 僅允許 `allowFrom` 傳送者 |
    | `open` | 要求 `allowFrom` 包含 `"*"` |
    | `disabled` | 封鎖所有私訊 |

    `allowFrom` 接受 E.164 格式的號碼（內部會正規化）。它僅是私訊傳送者存取控制清單，不會限制明確傳送至群組 JID 或 `@newsletter` 頻道 JID 的外寄訊息。

    多帳號覆寫：`channels.whatsapp.accounts.<id>.dmPolicy`（以及 `.allowFrom`）的優先順序高於該帳號的頻道層級預設值。

    執行階段注意事項：

    - 配對會持續儲存在頻道允許清單儲存區，並與已設定的 `allowFrom` 合併
    - 排程自動化與心跳偵測收件者的備援會使用明確的傳遞目標或已設定的 `allowFrom`；私訊配對核准不會隱含成為排程／心跳偵測收件者
    - 若未設定允許清單，預設允許已連結的自身號碼
    - OpenClaw 絕不會自動配對外寄的 `fromMe` 私訊（你從已連結裝置傳送給自己的訊息）

  </Tab>

  <Tab title="群組政策與允許清單">
    群組存取分為兩層：

    1. **群組成員資格允許清單**（`channels.whatsapp.groups`）：若省略 `groups`，所有群組皆符合資格；若存在，則作為群組允許清單（`"*"` 允許全部）。
    2. **群組傳送者政策**（`channels.whatsapp.groupPolicy` + `groupAllowFrom`）：`open` 會略過傳送者允許清單，`allowlist` 要求符合 `groupAllowFrom`（或 `*`），`disabled` 會封鎖所有群組傳入訊息。

    若未設定 `groupAllowFrom`，且 `allowFrom` 含有項目，傳送者檢查會退回使用該設定。系統會先評估傳送者允許清單，再進行提及／回覆啟用判定。

    若完全不存在 `channels.whatsapp` 區塊，執行階段會退回使用 `groupPolicy: "allowlist"`（並記錄警告），即使 `channels.defaults.groupPolicy` 設為其他值亦然。

    <Note>
    群組成員資格解析具備單一帳號安全機制：若只設定一個 WhatsApp 帳號，且其 `accounts.<id>.groups` 是明確的空物件（`{}`），系統會將其視為「未設定」，並退回使用根層級的 `channels.whatsapp.groups` 對應表，而不會無聲地封鎖所有群組。若設定了 2 個以上的帳號，明確的空帳號對應表會維持空白且不會退回使用其他設定——如此一來，某個帳號便可刻意停用所有群組，而不影響其他同層帳號。
    </Note>

  </Tab>

  <Tab title="提及與 /activation">
    群組回覆預設需要提及。提及偵測包括：

    - 明確提及機器人身分的 WhatsApp 提及
    - 已設定的提及規則運算式模式（`agents.entries.*.groupChat.mentionPatterns`，退回使用 `messages.groupChat.mentionPatterns`）
    - 已授權群組訊息的傳入語音留言轉錄文字
    - 隱含的回覆機器人偵測（回覆對象的傳送者與機器人身分相符）

    安全性：引用／回覆只會滿足提及閘控條件，**不會**授予傳送者授權。使用 `groupPolicy: "allowlist"` 時，不在允許清單中的傳送者仍會遭到封鎖，即使是回覆允許清單中使用者的訊息也一樣。

    工作階段層級的啟用命令：`/activation mention` 或 `/activation always`。這會更新工作階段狀態（而非全域設定），且僅限擁有者操作。

  </Tab>
</Tabs>

## 已設定的 ACP 繫結

WhatsApp 透過頂層 `bindings[]` 支援持久化 ACP 繫結：

```json5
{
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "whatsapp",
        accountId: "work",
        peer: { kind: "direct", id: "+15555550123" },
      },
    },
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "whatsapp",
        accountId: "work",
        peer: { kind: "group", id: "120363424282127706@g.us" },
      },
    },
  ],
}
```

直接聊天會比對 E.164 號碼；群組會比對 WhatsApp 群組 JID。在 OpenClaw 確保繫結的 ACP 工作階段存在之前，會先套用群組允許清單、傳送者政策與提及／啟用閘控。符合的繫結會擁有該路由——廣播群組不會將該回合分派至一般 WhatsApp 工作階段。

## 個人號碼與自我聊天行為

當已連結的自身號碼也存在於 `allowFrom` 中時，系統會啟用自我聊天防護措施：略過自我聊天回合的已讀回條、忽略會提示你自己的提及 JID 自動觸發行為，並在頻道／帳號的 `responsePrefix` 未設定時，預設將回覆傳送至 `[{identity.name}]`（或 `[openclaw]`）。

## 訊息正規化與上下文

<AccordionGroup>
  <Accordion title="傳入封裝與回覆上下文">
    傳入訊息會封裝於共用的傳入封裝中。引用回覆會以下列形式附加上下文：

    ```text
    [Replying to <sender> id:<stanzaId>]
    <quoted body or media placeholder>
    [/Replying]
    ```

    系統會在可取得時填入回覆中繼資料（`ReplyToId`、`ReplyToBody`、`ReplyToSender`、傳送者 JID／E.164）。若引用的目標是可下載的媒體，OpenClaw 會透過一般傳入媒體儲存區儲存該媒體，並公開 `MediaPath`/`MediaType`，讓代理程式可直接檢查，而非只看到 `<media:image>`。

  </Accordion>

  <Accordion title="媒體預留位置與位置／聯絡人擷取">
    僅含媒體的訊息會正規化為預留位置：`<media:image>`、`<media:video>`、`<media:audio>`、`<media:document>`、`<media:sticker>`。

    當訊息本文僅為 `<media:audio>` 時，已授權群組的語音留言會在提及閘控前進行轉錄，因此在語音留言中說出機器人提及內容即可觸發回覆。若轉錄文字仍未提及機器人，該文字會保留於待處理群組歷程記錄中，而不是保留原始預留位置。

    位置訊息本文會呈現為簡短的座標文字。位置標籤／註解以及聯絡人／vCard 詳細資料會呈現為圍欄式不受信任中繼資料，而非行內提示文字。

  </Accordion>

  <Accordion title="待處理群組歷程記錄注入">
    未處理的群組訊息會先緩衝，並在機器人最終被觸發時注入為上下文。

    - 預設限制：`50`
    - 設定：`channels.whatsapp.historyLimit`，後備 `messages.groupChat.historyLimit`
    - `0` 會停用

    注入標記：`[Chat messages since your last reply - for context]` 和 `[Current message - respond to this]`。

  </Accordion>

  <Accordion title="讀取回條">
    對已接受的傳入訊息預設啟用。若要全域停用：

    ```json5
    { channels: { whatsapp: { sendReadReceipts: false } } }
    ```

    每個帳號的覆寫設定：`channels.whatsapp.accounts.<id>.sendReadReceipts`。即使已全域啟用，與自己的聊天仍會略過讀取回條。

  </Accordion>
</AccordionGroup>

## 傳送、分塊與媒體

<AccordionGroup>
  <Accordion title="文字分塊">
    - 預設分塊限制：`channels.whatsapp.textChunkLimit = 4000`
    - `channels.whatsapp.streaming.chunkMode = "length" | "newline"`；`newline` 優先使用段落邊界（空白行），然後退回至符合長度安全限制的分塊方式

  </Accordion>

  <Accordion title="傳出媒體行為">
    - 支援圖片、影片、音訊（PTT 語音留言）和文件酬載
    - 音訊會以 Baileys `audio` 酬載搭配 `ptt: true` 傳送，呈現為即按即說語音留言；回覆酬載會保留 `audioAsVoice`，讓 TTS 語音留言輸出無論提供者的來源格式為何，皆維持使用此路徑
    - 原生 Ogg/Opus 音訊會以 `audio/ogg; codecs=opus` 傳送；其他所有格式（包括 Microsoft Edge TTS 的 MP3/WebM 輸出）都會在 PTT 傳送前，使用 `ffmpeg` 轉碼為 48 kHz 單聲道 Ogg/Opus
    - `/tts latest` 會將最新的助理回覆作為一則語音留言傳送，並避免重複傳送同一則回覆；`/tts chat on|off|default` 控制目前聊天的自動 TTS
    - 傳送影片時使用 `gifPlayback: true`，可啟用 GIF 動畫播放
    - `forceDocument`/`asDocument` 會透過 Baileys 文件酬載傳送圖片、GIF 和影片，以避免 WhatsApp 的媒體壓縮，並保留解析後的檔名與 MIME 類型
    - 在多媒體回覆中，說明文字會套用至第一個媒體項目，但 PTT 語音留言除外：音訊會先傳送且不含說明文字，接著再以獨立文字訊息傳送說明文字（WhatsApp 用戶端無法一致地顯示語音留言說明文字）
    - 媒體來源可以是 HTTP(S)、`file://` 或本機路徑

  </Accordion>

  <Accordion title="媒體大小限制與後備行為">
    - 傳入儲存上限與傳出傳送上限：`channels.whatsapp.mediaMaxMb`（預設為 `50`）
    - 每個帳號的覆寫設定：`channels.whatsapp.accounts.<id>.mediaMaxMb`
    - 除非 `forceDocument`/`asDocument` 要求以文件形式傳送，否則圖片會自動最佳化（調整大小／逐步調整品質）以符合限制
    - 媒體傳送失敗時，第一個項目的後備機制會傳送文字警告，而不會無聲地捨棄回覆

  </Accordion>
</AccordionGroup>

## 回覆引用

`channels.whatsapp.replyToMode` 控制原生回覆引用（傳出回覆會明顯引用傳入訊息）：

| 值                | 行為                                       |
| ----------------- | ------------------------------------------ |
| `"off"`（預設） | 永不引用；以一般訊息傳送                   |
| `"first"`         | 僅引用第一個傳出回覆區塊                   |
| `"all"`         | 引用每個傳出回覆區塊                       |
| `"batched"`         | 引用佇列中的批次回覆；即時回覆不加引用     |

每個帳號的覆寫設定：`channels.whatsapp.accounts.<id>.replyToMode`。

```json5
{ channels: { whatsapp: { replyToMode: "first" } } }
```

## 表情回應層級

`channels.whatsapp.reactionLevel` 控制代理程式使用表情符號回應的廣泛程度：

| 層級                  | 確認表情回應 | 代理程式主動發起的表情回應 |
| --------------------- | ------------ | -------------------------- |
| `"off"`    | 否           | 否                         |
| `"ack"`    | 是           | 否                         |
| `"minimal"`（預設） | 是           | 是，保守使用               |
| `"extensive"`    | 是           | 是，鼓勵使用               |

每個帳號的覆寫設定：`channels.whatsapp.accounts.<id>.reactionLevel`。

```json5
{ channels: { whatsapp: { reactionLevel: "ack" } } }
```

## 確認表情回應

`channels.whatsapp.ackReaction` 會在收到傳入訊息時立即傳送表情回應，並受 `reactionLevel` 限制（若為 `"off"` 則不傳送）：

```json5
{
  channels: {
    whatsapp: {
      ackReaction: {
        emoji: "👀",
        direct: true,
        group: "mentions", // always | mentions | never
      },
    },
  },
}
```

注意事項：在傳入訊息獲接受後立即傳送（回覆前）；如果有 `ackReaction` 但沒有 `emoji`，WhatsApp 會使用路由所指代理程式身分的表情符號，並以 "👀" 作為後備（若不需要確認表情回應，請省略 `ackReaction` 或設定 `emoji: ""`）；失敗會記錄於日誌，但不會阻擋回覆傳送；群組模式 `mentions` 僅會在由提及觸發的回合中做出表情回應，而群組啟用方式 `always` 會略過這項檢查；WhatsApp 僅使用 `channels.whatsapp.ackReaction`（舊版 `messages.ackReaction` 不適用於此處）。

## 生命週期狀態表情回應

設定 `messages.statusReactions.enabled: true`，讓 WhatsApp 在回合期間取代確認表情回應，而非保留靜態的收件表情符號，並依序切換已排入佇列、思考中、工具活動、壓縮、完成和錯誤等狀態：

```json5
{
  messages: {
    statusReactions: {
      enabled: true,
    },
  },
}
```

注意事項：`channels.whatsapp.ackReaction` 仍會控制私人訊息與群組的適用資格；已排入佇列狀態使用與一般確認表情回應相同的有效表情符號；WhatsApp 每則訊息只有一個機器人表情回應欄位，因此生命週期更新會直接取代目前的表情回應，並在最終完成／錯誤狀態後還原確認表情回應。

## 多帳號與認證資訊

<AccordionGroup>
  <Accordion title="帳號選擇與預設值">
    帳號 ID 來自 `channels.whatsapp.accounts`。若有 `default`，預設會選擇該帳號，否則選擇第一個已設定的帳號 ID（依字母順序排序）。帳號 ID 會在內部正規化，以供查詢。
  </Accordion>

  <Accordion title="認證資訊路徑與舊版相容性">
    - 目前的驗證路徑：`~/.openclaw/credentials/whatsapp/<accountId>/creds.json`（備份：`creds.json.bak`）
    - 仍會辨識／遷移 `~/.openclaw/credentials/` 中的舊版預設驗證，以供預設帳號流程使用

  </Accordion>

  <Accordion title="登出行為">
    `openclaw channels logout --channel whatsapp [--account <id>]` 會清除該帳號的 WhatsApp 驗證狀態。若可連線至閘道，登出時會先停止該帳號的即時監聽器，讓已連結的工作階段在下次重新啟動前停止接收訊息。`openclaw channels remove --channel whatsapp` 也會在停用或刪除帳號設定前停止即時監聽器。

    在舊版驗證目錄中，移除 Baileys 驗證檔案時會保留 `oauth.json`。

  </Accordion>
</AccordionGroup>

## 工具、動作與設定寫入

- 代理程式工具支援包含 WhatsApp 表情回應動作（`react`）。
- 動作閘門：`channels.whatsapp.actions.reactions`、`channels.whatsapp.actions.polls`（現有動作預設為 `true`）、`channels.whatsapp.actions.calls`（預設為 `false`，請參閱上方的 MeowCaller）。
- 預設允許由頻道發起設定寫入；可透過 `channels.whatsapp.configWrites: false` 停用。

## 疑難排解

<AccordionGroup>
  <Accordion title="尚未連結（需要 QR 碼）">
    症狀：頻道狀態顯示尚未連結。

```bash
openclaw channels login --channel whatsapp
openclaw channels status
```

  </Accordion>

  <Accordion title="已連結但中斷連線／重連迴圈">
    症狀：已連結的帳號反覆中斷連線或嘗試重新連線。

    活動較少的帳號即使超過一般訊息逾時仍可保持連線；只有在 WhatsApp Web 傳輸活動停止、通訊端關閉，或應用程式層級活動的靜默時間超過較長的安全時限時，監視程式才會重新啟動（請參閱上方的執行階段模型）。

    修正方式：

    ```bash
    openclaw channels status --probe
    openclaw doctor
    openclaw logs --follow
    openclaw gateway status
    ```

    若修正主機連線能力與時間設定後仍持續出現迴圈，請備份帳號驗證目錄並重新連結：

    ```bash
    cp -a ~/.openclaw/credentials/whatsapp/<accountId> \
      ~/.openclaw/credentials/whatsapp/<accountId>.bak
    openclaw channels logout --channel whatsapp --account <accountId>
    openclaw channels login --channel whatsapp --account <accountId>
    ```

    若 `~/.openclaw/logs/whatsapp-health.log` 顯示 `Gateway inactive`，但 `openclaw gateway status` 與 `openclaw channels status --probe` 均顯示運作正常，請執行 `openclaw doctor`。在 Linux 上，doctor 會針對叫用已淘汰之 `~/.openclaw/bin/ensure-whatsapp.sh` 指令碼的舊版 crontab 項目發出警告；請使用 `crontab -e` 移除這些項目——排程可能缺少 systemd 使用者匯流排環境，導致該舊指令碼錯誤回報閘道健康狀態。

  </Accordion>

  <Accordion title="透過 Proxy 登入時 QR 碼逾時">
    症狀：`openclaw channels login --channel whatsapp` 在顯示可用的 QR 碼前，因 `status=408 Request Time-out` 或 TLS 通訊端中斷連線而失敗。

    WhatsApp Web 登入會使用閘道主機的標準 Proxy 環境（`HTTPS_PROXY`、`HTTP_PROXY`、小寫變體、`NO_PROXY`）。確認閘道處理程序有繼承 Proxy 環境，且 `NO_PROXY` 不符合 `mmg.whatsapp.net`。

  </Accordion>

  <Accordion title="傳送時沒有使用中的監聽器">
    若目標帳號沒有使用中的閘道監聽器，傳出訊息會立即失敗。請確認閘道正在執行，且帳號已連結。
  </Accordion>

  <Accordion title="回覆出現在逐字稿中，但未出現在 WhatsApp">
    逐字稿資料列會記錄代理程式產生的內容；WhatsApp 傳遞則會另外檢查。只有在 Baileys 針對至少一次可見的文字或媒體傳送回傳傳出訊息 ID 後，OpenClaw 才會將自動回覆視為已傳送。

    確認表情回應是獨立於回覆前的收件確認——表情回應成功不代表後續的文字／媒體回覆已獲接受。請檢查閘道日誌中是否有 `auto-reply delivery failed` 或 `auto-reply was not accepted by WhatsApp provider`。

  </Accordion>

  <Accordion title="群組訊息意外遭到忽略">
    請依此順序檢查：`groupPolicy`、`groupAllowFrom`/`allowFrom`、`groups` 允許清單項目、提及閘門（`requireMention` + 提及模式），以及 `openclaw.json` 中的重複鍵（JSON5 中較後面的項目會覆寫較前面的項目——每個範圍僅保留一個 `groupPolicy`）。

    若有 `channels.whatsapp.groups`，WhatsApp 仍可觀察其他群組的訊息，但 OpenClaw 會在工作階段路由前捨棄這些訊息。請將群組 JID 新增至 `channels.whatsapp.groups`，或新增 `groups["*"]` 以允許所有群組，同時繼續由 `groupPolicy`/`groupAllowFrom` 控制傳送者授權。

  </Accordion>

  <Accordion title="Bun 執行階段警告">
    OpenClaw 閘道需要 Node。Bun 不提供標準狀態儲存區所使用的 `node:sqlite` API，而 doctor 會將舊版 Bun 服務遷移至 Node。
  </Accordion>
</AccordionGroup>

## 系統提示詞

WhatsApp 可透過 `groups` 與 `direct` 對應，為群組和直接聊天提供 Telegram 風格的系統提示詞。

群組訊息的解析方式：首先決定實際使用的 `groups` 對應——只要帳號有定義自己的 `groups` 鍵，就會完全取代根層級的 `groups` 對應（不進行深層合併）。接著只會在這個最終產生的單一對應中查詢提示詞：

1. **群組專屬提示詞**（`groups["<groupId>"].systemPrompt`）：當群組項目存在，**且**其 `systemPrompt` 鍵已有定義時使用。空字串（`""`）會抑制萬用字元，且不套用任何提示詞。
2. **群組萬用字元提示詞**（`groups["*"].systemPrompt`）：當特定群組項目不存在，或該項目存在但沒有 `systemPrompt` 鍵時使用。

直接訊息會依照相同模式，對 `direct` 對應與 `direct["*"]` 進行解析。

<Note>
`dms` 仍是輕量的個別私訊歷程覆寫儲存區（`dms.<id>.historyLimit`）。提示詞覆寫位於 `direct` 下。
</Note>

<Note>
這種在提示詞解析中以帳號取代根層級的行為，是一般的淺層覆寫：任何帳號的 `groups`/`direct` 鍵（包括明確指定的空物件）都會取代根層級對應。這與上述群組成員資格允許清單檢查不同；後者針對意外為空的 `groups: {}` 提供單一帳號安全網。
</Note>

**與 Telegram 的差異：**在多帳號設定中，Telegram 會對每個帳號抑制根層級的 `groups`（即使帳號本身沒有 `groups`），以防止機器人接收其未加入之群組的訊息。WhatsApp 不會套用此防護——任何沒有自身覆寫的帳號都會繼承根層級的 `groups`/`direct`，不受帳號數量影響。在多帳號 WhatsApp 設定中，若要使用個別帳號提示詞，請在每個帳號下明確定義完整對應。

重要行為：

- `channels.whatsapp.groups` 同時是個別群組設定對應，以及聊天層級的群組允許清單。在根層級或帳號範圍中，`groups["*"]` 都表示該範圍“允許所有群組”。
- 只有在確定要讓該範圍允許所有群組時，才新增萬用字元 `systemPrompt`。若只要讓固定的一組群組 ID 符合資格，請在每個明確列入允許清單的項目中重複提示詞，而不要使用 `groups["*"]`。
- 群組准入與傳送者授權是分開的檢查。`groups["*"]` 會擴大可進入群組處理程序的群組範圍，但不會授權這些群組中的所有傳送者——傳送者授權仍由 `groupPolicy`/`groupAllowFrom` 控制。
- `channels.whatsapp.direct` 對私訊沒有相同的附帶作用：`direct["*"]` 只會在私訊已由 `dmPolicy` 加上 `allowFrom` 或配對儲存區規則允許後，提供預設設定。

範例：

```json5
{
  channels: {
    whatsapp: {
      groups: {
        // 僅在根層級範圍應允許所有群組時使用。
        // 套用至所有未定義自身 groups 對應的帳號。
        "*": { systemPrompt: "所有群組的預設提示詞。" },
      },
      direct: {
        // 套用至所有未定義自身 direct 對應的帳號。
        "*": { systemPrompt: "所有直接聊天的預設提示詞。" },
      },
      accounts: {
        work: {
          groups: {
            // 此帳號定義了自己的 groups，因此根層級 groups 會被完全
            // 取代。若要保留萬用字元，也請在此明確定義 "*"。
            "120363406415684625@g.us": {
              requireMention: false,
              systemPrompt: "專注於專案管理。",
            },
            // 僅在此帳號應允許所有群組時使用。
            "*": { systemPrompt: "工作群組的預設提示詞。" },
          },
          direct: {
            // 此帳號定義了自己的 direct 對應，因此根層級 direct 項目會被
            // 完全取代。若要保留萬用字元，也請在此明確定義 "*"。
            "+15551234567": { systemPrompt: "特定工作直接聊天的提示詞。" },
            "*": { systemPrompt: "工作直接聊天的預設提示詞。" },
          },
        },
      },
    },
  },
}
```

## 設定參考指引

主要參考資料：[設定參考 - WhatsApp](/zh-TW/gateway/config-channels#whatsapp)

| 區域             | 欄位                                                                                                         |
| ---------------- | -------------------------------------------------------------------------------------------------------------- |
| 存取權           | `dmPolicy`、`allowFrom`、`groupPolicy`、`groupAllowFrom`、`groups`                                             |
| 傳遞             | `textChunkLimit`、`streaming.chunkMode`、`mediaMaxMb`、`sendReadReceipts`、`ackReaction`、`reactionLevel`      |
| 多帳號           | `accounts.<id>.enabled`、`accounts.<id>.authDir`，以及其他個別帳號覆寫                              |
| 操作             | `configWrites`、`enabled`                                                                                      |
| 傳入批次處理     | `messages.inbound.debounceMs`、`messages.inbound.byChannel.whatsapp`                                           |
| 工作階段行為     | `session.dmScope`、`historyLimit`、`dmHistoryLimit`、`dms.<id>.historyLimit`                                   |
| 提示詞           | `groups.<id>.systemPrompt`、`groups["*"].systemPrompt`、`direct.<id>.systemPrompt`、`direct["*"].systemPrompt` |

## 相關內容

- [配對](/zh-TW/channels/pairing)
- [群組](/zh-TW/channels/groups)
- [安全性](/zh-TW/gateway/security)
- [頻道路由](/zh-TW/channels/channel-routing)
- [多代理程式路由](/zh-TW/concepts/multi-agent)
- [疑難排解](/zh-TW/channels/troubleshooting)
