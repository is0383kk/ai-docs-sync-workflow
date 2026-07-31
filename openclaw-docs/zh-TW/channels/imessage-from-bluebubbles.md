---
read_when:
    - 規劃從 BlueBubbles 移轉至內建的 iMessage 外掛
    - 將 BlueBubbles 設定鍵轉換為對應的 iMessage 設定鍵
    - 啟用 iMessage 外掛前驗證 imsg
summary: 將舊版 BlueBubbles 設定轉換至內建的 iMessage 外掛：金鑰對應、群組允許清單閘門，以及切換驗證。
title: 從 BlueBubbles 遷移而來
x-i18n:
    generated_at: "2026-07-26T07:42:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5984ad1319b4bb3060496666bea6de663eba0105a89f82d13030c015c5df159d
    source_path: channels/imessage-from-bluebubbles.md
    workflow: 16
---

BlueBubbles 支援已移除。OpenClaw 僅透過內建的 `imessage` 外掛支援 iMessage；此外掛會透過 JSON-RPC 驅動 [`steipete/imsg`](https://github.com/steipete/imsg)，並使用與 BlueBubbles 相同的私有 API 介面（`react`、`edit`、`unsend`、`reply`、`sendWithEffect`、原生投票、群組管理、附件）。單一命令列介面執行檔取代了 BlueBubbles 伺服器、用戶端應用程式與網路鉤子串接：不再有 REST 端點，也不再有網路鉤子驗證。

本指南將舊的 `channels.bluebubbles` 設定遷移至 `channels.imessage`。沒有其他受支援的遷移路徑。在目前的 OpenClaw 中，殘留的 `channels.bluebubbles` 區塊不會生效，沒有任何執行階段會讀取它。

<Note>
如需簡短公告與操作人員摘要，請參閱 [BlueBubbles 移除與 imsg iMessage 路徑](/zh-TW/announcements/bluebubbles-imessage)。
</Note>

## 遷移檢查清單

若你已熟悉舊的 BlueBubbles 設定，以下是最短且安全的路徑：

1. 直接在執行 Messages.app 的 Mac 上驗證 `imsg`（`imsg chats`、`imsg history`、`imsg send`、`imsg rpc --help`）。
2. 將行為設定鍵從 `channels.bluebubbles` 複製至 `channels.imessage`：`dmPolicy`、`allowFrom`、`groupPolicy`、`groupAllowFrom`、`groups`、`includeAttachments`、`attachmentRoots`、`mediaMaxMb`、`textChunkLimit` 和 `actions`。
3. 移除已不存在的傳輸設定鍵：`serverUrl`、`password`、網路鉤子 URL，以及 BlueBubbles 伺服器設定。
4. 如果閘道並非在 Messages Mac 上執行，請將 `channels.imessage.cliPath` 設為 SSH 包裝程式，並設定 `remoteHost` 以供遠端擷取附件。
5. 啟用 `channels.imessage`、重新啟動閘道，然後執行 `openclaw channels status --probe --channel imessage`。
6. 測試一則私訊、一個允許的群組、附件（若已啟用），以及你預期代理程式使用的每個私有 API 動作。
7. 驗證 iMessage 路徑後，刪除 BlueBubbles 伺服器與舊的 `channels.bluebubbles` 設定。

## imsg 的功能

`imsg` 是 Messages 的本機 macOS 命令列介面。OpenClaw 會以子程序啟動 `imsg rpc`，並透過 stdin/stdout 使用 JSON-RPC 通訊。不需要公開任何 HTTP 伺服器、網路鉤子 URL、背景常駐程式、啟動代理程式或連接埠。

- 透過唯讀 SQLite 控制代碼從 `~/Library/Messages/chat.db` 讀取資料。
- 即時傳入訊息來自 `imsg watch` / `watch.subscribe`；它會追蹤 `chat.db` 檔案系統事件，並以輪詢作為備援。
- 一般文字與檔案傳送會使用 Messages.app 自動化。
- 進階動作會使用 `imsg launch`，將 `imsg` 輔助程式注入 Messages.app。這會解鎖已讀回條、輸入指示器、豐富內容傳送、編輯、收回、討論串回覆、點按回應、投票與群組管理。
- Linux 組建可以檢查複製的 `chat.db`，但無法傳送、監看即時 Mac 資料庫，或驅動 Messages.app。若要使用 OpenClaw iMessage，請在已登入的 Mac 上執行 `imsg`，或透過連至該 Mac 的 SSH 包裝程式執行。

## 開始之前

1. 在執行 Messages.app 的 Mac 上安裝 `imsg`：

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg chats --limit 3
   ```

   在一般本機設定中，OpenClaw 設定流程可在已登入 Messages 的 Mac 上，經使用者確認後安裝或更新 Homebrew 的 `imsg`。手動設定與 SSH 包裝程式拓撲仍由操作人員管理：請在將執行 `imsg` 的相同本機或遠端使用者情境中，重複執行 Homebrew 更新。如果 `imsg chats` 因 `unable to open database file` 而失敗、輸出為空，或出現 `authorization denied`，請將「完整磁碟存取權」授予啟動 `imsg` 的終端機、編輯器、Node 程序、閘道服務或 SSH 父程序，然後重新開啟該父程序。

2. 在變更 OpenClaw 設定之前，驗證讀取、監看、傳送與 RPC 介面：

   ```bash
   imsg chats --limit 10 --json | jq -s
   imsg history --chat-id 42 --limit 10 --attachments --json | jq -s
   imsg watch --chat-id 42 --reactions --json
   imsg send --chat-id 42 --text "OpenClaw imsg test"
   imsg rpc --help
   ```

   將 `42` 替換為 `imsg chats` 中的真實聊天 ID。傳送需要 Messages.app 的 Automation 權限。如果 OpenClaw 將透過 SSH 執行，請透過 OpenClaw 將使用的相同 SSH 包裝程式或使用者情境執行這些命令。如果讀取正常，但傳送因 AppleEvents `-1743` 而失敗，請檢查 Automation 權限是否已套用至 `/usr/libexec/sshd-keygen-wrapper`；請參閱 [SSH 包裝程式傳送因 AppleEvents -1743 而失敗](/zh-TW/channels/imessage#requirements-and-permissions-macos)。

3. 啟用私有 API 橋接器。強烈建議為 OpenClaw iMessage 啟用此功能，因為回覆、點按回應、效果、投票、附件回覆與群組動作皆依賴此功能：

   ```bash
   imsg launch
   imsg status --json
   ```

   `imsg launch` 需要停用 SIP（在新版 macOS 上，還需要放寬程式庫驗證限制——請參閱[啟用 imsg 私有 API](/zh-TW/channels/imessage#enabling-the-imsg-private-api)）。基本傳送、歷史記錄與監看功能可在沒有 `imsg launch` 的情況下運作；完整的 OpenClaw iMessage 動作介面則無法運作。

4. 啟用 `channels.imessage` 並啟動閘道後，透過 OpenClaw 驗證橋接器：

   ```bash
   openclaw channels status --probe
   ```

   iMessage 帳號應回報 `works`；使用 `--json` 時，探測承載資料會包含 `privateApi.available: true`。如果回報 `false`，請先修正該問題——請參閱[功能偵測](/zh-TW/channels/imessage#private-api-actions)。探測需要可連線的閘道（否則命令列介面會退回僅輸出設定），而且只會探測已設定並啟用的帳號。

5. 建立設定快照：

   ```bash
   cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
   ```

## 設定轉換

iMessage 與 BlueBubbles 共用大多數頻道層級的行為設定鍵。改變的是傳輸方式（REST 伺服器與本機命令列介面之別）以及群組登錄設定鍵格式。

| BlueBubbles                                                | 內建 iMessage                              | 備註                                                                                                                                                                                                                                                                            |
| ---------------------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `channels.bluebubbles.enabled`                             | `channels.imessage.enabled`               | 語意相同（區塊存在後，預設為 `true`）。                                                                                                                                                                                                                           |
| `channels.bluebubbles.serverUrl`                           | _（已移除）_                               | 沒有 REST 伺服器——此外掛會透過標準輸入輸出啟動 `imsg rpc`。                                                                                                                                                                                                                        |
| `channels.bluebubbles.password`                            | _（已移除）_                               | 不需要網路鉤子驗證。                                                                                                                                                                                                                                                |
| _（隱含）_                                               | `channels.imessage.cliPath`               | `imsg` 的路徑（預設為 `imsg`）；使用包裝指令碼來處理 SSH。                                                                                                                                                                                                                   |
| _（隱含）_                                               | `channels.imessage.dbPath`                | 可選的 Messages.app `chat.db` 覆寫值；省略時會自動偵測。                                                                                                                                                                                                            |
| _（隱含）_                                               | `channels.imessage.remoteHost`            | `host` 或 `user@host`——僅在 `cliPath` 是 SSH 包裝指令碼，且你想使用 SCP 擷取附件時才需要。                                                                                                                                                                        |
| `channels.bluebubbles.dmPolicy`                            | `channels.imessage.dmPolicy`              | 值相同（`pairing` / `allowlist` / `open` / `disabled`）；預設為 `pairing`。                                                                                                                                                                                                  |
| `channels.bluebubbles.allowFrom`                           | `channels.imessage.allowFrom`             | 控制代碼格式相同（`+15555550123`、`user@example.com`）。配對儲存區的核准不會移轉——請參閱下文。                                                                                                                                                                   |
| `channels.bluebubbles.groupPolicy`                         | `channels.imessage.groupPolicy`           | 值相同（`allowlist` / `open` / `disabled`）；預設為 `allowlist`。                                                                                                                                                                                                            |
| `channels.bluebubbles.groupAllowFrom`                      | `channels.imessage.groupAllowFrom`        | 相同。未設定時，iMessage 會回退至 `allowFrom`；明確設為空的 `groupAllowFrom: []` 會封鎖 `groupPolicy: "allowlist"` 下的所有群組。                                                                                                                               |
| `channels.bluebubbles.groups`                              | `channels.imessage.groups`                | 逐字複製 `"*"` 萬用字元項目；以數字 iMessage `chat_id` 重新設定各群組項目的索引鍵——請參閱「群組登錄陷阱」。`requireMention`、`tools`、`toolsBySender`、`systemPrompt` 會沿用。                                                                            |
| `channels.bluebubbles.sendReadReceipts`                    | `channels.imessage.sendReadReceipts`      | 預設為 `true`。使用內建外掛時，只有在私有 API 探測正常運作時才會觸發。                                                                                                                                                                                        |
| `channels.bluebubbles.includeAttachments`                  | `channels.imessage.includeAttachments`    | 結構相同，且同樣預設關閉。如果 BlueBubbles 原本會傳送附件，請明確設定此項——在此之前，傳入的照片／媒體會被無聲捨棄（不會出現 `Inbound message` 記錄行）。                                                                                             |
| `channels.bluebubbles.attachmentRoots`                     | `channels.imessage.attachmentRoots`       | 本機根目錄；萬用字元規則相同。                                                                                                                                                                                                                                                |
| _（不適用）_                                                    | `channels.imessage.remoteAttachmentRoots` | 僅在設定 `remoteHost` 以透過 SCP 擷取時使用。                                                                                                                                                                                                                              |
| `channels.bluebubbles.mediaMaxMb`                          | `channels.imessage.mediaMaxMb`            | iMessage 的預設值為 16 MB（BlueBubbles 的預設值為 8 MB）。若要維持較低的上限，請明確設定。                                                                                                                                                                                  |
| `channels.bluebubbles.textChunkLimit`                      | `channels.imessage.textChunkLimit`        | 兩者的預設值皆為 4000。                                                                                                                                                                                                                                                            |
| `channels.bluebubbles.coalesceSameSenderDms`               | _（已移除）_                               | 請勿遷移此索引鍵。`imsg` 0.13.1 及更新版本會在 OpenClaw 收到訊息前，合併 Apple URL 預覽造成的拆分傳送；`openclaw doctor --fix` 會移除過時的 iMessage 索引鍵。                                                                                                    |
| `channels.bluebubbles.enrichGroupParticipantsFromContacts` | _（不適用）_                                   | `imsg` 已會顯示來自 `chat.db` 的傳送者顯示名稱。                                                                                                                                                                                                                     |
| `channels.bluebubbles.actions.*`                           | `channels.imessage.actions.*`             | 各動作的切換項目相同（`reactions`、`edit`、`unsend`、`reply`、`sendWithEffect`、`renameGroup`、`setGroupIcon`、`addParticipant`、`removeParticipant`、`leaveGroup`、`sendAttachment`），另新增 `polls`。全部預設啟用；私有 API 動作仍需要橋接器。 |

多帳號設定（`channels.bluebubbles.accounts.*`）會一對一轉換為 `channels.imessage.accounts.*`。

## 群組登錄陷阱

內建 iMessage 外掛會依序執行兩道群組閘門。群組訊息必須同時通過兩者，才能送達代理程式：

1. **傳送者／聊天目標允許清單**（`channels.imessage.groupAllowFrom`）——比對傳送者控制代碼或聊天目標（`chat_id:`、`chat_guid:`、`chat_identifier:` 項目）。未設定 `groupAllowFrom` 時，此閘門會回退至 `allowFrom`；明確設定 `groupAllowFrom: []` 會停用該回退機制，並捨棄 `groupPolicy: "allowlist"` 下的每一則群組訊息。
2. **群組登錄**（`channels.imessage.groups`）——以數字 iMessage `chat_id` 為索引鍵：
   - 沒有 `groups` 區塊（或區塊為空）：只要閘門 1 的有效傳送者允許清單非空，群組就能通過此閘門；存取權由傳送者篩選控制，且啟動時不會觸發全部捨棄警告。
   - `groups` 有項目但沒有 `"*"`：只有列出的 `chat_id` 索引鍵能通過。即使在 `groupPolicy: "open"` 下，只要列出任何群組，登錄就會變成允許清單。
   - `groups: { "*": { ... } }`：所有群組都能通過此閘門。

遷移陷阱：BlueBubbles 使用聊天 GUID／聊天識別碼作為 `groups` 項目的索引鍵，而 iMessage 登錄則使用數字 `chat_id`。逐字複製各群組項目會建立非空的登錄，但其中的索引鍵永遠無法比對，因此每一則群組訊息都會在閘門 2 遭到捨棄。請逐字複製 `"*"` 萬用字元；使用 `imsg chats` 中的 `chat_id` 值，重新設定特定群組項目的索引鍵。

兩種捨棄路徑都可透過預設記錄層級的 `warn` 行查看：

- 每個帳號在啟動時一次：當設定了 `groupPolicy: "allowlist"`，且有效的群組傳送者允許清單為空時：`imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...`。設定 `groupAllowFrom`（或 `allowFrom`）以允許傳送者；僅新增 `groups` 無法滿足傳送者閘門。
- 執行階段中每個 `chat_id` 一次：當登錄捨棄群組時：`imessage: dropping group message from chat_id=<id> ... not in channels.imessage.groups allowlist`，其中會指出要新增的確切索引鍵。

無論如何，私訊都會繼續運作——它們採用不同的程式碼路徑，因此私訊成功並不能證明群組路由正常。

使用 `groupPolicy: "allowlist"`、僅限定傳送者的最小設定：

```json5
{
  channels: {
    imessage: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15555550123", "chat_guid:any;-;..."],
    },
  },
}
```

這會允許設定的傳送者在任何群組中傳送訊息。新增 `groups` 項目可限定允許的聊天，或設定 `requireMention` 等各聊天選項；請逐字複製 BlueBubbles 的 `"*"` 項目，但使用數字 iMessage `chat_id` 值重新設定特定項目的索引鍵。

## 逐步操作

1. 轉換設定。編輯時請讓新區塊維持停用；目前的 OpenClaw 會忽略舊的 `channels.bluebubbles` 區塊，因此可將其保留在旁作為參考：

   ```json5
   {
     channels: {
       imessage: {
         enabled: false, // 準備好切換時改為 true
         cliPath: "/opt/homebrew/bin/imsg",
         dmPolicy: "pairing",
         allowFrom: ["+15555550123"], // 從 bluebubbles.allowFrom 複製
         groupPolicy: "allowlist",
         groupAllowFrom: [], // 從 bluebubbles.groupAllowFrom 複製
         groups: { "*": { requireMention: true } }, // 萬用字元逐字複製；以 chat_id 重新設定各聊天項目的索引鍵
         // 動作預設啟用；將個別切換項目設為 false 即可停用
       },
     },
   }
   ```

2. **切換並探測。** 設定 `channels.imessage.enabled: true`、重新啟動閘道，並確認頻道回報狀態正常：

   ```bash
   openclaw gateway restart
   openclaw channels status --probe --channel imessage   # 預期顯示 "works"；--json 會顯示 privateApi.available: true
   ```

   探測需要可連線的閘道，且只會探測已設定並啟用的帳戶。使用[開始之前](#before-you-start)中的直接 `imsg` 命令來驗證 Mac 本身。

3. **驗證私訊。** 傳送直接訊息給代理程式；確認回覆成功送達。

4. **分別驗證群組。** 私訊和群組使用不同的程式碼路徑——私訊成功不代表群組路由正常。請在允許的群組聊天中傳送訊息，並確認回覆成功送達。如果群組毫無回應（沒有代理程式回覆，也沒有錯誤），請檢查閘道記錄中是否出現上方「群組登錄陷阱」提到的兩行 `warn`。啟動警告表示實際生效的傳送者允許清單為空；每個 `chat_id` 的警告則表示已有內容的 `groups` 登錄中不包含該聊天。

5. **驗證動作介面。** 從已配對的私訊中，要求代理程式加入回應、編輯、收回、回覆、傳送照片，以及（在群組中）重新命名群組或新增／移除參與者。每項動作都應原生呈現在 Messages.app 中。如果任何動作擲回 `iMessage <action> requires the imsg private API bridge`，請再次執行 `imsg launch`，並使用 `openclaw channels status --probe` 重新整理。

6. **移除 BlueBubbles 伺服器和 `channels.bluebubbles` 區塊**，但請先確認 iMessage 私訊、群組和動作均已通過驗證。OpenClaw 不會讀取 `channels.bluebubbles`。

## 動作功能對照一覽

| 動作                                                | 舊版 BlueBubbles    | 內建 iMessage                                                                 |
| --------------------------------------------------- | ------------------ | ----------------------------------------------------------------------------- |
| 傳送文字／SMS 備援                                 | ✅                 | ✅                                                                            |
| 傳送媒體（照片、影片、檔案、語音）                 | ✅                 | ✅                                                                            |
| 討論串回覆（`reply_to_guid`）                    | ✅                 | ✅（解決 [#51892](https://github.com/openclaw/openclaw/issues/51892)）       |
| Tapback（`react`）                       | ✅                 | ✅                                                                            |
| 編輯／收回（macOS 13+ 收件者）                     | ✅                 | ✅                                                                            |
| 使用螢幕效果傳送                                    | ✅                 | ✅（解決 [#9394](https://github.com/openclaw/openclaw/issues/9394) 的部分問題） |
| RTF 粗體／斜體／底線／刪除線                       | ✅                 | ✅（透過 attributedBody 的具型別文字範圍格式）                               |
| 原生 Messages 投票（建立與投票）                   | ❌                 | ✅（`actions.polls`；收件者需要 iOS/macOS 26+ 才能原生呈現）              |
| 重新命名群組／設定群組圖示                          | ✅                 | ✅                                                                            |
| 新增／移除參與者、離開群組                          | ✅                 | ✅                                                                            |
| 已讀回條和輸入中指示器                              | ✅                 | ✅（須通過私有 API 探測）                                                     |
| Apple URL 預覽分拆傳送合併                          | ✅                 | ✅（由上游 `imsg` 0.13.1 以上版本處理；無 OpenClaw 設定）         |
| 重新啟動後復原輸入訊息                              | ✅                 | ✅（自動：`since_rowid` 重播 + GUID 去重；本機的時間範圍較寬）          |

iMessage 會復原閘道停機期間遺漏的訊息：啟動時，它會透過 `imsg watch.subscribe` `since_rowid` 從最後派送的 rowid 開始重播、依 GUID 去重，並使用過期待處理訊息的時間限制來抑制 Push 清空造成的「待處理訊息炸彈」。此程序透過 `imsg` RPC 連線執行，因此也適用於遠端 SSH `cliPath` 設定；本機設定因為可以讀取 `chat.db`，所以復原時間範圍較寬。請參閱[橋接器或閘道重新啟動後的輸入訊息復原](/zh-TW/channels/imessage#inbound-recovery-after-a-bridge-or-gateway-restart)。

## 配對、工作階段和 ACP 繫結

- **允許清單會依識別代號沿用。** `channels.imessage.allowFrom` 可辨識 BlueBubbles 使用的相同 `+15555550123`／`user@example.com` 字串——請逐字複製。
- **配對儲存區的核准不會轉移。** 配對儲存區依頻道區分，且沒有任何機制會遷移舊的 BlueBubbles 儲存區。僅透過配對獲得核准的傳送者，必須在 iMessage 下重新配對一次，否則你需要將他們的識別代號新增至 `allowFrom`。
- **工作階段**仍以每個代理程式 + 聊天為範圍。在預設的 `session.dmScope=main` 下，私訊會合併至代理程式的主要工作階段；群組工作階段則會依各個 `chat_id`（`agent:<agentId>:imessage:group:<chat_id>`）保持隔離。BlueBubbles 工作階段金鑰下的舊對話記錄不會轉移至 iMessage 工作階段。
- **ACP 繫結**中引用的 `match.channel: "bluebubbles"` 必須改為 `"imessage"`。`match.peer.id` 格式（`chat_id:`、`chat_guid:`、`chat_identifier:`、單獨的識別代號）完全相同。

## 沒有復原頻道

沒有受支援的 BlueBubbles 執行階段可供切換回去。如果 iMessage 驗證失敗，請設定 `channels.imessage.enabled: false`、重新啟動閘道、修正 `imsg` 阻礙，然後重試移轉。

回覆快取位於 SQLite 外掛狀態中。`openclaw doctor --fix` 會在舊的 `imessage/reply-cache.jsonl` 輔助檔案存在時匯入並封存它。

## 相關內容

- [移除 BlueBubbles 與 imsg iMessage 路徑](/zh-TW/announcements/bluebubbles-imessage)——簡短公告與操作人員摘要。
- [iMessage](/zh-TW/channels/imessage)——完整的 iMessage 頻道參考資料，包括 `imsg launch` 設定與功能偵測。
- `/channels/bluebubbles`——重新導向至本移轉指南的舊版 URL。
- [配對](/zh-TW/channels/pairing)——私訊驗證與配對流程。
- [頻道路由](/zh-TW/channels/channel-routing)——閘道如何為輸出回覆選擇頻道。
