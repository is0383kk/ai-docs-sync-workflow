---
read_when:
    - 用户报告智能体陷入重复调用工具的循环中
    - 你需要控制重复调用保护机制
    - 你正在编辑智能体工具/运行时策略
    - 在上下文溢出重试后遇到 `compaction_loop_persisted` 中止错误
summary: 如何启用检测重复工具调用循环的防护机制
title: 工具循环检测
x-i18n:
    generated_at: "2026-07-26T07:03:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 79b5aa1d85e02b8cf46a95b3bcebb255178b91456517cab804cce77b8f3b818e
    source_path: tools/loop-detection.md
    workflow: 16
---

OpenClaw 有两道相互配合的防护机制，用于防止重复的工具调用模式，
两者均在 `tools.loopDetection` 下配置：

1. **循环检测**（`enabled`）- 默认禁用。监视滚动的
   工具调用历史记录，以发现重复模式和对未知工具的重试。
2. **压缩后防护** - 只要
   `enabled` 未被显式设置为 `false`，便会启用。每次压缩重试后进入防护状态；如果智能体在窗口内重复相同的 `(tool, args, result)` 三元组，
   则中止运行。

将 `tools.loopDetection.enabled: false` 设置为静默禁用两道防护机制。

## 存在原因

- 检测没有取得任何进展的重复序列。
- 检测高频且无结果的循环（相同工具、相同输入、重复
  错误）。
- 检测已知轮询工具的特定重复调用模式。
- 打破上下文溢出 -> 压缩 -> 相同循环的周期，而不是让它们
  无限期运行。

## 配置块

全局设置：

```json5
{
  tools: {
    loopDetection: {
      enabled: false, // 滚动历史记录检测器的总开关
    },
  },
}
```

按智能体覆盖（可选，位于 `agents.entries.*.tools.loopDetection`）：

```json5
{
  agents: {
    list: [
      {
        id: "safe-runner",
        tools: {
          loopDetection: {
            enabled: true,
          },
        },
      },
    ],
  },
}
```

按智能体设置会覆盖全局设置。

### 字段行为

| 字段     | 默认值 | 效果                                                                                            |
| --------- | ------- | ------------------------------------------------------------------------------------------------- |
| `enabled` | `false` | 滚动历史记录检测器的总开关。`false` 还会禁用压缩后防护。 |

对于 `exec`，无进展哈希会比较稳定的命令结果（状态、
退出代码、超时标志、输出），并忽略易变的运行时元数据，例如
持续时间、PID、会话 ID 和工作目录。出站消息发送
结果在哈希计算时会移除每次调用中易变的 ID（消息 ID、文件 ID、时间戳），
因此一个“已发送”结果不会与另一个不同的“已发送”
结果看起来完全相同。当运行 ID 可用时，只会在该次运行内评估历史记录，
因此定时 Heartbeat 周期和新运行不会继承
先前运行中陈旧的循环计数。

## 推荐设置

- 对于较小的模型，请设置 `enabled: true`。旗舰模型很少需要滚动历史记录检测，可以
  将总开关保持为 `false`，同时仍受益于
  压缩后防护。
- 若要禁用包括压缩后防护在内的所有机制，请显式设置
  `tools.loopDetection.enabled: false`。

## 压缩后防护

在上下文溢出后进行压缩重试之后，运行器会针对接下来的几次工具调用启用
短窗口防护。如果智能体在该窗口内足够多次地发出相同的
`(toolName, argsHash, resultHash)` 三元组，防护机制会判定压缩未能打破
循环，并以 `compaction_loop_persisted` 错误中止运行。

该防护机制受总开关 `tools.loopDetection.enabled` 标志控制，但有一个
特殊之处：当该标志未设置或为 `true` 时，它会保持**启用**，仅当该标志被显式设置为 `false` 时
才会关闭。这是有意为之——该防护机制用于摆脱原本会无限消耗 token 的压缩循环，
因此未进行配置的用户仍能获得保护。

```json5
{
  tools: {
    loopDetection: {
      // 总开关；设置为 false 可同时禁用该防护机制和滚动检测器
      enabled: true,
    },
  },
}
```

- 当结果仍在变化时，该防护机制绝不会中止运行；只有窗口内
  字节完全相同的结果才会触发它。
- 它仅在压缩重试后立即启用，而不会在运行中的其他
  时刻启用。

<Note>
  只要总开关标志未被显式设置为 `false`，压缩后防护就会运行，即使你从未编写过 `tools.loopDetection` 块也是如此。若要验证，请在发生压缩事件后立即在 Gateway 网关日志中查找 `post-compaction guard armed for N attempts`。
</Note>

## 日志和预期行为

检测到循环时，OpenClaw 会记录循环事件，并根据严重程度发出警告或阻止
下一个工具周期，从而防止 token 消耗失控和运行锁死，同时保留正常的工具访问能力。

- 首先发出警告。
- 当某种模式持续超过警告阈值后，便会进行阻止。
- 达到严重阈值时，会阻止下一个工具周期，并在运行记录中显示明确的
  循环检测原因。
- 压缩后防护会发出 `compaction_loop_persisted` 错误，其中会指出
  导致问题的工具和相同调用次数。

## 相关内容

<CardGroup cols={2}>
  <Card title="Exec 审批" href="/zh-CN/tools/exec-approvals" icon="shield">
    shell 执行的允许/拒绝策略。
  </Card>
  <Card title="思考级别" href="/zh-CN/tools/thinking" icon="brain">
    推理投入级别及其与提供商策略的交互。
  </Card>
  <Card title="子智能体" href="/zh-CN/tools/subagents" icon="users">
    生成隔离的智能体，以限制失控行为。
  </Card>
  <Card title="配置参考" href="/zh-CN/gateway/config-tools#toolsloopdetection" icon="gear">
    完整的 `tools.loopDetection` schema 和合并语义。
  </Card>
</CardGroup>
