---
read_when:
    - 設定由機器人撰寫的頻道訊息
    - 調校機器人對機器人迴圈防護機制
sidebarTitle: Bot loop protection
summary: 機器人對機器人循環防護的預設值與頻道覆寫設定
title: 機器人迴圈防護
x-i18n:
    generated_at: "2026-07-26T08:23:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d59d3b48dd5506e774282b880334df8970b05c4d001261ff7107e8e1678894db
    source_path: channels/bot-loop-protection.md
    workflow: 16
---

OpenClaw 可接受其他機器人在支援 `allowBots` 的頻道上撰寫的訊息。啟用該路徑後，配對迴圈防護可避免兩個機器人身分無限期地互相回覆。

此防護由核心輸入回覆執行器強制執行。每個支援的頻道都會將其輸入事件對應為通用資訊：帳號或範圍、對話 ID、傳送者機器人 ID，以及接收者機器人 ID。核心會雙向追蹤參與者配對（A 到 B 與 B 到 A 視為同一配對）、套用滑動視窗預算，並在超出預算後於冷卻期間抑制該配對。

## 預設值

只要頻道允許機器人撰寫的訊息進入分派流程，配對迴圈防護就會啟用。內建預設值：

| 鍵                  | 預設值 | 含義                                             |
| -------------------- | ------- | --------------------------------------------------- |
| `enabled`            | `true`  | 對支援此功能的頻道啟用防護。          |
| `maxEventsPerWindow` | `20`    | 機器人配對可在視窗內交換的事件數。   |
| `windowSeconds`      | `60`    | 滑動視窗長度。                              |
| `cooldownSeconds`    | `60`    | 配對超出預算後的抑制時間。 |

此防護不會影響由人員撰寫的訊息、單一機器人部署、自身訊息篩選，或未超出預算的機器人回覆。

## 設定共用預設值

設定一次 `channels.defaults.botLoopProtection`，即可讓每個支援的頻道使用相同基準。頻道也可能提供範圍更小的覆寫設定；Feishu 刻意只使用此共用基準。

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
  },
}
```

只有當你的頻道政策刻意允許機器人之間的對話不受自動抑制時，才設定 `enabled: false`。

## 依頻道、帳號或聊天室覆寫

支援的頻道會逐鍵將各自的設定疊加在共用預設值之上。優先順序由最小範圍開始：

1. `channels.<channel>.<room-or-space>.botLoopProtection`，當頻道支援個別對話覆寫時
2. `channels.<channel>.accounts.<account>.botLoopProtection`，當頻道支援帳號時
3. `channels.<channel>.botLoopProtection`，當頻道支援頂層預設值時
4. `channels.defaults.botLoopProtection`
5. 內建預設值

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
      },
    },
    discord: {
      botLoopProtection: {
        maxEventsPerWindow: 8,
      },
      accounts: {
        secondary: {
          allowBots: true,
          botLoopProtection: {
            maxEventsPerWindow: 5,
            cooldownSeconds: 90,
          },
        },
      },
    },
    googlechat: {
      allowBots: true,
      groups: {
        "spaces/AAAA": {
          botLoopProtection: {
            maxEventsPerWindow: 5,
          },
        },
      },
    },
    matrix: {
      allowBots: "mentions",
      groups: {
        "!roomid:example.org": {
          botLoopProtection: {
            maxEventsPerWindow: 5,
          },
        },
      },
    },
    slack: {
      allowBots: "mentions",
      botLoopProtection: {
        maxEventsPerWindow: 8,
      },
    },
  },
}
```

## 頻道支援

- Discord：原生 `author.bot` 資訊，以 Discord 帳號、頻道和機器人配對為鍵。
- Feishu：針對允許進入的機器人撰寫群組訊息提供原生 `sender_type=bot` 資訊，以 Feishu 帳號、聊天和機器人配對為鍵。Feishu 僅使用 `channels.defaults.botLoopProtection`。
- Google Chat：針對已接受的機器人撰寫訊息提供原生 `sender.type=BOT` 資訊，以帳號、聊天室和機器人配對為鍵。
- Matrix：已設定的 Matrix 機器人帳號，以 Matrix 帳號、聊天室和已設定的機器人配對為鍵。
- Slack：針對已接受的機器人撰寫訊息提供原生 `bot_id` 資訊，以 Slack 帳號、頻道和機器人配對為鍵。

未提供可靠輸入機器人身分的頻道會繼續使用其一般的自身訊息和存取政策篩選器。在能夠識別機器人配對中的兩個參與者之前，這些頻道不應啟用此防護。

如需外掛實作詳細資訊，請參閱 [SDK 執行階段](/zh-TW/plugins/sdk-runtime#reusable-runtime-utilities)。
