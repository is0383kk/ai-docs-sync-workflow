---
read_when:
    - 你希望代理程式要求取得精選的 1Password 機密資料
    - 你需要針對每個祕密設定核准政策與稽核記錄
    - 你正在為 OpenClaw 設定 1Password 服務帳號
summary: 使用選用的 1Password 外掛，作為經稽核的代理程式機密資訊中介服務
title: 1Password 密鑰代理服務
x-i18n:
    generated_at: "2026-07-26T08:42:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 255ab4fd2c63754fef29d3ea87dcedc9ca2bd2f34bec1f81139e2ce5b6acdba2
    source_path: plugins/onepassword.md
    workflow: 16
---

# 1Password 機密資訊代理程式

內建的 `onepassword` 外掛為代理程式提供一項受政策控制的工具，用於
讀取精選的一組 1Password 欄位。此工具預設為停用，且在
`plugins.entries.onepassword.config` 存在之前不會執行任何動作。

這是代理程式工具，而非 SecretRef 提供者。它不會注入環境
變數，也不會解析 OpenClaw 設定中的機密資訊。

## 安全模型

- 僅限服務帳號驗證。權杖會保留在本機認證資訊
  檔案中，且絕不接受在 `openclaw.json` 中設定。
- 僅限精選登錄項目。代理程式可以列出已設定的 slug，但此外掛絕不會
  列舉 1Password 保險庫。
- 每個 slug 可設定 `auto`、`approve` 或 `deny` 政策。
- 核准授權會過期。快取值絕不會繞過目前的政策。
- 每次存取嘗試都會記錄在 OpenClaw 的共用 SQLite 狀態中。稽核
  資料列包含所提供的理由；理由不得含有敏感資訊。代理程式
  絕不會將擷取到的值或服務權杖複製到稽核資料列中。
- 目前的工具執行結束後，OpenClaw 所管理的對話記錄持久化機制
  會將成功的 `get` 值替換為已遮蔽的中繼資料。
- 在該次執行期間，模型可以看到此值。如果模型將其複製到
  後續的工具呼叫或回覆中，該筆獨立記錄不在此外掛的
  持久化掛鉤涵蓋範圍內。請嚴格限制政策範圍，且不要要求模型覆述
  此值。
- 每次快取未命中時，此外掛會呼叫一次 `op`。它不會針對速率限制或
  其他失敗重試。
- 每次 `op` 呼叫都會在最小化環境中執行，並停用 1Password
  桌面應用程式整合（`OP_LOAD_DESKTOP_APP_SETTINGS=false`、
  `OP_BIOMETRIC_UNLOCK_ENABLED=false`），因此安裝在
  閘道主機上的 1Password 應用程式絕不會觸發生物辨識或 macOS 權限對話框。

僅授予服務帳號對外掛設定中登錄之保險庫和項目的讀取權限。

## 開始之前

需要準備：

- 在閘道主機上安裝 1Password 命令列介面（`op`）
- 可存取所選項目的 1Password 服務帳號
- 專用的服務帳號權杖檔案

啟用內建外掛：

```bash
openclaw plugins enable onepassword
```

在 OpenClaw 狀態目錄下建立權杖目錄和檔案：

```bash
mkdir -p ~/.openclaw/credentials/onepassword
chmod 700 ~/.openclaw/credentials/onepassword
printf '%s' "$OP_SERVICE_ACCOUNT_TOKEN" > \
  ~/.openclaw/credentials/onepassword/service-account-token
chmod 600 ~/.openclaw/credentials/onepassword/service-account-token
unset OP_SERVICE_ACCOUNT_TOKEN
```

設定 `OPENCLAW_STATE_DIR` 時，請將 `~/.openclaw` 替換為該目錄。
如果群組或其他使用者可讀取或寫入權杖檔案，此外掛會警告一次。

## 設定已登錄的機密資訊

將外掛設定新增至 `openclaw.json`：

```jsonc
{
  "plugins": {
    "entries": {
      "onepassword": {
        "enabled": true,
        "config": {
          "vault": "Automation",
          "defaultPolicy": "approve",
          "cacheTtlSeconds": 300,
          "grantTtlHours": 720,
          "opTimeoutMs": 15000,
          "items": {
            "repository-token": {
              "item": "Repository automation token",
              "field": "credential",
              "policy": "approve",
              "description": "Token for repository automation",
            },
            "model-key": {
              "item": "Model provider key",
              "vault": "Agent credentials",
              "policy": "auto",
            },
          },
        },
      },
    },
  },
}
```

Slug 使用小寫字母、數字和連字號，以字母或
數字開頭，且最多包含 64 個字元。一個登錄可包含最多 32 個
slug；說明最多可包含 200 個字元。`field` 接受一個欄位
標籤或 ID，不得包含逗號，且預設為 `credential`。
項目層級的 `vault` 會覆寫預設保險庫。`opBin` 可設定
`op` 可執行檔的絕對路徑；否則，外掛會從 `PATH` 解析 `op`。
項目標題不得以連字號開頭。

## 使用代理程式工具

工具名稱為 `onepassword`。

列出已登錄的 slug：

```json
{ "action": "list" }
```

結果僅包含 slug、說明、政策，以及長期
授權是否有效。它絕不包含機密值，也不會查詢 1Password。

要求一項機密資訊：

```json
{
  "action": "get",
  "slug": "repository-token",
  "reason": "Authenticate the requested repository operation"
}
```

`reason` 為必填，不得為空，且限制為 300 個字元。
成功的 `get` 會傳回該值，以及所設定的 slug、項目標題和
欄位標籤。

工具結構描述也宣告了一個內部 `authorizationNonce` 參數。
政策層會在評估要求後注入此參數，將授權
交給執行中的工具呼叫。絕不要手動設定：政策掛鉤會覆寫
任何提供的值，而未知值會使要求失敗。

## 政策層級與核准

- `auto`：立即擷取並稽核要求。
- `deny`：阻擋並稽核要求。
- `approve`：使用未過期的長期授權，或要求人員選擇允許一次、
  永遠允許或拒絕。

允許一次僅授權目前的工具呼叫。永遠允許會將該代理程式和 slug 的長期
授權寫入 SQLite；其他代理程式必須各自取得
核准。僅當呼叫端具有明確的代理程式
身分時，OpenClaw 才會提供永遠允許選項。授權會在 `grantTtlHours` 後過期，其預設值為 720 小時。
未解決或逾時的核准會拒絕要求；核准
等待時間上限為 600 秒。此外掛最多保留 1,024 個長期授權；達到該
上限時，最舊的授權會被淘汰，而其代理程式必須在下次存取時取得核准。

每個經過評估的授權皆為單次使用，並透過共用 SQLite 狀態交給執行中的工具
呼叫，因此即使閘道處理程序中有多個
外掛執行個體，此交接仍可運作。未使用的授權會在
600 秒的核准時限後過期。

記憶體內快取預設為 300 秒，且受限於所設定的
slug 登錄。將 `cacheTtlSeconds` 設為 `0` 即可停用快取。每次查詢快取前都會先評估政策，
且快取命中也會進行稽核。執行階段設定重新載入會在每個政策與執行邊界生效；停用外掛，或
移除、拒絕或重新指定 slug，都會使待處理的授權和
快取值失效。

## 檢查狀態與稽核歷程記錄

顯示就緒狀態和登錄項目數量：

```bash
openclaw onepassword status
```

此命令會回報權杖檔案是否存在、`op` 是否已解析及其路徑、
已登錄的項目數量，以及各政策的數量。它絕不會讀取或列印
權杖或機密值。

顯示最近 50 筆稽核資料列：

```bash
openclaw onepassword audit
openclaw onepassword audit --limit 100
```

資料列依時間由新到舊排列，並顯示時間戳記、代理程式、slug、結果、嘗試失敗時的
`errorCode`，以及截短的理由。理由會按照
提供的內容儲存；代理程式絕不會將擷取到的值新增至稽核記錄。

## 1Password 命令列介面行為

每次快取未命中時，都會使用所設定的項目、保險庫和精確
欄位選擇器、JSON 輸出、有上限的逾時，以及 `--cache=false` 執行 `op item get`。子處理程序
只會收到該欄位，而非完整項目。子處理程序環境中只存在
`OP_SERVICE_ACCOUNT_TOKEN` 和 `HOME`。

此外掛只會嘗試一次。應在稍後的代理程式要求前等待，以處理
`RATE_LIMITED` 錯誤；此外掛不會建立自動重試
迴圈。

## 錯誤代碼

失敗的嘗試會在工具結果和稽核
資料列中包含一個封閉集合內的錯誤代碼。

1Password 存取錯誤：

| 代碼              | 含義                                                          |
| ----------------- | ---------------------------------------------------------------- |
| `TOKEN_MISSING`   | 權杖檔案遺失或為空                                   |
| `OP_NOT_FOUND`    | 無法解析 `op` 二進位檔                                |
| `ITEM_NOT_FOUND`  | 所設定的項目不在保險庫中                              |
| `FIELD_NOT_FOUND` | 項目中沒有設定的欄位；會列出可用的標籤 |
| `RATE_LIMITED`    | 已達到 1Password 服務帳號速率限制                     |
| `AUTH_FAILED`     | 服務帳號驗證失敗                            |
| `TIMEOUT`         | `op` 超過 `opTimeoutMs`                                      |
| `OP_ERROR`        | 任何其他 `op` 失敗或無效輸出                         |

政策與驗證錯誤：

| 代碼                                               | 含義                                                                      |
| -------------------------------------------------- | ---------------------------------------------------------------------------- |
| `INVALID_ACTION`、`INVALID_REASON`、`INVALID_SLUG` | 要求未通過輸入驗證                                              |
| `UNKNOWN_SLUG`                                     | Slug 不在設定的登錄中                                       |
| `TOOL_CALL_ID_MISSING`                             | 呼叫到達時沒有工具呼叫 ID                                          |
| `POLICY_NOT_EVALUATED`                             | 此呼叫沒有相符的授權；要求未經政策核准 |
| `POLICY_CHANGED`                                   | 設定在核准與執行之間發生變更                                |
| `GRANT_EXPIRED`                                    | 長期授權在執行前失效                                       |
| `APPROVAL_CANCELLED`                               | 執行在核准待處理期間遭到中止                           |
