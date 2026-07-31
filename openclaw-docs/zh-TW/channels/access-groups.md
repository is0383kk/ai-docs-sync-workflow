---
read_when:
    - 在多個訊息頻道中設定相同的允許清單
    - 共用私訊與群組傳送者存取規則
    - 審查訊息頻道存取控制
summary: 訊息頻道的可重複使用傳送者允許清單
title: 存取群組
x-i18n:
    generated_at: "2026-07-26T08:06:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 099abc95e90d9a7b7006d19062c46b4ffdb2aecb1e8e714454a3182131a786d0
    source_path: channels/access-groups.md
    workflow: 16
---

存取群組是具名的傳送者清單，你只需在 `accessGroups` 下定義一次，之後即可在頻道允許清單中使用 `accessGroup:<name>` 參照。

當同一批人員應獲准使用多個訊息頻道，或同一個受信任群組應同時套用於私訊和群組傳送者授權時，請使用存取群組。

群組本身不會授予任何權限。只有允許清單欄位參照群組時，它才會生效。

## 靜態訊息傳送者群組

靜態傳送者群組使用 `type: "message.senders"`。`members` 以訊息頻道 ID 為鍵，並可使用 `"*"` 設定所有頻道共用的項目：

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        "*": ["global-owner-id"],
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
        whatsapp: ["+15551234567"],
      },
    },
  },
}
```

| 鍵                         | 意義                                                                 |
| -------------------------- | -------------------------------------------------------------------- |
| `"*"`                      | 所有參照此群組的訊息頻道都會檢查的共用項目。                         |
| `discord`、`telegram`、... | 僅在該頻道進行允許清單比對時檢查的項目。                             |

項目會依照目的地頻道的一般 `allowFrom` 規則進行比對。OpenClaw 不會在頻道之間轉換傳送者 ID：如果 Alice 同時有 Telegram ID 和 Discord ID，請將兩個 ID 分別列在對應的頻道鍵下。

## 從允許清單參照群組

在訊息頻道路徑支援傳送者允許清單的任何位置，使用 `accessGroup:<name>` 參照群組。

私訊允許清單範例：

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
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
    telegram: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
  },
}
```

群組傳送者允許清單範例：

```json5
{
  accessGroups: {
    oncall: {
      type: "message.senders",
      members: {
        whatsapp: ["+15551234567"],
        googlechat: ["users/1234567890"],
      },
    },
  },
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["accessGroup:oncall"],
    },
    googlechat: {
      groups: {
        "spaces/AAA": {
          users: ["accessGroup:oncall"],
        },
      },
    },
  },
}
```

你可以混用群組與直接項目：

```json5
{
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators", "discord:123456789012345678"],
    },
  },
}
```

## 支援的訊息頻道路徑

存取群組可用於共用的訊息頻道授權路徑：

- 私訊傳送者允許清單，例如 `channels.<channel>.allowFrom`
- 群組傳送者允許清單，例如 `channels.<channel>.groupAllowFrom`
- 使用相同傳送者比對規則的頻道專屬個別聊天室傳送者允許清單（例如 Google Chat `groups.<space>.users`）
- 重複使用訊息頻道傳送者允許清單的命令授權路徑

頻道是否支援取決於該頻道是否已連接至 OpenClaw 共用的傳送者授權輔助工具。目前隨附的支援包括 ClickClack、Discord、Feishu、Google Chat、iMessage、IRC、LINE、Mattermost、Microsoft Teams、Nextcloud Talk、Nostr、QQ Bot、Signal、Slack、SMS、Telegram、WhatsApp、Zalo 和 Zalo Personal。靜態 `message.senders` 群組不受頻道限制，因此新訊息頻道只要使用共用的外掛 SDK 輸入輔助工具，而非自訂允許清單展開邏輯，即可支援這些群組。

## Discord 頻道受眾

Discord 也支援動態存取群組類型：

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

`discord.channelAudience` 表示「允許目前可檢視頻道所屬 Discord 伺服器中此頻道的 Discord 私訊傳送者」。OpenClaw 會在授權時透過 Discord 解析傳送者，並套用 Discord `ViewChannel` 權限規則。`membership` 為選用項目，預設為 `canViewChannel`。

當 Discord 頻道已是某個團隊的權威資料來源時（例如 `#maintainers` 或 `#on-call`），請使用此功能。

需求與失敗行為：

- 機器人必須能存取 Discord 伺服器與頻道。
- 機器人必須具備 Discord Developer Portal 的 **Server Members Intent**。
- 當 Discord 傳回 `Missing Access`、無法將傳送者解析為 Discord 伺服器成員，或頻道屬於另一個 Discord 伺服器時，存取群組會採取拒絕存取的安全失敗模式。

更多 Discord 專屬範例：[Discord 存取控制](/zh-TW/channels/discord#access-control-and-routing)

## 外掛診斷

外掛作者可以檢查結構化存取群組狀態，而不必將其重新展開為扁平允許清單：

```typescript
import { resolveAccessGroupAllowFromState } from "openclaw/plugin-sdk/access-groups";

const state = await resolveAccessGroupAllowFromState({
  accessGroups: cfg.accessGroups,
  allowFrom: channelConfig.allowFrom,
  channel: "my-channel",
  accountId: "default",
  senderId,
  isSenderAllowed,
});
```

結果會回報已參照、已比對、遺失、不支援和失敗的群組。請將其用於診斷或一致性測試。只有仍預期使用扁平 `allowFrom` 陣列的相容性路徑才應使用 `expandAllowFromWithAccessGroups(...)`。

## 安全性注意事項

- 存取群組是允許清單別名，而非角色。它們本身不會建立擁有者、核准配對要求或授予工具權限。
- `dmPolicy: "open"` 仍要求有效的私訊允許清單中包含 `"*"`。參照存取群組並不等同於公開存取。
- 遺失的群組名稱會採取拒絕存取的安全失敗模式。如果 `allowFrom` 包含 `accessGroup:operators`，但 `accessGroups.operators` 不存在，該項目不會授權任何人。
- 請保持頻道 ID 穩定。如果頻道同時支援數字／使用者 ID 和顯示名稱，請優先使用前者。

## 疑難排解

如果傳送者應能比對卻遭到封鎖：

1. 確認允許清單欄位包含完全相符的 `accessGroup:<name>` 參照。
2. 確認 `accessGroups.<name>.type` 正確無誤。
3. 確認傳送者 ID 已列在相符的頻道鍵或 `"*"` 下。
4. 確認項目使用該頻道的一般允許清單語法。
5. 若使用 Discord 頻道受眾，請確認機器人能看見 Discord 伺服器頻道，且已啟用 Server Members Intent。

編輯存取控制設定後，請執行 `openclaw doctor`。它能在執行階段之前找出許多無效的允許清單與原則組合。
