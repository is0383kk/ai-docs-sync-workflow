---
read_when:
    - 安装或配置用于 Claude Code / Codex / Gemini CLI 的 acpx harness
    - 启用 plugin-tools 或 OpenClaw-tools MCP 桥接器
    - 配置 ACP 权限模式
summary: 设置 ACP 智能体：acpx harness 配置、插件设置和权限
title: ACP Agents 设置
x-i18n:
    generated_at: "2026-07-26T07:02:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ae3750092175b44252dd080717a1af176995df43c653f245f82d7e556cfd25eb
    source_path: tools/acp-agents-setup.md
    workflow: 16
---

有关概览、操作员运行手册和概念，请参阅 [ACP 智能体](/zh-CN/tools/acp-agents)。

本页介绍 acpx harness 配置、MCP 桥接的插件设置以及权限配置。

仅在设置 ACP/acpx 路由时使用本页。对于原生 Codex
app-server 运行时配置，请使用 [Codex harness](/zh-CN/plugins/codex-harness)。对于
OpenAI API 密钥或 Codex OAuth 模型提供商配置，请使用
[OpenAI](/zh-CN/providers/openai)。

Codex 有两种 OpenClaw 路由：

| 路由                       | 配置/命令                                              | 设置页面                                |
| -------------------------- | ------------------------------------------------------ | --------------------------------------- |
| 原生 Codex app-server      | `/codex ...`、`openai/gpt-*` 智能体引用                 | [Codex harness](/zh-CN/plugins/codex-harness) |
| 显式 Codex ACP 适配器      | `/acp spawn codex`、`runtime: "acp", agentId: "codex"` | 本页                                    |

除非明确需要 ACP/acpx 行为，否则优先使用原生路由。

## acpx harness 支持（当前）

内置 acpx harness 别名（来自固定版本的 `acpx` 依赖项）：

| 别名         | 封装                                                                                                            |
| ------------ | --------------------------------------------------------------------------------------------------------------- |
| `claude`     | [Claude Code](https://claude.ai/code)                                                                           |
| `codex`      | [Codex CLI](https://codex.openai.com)                                                                           |
| `copilot`    | [GitHub Copilot CLI](https://docs.github.com/copilot/how-tos/copilot-chat/use-copilot-chat-in-the-command-line) |
| `cursor`     | [Cursor CLI](https://cursor.com/docs/cli/acp)（`cursor-agent acp`）                                             |
| `droid`      | [Factory Droid](https://www.factory.ai)                                                                         |
| `fast-agent` | [fast-agent](https://fast-agent.ai)                                                                             |
| `gemini`     | [Gemini CLI](https://github.com/google/gemini-cli)                                                              |
| `iflow`      | [iFlow CLI](https://github.com/iflow-ai/iflow-cli)                                                              |
| `kilocode`   | [Kilocode](https://kilocode.ai)                                                                                 |
| `kimi`       | [Kimi CLI](https://github.com/MoonshotAI/kimi-cli)                                                              |
| `kiro`       | [Kiro CLI](https://kiro.dev)                                                                                    |
| `mux`        | [Mux](https://mux.coder.com)                                                                                    |
| `opencode`   | [OpenCode](https://opencode.ai)                                                                                 |
| `openclaw`   | OpenClaw ACP 桥接（原生 `openclaw acp`）                                                                    |
| `pi`         | [Pi Coding Agent](https://github.com/mariozechner/pi)                                                           |
| `qoder`      | [Qoder CLI](https://docs.qoder.com/cli/acp)                                                                     |
| `qwen`       | [Qwen Code](https://github.com/QwenLM/qwen-code)                                                                |
| `trae`       | [Trae CLI](https://docs.trae.cn/cli)                                                                            |

`factory-droid` 和 `factorydroid` 也会解析为内置的 `droid` 适配器。

当 OpenClaw 使用 acpx 后端时，除非你的 acpx 配置定义了自定义智能体别名，否则请优先为 `agentId` 使用这些值。
如果本地 Cursor 安装仍以 `agent acp` 的形式公开 ACP，请在 acpx 配置中覆盖 `cursor` 智能体命令，而不是更改内置默认值。

直接使用 acpx CLI 时，也可以通过 `--agent <command>` 指定任意适配器，但这个原始逃生口是 acpx CLI 的功能（并非正常的 OpenClaw `agentId` 路径）。

模型控制取决于适配器能力。OpenClaw 会在启动前规范化 Codex ACP 模型引用。其他 harness 需要 ACP `models` 以及
`session/set_model` 支持；如果 harness 既不公开该 ACP 能力，也没有自身的启动模型标志，OpenClaw/acpx 就无法强制选择模型。

## 必需配置

核心 ACP 基线配置：

```json5
{
  acp: {
    enabled: true,
    // 可选。默认为 true；设为 false 可暂停 ACP 分派，同时保留 /acp 控制功能。
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "codex",
    allowedAgents: [
      "claude",
      "codex",
      "copilot",
      "cursor",
      "droid",
      "gemini",
      "iflow",
      "kilocode",
      "kimi",
      "kiro",
      "openclaw",
      "opencode",
      "qwen",
    ],
    stream: {
      deliveryMode: "live",
    },
  },
}
```

支持的渠道适配器共享线程绑定配置：

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
      spawnSessions: true,
    },
  },
}
```

如果线程绑定的 ACP 生成无法工作，请先验证适配器功能标志：

- Discord：`session.threadBindings.spawnSessions=true`

当前对话绑定不需要创建子线程。它们需要活跃的对话上下文，以及公开 ACP 对话绑定的渠道适配器。

请参阅[配置参考](/zh-CN/gateway/configuration-reference)。

## acpx 后端的插件设置

打包安装使用官方 `@openclaw/acpx` 运行时插件来支持 ACP。
使用 ACP harness 会话之前，请先安装并启用它：

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

执行 `pnpm install` 后，源码检出也可以使用本地工作区插件。

首先运行：

```text
/acp doctor
```

如果你禁用了 `acpx`，通过 `plugins.allow` / `plugins.deny` 拒绝了它，或者想切换回打包插件，请使用显式软件包路径：

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

开发期间安装本地工作区插件：

```bash
openclaw plugins install ./path/to/local/acpx-plugin
```

然后验证后端健康状态：

```text
/acp doctor
```

### acpx 运行时启动探测

`acpx` 插件直接嵌入 ACP 运行时（无需配置单独的 `acpx` 二进制文件或
版本）。默认情况下，它会在 Gateway 网关启动期间注册嵌入式后端，并在发出 Gateway 网关 `ready`
信号前等待启动探测。仅应在有意禁用启动探测的脚本或环境中设置 `OPENCLAW_ACPX_RUNTIME_STARTUP_PROBE=0` 或
`OPENCLAW_SKIP_ACPX_RUNTIME_PROBE=1`。运行 `/acp doctor` 可执行显式的
按需探测。

当路径或标志值应作为单个 argv 令牌保留时，可使用结构化参数覆盖单个 ACP 智能体命令：

```json
{
  "plugins": {
    "entries": {
      "acpx": {
        "enabled": true,
        "config": {
          "agents": {
            "claude": {
              "command": "node",
              "args": ["/path/to/custom adapter.mjs", "--verbose"]
            }
          }
        }
      }
    }
  }
}
```

- `agents.<id>.command` 是该 ACP 智能体的可执行文件或现有命令字符串。
- `agents.<id>.args` 是可选项。在 OpenClaw 通过当前 acpx 命令字符串注册表传递各数组项前，会分别对其进行 shell 引用。

请参阅[插件](/zh-CN/tools/plugin)。

### 自动下载适配器

`acpx` 会在首次使用时通过 `npx` 自动下载 ACP 适配器（例如 Claude 和 Codex ACP
桥接）。无需手动安装适配器软件包，OpenClaw 本身也没有单独的 postinstall 步骤。如果
适配器下载或生成失败，`/acp doctor` 会报告失败。

### 插件工具 MCP 桥接

默认情况下，ACPX 会话**不会**向 ACP harness 公开由 OpenClaw 插件注册的工具。

如果希望 Codex 或 Claude Code 等 ACP 智能体能够调用已安装的
OpenClaw 插件工具（例如记忆检索/存储），请启用专用桥接：

```bash
openclaw config set plugins.entries.acpx.config.pluginToolsMcpBridge true
```

此功能会：

- 将名为 `openclaw-plugin-tools` 的内置 MCP 服务器注入 ACPX 会话
  引导过程。
- 公开已由安装并启用的 OpenClaw
  插件注册的插件工具。
- 将活跃 ACP 会话身份传递给插件工具工厂，使
  智能体范围的工具保留在该智能体的命名空间中。
- 确保该功能需要显式启用且默认关闭。

安全与信任注意事项：

- 此功能会扩大 ACP harness 的工具范围。
- ACP 智能体只能访问 Gateway 网关中已处于活动状态的插件工具。
- 应将其视为与允许这些插件在
  OpenClaw 自身内部执行相同的信任边界。
- 启用前请审查已安装的插件。

自定义 `mcpServers` 仍会像以前一样工作。内置插件工具桥接是一项额外的可选便利功能，并非通用 MCP 服务器配置的替代品。

### OpenClaw 工具 MCP 桥接

默认情况下，ACPX 会话也**不会**通过
MCP 公开内置 OpenClaw 工具。当 ACP 智能体需要 `cron` 等选定的
内置工具时，请启用单独的核心工具桥接：

```bash
openclaw config set plugins.entries.acpx.config.openClawToolsMcpBridge true
```

此功能会：

- 将名为 `openclaw-tools` 的内置 MCP 服务器注入 ACPX 会话
  引导过程。
- 公开选定的内置 OpenClaw 工具。初始服务器公开 `cron`。
- 确保核心工具公开功能需要显式启用且默认关闭。

### 运行时操作超时配置

`acpx` 插件默认给予嵌入式运行时启动和控制操作 120
秒。这使 Gemini CLI 等较慢的 harness 有足够时间
完成 ACP 启动和初始化。如果主机需要不同的操作限制，请覆盖此设置：

```bash
openclaw config set plugins.entries.acpx.config.timeoutSeconds 180
```

运行时轮次使用 OpenClaw 智能体/运行超时，包括 `/acp timeout`。
`sessions_spawn` 不接受单次调用的超时覆盖；操作员应使用
`agents.defaults.subagents.runTimeoutSeconds`。更改
`timeoutSeconds` 后，请重启 Gateway 网关。

### 健康探测智能体配置

当 `/acp doctor` 或启动探测检查后端时，内置 `acpx`
插件会探测一个 harness 智能体。如果设置了 `acp.allowedAgents`，则默认为
第一个允许的智能体；否则默认为 `codex`。如果部署
需要使用其他 ACP 智能体进行健康检查，请显式设置探测智能体：

```bash
openclaw config set plugins.entries.acpx.config.probeAgent claude
```

更改此值后，请重启 Gateway 网关。

## 权限配置

ACP 会话以非交互方式运行——没有 TTY 可用于批准或拒绝文件写入和 shell 执行权限提示。acpx 插件提供两个配置键，用于控制权限的处理方式：

这些 ACPX harness 权限独立于 OpenClaw Exec 审批，也独立于 CLI 后端供应商的绕过标志，例如 Claude CLI `--permission-mode bypassPermissions`。ACPX `approve-all` 是 ACP 会话在 harness 层级的紧急解锁开关。

有关 OpenClaw `tools.exec.mode`、Codex Guardian
审批与 ACPX harness 权限之间更全面的比较，请参阅
[权限模式](/zh-CN/tools/permission-modes)。

### `permissionMode`

控制 harness 智能体无需提示即可执行哪些操作。

| 值           | 行为                                                  |
| --------------- | --------------------------------------------------------- |
| `approve-all`   | 自动批准所有文件写入和 shell 命令。          |
| `approve-reads` | 仅自动批准读取；写入和执行需要提示。 |
| `deny-all`      | 拒绝所有权限提示。                              |

### `nonInteractivePermissions`

控制当本应显示权限提示但没有可用的交互式 TTY 时会发生什么（ACP 会话始终如此）。

| 值  | 行为                                                                 |
| ------ | ------------------------------------------------------------------------ |
| `fail` | 以 `PermissionPromptUnavailableError` 中止会话。**（默认）** |
| `deny` | 静默拒绝该权限并继续（优雅降级）。        |

### 配置

通过插件配置进行设置：

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

更改这些值后，重启 Gateway 网关。

<Warning>
OpenClaw 默认为 `permissionMode=approve-reads` 和 `nonInteractivePermissions=fail`。在非交互式 ACP 会话中，任何触发权限提示的写入或执行操作都可能因 `PermissionPromptUnavailableError: Permission prompt unavailable in non-interactive mode` 而失败。

如果需要限制权限，请将 `nonInteractivePermissions` 设置为 `deny`，使会话优雅降级而不是崩溃。
</Warning>

## 相关内容

- [ACP 智能体](/zh-CN/tools/acp-agents) — 概览、操作员运行手册、概念
- [子智能体](/zh-CN/tools/subagents)
- [多智能体路由](/zh-CN/concepts/multi-agent)
