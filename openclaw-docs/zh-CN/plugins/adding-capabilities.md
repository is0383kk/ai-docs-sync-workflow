---
read_when:
    - 添加新的核心能力和插件注册接口
    - 判断代码应归属于核心、供应商插件还是功能插件
    - 为渠道或工具接入新的运行时辅助函数
sidebarTitle: Adding capabilities
summary: 向 OpenClaw 插件系统添加新的共享能力的贡献者指南
title: 添加能力（贡献者指南）
x-i18n:
    generated_at: "2026-07-26T06:19:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 14f86c98eb10c6e92970d1b65009ac7bb103afcb6bc57bad2c39e59bc038c961
    source_path: plugins/adding-capabilities.md
    workflow: 16
---

<Info>
  这是面向 OpenClaw 核心开发者的**贡献者指南**。如果你正在
  构建外部插件，请改为参阅[构建插件](/zh-CN/plugins/building-plugins)。
  有关深入的架构参考（能力模型、所有权、加载流水线、运行时辅助函数），请参阅[插件内部机制](/zh-CN/plugins/architecture)。
</Info>

当 OpenClaw 需要嵌入、图像生成、视频生成或未来某种由供应商支持的新共享领域时，请采用此方法。

规则：

- **插件** = 所有权边界
- **能力** = 共享核心契约

不要将供应商直接接入渠道或工具。应先定义能力。

## 何时创建能力

仅当以下条件**全部**满足时，才创建新能力：

1. 可能有多个供应商能够实现它。
2. 渠道、工具或功能插件应当无需关注供应商即可使用它。
3. 核心需要负责回退、策略、配置或交付行为。

如果相关工作仅适用于某个供应商，并且尚不存在共享契约，请先定义契约。

## 标准流程

1. 定义类型化的核心契约。
2. 为该契约添加插件注册机制。
3. 添加共享运行时辅助函数。
4. 接入一个真实的供应商插件作为验证。
5. 将功能/渠道使用方迁移到运行时辅助函数。
6. 添加契约测试。
7. 记录面向操作员的配置和所有权模型。

## 各层职责

| 层                         | 负责                                                                                                                                                                                                                                  |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **核心**                   | 请求/响应类型；提供商注册表和解析；回退行为；在嵌套对象、通配符、数组项和组合节点上传播了 `title`/`description` 文档元数据的配置 schema；运行时辅助函数接口。 |
| **供应商插件**             | 供应商 API 调用、供应商身份验证处理、供应商特定的请求规范化，以及注册能力实现。                                                                                                     |
| **功能/渠道插件**          | 调用 `api.runtime.*` 或对应的 `plugin-sdk/*-runtime` 辅助函数。绝不直接调用供应商实现。                                                                                                                    |

## 提供商和 harness 接缝

当行为属于模型提供商契约而非通用 Agent loop 时，使用**提供商钩子**。示例包括选择传输方式后的提供商特定请求参数、身份验证配置文件偏好、提示词叠加，以及模型/配置文件故障转移后的后续回退路由。

当行为属于执行某一轮次的运行时时，使用 **agent harness 钩子**。Harness 可以对明确的协议结果进行分类，例如空输出、只有推理而没有可见输出，或只有结构化计划而没有最终答案，以便外层模型回退策略决定是否重试。

保持这两个接缝精简：

- 核心负责重试/回退策略。
- 提供商插件负责提供商特定的请求、身份验证和路由提示。
- Harness 插件负责运行时特定的尝试分类。
- 第三方插件返回提示，而不直接修改核心状态。

## 文件检查清单

对于一项新能力，通常需要修改以下区域：

- `src/<capability>/types.ts`
- `src/<capability>/...registry/runtime.ts`
- `src/plugins/types.ts`
- `src/plugins/registry.ts`
- `src/plugins/captured-registration.ts`
- `src/plugins/contracts/registry.ts`
- `src/plugins/runtime/types-core.ts`
- `src/plugins/runtime/index.ts`
- `src/plugin-sdk/<capability>.ts`
- `src/plugin-sdk/<capability>-runtime.ts`
- 一个或多个内置插件包。
- 配置、文档和测试。

## 完整示例：图像生成

图像生成遵循标准结构：

1. 核心定义 `ImageGenerationProvider`。
2. 核心公开 `registerImageGenerationProvider(...)`。
3. 核心公开 `api.runtime.imageGeneration.generate(...)` 和 `.listProviders(...)`。
4. 供应商插件（`comfy`、`deepinfra`、`fal`、`google`、`litellm`、`microsoft-foundry`、`minimax`、`openai`、`openrouter`、`vydra`、`xai`）注册由供应商支持的实现。
5. 未来的供应商可以注册同一契约，而无需更改渠道/工具。

该配置键有意与视觉分析路由分开：

- `agents.defaults.imageModel` 用于分析图像。
- `agents.defaults.mediaModels.image` 用于生成图像。

应将二者分开，以确保回退和策略保持明确。

## 嵌入提供商

对于可复用的向量嵌入提供商，请使用 `registerEmbeddingProvider(...)` / 契约 `embeddingProviders`。
此契约的适用范围有意设计得比记忆更广：
工具、搜索、检索、导入器或未来的功能插件
都可以使用嵌入，而无需依赖记忆引擎。记忆搜索
也使用通用的 `embeddingProviders`。

旧版记忆专用注册 API 和 `memoryEmbeddingProviders`
契约已弃用。所有新的嵌入提供商都应使用 `registerEmbeddingProvider`
和 `embeddingProviders`。

## 审查清单

发布新能力之前，请验证：

- 没有渠道/工具直接导入供应商代码。
- 运行时辅助函数是共享路径。
- 至少有一项契约测试对内置所有权作出断言。
- 配置文档注明了新的模型/配置键。
- 插件文档解释了所有权边界。

如果某个 PR 跳过能力层，并将供应商行为硬编码到渠道/工具中，请退回该 PR，并要求先定义契约。

## 相关内容

- [插件内部机制](/zh-CN/plugins/architecture) — 能力模型、所有权、加载流水线和运行时辅助函数。
- [构建插件](/zh-CN/plugins/building-plugins) — 首个插件教程。
- [插件 SDK 概览](/zh-CN/plugins/sdk-overview) — 导入映射和注册 API 参考。
- [创建技能](/zh-CN/tools/creating-skills) — 配套的贡献者接口。
