---
read_when:
    - 變更頻道路由或收件匣行為
summary: 各頻道（WhatsApp、Telegram、Discord、Slack）的路由規則與共用上下文
title: 頻道路由
x-i18n:
    generated_at: "2026-07-26T07:32:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa03f04a55015bf17e0fe1f3a9bc422875124bb64af5891c898a98bc6917d9e8
    source_path: channels/channel-routing.md
    workflow: 16
---

# 頻道與路由

OpenClaw 會將回覆**傳回訊息來源的頻道**。模型不會選擇頻道；路由是確定性的，並由主機設定控制。在預設的 DM 範圍下，來自每個頻道的直接訊息都會匯入代理程式的[主要工作階段](/zh-TW/concepts/main-session)。

## 關鍵術語

- **頻道**：隨附的頻道外掛，例如 `discord`、`googlechat`、`imessage`、`irc`、`line`、`signal`、`slack`、`telegram` 或 `whatsapp`，以及已安裝的外掛頻道。`webchat` 是內部 WebChat UI 頻道，不是可設定的傳出頻道。
- **AccountId**：各頻道的帳號執行個體（若支援）。
- 選用的頻道預設帳號：`channels.<channel>.defaultAccount` 會選擇
  當傳出路徑未指定 `accountId` 時所使用的帳號。
  - 在多帳號設定中，若設定了兩個以上的帳號，請設定明確的預設值（`defaultAccount` 或名為 `default` 的帳號）。若未設定，備援路由可能會選取第一個正規化的帳號 ID。
- **AgentId**：隔離的工作區 + 工作階段儲存區（「大腦」）。
- **SessionKey**：用於儲存上下文及控制並行處理的分組鍵。

## 傳出目標前綴

明確的傳出目標可以包含提供者前綴，例如 `telegram:123` 或 `tg:123`。只有當所選頻道為 `last` 或尚未解析，且載入的外掛宣告支援該前綴時，核心才會將該前綴視為頻道選擇提示。若呼叫端已明確選擇頻道，提供者前綴必須與該頻道相符；例如將 WhatsApp 傳遞至 `telegram:123` 的跨頻道組合，會在外掛特定的目標正規化之前失敗。

`channel:<id>`、`user:<id>`、`room:<id>`、`thread:<id>`、`imessage:<handle>` 和 `sms:<number>` 等目標種類與服務前綴會保留在所選頻道的語法內。它們本身不會選擇提供者。

## 工作階段鍵格式（範例）

直接訊息預設會合併至代理程式的**主要**工作階段：

- `agent:<agentId>:<mainKey>`（預設：`agent:main:main`）

`session.dmScope` 控制 DM 合併：`main`（預設）共用一個主要
工作階段，而 `per-peer`、`per-channel-peer` 和 `per-account-channel-peer`
會將 DM 保留在不同的工作階段中。路由繫結可透過 `bindings[].session.dmScope`
覆寫其相符對等端的範圍。

即使直接訊息的對話記錄與主要工作階段共用，沙箱與
工具原則仍會針對外部 DM 使用衍生的各帳號直接聊天執行階段鍵，
因此不會將源自頻道的訊息視為本機主要工作階段執行。

群組和頻道仍會依頻道各自隔離：

- 群組：`agent:<agentId>:<channel>:group:<id>`
- 頻道／聊天室：`agent:<agentId>:<channel>:channel:<id>`

討論串：

- Slack／Discord 討論串會將 `:thread:<threadId>` 附加至基礎鍵。
- Telegram 論壇主題會將 `:topic:<topicId>` 嵌入群組鍵中。

範例：

- `agent:main:telegram:group:-1001234567890:topic:42`
- `agent:main:discord:channel:123456:thread:987654`

## 主要 DM 路由固定

當 `session.dmScope` 為 `main` 時，直接訊息可能會共用一個主要工作階段。
為避免工作階段的 `lastRoute` 被非擁有者的 DM 覆寫，
當下列所有條件皆成立時，OpenClaw 會從 `allowFrom` 推斷固定擁有者：

- `allowFrom` 恰好有一個非萬用字元項目。
- 該項目可正規化為該頻道的具體傳送者 ID。
- 傳入 DM 的傳送者與該固定擁有者不符。

在此不相符的情況下，OpenClaw 仍會記錄傳入的工作階段中繼資料，但會
略過更新主要工作階段的 `lastRoute`。

## 受防護的傳入記錄

當受防護路徑不得建立新的 OpenClaw 工作階段時，頻道外掛可將傳入工作階段記錄標記為 `createIfMissing: false`。
在此模式下，OpenClaw 可以更新現有工作階段的中繼資料和 `lastRoute`，但不會
只因觀察到訊息就建立僅供路由使用的工作階段項目。

## 路由規則（如何選擇代理程式）

路由會為每則傳入訊息選擇**一個代理程式**：

1. **精確對等端比對**（`bindings` 搭配 `peer.kind` + `peer.id`）。
2. **父對等端比對**（討論串繼承）。
3. **對等端萬用字元比對**（對某個對等端種類使用 `peer.id: "*"`）。
4. **伺服器 + 角色比對**（Discord），透過 `guildId` + `roles`。
5. **伺服器比對**（Discord），透過 `guildId`。
6. **團隊比對**（Slack），透過 `teamId`。
7. **帳號比對**（頻道上的 `accountId`）。
8. **頻道比對**（該頻道上的任何帳號，`accountId: "*"`）。
9. **預設代理程式**（`agents.entries.*.default`，否則為清單中的第一個項目，最後備援至 `main`）。

當繫結包含多個比對欄位（`peer`、`guildId`、`teamId`、`roles`）時，該繫結必須**符合所有提供的欄位**才會套用。

相符的代理程式會決定使用哪個工作區和工作階段儲存區。

## 廣播群組（執行多個代理程式）

廣播群組可讓你在 OpenClaw **通常會回覆時**，為相同的對等端執行**多個代理程式**（例如：在 WhatsApp 群組中，通過提及／啟用管控後）。

設定：

```json5
{
  broadcast: {
    strategy: "parallel",
    "120363403215116621@g.us": ["alfred", "baerbel"],
    "+15555550123": ["support", "logger"],
  },
}
```

另請參閱：[廣播群組](/zh-TW/channels/broadcast-groups)。

## 設定概覽

- `agents.entries`：具名代理程式定義（工作區、模型等）。
- `bindings`：將傳入頻道／帳號／對等端對應至代理程式。

範例：

```json5
{
  agents: {
    list: [{ id: "support", name: "Support", workspace: "~/.openclaw/workspace-support" }],
  },
  bindings: [
    { match: { channel: "slack", teamId: "T123" }, agentId: "support" },
    { match: { channel: "telegram", peer: { kind: "group", id: "-100123" } }, agentId: "support" },
  ],
}
```

## 工作階段儲存

執行階段工作階段資料列位於每個代理程式在狀態
目錄下的 SQLite 資料庫中（預設為 `~/.openclaw`）：

- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`

較舊的安裝可能在 `~/.openclaw/agents/<agentId>/sessions/` 下有舊版逐字稿 JSONL 檔案和 `sessions.json` 資料列
儲存區。閘道啟動和
`openclaw doctor --fix` 會自動將使用中的舊版資料列／歷程記錄匯入 SQLite。
當你需要明確的遷移證據時，請使用 `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` 和
[Doctor](/zh-TW/cli/doctor#session-sqlite-migration) 驗證順序。
你仍可透過 `session.store` 和 `{agentId}`
範本選取舊版儲存區路徑，以供遷移和離線維護工作流程使用。

閘道和 ACP 工作階段探索也會掃描預設 `agents/` 根目錄，以及範本化 `session.store` 根目錄下，
位於磁碟上的代理程式儲存區。探索到的儲存區必須保留在該解析後的代理程式根目錄內，
並使用一般的舊版 `sessions.json` 檔案。符號連結和根目錄以外的路徑會被忽略。

## WebChat 行為

WebChat 會連接至**所選代理程式**，預設使用該代理程式的主要
工作階段。因此，WebChat 可讓你在同一處查看該
代理程式的跨頻道上下文。

## 回覆上下文

傳入回覆會包含：

- 可用時包含 `ReplyToId`、`ReplyToBody` 和 `ReplyToSender`。
- 引用的上下文會以 `[Replying to ...]` 區塊附加至 `Body`。

所有頻道的行為皆一致。

## 相關內容

- [群組](/zh-TW/channels/groups)
- [廣播群組](/zh-TW/channels/broadcast-groups)
- [配對](/zh-TW/channels/pairing)
