---
read_when:
    - 安裝 macOS App
    - 在 macOS 上選擇本機或遠端閘道模式
    - 正在尋找 macOS App 版本下載檔案
summary: 安裝並使用 OpenClaw macOS 選單列應用程式
title: macOS 應用程式
x-i18n:
    generated_at: "2026-07-26T08:39:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b319d72bcbffcf91b6bc012d352c2cf647abd66e08ab0146cf98f5edfae3bca1
    source_path: platforms/macos.md
    workflow: 16
---

macOS App 是 OpenClaw 的**選單列夥伴**：提供原生系統匣介面、macOS
權限提示、通知、WebChat、語音輸入、Canvas，以及
由 Mac 託管的節點工具，例如 `system.run`。

使用 **Quick Chat**，無須開啟完整視窗即可使用類似 Spotlight 的主工作階段輸入框。預設按下 Option-Space (⌥Space)、從選單列選單選取，或在 **Settings → General** 中錄製其他快捷鍵。

只需要命令列介面和閘道？請從[開始使用](/zh-TW/start/getting-started)著手。

## 下載

從 [OpenClaw GitHub 發行版本](https://github.com/openclaw/openclaw/releases)取得 macOS App 組建。
當發行版本包含 macOS App 資產時，請尋找：

- `OpenClaw-<version>.dmg`（建議）
- `OpenClaw-<version>.zip`

部分發行版本只包含命令列介面、證明資料或 Windows 資產。如果最新發行版本
沒有 macOS App 資產，請使用包含該資產的最新版本，或依照
[macOS 開發設定](/zh-TW/platforms/mac/dev-setup)從原始碼組建。

## 首次執行

1. 安裝並啟動 **OpenClaw.app**。
2. 為本機閘道選擇 **This Mac**，或連線至遠端閘道。
3. 等待 App 安裝相符的命令列介面執行環境。在本機模式下，它也會
   安裝並啟動閘道。
4. 透過即時模型檢查建立推論連線。通過檢查後，OpenClaw
   會處理其餘設定。
5. 完成 macOS 權限檢查清單，並傳送新手引導測試訊息。

如果 App 連線到現有閘道，且其預設代理程式已設定
模型，它會將該閘道視為已完成設定、略過供應商新手引導和
OpenClaw，並開啟儀表板。如果無法連線至閘道，或其
預設代理程式沒有模型，推論新手引導仍可用於
復原。

若要使用命令列介面／閘道設定流程，請參閱[開始使用](/zh-TW/start/getting-started)。
若要復原權限，請參閱 [macOS 權限](/zh-TW/platforms/mac/permissions)。

## 更新

儀表板更新卡片會標示 App 將更新的項目：

- **Update Mac app + Gateway** 表示已簽署的 App 擁有本機 launchd
  閘道。Sparkle 會先更新 App；App 重新啟動後，會自動
  將其閘道更新至相符版本並重新啟動，接著驗證
  連線。
- **Update Gateway** 表示 App 已連線至遠端閘道、手動
  管理的本機閘道，或 App 不擁有的其他安裝項目。此按鈕
  會執行該閘道的一般更新流程，而不變更 Mac App。

協調更新失敗時，會停留在其設定樣式視窗，並提供重試、
[更新指南](/zh-TW/install/updating)和 Discord 操作。自動修復絕不會
將較新的閘道降級，也不會覆寫 `extended-stable` 通道釘選。

成功更新後，App 會找出最近由真人使用的
最上層直接工作階段，並向該代理程式提供一次性更新事件。心跳偵測
和排程活動不會影響此選擇。接著，代理程式可以從你最可能使用的
對話中歡迎你回來。在遠端模式下，App
只會更新本機 Mac 節點執行環境；如果遠端閘道比 App 舊，
則會略過通知。

Sparkle 會遵循閘道的 `update.channel` 設定。`beta` 和 `dev` 會選擇加入
Beta App 組建；`stable`、`extended-stable`，以及缺少或未知的值
則會維持使用穩定版 App 組建。

## 開啟儀表板連結

在 macOS App 的內嵌儀表板中，按一下外部網頁連結會在可調整大小的瀏覽器側邊欄中開啟，預設占視窗寬度的一半，同時保留儀表板導覽。拖曳分隔線即可選擇其他寬度；App 會記住該設定。每個連結都會在自己的分頁中開啟；開啟多個頁面時會顯示分頁列，再次按一下相同連結會重複使用現有分頁。拖曳分頁可重新排序，使用分頁關閉按鈕或按一下滑鼠中鍵即可關閉分頁；在分頁上按一下滑鼠右鍵可使用 **Open in Default Browser**、**Copy Link**、**Reload**、**Close Tab** 和 **Close Other Tabs**。視窗標題列的上一頁／下一頁控制項和觸控式軌跡板滑動手勢會導覽儀表板歷程記錄；側邊欄本身的上一頁／下一頁控制項則會導覽目前分頁的歷程記錄。側邊欄也提供重新載入、在預設瀏覽器中開啟及關閉控制項。

標題列控制項會配合 App 側邊欄：展開時，上一頁／下一頁位於其右側邊緣、緊鄰側邊欄切換按鈕；收合時，這些控制項會讓位給搜尋按鈕（開啟命令選擇區）和新增工作階段按鈕。

在外部連結上按一下滑鼠右鍵，可選擇 **Open in Sidebar**、**Open in Default Browser** 或 **Copy Link**。從儀表板進行的輔助按鍵點擊，以及由使用者啟動的新視窗連結，仍會在預設瀏覽器中開啟；側邊欄內的新視窗連結則會以新的側邊欄分頁開啟。一般由瀏覽器託管的 Control UI 頁面會保留瀏覽器的一般連結與內容選單行為。

## 匯入瀏覽器登入資料

當 App 連線至本機閘道時，瀏覽器側邊欄首次開啟之際，如果 Mac 上存在含 Cookie 的 Chrome 系列設定檔，儀表板會顯示可關閉的橫幅。此橫幅可將這些 Cookie 複製到代理程式用於瀏覽的隔離式受管理設定檔。從其 **Import** 控制項選擇設定檔（可能需要 Touch ID）；進度和已匯入的 Cookie 數量會顯示在行內，而且只會複製 Cookie——密碼絕不會離開來源瀏覽器。關閉橫幅會記錄此選擇；**Settings → General → Browser login → Import…** 可隨時再次提供此功能。請參閱[瀏覽器](/zh-TW/cli/browser)，了解底層匯入流程和 `browser.allowSystemProfileImport` 閘門。

## 選擇閘道模式

| 模式   | 適用時機                                                                       | 詳細資訊頁                                       |
| ------ | ------------------------------------------------------------------------------ | -------------------------------------------------- |
| 本機   | 此 Mac 應執行閘道，並透過 launchd 使其持續運作。                              | [macOS 上的閘道](/zh-TW/platforms/mac/bundled-gateway) |
| 遠端   | 由其他主機執行閘道；此 Mac 透過 SSH、LAN 或 Tailnet 控制它。                  | [遠端控制](/zh-TW/platforms/mac/remote)                |

兩種模式都需要已安裝的 `openclaw` 命令列介面，因為 App 會重複使用其節點主機
執行環境。在全新的 Mac 上，App 會自動安裝相符的命令列介面；接著，本機
模式會啟動閘道精靈，而遠端模式則會連線至所選
閘道，不會啟動第二個本機閘道。
若要手動復原，請參閱 [macOS 上的閘道](/zh-TW/platforms/mac/bundled-gateway)。

## App 負責的項目

- 選單列狀態、通知、健康狀態、WebChat，以及浮動的 Quick Chat 列。
- 螢幕、麥克風、語音、Automation 和 Accessibility 的 macOS 權限提示。
- 一個 Mac 節點，將原生 Canvas、相機／螢幕擷取、通知、
  位置和電腦控制，與命令列介面節點主機的系統、瀏覽器、
  外掛、技能和 MCP 命令結合。
- Mac 託管命令的執行核准提示。
- 在 App 情境中執行已核准的 Shell 命令，在命令列介面執行環境
  負責共用節點原則的同時，保留 App 的 macOS 權限歸屬。
- 遠端模式 SSH 通道或直接閘道連線。

在內嵌 Control UI 中，**Settings → Notifications** 會顯示 App 的原生
通知權限，而不是瀏覽器推播，因為 App 會以原生方式傳遞通知。

App **不會**取代閘道或一般命令列介面文件。閘道
設定、供應商、外掛、通道、工具和安全性各有
專屬文件。

## macOS 詳細資訊頁

| 工作                                     | 閱讀                                                                                        |
| ---------------------------------------- | ------------------------------------------------------------------------------------------- |
| 安裝或偵錯命令列介面／閘道服務           | [macOS 上的閘道](/zh-TW/platforms/mac/bundled-gateway)                                          |
| 避免將狀態存放在雲端同步資料夾中         | [macOS 上的閘道](/zh-TW/platforms/mac/bundled-gateway#state-directory-on-macos)                 |
| 偵錯 App 探索和連線能力                   | [macOS 上的閘道](/zh-TW/platforms/mac/bundled-gateway#debug-app-connectivity)                   |
| 了解 launchd 行為                        | [閘道生命週期](/zh-TW/platforms/mac/child-process)                                               |
| 修正權限或簽署／TCC 問題                  | [macOS 權限](/zh-TW/platforms/mac/permissions)                                                   |
| 偵測你最近使用的 Mac                     | [使用中電腦狀態](/zh-TW/nodes/presence)                                                          |
| 連線至遠端閘道                           | [遠端控制](/zh-TW/platforms/mac/remote)                                                          |
| 查看選單列狀態和健康狀態檢查             | [選單列](/zh-TW/platforms/mac/menu-bar)、[健康狀態檢查](/zh-TW/platforms/mac/health)                   |
| 使用內嵌聊天介面                         | [WebChat](/zh-TW/platforms/mac/webchat)                                                           |
| 使用語音喚醒或按住說話                   | [語音喚醒](/zh-TW/platforms/mac/voicewake)                                                       |
| 使用 Canvas 和 Canvas 深層連結            | [Canvas](/zh-TW/platforms/mac/canvas)                                                             |
| 託管 PeekabooBridge 以進行 UI 自動化      | [Peekaboo 橋接器](/zh-TW/platforms/mac/peekaboo)                                                 |
| 設定命令核准                             | [執行核准](/zh-TW/tools/exec-approvals)、[進階詳細資訊](/zh-TW/tools/exec-approvals-advanced)          |
| 檢查 Mac 節點命令和 App IPC               | [macOS IPC](/zh-TW/platforms/mac/xpc)                                                             |
| 擷取記錄                                 | [macOS 記錄](/zh-TW/platforms/mac/logging)                                                       |
| 從原始碼組建                             | [macOS 開發設定](/zh-TW/platforms/mac/dev-setup)                                                 |

## 相關內容

- [平台](/zh-TW/platforms)
- [開始使用](/zh-TW/start/getting-started)
- [閘道](/zh-TW/gateway)
- [執行核准](/zh-TW/tools/exec-approvals)
