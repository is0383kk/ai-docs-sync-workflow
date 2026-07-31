---
doc-schema-version: 1
read_when:
    - 尋找公開發布頻道的定義
    - 執行版本驗證或套件驗收
    - 尋找版本命名與發布週期
summary: 發布管道、操作人員檢查清單、驗證方塊、版本命名與發布節奏
title: 發布政策
x-i18n:
    generated_at: "2026-07-26T08:35:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de2429f039bb42deabdcfe280b7d91afac3bae3dc24714203ab7a67672dcc10c
    source_path: reference/RELEASING.md
    workflow: 16
---

OpenClaw 提供四個面向使用者的更新通道：

- stable：npm 上已推廣的正式版本 `latest`
- extended-stable：前一個已結束月份在
  npm 上的 `.33+` 維護線 `extended-stable`
- beta：npm 上的預發行標籤 `beta`
- dev：持續移動的 `main` 頂端

Extended-stable 會提供前一個月份的閘道、官方 npm 外掛和
Docker 映像檔，而不會移動正式版的 `latest` 或 `main` 選擇器。

Tideclaw alpha 組建是另一條內部預發行軌道（npm dist-tag `alpha`），詳見 [NPM 工作流程輸入](#npm-workflow-inputs)和[發行測試機](#release-test-boxes)。

## 版本命名

- 每月閘道 extended-stable 發行版本：`YYYY.M.PATCH`，搭配 `PATCH >= 33`、git 標籤 `vYYYY.M.PATCH`
- 每日／正式最終發行版本：`YYYY.M.PATCH`，搭配 `PATCH < 33`、git 標籤 `vYYYY.M.PATCH`
- 正式備援修正發行版本：`YYYY.M.PATCH-N`，git 標籤 `vYYYY.M.PATCH-N`
- Beta 預發行版本：`YYYY.M.PATCH-beta.N`，git 標籤 `vYYYY.M.PATCH-beta.N`
- Alpha 預發行版本：`YYYY.M.PATCH-alpha.N`，git 標籤 `vYYYY.M.PATCH-alpha.N`
- 月份或修補版本號絕不補零
- `PATCH` 是依序遞增的每月發行列車編號，不是日曆日期。正式最終版和 beta 版會推進目前的列車；僅限 alpha 的標籤絕不會占用或推進 beta／正式版的修補版本號，因此選擇 beta 或正式版列車時，請忽略修補版本號較高的舊版僅限 alpha 標籤。
- Alpha／夜間組建使用下一個尚未發行的修補版本列車，重複組建時只遞增 `alpha.N`。該修補版本推出 beta 後，新的 alpha 組建會移至下一個修補版本。
- npm 版本不可變更：絕不刪除、重新發布或重複使用已發布的標籤。應改為建立下一個預發行編號或下一個每月修補版本。
- `latest` 持續跟隨目前的正式／每日 npm 發行線；`beta` 是目前的 beta 安裝目標
- `extended-stable` 代表受支援的前一月份閘道發行版，從修補版本 `33` 開始；修補版本 `34` 及後續版本是該每月發行線的維護版本
- 正式最終版和正式修正版預設發布至 npm `beta`；發行操作人員可以明確指定 `latest`，也可以稍後推廣已審核的 beta 組建
- 閘道 extended-stable 會以完全相同的版本發布核心、所有可發布至 npm 的官方外掛，
  以及其 Docker 映像檔；請參閱下方的專用工作流程。
- 每個正式最終版本都會一併提供 npm 套件、macOS 應用程式、已簽署的獨立 Android APK，以及已簽署的 Windows Hub 安裝程式。Beta 版本通常會先驗證並發布 npm／套件路徑；除非明確要求，原生應用程式的組建／簽署／公證／推廣會保留給正式最終版本。

## 發行週期

- 發行會先推出 beta；只有在最新 beta 通過驗證後，才會推出 stable
- 維護人員通常會從目前 `main` 建立的 `release/YYYY.M.PATCH` 分支進行發行，因此發行驗證和修正不會阻礙 `main` 上的新開發
- 如果 beta 標籤已推送或發布後需要修正，維護人員會建立下一個 `-beta.N` 標籤，而不是刪除或重新建立舊標籤
- 詳細的發行程序、核准、認證資訊和復原說明僅供維護人員使用

## 每月閘道 extended-stable 發布

針對已結束的月份 `YYYY.M`，建立 `extended-stable/YYYY.M.33`，並從該分支發布
`.33+`。標籤、分支、簽出、套件版本、預檢和
驗證必須指向同一個提交。在 `.33` 之前，受保護的 `main` 必須包含
後續月份且修補版本低於 `33` 的最終版本；之後的維護修補版本仍然
符合資格。

### 準備並穩定候選版本

稽核尚未稽核的主線範圍、協調私有安全性工作、核准一組
範圍有限的反向移植，並合併一個協調一致的 PR。不要直接推送至標準
分支。

在標準分支上設定 `YYYY.M.P`、執行 `pnpm release:prep`，並要求
每個可發布的官方外掛都使用該版本。根據已核准的清冊，
產生並提交完整的 `## YYYY.M.P` 區段，其中包含 `### Highlights`、
`### Changes` 和 `### Fixes`；對於等效的反向移植，應引用原始已合併的 `main` PR。
缺少區段或區段為空時，預檢會拒絕執行。

移植完整的目前主線 Docker 發行通道單元：工作流程、推廣程式、
原則、共用分類器、測試和工作流程驗證。GitHub 會從已加標籤的提交載入標籤
工作流程；不完整的複本可能會在完成組建後失敗，或移動正式版別名。執行聚焦檢查。

凍結完整的分支頂端 SHA。加上標籤前，先預檢其確切的 npm 位元組，
並針對該 SHA 執行完整發行驗證：

```bash
RELEASE_SHA="$(git rev-parse HEAD)"

gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag="$RELEASE_SHA" \
  -f preflight_only=true \
  -f npm_dist_tag=extended-stable

gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

SHA 形式僅供預檢使用。在標準分支上執行驗證；發布作業會綁定其
工作流程參照、頂端／目標 SHA、執行 ID 和嘗試次數。儲存兩個 ID 和
成功的 `run_attempt`；拒絕 `release-ci/*` 證據。

編輯前先將失敗分類：

- 產品：合併另一個已核准的反向移植 PR。
- 凍結目標工具：僅反向移植最小的相容性修正，
  並測試舊產品保持不變。
- 供應商、核准、執行器或服務：保持候選版本不變，並使用
  範圍有限的重試路徑。

任何分支變更都會使兩個關卡失效。兩者通過後，確認頂端仍然
等於 `RELEASE_SHA`，然後推送已簽署的 `vYYYY.M.P`。後續變更必須使用下一個
修補版本；絕不移動或刪除標籤。推送該標籤會啟動 `Docker Release`。

### 發布 npm 套件

從相同的 SHA 發布所有可發布至 npm 的官方外掛，並儲存
成功的執行 ID：

```bash
RELEASE_SHA="$(git rev-parse HEAD)"
gh workflow run plugin-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f publish_scope=all-publishable \
  -f ref="$RELEASE_SHA" \
  -f npm_dist_tag=extended-stable
```

此工作流程涵蓋所有 `all-publishable` 套件，包括未變更的套件，
並驗證每個確切版本和選擇器。重新執行時會重複使用已發布的版本。

接著使用全部三個已儲存的執行識別資訊，發布準備好的核心 tarball：

```bash
gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag=vYYYY.M.P \
  -f preflight_only=false \
  -f npm_dist_tag=extended-stable \
  -f preflight_run_id=<npm-preflight-run-id> \
  -f full_release_validation_run_id=<full-validation-run-id> \
  -f full_release_validation_run_attempt=<full-validation-run-attempt> \
  -f plugin_npm_run_id=<plugin-npm-run-id>
```

僅限非正式環境演練時，將
`-f bypass_extended_stable_guard=true` 加入預檢和發布。它只會略過
月份防護，絕不會略過標準參照、SHA／標籤／版本相等性、來源證明、
核准或讀回檢查。絕不可將其用於正式環境。

### 驗證與復原

從另一個乾淨且目前的 `main` 簽出，而非凍結分支，執行：

```bash
node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.P
npm view openclaw@YYYY.M.P version --userconfig "$(mktemp)"
npm view openclaw@extended-stable version --userconfig "$(mktemp)"
```

標準分支必須具有簽章和 npm 來源證明，且發布、
預檢和 tarball 摘要都必須綁定至發行 SHA。兩個命令都必須
傳回 `YYYY.M.P`。驗證每個已準備的核心套件，以及 `all-publishable`
個官方外掛的確切版本和選擇器。

如果只有根選擇器失敗，請使用工作流程摘要中印出的
`npm dist-tag add openclaw@YYYY.M.P extended-stable` 修復命令。
透過已核准且認證資訊隔離的工具，修復現有外掛或其他已準備的核心選擇器；
OIDC 來源無法變更它們。絕不重新發布不可變更的版本。

要求 `Docker Release` 驗證 GHCR 和 Docker Hub 中確切的預設、slim、browser 和架構
映像檔，包括證明和平台版本。它只能依
摘要推進 `extended-stable`、`extended-stable-slim` 和 `extended-stable-browser`；
正式版別名維持不變，且會拒絕自動復原。

如需修復別名，請從目前的
`main` 使用該標籤執行需要核准的 `Docker Channel Promotion`。它會重複摘要、證明和平台檢查、允許
明確復原，而且絕不會重新組建映像檔。

Slack、Discord 和 Codex 是最初記載的支援介面，而不是
發行允許清單：所有可發布至 npm 的官方外掛都會發布。只有正式版
檢查清單負責 beta／`latest`、GitHub Releases、ClawHub、原生應用程式、行動版、
網站和私有 dist-tag；請勿為此閘道路徑執行這些步驟。

## 正式版發行操作人員檢查清單

此檢查清單是發行流程的公開形式。私有認證資訊、簽署、公證、dist-tag 復原和緊急復原的詳細資料，仍保留在僅供維護人員使用的發行操作手冊中。

1. 從目前的 `main` 開始：提取最新內容、確認目標提交已推送，並確認 `main` CI 的綠燈狀態足以建立分支。
2. 從該提交建立 `release/YYYY.M.PATCH`。反向移植為選用；只套用操作人員選取的項目。提升每個必要位置的版本、執行 `pnpm release:prep`、完成發行修正和必要的正向移植，並審閱 `src/plugins/compat/registry.ts` 和 `src/commands/doctor/shared/deprecation-compat.ts`。
3. 將產品完整且尚未加入變更記錄的提交凍結為 **程式碼 SHA**。執行確定性的原始碼預檢，然後使用 `node scripts/full-release-validation-at-sha.mjs --sha <code-sha> --target-ref release/YYYY.M.PATCH`。如此會固定受信任的工作流程工具，而完整的 Vitest、Docker、QA、套件和效能矩陣會以確切的程式碼 SHA 為目標。
4. 編輯前先將失敗分類。產品／程式碼失敗會產生新的程式碼 SHA，且該 SHA 必須通過完整驗證。工作流程、測試框架、認證資訊、核准或基礎架構失敗，則在其所屬介面中修復，並針對相同的程式碼 SHA 重新執行。
5. 只有在程式碼 SHA 顯示綠燈後，才從自上一個可到達的已發布標籤以來合併的 PR 和直接提交，產生最上方的 `CHANGELOG.md` 區段。項目應面向使用者且不重複。如果分歧的已發布標籤或後續正向移植重新關聯已發布的 PR，請透過 `--shipped-ref` 明確傳入該標籤。
6. 只提交 `CHANGELOG.md`。此提交即為 **發行 SHA**。從程式碼 SHA 到發行 SHA 的完整差異必須恰好是 `CHANGELOG.md`；任何其他變更路徑都會使發行流程返回步驟 2。
7. 針對發行 SHA 執行固定 SHA 的完整發行驗證，並啟用證據重複使用。輕量父流程必須記錄 `changelog-only-release-v1`、指向顯示綠燈的程式碼 SHA，且不得派送任何產品子流程。這會重複使用產品證據，但不會重複使用套件位元組。
8. 針對發行 SHA／標籤，使用 `preflight_only=true` 執行 `OpenClaw NPM Release`。儲存成功的 `preflight_run_id`。這會組建並檢查包含最終變更記錄的確切套件位元組。
9. 為發行 SHA 加上標籤，接著以成功的發行 SHA 驗證父流程和 npm 預檢執行候選版本輔助工具，而不是再次派送其中任一流程：

   ```bash
   pnpm release:candidate -- \
     --tag vYYYY.M.PATCH-beta.N \
     --full-release-run <release-sha-validation-run-id> \
     --npm-preflight-run <preflight-run-id> \
     --skip-dispatch
   ```

   若為穩定版，也請傳入 `--windows-node-tag vX.Y.Z`。此輔助工具會驗證版本資訊來源、npm 發布前檢查位元組、Parallels 安裝／更新證明、Telegram 套件證明及外掛發布計畫，然後印出發布命令。

   `OpenClaw Release Publish` 會將選取的或所有可發布的外掛套件平行派送至 npm，並將同一組套件派送至 ClawHub；外掛成功發布至 npm 後，再使用相符的 dist-tag 提升已備妥的 OpenClaw npm 發布前檢查成品。發布簽出目錄仍是產品／資料根目錄，而規劃與最終驗證則從完全相符且受信任的工作流程來源簽出目錄執行，避免較舊的發布提交在未察覺的情況下使用過時的發布工具。在啟動任何發布子工作前，它會轉譯並快取完全相符的 GitHub 版本頁面內文。當完整且相符的 `CHANGELOG.md` 區段符合 GitHub 的 125,000 字元限制及轉譯器相符的 125,000 位元組安全上限時，頁面會包含該完全相符的 `## YYYY.M.PATCH` 區段及其標題。當來源區段超出限制時，頁面會保留完全相符的分組編輯版本資訊，並將過大的貢獻紀錄替換為指向由標籤固定之 `CHANGELOG.md` 中完整紀錄的穩定連結；絕不發布不完整的紀錄或遭截斷的項目符號。工作流程會先選擇完整或精簡內文，再加入 `### Release verification`；若證明結尾會超出限制，則保留標準內文，並改以不可變的附加證據為準。發布至 npm `latest` 的穩定版本會成為 GitHub 最新版本，而保留在 npm `beta` 上的穩定維護版本則會以 GitHub `latest=false` 建立。工作流程也會將發布前檢查相依性證據、完整驗證資訊清單及發布後登錄檔驗證證據上傳至 GitHub 版本，以供發布後事件處理使用。它會立即印出子工作執行 ID、自動核准工作流程權杖有權核准的發布環境閘門、使用記錄結尾摘要失敗的子工作、預先建立 GitHub 版本草稿頁面，並在 OpenClaw 發布至 npm 的同時提升 Windows 與 Android 成品；這些階段成功後，完成版本頁面與相依性證據；每當發布 OpenClaw npm 時，等待 ClawHub 完成；接著執行受信任 main 的 beta 驗證器，並上傳 GitHub 版本、npm 套件、選取的外掛 npm 套件、選取的 ClawHub 套件、子工作流程執行 ID，以及選用的 NPM Telegram 執行 ID 之發布後證據。ClawHub 啟動驗證器要求完全相符且受信任的 main 工作流程路徑與 SHA、生產者及最終執行嘗試、發布 SHA、要求的套件組、不可變的套件成品元組，以及最終登錄檔回讀成品；不接受成功的舊式發布參照執行。

   接著，針對已發布的 `openclaw@YYYY.M.PATCH-beta.N` 或 `openclaw@beta` 套件執行發布後套件驗收。如果已推送或發布的預發行版本需要修正，請建立下一個相符的預發行編號；絕不刪除或改寫舊版本。

10. 發布嘗試失敗時，除非該失敗證明產品或變更日誌有缺陷，否則請保持 Release SHA 不變。沿用成功且不可變的子工作及成品；絕不重新建置或重新發布已成功的套件版本。
11. 若為穩定版，僅在已審核的 beta 或候選版本具備所需驗證證據後才能繼續。穩定版 npm 發布也會經過 `OpenClaw Release Publish`，並透過 `preflight_run_id` 重複使用成功的發布前檢查成品。穩定版 macOS 發布就緒還要求 `main` 上已封裝的 `.zip`、`.dmg`、`.dSYM.zip`，以及已更新的 `appcast.xml`；macOS 發布工作流程會在驗證版本成品後，自動將已簽署的 appcast 發布至公開的 `main`，若分支保護阻止直接推送，則會建立或更新 appcast PR。穩定版 Windows Hub 就緒要求 OpenClaw GitHub 版本中有已簽署的 `OpenClawCompanion-Setup-x64.exe`、`OpenClawCompanion-Setup-arm64.exe` 及 `OpenClawCompanion-SHA256SUMS.txt` 成品。將完全相符且已簽署的 `openclaw/openclaw-windows-node` 版本標籤作為 `windows_node_tag` 傳入，並將其經候選版本核准的安裝程式摘要對應表作為 `windows_node_installer_digests` 傳入；`OpenClaw Release Publish` 會保留版本草稿、派送 `Windows Node Release`，並在發布前驗證全部三項成品。
12. 發布後，執行 npm 發布後驗證器；需要發布後頻道證明時，可選擇執行獨立的已發布 npm Telegram E2E；視需要提升 dist-tag、驗證產生的 GitHub 版本頁面、執行版本公告步驟，然後完成[穩定版 main 收尾](#stable-main-closeout)，之後才能宣告穩定版本完成。

## 穩定版 main 收尾

在 `main` 納入實際已發布的版本狀態前，穩定版發布尚未完成。

1. 從全新且最新的 `main` 開始。以其為基準稽核 `release/YYYY.M.PATCH`，並向前移植 `main` 中缺少的實際修正。不要盲目將僅供發布使用的相容性、測試或驗證轉接器合併至較新的 `main`。
2. 一般流程中，將 `main` 設為已發布的穩定版本。若 `main` 已推進至較新的 OpenClaw 穩定版 CalVer，延遲的收尾可以使用它；不要只為了結束上一個版本，就將已開始的發布列車降級。驗證器仍要求完全相符的已發布變更日誌區段與 appcast 項目，並記錄實際的 `main` 版本與 SHA。根版本有任何變更後，先執行 `pnpm release:prep`，再執行 `pnpm deps:shrinkwrap:generate`。
3. 使 `CHANGELOG.md` 在 `main` 上的 `## YYYY.M.PATCH` 區段與已加標籤的發布分支完全相符。若 Mac 版本發布了穩定版 `appcast.xml` 更新，請將其納入。
4. 在操作者明確啟動該發布列車前，不要將 `YYYY.M.PATCH+1`、beta 版本或空白的未來變更日誌區段加入 `main`。
5. 執行 `pnpm release:generated:check`、`pnpm deps:shrinkwrap:check` 及 `OPENCLAW_TESTBOX=1 pnpm check:changed`。推送後，確認 `origin/main` 包含已發布的版本與變更日誌，之後才能宣告穩定版本完成。
6. 每次私有復原演練後，請保持儲存庫變數 `RELEASE_ROLLBACK_DRILL_ID` 與 `RELEASE_ROLLBACK_DRILL_DATE` 為最新狀態。

`OpenClaw Stable Main Closeout` 從穩定版發布後，包含已發布版本、變更日誌與 appcast 的 `main` 推送開始。它會讀取不可變的發布後證據，將已發布標籤繫結至其完整版本驗證與發布執行，然後驗證穩定版 main 狀態、版本、必要的穩定版浸泡測試，以及阻擋性的效能證據。它會將不可變的收尾資訊清單與總和檢查碼附加至 GitHub 版本。自動推送觸發程序會略過早於不可變發布後證據的舊式版本，而且絕不將該略過視為已完成收尾。

完整收尾同時需要兩項成品及相符的總和檢查碼。不完整的資訊清單會重播其中記錄的 `main` SHA 與復原演練，以重新產生完全相同的位元組，然後附加缺少的總和檢查碼；無效的配對或只有總和檢查碼而沒有資訊清單時，仍會阻擋流程。若推送觸發的執行缺少復原演練儲存庫變數，會略過而不完成收尾；缺少演練紀錄或紀錄已超過 90 天，仍會阻擋手動的證據式收尾。私有復原命令仍保留在僅限維護者使用的操作手冊中。僅使用手動派送來修復或重播具證據支援的穩定版收尾。

如果 Release Publish 父工作僅在附加不可變的 npm／外掛證據後失敗，請先修復並發布每一項穩定版平台成品。之後，維護者可以使用 `allow_failed_publish_recovery=true` 手動派送收尾；該模式僅接受已完成但失敗的父工作，並且除了正常的 macOS／appcast 檢查外，還要求完全相符的 Android 與 Windows 成品合約、GitHub SHA-256 摘要、總和檢查碼驗證、Android 來源證明，以及由父工作派送且成功的 Windows 提升作業，其 Authenticode 檢查及經候選版本核准的摘要必須與已發布的安裝程式相符。自動推送收尾絕不啟用此復原模式。

舊式備援修正標籤只有在修正標籤與基礎穩定版標籤解析至相同來源提交時，才能重複使用基礎套件證據。其 Android 版本會重複使用基礎標籤已驗證的 APK，並加入修正標籤的來源證明。來源不同的修正版本必須發布並驗證自己的套件證據，並使用較高的 Android `versionCode`。

## 發布前檢查

- 在發布前檢查之前執行 `pnpm check:test-types`，以確保測試用 TypeScript 在較快速的本機 `pnpm check` 閘門之外仍有涵蓋。
- 在發布前檢查之前執行 `pnpm check:architecture`，以確保更廣泛的匯入循環與架構邊界檢查在較快速的本機閘門之外皆通過。
- 在 `pnpm release:check` 之前執行 `pnpm build && pnpm ui:build`，以便產生預期的 `dist/*` 發布成品及 Control UI 套件組合，供套件驗證步驟使用。
- 在根版本提升之後、加上標籤之前執行 `pnpm release:prep`。它會執行所有在版本／設定／API 變更後通常容易產生偏差的確定性發布產生器：外掛版本、npm shrinkwrap、外掛清冊、基礎設定結構描述、內建頻道設定中繼資料、設定文件基準、外掛 SDK 匯出、外掛 SDK API 合約資訊清單，以及 Control UI 語系套件組合。它也會持續阻擋，直到原生應用程式翻譯及平台產生的語系資源與來源清冊相符；若兩者落後，請先等待或派送 `Native App Locale Refresh`，再凍結 Code SHA。`pnpm release:check` 會以檢查模式重新執行這些防護措施（包括嚴格的語系閘門及外掛 SDK 介面預算），並在執行套件發布檢查之前，一次回報所有產生內容偏差失敗。
- 外掛版本同步預設會將可發布的 `@openclaw/ai` 執行階段套件、官方外掛套件版本，以及現有的 `openclaw.compat.pluginApi` 最低版本更新為 OpenClaw 發布版本。請將該欄位視為外掛 SDK／執行階段 API 的最低版本，而不只是套件版本的副本：對於刻意維持與較舊 OpenClaw 主機相容的純外掛版本，請將最低版本保留在最舊的支援主機 API，並在外掛發布證明中記錄此選擇。
- 在核准發布前執行手動 `Full Release Validation` 工作流程，從單一進入點啟動所有預發行測試機。它接受分支、標籤或完整提交 SHA，派送手動 `CI`，並派送 `OpenClaw Release Checks` 以執行安裝煙霧測試、套件驗收、跨作業系統套件檢查、QA Lab 一致性、Matrix 及 Telegram 測試執行區。穩定版與完整執行一律包含完整的即時／E2E 與 Docker 發布路徑浸泡測試；`run_release_soak=true` 則保留供明確的 beta 浸泡測試使用。套件驗收會在候選版本驗證期間提供標準的套件 Telegram E2E，避免同時執行第二個即時輪詢器。

  發布 beta 後提供 `release_package_spec`，即可在版本檢查、套件驗收及套件 Telegram E2E 中重複使用已發布的 npm 套件，而不必重新建置發布 tarball。只有當 Telegram 應使用與其餘版本驗證不同的已發布套件時，才提供 `npm_telegram_package_spec`。當套件驗收應使用與發布套件規格不同的已發布套件時，提供 `package_acceptance_package_spec`。當版本證據報告應證明驗證與已發布的 npm 套件相符，但不強制執行 Telegram E2E 時，提供 `evidence_package_spec`。

  ```bash
  node scripts/full-release-validation-at-sha.mjs \
    --sha <code-sha> \
    --target-ref release/YYYY.M.PATCH
  ```

- 若要在發布作業持續進行期間，為套件候選版本取得側通道證明，請執行手動 `Package Acceptance` 工作流程。使用 `source=npm` 指定 `openclaw@beta`、`openclaw@latest` 或確切的發布版本；使用 `source=ref`，透過目前的 `workflow_ref` 測試框架封裝受信任的 `package_ref` 分支／標籤／SHA；使用 `source=url` 指定具備必要 SHA-256 且符合嚴格公開 URL 政策的公開 HTTPS tarball；使用 `source=trusted-url` 指定具名的受信任來源政策，並提供必要的 `trusted_source_id` 與 SHA-256；或使用 `source=artifact` 指定由另一個 GitHub Actions 執行作業上傳的 tarball。

  此工作流程會將候選版本解析為 `package-under-test`、針對該 tarball 重複使用 Docker E2E 發布排程器，並可透過 `telegram_mode=mock-openai` 或 `telegram_mode=live-frontier`，針對相同的 tarball 執行 Telegram QA。當選取的 Docker 通道包含 `published-upgrade-survivor` 時，套件成品即為候選版本，而 `published_upgrade_survivor_baseline` 會選取已發布的基準版本。`update-restart-auth` 會同時以候選套件作為已安裝的命令列介面與受測套件，以便測試候選版本更新命令的受管理重新啟動路徑。

  範例：

  ```bash
  gh workflow run package-acceptance.yml --ref main -f workflow_ref=main -f source=npm -f package_spec=openclaw@beta -f suite_profile=product -f published_upgrade_survivor_baseline=openclaw@2026.4.26 -f telegram_mode=mock-openai
  ```

  常用設定檔：
  - `smoke`：安裝／頻道／代理程式、閘道網路，以及設定重新載入通道
  - `package`：不含 OpenWebUI 或即時 ClawHub 的成品原生套件／更新／重新啟動／外掛通道
  - `product`：套件設定檔，加上 MCP 頻道、排程／子代理程式清理、OpenAI 網頁搜尋，以及 OpenWebUI
  - `full`：含 OpenWebUI 的 Docker 發布路徑區塊
  - `custom`：用於聚焦重新執行的確切 `docker_lanes` 選取項目

- 若只需要發布候選版本的確定性一般 CI 涵蓋範圍，請直接執行手動 `CI` 工作流程。手動分派 CI 會略過變更範圍判定，並強制執行 Linux Node 分片、內建外掛分片、外掛與頻道契約分片、Node 22 相容性、`check-*`、`check-additional-*`、建置成品冒煙檢查、文件檢查、Python Skills、Windows、macOS，以及 Control UI 國際化通道。獨立的手動 CI 執行作業僅在使用 `include_android=true` 分派時執行 Android；`Full Release Validation` 會將該輸入傳遞給其 CI 子工作流程。

  ```bash
  gh workflow run ci.yml --ref release/YYYY.M.PATCH -f include_android=true
  ```

- 驗證發布遙測時，請執行 `pnpm qa:otel:smoke`。它會透過本機 OTLP/HTTP 接收器執行 QA-lab，並驗證追蹤、指標及記錄匯出，以及有限制的追蹤屬性與內容／識別碼遮蔽，而不需要 Opik、Langfuse 或其他外部收集器。
- 驗證收集器相容性時，請執行 `pnpm qa:otel:collector-smoke`。它會先將相同的 QA-lab OTLP 匯出資料路由通過真正的 OpenTelemetry Collector Docker 容器，再執行本機接收器判定。
- 驗證受保護的 Prometheus 抓取時，請執行 `pnpm qa:prometheus:smoke`。它會執行 QA-lab、拒絕未經驗證的抓取，並驗證發布關鍵指標系列不含提示詞內容、原始識別碼、驗證權杖及本機路徑。
- 若要連續執行原始碼簽出版本的 OpenTelemetry 與 Prometheus 冒煙通道，請執行 `pnpm qa:observability:smoke`。
- 每次建立帶標籤的發布版本前，請執行 `pnpm release:check`。
- `OpenClaw NPM Release` 預檢會在封裝 npm tarball 前產生相依套件發布證明。npm 公告漏洞閘門會阻擋發布。遞移資訊清單風險、相依套件擁有權／安裝介面及相依套件變更報告僅作為發布證明。相依套件變更報告會比較發布候選版本與上一個可到達的發布標籤。預檢會將相依套件證明上傳為 `openclaw-release-dependency-evidence-<tag>`，並將其嵌入已準備的 npm 預檢成品內的 `dependency-evidence/`。實際發布路徑會重複使用該預檢成品，然後將相同證明以 `openclaw-<version>-dependency-evidence.zip` 附加至 GitHub 發布版本。
- 標籤存在後，請執行 `OpenClaw Release Publish` 以進行會產生變更的發布序列。一般 beta 與穩定版發布應從受信任的 `main` 分派；發布標籤仍會選取確切的目標提交，且可能指向 `release/YYYY.M.PATCH`。Tideclaw alpha 發布仍保留在其對應的 alpha 分支上。請傳入成功的 OpenClaw npm `preflight_run_id`、成功的 `full_release_validation_run_id` 及確切的 `full_release_validation_run_attempt`，並保留預設的外掛發布範圍 `all-publishable`，除非是刻意執行聚焦修復。此工作流程會依序執行外掛 npm 發布、外掛 ClawHub 發布及 OpenClaw npm 發布，以避免在外部化外掛之前發布核心套件；Windows 與 Android 提升作業則會針對發布草稿頁面，與核心 npm 發布並行執行。發布重新執行可接續進度：對於已發布的核心 npm 版本，工作流程證明登錄檔 tarball 與標籤的預檢成品相符後，會略過核心分派；當發布版本已包含經驗證的成品契約時，也會略過 Windows／Android 提升，因此重試只會重新執行失敗的階段。僅針對外掛的聚焦修復需要 `plugin_publish_scope=selected` 與非空白的外掛清單。僅限外掛的 `all-publishable` 執行作業需要完整且不可變的預檢與完整發布驗證證明；部分證明會遭拒絕。
- 穩定版 `OpenClaw Release Publish` 需要在相符的非預發布 `openclaw/openclaw-windows-node` 發布版本存在後，提供確切的 `windows_node_tag`，以及候選版本已核准的 `windows_node_installer_digests` 對應表。在分派任何發布子工作流程前，它會驗證該來源發布版本已發布、並非預發布版本、包含必要的 x64／ARM64 安裝程式，且仍與該核准對應表相符。接著，它會在 OpenClaw 發布版本仍為草稿時分派 `Windows Node Release`，並原封不動地帶入固定的安裝程式摘要對應表。子工作流程會從該確切標籤下載已簽署的 Windows Hub 安裝程式、比對固定摘要、在 Windows 執行器上驗證其 Authenticode 簽章使用預期的 OpenClaw Foundation 簽署者、寫入 SHA-256 資訊清單，並將安裝程式與資訊清單上傳至標準 OpenClaw GitHub 發布版本；接著重新下載已提升的成品，驗證資訊清單成員資格與雜湊值。父工作流程會在發布前驗證目前的 x64、ARM64 及總和檢查碼成品契約。直接復原會先拒絕非預期的 `OpenClawCompanion-*` 成品名稱，再以固定的來源位元組取代預期的契約成品。

  僅在復原時手動分派 `Windows Node Release`，且一律傳入確切標籤，絕不可使用 `latest`，並提供來自已核准來源發布版本的明確 `expected_installer_digests` JSON 對應表。網站下載連結應指向目前穩定版本的確切 OpenClaw 發布成品 URL；或僅在驗證 GitHub 的最新版本重新導向指向相同發布版本後，才使用 `releases/latest/download/...`；請勿只連結至配套儲存庫的發布頁面。

- 發行檢查現在會在獨立的手動工作流程中執行：`OpenClaw Release Checks`。在核准發行前，它也會執行 QA Lab 模擬同等性工作區段，以及 Matrix 發行設定檔和 Telegram QA 工作區段。即時工作區段使用 `qa-live-shared` 環境；Telegram 還會使用 Convex CI 認證資訊租約。若要執行所有維護中的 Matrix 情境，請使用 `matrix_profile=all` 執行手動 `QA-Lab - All Lanes` 工作流程；該工作流程會將這項選擇分散至傳輸、媒體和 E2EE 設定檔，讓完整驗證能在各工作逾時限制內完成。
- 跨作業系統安裝與升級執行階段驗證是公開 `OpenClaw Release Checks` 和 `Full Release Validation` 的一部分，兩者會直接呼叫可重複使用的工作流程 `.github/workflows/openclaw-cross-os-release-checks-reusable.yml`。這項拆分是刻意的：讓實際 npm 發行路徑保持簡短、確定且聚焦於成品，而較慢的即時檢查則保留在各自的工作區段中，以免拖延或阻擋發布。
- 含有祕密的發行檢查應透過 `Full Release Validation` 分派，或從 `main`/release 工作流程參照分派，以確保工作流程邏輯和祕密維持受控。
- `OpenClaw Release Checks` 接受分支、標籤或完整提交 SHA，前提是解析出的提交可從 OpenClaw 分支或發行標籤觸及。
- `OpenClaw NPM Release` 的僅驗證預檢也接受目前完整的 40 字元工作流程分支提交 SHA，而不要求已推送的標籤。該 SHA 路徑僅供驗證，無法升級為實際發布。在 SHA 模式下，工作流程只會為套件中繼資料檢查合成 `v<package.json version>`；實際發布仍需要真正的發行標籤。
- 這兩個工作流程都會將實際發布和升級路徑保留在 GitHub 託管的執行器上，而不會變更狀態的驗證路徑則可使用較大型的 Blacksmith Linux 執行器。
- 該工作流程會同時使用 `OPENAI_API_KEY` 和 `ANTHROPIC_API_KEY` 工作流程祕密來執行 `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`。
- npm 發行預檢不再等待獨立的發行檢查工作區段。
- 在本機為候選版本加上標籤前，請執行 `RELEASE_TAG=vYYYY.M.PATCH-beta.N pnpm release:fast-pretag-check`。這個輔助工具會依序執行快速發行防護措施、外掛 npm/ClawHub 發行檢查、建置、UI 建置和 `release:openclaw:npm:check`，以便在 GitHub 發布工作流程啟動前，找出常見且會阻擋核准的錯誤。
- 請在核准前執行 `RELEASE_TAG=vYYYY.M.PATCH node --import tsx scripts/openclaw-npm-release-check.ts`（或相符的預發行／修正版標籤）。
- npm 發布後，請執行 `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.PATCH`（或相符的 beta／修正版版本），以在全新的暫存前綴中驗證已發布登錄檔的安裝路徑。
- beta 發布後，請執行 `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@YYYY.M.PATCH-beta.N OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci pnpm test:docker:npm-telegram-live`，使用共用的租用 Telegram 認證資訊集區，針對已發布的 npm 套件驗證已安裝套件的引導流程、Telegram 設定及實際 Telegram E2E。本機維護者的一次性執行可省略 Convex 變數，並直接傳入三項 `OPENCLAW_QA_TELEGRAM_*` 環境認證資訊。
- 若要從維護者機器執行完整的發布後 beta 冒煙測試，請使用 `pnpm release:beta-smoke -- --beta betaN`。這個輔助工具會執行 Parallels npm 更新／全新目標驗證、分派 `NPM Telegram Beta E2E`、輪詢確切的工作流程執行、下載成品，並輸出 Telegram 報告。
- 維護者可透過手動 `NPM Telegram Beta E2E` 工作流程，在 GitHub Actions 中執行相同的發布後檢查。它刻意僅允許手動執行，不會在每次合併時執行。
- 維護者發行自動化採用先預檢、再升級的方式：
  - 實際 npm 發布必須通過成功的 npm `preflight_run_id`。
  - 一般 beta 與穩定版的發布協調和預檢，會針對確切的目標標籤使用受信任的 `main`。Tideclaw alpha 發布和預檢則使用相符的 alpha 分支。
  - 穩定版 npm 發行預設為 `beta`；穩定版 npm 發布可透過工作流程輸入明確指定 `latest`。
  - 以權杖為基礎的 npm dist-tag 變更位於 `openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml`，因為 `npm dist-tag add` 仍需要 `NPM_TOKEN`，而來源儲存庫則維持僅使用 OIDC 發布。
  - 公開的 `macOS Release` 僅供驗證；若標籤只存在於發行分支上，但工作流程是從 `main` 分派，請設定 `public_release_branch=release/YYYY.M.PATCH`。
  - 實際 macOS 發布必須通過成功的 macOS `preflight_run_id` 和 `validate_run_id`。
  - 實際發布路徑會升級已準備好的成品，而不是再次重建。
- 對於 `YYYY.M.PATCH-N` 之類的穩定版修正發行，發布後驗證器也會檢查從 `YYYY.M.PATCH` 到 `YYYY.M.PATCH-N` 的相同暫存前綴升級路徑，確保發行修正不會在未察覺的情況下，讓較舊的全域安裝仍停留在基礎穩定版內容。
- 除非 tarball 同時包含 `dist/control-ui/index.html` 和非空的 `dist/control-ui/assets/` 內容，否則 npm 發行預檢會以關閉方式失敗，避免再次交付空白的瀏覽器儀表板。
- 發布後驗證也會檢查已發布外掛進入點和套件中繼資料是否存在於已安裝的登錄檔配置中。若發行版本缺少外掛執行階段內容，發布後驗證器會判定失敗，且無法升級至 `latest`。
- `pnpm test:install:smoke` 也會對候選更新 tarball 強制執行 npm pack `unpackedSize` 預算，因此安裝程式 E2E 能在發行發布路徑前抓出意外的套件膨脹。
- 若發行工作涉及 CI 規劃、擴充功能計時資訊清單或擴充功能測試矩陣，請在核准前從 `.github/workflows/plugin-prerelease.yml` 重新產生並審查由規劃器擁有的 `plugin-prerelease-extension-shard` 矩陣輸出，避免發行說明描述過時的 CI 配置。
- 穩定版 macOS 發行就緒檢查也包含更新程式介面：GitHub 發行最終必須包含已封裝的 `.zip`、`.dmg` 和 `.dSYM.zip`；發布後，`main` 上的 `appcast.xml` 必須指向新的穩定版 zip（macOS 發布工作流程會自動提交它，若直接推送遭阻擋，則會開啟 appcast PR）；已封裝的應用程式必須保留非偵錯套件識別碼、非空的 Sparkle 摘要 URL，以及不低於該發行版本之標準 Sparkle 建置下限的 `CFBundleVersion`。

## 發行測試機器

`Full Release Validation` 是操作人員從單一進入點啟動完整產品矩陣的方式。請使用這個輔助工具，讓每個子工作流程都從固定於單一受信任 `main` 工作流程 SHA 的暫存分支執行，同時將要求的提交維持為受測候選版本：

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

這個輔助工具會擷取目前的 `origin/main`、在該受信任工作流程提交上推送 `release-ci/<workflow-sha>-...`、針對 alpha/beta 套件版本推斷 `beta`，其他版本則推斷 `stable`，接著使用 `ref=<target-sha>` 從暫存分支分派 `Full Release Validation`，驗證每個子工作流程的 `headSha` 都與固定的父工作流程 SHA 相符，然後刪除暫存分支。傳入 `-f reuse_evidence=false` 可強制執行全新作業，傳入 `-f release_profile=full` 可執行廣泛的建議性掃描，或傳入 `--workflow-sha <trusted-main-sha>` 以固定仍可從目前 `origin/main` 觸及的較舊提交。工作流程本身絕不寫入儲存庫參照。這樣可在不將工具提交加入候選版本的情況下，使用僅存在於 main 的發行工具，並避免意外驗證較新的 `main` 子工作流程執行。

Code SHA 顯示綠燈後，只提交 `CHANGELOG.md`，並使用 Release SHA 執行相同的輔助工具：

```bash
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH
```

只有當 GitHub 證明 Release SHA 衍生自 Code SHA，且完整的變更路徑集合恰好是 `CHANGELOG.md` 時，第二個父工作流程才會重複使用產品證據。它會記錄 `changelog-only-release-v1`，且不分派任何產品子工作流程。npm 預檢和套件／安裝驗收仍會在 Release SHA 上執行，因為其 tarball 位元組已變更。

對於全新的 Code SHA，工作流程會解析目標、分派手動 `CI`，接著分派 `OpenClaw Release Checks`。`OpenClaw Release Checks` 會展開執行安裝冒煙測試、跨作業系統發行檢查、啟用浸泡測試時的即時／E2E Docker 發行路徑涵蓋、含標準 Telegram 套件 E2E 的套件驗收、QA Lab 同等性、即時 Matrix，以及即時 Telegram。完整／全部執行僅在 `Full Release Validation` 摘要顯示 `normal_ci`、`plugin_prerelease` 和 `release_checks` 均成功時才可接受，除非聚焦的重新執行刻意略過獨立的 `Plugin Prerelease` 子工作流程。只有在使用 `release_package_spec` 或 `npm_telegram_package_spec` 進行聚焦的已發布套件重新執行時，才使用獨立的 `npm-telegram` 子工作流程。最終驗證器摘要包含各子工作流程執行的最慢工作表格，因此發行管理者不必下載記錄，即可查看目前的關鍵路徑。

在這條發行路徑中，產品效能子工作流程僅產生成品。
總括工作流程會使用 `publish_reports=false` 分派它，除非其僅成品防護措施證明 Clawgrit 報告發布器維持略過狀態，
否則驗證會遭拒絕。

如需完整階段矩陣、確切的工作流程工作名稱、穩定版與完整設定檔的差異、成品及聚焦的重新執行控制代碼，請參閱[完整發行驗證](/zh-TW/reference/full-release-validation)。

子工作流程會從執行 `Full Release Validation`、固定於 SHA 的受信任參照分派。每個子工作流程執行都必須使用確切的父工作流程 SHA。請勿使用原始的 `--ref main -f ref=<sha>` 分派作為發行證據；請使用 `pnpm ci:full-release --sha <target-sha> --target-ref release/YYYY.M.PATCH`。

使用 `release_profile` 選擇即時／提供者涵蓋範圍：

- `beta`：最快的發行關鍵 OpenAI／核心即時與 Docker 路徑
- `stable`：供發行核准使用的 beta 加穩定版提供者／後端涵蓋
- `full`：穩定版加廣泛的建議性提供者／媒體涵蓋

穩定版和完整驗證在升級前一律會執行詳盡的即時／E2E、Docker 發行路徑，以及有界限的已發布版本升級存續掃描。使用 `run_release_soak=true` 可為 beta 要求相同掃描。該掃描涵蓋最新四個穩定版套件、固定的 `2026.4.23` 和 `2026.5.2` 基準，以及較舊的 `2026.4.15` 涵蓋；其中會移除重複基準，並將每個基準分片至各自的 Docker 執行器工作。

`OpenClaw Release Checks` 會使用受信任的工作流程參照，將目標參照解析一次為 `release-package-under-test`，並在執行浸泡測試時，於跨作業系統、套件驗收和發行路徑 Docker 檢查中重複使用該成品。如此可讓所有面向套件的測試機器使用相同位元組，並避免重複建置套件。beta 已發布至 npm 後，請設定 `release_package_spec=openclaw@YYYY.M.PATCH-beta.N`，讓發行檢查只下載一次已交付套件、從 `dist/build-info.json` 擷取其建置來源 SHA，並在跨作業系統、套件驗收、發行路徑 Docker 和套件 Telegram 工作區段中重複使用該成品。

當儲存庫／組織變數已設定時，跨作業系統 OpenAI 安裝冒煙測試會使用 `OPENCLAW_CROSS_OS_OPENAI_MODEL`，否則使用 `openai/gpt-5.6-luna`，因為此工作區段要驗證的是套件安裝、引導流程、閘道啟動和一次即時代理程式回合，而不是評測功能最強大的模型。較廣泛的即時提供者矩陣仍是進行模型特定涵蓋的地方。

請依發行階段使用以下變體：

```bash
# 驗證產品完整的 Code SHA。
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH

# 重複使用 Code SHA 的產品證據，驗證僅變更日誌的 Release SHA。
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH

# 發布 beta 後，新增已發布套件的 Telegram E2E。
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH \
  -f release_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f evidence_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f npm_telegram_provider_mode=mock-openai
```

在針對性修正後，第一次重新執行時不要使用完整的傘狀流程。若其中一個執行環境失敗，請在下一次驗證中使用失敗的子工作流程、作業、Docker 通道、套件設定檔、模型供應商或 QA 通道。只有當修正變更了共用的發布協調流程，或使先前所有執行環境的證據失效時，才再次執行完整傘狀流程。傘狀流程的最終驗證器會重新檢查記錄的子工作流程執行 ID，因此成功重新執行子工作流程後，只需重新執行失敗的 `Verify full validation` 父作業。

當發布設定檔、有效的浸泡測試設定及驗證輸入相符，且目標 SHA
相同，或新目標是其後代且完整變更路徑集合恰好為
`CHANGELOG.md` 時，`rerun_group=all` 可重複使用先前成功的傘狀流程執行結果。完全相同目標的重複使用會記錄
`exact-target-full-validation-v1`；驗證後的 Release SHA 會記錄
`changelog-only-release-v1`。後者只會重複使用產品驗證。Npm
預檢、套件位元組、版本說明來源，以及安裝／更新驗收
仍必須針對 Release SHA 執行。任何版本、原始碼、產生內容、
相依套件、套件或工作流程所擁有的目標變更，都需要新的 Code SHA
及全新的完整驗證。同一個 `release/*` ref 和
重新執行群組的較新傘狀流程執行，會自動取代進行中的執行。傳入
`reuse_evidence=false` 可強制執行全新的完整流程。

若要進行範圍有限的復原，請將 `rerun_group` 傳入傘狀流程。`all` 是真正的候選發布執行，`ci` 只執行一般 CI 子流程，`plugin-prerelease` 只執行發布專用的外掛子流程，`release-checks` 執行所有發布執行環境，而範圍較小的發布群組為 `install-smoke`、`cross-os`、`live-e2e`、`package`、`qa`、`qa-parity`、`qa-live` 和 `npm-telegram`。針對性的 `npm-telegram` 重新執行需要 `release_package_spec` 或 `npm_telegram_package_spec`；完整／全部執行會使用 Package Acceptance 內的標準套件 Telegram E2E。針對性的跨作業系統重新執行可加入 `cross_os_suite_filter=windows/packaged-upgrade` 或其他作業系統／測試套件篩選器。QA 發布檢查失敗會阻擋一般發布驗證，包括核心執行階段配對通道中的 OpenClaw 動態工具漂移。Tideclaw alpha 執行仍可將不涉及套件安全性的發布檢查通道視為諮詢性檢查。使用 `release_profile=beta` 時，`Run repo/live E2E validation` 即時供應商測試套件屬於諮詢性檢查（只發出警告，不會阻擋）；stable 和 full 設定檔仍會將其視為阻擋條件。當 `live_suite_filter` 明確要求有閘門控管的 QA 即時通道（例如 Discord、WhatsApp 或 Slack）時，必須啟用相符的 `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED` 儲存庫變數；否則輸入擷取會失敗，而不是默默跳過該通道。

### Vitest

Vitest 執行環境是手動 `CI` 子工作流程。手動 CI 會刻意略過變更範圍限定，並強制針對候選發布執行一般測試圖：Linux 節點分片、隨附外掛分片、外掛與頻道合約分片、節點 22 相容性、`check-*`、`check-additional-*`、建置成品冒煙檢查、文件檢查、Python skills、Windows、macOS，以及 Control UI i18n。當 `Full Release Validation` 執行此環境時，也會包含 Android，因為傘狀流程會傳入 `include_android=true`；獨立的手動 CI 需要 `include_android=true` 才會涵蓋 Android。

使用此執行環境回答「原始碼樹是否通過完整的一般測試套件？」這與發布路徑的產品驗證不同。應保留的證據：

- `Full Release Validation` 摘要，顯示已派送的 `CI` 執行 URL
- `CI` 在確切目標 SHA 上成功執行
- 調查迴歸問題時，CI 作業中失敗或緩慢的分片名稱
- 需要分析執行效能時，保留 Vitest 計時成品，例如 `.artifacts/vitest-shard-timings.json`

只有當發布需要具確定性的一般 CI，但不需要 Docker、QA Lab、即時、跨作業系統或套件執行環境時，才直接執行手動 CI。直接執行非 Android CI 時使用第一個命令。當直接執行的候選發布 CI 必須涵蓋 Android 時，加入 `include_android=true`：

```bash
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH -f include_android=true
```

### Docker

Docker 執行環境位於 `OpenClaw Release Checks` 至 `openclaw-live-and-e2e-checks-reusable.yml`，另包含發布模式的 `install-smoke` 工作流程。它透過封裝後的 Docker 環境驗證候選發布，而不只是執行原始碼層級測試。

發布 Docker 涵蓋範圍包括：

- 完整安裝冒煙測試，並啟用緩慢的 Bun 全域安裝冒煙測試
- 依目標 SHA 準備／重複使用根 Dockerfile 冒煙映像，QR、根目錄／閘道及安裝程式／Bun 冒煙作業會以個別的安裝冒煙分片執行
- 儲存庫 E2E 通道
- 發布路徑 Docker 區塊：`core`、`package-update-openai`、`package-update-anthropic`、`package-update-core`、`plugins-runtime-plugins`、`plugins-runtime-services`、`plugins-runtime-install-a` 至 `plugins-runtime-install-h`，以及 `openwebui`
- 要求時，在專用的大容量磁碟執行器上執行 OpenWebUI 涵蓋測試
- 拆分的隨附外掛安裝／解除安裝通道 `bundled-plugin-install-uninstall-0` 至 `bundled-plugin-install-uninstall-23`
- 當發布檢查包含即時測試套件時，執行即時／E2E 供應商測試套件及 Docker 即時模型涵蓋測試

重新執行前請先使用 Docker 成品。發布路徑排程器會上傳 `.artifacts/docker-tests/`，其中包含通道記錄、`summary.json`、`failures.json`、階段計時、排程器計畫 JSON，以及重新執行命令。若要進行針對性復原，請在可重複使用的即時／E2E 工作流程上使用 `docker_lanes=<lane[,lane]>`，而不是重新執行所有發布區塊。產生的重新執行命令會在可用時包含先前的 `package_artifact_run_id` 及已準備的 Docker 映像輸入，因此失敗的通道可以重複使用相同的 tarball 和 GHCR 映像。

### QA Lab

QA Lab 執行環境也是 `OpenClaw Release Checks` 的一部分。它是代理式行為及頻道層級的發布閘門，與 Vitest 和 Docker 套件機制分開。

發布 QA Lab 涵蓋範圍包括：

- 模擬同等性通道，使用代理式同等性套件比較 OpenAI 候選通道與 `anthropic/claude-opus-4-8` 基準
- 使用 `qa-live-shared` 環境的 Matrix 即時轉接器發布設定檔
- 使用 Convex CI 認證資訊租約的即時 Telegram QA 通道
- 當發布遙測需要明確的本機驗證時，使用 `pnpm qa:otel:smoke`、`pnpm qa:otel:collector-smoke`、`pnpm qa:prometheus:smoke` 或 `pnpm qa:observability:smoke`

使用此執行環境回答「此發布在 QA 情境及即時頻道流程中的行為是否正確？」核准發布時，請保留同等性、Matrix 和 Telegram 通道的成品 URL。完整的 Matrix 涵蓋測試仍可透過手動分片 QA-Lab 執行取得，而不屬於預設的發布關鍵通道。

### 套件

Package 執行環境是可安裝產品的閘門。它由 `Package Acceptance` 和解析器 `scripts/resolve-openclaw-package-candidate.mjs` 支援。解析器會將候選版本標準化為 Docker E2E 使用的 `package-under-test` tarball、驗證套件清單、記錄套件版本及 SHA-256，並將工作流程測試框架 ref 與套件來源 ref 分開保存。

支援的候選來源：

- `source=npm`：`openclaw@beta`、`openclaw@latest` 或確切的 OpenClaw 發布版本
- `source=ref`：使用所選的 `workflow_ref` 測試框架，封裝受信任的 `package_ref` 分支、標籤或完整提交 SHA
- `source=url`：下載需要 `package_sha256` 的公開 HTTPS `.tgz`；系統會拒絕 URL 認證資訊、非預設 HTTPS 連接埠、私有／內部／特殊用途主機名稱或解析後位址，以及不安全的重新導向
- `source=trusted-url`：使用 `.github/package-trusted-sources.json` 中具名原則所要求的 `package_sha256` 和 `trusted_source_id`，下載 HTTPS `.tgz`；對於維護者擁有的企業鏡像或私有套件儲存庫，請使用此方式，而不是在 `source=url` 中新增輸入層級的私有網路略過機制
- `source=artifact`：重複使用另一個 GitHub Actions 執行所上傳的 `.tgz`

`OpenClaw Release Checks` 使用 `source=artifact`、已準備的發布套件成品、`suite_profile=custom`、`docker_lanes=doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape`、`telegram_mode=mock-openai` 執行 Package Acceptance。Package Acceptance 會針對同一個解析後的 tarball，執行遷移、更新、由 root 管理的 VPS 升級、已設定驗證的更新後重新啟動、即時 ClawHub skill 安裝、過時外掛相依套件清理、離線外掛測試資料、外掛更新、外掛命令繫結跳脫強化，以及 Telegram 套件 QA。具阻擋性的發布檢查預設使用最新的已發布套件基準；搭配 `run_release_soak=true`、`release_profile=stable` 或 `release_profile=full` 的 beta 設定檔，會以 `reported-issues` 情境，將已發布升級存續測試擴展至 `last-stable-4`，以及固定的 `2026.4.23`、`2026.5.2` 和 `2026.4.15` 基準。對於已經出貨的候選版本，請使用搭配 `source=npm` 的 Package Acceptance；對於發布前由 SHA 支援的本機 npm tarball，使用 `source=ref`；對於維護者擁有的企業／私有鏡像，使用 `source=trusted-url`；對於由另一個 GitHub Actions 執行上傳的已準備 tarball，則使用 `source=artifact`。

這是 GitHub 原生的替代方案，可取代先前需要 Parallels 才能完成的大多數套件／更新涵蓋測試。跨作業系統發布檢查對特定作業系統的初始設定、安裝程式及平台行為仍然重要，但套件／更新產品驗證應優先使用 Package Acceptance。

更新與外掛驗證的標準檢查清單是[測試更新與外掛](/zh-TW/help/testing-updates-plugins)。在判斷應使用哪個本機、Docker、Package Acceptance 或發布檢查通道來驗證外掛安裝／更新、doctor 清理或已發布套件遷移變更時，請使用此清單。從每個穩定版 `2026.4.23+` 套件進行的完整已發布更新遷移，是獨立的手動 `Update Migration` 工作流程，不屬於 Full Release CI。

舊版 Package Acceptance 的寬容處理刻意設有時間限制。直到 `2026.4.25` 為止的套件，可針對已發布至 npm 的中繼資料缺漏使用相容性路徑：tarball 中缺少私有 QA 清單項目、缺少 `gateway install --wrapper`、從 tarball 衍生的 git 測試資料中缺少修補檔案、缺少持久保存的 `update.channel`、舊版外掛安裝記錄位置、缺少市集安裝記錄持久保存，以及在 `plugins update` 期間進行設定中繼資料遷移。已發布的 `2026.4.26` 套件可能會針對已經出貨的本機建置中繼資料戳記檔案發出警告。後續套件必須符合現代套件合約；相同缺漏將導致發布驗證失敗。

當發布問題涉及實際可安裝套件時，請使用範圍較廣的 Package Acceptance 設定檔：

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f published_upgrade_survivor_baseline=openclaw@2026.4.26
```

常見套件設定檔：

- `smoke`：快速套件安裝／頻道／代理程式、閘道網路及設定重新載入流程
- `package`：安裝／更新／重新啟動／外掛套件合約，加上即時 ClawHub Skill 安裝驗證；這是版本檢查的預設值
- `product`：`package` 加上 MCP 頻道、排程／子代理程式清理、OpenAI 網頁搜尋及 OpenWebUI
- `full`：包含 OpenWebUI 的 Docker 發布路徑區塊
- `custom`：用於聚焦重新執行的確切 `docker_lanes` 清單

若要進行套件候選版本的 Telegram 驗證，請在 Package Acceptance 上啟用 `telegram_mode=mock-openai` 或 `telegram_mode=live-frontier`。工作流程會將解析後的 `package-under-test` tarball 傳入 Telegram 流程；獨立的 Telegram 工作流程仍接受已發布的 npm 規格，以進行發布後檢查。

## 一般版本發布自動化

針對 beta、`latest`、外掛、GitHub Release 及平台發布，
`OpenClaw Release Publish` 是一般會進行變更的進入點。每月的
`.33+` 閘道延伸穩定版路徑不使用此協調器。
一般工作流程會依照版本發布所需的順序，協調受信任發布者工作流程：

1. 簽出版本標籤並解析其提交 SHA。
2. 確認標籤可從 `main` 或 `release/*` 觸及（若為 alpha 預發行版本，也可以是 Tideclaw alpha 分支）。
3. 執行 `pnpm plugins:sync:check`。
4. 使用 `publish_scope=all-publishable` 和 `ref=<release-sha>` 分派 `Plugin NPM Release`。
5. 使用相同範圍和 SHA 分派 `Plugin ClawHub Release`。
6. 確認已儲存的 `full_release_validation_run_id` 和確切執行嘗試次數後，使用版本標籤、npm dist-tag 及已儲存的 `preflight_run_id` 分派 `OpenClaw NPM Release`。
7. 針對穩定版本，以草稿形式建立或更新 GitHub 版本，使用明確的 `windows_node_tag` 和候選版本已核准的 `windows_node_installer_digests` 分派 `Windows Node Release`，並驗證正式 Windows 安裝程式／總和檢查碼資產。同時分派 `Android Release`，以建置確切標籤的已簽署 APK、總和檢查碼及來源證明。發布草稿前，請驗證這兩項原生資產合約。

Beta 發布範例：

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH-beta.N \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

將穩定版本發布至預設 beta dist-tag：

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

直接將穩定版本提升至 `latest` 必須明確指定：

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=latest
```

僅在進行聚焦修復或重新發布工作時，才使用較低階的 `Plugin NPM Release` 和 `Plugin ClawHub Release` 工作流程。當 `publish_openclaw_npm=true` 時，`OpenClaw Release Publish` 會拒絕 `plugin_publish_scope=selected`，因此核心套件若未包含所有可發布的官方外掛（包括 `@openclaw/diffs-language-pack`），便無法發布。若要修復所選外掛，請搭配 `plugin_publish_scope=selected` 和 `plugins=@openclaw/name` 設定 `publish_openclaw_npm=false`，或直接分派子工作流程。

首次發布的 ClawHub 啟動程序是例外：請從受信任的 `main`
分派 `Plugin ClawHub New`，並透過 `ref` 傳入完整的目標版本 SHA。
絕不可從版本標籤或分支執行啟動程序工作流程本身：

```bash
gh workflow run plugin-clawhub-new.yml \
  --ref main \
  -f plugins=@openclaw/name \
  -f ref=<full-40-character-release-sha> \
  -f pretag_validation=true \
  -f dry_run=true
```

標記前驗證需要 `dry_run=true`、拒絕版本標籤及父執行
輸入，且只接受可從 `main` 或 `release/*` 觸及的確切目標。
它不會載入 ClawHub 認證資訊、發布套件位元組，或變更受信任
發布者設定。此工作流程仍會解析即時登錄計畫，
只在無機密資訊的工作中簽出並封裝目標、具體化
已鎖定的 ClawHub 工具鏈，並在版本標籤存在前驗證不可變資產及套件
slug／身分。只有在無機密資訊的封裝工作
完成後，才核准 `clawhub-plugin-bootstrap` 環境；此受保護驗證工作不含認證資訊或變更命令。

標記後已核准的試執行或實際啟動程序，必須包含確切的
版本標籤，以及父 `OpenClaw Release Publish` 的執行 ID、嘗試次數及
分支。父工作流程會證明其自身的工作流程 SHA，以及 `Plugin ClawHub New` 的另一個確切受信任
`main` SHA；子執行和每項受保護
環境核准都必須符合該已核准的子 SHA。每次嘗試發布及變更受信任發布者前，都會
重新檢查版本標籤。

封裝工作會
上傳一項不可變資產，其名稱、Actions 資產 ID／摘要、
產生者執行／嘗試次數、目標 SHA，以及每個套件 tarball 的 SHA-256／大小，
都會傳入驗證和受保護工作。受保護工作只會簽出受信任的 `main`
工具、透過 GitHub API 驗證資產組合、依確切資產 ID 下載、
重新雜湊每個 tarball，並使用已釘選命令列介面的 USTAR 正規化規則驗證本機 TAR 路徑及
套件身分。接著每個候選項目都會通過已釘選命令列介面的發布試執行；該試執行會在
查詢登錄或驗證身分前返回。認證資訊工作的預先篩選會將壓縮後的 ClawPack
限制為 120 MiB、檔案承載總量限制為 50 MiB、展開後的 TAR 資料限制為 64 MiB，且
TAR 項目數限制為 10,000。既有套件的受信任發布者修復仍
只進行設定，但在變更受信任發布者
設定前，仍會封裝目標，並要求所請求的標籤與確切登錄位元組及中繼資料完全相同。
發布後驗證會下載 ClawHub 資產，並
要求相同的 SHA-256 和大小。僅當確切的產生者工作已
成功完成時，重新執行失敗工作的復原程序才能重用較早
嘗試的套件資產。最終證據也會繫結已鎖定的 ClawHub 版本、鎖定檔
SHA-256 及 npm 完整性。若不相符，則需要新的套件版本。

## NPM 工作流程輸入

`OpenClaw NPM Release` 接受以下由操作人員控制的輸入：

- `tag`：必要的版本標籤，例如 `v2026.4.2`、`v2026.4.2-1`、`v2026.4.2-beta.1` 或 `v2026.4.2-alpha.1`；當 `preflight_only=true` 時，也可以是目前完整的 40 字元工作流程分支提交 SHA，僅用於驗證預檢
- `preflight_only`：`true` 僅用於驗證／建置／封裝，`false` 用於實際發布路徑
- `preflight_run_id`：既有且成功的預檢執行 ID；實際發布路徑需要此項，工作流程才能重用已準備的 tarball，而非重新建置
- `full_release_validation_run_id`：此標籤／SHA 的成功 `Full Release Validation` 執行 ID，實際發布時需要。Beta 發布可只憑預檢繼續並顯示警告，但穩定版／`latest` 提升仍需要此項。
- `full_release_validation_run_attempt`：與 `full_release_validation_run_id` 配對的確切正整數執行嘗試次數；只要提供執行 ID，就必須提供此項，確保重新執行無法在發布期間變更授權證據。
- `release_publish_run_id`：已核准的 `OpenClaw Release Publish` 執行 ID；當此工作流程由該父工作流程分派時需要（機器人執行者的實際發布呼叫）
- `plugin_npm_run_id`：成功且與確切 HEAD 相符的 `Plugin NPM Release` 執行 ID；實際發布 `extended-stable` 核心套件時需要
- `npm_dist_tag`：發布路徑的 npm 目標標籤；接受 `alpha`、`beta`、`latest` 或 `extended-stable`，預設為 `beta`。最終修補版本 `33` 及之後版本必須使用 `extended-stable`；根據預設，`extended-stable` 會拒絕較早的修補版本，且一律拒絕非最終標籤。
- `bypass_extended_stable_guard`：僅供測試的布林值，預設為 `false`；搭配 `npm_dist_tag=extended-stable` 時，會略過每月延伸穩定版資格，同時保留版本身分、資產、核准及回讀檢查。

`Plugin NPM Release` 接受 `npm_dist_tag=default` 以使用既有版本發布
行為，或接受 `npm_dist_tag=extended-stable` 以使用受防護的每月路徑。
延伸穩定版選項需要 `publish_scope=all-publishable`、空白的
`plugins` 輸入、等於或高於 `33` 的最終修補版本，以及位於確切分支尖端的正式
`extended-stable/YYYY.M.33` 分支。它絕不會移動外掛
`latest` 或 `beta`。新的套件版本會透過 OIDC 受信任發布
（`npm publish --tag extended-stable`）以不可分割方式取得 `extended-stable`；此
來源工作流程不使用權杖驗證的 `npm dist-tag add`。重試時
會略過 npm 中已存在的確切版本，接著採取封閉式失敗，除非完整
回讀確認每個確切套件及 `extended-stable` 標籤均已收斂。

`OpenClaw Release Publish` 接受以下由操作人員控制的輸入：

- `tag`：必要的版本標籤；必須已存在
- `preflight_run_id`：成功的 `OpenClaw NPM Release` 預檢執行 ID；當 `publish_openclaw_npm=true` 或 `plugin_publish_scope=all-publishable` 時需要
- `full_release_validation_run_id`：成功的 `Full Release Validation` 執行 ID；當 `publish_openclaw_npm=true` 或 `plugin_publish_scope=all-publishable` 時需要
- `full_release_validation_run_attempt`：與 `full_release_validation_run_id` 配對的確切正整數嘗試次數；只要提供執行 ID，就必須提供此項
- `windows_node_tag`：確切且非預發行的 `openclaw/openclaw-windows-node` 版本標籤；發布 OpenClaw 穩定版本時需要
- `windows_node_installer_digests`：經候選版本核准的精簡 JSON 對應，將目前 Windows 安裝程式名稱對應至已釘選的 `sha256:` 摘要；發布 OpenClaw 穩定版本時需要
- `npm_telegram_run_id`：選用的成功 `NPM Telegram Beta E2E` 執行 ID，用於納入最終版本發布證據
- `npm_dist_tag`：OpenClaw 套件的 npm 目標標籤，為 `alpha`、`beta` 或 `latest` 之一
- `plugin_publish_scope`：預設為 `all-publishable`；只有在搭配 `publish_openclaw_npm=false` 進行聚焦的純外掛修復工作時，才使用 `selected`
- `plugins`：當 `plugin_publish_scope=selected` 時，以逗號分隔的 `@openclaw/*` 套件名稱
- `publish_openclaw_npm`：預設為 `true`；只有在將工作流程用作純外掛修復協調器時，才設定 `false`
- `release_profile`：用於版本發布證據摘要的版本涵蓋設定檔；預設為 `from-validation`，會從驗證資訊清單讀取，或使用 `beta`、`stable` 或 `full` 覆寫
- `wait_for_clawhub`：預設為 `false`，因此 npm 可用性不會受 ClawHub 輔助流程阻擋；只有當工作流程完成必須包括 ClawHub 完成時，才設定 `true`

`OpenClaw Release Checks` 接受以下由操作人員控制的輸入：

- `ref`：要驗證的分支、標籤或完整提交 SHA。包含機密資訊的檢查要求解析後的提交可從 OpenClaw 分支或發行標籤存取。
- `run_release_soak`：選擇啟用詳盡的即時／E2E、Docker 發行路徑，以及針對 Beta 發行檢查的所有歷來版本升級存續浸泡測試。`release_profile=stable` 和 `release_profile=full` 會強制啟用此選項。

規則：

- 修補版本低於 `33` 的一般正式版與修正版，可發佈至 `beta` 或 `latest`。修補版本為 `33` 或以上的正式版必須發佈至 `extended-stable`，並且會拒絕位於該邊界的修正後綴版本。
- Beta 預發行標籤只能發佈至 `beta`；Alpha 預發行標籤只能發佈至 `alpha`
- 對於 `OpenClaw NPM Release`，僅在 `preflight_only=true` 時才允許輸入完整提交 SHA
- `OpenClaw Release Checks` 和 `Full Release Validation` 一律僅供驗證
- 實際發佈路徑必須使用預檢期間所用的相同 `npm_dist_tag`；工作流程會先驗證該中繼資料，再繼續發佈

## 一般 Beta／最新穩定版發行順序

此舊版順序適用於也負責外掛、GitHub Release、Windows 及其他平台工作的常規協調式發行。這不是本頁頂端所記載的每月 `.33+` 閘道延伸穩定版路徑。

建立常規協調式穩定版發行時：

1. 使用 `preflight_only=true` 執行 `OpenClaw NPM Release`。在標籤存在之前，可使用目前工作流程分支的完整提交 SHA，對預檢工作流程執行僅供驗證的試執行。
2. 一般先發佈 Beta 的流程請選擇 `npm_dist_tag=beta`；只有在刻意要直接發佈穩定版時，才選擇 `latest`。
3. 若要透過單一手動工作流程取得一般 CI，以及即時提示快取、Docker、QA Lab、Matrix 和 Telegram 的涵蓋範圍，請在發行分支、發行標籤或完整提交 SHA 上執行 `Full Release Validation`。如果刻意只需要具決定性的一般測試圖，請改在發行參照上執行手動 `CI` 工作流程。
4. 選取其已簽署 x64 和 ARM64 安裝程式應出貨的確切非預發行 `openclaw/openclaw-windows-node` 發行標籤。將其儲存為 `windows_node_tag`，並將已驗證的摘要對應表儲存為 `windows_node_installer_digests`。候選發行版輔助工具會記錄兩者，並將其納入所產生的發佈命令。
5. 儲存成功的 `preflight_run_id`、`full_release_validation_run_id`，以及確切的 `full_release_validation_run_attempt`。
6. 從受信任的 `main` 執行 `OpenClaw Release Publish`，並使用相同的 `tag`、相同的 `npm_dist_tag`、所選的 `windows_node_tag`、其已儲存的 `windows_node_installer_digests`、已儲存的 `preflight_run_id`、`full_release_validation_run_id` 和 `full_release_validation_run_attempt`。它會先將外部化的外掛發佈至 npm 和 ClawHub，再提升 OpenClaw npm 套件。
7. 如果發行版已發佈至 `beta`，請使用 `openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml` 工作流程，將該穩定版本從 `beta` 提升至 `latest`。
8. 如果發行版刻意直接發佈至 `latest`，而 `beta` 應立即跟隨相同的穩定版建置，請使用同一發行工作流程，將兩個 dist-tag 都指向該穩定版本；或讓其排程的自我修復同步稍後移動 `beta`。

dist-tag 變更位於發行帳本儲存庫中，因為它仍需要 `NPM_TOKEN`，而原始碼儲存庫則維持僅使用 OIDC 發佈。如此可讓直接發佈路徑和先發佈 Beta 的提升路徑都有文件記載，且操作人員可見。

如果維護者必須退回使用本機 npm 驗證，請僅在專用的 tmux 工作階段中執行任何 1Password 命令列介面（`op`）命令。請勿直接從主要代理程式 Shell 呼叫 `op`；將其保留在 tmux 中，可讓提示、警示和 OTP 處理過程保持可觀察，並避免重複觸發主機警示。

## 公開參考資料

- [`.github/workflows/full-release-validation.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/full-release-validation.yml)
- [`.github/workflows/package-acceptance.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/package-acceptance.yml)
- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`.github/workflows/openclaw-cross-os-release-checks-reusable.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-cross-os-release-checks-reusable.yml)
- [`.github/workflows/docker-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/docker-release.yml)
- [`scripts/resolve-openclaw-package-candidate.mjs`](https://github.com/openclaw/openclaw/blob/main/scripts/resolve-openclaw-package-candidate.mjs)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

維護者會使用 [`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md) 中的私有發行文件作為實際操作手冊。

## 相關內容

- [發行通道](/zh-TW/install/development-channels)
