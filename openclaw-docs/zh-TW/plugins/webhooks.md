---
read_when:
    - 你想要從外部系統觸發或驅動 TaskFlow
    - 你正在設定隨附的網路鉤子外掛
summary: 網路鉤子外掛：供受信任外部自動化使用的已驗證 TaskFlow 輸入端點
title: 網路鉤子外掛
x-i18n:
    generated_at: "2026-07-26T08:31:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 77e455450d6183635c76a1e8002feeb287deb4ff242dbd555ef9d0f2b21ce5f6
    source_path: plugins/webhooks.md
    workflow: 16
---

Webhooks 外掛新增經過驗證的 HTTP 路由，讓受信任的外部
系統（Zapier、n8n、CI 作業、內部服務）無須撰寫自訂外掛，
即可透過 HTTP 建立及驅動受管理的 OpenClaw TaskFlow。

此外掛在閘道程序內執行。若使用遠端閘道，請在該主機上安裝並
設定外掛，然後重新啟動閘道。外掛隨附時未設定任何路由，因此在你新增至少一條路由前，
不會執行任何操作。

## 設定路由

在 `plugins.entries.webhooks.config` 下設定組態：

```json5
{
  plugins: {
    entries: {
      webhooks: {
        enabled: true,
        config: {
          routes: {
            zapier: {
              path: "/plugins/webhooks/zapier",
              sessionKey: "agent:main:main",
              secret: {
                source: "env",
                provider: "default",
                id: "OPENCLAW_WEBHOOK_SECRET",
              },
              controllerId: "webhooks/zapier",
              description: "Zapier TaskFlow bridge",
            },
          },
        },
      },
    },
  },
}
```

路由欄位：

| 欄位           | 必要 | 預設值                        | 備註                                          |
| -------------- | -------- | ----------------------------- | --------------------------------------------- |
| `enabled`      | 否       | `true`                        |                                               |
| `path`         | 否       | `/plugins/webhooks/<routeId>` | 在所有路由中必須是唯一的。                 |
| `sessionKey`   | 是      | -                             | 擁有所繫結 TaskFlow 的工作階段。        |
| `secret`       | 是      | -                             | 純文字字串或 SecretRef（見下文）。          |
| `controllerId` | 否       | `webhooks/<routeId>`          | 用作預設的 `create_flow` 控制器。 |
| `description`  | 否       | -                             | 僅供操作人員備註。                           |

`secret` 接受純文字字串或 SecretRef：`{ source: "env" | "file" | "exec", provider: "default", id: "..." }`。

SecretRef 會解析至閘道的啟動組態快照。當某條路由的
密鑰無法解析時，閘道會繼續執行，而該路由仍會保持
註冊但處於停用狀態：請求會收到一般驗證失敗回應（`401`）。
其他路由仍可使用。修正 SecretRef 來源後，重新載入或重新啟動
閘道以啟用新快照。絕不會在公開請求路徑上解析
SecretRef 值。

## 安全性模型

每條路由都會以其所設定 `sessionKey` 的 TaskFlow 權限運作：它
可以檢查及變更該工作階段擁有的任何 TaskFlow。TaskFlow 存取
一律經由 `api.runtime.tasks.managedFlows.bindSession(...)`，因此
路由絕不可能在其繫結的工作階段之外操作。若要限制影響範圍：

- 每條路由使用一組高強度且唯一的密鑰。
- 優先使用 SecretRef，而非內嵌的純文字密鑰。
- 將路由繫結至足以滿足工作流程需求的最小範圍工作階段。
- 僅公開你所需的特定網路鉤子路徑。

每個路徑的請求處理順序：HTTP 方法（僅限 `POST`）及
`Content-Type: application/json` 檢查，接著是固定時間窗速率限制（每個路徑與用戶端 IP 組合鍵在每個 60 秒時間窗內 120 個
請求，最多追蹤 4,096 個
鍵），再接著是進行中請求限制（每個鍵可同時處理 8 個請求，最多
追蹤 4,096 個鍵）、共用密鑰驗證，最後讀取上限為 256 KB／
15 秒的 JSON 內文。未通過較早階段檢查的請求絕不會進入
後續階段。

## 請求格式

傳送 `POST` 請求時，請使用 `Content-Type: application/json`，並提供
`Authorization: Bearer <secret>` 或 `x-openclaw-webhook-secret: <secret>`：

```bash
curl -X POST https://gateway.example.com/plugins/webhooks/zapier \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_SHARED_SECRET' \
  -d '{"action":"create_flow","goal":"Review inbound queue"}'
```

## 支援的動作

| 動作               | 用途                                                               |
| ------------------ | ------------------------------------------------------------------ |
| `create_flow`      | 為路由的工作階段建立受管理的 TaskFlow。                 |
| `get_flow`         | 依 ID 擷取一個 TaskFlow。                                          |
| `list_flows`       | 列出路由工作階段的 TaskFlow。                            |
| `find_latest_flow` | 擷取最近更新的 TaskFlow。                          |
| `resolve_flow`     | 透過不透明權杖解析 TaskFlow。                                |
| `get_task_summary` | 擷取 TaskFlow 的任務摘要。                             |
| `set_waiting`      | 將 TaskFlow 標記為等待中，並可選擇提供狀態／等待資料。            |
| `resume_flow`      | 恢復等待中／已阻擋的 TaskFlow。                                 |
| `finish_flow`      | 將 TaskFlow 標記為已完成。                                          |
| `fail_flow`        | 將 TaskFlow 標記為失敗。                                            |
| `request_cancel`   | 請求協同取消。                                  |
| `cancel_flow`      | 取消 TaskFlow（若子項目仍在執行，可能會傳回 `202`）。 |
| `run_task`         | 在現有 TaskFlow 中建立受管理的子任務。           |

變更動作（`set_waiting`、`resume_flow`、`finish_flow`、`fail_flow`、
`request_cancel`）需要 `flowId` 和 `expectedRevision` 以進行樂觀
並行控制；過期的修訂版本會傳回 `409 revision_conflict`。

### `create_flow`

```json
{
  "action": "create_flow",
  "goal": "Review inbound queue",
  "status": "queued",
  "notifyPolicy": "done_only"
}
```

### `run_task`

允許的 `runtime` 值：`subagent`、`acp`。只有當 `status` 為 `"running"` 時，
`startedAt`、`lastEventAt` 和
`progressSummary` 才有效；將這些值與任何其他狀態一起傳送時，會傳回 `400 invalid_request`。

```json
{
  "action": "run_task",
  "flowId": "flow_123",
  "runtime": "acp",
  "childSessionKey": "agent:main:acp:worker",
  "task": "Inspect the next message batch"
}
```

## 回應結構

```json
{
  "ok": true,
  "routeId": "zapier",
  "result": {}
}
```

```json
{
  "ok": false,
  "routeId": "zapier",
  "code": "not_found",
  "error": "TaskFlow not found.",
  "result": {}
}
```

流程和任務檢視絕不包含擁有者／工作階段中繼資料，因此回應無法
洩漏路由所繫結的 `sessionKey`。`code` 值包括 `not_found`、
`not_managed`、`revision_conflict`、`persist_failed`、`cancel_requested`、
`cancel_pending`、`terminal`、`invalid_request`、`request_rejected`，以及
動作專屬的後備代碼（`mutation_rejected`、`create_rejected`、
`task_not_created`、`cancel_rejected`），用於因上述具名代碼未涵蓋的
原因而拒絕變更時。

## 相關內容

- [鉤子](/zh-TW/automation/hooks) - 內部事件驅動鉤子與此 HTTP 型 TaskFlow 橋接器的比較
- [閘道網路鉤子（`hooks.*` 組態）](/zh-TW/automation/cron-jobs#webhooks) - 獨立的通用閘道 HTTP 端點功能；與此外掛的路由不同
- [外掛執行階段 SDK](/zh-TW/plugins/sdk-runtime)
- [命令列介面網路鉤子](/zh-TW/cli/webhooks)
