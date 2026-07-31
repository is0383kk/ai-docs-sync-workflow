---
read_when:
    - 發布 Skills
    - 偵錯發布失敗
summary: Skill 資料夾格式、必要檔案、支援成品、限制。
x-i18n:
    generated_at: "2026-07-26T07:45:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fdf16a589b8961ccd9181a53a9fa92a358952b9147d22eaf977f23e0b4b4d653
    source_path: clawhub/skill-format.md
    workflow: 16
---

# Skill 格式

## 磁碟上

Skill 是一個資料夾。

必要項目：

- `SKILL.md`（或 `skill.md`；也接受舊版的 `skills.md`）

選用項目：

- 任何支援用的一般檔案（請參閱「Skill 檔案」）
- `.clawhubignore`（發布時的忽略模式，舊版為 `.clawdhubignore`）
- `.gitignore`（也會套用）

## GitHub 匯入

網頁版 GitHub 匯入工具的限制比本機發布／同步更嚴格。它只會探索已登入 GitHub 帳號擁有的公開、非分支儲存庫中的
`SKILL.md` 或舊版 `skills.md` 檔案。它不會匯入私人儲存庫、分支、
已封存／停用的儲存庫或第三方公開儲存庫。

本機安裝中繼資料（由命令列介面寫入）：

- `<skill>/.clawhub/origin.json`（舊版為 `.clawdhub`）

工作目錄安裝狀態（由命令列介面寫入）：

- `<workdir>/.clawhub/lock.json`（舊版為 `.clawdhub`）

## `SKILL.md`

- 含選用 YAML frontmatter 的 Markdown。
- 伺服器會在發布期間從 frontmatter 擷取中繼資料。
- `description` 會用作 UI／搜尋中的 Skill 摘要。

若要建立可攜式 Agent Skills，`name` 應與父目錄相符，並使用
1–64 個小寫字母、數字或連字號。ClawHub 會將可路由的 slug 與
目錄顯示名稱分開保存，因此其他用戶端的既有名稱仍可發布，
且不會遭到靜默改寫。目錄清單在視覺上可能會縮短過長的名稱，
但不會變更儲存的名稱。

## Frontmatter 中繼資料

Skill 中繼資料是在 `SKILL.md` 頂端的 YAML frontmatter 中宣告。這會告知登錄檔（以及安全分析）你的 Skill 執行時需要什麼。

### 基本 frontmatter

```yaml
---
name: my-skill
description: 此 Skill 功能的簡短摘要。
version: 1.0.0
---
```

### 執行階段中繼資料（`metadata.openclaw`）

在 `metadata.openclaw` 下宣告 Skill 的執行階段需求（別名：`metadata.clawdbot`、`metadata.clawdis`）。

```yaml
---
name: my-skill
description: 透過 Todoist API 管理工作。
metadata:
  openclaw:
    requires:
      env:
        - TODOIST_API_KEY
      bins:
        - curl
    primaryEnv: TODOIST_API_KEY
---
```

針對 Skill 執行前必須存在的環境變數，請使用 `requires.env`。需要各變數的中繼資料時，請使用 `envVars`，包括使用 `required: false` 的選用變數。

### 完整欄位參考

| 欄位               | 類型       | 說明                                                                                                                                         |
| ------------------ | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `requires.env`     | `string[]` | Skill 預期存在的必要環境變數。                                                                                                               |
| `requires.bins`    | `string[]` | 必須全部安裝的命令列介面二進位檔。                                                                                                           |
| `requires.anyBins` | `string[]` | 至少必須存在一個的命令列介面二進位檔。                                                                                                       |
| `requires.config`  | `string[]` | Skill 讀取的設定檔路徑。                                                                                                                      |
| `primaryEnv`       | `string`   | Skill 的主要認證資訊環境變數。                                                                                                               |
| `envVars`          | `array`    | 含 `name`、選用的 `required` 及選用的 `description` 的環境變數宣告。選用環境變數請設定 `required: false`。 |
| `always`           | `boolean`  | 若為 `true`，Skill 將永遠啟用（不需要明確安裝）。                                                                                 |
| `skillKey`         | `string`   | 覆寫 Skill 的叫用鍵。                                                                                                                         |
| `emoji`            | `string`   | Skill 的顯示表情符號。                                                                                                                        |
| `homepage`         | `string`   | Skill 首頁或文件的 URL。                                                                                                                      |
| `os`               | `string[]` | 作業系統限制（例如 `["macos"]`、`["linux"]`）。                                                                                 |
| `install`          | `array`    | 相依性安裝規格（請參閱下文）。                                                                                                               |
| `nix`              | `object`   | Nix 外掛規格（請參閱 README）。                                                                                                               |
| `config`           | `object`   | Clawdbot 設定規格（請參閱 README）。                                                                                                          |

### 安裝規格

若 Skill 需要安裝相依性，請在 `install` 陣列中宣告：

```yaml
metadata:
  openclaw:
    install:
      - kind: brew
        formula: jq
        bins: [jq]
      - kind: node
        package: typescript
        bins: [tsc]
```

支援的安裝類型：`brew`、`node`、`go`、`uv`。

### 選用環境變數

在 `metadata.openclaw.envVars` 下宣告選用環境變數，並設定 `required: false`。請勿將選用項目新增至 `requires.env`，因為 `requires.env` 表示缺少這些變數時 Skill 無法執行。

```yaml
metadata:
  openclaw:
    primaryEnv: TODOIST_API_KEY
    envVars:
      - name: TODOIST_API_KEY
        required: true
        description: 用於已驗證要求的 Todoist API 權杖。
      - name: TODOIST_PROJECT_ID
        required: false
        description: 使用者未指定時採用的選用預設專案 ID。
```

### 這為何重要

ClawHub 的安全分析會檢查 Skill 宣告的內容是否與其實際行為相符。如果程式碼參照 `TODOIST_API_KEY`，但 frontmatter 未在 `requires.env`、`primaryEnv` 或 `envVars` 下宣告，分析便會標記中繼資料不符。保持宣告準確有助於 Skill 通過審查，也能協助使用者瞭解他們正在安裝什麼。

### 範例：完整 frontmatter

```yaml
---
name: todoist-cli
description: 從命令列管理 Todoist 工作、專案和標籤。
version: 1.2.0
metadata:
  openclaw:
    requires:
      env:
        - TODOIST_API_KEY
      bins:
        - curl
    primaryEnv: TODOIST_API_KEY
    envVars:
      - name: TODOIST_API_KEY
        required: true
        description: Todoist API 權杖。
      - name: TODOIST_PROJECT_ID
        required: false
        description: 選用的預設專案 ID。
    emoji: "\u2705"
    homepage: https://github.com/example/todoist-cli
---
```

## Skill 檔案

發布接受 Skill 資料夾中的所有一般檔案，不限副檔名。忽略檔案、
隱藏路徑、符號連結、macOS 中繼資料及伺服器端大小限制仍然適用。

- 大小受限且包含有效 UTF-8 的檔案，可以逸出的純文字形式預覽，並納入
  有限文字分析。
- 其他檔案會保留其確切位元組，並可供下載。
- 安全掃描器會收到完整的已儲存成品；文字偵測是呈現與
  分析方面的考量，而不是上傳允許清單。

限制（伺服器端）：

- 套件組合總大小：50MB。
- 嵌入文字包含 `SKILL.md` + 最多約 40 個大小受限的 UTF-8 檔案（盡力而為的上限）。

## Slug

- 預設衍生自資料夾名稱。
- 套件範圍必須與 ClawHub 發布者代號完全相符。發布者代號可使用小寫字母、數字、連字號、句點和底線；開頭與結尾必須是小寫字母或數字。
- 套件 slug 必須為小寫且符合 npm 安全格式，例如 `@example.tools/demo-plugin` 或 `demo-plugin`。

## 版本管理與標籤

- 每次發布都會建立新版本（semver）。
- 標籤是指向版本的字串指標；通常使用 `latest`。

## 授權條款

- 所有發布至 ClawHub 的 Skill 均依 `MIT-0` 授權。
- 任何人皆可使用、修改及重新散布已發布的 Skill，包括商業用途。
- 不要求標示出處。
- 請勿在 `SKILL.md` 中新增衝突的授權條款；ClawHub 不支援針對個別 Skill 覆寫授權條款。

## 付費 Skill

- ClawHub 不支援付費 Skill、個別 Skill 定價、付費牆或收益分潤。
- 請勿將定價中繼資料新增至 `SKILL.md`；該資料不屬於 Skill 格式，且不會使已發布的 Skill 成為付費項目。
- 如果 Skill 整合付費第三方服務，請在 Skill 指示與環境變數宣告中清楚記載外部費用和必要帳號（必要變數使用 `requires.env`，選用變數則使用 `envVars` 搭配 `required: false`）。
