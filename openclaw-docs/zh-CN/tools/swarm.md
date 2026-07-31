---
read_when:
    - 你想用代码模式脚本将工作分发给多个智能体
    - 你需要结构化的子智能体结果、决策门控或首次完成流水线
    - 你正在启用或调整 `tools.swarm` 限制
    - 你想在会话仪表板中查看收集器子项
sidebarTitle: Swarm
summary: 通过代码模式脚本编排并发子智能体，提供结构化结果、受限扇出和实时进度
title: Swarm
x-i18n:
    generated_at: "2026-07-26T07:04:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0bec17da7a2e144df35189a65d9b35d829815b545a4bb89652e6a681ca971a9
    source_path: tools/swarm.md
    workflow: 16
---

Swarm 是一种实验性的、可选择启用的方式，可通过
[代码模式](/tools/code-mode)脚本编排多个子智能体。使用普通的 JavaScript 或 TypeScript
控制流（例如 `Promise.all`、`while` 和 `if`）来分派工作、收集
结果并做出决策。

它没有图 DSL，也没有单独的工作流格式。程序本身就是
编排。Swarm 为该程序添加了可等待的收集器子项、结构化结果、
有界并发和进度报告。

## 启用 Swarm

推荐方式是在 Control UI 中打开 **设置 → 实验室 → Swarm**。该
开关会立即生效，并将 `tools.swarm.enabled` 写入你的
配置。

你也可以直接在 `openclaw.json` 中启用 Swarm：

```json5
{
  tools: {
    swarm: {
      enabled: true,
      maxConcurrent: 8,
      maxChildrenPerGroup: 50,
      maxTotalPerGroup: 200,
      waitTimeoutSecondsMax: 600,
      defaultAgentId: "",
    },
  },
}
```

布尔值简写可启用或禁用此功能，所有其他值均使用
默认值：

```json5
{
  tools: {
    swarm: true,
  },
}
```

| 字段                    | 默认值 | 说明                                                                                                                           |
| ----------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `enabled`               | `false` | 公开收集器模式的生成选项、`agents_wait` 和代码模式的 `agents.*` 访客 API。                                   |
| `maxConcurrent`         | `8`     | 一个 Swarm 组中并发运行的收集器子项最大数量。额外获准的子项按 FIFO 顺序排队。          |
| `maxChildrenPerGroup`   | `50`    | 一个组中存活的收集器子项最大数量。                                                                                  |
| `maxTotalPerGroup`      | `200`   | 一个组在其生命周期内可以生成的收集器子项最大数量。这是防止失控生成的最后保障。                            |
| `waitTimeoutSecondsMax` | `600`   | 单次 `agents_wait` 调用接受的最大超时时间。该调用的默认值为 30 秒。                                            |
| `defaultAgentId`        | `""`    | 生成时省略 `agentId` 所使用的目标智能体。空值表示使用发出请求的智能体。现有的子智能体允许列表仍然适用。 |

数值必须为正整数。OpenClaw 将
`maxConcurrent` 限制为 `1`–`1000`，将 `maxChildrenPerGroup` 限制为 `1`–`10000`，
将 `maxTotalPerGroup` 限制为 `1`–`100000`，并将 `waitTimeoutSecondsMax` 限制为
`1`–`86400`。

你可以使用 `agents.entries.*.tools.swarm` 为某个已配置的智能体
覆盖 Swarm 设置。每智能体对象会覆盖合并到顶层
`tools.swarm` 对象之上。

## 要求

`agents.run`、`phase` 和 `log` 访客全局对象同时需要启用 Swarm 和
OpenClaw 代码模式：

```json5
{
  tools: {
    codeMode: true,
    swarm: true,
  },
}
```

代码模式还必须具有对 `sessions_spawn` 的有效访问权限。工具配置文件、
允许/拒绝策略、提供商规则和沙箱策略都可能移除该工具。
如果脚本报告 `sessions_spawn` 不可用，请参阅[代码模式激活](/tools/code-mode#activation)和
[子智能体](/zh-CN/tools/subagents)。

`defaultAgentId` 和每次运行的 `agentId` 值必须指定一个已配置的目标，
且该目标须获得请求方 `subagents.allowAgents` 策略的允许。OpenClaw 会拒绝
未知或不允许的目标，而不会回退到其他智能体。

## 编写 Swarm 脚本

启用 Swarm 后，代码模式会公开以下访客 API：

```typescript
type AgentRunOptions = {
  label?: string;
  model?: string;
  thinking?: string;
  fastMode?: boolean | "auto";
  agentId?: string;
  schema?: Record<string, unknown>;
  phase?: string;
};

agents.run(prompt: string, options?: AgentRunOptions & { schema?: undefined }): Promise<string>;
agents.run<T>(prompt: string, options: AgentRunOptions & { schema: Record<string, unknown> }): Promise<T>;
phase(title: string): void;
log(message: string): void;
```

如果没有 `schema`，`agents.run()` 将解析为子项的最终文本。如果提供了
JSON Schema，它将解析为子项通过
`structured_output` 工具提交的值。失败、被终止、超时或架构无效的子项
会以 `SwarmAgentError` 拒绝 promise。可在代码模式内通过
`API.read("agents.d.ts")` 查看生成的准确声明和简短的编排惯用写法。

使用 `label` 可为仪表板和侧边栏中的子项设置易于识别的名称。使用
选项中的 `phase` 可在该子项启动前立即发布一个阶段，
或者当多个子项属于同一阶段时调用 `phase()`。
`log()` 会发布一条简短的进度说明。进度调用采用即发即弃方式；
即使 UI 不可用，也不会延迟脚本。

### 并行分派并获得结构化结果

此示例为每个主题启动一个研究智能体，等待所有智能体完成，然后
让最后一个子项综合它们的结构化报告：

```javascript
const reportSchema = {
  type: "object",
  properties: {
    finding: { type: "string" },
    evidence: { type: "array", items: { type: "string" } },
    confidence: { type: "number" },
  },
  required: ["finding", "evidence", "confidence"],
  additionalProperties: false,
};

const topics = ["身份验证", "存储", "恢复"];
phase("独立审查");

const reports = await Promise.all(
  topics.map((topic) =>
    agents.run(`审查 ${topic} 路径。返回一项发现及其证据。`, {
      label: `review-${topic}`,
      thinking: "high",
      fastMode: "auto",
      schema: reportSchema,
    }),
  ),
);

phase("综合");
log(`已收集 ${reports.length} 份独立报告。`);

return await agents.run(
  `协调这些报告并解释其中的分歧：\n${JSON.stringify(reports)}`,
  { label: "synthesis" },
);
```

`Promise.all` 是分派和汇聚的边界。OpenClaw 最多为该组启动
`maxConcurrent` 个子项，其余子项按提交
顺序排队。

代码模式还通过 `tools.codeMode.maxPendingToolCalls` 单独限制并发访客桥接调用
（默认值为 `16`，最大值为 `128`）。对于非常
大的组，应以低于该限制的数量启动有界批次，并为
`phase()`、`log()` 和子项等待状态转换预留余量。`maxConcurrent` 限制的是正在运行的
子项数量；它不会提高访客桥接调用限制。

### 循环执行决策关卡

当每一轮都需要决定是否再执行一轮时，请使用有界的 `while`
循环：

```javascript
const gateSchema = {
  type: "object",
  properties: {
    ready: { type: "boolean" },
    reason: { type: "string" },
    nextAction: { type: "string" },
  },
  required: ["ready", "reason", "nextAction"],
  additionalProperties: false,
};

let pass = 0;
let decision = { ready: false, reason: "尚未检查", nextAction: "审查" };

while (!decision.ready && pass < 4) {
  pass += 1;
  phase(`决策轮次 ${pass}`);
  decision = await agents.run(
    `检查发布证据是否完整。上一次决策：${JSON.stringify(decision)}`,
    {
      label: `release-gate-${pass}`,
      schema: gateSchema,
    },
  );
  log(decision.reason);
}

if (!decision.ready) {
  throw new Error(`经过 ${pass} 轮后关卡仍未开放：${decision.nextAction}`);
}

return decision;
```

决策循环必须始终设置边界。`maxTotalPerGroup` 是最终的安全保障，
不能替代明确的停止条件。

### 处理第一个完成的子项

`agents.run()` 返回普通 promise，因此 `Promise.race` 可以响应第一个
完成的代码模式子项。对于调用底层工具的 harness，
`agents_wait` 提供相同的首次完成边界：只要至少一个请求的运行完成，
或有界超时时间到期，它就会返回。
完整的排空循环请参阅[从其他 harness 使用 Swarm](#use-swarm-from-other-harnesses)。

## 收集器子项的行为方式

收集器子项是普通的隔离子智能体会话，但完成路径不同。
它们会写入持久化的收集器结果供父项等待，而不是将回复通知或引导回
父会话。

目标智能体按以下顺序解析：

1. 生成或 `agents.run()` 调用中的 `agentId`。
2. `tools.swarm.defaultAgentId`。
3. 发出请求的智能体。

当 Swarm 子项需要更小的工具表面、更便宜的模型或更严格的沙箱策略时，
专用的轻量工作智能体会很有用。OpenClaw 不提供内置的
`worker` 智能体 ID；将其指定为默认值之前，请先进行配置。
在该工作智能体的每智能体配置中使用 `tools.swarm: false` 对其进行强化，使其
可以被生成，但不能从自身的顶层会话启动 Swarm：

```json5
{
  tools: { swarm: { enabled: true, defaultAgentId: "worker" } },
  agents: {
    list: [
      {
        id: "main",
        default: true,
        subagents: { allowAgents: ["worker"] },
      },
      { id: "worker", tools: { swarm: false } },
    ],
  },
}
```

收集器审批采用失败即关闭策略。子项绝不会打开操作员审批
提示。需要审批的工具操作会被拒绝，子项可以
在结果中报告该拒绝，以便脚本决定下一步操作。

对于结构化输出，OpenClaw 会向子项添加一个合成的 `structured_output`
工具，并根据提供的 JSON Schema 验证其有效载荷。无效或缺失的
有效载荷会触发一次纠正提示。如果重试后仍未通过验证，
收集器完成结果会保留子项的原始文本，不设置
`structured`，并包含 `schemaError`。底层 `agents_wait`
结果会公开这些字段，以便执行显式恢复逻辑。

### 子项是叶节点

默认情况下，Swarm 子项是叶节点。通用的
`agents.defaults.subagents.maxSpawnDepth` 防护会在默认深度 `1`
阻止子项生成自己的子项。常见的编排惯用方式是
将工作返回给父项，而不是从子项生成更多工作：

```javascript
const plan = await agents.run("将此作业规划为彼此独立的任务。", {
  schema: {
    type: "object",
    properties: { tasks: { type: "array", items: { type: "string" } } },
    required: ["tasks"],
    additionalProperties: false,
  },
});
return await Promise.all(plan.tasks.map((task) => agents.run(task)));
```

嵌套子智能体需要操作员通过
`agents.defaults.subagents.maxSpawnDepth` 主动选择启用，不建议在 Swarm 中使用。
组上限、预算和可观测性均以扁平收集器组为前提。

每个子项只有一个准入所有者。通知型和交互式子项使用
`agents.defaults.subagents.maxChildrenPerAgent`（默认值为 `5`），且不计入
收集器子项。收集器子项仅使用 `maxChildrenPerGroup` 和
`maxTotalPerGroup`；它们不消耗每会话子项预算。生成
深度防护仍适用于这两种模式。

获准后，超出 `maxConcurrent` 的子项会在其 Swarm
组内按 FIFO 顺序排队，并嵌套在全局子智能体通道内。这些并发层会让
工作排队，而不是拒绝工作。超过任一组上限的收集器生成请求
会被拒绝，错误中会包含相关的配置键。

## 观察 Swarm

Swarm 活跃时，在 Control UI 中打开父会话的仪表板。
Swarm 小组件会渲染每个活跃的收集器组，每个子项对应一个圆点，并显示
已排队、正在运行、已完成或失败状态。标签会显示在圆点的工具提示中，因此简短、
稳定的标签能让大型 Swarm 更易于阅读。

会话侧边栏会保留常规的父子树。展开父项行，
即可检查收集器子项或打开其记录，而不会丢失 Swarm
层级结构。

收集器结果在其组被归档前始终可等待。所有成员都达到各自的保留期限后，OpenClaw 会批量归档该组的子项，使已完成的 Swarm 不会继续留在实时会话树中。

## 从其他 harness 使用 Swarm

无需使用 OpenClaw 代码模式也可使用 Swarm。其核心工具与 harness 无关：使用 `sessions_spawn({ collect: true })` 启动收集器子项，并通过有界的 `agents_wait` 调用取回其结果。

Codex 代码模式会自动在 `tools.*` 下公开符合条件的动态 OpenClaw 工具。它不使用 OpenClaw 的 QuickJS 客体 API，也不要求 `tools.codeMode`，但仍必须启用 `tools.swarm`。Codex harness 的 `agents_wait` 调用支持完整的 600 秒超时。

在当前支持的 Codex 运行时中，动态 OpenClaw 工具的结果会以 JSON 文本形式传入代码模式。读取字段前应解析每个结果。Codex 还会串行执行动态工具调用，因此 `Promise.all` 不会并发提交多个 `sessions_spawn` 调用。应在有界循环中启动收集器；后续启动请求提交时，已接受的子项仍可继续运行。

```javascript
function parseToolResult(value) {
  if (typeof value !== "string") return value;
  return JSON.parse(value);
}

const tasks = [
  "检查身份验证路径。",
  "检查存储路径。",
  "检查恢复路径。",
];
const launches = [];

for (const [index, task] of tasks.entries()) {
  const launch = parseToolResult(
    await tools.sessions_spawn({
      task,
      collect: true,
      label: `review-${index + 1}`,
    }),
  );
  if (launch.status !== "accepted") {
    throw new Error(launch.error ?? "收集器生成请求未被接受。");
  }
  launches.push(launch);
}

const pending = new Set(launches.map((launch) => launch.runId));
const completed = [];

while (pending.size > 0) {
  const ids = [...pending].slice(0, 1000);
  const batch = parseToolResult(
    await tools.agents_wait({
      ids,
      timeoutSeconds: 30,
    }),
  );

  // 将此有界窗口轮转到尚未检查的 ID 之后。
  for (const runId of ids) {
    if (pending.delete(runId)) pending.add(runId);
  }

  for (const item of batch.completed) {
    pending.delete(item.runId);
    if (item.status !== "done") {
      throw new Error(item.schemaError ?? item.result ?? `${item.runId}: ${item.status}`);
    }
    completed.push(item); // 每个结果完成后立即处理。
  }

  for (const failure of batch.errors ?? []) {
    pending.delete(failure.runId);
    throw new Error(`${failure.runId}: ${failure.error}`);
  }
}

return completed;
```

每次 `agents_wait` 调用接受 1–1000 个运行 ID。它返回：

```typescript
type AgentsWaitResult = {
  completed: Array<{
    runId: string;
    status: "done" | "failed" | "killed" | "timeout";
    result: string;
    structured?: unknown;
    schemaError?: string;
    sessionKey: string;
    label?: string;
    usage?: { inputTokens: number; outputTokens: number };
  }>;
  pending: string[];
  errors?: Array<{
    runId: string;
    error: "not_found" | "not_owner";
  }>;
};
```

出现以下任一情况时，调用会立即返回：请求的任一子项已经完成；至少一个待处理子项完成；没有剩余的有效待处理 ID；或者调用超时。已完成记录具有幂等性，因此传入已经完成的运行 ID 会再次返回其结果。只有发起生成的会话或其已获授权的父级链才能等待收集器。

这是有界长轮询，而非繁忙状态循环。持续仅传入剩余的运行 ID，直至 `pending` 为空。收集器模式支持原生 OpenClaw 子智能体；不支持 ACP 运行时、线程绑定、可见会话或持久会话模式。

## 限制与路线图

Swarm v1 运行一次性收集器子项；计划中的 `agents.session()` API 将增加有状态的多轮工作节点。子项目前在本地 Gateway 网关的子智能体通道上运行；云端放置计划作为显式生成选项提供。已保存的工作流定义和图 DSL 不属于 Swarm 当前的发展方向。

## 相关内容

- [代码模式](/tools/code-mode)：了解 QuickJS 客体运行时和激活规则
- [子智能体](/zh-CN/tools/subagents)：了解子项策略、隔离和会话行为
- [多 Agent 沙盒工具](/zh-CN/tools/multi-agent-sandbox-tools)：了解按 Agent 配置的限制
- [工具概览](/zh-CN/tools)：了解工具配置文件和策略路由
