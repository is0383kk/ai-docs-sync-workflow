---
read_when:
    - 在智能体已运行时使用 /steer 或 /tell
    - 比较 /steer 与 /queue 模式
    - 决定是引导当前运行还是 ACP 会话
sidebarTitle: Steer
summary: 在不更改队列模式的情况下 Steer 活跃运行
title: Steer
x-i18n:
    generated_at: "2026-07-26T07:05:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d420e14982d52520e415103ffa6d86923fad6f13c43ff7741ebbd8dde0d0073f
    source_path: tools/steer.md
    workflow: 16
---

`/steer` 会先尝试向已在运行的任务发送指导。它适用于
“在此次运行仍在进行时进行调整”的场景。如果当前运行时
无法接受 Steer，OpenClaw 会改为将该消息作为普通提示词发送，
而不会将其丢弃。

## 当前会话

使用顶层 `/steer` 定位当前会话的活动运行：

```text
/steer 优先采用较小的补丁，并保持测试聚焦
/tell 在进行下一次工具调用前先总结
```

行为：

- 仅定位当前会话的活动运行。
- 不受会话的 `/queue` 模式影响。
- 当会话空闲或活动运行无法接受 Steer 时，使用同一消息
  开始一个普通轮次。
- 使用活动运行时的 Steer 路径，因此模型会在
  下一个受支持的运行时边界看到该指导。

## Steer 与队列

当普通入站消息到达时某个运行正在进行，`/queue steer` 会让这些消息尝试
Steer 活动运行。`/steer <message>` 是一个显式命令，
无论存储的 `/queue` 设置如何，都会尝试在下一个
受支持的运行时边界将该命令的消息注入活动运行。如果
无法进行该注入，命令前缀会被移除，`<message>`
将作为普通提示词继续处理。

显式 `/steer`（以及 `/tell`）命令由 Gateway 网关支持。在
`openclaw chat` 或 `openclaw tui --local` 中，选择 `/queue steer` 并将
指导作为普通消息发送；嵌入式运行时会应用相同的 Steer
策略，而不会转发 Gateway 网关命令。

用法：

- `/steer <message>`：希望立即指导活动运行时使用。
- `/queue steer`：希望未来的普通消息默认对活动运行执行 Steer 时
  使用。
- `/queue collect` 或 `/queue followup`：未来的普通消息应等待
  后续轮次，而不是对活动运行执行 Steer 时使用。
- `/queue interrupt`：最新消息应替换活动运行，
  而不是对其执行 Steer 时使用。

有关队列模式和 Steer 边界，请参阅[命令队列](/zh-CN/concepts/queue)和
[Steering queue](/zh-CN/concepts/queue-steering)。

## 子智能体

顶层 `/steer` 定位当前会话的活动运行。子智能体会向
其父会话或请求方会话报告；`/subagents` 仅用于查看。

## ACP 会话

目标为 ACP harness 会话时，使用 `/acp steer`：

```text
/acp steer --session agent:main:acp:codex 收紧复现步骤
```

有关 ACP 会话选择和运行时行为，请参阅
[ACP 智能体](/zh-CN/tools/acp-agents)。

## 相关内容

- [斜杠命令](/zh-CN/tools/slash-commands)
- [命令队列](/zh-CN/concepts/queue)
- [Steering queue](/zh-CN/concepts/queue-steering)
- [子智能体](/zh-CN/tools/subagents)
