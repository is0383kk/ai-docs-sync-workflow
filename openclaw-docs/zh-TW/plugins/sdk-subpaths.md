---
read_when:
    - 為外掛匯入選擇正確的 plugin-sdk 子路徑
    - 稽核內建外掛的子路徑與輔助介面
summary: 外掛 SDK 子路徑目錄：各匯入項目的所在位置，依領域分組
title: 外掛 SDK 子路徑
x-i18n:
    generated_at: "2026-07-26T08:36:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 58df43436d0e26f1ffa1383be47fd108655e57d61cf5534d650a4fa2fb7b364c
    source_path: plugins/sdk-subpaths.md
    workflow: 16
---

外掛 SDK 包含狹窄的公開子路徑，以及位於 `openclaw/plugin-sdk/` 下、僅供儲存庫使用的內建輔助程式。本頁列出兩者，並明確標示私有本機項目。以下三個檔案定義其邊界：

- `scripts/lib/plugin-sdk-entrypoints.json`：建置時編譯的維護中進入點清單。
- `scripts/lib/plugin-sdk-private-local-only-subpaths.json`：從具型別且有文件記載的 SDK 中排除的內部子路徑。正式環境項目仍以僅 JavaScript 的主機執行階段匯出形式，供另行發布的官方外掛使用；僅供測試的項目則維持不匯出。
- `src/plugin-sdk/entrypoints.ts`：已棄用子路徑、保留的內建輔助程式、支援的內建外觀介面，以及外掛擁有的公開介面之分類中繼資料。

維護者使用 `pnpm plugin-sdk:surface` 稽核公開匯出數量，並使用 `pnpm plugins:boundary-report:summary` 稽核使用中的保留輔助子路徑；未使用的保留輔助匯出會導致 CI 報告失敗，而不會以休眠的相容性債務形式留在公開 SDK 中。

如需外掛編寫指南，請參閱[外掛 SDK 概觀](/zh-TW/plugins/sdk-overview)。

## 外掛進入點

| 子路徑                        | 主要匯出                                                                                                                                                                                             |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`      | `definePluginEntry`                                                                                                                                                                                     |
| `plugin-sdk/core`              | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema`, `buildJsonChannelConfigSchema`, `resolveTailscalePublishedHost` |
| `plugin-sdk/provider-entry`    | 2026 年 7 月後改為私有本機；`defineSingleProviderPluginEntry`                                                                                                                                        |
| `plugin-sdk/migration`         | 2026 年 7 月後改為私有本機；遷移提供者項目輔助程式，例如 `createMigrationItem`、原因常數、項目狀態標記、遮蔽輔助程式，以及 `summarizeMigrationItems`                   |
| `plugin-sdk/migration-runtime` | 2026 年 7 月後改為私有本機；執行階段遷移輔助程式，例如 `copyMigrationFileItem`、`resolvePlannedMigrationTargets`、`withCachedMigrationConfigRuntime` 及 `writeMigrationReport`              |
| `plugin-sdk/health`            | 供內建健康狀態消費端使用的 Doctor 健康檢查註冊、偵測、修復、選取、嚴重性及發現項目型別                                                                                |

### 相容性與私有本機輔助程式

只有較晚時段的已棄用子路徑仍維持匯出。2026 年 7 月的別名與未使用子路徑已刪除，而僅供內建使用的輔助程式已從公開套件移除，並在下方標示為私有本機。維護中的清單是 `scripts/lib/plugin-sdk-deprecated-public-subpaths.json`；CI 會拒絕內建
`plugin-sdk/text-runtime` 僅供相容性使用，而 `plugin-sdk/zod` 是相容性重新匯出：請直接從 `zod` 匯入 `zod`。廣泛的領域彙總進入點 `plugin-sdk/agent-runtime`、`plugin-sdk/channel-lifecycle`、
`plugin-sdk/conversation-runtime`、`plugin-sdk/hook-runtime`、
`plugin-sdk/media-runtime`、`plugin-sdk/plugin-runtime` 及
`plugin-sdk/security-runtime` 也同樣已棄用，應改用聚焦的子路徑。

OpenClaw 以 Vitest 為基礎的測試輔助子路徑僅供儲存庫本機使用，且不再是套件匯出：`agent-runtime-test-contracts`、
`channel-contract-testing`、`channel-target-testing`、`channel-test-helpers`、
`plugin-state-test-runtime`、`plugin-test-api`、`plugin-test-contracts`、
`plugin-test-runtime`、`provider-http-test-mocks`、`provider-test-contracts`、
`reply-payload-testing`、`sqlite-runtime-testing`、`test-env`、`test-fixtures`、
`test-live`、`test-live-auth`、`test-media-generation`、
`test-media-understanding`、`test-node-mocks` 及 `testing`。私有內建輔助介面 `ssrf-runtime-internal` 與 `codex-native-task-runtime` 也僅供儲存庫本機使用。

### 內建外掛輔助子路徑

在 2026 年 7 月清理後，僅供內建使用的輔助模組改為私有本機。套件合約防護機制會封鎖跨擁有者匯入。`src/plugin-sdk/entrypoints.ts` 另行追蹤仍維持公開的受支援內建外觀介面，也就是由其內建外掛支援的 SDK 進入點，直到通用合約取代 `plugin-sdk/qa-runner-runtime`、`plugin-sdk/telegram-account`；不建議新程式碼使用，請參閱下方各列附註。

<AccordionGroup>
  <Accordion title="頻道子路徑">
    | 子路徑 | 主要匯出 |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`、`defineSetupPluginEntry`、`createChatChannelPlugin`、`createChannelPluginBase`、`createChannelConfigUiHints` |
    | `plugin-sdk/json-schema-runtime` | 2026 年 7 月後改為私有本機；供外掛擁有之結構描述使用的快取 JSON Schema 驗證輔助程式 |
    | `plugin-sdk/channel-setup` | `defineChannelSetupContract`、頻道擁有的設定欄位／輸入型別、`createOptionalChannelSetupSurface`、`createOptionalChannelSetupAdapter`、`createOptionalChannelSetupWizard`，以及 `DEFAULT_ACCOUNT_ID`、`createTopLevelChannelDmPolicy`、`setSetupChannelEnabled`、`splitSetupEntries` |
    | `plugin-sdk/setup` | 共用設定精靈輔助程式、設定轉譯器、允許清單提示及設定狀態建構器 |
    | `plugin-sdk/setup-runtime` | `defineChannelSetupContract`、`createSetupTranslator`、`createPatchedAccountSetupAdapter`、`createEnvPatchedAccountSetupAdapter`、`createSetupInputPresenceValidator`、`noteChannelLookupFailure`、`noteChannelLookupSummary`、`promptResolvedAllowFrom`、`splitSetupEntries`、`createAllowlistSetupWizardProxy`、`createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`、`detectBinary`、`extractArchive`、`resolveBrewExecutable`、`formatDocsLink`、`CONFIG_DIR` |
    | `plugin-sdk/account-core` | 多帳號設定／動作閘門輔助程式、預設帳號後援輔助程式 |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`、帳號 ID 正規化輔助程式 |
    | `plugin-sdk/account-resolution` | 帳號查詢與預設後援輔助程式 |
    | `plugin-sdk/account-helpers` | 狹窄的帳號清單／帳號動作輔助程式 |
    | `plugin-sdk/access-groups` | 2026 年 7 月後改為私有本機；存取群組允許清單剖析與已遮蔽群組診斷輔助程式 |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | 已棄用的相容性外觀介面。請使用 `plugin-sdk/channel-outbound`。 |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter`、`resolveChannelDmAccess`、`resolveChannelDmAllowFrom`、`resolveChannelDmPolicy`、`normalizeChannelDmPolicy`、`normalizeLegacyDmAliases` |
    | `plugin-sdk/channel-config-schema` | 共用頻道設定結構描述基礎元件，以及 Zod 與直接 JSON／TypeBox 建構器 |
    | `plugin-sdk/bundled-channel-config-schema` | 2026 年 7 月後改為私有本機；僅供維護中內建外掛使用的內建 OpenClaw 頻道設定結構描述 |
    | `plugin-sdk/chat-channel-ids` | 2026 年 7 月後改為私有本機；`BUNDLED_CHAT_CHANNEL_IDS`、`BUNDLED_CHAT_CHANNEL_ENVELOPE_PREFIXES`、`ChatChannelId`。標準的內建／官方聊天頻道 ID，以及供需要辨識帶信封前綴文字而不必硬式編碼自有資料表之外掛使用的格式化工具標籤／別名。 |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-ingress-runtime` | 實驗性的高階頻道傳入執行階段解析器、隱含提及政策解析器，以及供已遷移頻道接收路徑使用的路由事實建構器。請優先使用此介面，而不要在每個外掛中組合有效允許清單、命令允許清單及舊版投影。請參閱[頻道傳入 API](/zh-TW/plugins/sdk-channel-ingress)。 |
    | `plugin-sdk/channel-lifecycle` | 已棄用的相容性外觀介面。請使用 `plugin-sdk/channel-outbound`。 |
    | `plugin-sdk/channel-outbound` | 訊息生命週期合約，以及回覆流水線選項、收執、即時預覽／串流、生命週期輔助程式、輸出身分、承載內容規劃、持久傳送及訊息傳送內容輔助程式。請參閱[頻道輸出 API](/zh-TW/plugins/sdk-channel-outbound)。 |
    | `plugin-sdk/channel-message` | `plugin-sdk/channel-outbound` 的已棄用相容性別名。 |
    | `plugin-sdk/inbound-envelope` | 共用傳入路由與信封建構器輔助程式 |
    | `plugin-sdk/inbound-reply-dispatch` | 已棄用的相容性外觀介面。傳入執行器與分派述詞請使用 `plugin-sdk/channel-inbound`，訊息遞送輔助程式則使用 `plugin-sdk/channel-outbound`。 |
    | `plugin-sdk/messaging-targets` | 已棄用的目標剖析別名；請使用 `plugin-sdk/channel-targets` |
    | `plugin-sdk/outbound-media` | 2026 年 7 月後改為私有本機；共用輸出媒體載入與託管媒體狀態輔助程式 |
    | `plugin-sdk/poll-runtime` | 2026 年 7 月後改為私有本機；狹窄的投票正規化輔助程式 |
    | `plugin-sdk/thread-bindings-runtime` | 2026 年 7 月後改為私有本機；討論串繫結生命週期與轉接器輔助程式 |
    | `plugin-sdk/agent-media-payload` | 舊版 `Media*` 承載內容投影的已棄用相容性外觀介面。請透過 `MsgContext.media`／`toInboundMediaFacts(...)` 傳遞排序後的事實；從 `plugin-sdk/media-local-roots` 匯入本機根目錄政策。 |
    | `plugin-sdk/conversation-runtime` | 對話／討論串繫結、配對及已設定繫結輔助程式的已棄用廣泛彙總進入點；請優先使用聚焦的繫結子路徑，例如 `plugin-sdk/thread-bindings-runtime` 與 `plugin-sdk/session-binding-runtime` |
    | `plugin-sdk/runtime-group-policy` | 執行階段群組政策解析輔助程式 |
    | `plugin-sdk/channel-status` | 共用頻道狀態快照／摘要輔助程式 |
    | `plugin-sdk/channel-config-primitives` | 狹窄的頻道設定結構描述基礎元件 |
    | `plugin-sdk/channel-config-writes` | 2026 年 7 月後改為私有本機；頻道設定寫入授權輔助程式 |
    | `plugin-sdk/channel-plugin-common` | 共用頻道外掛前導匯出 |
    | `plugin-sdk/allowlist-config-edit` | 允許清單設定編輯／讀取輔助程式 |
    | `plugin-sdk/group-access` | 已棄用的群組存取決策輔助程式；請使用來自 `plugin-sdk/channel-ingress-runtime` 的 `resolveChannelMessageIngress` |
    | `plugin-sdk/direct-dm-guard-policy` | 2026 年 7 月後改為私有本機；狹窄的直接私訊加密前防護政策輔助程式 |
    | `plugin-sdk/discord` | 供已發布 `@openclaw/discord@2026.3.13` 與受追蹤擁有者相容性使用的已棄用 Discord 相容性外觀介面；新外掛應使用通用頻道 SDK 子路徑 |
    | `plugin-sdk/telegram-account` | 供受追蹤擁有者相容性使用的已棄用 Telegram 帳號解析相容性外觀介面；新外掛應使用注入的執行階段輔助程式或通用頻道 SDK 子路徑 |
    | `plugin-sdk/interactive-runtime` | 語意訊息呈現、遞送及舊版互動式回覆輔助程式。請參閱[訊息呈現](/zh-TW/plugins/message-presentation) |
    | `plugin-sdk/question-gateway-runtime` | 從頻道互動處理常式，透過閘道解析執行階段編寫的 `ask_user` 選項 |
    | `plugin-sdk/channel-inbound` | 事件分類、內容建構、格式化、根目錄、防彈跳、提及比對、提及政策及傳入記錄的共用傳入輔助程式 |
    | `plugin-sdk/channel-inbound-debounce` | 狹窄的傳入防彈跳輔助程式 |
    | `plugin-sdk/channel-mention-gating` | 2026 年 7 月後改為私有本機；不含較廣泛傳入執行階段介面的狹窄提及政策、提及標記及提及文字輔助程式 |
    | `plugin-sdk/channel-streaming` | 已棄用的相容性外觀介面。請使用 `plugin-sdk/channel-outbound`。 |
    | `plugin-sdk/channel-send-result` | 回覆結果型別 |
    | `plugin-sdk/channel-actions` | 頻道訊息動作輔助程式，以及為外掛相容性而保留的已棄用原生結構描述輔助程式 |
    | `plugin-sdk/channel-route` | 2026 年 7 月後改為私有本機；共用路由正規化、由剖析器驅動的目標解析、討論串 ID 字串化、去重／精簡路由鍵、已剖析目標型別，以及路由／目標比較輔助程式 |
    | `plugin-sdk/channel-targets` | 2026 年 7 月後改為私有本機；目標剖析輔助程式；路由比較呼叫端應使用 `plugin-sdk/channel-route` |
    | `plugin-sdk/channel-contract` | 頻道合約型別 |
    | `plugin-sdk/channel-feedback` | 意見回饋／表情回應接線 |
  </Accordion>

較晚時段的頻道相容性子路徑僅在其登錄日期內維持公開。直接私訊存取、回覆選項、配對路徑及頻道執行階段分支等 7 月別名均已移除；僅供內建使用的輔助程式則為私有本機。

  <Accordion title="供應商子路徑">
    | 子路徑 | 主要匯出項目 |
    | --- | --- |
    | `plugin-sdk/provider-entry` | 2026 年 7 月後為私有本機項目；`defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | 2026 年 7 月後為私有本機項目；精選的本機／自行託管供應商設定輔助工具 |
    | `plugin-sdk/cli-backend` | 2026 年 7 月後為私有本機項目；命令列介面後端預設值與監看程式常數 |
    | `plugin-sdk/provider-auth-runtime` | 2026 年 7 月後為私有本機項目；供應商驗證執行階段輔助工具：OAuth 回送流程、權杖交換、驗證持久化與 API 金鑰解析 |
    | `plugin-sdk/provider-oauth-runtime` | 2026 年 7 月後為私有本機項目；通用供應商 OAuth 回呼型別、回呼頁面呈現、PKCE／狀態輔助工具、授權輸入剖析、權杖到期輔助工具與中止輔助工具 |
    | `plugin-sdk/provider-auth-api-key` | 2026 年 7 月後為私有本機項目；API 金鑰初始設定／設定檔寫入輔助工具，例如 `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | 2026 年 7 月後為私有本機項目；標準 OAuth 驗證結果建構器 |
    | `plugin-sdk/provider-env-vars` | 2026 年 7 月後為私有本機項目；供應商驗證環境變數查詢輔助工具 |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`、`ensureApiKeyFromOptionEnvOrPrompt`、`upsertAuthProfile`、`upsertApiKeyProfile`、`writeOAuthCredentials`、OpenAI Codex 驗證匯入輔助工具、已棄用的 `resolveOpenClawAgentDir` 相容性匯出項目 |
    | `plugin-sdk/provider-model-shared` | 2026 年 7 月後為私有本機項目；`ProviderReplayFamily`、`buildProviderReplayFamilyHooks`、`selectPreferredLocalModelId`、`normalizeModelCompat`、共用重播原則建構器、供應商端點輔助工具與共用模型 ID 正規化輔助工具 |
    | `plugin-sdk/provider-catalog-live-runtime` | 2026 年 7 月後為私有本機項目；用於受防護的 `/models` 樣式探索之即時供應商模型目錄輔助工具：`buildLiveModelProviderConfig`、`fetchLiveProviderModelRows`、`getCachedLiveProviderModelRows`、`fetchLiveProviderModelIds`、`LiveModelCatalogHttpError`、`clearLiveCatalogCacheForTests`、模型 ID 篩選、TTL 快取與靜態備援 |
    | `plugin-sdk/provider-catalog-runtime` | 供應商目錄擴充執行階段掛鉤，以及用於契約測試的外掛供應商登錄介面 |
    | `plugin-sdk/provider-catalog-shared` | 2026 年 7 月後為私有本機項目；`findCatalogTemplate`、`buildSingleProviderApiKeyCatalog`、`buildManifestModelProviderConfig`、`supportsNativeStreamingUsageCompat`、`applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | 2026 年 7 月後為私有本機項目；通用供應商 HTTP／端點功能輔助工具、供應商 HTTP 錯誤與音訊轉錄多部分表單輔助工具 |
    | `plugin-sdk/provider-web-fetch-contract` | 2026 年 7 月後為私有本機項目；精簡的網頁擷取設定／選擇契約輔助工具，例如 `enablePluginInConfig` 與 `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | 2026 年 7 月後為私有本機項目；網頁擷取供應商註冊／快取輔助工具 |
    | `plugin-sdk/provider-web-search-config-contract` | 2026 年 7 月後為私有本機項目；適用於不需要外掛啟用接線之供應商的精簡網頁搜尋設定／認證資訊輔助工具 |
    | `plugin-sdk/provider-web-search-contract` | 2026 年 7 月後為私有本機項目；精簡的網頁搜尋設定／認證資訊契約輔助工具，例如 `createWebSearchProviderContractFields`、`enablePluginInConfig`、`resolveProviderWebSearchPluginConfig`，以及限範圍的認證資訊設定器／取得器 |
    | `plugin-sdk/provider-web-search` | 2026 年 7 月後為私有本機項目；網頁搜尋供應商註冊／快取／執行階段輔助工具 |
    | `plugin-sdk/embedding-providers` | 2026 年 7 月後為私有本機項目；通用嵌入供應商型別與讀取輔助工具，包括 `EmbeddingProviderAdapter`、`getEmbeddingProvider(...)` 與 `listEmbeddingProviders(...)`；外掛透過 `api.registerEmbeddingProvider(...)` 註冊供應商，以強制執行資訊清單所有權 |
    | `plugin-sdk/provider-tools` | 2026 年 7 月後為私有本機項目；`ProviderToolCompatFamily`、`buildProviderToolCompatFamilyHooks`，以及 DeepSeek／Gemini／OpenAI 結構描述清理與診斷 |
    | `plugin-sdk/provider-usage` | 2026 年 7 月後為私有本機項目；供應商用量快照型別、共用用量擷取輔助工具，以及 `fetchClaudeUsage` 等供應商擷取器 |
    | `plugin-sdk/provider-stream` | 2026 年 7 月後為私有本機項目；`ProviderStreamFamily`、`buildProviderStreamFamilyHooks`、`composeProviderStreamWrappers`、串流包裝器型別、純文字工具呼叫相容性，以及共用的 Anthropic／Google／Kilocode／MiniMax／Moonshot／OpenAI／OpenRouter／Z.AI 包裝器輔助工具 |
    | `plugin-sdk/provider-stream-shared` | 2026 年 7 月後為私有本機項目；公開的共用供應商串流包裝器輔助工具，包括 `composeProviderStreamWrappers`、`createOpenAICompatibleCompletionsThinkingOffWrapper`、`createPlainTextToolCallCompatWrapper`、`createPayloadPatchStreamWrapper`、`createToolStreamWrapper`、`normalizeOpenAICompatibleReasoningPayload`、`setQwenChatTemplateThinking`，以及與 Anthropic／DeepSeek／OpenAI 相容的串流公用工具 |
    | `plugin-sdk/provider-transport-runtime` | 2026 年 7 月後為私有本機項目；原生供應商傳輸輔助工具，例如受防護的擷取、工具結果文字擷取、傳輸訊息轉換，以及可寫入的傳輸事件串流 |
    | `plugin-sdk/provider-onboard` | 2026 年 7 月後為私有本機項目；初始設定組態修補輔助工具 |
    | `plugin-sdk/global-singleton` | 2026 年 7 月後為私有本機項目；程序本機單例／對應表／快取輔助工具 |
    | `plugin-sdk/group-activation` | 2026 年 7 月後為私有本機項目；精簡的群組啟用模式與命令剖析輔助工具 |
  </Accordion>

供應商用量快照通常會回報一或多個配額 `windows`，每個項目都包含
標籤、已使用百分比，以及選用的重設時間。若供應商公開的是餘額或
帳戶狀態文字，而非可重設的配額期間，則應回傳
`summary` 並使用空的 `windows` 陣列，而不是捏造百分比。
OpenClaw 會在狀態輸出中顯示該摘要文字；僅當
用量端點失敗或未回傳可用的用量資料時，才使用 `error`。

  <Accordion title="驗證與安全性子路徑">
    | 子路徑 | 主要匯出項目 |
    | --- | --- |
    | `plugin-sdk/command-auth` | 已棄用的廣泛命令授權介面（`resolveControlCommandGate`、命令登錄輔助工具，包括動態引數選單格式化、傳送者授權輔助工具）；請改用頻道輸入／執行階段授權或命令狀態輔助工具 |
    | `plugin-sdk/command-status` | 命令／說明訊息建構器，例如 `buildCommandsMessagePaginated` 與 `buildHelpMessage` |
    | `plugin-sdk/approval-auth-runtime` | 核准者解析與同一聊天的動作驗證輔助工具 |
    | `plugin-sdk/approval-client-runtime` | 原生執行核准設定檔／篩選器輔助工具 |
    | `plugin-sdk/approval-delivery-runtime` | 原生核准功能／傳遞配接器 |
    | `plugin-sdk/approval-gateway-runtime` | 共用核准閘道解析器 |
    | `plugin-sdk/approval-reference-runtime` | 2026 年 7 月後為私有本機項目；適用於傳輸受限核准回呼的確定性持久定位器輔助工具 |
    | `plugin-sdk/approval-handler-adapter-runtime` | 適用於高頻頻道進入點的輕量原生核准配接器載入輔助工具 |
    | `plugin-sdk/approval-handler-runtime` | 較廣泛的核准處理常式執行階段輔助工具；當較精簡的配接器／閘道介面足夠時，應優先使用它們 |
    | `plugin-sdk/approval-native-runtime` | 原生核准目標、帳戶繫結、路由閘門、轉送備援，以及本機原生執行提示抑制輔助工具 |
    | `plugin-sdk/approval-reaction-runtime` | 2026 年 7 月後為私有本機項目；硬編碼的核准回應繫結、回應提示承載資料、回應目標儲存區、回應提示文字輔助工具，以及本機原生執行提示抑制的相容性匯出項目 |
    | `plugin-sdk/approval-reply-runtime` | 執行／外掛核准回覆承載資料輔助工具 |
    | `plugin-sdk/approval-runtime` | 執行／外掛核准承載資料輔助工具、核准功能建構器、核准驗證／設定檔輔助工具、原生核准路由／執行階段輔助工具，以及 `formatApprovalDisplayPath` 等結構化核准顯示輔助工具 |
    | `plugin-sdk/command-auth-native` | 原生命令驗證、動態引數選單格式化與原生工作階段目標輔助工具 |
    | `plugin-sdk/command-detection` | 共用命令偵測輔助工具 |
    | `plugin-sdk/command-primitives-runtime` | 適用於高頻頻道路徑的輕量命令文字述詞 |
    | `plugin-sdk/command-surface` | 2026 年 7 月後為私有本機項目；命令本文正規化與命令介面輔助工具 |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/provider-auth-login-flow-runtime` | 2026 年 7 月後為私有本機項目；用於私人頻道與 Web UI 裝置代碼配對的延遲供應商驗證登入流程輔助工具 |
    | `plugin-sdk/channel-secret-runtime` | 已棄用的廣泛祕密契約介面（`collectSimpleChannelFieldAssignments`、`getChannelSurface`、`pushAssignment`、祕密目標型別）；請優先使用下方的聚焦子路徑 |
    | `plugin-sdk/channel-secret-basic-runtime` | 用於非 TTS 頻道／外掛祕密介面的精簡祕密契約匯出項目與目標登錄建構器 |
    | `plugin-sdk/channel-secret-tts-runtime` | 2026 年 7 月後為私有本機項目；精簡的巢狀頻道 TTS 祕密指派輔助工具 |
    | `plugin-sdk/secret-ref-runtime` | 用於祕密契約／組態剖析的精簡 SecretRef 型別、解析，以及計畫目標路徑查詢 |
    | `plugin-sdk/security-runtime` | 已棄用的廣泛彙總介面，涵蓋信任、私人訊息閘控、根目錄範圍限制的檔案／路徑輔助工具，包括僅建立寫入、同步／非同步不可分割檔案取代、同層暫存寫入、跨裝置移動備援、私人檔案儲存輔助工具、符號連結父目錄防護、外部內容、敏感文字遮蔽、固定時間祕密比較，以及祕密收集輔助工具；請優先使用聚焦的安全性／SSRF／祕密子路徑 |
    | `plugin-sdk/ssrf-policy` | 主機允許清單與私人網路 SSRF 原則輔助工具 |
    | `plugin-sdk/ssrf-dispatcher` | 2026 年 7 月後為私有本機項目；不含廣泛基礎架構執行階段介面的精簡固定分派器輔助工具 |
    | `plugin-sdk/ssrf-runtime` | 固定分派器、受 SSRF 防護的擷取、SSRF 錯誤與 SSRF 原則輔助工具 |
    | `plugin-sdk/secret-input` | 祕密輸入剖析輔助工具 |
    | `plugin-sdk/webhook-ingress` | 網路鉤子要求／目標輔助工具，以及原始 WebSocket／本文強制轉型 |
    | `plugin-sdk/webhook-request-guards` | 要求本文大小／逾時輔助工具，以及用於追蹤確認後處理的 `runDetachedWebhookWork` |
  </Accordion>

  <Accordion title="Runtime and storage subpaths">
    | 子路徑 | 主要匯出項目 |
    | --- | --- |
    | `plugin-sdk/runtime` | 執行階段／記錄／備份輔助函式、外掛安裝路徑警告及程序輔助函式 |
    | `plugin-sdk/runtime-env` | 精簡的執行階段環境、記錄器、逾時、重試及退避輔助函式 |
    | `plugin-sdk/browser-config` | 2026 年 7 月後為私有本機項目；支援的瀏覽器設定門面，用於正規化的設定檔／預設值、CDP URL 剖析及瀏覽器控制驗證輔助函式 |
    | `plugin-sdk/agent-harness-task-runtime` | 2026 年 7 月後為私有本機項目；供使用主機核發任務範圍且由控管環境支援的代理程式使用的一般任務生命週期及完成傳遞輔助函式 |
    | `plugin-sdk/codex-mcp-projection` | 2026 年 7 月後為私有本機項目；保留的內建 Codex 輔助函式，用於將使用者的 MCP 伺服器設定投影至 Codex 執行緒設定；不供第三方外掛使用 |
    | `plugin-sdk/codex-native-task-runtime` | 儲存庫本機的內建 Codex 輔助函式，用於原生任務鏡像／執行階段接線；不是套件匯出項目 |
    | `plugin-sdk/channel-runtime-context` | 一般頻道執行階段情境註冊及查詢輔助函式 |
    | `plugin-sdk/matrix` | 已棄用的 Matrix 相容性門面，供較舊的第三方頻道套件使用；新外掛應直接匯入 `plugin-sdk/run-command` |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | 已棄用的廣泛匯出介面，涵蓋外掛命令／掛鉤／HTTP／互動式輔助函式；請優先使用聚焦的外掛執行階段子路徑 |
    | `plugin-sdk/hook-runtime` | 已棄用的廣泛匯出介面，涵蓋網路鉤子／內部掛鉤流水線輔助函式；請優先使用聚焦的掛鉤／外掛執行階段子路徑 |
    | `plugin-sdk/lazy-runtime` | 延遲載入的執行階段匯入／繫結輔助函式，例如 `createLazyRuntimeModule`、`createLazyRuntimeMethod` 及 `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | 2026 年 7 月後為私有本機項目；程序執行輔助函式 |
    | `plugin-sdk/node-host` | 2026 年 7 月後為私有本機項目；節點主機可執行檔解析及 PTY 繼續執行輔助函式 |
    | `plugin-sdk/cli-runtime` | 2026 年 7 月後為私有本機項目；已棄用的廣泛匯出介面，涵蓋命令列介面格式化、等待、版本、引數叫用及延遲載入命令群組輔助函式；請優先使用聚焦的命令列介面／執行階段子路徑 |
    | `plugin-sdk/qa-runner-runtime` | 2026 年 7 月後為私有本機項目；支援的門面，透過命令列介面命令介面公開外掛 QA 情境 |
    | `plugin-sdk/tts-runtime` | 2026 年 7 月後為私有本機項目；支援的文字轉語音設定結構描述及執行階段輔助函式門面 |
    | `plugin-sdk/gateway-method-runtime` | 保留的閘道方法分派輔助函式，供宣告 `contracts.gatewayMethodDispatch: ["authenticated-request"]` 的外掛 HTTP 路由使用 |
    | `plugin-sdk/gateway-runtime` | 閘道用戶端、事件迴圈就緒的用戶端啟動輔助函式、閘道命令列介面 RPC、閘道通訊協定錯誤、公告的區域網路主機解析，以及頻道狀態修補輔助函式 |
    | `plugin-sdk/config-contracts` | 聚焦的純型別設定介面，適用於 `OpenClawConfig` 等外掛設定形態及頻道／提供者設定型別 |
    | `plugin-sdk/plugin-config-runtime` | 已棄用的執行階段外掛設定輔助函式相容性門面；新外掛使用 `api.pluginConfig`，以及聚焦的設定合約、快照及異動輔助函式 |
    | `plugin-sdk/config-mutation` | 交易式設定異動輔助函式，例如 `mutateConfigFile`、`replaceConfigFile` 及 `logConfigUpdated` |
    | `plugin-sdk/message-tool-delivery-hints` | 2026 年 7 月後為私有本機項目；共用訊息工具傳遞中繼資料提示字串 |
    | `plugin-sdk/runtime-config-snapshot` | 目前程序設定快照輔助函式，例如 `getRuntimeConfig`、`getRuntimeConfigSnapshot` 及測試快照設定函式 |
    | `plugin-sdk/text-autolink-runtime` | 2026 年 7 月後為私有本機項目；不使用廣泛文字匯出介面的檔案參照自動連結偵測 |
    | `plugin-sdk/reply-runtime` | 共用傳入／回覆執行階段輔助函式、分塊、分派、心跳偵測、回覆規劃器 |
    | `plugin-sdk/reply-dispatch-runtime` | 精簡的回覆分派／完成處理及對話標籤輔助函式 |
    | `plugin-sdk/reply-history` | 共用短時間範圍回覆記錄輔助函式。新的訊息輪次程式碼應使用 `createChannelHistoryWindow`；較低階的對應表輔助函式僅保留為已棄用的相容性匯出項目 |
    | `plugin-sdk/reply-reference` | 2026 年 7 月後為私有本機項目；`createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | 精簡的文字／Markdown 分塊輔助函式 |
    | `plugin-sdk/session-store-runtime` | 工作階段工作流程輔助函式（`getSessionEntry`、`listSessionEntries`、`patchSessionEntry`、`upsertSessionEntry`）、修復／生命週期輔助函式（`deleteSessionEntry`、`cleanupSessionLifecycleArtifacts`、`resolveSessionStoreBackupPaths`）、過渡性 `sessionFile` 值的標記輔助函式、依工作階段身分進行有界限的近期使用者／助理逐字稿文字讀取、工作階段儲存區路徑／工作階段索引鍵輔助函式，以及更新時間讀取，且不匯入廣泛的設定寫入／維護功能 |
    | `plugin-sdk/session-transcript-runtime` | 2026 年 7 月後為私有本機項目；逐字稿身分、有界限的原始與可見游標、限定範圍的目標／讀取／寫入輔助函式、可見訊息項目投影、更新發布、寫入鎖，以及逐字稿記憶命中索引鍵 |
    | `plugin-sdk/sqlite-runtime` | 2026 年 7 月後為私有本機項目；供第一方執行階段使用且不含資料庫生命週期控制的聚焦 SQLite 代理程式結構描述、路徑及交易輔助函式 |
    | `plugin-sdk/cron-store-runtime` | 2026 年 7 月後為私有本機項目；排程儲存區路徑／載入／儲存輔助函式 |
    | `plugin-sdk/state-paths` | 狀態／OAuth 目錄路徑輔助函式 |
    | `plugin-sdk/plugin-state-runtime` | 2026 年 7 月後為私有本機項目；外掛限定範圍的鍵控狀態、BLOB 及協作式 SQLite 租約合約，以及連線 pragma、經驗證的 WAL 維護和不可分割的 STRICT 結構描述遷移輔助函式。租約回呼會收到中止訊號，且具型別的錯誤會區分逾時、取消、所有權遺失、無效輸入及儲存失敗 |
    | `plugin-sdk/routing` | 路由／工作階段索引鍵／帳號繫結輔助函式，例如 `resolveAgentRoute`、`buildAgentSessionKey` 及 `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | 共用頻道／帳號狀態摘要輔助函式、執行階段狀態預設值及問題中繼資料輔助函式 |
    | `plugin-sdk/target-resolver-runtime` | 2026 年 7 月後為私有本機項目；共用目標解析器輔助函式 |
    | `plugin-sdk/string-normalization-runtime` | 2026 年 7 月後為私有本機項目；Slug／字串正規化輔助函式 |
    | `plugin-sdk/request-url` | 2026 年 7 月後為私有本機項目；從類似 fetch／request 的輸入擷取字串 URL |
    | `plugin-sdk/run-command` | 具正規化 stdout／stderr 結果的計時命令執行器 |
    | `plugin-sdk/param-readers` | 通用工具／命令列介面參數讀取器 |
    | `plugin-sdk/tool-plugin` | 定義簡易具型別的代理程式工具外掛，並公開靜態中繼資料以產生資訊清單 |
    | `plugin-sdk/tool-payload` | 2026 年 7 月後為私有本機項目；從工具結果物件擷取正規化承載資料 |
    | `plugin-sdk/tool-send` | 從工具引數擷取標準傳送目標欄位 |
    | `plugin-sdk/sandbox` | 2026 年 7 月後為私有本機項目；沙箱後端型別及 SSH／OpenShell 命令輔助函式，包括快速失敗的執行命令預先檢查 |
    | `plugin-sdk/temp-path` | 共用暫存下載路徑輔助函式及私有安全暫存工作區 |
    | `plugin-sdk/logging-core` | 子系統記錄器及遮蔽輔助函式 |
    | `plugin-sdk/markdown-table-runtime` | 2026 年 7 月後為私有本機項目；Markdown 表格模式及轉換輔助函式 |
    | `plugin-sdk/model-session-runtime` | 模型／工作階段覆寫輔助函式，例如 `applyModelOverrideToSessionEntry` 及 `resolveAgentMaxConcurrent` |
    | `plugin-sdk/talk-config-runtime` | 2026 年 7 月後為私有本機項目；Talk 提供者設定解析輔助函式 |
    | `plugin-sdk/json-store` | 小型 JSON 狀態讀取／寫入輔助函式 |
    | `plugin-sdk/json-unsafe-integers` | 2026 年 7 月後為私有本機項目；將不安全的整數常值保留為字串的 JSON 剖析輔助函式 |
    | `plugin-sdk/file-lock` | 2026 年 7 月後為私有本機項目；可重新進入的檔案鎖定輔助函式，以及 Doctor 安全回收已確定過時、未變更且已停用的鎖定附屬檔案 |
    | `plugin-sdk/persistent-dedupe` | 磁碟支援的去重複快取輔助函式 |
    | `plugin-sdk/ingress-effect-once` | 非等冪傳入副作用的持久宣告／提交防護機制 |
    | `plugin-sdk/acp-runtime` | 2026 年 7 月後為私有本機項目；ACP 執行階段／工作階段及回覆分派輔助函式 |
    | `plugin-sdk/acp-runtime-backend` | 2026 年 7 月後為私有本機項目；供啟動時載入之外掛使用的輕量 ACP 後端註冊及回覆分派輔助函式 |
    | `plugin-sdk/acp-binding-resolve-runtime` | 2026 年 7 月後為私有本機項目；不匯入生命週期啟動功能的唯讀 ACP 繫結解析 |
    | `plugin-sdk/agent-config-primitives` | 已棄用的代理程式執行階段設定結構描述基礎元件；請從維護中的外掛所屬介面匯入結構描述基礎元件 |
    | `plugin-sdk/boolean-param` | 寬鬆布林值參數讀取器 |
    | `plugin-sdk/dangerous-name-runtime` | 2026 年 7 月後為私有本機項目；危險名稱比對解析輔助函式 |
    | `plugin-sdk/device-bootstrap` | 裝置啟動程序及配對權杖輔助函式，包括 `BOOTSTRAP_HANDOFF_OPERATOR_SCOPES` |
    | `plugin-sdk/extension-shared` | 共用被動頻道、狀態及環境代理輔助基礎元件 |
    | `plugin-sdk/models-provider-runtime` | `/models` 命令／提供者回覆輔助函式 |
    | `plugin-sdk/skill-commands-runtime` | Skill 命令列出輔助函式 |
    | `plugin-sdk/native-command-registry` | 原生命令登錄／建置／序列化輔助函式 |
    | `plugin-sdk/agent-harness` | 實驗性的受信任外掛介面，供低階代理程式控管環境使用：控管環境型別、作用中執行的引導／中止輔助函式、OpenClaw 工具橋接輔助函式、執行階段計畫工具原則輔助函式、終端結果分類、工具進度格式化／詳細資料輔助函式，以及嘗試結果公用程式 |
    | `plugin-sdk/async-lock-runtime` | 2026 年 7 月後為私有本機項目；供小型執行階段狀態檔案使用的程序本機非同步鎖定輔助函式 |
    | `plugin-sdk/channel-activity-runtime` | 2026 年 7 月後為私有本機項目；頻道活動遙測輔助函式 |
    | `plugin-sdk/concurrency-runtime` | 2026 年 7 月後為私有本機項目；有界限的非同步任務並行輔助函式 |
    | `plugin-sdk/dedupe-runtime` | 記憶體內及持久性後端支援的去重複快取輔助函式 |
    | `plugin-sdk/delivery-queue-runtime` | 2026 年 7 月後為私有本機項目；傳出待處理傳遞排空輔助函式 |
    | `plugin-sdk/file-access-runtime` | 2026 年 7 月後為私有本機項目；安全的本機檔案及媒體來源路徑輔助函式 |
    | `plugin-sdk/heartbeat-runtime` | 2026 年 7 月後為私有本機項目；心跳偵測喚醒、事件及可見性輔助函式 |
    | `plugin-sdk/expect-runtime` | 2026 年 7 月後為私有本機項目；用於可證明執行階段不變條件的必要值斷言輔助函式 |
    | `plugin-sdk/number-runtime` | 2026 年 7 月後為私有本機項目；數值強制轉型輔助函式 |
    | `plugin-sdk/secure-random-runtime` | 2026 年 7 月後為私有本機項目；安全權杖／UUID 輔助函式 |
    | `plugin-sdk/system-event-runtime` | 2026 年 7 月後為私有本機項目；系統事件佇列輔助函式 |
    | `plugin-sdk/transport-ready-runtime` | 2026 年 7 月後為私有本機項目；傳輸就緒等待輔助函式 |
    | `plugin-sdk/exec-approvals-runtime` | 2026 年 7 月後為私有本機項目；不使用廣泛基礎設施執行階段匯出介面的執行核准原則檔案輔助函式 |
    | `plugin-sdk/infra-runtime` | 已棄用的相容性墊片；請使用上述聚焦的執行階段子路徑 |
    | `plugin-sdk/collection-runtime` | 小型有界限快取輔助函式 |
    | `plugin-sdk/diagnostic-runtime` | 診斷旗標、事件及追蹤情境輔助函式 |
    | `plugin-sdk/error-runtime` | 錯誤圖、格式化、未知值強制轉型、共用錯誤分類輔助函式、`PlatformMessageNotDispatchedError`、`isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | 2026 年 7 月後為私有本機項目；包裝的 fetch、代理、EnvHttpProxyAgent 選項及固定查詢輔助函式 |
    | `plugin-sdk/runtime-fetch` | 2026 年 7 月後為私有本機項目；可感知分派器且不匯入代理／受防護 fetch 功能的執行階段 fetch |
    | `plugin-sdk/inline-image-data-url-runtime` | 2026 年 7 月後為私有本機項目；不使用廣泛媒體執行階段介面的行內影像資料 URL 清理及特徵碼探測輔助函式 |
    | `plugin-sdk/response-limit-runtime` | 2026 年 7 月後為私有本機項目；不使用廣泛媒體執行階段介面的位元組、閒置時間及期限有界限回應本文讀取器 |
    | `plugin-sdk/session-binding-runtime` | 2026 年 7 月後為私有本機項目；不含已設定繫結路由或配對儲存區的目前對話繫結狀態 |
    | `plugin-sdk/context-visibility-runtime` | 2026 年 7 月後為私有本機項目；不廣泛匯入設定／安全性功能的情境可見性解析及補充情境篩選 |
    | `plugin-sdk/string-coerce-runtime` | 不匯入 Markdown／記錄功能的精簡基礎記錄／字串強制轉型及正規化輔助函式 |
    | `plugin-sdk/html-entity-runtime` | 2026 年 7 月後為私有本機項目；不使用廣泛文字公用程式的單次處理、分號終止 HTML5 實體解碼 |
    | `plugin-sdk/text-utility-runtime` | 2026 年 7 月後為私有本機項目；低階文字與路徑輔助工具，包括五實體 HTML 跳脫 |
    | `plugin-sdk/widget-html` | 自含式 HTML 小工具的完整文件偵測、大小驗證，以及工具輸入錯誤 |
    | `plugin-sdk/host-runtime` | 2026 年 7 月後為私有本機項目；主機名稱與 SCP 主機正規化輔助工具 |
    | `plugin-sdk/retry-runtime` | 2026 年 7 月後為私有本機項目；重試設定與重試執行器輔助工具 |
    | `plugin-sdk/agent-runtime` | 已棄用的廣泛彙整匯出，適用於代理程式目錄／身分／工作區輔助工具，包括 `resolveAgentDir`、`resolveDefaultAgentDir`，以及已棄用的 `resolveOpenClawAgentDir` 相容性匯出；建議優先使用專用的代理程式／執行階段子路徑 |
    | `plugin-sdk/directory-runtime` | 設定支援的目錄查詢／去重 |
    | `plugin-sdk/keyed-async-queue` | 2026 年 7 月後為私有本機項目；`KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="功能與測試子路徑">
    | 子路徑 | 主要匯出項目 |
    | --- | --- |
    | `plugin-sdk/media-runtime` | 已淘汰的廣泛媒體彙整匯出，包含 `saveRemoteMedia`、`saveResponseMedia`、`readRemoteMediaBuffer`，以及已淘汰的 `fetchRemoteMedia`；請優先使用 `plugin-sdk/media-store`、`plugin-sdk/media-mime`、`plugin-sdk/outbound-media` 與功能執行階段子路徑；若 URL 應轉為 OpenClaw 媒體，請先使用儲存區輔助函式，再讀取緩衝區 |
    | `plugin-sdk/media-local-roots` | 供外掛所擁有的本機媒體讀取使用，聚焦於 `getAgentScopedMediaLocalRoots(...)` 且可感知原則的 `getAgentScopedMediaLocalRootsForSources(...)` 輔助函式 |
    | `plugin-sdk/media-mime` | 範圍精簡的 MIME 正規化、副檔名對應、MIME 偵測及媒體種類輔助函式 |
    | `plugin-sdk/media-store` | 範圍精簡的媒體儲存區輔助函式，例如 `saveMediaBuffer` 與 `saveMediaStream` |
    | `plugin-sdk/media-generation-runtime` | 2026 年 7 月後僅供內部使用；共用的媒體生成容錯移轉輔助函式、候選項目選擇及缺少模型時的訊息 |
    | `plugin-sdk/media-understanding` | 媒體理解供應商型別與輔助函式的已淘汰相容性外觀；新供應商透過注入的外掛 API 註冊，並由外掛擁有請求輔助函式 |
    | `plugin-sdk/text-chunking` | 對外文字與保留偏移量的範圍分塊、Markdown 分塊／算繪輔助函式、可感知引述的 HTML 標籤詞元化、Markdown 表格轉換、指示標籤移除及安全文字公用程式 |
    | `plugin-sdk/speech` | 2026 年 7 月後僅供內部使用；語音供應商型別，以及供應商端的指示、登錄庫、驗證、OpenAI 相容 TTS 建構器與語音輔助函式匯出項目 |
    | `plugin-sdk/speech-core` | 2026 年 7 月後僅供內部使用；共用的語音供應商型別、登錄庫、指示、正規化及語音輔助函式匯出項目 |
    | `plugin-sdk/speech-settings` | 不含供應商登錄庫或合成執行階段的輕量 TTS 設定解析與正規化基礎元件 |
    | `plugin-sdk/realtime-transcription` | 2026 年 7 月後僅供內部使用；即時轉錄供應商型別、登錄庫輔助函式及共用 WebSocket 工作階段輔助函式 |
    | `plugin-sdk/realtime-bootstrap-context` | 2026 年 7 月後僅供內部使用；用於有限範圍 `IDENTITY.md`、`USER.md` 與 `SOUL.md` 情境注入的即時設定檔啟動輔助函式 |
    | `plugin-sdk/realtime-voice` | 2026 年 7 月後僅供內部使用；即時語音供應商型別、登錄庫輔助函式、共用音訊能量／語音開始閘門及即時語音行為輔助函式，包括與傳輸無關的工作階段測試框架及輸出活動追蹤 |
    | `plugin-sdk/meeting-runtime` | 瀏覽器會議工作階段執行階段、即時音訊引擎／傳輸、`MeetingPlatformAdapter`、瀏覽器／節點控制、代理程式諮詢、語音通話委派、設定檢查及 SoX 命令輔助函式 |
    | `plugin-sdk/image-generation` | 2026 年 7 月後僅供內部使用；圖像生成供應商型別，以及圖像資產／資料 URL 輔助函式與 OpenAI 相容圖像供應商建構器 |
    | `plugin-sdk/image-generation-core` | 2026 年 7 月後僅供內部使用；共用的圖像生成型別、容錯移轉、驗證及登錄庫輔助函式 |
    | `plugin-sdk/music-generation` | 2026 年 7 月後僅供內部使用；音樂生成供應商／請求／結果型別 |
    | `plugin-sdk/video-generation` | 2026 年 7 月後僅供內部使用；影片生成供應商／請求／結果型別 |
    | `plugin-sdk/video-generation-core` | 2026 年 7 月後僅供內部使用；共用的影片生成型別、容錯移轉輔助函式、供應商查詢及模型參照解析 |
    | `plugin-sdk/transcripts` | 2026 年 7 月後僅供內部使用；共用的轉錄來源供應商型別、登錄庫輔助函式、會議供應商橋接工廠、工作階段描述元及發言中繼資料 |
    | `plugin-sdk/webhook-targets` | 2026 年 7 月後僅供內部使用；網路鉤子目標登錄庫及路由安裝輔助函式 |
    | `plugin-sdk/web-media` | 共用的遠端／本機媒體載入輔助函式 |
    | `plugin-sdk/zod` | 已淘汰的相容性重新匯出；請直接從 `zod` 匯入 `zod` |
    | `plugin-sdk/plugin-test-api` | 僅供儲存庫內使用的最小型 `createTestPluginApi` 輔助函式，用於直接外掛註冊單元測試，無須匯入儲存庫測試輔助橋接 |
    | `plugin-sdk/agent-runtime-test-contracts` | 僅供儲存庫內使用的原生代理程式執行階段轉接器合約固定資料，用於驗證、傳遞、備援、工具掛鉤、提示詞覆疊、結構描述及轉錄投影測試 |
    | `plugin-sdk/channel-test-helpers` | 僅供儲存庫內使用、面向頻道的測試輔助函式，用於通用動作／設定／狀態合約、目錄斷言、帳戶啟動生命週期、傳送設定串接、執行階段模擬、狀態問題、對外傳遞及掛鉤註冊 |
    | `plugin-sdk/channel-target-testing` | 僅供儲存庫內使用的共用目標解析錯誤案例套件，用於頻道測試 |
    | `plugin-sdk/channel-contract-testing` | 僅供儲存庫內使用、範圍精簡的頻道合約測試輔助函式，不含廣泛的測試彙整匯出 |
    | `plugin-sdk/plugin-test-contracts` | 僅供儲存庫內使用的外掛套件、註冊、公開成品、直接匯入、執行階段 API 及匯入副作用合約輔助函式 |
    | `plugin-sdk/plugin-state-test-runtime` | 僅供儲存庫內使用的外掛狀態儲存區、輸入佇列及狀態資料庫測試輔助函式 |
    | `plugin-sdk/provider-test-contracts` | 僅供儲存庫內使用的供應商執行階段、驗證、探索、上線、目錄、精靈、媒體功能、重播原則、即時 STT 實況音訊、網頁搜尋／擷取及串流合約輔助函式 |
    | `plugin-sdk/provider-http-test-mocks` | 2026 年 7 月後僅供內部使用；僅供儲存庫內使用、選擇性啟用的 Vitest HTTP／驗證模擬，用於會執行 `plugin-sdk/provider-http` 的供應商測試 |
    | `plugin-sdk/reply-payload-testing` | 僅供儲存庫內使用的輔助函式，用於將中繼資料附加至回覆承載固定資料 |
    | `plugin-sdk/sqlite-runtime-testing` | 僅供儲存庫內第一方測試使用的 SQLite 生命週期輔助函式 |
    | `plugin-sdk/test-fixtures` | 僅供儲存庫內使用的通用命令列介面執行階段擷取、沙箱情境、技能寫入器、代理程式訊息、系統事件、模組重新載入、隨附外掛路徑、終端文字、分塊、驗證權杖及具型別案例固定資料 |
    | `plugin-sdk/test-node-mocks` | 僅供儲存庫內使用、聚焦於 Node 內建模組的模擬輔助函式，用於 Vitest `vi.mock("node:*")` 工廠內部 |
  </Accordion>

  <Accordion title="記憶體子路徑">
    | 子路徑 | 主要匯出項目 |
    | --- | --- |
    | `plugin-sdk/memory-core-host-embedding-registry` | 2026 年 7 月後僅供內部使用；輕量記憶體嵌入供應商登錄庫輔助函式 |
    | `plugin-sdk/memory-core-host-engine-foundation` | 記憶體主機基礎引擎匯出項目 |
    | `plugin-sdk/memory-core-host-engine-embeddings` | 2026 年 7 月後僅供內部使用；記憶體主機嵌入合約、登錄庫存取、本機供應商及通用批次／遠端輔助函式。此介面上的 `registerMemoryEmbeddingProvider` 已淘汰；新供應商請使用通用嵌入供應商 API。 |
    | `plugin-sdk/memory-core-host-engine-qmd` | 2026 年 7 月後僅供內部使用；記憶體主機 QMD 引擎匯出項目 |
    | `plugin-sdk/memory-core-host-engine-storage` | 2026 年 7 月後僅供內部使用；記憶體主機儲存引擎匯出項目 |
    | `plugin-sdk/memory-core-host-secret` | 2026 年 7 月後僅供內部使用；記憶體主機密鑰輔助函式 |
    | `plugin-sdk/memory-core-host-status` | 2026 年 7 月後僅供內部使用；記憶體主機狀態輔助函式 |
    | `plugin-sdk/memory-core-host-runtime-cli` | 2026 年 7 月後僅供內部使用；記憶體主機命令列介面執行階段輔助函式 |
    | `plugin-sdk/memory-core-host-runtime-core` | 2026 年 7 月後僅供內部使用；記憶體主機核心執行階段輔助函式 |
    | `plugin-sdk/memory-core-host-runtime-files` | 2026 年 7 月後僅供內部使用；記憶體主機檔案／執行階段輔助函式 |
    | `plugin-sdk/memory-host-core` | 廠商中立記憶體主機輔助函式的已淘汰相容性外觀。新的記憶體外掛使用注入的記憶體功能及主機準備的提示詞；在提供聚焦的讀取介面之前，配套外掛仍會使用保留的外觀進行公開成品探索。 |
    | `plugin-sdk/memory-host-events` | 2026 年 7 月後僅供內部使用；記憶體主機事件日誌輔助函式的廠商中立別名 |
    | `plugin-sdk/memory-host-markdown` | 2026 年 7 月後僅供內部使用；供記憶體相關外掛使用的共用受管理 Markdown 輔助函式 |
    | `plugin-sdk/memory-host-search` | 2026 年 7 月後僅供內部使用；用於存取搜尋管理器的主動記憶執行階段外觀 |
  </Accordion>

  <Accordion title="保留的隨附輔助函式子路徑">
    保留的隨附輔助函式 SDK 子路徑，是供隨附外掛程式碼使用、範圍精簡且由特定擁有者管理的介面。這些子路徑會列入 SDK 清單中追蹤，讓套件建置與別名處理維持確定性，但它們不是通用的外掛開發 API。新的可重用主機合約應使用通用 SDK 子路徑，例如 `plugin-sdk/gateway-runtime` 與 `plugin-sdk/ssrf-runtime`。

    | 子路徑 | 擁有者與用途 |
    | --- | --- |
    | `plugin-sdk/codex-mcp-projection` | 2026 年 7 月後僅供內部使用；隨附 Codex 外掛輔助函式，用於將使用者 MCP 伺服器設定投影至 Codex 應用程式伺服器執行緒設定（保留的套件匯出項目） |
    | `plugin-sdk/codex-native-task-runtime` | 隨附 Codex 外掛輔助函式，用於將 Codex 應用程式伺服器原生子代理程式鏡像至 OpenClaw 任務狀態（僅供儲存庫內使用，不是套件匯出項目） |

  </Accordion>
</AccordionGroup>

## 相關內容

- [外掛 SDK 概觀](/zh-TW/plugins/sdk-overview)
- [外掛 SDK 設定](/zh-TW/plugins/sdk-setup)
- [建置外掛](/zh-TW/plugins/building-plugins)
