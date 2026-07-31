---
read_when:
    - 透過本機控制 API 編寫代理程式瀏覽器指令碼或進行偵錯
    - 尋找 `openclaw browser` 命令列介面參考文件
    - 新增使用快照和參照的自訂瀏覽器自動化
summary: OpenClaw 瀏覽器控制 API、命令列介面參考與指令碼動作
title: 瀏覽器控制 API
x-i18n:
    generated_at: "2026-07-26T08:44:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 812358a5ad366e419413b78507d3620ea9f3981224bc8cc62fb512b87eaadd9b
    source_path: tools/browser-control.md
    workflow: 16
---

若需設定、組態與疑難排解資訊，請參閱[瀏覽器](/zh-TW/tools/browser)。
本頁是本機控制 HTTP API、`openclaw browser`
命令列介面，以及指令碼模式（快照、參照、等待、偵錯流程）的參考資料。

## 控制 API（選用）

閘道僅針對本機整合提供一組小型的迴路位址 HTTP API。
此獨立伺服器須選擇啟用——請在閘道服務環境中設定環境變數
`OPENCLAW_EAGER_BROWSER_CONTROL_SERVER=1`，
並重新啟動閘道，HTTP 端點才會可用。若未設定
此變數，瀏覽器控制執行階段仍可透過命令列介面和
代理程式工具運作，但不會有任何服務監聽迴路位址控制連接埠。

- 狀態／啟動／停止：`GET /`、`GET /doctor`、`POST /start`、`POST /stop`、`POST /reset-profile`
- 設定檔：`GET /profiles`、`POST /profiles/create`、`DELETE /profiles/:name`
- 分頁：`GET /tabs`、`POST /tabs/open`、`POST /tabs/focus`、`DELETE /tabs/:targetId`、`POST /tabs/action`
- 快照／螢幕截圖：`GET /snapshot`、`POST /screenshot`
- 動作：`POST /navigate`、`POST /act`
- 鉤子：`POST /hooks/file-chooser`、`POST /hooks/dialog`
- 下載：`POST /download`、`POST /wait/download`
- 權限：`POST /permissions/grant`
- 偵錯：`GET /console`、`POST /pdf`
- 偵錯：`GET /errors`、`GET /requests`、`GET /dialogs`、`POST /trace/start`、`POST /trace/stop`、`POST /highlight`
- 網路：`POST /response/body`
- 狀態：`GET /cookies`、`POST /cookies/set`、`POST /cookies/clear`
- 狀態：`GET /storage/:kind`、`POST /storage/:kind/set`、`POST /storage/:kind/clear`
- 設定：`POST /set/offline`、`POST /set/headers`、`POST /set/credentials`、`POST /set/geolocation`、`POST /set/media`、`POST /set/timezone`、`POST /set/locale`、`POST /set/device`

`POST /tabs/action` 是命令列介面內部用於
`browser tab` 子命令（`{"action":"new"|"label"|"select"|"close"|"list", ...}`）的批次形式；
直接撰寫指令碼時，建議優先使用上述用途單一的分頁路由。

所有端點都接受 `?profile=<name>`。`POST /start?headless=true` 會為本機受管理的設定檔要求
一次性無頭啟動，而不變更持久化的
瀏覽器組態；僅附加、遠端 CDP 與既有工作階段設定檔會拒絕
此覆寫，因為 OpenClaw 不會啟動那些瀏覽器程序。

對於分頁端點，`targetId` 是相容性欄位名稱。建議傳入來自
`GET /tabs` 或 `POST /tabs/open` 的 `suggestedTargetId`；標籤與 `tabId`
控制代碼（例如 `t1`）也可接受。原始 CDP 目標 ID 與唯一的原始
目標 ID 前綴仍然有效，但它們是可能變動的診斷控制代碼。

若已設定共用密鑰閘道驗證，瀏覽器 HTTP 路由也需要驗證：

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>`，或使用該密碼進行 HTTP 基本驗證

注意事項：

- 此獨立的迴路位址瀏覽器 API **不會**使用受信任 Proxy 或
  Tailscale Serve 身分標頭。
- 若 `gateway.auth.mode` 是 `none` 或 `trusted-proxy`，這些迴路位址瀏覽器
  路由不會繼承那些帶有身分資訊的模式；請將它們限制為僅能透過迴路位址存取。

### `/act` 錯誤契約

`POST /act` 針對路由層級驗證與
政策失敗使用結構化錯誤回應：

```json
{ "error": "<message>", "code": "ACT_*" }
```

目前的 `code` 值：

- `ACT_KIND_REQUIRED`（HTTP 400）：`kind` 遺失或無法辨識。
- `ACT_INVALID_REQUEST`（HTTP 400）：動作承載資料正規化或驗證失敗。
- `ACT_SELECTOR_UNSUPPORTED`（HTTP 400）：`selector` 與不支援的動作種類搭配使用。
- `ACT_EVALUATE_DISABLED`（HTTP 403）：`evaluate`（或 `wait --fn`）已被組態停用。
- `ACT_TARGET_ID_MISMATCH`（HTTP 403）：頂層或批次的 `targetId` 與要求目標衝突。
- `ACT_EXISTING_SESSION_UNSUPPORTED`（HTTP 501）：既有工作階段設定檔不支援此動作。

其他執行階段失敗仍可能傳回 `{ "error": "<message>" }`，但不含
`code` 欄位。

### Playwright 需求

部分功能（導覽／動作／AI 快照／角色快照、元素螢幕截圖、
PDF）需要 Playwright。若未安裝 Playwright，這些端點會傳回
明確的 501 錯誤。

沒有 Playwright 仍可使用的功能：

- ARIA 快照
- 當每個分頁的 CDP WebSocket 可用時，可使用角色樣式的無障礙快照（`--interactive`、`--compact`、
  `--depth`、`--efficient`）。這是用於檢查與參照探索的
  備援方案；Playwright 仍是主要的
  動作引擎。
- 當每個分頁的 CDP WebSocket 可用時，可擷取受管理 `openclaw` 瀏覽器的頁面螢幕截圖
- 可擷取 `existing-session`／Chrome MCP 設定檔的頁面螢幕截圖
- 可從快照輸出擷取以 `existing-session` 參照為基礎的螢幕截圖（`--ref`）

仍需要 Playwright 的功能：

- `navigate`
- `act`
- 依賴 Playwright 原生 AI 快照格式的 AI 快照
- 使用 CSS 選取器的元素螢幕截圖（`--element`）
- 完整的瀏覽器 PDF 匯出

元素螢幕截圖也會拒絕 `--full-page`；該路由會傳回 `fullPage is
not supported for element screenshots`。

若看到 `Playwright is not available in this gateway build`，表示封裝版
閘道缺少核心瀏覽器執行階段相依套件。請重新安裝或更新
OpenClaw，然後重新啟動閘道。若使用 Docker，還需要依照下方說明安裝 Chromium
瀏覽器二進位檔。

#### Docker Playwright 安裝

若你的閘道在 Docker 中執行，請避免使用 `npx playwright`（npm 覆寫衝突）。
對於自訂映像，請將 Chromium 直接建置至映像中：

```bash
OPENCLAW_INSTALL_BROWSER=1 ./scripts/docker/setup.sh
```

對於既有映像，請改用隨附的命令列介面安裝：

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

若要持久保存瀏覽器下載內容，請設定 `PLAYWRIGHT_BROWSERS_PATH`（例如
`/home/node/.cache/ms-playwright`），並確保 `/home/node` 已透過
`OPENCLAW_HOME_VOLUME` 或繫結掛載進行持久化。OpenClaw 會在 Linux 上自動偵測持久化的
Chromium。請參閱 [Docker](/zh-TW/install/docker)。

## 運作方式（內部）

一個小型迴路位址控制伺服器會接受 HTTP 要求，並透過 CDP 連線至 Chromium 系瀏覽器。進階動作（點擊／輸入／快照／PDF）會在 CDP 之上透過 Playwright 執行；若缺少 Playwright，則只能使用不依賴 Playwright 的操作。代理程式看到的是單一穩定介面，而其底層可自由切換本機／遠端瀏覽器與設定檔。

## 命令列介面快速參考

所有命令都接受 `--browser-profile <name>` 以指定特定設定檔，也接受 `--json` 以輸出機器可讀格式。

<AccordionGroup>

<Accordion title="基本操作：狀態、分頁、開啟／聚焦／關閉">

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep    # 新增即時快照探查
openclaw browser start
openclaw browser start --headless # 一次性啟動本機受管理的無頭瀏覽器
openclaw browser stop            # 也會清除僅附加／遠端 CDP 的模擬設定
openclaw browser reset-profile   # 將設定檔的瀏覽器資料移至垃圾桶
openclaw browser tabs
openclaw browser tab             # 目前分頁的捷徑
openclaw browser tab new
openclaw browser tab new --label research
openclaw browser tab label abcd1234 research
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://example.com
openclaw browser focus abcd1234
openclaw browser close abcd1234
```

</Accordion>

<Accordion title="設定檔：列出、建立、刪除">

```bash
openclaw browser profiles
openclaw browser create-profile --name research --color "#0066CC"
openclaw browser create-profile --name attach --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser delete-profile --name research
```

</Accordion>

<Accordion title="檢查：螢幕截圖、快照、主控台、錯誤、要求">

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref 12        # 或使用 --ref e12
openclaw browser screenshot --labels
openclaw browser snapshot
openclaw browser snapshot --format aria --limit 200
openclaw browser snapshot --interactive --compact --depth 6
openclaw browser snapshot --efficient
openclaw browser snapshot --labels
openclaw browser snapshot --urls
openclaw browser snapshot --selector "#main" --interactive
openclaw browser snapshot --frame "iframe#main" --interactive
openclaw browser snapshot --out snapshot.txt
openclaw browser console --level error
openclaw browser errors --clear
openclaw browser requests --filter api --clear
openclaw browser pdf
openclaw browser responsebody "**/api" --max-chars 5000
```

</Accordion>

<Accordion title="動作：導覽、點擊、輸入、拖曳、等待、求值">

```bash
openclaw browser navigate https://example.com
openclaw browser resize 1280 720
openclaw browser click 12 --double           # 或使用 e12 作為角色參照
openclaw browser click-coords 120 340        # 檢視區座標
openclaw browser type 23 "hello" --submit
openclaw browser press Enter
openclaw browser hover 44
openclaw browser scrollintoview e12
openclaw browser drag 10 11
openclaw browser select 9 OptionA OptionB
openclaw browser download e12 report.pdf
openclaw browser waitfordownload report.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref e12
openclaw browser upload media://inbound/file.pdf
openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
openclaw browser wait --text "Done"
openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"
openclaw browser evaluate --fn '(el) => el.textContent' --ref 7
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
openclaw browser highlight e12
openclaw browser trace start
openclaw browser trace stop
```

</Accordion>

<Accordion title="狀態：Cookie、儲存空間、離線、標頭、地理位置、裝置">

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url "https://example.com"
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set theme dark
openclaw browser storage session clear
openclaw browser set offline on
openclaw browser set headers --headers-json '{"X-Debug":"1"}'
openclaw browser set credentials user pass            # 使用 --clear 移除
openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"
openclaw browser set media dark
openclaw browser set timezone America/New_York
openclaw browser set locale en-US
openclaw browser set device "iPhone 14"
```

</Accordion>

</AccordionGroup>

注意事項：

- 面向代理程式的 `browser` 工具提供 `action=download`（必填的 `ref` 和
  `path`）以及 `action=waitfordownload`（選填的 `path`）。兩者都會傳回已儲存的
  下載 URL、建議的檔名，以及受防護的本機路徑。受管理的 Playwright 設定檔可使用明確的下載
  攔截；現有工作階段設定檔則會傳回不支援操作的錯誤。
- 優先使用不可分割的選擇器上傳：上傳時一併傳入觸發器 `--ref`，讓 OpenClaw 在單一請求中完成準備和點擊。若刻意要稍後觸發，仍支援僅含路徑的 `upload`。使用 `--input-ref` 或 `--element` 可直接設定檔案輸入欄位。`dialog` 是準備呼叫；請在觸發對話方塊的點擊／按鍵操作之前執行。如果某項操作開啟強制回應視窗，操作回應會包含 `blockedByDialog` 和 `browserState.dialogs.pending`；請傳入該 `dialogId` 以直接回應。在 OpenClaw 外部處理的對話方塊會顯示於 `browserState.dialogs.recent` 下。
- `click`/`type`/等操作需要來自 `snapshot` 的 `ref`（數字 `12`、角色參照 `e12`，或可操作的 ARIA 參照 `ax12`）。操作刻意不支援 CSS 選擇器。當可見視窗位置是唯一可靠的目標時，請使用 `click-coords`。
- 下載和追蹤路徑僅限於 OpenClaw 暫存根目錄：`/tmp/openclaw{,/downloads}`（備援：`${os.tmpdir()}/openclaw/...`）。
- `upload` 接受來自 OpenClaw 暫存上傳根目錄和
  OpenClaw 管理的傳入媒體檔案。受管理的傳入媒體可透過
  `media://inbound/<id>`、相對於沙箱的 `media/inbound/<id>`，或受管理傳入媒體
  目錄內的已解析路徑來參照。巢狀媒體參照、
  路徑周遊、符號連結、硬式連結和任意本機路徑仍會遭到拒絕。
- `upload` 也可以透過 `--input-ref` 或 `--element` 直接設定檔案輸入欄位。

當 OpenClaw 能夠確認替換分頁時，穩定的分頁 ID 和標籤會在 Chromium 原始目標替換後保留，
例如同一 URL 存在唯一的舊／新配對，或提交表單後單一舊分頁變成單一新分頁。具有相同 URL
且無法明確判定的替換項目會取得新的控制代碼。原始目標 ID 仍不穩定；
指令碼中請優先使用來自 `tabs` 的 `suggestedTargetId`。

快覽快照旗標：

- `--format ai`（使用 Playwright 時的預設值）：包含數字參照（`aria-ref="<n>"`）的 AI 快照。
- `--format aria`：包含 `axN` 參照的無障礙樹狀結構。Playwright 可用時，OpenClaw 會使用後端 DOM ID 將參照繫結至即時頁面，讓後續操作能使用這些參照；否則，請將輸出視為僅供檢查。
- `--efficient`（或 `--mode efficient`）：精簡角色快照預設。設定 `browser.snapshotDefaults.mode: "efficient"` 可將其設為預設值（請參閱[閘道設定](/zh-TW/gateway/configuration-reference#browser)）。
- `--interactive`、`--compact`、`--depth`、`--selector` 會強制產生包含 `ref=e12` 參照的角色快照。`--frame "<iframe>"` 會將角色快照範圍限制在 iframe 內。
- 使用 Playwright 時，`--labels` 會新增一張疊加參照標籤的螢幕擷取畫面
  （輸出 `MEDIA:<path>`），以及包含每個參照邊界方框的 `annotations` 陣列。
  在 `screenshot` 上，由 Playwright 支援的標籤可搭配 `--full-page`、
  `--ref` 和 `--element` 使用；在 `snapshot` 上，隨附的螢幕擷取畫面仍
  僅限可見視窗。現有工作階段／chrome-mcp 設定檔會在頁面螢幕擷取畫面上呈現疊加標籤，
  但不會傳回 `annotations`，也不會使用 Playwright
  的完整頁面／參照／元素投影輔助程式。若沒有 Playwright 或 chrome-mcp，
  則無法使用帶標籤的螢幕擷取畫面。
- `--urls` 會將探索到的連結目的地附加至 AI 快照。

## 快照與參照

OpenClaw 支援兩種「快照」樣式：

- **AI 快照（數字參照）**：`openclaw browser snapshot`（預設；`--format ai`）
  - 輸出：包含數字參照的文字快照。
  - 操作：`openclaw browser click 12`、`openclaw browser type 23 "hello"`。
  - 內部會透過 Playwright 的 `aria-ref` 解析參照。

- **角色快照（如 `e12` 的角色參照）**：`openclaw browser snapshot --interactive`（或 `--compact`、`--depth`、`--selector`、`--frame`）
  - 輸出：包含 `[ref=e12]`（以及選用的 `[nth=1]`）且以角色為基礎的清單／樹狀結構。
  - 操作：`openclaw browser click e12`、`openclaw browser highlight e12`。
  - 內部會透過 `getByRole(...)`（重複項目另搭配 `nth()`）解析參照。
  - 新增 `--labels`，即可包含一張疊加 `e12` 標籤的螢幕擷取畫面。在
    由 Playwright 支援的設定檔中，這也會傳回每個參照的邊界方框中繼資料
    （`annotations[]`）。
  - 當連結文字語意不明，且代理程式需要明確的
    導覽目標時，請新增 `--urls`。

- **ARIA 快照（如 `ax12` 的 ARIA 參照）**：`openclaw browser snapshot --format aria`
  - 輸出：以結構化節點呈現的無障礙樹狀結構。
  - 操作：當快照路徑能透過 Playwright 和 Chrome 後端 DOM ID 繫結
    參照時，`openclaw browser click ax12` 可正常運作。
- 如果 Playwright 無法使用，ARIA 快照仍可用於
  檢查，但參照可能無法操作。需要操作參照時，請使用 `--format ai`
  或 `--interactive` 重新擷取快照。
- 原始 CDP 備援路徑的 Docker 驗證：`pnpm test:docker:browser-cdp-snapshot`
  會使用 CDP 啟動 Chromium、執行 `browser doctor --deep`，並驗證角色
  快照包含連結 URL、由游標提升為可點擊的項目，以及 iframe 中繼資料。

參照行為：

- 參照**不會跨導覽保持穩定**；如果發生失敗，請重新執行 `snapshot` 並使用新的參照。
- 若能確認替換分頁，`/act` 會在操作觸發替換後傳回目前的原始 `targetId`。
  後續命令請繼續使用穩定的分頁 ID／標籤。
- 如果角色快照是使用 `--frame` 擷取，則角色參照的範圍會限定於該 iframe，直到下一次角色快照為止。
- 未知或過期的 `axN` 參照會快速失敗，而不會改由
  Playwright 的 `aria-ref` 選擇器處理。發生此情況時，
  請在同一分頁上擷取新的快照。

## 瀏覽器批次命令列介面

`openclaw browser batch` 會在一次 `/act` 呼叫中執行一組巢狀 `/act`
操作（與透過代理程式工具存取的 `kind="batch"` 執行階段相同），因此命令列介面
使用者和指令碼能將 `wait`、`click`、`type` 和
`evaluate` 等操作合併為單一可重播的計畫，無須為每個操作往返呼叫。
`actions[]` 中的每個項目都是 `BrowserActRequest`，也就是 `/act`
路由接受的封閉聯集（`click`、`clickCoords`、`type`、`press`、`hover`、
`scrollIntoView`、`drag`、`select`、`fill`、`resize`、`wait`、`evaluate`、
`close`、`batch`），而不是任意的 `openclaw browser` 子命令。`batch`
不支援 `profile="user"` 和其他現有工作階段（chrome-mcp）
設定檔；請在這些設定檔中個別傳送操作。

- 命令列介面：`openclaw browser batch --actions '<json>'`、`openclaw browser batch
--actions-file plan.json`，或使用 `openclaw browser batch --actions-file -`
  從標準輸入讀取 JSON 陣列。`--continue` 會設定 `stopOnError=false`；
  預設會在第一個錯誤時停止。`--target-id` 會將整個批次限制在
  一個分頁中。
- 參照生命週期：參照來自批次前執行的 `snapshot`（快照
  不是巢狀操作）。會變更頁面狀態的巢狀操作，例如觸發導覽的
  `click`，或變更 DOM 的 `evaluate`，可能會使先前的參照
  在批次剩餘部分失效。請將變更狀態的操作放在前面，或在重新擷取快照後
  拆分成後續批次。導覽與重新擷取快照會在批次外進行（`openclaw browser navigate` /
  `snapshot`），因為 `open`、`navigate` 和 `snapshot` 並不是 `/act` 類型。
- 目標 ID 衝突：巢狀操作可以省略 `targetId`，或重複請求層級的
  `targetId`；如果明確指定的巢狀 `targetId` 解析為不同分頁，
  系統會在執行任何操作前以 `ACT_TARGET_ID_MISMATCH` 拒絕。
  依設計，批次操作共用請求的分頁。
- 錯誤摘要：回應為 `{ "results": [{ "ok": true }, { "ok": false,
"error": "<message>" }, ...] }`，依序對應每個操作各一個項目。當
  `stopOnError` 為預設值時，陣列會在第一次失敗時結束；使用
  `--continue` 時則會涵蓋每個操作。任何失敗項目都會使命令列介面以
  非零狀態結束；傳入 `--json` 可為指令碼保留完整的有序回應。

## 等待功能強化

等待條件不僅限於時間／文字：

- 等待 URL（支援 Playwright glob）：
  - `openclaw browser wait --url "**/dash"`
- 等待載入狀態：
  - `openclaw browser wait --load networkidle`
  - 受管理的 `openclaw` 和原始／遠端 CDP 設定檔均支援此功能。使用 `existing-session` 驅動程式的設定檔（包括預設的 `user` 設定檔）會拒絕 `networkidle`；在這些設定檔中，請改用 `--url`、`--text`、選擇器或 `--fn` 等待條件。
- 等待 JS 述詞：
  - `openclaw browser wait --fn "window.ready===true"`
- 等待選擇器變為可見：
  - `openclaw browser wait "#main"`

這些條件可以合併使用：

```bash
openclaw browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## 偵錯工作流程

操作失敗時（例如「不可見」、「嚴格模式違規」、「遭遮蔽」）：

1. `openclaw browser snapshot --interactive`
2. 使用 `click <ref>` / `type <ref>`（互動模式中優先使用角色參照）
3. 如果仍然失敗：使用 `openclaw browser highlight <ref>` 查看 Playwright 的目標
4. 如果頁面行為異常：
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. 若要深入偵錯，請記錄追蹤：
   - `openclaw browser trace start`
   - 重現問題
   - `openclaw browser trace stop`（輸出 `TRACE:<path>`）

## JSON 輸出

`--json` 適用於指令碼和結構化工具。

範例：

```bash
openclaw browser --json status
openclaw browser --json snapshot --interactive
openclaw browser --json requests --filter api
openclaw browser --json cookies
```

JSON 中的角色快照包含 `refs`，以及一個小型 `stats` 區塊（行數／字元數／參照數／互動項目數），讓工具能判斷承載資料大小和密度。

## 狀態與環境調整選項

這些選項適用於「讓網站表現得像 X」的工作流程：

- Cookie：`cookies`、`cookies set`、`cookies clear`
- 儲存空間：`storage local|session get|set|clear`
- 離線：`set offline on|off`
- 標頭：`set headers --headers-json '{"X-Debug":"1"}'`（或位置參數形式 `set headers '{"X-Debug":"1"}'`）
- HTTP 基本驗證：`set credentials user pass`（或 `--clear`）
- 地理位置：`set geo <lat> <lon> --origin "https://example.com"`（或 `--clear`）
- 媒體：`set media dark|light|no-preference|none`
- 時區／語系：`set timezone ...`、`set locale ...`
- 裝置／可見視窗：
  - `set device "iPhone 14"`（Playwright 裝置預設）
  - `set viewport 1280 720`

## 安全性與隱私權

- openclaw 瀏覽器設定檔可能包含已登入的工作階段；請將其視為敏感資訊。
- `browser act kind=evaluate` / `openclaw browser evaluate` 和 `wait --fn`
  會在頁面內容中執行任意 JavaScript。提示詞注入可能會操控此行為。
  如果你不需要此功能，請使用 `browser.evaluateEnabled=false` 停用。
- `openclaw browser evaluate --fn` 接受函式原始碼、運算式或
  陳述式主體。陳述式主體會包裝為非同步函式，因此請使用
  `return` 指定你想取得的傳回值。當頁面端函式可能需要比預設求值逾時
  更長的時間時，請使用 `--timeout-ms <ms>`。
- 如需登入與反機器人相關注意事項（X/Twitter 等），請參閱[瀏覽器登入與 X/Twitter 發文](/zh-TW/tools/browser-login)。
- 請將閘道／節點主機維持為私有（僅限回送介面或 tailnet）。
- 遠端 CDP 端點具備強大權限；請透過隧道連線並加以保護。

嚴格模式範例（預設封鎖私人／內部目的地）：

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // 可選的完全相符允許項目
    },
  },
}
```

## 相關內容

- [瀏覽器](/zh-TW/tools/browser) - 概觀、設定、設定檔、安全性
- [瀏覽器登入](/zh-TW/tools/browser-login) - 登入網站
- [瀏覽器 Linux 疑難排解](/zh-TW/tools/browser-linux-troubleshooting)
- [瀏覽器 WSL2 疑難排解](/zh-TW/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
