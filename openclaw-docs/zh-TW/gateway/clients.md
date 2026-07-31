---
read_when:
    - 在 OpenClaw 儲存庫之外建置操作介面、儀表板或 WebChat 用戶端
    - 實作閘道重新連線、歷史記錄、核准或裝置配對
    - 更新第三方用戶端以支援新的閘道通訊協定版本
summary: 為閘道 WebSocket 通訊協定建置第三方操作介面或 WebChat 用戶端
title: 建置閘道用戶端
x-i18n:
    generated_at: "2026-07-26T07:50:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fa24b196ff1fa28fb3b64d49ac25597f22cf1945aea56029e78e4375f1bdddb7
    source_path: gateway/clients.md
    workflow: 16
---

使用已發布的閘道套件來建置操作員儀表板、WebChat 用戶端及其他第三方應用程式。本指南涵蓋線路合約相關的用戶端生命週期：驗證、功能、重新連線復原、歷史記錄、訂閱及版本升級。

如需訊框格式、交握、錯誤及完整的方法介面，請閱讀[閘道通訊協定規格](https://docs.openclaw.ai/gateway/protocol)。

## 安裝套件

```bash
npm install @openclaw/gateway-client @openclaw/gateway-protocol
```

<Note>
這些套件會隨 OpenClaw 發布系列提供。初始推出期間，在第一個包含套件的 OpenClaw 版本發布之前，npm 可能會傳回 `E404`；請僅在下方登錄頁面可正常存取後安裝。
</Note>

- [`@openclaw/gateway-protocol`](https://www.npmjs.com/package/@openclaw/gateway-protocol)
  提供結構描述、執行階段驗證器、TypeScript 型別、用戶端身分與功能登錄、結構化錯誤讀取器，以及通訊協定版本常數。
  其 npm tarball 也包含產生的
  [`protocol.schema.json`](https://unpkg.com/@openclaw/gateway-protocol/protocol.schema.json)
  機器可讀合約。
- [`@openclaw/gateway-client`](https://www.npmjs.com/package/@openclaw/gateway-client)
  是參考連線實作。Node 用戶端請匯入套件根目錄；瀏覽器安全的通訊協定、裝置驗證及重新連線輔助程式則請匯入 `@openclaw/gateway-client/browser`。

Node 進入點自行管理其 WebSocket 傳輸。瀏覽器主機需提供 WebSocket 介面卡，以及用於裝置身分與裝置權杖的持久性儲存和簽署回呼。

## 選擇範圍並配對裝置

同時呈現核准提示的完整互動式聊天用戶端，應使用下列範圍要求
`role: "operator"`：

| 範圍                 | 用途                                                                                      |
| -------------------- | ----------------------------------------------------------------------------------------- |
| `operator.read`      | `chat.history`、`sessions.list`、`sessions.subscribe`、模型狀態及唯讀事件 |
| `operator.write`     | `chat.send` 及一般工作階段變更                                                |
| `operator.approvals` | 列出、顯示及處理執行或外掛核准                               |

僅在用戶端處理互動式問題時新增 `operator.questions`；
僅在其管理已配對裝置或節點時新增 `operator.pairing`；而
`operator.admin` 僅用於 `config.patch` 等管理操作。
[操作員範圍參考](https://docs.openclaw.ai/gateway/operator-scopes)
定義了完整的方法及核准時規則。

請勿手動編輯 `openclaw.json` 來建立每個用戶端的持有人權杖。請使用 `openclaw configure --section
gateway` 或 `openclaw onboard --gateway-auth ...` 選項設定閘道的共用啟動驗證，然後讓裝置配對簽發用戶端權杖：

1. 在用戶端中持久保存 Ed25519 裝置身分。
2. 等待 `connect.challenge`，簽署與挑戰繫結的裝置承載資料，並傳送
   `connect`，其中包含要求的操作員角色、範圍，以及用於啟動驗證的共用閘道權杖
   或密碼。
3. 如果閘道傳回結構化的 `PAIRING_REQUIRED` 詳細資料，請顯示要求
   ID，並依照 `error.details.recommendedNextStep` 暫停或重試。
4. 在閘道主機上，使用 `openclaw devices list` 檢閱要求，然後
   使用 `openclaw devices approve <requestId>` 核准該筆確切的目前要求。
5. 重新連線，並持久保存包含協商後角色與範圍的 `hello-ok.auth.deviceToken`。
   後續連線請使用該裝置權杖。

升級範圍或角色會建立新的待處理配對要求。權杖輪替無法擴大已核准的配對合約。核准、輪替及撤銷命令請參閱
[裝置命令列介面](https://docs.openclaw.ai/cli/devices)。

## 宣告用戶端功能

`connect.params.caps` 描述用戶端可以使用的選用行為，但
不會授予授權。請從 `GATEWAY_CLIENT_CAPS` 匯入名稱，而不要
重複字串常值：

```ts
import { GATEWAY_CLIENT_CAPS } from "@openclaw/gateway-protocol/client-info";

const caps = [GATEWAY_CLIENT_CAPS.TOOL_EVENTS];
```

目前的登錄包含 `approvals`、`exec-approvals`、`inline-widgets`、
`run-tool-bindings`、`session-scoped-events`、`plugin-approvals`、
`task-suggestions`、`terminal-offset-seq`、`tool-events` 及 `ui-commands`。
請僅宣告用戶端實際實作的功能。

<Warning>
`tool-events` 控制即時工具執行串流。閘道只會將
宣告此功能的連線登錄為某次執行之結構化工具事件的接收者。
若未宣告，該連線不會收到任何即時工具事件，且
交握不會回報錯誤。
</Warning>

受功能限制的代理程式工具，是同一宣告的另一種用途。如果某個
代理程式工具需要用戶端功能，除非發起要求的用戶端已宣告所有必要功能，
否則閘道會省略該工具。

## 重新連線後復原狀態

將每次成功重新連線視為基於持久歷史記錄與目前記憶體內執行狀態的新投影：

1. 重新建立 `sessions.subscribe` 及所選工作階段的
   `sessions.messages.subscribe` 訂閱。
2. 針對所選的 `sessionKey` 呼叫 `chat.history`，並以傳回的 `messages` 投影取代本機持久保存的資料列。
3. 如果存在 `inFlightRun`，請採用其 `runId`、緩衝的 `text` 及選用的
   `plan`。即使 `text` 為空，也要採用該次執行。
4. 讀取 `sessionInfo.hasActiveRun` 及 `sessionInfo.activeRunIds`。判斷保留的執行是否仍擁有
   串流 UI 時，優先使用 `activeRunIds` 中的精確成員資格。若 `hasActiveRun` 為 true 但未列出 ID，可能代表另一個
   作用中的執行階段投影。
5. 依 `payload.runId` 及 `payload.seq` 協調後續 `agent` 事件。
   為每次執行個別維護已接受的最高序號；忽略已見過或較低的序號，並將向前的序號缺口視為重新載入
   權威歷史記錄的理由。

外層事件訊框也有選用的 `seq`，用於排序目前 WebSocket 連線上的事件。建立新連線時會重設。`agent` 事件承載資料中的 `seq`
會依每次執行指派，並排序該次執行的生命週期、助理、計畫、工具及其他串流事件。

## 使用歷史中繼資料與穩定錨點

`chat.history` 傳回的資料列可以包含 `__openclaw` 中繼資料封套：

- `id` 是逐字記錄項目的身分。請用於錨定的歷史記錄要求，
  但不要將其當作唯一的顯示資料列索引鍵。
- `seq` 是正整數的逐字記錄序號。一筆已儲存記錄可以投影
  成多個顯示資料列，因此請將具有相同 `id` 與序號的同層資料列
  保持在一起。
- `kind` 用於識別合成資料列。壓縮邊界使用
  `kind: "compaction"`，而當相符的檢查點記錄了相關指標時，可能包含 `tokensBefore` 及 `tokensAfter`。

使用回應中的 `hasMore` 及 `nextOffset` 值向後分頁。數值
位移描述目前的逐字記錄投影，因此不要將其持久保存為跨重設或壓縮使用的
長期書籤。請改為持久保存 `__openclaw.id`。
若要還原至已知資料列附近，請使用 `messageId` 及傳回該值的
`sessionId` 呼叫 `chat.history`。閘道可以從重設封存
歷史記錄解析該錨點；錨定回應會刻意省略數值分頁中繼資料。

## 訂閱而非輪詢用量

使用 `sessions.list` 載入初始目錄，然後每個連線呼叫一次 `sessions.subscribe`。
依 `sessionKey` 合併 `sessions.changed` 事件。工作階段變更
承載資料可以包含即時 `inputTokens`、`outputTokens`、`totalTokens`、
`totalTokensFresh`、`contextTokens`、`estimatedCostUsd`、回應用量設定
及作用中執行狀態。

部分變更通知只是失效訊號。如果事件省略了檢視所需的
資料列欄位，請重新整理 `sessions.list`。不要輪詢 `usage.cost` 或
`sessions.usage` 來維持即時工作階段清單的最新狀態；這些方法應保留給
隨選彙總或詳細報告。

## 回補執行核准

具有 `operator.approvals` 的用戶端應在
`hello-ok` 完成後立即安裝事件監聽器，接著呼叫 `exec.approval.list`，以回補早於連線建立的要求。
請依核准 ID 協調清單與即時
`exec.approval.requested` / `exec.approval.resolved` 事件，確保與清單要求競速的狀態轉換既不會遺失，也不會重新出現。

## 追蹤通訊協定版本

目前的線路版本是 `4`。一般操作員與 WebChat 用戶端必須
使用 `minProtocol: 4` 及 `maxProtocol: 4` 協商完全相符的目前版本。
只有已驗證的節點用戶端及輕量探測器具有 N-1 接受範圍，
目前為通訊協定 `3` 至 `4`。

通訊協定變更會優先採用新增方式。`protocol.schema.json` 包含 `since`
版本年代中繼資料及核心方法所需的範圍中繼資料，但線路
版本提升對第三方用戶端而言仍是明確的破壞性事件。請固定你測試的
套件版本；當線路版本變更時，一併升級用戶端與閘道；並在每次升級前檢閱
[OpenClaw 變更記錄](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)。

## 相關內容

- [閘道通訊協定](https://docs.openclaw.ai/gateway/protocol)
- [嵌入 OpenClaw](https://docs.openclaw.ai/gateway/embedding)
- [閘道 RPC 參考](https://docs.openclaw.ai/reference/rpc)
- [外部應用程式的閘道整合](https://docs.openclaw.ai/gateway/external-apps)
