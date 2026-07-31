---
read_when:
    - 你想使用 Memory Wiki CLI
    - 你正在记录或更改 `openclaw wiki`
summary: '`openclaw wiki` 的 CLI 参考（Memory Wiki 仓库状态、搜索、编译、检查、应用、桥接、ChatGPT 导入和 Obsidian 辅助工具）'
title: Wiki
x-i18n:
    generated_at: "2026-07-26T06:45:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1f793d52de270068cf3a06b13f52242bb66738235718639486e090a2de213e73
    source_path: cli/wiki.md
    workflow: 16
---

# `openclaw wiki`

检查和维护 `memory-wiki` 保险库。此功能由内置的可选 `memory-wiki` 插件提供。首次使用前请启用它：

```bash
openclaw plugins enable memory-wiki
openclaw gateway restart
```

相关内容：[Memory Wiki 插件](/zh-CN/plugins/memory-wiki)、[记忆概览](/zh-CN/concepts/memory)、[CLI：memory](/zh-CN/cli/memory)

## 常用命令

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki okf import ./knowledge-catalog/okf/bundles/ga4
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki search "who should I ask about Teams?" --mode route-question
openclaw wiki get entity.alpha --from 1 --lines 80

openclaw wiki apply synthesis "Alpha Summary" \
  --body "Short synthesis body" \
  --source-id source.alpha

openclaw wiki apply metadata entity.alpha \
  --source-id source.alpha \
  --status review \
  --question "Still active?"

openclaw wiki bridge import
openclaw wiki unsafe-local import
openclaw wiki chatgpt import --export ./chatgpt-export --dry-run
openclaw wiki chatgpt rollback <run-id>

openclaw wiki obsidian status
openclaw wiki obsidian search "alpha"
openclaw wiki obsidian open syntheses/alpha-summary.md
openclaw wiki obsidian command workspace:quick-switcher
openclaw wiki obsidian daily
```

## Agent 选择

当 `plugins.entries.memory-wiki.config.vault.scope` 为 `agent` 时，请使用顶层 `--agent <id>` 选项选择保险库：

```bash
openclaw wiki --agent support status
openclaw wiki --agent support search "refund policy"
openclaw wiki --agent marketing ingest ./campaign-notes.md
```

在配置了多个 Agent 的设置中，CLI 操作必须提供 `--agent`，以防命令读取或写入任意的默认保险库。如果只配置了一个 Agent，该 Agent 仍为默认值。未知的 Agent ID 会在保险库操作开始前导致失败。当 `vault.scope` 为 `global` 时，此选项不会更改选定的路径。

Gateway 网关客户端遵循相同规则：在 Agent 范围的多 Agent 设置中，对基于保险库的 `wiki.*` 请求传递 `agentId`。缺失或未知的 ID 会导致错误。Agent 轮次、wiki 工具、记忆语料库补充内容和已编译的提示词摘要都已携带活跃运行时 Agent 上下文。

## 命令

### `wiki status`

显示保险库模式和范围、解析出的 Agent、健康状态以及 Obsidian CLI 可用性。请先使用此命令检查目标保险库是否已初始化、桥接模式是否健康，或 Obsidian 集成是否可用。

当桥接模式处于活跃状态并配置为读取记忆工件时，此命令会查询正在运行的 Gateway 网关，因此它看到的活跃记忆插件上下文与 Agent/运行时记忆相同。

### `wiki doctor`

运行 wiki 健康检查并报告可操作的修复方案。不健康时以非零状态退出。

当桥接模式处于活跃状态并配置为读取记忆工件时，此命令会在生成报告前查询正在运行的 Gateway 网关。已禁用的桥接导入以及未读取记忆工件的桥接配置仍在本地/离线运行。

常见问题：

- 启用了桥接模式，但没有公开记忆工件
- 保险库布局无效或缺失
- 预期使用 Obsidian 模式时缺少外部 Obsidian CLI

### `wiki init`

创建 wiki 保险库布局和起始页面，包括顶层索引和缓存目录。

### `wiki ingest <path>`

将本地 Markdown 或文本文件导入 wiki 的 `sources/` 文件夹，作为来源页面。`<path>` 必须是本地文件路径；目前不支持从 URL 导入。二进制文件会被拒绝。

导入的来源页面包含溯源 frontmatter（`sourceType: local-file`、`sourcePath`、`ingestedAt`）。导入后始终会重新编译保险库。

标志：`--title <title>` 会覆盖来源标题（默认值：从文件名派生）。

### `wiki okf import <path>`

将解压后的 Open Knowledge Format 包导入 wiki 概念页面。

导入器会读取 OKF 目录树中的每个非保留 `.md` 概念文档，要求 `type` 字段非空，并将未知的 OKF `type` 值视为通用概念。保留的 OKF `index.md` 和 `log.md` 文件不会作为概念导入。

导入的页面会扁平化到 `concepts/` 下，因此现有的 wiki 编译、搜索、获取、摘要和仪表板流程可以立即看到它们。原始 OKF 概念 ID、`type`、`resource`、`tags`、时间戳、来源路径和完整 frontmatter 都会保留在页面 frontmatter 中。内部 OKF Markdown 链接会重写为生成的 wiki 页面；失效链接或外部链接保持不变。导入后始终会重新编译保险库。

示例：

```bash
openclaw wiki okf import ./bundles/ga4
openclaw wiki okf import ./bundles/ga4 --json
openclaw wiki search "BigQuery Table" --mode source-evidence --json
openclaw wiki get <path-from-json-result>
```

### `wiki compile`

重新构建索引、相关内容块、仪表板和已编译的查询/提示词快照。快照会持久化到 OpenClaw 的共享 SQLite 插件状态中，并保留在内存中以进行同步提示词投影；它不会在保险库中创建缓存文件。

如果已启用 `render.createDashboards`，编译还会刷新报告页面。

### `wiki lint`

检查保险库并写入涵盖以下内容的报告：

- 结构问题（失效链接、缺失/重复的 ID、缺少页面类型或标题、无效的 frontmatter）
- 溯源缺口（缺少来源 ID、缺少导入溯源信息）
- 矛盾（已标记的矛盾、互相冲突的声明）
- 待解决问题
- 低置信度页面和声明
- 陈旧页面和声明

对 wiki 进行有意义的更新后，请运行此命令。

### `wiki search <query>`

搜索 wiki 内容。行为取决于配置：

- `search.backend`：`shared` 或 `local`
- `search.corpus`：`wiki`、`memory` 或 `all`
- `--mode`：`auto`、`find-person`、`route-question`、`source-evidence` 或 `raw-claim`

需要 wiki 专用排序和溯源信息时，请使用 `wiki search`。如果活跃记忆插件公开共享搜索，请优先使用 `openclaw memory search` 执行一次广泛的共享召回。

搜索模式：

- `find-person`：别名、用户名、社交账号、规范 ID 和人物页面
- `route-question`：适合咨询/最适用场景提示及关系上下文
- `source-evidence`：来源页面和结构化证据字段
- `raw-claim`：带声明/证据元数据的结构化声明文本

示例：

```bash
openclaw wiki search "bgroux" --mode find-person
openclaw wiki search "who knows Teams rollout?" --mode route-question
openclaw wiki search "maintainer-whois" --mode source-evidence
openclaw wiki search "strong route Teams" --mode raw-claim --json
```

当结果与结构化声明匹配时，文本输出会包含 `Claim:` 和 `Evidence:` 行。JSON 输出还会提供 `matchedClaimId`、`matchedClaimStatus`、`matchedClaimConfidence`、`evidenceKinds` 和 `evidenceSourceIds`，供 Agent 进一步查看。

### `wiki get <lookup>`

按 ID 或相对路径读取 wiki 页面。

```bash
openclaw wiki get entity.alpha
openclaw wiki get syntheses/alpha-summary.md --from 1 --lines 80
```

### `wiki apply`

在不对页面进行自由形式修改的情况下应用范围有限的变更：

- `apply synthesis <title>`：使用托管的摘要正文创建或刷新综合页面
- `apply metadata <lookup>`：更新现有页面的元数据

两者都接受 `--source-id`、`--contradiction`、`--question`（每个均可重复）、`--confidence <n>`（0-1）和 `--status <status>`。`apply metadata` 还接受 `--clear-confidence`，用于删除已存储的置信度值。这是演进 wiki 页面的受支持方式，可确保托管的生成内容块保持完整。

### `wiki bridge import`

将活跃记忆插件中的公开记忆工件导入桥接支持的来源页面。在 `bridge` 模式下使用此命令，可将最新导出的记忆工件提取到 wiki 保险库中。

对于活跃的桥接工件读取，CLI 会通过 Gateway 网关 RPC 路由导入，因此会使用运行时记忆插件上下文。如果桥接导入已禁用或工件读取已关闭，该命令会保持本地/离线的零导入行为。导入后的索引刷新由 `ingest.autoCompile` 控制。

### `wiki unsafe-local import`

在 `unsafe-local` 模式下，从显式配置的本地路径（`unsafeLocal.paths`）导入。此功能有意设为实验性，并且仅限同一台机器。导入后的索引刷新由 `ingest.autoCompile` 控制。

### `wiki chatgpt import`

将 ChatGPT 导出内容导入 wiki 来源页面草稿。

```bash
openclaw wiki chatgpt import --export ./chatgpt-export
openclaw wiki chatgpt import --export ./conversations.json --dry-run
```

| 标志              | 默认值    | 描述                                                   |
| ----------------- | ---------- | ------------------------------------------------------------- |
| `--export <path>` | （必需） | ChatGPT 导出目录或 `conversations.json` 路径。        |
| `--dry-run`       | `false`    | 在不写入页面的情况下预览创建/更新/跳过的数量。 |

非试运行导入只要更改了任何页面，就会记录一个导入运行 ID 并在摘要中输出；回滚时需要此 ID。

### `wiki chatgpt rollback <run-id>`

回滚之前应用的 ChatGPT 导入运行，删除它创建的页面并恢复它覆盖的页面。如果该运行已经回滚，则不执行任何操作（并报告 `alreadyRolledBack`）。

### `wiki obsidian ...`

用于以 Obsidian 友好模式运行的保险库的 Obsidian 辅助命令：`status`、`search`、`open`、`command`、`daily`。当启用 `obsidian.useOfficialCli` 时，这些命令要求 `PATH` 上存在官方 `obsidian` CLI。

当 `vault.scope` 为 `agent` 时，配置验证会拒绝 `obsidian.useOfficialCli: true`，因为 `obsidian.vaultName` 是一项全局设置，而不是按 Agent 映射的设置。Obsidian 友好的 Markdown 渲染仍然可用。

## 实用使用指南

- 当溯源信息和页面标识很重要时，请使用 `wiki search` + `wiki get`。
- 请使用 `wiki apply`，而不是手动编辑托管的生成内容部分。
- 在信任矛盾或低置信度内容之前，请使用 `wiki lint`。
- 如果希望立即获得最新的仪表板和已编译摘要，请在批量导入或来源变更后使用 `wiki compile`。
- 当数据目录、文档导出或 Agent 增强管道已生成 OKF Markdown 包时，请使用 `wiki okf import`。
- 当桥接模式依赖新导出的记忆工件时，请使用 `wiki bridge import`。

## 配置关联

`openclaw wiki` 的行为受以下配置影响：

- `plugins.entries.memory-wiki.config.vaultMode`
- `plugins.entries.memory-wiki.config.vault.scope`
- `plugins.entries.memory-wiki.config.vault.path`
- `plugins.entries.memory-wiki.config.search.backend`
- `plugins.entries.memory-wiki.config.search.corpus`
- `plugins.entries.memory-wiki.config.bridge.*`
- `plugins.entries.memory-wiki.config.obsidian.*`
- `plugins.entries.memory-wiki.config.ingest.autoCompile`
- `plugins.entries.memory-wiki.config.render.*`
- `plugins.entries.memory-wiki.config.context.includeCompiledDigestPrompt`

完整配置模型请参阅 [Memory Wiki 插件](/zh-CN/plugins/memory-wiki)。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [Memory wiki](/zh-CN/plugins/memory-wiki)
