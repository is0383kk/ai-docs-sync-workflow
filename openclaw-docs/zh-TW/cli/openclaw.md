---
read_when:
    - 你已完成推論設定，並希望 OpenClaw 設定其餘部分
    - 你需要使用本機設定代理程式檢查或修復 OpenClaw
    - 你正在設計或啟用訊息頻道救援模式
summary: 由推論支援的 OpenClaw 設定與修復輔助工具之命令列介面參考與安全模型
title: OpenClaw 設定代理程式
x-i18n:
    generated_at: "2026-07-26T07:46:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9578d1493ff514ea6dd07dae995bf83443e9e17f2c2134bc801faa45254615bf
    source_path: cli/openclaw.md
    workflow: 16
---

# `openclaw setup`

OpenClaw 隨附一個內建的系統代理程式——它以「OpenClaw」的身分發言——用於本機設定、修復與組態（先前稱為 Crestodian）。它只會在有效的預設模型完成一次真實回合後啟動。
全新安裝會先建立推論能力；格式錯誤的組態仍沿用傳統的 Doctor 流程。

## 啟動時機

執行不含子命令的 `openclaw` 時，會根據組態狀態決定流程：

- 組態不存在，或雖然存在但沒有使用者撰寫的設定（空白，或僅含 `$schema`/`meta` 鍵）：啟動具備即時 AI 驗證的引導式新手設定。
- 組態存在但驗證失敗：啟動傳統新手設定，回報問題並引導你使用 `openclaw doctor`。
- 組態存在且有效：開啟一般代理程式終端介面。若可連線至已設定的閘道，且其預設代理程式具有模型，便會直接進入該介面，
  不經過新手設定或 OpenClaw。之後可在終端介面內使用 `/openclaw`，或直接執行
  `openclaw setup`，以進入 OpenClaw。

執行 `openclaw setup` 時，會先對已設定的預設模型進行即時測試。回合通過後即啟動 OpenClaw。若互動式測試失敗，則會開啟引導式推論設定，並在候選項通過後交由 OpenClaw 接手。當推論不可用時，單次、JSON 與其他非互動式要求會失敗，並指示執行 `openclaw onboard`。`openclaw --help` 與 `openclaw --version` 仍沿用其一般快速流程。

非互動式執行不含參數的 `openclaw`（無 TTY）時，會顯示簡短訊息後結束，而非印出根命令說明：若為全新或無效的安裝，訊息會指向非互動式新手設定；若組態有效，則指向 `openclaw agent --local ...`。

`openclaw onboard --modern` 仍是 OpenClaw 的相容性別名，但會使用相同的推論閘門：推論正常時開啟聊天；互動式失敗時啟動引導式推論設定；非互動式失敗時則顯示新手設定指引並結束。`openclaw onboard --classic` 會開啟完整的逐步精靈。

## OpenClaw 顯示的內容

互動式 OpenClaw 會開啟與 `openclaw tui` 相同的終端介面殼層，並使用 OpenClaw 聊天後端。啟動問候語涵蓋：

- 組態有效性與預設代理程式
- OpenClaw 正在使用的已驗證模型
- 第一次啟動探測所確認的閘道可連線性
- 下一個建議的偵錯動作

它不會傾印密鑰，也不會僅為了啟動而載入外掛命令列介面命令。

使用 `status` 可查看詳細清單：組態路徑、文件／原始碼路徑、本機命令列介面探測、金鑰／權杖是否存在、代理程式、模型，以及閘道詳細資訊。

OpenClaw 使用與一般代理程式相同的參考資料探索方式：在 Git 簽出版本中，它會指向本機 `docs/` 與原始碼樹狀結構；在 npm 安裝版本中，它會使用隨附文件並連結至 [https://github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)，同時指引你在文件不足時查閱原始碼。

## 範例

```bash
openclaw
openclaw setup
openclaw setup --json
openclaw setup --message "models"
openclaw setup --message "validate config"
openclaw setup --message "setup workspace ~/Projects/work" --yes
openclaw setup --message "set default model openai/gpt-5.6" --yes
openclaw onboard --modern
```

在 OpenClaw 終端介面內：

```text
狀態
健康狀態
Doctor
驗證組態
設定
設定工作區 ~/Projects/work
組態設定 gateway.port 19001
組態設定參照 gateway.auth.token env OPENCLAW_GATEWAY_TOKEN
閘道狀態
重新啟動閘道
代理程式
建立代理程式 work 工作區 ~/Projects/work
模型
設定模型供應商
將預設模型設為 openai/gpt-5.6
頻道
頻道資訊 slack
連線 slack
開啟 slack 的頻道精靈
外掛清單
搜尋 slack 外掛
安裝外掛 clawhub:openclaw-codex-app-server
與 work 代理程式交談
與 ~/Projects/work 的代理程式交談
稽核
結束
```

## 操作與核准

OpenClaw 使用具型別的操作，而非臨時編輯組態。

唯讀操作會立即執行：顯示概覽、列出代理程式、列出已安裝的外掛、搜尋 ClawHub 外掛、顯示模型／後端狀態、執行狀態／健康狀態檢查、檢查閘道可連線性、執行不含互動式修復的 Doctor、驗證組態，以及顯示稽核記錄路徑。

啟動引導式頻道設定（`connect telegram`）也會立即執行。其精靈會收集明確的回答，並負責後續寫入。

持久性操作需要對話式核准（直接命令則可使用 `--yes`）：寫入組態、`config set`、`config set-ref`、設定／新手設定啟動程序、變更預設模型、啟動／停止／重新啟動閘道、建立代理程式，以及安裝外掛。

OpenClaw 內無法使用 Doctor 修復，因為這類修復可能會改寫支撐目前工作階段的供應商、驗證或預設代理程式推論路由。請結束 OpenClaw，並在終端機中執行 `openclaw doctor --fix`。唯讀的 `doctor` 在 OpenClaw 內仍可使用。

新代理程式會繼承經即時驗證的預設推論路由。代理程式 ID `openclaw` 與 `crestodian` 保留給系統代理程式，無法建立為一般代理程式。已停用的 ID 仍會遭到封鎖，以免舊組態占用它。

`config set` 與 `config set-ref` 可以變更使用者可變更的任何設定，
但有一份簡短且僅供人工判斷的拒絕清單：`$include`、`auth.*`、`env.*`、`models.*`
與 `secrets.*` 仍會遭拒，因為它們包含認證資訊材料、
替代組態引入項目，或提供給
推論路由的供應商／目錄定義。推論路由本身也受到保護：預設模型
路由（`agents.defaults` 的模型／參數／執行階段欄位）及支撐目前有效預設路由之代理程式的路由欄位
會遭拒，代理程式
身分／拓撲欄位（`id`、`agentDir`、`default`）亦同。其他代理程式的路由欄位
在核准後仍可寫入。閘道與頻道驗證仍是
一般組態介面。對於已設定的路由，請使用 `set default model <provider/model>`；
它會在儲存前對路由進行即時測試。若要設定或修復供應商／驗證存取權，請結束 OpenClaw 並執行
`openclaw onboard`。

`plugins.entries.<id>.*` 寫入（啟用／停用／設定已安裝的外掛）
皆可執行，除非該外掛支撐目前有效的推論路由。外掛
安裝來源與載入原則會在具型別的
外掛安裝工作流程中維持其信任邊界。基於相同原因，系統也會拒絕解除安裝
支撐路由的外掛；請結束 OpenClaw，並從終端機執行
`openclaw plugins uninstall <id>`。

請用自己的話表達核准：明確無歧義的回覆（「是」、「可以」、「繼續」、「現在不要」）會依封閉且確定性的清單進行判定。當已設定的路由支援獨立的完成呼叫時，其他回覆可以僅根據你的訊息與待處理提案進行分類——絕不由對話模型自行判定，因為它不能自行核准。無法分類或語意模糊的回覆會讓提案保持待處理狀態，對話也會再次詢問。

### 變更歷程

「詢問 OpenClaw」頁面可顯示近期已套用的系統代理程式操作、Doctor
遷移、「設定」與命令列介面組態寫入，以及對
`openclaw.json` 的手動編輯。組態日誌會在閘道
監看期間、OpenClaw 所擁有的寫入期間，或離線編輯後的
下次啟動時偵測外部編輯。

歷程儲存於共用
`~/.openclaw/state/openclaw.sqlite` 資料庫的 `diagnostic_events` 資料表中，位於 `system-agent-audit`
與 `config-audit` 範圍下。每個範圍會保留最新的 50,000 筆記錄。
探索與唯讀操作不包含在內。密鑰絕不會出現在
變更歷程中；組態日誌記錄包含的是變更路徑，而非組態
值，且值的比較會使用受保護的指紋。

頻道設定可採用託管對話執行，直到遇到密鑰為止。
本機 OpenClaw 終端介面不接受精靈中的敏感回答，因為終端機
聊天輸入為可見內容。它會立即提供 `open channel wizard`，並將
所選頻道帶入遮蔽式終端機精靈；你也可以稍後執行
`openclaw channels add --channel <channel>`。

### 切換至遮蔽式頻道設定

本機聊天可以將控制權交給遮蔽式頻道精靈：

```text
開啟 slack 的頻道精靈
頻道資訊 slack
```

`open channel wizard for <channel>` 會在聊天
終端介面關閉後開啟遮蔽式頻道設定。請先使用 `channel info <channel>` 查看頻道標籤、設定
狀態、先決條件摘要與文件連結。

OpenClaw 絕不會從自己的工作階段內變更供應商／驗證存取權：
該工作階段本身已依賴此推論路由。對於模型供應商設定或
修復，`configure model provider` 會傳回結束／新手設定指引，而不會
啟動精靈或寫入組態。請結束 OpenClaw 並執行 `openclaw
onboard`；新手設定會暫存認證資訊，且只會儲存
能完成一次真實即時回合的路由。新手設定成功後，請再次啟動 OpenClaw。

## 設定啟動程序

在引導式新手設定已建立推論能力後，`setup` 會設定剩餘的工作區與閘道狀態。它只會透過具型別的組態操作寫入，並會先要求核准。

```text
設定
設定工作區 ~/Projects/work
```

`setup` 會保留已驗證的有效模型。它不會設定或
取代推論。

若缺少推論能力或即時檢查失敗，請離開 OpenClaw 並執行 `openclaw onboard`。引導式新手設定會先嘗試已設定的模型，接著嘗試已驗證的訂閱命令列介面、API 金鑰及其餘受支援的命令列介面；它會要求每個候選項提供真實回覆，且只會保存通過的路由。通過此邊界後，OpenClaw 會立即啟動，隨後便可設定工作區、閘道、頻道、代理程式、外掛及其他選用功能。

當 macOS App 連線至已設定的閘道，
且其預設代理程式已有設定好的模型時，會完全略過此階梯，直接開啟一般代理程式
介面。
對於全新或尚未完成設定的閘道，App 會透過
`openclaw.setup.detect` 與 `openclaw.setup.activate` 閘道方法執行推論階梯：
detect 會列出它找到的每個候選後端，activate 則會即時測試一個
候選項（真實執行「回覆 OK」的完成要求），並只在測試通過後保存該路由所需的模型、
認證資訊與供應商／執行階段狀態。工作區與閘道預設值仍交由 OpenClaw 處理。失敗的候選項
絕不會變更組態；App 會自動沿階梯逐項嘗試，最後
提供手動金鑰／權杖步驟，其中的選項來自閘道目前啟用的
文字推論供應商外掛。所選供應商會擁有其起始模型
與組態，而認證資訊也會在儲存前以相同方式驗證。

Codex 監督與其他選用外掛功能不屬於此
推論啟用交易。請僅在推論正常運作且 OpenClaw
已啟動後進行設定；現有外掛原則與明確的
監督退出設定在推論設定期間會保持不變。

## AI 對話

互動式 OpenClaw 的自由形式對話會經由與一般 OpenClaw 代理程式相同的代理程式迴圈執行，但僅限使用一個具備 ring-zero OpenClaw 權限的工具 `openclaw`，由該工具包裝具型別的操作。讀取動作可自由執行；變更則需要你針對該項確切操作進行對話式核准（請參閱「操作與核准」）；每次套用的寫入都會經過稽核與重新驗證。代理程式工作階段會持續保留，因此 OpenClaw 具備真正的多回合記憶。若已驗證的推論路由之後停止運作，請返回 `openclaw onboard` 並修復它，再繼續操作。

主機不會將自然語言要求剖析成操作。自由形式
訊息——包括看似命令的文字，以及「我的
閘道為什麼停止了？」之類的問題——會傳送給 AI，由 AI 透過
`openclaw` 工具將要求對應至具型別的操作。

當變更操作待處理時，只有封閉清單中語意明確的核准或拒絕用語，才會在不進行推論的情況下解析。語意模糊的同意會交由另一個已設定的補全呼叫處理，否則會以封閉方式失敗。結構化精靈欄位和精確的主機導覽屬於 UI 控制項，而非自然語言操作解析。有一項認證資訊衛生的例外尤其重要：敏感路徑（權杖、金鑰、密碼）上的精確 `config set` 絕不會傳送至模型。主機會建立經遮蔽的提案，且該值會在 AI 可見的歷程記錄中遮罩。機密資料請優先使用 `config set-ref <path> env <ENV_VAR>`。

訊息頻道救援模式絕不使用模型輔助規劃器。遠端救援維持確定性，因此無法利用已損壞或遭入侵的一般代理路徑作為設定編輯器。

### 命令列介面控管程式信任模型

內嵌執行階段與 Codex app-server 控管程式會直接強制執行零環限制：該次執行會攜帶僅含 `openclaw` 工具的 OpenClaw 工具允許清單。對於 Codex，OpenClaw 也會在該次執行中停用環境、原生執行、多代理、目標、應用程式／外掛、Skill／MCP、網頁搜尋及 `request_user_input` 介面。Codex 仍會注入其無作用的原生 `update_plan` 公用程式；它可以更新模型的暫時檢查清單，但無法寫入檔案或 OpenClaw 設定。命令列介面控管程式不會使用 OpenClaw 的允許清單，因此 OpenClaw 只接受其自身工具選擇契約能證明相同限制的後端：

- 可選後端（包括 Claude Code）會以空白的原生工具選擇和一個 MCP 工具 `openclaw` 啟動。Claude 產生的 MCP 設定會使用 `--strict-mcp-config` 套用，因此不會載入其他 MCP 伺服器。
- 宣告不含原生工具的後端會收到相同的專用 OpenClaw MCP 伺服器。
- 原生工具永遠啟用或原生工具狀態未知的後端會在推論前以封閉方式失敗；它們無法代管 OpenClaw 工作階段。

只有 OpenClaw 工作階段會取得 openclaw MCP 伺服器；一般代理執行絕不會看到此工具。因此，可選／無原生命令列介面後端和 API 金鑰模型會強制執行字面上的單一工具迴圈。Codex app-server 模型則會強制使用單一 OpenClaw 權限工具，加上無作用的原生規劃公用程式。在這三種情況下，設定寫入仍受限於 OpenClaw 經稽核的核准契約。

Gemini CLI 仍可供一般代理使用，但它無法強制執行推論閘門所需的無工具探查，因此無法代管 OpenClaw。

## 切換至代理

使用自然語言選擇器離開 OpenClaw 並開啟一般終端介面：

```text
與代理交談
與工作代理交談
切換至主要代理
```

`openclaw tui`、`openclaw chat` 和 `openclaw terminal` 會直接開啟一般代理終端介面；它們不會啟動 OpenClaw。切換至一般終端介面後，`/openclaw` 會返回 OpenClaw，並可選擇附帶後續要求：

```text
/openclaw
/openclaw restart gateway
```

## 訊息救援模式

訊息救援模式是 OpenClaw 的訊息頻道進入點：當你的一般代理已停止運作，但受信任的頻道（例如 WhatsApp）仍能接收命令時，請使用此模式。

這是確定性的緊急命令處理常式，而非對話式 OpenClaw 代理。它不會啟動全新的設定，也不會放寬 OpenClaw 聊天的推論閘門。

支援的命令：`/openclaw <request>`。救援僅接受精確輸入的命令文法——自然語言會遭拒絕並顯示提示，絕不會猜測成某項操作，也絕不會查詢模型。

```text
你，在受信任的擁有者私訊中：/openclaw status
OpenClaw：OpenClaw 救援模式。閘道可連線：否。設定有效：否。
你：/openclaw restart gateway
OpenClaw：計畫：重新啟動閘道。回覆 /openclaw yes 以套用。
你：/openclaw yes
OpenClaw：已套用。已寫入稽核項目。
```

也可以在本機或透過救援將代理建立作業排入佇列：

```text
create agent work workspace ~/Projects/work model openai/gpt-5.6-sol
/openclaw create agent work workspace ~/Projects/work
```

建立代理時，只能指定目前經即時驗證的預設模型。省略模型即可繼承該路由。

遠端救援是管理介面，必須視同遠端設定修復，而非一般聊天。

遠端救援的安全性契約：

- 代理／工作階段啟用沙箱時停用；OpenClaw 會拒絕遠端救援，並引導使用本機命令列介面修復。
- 預設有效狀態為 `auto`：僅在受信任的 YOLO 操作中允許遠端救援，此時執行階段已具有未受沙箱限制的本機權限（`tools.exec.security` 解析為 `full`，而 `tools.exec.ask` 解析為 `off`，沙箱模式為 `off`）。
- 需要明確的擁有者身分；不允許萬用字元傳送者規則、開放群組政策、未驗證的網路鉤子或匿名頻道。
- 救援僅限擁有者私訊。
- 外掛搜尋與清單為唯讀。外掛安裝一律僅限本機（即使在其他情況下已啟用，仍會在救援模式中封鎖），因為它會下載可執行程式碼。本機 OpenClaw 與救援模式都會拒絕解除安裝外掛；請從終端機執行 `openclaw plugins uninstall <id>`。
- 遠端救援無法開啟本機終端介面或切換至互動式代理工作階段；請使用本機 `openclaw` 交接代理。
- 即使在救援模式中，持久性寫入仍需核准。
- 待處理的核准只能使用一次。同一帳號、頻道與傳送者的任何較新救援命令，都會撤銷較舊的計畫；執行失敗也會耗用核准，因此若要重試，請重新傳送命令。
- 每項已套用的救援操作都會接受稽核。訊息頻道救援會記錄頻道、帳號、傳送者與來源位址中繼資料；變更設定的操作也會記錄變更前後的設定雜湊。
- 絕不回顯機密資料。SecretRef 檢查只會報告可用性，不會回報值。
- 若閘道仍在運作，救援會優先使用閘道的型別化操作；若閘道已停止運作，救援只會使用不依賴一般代理迴圈的最小本機修復介面。

救援政策為內建政策：只有在有效執行階段為 YOLO、沙箱已關閉，且要求來自擁有者私訊時才可使用。待處理的寫入核准會在 15 分鐘後到期。`openclaw doctor --fix` 會移除已淘汰的 `systemAgent` 與 `crestodian` 設定區塊。

遠端救援由 Docker 測試路徑涵蓋：

```bash
pnpm test:docker:system-agent-rescue
```

選擇性啟用的即時頻道命令介面冒煙測試會檢查 `/openclaw status`，以及透過救援處理常式完成一次持久性核准往返：

```bash
pnpm test:live:system-agent-rescue-channel
```

受推論閘門控管的封裝版單次設定由以下項目涵蓋：

```bash
pnpm test:docker:system-agent-first-run
```

該封裝版命令列介面測試路徑會從空白狀態目錄開始，並證明 OpenClaw 會在沒有推論時以封閉方式失敗。接著，它會透過封裝版啟用模組測試並啟用模擬 Claude。只有在此之後，模糊要求才會送達規劃器並解析為型別化設定，接著執行單次命令來建立額外代理、透過啟用外掛加上權杖 SecretRef 設定 Discord、驗證設定，以及檢查稽核記錄。此測試路徑提供閘門／操作的輔助證據；它不會測試互動式上線設定或 OpenClaw 代理／工具／核准對話。下方的 QA Lab 情境會重新導向相同的 Docker 測試路徑：

```bash
pnpm openclaw qa suite --scenario system-agent-ring-zero-setup
```

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [Doctor](/zh-TW/cli/doctor)
- [終端介面](/zh-TW/cli/tui)
- [沙箱](/zh-TW/cli/sandbox)
- [安全性](/zh-TW/cli/security)
