---
read_when:
    - 你想要为语义记忆建立索引或进行搜索
    - 你正在调试记忆可用性或索引问题
    - 你希望将回忆起的短期记忆提升为 `MEMORY.md`
summary: '`openclaw memory` 的 CLI 参考（status/index/search/promote/promote-explain/rem-harness/rem-backfill）'
title: 记忆
x-i18n:
    generated_at: "2026-07-26T06:11:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6354745f8622ee80345325fa6f3e7d6c5f280cb63b9cdb100a766cf9e300af59
    source_path: cli/memory.md
    workflow: 16
---

# `openclaw memory`

管理语义记忆的索引、搜索以及提升至 `MEMORY.md`。
此功能由内置的 `memory-core` 插件提供，在
`plugins.slots.memory` 选择 `memory-core`（默认值）时可用。其他记忆
插件提供各自的 CLI 命名空间。

相关内容：[记忆](/zh-CN/concepts/memory)概念、[Dreaming](/zh-CN/concepts/dreaming)、
[记忆配置参考](/zh-CN/reference/memory-config)、[Memory Wiki](/zh-CN/plugins/memory-wiki)、
[wiki](/zh-CN/cli/wiki)、[插件](/zh-CN/tools/plugin)。

## `memory status`

```bash
openclaw memory status [--agent <id>] [--deep] [--index] [--fix] [--json] [--verbose]
```

未指定 `--agent` 时，将针对 `agents.entries` 中的每个智能体运行；如果未配置智能体列表，
则回退到默认智能体。

| 标志        | 效果                                                                                                                                                                                                                                                                                                    |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--deep`    | 探测向量存储、嵌入提供商和语义搜索的就绪状态（会产生额外的提供商调用）。普通的 `memory status` 会保持快速运行并跳过此操作；向量/语义状态未知表示未进行探测。即使指定 `--deep`，QMD 词法 `searchMode: "search"` 也始终跳过语义向量探测。 |
| `--index`   | 如果存储处于脏状态，则重新索引。隐含启用 `--deep`。                                                                                                                                                                                                                                                          |
| `--fix`     | 修复过期的召回锁并规范化提升元数据。                                                                                                                                                                                                                                               |
| `--json`    | 输出 JSON。                                                                                                                                                                                                                                                                                               |
| `--verbose` | 输出各阶段的详细日志。                                                                                                                                                                                                                                                                             |

如果即使指定 `dreaming.enabled: true`，`Dreaming` 行仍保持为 `off`，或者
定时扫描似乎从未运行，则托管式 Dreaming 定时任务依赖
默认智能体的 Heartbeat 触发后才能开始协调。调度详情请参阅
[Dreaming](/zh-CN/concepts/dreaming)。

状态还会列出 `memory.search.extraPaths` 中的所有额外搜索路径。

## `memory index`

```bash
openclaw memory index [--agent <id>] [--force] [--verbose]
```

每个智能体的作用域与 `status` 相同。`--force` 执行完整重新索引，而非
增量索引。`--verbose` 会先输出各智能体的提供商、模型、来源和
额外路径详情，然后显示索引进度。

## `memory search`

```bash
openclaw memory search [query] [--query <text>] [--agent <id>] [--max-results <n>] [--min-score <n>] [--json]
```

- 查询：位置参数 `[query]` 或 `--query <text>`。如果两者均已设置，则以 `--query`
  为准。如果两者均未设置，命令将报错。
- `--agent <id>`：默认为默认智能体（而非完整智能体列表）。
- `--max-results <n>`：限制结果数量（正整数）。
- `--min-score <n>`：过滤掉分数低于此值的匹配项。

## `memory promote`

对 `memory/YYYY-MM-DD.md` 中的短期候选项进行排名，并可选择将
排名靠前的条目追加到 `MEMORY.md`。

```bash
openclaw memory promote [--agent <id>] [--limit <n>] [--min-score <n>] \
  [--min-recall-count <n>] [--min-unique-queries <n>] [--apply] [--include-promoted] [--json]
```

| 标志                       | 默认值      | 效果                                                            |
| -------------------------- | ------------ | ----------------------------------------------------------------- |
| `--limit <n>`              |              | 返回/应用的最大候选项数量。                                   |
| `--min-score <n>`          | `0.75`       | 最低加权提升分数。                                 |
| `--min-recall-count <n>`   | `3`          | 所需的最低召回次数。                                    |
| `--min-unique-queries <n>` | `2`          | 所需的最低不同查询数量。                            |
| `--apply`                  | 仅预览 | 将选中的候选项追加到 `MEMORY.md`，并将其标记为已提升。 |
| `--include-promoted`       |              | 包含已在先前周期中提升的候选项。           |
| `--json`                   |              | 输出 JSON。                                                       |

这些 CLI 默认值与定时 Dreaming 扫描的深度阶段
阈值不同（请参阅下方的 [Dreaming](#dreaming)）；如需进行与
扫描行为一致的一次性手动运行，请显式传递标志。

排名信号包括：召回频率、检索相关性、查询多样性、
时间新近度、跨天整合以及派生概念的丰富度；这些信号
来自记忆召回和每日摄取流程，并会针对 Dreaming 期间重复回顾的内容
获得轻度/REM 阶段的强化加成。写入前，提升流程会重新读取当前每日笔记，
因此会遵循排名后对短期片段所做的编辑或删除，而不会从过期快照中提升内容。

## `memory promote-explain`

解释单个提升候选项的分数明细。

```bash
openclaw memory promote-explain <selector> [--agent <id>] [--include-promoted] [--json]
```

`<selector>` 可匹配候选项的键（精确匹配或子字符串）、路径或片段
文本。

## `memory rem-harness`

预览 REM 反思、候选事实和深度阶段的提升输出，
而不写入任何内容。

```bash
openclaw memory rem-harness [--agent <id>] [--path <file-or-dir>] [--grounded] [--include-promoted] [--json]
```

- `--path <file-or-dir>`：使用历史 `YYYY-MM-DD.md`
  每日文件为测试工具提供初始数据，而非使用当前工作区。
- `--grounded`：还根据历史笔记渲染有事实依据的 `What Happened` / `Reflections` /
  `Possible Lasting Updates` 预览。

## `memory rem-backfill`

将有事实依据的历史 REM 摘要写入 `DREAMS.md`，供 UI 审查。
此操作可撤销。

```bash
openclaw memory rem-backfill --path <file-or-dir> [--agent <id>] [--stage-short-term] [--json]
openclaw memory rem-backfill --rollback [--rollback-short-term] [--json]
```

- `--path <file-or-dir>`：除非设置了 `--rollback`/`--rollback-short-term`，
  否则此项为必需。用于回填的历史每日记忆文件或目录。
- `--stage-short-term`：还会将有事实依据的持久候选项植入当前
  短期提升存储，以便正常的深度阶段对其进行排名。
- `--rollback`：从
  `DREAMS.md` 中移除之前写入的、有事实依据的日记条目。
- `--rollback-short-term`：移除之前暂存的、有事实依据的短期
  候选项。

## Dreaming

Dreaming 是后台记忆整合系统，包含三个协作
阶段，并按同一调度依次运行：**轻度**（整理/暂存短期
材料）、**REM**（反思并呈现主题）、**深度**（将持久
事实提升至 `MEMORY.md`）。只有深度阶段会写入 `MEMORY.md`。

- 通过 `plugins.entries.memory-core.config.dreaming.enabled: true` 启用
  （默认值为 `false`）；`memory-core` 会自动管理扫描定时任务，无需手动
  `openclaw cron add`。
- 在聊天中使用 `/dreaming on|off` 切换；使用 `/dreaming status`
  （或 `/dreaming`/`/dreaming help`）检查。`on`/`off` 需要渠道所有者身份
  或 Gateway 网关 `operator.admin`；任何能够调用该命令的人仍可使用 `status` 和帮助。
- 人类可读的阶段输出将写入 `DREAMS.md`（或现有的 `dreams.md`）。
  默认情况下（`dreaming.storage.mode: "separate"`），每个阶段还会将
  独立报告写入 `memory/dreaming/<phase>/YYYY-MM-DD.md`；设置 `mode:
"inline"` 可改为将报告合并到每日记忆文件中，设置 `"both"`
  则同时写入两者。
- 定时运行和手动 `memory promote` 运行使用相同的深度阶段
  排名信号；只有默认阈值不同（请对比上表与
  下方的定时运行默认值）。
- 定时运行会分发至每个已配置智能体的记忆工作区。

定时运行默认值（`plugins.entries.memory-core.config.dreaming`）：

| 键                                    | 默认值     |
| -------------------------------------- | ----------- |
| `frequency`                            | `0 3 * * *` |
| `phases.deep.minScore`                 | `0.8`       |
| `phases.deep.minRecallCount`           | `3`         |
| `phases.deep.minUniqueQueries`         | `3`         |
| `phases.deep.recencyHalfLifeDays`      | `14`        |
| `phases.deep.maxAgeDays`               | `30`        |
| `phases.deep.maxPromotedSnippetTokens` | `160`       |

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

完整键列表和阶段详情：[Dreaming](/zh-CN/concepts/dreaming)、
[记忆配置参考](/zh-CN/reference/memory-config#dreaming)。

## SecretRef Gateway 网关依赖

如果主动记忆远程 API 密钥字段配置为 SecretRef，`memory`
命令会从当前 Gateway 网关快照中解析这些字段；如果 Gateway 网关
不可用，命令会快速失败。这需要 Gateway 网关支持
`secrets.resolve` 方法；较旧的 Gateway 网关会返回未知方法错误。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [记忆概览](/zh-CN/concepts/memory)
