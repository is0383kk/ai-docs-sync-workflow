---
doc-schema-version: 1
read_when:
    - 執行或重新執行完整版本驗證
    - 比較穩定版與完整版本的驗證設定檔
    - 偵錯發布驗證階段失敗問題
summary: 完整版本驗證階段、子工作流程、發布設定檔、重新執行控制代碼與證據
title: 完整版本發布驗證
x-i18n:
    generated_at: "2026-07-26T08:11:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf165d5515f4b9bb11d239382649d332d20bb8a32bd4492ae99092fb5ee2216
    source_path: reference/full-release-validation.md
    workflow: 16
---

`Full Release Validation` 是發布產品驗證的總括流程。大多數工作
都在子工作流程中進行，因此失敗的執行環境可重新執行，而無須重新啟動
整個發布流程。在凍結 Code SHA 前執行發布準備；若背景機器人尚未提交
Control UI 語系輸出，此步驟會重新產生該輸出，接著強制執行與發布 CI
相同的嚴格零後援檢查。

將產品完整、但尚未更新變更日誌的提交凍結為 **Code SHA**，接著執行：

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

`provider` 也接受 `anthropic` 或 `minimax`，用於跨作業系統的新手引導及
端對端代理程式回合。此輔助工具會從 alpha/beta
套件版本推斷 `beta` 設定檔，否則使用 `stable`。使用
`-f key=value` 傳入替代工作流程輸入；僅在廣泛的建議性掃描中使用 `-f release_profile=full`。

此輔助工具會建立暫時的 `release-ci/*` 參照，固定至一個受信任的
`origin/main` 工作流程 SHA；只將目標 SHA 作為候選 `ref` 傳入，
並在驗證後刪除暫時參照。每個已分派的子流程都必須
回報相同的工作流程 SHA。傳入
`-f reuse_evidence=false` 以強制執行全新流程，或傳入
`--workflow-sha <trusted-main-sha>` 以選取目前 `origin/main` 仍可觸及的較舊工作流程提交。
工作流程本身絕不會建立或更新儲存庫參照。

## 延伸穩定版例外

延伸穩定版發布要求工作流程和目標都必須是
標準分支：

```bash
gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

請勿使用 `pnpm ci:full-release` 或 `release-ci/*`。發布會將該次執行的
分支、頭端/目標 SHA、資訊清單 `workflowRef`、ID 與嘗試次數，綁定至標準
分支和發布提交。

回移產品失敗的修正；對凍結目標的工具進行最小且保持行為不變的修復；
若是供應商、核准或執行器失敗，則在不變更原始碼的情況下重試。任何分支變更都需要全新且完整的執行。
不得因目標較舊而省略必要的
套件、安裝程式、更新、頻道或即時行為。

一般發布中，Code SHA 通過後，只產生並提交
`CHANGELOG.md`。這個新提交即為 **Release SHA**。針對
Release SHA 執行相同的輔助工具。只有在 GitHub 證明 Release
SHA 衍生自 Code SHA，且完整變更路徑集合恰好為
`CHANGELOG.md` 時，才會重複使用產品證據；npm 預檢與套件/安裝驗收仍會在
Release SHA 上執行。

`release_profile=stable` 和 `release_profile=full` 一律執行完整的
即時/Docker 耐久測試。傳入 `run_release_soak=true`，即可使用
`beta` 設定檔納入相同的耐久測試執行區。若驗證資訊清單缺少這項耐久測試和具阻擋性的產品效能證據，
穩定版發布會予以拒絕。

套件驗收通常會從解析後的
`ref` 建置候選 tarball，包括使用 `pnpm ci:full-release` 分派的完整 SHA 執行。beta
發布後，傳入 `release_package_spec=openclaw@YYYY.M.PATCH-beta.N`，即可在發布檢查、套件驗收、跨作業系統、
發布路徑 Docker 與套件 Telegram 中重複使用
已發布的 npm 套件。僅當套件驗收應刻意驗證不同套件時，才使用 `package_acceptance_package_spec`。
Codex 外掛即時套件執行區會遵循相同狀態：已發布的
`release_package_spec` 值會衍生 `codex_plugin_spec=npm:@openclaw/codex@<version>`；
SHA/成品執行會從所選參照封裝 `extensions/codex`；操作人員
也可直接為 `npm:`、`npm-pack:` 或 `git:` 外掛
來源設定 `codex_plugin_spec`。此執行區會授予該外掛所需的明確 Codex 命令列介面安裝核准，
接著執行 Codex 命令列介面預檢及同一工作階段的 OpenAI 代理程式回合。
其最後一個零重試、中等思考程度的回合，會在省略 Codex `final` 的情況下傳送可見進度、
讀取隨機化的工作區輸入、寫入完全相符的成品，
並傳送明確的完成訊息。這可捕捉 v2026.7.1 中一般進度傳送
會終止回合的迴歸問題。

## 頂層階段

對於 `rerun_group=all`，會先執行
`Check for reusable validation evidence` 工作。它會尋找先前最新且通過的完整驗證，該驗證須具有相同的發布
設定檔、有效耐久測試設定和驗證輸入。完全相同目標的重新執行會使用
`exact-target-full-validation-v1`。若後代提交的完整差異恰好為
`CHANGELOG.md`，則使用 `changelog-only-release-v1`；所有產品執行區都會略過，
且驗證器會獨立重新檢查 GitHub 提交比較、不可變父成品、
子流程執行及分派記錄。任何其他目標變更都需要
全新的 Code SHA 驗證。傳入 `reuse_evidence=false` 以強制執行全新的完整
流程。只有 `main` 或標準且固定 SHA 的
`release-ci/*` 參照，其工作流程提交仍位於受信任的 `main` 譜系時，才會重複使用證據；
其他工作流程參照會重新執行所選執行區。

全新的套件相關驗證會先準備一個不可變 tarball 和一個 Docker
映像成品，再分派外掛發行前檢查與 OpenClaw 發布檢查。
兩個子流程都會在使用前驗證相同的套件 SHA、成品 ID、服務摘要、
產生者執行嘗試次數及 Docker 封存檔摘要。與套件無關的
裸 Docker 層使用內容定址的 GHCR 快取；候選版本特定映像
仍為不可變的 GitHub 成品。具有明確已發布
套件規格的聚焦執行則保留現有套件路徑。

此外，對於 `rerun_group=all`，`Verify Docker runtime image assets` 工作會使用
`OPENCLAW_EXTENSIONS=diagnostics-otel,codex` 建置 `runtime-assets` Docker 目標。
它會與其他階段並行執行，並由總括驗證器強制檢查；執行區分派前不再
等待它完成。較精簡的 `rerun_group` 會略過此預檢。

| 階段                    | 詳細資料                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 目標解析                | **工作：** `Resolve target ref`<br />**子工作流程：** 無<br />**驗證內容：** 解析發布分支、標籤或完整提交 SHA，並記錄所選輸入。<br />**重新執行：** 若此步驟失敗，請重新執行總括流程。                                                                                                                                                                                                                                                                                                           |
| 共用候選版本            | **工作：** `Prepare shared release candidate`<br />**子工作流程：** `OpenClaw Live And E2E Checks (Reusable)`<br />**驗證內容：** 封裝並驗證一個精確 SHA 套件、建置一個可運作的 Docker 映像，並為兩個套件相關子工作流程記錄不可變的套件與映像成品元組。<br />**重新執行：** 重新執行受影響的套件、外掛發行前、跨作業系統或即時/E2E 群組。                                                                                                                          |
| Docker 資產預檢         | **工作：** `Verify Docker runtime image assets`<br />**子工作流程：** 無<br />**驗證內容：** 在任何其他階段分派前，確認 `runtime-assets` Docker 建置目標仍可成功。僅針對 `rerun_group=all` 執行。<br />**重新執行：** 使用 `rerun_group=all` 重新執行總括流程。                                                                                                                                                                                                                                        |
| Vitest 與一般 CI        | **工作：** `Run normal full CI`<br />**子工作流程：** `CI`<br />**驗證內容：** 針對目標參照執行手動完整 CI 圖，包括 Linux Node 執行區、隨附外掛分片、外掛與頻道合約分片、Node 22 相容性、`check-*`、`check-additional-*`、已建置成品煙霧測試、文件檢查、Python Skills、Windows、macOS、Control UI i18n，以及透過總括流程執行的 Android。<br />**重新執行：** `rerun_group=ci`。                                                                                         |
| 外掛發行前檢查          | **工作：** `Run plugin prerelease validation`<br />**子工作流程：** `Plugin Prerelease`<br />**驗證內容：** 僅限發布的外掛靜態檢查、代理式外掛涵蓋範圍、完整外掛批次分片、外掛發行前 Docker 執行區，以及用於相容性分類、不具阻擋性的 `plugin-inspector-advisory` 成品。<br />**重新執行：** `rerun_group=plugin-prerelease`。                                                                                                                                                         |
| 發布檢查                | **工作：** `Run release/live/Docker/QA validation`<br />**子工作流程：** `OpenClaw Release Checks`<br />**驗證內容：** 安裝煙霧測試、跨作業系統套件檢查、套件驗收、QA Lab 一致性、即時 Matrix 和 Telegram，以及受閘門控管、屬建議性的 Discord、WhatsApp 和 Slack 執行區。穩定版和完整設定檔也會執行完整即時/E2E 套件及 Docker 發布路徑區塊；beta 可使用 `run_release_soak=true` 選擇加入。<br />**重新執行：** `rerun_group=release-checks` 或範圍更窄的發布檢查處理項目。             |
| 套件 Telegram           | **工作：** `Run package Telegram E2E`<br />**子工作流程：** `NPM Telegram Beta E2E`<br />**驗證內容：** 設定 `release_package_spec` 或 `npm_telegram_package_spec` 時，執行聚焦的已發布套件 Telegram E2E。完整候選版本驗證則改用標準的套件驗收 Telegram E2E。<br />**重新執行：** 使用 `release_package_spec` 或 `npm_telegram_package_spec` 執行 `rerun_group=npm-telegram`。                                                                                                             |
| 產品效能                | **工作：** `Run product performance evidence`<br />**子工作流程：** `OpenClaw Performance`<br />**驗證內容：** 針對目標 SHA 執行發布設定檔效能流程（`profile=release`、`repeat=3`、`fail_on_regression=true`、`publish_reports=false`）。Kova 輸出會保留在工作流程成品中，且子流程必須證明其報告發布器已略過。僅對 `rerun_group=all` 或 `rerun_group=performance` 為必要（具阻擋性）；範圍較窄的重新執行群組不需要。<br />**重新執行：** `rerun_group=performance`。 |
| 總括驗證器              | **工作：** `Verify full validation`<br />**子工作流程：** 無<br />**驗證內容：** 重新檢查已記錄的子流程執行結果，並附加各子工作流程中耗時最長的工作表格。<br />**重新執行：** 將失敗的子流程重新執行至通過後，只需重新執行此工作。                                                                                                                                                                                                                                                                |

總括流程一律以僅成品模式分派產品效能流程。
`OpenClaw Performance` 僅允許排程執行，或明確設定 `publish_reports=true` 的
手動分派發布報告。僅成品防護必須成功完成，以證明發布器工作維持略過狀態。
全新及重複使用的證據都會記錄
`controls.performanceReportPublication=artifact-only`；若證據缺少相符且已正規化的效能子流程
證明，驗證器與重複使用選擇器會予以拒絕。

驗證器會將標準資訊清單上傳為
`full-release-validation-<run-id>-<run-attempt>`。證據工具會在下載該確切成品 ID 前，驗證其成品 ID、摘要、產生者執行及嘗試次數。它會限制下載的 ZIP 大小、依據 REST
`sha256:` 摘要驗證其位元組，並以串流方式讀取唯一允許且大小受限的資訊清單項目，而不解壓縮封存檔。為了支援較舊的
發布取用端，會暫時保留穩定名稱別名。驗證器一律優先採用含嘗試次數限定的成品；
作為過渡措施，只有當產生者為第 1 次嘗試的資訊清單 v2 時，才接受穩定名稱。
對後續嘗試及資訊清單 v3，則拒絕該舊名稱。

對於具有 `rerun_group=all` 的 `ref=main`、`release/*` 參照，以及 Tideclaw
alpha 參照，較新的統括執行會取代具有相同參照與重新執行群組的較舊執行。
父執行取消時，其監控程序會取消所有已分派的子工作流程。
標籤驗證執行與固定 SHA 驗證執行不會互相取消。

## 發布檢查階段

`OpenClaw Release Checks` 是最大的子工作流程。它只解析目標一次，並在可用時驗證統括工作流程的共用套件成品。
直接或聚焦分派會在套件或 Docker 相關階段需要時，準備自己的 `release-package-under-test`
成品。

| 階段                    | 詳細資訊                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 發布目標           | **工作：** `Resolve target ref`<br />**支援工作流程：** 無<br />**測試：** 選定的參照、選用的預期 SHA、設定檔、重新執行群組，以及聚焦的即時套件篩選器。<br />**重新執行：** `rerun_group=release-checks`。                                                                                                                                                                                                                                                                                                                                                             |
| 套件成品         | **工作：** `Prepare release package artifact`<br />**支援工作流程：** 無<br />**測試：** 驗證統括工作流程不可變的套件組合，或為直接／聚焦的發布檢查分派封裝一個候選 tarball，接著提供給下游套件相關檢查。<br />**重新執行：** 受影響的套件、跨作業系統或即時／E2E 群組。                                                                                                                                                                                                                                |
| 安裝冒煙測試            | **工作：** `Run install smoke`<br />**支援工作流程：** `Install Smoke`<br />**測試：** 完整安裝路徑，包括重複使用根 Dockerfile 冒煙映像檔、QR 套件安裝、根目錄與閘道 Docker 冒煙測試、安裝程式 Docker 測試，以及 Bun 全域安裝映像檔供應商冒煙測試。<br />**重新執行：** `rerun_group=install-smoke`。                                                                                                                                                                                                                                                           |
| 跨作業系統                 | **工作：** `cross_os_release_checks`<br />**支援工作流程：** `OpenClaw Cross-OS Release Checks (Reusable)`<br />**測試：** 在 Linux、Windows 與 macOS 上，針對選定的供應商及模式執行全新安裝與升級路徑，並使用候選 tarball 加上基準套件。<br />**重新執行：** `rerun_group=cross-os`。                                                                                                                                                                                                                                                                 |
| 儲存庫與即時 E2E        | **工作：** `Run repo/live E2E validation`<br />**支援工作流程：** `OpenClaw Live And E2E Checks (Reusable)`<br />**測試：** 儲存庫 E2E、即時快取、OpenAI WebSocket 串流、原生即時供應商與外掛分片，以及由 `release_profile` 選定且以 Docker 支援的即時模型／後端／閘道測試框架。<br />**執行：** `run_release_soak=true`、`release_profile=full`，或聚焦的 `rerun_group=live-e2e`。<br />**重新執行：** `rerun_group=live-e2e`，可選擇搭配 `live_suite_filter`。                                                                                |
| Docker 發布路徑      | **工作：** `Run Docker release-path validation`<br />**支援工作流程：** `OpenClaw Live And E2E Checks (Reusable)`<br />**測試：** 針對共用套件成品執行發布路徑 Docker 區塊。<br />**執行：** `run_release_soak=true`、`release_profile=full`，或聚焦的 `rerun_group=live-e2e`。<br />**重新執行：** `rerun_group=live-e2e`。                                                                                                                                                                                                                                     |
| 套件驗收       | **工作：** `Run package acceptance`<br />**支援工作流程：** `Package Acceptance`<br />**測試：** 離線外掛套件固定資料、外掛更新、標準模擬 OpenAI Telegram 套件 E2E，以及針對相同 tarball 執行的已發布版本升級存續檢查。阻擋式發布檢查使用預設的最新已發布基準；耐久檢查（`run_release_soak=true`）會擴展至最近 4 個穩定 npm 發布版本，加上 3 個固定的歷史版本（`2026.4.23`、`2026.5.2`、`2026.4.15`），並針對已回報問題的升級固定資料執行。<br />**重新執行：** `rerun_group=package`。 |
| 成熟度評分卡       | **工作：** `Render maturity scorecard release docs`<br />**支援工作流程：** `maturity-scorecard.yml`<br />**測試：** 針對目標參照呈現建議性成熟度評分卡文件。僅在傳入 `run_maturity_scorecard=true` 時執行。<br />**重新執行：** 使用 `run_maturity_scorecard=true` 執行 `rerun_group=qa`。                                                                                                                                                                                                                                                           |
| QA 同等性                | **工作：** `Run QA Lab parity lane` 與 `Run QA Lab parity report`<br />**支援工作流程：** 直接工作<br />**測試：** 候選與基準代理式同等性套件，接著產生同等性報告。<br />**重新執行：** `rerun_group=qa-parity` 或 `rerun_group=qa`。                                                                                                                                                                                                                                                                                                                         |
| QA 執行階段同等性        | **工作：** `Verify QA Lab runtime-pair lanes`<br />**支援工作流程：** 直接工作<br />**測試：** 標準核心 `openclaw`/`codex` 路徑（`pnpm openclaw qa suite --runtime-pair openclaw,codex --runtime-pair-lane core`），以及搭配 `run_release_soak=true` 的耐久路徑。建議事項：個別路徑工作不會阻擋發布檢查驗證器。<br />**重新執行：** `rerun_group=qa-parity` 或 `rerun_group=qa`。                                                                                                                                                             |
| QA 執行階段工具涵蓋率 | **工作：** `Enforce QA Lab runtime tool coverage`<br />**支援工作流程：** 直接工作<br />**測試：** 使用標準核心執行階段配對路徑（`pnpm openclaw qa coverage --tools`）的輸出，檢查 `openclaw` 與 `codex` 之間的動態工具偏移。阻擋性：此工作無法以建議性設定覆寫。<br />**重新執行：** `rerun_group=qa-parity` 或 `rerun_group=qa`。                                                                                                                                                                                                     |
| QA 即時 Matrix           | **工作：** `Run QA Live Matrix profile`<br />**支援工作流程：** `QA-Lab - All Lanes` 可重複使用工作流程<br />**測試：** 在 `qa-live-shared` 環境中，透過共用 Matrix 即時配接器執行已證明同等的 YAML 情境。<br />**重新執行：** `rerun_group=qa-live` 或 `rerun_group=qa`；若要聚焦重新執行 Matrix，請使用 `live_suite_filter=qa-live-matrix`。                                                                                                                                                                                                                    |
| QA 即時 Telegram         | **工作：** `Run QA Lab live Telegram lane`<br />**支援工作流程：** 受信任的 `OpenClaw Release Telegram QA` 分派<br />**測試：** 使用 Convex CI 認證資訊租約進行即時 Telegram QA。<br />**重新執行：** `rerun_group=qa-live` 或 `rerun_group=qa`。                                                                                                                                                                                                                                                                                                                                 |
| QA 即時 Discord          | **工作：** `Run QA Lab live Discord lane`<br />**支援工作流程：** 直接建議性工作<br />**測試：** 啟用 `OPENCLAW_RELEASE_QA_DISCORD_LIVE_CI_ENABLED` 時，使用 Convex CI 認證資訊租約進行即時 Discord QA。<br />**重新執行：** 使用 `live_suite_filter=qa-live-discord` 執行 `rerun_group=qa-live`。                                                                                                                                                                                                                                                                            |
| QA 即時 WhatsApp         | **工作：** `Run QA Lab live WhatsApp lane`<br />**支援工作流程：** 直接建議性工作<br />**測試：** 啟用 `OPENCLAW_RELEASE_QA_WHATSAPP_LIVE_CI_ENABLED` 時，使用 Convex CI 認證資訊租約進行即時 WhatsApp QA。<br />**重新執行：** 使用 `live_suite_filter=qa-live-whatsapp` 執行 `rerun_group=qa-live`。                                                                                                                                                                                                                                                                        |
| QA 即時 Slack            | **工作：** `Run QA Lab live Slack lane`<br />**支援工作流程：** 直接建議性工作<br />**測試：** 啟用 `OPENCLAW_RELEASE_QA_SLACK_LIVE_CI_ENABLED` 時，使用 Convex CI 認證資訊租約進行即時 Slack QA。<br />**重新執行：** 使用 `live_suite_filter=qa-live-slack` 執行 `rerun_group=qa-live`。                                                                                                                                                                                                                                                                                    |
| 發布驗證器         | **工作：** `Verify release checks`<br />**支援工作流程：** 無<br />**測試：** 選定重新執行群組所需的發布檢查工作。<br />**重新執行：** 聚焦的子工作通過後重新執行。                                                                                                                                                                                                                                                                                                                                                                                   |

## Docker 發布路徑區塊

當 `live_suite_filter` 為空時，Docker 發布路徑階段會執行以下區塊：

| 區塊                                                            | 涵蓋範圍                                                                                                                                       |
| --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `core`                                                          | 核心 Docker 發布路徑冒煙測試通道。                                                                                                        |
| `package-update-openai`                                         | OpenAI 套件安裝／更新行為、Codex 隨需安裝、Codex 外掛即時進度後續追蹤，以及 Chat Completions 工具呼叫。 |
| `package-update-anthropic`                                      | Anthropic 套件安裝與更新行為。                                                                                               |
| `package-update-core`                                           | 與供應商無關的套件和更新行為。                                                                                                |
| `plugins-runtime-plugins`                                       | 驗證外掛行為的外掛執行階段通道。                                                                                          |
| `plugins-runtime-services`                                      | 由服務支援及即時外掛執行階段通道。                                                                                                |
| `plugins-runtime-install-a` 至 `plugins-runtime-install-h` | 為平行發布驗證而拆分的外掛安裝／執行階段批次。                                                                        |
| `openwebui`                                                     | 依要求在專用的大容量磁碟執行器上隔離執行 OpenWebUI 相容性冒煙測試。                                                      |

若只有一個 Docker 通道失敗，請在可重複使用的即時／E2E 工作流程中使用指定的 `docker_lanes=<lane[,lane]>`。若可用，發布成品會包含每個通道的重新執行命令，以及套件成品與映像檔重複使用輸入。

## 發布設定檔

`release_profile` 主要控制發布檢查內即時測試／供應商的涵蓋廣度。它不會移除一般完整 CI、外掛預發佈、安裝冒煙測試、套件驗收或 QA Lab。穩定版和完整設定檔一律執行詳盡的儲存庫／即時 E2E，以及 Docker 發布路徑浸泡測試。Beta 設定檔可透過 `run_release_soak=true` 選擇加入。套件驗收為每個完整候選版本提供標準套件 Telegram E2E，因此總括流程不會重複執行該即時輪詢程式。

| 設定檔  | 預定用途                      | 包含的即時測試／供應商涵蓋範圍                                                                                                                                                                            |
| -------- | --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `beta`   | 最快速的發布關鍵冒煙測試。   | OpenAI／核心即時路徑、OpenAI 的 Docker 即時模型、原生閘道核心、原生 OpenAI 閘道設定檔、原生 OpenAI 外掛，以及 Docker 即時閘道 OpenAI。                                            |
| `stable` | 預設發布核准設定檔。 | `beta` 加上 Anthropic 冒煙測試、Google、MiniMax、後端、原生即時測試工具、Docker 即時命令列介面後端、Docker ACP 繫結、Docker Codex 測試工具、Docker 子代理公告，以及一個 OpenCode Go 冒煙測試分片。 |
| `full`   | 廣泛的諮詢性掃描。             | `stable` 加上諮詢性供應商、外掛即時分片及媒體即時分片。                                                                                                                               |

## 僅完整設定檔新增的項目

`stable` 會略過以下測試套件，而 `full` 會包含它們：

| 領域                             | 僅完整設定檔涵蓋的範圍                                                                                                          |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Docker 即時模型               | OpenCode Go、OpenRouter、xAI、Z.ai 及 Fireworks。                                                                          |
| Docker 即時閘道              | 諮詢性供應商拆分為 DeepSeek／Fireworks、OpenCode Go／OpenRouter，以及 xAI／Z.ai 分片。                              |
| 原生閘道供應商設定檔 | 完整 Anthropic Opus 與 Sonnet／Haiku 分片、Fireworks、DeepSeek、完整 OpenCode Go 模型分片、OpenRouter、xAI 及 Z.ai。 |
| 原生外掛即時分片        | 外掛 A-K、L-N、O-Z 其他項目、Moonshot 及 xAI。                                                                             |
| 原生媒體即時分片         | 音訊、Google 音樂、MiniMax 音樂及影片群組 A-D。                                                                   |

`stable` 包含 `native-live-src-gateway-profiles-anthropic-smoke` 和 `native-live-src-gateway-profiles-opencode-go-smoke`；`full` 則使用涵蓋範圍更廣的 Anthropic 與 OpenCode Go 模型分片。指定重新執行仍可使用彙總的 `native-live-src-gateway-profiles-anthropic` 或 `native-live-src-gateway-profiles-opencode-go` 控制代號。

## 指定重新執行

使用 `rerun_group`，避免重複執行不相關的發布執行環境：

| 控制代號              | 範圍                                                                                           |
| ------------------- | ----------------------------------------------------------------------------------------------- |
| `all`               | 所有完整發布驗證階段。                                                             |
| `ci`                | 僅手動完整 CI 子工作流程。                                                                      |
| `plugin-prerelease` | 僅外掛預發佈子工作流程。                                                                   |
| `release-checks`    | 所有 OpenClaw 發布檢查階段。                                                             |
| `install-smoke`     | 從安裝冒煙測試到發布檢查。                                                           |
| `cross-os`          | 跨作業系統發布檢查。                                                                        |
| `live-e2e`          | 儲存庫／即時 E2E 與 Docker 發布路徑驗證。                                               |
| `package`           | 套件驗收。                                                                             |
| `qa`                | QA 一致性加上 QA 即時通道。                                                                   |
| `qa-parity`         | 僅 QA 一致性通道及報告。                                                                |
| `qa-live`           | QA 即時 Matrix／Telegram，以及啟用時受閘門控管的 Discord、WhatsApp 和 Slack 通道。             |
| `npm-telegram`      | 已發布套件的 Telegram E2E；需要 `release_package_spec` 或 `npm_telegram_package_spec`。 |
| `performance`       | 僅產品效能證據。                                                              |

若一個即時測試套件失敗，請搭配 `rerun_group=live-e2e` 使用 `live_suite_filter`。有效的篩選器 ID 定義於可重複使用的即時／E2E 工作流程中，包括 `docker-live-models`、`live-gateway-docker`、`live-gateway-anthropic-docker`、`live-gateway-google-docker`、`live-gateway-minimax-docker`、`live-gateway-advisory-docker`、`live-cli-backend-docker`、`live-acp-bind-docker` 及 `live-codex-harness-docker`。

若要指定重新執行 QA 傳輸測試，請設定 `rerun_group=qa-live`，並使用標準選擇器 `qa-live-matrix`、`qa-live-telegram`、`qa-live-discord`、`qa-live-whatsapp` 或 `qa-live-slack`。

`live-gateway-advisory-docker` 控制代號是其三個供應商分片的彙總重新執行控制代號，因此仍會展開執行所有諮詢性 Docker 閘道工作。

若一個跨作業系統通道失敗，請搭配 `rerun_group=cross-os` 使用 `cross_os_suite_filter`。篩選器接受作業系統 ID、測試套件 ID 或作業系統／測試套件配對，例如 `windows/packaged-upgrade`、`windows` 或 `packaged-fresh`。跨作業系統摘要會包含套件升級通道各階段的耗時，而長時間執行的命令會輸出心跳偵測行，讓卡住的更新能在工作逾時前被發現。

只有指定的 Matrix、Telegram 及 QA 執行階段工具涵蓋通道中發生的 QA 發布檢查失敗，才會阻擋一般發布驗證。QA 一致性、執行階段一致性，以及受閘門控管的 Discord、WhatsApp 和 Slack 即時通道屬於諮詢性質，會發布狀態成品而不阻擋發布驗證器。Tideclaw Alpha 執行仍可將不涉及套件安全性的發布檢查通道視為諮詢性質。使用 `release_profile=beta` 時，`Run repo/live E2E validation` 即時供應商測試套件屬於諮詢性質：第三方模型部署會在發布期間於底層發生變更，因此 Beta 會將其失敗顯示為警告，而穩定版和完整設定檔仍會讓這些失敗阻擋流程。當 `live_suite_filter` 明確要求 Discord、WhatsApp 或 Slack 等受閘門控管的 QA 即時通道時，必須啟用相符的 `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED` 儲存庫變數；否則輸入擷取會失敗，而非默默略過該通道。需要新的 QA 證據時，請重新執行 `rerun_group=qa`、`qa-parity` 或 `qa-live`。

## 應保留的證據

保留 `Full Release Validation` 摘要作為發布層級索引。它會連結子執行 ID，並包含最耗時工作表格。若發生失敗，請先檢查子工作流程，再重新執行上述最小的相符控制代號。

一般發布需記錄 Code SHA 與 Release SHA、重複使用政策與變更路徑集合、綠燈 Code SHA 父執行，以及輕量 Release SHA 父執行。對於延伸穩定版，需記錄標準分支、確切發布 SHA、新的父執行 ID 與嘗試次數、工作流程參照、每個子執行，以及任何凍結目標相容性修復或刻意省略的項目。

實用成品：

- `release-package-under-test`，來自 `OpenClaw Release Checks`
- `.artifacts/docker-tests/` 下的 Docker 發布路徑成品
- 套件驗收 `package-under-test` 與 Docker 驗收成品
- 各作業系統和測試套件的跨作業系統發布檢查成品
- QA 一致性、執行階段一致性，以及指定的 Matrix、Telegram、Discord、WhatsApp 或 Slack 成品

## 工作流程檔案

- `.github/workflows/full-release-validation.yml`
- `.github/workflows/openclaw-release-checks.yml`
- `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml`
- `.github/workflows/plugin-prerelease.yml`
- `.github/workflows/install-smoke.yml`
- `.github/workflows/install-smoke-reusable.yml`
- `.github/workflows/openclaw-cross-os-release-checks-reusable.yml`
- `.github/workflows/package-acceptance.yml`
- `.github/workflows/openclaw-performance.yml`
- `.github/workflows/npm-telegram-beta-e2e.yml`
