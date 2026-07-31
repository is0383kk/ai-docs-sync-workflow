---
summary: OpenClaw 如何组织内置 Agent 运行时：代码布局、边界、资源清单和运行时选择。
title: Agent runtime architecture
x-i18n:
    generated_at: "2026-07-26T06:01:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3e09ff21b4369a7c102db51e4458ad3ba1e86c9fe43a3a8bff72eef1713d2d51
    source_path: agent-runtime-architecture.md
    workflow: 16
---

OpenClaw 拥有内置智能体运行时。运行时代码位于 `src/agents/` 下，模型/提供商传输位于 `src/llm/` 下，面向插件的契约通过 `openclaw/plugin-sdk/*` barrel 导出。

## 运行时布局

| 路径                                | 负责                                                                                                                                                                                                                      |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/agents/embedded-agent-runner/` | 内置尝试循环（`run.ts`、`run/`）、模型选择和提供商规范化（`model*.ts`）、各提供商的请求参数（`extra-params.*`）、压缩、转录记录和会话接线。                            |
| `src/agents/sessions/`              | 会话持久化（`session-manager.ts`）、资源发现（`package-manager.ts`、`resource-loader.ts`）、会话内 `extensions` 加载、提示词模板、Skills、主题，以及由 TUI 支持的工具渲染器（`tools/`）。 |
| `packages/agent-core/`              | 可复用的智能体核心（`@openclaw/agent-core`）：Agent loop、harness 类型、消息、压缩辅助函数、提示词模板、Skills 和会话存储契约。                                                           |
| `src/agents/runtime/`               | OpenClaw 外观层，将 `@openclaw/agent-core` 接入插件 SDK 的 LLM 运行时，并重新导出该运行时及本地代理实用工具。                                                                                             |
| `src/agents/agent-tools*.ts`        | OpenClaw 自有的工具定义、参数模式、工具策略、工具调用前后适配器，以及主机/沙箱编辑工具。                                                                                            |
| `src/agents/agent-hooks/`           | 内置运行时钩子：压缩保护机制、压缩指令、上下文剪枝。                                                                                                                                   |
| `src/agents/harness/`               | 内置及插件注册 harness 的注册表、选择策略和生命周期。                                                                                                                       |
| `src/llm/`                          | 模型/提供商注册表、传输辅助函数，以及提供商专用的流式实现（`src/llm/providers/`）。                                                                                                          |

## 边界

核心通过 OpenClaw 模块和 SDK barrel 调用内置运行时；不再保留外部智能体框架包。插件使用已文档化的 `openclaw/plugin-sdk/*` 入口点，不导入 `src/**` 内部实现。

`@earendil-works/pi-tui` 仍是第三方依赖：本地 TUI 和会话工具渲染器使用的终端组件工具包。将其内部化需要单独开展依赖内置工作。

## 清单

资源包在 `package.json` 元数据中声明 OpenClaw 资源。条目是相对于包根目录的文件路径或 glob：

```json
{
  "openclaw": {
    "extensions": ["extensions/index.ts"],
    "skills": ["skills/*.md"],
    "prompts": ["prompts/*.md"],
    "themes": ["themes/*.json"]
  }
}
```

清单中未列出的资源类型会回退到发现常规的 `extensions/`、`skills/`、`prompts/` 和 `themes/` 目录。

## 运行时选择

- 内置运行时 ID 为 `openclaw`。旧版别名 `pi` 会规范化为 `openclaw`；`codex-app-server` 会规范化为 `codex`。
- 插件 harness 会注册其他运行时 ID（例如 `codex`）。
- 运行时策略是模型/提供商范围的 `agentRuntime.id` 配置（模型条目优先于提供商条目）。未设置或 `default` 会解析为 `auto`。
- `auto` 会选择支持有效提供商路由的已注册插件 harness，否则选择内置 OpenClaw 运行时。仅凭提供商或模型前缀绝不会选择 harness。
- 仅当路由与官方 HTTPS Platform Responses 或 ChatGPT Responses 路由完全匹配，且没有自行设定的请求覆盖时，OpenAI 才可以隐式选择 `codex`。Completions 适配器、自定义端点以及具有自行设定请求行为的路由仍使用 `openclaw`；官方明文 HTTP 端点会被拒绝。请参阅 [OpenAI 隐式智能体运行时](/zh-CN/providers/openai#implicit-agent-runtime)。

## 模型运行时世代

Gateway 网关启动以及配置、插件或身份验证发布时，会为每个已配置的智能体构建一个预备好的模型运行时世代。每个世代将已发现的身份验证模板、模型注册表和投影后的模型目录作为一个原子快照拥有。智能体运行会从该快照派生可变的身份验证和注册表存储；浏览、状态、定时任务、Doctor、TUI、PDF 和图像路径会读取已发布的目录，而不是重复执行文件系统发现。

独立的嵌入式运行时会在其激活边界发布相同形态的快照。失败或过时的世代绝不会与较新的不完整世代同时提供服务；生命周期所有者必须先发布一个完整的替代世代。

## 相关内容

- [OpenClaw agent runtime workflow](/zh-CN/openclaw-agent-runtime)
- [Agent Runtimes](/zh-CN/concepts/agent-runtimes)
