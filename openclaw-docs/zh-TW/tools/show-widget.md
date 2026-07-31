---
read_when:
    - 你想要代理程式在網頁聊天、原生應用程式或 Discord 中呈現互動式結果
    - 你希望小工具按鈕將後續提示傳送到聊天中
    - 你想使用共用的設計權杖來設定小工具的主題
    - 你需要 `show_widget` 的輸入、安全性或保留合約
sidebarTitle: Show widget
summary: 在支援的聊天介面上顯示獨立的 HTML 小工具
title: 顯示小工具
x-i18n:
    generated_at: "2026-07-26T08:50:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 903adff1fadeb9d224d3e2d839c86082b5244e1e319255c8d3f6619344b749a3
    source_path: tools/show-widget.md
    workflow: 16
---

`show_widget` 是一項核心工具，可在使用者目前的介面上顯示獨立的 HTML 小工具。OpenClaw 會在控制介面，以及 iOS、Android、macOS 與 Linux 快速聊天的對話記錄中以行內方式呈現；Linux 儀表板使用瀏覽器控制介面。在已啟用 [Activities](/zh-TW/channels/discord-activities) 的 Discord 工作階段中，Discord 外掛會發布一個 **Open widget** 按鈕，以 Activity 形式啟動該小工具。

## 小工具的運作方式

代理程式呼叫 `show_widget` 時，OpenClaw 核心會將 `widget_code` 包裝在最小化的 HTML 文件中、儲存為 Canvas 文件，並傳回預覽控制代碼。控制介面會在沙箱化的 iframe 中呈現該控制代碼，而 iOS、Android、macOS 與 Linux 快速聊天則使用隔離的網頁檢視。完整聊天用戶端會在重新載入歷史記錄後還原小工具；快速聊天則會在目前回覆期間保留小工具。

在控制介面工作階段中，Canvas 小工具也可以釘選到工作階段儀表板。在工具呼叫中設定 `pin: true`，或對現有的對話記錄小工具使用 **釘選到儀表板**。釘選的 HTML 會在 MCP Apps 所使用的同一個專用來源、雙 iframe 沙箱主機後方執行；瀏覽器絕不會在不受信任的框架內解析小工具資料繫結。

對於瀏覽器嵌入，包裝文件會在小工具程式碼周圍注入四個小型主機橋接器：

- 尺寸回報器會將呈現內容的高度傳送至嵌入它的聊天介面，由聊天介面限制高度並調整 iframe（160 至 1200 像素）。
- 主機橋接器會定義舊版 `sendPrompt(text)` 輔助函式，以及結構化的 `openclaw.prompt`、`openclaw.state`、`openclaw.data` 和 `openclaw.cron` API。行內聊天提示會保留其私有訊息通道；儀表板 API 則使用與檢視票證繫結的請求通道。請參閱[互動式小工具](#interactive-widgets)和[儀表板功能](#dashboard-capabilities)。
- 主題橋接器會監聽控制介面目前的設計權杖，並在載入時及每次主題變更時，將其套用為 CSS 變數。
- 快照橋接器會在嵌入它的聊天介面請求匯出時，將目前的小工具文件呈現為 PNG。

其他所有內容都會留在框架內：文件會在不透明來源中，以嚴格的內容安全政策執行，因此小工具指令碼無法存取控制介面、閘道或網路。

只有當來源閘道用戶端宣告 `inline-widgets` 功能時，才能使用核心實作。控制介面與支援的原生應用程式會自動宣告此功能。對於需要自訂 TLS 葉節點憑證釘選的閘道連線，Linux 快速聊天會維持純文字模式，因為其平台 WebView 無法繫結該釘選。Discord 實作僅適用於已設定 Activities 的 Discord 工作階段。其他頻道執行不會收到 `show_widget`。

功能傳輸涵蓋嵌入式、Codex app-server，以及由命令列介面支援的模型後端。以授權認證的 MCP 呼叫端與直接 HTTP 工具叫用端仍會採取封閉式失敗，因為它們不會宣告用戶端功能。

## 設計系統

每個 Canvas 小工具都包含無類別基礎樣式表與一組小型權杖：

| 權杖                                                                                 | 用途                               |
| ------------------------------------------------------------------------------------- | ------------------------------------- |
| `--surface`                                                                           | 頁面層級介面色彩              |
| `--card`                                                                              | 卡片、按鈕與程式碼背景     |
| `--elevated`                                                                          | 浮凸表單控制項背景      |
| `--text`                                                                              | 預設內文與控制項文字         |
| `--text-strong`                                                                       | 標題與醒目數值         |
| `--muted`                                                                             | 次要文字與細微邊框     |
| `--border`                                                                            | 標準分隔線與卡片邊框  |
| `--border-strong`                                                                     | 強調控制項邊框                |
| `--accent`                                                                            | 連結與焦點環                 |
| `--accent-fill`                                                                       | 主要動作填滿色                   |
| `--accent-fg`                                                                         | 主要動作上的文字              |
| `--ok`                                                                                | 成功狀態                         |
| `--warn`                                                                              | 警告狀態                         |
| `--danger`                                                                            | 錯誤或破壞性狀態            |
| `--info`                                                                              | 資訊狀態                   |
| `--radius`                                                                            | 共用控制項與卡片圓角半徑 |
| `--font-body`                                                                         | 主機內文字型堆疊                  |
| `--font-mono`                                                                         | 主機等寬字型堆疊             |
| `--accent-subtle`、`--ok-subtle`、`--warn-subtle`、`--danger-subtle`、`--info-subtle` | 衍生的半透明狀態背景 |

未加類別的標題、段落、連結、按鈕、輸入欄位、選取欄位、文字區域、表格與程式碼區塊都會套用基礎樣式。輔助類別提供常見模式：

- `.card` 用於有邊框的內容介面
- `.badge` 搭配 `.ok`、`.warn`、`.danger` 或 `.info`，用於精簡的狀態標籤
- `.metric` 用於醒目的數值
- `.muted` 用於次要文字
- `.row` 用於可換行的水平版面配置
- `button.primary` 用於主要動作

控制介面會在小工具載入及主題變更時，發布包含使用中主題值的 `openclaw:widget-theme` 訊息。因此，小工具無須重新載入即可跟隨所有主題系列，包括 Claw、Knot、Dash 與自訂主題。在控制介面之外，包括原生應用程式與直接開啟時，小工具會使用由 `prefers-color-scheme` 選取的內建淺色或深色調色盤。

撰寫小工具時請遵循三項規則：

1. 所有色彩與背景都使用設計變數。請勿硬式編碼色彩值。
2. 讓頁面背景保持透明，使小工具融入其主機介面。
3. `--accent-fill` 最多只能用於一個主要動作。

**匯出：** 在網頁聊天中，開啟小工具卡片選單，即可將呈現後的小工具複製到剪貼簿，或下載為 PNG。沒有快照橋接器的舊版小工具文件會改為下載 HTML 檔案。

## 使用工具

兩種實作都使用相同的必填欄位：

<ParamField path="title" type="string" required>
  與行內預覽一同顯示，並作為託管文件標題的簡短標題。
</ParamField>

<ParamField path="widget_code" type="string" required>
  獨立的 HTML 或 SVG。對於行內小工具用戶端，修剪後以 `<svg` 開頭的輸入會以 SVG 模式呈現；最大長度為 262,144 個字元。Discord 接受最大 48 KiB 的完整 HTML 文件或 body 片段。
</ParamField>

Discord 也接受選用的 `button_label` 文字，作為 Activity 啟動按鈕。Canvas 結構描述刻意省略這個僅供 Discord 使用的欄位。

核心 Canvas 工具接受以下選用的儀表板放置欄位：

- `pin`：同時將小工具放置於工作階段儀表板。
- `name`：穩定的小工具名稱；預設為 `title` 的 slug。
- `tab`：目的地分頁 slug。
- `size`：`sm`、`md`、`lg`、`xl` 或 `full` 其中之一。
- `after`：應將小工具放置於其後的同層小工具名稱。
- `capabilities`：釘選小工具要求的存取權。`netOrigins` 包含確切的 HTTPS 來源；`tools` 包含 `prompt`、允許清單中的讀取繫結，或確切的 `cron.trigger:<jobId>` 動作。

核心結果包含 Canvas 預覽控制代碼，因此控制介面與支援的原生應用程式會直接從工具呼叫呈現小工具，並在重新載入歷史記錄後還原它。釘選結果也會保留看板小工具名稱，因此控制介面不會在重新載入對話記錄後提供重複釘選。Discord 會傳回已儲存的小工具與已發布訊息識別碼。

`discord_widget` 會繼續註冊為已淘汰的別名，保留一個版本。新的代理程式呼叫應使用 `show_widget`。

## 互動式小工具

在控制介面中，小工具指令碼可以驅動對話。包裝文件會定義全域 `sendPrompt(text)` 函式；呼叫此函式會將 `text` 提交至聊天，就像使用者自行輸入並傳送訊息。將它連接至按鈕或其他控制項，即可建立選擇器、測驗或逐層深入儀表板等互動流程。原生應用程式會呈現互動式小工具程式碼，但不會公開此聊天提示橋接器。

```html
<button onclick="sendPrompt('詳細顯示失敗的測試')">失敗的測試</button>
```

每個提示都會在框架邊界的兩側進行驗證：

- `sendPrompt` 需要小工具內的[暫時性使用者啟用](https://developer.mozilla.org/en-US/docs/Web/Security/User_activation)：它只能在使用者於小工具中按一下或按下按鍵後的幾秒內運作，因此請將它連接至按鈕及其他點擊目標——載入時自動呼叫不會有任何作用。橋接器會將傳送端點保留為自身私有，並在未公開使用者啟用功能的瀏覽器中採取封閉式失敗，因此小工具程式碼無法繞過檢查。
- 提示權限僅屬於原始小工具文件。受信任的橋接器會在小工具程式碼可以執行或導覽框架之前，將其通道端點提供給聊天介面；聊天介面只採用第一次提供的端點，而該通道會在文件導覽時隨文件失效。外部允許的嵌入 URL 絕不會被採用。
- 小工具框架必須在聊天對話記錄中可見並取得焦點——這是一項額外由主機觀察的訊號，用以確認使用者確實正在與此小工具互動。
- 文字經修剪後不得為空，且最多為 4,000 個字元。
- 以 `/` 開頭的提示會遭拒絕，因此小工具程式碼無法觸發 `/approve` 或 `/stop` 等聊天命令。
- 每個小工具文件在每個滾動分鐘內最多可傳送 10 個提示；超出的提示會被無聲捨棄。

接受的提示會以一般使用者訊息顯示在對話記錄中，並在擁有該小工具的工作階段內啟動一般代理程式回合。小工具沒有任何回饋通道：被捨棄的提示會無聲失敗，且小工具無法讀取代理程式的回覆。

## 儀表板功能

操作人員檢閱待處理卡片上顯示的宣告後，釘選的小工具可使用一個與票證繫結的主機 API：

- `openclaw.prompt.send(text)` 需要暫時性的使用者啟用，並會在編輯器中發佈可見訊息。宣告並取得 `prompt` 工具授權後，便會略過每次點擊時的額外確認；驗證、焦點檢查與速率限制仍然適用。
- `openclaw.state.emit(payload)` 會新增工作階段通知。承載資料上限為 8 KiB，且用戶端在五秒內送出的相同內容會合併處理。
- `openclaw.data.read(bindingId, params?)` 僅在閘道解析。可授權的繫結為 `sessions.list`、`usage.status`、`usage.cost`、`cron.list`、`cron.status`、`agents.list` 和 `health`。
- `openclaw.cron.trigger(jobId)` 只有在已授予完全相符的 `cron.trigger:<jobId>` 功能時，才能立即執行現有工作。

網路存取與主機工具彼此獨立。請在 `capabilities.netOrigins` 中填入確切的 HTTPS 來源；核准後，只有這些來源會加入小工具的 `connect-src`。萬用字元、認證資訊、路徑、查詢字串和未宣告的來源仍會遭到封鎖。只有當常值連接埠是已宣告來源的一部分時，才允許使用。

## 安全性與儲存空間

小工具文件採用嚴格的內容安全政策。允許行內樣式與指令碼，但仍會封鎖外部資源載入。行內逐字稿小工具無法透過網路擷取內容。釘選的儀表板小工具只能從代理程式已宣告且操作人員已授權的確切 HTTPS 來源擷取內容。

即使全域嵌入模式為 `trusted`，控制介面的 iframe 一律省略 `allow-same-origin`，因此小工具指令碼無法讀取父應用程式來源。原生用戶端使用隔離且非持久性的網頁檢視，並封鎖離開代管小工具的導覽。核心文件主機也會透過 `Content-Security-Policy: sandbox allow-scripts` 回應標頭提供小工具，因此即使直接算繪，小工具仍會在不透明來源而非應用程式來源中執行。請只算繪你願意在該隔離框架中執行的小工具程式碼。

iframe 也遵循[`gateway.controlUi.embedSandbox`](/zh-TW/web/control-ui#hosted-embeds)。預設的 `scripts` 層級支援互動式小工具，同時維持來源隔離。

已接受的 WebRTC 資料通道輸出殘餘風險記錄於[儀表板架構](/zh-TW/web/dashboard-architecture#modeled-residual-webrtc-data-channels)。

Canvas 每個工作階段最多保留 32 個小工具（若沒有可用的工作階段，則以每個代理程式為單位）。建立其他小工具時，會移除該範圍內最舊的文件。

## 相關內容

- [控制介面代管嵌入項目](/zh-TW/web/control-ui#hosted-embeds)
- [Discord Activities](/zh-TW/channels/discord-activities)
- [Canvas 節點控制項](/zh-TW/plugins/reference/canvas)
- [閘道通訊協定用戶端功能](/zh-TW/gateway/protocol#client-capabilities)
