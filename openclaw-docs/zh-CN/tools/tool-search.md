---
read_when:
    - 你希望 OpenClaw 智能体使用大型工具目录，而不必将每个工具 schema 都添加到提示词中
    - 你希望通过一个紧凑的运行时界面统一提供 OpenClaw 工具、MCP 工具和客户端工具
    - 你正在为 OpenClaw 运行实现或调试工具发现功能
summary: 工具搜索：通过搜索、描述和调用来精简大型 OpenClaw 工具目录
title: 工具搜索
x-i18n:
    generated_at: "2026-07-26T06:32:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d31322d5ef108c52fd14d48771cc3c6c43fcfbc4bfb95652bc29a55fd706c903
    source_path: tools/tool-search.md
    workflow: 16
---

工具搜索是 OpenClaw 智能体运行时的一项实验性功能。它为智能体提供一种紧凑的方式来发现和调用大型工具目录。当一次运行有许多可用工具，但模型可能只需要其中少数几个时，它非常有用。

本页介绍 OpenClaw 工具搜索。它并非 Codex 原生的工具搜索或动态工具界面。Codex 原生代码模式、工具搜索、延迟动态工具和嵌套工具调用都是稳定的 Codex harness 界面，不依赖 `tools.toolSearch`。

对于公开 QuickJS-WASI `exec`/`wait` 界面而非工具搜索控制项的通用 OpenClaw 运行时，请参阅[代码模式](/tools/code-mode)。

为 OpenClaw 运行启用后，默认情况下模型会收到一个 `tool_search_code` 工具，以及结构化结果无法通过紧凑桥接传递的所有仅限直接调用工具。代码工具会在隔离的 Node 子进程中运行一小段 JavaScript 代码，并提供 `openclaw.tools` 桥接：

```js
const hits = await openclaw.tools.search("create a GitHub issue");
const tool = await openclaw.tools.describe(hits[0].id);
return await openclaw.tools.call(tool.id, {
  title: "Crash on startup",
  body: "Steps to reproduce...",
});
```

目录可以包含符合目录收录条件的 OpenClaw 工具、插件工具、MCP 工具和客户端提供的工具。模型不会预先看到目录中每个工具的 schema。它会搜索紧凑描述符，在需要精确 schema 时描述一个选定的工具，然后通过 OpenClaw 调用该工具。仅限直接调用的工具仍对模型可见，且不会添加到目录中。

Codex harness 运行不会收到这些实验性的 OpenClaw 工具搜索控制项。OpenClaw 将产品能力作为动态工具传递给 Codex，而稳定的原生代码模式、原生工具搜索、延迟动态工具和嵌套工具调用由 Codex 负责。

## 单个轮次如何运行

在规划阶段，OpenClaw 嵌入式运行器会为此次运行构建有效目录：

1. 解析智能体、配置文件、沙箱和会话的活动工具策略。
2. 列出符合条件的 OpenClaw 工具和插件工具。
3. 通过会话 MCP 运行时列出符合条件的 MCP 工具。
4. 添加为当前运行提供的符合条件的客户端工具。
5. 保持仅限直接调用的工具对模型可见，并为其余符合目录收录条件的工具建立紧凑描述符索引。
6. 在这些仅限直接调用的工具旁公开 OpenClaw 代码桥接、结构化回退工具或紧凑目录界面。

在执行阶段，每次实际工具调用都会返回 OpenClaw。隔离的 Node 运行时不持有插件实现、MCP 客户端对象或密钥。`openclaw.tools.call(...)` 会通过桥接返回 Gateway 网关，常规的策略、审批、钩子、日志和结果处理仍会在其中应用。

## 模式

`tools.toolSearch` 有三种面向模型的模式：

- `code`：公开 `tool_search_code`（默认的紧凑 JavaScript 桥接）以及仅限直接调用的工具。
- `tools`：将 `tool_search`、`tool_describe` 和 `tool_call` 作为普通结构化工具公开，适用于不应接收代码的提供商，同时还会公开仅限直接调用的工具。
- `directory`：公开 `tool_search`、`tool_describe` 和 `tool_call`，以及一个包含可用工具名称和描述的有界提示词目录，适用于应看到工具名称但不应看到每个完整 schema 的提供商。OpenClaw 还可以为当前轮次直接公开一小组有界的可能需要或必需的工具 schema。在此模式下，仅限直接调用的工具仍然可见。

所有模式均使用同一个经过策略筛选的目录和常规 OpenClaw 执行路径。标记为 `catalogMode: "direct-only"` 的工具不会进入该目录，并保持对模型可见。如果当前运行时无法启动隔离的 Node 代码模式子进程，默认的 `code` 模式会在目录压缩前回退到 `tools`。在 `directory` 模式下，客户端提供的工具在当前运行中保持直接可见，而 OpenClaw 工具、插件工具和 MCP 工具可以压缩到目录背后。直接调用一个被隐藏的精确目录名称时，执行前会从同一个已授权目录中加载该工具。

所有模式均为实验性功能。对于较小的 OpenClaw 工具目录，优先直接公开工具；对于 Codex harness 运行，优先使用稳定的 Codex 原生界面。

没有单独的来源选择配置。启用工具搜索后，目录会在常规策略筛选后包含符合目录收录条件的 OpenClaw、MCP 和客户端工具；仅限直接调用的工具则单独保留。

## 存在的原因

大型目录很有用，但开销较高。将每个工具 schema 都发送给模型会增大请求、减慢规划速度，并增加意外选择工具的概率。

工具搜索改变了其形式：

- 直接工具：模型会在生成第一个 token 前看到每个选定的 schema
- 工具搜索代码模式：模型会看到一个紧凑代码工具、一份简短的 API 契约，以及所有仅限直接调用的工具
- 工具搜索工具模式：模型会看到三个紧凑的结构化回退工具，以及所有仅限直接调用的工具
- 工具搜索目录模式：模型会看到一个有界目录、搜索/描述/调用控制项、一小组有界的可能需要或必需的 schema，以及所有仅限直接调用的工具
- 在轮次期间：模型可以按需加载其余 schema

对于小型目录，直接公开工具仍是正确的默认方式。当一次运行可访问许多工具时，尤其是来自 MCP 服务器或客户端提供的应用工具时，工具搜索最为适用。

## API

`openclaw.tools.search(query, options?)`

搜索当前运行的有效目录。结果紧凑，可以安全地放回提示词上下文中。每个命中项都包含一个有界的 TypeScript 风格 `input` 签名，例如 `{ id: string; mode?: "drip" | "flood" }`，因此当该签名已足够时，模型可以跳过 `describe`。受信任的 OpenClaw 核心工具或插件工具还可以包含紧凑的 `output` 提示，例如 `Array<{ id: string; paid: boolean }>`。MCP 和客户端的输出 schema 声明不会提升为此受信任提示。它们不受信任的输入 schema 也会延迟为 `input: "unknown"`；调用前请使用 `describe`。开放、过大或其他不完整的输出 schema 会省略该提示，但仍可通过 `describe` 获取。

```js
const hits = await openclaw.tools.search("calendar event", { limit: 5 });
```

`openclaw.tools.describe(id)`

加载一个搜索结果的完整元数据，包括精确的输入 schema；当工具声明了受信任的完整 `outputSchema` 时，也会加载它。

```js
const calendarCreate = await openclaw.tools.describe("mcp:calendar:create_event");
```

`openclaw.tools.call(id, args)`

通过 OpenClaw 调用选定的工具，并返回原始 `{ tool, result }` 信封。返回 JSON 的工具通常将其值放在 `result.details` 中。如果受信任的工具声明了 `outputSchema`，OpenClaw 会在执行前编译该 schema，并在常规工具钩子处理后验证最终的 `details`，然后再返回目录调用结果。

```js
await openclaw.tools.call(calendarCreate.id, {
  summary: "Planning",
  start: "2026-05-09T14:00:00Z",
});
```

工具作者通过工具的 `outputSchema` 属性声明输出契约。它描述的是 `AgentToolResult.details`，而非渲染后的内容块。请包含所有不会抛出异常的变体；如果结果不稳定，则省略该属性。请参阅[代码模式输出契约](/tools/code-mode#declared-output-contracts)和[工具插件](/zh-CN/plugins/tool-plugins#output-contracts)。

结构化回退模式会以工具形式公开相同的操作：

- `tool_search`
- `tool_describe`
- `tool_call`

目录模式公开：

- `tool_search`
- `tool_describe`
- `tool_call`

它还会保持客户端提供的工具和所有仅限直接调用的工具直接可见，并且可能为当前轮次直接公开一小组有界的可能需要或必需的目录工具 schema。如果有界目录省略了条目，请使用 `tool_search` 查找。如果模型直接请求一个被隐藏的精确目录工具名称，OpenClaw 会在常规执行前从已授权目录中加载该工具。
目录模式下的客户端工具名称不得与 OpenClaw、插件或 MCP 工具名称冲突，因为精确的延迟分派会使用这些名称。

## 运行时边界

代码桥接在短生命周期的 Node 子进程中运行。子进程启动时启用 Node 权限模式，使用空环境，不授予文件系统或网络权限，也不授予子进程或 worker 权限。OpenClaw 会强制执行父进程的实际时间超时，并在超时时终止子进程，包括异步延续开始后。

运行时仅公开：

- `console.log`、`console.warn` 和 `console.error`
- `openclaw.tools.search`
- `openclaw.tools.describe`
- `openclaw.tools.call`

最终调用仍适用常规 OpenClaw 行为：

- 工具允许和拒绝策略
- 按智能体和按沙箱配置的工具限制
- 渠道/运行时工具策略
- 审批钩子
- 插件 `before_tool_call` 钩子
- 会话身份、日志和遥测

## 配置

使用默认代码桥接为 OpenClaw 运行启用工具搜索：

```bash
openclaw config set tools.toolSearch true
```

等效 JSON：

```json5
{
  tools: {
    toolSearch: true,
  },
}
```

改为对 OpenClaw 运行使用结构化回退工具：

```json5
{
  tools: {
    toolSearch: {
      mode: "tools",
    },
  },
}
```

改为对 OpenClaw 运行使用紧凑目录界面：

```json5
{
  tools: {
    toolSearch: {
      mode: "directory",
    },
  },
}
```

调整代码模式超时和搜索结果限制（所示值为默认值）：

```json5
{
  tools: {
    toolSearch: {
      mode: "code",
      codeTimeoutMs: 10000,
      searchDefaultLimit: 8,
      maxSearchLimit: 20,
    },
  },
}
```

运行时将 `codeTimeoutMs` 限制在 1000-60000，将 `maxSearchLimit` 限制在 1-50，并将 `searchDefaultLimit` 限制在 1..`maxSearchLimit`。

禁用该功能：

```json5
{
  tools: {
    toolSearch: false,
  },
}
```

## 提示词和遥测

工具搜索会记录足够的遥测数据，以便与直接公开工具进行比较：

- 发送给 harness 的工具和提示词序列化总字节数
- 目录大小和来源明细
- 搜索、描述和调用次数
- 通过 OpenClaw 执行的最终工具调用
- 选定的工具 ID 和来源

会话日志应能用于回答：

- 模型预先看到了多少个工具 schema
- 模型执行了多少次搜索和描述操作
- 最终调用了哪个工具
- 结果来自 OpenClaw、MCP 还是客户端工具

## E2E 验证

QA Lab Gateway 网关场景使用 OpenClaw 运行时验证两条路径：

```bash
pnpm openclaw qa suite --provider-mode mock-openai --scenario tool-search-gateway-e2e
```

它会创建一个包含大型工具目录的临时虚假插件，启动模拟 OpenAI provider，以直接模式启动一次 Gateway 网关，再启用工具搜索启动一次 Gateway 网关，然后比较提供商请求载荷和会话日志。

该回归测试验证：

1. 直接模式可以调用虚假插件工具。
2. 工具搜索可以调用同一个虚假插件工具。
3. 直接模式将虚假插件工具的 schema 直接暴露给提供商。
4. 工具搜索仅暴露紧凑型桥接器以及所有仅限直接模式的工具。
5. 对于大型虚假目录，工具搜索的请求载荷更小。
6. 会话日志显示预期的工具调用次数和桥接调用遥测数据。

## 失败行为

工具搜索应采用失败时关闭策略：

- 如果某个工具不在有效策略中，搜索不应返回该工具
- 如果所选工具变得不可用，`tool_call` 应失败
- 如果策略或审批阻止执行，调用结果应报告该
  阻止，而不是绕过它
- 如果代码桥接器无法创建隔离运行时，请使用 `mode: "tools"`，或
  为该部署禁用工具搜索

## 相关内容

- [工具和插件](/zh-CN/tools)
- [多 Agent 沙盒和工具](/zh-CN/tools/multi-agent-sandbox-tools)
- [Exec 工具](/zh-CN/tools/exec)
- [ACP Agents 设置](/zh-CN/tools/acp-agents-setup)
- [构建插件](/zh-CN/plugins/building-plugins)
