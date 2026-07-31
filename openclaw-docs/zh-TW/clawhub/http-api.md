---
read_when:
    - 新增／變更端點
    - 偵錯命令列介面 ↔ 登錄服務請求
summary: HTTP API 參考（公開端點 + 命令列介面端點 + 驗證）。
x-i18n:
    generated_at: "2026-07-26T07:10:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5b180bbd56d20a3d88c1fe74ccab0fd0ecbe0e8c9624cd1afd2070a2ca1f7fb3
    source_path: clawhub/http-api.md
    workflow: 16
---

# HTTP API

基礎 URL：`https://clawhub.ai`（預設值）。

所有 v1 路徑都位於 `/api/v1/...` 之下。
為了相容性，仍保留舊版 `/api/...` 和 `/api/cli/...`（請參閱 `DEPRECATIONS.md`）。
OpenAPI：`/api/v1/openapi.json`。

## 公開目錄重複使用

第三方目錄可以使用公開讀取端點來列出或搜尋 ClawHub Skills。請快取結果、遵守 `429`/`Retry-After`、將使用者連結回 ClawHub 的標準清單（`https://clawhub.ai/<owner>/skills/<slug>`），並避免暗示 ClawHub 為第三方網站背書。請勿嘗試在公開 API 介面之外鏡像隱藏、私人或遭審核封鎖的內容。

網頁 slug 捷徑可以跨登錄檔系列解析，但 API 用戶端應使用
讀取端點傳回的標準 URL，而不是自行重建路由
優先順序。

## 速率限制

強制執行模型：

- 匿名要求：依 IP 強制執行。
- 已驗證要求（有效的 Bearer 權杖）：依使用者配額桶強制執行。
- 如果權杖遺失或無效，行為會退回依 IP 強制執行。
- 當伺服器知道原因時，需要驗證的寫入端點不應只傳回 `Unauthorized`。
  權杖遺失、權杖無效或已撤銷，以及帳號遭刪除、封禁或停用時，
  應分別提供可採取行動的文字，讓命令列介面
  用戶端可以告知使用者遭封鎖的原因。

- 讀取：每個 IP 每分鐘 3000 次，每個金鑰每分鐘 12000 次
- 寫入：每個 IP 每分鐘 300 次，每個金鑰每分鐘 3000 次
- 下載：每個 IP 每分鐘 1200 次，每個金鑰每分鐘 6000 次（下載端點）

標頭：

- 舊版相容性：`X-RateLimit-Limit`、`X-RateLimit-Reset`
- 標準化：`RateLimit-Limit`、`RateLimit-Reset`
- 在 `429` 上：`X-RateLimit-Remaining: 0` 和 `RateLimit-Remaining: 0`
- 在 `429` 上：`Retry-After`

標頭語意：

- `X-RateLimit-Reset`：Unix 紀元的絕對秒數
- `RateLimit-Reset`：距離重設的秒數（延遲）
- `X-RateLimit-Remaining` / `RateLimit-Remaining`：存在時表示確切的剩餘配額。
  分片處理的成功要求會省略此標頭，而不會傳回近似的全域值。
- `Retry-After`：發生 `429` 時，重試前要等待的秒數（延遲）

`429` 回應範例：

```http
HTTP/2 429
content-type: text/plain; charset=utf-8
x-ratelimit-limit: 20
x-ratelimit-remaining: 0
x-ratelimit-reset: 1771404540
ratelimit-limit: 20
ratelimit-remaining: 0
ratelimit-reset: 34
retry-after: 34

已超過速率限制
```

用戶端指引：

- 如果 `Retry-After` 存在，請先等待該秒數再重試。
- 使用加入隨機抖動的退避機制，以避免同步重試。
- 如果缺少 `Retry-After`，請退回使用 `RateLimit-Reset`（或根據 `X-RateLimit-Reset` 計算）。

IP 來源：

- 只有在部署明確啟用受信任的轉送標頭時，才會使用受信任的用戶端 IP 標頭，包括 `cf-connecting-ip`。
- ClawHub 使用受信任的轉送標頭，在邊緣識別用戶端 IP。
- 如果沒有可用的受信任用戶端 IP，匿名要求會使用
  僅按速率限制種類劃分的備援配額桶。這些備援配額桶不包含
  呼叫者提供的路徑、slug、套件名稱、版本、查詢字串或其他
  成品參數。

## 錯誤回應

公開 v1 錯誤回應是使用 `content-type: text/plain; charset=utf-8` 的純文字。
這包括驗證失敗（`400`）、缺少公開資源（`404`）、驗證和
權限失敗（`401`/`403`）、速率限制（`429`）以及遭封鎖的下載。用戶端
應將回應主體讀取為人類可讀的字串。為了相容性，未知的查詢參數會被
忽略，但已識別且值無效的查詢參數會傳回
`400`。

## 公開端點（不需驗證）

### `GET /api/v1/search`

查詢參數：

- `q`（必要）：查詢字串
- `limit`（選用）：整數
- `highlightedOnly`（選用）：使用 `true` 篩選出精選 Skills
- `nonSuspiciousOnly`（選用）：使用 `true` 隱藏可疑（`flagged.suspicious`）Skills
- `nonSuspicious`（選用）：`nonSuspiciousOnly` 的舊版別名

回應：

```json
{
  "results": [
    {
      "score": 0.123,
      "slug": "gifgrep",
      "displayName": "GifGrep",
      "summary": "…",
      "version": "1.2.3",
      "updatedAt": 1730000000000,
      "ownerHandle": "openclaw",
      "owner": {
        "handle": "openclaw",
        "displayName": "OpenClaw",
        "image": "https://example.com/avatar.png"
      }
    }
  ]
}
```

備註：

- 結果會按相關性排序傳回（嵌入相似度 + 完全相符的 slug／名稱詞元加權 + 小幅熱門程度先驗加權）。
- 相關性的權重高於熱門程度。精確的 slug 或顯示名稱詞元相符結果，可能勝過互動程度高得多但相符度較低的結果。
- ASCII 文字會按單字和標點符號邊界進行詞元化。例如，`personal-map` 包含獨立的 `map` 詞元，而 `amap-jsapi-skill` 包含 `amap`、`jsapi` 和 `skill`；因此搜尋 `map` 時，`personal-map` 的詞彙相符度會高於 `amap-jsapi-skill`。
- 熱門程度會以對數縮放並設有上限。當查詢文字的相符度較低時，高互動程度的 Skills 排名可能較低。
- 視呼叫者篩選條件和目前審核狀態而定，可疑或隱藏的審核狀態可能會使 Skills 從公開搜尋中移除。

發佈者可探索性指引：

- 將使用者實際會搜尋的詞彙放入顯示名稱、摘要和標籤中。只有在獨立的 slug 詞元也是你想保留的穩定識別名稱時，才使用該詞元。
- 除非新 slug 是更適合長期使用的標準名稱，否則不要只為了迎合某個查詢而重新命名 slug。舊 slug 會成為重新導向別名，但標準 URL、顯示的 slug 和未來的搜尋摘要都會使用新 slug。
- 重新命名別名會保留舊 URL 和透過登錄檔解析的安裝項目的解析能力，但重新命名完成索引後，搜尋排名會以標準 Skills 中繼資料為準。現有統計資料會保留在該 Skills 中。
- 如果 Skills 意外不可見，請先在登入狀態下使用 `clawhub inspect @owner/slug` 檢查審核狀態，再變更與排名相關的中繼資料。

### `GET /api/v1/skills`

查詢參數：

- `limit`（選用）：整數（1–200）
- `cursor`（選用）：任何非 `trending` 排序的分頁游標
- `sort`（選用）：`updated`（預設值）、`recommended`（別名：`default`）、`createdAt`（別名：`newest`）、`downloads`、`stars`（別名：`rating`），舊版安裝別名 `installsCurrent`/`installs`/`installsAllTime` 會對應至 `downloads`、`trending`
- `nonSuspiciousOnly`（選用）：使用 `true` 隱藏可疑（`flagged.suspicious`）Skills
- `nonSuspicious`（選用）：`nonSuspiciousOnly` 的舊版別名

無效的 `sort` 值會傳回 `400`。

備註：

- `recommended` 使用互動程度和時效性訊號。
- `trending` 依過去 7 天內的安裝次數排名（根據遙測資料）。
- `createdAt` 對新 Skills 的爬取作業而言是穩定的；重新發佈現有 Skills 時，`updated` 會變更。
- 當 `nonSuspiciousOnly=true` 時，基於游標的排序可能會在一頁中傳回少於 `limit` 個項目，因為可疑 Skills 會在擷取頁面後遭到篩除。
- 當 `nextCursor` 存在時，請使用它繼續分頁。頁面較短本身並不表示已到達結果末尾。

回應：

```json
{
  "items": [
    {
      "slug": "gifgrep",
      "displayName": "GifGrep",
      "summary": "…",
      "topics": ["Productivity"],
      "tags": { "latest": "1.2.3" },
      "stats": {},
      "createdAt": 0,
      "updatedAt": 0,
      "latestVersion": { "version": "1.2.3", "createdAt": 0, "changelog": "…" },
      "metadata": { "os": ["macos"], "systems": ["aarch64-darwin"] }
    }
  ],
  "nextCursor": null
}
```

### `GET /api/v1/skills/{slug}`

回應：

```json
{
  "skill": {
    "slug": "gifgrep",
    "displayName": "GifGrep",
    "summary": "…",
    "topics": ["Productivity"],
    "tags": { "latest": "1.2.3" },
    "stats": {},
    "createdAt": 0,
    "updatedAt": 0
  },
  "latestVersion": { "version": "1.2.3", "createdAt": 0, "changelog": "…" },
  "metadata": { "os": ["macos"], "systems": ["aarch64-darwin"] },
  "owner": { "handle": "steipete", "displayName": "Peter", "image": null },
  "moderation": {
    "isSuspicious": false,
    "isMalwareBlocked": false,
    "verdict": "clean",
    "reasonCodes": [],
    "summary": null,
    "engineVersion": "v2.0.0",
    "updatedAt": 0
  }
}
```

備註：

- 由擁有者重新命名／合併流程建立的舊 slug 會解析至標準 Skills。
- `metadata.os`：Skills frontmatter 中宣告的作業系統限制（例如 `["macos"]`、`["linux"]`）。若未宣告則為 `null`。
- `metadata.systems`：Nix 系統目標（例如 `["aarch64-darwin", "x86_64-linux"]`）。若未宣告則為 `null`。
- 如果 Skills 沒有平台中繼資料，`metadata` 會是 `null`。
- 只有在 Skills 已遭標記或由其擁有者檢視時，才會包含 `moderation`。

### `GET /api/v1/skills/{slug}/moderation`

傳回結構化審核狀態。

回應：

```json
{
  "moderation": {
    "isSuspicious": true,
    "isMalwareBlocked": false,
    "verdict": "suspicious",
    "reasonCodes": ["suspicious.dynamic_code_execution"],
    "summary": "Detected: suspicious.dynamic_code_execution",
    "engineVersion": "v2.0.0",
    "updatedAt": 0,
    "legacyReason": null,
    "evidence": [
      {
        "code": "suspicious.dynamic_code_execution",
        "severity": "critical",
        "file": "index.ts",
        "line": 3,
        "message": "Dynamic code execution detected.",
        "evidence": ""
      }
    ]
  }
}
```

備註：

- 擁有者和審核人員可以存取隱藏 Skills 的審核詳細資料。
- 公開呼叫者只能取得已遭標記且可見 Skills 的 `200`。
- 公開呼叫者看到的證據會經過遮蔽，只有擁有者／審核人員能看到原始片段。

### `POST /api/v1/skills/{slug}/report`

回報 Skills 以供審核人員審查。回報以 Skills 為單位，可選擇性地連結
至特定版本，並送入 Skills 回報佇列。

驗證：

- 需要 API 權杖。

要求：

```json
{ "reason": "可疑的安裝步驟", "version": "1.2.3" }
```

回應：

```json
{
  "ok": true,
  "reported": true,
  "alreadyReported": false,
  "reportId": "skillReports:...",
  "skillId": "skills:...",
  "reportCount": 1
}
```

### `GET /api/v1/skills/-/reports`

供審核人員／管理員接收 Skills 回報的端點。

查詢參數：

- `status`（選用）：`open`（預設值）、`confirmed`、`dismissed` 或 `all`
- `limit`（選用）：整數（1-200）
- `cursor`（選用）：分頁游標

回應：

```json
{
  "items": [
    {
      "reportId": "skillReports:...",
      "skillId": "skills:...",
      "skillVersionId": "skillVersions:...",
      "slug": "gifgrep",
      "displayName": "GifGrep",
      "version": "1.2.3",
      "reason": "可疑的安裝步驟",
      "status": "open",
      "createdAt": 1730000000000,
      "reporter": {
        "userId": "users:...",
        "handle": "reporter",
        "displayName": "檢舉者"
      },
      "triagedAt": null,
      "triagedBy": null,
      "triageNote": null
    }
  ],
  "nextCursor": null,
  "done": true
}
```

### `POST /api/v1/skills/-/reports/{reportId}/triage`

供版主／管理員解決或重新開啟 Skills 檢舉的端點。

請求：

```json
{ "status": "confirmed", "note": "已審查並隱藏受影響的版本。", "finalAction": "hide" }
```

`note` 是 `confirmed` 和 `dismissed` 的必填項目；將
`status` 設回 `open` 時可省略。若要在同一個可稽核工作流程中隱藏 Skills，請傳遞 `finalAction: "hide"` 與已分流處理的
檢舉。

### `GET /api/v1/skills/{slug}/versions`

查詢參數：

- `limit`（選用）：整數
- `cursor`（選用）：分頁游標

### `GET /api/v1/skills/{slug}/versions/{version}`

傳回版本中繼資料與檔案清單。

- `version.security` 包含正規化的掃描驗證狀態及掃描器詳細資料
  （VirusTotal + LLM，若有）。

### `GET /api/v1/skills/{slug}/scan`

傳回 Skills 版本的安全性掃描驗證詳細資料。

查詢參數：

- `version`（選用）：特定版本字串。
- `tag`（選用）：解析已標記的版本（例如 `latest`）。

注意事項：

- 若未提供 `version` 或 `tag`，則使用最新版本。
- 包含正規化的驗證狀態及各掃描器的詳細資料。
- 只有當掃描器產生明確判定（`clean`、`suspicious` 或 `malicious`）時，`security.hasScanResult` 才是 `true`。
- `moderation` 是衍生自最新版本的目前 Skills 層級內容管理快照。
- 查詢歷史版本時，在將 `moderation` 和 `security` 視為相同版本情境之前，請先檢查 `moderation.matchesRequestedVersion` 和 `moderation.sourceVersion`。

### `POST /api/v1/skills/-/scan`

用於新增 ClawScan 工作且需要驗證的提交端點。

已不再支援本機上傳掃描。使用
`multipart/form-data` 或 `{ "source": { "kind": "upload" } }` 的請求會傳回 `410`。

已發佈掃描使用 JSON：

```json
{
  "source": { "kind": "published", "slug": "gifgrep", "version": "1.2.3" },
  "update": false
}
```

注意事項：

- 掃描請求承載資料及可下載報告會在保留期限過後從掃描請求儲存區中到期。
- 已發佈掃描需要擁有者／發佈者的管理存取權，或平台版主／管理員權限。
- 只有當 `update: true` 且掃描成功完成時，已發佈掃描才會回寫。
- 回應為含有 `{ "ok": true, "scanId": "...", "jobId": "...", "status": "queued", "sourceKind": "published", "update": false, "queue": { "queuedAhead": 0, "queuedAheadIsEstimate": false, "position": 1, "running": 0, "runningIsEstimate": false, "note": "Scans are asynchronous and may take time to complete." } }` 的 `202`。
- 掃描工作以非同步方式執行。手動掃描請求的優先順序高於一般發佈／回填工作，但完成時間仍取決於工作程序的可用性。

### `GET /api/v1/skills/-/scan/{scanId}`

用於輪詢已提交掃描且需要驗證的端點。

- 傳回已排入佇列／執行中／成功／失敗狀態。
- 在排入佇列期間傳回 `queue.queuedAhead` 和 `queue.position`，讓用戶端能顯示此請求之前有多少個優先手動掃描。系統會限制非常大的佇列，並以 `queuedAheadIsEstimate: true` 回報。
- 若有，`report` 會包含 `clawscan`、`skillspector`、`staticAnalysis` 和 `virustotal` 區段。
- 失敗的掃描工作會傳回含有 `lastError` 的 `status: "failed"`。

### `GET /api/v1/skills/-/scan/{scanId}/download`

需要驗證的報告封存端點。

- 需要成功的掃描；尚未進入終止狀態的掃描會傳回 `409`。
- 傳回含有 `manifest.json`、`clawscan.json`、`skillspector.json`、`static-analysis.json`、`virustotal.json` 和 `README.md` 的 ZIP。

### `GET /api/v1/skills/-/scan/download/{name}?version=<version>&kind=skill|plugin`

用於已提交版本且需要驗證的已儲存報告封存端點。

- 需要 Skills 或外掛的擁有者／發佈者管理存取權，或平台版主／管理員權限。
- 傳回所提交確切版本的已儲存掃描結果，包括遭封鎖或隱藏的版本。
- `kind` 預設為 `skill`；外掛／套件掃描請使用 `kind=plugin`。
- 傳回與掃描請求下載相同結構的 ZIP。

### `POST /api/v1/skills/-/scan/batch`

僅限管理員使用的標準批次重新掃描路由。它接受與舊版 `POST /api/v1/skills/-/rescan-batch` 相同結構的承載資料。

### `POST /api/v1/skills/-/scan/batch/status`

僅限管理員使用的標準批次狀態路由。它接受 `{ "jobIds": ["..."] }`，並傳回與舊版 `POST /api/v1/skills/-/rescan-batch/status` 相同的彙總計數器。

### `GET /api/v1/skills/{slug}/verify`

傳回 `clawhub skill verify` 使用的 Skill Card 驗證封套。

查詢參數：

- `version`（選用）：特定版本字串。
- `tag`（選用）：解析已標記的版本（例如 `latest`）。

注意事項：

- 只有當所選版本具有已產生的 Skill Card、未因惡意軟體而遭內容管理封鎖，且 ClawScan 驗證結果為乾淨時，`ok` 才是 `true`。
- Skills 識別資訊、發佈者識別資訊和所選版本中繼資料是頂層封套欄位（`slug`、`displayName`、`publisherHandle`、`version`、`resolvedFrom`、`tag`、`createdAt`），因此 Shell 自動化無須解開巢狀包裝即可讀取。
- `security` 是頂層 ClawScan／安全性判定。自動化應以 `ok`、`decision`、`reasons` 和 `security.status` 作為判斷依據。
- `security.signals` 包含掃描器的輔助證據，例如 `staticScan`、`virusTotal` 和 `skillSpector`。
- 為了維持 v1 回應相容性而保留 `security.signals.dependencyRegistry`，但相依性登錄檔存在性掃描器已停用，此索引鍵一律為 `null`。
- 只有當 ClawHub 在發佈或匯入期間解析並儲存 GitHub 儲存庫／參照／提交／路徑時，`provenance` 才是 `server-resolved-github-import`；否則為 `unavailable`。

### `POST /api/v1/skills/-/security-verdicts`

傳回確切 Skills 版本目前的精簡安全性判定。此
集合端點供已經知道要顯示哪些已安裝
ClawHub Skills 版本的用戶端使用，例如 OpenClaw Control UI。

請求：

```json
{
  "items": [{ "slug": "gifgrep", "version": "1.2.3" }]
}
```

注意事項：

- `items` 必須包含 1-100 組不重複的 `{ slug, version }` 配對。
- 結果以個別項目為單位；缺少一個 Skills 或版本不會導致整個回應失敗。
- 回應僅包含安全性資料，不包含 Skill Card 資料、已產生卡片的狀態、成品檔案清單或詳細的掃描器承載資料。
- `security.signals` 僅包含狀態層級的輔助證據；如需完整的掃描器詳細資料，請使用 `/scan` 或 ClawHub 安全性稽核頁面。
- 為了維持 v1 回應相容性而保留 `security.signals.dependencyRegistry`，但相依性登錄檔存在性掃描器已停用，此索引鍵一律為 `null`。
- 缺少 Skill Card 不會影響此端點的 `ok`、`decision` 或 `reasons`；用戶端需要卡片內容時，應在本機讀取已安裝的 `skill-card.md`。
- 需要單一 Skills 的 Skill Card 驗證封套時，請使用 `/verify`；需要已產生的卡片 Markdown 時，請使用 `/card`；需要詳細掃描器資料時，請使用 `/scan`。

回應：

```json
{
  "schema": "clawhub.skill.security-verdicts.v1",
  "items": [
    {
      "ok": true,
      "decision": "pass",
      "reasons": [],
      "requestedSlug": "gifgrep",
      "slug": "gifgrep",
      "displayName": "GifGrep",
      "publisherHandle": "steipete",
      "publisherDisplayName": "Peter",
      "requestedVersion": "1.2.3",
      "version": "1.2.3",
      "createdAt": 0,
      "checkedAt": 0,
      "skillUrl": "https://clawhub.ai/steipete/skills/gifgrep",
      "securityAuditUrl": "https://clawhub.ai/steipete/skills/gifgrep/security-audit?version=1.2.3",
      "security": {
        "status": "clean",
        "passed": true,
        "signals": {
          "staticScan": { "status": "clean", "reasonCodes": [] },
          "virusTotal": null,
          "skillSpector": null,
          "dependencyRegistry": null
        }
      }
    },
    {
      "ok": false,
      "decision": "fail",
      "reasons": ["version.not_found"],
      "requestedSlug": "missing-version",
      "requestedVersion": "1.0.0",
      "error": { "code": "version_not_found", "message": "找不到版本" },
      "security": null
    }
  ]
}
```

### `GET /api/v1/skills/{slug}/file`

以下載形式傳回確切的已儲存檔案位元組。新增 `preview=1` 可要求有大小限制且經逸出的文字
預覽；只要檔案包含有效的 UTF-8 位元組，無論其副檔名或 MIME
中繼資料為何，都能預覽。

查詢參數：

- `path`（必填）
- `version`（選用）
- `tag`（選用）
- `preview=1`（選用；位元組不是有效的 UTF-8 時，傳回 `text/plain` 或 `415`）

注意事項：

- 預設為最新版本。
- 原始下載限制：10MB。
- 文字預覽限制：200KB。

### `GET /api/v1/packages`

適用於下列項目的統一目錄端點：

- Skills
- 程式碼外掛
- 套件組外掛

查詢參數：

- `limit`（選用）：整數（1–100）
- `cursor`（選用）：分頁游標
- `family`（選用）：`skill`、`code-plugin` 或 `bundle-plugin`
- `channel`（選用）：`official`、`community` 或 `private`
- `isOfficial`（選用）：`true` 或 `false`
- `sort`（選用）：`updated`（預設）、`recommended`、`trending`、`downloads`，舊版別名 `installs`
- `category`（選用）：外掛類別篩選器。僅在
  請求範圍限定為外掛套件（`/api/v1/plugins`、
  `/api/v1/code-plugins`、`/api/v1/bundle-plugins`，或含有
  `family=code-plugin`/`family=bundle-plugin` 的套件端點）時才支援。
  受控類別和舊版 v1 篩選器別名記載於 `GET /api/v1/plugins`。

注意事項：

- `family`、`channel`、`isOfficial`、`featured`、
  `highlightedOnly` 或 `sort` 的無效值會傳回 `400`。未知的查詢參數會被忽略。
- `GET /api/v1/code-plugins` 和 `GET /api/v1/bundle-plugins` 仍是固定系列別名。
- Skills 項目仍以 Skills 登錄檔為基礎，且仍只能透過 `POST /api/v1/skills` 發佈。
- `POST /api/v1/packages` 仍僅適用於程式碼外掛和套件組外掛版本。
- 匿名呼叫者只能看到公開套件頻道。
- 通過驗證的呼叫者可以在清單／搜尋結果中看到其所屬發佈者的私有套件。
- `channel=private` 僅傳回通過驗證的呼叫者可讀取的套件。

### `GET /api/v1/packages/search`

跨 Skills 與外掛套件的統一目錄搜尋。

查詢參數：

- `q`（必填）：查詢字串
- `limit`（選填）：整數（1–100）
- `family`（選填）：`skill`、`code-plugin` 或 `bundle-plugin`
- `channel`（選填）：`official`、`community` 或 `private`
- `isOfficial`（選填）：`true` 或 `false`
- `category`（選填）：外掛類別篩選器。僅在請求範圍限定於
  外掛套件時支援。受控類別和舊版 v1 篩選器別名記載於 `GET /api/v1/plugins`。

注意事項：

- `family`、`channel`、`isOfficial`、`featured` 或
  `highlightedOnly` 的值無效時會傳回 `400`。未知的查詢參數會被忽略。
- 匿名呼叫者只能看到公開套件頻道。
- 已驗證身分的呼叫者可以搜尋其所屬發布者的私人套件。
- `channel=private` 只會傳回已驗證身分的呼叫者可讀取的套件。

### `GET /api/v1/plugins`

僅限外掛的目錄瀏覽，涵蓋程式碼外掛與套件組合外掛套件。

查詢參數：

- `limit`（選填）：整數（1-100）
- `cursor`（選填）：分頁游標
- `isOfficial`（選填）：`true` 或 `false`
- `sort`（選填）：`recommended`（預設）、`trending`、`downloads`、`updated`、舊版別名 `installs`
- `category`（選填）：外掛類別篩選器。目前的值：
  `channels`、`models`、`memory`、`context`、`voice`、`media`、`web`、
  `tools`、`runtime`、`gateway`、`security`、`other`。

讀取端點仍接受舊版 v1 篩選器別名：

- `mcp-tooling`、`data` 和 `automation` 會解析為 `tools`。
- `observability` 和 `deployment` 會解析為 `gateway`。
- `dev-tools` 會解析為 `runtime`。

`trending` 是七天期的安裝／下載排行榜，不使用歷史累計總數。
在統一的 `/api/v1/packages` 端點上，它僅限外掛；Skills 目錄請使用
`/api/v1/skills?sort=trending`。

舊版別名不可作為儲存或由作者宣告的類別值。

### `GET /api/v1/skills/export`

大量匯出最新的公開 Skills，以供離線分析。

驗證：

- 需要 API 權杖。

查詢參數：

- `startDate`（必填）：Skill `updatedAt` 的 Unix 毫秒下限。
- `endDate`（必填）：Skill `updatedAt` 的 Unix 毫秒上限。
- `limit`（選填）：整數（1-250），預設為 `250`。
- `cursor`（選填）：上一個回應中的分頁游標。

回應：

- 本文：ZIP 封存檔。
- 每個匯出的 Skill 都以 `{publisher}/{slug}/` 為根目錄。
- 託管的 Skills 包含最新儲存版本的檔案，並以 `sourceRef: "public-clawhub"`
  列於 `_manifest.json` 中。
- 目前由 GitHub 支援且具有 `clean` 或 `suspicious` 掃描的 Skills，會包含
  以 `sourceRef: "public-github"`、儲存庫、提交、路徑、內容雜湊及封存檔 URL
  列於 `_source_handoff.json` 中的資料。它們不包含由 ClawHub 託管的原始檔案。
- 每個 Skill 都包含 `_export_skill_meta.json`。
- `_manifest.json` 一律包含在 ZIP 根目錄。
- 無法匯出個別 Skills 或檔案時，會包含
  `_errors.json`。

標頭：

- `X-Next-Cursor`
- `X-Has-More`
- `X-Total-Returned`
- `X-Date-Range`
- `X-Export-Errors`

### `GET /api/v1/plugins/export`

大量匯出最新的公開外掛版本，以供離線分析。

驗證：

- 需要 API 權杖。

查詢參數：

- `startDate`（必填）：外掛 `updatedAt` 的 Unix 毫秒下限。
- `endDate`（必填）：外掛 `updatedAt` 的 Unix 毫秒上限。
- `limit`（選填）：整數（1-250），預設為 `250`。
- `cursor`（選填）：上一個回應中的分頁游標。
- `family`（選填）：`code-plugin` 或 `bundle-plugin`。省略表示兩種
  外掛系列。

回應：

- 本文：ZIP 封存檔。
- 每個匯出的外掛都以 `{family}/{packageName}/` 為根目錄。
- 每個匯出的外掛都包含最新版本的已儲存檔案。
- 每個外掛的匯出中繼資料儲存於
  `__clawhub_export/{family}/{packageName}/plugin_meta.json`。
- `_manifest.json` 一律包含在 ZIP 根目錄。
- 無法匯出個別外掛或檔案時，會包含
  `_errors.json`。

標頭：

- `X-Next-Cursor`
- `X-Has-More`
- `X-Total-Returned`
- `X-Date-Range`
- `X-Export-Errors`

### `GET /api/v1/plugins/search`

僅限外掛的搜尋，涵蓋程式碼外掛與套件組合外掛套件。

查詢參數：

- `q`（必填）：查詢字串
- `limit`（選填）：整數（1-100）
- `isOfficial`（選填）：`true` 或 `false`
- `category`（選填）：外掛類別篩選器。目前的值：
  `channels`、`models`、`memory`、`context`、`voice`、`media`、`web`、
  `tools`、`runtime`、`gateway`、`security`、`other`。

注意事項：

- 也接受記載於 `GET /api/v1/plugins` 下的舊版 v1 篩選器別名。
- 類別篩選是由外掛類別摘要資料列支援的實際 API 篩選器，
  並非搜尋查詢重寫。
- 結果會依相關性順序傳回，目前不進行分頁。
- 瀏覽器 UI 中的外掛搜尋排序控制項會重新排列已載入的相關性結果，
  與目前的 `/skills` 瀏覽行為一致。

### `GET /api/v1/packages/{name}`

傳回套件詳細中繼資料。

注意事項：

- 在統一目錄中，Skills 也可以透過此路由解析。
- 除非呼叫者可讀取所屬發布者，否則私人套件會傳回 `404`。

### `DELETE /api/v1/packages/{name}`

對套件及其所有版本執行軟刪除。

注意事項：

- 需要套件擁有者、組織發布者擁有者／管理員、平台仲裁者或平台管理員的 API 權杖。

### `GET /api/v1/packages/{name}/versions`

傳回版本歷程記錄。

查詢參數：

- `limit`（選填）：整數（1–100）
- `cursor`（選填）：分頁游標

注意事項：

- 除非呼叫者可讀取所屬發布者，否則私人套件會傳回 `404`。

### `GET /api/v1/packages/{name}/versions/{version}`

傳回一個套件版本，包括檔案中繼資料、相容性、
驗證、成品中繼資料及掃描資料。

注意事項：

- 對舊式套件封存檔而言，`version.artifact.kind` 是 `legacy-zip`；
  對 ClawPack 支援的版本而言則是 `npm-pack`。
- ClawPack 版本包含與 npm 相容的 `npmIntegrity`、`npmShasum` 及
  `npmTarballName` 欄位。
- `version.sha256hash` 是供舊版用戶端使用的已淘汰相容性中繼資料。它會對
  `/api/v1/packages/{name}/download` 傳回的確切 ZIP 位元組進行雜湊。
  現代用戶端應使用 `version.artifact.sha256`，它可識別
  標準版本成品。
- 存在掃描資料時，會包含 `version.vtAnalysis`、`version.llmAnalysis` 及 `version.staticScan`。
- 除非呼叫者可讀取所屬發布者，否則私人套件會傳回 `404`。

### `GET /api/v1/packages/{name}/versions/{version}/security`

傳回供安裝用戶端使用的確切套件版本安全性與信任摘要。這是 OpenClaw 的公開取用介面，
用於判斷是否可以安裝已解析的版本。

驗證：

- 公開讀取端點。不需要擁有者、發布者、仲裁者或管理員權杖。

回應：

```json
{
  "package": {
    "name": "@openclaw/example-plugin",
    "displayName": "Example Plugin",
    "family": "code-plugin"
  },
  "release": {
    "releaseId": "packageReleases:...",
    "version": "1.2.3",
    "artifactKind": "npm-pack",
    "artifactSha256": "0123456789abcdef...",
    "npmIntegrity": "sha512-...",
    "npmShasum": "0123456789abcdef0123456789abcdef01234567",
    "npmTarballName": "example-plugin-1.2.3.tgz",
    "createdAt": 1730000000000
  },
  "trust": {
    "scanStatus": "malicious",
    "moderationState": "quarantined",
    "blockedFromDownload": true,
    "reasons": ["manual:quarantined", "scan:malicious"],
    "pending": false,
    "stale": false
  }
}
```

回應欄位：

- `package.name`、`package.displayName` 和 `package.family` 用於識別
  已解析的登錄套件。
- `release.releaseId`、`release.version` 和 `release.createdAt` 用於識別
  經過評估的確切版本。
- 當版本成品的 `release.artifactKind`、`release.artifactSha256`、`release.npmIntegrity`、
  `release.npmShasum` 和 `release.npmTarballName` 已知時，會包含這些欄位。
- `trust.scanStatus` 是從掃描器輸入與手動版本仲裁推導出的有效信任狀態。
- `trust.moderationState` 可為 null。沒有手動版本仲裁時，它是 `null`。
- `trust.blockedFromDownload` 是安裝封鎖訊號。當此值為 `true` 時，OpenClaw 和其他
  安裝用戶端應封鎖安裝，而不是根據掃描器或仲裁欄位重新推導封鎖規則。
- `trust.reasons` 是面向使用者及稽核用的說明清單。原因代碼是穩定且精簡的字串，
  例如 `manual:quarantined`、`scan:malicious` 和 `package:malicious`。
- `trust.pending` 表示一或多個信任輸入仍在等待完成。
- `trust.stale` 表示信任摘要是根據過時的輸入計算而得，
  在做出高信賴度的允許決策前，應視為需要重新整理。

注意事項：

- 此端點精確對應版本。用戶端應在解析預定安裝的套件版本後呼叫它，
  而不應只在讀取最新套件中繼資料後呼叫。
- 除非呼叫者可讀取所屬發布者，否則私人套件會傳回 `404`。
- 此端點刻意比擁有者／仲裁者仲裁端點更精簡。它會公開安裝決策和公開說明，
  但不公開檢舉者身分、檢舉本文、私人證據或內部審查時程。

### `GET /api/v1/packages/{name}/versions/{version}/artifact`

傳回套件版本的明確成品解析器中繼資料。

注意事項：

- 舊版套件版本會傳回 `legacy-zip` 成品及舊版 ZIP
  `downloadUrl`。
- ClawPack 版本會傳回 `npm-pack` 成品、npm 完整性欄位、`tarballUrl`
  及舊版 ZIP 相容性 URL。
- 這是 OpenClaw 的解析器介面；它可避免從共用 URL 猜測封存檔格式。

### `GET /api/v1/packages/{name}/versions/{version}/artifact/download`

透過明確的解析器路徑下載版本成品。

注意事項：

- ClawPack 版本會串流傳輸確切上傳的 npm-pack `.tgz` 位元組。
- 舊版 ZIP 版本會重新導向至 `/api/v1/packages/{name}/download?version=`。
- 使用下載速率配額。

### `GET /api/v1/packages/{name}/readiness`

傳回計算得出的就緒狀態，供 OpenClaw 未來使用。

就緒狀態檢查涵蓋：

- 官方頻道狀態
- 最新版本可用性
- ClawPack npm-pack 成品可用性
- 成品摘要
- 來源儲存庫與提交來源證明
- OpenClaw 相容性中繼資料
- 主機目標
- 掃描狀態

回應：

```json
{
  "package": {
    "name": "@openclaw/example-plugin",
    "displayName": "範例外掛",
    "family": "code-plugin",
    "isOfficial": true,
    "latestVersion": "1.2.3"
  },
  "ready": false,
  "checks": [
    {
      "id": "clawpack",
      "label": "ClawPack 成品",
      "status": "fail",
      "message": "最新版本僅提供舊版 ZIP。"
    }
  ],
  "blockers": ["clawpack"]
}
```

### `GET /api/v1/packages/migrations`

用於列出官方 OpenClaw 外掛遷移資料列的版主端點。

驗證：

- 需要版主或管理員使用者的 API 權杖。

查詢參數：

- `phase`（選填）：`planned`、`published`、`clawpack-ready`、
  `legacy-zip-only`、`metadata-ready`、`blocked`、`ready-for-openclaw`，或
  `all`（預設）。
- `limit`（選填）：整數（1-100）
- `cursor`（選填）：分頁游標

回應：

```json
{
  "items": [
    {
      "migrationId": "officialPluginMigrations:...",
      "bundledPluginId": "core.search",
      "packageName": "@openclaw/search-plugin",
      "packageId": "packages:...",
      "owner": "platform",
      "sourceRepo": "openclaw/openclaw",
      "sourcePath": "plugins/search",
      "sourceCommit": "abc123",
      "phase": "blocked",
      "blockers": ["缺少 ClawPack"],
      "hostTargetsComplete": true,
      "scanClean": false,
      "moderationApproved": false,
      "runtimeBundlesReady": false,
      "notes": null,
      "createdAt": 1760000000000,
      "updatedAt": 1760000000000
    }
  ],
  "nextCursor": null,
  "done": true
}
```

### `POST /api/v1/packages/migrations`

用於建立或更新官方外掛遷移資料列的管理員端點。

驗證：

- 需要管理員使用者的 API 權杖。

要求主體：

```json
{
  "bundledPluginId": "core.search",
  "packageName": "@openclaw/search-plugin",
  "owner": "platform",
  "sourceRepo": "openclaw/openclaw",
  "sourcePath": "plugins/search",
  "sourceCommit": "abc123",
  "phase": "blocked",
  "blockers": ["缺少 ClawPack"],
  "hostTargetsComplete": true,
  "scanClean": false,
  "moderationApproved": false,
  "runtimeBundlesReady": false,
  "notes": "正在等待發布者上傳"
}
```

注意事項：

- `bundledPluginId` 會正規化為小寫，並作為穩定的 upsert 鍵。
- `packageName` 會依 npm 名稱正規化；對於規劃中的
  遷移，套件可以不存在。
- 這只會追蹤遷移就緒狀態，不會修改 OpenClaw 或產生
  ClawPack。

### `GET /api/v1/packages/moderation/queue`

用於套件版本審查佇列的版主／管理員端點。

驗證：

- 需要版主或管理員使用者的 API 權杖。

查詢參數：

- `status`（選填）：`open`（預設）、`blocked`、`manual`，或 `all`
- `limit`（選填）：整數（1-100）
- `cursor`（選填）：分頁游標

狀態含義：

- `open`：可疑、惡意、待處理、已隔離、已撤銷或遭檢舉的版本。
- `blocked`：已隔離、已撤銷或惡意版本。
- `manual`：任何具有手動審核覆寫設定的版本。
- `all`：任何具有手動覆寫設定、非無異常掃描狀態或套件檢舉的版本。

回應：

```json
{
  "items": [
    {
      "packageId": "packages:...",
      "releaseId": "packageReleases:...",
      "name": "@openclaw/example-plugin",
      "displayName": "範例外掛",
      "family": "code-plugin",
      "channel": "community",
      "isOfficial": false,
      "version": "1.2.3",
      "createdAt": 1730000000000,
      "artifactKind": "npm-pack",
      "scanStatus": "malicious",
      "moderationState": "quarantined",
      "moderationReason": "手動審查",
      "sourceRepo": "openclaw/example-plugin",
      "sourceCommit": "abc123",
      "reportCount": 2,
      "lastReportedAt": 1730000001000,
      "reasons": ["manual:quarantined", "scan:malicious", "reports:2"]
    }
  ],
  "nextCursor": null,
  "done": true
}
```

### `POST /api/v1/packages/{name}/report`

檢舉套件以供版主審查。檢舉以套件為單位，並可選擇性地
連結至特定版本。檢舉會進入審核佇列，但本身不會自動隱藏套件或
封鎖下載；版主應使用版本審核來核准、隔離或撤銷成品。

驗證：

- 需要 API 權杖。

要求：

```json
{ "reason": "可疑的原生二進位檔", "version": "1.2.3" }
```

回應：

```json
{
  "ok": true,
  "reported": true,
  "alreadyReported": false,
  "packageId": "packages:...",
  "releaseId": "packageReleases:...",
  "reportCount": 1
}
```

### `GET /api/v1/packages/reports`

用於接收套件檢舉的版主／管理員端點。

驗證：

- 需要版主或管理員使用者的 API 權杖。

查詢參數：

- `status`（選填）：`open`（預設）、`confirmed`、`dismissed`，或 `all`
- `limit`（選填）：整數（1-100）
- `cursor`（選填）：分頁游標

回應：

```json
{
  "items": [
    {
      "reportId": "packageReports:...",
      "packageId": "packages:...",
      "releaseId": "packageReleases:...",
      "name": "@openclaw/example-plugin",
      "displayName": "範例外掛",
      "family": "code-plugin",
      "version": "1.2.3",
      "reason": "可疑的原生二進位檔",
      "status": "open",
      "createdAt": 1730000000000,
      "reporter": {
        "userId": "users:...",
        "handle": "reporter",
        "displayName": "檢舉者"
      },
      "triagedAt": null,
      "triagedBy": null,
      "triageNote": null
    }
  ],
  "nextCursor": null,
  "done": true
}
```

### `GET /api/v1/packages/{name}/moderation`

用於查看套件審核狀態的擁有者／版主端點。

驗證：

- 需要套件擁有者、發布者成員、版主或
  管理員使用者的 API 權杖。

回應：

```json
{
  "package": {
    "packageId": "packages:...",
    "name": "@openclaw/example-plugin",
    "displayName": "範例外掛",
    "family": "code-plugin",
    "channel": "community",
    "isOfficial": false,
    "reportCount": 2,
    "lastReportedAt": 1730000001000,
    "scanStatus": "malicious"
  },
  "latestRelease": {
    "releaseId": "packageReleases:...",
    "version": "1.2.3",
    "artifactKind": "npm-pack",
    "scanStatus": "malicious",
    "moderationState": "quarantined",
    "moderationReason": "手動審查",
    "blockedFromDownload": true,
    "reasons": ["manual:quarantined", "scan:malicious", "reports:2"],
    "createdAt": 1730000000000
  }
}
```

### `POST /api/v1/packages/reports/{reportId}/triage`

用於解決或重新開啟套件檢舉的版主／管理員端點。

要求：

```json
{
  "status": "confirmed",
  "note": "已審查並隔離受影響的版本。",
  "finalAction": "quarantine"
}
```

`note` 是 `confirmed` 和 `dismissed` 的必要欄位；將
`status` 設回 `open` 時可以省略。對已確認的檢舉傳入
`finalAction: "quarantine"` 或 `finalAction: "revoke"`，即可在同一個可稽核的工作流程中
套用版本審核。

回應：

```json
{
  "ok": true,
  "reportId": "packageReports:...",
  "packageId": "packages:...",
  "status": "confirmed",
  "reportCount": 0
}
```

### `POST /api/v1/packages/{name}/versions/{version}/moderation`

用於套件版本審查的版主／管理員端點。

要求：

```json
{ "state": "quarantined", "reason": "可疑的原生有效負載。" }
```

支援的狀態：

- `approved`：已經過手動審查並允許。
- `quarantined`：在後續處理前維持封鎖。
- `revoked`：先前受信任的版本後來遭到封鎖。

已隔離和已撤銷的版本會從成品下載路由傳回 `403`。
每次變更都會寫入一筆稽核記錄。

### `GET /api/v1/packages/{name}/file`

以下載方式傳回確切儲存的套件檔案位元組。加入 `preview=1`，可要求與 Skill 檔案相同且有大小限制的
UTF-8 文字預覽。

查詢參數：

- `path`（必填）
- `version`（選填）
- `tag`（選填）
- `preview=1`（選填；位元組不是有效的 UTF-8 時，傳回 `text/plain` 或 `415`）

注意事項：

- 預設為最新版本。
- 使用讀取速率配額，而不是下載速率配額。
- 原始下載限制：10MB。
- 文字預覽限制：200KB；不透明檔案只會對預覽要求傳回 `415`。
- 待處理的 VirusTotal 掃描不會封鎖讀取；惡意版本仍可能在其他位置遭到扣留。
- 除非呼叫者可以讀取所屬發布者，否則私人套件會傳回 `404`。

### `GET /api/v1/packages/{name}/download`

下載套件版本的舊版確定性 ZIP 封存檔。

查詢參數：

- `version`（選填）
- `tag`（選填）

注意事項：

- 預設為最新版本。
- Skills 會重新導向至 `GET /api/v1/download`。
- 外掛／套件封存檔是以 `package/` 為根目錄的 zip 檔案，讓舊版 OpenClaw
  用戶端能繼續運作。
- 此路由一律只提供 ZIP，不會串流傳輸 ClawPack `.tgz` 檔案。
- 回應會包含 `ETag`、`Digest`、`X-ClawHub-Artifact-Type` 和
  `X-ClawHub-Artifact-Sha256` 標頭，以供解析器進行完整性檢查。
- 不會將僅供登錄檔使用的中繼資料注入下載的封存檔。
- 待處理的 VirusTotal 掃描不會封鎖下載；惡意版本會傳回 `403`。
- 除非呼叫者是擁有者，否則私人套件會傳回 `404`。

### `GET /api/npm/{package}`

傳回由 ClawPack 支援之套件版本的 npm 相容 packument。

注意事項：

- 只會列出已上傳 ClawPack npm-pack tarball 的版本。
- 只提供舊版 ZIP 的版本會刻意省略。
- `dist.tarball`、`dist.integrity` 和 `dist.shasum` 使用 npm 相容
  欄位，讓使用者可以選擇將 npm 指向此鏡像。
- 具命名空間的套件 packument 同時支援 `/api/npm/@scope/name` 和 npm 的
  編碼 `/api/npm/@scope%2Fname` 要求路徑。

### `GET /api/npm/{package}/-/{tarball}.tgz`

為 npm 鏡像用戶端串流傳輸確切上傳的 ClawPack tarball 位元組。

注意事項：

- 使用下載速率配額。
- 下載標頭包含 ClawHub SHA-256，以及 npm 完整性／shasum 中繼資料。
- 審核和私人套件存取檢查仍然適用。

### `GET /api/v1/resolve`

供命令列介面將本機指紋對應至已知版本。

查詢參數：

- `slug`（必填）
- `hash`（必填）：套件組合指紋的 64 字元十六進位 sha256

回應：

```json
{ "slug": "gifgrep", "match": { "version": "1.2.2" }, "latestVersion": { "version": "1.2.3" } }
```

### `GET /api/v1/download`

下載託管的 skill 版本 ZIP；若目前由 GitHub 支援的 skill 通過 `clean` 或 `suspicious` 掃描且沒有託管版本，則傳回 GitHub 原始碼移交資訊。

查詢參數：

- `slug`（必填）
- `version`（選填）：semver 字串
- `tag`（選填）：標籤名稱（例如 `latest`）

注意事項：

- 若未提供 `version` 或 `tag`，則使用最新版本。
- 已軟刪除的版本會傳回 `410`。
- 由 GitHub 支援的 skill 移交不會代理或鏡像位元組。JSON 回應
  包含 `sourceRef: "public-github"`、`repo`、`commit`、`path`、`contentHash`
  和 `archiveUrl`；掃描／目前狀態是閘門條件，不會納入成功
  承載資料的中繼資料。
- 下載統計以每個 UTC 日的不重複身分計算（API 權杖有效時為 `userId`，否則為 IP）。

## 驗證端點（Bearer 權杖）

所有端點都需要：

```
Authorization: Bearer clh_...
```

### `GET /api/v1/whoami`

驗證權杖並傳回使用者代號。

### `POST /api/v1/skills`

發布新版本。

- 建議：使用包含 `payload` JSON 與 `files[]` blob 的 `multipart/form-data`。
- 也接受包含 `files`（以 storageId 為基礎）的 JSON 主體。
- 選填承載欄位：`ownerHandle`。若存在，API 會在伺服器端解析該
  發布者，並要求執行者具有發布者存取權。
- 選填承載欄位：`migrateOwner`。當 `true` 搭配 `ownerHandle` 時，
  若執行者同時是目前及目標發布者的管理員／擁有者，現有 skill 可移轉給該擁有者。
  若未選擇加入此功能，擁有者變更將遭拒絕。

### `POST /api/v1/packages`

發布程式碼外掛或套件外掛版本。

- 需要 Bearer 權杖驗證。
- 需要 `multipart/form-data`。
- 允許的表單欄位為 `payload`、重複的 `files` blob，或一個 `clawpack`
  tarball 參照。`clawpack` 可以是 `.tgz` blob，或上傳 URL 流程傳回的儲存空間 ID。
  使用暫存儲存空間 ID 發布時，也必須包含該上傳 URL 所傳回的
  `clawpackUploadTicket`。
- 請使用 `files` 或 `clawpack` 其中之一，絕不可在同一個請求中同時使用兩者。
- JSON 主體以及呼叫端提供的 `payload.files`／`payload.artifact`
  中繼資料會遭拒絕。
- 直接 multipart 發布請求的上限為 18MB。ClawPack tarball 可
  使用上傳 URL 流程，最高可達 120MB tarball 上限。
- 選填承載欄位：`ownerHandle`。若存在，只有管理員能代表該擁有者發布。

驗證重點：

- `family` 必須是 `code-plugin` 或 `bundle-plugin`。
- 外掛套件需要 `openclaw.plugin.json`。ClawPack `.tgz` 上傳內容必須
  在 `package/openclaw.plugin.json` 包含它。
- 程式碼外掛需要 `package.json`、原始碼儲存庫中繼資料、原始碼提交
  中繼資料、設定結構描述中繼資料、`openclaw.compat.pluginApi`，以及
  `openclaw.build.openclawVersion`。
- `openclaw.hostTargets` 和 `openclaw.environment` 是選填中繼資料。
- 只有 `openclaw` 組織發布者與目前 `openclaw` 組織成員的
  個人發布者可發布至 `official` 頻道。
- 代表他人發布時，仍會依目標擁有者帳號驗證官方頻道資格。

### `DELETE /api/v1/skills/{slug}`／`POST /api/v1/skills/{slug}/undelete`

軟刪除／還原 skill（擁有者、版主或管理員）。

選填 JSON 主體：

```json
{ "reason": "暫由審核保留，等待法律審查。" }
```

若存在，`reason` 會儲存為 skill 審核備註，並複製到稽核記錄。
由擁有者發起的軟刪除會保留 slug 30 天，之後其他發布者即可取得該 slug。
適用此到期規則時，刪除回應會包含 `slugReservedUntil`。
版主／管理員隱藏及安全性移除不會以此方式到期。

刪除回應：

```json
{ "ok": true, "slugReservedUntil": 1730000000000 }
```

狀態碼：

- `200`：成功
- `401`：未授權
- `403`：禁止存取
- `404`：找不到 skill／使用者
- `500`：內部伺服器錯誤

### `POST /api/v1/users/publisher`

僅限管理員。確保某個代號具有組織發布者。若該代號仍指向
舊版共用使用者／個人發布者，此端點會先將其移轉為組織發布者。
對於新建立的組織，請提供 `memberHandle`；執行操作的管理員不會被新增為成員。
`memberRole` 預設為 `owner`。

- 主體：`{ "handle": "openclaw", "displayName": "OpenClaw", "memberHandle": "alice", "memberRole": "owner", "trusted": true }`
- 回應：`{ "ok": true, "publisherId": "...", "handle": "openclaw", "created": true, "migrated": false, "trusted": true, "member": { "userId": "...", "handle": "alice", "role": "owner" } }`

### `POST /api/v1/publishers`

經驗證的自助式組織發布者建立功能。建立新的組織發布者，並將
呼叫者新增為擁有者。此端點不會移轉現有的使用者／個人代號，也不會
將發布者標記為受信任／官方。

- 主體：`{ "handle": "opik", "displayName": "Opik" }`
- 回應：`{ "ok": true, "publisherId": "...", "handle": "opik", "created": true, "trusted": false }`
- 若代號已由發布者、使用者或個人發布者使用，則傳回 `409`。

### `POST /api/v1/users/reserve`

僅限管理員。為合法擁有者保留根 slug 和套件名稱，而不發布
版本。套件名稱會成為沒有版本資料列的私人預留位置套件，使同一
擁有者之後可將真正的程式碼外掛或套件外掛版本發布至該名稱。

- 主體：`{ "handle": "openclaw", "slugs": ["diffs"], "packageNames": ["@openclaw/diffs"], "reason": "reserved for official OpenClaw plugin" }`
- 回應：`{ "ok": true, "succeeded": 2, "failed": 0, "results": [{ "kind": "slug", "name": "diffs", "ok": true, "action": "reserved" }] }`

### `POST /api/v1/users/publisher-recovery`

僅限管理員。為經驗證的替代 GitHub OAuth 主體復原個人發布者，
而不編輯 Convex Auth 帳號資料列。請求必須同時指定兩個不可變的 GitHub
提供者帳號 ID；可變的代號僅作為面向操作人員的防護條件。

端點預設為試執行。套用復原需要 `dryRun: false` 和
`confirmIdentityVerified: true`，且工作人員須先獨立驗證兩個
GitHub 主體之間的連續性。若目的地使用者目前的個人發布者擁有 skill、
套件或 GitHub skill 來源，復原將採取封閉式失敗。
復原也會移轉已復原發布者之 skill、skill slug 別名、套件、
套件檢查器警告及衍生搜尋摘要資料列中的舊版 `ownerUserId` 欄位，
使直接擁有者路徑與新的發布者權限保持一致。已復原代號的有效受保護代號
保留項目也會重新指派給替代使用者，以免後續的個人檔案同步恢復前使用者
相互衝突的權限。每個主要資料表在每次套用交易中上限為
100 筆資料列；較大型的復原必須先使用可續傳的擁有者移轉。
GitHub skill 來源的範圍以發布者為單位，會回報為已檢查，而不會重寫。

- 主體：`{ "handle": "gingiris", "nextUserHandle": "gingiris-1031", "previousGitHubProviderAccountId": "123", "nextGitHubProviderAccountId": "456", "reason": "Verified account continuity for issue #2555", "confirmIdentityVerified": true, "dryRun": false }`
- 回應：`{ "ok": true, "dryRun": false, "recovered": true, "publisherId": "...", "handle": "gingiris", "previousUser": { "userId": "...", "handle": "gingiris", "nextHandle": "gingiris-recovered", "githubProviderAccountId": "123", "authAccountCount": 1 }, "nextUser": { "userId": "...", "handle": "gingiris-1031", "nextHandle": "gingiris", "githubProviderAccountId": "456", "authAccountCount": 1 }, "retiredPersonalPublisher": null, "resourceOwnerMigration": { "limitPerTable": 100, "skills": 1, "skillSlugAliases": 1, "packages": 0, "packageInspectorWarnings": 0, "githubSourcesChecked": 1, "handleReservations": 1 }, "identityVerified": true, "reason": "Verified account continuity for issue #2555" }`

### 擁有者 slug 管理端點

- `POST /api/v1/skills/{slug}/rename`
  - 主體：`{ "newSlug": "new-canonical-slug" }`
  - 回應：`{ "ok": true, "slug": "new-canonical-slug", "previousSlug": "old-slug" }`
- `POST /api/v1/skills/{slug}/merge`
  - 主體：`{ "targetSlug": "canonical-target-slug" }`
  - 回應：`{ "ok": true, "sourceSlug": "old-slug", "targetSlug": "canonical-target-slug" }`

注意事項：

- 兩個端點都需要 API 權杖驗證，且僅適用於 skill 擁有者。
- `rename` 會將先前的 slug 保留為重新導向別名。
- `merge` 會隱藏來源清單，並將來源 slug 重新導向至目標清單。

### 擁有權移轉端點

- `POST /api/v1/skills/{slug}/transfer`
  - 主體：`{ "toUserHandle": "target_handle", "message": "optional" }`
  - 回應：`{ "ok": true, "transferId": "skillOwnershipTransfers:...", "toUserHandle": "target_handle", "expiresAt": 1730000000000 }`
- `POST /api/v1/skills/{slug}/transfer/accept`
- `POST /api/v1/skills/{slug}/transfer/reject`
- `POST /api/v1/skills/{slug}/transfer/cancel`
  - 回應（接受／拒絕／取消）：`{ "ok": true, "skillSlug": "demo-skill?" }`
- `GET /api/v1/transfers/incoming`
- `GET /api/v1/transfers/outgoing`
  - 回應格式：`{ "transfers": [{ "_id": "...", "skill": { "slug": "demo", "displayName": "Demo" }, "fromUser"|"toUser": { "handle": "..." }, "message": "...", "requestedAt": 0, "expiresAt": 0 }] }`

### `POST /api/v1/users/ban`

封鎖使用者並永久刪除其擁有的 skill（僅限版主／管理員）。

主體：

```json
{ "handle": "user_handle", "reason": "選填封鎖原因" }
```

或

```json
{ "userId": "users_...", "reason": "選填封鎖原因" }
```

回應：

```json
{ "ok": true, "alreadyBanned": false, "deletedSkills": 3 }
```

### `POST /api/v1/users/unban`

解除封鎖使用者並還原符合資格的 skill（僅限管理員）。

主體：

```json
{ "handle": "user_handle", "reason": "選填解除封鎖原因" }
```

或

```json
{ "userId": "users_...", "reason": "選填解除封鎖原因" }
```

回應：

```json
{ "ok": true, "alreadyUnbanned": false, "restoredSkills": 3 }
```

### `POST /api/v1/users/reclassify-ban`

變更現有封鎖的已儲存原因，但不解除封鎖或還原
內容（僅限管理員）。除非 `dryRun` 為 `false`，否則預設為試執行。

主體：

```json
{ "handle": "user_handle", "reason": "大量發布垃圾內容", "dryRun": true }
```

或

```json
{ "userId": "users_...", "reason": "大量發布垃圾內容", "dryRun": false }
```

回應：

```json
{
  "ok": true,
  "dryRun": false,
  "userId": "users_...",
  "handle": "user_handle",
  "previousReason": "惡意軟體自動封鎖",
  "nextReason": "大量發布垃圾內容",
  "changed": true
}
```

### `POST /api/v1/users/role`

變更使用者角色（僅限管理員）。

主體：

```json
{ "handle": "user_handle", "role": "moderator" }
```

或

```json
{ "userId": "users_...", "role": "admin" }
```

回應：

```json
{ "ok": true, "role": "moderator" }
```

### `GET /api/v1/users`

列出或搜尋使用者（僅限管理員）。

查詢參數：

- `q`（選填）：搜尋查詢
- `query`（選填）：`q` 的別名
- `limit`（選填）：結果數上限（預設 20，最高 200）

回應：

```json
{
  "items": [
    {
      "userId": "users_...",
      "handle": "user_handle",
      "displayName": "使用者",
      "name": "使用者",
      "role": "moderator"
    }
  ],
  "total": 1
}
```

### `POST /api/v1/stars/{slug}`／`DELETE /api/v1/stars/{slug}`

新增／移除書籤。為了相容性，舊版 `stars` 路由與回應欄位名稱仍然
保留。兩個端點都具冪等性。

回應：

```json
{ "ok": true, "starred": true, "alreadyStarred": false }
```

```json
{ "ok": true, "unstarred": true, "alreadyUnstarred": false }
```

## 舊版命令列介面端點（已棄用）

仍支援較舊的命令列介面版本：

- `GET /api/cli/whoami`
- `POST /api/cli/upload-url`
- `POST /api/cli/publish`
- `POST /api/cli/telemetry/install`
- `POST /api/cli/skill/delete`
- `POST /api/cli/skill/undelete`

移除計畫請參閱 `DEPRECATIONS.md`。

`POST /api/cli/upload-url` 會傳回 `uploadUrl` 和 `uploadTicket`。發布
暫存 ClawPack tarball 的套件時，必須將產生的儲存空間 ID 作為
`clawpack` 傳送，並將傳回的票證作為 `clawpackUploadTicket` 傳送。

## 登錄檔探索（`/.well-known/clawhub.json`）

命令列介面可從網站探索登錄檔／驗證設定：

- `/.well-known/clawhub.json`（JSON，建議）
- `/.well-known/clawdhub.json`（舊版）

結構描述：

```json
{ "apiBase": "https://clawhub.ai", "authBase": "https://clawhub.ai", "minCliVersion": "0.0.5" }
```

若自行託管，請提供此檔案（或明確設定 `CLAWHUB_REGISTRY`；舊版為 `CLAWDHUB_REGISTRY`）。
