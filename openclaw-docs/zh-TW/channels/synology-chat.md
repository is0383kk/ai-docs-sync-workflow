---
read_when:
    - 使用 OpenClaw 設定 Synology Chat
    - 偵錯 Synology Chat 網路鉤子路由
summary: Synology Chat 網路鉤子設定與 OpenClaw 設定
title: Synology Chat
x-i18n:
    generated_at: "2026-07-26T07:44:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3c03379944ee4187260a7287f6d2aed1ad8fdd1c22b5581c8a5d55515bbb6ad5
    source_path: channels/synology-chat.md
    workflow: 16
---

Synology Chat 透過一對網路鉤子連接 OpenClaw：Synology Chat 傳出網路鉤子會將收到的私訊發佈至閘道，回覆則透過 Synology Chat 傳入網路鉤子送回。

狀態：官方外掛，需另行安裝。僅支援私訊；支援文字及以 URL 為基礎的檔案傳送。

## 安裝

```bash
openclaw plugins install @openclaw/synology-chat
```

本機簽出（從 git 儲存庫執行時）：

```bash
openclaw plugins install ./path/to/local/synology-chat-plugin
```

詳細資訊：[外掛](/zh-TW/tools/plugin)

## 快速設定

1. 安裝外掛（如上所述）。
2. 在 Synology Chat 整合中：
   - 建立傳入網路鉤子並複製其 URL。
   - 使用你的秘密權杖建立傳出網路鉤子。
3. 將傳出網路鉤子 URL 指向你的 OpenClaw 閘道：
   - 預設為 `https://gateway-host/webhook/synology`。
   - 或你的自訂 `channels.synology-chat.webhookPath`。
4. 在 OpenClaw 中完成設定。Synology Chat 會出現在兩種流程的同一個頻道設定清單中：
   - 引導式：`openclaw onboard` 或 `openclaw channels add`
   - 直接：`openclaw channels add --channel synology-chat --token <token> --url <incoming-webhook-url>`
5. 重新啟動閘道，並向 Synology Chat 機器人傳送私訊。

網路鉤子驗證詳細資訊：

- OpenClaw 依序從 `body.token`、`?token=...`，再從標頭接受傳出網路鉤子權杖。
- 接受的標頭格式：
  - `x-synology-token`
  - `x-webhook-token`
  - `x-openclaw-token`
  - `Authorization: Bearer <token>`
- 權杖為空或缺少時，系統會採取封閉式失敗。
- 承載資料可以是 `application/x-www-form-urlencoded` 或 `application/json`；`token`、`user_id` 和 `text` 為必要欄位。

## 傳入持久性

權杖、傳送者政策和速率限制檢查通過後，OpenClaw 會從儲存的信封中移除網路鉤子權杖，並在確認事件前將其可靠地排入佇列。只有在附加成功後，路由才會傳回 `204`；持久化失敗會傳回 `503`，讓 Synology Chat 能夠重試，而不會無聲地遺失訊息。

待處理或可重試的事件會在閘道重新啟動後保留。當對應的作用中或保留完成記錄存在時，Synology 的穩定 `post_id` 會抑制重複的佇列項目。從佇列交接至代理程式的傳遞仍保證至少一次，因此在此邊界發生當機時，仍可能重播一個回合。

最小設定：

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      token: "synology-outgoing-token",
      incomingUrl: "https://nas.example.com/webapi/entry.cgi?api=SYNO.Chat.External&method=incoming&version=2&token=...",
      webhookPath: "/webhook/synology",
      dmPolicy: "allowlist",
      allowedUserIds: ["123456"],
      rateLimitPerMinute: 30,
      allowInsecureSsl: false,
    },
  },
}
```

## 環境變數

對於預設帳號，你可以使用環境變數：

- `SYNOLOGY_CHAT_TOKEN`
- `SYNOLOGY_CHAT_INCOMING_URL`
- `SYNOLOGY_NAS_HOST`
- `SYNOLOGY_ALLOWED_USER_IDS`（以逗號分隔）
- `SYNOLOGY_RATE_LIMIT`
- `OPENCLAW_BOT_NAME`

設定值會覆寫環境變數。

`SYNOLOGY_CHAT_INCOMING_URL` 和 `SYNOLOGY_NAS_HOST` 無法透過工作區的 `.env` 設定；請參閱[工作區 `.env` 檔案](/zh-TW/gateway/security#workspace-env-files)。

## 私訊政策與存取控制

- 支援的 `dmPolicy` 值：`allowlist`（預設）、`open` 和 `disabled`。Synology Chat 沒有配對流程；請將傳送者的數字 Synology 使用者 ID 加入 `allowedUserIds` 以核准傳送者。
- `allowedUserIds` 接受 Synology 使用者 ID 清單（或以逗號分隔的字串）。
- 在 `allowlist` 模式下，空白的 `allowedUserIds` 清單會被視為設定錯誤，網路鉤子路由將不會啟動。
- `dmPolicy: "open"` 僅在 `allowedUserIds` 包含 `"*"` 時允許公開私訊；若有受限項目，只有相符的使用者可以聊天。`open` 搭配空白的 `allowedUserIds` 清單時，也會拒絕啟動路由。
- `dmPolicy: "disabled"` 會封鎖私訊。
- 回覆收件者繫結預設會固定使用穩定的數字 `user_id`。`channels.synology-chat.dangerouslyAllowNameMatching: true` 是緊急相容模式，會重新啟用可變的使用者名稱／暱稱查詢以傳遞回覆。

## 傳出傳遞

使用數字 Synology Chat 使用者 ID 作為目標。接受 `synology-chat:`、`synology_chat:` 和 `synology:` 前綴。

範例：

```bash
openclaw message send --channel synology-chat --target 123456 --message "Hello from OpenClaw"
openclaw message send --channel synology-chat --target synology-chat:123456 --message "Hello again"
openclaw message send --channel synology-chat --target synology:123456 --message "Short prefix"
```

傳出文字會以每段 2000 個字元分割。媒體傳送支援以 URL 為基礎的檔案傳遞：NAS 會下載並附加檔案（上限 32 MB）。傳出檔案 URL 必須使用 `http` 或 `https`，而私人或其他遭封鎖的網路目標，會在 OpenClaw 將 URL 轉送至 NAS 網路鉤子前遭到拒絕。

## 多帳號

`channels.synology-chat.accounts` 下支援多個 Synology Chat 帳號。
每個帳號都可覆寫權杖、傳入 URL、網路鉤子路徑、私訊政策和限制。
私訊工作階段會依帳號和使用者隔離，因此兩個不同 Synology 帳號上的相同數字 `user_id`
不會共用對話記錄狀態。
請為每個已啟用的帳號指定不同的 `webhookPath`。OpenClaw 會拒絕完全重複的路徑，
並拒絕啟動在多帳號設定中僅繼承共用網路鉤子路徑的具名帳號。
如果你有意讓具名帳號使用舊版繼承，請在該帳號或 `channels.synology-chat` 設定
`dangerouslyAllowInheritedWebhookPath: true`，
但完全重複的路徑仍會以封閉式失敗方式遭到拒絕。建議為每個帳號明確設定路徑。

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      accounts: {
        default: {
          token: "token-a",
          incomingUrl: "https://nas-a.example.com/...token=...",
        },
        alerts: {
          token: "token-b",
          incomingUrl: "https://nas-b.example.com/...token=...",
          webhookPath: "/webhook/synology-alerts",
          dmPolicy: "allowlist",
          allowedUserIds: ["987654"],
        },
      },
    },
  },
}
```

## 安全性注意事項

- 請妥善保管 `token`，如有外洩請輪替。
- 除非你明確信任使用自簽憑證的本機 NAS，否則請維持 `allowInsecureSsl: false`。
- 傳入網路鉤子要求會驗證權杖，並依傳送者進行速率限制（`rateLimitPerMinute`，預設為 30）。
- 無效權杖檢查會使用固定時間的秘密值比較，並採取封閉式失敗；重複嘗試無效權杖會暫時鎖定來源 IP。
- 傳入訊息文字會針對已知的提示注入模式進行清理，並截斷至 4000 個字元。
- 正式環境建議使用 `dmPolicy: "allowlist"`。
- 除非你明確需要舊版以使用者名稱為基礎的回覆傳遞，否則請停用 `dangerouslyAllowNameMatching`。
- 除非你明確接受多帳號設定中的共用路徑路由風險，否則請停用 `dangerouslyAllowInheritedWebhookPath`。

## 疑難排解

- `Missing required fields (token, user_id, text)`：
  - 傳出網路鉤子承載資料缺少其中一個必要欄位
  - 如果 Synology 在標頭中傳送權杖，請確保閘道／Proxy 保留這些標頭
- `Invalid token`：
  - 傳出網路鉤子的秘密值與 `channels.synology-chat.token` 不相符
  - 要求送達錯誤的帳號／網路鉤子路徑
  - 反向 Proxy 在要求送達 OpenClaw 前移除了權杖標頭
- `Rate limit exceeded`：
  - 來自相同來源的無效權杖嘗試次數過多，可能暫時鎖定該來源
  - 已驗證的傳送者另有獨立的每位使用者訊息速率限制
- `Allowlist is empty. Configure allowedUserIds or use dmPolicy=open with allowedUserIds=["*"].`：
  - `dmPolicy="allowlist"` 已啟用，但未設定任何使用者
- `User not authorized`：
  - 傳送者的數字 `user_id` 不在 `allowedUserIds` 中

## 相關內容

- [頻道概覽](/zh-TW/channels) — 所有支援的頻道
- [群組](/zh-TW/channels/groups) — 群組聊天行為與提及閘門控制
- [頻道路由](/zh-TW/channels/channel-routing) — 訊息的工作階段路由
- [安全性](/zh-TW/gateway/security) — 存取模型與強化
