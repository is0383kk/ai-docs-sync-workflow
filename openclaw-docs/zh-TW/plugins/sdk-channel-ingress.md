---
read_when:
    - 建置或遷移訊息通道外掛
    - 變更私訊或群組允許清單、路由閘門、命令驗證、事件驗證或提及啟用設定
    - 審查頻道輸入遮蔽或 SDK 相容性邊界
sidebarTitle: Channel Ingress
summary: 用於傳入訊息授權的實驗性頻道輸入 API
title: 頻道輸入 API
x-i18n:
    generated_at: "2026-07-26T07:52:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 60feecb7bcf203cf37d2543a7855e89b5bfb2eb9d8263d804219e140facb8fc6
    source_path: plugins/sdk-channel-ingress.md
    workflow: 16
---

頻道入口是針對傳入頻道事件的實驗性存取控制邊界。外掛擁有平台事實與副作用；核心擁有一般政策：私訊／群組允許清單、配對儲存區私訊項目、路由閘門、命令閘門、事件授權、提及啟用、遮蔽診斷資訊，以及准入。

接收路徑請使用 `openclaw/plugin-sdk/channel-ingress-runtime`。

## 執行階段解析器

```ts
import {
  defineStableChannelIngressIdentity,
  resolveChannelMessageIngress,
} from "openclaw/plugin-sdk/channel-ingress-runtime";

const identity = defineStableChannelIngressIdentity({
  key: "platform-user-id",
  normalize: normalizePlatformUserId,
  sensitivity: "pii",
});

const result = await resolveChannelMessageIngress({
  channelId: "my-channel",
  accountId,
  identity,
  subject: { stableId: platformUserId },
  conversation: { kind: isGroup ? "group" : "direct", id: conversationId },
  event: { kind: "message", authMode: "inbound", mayPair: !isGroup },
  policy: {
    dmPolicy: config.dmPolicy,
    groupPolicy: config.groupPolicy,
    groupAllowFromFallbackToAllowFrom: true,
  },
  allowFrom: config.allowFrom,
  groupAllowFrom: config.groupAllowFrom,
  accessGroups: cfg.accessGroups,
  route,
  readStoreAllowFrom,
  command: hasControlCommand ? { allowTextCommands: true, hasControlCommand } : undefined,
});
```

請勿預先計算有效允許清單、命令擁有者或命令群組。解析器會從原始允許清單、儲存區回呼、路由描述元、存取群組、政策及對話種類推導這些項目。

## 結果

內建外掛應直接使用現代投影：

| 欄位              | 意義                                                            |
| ------------------ | ------------------------------------------------------------------ |
| `ingress`          | 依序執行的閘門決策與准入                                |
| `senderAccess`     | 僅限傳送者／對話授權                             |
| `routeAccess`      | 路由與路由傳送者投影                                  |
| `commandAccess`    | 命令授權；未執行命令閘門時為 `requested: false` |
| `activationAccess` | 提及／啟用結果                                          |

事件授權仍可透過依序執行的 `ingress.graph` 與具決定性的 `ingress.reasonCode` 取得；不會發出獨立的事件投影。

已棄用的第三方 SDK 輔助函式可在內部重建較舊的結構。新的內建接收路徑不應將現代結果轉換回本機 DTO。

## 存取群組

`accessGroup:<name>` 項目會保持遮蔽。核心會自行解析靜態 `message.senders` 群組，且僅針對需要查詢平台的動態群組呼叫 `resolveAccessGroupMembership`。缺少、不支援及失敗的群組皆採取拒絕存取的安全預設。

## 事件模式

| `authMode`       | 意義                                          |
| ---------------- | ------------------------------------------------ |
| `inbound`        | 一般傳入傳送者閘門                      |
| `command`        | 回呼或限定範圍按鈕的命令閘門    |
| `origin-subject` | 操作者必須符合原始訊息主體    |
| `route-only`     | 僅針對限定路由範圍的受信任事件套用路由閘門 |
| `none`           | 外掛擁有的內部事件會略過共用授權  |

回應、按鈕、回呼及原生命令請使用 `mayPair: false`。

## 路由與啟用

針對聊天室、主題、伺服器、討論串或巢狀路由政策，請使用路由描述元：

```ts
route: {
  id: "room",
  allowed: roomAllowed,
  enabled: roomEnabled,
  senderPolicy: "replace",
  senderAllowFrom: roomAllowFrom,
  blockReason: "room_sender_not_allowlisted",
}
```

當外掛具有數個選用路由描述元時，請使用 `channelIngressRoutes(...)`；它會篩除已停用的分支，同時維持路由事實的通用性，並依各描述元的 `precedence` 排序。

提及閘門是一種啟用閘門。未命中提及時會傳回 `admission: "skip"`，讓回合核心不會處理僅供觀察的回合。大多數頻道應將啟用閘門置於傳送者與命令閘門之後。若公開聊天介面必須先讓未提及的流量保持安靜，以免產生傳送者允許清單雜訊，則可在停用文字命令略過功能時選用 `activation.order: "before-sender"`。具有隱含啟用的頻道（例如機器人討論串中的回覆）會使用 `resolveChannelImplicitMentions(...)` 解析 `channels.defaults.implicitMentions` 以及頻道與帳號覆寫，然後將結果以 `activation.implicitMentions` 傳入。投影的 `activationAccess.shouldBypassMention` 會回報命令或隱含啟用何時略過了明確提及。

## 遮蔽

原始傳送者值與原始允許清單項目僅供解析器輸入使用。它們不得出現在已解析狀態、決策、診斷資訊、快照或相容性事實中。請使用不透明的主體 ID、項目 ID、路由 ID 及診斷 ID。

## 驗證

```bash
pnpm test src/channels/message-access/message-access.test.ts src/plugin-sdk/channel-ingress-runtime.test.ts
pnpm plugin-sdk:api:check
```
