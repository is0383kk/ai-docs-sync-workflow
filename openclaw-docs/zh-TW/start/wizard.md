---
read_when:
    - 執行或設定命令列介面新手引導
    - 設定新機器
sidebarTitle: 'Onboarding: CLI'
summary: 命令列介面新手引導：驗證推論，然後將其餘設定交給 OpenClaw
title: 新手設定（命令列介面）
x-i18n:
    generated_at: "2026-07-26T08:44:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 150adfac1424b42d66fa3035339082574cc631ce0dc3db09ad32376ef139bf1c
    source_path: start/wizard.md
    workflow: 16
---

```bash
openclaw onboard
```

命令列介面引導設定是在 macOS、Linux 和 Windows（原生或 WSL2）上建議使用的終端機設定方式。預設情況下，它會偵測機器上已有的 AI 存取方式、透過實際補全加以驗證，並啟動 OpenClaw 以設定工作區、閘道和選用功能。`openclaw setup` 會執行相同流程（[設定](/zh-TW/cli/setup)涵蓋僅設定組態的 `--baseline` 變體）。Windows 桌面使用者也可以從 [Windows Hub](/zh-TW/platforms/windows) 開始。

引導式設定會先建立推論能力。它會偵測可用的 AI 存取方式、要求成功完成一次實際補全，然後才啟動 [OpenClaw](/zh-TW/cli/openclaw) 以設定 OpenClaw 的其餘部分。選擇**暫時略過**會結束引導設定，而不啟動 OpenClaw。

傳統精靈仍可用於自訂供應商、遠端閘道設定、頻道配對、常駐程式控制、Skills 和匯入。請使用 `openclaw onboard --classic` 明確執行；引導式推論選擇器不會將流程轉交給它。推論通過後，OpenClaw 可以使用 `open channel wizard for
<channel>`，將需要機密資訊的頻道設定交給會遮罩輸入內容的終端機精靈。
若要變更模型供應商或其驗證方式，請結束 OpenClaw 並執行 `openclaw onboard`；OpenClaw 不會開啟引導式或傳統供應商流程。

<Info>
最快開始第一次聊天的方式：完成引導式設定、執行 `openclaw dashboard`，然後透過 Control UI 在瀏覽器中聊天。文件：[儀表板](/zh-TW/web/dashboard)。
</Info>

## 語系

精靈會將固定的引導設定文字本地化。它依序使用 `OPENCLAW_LOCALE`、`LC_ALL`、`LC_MESSAGES` 和 `LANG` 中第一個非空白值，若皆無則回退至英文。支援的語系：`en`、`zh-CN`、`zh-TW`。

```bash
OPENCLAW_LOCALE=zh-CN openclaw onboard
OPENCLAW_LOCALE=en openclaw onboard # 明確覆寫為英文
```

無論語系為何，產品名稱、命令、組態鍵、URL、供應商 ID、模型 ID，以及外掛／頻道標籤都會維持英文。

若之後要重新設定非推論相關設定：

```bash
openclaw configure
openclaw agents add <name>
```

<Note>
`--json` 並不表示非互動模式。若用於指令碼，請使用 `--non-interactive`（請參閱[命令列介面自動化](/zh-TW/start/wizard-cli-automation)）。
</Note>

<Tip>
傳統精靈包含網路搜尋步驟，可在其中選擇供應商：Brave、DuckDuckGo、Exa、Firecrawl、Gemini、Grok、Kimi、MiniMax Search、Ollama Web Search、Perplexity、SearXNG 或 Tavily。有些需要 API 金鑰，有些則不需要金鑰。之後可使用 `openclaw configure --section web` 進行設定。文件：[網路工具](/zh-TW/tools/web)。
</Tip>

## 引導式預設流程

直接執行 `openclaw onboard` 會遵循以下流程：

1. 接受安全性通知。
2. 偵測已設定的模型、API 金鑰環境變數、支援的本機 AI 命令列介面，以及閘道主機上可連線的 Ollama 或 LM Studio 伺服器中已安裝且支援工具的模型。此唯讀程序絕不會下載模型。若 Gemini CLI、Antigravity、Pi 和 OpenCode 的安裝無法作為引導式設定可重複使用的推論路徑，也會一併列出。Gemini 和 Antigravity 無法強制執行不使用工具的探測；Pi 和 OpenCode 則是完整的代理程式框架，而非設定用的推論路徑。
3. 使用實際補全測試第一個偵測到的候選項目。若失敗，顯示原因並繼續測試下一個可用候選項目。
4. 若所有偵測項目皆已用盡，請選擇 OpenAI、Anthropic、xAI (Grok)、Google 或 OpenRouter，或選擇**更多…**以查看其餘供應商。每個供應商的區域、方案，以及支援的瀏覽器、裝置、API 金鑰或權杖方式，會顯示在第二個選單中，並使用相同的實際補全進行測試。選擇**暫時略過**可直接結束，而不啟動 OpenClaw。
5. 只儲存已驗證的模型路徑，以及該路徑所需的任何認證資訊／外掛狀態。工作區和閘道設定不會變更。
6. 使用已驗證的模型啟動 OpenClaw，使其能夠設定工作區、閘道、頻道、代理程式、外掛和其餘選用設定。

在已完成設定的安裝環境中重新執行此命令時，會先測試目前的預設模型，因此引導式流程也可用於驗證與修復。檢查失敗時，絕不會自動取代已設定的模型；引導設定會停止並詢問如何繼續。若之後要新增非推論項目，請執行 `openclaw channels add` 或 `openclaw configure`；若要變更供應商或驗證路徑，請使用 `openclaw onboard`。

## 傳統精靈：快速開始與進階設定

執行 `openclaw onboard --classic` 以開啟完整精靈。開始時可選擇**快速開始**（預設值）或**進階設定**（完整控制）。傳入 `--flow quickstart` 或 `--flow advanced`（別名 `manual`）可選擇傳統流程並略過該提示。

<Tabs>
  <Tab title="快速開始（預設值）">
    - 本機閘道，繫結回送位址
    - 預設工作區（或現有工作區）
    - 閘道連接埠 **18789**
    - 閘道驗證**權杖**（自動產生，即使使用回送位址亦然）
    - 工具原則：新設定使用 `tools.profile: "coding"`（現有的明確設定檔會予以保留）
    - 私訊工作階段：引導設定會保留明確設定的 `session.dmScope`，否則維持未設定，讓 `"main"` 預設值將各頻道的所有私訊保留在代理程式持續累積的主要工作階段中——這是個人代理程式的預設值。對於共用或多使用者收件匣，請使用 `"per-channel-peer"`；當 `openclaw security audit` 偵測到多使用者私訊流量時，會建議採用隔離設定。詳細資訊：[命令列介面設定參考](/zh-TW/start/wizard-cli-reference#outputs-and-internals)
    - Tailscale 暴露設定**關閉**
    - Telegram 和 WhatsApp 私訊預設使用**允許清單**：Telegram 會要求輸入數字格式的 Telegram 使用者 ID，WhatsApp 則會要求輸入電話號碼

  </Tab>
  <Tab title="進階設定（完整控制）">
    - 顯示所有步驟：模式、工作區、閘道、頻道、常駐程式、Skills

  </Tab>
</Tabs>

遠端模式（`--mode remote`）一律使用進階流程；它只會設定這台機器以連線至其他位置的閘道，絕不會在遠端主機上安裝或變更任何內容。

## 傳統引導設定的設定項目

本機模式（預設）會依序進行以下步驟：

1. **模型／驗證** - 選擇供應商驗證流程（API 金鑰、OAuth 或供應商特定的手動驗證），包括自訂供應商（相容 OpenAI、相容 OpenAI Responses、相容 Anthropic，或未知自動偵測）。選擇預設模型。
   新的 OpenAI API 金鑰設定預設使用 `openai/gpt-5.6`（未限定的直接 API ID 會解析為 Sol）；新的 ChatGPT/Codex 設定預設使用 `openai/gpt-5.6-sol`。重新執行設定會保留現有的明確模型設定，包括 `openai/gpt-5.5`。如果帳戶未提供 GPT-5.6，請明確選擇 `openai/gpt-5.5`。
   安全性注意事項：如果此代理程式會執行工具或處理網路鉤子／鉤子內容，請優先使用最新世代中可用的最強模型，並維持嚴格的工具原則——較弱或較舊的級別更容易遭受提示注入攻擊。
   對於非互動式執行，`--secret-input-mode ref` 會儲存由環境變數支援的參照，而非純文字 API 金鑰值；所參照的環境變數必須已經設定，否則引導設定會立即失敗。互動式機密參照模式可指向環境變數或已設定的供應商參照（`file` 或 `exec`），並在儲存前執行快速的預先檢查。完成模型／驗證設定後，精靈會提供選用的即時補全測試；測試失敗時，可返回模型／驗證設定一次，或忽略失敗並繼續完成傳統精靈的其餘步驟。忽略失敗不會解鎖 OpenClaw；對話式設定仍需要通過推論檢查。
2. **工作區** - 代理程式檔案的目錄（預設為 `~/.openclaw/workspace`）。植入啟動檔案。
3. **閘道** - 連接埠、繫結位址、驗證模式、Tailscale 暴露設定。在互動式權杖模式中，選擇以純文字儲存權杖（預設），或選擇使用 SecretRef。非互動式 SecretRef 路徑：`--gateway-token-ref-env <ENV_VAR>`。
4. **頻道** - 內建與官方外掛聊天頻道，包括 Discord、Feishu、Google Chat、iMessage、Mattermost、Microsoft Teams、QQ Bot、Signal、Slack、Telegram、WhatsApp 等。
5. **常駐程式** - 安裝 LaunchAgent（macOS）、systemd 使用者單元（Linux/WSL2），或原生 Windows 排定工作，並以每位使用者的「啟動」資料夾作為回退方案。
   如果需要權杖驗證且 `gateway.auth.token` 由 SecretRef 管理，常駐程式安裝程序會驗證它，但不會將解析出的權杖持久儲存至監督程式的服務環境中繼資料；無法解析的 SecretRef 會阻止安裝並提供指引。如果 `gateway.auth.token` 和 `gateway.auth.password` 皆已設定，但 `gateway.auth.mode` 未設定，則在你明確設定模式前，安裝程序會遭到阻止。
6. **健康狀態檢查** - 啟動閘道並驗證其可連線性。
7. **Skills** - 安裝建議的 Skills 及其選用相依套件。

<Note>
重新執行引導設定**不會**清除任何內容，除非你明確選擇**重設**（或傳入 `--reset`）。命令列介面的 `--reset` 預設會移除組態、認證資訊和工作階段；若也要移除工作區，請使用 `--reset-scope full`。如果組態無效或包含舊版鍵，引導設定會要求你先執行 `openclaw doctor`。
</Note>

`--flow import` 會在傳統精靈中執行偵測到的移轉流程（例如 Hermes），而不是進行全新設定；請參閱[移轉](/zh-TW/cli/migrate)以及[安裝](/zh-TW/install/migrating-hermes)下的移轉指南。`openclaw onboard --modern` 是 [OpenClaw](/zh-TW/cli/openclaw) 的相容性別名。它使用與 `openclaw setup` 相同的推論閘門：通過驗證的推論會啟動助理，而互動式驗證失敗則會返回引導式推論設定。

## 新增其他代理程式

使用 `openclaw agents add <name>` 建立擁有自己工作區、工作階段和驗證設定檔的獨立代理程式。不含 `--workspace` 執行時，會啟動名稱、工作區、驗證、頻道和繫結的互動式流程——這並不是完整的 `openclaw onboard` 精靈。

它會設定：

- `agents.entries.*.name`
- `agents.entries.*.workspace`
- `agents.entries.*.agentDir`

注意事項：

- 預設工作區：`~/.openclaw/workspace-<agentId>`（若已設定 `agents.defaults.workspace`，則位於其下）。
- 新增 `bindings`，將傳入訊息路由至此代理程式（引導設定可代為完成）。
- 非互動式旗標：`--model`、`--agent-dir`、`--bind`、`--non-interactive`。

## 完整參考資料

如需詳細的逐步行為和組態輸出，請參閱[命令列介面設定參考](/zh-TW/start/wizard-cli-reference)。
如需非互動式範例，請參閱[命令列介面自動化](/zh-TW/start/wizard-cli-automation)。
如需完整旗標參考資料，請參閱 [`openclaw onboard`](/zh-TW/cli/onboard)。

## 相關文件

- 命令列介面命令參考：[`openclaw onboard`](/zh-TW/cli/onboard)
- 引導設定概覽：[引導設定概覽](/zh-TW/start/onboarding-overview)
- macOS 應用程式引導設定：[引導設定](/zh-TW/start/onboarding)
- 代理程式首次執行儀式：[代理程式啟動設定](/zh-TW/start/bootstrapping)
