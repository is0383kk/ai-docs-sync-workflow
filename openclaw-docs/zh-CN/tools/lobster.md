---
read_when:
    - 你需要具有明确审批机制的确定性多步骤工作流
    - 你需要在不重新运行先前步骤的情况下恢复工作流
summary: 适用于 OpenClaw 的类型化工作流运行时，支持可恢复的审批关卡。
title: Lobster
x-i18n:
    generated_at: "2026-07-26T07:04:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 85b7900f86bfedc9d73fcc91c3d0dac37b81f7413b1e68c54dd8a797b70f79fc
    source_path: tools/lobster.md
    workflow: 16
---

Lobster 将多步骤工具流水线作为一次确定性的工具调用运行，并提供明确的审批检查点和恢复令牌。它位于分离式后台工作的上一层：如需编排跨多个分离式任务的流程，请参阅 [Task Flow](/zh-CN/automation/taskflow)（`openclaw tasks flow`）；如需查看任务活动账本，请参阅[后台任务](/zh-CN/automation/tasks)。

## 原因

如果没有 Lobster，多步骤作业意味着需要多次往返工具调用，由模型编排每个步骤。Lobster 将这种编排移入类型化运行时：

- **一次调用代替多次调用**：一次 Lobster 工具调用即可返回整个流水线的结构化结果。
- **内置审批**：副作用操作（发送、发布、删除）会暂停工作流，直至获得明确批准。
- **可恢复**：暂停的工作流会返回一个令牌；批准后即可恢复，无需重新运行之前的步骤。

Lobster 是一种小型、受限的 DSL，而不是通用脚本语言：批准/恢复是持久的内置原语；流水线是数据（易于记录、比较差异、重放和审查）；精简的语法限制了“创造性”代码路径，使验证保持切合实际；超时、输出上限、沙箱检查和允许列表均由运行时强制执行，而不是由各个脚本执行。每个步骤仍然可以调用任意 CLI 或脚本——如果需要更丰富的编写语言，可以使用其他工具生成 `.lobster` 文件。

如果没有 Lobster，重复执行的电子邮件分类流程如下：

```text
用户：“检查我的电子邮件并起草回复”
→ openclaw 调用 gmail.list
→ LLM 进行总结
→ 用户：“为第 2 封和第 5 封起草回复”
→ LLM 起草回复
→ 用户：“发送第 2 封”
→ openclaw 调用 gmail.send
（每天重复，且不记得已经分类过哪些邮件）
```

使用 Lobster 后，同一作业只需一次调用，该调用会暂停以等待审批，并可在之后恢复：

```json
{ "action": "run", "pipeline": "email.triage --limit 20", "timeoutMs": 30000 }
```

```json
{
  "ok": true,
  "status": "needs_approval",
  "output": [{ "summary": "5 封需要回复，2 封需要处理" }],
  "requiresApproval": {
    "type": "approval_request",
    "prompt": "发送 2 封回复草稿？",
    "items": [],
    "resumeToken": "..."
  }
}
```

## 工作原理

OpenClaw 使用内置的 `@clawdbot/lobster` 软件包作为嵌入式运行器，**在进程内**运行 Lobster 工作流。不会生成外部 `lobster` 子进程；工具调用会直接返回 JSON 信封。如果流水线暂停以等待审批，信封中会携带恢复令牌（或简短的审批 ID），以便稍后继续。

## 启用

Lobster 是一个**可选**插件工具，默认未启用。它已内置提供，因此不需要单独安装——只需允许使用该工具：

```json
{
  "tools": {
    "alsoAllow": ["lobster"]
  }
}
```

或按智能体配置：

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": {
          "alsoAllow": ["lobster"]
        }
      }
    ]
  }
}
```

<Note>
`alsoAllow` 会在当前工具配置文件之上添加 `lobster`，而不会限制其他核心工具。仅当需要限制性允许列表模式时，才改用 `tools.allow`。
</Note>

在沙箱隔离的工具上下文中，该工具会被完全禁用。

如果需要使用独立的 Lobster CLI 进行开发或运行外部流水线（在嵌入式 Gateway 网关运行器之外），请从 [Lobster 仓库](https://github.com/openclaw/lobster)安装它，并将 `lobster` 放入 `PATH`。

## 模式：小型 CLI + JSON 管道 + 审批

构建使用 JSON 通信的小型命令，然后将它们链接为一次 Lobster 调用。（以下为示例命令名称——请替换为你自己的命令。）

```bash
inbox list --json
inbox categorize --json
inbox apply --json
```

```json
{
  "action": "run",
  "pipeline": "exec --json --shell 'inbox list --json' | exec --stdin json --shell 'inbox categorize --json' | exec --stdin json --shell 'inbox apply --json' | approve --preview-from-stdin --limit 5 --prompt 'Apply changes?'",
  "timeoutMs": 30000
}
```

如果流水线请求审批，请使用令牌恢复：

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

示例：将输入项映射为工具调用：

```bash
gog.gmail.search --query 'newer_than:1d' \
  | openclaw.invoke --tool message --action send --each --item-key message --args-json '{"provider":"telegram","to":"..."}'
```

## 仅 JSON 的 LLM 步骤（llm-task）

如需在工作流中使用**结构化 LLM 步骤**，请启用可选的 `llm-task` 插件工具，并从 Lobster 调用它：

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  },
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": { "alsoAllow": ["llm-task"] }
      }
    ]
  }
}
```

### 重要限制：嵌入式 Lobster 与 `openclaw.invoke`

内置 Lobster 插件在 Gateway 网关内**以进程内方式**运行工作流。在该嵌入式模式下，`openclaw.invoke` 不会自动继承用于嵌套 OpenClaw CLI 工具调用的 Gateway 网关 URL/身份验证上下文。

这意味着以下模式**目前在嵌入式运行器中并不可靠**：

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{ ... }'
```

仅当在已使用正确 Gateway 网关/身份验证上下文配置 `openclaw.invoke` 的环境中运行**独立 Lobster CLI** 时，才使用以下示例。

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "根据输入的电子邮件，返回意图和草稿。",
  "thinking": "low",
  "input": { "subject": "你好", "body": "你能帮忙吗？" },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

如果目前使用的是嵌入式 Lobster 插件，请优先选择以下任一方式：

- 在 Lobster 外部直接调用 `llm-task` 工具；或者
- 在添加受支持的嵌入式桥接之前，在 Lobster 流水线内使用非 `openclaw.invoke` 步骤。

有关详细信息和配置选项，请参阅 [LLM Task](/zh-CN/tools/llm-task)。

## 工作流文件（.lobster）

Lobster 可以运行包含 `name`、`args`、`steps`、`env`、`condition` 和 `approval` 字段的 YAML/JSON 工作流文件。在工具调用中将 `pipeline` 设置为文件路径。

```yaml
name: inbox-triage
args:
  tag:
    default: "family"
steps:
  - id: collect
    command: inbox list --json
  - id: categorize
    command: inbox categorize --json
    stdin: $collect.stdout
  - id: approve
    command: inbox apply --approve
    stdin: $categorize.stdout
    approval: required
  - id: execute
    command: inbox apply --execute
    stdin: $categorize.stdout
    condition: $approve.approved
```

注意：

- `stdin: $step.stdout` 和 `stdin: $step.json` 用于传递先前步骤的输出。
- `condition`（或 `when`）可以根据 `$step.approved` 控制步骤是否执行。

### 注入的环境变量

每个步骤的 shell 都会继承父环境以及以下由 Lobster 注入的变量，因此命令可以引用已解析的工作流参数，而无需将原始值嵌入命令字符串：

- `LOBSTER_ARG_<NAME>`——每个工作流参数对应一个。名称会转换为大写，并将每一段连续的非字母数字字符折叠为 `_`，因此参数 `user-id` 会变为 `LOBSTER_ARG_USER_ID`。
- `LOBSTER_ARGS_JSON`——将所有已解析参数表示为单个 JSON 字符串。

以上就是完整的注入变量集合。**不存在** `LOBSTER_STEP_<id>_STDOUT` 或 `LOBSTER_STEP_<id>_JSON_<field>` 之类的按步骤输出变量；shell 会将这些名称视为未设置，因此参数展开默认值可能会掩盖错误。应改为通过步骤引用读取先前步骤的输出——在 `stdin:`、`env:` 或 `condition:` 值中使用 `$step.stdout`、`$step.json` 或 `$step.json.<field>`。（`LOBSTER_STATE_DIR` 是用于状态目录的独立运行时设置，并非每次运行的参数。）

## 工具参数

### `run`

```json
{
  "action": "run",
  "pipeline": "gog.gmail.search --query 'newer_than:1d' | email.triage",
  "cwd": "workspace",
  "timeoutMs": 30000,
  "maxStdoutBytes": 512000
}
```

使用参数运行工作流文件：

```json
{
  "action": "run",
  "pipeline": "/path/to/inbox-triage.lobster",
  "argsJson": "{\"tag\":\"family\"}"
}
```

| 字段            | 默认值     | 说明                                                                                                        |
| ---------------- | ----------- | ------------------------------------------------------------------------------------------------------------ |
| `pipeline`       | 必填    | 内联流水线字符串，或以 `.lobster`/`.yaml`/`.yml`/`.json` 结尾的工作流文件路径。           |
| `cwd`            | Gateway 网关 cwd | 相对工作目录；必须解析到 Gateway 网关工作目录内（拒绝绝对路径）。 |
| `timeoutMs`      | `20000`     | 超过该值时中止运行。                                                                                  |
| `maxStdoutBytes` | `512000`    | 捕获的 stdout 或 stderr 超过此大小时中止运行。                                               |
| `argsJson`       | -           | 工作流文件参数的 JSON 字符串（内联流水线会忽略此项）。                                      |

### `resume`

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

`resume` 接受 `token`（来自 `requiresApproval` 的完整恢复令牌）或 `approvalId`（来自同一对象的简短 ID）——使用暂停运行所返回的任一值。`approve` 为必填项。

### 托管式 Task Flow 模式

在 `run` 上传入 `flowControllerId` 和 `flowGoal`（或在 `resume` 上传入 `flowId` 和 `flowExpectedRevision`），会通过插件运行时的托管式 [Task Flow](/zh-CN/automation/taskflow) API 驱动调用，而不是返回裸信封：OpenClaw 会创建或恢复持久化流程记录，将 Lobster 信封应用于该记录（审批时应用 `waiting`，完成时应用 `succeeded`/`failed`），并返回 `{ ok, envelope, flow, mutation }`。此模式需要绑定的 Task Flow 运行时，适用于需要在 Gateway 网关重启后仍保留持久化流程状态的插件/控制器代码，而不适用于典型的临时智能体使用场景。

## 输出信封

Lobster 返回一个 JSON 信封，其状态为以下三种之一：

- `ok`——成功完成
- `needs_approval`——已暂停；`requiresApproval` 携带 `resumeToken` 和简短的 `approvalId`，二者均可用于恢复运行
- `cancelled`——已明确拒绝或取消

该工具会同时通过 `content`（格式化 JSON）和 `details`（原始对象）提供该信封。

## 审批

如果存在 `requiresApproval`，请检查提示并作出决定：

- `approve: true`——恢复并继续执行副作用操作
- `approve: false`——取消并结束工作流

使用 `approve --preview-from-stdin --limit N` 可将 JSON 预览附加到审批请求，而无需自定义 jq/heredoc 拼接代码。恢复状态以小型 JSON 文件形式存储在 Lobster 状态目录下（默认为 `~/.lobster/state`，可使用 `LOBSTER_STATE_DIR` 覆盖）；令牌本身仅编码指向该状态的指针，而不包含完整的流水线状态。

## OpenProse

OpenProse 与 Lobster 配合良好：使用 `/prose` 编排多智能体准备工作，然后运行 Lobster 流水线进行确定性审批。如果 Prose 程序需要 Lobster，请通过 `tools.subagents.tools` 为子智能体允许使用 `lobster` 工具。请参阅 [OpenProse](/zh-CN/prose)。

## 安全性

- **仅限本地进程内** - 工作流在 Gateway 网关进程内执行；插件本身不发起
  网络调用。
- **无密钥** - Lobster 不管理 OAuth；它调用负责此功能的 OpenClaw 工具。
- **支持沙箱隔离** - 工具上下文处于沙箱隔离状态时禁用。
- **强化保护** - 嵌入式运行器强制执行超时和输出上限。

## 故障排查

| 错误                                                          | 原因/修复方法                                                                    |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `lobster runtime timed out`                                           | 管道超过了 `timeoutMs`。请增大该值或拆分管道。                            |
| `lobster stdout exceeded maxStdoutBytes`（或 `stderr`）                  | 捕获的输出超过上限。请提高 `maxStdoutBytes` 或减少输出。                       |
| `run --args-json must be valid JSON`                                           | `argsJson`（工作流文件运行）解析失败。请修复 JSON 字符串。               |
| `lobster runtime failed`（或其他 `runtime_error` 消息）         | 嵌入式运行时返回了错误封装。请查看 Gateway 网关日志以了解详情。                  |

## 了解更多

- [插件](/zh-CN/tools/plugin)
- [插件工具编写](/zh-CN/plugins/building-plugins#registering-agent-tools)

## 案例研究：社区工作流

一个公开示例：一个“第二大脑”CLI + Lobster 管道，用于管理三个
Markdown 仓库（个人、伴侣、共享）。该 CLI 输出统计信息、
收件箱列表和陈旧内容扫描的 JSON；Lobster 将这些命令串联成
`weekly-review`、`inbox-triage`、`memory-consolidation` 和
`shared-task-sync` 等工作流，每个工作流都设有审批关卡。AI 在可用时负责判断
（分类），不可用时则回退到确定性规则。

- 帖子：[https://x.com/plattenschieber/status/2014508656335770033](https://x.com/plattenschieber/status/2014508656335770033)
- 仓库：[https://github.com/bloomedai/brain-cli](https://github.com/bloomedai/brain-cli)

## 相关内容

- [自动化](/zh-CN/automation) - 所有自动化机制
- [工具概览](/zh-CN/tools) - 所有可用的智能体工具
