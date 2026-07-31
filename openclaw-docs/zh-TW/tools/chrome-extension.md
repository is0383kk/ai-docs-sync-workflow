---
read_when:
    - 你想要讓代理程式透過手機操控你實際已登入的 Chrome
    - 你持續遇到 Chrome 的「Allow remote debugging?」提示，但桌前沒有人操作
    - 你想瞭解透過擴充功能接管瀏覽器的安全模型
summary: Chrome 擴充功能：讓 OpenClaw 操控你已登入的 Chrome，且不顯示遠端偵錯提示
title: Chrome 擴充功能
x-i18n:
    generated_at: "2026-07-26T08:45:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3d974f62bb5697a23dd6a6852137ce6af5a8a4a2a8ff738eec0098f259e8faa0
    source_path: tools/chrome-extension.md
    workflow: 16
---

# Chrome 擴充功能

OpenClaw Chrome 擴充功能可讓代理程式控制你**已登入的 Chrome
分頁**，無須啟動個別的受管理瀏覽器，也**不會**出現 Chrome
會阻擋操作的「Allow remote debugging?」提示。

當你從手機（Telegram、WhatsApp 等）操作 OpenClaw 時，這一點很重要：
[`user` 設定檔](/zh-TW/tools/browser#profiles-openclaw-user-chrome)會透過
Chrome 的遠端偵錯連接埠連線，這會在桌面上彈出同意對話方塊，而你不在電腦旁時
沒有人能按下確認。擴充功能則改用 `chrome.debugger` API，
因此頁面內唯一的提示是 Chrome 可關閉的「OpenClaw started debugging
this browser」橫幅。

Anthropic 的 Claude in Chrome 和 OpenAI 的 Codex
Chrome 擴充功能也採用相同的架構。

## 運作方式

包含三個部分：

- **瀏覽器控制服務**（閘道或節點主機）：供 `browser`
  工具呼叫的 API。
- **擴充功能轉送器**（迴送 WebSocket）：由控制服務在
  `127.0.0.1` 上啟動的小型伺服器。它向 OpenClaw 提供 Chrome DevTools Protocol
  端點，並與擴充功能通訊。雙方都使用主機本機的權杖進行驗證（請參閱下文）。
- **OpenClaw Chrome 擴充功能**（MV3）：使用 `chrome.debugger`
  連接分頁、轉送 CDP 流量，並管理 **OpenClaw 分頁群組**。

OpenClaw 只能看到並控制 **OpenClaw 分頁群組**中的分頁。
此群組是同意授權的界線：將分頁拖入即可分享，拖出（或按一下
工具列按鈕）即可立即撤銷存取權。

## 安裝與配對

1. 顯示未封裝擴充功能的路徑：

   ```bash
   openclaw browser extension path
   ```

2. 開啟 `chrome://extensions`、啟用 **Developer mode**、按一下 **Load
   unpacked**，然後選取顯示的目錄。

3. 顯示配對字串：

   ```bash
   openclaw browser extension pair
   ```

4. 按一下 OpenClaw 工具列圖示，並將配對字串貼到彈出式視窗中。
   擴充功能連線至轉送器後，徽章會變成 **ON**。

配對權杖是首次使用時建立並儲存於狀態目錄中
`credentials/` 下的**主機本機機密**（模式為 `0600`）。每台執行
瀏覽器的機器（閘道主機以及每台瀏覽器節點主機）都有自己的
權杖，因此認證資訊無須在機器之間傳輸。若要輪替權杖，請刪除
`browser-extension-relay.secret` 檔案，然後重新配對。

## 使用方式

在 `browser` 工具呼叫中選取內建的 `chrome` 設定檔，或將其設為
預設值：

```bash
openclaw config set browser.defaultProfile chrome
```

```json5
{
  browser: {
    profiles: {
      chrome: { driver: "extension", color: "#FF4500" },
    },
  },
}
```

- 分享分頁：在該分頁上按一下 OpenClaw 工具列按鈕（它會加入
  OpenClaw 分頁群組），或將任意分頁拖入群組。
- 代理程式也可以開啟新分頁；這些分頁會自動加入群組。
- 撤銷存取權：再次按一下按鈕、將分頁拖出群組，或關閉
  Chrome 的偵錯橫幅。代理程式會立即失去該分頁的存取權。

### 分頁副駕駛側邊面板

配對擴充功能後，按一下其工具列彈出式視窗中的 **Open tab copilot**。
OpenClaw 會為該特定 Chrome 分頁設定 `sidepanel.html`；資訊清單未設定
全域側邊面板路徑。因此，每個分頁都會有個別的面板文件、
閘道工作階段、訊息訂閱，以及具型別的瀏覽器工具繫結。

面板不會在你的訊息中加入頁面 URL、標題、DOM 或可見文字。
它只會傳送你輸入的文字。瀏覽器動作會攜帶由閘道驗證的個別
繫結，其中包含 Chrome 分頁和 CDP 目標；瀏覽器工具會拒絕
取代該目標或使用瀏覽器全域動作的嘗試。回覆會保留在面板中
（`deliver: false`）；不會沿用 Telegram、Discord 或其他頻道路由。

副駕駛是專用的已配對閘道裝置，具有 `operator.read` 和
`operator.write` 範圍。首次使用時，請檢查並核准其要求：

```bash
openclaw devices list
openclaw devices approve <requestId>
```

擴充功能會保留該裝置身分，以及閘道核發的裝置權杖，
且其範圍限定於核發它們的標準閘道端點。配對不同的
閘道會建立個別的身分、權杖和工作階段保管關係；認證資訊和
工作階段絕不會跨端點重複使用。擴充功能不會永久儲存
閘道共用機密。面板只能訂閱其自身的分頁工作階段，而
閘道會在傳遞前篩選這些事件。

如果執行期間閘道連線中斷，擴充功能會持續保管該執行 ID。
重新連線時，它會先中止尚未解決的執行，再重新啟用任何面板，
接著重新載入文字記錄歷程。此失效關閉步驟可避免瀏覽器動作
在傳遞中斷期間持續於不可見的情況下執行。

關閉分頁會立即移除其即時訂閱、中止任何可見的執行，並將
該分頁的工作階段標記為已封存。如果閘道暫時離線，擴充功能會
保留待處理的封存作業，並且只會在同一個閘道端點重新連線時重試；
絕不會將封存要求傳送至不同的閘道。瀏覽器當機後，下次啟動時
會封存前一個瀏覽器執行個體遺留的工作階段。已封存的工作階段會
拒絕新工作，但其文字記錄仍可在工作階段歷程中取得。瀏覽器副駕駛
金鑰屬於討論串工作階段，因此一般的存續時間與項目數量維護會保留它們。
每個代理程式的工作階段磁碟配額仍然適用（預設為 `2gb`），並可能在空間
不足時移除最舊的工作階段；請參閱[工作階段維護](/zh-TW/reference/session-management-compaction#store-maintenance-and-disk-controls)。

側邊面板目前需要由閘道託管的擴充功能轉送器，或直接連線至遠端
閘道的轉送器。瀏覽器節點上的迴送轉送器目前尚無法提供具型別分頁
繫結所需的節點路由，因此面板會拒絕此拓撲，而不會退回瀏覽器全域路由。

## 將頁面傳送至 OpenClaw

使用工具列彈出式視窗中的 **Send page to OpenClaw**，與你的主要
OpenClaw 工作階段分享可讀取的頁面文字。你可以新增選填備註、使用頁面或
所選內容的右鍵選單，或按下 `Alt+Shift+S`。若目前有所選內容，OpenClaw 會優先
使用它，將分享內容排入系統事件佇列，並立即喚醒主要工作階段。

分頁不需要位於 OpenClaw 分頁群組中。這是一次性的
明確分享：頁面上的其他內容不會公開，也不會授予持續存取權。
Google 文件會使用你已登入的瀏覽器工作階段匯出為純文字，
不需要設定 Google API。擷取 X 和 Twitter 討論串時不會包含
周圍的介面元素。

頁面文字會包覆在 OpenClaw 的外部內容安全界線中。你的
選填備註會以你自己的指示形式保留在該界線之外。頁面文字和
所選內容的上限約為 120,000 個字元，縮短時會包含截斷標記。

當擴充功能轉送器由閘道託管，並使用同主機配對或直接
`wss://` 閘道配對時，即可使用頁面分享。節點託管的轉送器目前會傳回
明確的錯誤。若要重新對應鍵盤快速鍵，請開啟
`chrome://extensions/shortcuts`。

## 遠端／跨機器

Chrome 不必在閘道主機上執行。支援三種拓撲：

- **同一主機**（閘道與 Chrome 位於同一台機器）：在該機器上使用
  `openclaw browser extension pair` 進行配對。轉送器僅限迴送連線。
  如果本機閘道使用 TLS，請透過 `--gateway-url wss://gateway-host.example` 明確傳入其憑證主機名稱；
  配對絕不會以迴送 IP 取代該名稱。
- **直接連線至遠端閘道**（Chrome 位於你的筆電、閘道位於 VPS，且
  **筆電上不需安裝任何其他項目**）：在閘道上執行
  `openclaw browser extension pair --gateway-url wss://your-gateway.example.com`。
  它會顯示 `wss://…/browser/extension#<secret>` 字串；在筆電上載入並配對
  擴充功能。擴充功能會透過 `wss://` **直接連線至閘道**
  — 筆電上不需要安裝 OpenClaw、節點、命令列介面，也不需要開放輸入連接埠。
  這是受管理託管的使用路徑。
- **透過瀏覽器節點主機**（Chrome 位於已執行 OpenClaw
  節點的機器上）：在節點上執行 `pair` 並於本機配對；閘道會透過
  現有且已驗證的節點連結，將瀏覽器動作代理至該節點。

配對機密依主機而定（直接連線時為閘道的機密），並由
閘道的 `/browser/extension` 路由驗證。若使用直接連線路徑，請透過 TLS
（`wss://`）提供閘道服務，以加密配對機密與 CDP 流量。
機密會保留在配對字串的 URL 片段中，並在 WebSocket 交握期間
作為子通訊協定認證資訊提供，因此一般的 Proxy 存取記錄不會在
要求 URL 中收到該機密。請確保所有反向 Proxy 都保留標準
`Sec-WebSocket-Protocol` 標頭。

## 診斷

```bash
openclaw browser status --browser-profile chrome
openclaw browser doctor --browser-profile chrome
```

在擴充功能彈出式視窗顯示 **Connected** 之前，`doctor`
會將 **Chrome extension relay** 檢查回報為失敗。

## 安全性模型

- 轉送器僅繫結至迴送介面；WebSocket 兩端都使用衍生權杖進行驗證，
  且擴充功能端會檢查來源是否為 `chrome-extension://`。
- 直接閘道配對不接受要求 URL 中的轉送權杖；
  內建擴充功能會改為透過 WebSocket 子通訊協定清單攜帶該權杖。
- 代理程式只能看到並操作 **OpenClaw 分頁群組**中的分頁。你的
  其他分頁會保持私密。
- 側邊面板執行具有兩層範圍限制：閘道傳遞使用各工作階段的
  允許清單，而瀏覽器工具會強制執行在提示之外攜帶的 Chrome 分頁／目標繫結。
- 相較於 `user`（Chrome MCP）設定檔，後者在你核准
  遠端偵錯提示後會公開整個已登入的瀏覽器，而此擴充功能會將分享範圍
  限制在你一眼即可掌控的分頁群組中。

另請參閱：[瀏覽器](/zh-TW/tools/browser)，以瞭解完整的設定檔模型，以及
受管理的 `openclaw` 和 Chrome MCP `user` 設定檔。
