---
read_when:
    - 配置 `tools.*` 策略、允许列表或实验性功能
    - 注册自定义提供商或覆盖基础 URL
    - 设置 OpenAI 兼容的自托管端点
sidebarTitle: Tools and custom providers
summary: 工具配置（策略、实验性开关、由提供商支持的工具）和自定义提供商/基础 URL 设置
title: 配置 — 工具和自定义提供商
x-i18n:
    generated_at: "2026-07-26T05:47:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2010a2e48e8f4c8d0049e5c707bb8286e291a92312baac94301a7b5a674583c1
    source_path: gateway/config-tools.md
    workflow: 16
---

`tools.*` 配置键以及自定义提供商 / 基础 URL 设置。有关智能体、渠道和其他顶层配置键，请参阅[配置参考](/zh-CN/gateway/configuration-reference)。

## 工具

### 工具配置档

`tools.profile` 在 `tools.allow`/`tools.deny` 之前设置基础允许列表：

<Note>
未设置时，本地新手引导默认将新的本地配置设为 `tools.profile: "coding"`（保留现有的显式配置档）。
</Note>

| 配置档     | 包含                                                                                                                                                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | 仅 `session_status`                                                                                                                                                                                                                                   |
| `coding`    | `group:fs`、`group:runtime`、`group:web`、`group:sessions`、`group:memory`、`cron`、`get_goal`、`create_goal`、`update_goal`、`update_plan`、`ask_user`、`skill_workshop`、`image`、`image_generate`、`music_generate`、`video_generate`                |
| `messaging` | `group:messaging`、`sessions`、`sessions_list`、`sessions_history`、`sessions_search`、`conversations_list`、`conversations_send`、`conversations_turn`、`sessions_send`、`sessions_spawn`、`sessions_yield`、`subagents`、`session_status`、`ask_user` |
| `full`      | 无限制（与未设置相同）                                                                                                                                                                                                                          |

`coding` 和 `messaging` 还会隐式允许 `bundle-mcp`（已配置的 MCP 服务器）。

### 工具组

| 组              | 工具                                                                                                                                                                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec`、`process`、`code_execution`（`bash` 可作为 `exec` 的别名）                                                                                                                                                                        |
| `group:fs`         | `read`、`write`、`edit`、`apply_patch`                                                                                                                                                                                                                 |
| `group:sessions`   | `sessions`、`sessions_list`、`sessions_history`、`sessions_search`、`conversations_list`、`conversations_send`、`conversations_turn`、`sessions_send`、`sessions_spawn`、`sessions_yield`、`subagents`、`session_status`、`spawn_task`、`dismiss_task` |
| `group:memory`     | `memory_search`、`memory_get`                                                                                                                                                                                                                          |
| `group:web`        | `web_search`、`x_search`、`web_fetch`                                                                                                                                                                                                                  |
| `group:ui`         | `browser`、`screen`、`terminal`、`canvas`、`show_widget`                                                                                                                                                                                               |
| `group:automation` | `heartbeat_respond`、`cron`、`gateway`                                                                                                                                                                                                                 |
| `group:messaging`  | `message`                                                                                                                                                                                                                                              |
| `group:nodes`      | `nodes`、`computer`                                                                                                                                                                                                                                    |
| `group:agents`     | `agents_list`、`get_goal`、`create_goal`、`update_goal`、`update_plan`、`ask_user`、`skill_workshop`                                                                                                                                                   |
| `group:media`      | `image`、`image_generate`、`music_generate`、`video_generate`、`tts`                                                                                                                                                                                   |
| `group:openclaw`   | 除 `read`/`write`/`edit`/`apply_patch`/`exec`/`process`/`canvas` 之外的上述所有内置工具（不包括插件工具）                                                                                                                                  |
| `group:plugins`    | 已加载插件所拥有的工具，包括通过 `bundle-mcp` 公开的已配置 MCP 服务器                                                                                                                                                           |

`spawn_task` 允许编码智能体提出需确认的后续工作，而不立即启动它。Control UI 将标题和摘要显示为可操作的标签；由 Gateway 网关支持的 TUI 会显示等效的交互式提示。接受任一提示都会创建新的托管工作树会话，并将完整提示发送到该会话，同时当前轮次继续进行。`dismiss_task` 使用 `spawn_task` 返回的临时 `task_id` 撤回仍处于待处理状态的建议。

仅当发起操作的界面能够接收并处理 Gateway 网关任务建议事件时，才会提供这些工具。渠道会话和本地/嵌入式 TUI 会话不会接收这些事件；渠道传输需要可移植的类型化任务操作，之后才能安全地公开此流程。建议仅存在于当前进程中，并会在 Gateway 网关重启时消失。这两个工具仍包含在 `coding` 配置档和 `group:sessions` 中，因此当界面支持它们时，常规的 `tools.allow` 和 `tools.deny` 策略会自动配置它们。

### 沙箱工具策略中的 MCP 和插件工具

已配置的 MCP 服务器以 `bundle-mcp` 插件 ID 下由插件拥有的工具形式公开。常规工具配置档可以允许这些工具，但对于沙箱隔离的会话，`tools.sandbox.tools` 是一个额外关卡。如果沙箱模式为 `"all"` 或 `"non-main"`，并且需要显示 MCP/插件工具，请在沙箱工具允许列表中包含以下条目之一：

- `bundle-mcp`，用于来自 `mcp.servers` 的 OpenClaw 托管 MCP 服务器
- 特定原生插件的插件 ID
- `group:plugins`，用于所有已加载的插件自有工具
- 确切的 MCP 服务器工具名称或服务器 glob，例如 `outlook__send_mail` 或 `outlook__*`，适用于只需要一个服务器的情况

服务器 glob 使用提供商安全的 MCP 服务器前缀，不一定是原始 `mcp.servers` 键。非 `[A-Za-z0-9_-]` 字符会变为 `-`，不以字母开头的名称会添加 `mcp-` 前缀，过长或重复的前缀可能会被截断或添加后缀；例如，`mcp.servers["Outlook Graph"]` 使用类似 `outlook-graph__*` 的 glob。

```json5
{
  agents: { defaults: { sandbox: { mode: "all" } } },
  mcp: {
    servers: {
      outlook: { command: "node", args: ["./outlook-mcp.js"] },
    },
  },
  tools: {
    sandbox: {
      tools: {
        alsoAllow: ["web_search", "web_fetch", "memory_search", "memory_get", "bundle-mcp"],
      },
    },
  },
}
```

如果缺少该沙箱层条目，MCP 服务器仍可成功加载，但其工具会在提供商请求之前被过滤掉。使用 `openclaw doctor` 可捕获 `mcp.servers` 中 OpenClaw 托管服务器的这种配置情况。从内置插件清单或 Claude `.mcp.json` 加载的 MCP 服务器使用相同的沙箱关卡，但此诊断目前尚不会枚举这些来源；如果它们的工具在沙箱隔离的轮次中消失，请使用相同的允许列表条目。

### `tools.codeMode`

`tools.codeMode` 启用通用 OpenClaw 代码模式界面。为带工具的运行启用后，常规 OpenClaw 工具会移至沙箱内的 `tools.*` 目录桥接器之后，而 MCP 工具可通过生成的 `MCP` 命名空间使用。模型通常会看到 `exec` 和 `wait`；像 `computer` 这样结构化结果无法通过仅支持 JSON 的桥接器传递的工具仍保持直接提供。

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

也接受简写形式：

```json5
{
  tools: { codeMode: true },
}
```

在代码模式下，MCP 声明通过只读虚拟 API 文件界面公开。来宾代码可以调用 `API.list("mcp")` 和 `API.read("mcp/<server>.d.ts")`，在调用 `MCP.<server>.<tool>()` 之前检查 TypeScript 风格的签名。有关运行时契约、限制和调试步骤，请参阅[代码模式](/tools/code-mode)。

### `tools.allow` / `tools.deny`

全局工具允许/拒绝策略（拒绝优先）。不区分大小写，支持 `*` 通配符。即使 Docker 沙箱关闭也会应用。

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

`write` 和 `apply_patch` 是不同的工具 ID。`allow: ["write"]` 还会为兼容模型启用 `apply_patch`，但 `deny: ["write"]` 不会拒绝 `apply_patch`。要阻止所有文件修改，请拒绝 `group:fs`，或明确列出每个修改工具：

```json5
{
  tools: { deny: ["write", "edit", "apply_patch"] },
}
```

<Note>
不能在同一作用域（`tools`、`tools.byProvider.<id>`、`agents.entries.*.tools`）中同时设置 `allow` 和 `alsoAllow`——配置验证会拒绝这种情况。请将 `alsoAllow` 条目合并到 `allow`，或移除 `allow`，改用 `profile` + `alsoAllow`。
</Note>

### `tools.byProvider`

进一步限制特定提供商或模型可用的工具。顺序：基础配置档 → 提供商配置档 → 允许/拒绝。

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.toolsBySender`

限制当前轮次原始请求者可使用的工具。这是在渠道访问控制之上的纵深防御；发送者值必须来自渠道适配器，而非消息文本。它不会对模型提示中的其他内容进行身份验证；请参阅[请求者范围控制和提示上下文](/zh-CN/gateway/security#requester-scoped-controls-and-prompt-context)。

```json5
{
  tools: {
    toolsBySender: {
      "channel:discord:1234567890123": { alsoAllow: ["group:fs"] },
      "id:guest-user-id": { deny: ["group:runtime", "group:fs"] },
      "*": { deny: ["exec", "process", "write", "edit", "apply_patch"] },
    },
  },
}
```

键使用显式前缀：`channel:<channelId>:<senderId>`、`id:<senderId>`、`e164:<phone>`、`username:<handle>`、`name:<displayName>` 或 `"*"`。渠道 ID 是规范的 OpenClaw ID；`teams` 等别名会规范化为 `msteams`。旧版无前缀键仅作为 `id:` 接受。匹配顺序依次为渠道 + ID、ID、e164、用户名、名称，最后是通配符。

当每个智能体的 `agents.entries.*.tools.toolsBySender` 匹配时，它会覆盖全局发送者匹配，即使 `{}` 策略为空也是如此。

### `tools.elevated`

控制沙箱外提升权限的 Exec 访问：

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123", "987654321098765432"],
      },
    },
  },
}
```

- 每个智能体的覆盖配置（`agents.entries.*.tools.elevated`）只能进一步收紧限制。
- `/elevated on|off|ask|full` 按会话存储状态；内联指令仅应用于单条消息。
- 提升权限的 `exec` 会绕过沙箱隔离，并使用配置的逃逸路径（默认为 `gateway`；当 Exec 目标为 `node` 时则为 `node`）。

### `tools.exec`

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      approvalRunningNoticeMs: 10000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      commandHighlighting: false,
      applyPatch: {
        enabled: true,
        allowModels: ["gpt-5.6-sol"],
      },
    },
  },
}
```

除 `applyPatch.allowModels` 外，所示值均为默认值（默认值为空/未设置，表示任何兼容模型均可使用 `apply_patch`）。当由审批支持的 Exec 长时间运行时，`approvalRunningNoticeMs` 会发出运行中通知；`0` 会将其禁用。

### `tools.loopDetection`

工具循环安全检查**默认禁用**。将 `enabled: true` 设置为启用检测。设置可在 `tools.loopDetection` 中进行全局定义，并在每个智能体的 `agents.entries.*.tools.loopDetection` 中覆盖。

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
    },
  },
}
```

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // 或 BRAVE_API_KEY 环境变量（Brave 提供商）
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // 可选；省略则自动检测
        maxChars: 20000,
        maxCharsCap: 20000,
        maxResponseBytes: 750000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true,
        userAgent: "custom-ua",
      },
    },
  },
}
```

除 `provider` 和 `userAgent` 外，所示值均为默认值。`maxResponseBytes` 会限制在 32000–10000000 范围内；`maxChars` 会限制为 `maxCharsCap`（提高 `maxCharsCap` 可允许更大的响应）。

### `tools.media`

配置入站媒体理解（图像/音频/视频）：

```json5
{
  tools: {
    media: {
      concurrency: 2,
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          capabilities: ["audio"],
        },
        { provider: "ollama", model: "gemma4:26b", capabilities: ["image"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["video"] },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-mini-transcribe" },
      image: { enabled: true, preferredModel: "ollama/gemma4:26b" },
      video: { enabled: true },
    },
  },
}
```

`tools.media.models` 是唯一配置的模型列表。每个条目都声明其处理的能力。可选的 `preferredModel` 选择器接受 `provider/model`、模型 ID、用于提供商默认条目的 `provider:<id>` 或 `cli:command`；匹配的条目会移至相应能力回退顺序的最前面。对于已配置和自动检测的模型，每种能力的提示、限制、请求设置、范围、附件策略和音频转录回显均保持默认值；模型条目可以覆盖模型特定字段。

<AccordionGroup>
  <Accordion title="媒体模型条目字段">
    **提供商条目**（`type: "provider"` 或省略）：

    - `provider`：API 提供商 ID（`openai`、`anthropic`、`google`/`gemini`、`groq` 等）
    - `model`：模型 ID 覆盖
    - `profile` / `preferredProfile`：`auth-profiles.json` 配置文件选择

    **CLI 条目**（`type: "cli"`）：

    - `command`：要运行的可执行文件
    - `args`：模板化参数（支持 `{{AttachmentPath}}`、`{{AttachmentUrl}}`、`{{AttachmentContentType}}`、`{{AttachmentDir}}`、`{{AttachmentIndex}}`、`{{Prompt}}`、`{{MaxChars}}` 等；`openclaw doctor --fix` 会将已弃用的 `{input}` 占位符迁移到 `{{AttachmentPath}}`）。较旧的 `{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}` 和 `{{MediaDir}}` 别名在其兼容期内仍可使用，但已弃用。

    **通用字段：**

    - `capabilities`：包含 `image`、`audio` 和 `video` 中一项或多项的列表。
    - `prompt`、`maxChars`、`maxBytes`、`timeoutSeconds`、`language`：每个条目的覆盖配置。
    - 当智能体调用显式 `image` 工具时，匹配的图像模型 `timeoutSeconds` 条目也会应用。对于图像理解，此超时应用于请求本身，不会因之前的准备工作而缩短。
    - 失败时回退到下一个条目。

    提供商身份验证遵循标准顺序：`auth-profiles.json` → 环境变量 → `models.providers.*.apiKey`。

  </Accordion>
</AccordionGroup>

### `tools.agentToAgent`

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `tools.sessions`

控制会话工具（`sessions_list`、`sessions_history`、`sessions_send`）可以将哪些会话作为目标。

默认值：`tree`（当前会话 + 由其生成的会话，例如子智能体，以及同一智能体在环境中监视的群组会话）。

```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",
    },
  },
}
```

<AccordionGroup>
  <Accordion title="可见性范围">
    - `self`：仅当前会话键。
    - `tree`：当前会话 + 当前会话生成的会话（子智能体）。对于读取操作，它还包括当前会话通过环境群组感知所监视的同一智能体群组会话。
    - `agent`：属于当前智能体 ID 的任何会话（如果在同一智能体 ID 下运行按发送者划分的会话，可能包含其他用户）。
    - `all`：任何会话。跨智能体定向仍需要 `tools.agentToAgent`。
    - 沙箱限制：当前会话处于沙箱隔离状态且 `agents.defaults.sandbox.sessionToolsVisibility="spawned"`（默认值）时，即使 `tools.sessions.visibility="all"`，可见性也会被强制设为 `tree`。
    - 当不是 `all` 时，`sessions_list` 会包含一个精简的 `visibility` 字段，用于描述生效模式，并警告当前范围之外的某些会话可能会被省略。

  </Accordion>
</AccordionGroup>

使用默认的 `session.dmScope: "main"` 时，群组中的人类活动会使该同一智能体群组会话在环境中对智能体的主会话可见。在多用户设置中，`"main"` 还会在用户之间共享一个私信会话，因此路由到该会话的每个用户都可以读取在环境中监视的群组，包括通过会话记忆 `memory_search` 读取。请使用按对等方划分的 `dmScope` 来隔离私信，或设置 `tools.sessions.visibility: "self"` 以停用环境监视会话读取。

### `tools.sessions_spawn`

控制 `sessions_spawn` 的内联附件支持。

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // 选择启用：设为 true 以允许内联文件附件
        maxTotalBytes: 5242880, // 所有文件合计 5 MB
        maxFiles: 50,
        maxFileBytes: 1048576, // 每个文件 1 MB
        retainOnSessionKeep: false, // cleanup="keep" 时保留附件
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="附件说明">
    - 附件需要 `enabled: true`。
    - 子智能体附件会通过 `.manifest.json` 实体化到子工作区的 `.openclaw/attachments/<uuid>/` 中。
    - ACP 附件仅支持图像，在通过相同的文件数量、单文件字节数和总字节数限制后，会以内联方式转发到 ACP 运行时。
    - 附件内容会自动从持久化的转录记录中隐去。
    - Base64 输入会接受严格的字母表/填充检查以及解码前大小防护。
    - 子智能体附件的文件权限：目录为 `0700`，文件为 `0600`。
    - 子智能体清理遵循 `cleanup` 策略：`delete` 始终删除附件；仅当 `retainOnSessionKeep: true` 时，`keep` 才保留附件。

  </Accordion>
</AccordionGroup>

<a id="toolsexperimental"></a>

### `tools.experimental`

实验性内置工具标志。除非适用严格智能体式 GPT-5 自动启用规则，否则默认关闭。

```json5
{
  tools: {
    experimental: {
      planTool: true, // 启用实验性 update_plan
    },
  },
}
```

- `planTool`：启用结构化 `update_plan` 工具，用于跟踪非简单的多步骤工作。
- 默认值：`false`；但如果在针对 GPT-5 系列模型 ID 的 `openai` 提供商运行中，将 `agents.defaults.embeddedAgent.executionContract`（或每个智能体的覆盖配置）设为 `"strict-agentic"`，则例外（这也涵盖 OpenAI Codex CLI 运行，因为 Codex 身份验证/模型路由位于 `openai` 提供商下）。设置 `true` 可在该范围之外强制启用该工具；设置 `false` 可使其即使在严格智能体式 GPT-5 运行中也保持关闭。
- 启用后，系统提示还会添加使用指导，使模型仅将其用于实质性工作，并最多保持一个步骤为 `in_progress`。

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        allowAgents: ["research"],
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 8,
        runTimeoutSeconds: 900,
        announceTimeoutMs: 120000,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model`：生成的子智能体的默认模型。如果省略，子智能体将继承调用方的模型。
- `allowAgents`：当请求方智能体未设置自己的 `subagents.allowAgents` 时，为 `sessions_spawn` 配置的目标智能体 ID 的默认允许列表（`["*"]` = 任意已配置目标；默认：仅同一智能体）。智能体配置已删除的过期条目会被 `sessions_spawn` 拒绝，并从 `agents_list` 中省略；运行 `openclaw doctor --fix` 可将其清理。
- `maxConcurrent`：子智能体并发运行数上限。默认值：`8`。
- `runTimeoutSeconds`：调用方未传递自己的覆盖值时，`sessions_spawn` 的超时时间（秒）。默认值：`0`（无超时）；上面显示的 `900` 是常用的选择启用值，并非内置默认值。
- `announceTimeoutMs`：Gateway 网关 `agent` 公告投递尝试的单次调用超时时间（毫秒）。默认值：`120000`。临时重试可能使公告的总等待时间超过一次配置的超时时间。
- `archiveAfterMinutes`：子智能体会话完成后到自动归档前的分钟数。默认值：`60`；`0` 会禁用自动归档。
- 每个子智能体的工具策略：`tools.subagents.tools.allow` / `tools.subagents.tools.deny`。

---

## 自定义提供商和基础 URL

提供商插件会发布自己的模型目录行。通过配置中的 `models.providers` 或 `~/.openclaw/agents/<agentId>/agent/models.json` 添加自定义提供商。

配置自定义/本地提供商 `baseUrl`，也是针对模型 HTTP 请求的一项窄范围网络信任决策：OpenClaw 允许该 `scheme://host:port` 的确切源通过受保护的获取路径，而无需添加单独的配置选项或信任其他私有源。

```json5
{
  models: {
    mode: "merge", // merge (default) | replace
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions", // openai-completions | openai-responses | anthropic-messages | google-generative-ai | etc.
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            contextTokens: 96000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="身份验证和合并优先级">
    - 自定义身份验证需求请使用 `authHeader: true` + `headers`。
    - 使用 `OPENCLAW_AGENT_DIR` 覆盖智能体配置根目录。
    - 匹配提供商 ID 时的合并优先级：
      - 智能体中非空的 `models.json` `baseUrl` 值优先。
      - 仅当该提供商在当前配置/身份验证配置文件上下文中不由 SecretRef 管理时，智能体中非空的 `apiKey` 值才优先。
      - 由 SecretRef 管理的提供商 `apiKey` 值会从来源标记（环境引用使用 `ENV_VAR_NAME`，文件/exec 引用使用 `secretref-managed`）刷新，而不会持久化已解析的密钥。
      - 由 SecretRef 管理的提供商标头值会从来源标记（环境引用使用 `secretref-env:ENV_VAR_NAME`，文件/exec 引用使用 `secretref-managed`）刷新。
      - 智能体中为空或缺失的 `apiKey`/`baseUrl` 会回退到配置中的 `models.providers`。
      - 匹配模型的 `contextWindow`/`maxTokens`：如果显式配置值存在且有效（有限正数），则该值优先；否则使用隐式/生成的目录值。
      - 匹配模型的 `contextTokens` 遵循相同的“显式值优先，否则使用隐式值”规则；使用它可限制有效上下文，而无需更改原生模型元数据。
      - 提供商插件目录以生成的、归插件所有的目录分片形式存储在智能体的插件状态中。
      - 如果希望配置完全重写 `models.json`，并跳过合并归插件所有的目录分片，请使用 `models.mode: "replace"`。
      - 标记持久化以来源为准：标记从活动的来源配置快照（解析前）写入，而不是从解析后的运行时密钥值写入。

  </Accordion>
</AccordionGroup>

### 提供商字段详情

<AccordionGroup>
  <Accordion title="顶层目录">
    - `models.mode`：提供商目录行为（`merge` 或 `replace`）。
    - `models.providers`：以提供商 ID 为键的自定义提供商映射。
      - 安全编辑：使用 `openclaw config set models.providers.<id> '<json>' --strict-json --merge` 或 `openclaw config set models.providers.<id>.models '<json-array>' --strict-json --merge` 进行增量更新。除非传递 `--replace`，否则 `config set` 会拒绝破坏性替换。

  </Accordion>
  <Accordion title="提供商连接和身份验证">
    - `models.providers.*.api`：请求适配器（`openai-completions`、`openai-responses`、`openai-chatgpt-responses`、`anthropic-messages`、`google-generative-ai`、`google-vertex`、`github-copilot`、`bedrock-converse-stream`、`ollama`、`azure-openai-responses`）。对于 MLX、vLLM、SGLang 等自托管 `/v1/chat/completions` 后端以及大多数兼容 OpenAI 的本地服务器，请使用 `openai-completions`。具有 `baseUrl` 但没有 `api` 的自定义提供商默认为 `openai-completions`；仅当后端支持 `/v1/responses` 时才设置 `openai-responses`。
    - `models.providers.*.apiKey`：提供商凭据（优先使用 SecretRef/环境变量替换）。
    - `models.providers.*.auth`：身份验证策略（`api-key`、`token`、`oauth`、`aws-sdk`）。
    - `models.providers.*.contextWindow`：当模型条目未设置 `contextWindow` 时，此提供商下模型的默认原生上下文窗口。
    - `models.providers.*.contextTokens`：当模型条目未设置 `contextTokens` 时，此提供商下模型的默认有效运行时上下文上限。
    - `models.providers.*.maxTokens`：当模型条目未设置 `maxTokens` 时，此提供商下模型的默认输出 token 上限。
    - `models.providers.*.timeoutSeconds`：可选的每提供商模型 HTTP 请求超时时间（秒），涵盖连接、标头、正文和整个请求的中止处理。
    - `models.providers.*.injectNumCtxForOpenAICompat`：对于 Ollama + `openai-completions`，在请求中注入 `options.num_ctx`（默认值：`true`）。
    - `models.providers.*.authHeader`：需要时，强制通过 `Authorization` 标头传输凭据。
    - `models.providers.*.baseUrl`：上游 API 基础 URL。
    - `models.providers.*.headers`：用于代理/租户路由的额外静态标头。

  </Accordion>
  <Accordion title="请求传输覆盖">
    `models.providers.*.request`：模型提供商 HTTP 请求的传输覆盖。

    - `request.headers`：额外标头（与提供商默认值合并）。值接受 SecretRef。
    - `request.auth`：身份验证策略覆盖。模式：`"provider-default"`（使用提供商的内置身份验证）、`"authorization-bearer"`（配合 `token`）、`"header"`（配合 `headerName`、`value`，以及可选的 `prefix`）。
    - `request.proxy`：HTTP 代理覆盖。模式：`"env-proxy"`（使用 `HTTP_PROXY`/`HTTPS_PROXY` 环境变量）、`"explicit-proxy"`（配合 `url`）。两种模式都接受可选的 `tls` 子对象。
    - `request.tls`：直连的 TLS 覆盖。字段：`ca`、`cert`、`key`、`passphrase`（均接受 SecretRef）、`serverName`、`insecureSkipVerify`。
    - `request.allowPrivateNetwork`：当值为 `true` 时，允许模型提供商 HTTP 请求通过提供商 HTTP 获取保护机制访问私有、CGNAT 或类似范围。自定义/本地提供商基础 URL 已信任已配置的确切源，但元数据/链路本地源除外；若未显式选择启用，这些源仍会被阻止。将其设置为 `false` 可选择退出确切源信任。WebSocket 对标头/TLS 使用相同的 `request`，但不使用该获取 SSRF 防护机制。默认值为 `false`。

  </Accordion>
  <Accordion title="模型目录条目">
    - `models.providers.*.models`：显式的提供商模型目录条目。
    - `models.providers.*.models.*.input`：模型输入模态。纯文本模型使用 `["text"]`，原生图像/视觉模型使用 `["text", "image"]`。仅当所选模型标记为支持图像时，图像附件才会注入智能体轮次。
    - `models.providers.*.models.*.contextWindow`：原生模型上下文窗口元数据。这会覆盖该模型的提供商级 `contextWindow`。
    - `models.providers.*.models.*.contextTokens`：可选的运行时上下文上限。这会覆盖提供商级 `contextTokens`；如果希望有效上下文预算小于模型的原生 `contextWindow`，请使用此项；当两个值不同时，`openclaw models list` 会同时显示它们。

    #### 自定义提供商能力声明

    对于内置和目录已知的模型路由，提供商目录拥有 `compat`。请勿将这些标志复制到配置中：只要已配置的 `api` 和 `baseUrl` 仍标识该路由，OpenClaw 就会使用目录行。`openclaw doctor --fix` 会移除匹配的旧版覆盖值，并报告存在差异的值以供审查。

    对于真正的自定义提供商、自定义模型或路由到不同端点的目录模型，仍支持 `compat` 块。仅设置已经针对该端点验证的能力：

    | 自定义路由键 | 运行时契约 |
    | --- | --- |
    | `supportsStore` | 接受 OpenAI `store` 请求字段。 |
    | `supportsPromptCacheKey` | 接受 OpenAI 提示缓存/会话亲和性键。 |
    | `supportsDeveloperRole` | 接受 `developer` 消息，而不要求 `system`。 |
    | `supportsReasoningEffort` | 接受推理强度控制。 |
    | `supportsTemperature` | 接受此模型和适配器的 `temperature`。 |
    | `supportsUsageInStreaming` | 在流式响应中发出用量元数据。 |
    | `supportsTools` | 支持结构化工具/函数调用。设置 `false` 可禁用工具。 |
    | `supportsStrictMode` | 接受严格工具模式。 |
    | `requiresStringContent` | 要求 Chat Completions 消息内容为纯字符串。 |
    | `strictMessageKeys` | 要求出站消息仅包含可接受的键。 |
    | `visibleReasoningDetailTypes` | 指定可安全显示在转录记录中的推理详情块类型。 |
    | `supportedReasoningEfforts` | 列出端点接受的推理标签。 |
    | `reasoningEffortMap` | 将 OpenClaw 思考标签映射到端点专用标签。 |
    | `maxTokensField` | 选择 `max_tokens` 或 `max_completion_tokens`。 |
    | `thinkingFormat` | 选择端点的推理负载方言。 |
    | `requiresToolResultName` | 要求工具结果消息包含工具名称。 |
    | `requiresAssistantAfterToolResult` | 要求工具结果之后有一条助手消息。 |
    | `requiresThinkingAsText` | 将推理以文本而非结构化内容的形式重放。 |
    | `requiresReasoningContentOnAssistantMessages` | 重放期间保留 DeepSeek 风格的 `reasoning_content`。 |
    | `toolSchemaProfile` | 选择由提供商定义的工具模式规范化配置文件。 |
    | `unsupportedToolSchemaKeywords` | 移除端点拒绝的具名 JSON Schema 关键字。 |
    | `toolCallArgumentsEncoding` | 选择端点的工具调用参数编码。 |
    | `requiresOpenAiAnthropicToolPayload` | 将 OpenAI 形式的工具调用转换为 Anthropic 系列负载。 |

  </Accordion>
  <Accordion title="Amazon Bedrock 设备发现">
    - `plugins.entries.amazon-bedrock.config.discovery`：Bedrock 自动发现设置的根配置。
    - `plugins.entries.amazon-bedrock.config.discovery.enabled`：开启/关闭隐式发现。
    - `plugins.entries.amazon-bedrock.config.discovery.region`：用于发现的 AWS 区域。
    - `plugins.entries.amazon-bedrock.config.discovery.providerFilter`：用于定向发现的可选提供商 ID 筛选器。
    - `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`：发现刷新的轮询间隔。
    - `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`：已发现模型的备用上下文窗口。
    - `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`：已发现模型的备用最大输出 token 数。

  </Accordion>
</AccordionGroup>

交互式自定义提供商新手引导会根据已知视觉模型 ID 模式推断图像输入支持，包括 GPT-4o/GPT-4.1/GPT-5+、`o1`/`o3`/`o4` 推理系列、Claude、Gemini、任何以 `-vl` 结尾的 ID（Qwen-VL 及类似模型），以及 LLaVA、Pixtral、InternVL、Mllama、MiniCPM-V 和 GLM-4V 等具名系列；对于已知的纯文本系列（Llama、DeepSeek、Mistral/Mixtral、Kimi/Moonshot、Codestral、Devstral、Phi、QwQ、CodeLlama，以及没有 vl/vision 后缀的纯 Qwen ID），则会跳过额外问题。对于未知模型 ID，仍会询问是否支持图像。非交互式新手引导使用相同的推断；传入 `--custom-image-input` 可强制使用支持图像的元数据，传入 `--custom-text-input` 可强制使用纯文本元数据。

### 提供商示例

<AccordionGroup>
  <Accordion title="Cerebras（GLM 4.7 / GPT OSS）">
    官方外部 `cerebras` 提供商插件可通过 `openclaw onboard --auth-choice cerebras-api-key` 进行配置。仅在需要覆盖默认值时使用显式提供商配置。

    ```json5
    {
      env: { CEREBRAS_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: {
            primary: "cerebras/zai-glm-4.7",
            fallbacks: ["cerebras/gpt-oss-120b"],
          },
          models: {
            "cerebras/zai-glm-4.7": { alias: "GLM 4.7 (Cerebras)" },
            "cerebras/gpt-oss-120b": { alias: "GPT OSS 120B (Cerebras)" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          cerebras: {
            baseUrl: "https://api.cerebras.ai/v1",
            apiKey: "${CEREBRAS_API_KEY}",
            api: "openai-completions",
            models: [
              { id: "zai-glm-4.7", name: "GLM 4.7 (Cerebras)" },
              { id: "gpt-oss-120b", name: "GPT OSS 120B (Cerebras)" },
            ],
          },
        },
      },
    }
    ```

    Cerebras 使用 `cerebras/zai-glm-4.7`；直连 Z.AI 使用 `zai/glm-4.7`。

  </Accordion>
  <Accordion title="Kimi Coding">
    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: { "kimi/kimi-for-coding": { alias: "Kimi Code" } },
        },
      },
    }
    ```

    内置的 Anthropic 兼容提供商。快捷方式：`openclaw onboard --auth-choice kimi-code-api-key`。

  </Accordion>
  <Accordion title="本地模型（LM Studio）">
    请参阅[本地模型](/zh-CN/gateway/local-models)。简而言之：在性能强劲的硬件上，通过 LM Studio Responses API 运行大型本地模型；保留合并的托管模型作为备用。
  </Accordion>
  <Accordion title="MiniMax M3（直连）">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "Minimax" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          minimax: {
            baseUrl: "https://api.minimax.io/anthropic",
            apiKey: "${MINIMAX_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.12, cacheWrite: 0 },
                contextWindow: 1000000,
                maxTokens: 131072,
              },
            ],
          },
        },
      },
    }
    ```

    设置 `MINIMAX_API_KEY`。快捷方式：`openclaw onboard --auth-choice minimax-global-api` 或 `openclaw onboard --auth-choice minimax-cn-api`。模型目录默认使用 M3，并且还包括 M2.7 变体。在 Anthropic 兼容的流式传输路径上，除非你自行显式设置 `thinking`，否则 OpenClaw 默认禁用 MiniMax M2.x 的思考；MiniMax-M3（以及 M3.x）默认继续使用提供商的省略式/自适应思考路径。`/fast on` 或 `params.fastMode: true` 会将 `MiniMax-M2.7` 重写为 `MiniMax-M2.7-highspeed`。

  </Accordion>
  <Accordion title="Moonshot AI（Kimi）">
    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: { "moonshot/kimi-k2.6": { alias: "Kimi K2.6" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
            ],
          },
        },
      },
    }
    ```

    中国区端点使用：`baseUrl: "https://api.moonshot.cn/v1"` 或 `openclaw onboard --auth-choice moonshot-api-key-cn`。

    Moonshot 原生端点会在共享的 `openai-completions` 传输上声明流式用量兼容性，OpenClaw 根据端点能力而非仅根据内置提供商 ID 启用此功能。

  </Accordion>
  <Accordion title="OpenCode">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "opencode/claude-opus-4-6" },
          models: { "opencode/claude-opus-4-6": { alias: "Opus" } },
        },
      },
    }
    ```

    设置 `OPENCODE_API_KEY`（或 `OPENCODE_ZEN_API_KEY`）。Zen 目录使用 `opencode/...` 引用，Go 目录使用 `opencode-go/...` 引用。快捷方式：`openclaw onboard --auth-choice opencode-zen` 或 `openclaw onboard --auth-choice opencode-go`。

  </Accordion>
  <Accordion title="Synthetic（Anthropic 兼容）">
    ```json5
    {
      env: { SYNTHETIC_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
          models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          synthetic: {
            baseUrl: "https://api.synthetic.new/anthropic",
            apiKey: "${SYNTHETIC_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "hf:MiniMaxAI/MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```

    基础 URL 应省略 `/v1`（Anthropic 客户端会追加它）。快捷方式：`openclaw onboard --auth-choice synthetic-api-key`。

  </Accordion>
  <Accordion title="Z.AI（GLM-4.7）">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-4.7" },
          models: { "zai/glm-4.7": {} },
        },
      },
    }
    ```

    设置 `ZAI_API_KEY`。模型引用使用规范的 `zai/*` 提供商 ID。快捷方式：`openclaw onboard --auth-choice zai-api-key`。

    - 通用端点：`https://api.z.ai/api/paas/v4`
    - 编码端点：`https://api.z.ai/api/coding/paas/v4`
    - 默认的 `zai-api-key` 身份验证选项会探测你的密钥，并自动检测它所属的端点（如果检测结果不明确，则回退为提示，默认选择 Global）。此外还提供专用的 CN 和 Coding-Plan 身份验证选项，可供显式选择。
    - 对于通用端点，请定义自定义提供商并覆盖基础 URL。

  </Accordion>
</AccordionGroup>

---

## 相关内容

- [配置 — 智能体](/zh-CN/gateway/config-agents)
- [配置 — 渠道](/zh-CN/gateway/config-channels)
- [配置参考](/zh-CN/gateway/configuration-reference) — 其他顶层键
- [工具和插件](/zh-CN/tools)
