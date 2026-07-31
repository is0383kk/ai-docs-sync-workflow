---
read_when:
    - 你需要 Codex 控制框架執行階段支援合約
    - 你正在偵錯原生 Codex 工具、掛鉤、壓縮或意見回饋上傳問題
    - 你正在跨 OpenClaw 與 Codex 測試框架回合變更外掛行為
summary: Codex 控制框架的執行階段邊界、鉤子、工具、權限與診斷
title: Codex 控制框架執行階段
x-i18n:
    generated_at: "2026-07-26T07:25:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d18d42683df0d827b776547f7b45f60f572cf39410d00533f53f8fdcdccb0d2
    source_path: plugins/codex-harness-runtime.md
    workflow: 16
---

Codex 控制框架回合的執行階段合約。如需設定與路由資訊，請參閱
[Codex 控制框架](/zh-TW/plugins/codex-harness)。如需設定欄位資訊，請參閱
[Codex 控制框架參考](/zh-TW/plugins/codex-harness-reference)。

## 概觀

Codex 負責原生模型迴圈、原生執行緒續接、原生工具接續，以及原生壓縮。OpenClaw 負責頻道路由、工作階段檔案、可見訊息傳遞、OpenClaw 動態工具、核准、媒體傳遞，以及該邊界外圍的逐字稿鏡像。

提示路由依循所選的執行階段，而不只是提供者字串。原生 Codex 回合會取得 Codex app-server 開發者指示；明確的 OpenClaw 相容性路由即使使用 Codex 風格的 OpenAI 驗證或傳輸，仍會保留一般 OpenClaw 系統提示。

OpenClaw 啟動及續接原生 Codex 執行緒時，會停用 Codex 的內建
個性設定（`personality: "none"`），讓工作區個性檔案
與 OpenClaw 代理程式身分維持權威性。除此之外，原生 Codex 仍保留 Codex 所擁有的
基礎／模型指示與專案文件載入。輕量型
OpenClaw 執行（例如排程）仍會停用專案文件載入。

OpenClaw 開發者指示涵蓋 OpenClaw 執行階段相關事項：來源頻道
傳遞、OpenClaw 動態工具、ACP 委派、轉接器情境，以及
目前代理程式工作區的設定檔。Skills 目錄與透過工具路由的
`MEMORY.md` 指標會投射為限定於該回合的協作開發者
指示。記憶工具無法使用時，作用中的 `BOOTSTRAP.md` 內容
與完整的 `MEMORY.md` 會改以純文字回合輸入情境提供。

大多數 OpenClaw 動態工具使用可搜尋的 `openclaw` 命名空間。標記為
`catalogMode: "direct-only"` 的工具使用 `openclaw_direct`，Codex 會將其
以 `DirectModelOnly` 形式直接保持為模型可見，而不是向巢狀
Code Mode 執行公開。

## 執行緒繫結與模型變更

OpenClaw 工作階段附加至現有 Codex 執行緒時，下一個
回合會將目前選取的模型、核准原則、沙箱、
核准審查者與服務層級重新傳送至 app-server。從
`openai/gpt-5.5` 切換至 `openai/gpt-5.2` 會保留執行緒繫結，但會要求 Codex
使用新選取的模型繼續執行。

受監督的繫結則是例外。OpenClaw 模型選擇器會保持鎖定，
且續接時省略模型與提供者覆寫，讓 Codex 還原標準
執行緒已持久保存的模型與提供者。另一個原生 Codex 控制項可以
變更該持久保存的配對，而初始快照可能產生 Codex 一般的
模型差異警告；外層 OpenClaw 模型與備援鏈絕不會
取代其中任何一項。

## 監督與安全接續

Codex 監督是同一個 `codex` 外掛的選用功能。它會透過
獨立連線探索原生執行緒，並只將未封存的
工作階段投射至閘道目錄。若未明確設定 `appServer` 連線，
該連線會使用受管理的使用者家目錄 stdio，而一般
控制框架仍限定於代理程式範圍。清單與中繼資料讀取是被動的：它們不會
續接執行緒、不會讓 OpenClaw 訂閱其即時事件，也不會回應其
核准要求。

對於閘道電腦上已儲存或閒置的工作階段，**以分支接續**
會建立一般且模型鎖定的聊天，並鏡像有界的使用者與助理
歷史記錄，直到來源最後一個已持久保存的終止回合。第一個一般
聊天回合會安裝真正的核准處理常式，並使用暫時的原生分支
固定快照，而不覆寫模型或提供者。Codex App Server 會使用
目前的原生設定並回傳所選配對；如果該模型與來源最後記錄的模型不同，
則會發出一般警告。
在同一個監督連線上，OpenClaw 會在來源的 cwd 與執行階段原則下，
以該初始啟動所回傳的模型與提供者原封不動地啟動標準
`appServer` 來源 Codex 控制框架執行緒，注入
有界的可見歷史記錄，並封存暫時分支。來源絕不會
被續接。標準執行緒具備完整的 OpenClaw 控制框架工具介面；
來源的推理、工具呼叫與工具結果不會複製至其中。
私人連線範圍會在待處理與已提交的繫結狀態下持續存在，因此
之後每個回合都會留在該連線上，並使用原生驗證與提供者
設定。停用監督或發生繫結／連線偏移時會以關閉方式失敗，
而不會切換至一般的代理程式家目錄控制框架。

原始的命令列介面、VS Code、Atlas 或 ChatGPT 來源仍符合兩個
目錄的收錄資格。標準分支是原生 Codex 執行緒，但其來源種類為
`appServer`；原生用戶端可能會篩除此來源種類，因此不保證
它會出現在 Codex Desktop 中。

作用中的來源無法啟動新分支或被封存；現有的受監督
聊天仍可開啟。`notLoaded` 表示活動狀態未知，而非閒置；
只有在明確確認沒有其他執行器，且取得最新的處理程序本機狀態後，
OpenClaw 才允許封存本機 `idle` 或 `notLoaded` 資料列。Codex
會在單一 App Server 處理程序內序列化執行緒異動，但不提供
跨處理程序的專屬執行器或核准擁有者租約，因此該讀取結果無法
證明其他處理程序未使用該執行緒。對於完全相符的目標，或 Codex
分頁式子孫查詢所回傳的任何未封存衍生子孫，OpenClaw 會封鎖已知
作用中的繫結擁有者。列舉錯誤、循環與安全限制耗盡都會以關閉方式失敗。
原生封存仍可能與另一個處理程序中的新回合發生競爭，因此確認涵蓋
未知用戶端，以及狀態讀取與封存之間的空檔。受監督且模型鎖定的
聊天在保護原生繫結期間無法刪除。

配對節點目錄在初始版本中只提供中繼資料。目前的
節點叫用邊界採用要求／回應模式，無法承載真正 Codex 控制框架
繫結所需的長時間回合事件、核准要求或串流輸出。因此，即使
資料列處於閒置狀態，遠端 **接續** 與 **封存** 仍無法使用。

如需操作人員設定與 Control UI 可見行為，請參閱
[Codex 監督](/zh-TW/plugins/codex-supervision)。

## 可見回覆與心跳偵測

透過 Codex 控制框架進行的直接／來源聊天回合，預設會為內部 WebChat
介面自動傳遞最終助理回覆，這與 Pi 控制框架
合約一致：代理程式會正常回覆，而 OpenClaw 會將最終文字發布至
來源對話。設定 `messages.visibleReplies: "message_tool"`，即可讓
最終助理文字維持私密，除非代理程式呼叫 `message(action="send")`。

Codex 心跳偵測回合預設會在可搜尋的 OpenClaw 工具
目錄中取得 `heartbeat_respond`，讓代理程式記錄這次喚醒應保持安靜
或發出通知。心跳偵測主動性指引會以限定於該心跳偵測回合的 Codex 協作模式
開發者指示傳送；一般聊天回合維持 Codex Default 模式。
當 `HEARTBEAT.md` 非空白時，心跳偵測
指示會將 Codex 導向該檔案，而不是直接內嵌其內容。

## 掛鉤邊界

| 層級                                  | 擁有者                   | 用途                                                                |
| ------------------------------------- | ------------------------ | ------------------------------------------------------------------- |
| OpenClaw 外掛掛鉤                     | OpenClaw                 | 維持 OpenClaw 與 Codex 控制框架之間的產品／外掛相容性。             |
| Codex app-server 擴充中介軟體         | OpenClaw 內建外掛        | 圍繞 OpenClaw 動態工具的每回合轉接器行為。                           |
| Codex 原生掛鉤                        | Codex                    | 由 Codex 設定控制的低階 Codex 生命週期與原生工具原則。              |

OpenClaw 不會使用專案或全域 Codex `hooks.json` 檔案來路由
外掛行為。對於原生工具與權限橋接，OpenClaw 會為每個執行緒注入
`PreToolUse`、`PostToolUse`、`PermissionRequest`
及 `Stop` 的 Codex 設定。

啟用 Codex app-server 核准時（`approvalPolicy` 不是
`"never"`），預設注入的原生掛鉤設定會省略 `PermissionRequest`，
讓 Codex 的 app-server 審查者與 OpenClaw 的核准橋接在審查後處理真正的
權限提升。若仍要強制使用相容性轉送，請將 `permission_request` 新增至
`nativeHookRelay.events`。其他 Codex
掛鉤（例如 `SessionStart` 與 `UserPromptSubmit`）仍屬於 Codex 層級
控制項；在 v1 合約中，它們不會公開為 OpenClaw 外掛掛鉤。

對於 OpenClaw 動態工具，Codex 要求呼叫後會由 OpenClaw 執行工具，
因此外掛與中介軟體行為會在控制框架轉接器中執行。Codex
Code Mode 會以文字形式接收一般動態結果，並序列化巢狀
動態呼叫；呼叫端必須剖析看似 JSON 的結果，且無法依賴
`Promise.all` 進行並行提交。對於 Codex 原生工具，Codex 擁有
標準工具記錄；OpenClaw 可以鏡像特定事件，但除非 Codex 透過
app-server 或原生掛鉤回呼公開該能力，否則無法改寫
原生執行緒。

Codex app-server 報告模式的 `PreToolUse` 事件會將外掛核准延後至
相符的 app-server 核准。若 OpenClaw `before_tool_call` 掛鉤回傳
`requireApproval`，同時原生承載資料設定 `openclaw_approval_mode:
"report"`，原生掛鉤轉送會記錄外掛核准要求，
且不回傳原生決策。之後 Codex 針對相同工具使用傳送 app-server 核准
要求時，OpenClaw 會開啟外掛核准提示，並將
決策對應回 Codex。Codex `PermissionRequest` 事件是
獨立的核准路徑，設定該橋接後仍可透過 OpenClaw 核准進行路由。

Codex app-server 項目通知也會針對原生
`PostToolUse` 轉送尚未涵蓋的原生工具完成事件，提供非同步 `after_tool_call`
觀察結果。這些僅供遙測／相容性使用；無法
封鎖、延遲或變更原生工具呼叫。

壓縮與 LLM 生命週期投射來自 Codex app-server
通知及 OpenClaw 轉接器狀態，而非原生 Codex 掛鉤命令。
`before_compaction`、`after_compaction`、`llm_input` 與 `llm_output` 是
轉接器層級的觀察結果，而非 Codex 內部
要求或壓縮承載資料的逐位元組擷取。

Codex 原生 `hook/started` 與 `hook/completed` app-server 通知會
投射為 `codex_app_server.hook` 代理程式事件，用於軌跡記錄與
偵錯。它們不會叫用 OpenClaw 外掛掛鉤。

## V1 支援合約

Codex 執行階段 v1 支援：

| 介面                                          | 支援狀態                                                                         | 原因                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| --------------------------------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 透過 Codex 的 OpenAI 模型迴圈                 | 支援                                                                             | Codex app-server 負責 OpenAI 輪次、原生執行緒續接，以及原生工具接續執行。                                                                                                                                                                                                                                                                                                                                                                                                            |
| OpenClaw 頻道路由與傳遞                       | 支援                                                                             | Telegram、Discord、Slack、WhatsApp、iMessage 和其他頻道均位於模型執行階段之外。                                                                                                                                                                                                                                                                                                                                                                                                      |
| OpenClaw 動態工具                             | 支援                                                                             | Codex 要求 OpenClaw 執行這些工具，因此 OpenClaw 會持續位於執行路徑中。                                                                                                                                                                                                                                                                                                                                                                                                              |
| 提示詞與情境外掛                              | 支援                                                                             | OpenClaw 會將 OpenClaw 特有的提示詞／情境投射至 Codex 輪次，同時將 Codex 所有的基礎提示詞、模型提示詞及已設定的專案文件提示詞保留在原生 Codex 路徑中。OpenClaw 會停用原生執行緒中 Codex 的內建人格，讓代理程式工作區的人格檔案維持權威性。原生 Codex 開發者指示僅接受明確限定於 `codex_app_server` 的命令指引；舊版全域命令提示仍保留給非 Codex 提示詞介面。 |
| 情境引擎生命週期                              | 支援                                                                             | 組裝、擷取，以及輪次後維護會圍繞 Codex 輪次執行。情境引擎不會取代原生 Codex 壓縮。                                                                                                                                                                                                                                                                                                                                                                                                    |
| 動態工具掛鉤                                  | 支援                                                                             | `before_tool_call`、`after_tool_call` 和工具結果中介軟體會圍繞 OpenClaw 所有的動態工具執行。                                                                                                                                                                                                                                                                                                                                                                                        |
| 生命週期掛鉤                                  | 支援作為配接器觀察項目                                                           | `llm_input`、`llm_output`、`agent_end`、`before_compaction` 和 `after_compaction` 會以如實反映 Codex 模式的承載資料觸發。                                                                                                                                                                                                                                                                                                                                            |
| 最終答案修訂閘門                              | 透過原生掛鉤轉送提供支援                                                         | Codex `Stop` 會轉送至 `before_agent_finalize`；`revise` 會要求 Codex 在定稿前再執行一次模型輪次。                                                                                                                                                                                                                                                                                                                                                                    |
| 原生 shell、修補與 MCP 封鎖或觀察             | 透過原生掛鉤轉送提供支援                                                         | 對於已提交的原生工具介面，Codex `PreToolUse` 和 `PostToolUse` 會被轉送，包括 Codex app-server `0.142.0` 或更新版本上的 MCP 承載資料。支援封鎖，但不支援改寫引數。                                                                                                                                                                                                                                                                                                     |
| 原生權限政策                                  | 透過 Codex app-server 核准與相容性原生掛鉤轉送提供支援                           | Codex app-server 核准要求會在 Codex 審查後透過 OpenClaw 路由。`PermissionRequest` 原生掛鉤轉送在原生核准模式中為選擇性啟用，因為 Codex 會在守護程式審查前發出該掛鉤。                                                                                                                                                                                                                                                                                                                      |
| App-server 軌跡擷取                           | 支援                                                                             | OpenClaw 會記錄傳送給 app-server 的要求，以及從 app-server 收到的通知。                                                                                                                                                                                                                                                                                                                                                                                                              |

Codex 執行階段 v1 不支援：

| 介面                                                | V1 邊界                                                                                                                                             | 未來方向                                                                                  |
| --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| 原生工具引數變更                                    | Codex 原生工具前置掛鉤可以封鎖，但 OpenClaw 不會改寫 Codex 原生工具的引數。                                                                          | 需要 Codex 掛鉤／結構描述支援替換工具輸入。                                               |
| 可編輯的 Codex 原生逐字記錄歷程                     | Codex 擁有標準原生執行緒歷程。OpenClaw 擁有鏡像並可投射未來情境，但不應變更不受支援的內部資料。                                                       | 若需要修改原生執行緒，請新增明確的 Codex app-server API。                                  |
| 適用於 Codex 原生工具記錄的 `tool_result_persist`      | 該掛鉤會轉換 OpenClaw 所有的逐字記錄寫入，而非 Codex 原生工具記錄。                                                                                  | 可以鏡像轉換後的記錄，但標準改寫需要 Codex 支援。                                         |
| 豐富的原生壓縮中繼資料                              | OpenClaw 可以要求原生壓縮，但不會收到穩定的保留／捨棄清單、權杖差值、完成摘要或摘要承載資料。                                                        | 需要更豐富的 Codex 壓縮事件。                                                             |
| 壓縮介入                                            | OpenClaw 不允許外掛或情境引擎否決、改寫或取代原生 Codex 壓縮。                                                                                      | 若外掛需要否決或改寫原生壓縮，請新增 Codex 壓縮前／後掛鉤。                               |
| 逐位元組的模型 API 要求擷取                         | OpenClaw 可以擷取 app-server 要求與通知，但 Codex 核心會在內部建構最終的 OpenAI API 要求。                                                           | 需要 Codex 模型要求追蹤事件或偵錯 API。                                                   |

## 原生權限與 MCP 資訊徵詢

對於 `PermissionRequest`，僅當政策做出決定時，OpenClaw 才會傳回明確的允許或拒絕
決定。未做出決定的結果並不代表允許：Codex
會將其視為掛鉤未做決定，並轉而採用其自身的守護程式或使用者
核准路徑。

Codex app-server 核准模式預設會省略此原生掛鉤。除非
`permission_request` 明確包含在
`nativeHookRelay.events` 中，或由相容性執行階段安裝，否則皆適用此規則。

當操作人員針對 Codex 原生權限
要求選擇 `allow-always` 時，OpenClaw 會在有限的工作階段時段內記住該確切的提供者／工作階段／工具輸入／cwd
指紋。已記住的決定刻意僅適用於完全相符的情況：若命令、引數、工具承載資料或
cwd 有所變更，即會建立新的核准要求。

當 Codex 將 `_meta.codex_approval_kind` 標記為 `"mcp_tool_call"` 時，Codex MCP 工具核准資訊徵詢會透過 OpenClaw 的外掛核准
流程路由。Codex
`request_user_input` 會為發起要求的工作階段註冊一個不限定提供者的閘道問題。Control UI 會呈現閘道問題卡片；若只有一個非機密選項，且頻道支援，
則會使用具型別的頻道按鈕。按鈕點按、Control UI 回答，以及佇列中下一則純文字回覆，都會先解析同一筆閘道記錄，
OpenClaw 才會傳回 app-server 答案。
Codex 自動解析和嘗試中止會限制等待時間並取消該記錄。
機密問題會全程保留在帶有警告的文字回覆路徑上。其他 MCP
資訊徵詢要求皆會以封閉方式失敗。

關於承載這些提示的一般外掛核准流程，請參閱
[外掛權限要求](/zh-TW/plugins/plugin-permission-requests)。

## 佇列導向

執行中佇列導引會對應至 Codex app-server `turn/steer`。使用預設的 `messages.queue.mode: "steer"` 時，OpenClaw 會在設定的靜默時間範圍內，批次處理導引模式的聊天訊息，並依抵達順序將它們作為單一 `turn/steer` 要求傳送。

Codex 審查和手動壓縮回合可能會拒絕同一回合的導引。在這種情況下，OpenClaw 會等待執行中的作業完成，再開始處理提示。若訊息預設應進入佇列而非進行導引，請使用 `/queue followup` 或 `/queue collect`。請參閱[導引佇列](/zh-TW/concepts/queue-steering)。

## Codex 意見回饋上傳

在原生 Codex 控制框架上，當工作階段的 `/diagnostics [note]` 獲得核准時，OpenClaw 也會針對相關的 Codex 執行緒呼叫 Codex app-server `feedback/upload`，其中包含每個列出執行緒的記錄，以及可用時由其衍生的 Codex 子執行緒。

上傳會透過 Codex 的一般意見回饋路徑傳送至 OpenAI 伺服器。如果該 app-server 已停用 Codex 意見回饋，此命令會傳回 app-server 錯誤。完成後的診斷回覆會列出已傳送執行緒的頻道、OpenClaw 工作階段 ID、Codex 執行緒 ID，以及本機 `codex resume <thread-id>` 命令。

如果你拒絕或忽略核准要求，OpenClaw 不會輸出這些 Codex ID，也不會傳送 Codex 意見回饋。此上傳不會取代本機閘道診斷匯出。關於核准、隱私權、本機套件組合和群組聊天行為，請參閱[診斷匯出](/zh-TW/gateway/diagnostics)。

只有在你要為目前附加的執行緒上傳 Codex 意見回饋，而不需要完整的閘道診斷套件組合時，才使用 `/codex diagnostics [note]`。

## 壓縮與逐字稿鏡像

當選取的模型使用 Codex 控制框架時，原生執行緒壓縮由 Codex app-server 負責。OpenClaw 不會為 Codex 回合執行預檢壓縮、不會以內容引擎壓縮取代 Codex 壓縮，也不會在無法啟動原生壓縮時，改用 OpenClaw 或公開的 OpenAI 摘要功能。OpenClaw 會保留逐字稿鏡像，以供頻道歷史記錄、搜尋、`/new`、`/reset`，以及未來切換模型或控制框架時使用。

明確的壓縮要求（例如 `/compact` 或由外掛要求的手動壓縮操作）會使用 `thread/compact/start` 啟動原生 Codex 壓縮。OpenClaw 會保持要求和共用用戶端租約開啟，直到 Codex 發出相符的 `contextCompaction` 完成項目，接著將壓縮回合回報為已完成。如果該終止回合超過設定的壓縮逾時時間，OpenClaw 會要求中斷原生回合。在 Codex 回報終止狀態或確認中斷 RPC 之前，租約與每個執行緒的壓縮柵欄都會保持鎖定。如果 Codex 未在中斷寬限期內確認，OpenClaw 會先汰除連線，再釋放柵欄。遠端連線也會解除相符的執行緒繫結，避免後續作業與尚未確認的遠端回合重疊。已汰除連線上的其他回合會失敗，並可在新的用戶端上重試。用戶端關閉、要求取消或壓縮回合失敗時，都會傳回失敗的操作。因內容壓力而自動執行的壓縮由 Codex 負責；OpenClaw 只會針對手動要求的觸發條件啟動原生壓縮。

當內容引擎要求 Codex 執行緒啟動投影時，OpenClaw 會將工具呼叫的名稱與 ID、輸入結構，以及經過遮蔽的工具結果內容投影至新的 Codex 執行緒。它不會將原始工具呼叫引數值複製到該投影中。

鏡像包含使用者提示、助理的最終文字，以及 app-server 發出時的精簡 Codex 推理或計畫記錄。OpenClaw 會記錄原生壓縮的開始和終止狀態，但不會公開人類可讀的壓縮摘要，也不會提供可稽核的清單來列出 Codex 在壓縮後保留了哪些項目。

由於 Codex 擁有標準的原生執行緒，`tool_result_persist` 不會重寫 Codex 原生工具結果記錄。它僅在 OpenClaw 寫入由 OpenClaw 擁有的工作階段逐字稿工具結果時套用。

## 媒體與傳遞

OpenClaw 仍負責媒體傳遞和媒體供應商選擇。圖片、影片、音樂、PDF、TTS 和媒體理解會使用相符的供應商／模型設定，例如 `agents.defaults.mediaModels.image`、`agents.defaults.mediaModels.video`、`pdfModel` 和 `tts`。

文字、圖片、影片、音樂、TTS、核准和訊息工具輸出會繼續透過一般 OpenClaw 傳遞路徑處理；媒體生成不需要舊版執行階段。當 Codex 發出具有 `savedPath` 的原生圖片生成項目時，即使 Codex 回合沒有助理文字，OpenClaw 仍會透過一般回覆媒體路徑轉送該確切檔案。

## 相關內容

- [Codex 控制框架](/zh-TW/plugins/codex-harness)
- [Codex 控制框架參考](/zh-TW/plugins/codex-harness-reference)
- [Codex 監督](/zh-TW/plugins/codex-supervision)
- [原生 Codex 外掛](/zh-TW/plugins/codex-native-plugins)
- [外掛掛鉤](/zh-TW/plugins/hooks)
- [代理程式控制框架外掛](/zh-TW/plugins/sdk-agent-harness)
- [診斷匯出](/zh-TW/gateway/diagnostics)
- [軌跡匯出](/zh-TW/tools/trajectory)
