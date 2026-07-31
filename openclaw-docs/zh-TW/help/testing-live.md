---
read_when:
    - 執行即時模型矩陣／命令列介面後端／ACP／媒體供應商冒煙測試
    - 偵錯即時測試認證資訊解析
    - 新增供應商專屬的即時測試
sidebarTitle: Live tests
summary: 即時（會存取網路）測試：模型矩陣、命令列介面後端、ACP、媒體提供者、認證資訊
title: 測試：即時測試套件
x-i18n:
    generated_at: "2026-07-26T07:56:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea8279e734e3aa09dd1fa184806c925e0404edfa9acf0f682f73a4955ed90b8b
    source_path: help/testing-live.md
    workflow: 16
---

若要瞭解快速入門、QA 執行器、單元／整合測試套件與 Docker 流程，請參閱
[測試](/zh-TW/help/testing)。本頁涵蓋**即時**（會連線至網路）測試：
模型矩陣、命令列介面後端、ACP、媒體提供者與認證資訊處理。

## 即時測試與你的實際閘道

即時測試套件與臨時冒煙測試絕不能干擾已在
處理實際流量的閘道（無論是你的還是其他操作者的）：

- 使用你自己的閘道：使用行程內閘道（下方第 2 層），或以隔離的狀態目錄（`OPENCLAW_STATE_DIR=<scratch>`）
  與可用連接埠啟動開發執行個體。當實際閘道正在使用預設閘道連接埠（18789）時，
  請勿繫結該連接埠。
- 請勿對非本工作階段啟動的服務執行 `openclaw gateway stop`/`restart`（或 `launchctl`/`systemctl`/tmux
  的對應操作）——那是操作者的即時執行個體。請先取得明確核准。
- 需要擬真的資料嗎？將即時狀態／資料庫複製到開發狀態目錄，並針對該副本進行測試。
  若要就地遷移即時閘道的狀態，也需要明確核准。

## 即時測試：本機冒煙測試命令

執行臨時即時檢查前，請先在處理程序環境中匯出所需的提供者金鑰。

安全的媒體冒煙測試：

```bash
pnpm openclaw infer tts convert --local --json \
  --text "OpenClaw 即時冒煙測試。" \
  --output /tmp/openclaw-live-smoke.mp3
```

安全的語音通話就緒狀態冒煙測試：

```bash
pnpm openclaw voicecall setup --json
pnpm openclaw voicecall smoke --to "+15555550123"
```

除非同時提供 `--yes`，否則 `voicecall smoke` 是試執行；只有在你確實要撥打真實電話時，
才使用 `--yes`。對於 Twilio、Telnyx 與 Plivo，成功的就緒狀態檢查需要公開的網路鉤子 URL——
本機／私人迴路 URL 會遭拒絕，因為這些提供者無法連線至該類 URL。

## 即時測試：Android 節點能力全面測試

- 測試：`src/gateway/android-node.capabilities.live.test.ts`
- 指令碼：`pnpm android:test:integration`
- 目標：叫用已連線 Android 節點**目前公告的每一個命令**，並驗證命令契約行為。
- 範圍：
  - 需事先完成／手動設定（此測試套件不會安裝、執行或配對應用程式）。
  - 針對所選 Android 節點，逐一驗證命令的閘道 `node.invoke`。
- 必要的事前設定：
  - Android 應用程式已連線並配對至閘道。
  - 應用程式保持在前景。
  - 已針對預期通過的能力授予權限／擷取同意。
- 選用的目標覆寫：
  - `OPENCLAW_ANDROID_NODE_ID` 或 `OPENCLAW_ANDROID_NODE_NAME`。
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`。
- 完整 Android 設定詳情：[Android 應用程式](/zh-TW/platforms/android)

## 即時測試：模型冒煙測試（設定檔金鑰）

即時模型測試分為兩層，以便隔離失敗原因：

- 「直接模型」會確認提供者／模型是否能使用指定金鑰正常回應。
- 「閘道冒煙測試」會確認該模型的完整閘道＋代理程式流水線是否正常運作（工作階段、歷程記錄、工具、沙箱原則等）。

下方精選模型清單位於 `src/agents/live-model-filter.ts`，
並會隨時間變更；請以其中的陣列為唯一真實來源，而非本頁。

MiniMax M3 使用 `minimax/MiniMax-M3` 作為預設提供者／模型參照。

### 第 1 層：直接模型補全（無閘道）

- 測試：`src/agents/models.profiles.live.test.ts`
- 目標：
  - 列舉探索到的模型
  - 使用 `getApiKeyForModel` 選取你擁有認證資訊的模型
  - 為每個模型執行一次小型補全（並視需要執行針對性的迴歸測試）
- 啟用方式：
  - `pnpm test:live`（若直接叫用 Vitest，則為 `OPENCLAW_LIVE_TEST=1`）
  - 設定 `OPENCLAW_LIVE_MODELS=modern`、`small` 或 `all`（`modern` 的別名）才能實際執行此測試套件；否則會略過，因此單獨使用 `pnpm test:live` 時仍會專注於閘道冒煙測試。
- 選取模型的方式：
  - `OPENCLAW_LIVE_MODELS=modern` 會執行精選的高訊號優先清單（請參閱[即時測試：模型矩陣](#live-model-matrix-what-we-cover)）
  - `OPENCLAW_LIVE_MODELS=small` 會執行精選的小型模型優先清單
  - `OPENCLAW_LIVE_MODELS=all` 是 `modern` 的別名
  - 或使用 `OPENCLAW_LIVE_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,..."`（逗號分隔的允許清單）
  - 本機 Ollama 小型模型執行預設使用 `http://127.0.0.1:11434`；只有 LAN、自訂或 Ollama Cloud 端點才設定 `OPENCLAW_LIVE_OLLAMA_BASE_URL`。
  - 現代／全部與小型全面測試預設以各自精選清單的長度為上限；若要對所選設定檔進行完整全面測試，請設定 `OPENCLAW_LIVE_MAX_MODELS=0`，若要使用較小上限，則設定正數。
  - 完整全面測試使用 `OPENCLAW_LIVE_TEST_TIMEOUT_MS` 作為整個直接模型測試的逾時時間。預設值：60 分鐘。
  - 直接模型探查預設使用 20 路平行處理；設定 `OPENCLAW_LIVE_MODEL_CONCURRENCY` 可加以覆寫。
- 選取提供者的方式：
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"`（逗號分隔的允許清單）
- 金鑰來源：
  - 預設：設定檔儲存區與環境後援
  - 設定 `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`，強制**僅使用設定檔儲存區**
- 此機制存在的原因：
  - 將「提供者 API 故障／金鑰無效」與「閘道代理程式流水線故障」區分開來
  - 包含小型且隔離的迴歸測試（例如：OpenAI Responses/Codex Responses 推理重播＋工具呼叫流程）

### 第 2 層：閘道＋開發代理程式冒煙測試（「@openclaw」實際執行的內容）

- 測試：`src/gateway/gateway-models.profiles.live.test.ts`
- 目標：
  - 啟動行程內閘道
  - 建立／修補 `agent:dev:*` 工作階段（每次執行覆寫模型）
  - 逐一處理具備金鑰的模型，並驗證：
    - 「有意義的」回應（不使用工具）
    - 實際工具叫用能夠運作（讀取探查）
    - 選用的額外工具探查（執行＋讀取探查）
    - OpenAI 迴歸路徑（僅工具呼叫 → 後續回應）持續正常運作
- 探查詳情（以便快速說明失敗原因）：
  - `read` 探查：測試會在工作區中寫入一次性值檔案，並要求代理程式使用 `read` 讀取該檔案，再回傳一次性值。
  - `exec+read` 探查：測試會要求代理程式使用 `exec` 將一次性值寫入暫存檔案，再使用 `read` 將其讀回。
  - 影像探查：測試會附加產生的 PNG（貓＋隨機代碼），並預期模型傳回 `cat <CODE>`。
  - 實作參照：`src/gateway/gateway-models.profiles.live.test.ts` 與 `test/helpers/live-image-probe.ts`。
- 啟用方式：
  - `pnpm test:live`（若直接叫用 Vitest，則為 `OPENCLAW_LIVE_TEST=1`）
- 選取模型的方式：
  - 預設：精選的高訊號（`modern`）優先清單
  - `OPENCLAW_LIVE_GATEWAY_MODELS=small` 會透過完整閘道＋代理程式流水線執行精選的小型模型清單
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` 是 `modern` 的別名
  - 或設定 `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"`（或逗號分隔清單）以縮小範圍
  - 現代／全部與小型閘道全面測試預設以各自精選清單的長度為上限；若要對所選項目進行完整全面測試，請設定 `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0`，若要使用較小上限，則設定正數。
- 選取提供者的方式（避免「OpenRouter 包辦一切」）：
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"`（逗號分隔的允許清單）
- 此即時測試一律啟用工具＋影像探查：
  - `read` 探查＋`exec+read` 探查（工具壓力測試）
  - 當模型公告支援影像輸入時，會執行影像探查
  - 流程（概略）：
    - 測試產生含有「CAT」＋隨機代碼的小型 PNG（`test/helpers/live-image-probe.ts`）
    - 透過 `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]` 傳送
    - 閘道將附件剖析為 `images[]`（`src/gateway/server-methods/agent.ts`＋`src/gateway/chat-attachments.ts`）
    - 嵌入式代理程式將多模態使用者訊息轉送給模型
    - 驗證：回覆包含 `cat`＋該代碼（OCR 容錯：允許輕微錯誤）

<Tip>
若要查看你的機器可以測試哪些項目（以及確切的 `provider/model` ID），請執行：

```bash
openclaw models list
openclaw models list --json
```

</Tip>

## 即時測試：命令列介面後端冒煙測試（Claude、Gemini 或其他本機命令列介面）

- 測試：`src/gateway/gateway-cli-backend.live.test.ts`
- 目標：使用本機命令列介面後端驗證閘道＋代理程式流水線，且不變更你的預設設定。
- 後端專屬的冒煙測試預設值位於所屬外掛的 `cli-backend.ts` 定義中。
- 啟用：
  - `pnpm test:live`（若直接叫用 Vitest，則為 `OPENCLAW_LIVE_TEST=1`）
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- 預設值：
  - 預設提供者／模型：`claude-cli/claude-sonnet-4-6`
  - 命令／引數／影像行為來自所屬命令列介面後端外掛的中繼資料。
- 覆寫（選用）：
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/claude"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["-p","--output-format","json"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` 可傳送真實影像附件（路徑會注入提示詞）。在 Docker 配方中預設關閉。
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` 可將影像檔案路徑作為命令列介面引數傳遞，而非注入提示詞。
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"`（或 `"list"`）可在設定 `IMAGE_ARG` 時控制影像引數的傳遞方式。
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` 可傳送第二輪訊息並驗證續接流程。
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=1` 可選擇啟用 Claude Sonnet → Opus 同一工作階段的連續性探查，但所選模型必須支援切換目標。預設關閉，Docker 配方亦同。
  - `OPENCLAW_LIVE_CLI_BACKEND_MCP_PROBE=1` 可選擇啟用 MCP／工具迴路探查。在 Docker 配方中預設關閉。

範例：

```bash
  OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

低成本的 Gemini MCP 設定冒煙測試：

```bash
OPENCLAW_LIVE_TEST=1 \
  pnpm test:live src/agents/cli-runner/bundle-mcp.gemini.live.test.ts
```

此測試不會要求 Gemini 產生回應。它會寫入 OpenClaw 提供給 Gemini 的相同系統
設定，接著執行 `gemini --debug mcp list`，以證明已儲存的 `transport: "streamable-http"`
伺服器會正規化為 Gemini 的 HTTP MCP 形式，且能連線至本機可串流 HTTP MCP 伺服器。

Docker 配方：

```bash
pnpm test:docker:live-cli-backend
```

單一提供者 Docker 配方：

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:gemini
```

注意事項：

- Docker 執行器位於 `scripts/test-live-cli-backend-docker.sh`。
- 它會以非 root 的 `node` 使用者身分，在儲存庫 Docker 映像中執行即時命令列介面後端冒煙測試。
- 它會從所屬外掛解析命令列介面冒煙測試中繼資料，接著將相符的 Linux 命令列介面套件（`@anthropic-ai/claude-code` 或 `@google/gemini-cli`）安裝至 `OPENCLAW_DOCKER_CLI_TOOLS_DIR` 的可寫入快取前置路徑（預設：`~/.cache/openclaw/docker-cli-tools`）。
- `codex-cli` 已不再是隨附的命令列介面後端；請改用 `openai/*` 搭配 Codex app-server 執行階段（請參閱[即時：Codex app-server 測試框架冒煙測試](#live-codex-app-server-harness-smoke)）。
- `pnpm test:docker:live-cli-backend:claude-subscription` 需要透過 `~/.claude/.credentials.json` 搭配 `claudeAiOauth.subscriptionType`，或來自 `claude setup-token` 的 `CLAUDE_CODE_OAUTH_TOKEN`，使用可攜式 Claude Code 訂閱 OAuth。它會先在 Docker 中驗證直接 `claude -p`，接著在不保留 Anthropic API 金鑰環境變數的情況下，執行兩輪閘道命令列介面後端互動。此訂閱測試線預設停用 Claude MCP／工具與影像探測，因為這些探測會消耗已登入訂閱的用量限制，而且 Anthropic 可能在 OpenClaw 未發布新版本的情況下，變更 Claude Agent SDK／`claude -p` 的計費與速率限制行為。
- Claude 與 Gemini 可透過上述旗標支援相同的探測集（文字互動、影像分類、MCP `cron` 工具呼叫、模型切換連續性），但這些探測預設皆不執行，請依需要透過各旗標選擇啟用。

## 即時：APNs HTTP/2 Proxy 連線能力

- 測試：`src/infra/push-apns-http2.live.test.ts`
- 目標：透過本機 HTTP CONNECT Proxy 建立通往 Apple 沙盒 APNs 端點的通道、傳送 APNs HTTP/2 驗證要求，並確認 Apple 的實際 `403 InvalidProviderToken` 回應會經由 Proxy 路徑傳回。
- 啟用：
  - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_APNS_REACHABILITY=1 pnpm test:live src/infra/push-apns-http2.live.test.ts`
- 選用逾時：
  - `OPENCLAW_LIVE_APNS_TIMEOUT_MS=30000`

## 即時：ACP 繫結冒煙測試（`/acp spawn ... --bind here`）

- 測試：`src/gateway/gateway-acp-bind.live.test.ts`
- 目標：使用即時 ACP 代理程式驗證實際的 ACP 對話繫結流程：
  - 傳送 `/acp spawn <agent> --bind here`
  - 就地繫結合成的訊息頻道對話
  - 在同一對話中傳送一般的後續訊息
  - 驗證後續訊息已進入繫結的 ACP 工作階段轉錄記錄
- 啟用：
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- 預設值：
  - Docker 中的 ACP 代理程式：`claude,codex,gemini`
  - 用於直接 `pnpm test:live ...` 的 ACP 代理程式：`claude`
  - 合成頻道：Slack 私訊式對話內容
  - ACP 後端：`acpx`
- 覆寫：
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=droid`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=opencode`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
  - `OPENCLAW_LIVE_ACP_BIND_CODEX_MODEL=gpt-5.6-luna`
  - `OPENCLAW_LIVE_ACP_BIND_OPENCODE_MODEL=opencode/kimi-k2.6`
  - `OPENCLAW_LIVE_ACP_BIND_IMAGE_PROBE=1`（或 `on`/`true`/`yes`）會強制開啟影像探測；任何其他值都會強制關閉。除 `opencode` 外，所有代理程式預設都會執行。
  - `OPENCLAW_LIVE_ACP_BIND_REQUIRE_CRON=1`
  - `OPENCLAW_LIVE_ACP_BIND_PARENT_MODEL=openai/gpt-5.6-luna`
- 附註：
  - 此測試線會使用閘道 `chat.send` 介面及僅限管理員使用的合成來源路由欄位，讓測試無須假裝向外部遞送，即可附加訊息頻道內容。
  - 未設定 `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` 時，測試會針對所選的 ACP 測試框架代理程式，使用內嵌 `acpx` 外掛的內建代理程式登錄。
  - 繫結工作階段的排程 MCP 建立作業預設採盡力而為，因為外部 ACP 測試框架可能會在繫結／影像驗證通過後取消 MCP 呼叫；設定 `OPENCLAW_LIVE_ACP_BIND_REQUIRE_CRON=1` 可讓該繫結後的排程探測採嚴格模式。

範例：

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Docker 操作方式：

```bash
pnpm test:docker:live-acp-bind
```

單一代理程式 Docker 操作方式：

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:droid
pnpm test:docker:live-acp-bind:gemini
pnpm test:docker:live-acp-bind:opencode
```

Docker 附註：

- Docker 執行器位於 `scripts/test-live-acp-bind-docker.sh`。
- 預設會依序對彙總的即時命令列介面代理程式執行 ACP 繫結冒煙測試：`claude`、`codex`，接著是 `gemini`。
- 使用 `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=codex`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=droid`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` 或 `OPENCLAW_LIVE_ACP_BIND_AGENTS=opencode` 可縮小矩陣範圍。
- 它會將相符的命令列介面驗證資料暫存至容器中，接著在缺少時安裝要求的即時命令列介面（`@anthropic-ai/claude-code`、`@openai/codex`、透過 `https://app.factory.ai/cli` 的 Factory Droid、`@google/gemini-cli` 或 `opencode-ai`）。ACP 後端本身是官方 `acpx` 外掛內嵌的 `acpx/runtime` 套件。
- Droid Docker 變體會暫存 `~/.factory` 以供設定使用、轉送 `FACTORY_API_KEY`，並要求提供該 API 金鑰，因為本機 Factory OAuth／鑰匙圈驗證無法移植至容器中。它使用 ACPX 內建的 `droid exec --output-format acp` 登錄項目。
- OpenCode Docker 變體是嚴格的單一代理程式迴歸測試線。它會從 `OPENCLAW_LIVE_ACP_BIND_OPENCODE_MODEL`（預設為 `opencode/kimi-k2.6`）寫入暫時的 `OPENCODE_CONFIG_CONTENT` 預設模型。
- 直接呼叫 `acpx` 命令列介面僅是用於比較閘道外部行為的手動／因應措施路徑。Docker ACP 繫結冒煙測試會測試 OpenClaw 內嵌的 `acpx` 執行階段後端。

## 即時：Codex app-server 測試框架冒煙測試

- 目標：透過一般閘道
  `agent` 方法驗證外掛所屬的 Codex 測試框架：
  - 載入隨附的 `codex` 外掛
  - 透過 `/model <ref> --runtime codex` 選取 OpenAI 模型
  - 以要求的思考層級傳送第一輪閘道代理程式互動
  - 向同一個 OpenClaw 工作階段傳送第二輪互動，並驗證 app-server
    執行緒可繼續執行
  - 透過相同的閘道命令
    路徑執行 `/codex status` 與 `/codex models`
  - 選擇性執行兩項經 Guardian 審查且提高權限的 Shell 探測：一項應獲核准的無害
    命令，以及一項應遭拒絕的假機密上傳，讓代理程式
    回頭詢問
- 測試：`src/gateway/gateway-codex-harness.live.test.ts`
- 啟用：`OPENCLAW_LIVE_CODEX_HARNESS=1`
- 測試框架基準模型：`openai/gpt-5.6-luna`
- 全新 OpenAI API 金鑰選取預設值：`openai/gpt-5.6`
- 預設思考：`low`
- 模型覆寫：`OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/<model>`
- 思考覆寫：`OPENCLAW_LIVE_CODEX_HARNESS_THINKING=<level>`
- 非預設模型投入程度斷言：
  `OPENCLAW_LIVE_CODEX_HARNESS_EXPECTED_EFFORT=<level>`
- 矩陣覆寫：`OPENCLAW_LIVE_CODEX_HARNESS_TARGETS=<model>=<thinking>,...`
- 驗證模式：`OPENCLAW_LIVE_CODEX_HARNESS_AUTH=codex-auth`（預設）使用
  複製的 Codex 登入資料；`api-key` 透過 Codex app-server 使用 `OPENAI_API_KEY`。
- 選用影像探測：`OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- 選用 MCP／工具探測：`OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- 選用 Guardian 探測：`OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=1`
- 選用續接壓力測試：`OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1` 會加入
  四輪歷史互動，接著關閉並重新啟動閘道與 Codex app-server
  三次，同時要求維持相同的原生執行緒 ID 與對話
  歷史記錄。使用
  `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS_HISTORY_TURNS`（1-20）與
  `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS_RESTARTS`（1-10）可覆寫有界次數。
- 選用扇出壓力測試：設定 `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1`
  與 `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_COUNT`（1-12）。測試框架會同時啟動
  每個子項目、等待每個終止執行完成，並驗證各個
  不重複的子項目回覆與原生執行緒身分。
- 選用壓縮壓力測試：`OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS=1`
  會產生有界的原生工具輸出、要求發生自動壓縮事件、
  驗證持久化的壓縮次數與隱藏標記回想能力、重新啟動
  閘道與實體 Codex app-server，接著重複輸出與
  壓縮階段。使用
  `OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS_TURNS`（1-8）與
  `OPENCLAW_LIVE_CODEX_HARNESS_LARGE_OUTPUT_BYTES`（100000-800000）調整有界工作量。
- 完整直接 API 上下文：`OPENCLAW_LIVE_CODEX_HARNESS_FULL_CONTEXT=1` 會套用
  `922000` 上下文與 `700000` 總壓縮限制、傳送密集且有界的
  使用者互動、每個階段執行兩個明確的原生壓縮檢查點，並在
  每個檢查點後繼續進行後續互動。它需要
  `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key` 以及絕對
  `OPENCLAW_LIVE_CODEX_HARNESS_MODEL_CATALOG` 路徑。目錄必須以 `max_context_window: 922000` 公開
  所選模型，Codex 才不會將
  覆寫值限制回一般目錄視窗。上述一般的縮減臨界值
  壓力測試會保留更嚴格的自動壓縮與隱藏標記
  保留斷言。
- 選用迴圈轉送選擇停用探測：
  `OPENCLAW_LIVE_CODEX_HARNESS_DISABLE_LOOP_RELAY=1`
- 要求的思考偏好可能會對應至 Codex 為該模型公告的最接近投入程度。
  例如，Luna 會將 `minimal` 對應至 `low`。
- 已知的 Codex 目錄模型會自動推導出該確切的原生投入程度。
  未知的模型覆寫必須指定預期的對應投入程度。
- 冒煙測試會強制使用供應商／模型 `agentRuntime.id: "codex"`，因此損壞的 Codex
  測試框架無法藉由無聲回復至 OpenClaw 而通過。
- 驗證：使用本機 Codex 訂閱登入的 Codex app-server 驗證，或在
  `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key` 時使用 `OPENAI_API_KEY`。Docker 可
  複製 `~/.codex/auth.json` 與 `~/.codex/config.toml` 以執行訂閱測試。

本機操作方式：

```bash
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/gpt-5.6-luna \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

Docker 操作方式：

```bash
pnpm test:docker:live-codex-harness
```

重新啟動與歷史記錄壓力測試：

```bash
OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1 \
pnpm test:docker:live-codex-harness
```

扇出、大型輸出、壓縮與重新啟動壓力測試：

```bash
OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_COUNT=8 \
  OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS=1 \
  pnpm test:docker:live-codex-harness
```

完整原生 Codex `922000` 輸入預算壓縮壓力測試：

```bash
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_FULL_CONTEXT=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL_CATALOG=/absolute/path/to/models-api-1m.json \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/gpt-5.6-terra \
  OPENCLAW_LIVE_CODEX_HARNESS_THINKING=medium \
  OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS_TURNS=8 \
  OPENCLAW_LIVE_CODEX_HARNESS_LARGE_OUTPUT_BYTES=800000 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

GPT-5.6 原生 Codex 矩陣：

```bash
OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_TARGETS='openai/gpt-5.6-sol=ultra,openai/gpt-5.6-terra=ultra,openai/gpt-5.6-luna=max' \
  pnpm test:docker:live-codex-harness
```

## 即時：OpenAI 重複壓縮

- 目標：透過至少兩次真實的自動壓縮，執行內嵌的 OpenClaw `openai-responses` 代理程式迴圈，然後驗證持久性標記仍然存在。
- 測試：`src/agents/sessions/agent-session.openai-compaction.live.test.ts`
- 啟用：`OPENCLAW_LIVE_OPENAI_COMPACTION=1`
- 預設模型：`gpt-5.6-luna`
- 模型覆寫：`OPENCLAW_LIVE_OPENAI_COMPACTION_MODEL=<model>`
- 一般壓力模式會使用縮減的用戶端上下文預算，以有限的 API 費用觸發相同的真實壓縮路徑。
- 完整上下文模式會將用戶端預算設為 `922000`，並將壓縮保留量設為 `222000`，因此自動壓縮會在 `700000` 開始。它還要求觀察到的供應商輸入計數高於 `272000` 長上下文定價界線。

有限的即時測試做法：

```bash
OPENCLAW_LIVE_TEST=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION=1 \
  pnpm test:live -- src/agents/sessions/agent-session.openai-compaction.live.test.ts
```

完整 `922000` 輸入預算做法：

```bash
OPENCLAW_LIVE_TEST=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION_FULL=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION_MODEL=gpt-5.6-terra \
  pnpm test:live -- src/agents/sessions/agent-session.openai-compaction.live.test.ts
```

<Warning>
完整模式會刻意跨越 OpenAI 的長上下文定價界線，並且可能發出數次大型 API 呼叫。只有在取得明確的費用核准後才能使用。
</Warning>

全新 OpenAI API 金鑰預設值：

```bash
OPENCLAW_LIVE_GATEWAY_OPENAI_API_DEFAULT=1 \
  OPENCLAW_LIVE_GATEWAY_PROVIDERS=openai \
  OPENCLAW_LIVE_GATEWAY_THINKING=off \
  pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
```

此驗證會讓 `OPENCLAW_LIVE_GATEWAY_MODELS` 保持未設定狀態，透過全新入門設定的推論選擇接合點解析模型、斷言 `openai/gpt-5.6`，然後使用解析出的模型執行一次真實的閘道回合。

GPT-5.6 內嵌 OpenClaw 矩陣：

```bash
OPENCLAW_LIVE_GATEWAY_THINKING=ultra \
  OPENCLAW_LIVE_GATEWAY_PROVIDERS=openai \
  OPENCLAW_LIVE_GATEWAY_MODELS='openai/gpt-5.6-sol,openai/gpt-5.6-terra,openai/gpt-5.6-luna' \
  pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
```

Docker 注意事項：

- Docker 執行器位於 `scripts/test-live-codex-harness-docker.sh`。
- 它會傳遞 `OPENAI_API_KEY`、複製現有的 Codex 命令列介面驗證檔案、將 `@openai/codex` 安裝至可寫入且已掛載的 npm 前綴、暫存原始碼樹狀目錄，然後只執行 Codex 測試框架即時測試。
- Docker 預設會啟用映像、MCP／工具及 Guardian 探查。需要範圍較小的偵錯執行時，請設定 `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0`、`OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0` 或 `OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0`。
- Docker 使用相同的明確 Codex 執行階段設定，因此舊版別名或 OpenClaw 後援機制無法掩蓋 Codex 測試框架的迴歸。
- 矩陣目標會在單一容器中依序執行。Docker 指令碼會根據目標數量調整預設的 35 分鐘逾時時間；任何外層殼層或 CI 逾時設定都必須容許相同的總時間。標準 CI 會將每個 GPT-5.6 目標置於不同的分片。

### 建議的即時測試做法

範圍較小且明確的允許清單速度最快，也最不容易出現不穩定情況：

- 單一模型，直接執行（不使用閘道）：
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.6-luna" pnpm test:live src/agents/models.profiles.live.test.ts`

- 小型模型直接執行設定檔：
  - `OPENCLAW_LIVE_MODELS=small pnpm test:live src/agents/models.profiles.live.test.ts`

- 小型模型閘道設定檔：
  - `OPENCLAW_LIVE_GATEWAY_MODELS=small pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Ollama Cloud API 冒煙測試：
  - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com OPENCLAW_LIVE_OLLAMA_MODEL=glm-5.1:cloud OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0 pnpm test:live -- extensions/ollama/ollama.live.test.ts`

- 單一模型，閘道冒煙測試：
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- 跨多個供應商的工具呼叫：
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,google/gemini-3.5-flash,deepseek/deepseek-v4-flash,zai/glm-5.1,minimax/MiniMax-M3" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Z.AI Coding Plan GLM-5.2 直接冒煙測試：
  - `ZAI_CODING_LIVE_TEST=1 pnpm test:live src/agents/zai.live.test.ts`

- Google 重點測試（Gemini API 金鑰 + Antigravity）：
  - Gemini（API 金鑰）：`OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3.5-flash" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity（OAuth）：`OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google 自適應思考冒煙測試（來自私有 QA 命令列介面的 `qa manual` — 需要 `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1` 和原始碼簽出；請參閱 [QA 概觀](/zh-TW/concepts/qa-e2e-automation)）：
  - Gemini 3 動態預設值：`OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa manual --provider-mode live-frontier --model google/gemini-3.1-pro-preview --alt-model google/gemini-3.1-pro-preview --message '/think adaptive Reply exactly: GEMINI_ADAPTIVE_OK' --timeout-ms 180000`
  - Gemini 2.5 動態預算：`OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa manual --provider-mode live-frontier --model google/gemini-2.5-flash --alt-model google/gemini-2.5-flash --message '/think adaptive Reply exactly: GEMINI25_ADAPTIVE_OK' --timeout-ms 180000`

注意事項：

- `google/...` 使用 Gemini API（API 金鑰）。
- `google-antigravity/...` 使用 Antigravity OAuth 橋接器（Cloud Code Assist 風格的代理程式端點）。
- `google-gemini-cli/...` 使用你電腦上的本機 Gemini 命令列介面（具有獨立的驗證方式與工具特性）。
- Gemini API 與 Gemini 命令列介面：
  - API：OpenClaw 透過 HTTP 呼叫 Google 託管的 Gemini API（API 金鑰／設定檔驗證）；這是大多數使用者所指的「Gemini」。
  - 命令列介面：OpenClaw 透過殼層呼叫本機 `gemini` 二進位檔；它具有自己的驗證方式，而且行為可能不同（串流／工具支援／版本差異）。

## 即時測試：模型矩陣（涵蓋範圍）

即時測試必須選擇啟用，因此沒有固定的「CI 模型清單」。`OPENCLAW_LIVE_MODELS=modern`／`OPENCLAW_LIVE_GATEWAY_MODELS=modern`（以及其 `all` 別名）會依下列優先順序，執行 `src/agents/live-model-filter.ts` 中來自 `HIGH_SIGNAL_LIVE_MODEL_PRIORITY` 的精選優先清單：

| 供應商／模型                                  | 注意事項   |
| --------------------------------------------- | ---------- |
| `anthropic/claude-opus-5`                     |            |
| `anthropic/claude-opus-4-8`                   |            |
| `anthropic/claude-sonnet-5`                   |            |
| `anthropic/claude-sonnet-4-6`                 |            |
| `anthropic/claude-opus-4-7`                   |            |
| `google/gemini-3.1-pro-preview`               | Gemini API |
| `google/gemini-3.5-flash`                     | Gemini API |
| `cohere/command-a-plus-05-2026`               |            |
| `moonshot/kimi-k3`                            |            |
| `anthropic/claude-opus-4-6`                   |            |
| `deepseek/deepseek-v4-flash`                  |            |
| `deepseek/deepseek-v4-pro`                    |            |
| `minimax/MiniMax-M3`                          |            |
| `openai/gpt-5.5`                              |            |
| `openrouter/openai/gpt-5.2-chat`              |            |
| `openrouter/minimax/minimax-m2.7`             |            |
| `opencode-go/glm-5`                           |            |
| `openrouter/ai21/jamba-large-1.7`             |            |
| `xai/grok-4.5`                                |            |
| `xai/grok-4.20-0309-reasoning`                |            |
| `zai/glm-5.1`                                 |            |
| `fireworks/accounts/fireworks/models/glm-5p1` |            |
| `minimax-portal/minimax-m3`                   |            |

精選的**小型模型**清單（`OPENCLAW_LIVE_MODELS=small`／`OPENCLAW_LIVE_GATEWAY_MODELS=small`）來自 `SMALL_LIVE_MODEL_PRIORITY`：

| 供應商／模型                  |
| ---------------------------- |
| `lmstudio/qwen/qwen3.5-9b`   |
| `vllm/qwen/qwen3-8b`         |
| `sglang/qwen/qwen3-8b`       |
| `ollama/gemma3:4b`           |
| `openrouter/qwen/qwen3.5-9b` |
| `openrouter/z-ai/glm-5.1`    |
| `openrouter/z-ai/glm-5`      |
| `zai/glm-5.1`                |

新版清單的注意事項：

- `codex` 和 `codex-cli` 供應商不包含在預設的新版全面測試中（它們涵蓋命令列介面後端／ACP 行為，已於上方分別測試）。`openai/gpt-5.5` 本身預設會透過 Codex 應用程式伺服器測試框架進行路由；請參閱[即時測試：Codex 應用程式伺服器測試框架冒煙測試](#live-codex-app-server-harness-smoke)。
- `fireworks`、`google`、`openrouter` 和 `xai` 在新版全面測試中只會執行明確精選的模型 ID（不會自動展開為「此供應商的所有模型」）。
- 在 `OPENCLAW_LIVE_GATEWAY_MODELS` 中至少包含一個支援映像的模型（Claude／Gemini／OpenAI 系列視覺變體等），以執行映像探查。

使用手動挑選的跨供應商集合，執行包含工具和映像的閘道冒煙測試：

```bash
OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3.5-flash,google-antigravity/claude-opus-4-6-thinking,deepseek/deepseek-v4-flash,zai/glm-5.1,minimax/MiniMax-M3" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts
```

精選清單以外的選用額外涵蓋範圍（最好包含；選擇一個你已啟用且支援「工具」的模型）：

- Mistral：`mistral/...`
- Cerebras：`cerebras/...`（如果你有存取權）
- LM Studio：`lmstudio/...`（本機；工具呼叫取決於 API 模式）

### 彙整器／替代閘道

如果你已啟用金鑰，也可以透過下列方式測試：

- OpenRouter：`openrouter/...`（數百個模型；使用 `openclaw models scan` 尋找支援工具與映像的候選項目）
- OpenCode：Zen 使用 `opencode/...`，Go 使用 `opencode-go/...`（透過 `OPENCODE_API_KEY`／`OPENCODE_ZEN_API_KEY` 驗證）

可納入即時測試矩陣的其他供應商（如果你有認證資訊／設定）：

- 內建：`anthropic`、`cerebras`、`github-copilot`、`google`、`google-antigravity`、`google-gemini-cli`、`google-vertex`、`groq`、`mistral`、`openai`、`openrouter`、`opencode`、`opencode-go`、`xai`、`zai`
- 透過 `models.providers`（自訂端點）：`minimax`（雲端／API），以及任何與 OpenAI／Anthropic 相容的 Proxy（LM Studio、vLLM、LiteLLM 等）

<Tip>
請勿在文件中硬式編碼「所有模型」。權威清單取決於你電腦上的 `discoverModels(...)` 傳回內容，以及可用的金鑰。
</Tip>

## 認證資訊（絕不可提交）

即時測試會使用與命令列介面相同的方式探索認證資訊。實務上的影響如下：

- 如果命令列介面可以運作，即時測試應該也能找到相同的金鑰。
- 如果即時測試顯示「沒有認證資訊」，請使用與偵錯 `openclaw models list`／模型選擇相同的方式偵錯。

- 每個代理程式的驗證設定檔：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（這就是即時測試中「設定檔金鑰」的含義）
- 設定：`~/.openclaw/openclaw.json`（或 `OPENCLAW_CONFIG_PATH`）
- 舊版 OAuth 目錄：`~/.openclaw/credentials/`（若存在，會複製到暫存的即時測試主目錄，但不是主要的設定檔金鑰存放區）
- 本機即時測試執行會將作用中的設定（移除 `agents.*.workspace`／`agentDir` 覆寫）及每個代理程式的 `auth-profiles.json` 複製到暫存的測試主目錄，而不會複製該代理程式目錄的其餘部分，因此 `workspace/` 和 `sandboxes/` 資料絕不會進入暫存主目錄；此外也會複製舊版 `credentials/` 目錄，以及支援的外部命令列介面驗證檔案／目錄（`.claude.json`、`.claude/.credentials.json`、`.claude/settings*.json`、`.claude/backups`、`.codex/auth.json`、`.codex/config.toml`、`.gemini`、`.minimax`）。

如果要依賴環境金鑰，請在本機測試前匯出它們，或使用下方的 Docker 執行器並明確指定 `OPENCLAW_PROFILE_FILE`。

## Deepgram 即時測試（音訊轉錄）

- 測試：`extensions/deepgram/audio.live.test.ts`
- 啟用：`DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live extensions/deepgram/audio.live.test.ts`

## BytePlus 編碼方案即時測試

- 測試：`extensions/byteplus/live.test.ts`
- 啟用：`BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live extensions/byteplus/live.test.ts`
- 選用模型覆寫：`BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI 工作流程媒體即時測試

- 測試：`extensions/comfy/comfy.live.test.ts`
- 啟用：`OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- 範圍：
  - 執行隨附的 comfy 映像、影片及 `music_generate` 路徑
  - 除非已設定 `plugins.entries.comfy.config.<capability>`，否則會略過各項功能
  - 適合在變更 comfy 工作流程提交、輪詢、下載或外掛註冊後使用

## 映像產生即時測試

- 測試：`test/image-generation.runtime.live.test.ts`
- 命令：`pnpm test:live test/image-generation.runtime.live.test.ts`
- 測試框架：`pnpm test:live:media image`
- 範圍：
  - 列舉每個已註冊的影像生成提供者外掛
  - 在探測前使用已匯出的提供者環境變數
  - 預設優先使用即時／環境 API 金鑰，而非已儲存的驗證設定檔，因此 `auth-profiles.json` 中過時的測試金鑰不會遮蔽真正的 shell 認證資訊
  - 略過沒有可用驗證／設定檔／模型的提供者
  - 透過共用影像生成執行階段執行每個已設定的提供者：
    - `<provider>:generate`
    - 當提供者宣告支援編輯時，執行 `<provider>:edit`
- 目前涵蓋的內建提供者：
  - `deepinfra`
  - `fal`
  - `google`
  - `minimax`
  - `openai`
  - `openrouter`
  - `vydra`
  - `xai`
- 選用的縮小範圍設定：
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google,openrouter,xai"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="deepinfra"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-2,google/gemini-3.1-flash-image,openrouter/google/gemini-3.1-flash-image-preview,xai/grok-imagine-image"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit,openrouter:generate,xai:default-generate,xai:default-edit"`
- 選用的驗證行為：
  - 使用 `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` 強制採用設定檔儲存區驗證，並忽略僅限環境變數的覆寫

對於已發布的命令列介面路徑，請在提供者／執行階段即時測試通過後，新增一項 `infer` 煙霧測試：

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_INFER_CLI_TEST=1 pnpm test:live -- test/image-generation.infer-cli.live.test.ts
openclaw infer image providers --json
openclaw infer image generate \
  --model google/gemini-3.1-flash-image \
  --prompt "最簡約的扁平測試影像：白色背景上有一個藍色正方形，沒有文字。" \
  --output ./openclaw-infer-image-smoke.png \
  --json
```

這涵蓋命令列介面引數剖析、設定／預設代理程式解析、內建外掛啟用、共用影像生成執行階段，以及即時提供者請求。預期在載入執行階段前，外掛相依套件皆已存在。

## 音樂生成即時測試

- 測試：`extensions/music-generation-providers.live.test.ts`
- 啟用：`OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- 測試框架：`pnpm test:live:media music`
- 範圍：
  - 測試共用的內建音樂生成提供者路徑
  - 目前涵蓋 `fal`、`google`、`minimax` 和 `openrouter`
  - 在探測前使用已匯出的提供者環境變數
  - 預設優先使用即時／環境 API 金鑰，而非已儲存的驗證設定檔，因此 `auth-profiles.json` 中過時的測試金鑰不會遮蔽真正的 shell 認證資訊
  - 略過沒有可用驗證／設定檔／模型的提供者
  - 可用時執行兩種已宣告的執行階段模式：
    - `generate` 使用僅含提示詞的輸入
    - 當提供者宣告 `capabilities.edit.enabled` 時，執行 `edit`
  - `comfy` 有自己的獨立即時測試檔案，不在此共用掃描中
- 選用的縮小範圍設定：
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.6"`
- 選用的驗證行為：
  - 使用 `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` 強制採用設定檔儲存區驗證，並忽略僅限環境變數的覆寫

## 影片生成即時測試

- 測試：`extensions/video-generation-providers.live.test.ts`
- 啟用：`OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- 測試框架：`pnpm test:live:media video`
- 範圍：
  - 跨 `alibaba`、`byteplus`、`deepinfra`、`fal`、`google`、`minimax`、`openai`、`openrouter`、`pixverse`、`qwen`、`runway`、`together`、`vydra`、`xai` 測試共用的內建影片生成提供者路徑
  - 預設採用適合發布的煙霧測試路徑：每個提供者提出一次文字轉影片請求、使用一秒鐘的龍蝦提示詞，並套用來自 `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` 的每個提供者操作上限（預設為 `180000`）
  - 預設略過 FAL，因為提供者端的佇列延遲可能佔用大部分發布時間；傳入 `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"`（或清除略過清單）即可明確執行
  - 在探測前使用已匯出的提供者環境變數
  - 預設優先使用即時／環境 API 金鑰，而非已儲存的驗證設定檔，因此 `auth-profiles.json` 中過時的測試金鑰不會遮蔽真正的 shell 認證資訊
  - 略過沒有可用驗證／設定檔／模型的提供者
  - 預設僅執行 `generate`
  - 設定 `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1`，以便在可用時一併執行已宣告的轉換模式：
    - 當提供者宣告 `capabilities.imageToVideo.enabled`，且所選提供者／模型在共用掃描中接受以緩衝區為基礎的本機影像輸入時，執行 `imageToVideo`
    - 當提供者宣告 `capabilities.videoToVideo.enabled`，且所選提供者／模型在共用掃描中接受以緩衝區為基礎的本機影片輸入時，執行 `videoToVideo`
  - 共用掃描中目前已宣告但略過的 `imageToVideo` 提供者：
    - `vydra`（此測試通道不支援以緩衝區為基礎的本機影像輸入）
  - Vydra 的提供者專屬涵蓋範圍：
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - 該檔案會執行 `veo3` 文字轉影片，以及 `kling` 影像轉影片測試通道；後者預設使用遠端影像 URL 測試資料（可用 `OPENCLAW_LIVE_VYDRA_KLING_IMAGE_URL` 覆寫）。
  - xAI 的提供者專屬涵蓋範圍：
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "classic Grok Imagine"`
    - 傳統案例會先生成正方形的本機 PNG 首格影像、省略幾何設定、請求一秒鐘的影像轉影片片段、輪詢至完成，並驗證下載的緩衝區。
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "Grok Imagine Video 1.5"`
    - 1.5 案例會生成本機 PNG 首格影像、請求一秒鐘的 1080P 影像轉影片片段、輪詢至完成，並驗證下載的緩衝區。
  - 目前的 `videoToVideo` 即時測試涵蓋範圍：
    - 僅當所選模型解析為 `gen4_aleph` 時執行 `runway`
  - 共用掃描中目前已宣告但略過的 `videoToVideo` 提供者：
    - `alibaba`、`google`、`openai`、`qwen`、`xai`，因為這些路徑目前需要遠端 `http(s)` 參考 URL，而非以緩衝區為基礎的本機輸入
- 選用的縮小範圍設定：
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="deepinfra,google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - 使用 `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""`，將所有提供者納入預設掃描，包括 FAL
  - 使用 `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000`，降低每個提供者的操作上限，以執行更積極的煙霧測試
- 選用的驗證行為：
  - 使用 `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` 強制採用設定檔儲存區驗證，並忽略僅限環境變數的覆寫

## 媒體即時測試框架

- 命令：`pnpm test:live:media`
- 進入點：`test/e2e/qa-lab/media/hosted-media-provider-live.ts`，它會針對每個所選測試套件執行 `pnpm test:live -- <suite-test-file>`，因此心跳偵測與安靜模式行為會與其他 `pnpm test:live` 執行保持一致。
- 用途：
  - 透過單一儲存庫原生進入點執行共用的影像、音樂及影片即時測試套件
  - 從 `~/.profile` 自動載入缺少的提供者環境變數
  - 預設會自動將每個測試套件縮小至目前具有可用驗證的提供者
- 旗標：
  - `--providers <csv>` 是全域提供者篩選器；`--image-providers`／`--music-providers`／`--video-providers` 會將篩選器範圍限定至單一測試套件
  - `--all-providers` 會略過依據驗證狀態進行的自動篩選
  - 若篩選後沒有可執行的提供者，`--allow-empty` 會以 `0` 結束
  - `--quiet`／`--no-quiet` 會傳遞至 `test:live`
- 範例：
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## 相關內容

- [測試](/zh-TW/help/testing)－單元、整合、QA 與 Docker 測試套件
