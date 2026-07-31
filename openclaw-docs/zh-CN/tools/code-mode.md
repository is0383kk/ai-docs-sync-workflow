---
read_when:
    - 你想为一次智能体运行启用 OpenClaw 代码模式
    - 你需要解释为什么代码模式不同于 Codex 代码模式
    - 你正在审查紧凑工具契约、QuickJS-WASI 沙箱、TypeScript 转换或隐藏的工具目录桥接机制
    - 你正在添加或审查内部代码模式命名空间注册表集成
sidebarTitle: Code Mode
summary: 使用 OpenClaw 代码模式，在紧凑的 JavaScript 或 TypeScript 工作流中发现、调用并组合大型工具目录
title: 代码模式
x-i18n:
    generated_at: "2026-07-26T06:25:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a21df3bcfb11668da6dde1f7c69adcc284a28dc491c95f95097ce7f41e5c45bf
    source_path: tools/code-mode.md
    workflow: 16
---

代码模式是一项实验性的、需选择启用的 OpenClaw 智能体运行时功能。启用后，模型不再看到所有已启用工具的 schema；而是看到
`exec`、`wait`，以及任何其结构化结果无法通过
仅支持 JSON 的 guest 桥接层传递的仅直接调用工具。模型会编写一小段 JavaScript 或 TypeScript
程序，用于搜索、描述和调用隐藏的工具目录。

本页介绍的是 OpenClaw 代码模式，而不是 Codex Code Mode。这两个功能
名称相同，控制工具名称（`exec`、`wait`）也相同，但它们是
不同的实现：

- Codex Code Mode 在 Codex coding harness 内运行。其 `exec` 工具是
  自由形式语法工具：模型编写原始 JavaScript 源代码（可选择在开头添加
  用于指定执行选项的 `// @exec: {...}` pragma 行），由 Codex 的进程内 V8 Code Mode 运行时执行。
- OpenClaw 代码模式在通用 OpenClaw 智能体运行时中运行，除非配置了
  `tools.codeMode.enabled: true`，否则处于禁用状态。其 `exec`
  工具接收 JSON `{ code, language }` 载荷，并在 QuickJS-WASI
  工作线程中执行。

两者都是 JavaScript 执行界面，而不是 shell 命令界面。应将它们视为
相互独立、实现方式不同的功能，只是恰好暴露了名称相同的
`exec`/`wait` 工具。

## 功能说明

- 模型可见的工具列表变为 `exec`、`wait`，以及任何仅直接调用工具，
  例如 `computer`，或其图像结果无法通过 guest 桥接层传递的原生视觉
  `image` 加载器。
- `exec` 在隔离的 QuickJS-WASI 工作线程中执行模型生成的 JavaScript 或 TypeScript。
- 所有符合目录条件的已启用工具（OpenClaw 核心、插件、MCP、客户端）都会作为
  独立模型工具被隐藏，并通过 `ALL_TOOLS`
  和 `tools` 暴露给 guest 程序。
- `exec` 描述包含一个有界的快速索引，其中列出准确的 OpenClaw/插件
  目录 ID、精简输入提示，以及当受信任工具提供输出 schema 时的精简声明输出提示。
  它会省略描述、完整 schema、MCP 条目和超出容量的条目；guest 端目录查询仍作为后备方式。
- guest 代码会搜索隐藏目录、描述工具的 schema，并通过
  普通智能体轮次所使用的同一执行路径调用工具（策略、
  审批、钩子和遥测仍然适用）。
- MCP 工具归入 `MCP` 命名空间；在代码模式下，这是
  调用这些工具的唯一支持方式。
- 当嵌套工具调用仍处于待处理状态时，`wait` 会恢复已暂停的代码模式运行。

代码模式只会更改面向模型的编排界面。它不会
取代工具、插件工具、MCP 工具、身份验证、审批策略、渠道
行为或模型选择。

## 使用理由

- 更小的提示词界面：提供商只会获得两个控制工具、一个有界的原生工具
  索引，以及少量必需的直接工具，而不是数十或数百个
  完整工具 schema。
- 更强的编排能力：模型可以在一个代码单元中使用循环、合并、小型转换、
  条件逻辑和并行嵌套工具调用。
- 减少模型往返次数：声明的输出契约让模型能够在一次 `exec` 中调用并
  转换工具结果；未知输出仍优先返回原始值。
- 提供商中立：适用于 OpenClaw、插件、MCP 和客户端工具，
  不依赖提供商原生的代码执行功能。
- 故障时关闭：如果代码模式已启用，但 QuickJS-WASI 运行时
  不可用，则本次运行会失败，而不会静默回退到广泛暴露直接
  工具的模式。

对于启用了大型工具目录的智能体，或需要模型在回答前
搜索、组合并调用多个工具的工作流，此功能最为有用。

对于较小的工具目录，或无法可靠编写短程序的模型，应继续直接暴露工具。如果你希望使用
精简目录，但更偏好结构化的搜索/描述/调用控制，而不是
QuickJS-WASI guest，请使用[工具搜索](/zh-CN/tools/tool-search)。

## 快速开始

### 启用代码模式

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

简写形式：

```json5
{
  tools: {
    codeMode: true,
  },
}
```

当省略 `tools.codeMode`、设置为 `false`，或使用不含
`enabled: true` 的对象时，代码模式保持关闭。

如果你使用配置了 MCP 服务器的沙箱隔离智能体，还需要在沙箱工具策略中允许
内置 MCP 插件，例如
`tools.sandbox.tools.alsoAllow: ["bundle-mcp"]`。请参阅
[配置 - 工具和自定义提供商](/zh-CN/gateway/config-tools#mcp-and-plugin-tools-inside-sandbox-tool-policy)。

设置显式限制以实施更严格的边界：

```json5
{
  tools: {
    codeMode: {
      enabled: true,
      timeoutMs: 10000,
      memoryLimitBytes: 67108864,
      maxOutputBytes: 65536,
      maxSnapshotBytes: 10485760,
      maxPendingToolCalls: 16,
      snapshotTtlSeconds: 900,
      searchDefaultLimit: 8,
      maxSearchLimit: 50,
    },
  },
}
```

### 模型的工作方式

对于具有声明输出的工具，例如
`Array<{ id: string; paid: boolean; tons: number }>`，一个 guest 程序即可完成
选择、调用和转换：

```javascript
const [shipmentTool] = await tools.search("list shipments");
const shipments = await tools.callValue(shipmentTool.id, {});
return shipments.filter((shipment) => !shipment.paid && shipment.tons > 10);
```

当快速索引行以 `-> ?` 结尾时，输出形状未知。第一次
`exec` 必须原样返回 `await tools.callValue(...)`。后续 `exec` 才能
转换已观察到的值。这会额外消耗一个模型轮次，但可防止
模型猜测字段名称。

### 验证当前界面

要在调试时确认模型载荷的形状，请使用
针对性日志启动 Gateway 网关：

```bash
OPENCLAW_DEBUG_CODE_MODE=1 \
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 \
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools \
openclaw gateway
```

启用代码模式后，日志中面向模型的工具名称应为 `exec` 和
`wait`。如需查看完整的已脱敏提供商载荷，请在短期调试会话中添加
`OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted`。

## 使用 Swarm 扇出智能体

[Swarm](/tools/swarm) 会添加 `agents.run()`、`phase()` 和 `log()` guest 全局变量，
用于从代码模式脚本中编排并发子智能体。同时启用
`tools.codeMode` 和 `tools.swarm`，然后使用普通 JavaScript 控制流实现
扇出、决策关卡和结构化收集。Swarm 是一个独立的选择启用
开关；仅启用代码模式不会暴露 `agents.*` API。

## 技术导览

本页其余部分介绍运行时契约和实现细节，
面向维护者、调试工具暴露问题的插件作者，以及
验证高风险部署的操作员。

## 运行时状态

|                     |                                                                                             |
| ------------------- | ------------------------------------------------------------------------------------------- |
| 运行时             | [`quickjs-wasi`](https://github.com/vercel-labs/quickjs-wasi)                               |
| 默认状态       | 已禁用                                                                                    |
| 稳定性           | 实验性 OpenClaw 界面（Codex Code Mode 是独立且稳定的 Codex harness 界面） |
| 目标界面      | 通用 OpenClaw 智能体运行                                                                 |
| 安全立场    | 将模型代码视为恶意代码                                                                       |
| 面向用户的承诺 | 启用代码模式绝不会静默回退到广泛暴露直接工具的模式                  |

## 范围

代码模式负责已准备运行中面向模型的编排形态。它
不负责模型选择、渠道行为、身份验证、工具策略或工具
实现。

范围内：模型可见的控制/直接工具定义、隐藏工具目录
构建、JavaScript/TypeScript guest 执行、QuickJS-WASI 工作线程
运行时、用于搜索/描述/调用的主机回调、可恢复的状态（用于
已暂停的 guest 程序）、输出/超时/内存/待处理调用/快照限制，
以及嵌套工具调用的遥测/轨迹投影。

范围外：提供商原生远程代码执行、shell 执行
语义、修改现有工具授权、持久化的用户编写
脚本、guest 代码中的包管理器/文件/网络/模块访问，以及直接
复用 Codex Code Mode 内部机制。

提供商拥有的工具（例如远程 Python 沙箱）是独立工具。请参阅
[代码执行](/zh-CN/tools/code-execution)。

## 术语

- **代码模式**：OpenClaw 运行时模式，它会隐藏与目录兼容的模型
  工具，并暴露 `exec`、`wait`，以及必需的仅直接调用工具。
- **Guest 运行时**：用于执行模型代码的 QuickJS-WASI JavaScript VM。
- **主机桥接层**：从 guest 代码返回 OpenClaw 的精简 JSON 兼容回调界面。
- **目录**：经过常规工具策略、插件、MCP 和客户端工具解析后，
  在本次运行范围内生效的工具列表。
- **嵌套工具调用**：从 guest 代码通过主机桥接层发起的工具调用。
- **快照**：序列化的 QuickJS-WASI VM 状态，保存后可让 `wait` 继续
  已暂停的代码模式运行。

## 配置

`tools.codeMode.enabled` 是激活开关；仅设置其他字段
不会启用此功能。

| 字段                 | 默认值                        | 限制                                           |
| --------------------- | ------------------------------ | ----------------------------------------------- |
| `enabled`             | `false`                        | 布尔值；仅 `true` 会启用代码模式          |
| `runtime`             | `"quickjs-wasi"`               | 唯一支持的值                            |
| `mode`                | `"only"`                       | 暴露控制/直接工具，并将其余工具编入目录 |
| `languages`           | `["javascript", "typescript"]` | 可为这两者的任意子集                           |
| `timeoutMs`           | `10000`                        | `100`-`60000`                                   |
| `memoryLimitBytes`    | `67108864`                     | `1048576`-`1073741824`                          |
| `maxOutputBytes`      | `65536`                        | `1024`-`10485760`                               |
| `maxSnapshotBytes`    | `10485760`                     | `1024`-`268435456`                              |
| `maxPendingToolCalls` | `16`                           | `1`-`128`                                       |
| `snapshotTtlSeconds`  | `900`                          | `1`-`86400`                                     |
| `searchDefaultLimit`  | `8`                            | 限制为 `maxSearchLimit`                     |
| `maxSearchLimit`      | `50`                           | `1`-`50`                                        |

如果代码模式已启用，但 QuickJS-WASI 无法加载，OpenClaw 会对
本次运行采取故障关闭策略；它不会静默暴露普通工具作为后备方案。

## 激活

代码模式会在确定有效工具策略之后、组装
最终模型请求之前进行评估：

1. 解析智能体、模型、提供商、沙箱、渠道、发送者和运行
   策略。
2. 构建有效的 OpenClaw 工具列表，添加符合条件的插件、MCP 和
   客户端工具。
3. 应用允许/拒绝策略。
4. 如果 `tools.codeMode.enabled` 为 false，则继续按正常方式公开工具。
5. 如果已启用且本次运行有活动工具，则保留必需的仅限直接调用
   工具，并将所有符合目录条件的有效工具注册到代码模式
   目录中。
6. 从模型可见列表中移除已编入目录的工具；在保留的仅限直接调用工具之外，添加 `exec` 和
   `wait`。

有意不使用任何工具的运行（原始模型调用、`disableTools: true`
或空的 `tools.allow` 列表）不会激活代码模式界面，即使
已配置 `tools.codeMode.enabled: true`。对于一次运行，代码模式与 OpenClaw 工具
搜索互斥；如果代码模式激活，工具搜索的
压缩不会激活。

代码模式目录的作用域限定于运行，不得泄漏来自其他
智能体、会话、发送者或运行的工具。

## 模型可见工具

当代码模式处于活动状态时，模型会看到 `exec`、`wait` 以及所有必需的
仅限直接调用工具。所有其他已启用工具均从面向模型的
工具列表中隐藏，并注册到代码模式目录中。

使用 `exec` 进行工具编排、数据联接、循环、并行嵌套调用
和结构化转换。仅当 `exec` 返回可恢复的
`waiting` 结果时才使用 `wait`。

## `exec`

`exec` 启动一个代码模式单元并返回一个结果。输入代码由模型
生成，必须视为恶意代码。

输入：

```typescript
type CodeModeExecInput = {
  code?: string;
  command?: string;
  language?: "javascript" | "typescript";
};
```

规则：

- `code` 或 `command` 中必须有一个非空。
- `code` 是文档说明的模型可见字段。
- `command` 可作为与 exec 兼容的别名，用于钩子策略和
  可信重写（常规 OpenClaw shell exec 工具也使用 `command`
  字段）；当两者同时存在时，其值必须匹配。
- `language` 默认为 `"javascript"`；架构将其公开为扁平
  字符串枚举（`"javascript" | "typescript"`），而不是 `oneOf`/`anyOf` 联合类型，
  因为某些提供商会拒绝这些结构。
- 如果 `language` 为 `"typescript"`，OpenClaw 会在求值前进行转译。
- `exec` 会拒绝 `import`、`require`、动态导入和模块加载器
  模式。
- `exec` 绝不会递归公开常规 shell `exec` 实现。
- 外层代码模式 `exec` 钩子事件会携带 `toolKind: "code_mode_exec"` 和
  `toolInputKind: "javascript" | "typescript"`（如果已知），从而使策略能够
  区分代码模式单元与共用同一工具名称的 shell 风格
  `exec` 调用。

结果：

```typescript
type CodeModeResult = CodeModeCompletedResult | CodeModeWaitingResult | CodeModeFailedResult;

type CodeModeCompletedResult = {
  status: "completed";
  value: unknown;
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};

type CodeModeWaitingResult = {
  status: "waiting";
  runId: string;
  reason: "pending_tools" | "yield";
  pendingToolCalls?: CodeModePendingToolCall[];
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};

type CodeModeFailedResult = {
  status: "failed";
  error: string;
  code?: CodeModeErrorCode;
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};
```

当客户机因仍需模型可见续接的可恢复状态而挂起时，
`exec` 返回 `waiting`——即显式的 `yield_control(...)`，或
在 exec 截止期限内尚未完成的桥接工具调用。结果中包含用于
`wait` 的 `runId`。桥接工具调用——`tools.search`/`describe`/
`call` 和命名空间调用（包括 MCP 命名空间调用）——只要能在截止期限内完成，就会在同一次
`exec`/`wait` 调用中自动清空，因此，一个等待多个工具的
紧凑代码块可以在一个模型轮次中运行完成，而不必为每次 await
强制执行一次模型工具调用。可安全重启的运行绝不会
自动清空；其待处理工作仍会经过重放安全检查。

仅当客户机 VM 没有待处理工作，并且在 OpenClaw 的输出适配器运行后
最终值与 JSON 兼容时，`exec` 才会返回 `completed`。

## `wait`

`wait` 继续运行已挂起的代码模式 VM。

输入：

```typescript
type CodeModeWaitInput = {
  runId: string;
};
```

输出与 `exec` 返回的 `CodeModeResult` 联合类型相同。

之所以存在 `wait`，是因为嵌套的 OpenClaw 工具可能很慢、需要交互、
受审批限制或会流式传输部分更新；主机等待外部工作时，模型
不应需要让一次冗长的 `exec` 调用保持打开状态。

QuickJS-WASI 快照/恢复是其恢复机制：

1. `exec` 对代码求值，直到完成、失败或挂起。
2. 挂起时，OpenClaw 会创建 QuickJS VM 快照并记录待处理的主机
   工作。
3. 待处理工作完成后，`wait` 会恢复 VM 快照，并
   使用稳定名称重新注册主机回调。
4. OpenClaw 将嵌套工具结果传入恢复后的 VM，并清空
   QuickJS 待处理作业。
5. `wait` 返回 `completed`、`failed` 或另一个 `waiting` 结果。

快照是运行时状态，而不是用户工件：它们仅存于
进程内映射中（不会写入数据库或磁盘），具有大小限制和有效期，
并且其作用域限定于创建它们的运行和会话。

出现以下情况时，`wait` 会失败（返回 `failed` 结果）：

- `runId` 未知，或其快照已过期。
- 调用方与已挂起运行不在同一运行/会话作用域内。
- 该 `runId` 已有一个 `wait` 正在执行。
- QuickJS-WASI 恢复失败。
- 恢复会超过 `maxOutputBytes` 或 `maxSnapshotBytes`。

## 客户机运行时 API

```typescript
declare const ALL_TOOLS: ToolCatalogEntry[];
declare const tools: ToolCatalog;
declare const MCP: Record<string, unknown>;
declare const namespaces: Record<string, unknown>;

declare function text(value: unknown): void;
declare function json(value: unknown): void;
declare function yield_control(reason?: string): Promise<void>;
```

`ALL_TOOLS` 是运行作用域目录的紧凑元数据；默认不包含
完整架构。模型可见的 `exec` 描述还包含一个
有界且确定的精确 OpenClaw/插件 ID 子集、紧凑的输入
提示和可信的声明输出提示。描述会保持延迟加载，避免
恶意目录文本操纵模型。当该索引省略某个工具时，
请读取 `ALL_TOOLS`，或在客户机程序中调用 `tools.search(...)`。

每行快速索引中的箭头描述 `tools.callValue(...)` 值。
`-> Array<{ id: string }>` 表示声明的输出提示；`-> ?` 表示输出未知。
未知输出应优先保持原始形式：原样返回该值并进行观察，然后
在后续 `exec` 中筛选或映射它，而不是猜测字段名称。当声明输出的读取结果
传给最终的 `-> ?` 调用时，此规则同样适用：直接返回该
调用的原始值，不要将其包装成请求的答案结构。

```typescript
type ToolCatalogEntry = {
  id: string;
  name: string;
  label?: string;
  description: string;
  source: "openclaw" | "mcp" | "client";
  sourceName?: string;
  input: string;
  output?: string;
};
```

`input` 是适用于常见情况的有界 TypeScript 风格签名。如果仍需要
精确的完整架构，请使用 `tools.describe(...)`。远程 MCP
和客户端条目使用 `input: "unknown"`，使其不可信架构保持
延迟加载，直到 `describe`。只有从可信的 OpenClaw 核心
或插件 `outputSchema` 派生出完整的紧凑提示时，才会存在
`output`。MCP 和客户端的输出架构声明不会提升为
此可信目录提示。

插件工具使用 `source: "openclaw"`，其中 `sourceName` 设置为所属
插件 ID；不存在单独的 `"plugin"` 来源值。`source: "mcp"` 仅用于
`sourceName`/`mcp` 元数据中的 MCP 条目（并会从
`ALL_TOOLS`/`tools.*` 中滤除，见下文）。

仅在需要时加载完整架构：

```typescript
type ToolCatalogEntryWithSchema = ToolCatalogEntry & {
  parameters: unknown;
  outputSchema?: unknown;
};
```

目录辅助函数：

```typescript
type ToolCatalog = {
  search(query: string, options?: { limit?: number }): Promise<ToolCatalogEntry[]>;
  describe(id: string): Promise<ToolCatalogEntryWithSchema>;
  callValue(id: string, input?: unknown): Promise<unknown>;
  call(id: string, input?: unknown): Promise<unknown>;
  [safeToolName: string]: unknown;
};
```

仅为无歧义的安全名称安装便捷工具函数：

```typescript
const files = await tools.search("read local file");
const fileRead = await tools.describe(files[0].id);
const content = await tools.callValue(fileRead.id, { path: "README.md" });

// 如果隐藏目录中存在无歧义的 `web_search` 条目：
const hits = await tools.web_search({ query: "OpenClaw code mode" });
```

`tools.callValue(...)` 直接返回常规工具的 JSON `details` 值。
`tools.call(...)` 会为需要内容块或其他结果元数据的调用方
保留原始 `{ tool, result }` 信封。

## 声明的输出契约

OpenClaw 工具可以为放置在 `AgentToolResult.details` 中的结构化值声明
`outputSchema`。这对代码模式和工具搜索很有用；它
不是提供商原生的工具响应架构，也不会改变工具的直接
公开方式。

对于使用 `defineToolPlugin` 创建的工具，请在
`parameters` 旁声明该架构：

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

const Shipment = Type.Object(
  {
    id: Type.String(),
    paid: Type.Boolean(),
    tons: Type.Number(),
  },
  { additionalProperties: false },
);

export default defineToolPlugin({
  id: "shipping",
  name: "Shipping",
  description: "Shipment tools.",
  tools: (tool) => [
    tool({
      name: "shipping_list",
      description: "List shipments.",
      parameters: Type.Object({}),
      outputSchema: Type.Array(Shipment),
      execute: async () => loadShipments(),
    }),
  ],
});
```

对于 `api.registerTool(...)` 或工厂工具，请将相同的 `outputSchema`
属性放在返回的 `AnyAgentTool` 对象上。

当前内置契约包括 `agents_list`、`apply_patch`、
`conversations_list`、`conversations_send`、`conversations_turn`、`edit`、
`openclaw`、`read`、`screen`、
`sessions_history`、`sessions_list`、`sessions_search`、`sessions_send`、
`session_status`、`spawn_task`、`terminal`、`web_fetch` 和 `web_search`。
精确透传可以复用其所属的协议模式，而无需
重复仅供模型使用的契约。例如，会话工具公开的
Gateway 网关结果模式与 `conversations.list`、
`conversations.send` 和 `conversations.turn` 所使用的相同；`web_fetch` 拥有一个工具本地
模式，其提示公开稳定的元数据、文本、缓存状态和嵌套溢出
元数据；`web_search` 将其精确的规范化结果/答案/错误/原始数据
联合声明为完整的快速索引提示。文件系统契约返回结构化的
读取文本、图像、截断和可选的未找到结果；明确的编辑
变更状态以及 diff/patch 数据；以及应用补丁的路径摘要。当
快速索引声明了这些字段时，一个单元即可组合设备发现和交付，
无需单独的检查轮次：

```javascript
const listed = await tools.conversations_list({ query: "构建机器人" });
const target = listed.conversations.find((item) => item.label === "构建机器人");
if (!target) throw new Error("未找到会话");
return await tools.conversations_send({
  conversationRef: target.conversationRef,
  message: "构建已完成。",
});
```

嵌套调用仍使用常规工具策略、钩子和审批。如果完整
契约是精确的，但对于有界快速索引而言过大，则仍可
通过 `tools.describe(...)` 获取，并且箭头仍为 `-> ?`。

契约规则非常严格：

- 描述精确的 JSON 兼容 `details` 值，而不是渲染后的 `content`
  块或提供商封装。
- 包含每个不会抛出异常的成功或错误变体。当
  工具没有稳定的结构化结果时，省略 `outputSchema`。
- 使用 `{ additionalProperties: false }` 闭合对象层，以形成完整的
  快速索引提示。开放、过大或其他不完整的模式仍可
  通过 `tools.describe(...)` 获取，但不能启用单轮字段使用。
- OpenClaw 在运行工具前编译模式，然后在常规工具
  钩子执行后、目录调用返回前验证最终的 `details`。无效
  模式无法运行工具；不匹配会导致失败，且不会打印该值。
- 紧凑提示是确定且有界的。当紧凑提示不足时，
  `tools.describe(...)` 会公开完整的可信模式。
- 已安装的插件代码已经是可信的本地代码。远程 MCP 和客户端
  元数据仍不受信任，无法选择加入这些快速索引提示。

有关插件创作的详细信息，请参阅[工具插件](/zh-CN/plugins/tool-plugins#output-contracts)。

MCP 目录条目无法在代码模式下通过 `tools.callValue(...)`、
`tools.call(...)` 或便捷函数调用；它们仅通过生成的
`MCP` 命名空间公开。TypeScript 风格的声明文件
可通过只读的 `API` 虚拟文件表面获取，因此智能体可以
检查 MCP 签名，而无需将 MCP 模式添加到提示中：

```typescript
const files = await API.list("mcp");
const githubApi = await API.read("mcp/github.d.ts");

const issue = await MCP.github.createIssue({
  owner: "openclaw",
  repo: "openclaw",
  title: "调查 Gateway 网关日志",
});

const snapshot = await MCP.chromeDevtools.takeSnapshot({ output: "markdown" });
const resource = await MCP.docs.resources.read({ uri: "memo://one" });
const prompt = await MCP.docs.prompts.get({
  name: "brief",
  arguments: { topic: "release" },
});
```

`API.read("mcp/<server>.d.ts")` 返回根据 MCP
工具元数据推断出的紧凑声明：

```typescript
type McpToolResult = {
  content?: unknown[];
  structuredContent?: unknown;
  isError?: boolean;
  [key: string]: unknown;
};

declare namespace MCP.github {
  /** 返回此 TypeScript 风格的 API 标头。 */
  function $api(toolName?: string, options?: { schema?: boolean }): Promise<McpApiHeader>;

  /**
   * 创建 GitHub 议题。
   * @param owner 仓库所有者
   * @param repo 仓库名称
   * @param title 议题标题
   */
  function createIssue(input: {
    owner: string;
    repo: string;
    title: string;
    body?: string;
  }): Promise<McpToolResult>;
}
```

声明文件是虚拟的，不会写入工作区或状态
目录。对于每次代码模式 `exec` 调用，OpenClaw 都会构建运行作用域的工具
目录，保留可见的 MCP 条目，渲染 `mcp/index.d.ts`，并为每个可见服务器渲染一个
`mcp/<server>.d.ts`，然后将这个小型只读表
注入 QuickJS 工作线程。访客代码只能看到 `API` 对象：
`API.list(prefix?)` 返回文件元数据，而 `API.read(path)` 返回
所选声明的内容。未知路径以及 `.`/`..` 段会被
拒绝。

这可以避免将大型 MCP 模式放入模型提示：智能体通过
`exec` 工具描述得知虚拟 API 的存在，仅读取所需的
声明文件，然后使用一个对象参数调用 `MCP.<server>.<tool>()`。
`MCP.<server>.$api()` 仍可作为内联回退方案，用于在程序内
获取单个工具的模式响应。

访客运行时绝不会直接看到主机对象。输入和输出以
JSON 兼容值的形式跨越桥接层，并具有明确的大小上限。

## 内部命名空间

内部命名空间为代码模式提供简洁的领域 API，而不会增加更多
模型可见工具。由加载器所属的集成注册一个命名空间，例如
`Issues` 或 `Calendar`；访客代码随后在 QuickJS 程序内调用该命名空间，
而模型仍只会看到紧凑的控制/直接表面。

命名空间目前仅供内部使用。当前没有公开的插件 SDK 命名空间 API：
外部插件命名空间需要由加载器所有的契约，以确保插件身份、
已安装清单、身份验证状态和缓存的目录描述符不会与
支持该命名空间的插件工具发生偏离。核心代码模式仅负责
沙箱、序列化、目录门控和桥接分派。

访客代码既可以使用直接全局变量，也可以使用 `namespaces` 映射：

```javascript
const open = await Issues.list({ state: "open" });
const alsoOpen = await namespaces.Issues.list({ state: "open" });
return { count: open.length, alsoCount: alsoOpen.length };
```

### 注册表生命周期

命名空间注册表位于进程本地，并以命名空间 ID 为键：

1. 可信加载器调用 `registerCodeModeNamespaceForPlugin(pluginId, registration)`。
2. 代码模式为本次运行创建隐藏的 `ToolSearchRuntime`，并读取其
   运行作用域目录。
3. `createCodeModeNamespaceRuntime(ctx, catalog)` 仅保留
   `requiredToolNames` 全部可见且属于同一 `pluginId` 的注册。
4. 每个可见命名空间针对当前运行调用 `createScope(ctx)`，
   接收 `agentId`、`sessionKey`、`sessionId`、
   `runId`、配置和中止状态等运行上下文。
5. 作用域数据被序列化为普通描述符，并作为直接全局变量和
   `namespaces.<globalName>` 注入 QuickJS。
6. 访客调用通过工作线程桥接层挂起，在主机上解析命名空间路径，
   将调用映射到声明的插件所属目录工具，并
   通过 `ToolSearchRuntime.callExactId` 执行该工具。
7. 就绪的命名空间桥接调用会在活跃的
   `exec`/`wait` 调用内自动排空；如果在超时时命名空间工作仍处于待处理状态，
   或访客显式让出执行，`wait` 会在之后恢复同一命名空间运行时。
8. 插件回滚或卸载会调用
   `clearCodeModeNamespacesForPlugin(pluginId)`，以免过时的全局变量
   在插件加载失败后继续存在。

命名空间调用属于目录工具调用：它们使用与
`tools.call(...)` 相同的策略钩子、审批、中止处理、遥测、记录投影和
挂起/恢复行为。

### 注册结构

从拥有支持工具的集成中注册命名空间。保持
作用域精简，并且只公开映射到已声明目录
工具的领域动词。

```typescript
import {
  createCodeModeNamespaceTool,
  registerCodeModeNamespaceForPlugin,
} from "../agents/code-mode-namespaces.js";

const pluginId = "github";

registerCodeModeNamespaceForPlugin(pluginId, {
  id: "github-issues",
  globalName: "Issues",
  description: "当前仓库的 GitHub 议题辅助工具。",
  requiredToolNames: ["github_list_issues", "github_update_issue"],
  prompt: "使用 Issues.list(params) 和 Issues.update(number, patch)。",
  createScope: (ctx) => ({
    repository: ctx.config,
    list: createCodeModeNamespaceTool("github_list_issues", ([params]) => params ?? {}),
    update: createCodeModeNamespaceTool("github_update_issue", ([number, patch]) => ({
      number,
      patch,
    })),
  }),
});
```

`createCodeModeNamespaceTool(toolName, inputMapper)` 将作用域成员标记为
可调用的命名空间函数。可选的 `inputMapper` 接收访客
参数并返回支持目录工具的输入对象；如果没有
该参数，则使用第一个访客参数，省略时使用 `{}`。

访客代码运行前会拒绝原始主机函数：

```typescript
createScope: () => ({
  // 错误：这会绕过目录工具生命周期，因此会被拒绝。
  list: async () => githubClient.listIssues(),
});
```

### 所有权和可见性

命名空间所有权绑定到注册调用方的 `pluginId`。
`requiredToolNames` 同时是可见性门控和所有权检查：

- 每个必需工具都必须存在于运行目录中
- 每个必需工具都必须具有 `sourceName === pluginId`
- 当任何必需工具缺失或属于
  另一个插件时，命名空间会被隐藏
- 每个可调用路径只能以 `requiredToolNames` 中命名的工具为目标

这可以防止另一个插件通过注册
同名工具来公开命名空间，并使命名空间与普通智能体策略保持一致：如果
本次运行看不到支持工具，也就看不到命名空间。

例如，GitHub 命名空间应位于 GitHub 所属的插件后方，该插件
拥有 GitHub 身份验证、REST/GraphQL 客户端、速率限制、写入审批和
测试。核心代码模式不应嵌入 GitHub 专用 API、令牌处理
或提供商策略。

### 作用域序列化规则

`createScope(ctx)` 可以返回一个普通对象，其中包含 JSON 兼容
值、数组、嵌套对象和 `createCodeModeNamespaceTool(...)` 调用
标记。主机对象绝不会直接进入 QuickJS。

序列化器会拒绝：

- 原始函数
- 循环对象图
- 不安全的路径段：`__proto__`、`constructor`、`prototype`、空键，
  或包含内部路径分隔符的键
- 不是 JavaScript 标识符的 `globalName` 值
- `globalName` 与内置代码模式全局变量发生冲突，例如 `tools`、
  `namespaces`、`text`、`json`、`yield_control`、`MCP`、`API`、`ALL_TOOLS` 或
  `__openclaw*`

无法进行 JSON 序列化的值会在跨越桥接层前转换为 JSON 安全的回退
值。二进制数据、句柄、套接字、客户端和
类实例应保留在普通目录工具后方。

### 提示

仅当命名空间对该次运行可见时，命名空间的 `description` 和可选的 `prompt` 才会附加到模型
可见的 `exec` 模式。使用它们来讲解最小且实用的表面：

```typescript
{
  description: "小说制作服务辅助函数。",
  prompt:
    "使用 Fictions.riskAudit()、Fictions.promoteIfReady(id, status) 和 Fictions.unpaidOver(amount)。",
}
```

提示应聚焦于命名空间契约，而不是身份验证设置、实现历史或无关的插件行为。

### 清理

命名空间是进程本地注册项。当其所属插件被禁用、卸载或回滚时，应将其移除：

```typescript
clearCodeModeNamespacesForPlugin(pluginId);
```

代码模式清理由插件负责；当插件生命周期结束时，应清除该插件的命名空间注册项，而不是保留各命名空间的拆卸句柄。测试可以调用 `clearCodeModeNamespacesForTest()`，以避免注册项在测试用例之间泄漏。

### 测试检查清单

命名空间变更应覆盖安全边界和访客行为：

- 仅当后端工具可见时，命名空间提示文本才会出现
- 来自其他 `sourceName` 的同名工具不会暴露该命名空间
- 原始作用域函数会被拒绝
- 伪造的命名空间 ID 和伪造路径会被拒绝
- 可调用路径不能指向未声明的工具
- 嵌套对象和共享引用能够正确序列化
- 命名空间调用通过目录工具执行，并返回可安全转换为 JSON 的详细信息
- 访客代码可以捕获失败
- 已挂起的命名空间调用通过 `wait` 恢复
- 插件回滚会清除其所属的命名空间注册项

命名空间是通用 `tools.search`/`tools.call` 目录的补充：对于任意已启用的 OpenClaw、插件和客户端工具，请使用该目录；对于 MCP 工具，请使用 `MCP`；对于由插件所有且已有文档说明的领域 API，如果简洁代码比反复查询架构更可靠，请使用其他命名空间。

## 输出 API

- `text(value)` 将人类可读的输出追加到 `output` 数组。
- `json(value)` 在完成 JSON 兼容序列化后追加一个结构化输出项。
- 访客代码最终返回的值会成为 `completed` 结果中的 `value`。

```typescript
type CodeModeOutput = { type: "text"; text: string } | { type: "json"; value: unknown };
```

规则：输出顺序与访客调用顺序一致；输出受 `maxOutputBytes` 限制；不可序列化的值会转换为纯字符串或错误；不支持二进制值。图像和文件通过普通 OpenClaw 工具传输，而不是通过代码模式桥接器。

## 工具目录

隐藏目录包含经过有效策略过滤后的工具，顺序如下：OpenClaw 核心工具、内置插件工具、外部插件工具、MCP 工具，然后是当前运行中由客户端提供的工具。

目录 ID 在单次运行内保持稳定，并尽可能在等效工具集之间保持确定性。实际格式：

```text
<source>:<owner>:<tool-name>
```

其中，`<source>` 为 `openclaw`、`mcp` 或 `client`（插件工具使用 `openclaw`，并以插件 ID 作为 `<owner>`；核心工具使用 `openclaw:core:*`）。
示例：

```text
openclaw:core:message
openclaw:browser:browser_request
mcp:github:create_issue
client:app:select_file
```

该目录会省略代码模式控制工具（`exec`、`wait`、`tool_search_code`、`tool_search`、`tool_describe`、`tool_call`）以及仅限直接调用的工具。控制工具不得通过目录递归调用；仅限直接调用的工具仍对模型可见，因为其结构化结果无法跨越 QuickJS 桥接器。

MCP 条目保留在运行作用域的目录中，因此策略、审批、钩子、遥测、记录投影和精确工具 ID 与普通工具执行共用。面向访客的 `ALL_TOOLS`、`tools.search(...)`、`tools.describe(...)`、`tools.callValue(...)` 和 `tools.call(...)` 视图会省略 MCP 条目。生成的 `MCP.<server>.<tool>({ ...input })` 命名空间会解析回精确的目录 ID，并通过同一执行器路径分派。

## 工具搜索交互

在代码模式处于活动状态的运行中，代码模式会取代 OpenClaw 工具搜索模型界面。

当 `tools.codeMode.enabled` 为 true 且代码模式激活时：

- OpenClaw 不会将 `tool_search_code`、`tool_search`、`tool_describe` 或 `tool_call` 作为模型可见工具暴露。
- 相同的目录化机制会移入访客运行时内部。
- 访客运行时会接收精简的 `ALL_TOOLS` 元数据，以及面向非 MCP 工具的搜索/描述/调用辅助函数。
- MCP 调用使用生成的 `MCP` 命名空间及其 `$api()` 标头，而不是 `tools.call(...)`。
- 嵌套调用通过工具搜索所使用的同一 OpenClaw 执行器路径进行分派。

请参阅[工具搜索](/zh-CN/tools/tool-search)，了解当代码模式处于活动状态时会被其取代的 OpenClaw 精简目录桥接器。

## 工具名称和冲突

模型可见的 `exec` 工具是代码模式工具。如果启用了普通 OpenClaw shell `exec` 工具，它会对模型隐藏，并像其他工具一样加入目录。

在访客运行时内部：

- 如果策略允许，`tools.call("openclaw:core:exec", input)` 可以调用 shell exec 工具。
- 仅当 shell exec 目录条目具有明确且安全的名称时，才会安装 `tools.exec(...)`。
- 代码模式的 `exec` 工具绝不会通过 `tools` 递归提供。

如果两个工具规范化为同一个安全便捷名称，OpenClaw 会省略该便捷函数，并要求使用 `tools.call(id, input)`。

## 嵌套工具执行

每次嵌套工具调用都会跨越主机桥接器并重新进入 OpenClaw，同时保留：当前智能体 ID、会话 ID 和密钥、发送者和渠道上下文、沙箱策略、审批策略、插件 `before_tool_call` 钩子、中止信号、可用时的流式更新，以及轨迹/审计事件。

嵌套调用会作为真实工具调用投影到记录中，以便支持包显示实际发生的情况；投影会标识父级代码模式工具调用和嵌套工具 ID。

最多允许 `maxPendingToolCalls` 个并行嵌套调用。

## 运行和快照生命周期

每次代码模式运行都在一个以 `runId` 为键的进程内映射中进行跟踪（不会持久化到磁盘或数据库）。`exec`/`wait` 会返回三种结果状态之一：`completed`、`waiting` 或 `failed`。

- `waiting` 结果会存储 QuickJS 快照、待处理的桥接请求和作用域元数据（智能体运行 ID、会话 ID/密钥），直到 `wait` 恢复该结果或其过期。
- 过期、会话错误、运行错误以及未知/已在恢复的 `runId` 值不会产生独立的终止状态；它们会表现为带有 `code: "invalid_input"` 的 `failed` 结果，并包含类似 `code mode
run is unavailable or expired.` 或 `code mode run belongs to a different
session.` 的消息。
- 一旦运行结束为 `completed` 或 `failed`，其快照就会从映射中移除；Gateway 网关关闭时也会丢弃快照（重启后不会保留任何内容：这是临时运行时状态）。
- 对于只读工作，`exec` 可以设置 `restartSafe: true`。随后，OpenClaw 会在执行前拒绝会产生副作用的目录调用和插件命名空间，并将挂起的结果标记为可安全重放。如果重启中断 `wait`，[重启恢复](/zh-CN/gateway/restart-recovery)会根据记录重建该轮次，而不是恢复进程本地快照。恢复轮次本身仍仅限于经过审计的只读核心工具和明确可安全重放的插件工具。
- OpenClaw 将每个进程中并发挂起的运行数量限制为 64；超过该上限的新挂起请求会以 `too many suspended code mode
runs.` 拒绝。

快照存储受每次运行的 `maxSnapshotBytes`、上述每进程挂起运行上限以及 `snapshotTtlSeconds` 的限制。

## QuickJS-WASI 运行时

OpenClaw 将 `quickjs-wasi` 作为所属包中的直接依赖项加载；它不依赖为无关依赖项安装的传递副本。

运行时职责：编译/加载 QuickJS-WASI WebAssembly 模块；为每次代码模式运行或恢复创建一个隔离虚拟机；以稳定名称注册主机回调；设置内存和中断限制；执行 JavaScript；清空待处理任务；为已挂起的虚拟机状态创建快照；为 `wait` 恢复快照；在终止状态后释放虚拟机句柄和快照。

该运行时在 Node.js 工作线程中执行，位于 OpenClaw 主事件循环之外。访客代码中的无限循环不得无限期阻塞 Gateway 网关进程；工作线程的中断处理程序会强制执行实际时钟超时，不依赖访客代码主动配合。

## TypeScript

TypeScript 支持仅是一种源代码转换：接受的输入是一个 TypeScript 代码字符串；输出是由 QuickJS-WASI 执行的 JavaScript 字符串。不进行类型检查和模块解析，也没有 `import`/`require`。诊断信息以 `failed` 结果返回。

TypeScript 编译器仅针对 TypeScript 单元延迟加载；纯 JavaScript 单元和已禁用的代码模式不会加载它。

## 安全边界

模型代码是不可信的。运行时采用纵深防御：

- 在主事件循环之外的工作线程中运行 QuickJS-WASI
- 将 `quickjs-wasi` 作为直接依赖项加载，而不是通过 Codex 或传递依赖包加载
- 访客环境中不提供文件系统、网络、子进程、模块导入、环境变量或主机全局对象
- 使用 QuickJS 内存和中断限制，并配合父进程实际时钟超时
- 强制执行输出、快照、日志和待处理调用上限
- 通过狭窄的 JSON 适配器序列化主机桥接值
- 将主机错误转换为普通访客错误，绝不传递主机领域对象
- 在超时、中止、会话结束或过期时丢弃快照
- 拒绝递归访问 `exec`、`wait` 和工具搜索控制工具
- 防止便捷名称冲突遮蔽目录辅助函数

沙箱只是其中一层安全防护；对于高风险部署，操作员可能仍需实施操作系统级加固。

## 错误代码

```typescript
type CodeModeErrorCode =
  | "invalid_input"
  | "runtime_unavailable"
  | "timeout"
  | "output_limit_exceeded"
  | "snapshot_limit_exceeded"
  | "internal_error";
```

`invalid_input` 涵盖错误的 `exec`/`wait` 参数、已禁用的语言、被拒绝的模块访问、TypeScript 转换失败、未知/过期/作用域错误的 `runId` 值，以及挂起运行数量过多。`runtime_unavailable` 涵盖 QuickJS 工作线程启动失败或以非零状态退出的情况。

返回给访客的错误是普通数据；主机 `Error` 实例、堆栈对象、原型和主机函数不会进入 QuickJS。

## 遥测

每个结果的 `telemetry` 字段会报告：隐藏目录大小及来源明细（`openclaw`/`mcp`/`client` 计数）、该运行目录的累计搜索/描述/调用次数，以及模型可见的工具名称（`exec`、`wait` 和保留的仅限直接调用工具）。

除非现有 OpenClaw 轨迹策略允许，否则遥测不得包含机密、原始环境值或未经脱敏的工具输入。

## 调试

当代码模式的行为与普通工具运行不同时，请使用有针对性的模型传输日志：

```bash
OPENCLAW_DEBUG_CODE_MODE=1 \
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 \
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools \
OPENCLAW_DEBUG_SSE=events \
openclaw gateway
```

对于负载结构调试，请使用 `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted`。
这会记录模型请求的经过大小限制和脱敏处理的 JSON 快照；请仅在
调试时使用，因为提示词和消息文本仍可能出现。

对于流式传输调试，请使用 `OPENCLAW_DEBUG_SSE=peek` 记录前五个
经过脱敏处理的 SSE 事件。如果最终提供商负载并非恰好包含一个 `exec`、一个 `wait`，并且在代码模式表面激活后仅包含获准的
仅限直接调用工具，代码模式也会采取失败关闭策略。

## 实现布局

- 配置契约：`tools.codeMode`
- 目录构建器：将有效工具转换为紧凑条目和 ID 映射
- 模型表面适配器：用控制工具/直接调用工具替换可见工具
- QuickJS-WASI 运行时适配器：加载、求值、快照、恢复、释放
- 工作线程监管器：超时、中止、崩溃隔离
- 桥接适配器：JSON 安全的宿主回调和结果传递
- TypeScript 转换适配器
- 快照存储：TTL、大小上限、运行/会话作用域
- 嵌套工具调用的轨迹投影
- 遥测计数器和诊断

该实现复用工具搜索中的目录和执行器概念，但
不使用 `node:vm` 子项作为沙箱。

## 验证清单

代码模式覆盖应证明：

- 禁用配置不会改变现有工具暴露方式
- 不含 `enabled: true` 的对象配置会使代码模式保持禁用
- 当工具在本次运行中处于活动状态时，启用配置仅向
  模型暴露 `exec`、`wait` 和必需的仅限直接调用工具
- 原始无工具运行、`disableTools` 和空允许列表不会触发
  代码模式负载强制检查
- 所有符合目录条件的有效非 MCP 工具都会出现在 `ALL_TOOLS` 中
- 仅限直接调用工具保持对模型可见，且不会出现在 `ALL_TOOLS` 中
- 被拒绝的工具不会出现在 `ALL_TOOLS` 中
- `tools.search`、`tools.describe`、`tools.callValue` 和 `tools.call` 可用于 OpenClaw 工具
- `API.list("mcp")` 和 `API.read("mcp/<server>.d.ts")` 无需桥接/工具调用即可暴露 TypeScript 风格的
  MCP 声明
- MCP 命名空间 `$api()` 仍可用作架构的内联后备方案
- MCP 命名空间调用可用于具有单个对象输入的可见 MCP 工具，同时
  `tools.*` 中不存在直接 MCP 目录条目
- 工具搜索控制工具在模型表面和
  隐藏目录中均不可见
- 嵌套调用会保留审批和钩子行为
- Shell `exec` 对模型隐藏，但在获得
  允许时可通过目录 ID 调用
- 无法从访客代码调用递归代码模式 `exec` 和 `wait`
- TypeScript 输入会经过转换和求值，且不会在
  禁用路径或纯 JavaScript 路径中加载 TypeScript
- `import`、`require`、文件系统、网络和环境访问均会失败
- 无限循环会超时，且无法阻塞 Gateway 网关
- 内存上限故障会终止访客虚拟机
- 已完成和已挂起调用均会强制执行输出及快照上限
- `wait` 会恢复挂起的快照并返回最终值
- 已过期、已中止、会话不匹配和未知的 `runId` 值均会失败
- 记录重放和持久化会保留代码模式控制调用
- 记录和遥测会清晰显示嵌套工具调用

## E2E 测试计划

更改运行时时，将以下测试作为集成测试或端到端测试运行：

1. 使用 `tools.codeMode.enabled: false` 启动 Gateway 网关。
2. 发送一个采用小型直接调用工具集的智能体轮次。
3. 断言模型可见工具保持不变。
4. 使用 `tools.codeMode.enabled: true` 重新启动。
5. 发送一个包含 OpenClaw、插件、MCP 和客户端测试工具的智能体轮次。
6. 断言模型可见工具列表为 `exec`、`wait`，外加仅有的已配置
   仅限直接调用工具。
7. 在 `exec` 中读取 `ALL_TOOLS`，并断言符合目录条件的有效测试
   工具存在，而仅限直接调用工具不存在。
8. 在 `exec` 中，通过 `tools.search`、
   `tools.describe` 和 `tools.callValue`（或原始 `tools.call`）调用 OpenClaw/插件/客户端工具。
9. 在 `exec` 中调用 `API.list("mcp")` 和 `API.read("mcp/<server>.d.ts")`，并
   断言声明文件描述了可见的 MCP 工具。
10. 在 `exec` 中，通过 `MCP.<server>.<tool>({ ...input })` 调用 MCP 工具，并
    断言 `ALL_TOOLS` 和
    `tools.*` 中不存在直接 MCP 目录条目。
11. 断言被拒绝的工具不存在，且无法通过猜测的 ID 调用。
12. 启动一个在 `exec` 返回 `waiting` 后解析的嵌套工具调用。
13. 调用 `wait`，并断言恢复后的虚拟机接收到工具结果。
14. 断言最终答案包含恢复后生成的输出。
15. 断言超时、中止和快照过期会清理运行时状态。
16. 导出轨迹，并断言嵌套调用显示在父级
    代码模式调用下。

对此页面的纯文档更改仍应运行 `pnpm check:docs`。

## 相关内容

- [Swarm](/tools/swarm)，用于通过代码模式脚本编排扇出式智能体
- [工具搜索](/zh-CN/tools/tool-search)
- [Agent Runtimes](/zh-CN/concepts/agent-runtimes)
- [Exec 工具](/zh-CN/tools/exec)
- [代码执行](/zh-CN/tools/code-execution)
