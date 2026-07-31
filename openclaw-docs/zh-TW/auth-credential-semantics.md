---
read_when:
    - 處理驗證設定檔解析或認證資訊路由作業
    - 偵錯模型驗證失敗或設定檔順序
summary: 認證設定檔的標準認證資訊適用資格與解析語意
title: 驗證認證資訊語意
x-i18n:
    generated_at: "2026-07-26T07:43:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b0516b1bb23f400d5ac5fd39a628736034440216ac22823eef061b38564dff0
    source_path: auth-credential-semantics.md
    workflow: 16
---

這些語意可讓選擇時與執行階段的驗證行為保持一致。以下項目共用這些語意：

- `resolveAuthProfileOrder`（設定檔排序）
- `resolveApiKeyForProfile`（執行階段認證資訊解析）
- `openclaw models status --probe`
- `openclaw doctor` 驗證檢查（`doctor-auth`）

## 穩定的探查原因代碼

探查結果包含一個 `status` 分類（`ok`、`auth`、`rate_limit`、`billing`、`timeout`、`format`、`unknown`、`no_model`）；若探查從未進行模型呼叫，還會包含穩定的 `reasonCode`：

| `reasonCode`             | 意義                                                                         |
| ------------------------ | ---------------------------------------------------------------------------- |
| `excluded_by_auth_order` | 設定檔未列入其供應商的明確驗證順序。                                         |
| `missing_credential`     | 未設定行內認證資訊或 SecretRef。                                              |
| `expired`                | 權杖 `expires` 已是過去時間。                                                |
| `invalid_expires`        | `expires` 不是有效的正數 Unix 毫秒時間戳記。                                  |
| `unresolved_ref`         | 無法解析已設定的 SecretRef。                                                  |
| `ineligible_profile`     | 設定檔與供應商設定不相容（包括格式錯誤的金鑰輸入）。                           |
| `no_model`               | 認證資訊存在，但未解析出可探查的模型候選項目。                               |

資格檢查會以 `ok` 作為可用認證資訊的原因代碼。

## 權杖認證資訊

權杖認證資訊（`type: "token"`）支援行內 `token` 和／或 `tokenRef`。

### 資格規則

1. 當 `token` 與 `tokenRef` 皆不存在時，權杖設定檔不具資格（`missing_credential`）。
2. `expires` 為選填。若存在，必須是大於 `0`，且不超過 JavaScript `Date` 時間戳記最大值（8640000000000000）的有限 Unix 紀元毫秒數。
3. 若 `expires` 無效（類型錯誤、`NaN`、`0`、負數、非有限值，或超出該最大值），設定檔會因 `invalid_expires` 而不具資格。
4. 若 `expires` 已是過去時間，設定檔會因 `expired` 而不具資格。
5. `tokenRef` 不會略過 `expires` 驗證。

### 解析規則

1. 解析器對 `expires` 的語意與資格語意一致。
2. 對於具資格的設定檔，可從行內值或 `tokenRef` 解析權杖內容。
3. 無法解析的參照會在 `models status --probe` 輸出中產生 `unresolved_ref`。

## 代理程式複製可攜性

代理程式驗證繼承採用唯讀穿透方式。當代理程式沒有本機設定檔時，會在執行階段從預設／主要代理程式儲存區解析設定檔，而不會將機密內容複製到其自身的認證資訊儲存區（`agents/<agentId>/agent/openclaw-agent.sqlite`）。

明確的複製流程（例如 `openclaw agents add`）會採用以下可攜性原則：

- `api_key` 與 `token` 設定檔具有可攜性，除非 `copyToAgents: false`。
- `oauth` 設定檔預設不具可攜性，因為重新整理權杖可能只能使用一次，或對輪替相當敏感。
- 僅當已知可安全地跨代理程式複製重新整理內容時，供應商所擁有的 OAuth 流程才可透過 `copyToAgents: true` 選擇加入；此選擇加入僅適用於設定檔包含行內存取／重新整理內容的情況。

除非目標代理程式另行登入並建立自己的本機設定檔，否則仍可透過唯讀穿透繼承使用不可攜的設定檔。

## 僅限設定的驗證路由

具有 `mode: "aws-sdk"` 的 `auth.profiles` 項目是路由中繼資料，而非儲存的認證資訊。當目標供應商使用 `models.providers.<id>.auth: "aws-sdk"`（外掛所擁有的 Amazon Bedrock 設定所寫入的路由）時，這些項目即為有效。即使認證資訊儲存區中沒有相符的項目，這些設定檔 ID 仍可能出現在 `auth.order` 和工作階段覆寫中。

請勿將 `type: "aws-sdk"` 寫入認證資訊儲存區；儲存的認證資訊只能是 `api_key`、`token` 或 `oauth`。若舊版 `auth-profiles.json` 包含這類標記，`openclaw doctor --fix` 會將其移至 `auth.profiles`，並從儲存區移除該標記。

## 明確驗證順序篩選

- 為供應商設定 `auth.order.<provider>` 或驗證儲存區順序覆寫時，`models status --probe` 只會探查仍保留在該供應商解析後驗證順序中的設定檔 ID。儲存的覆寫優先於 `auth.order` 設定。
- 若該供應商的某個已儲存設定檔未列入明確順序，之後也不會在未告知的情況下嘗試使用。探查輸出會以 `reasonCode: excluded_by_auth_order` 回報，並附上詳細資訊 `Excluded by auth.order for this provider.`

## 探查目標解析

- 探查目標可以來自驗證設定檔、環境認證資訊或 `models.json`（結果 `source`：`profile`、`env`、`models.json`）。
- 若供應商具有認證資訊，但 OpenClaw 無法為其解析出可探查的模型候選項目，`models status --probe` 會以 `reasonCode: no_model` 回報 `status: no_model`。

## 外部命令列介面認證資訊探索

- 只有當供應商、執行階段或驗證設定檔在目前作業的範圍內，或該外部來源已存在已儲存的本機設定檔時，才會探索外部命令列介面所擁有、僅供執行階段使用的認證資訊（`claude-cli` 的 Claude CLI、`openai` 的 Codex CLI、`minimax-portal` 的 MiniMax CLI）。
- 驗證儲存區呼叫端會選擇明確的外部命令列介面探索模式：`none` 僅用於持久化／外掛驗證、`existing` 用於重新整理已儲存的外部命令列介面設定檔，或 `scoped` 用於具體的供應商／設定檔集合。
- 唯讀／狀態路徑會傳入 `allowKeychainPrompt: false`；它們只使用檔案支援的外部命令列介面認證資訊，不會讀取或重複使用 macOS Keychain 結果。

## OAuth SecretRef 原則防護

SecretRef 輸入僅適用於靜態認證資訊。OAuth 認證資訊可在執行階段變動（重新整理流程會持久保存輪替後的權杖），因此由 SecretRef 支援的 OAuth 內容會使可變狀態分散在多個儲存區中。

- 若設定檔認證資訊為 `type: "oauth"`，該設定檔的任何認證資訊內容欄位都會拒絕 SecretRef 物件。
- 若 `auth.profiles.<id>.mode` 為 `"oauth"`，該設定檔由 SecretRef 支援的 `keyRef`/`tokenRef` 輸入會遭拒絕。
- 在啟動／重新載入機密準備和設定檔解析路徑中，違規會導致硬性失敗（擲回錯誤）。

## 與舊版相容的訊息

為了維持指令碼相容性，探查錯誤會讓以下第一行保持不變：

`Auth profile credentials are missing or expired.`

後續各行會以 `↳ Auth reason [code]: ...` 的格式提供易於理解的詳細資訊和穩定原因代碼。

## 相關內容

- [機密管理](/zh-TW/gateway/secrets)
- [驗證儲存](/zh-TW/concepts/oauth)
