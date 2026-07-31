---
read_when:
    - 變更群組聊天行為或提及門檻設定
    - 將 mentionPatterns 的範圍限定於特定群組對話
sidebarTitle: Groups
summary: 跨介面（Discord/iMessage/Matrix/Microsoft Teams/QQ Bot/Signal/Slack/Telegram/WhatsApp/Zalo）的群組聊天行為
title: 群組
x-i18n:
    generated_at: "2026-07-26T07:33:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 146378f0fc31e129b6504df6778ab8633c048cd4d02af02a5e6da1bfef640d3f
    source_path: channels/groups.md
    workflow: 16
---

OpenClaw 會在支援群組的頻道中套用相同的群組規則，包括 Discord、iMessage、Matrix、Microsoft Teams、QQ Bot、Signal、Slack、Telegram、WhatsApp 和 Zalo。

對於應持續啟用，且除非代理明確傳送可見訊息，否則只提供安靜情境資訊的聊天室，請參閱[環境聊天室事件](/zh-TW/channels/ambient-room-events)。

## 新手簡介（2 分鐘）

OpenClaw「存在」於你自己的訊息帳號中。沒有獨立的 WhatsApp 機器人使用者：如果**你**在某個群組中，OpenClaw 就能看見該群組並在其中回覆。

預設行為：

- 群組受到限制（`groupPolicy: "allowlist"`）；群組傳送者在加入允許清單前會遭封鎖。
- 除非你為某個群組停用提及閘控，否則必須提及才會回覆。
- 最終回覆文字會自動發布至聊天室（`visibleReplies: "automatic"`）。

換句話說：允許清單中的傳送者可以透過提及 OpenClaw 來觸發它。

<Note>
**簡而言之**

- **私訊存取權**由 `*.allowFrom` 控制。
- **群組存取權**由 `*.groupPolicy` 與允許清單（`*.groups`、`*.groupAllowFrom`）控制。
- **回覆觸發**由提及閘控（`requireMention`、`/activation`）控制。

</Note>

快速流程（群組訊息的處理方式）：

```text
groupPolicy？disabled -> 丟棄
groupPolicy？allowlist -> 群組是否獲准？否 -> 丟棄
requireMention？是 -> 是否被提及？否 -> 僅儲存為情境資訊
提及／回覆／命令／私訊 -> 使用者要求
持續啟用的群組對話 -> 使用者要求，或在已設定時成為聊天室事件
```

## 可見回覆

對於一般的群組／頻道要求，OpenClaw 預設使用 `messages.groupChat.visibleReplies: "automatic"`：最終助理文字會作為可見回覆發布至聊天室。

如果共用聊天室應讓代理透過呼叫 `message(action=send)` 自行決定何時發言，請使用 `messages.groupChat.visibleReplies: "message_tool"`。這最適合能可靠使用工具的模型（例如 GPT-5.6 Sol）。如果模型未使用工具而傳回實質性的最終文字，OpenClaw 會將該文字保留為私密內容，而不會發布至聊天室。

對於無法可靠遵循僅透過工具傳送規則的模型或執行階段，請使用 `"automatic"`：一般最終文字會直接發布至聊天室，而代理仍可針對無法隨最終文字一併傳送的檔案、圖片或其他附件呼叫 `message(action=send)`。

如果目前的工具政策不允許使用訊息工具，OpenClaw 會退回自動傳送可見回覆，而不會無聲地隱藏回應。`openclaw doctor` 會針對此不一致發出警告。

對於直接聊天及任何其他來源事件，`messages.visibleReplies: "message_tool"` 會在全域套用相同的僅限工具行為；`messages.groupChat.visibleReplies` 仍是群組／頻道聊天室更具體的覆寫設定。內部 WebChat 的直接對話預設會自動傳送最終回覆，讓 Pi 與 Codex 採用相同的可見回覆合約。

僅限工具模式取代了舊有模式，不再強迫模型在大多數潛水模式對話中回答 `NO_REPLY`。在僅限工具模式下，提示詞不會定義 `NO_REPLY` 合約；不顯示任何內容，單純表示未呼叫訊息工具。

外掛所擁有的對話繫結是例外。一旦外掛繫結討論串並接管傳入對話，外掛傳回的回覆就是可見的繫結回應；不需要 `message(action=send)`。該回覆是外掛執行階段的輸出，而非私密的模型最終文字。

直接群組要求仍會傳送輸入中指示器。啟用後，環境持續啟用聊天室事件仍會保持嚴格且安靜，除非代理呼叫訊息工具。

工作階段預設會隱藏詳細的工具／進度摘要。偵錯時，使用 `/verbose on`（或 `/verbose full`）即可在目前工作階段顯示這些摘要；使用 `/verbose off` 則會恢復為僅顯示最終回覆的行為。詳細資訊狀態以工作階段為單位，且在直接聊天、群組、頻道與論壇主題中的運作方式相同。

若要將未提及代理的持續啟用群組對話，作為安靜的聊天室情境資訊提交，而不是作為使用者要求，請使用[環境聊天室事件](/zh-TW/channels/ambient-room-events)：

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
    },
  },
}
```

預設值為 `unmentionedInbound: "user_request"`。被提及的訊息、命令、中止要求與私訊仍會視為使用者要求。

若要要求群組／頻道要求的可見輸出必須透過訊息工具傳送：

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "message_tool",
    },
  },
}
```

若要對每個來源聊天套用此要求：

```json5
{
  messages: {
    visibleReplies: "message_tool",
  },
}
```

檔案儲存後，閘道無須重新啟動即可套用 `messages` 設定變更。只有在設定重新載入功能已停用（`gateway.reload.mode: "off"`）時才需要重新啟動。

命令對話會略過 `visibleReplies: "message_tool"`，並且一律提供可見回覆：原生斜線命令（Discord、Telegram，以及其他支援原生命令的介面）與已授權的文字 `/...` 命令，都會將回應發布至來源聊天。在群組中，未獲授權的文字 `/...` 對話仍僅限透過訊息工具處理；一般聊天對話則遵循已設定的預設值。

## 情境資訊可見性與允許清單

群組安全涉及兩種不同的控制項：

- **觸發授權**：誰可以觸發代理（`groupPolicy`、`groups`、`groupAllowFrom`、各頻道專屬的允許清單）。
- **情境資訊可見性**：哪些補充情境資訊會注入模型（回覆／引用文字、討論串歷程記錄、轉寄中繼資料）。

OpenClaw 預設會原樣保留收到的情境資訊：允許清單決定誰可以觸發動作，而不會決定模型可以看見哪些引用或歷史片段。若也要篩選補充情境資訊，請設定 `contextVisibility`：

| 模式                | 行為                                                                         |
| ------------------- | -------------------------------------------------------------------------------- |
| `"all"`（預設）   | 原樣保留收到的補充情境資訊。                                           |
| `"allowlist"`       | 僅注入來自允許清單中傳送者的歷程記錄／討論串／引用／轉寄情境資訊。     |
| `"allowlist_quote"` | `allowlist`，並保留明確引用或回覆的任何傳送者訊息。 |

可按頻道（`channels.<channel>.contextVisibility`）、按帳號（`channels.<channel>.accounts.<accountId>.contextVisibility`）或全域（`channels.defaults.contextVisibility`）設定。會擷取補充情境資訊的頻道（Discord、Feishu、iMessage、Matrix、Microsoft Teams、Signal、Slack、Telegram、WhatsApp）會在建構傳入情境資訊時套用此政策；未知的政策組合會採取封閉式失敗，並省略情境資訊。

這些模式只會篩選頻道提供的補充情境資訊。工具政策與僅限擁有者的工具清單，仍會根據目前對話的原始要求者選取，而不是提示詞中出現的每一位傳送者。請參閱[要求者範圍的控制項與提示詞情境資訊](/zh-TW/gateway/security#requester-scoped-controls-and-prompt-context)。

![群組訊息流程](/images/groups-flow.svg)

如果你想要……

| 目標                                         | 設定方式                                                |
| -------------------------------------------- | ---------------------------------------------------------- |
| 允許所有群組，但只回覆 @提及 | `groups: { "*": { requireMention: true } }`                |
| 停用所有群組回覆                    | `groupPolicy: "disabled"`                                  |
| 僅限特定群組                         | `groups: { "<group-id>": { ... } }`（不使用 `"*"` 鍵）         |
| 群組中只有你可以觸發               | `groupPolicy: "allowlist"`、`groupAllowFrom: ["+1555..."]` |
| 跨頻道重複使用同一組受信任的傳送者 | `groupAllowFrom: ["accessGroup:operators"]`                |

如需可重複使用的傳送者允許清單，請參閱[存取群組](/zh-TW/channels/access-groups)。

## 工作階段鍵

- 群組工作階段使用 `agent:<agentId>:<channel>:group:<id>` 工作階段鍵（聊天室／頻道使用 `agent:<agentId>:<channel>:channel:<id>`）。
- Telegram 論壇主題會將 `:topic:<threadId>` 加入群組 ID，讓每個主題都有自己的工作階段。
- 直接聊天使用主要工作階段（如果已設定 `session.dmScope`，則使用按傳送者區分的工作階段）。
- 心跳偵測會在已設定的心跳偵測工作階段中執行（預設為代理的主要工作階段）；群組工作階段不會執行自己的心跳偵測。

<a id="pattern-personal-dms-public-groups-single-agent"></a>

## 模式：個人私訊 + 公開群組（單一代理）

可以——如果你的「個人」流量是**私訊**，而「公開」流量是**群組**，這種方式會很有效。

原因是：在單一代理模式中，私訊通常會進入**主要**工作階段鍵（`agent:main:main`），而群組一律使用**非主要**工作階段鍵（`agent:main:<channel>:group:<id>`）。如果你透過 `mode: "non-main"` 啟用沙箱，這些群組工作階段會在已設定的沙箱後端中執行，而主要私訊工作階段則保留在主機上。如果未選擇後端，預設使用 Docker。

如此可提供一個代理「大腦」（共用工作區與記憶），但有兩種執行模式：

- **私訊**：完整工具（主機）
- **群組**：沙箱 + 受限制的工具

<Note>
如果需要真正分離的工作區／角色設定（「個人」與「公開」絕不可混合），請使用第二個代理加上繫結。請參閱[多代理路由](/zh-TW/concepts/multi-agent)。
</Note>

<Tabs>
  <Tab title="私訊在主機執行，群組在沙箱執行">
    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main", // 群組／頻道屬於非主要工作階段 -> 在沙箱中執行
            scope: "session", // 最強隔離（每個群組／頻道一個容器）
            workspaceAccess: "none",
          },
        },
      },
      tools: {
        sandbox: {
          tools: {
            // 如果 allow 非空，其他所有項目都會遭封鎖（deny 仍有優先權）。
            allow: ["group:messaging", "group:sessions"],
            deny: ["group:runtime", "group:fs", "group:ui", "nodes", "cron", "gateway"],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="群組只能看見允許清單中的資料夾">
    想要「群組只能看見資料夾 X」，而不是「無法存取主機」嗎？保留 `workspaceAccess: "none"`，並僅將允許清單中的路徑掛載至沙箱：

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",
            scope: "session",
            workspaceAccess: "none",
            docker: {
              binds: [
                // hostPath:containerPath:mode
                "/home/user/FriendsShared:/data:ro",
              ],
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

相關資訊：

- 設定鍵與預設值：[閘道設定](/zh-TW/gateway/config-agents#agentsdefaultssandbox)
- 偵錯工具遭封鎖的原因：[沙箱與工具政策及提升權限的比較](/zh-TW/gateway/sandbox-vs-tool-policy-vs-elevated)
- 繫結掛載詳細資訊：[沙箱](/zh-TW/gateway/sandboxing#custom-bind-mounts)

## 顯示標籤

- 如果 `displayName` 可用，使用者介面標籤會採用該值，並格式化為 `<channel>:<token>`。
- `#room` 保留給聊天室／頻道使用；群組聊天使用 `g-<slug>`（小寫、空格 -> `-`、保留 `#@+._-`）。非常長的不透明 ID 會縮短為穩定的權杖，以免在使用者介面中洩漏完整路由 ID。

## 群組政策

控制各頻道如何處理群組／聊天室訊息：

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "disabled", // "open" | "disabled" | "allowlist"
      groupAllowFrom: ["+15551234567"],
    },
    telegram: {
      groupPolicy: "disabled",
      groupAllowFrom: ["123456789"], // 數字 Telegram 使用者 ID（設定程序會解析 @username）
    },
    signal: {
      groupPolicy: "disabled",
      groupAllowFrom: ["+15551234567"],
    },
    imessage: {
      groupPolicy: "disabled",
      groupAllowFrom: ["chat_id:123"],
    },
    msteams: {
      groupPolicy: "disabled",
      groupAllowFrom: ["user@org.com"],
    },
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        GUILD_ID: { channels: { help: { enabled: true } } },
      },
    },
    slack: {
      groupPolicy: "allowlist",
      channels: { "#general": { enabled: true } },
    },
    matrix: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["@owner:example.org"],
      groups: {
        "!roomId:example.org": { enabled: true },
        "#alias:example.org": { enabled: true },
      },
    },
  },
}
```

| 原則        | 行為                                                     |
| ------------- | ------------------------------------------------------------ |
| `"open"`      | 群組會略過允許清單；提及閘門仍然適用。      |
| `"disabled"`  | 完全封鎖所有群組訊息。                           |
| `"allowlist"` | 僅允許符合所設定允許清單的群組／聊天室。 |

<AccordionGroup>
  <Accordion title="各頻道注意事項">
    - `groupPolicy` 與提及閘門（要求 @提及）分開運作。
    - WhatsApp／Telegram／Signal／iMessage／Microsoft Teams／Zalo：使用 `groupAllowFrom`（備援：明確設定 `allowFrom`）。
    - Signal：`groupAllowFrom` 可比對傳入的 Signal 群組 ID 或傳送者的電話號碼／UUID。
    - 私訊配對核准（`*-allowFrom` 儲存區項目）僅適用於私訊存取；群組傳送者的授權仍須明確使用群組允許清單。
    - Discord：允許清單使用 `channels.discord.guilds.<id>.channels`。
    - Slack：允許清單使用 `channels.slack.channels`。
    - Matrix：允許清單使用 `channels.matrix.groups`。請使用聊天室 ID（`!room:server`）或別名（`#alias:server`）；聊天室名稱索引鍵僅在搭配 `channels.matrix.dangerouslyAllowNameMatching: true` 時才會比對，而無法解析的項目會在執行階段被忽略。使用 `channels.matrix.groupAllowFrom` 限制傳送者；也支援各聊天室的 `users` 允許清單。
    - 群組私訊由其他設定分別控制（`channels.discord.dm.*`、`channels.slack.dm.*`：`groupEnabled`、`groupChannels`）。
    - Telegram：傳送者允許清單僅接受數字使用者 ID（`"123456789"`；`telegram:`/`tg:` 前綴會以不區分大小寫的方式移除）。`@username` 項目不會在執行階段比對，並會記錄警告；設定程序會將 `@username` 解析為 ID。負數聊天 ID 應放在 `channels.telegram.groups` 下，而非傳送者允許清單中。
    - 預設值為 `groupPolicy: "allowlist"`；如果群組允許清單為空，群組訊息將遭到封鎖。
    - 執行階段安全性：當供應商區塊完全缺失（沒有 `channels.<provider>`）時，群組原則會以失敗關閉方式套用 `allowlist`，而非繼承 `channels.defaults.groupPolicy`，且閘道會為每個帳號記錄一次此備援情況。

  </Accordion>
</AccordionGroup>

快速理解模型（群組訊息的評估順序）：

<Steps>
  <Step title="groupPolicy">
    `groupPolicy`（open/disabled/allowlist）。
  </Step>
  <Step title="群組允許清單">
    群組允許清單（`*.groups`、`*.groupAllowFrom`、頻道特定的允許清單）。
  </Step>
  <Step title="提及閘門">
    提及閘門（`requireMention`、`/activation`）。
  </Step>
</Steps>

## 提及閘門（預設）

除非針對個別群組覆寫，否則群組訊息必須包含提及。預設值位於各子系統的 `*.groups."*"` 下。

支援的隱含提及事實因頻道而異：

| 事實                  | 目前的內建產生來源                       |
| --------------------- | ------------------------------------------------ |
| 回覆機器人      | Discord、Microsoft Teams、QQ Bot、Slack、Telegram |
| 引用機器人      | WhatsApp、Zalo Personal                          |
| 機器人已加入討論串 | Mattermost、Slack、Tlon                          |

當頻道產生某項事實時，該事實預設為啟用。將對應的 `implicitMentions` 旗標設為 `false`，即可阻止該事實略過提及閘門；原生的明確提及不受影響。對於不會產生該事實的頻道，此旗標不會有任何作用。

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
        "123@g.us": { requireMention: false },
      },
    },
    telegram: {
      groups: {
        "*": { requireMention: true },
        "123456789": { requireMention: false },
      },
    },
    imessage: {
      groups: {
        "*": { requireMention: true },
        "123": { requireMention: false },
      },
    },
  },
  agents: {
    entries: {
      main: {
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw", "\\+15555550123"],
          historyLimit: 50,
        },
      },
    },
  },
}
```

## 限定所設定提及模式的範圍

所設定的 `mentionPatterns` 是正規表示式備援觸發條件。當平台未提供原生機器人提及，或你希望讓 `openclaw:` 之類的純文字也算作提及時，請使用這些模式。原生平台提及則是另一回事：當 Discord、Slack、Telegram、Matrix、Signal 或其他頻道能夠確認訊息明確提及機器人時，即使所設定的正規表示式模式遭到拒絕，該原生提及仍會觸發。

依預設，只要頻道將供應商與對話事實傳入提及偵測，所設定的提及模式就會套用。為避免寬泛模式在每個群組中喚醒代理程式，請使用 `channels.<channel>.mentionPatterns` 針對各頻道限定其範圍。

如果某個頻道預設應停用正規表示式提及模式，請使用 `mode: "deny"`，再透過 `allowIn` 為特定聊天室選擇啟用：

```json5
{
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b", "\\bops bot\\b"],
    },
  },
  channels: {
    slack: {
      mentionPatterns: {
        mode: "deny",
        allowIn: ["C0123OPS"],
      },
    },
  },
}
```

當正規表示式提及模式應廣泛套用時，請使用預設的 `mode: "allow"`（或省略 `mode`），再使用 `denyIn` 在訊息繁雜的聊天室中將其關閉：

```json5
{
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b"],
    },
  },
  channels: {
    telegram: {
      mentionPatterns: {
        denyIn: ["-1001234567890", "-1001234567890:topic:42"],
      },
    },
  },
}
```

原則解析：

| 欄位           | 效果                                                                                                                |
| --------------- | --------------------------------------------------------------------------------------------------------------------- |
| `mode: "allow"` | 除非對話 ID 位於 `denyIn` 中，否則會啟用正規表示式提及模式。這是預設值。                    |
| `mode: "deny"`  | 除非對話 ID 位於 `allowIn` 中，否則會停用正規表示式提及模式。                                       |
| `allowIn`       | 在拒絕模式下啟用正規表示式提及模式的對話 ID。                                               |
| `denyIn`        | 停用正規表示式提及模式的對話 ID。如果兩者都包含相同 ID，`denyIn` 優先於 `allowIn`。 |

目前支援的範圍式正規表示式原則：

| 頻道  | `allowIn` / `denyIn` 中使用的 ID                             |
| -------- | ------------------------------------------------------------ |
| Discord  | Discord 頻道 ID。                                         |
| Matrix   | Matrix 聊天室 ID。                                             |
| Slack    | Slack 頻道 ID。                                           |
| Telegram | 群組聊天 ID，或論壇主題使用的 `chatId:topic:threadId`。 |
| WhatsApp | WhatsApp 對話 ID，例如 `123@g.us`。                |

當頻道支援多個帳號時，帳號層級的頻道設定可以在 `channels.<channel>.accounts.<accountId>.mentionPatterns` 下設定相同原則。該帳號的帳號原則優先於頂層頻道原則。

<AccordionGroup>
  <Accordion title="提及閘控注意事項">
    - `mentionPatterns` 是不區分大小寫的安全正規表示式模式；無效模式與不安全的巢狀重複形式會被忽略（並發出警告）。
    - 模式優先順序：`agents.entries.*.groupChat.mentionPatterns`（多個代理共用一個群組時很有用）會覆寫 `messages.groupChat.mentionPatterns`；兩者皆未設定時，模式會從代理身分的名稱／表情符號衍生。
    - 只有在能夠偵測提及時（原生提及或已設定 `mentionPatterns`），才會強制執行提及閘控。
    - 將群組或傳送者加入允許清單不會停用提及閘控；當所有訊息都應觸發時，請將該群組的 `requireMention` 設為 `false`。
    - 自動群組聊天提示內容會在每一輪攜帶解析後的靜默回覆指示；工作區檔案不應重複 `NO_REPLY` 的運作機制。
    - 允許自動靜默回覆的群組，會將完全空白或僅含推理的模型輪次視為靜默，等同於 `NO_REPLY`。直接聊天絕不會收到 `NO_REPLY` 指引，而僅使用訊息工具的群組回覆則透過不呼叫 `message(action=send)` 來保持安靜。
    - 預設情況下，環境中持續進行的群組閒聊會採用使用者請求語意。請設定 `messages.groupChat.unmentionedInbound: "room_event"`，改為將其作為安靜的內容提交。設定範例請參閱[環境聊天室事件](/zh-TW/channels/ambient-room-events)。
    - 聊天室事件不會儲存為虛假的使用者請求，而未使用訊息工具的聊天室事件所產生的私人助理文字，也不會作為聊天記錄重新播放。
    - Discord 預設值位於 `channels.discord.guilds."*"`（可依伺服器／頻道覆寫）。
    - 群組記錄內容在各頻道中會以一致方式包裝。採用提及閘控的群組會保留尚待處理的略過訊息；若頻道支援，持續啟用的群組也可能保留近期已處理的聊天室訊息。全域預設值請使用 `messages.groupChat.historyLimit`，覆寫則使用 `channels.<channel>.historyLimit`（或 `channels.<channel>.accounts.*.historyLimit`）。設為 `0` 可停用。

  </Accordion>
</AccordionGroup>

## 群組／頻道工具限制（選用）

部分頻道設定支援限制**特定群組／聊天室／頻道內**可使用的工具。

- `tools`：允許／拒絕整個群組使用工具（`allow`、`alsoAllow`、`deny`；拒絕優先）。
- `toolsBySender`：群組內依傳送者覆寫。請使用明確的鍵前綴：`channel:<channelId>:<senderId>`、`id:<senderId>`、`e164:<phone>`、`username:<handle>`、`name:<displayName>`，以及 `"*"` 萬用字元。頻道 ID 使用標準 OpenClaw 頻道 ID；`teams` 等別名會正規化為 `msteams`。仍接受舊版無前綴鍵，但只會比對為 `id:`，並記錄棄用警告。

解析順序（最明確者優先）：

<Steps>
  <Step title="群組 toolsBySender">
    群組／頻道 `toolsBySender` 比對。
  </Step>
  <Step title="群組 tools">
    群組／頻道 `tools`。
  </Step>
  <Step title="預設 toolsBySender">
    預設（`"*"`）`toolsBySender` 比對。
  </Step>
  <Step title="預設 tools">
    預設（`"*"`）`tools`。
  </Step>
</Steps>

範例（Telegram）：

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { tools: { deny: ["exec"] } },
        "-1001234567890": {
          tools: { deny: ["exec", "read", "write"] },
          toolsBySender: {
            "id:123456789": { alsoAllow: ["exec"] },
          },
        },
      },
    },
  },
}
```

<Note>
群組／頻道工具限制會在全域／代理程式工具政策之外另外套用（拒絕規則仍優先）。部分頻道對聊天室／頻道使用不同的巢狀結構（例如 Discord `guilds.*.channels.*`、Slack `channels.*`、Microsoft Teams `teams.*.channels.*`）。
</Note>

## 群組允許清單

設定 `channels.whatsapp.groups`、`channels.telegram.groups` 或 `channels.imessage.groups` 時，其鍵會作為群組允許清單。使用 `"*"` 可允許所有群組，同時仍設定預設的提及行為。

<Warning>
常見混淆：私訊配對核准與群組授權並不相同。對於支援私訊配對的頻道，配對儲存區只會解鎖私訊。群組命令仍需透過設定允許清單（例如 `groupAllowFrom`）或該頻道文件所述的設定備援方式，明確授權群組傳送者。
</Warning>

常見用途（複製／貼上）：

<Tabs>
  <Tab title="停用所有群組回覆">
    ```json5
    {
      channels: { whatsapp: { groupPolicy: "disabled" } },
    }
    ```
  </Tab>
  <Tab title="僅允許特定群組（WhatsApp）">
    ```json5
    {
      channels: {
        whatsapp: {
          groups: {
            "123@g.us": { requireMention: true },
            "456@g.us": { requireMention: false },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="允許所有群組，但要求提及">
    ```json5
    {
      channels: {
        whatsapp: {
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```
  </Tab>
  <Tab title="僅限擁有者觸發（WhatsApp）">
    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## 啟用方式（僅限擁有者）

群組擁有者可透過獨立訊息切換各群組的啟用狀態：

- `/activation mention`
- `/activation always`

`/activation` 是核心中受擁有者權限控管的命令，且僅適用於群組聊天。擁有者是指傳送者符合 `commands.ownerAllowFrom`；頻道的 `allowFrom` 清單僅控制一般頻道與命令存取權。對於會查詢儲存模式的頻道（Google Chat、QQ Bot、Telegram、WhatsApp），儲存的模式會覆寫該群組的 `requireMention`，且各處的群組系統提示詞引言都會反映目前啟用的模式。

## 情境欄位

群組傳入承載資料會設定：

- `ChatType=group`
- `GroupSubject`（若已知）
- `GroupMembers`（若已知）
- `WasMentioned`（提及閘控結果）
- Telegram 論壇主題也包含 `MessageThreadId` 和 `IsForum`。

代理程式系統提示詞會在新群組工作階段的第一輪（以及 `/activation` 變更後）加入群組引言。它會提醒模型像真人一樣回應、減少空白行並遵循一般聊天間距，且避免輸入字面上的 `\n` 序列。若頻道宣告的表格模式無法保留原生或原始表格，也會不建議使用 Markdown 表格。來自頻道的群組名稱和參與者標籤會呈現為圍欄式不受信任中繼資料，而非行內系統指令。

## iMessage 特定事項

- 進行路由或設定允許清單時，建議使用 `chat_id:<id>`。
- 列出聊天：`imsg chats --limit 20`。
- 群組回覆一律傳回相同的 `chat_id`。

## WhatsApp 系統提示詞

請參閱 [WhatsApp](/zh-TW/channels/whatsapp#system-prompts)，瞭解標準 WhatsApp 系統提示詞規則，包括群組與直接提示詞解析、萬用字元行為，以及帳號覆寫語意。

## WhatsApp 特定事項

請參閱[群組訊息](/zh-TW/channels/group-messages)，瞭解僅適用於 WhatsApp 的行為（歷史記錄注入、提及處理細節）。

## 相關內容

- [廣播群組](/zh-TW/channels/broadcast-groups)
- [頻道路由](/zh-TW/channels/channel-routing)
- [群組訊息](/zh-TW/channels/group-messages)
- [配對](/zh-TW/channels/pairing)
