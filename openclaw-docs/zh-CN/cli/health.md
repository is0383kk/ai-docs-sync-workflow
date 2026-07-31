---
read_when:
    - 你想快速检查正在运行的 Gateway 网关的健康状态
summary: '`openclaw health` 的 CLI 参考（通过 RPC 获取 Gateway 健康快照）'
title: 健康
x-i18n:
    generated_at: "2026-07-26T06:38:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 51cc0e3dd61af3e6fa460dd646bfa1c3e5bd1a52da860eac26c12101151d081d
    source_path: cli/health.md
    workflow: 16
---

# `openclaw health`

通过 WebSocket RPC 从正在运行的 Gateway 网关获取健康快照（CLI 不直接连接渠道套接字）。

## 选项

| 标志             | 默认值 | 描述                                                                       |
| ---------------- | ------- | --------------------------------------------------------------------------------- |
| `--json`         | `false` | 输出机器可读的 JSON，而不是文本。                                      |
| `--timeout <ms>` | `10000` | 连接超时时间（毫秒）。                                               |
| `--verbose`      | `false` | 强制执行实时探测，并展开输出所有已配置的账户和智能体。 |
| `--debug`        | `false` | `--verbose` 的别名。                                                            |

示例：

```bash
openclaw health
openclaw health --json
openclaw health --timeout 2500
openclaw health --verbose
openclaw health --debug
```

## 行为

- 未使用 `--verbose` 时，Gateway 网关可以返回缓存的快照（最长 60 秒内保持新鲜，且与实时渠道运行时状态一致），并在后台刷新快照以供下一个调用者使用。
- `--verbose` 强制执行实时探测（逐渠道账户探测），输出 Gateway 网关连接详情，并展开所有已配置账户和智能体的可读文本输出，而非仅显示默认智能体。
- `--json` 始终返回完整快照：渠道、逐账户探测、插件加载状态、上下文引擎隔离状态、模型定价缓存状态、事件循环健康状况、投递队列死信，以及各智能体的会话存储。
- 当出站投递或入站渠道事件进入死信队列时，文本输出会报告其数量和最早失败的时长。入站数量按渠道账户分组；可通过 [`openclaw channels dead-letters`](/zh-CN/cli/channels#inbound-dead-letters) 检查或恢复单个事件。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [`openclaw status`](/zh-CN/cli/status) — 无需完整健康快照即可执行本地诊断和渠道探测
- [Gateway 健康](/zh-CN/gateway/health)
