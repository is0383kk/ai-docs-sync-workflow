---
read_when:
    - 你想要建立推論服務，然後使用 OpenClaw 完成設定
summary: '`openclaw onboard`（互動式初始設定）的命令列介面參考資料'
title: 入門設定
x-i18n:
    generated_at: "2026-07-26T07:46:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ec5cfc564aa14041d1aa67a978a4661e6105b7119a942940f71197c695e788b
    source_path: cli/onboard.md
    workflow: 16
---

# `openclaw onboard`

以先建立推論能力為核心的引導式設定：偵測現有的 AI 存取方式、
要求即時完成一次生成、僅保存可運作的路徑，然後啟動
OpenClaw 以設定其餘項目。`openclaw setup` 會在全新
系統或存在任何新手設定選項時進入此流程；已設定的系統會使用
不帶參數的 `openclaw setup` 進行系統代理程式聊天。`openclaw setup --baseline` 僅
寫入基準設定／工作區。

<CardGroup cols={2}>
  <Card title="命令列介面新手設定中心" href="/zh-TW/start/wizard" icon="rocket">
    互動式命令列介面流程的逐步指南。
  </Card>
  <Card title="新手設定概覽" href="/zh-TW/start/onboarding-overview" icon="map">
    OpenClaw 新手設定各部分如何協同運作。
  </Card>
  <Card title="命令列介面設定參考" href="/zh-TW/start/wizard-cli-reference" icon="book">
    輸出、內部機制及各步驟行為。
  </Card>
  <Card title="命令列介面自動化" href="/zh-TW/start/wizard-cli-automation" icon="terminal">
    非互動式旗標及指令碼化設定。
  </Card>
  <Card title="macOS App 新手設定" href="/zh-TW/start/onboarding" icon="apple">
    macOS 選單列 App 的新手設定流程。
  </Card>
</CardGroup>

## 範例

```bash
openclaw onboard
openclaw onboard --tui
openclaw onboard --classic
openclaw onboard --modern
openclaw onboard --flow quickstart
openclaw onboard --flow manual
openclaw onboard --flow import
openclaw onboard --import-from hermes --import-source ~/.hermes
openclaw onboard --skip-bootstrap
openclaw onboard recommendations --json
openclaw onboard recommendations acknowledge
openclaw onboard recommendations acknowledge --retry "<failed-id>"
openclaw onboard recommendations refresh
openclaw onboard --mode remote --remote-url wss://gateway-host:18789
```

`openclaw onboard recommendations` 會讀取新手設定期間
儲存的待處理 App 建議配對。加入 `--json` 可取得
首次執行啟動程序所使用的機器可讀清單。此命令不會重新掃描已安裝的 App，也不會呼叫
模型。其輸出僅包含已驗證的安裝 ID、來源和層級；並
刻意省略不受信任的市集說明、模型理由和本機 App
標籤。回覆建議提議後，此命令會傳回
空清單，而未來的新手設定執行將完全略過此步驟。
`openclaw onboard recommendations refresh` 會清除已儲存的提議，讓下次
新手設定執行重新掃描已安裝的 App 並建立新提議。

全新的工作區會將建議選擇延後至啟動對話中處理。
該對話處理完使用者的選擇後，
`openclaw onboard recommendations acknowledge` 會將已儲存的提議標記為已回覆。
確認操作具冪等性。如果所選的安裝失敗，請以 `--retry <id...>` 傳入每個失敗的
不透明 ID；成功和拒絕的配對會被消耗，
失敗的配對則會保持待處理，供稍後的新手設定執行使用。未知 ID
會導致失敗，且不會變更已儲存的提議。ClawHub Skill
安裝中斷後，只有在
`openclaw skills verify "@owner/slug"` 對相同的
含發布者限定建議 ID 執行成功，且其 JSON 輸出回報
`openclaw.resolution.source: "installed"` 時，現有目標才算安裝成功。僅驗證登錄檔
並不能證明本機已安裝。否則，請使用 `--retry` 讓該 ID 保持待處理，且
不要覆寫現有 Skill。

- `--classic`：開啟完整的逐步精靈。無法與
  `--non-interactive` 合併使用；自動化設定請省略 `--classic`。
- `--flow quickstart`：開啟僅提供少量提示的經典精靈，預設使用
  Token 驗證，且在沒有適用的已儲存或明確認證資訊時
  產生 Token。明確指定的本機閘道旗標（例如
  `--gateway-port`、`--gateway-bind`、`--gateway-auth` 和 `--tailscale`）
  會覆寫對應的已儲存或預設快速入門值；省略的
  選項會保留其目前值。
- `--flow manual`（別名 `advanced`）：開啟經典精靈，完整提示
  連接埠、繫結和驗證設定。
- `--flow import`：針對全新設定執行偵測到的遷移提供者（例如透過 `--import-from hermes` 使用 Hermes）。確認後，新手設定會在私有暫存目標下暫存設定、認證資訊、工作區檔案、記憶和 Skills；匯入的推論必須通過即時生成測試，之後才會提升工作區和代理程式狀態並提交設定。在提升之前失敗或取消，均不會變更正式目標。無法復原的外部啟用步驟（例如安裝 Codex 外掛）會在之後執行，並可從遷移報告中重試。如果已有任何設定、認證資訊、工作階段或工作區狀態，請先重設。使用 [`openclaw migrate`](/zh-TW/cli/migrate) 取得試執行計畫、覆寫模式、已驗證的備份、報告和精確對應。
- `--remote-url` 和 `--remote-token`：預先填入經典遠端閘道步驟，並覆寫此次執行所使用的已儲存遠端值。變更 URL 不會重複使用已儲存的認證資訊，除非你也傳入 Token。Token 在提示中會保持遮蔽，並遵循精靈現有的純文字或 SecretRef 儲存選擇。
- `--tailscale-reset-on-exit` 和 `--no-tailscale-reset-on-exit`：明確控制閘道結束時是否重設 Tailscale Serve 或 Funnel 設定。兩者皆省略時，非互動式重新執行期間會保留目前設定。
- `--modern` 是 OpenClaw 對話式設定
  助理的相容性別名。它使用與 `openclaw setup` 相同的即時推論閘門，且
  僅接受 `--workspace`、`--accept-risk`、
  `--non-interactive` 和 `--json`。其他設定旗標會遭到拒絕，而非
  被默默忽略。

## 引導式流程

不帶其他參數的 `openclaw onboard` 會啟動引導式流程。它會顯示安全性通知，
然後先詢問一個問題：**完整存取權**（建議使用——設定程序會自動尋找
AI App、金鑰和本機執行環境）或 **先詢問**（設定程序會在
查看系統前詢問一次，或讓你手動設定）。此
選擇會保存為 `wizard.accessMode`。允許探索時，新手設定會
偵測已可透過已設定模型、API 金鑰
環境變數和受支援本機命令列介面使用的 AI 存取方式，然後以實際生成測試建議的
候選項目。如果候選項目失敗，新手設定會靜默
嘗試下一個可用項目，並以一行摘要列出所有未回應的項目；
同時公布可運作的路徑，並提供按一次按鍵即可改為查看
其他所有選項的功能。

如果自動偵測已用盡所有選項，提供者選擇器會先顯示 OpenAI、
Anthropic、xAI (Grok)、Google 和 OpenRouter。選擇 **更多…** 可查看所有
其他受支援提供者，並依提供者分組；接著會在第二個選單顯示地區、方案和驗證方法。
受支援的瀏覽器或裝置登入方式，以及遮蔽的
API 金鑰或 Token 方法，會使用相同的即時生成流程。OpenClaw 僅會在
測試成功後保存已驗證的模型路徑及其認證資訊；
失敗的候選項目不會取代已設定模型，也不會儲存嘗試使用的
認證資訊。選擇 **暫時略過** 可在不啟動 OpenClaw 的情況下結束，
準備好後再重新執行 `openclaw onboard`。工作區和閘道設定在
OpenClaw 啟動前均不會變更。

在引導模式中，`--workspace <dir>` 會提供 OpenClaw 建議的工作區
及隔離的推論情境。你核准
OpenClaw 設定提案前，它不會被保存。經典和非互動式新手設定會透過
各自的正常設定流程保存工作區。若重新執行時已有代理程式
名冊，新手設定會保留已設定的群組工作區：經典
精靈會顯示兩個路徑，並要求明確確認後才會移動；
非互動式設定則會發出警告並保留目前值。

推論通過後，新手設定會檢查受支援本機 AI
工具中的記憶：Claude Code 自動記憶、Codex 彙整記憶和 Hermes 記憶
檔案。找到任何記憶時，會以單一頁面詢問是否將其複製到代理程式工作區的
`memory/imports/` 下，以供建立索引後回想。未經
確認不會匯入任何內容，先前已匯入的檔案會被略過，而且你隨時可以稍後從 Control UI 的
[記憶匯入頁面](/zh-TW/web/control-ui) 匯入；該頁面提供
相同的僅限記憶範圍。（完整執行 [`openclaw migrate`](/zh-TW/cli/migrate) 的
範圍更廣：也能匯入設定、Skills 和認證資訊。）經典
精靈會在準備好工作區後顯示相同頁面。

推論通過（且完成記憶匯入提議）後，引導式新手設定會
自動套用標準設定——工作區、閘道和工作階段，
也就是對話式 `openclaw setup` 聊天在收到 “yes” 時會套用的相同計畫——
接著會根據已安裝的 App 提供外掛和 Skill 建議；App 名稱
會透過你設定的模型和 ClawHub 搜尋進行比對，且可使用 [`wizard.appRecommendations`](/zh-TW/gateway/configuration-reference#wizard)
停用此步驟。
在 macOS、Linux 或 Windows 桌面工作階段中，接著會開啟已驗證身分的
Control UI 儀表板，並等待瀏覽器用戶端連線，最長 60 秒。
在無頭 Linux 或透過 SSH 執行時，則會顯著印出可直接複製貼上的
儀表板 URL；若為回送閘道，還會包含 SSH 連接埠轉送命令，
並等待最長五分鐘。成功連線後會在瀏覽器中繼續；
閘道無法連線或逾時時，則會退回與之前相同的終端替代入口。
傳入 `--tui` 可略過瀏覽器移交，並強制使用該終端替代入口。
如果套用設定失敗，新手設定會退回對話式 OpenClaw
聊天，以互動方式完成。頻道、代理程式、
外掛和其他選用功能仍由 OpenClaw 聊天處理：執行
`openclaw`，並使用 `open channel wizard for <channel>` 將頻道
認證資訊收集交由遮蔽輸入的終端精靈處理。若要變更模型
提供者或其驗證方式，請結束 OpenClaw 並執行 `openclaw onboard`；
OpenClaw 不會開啟引導式或經典提供者流程。

在已設定的安裝環境中，再次執行 `openclaw onboard` 會先驗證目前的
預設模型，因此相同流程可作為驗證和修復程序——
它不會重新套用設定、重新安裝或重新啟動閘道服務。
如果該檢查失敗，絕不會自動取代已設定模型——
新手設定會停止並詢問如何繼續。此檢查在你的
工作區之外執行，因此由工作區外掛提供的模型可能會在此處失敗，但仍可
在代理程式中正常運作。
使用 `openclaw onboard --classic` 可進行提供者專用驗證、頻道、Skills、
遠端閘道設定、匯入或完整閘道控制。若要進行非推論的對話式
設定和修復，請執行 `openclaw setup`；`openclaw onboard
--modern` 是透過相同推論閘門運作的相容性別名。經典
精靈可選擇以即時生成驗證預設模型，但
OpenClaw 在自身的即時推論檢查通過前不會啟動。

在互動式終端中，不帶子命令的 `openclaw` 會依設定
狀態決定路由：

- 如果找不到有效設定檔，或其中沒有自行撰寫的設定（空白或
  僅含中繼資料），則會啟動引導式新手設定。
- 如果設定檔存在但未通過驗證，則會依照 `openclaw doctor` 指引啟動經典
  新手設定路徑。OpenClaw 需要可運作的
  推論能力，因此不會用來修復此推論前狀態。
- 如果設定檔有效，則會開啟一般代理程式終端介面。可連線且
  已設定代理程式和模型的閘道會直接進入該介面，不會進行
  新手設定或啟動 OpenClaw。在已設定的安裝環境中，可透過終端介面內的
  `/openclaw` 或 `openclaw setup` 進入 OpenClaw。

純文字 `ws://` 可用於回送位址、私人 IP 常值、`.local` 和 Tailnet `*.ts.net` 閘道 URL。對於其他受信任的私人 DNS 名稱，請在新手設定程序環境中設定 `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1`。

## 重設

```bash
openclaw onboard --reset
openclaw onboard --reset --reset-scope full
```

`--reset` 會在執行設定前清除狀態。`--reset-scope` 控制清除程度：`config`（僅設定）、`config+creds+sessions`（傳入 `--reset` 但未指定範圍時的預設值），或 `full`（也會重設工作區）。只有使用 `--reset-scope full` 時才會重設工作區。

## 語系

互動式初始設定會使用命令列介面精靈的語系來顯示固定設定文案。它會依下列順序採用第一個非空白值：

1. `OPENCLAW_LOCALE`
2. `LC_ALL`
3. `LC_MESSAGES`
4. `LANG`
5. 英文備援

支援的精靈語系為 `en`、`zh-CN` 和 `zh-TW`。語系值可使用底線或 POSIX 後綴格式，例如 `zh_CN.UTF-8`。產品名稱、命令名稱、設定鍵、URL、提供者 ID、模型 ID，以及外掛／頻道標籤會維持原文。

```bash
OPENCLAW_LOCALE=zh-CN openclaw onboard
OPENCLAW_LOCALE=en openclaw onboard # 明確覆寫為英文
```

## 非互動式設定

`--non-interactive` 需要 `--accept-risk`（確認代理程式功能強大，且完整系統存取權限具有風險）。`--mode` 預設為 `local`。

```bash
openclaw onboard --non-interactive \
  --auth-choice custom-api-key \
  --custom-base-url "https://llm.example.com/v1" \
  --custom-model-id "foo-large" \
  --custom-api-key "$CUSTOM_API_KEY" \
  --secret-input-mode plaintext \
  --custom-compatibility openai \
  --custom-image-input
```

`--custom-api-key` 為選用；若省略，初始設定會檢查環境中的 `CUSTOM_API_KEY`。OpenClaw 會自動將常見的視覺模型 ID（GPT-4o/4.1/5.x、Claude 3/4、Gemini、Qwen-VL、LLaVA、Pixtral 及類似模型）標記為具備影像處理能力。對於未知的自訂視覺模型 ID，請傳入 `--custom-image-input`；若要強制使用純文字中繼資料，則傳入 `--custom-text-input`。若 OpenAI 相容端點支援 `/v1/responses` 但不支援 `/v1/chat/completions`，請使用 `--custom-compatibility openai-responses`；有效值為 `openai`（預設）、`openai-responses`、`anthropic`。

LM Studio 另有提供者專用的金鑰旗標：

```bash
openclaw onboard --non-interactive \
  --auth-choice lmstudio \
  --custom-base-url "http://localhost:1234/v1" \
  --custom-model-id "qwen/qwen3.5-9b" \
  --lmstudio-api-key "$LM_API_TOKEN" \
  --accept-risk
```

非互動式 Ollama：

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

`--custom-base-url` 預設為 `http://127.0.0.1:11434`。`--custom-model-id` 為選用；若省略，初始設定會使用 Ollama 建議的預設值。`kimi-k2.5:cloud` 等雲端模型 ID 也可在此使用。

將提供者金鑰儲存為參照，而非純文字：

```bash
openclaw onboard --non-interactive \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --accept-risk
```

使用 `--secret-input-mode ref` 時，初始設定會寫入由環境變數支援的參照，而非純文字金鑰值：對於由驗證設定檔支援的提供者，這會寫入 `keyRef: { source: "env", provider: "default", id: <envVar> }`；對於自訂提供者，則會以相同方式寫入 `models.providers.<id>.apiKey`（例如 `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`）。契約：請在初始設定程序的環境中設定提供者環境變數（例如 `OPENAI_API_KEY`），且除非該環境變數已設定，否則不要同時傳入行內金鑰旗標；若旗標有值但缺少相符的環境變數，程序會立即失敗並提供指引。

### 閘道驗證（非互動式）

- `--gateway-auth token --gateway-token <token>` 會儲存純文字權杖。`token` 是預設驗證模式。
- `--gateway-auth token --gateway-token-ref-env <name>` 會將 `gateway.auth.token` 儲存為環境變數 SecretRef。初始設定程序環境中必須有該名稱且非空白的環境變數。
- `--gateway-token` 與 `--gateway-token-ref-env` 互斥。
- 使用 `--install-daemon` 時：由 SecretRef 管理的 `gateway.auth.token` 會經過驗證，但不會以解析後的純文字形式持久保存於監督程式服務的環境中繼資料中；若參照無法解析，安裝會採取封閉式失敗並提供修復指引。若同時設定了 `gateway.auth.token` 和 `gateway.auth.password`，且未設定 `gateway.auth.mode`，安裝會封鎖，直到明確設定模式為止。
- 本機初始設定會將 `gateway.mode="local"` 寫入設定。若後續設定檔缺少 `gateway.mode`，表示設定已損壞或手動編輯未完成，而非有效的本機模式捷徑。
- 本機初始設定會安裝所選設定路徑需要的可下載外掛（例如這些驗證選項所需的 Codex 或 Copilot 執行階段外掛）。遠端初始設定只會寫入遠端閘道的連線資訊，絕不會安裝本機外掛套件。
- `--allow-unconfigured` 是獨立的 `openclaw gateway run` 緊急出口；它不允許初始設定略過 `gateway.mode`。

```bash
export OPENAI_API_KEY="your-provider-key"
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN \
  --accept-risk
```

### 本機閘道健康狀態

- 除非傳入 `--skip-health`，否則初始設定會等待本機閘道可連線後，才會成功結束。
- `--install-daemon` 會先啟動受管理的閘道安裝路徑。若未使用，本機閘道必須已在執行（例如 `openclaw gateway run`）。
- 若自動化流程只需要寫入設定、工作區及啟動資料，`--skip-health` 可略過等待。
- `--skip-bootstrap` 會設定 `agents.defaults.skipBootstrap: true`，並略過建立 `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md` 和 `BOOTSTRAP.md`。
- 在原生 Windows 上，`--install-daemon` 會先嘗試使用排程工作；若建立工作遭拒，則改用每位使用者的「啟動」資料夾登入項目。

### 互動式參照模式

- 出現提示時選擇 **使用祕密參照**，接著選擇 **環境變數** 或已設定的祕密提供者（`file` 或 `exec`）。
- 初始設定會在儲存參照前執行快速預檢驗證，若失敗則可重試。

### Z.AI 端點選項

<Note>
`--auth-choice zai-api-key` 會自動偵測最適合你的金鑰的 Z.AI 端點與模型：Coding Plan 端點會優先使用 `zai/glm-5.2`（若無法使用則退回 `glm-5.1`）；一般 API 端點預設為 `zai/glm-5.1`。若要強制使用 Coding Plan 端點，請直接選擇 `zai-coding-global` 或 `zai-coding-cn`。
</Note>

```bash
# 無提示選取端點
openclaw onboard --non-interactive \
  --auth-choice zai-coding-global \
  --zai-api-key "$ZAI_API_KEY"

# 其他 Z.AI 端點選項：zai-coding-cn、zai-global、zai-cn
```

Mistral：

```bash
openclaw onboard --non-interactive \
  --auth-choice mistral-api-key \
  --mistral-api-key "$MISTRAL_API_KEY"
```

## 其他非互動式旗標

權杖式模型驗證（搭配 `--auth-choice token` 使用）：

| 旗標                            | 說明                                                                                                                 |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `--token-provider <id>`         | 核發權杖的權杖提供者 ID                                                                                         |
| `--token <token>`               | 用於模型驗證的權杖值                                                                                        |
| `--token-profile-id <id>`       | 驗證設定檔 ID（預設為 `<provider>:manual`；部分由提供者擁有的流程會使用自己的預設值，例如 `anthropic:default`） |
| `--token-expires-in <duration>` | 選用的權杖到期時間長度（例如 `365d`、`12h`）                                                                         |

Cloudflare AI Gateway：`--cloudflare-ai-gateway-account-id <id>`、`--cloudflare-ai-gateway-gateway-id <id>`。

常駐程式安裝控制：`--no-install-daemon` / `--skip-daemon`（別名；略過閘道服務安裝）、`--daemon-runtime <node>`。

Skills：`--node-manager <npm|pnpm|bun>`（預設為 `npm`）、`--skip-skills`。

使用者介面與鉤子設定：`--skip-ui`（略過 Control UI／終端介面提示）、`--skip-hooks`（略過網路鉤子／鉤子設定）、`--skip-channels`、`--skip-search`。

輸出：`--suppress-gateway-token-output` 會抑制含權杖的閘道／使用者介面輸出（權杖提示、內嵌權杖的自動登入 URL，以及自動啟動 Control UI），適合用於共用終端與 CI。

<Note>
`--json` 在引導式或傳統初始設定中不代表非互動式模式。
使用 `--modern` 時，JSON 是一次性的 OpenClaw 概覽，並會在取得該單一結果後結束。
其他指令碼請使用 `--non-interactive`。
</Note>

## 提供者預先篩選

當驗證選項暗示偏好的提供者時，初始設定會將預設模型與允許清單選擇器預先篩選為該提供者的模型。篩選器也會比對由同一外掛擁有的其他提供者，因此可涵蓋 `volcengine`/`volcengine-plan` 和 `byteplus`/`byteplus-plan` 等 Coding Plan 變體。若偏好提供者篩選後沒有任何已載入模型，初始設定會改用未篩選的目錄，而不會讓選擇器空白。

## 網頁搜尋後續提示

部分網頁搜尋提供者會在初始設定期間觸發提供者專用的後續提示：

- **Grok** 可選擇提供使用相同 xAI 驗證的 `x_search` 設定，以及 `x_search` 模型選項。
- **Kimi** 可詢問 Moonshot API 區域（`api.moonshot.ai` 或 `api.moonshot.cn`）及預設的 Kimi 網頁搜尋模型。

## 其他行為

- 本機初始設定的私訊範圍行為：[命令列介面設定參考](/zh-TW/start/wizard-cli-reference#outputs-and-internals)。
- 最快開始第一次聊天：`openclaw dashboard`（Control UI，無需設定頻道）。
- 自訂提供者：連接任何與 OpenAI 或 Anthropic 相容的端點，包括未列出的託管提供者。使用 **未知** 相容性，透過即時探測自動偵測。
- 若偵測到 Hermes 狀態，初始設定會提供遷移流程（請參閱上方的 `--flow import`）。

## 常用後續命令

稍後若要進行不涉及推論的特定變更，請使用 `openclaw configure`；若只要設定頻道，請使用 `openclaw
channels add`。若要變更模型提供者或驗證路由，
請改為執行 `openclaw onboard`。

```bash
openclaw channels add
openclaw configure
openclaw agents add <name>
```
