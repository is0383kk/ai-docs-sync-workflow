---
read_when:
    - 實作或審查工作階段儀表板（看板）功能
    - 變更小工具託管、小工具橋接器或看板儲存空間
summary: 工作階段儀表板：架構與實作計畫（技術設計，GA 前）
title: 儀表板架構
x-i18n:
    generated_at: "2026-07-26T08:47:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a7c5da94ec19add55c6b7b530f0c17509a027e97fb301469ce48f520b325c169
    source_path: web/dashboard-architecture.md
    workflow: 16
---

<Note>
這是工作階段儀表板功能的技術設計文件，撰寫於實作之前及
實作期間。此文件是建置工作的唯一事實來源。功能推出後，
`/web/dashboard` 將成為面向使用者的頁面，而本頁會保留
作為架構參考。
</Note>

## 願景

目前與代理程式協作時，只有文字串流。儀表板將它變成
工作台：代理程式呈現即時互動式小工具；使用者將它們釘選到
持續存在的介面上；聊天停駐於側邊（或隱藏），而主要內容則是
面板。你不必離開工作階段，就能從「與代理程式交談」轉變為
「操作代理程式為你建置的控制面板」。

原則：

- **面板是工作階段的一個介面，而非新物件。** 每個工作階段（討論串）
  都有兩個介面：對話記錄與面板。沒有釘選小工具的工作階段
  就是一般聊天。釘選一個小工具後，面板即存在。面板沿用
  工作階段的身分、代理程式擁有權、命名、釘選與生命週期。不會有
  `dashboard_create`、面板登錄檔或獨立的 ACL 模型。
- **代理程式對等性。** 使用者能在面板上執行的一切操作，代理程式也能
  使用工具完成：新增／更新／移除小工具、排列小工具、管理分頁、切換
  顯示中的分頁，以及停駐或隱藏聊天。
- **原生而非嵌入。** 面板是控制介面殼層中的 Lit 元件
  （與應用程式其他部分使用相同的設計系統）。只有小工具的_內容_
  會在 iframe 中隔離。沒有網址列，也沒有瀏覽器介面元素。
- **精簡的代理程式介面。** 小工具透過穩定名稱定址，並在原處更新。
  版面配置採用可流動、自動緊縮的格線；代理程式指定尺寸與
  錨點，而非像素或座標。
- **能力優先於信任。** 小工具程式碼是代理程式任意撰寫的 HTML/JS，
  並在嚴格的沙箱中執行。其存取範圍（閘道資料、動作、網路）只能透過
  已宣告且由操作員授予的能力資訊清單取得。

## 概念

| 概念                | 定義                                                                                                                                                              |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 工作階段（討論串）  | 現有的閘道工作階段，以穩定的 `sessionKey` 作為索引鍵。由代理程式擁有。                                                                                      |
| 面板                | 單一工作階段的小工具介面。只有工作階段具有小工具／分頁時才存在。會保留於 `/new`/`/reset` 之後（附加至 `sessionKey`，而非對話記錄）。 |
| 分頁                | 面板的呈現頁面：包含哪些小工具、其排列方式，以及聊天停駐狀態（`left`/`right`/`bottom`/`hidden`）。面板一開始有一個隱含分頁。 |
| 小工具              | 由工作階段擁有、具名稱且在沙箱中執行的 HTML/JS 程式。以 `sessionKey` + `name` 定址。依名稱在原處更新。                                         |
| 能力資訊清單        | 每個小工具的存取範圍宣告：`data`（讀取繫結）、`actions`（允許清單中的動詞）、`prompt`（傳送至工作階段）、`net`（允許的來源）。 |
| 釘選（小工具）      | 將對話記錄中的小工具移至工作階段的面板（透過使用者操作介面或代理程式工具引數）。取消釘選會將其從面板移除。                                                        |
| 釘選（工作階段）    | 現有的側邊欄工作階段釘選功能。具有面板的已釘選工作階段會開啟其面板介面。                                                                                          |

## 使用者體驗流程

- **升級：** 代理程式在任何聊天中呼叫 `show_widget` → 小工具會與目前相同，
  直接呈現在對話記錄中 → 游標暫留時顯示 **Pin to dashboard** → 小工具
  出現在工作階段的面板上。代理程式可以傳遞 `pin: true` 來執行相同操作。
- **面板檢視：** 具有面板的工作階段會有介面切換器（聊天／儀表板）。
  面板檢視 = 分頁列（僅在有 >1 個分頁時顯示）+ 可流動格線 + 停駐的聊天窗格。
  聊天停駐區可以調整大小、移動（左側／右側／底部），並可像側邊欄一樣
  收合。系統會記住每個分頁的停駐狀態。
- **拖曳：** 使用者拖曳小工具；格線會自動緊縮（小工具向上浮動，相鄰項目
  重新排列）。透過控點調整大小時會貼齊尺寸級距。任何人都不能使用
  像素定位。
- **重設警告：** 在具有面板的工作階段中執行 `/new` / `/reset` 時，
  Web 使用者介面會要求確認（「內容會重設，但儀表板會保留」），並保留
  面板。
- **側邊欄：** 已釘選的工作階段若有面板，就會呈現其面板介面。
  首頁工作階段的面板是預設的「代理程式儀表板」。
- **互動**（分成三個層級，詳見下文）：靜默狀態事件、可見的
  提示傳送，以及自動化觸發程序。

## 互動層級

1. **狀態事件（預設）。** 模型應知悉但不必回應的
   小工具使用者介面互動。`bridge.emitState({...})` 會附加一則結構化的
   工作階段通知（使用與群組活動通知相同的機制）。不會啟動代理程式回合；
   模型會在下次執行時看到累積的通知。
2. **提示（明確交談）。** `bridge.sendPrompt(text)` — 需要使用者
   啟用；將一則可見的使用者訊息傳送至工作階段（停駐的聊天區會
   顯示該訊息）。受速率限制；除非小工具持有
   `prompt` 能力授權，否則每次傳送都需要使用者確認。
3. **自動化。** `bridge.runAction(name, args)` — 觸發資訊清單中宣告的
   動作。初始動詞集：`cron.trigger`（立即執行現有的排程工作）和
   `binding.refresh`。排程工作原本就會在可見且隔離的執行工作階段中執行，
   並且可以使用成本較低的模型：這就是「小型模型為小工具提供動力」的
   路徑。任何地方都不會有隱藏的工作階段。

## 小工具模型與託管

小工具 HTML/JS 由代理程式撰寫（通常透過 `show_widget`），包裝於
標準文件殼層中（CSP meta、尺寸回報器、橋接啟動程式），並在
`<iframe sandbox="allow-scripts">` 中呈現（絕不使用 `allow-same-origin`）。

- **內嵌（對話記錄）小工具**會保留目前的畫布文件管線：
  寫入狀態目錄、由閘道提供、依範圍清理，且無需核准（其設計上不具任何
  能力——傳送提示時需由使用者確認）。
- **面板小工具**屬於工作階段狀態：位元組儲存於所屬代理程式的 SQLite
  DB（`board_widgets`），並由讀取該 DB 的核心閘道路由
  （`/__openclaw__/board/<agentId>/<sessionKey>/<name>/`）提供。
  釘選對話記錄中的小工具會複製其位元組。上限：每個小工具 256 KB，
  每個面板 48 個小工具。
- **原處更新：** 重新發出具有相同 `name` 的小工具時，會取代
  位元組、遞增 `revision`、廣播 `board.changed`，且即時檢視只會重新載入
  該 iframe。
- **位元組凍結：** 已授予的能力會繫結至小工具位元組的 sha256。
  變更位元組後，只有在新修訂版宣告的是已授予資訊清單的子集時，才會保留
  `data`/`net`/`actions` 授權；擴大的資訊清單
  會再次提示操作員。

### 小工具託管內容；MCP 應用程式是其中一種內容類型

**小工具是 OpenClaw 的基本元素**：具有名稱、已釘選、已設定尺寸、
由工作階段擁有的面板儲存格，並具有授權記錄。在其中呈現的是
一種內容類型：

- `html` — 代理程式透過 `show_widget` 撰寫，位元組位於面板儲存空間中。
- `mcp-app` — 在小工具儲存格內託管的第三方 MCP 應用程式檢視（來自已設定伺服器的 `ui://` 資源）。

MCP 應用程式不會定義小工具模型；而是小工具取得了託管
這些應用程式的能力。身分、放置位置、釘選、授權及面向作者的 API 仍然
屬於 OpenClaw，因此 `show_widget` 程式碼能像目前一樣簡短，且完全
不需要知道 MCP Apps 規格的存在。

底層共用基礎架構（簡化發生於此）：

- **單一沙箱主機。** `html` 小工具會透過 MCP 應用程式隨附的相同強化
  管線呈現（在專用沙箱來源上使用雙重 iframe、宣告每個小工具的 CSP，並以失敗即關閉的方式解碼），
  而非使用第二套特製的 iframe 主機。Proxy 會以值的形式接收 HTML，因此本機內容
  自然就是一般情況。
- **單一授權模型。** 無論小工具的類型為何，其存取範圍都是已授予的允許清單：
  對 `html` 小工具而言是主機工具；對 `mcp-app` 小工具而言，
  則是伺服器提供給應用程式的可見工具（透過現有的 `allowedAppToolNames`
  機制，但改為每個小工具持久保留，而非僅限於建立該小工具的執行）。
- **`html` 小工具的主機工具**（透過小工具橋接公開，並依授權檢查）：
  - `openclaw.prompt.send` — 第 2 層級；透過可見的撰寫器路由，
    除非已授權，否則需要使用者確認
  - `openclaw.state.emit` — 第 1 層級工作階段通知（合併處理，且有大小上限）
  - `openclaw.data.read` — 參數化唯讀繫結（現有的
    允許清單讀取 RPC 集合），由閘道端解析
  - `openclaw.cron.trigger` — 第 3 層級自動化
- **`net` = CSP。** 網路存取範圍使用已推出的每個小工具 CSP
  宣告（`connect-src` 來源）——可自行更新的天氣小工具
  直接從沙箱擷取其 API，無需閘道參與。
- **授權。** 未宣告任何項目的小工具會立即呈現（在沙箱中執行、
  `default-src 'none'`、每次傳送提示時個別確認）——其信任程度與
  目前的內嵌聊天小工具相同。宣告的工具／來源會讓面板上的小工具進入
  `pending`：預留位置卡片會以人類可讀的方式列出這些項目，並提供單次點按的
  **Allow**/**Reject**。授權以小工具名稱為單位；對 `html` 小工具而言，
  授權會依位元組凍結（sha256），且只有在宣告範圍縮小時，變更後的位元組才會保留授權。
- **撰寫相容層。** 文件包裝函式會插入 `window.openclaw.prompt`、
  `window.openclaw.state`、`window.openclaw.data` 和 `window.openclaw.cron`
  作為穩定的作者 API。儀表板呼叫會共用一個繫結至檢視票證的
  請求通道；尺寸回報與佈景主題權杖仍是獨立的主機通知。

### 外掛能力宣告

已啟用的外掛可以透過 `openclaw.plugin.json` 中的 `dashboard.dataBindings`
和 `dashboard.actionVerbs` 擴充小工具主機。外掛本機識別碼會變成
以外掛識別碼為前綴的授權名稱，例如 `workboard.cards.list` 和
`workboard.dispatch`；外掛識別碼區段中的 `%` 和 `.` 會被逸出，避免
不同的外掛／本機識別碼分割方式繼承相同的持久化授權。在
外掛註冊期間，OpenClaw 會驗證每個繫結是否指向由相同外掛透過
`operator.read` 註冊的 RPC，並驗證每個動作是否指向透過
`operator.write` 註冊的 RPC；無效的宣告會導致外掛載入失敗。只有在外掛生命週期
變更時才會重建經驗證的登錄檔，而小工具授權仍會按小工具個別儲存，並繫結至
位元組與修訂版。

### 已建模的殘餘風險：WebRTC 資料通道

沙箱 CSP 會發出提議中的 `webrtc 'block'` 指令，但
[Chromium 目前的 CSP 指令集](https://chromium.googlesource.com/chromium/src/+/main/services/network/public/mojom/content_security_policy.mojom#95)
並未實作該指令。因此，在目前的 Chromium 中，可執行指令碼的小工具可以使用 WebRTC 資料
通道進行資料外傳。`main` 上的內嵌聊天小工具與 MCP Apps 主機
目前也已存在相同的殘餘風險。

**可接受的取捨：** OpenClaw 不會根據此殘餘風險封鎖可編寫指令碼的小工具。小工具內容只能透過由操作員授予、位元組凍結的 `data:read` 能力存取敏感的 OpenClaw 資料，而沙箱 Permissions Policy 會封鎖攝影機與麥克風存取。DOM API 防護屬於盡力而為的縱深防禦，而非安全邊界，應納入後續強化工作。

### 對話記錄顯示：單一小工具卡片

行內顯示統一採用小工具原語。當工具結果帶有 UI——`show_widget` 輸出，或含有應用程式資源的 MCP 工具結果——系統會具現化一個**暫時、自動命名的小工具**（以工作階段為範圍，且會清除），而對話記錄會呈現單一小工具卡片，並依內容種類分派。MCP 應用程式的自動顯示會完全維持規格所預期的行為（不增加任何模型工作）；其底層本身就是小工具。這會刪除聊天算繪中平行的 `mcpApp` 特殊處理（介面範圍限制、獨立去重），讓每個行內 UI 都具備相同的釘選操作，並使小工具登錄成為主要的重新開啟路徑（透過掃描對話記錄重建，仍作為從未釘選之歷史內容的備援）。唯讀、具票證的獨立主機與看板同樣提供持久的重新開啟介面——這是要在 T6 評估的整併候選項目，不預先假定會整併。

組合：v1 採用網格相鄰配置（同一分頁中，代理程式外框小工具位於應用程式小工具旁）。v2 新增**由主機管理的應用程式插槽**——代理程式小工具 HTML 會宣告插槽區域，而主機會將真正的應用程式檢視組合為同層沙箱。應用程式絕不會在代理程式的 iframe 內算繪：巢狀結構會破壞橋接器身分，並可能對已獲授權的應用程式 UI 進行覆蓋／點擊劫持，因此插槽是版面配置合約，而非嵌入。

### 伺服器來源的小工具（已釘選的 MCP 應用程式）

使用統一主機後，釘選第三方 MCP 應用程式，就只是建立一個從伺服器擷取內容而非儲存內容的小工具：`board_widgets` 會保留描述元（`serverName`、`toolName`、`uiResourceUri`、來源 `toolCallId` + `sessionKey`），而非 HTML 位元組；看板會在聊天回合的 10 分鐘 TTL 過期後重新鑄造檢視租約（失效時重新擷取 `ui://` 資源）。聊天中的行內 MCP 應用程式檢視，會獲得與代理程式小工具相同的**釘選至儀表板**操作。依設計，目前重新開啟的檢視為唯讀；應維持互動能力的已釘選應用程式，會取得對伺服器中應用程式可見工具的持久授權（釘選時向操作員顯示明確的允許清單），並與鑄造它的執行作業解耦。未獲授權的釘選項目仍維持唯讀——對顯示型儀表板仍然實用。v1 會釘選至來源工作階段的看板；跨工作階段釘選需要租約代理程式，因此延後處理。請與開放中的 PR #109807 協調（`ui/message` 編輯器路由、主題／尺寸傳遞）。

### WorkBoard 整合

WorkBoard 整合計畫會讓卡片與看板繼續由外掛擁有，同時透過現有的 `sessionKey` 與 `runId`，將已分派的卡片銜接回其工作階段看板；並透過外掛宣告的繫結與動作公開 WorkBoard 摘要來源及分派功能，再將這些結果與現有的 `html` 和 `mcp-app` 小工具種類組合，而不引入 WorkBoard 專用的小工具類型。

## 版面配置：流動式網格

12 欄、固定列高、**自動緊縮**（向上吸附，拖曳時推開其他項目——採用 gridstack 語意，但以原生方式實作；網格數學運算維持純函式且不依賴 DOM）。每個分頁的小工具版面配置狀態：`{ name, w (1-12), h (rows) }` 加上順序。代理程式詞彙：

- `size`：`sm`（3×3）· `md`（6×4）· `lg`（8×6）· `xl`（12×8）· `full`
  （單一小工具分頁）
- `after: <widgetName>` 選用的排序錨點；省略 = 附加至末尾
- 使用者可自由拖曳／調整大小；同一套順序＋尺寸模型可完整往返轉換。

## 資料模型（每個代理程式的資料庫）

在 `agents/<agentId>/agent/openclaw-agent.sqlite` 中新增資料表
（**需要提升代理程式資料庫結構版本——在此變更落地前，必須取得操作員核准**）：

```sql
CREATE TABLE board_tabs (
  session_key TEXT NOT NULL,
  tab_id      TEXT NOT NULL,           -- 代稱
  title       TEXT NOT NULL,
  position    INTEGER NOT NULL,
  chat_dock   TEXT NOT NULL DEFAULT 'right',  -- left|right|bottom|hidden
  created_by  TEXT NOT NULL,           -- 'user' | 'agent'
  PRIMARY KEY (session_key, tab_id)
) STRICT;

CREATE TABLE board_widgets (
  session_key  TEXT NOT NULL,
  name         TEXT NOT NULL,          -- 穩定的小工具名稱
  tab_id       TEXT NOT NULL,
  title        TEXT,
  html         BLOB NOT NULL,          -- 經包裝的文件原始碼
  sha256       TEXT NOT NULL,
  revision     INTEGER NOT NULL,
  size_w       INTEGER NOT NULL,
  size_h       INTEGER NOT NULL,
  position     INTEGER NOT NULL,       -- 分頁內的順序（自動緊縮輸入）
  manifest     TEXT NOT NULL DEFAULT '{}',  -- 能力資訊清單 JSON
  grant_state  TEXT NOT NULL DEFAULT 'none', -- none|pending|granted|rejected
  granted_sha  TEXT,                   -- 位元組凍結的授權
  created_by   TEXT NOT NULL,
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL,
  PRIMARY KEY (session_key, name)
) STRICT;
```

若 `sessionKey` 存在任何資料列，即表示看板存在。刪除工作階段時，會刪除其看板資料列。`/new`/`/reset` 不會變更這些資料列。

## 通訊協定介面

RPC（核心方法表，typebox 結構描述位於 `gateway-protocol`）：

- `board.get { sessionKey }` → 分頁＋小工具中繼資料（不含位元組）——`operator.read`
- `board.update { sessionKey, ops[] }`——分頁 CRUD／重新排序、小工具移動／調整大小／
  移除／取消釘選、停駐狀態、聚焦分頁——`operator.write`
- `board.widget.put { sessionKey, name, html, manifest, placement }`——
  `operator.write`（代理程式工具路徑與釘選路徑）
- `board.widget.grant { sessionKey, name, decision }`——`operator.approvals`
- `board.event { ticket, payload }`——受票證約束的第 1 層狀態事件擷取；
  舊版受信任主機的 `{ sessionKey, widget, payload }` 形狀會保留——
  `operator.write`
- `board.prompt.authorize { ticket }`——傳回可見提示傳送是否仍需逐次點擊確認——`operator.read`
- `board.data.read { ticket, bindingId, params? }`——由閘道端允許清單限制的
  核心或作用中外掛讀取繫結解析——`operator.read`
- `board.action { ticket, action, ... }`——透過現有排程立即執行路徑，或作用中外掛經驗證的動作動詞，進行精確授權的自動化分派——`operator.write`

事件（位於 `EVENT_SCOPE_GUARDS`，讀取範圍）：

- `board.changed { sessionKey, revision, widget? }`——持久化狀態已變更；
  UI 會重新擷取（若存在 `widget`，也會重新載入一個 iframe）。
- `board.command { sessionKey, command }`——暫時性 UI 驅動（代理程式切換
  可見分頁、切換聊天停駐區）——採用 `ui.command` 模式。

小工具位元組透過已驗證的 HTTP 介面提供，而非透過通訊端。

## 代理程式工具

總共三個工具（核心、永遠註冊；算繪仍如目前一樣受 `inline-widgets` 用戶端能力限制）：

- `show_widget { title, widget_code, name?, pin?, size?, tab?, after?,
capabilities? }`——依名稱建立／更新；`pin` 會將它放到看板上。
  若沒有 `name`/`pin`，其行為會與目前完全相同（行內、暫時）。
- `dashboard { action, ... }`——看板管理動詞：`read`、`tab_create`、
  `tab_update`、`tab_delete`、`tabs_reorder`、`widget_move`、`widget_remove`、
  `unpin`、`focus_tab`、`set_chat_dock`。
- 現有的 `cron` 工具涵蓋自動化層；不需要新增工具。

工具說明會教導尺寸／錨點詞彙及分層模型。系統會透過工作階段通知告知代理程式使用者的第 1 層事件，例如 `[dashboard] user clicked "Refresh" on widget weather (tab main)`。

## 這會取代什麼

- **刪除 `extensions/workspaces`。** 此為實驗性功能，`enabledByDefault:
false`，從未出現在穩定版本中（首次出現於 2026.7.2 beta 版本）。不進行遷移；doctor 規則會移除任何殘留的 `<stateDir>/workspaces/`。
  保留的構想：純網格數學運算、橋接器安全模型（連接埠啟動、
  繫結限制、速率限制）、位元組凍結核准。
- **小工具託管從 `extensions/canvas` 移至核心。** 畫布文件
  儲存區、文件包裝器、HTTP 服務，以及 `show_widget` 工具會成為核心功能
  （`src/canvas/`）；此外掛保留節點畫布控制工具（`canvas`）及
  A2UI。`pluginSurfaceUrls["canvas"]` 公告和
  `/__openclaw__/canvas` 路徑是已發布的原生用戶端合約，會維持
  穩定。Discord 工作階段繼續使用由 Discord 擁有的 `show_widget` 變體。

## 非目標（本計畫）

- 多使用者看板共用／ACL（未來功能；將透過工作階段共用提供）。
- 原生 macOS/iOS 看板算繪（凡是嵌入 Control UI 的地方都能取得；
  行內小工具路徑維持不變）。
- 內建資料小工具（工作階段／用量／排程卡片）——能力橋接器加上
  代理程式編寫的小工具即可涵蓋 v1；稍後可再加入內建種類登錄。

## 實作計畫

使用獨立工作樹，由 Codex 建置，依序審查＋落地。先落地再修正。

| #   | 分支                               | 範圍                                                                                                                                                                              | 相依項目                       |
| --- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| T1  | `claude/dashboard-remove-workspaces` | 刪除工作區外掛＋UI＋文件＋i18n 鍵；doctor 清理規則                                                                                                              | —                                |
| T2  | `claude/dashboard-canvas-core`       | 將小工具託管＋`show_widget` 提升至核心；畫布外掛保留節點工具；行為完全不變                                                                                | —                                |
| T3  | `claude/dashboard-domain`            | 代理程式資料庫資料表（結構版本提升）、`board.*` RPC＋事件、`dashboard` 工具、`show_widget` 釘選／名稱／資訊清單引數、第 1 層通知、重設時保留看板                                  | T2                               |
| T4  | `claude/dashboard-ui`                | 看板介面＋分頁列＋流動式自動緊縮網格＋聊天停駐區（左／右／底部／隱藏）＋對話記錄釘選操作＋側邊欄看板介面＋重設確認                           | T3（先透過開發固定資料模擬） |
| T5  | `claude/dashboard-capabilities`      | 授權儲存區／UI＋位元組凍結；將 `html` 小工具移至共用沙箱主機；主機工具（`openclaw.prompt.send/state.emit/data.read/cron.trigger`）；`net` CSP；編寫相容層 | T3、T4                           |
| T7  | `claude/dashboard-mcp-apps`          | `mcp-app` 內容種類：行內應用程式檢視的釘選操作、描述元儲存、租約重新鑄造／重新整理、持久的伺服器工具授權（重複使用已發布的 MCP Apps 主機）                   | T3、T4                           |
| T6  | 潤飾                               | 在暫用閘道上執行即時 E2E（真實金鑰）、螢幕截圖、修正、以使用者為中心重寫 `/web/dashboard`、預設啟用審查                                                     | 全部                              |

依儲存庫規則驗證：在本機執行聚焦的 vitest、在 Crabbox/Testbox 執行完整閘門、每次落地前執行 `$autoreview`，並為 T6 提供即時驗證。
