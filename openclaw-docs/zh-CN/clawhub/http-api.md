---
read_when:
    - 添加/更改端点
    - 调试 CLI ↔ 注册表请求
summary: HTTP API 参考（公共端点 + CLI 端点 + 身份验证）。
x-i18n:
    generated_at: "2026-07-26T05:40:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5b180bbd56d20a3d88c1fe74ccab0fd0ecbe0e8c9624cd1afd2070a2ca1f7fb3
    source_path: clawhub/http-api.md
    workflow: 16
---

# HTTP API

基础 URL：`https://clawhub.ai`（默认）。

所有 v1 路径均位于 `/api/v1/...` 下。
为保持兼容性，旧版 `/api/...` 和 `/api/cli/...` 仍然保留（参见 `DEPRECATIONS.md`）。
OpenAPI：`/api/v1/openapi.json`。

## 公共目录复用

第三方目录可以使用公共读取端点列出或搜索 ClawHub Skills。请缓存结果、遵守 `429`/`Retry-After`、将用户链接回规范的 ClawHub 列表（`https://clawhub.ai/<owner>/skills/<slug>`），并避免暗示 ClawHub 为第三方网站背书。请勿尝试在公共 API 范围之外镜像隐藏、私有或被审核拦截的内容。

Web slug 快捷方式可跨注册表族解析，但 API 客户端应使用读取端点返回的规范 URL，而不是自行重建路由优先级。

## 速率限制

执行模型：

- 匿名请求：按 IP 执行限制。
- 已验证身份的请求（有效的 Bearer 令牌）：按用户存储桶执行限制。
- 如果令牌缺失或无效，则回退到按 IP 执行限制。
- 当服务器知道原因时，需要身份验证的写入端点不应仅返回 `Unauthorized`。
  对于令牌缺失、令牌无效或已撤销，以及账户已删除、被封禁或已停用等情况，
  应分别提供可操作的文本，以便 CLI 客户端告知用户具体的受阻原因。

- 读取：每个 IP 每分钟 3000 次，每个密钥每分钟 12000 次
- 写入：每个 IP 每分钟 300 次，每个密钥每分钟 3000 次
- 下载：每个 IP 每分钟 1200 次，每个密钥每分钟 6000 次（下载端点）

响应头：

- 旧版兼容：`X-RateLimit-Limit`、`X-RateLimit-Reset`
- 标准化：`RateLimit-Limit`、`RateLimit-Reset`
- 在 `429` 上：`X-RateLimit-Remaining: 0` 和 `RateLimit-Remaining: 0`
- 在 `429` 上：`Retry-After`

响应头语义：

- `X-RateLimit-Reset`：绝对 Unix 纪元秒数
- `RateLimit-Reset`：距离重置的秒数（延迟）
- `X-RateLimit-Remaining` / `RateLimit-Remaining`：存在时表示准确的剩余配额。
  分片请求成功时会省略此响应头，而不会返回近似的全局值。
- `Retry-After`：在 `429` 上重试前需要等待的秒数（延迟）

`429` 响应示例：

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

超出速率限制
```

客户端指南：

- 如果存在 `Retry-After`，请等待指定秒数后再重试。
- 使用带抖动的退避策略，避免同步重试。
- 如果缺少 `Retry-After`，则回退到 `RateLimit-Reset`（或根据 `X-RateLimit-Reset` 计算）。

IP 来源：

- 仅当部署明确启用可信转发请求头时，才会使用可信客户端 IP 请求头，包括 `cf-connecting-ip`。
- ClawHub 使用可信转发请求头在边缘识别客户端 IP。
- 如果没有可用的可信客户端 IP，匿名请求将使用仅按速率限制类型划分的回退存储桶。
  这些回退存储桶不包含调用方提供的路径、slug、软件包名称、版本、查询字符串或其他工件参数。

## 错误响应

公共 v1 错误响应是包含 `content-type: text/plain; charset=utf-8` 的纯文本。
这包括验证失败（`400`）、公共资源缺失（`404`）、身份验证和
权限失败（`401`/`403`）、速率限制（`429`）以及下载被拦截。客户端
应将响应正文读取为人类可读的字符串。为保持兼容性，未知查询参数会被
忽略，但值无效的已识别查询参数会返回
`400`。

## 公共端点（无需身份验证）

### `GET /api/v1/search`

查询参数：

- `q`（必填）：查询字符串
- `limit`（可选）：整数
- `highlightedOnly`（可选）：使用 `true` 筛选精选 Skills
- `nonSuspiciousOnly`（可选）：使用 `true` 隐藏可疑（`flagged.suspicious`）Skills
- `nonSuspicious`（可选）：`nonSuspiciousOnly` 的旧版别名

响应：

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

说明：

- 结果按相关性顺序返回（嵌入相似度 + 完全匹配 slug/名称词元的加权 + 较小的热度先验）。
- 相关性的权重高于热度。精确匹配 slug 或显示名称词元的结果，排名可以高于互动量高得多但匹配较宽泛的结果。
- ASCII 文本按单词和标点边界进行词元化。例如，`personal-map` 包含独立的 `map` 词元，而 `amap-jsapi-skill` 包含 `amap`、`jsapi` 和 `skill`；因此，搜索 `map` 时，`personal-map` 的词法匹配强于 `amap-jsapi-skill`。
- 热度采用对数缩放并设有上限。当查询文本的匹配较弱时，高互动量的 Skills 也可能排名较低。
- 根据调用方筛选条件和当前审核状态，可疑或隐藏的审核状态可能会使 Skills 从公共搜索中移除。

发布者可发现性指南：

- 将用户实际会搜索的术语放入显示名称、摘要和标签中。仅当独立的 slug 词元也是你希望保留的稳定标识时才使用它。
- 不要仅为迎合某个查询而重命名 slug，除非新 slug 是更适合作为长期规范名称的选择。旧 slug 会成为重定向别名，但规范 URL、显示的 slug 和未来的搜索摘要将使用新 slug。
- 重命名别名会保留旧 URL 以及通过注册表进行解析的安装流程，但在重命名完成索引后，搜索排名将基于规范的 Skills 元数据。现有统计数据仍归属于该 Skills。
- 如果某个 Skills 意外不可见，请先在登录状态下使用 `clawhub inspect @owner/slug` 检查审核状态，然后再更改与排名相关的元数据。

### `GET /api/v1/skills`

查询参数：

- `limit`（可选）：整数（1–200）
- `cursor`（可选）：用于任何非 `trending` 排序方式的分页游标
- `sort`（可选）：`updated`（默认）、`recommended`（别名：`default`）、`createdAt`（别名：`newest`）、`downloads`、`stars`（别名：`rating`）；旧版安装别名 `installsCurrent`/`installs`/`installsAllTime` 映射到 `downloads`、`trending`
- `nonSuspiciousOnly`（可选）：使用 `true` 隐藏可疑（`flagged.suspicious`）Skills
- `nonSuspicious`（可选）：`nonSuspiciousOnly` 的旧版别名

无效的 `sort` 值会返回 `400`。

说明：

- `recommended` 使用互动量和新近度信号。
- `trending` 按最近 7 天内的安装次数排名（基于遥测数据）。
- `createdAt` 对新 Skills 抓取保持稳定；现有 Skills 重新发布时，`updated` 会发生变化。
- 当 `nonSuspiciousOnly=true` 时，基于游标的排序在一页中返回的条目数可能少于 `limit`，因为可疑 Skills 会在获取页面后被筛除。
- 如果存在 `nextCursor`，请使用它继续分页。页面较短本身并不表示结果已结束。

响应：

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

响应：

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

说明：

- 由所有者重命名/合并流程创建的旧 slug 会解析到规范 Skills。
- `metadata.os`：Skills frontmatter 中声明的操作系统限制（例如 `["macos"]`、`["linux"]`）。未声明时为 `null`。
- `metadata.systems`：Nix 系统目标（例如 `["aarch64-darwin", "x86_64-linux"]`）。未声明时为 `null`。
- 如果 Skills 没有平台元数据，`metadata` 为 `null`。
- 仅当 Skills 被标记或所有者正在查看时，才会包含 `moderation`。

### `GET /api/v1/skills/{slug}/moderation`

返回结构化审核状态。

响应：

```json
{
  "moderation": {
    "isSuspicious": true,
    "isMalwareBlocked": false,
    "verdict": "suspicious",
    "reasonCodes": ["suspicious.dynamic_code_execution"],
    "summary": "检测到：suspicious.dynamic_code_execution",
    "engineVersion": "v2.0.0",
    "updatedAt": 0,
    "legacyReason": null,
    "evidence": [
      {
        "code": "suspicious.dynamic_code_execution",
        "severity": "critical",
        "file": "index.ts",
        "line": 3,
        "message": "检测到动态代码执行。",
        "evidence": ""
      }
    ]
  }
}
```

说明：

- 所有者和审核员可以访问隐藏 Skills 的审核详情。
- 公共调用方仅能为已标记且可见的 Skills 获取 `200`。
- 面向公共调用方的证据会经过删减，仅向所有者/审核员提供原始代码片段。

### `POST /api/v1/skills/{slug}/report`

举报 Skills 以供审核员审查。举报以 Skills 为单位，可以选择关联到某个版本，并进入 Skills 举报队列。

身份验证：

- 需要 API 令牌。

请求：

```json
{ "reason": "安装步骤可疑", "version": "1.2.3" }
```

响应：

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

用于接收 Skills 举报的审核员/管理员端点。

查询参数：

- `status`（可选）：`open`（默认）、`confirmed`、`dismissed` 或 `all`
- `limit`（可选）：整数（1-200）
- `cursor`（可选）：分页游标

响应：

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
      "reason": "可疑的安装步骤",
      "status": "open",
      "createdAt": 1730000000000,
      "reporter": {
        "userId": "users:...",
        "handle": "reporter",
        "displayName": "报告者"
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

用于解决或重新打开技能报告的版主/管理员端点。

请求：

```json
{ "status": "confirmed", "note": "已审查并隐藏受影响的版本。", "finalAction": "hide" }
```

`note` 是 `confirmed` 和 `dismissed` 的必填项；将
`status` 重新设置为 `open` 时可以省略。对已分流的
报告传入 `finalAction: "hide"`，即可在同一可审计工作流中隐藏该技能。

### `GET /api/v1/skills/{slug}/versions`

查询参数：

- `limit`（可选）：整数
- `cursor`（可选）：分页游标

### `GET /api/v1/skills/{slug}/versions/{version}`

返回版本元数据和文件列表。

- `version.security` 包含规范化的扫描验证状态和扫描器详细信息
  （VirusTotal + LLM，如可用）。

### `GET /api/v1/skills/{slug}/scan`

返回技能版本的安全扫描验证详细信息。

查询参数：

- `version`（可选）：特定版本字符串。
- `tag`（可选）：解析带标签的版本（例如 `latest`）。

注意：

- 如果既未提供 `version`，也未提供 `tag`，则使用最新版本。
- 包含规范化的验证状态以及各扫描器的详细信息。
- 仅当扫描器给出明确结论（`clean`、`suspicious` 或 `malicious`）时，`security.hasScanResult` 才为 `true`。
- `moderation` 是根据最新版本得出的当前技能级审核快照。
- 查询历史版本时，在将 `moderation` 和 `security` 视为同一版本上下文之前，请检查 `moderation.matchesRequestedVersion` 和 `moderation.sourceVersion`。

### `POST /api/v1/skills/-/scan`

用于提交新 ClawScan 任务的已验证身份端点。

不再支持本地上传扫描。使用
`multipart/form-data` 或 `{ "source": { "kind": "upload" } }` 的请求将返回 `410`。

已发布扫描使用 JSON：

```json
{
  "source": { "kind": "published", "slug": "gifgrep", "version": "1.2.3" },
  "update": false
}
```

注意：

- 超过保留期限后，扫描请求负载和可下载报告将从扫描请求存储中失效。
- 已发布扫描要求拥有者/发布者管理访问权限，或平台版主/管理员权限。
- 仅当 `update: true` 且扫描成功完成时，已发布扫描才会回写。
- 响应为包含 `{ "ok": true, "scanId": "...", "jobId": "...", "status": "queued", "sourceKind": "published", "update": false, "queue": { "queuedAhead": 0, "queuedAheadIsEstimate": false, "position": 1, "running": 0, "runningIsEstimate": false, "note": "Scans are asynchronous and may take time to complete." } }` 的 `202`。
- 扫描任务以异步方式运行。手动扫描请求的优先级高于普通发布/回填工作，但完成时间仍取决于工作进程的可用性。

### `GET /api/v1/skills/-/scan/{scanId}`

用于轮询已提交扫描的已验证身份端点。

- 返回已排队/运行中/成功/失败状态。
- 排队期间返回 `queue.queuedAhead` 和 `queue.position`，以便客户端显示该请求之前有多少个优先手动扫描。超大队列会受到限制，并通过 `queuedAheadIsEstimate: true` 报告。
- 如可用，`report` 包含 `clawscan`、`skillspector`、`staticAnalysis` 和 `virustotal` 部分。
- 失败的扫描任务返回包含 `lastError` 的 `status: "failed"`。

### `GET /api/v1/skills/-/scan/{scanId}/download`

已验证身份的报告归档端点。

- 要求扫描已成功；非终止状态的扫描返回 `409`。
- 返回一个 ZIP，其中包含 `manifest.json`、`clawscan.json`、`skillspector.json`、`static-analysis.json`、`virustotal.json` 和 `README.md`。

### `GET /api/v1/skills/-/scan/download/{name}?version=<version>&kind=skill|plugin`

用于已提交版本的已验证身份存储报告归档端点。

- 要求拥有对技能或插件的拥有者/发布者管理访问权限，或平台版主/管理员权限。
- 返回所提交确切版本的已存储扫描结果，包括被阻止或隐藏的版本。
- `kind` 默认为 `skill`；插件/软件包扫描请使用 `kind=plugin`。
- 返回与扫描请求下载相同结构的 ZIP。

### `POST /api/v1/skills/-/scan/batch`

仅限管理员使用的规范批量重新扫描路由。它接受与旧版 `POST /api/v1/skills/-/rescan-batch` 相同结构的负载。

### `POST /api/v1/skills/-/scan/batch/status`

仅限管理员使用的规范批量状态路由。它接受 `{ "jobIds": ["..."] }`，并返回与旧版 `POST /api/v1/skills/-/rescan-batch/status` 相同的汇总计数器。

### `GET /api/v1/skills/{slug}/verify`

返回 `clawhub skill verify` 使用的技能卡验证信封。

查询参数：

- `version`（可选）：特定版本字符串。
- `tag`（可选）：解析带标签的版本（例如 `latest`）。

注意：

- 仅当所选版本已生成技能卡、未被审核机制因恶意软件而阻止且 ClawScan 验证结果为干净时，`ok` 才为 `true`。
- 技能身份、发布者身份和所选版本元数据是信封的顶层字段（`slug`、`displayName`、`publisherHandle`、`version`、`resolvedFrom`、`tag`、`createdAt`），因此 shell 自动化无需解包嵌套包装器即可读取它们。
- `security` 是顶层 ClawScan/安全结论。自动化应以 `ok`、`decision`、`reasons` 和 `security.status` 为判断依据。
- `security.signals` 包含支持性扫描器证据，例如 `staticScan`、`virusTotal` 和 `skillSpector`。
- `security.signals.dependencyRegistry` 为兼容 v1 响应而保留，但依赖项注册表存在性扫描器已停用，此键始终为 `null`。
- 仅当 ClawHub 在发布或导入期间解析并存储了 GitHub 仓库/ref/提交/路径时，`provenance` 才为 `server-resolved-github-import`；否则为 `unavailable`。

### `POST /api/v1/skills/-/security-verdicts`

返回确切技能版本的当前精简安全结论。此
集合端点适用于已经知道需要显示哪些已安装
ClawHub 技能版本的客户端，例如 OpenClaw Control UI。

请求：

```json
{
  "items": [{ "slug": "gifgrep", "version": "1.2.3" }]
}
```

注意：

- `items` 必须包含 1-100 个唯一的 `{ slug, version }` 对。
- 结果按项目返回；某个技能或版本缺失不会导致整个响应失败。
- 响应仅包含安全信息。它不包含技能卡数据、已生成卡片状态、工件文件列表或详细扫描器负载。
- `security.signals` 仅包含状态级支持证据；完整扫描器详细信息请使用 `/scan` 或 ClawHub 安全审计页面。
- `security.signals.dependencyRegistry` 为兼容 v1 响应而保留，但依赖项注册表存在性扫描器已停用，此键始终为 `null`。
- 缺少技能卡不会影响此端点的 `ok`、`decision` 或 `reasons`；客户端需要卡片内容时，应在本地读取已安装的 `skill-card.md`。
- 需要单个技能的技能卡验证信封时使用 `/verify`，需要生成的卡片 Markdown 时使用 `/card`，需要详细扫描器数据时使用 `/scan`。

响应：

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
      "error": { "code": "version_not_found", "message": "未找到版本" },
      "security": null
    }
  ]
}
```

### `GET /api/v1/skills/{slug}/file`

将确切存储的文件字节作为下载返回。添加 `preview=1` 可请求有界的转义文本
预览；只要文件字节是有效的 UTF-8，无论其扩展名或 MIME
元数据如何，都可以预览。

查询参数：

- `path`（必填）
- `version`（可选）
- `tag`（可选）
- `preview=1`（可选；字节不是有效 UTF-8 时返回 `text/plain` 或 `415`）

注意：

- 默认为最新版本。
- 原始下载限制：10MB。
- 文本预览限制：200KB。

### `GET /api/v1/packages`

以下内容的统一目录端点：

- 技能
- 代码插件
- 捆绑插件

查询参数：

- `limit`（可选）：整数（1–100）
- `cursor`（可选）：分页游标
- `family`（可选）：`skill`、`code-plugin` 或 `bundle-plugin`
- `channel`（可选）：`official`、`community` 或 `private`
- `isOfficial`（可选）：`true` 或 `false`
- `sort`（可选）：`updated`（默认）、`recommended`、`trending`、`downloads`、旧版别名 `installs`
- `category`（可选）：插件类别筛选器。仅当
  请求限定为插件软件包（`/api/v1/plugins`、
  `/api/v1/code-plugins`、`/api/v1/bundle-plugins`，或包含
  `family=code-plugin`/`family=bundle-plugin` 的软件包端点）时才受支持。受控类别和
  旧版 v1 筛选器别名记录在 `GET /api/v1/plugins` 下。

注意：

- `family`、`channel`、`isOfficial`、`featured`、
  `highlightedOnly` 或 `sort` 的无效值将返回 `400`。未知查询参数会被忽略。
- `GET /api/v1/code-plugins` 和 `GET /api/v1/bundle-plugins` 保持为固定系列别名。
- 技能条目仍由技能注册表支持，并且仍只能通过 `POST /api/v1/skills` 发布。
- `POST /api/v1/packages` 仍仅用于代码插件和捆绑插件版本。
- 匿名调用方只能看到公开软件包渠道。
- 已验证身份的调用方可以在列表/搜索结果中看到其所属发布者的私有软件包。
- `channel=private` 仅返回已验证身份的调用方有权读取的软件包。

### `GET /api/v1/packages/search`

跨技能和插件软件包的统一目录搜索。

查询参数：

- `q`（必填）：查询字符串
- `limit`（可选）：整数（1–100）
- `family`（可选）：`skill`、`code-plugin` 或 `bundle-plugin`
- `channel`（可选）：`official`、`community` 或 `private`
- `isOfficial`（可选）：`true` 或 `false`
- `category`（可选）：插件类别筛选器。仅当请求范围限定为插件包时
  才受支持。受控类别和旧版 v1 筛选器别名记录在 `GET /api/v1/plugins` 下。

注意：

- `family`、`channel`、`isOfficial`、`featured` 或
  `highlightedOnly` 的无效值会返回 `400`。未知的查询参数将被忽略。
- 匿名调用方只能看到公开包渠道。
- 经过身份验证的调用方可以搜索其所属发布者的私有包。
- `channel=private` 仅返回经过身份验证的调用方可读取的包。

### `GET /api/v1/plugins`

仅浏览代码插件包和插件包集合的插件目录。

查询参数：

- `limit`（可选）：整数（1-100）
- `cursor`（可选）：分页游标
- `isOfficial`（可选）：`true` 或 `false`
- `sort`（可选）：`recommended`（默认）、`trending`、`downloads`、`updated`、旧版别名 `installs`
- `category`（可选）：插件类别筛选器。当前值：
  `channels`、`models`、`memory`、`context`、`voice`、`media`、`web`、
  `tools`、`runtime`、`gateway`、`security`、`other`。

读取端点仍接受旧版 v1 筛选器别名：

- `mcp-tooling`、`data` 和 `automation` 解析为 `tools`。
- `observability` 和 `deployment` 解析为 `gateway`。
- `dev-tools` 解析为 `runtime`。

`trending` 是按七天统计的安装/下载排行榜，不使用累计总数。
在统一的 `/api/v1/packages` 端点上，它仅适用于插件；技能目录请使用
`/api/v1/skills?sort=trending`。

旧版别名不能用作存储的或作者声明的类别值。

### `GET /api/v1/skills/export`

批量导出最新的公开技能，以供离线分析。

身份验证：

- 需要 API 令牌。

查询参数：

- `startDate`（必填）：技能 `updatedAt` 的 Unix 毫秒时间下限。
- `endDate`（必填）：技能 `updatedAt` 的 Unix 毫秒时间上限。
- `limit`（可选）：整数（1-250），默认为 `250`。
- `cursor`（可选）：来自上一次响应的分页游标。

响应：

- 正文：ZIP 归档。
- 每个导出的技能均以 `{publisher}/{slug}/` 为根目录。
- 托管技能包含最新存储版本的文件，并在
  `_manifest.json` 中以 `sourceRef: "public-clawhub"` 列出。
- 当前由 GitHub 支持且具有 `clean` 或 `suspicious` 扫描结果的技能包含
  `_source_handoff.json`，其中含有 `sourceRef: "public-github"`、仓库、提交、路径、
  内容哈希和归档 URL。它们不包含由 ClawHub 托管的源文件。
- 每个技能都包含 `_export_skill_meta.json`。
- `_manifest.json` 始终包含在 ZIP 根目录中。
- 当个别技能或文件无法导出时，将包含
  `_errors.json`。

响应头：

- `X-Next-Cursor`
- `X-Has-More`
- `X-Total-Returned`
- `X-Date-Range`
- `X-Export-Errors`

### `GET /api/v1/plugins/export`

批量导出最新的公开插件版本，以供离线分析。

身份验证：

- 需要 API 令牌。

查询参数：

- `startDate`（必填）：插件 `updatedAt` 的 Unix 毫秒时间下限。
- `endDate`（必填）：插件 `updatedAt` 的 Unix 毫秒时间上限。
- `limit`（可选）：整数（1-250），默认为 `250`。
- `cursor`（可选）：来自上一次响应的分页游标。
- `family`（可选）：`code-plugin` 或 `bundle-plugin`。省略表示同时包含两种
  插件系列。

响应：

- 正文：ZIP 归档。
- 每个导出的插件均以 `{family}/{packageName}/` 为根目录。
- 每个导出的插件都包含最新版本所存储的文件。
- 每个插件的导出元数据存储在
  `__clawhub_export/{family}/{packageName}/plugin_meta.json`。
- `_manifest.json` 始终包含在 ZIP 根目录中。
- 当个别插件或文件无法导出时，将包含
  `_errors.json`。

响应头：

- `X-Next-Cursor`
- `X-Has-More`
- `X-Total-Returned`
- `X-Date-Range`
- `X-Export-Errors`

### `GET /api/v1/plugins/search`

仅搜索代码插件包和插件包集合。

查询参数：

- `q`（必填）：查询字符串
- `limit`（可选）：整数（1-100）
- `isOfficial`（可选）：`true` 或 `false`
- `category`（可选）：插件类别筛选器。当前值：
  `channels`、`models`、`memory`、`context`、`voice`、`media`、`web`、
  `tools`、`runtime`、`gateway`、`security`、`other`。

注意：

- 也接受记录在 `GET /api/v1/plugins` 下的旧版 v1 筛选器别名。
- 类别筛选是由插件类别摘要行支持的真实 API 筛选器，
  而不是对搜索查询的改写。
- 结果按相关性顺序返回，目前不支持分页。
- 浏览器 UI 中的插件搜索排序控件会对已加载的相关性结果重新排序，
  与当前 `/skills` 的浏览行为一致。

### `GET /api/v1/packages/{name}`

返回包的详细元数据。

注意：

- 在统一目录中，技能也可以通过此路由解析。
- 除非调用方可以读取所属发布者，否则私有包返回 `404`。

### `DELETE /api/v1/packages/{name}`

软删除包及其所有版本。

注意：

- 需要包所有者、组织发布者的所有者/管理员、平台版主或平台管理员的 API 令牌。

### `GET /api/v1/packages/{name}/versions`

返回版本历史记录。

查询参数：

- `limit`（可选）：整数（1–100）
- `cursor`（可选）：分页游标

注意：

- 除非调用方可以读取所属发布者，否则私有包返回 `404`。

### `GET /api/v1/packages/{name}/versions/{version}`

返回单个包版本，包括文件元数据、兼容性、
验证、工件元数据和扫描数据。

注意：

- 对于旧式包归档，`version.artifact.kind` 为 `legacy-zip`；对于
  由 ClawPack 支持的版本，则为 `npm-pack`。
- ClawPack 版本包含与 npm 兼容的 `npmIntegrity`、`npmShasum` 和
  `npmTarballName` 字段。
- `version.sha256hash` 是面向旧客户端的已弃用兼容性元数据。它
  对 `/api/v1/packages/{name}/download` 返回的精确 ZIP 字节进行哈希计算。
  现代客户端应使用 `version.artifact.sha256`，它用于标识
  规范版本工件。
- 存在扫描数据时，会包含 `version.vtAnalysis`、`version.llmAnalysis` 和 `version.staticScan`。
- 除非调用方可以读取所属发布者，否则私有包返回 `404`。

### `GET /api/v1/packages/{name}/versions/{version}/security`

向安装客户端返回确切的包版本安全与信任摘要。这是公开的 OpenClaw 使用界面，用于决定
已解析的版本是否可以安装。

身份验证：

- 公开读取端点。不需要所有者、发布者、版主或管理员令牌。

响应：

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

响应字段：

- `package.name`、`package.displayName` 和 `package.family` 标识
  已解析的注册表包。
- `release.releaseId`、`release.version` 和 `release.createdAt` 标识
  经过评估的确切版本。
- 已知版本工件的相关信息时，会包含 `release.artifactKind`、`release.artifactSha256`、`release.npmIntegrity`、
  `release.npmShasum` 和 `release.npmTarballName`。
- `trust.scanStatus` 是根据扫描器输入和手动版本审核得出的有效信任状态。
- `trust.moderationState` 可为空。不存在手动版本审核时，其值为 `null`。
- `trust.blockedFromDownload` 是安装阻止信号。当此值为 `true` 时，OpenClaw 和其他
  安装客户端应阻止安装，而不是根据扫描器或审核字段重新推导阻止规则。
- `trust.reasons` 是面向用户和审计的说明列表。原因代码是
  稳定且紧凑的字符串，例如 `manual:quarantined`、`scan:malicious`
  和 `package:malicious`。
- `trust.pending` 表示一个或多个信任输入仍在等待完成。
- `trust.stale` 表示信任摘要是根据过时的输入计算得出的，
  在作出高置信度允许决定之前，应将其视为需要刷新。

注意：

- 此端点精确对应版本。客户端应在解析出准备安装的
  包版本后调用它，而不应仅在读取最新的包元数据后调用。
- 除非调用方可以读取所属发布者，否则私有包返回 `404`。
- 此端点有意比所有者/版主审核端点的范围更窄。它公开的是
  安装决定和公开说明，而不是举报者身份、举报正文、私有证据或内部审核
  时间线。

### `GET /api/v1/packages/{name}/versions/{version}/artifact`

返回包版本的显式工件解析器元数据。

注意：

- 旧版包版本返回 `legacy-zip` 工件和旧版 ZIP
  `downloadUrl`。
- ClawPack 版本返回 `npm-pack` 工件、npm 完整性字段、
  `tarballUrl` 和旧版 ZIP 兼容性 URL。
- 这是 OpenClaw 解析器界面；它无需通过共享 URL 猜测归档格式。

### `GET /api/v1/packages/{name}/versions/{version}/artifact/download`

通过显式解析器路径下载版本工件。

注意：

- ClawPack 版本会流式传输已上传的 npm-pack `.tgz` 的精确字节。
- 旧版 ZIP 版本会重定向到 `/api/v1/packages/{name}/download?version=`。
- 使用下载速率桶。

### `GET /api/v1/packages/{name}/readiness`

返回为 OpenClaw 未来使用而计算的就绪状态。

就绪检查包括：

- 官方渠道状态
- 最新版本可用性
- ClawPack npm-pack 工件可用性
- 工件摘要
- 源代码仓库和提交来源
- OpenClaw 兼容性元数据
- 主机目标
- 扫描状态

响应：

```json
{
  "package": {
    "name": "@openclaw/example-plugin",
    "displayName": "示例插件",
    "family": "code-plugin",
    "isOfficial": true,
    "latestVersion": "1.2.3"
  },
  "ready": false,
  "checks": [
    {
      "id": "clawpack",
      "label": "ClawPack 工件",
      "status": "fail",
      "message": "最新版本仅提供旧版 ZIP。"
    }
  ],
  "blockers": ["clawpack"]
}
```

### `GET /api/v1/packages/migrations`

用于列出官方 OpenClaw 插件迁移记录的审核员端点。

身份验证：

- 需要审核员或管理员用户的 API 令牌。

查询参数：

- `phase`（可选）：`planned`、`published`、`clawpack-ready`、
  `legacy-zip-only`、`metadata-ready`、`blocked`、`ready-for-openclaw` 或
  `all`（默认）。
- `limit`（可选）：整数（1-100）
- `cursor`（可选）：分页游标

响应：

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

用于创建或更新官方插件迁移记录的管理员端点。

身份验证：

- 需要管理员用户的 API 令牌。

请求正文：

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
  "notes": "正在等待发布者上传"
}
```

注意：

- `bundledPluginId` 会标准化为小写，并作为稳定的 upsert 键。
- `packageName` 会按 npm 名称规则标准化；对于计划中的
  迁移，软件包可以不存在。
- 这仅跟踪迁移就绪状态，不会修改 OpenClaw 或生成
  ClawPack。

### `GET /api/v1/packages/moderation/queue`

用于软件包版本审核队列的审核员/管理员端点。

身份验证：

- 需要审核员或管理员用户的 API 令牌。

查询参数：

- `status`（可选）：`open`（默认）、`blocked`、`manual` 或 `all`
- `limit`（可选）：整数（1-100）
- `cursor`（可选）：分页游标

状态含义：

- `open`：可疑、恶意、待处理、已隔离、已撤销或已举报的版本。
- `blocked`：已隔离、已撤销或恶意版本。
- `manual`：任何具有手动审核覆盖设置的版本。
- `all`：任何具有手动覆盖设置、非干净扫描状态或软件包举报的版本。

响应：

```json
{
  "items": [
    {
      "packageId": "packages:...",
      "releaseId": "packageReleases:...",
      "name": "@openclaw/example-plugin",
      "displayName": "示例插件",
      "family": "code-plugin",
      "channel": "community",
      "isOfficial": false,
      "version": "1.2.3",
      "createdAt": 1730000000000,
      "artifactKind": "npm-pack",
      "scanStatus": "malicious",
      "moderationState": "quarantined",
      "moderationReason": "人工审核",
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

举报软件包以供审核员审查。举报以软件包为单位，可选择
关联到某个版本。举报会进入审核队列，但本身不会自动隐藏软件包或
阻止下载；审核员应使用版本审核来批准、隔离或撤销工件。

身份验证：

- 需要 API 令牌。

请求：

```json
{ "reason": "可疑的原生二进制文件", "version": "1.2.3" }
```

响应：

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

用于接收软件包举报的审核员/管理员端点。

身份验证：

- 需要审核员或管理员用户的 API 令牌。

查询参数：

- `status`（可选）：`open`（默认）、`confirmed`、`dismissed` 或 `all`
- `limit`（可选）：整数（1-100）
- `cursor`（可选）：分页游标

响应：

```json
{
  "items": [
    {
      "reportId": "packageReports:...",
      "packageId": "packages:...",
      "releaseId": "packageReleases:...",
      "name": "@openclaw/example-plugin",
      "displayName": "示例插件",
      "family": "code-plugin",
      "version": "1.2.3",
      "reason": "可疑的原生二进制文件",
      "status": "open",
      "createdAt": 1730000000000,
      "reporter": {
        "userId": "users:...",
        "handle": "reporter",
        "displayName": "举报者"
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

用于查看软件包审核可见性的所有者/审核员端点。

身份验证：

- 需要软件包所有者、发布者成员、审核员或
  管理员用户的 API 令牌。

响应：

```json
{
  "package": {
    "packageId": "packages:...",
    "name": "@openclaw/example-plugin",
    "displayName": "示例插件",
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
    "moderationReason": "人工审核",
    "blockedFromDownload": true,
    "reasons": ["manual:quarantined", "scan:malicious", "reports:2"],
    "createdAt": 1730000000000
  }
}
```

### `POST /api/v1/packages/reports/{reportId}/triage`

用于解决或重新打开软件包举报的审核员/管理员端点。

请求：

```json
{
  "status": "confirmed",
  "note": "已审查并隔离受影响的版本。",
  "finalAction": "quarantine"
}
```

`note` 对于 `confirmed` 和 `dismissed` 是必需的；将
`status` 重新设置为 `open` 时可以省略。对已确认的举报传递
`finalAction: "quarantine"` 或 `finalAction: "revoke"`，即可在同一个可审计工作流中应用版本审核。

响应：

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

用于软件包版本审核的审核员/管理员端点。

请求：

```json
{ "state": "quarantined", "reason": "可疑的原生载荷。" }
```

支持的状态：

- `approved`：已人工审核并允许。
- `quarantined`：在后续处理前阻止。
- `revoked`：先前受信任的版本之后被阻止。

已隔离和已撤销的版本会从工件下载路由返回 `403`。
每次更改都会写入审计日志条目。

### `GET /api/v1/packages/{name}/file`

以下载形式返回存储的软件包文件的精确字节。添加 `preview=1` 可请求用于技能文件的同一种有界
UTF-8 文本预览。

查询参数：

- `path`（必需）
- `version`（可选）
- `tag`（可选）
- `preview=1`（可选；字节不是有效 UTF-8 时返回 `text/plain` 或 `415`）

注意：

- 默认为最新版本。
- 使用读取速率桶，而不是下载速率桶。
- 原始下载限制：10MB。
- 文本预览限制：200KB；不透明文件仅在预览请求中返回 `415`。
- 待处理的 VirusTotal 扫描不会阻止读取；恶意版本在其他位置仍可能被阻止提供。
- 除非调用方可以读取所属发布者，否则私有软件包返回 `404`。

### `GET /api/v1/packages/{name}/download`

下载软件包版本的旧版确定性 ZIP 归档。

查询参数：

- `version`（可选）
- `tag`（可选）

注意：

- 默认为最新版本。
- Skills 会重定向到 `GET /api/v1/download`。
- 插件/软件包归档是以 `package/` 为根目录的 zip 文件，以便旧版 OpenClaw
  客户端继续工作。
- 此路由仅保留 ZIP，不会流式传输 ClawPack `.tgz` 文件。
- 响应包含 `ETag`、`Digest`、`X-ClawHub-Artifact-Type` 和
  `X-ClawHub-Artifact-Sha256` 标头，用于解析器完整性检查。
- 仅注册表使用的元数据不会注入下载的归档。
- 待处理的 VirusTotal 扫描不会阻止下载；恶意版本返回 `403`。
- 除非调用方是所有者，否则私有软件包返回 `404`。

### `GET /api/npm/{package}`

返回由 ClawPack 支持的软件包版本的 npm 兼容 packument。

注意：

- 仅列出已上传 ClawPack npm-pack tarball 的版本。
- 有意省略仅提供旧版 ZIP 的版本。
- `dist.tarball`、`dist.integrity` 和 `dist.shasum` 使用 npm 兼容
  字段，因此用户可以选择将 npm 指向该镜像。
- 限定作用域软件包的 packument 同时支持 `/api/npm/@scope/name` 和 npm 的
  编码请求路径 `/api/npm/@scope%2Fname`。

### `GET /api/npm/{package}/-/{tarball}.tgz`

为 npm 镜像客户端流式传输已上传 ClawPack tarball 的精确字节。

注意：

- 使用下载速率桶。
- 下载标头包含 ClawHub SHA-256，以及 npm integrity/shasum 元数据。
- 审核和私有软件包访问检查仍然适用。

### `GET /api/v1/resolve`

由 CLI 用于将本地指纹映射到已知版本。

查询参数：

- `slug`（必需）
- `hash`（必需）：包指纹的 64 字符十六进制 sha256

响应：

```json
{ "slug": "gifgrep", "match": { "version": "1.2.2" }, "latestVersion": { "version": "1.2.3" } }
```

### `GET /api/v1/download`

下载托管的技能版本 ZIP；对于当前由 GitHub 支持、具有 `clean` 或 `suspicious` 扫描且没有托管版本的技能，则返回 GitHub 源代码移交信息。

查询参数：

- `slug`（必填）
- `version`（可选）：semver 字符串
- `tag`（可选）：标签名称（例如 `latest`）

注意事项：

- 如果既未提供 `version`，也未提供 `tag`，则使用最新版本。
- 软删除的版本返回 `410`。
- 由 GitHub 支持的技能移交不会代理或镜像字节。JSON 响应
  包含 `sourceRef: "public-github"`、`repo`、`commit`、`path`、`contentHash`
  和 `archiveUrl`；扫描/当前状态仅作为门控条件，不会作为成功
  载荷的元数据包含在内。
- 下载统计按 UTC 日内的唯一身份计数（API 令牌有效时使用 `userId`，否则使用 IP）。

## 身份验证端点（Bearer 令牌）

所有端点均要求：

```
Authorization: Bearer clh_...
```

### `GET /api/v1/whoami`

验证令牌并返回用户 handle。

### `POST /api/v1/skills`

发布新版本。

- 首选：使用包含 `payload` JSON 和 `files[]` blob 的 `multipart/form-data`。
- 也接受包含 `files`（基于 storageId）的 JSON 正文。
- 可选载荷字段：`ownerHandle`。提供该字段时，API 会在服务器端解析该
  发布者，并要求执行者拥有该发布者的访问权限。
- 可选载荷字段：`migrateOwner`。当 `true` 与 `ownerHandle` 一同使用时，
  如果执行者同时是当前和目标发布者的管理员/所有者，则可将现有技能移交给该所有者。
  若未明确选择此选项，则拒绝更改所有者。

### `POST /api/v1/packages`

发布代码插件或捆绑插件版本。

- 需要 Bearer 令牌身份验证。
- 需要 `multipart/form-data`。
- 允许的表单字段包括 `payload`、重复的 `files` blob，或一个 `clawpack`
  tarball 引用。`clawpack` 可以是 `.tgz` blob，也可以是上传 URL 流程返回的存储 ID。
  使用暂存存储 ID 发布时，还必须包含该上传 URL 返回的
  `clawpackUploadTicket`。
- 使用 `files` 或 `clawpack` 中的任意一个，切勿在同一请求中同时使用二者。
- 拒绝 JSON 正文和调用方提供的 `payload.files` / `payload.artifact`
  元数据。
- 直接 multipart 发布请求上限为 18MB。ClawPack tarball 可
  使用上传 URL 流程，最高达到 120MB 的 tarball 上限。
- 可选载荷字段：`ownerHandle`。提供该字段时，只有管理员可以代表该所有者发布。

验证要点：

- `family` 必须为 `code-plugin` 或 `bundle-plugin`。
- 插件包需要 `openclaw.plugin.json`。ClawPack `.tgz` 上传必须
  在 `package/openclaw.plugin.json` 中包含该字段。
- 代码插件需要 `package.json`、源代码仓库元数据、源代码提交
  元数据、配置架构元数据、`openclaw.compat.pluginApi` 和
  `openclaw.build.openclawVersion`。
- `openclaw.hostTargets` 和 `openclaw.environment` 是可选元数据。
- 只有 `openclaw` 组织发布者和当前 `openclaw` 组织成员的
  个人发布者可以发布到 `official` 渠道。
- 代表他人发布时，仍会根据目标所有者账户验证官方渠道资格。

### `DELETE /api/v1/skills/{slug}` / `POST /api/v1/skills/{slug}/undelete`

软删除/恢复技能（所有者、版主或管理员）。

可选 JSON 正文：

```json
{ "reason": "因等待法律审查而暂缓，供版主管理。" }
```

提供 `reason` 时，它将存储为技能的版主管理备注，并复制到审计日志中。
所有者发起的软删除会保留 slug 30 天，之后其他发布者可认领该 slug。
当此过期规则适用时，删除响应会包含 `slugReservedUntil`。
版主/管理员隐藏和安全移除不会以这种方式过期。

删除响应：

```json
{ "ok": true, "slugReservedUntil": 1730000000000 }
```

状态码：

- `200`：成功
- `401`：未授权
- `403`：禁止访问
- `404`：未找到技能/用户
- `500`：内部服务器错误

### `POST /api/v1/users/publisher`

仅限管理员。确保指定 handle 存在组织发布者。如果该 handle 仍指向
旧版共享用户/个人发布者，该端点会先将其迁移为组织发布者。
对于新创建的组织，请提供 `memberHandle`；不会将执行操作的管理员添加为成员。
`memberRole` 默认为 `owner`。

- 正文：`{ "handle": "openclaw", "displayName": "OpenClaw", "memberHandle": "alice", "memberRole": "owner", "trusted": true }`
- 响应：`{ "ok": true, "publisherId": "...", "handle": "openclaw", "created": true, "migrated": false, "trusted": true, "member": { "userId": "...", "handle": "alice", "role": "owner" } }`

### `POST /api/v1/publishers`

经过身份验证的自助式组织发布者创建。创建新的组织发布者，并将
调用方添加为所有者。此端点不会迁移现有的用户/个人 handle，也
不会将发布者标记为受信任/官方发布者。

- 正文：`{ "handle": "opik", "displayName": "Opik" }`
- 响应：`{ "ok": true, "publisherId": "...", "handle": "opik", "created": true, "trusted": false }`
- 当该 handle 已被发布者、用户或个人发布者使用时，返回 `409`。

### `POST /api/v1/users/reserve`

仅限管理员。为合法所有者保留根 slug 和包名称，而不发布
版本。包名称会成为没有版本行的私有占位包，因此同一
所有者之后可以将真正的代码插件或捆绑插件版本发布到该名称下。

- 正文：`{ "handle": "openclaw", "slugs": ["diffs"], "packageNames": ["@openclaw/diffs"], "reason": "reserved for official OpenClaw plugin" }`
- 响应：`{ "ok": true, "succeeded": 2, "failed": 0, "results": [{ "kind": "slug", "name": "diffs", "ok": true, "action": "reserved" }] }`

### `POST /api/v1/users/publisher-recovery`

仅限管理员。为经过验证的替代 GitHub OAuth 主体恢复个人发布者，
无需编辑 Convex Auth 账户行。请求必须指定两个不可变的 GitHub
提供商账户 ID；可变 handle 仅用作面向操作员的防护条件。

该端点默认为试运行。应用恢复需要 `dryRun: false` 和
`confirmIdentityVerified: true`，且工作人员须独立验证两个
GitHub 主体之间的连续性。如果目标用户当前的个人
发布者拥有技能、包或 GitHub 技能源，恢复将以失败关闭。
恢复还会迁移已恢复发布者的技能、技能 slug 别名、包、包检查器警告和
派生搜索摘要行中的旧版 `ownerUserId` 字段，使
直接所有者路径与新的发布者权限保持一致。已恢复 handle 的有效受保护 handle
保留项也会重新分配给替代用户，以免后续
个人资料同步恢复原用户相互冲突的权限。每个主表在每次应用事务中最多处理
100 行；规模更大的恢复必须先使用可恢复的所有者迁移。
GitHub 技能源以发布者为作用域，并报告为已检查，而不是重写。

- 正文：`{ "handle": "gingiris", "nextUserHandle": "gingiris-1031", "previousGitHubProviderAccountId": "123", "nextGitHubProviderAccountId": "456", "reason": "Verified account continuity for issue #2555", "confirmIdentityVerified": true, "dryRun": false }`
- 响应：`{ "ok": true, "dryRun": false, "recovered": true, "publisherId": "...", "handle": "gingiris", "previousUser": { "userId": "...", "handle": "gingiris", "nextHandle": "gingiris-recovered", "githubProviderAccountId": "123", "authAccountCount": 1 }, "nextUser": { "userId": "...", "handle": "gingiris-1031", "nextHandle": "gingiris", "githubProviderAccountId": "456", "authAccountCount": 1 }, "retiredPersonalPublisher": null, "resourceOwnerMigration": { "limitPerTable": 100, "skills": 1, "skillSlugAliases": 1, "packages": 0, "packageInspectorWarnings": 0, "githubSourcesChecked": 1, "handleReservations": 1 }, "identityVerified": true, "reason": "Verified account continuity for issue #2555" }`

### 所有者 slug 管理端点

- `POST /api/v1/skills/{slug}/rename`
  - 正文：`{ "newSlug": "new-canonical-slug" }`
  - 响应：`{ "ok": true, "slug": "new-canonical-slug", "previousSlug": "old-slug" }`
- `POST /api/v1/skills/{slug}/merge`
  - 正文：`{ "targetSlug": "canonical-target-slug" }`
  - 响应：`{ "ok": true, "sourceSlug": "old-slug", "targetSlug": "canonical-target-slug" }`

注意事项：

- 两个端点均需要 API 令牌身份验证，且仅适用于技能所有者。
- `rename` 将以前的 slug 保留为重定向别名。
- `merge` 会隐藏源列表，并将源 slug 重定向到目标列表。

### 所有权转移端点

- `POST /api/v1/skills/{slug}/transfer`
  - 正文：`{ "toUserHandle": "target_handle", "message": "optional" }`
  - 响应：`{ "ok": true, "transferId": "skillOwnershipTransfers:...", "toUserHandle": "target_handle", "expiresAt": 1730000000000 }`
- `POST /api/v1/skills/{slug}/transfer/accept`
- `POST /api/v1/skills/{slug}/transfer/reject`
- `POST /api/v1/skills/{slug}/transfer/cancel`
  - 响应（接受/拒绝/取消）：`{ "ok": true, "skillSlug": "demo-skill?" }`
- `GET /api/v1/transfers/incoming`
- `GET /api/v1/transfers/outgoing`
  - 响应结构：`{ "transfers": [{ "_id": "...", "skill": { "slug": "demo", "displayName": "Demo" }, "fromUser"|"toUser": { "handle": "..." }, "message": "...", "requestedAt": 0, "expiresAt": 0 }] }`

### `POST /api/v1/users/ban`

封禁用户并硬删除其拥有的技能（仅限版主/管理员）。

正文：

```json
{ "handle": "user_handle", "reason": "可选的封禁原因" }
```

或

```json
{ "userId": "users_...", "reason": "可选的封禁原因" }
```

响应：

```json
{ "ok": true, "alreadyBanned": false, "deletedSkills": 3 }
```

### `POST /api/v1/users/unban`

解除用户封禁并恢复符合条件的技能（仅限管理员）。

正文：

```json
{ "handle": "user_handle", "reason": "可选的解封原因" }
```

或

```json
{ "userId": "users_...", "reason": "可选的解封原因" }
```

响应：

```json
{ "ok": true, "alreadyUnbanned": false, "restoredSkills": 3 }
```

### `POST /api/v1/users/reclassify-ban`

更改现有封禁所存储的原因，而不解除封禁或恢复
内容（仅限管理员）。除非 `dryRun` 为 `false`，否则默认为试运行。

正文：

```json
{ "handle": "user_handle", "reason": "批量发布垃圾内容", "dryRun": true }
```

或

```json
{ "userId": "users_...", "reason": "批量发布垃圾内容", "dryRun": false }
```

响应：

```json
{
  "ok": true,
  "dryRun": false,
  "userId": "users_...",
  "handle": "user_handle",
  "previousReason": "恶意软件自动封禁",
  "nextReason": "批量发布垃圾内容",
  "changed": true
}
```

### `POST /api/v1/users/role`

更改用户角色（仅限管理员）。

正文：

```json
{ "handle": "user_handle", "role": "moderator" }
```

或

```json
{ "userId": "users_...", "role": "admin" }
```

响应：

```json
{ "ok": true, "role": "moderator" }
```

### `GET /api/v1/users`

列出或搜索用户（仅限管理员）。

查询参数：

- `q`（可选）：搜索查询
- `query`（可选）：`q` 的别名
- `limit`（可选）：最大结果数（默认 20，最大 200）

响应：

```json
{
  "items": [
    {
      "userId": "users_...",
      "handle": "user_handle",
      "displayName": "用户",
      "name": "用户",
      "role": "moderator"
    }
  ],
  "total": 1
}
```

### `POST /api/v1/stars/{slug}` / `DELETE /api/v1/stars/{slug}`

添加/移除书签。为保持兼容性，旧版 `stars` 路由和响应字段名称仍然
保留。两个端点均具有幂等性。

响应：

```json
{ "ok": true, "starred": true, "alreadyStarred": false }
```

```json
{ "ok": true, "unstarred": true, "alreadyUnstarred": false }
```

## 旧版 CLI 端点（已弃用）

仍支持旧版 CLI：

- `GET /api/cli/whoami`
- `POST /api/cli/upload-url`
- `POST /api/cli/publish`
- `POST /api/cli/telemetry/install`
- `POST /api/cli/skill/delete`
- `POST /api/cli/skill/undelete`

移除计划请参阅 `DEPRECATIONS.md`。

`POST /api/cli/upload-url` 返回 `uploadUrl` 和 `uploadTicket`。暂存 ClawPack tarball 的包
发布必须将生成的存储 ID 作为 `clawpack` 发送，并将返回的票据作为 `clawpackUploadTicket` 发送。

## 注册表发现（`/.well-known/clawhub.json`）

CLI 可以从站点发现注册表/身份验证设置：

- `/.well-known/clawhub.json`（JSON，首选）
- `/.well-known/clawdhub.json`（旧版）

架构：

```json
{ "apiBase": "https://clawhub.ai", "authBase": "https://clawhub.ai", "minCliVersion": "0.0.5" }
```

如果自行托管，请提供此文件（或显式设置 `CLAWHUB_REGISTRY`；旧版为 `CLAWDHUB_REGISTRY`）。
