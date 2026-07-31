---
read_when:
    - 你正在建置與 OpenClaw 通訊的外部應用程式、指令碼、儀表板、CI 工作或 IDE 擴充功能
    - 你正在 Gateway RPC 與外掛 SDK 之間進行選擇
    - 你正在整合閘道的代理程式執行、工作階段、事件、核准、模型或工具
    - 你正在將託管控制器與外部喚醒排程器配對
sidebarTitle: External apps
summary: 外部應用程式、指令碼、儀表板、CI 作業和 IDE 擴充功能目前的整合路徑
title: 外部應用程式的閘道整合
x-i18n:
    generated_at: "2026-07-26T07:41:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 276c6f4173197683a60770327e131e6ab2fa4d33f416ba96c170539df7246f83
    source_path: gateway/external-apps.md
    workflow: 16
---

外部應用程式透過閘道協定與 OpenClaw 通訊：使用 WebSocket
傳輸加上 RPC 方法。當指令碼、儀表板、CI 工作、IDE
擴充功能或其他程序想要啟動代理程式執行、串流事件、等待
結果、取消工作或檢查閘道資源時，請使用此方式。

<Note>
  關於 npm 套件、裝置配對、重新連線復原、歷史記錄、訂閱
  和核准，請先參閱
  [建置閘道用戶端](https://docs.openclaw.ai/gateway/clients)。如果你的
  應用程式將閘道作為子程序監管，也請閱讀
  [嵌入 OpenClaw](https://docs.openclaw.ai/gateway/embedding)。在
  初始套件推出期間，在第一個包含套件的
  OpenClaw 版本發布前，npm 可能會傳回 `E404`。
</Note>

<Note>
  本頁適用於 OpenClaw 程序外部的程式碼。在
  OpenClaw 內部執行的外掛程式碼應改用已有文件說明的 `openclaw/plugin-sdk/*` 子路徑。
</Note>

## 目前可用的項目

| 介面                                                          | 狀態        | 用途                                                                                    |
| ---------------------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------- |
| [閘道用戶端指南](https://docs.openclaw.ai/gateway/clients) | 發布列車 | npm 套件、驗證、重新連線、歷史記錄、事件、核准及版本政策。                |
| [嵌入指南](https://docs.openclaw.ai/gateway/embedding)    | 發布列車 | 子程序環境、就緒狀態、生命週期、復原、RPC 所有權及封裝。      |
| [閘道協定](/zh-TW/gateway/protocol)                            | 就緒         | WebSocket 傳輸、連線交握、驗證範圍、協定版本控制及事件。         |
| [閘道 RPC 參考](/zh-TW/reference/rpc)                          | 就緒         | 目前用於代理程式、工作階段、任務、模型、工具、成品及核准的閘道方法。 |
| [`openclaw agent`](/zh-TW/cli/agent)                                   | 就緒         | 當透過 shell 呼叫命令列介面已足夠時，適用於一次性指令碼整合。                           |
| [`openclaw message`](/zh-TW/cli/message)                               | 就緒         | 從指令碼傳送訊息或頻道動作。                                             |

## 建議途徑

1. 執行或探索閘道。
2. 透過[閘道協定](/zh-TW/gateway/protocol)連線。
3. 呼叫[閘道 RPC 參考](/zh-TW/reference/rpc)中記載的 RPC 方法。
4. 固定你測試所用的 OpenClaw 版本。
5. 升級 OpenClaw 時重新查閱 RPC 參考。

對於代理程式執行，請從 `agent` RPC 開始，並搭配 `agent.wait` 取得
終止結果。對於持久的對話狀態，請使用 `sessions.*` 方法。
對於 UI 整合，請訂閱閘道事件，並只呈現你的應用程式
能理解的事件類別。

## 協作式主機暫停

凍結正在執行的程序或建立其快照的託管控制器，可以使用
主機中立的暫停交握：

1. 停止接受由主機控制的外部輸入流量。
2. 使用穩定且唯一的 `requestId` 呼叫 `gateway.suspend.prepare`。
3. 如果回應為 `busy`，請讓程序繼續執行，並稍後重試。
4. 如果回應為 `ready`，請儲存傳回的 `suspensionId`，然後在
   `expiresAtMs` 前凍結程序或建立其快照。
5. 解除凍結後，或放棄暫停時，請透過現有的 WebSocket 或管理 HTTP 控制
   路徑，使用該 `suspensionId` 呼叫 `gateway.suspend.resume`。

已準備就緒的閘道會拒絕新的 WebSocket 交握。WebSocket 控制器
必須在主機作業期間保持其已驗證連線開啟。如果無法
保證這一點，請在準備前啟用並使用
[管理 HTTP RPC 外掛](/zh-TW/plugins/admin-http-rpc)。如果
控制路徑中斷，請等候兩分鐘租約到期後再
重新連線；到期時會自動重新開放接受連線。

RPC 合約如下：

- `gateway.suspend.prepare` — `operator.admin`；參數
  `{ "requestId": "stable-host-operation-id" }`
- `gateway.suspend.status` — `operator.read`；參數
  `{ "suspensionId": "id-from-prepare" }`
- `gateway.suspend.resume` — `operator.admin`；參數
  `{ "suspensionId": "id-from-prepare" }`

ID 會移除前後空白、必須包含非空白字元，且上限為
128 個字元。忙碌中的準備結果包含 `status: "busy"`、`reason`、
`retryAfterMs`、`activeCount` 和 `blockers`。就緒結果的格式如下：

```json
{
  "status": "ready",
  "suspensionId": "2c3f...",
  "expiresAtMs": 1770000000000,
  "activeCount": 0,
  "blockers": []
}
```

狀態會傳回 `{"status":"running"}`，或傳回包含 `expiresAtMs` 的就緒結果。
繼續執行會傳回 `{"ok":true,"status":"running","resumed":true}`；成功繼續執行後再次呼叫，
則會傳回 `resumed: false`。

相互競爭的請求 ID 或暫時性的排程器恢復失敗，會傳回可重試的
`UNAVAILABLE`，其中包含 `retryAfterMs`。在排程器復原期間，準備、狀態
和繼續執行都會傳回該錯誤，閘道會維持未就緒並
採取失敗關閉模式，且主機不得凍結閘道或建立其快照。OpenClaw 會自動
重試排程器，且僅會在復原成功後重新開放接受連線。
不相符的繼續執行 ID 會傳回 `INVALID_REQUEST`。準備作業與閘道共用
每分鐘三次嘗試的控制平面寫入預算；請遵循傳回的
重試延遲。WebSocket 用戶端依裝置和 IP 分組計算。管理 HTTP
控制器依解析後的用戶端 IP 分組計算，因此位於同一個
Proxy 後方的控制器可能會共用一份預算。

準備作業僅能拒絕：OpenClaw 會關閉新的根層級／工作階段／命令接收、
暫停自動排程計時，並同步檢查工作。如果有任何
工作處於活動狀態，則會在傳回 `busy` 前恢復排程器並重新開放接收；
它不會中斷該工作，也不會等待該工作排空。就緒租約持續兩
分鐘。使用相同的 `requestId` 重複呼叫 `prepare` 會續期；租約到期時會先恢復
排程器，再重新開放接收。
在就緒租約期間到期應觸發的重新啟動發送，會等到租約
恢復後再執行；正在進行的重新啟動會使準備作業傳回 `busy`。

處於就緒狀態時，`/healthz` 仍可使用，而 `/readyz` 會傳回 `503`。本機或
已驗證的就緒狀態回應包含 `gateway-draining`；未驗證的
遠端探測只會收到 `{ "ready": false }`。HTTP 健康狀態探測、
現有 WebSocket 連線上的暫停方法，以及已啟用的
管理 HTTP RPC 路由仍可使用。其他 RPC 會傳回可重試的
`UNAVAILABLE`。內建 HTTP 使用者工作路由和一般外掛 HTTP 路由，
包括與 OpenAI 相容的 API、工具／工作階段作業、節點監看及
已設定的鉤子，會傳回包含 `error.code: "gateway_unavailable"` 的 `503`。新的
外掛所擁有的 WebSocket 升級也會傳回 `503`；這涵蓋升級
所有權，不包含之後透過已建立的外掛 Socket 執行的工作。

此交握不會保存傳入訊息、不會停止第三方頻道
傳輸，也不會控制託管平台。主機必須在準備前封鎖其輸入流量，
並持續負責喚醒、快照／凍結及
停止。`activeCount` 是彙總的受追蹤工作數量，而 `blockers`
包含非零類別計數和有數量限制的任務詳細資料。這不是
通用的程序靜止屏障。`background-exec` 阻擋項目僅提供彙總資訊：
命令文字、程序 ID、輸出以及工作階段或範圍識別碼絕不會
透過協定傳送。頻道健康狀態、維護、快取重新整理、已建立的
外掛 WebSocket 工作階段，以及未登錄的外掛所擁有背景工作，都可能
繼續處於活動狀態。
託管平台必須一致地凍結整個程序樹及其
檔案系統，或為其建立快照；這份初始合約無法證明未登錄的工作
處於閒置狀態。

<Tip>
  對於主機喚醒排程，請將面向 OpenClaw 的部分放在程序內
  外掛中，並將具等冪性的完整快照投射到外部主機配接器。
  託管控制器不應匯入外掛 SDK，也不應根據事件差異重建排程
  狀態。請參閱[安全的外部排程
  投射](/zh-TW/plugins/hooks#safe-external-cron-projection)。
</Tip>

## 應用程式碼與外掛程式碼

當程式碼位於 OpenClaw 外部時，請使用閘道 RPC：

- 啟動或觀察代理程式執行的 Node 指令碼
- 呼叫閘道的 CI 工作
- 儀表板和管理面板
- IDE 擴充功能
- 不需要成為頻道外掛的外部橋接器
- 使用模擬或實際閘道傳輸的整合測試

當程式碼在 OpenClaw 內部執行時，請使用外掛 SDK：

- 供應商外掛
- 頻道外掛
- 工具或生命週期鉤子
- 代理程式控管外掛
- 受信任的執行階段輔助工具

外部應用程式不應匯入 `openclaw/plugin-sdk/*`；這些子路徑是供
OpenClaw 載入的外掛使用。

## 相關內容

- [建置閘道用戶端](https://docs.openclaw.ai/gateway/clients)
- [嵌入 OpenClaw](https://docs.openclaw.ai/gateway/embedding)
- [閘道協定](/zh-TW/gateway/protocol)
- [閘道 RPC 參考](/zh-TW/reference/rpc)
- [命令列介面代理程式命令](/zh-TW/cli/agent)
- [命令列介面訊息命令](/zh-TW/cli/message)
- [代理程式迴圈](/zh-TW/concepts/agent-loop)
- [代理程式執行階段](/zh-TW/concepts/agent-runtimes)
- [工作階段](/zh-TW/concepts/session)
- [背景任務](/zh-TW/automation/tasks)
- [ACP 代理程式](/zh-TW/tools/acp-agents)
- [外掛 SDK 概觀](/zh-TW/plugins/sdk-overview)
