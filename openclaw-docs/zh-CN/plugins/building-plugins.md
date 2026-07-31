---
doc-schema-version: 1
read_when:
    - 你想创建一个新的 OpenClaw 插件
    - 你需要插件开发的快速开始指南
    - 你正在频道、提供商、CLI 后端、工具或 Hooks 文档之间进行选择
sidebarTitle: Getting Started
summary: 几分钟内创建你的第一个 OpenClaw 插件
title: 构建插件
x-i18n:
    generated_at: "2026-07-26T06:23:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d156ea305e46d3ca311a0b2cfc42e2c4522f6f10eb70cdd5526d9e9fcd7d4ef
    source_path: plugins/building-plugins.md
    workflow: 16
---

插件无需更改核心即可扩展 OpenClaw。插件可以添加消息渠道、模型提供商、本地 CLI 后端、智能体工具、钩子、媒体提供商或其他由插件拥有的能力。

无需将外部插件添加到 OpenClaw 仓库。将软件包发布到 [ClawHub](/clawhub)，用户可通过以下命令安装：

```bash
openclaw plugins install clawhub:<package-name>
```

在发布切换期间，裸软件包说明符仍会从 npm 安装。如果希望通过 ClawHub 解析，请使用 `clawhub:` 前缀。

## 要求

- Node 22.22.3+、Node 24.15+ 或 Node 25.9+，以及 `npm` 或 `pnpm`。
- TypeScript ESM 模块。
- 对于仓库内的内置插件开发，请克隆仓库并运行 `pnpm install`。
  源码检出环境中的插件开发仅支持 pnpm，因为 OpenClaw 会从
  `extensions/*` 工作区软件包中发现内置插件。

## 选择插件形式

<CardGroup cols={2}>
  <Card title="渠道插件" icon="messages-square" href="/zh-CN/plugins/sdk-channel-plugins">
    将 OpenClaw 连接到消息平台。
  </Card>
  <Card title="提供商插件" icon="cpu" href="/zh-CN/plugins/sdk-provider-plugins">
    添加模型、媒体、搜索、抓取、语音或实时提供商。
  </Card>
  <Card title="CLI 后端插件" icon="terminal" href="/zh-CN/plugins/cli-backend-plugins">
    通过 OpenClaw 模型回退运行本地 AI CLI。
  </Card>
  <Card title="工具插件" icon="wrench" href="/zh-CN/plugins/tool-plugins">
    注册智能体工具。
  </Card>
</CardGroup>

## 快速开始

通过注册一个必需的智能体工具来构建最小工具插件。这是最精简的实用插件形式，涵盖软件包、清单、入口点和本地验证。

<Steps>
  <Step title="创建软件包元数据">
    <CodeGroup>

```json package.json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "typebox": "1.1.39"
  },
  "peerDependencies": {
    "openclaw": ">=2026.3.24-beta.2"
  },
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
  }
}
```

```json openclaw.plugin.json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds a custom tool to OpenClaw",
  "contracts": {
    "tools": ["my_tool"]
  },
  "activation": {
    "onStartup": true
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

    </CodeGroup>

    已发布的外部插件应将运行时入口指向构建后的 JavaScript 文件。有关完整的入口点契约，请参阅 [SDK 入口点](/zh-CN/plugins/sdk-entrypoints)。

    每个插件都需要清单，即使没有配置也不例外。运行时工具必须出现在 `contracts.tools` 中，以便 OpenClaw 无需急切加载每个插件运行时即可发现所有权。请有意设置 `activation.onStartup`；此示例会在 Gateway 网关启动时加载。

    主机信任的插件表面同样受清单约束，并且已安装插件需要显式声明：`api.registerAgentToolResultMiddleware(...)` 要求在 `contracts.agentToolResultMiddleware` 中列出每个目标运行时，而 `api.registerTrustedToolPolicy(...)` 要求在 `contracts.trustedToolPolicies` 中列出每个策略 ID。这些声明可使安装时检查与运行时注册保持一致。

    有关每个清单字段，请参阅[插件清单](/zh-CN/plugins/manifest)。

  </Step>

  <Step title="注册工具">
    ```typescript index.ts
    import { Type } from "typebox";
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

    export default definePluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Adds a custom tool to OpenClaw",
      register(api) {
        api.registerTool({
          name: "my_tool",
          description: "Echo one input value",
          parameters: Type.Object({ input: Type.String() }),
          outputSchema: Type.Object(
            { input: Type.String() },
            { additionalProperties: false },
          ),
          async execute(_id, params) {
            const details = { input: params.input };
            return {
              content: [{ type: "text", text: `Got: ${params.input}` }],
              details,
            };
          },
        });
      },
    });
    ```

    非渠道插件请使用 `definePluginEntry`。渠道插件则改用 `openclaw/plugin-sdk/core` 中的 `defineChannelPluginEntry`。

  </Step>

  <Step title="测试运行时">
    对于已安装或外部插件，请检查已加载的运行时：

    ```bash
    openclaw plugins inspect my-plugin --runtime --json
    ```

    如果插件注册了 CLI 命令，也请运行该命令并确认输出，例如 `openclaw demo-plugin ping`。

    对于此仓库中的内置插件，OpenClaw 会从 `extensions/*` 工作区发现源码检出环境中的插件软件包。运行最接近的针对性测试：

    ```bash
    pnpm test extensions/my-plugin/
    pnpm check
    ```

  </Step>

  <Step title="测试软件包安装">
    发布可打包插件之前，请测试用户实际会获得的相同安装形式。首先添加构建步骤，将 `openclaw.extensions` 等运行时入口指向 `./dist/index.js` 之类的构建后 JavaScript，并确保 `npm pack` 包含该 `dist/` 输出。TypeScript 源码入口仅用于源码检出环境和本地开发路径。

    然后打包插件，并使用 `npm-pack:` 安装 tarball：

    ```bash
    npm pack --pack-destination /tmp
    openclaw plugins install npm-pack:/tmp/<plugin-package>.tgz --force
    openclaw plugins inspect my-plugin --runtime --json
    ```

    `npm-pack:` 使用 OpenClaw 管理的每插件 npm 项目，因此能够发现源码检出测试可能掩盖的运行时依赖错误。它验证软件包和依赖项形式，而不验证与目录关联的官方信任。运行时导入必须位于 `dependencies` 或 `optionalDependencies` 中；仅留在 `devDependencies` 中的依赖项不会为托管运行时项目安装。

    不要将原始归档/路径安装用作官方或特权插件行为的最终验证。原始源码适用于本地调试，但无法验证与 npm 或 ClawHub 安装相同的依赖路径。如果插件依赖受信任的官方插件状态，请通过目录支持的官方安装，或会记录官方信任的已发布软件包路径，添加第二项验证。有关安装根目录和依赖项所有权的详细信息，请参阅[插件依赖解析](/zh-CN/plugins/dependency-resolution)。

  </Step>

  <Step title="发布">
    发布前验证软件包：

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    ```

    标准 ClawHub 软件包片段位于 `docs/snippets/plugin-publish/`。

  </Step>

  <Step title="安装">
    通过 ClawHub 安装已发布的软件包：

    ```bash
    openclaw plugins install clawhub:your-org/your-plugin
    ```

  </Step>
</Steps>

<a id="registering-agent-tools"></a>

## 注册工具

工具可以是必需的，也可以是可选的。启用插件后，必需工具始终可用。可选工具需要用户显式选择启用，OpenClaw 才会加载其所属插件的运行时。

工具工厂会接收受信任的运行时上下文，包括 `deliveryContext`、可用时当前平台对话的 `nativeChannelId`，以及 `requesterSenderId`。

```typescript
register(api) {
  api.registerTool(
    {
      name: "workflow_tool",
      description: "Run a workflow",
      parameters: Type.Object({ pipeline: Type.String() }),
      outputSchema: Type.Object(
        { pipeline: Type.String() },
        { additionalProperties: false },
      ),
      async execute(_id, params) {
        return {
          content: [{ type: "text", text: params.pipeline }],
          details: { pipeline: params.pipeline },
        };
      },
    },
    { optional: true },
  );
}
```

`outputSchema` 是可选的。它描述了[代码模式](/tools/code-mode)和[工具搜索](/zh-CN/tools/tool-search)所使用的结构化 `details` 值。目录调用会在执行前拒绝无效架构，并在工具钩子执行后验证最终值。对于没有稳定 JSON 结果的工具，请省略它。有关完整契约，请参阅[工具插件](/zh-CN/plugins/tool-plugins#output-contracts)。

使用 `api.registerTool(...)` 注册的每个工具也必须在插件清单中声明：

```json
{
  "contracts": {
    "tools": ["workflow_tool"]
  },
  "toolMetadata": {
    "workflow_tool": {
      "optional": true
    }
  }
}
```

用户通过 `tools.allow` 选择启用：

```json5
{
  tools: { allow: ["workflow_tool"] }, // or ["my-plugin"] for every tool from one plugin
}
```

可选工具控制是否向模型公开工具。当工具或钩子应在模型选择它之后、操作运行之前请求审批时，请使用[插件权限请求](/zh-CN/plugins/plugin-permission-requests)。

对于具有副作用、依赖不常见二进制文件或默认不应公开的能力，请使用可选工具。工具名称不得与核心工具名称冲突；冲突项会被跳过，并在插件诊断中报告。格式错误的注册同样会被跳过并报告：缺少非空的 `name`、`execute` 不是函数，或者工具描述符缺少 `parameters` 对象。

工具工厂会接收由运行时提供的上下文对象。当工具需要记录、显示或适配当前轮次的活动模型时，请使用 `ctx.activeModel`；它可以包含 `provider`、`modelId` 和 `modelRef`。应将其视为信息性运行时元数据，而不是针对本地操作员、已安装插件代码或修改版 OpenClaw 运行时的安全边界。敏感的本地工具仍应要求显式的插件或操作员选择启用，并在活动模型元数据缺失或不适用时以关闭方式失败。

清单声明所有权和发现信息；执行时仍会调用实时注册的工具实现。请保持 `toolMetadata.<tool>.optional: true` 与 `api.registerTool(..., { optional: true })` 一致，以便 OpenClaw 在工具被显式加入允许列表之前避免加载该插件运行时。

## 导入约定

从聚焦的插件 SDK 子路径导入：

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
```

在插件软件包内部，使用 `api.ts` 和 `runtime-api.ts` 等本地聚合文件进行内部导入。不要通过 SDK 路径导入自己的插件。特定于提供商的辅助工具应保留在提供商软件包中，除非该接口确实具有通用性。

自定义 Gateway RPC 方法属于高级入口点。请为其使用插件专属前缀；`config.*`、`exec.approvals.*`、`operator.admin.*`、`wizard.*` 和 `update.*` 等核心管理命名空间仍为保留项，并会解析为 `operator.admin`。`openclaw/plugin-sdk/gateway-method-runtime` 桥接专供声明了 `contracts.gatewayMethodDispatch: ["authenticated-request"]` 的插件 HTTP 路由使用。

有关完整导入映射，请参阅[插件 SDK 概览](/zh-CN/plugins/sdk-overview)。

OpenClaw SDK 兼容性字段带有 TypeScript `@deprecated` 注解，编辑器会将其显示为迁移警告。若要在构建时强制执行这些注解，请启用类型感知规则，例如 [`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/)。Oxlint 不具备类型感知能力，因此无法强制执行这些注解。

## 提交前检查清单

<Check>**package.json** 包含正确的 `openclaw` 元数据</Check>
<Check>**openclaw.plugin.json** 清单存在且有效</Check>
<Check>入口点使用 `defineChannelPluginEntry` 或 `definePluginEntry`</Check>
<Check>所有导入均使用明确的 `plugin-sdk/<subpath>` 路径</Check>
<Check>内部导入使用本地模块，而不是 SDK 自导入</Check>
<Check>测试通过（`pnpm test <bundled-plugin-root>/my-plugin/`）</Check>
<Check>`pnpm check` 通过（仓库内插件）</Check>

## 针对 Beta 版本进行测试

1. 关注 [openclaw/openclaw](https://github.com/openclaw/openclaw/releases) 版本发布（`Watch` > `Releases`）。Beta 标签类似 `v2026.3.N-beta.1`。你也可以在 X 上关注 [@openclaw](https://x.com/openclaw)，获取版本发布公告。
2. Beta 标签出现后，请尽快针对该标签测试你的插件。距离稳定版发布通常只有几个小时。
3. 测试后，在 `plugin-forum` Discord 频道（[discord.gg/clawd](https://discord.gg/clawd)）中你的插件主题帖内发布 `all good` 或说明出现的问题。如果还没有主题帖，请创建一个。
4. 如果出现问题，请创建或更新标题为 `Beta blocker: <plugin-name> - <summary>` 的议题，并添加 `beta-blocker` 标签。在你的主题帖中链接该议题。
5. 向 `main` 提交标题为 `fix(<plugin-id>): beta blocker - <summary>` 的 PR，并在 PR 和 Discord 主题帖中链接该议题。贡献者无法为 PR 添加标签，因此标题是供维护者和自动化系统识别的 PR 端信号。已有 PR 的阻塞问题会被合并；没有 PR 的阻塞问题可能仍会随版本发布。
6. 没有反馈即表示一切正常。错过此时间窗口通常意味着你的修复会在下一个周期合入。

## 后续步骤

<CardGroup cols={2}>
  <Card title="渠道插件" icon="messages-square" href="/zh-CN/plugins/sdk-channel-plugins">
    构建消息渠道插件
  </Card>
  <Card title="提供商插件" icon="cpu" href="/zh-CN/plugins/sdk-provider-plugins">
    构建模型提供商插件
  </Card>
  <Card title="CLI 后端插件" icon="terminal" href="/zh-CN/plugins/cli-backend-plugins">
    注册本地 AI CLI 后端
  </Card>
  <Card title="SDK 概览" icon="book-open" href="/zh-CN/plugins/sdk-overview">
    导入映射和注册 API 参考
  </Card>
  <Card title="运行时辅助工具" icon="settings" href="/zh-CN/plugins/sdk-runtime">
    通过 api.runtime 使用 TTS、搜索和子智能体
  </Card>
  <Card title="测试" icon="test-tubes" href="/zh-CN/plugins/sdk-testing">
    测试工具和模式
  </Card>
  <Card title="插件清单" icon="file-json" href="/zh-CN/plugins/manifest">
    完整的清单架构参考
  </Card>
</CardGroup>

## 相关内容

- [插件钩子](/zh-CN/plugins/hooks)
- [插件架构](/zh-CN/plugins/architecture)
