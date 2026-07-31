---
read_when:
    - 新增由代理程式控制的瀏覽器自動化
    - 偵錯 OpenClaw 為何會干擾你自己的 Chrome
    - 在 macOS App 中實作瀏覽器設定與生命週期
summary: 整合式瀏覽器控制服務 + 動作命令
title: 瀏覽器（由 OpenClaw 管理）
x-i18n:
    generated_at: "2026-07-26T08:15:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3afa2dda17520ae6c53fe3f1a7a12e7ca8a1414b2c12b79cf4a09ac8906bb3ca
    source_path: tools/browser.md
    workflow: 16
---

OpenClaw 可以執行由代理程式控制的**專用 Chrome/Brave/Edge/Chromium 設定檔**。它透過閘道內的小型本機控制服務（僅限回送介面）運作，並與你的個人瀏覽器隔離。

- 可以把它視為一個**獨立且僅供代理程式使用的瀏覽器**。`openclaw` 設定檔絕不會存取你的個人瀏覽器設定檔。
- 代理程式會在這個隔離環境中開啟分頁、讀取頁面、點選及輸入內容。
- 內建的 `user` 設定檔則會透過 Chrome DevTools MCP 連接到你實際已登入的 Chrome 工作階段。

## 你會獲得什麼

- 名為 **openclaw** 的獨立瀏覽器設定檔（預設使用橘色強調色）。
- 可預期的分頁控制（列出／開啟／聚焦／關閉）。
- 代理程式動作（點選／輸入／拖曳／選取）、快照、螢幕截圖及 PDF。
- 由 Playwright 支援的設定檔會將直接連至附件的導覽儲存在受管理的下載目錄中，並在完成最終 URL 政策驗證後傳回 `{ url, suggestedFilename, path }` 中繼資料。
- 當動作立即開始一項或多項下載時，由 Playwright 支援的代理程式動作會傳回包含相同受管理中繼資料的 `downloads` 陣列。
- 啟用瀏覽器外掛時，隨附的 `browser-automation` Skill 會教導代理程式如何執行快照、
  穩定分頁、過期參照及手動阻礙因素的復原迴圈。
- 選用的多設定檔支援（`openclaw`、`work`、`remote`，……）。

這個瀏覽器**不是**你的日常主要瀏覽器，而是供代理程式自動化與驗證使用的
安全隔離介面。

在 macOS 上，你可以明確地將 Cookie 從 Chrome 系列的系統設定檔複製到獨立的受管理設定檔。受管理的瀏覽器仍會使用自己的使用者資料目錄；只會複製所選的 Cookie，本機儲存空間與 IndexedDB 則會保留在原處。匯入命令與限制請參閱[設定檔](#profiles-multi-browser)或 [`openclaw browser` 命令列介面參考資料](/zh-TW/cli/browser)。

## 快速開始

```bash
openclaw browser --browser-profile openclaw doctor
openclaw browser --browser-profile openclaw doctor --deep
openclaw browser --browser-profile openclaw status
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

「瀏覽器已停用」表示外掛或 `browser.enabled` 已關閉；請參閱
[設定](#configuration)與[外掛控制](#plugin-control)。

如果完全找不到 `openclaw browser`，或代理程式表示瀏覽器工具
無法使用，請直接前往[缺少瀏覽器命令或工具](#missing-browser-command-or-tool)。

## 外掛控制

預設的 `browser` 工具是隨附外掛。若要使用其他註冊相同 `browser` 工具名稱的外掛取代它，請停用此工具：

```json5
{
  plugins: {
    entries: {
      browser: {
        enabled: false,
      },
    },
  },
}
```

預設行為同時需要 `plugins.entries.browser.enabled` **及** `browser.enabled=true`。只停用外掛時，會將 `openclaw browser` 命令列介面、`browser.request` 閘道方法、代理程式工具及控制服務視為一個整體一併移除；你的 `browser.*` 設定會保持不變，以供替代項目使用。

變更瀏覽器設定後必須重新啟動閘道，外掛才能重新註冊其服務。

## 代理程式指引

工具設定檔注意事項：`tools.profile: "coding"` 包含 `web_search` 和
`web_fetch`，但不包含完整的 `browser` 工具。若要讓代理程式或
產生的子代理程式使用瀏覽器自動化，請在設定檔
階段加入 browser：

```json5
{
  tools: {
    profile: "coding",
    alsoAllow: ["browser"],
  },
}
```

對於單一代理程式，請使用 `agents.entries.*.tools.alsoAllow: ["browser"]`。
只有 `tools.subagents.tools.allow: ["browser"]` 並不足夠，因為子代理程式
政策會在設定檔篩選後套用。

瀏覽器外掛提供兩個層級的代理程式指引：

- `browser` 工具描述包含精簡且持續生效的契約：選擇
  正確的設定檔、讓參照維持在同一分頁、使用 `tabId`／標籤指定
  分頁，並在多步驟工作中載入瀏覽器 Skill。
- 隨附的 `browser-automation` Skill 包含較完整的操作迴圈：
  先檢查狀態／分頁、標記工作分頁、操作前建立快照、介面變更後重新建立快照、
  嘗試復原過期參照一次，並將登入／雙因素驗證／CAPTCHA 或
  相機／麥克風阻礙因素回報為需要手動操作，而不是自行猜測。

啟用外掛後，外掛隨附的 Skill 會列在代理程式的可用 Skill 中。
完整的 Skill 指示會視需要載入，因此例行
回合無須負擔完整的 Token 成本。

## 缺少瀏覽器命令或工具

如果升級後無法辨識 `openclaw browser`、缺少 `browser.request`，或代理程式回報瀏覽器工具無法使用，通常是因為 `plugins.allow` 清單省略了 `browser`，且不存在根層級的 `browser` 設定區塊。請加入：

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

明確的根層級 `browser` 區塊（`browser` 下的任何鍵，例如
`browser.enabled=true` 或 `browser.profiles.<name>`）即使在限制嚴格的 `plugins.allow` 下，
也會啟用隨附的瀏覽器外掛，這與隨附頻道的設定行為一致。`plugins.entries.browser.enabled=true` 和
`tools.alsoAllow: ["browser"]` 本身無法取代允許清單成員資格。
完全移除 `plugins.allow` 也會恢復預設值。

## 設定檔：`openclaw`、`user`、`chrome`

- `openclaw`：受管理且隔離的瀏覽器（不需要擴充功能）。
- `user`：內建的 Chrome DevTools MCP 連接設定檔，用於你**實際
  已登入的 Chrome** 工作階段。OpenClaw 首次連接時，Chrome 會顯示阻擋操作的「Allow remote debugging?」
  提示，因此必須有人在電腦前。
- `chrome`：內建的 [Chrome 擴充功能](/zh-TW/tools/chrome-extension)設定檔，用於你
  **實際已登入的 Chrome** 工作階段。即使桌前無人也能透過手機運作，
  因為它是透過 OpenClaw 瀏覽器擴充功能控制分頁，而非使用
  遠端偵錯連接埠，所以不會顯示「Allow remote debugging?」提示。

對代理程式的瀏覽器工具呼叫而言：

- 預設：使用隔離的 `openclaw` 瀏覽器。
- 當現有登入工作階段很重要，且使用者**不在電腦前**時（Telegram、WhatsApp 等），
  優先使用 `profile="chrome"`（擴充功能）。
- 當現有登入工作階段很重要，且使用者**在電腦前**可以核准連接提示時，
  優先使用 `profile="user"`（Chrome MCP）。
- 若需要特定瀏覽器模式，`profile` 可用於明確覆寫。

若要預設使用受管理模式，請設定 `browser.defaultProfile: "openclaw"`。

## 設定

瀏覽器設定位於 `~/.openclaw/openclaw.json`。

```json5
{
  browser: {
    enabled: true, // 預設值：true
    evaluateEnabled: true, // 預設值：true；false 會停用 act:evaluate（任意 JS）
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // 僅針對受信任的私人網路存取選擇啟用
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    // cdpUrl: "http://127.0.0.1:18792", // 舊版單一設定檔覆寫
    tabCleanup: {
      enabled: true, // 預設值：true
    },
    // snapshotDefaults: { mode: "efficient" }, // 呼叫端省略時的預設快照模式
    defaultProfile: "openclaw",
    color: "#FF4500",
    headless: false,
    noSandbox: false,
    attachOnly: false,
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        headless: true,
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: {
        driver: "existing-session",
        attachOnly: true,
        color: "#00AA00",
      },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
  },
}
```

當呼叫端未傳入明確的 `snapshotFormat` 或
`mode` 時，`browser.snapshotDefaults.mode: "efficient"` 會變更預設的 `snapshot`
擷取模式；各次呼叫的快照選項請參閱[瀏覽器控制 API](/zh-TW/tools/browser-control)。

### 分頁清理的所有權

工作階段分頁清理僅適用於 OpenClaw 瀏覽器工具使用
`action: "open"` 建立的分頁。OpenClaw 不會接管原先已開啟、
由使用者開啟，或所有權不明的分頁。
`browser.tabCleanup` 區塊控制主要工作階段的定期閒置與數量上限清理；
停用它不會停用明確的工作階段生命週期清理。

對於主機本機開啟的分頁，具有穩定原生 CDP 目標和瀏覽器
身分的所有權會儲存在共用 SQLite 狀態中。這些記錄會在閘道
重新啟動後保留，並繼續適用於 `/new` 及其他工作階段生命週期清理；
工作階段生命週期清理包含子代理程式、排程及 ACP 工作階段結束。
工具介面所用目標即為原生 CDP 目標的記錄，也會在重新啟動後繼續
適用於閒置清理及各工作階段的數量上限清理。Chrome MCP 目標控制代碼
僅存在於處理程序本機，因此冷啟動的現有工作階段記錄會等待生命週期清理，
而不會冒險執行閒置清理，因為重新啟動後無法安全地判定活動歸屬。
此持久化路徑可以涵蓋 OpenClaw 管理的設定檔、
一般遠端 CDP 設定檔，以及具有明確 `cdpUrl` 的現有工作階段設定檔，
前提是 OpenClaw 能解析原生目標和穩定的瀏覽器身分。
關閉持久化記錄前，OpenClaw 會驗證設定的設定檔與瀏覽器執行個體仍然相符。

Chrome MCP `--autoConnect`、其 `/json/version` 回應缺少
穩定瀏覽器身分的 CDP 端點，以及無法解析原生目標的開啟操作，
仍會採用僅限處理程序本機的盡力追蹤。它們可在該
閘道處理程序執行期間清理，但閘道重新啟動後不會自動關閉。
在提供持久化追蹤前就已開啟的分頁不會被追溯接管；請手動關閉這些分頁。

清理採盡力而為，不保證每個符合條件的分頁都會
立即關閉。暫時性的所有權檢查或關閉失敗會讓持久化
清理維持待處理狀態，以便稍後重試。重試並非無限次：當瀏覽器
持續無法連線，且分頁已超過一天未使用時，系統會移除追蹤資料列，
避免持久化儲存區被永遠無法再次驗證的分頁填滿。

### 螢幕截圖視覺辨識（支援純文字模型）

當主要模型是純文字模型（不支援視覺／多模態）時，瀏覽器
螢幕截圖會傳回模型無法讀取的影像區塊。瀏覽器螢幕截圖
會重複使用現有的影像理解設定，因此為媒體理解設定的影像模型
可以將螢幕截圖描述成文字，而不需要任何瀏覽器專用的模型設定。

```json5
{
  tools: {
    media: {
      image: {
        models: [
          { provider: "bytedance", model: "doubao-seed-2.0-pro" },
          // 新增備援候選項目；第一個成功者生效
          { provider: "openai", model: "gpt-4o" },
        ],
      },
      // 共用媒體模型若標記為支援影像也可運作。
      // models: [{ provider: "openai", model: "gpt-4o", capabilities: ["image"] }],
    },
  },
  agents: {
    defaults: {
      // 也會採用現有的影像模型預設值。
      // imageModel: { primary: "openai/gpt-4o" },
    },
  },
}
```

**運作方式：**

1. 代理程式呼叫 `browser screenshot`，並如往常將影像擷取至磁碟。
2. 瀏覽器工具會詢問現有的影像理解執行階段，確認是否能使用已設定的媒體影像模型、共用媒體模型、影像模型預設值或有驗證支援的影像供應商來描述螢幕截圖。
3. 視覺模型會傳回文字描述，該描述會以
   `wrapExternalContent`（提示詞注入防護）封裝，並以文字區塊而非影像區塊的形式傳回代理程式。
4. 如果影像理解功能無法使用、遭到略過或失敗，瀏覽器會改為傳回原始影像區塊。

螢幕截圖影像區塊是私有工具結果：代理程式可以檢查它們，
但 OpenClaw 不會自動將其附加至頻道回覆。若要分享螢幕截圖，
請要求代理程式使用訊息工具明確傳送。

使用現有的 `tools.media.image` / `tools.media.models` 欄位來設定模型
備援、逾時、位元組限制、設定檔及供應商請求設定。

如果目前作用中的主要模型已支援視覺功能，且未設定明確的影像
理解模型，OpenClaw 會保留一般影像結果，讓主要模型可以直接讀取螢幕截圖。

<AccordionGroup>

<Accordion title="連接埠與可連線性">

- 控制服務會繫結至迴路介面，連接埠由 `gateway.port` 衍生（預設 `18791` = 閘道 + 2）。`OPENCLAW_GATEWAY_PORT` 的優先順序高於 `gateway.port`；兩者都會偏移同一系列的衍生連接埠。
- 本機 `openclaw` 設定檔會從控制連接埠上方 9 個連接埠起始的範圍，自動指派 `cdpPort`/`cdpUrl`（預設 `18800`-`18899`）；僅針對
  遠端 CDP 設定檔或現有工作階段端點附加設定這些值。未設定時，`cdpUrl` 預設為
  受管理的本機 CDP 連接埠。
- 遠端及 `attachOnly` CDP 的可連線性、WebSocket 交握，以及本機
  受管理 Chrome 的啟動，均使用內建期限。
- 受管理 Chrome 重複發生啟動或就緒失敗時，會依設定檔啟動斷路機制。
  連續失敗數次後，OpenClaw 會短暫暫停新的啟動嘗試，而不會在每次
  瀏覽器工具呼叫時都產生 Chromium。請修正啟動問題、不需要瀏覽器時
  將其停用，或在修復後重新啟動閘道。

</Accordion>

<Accordion title="SSRF 政策">

- 瀏覽器導覽及開啟分頁請求會接受預先檢查。在動作執行期間及有界限的動作後寬限期內，受防護的 Playwright 互動（點擊、座標點擊、懸停、拖曳、捲動、選取、按鍵、輸入、填寫表單及求值）會在送出 HTTP 請求位元組前，攔截政策拒絕的頂層及子框架文件載入，然後以最佳努力重新檢查最終的 `http(s)` URL。
- 每次全新啟動由 OpenClaw 管理的 Chrome 前，OpenClaw 都會以最佳努力停用網路預測，抑制 Chromium 針對這些遭拒載入所觀察到的推測性預先連線。這是縱深防禦，而非政策邊界：跨控制服務重新啟動而重複使用的瀏覽器，以及其他瀏覽器後端，可能不具備相同的強化措施。Playwright 路由仍不是網路防火牆，且不會攔截重新導向的中間跳轉、彈出式視窗的第一個請求、Service Worker 流量、在有界限的防護時窗後執行的頁面程式碼，或每一條背景／子資源路徑。完整的對外連線隔離需要由擁有者端進行隔離，或使用強制執行政策的 Proxy。
- 在嚴格 SSRF 模式下，也會檢查遠端 CDP 端點探索及 `/json/version` 探查（`cdpUrl`）。
- 閘道／供應商的 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY` 和 `NO_PROXY` 環境變數不會自動代理 OpenClaw 管理的瀏覽器。受管理的 Chrome 預設會直接啟動，因此供應商 Proxy 設定不會削弱瀏覽器 SSRF 檢查。
- OpenClaw 管理的本機 CDP 就緒探查及 DevTools WebSocket 連線，會針對確切啟動的迴路端點略過受管理的網路 Proxy，因此即使操作員的 Proxy 阻擋迴路對外連線，`openclaw browser start` 仍可運作。
- 若要代理受管理的瀏覽器本身，請透過 `browser.extraArgs` 傳遞明確的 Chrome Proxy 旗標，例如 `--proxy-server=...` 或 `--proxy-pac-url=...`。除非刻意啟用私人網路瀏覽器存取，否則嚴格 SSRF 模式會封鎖明確的瀏覽器 Proxy 路由。
- `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` 預設為關閉；僅在刻意信任私人網路瀏覽器存取時啟用。
- `browser.ssrfPolicy.allowPrivateNetwork` 仍支援作為舊版別名。

</Accordion>

<Accordion title="設定檔行為">

- `attachOnly: true` 表示絕不啟動本機瀏覽器；只有在瀏覽器已執行時才附加。
- `headless` 可設為全域值或個別本機受管理設定檔的值。個別設定檔的值會覆寫 `browser.headless`，因此一個本機啟動的設定檔可維持無頭模式，另一個則維持可見。
- `POST /start?headless=true` 和 `openclaw browser start --headless` 會要求
  本機受管理設定檔進行一次性無頭啟動，而不改寫
  `browser.headless` 或設定檔組態。現有工作階段、僅附加及
  遠端 CDP 設定檔會拒絕此覆寫，因為 OpenClaw 不會啟動這些
  瀏覽器程序。
- 在沒有 `DISPLAY` 或 `WAYLAND_DISPLAY` 的 Linux 主機上，當環境及設定檔／全域
  組態都未明確選擇有頭模式時，本機受管理設定檔會自動預設為無頭模式。
  請使用意思明確的瀏覽器層級形式
  `openclaw browser --json status`；尾端的 `openclaw browser status --json`
  也可運作，因為 `status` 未定義自己的 `--json`。此命令會將
  `headlessSource` 回報為 `env`、`profile`、`config`、
  `request`、`linux-display-fallback` 或 `default`。
- `OPENCLAW_BROWSER_HEADLESS=1` 會強制目前程序以無頭模式啟動本機受管理瀏覽器。
  `OPENCLAW_BROWSER_HEADLESS=0` 會強制一般啟動使用有頭模式，並在沒有顯示伺服器的
  Linux 主機上傳回可操作的錯誤；明確的 `start --headless` 請求
  仍會在該次啟動時優先採用。
- 瀏覽器控制路由及程式化用戶端會保留無顯示器錯誤中
  人類可讀的 `error`，並公開穩定的原因
  `no_display_for_headed_profile`。其 `details` 僅包含 `profile`、
  `requestedHeadless`、`headlessSource` 和 `displayPresent`，因此 API 用戶端可
  選擇正確的修正方式，而不必比對訊息文字。
- 對於正在執行的本機受管理設定檔，狀態及 doctor 會查詢 Chrome 的
  瀏覽器層級 CDP 端點，以取得算繪器、後端、裝置／驅動程式、功能
  狀態、驅動程式因應措施及硬體加速視訊能力。結果會針對該瀏覽器
  程序快取，並由 `openclaw browser --json status` 完整公開。
  被動狀態呼叫不會啟動 Chrome。現有工作階段、擴充功能、遠端 CDP
  及沙箱瀏覽器仍各自獨立，不會透過此受管理主機路徑進行檢查。
- 無頭模式的受管理 Chrome 仍使用保守的 `--disable-gpu` 預設值。
  診斷不會啟用加速、新增全域加速設定，或授予沙箱瀏覽器裝置存取權。
- `executablePath` 可設為全域值或個別本機受管理設定檔的值。個別設定檔的值會覆寫 `browser.executablePath`，因此不同的受管理設定檔可以啟動不同的 Chromium 系瀏覽器。兩種形式都接受 `~` 來表示你作業系統的家目錄。
- `color`（頂層及個別設定檔）會為瀏覽器 UI 加上色調，讓你看出目前作用中的設定檔。
- 預設設定檔為 `openclaw`（受管理的獨立執行個體）。使用 `defaultProfile: "user"` 可選擇使用已登入的使用者瀏覽器。
- 自動偵測順序：若系統預設瀏覽器以 Chromium 為基礎，則使用該瀏覽器；否則依序為 Chrome、Brave、Edge、Chromium、Chrome Canary。
- `driver: "existing-session"` 使用 Chrome DevTools MCP，而非原始 CDP。它可以透過 Chrome MCP 自動連線附加，或在你已有執行中瀏覽器的 DevTools 端點時，透過 `cdpUrl` 附加。
- `driver: "extension"` 透過 [OpenClaw Chrome 擴充功能](/zh-TW/tools/chrome-extension)控制你已登入的 Chrome。轉送器擁有其迴路端點，因此這些設定檔不接受 `cdpUrl`。這是唯一能在電腦前無人操作時運作的已登入瀏覽器模式。
- 當現有工作階段設定檔應附加至非預設的 Chromium 使用者設定檔（Brave、Edge 等）時，請設定 `browser.profiles.<name>.userDataDir`。此路徑也接受 `~` 來表示你作業系統的家目錄。

</Accordion>

</AccordionGroup>

## 使用 Brave 或其他 Chromium 系瀏覽器

如果你的**系統預設**瀏覽器以 Chromium 為基礎（Chrome/Brave/Edge 等），
OpenClaw 會自動使用它。設定 `browser.executablePath` 可覆寫
自動偵測。頂層及個別設定檔的 `executablePath` 值接受 `~`
來表示你作業系統的家目錄：

```bash
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set browser.profiles.work.executablePath "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
```

或者依平台在組態中設定：

<Tabs>
  <Tab title="macOS">
```json5
{
  browser: {
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
  },
}
```
  </Tab>
  <Tab title="Windows">
```json5
{
  browser: {
    executablePath: "C:\\Program Files\\BraveSoftware\\Brave-Browser\\Application\\brave.exe",
  },
}
```
  </Tab>
  <Tab title="Linux">
```json5
{
  browser: {
    executablePath: "/usr/bin/brave-browser",
  },
}
```
  </Tab>
</Tabs>

個別設定檔的 `executablePath` 僅影響由 OpenClaw
啟動的本機受管理設定檔。`existing-session` 設定檔會改為附加至已執行的瀏覽器，
遠端 CDP 設定檔則使用 `cdpUrl` 背後的瀏覽器。

## 本機與遠端控制

- **本機控制（預設）：**閘道會啟動迴路控制服務，並可啟動本機瀏覽器。
- **遠端控制（節點主機）：**在具有瀏覽器的電腦上執行節點主機；閘道會將瀏覽器動作代理至該主機。
- **遠端 CDP：**設定 `browser.profiles.<name>.cdpUrl`（或 `browser.cdpUrl`）以
  附加至遠端 Chromium 系瀏覽器。在此情況下，OpenClaw 不會啟動本機瀏覽器。
- 對於迴路介面上由外部管理的 CDP 服務（例如在 Docker 中發布至
  `127.0.0.1` 的 Browserless），也請設定 `attachOnly: true`。未設定
  `attachOnly` 的迴路 CDP 會視為由 OpenClaw 管理的本機瀏覽器設定檔。
- `headless` 僅影響由 OpenClaw 啟動的本機受管理設定檔。它不會重新啟動或變更現有工作階段或遠端 CDP 瀏覽器。
- `executablePath` 遵循相同的本機受管理設定檔規則。在執行中的
  本機受管理設定檔上變更此值，會將該設定檔標記為需要重新啟動／協調，
  使下次啟動使用新的二進位檔。

停止行為會依設定檔模式而異：

- 本機受管理設定檔：`openclaw browser stop` 會停止由
  OpenClaw 啟動的瀏覽器程序
- 僅附加及遠端 CDP 設定檔：`openclaw browser stop` 會關閉作用中的
  控制工作階段，並釋放 Playwright/CDP 模擬覆寫（檢視區、
  配色、地區設定、時區、離線模式及類似狀態），即使
  OpenClaw 並未啟動任何瀏覽器程序

遠端 CDP URL 可包含驗證資訊：

- 查詢權杖（例如 `https://provider.example?token=<token>`）
- HTTP Basic 驗證（例如 `https://user:pass@provider.example`）

OpenClaw 在呼叫 `/json/*` 端點及連線至 CDP WebSocket 時會保留驗證資訊。請優先使用環境變數或密鑰管理工具來儲存權杖，而不要將其提交至設定檔。

## 節點瀏覽器代理（零設定預設值）

如果你在有瀏覽器的機器上執行**節點主機**，OpenClaw 可以自動將瀏覽器工具呼叫路由至該節點，無須任何額外的瀏覽器設定。這是遠端閘道的預設路徑。

注意事項：

- 節點主機透過**代理命令**公開其本機瀏覽器控制伺服器。
- 設定檔來自節點自身的 `browser.profiles` 設定（與本機相同）。
- 無論 `allowProfiles` 為何，代理命令都絕不允許持久變更設定檔（`create-profile`、`delete-profile`、`reset-profile`）；請直接在節點上進行這些變更。
- `nodeHost.browserProxy.allowProfiles` 是選用項目。將其留空即可使用舊版／預設行為：所有已設定的設定檔仍可透過代理存取。
- 如果設定 `nodeHost.browserProxy.allowProfiles`，OpenClaw 會將其視為最小權限邊界，限制代理可指定的設定檔名稱。
- 如果不需要此功能，請將其停用：
  - 在節點上：`nodeHost.browserProxy.enabled=false`
  - 在閘道上：`gateway.nodes.browser.mode="off"`（也接受 `"auto"` 以選取單一已連線的瀏覽器節點，或使用 `"manual"` 以要求明確的節點參數）

## Browserless（託管的遠端 CDP）

[Browserless](https://browserless.io) 是一項託管的 Chromium 服務，透過 HTTPS 和 WebSocket 公開 CDP 連線 URL。OpenClaw 可使用任一種形式，但對於遠端瀏覽器設定檔，最簡單的選項是使用 Browserless 連線文件中的直接 WebSocket URL。

範例：

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    profiles: {
      browserless: {
        cdpUrl: "wss://production-sfo.browserless.io?token=<BROWSERLESS_API_KEY>",
        color: "#00AA00",
      },
    },
  },
}
```

注意事項：

- 將 `<BROWSERLESS_API_KEY>` 替換為你實際的 Browserless 權杖。
- 選擇與你的 Browserless 帳號相符的區域端點（請參閱其文件）。
- 如果 Browserless 提供 HTTPS 基底 URL，你可以將其轉換為 `wss://` 以進行直接 CDP 連線，或保留 HTTPS URL，讓 OpenClaw 探索 `/json/version`。

### 同一主機上的 Browserless Docker

當 Browserless 自行託管於 Docker 中，而 OpenClaw 在主機上執行時，請將 Browserless 視為外部管理的 CDP 服務：

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    profiles: {
      browserless: {
        cdpUrl: "ws://127.0.0.1:3000",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

OpenClaw 程序必須能夠連線至 `browser.profiles.browserless.cdpUrl` 中的位址。Browserless 也必須公告相符且可連線的端點；請將 Browserless 的 `EXTERNAL` 設為同一個可由 OpenClaw 公開存取的 WebSocket 基底，例如 `ws://127.0.0.1:3000`、`ws://browserless:3000`，或穩定的私有 Docker 網路位址。如果 `/json/version` 傳回的 `webSocketDebuggerUrl` 指向 OpenClaw 無法連線的位址，CDP HTTP 看似正常，但 WebSocket 附加仍會失敗。

對於迴路 Browserless 設定檔，請勿讓 `attachOnly` 保持未設定。若沒有 `attachOnly`，OpenClaw 會將迴路連接埠視為本機管理的瀏覽器設定檔，並可能回報該連接埠正在使用中，但不屬於 OpenClaw。

## 直接 WebSocket CDP 提供者

部分託管瀏覽器服務公開的是**直接 WebSocket** 端點，而非標準的 HTTP 型 CDP 探索（`/json/version`）。OpenClaw 接受三種 CDP URL 形式，並會自動選擇正確的連線策略：

- **HTTP(S) 探索**——`http://host[:port]` 或 `https://host[:port]`。
  OpenClaw 會呼叫 `/json/version` 以探索 WebSocket 偵錯工具 URL，然後進行連線。不會回退至 WebSocket。
- **直接 WebSocket 端點**——具有 `/devtools/browser|page|worker|shared_worker|service_worker/<id>` 路徑的 `ws://host[:port]/devtools/<kind>/<id>` 或 `wss://...`。OpenClaw 會直接透過 WebSocket 交握連線，並完全略過 `/json/version`。
- **裸 WebSocket 根端點**——不含 `/devtools/...` 路徑的 `ws://host[:port]` 或 `wss://host[:port]`（例如 [Browserless](https://browserless.io)、[Browserbase](https://www.browserbase.com)）。OpenClaw 會先嘗試 HTTP `/json/version` 探索（將配置標準化為 `http`/`https`）；如果探索傳回 `webSocketDebuggerUrl`，便會使用它，否則 OpenClaw 會回退至裸根端點的直接 WebSocket 交握。如果公告的 WebSocket 端點拒絕 CDP 交握，但設定的裸根端點接受交握，OpenClaw 也會回退至該根端點。這可讓指向本機 Chrome 的裸 `ws://` 仍能連線，因為 Chrome 僅接受來自 `/json/version`、位於特定個別目標路徑上的 WebSocket 升級；同時，當託管提供者的探索端點公告不適用於 Playwright CDP 的短期 URL 時，仍可使用其根 WebSocket 端點。

`openclaw browser doctor` 使用與執行階段附加相同的先探索、再回退至 WebSocket 邏輯，因此可成功連線的裸根 URL 不會被診斷功能回報為無法連線。

### Browserbase

[Browserbase](https://www.browserbase.com) 是用於執行無頭瀏覽器的雲端平台，內建 CAPTCHA 解決功能、隱匿模式和住宅代理。

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserbase",
    profiles: {
      browserbase: {
        cdpUrl: "wss://connect.browserbase.com?apiKey=<BROWSERBASE_API_KEY>",
        color: "#F97316",
      },
    },
  },
}
```

注意事項：

- [註冊](https://www.browserbase.com/sign-up)，並從 [Overview dashboard](https://www.browserbase.com/overview) 複製你的 **API Key**。
- 將 `<BROWSERBASE_API_KEY>` 替換為你實際的 Browserbase API 金鑰。
- Browserbase 會在 WebSocket 連線時自動建立瀏覽器工作階段，因此不需要手動建立工作階段。
- 請參閱[價格](https://www.browserbase.com/pricing)，以瞭解目前的免費方案限制和付費方案。
- 請參閱 [Browserbase 文件](https://docs.browserbase.com)，以取得完整的 API 參考、SDK 指南和整合範例。

### Notte

[Notte](https://www.notte.cc) 是用於執行無頭瀏覽器的雲端平台，內建隱匿功能、住宅代理，以及 CDP 原生 WebSocket 閘道。

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "notte",
    profiles: {
      notte: {
        cdpUrl: "wss://us-prod.notte.cc/sessions/connect?token=<NOTTE_API_KEY>",
        color: "#7C3AED",
      },
    },
  },
}
```

注意事項：

- [註冊](https://console.notte.cc)，並從主控台設定頁面複製你的 **API Key**。
- 將 `<NOTTE_API_KEY>` 替換為你實際的 Notte API 金鑰。
- Notte 會在 WebSocket 連線時自動建立瀏覽器工作階段，因此不需要手動建立工作階段。WebSocket 中斷連線時，該工作階段會被銷毀。
- 請參閱[價格](https://www.notte.cc/#pricing)，以瞭解目前的免費方案限制和付費方案。
- 請參閱 [Notte 文件](https://docs.notte.cc)，以取得完整的 API 參考、SDK 指南和整合範例。

## 安全性

核心概念：

- 瀏覽器控制僅限迴路存取；存取流量會通過閘道的驗證或節點配對。
- 獨立的迴路瀏覽器 HTTP API **僅使用共享密鑰驗證**：閘道權杖的 Bearer 驗證、`x-openclaw-password`，或使用已設定閘道密碼的 HTTP Basic 驗證。
- Tailscale Serve 身分標頭和 `gateway.auth.mode: "trusted-proxy"` **無法**驗證此獨立的迴路瀏覽器 API。
- 如果瀏覽器控制已啟用，但未設定共享密鑰驗證，OpenClaw 會在啟動時自動產生並持久儲存瀏覽器控制認證資訊：當 `gateway.auth.mode` 為 `none` 時產生權杖；當其為 `trusted-proxy` 時則產生密碼（透過 `gateway.auth.password` 持久儲存，以便程序外的迴路用戶端解析）。如果該模式已明確設定字串認證資訊，或 `gateway.auth.mode` 為 `password`，則會略過自動產生。
- 如果你想使用由自己控制的穩定密鑰，而非自動產生的密鑰，請明確設定 `gateway.auth.token`、`gateway.auth.password`、`OPENCLAW_GATEWAY_TOKEN` 或 `OPENCLAW_GATEWAY_PASSWORD`。

遠端 CDP 提示：

- 請盡可能優先使用加密端點（HTTPS 或 WSS）和短期權杖。
- 避免將長期權杖直接嵌入設定檔。
- 將閘道和所有節點主機保留在私有網路（Tailscale）中；避免公開暴露。
- 將遠端 CDP URL／權杖視為密鑰；請優先使用環境變數或密鑰管理工具。

## 設定檔（多瀏覽器）

OpenClaw 支援多個具名設定檔（路由設定）。設定檔可以是：

- **OpenClaw 管理**：專用的 Chromium 型瀏覽器執行個體，具有自己的使用者資料目錄和 CDP 連接埠
- **遠端**：明確的 CDP URL（在其他位置執行的 Chromium 型瀏覽器）
- **現有工作階段**：透過 Chrome DevTools MCP 自動連線使用你現有的 Chrome 設定檔

預設值：

- 如果缺少 `openclaw` 設定檔，系統會自動建立。
- `user` 設定檔是內建設定檔，用於 Chrome MCP 現有工作階段附加。
- 除 `user` 外，現有工作階段設定檔皆為選用；請使用 `--driver existing-session` 建立。
- 本機 CDP 連接埠預設從 **18800-18899** 範圍配置。
- 刪除設定檔時，其本機資料目錄會移至垃圾桶。

所有控制端點都接受 `?profile=<name>`；命令列介面使用 `--browser-profile`。

## 透過 Chrome DevTools MCP 使用現有工作階段

OpenClaw 也可以透過官方 Chrome DevTools MCP 伺服器，附加至正在執行的 Chromium 型瀏覽器設定檔。這會重複使用該瀏覽器設定檔中已開啟的分頁和登入狀態。

官方背景資訊和設定參考：

- [Chrome for Developers：搭配瀏覽器工作階段使用 Chrome DevTools MCP](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- [Chrome DevTools MCP README](https://github.com/ChromeDevTools/chrome-devtools-mcp)

內建設定檔：`user`。如果你想使用不同的名稱、顏色或瀏覽器資料目錄，請建立自己的自訂現有工作階段設定檔。

內建的 `user` 設定檔預設使用 Chrome MCP 自動連線，其目標為預設的本機 Google Chrome 設定檔。對於 Brave、Edge、Chromium 或非預設 Chrome 設定檔，請使用 `userDataDir`。`~` 會展開為你的作業系統家目錄：

```json5
{
  browser: {
    profiles: {
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
    },
  },
}
```

接著，在相符的瀏覽器中：

1. 開啟該瀏覽器用於遠端偵錯的檢查頁面。
2. 啟用遠端偵錯。
3. 讓瀏覽器保持執行，並在 OpenClaw 附加時核准連線提示。

常見檢查頁面：

- Chrome：`chrome://inspect/#remote-debugging`
- Brave：`brave://inspect/#remote-debugging`
- Edge：`edge://inspect/#remote-debugging`

即時附加冒煙測試：

```bash
openclaw browser --browser-profile user start
openclaw browser --browser-profile user status
openclaw browser --browser-profile user tabs
openclaw browser --browser-profile user snapshot --format ai
```

成功時的狀態：

- `status` 顯示 `driver: existing-session`
- `status` 顯示 `transport: chrome-mcp`
- `status` 顯示 `running: true`
- `tabs` 列出你已開啟的瀏覽器分頁
- `snapshot` 傳回所選即時分頁的參照

附加無法運作時的檢查事項：

- 目標 Chromium 架構瀏覽器的版本為 `144+`
- 該瀏覽器的檢查頁面已啟用遠端偵錯
- 瀏覽器已顯示附加同意提示，且你已接受
- 如果 Chrome 啟動時明確指定了 `--remote-debugging-port`，請將
  `browser.profiles.<name>.cdpUrl` 設為該 DevTools 端點，而不要依賴
  Chrome MCP 自動連線
- `openclaw doctor` 會遷移舊的擴充功能式瀏覽器設定，並檢查
  預設自動連線設定檔所需的 Chrome 是否安裝於本機，但無法代你
  啟用瀏覽器端的遠端偵錯

Agent 使用方式：

- 需要使用者已登入的瀏覽器狀態時，請使用 `profile="user"`。
- 如果使用自訂的現有工作階段設定檔，請傳入該明確的設定檔名稱。
- 只有當使用者位於電腦前、能核准附加提示時，才選擇此模式。
- 閘道或節點主機可以啟動 `npx chrome-devtools-mcp@latest --autoConnect`。

注意事項：

- 此路徑的風險高於隔離的 `openclaw` 設定檔，因為它可以
  在你已登入的瀏覽器工作階段中執行操作。
- OpenClaw 不會為此驅動程式啟動瀏覽器；它只會附加至瀏覽器。
- OpenClaw 在此使用官方 Chrome DevTools MCP `--autoConnect` 流程。如果
  已設定 `userDataDir`，系統會將其原樣傳遞，以指定該使用者資料目錄。
- 現有工作階段可以附加至所選主機，或透過已連線的
  瀏覽器節點附加。如果 Chrome 位於其他位置，且沒有連線任何瀏覽器節點，請改用
  遠端 CDP 或節點主機。
- Chrome MCP 目標與快照參照的範圍僅限於單一 MCP 子程序。該程序
  重新啟動後，請再次執行 `browser tabs`，在進行特定目標的操作前明確選取新的
  目標，並在使用參照前擷取新快照。
  每個參照僅對其目標及最新快照有效。即使替代分頁的 URL 相同，
  舊別名也不會轉移至該分頁。
- Chrome DevTools MCP 目前使用程序區域的數字頁面
  ID 路由頁面工具。程序範圍的控制代碼可防止跨子程序替換重複使用，但在相鄰工具呼叫之間
  於程序內替換瀏覽器內容時，仍可能重新指定動作的目標。要實現完全不可分割的路由，需要上游頁面工具
  支援穩定的目標 ID。

### 自訂 Chrome MCP 啟動方式

如果預設的 `npx chrome-devtools-mcp@latest` 流程不符合需求（離線主機、
固定版本、隨附的二進位檔），可依設定檔覆寫所啟動的 Chrome DevTools MCP 伺服器：

| 欄位        | 功能                                                                                                               |
| ------------ | -------------------------------------------------------------------------------------------------------------------------- |
| `mcpCommand` | 用來取代 `npx` 啟動的可執行檔。依原樣解析；支援絕對路徑。                                          |
| `mcpArgs`    | 原樣傳遞給 `mcpCommand` 的引數陣列。取代預設的 `chrome-devtools-mcp@latest --autoConnect` 引數。 |

在現有工作階段設定檔中設定 `cdpUrl` 後，OpenClaw 會略過
`--autoConnect`，並自動將端點轉送至 Chrome MCP：

- `http(s)://...` → `--browserUrl <url>`（DevTools HTTP 探索端點）。
- `ws(s)://...` → `--wsEndpoint <url>`（直接 CDP WebSocket）。

端點旗標不可與 `userDataDir` 合併使用：設定 `cdpUrl` 後，
啟動 Chrome MCP 時會忽略 `userDataDir`，因為 Chrome MCP 會附加至
端點後方正在執行的瀏覽器，而不是開啟設定檔
目錄。

<Accordion title="現有工作階段功能限制">

與受管理的 `openclaw` 設定檔相比，現有工作階段驅動程式受到較多限制：

- **螢幕截圖** - 頁面擷取與 `--ref` 元素擷取可正常運作；CSS `--element` 選擇器則無法使用。頁面或參照式元素螢幕截圖不需要 Playwright。（`--full-page` 在任何設定檔中都不能與 `--ref` 或 `--element` 合併使用，不僅限於現有工作階段。）
- **動作** - `click`、`type`、`hover`、`scrollIntoView`、`drag` 和 `select` 需要快照參照（不支援 CSS 選擇器）。`click-coords` 會點選可見檢視區座標，不需要快照參照。`click` 僅支援滑鼠左鍵（不支援按鈕覆寫或輔助按鍵）。`type` 不支援 `slowly=true`；請使用 `fill` 或 `press`。`press` 不支援 `delayMs`。`type`、`hover`、`scrollIntoView`、`drag`、`select` 和 `fill` 不支援每次呼叫的 `timeoutMs` 覆寫；`evaluate` 則支援。`select` 接受單一值。不支援 `batch`；請逐一傳送動作。
- **等待／上傳／對話方塊** - `wait --url` 支援完全相符、子字串及 glob 模式（與受管理模式相同）；現有工作階段設定檔不支援 `wait --load networkidle`（受管理及原始／遠端 CDP 設定檔支援）。上傳掛鉤需要 `ref` 或 `inputRef`，一次一個檔案，不支援 CSS `element`。對話方塊掛鉤不支援逾時覆寫或 `dialogId`。
- **對話方塊可見性** - 當動作開啟強制回應對話方塊時，受管理瀏覽器的動作回應會包含 `blockedByDialog` 和 `browserState.dialogs.pending`；快照也會包含待處理的對話方塊狀態。對話方塊待處理時，請使用 `browser dialog --accept/--dismiss --dialog-id <id>` 回應。在 OpenClaw 外部處理的對話方塊會顯示於 `browserState.dialogs.recent` 下。
- **僅限受管理模式的功能** - PDF 匯出、下載攔截及 `responsebody` 仍需使用受管理瀏覽器路徑。

</Accordion>

## 隔離保證

- **專用使用者資料目錄**：絕不存取你的個人瀏覽器設定檔。
- **專用連接埠**：避開 `9222`，防止與開發工作流程衝突。
- **確定性的分頁控制**：`tabs` 會先傳回 `suggestedTargetId`，接著傳回
  穩定的 `tabId` 控制代碼（例如 `t1`）、選用標籤，以及原始 `targetId`。
  Agent 應重複使用 `suggestedTargetId`；原始 ID 仍可用於
  偵錯及相容性用途。

## 瀏覽器選擇

在本機啟動時，OpenClaw 會選擇第一個可用的瀏覽器：

1. Chrome
2. Brave
3. Edge
4. Chromium
5. Chrome Canary

你可以使用 `browser.executablePath` 覆寫。

平台：

- macOS：檢查 `/Applications` 和 `~/Applications`。
- Linux：檢查 `/usr/bin`、
  `/snap/bin`、`/opt/google`、`/opt/brave.com`、`/usr/lib/chromium` 和
  `/usr/lib/chromium-browser` 下常見的 Chrome／Brave／Edge／Chromium 位置，以及
  `PLAYWRIGHT_BROWSERS_PATH` 或 `~/.cache/ms-playwright` 下由 Playwright 管理的 Chromium。
- Windows：檢查常見安裝位置。

## 控制 API（選用）

為了執行指令碼和偵錯，閘道提供小型的**僅限回送 HTTP
控制 API**，以及對應的 `openclaw browser` 命令列介面（快照、參照、增強等待功能、
JSON 輸出、偵錯工作流程）。完整參考資料請參閱
[瀏覽器控制 API](/zh-TW/tools/browser-control)。

## 疑難排解

如需 Linux 特有問題（尤其是 snap Chromium）的說明，請參閱
[瀏覽器疑難排解](/zh-TW/tools/browser-linux-troubleshooting)。

如需 WSL2 閘道與 Windows Chrome 分離主機設定的說明，請參閱
[WSL2 + Windows + 遠端 Chrome CDP 疑難排解](/zh-TW/tools/browser-wsl2-windows-remote-cdp-troubleshooting)。

### CDP 啟動失敗與導覽 SSRF 封鎖

這是兩種不同的失敗類別，分別指向不同的程式碼路徑。

- **CDP 啟動或就緒失敗**表示 OpenClaw 無法確認瀏覽器控制平面是否正常。
- **導覽 SSRF 封鎖**表示瀏覽器控制平面正常，但頁面導覽目標遭原則拒絕。

常見範例：

- CDP 啟動或就緒失敗：
  - `Chrome CDP websocket for profile "openclaw" is not reachable after start`
  - `Remote CDP for profile "<name>" is not reachable at <cdpUrl>`
  - 在未設定 `attachOnly: true` 的情況下設定
    回送外部 CDP 服務時出現 `Port <port> is in use for profile "<name>" but not by openclaw`
- 導覽 SSRF 封鎖：
  - `open`、`navigate`、快照或開啟分頁流程因瀏覽器／網路原則錯誤而失敗，但 `start` 和 `tabs` 仍可運作

請使用以下最短操作序列區分兩者：

```bash
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw tabs
openclaw browser --browser-profile openclaw open https://example.com
```

結果判讀方式：

- 如果 `start` 因 `not reachable after start` 而失敗，請先排解 CDP 就緒問題。
- 如果 `start` 成功，但 `tabs` 失敗，表示控制平面仍不正常。請將其視為 CDP 連線能力問題，而非頁面導覽問題。
- 如果 `start` 和 `tabs` 成功，但 `open` 或 `navigate` 失敗，表示瀏覽器控制平面已啟動，而問題出在導覽原則或目標頁面。
- 如果 `start`、`tabs` 和 `open` 均成功，表示基本的受管理瀏覽器控制路徑正常。

重要行為細節：

- 即使未設定 `browser.ssrfPolicy`，瀏覽器設定仍預設使用失敗時封閉的 SSRF 原則物件。
- 對於本機回送的 `openclaw` 受管理設定檔，CDP 健康狀態檢查會刻意略過 OpenClaw 自身本機控制平面的瀏覽器 SSRF 可連線性強制檢查。
- 導覽保護是獨立機制。`start` 或 `tabs` 成功，不代表後續的 `open` 或 `navigate` 目標會獲准。

安全性指引：

- 預設情況下，**請勿**放寬瀏覽器 SSRF 原則。
- 請優先採用 `hostnameAllowlist` 或 `allowedHostnames` 等範圍有限的主機例外，而非廣泛開放私人網路存取。
- 只有在刻意建立、需要且已審查私人網路瀏覽器存取權的可信任環境中，才使用 `dangerouslyAllowPrivateNetwork: true`。

## Agent 工具與控制運作方式

Agent 會取得**一項工具**來進行瀏覽器自動化：

- `browser` - 診斷／狀態／啟動／停止／分頁／開啟／聚焦／關閉／快照／螢幕截圖／導覽／動作

對應方式：

- `browser snapshot` 會傳回穩定的 UI 樹狀結構（AI 或 ARIA）。
- `browser act` 使用快照的 `ref` ID 進行點擊、輸入、拖曳或選取。
- `browser screenshot` 擷取像素（完整頁面、元素或帶標籤的參照）。
- `browser doctor` 檢查閘道、外掛、設定檔、瀏覽器和分頁是否就緒。
- `browser` 接受：
  - `profile`，用於選擇具名瀏覽器設定檔（openclaw、chrome 或遠端 CDP）。
  - `target`（`sandbox` | `host` | `node`），用於選擇瀏覽器的執行位置。
  - 在沙箱化工作階段中，`target: "host"` 需要 `agents.defaults.sandbox.browser.allowHostControl=true`。
  - 若省略 `target`：沙箱化工作階段預設為 `sandbox`，非沙箱工作階段預設為 `host`。
  - 若已連線具備瀏覽器功能的節點，除非你固定使用 `target="host"` 或 `target="node"`，否則工具可能會自動將工作路由至該節點。

這可讓代理程式保持確定性，並避免使用容易失效的選擇器。

## 相關內容

- [工具概覽](/zh-TW/tools) - 所有可用的代理程式工具
- [沙箱化](/zh-TW/gateway/sandboxing) - 沙箱化環境中的瀏覽器控制
- [安全性](/zh-TW/gateway/security) - 瀏覽器控制的風險與強化措施
