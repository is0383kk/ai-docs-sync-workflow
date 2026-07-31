---
read_when:
    - 实现或审查会话仪表板（看板）功能
    - 更改小组件托管、小组件桥接或看板存储
summary: 会话仪表板：架构与实施计划（技术设计，GA 前）
title: 仪表板架构
x-i18n:
    generated_at: "2026-07-26T07:06:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a7c5da94ec19add55c6b7b530f0c17509a027e97fb301469ce48f520b325c169
    source_path: web/dashboard-architecture.md
    workflow: 16
---

<Note>
会话仪表板功能的技术设计文档，编写于实现之前及实现期间。它是功能构建的权威依据。功能发布后，`/web/dashboard` 将成为面向用户的页面，而本页面将继续作为架构参考。
</Note>

## 愿景

如今，与智能体协作就是面对文本流。仪表板将其变成工作台：智能体渲染实时交互式小组件；用户将它们固定到持久化界面上；聊天停靠在侧边（或隐藏），主内容则是看板。无需离开会话，即可从“与智能体交谈”转变为“操作智能体为你构建的控制面板”。

原则：

- **看板是会话的一个界面，而不是新对象。** 每个会话（线程）都有两个界面：对话记录和看板。没有固定小组件的会话就是普通聊天。固定一个小组件，看板随即存在。看板继承会话的身份、智能体所有权、命名、固定状态和生命周期。不存在 `dashboard_create`，没有看板注册表，也没有单独的 ACL 模型。
- **智能体能力对等。** 用户能在看板上执行的一切操作，智能体都能通过工具完成：添加/更新/移除小组件、排列小组件、管理标签页、切换可见标签页，以及停靠或隐藏聊天。
- **原生，而非嵌入。** 看板由 Control UI 外壳中的 Lit 组件构成（与应用其余部分使用相同的设计系统）。只有小组件的_内容_在 iframe 中进行沙箱隔离。没有地址栏，也没有浏览器边框。
- **精简的智能体接口。** 小组件通过稳定名称寻址，并就地更新。布局采用可自动紧凑排列的流式网格；智能体只指定尺寸和锚点，绝不指定像素或坐标。
- **能力优先于信任。** 小组件代码是经过严格沙箱隔离、由智能体任意编写的 HTML/JS。访问能力（Gateway 网关数据、操作、网络）只能通过已声明且由操作员授予的能力清单获得。

## 概念

| 概念                | 定义                                                                                                                                                              |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 会话（线程）        | 现有 Gateway 网关会话，以稳定的 `sessionKey` 为键。归智能体所有。                                                                                           |
| 看板                | 单个会话的小组件界面。仅当会话具有小组件/标签页时存在。在 `/new`/`/reset` 后仍会保留（附加到 `sessionKey`，而非对话记录）。             |
| 标签页              | 看板的呈现页面：包含哪些小组件、它们的排列方式，以及聊天停靠状态（`left`/`right`/`bottom`/`hidden`）。看板初始包含一个隐式标签页。 |
| 小组件              | 归会话所有、具有名称且经过沙箱隔离的 HTML/JS 程序。以 `sessionKey` + `name` 寻址。按名称就地更新。                                               |
| 能力清单            | 每个小组件的访问能力声明：`data`（读取绑定）、`actions`（列入允许列表的动词）、`prompt`（发送到会话）、`net`（允许的来源）。 |
| 固定（小组件）      | 将对话记录中的小组件移到会话看板上（通过用户操作界面或智能体工具参数）。取消固定会将其从看板中移除。                                                              |
| 固定（会话）        | 现有的侧边栏会话固定功能。具有看板的已固定会话会打开其看板界面。                                                                                                  |

## 用户体验流程

- **升级：** 智能体在任意聊天中调用 `show_widget` → 小组件与当前行为完全相同，在对话记录中内联渲染 → 悬停时显示 **固定到仪表板** → 小组件出现在会话看板上。智能体可以传入 `pin: true` 来实现相同效果。
- **看板视图：** 具有看板的会话会显示界面切换开关（聊天 / 仪表板）。看板视图 = 标签栏（仅在标签页数量 >1 时显示）+ 流式网格 + 停靠的聊天窗格。聊天停靠区与侧边栏完全相同，可调整大小、移动（左侧/右侧/底部）和折叠。系统会记住每个标签页的停靠状态。
- **拖动：** 用户拖动小组件；网格自动紧凑排列（小组件向上浮动，相邻项重新排布）。通过手柄调整大小时会吸附到尺寸档位。任何人都不能按像素定位。
- **重置警告：** 对具有看板的会话执行 `/new` / `/reset` 时，Web UI 会请求确认（“上下文将重置，仪表板会保留”），并保留看板。
- **侧边栏：** 已固定会话如果具有看板，则渲染其看板界面。Home 会话的看板是默认的“智能体仪表板”。
- **交互**（分为以下三个层级）：静默状态事件、可见提示发送和自动化触发器。

## 交互层级

1. **状态事件（默认）。** 模型应知晓但无需响应的小组件 UI 交互。`bridge.emitState({...})` 会追加结构化会话通知（机制与群组活动通知相同）。不会启动智能体轮次；模型会在下次运行时看到积累的通知。
2. **提示（显式交谈）。** `bridge.sendPrompt(text)` — 需要用户激活；将一条可见的用户消息发送到会话中（停靠的聊天区会显示该消息）。受速率限制；除非小组件持有 `prompt` 能力授权，否则每次发送都需用户确认。
3. **自动化。** `bridge.runAction(name, args)` — 触发清单中声明的操作。初始动词集：`cron.trigger`（立即运行现有定时任务）和 `binding.refresh`。定时任务已在可见且隔离的运行会话中执行，并可使用成本更低的模型：这就是“小模型驱动小组件”的实现路径。任何地方都不存在隐藏会话。

## 小组件模型与托管

小组件 HTML/JS 由智能体编写（通常通过 `show_widget`），封装在标准文档外壳中（CSP 元标记、尺寸报告器、桥接引导程序），并在 `<iframe sandbox="allow-scripts">` 中渲染（绝不使用 `allow-same-origin`）。

- **内联（对话记录）小组件**保留当前的画布文档流水线：写入状态目录，由 Gateway 网关提供，按作用域清理，无需审批（它们在设计上不具备任何能力——发送提示需用户确认）。
- **看板小组件**属于会话状态：字节存储在所属智能体的 SQLite 数据库中（`board_widgets`），由读取该数据库的核心 Gateway 网关路由（`/__openclaw__/board/<agentId>/<sessionKey>/<name>/`）提供。固定对话记录小组件时会复制其字节。限制：每个小组件 256 KB，每个看板 48 个小组件。
- **就地更新：** 重新发出具有相同 `name` 的小组件会替换其字节、递增 `revision`、广播 `board.changed`，并且实时视图只重新加载该 iframe。
- **字节冻结：** 已授予的能力绑定到小组件字节的 sha256。更改字节后，仅当新版本声明的是已授权清单的子集时，才会保留 `data`/`net`/`actions` 授权；扩大的清单会再次提示操作员。

### 小组件托管内容；MCP 应用是其中一种内容类型

**小组件是 OpenClaw 的基础单元**：它是有名称、已固定、具有尺寸、归会话所有且带有授权记录的看板单元格。其中渲染的是某种内容类型：

- `html` — 智能体通过 `show_widget` 编写，字节存储在看板存储中。
- `mcp-app` — 托管在小组件单元格中的第三方 MCP 应用视图（来自已配置服务器的 `ui://` 资源）。

MCP 应用不定义小组件模型；小组件只是获得了托管它们的能力。身份、位置、固定、授权以及面向作者的 API 仍由 OpenClaw 定义，因此 `show_widget` 代码可以像现在一样简短，并且永远不需要知道 MCP Apps 规范的存在。

底层共享基础设施（简化在此落地）：

- **一个沙箱宿主。** `html` 小组件通过与 MCP 应用发布时相同的加固流水线渲染（在专用沙箱来源上使用双 iframe，按小组件声明 CSP，并以失败关闭方式解码），而不是使用第二套定制 iframe 宿主。代理按值接收 HTML，因此本地内容是自然适配的场景。
- **一个授权模型。** 无论小组件属于哪种类型，其访问能力都是已授予的允许列表：对于 `html` 小组件，是宿主工具；对于 `mcp-app` 小组件，是服务器向应用公开的工具（通过现有 `allowedAppToolNames` 机制，并将其从按签发运行改为按小组件持久化）。
- **`html` 小组件的宿主工具**（通过小组件桥接公开，并根据授权进行检查）：
  - `openclaw.prompt.send` — 第 2 层级；通过可见编辑器路由，除非已授权，否则需用户确认
  - `openclaw.state.emit` — 第 1 层级会话通知（合并处理并限制大小）
  - `openclaw.data.read` — 参数化只读绑定（现有列入允许列表的读取 RPC 集），由 Gateway 网关侧解析
  - `openclaw.cron.trigger` — 第 3 层级自动化
- **`net` = CSP。** 网络访问使用已发布的按小组件 CSP 声明（`connect-src` 来源）——可自行更新的天气小组件直接从沙箱获取其 API 数据，无需 Gateway 网关参与。
- **授权。** 未声明任何内容的小组件会立即渲染（经过沙箱隔离、`default-src 'none'`、每次发送提示均单独确认）——信任级别与当前内联聊天小组件相同。声明工具/来源会使看板上的小组件进入 `pending`：占位卡片以易于理解的方式列出这些内容，并提供一键 **允许**/**拒绝**。授权按小组件名称记录；对于 `html` 小组件，授权会按字节冻结（sha256），而字节发生变化后，仅当声明范围缩小时才保留授权。
- **创作适配层。** 文档包装器会注入 `window.openclaw.prompt`、`window.openclaw.state`、`window.openclaw.data` 和 `window.openclaw.cron`，作为稳定的作者 API。仪表板调用共享一个绑定到视图票据的请求通道；尺寸报告和主题令牌仍作为独立的宿主通知。

### 插件能力声明

启用的插件可以通过 `openclaw.plugin.json` 中的 `dashboard.dataBindings` 和 `dashboard.actionVerbs` 扩展小组件宿主。插件本地 ID 会成为以插件 ID 为前缀的授权名称，例如 `workboard.cards.list` 和 `workboard.dispatch`；插件 ID 段中的 `%` 和 `.` 会被转义，以免采用不同插件/本地 ID 拆分方式的对象继承相同的持久化授权。插件注册期间，OpenClaw 会验证每个绑定都指向同一插件使用 `operator.read` 注册的 RPC，并验证每个操作都指向使用 `operator.write` 注册的 RPC；无效声明会导致插件加载失败。仅当插件生命周期发生变化时，才会重新构建经过验证的注册表；小组件授权则继续按小组件记录，并同时绑定字节和版本。

### 已建模的残余风险：WebRTC 数据通道

沙箱 CSP 会发出提议的 `webrtc 'block'` 指令，但 [Chromium 当前的 CSP 指令集](https://chromium.googlesource.com/chromium/src/+/main/services/network/public/mojom/content_security_policy.mojom#95)并未实现该指令。因此，在当前 Chromium 中，可编写脚本的小组件可以使用 WebRTC 数据通道向外传输数据。`main` 上的内联聊天小组件和 MCP Apps 宿主已经存在相同的残余风险。

**已接受的权衡：** OpenClaw 不会基于此残余风险限制可脚本化小组件。
小组件内容仅可通过操作员授予的、字节冻结的 `data:read` 能力访问
OpenClaw 敏感数据，而沙箱 Permissions Policy 会阻止摄像头和麦克风访问。
DOM API 防护属于尽力而为的纵深防御，而非安全边界，应在后续
加固中处理。

### 对话记录显示：单个小组件卡片

内联显示统一使用小组件原语。当工具结果携带 UI——
`show_widget` 输出或带有应用资源的 MCP 工具结果——时，系统会
实例化一个**临时、自动命名的小组件**（会话作用域、会被清理），
对话记录则渲染单个小组件卡片，并根据内容类型分派。
MCP 应用自动显示完全保持规范预期的行为（模型无需额外工作）；
其底层本身就是一个小组件。这会删除聊天渲染中并行的 `mcpApp`
特殊处理（界面能力限制、单独去重），为每个内联 UI 提供相同的固定操作，
并使小组件注册表成为主要的重新打开路径（对于从未固定的历史记录，
仍以扫描对话记录进行重建作为回退）。带票据的只读独立宿主与看板在
持久化重新打开界面方面存在重叠——这是 T6 中需要评估的整合候选项，
而非既定假设。

组合：v1 采用网格相邻布局（在一个标签页中，智能体外框小组件与应用
小组件相邻）。v2 增加**宿主管理的应用插槽**——智能体小组件 HTML 声明
一个插槽区域，由宿主将真实应用视图合成为同级沙箱。应用绝不会在智能体的
iframe 内渲染：嵌套会破坏桥接身份，并可能对已授权的应用 UI 实施覆盖/
点击劫持，因此插槽是一种布局契约，而不是嵌入。

### 服务端来源的小组件（已固定的 MCP 应用）

使用统一宿主后，固定第三方 MCP 应用只是创建一个从服务端获取内容而非
存储内容的小组件：`board_widgets` 保存描述符（`serverName`、
`toolName`、`uiResourceUri`、来源
`toolCallId` + `sessionKey`），而不是 HTML 字节；看板会在
聊天轮次的 10 分钟 TTL 过后重新签发视图租约（过期时重新获取
`ui://` 资源）。聊天中的内联 MCP 应用视图会获得与智能体
小组件相同的**固定到仪表板**操作。按照当前设计，重新打开的视图为只读；
需要保持交互性的已固定应用会获得对服务端应用可见工具的持久授权
（固定时向操作员显示明确的允许列表），且该授权与签发运行解耦。
未获授权的固定项仍保持只读——对于展示型仪表板依然有用。
v1 固定到来源会话的看板；跨会话固定需要租约代理，因此暂缓实现。
需与开放的 PR #109807（`ui/message` 编排器路由、主题/尺寸传播）
协调。

### WorkBoard 集成

WorkBoard 集成计划让卡片和看板继续归插件所有，同时通过现有的 `sessionKey` 和 `runId` 将已分派的卡片重新连接到其会话看板，通过插件声明的绑定和操作公开 WorkBoard 信息流及分派能力，并将这些结果与现有的 `html` 和 `mcp-app` 小组件类型组合，而不是引入 WorkBoard 专用的小组件类型。

## 布局：流式网格

12 列、固定行高、**自动压缩**（向上吸附，拖动时推开——采用
gridstack 语义，但以原生方式实现；网格计算保持纯粹且不依赖 DOM）。
每个标签页的小组件布局状态：`{ name, w (1-12), h (rows) }` 加顺序。
智能体词汇：

- `size`：`sm` (3×3) · `md` (6×4) · `lg` (8×6) · `xl` (12×8) · `full`
  （单小组件标签页）
- `after: <widgetName>` 可选的排序锚点；省略 = 追加
- 用户可自由拖动和调整尺寸；同一套顺序和尺寸模型可往返转换。

## 数据模型（每个 Agent 数据库）

在 `agents/<agentId>/agent/openclaw-agent.sqlite` 中新增表
（**需要提升 Agent 数据库架构版本——落地前必须获得操作员确认**）：

```sql
CREATE TABLE board_tabs (
  session_key TEXT NOT NULL,
  tab_id      TEXT NOT NULL,           -- slug
  title       TEXT NOT NULL,
  position    INTEGER NOT NULL,
  chat_dock   TEXT NOT NULL DEFAULT 'right',  -- left|right|bottom|hidden
  created_by  TEXT NOT NULL,           -- 'user' | 'agent'
  PRIMARY KEY (session_key, tab_id)
) STRICT;

CREATE TABLE board_widgets (
  session_key  TEXT NOT NULL,
  name         TEXT NOT NULL,          -- stable widget name
  tab_id       TEXT NOT NULL,
  title        TEXT,
  html         BLOB NOT NULL,          -- wrapped document source
  sha256       TEXT NOT NULL,
  revision     INTEGER NOT NULL,
  size_w       INTEGER NOT NULL,
  size_h       INTEGER NOT NULL,
  position     INTEGER NOT NULL,       -- order within tab (auto-compact input)
  manifest     TEXT NOT NULL DEFAULT '{}',  -- capability manifest JSON
  grant_state  TEXT NOT NULL DEFAULT 'none', -- none|pending|granted|rejected
  granted_sha  TEXT,                   -- byte-frozen grant
  created_by   TEXT NOT NULL,
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL,
  PRIMARY KEY (session_key, name)
) STRICT;
```

看板存在 = 对应 `sessionKey` 存在任意行。删除会话会删除其
看板行。`/new`/`/reset` 不会触及这些行。

## 协议接口

RPC（核心方法表，typebox 架构位于 `gateway-protocol`）：

- `board.get { sessionKey }` → 标签页 + 小组件元数据（不含字节）— `operator.read`
- `board.update { sessionKey, ops[] }` — 标签页 CRUD/重新排序、小组件移动/调整尺寸/
  删除/取消固定、停靠状态、聚焦标签页 — `operator.write`
- `board.widget.put { sessionKey, name, html, manifest, placement }` —
  `operator.write`（智能体工具路径和固定路径）
- `board.widget.grant { sessionKey, name, decision }` — `operator.approvals`
- `board.event { ticket, payload }` — 绑定票据的第 1 层状态事件摄取；
  保留旧版可信宿主的 `{ sessionKey, widget, payload }` 形态 —
  `operator.write`
- `board.prompt.authorize { ticket }` — 返回可见提示发送是否仍需
  每次点击确认 — `operator.read`
- `board.data.read { ticket, bindingId, params? }` — Gateway 网关侧允许列表中的
  核心或活动插件只读绑定解析 — `operator.read`
- `board.action { ticket, action, ... }` — 通过现有定时任务立即运行路径
  或活动插件经过验证的操作动词执行精确授权的自动化分派 —
  `operator.write`

事件（位于 `EVENT_SCOPE_GUARDS`，只读权限范围）：

- `board.changed { sessionKey, revision, widget? }` — 持久化状态已更改；
  UI 重新获取数据（存在 `widget` 时还会重新加载一个 iframe）。
- `board.command { sessionKey, command }` — 临时 UI 驱动（智能体切换
  可见标签页、切换聊天停靠栏）— 采用 `ui.command` 模式。

小组件字节通过经过身份验证的 HTTP 接口提供，而不是通过套接字。

## 智能体工具

共三个工具（核心工具，始终注册；渲染仍与当前一样受
`inline-widgets` 客户端能力限制）：

- `show_widget { title, widget_code, name?, pin?, size?, tab?, after?,
capabilities? }` — 按名称创建/更新；`pin` 将其放置在看板上。
  不带 `name`/`pin` 时，其行为与当前完全一致（内联、临时）。
- `dashboard { action, ... }` — 看板管理动词：`read`、`tab_create`、
  `tab_update`、`tab_delete`、`tabs_reorder`、`widget_move`、`widget_remove`、
  `unpin`、`focus_tab`、`set_chat_dock`。
- 现有 `cron` 工具已覆盖自动化层；无需新增工具。

工具描述会说明尺寸/锚点词汇和分层模型。智能体会通过会话通知获知用户的
第 1 层事件，例如 `[dashboard] user clicked "Refresh" on widget weather (tab main)`。

## 此方案替代的内容

- **删除 `extensions/workspaces`。** 这是实验性功能，`enabledByDefault:
false`，从未进入稳定版本（首次出现在 2026.7.2 测试版中）。不做
  迁移；Doctor 规则会删除存在的陈旧 `<stateDir>/workspaces/`。
  沿用的设计：纯网格计算、桥接安全模型（端口引导、
  绑定限制、速率限制）、字节冻结审批。
- **小组件托管从 `extensions/canvas` 移至核心。** Canvas 文档
  存储、文档包装器、HTTP 服务和 `show_widget` 工具转为核心功能
  （`src/canvas/`）；插件保留节点 Canvas 控制工具（`canvas`）和
  A2UI。`pluginSurfaceUrls["canvas"]` 公告和
  `/__openclaw__/canvas` 路径属于已发布的原生客户端契约，保持
  稳定。Discord 会话继续使用 Discord 所有的 `show_widget` 变体。

## 非目标（本计划）

- 多用户看板共享/ACL（未来功能；将通过会话共享提供）。
- 原生 macOS/iOS 看板渲染（只要嵌入 Control UI 即可获得；
  内联小组件路径保持不变）。
- 内置数据小组件（会话/用量/定时任务卡片）——能力桥接加上
  智能体编写的小组件足以覆盖 v1；之后可以增加内置类型注册表。

## 实施计划

使用独立工作树，由 Codex 构建，依次审查并落地。先落地，再修复。

| #   | 分支                               | 范围                                                                                                                                                                              | 依赖                       |
| --- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| T1  | `claude/dashboard-remove-workspaces` | 删除工作区插件 + UI + 文档 + i18n 键；Doctor 清理规则                                                                                                              | —                                |
| T2  | `claude/dashboard-canvas-core`       | 将小组件托管 + `show_widget` 提升至核心；Canvas 插件保留节点工具；行为零变化                                                                                | —                                |
| T3  | `claude/dashboard-domain`            | Agent 数据库表（架构版本提升）、`board.*` RPC + 事件、`dashboard` 工具、`show_widget` 固定/名称/清单参数、第 1 层通知、重置时保留看板                                  | T2                               |
| T4  | `claude/dashboard-ui`                | 看板界面 + 标签栏 + 流式自动压缩网格 + 聊天停靠栏（左/右/底部/隐藏）+ 对话记录固定操作 + 侧边栏看板界面 + 重置确认                           | T3（通过开发夹具先使用模拟） |
| T5  | `claude/dashboard-capabilities`      | 授权存储/UI + 字节冻结；将 `html` 小组件迁移到共享沙箱宿主；宿主工具（`openclaw.prompt.send/state.emit/data.read/cron.trigger`）；`net` CSP；编写兼容层 | T3、T4                           |
| T7  | `claude/dashboard-mcp-apps`          | `mcp-app` 内容类型：内联应用视图上的固定操作、描述符存储、租约重新签发/刷新、持久服务端工具授权（复用已发布的 MCP Apps 宿主）                   | T3、T4                           |
| T6  | 完善                               | 在临时 Gateway 网关上执行实时 E2E（真实密钥）、截图、修复、以用户为中心重写 `/web/dashboard`、审查是否默认启用                                                     | 全部                              |

按仓库规则验证：在本地运行聚焦的 vitest，在 Crabbox/Testbox 上运行
完整检查，每次落地前执行 `$autoreview`，并为 T6 提供实时验证。
