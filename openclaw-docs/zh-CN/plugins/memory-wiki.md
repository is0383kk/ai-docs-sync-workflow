---
read_when:
    - 你需要超越普通 MEMORY.md 笔记的持久化知识存储
    - 你正在配置内置的 Memory Wiki 插件
    - 你需要为同一个 Gateway 网关中的智能体分别设置独立的 wiki 仓库
    - 你想了解 wiki_search、wiki_get 或桥接模式
summary: memory-wiki：包含来源追溯、声明、仪表板和桥接模式的编译式知识库
title: Memory Wiki
x-i18n:
    generated_at: "2026-07-26T06:21:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fda3c801ae39b529a3f1fcaf8791b6dcb1d8116ba2e73e99cca62dca6c64140a
    source_path: plugins/memory-wiki.md
    workflow: 16
---

`memory-wiki` 是一个内置插件，可将持久知识编译为
可导航的 wiki：确定性页面、带证据的结构化声明、
来源信息、仪表板以及机器可读的摘要。

它不会取代主动记忆插件。召回、提升、索引和
Dreaming 仍由所配置的记忆后端
（`memory-core`、QMD、Honcho 等）负责。`memory-wiki` 与其并行运行，将
知识编译为持续维护的 wiki 层。

在使用其 CLI、工具或运行时集成之前，请先启用该插件：

```bash
openclaw plugins enable memory-wiki
openclaw gateway restart
```

| 层                   | 负责的内容                                                                          |
| -------------------- | --------------------------------------------------------------------------------- |
| 主动记忆插件         | 召回、语义搜索、提升、Dreaming、记忆运行时                                          |
| `memory-wiki`        | 编译后的 wiki 页面、富含来源信息的综合内容、仪表板、wiki 搜索/获取/应用             |

实用规则：

- 使用 `memory_search` 对已配置的所有语料库执行一次广泛召回
- 当需要 wiki 专用排序、来源信息或页面级信念结构时，使用 `wiki_search` / `wiki_get`
- 当主动记忆插件支持选择语料库时，使用 `memory_search corpus=all` 在一次调用中涵盖两个层

一种常见的本地优先设置：使用 QMD 作为负责召回的主动记忆后端，并以
`bridge` 模式运行 `memory-wiki`，用于生成持久的综合页面。请参阅
[配置](#configuration)下的 QMD + 桥接模式示例。

如果桥接模式报告导出的工件数量为零，则主动记忆插件
当前未公开公共桥接输入。请先运行 `openclaw wiki doctor`，
然后确认主动记忆插件支持公共工件。

## 仓库模式

- `isolated`（默认）：拥有独立的仓库和来源，不依赖主动记忆插件。适用于自包含的精选知识存储。
- `bridge`：通过公共插件 SDK 接口，从主动记忆插件读取公共记忆工件和事件日志。用于编译记忆插件导出的工件，而无需访问插件的私有内部机制。
- `unsafe-local`：针对本机私有路径的显式逃生通道。此模式有意保持实验性且不可移植；仅在理解信任边界，并且确实需要桥接模式无法提供的本地文件系统访问权限时使用。

仓库模式和仓库作用域是两个独立的选择：

- `vaultMode` 选择 wiki 输入的来源。
- `vault.scope` 选择所有智能体共用一个仓库，还是每个智能体拥有一个子仓库。

`vault.scope: "global"` 是默认值，并保留现有的单仓库
行为。当智能体之间不得共享 wiki 页面、编译摘要、搜索结果或写入内容时，
请将 `vault.scope: "agent"` 与 `isolated` 或 `bridge` 模式结合使用。
智能体作用域不能与 `unsafe-local` 模式结合使用，因为这些已配置的
私有路径并非由智能体拥有的输入。配置验证会拒绝此
组合。

根据 `bridge.*` 配置开关，桥接模式可以索引：

- 导出的记忆工件（`indexMemoryRoot`）
- 每日笔记（`indexDailyNotes`）
- Dreaming 报告（`indexDreamReports`）
- 记忆事件日志（`followMemoryEvents`）

当桥接模式处于活动状态且已启用 `bridge.readMemoryArtifacts` 时，
`openclaw wiki status`、`openclaw wiki doctor` 和 `openclaw wiki bridge
import` 会通过正在运行的 Gateway 网关路由，因此它们看到的主动记忆
插件上下文与智能体/运行时记忆相同。如果桥接已禁用或工件
读取已关闭，这些命令会继续保持本地/离线行为。

## 仓库布局

```text
<vault>/
  AGENTS.md
  WIKI.md
  index.md
  inbox.md
  entities/
  concepts/
  syntheses/
  sources/
  reports/
  _attachments/
  _views/
  .openclaw-wiki/
```

托管内容保留在生成的块中；人工笔记块在
重新生成后仍会保留。

- `sources/`：导入的原始材料，以及由桥接/不安全本地模式支持的页面
- `entities/`：持久存在的事物、人员、系统、项目和对象
- `concepts/`：思想、抽象概念、模式和策略（也是 OKF 导入内容的存放位置）
- `syntheses/`：编译后的摘要和持续维护的汇总
- `reports/`：生成的仪表板

## Open Knowledge Format 导入

```bash
openclaw wiki okf import ./bundles/ga4
```

将已解包的 Open Knowledge Format 包导入 wiki 概念页面。当数据目录、文档爬虫或增强智能体已
生成 OKF 时，此方式非常合适：将 OKF 保留为可移植的交换工件，并让 `memory-wiki`
将其转换为 OpenClaw 原生概念页面和编译摘要。

- 非保留的 `.md` 文件是概念文档
- 每个导入的概念都必须包含非空的 `type` frontmatter 字段；缺少 `type` 会产生 `missing-type` 警告，并跳过该文件
- 未知的 `type` 值会作为通用概念接受
- `index.md` 和 `log.md` 是保留项，绝不会作为概念导入
- 损坏或外部的 Markdown 链接保持不变

导入的页面会平铺到 `concepts/` 下，因此现有的编译、搜索、获取和
仪表板流程无需第二棵 wiki 树即可看到它们。每个页面都会保留
原始 OKF 概念 ID、源路径、`type`、`resource`、`tags`、时间戳
以及完整的生成方 frontmatter。内部 OKF 链接会重写为生成的
wiki 概念页面，并同时发出包含
`kind: okf-link` 的结构化 `relationships` 条目。

## 结构化声明和证据

页面携带结构化的 `claims` frontmatter，而不只是自由格式文本。每条
声明可以包含 `id`、`text`、`status`、`confidence`、`evidence[]` 和
`updatedAt`。每个证据条目可以包含 `kind`、`sourceId`、`path`、
`lines`、`weight`、`confidence`、`privacyTier`、`note` 和 `updatedAt`。

这使 wiki 表现为信念层，而不是被动的笔记堆积。
声明可以被跟踪、评分、质疑，并追溯到来源进行解决。

## 面向智能体的实体元数据

实体页面携带通用路由元数据，可用于人员、团队、
系统、项目或任何其他实体类型：

- `entityType`：例如 `person`、`team`、`system`、`project`
- `canonicalId`：跨别名和导入保持稳定的身份键
- `aliases`：解析到同一页面的名称、账号名或标签
- `privacyTier`：自由格式字符串；`public` 被视为无需审查，任何其他值（例如 `local-private`、`sensitive`、`confirm-before-use`）都会在 `reports/privacy-review.md` 中标记
- `bestUsedFor` / `notEnoughFor`：紧凑的路由提示
- `lastRefreshedAt`：来源刷新时间戳，与页面编辑时间分开记录
- `personCard`：可选的人员专用路由卡片（账号名、社交资料、电子邮件、时区、负责领域、适合询问的事项、应避免询问的事项、置信度、隐私层级）
- `relationships`：指向相关页面的类型化边（目标、类型、权重、置信度、证据类型、隐私层级、备注）

对于人员 wiki，请从 `reports/person-agent-directory.md` 开始，然后在使用联系方式或推断
事实之前，通过 `wiki_get` 打开人员页面。

<Accordion title="实体页面示例">
```yaml
pageType: entity
entityType: person
id: entity.example-person
canonicalId: maintainer.example-person
aliases:
  - Alex
  - example-handle
privacyTier: local-private
bestUsedFor:
  - 示例生态系统路由
notEnoughFor:
  - 法律审批
lastRefreshedAt: "2026-04-29T00:00:00.000Z"
personCard:
  handles:
    - "@example-handle"
  socials:
    - "https://x.example/example-handle"
  emails:
    - alex@example.com
  timezone: America/Chicago
  lane: 示例生态系统
  askFor:
    - 示例发布问题
  avoidAskingFor:
    - 不相关的计费决策
  confidence: 0.8
  privacyTier: confirm-before-use
relationships:
  - targetId: entity.other-person
    targetTitle: 其他人员
    kind: collaborates-with
    confidence: 0.7
    evidenceKind: discrawl-stat
claims:
  - id: claim.example.routing
    text: Alex 适合处理示例生态系统路由。
    status: supported
    confidence: 0.9
    evidence:
      - kind: maintainer-whois
        sourceId: source.maintainers
        privacyTier: local-private
```
</Accordion>

## 编译流水线

编译过程会读取 wiki 页面、规范化摘要，并将面向机器的
快照持久化到 OpenClaw 的共享 SQLite 插件状态中。运行时代码使用
由生命周期管理的所有者快照，在异步提示词准备期间加载 SQLite；
同步提示词组装绝不会抓取 Markdown 或读取缓存文件。
编译后的输出还支持搜索/获取的第一阶段 wiki 索引、将声明 ID
查找回其所属页面、紧凑的提示词补充以及报告
生成。

来源编辑和仓库恢复只有在下次
编译后才会对机器可见。重启或刷新插件生命周期时，会将仓库中
以因果链连接的编译发布与 SQLite 进行比较，并拒绝来自
更新但已回滚状态的快照。在回滚前启动的编译器无法
基于恢复后的前序状态发布。提示词准备不会轮询
仓库，也不会安装文件监视器。
进入回滚隔离状态后，在运行进程中执行编译会立即清除所有者；
单独的编译器进程则需要刷新插件生命周期，以便
守护进程确认新的持久发布。
编译缓存可以重建：发布周期之前的缓存行会被
视为未命中，并由下次编译替换；它们不会被迁移。

## 仪表板和健康报告

启用 `render.createDashboards` 后，编译过程会在
`reports/` 下维护仪表板：

| 报告                                | 跟踪内容                                           |
| ----------------------------------- | -------------------------------------------------- |
| `reports/open-questions.md`         | 包含未解决问题的页面                               |
| `reports/contradictions.md`         | 矛盾备注聚类                                       |
| `reports/low-confidence.md`         | 低置信度页面和声明                                 |
| `reports/claim-health.md`           | 缺少结构化证据的声明                               |
| `reports/stale-pages.md`            | 已过期或新鲜度未知的内容                           |
| `reports/person-agent-directory.md` | 人员/实体路由卡片                                  |
| `reports/relationship-graph.md`     | 结构化关系边                                       |
| `reports/provenance-coverage.md`    | 证据类别覆盖情况                                   |
| `reports/privacy-review.md`         | 使用前需要审查的非公开隐私层级                     |

## 搜索和检索

两个搜索后端：

- `shared`：可用时使用共享记忆搜索流程
- `local`：在本地搜索 wiki

三个语料库：`wiki`、`memory`、`all`。

- `wiki_search` / `wiki_get` 会尽可能使用编译摘要作为第一阶段
- 声明 ID 会解析回其所属页面
- 有争议/已过期/新鲜的声明会影响排序
- 来源标签会保留到结果中

搜索模式（`--mode` / 工具 `mode` 参数）：

| 模式              | 增强                                                         |
| ----------------- | -------------------------------------------------------------- |
| `auto`            | 均衡的默认模式                                               |
| `find-person`     | 类人物实体、别名、账号名、社交账号、规范 ID |
| `route-question`  | 智能体卡片、适用问题/最佳用途提示、关系上下文 |
| `source-evidence` | 来源页面和结构化证据元数据                  |
| `raw-claim`       | 匹配结构化声明；返回声明/证据元数据    |

当结果与结构化声明匹配时，`wiki_search` 会在其详情载荷中返回
`matchedClaimId`、`matchedClaimStatus`、`matchedClaimConfidence`、
`evidenceKinds` 和 `evidenceSourceIds`。文本输出在可用时
包含紧凑的 `Claim:` 和 `Evidence:` 行。

## 智能体工具

| 工具          | 用途                                                                                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `wiki_status` | 当前知识库模式和范围、解析后的智能体、健康状态、Obsidian CLI 可用性                                                                               |
| `wiki_search` | 搜索 wiki 页面，并在配置后搜索共享记忆语料库；接受 `mode`，用于人物查找、问题路由、来源证据或原始声明深入查询 |
| `wiki_get`    | 按 ID/路径读取 wiki 页面；启用共享搜索且查找未命中时，回退到共享记忆语料库                                     |
| `wiki_apply`  | 执行范围明确的综合分析/元数据变更，不进行自由形式的页面修改                                                                                             |
| `wiki_lint`   | 结构检查、来源缺口、矛盾、待解决问题                                                                                            |

该插件还会注册非独占的记忆语料库补充源，因此当活动记忆
插件支持语料库选择时，共享的 `memory_search` 和 `memory_get`
可以访问 wiki。

## 提示词和上下文行为

启用 `context.includeCompiledDigestPrompt` 后，记忆提示词区段会
附加来自插件状态的紧凑编译快照：仅包含排名靠前的页面、
排名靠前的声明、矛盾数量、问题数量、置信度/新鲜度
限定信息。此功能为可选，因为它会改变提示词结构；它主要适用于
明确使用记忆补充内容的上下文引擎或提示词组装流程。

## 配置

将配置放在 `plugins.entries.memory-wiki.config` 下：

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "isolated",
          vault: {
            scope: "global",
            path: "~/.openclaw/wiki/main",
            renderMode: "obsidian",
          },
          obsidian: {
            enabled: true,
            useOfficialCli: true,
            vaultName: "OpenClaw Wiki",
            openAfterWrites: false,
          },
          bridge: {
            enabled: false,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          unsafeLocal: {
            allowPrivateMemoryCoreAccess: false,
            paths: [],
          },
          ingest: {
            autoCompile: true,
            maxConcurrentJobs: 1,
            allowUrlIngest: true,
          },
          search: {
            backend: "shared",
            corpus: "wiki",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
          render: {
            preserveHumanBlocks: true,
            createBacklinks: true,
            createDashboards: true,
          },
        },
      },
    },
  },
}
```

关键开关：

| 键                                        | 值/默认值                               | 说明                                                                         |
| ------------------------------------------ | ---------------------------------------------- | ----------------------------------------------------------------------------- |
| `vaultMode`                                | `isolated`（默认）、`bridge`、`unsafe-local` | 选择输入和集成行为                                        |
| `vault.scope`                              | `global`（默认）、`agent`                    | 使用一个共享知识库，或每个智能体使用一个子知识库                                 |
| `vault.path`                               | 全局默认值 `~/.openclaw/wiki/main`         | 全局范围下为具体知识库；智能体范围下的父目录默认为 `~/.openclaw/wiki`       |
| `vault.renderMode`                         | `native`（默认）、`obsidian`                 |                                                                               |
| `bridge.readMemoryArtifacts`               | 默认值 `true`                                 | 导入活动记忆插件的公共工件                                  |
| `bridge.followMemoryEvents`                | 默认值 `true`                                 | 在桥接模式中包含事件日志                                             |
| `unsafeLocal.allowPrivateMemoryCoreAccess` | 默认值 `false`                                | 运行 `unsafe-local` 导入时必需                                        |
| `unsafeLocal.paths`                        | 默认值 `[]`                                   | 在 `unsafe-local` 模式中要导入的显式本地路径                         |
| `search.backend`                           | `shared`（默认）、`local`                    |                                                                               |
| `search.corpus`                            | `wiki`（默认）、`memory`、`all`              |                                                                               |
| `context.includeCompiledDigestPrompt`      | 默认值 `false`                                | 将所选智能体的紧凑摘要快照附加到记忆提示词区段 |
| `render.createBacklinks`                   | 默认值 `true`                                 | 生成确定性的相关内容块                                         |
| `render.createDashboards`                  | 默认值 `true`                                 | 生成仪表板页面                                                      |

### 每智能体知识库

将 `vault.scope` 设为 `agent`，可为每个已配置的智能体提供独立的 wiki。
在此范围内，`vault.path` 是父目录，OpenClaw 会附加
规范化后的智能体 ID：

```json5
{
  agents: {
    list: [{ id: "support" }, { id: "marketing" }],
  },
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "bridge",
          vault: {
            scope: "agent",
            path: "~/.openclaw/wiki",
          },
          bridge: {
            enabled: true,
            readMemoryArtifacts: true,
          },
        },
      },
    },
  },
}
```

这会解析为 `~/.openclaw/wiki/support` 和
`~/.openclaw/wiki/marketing`。如果在智能体范围内省略 `vault.path`，
父目录默认为 `~/.openclaw/wiki`。因此，默认的 `main` 智能体会继续使用
现有的 `~/.openclaw/wiki/main` 路径。

智能体工具、编译后的提示词摘要，以及通过
`memory_search` / `memory_get` 公开的 wiki 补充内容，都会根据活动智能体上下文解析知识库。
在配置了多个智能体的环境中进行 CLI 和 Gateway 网关调用时，请通过
`openclaw wiki --agent <agentId> ...` 或 Gateway 网关请求的
`agentId` 明确指定智能体。仅配置一个智能体时，如果未提供 ID，
该智能体仍为默认值。

在桥接模式下，仅当公共记忆工件的
`agentIds` 包含所选智能体时，智能体范围的导入才会接受该工件。属于其他智能体、
没有所有权元数据或所有者未知的工件都会被跳过。全局范围
继续沿用现有的共享工件行为。

<Warning>
更改 `vault.scope` 不会复制或拆分现有知识库。在智能体范围内，
显式配置的 `vault.path` 会成为父目录，因此在切换生产环境中的智能体之前，
请有计划地移动或导入现有页面。请先备份
知识库。

每智能体知识库是同一进程内的知识边界，而不是操作系统级
安全边界。具备主机文件系统访问权限的插件和非沙箱隔离工具仍可
读取其他智能体的目录。当智能体之间互不信任时，请使用[沙箱隔离](/zh-CN/gateway/sandboxing)或
[独立的 Gateway 配置文件](/zh-CN/gateway/multiple-gateways)。
</Warning>

### 示例：QMD + 桥接模式

如果希望使用 QMD 进行回忆，并使用 `memory-wiki` 维护
知识层，请采用此配置。每一层各司其职：QMD 让原始笔记、会话
导出内容和额外集合保持可搜索，而 `memory-wiki` 则编译
稳定的实体、声明、仪表板和来源页面。

```json5
{
  memory: {
    backend: "qmd",
  },
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "bridge",
          bridge: {
            enabled: true,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          search: {
            backend: "shared",
            corpus: "all",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
        },
      },
    },
  },
}
```

这样可让 QMD 负责主动记忆的回忆，让 `memory-wiki` 专注于
编译页面和仪表板，并在你主动启用编译摘要提示词之前
保持提示词结构不变。

## CLI

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki get entity.alpha
openclaw wiki apply synthesis "Alpha Summary" --body "..." --source-id source.alpha
openclaw wiki bridge import
openclaw wiki obsidian status
```

有关完整命令参考，请参阅 [CLI：wiki](/zh-CN/cli/wiki)，其中包括
`wiki okf import`、`wiki apply metadata`、`wiki unsafe-local import`、
`wiki chatgpt import` / `wiki chatgpt rollback`，以及完整的 `wiki obsidian`
子命令集。

## Obsidian 支持

当 `vault.renderMode` 为 `obsidian` 时，插件会写入适合 Obsidian 的
Markdown，并可选择使用官方 `obsidian` CLI 执行状态
探测、知识库搜索、打开页面、调用命令以及跳转到
每日笔记。此功能为可选；没有 Obsidian 时，wiki 仍可在原生模式下
运行。

智能体范围的知识库仍可使用适合 Obsidian 的 Markdown，但配置
验证会拒绝 `obsidian.useOfficialCli: true` 与 `vault.scope: "agent"` 的组合。
当前的 `obsidian.vaultName` 设置是全局性的，无法为每个智能体选择不同的
Obsidian 知识库。请改用 wiki 工具和 CLI 操作，
或将由 Obsidian 操作的 wiki 保持在全局范围内。

## 推荐工作流

<Steps>
<Step title="保留用于回忆的主动记忆插件">
回忆、提升和 Dreaming 仍由配置的记忆后端负责。
</Step>
<Step title="启用 memory-wiki">
除非明确希望使用桥接模式，否则请从 `isolated` 模式开始。
</Step>
<Step title="当溯源信息很重要时使用 wiki_search / wiki_get">
如果需要 wiki 专用排序或页面级信念结构，请优先使用这些工具，而不是 `memory_search`。
</Step>
<Step title="使用 wiki_apply 进行小范围综合或元数据更新">
避免手动编辑托管的生成块。
</Step>
<Step title="在进行实质性更改后运行 wiki_lint">
可发现矛盾、未解决的问题和溯源信息缺口。
</Step>
<Step title="启用仪表板以查看过期内容和矛盾">
设置为 `render.createDashboards: true`（默认值）。
</Step>
</Steps>

## 相关文档

- [记忆概览](/zh-CN/concepts/memory)
- [CLI：记忆](/zh-CN/cli/memory)
- [CLI：wiki](/zh-CN/cli/wiki)
- [插件 SDK 概览](/zh-CN/plugins/sdk-overview)
