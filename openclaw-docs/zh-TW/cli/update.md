---
read_when:
    - 你想要安全地更新原始碼工作目錄
    - 你正在偵錯 `openclaw update` 輸出或選項
    - 你需要了解 `--update` 簡寫行為
summary: '`openclaw update` 的命令列介面參考（相對安全的原始碼更新 + 閘道自動重新啟動）'
title: 更新
x-i18n:
    generated_at: "2026-07-26T07:48:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b46696f6b9cba5c318f870bcb6c5ea8e0652940968da2ad85e86709fe4c11146
    source_path: cli/update.md
    workflow: 16
---

# `openclaw update`

更新 OpenClaw，並在 stable/extended-stable/beta/dev 頻道之間切換。

如果你是透過 **npm/pnpm/bun** 安裝（全域安裝，沒有 git 中繼資料），
更新會依照[更新](/zh-TW/install/updating)中所述的套件管理器流程進行。

## 使用方式

```bash
openclaw update
openclaw update status
openclaw update repair
openclaw update wizard
openclaw update --channel extended-stable
openclaw update --channel beta
openclaw update --channel dev
openclaw update --tag beta
openclaw update --tag main
openclaw update --dry-run
openclaw update --no-restart
openclaw update --yes
openclaw update --acknowledge-clawhub-risk
openclaw update --json
openclaw --update
```

`openclaw --update` 會改寫為 `openclaw update`（適用於 shell 和
啟動器指令碼）。

## 選項

| 旗標                                             | 說明                                                                                                                                                                                                                                                                                                                                  |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--no-restart`                                   | 成功更新後略過重新啟動閘道服務。會重新啟動的套件管理器更新會先驗證重新啟動的服務回報預期版本，命令才會成功。                                                                                                                                                |
| `--channel <stable\|extended-stable\|beta\|dev>` | 設定更新頻道，並在核心更新成功後保存。Extended-stable 僅支援套件安裝。                                                                                                                                                                                                                                            |
| `--tag <dist-tag\|version\|spec>`                | 僅覆寫本次更新的套件目標。它不能與生效的 `extended-stable` 頻道搭配使用，因為該頻道強制要求經過驗證的精確目標。對於其他套件安裝，`main` 會對應至 `github:openclaw/openclaw#main`；GitHub/git 來源規格會先封裝為暫存 tarball，再進行分階段的全域 npm 安裝。 |
| `--dry-run`                                      | 預覽規劃的動作（頻道/標籤/目標/重新啟動流程），但不寫入設定、不安裝、不同步外掛，也不重新啟動。                                                                                                                                                                                                                |
| `--json`                                         | 輸出機器可讀的 `UpdateRunResult` JSON。當受管理的外掛需要修復時，會包含 `postUpdate.plugins.warnings`、beta 頻道外掛的備援詳細資料，以及在更新後同步期間偵測到 npm 外掛成品偏移時的 `postUpdate.plugins.integrityDrifts`。                                                                 |
| `--timeout <seconds>`                            | 每個步驟的逾時時間。預設為 `1800`。                                                                                                                                                                                                                                                                                                            |
| `--yes`                                          | 略過確認提示（例如降級確認）。                                                                                                                                                                                                                                                                              |
| `--acknowledge-clawhub-risk`                     | 允許更新後的外掛同步在沒有互動式提示的情況下，略過社群 ClawHub 信任警告並繼續執行。若未使用此選項，當 OpenClaw 無法提示時，會略過有風險的社群版本並維持不變。官方 ClawHub 套件和內建外掛來源不會顯示此提示。                                                     |

沒有 `--verbose` 旗標。請使用 `--dry-run` 預覽規劃的動作、
使用 `--json` 取得機器可讀的結果，並使用 `openclaw update status --json`
僅查看頻道/可用性。閘道主控台詳細程度（`--verbose`）與
檔案日誌層級（`logging.level: "debug"`/`"trace"`）是彼此獨立的設定；請參閱
[閘道記錄](/zh-TW/gateway/logging)。

<Note>
在 Nix 模式（`OPENCLAW_NIX_MODE=1`）中，會停用會變更狀態的 `openclaw update` 執行。請改為更新此安裝的 Nix 來源或 flake 輸入；若使用 nix-openclaw，請採用代理程式優先的[快速入門](https://github.com/openclaw/nix-openclaw#quick-start)。`openclaw update status` 和 `openclaw update --dry-run` 仍為唯讀。
</Note>

<Warning>
降級需要確認，因為較舊的版本可能會破壞設定。
如果此安裝已將工作階段移轉至 SQLite，請先還原封存的舊版
逐字稿成品，再啟動較舊的檔案式版本。請參閱
[Doctor：工作階段移轉至 SQLite 後進行降級](/zh-TW/cli/doctor#downgrading-after-session-sqlite-migration)。
</Warning>

## `update status`

顯示使用中的更新頻道、git 標籤/分支/SHA（僅限原始碼簽出），
以及更新可用性。

```bash
openclaw update status
openclaw update status --json
openclaw update status --timeout 10
```

| 旗標                  | 預設值 | 說明                         |
| --------------------- | ------- | ----------------------------------- |
| `--json`              | `false` | 輸出機器可讀的狀態 JSON。 |
| `--timeout <seconds>` | `3`     | 檢查的逾時時間。                 |

對於 extended-stable 套件安裝，狀態檢查會執行與前景更新相同的公開選擇器
與精確套件驗證。當已安裝的版本較新時，它可能會回報
`ahead of extended-stable`。JSON 失敗結果會包含 `registry.reason`（`selector_missing`、`selector_query_failed`、
`exact_package_mismatch` 或 `unsupported_git_channel`）。

## `update repair`

當核心套件已變更，但後續修復工作未順利完成時，
重新執行更新的收尾作業。若 `openclaw update` 已安裝新的核心套件，但核心更新後的外掛同步、
受管理的 npm 外掛中繼資料、登錄檔重新整理或 doctor 修復未能
收斂，這是受支援的復原路徑。

```bash
openclaw update repair
openclaw update repair --channel beta
openclaw update repair --acknowledge-clawhub-risk
openclaw update repair --json
```

| 旗標                                             | 說明                                                                                                                                                                                                                                                         |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--channel <stable\|extended-stable\|beta\|dev>` | 在修復前保存核心更新頻道。對於 extended-stable，依循裸露/預設或 `latest` 意圖的合格官方 npm 外掛，會以已安裝核心的精確版本為目標。Git 簽出會拒絕 extended-stable 修復，且不變更設定。 |
| `--json`                                         | 輸出機器可讀的收尾作業 JSON。                                                                                                                                                                                                                           |
| `--timeout <seconds>`                            | 修復步驟的逾時時間。預設為 `1800`。                                                                                                                                                                                                                           |
| `--yes`                                          | 略過確認提示。                                                                                                                                                                                                                                          |
| `--acknowledge-clawhub-risk`                     | 行為與 `openclaw update` 相同。                                                                                                                                                                                                                              |
| `--no-restart`                                   | 為保持一致而接受此選項；修復絕不會重新啟動閘道。                                                                                                                                                                                                             |

`update repair` 會執行 `openclaw doctor --fix`、重新載入已修復的設定和
安裝記錄、同步使用中更新頻道所追蹤的外掛、更新
受管理的 npm 外掛安裝、修復缺少的已設定外掛承載內容、
重新整理外掛登錄檔，並寫入已收斂的安裝記錄中繼資料。
它不會安裝新的核心套件，也不會重新啟動閘道。

## `update wizard`

互動式流程，用於選擇更新頻道，並確認之後是否重新啟動
閘道（預設會重新啟動）。若在沒有 git
簽出的情況下選擇 `dev`，系統會詢問是否要建立一個。

| 旗標                  | 預設值 | 說明                   |
| --------------------- | ------- | ----------------------------- |
| `--timeout <seconds>` | `1800`  | 每個更新步驟的逾時時間。 |

## 執行內容

明確切換頻道（`--channel ...`）也會讓安裝方式
保持一致：

- `dev` -> 確保存在 git 簽出（預設為 `~/openclaw`，或在
  設定 `OPENCLAW_HOME` 時使用 `$OPENCLAW_HOME/openclaw`；可透過
  `OPENCLAW_GIT_DIR` 覆寫），更新該簽出，並從該
  簽出安裝全域命令列介面。
- `stable` -> 使用 `latest` 從 npm 安裝。
- `extended-stable` -> 解析公開 npm `extended-stable` 選擇器、
  驗證所選的精確套件，並安裝該精確版本。它
  不會備援至其他選擇器，且 Git 簽出會拒絕此操作。
- `beta` -> 優先使用 npm dist-tag `beta`；當 beta
  不存在或比目前的穩定版本舊時，則備援至 `latest`。

### 重新啟動交接

閘道核心自動更新程式（透過設定啟用時）會在即時閘道
請求處理常式之外啟動命令列介面更新路徑。控制平面的
`update.run` 套件管理器更新和受監督的 git 簽出更新會使用
相同的受管理服務交接，而不會在即時閘道處理程序內取代套件樹或
重新建置 `dist/`：閘道會啟動一個
中斷連結的輔助程式並結束，而該輔助程式會從閘道處理程序樹之外執行 `openclaw update --yes --json`。
如果無法進行交接，`update.run` 會傳回結構化回應，其中包含可供手動執行的
安全 shell 命令。

儲存的延伸穩定版選擇在啟用 `update.checkOnStart` 時，會收到唯讀的啟動與每 24 小時一次的更新
提示。這些檢查絕不會套用更新、
啟動交接、重新啟動閘道、使用穩定版的延遲／抖動，或使用測試版的
輪詢頻率。明確的前景更新、使用已儲存
`update.channel: "extended-stable"` 的無參數前景更新、隨選狀態，以及其受管理的
閘道交接仍受支援。

安裝本機受管理的閘道服務且啟用重新啟動時，
套件管理器與 Git 簽出更新會先停止執行中的服務，再
取代套件樹或變更簽出／建置輸出。接著，更新程式會
重新整理服務中繼資料、重新啟動服務，並驗證重新啟動的
閘道，之後才回報 `Gateway: restarted and verified.`。
套件管理器更新還會驗證重新啟動的閘道回報
預期的套件版本；Git 簽出更新則會在重新建置後驗證閘道健康狀態與
服務就緒狀態。

套件管理器更新通常會繼續使用受管理服務中記錄的
Node 二進位檔。如果該 Node 無法執行目標版本，但目前
命令列介面的 Node 可以，且已證明該服務屬於正在更新的套件，
則啟用重新啟動的更新會使用目前的 Node 完成最終處理，並將
服務中繼資料改寫為該執行階段。`--no-restart` 無法修復服務
中繼資料，因此遇到相同的執行階段不相符時，會在變更套件前停止。

在 macOS 上，更新後檢查也會驗證 LaunchAgent 已針對
使用中的設定檔載入／執行，且設定的回送連接埠狀態正常。
如果 plist 已安裝，但 launchd 未監督它，OpenClaw
會自動重新引導 LaunchAgent，並重新執行健康狀態／版本／
頻道就緒檢查（全新引導會直接載入 `RunAtLoad` 工作，
因此復原程序不會立即 `kickstart -k` 新產生的閘道）。如果
閘道仍未恢復正常，命令會以非零狀態結束，並
印出重新啟動記錄路徑，以及重新啟動、重新安裝和套件回復
指示。

如果無法執行重新啟動，命令會印出 `Gateway: restart skipped (...)` 或
`Gateway: restart failed: ...`，並附上手動執行 `openclaw gateway restart` 的提示。
使用 `--no-restart` 時，套件取代或 Git 重新建置仍會執行，但
受管理服務不會停止或重新啟動，因此執行中的閘道會繼續使用舊
程式碼，直到你手動重新啟動它。

### 控制平面回應格式

當 `update.run` 透過閘道控制平面，在套件管理器
安裝或受監督的 Git 簽出上執行時，處理常式會分別回報交接啟動
與閘道結束後繼續執行的命令列介面更新：

- `ok: true`、`result.status: "skipped"`、
  `result.reason: "managed-service-handoff-started"` 和
  `handoff.status: "started"`：閘道已建立受管理服務交接，
  並排定自身重新啟動，讓分離的輔助程式可以在
  即時服務程序之外執行 `openclaw update --yes --json`。
- `ok: false`、`result.reason: "managed-service-handoff-unavailable"` 和
  `handoff.status: "unavailable"`：OpenClaw 找不到可用於安全交接的
  監督服務邊界與持久服務識別資訊（例如，
  systemd 交接需要 `OPENCLAW_SYSTEMD_UNIT` 單元識別資訊，
  而不只是環境中的 systemd 程序標記）。回應中包含
  `handoff.command`，也就是要從閘道外部執行的 Shell 命令。
- `ok: false`、`result.reason: "managed-service-handoff-failed"`：閘道
  已嘗試建立交接，但無法產生分離的輔助程式。

`sentinel` 承載資料會在閘道結束前寫入，而命令列介面
交接會在受管理服務重新啟動的健康狀態檢查完成後，
更新同一個重新啟動哨兵。在交接期間，哨兵可能帶有
`stats.reason: "restart-health-pending"`，但沒有成功延續動作；
重新啟動的閘道會輪詢它，且只有在命令列介面
驗證服務健康狀態，並將哨兵改寫為最終 `ok` 結果後，
才會觸發延續動作。
當該哨兵處於待處理或失敗狀態時，`openclaw status` 和 `openclaw status --all`
會顯示一列 `Update restart`，而 `update.status` 會重新整理並
傳回最新的哨兵。

## Git 簽出流程

### 頻道選擇

- `stable`：簽出最新的非測試版標籤，然後建置並執行 doctor。
- `beta`：優先使用最新的 `-beta` 標籤；當測試版不存在或較舊時，
  回退至最新的穩定版標籤。
- `dev`：簽出 `main`，然後擷取並重定基底。
- `extended-stable`：Git 簽出不支援；不會變更簽出內容。

### 更新步驟

<Steps>
  <Step title="驗證工作樹乾淨">
    要求沒有未提交的變更。
  </Step>
  <Step title="切換頻道">
    切換至選取的頻道（標籤或分支）。
  </Step>
  <Step title="擷取上游">
    僅限開發版。
  </Step>
  <Step title="預先檢查建置（僅限開發版）">
    在暫存工作樹中執行 TypeScript 建置。如果頂端提交失敗，最多往回檢查 10 個提交，以尋找最新可建置的提交。設定 `OPENCLAW_UPDATE_PREFLIGHT_LINT=1`，可在此預先檢查期間一併執行 lint；lint 會以受限的循序模式執行，因為使用者的更新主機通常比 CI 執行器小。
  </Step>
  <Step title="重定基底">
    重定基底至選取的提交（僅限開發版）。
  </Step>
  <Step title="安裝相依套件">
    使用儲存庫的套件管理器。對於 pnpm 簽出，更新程式會視需要引導 `pnpm`（先透過 `corepack`，再使用暫時的 `npm install pnpm@11` 回退方案），而不是在 pnpm 工作區內執行 `npm run build`。如果 pnpm 引導仍然失敗，更新程式會提前停止並顯示套件管理器專屬錯誤，而不會嘗試在簽出中執行 `npm run build`。
  </Step>
  <Step title="建置控制介面">
    建置閘道與控制介面。
  </Step>
  <Step title="執行 doctor">
    `openclaw doctor` 會作為最終的安全更新檢查執行。
  </Step>
  <Step title="同步外掛">
    將外掛同步至使用中的頻道。開發版使用隨附外掛；穩定版和測試版使用 npm。更新已追蹤的外掛安裝。
  </Step>
</Steps>

### 外掛同步詳細資訊

在測試版頻道上，遵循預設／最新版本線的已追蹤 npm 與 ClawHub
外掛安裝，會先嘗試外掛的 `@beta` 版本。如果外掛沒有
測試版，OpenClaw 會回退至記錄的預設／最新規格，並
回報警告。對於 npm 外掛，當測試版
套件存在但無法通過安裝驗證時，OpenClaw 也會回退。這些回退警告不會
讓核心更新失敗。確切版本和明確標籤絕不會被改寫。

<Warning>
如果確切釘選的 npm 外掛更新解析出完整性與儲存的安裝記錄不同的成品，`openclaw update` 會中止該外掛成品更新，而不會安裝它。只有在確認你信任新的成品後，才明確重新安裝或更新該外掛。
</Warning>

<Note>
更新後的外掛同步失敗若僅限於受管理的外掛，且同步路徑可以繞過該失敗（例如非必要外掛的 npm 登錄檔無法連線），則會在核心更新成功後回報為警告。JSON 結果會保留頂層更新 `status: "ok"`，並回報 `postUpdate.plugins.status: "warning"`，其中包含 `openclaw update repair` 和 `openclaw plugins inspect <id> --runtime --json` 指引。未預期的更新程式或同步例外仍會使更新結果失敗。修正外掛安裝或更新錯誤，然後重新執行 `openclaw update repair`。當失敗的更新使受管理的外掛無法使用時，OpenClaw 會停用其執行階段項目並重設使用中的插槽，但不會變更操作員編寫的 `plugins.allow` 或 `plugins.deny` 原則。

在逐一同步外掛的步驟之後，`openclaw update` 會在閘道重新啟動前，強制執行一次**核心更新後收斂**階段：它會修復缺少的已設定外掛承載資料、驗證磁碟上每筆_使用中_的已追蹤安裝記錄，並以靜態方式驗證其 `package.json` 可解析（以及任何明確宣告的 `main` 是否存在）。此階段的失敗以及無效的設定快照會傳回 `postUpdate.plugins.status: "error"`，並將頂層更新 `status` 翻轉為 `"error"`，因此 `openclaw update` 會以非零狀態結束，且閘道_不會_以未經驗證的外掛集合重新啟動。錯誤包含結構化的 `postUpdate.plugins.warnings[].guidance` 行，指向 `openclaw update repair` 和 `openclaw plugins inspect <id> --runtime --json`。此處會略過已停用的外掛項目，以及並非受信任來源連結之官方同步目標的記錄（比照缺少承載資料檢查所使用的 `skipDisabledPlugins` 原則），因此過時的已停用外掛記錄不會阻擋其他方面均有效的更新。

更新後的閘道啟動時，外掛載入僅執行驗證：啟動程序不會執行套件管理器或變更相依套件樹。套件管理器的 `update.run` 重新啟動會交由命令列介面的受管理服務路徑處理，因此套件交換會在舊閘道程序之外進行，並由服務健康狀態檢查決定是否能將更新回報為完成。
</Note>

延伸穩定版核心更新成功後，核心更新後的外掛完整性與
收斂會以確切的已安裝核心版本為目標，處理符合資格的官方 npm 外掛。
對於預設／`latest` 意圖，OpenClaw 不會查詢外掛
`@extended-stable`，也不會回退至 npm `latest`；而是從已安裝的核心
推導套件版本。明確的版本釘選、明確的非 `latest` 標籤、
第三方套件和非 npm 來源會保留其現有意圖。

對於套件管理器安裝，`openclaw update` 會在叫用套件管理器前解析目標套件
版本。npm 全域安裝使用分階段安裝：OpenClaw 將新套件安裝至暫時的 npm 前綴，
讓候選套件在 `preinstall` 期間驗證主機 Node 版本，
並在該處驗證封裝的 `dist` 清單。封裝的完成防護會
留在該清單之外，直到 `preinstall` 成功，因此略過生命週期指令碼的
套件管理器也會在啟用前停止。在 npm 12 及更新版本上，
更新程式只核准候選 OpenClaw 的生命週期；遞移
相依套件的指令碼仍會被封鎖。接著，OpenClaw 會將乾淨的套件樹
交換至實際的全域前綴。如果驗證失敗，更新後 doctor、外掛
同步和重新啟動工作都不會從可疑的套件樹執行。即使
已安裝版本已符合目標，命令仍會重新整理
全域套件安裝，然後執行外掛同步、核心命令補全
重新整理和重新啟動工作。這會讓封裝的附帶元件和頻道所擁有的
外掛記錄與已安裝的 OpenClaw 建置保持一致，同時將完整的
外掛命令補全重新建置留給明確執行的
`openclaw completion --write-state`。

## 相關內容

- `openclaw doctor`（在 Git 簽出上會先提議執行更新）
- [開發頻道](/zh-TW/install/development-channels)
- [更新](/zh-TW/install/updating)
- [命令列介面參考](/zh-TW/cli)
