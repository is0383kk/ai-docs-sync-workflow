---
read_when:
    - 發布 Skills 或外掛
    - 偵錯擁有者或套件範圍錯誤
    - 新增發布 UI、命令列介面或後端行為
summary: ClawHub 如何發布 Skills、外掛、擁有者、範圍、版本與審查。
x-i18n:
    generated_at: "2026-07-26T08:18:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 582dffaf4429e9f24d7c38f2809cc7dc05f8471e4ae2f9c6be60153cc8604e3f
    source_path: clawhub/publishing.md
    workflow: 16
---

# 發布

發布會將 Skills 資料夾或外掛套件傳送至 ClawHub，並歸屬於你選擇的擁有者。ClawHub 會檢查你的權杖是否可代表該擁有者發布，驗證中繼資料、名稱、版本、檔案及原始碼資訊，接著儲存該版本並啟動自動化安全性檢查。

如果驗證失敗，將不會發布任何內容。新版本在審查完成前，也可能不會出現在一般的安裝及下載介面中。

## Skills

最簡單的發布方式是使用命令列介面。登入後，發布本機 Skills 資料夾：

```bash
clawhub login
clawhub skill publish ./my-skill \
  --slug my-skill \
  --name "My Skill" \
  --owner <owner>
```

發布至組織擁有者時，請使用 `--owner <handle>`。省略此項即可用已驗證身分的使用者身分發布。發布時會略過未變更的內容。新的 Skills 會從 `1.0.0` 開始，之後的變更會自動發布下一個修補版本。只有需要明確指定版本時，才傳入 `--version`。

對於目錄存放庫，請使用 ClawHub 的可重複使用
[`skill-publish.yml` 工作流程](https://github.com/openclaw/clawhub/blob/main/.github/workflows/skill-publish.yml)。
它會針對 `root` 下的每個直屬 Skills 資料夾（預設值：
`skills`）呼叫 `skill publish`，或只處理以 `skill_path` 提供的資料夾。

```yaml
jobs:
  publish:
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@main
    with:
      owner: <owner>
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

使用 `dry_run: true` 可預覽新增及已變更的 Skills，而不實際發布。

## 外掛

外掛使用 npm 樣式的套件名稱。具命名空間的套件名稱會在名稱的第一部分包含擁有者：

```text
@owner/package-name
```

命名空間必須與所選的發布擁有者相符。如果你的套件名稱為
`@openclaw/dronzer`，則只能以 `@openclaw` 發布。如果你以
`@vintageayu` 發布，請將套件重新命名為 `@vintageayu/dronzer`。

這可防止套件宣稱發布者無權控制的組織命名空間。

如果你是 ClawHub 上已被宣稱或保留之組織、品牌、套件命名空間、擁有者帳號或命名空間的合法擁有者，請提出
[組織／命名空間認領問題](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml)，
並附上公開且非敏感的證明。請參閱
[組織與命名空間認領](/zh-TW/clawhub/namespace-claims)，了解應包含哪些內容，以及哪些內容不應放入公開問題中。

### 發布外掛前

- 選擇與套件命名空間相符的擁有者。
- 包含 `openclaw.plugin.json`。程式碼外掛還需要包含 `package.json`，以及
  `openclaw.compat.pluginApi` 和 `openclaw.build.openclawVersion`。
- 若要在首頁及外掛清單頁面顯示自訂外掛目錄圖示，
  請將 `icon` 加入 `openclaw.plugin.json`，並使用任何 HTTPS 圖片 URL。
- 包含原始碼存放庫及確切的提交中繼資料，或從 GitHub 支援的簽出目錄使用命令列介面，以便其自動偵測這些資訊。
- 發布前執行 `clawhub package validate <source>`。若遇到套件、
  資訊清單、SDK 匯入或成品相關問題，請參閱
  [外掛驗證修正](/zh-TW/clawhub/plugin-validation-fixes)。
- 建立版本前執行 `clawhub package publish <source> --dry-run`。
- 在自動化安全性檢查與驗證完成前，新版本預期不會出現在公開安裝介面中。

### 套件的受信任發布

套件的受信任發布需要兩個設定步驟：

1. 先透過一般手動方式或使用權杖驗證的
   `clawhub package publish` 發布套件一次。這會建立套件資料列，並確立可變更其受信任發布者設定的套件管理員。
2. 由套件管理員設定 GitHub Actions 受信任發布者組態：

```bash
clawhub package trusted-publisher set @owner/package-name \
  --repository owner/repo \
  --workflow-filename package-publish.yml
```

設定組態後，日後支援的 GitHub Actions 發布便可使用 OIDC／受信任發布，而無須在存放庫中儲存長效 ClawHub 權杖。所設定的存放庫及工作流程檔名必須與 GitHub Actions OIDC 宣告相符。如果你也傳入 `--environment <name>`，GitHub Actions 的環境宣告必須與該名稱完全相符。

設定受信任發布者組態時，ClawHub 會驗證所設定的 GitHub 存放庫。公開存放庫可透過公開的 GitHub 中繼資料進行驗證。私有存放庫則需要 ClawHub 具備該存放庫的 GitHub 存取權，例如透過未來安裝的 ClawHub GitHub App，或其他已授權的 GitHub 整合。

目前的可重複使用套件發布工作流程，在 `id-token: write` 可用時，支援對 `workflow_dispatch` 發布進行無密鑰的受信任發布。透過推送標籤進行的實際發布仍需要 `clawhub_token`，因此請保留 `CLAWHUB_TOKEN`，以供標籤版本、首次發布、不受信任的套件或緊急發布使用。

使用下列命令檢查或移除組態：

```bash
clawhub package trusted-publisher get @owner/package-name
clawhub package trusted-publisher delete @owner/package-name
```

刪除受信任發布者組態是復原方式。在套件管理員重新設定組態前，這會停用日後受信任發布權杖的簽發。

## 常見問題

### 套件命名空間必須與所選擁有者相符

如果套件命名空間與所選擁有者不相符，ClawHub 會拒絕發布：

```text
套件命名空間 "@openclaw" 必須與所選擁有者 "@vintageayu" 相符。
請以 "@openclaw" 發布，或將此套件重新命名為 "@vintageayu/dronzer"。
```

若要修正此問題，請選擇套件命名空間所指定的擁有者，或重新命名套件，使其命名空間與你可代表發布的擁有者相符。

如果套件名稱已有正確的命名空間，但套件由錯誤的發布者擁有，請改為轉移擁有權：

```sh
clawhub package transfer @opik/opik-openclaw --to opik
```

只有當你同時擁有目前擁有者及目標發布者的管理員存取權時，才能使用套件或 Skills 轉移。套件轉移無法讓你發布至無權管理的命名空間。

如果你無法存取目前的擁有者，但認為你的組織、專案或品牌是該命名空間的合法擁有者，請提出
[組織／命名空間認領問題](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml)，
並附上公開且非敏感的證明，以供工作人員審查。提出前請參閱
[組織與命名空間認領](/zh-TW/clawhub/namespace-claims)。

這可保護組織命名空間。名稱為 `@openclaw/dronzer` 的套件會宣稱
`@openclaw` 命名空間，因此只有能存取 `@openclaw` 擁有者的發布者才能發布該套件。
