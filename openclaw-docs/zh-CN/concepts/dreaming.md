---
read_when:
    - 你希望自动运行记忆提升
    - 你想了解每个 Dreaming 阶段的作用
    - 你希望调整整合机制，而不污染 MEMORY.md
sidebarTitle: Dreaming
summary: 采用浅层、深层和 REM 阶段的后台记忆整合，以及梦境日记
title: Dreaming
x-i18n:
    generated_at: "2026-07-26T05:46:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 501ab42cfdfa0216c308896aa8c1719b06b49d64a62afdb004e097102a376eac
    source_path: concepts/dreaming.md
    workflow: 16
---

Dreaming 是 `memory-core` 中的后台记忆整合系统。它将强烈的短期信号转化为持久记忆，同时确保整个过程可解释、可审查。

<Note>
Dreaming 是**选择启用**的，默认处于禁用状态。
</Note>

## Dreaming 写入的内容

- `memory/.dreams/` 中的**机器状态**（召回存储、阶段信号、摄取检查点、锁）。
- `DREAMS.md`（或现有的 `dreams.md`）中的**人类可读输出**，以及 `memory/dreaming/<phase>/YYYY-MM-DD.md` 下的可选阶段报告文件。

长期提升仍然只写入 `MEMORY.md`。

## 阶段模型

Dreaming 每次扫描按顺序运行三个协作阶段：light -> REM -> deep。这些是内部实现阶段，而不是由用户分别配置的模式。

| 阶段 | 用途                                   | 持久写入          |
| ----- | ----------------------------------------- | ----------------- |
| Light | 对近期短期材料进行排序和暂存 | 否                |
| REM   | 反思主题和反复出现的想法     | 否                |
| Deep  | 对持久候选项评分并进行提升      | 是（`MEMORY.md`） |

<AccordionGroup>
  <Accordion title="Light 阶段">
    - 读取近期短期召回状态、每日记忆文件，以及可用时经过脱敏处理的会话转录。
    - 对信号进行去重并暂存候选行。
    - 当存储包含内联输出时，写入一个托管的 `## Light Sleep` 块。
    - 记录增强信号，供之后的深度排序使用。
    - 绝不写入 `MEMORY.md`。

  </Accordion>
  <Accordion title="REM 阶段">
    - 根据近期短期轨迹构建主题和反思摘要。
    - 当存储包含内联输出时，写入一个托管的 `## REM Sleep` 块。
    - 记录供深度排序使用的 REM 增强信号。
    - 绝不写入 `MEMORY.md`。

  </Accordion>
  <Accordion title="Deep 阶段">
    - 使用加权评分和阈值门控对候选项进行排序（`minScore`、`minRecallCount`、`minUniqueQueries` 必须全部通过）。
    - 在写入前从实时每日文件中重新载入片段，因此会跳过陈旧或已删除的片段。
    - 将提升后的条目追加到 `MEMORY.md`。
    - 将 `## Deep Sleep` 摘要写入 `DREAMS.md`，并可选择写入 `memory/dreaming/deep/YYYY-MM-DD.md`。

  </Accordion>
</AccordionGroup>

## 会话转录摄取

Dreaming 可以将经过脱敏处理的会话转录摄取到 Dreaming 语料库中。可用时，转录会与每日记忆信号和召回轨迹一起输入 Light 阶段。个人和敏感内容会在摄取前进行脱敏处理。

## 梦境日记

Dreaming 在 `DREAMS.md` 中维护叙事性的**梦境日记**。每个阶段积累足够材料后，`memory-core` 会尽力在后台运行一次子智能体轮次并追加一则简短的日记条目；除非配置了 `dreaming.model`，否则使用默认运行时模型。如果配置的模型不可用，日记运行会使用会话默认模型重试一次；信任或允许列表失败不会重试，而会保留在日志中清晰可见，不会静默回退到通用日记条目。

<Note>
日记供用户在 Dreams UI 中阅读，不是提升来源。日记和报告工件不会参与短期提升；只有基于实际依据的记忆片段才有资格提升到 `MEMORY.md`。
</Note>

此外，还有一个基于实际依据的历史回填通道，用于审查和恢复工作：

<AccordionGroup>
  <Accordion title="回填命令">
    - `memory rem-harness --path ... --grounded` 预览根据历史 `YYYY-MM-DD.md` 笔记生成的有据日记输出。
    - `memory rem-backfill --path ...` 将可撤销的有据日记条目写入 `DREAMS.md`。
    - `memory rem-backfill --path ... --stage-short-term` 将有据持久候选项暂存到普通 Deep 阶段所使用的同一短期证据存储中。
    - `memory rem-backfill --rollback` 和 `--rollback-short-term` 会移除这些暂存的回填工件，而不触及普通日记条目或实时短期召回。

  </Accordion>
</AccordionGroup>

Control UI 在智能体的 Memory 选项卡（Agents 页面）中提供相同的日记回填和重置流程，以便在决定有据候选项是否值得提升之前，先在梦境场景中检查结果。独立的有据 Scene 通道会显示哪些暂存的短期条目来自历史重放、哪些已提升项目由有据内容主导，并允许只清除仅有据的暂存条目，而不触及实时短期状态。

## 深度排序信号

深度排序使用六个加权基础信号以及阶段增强：

| 信号              | 权重 | 描述                                       |
| ------------------- | ------ | ------------------------------------------------- |
| 相关性           | 0.30   | 条目的平均检索质量           |
| 频率           | 0.24   | 条目累计的短期信号数量 |
| 查询多样性     | 0.15   | 使其浮现的不同查询/日期上下文      |
| 时效性             | 0.15   | 按时间衰减的新鲜度评分                      |
| 整合程度       | 0.10   | 多日重复出现的强度                     |
| 概念丰富度 | 0.06   | 片段/路径中的概念标签密度             |

Light 和 REM 阶段的命中会添加一个来自 `memory/.dreams/phase-signals.json`、随时间衰减的小幅增强。

在进行任何持久写入之前，影子试验结果可以叠加到基础分数之上，作为审查信号：有帮助的试验会为候选项提供小幅且有上限的增强，中性的试验会使其保持推迟状态，有害的试验则会将其标记为在该次评分中被拒绝。此信号仅用于报告——它可以改变候选项顺序或审查元数据，但绝不会写入 `MEMORY.md`，也不会自行提升候选项。

### QA 影子试验报告覆盖范围

QA Lab 包含一个仅生成报告的场景，用于探索未来的 Dreaming 影子试验可以如何在提升前审查候选记忆：智能体将基准答案与可使用候选记忆的答案进行比较，然后写入包含结论、原因和风险标志的本地报告。此覆盖范围仅限于 QA——它会验证报告工件与 `MEMORY.md` 保持分离，并且智能体绝不会声称候选项已被提升。它不会添加生产环境影子试验行为，也不会更改 Deep 阶段提升引擎。

`memory-core` 影子试验运行器为需要稳定工件的代码路径保持相同的仅报告契约。它接受候选项、试验提示词、基准结果、候选结果、结论、原因、风险标志和证据引用，然后使用 `promotion action: report-only` 写入报告。有帮助的结论映射到 `promote` 建议，中性的结论映射到 `defer`，有害的结论映射到 `reject`——这些操作均不会写入 `MEMORY.md`，也不会应用 Deep 阶段提升。

## 调度

启用后，`memory-core` 会自动管理一个用于完整 Dreaming 扫描的定时任务，并在主运行时工作区和任何已配置的 Agent 工作区之间进行去重，因此子智能体工作区的扇出不会排除主智能体的 `DREAMS.md` 和记忆状态。

| 设置              | 默认值       |
| -------------------- | ------------- |
| `dreaming.frequency` | `0 3 * * *`   |
| `dreaming.model`     | 默认模型 |

## 快速开始

<Tabs>
  <Tab title="启用 Dreaming">
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
  </Tab>
  <Tab title="自定义扫描频率">
    ```json
    {
      "plugins": {
        "entries": {
          "memory-core": {
            "config": {
              "dreaming": {
                "enabled": true,
                "timezone": "America/Los_Angeles",
                "frequency": "0 */6 * * *"
              }
            }
          }
        }
      }
    }
    ```
  </Tab>
</Tabs>

## 斜杠命令

```text
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

对于渠道调用方，`/dreaming on` 和 `/dreaming off` 要求所有者状态；对于 Gateway 网关客户端，则要求 `operator.admin`。`/dreaming status` 和 `/dreaming help` 为只读操作。

## CLI 工作流

<Tabs>
  <Tab title="提升预览/应用">
    ```bash
    openclaw memory promote
    openclaw memory promote --apply
    openclaw memory promote --limit 5
    openclaw memory status --deep
    ```

    除非通过 CLI 标志覆盖，否则手动 `memory promote` 默认使用 Deep 阶段阈值。

  </Tab>
  <Tab title="解释提升">
    解释特定候选项为何会或不会被提升：

    ```bash
    openclaw memory promote-explain "router vlan"
    openclaw memory promote-explain "router vlan" --json
    ```

  </Tab>
  <Tab title="REM harness 预览">
    在不写入任何内容的情况下预览 REM 反思、候选事实和深度提升输出：

    ```bash
    openclaw memory rem-harness
    openclaw memory rem-harness --json
    ```

  </Tab>
</Tabs>

## 关键默认值

所有设置均位于 `plugins.entries.memory-core.config.dreaming` 下。

<ParamField path="enabled" type="boolean" default="false">
  启用或禁用 Dreaming 扫描。
</ParamField>
<ParamField path="frequency" type="string" default="0 3 * * *">
  完整 Dreaming 扫描的定时任务频率。
</ParamField>
<ParamField path="model" type="string">
  可选的梦境日记子智能体模型覆盖值。同时设置子智能体 `allowedModels` 允许列表时，请使用规范的 `provider/model` 值。
</ParamField>
<ParamField path="phases.deep.maxPromotedSnippetTokens" type="number" default="160">
  从每个提升到 `MEMORY.md` 的短期召回片段中保留的最大估算 token 数。排序来源信息仍然可见。
</ParamField>

<Warning>
`dreaming.model` 要求 `plugins.entries.memory-core.subagent.allowModelOverride: true`。要对其进行限制，还需设置 `plugins.entries.memory-core.subagent.allowedModels`。自动重试仅涵盖模型不可用错误；信任或允许列表失败会保留在日志中清晰可见，而不会静默回退。
</Warning>

<Note>
大多数阶段策略、阈值和存储行为都是内部实现细节。有关完整键列表，请参阅[记忆配置参考](/zh-CN/reference/memory-config#dreaming)。
</Note>

## Dreams UI

启用后，Gateway 网关的 **Dreams** 选项卡会显示：

- 当前 Dreaming 启用状态
- 阶段级状态和托管扫描是否存在
- 短期、有据、信号和今日已提升数量
- 下次计划运行时间
- 用于暂存历史重放条目的独立有据 Scene 通道
- 由 `doctor.memory.dreamDiary` 支持的可展开梦境日记阅读器

## 相关内容

- [记忆](/zh-CN/concepts/memory)
- [记忆 CLI](/zh-CN/cli/memory)
- [记忆配置参考](/zh-CN/reference/memory-config)
- [记忆搜索](/zh-CN/concepts/memory-search)
