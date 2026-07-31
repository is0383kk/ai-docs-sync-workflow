---
read_when:
    - 你想要新增或移除頻道帳號（Discord、Google Chat、iMessage、Matrix、Signal、Slack、Telegram、WhatsApp 等）
    - 你想要檢查頻道狀態或追蹤頻道日誌
    - 你需要檢查或重新提交失敗的傳入頻道事件
summary: '`openclaw channels` 的命令列介面參考（帳號、狀態、無法投遞的訊息、功能、解析、日誌、登入／登出）'
title: 頻道
x-i18n:
    generated_at: "2026-07-26T08:27:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5b7d674264af51d6fec34c8c95256129d66918b7c4515ac0f2c2bd311f2c3b
    source_path: cli/channels.md
    workflow: 16
---

# `openclaw channels`

在閘道上管理聊天頻道帳號及其執行階段狀態。

相關文件：

- 頻道指南：[頻道](/zh-TW/channels)
- 閘道設定：[設定](/zh-TW/gateway/configuration)

## 常用命令

```bash
openclaw channels list
openclaw channels list --all
openclaw channels status
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels logs --channel all
openclaw channels dead-letters list --channel telegram --account default
```

`channels list` 僅顯示聊天頻道：預設顯示已設定的帳號，並為每個帳號加上 `installed`、`configured` 和 `enabled` 狀態標籤（機器可讀輸出使用 `--json`）。傳入 `--all`，也會顯示尚未設定帳號的內建頻道，以及尚未安裝至磁碟的可安裝目錄頻道。提供者驗證與模型用量位於其他功能中：提供者驗證設定檔使用 `openclaw models auth list`，用量／配額使用 `openclaw status` 或 `openclaw models list`。

## 狀態／功能／解析／日誌

- `channels status`：`--channel <name>`、`--probe`、`--timeout <ms>`（預設為 `10000`）、`--json`
- `channels capabilities`：`--channel <name>`、`--account <id>`（需要 `--channel`）、`--target <dest>`（需要 `--channel`）、`--timeout <ms>`（預設為 `10000`，上限為 `30000`）、`--json`
- `channels resolve <entries...>`：`--channel <name>`、`--account <id>`、`--kind <auto|user|group>`（預設為 `auto`）、`--json`
- `channels logs`：`--channel <name|all>`（預設為 `all`）、`--lines <n>`（預設為 `200`）、`--json`

`channels status --probe` 是即時路徑：在可連線的閘道上，它會針對每個帳號執行
`probeAccount` 和選用的 `auditAccount` 檢查，因此輸出可能包含傳輸
狀態，以及 `works`、`probe failed`、`audit ok` 或 `audit failed` 等探測結果。
如果無法連線至閘道，`channels status` 會改為提供僅依據設定的摘要，
而非即時探測輸出。

## 傳入無法投遞事件

耗盡重試原則的傳入事件，會在佇列現有的失敗項目保留期間內留在共用狀態資料庫中。使用下列命令檢查某個頻道帳號：

```bash
openclaw channels dead-letters list --channel telegram --account default
openclaw channels dead-letters list --channel telegram --account default --json
```

文字檢視會顯示事件 ID、失敗原因、嘗試次數及失敗經過時間。JSON 輸出還會包含保留的承載資料、中繼資料、處理通道及各次嘗試的時間戳記，以供診斷。

修正根本問題後，使用事件原始 ID 將單一事件重新加入佇列：

```bash
openclaw channels dead-letters resubmit <event-id> --channel telegram --account default
```

請在閘道主機上執行這些命令，使其存取與頻道執行階段相同的共用狀態資料庫。重新提交會保留承載資料、中繼資料及處理通道，但會重設嘗試計數器與佇列等待時間。此操作會以不可分割方式取代該事件的失敗標記，因此若事件仍在等待處理或已被領取，再次執行命令將遭拒絕，而不會建立第二次派送。執行中的頻道會在下一次排空傳入佇列時取用它。已完成的事件會維持終止狀態，無法重新提交。在新增承載資料保留功能之前建立的失敗資料列仍可能出現在清單中，但由於其承載資料無法取得，重新提交會遭拒絕。

`openclaw health` 會回報每個頻道帳號的無法投遞事件數量，以及最早失敗事件的經過時間。`openclaw doctor` 會列出受影響的帳號，並指向上述檢查命令。

請勿將 `openclaw sessions`、閘道 `sessions.list` 或代理程式
`sessions_list` 工具用作頻道通訊端健康狀態的訊號。這些介面回報的是
已儲存的對話資料列，而非提供者執行階段狀態。Discord 提供者
重新啟動後，已連線但無活動的帳號可能仍正常運作，而 Discord 工作階段
資料列要到下一個傳入或傳出對話事件發生後才會出現。

## 新增／移除帳號

```bash
openclaw channels add --channel telegram --token <bot-token>
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels remove --channel telegram --delete
```

<Tip>
`openclaw channels add telegram --help` 或 `openclaw channels add --channel telegram --help` 僅顯示 Telegram 的設定旗標。`openclaw channels add --help` 僅顯示共用命令封套。
</Tip>

`channels remove` 僅適用於已安裝／已設定的頻道外掛。對於可安裝的目錄頻道，請先使用 `channels add`。若未使用 `--delete`，系統會詢問是否停用帳號並保留其設定；`--delete` 則會移除設定項目而不提示。
對於由執行階段支援的頻道外掛，`channels remove` 也會在更新設定前要求執行中的閘道停止選取的帳號，因此停用或刪除帳號後，舊的監聽器不會持續運作至重新啟動為止。

共用控制封套僅包含 `--channel`、`--account`，以及選用的帳號顯示 `--name`。每個現代頻道外掛都自行定義其認證資訊、傳輸方式及提供者專屬語意。透過位置 ID 或 `--channel <id>` 選取頻道後，命令列介面只會根據內建或已安裝外掛套件的中繼資料建立該頻道的選項，而不載入頻道執行階段程式碼。

當現代合約處理 `--token`、`--url` 或 `--use-env` 等看似通用的旗標時，這些旗標仍由頻道擁有。當選取的第三方外掛仍使用舊版共用設定轉接器時，核心只會為該頻道註冊已發布的相容旗標集，以及其舊版 `cliAddOptions`。不相關的舊版欄位不會洩漏至其他頻道，而選取的現代頻道會拒絕其未宣告的相容旗標。

頻道自有旗標的範例如下：

| 頻道        | 旗標                                                                                                 |
| ----------- | ---------------------------------------------------------------------------------------------------- |
| Google Chat | `--webhook-path`、`--webhook-url`、`--audience-type`、`--audience`                                   |
| iMessage    | `--cli-path`、`--db-path`、`--service`、`--region`                                                   |
| Matrix      | `--homeserver`、`--user-id`、`--access-token`、`--password`、`--device-name`、`--initial-sync-limit` |
| Nostr       | `--private-key`、`--relay-urls`                                                                      |
| Signal      | `--signal-number`、`--signal-transport`、`--cli-path`、`--http-url`、`--http-host`、`--http-port`    |
| Tlon        | `--ship`、`--url`、`--code`、`--group-channels`、`--dm-allowlist`、`--auto-discover-channels`        |
| WhatsApp    | `--auth-dir`                                                                                         |

如果在使用旗標新增頻道的命令期間需要安裝頻道外掛，OpenClaw 會使用該頻道的預設安裝來源，而不開啟互動式外掛安裝提示。

引導式設定與旗標驅動設定都會經過所選頻道的剖析器、驗證、帳號解析、設定寫入器及寫入後掛鉤。不支援的旗標會以所屬頻道的設定錯誤失敗，而不會透過全域輸入集合予以接受。

當執行 `openclaw channels add` 且未提供直接帳號、認證資訊或頻道設定旗標時，互動式精靈可以顯示提示。位置頻道 ID 和 `--channel <id>` 都會預先選取該頻道，而不略過引導：

```bash
openclaw channels add telegram
openclaw channels add --channel telegram
```

精靈可提示輸入：

- 每個所選頻道的帳號 ID
- 這些帳號的選用顯示名稱
- `Route these channel accounts to agents now?`

如果確認立即繫結，精靈會詢問哪個代理程式應擁有各個已設定的頻道帳號，並寫入帳號範圍的路由繫結。

之後也可以使用 `openclaw agents bindings`、`openclaw agents bind` 和 `openclaw agents unbind` 管理相同的路由規則（請參閱 [代理程式](/zh-TW/cli/agents)）。

當你為仍使用單一帳號頂層設定的頻道新增非預設帳號時，OpenClaw 會先將這些頂層值提升至該頻道的帳號對應表，再寫入新帳號。若頻道恰好只有一個具名帳號，或 `defaultAccount` 指向某個帳號，提升作業會重用該現有具名帳號；否則這些值會寫入 `channels.<channel>.accounts.default`。

路由行為維持一致：

- 現有的僅限頻道繫結（沒有 `accountId`）會繼續比對預設帳號。
- `channels add` 不會在非互動模式中自動建立或重寫繫結。
- 互動式設定可選擇新增帳號範圍的繫結。

如果設定已處於混合狀態（同時存在具名帳號，且仍設定了頂層單一帳號值），請執行 `openclaw doctor --fix`，將帳號範圍的值移至為該頻道選定的提升帳號中。

## 登入與登出（互動式）

```bash
openclaw channels login --channel whatsapp
openclaw channels logout --channel whatsapp
```

- `channels login` 支援 `--account <id>` 和 `--verbose`；`channels logout` 支援 `--account <id>`。
- 當只有一個已設定的頻道支援該動作時，`channels login` 和 `logout` 可以推斷頻道；若有多個，請傳入 `--channel`。
- `channels logout` 會在閘道可連線時優先使用即時閘道路徑，因此登出會先停止任何作用中的監聽器，再清除頻道驗證狀態。如果本機閘道無法連線，則會改用本機驗證清理；使用 `gateway.mode: "remote"` 時，閘道錯誤會改為使命令失敗。
- 成功登入後，命令列介面會要求可連線的本機閘道啟動該帳號；在遠端模式中，它會將驗證資訊儲存於本機，並註明遠端執行階段未重新啟動。
- 請在閘道主機的終端機中執行 `channels login`。代理程式 `exec` 會阻擋此互動式登入流程；從聊天中操作時，應在可用情況下使用頻道原生的代理程式登入工具，例如 `whatsapp_login`。

## 疑難排解

- 執行 `openclaw status --deep` 以進行廣泛探測。
- 使用 `openclaw doctor` 進行引導式修正。
- 當閘道無法連線時，`openclaw channels status` 會改為提供僅依據設定的摘要。如果支援的頻道認證資訊是透過 SecretRef 設定，但在目前的命令路徑中無法取得，它會將該帳號回報為已設定並附上降級註記，而不會顯示為未設定。

## 功能探測

擷取提供者功能提示（如有，可包含 intents／scopes）以及靜態功能支援：

```bash
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
```

注意事項：

- `--channel` 是選用項目；省略即可列出所有頻道（包括外掛提供的頻道）。
- `--account` 僅能與 `--channel` 搭配使用。
- `--target` 接受 `channel:<id>` 或原始數字頻道 ID，且僅適用於 Discord。對於 Discord 語音頻道，權限檢查會標示缺少的 `ViewChannel`、`Connect`、`Speak`、`SendMessages` 和 `ReadMessageHistory`。
- 探測依提供者而異：Discord 機器人身分與 intents，以及選用的頻道權限；Slack 機器人與使用者 scopes；Telegram 機器人旗標與網路鉤子；Signal daemon 版本；Microsoft Teams 應用程式權杖與 Graph 角色／scopes（已知時會加上註記）。沒有探測功能的頻道會回報 `Probe: unavailable`。

## 將名稱解析為 ID

使用提供者目錄將頻道／使用者名稱解析為 ID：

```bash
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels resolve --channel discord "My Server/#support" "@someone"
openclaw channels resolve --channel matrix "Project Room"
```

注意事項：

- 使用 `--kind user|group|auto` 強制指定目標類型。
- 當多個項目共用相同名稱時，解析會優先選擇作用中的相符項目。
- `channels resolve` 為唯讀。如果所選帳號透過 SecretRef 設定，但目前的命令路徑無法取得該認證資訊，命令會傳回附有註記的降級未解析結果，而不是中止整次執行。
- `channels resolve` 不會安裝頻道外掛。若目錄中的頻道可供安裝，請先使用 `channels add --channel <name>`，再解析名稱。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [頻道概覽](/zh-TW/channels)
