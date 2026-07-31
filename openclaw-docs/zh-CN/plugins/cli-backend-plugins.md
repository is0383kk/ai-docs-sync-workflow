---
read_when:
    - 你正在构建一个本地 AI CLI 后端插件
    - 你想为 `acme-cli/model` 之类的模型引用注册后端。
    - 你需要将第三方 CLI 映射到 OpenClaw 的文本回退运行器中
sidebarTitle: CLI backend plugins
summary: 构建一个注册本地 AI CLI 后端的插件
title: 构建 CLI 后端插件
x-i18n:
    generated_at: "2026-07-26T06:49:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1923b0829b46a309e4b5a6cbbbfd3dcb76a1e14fe4106310d7a9fb37bca41d70
    source_path: plugins/cli-backend-plugins.md
    workflow: 16
---

CLI 后端插件让 OpenClaw 能够调用本地 AI CLI 作为文本推理
后端。该后端在模型引用中显示为提供商前缀：

```text
acme-cli/acme-large
```

当上游集成已作为本地命令提供、CLI 管理本地登录状态，或 API
提供商不可用时需要备用方案，可使用 CLI 后端。

<Info>
  如果上游服务提供常规 HTTP 模型 API，请改为编写
  [提供商插件](/zh-CN/plugins/sdk-provider-plugins)。如果上游
  运行时管理完整的智能体会话、工具事件、压缩或后台
  任务状态，请使用 [Agent harness](/zh-CN/plugins/sdk-agent-harness)。
</Info>

## 插件负责的内容

CLI 后端插件包含三项契约：

| 契约                 | 文件                   | 用途                                                      |
| -------------------- | ---------------------- | --------------------------------------------------------- |
| 包入口               | `package.json`         | 将 OpenClaw 指向插件运行时模块                            |
| 清单所有权           | `openclaw.plugin.json` | 在运行时加载前声明后端 ID                                 |
| 运行时注册           | `index.ts`             | 使用命令默认值调用 `api.registerCliBackend(...)`                     |

清单是设备发现元数据：它不会执行 CLI 或注册
运行时行为。当插件入口调用
`api.registerCliBackend(...)` 时，运行时行为才会开始。

## 最小后端插件

<Steps>
  <Step title="创建包元数据">
    ```json package.json
    {
      "name": "@acme/openclaw-acme-cli",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      },
      "dependencies": {
        "openclaw": "^2026.3.24"
      },
      "devDependencies": {
        "typescript": "^5.9.0"
      }
    }
    ```

    发布的包必须包含构建后的 JavaScript 运行时文件。如果源
    入口为 `./src/index.ts`，请添加指向构建后
    JavaScript 对应文件的 `openclaw.runtimeExtensions`。请参阅[入口点](/zh-CN/plugins/sdk-entrypoints)。

  </Step>

  <Step title="声明后端所有权">
    ```json openclaw.plugin.json
    {
      "id": "acme-cli",
      "name": "Acme CLI",
      "description": "Run Acme's local AI CLI through OpenClaw",
      "cliBackends": ["acme-cli"],
      "setup": {
        "cliBackends": ["acme-cli"],
        "requiresRuntime": false
      },
      "activation": {
        "onStartup": false
      },
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```

    `cliBackends` 是运行时所有权列表；当模型选择或
    `agentRuntime.id` 提及 `acme-cli` 时，它使 OpenClaw 能够自动加载
    插件。

    `setup.cliBackends` 是描述符优先的设置界面。当需要在不加载插件运行时的情况下，
    让模型发现、新手引导或状态识别后端时，请添加它。
    仅当这些静态描述符足以完成设置时，才使用
    `requiresRuntime: false`。

  </Step>

  <Step title="注册后端">
    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      CLI_FRESH_WATCHDOG_DEFAULTS,
      CLI_RESUME_WATCHDOG_DEFAULTS,
      type CliBackendPlugin,
    } from "openclaw/plugin-sdk/cli-backend";

    function buildAcmeCliBackend(): CliBackendPlugin {
      return {
        id: "acme-cli",
        liveTest: {
          defaultModelRef: "acme-cli/acme-large",
          defaultImageProbe: false,
          defaultMcpProbe: false,
          docker: {
            npmPackage: "@acme/acme-cli",
            binaryName: "acme",
          },
        },
        config: {
          command: "acme",
          args: ["chat", "--output-format", "stream-json", "--prompt", "{prompt}"],
          resumeArgs: [
            "chat",
            "--resume",
            "{sessionId}",
            "--output-format",
            "stream-json",
            "--prompt",
            "{prompt}",
          ],
          output: "jsonl",
          resumeOutput: "jsonl",
          jsonlDialect: "gemini-stream-json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            large: "acme-large-2026",
            fast: "acme-fast-2026",
          },
          sessionArgs: ["--session", "{sessionId}"],
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptFileArg: "--system-file",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          imagePathScope: "workspace",
          reliability: {
            watchdog: {
              fresh: { ...CLI_FRESH_WATCHDOG_DEFAULTS },
              resume: { ...CLI_RESUME_WATCHDOG_DEFAULTS },
            },
          },
          serialize: true,
        },
      };
    }

    export default definePluginEntry({
      id: "acme-cli",
      name: "Acme CLI",
      description: "Run Acme's local AI CLI through OpenClaw",
      register(api) {
        api.registerCliBackend(buildAcmeCliBackend());
      },
    });
    ```

    后端 ID 必须与清单的 `cliBackends` 条目匹配。注册的
    适配器是权威插件代码；OpenClaw 配置负责选择后端，
    但不会重写其命令契约。

  </Step>
</Steps>

## 配置结构

`CliBackendConfig` 描述 OpenClaw 应如何启动和解析 CLI。上面的
完整示例有意使用与内置
`google-gemini-cli` 适配器相同的命令、恢复、JSONL、
模型别名、会话、图像和看门狗字段：

| 字段                                                      | 用途                                                                              |
| --------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `command`                                                 | 二进制文件名或命令的绝对路径                                                     |
| `args`                                                    | 全新运行的基础 argv                                                              |
| `resumeArgs`                                              | 已恢复会话的备用 argv；支持 `{sessionId}`                                   |
| `output` / `resumeOutput`                                 | 解析器：`json`、`jsonl` 或 `text`             |
| `jsonlDialect`                                            | JSONL 事件方言：`claude-stream-json` 或 `gemini-stream-json`                         |
| `liveSession`                                             | 长期运行的 CLI 进程模式（`claude-stdio`）                                    |
| `input`                                                   | 提示词传输方式：`arg` 或 `stdin`                         |
| `maxPromptArgChars`                                       | `arg` 模式回退到 stdin 前允许的最大提示词长度                        |
| `env` / `clearEnv`                                        | 要注入的额外环境变量，或启动前要移除的环境变量名称                               |
| `modelArg`                                                | 模型 ID 前使用的标志                                                             |
| `modelAliases`                                            | 将 OpenClaw 模型 ID 映射到 CLI 原生 ID                                            |
| `sessionArgs`                                             | 如何使用 `{sessionId}` 传递会话 ID                                          |
| `sessionMode`                                             | `always`、`existing` 或 `none`                     |
| `sessionIdFields`                                         | OpenClaw 从 CLI 输出中读取的 JSON 字段                                            |
| `systemPromptArg` / `systemPromptFileArg`                 | 系统提示词传输方式                                                               |
| `systemPromptFileConfigArg` / `systemPromptFileConfigKey` | 系统提示词文件的配置覆盖传输方式（例如 `-c`）                      |
| `systemPromptMode`                                        | `append` 或 `replace`                                         |
| `systemPromptWhen`                                        | `first`、`always` 或 `never`                     |
| `imageArg` / `imageMode`                                  | 图像路径标志以及如何传递多张图像（`repeat` 或 `list`）     |
| `imagePathScope`                                          | 移交前暂存图像文件的位置：`temp` 或 `workspace`               |
| `serialize`                                               | 保持同一后端的运行有序                                                           |
| `reseedFromRawTranscriptWhenUncompacted`                  | 选择启用在压缩前进行有界原始记录重新播种，以安全重置会话                         |
| `reliability.watchdog`                                    | 无输出超时调优，分别用于全新运行和已恢复运行                                     |

优先采用与 CLI 匹配的最小静态配置。仅为真正属于后端的行为
添加插件回调。

## 高级后端钩子

`CliBackendPlugin` 还可以定义：

| 钩子                               | 用途                                                                         |
| ---------------------------------- | --------------------------------------------------------------------------- |
| `normalizeConfig(config, context)` | 使用运行时上下文规范化已注册的静态适配器                                    |
| `resolveExecutionArgs(ctx)`        | 添加请求作用域的标志，例如思考强度或附带问题隔离                            |
| `prepareExecution(ctx)`            | 启动前创建临时身份验证、配置或环境桥接                                      |
| `transformSystemPrompt(ctx)`       | 应用最终的 CLI 特定系统提示词转换                                           |
| `textTransforms`                   | 双向替换提示词/输出                                                         |
| `defaultAuthProfileId`             | 优先使用特定的 OpenClaw 身份验证配置文件                                    |
| `authEpochMode`                    | 决定身份验证变更如何使已存储的 CLI 会话失效                                |
| `nativeToolMode`                   | 声明原生工具是不存在、始终启用，还是可由宿主选择                            |
| `toolAvailabilityEnforcement`      | 声明是否在 argv 或执行暂存阶段强制实施精确工具限制                          |
| `sideQuestionToolMode`             | 声明在 `/btw` 附带问题中禁用的原生工具                          |
| `bundleMcp` / `bundleMcpMode`      | 选择启用 OpenClaw 的回环 MCP 工具桥接                                       |
| `ownsNativeCompaction`             | 后端自行负责压缩——OpenClaw 推迟处理                                         |
| `subscriptionAuthDispatch`         | 使用订阅凭据且选择启用的嵌入式运行通过此后端执行                            |
| `runtimeArtifact`                  | 将脚本启动器限定在其完整的内置包树中                                        |

让这些钩子归提供商所有。当后端钩子能够表达相应行为时，
不要向核心添加 CLI 特定分支。

`prepareExecution(ctx)` 接收 `ctx.contextTokenBudget`，即为本次运行选择的有效令牌
限制。拥有原生压缩能力的后端可以将该预算映射到其
CLI 特定的启动契约中。

`runtimeArtifact` 由插件所有。仅当实时推理轮次
签发或重新验证已核验的设置授权时才会查询它；
普通 CLI 运行不需要它。没有此声明的后端无法
签发已核验的 CLI 设置授权。`bundled-package-tree` 声明会指定
确切的 `package.json` 所有者，并要求包入口点就是该
命令。OpenClaw 会对有边界的完整已安装包目录树进行哈希计算，其中包括
嵌套依赖项；如果存在重定向符号链接、
声明包之外的启动器、必需的外部依赖项
声明、过大的目录树或未知脚本，则采用故障关闭策略。仅当该
目录树包含完整推理实现时才进行此声明；可选工具集成
并不能使外部实现依赖图变得安全。

如果同一后端还提供自包含的原生可执行文件，请在
`nativeExecutableNames` 中列出其规范基本名称。其他原生命令仍然
未经验证。

对于普通轮次，`ctx.executionMode` 为 `"agent"`；对于
临时 `/btw` 调用，则为 `"side-question"`。当 CLI 需要不同的一次性标志时，
例如为 BTW 禁用原生工具、会话持久化或恢复行为，
请使用它。如果后端通常具有 `nativeToolMode: "always-on"`，但其
旁路问题 argv 能可靠地禁用这些工具，还应设置
`sideQuestionToolMode: "disabled"`；否则，当 BTW 要求
无工具的 CLI 运行时，OpenClaw 会采用故障关闭策略。

仅当后端能够为单次运行禁用所有
后端原生工具时，才设置 `nativeToolMode: "selectable"`。受限运行会收到规范
契约：`ctx.toolAvailability.native` 是确切的后端原生工具列表，
`ctx.toolAvailability.openClaw` 是确切的 OpenClaw 工具名称列表。
宿主会独立地将生成的 MCP 配置和授权限制为该
OpenClaw 列表；插件不得在核心中转换它，也不得添加传输前缀。

声明后端如何执行该契约：

- `toolAvailabilityEnforcement: "execution-args"` 要求
  `resolveExecutionArgs`。该钩子必须替换冲突的工具标志，禁用
  可能在所选工具之外执行操作的自定义入口，并为全新运行和恢复运行
  返回具备强制执行能力的 argv。
- `toolAvailabilityEnforcement: "prepare-execution"` 要求
  `prepareExecution`。该钩子必须暂存精确的单次运行策略并返回
  `toolAvailabilityEnforced: true`；缺少确认时会采用故障关闭策略，并且
  OpenClaw 会在启动前清理暂存的资源。

cron `toolsAllow` 等运行时上限会由 OpenClaw
在构建此契约前进行规范化和组展开。原生工具会被禁用，而
没有完整已声明执行路径的后端会在执行前失败。

基于 `v2026.7.2-beta.1` 至 `v2026.7.2-beta.3` 构建的插件仍可
读取已弃用的 `ctx.toolAvailability.mcp` 传输名称投影；当可选择的后端实现
`resolveExecutionArgs` 时，也可以省略 `toolAvailabilityEnforcement`。
OpenClaw 会根据插件包必需的 `openclaw.build.openclawVersion` 元数据识别
这一已发布的 Beta 路径，并在 `2026.8.x` 系列中保留它。新插件和更新后的插件应使用规范
`ctx.toolAvailability.openClaw` 名称，并显式声明
`toolAvailabilityEnforcement: "execution-args"`；该 Beta
兼容路径计划在此窗口结束后移除。

### `ownsNativeCompaction`：选择不使用 OpenClaw 压缩

如果你的后端运行的智能体会压缩其**自己的**转录记录，请设置
`ownsNativeCompaction: true`，这样 OpenClaw 的保护性摘要器就永远不会针对
其会话运行——CLI 压缩生命周期会返回空操作，轮次继续执行。
`claude-cli` 会进行此声明，因为 Claude Code 在内部压缩，
且没有 harness 端点。Codex 等原生 harness 会话则继续路由到
其 harness 压缩端点。

**仅当以下所有条件均成立时才进行声明**，否则延迟处理的
超预算会话可能会一直超出预算或变得陈旧（OpenClaw 不再
挽救它）：

- 后端在转录记录接近其上下文窗口时，能可靠地压缩转录记录或限制其
  大小；
- 后端会持久化可恢复会话，使压缩后的状态能够跨轮次保留
  （例如 `--resume` / `--session-id`）；
- 它不是原生 harness 压缩会话——匹配 `agentHarnessId` 的
  会话会改为路由到 harness 端点。

## MCP 工具桥接

CLI 后端默认不会接收 OpenClaw 工具。如果 CLI 能够使用
MCP 配置，请显式选择启用：

```typescript
return {
  id: "acme-cli",
  bundleMcp: true,
  bundleMcpMode: "codex-config-overrides",
  config: {
    command: "acme",
    args: ["chat", "--json"],
    output: "json",
  },
};
```

支持的桥接模式：

| 模式                     | 用途                                                              |
| ------------------------ | ---------------------------------------------------------------- |
| `claude-config-file`     | 接受 MCP 配置文件的 CLI                              |
| `codex-config-overrides` | 接受 argv 配置覆盖项的 CLI                        |
| `gemini-system-settings` | 从其系统设置目录读取 MCP 设置的 CLI |

仅当 CLI 确实能够使用桥接时才启用它。如果 CLI 有
无法禁用的内置工具层，请设置 `nativeToolMode:
"always-on"`，以便当调用方要求不使用原生
工具时，OpenClaw 可以采用故障关闭策略。如果它能为每次运行禁用所有原生工具，请结合上述
`resolveExecutionArgs` 契约使用 `"selectable"`。

## 选择后端

用户通过独立后端的模型引用前缀选择该后端。声明了规范
`modelProvider` 的后端也可以通过该提供商模型的
`agentRuntime.id` 进行选择。适配器机制仍保留在插件中：

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "openai/gpt-5.6-sol",
        fallbacks: ["acme-cli/large"],
      },
    },
  },
}
```

将凭据放入 OpenClaw 身份验证配置文件或插件自有配置中。确保
已注册命令位于 Gateway 网关服务的 `PATH` 中；需要不同
路径或 argv 的部署应更改或封装插件注册。

## 验证

对于内置插件，请围绕构建器和设置注册添加有针对性的测试，
然后运行该插件的定向测试通道：

```bash
pnpm test extensions/acme-cli
```

对于本地或已安装的插件，请验证发现流程并执行一次真实模型运行：

```bash
openclaw plugins inspect acme-cli --runtime --json
openclaw agent --message "准确回复：后端正常" --model acme-cli/acme-large
```

如果后端支持图像或 MCP，请添加实时冒烟测试，使用真实 CLI
验证这些路径。不要仅依靠静态检查来验证提示词、图像、
MCP 或会话恢复行为。

## 检查清单

<Check>`package.json` 包含 `openclaw.extensions`，并为已发布包提供构建后的运行时入口</Check>
<Check>`openclaw.plugin.json` 声明了 `cliBackends` 和有意设置的 `activation.onStartup`</Check>
<Check>当设置/模型发现应在后端未加载时识别它，`setup.cliBackends` 存在</Check>
<Check>`api.registerCliBackend(...)` 使用与清单相同的后端 ID</Check>
<Check>后端模型前缀或模型范围的 `agentRuntime.id` 能选择该注册</Check>
<Check>会话、系统提示词、图像和输出解析器设置与真实 CLI 契约一致</Check>
<Check>定向测试和至少一次实时 CLI 冒烟测试验证了后端路径</Check>

## 相关内容

- [CLI 后端](/zh-CN/gateway/cli-backends) - 运行时选择和行为
- [Building Plugins](/zh-CN/plugins/building-plugins) - 包和清单基础知识
- [插件 SDK 概览](/zh-CN/plugins/sdk-overview) - 注册 API 参考
- [插件清单](/zh-CN/plugins/manifest) - `cliBackends` 和设置描述符
- [Agent harness](/zh-CN/plugins/sdk-agent-harness) - 完整的外部 Agent Runtimes
