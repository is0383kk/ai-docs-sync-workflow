---
read_when:
    - 准备错误报告或支持请求
    - 调试 Gateway 网关崩溃、重启、内存压力或超大负载问题
    - 审查记录或脱敏了哪些诊断数据
summary: 创建可共享的 Gateway 网关诊断包，用于提交错误报告
title: 诊断导出
x-i18n:
    generated_at: "2026-07-26T06:42:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 97a805fed8d51de2e63e5c6a12ce03e91701d69654882cca7795c9f3553b1c55
    source_path: gateway/diagnostics.md
    workflow: 16
---

OpenClaw 可以为错误报告构建本地诊断 `.zip`：经过清理的 Gateway 网关
状态、健康信息、日志、配置结构以及近期不含有效载荷的稳定性事件。

在审查之前，请像对待机密信息一样对待诊断包。有效载荷和凭据
按设计会被编辑，但该包仍会汇总本地 Gateway 网关日志和
主机级运行时状态。

## 快速开始

```bash
openclaw gateway diagnostics export
```

输出已写入的 zip 路径。选择输出路径：

```bash
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
```

用于自动化：

```bash
openclaw gateway diagnostics export --json
```

## 聊天命令

所有者可以在任何对话中运行 `/diagnostics [note]`，请求生成一个本地
Gateway 网关导出，并将其作为一份可直接复制粘贴的支持报告：

1. 发送 `/diagnostics`，可选择附加简短说明（`/diagnostics bad tool choice`）。
2. OpenClaw 会发送前置说明，并请求一次明确的 Exec 审批，该审批会运行
   `openclaw gateway diagnostics export --json`。不要通过
   允许全部规则批准诊断操作。
3. 审批后，OpenClaw 会回复本地诊断包路径、清单
   摘要、隐私说明和相关会话 ID。

在群聊中，所有者仍可运行 `/diagnostics`，但 OpenClaw 会将
导出结果、审批提示以及 Codex 会话/线程明细私下发送给
所有者。群组中只会看到一条简短通知，说明诊断信息已私下发送。
如果不存在通往所有者的私信路由，该命令会以安全关闭方式失败，并要求
所有者从私信中运行该命令。

当活动会话使用原生 OpenAI Codex harness 时，同一次 Exec
审批还会涵盖将 OpenClaw 已知 Codex 线程的 OpenAI 反馈上传。
该上传独立于本地 Gateway 网关 zip，并且仅在 Codex harness 会话中
发生。审批提示会说明批准操作也会发送 Codex 反馈，但不会列出 Codex
会话或线程 ID。审批后，回复会列出已发送至 OpenAI 的线程所对应的
渠道、OpenClaw 会话 ID、Codex 线程 ID 和本地恢复命令。拒绝或
忽略审批将跳过导出、Codex 反馈上传和 Codex ID 列表。

这样可以缩短 Codex 调试流程：在渠道中发现异常行为后，
运行 `/diagnostics`，批准一次，共享报告，然后如果想自行检查线程，
可在本地运行输出的 `codex resume <thread-id>` 命令。
请参阅 [Codex harness](/zh-CN/plugins/codex-harness#inspect-codex-threads-locally)。

## 导出内容

- `summary.md`：供支持人员阅读的概览。
- `diagnostics.json`：配置、日志、状态、健康信息和
  稳定性数据的机器可读摘要。
- `manifest.json`：导出元数据和文件列表。
- 经过清理的配置结构和非机密配置详情。
- 经过清理的日志摘要和近期已编辑的日志行。
- 尽力获取的 Gateway 网关状态和健康快照。
- `stability/latest.json`：可用时最新的持久化稳定性包。

即使 Gateway 网关运行不正常，该导出仍然有用：如果状态/健康
请求失败，仍会在可用时收集本地日志、配置结构和最新的稳定性包。

## 隐私模型

保留：子系统名称、插件 ID、提供商 ID、渠道 ID、已配置的
模式、状态码、持续时间、字节数、队列状态、内存读数、
经过清理的日志元数据、已编辑的操作消息、配置结构和
非机密功能设置。

省略或编辑：聊天文本、提示词、指令、webhook 正文、工具
输出、凭据、API 密钥、令牌、Cookie、机密值、原始
请求/响应正文、账户 ID、消息 ID、原始会话 ID、
主机名和本地用户名。

当日志消息看起来包含用户、聊天、提示词或工具有效载荷文本时，
导出内容只会保留消息已被省略这一事实及其字节数。

## 稳定性记录器

默认情况下，启用诊断后，Gateway 网关会记录一个有界且不含有效载荷的
稳定性事件流。它捕获的是操作事实，而不是内容。

当事件循环或 CPU 看起来达到饱和状态时，同一 Heartbeat 还会采样
活跃性，并发出 `diagnostic.liveness.warning` 事件，其中包含事件循环延迟、
事件循环利用率、CPU 核心比率、活动/等待/排队的会话数量、
当前启动/运行时阶段（若已知）、近期阶段跨度以及
有界的工作标签。只有当有工作正在等待或排队，或者活动工作与持续的
事件循环延迟重叠时，这些事件才会成为 Gateway 网关 `warn` 级别的日志行；
否则将以 `debug` 级别记录。空闲时的活跃性样本仍会作为
诊断事件记录，但其本身绝不会升级为警告。

启动阶段会发出包含实际时间和
CPU 计时的 `diagnostic.phase.completed` 事件。当最后一次桥接进度看起来已结束
（例如原始响应项或响应完成事件），但 Gateway 网关仍认为
嵌入式运行处于活动状态时，卡住的嵌入式运行诊断会标记 `terminalProgressStale=true`。

检查实时记录器：

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --json
```

在致命退出、关闭超时或
重启启动失败后，检查最新的持久化包：

```bash
openclaw gateway stability --bundle latest
```

根据最新的持久化包创建诊断 zip：

```bash
openclaw gateway stability --bundle latest --export
```

存在事件时，持久化包位于 `~/.openclaw/logs/stability/` 下。

## 常用选项

```bash
openclaw gateway diagnostics export \
  --output openclaw-diagnostics.zip \
  --log-lines 5000 \
  --log-bytes 1000000
```

| 标志                    | 默认值                                                                       | 说明                                        |
| ----------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------- |
| `--output <path>`       | `$OPENCLAW_STATE_DIR/logs/support/openclaw-diagnostics-<timestamp>-<pid>.zip` | 写入指定的 zip 路径（或目录）。       |
| `--log-lines <count>`   | `5000`                                                                        | 要包含的经过清理的日志行数上限。            |
| `--log-bytes <bytes>`   | `1000000`                                                                     | 要检查的日志字节数上限。                      |
| `--url <url>`           | -                                                                             | 用于状态/健康快照的 Gateway 网关 WebSocket URL。 |
| `--token <token>`       | -                                                                             | 用于状态/健康快照的 Gateway 网关令牌。         |
| `--password <password>` | -                                                                             | 用于状态/健康快照的 Gateway 网关密码。      |
| `--timeout <ms>`        | `3000`                                                                        | 状态/健康快照超时时间。                    |
| `--no-stability-bundle` | 关闭                                                                           | 跳过持久化稳定性包查找。            |
| `--json`                | 关闭                                                                           | 输出机器可读的导出元数据。            |

## 禁用诊断

诊断默认启用。要禁用稳定性记录器和
诊断事件收集：

```json5
{
  diagnostics: {
    enabled: false,
  },
}
```

禁用诊断会减少错误报告的详细信息，但不会影响正常的
Gateway 网关日志记录。

内存压力事件会记录 RSS、堆、阈值和增长情况
（`rss_threshold`、`heap_threshold`、`rss_growth`），且不会执行
文件系统扫描或写入 OOM 前快照。

## 相关内容

- [健康检查](/zh-CN/gateway/health)
- [Gateway CLI](/zh-CN/cli/gateway#gateway-diagnostics-export)
- [Gateway 网关协议](/zh-CN/gateway/protocol#rpc-method-families)
- [日志](/zh-CN/logging)
- [OpenTelemetry 导出](/zh-CN/gateway/opentelemetry) - 用于将诊断数据流式传输到收集器的独立流程
