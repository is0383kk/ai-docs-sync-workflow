---
read_when:
    - 你想了解记忆的工作原理
    - 你想知道要写入哪些记忆文件
summary: OpenClaw 如何跨会话记住信息
title: 记忆概览
x-i18n:
    generated_at: "2026-07-26T06:42:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cdfd5276d6289a4ee38b5203eb5443312c4b040d4ea67abe4a9c579703136339
    source_path: concepts/memory.md
    workflow: 16
---

OpenClaw 通过在智能体的工作区中写入纯 Markdown 文件来记住信息（默认 `~/.openclaw/workspace`）。模型只会记住保存到磁盘的内容；不存在隐藏状态。

## 工作原理

智能体有三个与记忆相关的文件：

- **`MEMORY.md`** — 长期记忆。持久保存的事实、偏好和决定。在会话开始时加载。
- **`memory/YYYY-MM-DD.md`**（或 `memory/YYYY-MM-DD-<slug>.md`）— 每日笔记。
  持续记录的上下文和观察。仅使用 `/new` 或 `/reset` 时，会自动加载今天和昨天按日期命名的笔记；带有 slug 的变体（例如内置 session-memory 钩子写入的文件）也会与仅含日期的文件一同加载。
- **`DREAMS.md`**（可选）— 供人工审阅的梦境日记和 Dreaming 扫描摘要，包括有依据的历史回填条目。

<Tip>
如果想让智能体记住某件事，直接告诉它：“记住我更喜欢 TypeScript。”它会将笔记写入适当的文件。
</Tip>

## 各类内容的存放位置

`MEMORY.md` 是精简、经过整理的层：存放持久事实、偏好、长期决定，以及应在会话开始时可用的简短摘要。它不是原始对话记录、每日日志或完整归档。

`memory/YYYY-MM-DD.md` 文件是工作层：存放详细的每日笔记、观察、会话摘要，以及以后可能仍有用的原始上下文。这些文件会建立索引，以供 `memory_search` 和 `memory_get` 使用，但不会在每一轮都注入引导提示词。

随着时间推移，智能体会从每日笔记中提炼有用内容并写入 `MEMORY.md`，同时移除过时的长期条目。生成的工作区指令和 Heartbeat 流程会定期执行此操作；无需为每个细节手动编辑 `MEMORY.md`。

如果 `MEMORY.md` 超出引导文件预算，OpenClaw 会完整保留磁盘上的文件，但会截断注入上下文的副本。应将其视为一个信号：把详细内容移至 `memory/*.md`，在 `MEMORY.md` 中仅保留持久摘要；如果愿意消耗更多提示词预算，也可以提高引导限制。使用 `/context list`、`/context detail` 或 `openclaw doctor` 可查看原始大小、注入大小和截断状态。

## 从编码助手导入

Control UI 可以从 Codex 和 Claude Code 导入现有的本地记忆。
打开 **Settings** → **Import Memory**，选择目标智能体，审阅检测到的文件，然后确认导入。OpenClaw 只会复制 Markdown 记忆：

- Codex：位于 `~/.codex/memories`（或 `CODEX_HOME/memories`）下整合后的 `MEMORY.md` 和 `memory_summary.md` 文件。不会导入原始 rollout 和对话记录文件。
- Claude Code：每个项目位于 `~/.claude/projects/*/memory` 下的自动记忆目录中的 Markdown 文件，以及存在时由用户配置的 `autoMemoryDirectory`。项目指令、会话、设置和凭据不属于这项仅导入记忆的操作。

导入的文件会分别保留在所选智能体工作区的 `memory/imports/codex/` 和 `memory/imports/claude-code/` 下。它们会建立索引，以供 `memory_search` 使用，也可以通过 `memory_get` 访问；它们不会合并到智能体的引导 `MEMORY.md` 中。源文件保持不变。

预览会标记目标位置冲突。启用 **Replace existing imports** 可替换这些文件；应用操作会创建经过验证的导入前备份，并在迁移报告中保留被覆盖文件的逐项副本。

## 操作敏感型记忆

大多数记忆都是普通的 Markdown 笔记。有些记忆会影响智能体以后应执行的操作；对于这些记忆，不仅要记录事实本身，还要记录何时可以安全地根据该笔记采取行动。

当笔记涉及以下内容时，应记录这一操作边界：

- 审批或权限要求，
- 临时限制，
- 移交给其他会话、线程或人员，
- 到期条件，
- 可安全采取行动的时机，
- 来源或所有者的权限，
- 避免执行某项诱人操作的指令。

实用的操作敏感型记忆应明确说明：

- 哪些内容会改变未来行为，
- 何时或在什么条件下适用，
- 何时到期，或什么条件会解锁操作，
- 智能体应避免做什么，
- 来源或所有者是谁（如果这会影响可信度或权限）。

记忆可以保留审批上下文，但不会强制执行策略。请使用 OpenClaw 的审批设置、沙箱隔离和定时任务来实施严格的操作控制。

示例：

```md
API 迁移正在另一个会话中设计。未来的轮次不应在此线程中编辑 API 实现；在迁移计划落地之前，只能将这里的发现用作设计输入。
```

另一个示例：

```md
来自不可信来源的报告需要经过审阅才能采用。未来的轮次应只将其视为证据；在可信审阅者确认其内容之前，不要将其存储为持久记忆。
```

这并非每条记忆都必须遵循的架构；简单事实可以保持简洁。如果丢失时机、权限、到期条件或可安全采取行动的上下文可能导致智能体以后执行错误操作，请使用操作敏感型边界。

使用[定时任务](/zh-CN/automation/cron-jobs)执行精确提醒、定时检查和重复工作。记忆仍可概括这些工作周围的持久上下文。

## 已停用的推断式跟进承诺

有些未来的跟进事项并不是持久事实。如果提到明天有面试，有用的记忆可能是“面试后跟进”，而不是“将此内容永久存储在 `MEMORY.md` 中”。

推断式跟进承诺实验已停用。OpenClaw 不再提取或发送这些跟进事项。请使用[定时任务](/zh-CN/automation/cron-jobs)处理未来操作；旧版 `openclaw commitments` 命令仍可用于检查或删除现有的已存储行。

## 记忆工具

智能体有两个用于处理记忆的工具：

- **`memory_search`** — 使用语义搜索查找相关笔记，即使用词与原文不同也能找到。
- **`memory_get`** — 读取特定记忆文件或行范围。

这两个工具均由当前启用的记忆插件提供（默认：`memory-core`）。

## 记忆搜索

配置嵌入提供商后，`memory_search` 会使用混合搜索：将向量相似度（语义含义）与关键词匹配（ID 和代码符号等精确术语）相结合。为任何受支持的提供商配置 API 密钥后即可直接使用。

<Info>
OpenClaw 默认使用 OpenAI 嵌入。显式设置 `memory.search.provider`，可使用 Gemini、Voyage、Mistral、Bedrock、DeepInfra、本地 GGUF、Ollama、LM Studio、GitHub Copilot 或通用的 OpenAI 兼容端点。
</Info>

有关搜索的工作原理、调优选项和提供商设置，请参阅[记忆搜索](/zh-CN/concepts/memory-search)。

## 记忆后端

<CardGroup cols={3}>
<Card title="内置（默认）" icon="database" href="/zh-CN/concepts/memory-builtin">
基于 SQLite。开箱即用，支持关键词搜索、向量相似度和混合搜索。无需额外依赖。
</Card>
<Card title="QMD" icon="search" href="/zh-CN/concepts/memory-qmd">
本地优先的 sidecar，支持重新排序、查询扩展，以及为工作区外的目录建立索引。
</Card>
<Card title="Honcho" icon="brain" href="/zh-CN/concepts/memory-honcho">
AI 原生的跨会话记忆，支持用户建模、语义搜索和多智能体感知。需要安装插件。
</Card>
<Card title="LanceDB" icon="layers" href="/zh-CN/plugins/memory-lancedb">
由 LanceDB 支持的记忆，提供 OpenAI 兼容嵌入、自动召回、自动捕获和本地 Ollama 嵌入支持。需要安装插件。
</Card>
</CardGroup>

## 知识 wiki 层

如果希望持久记忆的行为更接近维护良好的知识库，而不是原始笔记，请使用内置的 `memory-wiki` 插件。它将持久知识编译为 wiki 仓库，其中包含确定性的页面结构、结构化主张和证据、矛盾与新鲜度跟踪、生成的仪表板、编译后的摘要，以及 wiki 原生工具（`wiki_status`、`wiki_search`、`wiki_get`、`wiki_apply`、`wiki_lint`）。

`memory-wiki` 不会替代当前启用的记忆插件；当前启用的记忆插件仍负责召回、晋升和 Dreaming。`memory-wiki` 会在其旁边添加一个包含丰富来源信息的知识层。

<CardGroup cols={1}>
<Card title="Memory Wiki" icon="book" href="/zh-CN/plugins/memory-wiki">
将持久记忆编译为包含丰富来源信息的 wiki 仓库，支持主张、仪表板、桥接模式和适合 Obsidian 的工作流。
</Card>
</CardGroup>

## 自动刷新记忆

在[压缩](/zh-CN/concepts/compaction)总结对话之前，OpenClaw 会运行一个静默轮次，提醒智能体将重要上下文保存到记忆文件中。此功能默认启用；设置 `agents.defaults.compaction.memoryFlush.enabled: false` 可将其关闭。

要让该内务处理轮次使用本地模型，请设置仅适用于记忆刷新轮次的精确覆盖项（它不会继承当前会话的模型回退链）：

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "model": "ollama/qwen3:8b"
        }
      }
    }
  }
}
```

<Tip>
记忆刷新可防止压缩期间丢失上下文。如果对话中包含尚未写入文件的重要事实，系统会在生成摘要前自动保存它们。
</Tip>

## Dreaming

Dreaming 是一个可选的后台记忆整合流程。它会收集短期召回信号、为候选项评分，并且只将符合条件的条目晋升到长期记忆（`MEMORY.md`）：

- **选择启用**：默认禁用。
- **定时执行**：启用后，`memory-core` 会自动管理一个重复执行完整 Dreaming 扫描的定时任务。
- **设有阈值**：晋升必须通过分数、召回频率和查询多样性关卡。
- **可审阅**：阶段摘要和日记条目会写入 `DREAMS.md`，供人工审阅。

有关阶段行为、评分信号和梦境日记的详细信息，请参阅 [Dreaming](/zh-CN/concepts/dreaming)。

## 有依据的回填和实时晋升

Dreaming 系统包含两个相关的审阅通道：

- **实时 Dreaming** 使用 `memory/.dreams/` 下的短期 Dreaming 存储；正常的深度阶段会据此决定哪些内容可以晋升到 `MEMORY.md`。
- **有依据的回填** 将历史 `memory/YYYY-MM-DD.md` 笔记作为独立的每日文件读取，并将结构化审阅输出写入 `DREAMS.md`。

有依据的回填适合用于重放旧笔记并检查系统认定哪些内容具有持久价值，而无需手动编辑 `MEMORY.md`。

```bash
openclaw memory rem-backfill --path ./memory --stage-short-term
```

`--stage-short-term` 标志会将有依据的持久候选项暂存到正常深度阶段已经使用的同一短期 Dreaming 存储中；它不会直接晋升这些候选项。因此：

- `DREAMS.md` 仍是供人工审阅的界面。
- 短期存储仍是面向机器的排序界面。
- `MEMORY.md` 仍只由深度晋升写入。

要撤销重放而不影响普通日记条目或正常召回状态：

```bash
openclaw memory rem-backfill --rollback
openclaw memory rem-backfill --rollback-short-term
```

## CLI

```bash
openclaw memory status          # 检查索引状态和提供商
openclaw memory search "query"  # 从命令行搜索
openclaw memory index --force   # 重建索引
```

## 延伸阅读

- [记忆搜索](/zh-CN/concepts/memory-search)：搜索管线、提供商和调优。
- [内置记忆引擎](/zh-CN/concepts/memory-builtin)：默认 SQLite 后端。
- [QMD 记忆引擎](/zh-CN/concepts/memory-qmd)：高级本地优先边车。
- [Honcho 记忆](/zh-CN/concepts/memory-honcho)：AI 原生跨会话记忆。
- [Memory LanceDB](/zh-CN/plugins/memory-lancedb)：由 LanceDB 支持的插件，使用 OpenAI 兼容的嵌入模型。
- [Memory Wiki](/zh-CN/plugins/memory-wiki)：编译式知识库和 wiki 原生工具。
- [Dreaming](/zh-CN/concepts/dreaming)：在后台将短期回忆提升为长期记忆。
- [记忆配置参考](/zh-CN/reference/memory-config)：所有配置选项。
- [压缩](/zh-CN/concepts/compaction)：压缩如何与记忆交互。
- [主动记忆](/zh-CN/concepts/active-memory)：用于交互式聊天会话的子智能体记忆。
