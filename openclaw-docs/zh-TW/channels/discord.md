---
read_when:
    - 開發 Discord 頻道功能
summary: Discord 機器人設定、設定鍵、元件、語音與疑難排解
title: Discord
x-i18n:
    generated_at: "2026-07-26T08:06:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52a2926217f3a8dfb9398551ddacb0bc6aae6de0a164b215c55256eda9b6245e
    source_path: channels/discord.md
    workflow: 16
---

OpenClaw 透過官方 Discord 閘道以機器人身分連線至 Discord。支援私訊與伺服器頻道。

<CardGroup cols={3}>
  <Card title="配對" icon="link" href="/zh-TW/channels/pairing">
    Discord 私訊預設使用配對模式。
  </Card>
  <Card title="斜線命令" icon="terminal" href="/zh-TW/tools/slash-commands">
    原生命令行為與命令目錄。
  </Card>
  <Card title="頻道疑難排解" icon="wrench" href="/zh-TW/channels/troubleshooting">
    跨頻道診斷與修復流程。
  </Card>
</CardGroup>

## 快速設定

建立含機器人的 Discord 應用程式，將機器人加入你的伺服器，並與 OpenClaw 配對。可以的話，請使用私人伺服器；如有需要，請先[建立一個](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server)（**Create My Own > For me and my friends**）。

<Steps>
  <Step title="建立 Discord 應用程式與機器人">
    在 [Discord Developer Portal](https://discord.com/developers/applications) 中，按一下 **New Application** 並為其命名（例如 “OpenClaw”）。

    在側邊欄開啟 **Bot**，並將 **Username** 設為你的代理程式名稱。

  </Step>

  <Step title="啟用特殊權限意圖">
    仍在 **Bot** 頁面的 **Privileged Gateway Intents** 下，啟用：

    - **Message Content Intent**（必要）
    - **Server Members Intent**（建議；角色允許清單、名稱對 ID 配對及頻道受眾存取群組所必需）
    - **Presence Intent**（選用；僅用於上線狀態更新）

  </Step>

  <Step title="複製你的機器人權杖">
    在 **Bot** 頁面按一下 **Reset Token**，然後複製權杖。

    <Note>
    儘管名稱如此，這會產生你的第一個權杖，並沒有任何內容被“重設”。
    </Note>

  </Step>

  <Step title="產生邀請 URL 並將機器人加入伺服器">
    在側邊欄開啟 **OAuth2**。在 **OAuth2 URL Generator** 中啟用以下範圍：

    - `bot`
    - `applications.commands`

    在出現的 **Bot Permissions** 區段中，至少啟用：

    **General Permissions**
      - View Channels

    **Text Permissions**
      - Send Messages
      - Read Message History
      - Embed Links
      - Attach Files
      - Add Reactions（選用）

    這是一般文字頻道的基本需求。如果機器人會在討論串中發文，包括會建立或延續討論串的論壇或媒體頻道工作流程，也請啟用 **Send Messages in Threads**。

    複製產生的 URL，在瀏覽器中開啟，選取你的伺服器，然後按一下 **Continue**。機器人現在應會出現在你的伺服器中。

  </Step>

  <Step title="啟用開發者模式並收集你的 ID">
    在 Discord 應用程式中啟用開發者模式，以便複製 ID：

    1. **User Settings**（齒輪圖示）→ **Developer** → 開啟 **Developer Mode**
       *（行動版：**App Settings** → **Advanced**）*
    2. 在你的**伺服器圖示**上按一下滑鼠右鍵 → **Copy Server ID**
    3. 在你**自己的頭像**上按一下滑鼠右鍵 → **Copy User ID**

    將伺服器 ID 和使用者 ID 與機器人權杖放在一起；下一步需要這三項資訊。

  </Step>

  <Step title="允許來自伺服器成員的私訊">
    若要進行配對，Discord 必須允許機器人傳送私訊給你。在你的**伺服器圖示**上按一下滑鼠右鍵 → **Privacy Settings** → 開啟 **Direct Messages**。

    如果你會搭配 OpenClaw 使用 Discord 私訊，請保持啟用。如果你只使用伺服器頻道，可以在配對後停用。

  </Step>

  <Step title="安全地設定機器人權杖（不要在聊天中傳送）">
    機器人權杖是機密資訊。在傳訊息給代理程式之前，請先在執行 OpenClaw 的機器上設定：

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
cat > discord.patch.json5 <<'JSON5'
{
  channels: {
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./discord.patch.json5 --dry-run
openclaw config patch --file ./discord.patch.json5
openclaw gateway
```

    如果 OpenClaw 已作為背景服務執行，請透過 OpenClaw Mac 應用程式重新啟動，或停止後重新啟動 `openclaw gateway run` 處理程序。
    若為受管理的服務安裝，請從已設定 `DISCORD_BOT_TOKEN` 的 shell 執行 `openclaw gateway install`，或將變數儲存在 `~/.openclaw/.env`，讓服務在重新啟動後能解析環境變數 SecretRef。
    如果你的主機受到 Discord 啟動時應用程式查詢的封鎖或速率限制，請設定 Developer Portal 中的應用程式／用戶端 ID，讓啟動程序略過該 REST 呼叫：預設帳號使用 `channels.discord.applicationId`，或為每個機器人設定 `channels.discord.accounts.<accountId>.applicationId`。

  </Step>

  <Step title="設定 OpenClaw 並配對">

    <Tabs>
      <Tab title="詢問你的代理程式">
        在現有頻道（例如 Telegram）上與你的 OpenClaw 代理程式交談，並告知它。如果 Discord 是你的第一個頻道，請改用命令列介面／設定分頁。

        > “我已在設定中設定 Discord 機器人權杖。請使用使用者 ID `<user_id>` 和伺服器 ID `<server_id>` 完成 Discord 設定。”
      </Tab>
      <Tab title="命令列介面／設定">
        檔案式設定：

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: {
        source: "env",
        provider: "default",
        id: "DISCORD_BOT_TOKEN",
      },
    },
  },
}
```

        預設帳號的環境變數後援：

```bash
DISCORD_BOT_TOKEN=...
```

        若要進行指令碼式或遠端設定，請使用 `openclaw config patch --file ./discord.patch.json5 --dry-run` 寫入相同的 JSON5 區塊，然後移除 `--dry-run` 後重新執行。純文字 `token` 字串也可使用，且 `channels.discord.token` 支援環境變數／檔案／執行提供者的 SecretRef 值。請參閱[機密資訊管理](/zh-TW/gateway/secrets)。

        若使用多個 Discord 機器人，請將每個機器人的權杖和應用程式 ID 保留在各自的帳號下。最上層的 `channels.discord.applicationId` 會由帳號繼承，因此只有在每個帳號都使用相同應用程式 ID 時，才在該處設定。

```json5
{
  channels: {
    discord: {
      enabled: true,
      accounts: {
        personal: {
          token: { source: "env", provider: "default", id: "DISCORD_PERSONAL_TOKEN" },
          applicationId: "111111111111111111",
        },
        work: {
          token: { source: "env", provider: "default", id: "DISCORD_WORK_TOKEN" },
          applicationId: "222222222222222222",
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="核准第一次私訊配對">
    閘道執行後，在 Discord 中私訊你的機器人。它會回覆配對代碼。

    <Tabs>
      <Tab title="詢問你的代理程式">
        在現有頻道上將配對代碼傳送給你的代理程式：

        > “核准此 Discord 配對代碼：`<CODE>`”
      </Tab>
      <Tab title="命令列介面">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    配對代碼會在 1 小時後到期。核准後，即可在 Discord 私訊中與你的代理程式交談。

  </Step>
</Steps>

<Note>
權杖解析會區分帳號。設定中的權杖值優先於環境變數後援，而 `DISCORD_BOT_TOKEN` 僅供預設帳號使用。
如果兩個已啟用的 Discord 帳號解析為相同的機器人權杖，OpenClaw 只會為該權杖啟動一個閘道監控器：來自設定的權杖優先於環境變數後援；否則由第一個已啟用的帳號優先，重複帳號則會回報為已停用，原因為 `duplicate bot token`。
針對進階輸出呼叫（訊息工具／頻道動作），會使用每次呼叫明確指定的 `token`。這適用於傳送和讀取／探測類動作（讀取／搜尋／擷取／討論串／釘選／權限）。帳號原則／重試設定仍來自使用中執行階段快照內選定的帳號。
</Note>

## 建議：設定伺服器工作區

私訊可正常運作後，你可以將伺服器轉換為完整工作區，讓每個頻道都有具備各自內容脈絡的獨立代理程式工作階段。建議用於只有你和機器人的私人伺服器。

<Steps>
  <Step title="將你的伺服器加入伺服器允許清單">
    這可讓代理程式回應伺服器上的任何頻道，而不僅是私訊。

    <Tabs>
      <Tab title="詢問你的代理程式">
        > “將我的 Discord 伺服器 ID `<server_id>` 加入伺服器允許清單”
      </Tab>
      <Tab title="設定">

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="允許不使用 @提及的回應">
    預設情況下，代理程式只有在伺服器頻道中被 @提及時才會回應。在私人伺服器上，你可能會希望它回應每則訊息。

    在伺服器頻道中，一般回覆預設會自動發送。針對共用且持續開啟的聊天室，請選擇啟用 `messages.groupChat.visibleReplies: "message_tool"`，讓代理程式可以潛伏，並只在判斷頻道回覆有用時才發文。這最適合 GPT-5.6 Sol 等最新世代、工具使用可靠的模型。除非工具傳送訊息，否則環境聊天室事件會保持安靜。請參閱[環境聊天室事件](/zh-TW/channels/ambient-room-events)，以了解完整的潛伏模式設定。

    如果 Discord 顯示正在輸入，且記錄顯示有使用權杖，但未發出訊息，請檢查該輪是否設為環境聊天室事件，或是否選擇使用訊息工具提供可見回覆。

    <Tabs>
      <Tab title="詢問你的代理程式">
        > “允許我的代理程式在此伺服器上回應，而不必被 @提及”
      </Tab>
      <Tab title="設定">
        在你的伺服器設定中設定 `requireMention: false`：

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

        若要要求使用訊息工具傳送可見的群組／頻道回覆，請設定 `messages.groupChat.visibleReplies: "message_tool"`。

      </Tab>
    </Tabs>

  </Step>

  <Step title="規劃伺服器頻道中的記憶">
    長期記憶（MEMORY.md）只會在私訊工作階段中自動載入；伺服器頻道不會載入。

    <Tabs>
      <Tab title="詢問你的代理程式">
        > “當我在 Discord 頻道中提問時，如果需要 MEMORY.md 的長期內容脈絡，請使用 memory_search 或 memory_get。”
      </Tab>
      <Tab title="手動">
        若要在每個頻道中共用內容脈絡，請將穩定指示放在 `AGENTS.md` 或 `USER.md` 中（會注入每個工作階段）。將長期筆記保存在 `MEMORY.md` 中，並視需要使用記憶工具存取。
      </Tab>
    </Tabs>

  </Step>
</Steps>

現在建立頻道並開始聊天。代理程式可以看到頻道名稱，而且每個頻道都是隔離的工作階段——請設定 `#coding`、`#home`、`#research`，或任何適合你工作流程的頻道。

## 執行階段模型

- 閘道負責 Discord 連線。
- 回覆路由是確定性的：Discord 的輸入回覆至 Discord。
- Discord 伺服器／頻道中繼資料會以不受信任的內容脈絡加入模型提示，而不是作為使用者可見的回覆前綴。如果模型將該封套複製回來，OpenClaw 會從輸出回覆和未來的重播內容脈絡中移除複製的中繼資料。
- 預設情況下（`session.dmScope=main`），直接聊天會共用代理程式的主要工作階段（`agent:main:main`）。
- 伺服器頻道使用隔離的工作階段鍵（`agent:<agentId>:discord:channel:<channelId>`）。
- 群組私訊預設會被忽略（`channels.discord.dm.groupEnabled=false`）。
- 原生斜線命令會在隔離的命令工作階段中執行（`agent:<agentId>:discord:slash:<userId>`），同時仍會將 `CommandTargetSessionKey` 帶入路由後的對話工作階段。
- 傳送至 Discord 的純文字排程／心跳偵測公告會合併為最終可供助理查看的答案，並只傳送一次。當代理程式產生多個可傳送的承載資料時，媒體與結構化元件承載資料仍會保留為多則訊息。

## 論壇頻道

Discord 論壇與媒體頻道僅接受討論串貼文。OpenClaw 支援兩種建立方式：

- 傳送訊息至論壇上層頻道（`channel:<forumId>`），以自動建立討論串。討論串標題取自訊息中第一個非空白行（截斷至 Discord 的 100 字元討論串名稱限制）。
- 使用 `openclaw message thread create` 直接建立討論串。對論壇頻道請勿傳入 `--message-id`。

傳送至論壇上層頻道以建立討論串：

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "主題標題\n貼文內容"
```

明確建立論壇討論串：

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "主題標題" --message "貼文內容"
```

論壇上層頻道不接受 Discord 元件。若需要元件，請傳送至討論串本身（`channel:<threadId>`）。

## 互動式元件

OpenClaw 支援在代理程式訊息中使用 Discord components v2 容器。請搭配 `components` 承載資料使用訊息工具。互動結果會以一般傳入訊息的形式路由回代理程式，並遵循現有的 Discord `replyToMode` 設定。

支援的區塊：

- `text`、`section`、`separator`、`actions`、`media-gallery`、`file`
- 動作列最多允許 5 個按鈕或一個選取選單
- 選取類型：`string`、`user`、`role`、`mentionable`、`channel`

元件預設只能使用一次。設定 `components.reusable=true`，可讓按鈕、選取項目與表單在到期前重複使用。

若要限制誰能點擊按鈕，請在該按鈕上設定 `allowedUsers`（Discord 使用者 ID、標籤或 `*`）。不符合的使用者會收到僅自己可見的拒絕訊息。

元件回呼預設會在 30 分鐘後到期。設定 `channels.discord.agentComponents.ttlMs` 可變更預設帳號的回呼登錄存續時間，或為個別帳號設定 `channels.discord.accounts.<accountId>.agentComponents.ttlMs`。此值以毫秒為單位，必須是正整數，且上限為 `86400000`（24 小時）。較長的 TTL 適合需要讓按鈕持續可用的審查／核准工作流程，但也會延長舊 Discord 訊息仍可觸發動作的時間範圍。請優先使用符合需求的最短 TTL；若過期回呼可能造成意外，請保留預設值。

`/model` 與 `/models` 斜線命令會開啟互動式模型選擇器，其中包含供應商、模型及相容執行環境的下拉式選單，並附有提交步驟。`/models add` 已淘汰，會傳回淘汰訊息，而不再從聊天中登錄模型。選擇器回覆僅自己可見，且只有叫用它的使用者能操作。Discord 選取選單最多只能包含 25 個選項，因此若希望選擇器只為 `openai` 或 `vllm` 等指定供應商顯示動態探索到的模型，請將 `provider/*` 項目新增至 `agents.defaults.modelPolicy.allow`。

檔案附件：

- `file` 區塊必須指向附件參照（`attachment://<filename>`）
- 透過 `media`/`path`/`filePath` 提供附件（單一檔案）；多個檔案請使用 `media-gallery`
- 若上傳名稱應與附件參照一致，請使用 `filename` 覆寫上傳名稱

互動視窗表單：

- 新增 `components.modal`，最多可包含 5 個欄位
- 欄位類型：`text`、`checkbox`、`radio`、`select`、`role-select`、`user-select`
- OpenClaw 會自動新增觸發按鈕

範例：

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "選用的備用文字",
  components: {
    reusable: true,
    text: "選擇路徑",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "核准",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "拒絕", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "選擇一個選項",
          options: [
            { label: "選項 A", value: "a" },
            { label: "選項 B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "詳細資料",
      triggerLabel: "開啟表單",
      fields: [
        { type: "text", label: "申請者" },
        {
          type: "select",
          label: "優先順序",
          options: [
            { label: "低", value: "low" },
            { label: "高", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## 存取控制與路由

<Tabs>
  <Tab title="私訊政策">
    `channels.discord.dmPolicy` 控制私訊存取。`channels.discord.allowFrom` 是正式的私訊允許清單。

    - `pairing`（預設）
    - `allowlist`（至少需要一個 `allowFrom` 傳送者）
    - `open`（要求 `channels.discord.allowFrom` 包含 `"*"`）
    - `disabled`

    若私訊政策並非開放，未知使用者將被封鎖（或在 `pairing` 模式中收到配對提示）。

    多帳號優先順序：

    - `channels.discord.accounts.default.allowFrom` 僅套用至 `default` 帳號。
    - 對單一帳號而言，`allowFrom` 的優先順序高於舊版 `dm.allowFrom`。
    - 當具名帳號本身的 `allowFrom` 與舊版 `dm.allowFrom` 均未設定時，會繼承 `channels.discord.allowFrom`。
    - 具名帳號不會繼承 `channels.discord.accounts.default.allowFrom`。

    為維持相容性，仍會讀取舊版 `channels.discord.dm.policy` 與 `channels.discord.dm.allowFrom`。若能在不變更存取權的情況下完成，`openclaw doctor --fix` 會將其遷移至 `dmPolicy` 與 `allowFrom`。

    用於傳遞的私訊目標格式：

    - `user:<id>`
    - `<@id>` 提及

    啟用頻道預設值時，單純的數字 ID 通常會解析為頻道 ID；但為維持相容性，列於帳號有效私訊 `allowFrom` 中的 ID 會視為使用者私訊目標。

  </Tab>

  <Tab title="存取群組">
    Discord 私訊與文字命令授權可以使用 `channels.discord.allowFrom` 中的動態 `accessGroup:<name>` 項目。

    存取群組名稱會在各訊息頻道之間共用。若要建立成員以各頻道一般 `allowFrom` 語法表示的靜態群組，請使用 `type: "message.senders"`；若要以 Discord 頻道目前的 `ViewChannel` 對象動態定義成員資格，請使用 `type: "discord.channelAudience"`。共用的存取群組行為：[存取群組](/zh-TW/channels/access-groups)。

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        "*": ["global-owner-id"],
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
      },
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
  },
}
```

    Discord 文字頻道沒有獨立的成員清單。`type: "discord.channelAudience"` 對成員資格的定義如下：私訊傳送者是所設定伺服器的成員，且套用身分組與頻道覆寫後，目前對所設定頻道具有有效的 `ViewChannel` 權限。

    範例：允許任何可看見 `#maintainers` 的人向機器人傳送私訊，同時對其他所有人關閉私訊。

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
      membership: "canViewChannel",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers"],
    },
  },
}
```

    你可以混用動態與靜態項目：

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers", "discord:123456789012345678"],
    },
  },
}
```

    查詢採取失敗即拒絕的方式。如果 Discord 傳回 `Missing Access`、成員查詢失敗，或頻道屬於不同伺服器，私訊傳送者會被視為未經授權。

    使用頻道對象存取群組時，請在 Discord Developer Portal 啟用 **Server Members Intent**。私訊不包含伺服器成員狀態，因此 OpenClaw 會在授權時透過 Discord REST 解析成員。

  </Tab>

  <Tab title="伺服器政策">
    伺服器處理由 `channels.discord.groupPolicy` 控制：

    - `open`
    - `allowlist`
    - `disabled`

    當 `channels.discord` 存在時，安全基準為 `allowlist`。

    `allowlist` 行為：

    - 伺服器必須符合 `channels.discord.guilds`（建議使用 `id`，也接受代稱）
    - 選用的傳送者允許清單：`users`（建議使用穩定 ID）與 `roles`（僅限身分組 ID）；若設定其中任一項，傳送者符合 `users`「或」`roles` 時即獲允許
    - 預設停用直接比對名稱／標籤；僅在緊急相容模式下啟用 `channels.discord.dangerouslyAllowNameMatching: true`
    - `users` 支援名稱／標籤，但 ID 較安全；使用名稱／標籤項目時，`openclaw security audit` 會發出警告
    - 若伺服器設定了 `channels`，未列出的頻道將遭拒絕
    - 若伺服器沒有 `channels` 區塊，該允許清單伺服器中的所有頻道都會獲允許

    範例：

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { enabled: true },
            help: { enabled: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    `openclaw doctor --fix` 會將舊版的個別頻道 `allow` 鍵遷移至 `enabled`。

    如果只設定 `DISCORD_BOT_TOKEN`，但未建立 `channels.discord` 區塊，則執行階段的備用值為 `groupPolicy="allowlist"`（日誌中會顯示警告），即使 `channels.defaults.groupPolicy` 是 `open` 亦然。

  </Tab>

  <Tab title="提及與群組私訊">
    伺服器訊息預設需要提及。

    提及偵測包括：

    - 明確提及機器人
    - 設定的提及模式（`agents.entries.*.groupChat.mentionPatterns`，備用值為 `messages.groupChat.mentionPatterns`）
    - 在支援的情況下，隱含的回覆機器人行為

    撰寫傳出的 Discord 訊息時，請使用標準提及語法：使用 `<@USER_ID>` 提及使用者、`<#CHANNEL_ID>` 提及頻道，以及 `<@&ROLE_ID>` 提及身分組。請勿使用舊版 `<@!USER_ID>` 暱稱提及格式。

    `requireMention` 會針對個別伺服器／頻道設定（`channels.discord.guilds...`）。
    `ignoreOtherMentions` 可選擇性捨棄提及其他使用者／身分組但未提及機器人的訊息（不包括 @everyone/@here）。

    群組私訊：

    - 預設：忽略（`dm.groupEnabled=false`）
    - 可透過 `dm.groupChannels` 設定選用的允許清單（頻道 ID 或代稱）

  </Tab>
</Tabs>

### 以身分組為基礎的代理程式路由

使用 `bindings[].match.roles`，依身分組 ID 將 Discord 伺服器成員路由至不同的代理程式。以身分組為基礎的繫結僅接受身分組 ID，且會在對等或上層對等繫結之後、僅限伺服器的繫結之前進行評估。若繫結也設定了其他比對欄位（例如 `peer` + `guildId` + `roles`），則所有已設定欄位都必須符合。

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## 原生命令與命令授權

- `commands.native` 預設為 `"auto"`，並已為 Discord 啟用。
- 每個頻道的覆寫設定：`channels.discord.commands.native`。
- `commands.native=false` 會略過啟動期間的 Discord 斜線命令註冊與清理。先前註冊的命令可能會繼續顯示在 Discord 中，直到你從 Discord 應用程式移除它們。
- 原生命令授權使用與一般訊息處理相同的 Discord 允許清單／原則。
- 未獲授權的使用者可能仍會在 Discord 使用者介面中看到命令；執行時會強制套用 OpenClaw 授權，並回覆 “not authorized”。
- 預設斜線命令設定：`ephemeral: true`（`channels.discord.slashCommand.ephemeral`）。

如需命令目錄與行為的詳細資訊，請參閱[斜線命令](/zh-TW/tools/slash-commands)。

## 功能詳細資訊

<AccordionGroup>
  <Accordion title="回覆標籤與原生回覆">
    Discord 支援代理程式輸出中的回覆標籤：

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    由 `channels.discord.replyToMode` 控制：

    - `off`（預設）：不會隱含建立回覆討論串；仍會遵循明確的 `[[reply_to_*]]` 標籤
    - `first`：將隱含的原生回覆參照附加至該回合第一則傳出的 Discord 訊息
    - `all`：將其附加至每一則傳出訊息
    - `batched`：僅在傳入事件是由多則訊息組成、經防彈跳處理的批次時附加；適合主要針對語意不明的突發聊天使用原生回覆，而不是每個僅含單則訊息的回合

    訊息 ID 會顯示在情境／歷史記錄中，讓代理程式可指定特定訊息。

  </Accordion>

  <Accordion title="連結預覽">
    Discord 預設會為 URL 產生豐富連結嵌入。OpenClaw 預設會抑制傳出 Discord 訊息中這些產生的嵌入，因此代理程式傳送的 URL 會維持為純連結，除非你選擇啟用：

```json5
{
  channels: {
    discord: {
      suppressEmbeds: false,
    },
  },
}
```

    設定 `channels.discord.accounts.<id>.suppressEmbeds` 可覆寫單一帳號。代理程式透過訊息工具傳送時，也可針對單則訊息傳入 `suppressEmbeds: false`。明確的 Discord `embeds` 承載資料不會受到預設連結預覽設定的抑制。

  </Accordion>

  <Accordion title="即時串流預覽">
    OpenClaw 可藉由傳送暫時訊息，並在文字抵達時編輯該訊息，以串流方式顯示回覆草稿。`channels.discord.streaming.mode` 接受 `off` | `partial` | `block` | `progress`（未設定 `streaming`/舊版 `streamMode` 鍵時的預設值）。`streamMode` 是舊版別名；執行 `openclaw doctor --fix` 可將持久化設定改寫為標準的巢狀 `streaming` 形狀。

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLines: 8,
          maxLineChars: 120,
          toolProgress: false,
          commentary: false,
        },
      },
    },
  },
}
```

    - `off` 會停用 Discord 預覽編輯。
    - `partial` 會在權杖抵達時編輯單一預覽訊息。
    - `block` 會發出草稿大小的區塊；可使用 `streaming.preview.chunk`（`minChars`、`maxChars`、`breakPreference`）調整大小與斷點，且上限為 `textChunkLimit`。明確啟用區塊串流時，OpenClaw 會略過預覽串流，以避免重複串流。
    - `progress` 會保留一則可編輯的狀態草稿，直到最終傳遞為止。預設會顯示代理程式最新前言或敘述的一行內容，不含產生的標籤、間隔或工具列。
    - 媒體、錯誤及明確回覆的最終訊息會取消待處理的預覽編輯。
    - `streaming.preview.toolProgress` 在 `partial`/`block` 模式中預設為 `true`。Discord 進度模式預設不顯示工具列；設定 `streaming.progress.toolProgress: true` 可選擇啟用。
    - 設定 `streaming.progress.toolProgress: true` 可新增精簡的工具／進度列，例如 `🛠️ Bash: run tests` 或 `🔎 Web Search: for "query"`。為維持相容性，現有的 `progress.label` 或 `progress.labels` 設定會保留先前的工具列預設值；若要使用自訂標籤但不顯示列，請設定 `toolProgress: false`。
    - `streaming.progress.commentary`（預設為 `false`）可選擇在暫時進度草稿中顯示原始助理評論。預設的前言／敘述狀態列不受此選項影響。評論會在顯示前清理、維持暫時性，且不會變更最終答案的傳遞方式。
    - `streaming.progress.maxLineChars` 控制每行進度預覽的額度。一般文字會在詞語邊界縮短；命令與路徑詳細資訊則會保留實用的尾端部分。
    - `streaming.preview.commandText` / `streaming.progress.commandText` 控制精簡進度列中的命令／執行詳細資訊：`raw`（預設）或 `status`（僅顯示工具標籤）。

    隱藏原始命令／執行文字，同時保留精簡進度列：

    ```json
    {
      "channels": {
        "discord": {
          "streaming": {
            "mode": "progress",
            "progress": {
              "toolProgress": true,
              "commandText": "status"
            }
          }
        }
      }
    }
    ```

    預覽串流僅支援文字；媒體回覆會退回一般傳遞方式。

  </Accordion>

  <Accordion title="歷史記錄、情境與討論串行為">
    伺服器歷史記錄情境：

    - `channels.discord.historyLimit` 預設為 `20`
    - 備援：`messages.groupChat.historyLimit`
    - `0` 會停用

    私訊歷史記錄控制：

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    討論串行為：

    - Discord 討論串會路由為頻道工作階段，並繼承父頻道設定，除非另有覆寫。
    - 討論串工作階段會繼承父頻道工作階段層級的 `/model` 選擇，僅作為模型備援；討論串本機的 `/model` 選擇具有優先權，且除非啟用對話記錄繼承，否則不會複製父層對話記錄歷史。
    - `channels.discord.thread.inheritParent`（預設為 `false`）可讓新的自動討論串選擇從父層對話記錄植入內容。每個帳號的覆寫設定：`channels.discord.accounts.<id>.thread.inheritParent`。
    - 訊息工具的反應可解析 `user:<id>` 私訊目標。
    - `guilds.<guild>.channels.<channel>.requireMention: false` 會在回覆階段啟用備援期間保留。

    頻道主題會以**不受信任**的情境注入。允許清單會限制誰能觸發代理程式，但並非完整的補充情境遮蔽邊界。

  </Accordion>

  <Accordion title="子代理程式的討論串綁定工作階段">
    Discord 可將討論串綁定至工作階段目標，使該討論串中的後續訊息持續路由至相同工作階段（包括子代理程式工作階段）。

    命令：

    - `/focus <target>` 將目前／新的討論串綁定至子代理程式／工作階段目標
    - `/unfocus` 移除目前的討論串綁定
    - `/agents` 顯示作用中的執行與綁定狀態
    - `/session idle <duration|off>` 檢視／更新已聚焦綁定因閒置而自動取消聚焦的設定
    - `/session max-age <duration|off>` 檢視／更新已聚焦綁定的最大存續時間硬性上限

    設定：

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
      spawnSessions: true,
      defaultSpawnContext: "fork",
    },
  },
}
```

    注意事項：

    - `session.threadBindings.*` 是 Discord 與 Telegram 的標準原則。
    - `spawnSessions` 控制為 `sessions_spawn({ thread: true })` 與 ACP 討論串衍生作業自動建立／綁定討論串。預設值：`true`。
    - `defaultSpawnContext` 控制討論串綁定衍生作業的原生子代理程式情境。預設值：`"fork"`。
    - 已淘汰的 `spawnSubagentSessions`/`spawnAcpSessions` 鍵會由 `openclaw doctor --fix` 遷移。
    - 如果停用討論串綁定，將無法使用 `/focus` 及相關操作。

    請參閱[子代理程式](/zh-TW/tools/subagents)、[ACP 代理程式](/zh-TW/tools/acp-agents)與[設定參考](/zh-TW/gateway/configuration-reference)。

  </Accordion>

  <Accordion title="來源訊息上的子代理程式進度">
    設定 `channels.discord.subagentProgress: true`，即可在啟動父層執行的 Discord 訊息上顯示背景子執行活動。

```json5
{
  channels: {
    discord: {
      subagentProgress: true,
    },
  },
}
```

    子執行處於作用中時，OpenClaw 會讓 Discord 輸入狀態維持作用中最長一小時，並隨並行數量變化替換一個計數反應（`1️⃣` 至 `🔟`）；`🔟` 也代表 10 個以上。最後一個子執行結束後，計數反應會被移除。失敗、逾時或遭終止的子執行會留下 `🔴` 反應。

    此功能需選擇啟用，並使用固定的內部計時與表情符號預設值。機器人需要 **Add Reactions** 權限才能提供反應回饋。帳號層級的 `channels.discord.accounts.<id>.subagentProgress` 會覆寫頂層值。

  </Accordion>

  <Accordion title="持久 ACP 頻道綁定">
    若要使用穩定的「永遠在線」ACP 工作區，請設定以 Discord 對話為目標的頂層具型別 ACP 綁定。

    設定路徑：`bindings[]`，搭配 `type: "acp"` 與 `match.channel: "discord"`。

```json5
{
  agents: {
    entries: {
      codex: {
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
    },
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

    注意事項：

    - `/acp spawn codex --bind here` 會就地綁定目前的頻道或討論串，並讓未來訊息留在同一個 ACP 工作階段。討論串訊息會繼承父頻道綁定。
    - 在已綁定的頻道或討論串中，`/new` 與 `/reset` 會就地重設同一個 ACP 工作階段。暫時討論串綁定可在作用期間覆寫目標解析。
    - `spawnSessions` 會透過 `--thread auto|here` 控制子討論串的建立／綁定。

    如需綁定行為的詳細資訊，請參閱 [ACP 代理程式](/zh-TW/tools/acp-agents)。

  </Accordion>

  <Accordion title="反應通知">
    每個伺服器的反應通知模式（`guilds.<id>.reactionNotifications`）：

    - `off`
    - `own`（預設）
    - `all`
    - `allowlist`（使用 `guilds.<id>.users`）

    反應事件會轉換為系統事件，並附加至經路由的 Discord 工作階段。

  </Accordion>

  <Accordion title="在線狀態事件">
    選擇讓伺服器在真人成員從離線轉為在線時，觸發經路由的代理程式喚醒：

    ```json5
    {
      channels: {
        discord: {
          intents: { presence: true },
          guilds: {
            "111111111111111111": {
              presenceEvents: {
                channelId: "222222222222222222",
                users: ["333333333333333333"], // 選用；進一步縮小頻道檢視者範圍
                reconnectSuppressSeconds: 300, // 選用；新工作階段靜默期間（0 表示停用）
                burstLimit: 8, // 選用；每個突發時段的事件數上限
                burstWindowSeconds: 60, // 選用；滑動式突發偵測時段
              },
            },
          },
        },
      },
    }
    ```

    `presenceEvents` 需要為路由的代理程式啟用心跳偵測，並在 Discord Developer Portal 的應用程式 Bot 頁面啟用具特殊權限的 **Presence Intent**。OpenClaw 會從每個完整的 `GUILD_CREATE` 快照建立目前線上成員的初始狀態、路由觀察到的離線轉線上狀態變化，並將之後首次收到、來自未曾出現成員的線上訊號視為新近可用。該成員可能是在快照後上線或加入，因此事件不會斷言其先前的確切狀態。只有可檢視 `channelId` 的真人才符合資格：頻道和公開討論串需要對該頻道或父頻道具備 **View Channel** 權限，而私人討論串還需要成員資格或 **Manage Threads** 權限。`users` 可進一步縮小該受眾範圍。OpenClaw 會忽略機器人和未變更的線上狀態，並在閘道重新啟動後仍保留每位使用者八小時的冷卻時間。當 Discord 建立新的閘道工作階段並傳送 `READY` 時，OpenClaw 會在重建伺服器在線狀態期間，抑制由在線狀態衍生的事件 `reconnectSuppressSeconds`（預設 300，`0` 表示停用），以免再次觀察到的成員逐一喚醒代理程式。此外，它會針對每個伺服器，將成功排入佇列的事件速率限制為每個 `burstWindowSeconds` 滑動時段（預設 60）最多 `burstLimit` 個事件（預設 8），並針對每個伺服器的每段抑制期間只記錄一次。恢復的工作階段不會被視為新工作階段。Discord 會限制成員超過 75,000 人之伺服器的快照；在這種情況下，OpenClaw 要求先收到明確的離線更新才會問候。系統事件會攜帶不可變的使用者、伺服器和頻道 ID，而不嵌入可變的顯示名稱。代理程式會決定是否問候及問候方式。

  </Accordion>

  <Accordion title="確認回應">
    `ackReaction` 會在 OpenClaw 處理傳入訊息時傳送確認表情符號。

    解析順序：

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - 代理程式身分表情符號備援（`agents.entries.*.identity.emoji`，否則為 "👀"）

    注意事項：

    - Discord 接受 Unicode 表情符號或自訂表情符號名稱。
    - 使用 `""` 可對特定頻道或帳號停用此回應。

    **範圍（`messages.ackReactionScope`）：**

    值：`"all"`（私訊 + 群組，包括環境房間事件）、`"direct"`（僅私訊）、`"group-all"`（除環境房間事件外的所有群組訊息，不含私訊）、`"group-mentions"`（在群組中提及機器人時；**不含私訊**，預設值）、`"off"` / `"none"`（停用）。

    <Note>
    預設範圍（`"group-mentions"`）不會在直接訊息或環境房間事件中觸發確認回應。若要對傳入的 Discord 私訊和安靜房間事件加入確認回應，請將 `messages.ackReactionScope` 設為 `"all"`。
    </Note>

  </Accordion>

  <Accordion title="設定寫入">
    預設會啟用由頻道發起的設定寫入。這會影響 `/config set|unset` 流程（啟用命令功能時）。

    停用：

```json5
{
  channels: {
    discord: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="閘道代理伺服器">
    使用 `channels.discord.proxy`，透過 HTTP(S) 代理伺服器路由 Discord 閘道 WebSocket 流量和啟動時的 REST 查詢（應用程式 ID + 允許清單解析）。
    Discord 閘道 WebSocket 代理必須明確設定；WebSocket 連線不會繼承閘道程序中的環境代理變數。設定 `channels.discord.proxy` 後，啟動時的 REST 查詢會使用此代理伺服器。

```json5
{
  channels: {
    discord: {
      proxy: "http://proxy.example:8080",
    },
  },
}
```

    個別帳號覆寫：

```json5
{
  channels: {
    discord: {
      accounts: {
        primary: {
          proxy: "http://proxy.example:8080",
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="PluralKit 支援">
    啟用 PluralKit 解析，將代理訊息對應至系統成員身分：

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // 選用；私人系統需要此項目
      },
    },
  },
}
```

    注意事項：

    - 允許清單可使用 `pk:<memberId>`
    - 僅在 `channels.discord.dangerouslyAllowNameMatching: true` 時，才會依名稱／代稱比對成員顯示名稱
    - 查詢會使用原始訊息 ID 呼叫 PluralKit API
    - 若查詢失敗，代理訊息會被視為機器人訊息並捨棄，除非 `allowBots` 允許其通過

  </Accordion>

  <Accordion title="外送提及別名">
    當代理程式需要確定性地提及已知 Discord 使用者時，請使用 `mentionAliases`。鍵是省略開頭 `@` 的帳號代稱；值則是 Discord 使用者 ID。未知帳號代稱、`@everyone`、`@here`，以及 Markdown 程式碼範圍內的提及都會保持不變。

```json5
{
  channels: {
    discord: {
      mentionAliases: {
        SupportLead: "123456789012345678",
      },
      accounts: {
        ops: {
          mentionAliases: {
            OpsLead: "234567890123456789",
          },
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="在線狀態設定">
    當你設定狀態或活動欄位，或啟用自動在線狀態時，系統會套用在線狀態更新。

    僅設定狀態：

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    活動（設定 `activity` 時，自訂狀態是預設活動類型）：

```json5
{
  channels: {
    discord: {
      activity: "專注時間",
      activityType: 4,
    },
  },
}
```

    直播：

```json5
{
  channels: {
    discord: {
      activity: "即時程式設計",
      activityType: 1,
      activityUrl: "https://twitch.tv/openclaw",
    },
  },
}
```

    活動類型對照表：

    - 0: Playing
    - 1: Streaming（需要 `activityUrl`；而 `activityUrl` 又需要 `activityType: 1`）
    - 2: Listening
    - 3: Watching
    - 4: Custom（使用活動文字作為狀態內容；表情符號為選用）
    - 5: Competing

    自動在線狀態（執行階段健康訊號）：

```json5
{
  channels: {
    discord: {
      autoPresence: {
        enabled: true,
        intervalMs: 30000,
        minUpdateIntervalMs: 15000,
        exhaustedText: "權杖已耗盡",
      },
    },
  },
}
```

    自動在線狀態會將執行階段可用性對應至 Discord 狀態：健康 => online、降級或未知 => idle、耗盡或無法使用 => dnd。預設值：`intervalMs` 30000、`minUpdateIntervalMs` 15000（必須小於或等於 `intervalMs`）。選用文字覆寫：

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText`（支援 `{reason}` 預留位置）

  </Accordion>

  <Accordion title="Discord 中的核准">
    Discord 支援在私訊中使用按鈕處理核准，也可選擇在原始頻道中發布核准提示。

    設定路徑：

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers`（選用；可行時會備援至 `commands.ownerAllowFrom`）
    - `channels.discord.execApprovals.target`（`dm` | `channel` | `both`，預設值：`dm`）
    - `agentFilter`、`sessionFilter`、`cleanupAfterResolve`

    當 `enabled` 未設定或為 `"auto"`，且至少可解析一位核准者時，Discord 會自動啟用原生執行核准；核准者可來自 `execApprovals.approvers` 或 `commands.ownerAllowFrom`。Discord 不會從頻道 `allowFrom`、舊版 `dm.allowFrom` 或直接訊息 `defaultTo` 推斷執行核准者。若要明確停用 Discord 作為原生核准用戶端，請設定 `enabled: false`。

    對於 `/diagnostics` 和 `/export-trajectory` 等敏感且僅限擁有者使用的群組命令，OpenClaw 會私下傳送核准提示和最終結果。若叫用命令的擁有者具有 Discord 擁有者路由，它會先嘗試 Discord 私訊；否則會備援至 `commands.ownerAllowFrom` 中第一個可用的擁有者路由，例如 Telegram。

    當 `target` 為 `channel` 或 `both` 時，核准提示會顯示在頻道中。只有已解析的核准者能使用按鈕；其他使用者會收到僅自己可見的拒絕訊息。核准提示包含命令文字，因此只能在受信任的頻道中啟用頻道傳送。若無法從工作階段金鑰取得頻道 ID，OpenClaw 會改用私訊傳送。

    Discord 會呈現其他聊天頻道共用的核准按鈕；原生 Discord 轉接器主要負責加入核准者私訊路由和頻道分流。這些按鈕存在時，會成為主要的核准使用者體驗；只有當工具結果指出聊天核准無法使用，或手動核准是唯一途徑時，OpenClaw 才應包含手動 `/approve` 命令。若 Discord 原生核准執行階段未啟用，OpenClaw 會保持顯示本機確定性的 `/approve <id> <decision>` 提示。若執行階段已啟用，但原生卡片無法傳送至任何目標，OpenClaw 會在同一聊天中傳送備援通知，其中包含待處理核准中的完整 `/approve` 命令。

    閘道驗證和核准解析遵循共用的閘道用戶端合約（`plugin:` ID 透過 `plugin.approval.resolve` 解析；其他 ID 則透過 `exec.approval.resolve` 解析）。核准預設會在 30 分鐘後到期。

    請參閱[執行核准](/zh-TW/tools/exec-approvals)。

  </Accordion>
</AccordionGroup>

## 工具和動作閘門

Discord 訊息動作涵蓋訊息傳送、頻道管理、內容管理、在線狀態和中繼資料。

核心範例：

- 訊息傳送：`sendMessage`、`readMessages`、`editMessage`、`deleteMessage`、`threadReply`
- 回應：`react`、`reactions`、`emojiList`
- 內容管理：`timeout`、`kick`、`ban`
- 在線狀態：`setPresence`

`event-create` 動作接受選用的 `image` 參數（URL 或本機檔案路徑），用於設定排定事件的封面圖片。

動作閘門位於 `channels.discord.actions.*` 之下。

預設閘門行為：

| 動作群組                                                                                                                                                             | 預設值  |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------- |
| reactions, messages, threads, pins, polls, search, memberInfo, roleInfo, channelInfo, channels, voiceStatus, events, stickers, emojiUploads, stickerUploads, permissions | 已啟用  |
| roles                                                                                                                                                                    | 已停用 |
| moderation                                                                                                                                                               | 已停用 |
| presence                                                                                                                                                                 | 已停用 |

## Components v2 使用者介面

OpenClaw 使用 Discord components v2 進行執行核准與跨情境標記。Discord 訊息動作也可接受 `components` 以建立自訂使用者介面（進階功能；需要透過 discord 工具建構元件承載資料），而舊版 `embeds` 仍可使用，但不建議使用。

- `channels.discord.ui.components.accentColor` 設定 Discord 元件容器使用的強調色（十六進位）。各帳號設定：`channels.discord.accounts.<id>.ui.components.accentColor`。
- `channels.discord.agentComponents.ttlMs` 控制已傳送的 Discord 元件回呼維持註冊的時間（預設 `1800000`，最長 `86400000`）。各帳號設定：`channels.discord.accounts.<id>.agentComponents.ttlMs`。
- 當 components v2 存在時，會忽略 `embeds`。
- 預設會抑制純 URL 預覽。當訊息動作中的單一外連連結應展開時，請設定 `suppressEmbeds: false`。

範例：

```json5
{
  channels: {
    discord: {
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
    },
  },
}
```

## 語音

Discord 有兩種不同的語音介面：即時的**語音頻道**（持續對話）與**語音訊息附件**（波形預覽格式）。閘道同時支援兩者。

### 語音頻道

設定檢查清單：

1. 在 Discord Developer Portal 中啟用 Message Content Intent。
2. 使用角色／使用者允許清單時，啟用 Server Members Intent。
3. 使用 `bot` 與 `applications.commands` 範圍邀請機器人。
4. 在目標語音頻道中授予 Connect、Speak、Send Messages 與 Read Message History。
5. 啟用原生命令（`commands.native` 或 `channels.discord.commands.native`）。
6. 設定 `channels.discord.voice`。

使用 `/vc join|leave|status` 控制工作階段。此命令使用帳號的預設代理程式，並遵循與其他 Discord 命令相同的允許清單和群組政策規則。

```bash
/vc join channel:<voice-channel-id>
/vc status
/vc leave
```

若要在加入前檢查機器人的有效權限：

```bash
openclaw channels capabilities --channel discord --target channel:<voice-channel-id>
```

自動加入範例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        model: "openai/gpt-5.6-sol",
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        allowedChannels: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        connectTimeoutMs: 30000,
        reconnectGraceMs: 15000,
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

注意事項：

- 對於純文字設定，Discord 語音功能為選用；設定 `channels.discord.voice.enabled=true`（或保留現有的 `channels.discord.voice` 區塊）即可啟用 `/vc` 命令、語音執行階段及 `GuildVoiceStates` 閘道意圖。`channels.discord.intents.voiceStates` 可明確覆寫意圖訂閱；若未設定，則會依照實際的語音啟用狀態。
- `voice.mode` 控制對話路徑。預設值為 `agent-proxy`：即時語音前端負責發言輪次時機、中斷及播放，透過 `openclaw_agent_consult` 將實質工作委派給路由至的 OpenClaw 代理，並將結果視為該發言者輸入的 Discord 提示。`stt-tts` 保留較舊的批次 STT 加 TTS 流程。`bidi` 讓即時模型直接對話，同時公開 `openclaw_agent_consult` 以供 OpenClaw 大腦使用。
- `voice.agentSession` 控制哪個 OpenClaw 對話接收語音發言輪次。若未設定，則使用語音頻道自身的工作階段；也可設定 `{ mode: "target", target: "channel:<text-channel-id>" }`，讓語音頻道充當現有 Discord 文字頻道工作階段（例如 `#maintainers`）的麥克風／揚聲器延伸。
- `voice.model` 會覆寫用於 Discord 語音回應及即時諮詢的 OpenClaw 代理大腦。若未設定，則繼承路由至的代理模型。此設定與 `voice.realtime.model` 分開。
- `voice.followUsers` 可讓機器人隨選定使用者加入、移動及離開 Discord 語音頻道。請參閱[在語音頻道中跟隨使用者](#follow-users-in-voice)。
- `agent-proxy` 透過 `discord-voice` 路由語音，為發言者及目標工作階段保留正常的擁有者／工具授權，但會隱藏代理的 `tts` 工具，因為播放由 Discord 語音功能負責。依預設，`agent-proxy` 會為擁有者發言者提供等同完整擁有者的工具存取權（`voice.realtime.toolPolicy: "owner"`），並強烈優先要求在提供實質回答前諮詢 OpenClaw 代理（`voice.realtime.consultPolicy: "always"`）。在此預設的 `always` 模式下，即時層不會在諮詢答案前自動說出填充語；它會擷取並轉錄語音，接著說出路由至的 OpenClaw 答案。如果 Discord 仍在播放第一個答案時，有多個強制諮詢答案完成，後續的精確語音答案會排入佇列，直到播放閒置，而不會在句子中途取代語音。
- 在 `stt-tts` 模式下，STT 使用 `tools.media.audio`；`voice.model` 不會影響轉錄。
- 在即時模式下，`voice.realtime.provider`、`voice.realtime.model` 及 `voice.realtime.speakerVoice` 用於設定即時音訊工作階段。若要搭配使用 OpenAI Realtime 2.1 與 Codex 大腦，請使用 `voice.realtime.model: "gpt-realtime-2.1"` 和 `voice.model: "openai/gpt-5.6-sol"`。
- 依預設，即時語音模式會在即時提供者指示中加入小型的 `IDENTITY.md`、`USER.md` 及 `SOUL.md` 設定檔，讓快速直接的發言輪次維持與路由至的 OpenClaw 代理相同的身分、使用者背景及角色設定。可將 `voice.realtime.bootstrapContextFiles` 設為其中一部分以自訂此行為，或設定 `[]` 以停用。僅支援這些設定檔；`AGENTS.md` 仍保留在一般代理情境中。注入的設定檔情境不會取代 `openclaw_agent_consult` 在工作區作業、目前事實、記憶查詢或工具支援動作中的用途。
- 在 OpenAI `agent-proxy` 即時模式下，依預設，喚醒名稱門控會配合房間情況調整：只有一名人類時，可以不使用喚醒名稱自然交談；有兩名以上人類時，則必須以喚醒名稱開始或結束發言輪次。其他機器人不會計入人數。設定 `voice.realtime.requireWakeName: true` 可一律要求喚醒名稱，設定 `false` 則永不要求。設定的喚醒名稱必須由一或兩個單字組成。如果未設定 `voice.realtime.wakeNames`，OpenClaw 會使用路由至的代理 `name` 加上 `OpenClaw`，並在無法使用時改用代理 ID 加上 `OpenClaw`。啟用中的喚醒名稱門控會停用即時提供者的自動回應、透過 OpenClaw 代理諮詢路徑路由已接受的發言輪次，並在最終轉錄送達前，從部分轉錄辨識出開頭的喚醒名稱時，提供簡短的語音確認。此原則會隨即時加入及離開情況更新，不需重新連線語音。
- OpenAI 即時提供者接受目前的 Realtime 2 事件名稱，以及與 Codex 相容的舊版輸出音訊及轉錄事件別名，因此相容的提供者快照即使有所變動，也不會遺失助理音訊。
- `voice.realtime.bargeIn` 控制 Discord 發言者開始事件是否中斷進行中的即時播放。若未設定，則會依照即時提供者的輸入音訊中斷設定。
- `voice.realtime.minBargeInAudioEndMs` 控制 OpenAI 即時插話截斷音訊前，助理播放的最短持續時間。預設值：`250`。在低回音房間中，可設定 `0` 以立即中斷；若揚聲器配置回音較嚴重，則可提高此值。
- `voice.tts` 僅會覆寫用於 `stt-tts` 語音播放的 `tts`；即時模式則改用 `voice.realtime.speakerVoice`。若要在 Discord 播放中使用 OpenAI 語音，請設定 `voice.tts.provider: "openai"`，並在 `voice.tts.providers.openai.speakerVoice` 下選擇文字轉語音的聲音。在目前的 OpenAI TTS 模型中，`cedar` 是聽起來較陽剛的良好選擇。
- 每個頻道的 Discord `systemPrompt` 覆寫設定會套用至該語音頻道的語音轉錄發言輪次。
- 當 OpenClaw 加入語音頻道時，路由至的代理工作階段會收到包含目前參與者名單的靜默系統事件。之後參與者加入及離開時，該工作階段會隨之更新，但不會觸發未經要求的語音回覆；Discord 顯示名稱會被視為不受信任的標籤。已授權的語音發言輪次也會收到最新的參與者名單快照。
- 語音轉錄發言輪次及 `/vc` 命令會使用 `commands.ownerAllowFrom` 中的 Discord 項目判斷擁有者狀態。若未設定 Discord 命令擁有者，所選 Discord 帳號的 `allowFrom`（或舊版 `dm.allowFrom`）仍可授權語音存取權，但不會授予擁有者狀態。代理工具的可見性會遵循路由至的工作階段所設定的工具原則。
- 如果 `voice.autoJoin` 對同一伺服器有多個項目，OpenClaw 會加入該伺服器最後設定的頻道。
- `voice.allowedChannels` 是選用的駐留允許清單。若未設定，則允許 `/vc join` 進入任何已授權的 Discord 語音頻道。設定後，`/vc join`、啟動時自動加入及機器人的語音狀態移動都會限制於列出的 `{ guildId, channelId }` 項目。將其設為空陣列可拒絕加入所有 Discord 語音頻道。如果 Discord 將機器人移至允許清單以外，OpenClaw 會離開該頻道，並在有可用的已設定自動加入目標時重新加入。
- `voice.daveEncryption` 和 `voice.decryptionFailureTolerance` 會直接傳遞至 `@discordjs/voice` 加入選項；上游預設值為 `daveEncryption=true` 和 `decryptionFailureTolerance=24`。
- OpenClaw 使用內建的 `libopus-wasm` 編解碼器接收 Discord 語音及即時播放原始 PCM。它隨附固定版本的 libopus WebAssembly 組建，不需要原生 opus 附加元件。
- `voice.connectTimeoutMs` 控制 `/vc join` 及自動加入嘗試初始等待 `@discordjs/voice` Ready 的時間。預設值：`30000`。
- `voice.reconnectGraceMs` 控制 OpenClaw 等待已中斷連線的語音工作階段開始重新連線的時間，逾時後即予以銷毀。預設值：`15000`。
- 在 `stt-tts` 模式下，語音播放不會只因另一名使用者開始說話而停止。為避免回授迴路，OpenClaw 會在 TTS 播放期間忽略新的語音擷取；請在播放結束後說話，以開始下一個發言輪次。即時模式會將發言者開始事件轉送為即時提供者的插話訊號。
- 在即時模式下，揚聲器傳入開啟中麥克風的回音可能會被視為插話並中斷播放。對於回音較嚴重的 Discord 房間，請設定 `voice.realtime.providers.openai.interruptResponseOnInputAudio: false`，避免 OpenAI 因輸入音訊而自動中斷。如果仍要讓 Discord 發言者開始事件中斷進行中的播放，請加入 `voice.realtime.bargeIn: true`。OpenAI 即時橋接器會將短於 `voice.realtime.minBargeInAudioEndMs` 的播放截斷視為可能的回音／雜訊並予以忽略，將其記錄為已略過，而不會清除 Discord 播放。
- `voice.captureSilenceGraceMs` 控制 Discord 回報發言者停止後，OpenClaw 在將該音訊片段定稿並送交 STT 前等待的時間。預設值：`2000`；如果 Discord 將正常停頓切割成斷續的部分轉錄，請提高此值。
- 當 ElevenLabs 是選定的 TTS 提供者時，Discord 語音播放會使用串流 TTS，並從提供者回應串流開始播放。不支援串流的提供者會改用合成的暫存檔案路徑。
- OpenClaw 會監看接收解密失敗，若短時間內重複失敗，便會離開並重新加入語音頻道以自動復原。
- 如果更新後的接收記錄反覆顯示 `DecryptionFailed(UnencryptedWhenPassthroughDisabled)`，請收集相依套件報告及記錄。內建的 `@discordjs/voice` 版本包含來自 discord.js PR #11449 的上游填補修正，該 PR 已關閉 discord.js 議題 #11419。
- 當 OpenClaw 將擷取的發言者片段定稿時，出現 `The operation was aborted` 接收事件是預期行為；這些是詳細診斷資訊，不是警告。
- 詳細的 Discord 語音記錄會針對每個已接受的發言者片段，加入長度受限的單行 STT 轉錄預覽，讓偵錯時能同時顯示使用者端及代理回覆端，而不會傾印長度不受限的轉錄文字。
- 在 `agent-proxy` 模式下，強制諮詢備援會略過可能不完整的轉錄片段，例如以 `...` 結尾的文字或尾端為 "and" 之類的連接詞，以及 "be right back" 或 "bye" 等明顯無須採取動作的結語。當此機制避免使用過時的排隊答案時，記錄會顯示 `forced agent consult skipped reason=...`。

### 在語音頻道中跟隨使用者

如果希望 Discord 語音機器人跟隨一或多名已知的 Discord 使用者，而不是在啟動時加入固定頻道或等待 `/vc join`，請使用 `voice.followUsers`。

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        followUsersEnabled: true,
        followUsers: ["discord:123456789012345678"],
        allowedChannels: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
      },
    },
  },
}
```

行為：

- `followUsers` 接受原始 Discord 使用者 ID 和 `discord:<id>` 值。OpenClaw 會先正規化這兩種格式，再比對語音狀態事件。
- 設定 `followUsers` 時，`followUsersEnabled` 預設為 `true`。將其設為 `false`，可保留已儲存的清單，但停止自動跟隨語音。
- `followUsers` 僅控制語音駐留。它不會授予發言者存取權或擁有者權限；請分別設定 `commands.ownerAllowFrom`，以及伺服器或頻道的使用者與角色。
- 當被跟隨的使用者加入允許的語音頻道時，OpenClaw 會加入該頻道。使用者移動時，OpenClaw 會隨之移動。作用中的被跟隨使用者中斷連線時，OpenClaw 會離開。
- 如果同一伺服器中有多位被跟隨的使用者，且作用中的被跟隨使用者離開，OpenClaw 會先移至另一位受追蹤之被跟隨使用者所在的頻道，再離開伺服器。如果多位被跟隨的使用者同時移動，以最後觀察到的語音狀態事件為準。
- `allowedChannels` 仍然適用。位於不允許頻道中的被跟隨使用者會被忽略，而由跟隨功能擁有的工作階段會移至另一位被跟隨的使用者，或直接離開。
- OpenClaw 會在啟動時及有界的時間間隔內，協調遺漏的語音狀態事件。協調作業會取樣已設定的伺服器，並限制每次執行的 REST 查詢次數，因此非常大的 `followUsers` 清單可能需要超過一個間隔才能收斂。
- 如果 Discord 或管理員在機器人跟隨使用者時移動機器人，當目的地受到允許時，OpenClaw 會重建語音工作階段並保留跟隨擁有權。如果機器人被移至 `allowedChannels` 之外，OpenClaw 會離開，並在有設定目標時重新加入該目標。
- DAVE 接收復原可能會在重複解密失敗後離開並重新加入同一頻道。由跟隨功能擁有的工作階段會在此復原路徑中保留其跟隨擁有權，因此被跟隨的使用者稍後中斷連線時，仍會離開該頻道。

選擇加入模式：

- 對於個人或操作員設定，如果機器人應在你使用語音時自動進入語音，請使用 `followUsers`。
- 對於即使沒有受追蹤使用者使用語音，也應保持在線的固定房間機器人，請使用 `autoJoin`。
- 對於一次性加入，或自動出現在語音中會令人意外的房間，請使用 `/vc join`。

Discord 語音轉碼器：

- 語音接收記錄會顯示 `discord voice: opus decoder: libopus-wasm`。
- 即時播放會使用同一個隨附的 `libopus-wasm` 套件，將原始 48 kHz 立體聲 PCM 編碼為 Opus，再將封包交給 `@discordjs/voice`。
- 檔案和供應商串流播放會使用 ffmpeg 轉碼為原始 48 kHz 立體聲 PCM，接著使用 `libopus-wasm` 產生傳送至 Discord 的 Opus 封包串流。

STT 加 TTS 流水線：

- Discord PCM 擷取內容會轉換為 WAV 暫存檔。
- `tools.media.audio` 處理 STT，例如 `openai/gpt-4o-mini-transcribe`。
- 逐字稿會透過 Discord 輸入和路由傳送，同時回應 LLM 會依語音輸出政策執行；該政策會隱藏代理程式的 `tts` 工具並要求傳回文字，因為最終的 TTS 播放由 Discord 語音負責。
- 設定 `voice.model` 時，它只會覆寫此語音頻道回合的回應 LLM。
- `voice.tts` 會合併覆寫 `tts`；支援串流的供應商會直接將內容傳給播放器，否則會在已加入的頻道中播放產生的音訊檔案。

預設代理程式代理語音頻道工作階段範例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        model: "openai/gpt-5.6-sol",
        followUsersEnabled: true,
        followUsers: ["123456789012345678"],
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

沒有 `voice.agentSession` 區塊時，每個語音頻道都會取得各自路由的 OpenClaw 工作階段。例如，`/vc join channel:234567890123456789` 會與該 Discord 語音頻道的工作階段交談。即時模型僅作為語音前端；實質要求會交給已設定的 OpenClaw 代理程式。如果即時模型未呼叫諮詢工具便產生最終逐字稿，OpenClaw 會強制以諮詢作為後援，讓預設行為仍如同與代理程式交談。

舊版 STT 加 TTS 範例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "stt-tts",
        model: "openai/gpt-5.4-mini",
        tts: {
          provider: "openai",
          providers: {
            openai: {
              model: "gpt-4o-mini-tts",
              speakerVoice: "cedar",
            },
          },
        },
      },
    },
  },
}
```

即時雙向範例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "bidi",
        model: "openai/gpt-5.6-sol",
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
          toolPolicy: "safe-read-only",
          consultPolicy: "always",
        },
      },
    },
  },
}
```

將語音作為現有 Discord 頻道工作階段的延伸：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "agent-proxy",
        model: "openai/gpt-5.6-sol",
        agentSession: {
          mode: "target",
          target: "channel:123456789012345678",
        },
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

在 `agent-proxy` 模式中，機器人會加入已設定的語音頻道，但 OpenClaw 代理程式回合會使用目標頻道正常路由的工作階段和代理程式。即時語音工作階段會將傳回的結果說回語音頻道。監督代理程式仍可依其工具政策使用一般訊息工具，包括在適當時另行傳送 Discord 訊息。

當委派的 OpenClaw 執行處於作用中時，新的 Discord 語音逐字稿會先視為即時執行控制，再開始另一個代理程式回合。「狀態」、「取消那個」、「使用較小的修正」或「完成後也檢查測試」等詞句，會分類為作用中工作階段的狀態、取消、引導或後續輸入。狀態、取消、已接受的引導和後續結果都會說回語音頻道，讓呼叫者知道 OpenClaw 是否已處理要求。

實用的目標格式：

- `target: "channel:123456789012345678"` 會透過 Discord 文字頻道工作階段路由。
- `target: "123456789012345678"` 會視為頻道目標。
- `target: "dm:123456789012345678"` 或 `target: "user:123456789012345678"` 會透過該私人訊息工作階段路由。

回音嚴重的 OpenAI Realtime 範例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "bidi",
        model: "openai/gpt-5.6-sol",
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
          bargeIn: true,
          minBargeInAudioEndMs: 500,
          consultPolicy: "always",
          providers: {
            openai: {
              interruptResponseOnInputAudio: false,
            },
          },
        },
      },
    },
  },
}
```

當模型透過開啟的麥克風聽到自己在 Discord 的播放內容，但你仍希望透過說話打斷它時，請使用此設定。OpenClaw 會阻止 OpenAI 因原始輸入音訊而自動中斷，同時 `bargeIn: true` 讓 Discord 發言者開始事件和已處於作用中的發言者音訊，可在下一個擷取回合抵達 OpenAI 前取消作用中的即時回應。當 `audioEndMs` 低於 `minBargeInAudioEndMs` 時，過早的插話訊號會視為可能的回音／雜訊並予以忽略，讓模型不會在第一個播放影格就中止。

預期的語音記錄：

- 加入時：`discord voice: joining ... voiceSession=... supervisorSession=... agentSessionMode=... voiceModel=... realtimeModel=...`
- 即時開始時：`discord voice: realtime bridge starting ... autoRespond=false interruptResponse=false bargeIn=false minBargeInAudioEndMs=...`
- 發言者音訊出現時：`discord voice: realtime speaker turn opened ...`、`discord voice: realtime input audio started ... outputAudioMs=... outputActive=...` 和 `discord voice: realtime speaker turn closed ... chunks=... discordBytes=... realtimeBytes=... interruptedPlayback=...`
- 略過過時語音時：`discord voice: realtime forced agent consult skipped reason=incomplete-transcript ...` 或 `reason=non-actionable-closing ...`
- 即時回應完成時：`discord voice: realtime audio playback finishing reason=response.done ... audioMs=... chunks=...`
- 播放停止／重設時：`discord voice: realtime audio playback stopped reason=... audioMs=... elapsedMs=... chunks=...`
- 即時諮詢時：`discord voice: realtime consult requested ... voiceSession=... supervisorSession=... question=...`
- 代理程式回答時：`discord voice: agent turn answer ...`
- 精確語音排入佇列時：`discord voice: realtime exact speech queued ... queued=... outputAudioMs=... outputActive=...`，後接 `discord voice: realtime exact speech dequeued reason=player-idle ...`
- 偵測到插話時：`discord voice: realtime barge-in detected source=speaker-start ...` 或 `discord voice: realtime barge-in detected source=active-speaker-audio ...`，後接 `discord voice: realtime barge-in requested reason=... outputAudioMs=... outputActive=...`
- 即時中斷時：`discord voice: realtime model interrupt requested client:response.cancel reason=barge-in`，後接 `discord voice: realtime model audio truncated client:conversation.item.truncate reason=barge-in audioEndMs=...` 或 `discord voice: realtime model interrupt confirmed server:response.done status=cancelled ...`
- 忽略回音／雜訊時：`discord voice: realtime model interrupt ignored client:conversation.item.truncate.skipped reason=barge-in audioEndMs=0 minAudioEndMs=250`
- 停用插話時：`discord voice: realtime capture ignored during playback (barge-in disabled) ...`
- 播放閒置時：`discord voice: realtime barge-in ignored reason=... outputActive=false ... playbackChunks=0`

若要偵錯音訊遭截斷的問題，請依時間軸閱讀即時語音記錄：

1. `realtime audio playback started` 表示 Discord 已開始播放助理音訊。橋接器會從此時開始計算助理輸出區塊、Discord PCM 位元組、供應商即時位元組，以及合成音訊持續時間。
2. `realtime speaker turn opened` 標示 Discord 發言者開始活動。如果播放已處於作用中且已啟用 `bargeIn`，後面可能會接著出現 `barge-in detected source=speaker-start`。
3. `realtime input audio started` 標示該發言者回合收到的第一個實際音訊影格。此處出現 `outputActive=true` 或非零的 `outputAudioMs`，表示助理播放仍處於作用中時，麥克風正在傳送輸入。
4. `barge-in detected source=active-speaker-audio` 表示 OpenClaw 在助理播放處於作用中時，偵測到即時發言者音訊。這有助於區分真正的中斷，與沒有實用音訊的 Discord 發言者開始事件。
5. `barge-in requested reason=...` 表示 OpenClaw 已要求即時供應商取消或截斷作用中的回應。它包含 `outputAudioMs`、`outputActive` 和 `playbackChunks`，讓你能看出中斷前實際已播放多少助理音訊。
6. `realtime audio playback stopped reason=...` 是本機 Discord 播放的重設點。原因會指出由誰停止播放：`barge-in`、`player-idle`、`provider-clear-audio`、`forced-agent-consult`、`stream-close` 或 `session-close`。
7. `realtime speaker turn closed` 會摘要擷取到的輸入回合。`chunks=0` 或 `hasAudio=false` 表示發言者回合已開始，但沒有可用音訊抵達即時橋接器。`interruptedPlayback=true` 表示該輸入回合與助理輸出重疊，並觸發插話邏輯。

實用欄位：

- `outputAudioMs`：即時供應商在該記錄行之前產生的助理音訊持續時間。
- `audioMs`：播放停止前，OpenClaw 計算的助理音訊持續時間。
- `elapsedMs`：開啟與關閉播放串流或發言者回合之間的實際經過時間。
- `discordBytes`：傳送至 Discord 語音或從中接收的 48 kHz 立體聲 PCM 位元組。
- `realtimeBytes`：傳送至即時供應商或從中接收的供應商格式 PCM 位元組。
- `playbackChunks`：針對作用中回應轉送至 Discord 的助理音訊區塊。
- `sinceLastAudioMs`：最後一個擷取到的發言者音訊影格與發言者回合關閉之間的間隔。

常見模式：

- 立即中斷，且伴隨 `source=active-speaker-audio`、較小的 `outputAudioMs`，以及同一位使用者在相近時間出現，通常表示喇叭回音進入麥克風。提高 `voice.realtime.minBargeInAudioEndMs`、降低喇叭音量、使用耳機，或設定 `voice.realtime.providers.openai.interruptResponseOnInputAudio: false`。
- `source=speaker-start` 後接 `speaker turn closed ... hasAudio=false`，表示 Discord 回報喇叭已開始播放，但沒有音訊送達 OpenClaw。這可能是暫時性的 Discord 語音事件、噪音閘門行為，或用戶端短暫啟用麥克風。
- 若 `audio playback stopped reason=stream-close` 附近沒有插話或 `provider-clear-audio`，表示本機 Discord 播放串流意外結束。請檢查先前的提供者與 Discord 播放器日誌。
- `capture ignored during playback (barge-in disabled)` 表示 OpenClaw 在助理音訊播放期間刻意捨棄輸入。若要讓語音中斷播放，請啟用 `voice.realtime.bargeIn`。
- `barge-in ignored ... outputActive=false` 表示 Discord 或提供者的 VAD 回報有語音，但 OpenClaw 沒有可中斷的進行中播放。這不應中斷音訊。

認證資訊會依元件分別解析：`voice.model` 的 LLM 路由驗證、`tools.media.audio` 的 STT 驗證、`tts`/`voice.tts` 的 TTS 驗證，以及 `voice.realtime.providers` 或提供者一般驗證設定的即時提供者驗證。

### 語音訊息

Discord 語音訊息會顯示波形預覽，且需要 OGG/Opus 音訊。OpenClaw 會自動產生波形，但需要閘道主機上的 `ffmpeg` 和 `ffprobe` 來檢查與轉換。

- 提供**本機檔案路徑**（不接受 URL）。
- 省略文字內容（Discord 不接受在同一個承載資料中同時包含文字與語音訊息）。
- 接受任何音訊格式；OpenClaw 會視需要轉換為 OGG/Opus。

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## 疑難排解

<AccordionGroup>
  <Accordion title="使用了不允許的 intents，或機器人看不到伺服器訊息">

    - 啟用 Message Content Intent
    - 當你依賴使用者／成員解析時，啟用 Server Members Intent
    - 變更 intents 後重新啟動閘道

  </Accordion>

  <Accordion title="伺服器訊息意外遭封鎖">

    - 確認 `groupPolicy`
    - 確認 `channels.discord.guilds` 下的伺服器允許清單
    - 若存在伺服器 `channels` 對應表，則只允許其中列出的頻道
    - 確認 `requireMention` 行為與提及模式

    實用的檢查方式：

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="不要求提及，但仍遭封鎖">
    常見原因：

    - `groupPolicy="allowlist"` 沒有相符的伺服器／頻道允許清單
    - `requireMention` 設定在錯誤的位置（必須位於 `channels.discord.guilds` 或頻道項目下）
    - 傳送者遭伺服器／頻道的 `users` 允許清單封鎖

  </Accordion>

  <Accordion title="長時間執行的 Discord 回合或重複回覆">

    典型日誌：

    - `Slow listener detected ...`
    - `stuck session: sessionKey=agent:...:discord:... state=processing ...`

    Discord 不會對已排入佇列的代理程式回合套用頻道所擁有的逾時。訊息監聽器會立即移交工作，而排入佇列的 Discord 執行會維持各工作階段的順序，直到工作階段／工具／執行階段生命週期完成或中止工作。

  </Accordion>

  <Accordion title="閘道中繼資料查詢逾時警告">
    OpenClaw 會在連線前擷取 Discord `/gateway/bot` 中繼資料。暫時性失敗會回退至 Discord 的預設閘道 URL，且日誌輸出會受到速率限制。

    中繼資料逾時預設為 30 秒。`OPENCLAW_DISCORD_GATEWAY_INFO_TIMEOUT_MS` 可針對特殊主機環境覆寫此設定。

  </Accordion>

  <Accordion title="閘道 READY 逾時重新啟動">
    OpenClaw 會在啟動期間以及執行階段重新連線後，等待 Discord 閘道的 `READY` 事件。採用交錯啟動的多帳號設定，可能需要比預設值更長的啟動 READY 等待時間。

    啟動時等待 15 秒，執行階段重新連線時等待 30 秒。`OPENCLAW_DISCORD_READY_TIMEOUT_MS` 和 `OPENCLAW_DISCORD_RUNTIME_READY_TIMEOUT_MS` 仍可供特殊主機環境使用。

  </Accordion>

  <Accordion title="權限稽核不符">
    `channels status --probe` 權限檢查僅適用於數字頻道 ID。

    若使用 slug 鍵，執行階段比對仍可運作，但探查無法完整驗證權限。

  </Accordion>

  <Accordion title="私人訊息與配對問題">

    - 私人訊息已停用：`channels.discord.dm.enabled=false`
    - 私人訊息原則已停用：`channels.discord.dmPolicy="disabled"`（舊版：`channels.discord.dm.policy`）
    - 在 `pairing` 模式下等待配對核准

  </Accordion>

  <Accordion title="機器人對機器人迴圈">
    預設會忽略由機器人撰寫的訊息。

    若設定 `channels.discord.allowBots=true`，請使用嚴格的提及與允許清單規則，以避免迴圈行為。
    建議使用 `channels.discord.allowBots="mentions"`，只接受提及該機器人的機器人訊息。

    OpenClaw 也隨附共用的[機器人迴圈防護](/zh-TW/channels/bot-loop-protection)。每當 `allowBots` 允許機器人撰寫的訊息進入分派流程時，Discord 會將傳入事件對應至 `(account, channel, bot pair)` 事實，而通用配對防護會在該配對超過設定的事件預算後加以抑制。此防護可避免過去必須依靠 Discord 速率限制才能停止的失控雙機器人迴圈；它不會影響單一機器人部署，也不會影響保持在預算內的單次機器人回覆。

    預設設定（設定 `allowBots` 時生效）：

    - `maxEventsPerWindow: 20` -- 機器人配對可在滑動時間範圍內交換 20 則訊息
    - `windowSeconds: 60` -- 滑動時間範圍長度
    - `cooldownSeconds: 60` -- 一旦超過預算，任一方向後續的所有機器人對機器人訊息都會遭捨棄一分鐘

    請在 `channels.defaults.botLoopProtection` 下設定一次共用預設值，然後在合法工作流程需要更多餘裕時覆寫 Discord 設定。優先順序如下：

    - `channels.discord.accounts.<account>.botLoopProtection`
    - `channels.discord.botLoopProtection`
    - `channels.defaults.botLoopProtection`
    - 內建預設值

    Discord 使用通用的 `maxEventsPerWindow`、`windowSeconds` 和 `cooldownSeconds` 鍵。

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
        windowSeconds: 60,
        cooldownSeconds: 60,
      },
    },
    discord: {
      // 選用的 Discord 全域覆寫。帳號區塊會覆寫個別
      // 欄位，並從此處繼承省略的欄位。
      botLoopProtection: {
        maxEventsPerWindow: 4,
      },
      accounts: {
        alpha: {
          // Alpha 僅在其他機器人提及它時才會接收其訊息。
          allowBots: "mentions",
        },
        bravo: {
          // Bravo 會接收所有由機器人撰寫的 Discord 訊息。
          allowBots: true,
          mentionAliases: {
            // 讓 Bravo 使用設定的使用者 ID 寫入對 Alpha 的 Discord 提及。
            Alpha: "ALPHA_DISCORD_USER_ID",
          },
          botLoopProtection: {
            // 在抑制該配對前，每分鐘最多允許五則訊息。
            maxEventsPerWindow: 5,
            windowSeconds: 60,
            cooldownSeconds: 90,
          },
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="語音 STT 因 DecryptionFailed(...) 而中斷">

    - 讓 OpenClaw 保持最新（`openclaw update`），以確保具備 Discord 語音接收復原邏輯
    - 確認 `channels.discord.voice.daveEncryption=true`（預設值）
    - 從 `channels.discord.voice.decryptionFailureTolerance=24`（上游預設值）開始，僅在需要時調整
    - 監看日誌中的：
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - 若自動重新加入後仍持續失敗，請收集日誌，並與 [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419) 和 [discord.js #11449](https://github.com/discordjs/discord.js/pull/11449) 中的上游 DAVE 接收歷程比較

  </Accordion>
</AccordionGroup>

## 設定參考

主要參考：[設定參考 - Discord](/zh-TW/gateway/config-channels#discord)。

<Accordion title="重要的 Discord 欄位">

- 啟動／驗證：`enabled`、`token`、`applicationId`、`accounts.*`、`allowBots`
- 原則：`groupPolicy`、`dmPolicy`、`allowFrom`、`dm.*`、`guilds.*`、`guilds.*.channels.*`
- 命令：`commands.native`、`commands.useAccessGroups`（全域）、`configWrites`、`slashCommand.ephemeral`
- 閘道：`proxy`
- 回覆／歷程：`replyToMode`、`historyLimit`、`dmHistoryLimit`、`dms.*.historyLimit`
- 傳遞：`textChunkLimit`（預設 `2000`）、`maxLinesPerMessage`（預設 `17`）
- 串流：`streaming.mode`、`streaming.chunkMode`、`streaming.preview.*`、`streaming.progress.*`、`streaming.block.*`（舊版扁平 `streamMode`、`draftChunk`、`blockStreaming`、`blockStreamingCoalesce`、`chunkMode` 鍵會由 `openclaw doctor --fix` 遷移至 `streaming.*`）
- 媒體：`mediaMaxMb`（限制傳出的 Discord 上傳，預設 `100`）
- 動作：`actions.*`
- 狀態：`activity`、`status`、`activityType`、`activityUrl`、`autoPresence.*`
- 使用者介面：`ui.components.accentColor`
- 功能：`threadBindings`、頂層 `bindings[]`（`type: "acp"`）、`pluralkit`、`execApprovals`、`intents`、`agentComponents.enabled`、`agentComponents.ttlMs`、`activities`、`heartbeat`、`responsePrefix`

</Accordion>

### Discord Activities

設定 `channels.discord.activities`，讓代理程式可發布在 Discord 內開啟的獨立 HTML 小工具。此區塊為選用；若省略，OpenClaw 不會註冊任何 Activity 路由、工具或互動處理常式。如需 Developer Portal、通道、安全性與疑難排解設定，請參閱 [Discord Activities](/zh-TW/channels/discord-activities)。

- `activities.clientSecret`：Discord 應用程式的 OAuth2 用戶端密鑰；回退至 `DISCORD_CLIENT_SECRET`
- `activities.applicationId`：選用的 Activity 應用程式 ID；預設為閘道啟動時取得的機器人應用程式 ID

## 安全性與操作

- 將機器人權杖視為機密資訊（在受監督的環境中建議使用 `DISCORD_BOT_TOKEN`）。
- 授予最低權限的 Discord 權限。
- 若命令部署／狀態已過時，請重新啟動閘道，並使用 `openclaw channels status --probe` 重新檢查。

## 相關內容

<CardGroup cols={2}>
  <Card title="Discord Activities" icon="window" href="/zh-TW/channels/discord-activities">
    在 Discord 內啟動互動式 HTML 小工具。
  </Card>
  <Card title="配對" icon="link" href="/zh-TW/channels/pairing">
    將 Discord 使用者與閘道配對。
  </Card>
  <Card title="群組" icon="users" href="/zh-TW/channels/groups">
    群組聊天與允許清單行為。
  </Card>
  <Card title="頻道路由" icon="route" href="/zh-TW/channels/channel-routing">
    將傳入訊息路由至代理程式。
  </Card>
  <Card title="安全性" icon="shield" href="/zh-TW/gateway/security">
    威脅模型與強化措施。
  </Card>
  <Card title="多代理程式路由" icon="sitemap" href="/zh-TW/concepts/multi-agent">
    將伺服器與頻道對應至代理程式。
  </Card>
  <Card title="斜線命令" icon="terminal" href="/zh-TW/tools/slash-commands">
    原生命令行為。
  </Card>
</CardGroup>
