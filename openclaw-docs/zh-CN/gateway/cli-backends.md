---
read_when:
    - 当 API 提供商发生故障时，你希望有可靠的回退方案
    - 你正在运行本地 AI CLI，并希望复用它们
    - 你想了解用于 CLI 后端工具访问的 MCP 回环桥接机制
summary: CLI 后端：本地 AI CLI 回退方案，可选配 MCP 工具桥接器
title: CLI 后端
x-i18n:
    generated_at: "2026-07-26T06:13:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ce0427c587bf2a1e0a2ff24b5e76952eecae059e6f900af777b897b2d8d4210
    source_path: gateway/cli-backends.md
    workflow: 16
---

OpenClaw 可在 API 提供商宕机、受到速率限制或行为异常时，将本地 AI CLI 作为纯文本回退方案运行。其设计刻意保持保守：

- 不会直接注入 OpenClaw 工具，但具有 `bundleMcp: true` 的后端可通过 loopback MCP 桥接接收 Gateway 网关工具。
- 对支持 JSONL 的 CLI 进行流式传输。
- 支持会话，因此后续轮次能够保持连贯。
- 如果 CLI 接受图像路径，则图像会直接传递。

将其作为确保文本响应“始终可用”的安全网，而不是主要路径。如需具备 ACP 会话控制、后台任务、线程/对话绑定和持久外部编码会话的完整 harness 运行时，请改用 [ACP 智能体](/zh-CN/tools/acp-agents)；CLI 后端并非 ACP。

<Tip>
  要构建新的后端插件？请参阅 [CLI 后端插件](/zh-CN/plugins/cli-backend-plugins)。本页介绍如何配置和操作已注册的后端。
</Tip>

## 快速开始

内置 Anthropic 插件注册了默认的 `claude-cli` 后端，因此只要安装 Claude Code 并完成登录，无需其他配置即可使用：

```bash
openclaw agent --agent main --message "hi" --model claude-cli/claude-sonnet-4-6
```

未配置明确的智能体列表时，`main` 是默认智能体 ID；否则请替换为你自己的智能体 ID。

Gateway 网关服务的 `PATH` 中必须包含该 CLI。如果部署需要
非标准的可执行文件路径或参数，请在
[CLI 后端插件](/zh-CN/plugins/cli-backend-plugins)中注册该适配器，而不是将启动
机制放入 `openclaw.json`。

当模型选择或模型作用域的 `agentRuntime.id` 引用某个后端时，OpenClaw 会自动加载
拥有该后端的内置插件。

## 将其用作回退方案

将 CLI 后端添加到回退列表，使其仅在主模型失败时运行：

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["claude-cli/claude-sonnet-4-6"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "claude-cli/claude-sonnet-4-6": {},
      },
    },
  },
}
```

当主要提供商失败（身份验证、速率限制、超时）时，已配置的回退项仍可使用，即使它们不在 `agents.defaults.modelPolicy.allow` 中也是如此。只有当还应允许用户通过 `/model`、会话覆盖或 `--model` 直接选择某个 CLI 后端模型时，才将其添加到该策略。`agents.defaults.models` 仅负责各模型的别名、参数和元数据。

## 配置

用户通过模型和运行时策略选择已注册的后端。保持
模型引用规范，并按模型选择 CLI 运行时：

```json5
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-5",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
      },
    },
  },
}
```

凭据仍保留在 OpenClaw 身份验证配置文件或所属插件的配置中。
命令、argv、环境、解析、会话、图像和看门狗机制均由
通过 `api.registerCliBackend(...)` 注册的插件代码实现。

## 工作原理

1. 按提供商前缀（`claude-cli/...`）选择后端。
2. 使用相同的 OpenClaw 提示词和工作区上下文构建系统提示词。
3. 使用会话 ID（如果支持）执行 CLI，以保持历史记录一致。内置的 `claude-cli` 后端会为每个 OpenClaw 会话保持一个 Claude stdio 进程运行，并通过 stream-json stdin 发送后续轮次。
4. 解析输出（JSON 或纯文本）并返回最终文本。
5. 按后端持久化会话 ID，使后续轮次复用同一 CLI 会话。

## 超时和长时间运行的工作

CLI 后端有两个相互独立的限制：

- `agents.defaults.timeoutSeconds` 限制整个智能体轮次。常规 Gateway 网关轮次继承 48 小时默认值；`0` 会使轮次预算不受限制。已存储的覆盖值（如 `600`）会替换该默认值。
- CLI 无输出看门狗会停止持续无输出的子进程。每个后端插件分别拥有全新会话和恢复会话的配置，并且即使总体轮次预算不受限制，看门狗仍然有效。

移除较短的总体超时覆盖即可恢复 48 小时默认值，也可以设置明确的预算，例如 12 小时：

```bash
# 恢复 48 小时默认值：
openclaw config unset agents.defaults.timeoutSeconds

# 或选择明确的 12 小时限制：
openclaw config set agents.defaults.timeoutSeconds 43200
```

CLI 内启动的后台工作仍属于该 CLI 子进程。如果父轮次达到总体限制，OpenClaw 会同时停止该子进程及其 CLI 内部后台任务。如需持久执行长时间工作，请使用分离式 OpenClaw [子智能体](/zh-CN/tools/subagents)或 [ACP 智能体](/zh-CN/tools/acp-agents)；分离式子智能体默认没有运行超时。

`openclaw agent` 命令也有自己的请求截止时间。其 600 秒回退默认值仅适用于该命令调用，而不适用于常规 Gateway 网关轮次；请参阅 [`openclaw agent`](/zh-CN/cli/agent)。

### Claude CLI 特有行为

内置的 `claude-cli` 后端优先使用 Claude Code 的原生技能解析器。当当前技能快照中至少有一个具有实体化路径的已选技能时，OpenClaw 会通过 `--plugin-dir` 传递临时 Claude Code 插件，并从追加的系统提示词中省略重复的 OpenClaw 技能目录。没有实体化的插件技能时，OpenClaw 会保留提示词目录作为回退方案。技能环境变量/API 密钥覆盖仍会应用到本次运行的子进程环境中。

Claude CLI 有自己的非交互式权限模式；OpenClaw 会将其映射到现有 Exec 策略，而不是添加 Claude 专用配置。对于由 OpenClaw 管理的 Claude 实时会话，以生效的 Exec 策略为准：YOLO（`tools.exec.mode: "full"`）通常会使用 `--permission-mode bypassPermissions` 启动 Claude，而限制性策略会使用 `--permission-mode default` 启动。以 root 身份运行的 Gateway 网关也会使用 `default`，因为 Claude Code 拒绝 root 使用绕过模式。每个智能体的 `agents.entries.*.tools.exec` 设置会覆盖该智能体的全局 `tools.exec`。Anthropic 插件会规范化 Claude 的权限标志，以匹配生效的策略和主机限制。

在限制性策略下，Claude 使用其原生工具或扩展工具（其自带的 Bash、WebFetch 或 Claude in Chrome 浏览器工具）前，会通过 stdio 向 OpenClaw 请求许可。当生效的 Exec 询问设置为 `on-miss` 或 `always` 时，OpenClaw 会将每个请求作为交互式审批转发到会话所在渠道：**允许一次**允许单次调用，**始终允许**允许在当前 Claude 实时会话剩余时间内使用该工具名称（仅保存在内存中，绝不持久化），而**拒绝**、超时或无法访问审批路由都会拒绝该调用。永不提示的策略保持原有行为：`security: "deny"` 会拒绝所有请求，而当安全级别低于完全安全（Exec 模式为 `allowlist`）时，询问设置 `off` 会直接拒绝且不会询问。

### Claude 浏览器工具和 1Password 登录

Claude Code 可通过 [Claude in Chrome 扩展](https://code.claude.com/docs/en/chrome)操控 Chrome 浏览器，包括使用 [1Password for Claude](/zh-CN/gateway/1password#browser-sign-in-with-1password-for-claude)自动填充凭据。内置后端不会启用此功能；请注册一个 [CLI 后端插件](/zh-CN/plugins/cli-backend-plugins)，将 `--chrome` 追加到 `claude-stream-json` 方言后端的启动参数中。OpenClaw 会在常规运行中保留已配置的 `--chrome`，并始终在使用限制性工具策略的运行中强制设置 `--no-chrome`，例如旁支问题。Chrome 窗口、该扩展以及所有 1Password 审批提示均位于 Gateway 网关主机上，因此必须有人在该机器旁批准凭据使用。

该后端还会将 OpenClaw `/think` 级别映射到 Claude Code 的原生 `--effort` 标志：`minimal`/`low` -> `low`，`medium` -> `medium`，而 `high`/`xhigh`/`max` 会直接透传。这样可使订阅支持的 Claude CLI 路由和 API 密钥路由采用相同的受支持 Fable 5 工作量级别。`adaptive` 会移除已配置的 `--effort` 标志且不提供替代值，因此 Claude Code 会根据其自身环境、设置和模型默认值确定生效的工作量。其他 CLI 后端需要由其所属插件声明等效的 argv 映射器，之后 `/think` 才会影响生成的 CLI。

OpenClaw 使用 `claude-cli` 前，必须先在同一主机上登录 Claude Code：

```bash
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

Docker 安装需要在持久化容器主目录内安装并登录 Claude Code，而不能只在主机上完成；请参阅 [Docker 中的 Claude CLI 后端](/zh-CN/install/docker#claude-cli-backend-in-docker)。

Gateway 网关服务必须能在 `PATH` 中解析 `claude`。对于非标准路径，
请注册一个小型包装后端插件。

## 会话

- 如果 CLI 支持会话，请使用 `{sessionId}` 占位符设置 `sessionArgs`（例如 `["--session-id", "{sessionId}"]`）。
- 如果 CLI 使用具有不同标志的恢复子命令，请设置 `resumeArgs`（恢复时替换 `args`），并可选择为非 JSON 恢复设置 `resumeOutput`。
- `sessionMode`：
  - `always`：始终发送会话 ID（如果未存储，则使用新的 UUID）。
  - `existing`：仅当之前存储过会话 ID 时才发送。
  - `none`：永不发送会话 ID。
- `claude-cli` 默认为 `liveSession: "claude-stdio"`、`output: "jsonl"` 和 `input: "stdin"`，因此后续轮次会在实时 Claude 进程处于活动状态时复用该进程，包括省略传输字段的自定义配置。如果 Gateway 网关重启或空闲进程退出，OpenClaw 会从已存储的 Claude 会话 ID 恢复。恢复前会根据可读取的项目转录记录验证已存储的会话 ID；如果转录记录缺失，则会清除绑定（记录为 `reason=transcript-missing`），而不是在 `--resume` 下静默启动新会话。
- Claude 实时会话采用有界 JSONL 输出防护：每轮 8 MiB 和 20,000 行原始 JSONL。
- 已存储的 CLI 会话是由提供商负责的连续性状态。默认禁用自动重置；`/reset` 以及明确的每日或空闲 `session.reset` 策略仍会将其切断。
- 全新 CLI 会话通常仅根据 OpenClaw 的压缩摘要及压缩后的尾部内容重新注入上下文。为恢复在压缩前失效的短会话，后端可通过 `reseedFromRawTranscriptWhenUncompacted: true` 选择启用此功能。原始转录记录的重新注入始终有界，并仅限于安全的失效情况，例如 CLI 转录记录缺失、孤立的工具使用尾部、消息策略/系统提示词/cwd/MCP 发生变化，或会话过期后重试；身份验证配置文件或凭据纪元发生变化时，绝不会重新注入原始转录历史记录。

串行化：`serialize: true` 会保持同一通道中的运行有序（大多数 CLI 会在单个提供商通道上串行执行）。当所选身份验证身份发生变化时，OpenClaw 也会放弃复用已存储的 CLI 会话，包括身份验证配置文件 ID、静态 API 密钥、静态令牌发生变化，或 CLI 暴露 OAuth 账户身份时该身份发生变化；仅 OAuth 访问令牌/刷新令牌轮换不会切断会话。如果 CLI 没有稳定的 OAuth 账户 ID，OpenClaw 会让该 CLI 自行执行其恢复权限。

## claude-cli 会话的回退前置内容

当一次 `claude-cli` 尝试故障转移到 [`agents.defaults.model.fallbacks`](/zh-CN/concepts/model-failover) 中的非 CLI 候选项时，OpenClaw 会使用从 Claude Code 本地 JSONL 转录记录（位于 `~/.claude/projects/` 下，按工作区设键）中提取的上下文前导内容来初始化下一次尝试。如果没有此初始化内容，回退提供商将从空上下文开始，因为对于 `claude-cli` 运行，OpenClaw 自身的会话转录记录为空。

- 前导内容优先采用最新的 `/compact` 摘要或 `compact_boundary` 标记，然后在字符预算范围内追加边界之后最近的若干轮次。边界之前的轮次会被丢弃，因为摘要已涵盖这些内容。
- 工具块会被合并为紧凑的 `(tool call: name)` 和 `(tool result: …)` 提示，以确保提示词预算准确；过大的摘要会被截断并标记为 `(truncated)`。
- 从同一提供商的 `claude-cli` 到 `claude-cli` 的回退依赖 Claude 自身的 `--resume`，并跳过前导内容。
- 初始化内容复用现有的 Claude 会话文件路径验证，因此无法读取任意路径。

## 图像

插件作者通过 `imageArg` 声明图像路径支持：

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw 将 base64 图像写入临时文件。如果设置了 `imageArg`，这些路径将作为 CLI 参数传递；否则，OpenClaw 会将文件路径追加到提示词中（路径注入），这适用于能够从纯文本路径自动加载本地文件的 CLI。

## 输入和输出

- `output: "text"`（默认）将 stdout 视为最终响应。
- `output: "json"` 尝试解析 JSON，并提取文本和会话 ID。
- `output: "jsonl"` 解析 JSONL 流，并提取最终智能体消息以及存在的会话标识符。
- 对于 Gemini CLI JSON 输出，当 `usage` 缺失或为空时，OpenClaw 从 `response` 读取回复文本，并从 `stats` 读取用量。内置 Gemini CLI 适配器使用 `stream-json`。

输入模式：

- `input: "arg"`（默认）将提示词作为最后一个 CLI 参数传递。
- `input: "stdin"` 通过 stdin 发送提示词。
- 如果提示词很长且设置了 `maxPromptArgChars`，则改用 stdin。

## 插件自有默认值

CLI 后端默认值是插件表面的一部分：

- 插件通过 `api.registerCliBackend(...)` 注册这些默认值。
- 后端 `id` 将成为模型引用中的提供商前缀。
- 命令、argv、环境、解析器、会话和看门狗行为保留在插件代码中。
- 后端特定的规范化通过可选的 `normalizeConfig` 钩子继续由插件所有。

Anthropic 拥有 `claude-cli`，Google 拥有 `google-gemini-cli`。OpenAI Codex 智能体运行通过 `openai/*` 使用 Codex app-server harness；OpenClaw 不再注册内置的 `codex-cli` 后端。

内置 Anthropic 插件为 `claude-cli` 注册以下配置：

| 键                    | 值                                                                                                                                                                                                            |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`             | `claude`                                                                                                                                                                                                      |
| `args`                | `-p --output-format stream-json --include-partial-messages --verbose --setting-sources user --allowedTools mcp__openclaw__* --disallowedTools ScheduleWakeup,CronCreate,Bash(run_in_background:true),Monitor` |
| `output`              | `jsonl`                                                                                                                                                                                                       |
| `input`               | `stdin`                                                                                                                                                                                                       |
| `modelArg`            | `--model`                                                                                                                                                                                                     |
| `sessionArgs`         | `["--session-id", "{sessionId}"]`                                                                                                                                                                             |
| `sessionMode`         | `always`                                                                                                                                                                                                      |
| `imageArg`            | `@`                                                                                                                                                                                                           |
| `imagePathScope`      | `workspace`                                                                                                                                                                                                   |
| `systemPromptFileArg` | `--append-system-prompt-file`                                                                                                                                                                                 |
| `systemPromptMode`    | `append`                                                                                                                                                                                                      |

内置 Google 插件为 `google-gemini-cli` 注册以下配置：

| 键                        | 值                                                                                     |
| ------------------------- | -------------------------------------------------------------------------------------- |
| `command`                 | `gemini`                                                                               |
| `args`                    | `--skip-trust --approval-mode auto_edit --output-format stream-json --prompt {prompt}` |
| `resumeArgs`              | 相同，但带有 `--resume {sessionId}`                                                      |
| `output` / `resumeOutput` | `jsonl`                                                                                |
| `jsonlDialect`            | `gemini-stream-json`                                                                   |
| `imageArg`                | `@`                                                                                    |
| `imagePathScope`          | `workspace`                                                                            |
| `modelArg`                | `--model`                                                                              |
| `sessionMode`             | `existing`                                                                             |
| `sessionIdFields`         | `["session_id", "sessionId"]`                                                          |

前提条件：本地 Gemini CLI 必须已安装，并以 `gemini`（`brew install gemini-cli` 或 `npm install -g @google/gemini-cli`）的形式位于 `PATH` 中。

Gemini CLI 输出说明：

- 默认的 `stream-json` 解析器会读取助手 `message` 事件、工具事件、最终 `result` 用量以及致命 Gemini 错误事件。
- 当 `usage` 缺失或为空时，用量会回退到 `stats`；`stats.cached` 会规范化为 OpenClaw `cacheRead`，如果 `stats.input` 缺失，则根据 `stats.input_tokens - stats.cached` 推导输入 token 数。

## 文本转换叠加层

需要小型提示词/消息兼容性适配的插件可以声明双向文本转换，而无需替换提供商或 CLI 后端：

```typescript
api.registerTextTransforms({
  input: [{ from: /red basket/g, to: "blue basket" }],
  output: [{ from: /blue basket/g, to: "red basket" }],
});
```

`input` 会重写传递给 CLI 的系统提示词和用户提示词。`output` 会在 OpenClaw 处理自身控制标记和渠道投递之前，重写流式助手文本和解析后的最终文本；对于由提供商支持的模型调用，它还会在流修复之后、工具执行之前，恢复结构化工具调用参数中的字符串值。原始提供商 JSON 片段保持不变；使用方应使用结构化的部分、结束或结果载荷。

对于发出提供商特定 JSONL 事件的 CLI，请在该后端的配置中设置 `jsonlDialect`：Claude Code 兼容流使用 `claude-stream-json`，Gemini CLI `stream-json` 事件使用 `gemini-stream-json`。

## 原生压缩所有权

某些 CLI 后端会运行自行压缩其转录记录的智能体，因此 OpenClaw 不得对其运行安全保障摘要器——否则会与后端自身的压缩机制冲突，并可能导致该轮次彻底失败。

`claude-cli` 没有 harness 端点（Claude Code 在内部执行压缩），因此它声明 `ownsNativeCompaction: true`，OpenClaw 的压缩路径会原样返回会话条目。OpenClaw 通过 Claude Code 文档中说明的 [`CLAUDE_CODE_AUTO_COMPACT_WINDOW`](https://code.claude.com/docs/en/env-vars) 传递该次运行的有效上下文预算，使原生自动压缩与配置的 Anthropic `contextTokens` 限制保持一致。Codex 等原生 harness 会话则继续路由到其 harness 压缩端点。

```typescript
api.registerCliBackend({ id: "my-cli", ownsNativeCompaction: true /* ... */ });
```

仅为真正拥有压缩机制的后端声明 `ownsNativeCompaction`：它必须能够在上下文窗口附近可靠地限制自身转录记录，并持久化可恢复的会话（例如 `--resume` / `--session-id`），否则延迟的会话可能持续超出预算。

## 内置 MCP 叠加层

CLI 后端不会直接接收 OpenClaw 工具调用，但后端可以通过 `bundleMcp: true` 选择启用生成的 MCP 配置叠加层。当前内置行为：

- `claude-cli`：生成的严格 MCP 配置文件。
- `google-gemini-cli`：生成的 Gemini 系统设置文件。

启用内置 MCP 后，OpenClaw 会：

- 生成一个回环 HTTP MCP 服务器，向 CLI 进程公开 Gateway 网关工具，并使用仅在当前执行尝试期间有效的单次运行上下文授权（`OPENCLAW_MCP_TOKEN`）进行身份验证；
- 将工具访问绑定到 Gateway 网关所选的会话、账户和渠道上下文，而不是信任子进程标头；
- 加载当前工作区中已启用的内置 MCP 服务器，并将其与任何现有的后端 MCP 配置/设置结构合并；
- 使用所属插件拥有的后端集成模式重写启动配置。

受限运行（例如使用 `toolsAllow` 的定时任务）需要由后端提供精确转换。内置的 `claude-cli` 后端会禁用 Claude 的原生工具，以及用户、项目和本地自定义项，包括钩子、插件、智能体、Skills 和 `CLAUDE.md`。然后，它通过受授权范围约束的 MCP 服务器公开所有允许使用的 OpenClaw 工具。这样，文件系统、进程、Exec、审批和沙箱策略都保留在 OpenClaw 内部，而不会将权限扩大到 Claude 的原生工具或自定义进程。同一 MCP 列表会在 Claude 生成的配置中强制执行，并由 Gateway 网关在工具列出和执行时再次强制执行。在签发授权之前，核心会拒绝任何包含原始允许列表之外 MCP 权限的后端转换。无法提供精确转换的后端仍会以关闭方式失败。

如果未启用任何 MCP 服务器，当后端选择使用内置 MCP 时，OpenClaw 仍会注入严格配置，使后台运行保持隔离。

会话范围的内置 MCP 运行时会被缓存，以便在会话内复用，并在空闲 10 分钟后回收。一次性嵌入式运行（例如身份验证探测、slug 生成和主动记忆检索）会请求在运行结束时进行清理，确保 stdio 子进程和可流式 HTTP/SSE 流不会在运行结束后继续存在。

对于 `claude-cli`，系统会将兼容且已选定或已排序的 OpenClaw OAuth/令牌配置文件转发给该 Claude 子进程。这样，每个智能体的配置文件便会成为该轮次的权威配置，同时在不存在兼容配置文件时保留 Claude 的原生主机登录状态。

## 重新播种历史记录上限

当新的 CLI 会话基于先前的 OpenClaw 记录进行播种时（例如在 `session_expired` 重试后），系统会限制渲染后的 `<conversation_history>` 块大小，防止重新播种提示急剧膨胀。默认上限为 12,288 个字符（约 3,000 个令牌）。

Claude CLI 后端会根据解析后的 Claude 上下文窗口调整此上限：上下文窗口越大，可包含的先前历史记录片段就越大，但不会超过固定上限；其他 CLI 后端则继续使用较为保守的默认值。此上限仅控制重新播种提示中的先前历史记录块。

## 限制

- OpenClaw 不会将工具调用注入 CLI 后端协议。后端只有在选择使用 `bundleMcp: true` 时才能看到 Gateway 网关工具。
- 流式传输行为因后端而异：部分后端以 JSONL 方式流式传输，其他后端则缓冲到进程退出。
- 结构化输出取决于 CLI 自身的 JSON 格式。

## 故障排查

| 症状                  | 解决方法                                                                                                        |
| --------------------- | --------------------------------------------------------------------------------------------------------------- |
| 找不到 CLI            | 将 CLI 添加到 Gateway 网关服务的 `PATH` 中，或更新所属插件注册的命令。                              |
| 模型名称错误          | 更新插件的 `modelAliases` 映射。                                                                            |
| 会话无法延续          | 检查插件的 `sessionArgs` 和 `sessionMode`。                                                           |
| 图像被忽略            | 检查插件的 `imageArg` 以及 CLI 的文件路径支持。                                                         |

## 相关内容

- [Gateway 网关运行手册](/zh-CN/gateway)
- [本地模型](/zh-CN/gateway/local-models)
