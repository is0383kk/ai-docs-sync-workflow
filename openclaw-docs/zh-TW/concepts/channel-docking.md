---
read_when:
    - 你想要將一個作用中工作階段的回覆從 Telegram 移至 Discord、Slack、Mattermost 或其他已連結的頻道
    - 你正在設定 `session.identityLinks`，以用於跨頻道私訊
    - /dock 命令顯示傳送者尚未連結，或不存在作用中的工作階段
summary: 在已連結的聊天頻道之間移動一個 OpenClaw 工作階段的回覆路由
title: 頻道停駐
x-i18n:
    generated_at: "2026-07-26T07:39:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d7af3a59b95b2c73cb74a9529584e51caed055719db2df8aad2ba8e8c9b0593
    source_path: concepts/channel-docking.md
    workflow: 16
---

頻道停靠是針對單一 OpenClaw 工作階段的來電轉接。它會保留相同的
對話情境，但會變更該工作階段未來回覆的傳送位置。
停靠只能從直接聊天執行；無法從群組聊天執行。

## 範例

Alice 可以透過 Telegram 和 Discord 傳訊息給 OpenClaw：

```json5
{
  session: {
    identityLinks: {
      alice: ["telegram:123", "discord:456"],
    },
  },
}
```

如果 Alice 從 Telegram 直接聊天傳送以下內容：

```text
/dock_discord
```

OpenClaw 會保留目前的工作階段情境，並變更回覆路由：

| 停靠前               | `/dock_discord`後       |
| ---------------------------- | --------------------------- |
| 回覆會傳送至 Telegram `123` | 回覆會傳送至 Discord `456` |

系統不會重新建立工作階段。逐字記錄歷史仍會附加至
同一個工作階段。

## 使用原因

當任務在一個聊天應用程式中開始，但接下來的回覆應傳送至
其他位置時，請使用停靠功能。

常見流程：

1. 從 Telegram 啟動代理程式任務。
2. 移至你正在協調工作的 Discord。
3. 從 Telegram 直接聊天傳送 `/dock_discord`。
4. 保留相同的 OpenClaw 工作階段，但在 Discord 接收未來的回覆。

## 必要設定

停靠需要 `session.identityLinks`。來源傳送者和目標對等端
必須位於同一個身分群組中：

```json5
{
  session: {
    identityLinks: {
      alice: ["telegram:123", "discord:456", "slack:U123"],
    },
  },
}
```

這些值是帶有頻道前綴的對等端 ID：

| 值          | 意義                      |
| -------------- | ---------------------------- |
| `telegram:123` | Telegram 傳送者 ID `123`     |
| `discord:456`  | Discord 直接對等端 ID `456` |
| `slack:U123`   | Slack 使用者 ID `U123`         |

標準鍵（上方的 `alice`）只是共用身分群組名稱。停靠
命令會使用帶有頻道前綴的值，證明來源傳送者和
目標對等端是同一個人。

## 命令

OpenClaw 會為每個支援原生命令且已載入的頻道外掛
產生一個 `/dock-<channel>` 命令，因此清單會隨著新增外掛而增加。目前支援此功能的
隨附外掛包括：

| 目標頻道 | 命令            | 別名              |
| -------------- | ------------------ | ------------------ |
| Discord        | `/dock-discord`    | `/dock_discord`    |
| Mattermost     | `/dock-mattermost` | `/dock_mattermost` |
| Slack          | `/dock-slack`      | `/dock_slack`      |
| Telegram       | `/dock-telegram`   | `/dock_telegram`   |

底線形式也是 Telegram 等直接公開斜線命令之介面上的
原生命令名稱。

## 變更內容

停靠會更新作用中工作階段的傳送欄位：

| 工作階段欄位   | `/dock_discord` 後的範例            |
| --------------- | ---------------------------------------- |
| `lastChannel`   | `discord`                                |
| `lastTo`        | `456`                                    |
| `lastAccountId` | 目標頻道帳號，或 `default` |

這些欄位會持久儲存在工作階段存放區中，並供該工作階段後續傳送
回覆時使用。

## 不會變更的內容

停靠不會：

- 建立頻道帳號
- 連接新的 Discord、Telegram、Slack 或 Mattermost 機器人
- 授予使用者存取權
- 繞過頻道允許清單或私人訊息政策
- 將逐字記錄歷史移至另一個工作階段
- 讓不相關的使用者共用工作階段

它只會變更目前工作階段的傳送路由。

## 疑難排解

**命令表示傳送者尚未連結。**

將目前的傳送者和目標對等端都新增至相同的
`session.identityLinks` 群組。例如，如果 Telegram 傳送者 `123` 應停靠至
Discord 對等端 `456`，請同時加入 `telegram:123` 和 `discord:456`。

**命令表示停靠功能只能從直接聊天使用。**

請從與 OpenClaw 的直接聊天傳送停靠命令，而非從群組聊天傳送。

**命令表示沒有作用中的工作階段。**

請從現有的直接聊天工作階段進行停靠。此命令需要作用中的工作階段
項目，才能持久儲存新路由。

**回覆仍傳送至舊頻道。**

請檢查命令是否回覆成功訊息，並確認目標
對等端 ID 與該頻道使用的 ID 相符。停靠只會變更作用中的
工作階段路由；其他工作階段仍可能路由至別處。

**我需要切換回去。**

請從已連結的傳送者傳送原始頻道的對應命令，例如 `/dock_telegram` 或
`/dock-telegram`。
