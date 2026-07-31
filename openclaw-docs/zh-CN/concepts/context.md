---
read_when:
    - 你想了解 OpenClaw 中的“上下文”是什么意思
    - 你正在调试模型为什么“知道”某件事（或忘记了它）
    - 你希望减少上下文开销（/context、/status、/compact）
summary: 上下文：模型看到的内容、其构建方式及检查方法
title: 上下文
x-i18n:
    generated_at: "2026-07-26T06:42:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1eb3d342a601a447487640587f746cc80a133ede338a880741f53c3e01f20ed1
    source_path: concepts/context.md
    workflow: 16
---

“上下文”是 **OpenClaw 为一次运行发送给模型的全部内容**。其容量受模型的**上下文窗口**（Token 限制）约束。

初学者心智模型：

- **系统提示词**（由 OpenClaw 构建）：规则、工具、Skills 列表、时间/运行时，以及注入的工作区文件。
- **对话历史**：本会话中你的消息 + 助手的消息。
- **工具调用/结果 + 附件**：命令输出、读取的文件、图像/音频等。

上下文与“记忆”_并不相同_：记忆可以存储在磁盘上并在之后重新加载；上下文是模型当前窗口内的内容。

## 快速开始（检查上下文）

- `/status` → 快速查看“我的窗口已使用多少？”以及会话设置。
- `/context list` → 查看注入了哪些内容及其大致大小（每个文件 + 总计）。
- `/context detail` → 更深入的明细：每个文件、每个工具架构的大小、每个 Skill 条目的大小、系统提示词大小，以及可压缩的对话记录消息数。
- `/context map` → 当前会话中已跟踪上下文来源的 WinDirStat 风格树状图。
- `/usage tokens` → 在普通回复后附加每次回复的用量页脚。
- `/compact` → 将较早的历史记录总结为精简条目，以释放窗口空间。

另请参阅：[斜杠命令](/zh-CN/tools/slash-commands)、[Token 使用量和成本](/zh-CN/reference/token-use)、[压缩](/zh-CN/concepts/compaction)。

## 输出示例

具体值因模型、提供商、工具策略和工作区内容而异。

### `/context list`

```text
🧠 上下文明细
工作区：<workspaceDir>
引导内容每个文件上限：12,000 个字符
沙箱：mode=non-main sandboxed=false
系统提示词（运行）：38,412 个字符（约 9,603 个 Token）（项目上下文 23,901 个字符（约 5,976 个 Token））

注入的工作区文件：
- AGENTS.md：正常 | 原始 1,742 个字符（约 436 个 Token）| 注入 1,742 个字符（约 436 个 Token）
- SOUL.md：正常 | 原始 912 个字符（约 228 个 Token）| 注入 912 个字符（约 228 个 Token）
- TOOLS.md：已截断 | 原始 54,210 个字符（约 13,553 个 Token）| 注入 20,962 个字符（约 5,241 个 Token）
- IDENTITY.md：正常 | 原始 211 个字符（约 53 个 Token）| 注入 211 个字符（约 53 个 Token）
- USER.md：正常 | 原始 388 个字符（约 97 个 Token）| 注入 388 个字符（约 97 个 Token）
- HEARTBEAT.md：缺失 | 原始 0 | 注入 0
- BOOTSTRAP.md：正常 | 原始 0 个字符（约 0 个 Token）| 注入 0 个字符（约 0 个 Token）

Skills 列表（系统提示词文本）：2,184 个字符（约 546 个 Token）（12 个 Skills）
工具：read、edit、write、exec、process、browser、message、sessions_send、…
工具列表（系统提示词文本）：1,032 个字符（约 258 个 Token）
工具架构（JSON）：31,988 个字符（约 7,997 个 Token）（计入上下文；不显示为文本）
工具：（同上）

会话 Token（缓存）：总计 14,250 / ctx=32,000
```

### `/context detail`

```text
🧠 上下文明细（详细）
…
最大的 Skills（提示词条目大小）：
- frontend-design：412 个字符（约 103 个 Token）
- oracle：401 个字符（约 101 个 Token）
…（另有 10 个 Skills）

最大的工具（架构大小）：
- browser：9,812 个字符（约 2,453 个 Token）
- exec：6,240 个字符（约 1,560 个 Token）
…（另有 N 个工具）
```

### `/context map`

发送一张根据最新缓存的运行报告和会话对话记录生成的图像。在普通消息生成本会话的运行报告之前，`/context map` 会返回不可用消息，而不是渲染估算结果。矩形面积与所跟踪提示词的字符数成正比：

- 对话记录（用户消息、助手回复、工具结果、压缩摘要），以及仅发送给模型的每轮运行时上下文和钩子提示词增补内容
- 注入的工作区文件
- 基础系统提示词文本
- Skill 提示词条目
- 工具 JSON 架构

对话组会随会话增长，因此此图会逐轮变化；压缩后，它会折叠为摘要区块。

没有缓存运行报告时，`/context list`、`/context detail` 和 `/context json` 仍可检查按需估算结果。

## 哪些内容会计入上下文窗口

模型接收的所有内容都会计入，包括：

- 系统提示词（所有部分）。
- 对话历史。
- 工具调用 + 工具结果。
- 附件/转录内容（图像/音频/文件）。
- 压缩摘要和剪枝工件。
- 提供商“包装器”或隐藏标头（不可见，但仍会计入）。

## OpenClaw 如何构建系统提示词

系统提示词**归 OpenClaw 所有**，并在每次运行时重新构建。其中包括：

- 工具列表 + 简短说明。
- Skills 列表（仅元数据；见下文）。
- 工作区位置。
- 时间（UTC + 已配置时转换后的用户时间）。
- 运行时元数据（主机/操作系统/模型/思考模式）。
- 在**项目上下文**下方注入的工作区引导文件。

完整明细：[系统提示词](/zh-CN/concepts/system-prompt)。

## 注入的工作区文件（项目上下文）

默认情况下，OpenClaw 会注入一组固定的工作区文件（如果存在）：

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`（仅首次运行）

大文件会使用 `agents.defaults.bootstrapMaxChars` 按文件截断（默认 `20000` 个字符）。OpenClaw 还会通过 `agents.defaults.bootstrapTotalMaxChars` 对所有文件强制执行引导内容注入总上限（默认 `60000` 个字符）。`/context` 会显示**原始大小与注入大小**，以及是否发生截断。

发生截断时，运行时可以在项目上下文下方注入提示词内警告块。通过 `agents.defaults.bootstrapPromptTruncationWarning` 配置此行为（`off`、`once`、`always`；默认 `always`）。

## Skills：注入与按需加载

系统提示词包含精简的 **Skills 列表**（名称 + 描述 + 位置）。此列表会产生实际开销。

默认不包含 Skill 指令。模型应当**仅在需要时** `read` 该 Skill 的 `SKILL.md`。

## 工具：存在两类成本

工具通过两种方式影响上下文：

1. 系统提示词中的**工具列表文本**（即你看到的“工具”）。
2. **工具架构**（JSON）。这些内容会发送给模型，以便模型调用工具。即使你无法看到其纯文本形式，它们仍会计入上下文。

`/context detail` 会列出最大的工具架构明细，方便查看哪些内容占比最高。

## 命令、指令和“内联快捷方式”

斜杠命令由 Gateway 网关处理。它们有以下几种不同行为：

- **独立命令**：仅包含 `/...` 的消息会作为命令运行。
- **指令**：`/think`、`/fast`、`/verbose`、`/trace`、`/reasoning`、`/elevated`、`/exec`、`/model`、`/queue` 会在模型看到消息之前被移除。
  - 仅包含指令的消息会持久保存会话设置。
  - 普通消息中的内联指令会作为单条消息级别的提示。
- **内联快捷方式**（仅限允许列表中的发送者）：普通消息中的某些 `/...` Token 可以立即运行（例如：“hey /status”），并会在模型看到其余文本之前被移除。

详情：[斜杠命令](/zh-CN/tools/slash-commands)。

## 会话、压缩和剪枝（哪些内容会持久保存）

消息之间持久保存哪些内容取决于所用机制：

- **普通历史记录**会保留在会话对话记录中，直到按策略压缩/剪枝。
- **压缩**会将摘要持久保存到对话记录中，同时完整保留近期消息。
- **剪枝**会从_内存中的_提示词删除旧工具结果，以释放上下文窗口空间，但不会重写会话对话记录——完整历史记录仍可在磁盘上检查。

文档：[会话](/zh-CN/concepts/session)、[压缩](/zh-CN/concepts/compaction)、[会话剪枝](/zh-CN/concepts/session-pruning)。

默认情况下，OpenClaw 使用内置的 `legacy` 上下文引擎进行组装和
压缩。如果安装了提供 `kind: "context-engine"` 的插件，并通过
`plugins.slots.contextEngine` 选择它，OpenClaw 会将上下文
组装、`/compact` 及相关子智能体上下文生命周期钩子委托给该
引擎。`ownsCompaction: false` 不会自动回退到旧版
引擎；活动引擎仍必须正确实现 `compact()`。有关完整的
可插拔接口、生命周期钩子和配置，请参阅
[上下文引擎](/zh-CN/concepts/context-engine)。

## `/context` 实际报告的内容

如果可用，`/context` 会优先使用最新的**运行时构建**系统提示词报告：

- `System prompt (run)` = 从上次嵌入式（支持工具）运行中捕获，并持久保存到会话存储中。
- `System prompt (estimate)` = 当不存在运行报告时（或通过不会生成该报告的 CLI 后端运行时）即时计算。

无论哪种方式，它都会报告大小和主要来源；但**不会**转储完整的系统提示词或工具架构。在详细模式下，它还会使用压缩所采用的同一真实对话消息判定条件，将会话对话记录进行比较，以便更容易区分较高的提示词/缓存用量与可压缩的对话历史。

## 相关内容

<CardGroup cols={2}>
  <Card title="上下文引擎" href="/zh-CN/concepts/context-engine" icon="puzzle-piece">
    通过插件自定义上下文注入。
  </Card>
  <Card title="压缩" href="/zh-CN/concepts/compaction" icon="compress">
    总结长对话，使其保持在模型窗口内。
  </Card>
  <Card title="系统提示词" href="/zh-CN/concepts/system-prompt" icon="message-lines">
    系统提示词的构建方式以及每轮注入的内容。
  </Card>
  <Card title="Agent loop" href="/zh-CN/concepts/agent-loop" icon="arrows-rotate">
    从入站消息到最终回复的完整智能体执行周期。
  </Card>
</CardGroup>
