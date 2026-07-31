---
read_when:
    - 你遇到連線／驗證問題，並希望取得引導式修正協助
    - 你已完成更新，並想進行基本檢查
summary: '`openclaw doctor` 的命令列介面參考（健康檢查 + 引導式修復）'
title: 診斷工具
x-i18n:
    generated_at: "2026-07-26T07:13:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e2b0aa9b51d7bccd4357d3ec747be514a0245b44a90e6e6c7ea789ab68420465
    source_path: cli/doctor.md
    workflow: 16
---

# `openclaw doctor`

針對閘道、頻道、外掛、Skills、模型路由、本機狀態與設定遷移進行健康檢查及快速修復。每當系統行為不如預期，且你希望使用單一命令說明問題所在時，請使用此功能。

當閘道狀態回報 SecretRef 擁有者處於降級狀態時，doctor 會顯示 **Secret 執行階段降級**警告，列出每個冷啟動或過期的擁有者、受影響的設定路徑、經遮蔽的原因，以及 `openclaw secrets reload` 重試命令。

當頻道輸入事件遭移入無法投遞佇列時，doctor 會列出每個受影響的頻道帳號，並引導至 [`openclaw channels dead-letters list`](/zh-TW/cli/channels#inbound-dead-letters) 進行檢查與復原。

相關內容：

- 疑難排解：[疑難排解](/zh-TW/gateway/troubleshooting)
- 安全性稽核：[安全性](/zh-TW/gateway/security)

## 運作模式

Doctor 有五種運作模式：

| 運作模式                  | 命令                                      | 行為                                                                            |
| ------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------- |
| 檢查                      | `openclaw doctor`                        | 執行以人為導向的檢查並提供引導式提示。                                          |
| 修復                      | `openclaw doctor --fix`                        | 套用支援的修復；除非可安全地進行非互動式修復，否則會使用提示。                  |
| Lint                      | `openclaw doctor --lint`                        | 為 CI、預檢與審查閘門提供唯讀的結構化發現。                                     |
| 共用 SQLite 維護          | `openclaw doctor --state-sqlite compact`                        | 明確地為標準共用狀態資料庫建立檢查點、壓縮並進行驗證。                          |
| 工作階段 SQLite 遷移      | `openclaw doctor --session-sqlite <mode>`                        | 檢查、匯入、驗證、壓縮、復原或還原工作階段狀態。                                |

當自動化流程需要穩定的結果時，建議使用 `--lint`。當人工操作員希望 doctor 編輯設定或狀態時，建議使用 `--fix`。

## 範例

```bash
openclaw doctor
openclaw doctor --lint
openclaw doctor --lint --json
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --all
openclaw doctor --lint --allow-exec
openclaw doctor --deep
openclaw doctor --fix
openclaw doctor --fix --non-interactive
openclaw doctor --generate-gateway-token
openclaw doctor --post-upgrade
openclaw doctor --post-upgrade --json
openclaw doctor --state-sqlite compact
openclaw doctor --state-sqlite compact --json
openclaw doctor --session-sqlite inspect --session-sqlite-all-agents
openclaw doctor --session-sqlite dry-run --session-sqlite-agent main --json
openclaw doctor --session-sqlite import --session-sqlite-all-agents
openclaw doctor --session-sqlite validate --session-sqlite-all-agents --json
openclaw doctor --session-sqlite compact --session-sqlite-all-agents
openclaw doctor --session-sqlite recover --github-issue
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

針對特定頻道的權限，請使用頻道探測，而非 `doctor`：

```bash
openclaw channels capabilities --channel discord --target channel:<channel-id>
openclaw channels status --probe
```

`channels capabilities` 會回報機器人對特定頻道目標的實際權限。`channels status --probe` 會稽核所有已設定的頻道與語音自動加入目標。

## 選項

| 選項                            | 效果                                                                                                                                                                                    |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--no-workspace-suggestions`              | 停用工作區記憶體／搜尋建議。                                                                                                                                                            |
| `--yes`              | 接受預設值而不顯示提示。                                                                                                                                                                |
| `--repair` / `--fix` | 套用建議的非服務修復而不顯示提示（`--fix` 是別名）。閘道服務的安裝／重寫仍需互動式確認或明確的 `gateway` 命令。 |
| `--force`              | 套用積極修復，包括覆寫自訂服務設定。                                                                                                                                                    |
| `--non-interactive`              | 執行時不顯示提示；僅執行安全遷移與非服務修復。                                                                                                                                          |
| `--generate-gateway-token`              | 產生並設定閘道權杖。                                                                                                                                                                    |
| `--allow-exec`              | 允許 doctor 在驗證密鑰時執行已設定的 `exec` SecretRef。                                                                                                                     |
| `--deep`              | 掃描系統服務以尋找額外的閘道安裝；回報最近的閘道監督程式重新啟動交接。                                                                                                                  |
| `--lint`              | 以唯讀模式執行現代化健康檢查並輸出診斷發現。                                                                                                                                            |
| `--post-upgrade`              | 執行升級後的外掛相容性探測；發現會輸出至 stdout；若存在任何錯誤層級的發現，結束代碼為 1。                                                                                                |
| `--state-sqlite <mode>`              | 執行明確的共用狀態 SQLite 維護。唯一模式為 `compact`。                                                                                                                         |
| `--session-sqlite <mode>`              | 執行指定的工作階段 SQLite 遷移模式：`inspect`、`dry-run`、`import`、`validate`、`compact`、`recover` 或 `restore`。 |
| `--session-sqlite-store <path>`              | 搭配 `--session-sqlite`：選取一個舊版 `sessions.json` 儲存區路徑。                                                                                                                   |
| `--session-sqlite-agent <id>`              | 搭配 `--session-sqlite`：選取一個已設定的代理程式。                                                                                                                                     |
| `--session-sqlite-all-agents`              | 搭配 `--session-sqlite`：選取已設定及已探索到的代理程式儲存區。                                                                                                                        |
| `--github-issue`              | 搭配 `--session-sqlite recover`：準備經過清理的 openclaw/openclaw 問題回報；doctor 會在 `gh` 之後，經 `--yes` 或互動式確認後建立問題。                                   |
| `--json`              | 搭配 `--lint`：輸出 JSON 發現。搭配 `--post-upgrade`：`{ probesRun, findings }`。搭配 `--state-sqlite` 或 `--session-sqlite`：以 JSON 輸出維護報告。                              |
| `--severity-min <level>`              | 搭配 `--lint`：捨棄低於 `info`、`warning` 或 `error` 的發現。                                                                                 |
| `--all`              | 搭配 `--lint`：執行所有已註冊的檢查，包括預設集合中排除的選用檢查。                                                                                                            |
| `--skip <id>`              | 搭配 `--lint`：略過一個檢查 ID。可重複指定。                                                                                                                                  |
| `--only <id>`              | 搭配 `--lint`：僅執行指定的檢查 ID。可重複指定。                                                                                                                              |

`--severity-min`、`--all`、`--only` 與 `--skip` 僅能與 `--lint` 一起使用；`--json` 可與 `--lint`、`--post-upgrade`、`--state-sqlite` 及 `--session-sqlite` 一起使用。

## Lint 模式

`openclaw doctor --lint` 為唯讀模式：不顯示提示、不進行修復，也不重寫設定／狀態。

```bash
openclaw doctor --lint
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --json
openclaw doctor --lint --all
openclaw doctor --lint --allow-exec
openclaw doctor --lint --only core/doctor/gateway-config --json
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

供人閱讀的輸出十分精簡：

```text
doctor --lint：已執行 6 項檢查，發現 1 個問題
  [warning] core/doctor/gateway-config gateway.mode - gateway.mode 尚未設定；閘道將無法啟動。
    修復方式：執行 `openclaw configure` 並設定閘道模式（local/remote），或執行 `openclaw config set gateway.mode local`。
```

JSON 輸出是供指令碼使用的介面：

```json
{
  "ok": false,
  "checksRun": 5,
  "checksSkipped": 0,
  "findings": [
    {
      "checkId": "core/doctor/gateway-config",
      "severity": "warning",
      "message": "gateway.mode 尚未設定；閘道將無法啟動。",
      "path": "gateway.mode",
      "fixHint": "執行 `openclaw configure` 並設定閘道模式（local/remote），或執行 `openclaw config set gateway.mode local`。"
    }
  ]
}
```

結束代碼：

| 代碼 | 意義                                                        |
| ---- | ----------------------------------------------------------- |
| `0` | 沒有任何發現達到或超過所選的嚴重性門檻。                    |
| `1` | 至少有一項發現達到所選門檻。                                |
| `2` | 在產生 lint 發現之前發生命令／執行階段失敗。                 |

`--severity-min` 同時控制要顯示哪些發現以及結束門檻：即使存在嚴重性較低的 `info`/`warning` 發現，`openclaw doctor --lint --severity-min error` 仍可能不顯示任何內容並以 `0` 結束。

`--all` 控制在套用嚴重性篩選前要選取哪些檢查。預設 lint 執行會排除深入、歷史性，或較可能顯示可修復舊版殘留項目的檢查；若要使用完整清單，請使用 `--all`。`--only <id>` 是最精確的選取器，可依 ID 執行任何已註冊的檢查。

`core/doctor/local-audio-acceleration` 會回報自動選取的本機 STT 命令、分開列出的可用／要求／觀察到的後端證據，以及備援順序，而不載入語音模型。它會產生資訊層級的發現，因此請加入 `--severity-min info` 以顯示該發現。

## 結構化健康檢查

現代化的 doctor 檢查使用小型的拆分合約：

```ts
detect(ctx, scope?) -> HealthFinding[]
repair?(ctx, findings) -> HealthRepairResult
```

`detect()` 為 `doctor --lint` 提供支援。`repair()` 為選用項目，且僅會在 `doctor --fix` / `doctor --repair` 下執行。尚未遷移至此形式的檢查，仍會使用舊版 doctor 貢獻流程。

修復情境可攜帶 `dryRun`/`diff` 請求；修復結果可傳回結構化的 `diffs`（設定／檔案編輯）和 `effects`（服務、程序、套件、狀態或其他副作用），讓轉換後的檢查能朝 `doctor --fix --dry-run` 發展，而不必將變更規劃移入 `detect()`。

`repair()` 會回報 `status: "repaired" | "skipped" | "failed"`（省略狀態表示 `repaired`）。當修復傳回 `skipped` 或 `failed` 時，Doctor 會回報原因，並略過該檢查的驗證。修復成功後，Doctor 會針對已修復的發現重新執行 `detect()`；若發現仍然存在，Doctor 會回報修復警告，而不會將變更視為已完成。

一項發現包含：

| 欄位             | 用途                                                |
| ----------------- | ------------------------------------------------------ |
| `checkId`         | 用於略過／僅限篩選器及 CI 允許清單的穩定 ID。     |
| `severity`        | `info`、`warning` 或 `error`。                         |
| `message`         | 供人閱讀的問題陳述。                      |
| `path`            | 可取得時的設定、檔案或邏輯路徑。          |
| `line` / `column` | 可取得時的來源位置。                        |
| `ocPath`          | 檢查可指向特定位置時的精確 `oc://` 位址。 |
| `fixHint`         | 建議的操作人員動作或修復摘要。           |

現代化的核心 Doctor 檢查會繼續附加於擁有其人工 `doctor` / `doctor --fix` 行為的有序 Doctor 貢獻項目。共用的結構化健康狀態登錄是擴充點：套裝及由外掛支援的檢查，在其所屬套件於作用中的命令路徑註冊後，會於核心 Doctor 檢查之後執行。`openclaw/plugin-sdk/health` 會向外掛作者公開相同的合約。

## 檢查選取

```bash
openclaw doctor --lint --only core/doctor/gateway-config --json
openclaw doctor --lint --skip core/doctor/skills-readiness
openclaw doctor --lint --all --skip core/doctor/session-locks
```

`--only` 和 `--skip` 接受完整的檢查 ID，且可重複指定。若某個 `--only` ID 尚未註冊，該 ID 不會執行任何檢查；請使用輸出中的 `checksRun`/`checksSkipped`，確認聚焦式閘門選取了你預期的檢查。

## 升級後模式

`openclaw doctor --post-upgrade` 會執行外掛相容性探測，以便串接於建置或升級之後。發現會輸出至標準輸出；若任何發現具有 `level: "error"`，結束代碼為 1。加入 `--json` 可取得機器可讀的封裝（`{ probesRun, findings }`），適用於 CI、社群 `fork-upgrade` skill，以及其他升級後冒煙測試工具。若已安裝的外掛索引遺失或格式錯誤，JSON 模式仍會輸出封裝，其中包含一項 `plugin.index_unavailable` 錯誤發現。

容器映像啟動是一般「更新後執行 Doctor」流程的例外。當 `openclaw gateway run` 以新版 OpenClaw 啟動時，會先執行安全的狀態及外掛修復，再回報已就緒。若無法安全完成修復，啟動程序會結束，並指示你在正常重新啟動容器前，針對相同的掛載狀態／設定，以 `openclaw doctor --fix` 執行相同映像一次。

## 舊版狀態遷移

`openclaw doctor --fix` 是持久性檔案至 SQLite 遷移的唯一擁有者。它會驗證並認領每個可辨識的來源、寫入並驗證標準資料列、記錄遷移收據，然後移除已淘汰的來源。執行階段程式碼不會執行延遲匯入或備援讀取。

這包括 `<state-dir>/mcp-oauth/*.json` 下已淘汰的 MCP OAuth 檔案。修復前請停止閘道。Doctor 會將有效的認證資訊匯入 `<state-dir>/state/openclaw.sqlite`；當兩種儲存區都存在時，保留既有的標準 SQLite 工作階段；捨棄過時的持久化 OAuth `state` 值；並使用其收據，防止重新建立的過時檔案使已登出的認證資訊復原。已淘汰的 `.lock` 側邊檔案會採取失敗關閉：若 Doctor 回報過時的擁有者，請確認沒有較舊的 OpenClaw 程序仍在執行、移除該側邊檔案，然後重新執行 Doctor。

## 共用狀態 SQLite 壓縮

如需結構描述版本控制、完整性檢查及降級復原的相關資訊，請參閱[資料庫結構描述](/zh-TW/reference/database-schemas)。

`openclaw doctor --state-sqlite compact` 是針對位於 `<state-dir>/state/openclaw.sqlite` 的標準共用狀態資料庫所進行的明確離線維護。它不接受任意資料庫路徑、絕不會由正常的閘道作業呼叫，也不屬於 `openclaw doctor --fix`。此命令會取得與閘道啟動相同的狀態擁有權鎖定，並在驗證、建立檢查點、`VACUUM` 及最終完整性檢查期間持續持有該鎖定。當閘道或另一個 SQLite 維護命令擁有該鎖定時，它會拒絕執行。當 `OPENCLAW_ALLOW_MULTI_GATEWAY=1` 略過每個設定的閘道單一執行個體時，狀態鎖定仍會保持作用中，因此操作人員的 shell 不必繼承閘道服務的環境，也能讓維護程序偵測到該鎖定。

請先停止閘道並建立已驗證的備份：

```bash
openclaw gateway stop
openclaw backup create --verify
openclaw doctor --state-sqlite compact --json
openclaw gateway start
```

此命令：

1. 要求標準共用狀態路徑上存在一般檔案。遺失的資料庫會回報為 `skipped`，並成功結束。
2. 在建立檢查點或變更檔案前，驗證目前支援的結構描述版本及 `schema_meta.role = "global"`。
3. 要求 `wal_checkpoint(TRUNCATE)` 處於非忙碌狀態。若檢查點忙碌，請停止所有其餘的 OpenClaw 程序後重試。
4. 將 `auto_vacuum` 設為 `INCREMENTAL`、執行完整的 `VACUUM`，然後再次建立檢查點。
5. 執行 `quick_check`、`integrity_check` 及 `foreign_key_check`，然後將僅限擁有者的權限重新套用至資料庫及 SQLite 側邊檔案。

JSON 輸出會回報壓縮前後的資料庫與 WAL 大小、可用清單頁面、頁面大小及 `auto_vacuum` 值，以及回收的位元組數和 `quick_check` 與 `integrity_check` 的結果。`foreign_key_check` 會強制採取失敗關閉，且沒有個別的成功欄位。SQLite 會將 `auto_vacuum` 回報為：無時為 `0`、完整時為 `1`、增量時為 `2`。

當結構描述過舊、比執行中的 OpenClaw 組建更新，或屬於代理程式資料庫時，壓縮會在不進行變更的情況下失敗。若共用狀態結構描述較舊，請先執行 `openclaw doctor --fix`。若結構描述較新，請還原相容的備份或升級 OpenClaw。

## 工作階段 SQLite 遷移

OpenClaw 會在閘道啟動期間及執行 `openclaw doctor --fix` 時，自動將舊版工作階段資料列與逐字記錄歷程匯入各代理程式的 SQLite 資料庫。`openclaw doctor --session-sqlite <mode>` 是該遷移的針對性檢查與驗證工具。目前執行階段工作階段資料列位於 `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`。舊版 `sessions.json` 檔案是遷移來源。作用中的逐字記錄 JSONL 檔案會在成功匯入後完成匯入，並封存至作用中工作階段目錄之外；封存層級的 JSONL 檔案仍是支援成品，不是執行階段備援。

模式：

| 模式       | 行為                                                                                                               |
| ---------- | ---------------------------------------------------------------------------------------------------------------------- |
| `inspect`  | 讀取舊版及 SQLite 計數，以及未參照的 JSONL 檔案，而不進行匯入。                                       |
| `dry-run`  | 剖析舊版項目及逐字記錄 JSONL 檔案、計算可匯入資料列，並回報問題，而不寫入 SQLite 資料列。 |
| `import`   | 將舊版項目及逐字記錄事件匯入所選目標的 SQLite。                                      |
| `validate` | 比較所選舊版來源與 SQLite 資料列及逐字記錄事件計數。                                   |
| `compact`  | 對所選代理程式 SQLite 資料庫建立檢查點並執行 VACUUM，以在大量刪除或封存清理後回收可用頁面。    |
| `recover`  | 還原最新失敗的遷移執行、驗證其目標，並準備經過清理的 GitHub 議題報告。            |
| `restore`  | 從已記錄的遷移資訊清單還原已封存的逐字記錄成品，而不刪除 SQLite 資料。                  |

選取器：

- 預設：已設定的預設代理程式儲存區（當該舊版儲存區檔案存在時）。
- `--session-sqlite-agent <id>`：一個已設定的代理程式。
- `--session-sqlite-all-agents`：已設定的代理程式儲存區，加上已探索到的代理程式儲存區。
- `--session-sqlite-store <path>`：一個明確的舊版 `sessions.json` 路徑。

手動檢查順序：

```bash
openclaw doctor --session-sqlite inspect --session-sqlite-all-agents
openclaw doctor --session-sqlite dry-run --session-sqlite-all-agents --json
openclaw doctor --session-sqlite import --session-sqlite-all-agents
openclaw doctor --session-sqlite validate --session-sqlite-all-agents --json
openclaw doctor --session-sqlite compact --session-sqlite-all-agents
openclaw doctor --session-sqlite recover --github-issue
```

在含有重要歷程的安裝環境中執行 `import` 前，請先備份 OpenClaw 狀態目錄。當所選舊版項目未出現在 SQLite 中、工作階段 ID 不同，或逐字記錄事件計數不同時，`validate` 會以非零狀態結束。使用 `--session-sqlite-store <path>` 時，請檢查報告是否包含預期的目標數量；不存在的明確儲存區路徑不會選取任何目標。

SQLite 刪除作業會先回收資料庫內的頁面；不一定會立即縮小資料庫檔案。刪除或封存大型逐字記錄後，請執行 `openclaw doctor --session-sqlite compact --session-sqlite-all-agents`，以對 WAL 檔案建立檢查點、執行 `VACUUM`，並回報作業前後的資料庫及 WAL 大小。壓縮要求存在採用目前代理程式結構描述的一般檔案、所選代理程式的持久擁有者中繼資料，且 Doctor 程序中沒有開啟的控制代碼。具破壞性的 `import`、`compact`、`recover` 及 `restore` 模式會在整個作業期間，持有與閘道啟動相同的狀態擁有權鎖定；`inspect`、`dry-run` 及 `validate` 則維持唯讀，不會取得該鎖定。請先停止閘道。具破壞性的模式會直接失敗，而不會與即時寫入或其他維護命令產生競爭。具破壞性的 `--session-sqlite-store` 目標必須位於作用中的狀態目錄內；維護其他安裝環境前，請將 `OPENCLAW_STATE_DIR` 設為該儲存區所屬的狀態目錄。既有的硬連結目標會遭拒絕，因為另一條路徑可能會在鎖定的狀態目錄之外，共用相同的資料庫 inode。相同的擁有權檢查也涵蓋 SQLite WAL、共用記憶體及回復日誌側邊檔案。

每次匯入都會先在 `~/.openclaw/session-sqlite-migration-runs/` 下寫入資訊清單，再將逐字記錄成品移入封存區。若成品移動後，啟動程序回報工作階段 SQLite 遷移失敗，請執行復原：

```bash
openclaw doctor --session-sqlite recover --github-issue
```

復原會選取最新的失敗遷移資訊清單、僅還原該資訊清單封存的成品、驗證受影響的目標、重新整理經過清理的 `.failure.md` 與 `.failure.json` 報告，並準備一份不含對話記錄內容、原始環境、祕密及無界限設定的 GitHub 議題內文。若不存在失敗的遷移資訊清單，但所選代理程式的 SQLite 資料庫已損毀、並非資料庫，或僅有日誌附屬檔案而沒有主要資料庫，復原程序會將完整檔案集複製到暫存檢查目錄。SQLite 可在該可丟棄副本中回復有效的熱日誌，之後再執行 `quick_check`、`integrity_check` 與 `foreign_key_check`，同時保持原始鑑識檔案不受更動。若完整性檢查失敗或存在孤立的附屬檔案，系統會以一個 `.corrupt-<timestamp>` 後綴重新命名整個已發現的檔案集，以保留 DB、WAL、SHM 與回復日誌檔案。若重新命名失敗且錯誤被捕捉，系統會先將已移動的檔案移回，再回報失敗，因此不會在未告知的情況下拆散可復原的檔案集。請在復原前停止閘道；複製或重新命名仍在持續變動的 SQLite 檔案集並不安全，而且在不同作業系統上的行為也不相同。使用 `--github-issue --yes` 時，doctor 會使用 GitHub 命令列介面在 `openclaw/openclaw` 中建立議題；若未確認，則會寫入本機支援報告並印出預先填妥的議題 URL。

`restore` 仍是較底層的復原操作。它使用資訊清單中的 `sourcePath -> archivePath` 記錄，僅在原始路徑不存在時將封存成品移回；若兩個路徑都存在，則會回報衝突，並將 SQLite 資料庫留在原處。

### 工作階段 SQLite 遷移後降級

在啟動較舊、以檔案為後端的 OpenClaw 版本之前，請還原已封存的舊版對話記錄成品：

```bash
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

較舊的版本會讀取 `sessions.json` 項目，以及這些項目中記錄的 `sessionFile` 路徑。完成 SQLite 遷移後，成功匯入會將作用中的 JSONL 對話記錄移至 `session-sqlite-import-archive/`，因此在還原程序將資訊清單所記錄的這些成品移回原始路徑之前，較舊的執行階段無法看到該歷程記錄。

還原不會刪除 SQLite 資料。在切換至 SQLite 後建立的工作階段僅存在於 SQLite 中，不會出現在較舊的執行階段。若之後再次升級，請執行上述正常的遷移驗證順序，讓 OpenClaw 能在匯入前比較已還原的舊版成品與 SQLite 資料列。

## 注意事項

- 在 Nix 模式（`OPENCLAW_NIX_MODE=1`）下，唯讀的 doctor 檢查仍可運作，但 `doctor --fix`、`doctor --repair`、`doctor --yes` 和 `doctor --generate-gateway-token` 會停用，因為 `openclaw.json` 不可變。請改為編輯此安裝的 Nix 來源；若使用 nix-openclaw，請參閱以代理程式為優先的[快速入門](https://github.com/openclaw/nix-openclaw#quick-start)。
- 互動式提示（鑰匙圈／OAuth 修復等）只會在 stdin 是 TTY 且**未**設定 `--non-interactive` 時執行。無頭執行（排程、Telegram、無終端機）會略過提示。
- 非互動式 `doctor` 執行會略過預先載入外掛，讓無頭健康狀態檢查保持快速。互動式工作階段仍會載入舊版健康狀態／修復流程所需的外掛介面。
- `--lint` 比 `--non-interactive` 更嚴格：一律唯讀、絕不顯示提示，也絕不套用安全遷移。若要讓 doctor 進行變更，請使用 `doctor --fix` 或 `doctor --repair`。
- 根據預設，Doctor 在檢查密鑰時不會執行 `exec` SecretRef。只有當你刻意要讓 doctor 執行這些已設定的密鑰解析器時，才使用 `--allow-exec`（可搭配或不搭配 `--lint`）。
- 任何設定寫入（包括 `--fix` 修復）都會將備份輪替至 `~/.openclaw/openclaw.json.bak`（另有編號的 `.bak.1`..`.bak.4` 環狀備份）。`--fix` 也會移除結構描述驗證所回報的未知設定鍵，並逐一列出移除項目；更新進行期間會略過此操作，避免部分寫入的升級狀態在遷移完成前遭到移除。
- 如果無法剖析 `openclaw.json`，且無法復原最近一次確認良好的設定，`doctor --fix` 會將原始檔保留為 `openclaw.json.clobbered.<timestamp>`、維持目前檔案不變，並以錯誤結束，而不會寫入不完整的替代內容。
- 當閘道生命週期由其他監督程式管理時，請設定 `OPENCLAW_SERVICE_REPAIR_POLICY=external`。Doctor 仍會回報閘道／服務健康狀態並套用非服務修復，但會略過服務安裝、啟動、重新啟動、引導，以及舊版服務清理。
- Doctor 會回報受管理閘道已套用的堆積限制，以及依目前主機或容器記憶體限制所使用的自適應推導值。若要在修復流程以外取得相同報告，請使用 `openclaw gateway status`。
- 在 Linux 上，doctor 會忽略未啟用的額外類閘道 systemd 單元，且修復期間不會改寫執行中 systemd 閘道服務的命令／進入點中繼資料。請先停止服務，或使用 `openclaw gateway install --force` 取代使用中的啟動器。
- `doctor --fix --non-interactive` 會回報遺失或過時的閘道服務定義，但在更新修復模式以外不會安裝或改寫它們。若服務遺失，請執行 `openclaw gateway install`；若要取代啟動器，請執行 `openclaw gateway install --force`。
- 狀態完整性檢查會偵測工作階段目錄中的孤立逐字稿檔案。將它們封存為 `.deleted.<timestamp>` 需要互動式確認；`--fix`、`--yes` 和無頭執行都會將它們保留在原處。
- Doctor 會掃描 `~/.openclaw/cron/jobs.json`（或 `cron.store`）中的舊版排程工作格式，並在將標準資料列匯入 SQLite 前改寫它們。
- Doctor 會回報具有明確 `payload.model` 覆寫的排程工作，包括供應商命名空間計數及其與 `agents.defaults.model` 的不符之處，讓未繼承預設模型的排程工作能在認證或計費調查期間顯示出來。
- Doctor 會回報仍標記為執行中的排程工作（`state.runningAtMs`），這可能使 `openclaw cron list` 將它們顯示為 `running`。此檢查為唯讀：若目前沒有閘道正在執行已標記的工作，下次排程服務啟動時會記錄遭中斷的執行並清除標記。
- 在 Linux 上，如果使用者的 crontab 仍在執行已停止維護的舊版 `~/.openclaw/bin/ensure-whatsapp.sh`，doctor 會發出警告；當排程環境缺少 systemd 使用者匯流排環境時，該程式可能錯誤回報 `Gateway inactive`。
- 啟用 WhatsApp 時，doctor 會檢查閘道事件迴圈是否效能下降，且本機 `openclaw-tui` 用戶端是否仍在執行。`doctor --fix` 只會停止經驗證的本機終端介面用戶端，避免 WhatsApp 回覆因過時的終端介面重新整理迴圈而排隊等候。
- 若存在 HTTP(S) Proxy 環境變數但停用了 `tools.web.fetch.useTrustedEnvProxy`，doctor 會說明 `web_fetch` 仍使用直接路由、執行簡短的直接 TLS 連線探測，並指出明確的選擇啟用方式。它絕不會自動啟用 Proxy 信任。
- Doctor 會將舊版 `codex/*` 和 `openai-codex/*` 模型參照改寫為標準 `openai/*` 參照，範圍涵蓋主要模型、備援模型、模型允許清單、影像／影片生成模型、心跳偵測／子代理程式／壓縮覆寫、鉤子、頻道模型覆寫、排程承載資料，以及過時的工作階段／逐字稿路由固定設定。`--fix` 也會在安全時合併舊版 `models.providers.codex` 和 `models.providers.openai-codex` 設定，將舊版 `openai-codex:*` 認證設定檔和 `auth.order.openai-codex` 項目遷移至 `openai:*`，將 Codex 意圖移至供應商／模型範圍的 `agentRuntime.id: "codex"` 項目，移除過時的整體代理程式／工作階段執行階段固定設定，並讓修復後的 OpenAI 代理程式參照繼續使用 Codex 認證路由，而非直接使用 OpenAI API 金鑰認證。
- 當非空白的 `auth.order.<provider>` 清單所參照的設定檔已全部不存在，但仍有相容的已儲存認證資訊時，Doctor 會回報此情況。`doctor --fix` 只會刪除這些過時的覆寫，恢復自動為各代理程式選取認證資訊；明確的空白順序、仍有部分有效項目的清單，以及沒有相容已儲存認證資訊的順序都會保持不變。若使用中的 SQLite 認證儲存區無法讀取或格式錯誤，doctor 會說明略過此修復的原因。如果執行中閘道的設定重新載入模式不會自動套用寫入，請先重新啟動閘道，再重新檢查認證狀態。
- Doctor 會清理舊版 OpenClaw 遺留的外掛相依套件暫存狀態，並為將主機 `openclaw` 套件宣告為對等相依套件的受管理 npm 外掛重新建立連結。它也會修復設定所參照但遺失的可下載外掛（`plugins.entries`、已設定的頻道、已設定的供應商／搜尋設定、已設定的代理程式執行階段）。套件更新期間，doctor 會略過套件管理器的外掛修復，直到套件交換完成；若已設定的外掛之後仍需復原，請重新執行 `openclaw doctor --fix`。如果下載失敗，doctor 會回報安裝錯誤，並保留已設定的外掛項目供下次修復嘗試使用。
- 當外掛探索功能正常時，Doctor 會從 `plugins.allow`/`plugins.deny`/`plugins.entries` 移除遺失的外掛 ID，以及相符但懸空的頻道設定、心跳偵測目標和頻道模型覆寫，以修復過時的外掛設定。
- Doctor 會隔離無效的外掛設定，方法是停用受影響的 `plugins.entries.<id>` 項目並移除其無效的 `config` 承載資料。閘道啟動時本來就只會略過該錯誤外掛，因此其他外掛和頻道仍可繼續執行。
- Doctor 會移除已淘汰的 `plugins.entries.codex.config.codexDynamicToolsProfile`；Codex app-server 一律將 Codex 原生工作區工具保留為原生工具。
- Doctor 會自動將舊版扁平 Talk 設定（`talk.voiceId`、`talk.modelId` 等）遷移至 `talk.provider` + `talk.providers.<provider>`。當唯一差異只是物件鍵順序時，重複執行 `doctor --fix` 不再回報／套用 Talk 正規化。
- Doctor 包含記憶體搜尋就緒狀態檢查，並可在缺少嵌入認證資訊時建議 `openclaw configure --section model`。
- 未設定命令擁有者時，Doctor 會發出警告。命令擁有者是獲准執行僅限擁有者命令及核准危險動作的人類操作員帳號。DM 配對只允許某人與機器人對話；如果你在首次擁有者引導功能存在之前就已核准某位傳送者，請明確設定 `commands.ownerAllowFrom`。
- 若已設定 Codex 模式代理程式，且操作員的 Codex 主目錄中存在個人 Codex 命令列介面資產，Doctor 會回報資訊性提示。本機 Codex app-server 啟動會使用隔離的各代理程式主目錄；如有需要，請先安裝 Codex 外掛，再使用 `openclaw migrate plan codex` 盤點應刻意提升使用的資產。
- 若預設代理程式允許使用的 Skills 在目前執行階段環境中不可用（缺少二進位檔、環境變數、設定或作業系統需求），Doctor 會發出警告。`doctor --fix` 可透過 `skills.entries.<skill>.enabled=false` 停用這些不可用的 Skills；若要讓 Skills 保持啟用，請改為安裝／設定缺少的需求。
- 若已啟用沙箱模式但 Docker 不可用，doctor 會回報明確的警告及修復方式（`install Docker` 或 `openclaw config set agents.defaults.sandbox.mode off`）。
- 若存在舊版沙箱登錄檔或分片目錄（`~/.openclaw/sandbox/containers.json`、`~/.openclaw/sandbox/browsers.json`、`~/.openclaw/sandbox/containers/` 或 `~/.openclaw/sandbox/browsers/`），doctor 會回報它們；`--fix` 會將有效項目遷移至 SQLite，並隔離無效的舊版檔案。
- 若 `gateway.auth.token`/`gateway.auth.password` 由 SecretRef 管理，且在目前命令路徑中不可用，doctor 會回報唯讀警告，且不會寫入明文備援認證資訊。對於由 exec 支援的 SecretRef，除非存在 `--allow-exec`，否則 doctor 會略過執行。
- 若在修復路徑中檢查頻道 SecretRef 失敗，doctor 會繼續執行並回報警告，而不會提前結束。
- 狀態目錄遷移後，如果已啟用的預設 Telegram 或 Discord 帳號依賴環境備援，而 doctor 程序無法取得 `TELEGRAM_BOT_TOKEN` 或 `DISCORD_BOT_TOKEN`，doctor 會發出警告。
- Telegram `allowFrom` 使用者名稱自動解析（`doctor --fix`）需要目前命令路徑中可解析的 Telegram Token。如果無法檢查 Token，doctor 會回報警告，並略過該次執行的自動解析。

## macOS：`launchctl` 環境變數覆寫

如果你先前執行過 `launchctl setenv OPENCLAW_GATEWAY_TOKEN ...`（或 `...PASSWORD`），該值會覆寫你的設定檔，並可能造成持續的「未授權」錯誤。

```bash
launchctl getenv OPENCLAW_GATEWAY_TOKEN
launchctl getenv OPENCLAW_GATEWAY_PASSWORD

launchctl unsetenv OPENCLAW_GATEWAY_TOKEN
launchctl unsetenv OPENCLAW_GATEWAY_PASSWORD
```

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [閘道 doctor](/zh-TW/gateway/doctor)
