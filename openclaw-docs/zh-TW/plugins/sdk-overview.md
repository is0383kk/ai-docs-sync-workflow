---
read_when:
    - 你需要知道要從哪個 SDK 子路徑匯入
    - 你想查閱 OpenClawPluginApi 所有註冊方法的參考資料
    - 你正在查詢特定的 SDK 匯出項目
sidebarTitle: Plugin SDK overview
summary: 匯入對應表、註冊 API 參考與 SDK 架構
title: 外掛 SDK 概覽
x-i18n:
    generated_at: "2026-07-26T07:52:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4f490aa8670c57cfc1a635fb1f5d9950fa1cabdb3d45abbc2295da796edcd52e
    source_path: plugins/sdk-overview.md
    workflow: 16
---

外掛 SDK 是外掛與核心之間的型別化合約。本頁是
**可匯入哪些項目**以及**可註冊哪些項目**的參考資料。

<Note>
  本頁適用於在 OpenClaw 內使用 `openclaw/plugin-sdk/*` 的外掛作者。
  若是要透過閘道執行代理程式的外部應用程式、指令碼、儀表板、CI 工作及 IDE 擴充功能，
  請改用[外部應用程式的閘道整合](/zh-TW/gateway/external-apps)。
</Note>

<Tip>
想找操作指南嗎？請從[建置外掛](/zh-TW/plugins/building-plugins)開始。頻道請參閱[頻道外掛](/zh-TW/plugins/sdk-channel-plugins)，模型供應商請參閱[供應商外掛](/zh-TW/plugins/sdk-provider-plugins)，本機 AI 命令列介面後端請參閱[命令列介面後端外掛](/zh-TW/plugins/cli-backend-plugins)，原生代理程式執行器請參閱[代理程式執行框架外掛](/zh-TW/plugins/sdk-agent-harness)，工具或生命週期鉤子則請參閱[外掛鉤子](/zh-TW/plugins/hooks)。
</Tip>

## 匯入慣例

一律從特定子路徑匯入：

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

每個子路徑都是小型且自含的模組。這可加快啟動速度，並
避免循環相依性問題。對於頻道專屬的進入點／建置輔助工具，
請優先使用 `openclaw/plugin-sdk/channel-core`；`openclaw/plugin-sdk/core` 則保留給
較廣泛的統整介面，以及 `buildChannelConfigSchema` 等共用輔助工具。

對於頻道設定，請透過 `openclaw.plugin.json#channelConfigs`
發布該頻道擁有的 JSON Schema。`plugin-sdk/channel-config-schema`
子路徑用於共用結構描述基礎元件及通用建構器。OpenClaw
隨附的外掛使用 `plugin-sdk/bundled-channel-config-schema` 來保存
隨附頻道的結構描述。該隨附結構描述子路徑並非新外掛應仿效的模式。

<Warning>
  請勿匯入帶有供應商或頻道品牌名稱的便利介面（例如
  `openclaw/plugin-sdk/slack`、`.../discord`、`.../signal`、`.../whatsapp`）。
  隨附外掛會在其自身的 `api.ts` /
  `runtime-api.ts` 統整匯出中組合通用 SDK 子路徑；核心使用端應使用這些外掛區域內的
  統整匯出，或在需求確實跨頻道時新增範圍狹窄的通用 SDK 合約。

少數隨附外掛輔助介面在有受追蹤的擁有者使用情況時，仍會出現在產生的匯出
對應表中。這些介面僅供維護隨附外掛使用，不建議新第三方
外掛將其作為匯入路徑。

`openclaw/plugin-sdk/discord` 與 `openclaw/plugin-sdk/telegram-account`
也會保留為已淘汰的相容性門面，以供受追蹤的擁有者使用。請勿
將這些匯入路徑複製到新外掛中；請改用注入的執行階段輔助工具與
通用頻道 SDK 子路徑。
</Warning>

## 子路徑參考

外掛 SDK 以一組依領域分組的狹窄子路徑公開（外掛
進入點、頻道、供應商、驗證、執行階段、能力、記憶，以及保留的
隨附外掛輔助工具）。如需分組並附連結的完整目錄，請參閱
[外掛 SDK 子路徑](/zh-TW/plugins/sdk-subpaths)。

編譯器進入點清單位於
`scripts/lib/plugin-sdk-entrypoints.json`；具型別的公開匯出會排除
`scripts/lib/plugin-sdk-private-local-only-subpaths.json` 中列出的內部子路徑。
該清單中的正式環境進入點會保留僅供 JavaScript 使用的主機執行階段匯出，
供另行發布的官方外掛使用，而僅供測試的進入點仍不會匯出。執行
`pnpm plugin-sdk:surface` 以稽核公開匯出數量。年代夠久且未由隨附擴充功能正式環境程式碼使用的
已淘汰公開子路徑，會在 `scripts/lib/plugin-sdk-deprecated-public-subpaths.json` 中追蹤；範圍廣泛的
已淘汰重新匯出統整介面則會在
`scripts/lib/plugin-sdk-deprecated-barrel-subpaths.json` 中追蹤。

## 註冊 API

`register(api)` 回呼會接收一個 `OpenClawPluginApi` 物件，其中包含以下
方法：

為工作階段提供外部團隊聊天介面的外掛，可註冊由
`openclaw/plugin-sdk/session-discussion` 匯出的單一程序全域供應商。其 `info({ sessionKey })` 方法
會回報討論無法使用、可開啟，或已經開啟；
`open({ sessionKey })` 會建立或解析討論，並傳回其嵌入網址
與外部網址。註冊另一個供應商會取代目前的供應商。

### 能力註冊

| 方法                                           | 註冊內容                                                                                                                         |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerProvider(...)`                      | 文字推論（LLM）                                                                                                                      |
| `api.registerWorkerProvider(...)`                | 雲端工作節點生命週期租約                                                                                                             |
| `api.registerModelCatalogProvider(...)`          | 文字與媒體生成的模型目錄資料列                                                                                          |
| `api.registerAgentHarness(...)`                  | [實驗性](/zh-TW/plugins/sdk-agent-harness)原生代理程式執行器（Codex、Copilot）                                                         |
| `api.registerCliBackend(...)`                    | 本機命令列介面推論後端                                                                                                               |
| `api.registerChannel(...)`                       | 訊息頻道                                                                                                                         |
| `api.registerEmbeddingProvider(...)`             | 可重複使用的向量嵌入供應商                                                                                                        |
| `api.registerSpeechProvider(...)`                | 文字轉語音／STT 合成                                                                                                            |
| `api.registerRealtimeTranscriptionProvider(...)` | 串流即時轉錄                                                                                                          |
| `api.registerRealtimeVoiceProvider(...)`         | 雙向即時語音工作階段                                                                                                            |
| `api.registerMediaUnderstandingProvider(...)`    | 圖像／音訊／影片分析                                                                                                                |
| `api.registerTranscriptSourceProvider(...)`      | 即時或匯入的會議逐字稿來源；會議外掛可使用 `plugin-sdk/transcripts` 中的 `createMeetingTranscriptSourceProvider` |
| `api.registerImageGenerationProvider(...)`       | 圖像生成                                                                                                                          |
| `api.registerMusicGenerationProvider(...)`       | 音樂生成                                                                                                                          |
| `api.registerVideoGenerationProvider(...)`       | 影片生成                                                                                                                          |
| `api.registerWebFetchProvider(...)`              | 網頁擷取／抓取供應商                                                                                                               |
| `api.registerWebSearchProvider(...)`             | 網頁搜尋                                                                                                                                |
| `api.registerCompactionProvider(...)`            | 可插拔的逐字稿壓縮後端                                                                                                   |

工作節點供應商也必須在 `contracts.workerProviders` 中宣告其 ID。
核心會在 `provision(profile, operationId)` 之前保存持久意圖。供應商會在配置外部資源前驗證設定，並在永久拒絕設定檔時擲回 `WorkerProviderError`。當操作 ID 重複時，`provision` 必須採用相同的租約。
核心會將已驗證的設定檔設定與租約一併保存，並將該快照提供給必須具冪等性的 `destroy({ leaseId, profile })`，以及會傳回 `active`、`destroyed` 或 `unknown` 的 `inspect({ leaseId, profile })`。如此一來，供應商便能在閘道重新啟動或具名設定檔移除後，繼續路由生命週期呼叫。SSH 端點使用 `SecretRef` 作為 `keyRef`，絕不直接內嵌金鑰資料，並納入來自受信任配置輸出的 `hostKey`，其格式必須完全是 `algorithm base64`，不得包含主機名稱或註解。核心會固定 `hostKey`，且絕不信任首次連線取得的金鑰。建立動態 `keyRef` 的供應商可實作 `resolveSshIdentity({ leaseId, profile, keyRef })`；若存在，該解析器即為權威來源，而未實作的供應商則使用已設定的通用祕密解析器。
具有可續期租約的供應商也可實作 `renew(leaseId)`。
`inspect` 在暫時性或不確定的失敗時必須擲回錯誤；只有在權威確認不存在時才能傳回 `unknown`。核心會將使用中的本機記錄標記為孤立，或在已保存銷毀要求後，將該不存在狀態視為拆除完成。

使用 `api.registerEmbeddingProvider(...)` 註冊的嵌入供應商，也必須
列在外掛資訊清單的 `contracts.embeddingProviders` 中。這是供可重複使用向量生成使用的
通用嵌入介面。記憶搜尋可使用這個通用供應商介面。較舊的
`api.registerMemoryEmbeddingProvider(...)` 與
`contracts.memoryEmbeddingProviders` 介面屬於已淘汰的相容性設計，供現有的記憶專用供應商
遷移期間使用。

仍公開執行階段 `batchEmbed(...)` 的記憶專用供應商，會維持
現有的逐檔批次合約，除非其執行階段明確設定
`sourceWideBatchEmbed: true`。這項選擇性啟用功能可讓記憶主機在主機批次限制內，
透過一次 `batchEmbed(...)` 呼叫提交來自多個已變更記憶檔案及已啟用來源的區塊。
上傳 JSONL 要求檔案的批次配接器，必須在達到上傳大小上限及要求數量上限前
拆分供應商工作。供應商必須依照與
`batch.chunks` 相同的順序，為每個輸入區塊傳回一個嵌入；若供應商預期逐檔批次，
或無法在較大的全來源工作中維持輸入順序，請省略此旗標。

### 工具與命令

若是工具名稱固定的簡易純工具外掛，請使用
[`defineToolPlugin`](/zh-TW/plugins/tool-plugins)。
混合型外掛或完全動態的工具註冊，請直接使用 `api.registerTool(...)`。

| 方法                                 | 註冊內容                                                                                                                        |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerTool(tool, opts?)`        | 代理程式工具（必要或 `{ optional: true }`）                                                                                            |
| `api.registerCommand(def)`             | 自訂命令（略過 LLM）                                                                                                        |
| `api.registerNodeHostCommand(command)` | 由 `openclaw node run` 處理的命令；選用的 `agentTool` 中繼資料可在節點連線時，將其公開為代理程式可見的工具 |

當代理程式需要簡短且由命令擁有的路由提示時，外掛命令可設定
`agentPromptGuidance`。該文字應只描述命令本身；請勿將
供應商或外掛專屬政策加入核心提示建構器。

指引項目可以是套用於所有提示介面的舊式字串，也可以是
結構化項目：

```ts
agentPromptGuidance: [
  "全域命令提示。",
  { text: "僅在 OpenClaw 主要提示中顯示此內容。", surfaces: ["openclaw_main"] },
];
```

結構化的 `surfaces` 可包含 `openclaw_main`、`codex_app_server`、
`cli_backend`、`acp_backend` 或 `subagent`。`pi_main` 仍是
`openclaw_main` 的已棄用別名。若刻意讓指引套用至所有介面，請省略 `surfaces`。請勿
傳入空的 `surfaces` 陣列；系統會拒絕該陣列，避免意外遺失範圍設定而使其
成為全域提示文字。

原生 Codex app-server 開發人員指示比其他提示
介面更嚴格：只有明確限定於 `codex_app_server` 的指引，才會提升至
該較高優先順序的通道。為了相容性，舊版字串指引及未限定範圍的結構化
指引仍可供非 Codex 提示介面使用。

節點主機命令會在已連線的節點主機上執行，而不是在閘道
程序內執行。若存在 `agentTool`，節點會在成功連線至
閘道後發布描述元；只有在該節點保持連線，且描述元的 `command` 位於節點
核准的命令介面中時，閘道才會將其提供給代理程式執行。將 `agentTool.defaultPlatforms` 設為
讓非危險命令加入預設節點命令允許清單；否則必須有
明確的 `gateway.nodes.commands.allow` 或節點叫用原則。`agentTool.name`
必須符合提供者安全規則：以字母開頭，僅使用字母、數字、
底線或連字號，且不得超過 64 個字元。由 MCP 支援的節點工具
可設定 `agentTool.mcp` 中繼資料，讓目錄與工具搜尋介面顯示
遠端 MCP 伺服器／工具身分，但執行仍會透過
公告的節點命令進行。

### 基礎架構

| 方法                                          | 註冊項目                                                      |
| ----------------------------------------------- | ---------------------------------------------------------------------- |
| `api.registerHook(events, handler, opts?)`      | 事件鉤子                                                             |
| `api.registerHttpRoute(params)`                 | 閘道 HTTP 端點                                                  |
| `api.registerGatewayMethod(name, handler)`      | 閘道 RPC 方法                                                     |
| `api.registerGatewayDiscoveryService(service)`  | 本機閘道探索公告器                                     |
| `api.registerCli(registrar, opts?)`             | 命令列介面子命令                                                         |
| `api.registerNodeCliFeature(registrar, opts?)`  | `openclaw nodes` 下的節點功能命令列介面                                |
| `api.registerService(service)`                  | 背景服務                                                     |
| `api.registerInteractiveHandler(registration)`  | 互動式處理常式                                                    |
| `api.registerAgentToolResultMiddleware(...)`    | 執行階段工具結果中介軟體                                         |
| `api.registerMemoryPromptSupplement(builder)`   | 可附加的記憶相鄰提示區段                                |
| `api.registerMemoryPromptPreparation(prepare)`  | 記憶相鄰提示區段的非同步準備                 |
| `api.registerMemoryCorpusSupplement(adapter)`   | 可附加的記憶搜尋／讀取語料庫                                     |
| `api.registerHostedMediaResolver(resolver)`     | 瀏覽器樣式託管媒體 URL 的解析器                           |
| `api.registerMcpServerConnectionResolver(...)`  | 靜態伺服器名稱的每請求者 MCP 傳輸（`url`/`headers`） |
| `api.registerTextTransforms(transforms)`        | 外掛擁有的提示／訊息相容性文字改寫                |
| `api.registerConfigMigration(migrate)`          | 外掛執行階段載入前執行的輕量設定遷移           |
| `api.registerMigrationProvider(provider)`       | `openclaw migrate` 的匯入器                                        |
| `api.registerAutoEnableProbe(probe)`            | 可自動啟用此外掛的設定探測器                          |
| `api.registerReload(registration)`              | 重新載入處理的重新啟動／熱更新／無動作設定前綴原則              |
| `api.registerNodeHostCommand(command)`          | 提供給已配對節點的命令處理常式                                |
| `api.registerNodeInvokePolicy(policy)`          | 節點叫用命令的允許清單／核准原則                    |
| `api.registerSecurityAuditCollector(collector)` | `openclaw security audit` 的發現項目收集器                       |

#### 確認回應後的網路鉤子工作

在處理完成前即確認請求的網路鉤子路由，必須將
該分離工作移至其自身受追蹤的准入根：

```typescript
import { runDetachedWebhookWork } from "openclaw/plugin-sdk/webhook-request-guards";

void runDetachedWebhookWork(() => processWebhookEvent(event)).catch((error) => {
  runtime.error?.(`網路鉤子分派失敗：${String(error)}`);
});
```

請在 HTTP 請求仍獲准入時同步呼叫 `runDetachedWebhookWork(...)`。
此輔助程式會立即保留獨立根，接著於下一個微任務中啟動
回呼，讓請求處理常式可以先寫入
確認回應。傳回的 Promise 會採用回呼結果；呼叫端
仍負責處理拒絕。這可讓確認回應後的佇列工作保持獲准，並讓
重新啟動或暫停時的排空作業等待其完成。若處理常式會在傳回前等待所有處理
完成，則不需要此輔助程式。

#### 以請求者為範圍的 MCP 連線

請將 MCP 伺服器**身分**（名稱、工具篩選器）靜態保留在 `mcp.servers`、
原生外掛的 `mcpServers` 資訊清單欄位，或套件資訊清單中。可選擇註冊連線解析器，讓每個受信任的
訊息請求者使用自己的傳輸：

```ts
api.registerMcpServerConnectionResolver({
  serverName: "user-email",
  resolve: async (ctx) => {
    // ctx.requesterSenderId 受主機信任；切勿在此虛構傳送者身分。
    const token = await lookupUserToken(ctx.requesterSenderId);
    if (!token) {
      return null; // 在目前執行中省略此伺服器
    }
    return {
      url: "https://mcp.example.com/email",
      headers: { Authorization: `Bearer ${token}` },
    };
  },
});
```

契約注意事項：

- 解析器內容只攜帶受信任的主機身分（`requesterSenderId`、
  選用的 `agentAccountId`／`messageChannel`）。未來可用附加方式加入受信任欄位（例如
  排程／子代理程式使用者內容）。
- 一個伺服器名稱只由一個外掛擁有：若另一個
  外掛為相同的 `serverName` 重複註冊
  `registerMcpServerConnectionResolver`，系統會以錯誤診斷拒絕（第一次註冊優先），因此
  連線擁有權絕不取決於外掛載入順序。
- 工具名稱衍生自完整的已宣告伺服器集合，因此部分解析
  絕不會在不同請求者或回合之間變更安全伺服器名稱。核心不會
  驗證不同請求者端點是否提供相同的工具結構描述；解析器
  必須將每個請求者指向相同的邏輯服務，否則每個請求者的工具
  結構描述（以及提示快取穩定性）會有所差異。
- 沒有受信任 `requesterSenderId` 的執行（排程、子代理程式、心跳偵測、公開
  閘道）絕不會具現化以請求者為範圍的伺服器。不存在共用的
  後援連線。
- 每個伺服器的 `resolve` 上限為 10 秒；逾時或擲回錯誤時，會在該次
  執行中省略該伺服器，而不會導致靜態 MCP 失敗。
- 每個請求者的已解析連線最多每 5 分鐘重新驗證一次：
  輪替會使用新的認證資訊重建傳輸，而 `null` 結果
  會撤銷連線（即使在工作階段途中，也會處置快取的執行階段）。因此，已撤銷或
  已輪替的認證資訊可能仍會繼續使用最多 5 分鐘。
- 已解析的 `headers` 絕不會記錄或持久保存；核心只保留暫時性的
  記憶體內鍵控摘要（程序本機 HMAC）以偵測認證資訊輪替，並
  向記錄／偵錯擷取遮蔽登錄表註冊已解析的標頭／URL 認證資訊值。
- 以請求者為範圍的伺服器不會建立 MCP App 檢視：檢視的存續時間會超過
  經請求者驗證的執行，而閘道檢視邊界沒有請求者
  身分，因此這些伺服器的應用程式預覽會維持故障關閉。工具結果
  不受影響。
- 沒有解析器的靜態伺服器會維持現有的工作階段範圍生命週期。
- **Harness 傳遞規則：**以請求者為範圍的伺服器絕不會進入 Harness 原生
  MCP 用戶端設定（Codex 執行緒 `mcp_servers`、命令列介面 `-c mcp_servers=…`，或任何
  其他工作階段共用的 MCP 投影）。Harness 會改以執行範圍
  工具提供這些伺服器：
  - 內嵌執行器：工作階段 MCP 執行階段 + 套件工具（靜態 + 範圍限定）。
  - Codex app-server：透過
    `materializeRequesterScopedMcpToolsForHarnessRun` 提供動態工具（僅限範圍限定；
    靜態伺服器仍使用 Codex 的原生 MCP 用戶端）。
- 範圍限定工具**規格**在該工作階段第一次成功解析後會保持工作階段穩定，
  因此共用執行緒的 Harness（Codex）不會在
  傳送者變更時輪替執行緒。在任何請求者解析前，不會公告範圍限定規格。
- 共用執行緒 Harness 上未經驗證的請求者仍會看到公告的
  範圍限定工具；呼叫其中任一工具時，會針對該請求者傳回清楚的未連線工具錯誤。
  OpenClaw 絕不會改用另一位請求者的認證資訊作為後援。

記憶提示補充建構器會接收選用的 `agentId`、
`agentSessionKey` 和 `sandboxed` 內容。記憶語料庫補充 `search`
和 `get` 呼叫會接收選用的 `agentId` 和 `sandboxed` 內容。具有
代理程式擁有之儲存空間的外掛，應為每次呼叫解析該儲存空間，而不是
在註冊期間擷取單一全域路徑。若多代理程式作業需要代理程式 ID 卻
未提供，應故障關閉，而不是選擇任意
代理程式。

當提示文字取決於非同步
外掛狀態時，請使用 `registerMemoryPromptPreparation(...)`。回呼會在每次完整代理程式提示前執行一次，並接收
與同步記憶提示建構器相同的工具、代理程式、工作階段及沙箱內容。
載入持久化狀態前，請驗證目前的儲存空間擁有者執行個體，接著只傳回該次執行所需的行。
OpenClaw 會凍結這些行，並將不可變的結果交給同步提示組裝程序。
持久化、不可部分完成的替換及擁有者移除時的刪除作業，應留在所屬外掛內；請勿
從提示建構器輪詢或讀取檔案。

Telegram 互動式處理常式可傳回 `{ submitText }`，在處理常式成功後，將文字透過
Telegram 的一般入站代理程式路徑傳送。當入站原則略過文字或處理失敗時，OpenClaw 會保留
回呼按鈕，讓使用者可在阻擋條件改變後重試。此結果欄位
僅適用於 Telegram；其他頻道會維持各自的互動式結果契約。

### 工作流程外掛的主機鉤子

主機鉤子是供需要參與主機
生命週期，而不只是新增提供者、頻道或工具的外掛使用的 SDK 介面。它們是
通用契約；計畫模式可以使用，核准工作流程、
工作區原則閘門、背景監控程式、設定精靈及 UI 伴隨
外掛也可以使用。

| 方法                                                                               | 其所擁有的契約                                                                                                                                           |
| ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.session.state.registerSessionExtension(...)`                                    | 由外掛擁有、與 JSON 相容的工作階段狀態，透過閘道工作階段投影                                                                             |
| `api.session.workflow.enqueueNextTurnInjection(...)`                                 | 注入單一工作階段之下一個代理程式回合的持久化恰好一次內容                                                                             |
| `api.registerTrustedToolPolicy(...)`                                                 | 由資訊清單管控、受信任且在外掛之前執行的工具政策，可封鎖或改寫工具參數                                                                        |
| `api.registerToolMetadata(...)`                                                      | 工具目錄的顯示中繼資料，不變更工具實作                                                                                     |
| `api.registerCommand(...)`                                                           | 限定範圍的外掛命令；命令結果可設定 `continueAgent: true` 或 `suppressReply: true`；Discord 原生命令支援 `descriptionLocalizations` |
| `api.session.controls.registerControlUiDescriptor(...)`                              | 工作階段、工具、執行、設定或分頁介面的控制 UI 貢獻描述元                                                                      |
| `api.lifecycle.registerRuntimeLifecycle(...)`                                        | 在重設／刪除／重新載入路徑上，清理外掛所擁有執行階段資源的回呼                                                                          |
| `api.agent.events.registerAgentEventSubscription(...)`                               | 工作流程狀態與監控器的清理後事件訂閱                                                                                              |
| `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`  | 每次執行的外掛暫存狀態，會在終止執行生命週期時清除                                                                                             |
| `api.session.workflow.registerSessionSchedulerJob(...)`                              | 外掛所擁有之排程器工作的清理中繼資料；不會排程工作或建立任務記錄                                                            |
| `api.session.workflow.sendSessionAttachment(...)`                                    | 僅限內建外掛、由主機居中協調，將檔案附件傳送至作用中的直接外送工作階段路由                                                            |
| `api.session.workflow.scheduleSessionTurn(...)` / `unscheduleSessionTurnsByTag(...)` | 僅限內建外掛、由排程支援的排定工作階段回合，以及基於標籤的清理                                                                                    |
| `api.session.controls.registerSessionAction(...)`                                    | 用戶端可透過閘道分派的具型別工作階段動作                                                                                             |

`surface: "tab"` 描述元會在控制 UI 中新增側邊欄分頁。作用中
外掛的分頁描述元會透過閘道 hello（`controlUiTabs`）公告給儀表板
用戶端，因此該分頁只會在外掛啟用時顯示。
內建外掛可為其分頁提供第一級儀表板檢視；其他
外掛可將 `path` 設為外掛 HTTP 路由（請參閱
`api.registerHttpRoute(...)`），儀表板會在沙箱化框架中呈現該路由。
`icon` 是儀表板圖示名稱提示，`group` 選擇側邊欄區段
（`control` 或 `agent`），`order` 決定外掛分頁間的排序，而 `requiredScopes`
會對不具備這些操作員範圍的連線隱藏分頁：

若是受閘道保護的外部分頁，請在相同外掛的
`auth: "gateway"` HTTP 路由下註冊描述元 `path`。完成已驗證的啟動程序後，瀏覽器會取得
限定於該外掛與路由根目錄、短效且設為 HttpOnly 的授權，讓
沙箱化框架無須將閘道持有人權杖複製到其 URL
或 JavaScript 中即可載入。已驗證的父頁面會在外部分頁
作用期間續期授權，並在導覽或瀏覽器恢復後掛載分頁前續期。它也會在掛載前，
從同一個不透明沙箱探測授權，因此會封鎖 Cookie 的瀏覽器
隱私模式將採取封閉式失敗，顯示無法使用的面板。
框架授權僅接受 `GET` 和 `HEAD`，且一律帶有
`operator.read`；`requiredScopes` 控制分頁可見性，但絕不擴大
Cookie 授權。變更操作仍須透過明確經閘道驗證的父頁面或
持有人介面執行。外部分頁需要 HTTPS／Tailscale Serve 或
瀏覽器信任的回送來源；區域網路主機上的純 HTTP 會顯示
安全內容錯誤，而不會掛載無法驗證的面板。
完全封鎖第三方 Cookie 也會使受閘道保護的分頁無法使用。
與所有原生外掛介面相同，框架仍位於已安裝
外掛的信任邊界內；OpenClaw 不會將已安裝外掛視為彼此
隔離的瀏覽器安全主體。
Cookie 授權使用瀏覽器的主機名稱邊界，而非連接埠邊界。即使使用不同
連接埠，也請勿在閘道主機名稱上共同託管互不信任的服務。
由外掛管理驗證的分頁會保留其直接 iframe 行為，且不會
要求或需要此閘道授權。

```typescript
api.session.controls.registerControlUiDescriptor({
  surface: "tab",
  id: "logbook",
  label: "日誌",
  description: "以螢幕快照建構時間軸，呈現你的一天。",
  icon: "sun",
  group: "control",
  requiredScopes: ["operator.write"],
});
```

新的外掛程式碼請使用分組命名空間：

- `api.session.state.registerSessionExtension(...)`
- `api.session.workflow.enqueueNextTurnInjection(...)`
- `api.session.workflow.registerSessionSchedulerJob(...)`
- `api.session.workflow.sendSessionAttachment(...)`
- `api.session.workflow.scheduleSessionTurn(...)`
- `api.session.workflow.unscheduleSessionTurnsByTag(...)`
- `api.session.controls.registerSessionAction(...)`
- `api.session.controls.registerControlUiDescriptor(...)`
- `api.agent.events.registerAgentEventSubscription(...)`
- `api.agent.events.emitAgentEvent(...)`
- `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`
- `api.lifecycle.registerRuntimeLifecycle(...)`

等效的扁平方法仍可作為已棄用的相容性
別名供現有外掛使用。請勿新增直接呼叫
`api.registerSessionExtension`、`api.enqueueNextTurnInjection`、
`api.registerControlUiDescriptor`、`api.registerRuntimeLifecycle`、
`api.registerAgentEventSubscription`、`api.emitAgentEvent`、
`api.setRunContext`、`api.getRunContext`、`api.clearRunContext`、
`api.registerSessionSchedulerJob`、`api.registerSessionAction`、
`api.sendSessionAttachment`、`api.scheduleSessionTurn` 或
`api.unscheduleSessionTurnsByTag` 的外掛程式碼。

`scheduleSessionTurn(...)` 是閘道
排程器之上的工作階段限定便利介面。排程擁有計時，並在
回合執行時建立背景任務記錄；外掛 SDK 只會限制目標工作階段、外掛擁有的
命名與清理。當工作本身需要持久化的多步驟 Task Flow 狀態時，請在排定的
回合內使用 `api.runtime.tasks.managedFlows`。

這些契約刻意分離權限：

- 外部外掛可擁有工作階段擴充、UI 描述元、命令、工具
  中繼資料、下一回合注入及一般掛鉤。
- 受信任工具政策會在一般 `before_tool_call` 掛鉤之前執行，並且
  受主機信任。內建政策會先執行；已安裝外掛的政策需要
  明確啟用，並將其本機 ID 加入
  `contracts.trustedToolPolicies`，接著依外掛載入順序執行。政策 ID
  的範圍限定於註冊該政策的外掛。
- 保留命令的擁有權僅限內建外掛。外部外掛應使用自己的
  命令名稱或別名。
- `allowPromptInjection=false` 會停用會修改提示詞的掛鉤，包括
  `agent_turn_prepare`、`before_prompt_build`、`heartbeat_prompt_contribution`
  和 `enqueueNextTurnInjection`。

非 Plan 使用者的範例：

| 外掛原型             | 使用的掛鉤                                                                                                                             |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| 核准工作流程            | 工作階段擴充、命令接續、下一回合注入、UI 描述元                                                            |
| 預算／工作區政策閘門 | 受信任工具政策、工具中繼資料、工作階段投影                                                                                 |
| 背景生命週期監控器 | 執行階段生命週期清理、代理程式事件訂閱、工作階段排程器擁有權／清理、心跳偵測提示詞貢獻、UI 描述元 |
| 設定或初始設定精靈   | 工作階段擴充、限定範圍的命令、控制 UI 描述元                                                                              |

<Note>
  保留的核心管理命名空間（`config.*`、`exec.approvals.*`、`wizard.*`、
  `update.*`）一律維持 `operator.admin`，即使外掛嘗試指派
  範圍更窄的閘道方法範圍亦然。外掛所擁有的方法應優先使用
  外掛專屬前綴。
</Note>

<Accordion title="何時使用工具結果中介軟體">
  內建外掛以及明確啟用且具備相符
  資訊清單契約的已安裝外掛，可在需要於執行後且執行階段將工具結果
  回饋給模型之前改寫該結果時，使用 `api.registerAgentToolResultMiddleware(...)`。
  這是供 tokenjuice 等非同步輸出歸約器使用、受信任且不依賴執行階段的介面。

外掛必須為每個目標
執行階段宣告 `contracts.agentToolResultMiddleware`，例如 `["openclaw", "codex"]`。不具備該
契約或未明確啟用的已安裝外掛無法註冊此中介軟體；對於不需要
模型前工具結果時機的工作，請繼續使用一般 OpenClaw 外掛掛鉤。舊有
僅限內嵌執行器的擴充工廠註冊路徑已移除。
</Accordion>

### 閘道探索註冊

`api.registerGatewayDiscoveryService(...)` 可讓外掛在 mDNS／Bonjour 等本機探索
傳輸上公告作用中的閘道。啟用本機探索時，OpenClaw 會在
閘道啟動期間呼叫該服務、傳入目前的閘道連接埠與非機密 TXT 提示資料，
並在閘道關閉期間呼叫傳回的
`stop` 處理常式。

```typescript
api.registerGatewayDiscoveryService({
  id: "my-discovery",
  async advertise(ctx) {
    const handle = await startMyAdvertiser({
      gatewayPort: ctx.gatewayPort,
      tls: ctx.gatewayTlsEnabled,
      displayName: ctx.machineDisplayName,
    });
    return { stop: () => handle.stop() };
  },
});
```

閘道探索外掛不得將公告的 TXT 值視為機密或
驗證資訊。探索只提供路由提示；信任仍由閘道驗證與 TLS
釘選負責。

### 命令列介面註冊中繼資料

`api.registerCli(registrar, opts?)` 接受兩種命令中繼資料：

- `commands`：由註冊者擁有的明確命令名稱
- `descriptors`：用於命令列介面說明、
  路由及延遲外掛命令列介面註冊的剖析階段命令描述元
- `parentPath`：巢狀命令群組的選用父命令路徑，例如
  `["nodes"]`

對於配對節點功能，請優先使用
`api.registerNodeCliFeature(registrar, opts?)`。它是
`api.registerCli(..., { parentPath: ["nodes"] })` 的小型包裝函式，可將
`openclaw nodes canvas` 等命令明確標示為外掛所擁有的節點功能。

如果你希望外掛命令在一般根命令列介面路徑中維持延遲載入，
請提供涵蓋該註冊者公開之每個頂層命令根的 `descriptors`。

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "管理 Matrix 帳號、驗證、裝置及個人資料狀態",
        hasSubcommands: true,
      },
    ],
  },
);
```

巢狀命令會接收解析後的父命令作為 `program`：

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerNodesCanvasCommands } = await import("./src/cli.js");
    registerNodesCanvasCommands(program);
  },
  {
    parentPath: ["nodes"],
    descriptors: [
      {
        name: "canvas",
        description: "從已配對的節點擷取或算繪畫布內容",
        hasSubcommands: true,
      },
    ],
  },
);
```

只有在不需要延遲註冊根命令列介面時，才單獨使用 `commands`。
該即時相容路徑仍受支援，但不會安裝由描述項支援的預留位置，
以供剖析時延遲載入。

### 命令列介面後端註冊

`api.registerCliBackend(...)` 可讓外掛擁有本機
AI 命令列介面後端（例如 `claude-cli` 或 `my-cli`）的預設設定。

- 後端 `id` 會成為模型參照（例如 `my-cli/gpt-5`）中的供應商前綴。
- 後端 `config` 是權威命令介接器：argv、環境、
  剖析器、工作階段、影像及可靠性行為均位於外掛程式碼中。
- 使用者透過模型參照或模型範圍的 `agentRuntime.id` 選取後端；
  `openclaw.json` 不會重寫介接器。
- 當已註冊的靜態欄位需要可感知執行階段的
  正規化處理時，請使用 `normalizeConfig`。
- 對於屬於命令列介面方言的請求範圍 argv 重寫，請使用 `resolveExecutionArgs`，
  例如將 OpenClaw 思考層級對應至原生的投入程度
  旗標。該掛鉤會接收 `ctx.executionMode`；請使用 `"side-question"`，
  為暫時性的 `/btw` 呼叫新增後端原生隔離旗標。若這些旗標
  能可靠停用原本始終啟用命令列介面的原生工具，
  也請宣告 `sideQuestionToolMode: "disabled"`。
- 對於後端擁有的啟動環境或暫時性
  驗證／設定橋接，請使用 `prepareExecution`。其 `ctx.contextTokenBudget` 是為該次執行選取的有效權杖
  限制，因此具備原生壓縮功能的後端可對齊自身的
  閾值，而無須在核心中加入供應商特定分支。當後端暫存必須擴充隨附的 MCP 設定時，
  它也會接收核心準備的 `ctx.env`。
- 能針對特定執行停用所有原生工具的後端，可宣告
  `nativeToolMode: "selectable"`。受限呼叫會傳入精確的
  `ctx.toolAvailability.native` 清單及標準的
  `ctx.toolAvailability.openClaw` 名稱。請宣告
  `toolAvailabilityEnforcement: "execution-args"` 並在最終全新／恢復 argv 中強制執行該合約，或宣告 `"prepare-execution"`、
  在暫存政策中強制執行，並傳回 `toolAvailabilityEnforced: true`。OpenClaw 會針對排程 `toolsAllow` 等執行階段能力限制
  停用原生工具，且當宣告的強制執行路徑不完整時採取封閉式失敗。

如需端對端撰寫指南，請參閱
[命令列介面後端外掛](/zh-TW/plugins/cli-backend-plugins)。

### 獨佔插槽

| 方法                                     | 註冊內容                                                                                                                                                                                  |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | 情境引擎（一次只能啟用一個）。當主機可提供模型／供應商／模式診斷時，生命週期回呼會接收 `runtimeSettings`；較舊的嚴格引擎會在不含該鍵的情況下重試。 |
| `api.registerMemoryCapability(capability)` | 統一記憶能力                                                                                                                                                                          |

### 已棄用的記憶嵌入介接器

| 方法                                         | 註冊內容                              |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | 作用中外掛的記憶嵌入介接器 |

- `registerMemoryCapability` 是獨佔的記憶外掛 API。
- `registerMemoryCapability` 也可公開 `publicArtifacts.listArtifacts(...)`，
  供主機管理的匯出使用。列舉這些已宣告成品的配套外掛，
  在專用公開消費端 API 出現前，仍會使用保留的
  `openclaw/plugin-sdk/memory-host-core` 門面所提供的 `listActiveMemoryPublicArtifacts(...)`；
  它們不得存取其他外掛的私有配置。
- `MemoryFlushPlan.model` 可將清除回合固定至精確的 `provider/model`
  參照（例如 `ollama/qwen3:8b`），而不繼承作用中的後援
  鏈。
- `registerMemoryEmbeddingProvider` 已棄用。新的嵌入供應商
  應使用 `api.registerEmbeddingProvider(...)` 和
  `contracts.embeddingProviders`。
- 現有的記憶專用供應商在遷移
  期間會繼續運作，但外掛檢查會將非隨附外掛的此情況
  回報為相容性技術債。

### 事件與生命週期

| 方法                                       | 功能                  |
| -------------------------------------------- | ----------------------------- |
| `api.on(hookName, handler, opts?)`           | 型別化生命週期掛鉤          |
| `api.onConversationBindingResolved(handler)` | 對話繫結回呼 |

如需範例、常見掛鉤名稱及防護
語意，請參閱[外掛掛鉤](/zh-TW/plugins/hooks)。

### 掛鉤決策語意

`before_install` 是外掛執行階段生命週期掛鉤，而非操作員安裝
政策介面。當允許／封鎖決策必須涵蓋命令列介面及由閘道支援的安裝或更新路徑時，
請使用 `security.installPolicy`。

- `before_tool_call`：傳回 `{ block: true }` 即為終止。一旦任何處理常式設定它，便會略過優先順序較低的處理常式。
- `before_tool_call`：傳回 `{ block: false }` 會視為未作決策（等同省略 `block`），而非覆寫。
- `before_install`：傳回 `{ block: true }` 即為終止。一旦任何處理常式設定它，便會略過優先順序較低的處理常式。
- `before_install`：傳回 `{ block: false }` 會視為未作決策（等同省略 `block`），而非覆寫。
- `reply_dispatch`：傳回 `{ handled: true, ... }` 即為終止。一旦任何處理常式宣告分派，便會略過優先順序較低的處理常式及預設模型分派路徑。
- `message_sending`：傳回 `{ cancel: true }` 即為終止。一旦任何處理常式設定它，便會略過優先順序較低的處理常式。
- `message_sending`：傳回 `{ cancel: false }` 會視為未作決策（等同省略 `cancel`），而非覆寫。
- `message_received`：需要傳入執行緒／主題路由時，請使用型別化的 `threadId` 欄位。將 `metadata` 保留給頻道特定的額外資料。
- `message_sending`：請先使用型別化的 `replyToId`／`threadId` 路由欄位，再後援至頻道特定的 `metadata`。
- `gateway_start`：對於閘道擁有的啟動狀態，請使用 `ctx.config`、`ctx.workspaceDir` 和 `ctx.getCron?.()`，而非依賴內部 `gateway:startup` 掛鉤。此時排程可能仍在載入。
- `cron_reconciled`：在啟動或排程器重新載入後，重建完整的外部排程投影。它包含 `reason` 及有效的 `enabled` 狀態（包括 `enabled: false`），而 `ctx.getCron?.()` 會傳回精確的已協調排程器。將 `ctx.abortSignal` 傳入持久投影工作；當該排程器快照被取代或閘道關閉時，它會中止。
- `cron_changed`：觀察閘道擁有的排程生命週期變更。`scheduled` 和 `removed` 事件是提交後的協調提示，而非有序的差異記錄。當工作沒有下一次喚醒時間時，排程事件的 `event.nextRunAtMs` 不存在；移除事件仍會攜帶已刪除工作的快照。

外部喚醒排程器應對 `cron_changed` 事件進行防抖或合併，
然後從 `cron_reconciled` 最後擷取的排程器重新讀取完整的持久檢視。
請勿採用 `cron_changed` 情境中的排程器：來自較舊排程器且已脫離的提示，
可能會與之後的重新載入重疊。

將 `cron_reconciled` 用作完整快照觸發程序，以處理
閘道啟動或排程器替換時載入的持久狀態。僅重新熱載入外掛時不會重播它。
觀察處理常式會平行執行，而發後即忘的分派可能重疊，
因此消費端不得依賴事件完成順序。
應繼續以 OpenClaw 作為到期檢查與執行的真實資料來源。

如需具備持久替換、重試／退避及乾淨
關閉功能的單次執行介接器，請參閱[安全的外部排程投影](/zh-TW/plugins/hooks#safe-external-cron-projection)。

### API 物件欄位

| 欄位                    | 型別                      | 說明                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | 外掛 ID                                                                                   |
| `api.name`               | `string`                  | 顯示名稱                                                                                |
| `api.version`            | `string?`                 | 外掛版本（選用）                                                                   |
| `api.description`        | `string?`                 | 外掛說明（選用）                                                               |
| `api.source`             | `string`                  | 外掛來源路徑                                                                          |
| `api.rootDir`            | `string?`                 | 外掛根目錄（選用）                                                            |
| `api.config`             | `OpenClawConfig`          | 目前的設定快照（若可用，則為作用中的記憶體內執行階段快照）                  |
| `api.pluginConfig`       | `Record<string, unknown>` | 來自 `plugins.entries.<id>.config` 的外掛專用設定                                   |
| `api.runtime`            | `PluginRuntime`           | [執行階段輔助工具](/zh-TW/plugins/sdk-runtime)                                                     |
| `api.logger`             | `PluginLogger`            | 範圍限定的記錄器（`debug`、`info`、`warn`、`error`）                                            |
| `api.registrationMode`   | `PluginRegistrationMode`  | 目前的載入模式；`"setup-runtime"` 是完整進入點啟動前的輕量啟動／設定時段 |
| `api.resolvePath(input)` | `(string) => string`      | 解析相對於外掛根目錄的路徑                                                        |

## 內部模組慣例

在你的外掛中，內部匯入請使用本機 barrel 檔案：

```text
my-plugin/
  api.ts            # 供外部消費端使用的公開匯出
  runtime-api.ts    # 僅供內部使用的執行階段匯出
  index.ts          # 外掛進入點
  setup-entry.ts    # 僅供設定使用的輕量進入點（選用）
```

<Warning>
  絕勿在正式環境程式碼中透過 `openclaw/plugin-sdk/<your-plugin>`
  匯入你自己的外掛。內部匯入應透過 `./api.ts` 或
  `./runtime-api.ts`。SDK 路徑僅供外部合約使用。
</Warning>

透過外觀載入的內建外掛公開介面（`api.ts`、`runtime-api.ts`、
`index.ts`、`setup-entry.ts`，以及類似的公開進入點檔案）會在 OpenClaw
已執行時優先使用作用中的執行階段設定快照。如果執行階段快照尚不存在，
則會退回使用磁碟上已解析的設定檔。封裝的內建外掛外觀應透過 OpenClaw 的外掛
外觀載入器載入；直接從 `dist/extensions/...` 匯入，會略過封裝安裝在載入外掛自有
程式碼時所使用的資訊清單與執行階段輔助程序檢查。

當輔助程式刻意限定於特定提供者，且尚不適合放入通用 SDK
子路徑時，提供者外掛可以公開範圍有限的外掛區域合約彙整入口。
內建範例：

- **Anthropic**：公開的 `api.ts` / `contract-api.ts` 介面，用於 Claude
  beta 標頭和 `service_tier` 串流輔助程式。
- **`@openclaw/openai-provider`**：`api.ts` 會匯出提供者建構器、
  預設模型輔助程式和即時提供者建構器。
- **`@openclaw/openrouter-provider`**：`api.ts` 會匯出提供者建構器，
  以及初始設定／設定輔助程式。

<Warning>
  擴充功能的正式環境程式碼也應避免從 `openclaw/plugin-sdk/<other-plugin>`
  匯入。如果輔助程式確實為共用，應將其提升至中立的 SDK 子路徑，
  例如 `openclaw/plugin-sdk/speech`、`.../provider-model-shared` 或其他
  以功能為導向的介面，而非將兩個外掛耦合在一起。
</Warning>

## 相關內容

<CardGroup cols={2}>
  <Card title="進入點" icon="door-open" href="/zh-TW/plugins/sdk-entrypoints">
    `definePluginEntry` 和 `defineChannelPluginEntry` 選項。
  </Card>
  <Card title="執行階段輔助程式" icon="gears" href="/zh-TW/plugins/sdk-runtime">
    完整的 `api.runtime` 命名空間參考資料。
  </Card>
  <Card title="初始設定與設定" icon="sliders" href="/zh-TW/plugins/sdk-setup">
    封裝、資訊清單與設定結構描述。
  </Card>
  <Card title="測試" icon="vial" href="/zh-TW/plugins/sdk-testing">
    測試公用程式與 lint 規則。
  </Card>
  <Card title="SDK 遷移" icon="arrows-turn-right" href="/zh-TW/plugins/sdk-migration">
    從已棄用的介面遷移。
  </Card>
  <Card title="外掛內部架構" icon="diagram-project" href="/zh-TW/plugins/architecture">
    深入解析架構與功能模型。
  </Card>
</CardGroup>
