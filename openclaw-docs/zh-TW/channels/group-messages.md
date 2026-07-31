---
read_when:
    - 專門設定 WhatsApp 群組
    - 變更 WhatsApp 啟用模式（`mention` 與 `always`）
    - 調整 WhatsApp 群組工作階段金鑰或待處理訊息的上下文
sidebarTitle: WhatsApp groups
summary: WhatsApp 群組訊息處理 — 啟用、允許清單、工作階段與內容脈絡注入
title: WhatsApp 群組訊息
x-i18n:
    generated_at: "2026-07-26T07:08:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7325dd3ae64d7abca8c1de0504f294ae280394fa5dd336d2532c5eaefcb03828
    source_path: channels/group-messages.md
    workflow: 16
---

關於跨頻道群組模型（Discord、iMessage、Matrix、Microsoft Teams、QQ Bot、Signal、Slack、Telegram、WhatsApp、Zalo），請參閱[群組](/zh-TW/channels/groups)。本頁說明該模型之上的 WhatsApp 特有行為：啟用、群組允許清單、各群組工作階段金鑰，以及待處理訊息的情境注入。

目標：讓 OpenClaw 留在 WhatsApp 群組中，僅在被點名時喚醒，並讓該討論串與個人私訊工作階段分開。

<Note>
`agents.entries.*.groupChat.mentionPatterns` 與其他頻道的提及閘控共用。若為多代理程式設定，請針對各代理程式設定，或使用 `messages.groupChat.mentionPatterns` 作為全域備援。若兩者皆未設定，系統會從代理程式身分的名稱／表情符號衍生模式。
</Note>

## 行為

- 啟用模式：`mention`（預設）或 `always`。`mention` 要求點名：真正的 WhatsApp @提及（`mentionedJids`）、已設定的規則運算式模式、文字中任何位置出現機器人的 E.164 數字，或引用回覆機器人的其中一則訊息（共用號碼的自我聊天設定除外）。`always` 會在每則訊息時喚醒代理程式，但注入的群組提示會要求它僅在能帶來價值時回覆，否則傳回完全相同的靜默權杖 `NO_REPLY`（不區分大小寫）。預設值來自設定（`channels.whatsapp.groups` `requireMention`），並可透過 `/activation` 針對各群組覆寫。
- 群組允許清單：設定 `channels.whatsapp.groups` 後，僅接受列出的群組 JID（包含 `"*"` 即可允許全部）；未列出群組的訊息會遭捨棄，並在記錄中提供提示。
- 群組原則：`channels.whatsapp.groupPolicy` 控制是否接受群組訊息（`open|disabled|allowlist`）。`allowlist` 使用 `channels.whatsapp.groupAllowFrom`（備援：明確設定的 `channels.whatsapp.allowFrom`）。預設為 `allowlist`（在你新增傳送者前一律封鎖）。
- 各群組工作階段：工作階段金鑰的格式類似 `agent:<agentId>:whatsapp:group:<jid>`（非預設帳號會附加 `:thread:whatsapp-account-<accountId>`），因此 `/verbose on`、`/trace on` 或 `/think high` 等指令（以獨立訊息傳送）僅作用於該群組；個人私訊狀態不受影響。
- 情境注入：**僅待處理**且_未_觸發執行的群組訊息（預設 50 則）會加上 `[Chat messages since your last reply - for context]` 前置標頭，而觸發訊息行則位於 `[Current message - respond to this]` 下。執行後會清除待處理視窗；已在工作階段中的訊息不會重複注入。
- 傳送者歸屬：每個群組訊息行都會在訊息封套內包含傳送者標籤，例如 `[WhatsApp <groupJid> <timestamp>] Alice (+447700900123): text`；傳送者身分以及群組主旨／成員資訊也會一併放入不受信任的對話中繼資料區塊。
- 限時／僅限檢視一次：系統會先解除包裝再擷取文字／提及，因此其中的點名仍可觸發。
- 群組系統提示：群組工作階段的第一輪（以及 `/activation` 變更模式後的任何一輪）會將啟用指引注入系統提示（`Activation: trigger-only ...` 或 `Activation: always-on ...`，外加「回應特定傳送者」）。系統一律會包含持續性的群組聊天傳送指引（「你正在 WhatsApp 群組聊天中……」）。

## 設定範例（WhatsApp）

即使 WhatsApp 從文字本文移除可見的 `@`，仍可讓顯示名稱點名生效：

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
      },
      historyLimit: 50, // 待處理群組情境視窗（預設 50）
    },
  },
  agents: {
    entries: {
      main: {
        groupChat: {
          mentionPatterns: ["@?openclaw", "\\+?15555550123"],
        },
      },
    },
  },
}
```

注意事項：

- 規則運算式不區分大小寫，並使用與其他設定規則運算式介面相同的安全規則運算式防護；無效模式及不安全的巢狀重複會被忽略。
- 當有人點選聯絡人時，WhatsApp 仍會透過 `mentionedJids` 傳送標準提及，因此很少需要號碼備援，但它是實用的安全網。
- 待處理情境視窗依 `channels.whatsapp.accounts.<id>.historyLimit` → `channels.whatsapp.historyLimit` → `messages.groupChat.historyLimit` → 50 的順序解析。

### 啟用命令（僅限擁有者）

使用群組聊天命令：

- `/activation mention`
- `/activation always`

只有擁有者號碼（來自 `channels.whatsapp.allowFrom`；若未設定，則為機器人本身的 E.164 號碼）可以變更此設定；其他人傳送的 `/activation` 會被忽略，僅儲存為情境。請在群組中將 `/status` 作為獨立訊息傳送，以查看目前的啟用模式。

## 使用方式

1. 將你的 WhatsApp 帳號（執行 OpenClaw 的帳號）加入群組。
2. 傳送 `@openclaw ...`（或包含號碼）。除非設定 `groupPolicy: "open"`，否則只有允許清單中的傳送者可以觸發。
3. 代理程式提示包含待處理群組情境及標示傳送者的訊息行，讓它能回應正確的人。
4. 工作階段指令（`/verbose on`、`/trace on`、`/think high`、`/new` 或 `/reset`、`/compact`）僅套用至該群組的工作階段；請將它們作為獨立訊息傳送，以便系統登錄。你的個人私訊工作階段會維持獨立。

## 測試／驗證

- 手動冒煙測試：
  - 在群組中傳送 `@openclaw` 點名，並確認回覆有提及傳送者名稱。
  - 傳送第二次點名，確認包含歷程記錄區塊，然後確認其在下一輪已清除。
- 檢查閘道記錄（使用 `--verbose` 執行），尋找顯示 `from: <groupJid>` 及標示傳送者本文的 `inbound web message` 項目。

## 已知注意事項

- 心跳偵測在代理程式的主要工作階段中執行；群組工作階段絕不會執行心跳偵測。
- 回音抑制會依工作階段記住合併後的提示（歷程記錄 + 目前訊息），因此機器人自己送達的訊息不會再次觸發；完全相同的重複批次可能會被視為回音而略過。
- 工作階段儲存區項目會以 `agent:<agentId>:whatsapp:group:<jid>` 的形式出現在各代理程式的 SQLite 工作階段儲存區中；缺少項目僅表示該群組尚未觸發執行。
- 輸入指示器遵循 `agents.entries.*.typingMode`／`agents.defaults.typingMode`。當可見回覆選用僅訊息工具模式時，預設會立即開始顯示輸入狀態，讓群組成員即使未發布自動最終回覆，也能看到代理程式正在處理。明確的輸入模式設定仍具有優先權。

## 相關內容

- [群組](/zh-TW/channels/groups)
- [頻道路由](/zh-TW/channels/channel-routing)
- [廣播群組](/zh-TW/channels/broadcast-groups)
