---
read_when:
    - 設定 iMessage 支援
    - 偵錯 iMessage 傳送／接收
summary: 透過 imsg（經由標準輸入輸出的 JSON-RPC）提供原生 iMessage 支援，並以私有 API 執行回覆、點按回應、特效、投票、附件及群組管理等操作。當主機需求相符時，建議新的 OpenClaw iMessage 設定採用此方式。
title: iMessage
x-i18n:
    generated_at: "2026-07-26T07:43:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f3e8b1a65c76b25d03615c06a976f86a8af555cd96d5bfdb10cef9c955893ddc
    source_path: channels/imessage.md
    workflow: 16
---

<Note>
在一般的 OpenClaw iMessage 部署中，請在同一台已登入 macOS「訊息」的主機上執行閘道和 `imsg`。如果閘道在其他地方執行，請將 `channels.imessage.cliPath` 指向透明的 SSH 包裝程式，由它在 Mac 上執行 `imsg`。

**輸入復原會自動進行。** 在橋接器或閘道重新啟動後，iMessage 會重播停機期間遺漏的訊息，並抑制 Apple 在推播復原後可能一次送出的過時「積壓訊息炸彈」，同時進行去重，確保不會重複分派任何內容。無需設定即可啟用——請參閱[橋接器或閘道重新啟動後的輸入復原](#inbound-recovery-after-a-bridge-or-gateway-restart)。
</Note>

<Warning>
已移除 BlueBubbles 支援。請將 `channels.bluebubbles` 設定遷移至 `channels.imessage`；OpenClaw 僅透過 `imsg` 支援 iMessage。簡短公告請先參閱 [BlueBubbles 移除與 imsg iMessage 路徑](/zh-TW/announcements/bluebubbles-imessage)，完整遷移表則請參閱[從 BlueBubbles 遷移](/zh-TW/channels/imessage-from-bluebubbles)。
</Warning>

狀態：原生外部命令列介面整合。閘道會啟動 `imsg rpc`，並透過 stdio 使用 JSON-RPC 通訊——不需要獨立的常駐程式或連接埠。強烈建議使用 Private API 模式，以獲得完整的 iMessage 頻道功能；回覆、點按回應、特效、投票、附件回覆和群組操作都需要 `imsg launch`，且 Private API 探測必須成功。

對於常見的本機設定，OpenClaw 設定流程可在已登入「訊息」的 Mac 上，經使用者確認後透過 Homebrew 安裝或更新 `imsg`。手動設定與 SSH 包裝程式拓撲仍由操作人員管理：請在將執行閘道或包裝程式的相同使用者環境中安裝或更新 `imsg`。

<CardGroup cols={3}>
  <Card title="Private API 操作" icon="wand-sparkles" href="#private-api-actions">
    回覆、點按回應、特效、投票、附件和群組管理。
  </Card>
  <Card title="配對" icon="link" href="/zh-TW/channels/pairing">
    iMessage 私訊預設使用配對模式。
  </Card>
  <Card title="遠端 Mac" icon="terminal" href="#remote-mac-over-ssh">
    當閘道未在「訊息」Mac 上執行時，請使用 SSH 包裝程式。
  </Card>
  <Card title="設定參考" icon="settings" href="/zh-TW/gateway/config-channels#imessage">
    完整的 iMessage 欄位參考。
  </Card>
</CardGroup>

## 快速設定

<Tabs>
  <Tab title="本機 Mac（快速路徑）">
    <Steps>
      <Step title="安裝並驗證 imsg">

```bash
brew install steipete/tap/imsg
brew update && brew upgrade imsg
imsg rpc --help
imsg launch
openclaw channels status --probe
```

        當本機設定精靈偵測到缺少預設的 `imsg` 命令時，可以提示透過 Homebrew 安裝 `steipete/tap/imsg`。如果偵測到由 Homebrew 管理的 `imsg`，則可以提示重新安裝或更新。自訂的 `cliPath` 包裝程式不會被修改。

      </Step>

      <Step title="設定 OpenClaw">

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/user/Library/Messages/chat.db",
    },
  },
}
```

      </Step>

      <Step title="啟動閘道">

```bash
openclaw gateway
```

      </Step>

      <Step title="核准第一個私訊配對（預設 dmPolicy）">

```bash
openclaw pairing list imessage
openclaw pairing approve imessage <CODE>
```

        配對要求會在 1 小時後到期。
      </Step>
    </Steps>

  </Tab>

  <Tab title="透過 SSH 使用遠端 Mac">
    大多數設定不需要 SSH。只有在閘道無法於已登入「訊息」的 Mac 上執行時，才使用此拓撲。OpenClaw 只需要與 stdio 相容的 `cliPath`，因此可以將 `cliPath` 指向一個包裝指令碼，由它透過 SSH 連線至遠端 Mac 並執行 `imsg`。
    請在該遠端 Mac 上安裝及更新 `imsg`，而不是在閘道主機上：

```bash
ssh messages-mac 'brew install steipete/tap/imsg && brew update && brew upgrade imsg'
```

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    啟用附件時的建議設定：

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "user@gateway-host", // used for SCP attachment fetches
      includeAttachments: true,
      // Optional: extra allowed attachment roots (merged with the default
      // /Users/*/Library/Messages/Attachments).
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
    },
  },
}
```

    如果未設定 `remoteHost`，OpenClaw 會嘗試剖析 SSH 包裝指令碼來自動偵測。
    `remoteHost` 必須是 `host` 或 `user@host`（不可包含空格或 SSH 選項）；不安全的值會被忽略。
    OpenClaw 會對 SCP 使用嚴格的主機金鑰檢查，因此中繼主機金鑰必須已存在於 `~/.ssh/known_hosts` 中。
    附件路徑會依據允許的根目錄（`attachmentRoots` / `remoteAttachmentRoots`）進行驗證。

<Warning>
任何放在 `imsg` 前方的 `cliPath` 包裝程式或 SSH Proxy，都**必須**像透明的 stdio 管線一樣處理長時間運作的 JSON-RPC。在頻道的整個生命週期中，OpenClaw 會透過包裝程式的 stdin/stdout 交換以換行框定的小型 JSON-RPC 訊息：

- 每當位元組可用時，**立即**轉送每個 stdin 資料區塊／行——不要等待 EOF。
- 立即反向轉送每個 stdout 資料區塊／行。
- 保留換行字元。
- 避免使用固定大小的阻塞讀取（`read(4096)`、`cat | buffer`、Shell 預設的 `read`），否則可能使小型訊框無法傳送。
- 讓 stderr 與 JSON-RPC 的 stdout 串流保持分離。

如果包裝程式會緩衝 stdin，直到填滿大型資料區塊才送出，就會產生看似 iMessage 中斷的症狀——`imsg rpc timeout (chats.list)` 或頻道反覆重新啟動——即使 `imsg rpc` 本身運作正常。上述 `ssh -T host imsg "$@"` 是安全的，因為它會轉送 OpenClaw 的 `cliPath` 引數，例如 `rpc` 和 `--db`。像 `ssh host imsg | grep -v '^DEBUG'` 這類管線則**不安全**——即使是行緩衝工具仍可能留住訊框；如果必須篩選，請在每個階段使用 `stdbuf -oL -eL`。
</Warning>

  </Tab>
</Tabs>

## 需求與權限（macOS）

- 在執行 `imsg` 的 Mac 上，必須登入「訊息」。
- 執行 OpenClaw／`imsg` 的程序環境需要「完整磁碟存取權限」（用於存取「訊息」資料庫）。
- 透過 Messages.app 傳送訊息需要「自動化」權限。
- 若要使用進階操作（回應／編輯／收回／討論串回覆／特效／投票／群組操作），必須停用「系統完整性保護」——請參閱[啟用 imsg Private API](#enabling-the-imsg-private-api)。不停用也能使用基本的文字與媒體收發功能。

<Tip>
權限是依程序環境授予。如果閘道以無介面方式執行（LaunchAgent／SSH），請在相同環境中執行一次互動式命令以觸發權限提示：

```bash
imsg chats --limit 1
# or
imsg send <handle> "test"
```

</Tip>

<Accordion title="SSH 包裝程式傳送失敗並顯示 AppleEvents -1743">
  遠端 SSH 設定可能可以讀取聊天、通過 `channels status --probe` 並處理輸入訊息，但輸出傳送仍會因 AppleEvents 授權錯誤而失敗：

```text
Not authorized to send Apple events to Messages. (-1743)
```

請檢查已登入 Mac 使用者的 TCC 資料庫，或開啟 System Settings > Privacy & Security > Automation。如果「自動化」項目是記錄給 `/usr/libexec/sshd-keygen-wrapper`，而不是 `imsg` 或本機 Shell 程序，macOS 可能不會為該 SSH 伺服器端用戶端提供可用的「訊息」切換開關：

```text
kTCCServiceAppleEvents | /usr/libexec/sshd-keygen-wrapper | auth_value=0 | com.apple.MobileSMS
```

在此狀態下，重複執行 `tccutil reset AppleEvents`，或透過相同 SSH 包裝程式重新執行 `imsg send`，可能仍會失敗，因為需要「訊息」自動化權限的程序環境是 SSH 包裝程式，而不是 UI 可以授權的應用程式。

請改用下列其中一種受支援的 `imsg` 程序環境：

- 在已登入「訊息」的使用者本機工作階段中執行閘道，或至少執行 `imsg` 橋接器。
- 從相同工作階段授予「完整磁碟存取權限」和「自動化」權限後，以該使用者的 LaunchAgent 啟動閘道。
- 如果保留雙使用者 SSH 拓撲，請在啟用頻道前，確認實際的輸出 `imsg send` 可透過完全相同的包裝程式成功傳送。如果無法授予「自動化」權限，請重新設定為單一使用者的 `imsg` 設定，而不要依賴 SSH 包裝程式進行傳送。

</Accordion>

## 啟用 imsg Private API

`imsg` 提供兩種運作模式。對 OpenClaw 而言，建議使用 Private API 模式，因為它能讓頻道提供使用者期望的原生 iMessage 操作。基本模式仍適用於低風險安裝、初始驗證，或無法停用 SIP 的主機。

- **基本模式**（預設，無需變更 SIP）：透過 `send` 傳送文字與媒體、監看／取得輸入歷史記錄，以及列出聊天。全新安裝 `brew install steipete/tap/imsg` 並授予上述標準 macOS 權限後，即可直接使用這些功能。
- **Private API 模式**：`imsg` 會將輔助 dylib 注入 `Messages.app`，以呼叫內部的 `IMCore` 函式。這會解鎖 `react`、`edit`、`unsend`、`reply`（討論串）、`sendWithEffect`、`poll` 和 `poll-vote`（「訊息」原生投票）、`renameGroup`、`setGroupIcon`、`addParticipant`、`removeParticipant`、`leaveGroup`，以及輸入中指示器和已讀回條。

本頁建議的操作介面需要 Private API 模式。`imsg` README 明確說明了這項要求：

> `read`、`typing`、`launch`、由橋接器支援的豐富傳送功能、訊息修改及聊天管理等進階功能皆為選用功能。這些功能需要停用 SIP，並將輔助 dylib 注入 `Messages.app`。啟用 SIP 時，`imsg launch` 會拒絕注入。

此輔助程式注入技術使用 `imsg` 自己的 dylib 來存取「訊息」的 Private API。OpenClaw iMessage 路徑中沒有第三方伺服器或 BlueBubbles 執行階段。

<Warning>
**停用 SIP 確實會帶來安全性取捨。** SIP 是 macOS 防止執行遭修改系統程式碼的核心保護機制之一；在整個系統中將其關閉，會增加額外的攻擊面與副作用。特別要注意的是，**在 Apple Silicon Mac 上停用 SIP，也會讓你無法在 Mac 上安裝及執行 iOS App**。

請將此視為刻意做出的維運選擇，尤其是在主要的個人 Mac 上。若要獲得可用於正式環境的 OpenClaw iMessage，建議使用專用 Mac 或專用的機器人 macOS 使用者，並確保可以接受啟用此橋接器。如果你的威脅模型無法容許任何地方停用 SIP，內建的 iMessage 就只能使用基本模式——僅限文字與媒體收發，不支援回應／編輯／收回／特效／群組操作。
</Warning>

### 設定

1. 在執行 Messages.app 的 Mac 上**安裝（或升級）`imsg`**：

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg status --json
   ```

   `imsg status --json` 輸出會回報 `bridge_version`、`rpc_methods`，以及各方法的 `selectors`，讓你在開始前就能查看目前版本支援哪些功能。

2. **停用系統完整性保護，並且（在現代 macOS 上）停用程式庫驗證。** 將非 Apple 的輔助 dylib 注入由 Apple 簽署的 `Messages.app`，需要關閉 SIP，**並且**放寬程式庫驗證。復原模式中的 SIP 步驟會因 macOS 版本而異：
   - **macOS 10.13-10.15（Sierra-Catalina）：**透過終端機停用程式庫驗證，重新啟動至復原模式，執行 `csrutil disable`，然後重新啟動。
   - **macOS 11+（Big Sur 及更新版本），Intel：**進入復原模式（或網際網路復原），執行 `csrutil disable`，然後重新啟動。
   - **macOS 11+，Apple Silicon：**使用電源按鈕啟動流程進入復原模式；在近期的 macOS 版本中，按一下 Continue 時按住 **Left Shift** 鍵，然後執行 `csrutil disable`。虛擬機器設定採用不同的流程，因此請先建立 VM 快照。

   **在 macOS 11 及更新版本上，只有 `csrutil disable` 通常不夠。** Apple 仍會將 `Messages.app` 視為平台二進位檔並對其強制執行程式庫驗證，因此即使關閉 SIP，臨時簽署的輔助程式仍會遭到拒絕（`Library Validation failed: ... platform binary, but mapped file is not`）。停用 SIP 後，也請停用程式庫驗證並重新啟動：

   ```bash
   sudo defaults write /Library/Preferences/com.apple.security.libraryvalidation.plist DisableLibraryValidation -bool true
   ```

   **macOS 26（Tahoe），已於 26.5.1 驗證：**關閉 SIP **加上**上述 `DisableLibraryValidation` 命令，就足以在 26.0 至 26.5.x 之間注入輔助程式。**不需要任何 boot-args。** 此 plist 是決定性因素，也是 Tahoe 注入失敗時最常遺漏的步驟：
   - **有 plist 時：**`imsg launch` 會完成注入，且 `imsg status` 會回報 `advanced_features: true`。
   - **沒有 plist 時（即使已關閉 SIP）：**`imsg launch` 會失敗並顯示 `Failed to launch: Timeout waiting for Messages.app to initialize`。AMFI 會在載入時拒絕臨時簽署的輔助程式，因此橋接器永遠無法就緒，啟動作業最終會逾時。這個逾時是大多數人在 Tahoe 上遇到的症狀；解法是使用上述 plist，而不是採取更激進的措施。

   如果在 macOS 升級後，`imsg launch` 注入或特定 `selectors` 開始傳回 false，通常是這個閘門所致。在認定 SIP 步驟本身失敗前，請先檢查 SIP 和程式庫驗證狀態。如果這些設定正確，但橋接器仍無法注入，請收集 `imsg status --json` 以及 `imsg launch` 的輸出，並向 `imsg` 專案回報，而不要進一步削弱其他全系統安全控制。

3. **注入輔助程式。** 在已停用 SIP 且 Messages.app 已登入的情況下：

   ```bash
   imsg launch
   ```

   SIP 仍啟用時，`imsg launch` 會拒絕注入，因此這也可同時確認步驟 2 已生效。

4. **從 OpenClaw 驗證橋接器：**

   ```bash
   openclaw channels status --probe
   ```

   iMessage 項目應回報 `works`，且 `imsg status --json | jq '{rpc_methods, selectors}'` 應顯示你的 macOS 組建所提供的功能。建立投票需要 `selectors.pollPayloadMessage`；投票則同時需要 `selectors.pollVoteMessage` 和 `poll.vote` RPC 方法。OpenClaw 外掛只會公告快取探測所支援的動作，而空白快取會保持樂觀，並在首次分派時進行探測。

如果 `openclaw channels status --probe` 將頻道回報為 `works`，但特定動作在分派時擲回「iMessage `<action>` requires the imsg private API bridge」，請再次執行 `imsg launch`——輔助程式可能會脫離（Messages.app 重新啟動、作業系統更新等），而快取的 `available: true` 狀態會持續公告動作，直到下一次探測重新整理。

### SIP 保持啟用時

如果你的威脅模型無法接受停用 SIP：

- `imsg` 會退回基本模式——僅支援文字、媒體與接收。
- OpenClaw 外掛仍會公告文字／媒體傳送及入站監控；它會從動作介面隱藏 `react`、`edit`、`unsend`、`reply`、`sendWithEffect` 和群組操作（依各方法的功能閘門而定）。
- 你可以使用另一台已關閉 SIP 的非 Apple Silicon Mac（或專用機器人 Mac）處理 iMessage 工作負載，同時讓主要裝置維持 SIP 啟用。請參閱下方的[專用機器人 macOS 使用者（獨立的 iMessage 身分）](#deployment-patterns)。

## 存取控制與路由

<Tabs>
  <Tab title="私訊政策">
    `channels.imessage.dmPolicy` 控制直接訊息：

    - `pairing`（預設）
    - `allowlist`（至少需要一個 `allowFrom` 項目）
    - `open`（需要 `allowFrom` 包含 `"*"`）
    - `disabled`

    允許清單欄位：`channels.imessage.allowFrom`。

    允許清單項目必須識別傳送者：控制代碼或靜態傳送者存取群組（`accessGroup:<name>`）。對 `chat_id:*`、`chat_guid:*` 或 `chat_identifier:*` 等聊天目標使用 `channels.imessage.groupAllowFrom`；對數字型 `chat_id` 登錄機碼使用 `channels.imessage.groups`。

  </Tab>

  <Tab title="群組政策與提及">
    `channels.imessage.groupPolicy` 控制群組處理：

    - `allowlist`（預設）
    - `open`
    - `disabled`

    群組傳送者允許清單：`channels.imessage.groupAllowFrom`。

    `groupAllowFrom` 項目也可以參照靜態傳送者存取群組（`accessGroup:<name>`）。

    執行階段後援：如果未設定 `groupAllowFrom`，iMessage 群組傳送者檢查會使用 `allowFrom`；當私訊與群組准入應有所不同時，請設定 `groupAllowFrom`。明確設為空白的 `groupAllowFrom: []` 不會後援——它會在 `allowlist` 下封鎖所有群組傳送者。
    執行階段注意事項：如果完全缺少 `channels.imessage`，執行階段會退回 `groupPolicy="allowlist"` 並記錄警告（即使已設定 `channels.defaults.groupPolicy`）。

    <Warning>
    `groupPolicy: "allowlist"` 下的群組路由會接連執行**兩個**閘門：

    1. **傳送者允許清單**（`channels.imessage.groupAllowFrom`）——控制代碼、`accessGroup:<name>`、`chat_guid`、`chat_identifier` 或 `chat_id`。有效清單為空白（沒有 `groupAllowFrom`，也沒有 `allowFrom` 後援）時，會封鎖所有群組傳送者。
    2. **群組登錄**（`channels.imessage.groups`）——對應表中有項目後便會強制執行：聊天必須符合明確的個別 `chat_id` 項目或 `groups: { "*": { ... } }` 萬用字元。當 `groups` 為空白或缺少時，僅由傳送者允許清單決定是否准入。

    如果未設定有效的群組傳送者允許清單，每則群組訊息都會在登錄閘門前遭到捨棄。每個閘門在預設記錄層級都有自己的 `warn` 層級訊號，且各自指出不同的修正方式：

    - 每個帳號在啟動時一次：當有效的群組傳送者允許清單為空白時，會顯示 `imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...`——請設定 `channels.imessage.groupAllowFrom`（或 `allowFrom`）來修正；只新增 `groups` 項目仍會讓閘門 1 封鎖所有傳送者。
    - 每個 `chat_id` 在執行階段一次：當傳送者通過閘門 1，但聊天不存在於已有內容的 `groups` 登錄中時，會顯示 `imessage: dropping group message from chat_id=<id> ...`——請在 `channels.imessage.groups` 下新增該 `chat_id`（或 `"*"`）來修正。

    私訊不受影響——它們使用不同的程式碼路徑。

    `groupPolicy: "allowlist"` 下群組流程的建議設定：

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: { "*": { "requireMention": true } },
        },
      },
    }
    ```

    僅 `groupAllowFrom` 即可允許這些傳送者進入任何群組；新增 `groups` 區塊可限定允許哪些聊天（並設定 `requireMention` 等個別聊天選項）。
    </Warning>

    群組的提及閘門：

    - iMessage 沒有原生提及中繼資料
    - 提及偵測使用規則運算式模式（`agents.entries.*.groupChat.mentionPatterns`，後援為 `messages.groupChat.mentionPatterns`）
    - 未設定模式時，無法強制執行提及閘門
    - 來自已授權傳送者的控制命令會略過提及閘門

    個別群組的 `systemPrompt`：

    `channels.imessage.groups.*` 下的每個項目都接受選用的 `systemPrompt` 字串；每當處理該群組中的訊息時，此字串都會注入代理程式的系統提示詞。解析方式與 `channels.whatsapp.groups` 相同：

    1. **群組專屬系統提示詞**（`groups["<chat_id>"].systemPrompt`）：當對應表中存在特定群組項目，**且**已定義其 `systemPrompt` 鍵時使用。如果 `systemPrompt` 是空字串（`""`），則會抑制萬用字元，且不會對該群組套用任何系統提示詞。
    2. **群組萬用字元系統提示詞**（`groups["*"].systemPrompt`）：當對應表中完全沒有特定群組項目，或該項目存在但未定義 `systemPrompt` 鍵時使用。

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: {
            "*": { systemPrompt: "使用英式拼字。" },
            "8421": {
              requireMention: true,
              systemPrompt: "這是值班輪調聊天。回覆請勿超過 3 句。",
            },
            "9907": {
              // 明確抑制：萬用字元「使用英式拼字。」不會套用於此處
              systemPrompt: "",
            },
          },
        },
      },
    }
    ```

    個別群組提示詞只會套用於群組訊息——直接訊息不受影響。

  </Tab>

  <Tab title="工作階段與確定性回覆">
    - 私訊使用直接路由；群組使用群組路由。
    - 使用預設的 `session.dmScope=main` 時，iMessage 私訊會合併至代理程式的主要工作階段。
    - 群組工作階段會彼此隔離（`agent:<agentId>:imessage:group:<chat_id>`）。
    - 回覆會使用來源頻道／目標中繼資料路由回 iMessage。

    類群組討論串的行為：

    部分多人參與的 iMessage 討論串可能會隨 `is_group=false` 抵達。
    如果該 `chat_id` 已在 `channels.imessage.groups` 下明確設定，OpenClaw 會將其視為群組流量（群組閘門與群組工作階段隔離）。

  </Tab>
</Tabs>

## ACP 對話綁定

iMessage 聊天可以綁定至 ACP 工作階段。

快速操作流程：

- 在私訊或允許的群組聊天中執行 `/acp spawn codex --bind here`。
- 之後同一個 iMessage 對話中的訊息會路由至已產生的 ACP 工作階段。
- `/new` 和 `/reset` 會就地重設同一個已綁定的 ACP 工作階段。
- `/acp close` 會關閉 ACP 工作階段並移除綁定。

設定的持續性綁定使用頂層 `bindings[]` 項目，並搭配 `type: "acp"` 和 `match.channel: "imessage"`。

`match.peer.id` 可以使用：

- 正規化的私訊控制代碼，例如 `+15555550123` 或 `user@example.com`
- `chat_id:<id>`（建議用於穩定的群組綁定）
- `chat_guid:<guid>`
- `chat_identifier:<identifier>`

範例：

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: { agent: "codex", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "imessage",
        accountId: "default",
        peer: { kind: "group", id: "chat_id:123" },
      },
      acp: { label: "codex-group" },
    },
  ],
}
```

共用 ACP 綁定行為請參閱 [ACP 代理程式](/zh-TW/tools/acp-agents)。

## 部署模式

<AccordionGroup>
  <Accordion title="專用機器人 macOS 使用者（獨立的 iMessage 身分）">
    使用專用 Apple ID 和 macOS 使用者，將機器人流量與你的個人 Messages 設定檔隔離。

    一般流程：

    1. 建立／登入專用的 macOS 使用者。
    2. 在該使用者中，使用機器人的 Apple ID 登入「訊息」。
    3. 在該使用者中安裝 `imsg`。
    4. 建立 SSH 包裝程式，讓 OpenClaw 能在該使用者的環境中執行 `imsg`。
    5. 將 `channels.imessage.accounts.<id>.cliPath` 和 `.dbPath` 指向該使用者設定檔。

    第一次執行時，可能需要在該機器人使用者工作階段的圖形介面中核准權限（自動化 + 完整磁碟存取權）。

  </Accordion>

  <Accordion title="透過 Tailscale 連線至遠端 Mac（範例）">
    常見拓撲：

    - 閘道在 Linux／VM 上執行
    - iMessage + `imsg` 在你 tailnet 中的 Mac 上執行
    - `cliPath` 包裝程式使用 SSH 執行 `imsg`
    - `remoteHost` 啟用透過 SCP 擷取附件

    範例：

    ```json5
    {
      channels: {
        imessage: {
          enabled: true,
          cliPath: "~/.openclaw/scripts/imsg-ssh",
          remoteHost: "bot@mac-mini.tailnet-1234.ts.net",
          includeAttachments: true,
          dbPath: "/Users/bot/Library/Messages/chat.db",
        },
      },
    }
    ```

    ```bash
    #!/usr/bin/env bash
    exec ssh -T bot@mac-mini.tailnet-1234.ts.net imsg "$@"
    ```

    使用 SSH 金鑰，讓 SSH 和 SCP 都能以非互動方式執行。
    請先確保主機金鑰已受信任（例如 `ssh bot@mac-mini.tailnet-1234.ts.net`），以便填入 `known_hosts`。

  </Accordion>

  <Accordion title="多帳號模式">
    iMessage 支援在 `channels.imessage.accounts` 下設定各帳號。

    每個帳號都能覆寫 `cliPath`、`dbPath`、`allowFrom`、`groupPolicy`、`mediaMaxMb`、歷史記錄設定和附件根目錄允許清單等欄位。

  </Accordion>

  <Accordion title="私人訊息歷史記錄">
    設定 `channels.imessage.dmHistoryLimit`，以使用該對話最近解碼的 `imsg` 歷史記錄，作為新私人訊息工作階段的初始內容。使用 `channels.imessage.dms["<sender>"].historyLimit` 設定各傳送者的覆寫值，包括使用 `0` 停用某位傳送者的歷史記錄。

    iMessage 私人訊息歷史記錄會視需要從 `imsg` 擷取。若未設定 `dmHistoryLimit`，將停用全域私人訊息歷史記錄的初始填入；但若某位傳送者的 `channels.imessage.dms["<sender>"].historyLimit` 為正值，仍會為該傳送者啟用初始填入。

  </Accordion>
</AccordionGroup>

## 媒體、分段與傳送目標

<AccordionGroup>
  <Accordion title="附件與媒體">
    - 傳入附件的擷取功能**預設為關閉** — 設定 `channels.imessage.includeAttachments: true`，將照片、語音備忘錄、影片和其他附件轉送給代理程式。停用時，僅含附件的 iMessage 會在送達代理程式前遭到捨棄，並且可能完全不會產生 `Inbound message` 記錄行。
    - 設定 `remoteHost` 後，可透過 SCP 擷取遠端附件路徑
    - 附件路徑必須符合允許的根目錄：
      - `channels.imessage.attachmentRoots`（本機）
      - `channels.imessage.remoteAttachmentRoots`（遠端 SCP 模式）
      - 設定的根目錄會擴充預設根目錄模式 `/Users/*/Library/Messages/Attachments`（合併，而非取代）
    - SCP 使用嚴格主機金鑰檢查（`StrictHostKeyChecking=yes`）
    - 傳出媒體大小使用 `channels.imessage.mediaMaxMb`（預設為 16 MB）

  </Accordion>

  <Accordion title="傳出文字與分段">
    - 文字分段限制：`channels.imessage.textChunkLimit`（預設為 4000）
    - 分段模式：`channels.imessage.streaming.chunkMode`
      - `length`（預設）
      - `newline`（優先依段落分割）
    - 傳出 Markdown 的粗體／斜體／底線／刪除線會轉換成原生樣式文字（macOS 15+ 的收件者會顯示樣式；較舊版本的收件者會看到不含標記的純文字）；Markdown 表格會依照頻道的 Markdown 表格模式轉換
    - `channels.imessage.sendTransport`（`auto` 為預設值，另有 `bridge`、`applescript`）選擇 `imsg` 傳送訊息的方式

  </Accordion>

  <Accordion title="定址格式">
    建議使用明確目標：

    - `chat_id:123`（建議用於穩定路由）
    - `chat_guid:...`
    - `chat_identifier:...`

    也支援帳號代稱目標：

    - `imessage:+1555...`
    - `sms:+1555...`
    - `user@example.com`

    ```bash
    imsg chats --limit 20
    ```

  </Accordion>
</AccordionGroup>

## 私有 API 動作

當 `imsg launch` 正在執行，且 `openclaw channels status --probe` 回報 `privateApi.available: true` 時，訊息工具除了傳送一般文字外，還能使用 iMessage 原生動作。

所有動作預設皆已啟用；使用 `channels.imessage.actions` 可個別關閉動作：

```json5
{
  channels: {
    imessage: {
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
        renameGroup: true,
        setGroupIcon: true,
        addParticipant: true,
        removeParticipant: true,
        leaveGroup: true,
        polls: true,
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="可用動作">
    - **react**：新增／移除 iMessage 點按回應（`messageId`、`emoji`、`remove`）。支援的點按回應對應愛心、讚、倒讚、大笑、強調和疑問。不指定表情符號即執行移除時，會清除目前設定的任何點按回應。
    - **reply**：對現有訊息傳送討論串回覆（`messageId`、`text` 或 `message`，以及 `chatGuid`、`chatId`、`chatIdentifier` 或 `to`）。回覆並附加附件還需要使用其 `send-rich` 支援 `--file` 的 `imsg` 組建版本。
    - **sendWithEffect**：使用 iMessage 特效傳送文字（`text` 或 `message`、`effect` 或 `effectId`）。短名稱：slam、loud、gentle、invisibleink、confetti、lasers、fireworks、balloon、heart、echo、happybirthday、shootingstar、sparkles、spotlight。
    - **edit**：在支援的 macOS／私有 API 版本上編輯已傳送的訊息（`messageId`、`text` 或 `newText`）。只能編輯由閘道本身傳送的訊息。
    - **unsend**：在支援的 macOS／私有 API 版本上收回已傳送的訊息（`messageId`）。只能收回由閘道本身傳送的訊息。
    - **upload-file**：傳送媒體／檔案（`buffer` 使用 base64，或已載入的 `media`/`path`/`filePath`、`filename`，以及選用的 `asVoice`）。舊版別名：`sendAttachment`。
    - **renameGroup**、**setGroupIcon**、**addParticipant**、**removeParticipant**、**leaveGroup**：目前目標為群組對話時管理群組聊天。這些動作會變更主機的「訊息」身分，因此需要擁有者傳送者或 `operator.admin` 閘道用戶端。
    - **poll**：建立原生 Apple「訊息」投票（`pollQuestion`、重複 2 至 12 次的 `pollOption`，以及 `chatGuid`、`chatId`、`chatIdentifier` 或 `to`）。使用 iOS/iPadOS/macOS 26+ 的收件者可直接查看並投票；較舊的作業系統版本會收到「Sent a poll」文字備援訊息。需要 `selectors.pollPayloadMessage`。
    - **poll-vote**：對現有投票進行投票（`pollId` 或 `messageId`，以及 `pollOptionIndex`、`pollOptionId` 或 `pollOptionText` 中恰好一項）。需要 `selectors.pollVoteMessage` 和 `poll.vote` RPC 方法。

    已接受的傳入投票會呈現給代理程式，其中包含問題、編號的選項標籤、票數，以及 `poll-vote` 所需的投票訊息 ID。

  </Accordion>

  <Accordion title="訊息 ID">
    傳入的 iMessage 環境資訊會同時包含簡短的 `MessageSid` 值，以及可用時的完整訊息 GUID（`MessageSidFull`）。簡短 ID 的有效範圍僅限近期以 SQLite 為後端的回覆快取，使用前會先核對目前的聊天。如果簡短 ID 已過期，請在以提供該 ID 的對話為目標時，改用其 `MessageSidFull` 重試。完整 ID 不會略過對話或帳號繫結，因此若 ID 來自其他聊天，請換成目前目標中的 ID。當無法取得目前對話的證據時，遠端委派呼叫可能會拒絕過時的完整 ID。

  </Accordion>

  <Accordion title="功能偵測">
    OpenClaw 只會在快取的探查狀態指出橋接器無法使用時，隱藏私有 API 動作。如果狀態未知，動作仍會顯示，且會在分派時延遲探查，使 `imsg launch` 之後的第一個動作無須另外手動重新整理狀態即可成功。

  </Accordion>

  <Accordion title="已讀回條與輸入狀態">
    私有 API 橋接器啟用時，已接受的傳入聊天會標示為已讀；而在回合獲接受後，私人聊天會立即顯示輸入中的氣泡，同時代理程式準備環境資訊並產生內容。若要停用標示已讀：

    ```json5
    {
      channels: {
        imessage: {
          sendReadReceipts: false,
        },
      },
    }
    ```

    在各方法功能清單出現前的舊版 `imsg` 組建版本中，輸入狀態／已讀功能會在無提示的情況下遭到停用；OpenClaw 每次重新啟動時會記錄一次警告，以便判斷未收到回條的原因。

  </Accordion>

  <Accordion title="傳入點按回應">
    OpenClaw 會訂閱 iMessage 點按回應，並將已接受的回應作為系統事件路由，而非一般訊息文字，因此使用者的點按回應不會觸發一般回覆迴圈。

    通知模式由 `channels.imessage.reactionNotifications` 控制：

    - `"own"`（預設）：僅在使用者回應機器人撰寫的訊息時通知。
    - `"all"`：針對已授權傳送者的所有傳入點按回應發出通知。
    - `"off"`：忽略傳入點按回應。

    各帳號的覆寫值使用 `channels.imessage.accounts.<id>.reactionNotifications`。

  </Accordion>

  <Accordion title="核准回應（👍 / 👎）">
    當 `approvals.exec.enabled` 或 `approvals.plugin.enabled` 為 true，且要求路由至 iMessage 時，閘道會以原生方式傳送核准提示，並接受點按回應以完成處理：

    - `👍`（讚點按回應）→ `allow-once`
    - `👎`（倒讚點按回應）→ `deny`
    - `allow-always` 仍可作為手動備援：將 `/approve <id> allow-always` 作為一般回覆傳送。

    回應處理要求做出回應之使用者的帳號代稱必須明確列為核准者。核准者清單讀取自 `channels.imessage.allowFrom`（或 `channels.imessage.accounts.<id>.allowFrom`）；請加入使用者採 E.164 格式的電話號碼或其 Apple ID 電子郵件地址（`chat_id:*` 等聊天目標不是有效的核准者項目）。支援萬用字元項目 `"*"`，但這會允許任何傳送者核准；空白核准者清單會完全停用回應捷徑。回應捷徑會刻意略過 `reactionNotifications`、`dmPolicy` 和 `groupAllowFrom`，因為明確核准者允許清單是完成核准時唯一重要的閘門。

    `/approve` 文字命令授權遵循相同清單：當 `channels.imessage.allowFrom` 不為空時，會依據該核准者清單授權 `/approve <id> <decision>`（而非較廣泛的私人訊息允許清單），且私人訊息允許清單允許、但不在 `allowFrom` 中的傳送者會收到明確的拒絕訊息。當 `allowFrom` 為空時，同一聊天備援會維持生效，且 `/approve` 會授權私人訊息允許清單所允許的所有人。請將所有應能核准的操作員（無論透過 `/approve` 或回應）加入 `allowFrom`。

    Operator 注意事項：
    - 回應綁定會同時儲存在記憶體與閘道的持久化鍵值儲存區中（TTL 與核准到期時間一致），而且閘道也會輪詢待處理提示中的 tapback，因此即使 tapback 在閘道重新啟動後不久才送達，仍可完成核准。
    - 當操作者自己的 `is_from_me=true` tapback（例如來自已配對的 Apple 裝置）所使用的識別代號是明確指定的核准者時，便可完成核准。
    - 只有在已設定明確核准者時，核准提示才會路由至群組對話；否則任何群組成員都可能核准。
    - 舊版文字樣式的 tapback（`Liked "…"`，來自非常舊的 Apple 用戶端的純文字）無法完成核准，因為其中不含訊息 GUID；回應解析需要目前 macOS / iOS 用戶端所發出的結構化 tapback 中繼資料。

  </Accordion>

  <Accordion title="問題回應（1️⃣ / 2️⃣ / 3️⃣ / 4️⃣）">
    對於含有一個非機密單選問題及一至四個選項的 `ask_user` 提示，OpenClaw 會加入編號表情符號選項。請以相符的數字回應已送達的提示來作答。該回應必須包含由機器人撰寫之訊息的穩定 GUID；OpenClaw 隨後會透過閘道將數字對應至標準選項。過期或重複的點按會被忽略。

    多問題、多選及自由文字提示仍只能透過文字回覆。問題回應遵循一般 iMessage 私訊／群組准入規則。即使一般 `reactionNotifications` 為 `"off"`，仍會辨識這類回應，而不會將無關回應轉為代理程式事件。

  </Accordion>
</AccordionGroup>

## 設定寫入

iMessage 預設允許由頻道發起設定寫入（適用於 `/config set|unset` 為 `commands.config: true` 時）。

停用：

```json5
{
  channels: {
    imessage: {
      configWrites: false,
    },
  },
}
```

<a id="coalescing-split-send-dms-command--url-in-one-composition"></a>

## 合併分開傳送的私訊（在同一次撰寫中包含命令與 URL）

Apple 可能會將命令及其 URL 預覽儲存為不同的實體 `chat.db` 資料列。`imsg` 0.13.1 以上版本會在監看、歷程記錄或搜尋傳回訊息前合併這些資料列，因此 OpenClaw 會收到一則邏輯上的入站訊息，而不需增加頻道特定的私訊延遲。

不需設定任何 iMessage 合併選項。已停用的 `channels.imessage.coalesceSameSenderDms` 鍵會由 `openclaw doctor --fix` 移除。若你有意將某個頻道中快速連續傳送的文字訊息批次處理，仍可使用通用 `messages.inbound` 防彈跳機制。

如果命令加 URL 的傳送內容成為不同的代理程式輪次，請在 Messages Mac 上更新 `imsg`：

```bash
brew update && brew upgrade imsg
```

## 橋接器或閘道重新啟動後的入站復原

iMessage 會復原閘道停機期間遺漏的訊息，同時抑制 Apple 在 Push 復原後可能一次排出的過期「積壓轟炸」。此預設行為一律啟用，建構於持久入站機制與存留時間界線之上。

- **持久化重播防護。** 在推進復原游標之前，OpenClaw 會將每個原始資料列記錄至共用 SQLite 入站佇列，並以其 Apple GUID 作為事件 ID。已完成的資料列會留下約 4 小時的墓碑記錄，上限為 10,000 筆，因此即使重新啟動後，以相同 GUID 重播的項目仍會被捨棄。待處理資料列則會持續保持可復原狀態，直到分派流程接管為止。
- **停機復原。** 啟動時，監視器會記住最後一個已持久化准入之 `chat.db` 資料列的 rowid（每個帳號各自持久保存的游標），並將其以 `since_rowid` 傳給 `imsg watch.subscribe`，讓 imsg 重播尚未記錄的資料列，之後再接續追蹤即時資料。當機前已記錄的資料列會從 SQLite 恢復。重播僅限最近 500 個資料列，以及最長約 2 小時前的訊息；GUID 墓碑記錄則會捨棄任何已處理項目。
- **過期積壓的存留時間界線。** 高於啟動界線的資料列確實是即時資料；若其傳送日期比抵達時間早超過約 15 分鐘，則屬於 Push 一次排出的積壓資料，會受到抑制。重播資料列（位於界線或界線以下）則使用較寬的復原時間範圍，因此近期遺漏的訊息會送達，而年代久遠的歷程記錄不會送達。

復原功能同時適用於本機與遠端 `cliPath` 設定，因為 `since_rowid` 重播會透過同一個 `imsg` RPC 連線執行。兩者差異在於時間範圍：當閘道可讀取 `chat.db`（本機）時，會以此錨定啟動時的 rowid 界線、限制重播範圍，並送達最長數小時前遺漏的訊息。透過遠端 SSH `cliPath` 時，閘道無法讀取資料庫，因此重播範圍不受限制，且每個資料列都使用即時存留時間界線——它仍會復原近期遺漏的訊息並抑制舊積壓，只是採用較窄的即時時間範圍。若要使用較寬的復原時間範圍，請在 Messages Mac 上執行閘道。

### 操作者可見的訊號

受到抑制的積壓會以預設層級記錄，絕不會在未留下記錄的情況下捨棄（`recovery` 旗標會顯示套用的時間範圍）：

```text
imessage: suppressed stale inbound backlog account=<id> sent=<iso> recovery=<bool> (<N> suppressed since start)
```

### 移轉

`channels.imessage.catchup.*` 已棄用——停機復原會自動執行，新設定不需要任何組態。含有 `catchup.enabled: true` 的現有組態仍會作為復原重播時間範圍的相容性設定檔予以採用。已停用的追補區塊（`enabled: false` 或沒有 `enabled: true`）已停用；`openclaw doctor --fix` 會將其移除。

## 疑難排解

<AccordionGroup>
  <Accordion title="找不到 imsg 或不支援 RPC">
    驗證二進位檔及 RPC 支援：

    ```bash
    imsg rpc --help
    imsg status --json
    openclaw channels status --probe
    ```

    如果探測回報不支援 RPC，請更新 `imsg`。如果無法使用私有 API 動作，請在已登入的 macOS 使用者工作階段中執行 `imsg launch`，然後再次探測。如果閘道並非在 macOS 上執行，請改用上方的「透過 SSH 使用遠端 Mac」設定，而非預設的本機 `imsg` 路徑。

  </Accordion>

  <Accordion title="訊息可以傳送，但入站 iMessage 未送達">
    首先確認訊息是否已抵達本機 Mac。如果 `chat.db` 沒有變化，即使 `imsg status --json` 回報橋接器狀態正常，OpenClaw 也無法接收訊息。

```bash
imsg chats --limit 10 --json
imsg watch --chat-id <chat-id> --json
sqlite3 ~/Library/Messages/chat.db \
  "select datetime(max(date)/1000000000 + 978307200, 'unixepoch', 'localtime'), max(ROWID) from message;"
```

    如果從手機傳送的訊息未建立新資料列，請先修復 macOS Messages 與 Apple Push 層，再變更 OpenClaw 組態。通常一次性的服務重新整理便已足夠：

```bash
launchctl kickstart -k system/com.apple.apsd
launchctl kickstart -k gui/$(id -u)/com.apple.CommCenter
launchctl kickstart -k gui/$(id -u)/com.apple.identityservicesd
launchctl kickstart -k gui/$(id -u)/com.apple.imagent
imsg launch
openclaw gateway restart
```

    從手機傳送一則新的 iMessage，並在偵錯 OpenClaw 工作階段之前，確認出現新的 `chat.db` 資料列或 `imsg watch` 事件。請勿將此操作設為定期執行的橋接器重新啟動迴圈；在工作進行期間反覆執行 `imsg launch` 並重新啟動閘道，可能中斷傳遞並使進行中的頻道執行流程擱置。

  </Accordion>

  <Accordion title="閘道並非在 macOS 上執行">
    預設的 `cliPath: "imsg"` 必須在已登入 Messages 的 Mac 上執行。在 Linux 或 Windows 上，請將 `channels.imessage.cliPath` 設定為包裝函式指令碼，透過 SSH 連線至該 Mac 並執行 `imsg "$@"`。

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    然後執行：

```bash
openclaw channels status --probe --channel imessage
```

  </Accordion>

  <Accordion title="私訊遭到忽略">
    檢查：

    - `channels.imessage.dmPolicy`
    - `channels.imessage.allowFrom`
    - 配對核准（`openclaw pairing list imessage`）

  </Accordion>

  <Accordion title="群組訊息遭到忽略">
    檢查：

    - `channels.imessage.groupPolicy`
    - `channels.imessage.groupAllowFrom`
    - `channels.imessage.groups` 允許清單行為
    - 提及模式組態（`agents.entries.*.groupChat.mentionPatterns`）

  </Accordion>

  <Accordion title="遠端附件失敗">
    檢查：

    - `channels.imessage.remoteHost`
    - `channels.imessage.remoteAttachmentRoots`
    - 從閘道主機進行的 SSH/SCP 金鑰驗證
    - 閘道主機上的 `~/.ssh/known_hosts` 中存在主機金鑰
    - 執行 Messages 的 Mac 上可讀取遠端路徑

  </Accordion>

  <Accordion title="錯過 macOS 權限提示">
    請在相同使用者／工作階段情境中的互動式 GUI 終端機內重新執行，並核准提示：

    ```bash
    imsg chats --limit 1
    imsg send <handle> "test"
    ```

    確認執行 OpenClaw/`imsg` 的處理程序情境已獲授予「完全磁碟存取權」與「自動化」權限。

  </Accordion>
</AccordionGroup>

## 組態參考連結

- [組態參考資料 - iMessage](/zh-TW/gateway/config-channels#imessage)
- [閘道組態](/zh-TW/gateway/configuration)
- [配對](/zh-TW/channels/pairing)

## 相關內容

- [頻道概覽](/zh-TW/channels) — 所有支援的頻道
- [移除 BlueBubbles 與 imsg iMessage 路徑](/zh-TW/announcements/bluebubbles-imessage) — 公告與移轉摘要
- [從 BlueBubbles 移轉](/zh-TW/channels/imessage-from-bluebubbles) — 組態轉換表與逐步切換流程
- [配對](/zh-TW/channels/pairing) — 私訊驗證與配對流程
- [群組](/zh-TW/channels/groups) — 群組聊天行為與提及門控
- [頻道路由](/zh-TW/channels/channel-routing) — 訊息的工作階段路由
- [安全性](/zh-TW/gateway/security) — 存取模型與強化措施
