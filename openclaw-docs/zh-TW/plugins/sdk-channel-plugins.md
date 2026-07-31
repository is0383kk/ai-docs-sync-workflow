---
read_when:
    - 你正在建立新的訊息通道外掛
    - 你想要將 OpenClaw 連接至訊息平台
    - 你需要瞭解 ChannelPlugin 介面卡介面範圍
sidebarTitle: Channel Plugins
summary: 建立 OpenClaw 訊息通道外掛的逐步指南
title: 建置頻道外掛
x-i18n:
    generated_at: "2026-07-26T08:30:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ff8ad04346babf3eece7e10bd38946ee290947b2e504b6b5ca438865531bf38
    source_path: plugins/sdk-channel-plugins.md
    workflow: 16
---

本指南將建立一個把 OpenClaw 連接至訊息平台的頻道外掛，涵蓋私訊安全性、配對、回覆討論串與對外傳訊。

<Info>
  第一次使用 OpenClaw 外掛？請先閱讀[入門指南](/zh-TW/plugins/building-plugins)，
  了解套件結構與資訊清單設定。
</Info>

## 你的外掛負責的項目

頻道外掛不會實作傳送、編輯或回應工具；核心提供一個
共用的 `message` 工具。你的外掛負責：

- **設定** - 帳號解析與設定精靈
- **安全性** - 私訊政策與允許清單
- **配對** - 私訊核准流程
- **工作階段文法** - 供應商特定的對話 ID 如何對應至基礎
  聊天、討論串 ID 與父層備援
- **對外傳送** - 將文字、媒體與投票傳送至平台
- **討論串** - 回覆如何串成討論串
- **心跳偵測輸入狀態** - 為心跳偵測傳遞目標提供選用的輸入中／忙碌訊號

核心負責共用訊息工具、提示詞接線、外層工作階段金鑰格式、
通用 `:thread:` 簿記與分派。

## 訊息轉接器

公開一個具備 `defineChannelMessageAdapter` 的 `message` 轉接器，其來源為
`openclaw/plugin-sdk/channel-outbound`。僅宣告原生傳輸實際支援的持久最終傳送
能力，並以合約測試證明原生副作用與傳回的收據。將文字／媒體
傳送指向舊版 `outbound` 轉接器所使用的相同傳輸函式。如需
完整 API 合約、能力矩陣、收據規則、即時預覽
最終化、接收確認政策、測試與遷移表，請參閱
[頻道對外傳送 API](/zh-TW/plugins/sdk-channel-outbound)。

如果現有的 `outbound` 轉接器已具備正確的傳送方法與
能力中繼資料，請使用 `createChannelMessageAdapterFromOutbound(...)` 衍生
`message` 轉接器，而不要手寫另一個
橋接層。轉接器傳送會傳回 `MessageReceipt` 值。對於舊版 ID，請使用
`listMessageReceiptPlatformIds(...)` 或
`resolveMessageReceiptPrimaryId(...)` 衍生，而不要保留平行的 `messageIds`
欄位。

請精確宣告即時與最終化工具能力——核心會使用這些資訊判斷
頻道能執行哪些操作，而宣告與實際行為之間的偏差會導致
合約測試失敗：

| 介面                                  | 值                                                                                               |
| ------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `message.live.capabilities`           | `draftPreview`, `previewFinalization`, `progressUpdates`, `nativeStreaming`, `quietFinalization` |
| `message.live.finalizer.capabilities` | `finalEdit`, `normalFallback`, `discardPending`, `previewReceipt`, `retainOnAmbiguousFailure`    |

會原地完成草稿預覽的頻道，應透過 `defineFinalizableLivePreviewAdapter(...)` 加上
`deliverWithFinalizableLivePreviewAdapter(...)` 路由執行階段邏輯，並讓宣告的
能力持續由 `verifyChannelMessageLiveCapabilityAdapterProofs(...)`
與 `verifyChannelMessageLiveFinalizerProofs(...)` 測試支援，確保原生預覽、
進度、編輯、備援／保留、清理與收據行為不會在未察覺的情況下
產生偏差。

延後平台確認的輸入接收器應宣告
`message.receive.defaultAckPolicy` 與 `supportedAckPolicies`，而不是把
確認時機隱藏在監控器的區域狀態中。請使用
`verifyChannelMessageReceiveAckPolicyAdapterProofs(...)` 涵蓋每項宣告的政策。

`dispatchInboundReplyWithBase` 與
`recordInboundSessionAndDispatchReply` 等舊版回覆輔助函式仍可供相容性
分派器使用。不要在新的頻道程式碼中使用它們；請改從 `message`
轉接器、收據，以及 `openclaw/plugin-sdk/channel-outbound` 上的接收／傳送生命週期輔助函式開始。

### 輸入進入點（實驗性）

正在遷移輸入授權的頻道，可從執行階段接收
路徑使用實驗性的 `openclaw/plugin-sdk/channel-ingress-runtime` 子路徑。
它接受平台事實、原始允許清單、路由描述元、命令
事實與存取群組設定，接著傳回傳送者／路由／命令／啟用
投影以及依序排列的進入圖，而平台查詢與副作用則留在外掛中。
請在傳給解析器的描述元中保留外掛身分正規化；不要序列化
已解析狀態或決策中的原始比對值。API 設計、
責任邊界與測試預期請參閱
[頻道輸入 API](/zh-TW/plugins/sdk-channel-ingress)。

### 持久輸入與重播去重

採用持久輸入的頻道應使用來自 `openclaw/plugin-sdk/channel-outbound` 的
`createChannelIngressMonitor`，除非需要本質上不同的
准入或抽取合約。請在單一接收瓶頸點將原始傳輸封套加入佇列
（接收時不進行正規化）；對於網路鉤子傳輸，應以持久附加結果控制
傳輸確認；每個對話衍生一條序列化通道，並在分派
接納時將事件標記為完成。佇列的主鍵為 `(queue_name, event_id)`，且完成時
會將資料列標記為墓碑而非刪除，因此平台稍後再次傳遞相同的
`event_id` 時，在墓碑保留期間內會被持久拒絕。
如需監控器 API 與關閉合約，請參閱[頻道對外傳送 API](/zh-TW/plugins/sdk-channel-outbound#durable-ingress-monitors)。

該墓碑是重播防護
（`openclaw/plugin-sdk/persistent-dedupe`）的分層規則：已排空的頻道只有在防護的身分或保留期間超出佇列時，
才會保留獨立的重播防護——例如與傳輸遞送 ID 不同的邏輯訊息金鑰（Telegram
會對 `chat_id:message_id` 去重，因為防彈跳合併可能會讓訊息以新的
`update_id` 再次出現），或保留期間長於頻道的墓碑
保留期。如果防護金鑰會等同於排空作業的 `event_id`，採用排空作業時請刪除該
防護，並調整 `completedTtlMs`/`completedMaxEntries` 的大小，
改為涵蓋舊有防護期間。年齡
界線等非去重保護與此規則無關。穩定的對外訊息 ID 應使用
`openclaw/plugin-sdk/channel-outbound` 的共用對外回音登錄，而不是
頻道區域的 TTL 快取。

#### 傳輸類別與保留

請依接收邊界的復原保證為傳輸分類：

- **由確認控制的網路鉤子或事件遞送：**僅在
  持久附加完成後才確認或傳回成功。附加失敗時，必須讓該遞送仍可
  重試，或讓接收邊界失敗。此類別包括 Slack、SMS、Zalo、
  Microsoft Teams、Google Chat、LINE 與 Synology Chat。
- **等待式輪詢或串流遞送：**僅在附加完成後才推進遠端游標或傳送
  傳輸確認。如果沒有明確的游標，請維持接收回呼的序列化並等待其完成，確保附加失敗時
  接收迴圈不會向前超跑。Telegram 輪詢、Signal 與 Tlon 使用此類別；
  Telegram 網路鉤子遞送則遵循上述由確認控制的規則。
- **不可重播的通訊端：**IRC、Mattermost、Twitch 與 Zalo Personal 無法要求
  平台重新遞送已接受的事件。其持久佇列可保護
  程序當機期間並支援本機重新啟動復原；完成
  墓碑對平台重播幾乎不起作用。

請將 30 天作為整體部署的墓碑 TTL 慣例，而不是 SDK 預設值。高流量
重新遞送期間通常使用 20,000 筆已完成項目上限；
低流量的等待式與不可重播傳輸通常使用 1,000-2,000。
目前的例外包括 LINE 的 4,096 筆上限、SMS 的 24 小時已完成
TTL，以及 Tlon 僅以上限控制的已完成保留。失敗資料列的上限也可能低於
已完成項目的上限。TTL 與上限都會修剪資料列，因此有效保留期會在
先達到任一界限時結束。只有在有文件記載的平台重試
期限、需保留的已發布重播防護期間、預期流量或磁碟預算，
或不可重播傳輸的情況下才能偏離，並應以測試涵蓋保留合約。

#### 至少一次副作用

排空分派會在輸入資料列成為完成
墓碑前執行命令副作用。如果程序在這兩個步驟之間當機，資料列會重播，
且可能再次執行副作用。這個至少一次的當機期間是
預設合約。對於設定寫入、儲存空間
清除，或回覆通道之外的可見確認等非等冪工作，請使用來自
`openclaw/plugin-sdk/ingress-effect-once` 的 `createIngressEffectOnce(...)`。
每次呼叫都應提供穩定的輸入
`eventId` 與效果名稱。每個輸入佇列／帳號建立一個輔助函式，並為該範圍
使用穩定且唯一的 `namespacePrefix`，因為傳輸事件
ID 可能僅在佇列內唯一。輔助函式只會在效果成功後提交其持久
宣告；效果擲回錯誤時會釋放宣告，讓排空作業重試時可再次
執行，而同時呼叫者則等待作用中的宣告。若有提供
`onDiskError`，持久狀態錯誤會呼叫它並拒絕作業，而不是退回
程序記憶體。

將輔助函式的 `ttlMs` 設為至少等於頻道的輸入墓碑保留期，
再加上效果提交與資料列完成之間的最長延遲，包括
有界限的停機時間與排空重試。效果記錄的 TTL 從提交時開始，
而墓碑保留期稍後才從完成時開始；如果待處理資料列的存留時間
沒有上限，任何有限 TTL 都無法涵蓋任意長度的停機時間。當墓碑已
無法再重播資料列後，更舊的效果記錄便是無用負擔。請調整
`stateMaxEntries` 的大小，以容納該保留期間內可能存在的每個不同事件／效果金鑰，
並考量佇列的已完成項目上限與每個事件的
效果數量上限。較低的上限會在記錄 TTL 到期前逐出最舊記錄，
使該效果可能再次執行。如果程序在效果成功後、宣告提交前
終止，或持久化失敗，或者記錄在其輸入資料列仍
待處理時到期，仍會留下至少一次的執行期間。

#### 帳號範圍重新啟動合約

頻道設定變更預設會重新啟動整個頻道。多帳號
頻道只有在設定解析僅讀取頻道層級共用欄位與選定帳號，
絕不讀取同層其他帳號，且閘道可以停止並啟動單一 `(channel, accountId)`
執行階段而不替換同層其他執行階段時，才可設定 `reload.accountScopedRestart: true`。

此範圍路徑僅適用於
`channels.<channel>.accounts.<non-default-id>.*` 下的變更。共用頻道
欄位、`accounts.default`、已移除或無法解析的帳號，以及可能影響繼承的混合變更，
都會提升為整個頻道重新啟動。未選擇採用此功能的外掛
一律使用整個頻道路徑。

對於使用持久輸入排空作業的頻道，帳號監控器的停止路徑
必須先完成所有已接受的傳輸准入，再處置並等待其
排空作業。啟動帳號時會開啟相同的帳號金鑰佇列，其初始
排空作業會復原尚未分派的持久資料列。不要新增第二次重新載入專用的
重播階段；佇列復原才是標準重新啟動路徑。

請將此旗標視為能力宣告，而非效能偏好。合約
測試應證明新增與編輯一個具名帳號時，同層其他帳號的
已解析設定維持不變；停止一個帳號時，只會完成該帳號的
監控器與排空作業；而全新的監控器只會復原該帳號的資料列
一次。如果無法證明任何一項保證，請省略此旗標。

### 輸入狀態指示器

如果頻道支援輸入回覆以外的輸入狀態指示器，請在頻道外掛上公開
`heartbeat.sendTyping(...)`。核心會在心跳偵測模型執行開始前，
以已解析的心跳偵測傳遞目標呼叫它，並使用共用的輸入狀態保持運作／清理生命週期。
若平台需要明確的停止訊號，請新增
`heartbeat.clearTyping(...)`。

### 媒體來源參數

如果頻道新增了攜帶媒體來源的訊息工具參數，請透過
`plugin.actions.describeMessageTool(...).mediaSourceParams` 公開這些參數名稱。
核心會使用該明確清單進行沙箱路徑正規化與對外
媒體存取政策處理，因此外掛不需要為供應商特定的頭像、附件或封面圖片參數
在共用核心加入特殊處理。

偏好使用以動作為鍵的對應表，例如 `{ "set-profile": ["avatarUrl", "avatarPath"] }`，
如此不相關的動作便不會繼承其他動作的媒體引數。若參數是刻意在每個公開動作之間共用，
仍可使用扁平陣列。

必須公開暫時性公用 URL 以供平台端擷取媒體的頻道，
可搭配外掛狀態儲存區使用來自
`openclaw/plugin-sdk/outbound-media` 的 `createHostedOutboundMediaStore(...)`。請將平台
路由剖析與權杖強制檢查保留在頻道外掛中；共用輔助工具
僅負責媒體載入、到期中繼資料、分塊資料列與清理。

輸入附件使用有序事實，而非平行的 `Media*` 欄位。請使用來自
`openclaw/plugin-sdk/channel-inbound` 的 `toInboundMediaFacts(...)` 正規化
頻道記錄，並在建立輸入內容時將其作為 `media` 傳入。當外掛必須授權讀取本機媒體時，請從專用的
`openclaw/plugin-sdk/media-local-roots` 子路徑匯入
`getAgentScopedMediaLocalRoots(...)` 或
`getAgentScopedMediaLocalRootsForSources(...)`。舊版
`agent-media-payload` 建構器／根外觀已棄用，僅供相容性使用。

### 原生承載資料塑形

若你的頻道需要針對 `message(action="send")` 進行供應商特定的塑形，
請優先使用 `actions.prepareSendPayload(...)`。將原生卡片、區塊、嵌入內容或
其他持久資料放在 `payload.channelData.<channel>` 下，並讓核心透過輸出／訊息介面卡傳送。
僅將 `actions.handleAction(...)` 用於傳送無法序列化及重試之承載資料的相容性備援。

### 工作階段對話文法

若你的平台在對話 ID 內儲存額外範圍，請在外掛中使用
`messaging.resolveSessionConversation(...)` 保留該剖析邏輯。這是將
`rawId` 對應至基礎對話 ID、選用討論串 ID、明確的
`baseConversationId`，以及任何
`parentConversationCandidates` 的標準掛鉤。傳回 `parentConversationCandidates` 時，
請依照從範圍最窄的上層到範圍最廣／基礎對話的順序排列。

`messaging.resolveParentConversationCandidates(...)` 是已棄用的
相容性備援，適用於只需在通用／原始 ID 之上提供上層備援的外掛。
若兩個掛鉤皆存在，核心會先使用
`resolveSessionConversation(...).parentConversationCandidates`，只有在標準掛鉤省略這些項目時，
才會改用 `resolveParentConversationCandidates(...)`。

在頻道登錄檔啟動前需要相同剖析邏輯的內建外掛，
可公開頂層 `session-key-api.ts` 檔案，並提供相符的
`resolveSessionConversation(...)` 匯出（請參閱 Feishu 與 Telegram
外掛）。只有在執行階段外掛登錄檔尚不可用時，核心才會使用這個可安全啟動的介面。

當外掛程式碼需要正規化類路由欄位、比較子討論串與其上層路由，或從
`{ channel, to, accountId, threadId }` 建立穩定的重複資料刪除鍵時，請使用
`openclaw/plugin-sdk/channel-route`。此輔助工具正規化數字討論串 ID 的方式與核心相同，
因此請優先使用它，而非臨時的 `String(threadId)` 比較。
具有供應商特定目標文法的外掛應公開 `messaging.resolveOutboundSessionRoute(...)`，
讓核心不必透過剖析器相容層，即可取得供應商原生的工作階段與討論串身分。

### 帳號範圍的對話繫結支援

當頻道支援通用的目前對話繫結時，請設定
`conversationBindings.supportsCurrentConversationBinding`。`createChatChannelPlugin(...)`
預設會將此靜態功能設為 `true`。

若支援情況會因已設定的帳號而異，另請實作
`conversationBindings.isCurrentConversationBindingSupported({ accountId })`。
核心只有在啟用靜態功能後，才會評估此同步掛鉤。傳回
`false` 會讓該帳號無法使用通用的目前對話功能、繫結、查詢、列出、更新存取時間及解除繫結操作。
省略此掛鉤則會將靜態功能套用至每個帳號。

請從已載入的帳號設定或執行階段狀態解析答案。此掛鉤只會管控通用的目前對話繫結；
不會取代已設定的繫結規則或外掛自有的工作階段路由。
契約測試至少應透過
`openclaw/plugin-sdk/channel-core` 匯出的 `ChannelPlugin["conversationBindings"]` 契約，
涵蓋一個受支援及一個不受支援的帳號。

## 核准與頻道功能

大多數頻道外掛不需要核准專用程式碼。核心負責同一聊天室的
`/approve`、共用核准按鈕承載資料，以及通用備援傳遞。
`ChannelPlugin.approvals` 已移除；請改將核准傳遞／原生／呈現／驗證
事實放在單一 `approvalCapability` 物件上。`plugin.auth` 僅供登入／登出使用，
核心不再從該物件讀取核准驗證掛鉤。

僅針對原生核准路由或抑制備援使用 `approvalCapability.delivery`；
只有在頻道確實需要自訂核准承載資料，而非共用呈現器時，才使用 `approvalCapability.render`。

### 核准驗證

- `approvalCapability.authorizeActorAction` 與
  `approvalCapability.getActionAvailabilityState` 是標準的
  核准驗證介面。
- 使用 `getActionAvailabilityState` 表示同一聊天室的核准驗證可用性。
  即使原生傳遞已停用，也要讓已設定的核准者仍可供 `/approve` 使用；
  請改用原生發起介面狀態提供傳遞／設定指引。
- 若你的頻道公開原生執行核准，當發起介面／原生用戶端狀態
  與同一聊天室的核准驗證不同時，請使用
  `approvalCapability.getExecInitiatingSurfaceState`。核心會使用該執行專用掛鉤區分
  `enabled` 與 `disabled`、判斷發起頻道是否支援原生執行核准，
  並將該頻道納入原生用戶端備援指引。
  在一般情況下，`createApproverRestrictedNativeApprovalCapability(...)` 會填入此資訊。
- 若頻道可從現有設定推斷穩定、類似擁有者的私訊身分，
  請使用來自
  `openclaw/plugin-sdk/approval-runtime` 的 `createResolvedApproverActionAuthAdapter` 限制同一聊天室的
  `/approve`，而不必新增核准專用核心邏輯。
- 若自訂核准驗證刻意只允許同一聊天室備援，請從
  `openclaw/plugin-sdk/approval-auth-runtime` 傳回
  `markImplicitSameChatApprovalAuthorization({ authorized: true })`；否則核心會將結果視為明確的核准者授權。
- 若頻道自有的原生回呼會直接解析核准，請在解析前使用
  `isImplicitSameChatApprovalAuthorization(...)`，如此隱含備援仍會經過頻道的一般動作者授權。

### 承載資料生命週期與設定指引

- 針對頻道特定的承載資料生命週期行為，例如隱藏重複的本機核准提示，
  或在傳遞前傳送輸入中指示器，請使用 `outbound.shouldSuppressLocalPayloadPrompt` 或
  `outbound.beforeDeliverPayload`。
- 當頻道希望在停用路徑的回覆中說明啟用原生執行核准
  所需的確切設定選項時，請使用 `approvalCapability.describeExecApprovalSetup`。
  此掛鉤接收 `{ channel, channelLabel, accountId }`；
  具名帳號頻道應呈現帳號範圍的路徑，例如
  `channels.<channel>.accounts.<id>.execApprovals.*`，而非頂層預設值。
- 當外掛核准失敗指引可安全顯示於外掛核准的無路由與逾時失敗時，
  請使用 `approvalCapability.describePluginApprovalSetup`。`createApproverRestrictedNativeApprovalCapability(...)`
  不會從 `describeExecApprovalSetup` 推斷此資訊；只有在外掛核准與執行核准
  確實使用相同原生設定時，才明確傳入相同的輔助工具。

### 原生核准傳遞

若頻道需要原生核准傳遞，請讓頻道程式碼專注於目標正規化，
以及傳輸／呈現事實。請使用來自
`openclaw/plugin-sdk/approval-runtime` 的
`createChannelExecApprovalProfile`、`createChannelNativeOriginTargetResolver`、
`createChannelApproverDmTargetResolver` 與
`createApproverRestrictedNativeApprovalCapability`。將頻道特定事實置於
`approvalCapability.nativeRuntime` 後方，最好透過
`createChannelApprovalNativeRuntimeAdapter(...)` 或
`createLazyChannelApprovalNativeRuntimeAdapter(...)`，如此核心即可組裝處理常式，並負責請求篩選、路由、重複資料刪除、到期、閘道訂閱，
以及已路由至其他位置的通知。

`nativeRuntime` 已拆分為幾個較小的介面：

- `availability` - 帳號是否已設定，以及是否應處理請求
- `presentation` - 將共用核准檢視模型對應至
  待處理／已解析／已到期的原生承載資料或最終動作
- `transport` - 準備目標，並傳送／更新／刪除原生核准訊息
- `interactions` - 原生按鈕或回應的選用繫結／解除繫結／清除動作掛鉤，
  以及選用的 `cancelDelivered` 掛鉤。當 `deliverPending`
  登錄程序內或持久狀態（例如回應目標儲存區）時，請實作
  `cancelDelivered`，如此若處理常式停止導致傳遞在
  `bindPending` 執行前取消，或
  `bindPending` 未傳回控制代碼時，便可釋放該狀態
- `observe` - 選用的傳遞診斷掛鉤

其他核准輔助工具：

- 當頻道同時支援源自工作階段的原生傳遞，以及明確的核准轉送目標時，
  請使用來自
  `openclaw/plugin-sdk/approval-native-runtime` 的 `createNativeApprovalChannelRouteGates`。此輔助工具集中處理核准設定選擇、
  `mode` 處理、代理程式／工作階段篩選器、帳號繫結、工作階段目標比對與目標清單比對，
  而呼叫端仍負責頻道 ID、預設轉送模式、帳號查詢、傳輸啟用檢查、目標正規化，
  以及回合來源目標解析。請勿使用它建立核心自有的頻道原則預設值；
  請明確傳入頻道記載的預設模式。
- `createChannelNativeOriginTargetResolver` 預設會針對 `{ to, accountId, threadId }`
  目標使用共用頻道路由比對器。只有在頻道具有供應商特定的等價規則時，
  才傳入 `targetsMatch`，例如 Slack 時間戳記前綴比對。
  當頻道需要在預設路由比對器或自訂 `targetsMatch` 回呼執行前，
  將供應商 ID 標準化，同時保留原始目標以供傳遞時，請傳入
  `normalizeTargetForMatch`。只有在解析出的傳遞目標本身應標準化時，才使用
  `normalizeTarget`。
- 若頻道需要由執行階段擁有的物件，例如用戶端、權杖、Bolt
  應用程式或網路鉤子接收器，請透過
  `openclaw/plugin-sdk/channel-runtime-context` 登錄。通用執行階段內容登錄檔
  讓核心可從頻道啟動狀態啟動由功能驅動的處理常式，而不必新增核准專用包裝黏合程式碼。
- 只有在功能驅動的介面表達能力仍不足時，才使用較低階的
  `createChannelApprovalHandler` 或
  `createChannelNativeApprovalRuntime`。
- 原生核准頻道必須透過這些輔助工具路由
  `accountId` 與 `approvalKind`。
  `accountId` 讓多帳號核准原則限定於正確的機器人帳號，
  而 `approvalKind` 讓頻道仍可使用執行核准與外掛核准的行為，
  不必在核心中加入硬式編碼的分支。
- 核心也負責核准重新路由通知。頻道外掛不應從
  `createChannelNativeApprovalRuntime` 傳送自己的“核准已前往私訊／其他頻道”後續訊息；
  請改透過共用核准功能輔助工具公開準確的來源與核准者私訊路由，
  並讓核心彙總實際傳遞後，再將任何通知傳回發起聊天室。
- 請端對端保留已傳遞的核准 ID 種類。原生用戶端不應根據頻道本機狀態，
  猜測或重寫執行核准與外掛核准的路由。
- 將該明確的 `approvalKind` 傳入 `resolveApprovalOverGateway`。
  這會使用標準的 `approval.resolve` 服務，並在其他介面先回應時傳回已記錄的勝出者。
  較舊的明確 `resolveMethod` 輸入仍保留供命令支援的控制項使用；
  新的原生動作不得使用它，也不得從 ID 推斷種類。
- 不同核准種類可刻意公開不同的原生介面。目前的內建範例：
  Matrix 對執行核准與外掛核准維持相同的原生私訊／頻道路由與回應使用者體驗，
  同時仍允許驗證方式依核准種類而異；Slack 則讓執行核准與外掛核准 ID
  都可使用原生核准路由。
- `createApproverRestrictedNativeApprovalAdapter` 仍作為
  相容性包裝器存在，但新程式碼應優先使用功能建構器，
  並在外掛上公開 `approvalCapability`。

### 範圍較窄的核准執行階段子路徑

對於頻繁使用的頻道進入點，若只需要該系列的一部分，
請優先使用以下範圍較窄的子路徑，而非較廣泛的
`approval-runtime` 匯出集合：

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-reference-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

同樣地，當你不需要所有功能時，請優先使用 `openclaw/plugin-sdk/reply-runtime`、
`openclaw/plugin-sdk/reply-dispatch-runtime`、
`openclaw/plugin-sdk/reply-reference` 和
`openclaw/plugin-sdk/reply-chunking`，而不是涵蓋範圍更廣的整合介面。

### 設定子路徑

- `openclaw/plugin-sdk/setup-runtime` 涵蓋可安全用於執行階段的設定輔助工具：
  `createSetupTranslator`、可安全匯入的設定修補轉接器
  （`createPatchedAccountSetupAdapter`、`createEnvPatchedAccountSetupAdapter`、
  `createSetupInputPresenceValidator`）、查詢備註輸出、
  `promptResolvedAllowFrom`、`splitSetupEntries`，以及委派的
  設定代理建構器。
- `openclaw/plugin-sdk/channel-setup` 涵蓋選用安裝的設定
  建構器以及一些可安全用於設定的基礎元件：`createOptionalChannelSetupSurface`、
  `createOptionalChannelSetupAdapter`、`createOptionalChannelSetupWizard`、
  `DEFAULT_ACCOUNT_ID`、`createTopLevelChannelDmPolicy`、
  `setSetupChannelEnabled` 和 `splitSetupEntries`。
- 只有當你也需要較繁重的共用設定／組態輔助工具（例如
  `moveSingleAccountChannelSectionToDefaultAccount(...)`）時，才使用範圍較廣的 `openclaw/plugin-sdk/setup` 介面。

如果你的頻道只想在設定介面中提示「請先安裝此外掛」，
請優先使用 `createOptionalChannelSetupSurface(...)`。產生的
轉接器／精靈會在寫入組態和完成設定時採取失敗即關閉策略，並在驗證、完成設定和文件連結
文案中重複使用相同的必要安裝訊息。

如果你的頻道支援由環境變數驅動的設定或驗證，請透過
頻道組態結構描述和設定描述項公開該功能。頻道執行階段的 `envVars` 或
本機常數只能用於面向操作人員的文案。

如果你的頻道可能在外掛執行階段啟動前出現在 `status`、`channels list`、`channels status` 或
SecretRef 掃描中，請在
`package.json` 加入 `openclaw.setupEntry`。此進入點應可安全匯入至唯讀命令
路徑，並應傳回這些摘要所需的頻道中繼資料、可安全用於設定的組態轉接器、
狀態轉接器及頻道祕密目標中繼資料。
請勿從設定進入點啟動用戶端、監聽器或傳輸執行階段。

主要頻道進入點的匯入路徑也應保持精簡。探索程序可以評估
該進入點和頻道外掛模組，以註冊功能，而不必
啟用頻道。`channel-plugin-api.ts` 等檔案應匯出
頻道外掛物件，而不要匯入設定精靈、傳輸
用戶端、通訊端監聽器、子程序啟動器或服務啟動模組。
請將這些執行階段元件放在由 `registerFull(...)` 載入的模組、執行階段
設定器或延遲載入的功能轉接器中。

### 其他精簡頻道子路徑

對於其他頻繁使用的頻道路徑，請優先使用精簡的輔助工具，而非範圍較廣的舊版
介面：

- `openclaw/plugin-sdk/account-core`、`openclaw/plugin-sdk/account-id`、
  `openclaw/plugin-sdk/account-resolution` 和
  `openclaw/plugin-sdk/account-helpers`，用於多帳號組態和
  預設帳號備援
- `openclaw/plugin-sdk/inbound-envelope` 和
  `openclaw/plugin-sdk/channel-inbound`，用於傳入路由／封裝及
  記錄並分派的接線
- `openclaw/plugin-sdk/channel-targets`，用於目標剖析輔助工具
- `openclaw/plugin-sdk/channel-outbound`，用於傳出身分／傳送委派
  和型別化承載內容規劃
- 當傳出路由應保留明確的
  `replyToId`/`threadId`，或在基礎工作階段金鑰仍相符後復原目前的 `:thread:`
  工作階段時，請使用來自
  `openclaw/plugin-sdk/channel-core` 的 `buildThreadAwareOutboundSessionRoute(...)`。如果供應商外掛的平台具有原生討論串傳遞語意，
  它們可以覆寫優先順序、後綴行為和討論串 ID 正規化。
- `openclaw/plugin-sdk/thread-bindings-runtime`，用於討論串繫結生命週期
  和轉接器註冊

僅驗證的頻道通常使用預設路徑即可：核心會處理
核准，而外掛只需公開傳出／驗證功能。Matrix、Slack、Telegram 等原生
核准頻道及自訂聊天傳輸應使用共用的原生輔助工具，而不是自行實作核准
生命週期。

## 傳入提及原則

將傳入提及處理分為兩層：

- 由外掛擁有的證據收集
- 共用原則評估

使用 `openclaw/plugin-sdk/channel-mention-gating` 進行提及原則判斷。
只有當你需要範圍較廣的
傳入輔助工具匯出介面時，才使用 `openclaw/plugin-sdk/channel-inbound`。

適合放在外掛本機邏輯中的項目：

- 偵測是否回覆機器人
- 偵測是否引用機器人
- 討論串參與檢查
- 排除服務／系統訊息
- 證明機器人參與情況所需的平台原生快取

適合使用共用輔助工具的項目：

- `requireMention`
- 明確提及結果
- 隱含提及允許清單
- 命令略過機制
- 最終略過判斷

建議流程：

1. 計算本機提及事實。
2. 將這些事實傳入 `resolveInboundMentionDecision({ facts, policy })`。
3. 在傳入閘門中使用 `decision.effectiveWasMentioned`、`decision.shouldBypassMention` 和
   `decision.shouldSkip`。

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";
import { resolveChannelImplicitMentions } from "openclaw/plugin-sdk/channel-ingress-runtime";

const wasMentioned = matchesMentionWithExplicit({
  text,
  mentionRegexes,
  explicit: {
    hasAnyMention,
    isExplicitlyMentioned,
    canResolveExplicit,
  },
});

const facts = {
  canDetectMention: true,
  wasMentioned,
  hasAnyMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const implicitMentions = resolveChannelImplicitMentions({
  cfg,
  channel: channelId,
  accountId,
});

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    implicitMentions,
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

`matchesMentionWithExplicit(...)` 會傳回布林值。`hasAnyMention`、
`isExplicitlyMentioned` 和 `canResolveExplicit` 來自頻道本身的
原生提及中繼資料（訊息實體、回覆機器人旗標及類似資訊）；
如果你的平台無法偵測它們，請提供 `false`/`undefined` 值。

`api.runtime.channel.mentions` 為已依賴執行階段注入的
內建頻道外掛公開相同的共用提及輔助工具：
`buildMentionRegexes`、`matchesMentionPatterns`、`matchesMentionWithExplicit`、
`implicitMentionKindWhen`、`resolveInboundMentionDecision`。

如果你只需要 `implicitMentionKindWhen` 和 `resolveInboundMentionDecision`，
請從 `openclaw/plugin-sdk/channel-mention-gating` 匯入，以避免載入
不相關的傳入執行階段輔助工具。

## 操作示範

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="套件與資訊清單">
    建立標準外掛檔案。`openclaw.plugin.json` 中的
    `channels` 欄位（不是 `kind` 欄位）會將資訊清單標記為
    擁有某個頻道。如需完整的套件中繼資料介面，請參閱
    [外掛設定與組態](/zh-TW/plugins/sdk-setup#openclaw-channel)：

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "將 OpenClaw 連線至 Acme Chat。"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Acme Chat 頻道外掛",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {}
      },
      "channelConfigs": {
        "acme-chat": {
          "schema": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          },
          "uiHints": {
            "token": {
              "label": "機器人權杖",
              "sensitive": true
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

    `configSchema` 會驗證 `plugins.entries.acme-chat.config`。請將它用於
    不屬於頻道帳號組態、但由外掛擁有的設定。
    `channelConfigs.acme-chat.schema` 會驗證 `channels.acme-chat`，並且是
    外掛執行階段載入前，由組態結構描述、設定和 UI 介面使用的
    冷路徑來源。如需完整的頂層欄位參考，請參閱[外掛資訊清單](/zh-TW/plugins/manifest)。

  </Step>

  <Step title="建構頻道外掛物件">
    `ChannelPlugin` 介面有許多選用的轉接器介面。先從最少項目
    `id`、`config` 和 `setup` 開始，再依需要加入
    轉接器。

    建立 `src/channel.ts`：

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // your platform API client

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        // Account resolution/inspection belongs on `config`, not `setup`.
        // `setup` covers onboarding writes (applyAccountConfig, validateInput).
        config: {
          listAccountIds: () => ["default"],
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
        setup: {
          applyAccountConfig: ({ cfg, input }) => ({
            ...cfg,
            channels: {
              ...cfg.channels,
              "acme-chat": { ...(cfg.channels as any)?.["acme-chat"], ...input },
            },
          }),
        },
      }),

      // DM security: who can message the bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: approval flow for new DM contacts
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading: how replies are delivered
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: send messages to the platform
      outbound: {
        attachedResults: {
          channel: "acme-chat",
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    對於同時接受標準頂層 DM 金鑰與舊版巢狀金鑰的頻道，請使用 `plugin-sdk/channel-config-helpers` 中的輔助函式：`resolveChannelDmAccess`、`resolveChannelDmPolicy`、`resolveChannelDmAllowFrom` 和 `normalizeChannelDmPolicy`，以確保帳號本機值優先於繼承的根層級值。請透過 `normalizeLegacyDmAliases` 將相同的解析器與 doctor 修復配對，讓執行階段與遷移作業讀取相同的合約。

    <Accordion title="createChatChannelPlugin 為你處理的工作">
      你不必手動實作低階轉接器介面，只需傳入
      宣告式選項，建構器便會將它們組合起來：

      | 選項 | 連接的功能 |
      | --- | --- |
      | `security.dm` | 從設定欄位解析具範圍限制的 DM 安全性 |
      | `pairing.text` | 使用代碼交換的文字式 DM 配對流程 |
      | `threading` | 回覆模式解析器（固定、帳號範圍或自訂） |
      | `outbound.attachedResults` | 傳回結果中繼資料（訊息 ID）的傳送函式；需要同層的 `channel` ID，讓核心能在傳回的遞送結果上加註頻道資訊 |

      若你需要完整控制，也可以傳入原始轉接器物件，
      而不使用宣告式選項。

      原始輸出轉接器可以定義 `chunker(text, limit, ctx)` 函式。
      選用的 `ctx.formatting` 會攜帶遞送時的格式化決策，
      例如 `maxLinesPerMessage`；請在傳送前套用，以便共用輸出遞送
      一次完成回覆串接與分塊邊界的解析。
      當原生回覆目標已解析時，傳送內容也會包含 `replyToIdSource`（`implicit` 或 `explicit`），
      讓酬載輔助函式可以保留明確的回覆標記，而不會消耗隱含的單次使用回覆位置。
    </Accordion>

    ### 群組工具政策轉接器

    實作 `group.resolveToolPolicy` 且支援
    `toolsBySender` 的頻道，必須將完整的 `ChannelGroupContext` 轉送至其
    共用政策解析器。尤其必須遵守 `senderPolicyMode: "never"`：
    在相符群組與萬用字元範圍中略過傳送者特定的覆寫，
    同時仍套用基礎 `tools` 政策。

    OpenClaw 只會針對受信任的非輸入執行設定此模式；這類執行的傳送者
    權限已記錄在伺服器擁有的封套中，例如明確設有限制的排程執行。
    外掛不得從輸入中繼資料推導此模式、將其持久化為頻道狀態，
    或將其公開為設定。請新增轉接器測試，證明此模式會略過萬用字元
    `toolsBySender` 項目，但不會捨棄相符的基礎 `tools` 限制。

  </Step>

  <Step title="連接進入點">
    建立 `index.ts`：

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Acme Chat channel plugin",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Acme Chat management");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Acme Chat management",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    請將頻道擁有的命令列介面描述元放在 `registerCliMetadata(...)` 中，讓 OpenClaw
    無須啟用完整的頻道執行階段，即可在根層級說明中顯示它們；
    一般的完整載入仍會取得相同描述元，以進行實際的命令
    註冊。請將 `registerFull(...)` 保留給僅限執行階段的工作。
    `defineChannelPluginEntry` 會自動處理註冊模式的分流。
    如果 `registerFull(...)` 註冊閘道 RPC 方法，請使用
    外掛專屬的前綴。核心管理命名空間（`config.*`、
    `exec.approvals.*`、`wizard.*`、`update.*`）會維持保留，且一律
    解析為 `operator.admin`。所有選項請參閱
    [進入點](/zh-TW/plugins/sdk-entrypoints#definechannelpluginentry)。

  </Step>

  <Step title="新增設定進入點">
    建立 `setup-entry.ts`，以便在初始設定期間進行輕量載入：

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    當頻道停用或尚未設定時，OpenClaw 會載入此項目，而非完整進入點。
    這可避免在設定流程期間載入龐大的執行階段程式碼。
    詳情請參閱[設定與組態](/zh-TW/plugins/sdk-setup#setup-entry)。

    將設定安全的匯出拆分至附屬模組的內建工作區頻道，
    若也需要明確的設定階段執行環境設定函式，可從
    `openclaw/plugin-sdk/channel-entry-contract` 使用 `defineBundledChannelSetupEntry(...)`。

  </Step>

  <Step title="處理傳入訊息">
    你的外掛需要從平台接收訊息，並將其轉送至 OpenClaw。
    典型模式是使用網路鉤子驗證請求，並透過頻道的傳入處理常式分派請求：

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // 由外掛管理驗證（請自行驗證簽章）
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // 你的傳入處理常式會將訊息分派至 OpenClaw。
          // 確切的連接方式取決於你的平台 SDK —
          // 請參閱內建 Microsoft Teams 或 Google Chat 外掛套件中的實際範例。
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      傳入訊息的處理方式因頻道而異。每個頻道外掛都擁有
      自己的傳入處理流程。請參閱內建頻道外掛
      （例如 Microsoft Teams 或 Google Chat 外掛套件）以瞭解實際模式。
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="測試">
在 `src/channel.test.ts` 中撰寫同位置測試：

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("acme-chat 外掛", () => {
      it("從組態解析帳戶", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.config.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("在不具體化密鑰的情況下檢查帳戶", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("回報缺少組態", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test <bundled-plugin-root>/acme-chat/
    ```

    共用測試輔助工具請參閱[測試](/zh-TW/plugins/sdk-testing)。

</Step>
</Steps>

## 檔案結構

```text
<bundled-plugin-root>/acme-chat/
├── package.json              # openclaw.channel 中繼資料
├── openclaw.plugin.json      # 包含組態結構描述的資訊清單
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # 公開匯出（選用）
├── runtime-api.ts            # 內部執行階段匯出（選用）
└── src/
    ├── channel.ts            # 透過 createChatChannelPlugin 建立的 ChannelPlugin
    ├── channel.test.ts       # 測試
    ├── client.ts             # 平台 API 用戶端
    └── runtime.ts            # 執行階段儲存區（如有需要）
```

## 進階主題

<CardGroup cols={2}>
  <Card title="串接選項" icon="git-branch" href="/zh-TW/plugins/sdk-entrypoints#registration-mode">
    固定、帳號範圍或自訂回覆模式
  </Card>
  <Card title="訊息工具整合" icon="puzzle" href="/zh-TW/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool 與動作探索
  </Card>
  <Card title="目標解析" icon="crosshair" href="/zh-TW/plugins/architecture-internals#channel-target-resolution">
    inferTargetChatType、looksLikeId、reservedLiterals、resolveTarget
  </Card>
  <Card title="執行階段輔助工具" icon="settings" href="/zh-TW/plugins/sdk-runtime">
    透過 api.runtime 使用 TTS、STT、媒體、子代理程式
  </Card>
  <Card title="頻道傳入 API" icon="bolt" href="/zh-TW/plugins/sdk-channel-inbound">
    共用的傳入事件生命週期：擷取、解析、記錄、分派、完成
  </Card>
</CardGroup>

<Note>
部分隨附的輔助接合介面仍保留供隨附外掛維護與
相容性使用。它們不是新頻道外掛的建議模式；
除非你正直接維護該隨附外掛系列，否則請優先使用共用 SDK
介面中的通用頻道、設定、回覆及執行階段子路徑。
</Note>

## 後續步驟

- [供應商外掛](/zh-TW/plugins/sdk-provider-plugins) - 如果你的外掛也提供模型
- [SDK 概覽](/zh-TW/plugins/sdk-overview) - 完整的子路徑匯入參考
- [SDK 測試](/zh-TW/plugins/sdk-testing) - 測試公用程式與契約測試
- [外掛資訊清單](/zh-TW/plugins/manifest) - 完整的資訊清單結構描述

## 相關內容

- [外掛 SDK 設定](/zh-TW/plugins/sdk-setup)
- [建置外掛](/zh-TW/plugins/building-plugins)
- [代理程式框架外掛](/zh-TW/plugins/sdk-agent-harness)
