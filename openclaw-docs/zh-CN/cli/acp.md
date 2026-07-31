---
read_when:
    - 设置基于 ACP 的 IDE 集成
    - 调试 ACP 会话到 Gateway 网关的路由
summary: 为 IDE 集成运行 ACP 桥接器
title: ACP
x-i18n:
    generated_at: "2026-07-26T06:39:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: becdcfdd1cc62b206cc92e9b8248c79a2ff63cfc3779d8a124b9713e779ad33c
    source_path: cli/acp.md
    workflow: 16
---

运行与 OpenClaw Gateway 网关通信的 [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) 桥接器。

`openclaw acp` 通过 stdio 与 IDE 进行 ACP 通信，并通过 WebSocket 将提示转发到 Gateway 网关，同时保持 ACP 会话与 Gateway 网关会话键的映射。它是由 Gateway 网关支持的 ACP 桥接器，而不是完整的 ACP 原生编辑器运行时：它专注于会话路由、提示传递和流式更新。

如果你希望外部 MCP 客户端直接与 OpenClaw 渠道会话通信，而不是托管 ACP harness 会话，请改用 [`openclaw mcp serve`](/zh-CN/cli/mcp)。

## 这不是什么

`openclaw acp` 表示 OpenClaw 充当 ACP 服务器：IDE 或 ACP 客户端连接到 OpenClaw，而 OpenClaw 将该工作转发到 Gateway 网关会话中。

这不同于 [ACP 智能体](/zh-CN/tools/acp-agents)，后者由 OpenClaw 通过 `acpx` 运行 Codex 或 Claude Code 等外部 harness。

快速判断规则：

- 编辑器/客户端希望通过 ACP 与 OpenClaw 通信：使用 `openclaw acp`
- OpenClaw 应将 Codex/Claude/Gemini 作为 ACP harness 启动：使用 `/acp spawn` 和 [ACP 智能体](/zh-CN/tools/acp-agents)

## 兼容性矩阵

| ACP 领域                                                               | 状态        | 说明                                                                                                                                                                                                                                  |
| --------------------------------------------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `initialize`、`newSession`、`prompt`、`cancel`                        | 已实现 | 通过 stdio 连接到 Gateway 网关 chat/send + abort 的核心桥接流程。                                                                                                                                                                             |
| `listSessions`、斜杠命令                                        | 已实现 | 会话列表基于 Gateway 网关会话状态运行，使用有界游标分页；当 Gateway 网关会话行带有工作区元数据时，支持 `cwd` 筛选；命令通过 `available_commands_update` 发布。                     |
| 会话沿袭关系元数据                                              | 已实现 | 会话列表和会话信息快照在 `_meta` 中包含 OpenClaw 父子沿袭关系，使 ACP 客户端无需使用 Gateway 网关私有旁路即可呈现子智能体图。                                                     |
| `resumeSession`、`closeSession`                                       | 已实现 | 恢复操作将 ACP 会话重新绑定到现有 Gateway 网关会话，但不重放历史记录。关闭操作会取消活跃的桥接工作，将待处理提示解析为已取消，并释放桥接会话状态。                                   |
| `loadSession`                                                         | 部分支持     | 将 ACP 会话重新绑定到 Gateway 网关会话键，并为桥接器创建的会话重放 ACP 事件账本历史记录。较旧或没有账本的会话会回退到存储的用户/助手文本。                                                  |
| 提示内容（`text`、嵌入式 `resource`、图像）                  | 部分支持     | 文本/资源会被展平为聊天输入；图像会成为 Gateway 网关附件。                                                                                                                                                            |
| 会话模式                                                         | 部分支持     | 支持 `session/set_mode`；该桥接器提供由 Gateway 网关支持的会话控件，用于思考级别、工具详细程度、推理、用量详情和提升权限的操作。更广泛的 ACP 原生模式/配置界面仍不在范围内。 |
| 思考过程流式传输                                                     | 已实现 | 模型思考内容以 `agent_thought_chunk` 会话更新的形式流式传输。不发送 ACP 原生会话计划。                                                                                                                    |
| 会话信息和用量更新                                        | 部分支持     | 该桥接器根据缓存的 Gateway 网关会话快照发送 `session_info_update` 和尽力而为的 `usage_update` 通知。用量为近似值，并且仅在 Gateway 网关令牌总数被标记为最新时发送。                             |
| 工具流式传输                                                        | 部分支持     | 当 Gateway 网关工具参数/结果公开相关信息时，`tool_call`/`tool_call_update` 事件会包含原始 I/O、文本内容以及尽力而为获取的文件位置。不提供嵌入式终端和更丰富的原生差异输出。                     |
| Exec 审批                                                        | 部分支持     | 活跃 ACP 提示轮次期间的 Gateway 网关 Exec 审批提示会通过 `session/request_permission` 中继到 ACP 客户端。                                                                                                               |
| 每会话 MCP 服务器（`mcpServers`）                                | 不支持 | 桥接模式拒绝每会话 MCP 服务器请求。请改为在 OpenClaw Gateway 网关或智能体上配置 MCP。                                                                                                                          |
| 客户端文件系统方法（`fs/read_text_file`、`fs/write_text_file`） | 不支持 | 该桥接器不调用 ACP 客户端文件系统方法。                                                                                                                                                                               |
| 客户端终端方法（`terminal/*`）                                | 不支持 | 该桥接器不会创建 ACP 客户端终端，也不会通过工具调用传输终端 ID。                                                                                                                                            |

## 已知限制

- `loadSession` 仅对桥接器创建的会话重放完整的 ACP 事件账本历史记录。较旧或没有账本的会话使用对话记录回退机制，且不会重建历史工具调用或系统通知。
- 如果多个 ACP 客户端共享同一个 Gateway 网关会话键，事件和取消操作的路由仅为尽力而为，无法严格做到按客户端隔离。需要清晰的编辑器本地轮次时，建议使用默认隔离的 `acp-bridge:<uuid>` 会话。
- Gateway 网关停止状态会转换为 ACP 停止原因，但这种映射的表达能力不如完全原生的 ACP 运行时。
- 会话控件仅提供一组精简的 Gateway 网关选项：思考级别、工具详细程度、推理、用量详情和提升权限的操作。模型选择和 Exec 主机控件不会作为 ACP 配置选项提供。
- `session_info_update` 和 `usage_update` 源自 Gateway 网关会话快照，而非实时 ACP 原生运行时计量。用量为近似值，不包含成本数据，并且仅在 Gateway 网关将令牌总数数据标记为最新时发送。
- 工具跟随数据为尽力而为：桥接器会提供已知工具参数/结果中出现的文件路径，但不会发送 ACP 终端或结构化文件差异。
- Exec 审批中继仅限活跃的 ACP 提示轮次；来自其他 Gateway 网关会话的审批会被忽略。

## 用法

```bash
openclaw acp

# 远程 Gateway 网关
openclaw acp --url wss://gateway-host:18789 --token <token>

# 远程 Gateway 网关（从文件读取令牌）
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# 附加到现有会话键
openclaw acp --session agent:main:main

# 按标签附加（必须已存在）
openclaw acp --session-label "support inbox"

# 在第一个提示前重置会话键
openclaw acp --session agent:main:main --reset-session
```

## ACP 客户端（调试）

使用内置 ACP 客户端，无需 IDE 即可对桥接器执行完整性检查。它会生成 ACP 桥接器，并允许你以交互方式输入提示。

```bash
openclaw acp client

# 让生成的桥接器连接到远程 Gateway 网关
openclaw acp client --server-args --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# 覆盖服务器命令（默认值：openclaw）
openclaw acp client --server "node" --server-args openclaw.mjs acp --url ws://127.0.0.1:19001
```

权限模型（客户端调试模式）：

- 自动审批基于允许列表，并且仅适用于受信任的核心工具 ID。
- `read` 自动审批仅限当前工作目录（设置时为 `--cwd`）。
- ACP 仅自动批准范围狭窄的只读类别：活跃 cwd 下限定范围的 `read` 调用，以及只读搜索工具（`search`、`web_search`、`memory_search`）。未知/非核心工具、超出范围的读取、可执行命令的工具、控制平面工具、修改型工具和交互式流程始终需要明确的提示审批。
- 服务器提供的 `toolCall.kind` 被视为不受信任的元数据，而不是授权来源。
- 此 ACP 桥接策略独立于 ACPX harness 权限。如果通过 `acpx` 后端运行 OpenClaw，`plugins.entries.acpx.config.permissionMode=approve-all` 是该 harness 会话的紧急“yolo”开关。

## 协议冒烟测试

若要进行协议级调试，请使用隔离状态启动 Gateway 网关，并通过 ACP JSON-RPC 客户端经由 stdio 驱动 `openclaw acp`。测试应覆盖 `initialize`、`session/new`、带有绝对 `cwd` 的 `session/list`、`session/resume`、`session/close`、重复关闭和缺失的恢复目标。

证明材料应包含发布的生命周期能力、由 Gateway 网关支持的会话行、更新通知以及 Gateway 网关 `sessions.list` 日志：

```json
{
  "initialize": {
    "protocolVersion": 1,
    "agentCapabilities": {
      "sessionCapabilities": {
        "list": {},
        "resume": {},
        "close": {}
      }
    }
  },
  "listSessions": {
    "sessions": [
      {
        "sessionId": "agent:main:acp-smoke",
        "cwd": "/path/to/workspace",
        "_meta": {
          "sessionKey": "agent:main:acp-smoke",
          "kind": "direct"
        }
      }
    ],
    "nextCursor": null
  },
  "notifications": ["session_info_update", "available_commands_update", "usage_update"],
  "gatewayLogTail": ["[gateway] ready", "[ws] ⇄ res ✓ sessions.list 305ms"]
}
```

避免将 `openclaw gateway call sessions.list` 作为唯一的 ACP 证明。该 CLI 路径可能会请求全新令牌的操作员权限范围升级；ACP 桥接器的正确性应通过 ACP stdio 帧和 Gateway 网关 `sessions.list` 日志来证明。

## 如何使用

当 IDE（或其他客户端）使用 Agent Client Protocol，并且你希望它驱动 OpenClaw Gateway 网关会话时，请使用 ACP。

1. 确保 Gateway 网关正在运行（本地或远程）。
2. 配置 Gateway 网关目标（通过配置或标志）。
3. 将 IDE 配置为通过 stdio 运行 `openclaw acp`。

配置示例（持久化）：

```bash
openclaw config set gateway.remote.url wss://gateway-host:18789
openclaw config set gateway.remote.token <token>
```

直接运行示例（不写入配置）：

```bash
openclaw acp --url wss://gateway-host:18789 --token <token>
# 为确保本地进程安全，建议使用此方式
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token
```

## 选择智能体

ACP 不直接选择智能体。它通过 Gateway 网关会话键进行路由。使用智能体范围的会话键来指定特定智能体：

```bash
openclaw acp --session agent:main:main
openclaw acp --session agent:design:main
openclaw acp --session agent:qa:bug-123
```

每个 ACP 会话映射到一个 Gateway 网关会话键。一个智能体可以有多个会话；除非你覆盖会话键或标签，否则 ACP 默认使用隔离的 `acp-bridge:<uuid>` 会话。

桥接模式不支持每会话 `mcpServers`。如果 ACP 客户端在 `newSession` 或 `loadSession` 期间发送这些内容，桥接器会返回明确的错误，而不是静默忽略。

如果你希望由 ACPX 支持的会话可以使用 OpenClaw 插件工具或 `cron` 等选定的内置工具，请启用 Gateway 网关侧的 ACPX MCP 桥接，而不要尝试传递每会话 `mcpServers`。请参阅 [ACP 智能体](/zh-CN/tools/acp-agents-setup#plugin-tools-mcp-bridge)和 [OpenClaw 工具 MCP 桥接](/zh-CN/tools/acp-agents-setup#openclaw-tools-mcp-bridge)。

## 从 `acpx` 使用（Codex、Claude 及其他 ACP 客户端）

如果你希望 Codex 或 Claude Code 等编码智能体通过 ACP 与你的 OpenClaw bot 通信，请使用 `acpx` 及其内置的 `openclaw` 目标。

典型流程：

1. 运行 Gateway 网关，并确保 ACP 桥接可以连接到它。
2. 将 `acpx openclaw` 指向 `openclaw acp`。
3. 指定你希望编码智能体使用的 OpenClaw 会话键。

示例：

```bash
# 向默认 OpenClaw ACP 会话发送一次性请求
acpx openclaw exec "总结活跃 OpenClaw 会话的状态。"

# 用于后续轮次的持久命名会话
acpx openclaw sessions ensure --name codex-bridge
acpx openclaw -s codex-bridge --cwd /path/to/repo \
  "向我的 OpenClaw 工作智能体询问与此仓库相关的近期上下文。"
```

如果你希望 `acpx openclaw` 每次都以特定的 Gateway 网关和会话键为目标，请在 `~/.acpx/config.json` 中覆盖 `openclaw` 智能体命令：

```json
{
  "agents": {
    "openclaw": {
      "command": "env OPENCLAW_HIDE_BANNER=1 OPENCLAW_SUPPRESS_NOTES=1 openclaw acp --url ws://127.0.0.1:18789 --token-file ~/.openclaw/gateway.token --session agent:main:main"
    }
  }
}
```

对于仓库本地的 OpenClaw 检出，请使用直接的 CLI 入口点，而不是开发运行器，以保持 ACP 流干净：

```bash
env OPENCLAW_HIDE_BANNER=1 OPENCLAW_SUPPRESS_NOTES=1 node openclaw.mjs acp ...
```

这是让 Codex、Claude Code 或其他支持 ACP 的客户端从 OpenClaw 智能体获取上下文信息，而无需抓取终端内容的最简便方式。

## Zed 编辑器设置

在 `~/.config/zed/settings.json` 中添加自定义 ACP 智能体（或使用 Zed 的 Settings UI）：

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": ["acp"],
      "env": {}
    }
  }
}
```

要指定特定的 Gateway 网关或智能体：

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": [
        "acp",
        "--url",
        "wss://gateway-host:18789",
        "--token",
        "<token>",
        "--session",
        "agent:design:main"
      ],
      "env": {}
    }
  }
}
```

在 Zed 中，打开 Agent 面板并选择 "OpenClaw ACP" 以启动线程。

## 会话映射

默认情况下，ACP 桥接会话会获得一个带有 `acp-bridge:` 前缀的隔离 Gateway 网关会话键。这些普通模型桥接会话是合成且可丢弃的：它们会受到陈旧条目清理的影响，并且不会被视为受保护的人工对话界面。要复用已知会话，请传递会话键或标签：

- `--session <key>`：使用特定的 Gateway 网关会话键。
- `--session-label <label>`：按标签解析现有会话。
- `--reset-session`：为该键生成新的会话 ID（键相同，记录文本为新内容）。

如果你的 ACP 客户端支持元数据，可以按会话覆盖：

```json
{
  "_meta": {
    "sessionKey": "agent:main:main",
    "sessionLabel": "support inbox",
    "resetSession": true
  }
}
```

要详细了解会话键，请参阅 [/concepts/session](/zh-CN/concepts/session)。

## 选项

- `--url <url>`：Gateway 网关 WebSocket URL（配置后默认为 `gateway.remote.url`）。
- `--token <token>`：Gateway 网关身份验证令牌。
- `--token-file <path>`：从文件读取 Gateway 网关身份验证令牌。
- `--password <password>`：Gateway 网关身份验证密码。
- `--password-file <path>`：从文件读取 Gateway 网关身份验证密码。
- `--session <key>`：默认会话键。
- `--session-label <label>`：要解析的默认会话标签。
- `--require-existing`：如果会话键/标签不存在，则失败。
- `--reset-session`：首次使用前重置会话键。
- `--no-prefix-cwd`：不在提示词前添加工作目录。
- `--provenance <off|meta|meta+receipt>`：包含 ACP 来源元数据或回执。
- `--verbose, -v`：将详细日志写入 stderr。

安全说明：

- 在某些系统上，`--token` 和 `--password` 可能会显示在本地进程列表中。优先使用 `--token-file`/`--password-file` 或环境变量（`OPENCLAW_GATEWAY_TOKEN`、`OPENCLAW_GATEWAY_PASSWORD`）。
- Gateway 网关身份验证解析遵循其他 Gateway 网关客户端使用的共享约定：
  - 本地模式：先使用环境变量（`OPENCLAW_GATEWAY_*`），再使用 `gateway.auth.*`；仅当 `gateway.auth.*` 未设置时才回退到 `gateway.remote.*`（已配置但无法解析的本地 SecretRef 会以关闭方式失败，而不是静默回退）
  - 远程模式：使用 `gateway.remote.*`，并按照远程优先级规则进行环境变量/配置回退
  - `--url` 可安全覆盖，并且不会复用隐式配置/环境变量凭据；请传递显式的 `--token`/`--password`（或文件变体）

### `acp client` 选项

- `--cwd <dir>`：ACP 会话的工作目录。
- `--server <command>`：ACP 服务器命令（默认值：`openclaw`）。
- `--server-args <args...>`：传递给 ACP 服务器的额外参数。
- `--server-verbose`：在 ACP 服务器上启用详细日志。
- `--verbose, -v`：详细客户端日志。
- `openclaw acp client` 会在生成的桥接进程上设置 `OPENCLAW_SHELL=acp-client`，可用于特定上下文的 shell/profile 规则。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [ACP 智能体](/zh-CN/tools/acp-agents)
