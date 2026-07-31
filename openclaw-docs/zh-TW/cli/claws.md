---
read_when:
    - 你正在撰寫或驗證 CLAW.md 資訊清單
    - 你想要預覽或新增 Claw 中的一個代理程式
    - 你需要檢查 Claw 的擁有權、偏移或清理行為
summary: 建立、新增、更新及移除實驗性 Claw 代理程式套件
title: 爪子
x-i18n:
    generated_at: "2026-07-26T07:36:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: da4b52bdee2b4cf4898677aadeeabb2c0cf98e7c3c53cec6f0b4c6d0b8ab3ae5
    source_path: cli/claws.md
    workflow: 16
---

# `openclaw claws`

Claw 是用於一個新 OpenClaw 代理程式的版本化設定。它可以描述代理程式的可攜式身分、工作區檔案、Skills、外掛、MCP 伺服器和排程工作。特定執行框架的代理程式設定可包含在所參照的套件設定檔中。Claw 不會取代或修改現有的代理程式。

Claw 是實驗性功能。其結構描述、命令輸出和生命週期可能會變更。請明確啟用此命令介面：

```bash
export OPENCLAW_EXPERIMENTAL_CLAWS=1
```

目前的命令列介面會讀取本機套件目錄、`CLAW.md` 或分組的 JSON 資訊清單。透過 ClawHub 發布、搜尋和安裝完整 Claw 屬於另一條登錄機制軌道，目前尚未納入此命令介面。

## 建立 Claw 套件

套件包含 `package.json`、`CLAW.md` 資訊清單，以及該資訊清單參照的任何設定檔或工作區附屬檔案：

```json
{
  "name": "@acme/incident-triage-claw",
  "version": "1.0.0",
  "type": "module",
  "openclaw": { "claw": "CLAW.md" }
}
```

`CLAW.md` 以 YAML frontmatter 開頭。其 Markdown 本文供人員瞭解 Claw，不屬於代理程式設定：

```md
---
schemaVersion: 1
agent:
  id: incident-triage
  name: 事件分流
metadata:
  openclaw.config: profiles/openclaw.yml
workspace:
  bootstrapFiles: {}
packages: []
mcpServers: {}
cronJobs: []
---

# 事件分流

建立一個用於審查和分流事件的代理程式。
```

`metadata` 是可攜式使用端提示的字串對字串對應表。OpenClaw 的 `openclaw.config` 鍵指向選用的套件相對 YAML 設定檔。匯出的預設值為 `profiles/openclaw.yml`；此指標具規範效力，因此套件可選擇其他安全的相對 `.yml` 或 `.yaml` 路徑。

```yaml
schemaVersion: 1
agent:
  tools:
    profile: coding
    alsoAllow: [cron]
    deny: [exec]
    fs:
      workspaceOnly: true
  memory:
    search:
      enabled: true
      rememberAcrossConversations: true
      sources: [memory, sessions]
```

此設定檔僅存在於 Claw 套件內。OpenClaw 在檢查、新增、更新和匯出該 Claw 時會驗證並使用此設定檔；它不會被複製到使用者的一般 OpenClaw 設定路徑。其他執行框架可忽略具命名空間的中繼資料鍵，並使用可攜式資訊清單欄位。

同一個嚴格的第 1 版結構描述仍接受分組的 JSON 資訊清單。分組 JSON 使用同一個 `metadata.openclaw.config` 指標，而不會嵌入 OpenClaw 設定檔的第二份副本。本頁其餘結構描述片段使用 JSON，`CLAW.md` frontmatter 中也提供等效的鍵。

OpenClaw 套件設定檔可選擇執行中 OpenClaw 版本登錄的任何內建工具設定檔，然後以 `alsoAllow`、`deny` 和 `tools.fs.workspaceOnly: true` 進一步調整。Claw 不得將該欄位設為 `false` 並削弱主機檔案系統限制。`tools.allow` 仍可作為明確的允許清單，但不能與 `alsoAllow` 結合使用。Claw 也可設定 `memory.search.enabled`、選擇可攜式 `memory` 和 `sessions` 來源，並使用 `rememberAcrossConversations` 選擇啟用跨對話記憶。宣告 `sessions` 來源必須選擇啟用此功能。主機原則仍會限制這些設定，且 Claw 不會攜帶自訂設定檔定義、提供者、認證資訊、繫結或本機記憶路徑。所參照的設定檔上限為 256 KiB，必須是與 JSON 相容的 YAML，不得使用別名、錨點、標籤或合併鍵，且必須是套件內一般、非符號連結、非硬式連結的檔案。

套件和工作區路徑必須位於套件根目錄內。資訊清單上限為 1 MiB，套件中繼資料上限為 256 KiB，工作區來源則強制執行個別檔案和總計限制。工作區來源也會拒絕含符號連結的父目錄。

工作區檔案依路徑宣告，並從套件附屬檔案讀取。`SOUL.md` 等啟動檔案使用具名項目；其他檔案則使用套件相對來源和工作區相對目標：

```json
{
  "workspace": {
    "bootstrapFiles": {
      "SOUL.md": { "source": "workspace/SOUL.md" }
    },
    "files": [
      {
        "source": "workspace/reference/policy.md",
        "path": "reference/policy.md"
      }
    ]
  }
}
```

Skills 和外掛使用確切的 ClawHub 版本：

```json
{
  "packages": [
    {
      "kind": "skill",
      "source": "clawhub",
      "ref": "incident-triage",
      "version": "1.0.0"
    },
    {
      "kind": "plugin",
      "source": "clawhub",
      "ref": "@acme/audit-plugin",
      "version": "2.0.0"
    }
  ]
}
```

試執行會使用現有的 Skill 和外掛預檢路徑，在同意前解析確切的成品、完整性資訊和任何 ClawHub 信任警告。該警告仍會顯示在受完整性繫結的計畫中。套用時會安裝缺少的成品或重複使用相符的成品，並記錄 Claw 是引入還是參照每項資源。外掛仍是整個程序範圍的 OpenClaw 能力，而非每個代理程式各自安裝。

排程工作會宣告新代理程式的排定工作：

```json
{
  "cronJobs": [
    {
      "id": "daily-summary",
      "name": "每日事件摘要",
      "schedule": { "cron": "0 9 * * *", "timezone": "UTC" },
      "session": "isolated",
      "message": "摘要說明進行中的事件。"
    }
  ]
}
```

Claw 使用現有的閘道排程器，並將建立的工作繫結至新代理程式。預覽、來源追蹤、狀態和移除會涵蓋這些工作，而不會變更一般排程命令的行為。移除時會透過閘道重新讀取即時工作；若其擁有的定義在規劃後發生變更，則會保留該工作。

MCP 宣告使用現有的 `mcp.servers` 設定模型：

```json
{
  "mcpServers": {
    "statuspage": {
      "command": "npx",
      "args": ["--yes", "@acme/statuspage-mcp@1.0.0"],
      "env": { "STATUSPAGE_TOKEN": "${STATUSPAGE_TOKEN}" }
    }
  }
}
```

環境參照會維持為參照；Claw 不會嵌入解析後的機密值。沒有衝突的宣告會成為受管理項目，而完全相同的現有或共用宣告則會被參照。預覽、來源追蹤、狀態、匯出和移除遵循與其他 Claw 資源相同的擁有權原則。

## 檢查與預覽

驗證來源而不規劃本機變更：

```bash
openclaw claws inspect ./incident-triage.claw.json
```

預覽所有建議的生命週期動作：

```bash
openclaw claws add ./incident-triage.claw.json --dry-run --json
```

計畫會報告衍生出的代理程式和工作區、每個建議動作、必要條件、阻礙因素、不同的能力升級，以及 `planIntegrity` 摘要。能力記錄會顯示確切的套件、MCP、排定工作、沙箱、工具或心跳偵測影響。建立代理程式前請檢閱計畫：

```bash
openclaw claws add ./incident-triage.claw.json \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

僅有 `--yes` 並不足夠。OpenClaw 會重建計畫；若來源、目的地或即時設定在預覽後發生變更，便會拒絕同意。當套件預設值與本機狀態衝突時，請在預覽和套用期間同時使用 `--agent-id` 或 `--workspace`。若使用拋棄式設定檔或進行平行驗證，請傳入明確的 `--workspace`；`OPENCLAW_STATE_DIR` 會重新定位執行階段狀態，但不會變更預設工作區位置。

新增 Claw 會建立新的代理程式和工作區設定、寫入宣告的工作區檔案、安裝或重複使用宣告的 Skill 與外掛成品，並記錄套件、MCP 和排程來源。既有檔案不會被覆寫；如果擁有的內容已發生漂移，重試會採取封閉式失敗。

## 檢查已安裝的狀態

```bash
openclaw claws status
openclaw claws status incident-triage --json
openclaw doctor
```

`status` 會將已安裝的代理程式及其記錄的工作區、套件、MCP 和排程來源與目前狀態比較。它會報告未完成的安裝、缺少的資源和漂移，而不會變更本機狀態。`openclaw doctor` 會新增 Claw 專用診斷，以處理不完整的擁有權記錄、不安全的受管理檔案，以及無法透過即時閘道清冊佐證的排程工作。

Claw 來源追蹤會區分兩種關係：

- **受管理：**Claw 引入並目前管理此資源。當資源未變更且沒有衝突的擁有者時，它是清理候選項目。
- **已參照：**資源原本已獨立存在或由多方共用。移除時會解除此 Claw 的參照，且預設保留該資源。

這不是參照計數。一般的外掛、Skill 和代理程式命令會維持既有行為；Claw 在其上新增來源追蹤和受保護的生命週期操作。

## 更新已安裝的 Claw

依預設，更新會使用新增 Claw 時所記錄的來源。當該來源已移動，或測試其他套件目錄時，請使用 `--from`：

```bash
openclaw claws update incident-triage --dry-run --json
openclaw claws update incident-triage \
  --from ./incident-triage-next \
  --dry-run --json
```

計畫會比較目前的來源追蹤和即時狀態與目標資訊清單。它會報告代理程式、工作區、套件、MCP、排程和擁有權變更，包括能力升級和阻礙因素。能力升級具有獨立的機器可讀記錄，且在供人閱讀的輸出中包含具確切遮蔽效果的 `!` 行。已解析的套件完整性、安裝身分和任何信任警告也會包含在內。移除套件宣告會解除此 Claw 的關聯，而不會在更新期間解除安裝成品。最終的確切 `planIntegrity` 確認會繫結該已揭露集合以及一般內容變更。主機可使用相同記錄來顯示獨立對話方塊，或進行多代理程式彙總檢閱。以明確同意套用確切的已檢閱計畫：

```bash
openclaw claws update incident-triage \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

OpenClaw 會重建計畫，並在每次異動前對擁有的狀態執行比較後交換。已移除的套件宣告會解除相依性關聯，而不解除安裝成品。排程變更會重新讀取即時排程器定義，並在發生操作人員漂移時停止。套件安裝程式、來源設定寫入器和閘道排程器不屬於同一個交易。如果在外部異動後無法證明補償成功，OpenClaw 會回報錯誤代碼 `update_partial`，並附上結構化的 `status: partial`、保留不確定的來源追蹤，然後停止。請檢查 `claws status`、受影響的資源和 `openclaw doctor`；然後再次預覽，再重試或移除任何項目。

## 移除已安裝的 Claw

選擇清理項目之前，請先預覽移除操作：

```bash
openclaw claws remove incident-triage --dry-run --json
openclaw claws remove incident-triage \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

預設會移除符合資格的受管理狀態，並解除已參照狀態。已修改的檔案及具有其他目前擁有者的資源會予以保留或受到阻擋。清理選項是計畫摘要的一部分；`--yes` 絕不會擴大這些選項的範圍。全域安裝的外掛會予以保留，同時解除此 Claw 的參照；若要解除安裝程序範圍的外掛，請另行使用一般的外掛生命週期。

若要移除由 Claw 引入、未變更且沒有其他目前擁有者的參照，請在預覽和套用時都加入 `--remove-unused`。若要改為選取確切的已參照資源，請重複使用 `--remove-referenced`：

```bash
openclaw claws remove incident-triage \
  --dry-run \
  --remove-referenced 'plugin:@acme/audit-plugin@2.0.0'
```

只有在檢閱顯示的相依項目、獨立擁有者和預先存在的來源後，才能使用 `--force-referenced`。它允許在有這些衝突的情況下執行所選清理；不會略過計畫完整性同意。

## 匯出已安裝的代理程式

匯出會建立新的套件目錄；如果目的地已存在或
受管理的狀態已發生偏移，則會失敗：

```bash
openclaw claws export incident-triage --out ./incident-triage-export --json
```

結果包含 `package.json`、標準 `CLAW.md`，以及受管理工作區的
輔助檔案。這是可攜式 Claw 套件，而不是整個執行個體的備份：不相關的
代理程式、認證資訊、工作階段，以及不受管理的本機狀態均不包含在內。

## 命令參考

| 命令                             | 用途                                             |
| ----------------------------------- | --------------------------------------------------- |
| `claws inspect <source>`            | 驗證套件目錄或分組資訊清單。   |
| `claws add <source>`                | 預覽或建立一個新的代理程式與工作區。      |
| `claws status [claw-or-agent]`      | 回報已安裝狀態、擁有權及偏移。       |
| `claws update <claw-or-agent>`      | 預覽或套用所選來源的變更。  |
| `claws remove <claw-or-agent>`      | 預覽或移除代理程式及符合條件的資源。 |
| `claws export <agent> --out <path>` | 從已安裝的代理程式建立可攜式套件。  |

使用 `--json` 取得實驗性的機器可讀輸出。

## 另請參閱

- [代理程式](/zh-TW/cli/agents)
- [Skills](/zh-TW/tools/skills)
- [外掛](/zh-TW/tools/plugin)
- [排程工作](/zh-TW/automation/cron-jobs)
- [MCP 設定](/zh-TW/gateway/configuration-reference#mcp)
