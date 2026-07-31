---
read_when:
    - ClawHub 命令列介面或 OpenClaw 登錄檔命令失敗
    - 套件無法安裝、發布或更新
summary: 疑難排解 ClawHub 登入、安裝、發布、更新與 API 問題。
x-i18n:
    generated_at: "2026-07-26T08:11:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fc789fcc891cf8c44b5d1a10d38a4e6dd4dec9474d8d13f8058ea1c3392a9f91
    source_path: clawhub/troubleshooting.md
    workflow: 16
---

# 疑難排解

## `clawhub login` 會開啟瀏覽器，但始終無法完成

命令列介面會在瀏覽器登入期間啟動一個短暫運作的本機回呼伺服器。

- 請確認瀏覽器可以連線至 `http://127.0.0.1:<port>/callback`。
- 如果始終未收到回呼，請檢查本機防火牆、VPN 和 Proxy 規則。
- 在無頭環境中，請在 ClawHub Web UI 建立 API Token，然後執行：

```bash
clawhub login --token clh_...
```

## `whoami` 或 `publish` 傳回 `Unauthorized` (401)

- 請使用 `clawhub login` 重新登入。
- 如果你使用自訂設定路徑，請確認 `CLAWHUB_CONFIG_PATH` 指向
  包含目前 Token 的檔案。
- 如果你使用 API Token，請確認該 Token 未在 Web UI 中遭撤銷。

## 搜尋或安裝傳回 `Rate limit exceeded` (429)

請閱讀回應中的重試資訊：

- `Retry-After`：重試前要等待的秒數。
- `RateLimit-Limit`：套用至此請求的限制。
- `RateLimit-Remaining`：標頭存在時，你的確切剩餘配額。在 `429` 上，此值為 `0`。
- `RateLimit-Reset` 或 `X-RateLimit-Reset`：重設時間。

如果許多使用者共用同一個對外 IP，即使每人只傳送少量請求，也可能
觸及匿名 IP 限制。請盡可能登入，並在所回報的延遲時間過後重試。

## 透過 Proxy 時搜尋或安裝失敗

命令列介面支援標準 Proxy 變數：

```bash
export HTTPS_PROXY=http://proxy.example.com:3128
clawhub search "my query"
```

支援的名稱包括 `HTTPS_PROXY`、`HTTP_PROXY`、`https_proxy` 和
`http_proxy`。

## 搜尋中未顯示某項 Skill

- 如果你知道確切的 slug 或擁有者頁面，請加以檢查。
- 請確認該版本為公開狀態，且未因掃描或審核而遭保留。
- 如果你擁有該 Skill，請登入並檢查它：

```bash
clawhub inspect @openclaw/demo
```

只有擁有者可見的診斷資訊可能會說明掃描、上傳閘門或審核狀態。

## 發布因缺少必要的中繼資料而失敗

對於 Skills，請檢查 `SKILL.md` frontmatter。應宣告必要的環境變數和
工具，讓使用者與掃描器能夠瞭解該套件。

對於外掛，請檢查 `package.json` 相容性中繼資料。發布程式碼外掛時，
需要 OpenClaw 相容性欄位，例如 `openclaw.compat.pluginApi` 和
`openclaw.build.openclawVersion`。

請先預覽發布承載內容：

```bash
clawhub package publish <source> --family code-plugin --dry-run
```

## 發布因 GitHub 擁有者或來源錯誤而失敗

ClawHub 使用 GitHub 身分和來源歸屬，將套件與其
發布者建立關聯。

- 請確認你已使用擁有該套件或可發布該套件的 GitHub 帳號登入。
- 請檢查來源 URL 是否公開，或 ClawHub 是否可存取。
- 對於 GitHub 來源，請使用 `owner/repo`、`owner/repo@ref` 或完整的 GitHub URL。

## 發布因命名空間已被認領或保留而失敗

如果發布因擁有者代稱、組織命名空間、套件範圍、Skill
slug 或套件名稱已被認領或保留而失敗，請先確認你是以符合該命名空間的
擁有者身分發布。對於外掛套件，
`@example-org/example-plugin` 等限定範圍名稱必須以相符的
`example-org` 擁有者身分發布。

如果你認為你的組織、專案或品牌才是命名空間的正當擁有者，但
無法管理目前的 ClawHub 擁有者，請使用公開且非敏感的證明建立
[組織／命名空間認領問題](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml)。
如需證明指引，以及瞭解哪些內容不應放入公開 Issue，請參閱
[組織與命名空間認領](/zh-TW/clawhub/namespace-claims)。

## `sync` 顯示找不到 Skills

`sync` 會尋找包含 `SKILL.md` 或 `skill.md` 的資料夾。

請將它指向你要掃描的根目錄：

```bash
clawhub sync --root /path/to/skills
```

如果你不確定會發布哪些內容，請先預覽：

```bash
clawhub sync --all --dry-run --no-input
```

## `update` 因本機變更而拒絕執行

本機檔案與 ClawHub 已知的任何版本皆不相符。請選擇一種做法：

- 保留本機編輯並略過更新。
- 以已發布的版本覆寫：

```bash
clawhub update @openclaw/demo --force
```

- 以新的 slug 或分支版本發布你編輯過的副本。

## 在 OpenClaw 中安裝外掛失敗

- 請使用明確的 ClawHub 來源：

```bash
openclaw plugins install clawhub:<package>
```

- 請檢查套件詳細資料頁面的掃描狀態和相容性中繼資料。
- 請確認你的 OpenClaw 版本符合套件所宣告的
  相容性範圍。
- 如果套件遭隱藏、保留或封鎖，在擁有者解決問題前，
  該套件可能無法安裝。

## 公開 API 請求失敗

- 請遵守 `429` 重試標頭，並快取公開的清單／搜尋回應。
- 請將使用者連回標準的 ClawHub 項目頁面。
- 請勿在公開 API 介面之外鏡像隱藏、私人、遭保留或因審核而封鎖的內容。

如需端點詳細資料，請參閱 [HTTP API](/zh-TW/clawhub/http-api)。
