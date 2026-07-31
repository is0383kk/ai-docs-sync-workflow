---
read_when:
    - 你想快速诊断渠道健康状况和最近的会话接收者
    - 你需要一个可直接粘贴用于调试的“全部”状态信息
summary: '`openclaw status` 的 CLI 参考（诊断、探测、使用情况快照）'
title: openclaw status
x-i18n:
    generated_at: "2026-07-26T06:39:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52e8076339216f11ddadf35e0ae8e5604322a47a5a9e2ee305468b2624d7cfde
    source_path: cli/status.md
    workflow: 16
---

渠道和会话的诊断。

```bash
openclaw status
openclaw status --all
openclaw status --deep
openclaw status --usage
```

| 标志                    | 说明                                                                                                     |
| ----------------------- | --------------------------------------------------------------------------------------------------------------- |
| `--all`                 | 完整诊断（只读，可直接粘贴）。包括安全审计、插件兼容性和记忆向量探测。 |
| `--deep`                | 运行实时探测（WhatsApp Web + Telegram + Discord + Slack + Signal）。同时启用安全审计。         |
| `--usage`               | 以 `X% left` 形式输出标准化的提供商用量时间窗口。                                                          |
| `--json`                | 机器可读输出。                                                                                        |
| `--verbose` / `--debug` | 还会在报告前输出原始 Gateway 网关目标解析结果。                                                 |

普通的 `openclaw status` 仍采用快速只读路径；跳过记忆检查时，会将记忆标记为
`not checked`，而不是不可用。繁重的安全审计、插件兼容性和记忆向量探测则交由
`openclaw status --all`、`openclaw status --deep`、`openclaw security audit`
和 `openclaw memory status --deep` 执行。

## 会话和模型解析

- 会话状态输出会区分 `Execution:` 和 `Runtime:`。`Execution`
  是沙箱路径（`direct`、`docker/*`），而 `Runtime` 会指明
  会话使用的是 `OpenClaw Default`、`OpenAI Codex`、CLI
  后端，还是 `codex (acp/acpx)` 等 ACP 后端。有关提供商、模型和运行时之间的
  区别，请参阅 [Agent Runtimes](/zh-CN/concepts/agent-runtimes)。
- 当前会话快照信息不足时，`/status` 可以从最新的脚本用量日志中回填 token
  和缓存计数器。现有的非零实时值仍优先于脚本回退值。
- 当实时会话条目缺少当前运行时模型标签时，脚本回退也可以恢复该标签。如果该脚本模型
  与所选模型不同，状态会根据恢复的运行时模型（而非所选模型）解析上下文窗口。
- 计算提示大小时，如果会话元数据缺失或数值较小，脚本回退会优先采用较大的
  提示导向总量，以免自定义提供商会话的 token 显示缩减为 `0`。
- 当会话固定使用的模型不同于配置的主模型时，状态会输出这两个值、原因（`session override`）
  以及提示 `/model default`。配置的主模型适用于新会话或未固定模型的会话；
  现有的固定模型会话在清除设置前会继续沿用其会话选择。
- 配置了多个智能体时，输出会包括每个智能体的会话存储。

## 用量和配额

- `--usage` 以 `X% left` 形式输出标准化的提供商用量时间窗口。
- MiniMax 的原始 `usage_percent` / `usagePercent` 字段表示剩余配额，
  因此 OpenClaw 会在显示前将其反转；如果存在基于计数的字段，则优先使用这些字段。
  `model_remains` 响应会优先使用聊天模型条目，必要时根据时间戳推导窗口标签，并在
  套餐标签中包含模型名称。
- 模型定价刷新失败会显示为可选的定价警告。
  这并不表示 Gateway 网关或渠道运行异常。

## 概览和更新状态

- 如果可用，概览会包括 Gateway 网关和节点主机服务的安装/运行时状态，
  以及精简的 Gateway 网关进程运行时长和主机系统运行时长。
- 概览会包括更新渠道和 git SHA（对于源代码检出）。
- 更新信息会显示在概览中；如果有可用更新，状态会提示运行
  `openclaw update`（参阅[更新](/zh-CN/install/updating)）。

## 密钥

- 如果正在运行的 Gateway 网关中有任何在启动、重新加载或写入配置时被隔离的 SecretRef 所有者，状态的 JSON 输出会包括 `degradedSecretOwners`，人类可读输出的概览中则会出现 **密钥降级** 行。每个条目都会列出所有者、降级状态（`cold` 或 `stale`）、配置路径和经过脱敏的原因。冷所有者不可用；过期所有者会继续使用最后一个已知的有效值。
- 只读状态界面（`status`、`status --json`、`status --all`）
  会尽可能解析其目标配置路径所支持的 SecretRef。
- 如果配置了受支持的渠道 SecretRef，但它在当前命令路径中不可用，
  状态仍保持只读并报告降级输出，而不会崩溃。人类可读输出会显示“配置的 token
  在此命令路径中不可用”等警告，JSON 输出则会包括
  `secretDiagnostics`。
- 当命令本地的 SecretRef 解析成功时，状态会优先使用解析后的快照，并从最终输出中
  清除临时的“密钥不可用”渠道标记。
- `status --all` 包括密钥概览行和诊断部分，
  其中会汇总密钥诊断信息（为便于阅读会截断），且不会停止生成报告。

## 记忆

`status --json --all` 会报告由 `plugins.slots.memory` 选择的主动记忆插件
运行时中的记忆详情。自定义记忆插件可以保持内置
`memory.search.enabled` 禁用，同时仍报告自己的
文件、分块、向量和 FTS 状态。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [Doctor](/zh-CN/gateway/doctor)
