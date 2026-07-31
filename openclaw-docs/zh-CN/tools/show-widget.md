---
read_when:
    - 你希望智能体在网页聊天、原生应用或 Discord 中呈现交互式结果
    - 你希望小组件按钮将后续提示词发送到聊天中
    - 你希望使用共享设计令牌为小组件设置主题样式
    - 你需要 `show_widget` 的输入、安全性或保留契约
sidebarTitle: Show widget
summary: 在支持的聊天界面上显示独立的 HTML 小组件
title: 显示小组件
x-i18n:
    generated_at: "2026-07-26T07:04:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 903adff1fadeb9d224d3e2d839c86082b5244e1e319255c8d3f6619344b749a3
    source_path: tools/show-widget.md
    workflow: 16
---

`show_widget` 是一个核心工具，可在用户当前界面上显示一个自包含的 HTML 小组件。OpenClaw 会在 Control UI 以及 iOS、Android、macOS 和 Linux Quick Chat 对话记录中内联渲染该小组件；Linux 仪表板使用浏览器版 Control UI。在启用了 [Activities](/channels/discord-activities) 的 Discord 会话中，Discord 插件会发布一个 **Open widget** 按钮，以 Activity 形式启动该小组件。

## 小组件的工作原理

当智能体调用 `show_widget` 时，OpenClaw 核心会将 `widget_code` 包装在一个最小化 HTML 文档中，将其存储为 Canvas 文档，并返回预览句柄。Control UI 在沙箱隔离的 iframe 中渲染该句柄，而 iOS、Android、macOS 和 Linux Quick Chat 使用隔离的 WebView。完整聊天客户端会在重新加载历史记录后恢复小组件；Quick Chat 会在其当前回复期间保留小组件。

在 Control UI 会话中，还可将 Canvas 小组件固定到会话仪表板。可在工具调用中设置 `pin: true`，或对现有对话记录小组件使用 **Pin to dashboard**。固定的 HTML 在 MCP Apps 所使用的同一专用源、双 iframe 沙箱宿主中运行；浏览器绝不会在不受信任的框架内解析小组件数据绑定。

对于浏览器嵌入，包装文档会围绕小组件代码注入四个小型宿主桥接器：

- 尺寸报告器将渲染内容的高度发送到嵌入聊天界面，后者会限制该高度并调整 iframe 大小（160 到 1200 像素）。
- 宿主桥接器定义旧版 `sendPrompt(text)` 辅助函数，以及结构化的 `openclaw.prompt`、`openclaw.state`、`openclaw.data` 和 `openclaw.cron` API。内联聊天提示保留其私有消息渠道；仪表板 API 使用绑定到视图票据的请求渠道。请参阅[交互式小组件](#interactive-widgets)和[仪表板能力](#dashboard-capabilities)。
- 主题桥接器监听 Control UI 当前的设计令牌，并在加载时以及每次主题变化时将其应用为 CSS 变量。
- 当嵌入聊天界面请求导出时，快照桥接器会将当前小组件文档渲染为 PNG。

其他所有内容均保留在框架内：文档在具有严格内容安全策略的不透明源中运行，因此小组件脚本无法访问 Control UI、Gateway 网关或网络。

仅当发起请求的 Gateway 网关客户端声明 `inline-widgets` 能力时，核心实现才可用。Control UI 和受支持的原生应用会自动声明此能力。对于需要自定义 TLS 叶证书固定的 Gateway 网关连接，Linux Quick Chat 会保持纯文本模式，因为其平台 WebView 无法绑定该固定证书。Discord 实现仅在已配置 Activities 的 Discord 会话中可用。其他渠道运行不会收到 `show_widget`。

能力传输涵盖嵌入式、Codex app-server 和基于 CLI 的模型后端。通过授权认证的 MCP 调用方和直接 HTTP 工具调用方仍会以失败关闭方式处理，因为它们不声明客户端能力。

## 设计系统

每个 Canvas 小组件都包含一个无类名基础样式表和一组小型令牌：

| 令牌                                                                                  | 用途                                  |
| ------------------------------------------------------------------------------------- | ------------------------------------- |
| `--surface`                                                                    | 页面级界面背景色                      |
| `--card`                                                                    | 卡片、按钮和代码背景                  |
| `--elevated`                                                                    | 浮层表单控件背景                      |
| `--text`                                                                    | 默认正文和控件文本                    |
| `--text-strong`                                                                    | 标题和突出显示的值                    |
| `--muted`                                                                    | 次要文本和浅色边框                    |
| `--border`                                                                    | 标准分隔线和卡片边框                  |
| `--border-strong`                                                                    | 强调控件边框                          |
| `--accent`                                                                    | 链接和焦点环                          |
| `--accent-fill`                                                                    | 主要操作填充色                        |
| `--accent-fg`                                                                    | 主要操作上的文本                      |
| `--ok`                                                                    | 成功状态                              |
| `--warn`                                                                    | 警告状态                              |
| `--danger`                                                                    | 错误或破坏性状态                      |
| `--info`                                                                    | 信息状态                              |
| `--radius`                                                                    | 控件和卡片共用的圆角半径              |
| `--font-body`                                                                    | 宿主正文字体栈                        |
| `--font-mono`                                                                    | 宿主等宽字体栈                        |
| `--accent-subtle`、`--ok-subtle`、`--warn-subtle`、`--danger-subtle`、`--info-subtle` | 派生的半透明状态背景                  |

无类名的标题、段落、链接、按钮、输入框、选择框、文本区域、表格和代码块会获得基础样式。辅助类提供常用模式：

- `.card` 用于带边框的内容界面
- `.badge`，以及 `.ok`、`.warn`、`.danger` 或 `.info`，用于紧凑型状态标签
- `.metric` 用于突出显示的数值
- `.muted` 用于次要文本
- `.row` 用于可换行的水平布局
- `button.primary` 用于主要操作

当小组件加载以及每次主题变化时，Control UI 都会发送一条包含当前主题值的 `openclaw:widget-theme` 消息。因此，小组件无需重新加载即可跟踪所有主题系列，包括 Claw、Knot、Dash 和自定义主题。在 Control UI 之外（包括原生应用和直接打开的页面），小组件使用由 `prefers-color-scheme` 选择的内置浅色或深色调色板。

编写小组件时遵循三条规则：

1. 所有颜色和背景均使用设计变量。不要硬编码颜色值。
2. 保持页面背景透明，使小组件融入其宿主界面。
3. 最多仅为一个主要操作保留 `--accent-fill`。

**导出：**在网页聊天中，打开小组件卡片菜单，可将渲染后的小组件复制到剪贴板或下载为 PNG。不包含快照桥接器的旧版小组件文档会回退为下载 HTML 文件。

## 使用工具

两种实现使用相同的必填字段：

<ParamField path="title" type="string" required>
  与内联预览一同显示并用作托管文档标题的简短标题。
</ParamField>

<ParamField path="widget_code" type="string" required>
  自包含的 HTML 或 SVG。对于内联小组件客户端，如果去除首尾空白后的输入以 `<svg` 开头，则会以 SVG 模式渲染；最大长度为 262,144 个字符。Discord 接受最大 48 KiB 的完整 HTML 文档或正文片段。
</ParamField>

Discord 还接受可选的 `button_label` 文本，用于 Activity 启动按钮。Canvas 架构有意省略了这个仅限 Discord 的字段。

核心 Canvas 工具接受以下可选的仪表板放置字段：

- `pin`：同时将小组件放置在会话仪表板上。
- `name`：稳定的小组件名称；默认为 `title` 的 slug。
- `tab`：目标标签页 slug。
- `size`：`sm`、`md`、`lg`、`xl` 或 `full` 之一。
- `after`：将小组件放置在其后的同级小组件名称。
- `capabilities`：固定小组件请求的访问权限。`netOrigins` 包含确切的 HTTPS 源；`tools` 包含 `prompt`、允许列表中的读取绑定或确切的 `cron.trigger:<jobId>` 操作。

核心结果包含 Canvas 预览句柄，因此 Control UI 和受支持的原生应用可以直接根据工具调用渲染小组件，并在重新加载历史记录后恢复它。固定结果还会保留看板小组件名称，因此 Control UI 不会在重新加载对话记录后再次提供重复固定选项。Discord 返回已存储的小组件标识符和已发布消息标识符。

`discord_widget` 会作为已弃用的别名继续注册一个版本。新的智能体调用应使用 `show_widget`。

## 交互式小组件

在 Control UI 中，小组件脚本可以驱动对话。包装文档定义了一个全局 `sendPrompt(text)` 函数；调用它会将 `text` 提交到聊天中，就像用户键入并发送了该消息一样。可将其连接到按钮或其他控件，以构建选择器、测验或逐层深入的仪表板等交互流程。原生应用会渲染交互式小组件代码，但不公开此聊天提示桥接器。

```html
<button onclick="sendPrompt('详细显示失败的测试')">失败的测试</button>
```

每个提示都会在框架边界的两侧进行验证：

- `sendPrompt` 要求小组件内存在[瞬时用户激活](https://developer.mozilla.org/en-US/docs/Web/Security/User_activation)：它仅在用户点击小组件或在其中按键后的几秒内有效，因此应将其连接到按钮和其他点击目标——加载时自动调用不会产生任何效果。桥接器会将发送端点仅保留给自身，并在不公开用户激活状态的浏览器中以失败关闭方式处理，因此小组件代码无法绕过此检查。
- 提示权限仅属于原始小组件文档。可信桥接器会在小组件代码能够运行或导航框架之前，将其渠道端点提供给聊天界面；聊天界面仅采用第一次提供的端点，并且导航后该渠道会随文档一同失效。外部允许的嵌入 URL 永远不会被采用。
- 小组件框架必须在聊天对话记录中可见并持有焦点——这是宿主观察到的另一项信号，用于确认用户确实正在与此小组件交互。
- 文本去除首尾空白后不得为空，并且最多为 4,000 个字符。
- 以 `/` 开头的提示会被拒绝，因此小组件代码无法触发 `/approve` 或 `/stop` 等聊天命令。
- 每个小组件文档在任意滚动的一分钟内最多可发送 10 个提示；超出的提示会被静默丢弃。

接受的提示会作为普通用户消息出现在对话记录中，并在拥有该小组件的会话中启动正常的智能体轮次。小组件没有反馈渠道：被丢弃的提示会静默失败，并且小组件无法读取智能体的回复。

## 仪表板能力

在操作员审查待处理卡片上显示的声明后，固定小组件可以使用一个绑定到票据的宿主 API：

- `openclaw.prompt.send(text)` 需要临时用户激活，并会发布一条可见的编辑器消息。声明并获得 `prompt` 工具授权后，可跳过每次点击时的额外确认；验证、焦点检查和速率限制仍然适用。
- `openclaw.state.emit(payload)` 会添加一条会话通知。载荷上限为 8 KiB，客户端在五秒内发出的相同内容会被合并。
- `openclaw.data.read(bindingId, params?)` 仅在 Gateway 网关处解析。可授权的绑定包括 `sessions.list`、`usage.status`、`usage.cost`、`cron.list`、`cron.status`、`agents.list` 和 `health`。
- `openclaw.cron.trigger(jobId)` 仅当已授予完全匹配的 `cron.trigger:<jobId>` 能力时，才会立即运行现有任务。

网络访问与主机工具相互独立。将确切的 HTTPS 源放入 `capabilities.netOrigins`；获得批准后，只有这些源会进入小组件的 `connect-src`。通配符、凭据、路径、查询字符串和未声明的源仍会被阻止。仅当字面端口是所声明源的一部分时，才允许使用。

## 安全与存储

小组件文档使用严格的内容安全策略。允许内联样式和脚本，但仍会阻止加载外部资源。内联会话记录小组件无法访问网络。固定到仪表板的小组件只能访问由智能体声明并经操作员授权的确切 HTTPS 源。

Control UI iframe 始终省略 `allow-same-origin`，即使全局嵌入模式为 `trusted`，因此小组件脚本无法读取父应用程序的源。原生客户端使用隔离的非持久化 Web 视图，并阻止导航离开托管的小组件。核心文档主机还会在提供小组件时附加 `Content-Security-Policy: sandbox allow-scripts` 响应标头，因此直接渲染时，小组件仍会在不透明源而非应用程序源中运行。仅渲染你愿意在该隔离框架中执行的小组件代码。

iframe 还遵循 [`gateway.controlUi.embedSandbox`](/zh-CN/web/control-ui#hosted-embeds)。默认的 `scripts` 层级支持交互式小组件，同时保持源隔离。

已接受的 WebRTC 数据通道出站残余风险记录在[仪表板架构](/web/dashboard-architecture#modeled-residual-webrtc-data-channels)中。

Canvas 每个会话最多保留 32 个小组件（没有可用会话时，则按每个智能体计算）。创建其他小组件会移除该范围内最早的文档。

## 相关内容

- [Control UI 托管嵌入](/zh-CN/web/control-ui#hosted-embeds)
- [Discord Activities](/channels/discord-activities)
- [Canvas 节点控件](/zh-CN/plugins/reference/canvas)
- [Gateway 网关协议客户端能力](/zh-CN/gateway/protocol#client-capabilities)
