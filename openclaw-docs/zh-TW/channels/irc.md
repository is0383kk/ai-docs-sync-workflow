---
read_when:
    - 你想要將 OpenClaw 連接至 IRC 頻道或私訊
    - 你正在設定 IRC 允許清單、群組政策或提及閘控
summary: IRC 外掛設定、存取控制與疑難排解
title: IRC
x-i18n:
    generated_at: "2026-07-26T08:16:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 85c3da80b45d6611872ddbd10b3be4a5742b46e355e8bb554353a478f2a1702f
    source_path: channels/irc.md
    workflow: 16
---

當你想在經典頻道（`#room`）和私人訊息中使用 OpenClaw 時，請使用 IRC。
安裝官方 IRC 外掛，然後在 `channels.irc` 下進行設定。

## 快速開始

1. 安裝外掛：

```bash
openclaw plugins install @openclaw/irc
```

2. 至少在 `~/.openclaw/openclaw.json` 中設定主機、暱稱和要加入的頻道：

```json5
{
  channels: {
    irc: {
      enabled: true,
      host: "irc.example.com",
      port: 6697,
      tls: true,
      nick: "openclaw-bot",
      channels: ["#openclaw"],
    },
  },
}
```

3. 啟動／重新啟動閘道：

```bash
openclaw gateway run
```

建議使用私人 IRC 伺服器進行機器人協調。如果你刻意使用公用 IRC 網路，常見選擇包括 Libera.Chat、OFTC 和 Snoonet。避免使用容易猜到的公用頻道來傳輸機器人或機器人群的後端通訊流量。

## 輸入耐久性

OpenClaw 會先將每個已接受的 IRC `PRIVMSG` 寫入其持久性輸入佇列，再執行一般政策檢查和代理程式分派。待處理或可重試的訊息會在閘道重新啟動後保留，並且依各頻道或私人訊息對象維持循序處理。

IRC 不會提供可重播的傳送 ID，也不會重新傳送用戶端中斷連線期間遺漏的訊息。因此，OpenClaw 會指派一個僅在目前 TCP 連線內保持穩定的本機 ID。佇列可保護從本機接受訊息到分派訊息之間的處理區段；它無法復原從未送達 OpenClaw 的訊息，也無法對跨連線的伺服器重送訊息進行去重。

## 連線設定

| 鍵                            | 預設值                        | 備註                                                        |
| ----------------------------- | ----------------------------- | ----------------------------------------------------------- |
| `host`                        | 無（必填）                    | IRC 伺服器主機名稱                                          |
| `port`                        | 使用 TLS 時為 `6697`，純文字為 `6667` | 1-65535                                                     |
| `tls`                         | `true`                        | 僅在刻意使用純文字連線時才設定 `false`              |
| `nick`                        | 無（必填）                    | 機器人暱稱                                                  |
| `username`                    | 暱稱，否則為 `openclaw`       | IRC 使用者名稱                                              |
| `realname`                    | `OpenClaw`                    | 真實名稱／GECOS 欄位                                        |
| `password` / `passwordFile`   | 無                            | 伺服器密碼；檔案必須是一般檔案                              |
| `channels`                    | 無                            | 要加入的頻道（`["#openclaw"]`）                           |
| `accounts` / `defaultAccount` | 無                            | 多帳號設定；環境變數僅會填入預設帳號                        |

## 安全性預設值

- IRC 使用 OpenClaw 操作者管理的轉送 Proxy 路由以外的原始 TCP/TLS 通訊端。在要求所有輸出流量皆通過該轉送 Proxy 的部署中，除非已明確核准 IRC 直接輸出，否則請設定 `channels.irc.enabled=false`。
- `channels.irc.dmPolicy` 預設為 `"pairing"`：未知的私人訊息傳送者會收到配對碼，你可使用 `openclaw pairing approve irc <code>` 核准。
- `channels.irc.groupPolicy` 預設為 `"allowlist"`。
- 使用 `groupPolicy="allowlist"` 時，請設定 `channels.irc.groups` 來定義允許的頻道。
- 除非你刻意接受純文字傳輸，否則請使用 TLS（`channels.irc.tls=true`）。

## 存取控制

IRC 頻道有兩道獨立的「關卡」：

1. **頻道存取權**（`groupPolicy` + `groups`）：機器人是否接受來自該頻道的任何訊息。
2. **傳送者存取權**（`groupAllowFrom`／各頻道的 `groups["#channel"].allowFrom`）：誰可以在該頻道內觸發機器人。

設定鍵：

- 私人訊息允許清單（私人訊息傳送者存取權）：`channels.irc.allowFrom`
- 群組傳送者允許清單（頻道傳送者存取權）：`channels.irc.groupAllowFrom`
- 各頻道控制項（頻道、傳送者和提及規則）：`channels.irc.groups["#channel"]`，包含 `requireMention`、`allowFrom`、`enabled`、`tools`、`toolsBySender`、`skills` 和 `systemPrompt`
- `channels.irc.groupPolicy="open"` 允許未設定的頻道（**預設仍受提及關卡限制**）

允許清單項目應使用穩定的傳送者身分（`nick!user@host`）。
僅比對暱稱的方式可變動，而且只有在 `channels.irc.dangerouslyAllowNameMatching: true` 時才會啟用。

### 常見陷阱：`allowFrom` 用於私人訊息，而非頻道

如果你看到如下記錄：

- `irc: drop group sender alice!ident@host (policy=allowlist)`

……這表示該傳送者無權傳送**群組／頻道**訊息。可透過下列任一方式修正：

- 設定 `channels.irc.groupAllowFrom`（全域套用至所有頻道），或
- 設定各頻道的傳送者允許清單：`channels.irc.groups["#channel"].allowFrom`

範例（允許 `#openclaw` 中的任何人與機器人對話）：

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": { allowFrom: ["*"] },
      },
    },
  },
}
```

## 回覆觸發方式（提及）

即使頻道已獲允許（透過 `groupPolicy` + `groups`），且傳送者也獲允許，OpenClaw 在群組情境中預設仍會使用**提及關卡**。當訊息包含目前連線中的機器人暱稱，或符合你設定的提及模式時，便視為已提及機器人。

這表示除非訊息包含符合機器人的提及模式，否則你可能會看到類似 `drop channel … (missing-mention)` 的記錄。

若要讓機器人在 IRC 頻道中**不需要被提及即可回覆**，請停用該頻道的提及關卡：

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": {
          requireMention: false,
          allowFrom: ["*"],
        },
      },
    },
  },
}
```

或者，若要允許**所有** IRC 頻道（不使用各頻道允許清單），並且仍可在未被提及時回覆：

```json5
{
  channels: {
    irc: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: false, allowFrom: ["*"] },
      },
    },
  },
}
```

## 安全性注意事項（建議公用頻道採用）

如果你在公用頻道中允許 `allowFrom: ["*"]`，任何人都可以向機器人下提示。
若要降低風險，請限制該頻道可使用的工具。

### 頻道中的所有人使用相同工具

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          tools: {
            deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
          },
        },
      },
    },
  },
}
```

### 每位傳送者使用不同工具（擁有者具備更多權限）

使用 `toolsBySender` 對 `"*"` 套用較嚴格的政策，並對你的暱稱套用較寬鬆的政策：

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          toolsBySender: {
            "*": {
              deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
            },
            "id:alice": {
              deny: ["gateway", "nodes", "cron"],
            },
          },
        },
      },
    },
  },
}
```

備註：

- `toolsBySender` 鍵應使用明確的前置詞（`channel:`、`id:`、`e164:`、`username:`、`name:`）。對於 IRC，請搭配傳送者身分值使用 `id:`：使用 `id:alice` 或 `id:alice!~alice@203.0.113.7` 可進行更嚴格的比對。
- 仍接受舊版無前置詞的鍵，但只會以 `id:` 的方式進行比對，並發出淘汰警告。
- 第一個符合的傳送者政策優先；`"*"` 是萬用字元後援政策。

如需進一步瞭解群組存取權與提及關卡（以及兩者如何互動），請參閱：[/channels/groups](/zh-TW/channels/groups)。

## NickServ

若要在連線後向 NickServ 識別身分：

```json5
{
  channels: {
    irc: {
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "your-nickserv-password",
      },
    },
  },
}
```

設定密碼後，NickServ 身分識別預設都會執行（只有在 `enabled` 為 `false` 時才會停用）。`service` 預設為 `NickServ`；`passwordFile` 可替代行內的 `password`。

連線時選擇性執行一次註冊（`register: true` 需要 `registerEmail`）：

```json5
{
  channels: {
    irc: {
      nickserv: {
        register: true,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

暱稱註冊完成後，請停用 `register`，以避免重複嘗試 REGISTER。

## 環境變數

預設帳號支援：

- `IRC_HOST`
- `IRC_PORT`
- `IRC_TLS`
- `IRC_NICK`
- `IRC_USERNAME`
- `IRC_REALNAME`
- `IRC_PASSWORD`
- `IRC_CHANNELS`（以逗號分隔）
- `IRC_NICKSERV_PASSWORD`
- `IRC_NICKSERV_REGISTER_EMAIL`

無法從工作區 `.env` 設定 `IRC_HOST`；請參閱[工作區 `.env` 檔案](/zh-TW/gateway/security)。

## 疑難排解

- 如果機器人已連線，但從不在頻道中回覆，請確認 `channels.irc.groups`，**並且**檢查提及關卡是否捨棄了訊息（`missing-mention`）。如果你希望它不需被點名即可回覆，請為該頻道設定 `requireMention:false`。
- 如果登入失敗，請確認暱稱是否可用，以及伺服器密碼是否正確。
- 如果自訂網路上的 TLS 連線失敗，請確認主機／連接埠和憑證設定。

## 相關內容

- [頻道概覽](/zh-TW/channels) — 所有支援的頻道
- [配對](/zh-TW/channels/pairing) — 私人訊息驗證和配對流程
- [群組](/zh-TW/channels/groups) — 群組聊天行為和提及關卡
- [頻道路由](/zh-TW/channels/channel-routing) — 訊息的工作階段路由
- [安全性](/zh-TW/gateway/security) — 存取模型和強化措施
