---
doc-schema-version: 1
read_when:
    - 你希望 OpenClaw 在长时间会话中始终显示一个目标
    - 你需要暂停、恢复、阻止、完成或清除会话目标
    - 你想了解 `get_goal`、`create_goal` 和 `update_goal` 工具
    - 你想查看目标在 TUI 中的显示方式
summary: 会话目标：持久的每会话目标、/goal 控制、模型目标工具、令牌预算和 TUI 状态
title: 目标
x-i18n:
    generated_at: "2026-07-26T07:03:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8bfe25eb9901394b32b61729fbcb6a7bd711ed859d284fa39b637000ed7f0a18
    source_path: tools/goal.md
    workflow: 16
---

# 目标

**目标**是附加到当前 OpenClaw 会话的一项持久目标。
它让智能体和操作员能够为长期工作设定共同目标，
而不会将该目标变成后台任务、提醒、定时任务或
长期指令。

目标属于会话状态：它们随会话键一起移动，可在进程
重启后保留，并显示在 `/goal`、面向模型的目标工具以及 TUI
页脚中。

分离命令的完成结果会返回最初面向用户的线程，因此
即使命令执行使用了单独的沙箱策略会话，下一轮仍会看到
同一个目标。

## 快速开始

```text
/goal start get CI green for PR 87469 and push the fix
/goal
/goal edit get CI green for PR 87469, push the fix, and update docs
/goal pause waiting for CI
/goal resume
/goal complete pushed and verified
/goal clear
```

`start` 是可选的：`/goal get CI green for PR 87469` 也会创建目标，
因为 `/goal` 后面任何不是已知操作词的文本都会被视为
新目标。

## 目标的用途

当会话有一个应在多个轮次中始终可见的具体结果时，
可以使用目标：

- PR 收尾：修复、验证、自动审查、推送，然后创建或更新 PR。
- 调试过程：复现错误、确定归属的功能面、应用补丁并
  证明修复有效。
- 文档处理：阅读相关文档、编写新页面、添加交叉链接并
  验证文档构建。
- 维护任务：检查当前状态、进行有界变更、运行
  正确的检查，并报告变更内容。

目标不是任务队列。如果工作需要分离运行、
按计划重复、扇出为受管理的子工作，或作为策略持久存在，请使用 [Task Flow](/zh-CN/automation/taskflow)、
[任务](/zh-CN/automation/tasks)、[定时任务](/zh-CN/automation/cron-jobs)或
[长期指令](/zh-CN/automation/standing-orders)。

## 命令参考

不带参数的 `/goal` 会输出当前目标摘要：

```text
目标
状态：活跃
目标：让 PR 87469 的 CI 通过并推送修复
已用 Token：12k
Token 预算：12k/50k

命令：/goal edit <objective>、/goal pause、/goal complete、/goal clear
```

| 命令                                                | 效果                                                                     |
| --------------------------------------------------- | ------------------------------------------------------------------------ |
| `/goal` 或 `/goal status`                           | 显示当前目标。                                                           |
| `/goal start <objective>`                           | 为当前会话创建新目标。                                                   |
| `/goal set <objective>`、`/goal create <objective>` | `start` 的别名。                                                     |
| `/goal <objective>`                                 | 也会创建新目标（任何不是已识别操作词的文本）。                           |
| `/goal edit <objective>`                            | 重新表述当前目标；状态和 Token 统计保持不变。                            |
| `/goal pause [note]`                                | 暂停活跃目标。                                                           |
| `/goal resume [note]`                               | 恢复已暂停、已阻塞、用量受限或预算受限的目标。                           |
| `/goal complete [note]`                             | 将目标标记为已达成。                                                     |
| `/goal done [note]`                                 | `complete` 的别名。                                                    |
| `/goal block [note]`                                | 将目标标记为已阻塞。                                                     |
| `/goal blocked [note]`                              | `block` 的别名。                                                       |
| `/goal clear`                                       | 从会话中移除目标。                                                       |

每个会话同时只能存在一个目标。在清除当前目标之前，启动第二个目标会失败，
并显示 `Goal error: goal already exists`。

`/goal start` 不接受 Token 预算标志；预算只能通过
面向模型的 `create_goal` 工具设置。

## 状态

- `active`：会话正在推进目标。
- `paused`：操作员暂停了目标；`/goal resume` 会使其再次变为活跃状态。
- `blocked`：智能体或操作员报告了真实的阻塞因素；当有新信息或状态可用时，`/goal resume`
  会使其再次变为活跃状态。
- `budget_limited`：已达到配置的 Token 预算；`/goal resume`
  会使用新的预算窗口，从同一目标重新开始推进。
- `usage_limited`：为未来的用量限制停止状态保留；`/goal
resume` 会以相同方式重新开始推进。
- `complete`：目标已达成。已完成的目标是终止状态；开始另一个目标前请使用 `/goal
clear`。

`/new` 和 `/reset` 会清除当前会话目标，因为它们有意
从新的会话上下文开始。

## Token 预算

目标可以设置可选的正数 Token 预算，通过
`create_goal` 工具的 `token_budget` 参数进行设置。预算从
创建目标时会话的最新 Token 计数开始计算。如果目标启动时，会话只有
过期或未知的 Token 快照，OpenClaw 会等待下一个
最新快照并将其用作基线，因此目标存在之前消耗的 Token
不会计入其中。

当用量达到预算时，目标会转为 `budget_limited`。这
不会删除目标或清除目标内容；它会告知操作员和
智能体，在恢复或清除目标之前，不再主动推进该目标。
恢复目标会以当前最新 Token
计数为起点启动新的预算窗口。

Token 预算是会话目标的护栏，而不是计费上限。提供商
配额、费用报告和上下文窗口行为仍使用 OpenClaw 的常规
用量和模型控制。

## 模型工具

OpenClaw 向 Agent harnesses 提供三个目标工具：

| 工具          | 用途                                                                                                                       |
| ------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `get_goal`    | 读取当前会话目标：状态、目标内容、Token 用量和 Token 预算。                                                        |
| `create_goal` | 仅当用户或系统指令明确要求时才创建目标。如果会话已有目标，则失败。                                               |
| `update_goal` | 将目标标记为 `complete` 或 `blocked`。                                                                               |

模型不能在不作说明的情况下暂停、恢复、清除或替换目标。这些操作仍然是
操作员/会话通过 `/goal` 和重置命令执行的控制，因此智能体
可以报告目标已达成或存在真实阻塞，而不会悄然
改变目标。

仅当目标确实达成时，`update_goal` 才应将其标记为
`complete`。仅当同一阻塞条件至少连续出现
三个目标轮次后，才应将目标标记为 `blocked`，不能因为
一般困难或尚未完善而这样做。

## 每一轮中的目标上下文

每个具有活跃目标的用户/聊天轮次都会包含以下用户角色上下文行：

```text
活跃目标：<objective> — 推进目标或更新其状态（get_goal/update_goal）。
```

OpenClaw 会截断过长的目标内容，使该行保持紧凑。已暂停、
已阻塞、预算受限、用量受限和已完成的目标不会被注入，
因此操作员的停止操作会持续生效，直到目标恢复。

## Control UI

Web Control UI 会在聊天编辑器上方以紧凑胶囊形式显示目标：
包括状态图标、状态标签（例如 `Pursuing goal`）、截断后的
目标内容和实时计时器。

胶囊包含内联控件：

- **铅笔**会在编辑器中预填 `/goal edit <objective>`，以便
  重新表述并提交目标。
- **暂停/恢复**会根据当前状态在 `/goal pause` 和 `/goal resume` 之间
  切换。
- **垃圾桶**会发送 `/goal clear`。
- **箭头**会展开胶囊，显示完整目标、最新状态
  说明、Token 用量和已用时间。

当编辑器无法发送内容时（例如
Gateway 网关连接断开），操作按钮会隐藏；展开箭头仍可使用。

## TUI

TUI 页脚会将活跃会话的目标显示在智能体、
会话和模型字段旁边，并置于 Token/模式指示器之前。

页脚示例：

- `Pursuing goal (12k/50k)` 表示具有 Token 预算的活跃目标。
- `Goal paused (/goal resume)` 表示已暂停的目标。
- `Goal blocked (/goal resume)` 表示已阻塞的目标。
- `Goal hit usage limits (/goal resume)` 表示用量受限的目标。
- `Goal unmet (50k/50k)` 表示预算受限的目标。
- `Goal achieved (42k)` 表示已完成的目标。

页脚有意保持紧凑。使用 `/goal` 可查看完整目标、
说明、Token 预算和可用命令。

## 渠道行为

`/goal` 可在支持命令的 OpenClaw 会话中使用，包括 TUI 和
允许文本命令的聊天界面。目标状态附加到
会话键而非传输协议，因此共享同一会话键的两个界面会看到
同一个目标。

目标状态不是交付指令：它不会强制通过某个
渠道回复、改变队列行为、批准工具或安排工作。

## 故障排查

| 消息                                   | 含义                                                                                                                                         |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `Goal error: goal already exists`      | 会话已有目标。使用 `/goal` 检查目标；如果已完成，使用 `/goal complete`；或者在启动其他目标前使用 `/goal clear`。 |
| `Goal error: goal not found`           | 会话尚无目标。使用 `/goal start <objective>` 启动一个目标。                                                                        |
| `Goal error: goal is already complete` | 目标处于终止状态。启动或恢复其他目标前，请先将其清除。                                                                                       |

如果 Token 用量显示 `0` 或看起来已过期，活跃会话可能尚无
最新的 Token 快照。OpenClaw 记录会话用量
和根据对话记录得出的总量时，用量会随之刷新。

## 相关内容

- [斜杠命令](/zh-CN/tools/slash-commands)
- [TUI](/zh-CN/web/tui)
- [会话工具](/zh-CN/concepts/session-tool)
- [压缩](/zh-CN/concepts/compaction)
- [Task Flow](/zh-CN/automation/taskflow)
- [长期指令](/zh-CN/automation/standing-orders)
