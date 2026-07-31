---
read_when:
    - 你需要使用外掛掛鉤或工具，在執行副作用前先詢問。
    - 你需要設定外掛核准提示的傳送位置
    - 你正在選擇選用工具、執行核准與外掛核准的配置方式
sidebarTitle: Permission requests
summary: 要求使用者核准外掛工具呼叫及外掛所屬的權限提示
title: 外掛權限要求
x-i18n:
    generated_at: "2026-07-26T07:49:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 675534212e70cc7b2e7bdc801955929c6a8156b08d620483edf0133afc3bfdaa
    source_path: plugins/plugin-permission-requests.md
    workflow: 16
---

外掛權限請求可讓外掛程式碼暫停工具呼叫或外掛擁有的
操作，直到使用者核准或拒絕為止。它們使用閘道
`plugin.approval.*` 流程，以及處理聊天
核准按鈕和 `/approve` 命令的相同核准 UI 介面。

外掛權限請求適用於外掛／應用程式權限。它們無法取代
主機執行核准、選用工具允許清單，或 Codex 的原生權限
審查。

## 選擇正確的管控機制

請選擇符合所需決策點的管控機制：

| 管控機制                         | 適用時機                                                                 | 控制範圍                                                                                                               |
| -------------------------------- | ------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| 選用工具                         | 在使用者選擇加入之前，不應讓模型看到某項工具。                           | 透過 `tools.allow` 控制工具呈現。                                                                                 |
| 外掛權限請求                     | 外掛掛鉤或外掛擁有的操作在執行某項動作前必須詢問。                       | 透過 `plugin.approval.*` 控制執行階段核准。                                                                             |
| 執行核准                         | 主機命令或類似 shell 的工具需要操作員核准。                              | 主機執行原則與持久執行允許清單。                                                                                       |
| Codex 原生權限請求               | Codex 在執行原生 shell、檔案、MCP 或應用程式伺服器動作前詢問。           | Codex 應用程式伺服器或原生掛鉤核准處理；當 OpenClaw 擁有提示時，會透過外掛核准路由。                                   |
| MCP 核准引導                     | Codex MCP 伺服器要求核准工具呼叫。                                       | 透過 OpenClaw 外掛核准橋接 MCP 核准回應。                                                                              |

選用工具是探索階段的管控機制。外掛權限請求則是
每次呼叫的管控機制。若敏感工具必須在模型可見之前明確選擇加入，
且執行動作前也必須經過核准，請同時使用兩者。

## 在工具呼叫前請求核准

大多數由外掛撰寫的提示應從 `before_tool_call` 掛鉤開始。此掛鉤
會在模型選取工具後、OpenClaw 執行工具前執行：

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "deploy-policy",
  name: "Deploy Policy",
  register(api) {
    api.on("before_tool_call", async (event) => {
      if (event.toolName !== "deploy_service") {
        return;
      }

      const environment =
        typeof event.params.environment === "string" ? event.params.environment : "unknown";

      return {
        requireApproval: {
          title: "Deploy service",
          description: `Deploy service to ${environment}.`,
          severity: environment === "production" ? "critical" : "warning",
          allowedDecisions:
            environment === "production"
              ? ["allow-once", "deny"]
              : ["allow-once", "allow-always", "deny"],
          timeoutMs: 120_000,
          onResolution(decision) {
            console.log(`deploy approval resolved: ${decision}`);
          },
        },
      };
    });
  },
});
```

請為將核准動作的人員撰寫提示文字：

- 請保持 `title` 簡短並聚焦於動作；閘道將其限制為 80 個字元。
- 請讓 `description` 明確且範圍有限；閘道將其限制為 512
  個字元。
- 請包含動作、目標及風險。請勿包含不應出現在聊天核准介面中的祕密、權杖或
  私人承載資料。
- 若省略 `severity`，預設為 `"warning"`。僅針對
  錯誤決策可能造成正式環境損害或資料遺失的動作使用 `"critical"`。
- 若省略 `allowedDecisions`，預設為 `["allow-once", "allow-always", "deny"]`。
  若該動作不適合持久信任，請傳入 `["allow-once", "deny"]`。
- `timeoutMs` 預設為 120000（2 分鐘），且無論要求的值為何，上限皆為 600000（10
  分鐘）。

## 決策行為

OpenClaw 會使用 `plugin:` ID 建立待處理核准，將其傳送至
可用的核准介面，並等待決策。

| 決策              | 結果                                                                      |
| ----------------- | ------------------------------------------------------------------------- |
| `allow-once`      | 繼續目前的呼叫。                                                          |
| `allow-always`    | 繼續目前的呼叫，並將決策傳遞給外掛。                                      |
| `deny`            | 以遭拒絕的工具結果封鎖呼叫。                                              |
| 逾時              | 封鎖呼叫。                                                                |
| 取消              | 執行中止時封鎖呼叫。                                                      |
| 無核准路由        | 因沒有已連線的核准介面可解決請求而封鎖呼叫。                              |

只有請求允許的確切 `allow-once` 和 `allow-always` 決策
才能允許執行。未知、格式錯誤、不相符、缺少及逾時的
決策一律採取封閉式失敗。為了外掛相容性，仍接受舊版 `timeoutBehavior` 欄位，
但該欄位已淘汰且會被忽略；請勿在新掛鉤中設定。

只有當提出請求的外掛或執行階段實作持久化時，`allow-always` 才具有持久性。
對一般的 `before_tool_call.requireApproval` 掛鉤而言，
OpenClaw 會將 `allow-once` 和 `allow-always` 視為目前呼叫的核准決策，
並將解析後的值傳遞給 `onResolution`。如果你的外掛
提供 `allow-always`，請記錄並實作它對未來呼叫所信任的確切範圍。

如果掛鉤也傳回 `params`，OpenClaw 只會在核准成功後
套用這些參數變更。即使較高優先順序的掛鉤已請求核准，
較低優先順序的掛鉤仍可封鎖呼叫。

`allowedDecisions` 會限制向使用者顯示的按鈕和命令。
對於請求未提供的任何決策，閘道都會拒絕解析嘗試。

## 路由核准提示

核准提示可在本機 UI 介面中解析，也可在
支援核准處理的聊天頻道中解析。若要將外掛核准提示轉送至明確的聊天
目標，請設定 `approvals.plugin`：

```json5
{
  approvals: {
    plugin: {
      enabled: true,
      mode: "targets",
      agentFilter: ["main"],
      targets: [{ channel: "slack", to: "U12345678" }],
    },
  },
}
```

`approvals.plugin` 與 `approvals.exec` 彼此獨立。啟用執行核准
轉送不會路由外掛核准提示，而啟用外掛核准
轉送也不會變更主機執行原則。

當提示包含手動核准文字時，請使用其中一個提供的
決策進行解析：

```text
/approve <id> allow-once
/approve <id> allow-always
/approve <id> deny
```

完整的轉送模型、同一聊天核准行為、原生頻道
傳送及頻道專屬核准者規則，請參閱[進階執行核准](/zh-TW/tools/exec-approvals-advanced#plugin-approval-forwarding)。

## Codex 原生權限

Codex 原生權限提示也可透過外掛核准傳遞，但
其所有權與外掛撰寫的掛鉤不同。

- Codex 應用程式伺服器核准請求會在 Codex 審查後透過 OpenClaw 路由。
- 啟用原生掛鉤 `permission_request` 中繼時，可透過
  `plugin.approval.request` 詢問。
- 當 Codex 將 `_meta.codex_approval_kind` 標記為 `"mcp_tool_call"` 時，
  MCP 工具核准引導會透過外掛核准路由。

Codex 專屬行為及備援規則，請參閱 [Codex 控制框架執行階段](/zh-TW/plugins/codex-harness-runtime#native-permissions-and-mcp-elicitations)。

## 疑難排解

**工具顯示外掛核准無法使用。** 沒有核准 UI 或已設定的
核准路由接受請求。請連線支援核准的用戶端、使用
支援同一聊天 `/approve` 的頻道，或設定 `approvals.plugin`。

**出現 `allow-always`，但下一次呼叫再次提示。** 一般外掛
核准流程不會自動為任意掛鉤持久保存信任。請在 `onResolution("allow-always")` 後
於外掛中持久保存外掛擁有的信任，或
僅提供 `allow-once` 和 `deny`。

**`/approve` 拒絕該決策。** 請求限制了
`allowedDecisions`。請使用提示中列出的其中一個決策。

**Discord、Matrix、Slack 或 Telegram 提示的路由方式與執行
核准不同。** 外掛核准與執行核准使用不同的設定，且可能採用
不同的授權檢查。請驗證 `approvals.plugin` 和該頻道的
外掛核准支援，而不是只檢查 `approvals.exec`。

## 相關內容

- [外掛掛鉤](/zh-TW/plugins/hooks#tool-call-policy)
- [建置外掛](/zh-TW/plugins/building-plugins#registering-tools)
- [進階執行核准](/zh-TW/tools/exec-approvals-advanced#plugin-approval-forwarding)
- [閘道通訊協定](/zh-TW/gateway/protocol)
- [Codex 控制框架執行階段](/zh-TW/plugins/codex-harness-runtime#native-permissions-and-mcp-elicitations)
