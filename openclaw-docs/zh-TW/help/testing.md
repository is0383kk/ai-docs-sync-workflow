---
read_when:
    - 在本機或 CI 中執行測試
    - 為模型／供應商錯誤新增迴歸測試
    - 偵錯閘道與代理程式行為
summary: 測試工具組：單元／端對端／即時測試套件、Docker 執行器，以及各項測試的涵蓋範圍
title: 測試
x-i18n:
    generated_at: "2026-07-26T08:20:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 20e0aa22bf16561334f83342abffabb387ed0b41b901773939123ecfbc0ae330
    source_path: help/testing.md
    workflow: 16
---

OpenClaw 有三套 Vitest 測試套件（單元／整合、端對端、即時），另有 Docker
執行器。本頁說明每套測試涵蓋的範圍、特定工作流程應執行哪個命令、
即時測試如何探索認證資訊，以及如何為實際的供應商／模型錯誤新增
迴歸測試。

<Note>
**QA 技術堆疊（qa-lab、qa-channel、即時傳輸通道）**另有文件說明：

- [QA 概覽](/zh-TW/concepts/qa-e2e-automation) - 架構、命令介面、情境編寫方式，以及 Matrix 設定檔。
- [成熟度計分卡](/zh-TW/maturity/scorecard) - 發行版 QA 證據如何支援穩定性與 LTS 決策。
- [QA 頻道](/zh-TW/channels/qa-channel) - 儲存庫支援情境所使用的合成傳輸外掛。

本頁涵蓋一般測試套件與 Docker／Parallels 執行器。下方的 [QA 專用執行器](#qa-specific-runners)列出具體的 `qa` 呼叫方式，並連回上述參考資料。
</Note>

## 快速開始

大多數時候：

- 完整閘門（預期在推送前執行）：`pnpm build && pnpm check && pnpm check:test-types && pnpm test`
- 在資源充裕的機器上較快執行本機完整測試套件：`pnpm test:max`
- 直接執行 Vitest 監看迴圈：`pnpm test:watch`
- 直接指定檔案也會路由外掛／頻道路徑：`pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- 針對單一失敗案例反覆修改時，請優先執行目標明確的測試。
- 以 Docker 為基礎的 QA 站台：`pnpm qa:lab:up`
- 以 Linux VM 為基礎的 QA 通道：`pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

修改測試或需要更高信心時：

- 供參考的 V8 覆蓋率報告：`pnpm test:coverage`
- 端對端測試套件：`pnpm test:e2e`

## 測試暫存目錄

測試擁有的暫存目錄請使用 `test/helpers/temp-dir.ts` 中的共用輔助工具，
以明確標示擁有權，並讓清理作業留在測試生命週期內：

```ts
import { afterEach } from "vitest";
import { useAutoCleanupTempDirTracker } from "../helpers/temp-dir.js";

const tempDirs = useAutoCleanupTempDirTracker(afterEach);

it("使用暫存工作區", () => {
  const workspace = tempDirs.make("openclaw-example-");
  // 使用工作區
});
```

`useAutoCleanupTempDirTracker(afterEach)` 刻意不提供手動
清理方法，因為 Vitest 會負責在每次測試後清理。較舊的低階
輔助工具（`makeTempDir`、`cleanupTempDirs`、`createTempDirTracker`）仍然存在，
供尚未遷移的測試使用；請避免新增這些工具的使用方式，也不要新增直接的
`fs.mkdtemp*` 呼叫，除非測試明確要驗證原始暫存目錄
行為。確實需要直接使用暫存目錄時，請新增可稽核且附有原因的允許
註解：

```ts
// openclaw-temp-dir: allow 驗證原始檔案系統清理行為
const workspace = fs.mkdtempSync(prefix);
```

`node scripts/report-test-temp-creations.mjs` 會報告新增差異行中直接建立暫存目錄，
以及新增手動使用共用輔助工具的情況，但不會封鎖既有的清理方式。
它採用與 `scripts/changed-lanes.mjs` 相同的測試路徑分類方式，
並略過共用輔助工具本身的實作。`check:changed` 會針對已變更的測試路徑執行此報告，
作為僅警告的 CI 訊號（GitHub 警告註解，而非失敗）。

## 即時與 Docker／Parallels 工作流程

偵錯實際的供應商／模型時（需要真實認證資訊）：

- 即時測試套件（模型＋閘道工具／影像探查）：`pnpm test:live`
- 以安靜模式指定一個即時測試檔案：`pnpm test:live -- src/agents/models.profiles.live.test.ts`
- 執行階段效能報告：分派 `OpenClaw Performance`，並搭配
  `live_openai_candidate=true` 執行真實的 `openai/gpt-5.6-luna` 代理程式回合，或搭配
  `deep_profile=true` 產生 Kova CPU／堆積／追蹤成品。每日排程執行會
  透過獨立的成品取用發布工作，將模擬供應商、深度分析及 GPT-5.6 Luna 通道報告
  發布至 `openclaw/clawgrit-reports`；
  發布者驗證缺失或無效時，排程與
  `profile=release` 執行都會失敗。非發行版的手動分派會保留 GitHub 成品，
  並將報告發布視為建議性質。模擬供應商報告也包含
  原始碼層級的閘道啟動、記憶體、外掛壓力、重複的
  假模型 hello 迴圈，以及命令列介面啟動數據。
- Docker 即時模型全面測試：`pnpm test:docker:live-models`
  - 每個選定的模型都會執行一個文字回合及小型的檔案讀取型探查。
    中繼資料宣告支援 `image` 輸入的模型，也會執行一個微型影像回合。
    隔離供應商失敗時，可使用 `OPENCLAW_LIVE_MODEL_FILE_PROBE=0` 或
    `OPENCLAW_LIVE_MODEL_IMAGE_PROBE=0` 停用額外探查。
  - CI 覆蓋範圍：每日 `OpenClaw Scheduled Live And E2E Checks` 與手動
    `OpenClaw Release Checks` 都會搭配
    `include_live_suites: true` 呼叫可重複使用的即時／端對端工作流程，其中包括
    依供應商分片的 Docker 即時模型矩陣工作。
  - 如需執行聚焦的 CI 重跑，請搭配 `include_live_suites: true` 和 `live_models_only: true`
    分派 `OpenClaw Live And E2E Checks (Reusable)`。
  - 請將新的高訊號供應商密鑰加入 `scripts/ci-hydrate-live-auth.sh`，
    以及 `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml` 和其
    排程／發行呼叫端。
- 原生 Codex 綁定聊天冒煙測試：`pnpm test:docker:live-codex-bind`
  - 針對 Codex app-server 路徑執行 Docker 即時通道，使用
    `/codex bind` 綁定合成 Slack 私訊，操作 `/codex fast` 和
    `/codex permissions`，然後驗證純文字回覆與影像附件
    是否透過原生外掛綁定而非 ACP 路由。
- Codex app-server 測試框架冒煙測試：`pnpm test:docker:live-codex-harness`
  - 透過外掛擁有的 Codex app-server
    測試框架執行閘道代理程式回合、驗證 `/codex status` 和 `/codex models`，並預設
    執行影像、排程 MCP、子代理程式及 Guardian 探查。隔離其他失敗時，
    可使用 `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=0` 停用
    子代理程式探查。若要聚焦檢查子代理程式，請停用
    其他探查：
    `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 pnpm test:docker:live-codex-harness`。
    除非設定 `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_ONLY=0`，
    否則此程序會在子代理程式探查後結束。
- Codex 隨選安裝冒煙測試：`pnpm test:docker:codex-on-demand`
  - 在 Docker 中安裝已封裝的 OpenClaw tarball、執行 OpenAI API 金鑰
    初始設定，並驗證 Codex 外掛及 `@openai/codex` 相依套件
    已隨選下載至受管理的 npm 專案根目錄。
- Codex npm 外掛即時套件冒煙測試：`pnpm test:docker:live-codex-npm-plugin`
  - 在 Docker 中安裝候選 OpenClaw 套件及精確版本的 Codex 外掛，
    然後使用真實的 OpenAI 金鑰進行命令列介面預檢與同工作階段回合。
  - 其零重試、中等思考的後續回合必須傳送進度、持續
    完成隨機化工作區讀取及精確的成品寫入，
    再傳送完成訊息。僅回報進度的終止回合會使該通道失敗。
- 即時外掛工具相依套件冒煙測試：`pnpm test:docker:live-plugin-tool`
  - 封裝一個含有真實 `slugify` 相依套件的固定資料外掛，透過
    `npm-pack:` 安裝，驗證受管理 npm
    專案根目錄下的相依套件，然後要求即時 OpenAI 模型呼叫外掛工具並
    傳回隱藏的 slug。
- OpenClaw 救援命令冒煙測試：`pnpm test:live:system-agent-rescue-channel`
  - 針對訊息頻道救援命令
    介面的選用雙重保障檢查。操作 `/openclaw status`、將持久性模型
    變更排入佇列、回覆 `/openclaw yes`，並驗證稽核／設定寫入
    路徑。
- OpenClaw 首次執行 Docker 冒煙測試：`pnpm test:docker:system-agent-first-run`
  - 從空白 OpenClaw 狀態目錄開始，首先證明已封裝的
    `openclaw setup` 命令列介面會在無法推論時採取封閉式失敗。接著
    透過已封裝的啟用模組測試並啟用假的 Claude。
    此後，模糊的已封裝命令列介面請求才會到達規劃器，
    並解析為具型別的設定，接著執行一次性模型、代理程式、Discord 設定
    及 SecretRef 操作。它會驗證設定與稽核項目。這是
    輔助性的閘門／操作證據，而非互動式初始設定，也不是
    OpenClaw 代理程式／工具／核准證明。相同通道也透過
    `pnpm openclaw qa suite --scenario system-agent-ring-zero-setup` 公開於 QA Lab。
- Moonshot／Kimi 成本冒煙測試：設定 `MOONSHOT_API_KEY` 後，執行
  `openclaw models list --provider moonshot --json`，再針對 `moonshot/kimi-k2.6` 執行隔離的
  `openclaw agent --local --session-id live-kimi-cost --message 'Reply exactly: KIMI_LIVE_OK' --thinking off --json`。
  驗證 JSON 回報 Moonshot/K2.6，且助理逐字稿儲存正規化的 `usage.cost`。

<Tip>
如果只需要處理一個失敗案例，請優先使用下方所述的允許清單環境變數縮小即時測試範圍。
</Tip>

## QA 專用執行器

需要 QA Lab 擬真度時，這些命令可與主要測試套件搭配使用。

CI 會在專用工作流程中執行 QA Lab。代理式一致性測試內嵌於
`QA-Lab - All Lanes` 與發行驗證中，而不是獨立的 PR 工作流程。
廣泛驗證應搭配 `rerun_group=qa-parity` 使用 `Full Release Validation`，
或使用發行檢查的 QA 群組。穩定版／預設發行
檢查會將完整的即時／Docker 長時間測試置於 `run_release_soak=true` 後方；
`full` 設定檔會強制啟用長時間測試。`QA-Lab - All Lanes` 每晚在 `main` 上執行，
也可透過手動分派執行，並將模擬一致性通道、即時 Matrix 通道、
由 Convex 管理的即時 Telegram 通道，以及由 Convex 管理的即時 Discord 通道作為
平行工作。排程 QA 與發行檢查會透過共用即時配接器執行 Matrix 發行設定檔。
Matrix 命令列介面與手動工作流程輸入的預設值維持為 `all`；
手動 `all` 分派會展開傳輸、媒體及
E2EE 設定檔，而聚焦分派可選擇 `fast`、`release` 或
`transport`。`OpenClaw Release Checks` 會在核准發行前執行一致性測試、
可重複使用的 Matrix 即時配接器設定檔及 Telegram 通道。
發行傳輸檢查使用 `mock-openai/gpt-5.6-luna`，以維持確定性並
避免一般的供應商外掛啟動。這些即時傳輸閘道
會停用記憶體搜尋；記憶體行為仍由 QA 一致性測試套件涵蓋。

完整發行版即時媒體分片使用
`ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04`，其中已包含
`ffmpeg` 和 `ffprobe`。Docker 即時模型／後端分片使用針對每個選定
提交只建置一次的共用 `ghcr.io/openclaw/openclaw-live-test:<sha>` 映像，
之後以 `OPENCLAW_SKIP_DOCKER_BUILD=1` 提取，而非在
每個分片內重新建置。

- `pnpm openclaw qa suite`
  - 直接在主機上執行由儲存庫支援的 QA 情境。
  - 為所選情境集寫入頂層 `qa-evidence.json`、`qa-suite-summary.json` 和
    `qa-suite-report.md` 成品，包括
    混合流程、Vitest 和 Playwright 情境選項。
  - 由 `pnpm openclaw qa run --qa-profile <profile>` 分派時，會將
    所選分類設定檔的評分卡嵌入同一個 `qa-evidence.json`。
    `smoke-ci` 會寫入精簡證據（`evidenceMode: "slim"`，沒有逐項
    `execution`）。`release` 涵蓋精選的發行就緒範圍；`all`
    會選取每個使用中的成熟度類別，並在需要完整評分卡成品時，
    以明確的 QA Profile Evidence 工作流程分派為目標。
  - 預設使用隔離的
    閘道工作程序，平行執行多個所選情境。`qa-channel` 的預設並行數為 4（上限為
    所選情境數量）。使用 `--concurrency <count>` 調整工作程序
    數量，或使用 `--concurrency 1` 啟用舊版循序執行路徑。
  - 任何情境失敗時，以非零狀態結束。若需要產生成品但不希望使用失敗結束碼，
    請使用 `--allow-failures`。
  - 支援提供者模式 `live-frontier`、`mock-openai` 和 `aimock`。
    `aimock` 會啟動由本機 AIMock 支援的提供者伺服器，用於實驗性
    測試資料與通訊協定模擬涵蓋範圍，且不會取代能感知情境的
    `mock-openai` 執行路徑。
- `pnpm openclaw qa coverage --match <query>`
  - 搜尋情境 ID、標題、介面、涵蓋範圍 ID、文件參照、程式碼
    參照、外掛和提供者需求，然後列印相符的套件
    目標。
  - 如果你知道受影響的行為或檔案路徑，但不知道最小的情境，
    請在執行 QA Lab 前使用此功能。這僅供參考——仍應根據正在變更的行為，
    選擇模擬、即時、Multipass、Matrix 或傳輸證明。
- `pnpm test:plugins:kitchen-sink-live`
  - 透過 QA Lab 執行即時 OpenAI Kitchen Sink 外掛的全套嚴格測試。
    安裝外部 Kitchen Sink 套件、驗證外掛 SDK
    介面清單、探測 `/healthz` 和 `/readyz`、記錄閘道
    CPU/RSS 證據、執行一次即時 OpenAI 回合，並檢查對抗性
    診斷。需要即時 OpenAI 認證，例如 `OPENAI_API_KEY`。在
    已填入認證資訊的 Testbox 工作階段中，如果有 `openclaw-testbox-env` 輔助工具，
    便會自動載入 Testbox 即時驗證設定檔。
- `pnpm test:gateway:cpu-scenarios`
  - 執行閘道啟動效能基準與一小組模擬 QA Lab 情境套件
    （`channel-chat-baseline`、`memory-failure-fallback`、
    `gateway-restart-inflight-run`），並在 `.artifacts/gateway-cpu-scenarios/` 下寫入合併的 CPU 觀察
    摘要。
  - 預設僅標記持續的高 CPU 觀察結果（`--cpu-core-warn`，
    預設 `0.9`；`--hot-wall-warn-ms`，預設 `30000`），因此短暫的啟動
    峰值會記錄為指標，而不會看起來像持續數分鐘的
    閘道滿載迴歸問題。
  - 針對已建置的 `dist` 成品執行；如果簽出的工作目錄中
    尚無最新的執行階段輸出，請先執行建置。
- `pnpm openclaw qa suite --runner multipass`
  - 在可拋棄的 Multipass Linux VM 中執行相同的 QA 套件，並沿用
    `qa suite` 的相同情境選取及提供者／模型旗標。
  - 即時執行會轉送適合提供給客體的 QA 驗證輸入：
    以環境變數提供的提供者金鑰、QA 即時提供者設定路徑，以及
    存在時的 `CODEX_HOME`。
  - 輸出目錄必須位於儲存庫根目錄下，客體才能透過
    掛載的工作區寫回。
  - 寫入一般 QA 報告與摘要，以及位於
    `.artifacts/qa-e2e/...` 下的 Multipass 記錄。
- `pnpm qa:lab:up`
  - 啟動由 Docker 支援的 QA 網站，以進行操作人員形式的 QA 工作。
- `pnpm test:docker:npm-onboard-channel-agent`
  - 從目前簽出的工作目錄建置 npm tarball、在
    Docker 中全域安裝、執行非互動式 OpenAI API 金鑰上線設定、預設設定
    Telegram、驗證封裝後的外掛執行階段載入時不需要
    修復啟動相依性、執行 doctor，並針對模擬的 OpenAI 端點
    執行一次本機代理程式回合。
  - 使用 `OPENCLAW_NPM_ONBOARD_CHANNEL=discord`，透過 Discord 執行相同的封裝安裝
    路徑。
- `pnpm test:docker:session-runtime-context`
  - 針對嵌入式執行階段內容
    逐字記錄，執行確定性的已建置應用程式 Docker 冒煙測試。驗證隱藏的 OpenClaw 執行階段內容會以
    不顯示的自訂訊息形式保留，而不會洩漏至可見的使用者
    回合；接著植入受影響的損壞工作階段 JSONL，並驗證
    `openclaw doctor --fix` 會將其重寫至使用中的分支並建立備份。
- `pnpm test:docker:npm-telegram-live`
  - 在 Docker 中安裝 OpenClaw 套件候選版本、執行已安裝套件的
    上線設定、透過已安裝的命令列介面設定 Telegram，然後重複使用
    即時 Telegram QA 路徑，並將該已安裝套件作為受測系統
    閘道。
  - 包裝程式僅從簽出的工作目錄掛載 `qa-lab` 測試框架原始碼；
    已安裝套件擁有 `dist`、`openclaw/plugin-sdk` 和隨附的
    外掛執行階段，因此此路徑不會將目前簽出工作目錄的外掛混入
    受測套件。
  - 預設為 `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@beta`；設定
    `OPENCLAW_NPM_TELEGRAM_PACKAGE_TGZ=/path/to/openclaw-current.tgz` 或
    `OPENCLAW_CURRENT_PACKAGE_TGZ`，即可測試解析後的本機 tarball，而非
    從登錄檔安裝。
  - 預設使用 `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES=20`，在 `qa-evidence.json` 中
    輸出重複的 RTT 計時。覆寫
    `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`、
    `OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS` 或
    `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES` 以調整執行。
    `OPENCLAW_NPM_TELEGRAM_RTT_CHECKS` 會選取要取樣的 Telegram QA 情境；
    支援的 RTT 目標為 `channel-canary`。
  - 使用與 `pnpm openclaw qa telegram` 相同的 Telegram 環境認證資訊或 Convex 認證資訊來源。
    對於 CI／發行自動化，請設定
    `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex`、`OPENCLAW_QA_CONVEX_SITE_URL`
    和角色密鑰。如果 CI 中有
    `OPENCLAW_QA_CONVEX_SITE_URL` 和 Convex 角色密鑰，
    Docker 包裝程式會自動選取 Convex。
  - 包裝程式會先在主機上驗證 Telegram 或 Convex 認證資訊環境變數，
    再執行 Docker 建置／安裝工作。僅在
    刻意偵錯認證資訊設定前的階段時，才設定
    `OPENCLAW_NPM_TELEGRAM_SKIP_CREDENTIAL_PREFLIGHT=1`。
  - `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci|maintainer` 僅針對此路徑覆寫
    共用的 `OPENCLAW_QA_CREDENTIAL_ROLE`。選取 Convex
    認證資訊且未設定角色時，包裝程式在 CI 中使用 `ci`，
    在 CI 以外使用 `maintainer`。
  - GitHub Actions 將此路徑公開為手動維護者工作流程
    `NPM Telegram Beta E2E`。合併時不會執行。該工作流程使用
    `qa-live-shared` 環境和 Convex CI 認證資訊租約。
- GitHub Actions 也提供 `Package Acceptance`，用於針對單一候選套件進行旁路產品證明。
  它接受 Git 參照、已發布的 npm 規格、
  HTTPS tarball URL 加 SHA-256、受信任 URL 政策，或另一個執行中的 tarball 成品
  （`source=ref|npm|url|trusted-url|artifact`），將標準化的
  `openclaw-current.tgz` 以 `package-under-test` 名稱上傳，然後使用 `smoke`、`package`、`product`、`full`
  或 `custom` 路徑設定檔執行現有的 Docker E2E 排程器。設定 `telegram_mode=mock-openai` 或
  `live-frontier`，以針對相同的
  `package-under-test` 成品執行 Telegram QA 工作流程。
  - 最新 Beta 版產品證明：

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai
```

- 精確 tarball URL 證明需要摘要，並使用公用 URL 安全政策：

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=url \
  -f package_url=https://registry.npmjs.org/openclaw/-/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

- 企業／私人 tarball 鏡像使用明確的受信任來源政策：

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

`source=trusted-url` 會從受信任的工作流程參照讀取 `.github/package-trusted-sources.json`，且不接受 URL 認證資訊或透過工作流程輸入略過私人網路限制。如果具名政策宣告使用持有人驗證，請設定固定的 `OPENCLAW_TRUSTED_PACKAGE_TOKEN` 密鑰。

- 成品證明會從另一個 Actions 執行下載 tarball 成品：

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=<artifact-name> \
  -f suite_profile=smoke
```

- `pnpm test:docker:plugins`
  - 在 Docker 中封裝並安裝目前的 OpenClaw 組建、啟動已設定
    OpenAI 的閘道，然後透過設定編輯啟用隨附的頻道／外掛。
  - 驗證設定探索程序會讓未設定且可下載的外掛
    保持不存在；第一次設定後的 doctor 修復會明確安裝每個缺少的
    可下載外掛；第二次重新啟動則不會執行
    隱藏的相依性修復。
  - 也會安裝已知的舊版 npm 基準版本，在執行
    `openclaw update --tag <candidate>` 前啟用 Telegram，並驗證
    候選版本更新後的 doctor 會清除舊版外掛相依性殘留，
    而不需要測試框架端的 postinstall 修復。
- `pnpm test:parallels:npm-update`
  - 在各個 Parallels 客體上執行原生封裝安裝更新冒煙測試。
    每個所選平台會先安裝指定的基準套件，
    接著在同一客體中執行已安裝的 `openclaw update` 命令，並
    驗證已安裝版本、更新狀態、閘道就緒狀態，以及
    一次本機代理程式回合。
  - 迭代單一客體時，請使用 `--platform macos`、`--platform windows` 或 `--platform linux`。
    使用 `--json` 取得摘要成品
    路徑和各路徑狀態。
  - OpenAI 路徑預設使用 `openai/gpt-5.6-luna` 進行即時代理程式回合證明。
    傳入 `--model <provider/model>` 或設定
    `OPENCLAW_PARALLELS_OPENAI_MODEL`，以驗證另一個 OpenAI 模型。
  - 請使用主機逾時包裝長時間的本機執行，避免 Parallels 傳輸停滯
    耗盡剩餘的測試時間：

    ```bash
    timeout --foreground 150m pnpm test:parallels:npm-update -- --json
    timeout --foreground 90m pnpm test:parallels:npm-update -- --platform windows --json
    ```

  - 此指令碼會在
    `/tmp/openclaw-parallels-npm-update.*` 下寫入巢狀路徑記錄。在假設外層
    包裝程式已停滯前，請先檢查 `windows-update.log`、
    `macos-update.log` 或 `linux-update.log`。
  - 在冷啟動客體上，Windows 更新可能會花費 10 到 15 分鐘執行更新後的 doctor 和
    套件更新工作；只要巢狀 npm 偵錯記錄仍持續推進，
    就仍屬正常。
  - 請勿將此彙總包裝程式與個別 Parallels
    macOS、Windows 或 Linux 冒煙測試路徑平行執行。它們共用 VM 狀態，可能會在
    快照還原、套件供應或客體閘道狀態上
    發生衝突。
  - 更新後的證明會執行一般的隨附外掛介面，因為
    語音、影像生成和媒體
    理解等功能外觀，即使代理程式
    回合本身只檢查簡單文字回應，也會透過隨附的執行階段 API 載入。

- `pnpm openclaw qa aimock`
  - 僅啟動本機 AIMock 提供者伺服器，以直接進行通訊協定冒煙
    測試。
- `pnpm openclaw qa matrix`
  - 針對由一次性 Docker 支援的 Tuwunel
    主伺服器執行 Matrix 即時 QA 測試道。僅限原始碼簽出——封裝安裝不會隨附
    `qa-lab`。
  - 完整的命令列介面、設定檔／情境目錄、環境變數與成品配置：
    [Matrix 冒煙測試道](/zh-TW/concepts/qa-e2e-automation#matrix-smoke-lanes)。
- `pnpm openclaw qa telegram`
  - 使用環境中的驅動程式與受測系統機器人權杖，針對真實私人群組
    執行 Telegram 即時 QA 測試道。
  - 需要 `OPENCLAW_QA_TELEGRAM_GROUP_ID`、
    `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` 和
    `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`。群組 ID 必須是數字格式的
    Telegram 聊天 ID。
  - 支援透過 `--credential-source convex` 使用共用集區認證資訊。
    預設使用環境模式，或設定 `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`
    以選擇使用集區租約。
  - 預設涵蓋金絲雀測試、提及閘控、命令定址、`/status`、
    機器人間的提及回覆，以及核心原生命令回覆。
    `mock-openai` 預設也涵蓋確定性的回覆鏈與
    Telegram 最終訊息串流迴歸。使用 `--list-scenarios`
    執行選用探測，例如 `session_status`。
  - 任何情境失敗時會以非零狀態結束。使用 `--allow-failures`
    產生成品而不傳回失敗結束代碼。
  - 需要同一個私人群組中的兩個不同機器人，且受測系統機器人須
    公開 Telegram 使用者名稱。
  - 為了穩定觀察機器人間的互動，請在 `@BotFather` 中為兩個機器人啟用 Bot-to-Bot Communication Mode，並確保驅動程式機器人能觀察
    群組中的機器人流量。
  - 在 `.artifacts/qa-e2e/...` 下寫入 Telegram QA 報告、摘要與
    `qa-evidence.json`。回覆情境包含從驅動程式傳送
    要求到觀察到受測系統回覆的 RTT。

`Mantis Telegram Live` 是此測試道的 PR 證據包裝器。它會使用從 Convex 租用的 Telegram 認證資訊執行
候選參照，在 Crabbox 桌面瀏覽器中呈現經遮蔽處理的
QA 報告／證據套件、錄製 MP4 證據、產生經動作裁剪的 GIF、上傳成品套件，並在設定
`pr_number` 時透過 Mantis GitHub App 發布行內 PR 證據。維護者可以透過 `Mantis Scenario`
（`scenario_id: telegram-live`）從 Actions UI 啟動，或直接透過 PR 留言啟動：

```text
@openclaw-mantis telegram
@openclaw-mantis telegram scenario=telegram-status-command
@openclaw-mantis telegram scenarios=telegram-status-command,channel-canary
```

`Mantis Telegram Desktop Proof` 是用於 PR 視覺證明的代理式原生 Telegram Desktop
前後比較包裝器。可從 Actions UI 使用自由格式的 `instructions` 啟動、
透過 `Mantis Scenario`（`scenario_id:
telegram-desktop-proof`）啟動，或從 PR 留言啟動：

```text
@openclaw-mantis telegram desktop proof
```

Mantis 代理會讀取 PR、判斷哪些 Telegram 可見行為能證明
該變更、在基準與候選參照上執行真實使用者的 Crabbox Telegram Desktop 證明測試道、
反覆調整直到原生 GIF 足以使用、寫入成對的 `motionPreview` 資訊清單，並在設定
`pr_number` 時透過 Mantis GitHub App 發布相同的雙欄 GIF
表格。

- `pnpm openclaw qa mantis telegram-desktop-builder`
  - 租用或重複使用 Crabbox Linux 桌面、安裝原生 Telegram
    Desktop、使用租用的 Telegram 受測系統機器人權杖設定 OpenClaw、
    啟動閘道，並從可見的 VNC 桌面錄製螢幕截圖／MP4 證據。
  - 預設為 `--credential-source convex`，因此工作流程只需要
    Convex 經紀服務密鑰。使用 `--credential-source env` 時，採用與
    `pnpm openclaw qa telegram` 相同的 `OPENCLAW_QA_TELEGRAM_*` 變數。
  - Telegram Desktop 仍需要使用者登入／設定檔。機器人權杖
    僅用於設定 OpenClaw。可使用 `--telegram-profile-archive-env <name>`
    提供 base64 `.tgz` 設定檔封存檔，或使用 `--keep-lease` 並透過 VNC
    手動登入一次。
  - 在輸出目錄下寫入 `mantis-telegram-desktop-builder-report.md`、
    `mantis-telegram-desktop-builder-summary.json`、
    `telegram-desktop-builder.png` 和 `telegram-desktop-builder.mp4`。

即時傳輸測試道共用一套標準契約，避免新增傳輸實作產生
偏差；各測試道的涵蓋範圍矩陣位於
[QA 概觀——即時傳輸涵蓋範圍](/zh-TW/concepts/qa-e2e-automation#live-transport-coverage)。
`qa-channel` 是廣泛的合成測試套件，不屬於該矩陣。

### 透過 Convex 共用 Telegram 認證資訊（v1）

為即時傳輸 QA 啟用 `--credential-source convex`（或 `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`）
時，QA 實驗室會從由 Convex 支援的集區取得獨占租約、在測試道執行期間
對該租約傳送心跳偵測，並於關閉時釋放租約。此章節名稱早於 Discord、Slack 和
WhatsApp 支援；各種類型共用相同的租約契約。

參考 Convex 專案鷹架：`qa/convex-credential-broker/`

必要的環境變數：

- `OPENCLAW_QA_CONVEX_SITE_URL`（例如 `https://your-deployment.convex.site`）
- 所選角色的一個密鑰：
  - `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`，用於 `maintainer`
  - `OPENCLAW_QA_CONVEX_SECRET_CI`，用於 `ci`
- 認證資訊角色選擇：
  - 命令列介面：`--credential-role maintainer|ci`
  - 環境預設值：`OPENCLAW_QA_CREDENTIAL_ROLE`（在 CI 中預設為 `ci`，否則為 `maintainer`）

選用的環境變數：

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS`（預設為 `1200000`）
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS`（預設為 `30000`）
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS`（預設為 `90000`）
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS`（預設為 `15000`）
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX`（預設為 `/qa-credentials/v1`）
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID`（選用的追蹤 ID）
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` 允許僅供本機開發使用的回送 `http://` Convex URL。

正常運作時，`OPENCLAW_QA_CONVEX_SITE_URL` 應使用 `https://`。

維護者管理命令（新增／移除／列出集區）明確需要
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`。

維護者適用的命令列介面輔助工具：

```bash
pnpm openclaw qa credentials doctor
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

即時執行前使用 `doctor` 檢查 Convex 站台 URL、經紀服務密鑰、
端點前置詞、HTTP 逾時，以及管理／清單可達性，且不會印出
密鑰值。在指令碼與 CI 公用程式中使用 `--json` 取得機器可讀的輸出。

預設端點契約（`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`）。
要求使用 `Authorization: Bearer <role secret>` 標頭進行驗證；
以下本文省略該標頭：

- `POST /acquire`
  - 要求：`{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - 成功：`{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - 已耗盡／可重試：`{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /payload-chunk`
  - 要求：`{ kind, ownerId, actorRole, credentialId, leaseToken, index }`
  - 成功：`{ status: "ok", index, data }`
- `POST /heartbeat`
  - 要求：`{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - 成功：`{ status: "ok" }`（或空的 `2xx`）
- `POST /release`
  - 要求：`{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - 成功：`{ status: "ok" }`（或空的 `2xx`）
- `POST /admin/add`（僅限維護者密鑰）
  - 要求：`{ kind, actorId, payload, note?, status? }`
  - 成功：`{ status: "ok", credential }`
- `POST /admin/remove`（僅限維護者密鑰）
  - 要求：`{ credentialId, actorId }`
  - 成功：`{ status: "ok", changed, credential }`
  - 有效租約防護：`{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list`（僅限維護者密鑰）
  - 要求：`{ kind?, status?, includePayload?, limit? }`
  - 成功：`{ status: "ok", credentials, count }`

Telegram 類型的承載資料形狀：

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` 必須是數字格式的 Telegram 聊天 ID 字串。
- `admin/add` 會針對 `kind: "telegram"` 驗證此形狀，並拒絕格式錯誤的承載資料。

Telegram 真實使用者類型的承載資料形狀：

- `{ groupId: string, sutToken: string, testerUserId: string, testerUsername: string, telegramApiId: string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string, tdlibArchiveBase64: string, tdlibArchiveSha256: string, desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }`
- `groupId`、`testerUserId` 和 `telegramApiId` 必須是數字字串。
- `tdlibArchiveSha256` 和 `desktopTdataArchiveSha256` 必須是 SHA-256 十六進位字串。
- `kind: "telegram-user"` 保留供 Mantis Telegram Desktop 證明工作流程使用。一般 QA 實驗室測試道不得取得它。

由經紀服務驗證的多頻道承載資料：

- Discord：`{ guildId: string, channelId: string, driverBotToken: string, sutBotToken: string, sutApplicationId: string, voiceChannelId?: string }`
- WhatsApp：`{ driverPhoneE164: string, sutPhoneE164: string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string, groupJid?: string }`

Slack 測試道也能從集區租用，但 Slack 承載資料驗證
目前位於 Slack QA 執行器，而非經紀服務中。Slack 資料列請使用
`{ channelId: string, driverBotToken: string, sutBotToken: string, sutAppToken: string }`。

### 將頻道新增至 QA

新頻道配接器的架構與情境輔助工具名稱位於
[QA 概觀——新增頻道](/zh-TW/concepts/qa-e2e-automation#adding-a-channel)。
最低要求：在共用的 `qa-lab` 主機接縫上實作傳輸執行器、
為共用情境新增 `adapterFactory`、在外掛資訊清單中宣告 `qaRunners`、
掛載為 `openclaw qa <runner>`，並在 `qa/scenarios/` 下撰寫情境。

## 測試套件（各自在哪裡執行）

可將這些套件視為“逐步提高真實程度”（同時提高不穩定性／成本）。

### 單元／整合（預設）

- 命令：`pnpm test`
- 設定：無目標的執行使用 `vitest.full-*.config.ts` 分片集，並可能
  將多專案分片展開為各專案設定，以進行平行
  排程
- 檔案：核心／單元清單位於 `src/**/*.test.ts`、
  `packages/**/*.test.ts` 和 `test/**/*.test.ts` 下；UI 單元測試在
  專用的 `unit-ui` 分片中執行
- 範圍：
  - 純單元測試
  - 處理程序內整合測試（閘道驗證、路由、工具、剖析、設定）
  - 已知錯誤的確定性迴歸測試
- 預期：
  - 在 CI 中執行
  - 不需要真實金鑰
  - 應快速且穩定
  - 解析器與公開介面載入器測試必須使用產生的小型外掛固定資料，證明廣泛的 `api.js` 和
    `runtime-api.js` 備援行為，
    而非真實的內建外掛原始碼 API。真實的外掛 API 載入應放在
    外掛自有的契約／整合套件中。

原生相依性政策：

- 預設測試安裝會略過選用的原生 Discord opus 建置。Discord
  語音使用隨附的 `libopus-wasm`，且 `@discordjs/opus` 在
  `allowBuilds` 中維持停用，使本機測試與 Testbox 測試道不會編譯原生
  附加元件。
- 請在 `libopus-wasm` 基準測試存放庫中比較原生 opus 效能，而非
  在預設 OpenClaw 安裝／測試迴圈中進行。請勿在預設的 `allowBuilds` 中將 `@discordjs/opus` 設為
  `true`；這會使無關的安裝／測試
  迴圈編譯原生程式碼。

<AccordionGroup>
  <Accordion title="專案、分片與範圍限定測試道">

    - 未指定目標的 `pnpm test` 會執行十三個較小的分片設定（`core-unit-fast`、`core-unit-src`、`core-unit-security`、`core-unit-ui`、`core-unit-support`、`core-support-boundary`、`core-tooling`、`core-contracts`、`core-bundled`、`core-runtime`、`agentic`、`auto-reply`、`extensions`），而非單一龐大的原生根專案程序。這能降低高負載機器上的 RSS 峰值，並避免自動回覆／外掛工作使不相關的測試套件資源不足。
    - `pnpm test --watch` 仍使用原生根 `vitest.config.ts` 專案圖，因為多分片監看迴圈並不實際。
    - `pnpm test`、`pnpm test:watch` 和 `pnpm test:perf:imports` 會先透過範圍限定的執行區路由明確的檔案／目錄目標，因此 `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` 可免除完整根專案的啟動成本。
    - `pnpm test:changed` 預設會將變更的 Git 路徑展開至低成本的範圍限定執行區：直接測試編輯、同層 `*.test.ts` 檔案、明確的來源對應，以及本機匯入圖中的相依項目。除非你明確使用 `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed`，否則設定／建置／套件編輯不會廣泛執行測試。
    - `pnpm check:changed` 是窄範圍工作的標準智慧型本機檢查關卡。它會將差異分類為核心、核心測試、擴充功能、擴充功能測試、應用程式、文件、發行中繼資料、即時 Docker 工具和工具，然後執行相符的型別檢查、lint 與防護命令。它不會執行 Vitest 測試；若要提供測試證明，請呼叫 `pnpm test:changed` 或明確的 `pnpm test <target>`。僅有發行中繼資料的版本遞增會執行針對性的版本／設定／根相依套件檢查，並以防護機制拒絕頂層版本欄位以外的套件變更。
    - 即時 Docker ACP 測試框架編輯會執行聚焦檢查：即時 Docker 驗證指令碼的 shell 語法，以及即時 Docker 排程器的試執行。只有當差異僅限於 `scripts["test:docker:live-*"]` 時，才會納入 `package.json` 變更；相依套件、匯出、版本及其他套件表面編輯仍會使用範圍較廣的防護機制。
    - 來自代理程式、命令、外掛、自動回覆輔助程式、`plugin-sdk` 和類似純工具區域的低匯入量單元測試，會透過 `unit-fast` 執行區路由，並略過 `test/setup-openclaw-runtime.ts`；有狀態／高執行階段負載的檔案則留在現有執行區。
    - 選定的 `plugin-sdk` 和 `commands` 輔助程式來源檔案，也會將變更模式執行對應到這些輕量執行區中的明確同層測試，因此輔助程式編輯不必重新執行該目錄的完整高負載測試套件。
    - `auto-reply` 針對頂層核心輔助程式、頂層 `reply.*` 整合測試和 `src/auto-reply/reply/**` 子樹各設有專用分組。CI 會進一步將回覆子樹拆分為代理程式執行器、分派，以及命令／狀態路由分片，避免單一高匯入量分組占用整個節點尾端。
    - 一般 PR／main CI 會刻意略過隨附外掛批次掃描和僅供發行使用的 `agentic-plugins` 分片。完整發行驗證會針對候選發行版本上的這些高外掛負載測試套件，分派獨立的 `Plugin Prerelease` 子工作流程。

  </Accordion>

  <Accordion title="內嵌執行器涵蓋範圍">

    - 變更訊息工具探索輸入或壓縮執行階段
      上下文時，請維持兩個層級的涵蓋範圍。
    - 針對純路由和正規化
      邊界新增聚焦的輔助程式迴歸測試。
    - 維持內嵌執行器整合測試套件的健全狀態：
      `src/agents/embedded-agent-runner/compact.hooks.test.ts`、
      `src/agents/embedded-agent-runner/run.overflow-compaction.test.ts` 和
      `src/agents/embedded-agent-runner/run.overflow-compaction.loop.test.ts`。
    - 這些測試套件會驗證範圍限定 ID 和壓縮行為仍會流經
      真正的 `run.ts`／`compact.ts` 路徑；僅有輔助程式的測試
      無法充分取代這些整合路徑。

  </Accordion>

  <Accordion title="Vitest 集區與隔離預設值">

    - 基礎 Vitest 設定預設為 `threads`。
    - 共用 Vitest 設定會固定 `isolate: false`，並在根專案、端對端和即時設定中
      使用非隔離執行器。
    - 根 UI 執行區會保留其 `jsdom` 設定與最佳化器，但也會在
      共用非隔離執行器上執行。
    - 每個 `pnpm test` 分片都會從共用 Vitest 設定繼承相同的 `threads` + `isolate: false`
      預設值。
    - `scripts/run-vitest.mjs` 預設會為 Vitest 子節點
      程序新增 `--no-maglev`，以減少大型本機執行期間的 V8 編譯耗損。
      設定 `OPENCLAW_VITEST_ENABLE_MAGLEV=1` 可與標準 V8
      行為比較。
    - `scripts/run-vitest.mjs` 會在明確的非監看 Vitest 執行
      連續 5 分鐘沒有標準輸出或標準錯誤輸出後終止執行。若要進行刻意保持靜默的調查，
      請設定 `OPENCLAW_VITEST_NO_OUTPUT_TIMEOUT_MS=0` 以停用
      監控程序。

  </Accordion>

  <Accordion title="快速本機反覆運算">

    - `pnpm changed:lanes` 會顯示差異觸發哪些架構執行區。
    - 預先提交鉤子僅執行格式化。它會重新暫存已格式化的檔案，
      且不會執行 lint、型別檢查或測試。
    - 需要智慧型本機檢查關卡時，請在交接或推送前明確執行
      `pnpm check:changed`。
    - `pnpm test:changed` 預設透過低成本的範圍限定執行區路由。只有當代理程式
      判定測試框架、設定、套件或合約編輯確實需要
      更廣泛的 Vitest 涵蓋範圍時，才使用 `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed`。
    - `pnpm test:max` 和 `pnpm test:changed:max` 會維持相同的路由
      行為，但工作程序上限較高。
    - 本機工作程序自動調整刻意採取保守策略，並會在主機平均負載已偏高時
      降低負載，因此多個並行
      Vitest 執行預設會降低影響。
    - 基礎 Vitest 設定會將專案／設定檔標記為
      `forceRerunTriggers`，確保測試
      配線變更時，變更模式的重新執行仍然正確。
    - 此設定會在支援的主機上保持啟用
      `OPENCLAW_VITEST_FS_MODULE_CACHE`；設定 `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path`
      可為直接效能分析指定一個明確的快取位置。

  </Accordion>

  <Accordion title="效能偵錯">

    - `pnpm test:perf:imports` 會啟用 Vitest 匯入持續時間報告及
      匯入明細輸出。
    - `pnpm test:perf:imports:changed` 會將相同的效能分析檢視範圍限定為
      自 `origin/main` 起變更的檔案。
    - 分片計時資料會寫入 `.artifacts/vitest-shard-timings.json`。
      完整設定執行會使用設定路徑作為索引鍵；包含模式 CI
      分片會附加分片名稱，以便個別追蹤經篩選的分片。
    - 當某個高負載測試仍將大部分時間耗費在啟動匯入時，
      請將高負載相依套件放在狹窄的本機 `*.runtime.ts` 接縫後方，並
      直接模擬該接縫，而不是深層匯入執行階段輔助程式，
      只為透過 `vi.mock(...)` 傳遞它們。
    - `pnpm test:perf:changed:bench -- --ref <git-ref>` 會針對該
      已提交差異，比較經路由的 `test:changed` 與原生根專案路徑，
      並輸出實際經過時間及 macOS RSS 上限。
    - `pnpm test:perf:changed:bench -- --worktree` 會將變更的檔案清單透過
      `scripts/test-projects.mjs` 和根 Vitest 設定路由，以評測目前的
      工作目錄髒狀態。
    - `pnpm test:perf:profile:main` 會為
      Vitest／Vite 啟動與轉換額外負荷寫入主執行緒 CPU 效能分析。
    - `pnpm test:perf:profile:runner` 會在停用檔案平行處理的情況下，
      為單元測試套件寫入執行器 CPU + 堆積效能分析。

  </Accordion>
</AccordionGroup>

### 穩定性（閘道）

- 命令：`pnpm test:stability:gateway`
- 設定：`test/vitest/vitest.gateway.config.ts`、`test/vitest/vitest.logging.config.ts` 和 `test/vitest/vitest.infra.config.ts`，各自強制使用一個工作程序
- 範圍：
  - 啟動真正的迴路介面閘道，並預設啟用診斷
  - 透過診斷事件路徑驅動合成的閘道訊息、記憶體和大型酬載反覆變動
  - 透過閘道 WS RPC 查詢 `diagnostics.stability`
  - 涵蓋診斷穩定性套件持續性輔助程式
  - 斷言記錄器維持在界限內、合成 RSS 樣本保持低於壓力預算，且各工作階段的佇列深度會排空並回到零
- 預期：
  - 可安全用於 CI，且不需要金鑰
  - 供穩定性迴歸後續處理使用的窄範圍執行區，不能取代完整的閘道測試套件

### 端對端（儲存庫彙總）

- 命令：`pnpm test:e2e`
- 範圍：
  - 執行閘道冒煙端對端執行區
  - 執行模擬 Control UI 瀏覽器端對端執行區
- 預期：
  - 可安全用於 CI，且不需要金鑰
  - 需要安裝 Playwright Chromium

### 端對端（閘道冒煙測試）

- 命令：`pnpm test:e2e:gateway`
- 設定：`test/vitest/vitest.e2e.config.ts`
- 檔案：`src/**/*.e2e.test.ts`、`test/**/*.e2e.test.ts`，以及 `extensions/` 下的隨附外掛端對端測試
- 執行階段預設值：
  - 使用 Vitest `threads` 搭配 `isolate: false`，與儲存庫其餘部分一致。
  - 使用自適應工作程序（CI：最多 2 個，本機：預設 1 個）。
  - 預設以靜默模式執行，以降低主控台 I/O 額外負荷。
- 實用覆寫：
  - `OPENCLAW_E2E_WORKERS=<n>` 可強制指定工作程序數量（上限為 16）。
  - `OPENCLAW_E2E_VERBOSE=1` 可重新啟用詳細主控台輸出。
- 範圍：
  - 多執行個體閘道的端對端行為
  - WebSocket／HTTP 介面、節點配對和較高負載的網路功能
- 預期：
  - 在 CI 中執行（當流水線中啟用時）
  - 不需要真實金鑰
  - 比單元測試涉及更多活動元件（可能較慢）

### 端對端（Control UI 模擬瀏覽器）

- 命令：`pnpm test:ui:e2e`
- 設定：`test/vitest/vitest.ui-e2e.config.ts`
- 檔案：`ui/src/**/*.e2e.test.ts`
- 範圍：
  - 啟動 Vite Control UI
  - 透過 Playwright 驅動真正的 Chromium 頁面
  - 以確定性的瀏覽器內模擬取代閘道 WebSocket
- 預期：
  - 在 CI 中作為 `pnpm test:e2e` 的一部分執行
  - 不需要真正的閘道、代理程式或供應商金鑰
  - 必須存在瀏覽器相依套件（`pnpm --dir ui exec playwright install chromium`）

### 端對端：OpenShell 後端冒煙測試

- 命令：`pnpm test:e2e:openshell`
- 檔案：`extensions/openshell/src/backend.e2e.test.ts`
- 範圍：
  - 重複使用作用中的本機 OpenShell 閘道
  - 從暫存本機 Dockerfile 建立沙箱
  - 透過真正的 `sandbox ssh-config` + SSH exec 操作 OpenClaw 的 OpenShell 後端
  - 透過沙箱檔案系統橋接器驗證以遠端為準的檔案系統行為
- 預期：
  - 僅供選擇性啟用；不屬於預設 `pnpm test:e2e` 執行的一部分
  - 需要本機 `openshell` 命令列介面及正常運作的 Docker 常駐程式
  - 需要作用中的本機 OpenShell 閘道及其設定來源
  - 使用隔離的 `HOME`／`XDG_CONFIG_HOME`，然後銷毀測試沙箱
- 實用覆寫：
  - `OPENCLAW_E2E_OPENSHELL=1` 可在手動執行較廣泛的端對端測試套件時啟用此測試
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` 可指向非預設的命令列介面二進位檔或包裝指令碼
  - `OPENCLAW_E2E_OPENSHELL_CONFIG_HOME=/path/to/config` 可向隔離測試公開已註冊的閘道設定
  - `OPENCLAW_E2E_OPENSHELL_HOST_IP=172.18.0.1` 可覆寫主機原則測試資料使用的 Docker 閘道 IP

### 即時（真實供應商 + 真實模型）

- 命令：`pnpm test:live`
- 設定：`test/vitest/vitest.live.config.ts`
- 檔案：`src/**/*.live.test.ts`、`test/**/*.live.test.ts`，以及 `extensions/` 下的內建外掛即時測試
- 預設：由 `pnpm test:live` **啟用**（設定 `OPENCLAW_LIVE_TEST=1`）
- 範圍：
  - 「這個供應商／模型目前使用真實認證資訊真的能運作嗎？」
  - 找出供應商格式變更、工具呼叫的特殊行為、驗證問題及速率限制行為
- 預期：
  - 設計上不保證在 CI 中穩定（真實網路、真實供應商政策、配額及服務中斷）
  - 會產生費用／使用速率限制額度
  - 建議執行縮小範圍的子集，而非「全部」
- 即時執行會使用已匯出的 API 金鑰及預先建立的驗證設定檔。
- 依預設，即時執行仍會隔離 `HOME`，並將設定／驗證資料複製到暫存測試家目錄，避免單元測試固定資料修改你實際的 `~/.openclaw`。
- 只有在刻意需要即時測試使用實際家目錄時，才設定 `OPENCLAW_LIVE_USE_REAL_HOME=1`。
- `pnpm test:live` 預設採用較安靜的模式：保留 `[live] ...` 進度輸出，並將閘道啟動記錄／Bonjour 訊息靜音。若要恢復完整啟動記錄，請設定 `OPENCLAW_LIVE_TEST_QUIET=0`。
- API 金鑰輪替（依供應商而異）：以逗號／分號格式設定 `*_API_KEYS`，或設定 `*_API_KEY_1`、`*_API_KEY_2`（例如 `OPENAI_API_KEYS`、`ANTHROPIC_API_KEYS`、`GEMINI_API_KEYS`），也可透過 `OPENCLAW_LIVE_*_KEY` 為每次即時執行覆寫；測試遇到速率限制回應時會重試。
- 進度／心跳偵測輸出：
  - 即時測試套件會將進度行輸出至 stderr，因此即使 Vitest 主控台擷取沒有輸出，也能看出耗時的供應商呼叫仍在執行。
  - `test/vitest/vitest.live.config.ts` 會停用 Vitest 主控台攔截，讓供應商／閘道進度行在即時執行期間立即串流顯示。
  - 使用 `OPENCLAW_LIVE_HEARTBEAT_MS` 調整直接模型的心跳偵測。
  - 使用 `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` 調整閘道／探測的心跳偵測。

## 我應該執行哪個測試套件？

請使用此決策表：

- 編輯邏輯／測試：執行 `pnpm test`（若變更很多，也執行 `pnpm test:coverage`）
- 變更閘道網路／WS 通訊協定／配對：加入 `pnpm test:e2e`
- 偵錯「我的機器人離線了」／特定供應商失敗／工具呼叫：執行縮小範圍的 `pnpm test:live`

## 即時（會使用網路）測試

關於即時模型矩陣、命令列介面後端冒煙測試、ACP 冒煙測試、Codex app-server
測試框架，以及所有媒體供應商即時測試（Deepgram、BytePlus、ComfyUI、
圖片、音樂、影片、媒體測試框架），還有即時執行的認證資訊處理

- 請參閱[測試即時測試套件](/zh-TW/help/testing-live)。如需專門的更新與
  外掛驗證檢查清單，請參閱
  [測試更新與外掛](/zh-TW/help/testing-updates-plugins)。

## Docker 執行器（選用的「可在 Linux 運作」檢查）

這些 Docker 執行器分為兩類：

- 即時模型執行器：`test:docker:live-models` 與 `test:docker:live-gateway` 只會在儲存庫 Docker 映像檔（`src/agents/models.profiles.live.test.ts` 與 `src/gateway/gateway-models.profiles.live.test.ts`）內執行各自相符的設定檔金鑰即時檔案，並掛載你的本機設定目錄、工作區及選用的設定檔環境變數檔案。對應的本機進入點為 `test:live:models-profiles` 與 `test:live:gateway-profiles`。
- Docker 即時執行器會視需要保留各自實用的上限：
  `test:docker:live-models` 預設使用精選且受支援的高訊號集合，而
  `test:docker:live-gateway` 預設使用 `OPENCLAW_LIVE_GATEWAY_SMOKE=1`、
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`、
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` 及
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`。只有在明確需要較小上限或較大掃描範圍時，才設定 `OPENCLAW_LIVE_MAX_MODELS`
  或閘道環境變數。
- `test:docker:all` 會透過 `test:docker:live-build` 建置即時 Docker 映像檔一次，透過 `scripts/package-openclaw-for-docker.mjs` 將 OpenClaw 封裝成 npm tarball 一次，然後建置／重複使用兩個 `scripts/e2e/Dockerfile` 映像檔。基礎映像檔只作為安裝／更新／外掛相依套件測試通道的 Node/Git 執行器；這些測試通道會掛載預先建置的 tarball。功能映像檔會將同一個 tarball 安裝至 `/app`，供已建置應用程式功能測試通道使用。Docker 測試通道定義位於 `scripts/lib/docker-e2e-scenarios.mjs`；規劃器邏輯位於 `scripts/lib/docker-e2e-plan.mjs`；`scripts/test-docker-all.mjs` 會執行選取的計畫。彙總執行器使用加權本機排程器：`OPENCLAW_DOCKER_ALL_PARALLELISM` 控制程序時槽，而資源上限會避免高負載即時、npm 安裝及多服務測試通道同時全部啟動。若單一測試通道的負載高於目前上限，排程器仍可在資源池為空時啟動該通道，並讓它單獨執行，直到再次有可用容量。預設值為 10 個時槽、`OPENCLAW_DOCKER_ALL_LIVE_LIMIT=9`、`OPENCLAW_DOCKER_ALL_NPM_LIMIT=5` 及 `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT=7`；只有在 Docker 主機有更多餘裕時，才調整 `OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` 或 `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT`（以及其他 `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT` 覆寫值）。執行器預設會執行 Docker 前置檢查、移除過時的 OpenClaw E2E 容器、每 30 秒輸出狀態、將成功測試通道的耗時儲存在 `.artifacts/docker-tests/lane-timings.json`，並在後續執行時使用這些耗時資料優先啟動較久的測試通道。使用 `OPENCLAW_DOCKER_ALL_DRY_RUN=1` 可在不建置或執行 Docker 的情況下輸出加權測試通道清單；使用 `node scripts/test-docker-all.mjs --plan-json` 可輸出所選測試通道、套件／映像檔需求及認證資訊的 CI 計畫。
- `Package Acceptance` 是 GitHub 原生套件閘門，用於確認「這個可安裝的 tarball 能否作為產品正常運作？」它會從 `source=npm`、`source=ref`、`source=url`、`source=trusted-url` 或 `source=artifact` 解析一個候選套件，將其上傳為 `package-under-test`，然後針對該確切 tarball 執行可重複使用的 Docker E2E 測試通道，而不是重新封裝所選參照。設定檔依涵蓋範圍排序：`smoke`、`package`、`product` 及 `full`（另有 `custom` 可用於明確的測試通道清單）。關於套件／更新／外掛契約、已發布升級的存續矩陣、發布預設值及失敗分流，請參閱[測試更新與外掛](/zh-TW/help/testing-updates-plugins)。
- 建置與發布檢查會在 tsdown 後執行 `scripts/check-cli-bootstrap-imports.mjs`。此防護會從 `dist/entry.js` 與 `dist/cli/run-main.js` 遍歷靜態建置圖；若該分派前啟動圖在命令分派前靜態匯入任何外部套件（Commander、提示 UI、undici、記錄功能及類似的高啟動負載相依套件都包含在內），檢查就會失敗；它也會將內建閘道執行區塊限制在 70 KB，並禁止該區塊靜態匯入已知的冷閘道路徑（`control-ui-assets`、`diagnostic-stability-bundle`、`onboard-helpers`、`process-respawn`、`restart-sentinel`、`server-close`、`server-reload-handlers`）。`scripts/release-check.ts` 會另外使用 `--help`、`onboard --help`、`doctor --help`、`status --json --timeout 1`、`config schema` 及 `models list --provider openai` 對已封裝的命令列介面執行冒煙測試。
- 套件驗收的舊版相容性上限為 `2026.4.25`（包含 `2026.4.25-beta.*`）。在此截止版本以前，測試框架只容許已發布套件的中繼資料缺漏：省略私有 QA 清單項目、缺少 `gateway install --wrapper`、衍生自 tarball 的 git 固定資料缺少修補檔案、缺少持久化的 `update.channel`、舊版外掛安裝記錄位置、缺少市集安裝記錄持久化，以及在 `plugins update` 期間進行設定中繼資料遷移。對於 `2026.4.25` 之後的套件，這些情況都會造成嚴格失敗。
- 容器冒煙測試執行器：`test:docker:openwebui`、`test:docker:onboard`、`test:docker:npm-onboard-channel-agent`、`test:docker:release-user-journey`、`test:docker:release-typed-onboarding`、`test:docker:release-media-memory`、`test:docker:release-upgrade-user-journey`、`test:docker:release-plugin-marketplace`、`test:docker:skill-install`、`test:docker:update-channel-switch`、`test:docker:upgrade-survivor`、`test:docker:published-upgrade-survivor`、`test:docker:session-runtime-context`、`test:docker:agents-delete-shared-workspace`、`test:docker:gateway-network`、`test:docker:browser-cdp-snapshot`、`test:docker:mcp-channels`、`test:docker:agent-bundle-mcp-tools`、`test:docker:cron-mcp-cleanup`、`test:docker:plugins`、`test:docker:plugin-update`、`test:docker:plugin-lifecycle-matrix` 及 `test:docker:config-reload` 會啟動一或多個真實容器，並驗證較高階的整合路徑。
- 透過 `scripts/lib/openclaw-e2e-instance.sh` 安裝已封裝 OpenClaw tarball 的 Docker/Bash E2E 測試通道，會將 `npm install` 限制為 `OPENCLAW_E2E_NPM_INSTALL_TIMEOUT`（預設為 `600s`；設定 `0` 可停用包裝器以進行偵錯）。

即時模型 Docker 執行器也只會繫結掛載所需的命令列介面驗證家目錄
（若執行範圍未縮小，則掛載所有支援的家目錄），接著在執行前將它們複製到
容器家目錄，使外部命令列介面 OAuth 能重新整理權杖，
而不會修改主機的驗證儲存區：

- 直接模型：`pnpm test:docker:live-models`（指令碼：`scripts/test-live-models-docker.sh`）
- ACP 繫結冒煙測試：`pnpm test:docker:live-acp-bind`（指令碼：`scripts/test-live-acp-bind-docker.sh`；預設涵蓋 Claude、Codex 及 Gemini，並透過 `pnpm test:docker:live-acp-bind:droid` 與 `pnpm test:docker:live-acp-bind:opencode` 嚴格涵蓋 Droid/OpenCode）
- 命令列介面後端冒煙測試：`pnpm test:docker:live-cli-backend`（指令碼：`scripts/test-live-cli-backend-docker.sh`）
- Codex app-server 測試框架冒煙測試：`pnpm test:docker:live-codex-harness`（指令碼：`scripts/test-live-codex-harness-docker.sh`）
- 閘道 + 開發代理程式：`pnpm test:docker:live-gateway`（指令碼：`scripts/test-live-gateway-models-docker.sh`）
- 可觀測性冒煙測試：`pnpm qa:otel:smoke`、`pnpm qa:prometheus:smoke` 及 `pnpm qa:observability:smoke` 是私有 QA 原始碼簽出測試通道。它們刻意不納入套件 Docker 發布測試通道，因為 npm tarball 會省略 QA Lab。
- Open WebUI 即時冒煙測試：`pnpm test:docker:openwebui`（指令碼：`scripts/e2e/openwebui-docker.sh`）
- 新手設定精靈（TTY，完整架構建置）：`pnpm test:docker:onboard`（指令碼：`scripts/e2e/onboard-docker.sh`）
- Npm tarball 新手設定／頻道／代理程式冒煙測試：`pnpm test:docker:npm-onboard-channel-agent` 會在 Docker 中全域安裝已封裝的 OpenClaw tarball，預設透過環境變數參照新手設定來設定 OpenAI 與 Telegram、執行 doctor，並執行一次模擬的 OpenAI 代理程式回合。使用 `OPENCLAW_CURRENT_PACKAGE_TGZ=/path/to/openclaw-*.tgz` 可重複使用預先建置的 tarball，使用 `OPENCLAW_NPM_ONBOARD_HOST_BUILD=0` 可略過主機重新建置，或使用 `OPENCLAW_NPM_ONBOARD_CHANNEL=discord` 或 `OPENCLAW_NPM_ONBOARD_CHANNEL=slack` 切換頻道。

- 發布使用者歷程冒煙測試：`pnpm test:docker:release-user-journey` 會在乾淨的 Docker 主目錄中全域安裝已封裝的 OpenClaw tarball、執行新手引導、設定模擬的 OpenAI 提供者、執行一次代理程式回合、安裝／解除安裝外部外掛、針對本機固定測試資料設定 ClickClack、驗證傳出／傳入訊息、重新啟動閘道，並執行 doctor。
- 發布具型別新手引導冒煙測試：`pnpm test:docker:release-typed-onboarding` 會安裝已封裝的 tarball、透過真實 TTY 操作 `openclaw onboard`、將 OpenAI 設定為環境變數參照提供者、驗證不會保存原始金鑰，並執行一次模擬的代理程式回合。
- 發布媒體／記憶冒煙測試：`pnpm test:docker:release-media-memory` 會安裝已封裝的 tarball、驗證對 PNG 附件的影像理解、OpenAI 相容的影像生成輸出、記憶搜尋回想能力，以及回想能力在閘道重新啟動後仍可保留。
- 發布升級使用者歷程冒煙測試：`pnpm test:docker:release-upgrade-user-journey` 預設會安裝比候選 tarball 舊的最新已發布基準版本、在已發布套件上設定提供者／外掛／ClickClack 狀態、升級至候選 tarball，然後重新執行核心代理程式／外掛／頻道歷程。如果沒有更舊的已發布基準版本，則重複使用候選版本。使用 `OPENCLAW_RELEASE_UPGRADE_BASELINE_SPEC=openclaw@<version>` 覆寫基準版本。
- 發布外掛市集冒煙測試：`pnpm test:docker:release-plugin-marketplace` 會從本機固定測試市集安裝、更新已安裝的外掛、解除安裝該外掛，並驗證外掛命令列介面隨安裝中繼資料一併清除而消失。
- Skill 安裝冒煙測試：`pnpm test:docker:skill-install` 會在 Docker 中全域安裝已封裝的 OpenClaw tarball、在設定中停用上傳封存檔安裝、從搜尋中解析目前線上 ClawHub skill slug、使用 `openclaw skills install` 安裝，並驗證已安裝的 skill 以及 `.clawhub` 來源／鎖定中繼資料。
- 更新頻道切換冒煙測試：`pnpm test:docker:update-channel-switch` 會在 Docker 中全域安裝已封裝的 OpenClaw tarball、從套件 `stable` 切換至 git `dev`、驗證持久保存的頻道及外掛更新後工作，接著切回套件 `stable` 並檢查更新狀態。
- 升級存續冒煙測試：`pnpm test:docker:upgrade-survivor` 會將已封裝的 OpenClaw tarball 安裝至含有代理程式、頻道設定、外掛允許清單、過時外掛相依性狀態，以及現有工作區／工作階段檔案的未清理舊使用者固定測試環境。它會在沒有線上提供者或頻道金鑰的情況下執行套件更新及非互動式 doctor，接著啟動回送閘道，並檢查設定／狀態保留情況以及啟動／狀態時間預算。
- 已發布版本升級存續冒煙測試：`pnpm test:docker:published-upgrade-survivor` 預設會安裝 `openclaw@latest`、植入符合實際情況的現有使用者檔案、使用內建的命令配方設定該基準版本、驗證產生的設定、將該已發布安裝更新至候選 tarball、執行非互動式 doctor、寫入 `.artifacts/upgrade-survivor/summary.json`，然後啟動回送閘道，並檢查已設定的意圖、狀態保留、啟動、`/healthz`、`/readyz` 以及 RPC 狀態時間預算。使用 `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC` 覆寫單一基準版本；使用 `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS`（例如 `openclaw@2026.5.2 openclaw@2026.4.23 openclaw@2026.4.15`）要求彙總排程器展開確切的本機基準版本；使用 `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS`（例如 `reported-issues`）展開以問題為模型的固定測試資料；已回報問題集合包含 `configured-plugin-installs`，用於自動修復外部 OpenClaw 外掛安裝。套件驗收會將這些公開為 `published_upgrade_survivor_baseline`、`published_upgrade_survivor_baselines` 和 `published_upgrade_survivor_scenarios`，解析 `last-stable-4` 或 `all-since-2026.4.23` 等中繼基準權杖，而完整發布驗證會將發布浸泡套件閘門展開為 `last-stable-4 2026.4.23 2026.5.2 2026.4.15` 加上 `reported-issues`。
- 工作階段執行階段內容冒煙測試：`pnpm test:docker:session-runtime-context` 會驗證隱藏執行階段內容的逐字記錄持久保存，以及 doctor 對受影響之重複提示重寫分支的修復。
- Bun 全域安裝冒煙測試：`bash scripts/e2e/bun-global-install-smoke.sh` 會封裝目前的原始碼樹、在隔離的主目錄中使用 `bun install -g` 安裝，並驗證 `openclaw infer image providers --json` 會傳回內附的影像提供者，而不是停滯不動。使用 `OPENCLAW_BUN_GLOBAL_SMOKE_PACKAGE_TGZ=/path/to/openclaw-*.tgz` 重複使用預先建置的 tarball、使用 `OPENCLAW_BUN_GLOBAL_SMOKE_HOST_BUILD=0` 略過主機建置，或使用 `OPENCLAW_BUN_GLOBAL_SMOKE_DIST_IMAGE=openclaw-dockerfile-smoke:local` 從已建置的 Docker 映像複製 `dist/`。
- 安裝程式 Docker 冒煙測試：`bash scripts/test-install-sh-docker.sh` 會在其 root、更新及直接 npm 容器之間共用一個 npm 快取。更新冒煙測試預設使用 npm `latest` 作為穩定基準版本，再升級至候選 tarball。在本機使用 `OPENCLAW_INSTALL_SMOKE_UPDATE_BASELINE=2026.4.22` 覆寫，或在 GitHub 上使用 Install Smoke 工作流程的 `update_baseline_version` 輸入覆寫。非 root 安裝程式檢查會保留隔離的 npm 快取，以免 root 擁有的快取項目掩蓋使用者本機安裝行為。設定 `OPENCLAW_INSTALL_SMOKE_NPM_CACHE_DIR=/path/to/cache`，即可在本機重新執行時重複使用 root／更新／直接 npm 快取。
- Install Smoke CI 會使用 `OPENCLAW_INSTALL_SMOKE_SKIP_NPM_GLOBAL=1` 略過重複的直接 npm 全域更新；需要涵蓋直接 `npm install -g` 時，請在本機執行指令碼且不要設定該環境變數。
- 代理程式刪除共用工作區的命令列介面冒煙測試：`pnpm test:docker:agents-delete-shared-workspace`（指令碼：`scripts/e2e/agents-delete-shared-workspace-docker.sh`）預設會建置根目錄 Dockerfile 映像、在隔離的容器主目錄中植入兩個共用同一工作區的代理程式、執行 `agents delete --json`，並驗證有效的 JSON 以及工作區保留行為。使用 `OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_IMAGE=openclaw-dockerfile-smoke:local OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_SKIP_BUILD=1` 重複使用 install-smoke 映像。
- 閘道網路與主機生命週期：`pnpm test:docker:gateway-network`（指令碼：`scripts/e2e/gateway-network-docker.sh`）會保留雙容器區域網路 WebSocket 驗證／健康狀態冒煙測試，接著使用回送 Admin HTTP 證明準備隔離、保留控制權存取、繼續執行後復原，以及已準備之同一容器的停止／啟動。重新啟動檢查必須在原始租約到期前完成，驗證暫停狀態僅限於處理程序本機，而持久保存的閘道設定與容器身分則會保留，並輸出機器可讀的階段計時 JSON。
- 瀏覽器 CDP 快照冒煙測試：`pnpm test:docker:browser-cdp-snapshot`（指令碼：`scripts/e2e/browser-cdp-snapshot-docker.sh`）會建置原始碼 E2E 映像及 Chromium 層、以原始 CDP 啟動 Chromium、執行 `browser doctor --deep`，並驗證 CDP 角色快照涵蓋連結 URL、由游標提升的可點擊項目、iframe 參照及影格中繼資料。
- OpenAI Responses web_search 最小推理迴歸測試：`pnpm test:docker:openai-web-search-minimal`（指令碼：`scripts/e2e/openai-web-search-minimal-docker.sh`）會透過閘道執行模擬的 OpenAI 伺服器、驗證 `web_search` 將 `reasoning.effort` 從 `minimal` 提升至 `low`，接著強制提供者結構描述拒絕，並檢查原始詳細資料是否出現在閘道記錄中。
- MCP 頻道橋接器（已植入資料的閘道 + stdio 橋接器 + 原始 Claude 通知影格冒煙測試）：`pnpm test:docker:mcp-channels`（指令碼：`scripts/e2e/mcp-channels-docker.sh`）
- OpenClaw 套件 MCP 工具（真實 stdio MCP 伺服器 + 內嵌 OpenClaw 設定檔允許／拒絕冒煙測試）：`pnpm test:docker:agent-bundle-mcp-tools`（指令碼：`scripts/e2e/agent-bundle-mcp-tools-docker.sh`）
- 排程／子代理程式 MCP 清理（真實閘道 + 在隔離排程及單次子代理程式執行後終止 stdio MCP 子處理程序）：`pnpm test:docker:cron-mcp-cleanup`（指令碼：`scripts/e2e/cron-mcp-cleanup-docker.sh`）
- 外掛（本機路徑、`file:`、具提升相依性的 npm 登錄檔、格式錯誤的 npm 套件中繼資料、git 移動參照、ClawHub 廚房水槽、外掛市集更新，以及 Claude 套件啟用／檢查的安裝／更新冒煙測試）：`pnpm test:docker:plugins`（指令碼：`scripts/e2e/plugins-docker.sh`）
  設定 `OPENCLAW_PLUGINS_E2E_CLAWHUB=0` 以略過 ClawHub 區塊，或使用 `OPENCLAW_PLUGINS_E2E_CLAWHUB_SPEC` 和 `OPENCLAW_PLUGINS_E2E_CLAWHUB_ID` 覆寫預設的廚房水槽套件／執行階段組合。若未設定 `OPENCLAW_CLAWHUB_URL`/`CLAWHUB_URL`，測試會使用密閉的本機 ClawHub 固定測試伺服器。
- 外掛更新未變更冒煙測試：`pnpm test:docker:plugin-update`（指令碼：`scripts/e2e/plugin-update-unchanged-docker.sh`）
- 外掛生命週期矩陣冒煙測試：`pnpm test:docker:plugin-lifecycle-matrix` 會在空白容器中安裝已封裝的 OpenClaw tarball、安裝 npm 外掛、切換啟用／停用、透過本機 npm 登錄檔升級及降級該外掛、刪除已安裝的程式碼，接著驗證解除安裝仍會移除過時狀態，同時記錄每個生命週期階段的 RSS／CPU 指標。
- 設定重新載入中繼資料冒煙測試：`pnpm test:docker:config-reload`（指令碼：`scripts/e2e/config-reload-source-docker.sh`）
- 外掛：`pnpm test:docker:plugins` 涵蓋本機路徑、`file:`、具提升相依性的 npm 登錄檔、git 移動參照、ClawHub 固定測試資料、外掛市集更新，以及 Claude 套件啟用／檢查的安裝／更新冒煙測試。`pnpm test:docker:plugin-update` 涵蓋已安裝外掛的未變更更新行為。`pnpm test:docker:plugin-lifecycle-matrix` 涵蓋具資源追蹤的 npm 外掛安裝、啟用、停用、升級、降級，以及程式碼遺失時的解除安裝。

若要手動預先建置並重複使用共用功能映像：

```bash
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local pnpm test:docker:e2e-build
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local OPENCLAW_SKIP_DOCKER_BUILD=1 pnpm test:docker:mcp-channels
```

若有設定 `OPENCLAW_GATEWAY_NETWORK_E2E_IMAGE` 等測試套件專用映像覆寫值，仍會優先採用。當 `OPENCLAW_SKIP_DOCKER_BUILD=1` 指向遠端共用映像時，如果本機尚無該映像，指令碼會將其提取下來。QR 與安裝程式 Docker 測試會保留各自的 Dockerfile，因為它們驗證的是套件／安裝行為，而不是共用的已建置應用程式執行階段。

線上模型 Docker 執行程式也會以唯讀方式繫結掛載目前的簽出，
並將其暫存至容器內的臨時工作目錄。這可維持
執行階段映像精簡，同時仍使用你確切的本機
原始碼／設定執行 Vitest。暫存步驟會略過大型的僅限本機快取和應用程式建置
輸出，例如 `.pnpm-store`、`.worktrees`、`__openclaw_vitest__`，以及
應用程式本機的 `.build` 或 Gradle 輸出目錄，因此 Docker 線上執行不會
花費數分鐘複製特定於機器的成品。它們也會設定
`OPENCLAW_SKIP_CHANNELS=1`，讓閘道線上探查不會在容器內啟動真正的
Telegram／Discord／其他頻道工作程式。
`test:docker:live-models` 仍會執行 `pnpm test:live`，因此若需要從該 Docker 執行管道中
縮小或排除閘道線上涵蓋範圍，也請傳入
`OPENCLAW_LIVE_GATEWAY_*`。

`test:docker:openwebui` 是較高階的相容性冒煙測試：它會啟動
已啟用 OpenAI 相容 HTTP 端點的 OpenClaw 閘道容器、
啟動連線至該閘道的固定版本 Open WebUI 容器、透過
Open WebUI 登入、驗證 `/api/models` 公開 `openclaw/default`，接著透過 Open WebUI 的
`/api/chat/completions` Proxy 傳送真實聊天要求。若發布路徑 CI 檢查應在
Open WebUI 登入及模型探索後停止，而不等待線上模型
完成，請設定 `OPENWEBUI_SMOKE_MODE=models`。第一次執行可能明顯較慢，因為 Docker 可能需要
提取 Open WebUI 映像，而 Open WebUI 可能需要完成其自身的
冷啟動設定。此執行管道需要可用的線上模型金鑰，可透過
處理程序環境、暫存的驗證設定檔或明確的
`OPENCLAW_PROFILE_FILE` 提供。成功執行時會輸出類似
`{ "ok": true, "model": "openclaw/default", ... }` 的小型 JSON 承載資料。

`test:docker:mcp-channels` 刻意採用確定性設計，不需要
真正的 Telegram、Discord 或 iMessage 帳戶。它會啟動已植入資料的閘道
容器、啟動第二個會衍生 `openclaw mcp serve` 的容器，接著
透過真實 stdio MCP 橋接器驗證路由後的對話探索、逐字記錄讀取、附件
中繼資料、即時事件佇列行為、傳出傳送路由，以及 Claude 風格的
頻道 + 權限通知。通知檢查會直接檢查原始 stdio MCP 影格，
因此此冒煙測試驗證的是橋接器實際輸出的內容，而不只是特定用戶端 SDK
碰巧公開的內容。

`test:docker:agent-bundle-mcp-tools` 具確定性，不需要即時模型金鑰。它會建置儲存庫 Docker 映像、在容器內啟動真正的 stdio MCP 探查伺服器、透過內嵌的 OpenClaw 套件 MCP 執行階段具現化該伺服器、執行工具，然後驗證 `coding` 和 `messaging` 會保留 `bundle-mcp` 工具，而 `minimal` 和 `tools.deny: ["bundle-mcp"]` 會將其篩除。

`test:docker:cron-mcp-cleanup` 具確定性，不需要即時模型金鑰。它會啟動已植入種子資料的閘道及真正的 stdio MCP 探查伺服器、執行隔離的排程回合與 `sessions_spawn` 單次子回合，然後驗證 MCP 子處理程序會在每次執行後結束。

手動 ACP 自然語言執行緒冒煙測試（不屬於 CI）：

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- 請保留此指令碼以供迴歸／偵錯工作流程使用。ACP 執行緒路由驗證可能會再次需要它，因此請勿刪除。

實用的環境變數：

- `OPENCLAW_CONFIG_DIR=...`（預設：`~/.openclaw`）掛載至 `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...`（預設：`~/.openclaw/workspace`）掛載至 `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` 會在執行測試前掛載並載入
- `OPENCLAW_DOCKER_PROFILE_ENV_ONLY=1` 僅驗證從 `OPENCLAW_PROFILE_FILE` 載入的環境變數，並使用暫存設定／工作區目錄，且不掛載外部命令列介面驗證資訊
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...`（預設：`~/.cache/openclaw/docker-cli-tools`，除非該次執行已使用 CI／受管理的繫結目錄）掛載至 `/home/node/.npm-global`，供 Docker 內快取命令列介面安裝項目
- `$HOME` 下的外部命令列介面驗證目錄／檔案會以唯讀方式掛載至 `/host-auth...`，然後在測試開始前複製至 `/home/node/...`
  - 預設目錄（執行未限縮至特定提供者時使用）：`.factory`、`.gemini`、`.minimax`
  - 預設檔案：`~/.codex/auth.json`、`~/.codex/config.toml`、`.claude.json`、`~/.claude/.credentials.json`、`~/.claude/settings.json`、`~/.claude/settings.local.json`
  - 限縮的提供者執行只會掛載根據 `OPENCLAW_LIVE_PROVIDERS`／`OPENCLAW_LIVE_GATEWAY_PROVIDERS` 推斷所需的目錄／檔案
  - 可使用 `OPENCLAW_DOCKER_AUTH_DIRS=all`、`OPENCLAW_DOCKER_AUTH_DIRS=none` 或如 `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex` 的逗號分隔清單手動覆寫
- 使用 `OPENCLAW_LIVE_GATEWAY_MODELS=...`／`OPENCLAW_LIVE_MODELS=...` 限縮執行範圍
- 使用 `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...`／`OPENCLAW_LIVE_PROVIDERS=...` 在容器內篩選提供者
- 使用 `OPENCLAW_SKIP_DOCKER_BUILD=1` 重複使用現有的 `openclaw:local-live` 映像，以供不需重新建置的重新執行
- 使用 `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` 確保認證資訊來自設定檔存放區（而非環境變數）
- 使用 `OPENCLAW_OPENWEBUI_MODEL=...` 選擇閘道為 Open WebUI 冒煙測試公開的模型
- 使用 `OPENCLAW_OPENWEBUI_PROMPT=...` 覆寫 Open WebUI 冒煙測試所使用的 nonce 檢查提示
- 使用 `OPENWEBUI_IMAGE=...` 覆寫固定的 Open WebUI 映像標籤

## 文件健全性檢查

編輯文件後執行文件檢查：`pnpm check:docs`。
若也需要檢查頁面內標題，請執行完整的 Mintlify 錨點驗證：`pnpm docs:check-links:anchors`。

## 離線迴歸（適用於 CI）

這些是不使用真實提供者的「真實流水線」迴歸測試：

- 閘道工具呼叫（模擬 OpenAI、真實閘道與代理程式迴圈）：`src/gateway/gateway.test.ts`（案例：「透過閘道代理程式迴圈端對端執行模擬 OpenAI 工具呼叫」）
- 閘道精靈（WS `wizard.start`/`wizard.next`，寫入設定並強制驗證）：`src/gateway/gateway.test.ts`（案例：「透過 ws 執行精靈並寫入驗證權杖設定」）

## 代理程式可靠性評估（Skills）

目前已有幾項適用於 CI、行為類似「代理程式可靠性評估」的測試：

- 透過真實閘道與代理程式迴圈進行模擬工具呼叫（`src/gateway/gateway.test.ts`）。
- 驗證工作階段接線與設定效果的端對端精靈流程（`src/gateway/gateway.test.ts`）。

Skills 仍缺少的項目（請參閱 [Skills](/zh-TW/tools/skills)）：

- **決策：**當提示中列出 Skills 時，代理程式是否會選擇正確的 Skill（或避開不相關的 Skill）？
- **遵循性：**代理程式是否會在使用前讀取 `SKILL.md`，並遵循必要的步驟／引數？
- **工作流程合約：**斷言工具順序、工作階段歷史延續及沙箱邊界的多回合情境。

未來的評估應優先維持確定性：

- 使用模擬提供者的情境執行器，用於斷言工具呼叫與順序、Skill 檔案讀取及工作階段接線。
- 一組以 Skill 為重點的小型情境套件（使用與避用、閘控、提示注入）。
- 僅在適用於 CI 的套件就緒後，才加入選用且由環境變數控管的即時評估。

## 合約測試（外掛與頻道形式）

合約測試會驗證每個已註冊的外掛與頻道都符合其介面合約。它們會逐一檢查所有探索到的外掛，並執行一套形式與行為斷言。預設的 `pnpm test` 單元測試通道會刻意略過這些共用接合面與冒煙測試檔案；異動共用頻道或提供者介面時，請明確執行合約命令。

### 命令

- 所有合約：`pnpm test:contracts`
- 僅頻道合約：`pnpm test:contracts:channels`
- 僅提供者合約：`pnpm test:contracts:plugins`

### 頻道合約

位於 `src/channels/plugins/contracts/*.contract.test.ts`。目前的頂層類別：

- **channel-catalog**－內建／登錄檔頻道目錄項目中繼資料
- **plugin**（由登錄檔支援、分片）－基本外掛註冊形式
- **surfaces-only**（由登錄檔支援、分片）－針對 `actions`、`setup`、`status`、`outbound`、`messaging`、`threading`、`directory` 及 `gateway` 的個別介面形式檢查
- **session-binding**（由登錄檔支援）－工作階段繫結行為
- **outbound-payload**－訊息承載資料結構與正規化
- **group-policy**（後援）－各頻道的預設群組原則強制執行
- **threading**（由登錄檔支援、分片）－執行緒 ID 處理
- **directory**（由登錄檔支援、分片）－目錄／名冊 API
- **registry** 與 **plugins-core.\***－頻道外掛登錄檔、載入器及設定寫入授權內部機制

這些套件使用的輸入分派擷取與輸出承載資料測試工具輔助程式，會透過 `src/plugin-sdk/channel-contract-testing.ts` 在內部公開（不包含於 npm，不是公開的 SDK 子路徑）；此目錄中沒有獨立的 `inbound.contract.test.ts` 檔案。

### 提供者合約

位於 `src/plugins/contracts/*.contract.test.ts`。目前的類別包括：

- **shape**－外掛資訊清單、API 及執行階段匯出形式
- **plugin-registration**（及平行版本）－資訊清單註冊案例
- **package-manifest**－套件資訊清單要求
- **loader**－外掛載入器設定／拆卸行為
- **registry**－外掛合約登錄檔內容與查詢
- **providers**－各內建提供者之間的共用提供者行為，以及網路搜尋提供者
- **auth-choice**－驗證選項中繼資料與設定行為
- **provider-catalog-deprecation**－已棄用的提供者目錄中繼資料
- **wizard.choice-resolution**、**wizard.model-picker**、**wizard.setup-options**－提供者設定精靈合約
- **embedding-provider**、**memory-embedding-provider**、**web-fetch-provider**、**tts**－功能專屬的提供者合約
- **session-actions**、**session-attachments**、**session-entry-projection**－由外掛擁有的工作階段狀態合約
- **scheduled-turns**－外掛排程回合中繼資料與時間戳記界限
- **host-hooks**、**run-context-lifecycle**、**runtime-import-side-effects**、**runtime-seams**－外掛主機／執行階段生命週期與匯入邊界合約
- **extension-runtime-dependencies**－外掛的執行階段相依套件配置位置

### 執行時機

- 變更 plugin-sdk 匯出或子路徑後
- 新增或修改頻道或提供者外掛後
- 重構外掛註冊或探索機制後

合約測試會在 CI 中執行，且不需要真實 API 金鑰。

## 新增迴歸測試（指南）

修正在即時環境中發現的提供者／模型問題時：

- 如有可能，請新增適用於 CI 的迴歸測試（模擬／虛設提供者，或擷取確切的要求形式轉換）
- 若問題本質上只能在即時環境測試（速率限制、驗證原則），請將即時測試保持在最小範圍，並透過環境變數選擇啟用
- 優先針對能擷取錯誤的最小層級：
  - 提供者要求轉換／重播錯誤 -> 直接模型測試
  - 閘道工作階段／歷史／工具流水線錯誤 -> 閘道即時冒煙測試或適用於 CI 的閘道模擬測試
- SecretRef 周遊防護措施：
  - `src/secrets/exec-secret-ref-id-parity.test.ts` 會從登錄檔中繼資料（`listSecretTargetRegistryEntries()`）為每個 SecretRef 類別衍生一個取樣目標，然後斷言含周遊區段的 exec ID 會遭拒絕。
  - 若在 `src/secrets/target-registry-data.ts` 中新增 `includeInPlan` SecretRef 目標系列，請更新該測試中的 `classifyTargetClass`。此測試會刻意在目標 ID 尚未分類時失敗，確保新類別無法被無聲略過。

## 相關內容

- [即時測試](/zh-TW/help/testing-live)
- [測試更新與外掛](/zh-TW/help/testing-updates-plugins)
- [CI](/zh-TW/ci)
