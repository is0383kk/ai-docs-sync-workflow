---
read_when:
    - 你正在构建一个需要 before_tool_call、before_agent_reply、消息钩子或生命周期钩子的插件
    - 你需要阻止或重写插件发出的工具调用，或者要求对其进行审批
    - 你正在内部钩子和插件钩子之间进行选择
    - 你正在将 OpenClaw 的定时唤醒任务映射到外部主机调度器中
summary: 插件钩子：拦截智能体、工具、消息、会话和 Gateway 网关生命周期事件
title: 插件钩子
x-i18n:
    generated_at: "2026-07-26T06:23:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 95d7ea2f7bfe26b5904ea3cd8f8db85ffd8163af58e03ec56d11eee992bc13d2
    source_path: plugins/hooks.md
    workflow: 16
---

插件钩子是 OpenClaw 插件的进程内扩展点：可检查或更改智能体运行、工具调用、消息流、会话生命周期、子智能体路由、安装或 Gateway 网关启动。

对于操作员安装的、响应命令和 Gateway 网关事件（例如 `/new`、`/reset`、`/stop`、`agent:bootstrap` 或 `gateway:startup`）的小型 `HOOK.md` 脚本，请改用[内部钩子](/zh-CN/automation/hooks)。

## 快速开始

在插件入口中使用 `api.on(...)` 注册类型化钩子：

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "tool-preflight",
  name: "Tool Preflight",
  register(api) {
    api.on(
      "before_tool_call",
      async (event) => {
        if (event.toolName !== "web_search") {
          return;
        }

        return {
          requireApproval: {
            title: "Run web search",
            description: `Allow search query: ${String(event.params.query ?? "")}`,
            severity: "info",
            timeoutMs: 60_000,
          },
        };
      },
      { priority: 50 },
    );
  },
});
```

可返回决策或修改的处理程序按 `priority` 降序依次运行；优先级相同的处理程序保持注册顺序。仅观察型处理程序并行运行，而即发即弃的观察调度可能与后续事件重叠。不要使用优先级来安排观察副作用的顺序。

`api.on(name, handler, opts?)` 接受：

| 选项      | 效果                                                                                                                                                                                            |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `priority`  | 排序；值越高越先运行。                                                                                                                                                                      |
| `timeoutMs` | 每个钩子的等待时间预算。到期后，OpenClaw 将停止等待该处理程序并继续执行。它不会取消该处理程序或其副作用。省略时使用运行器默认的每钩子超时时间。 |

操作员无需修改插件代码即可设置钩子预算：

```json
{
  "plugins": {
    "entries": {
      "my-plugin": {
        "hooks": {
          "timeoutMs": 30000,
          "timeouts": {
            "before_prompt_build": 90000,
            "agent_end": 60000
          }
        }
      }
    }
  }
}
```

`hooks.timeouts.<hookName>` 覆盖 `hooks.timeoutMs`，后者覆盖插件编写者设置的 `api.on(..., { timeoutMs })` 值。每个值必须是最大为 600000 ms 的正整数。对于已知较慢的钩子，优先使用每钩子覆盖，以免一个插件在所有位置都获得更长的预算。

由于钩子回调不会收到取消信号，已超时的处理程序 Promise 会继续运行。当该插件的工作仍在进行时，钩子调度可以释放其 Gateway 网关准入。拥有长时间运行工作的插件必须自行提供取消和关闭生命周期机制。

出站修改钩子 `message_sending` 和 `reply_payload_sending` 的每个处理程序默认超时时间为 15 秒。如果其中一个超时，OpenClaw 会记录插件错误并使用最新载荷继续执行，以便串行化投递通道能够完成收尾。对于有意在投递前执行较慢工作的插件，请设置更大的每钩子预算。

使用 `createReplyDispatcher` 的渠道插件同样可以通过 `beforeDeliverOptions: { timeoutMs }` 声明更大的正数阶段预算，或者在使用 `dispatcher.appendBeforeDeliver(handler, { timeoutMs })` 追加工作时声明。如果所有者未声明预算，这些回调将使用相同的 15 秒默认值，防止挂起的回调持续占用串行化投递通道。

每个钩子都会收到 `event.context.pluginConfig`，即为注册该处理程序的插件解析后的配置。OpenClaw 会按处理程序注入该配置，而不会改变其他插件看到的共享事件对象。

## 钩子目录

钩子按其扩展的功能界面分组。**粗体**名称接受决策结果（阻止、取消、覆盖或要求审批）；其余钩子仅用于观察。

**智能体轮次**

| 钩子                            | 用途                                                                                  |
| ------------------------------- | ---------------------------------------------------------------------------------------- |
| `before_model_resolve`          | 在加载会话消息前覆盖提供商或模型                                  |
| `agent_turn_prepare`            | 使用排队的插件轮次注入，并在提示词钩子之前添加同轮次上下文      |
| `before_prompt_build`           | 在模型调用前添加动态上下文或系统提示词文本                          |
| **`before_agent_run`**          | 在提交给模型前检查最终提示词和会话消息；可阻止运行 |
| **`before_agent_reply`**        | 使用合成回复或静默响应跳过模型轮次                           |
| **`before_agent_finalize`**     | 检查自然生成的最终答案，并请求再执行一次模型处理                         |
| `agent_end`                     | 观察最终消息、成功状态和运行时长                                  |
| `heartbeat_prompt_contribution` | 为后台监控和生命周期插件添加仅限 Heartbeat 的上下文                  |

**对话观察**

| 钩子                                      | 用途                                                                                                            |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `model_call_started` / `model_call_ended` | 经过净化的提供商/模型调用元数据：时序、结果和有界请求 ID 哈希。不包含提示词或响应内容。 |
| `llm_input`                               | 提供商输入：系统提示词、提示词、历史记录                                                                     |
| `llm_output`                              | 提供商输出、用量，以及可用时解析出的 `contextTokenBudget`                                       |

**工具**

| 钩子                       | 用途                                                   |
| -------------------------- | --------------------------------------------------------- |
| **`before_tool_call`**     | 重写工具参数、阻止执行或要求审批 |
| `after_tool_call`          | 观察工具结果、错误和持续时间                |
| `resolve_exec_env`         | 向 `exec` 提供插件拥有的环境变量   |
| **`tool_result_persist`**  | 重写根据工具结果生成的助手消息 |
| **`before_message_write`** | 检查或阻止正在进行的消息写入（少见）      |

**消息和投递**

| 钩子                            | 用途                                                           |
| ------------------------------- | ----------------------------------------------------------------- |
| **`inbound_claim`**             | 在智能体路由前接管入站消息（合成回复） |
| **`channel_pairing_requested`** | 观察新创建的私信配对请求                         |
| `message_received`              | 观察入站内容、发送者、线程和元数据             |
| **`message_sending`**           | 重写出站内容或取消投递                       |
| **`reply_payload_sending`**     | 在投递前修改或取消规范化回复载荷        |
| `message_sent`                  | 观察出站投递成功或失败                      |
| **`before_dispatch`**           | 在交接给渠道前检查或重写出站调度    |
| **`reply_dispatch`**            | 参与最终回复调度流水线                  |

**会话和压缩**

| 钩子                                     | 用途                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `session_start` / `session_end`          | 跟踪会话生命周期边界。`reason` 是 `new`、`reset`、`idle`、`daily`、`compaction`、`deleted`、`shutdown`、`restart` 或 `unknown` 之一。当进程在存在活动会话的情况下停止或重启时，`shutdown`/`restart` 会从 Gateway 网关关闭终结器触发，使插件（记忆、转录存储）能够完成幽灵行的收尾，而不是让它们在重启后仍保持打开状态。该终结器有时间限制，因此缓慢的插件无法阻塞 SIGTERM/SIGINT。 |
| `before_compaction` / `after_compaction` | 观察压缩周期或为其添加注释                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `before_reset`                           | 观察会话重置事件（`/reset`、程序化重置）                                                                                                                                                                                                                                                                                                                                                                                                     |

对于带有 `parentSessionKey` 和 `emitCommandHooks: true` 的 `sessions.create` 调用，不同的子项始终会收到 `session_start`。调用方通过 `succeedsParent` 声明父项是否也会收到终止事件 `session_end`：`true` 表示后继项，`false` 表示并行子项。省略时保留旧版父项滚转行为。无论哪种情况，`command:new` 和 `before_reset` 钩子仍会描述请求的 `/new` 操作。

**子智能体**

- `subagent_spawned` / `subagent_ended` - 观察子智能体的启动和完成。
- `subagent_delivery_target` - 当核心会话绑定无法投射路由时，用于完成事件传递的兼容性钩子。
- `subagent_spawning` - 已弃用的兼容性钩子。现在，核心会在 `subagent_spawned` 触发之前，通过渠道会话绑定适配器为 `thread: true` 子智能体准备绑定。
- `subagent_spawned` 包含 `resolvedModel` 和 `resolvedProvider`，前提是 OpenClaw 在启动前已解析子会话的原生模型。
- `subagent_ended` 携带 `targetSessionKey`（身份标识，与 `subagent_spawned.childSessionKey` 匹配）、`targetKind`（`"subagent"` 或 `"acp"`）、`reason`、可选的 `outcome`（`"ok"`、`"error"`、`"timeout"`、`"killed"`、`"reset"` 或 `"deleted"`）、可选的 `error`、`runId`、`endedAt`、`accountId` 和 `sendFarewell`。它**不**包含 `agentId` 或 `childSessionKey`；请使用 `targetSessionKey` 与对应的 `subagent_spawned` 事件建立关联。

**生命周期**

| 钩子                             | 用途                                                                                              |
| -------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `gateway_start` / `gateway_stop` | 随 Gateway 网关启动或停止插件所有的服务                                                 |
| `deactivate`                     | `gateway_stop` 的已弃用兼容性别名；新插件请使用 `gateway_stop`                 |
| `cron_reconciled`                | 启动或重新加载后，根据完整的 Gateway 网关定时任务状态进行协调                            |
| `cron_changed`                   | 观察 Gateway 网关所有的定时任务生命周期变化（已添加、已更新、已移除、已启动、已完成、已调度） |
| **`before_install`**             | 检查来自已加载插件运行时的暂存技能或插件安装材料                         |

### 渠道配对请求

当未配对的私信发送者创建待处理的配对请求后，插件需要通知操作员或
写入审计记录时，请使用 `channel_pairing_requested`。
该钩子会在请求创建时分派；钩子处理程序缓慢或失败不会延迟
配对回复的渠道传递。

```typescript
api.on("channel_pairing_requested", async (event) => {
  await notifyOperator({
    text: `来自 ${event.senderId} 的新 ${event.channel} 配对请求：${event.code}`,
  });
});
```

该钩子仅用于观察。它不会批准、拒绝、阻止或重写
配对回复。有效负载包括渠道、可选的 `accountId`、
渠道范围内的 `senderId`、配对 `code` 和渠道元数据。应将
配对码视为有效的单次使用批准凭据，并且仅传递至
受信任的操作员接收端。应将 `metadata` 视为发送者提供的不受信任身份
文本。该钩子不包含入站消息正文或媒体。

## 调试运行时钩子

使用 `before_model_resolve` 可切换智能体轮次的提供商或模型——它
在模型解析之前运行。`llm_output` 仅在模型尝试
生成智能体输出后运行。

要验证会话实际使用的模型，请检查运行时注册信息，然后
使用 `openclaw sessions` 或 Gateway 网关会话/状态界面。要调试
提供商有效负载，请使用 `--raw-stream` 和
`--raw-stream-path <path>` 启动 Gateway 网关，以将原始模型流事件写入 jsonl 文件。

## 工具调用策略

`before_tool_call` 接收：

- `event.toolName`
- `event.params`
- 可选的 `event.toolKind` 和 `event.toolInputKind`，它们是由主机权威确定的
  判别字段，用于有意共享名称的工具；例如，外层
  代码模式 `exec` 调用使用 `toolKind: "code_mode_exec"`，并在已知输入语言时
  包含 `toolInputKind: "javascript" | "typescript"`
- 可选的 `event.derivedPaths`，由主机尽力推导的目标路径提示，
  用于 `apply_patch` 等常见工具信封；这些路径可能
  不完整，或可能高估工具实际将触及的范围（例如，
  输入格式错误或不完整时）
- 可选的 `event.runId`
- 可选的 `event.toolCallId`
- 上下文字段，例如 `ctx.agentId`、`ctx.sessionKey`、`ctx.sessionId`、
  `ctx.runId`、`ctx.toolKind`、`ctx.toolInputKind` 和诊断用的 `ctx.trace`
- 可选的 `ctx.requester`，即由主机推导的当前
  消息运行发起者。它可以包括 `channel`、`accountId`、`senderId`、
  `senderIsOwner` 和提供商原生的 `roleIds`。缺失字段表示尚未证实，
  并不构成否定性保证；当策略要求这些字段时，应采用拒绝优先。

它可以返回：

```typescript
type BeforeToolCallResult = {
  params?: Record<string, unknown>;
  block?: boolean;
  blockReason?: string;
  requireApproval?: {
    title: string;
    description: string;
    severity?: "info" | "warning" | "critical";
    timeoutMs?: number;
    /** @deprecated 未解决的审批始终拒绝。 */
    timeoutBehavior?: "allow" | "deny";
    allowedDecisions?: Array<"allow-once" | "allow-always" | "deny">;
    pluginId?: string;
    onResolution?: (
      decision: "allow-once" | "allow-always" | "deny" | "timeout" | "cancelled",
    ) => Promise<void> | void;
  };
};
```

类型化生命周期钩子的防护行为：

- `block: true` 是终止性决定，会跳过优先级较低的处理程序。
- `block: false` 被视为未作决定。
- `params` 会重写用于执行的工具参数。
- `requireApproval` 会暂停智能体运行，并通过插件
  审批请求用户确认。`/approve` 可同时批准 Exec 审批和插件审批。在 Codex
  app-server 报告模式的原生 `PreToolUse` 中继中，它会转交给
  对应的 app-server 审批请求；请参阅
  [Codex harness runtime](/zh-CN/plugins/codex-harness-runtime#hook-boundaries)。
- 即使优先级较高的钩子请求了审批，优先级较低的 `block: true`
  仍可阻止调用。
- `onResolution` 会收到已解析的决定：`allow-once`、`allow-always`、
  `deny`、`timeout` 或 `cancelled`。

### 在单个文件中配置发送者感知策略

独立插件文件可以将部署特定的策略保留在代码中，
而无需添加另一套配置架构。此示例向所有者开放所有工具，
允许已配置的维护者使用一组保守的工具和消息操作，
并向已获渠道配置授权的发送者开放 `/fix`：

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const AGENT_ID = "maintenance-agent";
const MAINTAINER_SCOPES = [
  {
    channel: "discord",
    accountId: "operations",
    senderIds: new Set(["maintainer-user-id"]),
    roleIds: new Set(["maintainer-role-id"]),
  },
];
const MAINTAINER_TOOLS = new Set(["read", "web_fetch", "web_search", "session_status", "message"]);
const MAINTAINER_MESSAGE_ACTIONS = new Set(["react", "reply", "thread-create", "thread-reply"]);

export default definePluginEntry({
  id: "maintenance-access",
  name: "维护访问",
  description: "对维护智能体应用发送者感知的工具策略。",
  register(api) {
    api.on("before_tool_call", (event, ctx) => {
      if (ctx.agentId !== AGENT_ID) {
        return;
      }

      const requester = ctx.requester;
      if (requester?.senderIsOwner === true) {
        return;
      }

      const maintainerScope = requester
        ? MAINTAINER_SCOPES.find(
            (scope) =>
              scope.channel === requester.channel && scope.accountId === requester.accountId,
          )
        : undefined;
      const isMaintainer =
        maintainerScope !== undefined &&
        ((requester?.senderId !== undefined && maintainerScope.senderIds.has(requester.senderId)) ||
          requester?.roleIds?.some((roleId) => maintainerScope.roleIds.has(roleId)) === true);
      if (!isMaintainer) {
        return { block: true, blockReason: "需要维护者访问权限。" };
      }

      if (event.toolName === "message") {
        const action = typeof event.params.action === "string" ? event.params.action : "";
        if (MAINTAINER_MESSAGE_ACTIONS.has(action)) {
          return;
        }
        return { block: true, blockReason: `执行 message.${action || "unknown"} 需要所有者权限。` };
      }

      if (MAINTAINER_TOOLS.has(event.toolName)) {
        return;
      }
      return { block: true, blockReason: `执行 ${event.toolName} 需要所有者权限。` };
    });

    api.registerCommand({
      name: "fix",
      description: "请求维护智能体调查并修复问题。",
      acceptsArgs: true,
      requireAuth: true,
      handler: async (ctx) =>
        ctx.agentId === AGENT_ID
          ? { continueAgent: true }
          : { text: "此命令仅在维护对话中可用。" },
    });
  },
});
```

直接加载该文件并重启 Gateway 网关：

```json5
{
  agents: {
    list: [
      {
        id: "maintenance-agent",
        workspace: "~/.openclaw/workspace-maintenance",
      },
    ],
  },
  bindings: [
    {
      agentId: "maintenance-agent",
      match: {
        channel: "discord",
        accountId: "operations",
        peer: { kind: "channel", id: "maintenance-channel-id" },
      },
    },
  ],
  plugins: {
    load: { paths: ["~/.openclaw/policies/maintenance-access.ts"] },
  },
}
```

`AGENT_ID` 必须指定绑定到维护对话的智能体。该
绑定会为普通消息和 `/fix` 选择该智能体；独立文件
仍是所有者与维护者工具策略的唯一所有者。

`requireAuth: true` 会复用各渠道现有的发送者准入机制。对于
Discord，服务器或渠道的 `users`/`roles` 允许列表可授权
维护受众。其他渠道可以使用稳定的发送者 ID。随后，该钩子会
对运行中的每次工具调用应用更细粒度的逐工具决策，包括
Codex 原生 `PreToolUse` 调用。它可以否决模型可见的工具，但无法
添加主机省略的工具。现有沙箱、Exec 审批、仅限所有者的
核心工具和渠道策略仍然适用；该钩子无法越过这些限制授予权限。

如示例所示，将发送者和角色 ID 的范围限定到确切的渠道/账户组合；
二者均属于提供商本地命名空间。应保守设置允许列表。仅当
部署的沙箱和审批策略能够确保安全时，才添加写入或
执行工具。对于自动化或系统运行，应明确决定缺少
`ctx.requester` 时是否放行；本示例会针对该范围内的智能体拒绝放行。

有关审批路由、决定行为以及何时应使用 `requireApproval`
而非可选工具或 Exec 审批，请参阅
[插件权限请求](/zh-CN/plugins/plugin-permission-requests)。

需要主机级策略的插件可以使用
`api.registerTrustedToolPolicy(...)` 注册受信任的工具策略。这些策略先于普通的
`before_tool_call` 钩子和常规钩子决定运行。内置受信任
策略最先运行；已安装插件的受信任策略随后按插件加载
顺序运行；普通的 `before_tool_call` 钩子在它们之后运行。内置插件保留
现有的受信任策略路径。已安装的插件必须被显式启用，
并在 `contracts.trustedToolPolicies` 中声明每个策略 ID；未声明的 ID
会在注册前被拒绝。策略 ID 的作用域限定于注册它的
插件，因此不同插件可以复用相同的本地 ID。仅将此层级
用于主机信任的防护措施，例如工作区策略、预算执行或
保留工作流的安全保障。

### Exec 环境钩子

`resolve_exec_env` 允许插件在命令运行前向 `exec`
工具调用提供环境变量。它接收：

- `event.sessionKey`
- `event.toolName`，目前始终为 `"exec"`
- `event.host`，取值为 `"gateway"`、`"sandbox"` 或 `"node"` 之一
- 上下文字段，例如 `ctx.agentId`、`ctx.sessionKey`、
  `ctx.messageProvider` 和 `ctx.channelId`

返回一个 `Record<string, string>`，以合并到 Exec 环境中。处理程序
按优先级顺序运行；对于相同的键，后面的结果会覆盖前面的结果。

在合并之前，钩子输出会通过主机 Exec 环境键策略进行筛选。
`PATH` 始终会被丢弃（命令解析和安全二进制文件检查依赖于它）。
无效键和危险的主机覆盖键（例如 `LD_*`、
`DYLD_*`、`NODE_OPTIONS`）、代理变量（`HTTP_PROXY`、`HTTPS_PROXY`、
`ALL_PROXY`、`NO_PROXY`）以及 TLS 覆盖变量（`NODE_TLS_REJECT_UNAUTHORIZED`、
`SSL_CERT_FILE` 等）都会被丢弃。筛选后的插件环境变量会包含在
Gateway 网关审批/审计元数据中，并转发至节点主机执行请求。

### 工具结果持久化

工具结果可包含用于 UI 渲染、诊断、媒体路由或插件自有元数据的结构化
`details`。应将 `details` 视为运行时元数据，而不是提示词内容：

- OpenClaw 会在提供商重放和压缩输入之前移除 `toolResult.details`，
  以免元数据成为模型上下文。
- 持久化的会话条目仅保留有界的 `details`。过大的详细信息会被
  替换为紧凑摘要和 `persistedDetailsTruncated: true`。
- `tool_result_persist` 和 `before_message_write` 在最终
  持久化上限应用前运行。请保持返回的 `details` 较小，并避免仅将
  与提示词相关的文本放入 `details`；应将模型可见的工具输出放入
  `content`。

## 提示词和模型钩子

新插件应使用特定阶段的钩子：

- `before_model_resolve`：仅接收当前提示词和附件
  元数据。返回 `providerOverride` 或 `modelOverride`。
- `agent_turn_prepare`：接收当前提示词、准备好的会话
  消息，以及为此会话提取的所有仅一次排队注入内容。
  返回 `prependContext` 或 `appendContext`。
- `before_prompt_build`：接收当前提示词和会话消息。
  返回 `prependContext`、`appendContext`、`systemPrompt`、
  `prependSystemContext` 或 `appendSystemContext`。
- `heartbeat_prompt_contribution`：仅在 Heartbeat 轮次中运行，并返回
  `prependContext` 或 `appendContext`。用于需要汇总当前状态、
  但不应改变用户发起轮次的后台监控器。

`before_agent_run` 在提示词构建完成后、任何模型输入之前运行，
包括提示词本地图片加载和 `llm_input` 观测。它通过 `prompt`
接收当前用户输入，并通过 `messages` 接收已加载的会话历史记录和当前系统提示词。
返回 `{ outcome: "block", reason, message? }` 可在模型读取提示词之前停止运行。`reason` 供内部使用；
`message` 是面向用户的替代文本。仅支持 `pass` 和 `block`
结果；不受支持的决策结构会以关闭方式失败。

运行被阻止时，OpenClaw 仅在 `message.content` 中存储替代文本，
以及阻止插件 ID 和时间戳等非敏感阻止元数据。原始用户文本不会保留在文字记录
或未来上下文中。内部阻止原因被视为敏感信息，并从文字记录、历史记录、广播、
日志和诊断载荷中排除。可观测性应使用经过净化的字段，例如阻止者 ID、结果、
时间戳或安全类别。

包括 `agent_end` 在内的 Agent 轮次钩子会在 OpenClaw 能识别当前运行时包含
`event.runId`；相同值也会出现在 `ctx.runId` 上。定时任务驱动的运行还会在
Agent 轮次上下文中公开 `ctx.jobId`（来源定时任务 ID），使钩子可以将指标、
副作用或状态限定到特定定时任务。`ctx.jobId` 不属于 `before_tool_call`
工具上下文。

对于源自渠道的运行，`ctx.channel` 和 `ctx.messageProvider` 用于标识
`discord` 或 `telegram` 等提供商界面，而当 OpenClaw 能从
会话键或投递元数据推导时，`ctx.channelId` 是对话目标标识符。

当发送者身份可用时，Agent 钩子上下文还包括：

- `ctx.senderId` - 渠道范围内的发送者 ID（例如 Feishu `open_id`、Discord
  用户 ID）。当运行源自具有已知发送者元数据的用户消息时填充。
- `ctx.chatId` - 传输原生的对话标识符（例如 Feishu
  `chat_id`、Telegram `chat_id`）。当来源渠道提供原生对话 ID 时填充。
- `ctx.channelContext.sender.id` - 与 `ctx.senderId` 相同的发送者 ID，位于
  渠道自有对象下，插件可使用渠道特定字段扩展该对象。
- `ctx.channelContext.chat.id` - 与 `ctx.chatId` 相同的对话 ID，
  位于渠道自有对象下，插件可使用渠道特定字段扩展该对象。

核心仅定义嵌套的 `id` 字段。通过入站辅助程序传递更丰富
发送者或聊天元数据的渠道插件，可以从 `openclaw/plugin-sdk/channel-inbound`
扩充 `PluginHookChannelSenderContext` 或 `PluginHookChannelChatContext`：

```ts
declare module "openclaw/plugin-sdk/channel-inbound" {
  interface PluginHookChannelSenderContext {
    unionId?: string;
    userId?: string;
  }
}
```

渠道插件通过入站 SDK 辅助程序传递这些字段：

```ts
buildChannelInboundEventContext({
  // ...
  channelContext: {
    sender: { id: senderOpenId, unionId, userId },
    chat: { id: chatId },
  },
});
```

这些字段为可选字段，在系统发起的运行（Heartbeat、
定时任务、Exec 事件）中不存在。

`ctx.senderExternalId` 仍作为旧版插件的已弃用源代码兼容字段保留。
核心不会填充它；新的渠道特定发送者身份应通过模块扩充放在
`ctx.channelContext.sender` 下。

`agent_end` 是一个观测钩子。Gateway 网关和持久化 harness 路径会在轮次结束后
以即发即弃方式运行它，而短生命周期的一次性 CLI 路径会在进程清理前等待
钩子 Promise，以便受信任的插件可以刷新终端可观测性数据或捕获状态。
钩子运行程序会应用 30 秒超时，防止卡死的插件或嵌入端点使钩子 Promise
永久保持待处理状态。超时会被记录，OpenClaw 将继续运行；除非插件还使用
自己的中止信号，否则不会取消插件自有的网络任务。

将 `model_call_started` 和 `model_call_ended` 用于不应接收原始提示词、历史记录、
响应、标头、请求正文或提供商请求 ID 的提供商调用遥测。这些钩子包含
`runId`、`callId`、`provider`、`model`、
可选的 `api`/`transport`、终止状态
`durationMs`/`outcome` 等稳定元数据；当 OpenClaw 能推导出
有界的提供商请求 ID 哈希时，还会包含 `upstreamRequestIdHash`。当运行时已解析
上下文窗口元数据时，钩子事件和上下文还会包含 `contextTokenBudget`，
即应用模型/配置/Agent 上限后的有效 token 预算；应用更低上限时，还会包含
`contextWindowSource` 和 `contextWindowReferenceTokens`。

`before_agent_finalize` 仅在 harness 即将接受自然生成的最终助手回答时运行。
它不是 `/stop` 取消路径，且不会在用户中止轮次时运行。
返回 `{ action: "revise", reason }` 可要求 harness 在最终确定前再执行一次模型调用，
返回 `{ action:
"finalize", reason? }` 可强制最终确定，也可省略结果以继续。
处理程序的默认时间预算为 15s；超时时，OpenClaw 会记录失败并
继续使用原始最终回答。
Codex 原生 `Stop` 钩子会中继到此钩子，作为 OpenClaw
`before_agent_finalize` 决策。

返回 `action: "revise"` 时，插件可以包含 `retry` 元数据，
以使额外的模型调用有界且可安全重放：

```typescript
type BeforeAgentFinalizeRetry = {
  instruction: string;
  idempotencyKey?: string;
  maxAttempts?: number;
};
```

`instruction` 会附加到发送给 harness 的修订原因中。
`idempotencyKey` 允许主机跨等效的最终确定决策统计同一插件请求的重试次数，
而 `maxAttempts` 会限制主机允许执行的额外调用次数，超过后将继续使用
自然生成的最终回答。

需要原始对话钩子（`before_model_resolve`、
`before_agent_reply`、`llm_input`、`llm_output`、`before_agent_finalize`、
`agent_end` 或 `before_agent_run`）的非内置插件必须设置：

```json
{
  "plugins": {
    "entries": {
      "my-plugin": {
        "hooks": {
          "allowConversationAccess": true
        }
      }
    }
  }
}
```

可以使用 `plugins.entries.<id>.hooks.allowPromptInjection=false` 按插件禁用会修改提示词的钩子和持久化的下一轮注入。

### 会话扩展和下一轮注入

工作流插件可以使用 `api.session.state.registerSessionExtension(...)` 持久化少量兼容 JSON 的会话状态，
并通过 Gateway 网关的 `sessions.pluginPatch` 方法更新该状态。会话行通过
`pluginExtensions` 投影已注册的扩展状态，使 Control UI 和其他客户端能够
呈现插件自有状态，而无需了解插件内部机制。
`api.registerSessionExtension(...)` 仍然有效，但已弃用，建议改用 `api.session.state`
命名空间。

当插件需要将持久化上下文恰好一次传递到下一次模型轮次时，请使用
`api.session.workflow.enqueueNextTurnInjection(...)`（顶层的 `api.enqueueNextTurnInjection(...)` 是行为相同的已弃用别名）。
OpenClaw 会在提示词钩子之前提取排队的注入内容，丢弃已过期的注入内容，
并按插件的 `idempotencyKey` 去重。这是适用于审批恢复、策略摘要、
后台监控器增量，以及应在下一轮对模型可见但不应成为永久系统提示词文本的
命令续接的正确接缝。

清理语义是契约的一部分。会话扩展清理和运行时生命周期清理回调会接收
`reset`、`delete`、`disable` 或
`restart`。主机会在重置/删除/禁用时移除所属插件的持久化会话扩展状态
和待处理的下一轮注入内容；重启会保留持久化会话状态，同时清理回调允许插件
为旧运行时世代释放调度器任务、运行上下文和其他带外资源。

## 消息钩子

使用消息钩子处理渠道级路由和投递策略：

- `message_received`：观测入站内容、发送者、`threadId`、
  `messageId`、`senderId`、可选的运行/会话关联、有序的 `media`
  以及元数据。
- `message_sending`：重写 `content` 或返回 `{ cancel: true }`。
- `reply_payload_sending`：重写规范化的 `ReplyPayload` 对象
  （包括 `presentation`、`delivery`、媒体引用和文本）或返回
  `{ cancel: true }`。
- `message_sent`：观测最终成功或失败。

对于仅包含音频的 TTS 回复，即使渠道载荷中没有可见文本/字幕，
`content` 也可能包含隐藏的语音文字记录。重写该 `content`
只会更新钩子可见的文字记录；它不会呈现为媒体字幕。

`reply_payload_sending` 事件可能包含 `usageState`，即尽力提供的实时
每轮模型/用量/上下文快照。持久化投递、恢复的重放以及没有精确运行关联的
回复会省略该字段。

消息钩子上下文在可用时会公开稳定的关联字段：
`ctx.sessionKey`、`ctx.runId`、`ctx.messageId`、`ctx.senderId`、`ctx.trace`、
`ctx.traceId`、`ctx.spanId`、`ctx.parentSpanId` 和 `ctx.callDepth`。入站
和 `before_dispatch` 上下文还会在渠道具有经过可见性过滤的引用消息数据时公开回复元数据：`replyToId`、`replyToIdFull`、
`replyToBody`、`replyToSender` 和 `replyToIsQuote`。在读取旧版元数据之前，优先使用这些
一等字段。

在使用渠道特定元数据之前，优先使用类型化的 `threadId` 和 `replyToId` 字段。

入站声明和消息接收事件将 `media?:
PluginHookMediaFact[]` 作为规范附件 API 公开。每条事实可携带
`path`、`url`、`contentType`、`kind`、`transcribed`、`messageId` 和
`workspaceDir`；数组位置即附件标识。当远程附件
尚未暂存到本地时，将省略 `media`，
`mediaStagingPending: true`，而 `originalMedia` 包含提供商侧的
事实。在后续暂存事件提供 `media` 之前，不要将
`originalMedia.path` 视为可在本地读取。

单数/复数形式的 `mediaPath`、`mediaUrl`、`mediaType`、`mediaPaths`、
`mediaUrls`、`mediaTypes` 及对应的 `originalMedia*` 元数据属性是
已弃用的兼容性别名。新钩子应使用类型化的顶层
数组。

决策规则：

- 带有 `cancel: true` 的 `message_sending` 是终止性决策。
- 带有 `cancel: false` 的 `message_sending` 被视为未作决策。
- 重写后的 `content` 会继续传递给优先级更低的钩子，除非后续钩子
  取消投递。
- `reply_payload_sending` 在负载规范化之后、渠道
  投递之前运行，包括路由回原始渠道的回复。
  处理程序按顺序运行，每个处理程序都能看到优先级更高的处理程序生成的最新负载。
- `reply_payload_sending` 负载不会公开 `trustedLocalMedia` 等运行时信任标记；
  插件可以编辑负载结构，但无法授予本地
  媒体信任。
- `message_sending` 可以在取消时返回 `cancelReason` 和有界的 `metadata`。
  新消息生命周期 API 将此公开为原因是 `cancelled_by_message_sending_hook` 的受抑制
  投递结果；为保持兼容性，旧版
  直接投递仍返回空结果数组。
- `message_sent` 仅用于观察。处理程序失败会被记录，但不会
  更改投递结果。

## 安装钩子

使用 `security.installPolicy` 执行由操作员管理的允许/阻止决策。该
策略从 OpenClaw 配置运行，覆盖 CLI 安装和更新路径，并且
在启用但不可用时采用故障关闭策略。

`before_install` 是插件运行时生命周期钩子。它仅在
插件钩子已加载的 OpenClaw 进程中于 `security.installPolicy` 之后运行，
例如由 Gateway 网关支持的安装流程。它适用于
插件自身的观察、警告和兼容性检查，但不是
安装的主要企业或主机安全边界。为保持兼容性，
事件负载中仍保留 `builtinScan` 字段，但 OpenClaw
不再执行内置的安装时危险代码阻止，因此它是空的
`ok` 结果。返回其他发现或
`{ block: true, blockReason }` 可停止该进程中的安装。

`block: true` 是终止性决策。`block: false` 被视为未作决策。处理程序
失败会采用故障关闭策略阻止安装。

## Gateway 网关生命周期

使用 `gateway_start` 启动常规插件服务，并使用 `gateway_stop`
清理长期运行的资源。`gateway_start` 运行时，定时任务调度器可能仍在加载，
因此不要将其用作外部
定时任务投影的基准信号。

不要依赖内部 `gateway:startup` 钩子来运行插件自身的运行时
服务。

`cron_reconciled` 在 Gateway 网关的定时任务调度器及其退出时
观察器完成持久状态协调后触发。它会在初始
启动以及配置重新加载期间替换调度器时触发。事件会报告
`reason`（`startup` 或 `reload`）以及有效的 `enabled` 状态。禁用的
定时任务仍会发出带有 `enabled: false` 的事件，使外部投影能够
清除陈旧的唤醒项。使用 `ctx.getCron?.()` 获取完成协调的确切调度器实例；
后续重新加载不会将该回调重新指向其他实例。
`ctx.abortSignal` 拥有同一调度器快照。一旦更新的调度器进入待命状态或开始关闭，
Gateway 网关就会中止它。将它传递给每个
持久副作用，并且中止后不要接受该快照。
这是调度器生命周期信号，而不是插件激活信号：
仅插件热重载不会重放该信号。新启用的使用方会在下一次
替换调度器或启动 Gateway 网关时收到第一个基准。

与其他观察钩子一样，`gateway_start` 和 `cron_reconciled` 回调
可能重叠。如果两个处理程序共享插件初始化，请使用
插件本地的就绪 Promise 进行协调，而不要依赖回调顺序。

`cron_changed` 会针对 Gateway 网关拥有的定时任务生命周期事件触发，并携带类型化的
事件负载，涵盖 `added`、`updated`、`removed`、`started`、`finished`
和 `scheduled` 原因。该事件携带一个 `PluginHookGatewayCronJob`
快照（存在时包括 `state.nextRunAtMs`、`state.lastRunStatus` 和
`state.lastError`），以及一个取值为
`not-requested` | `delivered` | `not-delivered` | `unknown` 的 `PluginHookGatewayCronDeliveryStatus`。移除事件
发生在提交后：只有在持久删除成功后才会触发，并且仍会携带
已删除的任务快照，以便外部调度器协调状态。

`scheduled` 事件发生在提交后：只有当成功的持久
写入更改现有任务的有效 `nextRunAtMs` 时才会触发，并排除该任务的
显式 `added`、`updated` 或 `removed` 生命周期事件。顶层
`event.nextRunAtMs` 是已提交的下一次唤醒；如果不存在，则该任务
没有下一次唤醒。将这些事件视为协调提示，而不是有序的增量
日志。将其作为可合并的提示，用于重新读取 `cron_reconciled` 最后捕获的调度器；
不要采用 `cron_changed` 上下文中的调度器。
对于到期检查和执行，请继续将 OpenClaw 作为事实来源。

### 安全的外部定时任务投影

投影完整的唤醒快照，而不是转发定时任务事件增量。外部
适配器的 `replaceAll` 操作必须是原子且幂等的，并且只有在
主机持久接受快照后才能完成。它还必须
遵循提供的中止信号：如果信号在持久
接受之前中止，适配器不得接受该快照。

此模式确保同一时间只有一个最新状态工作器在运行。只有 `cron_reconciled`
会采用调度器实例；`cron_changed` 仅请求该工作器重新读取
权威实例，因此迟到的提示无法恢复较旧的调度器。
更新的修订会中止活跃的主机尝试，防止其接受陈旧
快照。

```typescript
import { setTimeout as sleep } from "node:timers/promises";
import type { OpenClawPluginApi } from "openclaw/plugin-sdk/plugin-entry";

type ExternalWake = { jobId: string; runAtMs: number };

type ExternalWakeHost = {
  replaceAll(wakes: readonly ExternalWake[], options: { signal: AbortSignal }): Promise<void>;
  close(): Promise<void>;
};

type CronReader = {
  list(options: { includeDisabled: true }): Promise<
    Array<{
      id: string;
      enabled?: boolean;
      state?: { nextRunAtMs?: number };
    }>
  >;
};

export function registerCronProjection(api: OpenClawPluginApi, host: ExternalWakeHost) {
  const lifecycle = new AbortController();
  let cron: CronReader | undefined;
  let enabled = false;
  let hasBaseline = false;
  let reconciliationSignal: AbortSignal | undefined;
  let requestedRevision = 0;
  let appliedRevision = 0;
  let worker = Promise.resolve();
  let activeAttempt: AbortController | undefined;

  const projectLatest = async () => {
    let retryMs = 1_000;

    while (!lifecycle.signal.aborted && appliedRevision < requestedRevision) {
      const ownerSignal = reconciliationSignal;
      if (!ownerSignal || ownerSignal.aborted) {
        return;
      }
      const targetRevision = requestedRevision;
      const attempt = new AbortController();
      const signal = AbortSignal.any([lifecycle.signal, ownerSignal, attempt.signal]);
      activeAttempt = attempt;

      try {
        const jobs = enabled && cron ? await cron.list({ includeDisabled: true }) : [];
        if (signal.aborted || targetRevision !== requestedRevision) {
          continue;
        }
        const wakes = jobs
          .flatMap((job): ExternalWake[] => {
            const runAtMs = job.enabled === false ? undefined : job.state?.nextRunAtMs;
            return runAtMs === undefined ? [] : [{ jobId: job.id, runAtMs }];
          })
          .sort((a, b) => a.runAtMs - b.runAtMs || a.jobId.localeCompare(b.jobId));

        await host.replaceAll(wakes, { signal });
        if (signal.aborted || targetRevision !== requestedRevision) {
          continue;
        }
        appliedRevision = targetRevision;
        retryMs = 1_000;
      } catch {
        if (lifecycle.signal.aborted || ownerSignal.aborted) {
          return;
        }
        if (attempt.signal.aborted) {
          continue;
        }
        api.logger.warn(`外部定时任务投影失败；将在 ${retryMs}ms 后重试`);
        try {
          await sleep(retryMs, undefined, { signal });
        } catch {
          if (lifecycle.signal.aborted) {
            return;
          }
          if (attempt.signal.aborted) {
            continue;
          }
        }
        retryMs = Math.min(retryMs * 2, 30_000);
      } finally {
        if (activeAttempt === attempt) {
          activeAttempt = undefined;
        }
      }
    }
  };

  const requestProjection = () => {
    const targetRevision = ++requestedRevision;
    activeAttempt?.abort();
    worker = worker.then(async () => {
      if (!lifecycle.signal.aborted && appliedRevision < targetRevision) {
        await projectLatest();
      }
    });
    return worker;
  };

  api.on("cron_reconciled", (event, ctx) => {
    const reconciledCron = ctx.getCron?.();
    if (event.enabled && !reconciledCron) {
      api.logger.warn("定时任务协调未公开调度器");
      return;
    }
    cron = reconciledCron;
    enabled = event.enabled;
    hasBaseline = true;
    reconciliationSignal = ctx.abortSignal;
    return requestProjection();
  });

  api.on("cron_changed", () => {
    if (hasBaseline) {
      return requestProjection();
    }
  });

  api.on("gateway_stop", async () => {
    lifecycle.abort();
    await worker;
    await host.close();
  });
}
```

当 `cron_reconciled` 报告 `enabled: false` 时，同一路径会调用
`replaceAll([])` 并清除陈旧的外部唤醒项。此示例中的重试/退避
位于进程本地，并将运行时适配器故障视为暂时性故障；请在注册前
验证不可重试的配置。OpenClaw 不为插件钩子副作用提供
发件箱。如果进程在持久接受之前退出，
下次 Gateway 网关启动会发出新的权威 `cron_reconciled` 快照。
`gateway_stop` 会中止正在进行的主机工作，等待工作器结束，然后
关闭适配器。

## 即将弃用的功能

一些与钩子相邻的接口已弃用但仍受支持。请在
下一个主要版本发布前迁移：

- `inbound_claim` 和 `message_received`
  处理程序中的**纯文本渠道信封**。请读取 `BodyForAgent` 和结构化用户上下文块，
  而不是解析扁平的信封文本。请参阅
  [纯文本渠道信封 → BodyForAgent](/zh-CN/plugins/sdk-migration#active-deprecations)。
- **`subagent_spawning`** 为兼容旧插件而保留，但
  新插件不应从中返回线程路由。在 `subagent_spawned` 触发之前，核心会通过
  渠道会话绑定适配器准备 `thread: true` 子智能体绑定。
- **`deactivate`** 将继续作为已弃用的清理兼容别名保留，
  直至 2026-08-16 之后。新插件应使用 `gateway_stop`。
- **`before_tool_call` 中的 `onResolution`** 现在使用带类型的
  `PluginApprovalResolution` 联合类型（`allow-once` / `allow-always` / `deny` /
  `timeout` / `cancelled`），而不是自由形式的 `string`。
- **`api.registerSessionExtension` / `api.enqueueNextTurnInjection`** 仍作为
  顶层兼容别名保留。新插件应使用
  `api.session.state.registerSessionExtension(...)` 和
  `api.session.workflow.enqueueNextTurnInjection(...)`。

完整列表包括记忆能力注册、提供商思考配置文件、外部身份验证提供商、
提供商发现类型、任务运行时访问器，以及 `command-auth` → `command-status`
重命名，请参阅
[插件 SDK 迁移 → 当前弃用项](/zh-CN/plugins/sdk-migration#active-deprecations)。

## 相关内容

- [插件 SDK 迁移](/zh-CN/plugins/sdk-migration) - 当前弃用项和移除时间表
- [构建插件](/zh-CN/plugins/building-plugins)
- [插件 SDK 概览](/zh-CN/plugins/sdk-overview)
- [插件入口点](/zh-CN/plugins/sdk-entrypoints)
- [内部钩子](/zh-CN/automation/hooks)
- [插件架构内部机制](/zh-CN/plugins/architecture-internals)
