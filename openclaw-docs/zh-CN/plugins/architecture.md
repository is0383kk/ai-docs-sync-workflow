---
read_when:
    - 构建或调试原生 OpenClaw 插件
    - 理解插件能力模型或所有权边界
    - 处理插件加载管线或注册表相关工作
    - 实现提供商运行时钩子或渠道插件
sidebarTitle: Internals
summary: 插件内部机制：能力模型、所有权、契约、加载流水线和运行时辅助函数
title: 插件内部机制
x-i18n:
    generated_at: "2026-07-26T05:53:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d47551b1bc2f71ce2ade3dfdd14bff8ee187616c3807f8101c1a3236e1443cc1
    source_path: plugins/architecture.md
    workflow: 16
---

这是 OpenClaw 插件系统的**深度架构参考**。如需实用指南，请从以下专题页面之一开始。

<CardGroup cols={2}>
  <Card title="安装和使用插件" icon="plug" href="/zh-CN/tools/plugin">
    面向最终用户的插件添加、启用和故障排除指南。
  </Card>
  <Card title="Building Plugins" icon="rocket" href="/zh-CN/plugins/building-plugins">
    首个插件教程，包含最小可用的插件清单。
  </Card>
  <Card title="渠道插件" icon="comments" href="/zh-CN/plugins/sdk-channel-plugins">
    构建消息渠道插件。
  </Card>
  <Card title="提供商插件" icon="microchip" href="/zh-CN/plugins/sdk-provider-plugins">
    构建模型提供商插件。
  </Card>
  <Card title="SDK 概览" icon="book" href="/zh-CN/plugins/sdk-overview">
    导入映射和注册 API 参考。
  </Card>
</CardGroup>

## 公共能力模型

能力是 OpenClaw 内部公开的**原生插件**模型。每个原生 OpenClaw 插件都会针对一种或多种能力类型进行注册：

| 能力                   | 注册方法                                         | 示例插件                                                    |
| ---------------------- | ------------------------------------------------ | ----------------------------------------------------------- |
| 文本推理               | `api.registerProvider(...)`                      | `anthropic`, `openai`                                       |
| CLI 推理后端           | `api.registerCliBackend(...)`                    | `anthropic`, `openai`                                       |
| 嵌入                   | `api.registerEmbeddingProvider(...)`             | 提供商自有的向量插件                                        |
| 语音                   | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`                                   |
| 实时转录               | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                                                    |
| 实时语音               | `api.registerRealtimeVoiceProvider(...)`         | `google`, `openai`                                          |
| 媒体理解               | `api.registerMediaUnderstandingProvider(...)`    | `google`, `openai`                                          |
| 转录来源               | `api.registerTranscriptSourceProvider(...)`      | `discord`, `google-meet`, `teams-meetings`, `zoom-meetings` |
| 图像生成               | `api.registerImageGenerationProvider(...)`       | `fal`, `google`, `openai`                                   |
| 音乐生成               | `api.registerMusicGenerationProvider(...)`       | `fal`, `google`, `minimax`                                  |
| 视频生成               | `api.registerVideoGenerationProvider(...)`       | `fal`, `google`, `qwen`                                     |
| Web 获取               | `api.registerWebFetchProvider(...)`              | `firecrawl`                                                 |
| Web 搜索               | `api.registerWebSearchProvider(...)`             | `brave`, `firecrawl`, `google`                              |
| 渠道/消息传递          | `api.registerChannel(...)`                       | `matrix`, `msteams`                                         |
| Gateway 网关设备发现   | `api.registerGatewayDiscoveryService(...)`       | `bonjour`                                                   |

<Note>
注册零项能力但提供钩子、工具、设备发现服务或后台服务的插件属于**旧版纯钩子**插件。此模式仍受到全面支持。
</Note>

### 外部兼容性立场

能力模型已经落地核心，并已用于当前的内置/原生插件，但外部插件兼容性仍需遵循比“既然已导出，就代表已冻结”更严格的标准。

| 插件情况                                         | 指导原则                                                                                         |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| 现有外部插件                                     | 保持基于钩子的集成正常工作；这是兼容性基线。                                                     |
| 新的内置/原生插件                                | 优先采用显式能力注册，而非特定于供应商的内部访问方式或新的纯钩子设计。                           |
| 采用能力注册的外部插件                           | 允许采用，但除非文档将特定能力的辅助接口标记为稳定，否则应视其为仍在演进。                       |

能力注册是预期的发展方向。在过渡期间，对于外部插件，旧版钩子仍是避免破坏兼容性的最安全路径。各个已导出的辅助子路径并非同等稳定——应优先使用范围明确且有文档说明的契约，而非偶然导出的辅助接口。

### 插件形态

OpenClaw 根据每个已加载插件的实际注册行为（而不仅仅是静态元数据）将其归类为一种形态：

<AccordionGroup>
  <Accordion title="plain-capability">
    仅注册一种能力类型（例如像 `arcee` 或 `chutes` 这样仅提供商的插件）。
  </Accordion>
  <Accordion title="hybrid-capability">
    注册多种能力类型（例如 `openai` 拥有文本推理、语音、媒体理解和图像生成能力）。
  </Accordion>
  <Accordion title="hook-only">
    仅注册钩子（类型化或自定义），不注册能力、工具、命令或服务。
  </Accordion>
  <Accordion title="non-capability">
    注册工具、命令、服务或路由，但不注册能力。
  </Accordion>
</AccordionGroup>

使用 `openclaw plugins inspect <id>` 可查看插件的形态和能力明细。有关详情，请参阅 [CLI 参考](/zh-CN/cli/plugins#inspect)。

### 兼容性信号

`openclaw doctor`、`openclaw plugins inspect <id>`、`openclaw status --all` 和 `openclaw plugins doctor` 会显示以下兼容性通知：

| 信号                                         | 含义                                                                                                       |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **配置有效**                                 | 配置解析正常，且插件可解析                                                                                 |
| **纯钩子**（信息）                           | 插件仅注册钩子；这是受支持的路径，但尚未迁移到能力注册                                                     |
| **已弃用的记忆嵌入 API**（警告）             | 非内置插件使用旧的记忆专用嵌入提供商 API，而不是 `registerEmbeddingProvider` |
| **严重错误**                                 | 配置无效或插件加载失败                                                                                     |

这些提示/警告信号目前都不会导致你的插件无法运行。这些信号也会出现在 `openclaw status --all` 和 `openclaw plugins doctor` 中。

## 架构概览

OpenClaw 的插件系统包含四层：

<Steps>
  <Step title="插件清单 + 设备发现">
    OpenClaw 从已配置路径、工作区根目录、全局插件根目录和内置插件中查找候选插件。设备发现会首先读取原生 `openclaw.plugin.json` 插件清单以及受支持的包清单。
  </Step>
  <Step title="启用 + 验证">
    核心决定已发现的插件是启用、禁用、阻止，还是被选中用于记忆等独占插槽。
  </Step>
  <Step title="运行时加载">
    原生 OpenClaw 插件在进程内加载，并将能力注册到中央注册表中。打包的 JavaScript 通过原生 `require` 加载；第三方本地源代码 TypeScript 则使用应急的 Jiti 回退方案。兼容的软件包会被规范化为注册表记录，而不会导入运行时代码。
  </Step>
  <Step title="接口消费">
    OpenClaw 的其余部分读取注册表，以公开工具、渠道、提供商设置、钩子、HTTP 路由、CLI 命令和服务。
  </Step>
</Steps>

对于插件 CLI，根命令设备发现具体分为两个阶段：

- 解析时元数据来自 `registerCli(..., { descriptors: [...] })`
- 实际的插件 CLI 模块可保持延迟加载，并在首次调用时注册

这样既能将插件自有的 CLI 代码保留在插件内部，又能让 OpenClaw 在解析前预留根命令名称。

重要的设计边界：

- 插件清单/配置验证应基于**插件清单/架构元数据**运行，而无需执行插件代码
- 原生能力设备发现可以加载受信任的插件入口代码，以构建不激活插件的注册表快照
- 原生运行时行为来自插件模块通过 `api.registrationMode === "full"` 加载的 `register(api)` 路径

这种拆分让 OpenClaw 可以在完整运行时激活之前验证配置、解释插件缺失/禁用的原因，并构建 UI/架构提示。

### 插件元数据快照和查找表

Gateway 网关启动时会为当前配置快照构建一个 `PluginMetadataSnapshot`。该快照仅包含元数据：它存储已安装插件索引、插件清单注册表、插件清单诊断信息、所有者映射、插件 ID 规范化器以及插件清单记录。它不保存已加载的插件模块、提供商 SDK、软件包内容或运行时导出。

插件感知的配置验证、启动时自动启用和 Gateway 网关插件引导会使用该快照，而不是各自重新构建插件清单/索引元数据。`PluginLookUpTable` 派生自同一快照，并为当前运行时配置添加启动插件计划。

启动后，Gateway 网关会将当前元数据快照保留为可替换的运行时产物。重复执行的运行时提供商设备发现可以借用该快照，而不必在每次提供商目录遍历时重新构建已安装索引和插件清单注册表。Gateway 网关关闭、配置/插件清单发生变化以及写入已安装索引时，会清除或替换该快照；如果不存在兼容的当前快照，调用方将回退到冷启动的插件清单/索引路径。兼容性检查必须包含 `plugins.load.paths` 和默认 Agent 工作区等插件设备发现根目录，因为工作区插件属于元数据范围的一部分。

快照和查找表使重复的启动决策保持在快速路径上：

- 渠道所有权
- 延迟渠道启动
- 启动插件 ID
- 提供商和 CLI 后端所有权
- 设置提供商、命令别名、模型目录提供商和插件清单契约所有权
- 插件配置架构和渠道配置架构验证
- 启动时自动启用决策

安全边界是替换快照，而非修改快照。当配置、插件清单、安装记录或持久化索引策略发生变化时，应重新构建快照。不要将其视为广泛的可变全局注册表，也不要保留无限量的历史快照。运行时插件加载与元数据快照保持分离，因此过时的运行时状态无法隐藏在元数据缓存之后。

缓存规则记录在[插件架构内部机制](/zh-CN/plugins/architecture-internals#plugin-cache-boundary)中：除非调用方持有当前流程的显式快照、查找表或插件清单注册表，否则插件清单和设备发现元数据始终保持最新。隐藏的元数据缓存和基于实际时间的 TTL 不属于插件加载机制。只有在代码或已安装工件实际加载后，运行时加载器、模块和依赖工件缓存才可以持续存在。

一些冷路径调用方仍会直接从持久化的已安装插件索引重建清单注册表，而不是接收 Gateway 网关 `PluginLookUpTable`。该路径现在会按需重建注册表；如果调用方已有当前查找表或显式清单注册表，优先在运行时流程中传递它。

### 激活规划

激活规划属于控制平面的一部分。调用方可以在加载更广泛的运行时注册表之前，查询哪些插件与具体命令、提供商、渠道、路由、Agent harness 或能力相关。

规划器保持与当前清单行为兼容：

- `activation.*` 字段是显式规划器提示
- `providers`、`channels`、`commandAliases`、`setup.providers`、`contracts.tools` 和钩子仍作为清单所有权回退依据
- 仅返回 ID 的规划器 API 仍可供现有调用方使用
- 规划 API 会报告原因标签，以便诊断区分显式提示与所有权回退

<Warning>
不要将 `activation` 视为生命周期钩子或 `register(...)` 的替代项。它是用于缩小加载范围的元数据。当所有权字段已能描述关系时，优先使用所有权字段；仅将 `activation` 用于额外的规划器提示。
</Warning>

### 渠道插件和共享消息工具

对于常规聊天操作，渠道插件无需单独注册发送、编辑或表情回应工具。OpenClaw 在核心中保留一个共享的 `message` 工具，渠道插件负责其背后的渠道特定发现和执行。

当前边界如下：

- 核心负责共享的 `message` 工具宿主、提示词接线、会话/线程记录及执行分派
- 渠道插件负责限定作用域的操作发现、能力发现以及任何渠道特定的 schema 片段
- 渠道插件负责提供商特定的会话对话语法，例如对话 ID 如何编码线程 ID 或从父对话继承
- 渠道插件通过其操作适配器执行最终操作

对于渠道插件，SDK 接口是 `ChannelMessageActionAdapter.describeMessageTool(...)`。该统一发现调用让插件能够同时返回其可见操作、能力和 schema 贡献，从而避免这些部分彼此偏离。

消息操作名称采用刻意封闭且由核心管理的词汇表，以便每种传输协议都能呈现每项操作。插件通过核心 PR 添加操作名称；有意不支持运行时注册。

当渠道特定的消息工具参数携带媒体来源（例如本地路径或远程媒体 URL）时，插件还应通过 `describeMessageTool(...)` 返回 `mediaSourceParams`。核心使用该显式列表应用沙箱路径规范化和出站媒体访问提示，而无需硬编码插件所有的参数名称。此处优先使用按操作划分的映射，而不是覆盖整个渠道的扁平列表，以免仅用于个人资料的媒体参数在 `send` 等无关操作中被规范化。

核心会将运行时作用域传入该发现步骤。重要字段包括：

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- 受信任的入站 `requesterSenderId`

这对上下文敏感的插件很重要。渠道可以根据活动账户、当前房间/线程/消息或受信任请求者身份隐藏或公开消息操作，而无需在核心 `message` 工具中硬编码渠道特定分支。

因此，嵌入式运行器的路由变更仍属于插件工作：运行器负责将当前聊天/会话身份转发到插件发现边界，使共享的 `message` 工具能为当前轮次公开正确的渠道所有接口。

对于渠道所有的执行辅助程序，渠道插件应将执行运行时保留在各自的插件模块中。核心不再管理 `src/agents/tools` 下 Discord、Slack、Telegram 或 WhatsApp 的消息操作运行时。我们不会发布单独的 `plugin-sdk/*-action-runtime` 子路径，这些插件应直接从其插件所有的模块导入本地运行时代码。

同一边界通常也适用于以提供商命名的 SDK 接口：核心不应导入 Discord、Signal、Slack、WhatsApp 或类似插件的渠道特定便利 barrel。如果核心需要某项行为，应使用内置插件自身的 `api.ts` / `runtime-api.ts` barrel，或将该需求提升为共享 SDK 中狭窄的通用能力。

内置插件遵循相同规则。内置插件的 `runtime-api.ts` 不应重新导出其自身品牌化的 `openclaw/plugin-sdk/<plugin-id>` facade。这些品牌化 facade 仍作为外部插件和旧版使用方的兼容性 shim 保留，但内置插件应使用本地导出，以及 `openclaw/plugin-sdk/channel-policy`、`openclaw/plugin-sdk/runtime-store` 或 `openclaw/plugin-sdk/webhook-ingress` 等狭窄的通用 SDK 子路径。除非现有外部生态系统的兼容性边界有此要求，否则新代码不应添加插件 ID 特定的 SDK facade。

具体到投票，有两条执行路径：

- `outbound.sendPoll` 是适用于符合通用投票模型的渠道的共享基线
- `actions.handleAction("poll")` 是适用于渠道特定投票语义或额外投票参数的首选路径

核心现在会将共享投票解析推迟到插件投票分派拒绝该操作之后，因此插件所有的投票处理程序可以接受渠道特定的投票字段，而不会先被通用投票解析器阻止。

如需了解完整启动序列，请参阅[插件架构内部机制](/zh-CN/plugins/architecture-internals)。

## 能力所有权模型

OpenClaw 将原生插件视为某个**公司**或某项**功能**的所有权边界，而不是互不相关集成的杂物集合。

这意味着：

- 公司插件通常应管理该公司面向 OpenClaw 的所有接口
- 功能插件通常应管理其引入的完整功能接口
- 渠道应使用共享核心能力，而不是临时重新实现提供商行为

<AccordionGroup>
  <Accordion title="多能力供应商">
    `google` 负责文本推理、CLI 后端、嵌入、语音、实时语音、媒体理解、图像/音乐/视频生成和 Web 搜索。`openai` 负责文本推理、嵌入、语音、实时转录、实时语音、媒体理解以及图像/视频生成。`minimax` 负责文本推理，以及媒体理解、语音、图像/音乐/视频生成和 Web 搜索。
  </Accordion>
  <Accordion title="单能力供应商">
    `arcee` 和 `chutes` 仅负责文本推理；`microsoft` 仅负责语音。在需要覆盖该供应商更多接口之前，供应商插件可以一直保持如此狭窄的范围。
  </Accordion>
  <Accordion title="功能插件">
    `voice-call` 负责通话传输、工具、CLI、路由和 Twilio 媒体流桥接，但会使用共享语音、实时转录和实时语音能力，而不是直接导入供应商插件。
  </Accordion>
</AccordionGroup>

预期的最终状态是：

- 即使某个供应商面向 OpenClaw 的接口横跨文本模型、语音、图像和视频，也应位于同一个插件中
- 其他供应商也可以针对各自的接口范围采用相同做法
- 渠道无需关心哪个供应商插件负责该提供商；它们使用核心公开的共享能力契约

关键区别如下：

- **插件** = 所有权边界
- **能力** = 多个插件可以实现或使用的核心契约

因此，如果 OpenClaw 添加视频等新领域，首要问题不是“哪个提供商应该硬编码视频处理？”首要问题是“核心视频能力契约是什么？”该契约建立后，供应商插件即可针对它进行注册，渠道/功能插件也可以使用它。

如果该能力尚不存在，正确做法通常是：

<Steps>
  <Step title="定义能力">
    在核心中定义缺失的能力。
  </Step>
  <Step title="通过 SDK 公开">
    以类型化方式通过插件 API/运行时公开该能力。
  </Step>
  <Step title="接入使用方">
    将渠道/功能接入该能力。
  </Step>
  <Step title="供应商实现">
    让供应商插件注册实现。
  </Step>
</Steps>

这样既能保持所有权明确，又能避免核心行为依赖单一供应商或一次性的插件特定代码路径。

### 能力分层

在决定代码归属时，使用以下思维模型：

<Tabs>
  <Tab title="核心能力层">
    共享编排、策略、回退、配置合并规则、交付语义和类型化契约。
  </Tab>
  <Tab title="供应商插件层">
    供应商特定的 API、身份验证、模型目录、语音合成、图像生成、视频后端和用量端点。
  </Tab>
  <Tab title="渠道/功能插件层">
    使用核心能力并将其呈现在某一界面上的 Discord/Slack/语音通话等集成。
  </Tab>
</Tabs>

例如，TTS 遵循以下结构：

- 核心负责回复时 TTS 策略、回退顺序、偏好设置和渠道交付
- `elevenlabs`、`google`、`microsoft` 和 `openai` 负责合成实现
- `voice-call` 使用电话 TTS 运行时辅助程序

未来的能力也应优先采用相同模式。

### 多能力公司插件示例

从外部看，公司插件应具有内聚性。如果 OpenClaw 为模型、语音、实时转录、实时语音、媒体理解、图像生成、视频生成、Web 获取和 Web 搜索提供共享契约，供应商可以在一个位置管理其所有接口：

```ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { exampleAiMedia } from "./exampleai-media.js";

export default definePluginEntry({
  id: "exampleai",
  name: "ExampleAI",
  description: "ExampleAI 模型和媒体能力。",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // 身份验证/模型目录/运行时钩子
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // 供应商语音配置 — 直接实现 SpeechProviderPlugin 接口
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      describeImage: (req) => exampleAiMedia.describeImage(req),
      transcribeAudio: (req) => exampleAiMedia.transcribeAudio(req),
      describeVideo: (req) => exampleAiMedia.describeVideo(req),
    });

    api.registerWebSearchProvider({
      id: "exampleai-search",
      createTool() {
        // 返回供应商所有的 Web 搜索工具。
      },
    });
  },
});
```

重要的不是确切的辅助程序名称，而是整体结构：

- 一个插件负责供应商接口
- 核心仍负责能力契约
- 提供商请求转换和 HTTP 辅助程序保留在供应商插件中
- 渠道和功能插件使用 `api.runtime.*` 辅助程序，而不是供应商代码
- 契约测试可以断言插件注册了其声称负责的能力

### 能力示例：视频理解

OpenClaw 已将图像/音频/视频理解视为一项共享能力。同一所有权模型也适用于该能力：

<Steps>
  <Step title="核心定义契约">
    核心定义媒体理解契约。
  </Step>
  <Step title="供应商插件注册">
    供应商插件根据需要注册 `describeImage`、`transcribeAudio` 和 `describeVideo`。
  </Step>
  <Step title="使用方采用共享行为">
    渠道和功能插件使用共享的核心行为，而不是直接连接到供应商代码。
  </Step>
</Steps>

这样可避免将某个提供商对视频的假设固化到核心中。插件负责供应商接口；核心负责能力契约和回退行为。

视频生成已经采用相同的流程：核心负责类型化能力契约和运行时辅助程序，供应商插件则针对该契约注册 `api.registerVideoGenerationProvider(...)` 实现。

需要具体的推出检查清单？请参阅[能力扩展手册](/zh-CN/plugins/adding-capabilities)。

## 契约与强制执行

插件 API 接口在 `OpenClawPluginApi` 中有意采用类型化和集中式设计。该契约定义了受支持的注册点，以及插件可以依赖的运行时辅助程序。

这很重要，原因如下：

- 插件作者可以遵循一个稳定的内部标准
- 核心可以拒绝重复所有权，例如两个插件注册相同的提供商 ID
- 启动时可以针对格式错误的注册提供可操作的诊断信息
- 契约测试可以强制执行内置插件所有权并防止无提示偏移

强制执行分为两层：

<AccordionGroup>
  <Accordion title="运行时注册强制执行">
    插件注册表会在插件加载时验证注册。示例：重复的提供商 ID、重复的语音提供商 ID 和格式错误的注册会产生插件诊断信息，而不是导致未定义行为。
  </Accordion>
  <Accordion title="契约测试">
    测试运行期间，内置插件会记录在契约注册表中，以便 OpenClaw 能够明确断言所有权。目前，这用于模型提供商、语音提供商、Web 搜索提供商以及内置注册所有权。
  </Accordion>
</AccordionGroup>

实际效果是，OpenClaw 可以预先知道哪个插件负责哪个接口。由于所有权是显式声明、类型化且可测试的，而非隐式的，因此核心和渠道可以无缝组合。

### 契约中应包含的内容

<Tabs>
  <Tab title="良好的契约">
    - 类型化
    - 精简
    - 特定于能力
    - 由核心负责
    - 可由多个插件复用
    - 渠道和功能无需了解供应商即可使用

  </Tab>
  <Tab title="不良的契约">
    - 隐藏在核心中的供应商特定策略
    - 绕过注册表的一次性插件逃生通道
    - 渠道代码直接访问供应商实现
    - 不属于 `OpenClawPluginApi` 或 `api.runtime` 的临时运行时对象

  </Tab>
</Tabs>

如有疑问，请提高抽象层级：先定义能力，再让插件接入该能力。

## 执行模型

原生 OpenClaw 插件与 Gateway 网关在**同一进程中**运行。它们未进行沙箱隔离。加载的原生插件与核心代码具有相同的进程级信任边界。

<Warning>
原生插件的影响：插件可以注册工具、网络处理程序、钩子和服务；插件缺陷可能导致 Gateway 网关崩溃或不稳定；恶意原生插件等同于在 OpenClaw 进程内执行任意代码。
</Warning>

兼容包默认更安全，因为 OpenClaw 目前将其视为元数据/内容包。在当前版本中，这主要指内置 Skills。

对于非内置插件，请使用允许列表和显式安装/加载路径。应将工作区插件视为开发时代码，而不是生产环境默认项。

对于内置工作区软件包名称，默认情况下应将插件 ID 与 npm 名称绑定：`@openclaw/<id>`；当软件包有意公开范围更窄的插件角色时，也可以使用经批准的类型化后缀，例如 `-provider`、`-plugin`、`-speech`、`-sandbox` 或 `-media-understanding`。

<Note>
**信任说明：**`plugins.allow` 信任的是**插件 ID**，而不是来源出处。启用工作区插件或将其加入允许列表后，如果它与内置插件具有相同的 ID，该工作区插件会有意覆盖内置副本。这是正常行为，有助于本地开发、补丁测试和热修复。内置插件的信任由加载时磁盘上的源快照（清单和代码）决定，而不是由安装元数据决定。损坏或被替换的安装记录无法在实际源代码所声明的范围之外，悄然扩大内置插件的信任范围。
</Note>

## 导出边界

OpenClaw 导出能力，而不是为了实现方便而导出内容。

保持能力注册公开。精简不属于契约的辅助程序导出：

- 内置插件特定的辅助程序子路径
- 不打算作为公共 API 的运行时管道子路径
- 供应商特定的便捷辅助程序
- 属于实现细节的设置/新手引导辅助程序

为内置插件保留的辅助程序子路径已从生成的 SDK 导出映射中移除。将所有者特定的辅助程序保留在所属插件软件包内；仅将可复用的宿主行为提升为通用 SDK 契约，例如 `plugin-sdk/gateway-runtime`、`plugin-sdk/security-runtime` 和注入的插件 API 能力。

## 内部机制与参考

有关加载管道、注册表模型、提供商运行时钩子、Gateway 网关 HTTP 路由、消息工具架构、渠道目标解析、提供商目录、上下文引擎插件以及添加新能力的指南，请参阅[插件架构内部机制](/zh-CN/plugins/architecture-internals)。

## 相关内容

- [构建插件](/zh-CN/plugins/building-plugins)
- [插件清单](/zh-CN/plugins/manifest)
- [插件 SDK 设置](/zh-CN/plugins/sdk-setup)
