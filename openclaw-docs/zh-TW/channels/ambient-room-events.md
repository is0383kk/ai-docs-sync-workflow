---
read_when:
    - 設定永遠啟用的群組或頻道聊天室
    - 你希望代理程式關注聊天室中的對話，但不要自動發布最終文字內容
    - 偵錯輸入狀態與權杖用量，但聊天室中沒有可見訊息
sidebarTitle: Ambient room events
summary: 讓支援的群組聊天室提供安靜的情境資訊，除非代理使用訊息工具傳送訊息
title: 環境房間事件
x-i18n:
    generated_at: "2026-07-26T08:15:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 15c083c139058c9bd2c651794965bd8252d74691e536db2ad2a2ae0b4ac886e8
    source_path: channels/ambient-room-events.md
    workflow: 16
---

環境房間事件讓 OpenClaw 將群組或頻道中未提及它的閒聊作為低干擾的上下文處理。代理程式可以更新記憶與工作階段狀態，但除非代理程式明確呼叫 `message` 工具，否則房間會保持靜默。

對於始終開啟的群組聊天，請將 `messages.groupChat.unmentionedInbound: "room_event"` 與 `messages.groupChat.visibleReplies: "message_tool"` 搭配使用。代理程式會持續聆聽、判斷何時回覆有幫助，而且不再需要以 `NO_REPLY` 回答的舊提示詞模式。

目前支援：Discord 伺服器頻道、Slack 頻道與私人頻道、Slack 多人私訊，以及 Telegram 群組或超級群組。其他群組頻道會維持其現有群組行為，除非其頻道頁面說明支援環境房間事件。

## 建議設定

設定全域群組聊天行為：

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
}
```

接著停用該房間的提及閘控，使房間始終開啟。房間仍須通過其一般 `groupPolicy`、房間允許清單及傳送者允許清單。

儲存設定後，閘道會熱套用 `messages` 設定。只有在檔案監看或設定重新載入已停用時才需重新啟動（`gateway.reload.mode: "off"`）。

## 變更內容

使用 `messages.groupChat.unmentionedInbound: "room_event"` 時：

- 允許的群組或頻道中未提及代理程式的訊息會成為低干擾的房間事件
- 提及代理程式的訊息仍是使用者要求
- 文字控制命令與原生命令仍是使用者要求
- 中止或停止要求仍是使用者要求
- 私訊仍是使用者要求

房間事件使用嚴格的可見傳送規則。助理的最終文字屬於私密內容。代理程式必須呼叫 `message(action=send)`，才能在房間中發文。

房間事件仍會抑制輸入中提示與生命週期狀態反應。唯一明確的收件確認例外是 `messages.ackReactionScope: "all"`，它會傳送已設定的確認反應；若房間必須完全保持靜默，請使用更窄的範圍或 `"off"`。

## Discord 範例

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "<DISCORD_SERVER_ID>": {
          requireMention: false,
          users: ["<YOUR_DISCORD_USER_ID>"],
        },
      },
    },
  },
}
```

只有一個頻道應採用環境模式時，請使用 Discord 的個別頻道設定。在 `groupPolicy: "allowlist"` 下方列出頻道即代表允許該頻道（`enabled: false` 會停用項目）：

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "<DISCORD_SERVER_ID>": {
          channels: {
            "<DISCORD_CHANNEL_ID_OR_NAME>": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

## Slack 範例

Slack 頻道允許清單以 ID 為優先。請使用 `C12345678` 等頻道 ID，而非 `#channel-name`。在 `channels.slack.channels` 下方列出頻道即代表允許該頻道（`enabled: false` 會停用項目）：

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    slack: {
      groupPolicy: "allowlist",
      channels: {
        "<SLACK_CHANNEL_ID>": {
          requireMention: false,
        },
      },
    },
  },
}
```

## Telegram 範例

對於 Telegram 群組，機器人必須能看到一般群組訊息。如果 `requireMention: false`，請停用 BotFather 隱私模式，或使用其他能將完整群組流量傳送給機器人的 Telegram 設定。

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    telegram: {
      groups: {
        "<TELEGRAM_GROUP_CHAT_ID>": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

Telegram 群組 ID 通常是 `-1001234567890` 之類的負數。請從 `openclaw logs --follow` 讀取 `chat.id`、將群組訊息轉傳給 ID 輔助機器人，或檢查 Bot API `getUpdates`。

## 代理程式特定原則

當多個代理程式共用同一個房間，但只有其中一個應將未提及它的閒聊視為環境上下文時，請使用代理程式覆寫：

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "message_tool",
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          unmentionedInbound: "room_event",
          mentionPatterns: ["@openclaw", "openclaw"],
        },
      },
    ],
  },
}
```

代理程式特定的 `agents.entries.*.groupChat.unmentionedInbound` 值會為該代理程式覆寫 `messages.groupChat.unmentionedInbound`。

## 可見回覆模式

對於一般群組／頻道使用者要求，`messages.groupChat.visibleReplies` 預設為 `"automatic"`。當助理最終文字應在不明確呼叫訊息工具的情況下公開發文時，請保留此預設值。

對於環境模式的始終開啟房間，仍建議使用 `messages.groupChat.visibleReplies: "message_tool"`，尤其是搭配 GPT-5.6 Sol 等最新一代、能可靠使用工具的模型。它讓代理程式透過呼叫訊息工具來決定何時發言。如果模型未呼叫工具便傳回最終文字，OpenClaw 會將該最終文字保持為私密內容，並記錄已抑制傳送的中繼資料。

即使其他群組要求使用自動回覆，房間事件仍採用嚴格模式。未提及代理程式的環境房間事件一律需要 `message(action=send)` 才能產生可見輸出。

## 歷史記錄

`messages.groupChat.historyLimit` 設定全域群組歷史記錄預設值（未設定時為 50；必須是正整數）。頻道可以使用 `channels.<channel>.historyLimit` 覆寫此值，部分頻道也支援個別帳號的歷史記錄限制。將頻道層級的 `historyLimit: 0` 設為停用，即可停用該頻道的群組歷史記錄上下文。

支援房間事件的頻道會保留最近的環境房間訊息作為上下文。Telegram 會為每個群組保留一個始終開啟、受 `historyLimit` 限制的滾動視窗；使用者要求回合會選取機器人最後一次記錄回覆之後的項目，而房間事件回合會收到完整的近期視窗，使模型能看到自己最近的發文。已淘汰的 Telegram `includeGroupHistoryContext` 模式鍵會由 `openclaw doctor --fix` 移除。

## 疑難排解

如果房間顯示輸入中或權杖用量，卻沒有可見訊息：

1. 確認頻道允許清單與傳送者允許清單均允許該房間。
2. 確認 `requireMention: false` 已設於你預期的房間層級。
3. 檢查 `messages.groupChat.unmentionedInbound` 或代理程式覆寫是否為 `"room_event"`。
4. 檢查日誌中的已抑制最終承載資料中繼資料或 `didSendViaMessagingTool: false`。
5. 對於一般群組要求，如果你希望自動發佈最終回覆，請保留或還原 `messages.groupChat.visibleReplies: "automatic"`。對於使用 `message_tool` 的環境房間，請使用能可靠呼叫工具的模型／執行階段。

如果 Telegram 環境房間完全未觸發，請檢查 BotFather 隱私模式，並確認閘道正在接收一般群組訊息。

如果 Slack 環境房間未觸發，請確認頻道鍵是 Slack 頻道 ID，且應用程式具有該房間類型的歷史記錄範圍：`channels:history`（公開）、`groups:history`（私人）或 `mpim:history`（多人私訊）。

## 相關內容

- [群組](/zh-TW/channels/groups)
- [Discord](/zh-TW/channels/discord)
- [Slack](/zh-TW/channels/slack)
- [Telegram](/zh-TW/channels/telegram)
- [頻道疑難排解](/zh-TW/channels/troubleshooting)
- [頻道設定參考](/zh-TW/gateway/config-channels)
