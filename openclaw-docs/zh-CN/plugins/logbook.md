---
read_when:
    - 你希望在 Control UI 中查看类似 Dayflow 的全天时间线
    - 你正在启用或配置内置的 Logbook 插件
    - 你希望获得基于屏幕活动的站会摘要或当日回顾
summary: 基于定期屏幕快照构建的可选自动工作日志
title: 日志簿插件
x-i18n:
    generated_at: "2026-07-26T06:52:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 19197e580421dfe81f82f8599578e4c68a15004813bb2b6c3de761c14f426b08
    source_path: plugins/logbook.md
    workflow: 16
---

Logbook 插件可将屏幕活动自动转化为工作日志。它会定期从已配对的节点捕获屏幕快照，将其总结为带时间戳的观察记录，并在
[Control UI](/zh-CN/web/control-ui) 中构建时间线卡片。它还可以生成每日站会记录，并回答有关某个已跟踪日期的问题。

OpenClaw 所有的状态保留在 Gateway 网关上的 `<state-dir>/logbook/` 中，但模型处理不一定在本地进行。采样的屏幕截图会发送到已配置的视觉路由；观察记录和时间线文本会发送到默认智能体模型。如果屏幕内容和派生的活动文本必须保留在本机，请为两个阶段都使用本地模型路由。

Logbook 已内置，但默认禁用。启用该插件即表示允许 Gateway 网关捕获屏幕，因为 `captureEnabled` 默认为 `true`。

## 开始之前

你需要：

- 一个已连接且公开 `screen.snapshot` 或 `logbook.snapshot` 的节点。
  macOS 应用节点需要 Screen Recording 权限。无头 macOS 节点主机
  (`openclaw node host run`) 会获得由插件提供的 `logbook.snapshot`
  命令，该命令由系统的 `screencapture` 工具提供支持。
- 已启用并完成身份验证的内置 Codex 插件。Codex 目前提供
  Logbook 所需的结构化图像提取契约。使用
  `openclaw models auth login --provider openai` 登录；其他身份验证方式请参阅
  [Codex harness](/zh-CN/plugins/codex-harness)。
- 一个可用的默认智能体模型。完成视觉处理后，Logbook 使用该模型合成卡片、站会记录以及日期问答。

## 快速开始

启用 Codex 和 Logbook 插件：

```bash
openclaw plugins enable codex
openclaw plugins enable logbook
```

配置显式视觉模型，以确保启动行为确定：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          visionModel: "codex/gpt-5.6-sol",
        },
      },
    },
  },
}
```

如果使用 `plugins.allow`，请同时包含 `codex` 和 `logbook`。更改插件配置后重启
Gateway 网关，然后检查注册信息并打开仪表板：

```bash
openclaw gateway restart
openclaw plugins inspect logbook --runtime --json
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw dashboard
```

节点描述必须包含 `screen.snapshot` 或 `logbook.snapshot`。
无头节点仅在插件处于活动状态后才会公布 `logbook.snapshot`。
如果缺少该命令，请参阅[节点故障排查](/zh-CN/nodes/troubleshooting)。

仅当插件已启用且 Control UI 会话具有 `operator.write` 时，才会显示 Logbook 标签页。状态行应显示 **正在捕获**，且没有错误。
分析窗口关闭时会显示时间线卡片；捕获活动后，也可以选择
**立即分析**。

## 工作原理

1. **捕获**：Logbook 每隔 `captureIntervalSeconds`（默认 30 秒）调用
   所选节点的捕获命令，并存储一帧缩放后的 JPEG 图像。
   连续相同的帧会标记为空闲，并从分析中排除。
2. **观察**：分析窗口（默认 15 分钟）结束后，
   插件最多采样 16 个活动帧并将其发送到视觉模型，
   后者返回带时间戳的活动观察记录（“VS Code：正在编辑
   store.ts，修复类型错误”）。超过两分钟的捕获间隔或
   本地午夜也会关闭当前窗口。
3. **合成**：观察记录与现有卡片中最近 45 分钟的内容
   会被修订为时间线卡片（每张 10-60 分钟），其中包含标题、摘要、
   类别、主要应用以及任何短暂的分心活动。
4. **清理**：删除超过 `retentionDays` 天（默认 14 天）的帧。
   卡片、观察记录和缓存的站会记录会保留。

日期边界和时间线时钟使用 Gateway 网关的本地时区，而不是
浏览器时区。帧和 SQLite 时间线数据库位于
`<state-dir>/logbook/` 下。

## 模型和数据流

Logbook 使用两个独立的模型路由：

| 阶段             | 发送的数据                                                  | 模型路由                                                          |
| ---------------- | --------------------------------------------------------- | ----------------------------------------------------------------- |
| 观察             | 最多 16 个采样的 JPEG 帧及其捕获时间                       | `visionModel`，或兼容的借用 `tools.media` Codex 条目 |
| 合成卡片         | 带时间戳的观察记录和近期时间线卡片                         | 通过插件 LLM 运行时使用默认智能体模型                             |
| 生成站会记录     | 所选日期和前一天的卡片                                     | 通过插件 LLM 运行时使用默认智能体模型                             |
| 询问当天活动     | 问题、所选日期的卡片和近期观察记录                         | 通过插件 LLM 运行时使用默认智能体模型                             |

完整的 SQLite 数据库不会发送给任一模型。原始屏幕截图仅发送到
观察阶段；卡片合成、站会记录和问答接收的是派生文本。

## 配置

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          captureEnabled: true,
          captureIntervalSeconds: 30,
          analysisIntervalMinutes: 15,
          nodeId: "my-mac",
          screenIndex: 0,
          maxWidth: 1440,
          visionModel: "codex/gpt-5.6-sol",
          retentionDays: 14,
        },
      },
    },
  },
}
```

所有 Logbook 配置键均为可选。数值会舍入为整数，
并限制在支持的范围内。

| 键                        | 默认值  | 范围或值                | 行为                                                                                         |
| ------------------------- | ------- | ----------------------- | -------------------------------------------------------------------------------------------- |
| `captureEnabled`          | `true`  | 布尔值                  | 新快照的持久主开关；当值为 `false` 时，时间线仍然可用                             |
| `captureIntervalSeconds`  | `30`    | `5`-`600`               | 两次捕获尝试之间的延迟                                                                       |
| `analysisIntervalMinutes` | `15`    | `3`-`120`               | 目标观察窗口；捕获间隔和午夜可能使其提前关闭                                                 |
| `nodeId`                  | 未设置  | 节点 ID 或显示名称      | 将捕获固定到一个已连接节点；匹配不区分大小写                                                 |
| `screenIndex`             | `0`     | `0`-`16`                | 从零开始的显示器索引                                                                         |
| `maxWidth`                | `1440`  | `480`-`3840`            | 请求的捕获尺寸上限；无头 macOS 将其应用于最大尺寸                                            |
| `visionModel`             | 未设置  | `provider/model`        | 显式结构化路由；格式错误的引用会暂停分析，不受支持的提供商会导致批次失败                     |
| `retentionDays`           | `14`    | `1`-`365`               | 删除旧帧；卡片、观察记录和站会记录会保留                                                     |

未设置 `nodeId` 时，Logbook 优先选择公开
`screen.snapshot` 的已连接应用节点，然后回退到公开
`logbook.snapshot` 的无头节点。在未固定节点的设置中，失败的节点会排到其他符合条件的节点之后。
仪表板的暂停开关仅对当前会话有效，并会在 Gateway 网关重启时重置；如需持久停止，请使用 `captureEnabled: false`。

### 视觉模型选择

Logbook 按以下顺序解析观察模型：

1. `plugins.entries.logbook.config.visionModel`
2. `tools.media.models` 下第一个支持图像的 Codex 条目

系统会跳过其他媒体提供商，因为它们目前不公开
Logbook 所需的结构化提取契约。设置
`tools.media.image.enabled: false` 会禁用借用的媒体默认值，但显式设置的
Logbook `visionModel` 仍然有效。

## 仪表板标签页

- **时间线**：按活动显示可展开的卡片，其中包含类别颜色、主要
  应用、分心活动标签和快照关键帧。
- **日期概览**：专注比例、类别细分、最常用应用。
- **每日站会记录**：将昨天和今天的内容转化为可直接粘贴的更新。
- **询问当天活动**：根据已跟踪的时间线回答自然语言问题
  （“我是什么时候审查 Gateway 网关 PR 的？”）。
- **立即分析**：立即关闭当前捕获窗口，而不是
  等待分析间隔结束。

## Gateway 网关方法

Logbook 注册以下 Gateway 网关 RPC 方法：

| 方法                  | 参数                     | 权限范围         | 结果                                                                     |
| --------------------- | ------------------------ | ---------------- | ------------------------------------------------------------------------ |
| `logbook.status`      | 无                       | `operator.read`  | 捕获、分析、模型、节点、Gateway 网关日期和 Gateway 网关时区状态          |
| `logbook.days`        | 无                       | `operator.read`  | 包含时间线卡片数量和卡片时间范围的日期                                   |
| `logbook.timeline`    | `{ day?: "YYYY-MM-DD" }` | `operator.read`  | 派生卡片和日期统计信息；默认为 Gateway 网关的当前日期                    |
| `logbook.frames`      | `{ startMs, endMs }`     | `operator.write` | 所请求的 Unix 纪元毫秒范围内的帧元数据                                   |
| `logbook.frame`       | `{ frameId }`            | `operator.write` | 一个以 base64 编码的原始 JPEG 帧                                         |
| `logbook.standup`     | `{ day?, refresh? }`     | `operator.write` | 某一天的缓存或重新生成的站会文本                                         |
| `logbook.ask`         | `{ day?, question }`     | `operator.write` | 基于时间线的某一天回答                                                   |
| `logbook.capture.set` | `{ paused }`             | `operator.write` | 仅限会话的暂停状态和更新后的状态                                         |
| `logbook.analyze.now` | 无                       | `operator.write` | 启动待处理的分析，或返回无法启动的原因                                   |

读取方法返回运行状态或派生文本。原始屏幕截图像素、产生模型费用的操作和运行时变更需要
`operator.write`。Control UI 标签页还需要 `operator.write`，因为它会公开这些操作和原始帧预览；只读客户端仍可直接调用派生文本方法。

## 隐私说明

- 快照可能包含屏幕上的任何内容，包括密钥。除非作为采样输入发送给已配置的观察模型，否则帧绝不会
  离开本机。
- 在卡片合成、站会内容生成或问答期间，观察结果、最近的卡片和问题可能会通过
  默认智能体模型离开本机。将提供商的数据处理策略同时应用于这两条模型路由。
- 需要完全本地化的流水线时，请为结构化观察模型和默认智能体
  模型都使用本地路由。
- 帧、时间线数据库和临时捕获内容均以
  仅所有者可访问的文件权限写入。
- 将 `screen.snapshot` 添加到 `gateway.nodes.commands.deny` 是
  屏幕捕获的终止开关：它既会阻止应用节点捕获，也会阻止 Logbook 自身的
  `logbook.snapshot` 命令。
- 设置 `tools.media.image.enabled: false` 还会阻止 Logbook 借用
  媒体图像模型进行分析；此时仅使用插件配置中显式指定的 `visionModel`。

## 故障排查

### Logbook 选项卡缺失

检查以下三个条件：

1. `openclaw plugins list --enabled` 包含 `logbook`。
2. 更改插件或允许列表后，Gateway 网关已重启。
3. Control UI 连接具有 `operator.write`；只读会话不会
   收到交互式选项卡描述符。

如果已设置 `plugins.allow`，则推荐配置要求它必须同时包含 `logbook` 和 `codex`。

### 捕获报告错误

```bash
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw logs --follow
```

- 确认节点公开了 `screen.snapshot` 或 `logbook.snapshot`。
- 在执行捕获的 Mac 上授予“屏幕录制”权限。
- 如果配置了 `nodeId`，请确认它与节点 ID 或显示名称匹配。
- 检查 `gateway.nodes.commands.deny` 是否不包含
  `screen.snapshot`。

连续失败三次后，Logbook 会暂停十个捕获周期，
然后重试。未固定节点的设置可以轮换到另一个符合条件的节点。

### 捕获成功但未显示卡片

- **缺少模型**状态表示未找到兼容的结构化视觉路由。
  请启用 Codex 插件并完成身份验证，或设置有效且显式的
  `visionModel`。缺少模型期间，捕获的帧会保持待处理状态，
  修复配置后即可进行分析。
- 等待 `analysisIntervalMinutes`，或者在捕获到活动后选择 **立即分析**。
- 连续相同的帧会被视为空闲证据，不会进入分析
  批次。测试前请更改屏幕上的可见内容。
- 如果最新批次显示错误，请修复模型或身份验证问题，然后选择
  **立即分析**。为避免重复产生模型费用，失败的批次仅在执行该显式操作时
  才会重试。

## 相关内容

- [管理插件](/zh-CN/plugins/manage-plugins)
- [Codex harness](/zh-CN/plugins/codex-harness)
- [媒体理解](/zh-CN/nodes/media-understanding)
- [节点](/zh-CN/nodes)
- [节点故障排查](/zh-CN/nodes/troubleshooting)
- [Control UI](/zh-CN/web/control-ui)
