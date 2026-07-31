---
read_when:
    - 實作提供者執行階段鉤子、頻道生命週期或套件組合
    - 偵錯外掛載入順序或登錄檔狀態
    - 新增外掛功能或上下文引擎外掛
summary: 外掛架構內部機制：載入流水線、登錄檔、執行階段鉤子、HTTP 路由與參考表格
title: 外掛架構內部機制
x-i18n:
    generated_at: "2026-07-26T08:39:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 278ac23a9454ab69407c59fa197e75756fa0dc5880fcae6c3eecc15bd4733a09
    source_path: plugins/architecture-internals.md
    workflow: 16
---

關於公開能力模型、外掛形態，以及擁有權／執行合約，請參閱[外掛架構](/zh-TW/plugins/architecture)。本頁涵蓋內部機制：載入流水線、登錄檔、執行階段掛鉤、閘道 HTTP 路由、匯入路徑與結構描述資料表。

## 載入流水線

啟動時，OpenClaw 大致會執行以下動作：

1. 探索候選外掛根目錄
2. 讀取原生或相容的套件組合資訊清單與套件中繼資料
3. 拒絕不安全的候選項目
4. 正規化外掛設定（`plugins.enabled`、`allow`、`deny`、`entries`、
   `slots`、`load.paths`）
5. 決定每個候選項目是否啟用
6. 載入已啟用的原生模組：建置完成的內建模組使用原生載入器；
   第三方本機原始碼 TypeScript 使用緊急備援 Jiti
7. 呼叫原生 `register(api)` 掛鉤，並將註冊項目收集至外掛登錄檔
8. 向命令／執行階段介面公開登錄檔

安全閘門會在執行階段執行**之前**運作。下列情況會使探索程序封鎖候選項目：

- 其解析後的進入點逸出外掛根目錄
- 其路徑（或其根目錄）允許所有人寫入
- 對於非內建外掛，路徑擁有者與目前的 uid（或 root）不符

對於允許所有人寫入的內建目錄，閘門重新檢查之前會先嘗試就地執行 `chmod` 修復（npm／全域安裝可能會使套件目錄處於 `0777`）；對內建來源則完全略過擁有權檢查。

若已知遭封鎖候選項目的外掛 id，所發出的診斷仍會包含該 id（包括從原本會遭拒絕的目錄內資訊清單解析出的 id），因此參照該 id 的設定會看到與路徑安全警告相關聯的遭封鎖外掛，而非無關的「未知外掛」錯誤。

### 資訊清單優先行為

資訊清單是控制平面的事實來源。OpenClaw 使用它來：

- 識別外掛
- 探索宣告的頻道／Skills／設定結構描述或套件組合能力
- 驗證 `plugins.entries.<id>.config`
- 補充控制介面的標籤／預留位置文字
- 顯示安裝／目錄中繼資料
- 保留低成本的啟用與設定描述元，而不載入外掛執行階段

對於原生外掛，執行階段模組是資料平面的部分。它會註冊掛鉤、工具、命令或供應商流程等實際行為。

選用的資訊清單 `activation` 與 `setup` 區塊會留在控制平面。它們是僅含中繼資料的描述元，用於啟用規劃與設定探索；不會取代執行階段註冊、`register(...)` 或 `setupEntry`。即時啟用取用者會使用資訊清單中的命令、頻道與供應商提示，在進行更廣泛的登錄檔具現化之前縮小外掛載入範圍：

- 命令列介面載入會將範圍縮小至擁有所要求主要命令的外掛
- 頻道設定／外掛解析會將範圍縮小至擁有所要求
  頻道 id 的外掛
- 明確的供應商設定／執行階段解析會將範圍縮小至擁有所要求
  供應商 id 的外掛
- 閘道啟動規劃使用 `activation.onStartup` 進行明確的啟動
  匯入；缺少啟動中繼資料的外掛只會透過範圍更窄的
  啟用觸發條件載入

啟用規劃器同時公開僅含 id 的 API 供現有呼叫端使用，以及供診斷使用的規劃 API。規劃項目會回報選取外掛的原因，並區分明確的 `activation.*` 提示與資訊清單擁有權備援：

| 原因（來自 `activation.*` 提示）   | 原因（來自資訊清單擁有權）                                                             |
| ------------------------------------ | -------------------------------------------------------------------------------------------- |
| `activation-agent-harness-hint`      | —                                                                                            |
| `activation-capability-hint`         | —                                                                                            |
| `activation-channel-hint`            | `manifest-channel-owner`（`channels`）                                                        |
| `activation-command-hint`            | `manifest-command-alias`（`commandAliases`）                                                  |
| `activation-provider-hint`           | `manifest-provider-owner`（`providers`）、`manifest-setup-provider-owner`（`setup.providers`） |
| `activation-route-hint`              | —                                                                                            |
| —（掛鉤觸發條件沒有提示變體） | `manifest-hook-owner`（`hooks`）、`manifest-tool-contract`（`contracts.tools`）                |

此原因區分即為相容性邊界：現有外掛中繼資料會繼續運作，而新程式碼可偵測廣泛提示或備援行為，且不會改變執行階段載入語意。

要求時間點的執行階段預載若要求廣泛的 `all` 範圍，仍會從設定、啟動規劃、已設定的頻道、插槽與自動啟用規則推導出明確的有效外掛 id 集合（`src/plugins/effective-plugin-ids.ts` 中的 `resolveEffectivePluginIds`）。如果推導出的集合為空，OpenClaw 會維持空範圍，而不會擴大至所有可探索的外掛。

設定探索會優先使用描述元擁有的 id（例如 `setup.providers` 與 `setup.cliBackends`）縮小候選外掛範圍，之後才會針對仍需要設定階段執行階段掛鉤的外掛，備援至 `setup-api`。供應商設定清單會使用資訊清單 `providerAuthChoices`、由描述元衍生的設定選項，以及安裝目錄中繼資料，而不載入供應商執行階段。明確的 `setup.requiresRuntime: false` 是僅限描述元的截止點；省略 `requiresRuntime` 會保留舊版 setup-api 備援以維持相容性。如果有多個探索到的外掛宣告相同的正規化設定供應商或命令列介面後端 id，設定查詢會拒絕這個擁有者不明確的情況，而不會依賴探索順序。設定執行階段實際執行時，登錄檔診斷會回報 `setup.providers`／`setup.cliBackends` 與 setup-api 實際註冊的供應商或命令列介面後端之間的偏差，但不會封鎖舊版外掛。

### 外掛快取邊界

OpenClaw 不會在以實際時間為基礎的時間窗後方快取外掛探索結果或直接的資訊清單登錄檔資料。安裝、資訊清單編輯與載入路徑變更，必須在下一次明確讀取中繼資料或重建快照時顯現。資訊清單檔案剖析器會保留有界的檔案簽章快取，其索引鍵由已開啟的資訊清單路徑，加上裝置／inode、大小及 mtime／ctime 組成；此快取僅用於避免重新剖析未變更的位元組，且不得快取探索、登錄檔、擁有者或原則答案。

安全的中繼資料快速路徑是明確的物件擁有權，而非隱藏快取。閘道啟動的熱路徑應在整條呼叫鏈中傳遞目前的 `PluginMetadataSnapshot`、衍生的 `PluginLookUpTable` 或明確的資訊清單登錄檔。只要這些物件仍代表目前的設定與外掛清單，設定驗證、啟動自動啟用、外掛啟動程序及供應商選取便可重複使用它們。除非特定設定路徑收到明確的資訊清單登錄檔，否則設定查詢仍會視需要重新建構資訊清單中繼資料；應將其保留為冷路徑備援，而非新增隱藏的查詢快取。輸入變更時，應重建並取代快照，而非修改快照或保留歷史副本。作用中外掛登錄檔的檢視，以及內建頻道啟動輔助程式，應從目前的登錄檔／根目錄重新計算。短期對應表可在單次呼叫內用於工作去重或防止重新進入；不得成為行程中繼資料快取。

對外掛載入而言，持久快取層位於執行階段載入。當程式碼或已安裝成品確實載入時，它可重複使用載入器狀態，例如：

- `PluginLoaderCacheState` 與相容的作用中執行階段登錄檔
- 用於避免重複匯入相同執行階段介面的 jiti／模組快取與公開介面載入器快取
- 已安裝外掛成品的檔案系統快取
- 用於路徑正規化或重複項目解析的短期單次呼叫對應表

這些快取是資料平面的實作細節。除非呼叫端刻意要求載入執行階段，否則不得用它們回答控制平面問題，例如「哪個外掛擁有這個供應商？」

請勿為下列項目新增持久或以實際時間為基礎的快取：

- 探索結果
- 直接的資訊清單登錄檔
- 從已安裝外掛索引重新建構的資訊清單登錄檔
- 供應商擁有者查詢、模型抑制、供應商原則或公開成品
  中繼資料
- 任何其他衍生自資訊清單的答案，而變更後的資訊清單、已安裝索引
  或載入路徑應在下一次讀取中繼資料時顯現

從持久化的已安裝外掛索引重建資訊清單中繼資料的呼叫端，會視需要重新建構該登錄檔。已安裝索引是持久的來源平面狀態；它不是隱藏的行程內中繼資料快取。

## 登錄檔模型

載入的外掛不會直接修改任意的核心全域項目。它們會註冊至中央外掛登錄檔（`src/plugins/registry-types.ts` 中的 `PluginRegistry`），此登錄檔會追蹤外掛記錄（身分、來源、來源類型、狀態、診斷），以及各種能力的陣列：工具、舊版掛鉤與型別化掛鉤、頻道、供應商、閘道 RPC 處理常式、HTTP 路由、命令列介面註冊器、背景服務、外掛擁有的命令，以及數十種其他型別化供應商系列（語音、嵌入、影像／影片／音樂生成、網頁擷取／搜尋、代理程式框架、工作階段動作等）。

接著，核心功能會從該登錄檔讀取，而非直接與外掛模組通訊。這會使載入維持單向：

- 外掛模組 -> 登錄檔註冊
- 核心執行階段 -> 登錄檔取用

這項分離對可維護性很重要。這表示大多數核心介面只需要一個整合點：「讀取登錄檔」，而非「為每個外掛模組編寫特殊處理」。

## 對話繫結回呼

繫結對話的外掛可在核准作業獲得解決時做出反應。

使用 `api.onConversationBindingResolved(...)`，在繫結要求獲核准或拒絕後接收回呼：

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // 此外掛與對話之間現在已有繫結。
        console.log(event.binding?.conversationId);
        return;
      }

      // 要求遭到拒絕；清除任何本機待處理狀態。
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

回呼承載資料欄位：

- `status`：`"approved"` 或 `"denied"`
- `decision`：`"allow-once"`、`"allow-always"` 或 `"deny"`
- `binding`：已核准要求的已解析繫結
- `request`：原始要求摘要、解除附加提示、傳送者 id，以及
  對話中繼資料

此回呼僅用於通知。它不會變更誰有權繫結對話，且會在核心核准處理完成後執行。

## 供應商執行階段掛鉤

供應商外掛分為三層：

- 用於低成本執行階段前查詢的**資訊清單中繼資料**：
  `setup.providers[].envVars`、`providerAuthAliases`、`providerAuthChoices`
  與 `channelConfigs`。
- **設定階段掛鉤**：`catalog` 加上 `applyConfigDefaults`。
- **執行階段掛鉤**：40+ 個選用掛鉤，涵蓋驗證、模型解析、
  串流包裝、思考層級、重播原則及用量端點。請參閱
  [掛鉤順序與用量](#hook-order-and-usage)。

OpenClaw 仍負責通用代理程式迴圈、容錯移轉、對話記錄處理，以及
工具政策。這些掛鉤是供應商特定行為的擴充介面，
無需使用完整的自訂推論傳輸。

當供應商具有以環境變數為基礎的認證資訊，且通用驗證／狀態／模型選擇器路徑應在不
載入外掛執行階段的情況下看到這些資訊時，請使用資訊清單 `setup.providers[].envVars`。
當某個供應商 ID 應重複使用另一個供應商 ID 的環境變數、驗證設定檔、
以設定為基礎的驗證，以及 API 金鑰初始設定選項時，請使用資訊清單 `providerAuthAliases`。
當初始設定／驗證選項的命令列介面需要在不載入
供應商執行階段的情況下，得知供應商的選項 ID、群組標籤，以及簡單的單一旗標驗證配線時，
請使用資訊清單 `providerAuthChoices`。供應商執行階段
`envVars` 應保留用於面向操作人員的提示，例如初始設定標籤或 OAuth
用戶端 ID／用戶端密鑰設定變數。

透過所屬的 `channelConfigs.<id>.schema` 和設定描述項，
描述由環境變數驅動的頻道設定與驗證。

### 掛鉤順序與用法

對於模型／供應商外掛，OpenClaw 會大致依照以下順序呼叫掛鉤。
「使用時機」欄是快速決策指南。
OpenClaw 已不再呼叫、僅供相容性使用的供應商欄位，例如
`ProviderPlugin.capabilities` 和 `suppressBuiltInModel`，刻意未列於此處。

| 掛鉤                              | 功能                                                                                                   | 使用時機                                                                                                                                   |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `catalog`                         | 在產生 `models.json` 期間，將供應商設定發布至 `models.providers`                                | 供應商擁有目錄或基礎 URL 預設值                                                                                                  |
| `applyConfigDefaults`             | 在設定具體化期間套用供應商所屬的全域設定預設值                                      | 預設值取決於驗證模式、環境或供應商模型系列的語意                                                                         |
| _(內建模型查詢)_         | OpenClaw 會先嘗試一般的登錄檔／目錄路徑                                                          | _(不是外掛掛鉤)_                                                                                                                         |
| `normalizeModelId`                | 在查詢前正規化舊版或預覽版模型 ID 別名                                                     | 供應商負責在解析標準模型前清理別名                                                                                 |
| `normalizeTransport`              | 在通用模型組合前正規化供應商系列的 `api` / `baseUrl`                                      | 供應商負責清理同一傳輸系列中自訂供應商 ID 的傳輸設定                                                          |
| `normalizeConfig`                 | 在執行階段／供應商解析前正規化 `models.providers.<id>`                                           | 供應商需要應歸屬於外掛的設定清理；內建的 Google 系列輔助工具也為支援的 Google 設定項目提供後援   |
| `applyNativeStreamingUsageCompat` | 對設定供應商套用原生串流用量相容性重寫                                               | 供應商需要由端點驅動的原生串流用量中繼資料修正                                                                          |
| `resolveConfigApiKey`             | 在載入執行階段驗證前，解析設定供應商的環境標記驗證                                       | 供應商公開其自有的環境標記 API 金鑰解析掛鉤                                                                                |
| `resolveSyntheticAuth`            | 公開本機／自託管或由設定支援的驗證，且不保存明文                                   | 供應商可使用合成／本機認證資訊標記運作                                                                                 |
| `resolveExternalAuthProfiles`     | 疊加供應商所屬的外部驗證設定檔；對於命令列介面／應用程式所屬的認證資訊，預設 `persistence` 為 `runtime-only` | 供應商重複使用外部驗證認證資訊，而不保存複製的重新整理權杖；請在資訊清單中宣告 `contracts.externalAuthProviders` |
| `shouldDeferSyntheticProfileAuth` | 降低由環境／設定支援之驗證背後已儲存的合成設定檔預留位置優先權                                      | 供應商儲存不應取得優先權的合成預留位置設定檔                                                                 |
| `resolveDynamicModel`             | 對本機登錄檔尚未收錄且由供應商所屬的模型 ID 進行同步後援                                       | 供應商接受任意上游模型 ID                                                                                                 |
| `prepareDynamicModel`             | 非同步預熱，接著再次執行 `resolveDynamicModel`                                                           | 供應商需要網路中繼資料，才能解析未知 ID                                                                                  |
| `normalizeResolvedModel`          | 內嵌執行器使用已解析模型前的最終重寫                                               | 供應商需要傳輸重寫，但仍使用核心傳輸                                                                             |
| `normalizeToolSchemas`            | 在內嵌執行器看到工具結構描述前進行正規化                                                    | 供應商需要傳輸系列的結構描述清理                                                                                                |
| `inspectToolSchemas`              | 在正規化後公開供應商所屬的結構描述診斷                                                  | 供應商需要關鍵字警告，但不希望核心理解供應商特定規則                                                                 |
| `resolveReasoningOutputMode`      | 選取原生或標記式推理輸出合約                                                              | 供應商需要標記式推理／最終輸出，而非原生欄位                                                                         |
| `prepareExtraParams`              | 在通用串流選項包裝函式前進行請求參數正規化                                              | 供應商需要預設請求參數或各供應商的參數清理                                                                           |
| `createStreamFn`                  | 使用自訂傳輸完全取代一般串流路徑                                                   | 供應商需要自訂有線通訊協定，而不只是包裝函式                                                                                     |
| `wrapStreamFn`                    | 套用通用包裝函式後的串流包裝函式                                                              | 供應商需要請求標頭／本文／模型相容性包裝函式，但不需要自訂傳輸                                                          |
| `resolveTransportTurnState`       | 附加每回合的原生傳輸標頭或中繼資料                                                           | 供應商希望通用傳輸傳送供應商原生的回合識別資訊                                                                       |
| `resolveWebSocketSessionPolicy`   | 附加原生 WebSocket 標頭或工作階段冷卻政策                                                    | 供應商希望通用 WS 傳輸調整工作階段標頭或後援政策                                                               |
| `formatApiKey`                    | 驗證設定檔格式器：已儲存的設定檔會成為執行階段的 `apiKey` 字串                                     | 供應商儲存額外驗證中繼資料，且需要自訂的執行階段權杖格式                                                                    |
| `refreshOAuth`                    | 自訂重新整理端點或重新整理失敗政策的 OAuth 重新整理覆寫                                  | 供應商不適用共用的 OpenClaw 重新整理器                                                                                          |
| `buildAuthDoctorHint`             | OAuth 重新整理失敗時附加的修復提示                                                                  | 供應商需要在重新整理失敗後提供由供應商所屬的驗證修復指引                                                                      |
| `matchesContextOverflowError`     | 供應商所屬的上下文視窗溢位比對器                                                                 | 供應商具有通用啟發式方法會漏掉的原始溢位錯誤                                                                                |
| `classifyFailoverReason`          | 供應商所屬的容錯移轉原因分類                                                                  | 供應商可將原始 API／傳輸錯誤對應至速率限制／過載等類別                                                                          |
| `isCacheTtlEligible`              | Proxy／回程供應商的提示快取政策                                                               | 供應商需要 Proxy 特定的快取 TTL 限制                                                                                                |
| `buildMissingAuthMessage`         | 取代通用的缺少驗證復原訊息                                                      | 供應商需要供應商特定的缺少驗證復原提示                                                                                 |
| `augmentModelCatalog`             | 探索後附加的合成／最終目錄資料列（已淘汰，請參閱下文）                                  | 供應商需要在 `models list` 和選擇器中加入合成的向前相容資料列                                                                     |
| `resolveThinkingProfile`          | 模型特定的 `/think` 等級集合、顯示標籤和預設值                                                 | 供應商為選定模型公開自訂的思考層級或二元標籤                                                                 |
| `isBinaryThinking`                | 開啟／關閉推理切換相容性掛鉤                                                                     | 供應商僅公開二元的思考開啟／關閉功能                                                                                                  |
| `supportsXHighThinking`           | `xhigh` 推理支援相容性掛鉤                                                                   | 供應商只希望在部分模型上使用 `xhigh`                                                                                             |
| `resolveDefaultThinkingLevel`     | 預設 `/think` 等級相容性掛鉤                                                                      | 供應商負責某個模型系列的預設 `/think` 政策                                                                                      |
| `isModernModelRef`                | 用於即時設定檔篩選和冒煙測試選取的現代模型比對器                                              | 供應商負責即時／冒煙測試的偏好模型比對                                                                                             |
| `prepareRuntimeAuth`              | 在推論前一刻，將已設定的認證資訊交換為實際的執行階段權杖／金鑰                       | 供應商需要權杖交換或短效請求認證資訊                                                                             |
| `resolveUsageAuth`                | 解析 `/usage` 和相關狀態介面的用量／計費認證資訊                                     | 供應商需要自訂用量／配額權杖剖析，或不同的用量認證資訊                                                               |
| `fetchUsageSnapshot`              | 驗證解析後，擷取並正規化供應商特定的用量／配額快照                             | 供應商需要供應商特定的用量端點或承載資料剖析器                                                                           |
| `createEmbeddingProvider`         | 為記憶／搜尋建置由提供者擁有的嵌入配接器                                                     | 記憶嵌入行為屬於提供者外掛                                                                                    |
| `buildReplayPolicy`               | 傳回控制提供者對逐字稿處理方式的重播原則                                        | 提供者需要自訂逐字稿原則（例如移除思考區塊）                                                               |
| `sanitizeReplayHistory`           | 在一般逐字稿清理後重寫重播歷程                                                        | 除了共用壓縮輔助工具外，提供者還需要提供者專屬的重播重寫                                                             |
| `validateReplayTurns`             | 在嵌入式執行器之前進行最終重播輪次驗證或重塑                                           | 提供者傳輸在一般清理後需要更嚴格的輪次驗證                                                                    |
| `onModelSelected`                 | 執行由提供者擁有的選擇後副作用                                                                 | 模型啟用時，提供者需要遙測或由提供者擁有的狀態                                                                  |

`normalizeModelId`、`normalizeTransport` 和 `normalizeConfig` 會先檢查
相符的供應商外掛，接著依序嘗試其他支援鉤子的供應商外掛，
直到其中一個實際變更模型 ID 或傳輸方式／設定。如此可讓
別名／相容性供應商墊片繼續運作，而不要求呼叫端知道由哪個
內建外掛負責改寫。若沒有供應商鉤子改寫支援的
Google 系列設定項目，內建的 Google 設定正規化器仍會套用
該相容性清理。

若供應商需要完全自訂的線路通訊協定或自訂要求執行器，
那屬於不同類型的擴充。這些鉤子適用於仍在 OpenClaw
一般推論迴圈中執行的供應商行為。

`resolveUsageAuth` 會決定 OpenClaw 應呼叫 `fetchUsageSnapshot`，
或針對用量／狀態介面回退至通用認證資訊解析。當供應商
具有用量認證資訊時，回傳 `{ token, accountId?, subscriptionType?, rateLimitTier? }`
（選用的方案中繼資料會傳入
`fetchUsageSnapshot`）；當供應商自行管理的用量驗證已處理要求，
且必須抑制通用 API 金鑰／OAuth 回退時，回傳
`{ handled: true }`；當供應商未處理用量驗證時，則回傳
`null` 或 `undefined`。

在資訊清單的 `providerUsageAuthEnvVars` 中宣告組織或帳務認證資訊。
這可讓通用探索和機密資訊清除介面辨識它們，而不會將其列為
推論驗證的候選項目。

### 供應商範例

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### 內建範例

內建供應商外掛會組合上述鉤子，以符合各廠商的目錄、
驗證、思考、重播和用量需求。具權威性的鉤子集合位於
`extensions/` 下的各個外掛中；本頁旨在說明其形式，而非
複製該清單。

<AccordionGroup>
  <Accordion title="直通式目錄供應商">
    OpenRouter、Kilocode、Z.AI、xAI 會註冊 `catalog`，
    並搭配 `resolveDynamicModel`／`prepareDynamicModel`，以便在
    OpenClaw 的靜態目錄之前呈現上游模型 ID。
  </Accordion>
  <Accordion title="OAuth 與用量端點供應商">
    GitHub Copilot、Gemini CLI、ChatGPT Codex、MiniMax、Xiaomi、z.ai
    會將 `prepareRuntimeAuth` 或 `formatApiKey` 與
    `resolveUsageAuth` + `fetchUsageSnapshot` 配對，以自行管理權杖交換及
    `/usage` 整合。
  </Accordion>
  <Accordion title="重播與逐字稿清理系列">
    共用的具名系列（`google-gemini`、`passthrough-gemini`、
    `anthropic-by-model`、`hybrid-anthropic-openai`）可讓供應商透過
    `buildReplayPolicy` 選擇採用逐字稿原則，而無須讓每個外掛
    各自重新實作清理作業。
  </Accordion>
  <Accordion title="僅目錄供應商">
    `byteplus`、`cloudflare-ai-gateway`、`huggingface`、`kimi-coding`、`nvidia`、
    `qianfan`、`synthetic`、`together`、`venice`、`vercel-ai-gateway` 和
    `volcengine` 僅註冊 `catalog`，並沿用共用推論迴圈。
  </Accordion>
  <Accordion title="Anthropic 專用串流輔助工具">
    Beta 標頭、`/fast`／`serviceTier` 和 `context1m`
    位於 Anthropic 外掛的公開 `api.ts`／`contract-api.ts`
    接合面（`wrapAnthropicProviderStream`、`resolveAnthropicBetas`、
    `resolveAnthropicFastMode`、`resolveAnthropicServiceTier`）內，而非
    通用 SDK 中。
  </Accordion>
</AccordionGroup>

## 執行階段輔助工具

外掛可透過 `api.runtime` 存取選定的核心輔助工具。以 TTS 為例：

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

注意事項：

- `textToSpeech` 會針對檔案／語音訊息介面回傳一般的核心 TTS 輸出承載資料。
- 使用核心 `tts` 設定和供應商選擇。
- 回傳 PCM 音訊緩衝區與取樣率。外掛必須針對供應商重新取樣／編碼。
- 每個供應商可選擇是否提供 `listVoices`。請將其用於廠商自行管理的語音選擇器或設定流程。
- 核心會將解析後的要求截止時間傳給供應商的 `listVoices` 鉤子；供應商專屬的逾時設定可能會覆寫它。
- 語音清單可包含更豐富的中繼資料，例如語言區域、性別和個性標籤，以供支援供應商感知的選擇器使用。
- 目前 OpenAI 和 ElevenLabs 支援電話語音。Microsoft 不支援。

外掛也可透過 `api.registerSpeechProvider(...)` 註冊語音供應商。

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

注意事項：

- 將 TTS 原則、回退和回覆傳送保留在核心中。
- 使用語音供應商實作由廠商自行管理的語音合成行為。
- 舊版 Microsoft `edge` 輸入會正規化為 `microsoft` 供應商 ID。
- 偏好的權責模型以公司為導向：隨著 OpenClaw 新增這些
  能力合約，一個廠商外掛可管理文字、語音、影像及未來的媒體供應商。

對於影像／音訊／影片理解，外掛會註冊一個具型別的
媒體理解供應商，而非通用的鍵／值集合：

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

注意事項：

- 將協調、回退、設定和頻道接線保留在核心中。
- 將廠商行為保留在供應商外掛中。
- 新增式擴充應維持具型別：新增選用方法、新增選用
  結果欄位、新增選用能力。
- 影片生成已遵循相同模式：
  - 核心管理能力合約和執行階段輔助工具
  - 廠商外掛註冊 `api.registerVideoGenerationProvider(...)`
  - 功能／頻道外掛使用 `api.runtime.videoGeneration.*`

針對媒體理解執行階段輔助工具，外掛可呼叫：

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});

const extraction = await api.runtime.mediaUnderstanding.extractStructuredWithModel({
  provider: "codex",
  model: "gpt-5.6-sol",
  input: [
    {
      type: "image",
      buffer: receiptImageBuffer,
      fileName: "receipt.png",
      mime: "image/png",
    },
    { type: "text", text: "Use the printed fields as the source of truth." },
  ],
  instructions: "Return entities and searchable tags.",
  schemaName: "example.evidence",
  jsonSchema: {
    type: "object",
    properties: {
      entities: { type: "array", items: { type: "string" } },
      tags: { type: "array", items: { type: "string" } },
    },
  },
  cfg: api.config,
});
```

針對音訊轉錄，外掛可使用媒體理解執行階段
或較舊的 STT 別名：

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

注意事項：

- `api.runtime.mediaUnderstanding.*` 是影像／音訊／影片理解的首選共用介面。
- `extractStructuredWithModel(...)` 是供外掛使用的接合面，用於有界限且
  由供應商管理、以影像為優先的擷取。至少包含一個影像輸入；
  文字輸入為補充情境。產品外掛管理其路由和結構描述，
  而 OpenClaw 管理供應商／執行階段邊界。
- 使用核心媒體理解音訊設定（`tools.media.audio`）和供應商回退順序。
- 未產生任何轉錄輸出時（例如略過／不支援的輸入），回傳 `{ text: undefined }`。

外掛也可透過 `api.runtime.subagent` 啟動背景子代理程式執行：

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  toolsAlsoAllow: ["my_plugin_progress"],
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

注意事項：

- `provider` 和 `model` 是每次執行可選用的覆寫，而非持久性工作階段變更。
- `toolsAlsoAllow` 接受由呼叫外掛註冊、名稱完全相符且權責歸屬唯一的工具。核心工具及名稱有歧義的工具會遭拒。此設定會附加至一般設定檔，但操作員的允許清單和拒絕規則仍具有最終決定權。
- OpenClaw 僅對受信任的呼叫端採用這些覆寫欄位。
- 對於外掛自行管理的回退執行，操作員必須透過 `plugins.entries.<id>.subagent.allowModelOverride: true` 選擇啟用。
- 使用 `plugins.entries.<id>.subagent.allowedModels` 將受信任的外掛限制為特定標準 `provider/model` 目標，或使用 `"*"` 明確允許任何目標。
- 不受信任外掛的子代理程式執行仍可運作，但覆寫要求會遭拒，而非無聲地回退。
- 外掛建立的子代理程式工作階段會加上建立者外掛 ID 標籤。回退 `api.runtime.subagent.deleteSession(...)` 只能刪除這些由其擁有的工作階段；刪除任意工作階段仍需要具管理員範圍的閘道要求。

針對網頁搜尋，外掛可使用共用執行階段輔助工具，而無須
深入存取代理程式工具接線：

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw plugin runtime helpers",
    count: 5,
  },
});
```

外掛也可透過
`api.registerWebSearchProvider(...)` 註冊網頁搜尋供應商。

注意事項：

- 將供應商選擇、認證資訊解析和共用要求語意保留在核心中。
- 使用網頁搜尋供應商實作廠商專屬的搜尋傳輸方式。
- `api.runtime.webSearch.*` 是需要搜尋行為、但不想依賴代理程式工具包裝函式的功能／頻道外掛之首選共用介面。

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "友善的龍蝦吉祥物", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`：使用已設定的影像生成提供者鏈生成影像。
- `listProviders(...)`：列出可用的影像生成提供者及其功能。

## 閘道 HTTP 路由

外掛可以使用 `api.registerHttpRoute(...)` 公開 HTTP 端點。

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

路由欄位：

- `path`：閘道 HTTP 伺服器下的路由路徑。
- `auth`：必填，`"gateway"` 或 `"plugin"`。使用 `"gateway"` 要求一般閘道驗證，或使用 `"plugin"` 進行外掛管理的驗證／網路鉤子驗證。
- `match`：選填。`"exact"`（預設）或 `"prefix"`。
- `handleUpgrade`：選填，用於相同路由上 WebSocket 升級要求的處理常式。
- `replaceExisting`：選填。允許相同外掛取代自己現有的路由註冊。
- `handler`：當路由已處理要求時，傳回 `true`。

注意事項：

- `api.registerHttpHandler(...)` 已移除，並會導致外掛載入錯誤。請改用 `api.registerHttpRoute(...)`。
- 外掛路由必須明確宣告 `auth`。
- 除非使用 `replaceExisting: true`，否則完全相同的 `path + match` 衝突會遭拒絕，而且一個外掛不能取代另一個外掛的路由。
- 具有不同 `auth` 層級的重疊路由會遭拒絕。`exact`/`prefix` 後援鏈只能維持在相同的驗證層級。
- `auth: "plugin"` 路由**不會**自動取得操作者執行階段範圍。它們用於外掛管理的網路鉤子／簽章驗證，而非具特殊權限的閘道輔助程式呼叫。
- `auth: "gateway"` 路由會在閘道要求的執行階段範圍內執行。預設介面（`gatewayRuntimeScopeSurface: "write-default"`）刻意採取保守設計：
  - 共用密鑰持有人驗證（`gateway.auth.mode = "token"` / `"password"`）以及任何非受信任 Proxy 的驗證方法，都只會取得單一 `operator.write` 範圍，即使呼叫端傳送 `x-openclaw-scopes` 也是如此
  - 未提供明確 `x-openclaw-scopes` 標頭的 `trusted-proxy` 呼叫端，也會保留舊版僅限 `operator.write` 的介面
  - 有傳送 `x-openclaw-scopes` 的 `trusted-proxy` 呼叫端，則會取得已宣告的範圍
  - 路由可以選擇加入 `gatewayRuntimeScopeSurface: "trusted-operator"`，以便對具備身分資訊的驗證模式一律採用 `x-openclaw-scopes`（缺少該標頭時，則後援至完整的命令列介面預設範圍集合）
- 由 `auth: "gateway"` 路由支援的沙箱化外部 Control UI 分頁，會使用僅由已驗證啟動程序核發的短效期簽署 Cookie 授權；外掛驗證分頁則保留其直接 iframe 路徑。掛載前，父層會在相同的不透明沙箱內執行路由自有的探測，若瀏覽器隱私權政策封鎖該 Cookie，就會採取封閉式失敗。授權會繫結至所屬外掛、相符的路由根目錄及目前的驗證世代；其程序隨機 Cookie 名稱可防止同一主機上受信任的閘道彼此覆寫，但 Cookie 永遠無法隔離 TCP 連接埠。因此，閘道主機名稱是一個認證資訊邊界：請勿在該主機名稱上共同託管彼此不信任的服務，包括其他連接埠上的服務。路由分派會拒絕對另一個外掛所擁有之巢狀路由的重複使用。由於沙箱子項在 Cookie 判定上屬於跨網站，因此該授權只接受具有 `operator.read` 的 `GET` 和 `HEAD`；變更操作和 WebSocket 升級仍須使用明確經閘道驗證的介面。此 Cookie 刻意不使用 CHIPS：目前的瀏覽器會在分割區索引鍵中包含跨網站祖先位元，因此巢狀不透明沙箱框架將無法存取相同路由的資產。此 Cookie 需要安全內容環境以及瀏覽器對跨網站 Cookie 的權限，因此在純 HTTP 區域網路來源或完全封鎖第三方 Cookie 的環境下，無法使用閘道驗證的外部分頁；請使用 HTTPS/Tailscale Serve，或搭配相容 Cookie 政策且受瀏覽器信任的迴路位址。
- 此授權可防止閘道持有人權杖洩漏，以及意外重複使用路由／範圍；但不會在原生外掛之間建立安全邊界。原生外掛程式碼及其提供的 UI 內容，仍屬於相同的受信任程序內外掛邊界。
- 實務規則：不要假設閘道驗證的外掛路由隱含為管理員介面。如果路由需要僅限管理員的行為，請選擇加入 `trusted-operator` 範圍介面、要求具備身分資訊的驗證模式，並記錄明確的 `x-openclaw-scopes` 標頭合約。
- 完成路由比對與驗證後，一般處理常式會參與閘道根工作准入。處於已準備或重新啟動狀態的閘道，會在叫用處理常式前傳回 `503`。唯一的狹義例外，是在資訊清單中具有權限且同時選擇加入路由專用 `trusted-operator` 介面的 `auth: "gateway"` 路由；它會保持可存取，以免暫停控制分派無法執行，而相同外掛的一般同層路由仍會位於准入邊界之後。WebSocket `handleUpgrade` 的所有權使用相同的不可分割准入邊界；處理常式接受通訊端後，該通訊端後續的生命週期由外掛擁有，且不受此邊界追蹤。

## 外掛 SDK 匯入路徑

撰寫新外掛時，請使用範圍較窄的 SDK 子路徑，而非單體式 `openclaw/plugin-sdk` 根
彙總匯出。核心子路徑：

| 子路徑                            | 用途                                      |
| ---------------------------------- | -------------------------------------------- |
| `openclaw/plugin-sdk/plugin-entry` | 外掛註冊基本元件               |
| `openclaw/plugin-sdk/channel-core` | 頻道進入點／建置輔助程式                  |
| `openclaw/plugin-sdk/core`         | 通用共用輔助程式及傘狀合約 |

頻道外掛可從一系列範圍較窄的接合介面中選用，包括 `channel-setup`、
`setup-runtime`、`setup-tools`、`channel-pairing`、
`channel-contract`、`channel-feedback`、`channel-inbound`、`channel-outbound`、
`command-auth`、`secret-input`、`webhook-ingress`、
`channel-targets` 及 `channel-actions`。核准行為應統一
使用單一 `approvalCapability` 合約，而非混用不相關的
外掛欄位。請參閱[頻道外掛](/zh-TW/plugins/sdk-channel-plugins)。

執行階段和設定輔助程式位於相對應、用途集中的 `*-runtime` 子路徑下
（`approval-runtime`、`agent-runtime`、`lazy-runtime`、`directory-runtime`、
`text-runtime`、`runtime-store`、`system-event-runtime`、`heartbeat-runtime`、
`channel-activity-runtime` 等）。請優先使用 `config-contracts`、
`plugin-config-runtime`、`runtime-config-snapshot` 和 `config-mutation`，
而非廣泛的 `config-runtime` 相容性彙總匯出。

<Info>
`openclaw/plugin-sdk/channel-lifecycle`、小型頻道輔助程式外觀介面、
`openclaw/plugin-sdk/config-runtime` 和 `openclaw/plugin-sdk/infra-runtime`
都是供舊版外掛使用、已淘汰的相容性墊片。新程式碼應改為匯入
範圍更窄的通用基本元件。
</Info>

儲存庫內部進入點（依各內建外掛套件根目錄）：

- `index.js` — 內建外掛進入點
- `api.js` — 輔助程式／型別彙總匯出
- `runtime-api.js` — 僅限執行階段的彙總匯出
- `setup-entry.js` — 設定外掛進入點

外部外掛只能匯入 `openclaw/plugin-sdk/*` 子路徑。絕對不要
從核心或另一個外掛匯入其他外掛套件的 `src/*`。
透過外觀介面載入的進入點會優先使用目前的執行階段設定快照（若有），
然後才後援至磁碟上已解析的設定檔。

`image-generation`、`media-understanding`
和 `speech` 等功能專用子路徑之所以存在，是因為內建外掛目前正在使用它們。它們不會
自動成為長期凍結的外部合約；若要依賴這些子路徑，請查閱相關的 SDK
參考頁面。

## 訊息工具結構描述

外掛應擁有頻道專用的 `describeMessageTool(...)` 結構描述
貢獻項目，用於回應、已讀和投票等非訊息基本元件。
共用傳送呈現應使用通用 `MessagePresentation` 合約，
而非提供者原生的按鈕、元件、區塊或卡片欄位。
如需合約、後援規則、提供者對應及外掛作者檢查清單，請參閱
[訊息呈現](/zh-TW/plugins/message-presentation)。

具備傳送能力的外掛會透過訊息功能宣告其可呈現的內容：

- `presentation`：用於語意呈現區塊（`text`、`context`、
  `divider`、`chart`、`table`、`buttons`、`select`）
- `delivery-pin`：用於固定式傳遞要求

核心會決定要以原生方式呈現，還是將其降級為文字。
請勿從通用訊息工具公開提供者原生 UI 的逃生出口。
適用於舊版原生結構描述、已淘汰的 SDK 輔助程式仍會匯出，以支援現有的
第三方外掛，但新外掛不應使用它們。

## 頻道目標解析

頻道外掛應擁有頻道專用的目標語意。請維持共用
輸出主機的通用性，並針對提供者規則使用訊息傳遞轉接器介面：

- `messaging.inferTargetChatType({ to })` 會在目錄查詢前，決定正規化目標
應視為 `direct`、`group` 或 `channel`。
- `messaging.targetResolver.looksLikeId(raw, normalized)` 會告訴核心某個
輸入是否應略過目錄搜尋，直接進入類 ID 解析。
- `messaging.targetResolver.reservedLiterals` 會列出對該提供者而言屬於
頻道／工作階段參照的單獨字詞。解析會先保留已設定的
目錄項目，再拒絕保留字面值，接著在目錄未命中時採取封閉式失敗。
- `messaging.targetResolver.resolveTarget(...)` 是當核心在正規化後或
目錄未命中後需要最終由提供者擁有的解析時，所使用的外掛後援。
- `messaging.resolveOutboundSessionRoute(...)` 會在目標解析完成後，負責建立
提供者專用的工作階段路由。

建議分工：

- 將 `inferTargetChatType` 用於應在搜尋
對等方／群組前完成的類別判斷。
- 將 `looksLikeId` 用於「將此項目視為明確／原生目標 ID」的檢查。
- 將 `resolveTarget` 用於提供者專用的正規化後援，而非
廣泛的目錄搜尋。
- 將聊天 ID、討論串 ID、JID、控制代碼及聊天室
ID 等提供者原生 ID 保留在 `target` 值或提供者專用參數中，而非通用 SDK
欄位。

## 設定支援的目錄

從設定衍生目錄項目的外掛，應將該邏輯保留在
外掛中，並重複使用來自
`openclaw/plugin-sdk/directory-runtime` 的共用輔助程式。

當頻道需要下列由設定支援的對等方／群組時，請使用此方式：

- 由允許清單驅動的私訊對等方
- 已設定的頻道／群組對應
- 帳號範圍的靜態目錄後援

`directory-runtime` 中的共用輔助程式只處理通用操作：

- 查詢篩選
- 套用限制
- 去除重複／正規化輔助程式
- 建置 `ChannelDirectoryEntry[]`

頻道專用的帳號檢查和 ID 正規化應保留在
外掛實作中。

## 提供者目錄

提供者外掛可以使用 `registerProvider({ catalog: { run(...) { ... } } })`
定義用於推論的模型目錄。

`catalog.run(...)` 會傳回與 OpenClaw 寫入
`models.providers` 相同的結構：

- `{ provider }` 用於單一供應商項目
- `{ providers }` 用於多個供應商項目

當外掛擁有供應商特定的模型 ID、基礎 URL
預設值或受驗證限制的模型中繼資料時，請使用 `catalog`。

`catalog.order` 控制外掛目錄相對於 OpenClaw
內建隱含供應商的合併時機：

- `simple`：使用一般 API 金鑰或由環境驅動的供應商
- `profile`：存在驗證設定檔時才會出現的供應商
- `paired`：會合成多個相關供應商項目的供應商
- `late`：在其他隱含供應商之後進行的最後一輪

發生鍵衝突時，後面的供應商會勝出，因此外掛可刻意使用相同的供應商 ID
覆寫內建供應商項目。

外掛也可以透過
`api.registerModelCatalogProvider({ provider, kinds, staticCatalog, liveCatalog
})` 發布唯讀模型資料列。這是清單／說明／選擇器介面的未來標準路徑，並支援
`text`、`voice`、`image_generation`、`video_generation` 和 `music_generation`
資料列。供應商外掛仍負責即時端點呼叫、權杖交換及
廠商回應對應；核心則負責共用資料列形狀、來源標籤及
媒體工具說明格式。媒體生成供應商註冊會自動根據
`defaultModel`、`models` 和
`capabilities` 合成靜態目錄資料列。

相容性：

- `discovery` 仍可作為舊版別名使用，但會發出棄用警告
- 如果同時註冊 `catalog` 和 `discovery`，OpenClaw 會使用 `catalog`
  並發出警告
- `augmentModelCatalog` 已棄用；隨附供應商應透過
  `registerModelCatalogProvider` 發布補充資料列

## 唯讀頻道檢查

如果你的外掛註冊頻道，建議在實作
`resolveAccount(...)` 的同時實作 `plugin.config.inspectAccount(cfg, accountId)`。

原因：

- `resolveAccount(...)` 是執行階段路徑。它可以假設認證資訊
  已完全具體化，並可在缺少必要祕密時快速失敗。
- 唯讀命令路徑（例如 `openclaw status`、`openclaw status --all`、
  `openclaw channels status`、`openclaw channels resolve`）及 doctor／設定
  修復流程不應只為了描述設定，就必須具體化執行階段認證資訊。

建議的 `inspectAccount(...)` 行為：

- 只傳回描述性的帳號狀態。
- 保留 `enabled` 和 `configured`。
- 在相關情況下納入認證資訊來源／狀態欄位，例如：
  - `tokenSource`、`tokenStatus`
  - `botTokenSource`、`botTokenStatus`
  - `appTokenSource`、`appTokenStatus`
  - `signingSecretSource`、`signingSecretStatus`
- 你不需要只為了回報唯讀可用性而傳回原始權杖值。
  對狀態類命令而言，傳回 `tokenStatus: "available"`（以及相符的來源
  欄位）就足夠了。
- 當認證資訊透過 SecretRef 設定，但在目前命令路徑中
  無法使用時，請使用 `configured_unavailable`。

如此一來，唯讀命令便可回報「已設定，但在此命令路徑中無法使用」，
而不會當機或將帳號錯誤回報為未設定。

## 套件組合包

外掛目錄可包含具有 `openclaw.extensions` 的 `package.json`：

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

每個項目都會成為一個外掛。如果組合包列出多個擴充功能，外掛
ID 會變成 `<manifestOrPackageName>/<fileBase>`（若有資訊清單 ID，則以其為準；
否則使用未限定範圍的 `package.json` 名稱）。

如果你的外掛匯入 npm 相依套件，請將它們安裝在該目錄中，以便
使用 `node_modules`（`npm install`／`pnpm install`）。

安全護欄：每個 `openclaw.extensions` 項目在符號連結解析後，都必須位於外掛
目錄內。超出套件目錄的項目會遭到拒絕。

安全性注意事項：`openclaw plugins install` 會使用專案本機的
`npm install --omit=dev --ignore-scripts` 安裝外掛相依套件（不執行生命週期指令碼，
執行階段不安裝開發相依套件），並忽略繼承的全域 npm 安裝設定。
請保持外掛相依套件樹為「純 JS／TS」，並避免使用需要
`postinstall` 建置的套件。

選用：`openclaw.setupEntry` 可指向僅供設定使用的輕量模組。
當 OpenClaw 需要已停用頻道外掛的設定介面，
或頻道外掛已啟用但仍未設定時，它會載入 `setupEntry`，
而非完整的外掛進入點。若你的主要外掛進入點還會連接工具、掛鉤
或其他僅供執行階段使用的程式碼，這可減輕啟動和設定負擔。

選用：`openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
可讓頻道外掛選擇在閘道的監聽前啟動階段使用相同的 `setupEntry`
路徑，即使頻道已設定亦然。

只有在 `setupEntry` 完整涵蓋閘道開始監聽前必須存在的啟動介面時，
才使用此選項。實際上，這表示設定進入點必須註冊啟動所依賴的每項
頻道自有功能，例如：

- 頻道註冊本身
- 閘道開始監聽前必須可用的任何 HTTP 路由
- 在同一時間範圍內必須存在的任何閘道方法、工具或服務

如果完整進入點仍擁有任何必要的啟動功能，請勿啟用
此旗標。讓外掛維持預設行為，並由 OpenClaw 在
啟動期間載入完整進入點。

隨附頻道也可以發布僅供設定使用的契約介面輔助程式，讓核心可在完整
頻道執行階段載入前查詢。目前的設定提升介面為：

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

當核心需要在不載入完整外掛進入點的情況下，將舊版單一帳號頻道
設定提升至 `channels.<id>.accounts.*` 時，便會使用該介面。
Matrix 是目前的隨附範例：當具名帳號已存在時，它只會將驗證／啟動程序金鑰
移至具名的提升帳號中，並且可保留已設定的非標準預設帳號鍵，而不一定
一律建立 `accounts.default`。

這些設定修補配接器會讓隨附契約介面的探索保持延遲載入。匯入
時間維持輕量；提升介面只會在首次使用時載入，而不會在
模組匯入時重新進入隨附頻道啟動流程。

當這些啟動介面包含閘道 RPC 方法時，請將其置於
外掛專用前綴下。核心管理命名空間（`config.*`、
`exec.approvals.*`、`wizard.*`、`update.*`）仍為保留項目，而且一律解析為
`operator.admin`，即使外掛要求較窄的範圍亦然。

範例：

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### 頻道目錄中繼資料

頻道外掛可透過 `openclaw.channel` 公告設定／探索中繼資料，並透過
`openclaw.install` 提供安裝提示。這可讓核心目錄不含資料。

範例：

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (自架)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "透過 Nextcloud Talk 網路鉤子機器人提供自架聊天。",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

除了最小範例外，實用的 `openclaw.channel` 欄位包括：

- `detailLabel`：供較豐富目錄／狀態介面使用的次要標籤
- `docsLabel`：覆寫文件連結文字
- `preferOver`：此目錄項目應優先於其上的較低優先級外掛／頻道 ID
- `selectionDocsPrefix`、`selectionDocsOmitLabel`、`selectionExtras`：選擇介面的文案控制項
- `markdownCapable`：將頻道標示為支援 Markdown，以供輸出格式決策使用
- `exposure.configured`：設為 `false` 時，在已設定頻道清單介面中隱藏該頻道
- `exposure.setup`：設為 `false` 時，在互動式設定／配置選擇器中隱藏該頻道
- `exposure.docs`：將頻道標示為內部／私人，以供文件導覽介面使用
- `quickstartAllowFrom`：讓頻道採用標準快速入門 `allowFrom` 流程
- `forceAccountBinding`：即使只有一個帳號，也要求明確繫結帳號
- `preferSessionLookupForAnnounceTarget`：解析公告目標時，優先使用工作階段查詢

OpenClaw 也可以合併**外部頻道目錄**（例如 MPM
登錄匯出）。將 JSON 檔案放在下列其中一處：

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

或者將 `OPENCLAW_PLUGIN_CATALOG_PATHS`（或 `OPENCLAW_MPM_CATALOG_PATHS`）指向
一或多個 JSON 檔案（以逗號／分號／`PATH` 分隔）。每個檔案應
包含 `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`。剖析器也接受 `"packages"` 或 `"plugins"` 作為 `"entries"` 鍵的舊版別名。

產生的頻道目錄項目和供應商安裝目錄項目，會在原始 `openclaw.install`
區塊旁公開正規化的安裝來源資訊。正規化資訊會識別 npm 規格是確切版本
還是浮動選擇器、是否存在預期的完整性中繼資料，以及是否也有本機
來源路徑可用。若目錄／套件身分已知，且剖析出的 npm 套件名稱
偏離該身分，正規化資訊會發出警告。
當 `defaultChoice` 無效或指向不可用的來源，以及存在 npm 完整性
中繼資料但沒有有效的 npm 來源時，它們也會發出警告。使用者應將
`installSource` 視為增補的選用欄位，因此手動建立的項目和目錄轉接層
無須合成該欄位。
如此一來，初始設定和診斷便可在不匯入外掛執行階段的情況下，
說明來源層狀態。

官方外部 npm 項目應優先使用確切的 `npmSpec` 加上
`expectedIntegrity`。為了相容性，單純套件名稱和 dist-tag 仍可運作，
但會顯示來源層警告，讓目錄可逐步轉向固定版本且經完整性檢查的安裝，
而不致破壞現有外掛。
當初始設定從本機目錄路徑安裝時，它會記錄一個受管理的外掛
外掛索引項目，包含 `source: "path"`，並在可行時使用工作區相對的
`sourcePath`。絕對的操作載入路徑仍保留在
`plugins.load.paths`；安裝記錄可避免將本機工作站
路徑重複寫入長期設定中。這讓來源層診斷仍可看見本機開發安裝，
而無須新增第二個原始檔案系統路徑揭露
介面。持久化的 `installed_plugin_index` SQLite 資料表是安裝
來源的唯一真實來源，並可在不載入外掛執行階段模組的情況下重新整理。
即使外掛資訊清單遺失或無效，其 `installRecords` 對應仍會持久保存；
其 `plugins` 承載內容則是可重建的資訊清單檢視。

## 上下文引擎外掛

上下文引擎外掛負責擷取、組裝與壓縮的工作階段上下文協調。
請在外掛中使用 `api.registerContextEngine(id, factory)` 註冊，
再以 `plugins.slots.contextEngine` 選取作用中的引擎。

當你的外掛需要取代或擴充預設上下文
流水線，而不只是新增記憶搜尋或掛鉤時，請使用此功能。

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("lossless-claw", (ctx) => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, sessionKey, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

工廠函式 `ctx` 公開選用的 `config`、`agentDir` 和 `workspaceDir`
值，以供建構階段初始化。

主機會先完成已註冊的非同步記憶提示準備，再呼叫非舊版引擎的
`assemble()`。當 `assemble()` 啟用時，`buildMemorySystemPromptAddition(...)` 會維持
同步，並讀取該不可變的執行快照。
請原封不動地傳遞所提供的工具與引用內容，確保快照
不會跨越執行邊界。

當使用中的測試框架具有持久性後端執行緒時，`assemble()` 可傳回
`contextProjection`。舊版的逐輪投影則省略此項。當組裝後的內容應
一次注入後端執行緒，並在 epoch 變更前重複使用時，請傳回
`{ mode: "thread_bootstrap", epoch }`。在引擎的語意內容變更後，
例如完成由引擎擁有的壓縮處理後，請變更 epoch。主機可在線程啟動投影中
保留工具呼叫中繼資料、輸入形狀，以及經遮蔽的工具結果，讓新的
後端執行緒得以維持工具連續性，而不必複製含有原始機密的
酬載。

如果你的引擎**不**擁有壓縮演算法，請保留 `compact()`
的實作，並明確委派：

```ts
import {
  buildMemorySystemPromptAddition,
  delegateCompactionToRuntime,
} from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", (ctx) => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, sessionKey, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## 新增功能

當外掛需要目前 API 無法容納的行為時，請勿透過私有的內部存取
繞過外掛系統。請新增缺少的功能。

建議順序：

1. **定義核心合約。** 決定核心應擁有哪些共用行為：
   政策、後援機制、設定合併、生命週期、面向通道的語意，以及
   執行階段輔助函式的形式。
2. **新增具型別的外掛註冊／執行階段介面。** 以最小且實用的具型別
   功能介面擴充 `OpenClawPluginApi` 和／或 `api.runtime`。
3. **串接核心與通道／功能使用端。** 通道與功能外掛
   應透過核心使用新功能，而不是直接匯入供應商
   實作。
4. **註冊供應商實作。** 接著由供應商外掛針對此功能
   註冊其後端。
5. **新增合約涵蓋範圍。** 新增測試，確保所有權與註冊形式
   隨時間推移仍保持明確。

OpenClaw 正是透過這種方式維持鮮明立場，同時避免將單一
供應商的觀點硬編碼其中。具體的檔案檢查清單與完整範例，請參閱
[功能實作指南](/zh-TW/plugins/adding-capabilities)。

### 功能檢查清單

新增功能時，實作通常應同時涵蓋以下
介面：

- `src/<capability>/types.ts` 中的核心合約型別
- `src/<capability>/runtime.ts` 中的核心執行器／執行階段輔助函式
- `src/plugins/types.ts` 中的外掛 API 註冊介面
- `src/plugins/registry.ts` 中的外掛登錄串接
- 當功能／通道外掛需要使用此功能時，於 `src/plugins/runtime/*`
  公開外掛執行階段
- `src/test-utils/plugin-registration.ts` 中的擷取／測試輔助函式
- `src/plugins/contracts/registry.ts` 中的所有權／合約斷言
- `docs/` 中的操作人員／外掛文件

若缺少上述任一介面，通常表示該功能
尚未完整整合。

### 功能範本

最小模式：

```ts
// core contract
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// plugin API
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// shared runtime helper for feature/channel plugins
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

合約測試模式（`src/plugins/contracts/registry.ts` 公開 `providerContractPluginIds` 等所有權
查詢；測試會斷言外掛的 `contracts.videoGenerationProviders` 清單與其實際註冊項目相符）：

```ts
expect(pluginManifest.contracts?.videoGenerationProviders).toEqual(["openai"]);
```

如此可讓規則保持簡單：

- 核心擁有功能合約與協調流程
- 供應商外掛擁有供應商實作
- 功能／通道外掛使用執行階段輔助函式
- 合約測試確保所有權保持明確

## 相關內容

- [外掛架構](/zh-TW/plugins/architecture) — 公開功能模型與形式
- [外掛 SDK 子路徑](/zh-TW/plugins/sdk-subpaths)
- [外掛 SDK 設定](/zh-TW/plugins/sdk-setup)
- [建置外掛](/zh-TW/plugins/building-plugins)
