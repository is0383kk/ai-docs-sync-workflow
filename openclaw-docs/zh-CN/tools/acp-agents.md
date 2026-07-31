---
read_when:
    - 通过 ACP 运行编码 harness
    - 在消息渠道上设置与对话绑定的 ACP 会话
    - 将消息渠道对话绑定到持久 ACP 会话
    - ACP 后端、插件连接或完成结果交付的故障排查
    - 在聊天中操作 /acp 命令
sidebarTitle: ACP agents
summary: 通过 ACP 后端运行外部编码 harness（Claude Code、Cursor、Gemini CLI、显式 Codex ACP、OpenClaw ACP、OpenCode）
title: ACP 智能体
x-i18n:
    generated_at: "2026-07-26T06:02:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fc7f32ff927c7e949be1595f6aa00ed034a51185c6a6b1e0df01a242954667d1
    source_path: tools/acp-agents.md
    workflow: 16
---

[Agent Client Protocol (ACP)](https://agentclientprotocol.com/) 会话让
OpenClaw 能够通过 ACP 后端插件运行外部编码 harness（Claude Code、Cursor、Copilot、Droid、
OpenClaw ACP、OpenCode、Gemini CLI 以及其他受支持的 ACPX harness）。
每次生成都会作为[后台任务](/zh-CN/automation/tasks)进行跟踪。

<Note>
**ACP 是外部 harness 路径，而不是默认的 Codex 路径。** 原生
Codex app-server 插件负责 `/codex ...` 控制以及用于智能体轮次的默认
`openai/gpt-*` 嵌入式运行时；ACP 负责 `/acp ...` 控制
和 `sessions_spawn({ runtime: "acp" })` 会话。

若要让 Codex 或 Claude Code 作为外部 MCP 客户端直接连接到
现有 OpenClaw 渠道对话，请使用
[`openclaw mcp serve`](/zh-CN/cli/mcp)，而不是 ACP。
</Note>

## 我需要哪个页面？

| 你想要……                                                                                       | 使用此项                              | 说明                                                                                                                                                                                |
| ----------------------------------------------------------------------------------------------- | ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 在当前对话中绑定或控制 Codex                                                                    | `/codex bind`、`/codex threads`       | 启用 `codex` 插件时使用原生 Codex app-server 路径：绑定聊天回复、图像转发、模型/快速模式/权限、停止和 Steer。ACP 是显式后备方案 |
| 通过 OpenClaw 运行 Claude Code、Gemini CLI、显式 Codex ACP 或其他外部 harness                   | 本页                                  | 与聊天绑定的会话、`/acp spawn`、`sessions_spawn({ runtime: "acp" })`、后台任务、运行时控制                                                                 |
| 将 OpenClaw Gateway 网关会话作为 ACP 服务器公开给编辑器或客户端                                 | [`openclaw acp`](/zh-CN/cli/acp)            | 桥接模式：IDE/客户端通过 stdio/WebSocket 使用 ACP 与 OpenClaw 通信                                                                                                                  |
| 将本地 AI CLI 复用为纯文本后备模型                                                              | [CLI 后端](/zh-CN/gateway/cli-backends)     | 不是 ACP：没有 OpenClaw 工具、ACP 控制或 harness 运行时                                                                                                                             |

## 是否可以开箱即用？

可以，安装官方 ACP 运行时插件后即可：

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

源代码检出在 `pnpm install` 后可以使用本地 `extensions/acpx` 工作区插件。
运行 `/acp doctor` 进行就绪检查。

只有当 ACP **真正可用**时，OpenClaw 才会向智能体说明如何生成 ACP：
必须启用 ACP、不得禁用分派、当前会话不得被沙箱阻止，并且必须已加载
且后端运行时健康。如果任何条件不满足，ACP Skills 和
`sessions_spawn` ACP 指引将保持隐藏，以免智能体建议不可用的后端。

<AccordionGroup>
  <Accordion title="首次运行注意事项">
    - 如果设置了 `plugins.allow`，它就是限制性的插件清单，并且**必须**包含 `acpx`，否则已安装的 ACP 后端会被有意阻止（`/acp doctor` 会报告缺失的允许列表条目）。
    - Codex ACP 适配器随 `acpx` 插件提供，并会尽可能在本地启动。
    - Codex ACP 使用隔离的 `CODEX_HOME` 运行。OpenClaw 会从宿主 Codex 配置中复制可信项目的信任条目以及安全的模型/提供商路由配置（`model`、`model_provider`、`model_reasoning_effort`、`sandbox_mode` 和安全的 `model_providers.<name>` 字段）；身份验证、通知和钩子仅保留在宿主配置中。
    - 首次使用其他目标 harness 适配器时，可能会按需通过 `npx` 获取。
    - 宿主上必须已存在该 harness 的供应商身份验证。
    - 如果宿主没有 npm 或网络访问权限，首次运行时获取适配器会失败，直到预热缓存或以其他方式安装适配器。

  </Accordion>
  <Accordion title="运行时前提条件">
    ACP 会启动真实的外部 harness 进程。OpenClaw 负责路由、
    后台任务状态、投递、绑定和策略；harness 负责其
    提供商登录、模型目录、文件系统行为和原生工具。

    在归咎于 OpenClaw 之前，请验证：

    - `/acp doctor` 报告后端已启用且健康。
    - 设置 `acp.allowedAgents` 允许列表时，目标 ID 在其允许范围内。
    - harness 命令可以在 Gateway 网关主机上启动。
    - 该 harness 已配置提供商身份验证（`claude`、`codex`、`gemini`、`opencode`、`droid` 等）。
    - 所选模型在该 harness 中存在——模型 ID 无法跨 harness 通用。
    - 请求的 `cwd` 存在且可访问，或者省略 `cwd`，让后端使用其默认值。
    - 权限模式与工作相匹配。非交互式会话无法点击原生权限提示，因此大量涉及写入/执行的编码运行通常需要能够无头运行的 ACPX 权限配置文件。

  </Accordion>
</AccordionGroup>

默认情况下，OpenClaw 插件工具和 OpenClaw 内置工具**不会**向 ACP
harness 公开。仅当 harness 应直接调用这些工具时，才在
[ACP 智能体设置](/zh-CN/tools/acp-agents-setup)中启用显式 MCP 桥接。

## 支持的 harness 目标

使用 `acpx` 后端时，将以下 ID 用作 `/acp spawn <id>` 或
`sessions_spawn({ runtime: "acp", agentId: "<id>" })` 目标：

| Harness ID   | 典型后端                                       | 说明                                                                                         |
| ------------ | ---------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `claude`     | Claude Code ACP 适配器                         | 需要宿主上的 Claude Code 身份验证。                                                          |
| `codex`      | Codex ACP 适配器                               | 仅当原生 `/codex` 不可用或明确请求 ACP 时，才作为显式 ACP 后备方案。                |
| `copilot`    | GitHub Copilot ACP 适配器                      | 需要 Copilot CLI/运行时身份验证。                                                            |
| `cursor`     | Cursor CLI ACP（`cursor-agent acp`）           | 如果本地安装公开了不同的 ACP 入口点，请覆盖 acpx 命令。                                      |
| `droid`      | Factory Droid CLI                              | 需要 Factory/Droid 身份验证或 harness 环境中的 `FACTORY_API_KEY`。                          |
| `fast-agent` | fast-agent-mcp ACP 适配器                      | 通过 `uvx` 按需获取。                                                           |
| `gemini`     | Gemini CLI ACP 适配器                          | 需要 Gemini CLI 身份验证或 API 密钥设置。                                                    |
| `iflow`      | iFlow CLI                                      | 适配器可用性和模型控制取决于已安装的 CLI。                                                   |
| `kilocode`   | Kilo Code CLI                                  | 适配器可用性和模型控制取决于已安装的 CLI。                                                   |
| `kimi`       | Kimi/Moonshot CLI                              | 需要宿主上的 Kimi/Moonshot 身份验证。                                                        |
| `kiro`       | Kiro CLI                                       | 适配器可用性和模型控制取决于已安装的 CLI。                                                   |
| `mux`        | Mux CLI ACP 适配器                             | 通过 `npx` 按需获取。                                                           |
| `opencode`   | OpenCode ACP 适配器                            | 需要 OpenCode CLI/提供商身份验证。                                                           |
| `openclaw`   | 通过 `openclaw acp` 的 OpenClaw Gateway 网关桥接 | 让支持 ACP 的 harness 与 OpenClaw Gateway 网关会话通信。                                     |
| `qoder`      | Qoder CLI                                      | 适配器可用性和模型控制取决于已安装的 CLI。                                                   |
| `qwen`       | Qwen Code / Qwen CLI                           | 需要宿主上的 Qwen 兼容身份验证。                                                             |
| `trae`       | Trae CLI ACP 适配器                            | 适配器可用性和模型控制取决于已安装的 CLI。                                                   |

`pi`（pi-acp）也注册在 acpx 后端中，但与上述其他项目不同，
它并非同类的编码 harness。

可以在 acpx 本身中配置自定义 acpx 智能体别名，但 OpenClaw
策略在分派前仍会检查 `acp.allowedAgents` 以及任何
`agents.entries.*.runtime.acp.agent` 映射。

## 操作员运行手册

从聊天开始的快速 `/acp` 流程：

<Steps>
  <Step title="生成">
    `/acp spawn claude --bind here`、
    `/acp spawn gemini --mode persistent --thread auto`，或显式
    `/acp spawn codex --bind here`。
  </Step>
  <Step title="工作">
    在绑定的对话或话题串中继续（或显式指定会话键）。
  </Step>
  <Step title="检查状态">
    `/acp status`
  </Step>
  <Step title="调整">
    `/acp model <provider/model>`、`/acp permissions <profile>`、
    `/acp timeout <seconds>`。
  </Step>
  <Step title="Steer">
    在不替换上下文的情况下：`/acp steer tighten logging and continue`。
  </Step>
  <Step title="停止">
    `/acp cancel`（当前轮次）或 `/acp close`（会话 + 绑定）。
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="生命周期详情">
    - 生成操作会创建或恢复 ACP 运行时会话，在 OpenClaw 会话存储中记录 ACP 元数据，并且当运行由父任务所有时，可能会创建后台任务。
    - 由父任务所有的 ACP 会话会被视为后台工作，即使运行时会话是持久会话；完成通知和跨界面交付会通过父任务通知器进行，而不会像普通的面向用户的聊天会话那样处理。
    - 任务维护会关闭已终止或失去父任务的、由父任务所有的一次性 ACP 会话。只要仍存在活跃的对话绑定，持久 ACP 会话就会被保留；没有活跃绑定的陈旧持久会话会被关闭，以免在所有者任务完成或其任务记录消失后被静默恢复。
    - 绑定后的后续消息会直接发送到 ACP 会话，直到绑定被关闭、取消聚焦、重置或过期。
    - Gateway 网关命令保持在本地处理。`/acp ...`、`/status` 和 `/unfocus` 绝不会作为普通提示文本发送到已绑定的 ACP harness。
    - `cancel` 会在后端支持取消时中止当前轮次；它不会删除绑定或会话元数据。
    - `close` 会从 OpenClaw 的角度结束 ACP 会话并移除绑定。如果 harness 支持恢复，它仍可能保留自己的上游历史记录。
    - acpx 插件会在 `close` 后清理 OpenClaw 所有的包装器和适配器进程树，并在 Gateway 网关启动期间清除陈旧的、由 OpenClaw 所有的 ACPX 孤儿进程。
    - 空闲运行时工作进程在达到内置空闲时限后可被清理；存储的会话元数据仍可供 `/acp sessions` 使用。

  </Accordion>
  <Accordion title="Native Codex 路由规则">
    启用 **Native Codex plugins** 后，应路由到该插件的自然语言触发语：

    - “将此 Discord 频道绑定到 Codex。”
    - “将此聊天附加到 Codex 线程 `<id>`。”
    - “显示 Codex 线程，然后绑定这一个。”

    Native Codex 对话绑定是默认的聊天控制路径。
    OpenClaw 动态工具仍通过 OpenClaw 执行，而 shell/apply-patch 等 Codex 原生
    工具则在 Codex 内部执行。对于 Codex 原生工具事件，OpenClaw 会注入按轮次的
    原生钩子中继，使插件钩子能够阻止 `before_tool_call`、观察 `after_tool_call`，
    并通过 OpenClaw 审批路由 Codex `PermissionRequest` 事件。Codex `Stop`
    钩子会中继到 OpenClaw `before_agent_finalize`，插件可在 Codex 最终确定回答之前
    请求再进行一次模型调用。该中继有意保持保守：它不会修改 Codex 原生工具参数，
    也不会重写 Codex 线程记录。仅当需要 ACP 运行时/会话模型时，才显式使用 ACP。
    嵌入式 Codex 的支持边界记录在
    [Codex harness v1 支持契约](/zh-CN/plugins/codex-harness-runtime#v1-support-contract)中。

  </Accordion>
  <Accordion title="模型 / 提供商 / 运行时选择速查表">
    - 旧版 Codex 模型引用 — 由 Doctor 修复的旧版 Codex OAuth/订阅模型路由。
    - `openai/*` — 用于 OpenAI 智能体轮次的 Native Codex app-server 嵌入式运行时。
    - `/codex ...` — Native Codex 对话控制。
    - `/acp ...` 或 `runtime: "acp"` — 显式 ACP/acpx 控制。

  </Accordion>
  <Accordion title="ACP 路由自然语言触发语">
    应路由到 ACP 运行时的触发语：

    - “将此任务作为一次性 Claude Code ACP 会话运行，并总结结果。”
    - “在线程中使用 Gemini CLI 完成此任务，然后让后续消息继续使用同一线程。”
    - “通过 ACP 在后台线程中运行 Codex。”

    OpenClaw 会选择 `runtime: "acp"`、解析 harness `agentId`，
    在支持时绑定到当前对话或线程，并将后续消息路由到该会话，直到会话关闭或过期。
    仅当明确指定 ACP/acpx，或者 Native Codex plugins 无法用于所请求的操作时，
    Codex 才会遵循此路径。

    对于 `sessions_spawn`，仅当 ACP 已启用、请求方未处于沙箱隔离状态且已加载
    ACP 运行时后端时，才会公开 `runtime: "acp"`。`acp.dispatch.enabled=false` 会暂停
    ACP 线程的自动分派，但不会隐藏或阻止显式 `sessions_spawn({ runtime: "acp" })` 调用。
    它面向 `codex`、`claude`、`droid`、
    `gemini` 或 `opencode` 等 ACP harness ID。不要传递
    `agents_list` 中的普通 OpenClaw 配置智能体 ID，除非该条目已显式配置
    `agents.entries.*.runtime.type="acp"`；否则应使用默认子智能体运行时。当 OpenClaw 智能体配置了
    `runtime.type="acp"` 时，OpenClaw 会使用 `runtime.acp.agent` 作为底层 harness ID。

  </Accordion>
</AccordionGroup>

## ACP 与子智能体的对比

需要外部 harness 运行时时使用 ACP。当 `codex` 插件已启用时，
使用 **Native Codex app-server** 进行 Codex 对话绑定/控制。需要 OpenClaw
原生委派运行时使用**子智能体**。

| 范畴          | ACP 会话                               | 子智能体运行                       |
| ------------- | -------------------------------------- | ---------------------------------- |
| 运行时        | ACP 后端插件（例如 acpx）              | OpenClaw 原生子智能体运行时        |
| 会话键        | `agent:<agentId>:acp:<uuid>`                     | `agent:<agentId>:subagent:<uuid>`                 |
| 主要命令      | `/acp ...`                     | `/subagents ...`                 |
| 生成工具      | 带 `runtime:"acp"` 的 `sessions_spawn` | `sessions_spawn`（默认运行时） |

另请参阅[子智能体](/zh-CN/tools/subagents)。

## ACP 如何运行 Claude Code

对于通过 ACP 运行的 Claude Code，其技术栈为：

1. OpenClaw ACP 会话控制平面。
2. 官方 `@openclaw/acpx` 运行时插件。
3. Claude ACP 适配器。
4. Claude 端运行时/会话机制。

ACP Claude 是一个具有 ACP 控制、会话恢复、后台任务跟踪以及可选对话/线程绑定的
**harness 会话**。

CLI 后端是独立的纯文本本地回退运行时 — 请参阅
[CLI 后端](/zh-CN/gateway/cli-backends)。

对于操作员，实用规则如下：

- **需要 `/acp spawn`、可绑定会话、运行时控制或持久 harness 工作？** 使用 ACP。
- **需要通过原始 CLI 进行简单的本地文本回退？** 使用 CLI 后端。

## 已绑定会话

### 心智模型

- **聊天界面** — 用户持续交谈的位置（Discord 频道、Telegram 话题、iMessage 聊天）。
- **ACP 会话** — OpenClaw 路由到的持久 Codex/Claude/Gemini 运行时状态。
- **子线程/话题** — 仅由 `--thread ...` 创建的可选额外消息界面。
- **运行时工作区** — harness 运行所在的文件系统位置（`cwd`、仓库检出目录、后端工作区）。它独立于聊天界面。

### 当前对话绑定

`/acp spawn <harness> --bind here` 会将当前对话固定到已生成的 ACP 会话 — 不创建子线程，
继续使用同一聊天界面。OpenClaw 继续负责传输、身份验证、安全和交付。
该对话中的后续消息会路由到同一会话；`/new` 和
`/reset` 会原地重置会话；`/acp close` 会移除绑定。

示例：

```text
/codex bind                                              # 原生 Codex 绑定，将后续消息路由到此处
/codex model gpt-5.4                                     # 调整已绑定的原生 Codex 线程
/codex stop                                              # 控制当前原生 Codex 轮次
/acp spawn codex --bind here                             # Codex 的显式 ACP 回退
/acp spawn codex --thread auto                           # 可能创建子线程/话题并绑定到其中
/acp spawn codex --bind here --cwd /workspace/repo       # 使用同一聊天绑定，Codex 在 /workspace/repo 中运行
```

<AccordionGroup>
  <Accordion title="绑定规则和互斥性">
    - `--bind here` 和 `--thread ...` 互斥。
    - `--bind here` 仅适用于声明支持当前对话绑定的渠道；否则 OpenClaw 会返回明确的不支持消息。绑定在 Gateway 网关重启后仍然保留。
    - 在 Discord 上，`spawnSessions` 控制 `--thread auto|here` 的子线程创建，而不控制 `--bind here`。
    - 如果未使用 `--cwd` 生成到其他 ACP 智能体，OpenClaw 默认继承**目标智能体的**工作区。缺失的继承路径（`ENOENT`/`ENOTDIR`）会回退到后端默认值；其他访问错误（例如 `EACCES`）则会作为生成错误显示。
    - Gateway 网关管理命令在已绑定对话中保持本地处理 — 即使普通后续文本会路由到已绑定的 ACP 会话，`/acp ...` 命令仍由 OpenClaw 处理；只要该界面启用了命令处理，`/status` 和 `/unfocus` 也始终保持本地处理。

  </Accordion>
  <Accordion title="线程绑定会话">
    为渠道适配器启用线程绑定后：

    - OpenClaw 将线程绑定到目标 ACP 会话。
    - 该线程中的后续消息会路由到已绑定的 ACP 会话。
    - ACP 输出会交付回同一线程。
    - 取消聚焦、关闭、归档、空闲超时或最长存续期到期会移除绑定。
    - `/acp close`、`/acp cancel`、`/acp status`、`/status` 和 `/unfocus` 是 Gateway 网关命令，而不是发给 ACP harness 的提示。

    线程绑定 ACP 所需的功能标志：

    - `acp.enabled=true`
    - `acp.dispatch.enabled` 默认开启（将 `false` 设置为暂停 ACP 线程自动分派；显式 `sessions_spawn({ runtime: "acp" })` 调用仍然有效）。
    - 启用渠道适配器线程会话生成（默认值：`true`）：
      - Discord/Telegram：`session.threadBindings.spawnSessions=true`

    线程绑定支持取决于具体适配器。如果当前渠道适配器不支持线程绑定，
    OpenClaw 会返回明确的不支持/不可用消息。

  </Accordion>
  <Accordion title="支持线程的渠道">
    - 任何公开会话/线程绑定能力的渠道适配器。
    - 当前内置支持：**Discord** 线程/频道、**Telegram** 话题（群组/超级群组中的论坛话题及私信话题）。
    - 插件渠道可以通过相同的绑定接口添加支持。

  </Accordion>
</AccordionGroup>

## 持久渠道绑定

对于非临时工作流，请在顶层 `bindings[]` 条目中配置持久 ACP 绑定。

### 绑定模型

<ParamField path="bindings[].type" type='"acp"'>
  标记持久 ACP 对话绑定。
</ParamField>
<ParamField path="bindings[].match" type="object">
  标识目标对话。各渠道的结构如下：

- **Discord 频道/线程：** `match.channel="discord"` + `match.peer.id="<channelOrThreadId>"`
- **Slack 频道/私信：** `match.channel="slack"` + `match.peer.id="<channelId|channel:<channelId>|#<channelId>|userId|user:<userId>|slack:<userId>|<@userId>>"`。优先使用稳定的 Slack ID；频道绑定也会匹配该频道线程中的回复。
- **Telegram 论坛话题：** `match.channel="telegram"` + `match.peer.id="<chatId>:topic:<topicId>"`
- **WhatsApp 私信/群组：** `match.channel="whatsapp"` + `match.peer.id="<E.164|group JID>"`。直接聊天使用 E.164 号码，例如 `+15555550123`；群组使用 WhatsApp 群组 JID，例如 `120363424282127706@g.us`。
- **iMessage 私信/群组：** `match.channel="imessage"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`。对于稳定的群组绑定，优先使用 `chat_id:*`。

</ParamField>
<ParamField path="bindings[].agentId" type="string">
  所属 OpenClaw 智能体的 ID。
</ParamField>
<ParamField path="bindings[].acp.mode" type='"persistent" | "oneshot"'>
  可选的 ACP 覆盖设置。
</ParamField>
<ParamField path="bindings[].acp.label" type="string">
  可选的面向操作员的标签。
</ParamField>
<ParamField path="bindings[].acp.cwd" type="string">
  可选的运行时工作目录。
</ParamField>
<ParamField path="bindings[].acp.backend" type="string">
  可选的后端覆盖设置。
</ParamField>

### 每个智能体的运行时默认值

使用 `agents.entries.*.runtime` 为每个智能体统一定义 ACP 默认值：

- `agents.entries.*.runtime.type="acp"`
- `agents.entries.*.runtime.acp.agent`（harness ID，例如 `codex` 或 `claude`）
- `agents.entries.*.runtime.acp.backend`
- `agents.entries.*.runtime.acp.mode`
- `agents.entries.*.runtime.acp.cwd`

**ACP 绑定会话的覆盖优先级：**

1. `bindings[].acp.*`
2. `agents.entries.*.runtime.acp.*`
3. 全局 ACP 默认值（例如 `acp.backend`）

### 示例

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
      {
        id: "claude",
        runtime: {
          type: "acp",
          acp: { agent: "claude", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
    {
      type: "acp",
      agentId: "claude",
      match: {
        channel: "telegram",
        accountId: "default",
        peer: { kind: "group", id: "-1001234567890:topic:42" },
      },
      acp: { cwd: "/workspace/repo-b" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "discord", accountId: "default" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "telegram", accountId: "default" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": { requireMention: false },
          },
        },
      },
    },
    telegram: {
      groups: {
        "-1001234567890": {
          topics: { "42": { requireMention: false } },
        },
      },
    },
  },
}
```

### 行为

- OpenClaw 会在通过特定渠道的准入检查后、使用前，确保配置的 ACP 会话存在。
- 该频道、话题或聊天中的消息会路由到配置的 ACP 会话。
- 配置的 ACP 绑定拥有其会话路由。对于匹配的绑定，频道广播的扇出不会取代配置的 ACP 会话。
- 在绑定的对话中，`/new` 和 `/reset` 会原地重置同一个 ACP 会话键。
- 临时运行时绑定（例如由线程聚焦流程创建的绑定）在存在时仍会生效。
- 对于未显式指定 `cwd` 的跨智能体 ACP 生成，OpenClaw 会从智能体配置继承目标智能体工作区。
- 继承的工作区路径不存在时，会回退到后端默认 cwd；路径存在但访问失败时，会显示为生成错误。

## 启动 ACP 会话

启动 ACP 会话有两种方式：

<Tabs>
  <Tab title="通过 sessions_spawn">
    使用 `runtime: "acp"` 从智能体轮次或工具调用中启动 ACP 会话。

    ```json
    {
      "task": "打开仓库并总结失败的测试",
      "runtime": "acp",
      "agentId": "codex",
      "thread": true,
      "mode": "session"
    }
    ```

    <Note>
    `runtime` 默认为 `subagent`，因此 ACP 会话需要显式设置 `runtime: "acp"`。如果省略 `agentId`，OpenClaw 会在已配置时使用 `acp.defaultAgent`。`mode: "session"` 需要 `thread: true` 才能维持持久绑定的对话。
    </Note>

  </Tab>
  <Tab title="通过 /acp 命令">
    使用 `/acp spawn` 从聊天中进行显式的操作员控制。

    ```text
    /acp spawn codex --mode persistent --thread auto
    /acp spawn codex --mode oneshot --thread off
    /acp spawn codex --bind here
    /acp spawn codex --thread here
    ```

    关键标志：

    - `--mode persistent|oneshot`
    - `--bind here|off`
    - `--thread auto|here|off`
    - `--cwd <absolute-path>`
    - `--label <name>`

    参见[斜杠命令](/zh-CN/tools/slash-commands)。

  </Tab>
</Tabs>

### `sessions_spawn` 参数

<ParamField path="task" type="string" required>
  发送给 ACP 会话的初始提示词。
</ParamField>
<ParamField path="runtime" type='"acp"' required>
  ACP 会话必须设置为 `"acp"`。
</ParamField>
<ParamField path="agentId" type="string">
  ACP 目标 harness ID。如果已设置，则回退到 `acp.defaultAgent`。
</ParamField>
<ParamField path="thread" type="boolean" default="false">
  在支持的情况下请求线程绑定流程。
</ParamField>
<ParamField path="mode" type='"run" | "session"' default="run">
  `"run"` 为一次性模式；`"session"` 为持久模式。如果设置了 `thread: true` 且省略 `mode`，OpenClaw 可能会根据运行时路径默认采用持久行为。`mode: "session"` 需要 `thread: true`。
</ParamField>
<ParamField path="cwd" type="string">
  请求的运行时工作目录（由后端/运行时策略验证）。
  如果省略，且目标智能体工作区已配置，ACP 生成会继承该工作区；
  继承的路径不存在时会回退到后端默认值，而实际访问错误
  会原样返回。
</ParamField>
<ParamField path="label" type="string">
  会话/横幅文本中使用的面向操作员的标签。
</ParamField>
<ParamField path="resumeSessionId" type="string">
  恢复现有 ACP 会话，而不是创建新会话。智能体会通过 `session/load` 重放其对话历史。需要 `runtime: "acp"`。
</ParamField>
<ParamField path="streamTo" type='"parent"'>
  `"parent"` 会将初始 ACP 运行进度摘要作为系统事件流式传回请求方会话。OpenClaw 会在子智能体的 SQLite 状态中记录完整的中继历史，并在删除子会话时一并删除。除非设置了 `streaming.progress.commentary=false`，否则父会话的进度流默认显示助手评注和 ACP 状态进度。在未配置流模式时，Discord 的父会话预览也默认使用进度模式。状态进度仍遵循 `acp.stream.tagVisibility`，因此 `plan` 等标签会保持隐藏，除非显式启用。
</ParamField>

ACP `sessions_spawn` 运行使用 `agents.defaults.subagents.runTimeoutSeconds` 作为其默认子轮次限制。该工具不接受每次调用的超时覆盖设置（`runTimeoutSeconds`/`timeoutSeconds` 会被拒绝，并返回要求配置默认值的错误）。

<ParamField path="model" type="string">
  ACP 子会话的显式模型覆盖设置。Codex ACP 生成会在 `session/new` 之前，将 `openai/gpt-5.4` 等 OpenAI 引用规范化为 Codex ACP 启动配置；`openai/gpt-5.4/high` 等斜杠形式还会设置 Codex ACP 推理强度。如果省略，`sessions_spawn({ runtime: "acp" })` 会在已配置时使用现有的子智能体模型默认值（`agents.defaults.subagents.model` 或 `agents.entries.*.subagents.model`）；否则由 ACP harness 使用其自身的默认模型。其他 harness 必须声明 ACP `models` 并支持 `session/set_model`；否则 OpenClaw/acpx 会明确报错，而不会静默回退到目标智能体的默认值。
</ParamField>
<ParamField path="thinking" type="string">
  显式的思考/推理强度。对于 Codex ACP，`minimal` 映射为低强度，`low`/`medium`/`high`/`xhigh` 直接映射，`off` 则省略启动时的推理强度覆盖设置。如果省略，ACP 生成会使用现有的子智能体思考默认值，以及所选模型的按模型 `agents.defaults.models["provider/model"].params.thinking`。
</ParamField>

## 生成绑定和线程模式

<Tabs>
  <Tab title="--bind here|off">
    | 模式   | 行为                                                               |
    | ------ | ----------------------------------------------------------------------- |
    | `here` | 原地绑定当前活动对话；如果没有活动对话则失败。 |
    | `off`  | 不创建当前对话绑定。                          |

    注意事项：

    - `--bind here` 是实现“让此频道或聊天由 Codex 提供支持”的最简单操作员路径。
    - `--bind here` 不会创建子线程。
    - `--bind here` 仅适用于提供当前对话绑定支持的渠道。
    - `--bind` 和 `--thread` 不能在同一次 `/acp spawn` 调用中组合使用。

  </Tab>
  <Tab title="--thread auto|here|off">
    | 模式   | 行为                                                                                            |
    | ------ | ------------------------------------------------------------------------------------------------- |
    | `auto` | 位于活动线程中时：绑定该线程。位于线程外时：在支持的情况下创建并绑定子线程。 |
    | `here` | 要求存在当前活动线程；如果不在线程中则失败。                                                  |
    | `off`  | 不绑定。会话以未绑定状态启动。                                                                 |

    注意事项：

    - 在不支持线程绑定的界面上，默认行为实际上等同于 `off`。
    - 线程绑定生成需要渠道策略支持：
      - Discord/Telegram：`session.threadBindings.spawnSessions=true`
    - 如果要固定当前对话而不创建子线程，请使用 `--bind here`。

  </Tab>
</Tabs>

## 交付模型

ACP 会话既可以是交互式工作区，也可以是由父会话拥有的后台工作。交付路径取决于其形态。

<AccordionGroup>
  <Accordion title="交互式 ACP 会话">
    交互式会话旨在持续通过可见的聊天界面进行对话：

    - `/acp spawn ... --bind here` 将当前对话绑定到 ACP 会话。
    - `/acp spawn ... --thread ...` 将频道线程/话题绑定到 ACP 会话。
    - 持久配置的 `bindings[].type="acp"` 会将匹配的对话路由到同一个 ACP 会话。

    绑定对话中的后续消息会直接路由到 ACP 会话，ACP 输出也会返回到同一个频道/线程/话题。

    OpenClaw 发送给 harness 的内容：

    - 常规的绑定后续消息以提示文本形式发送；仅当 harness/后端支持时，才会同时发送附件。
    - `/acp` 管理命令和本地 Gateway 网关命令会在 ACP 分派前被拦截。
    - 运行时生成的完成事件会按目标具体化。OpenClaw 智能体会收到 OpenClaw 的内部运行时上下文信封；外部 ACP harness 会收到包含子项结果和指令的纯文本提示。绝不能将原始 `<<<BEGIN_OPENCLAW_INTERNAL_CONTEXT>>>` 信封发送到外部 harness，也不能将其作为 ACP 用户转录文本持久化。
    - ACP 转录条目使用用户可见的触发文本或纯文本完成提示。内部事件元数据会尽可能在 OpenClaw 中保持结构化，不会被视为用户撰写的聊天内容。

  </Accordion>
  <Accordion title="父级拥有的一次性 ACP 会话">
    由另一个智能体运行生成的一次性 ACP 会话属于后台子项，
    类似于子智能体：

    - 父级通过 `sessions_spawn({ runtime: "acp", mode: "run" })` 请求执行工作。
    - 子项在其自己的 ACP harness 会话中运行。
    - 子项轮次在原生子智能体生成所用的同一后台通道上运行，因此缓慢的 ACP harness 不会阻塞无关的主会话工作。
    - 完成报告通过任务完成通知路径返回。OpenClaw 在将内部完成元数据发送到外部 harness 前，会将其转换为纯文本 ACP 提示，因此 harness 不会看到仅供 OpenClaw 使用的运行时上下文标记。
    - 当需要向用户回复时，父级会以正常的助手口吻重写子项结果。

    **不要**将此路径视为父级与子项之间的点对点聊天。
    子项已有将完成结果返回父级的渠道。

  </Accordion>
  <Accordion title="sessions_send 和 A2A 交付">
    `sessions_send` 可以在生成后以另一个会话为目标。对于常规对等
    会话，OpenClaw 会在注入消息后使用智能体到智能体（A2A）后续路径：

    - 等待目标会话的回复。
    - 可以选择让请求方和目标进行有限次数的后续轮次交流。
    - 要求目标生成一条通知消息。
    - 将该通知交付到可见渠道或线程。

    此 A2A 路径是对等发送中发送方需要可见后续消息时的回退路径。
    当无关会话能够看到 ACP 目标并向其发送消息时，该路径仍会启用，
    例如在宽泛的 `tools.sessions.visibility` 设置下。

    仅当请求方是其自行拥有、由父级管理的一次性 ACP 子项的父级时，
    OpenClaw 才会跳过 A2A 后续操作。在这种情况下，如果在任务完成之上
    再运行 A2A，可能会用子项结果唤醒父级、将父级回复转发回子项，
    并形成父级/子项回声循环。对于这种自有子项情况，
    `sessions_send` 结果会报告 `delivery.status="skipped"`，
    因为完成路径已经负责返回结果。

  </Accordion>
  <Accordion title="恢复现有会话">
    使用 `resumeSessionId` 继续之前的 ACP 会话，而不是重新开始。
    智能体会通过 `session/load` 重放其对话历史记录，
    因此可以带着之前的完整上下文继续工作。

    ```json
    {
      "task": "从我们上次停下的位置继续——修复剩余的测试失败",
      "runtime": "acp",
      "agentId": "codex",
      "resumeSessionId": "<previous-session-id>"
    }
    ```

    常见使用场景：

    - 将 Codex 会话从笔记本电脑移交到手机——让你的智能体从上次停下的位置继续。
    - 继续你之前在 CLI 中以交互方式启动的编码会话，现在通过你的智能体以无头方式运行。
    - 继续因 Gateway 网关重启或空闲超时而中断的工作。

    注意：

    - `resumeSessionId` 仅在 `runtime: "acp"` 时适用；默认子智能体运行时会忽略这个仅供 ACP 使用的字段。
    - `streamTo` 仅在 `runtime: "acp"` 时适用；默认子智能体运行时会忽略这个仅供 ACP 使用的字段。
    - `resumeSessionId` 是主机本地的 ACP/harness 恢复 ID，而不是 OpenClaw 渠道会话键；OpenClaw 在分派前仍会检查 ACP 生成策略和目标智能体策略，而 ACP 后端或 harness 负责授权加载该上游 ID。
    - `resumeSessionId` 会恢复上游 ACP 对话历史记录；`thread` 和 `mode` 仍会正常应用于你正在创建的新 OpenClaw 会话，因此 `mode: "session"` 仍要求 `thread: true`。
    - 目标智能体必须支持 `session/load`（Codex 和 Claude Code 均支持）。
    - 如果找不到会话 ID，生成操作会失败并返回明确错误，不会静默回退到新会话。

  </Accordion>
  <Accordion title="部署后冒烟测试">
    部署 Gateway 网关后，应运行实时端到端检查，而不是依赖
    单元测试：

    1. 在目标主机上验证已部署的 Gateway 网关版本和提交。
    2. 打开一个连接到实时智能体的临时 ACPX 桥接会话。
    3. 要求该智能体使用 `runtime: "acp"`、`agentId: "codex"`、`mode: "run"` 和任务 `Reply with exactly LIVE-ACP-SPAWN-OK` 调用 `sessions_spawn`。
    4. 验证 `accepted=yes`、一个真实的 `childSessionKey`，并确认没有验证器错误。
    5. 清理临时桥接会话。

    将门禁保持在 `mode: "run"`，并跳过 `streamTo: "parent"`——
    线程绑定的 `mode: "session"` 和流中继路径是独立且更丰富的
    集成验证流程。

  </Accordion>
</AccordionGroup>

## 沙箱兼容性

ACP 会话目前在主机运行时中运行，**而不是**在 OpenClaw
沙箱内运行。

<Warning>
**安全边界：**

- 外部 harness 可以根据其自身的 CLI 权限和所选 `cwd` 进行读写。
- OpenClaw 的沙箱策略**不会**封装 ACP harness 执行。
- OpenClaw 仍会强制执行 ACP 功能门禁、允许的智能体、会话所有权、渠道绑定和 Gateway 网关交付策略。
- 需要由沙箱强制执行的 OpenClaw 原生工作时，请使用 `runtime: "subagent"`。

</Warning>

当前限制：

- 如果请求方会话已进行沙箱隔离，则 `sessions_spawn({ runtime: "acp" })` 和 `/acp spawn` 的 ACP 生成都会被阻止。
- 使用 `runtime: "acp"` 的 `sessions_spawn` 不支持 `sandbox: "require"`。

## 会话目标解析

大多数 `/acp` 操作接受可选会话目标（`session-key`、
`session-id` 或 `session-label`）。

**解析顺序：**

1. 显式目标参数（对于 `/acp steer`，则为 `--session`）
   - 先尝试键
   - 然后尝试 UUID 形式的会话 ID
   - 然后尝试标签
2. 当前线程绑定（如果此对话/线程已绑定到 ACP 会话）。
3. 回退到当前请求方会话。

当前对话绑定和线程绑定都会参与第 2 步。

如果无法解析任何目标，OpenClaw 会返回明确错误
（`Unable to resolve session target: ...`）。

## ACP 控制

| 命令                 | 作用                                                      | 示例                                                          |
| -------------------- | --------------------------------------------------------- | ------------------------------------------------------------- |
| `/acp spawn`         | 创建 ACP 会话；可选择绑定当前会话或线程。                 | `/acp spawn codex --bind here --cwd /repo`                    |
| `/acp cancel`        | 取消目标会话中正在执行的轮次。                            | `/acp cancel agent:codex:acp:<uuid>`                          |
| `/acp steer`         | 向正在运行的会话发送 Steer 指令。                         | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close`         | 关闭会话并解除线程目标绑定。                              | `/acp close`                                                  |
| `/acp status`        | 显示后端、模式、状态、运行时选项和能力。                  | `/acp status`                                                 |
| `/acp set-mode`      | 设置目标会话的运行时模式。                                | `/acp set-mode plan`                                          |
| `/acp set`           | 写入通用运行时配置选项。                                  | `/acp set model openai/gpt-5.4`                               |
| `/acp cwd`           | 设置运行时工作目录覆盖值。                                | `/acp cwd /Users/user/Projects/repo`                          |
| `/acp permissions`   | 设置审批策略配置文件。                                    | `/acp permissions strict`                                     |
| `/acp timeout`       | 设置运行时超时（秒）。                                    | `/acp timeout 120`                                            |
| `/acp model`         | 设置运行时模型覆盖值。                                    | `/acp model anthropic/claude-opus-4-6`                        |
| `/acp reset-options` | 移除会话运行时选项覆盖值。                                | `/acp reset-options`                                          |
| `/acp sessions`      | 列出存储中的近期 ACP 会话。                               | `/acp sessions`                                               |
| `/acp doctor`        | 显示后端健康状态、能力和可执行的修复措施。                | `/acp doctor`                                                 |
| `/acp install`       | 输出确定性的安装和启用步骤。                              | `/acp install`                                                |

运行时控制（`spawn`、`cancel`、`steer`、`close`、`status`、`set-mode`、
`set`、`cwd`、`permissions`、`timeout`、`model` 和 `reset-options`）要求
来自外部渠道的所有者身份，以及来自内部 Gateway 网关客户端的
`operator.admin`。已获授权的非所有者发送方仍可使用 `sessions`、
`doctor`、`install` 和 `help`。对于非所有者发送方，`/acp sessions`
仅列出当前绑定的会话或请求方会话；所有者身份和
`operator.admin` 客户端可以看到所有近期会话。

`/acp status` 会显示有效的运行时选项，以及运行时级别和
后端级别的会话标识符。当后端缺少某项能力时，会明确显示不支持该控制的错误。
接受目标令牌（`session-key`、`session-id` 或 `session-label`）的命令
会通过 Gateway 网关会话发现机制解析它们，包括每个智能体的自定义
`session.store` 根目录。`/acp sessions` 不接受目标令牌。

### 运行时选项映射

`/acp` 提供便捷命令和通用设置器。等效操作：

| 命令                      | 映射到                              | 说明                                                                                                                                                                                                      |
| ---------------------------- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/acp model <id>`            | 运行时配置键 `model`           | 对于 Codex ACP，OpenClaw 会将 `openai/<model>` 规范化为适配器模型 ID，并将 `openai/gpt-5.4/high` 等斜杠推理后缀映射到 `reasoning_effort`。                                         |
| `/acp set thinking <level>`  | 规范选项 `thinking`          | 如果存在后端公布的等效项，OpenClaw 会发送该等效项，并依次优先选择 `thinking`、`effort`、`reasoning_effort` 或 `thought_level`。对于 Codex ACP，适配器会将值映射到 `reasoning_effort`。 |
| `/acp permissions <profile>` | 规范选项 `permissionProfile` | 如果存在后端公布的等效项，OpenClaw 会发送该等效项，例如 `approval_policy`、`permission_profile`、`permissions` 或 `permission_mode`。                                                       |
| `/acp timeout <seconds>`     | 规范选项 `timeoutSeconds`    | 如果存在后端公布的等效项，OpenClaw 会发送该等效项，例如 `timeout` 或 `timeout_seconds`。                                                                                                     |
| `/acp cwd <path>`            | 运行时 cwd 覆盖                 | 直接更新。                                                                                                                                                                                             |
| `/acp set <key> <value>`     | 通用                              | `key=cwd` 使用 cwd 覆盖路径。                                                                                                                                                                      |
| `/acp reset-options`         | 清除所有运行时覆盖         | -                                                                                                                                                                                                          |

## acpx harness、插件设置和权限

有关 acpx harness 配置（Claude Code / Codex / Gemini CLI 别名）、
plugin-tools 和 OpenClaw-tools MCP 桥接以及 ACP 权限模式，
请参阅 [ACP Agents 设置](/zh-CN/tools/acp-agents-setup)。

## 故障排查

| 症状                                                                                   | 可能原因                                                                                                           | 修复方法                                                                                                                                                                      |
| ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ACP runtime backend is not configured`                                                   | 后端插件缺失、已禁用或被 `plugins.allow` 阻止。                                                       | 安装并启用后端插件；如果设置了该允许列表，请将 `acpx` 加入 `plugins.allow`，然后运行 `/acp doctor`。                                                 |
| `ACP is disabled by policy (acp.enabled=false)`                                           | ACP 已全局禁用。                                                                                                 | 设置 `acp.enabled=true`。                                                                                                                                                  |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)`                         | 已禁用从普通线程消息自动分派。                                                               | 设置 `acp.dispatch.enabled=true` 以恢复自动线程路由；显式调用 `sessions_spawn({ runtime: "acp" })` 仍然有效。                                      |
| `ACP agent "<id>" is not allowed by policy`                                               | Agent 不在允许列表中。                                                                                                | 使用允许的 `agentId`，或更新 `acp.allowedAgents`。                                                                                                                     |
| `/acp doctor` 在启动后立即报告后端未就绪                               | 后端插件缺失、已禁用、被允许/拒绝策略阻止，或其配置的可执行文件不可用。        | 安装/启用后端插件，重新运行 `/acp doctor`；如果仍不健康，请检查后端安装或策略错误。                                           |
| 找不到 harness 命令                                                                 | 适配器 CLI 未安装、外部插件缺失，或非 Codex 适配器的首次运行 `npx` 获取失败。 | 运行 `/acp doctor`，在 Gateway 网关主机上安装/预热适配器，或显式配置 acpx Agent 命令。                                                      |
| harness 报告找不到模型                                                          | 模型 ID 对其他提供商/harness 有效，但对该 ACP 目标无效。                                                | 使用该 harness 列出的模型、在 harness 中配置模型，或省略覆盖。                                                                            |
| harness 报告供应商身份验证错误                                                        | OpenClaw 运行正常，但目标 CLI/提供商尚未登录。                                                     | 在 Gateway 网关主机环境中登录或提供所需的提供商密钥。                                                                                             |
| `Unable to resolve session target: ...`                                                   | 键/ID/标签令牌不正确。                                                                                                | 运行 `/acp sessions`，复制准确的键/标签，然后重试。                                                                                                                        |
| `--bind here requires running /acp spawn inside an active ... conversation`               | 在没有可绑定的活动对话时使用了 `--bind here`。                                                            | 移至目标聊天/渠道并重试，或使用未绑定的生成方式。                                                                                                         |
| `Conversation bindings are unavailable for <channel>.`                                    | 适配器缺少当前对话的 ACP 绑定能力。                                                             | 在支持的情况下使用 `/acp spawn ... --thread ...`，配置顶层 `bindings[]`，或移至支持的渠道。                                                     |
| `--thread here requires running /acp spawn inside an active ... thread`                   | 在线程上下文之外使用了 `--thread here`。                                                                         | 移至目标线程，或使用 `--thread auto`/`off`。                                                                                                                      |
| `Only <user-id> can rebind this channel/conversation/thread.`                             | 另一个用户拥有活动绑定目标。                                                                           | 以所有者身份重新绑定，或使用其他对话或线程。                                                                                                               |
| `Thread bindings are unavailable for <channel>.`                                          | 适配器缺少线程绑定能力。                                                                               | 使用 `--thread off`，或移至支持的适配器/渠道。                                                                                                                 |
| `Sandboxed sessions cannot spawn ACP sessions ...`                                        | ACP 运行时位于主机端；请求者会话处于沙箱隔离状态。                                                              | 从沙箱隔离会话使用 `runtime="subagent"`，或从非沙箱隔离会话运行 ACP 生成。                                                                         |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...`                   | 为 ACP 运行时请求了 `sandbox="require"`。                                                                         | 如果必须使用沙箱隔离，请使用 `runtime="subagent"`；或者从非沙箱隔离会话通过 `sandbox="inherit"` 使用 ACP。                                                      |
| `Cannot apply --model ... did not advertise model support`                                | 目标 harness 未公开通用 ACP 模型切换功能。                                                        | 使用公布 ACP `models`/`session/set_model` 的 harness，使用 Codex ACP 模型引用；如果 harness 有自己的启动标志，也可直接在其中配置模型。 |
| 绑定会话缺少 ACP 元数据                                                    | ACP 会话元数据已过时/删除。                                                                                    | 使用 `/acp spawn` 重新创建，然后重新绑定/聚焦线程。                                                                                                                    |
| `PermissionPromptUnavailableError: Permission prompt unavailable in non-interactive mode` | `permissionMode` 会阻止非交互式 ACP 会话中的写入/执行操作。                                                    | 将 `plugins.entries.acpx.config.permissionMode` 设置为 `approve-all` 并重启 Gateway 网关。请参阅[权限配置](/zh-CN/tools/acp-agents-setup#permission-configuration)。 |
| ACP 会话过早失败且几乎没有输出                                                | 权限提示被 `permissionMode`/`nonInteractivePermissions` 阻止。                                        | 检查 Gateway 网关日志中的 `AcpRuntimeError`。如需完整权限，请设置 `permissionMode=approve-all`；如需优雅降级，请设置 `nonInteractivePermissions=deny`。        |
| ACP 会话完成工作后无限期停滞                                     | harness 进程已结束，但 ACP 会话未报告完成。                                                    | 更新 OpenClaw；当前的 acpx 清理流程会在关闭时和 Gateway 网关启动时清除 OpenClaw 所属的陈旧包装器和适配器进程。                                             |
| harness 看到 `<<<BEGIN_OPENCLAW_INTERNAL_CONTEXT>>>`                                      | 内部事件信封泄漏到了 ACP 边界之外。                                                                | 更新 OpenClaw 并重新运行完成流程；外部 harness 应仅接收纯文本完成提示。                                                          |

<Note>
`Command blocked by PreToolUse hook: Native hook relay unavailable` 属于
原生 Codex 钩子中继，而不是 ACP/acpx。在已绑定的 Codex 聊天中，使用
`/new` 或 `/reset` 启动新会话；如果它成功一次，但在
下一次原生工具调用时再次出现，请重启 Codex app-server 或 OpenClaw Gateway 网关，
而不是重复执行 `/new`。请参阅
[Codex harness 故障排查](/zh-CN/plugins/codex-harness#troubleshooting)。
</Note>

## 相关内容

- [ACP Agents 设置](/zh-CN/tools/acp-agents-setup)
- [Agent 发送](/zh-CN/tools/agent-send)
- [CLI 后端](/zh-CN/gateway/cli-backends)
- [Codex harness](/zh-CN/plugins/codex-harness)
- [Codex harness runtime](/zh-CN/plugins/codex-harness-runtime)
- [多 Agent 沙盒工具](/zh-CN/tools/multi-agent-sandbox-tools)
- [`openclaw acp`（桥接模式）](/zh-CN/cli/acp)
- [子智能体](/zh-CN/tools/subagents)
