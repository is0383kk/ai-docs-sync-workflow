---
read_when:
    - 更改 Agent 运行时、工作区引导或会话行为
summary: Agent 运行时、工作区契约和会话引导启动
title: Agent 运行时
x-i18n:
    generated_at: "2026-07-26T06:45:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4d3dd9c0c65e4ccd791a2a6131f1b7457c8cfee6da71502d93c355280e094390
    source_path: concepts/agent.md
    workflow: 16
---

OpenClaw 提供一个**嵌入式智能体运行时**：内置的 Agent loop、工具
连接和提示词组装机制，不同于将轮次委托给外部
harness 进程。每个已配置的智能体（如需运行多个智能体，请参阅[多智能体路由](/zh-CN/concepts/multi-agent)）
都有自己的工作区、引导文件和会话
存储。本页介绍该运行时契约：工作区必须
包含哪些内容、注入哪些文件，以及会话如何基于这些内容进行引导。

## 工作区（必需）

每个智能体使用一个工作区目录（`agents.defaults.workspace`，或每个智能体使用
`agents.entries.*.workspace`）作为工具和上下文的**唯一**工作目录（`cwd`）。

建议：使用 `openclaw setup` 在 `~/.openclaw/openclaw.json` 缺失时创建它，并初始化工作区文件。

完整的工作区布局和备份指南：[Agent 工作区](/zh-CN/concepts/agent-workspace)

如果启用了 `agents.defaults.sandbox`，非主会话可以使用
`agents.defaults.sandbox.workspaceRoot` 下的每会话工作区覆盖此设置（请参阅
[Gateway 配置](/zh-CN/gateway/configuration)）。

## 引导文件（注入）

在工作区内，OpenClaw 需要以下可由用户编辑的文件：

| 文件           | 用途                                              |
| -------------- | ---------------------------------------------------- |
| `AGENTS.md`    | 操作说明 + “记忆”                    |
| `SOUL.md`      | 人格、边界、语气                            |
| `TOOLS.md`     | 用户维护的工具说明和约定           |
| `IDENTITY.md`  | 智能体名称/风格/表情符号                                |
| `USER.md`      | 用户资料 + 首选称呼                     |
| `HEARTBEAT.md` | Heartbeat 专用说明                      |
| `BOOTSTRAP.md` | 一次性的首次运行仪式（完成后删除） |
| `MEMORY.md`    | 根级长期记忆文件（如果存在）               |

在新会话的第一个轮次中，OpenClaw 会将这些文件的内容注入系统提示词的项目上下文。仅当 `MEMORY.md` 存在于工作区根目录时才会注入它。

空文件会被跳过。大型文件会经过裁剪和截断，并附加标记，以保持提示词精简（请读取文件以查看完整内容）。缺失的文件（`MEMORY.md` 除外）会改为注入一行“文件缺失”标记；`openclaw setup` 会为其创建安全的默认模板。

仅当工作区**全新**（不存在其他引导文件）时，才会创建 `BOOTSTRAP.md`。在其处于待处理状态期间，OpenClaw 会将其保留在项目上下文中，并在系统提示词中添加初始仪式的引导说明，而不是将其复制到用户消息中。如果在完成仪式后将其删除，则以后重启时不会重新创建。

工作区被观测到后，OpenClaw 会将其设置状态和
证明存储在共享 SQLite 数据库
`~/.openclaw/state/openclaw.sqlite` 中。如果最近经过证明的工作区
消失或被清空，启动过程将拒绝静默重新生成 `BOOTSTRAP.md`；
请恢复工作区，或执行完整的引导设置重置，以便同时清除工作区及其
数据库状态。

旧版本使用工作区 JSON 和 `.attested` 辅助文件。运行时
不会读取这些文件。请运行 `openclaw doctor --fix` 验证这些文件，将其
状态导入 SQLite，并在验证导入的行后删除每个源文件。

要完全禁用引导文件创建（适用于预先填充的工作区），请设置：

```json5
{ agents: { defaults: { skipBootstrap: true } } }
```

## 内置工具

核心工具（read/exec/edit/write 及相关系统工具）始终可用，
但受工具策略约束。对于 OpenAI 模型，`apply_patch` 默认启用，并受
`tools.exec.applyPatch`（`enabled`、`workspaceOnly`、`allowModels`）限制。`TOOLS.md` **不**控制存在哪些工具；它用于指导工具应如何按_你的_意愿使用。

## Skills

OpenClaw 从以下位置加载 Skills（优先级从高到低）：

- 工作区：`<workspace>/skills`
- 项目智能体 Skills：`<workspace>/.agents/skills`
- 个人智能体 Skills：`~/.agents/skills`
- 托管/本地：`~/.openclaw/skills`
- 内置（随安装一起提供）
- 额外 Skills 文件夹：`skills.load.extraDirs`

Skills 根目录可以包含分组文件夹，例如
`<workspace>/skills/personal/foo/SKILL.md`；该 Skill 仍通过其
扁平的 frontmatter 名称公开，例如 `foo`。

Skills 可以由配置/环境变量控制（请参阅 [Gateway 配置](/zh-CN/gateway/configuration)中的 `skills`）。

## 运行时边界

嵌入式智能体运行时归 OpenClaw 所有：模型发现、工具连接、
提示词组装、会话管理和渠道交付共享一个集成的
运行时表面。

## 会话

会话行存储在每个智能体的 SQLite 数据库中：

- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`

转录 JSONL 文件仍可位于
`~/.openclaw/agents/<agentId>/sessions/` 下，作为旧版迁移输入、已删除或
重置的归档、导入、导出和支持工件。活跃的智能体历史记录
与会话行一起存储在 SQLite 中。会话 ID 保持稳定，并由
OpenClaw 选择。OpenClaw 不会读取其他工具的会话文件夹。

## 流式传输期间的 Steer

默认情况下，运行期间收到的入站提示词会通过 Steer 加入当前运行。
Steer 会在**当前助手轮次执行完其
工具调用后**、下一次 LLM 调用前传递，并且不再跳过
当前助手消息中剩余的工具调用。

`/queue steer` 是活跃运行的默认行为。`/queue followup` 和
`/queue collect` 会让消息等待后续轮次，而不是执行 Steer。
`/queue interrupt` 会改为中止活跃运行。有关队列和边界行为，请参阅[队列](/zh-CN/concepts/queue)
和 [Steering queue](/zh-CN/concepts/queue-steering)。

分块流式传输会在完成每个助手内容块后立即发送；
**默认关闭**（`agents.defaults.blockStreamingDefault: "off"`）。
通过 `agents.defaults.blockStreamingBreak` 调整边界（`text_end` 与 `message_end`；默认为 `text_end`）。
使用 `agents.defaults.blockStreamingChunk` 控制软分块（默认为
800-1200 个字符；优先在段落分隔处拆分，其次是换行，最后是句子）。
使用 `agents.defaults.blockStreamingCoalesce` 合并流式分块，以减少
单行刷屏（发送前基于空闲状态合并）。非 Telegram 渠道需要
显式设置 `*.streaming.block.enabled: true` 才能启用分块回复（QQ Bot
则会流式传输分块回复，除非 `channels.qqbot.streaming.mode` 为 `"off"`）。
详细工具摘要会在工具启动时发出（无防抖）；如果可用，Control UI
会通过智能体事件流式传输工具输出。
更多详情：[流式传输和分块](/zh-CN/concepts/streaming)。

## 模型引用

配置中的模型引用（例如 `agents.defaults.model` 和 `agents.defaults.models`）通过在**第一个** `/` 处分割来解析。

- 配置模型时使用 `provider/model`。
- 如果模型 ID 本身包含 `/`（OpenRouter 风格），请包含提供商前缀（示例：`openrouter/moonshotai/kimi-k2`）。
- 如果省略提供商，OpenClaw 会先尝试别名，然后查找与该模型 ID 完全匹配的唯一
  已配置提供商，最后才回退到
  已配置的默认提供商。如果该提供商不再提供
  已配置的默认模型，OpenClaw 会回退到第一个已配置的
  提供商/模型，而不是暴露一个指向已移除提供商的过时默认值。

## 配置（最小）

至少设置：

- `agents.defaults.workspace`
- `channels.whatsapp.allowFrom`（强烈建议）

## 相关内容

- [Agent 工作区](/zh-CN/concepts/agent-workspace)
- [多智能体路由](/zh-CN/concepts/multi-agent)
- [会话管理](/zh-CN/concepts/session)
- [群聊](/zh-CN/channels/group-messages)
