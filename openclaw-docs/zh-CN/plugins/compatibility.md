---
read_when:
    - 你维护一个 OpenClaw 插件
    - 你看到插件兼容性警告
    - 你正在规划插件 SDK 或清单迁移
summary: 插件兼容性契约、弃用元数据和迁移预期
title: 插件兼容性
x-i18n:
    generated_at: "2026-07-26T06:51:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80cf1dfce9e0538e78138ff80a6807ee36267a07d3eee6f19bd8e56e5c0c9cd3
    source_path: plugins/compatibility.md
    workflow: 16
---

OpenClaw 在移除旧版插件契约之前，会通过具名兼容性适配器继续连接这些契约。这可在 SDK、清单、设置、配置和 Agent Runtimes 契约演进期间保护现有的内置插件和外部插件。

## 兼容性注册表

插件兼容性契约记录在核心注册表
`src/plugins/compat/registry.ts` 中。每条记录包含：

- 稳定的兼容性代码
- 状态：`active`、`deprecated`、`removal-pending` 或 `removed`
- 所有者：`sdk`、`config`、`setup`、`channel`、`provider`、`plugin-execution`、
  `agent-runtime` 或 `core`
- 适用时的引入日期和弃用日期
- 所有者维护者批准后的确切移除日期；若省略
  `removeAfter`，已弃用的表面将不具备移除资格
- 替代方案指南
- 涵盖新旧行为的文档、诊断和测试

该注册表是维护者规划和未来插件检查器检查的依据。如果面向插件的行为发生变化，请在添加适配器的同一项变更中添加或更新兼容性记录。

Doctor 修复和迁移兼容性单独记录在
`src/commands/doctor/shared/deprecation-compat.ts` 中。这些记录涵盖旧配置结构、安装账本布局，以及可能需要在运行时兼容路径移除后继续保留的修复适配层。

发布清理应检查这两个注册表。不要仅仅因为匹配的运行时或配置兼容性记录已过期，就删除 Doctor 迁移；应先确认不存在仍需该修复的受支持升级路径。在发布规划期间，也要重新验证每条替代方案注释，因为随着提供商和渠道移出核心，插件所有权和配置占用范围可能会发生变化。

## 弃用策略

OpenClaw 不应在引入替代方案的同一版本中移除已有文档记录的插件契约。迁移顺序：

1. 添加新契约。
2. 通过具名兼容性适配器继续连接旧行为。
3. 在插件作者能够采取行动时发出诊断或警告。
4. 记录替代方案和时间表。
5. 测试新旧两条路径。
6. 等待已公布的迁移窗口结束。
7. 仅在获得明确的破坏性版本批准后移除。

已弃用的记录必须包含警告开始日期、替代方案、文档链接，以及不晚于警告开始后三个月的最终移除日期。不要添加移除窗口无明确期限的已弃用兼容路径，除非维护者明确决定将其作为永久兼容性，并将其标记为 `active`。

## 当前兼容性区域

2026 年 7 月的清理移除了已过期的根 SDK、清单、提供商、运行时、注册表标志和插件自有 Web 配置别名。Doctor 迁移仍单独记录，以便受支持的升级路径仍可修复旧配置。

剩余的有日期限制的兼容性区域包括：

- 迁移指南中列出的 8 月和 9 月 SDK 子路径窗口
- `api.on("deactivate", ...)` 和 `api.on("subagent_spawning", ...)` 钩子别名
- 内存专用嵌入注册和 beta.5 会话存储桥接
- 下文所述的 WhatsApp 入站回调别名
- 显式渠道目标解析和 `openclaw/plugin-sdk/messaging-targets`
- 嵌入式 Pi 智能体别名
- 已发布的 Agent harness SDK 别名，其移除仍有待新的外部迁移文档决策

活跃且无日期限制的注册表记录涵盖受支持的行为，而非待移除的兼容性负担，包括激活提示、插件捕获、内置插件启用，以及生成的渠道配置回退。

### WhatsApp 入站回调扁平别名

WhatsApp 运行时回调会传递 `WebInboundMessage`：规范的嵌套 `event`、`payload`、`quote`、`group` 和 `platform` 上下文，以及已发布回调字段的已弃用扁平别名。新的回调代码应读取嵌套上下文。构造纯净嵌套回调消息的代码可以使用 `WebInboundCallbackMessage`；仍注入旧版扁平测试或插件消息的兼容性监听器应使用 `LegacyFlatWebInboundMessage` 或 `WebInboundMessageInput`。

扁平别名将保留至 **2026-08-30**；该窗口仅适用于扁平别名访问，不适用于嵌套结构，后者是规范的运行时契约。每个扁平别名的 TypeScript `@deprecated` 注释都会指明其确切的嵌套替代项。常见示例：

- `id`、`timestamp` 和 `isBatched` 移至 `event` 下。
- `body`、`mediaPath`、`mediaType`、`mediaFileName`、`mediaUrl`、`location`
  和 `untrustedStructuredContext` 移至 `payload` 下。
- `to`、`chatId`、发送者/自身字段、`sendComposing`、`reply(...)` 和
  `sendMedia(...)` 移至 `platform` 下。
- `replyTo*` 字段移至 `quote` 下；群组主题/参与者/提及
  字段移至 `group` 下。

`payload.untrustedStructuredContext` 从入站提供商载荷中提取。插件应先检查 `label`、`source` 和 `type`，再将其 `payload` 视为权威信息。

### WhatsApp 入站准入字段

已接受的 WhatsApp 回调消息会携带 `admission`，这是一个可安全公开的信封，包含准许该消息进入的访问控制决策。新的回调代码应从 `msg.admission` 读取准入事实，而非读取较旧的顶层准入字段。

顶层字段将保留至 **2026-08-30**。每个字段的 TypeScript `@deprecated` 注释都会指明其替代项：

- `from` 和 `conversationId` 移至 `admission.conversation.id`。
- `accountId` 移至 `admission.accountId`。
- `accessControlPassed` 是 `admission.ingress.decision === "allow"` 的派生兼容性视图；对于已经携带
  `admission` 的消息，写入旧版布尔值不会重写入口
  图。
- `chatType` 移至 `admission.conversation.kind`。

## 插件检查器软件包

插件检查器应作为独立的软件包/仓库存在于 OpenClaw 核心仓库之外，并由版本化的兼容性和清单契约提供支持。首日 CLI 应为：

```sh
openclaw-plugin-inspector ./my-plugin
```

它应输出清单/模式验证、正在检查的契约兼容性版本、安装/来源元数据检查、冷路径导入检查，以及弃用/兼容性警告。在 CI 注释中使用 `--json` 提供稳定的机器可读输出。OpenClaw 核心应公开检查器可使用的契约和固件，但不应从主 `openclaw` 软件包发布检查器二进制文件。

### 维护者验收通道

针对 OpenClaw 插件软件包验证外部检查器时，请使用由 Crabbox 支持的 Blacksmith Testbox 作为可安装软件包的验收通道。构建软件包后，从干净的 OpenClaw 检出中运行：

```sh
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "pnpm install && pnpm build && npm exec --yes @openclaw/plugin-inspector@0.1.0 -- ./extensions/telegram --json"
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "npm exec --yes @openclaw/plugin-inspector@0.1.0 -- ./extensions/discord --json"
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "npm exec --yes @openclaw/plugin-inspector@0.1.0 -- <clawhub-plugin-dir> --json"
```

此通道应仅供维护者选择启用，因为它会安装外部 npm 软件包，并且可能检查从仓库外部克隆的插件软件包。本地仓库防护覆盖 SDK 导出映射、兼容性注册表元数据、已弃用 SDK 导入的逐步清除，以及内置扩展的导入边界；Testbox 检查器验证则覆盖外部插件作者实际使用该软件包的方式。

## 发布说明

在兼容路径转为 `removal-pending` 或 `removed` 之前，发布说明应包含即将到来的插件弃用事项、目标日期和迁移文档链接。
