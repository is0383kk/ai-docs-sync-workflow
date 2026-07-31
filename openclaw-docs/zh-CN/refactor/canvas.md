---
read_when:
    - 迁移 Canvas 主机、工具、命令、文档或协议的所有权
    - 审核 Canvas 是否仍归核心所有
    - 准备或审查实验性 Canvas 插件 PR
summary: 将 Canvas 从核心移出并迁移到内置实验性插件的计划和审核清单。
title: Canvas 插件重构
x-i18n:
    generated_at: "2026-07-26T06:00:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ead3f865ea80acb1e47f45a5ab07acf19a6470035c00c81006b2b1230bedd71e
    source_path: refactor/canvas.md
    workflow: 16
---

# Canvas 插件重构

Canvas 使用率较低，且仍处于实验阶段。应将其视为内置插件，而非核心功能。核心可以保留通用的 Gateway 网关、节点、HTTP、身份验证、配置和原生客户端管道，但 Canvas 特有的行为应位于 `extensions/canvas` 下。

## 目标

将 Canvas 的所有权移至 `extensions/canvas`，同时保留当前的已配对节点行为：

- 面向智能体的 `canvas` 工具由 Canvas 插件注册
- 仅当 Canvas 插件注册 Canvas 节点命令时，才允许使用这些命令
- A2UI 主机/源文件位于 Canvas 插件下
- Canvas 文档具体化逻辑位于 Canvas 插件下
- CLI 命令实现位于 Canvas 插件下，或通过插件所有的运行时桶形导出委托
- 文档和插件清单将 Canvas 描述为实验性且由插件支持

## 非目标

- 本次重构不重新设计原生应用的 Canvas UI。
- 除非另有产品决策要求删除 Canvas，否则不要移除 iOS、Android 或 macOS 中的 Canvas 协议/客户端支持。
- 除非至少还有一个其他内置插件需要相同的衔接层，否则不要仅为 Canvas 构建通用的插件服务框架。

## 当前分支状态

已完成：

- 在 `extensions/canvas` 中添加了内置插件包。
- 添加了 `extensions/canvas/openclaw.plugin.json`。
- 将智能体的 `canvas` 工具从 `src/agents/tools/canvas-tool.ts` 移至 `extensions/canvas/src/tool.ts`。
- 从 `src/agents/openclaw-tools.ts` 中移除了核心对 `createCanvasTool` 的注册。
- 将 Canvas 主机实现从 `src/canvas-host` 移至 `extensions/canvas/src/host`。
- 保留 `extensions/canvas/runtime-api.ts`，作为插件所有的兼容性桶形导出，供测试、打包和外部公共 Canvas 辅助程序使用。
- 将 Canvas 文档具体化逻辑从 `src/gateway/canvas-documents.ts` 移至 `extensions/canvas/src/documents.ts`。
- 将 Canvas CLI 实现和 A2UI JSONL 辅助程序移至 `extensions/canvas/src/cli.ts`。
- 将 Canvas 主机 URL 和限定作用域的能力辅助程序移至 `extensions/canvas/src`。
- 将 Canvas 节点命令默认值从硬编码的核心列表移至插件 `nodeInvokePolicies`。
- 在 `plugins.entries.canvas.config.host` 中添加了插件所有的 Canvas 主机配置。
- 将 Canvas 和 A2UI HTTP 服务移至 Canvas 插件 HTTP 路由注册之后。
- 为插件所有的 HTTP 路由添加了通用插件 WebSocket 升级分派。
- 使用通用的托管插件表面和节点能力辅助程序，取代了 Canvas 特有的 Gateway 网关主机 URL 和节点能力身份验证。
- 添加了插件所有的托管媒体解析器，使 Canvas 文档 URL 通过 Canvas 插件解析，而无需核心导入 Canvas 文档内部实现。
- 添加了 `api.registerNodeCliFeature(...)`，使 Canvas 能够将 `openclaw nodes canvas` 声明为插件所有的节点功能，而无需手动写出父命令路径。
- 移除了生产环境中 `src/**` 对 `extensions/canvas/runtime-api.js` 的导入。
- 将 A2UI 捆绑包源代码从 `apps/shared/OpenClawKit/Tools/CanvasA2UI` 移至 `extensions/canvas/src/host/a2ui-app`。
- 将 A2UI 构建/复制实现移至 `extensions/canvas/scripts` 下，并用通用的内置插件资源钩子取代了根级构建接线。
- 移除了运行时旧版顶层 `canvasHost` 配置别名。
- 保留了 Canvas Doctor 迁移，使 `openclaw doctor --fix` 能够将旧的 `canvasHost` 配置重写为 `plugins.entries.canvas.config.host`。
- 移除了 Gateway 网关协议 v4 之后面向旧版智能体的 Canvas 协议兼容性。原生客户端和 Gateway 网关现在仅使用 `pluginSurfaceUrls.canvas` 和 `node.pluginSurface.refresh`；本次实验性重构有意不支持已弃用的 `canvasHostUrl`、`canvasCapability` 和 `node.canvas.capability.refresh` 路径。
- 更新了生成的插件清单以包含 Canvas。
- 在 `docs/plugins/reference/canvas.md` 添加了插件参考文档。

已知仍由核心所有的 Canvas 表面：

- `apps/` 下的原生应用 Canvas 处理程序仍有意使用 Canvas 插件表面
- `apps/` 下的原生应用 Canvas 协议/客户端处理程序
- 为保持运行时查找向后兼容，已发布工件的输出仍使用 `dist/canvas-host/a2ui`，但复制步骤现在由插件所有

## 目标形态

`extensions/canvas` 应负责：

- 插件清单和包元数据
- 智能体工具注册
- 节点调用命令策略
- Canvas 主机和 A2UI 运行时
- Canvas A2UI 捆绑包源代码和资源构建/复制脚本
- Canvas 文档创建和资源解析
- Canvas CLI 实现
- Canvas 文档页面和插件清单条目

核心应仅负责通用衔接层：

- 插件发现和注册
- 通用智能体工具注册表
- 通用节点调用策略注册表
- 通用 Gateway 网关 HTTP/身份验证和 WebSocket 升级分派
- 通用托管插件表面 URL 解析
- 通用托管媒体解析器注册
- 通用节点能力传输
- 通用配置管道
- 通用内置插件资源钩子发现

原生应用可以保留 Canvas 命令处理程序作为协议客户端。它们不是插件运行时的所有者。

## 迁移步骤

1. 将 `plugins.entries.canvas.config.host` 视为插件所有的配置表面。
2. 更新文档，将 Canvas 描述为实验性内置插件。
3. 运行聚焦于 Canvas 的测试、插件清单检查、插件 SDK API 检查，以及受运行时边界影响的构建/类型检查。

## 审计检查清单

在宣布重构完成之前：

- `rg "src/canvas-host|../canvas-host"` 不返回任何实时源代码导入。
- `rg "canvas-tool|createCanvasTool" src` 找不到核心所有的 Canvas 工具实现。
- `rg "canvas.present|canvas.snapshot|canvas.a2ui" src/gateway` 在通用插件策略测试之外找不到硬编码的允许列表默认值。
- `rg "extensions/canvas/runtime-api" src --glob '!**/*.test.ts'` 为空。
- `rg "canvas-documents" src` 为空。
- `rg "registerNodesCanvasCommands|nodes-canvas" src` 为空；Canvas 插件通过嵌套插件 CLI 元数据注册 `openclaw nodes canvas`。
- `rg "createCanvasHostHandler|handleA2uiHttpRequest" src/gateway` 不返回任何 Gateway 网关运行时所有权。
- `rg "apps/shared/OpenClawKit/Tools/CanvasA2UI|canvas-a2ui-copy|extensions/canvas/src/host/a2ui" scripts .github package.json` 仅找到兼容性包装器或插件所有的路径。
- `pnpm plugins:inventory:check` 通过。
- `pnpm plugin-sdk:api:check` 通过，或者生成的 API 契约记录已被有意更新并经过审查。
- 针对性的 Canvas 测试通过。
- Canvas 主机/A2UI 路径的变更通道测试通过。
- PR 正文明确说明 Canvas 是实验性的，并由插件支持。

## 验证命令

迭代期间使用针对性的本地检查：

```sh
pnpm test extensions/canvas/src/host/server.test.ts extensions/canvas/src/host/server.state-dir.test.ts extensions/canvas/src/host/file-resolver.test.ts
pnpm test src/gateway/server.plugin-node-capability-auth.test.ts src/gateway/server-import-boundary.test.ts
pnpm test extensions/canvas/src/config-migration.test.ts src/commands/doctor-legacy-config.migrations.test.ts
pnpm test test/scripts/changed-lanes.test.ts test/scripts/build-all.test.ts extensions/canvas/scripts/bundle-a2ui.test.ts test/scripts/bundled-plugin-assets.test.ts extensions/canvas/scripts/copy-a2ui.test.ts src/infra/run-node.test.ts
pnpm tsgo:extensions
pnpm plugins:inventory:check
pnpm plugin-sdk:api:check
```

如果运行时桶形导出、延迟导入、打包或已发布插件表面发生变化，请在推送前运行 `pnpm build`。
