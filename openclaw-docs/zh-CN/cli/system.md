---
read_when:
    - 你想在不创建定时任务的情况下将系统事件加入队列
    - 你需要启用或禁用 Heartbeat
    - 你想检查系统在线状态条目
summary: '`openclaw system` 的 CLI 参考（系统事件、Heartbeat、在线状态）'
title: 系统
x-i18n:
    generated_at: "2026-07-26T06:41:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aaca206d8b463fd33f9e3cb21382bbf36469e9daa2706d8a9e2c7fab14b76e7a
    source_path: cli/system.md
    workflow: 16
---

# `openclaw system`

Gateway 网关的系统级辅助工具：将系统事件加入队列、控制 Heartbeat，以及查看在线状态。

所有 `system` 子命令都使用 Gateway RPC，并接受共享客户端标志：

| 标志              | 默认值                              | 说明                                                                                                                                                                                            |
| ----------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `--url <url>`     | 已配置时为 `gateway.remote.url` | Gateway WebSocket URL。                                                                                                                                                                                 |
| `--token <token>` | 无                                 | Gateway 令牌（如果需要）。                                                                                                                                                                           |
| `--timeout <ms>`  | `30000`                              | RPC 超时（毫秒）。                                                                                                                                                                           |
| `--expect-final`  | 关闭                                  | 等待最终响应（智能体）。                                                                                                                                                                       |
| `--json`          | 关闭                                  | 输出 JSON。无论此标志如何设置，`heartbeat last/enable/disable` 和 `system presence` 始终输出原始 RPC JSON 载荷；`system event` 使用此标志在 JSON 和纯 `ok` 行之间切换。 |

## 常用命令

```bash
openclaw system event --text "检查是否有紧急的后续事项" --mode now
openclaw system event --text "检查是否有紧急的后续事项" --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
openclaw system heartbeat enable
openclaw system heartbeat last
openclaw system presence
```

## `system event`

默认将系统事件加入**主**会话的队列。下一次 Heartbeat 会将其作为 `System:` 行注入提示词。使用 `--mode now` 可立即触发 Heartbeat；`next-heartbeat`（默认）会等待下一次计划触发。

传入 `--session-key` 可指定特定会话，例如，将异步任务完成事件转发回启动该任务的渠道。

<Note>
**使用 `--session-key` 时的时序例外：**提供 `--session-key` 后，`--mode next-heartbeat` 会转变为立即定向唤醒，而不是等待下一次计划触发。定向唤醒使用 Heartbeat 意图 `immediate`，因此会绕过运行器的“尚未到期”门控；否则，该门控会推迟（并实际丢弃）使用 `event` 意图的唤醒。如果需要延迟投递，请省略 `--session-key`，使事件进入主会话，并随下一次常规 Heartbeat 一起投递。
</Note>

标志：

- `--text <text>`：必需的系统事件文本。
- `--mode <mode>`：`now` 或 `next-heartbeat`（默认）。
- `--session-key <sessionKey>`：可选；指定特定的智能体会话，而不是该智能体的主会话。不属于已解析智能体的键将回退到该智能体的主会话。

## `system heartbeat last|enable|disable`

- `last`：显示上一次 Heartbeat 事件。
- `enable`：重新开启 Heartbeat（如果已禁用，请使用此项）。
- `disable`：暂停 Heartbeat。

## `system presence`

列出 Gateway 网关当前已知的系统在线状态条目（节点、实例及类似状态行）。

## 备注

- 需要有一个正在运行且可通过当前配置访问的 Gateway 网关（本地或远程）。
- 系统事件是临时的，不会跨重启持久保留。

## 相关内容

- [CLI 参考](/zh-CN/cli)
