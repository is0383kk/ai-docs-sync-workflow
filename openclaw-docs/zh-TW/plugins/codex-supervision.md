---
read_when:
    - 你希望 Codex Desktop 或命令列介面工作階段顯示在 OpenClaw 中
    - 你需要從已儲存或閒置的本機 Codex 工作階段建立分支，或將其封存
    - 你正在公開已配對節點中的 Codex 工作階段與對話記錄歷史
sidebarTitle: Codex supervision
summary: 瀏覽 OpenClaw 節點上未封存的原生 Codex 工作階段與分頁顯示的逐字稿
title: 監督 Codex 工作階段
x-i18n:
    generated_at: "2026-07-26T07:57:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f365e3207dff092c3dfd8f7588d60d70a16f0cce484991eb4ab3fc0bd15f8051
    source_path: plugins/codex-supervision.md
    workflow: 16
---

Codex 監督是官方 `codex` 外掛的一項選用功能。它會在一般工作階段側邊欄與 Chat 窗格中，顯示來自閘道電腦以及已選擇啟用之配對電腦的未封存 Codex 命令列介面、VS Code、Atlas 和 ChatGPT 來源工作階段。

初始版本刻意將所有權範圍維持在最小：

- 已儲存或閒置的本機工作階段，可以從其有界的持久化使用者與助理歷程建立鎖定模型的 OpenClaw Chat。第一則訊息會啟動原生快照分支，接著使用 Codex App Server 為該分支選取的確切模型與供應商，啟動完整的 Codex 控制框架執行緒。後續輪次會還原標準原生執行緒的持久化配對，而受監督的繫結會防止 OpenClaw 替換成其他執行階段、模型或備援。個別的原生 Codex 控制項仍可變更該持久化配對。已建立的分支會開啟其現有的 Chat。
- 從另一個 Codex 程序探索到的已儲存工作階段，其即時活動狀態未知。它可以建立分支；或者，只有在操作員確認沒有其他 Codex 用戶端正在使用它之後，才能將其封存。
- 作用中的來源會保持可見，但在目前輪次結束前無法建立分支或封存。如果它已有受監督的 Chat，**開啟 Chat** 仍然可用。
- 配對節點上的工作階段，會透過有界、以游標分頁的 App Server 讀取來公開其持久化逐字稿。遠端接續需要未來的串流節點橋接；遠端封存則還需要執行器所有權租約或同等的隔離機制。
- 封存的工作階段不會列出。只有在操作員確認沒有其他 Codex 用戶端正在使用已儲存或閒置的本機工作階段之後，才能將其封存。

## 開始之前

- 在閘道上安裝官方 `@openclaw/codex` 外掛。啟用 Codex 功能時，OpenClaw macOS 應用程式可以安裝此外掛；命令列介面安裝則可以執行 `openclaw plugins install @openclaw/codex`。
- 在你想要列出其工作階段的每部電腦上，安裝並登入 Codex Desktop 或 Codex 命令列介面。
- 將遠端電腦配對為 OpenClaw 節點。每部電腦都必須在本機選擇啟用；只在閘道上啟用監督，並不會授權其他節點。
- 使用由擁有者控制的閘道。工作階段標題、工作目錄和 Git 分支可能會揭露敏感的專案資訊。

## 啟用監督

引導式 `openclaw onboard` 與 macOS 初次執行設定，會在偵測到原生 Codex 安裝並成功啟用所選的推論後端後，嘗試安裝及啟用 Codex 監督。Codex 不需要是主要後端。當此外掛的機會式啟用成功時，監督功能即可使用。監督首次連線時會檢查 App Server 的可用性。明確停用 Codex 外掛或政策封鎖會阻止機會式啟用，而現有的明確 `supervision.enabled: false` 會停用面向代理程式的監督工具；只要 Codex 外掛處於作用中，操作員目錄就會保持註冊，除非 `sessionCatalog.enabled: false` 將其停用。這個獨立開關會從此主機移除配對節點目錄的列出／讀取命令，同時保持 Codex 供應商、控制框架及面向代理程式的監督政策不變。現有安裝可以手動啟用相同功能：

在 `openclaw.json` 中啟用 `codex` 外掛及其監督功能：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          supervision: {
            enabled: true,
          },
        },
      },
    },
  },
}
```

如果存在 `plugins.allow`，請納入 `codex`。變更外掛啟用狀態後，請重新啟動閘道。

若未明確設定 `appServer` 連線設定，監督會針對原生使用者 Codex 主目錄，使用獨立的受管理 stdio 監督連線。一般 Codex 控制框架預設仍限定於代理程式範圍。如此可讓原生工作階段顯示於兩個應用程式中，而不會使一般 OpenClaw 輪次共用原生 Codex 狀態。如果控制框架也應共用該狀態，請明確設定 `appServer.homeScope: "user"`。監督會遵循明確的 `appServer` 連線設定，而不會以其本機使用者主目錄預設值取代這些設定。

從 **Codex** 側邊欄群組採用的 Chat 並非一般控制框架工作階段。其私有監督繫結會使用監督連線進行來源讀取、標準分支建立、歷程注入，以及每個後續輪次。使用預設本機連線時，這會保留原生使用者 Codex 主目錄、驗證和供應商設定，而不會變更其他工作階段的預設值。受監看且已採用的 Chat 也會參與[工作階段狀態感知](/zh-TW/concepts/session-state)。

對預設本機監督連線而言，儲存區會與原生 Codex 用戶端共用。OpenClaw 不會假設另一個用戶端共用相同的即時 App Server 程序，而原生狀態所有權限定於程序本機。因此，它會將監督 App Server 回報為 `notLoaded` 的執行緒視為**已儲存／活動狀態未知**，而非閒置。

在每個應顯示其工作階段的無周邊設備節點主機上套用相同的選用設定。原生 OpenClaw macOS 應用程式向配對閘道公告其 Codex 目錄時，會讀取相同的本機設定。該配對的原生 Mac 目錄僅支援預設或明確的 `appServer.transport: "stdio"`，且 `appServer.homeScope: "user"` 未設定或已明確設定。該 stdio 程序會遵循 `command`、`args` 和 `clearEnv`。如果 Mac 設定選取 `"unix"`、`"websocket"` 或 `homeScope: "agent"`，應用程式不會公告目錄功能或命令；過時的直接叫用會失敗，而不是公開使用者 Codex 主目錄或產生另一個本機 stdio App Server。

新公告的節點命令會變更節點已核准的命令介面。請從閘道主機核准更新：

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

未封存的 Codex 工作階段也會顯示於主要控制介面側邊欄，並依主機分組。選取其中一個即可讀取其持久化逐字稿。檢視器使用最新的 Codex `thread/turns/list` API 搭配 `itemsView: "full"`，每次要求最多載入 20 個輪次；**載入較舊的逐字稿項目**會沿用最新頁面中的不透明 App Server 游標。已載入的頁面會依時間順序呈現。檢視器絕不會載入無界的 `thread/read` 歷程。超過 20 MiB 傳輸安全上限的頁面會以關閉方式失敗，避免危及節點或閘道連線。

在一般工作階段側邊欄中開啟 **Codex** 群組。它會列出相同的工作階段，並依主機分組。**載入更多工作階段**會從每個仍有較舊資料列的主機附加下一頁，且這些附加的資料列會在側邊欄定期重新整理後保留。每個主機都會在其自身的原生清單完成後立即顯示。可見頁面會在節點連線狀態變更、重新取得焦點時進行協調，且最多每 30 秒協調一次；結果有變更時，會更快進行後續協調。因此，在 Codex Desktop、命令列介面或其他原生用戶端中建立的工作階段，無須完整重新載入頁面即可顯示。第一頁會遵循 Codex 自身最近更新優先的順序，因此新建立的原生工作階段會立即符合顯示資格。
由於原生搜尋也可以比對逐字稿預覽，因此每個傳回的搜尋頁面會掃描每部主機上數量有限的原生頁面，而不是將查詢傳送至 App Server。

主機可用性與執行緒狀態彼此獨立。**離線**或**無法使用**描述的是主機重新整理狀態；無法使用的主機不會傳回新的工作階段資料列，也不會將執行緒的原生狀態變更為 `offline`。工作階段資料列使用 `idle`、`active`、`notLoaded` 或錯誤等 Codex 狀態。某部主機失敗時，不會隱藏正常主機的結果。

側邊欄警告會包含目錄錯誤代碼與安全的底層閘道錯誤。開啟 **Settings > Automation > Plugins > Codex > Native Session Discovery**，即可停用探索而不停用 Codex。若為 `NODE_LIST_FAILED`，請比較 `openclaw nodes list` 與 **Settings > Devices**；詳細原因會指出需要修復的配對儲存區、節點登錄、權限或閘道生命週期失敗。

## 使用操作員命令列介面

終端命令列介面會公開相同的未封存目錄，以及僅限閘道本機的分支和封存動作：

```bash
openclaw codex sessions [--search <text>] [--host <id>] [--limit <count>] [--cursor <cursor>] [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
openclaw codex continue <thread-id> [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
openclaw codex archive <thread-id> --confirm-no-other-runner [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
```

`openclaw codex sessions` 選項：

- `--search <text>` 會以不區分大小寫的方式搜尋工作階段標題。
- `--host <id>` 會將回應限制在單一穩定的目錄主機，例如 `gateway:local` 或 `node:<node-id>`。
- `--limit <count>` 設定每部主機 1 至 100 個資料列；預設值為 50。
- `--cursor <cursor>` 會接續單一主機頁面，因此需要 `--host`。
- `--json` 會輸出結構化閘道回應。

這三個命令都會從閘道用戶端繼承 `--url`、`--token` 和 `--timeout <ms>`。工作階段清單的預設逾時為 75,000 ms，讓冷啟動的配對節點目錄有時間完成；接續和封存的預設逾時為 30,000 ms。它們也會公開共用的 `--expect-final` 開關，但這不會變更這些一元監督 RPC。每個命令都需要 `operator.write` 閘道範圍。
每個子命令都提供標準 `-h, --help` 輸出。
沒有「已封存」或「包含已封存」選項。`sessions` 可以列出配對主機，但 `continue` 和 `archive` 一律以 `gateway:local` 為目標；配對資料列僅供列出。封存一律需要 `--confirm-no-other-runner`。

這些 shell 命令不同於 Chat 內的 `/codex` 執行階段命令。
`/codex threads [filter]` 會列出目前對話連線可用的 App Server 執行緒。`/codex sessions --host <node>` 會列出單一節點上可恢復的 Codex 命令列介面工作階段檔案，而非監督叢集目錄。`/codex
resume` 和 `/codex bind` 會附加目前對話，而不是建立安全的受監督分支；鎖定模型的受監督 Chat 會拒絕這些繫結變更。不存在 `/codex continue` 或 `/codex archive` 執行階段命令。

## 從本機工作階段建立分支

在閘道電腦的已儲存或閒置資料列上選擇**以分支接續**。OpenClaw 會建立一般 Chat 項目，鏡像從來源到最後一個終止持久化輪次（已完成、中斷或失敗）為止的有界使用者與助理歷程、記錄待處理的控制框架分支，然後開啟 Chat。通用模型選擇器會被鎖定，但尚未選取具體模型或供應商。來源不會恢復，標準控制框架執行緒也尚未啟動。重複執行此動作會開啟現有 Chat，而不是建立另一個分支。

鏡像會保留符合全部三項限制的最新可見尾端：最多 200 則使用者或助理訊息、UTF-8 文字總計 512 KiB，以及每則訊息 64 KiB。過大的訊息會使用標記截斷，達到上限時會省略較舊的訊息。圖片或本機圖片輸入會變成字面值 `[Image attachment]` 預留位置；不會複製圖片資料與本機路徑。

傳送第一則一般聊天訊息以開始工作。Codex 控制框架會安裝真正的
核准、資訊徵詢、事件與傳遞處理常式。它會在監督連線上使用暫時的
原生分支來固定來源快照，而不提供模型或供應商覆寫。Codex App Server 會從
目前的原生設定中選取兩者，並傳回實際選擇。在同一條
連線上，OpenClaw 會在其目前工作目錄與執行階段政策下，以該傳回配對完整啟動標準的 `appServer` 來源控制框架執行緒，
注入有界的可見歷程，並封存暫時分支。標準執行緒
具備完整的 OpenClaw 控制框架工具介面。這是可見歷程分支，而不是
完整的原生執行展開複本：來源推理、工具呼叫與工具結果都會
省略。此回合及之後的每個回合都會留在受監督的 Codex 連線上，
而不是使用另一個 OpenClaw 模型執行階段或一般的代理程式主目錄控制框架。

傳回的選擇並不能證明來源過去所用的模型。如果
目前的原生設定與來源上一回合記錄的模型不同，
Codex 會發出其一般的模型差異警告。OpenClaw 會使用
傳回的配對啟動標準執行緒。Codex 會保存該標準
執行緒的原生模型與供應商，而之後的繼續作業會保留它們，因為
OpenClaw 會省略模型與供應商覆寫。如果標準執行緒透過
另一個原生 Codex 控制介面變更，OpenClaw 會接受 Codex 保存的
選擇。OpenClaw 絕不會以其外層模型或後援鏈取代它。

受監督且鎖定模型的聊天無法刪除、切換模型、使用 `/new`
或 `/reset`、叫用閘道工作階段重設動作，或使用一般的
**分支工作階段**動作。變更 `/codex model <model>`、`/codex
bind`、`/codex resume`（包括具有 `--bind here` 的節點工作階段），以及
`/codex detach` 或 `/codex unbind` 也會遭到拒絕，因為這些動作會取代
或清除鎖定的原生繫結。`/codex model` 查詢以及 `/codex fast`、
`/codex permissions` 和 `/codex threads` 仍可使用。若要使用不同模型或全新執行緒，
請啟動另一個一般工作階段。

請讓此聊天維持啟用監督。如果停用監督，或其
儲存的連線繫結變得無法使用或不一致，該回合會
採取失敗關閉，而不會移至一般的代理程式主目錄工作階段。

停用或解除安裝 `codex` 外掛不會解除該擁有權，也不會
讓聊天可改用其他模型。鎖定的聊天會保留但
無法使用；請重新安裝或重新啟用相同外掛，並重新啟動閘道以
繼續使用。這項刻意的失敗關閉行為可防止保留資料清理或
暫時的外掛中斷在未通知的情況下使原生繫結成為孤立狀態。

`codex_threads` 代理程式工具遵循相同邊界。它無法附加
不同的分支，也無法封存聊天所繫結的原生執行緒。清單與僅中繼資料的
讀取仍可使用。原始逐字稿讀取需要 `allowRawTranscripts`。
停用原始存取時，`codex_threads` 也會拒絕清單搜尋，因為
原生搜尋包含逐字稿預覽；控制介面與操作員命令列介面
仍提供有界的僅標題搜尋。重新命名、取消封存、分離式分支，以及
封存不相關且無擁有者的執行緒需要
`allowWriteControls`。這兩個選項都無法繞過鎖定繫結。

OpenClaw 在僅列出來源執行緒或顯示待處理聊天時，
不會訂閱或回覆核准要求。在第一個回合啟動獨立的標準
控制框架執行緒，可讓另一個 Codex 程序繼續擁有
來源，而不會建立相互競爭的執行展開寫入者。

原始的命令列介面、VS Code、Atlas 或 ChatGPT 來源仍會顯示於原生
用戶端與 OpenClaw 目錄中。標準分支會儲存為原生
Codex 執行緒，但其來源種類為 `appServer`；Codex Desktop 或其他
原生用戶端可能會篩選該來源種類，因此無法保證分支本身
會顯示於每個原生歷程檢視中。

OpenClaw 的 App Server 回報為作用中的資料列無法啟動新分支。請等候
目前回合完成，然後重新整理目錄。Codex App Server
會在單一程序內序列化變更，但不提供獨佔的
跨程序執行器或核准擁有者租約。

對於**已儲存／活動狀態不明**的資料列，聊天鏡像與第一回合快照
固定作業會使用 Codex 截至最後一個已保存終止回合的狀態。來源
執行緒不會被繼續、中斷或封存。如果另一個程序有
正在進行的回合，其最新的進行中工作可能不會出現在分支中。

## 封存本機工作階段

在已儲存或閒置的閘道本機資料列上選擇 **Archive**，然後確認沒有
其他 Codex 用戶端或 OpenClaw 執行器正在使用該執行緒或其衍生的
後代執行緒。OpenClaw 會重新讀取程序本機狀態，僅在狀態為
`idle` 或 `notLoaded` 時繼續，呼叫原生 Codex 封存作業，並從
未封存清單中移除該工作階段。原生 Codex 也會嘗試封存
該執行緒衍生的後代執行緒。

如果重新讀取回報工作階段為作用中或處於錯誤狀態、它屬於已配對節點，
或新建立的受監督聊天仍有來自該來源的待處理分支，
便無法使用封存。請先傳送聊天的第一則訊息以具現化其標準分支，再封存來源。
如果 OpenClaw 知道某個作用中繫結擁有
確切目標執行緒或任何未封存的衍生後代執行緒，封存也會遭到封鎖。OpenClaw 會逐頁追蹤
實驗性的 Codex 後代查詢；無效回應、
要求失敗、重複的游標或執行緒，或耗盡安全限制，都會導致
封存遭到拒絕。

讀取、後代列舉與封存要求並非單一條件式
作業，因此兩者之間仍可能啟動回合。App Server 狀態也
不會在獨立程序之間共用。因此，確認是
未知用戶端與該競態狀況的安全邊界：請在確認前結束或以其他方式驗證
所有其他用戶端。使用 Codex Desktop、Codex 命令列介面，或經擁有者授權的原生執行緒管理流程
還原已封存的執行緒；取消封存後它會重新出現。

```bash
codex unarchive <thread-id>
```

## 瞭解已配對節點的限制

已配對節點會公開有版本控制的唯讀
`codex.appServer.threads.list.v1` 與
`codex.appServer.thread.turns.list.v1` 命令。可使用
Codex 命令列介面的原生節點主機也會公開允許清單中的 `codex.terminal.resume.v1`
命令。閘道會接收正規化的
中繼資料與明確要求的有界逐字稿分頁，絕不會接收原始 App Server
端點。在操作員終端中開啟資料列時，會在擁有該資料列的主機上執行 `codex resume <thread-id>`
並轉送該命令的 PTY；這不會公開一般
Shell 或由閘道提供的 argv。

終端轉送不提供控制框架繼續作業或封存擁有權
合約。因此，遠端資料列仍保持可見，但不提供 **Continue** 或
**Archive**，即使遠端執行緒處於閒置狀態亦然。請透過 **Open in terminal**
在該電腦上使用 Codex，或使用未來具備安全
執行器擁有權邊界的繼續作業流程。

## 中繼資料與權限

目錄資料列可能包括：

- 執行緒與工作階段識別碼
- 標題與工作目錄
- 目前狀態與作用中的等候旗標
- 建立、更新與活動時間戳記
- 來源、模型供應商、Codex 命令列介面版本與 Git 分支

目錄投影會排除逐字稿預覽、回合、執行展開路徑、
Codex 主目錄路徑、Git 遠端、提交 SHA 與原始 App Server 錯誤。目錄
存取與控制介面逐字稿讀取需要 `operator.write` 閘道
範圍，因為機群彙總使用標準的 `node.invoke` 路徑，即使
兩個節點命令都是唯讀亦然。

`supervision.allowRawTranscripts` 與 `supervision.allowWriteControls` 管控
自主代理程式與獨立 MCP 工具。兩者預設皆為 `false`。啟用
監督時，除非允許原始逐字稿，否則 `codex_threads` 會從
清單與僅中繼資料的讀取結果中移除逐字稿預覽與回合；
包含回合的讀取會採取失敗關閉。每個分支、重新命名、封存與取消封存
都需要寫入控制。這些選項不會管制已驗證的控制介面
逐字稿檢視，也不會繞過繫結、主機、狀態或確認檢查。

### 相容性工具

官方 `codex` 外掛會為現有代理程式與獨立 MCP 用戶端保留五個已發布的 Supervisor 工具名稱：

- `codex_endpoint_probe`
- `codex_sessions_list`
- `codex_session_read`
- `codex_session_send`
- `codex_session_interrupt`

`codex_sessions_list` 預設僅限已載入項目；沒有 `loaded_only`
參數。設定 `include_stored: true`，也可從
Codex 的狀態資料庫讀取未封存的已儲存資料列。選用的 `max_stored_sessions` 上限預設為 200，
每個端點接受 1 至 1,000 個資料列。它不會限制已載入資料列。
若無原始逐字稿權限，清單結果會省略衍生自逐字稿的名稱、
預覽與詳細端點錯誤。
`codex_session_read` 需要 `allowRawTranscripts`；`include_turns: true`
還會向 Codex 要求回合。

`codex_session_send` 與 `codex_session_interrupt` 需要
`allowWriteControls`。傳送接受 `mode: "auto" | "start" | "steer"`，但
`"start"` 一律遭到拒絕，且 `"auto"` 與 `"steer"` 都只能引導
可讀取的作用中回合。閒置執行緒會遭到拒絕，並提示使用 **Codex
Sessions**；完整控制框架會在繼續作業前於該處安裝核准與工具處理常式。
中斷同樣需要可讀取的作用中回合。這些工具
不會繼續或啟動閒置的來源執行緒。

`openclaw doctor --fix` 會將已淘汰的 `codex-supervisor` 項目、其端點
與權限欄位，以及外掛允許／拒絕政策參照移至官方
`codex` 外掛，而不會覆寫明確的標準設定。獨立的
相容性 MCP 配接器會繼續從該
外掛載入相同的五個工具；舊版政策環境變數僅適用於該受信任的
配接器內。

關於每個監督設定欄位，請參閱
[Codex 控制框架參考](/zh-TW/plugins/codex-harness-reference#supervision)。

## 疑難排解

**未顯示任何工作階段：**請確認已安裝 `@openclaw/codex`、外掛與
`supervision.enabled` 都是 true、目前的外掛允許清單允許
`codex`，且工作階段尚未封存。變更啟用狀態後，請重新啟動閘道或節點。

**Continue 已停用：**未對應的資料列處於作用中、屬於已配對節點、
其主機離線，或有另一個動作正在等待處理。閘道本機的已儲存與閒置
資料列會提供 **Continue as branch**，而不是不安全地接管確切執行緒。已有受監督聊天的資料列
會提供 **Open Chat**。

**Archive 已停用：**在完成「沒有其他執行器」確認後，已儲存／活動狀態不明與
閒置的閘道本機資料列可使用封存。作用中、錯誤、
離線、已配對節點、待處理分支，以及已知具有確切繫結擁有者的資料列，在封存方面仍為
唯讀。

**已封存的工作階段消失：**這是預期行為。監督頁面
沒有封存項目檢視。請執行 `codex unarchive <thread-id>` 或使用 Codex Desktop 讓它
再次顯示。

**仍保留舊的 `codex-supervisor` 設定：**請執行 `openclaw doctor --fix`。Doctor
會將已淘汰的外掛項目與相關外掛政策參照移至
`plugins.entries.codex.config.supervision`，而不會覆寫明確的 Codex
設定。

## 相關內容

- [Codex 控制框架](/zh-TW/plugins/codex-harness)
- [Codex 控制框架參考](/zh-TW/plugins/codex-harness-reference)
- [Codex 控制框架執行階段](/zh-TW/plugins/codex-harness-runtime)
- [Codex 監督架構](/zh-TW/specs/codex-supervision)
- [節點](/zh-TW/nodes)
- [閘道安全性](/zh-TW/gateway/security)
