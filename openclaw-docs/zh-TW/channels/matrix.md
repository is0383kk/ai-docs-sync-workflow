---
read_when:
    - 在 OpenClaw 中設定 Matrix
    - 設定 Matrix 端對端加密與驗證
summary: Matrix 支援狀態、設定與配置範例
title: Matrix
x-i18n:
    generated_at: "2026-07-26T08:07:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa84c7d9d9019040a3fec3cfaabb78590006a4a2dd4bb95836f2cf37072777c5
    source_path: channels/matrix.md
    workflow: 16
---

Matrix 是基於官方 `matrix-js-sdk` 建構的可下載頻道外掛（`@openclaw/matrix`）。它支援私訊、聊天室、討論串、媒體、表情回應、投票、位置資訊及端對端加密。

## 安裝

```bash
openclaw plugins install @openclaw/matrix
```

未指定來源的外掛規格會先嘗試 ClawHub，再以 npm 作為備援。使用 `openclaw plugins install clawhub:@openclaw/matrix` 或 `npm:@openclaw/matrix` 強制指定來源。若從本機簽出版本安裝：`openclaw plugins install ./path/to/local/matrix-plugin`。

`plugins install` 會註冊並啟用此外掛；不需要額外執行 `enable` 步驟。在完成下方設定前，此頻道仍不會執行任何動作。一般安裝規則請參閱[外掛](/zh-TW/tools/plugin)。

## 設定

1. 在你的主伺服器上建立 Matrix 帳號。
2. 使用 `homeserver` + `accessToken`，或 `homeserver` + `userId` + `password` 設定 `channels.matrix`。
3. 重新啟動閘道。
4. 與機器人開始私訊，或邀請它加入聊天室。只有在 [`autoJoin`](#auto-join) 允許時，新的邀請才會生效。

### 互動式設定

```bash
openclaw channels add
openclaw configure --section channels
```

精靈會詢問主伺服器 URL、驗證方式（權杖或密碼）、使用者 ID（僅限密碼驗證）、選用的裝置名稱、是否啟用端對端加密，以及聊天室存取權與自動加入設定。若已存在相符的 `MATRIX_*` 環境變數，且帳號沒有已儲存的驗證資訊，精靈會提供環境變數捷徑。使用 `openclaw channels resolve --channel matrix "Project Room"` 儲存允許清單前，請先解析聊天室名稱。在精靈中啟用端對端加密時，會執行與 [`openclaw matrix encryption setup`](#encryption-and-verification) 相同的啟動程序。

### 最小設定

以權杖為基礎：

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

以密碼為基礎（首次登入後會快取權杖）：

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

### 自動加入

`channels.matrix.autoJoin` 預設為 `"off"`：在你手動加入之前，機器人不會因新的邀請而出現在新聊天室或私訊中。OpenClaw 無法在收到邀請時判斷它是私訊還是群組，因此每個邀請都會先經過 `autoJoin`；`dm.policy` 只會在機器人加入且聊天室完成分類後套用。

<Warning>
設定 `autoJoin: "allowlist"` 加上 `autoJoinAllowlist` 以限制接受的邀請，或設定 `autoJoin: "always"` 以接受所有邀請。

`autoJoinAllowlist` 僅接受 `!roomId:server`、`#alias:server` 或 `*`。一般聊天室名稱會遭拒絕；別名會根據主伺服器解析，而不是根據受邀聊天室宣告的狀態解析。
</Warning>

```json5
{
  channels: {
    matrix: {
      autoJoin: "allowlist",
      autoJoinAllowlist: ["!ops:example.org", "#support:example.org"],
      groups: {
        "!ops:example.org": { requireMention: true },
      },
    },
  },
}
```

### 允許清單目標格式

- 私訊（`dm.allowFrom`、`groupAllowFrom`、`groups.<room>.users`）：使用 `@user:server`。預設會忽略顯示名稱（可變動）；只有在明確需要相容顯示名稱時才設定 `dangerouslyAllowNameMatching: true`。
- 聊天室允許清單索引鍵（`groups`、舊版別名 `rooms`）：使用 `!room:server` 或 `#alias:server`。除非設定 `dangerouslyAllowNameMatching: true`，否則會忽略一般名稱。
- 邀請允許清單（`autoJoinAllowlist`）：使用 `!room:server`、`#alias:server` 或 `*`。一般名稱一律會遭拒絕。

### 帳號 ID 正規化

精靈會將易讀名稱轉換為正規化帳號 ID（`Ops Bot` -> `ops-bot`）。在限定範圍的環境變數名稱中，標點符號會以十六進位跳脫，避免帳號發生衝突：`-`（0x2D）會變成 `_X2D_`，因此 `ops-prod` 會對應至環境變數前綴 `MATRIX_OPS_X2D_PROD_`。

### 快取的認證資訊

Matrix 會將帳號認證資訊快取在共用的 `state/openclaw.sqlite` 外掛狀態中。若存在快取的認證資訊，即使設定檔中沒有 `accessToken`，OpenClaw 仍會將 Matrix 視為已設定；這涵蓋設定、`openclaw doctor` 及頻道狀態探查。升級時會透過 `openclaw doctor --fix` 匯入已淘汰的 `~/.openclaw/credentials/matrix/credentials*.json` 檔案、驗證 SQLite 資料列，然後封存這些檔案。

### 環境變數

當對應的設定鍵未設定時，會使用由設定鍵支援的環境變數。預設帳號使用不含前綴的名稱；具名帳號則會在後綴前插入帳號權杖（請參閱[正規化](#account-id-normalization)）。

| 預設帳號               | 具名帳號（`<ID>` = 帳號權杖） |
| --------------------- | -------------------------------------- |
| `MATRIX_HOMESERVER`   | `MATRIX_<ID>_HOMESERVER`               |
| `MATRIX_ACCESS_TOKEN` | `MATRIX_<ID>_ACCESS_TOKEN`             |
| `MATRIX_USER_ID`      | `MATRIX_<ID>_USER_ID`                  |
| `MATRIX_PASSWORD`     | `MATRIX_<ID>_PASSWORD`                 |
| `MATRIX_DEVICE_ID`    | `MATRIX_<ID>_DEVICE_ID`                |
| `MATRIX_DEVICE_NAME`  | `MATRIX_<ID>_DEVICE_NAME`              |

對於帳號 `ops`，名稱會變成 `MATRIX_OPS_HOMESERVER`、`MATRIX_OPS_ACCESS_TOKEN`，依此類推。`MATRIX_HOMESERVER`（以及任何限定 `*_HOMESERVER` 範圍的變體）無法從工作區 `.env` 設定；請參閱[工作區 `.env` 檔案](/zh-TW/gateway/security)。

<Note>
復原金鑰不是由設定支援的環境變數：OpenClaw 絕不會自行從環境中讀取它。命令列介面的引導文字建議，預設帳號可透過名為 `MATRIX_RECOVERY_KEY` 的 shell 變數以管線傳入，具名帳號則使用 `MATRIX_RECOVERY_KEY_<ID>`（僅將帳號 ID 轉為大寫，不進行十六進位跳脫）；請參閱[使用復原金鑰驗證此裝置](#verify-this-device-with-a-recovery-key)。
</Note>

## 設定範例

以下是包含私訊配對、聊天室允許清單及端對端加密的實用基準設定：

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": { requireMention: true },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: { mode: "partial" },
    },
  },
}
```

## 串流預覽

Matrix 回覆串流為選擇性啟用。`streaming.mode` 控制 OpenClaw 如何傳送產生中的助理回覆；`streaming.block.enabled` 控制是否將每個已完成區塊保留為獨立的 Matrix 訊息。

```json5
{
  channels: {
    matrix: {
      streaming: { mode: "partial" },
    },
  },
}
```

若要保留即時回答預覽，但隱藏暫時性的工具／進度行：

```json5
{
  channels: {
    matrix: {
      streaming: {
        mode: "partial",
        preview: {
          toolProgress: false,
        },
      },
    },
  },
}
```

完整設定接受 `{ mode, chunkMode, block, preview, progress }`：

```json5
{
  channels: {
    matrix: {
      streaming: {
        mode: "progress",
        progress: {
          label: "auto", // 從已設定或內建標籤選取（設為 false 可隱藏）
          labels: ["思考中", "撰寫中", "搜尋中"], // label: "auto" 的候選項目
          maxLines: 8, // 最多保留的滾動進度行數（預設：8）
          maxLineChars: 120, // 截斷前每行的字元數上限（預設：120）
          toolProgress: true, // 顯示工具／進度活動（預設：true）
        },
      },
    },
  },
}
```

- `progress.label`：自訂標籤；使用 `"auto"`／未設定可選取已設定或內建標籤，或使用 `false` 隱藏標籤。
- `progress.labels`：僅在 `label` 為 `"auto"` 或未設定時使用的候選項目。
- `progress.maxLines`：草稿中保留的滾動進度行數上限；超出後會移除較舊的行。
- `progress.maxLineChars`：每個精簡進度行在截斷前的字元數上限。
- `progress.toolProgress`：設為 `true`（預設）時，草稿中會顯示即時工具／進度活動。

| `streaming.mode`  | 行為                                                                                                                                                     |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `"off"`（預設） | 等待完整回覆後一次傳送。                                                                                                                                  |
| `"partial"`       | 模型撰寫目前區塊時，就地編輯一則一般文字訊息。原生用戶端可能會在第一次預覽時通知，而不是在最終編輯時通知。                                                    |
| `"quiet"`         | 與 `"partial"` 相同，但訊息是不發出通知的公告。當每位使用者的推播規則符合最終編輯內容時，收件者會收到一次通知（請參閱下方）。 |
| `"progress"`      | 使用進度草稿傳送個別的精簡進度行。                                                                                                                        |

`streaming.block.enabled`（預設為 `false`）與 `streaming.mode` 各自獨立：

| `streaming.mode`        | `block.enabled: true`                                               | `block.enabled: false`（預設）                     |
| ----------------------- | ------------------------------------------------------------------- | ---------------------------------------------------- |
| `"partial"` / `"quiet"` | 目前區塊使用即時草稿，已完成區塊則保留為訊息 | 目前區塊使用即時草稿，並就地完成定稿 |
| `"off"`                 | 每個完成區塊各傳送一則會發出通知的 Matrix 訊息                     | 整則回覆只傳送一則會發出通知的 Matrix 訊息      |

注意事項：

- 若預覽內容超過 Matrix 的單一事件大小限制，OpenClaw 會停止預覽串流，並改為只傳送最終內容。
- 媒體回覆一律會正常傳送附件；若無法安全地重複使用過時的預覽，OpenClaw 會先遮蔽該預覽，再傳送最終媒體回覆。
- 啟用預覽串流時，預設會開啟工具進度預覽更新。設定 `streaming.preview.toolProgress: false` 可保留回答文字的預覽編輯，但讓工具進度仍沿用一般傳送路徑。
- 預覽編輯會增加 Matrix API 呼叫次數。若要採用最保守的速率限制設定，請保留 `streaming.mode: "off"`。
- 舊版純量／布林值 `streaming`，以及扁平的 `blockStreaming`／`chunkMode` 鍵，會由 `openclaw doctor --fix` 改寫為這個巢狀結構。

## 語音訊息

傳入的 Matrix 語音訊息會在聊天室提及閘門之前轉錄，因此在 `requireMention: true` 聊天室中，說出機器人名稱的語音訊息可以觸發代理程式，而代理程式取得的是轉錄文字，不會只有音訊附件預留位置。

Matrix 使用 `tools.media.audio` 下的共用音訊媒體供應商，例如 OpenAI `gpt-4o-mini-transcribe`。供應商設定與限制請參閱[媒體工具概覽](/zh-TW/tools/media-overview)。

- `m.audio` 事件以及具有 `audio/*` MIME 類型的 `m.file` 事件符合資格。
- 在加密房間中，OpenClaw 會先透過現有的 Matrix 媒體路徑解密附件，再進行轉錄。
- 轉錄稿會在代理提示中標示為由機器產生且不受信任。
- 附件會標示為已轉錄，因此下游媒體工具不會再次轉錄。
- 將 `tools.media.audio.enabled: false` 設定為停用全域音訊轉錄。

## 核准中繼資料

Matrix 原生核准提示是一般的 `m.room.message` 事件，並在 `com.openclaw.approval` 鍵下包含 OpenClaw 專用內容。標準用戶端仍會呈現文字本文；支援 OpenClaw 的用戶端則可讀取結構化的核准 ID、類型、狀態、決策，以及執行／外掛詳細資料。

當提示過長，無法容納於單一 Matrix 事件時，OpenClaw 會將可見文字分塊，且僅在第一個區塊附加 `com.openclaw.approval`。允許／拒絕反應會繫結至該第一個事件，因此長提示與單一事件提示會維持相同的核准目標。

### 靜默最終預覽的自架推播規則

`streaming.mode: "quiet"` 僅會在區塊或回合最終確定時通知收件者——每位使用者的推播規則必須符合最終預覽標記。完整設定方式請參閱 [靜默預覽的 Matrix 推播規則](/zh-TW/channels/matrix-push-rules)。

## 機器人對機器人房間

依預設，來自其他已設定 OpenClaw Matrix 帳號的 Matrix 訊息會被忽略。使用 `allowBots` 可刻意允許代理間流量：

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- `allowBots: true` 會接受允許房間和私訊中，來自其他已設定 Matrix 機器人帳號的訊息。
- `allowBots: "mentions"` 僅會在房間中的訊息明確提及此機器人時接受這些訊息；無論是否提及，私訊仍會被接受。
- `groups.<room>.allowBots` 會覆寫單一房間的帳號層級設定。
- 已接受的設定機器人訊息會使用共用的[機器人迴圈防護](/zh-TW/channels/bot-loop-protection)。設定 `channels.defaults.botLoopProtection`，然後使用 `channels.matrix.botLoopProtection` 依帳號覆寫，或使用 `channels.matrix.groups.<room>.botLoopProtection` 依房間覆寫。
- OpenClaw 仍會忽略來自相同 Matrix 使用者 ID 的訊息，以避免自我回覆迴圈。
- Matrix 沒有原生機器人旗標；OpenClaw 將「由機器人撰寫」視為「由此 OpenClaw 閘道上另一個已設定的 Matrix 帳號傳送」。

在共用房間中啟用機器人對機器人流量時，請使用嚴格的房間允許清單和提及要求。

## 加密與驗證

在加密（E2EE）房間中，傳出的圖片事件使用 `thumbnail_file`，因此圖片預覽會與完整附件一併加密；未加密房間則使用純 `thumbnail_url`。不需要任何設定——外掛會自動偵測 E2EE 狀態。

所有 `openclaw matrix` 命令都接受 `--verbose`（完整診斷）、`--json`（機器可讀輸出）和 `--account <id>`（多帳號設定）。依預設，輸出會保持精簡。

### 啟用加密

```bash
openclaw matrix encryption setup
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix encryption setup --recovery-key-stdin
```

此命令會啟動秘密儲存空間與交叉簽署、視需要建立房間金鑰備份，然後列印狀態和後續步驟。實用旗標：

- `--recovery-key-stdin` 會從標準輸入讀取復原金鑰，而不在程序引數中暴露；`--recovery-key <key>` 仍保留以維持相容性
- `--force-reset-cross-signing` 捨棄目前的交叉簽署身分並建立新身分（僅限刻意使用）

若是新帳號，請在建立時啟用 E2EE：

```bash
openclaw matrix account add \
  --homeserver https://matrix.example.org \
  --access-token syt_xxx \
  --enable-e2ee
```

`--encryption` 是 `--enable-e2ee` 的別名。對應的手動設定：

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

### 狀態與信任訊號

```bash
openclaw matrix verify status
openclaw matrix verify status --include-recovery-key --json
```

`verify status` 會回報三個彼此獨立的信任訊號（`--verbose` 會顯示全部訊號）：

- `Locally trusted`：僅受此用戶端信任
- `Cross-signing verified`：SDK 回報已透過交叉簽署驗證
- `Signed by owner`：由你自己的自我簽署金鑰簽署（僅供診斷）

只有當 `Cross-signing verified` 為 `yes` 時，`Verified by owner` 才會是 `yes`；僅有本機信任或擁有者簽章並不足夠。

`--allow-degraded-local-state` 會在不先準備 Matrix 帳號的情況下回傳盡力而為的診斷資訊；適合離線或僅完成部分設定的探測。

### 使用復原金鑰驗證此裝置

請透過標準輸入傳入復原金鑰，而不要在命令列中傳遞：

```bash
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin
```

此命令會回報三種狀態：

- `Recovery key accepted`：Matrix 已接受此金鑰，用於秘密儲存空間或裝置信任。
- `Backup usable`：可使用受信任的復原資料載入房間金鑰備份。
- `Device verified by owner`：此裝置具有完整的 Matrix 交叉簽署身分信任。

即使復原金鑰已解鎖備份資料，只要完整身分信任尚未完成，此命令仍會以非零狀態結束。在這種情況下，請從另一個 Matrix 用戶端完成自我驗證：

```bash
openclaw matrix verify self
```

`verify self` 會等待 `Cross-signing verified: yes`，之後才成功結束。使用 `--timeout-ms <ms>` 調整等待時間。

也可使用字面金鑰形式 `openclaw matrix verify device "<recovery-key>"`，但金鑰會留在 Shell 歷史記錄中。

### 啟動或修復交叉簽署

```bash
openclaw matrix verify bootstrap
```

這是加密帳號的修復／設定命令。它會依序：

- 啟動秘密儲存空間，並盡可能重複使用現有的復原金鑰
- 啟動交叉簽署並上傳缺少的公開金鑰
- 標記目前裝置並進行交叉簽署
- 如果伺服器端房間金鑰備份尚不存在，則建立該備份

如果主伺服器要求使用 UIA 上傳交叉簽署金鑰，OpenClaw 會先嘗試不使用驗證，接著嘗試 `m.login.dummy`，最後嘗試 `m.login.password`（需要 `channels.matrix.password`）。

實用旗標：

- `--recovery-key-stdin`（與 `printf '%s\n' "$MATRIX_RECOVERY_KEY" | ...` 搭配）或 `--recovery-key <key>`
- `--force-reset-cross-signing`，用於捨棄目前的交叉簽署身分（僅限刻意使用；需要已儲存的有效復原金鑰，或透過 `--recovery-key-stdin` 提供）

### 房間金鑰備份

```bash
openclaw matrix verify backup status
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin
```

`backup status` 會顯示伺服器端備份是否存在，以及此裝置能否解密該備份。`backup restore` 會將已備份的房間金鑰匯入本機密碼編譯儲存區；如果復原金鑰已儲存在磁碟上，請省略 `--recovery-key-stdin`。

若要使用新的基準取代損壞的備份（接受遺失無法復原的舊歷史記錄；如果目前的備份秘密無法載入，也可以重新建立秘密儲存空間）：

```bash
openclaw matrix verify backup reset --yes
```

只有在刻意不希望先前的復原金鑰繼續解鎖新備份基準時，才加入 `--rotate-recovery-key`。

### 列出、要求及回應驗證

```bash
openclaw matrix verify list
```

列出所選帳號待處理的驗證要求。

```bash
openclaw matrix verify request --own-user
openclaw matrix verify request --user-id @ops:example.org --device-id ABCDEF
```

從此帳號傳送驗證要求。`--own-user` 會要求自我驗證（在同一使用者的另一個 Matrix 用戶端中接受提示）；`--user-id`／`--device-id`／`--room-id` 會指定其他人為目標。`--own-user` 無法與其他目標指定旗標組合使用。

針對較低階的生命週期處理——通常是在跟隨另一個用戶端傳入的要求時——這些命令會對特定要求 `<id>` 執行動作（由 `verify list` 和 `verify request` 列印）：

| 命令                                    | 用途                                                             |
| ------------------------------------------ | ------------------------------------------------------------------- |
| `openclaw matrix verify accept <id>`       | 接受傳入的要求                                           |
| `openclaw matrix verify start <id>`        | 啟動 SAS 流程                                                  |
| `openclaw matrix verify sas <id>`          | 列印 SAS 表情符號或十進位數字                                     |
| `openclaw matrix verify confirm-sas <id>`  | 確認 SAS 與另一個用戶端顯示的內容相符            |
| `openclaw matrix verify mismatch-sas <id>` | 當表情符號或十進位數字不相符時拒絕 SAS              |
| `openclaw matrix verify cancel <id>`       | 取消；接受選用的 `--reason <text>` 和 `--code <matrix-code>` |

當驗證繫結至特定私訊房間時，`accept`、`start`、`sas`、`confirm-sas`、`mismatch-sas` 和 `cancel` 都接受 `--user-id` 和 `--room-id` 作為私訊後續提示。

### 多帳號注意事項

若未使用 `--account <id>`，Matrix 命令列介面命令會使用隱含的預設帳號。如果有多個具名帳號但未指定 `channels.matrix.defaultAccount`，命令會拒絕猜測並要求你選擇。當具名帳號停用或無法使用 E2EE 時，錯誤會指出該帳號的設定鍵，例如 `channels.matrix.accounts.assistant.encryption`。

<AccordionGroup>
  <Accordion title="啟動行為">
    使用 `encryption: true` 時，`startupVerification` 預設為 `"if-unverified"`。啟動時，未驗證的裝置會在另一個 Matrix 用戶端中要求自我驗證、略過重複要求，並套用冷卻時間（預設為 24 小時）。使用 `startupVerificationCooldownHours` 調整，或使用 `startupVerification: "off"` 停用。

    啟動程序也會執行保守的密碼編譯啟動流程，重複使用目前的秘密儲存空間和交叉簽署身分。如果啟動狀態損壞，即使沒有 `channels.matrix.password`，OpenClaw 仍會嘗試受防護的修復；如果主伺服器要求密碼 UIA，啟動程序會記錄警告並維持非致命狀態。已由擁有者簽署的裝置會予以保留。

    完整升級流程請參閱 [Matrix 遷移](/zh-TW/channels/matrix-migration)。

  </Accordion>

  <Accordion title="驗證通知">
    Matrix 會將驗證生命週期通知以 `m.notice` 訊息的形式張貼至嚴格的私訊驗證房間：要求、就緒（包含「使用表情符號驗證」指引）、開始／完成，以及可用時的 SAS（表情符號／十進位）詳細資料。

    來自另一個 Matrix 用戶端的傳入要求會受到追蹤並自動接受。對於自我驗證，OpenClaw 會自動啟動 SAS 流程，並在表情符號驗證可用後確認自身端——你仍需在 Matrix 用戶端中進行比較並確認 “They match”。

    驗證系統通知不會轉送至代理聊天流水線。

  </Accordion>

  <Accordion title="已刪除或無效的 Matrix 裝置">
    如果 `verify status` 表示目前裝置已不再列於主伺服器上，請建立新的 OpenClaw Matrix 裝置。若使用密碼登入：

```bash
openclaw matrix account add \
  --account assistant \
  --homeserver https://matrix.example.org \
  --user-id '@assistant:example.org' \
  --password '<password>' \
  --device-name OpenClaw-Gateway
```

    若使用權杖驗證，請在 Matrix 用戶端或管理員 UI 中建立新的存取權杖，然後更新 OpenClaw：

```bash
openclaw matrix account add \
  --account assistant \
  --homeserver https://matrix.example.org \
  --access-token '<token>'
```

    將 `assistant` 替換為失敗命令中的帳戶 ID，或省略 `--account` 以使用預設帳戶。

  </Accordion>

  <Accordion title="裝置管理">
    舊的 OpenClaw 管理裝置可能會不斷累積。列出並清除：

```bash
openclaw matrix devices list
openclaw matrix devices prune-stale
```

  </Accordion>

  <Accordion title="加密儲存區">
    Matrix E2EE 使用官方 `matrix-js-sdk` Rust 加密路徑，並以 `fake-indexeddb` 作為 IndexedDB 墊片。加密狀態會持久儲存至 `crypto-idb-snapshot.json`（採用嚴格的檔案權限）。

    加密的執行階段狀態位於 `~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/` 下方，包括同步儲存區、加密儲存區、復原金鑰、IDB 快照、討論串繫結及啟動驗證狀態。當權杖變更但帳戶身分維持不變時，OpenClaw 會重複使用最佳的現有根目錄，讓先前的狀態仍然可見。

    單一較舊的權杖雜湊根目錄可能是正常的權杖輪替連續性路徑。如果 OpenClaw 記錄 `matrix: multiple populated token-hash storage roots detected`，請檢查帳戶目錄，並僅在確認選定的作用中根目錄運作正常後，才封存過時的同層根目錄。建議先將過時的根目錄移至 `_archive/` 目錄，而非立即刪除。

  </Accordion>
</AccordionGroup>

## 個人檔案管理

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

請在一次呼叫中同時傳入兩個選項。Matrix 可直接接受 `mxc://` 頭像 URL；傳入 `http://`/`https://` 時，會先上傳檔案，並將解析後的 `mxc://` URL 儲存至 `channels.matrix.avatarUrl`（或各帳戶的覆寫設定）。

## 討論串

Matrix 的自動回覆與訊息工具傳送均支援原生討論串。兩個獨立設定控制其行為：

### 工作階段路由（`sessionScope`）

`dm.sessionScope` 決定 Matrix 私訊聊天室如何對應至 OpenClaw 工作階段：

- `"per-user"`（預設）：路由至同一對象的所有私訊聊天室共用一個工作階段。
- `"per-room"`：每個 Matrix 私訊聊天室都有自己的工作階段金鑰，即使對象相同亦然。

明確的對話繫結一律優先於 `sessionScope`；已繫結的聊天室與討論串會保留其選定的目標工作階段。

### 回覆討論串（`threadReplies`）

`threadReplies` 決定機器人要將回覆發布至何處：

- `"off"`：回覆位於頂層。傳入的討論串訊息會留在父工作階段。
- `"inbound"`：僅當傳入訊息已位於該討論串時，才在討論串中回覆。
- `"always"`：在以觸發訊息為根的討論串內回覆；從首次觸發開始，該對話會透過相符的討論串範圍工作階段進行路由。

`dm.threadReplies` 僅針對私訊覆寫此設定，例如讓聊天室討論串保持隔離，同時讓私訊維持非討論串模式。

### 討論串繼承與斜線命令

- 傳入的討論串訊息會包含討論串根訊息，作為額外的代理程式情境。
- 訊息工具傳送至同一聊天室（或同一私訊使用者目標）時，會自動繼承目前的 Matrix 討論串，除非明確提供 `threadId`。
- 只有目前的工作階段中繼資料能證明是同一 Matrix 帳戶上的同一私訊對象時，才會重複使用私訊使用者目標；否則 OpenClaw 會退回一般的使用者範圍路由。
- `/focus`、`/unfocus`、`/agents`、`/session idle`、`/session max-age`，以及討論串繫結的 `/acp spawn`，均可在 Matrix 聊天室與私訊中使用。
- 啟用 `threadBindings.spawnSessions` 時，頂層 `/focus` 會建立新的 Matrix 討論串，並將其繫結至目標工作階段。
- 在現有 Matrix 討論串內執行 `/focus` 或 `/acp spawn --thread here`，會直接繫結該討論串。

當 OpenClaw 偵測到 Matrix 私訊聊天室與同一共用工作階段上的另一個私訊聊天室衝突時，會發布一次性的 `m.notice`，指向 `/focus` 應變方式並建議變更 `dm.sessionScope`。此通知僅在啟用討論串繫結時出現。

## ACP 對話繫結

Matrix 聊天室、私訊及現有 Matrix 討論串可以成為持久的 ACP 工作區，而不需變更聊天介面。

快速操作流程：

- 在要繼續使用的 Matrix 私訊、聊天室或現有討論串內執行 `/acp spawn codex --bind here`。
- 在頂層私訊或聊天室中，目前的私訊／聊天室會維持為聊天介面，後續訊息則路由至產生的 ACP 工作階段。
- 在現有討論串內，`--bind here` 會直接繫結目前的討論串。
- `/new` 和 `/reset` 會就地重設同一個已繫結的 ACP 工作階段。
- `/acp close` 會關閉 ACP 工作階段並移除繫結。

`--bind here` 不會建立子 Matrix 討論串。`threadBindings.spawnSessions` 會控管 `/acp spawn --thread auto|here`，後者需要 OpenClaw 建立或繫結子討論串。

### 討論串繫結設定

Matrix 會繼承 `session.threadBindings` 的全域預設值，並支援各頻道的覆寫設定：

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSessions`：同時控管子代理程式與 ACP 討論串的產生。
- 已淘汰的 `threadBindings.spawnSubagentSessions` / `threadBindings.spawnAcpSessions` 鍵會由 `openclaw doctor --fix` 遷移至 `spawnSessions`。
- `threadBindings.defaultSpawnContext`

Matrix 討論串繫結的工作階段產生功能預設為開啟。設定 `threadBindings.spawnSessions: false`，可阻止頂層 `/focus` 與 `/acp spawn --thread auto|here` 建立／繫結 Matrix 討論串。若不應讓原生子代理程式討論串的產生分支父逐字稿，請設定 `threadBindings.defaultSpawnContext: "isolated"`。

## 表情回應

Matrix 支援傳出表情回應、傳入表情回應通知及確認表情回應。

傳出表情回應工具由 `channels.matrix.actions.reactions` 控管：

- `react` 會將表情回應新增至 Matrix 事件。
- `reactions` 會列出 Matrix 事件目前的表情回應摘要。
- `emoji=""` 會移除機器人在該事件上自己的表情回應。
- `remove: true` 僅會移除機器人的指定表情符號回應。

**解析順序**（第一個已定義的值優先）：

| 設定                 | 順序                                                                               |
| ----------------------- | ----------------------------------------------------------------------------------- |
| `ackReaction`           | 各帳戶 -> 頻道 -> `messages.ackReaction` -> 代理程式身分表情符號備援   |
| `ackReactionScope`      | 各帳戶 -> 頻道 -> `messages.ackReactionScope` -> 預設 `"group-mentions"` |
| `reactionNotifications` | 各帳戶 -> 頻道 -> 預設 `"own"`                                           |

當新增的 `m.reaction` 事件以機器人撰寫的 Matrix 訊息為目標時，`reactionNotifications: "own"` 會轉送這些事件；`"off"` 會停用表情回應系統事件。移除表情回應不會合成為系統事件，因為 Matrix 會將其呈現為遮蓋，而非獨立的 `m.reaction` 移除事件。

## 歷史情境

- `channels.matrix.historyLimit` 控制聊天室訊息觸發代理程式時，要將多少則近期聊天室訊息納入 `InboundHistory`。備援至 `messages.groupChat.historyLimit`；若兩者皆未設定，實際預設值為 `0`（停用）。
- Matrix 聊天室歷史僅限聊天室；私訊仍使用一般工作階段歷史。
- 聊天室歷史僅包含待處理訊息：OpenClaw 會緩衝尚未觸發回覆的聊天室訊息，接著在提及或其他觸發條件出現時擷取該視窗的快照。
- 目前的觸發訊息不會納入 `InboundHistory`；該訊息會留在該回合的主要傳入本文中。
- 重試同一個 Matrix 事件時，會重複使用原始歷史快照，而不會向前漂移至較新的聊天室訊息。

## 情境可見性

Matrix 支援共用的 `contextVisibility` 控制項，用於擷取的回覆文字、討論串根訊息及待處理歷史等補充聊天室情境。

- `contextVisibility: "all"` 為預設值。補充情境會依接收時的內容保留。
- `contextVisibility: "allowlist"` 會篩選補充情境，只保留通過目前聊天室／使用者允許清單檢查的傳送者。
- `contextVisibility: "allowlist_quote"` 的行為與 `allowlist` 相同，但仍會保留一則明確引用的回覆。

這只影響補充情境的可見性，不影響傳入訊息本身能否觸發回覆。觸發授權仍由 `groupPolicy`、`groups`、`groupAllowFrom` 及私訊原則設定決定。

## 私訊與聊天室原則

```json5
{
  channels: {
    matrix: {
      dm: {
        policy: "allowlist",
        allowFrom: ["@admin:example.org"],
        threadReplies: "off",
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": { requireMention: true },
      },
    },
  },
}
```

若要完全靜音私訊但保持聊天室運作，請設定 `dm.enabled: false`：

```json5
{
  channels: {
    matrix: {
      dm: { enabled: false },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
    },
  },
}
```

提及控管與允許清單行為請參閱[群組](/zh-TW/channels/groups)。

Matrix 私訊的配對範例：

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

如果未核准的 Matrix 使用者在核准前持續傳送訊息，OpenClaw 會重複使用相同的待處理配對碼，並可能在短暫冷卻時間後傳送提醒回覆，而非產生新的配對碼。

共用私訊配對流程與儲存配置請參閱[配對](/zh-TW/channels/pairing)。

## 直接聊天室修復

如果直接訊息狀態發生偏移，OpenClaw 最終可能會有過時的 `m.direct` 對應，指向舊的單人聊天室，而非目前的私訊。檢查某個對象目前的對應：

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

進行修復：

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

兩個命令都接受 `--account <id>`，以用於多帳戶設定。修復流程：

- 優先使用已在 `m.direct` 中對應的嚴格 1:1 私訊
- 若無，則使用目前已加入且包含該使用者的任何嚴格 1:1 私訊
- 若不存在運作正常的私訊，則建立新的直接聊天室並重寫 `m.direct`

此流程不會自動刪除舊聊天室。它會選取運作正常的私訊並更新對應，讓後續 Matrix 傳送、驗證通知及其他直接訊息流程以正確的聊天室為目標。

## 執行核准

Matrix 可作為原生核准用戶端。請在 `channels.matrix.execApprovals` 下設定（或使用 `channels.matrix.accounts.<account>.execApprovals` 作為各帳戶覆寫設定）：

- `enabled`：透過 Matrix 原生提示傳遞核准要求。未設定或設為 `"auto"` 時，只要能解析出至少一名核准者，就會自動啟用；設為 `false` 可明確停用。
- `approvers`：允許核准執行要求的 Matrix 使用者 ID（`@owner:example.org`）。備援至 `channels.matrix.dm.allowFrom`。
- `target`：提示的傳送位置。`"dm"`（預設）會傳送至核准者的私訊；`"channel"` 會傳送至要求來源的聊天室或私訊；`"both"` 則會傳送至兩者。
- `agentFilter` / `sessionFilter`：選用的允許清單，用於指定哪些代理程式／工作階段會觸發 Matrix 傳遞。

不同核准類型的授權方式略有差異：

- **執行核准**使用 `execApprovals.approvers`，並以 `dm.allowFrom` 作為備援。
- **外掛核准**僅透過 `dm.allowFrom` 授權。

這兩種核准共用 Matrix 回應捷徑與訊息更新。核准者會在主要核准訊息上看到回應捷徑：

- ✅ 允許一次
- ❌ 拒絕
- ♾️ 一律允許（當有效的執行政策允許時）

備援斜線指令：`/approve <id> allow-once`、`/approve <id> allow-always`、`/approve <id> deny`。

只有已解析的核准者可以核准或拒絕。執行核准的頻道傳遞內容包含指令文字——請只在受信任的聊天室中啟用 `channel` 或 `both`。

相關內容：[執行核准](/zh-TW/tools/exec-approvals)。

## 斜線指令

斜線指令（`/new`、`/reset`、`/model`、`/focus`、`/unfocus`、`/agents`、`/session`、`/acp`、`/approve` 等）可直接在私訊中使用。在聊天室中，OpenClaw 也能辨識以機器人自身 Matrix 提及為前綴的指令，因此 `@bot:server /new` 無須自訂提及正規表示式即可觸發指令路徑——如此一來，當使用者在輸入指令前先用 Tab 鍵補全機器人名稱時，機器人仍能回應 Element 與類似用戶端送出的聊天室格式 `@mention /command` 貼文。

授權規則仍然適用：指令傳送者必須符合與一般訊息相同的私訊或聊天室允許清單／擁有者政策。

## 多帳號

```json5
{
  channels: {
    matrix: {
      enabled: true,
      defaultAccount: "assistant",
      dm: { policy: "pairing" },
      accounts: {
        assistant: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_assistant_xxx",
          encryption: true,
        },
        alerts: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_alerts_xxx",
          dm: {
            policy: "allowlist",
            allowFrom: ["@ops:example.org"],
            threadReplies: "off",
          },
        },
      },
    },
  },
}
```

**繼承：**

- 除非帳號覆寫，否則頂層 `channels.matrix` 值會作為具名帳號的預設值。
- 使用 `groups.<room>.account` 將繼承的聊天室項目限定於特定帳號。沒有 `account` 的項目會由各帳號共用；當預設帳號設定於頂層時，`account: "default"` 仍可正常運作。

**預設帳號選擇：**

- 設定 `defaultAccount`，以選擇隱含路由、探測與命令列介面指令優先使用的具名帳號。
- 如果你有多個帳號，且其中一個帳號的名稱就是 `default`，即使未設定 `defaultAccount`，OpenClaw 也會隱含使用該帳號。
- 若有多個具名帳號且未選擇預設帳號，命令列介面指令會拒絕猜測——請設定 `defaultAccount` 或傳入 `--account <id>`。
- 只有在頂層 `channels.matrix.*` 區塊的驗證資訊完整時（`homeserver` + `accessToken`，或 `homeserver` + `userId` + `password`），才會將其視為隱含的 `default` 帳號。快取的認證資訊足以完成驗證後，仍可從 `homeserver` + `userId` 探索具名帳號。

**提升：**

- 當 OpenClaw 在修復或設定期間將單帳號設定提升為多帳號設定時，如果已有具名帳號，或 `defaultAccount` 已指向某個帳號，便會保留現有具名帳號。只有 Matrix 驗證／啟動鍵會移入提升後的帳號；共用的傳遞政策鍵會保留在頂層。

共用的多帳號模式請參閱[設定參考](/zh-TW/gateway/config-channels#multi-account-all-channels)。

## 私有／區域網路家伺服器

為防範 SSRF，OpenClaw 預設會封鎖私有／內部 Matrix 家伺服器，除非你針對各帳號選擇加入。

如果你的家伺服器在 localhost、區域網路／Tailscale IP 或內部主機名稱上執行，請為該帳號啟用 `network.dangerouslyAllowPrivateNetwork`：

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
      accessToken: "syt_internal_xxx",
    },
  },
}
```

命令列介面設定範例：

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

此選擇加入僅允許受信任的私有／內部目標。`http://matrix.example.org:8008` 等公開明文家伺服器仍會遭到封鎖。請盡可能優先使用 `https://`。

## 代理 Matrix 流量

如果你的 Matrix 部署需要明確的對外 HTTP(S) Proxy，請設定 `channels.matrix.proxy`：

```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
    },
  },
}
```

具名帳號可以使用 `channels.matrix.accounts.<id>.proxy` 覆寫頂層預設值。OpenClaw 會對執行階段 Matrix 流量與帳號狀態探測使用相同的 Proxy 設定。

## 目標解析

凡是 OpenClaw 要求提供聊天室或使用者目標之處，Matrix 都接受以下目標格式：

- 使用者：`@user:server`、`user:@user:server` 或 `matrix:user:@user:server`
- 聊天室：`!room:server`、`room:!room:server` 或 `matrix:room:!room:server`
- 別名：`#alias:server`、`channel:#alias:server` 或 `matrix:channel:#alias:server`

Matrix 聊天室 ID 區分大小寫。設定明確傳遞目標、排程工作、繫結或允許清單時，請使用 Matrix 中聊天室 ID 的確切大小寫。OpenClaw 會維持內部工作階段鍵的標準化形式以供儲存，因此這些小寫鍵並非 Matrix 傳遞 ID 的可靠來源。

即時目錄查詢會使用已登入的 Matrix 帳號：

- 使用者查詢會查詢該家伺服器上的 Matrix 使用者目錄。
- 聊天室查詢會直接接受明確的聊天室 ID 與別名。已加入聊天室的名稱查詢採盡力而為方式，且僅在設定 `dangerouslyAllowNameMatching: true` 時套用於執行階段聊天室允許清單。
- 如果無法將聊天室名稱解析為 ID 或別名，執行階段允許清單解析便會忽略該名稱。

## 設定參考

允許清單型使用者欄位（`groupAllowFrom`、`dm.allowFrom`、`groups.<room>.users`）接受完整的 Matrix 使用者 ID（最安全）。預設會忽略非 ID 項目。如果設定 `dangerouslyAllowNameMatching: true`，系統會在啟動時，以及監視器執行期間允許清單每次變更時，解析 Matrix 目錄中完全相符的顯示名稱；執行階段會忽略無法解析的項目。

聊天室允許清單鍵（`groups`、舊版 `rooms`）應為聊天室 ID 或別名。預設會忽略純聊天室名稱鍵；`dangerouslyAllowNameMatching: true` 會恢復對已加入聊天室名稱的盡力而為查詢。

### 帳號與連線

- `enabled`：啟用或停用頻道。
- `name`：帳號的選用顯示標籤。
- `defaultAccount`：設定多個 Matrix 帳號時偏好的帳號 ID。
- `accounts`：具名的各帳號覆寫值。頂層 `channels.matrix` 值會繼承為預設值。
- `homeserver`：家伺服器 URL，例如 `https://matrix.example.org`。
- `network.dangerouslyAllowPrivateNetwork`：允許此帳號連線至 `localhost`、區域網路／Tailscale IP 或內部主機名稱。
- `proxy`：Matrix 流量的選用 HTTP(S) Proxy URL。支援各帳號覆寫。
- `userId`：完整的 Matrix 使用者 ID（`@bot:example.org`）。
- `accessToken`：權杖式驗證的存取權杖。環境變數／檔案／執行供應者皆支援純文字與 SecretRef 值（[密鑰管理](/zh-TW/gateway/secrets)）。
- `password`：密碼式登入的密碼。支援純文字與 SecretRef 值。
- `deviceId`：明確的 Matrix 裝置 ID。
- `deviceName`：密碼登入時使用的裝置顯示名稱。
- `avatarUrl`：用於個人資料同步與 `profile set` 更新的已儲存個人頭像 URL。
- `initialSyncLimit`：啟動同步期間擷取的事件數量上限。

### 加密

- `encryption`：啟用 E2EE。預設值：`false`。
- `startupVerification`：`"if-unverified"`（啟用 E2EE 時的預設值）或 `"off"`。當此裝置尚未驗證時，會在啟動時自動要求自我驗證。
- `startupVerificationCooldownHours`：下次自動啟動要求前的冷卻時間。預設值：`24`。

### 存取與政策

- `groupPolicy`：`"open"`、`"allowlist"` 或 `"disabled"`。預設值：`"allowlist"`。
- `groupAllowFrom`：聊天室流量的使用者 ID 允許清單。
- `mentionPatterns`：聊天室提及的限定範圍正規表示式模式。包含 `{ mode: "allow"|"deny", allowIn: [roomId, ...], denyIn: [roomId, ...] }` 的物件。控制設定的 `agents.entries.*.groupChat.mentionPatterns` 是否依聊天室套用。
- `dm.enabled`：為 `false` 時，忽略所有私訊。預設值：`true`。
- `dm.policy`：`"pairing"`（預設值）、`"allowlist"`、`"open"` 或 `"disabled"`。在機器人加入聊天室並將其分類為私訊後套用；不影響邀請處理。
- `dm.allowFrom`：私訊流量的使用者 ID 允許清單。
- `dm.sessionScope`：`"per-user"`（預設值）或 `"per-room"`。
- `dm.threadReplies`：僅限私訊的回覆串覆寫值（`"off"`、`"inbound"`、`"always"`）。
- `allowBots`：接受來自其他已設定 Matrix 機器人帳號的訊息（`true` 或 `"mentions"`）。
- `allowlistOnly`：為 `true` 時，會強制將所有有效的私訊政策（`"disabled"` 除外）與 `"open"` 群組政策設為 `"allowlist"`。不會變更 `"disabled"` 政策。
- `dangerouslyAllowNameMatching`：為 `true` 時，允許針對使用者允許清單項目進行 Matrix 顯示名稱目錄查詢，並針對聊天室允許清單鍵進行已加入聊天室名稱查詢。請優先使用完整的 `@user:server` ID，以及聊天室 ID 或別名。
- `autoJoin`：`"always"`、`"allowlist"` 或 `"off"`。預設值：`"off"`。套用於每一則 Matrix 邀請，包括私訊形式的邀請。
- `autoJoinAllowlist`：當 `autoJoin` 為 `"allowlist"` 時允許的聊天室／別名。別名項目會根據家伺服器解析，而非根據受邀聊天室宣告的狀態解析。
- `contextVisibility`：補充內容可見性（`"all"` 為預設值、`"allowlist"`、`"allowlist_quote"`）。

### 回覆行為

- `replyToMode`: `"off"`（預設）、`"first"`、`"all"` 或 `"batched"`。
- `threadReplies`: `"off"`（除非明確設定，否則頂層預設值會解析為 `"inbound"`）、`"inbound"` 或 `"always"`。
- `threadBindings`: 用於執行緒綁定工作階段路由與生命週期的各頻道覆寫設定。
- `streaming`: 巢狀物件 `{ mode, chunkMode, block: { enabled, coalesce }, preview: { toolProgress }, progress: { label, labels, maxLines, maxLineChars, toolProgress } }`。`mode` 可為 `"off"`（預設）、`"partial"`、`"quiet"` 或 `"progress"`。舊版純量／布林值寫法會透過 `openclaw doctor --fix` 遷移。
- `streaming.block.enabled`: 當 `true` 時，已完成的助理區塊會保留為個別的進度訊息。預設值：`false`。
- `markdown`: 選用的外送文字 Markdown 轉譯設定。
- `responsePrefix`: 選用的字串，會加在外送回覆前面。
- `textChunkLimit`: 當 `streaming.chunkMode: "length"` 時，外送區塊的字元數大小。預設值：`4000`。
- `streaming.chunkMode`: `"length"`（預設，依字元數分割）或 `"newline"`（於行邊界分割）。
- `historyLimit`: 當聊天室訊息觸發代理程式時，以 `InboundHistory` 納入的近期聊天室訊息數量。若未設定，則使用 `messages.groupChat.historyLimit`；實際預設值為 `0`（停用）。
- `mediaMaxMb`: 外送傳送與接收處理的媒體大小上限，單位為 MB。預設值：`20`。

### 回應設定

- `ackReaction`: 此頻道／帳號的確認回應覆寫設定。
- `ackReactionScope`: 範圍覆寫設定（預設為 `"group-mentions"`，亦可為 `"group-all"`、`"direct"`、`"all"`、`"none"`、`"off"`）。
- `reactionNotifications`: 接收回應通知模式（預設為 `"own"`，亦可為 `"off"`）。

### 工具與各聊天室覆寫設定

- `actions`: 各動作的工具管控（`messages`、`reactions`、`pins`、`profile`、`memberInfo`、`channelInfo`、`verification`）。
- `groups`: 各聊天室的原則對應表。解析後，工作階段識別會使用穩定的聊天室 ID。（`rooms` 是舊版別名。）
  - `groups.<room>.account`: 將一個繼承的聊天室項目限制為特定帳號。
  - `groups.<room>.enabled`: 各聊天室切換設定。當 `false` 時，該聊天室會被忽略，如同未列於對應表中。
  - `groups.<room>.requireMention`: 各聊天室對頻道層級提及要求的覆寫設定。
  - `groups.<room>.allowBots`: 各聊天室對頻道層級設定的覆寫（`true` 或 `"mentions"`）。
  - `groups.<room>.botLoopProtection`: 各聊天室對機器人之間迴圈防護預算的覆寫設定。
  - `groups.<room>.users`: 各聊天室的傳送者允許清單。
  - `groups.<room>.tools`: 各聊天室的工具允許／拒絕覆寫設定。
  - `groups.<room>.autoReply`: 各聊天室的提及管控覆寫設定。`true` 會停用該聊天室的提及要求；`false` 則會強制重新啟用。
  - `groups.<room>.skills`: 各聊天室的 Skills 篩選器。
  - `groups.<room>.systemPrompt`: 各聊天室的系統提示詞片段。

### Exec 核准設定

- `execApprovals.enabled`: 透過 Matrix 原生提示傳送 exec 核准請求。
- `execApprovals.approvers`: 允許核准的 Matrix 使用者 ID。若未設定，則使用 `dm.allowFrom`。
- `execApprovals.target`: `"dm"`（預設）、`"channel"` 或 `"both"`。
- `execApprovals.agentFilter` / `execApprovals.sessionFilter`: 選用的代理程式／工作階段傳送允許清單。

## 相關內容

- [頻道概覽](/zh-TW/channels) - 所有支援的頻道
- [配對](/zh-TW/channels/pairing) - 私訊驗證與配對流程
- [群組](/zh-TW/channels/groups) - 群組聊天行為與提及管控
- [頻道路由](/zh-TW/channels/channel-routing) - 訊息的工作階段路由
- [安全性](/zh-TW/gateway/security) - 存取模型與強化措施
