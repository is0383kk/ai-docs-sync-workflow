---
read_when:
    - 发布技能
    - 调试发布失败问题
summary: 技能文件夹格式、必需文件、支持性工件和限制。
x-i18n:
    generated_at: "2026-07-26T06:08:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fdf16a589b8961ccd9181a53a9fa92a358952b9147d22eaf977f23e0b4b4d653
    source_path: clawhub/skill-format.md
    workflow: 16
---

# 技能格式

## 磁盘结构

技能是一个文件夹。

必需：

- `SKILL.md`（或 `skill.md`；也接受旧版 `skills.md`）

可选：

- 任何常规支持文件（参见“技能文件”）
- `.clawhubignore`（发布时使用的忽略模式，旧版为 `.clawdhubignore`）
- `.gitignore`（同样会采用）

## GitHub 导入

Web 端 GitHub 导入器比本地发布/同步更严格。它仅发现
已登录 GitHub 账户所拥有的公开、非复刻仓库中的 `SKILL.md` 或旧版
`skills.md` 文件。它不会导入私有仓库、复刻仓库、
已归档/已禁用仓库或第三方公开仓库。

本地安装元数据（由 CLI 写入）：

- `<skill>/.clawhub/origin.json`（旧版为 `.clawdhub`）

工作目录安装状态（由 CLI 写入）：

- `<workdir>/.clawhub/lock.json`（旧版为 `.clawdhub`）

## `SKILL.md`

- 带有可选 YAML frontmatter 的 Markdown。
- 服务器在发布期间从 frontmatter 中提取元数据。
- `description` 用作 UI/搜索中的技能摘要。

对于可移植的 Agent Skills，`name` 应与父目录名称一致，并使用
1–64 个小写字母、数字或连字符。ClawHub 将可路由的 slug 与
目录显示名称分开保存，因此来自其他客户端的现有名称仍可
发布，且不会被静默改写。目录列表可能会在视觉上缩短过长的名称，
但不会更改存储的名称。

## Frontmatter 元数据

技能元数据在 `SKILL.md` 顶部的 YAML frontmatter 中声明。它会告知注册表（以及安全分析）技能运行时需要哪些条件。

### 基本 frontmatter

```yaml
---
name: my-skill
description: 此技能功能的简短摘要。
version: 1.0.0
---
```

### 运行时元数据（`metadata.openclaw`）

在 `metadata.openclaw` 下声明技能的运行时要求（别名：`metadata.clawdbot`、`metadata.clawdis`）。

```yaml
---
name: my-skill
description: 通过 Todoist API 管理任务。
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

使用 `requires.env` 声明技能运行前必须存在的环境变量。当需要各变量的元数据（包括通过 `required: false` 声明的可选变量）时，请使用 `envVars`。

### 完整字段参考

| 字段               | 类型       | 描述                                                                                                                                         |
| ------------------ | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `requires.env`     | `string[]` | 技能所需的环境变量。                                                                                                                         |
| `requires.bins`    | `string[]` | 必须全部安装的 CLI 二进制文件。                                                                                                              |
| `requires.anyBins` | `string[]` | 至少必须存在一个的 CLI 二进制文件。                                                                                                          |
| `requires.config`  | `string[]` | 技能读取的配置文件路径。                                                                                                                     |
| `primaryEnv`       | `string`   | 技能的主要凭据环境变量。                                                                                                                     |
| `envVars`          | `array`    | 带有 `name`、可选 `required` 和可选 `description` 的环境变量声明。对于可选环境变量，请设置 `required: false`。 |
| `always`           | `boolean`  | 如果为 `true`，技能将始终处于活动状态（无需显式安装）。                                                                           |
| `skillKey`         | `string`   | 覆盖技能的调用键。                                                                                                                           |
| `emoji`            | `string`   | 技能的显示表情符号。                                                                                                                         |
| `homepage`         | `string`   | 技能主页或文档的 URL。                                                                                                                       |
| `os`               | `string[]` | 操作系统限制（例如 `["macos"]`、`["linux"]`）。                                                                                |
| `install`          | `array`    | 依赖项的安装规范（见下文）。                                                                                                                 |
| `nix`              | `object`   | Nix 插件规范（参见 README）。                                                                                                                |
| `config`           | `object`   | Clawdbot 配置规范（参见 README）。                                                                                                           |

### 安装规范

如果技能需要安装依赖项，请在 `install` 数组中声明：

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

支持的安装类型：`brew`、`node`、`go`、`uv`。

### 可选环境变量

在 `metadata.openclaw.envVars` 下声明可选环境变量，并设置 `required: false`。不要将可选条目添加到 `requires.env`，因为 `requires.env` 表示缺少这些变量时技能无法运行。

```yaml
metadata:
  openclaw:
    primaryEnv: TODOIST_API_KEY
    envVars:
      - name: TODOIST_API_KEY
        required: true
        description: 用于已认证请求的 Todoist API 令牌。
      - name: TODOIST_PROJECT_ID
        required: false
        description: 用户未指定项目 ID 时使用的可选默认项目 ID。
```

### 这为何重要

ClawHub 的安全分析会检查技能声明的内容是否与其实际行为一致。如果代码引用了 `TODOIST_API_KEY`，但 frontmatter 未在 `requires.env`、`primaryEnv` 或 `envVars` 下声明它，分析将标记元数据不匹配。保持声明准确有助于技能通过审查，也有助于用户了解他们正在安装的内容。

### 示例：完整的 frontmatter

```yaml
---
name: todoist-cli
description: 从命令行管理 Todoist 任务、项目和标签。
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
        description: Todoist API 令牌。
      - name: TODOIST_PROJECT_ID
        required: false
        description: 可选的默认项目 ID。
    emoji: "\u2705"
    homepage: https://github.com/example/todoist-cli
---
```

## 技能文件

发布功能接受技能文件夹中的所有常规文件，无论扩展名为何。忽略文件、
隐藏路径、符号链接、macOS 元数据和服务器端大小限制仍然适用。

- 包含有效 UTF-8 的有限大小文件可预览为经过转义的纯文本，并会纳入
  有限文本分析。
- 其他文件会保留其精确字节，并可供下载。
- 安全扫描程序会接收完整的已存储工件；文本检测是渲染和
  分析方面的事项，而不是上传允许列表。

限制（服务器端）：

- 捆绑包总大小：50MB。
- 嵌入文本包括 `SKILL.md` + 最多约 40 个有限大小的 UTF-8 文件（尽力而为的上限）。

## Slug

- 默认从文件夹名称派生。
- 包作用域必须与 ClawHub 发布者用户名完全匹配。发布者用户名可以使用小写字母、数字、连字符、点和下划线；必须以小写字母或数字开头和结尾。
- 包 slug 必须为小写且符合 npm 安全规范，例如 `@example.tools/demo-plugin` 或 `demo-plugin`。

## 版本控制 + 标签

- 每次发布都会创建一个新版本（semver）。
- 标签是指向某个版本的字符串指针；通常使用 `latest`。

## 许可证

- ClawHub 上发布的所有技能均采用 `MIT-0` 许可。
- 任何人都可以使用、修改和重新分发已发布的技能，包括用于商业用途。
- 无需署名。
- 不要在 `SKILL.md` 中添加冲突的许可条款；ClawHub 不支持按技能覆盖许可证。

## 付费技能

- ClawHub 不支持付费技能、按技能定价、付费墙或收入分成。
- 不要向 `SKILL.md` 添加定价元数据；它不属于技能格式，也不会使已发布的技能变为付费技能。
- 如果技能与付费第三方服务集成，请在技能说明和环境变量声明中清楚说明外部费用和所需账户（必需变量使用 `requires.env`，可选变量使用 `envVars` 并设置 `required: false`）。
