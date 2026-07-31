---
read_when:
    - 你使用 `openclaw browser`，並想查看常見工作的範例
    - 你想透過節點主機控制在另一台機器上執行的瀏覽器
    - 你想透過 Chrome MCP 連接至本機已登入的 Chrome
summary: '`openclaw browser` 的命令列介面參考（生命週期、設定檔、分頁、動作、狀態與偵錯）'
title: 瀏覽器
x-i18n:
    generated_at: "2026-07-26T08:18:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 62eb41248cda87cef96be7b0dfe3e0d36a9d3e1ee55c165bd8e3efd68d1e9a5e
    source_path: cli/browser.md
    workflow: 16
---

# `openclaw browser`

管理 OpenClaw 的瀏覽器控制介面並執行瀏覽器操作：生命週期、設定檔、分頁、快照、螢幕截圖、導覽、輸入、狀態模擬及偵錯。

相關資訊：[瀏覽器工具](/zh-TW/tools/browser)

## 常用旗標

- `--url <gatewayWsUrl>`：閘道 WebSocket URL（預設使用設定值）。
- `--token <token>`：閘道權杖（如有需要）。
- `--timeout <ms>`：要求逾時時間，以毫秒為單位（預設：`30000`）。
- `--expect-final`：等待閘道的最終回應。
- `--browser-profile <name>`：選擇瀏覽器設定檔（預設：`openclaw` 或 `browser.defaultProfile`）。
- `--json`：機器可讀輸出（在支援之處）。這是瀏覽器層級的選項，因此
  請將它放在子命令之前，以形成明確無歧義的形式，例如
  `openclaw browser --json status`。如果所選的子命令並未定義自己的
  `--json`，也可以放在尾端，例如
  `openclaw browser status --json`。

## 快速開始（本機）

```bash
openclaw browser profiles
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

代理程式可以使用 `browser({ action: "doctor" })` 執行相同的就緒狀態檢查。

## 快速疑難排解

如果 `start` 因 `not reachable after start` 而失敗，請先排解 CDP 就緒狀態問題。如果 `start` 和 `tabs` 成功，但 `open` 或 `navigate` 失敗，則瀏覽器控制平面運作正常，失敗通常是導覽遭到 SSRF 政策封鎖。

最小操作順序：

```bash
openclaw browser --browser-profile openclaw doctor
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw tabs
openclaw browser --browser-profile openclaw open https://example.com
```

詳細指引：[瀏覽器疑難排解](/zh-TW/tools/browser#cdp-startup-failure-vs-navigation-ssrf-block)

## 生命週期

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep
openclaw browser start
openclaw browser start --headless
openclaw browser stop
openclaw browser --browser-profile openclaw reset-profile
```

- `doctor --deep` 會加入即時快照探查：當基本 CDP 就緒狀態正常，但你想確認目前分頁可供檢查時，這會很有用。
- 對於正在執行的本機受管理設定檔，`status` 和 `doctor` 會回報來自 Chrome 的快取
  圖形診斷資訊：硬體／軟體分類、轉譯器、
  後端、裝置／驅動程式、功能與停用狀態詳細資料，以及硬體加速
  視訊能力。`openclaw browser --json status` 會傳回完整的結構化承載資料。
  被動狀態絕不會只為了收集這些資訊而啟動 Chrome。
- `stop` 會關閉作用中的控制工作階段並清除暫時的模擬覆寫，即使是 `attachOnly` 和遠端 CDP 設定檔，亦即 OpenClaw 並非自行啟動瀏覽器程序時也是如此。對於本機受管理設定檔，`stop` 也會停止所產生的瀏覽器程序。
- `start --headless` 僅套用於該次啟動要求，而且只有在 OpenClaw 啟動本機受管理瀏覽器時才適用。它不會改寫 `browser.headless` 或設定檔設定，對已在執行的瀏覽器也不會產生任何作用。
- 在沒有 `DISPLAY` 或 `WAYLAND_DISPLAY` 的 Linux 主機上，本機受管理設定檔會自動以無頭模式執行，除非 `OPENCLAW_BROWSER_HEADLESS=0`、`browser.headless=false` 或 `browser.profiles.<name>.headless=false` 明確要求顯示瀏覽器。

## 如果找不到命令

如果 `openclaw browser` 是未知命令，請檢查 `~/.openclaw/openclaw.json` 中的 `plugins.allow`。當存在 `plugins.allow` 時，除非設定中已經有根層級 `browser` 區塊，否則請明確列出隨附的瀏覽器外掛：

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

明確的根層級 `browser` 區塊（例如 `browser.enabled=true` 或 `browser.profiles.<name>`）也會在限制性的外掛允許清單下啟用隨附的瀏覽器外掛。

相關資訊：[瀏覽器工具](/zh-TW/tools/browser#missing-browser-command-or-tool)

## 設定檔

設定檔是具名的瀏覽器路由設定：

- `openclaw`（預設）：啟動或連接至 OpenClaw 專用管理的 Chrome 執行個體（隔離的使用者資料目錄）。
- `user`：透過 Chrome DevTools MCP 控制你現有且已登入的 Chrome 工作階段。
- 自訂 CDP 設定檔：指向本機或遠端 CDP 端點。

```bash
openclaw browser profiles
openclaw browser system-profiles
openclaw browser system-profiles --browser brave
openclaw browser import-profile --browser chrome --system Default --into imported
openclaw browser import-profile --system "Profile 1" --into work --domains google.com,youtube.com
openclaw browser create-profile --name work --color "#FF5A36"
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name remote --cdp-url https://browser-host.example.com
openclaw browser delete-profile --name work
```

在任何子命令上使用 `--browser-profile <name>` 來指定特定設定檔，例如 `openclaw browser --browser-profile work tabs`。

在 macOS 上，`system-profiles` 會列出主機上可用的實際 Chrome、Brave、Edge 或 Chromium 設定檔。`import-profile` 會在一次 macOS 鑰匙圈／Touch ID 同意提示後解密其 Cookie，並將 Cookie 注入全新的 OpenClaw 受管理設定檔。它只會匯入 Cookie；本機儲存空間和 IndexedDB 不會變更。部分 Google 工作階段使用裝置繫結工作階段認證資訊（DBSC），匯入後仍可能需要重新驗證。

當 macOS App 使用本機閘道時，它可以提供一次此匯入選項，並將隔離的已匯入設定檔設為代理程式瀏覽的預設值。匯入一律需要明確點擊；成功匯入或關閉提示後，將不再自動顯示後續提示，而**設定 → 一般 → 瀏覽器登入**仍可用於重新匯入。

系統設定檔匯入預設為啟用。設定 `browser.allowSystemProfileImport=false` 可同時停用命令列介面和代理程式觸發的匯入。匯入僅能在主機本機執行，無法透過瀏覽器節點 Proxy 執行。

## 分頁

```bash
openclaw browser tabs
openclaw browser tab new --label docs
openclaw browser tab label t1 docs
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://docs.openclaw.ai --label docs
openclaw browser focus docs
openclaw browser close t1
```

`tabs` 會先傳回 `suggestedTargetId`，接著是穩定的 `tabId`（例如 `t1`）、選用標籤，以及原始 `targetId`。將 `suggestedTargetId` 傳回 `focus`、`close`、快照和動作。使用 `open --label`、`tab new --label` 或 `tab label` 指派標籤；標籤、分頁 ID、原始目標 ID，以及唯一的目標 ID 前綴皆可接受。為了相容性，要求欄位仍命名為 `targetId`，但它接受上述任何分頁參照。

原始目標 ID 是不穩定的診斷控制代碼，不是持久的代理程式記憶：當 Chromium 在導覽或提交表單期間取代底層原始目標時，只要 OpenClaw 能確認相符，就會讓穩定的 `tabId`／標籤繼續附加至替代分頁。建議使用 `suggestedTargetId`。

## 快照／螢幕截圖／動作

快照：

```bash
openclaw browser snapshot
openclaw browser snapshot --urls
```

螢幕截圖：

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref e12
openclaw browser screenshot --labels
```

- `--full-page` 僅供頁面擷取使用；不能與 `--ref` 或 `--element` 結合使用。
- `existing-session`／`user` 設定檔支援頁面螢幕截圖，以及快照輸出中的 `--ref` 螢幕截圖，但不支援 CSS `--element` 螢幕截圖。
- `--labels` 會將目前的快照參照疊加在螢幕截圖上。在以 Playwright 為後端的設定檔中，它可搭配 `--full-page`（整頁疊加）、`--ref`（依 ARIA 參照裁切元素並疊加）及 `--element`（依 CSS 選擇器裁切元素並疊加）使用；在元素裁切模式中，標籤會相對於元素投影。回應也會包含 `annotations` 陣列（空白時省略），其中具有各參照的邊界方框：`ref`、`number`、`role`、選用的 `name`，以及擷取影像座標空間中的 `box: {x, y, width, height}`（檢視區／整頁／元素相對）。
  `existing-session` 設定檔會在頁面螢幕截圖上呈現 chrome-mcp 疊加層，但不使用 Playwright 投影輔助程式，也不包含 `annotations`；該處不支援 CSS `--element` 螢幕截圖。若沒有 Playwright 或 chrome-mcp，則無法使用帶標籤的螢幕截圖。
- `snapshot --urls` 會將探索到的連結目的地附加至 AI 快照，使代理程式可以選擇直接導覽目標，而不必只根據連結文字猜測。

導覽／點擊／輸入（以參照為基礎的 UI 自動化）：

```bash
openclaw browser navigate https://example.com
openclaw browser click <ref>
openclaw browser click-coords 120 340
openclaw browser type <ref> "hello"
openclaw browser press Enter
openclaw browser hover <ref>
openclaw browser scrollintoview <ref>
openclaw browser drag <startRef> <endRef>
openclaw browser select <ref> OptionA OptionB
openclaw browser fill --fields '[{"ref":"1","value":"Ada"}]'
openclaw browser wait --text "Done"
openclaw browser evaluate --fn '(el) => el.textContent' --ref <ref>
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
```

`evaluate --fn` 接受函式原始碼、運算式或陳述式主體。陳述式主體會封裝為非同步函式，因此請使用 `return` 指定你要傳回的值。當頁面端函式可能需要比預設求值逾時更長的時間時，請使用 `--timeout-ms`。`browser.evaluateEnabled=false`（預設：`true`）會同時停用 `evaluate` 和 `wait --fn`。

當動作觸發頁面替換時，只要 OpenClaw 能確認替代分頁，動作回應就會傳回目前的原始 `targetId`。對於長期執行的工作流程，指令碼仍應儲存並傳遞 `suggestedTargetId`／標籤。

檔案與對話方塊輔助程式：

```bash
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref <ref>
openclaw browser upload media://inbound/file.pdf --ref <ref>
openclaw browser waitfordownload
openclaw browser download <ref> report.pdf
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
```

受管理的 Chrome 設定檔會將一般點擊所觸發的下載儲存至 OpenClaw 下載目錄（預設為 `/tmp/openclaw/downloads`，或所設定的暫存根目錄）。當代理程式需要等待特定檔案並傳回其路徑時，請使用 `waitfordownload` 或 `download`；這些明確的等待程式會取得下一次下載的所有權。上傳接受來自 OpenClaw 暫存上傳根目錄及 OpenClaw 管理之輸入媒體的檔案，包括 `media://inbound/<id>` 和沙箱相對 `media/inbound/<id>` 參照。巢狀媒體參照、路徑周遊和任意本機路徑都會遭到拒絕。

當動作開啟強制回應對話方塊時，動作回應會傳回具有 `browserState.dialogs.pending` 的 `blockedByDialog`；傳遞 `--dialog-id` 即可直接回應。由 OpenClaw 以外機制處理的對話方塊會顯示在 `browserState.dialogs.recent` 下。

批次動作：

```bash
openclaw browser batch --actions '[{"kind":"wait","timeMs":500},{"kind":"click","ref":"12"},{"kind":"type","ref":"23","text":"hello"}]'
openclaw browser batch --actions-file plan.json
openclaw browser batch --actions-file - --continue
```

`openclaw browser batch` 會傳送一個 `kind="batch"` `/act` 請求，其中包含巢狀的 `BrowserActRequest` 動作（`wait`、`click`、`type`、`evaluate`，……）— 而不是 `open`/`navigate`/`snapshot`/`screenshot`；後者是命令列介面子命令，而非 `/act` 類型。`--continue` 會設定 `stopOnError=false`（預設在第一個錯誤時停止）；`--target-id` 會將整個批次限定於單一分頁。巢狀動作失敗會使命令以非零狀態結束；使用 `--json` 可保留依序排列的 `results` 回應。完整契約（參照生命週期、目標 ID 衝突、錯誤摘要）請參閱[瀏覽器批次命令列介面](/zh-TW/tools/browser-control#browser-batch-cli)。`batch` 不支援 `profile="user"`／現有工作階段設定檔。

## 狀態與儲存空間

檢視區與模擬：

```bash
openclaw browser resize 1280 720
openclaw browser set viewport 1280 720
openclaw browser set offline on
openclaw browser set media dark
openclaw browser set timezone Europe/London
openclaw browser set locale en-GB
openclaw browser set geo 51.5074 -0.1278 --accuracy 25
openclaw browser set device "iPhone 14"
openclaw browser set headers '{"x-test":"1"}'
openclaw browser set credentials myuser mypass
```

Cookie 與儲存空間：

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url https://example.com
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set token abc123
openclaw browser storage session clear
```

## 偵錯

```bash
openclaw browser console --level error
openclaw browser pdf
openclaw browser responsebody "**/api"
openclaw browser highlight <ref>
openclaw browser errors --clear
openclaw browser requests --filter api
openclaw browser trace start
openclaw browser trace stop --out trace.zip
```

## 透過 MCP 使用現有的 Chrome

使用內建的 `user` 設定檔，或建立自己的 `existing-session` 設定檔：

```bash
openclaw browser --browser-profile user tabs
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name brave-live --driver existing-session --user-data-dir "~/Library/Application Support/BraveSoftware/Brave-Browser"
openclaw browser create-profile --name chrome-port --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser --browser-profile chrome-live tabs
```

預設的現有工作階段路徑是僅限主機的 Chrome MCP 自動連線。如果瀏覽器已使用 DevTools 端點執行，請傳入 `--cdp-url`，讓 Chrome MCP 改為連接至該端點。若是 Docker、Browserless 或其他不需要 Chrome MCP 語意的遠端設定，請改用 CDP 設定檔。

目前現有工作階段的限制：

- 快照驅動的動作使用參照，而非 CSS 選擇器。
- 當呼叫端省略 `timeoutMs` 時，支援的 `act` 請求會使用內建的 60000 ms 預設值；每次呼叫的 `timeoutMs` 仍具有優先權。
- `click` 僅支援按滑鼠左鍵。
- `type` 不支援 `slowly=true`。
- `press` 不支援 `delayMs`。
- `hover`、`scrollintoview`、`drag`、`select` 和 `fill` 會拒絕每次呼叫的逾時覆寫；`evaluate` 接受 `--timeout-ms`。
- `select` 僅支援一個值。
- 不支援 `wait --load networkidle`（可用於受管理及原始／遠端 CDP 設定檔）。
- 檔案上傳需要 `--ref`／`--input-ref`，不支援 CSS `--element`，且一次僅支援一個檔案。
- 對話方塊掛鉤不支援 `--timeout`。
- 螢幕截圖支援頁面擷取和 `--ref`，但不支援 CSS `--element`。
- `responsebody`、下載攔截、PDF 匯出及批次動作仍需要受管理的瀏覽器或原始 CDP 設定檔。

## 遠端瀏覽器控制（節點主機 Proxy）

如果閘道與瀏覽器在不同機器上執行，請在具有 Chrome/Brave/Edge/Chromium 的機器上執行**節點主機**。閘道會將瀏覽器動作代理至該節點；不需要另外架設瀏覽器控制伺服器。

使用 `gateway.nodes.browser.mode` 控制自動路由，並在連線多個節點時使用 `gateway.nodes.browser.node` 固定使用特定節點。

安全性與遠端設定：[瀏覽器工具](/zh-TW/tools/browser)、[遠端存取](/zh-TW/gateway/remote)、[Tailscale](/zh-TW/gateway/tailscale)、[安全性](/zh-TW/gateway/security)

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [瀏覽器](/zh-TW/tools/browser)
