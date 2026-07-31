---
read_when:
    - 你想构建一个仅添加智能体工具的简单 OpenClaw 插件
    - 你想使用 defineToolPlugin，而不是手动编写插件清单元数据
    - 你需要搭建、生成、验证、测试或发布一个仅包含工具的插件
sidebarTitle: Tool Plugins
summary: 使用 defineToolPlugin 和 openclaw plugins init/build/validate 构建简单的类型化智能体工具
title: 工具插件
x-i18n:
    generated_at: "2026-07-26T06:19:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac23d15ba79cbdd1d8b8eab7c87007b44af16361b2866b14123e18f816bf4075
    source_path: plugins/tool-plugins.md
    workflow: 16
---

`defineToolPlugin` 构建一个仅添加智能体可调用工具的插件：不包含
渠道、模型提供商、钩子、服务或设置后端。它会生成 OpenClaw 所需的
清单元数据，以便无需加载插件运行时代码即可发现工具。

对于提供商、渠道、钩子、服务或混合能力插件，请改从
[构建插件](/zh-CN/plugins/building-plugins)、[渠道插件](/zh-CN/plugins/sdk-channel-plugins)
或[提供商插件](/zh-CN/plugins/sdk-provider-plugins)开始。

## 要求

- Node 22.22.3+、Node 24.15+ 或 Node 25.9+。
- TypeScript ESM 软件包输出。
- `typebox` 位于 `dependencies` 中（不能只位于 `devDependencies` 中，因为生成的
  插件会在运行时导入它）。
- `openclaw >=2026.5.17`，即首个导出
  `openclaw/plugin-sdk/tool-plugin` 的版本。
- 一个会发布 `dist/`、`openclaw.plugin.json` 和
  `package.json` 的软件包根目录。

## 快速开始

```bash
openclaw plugins init stock-quotes --name "Stock Quotes"
cd stock-quotes
npm install
npm run plugin:build
npm run plugin:validate
npm test
```

`plugins init` 会搭建：

| 文件                   | 用途                                                           |
| ---------------------- | ----------------------------------------------------------------- |
| `src/index.ts`         | 包含一个 `echo` 工具的 `defineToolPlugin` 入口                     |
| `src/index.test.ts`    | 断言工具列表的元数据测试                             |
| `tsconfig.json`        | 输出到 `dist/` 的 NodeNext TypeScript                             |
| `vitest.config.ts`     | 用于 `src/**/*.test.ts` 的 Vitest 配置                              |
| `package.json`         | 脚本、运行时依赖项、`openclaw.extensions: ["./dist/index.js"]` |
| `openclaw.plugin.json` | 初始工具的已生成清单元数据                  |

`npm run plugin:build` 会运行 `npm run build`（tsc），然后运行
`openclaw plugins build --entry ./dist/index.js`。`npm run plugin:validate`
会重新构建并运行 `openclaw plugins validate --entry ./dist/index.js`。
验证成功时会输出：

```text
插件 stock-quotes 有效。
```

`openclaw plugins init <id>` 选项：

| 标志                 | 默认值            | 效果                                 |
| -------------------- | ------------------ | -------------------------------------- |
| `--directory <path>` | `<id>`             | 输出目录                       |
| `--name <name>`      | 采用标题式大小写的 `<id>` | 显示名称                           |
| `--type <type>`      | `tool`             | 搭建类型：`tool` 或 `provider`    |
| `--force`            | 关闭                | 覆盖现有输出目录 |

## 编写工具

`defineToolPlugin` 接受插件标识、可选配置架构和
静态工具列表。参数和配置类型从
TypeBox 架构推断。

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "获取股票报价快照。",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "报价 API 密钥。" })),
    baseUrl: Type.Optional(Type.String({ description: "报价 API 基础 URL。" })),
  }),
  tools: (tool) => [
    tool({
      name: "stock_quote",
      label: "股票报价",
      description: "获取股票报价快照。",
      parameters: Type.Object({
        symbol: Type.String({ description: "股票代码，例如 OPEN。" }),
      }),
      outputSchema: Type.Object(
        {
          symbol: Type.String(),
          configured: Type.Boolean(),
          baseUrl: Type.String(),
        },
        { additionalProperties: false },
      ),
      async execute({ symbol }, config, context) {
        context.signal?.throwIfAborted();
        return {
          symbol: symbol.toUpperCase(),
          configured: Boolean(config.apiKey),
          baseUrl: config.baseUrl ?? "https://api.example.com",
        };
      },
    }),
  ],
});
```

工具名称是稳定的 API。应选择唯一、全小写且
足够具体的名称，以避免与核心工具或其他插件发生冲突。

## 可选工具和工厂工具

当用户应先将工具显式加入允许列表，之后才将其发送给
模型时，请设置 `optional: true`。`openclaw plugins build` 会写入匹配的
`toolMetadata.<tool>.optional` 清单条目，因此 OpenClaw 无需加载插件运行时代码
即可发现该工具为可选工具。

```typescript
tool({
  name: "workflow_run",
  description: "运行外部工作流。",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  execute: ({ goal }) => ({ queued: true, goal }),
});
```

当工具需要运行时工具上下文才能创建时，请使用 `factory`，例如针对特定运行
选择退出、检查沙箱状态或绑定
运行时辅助函数。虽然具体工具在运行时构建，但元数据仍保持静态。

```typescript
tool({
  name: "local_workflow",
  description: "在沙箱隔离会话之外运行本地工作流。",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  factory({ api, toolContext }) {
    if (toolContext.sandboxed) {
      return null;
    }
    return createLocalWorkflowTool(api);
  },
});
```

工厂仍需预先声明固定工具名称。当插件动态计算工具名称，或将工具
与钩子、服务、提供商或命令组合时，请直接使用 `definePluginEntry`。

## 返回值

`defineToolPlugin` 会将普通返回值包装为 OpenClaw 工具结果
格式：

- 当模型应看到完全一致的文本时，返回字符串。
- 当你希望模型看到格式化的 JSON，并让 OpenClaw 将原始值保留在 `details` 中时，
  返回与 JSON 兼容的值。

```typescript
tool({
  name: "echo_text",
  description: "回显输入文本。",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => input,
});
```

```typescript
tool({
  name: "echo_json",
  description: "以结构化 JSON 回显输入。",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => ({ input, length: input.length }),
});
```

当需要自定义 `AgentToolResult`，或希望复用现有
`api.registerTool` 实现时，请使用工厂工具。

## 输出契约

当工具返回稳定的 JSON 兼容数据时，请添加 `outputSchema`。它描述的是
存储在 `AgentToolResult.details` 中的原始值，而非
`content` 中的格式化文本：

```typescript
tool({
  name: "shipment_list",
  description: "列出货运记录。",
  parameters: Type.Object({
    buyer: Type.Optional(Type.String()),
  }),
  outputSchema: Type.Array(
    Type.Object(
      {
        id: Type.String(),
        buyer: Type.String(),
        paid: Type.Boolean(),
        tons: Type.Number(),
      },
      { additionalProperties: false },
    ),
  ),
  execute: ({ buyer }) => listShipments(buyer),
});
```

[代码模式](/zh-CN/tools/code-mode)和[工具搜索](/zh-CN/tools/tool-search)会将此
架构转换为有边界的 TypeScript 风格输出提示。这样，模型可以在一个程序中调用并
转换已知结果，而不必再耗费一个模型轮次来
观察其结构。

OpenClaw 会在执行目录调用前编译架构，然后在工具钩子执行后验证最终的
`details` 值，之后再通过桥接返回。无效的架构无法运行工具；结果不匹配会使已完成的
调用失败。请包含所有不会抛出异常的结果变体，包括结构化错误
变体；如果结果不稳定，则省略该架构。不要在架构描述中放置机密信息
或敏感值，因为受信任的输出元数据可能会对模型可见。
当你需要完整、紧凑的输出提示时，请在对象层使用 `{ additionalProperties: false }`；
开放或截断的架构仍可通过 `tools.describe(...)` 使用，但不会作为完整的快速索引契约进行公布。

工厂工具在其返回的具体 `AnyAgentTool` 上声明 `outputSchema`。
静态 `tool({ factory })` 声明不接受单独的
输出架构，因为它可能会与运行时工具产生偏差。

## 配置

`configSchema` 是可选的。省略它时，OpenClaw 会应用严格的空对象
架构；生成的清单仍会包含 `configSchema`。

```typescript
export default defineToolPlugin({
  id: "no-config-tools",
  name: "No Config Tools",
  description: "添加无需配置的工具。",
  tools: () => [],
});
```

使用 `configSchema` 时，第二个 `execute` 参数的类型会从中推断：

```typescript
const configSchema = Type.Object({
  apiKey: Type.String(),
});

export default defineToolPlugin({
  id: "configured-tools",
  name: "Configured Tools",
  description: "添加已配置的工具。",
  configSchema,
  tools: (tool) => [
    tool({
      name: "configured_ping",
      description: "检查配置是否可用。",
      parameters: Type.Object({}),
      execute: (_params, config) => ({ hasKey: config.apiKey.length > 0 }),
    }),
  ],
});
```

OpenClaw 从 Gateway 网关配置中的插件条目读取插件配置。不要
在源代码或文档示例中硬编码机密信息；请根据插件的安全模型使用配置、环境
变量或 SecretRefs。

## 生成的元数据

OpenClaw 必须先读取插件清单，之后才能导入插件运行时代码。
`defineToolPlugin` 会为此公开静态元数据，而
`openclaw plugins build` 会将其写入软件包。更改插件 ID、名称、描述、配置架构、激活设置或工具
名称后，请重新运行生成器：

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

单工具插件生成的清单：

```json
{
  "id": "stock-quotes",
  "name": "Stock Quotes",
  "description": "Fetch stock quote snapshots.",
  "version": "0.1.0",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  },
  "activation": {
    "onStartup": true
  },
  "contracts": {
    "tools": ["stock_quote"]
  }
}
```

`contracts.tools` 是重要的发现契约：它告诉 OpenClaw 每个工具
归哪个插件所有，而无需加载所有已安装插件的运行时。过期的清单
可能导致工具无法被发现，或将注册
错误错误地归咎于其他插件。

## 软件包元数据

`openclaw plugins build` 还会将 `package.json` 与所选运行时
入口对齐：

```json
{
  "type": "module",
  "files": ["dist", "openclaw.plugin.json", "README.md"],
  "dependencies": {
    "typebox": "^1.1.38"
  },
  "peerDependencies": {
    "openclaw": ">=2026.5.17"
  },
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

请发布构建后的 JavaScript（`./dist/index.js`），而不是 TypeScript 源代码入口。
源代码入口仅适用于工作区本地开发。

## 在 CI 中验证

当生成的元数据过期时，`plugins build --check` 会在不重写文件的情况下失败：

```bash
npm run build
openclaw plugins build --entry ./dist/index.js --check
openclaw plugins validate --entry ./dist/index.js
npm test
```

OpenClaw SDK 兼容性字段带有 TypeScript `@deprecated` 注解，
编辑器会将其显示为迁移警告。若要在 CI 中强制检查，请启用
类型感知规则，例如
[`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/)。
Oxlint 不具备类型感知能力，因此无法强制检查这些注解。因此，生成的
`plugins init` 脚手架不会添加弃用 lint 配置。

`plugins validate` 会检查：

- `openclaw.plugin.json` 存在并通过常规清单加载器。
- 当前入口导出 `defineToolPlugin` 元数据。
- 生成的清单字段与入口元数据匹配。
- `contracts.tools` 与声明的工具名称匹配。
- `package.json` 将 `openclaw.extensions` 指向所选的运行时入口。

## 在本地安装并检查

从另一个 OpenClaw 检出目录或已安装的 CLI 安装该软件包路径：

```bash
openclaw plugins install ./stock-quotes
openclaw plugins inspect stock-quotes --runtime
```

如需执行打包后的冒烟测试，请先打包，然后安装 tarball：

```bash
npm pack
openclaw plugins install npm-pack:./openclaw-plugin-stock-quotes-0.1.0.tgz
openclaw plugins inspect stock-quotes --runtime --json
```

安装后，重启或重新加载 Gateway 网关，并让智能体使用该
工具。如果工具不可见，请先检查插件运行时和生效的
工具目录，再更改代码（请参阅[故障排查](#troubleshooting)）。

## 发布

软件包准备就绪后，通过 ClawHub 发布。`clawhub package publish`
接受一个来源：本地文件夹、GitHub 仓库（`owner/repo[@ref]`）或
tarball URL。

```bash
clawhub package publish ./stock-quotes --dry-run
clawhub package publish ./stock-quotes
```

使用明确的 ClawHub 定位符安装：

```bash
openclaw plugins install clawhub:your-org/stock-quotes
```

在启动切换期间，不带前缀的 npm 软件包规范仍会从 npm 安装，但
ClawHub 是 OpenClaw 插件首选的发现和分发平台。有关所有者范围和
发布审查，请参阅 [ClawHub 发布](/zh-CN/clawhub/publishing)。

## 故障排查

### `plugin entry not found: ./dist/index.js`

所选入口文件不存在。运行 `npm run build`，然后重新运行
`openclaw plugins build --entry ./dist/index.js` 或
`openclaw plugins validate --entry ./dist/index.js`。

### `plugin entry does not expose defineToolPlugin metadata`

该入口未导出由 `defineToolPlugin` 创建的值。请确认
模块的默认导出是 `defineToolPlugin(...)` 的结果，或通过
`--entry` 传入正确的入口。

### `openclaw.plugin.json generated metadata is stale`

清单不再与入口元数据匹配。运行：

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

同时提交 `openclaw.plugin.json` 和 `package.json` 的更改。

### `package.json openclaw.extensions must include ./dist/index.js`

软件包元数据指向了另一个运行时入口。运行
`openclaw plugins build --entry ./dist/index.js`，使生成器将
软件包元数据与要发布的入口对齐。

### `Cannot find package 'typebox'`

构建后的插件在运行时导入 `typebox`。将其保留在 `dependencies` 中，
重新安装、重新构建并再次运行验证。

### 安装后工具未出现

按以下顺序检查：

1. `openclaw plugins inspect <plugin-id> --runtime`
2. `openclaw plugins validate --root <plugin-root> --entry ./dist/index.js`
3. `openclaw.plugin.json` 包含具有预期工具名称的 `contracts.tools`。
4. `package.json` 包含 `openclaw.extensions: ["./dist/index.js"]`。
5. 安装插件后已重启或重新加载 Gateway 网关。

## 另请参阅

- [Building Plugins](/zh-CN/plugins/building-plugins)
- [插件入口点](/zh-CN/plugins/sdk-entrypoints)
- [插件 SDK 子路径](/zh-CN/plugins/sdk-subpaths)
- [插件清单](/zh-CN/plugins/manifest)
- [插件 CLI](/zh-CN/cli/plugins)
- [ClawHub 发布](/zh-CN/clawhub/publishing)
