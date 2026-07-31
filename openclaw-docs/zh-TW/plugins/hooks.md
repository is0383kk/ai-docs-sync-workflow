---
read_when:
    - 你正在建置一個需要 before_tool_call、before_agent_reply、訊息鉤子或生命週期鉤子的外掛
    - 你需要封鎖、改寫或要求核准來自外掛的工具呼叫
    - 你正在內部鉤子與外掛鉤子之間做選擇
    - 你正在將 OpenClaw 的排程喚醒作業投射至外部主機排程器中
summary: 外掛鉤子：攔截代理程式、工具、訊息、工作階段及閘道生命週期事件
title: 外掛鉤子
x-i18n:
    generated_at: "2026-07-26T08:19:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 95d7ea2f7bfe26b5904ea3cd8f8db85ffd8163af58e03ec56d11eee992bc13d2
    source_path: plugins/hooks.md
    workflow: 16
---

外掛鉤子是 OpenClaw 外掛的處理程序內擴充點：可檢查或
變更代理執行、工具呼叫、訊息流程、工作階段生命週期、子代理
路由、安裝或閘道啟動。

若要讓由操作人員安裝的小型 `HOOK.md` 指令碼回應命令和閘道事件，例如 `/new`、
`/reset`、`/stop`、`agent:bootstrap` 或 `gateway:startup`，請改用[內部鉤子](/zh-TW/automation/hooks)。

## 快速開始

從外掛進入點使用 `api.on(...)` 註冊具型別的鉤子：

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "tool-preflight",
  name: "Tool Preflight",
  register(api) {
    api.on(
      "before_tool_call",
      async (event) => {
        if (event.toolName !== "web_search") {
          return;
        }

        return {
          requireApproval: {
            title: "Run web search",
            description: `Allow search query: ${String(event.params.query ?? "")}`,
            severity: "info",
            timeoutMs: 60_000,
          },
        };
      },
      { priority: 50 },
    );
  },
});
```

可傳回決策或修改內容的處理常式，會依 `priority` 由高至低循序執行；優先順序相同的處理常式會維持註冊順序。
僅觀察的處理常式會平行執行，而發送後不等待結果的觀察分派可能與後續事件重疊。請勿使用優先順序排列
觀察副作用。

`api.on(name, handler, opts?)` 接受：

| 選項      | 效果                                                                                                                                                                                            |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `priority`  | 執行順序；數值越高越先執行。                                                                                                                                                                      |
| `timeoutMs` | 每個鉤子的等待時間預算。期限屆滿時，OpenClaw 會停止等待該處理常式並繼續執行。這不會取消處理常式或其副作用。省略時，使用執行器預設的每鉤子逾時。 |

操作人員無須修補外掛程式碼即可設定鉤子時間預算：

```json
{
  "plugins": {
    "entries": {
      "my-plugin": {
        "hooks": {
          "timeoutMs": 30000,
          "timeouts": {
            "before_prompt_build": 90000,
            "agent_end": 60000
          }
        }
      }
    }
  }
}
```

`hooks.timeouts.<hookName>` 會覆寫 `hooks.timeoutMs`，後者會覆寫
外掛編寫者提供的 `api.on(..., { timeoutMs })` 值。每個值都必須是
不超過 600000 ms 的正整數。對已知速度較慢的鉤子，建議使用個別鉤子覆寫，避免單一外掛在所有位置都取得較長的時間預算。

逾時的處理常式 Promise 會繼續執行，因為鉤子回呼不會
收到取消訊號。即使該外掛工作仍在進行中，鉤子分派仍可釋放其閘道
許可。擁有長時間執行工作的外掛必須自行提供取消與關閉生命週期。

輸出修改鉤子 `message_sending` 和 `reply_payload_sending` 對每個處理常式使用
15 秒的預設值。若其中一個逾時，OpenClaw 會記錄外掛錯誤
並使用最新承載繼續執行，讓序列化傳遞通道得以完成。若外掛刻意在
傳遞前執行較慢的工作，請設定較大的個別鉤子時間預算。

使用 `createReplyDispatcher` 的頻道外掛同樣可透過 `beforeDeliverOptions: { timeoutMs }` 宣告較大的
正數個別階段時間預算，或在
使用 `dispatcher.appendBeforeDeliver(handler, { timeoutMs })` 附加工作時宣告。
若擁有者未宣告時間預算，這些回呼會使用相同的 15 秒
預設值，避免停滯的回呼持續占用序列化傳遞通道。

每個鉤子都會收到 `event.context.pluginConfig`，也就是
註冊該處理常式之外掛的解析後設定。OpenClaw 會針對每個處理常式注入此設定，
且不會修改其他外掛看到的共用事件物件。

## 鉤子目錄

鉤子依其擴充的介面分組。**粗體**名稱接受決策
結果（封鎖、取消、覆寫或要求核准）；其餘僅供觀察。

**代理回合**

| 鉤子                            | 用途                                                                                  |
| ------------------------------- | ---------------------------------------------------------------------------------------- |
| `before_model_resolve`          | 在載入工作階段訊息前覆寫供應商或模型                                  |
| `agent_turn_prepare`            | 取用排入佇列的外掛回合注入，並在提示鉤子前加入同回合情境      |
| `before_prompt_build`           | 在呼叫模型前加入動態情境或系統提示文字                          |
| **`before_agent_run`**          | 在提交模型前檢查最終提示和工作階段訊息；可封鎖執行 |
| **`before_agent_reply`**        | 使用合成回覆或靜默方式短路模型回合                           |
| **`before_agent_finalize`**     | 檢查自然產生的最終答案，並要求模型再執行一次                         |
| `agent_end`                     | 觀察最終訊息、成功狀態和執行時間                                  |
| `heartbeat_prompt_contribution` | 為背景監控和生命週期外掛加入僅限心跳偵測的情境                  |

**對話觀察**

| 鉤子                                      | 用途                                                                                                            |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `model_call_started` / `model_call_ended` | 經清理的供應商／模型呼叫中繼資料：時間、結果、限定長度的請求 ID 雜湊。不含提示或回應內容。 |
| `llm_input`                               | 供應商輸入：系統提示、提示、歷史記錄                                                                     |
| `llm_output`                              | 供應商輸出、用量，以及可用時解析後的 `contextTokenBudget`                                       |

**工具**

| 鉤子                       | 用途                                                   |
| -------------------------- | --------------------------------------------------------- |
| **`before_tool_call`**     | 改寫工具參數、封鎖執行或要求核准 |
| `after_tool_call`          | 觀察工具結果、錯誤和持續時間                |
| `resolve_exec_env`         | 將外掛擁有的環境變數提供給 `exec`   |
| **`tool_result_persist`**  | 改寫由工具結果產生的助理訊息 |
| **`before_message_write`** | 檢查或封鎖進行中的訊息寫入（少見）      |

**訊息與傳遞**

| 鉤子                            | 用途                                                           |
| ------------------------------- | ----------------------------------------------------------------- |
| **`inbound_claim`**             | 在代理路由前接管輸入訊息（合成回覆） |
| **`channel_pairing_requested`** | 觀察新建立的私訊配對要求                         |
| `message_received`              | 觀察輸入內容、傳送者、討論串和中繼資料             |
| **`message_sending`**           | 改寫輸出內容或取消傳遞                       |
| **`reply_payload_sending`**     | 在傳遞前修改或取消標準化回覆承載        |
| `message_sent`                  | 觀察輸出傳遞成功或失敗                      |
| **`before_dispatch`**           | 在交接給頻道前檢查或改寫輸出分派    |
| **`reply_dispatch`**            | 參與最終回覆分派流水線                  |

**工作階段與壓縮**

| 鉤子                                     | 用途                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `session_start` / `session_end`          | 追蹤工作階段生命週期邊界。`reason` 是 `new`、`reset`、`idle`、`daily`、`compaction`、`deleted`、`shutdown`、`restart` 或 `unknown` 之一。當處理程序在仍有作用中工作階段時停止或重新啟動，`shutdown`/`restart` 會由閘道關閉完成器觸發，讓外掛（記憶、逐字稿儲存區）可完成幽靈資料列，而非在重新啟動後仍讓其保持開啟。完成器有時間上限，因此緩慢的外掛無法阻擋 SIGTERM/SIGINT。 |
| `before_compaction` / `after_compaction` | 觀察壓縮週期或加上註解                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `before_reset`                           | 觀察工作階段重設事件（`/reset`、程式化重設）                                                                                                                                                                                                                                                                                                                                                                                                     |

對於使用 `parentSessionKey` 和 `emitCommandHooks: true` 的 `sessions.create` 呼叫，不同的子項目一律會收到 `session_start`。呼叫端使用 `succeedsParent` 宣告父項目是否也會收到終止 `session_end`：`true` 表示後繼者，`false` 表示平行子項目。省略時會保留舊版的父項目輪替行為。在這兩種情況下，`command:new` 和 `before_reset` 鉤子仍會描述要求的 `/new` 動作。

**子代理**

- `subagent_spawned` / `subagent_ended` - 觀察子代理的啟動與完成。
- `subagent_delivery_target` - 當沒有核心工作階段繫結可投射路由時，用於傳遞完成結果的相容性掛鉤。
- `subagent_spawning` - 已棄用的相容性掛鉤。核心現在會在 `subagent_spawned` 觸發前，透過頻道工作階段繫結配接器準備 `thread: true` 子代理繫結。
- `subagent_spawned` 會在 OpenClaw 於啟動前解析出子工作階段的原生模型時，包含 `resolvedModel` 和 `resolvedProvider`。
- `subagent_ended` 會攜帶 `targetSessionKey`（身分識別資訊 — 與 `subagent_spawned.childSessionKey` 相符）、`targetKind`（`"subagent"` 或 `"acp"`）、`reason`、選用的 `outcome`（`"ok"`、`"error"`、`"timeout"`、`"killed"`、`"reset"` 或 `"deleted"`）、選用的 `error`、`runId`、`endedAt`、`accountId` 和 `sendFarewell`。它**不會**包含 `agentId` 或 `childSessionKey`；請使用 `targetSessionKey` 與相符的 `subagent_spawned` 事件建立關聯。

**生命週期**

| 掛鉤                             | 用途                                                                                              |
| -------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `gateway_start` / `gateway_stop` | 隨閘道啟動或停止外掛所擁有的服務                                                 |
| `deactivate`                     | `gateway_stop` 的已棄用相容性別名；新外掛請使用 `gateway_stop`                 |
| `cron_reconciled`                | 在啟動或重新載入後，依據完整的閘道排程狀態進行協調                            |
| `cron_changed`                   | 觀察閘道所擁有的排程生命週期變更（新增、更新、移除、啟動、完成、已排程） |
| **`before_install`**             | 從已載入的外掛執行階段檢查暫存的技能或外掛安裝內容                         |

### 頻道配對要求

當外掛需要在未配對的私訊傳送者建立待處理的配對要求後通知操作人員或
寫入稽核記錄時，請使用 `channel_pairing_requested`。
此掛鉤會在要求建立時派送；配對回覆的頻道傳遞
不會因緩慢或失敗的掛鉤處理常式而延遲。

```typescript
api.on("channel_pairing_requested", async (event) => {
  await notifyOperator({
    text: `來自 ${event.senderId} 的新 ${event.channel} 配對要求：${event.code}`,
  });
});
```

此掛鉤僅供觀察。它不會核准、拒絕、抑制或重寫
配對回覆。承載資料包含頻道、選用的 `accountId`、
頻道範圍的 `senderId`、配對 `code` 和頻道中繼資料。請將
配對碼視為有效且僅能使用一次的核准認證資訊，並且只將它傳遞至
受信任的操作人員接收端。請將 `metadata` 視為由不受信任的傳送者提供的身分識別
文字。此掛鉤不包含傳入訊息的本文或媒體。

## 偵錯執行階段掛鉤

使用 `before_model_resolve` 可為代理執行回合切換提供者或模型 — 它會
在模型解析前執行。`llm_output` 僅在某次模型嘗試
產生助理輸出後執行。

若要證明工作階段實際使用的模型，請檢查執行階段註冊項目，然後
使用 `openclaw sessions` 或閘道工作階段／狀態介面。若要偵錯
提供者承載資料，請使用 `--raw-stream` 和
`--raw-stream-path <path>` 啟動閘道，將原始模型串流事件寫入 jsonl 檔案。

## 工具呼叫政策

`before_tool_call` 會接收：

- `event.toolName`
- `event.params`
- 選用的 `event.toolKind` 和 `event.toolInputKind`，它們是由主機權威判定、
  用於刻意共用名稱之工具的區別項；例如，外層
  程式碼模式的 `exec` 呼叫使用 `toolKind: "code_mode_exec"`，並在
  已知輸入語言時包含
  `toolInputKind: "javascript" | "typescript"`
- 選用的 `event.derivedPaths`，由主機盡力推導的目標路徑提示，
  適用於 `apply_patch` 等已知工具封裝；這些路徑可能
  不完整，或過度估計工具實際會觸及的範圍（例如，
  輸入格式錯誤或不完整時）
- 選用的 `event.runId`
- 選用的 `event.toolCallId`
- 內容欄位，例如 `ctx.agentId`、`ctx.sessionKey`、`ctx.sessionId`、
  `ctx.runId`、`ctx.toolKind`、`ctx.toolInputKind`，以及診斷用的 `ctx.trace`
- 選用的 `ctx.requester`，即由主機推導、發起目前
  訊息執行的要求者。它可包含 `channel`、`accountId`、`senderId`、
  `senderIsOwner` 和提供者原生的 `roleIds`。缺少的欄位代表尚未證實，
  而非保證為否；若政策要求這些欄位，請採取封閉式失敗。

它可以傳回：

```typescript
type BeforeToolCallResult = {
  params?: Record<string, unknown>;
  block?: boolean;
  blockReason?: string;
  requireApproval?: {
    title: string;
    description: string;
    severity?: "info" | "warning" | "critical";
    timeoutMs?: number;
    /** @deprecated 未解決的核准一律拒絕。 */
    timeoutBehavior?: "allow" | "deny";
    allowedDecisions?: Array<"allow-once" | "allow-always" | "deny">;
    pluginId?: string;
    onResolution?: (
      decision: "allow-once" | "allow-always" | "deny" | "timeout" | "cancelled",
    ) => Promise<void> | void;
  };
};
```

型別化生命週期掛鉤的防護行為：

- `block: true` 是終止性決定，並會略過優先順序較低的處理常式。
- `block: false` 會被視為未做決定。
- `params` 會重寫執行工具時使用的參數。
- `requireApproval` 會暫停代理執行，並透過外掛
  核准向使用者詢問。`/approve` 可同時核准執行和外掛核准。在 Codex
  app-server 報告模式的原生 `PreToolUse` 中繼中，這會交由
  相符的 app-server 核准要求處理；請參閱
  [Codex 控制框架執行階段](/zh-TW/plugins/codex-harness-runtime#hook-boundaries)。
- 即使優先順序較高的掛鉤已要求核准，優先順序較低的 `block: true`
  仍可加以封鎖。
- `onResolution` 會收到已解析的決定：`allow-once`、`allow-always`、
  `deny`、`timeout` 或 `cancelled`。

### 在單一檔案中實作可感知傳送者的政策

獨立外掛檔案可將部署專用政策保留在程式碼中，而不必
新增另一個設定結構描述。此範例讓擁有者可使用所有工具，
讓已設定的維護人員使用一組保守的工具與訊息動作，
並向已獲頻道設定授權的傳送者公開 `/fix`：

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const AGENT_ID = "maintenance-agent";
const MAINTAINER_SCOPES = [
  {
    channel: "discord",
    accountId: "operations",
    senderIds: new Set(["maintainer-user-id"]),
    roleIds: new Set(["maintainer-role-id"]),
  },
];
const MAINTAINER_TOOLS = new Set(["read", "web_fetch", "web_search", "session_status", "message"]);
const MAINTAINER_MESSAGE_ACTIONS = new Set(["react", "reply", "thread-create", "thread-reply"]);

export default definePluginEntry({
  id: "maintenance-access",
  name: "維護存取權",
  description: "對維護代理套用可感知傳送者的工具政策。",
  register(api) {
    api.on("before_tool_call", (event, ctx) => {
      if (ctx.agentId !== AGENT_ID) {
        return;
      }

      const requester = ctx.requester;
      if (requester?.senderIsOwner === true) {
        return;
      }

      const maintainerScope = requester
        ? MAINTAINER_SCOPES.find(
            (scope) =>
              scope.channel === requester.channel && scope.accountId === requester.accountId,
          )
        : undefined;
      const isMaintainer =
        maintainerScope !== undefined &&
        ((requester?.senderId !== undefined && maintainerScope.senderIds.has(requester.senderId)) ||
          requester?.roleIds?.some((roleId) => maintainerScope.roleIds.has(roleId)) === true);
      if (!isMaintainer) {
        return { block: true, blockReason: "需要維護人員存取權。" };
      }

      if (event.toolName === "message") {
        const action = typeof event.params.action === "string" ? event.params.action : "";
        if (MAINTAINER_MESSAGE_ACTIONS.has(action)) {
          return;
        }
        return { block: true, blockReason: `message.${action || "unknown"} 需要擁有者權限。` };
      }

      if (MAINTAINER_TOOLS.has(event.toolName)) {
        return;
      }
      return { block: true, blockReason: `${event.toolName} 需要擁有者權限。` };
    });

    api.registerCommand({
      name: "fix",
      description: "要求維護代理調查並修正問題。",
      acceptsArgs: true,
      requireAuth: true,
      handler: async (ctx) =>
        ctx.agentId === AGENT_ID
          ? { continueAgent: true }
          : { text: "此命令僅能在維護對話中使用。" },
    });
  },
});
```

直接載入檔案並重新啟動閘道：

```json5
{
  agents: {
    list: [
      {
        id: "maintenance-agent",
        workspace: "~/.openclaw/workspace-maintenance",
      },
    ],
  },
  bindings: [
    {
      agentId: "maintenance-agent",
      match: {
        channel: "discord",
        accountId: "operations",
        peer: { kind: "channel", id: "maintenance-channel-id" },
      },
    },
  ],
  plugins: {
    load: { paths: ["~/.openclaw/policies/maintenance-access.ts"] },
  },
}
```

`AGENT_ID` 必須指定繫結至維護對話的代理。此
繫結會為一般訊息和 `/fix` 選取該代理；獨立檔案
仍是擁有者與維護人員工具政策的唯一擁有者。

`requireAuth: true` 會重複使用各頻道現有的傳送者准入機制。對於
Discord，伺服器或頻道的 `users`/`roles` 允許清單可授權
維護對象。其他頻道可使用穩定的傳送者 ID。接著，此掛鉤會
對執行中的每次工具呼叫套用更細緻的逐工具決定，包括
Codex 原生 `PreToolUse` 呼叫。它可以否決模型看得到的工具，但無法
新增主機所省略的工具。現有的沙箱、執行核准、僅限擁有者的
核心工具和頻道政策仍然適用；此掛鉤無法越過它們授予權限。

請如範例所示，將傳送者和角色 ID 限定於確切的頻道／帳戶配對；
兩者都是提供者本機的命名空間。允許清單應保持保守。只有在部署環境的沙箱與核准政策能
確保安全時，才新增寫入或執行工具。對於自動化或系統執行，請明確決定缺少
`ctx.requester` 時是否應放行；此範例會針對指定範圍的代理加以拒絕。

請參閱[外掛權限要求](/zh-TW/plugins/plugin-permission-requests)，瞭解
核准路由、決定行為，以及何時應使用 `requireApproval`，而非
選用工具或執行核准。

需要主機層級政策的外掛，可使用
`api.registerTrustedToolPolicy(...)` 註冊受信任的工具政策。這些政策會在一般
`before_tool_call` 掛鉤和一般掛鉤決定之前執行。隨附的受信任
政策會最先執行；已安裝外掛的受信任政策接著依外掛載入
順序執行；一般 `before_tool_call` 掛鉤則在其後執行。隨附外掛會保留
現有的受信任政策路徑。已安裝的外掛必須明確啟用，
並在 `contracts.trustedToolPolicies` 中宣告每個政策 ID；未宣告的 ID
會在註冊前遭到拒絕。政策 ID 的範圍限定於進行註冊的
外掛，因此不同外掛可重複使用相同的本機 ID。此層級僅適用於
主機信任的防護措施，例如工作區政策、預算強制執行或
保留工作流程的安全性。

### Exec 環境掛鉤

`resolve_exec_env` 可讓外掛在命令執行前，為 `exec`
工具叫用提供環境變數。它會接收：

- `event.sessionKey`
- `event.toolName`，目前一律為 `"exec"`
- `event.host`，為 `"gateway"`、`"sandbox"` 或 `"node"` 之一
- 內容相關欄位，例如 `ctx.agentId`、`ctx.sessionKey`、
  `ctx.messageProvider` 和 `ctx.channelId`

傳回 `Record<string, string>`，以合併至 exec 環境。處理常式會依優先順序
執行；若索引鍵相同，較後的結果會覆寫較早的結果。

掛鉤輸出會先經過主機 exec 環境索引鍵原則篩選，再進行合併。
`PATH` 一律會遭捨棄（命令解析和安全二進位檔檢查
依賴此項）。無效索引鍵，以及 `LD_*`、`DYLD_*`、
`NODE_OPTIONS` 等危險的主機覆寫索引鍵、Proxy 變數
（`HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`、`NO_PROXY`）
和 TLS 覆寫變數（`NODE_TLS_REJECT_UNAUTHORIZED`、`SSL_CERT_FILE` 及類似變數）都會遭捨棄。
篩選後的外掛環境會納入閘道核准／稽核中繼資料，並轉送至節點主機執行要求。

### 工具結果持久化

工具結果可包含結構化的 `details`，供 UI 轉譯、診斷、
媒體路由或外掛自有中繼資料使用。請將 `details` 視為執行階段中繼資料，
而非提示內容：

- OpenClaw 會在提供者重播及壓縮輸入前移除 `toolResult.details`，
  以免中繼資料成為模型內容。
- 持久化的工作階段項目僅保留有界的 `details`。過大的詳細資料會
  以精簡摘要和 `persistedDetailsTruncated: true` 取代。
- `tool_result_persist` 和 `before_message_write` 會在最終
  持久化上限之前執行。請保持傳回的 `details` 精簡，並避免只將
  與提示相關的文字放在 `details` 中；請將模型可見的工具輸出放在
  `content` 中。

## 提示與模型掛鉤

新外掛請使用特定階段的掛鉤：

- `before_model_resolve`：僅接收目前的提示和附件
  中繼資料。傳回 `providerOverride` 或 `modelOverride`。
- `agent_turn_prepare`：接收目前的提示、準備好的工作階段
  訊息，以及為此工作階段取出的任何僅執行一次的排隊注入內容。
  傳回 `prependContext` 或 `appendContext`。
- `before_prompt_build`：接收目前的提示和工作階段訊息。
  傳回 `prependContext`、`appendContext`、`systemPrompt`、
  `prependSystemContext` 或 `appendSystemContext`。
- `heartbeat_prompt_contribution`：僅在心跳偵測輪次中執行，並傳回
  `prependContext` 或 `appendContext`。適用於需要摘要目前狀態，
  但不應變更由使用者起始之輪次的背景監控程式。

`before_agent_run` 會在提示建構完成後、任何模型輸入之前執行，
包括提示區域影像載入和 `llm_input` 觀察。它會以 `prompt`
接收目前的使用者輸入，並透過 `messages` 接收已載入的工作階段歷史記錄
和作用中的系統提示。傳回 `{ outcome: "block", reason, message? }`，
可在模型讀取提示前停止執行。`reason` 供內部使用；
`message` 是面向使用者的替代內容。僅支援 `pass` 和
`block` 結果；不支援的決策格式會採封閉式失敗。

當執行遭封鎖時，OpenClaw 只會在 `message.content` 中儲存替代文字，
以及封鎖外掛 ID 和時間戳記等非敏感封鎖中繼資料。原始使用者文字不會保留在
逐字記錄或未來內容中。內部封鎖原因會視為敏感資訊，並從逐字記錄、歷史記錄、
廣播、日誌和診斷承載資料中排除。可觀測性應使用已清理的欄位，例如封鎖者 ID、
結果、時間戳記或安全類別。

包含 `agent_end` 在內的代理程式輪次掛鉤，會在 OpenClaw 能識別
作用中執行時包含 `event.runId`；`ctx.runId` 上也會有相同值。
由排程驅動的執行還會在代理程式輪次內容上公開 `ctx.jobId`
（來源排程工作 ID），讓掛鉤可將指標、副作用或狀態限定於特定排程工作。
`ctx.jobId` 不屬於 `before_tool_call` 工具內容的一部分。

對於源自頻道的執行，`ctx.channel` 和 `ctx.messageProvider` 會識別
`discord` 或 `telegram` 等提供者介面，而 `ctx.channelId`
則是 OpenClaw 能從工作階段索引鍵或遞送中繼資料推導出的對話目標識別碼。

當寄件者身分可用時，代理程式掛鉤內容還會包含：

- `ctx.senderId` - 頻道範圍的寄件者 ID（例如 Feishu
  `open_id`、Discord 使用者 ID）。當執行源自具有已知寄件者
  中繼資料的使用者訊息時填入。
- `ctx.chatId` - 傳輸原生的對話識別碼（例如 Feishu
  `chat_id`、Telegram `chat_id`）。當來源頻道
  提供原生對話 ID 時填入。
- `ctx.channelContext.sender.id` - 與 `ctx.senderId` 相同的寄件者 ID，
  位於頻道自有物件下，外掛可使用頻道特定欄位擴充該物件。
- `ctx.channelContext.chat.id` - 與 `ctx.chatId` 相同的對話 ID，
  位於頻道自有物件下，外掛可使用頻道特定欄位擴充該物件。

核心只定義巢狀的 `id` 欄位。透過入站協助程式傳遞更豐富
寄件者或聊天中繼資料的頻道外掛，可以從 `openclaw/plugin-sdk/channel-inbound`
擴充 `PluginHookChannelSenderContext` 或 `PluginHookChannelChatContext`：

```ts
declare module "openclaw/plugin-sdk/channel-inbound" {
  interface PluginHookChannelSenderContext {
    unionId?: string;
    userId?: string;
  }
}
```

頻道外掛會透過入站 SDK 協助程式傳遞這些欄位：

```ts
buildChannelInboundEventContext({
  // ...
  channelContext: {
    sender: { id: senderOpenId, unionId, userId },
    chat: { id: chatId },
  },
});
```

這些欄位為選用；若執行源自系統（心跳偵測、排程、exec 事件），
則不會出現。

`ctx.senderExternalId` 仍保留為舊版外掛的已棄用原始碼相容性欄位。
核心不會填入此欄位；新的頻道特定寄件者身分應透過模組擴充，
置於 `ctx.channelContext.sender` 下。

`agent_end` 是觀察掛鉤。閘道和持久性測試框架路徑會在輪次結束後
以觸發後不等待的方式執行，而短期的一次性命令列介面路徑則會在程序清理前等待
掛鉤 Promise，讓受信任的外掛能清除終端可觀測性資料或擷取狀態。掛鉤執行器
會套用 30 秒逾時，避免卡死的外掛或嵌入端點讓掛鉤 Promise 永久維持待處理。
逾時會記錄至日誌，且 OpenClaw 會繼續執行；除非外掛也使用自己的中止訊號，
否則不會取消外掛自有的網路工作。

若提供者呼叫遙測不應接收原始提示、歷史記錄、回應、標頭、要求本文或提供者
要求 ID，請使用 `model_call_started` 和 `model_call_ended`。這些掛鉤包含
`runId`、`callId`、`provider`、
`model`、選用的 `api`/`transport`、
終止狀態 `durationMs`/`outcome` 等穩定中繼資料，
以及 OpenClaw 能推導出有界提供者要求 ID 雜湊時的 `upstreamRequestIdHash`。
當執行階段已解析內容視窗中繼資料時，掛鉤事件和內容也會包含
`contextTokenBudget`（套用模型／設定／代理程式上限後的有效權杖預算），
以及套用較低上限時的 `contextWindowSource` 和 `contextWindowReferenceTokens`。

`before_agent_finalize` 僅在測試框架即將接受自然產生的最終助理回答時執行。
它不是 `/stop` 取消路徑，且使用者中止輪次時不會執行。
傳回 `{ action: "revise", reason }`，可要求測試框架在最終定案前再執行一次模型；
傳回 `{ action:
"finalize", reason? }` 可強制最終定案；若省略結果則繼續。
處理常式預設有 15 秒預算；逾時時，OpenClaw 會記錄失敗並繼續使用
原始最終回答。
Codex 原生 `Stop` 掛鉤會以 OpenClaw
`before_agent_finalize` 決策轉送至此掛鉤。

傳回 `action: "revise"` 時，外掛可包含 `retry` 中繼資料，
使額外模型執行有界且可安全重播：

```typescript
type BeforeAgentFinalizeRetry = {
  instruction: string;
  idempotencyKey?: string;
  maxAttempts?: number;
};
```

`instruction` 會附加至傳送給測試框架的修訂原因。
`idempotencyKey` 可讓主機計算同一外掛要求在等效最終定案決策中的重試次數，
而 `maxAttempts` 則限制主機在繼續採用自然產生的最終回答前，
允許的額外執行次數。

需要原始對話掛鉤（`before_model_resolve`、`before_agent_reply`、
`llm_input`、`llm_output`、`before_agent_finalize`、
`agent_end` 或 `before_agent_run`）的非內建外掛必須設定：

```json
{
  "plugins": {
    "entries": {
      "my-plugin": {
        "hooks": {
          "allowConversationAccess": true
        }
      }
    }
  }
}
```

可透過 `plugins.entries.<id>.hooks.allowPromptInjection=false`，針對個別外掛停用修改提示的掛鉤和持久性的
下一輪注入。

### 工作階段擴充與下一輪注入

工作流程外掛可使用 `api.session.state.registerSessionExtension(...)` 持久保存小型 JSON 相容工作階段狀態，
並透過閘道 `sessions.pluginPatch` 方法更新。工作階段資料列會透過
`pluginExtensions` 投影已註冊的擴充狀態，讓 Control UI 和其他用戶端
能轉譯外掛自有狀態，而無須瞭解外掛內部實作。
`api.registerSessionExtension(...)` 仍可運作，但已棄用，建議改用
`api.session.state` 命名空間。

當外掛需要讓持久性內容恰好一次地進入下一個模型輪次時，請使用
`api.session.workflow.enqueueNextTurnInjection(...)`（頂層的 `api.enqueueNextTurnInjection(...)` 是具有相同行為的已棄用別名）。
OpenClaw 會在提示掛鉤之前取出排隊的注入內容、捨棄已過期的注入內容，
並依每個外掛的 `idempotencyKey` 去除重複項目。對於應在下一輪向模型顯示，
但不應成為永久系統提示文字的核准續接、原則摘要、背景監控程式差異，
以及命令接續，這是適當的接合點。

清理語意是合約的一部分。工作階段擴充清理和執行階段生命週期清理回呼會接收
`reset`、`delete`、`disable` 或
`restart`。主機會在重設／刪除／停用時，移除所屬外掛的持久性
工作階段擴充狀態和待處理的下一輪注入；重新啟動則會保留持久性工作階段狀態，
同時清理回呼可讓外掛為舊的執行階段世代釋放排程器工作、執行內容，
以及其他頻帶外資源。

## 訊息掛鉤

使用訊息掛鉤處理頻道層級路由和遞送原則：

- `message_received`：觀察入站內容、寄件者、`threadId`、
  `messageId`、`senderId`、選用的執行／工作階段關聯、
  已排序的 `media` 和中繼資料。
- `message_sending`：重寫 `content` 或傳回 `{ cancel: true }`。
- `reply_payload_sending`：重寫正規化的 `ReplyPayload` 物件
  （包括 `presentation`、`delivery`、媒體參照和文字），或傳回
  `{ cancel: true }`。
- `message_sent`：觀察最終成功或失敗。

對於僅含音訊的 TTS 回覆，即使頻道承載資料沒有可見文字／字幕，
`content` 仍可能包含隱藏的語音逐字稿。重寫該 `content`
只會更新掛鉤可見的逐字稿；不會將其轉譯為媒體字幕。

`reply_payload_sending` 事件可能包含 `usageState`，這是盡力提供的即時
每輪模型／用量／內容快照。持久性遞送、復原的重播，以及缺乏精確執行關聯的
回覆會省略此項。

訊息鉤子內容在可用時會公開穩定的關聯欄位：
`ctx.sessionKey`、`ctx.runId`、`ctx.messageId`、`ctx.senderId`、`ctx.trace`、
`ctx.traceId`、`ctx.spanId`、`ctx.parentSpanId` 和 `ctx.callDepth`。輸入
和 `before_dispatch` 內容也會在頻道具有經可見性篩選的引用訊息資料時公開回覆中繼資料：`replyToId`、`replyToIdFull`、
`replyToBody`、`replyToSender` 和 `replyToIsQuote`。請先使用這些
第一級欄位，再讀取舊版中繼資料。

請先使用具型別的 `threadId` 和 `replyToId` 欄位，再使用頻道特定的
中繼資料。

輸入宣告與訊息接收事件會將 `media?:
PluginHookMediaFact[]` 公開為標準附件 API。每項事實都可包含
`path`、`url`、`contentType`、`kind`、`transcribed`、`messageId` 和
`workspaceDir`；陣列位置即為附件識別。當遠端附件
尚未在本機暫存時，會省略 `media`，
`mediaStagingPending: true`，而 `originalMedia` 則包含提供者端的
事實。在之後的暫存事件提供 `media` 之前，請勿將
`originalMedia.path` 視為可在本機讀取。

單數／複數的 `mediaPath`、`mediaUrl`、`mediaType`、`mediaPaths`、
`mediaUrls`、`mediaTypes`，以及相符的 `originalMedia*` 中繼資料屬性，皆為
已淘汰的相容性別名。新的鉤子應使用具型別的頂層
陣列。

決策規則：

- `message_sending` 搭配 `cancel: true` 時為終止狀態。
- `message_sending` 搭配 `cancel: false` 時視為未做決策。
- 重寫後的 `content` 會繼續傳至優先順序較低的鉤子，除非之後的鉤子
  取消傳遞。
- `reply_payload_sending` 會在承載內容正規化後、傳遞至頻道前執行，
  包括路由回原始頻道的回覆。
  處理常式會依序執行，每個處理常式都會看到由
  優先順序較高的處理常式產生的最新承載內容。
- `reply_payload_sending` 承載內容不會公開執行階段信任標記，例如
  `trustedLocalMedia`；外掛可以編輯承載內容的形態，但無法授予本機
  媒體信任。
- `message_sending` 可隨取消傳回 `cancelReason` 和有界的 `metadata`。
  新的訊息生命週期 API 會將此公開為原因為
  `cancelled_by_message_sending_hook` 的已抑制傳遞結果；舊版
  直接傳遞則為了相容性繼續傳回空的結果陣列。
- `message_sent` 僅供觀察。處理常式失敗會記錄至日誌，但不會
  變更傳遞結果。

## 安裝鉤子

請使用 `security.installPolicy` 進行由操作員擁有的允許／封鎖決策。該
政策由 OpenClaw 設定執行，涵蓋命令列介面的安裝與更新路徑，且在
已啟用但無法使用時採取故障關閉。

`before_install` 是外掛執行階段生命週期鉤子。它僅會在
已載入外掛鉤子的 OpenClaw 程序中，於
`security.installPolicy` 之後執行，例如由閘道支援的安裝流程。它適用於
外掛所擁有的觀察、警告與相容性檢查，但不是
安裝作業的主要企業或主機安全邊界。
`builtinScan` 欄位基於相容性仍保留在事件承載內容中，但
OpenClaw 不再執行內建的安裝時危險程式碼封鎖，因此它是空的
`ok` 結果。傳回額外的發現項目或
`{ block: true, blockReason }`，即可在該程序中停止安裝。

`block: true` 為終止狀態。`block: false` 視為未做決策。處理常式
失敗會以故障關閉方式封鎖安裝。

## 閘道生命週期

使用 `gateway_start` 啟動一般外掛服務，並使用 `gateway_stop`
清理長時間執行的資源。當
`gateway_start` 執行時，排程器可能仍在載入，因此請勿將它用作外部
排程投影的基準訊號。

請勿依賴內部 `gateway:startup` 鉤子執行由外掛擁有的執行階段
服務。

`cron_reconciled` 會在閘道排程器及其結束時
監看器協調完持久狀態後觸發。它會在初始
啟動以及設定重新載入期間更換排程器時觸發。事件會回報
`reason`（`startup` 或 `reload`）及有效的 `enabled` 狀態。停用的
排程仍會以 `enabled: false` 發出，讓外部投影能夠
清除過時的喚醒項目。請使用 `ctx.getCron?.()` 取得完成協調的確切排程器執行個體；
之後的重新載入不會重新指定該回呼的目標。
`ctx.abortSignal` 擁有同一份排程器快照。只要
更新的排程器就緒或開始關閉，閘道就會立即中止它。請將它傳入每個
持久性副作用，並且不要在它中止後接受該快照。
這是排程器生命週期訊號，而不是外掛啟用訊號：
僅外掛的熱重新載入不會重播此訊號。新啟用的消費者會在
下一次更換排程器或啟動閘道時收到第一份基準。

如同其他觀察鉤子，`gateway_start` 和 `cron_reconciled` 回呼
可能重疊。如果兩個處理常式共用外掛初始化，請使用
外掛本機的就緒 Promise 協調，而不要依賴回呼順序。

`cron_changed` 會針對由閘道擁有的排程生命週期事件觸發，其具型別的
事件承載內容涵蓋 `added`、`updated`、`removed`、`started`、`finished`
和 `scheduled` 原因。事件會攜帶 `PluginHookGatewayCronJob`
快照（包括存在時的 `state.nextRunAtMs`、`state.lastRunStatus` 和
`state.lastError`），以及 `not-requested` | `delivered` | `not-delivered` | `unknown` 的 `PluginHookGatewayCronDeliveryStatus`。
移除事件會在提交後觸發：只有在持久性刪除成功後才會觸發，且仍會攜帶
已刪除的工作快照，讓外部排程器能夠協調狀態。

`scheduled` 事件會在提交後觸發：只有在成功的持久性
寫入變更現有工作的有效 `nextRunAtMs` 後才會觸發，但不包括該工作的
明確 `added`、`updated` 或 `removed` 生命週期事件。頂層
`event.nextRunAtMs` 是已提交的下次喚醒時間；若不存在，表示該工作
沒有下次喚醒。請將這些事件視為協調提示，而不是有序的差異
日誌。將它們用作可合併的提示，以重新讀取
`cron_reconciled` 最後擷取的排程器；請勿採用 `cron_changed` 內容中的排程器。
讓 OpenClaw 繼續作為到期檢查與執行的單一事實來源。

### 安全的外部排程投影

請投影完整的喚醒快照，而不是轉送排程事件差異。外部
配接器的 `replaceAll` 作業必須具備不可分割性與冪等性，而且
只有在主機已持久接受快照後才能完成。它也
必須遵守提供的中止訊號：若訊號在持久接受前中止，
配接器不得接受該快照。

此模式會讓一個最新狀態工作程式持續執行。只有 `cron_reconciled`
會採用排程器執行個體；`cron_changed` 僅要求該工作程式重新讀取
權威執行個體，因此延遲的提示無法還原較舊的排程器。
較新的修訂會中止作用中的主機嘗試，避免它接受過時的
快照。

```typescript
import { setTimeout as sleep } from "node:timers/promises";
import type { OpenClawPluginApi } from "openclaw/plugin-sdk/plugin-entry";

type ExternalWake = { jobId: string; runAtMs: number };

type ExternalWakeHost = {
  replaceAll(wakes: readonly ExternalWake[], options: { signal: AbortSignal }): Promise<void>;
  close(): Promise<void>;
};

type CronReader = {
  list(options: { includeDisabled: true }): Promise<
    Array<{
      id: string;
      enabled?: boolean;
      state?: { nextRunAtMs?: number };
    }>
  >;
};

export function registerCronProjection(api: OpenClawPluginApi, host: ExternalWakeHost) {
  const lifecycle = new AbortController();
  let cron: CronReader | undefined;
  let enabled = false;
  let hasBaseline = false;
  let reconciliationSignal: AbortSignal | undefined;
  let requestedRevision = 0;
  let appliedRevision = 0;
  let worker = Promise.resolve();
  let activeAttempt: AbortController | undefined;

  const projectLatest = async () => {
    let retryMs = 1_000;

    while (!lifecycle.signal.aborted && appliedRevision < requestedRevision) {
      const ownerSignal = reconciliationSignal;
      if (!ownerSignal || ownerSignal.aborted) {
        return;
      }
      const targetRevision = requestedRevision;
      const attempt = new AbortController();
      const signal = AbortSignal.any([lifecycle.signal, ownerSignal, attempt.signal]);
      activeAttempt = attempt;

      try {
        const jobs = enabled && cron ? await cron.list({ includeDisabled: true }) : [];
        if (signal.aborted || targetRevision !== requestedRevision) {
          continue;
        }
        const wakes = jobs
          .flatMap((job): ExternalWake[] => {
            const runAtMs = job.enabled === false ? undefined : job.state?.nextRunAtMs;
            return runAtMs === undefined ? [] : [{ jobId: job.id, runAtMs }];
          })
          .sort((a, b) => a.runAtMs - b.runAtMs || a.jobId.localeCompare(b.jobId));

        await host.replaceAll(wakes, { signal });
        if (signal.aborted || targetRevision !== requestedRevision) {
          continue;
        }
        appliedRevision = targetRevision;
        retryMs = 1_000;
      } catch {
        if (lifecycle.signal.aborted || ownerSignal.aborted) {
          return;
        }
        if (attempt.signal.aborted) {
          continue;
        }
        api.logger.warn(`外部排程投影失敗；將於 ${retryMs}ms 後重試`);
        try {
          await sleep(retryMs, undefined, { signal });
        } catch {
          if (lifecycle.signal.aborted) {
            return;
          }
          if (attempt.signal.aborted) {
            continue;
          }
        }
        retryMs = Math.min(retryMs * 2, 30_000);
      } finally {
        if (activeAttempt === attempt) {
          activeAttempt = undefined;
        }
      }
    }
  };

  const requestProjection = () => {
    const targetRevision = ++requestedRevision;
    activeAttempt?.abort();
    worker = worker.then(async () => {
      if (!lifecycle.signal.aborted && appliedRevision < targetRevision) {
        await projectLatest();
      }
    });
    return worker;
  };

  api.on("cron_reconciled", (event, ctx) => {
    const reconciledCron = ctx.getCron?.();
    if (event.enabled && !reconciledCron) {
      api.logger.warn("排程協調未公開排程器");
      return;
    }
    cron = reconciledCron;
    enabled = event.enabled;
    hasBaseline = true;
    reconciliationSignal = ctx.abortSignal;
    return requestProjection();
  });

  api.on("cron_changed", () => {
    if (hasBaseline) {
      return requestProjection();
    }
  });

  api.on("gateway_stop", async () => {
    lifecycle.abort();
    await worker;
    await host.close();
  });
}
```

當 `cron_reconciled` 回報 `enabled: false` 時，同一路徑會呼叫
`replaceAll([])` 並清除過時的外部喚醒項目。此範例中的重試／退避
作用於程序本機，並將執行階段配接器失敗視為暫時性失敗；請在
註冊前驗證不可重試的設定。OpenClaw 不會為外掛鉤子作用提供
寄件匣。如果程序在持久接受前結束，
下一次啟動閘道時會發出新的權威 `cron_reconciled` 快照。
`gateway_stop` 會中止進行中的主機工作、等待工作程式結束，然後
關閉配接器。

## 即將淘汰的項目

少數與鉤子相鄰的介面已淘汰，但仍受支援。請在
下一個主要版本發布前遷移：

- **純文字頻道信封**位於 `inbound_claim` 和 `message_received`
  處理常式中。請讀取 `BodyForAgent` 和結構化的使用者情境區塊，
  而不要剖析扁平的信封文字。請參閱
  [純文字頻道信封 → BodyForAgent](/zh-TW/plugins/sdk-migration#active-deprecations)。
- **`subagent_spawning`** 為了與較舊的外掛相容而保留，但
  新外掛不應從中傳回討論串路由。在 `subagent_spawned` 觸發之前，
  核心會透過頻道工作階段繫結配接器準備 `thread: true` 子代理程式繫結。
- **`deactivate`** 會作為已淘汰的清理相容性別名保留至
  2026-08-16 之後。新外掛應使用 `gateway_stop`。
- **`before_tool_call` 中的 `onResolution`** 現在使用具型別的
  `PluginApprovalResolution` 聯集（`allow-once` / `allow-always` / `deny` /
  `timeout` / `cancelled`），而非自由格式的 `string`。
- **`api.registerSessionExtension` / `api.enqueueNextTurnInjection`** 仍
  作為頂層相容性別名保留。新外掛應使用
  `api.session.state.registerSessionExtension(...)` 和
  `api.session.workflow.enqueueNextTurnInjection(...)`。

如需完整清單，包括記憶體能力註冊、供應商思考
設定檔、外部驗證供應商、供應商探索型別、任務執行階段
存取子，以及 `command-auth` → `command-status` 重新命名，請參閱
[外掛 SDK 遷移 → 作用中的淘汰項目](/zh-TW/plugins/sdk-migration#active-deprecations)。

## 相關內容

- [外掛 SDK 遷移](/zh-TW/plugins/sdk-migration) - 作用中的淘汰項目與移除時程
- [建置外掛](/zh-TW/plugins/building-plugins)
- [外掛 SDK 概觀](/zh-TW/plugins/sdk-overview)
- [外掛進入點](/zh-TW/plugins/sdk-entrypoints)
- [內部掛鉤](/zh-TW/automation/hooks)
- [外掛架構內部機制](/zh-TW/plugins/architecture-internals)
