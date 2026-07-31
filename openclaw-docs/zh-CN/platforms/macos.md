---
read_when:
    - 安装 macOS 应用
    - 在 macOS 上选择本地或远程 Gateway 网关模式
    - 正在查找 macOS 应用发布版下载项
summary: 安装并使用 OpenClaw macOS 菜单栏应用
title: macOS 应用
x-i18n:
    generated_at: "2026-07-26T06:53:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b319d72bcbffcf91b6bc012d352c2cf647abd66e08ab0146cf98f5edfae3bca1
    source_path: platforms/macos.md
    workflow: 16
---

macOS 应用是 OpenClaw 的**菜单栏配套应用**：提供原生托盘 UI、macOS
权限提示、通知、WebChat、语音输入、Canvas，以及
由 Mac 托管的节点工具，例如 `system.run`。

使用**快速聊天**可在不打开完整窗口的情况下，通过类似 Spotlight 的主会话输入框编写消息。默认按 Option-Space（⌥Space），也可以从菜单栏菜单中选择，或在 **Settings → General** 中录制其他快捷键。

只需要 CLI 和 Gateway 网关？请从[入门指南](/zh-CN/start/getting-started)开始。

## 下载

从 [OpenClaw GitHub 发布页面](https://github.com/openclaw/openclaw/releases)获取 macOS 应用构建版本。
当某个发布版本包含 macOS 应用资产时，请查找：

- `OpenClaw-<version>.dmg`（首选）
- `OpenClaw-<version>.zip`

有些发布版本仅包含 CLI、证据或 Windows 资产。如果最新发布版本
没有 macOS 应用资产，请使用最新的含有此资产的版本，或按照
[macOS 开发设置](/zh-CN/platforms/mac/dev-setup)从源代码构建。

## 首次运行

1. 安装并启动 **OpenClaw.app**。
2. 选择 **This Mac** 以使用本地 Gateway 网关，或连接到远程 Gateway 网关。
3. 等待应用安装匹配的 CLI 运行时。在本地模式下，它还会
   安装并启动 Gateway 网关。
4. 通过实时模型检查建立推理连接。检查通过后，OpenClaw
   会处理其余设置。
5. 完成 macOS 权限检查清单并发送新手引导测试消息。

如果应用连接到现有 Gateway 网关，且其默认智能体已配置
模型，则会将该 Gateway 网关视为已设置，跳过提供商新手引导和
OpenClaw，并打开仪表板。如果无法连接 Gateway 网关，或其
默认智能体未配置模型，推理新手引导仍可用于
恢复。

对于 CLI/Gateway 网关设置路径，请使用[入门指南](/zh-CN/start/getting-started)。
对于权限恢复，请使用 [macOS 权限](/zh-CN/platforms/mac/permissions)。

## 更新

仪表板更新卡片会注明应用将更新的内容：

- **更新 Mac 应用 + Gateway 网关**表示已签名应用拥有本地 launchd
  Gateway 网关。Sparkle 会先更新应用；应用重新启动后，会自动
  将其 Gateway 网关更新至匹配版本并重启，然后验证
  连接。
- **更新 Gateway 网关**表示应用已连接到远程 Gateway 网关、手动
  管理的本地 Gateway 网关，或应用不拥有的其他安装。该按钮
  会运行相应 Gateway 网关的常规更新流程，而不会更改 Mac 应用。

协调更新失败后，应用会停留在设置样式窗口中，并提供重试、
[更新指南](/zh-CN/install/updating)和 Discord 操作。自动修复绝不会
降级较新的 Gateway 网关，也不会覆盖 `extended-stable` 渠道固定版本。

成功更新后，应用会查找最近由用户使用的
顶层直接会话，并向该智能体发送一次性更新事件。Heartbeat
和定时任务活动不会影响此选择。然后，该智能体可以从
你最可能正在使用的对话中欢迎你回来。在远程模式下，应用
仅更新本地 Mac 节点运行时；当远程 Gateway 网关版本早于应用时，
会跳过通知。

Sparkle 遵循 Gateway 网关的 `update.channel` 设置。`beta` 和 `dev` 会选择加入
测试版应用构建；`stable`、`extended-stable` 以及缺失或未知的值
均继续使用稳定版应用构建。

## 打开仪表板链接

在 macOS 应用的嵌入式仪表板中，点击外部网页链接会在窗口一半宽度的可调整浏览器侧边栏中打开，同时保持仪表板导航可见。拖动分隔线可选择其他宽度；应用会记住该宽度。每个链接会在各自的标签页中打开，打开多个页面时会显示标签栏；再次点击同一链接会复用其现有标签页。拖动标签页可重新排序，使用标签页关闭按钮或鼠标中键点击可将其关闭；右键点击标签页可使用 **Open in Default Browser**、**Copy Link**、**Reload**、**Close Tab** 和 **Close Other Tabs**。窗口标题栏的后退/前进控件和触控板轻扫用于浏览仪表板历史记录；侧边栏自己的后退/前进控件用于浏览当前标签页的历史记录。侧边栏还提供重新加载、在默认浏览器中打开和关闭控件。

标题栏控件会随应用侧边栏变化：侧边栏展开时，后退/前进控件位于其右边缘、侧边栏切换按钮旁；侧边栏折叠时，它们会让位于搜索按钮（打开命令面板）和新建会话按钮。

右键点击外部链接可选择 **Open in Sidebar**、**Open in Default Browser** 或 **Copy Link**。从仪表板发起的组合键点击和由用户触发的新窗口链接仍会在默认浏览器中打开；侧边栏内的新窗口链接则作为新的侧边栏标签页打开。由常规浏览器托管的 Control UI 页面仍使用浏览器正常的链接和上下文菜单行为。

## 导入浏览器登录信息

当应用连接本地 Gateway 网关运行时，首次打开浏览器侧边栏时，如果 Mac 上存在带 Cookie 的 Chrome 系浏览器配置文件，仪表板会显示一条可关闭的横幅。该横幅允许将这些 Cookie 复制到智能体浏览时使用的隔离托管配置文件中。从其 **Import** 控件中选择一个配置文件（可能需要 Touch ID）；进度和已导入的 Cookie 数量会内联显示，并且只复制 Cookie——密码绝不会离开源浏览器。关闭横幅会记录此选择；随时可通过 **Settings → General → Browser login → Import…** 再次显示。底层导入流程及 `browser.allowSystemProfileImport` 门控请参阅[浏览器](/zh-CN/cli/browser)。

## 选择 Gateway 网关模式

| 模式   | 适用情形                                                                       | 详情页面                                           |
| ------ | ------------------------------------------------------------------------------ | -------------------------------------------------- |
| 本地   | 由这台 Mac 运行 Gateway 网关，并通过 launchd 保持其运行。                      | [macOS 上的 Gateway 网关](/zh-CN/platforms/mac/bundled-gateway) |
| 远程   | 由另一台主机运行 Gateway 网关；这台 Mac 通过 SSH、LAN 或 Tailnet 控制它。       | [远程控制](/zh-CN/platforms/mac/remote)                  |

两种模式都需要安装 `openclaw` CLI，因为应用会复用其节点主机
运行时。在全新的 Mac 上，应用会自动安装匹配的 CLI；随后，本地
模式会启动 Gateway 网关向导，而远程模式会连接到所选的
Gateway 网关，而不会启动第二个本地 Gateway 网关。
有关手动恢复，请参阅 [macOS 上的 Gateway 网关](/zh-CN/platforms/mac/bundled-gateway)。

## 应用负责的内容

- 菜单栏状态、通知、健康状况、WebChat 和浮动快速聊天栏。
- 针对屏幕、麦克风、语音、自动化和辅助功能的 macOS 权限提示。
- 一个 Mac 节点，它将原生 Canvas、摄像头/屏幕捕获、通知、
  位置和计算机控制，与 CLI 节点主机的系统、浏览器、
  插件、Skills 和 MCP 命令相结合。
- Mac 托管命令的 Exec 审批提示。
- 在应用上下文中执行已批准的 shell 命令，在由 CLI 运行时负责共享节点策略的同时，保留应用的 macOS
  权限归属。
- 远程模式 SSH 隧道或直接 Gateway 网关连接。

在嵌入式 Control UI 中，**Settings → Notifications** 显示应用的原生
通知权限，而不是浏览器推送权限，因为应用以原生方式发送通知。

应用**不会**取代 Gateway 网关或常规 CLI 文档。Gateway 网关
配置、提供商、插件、渠道、工具和安全性均在各自的
文档中说明。

## macOS 详情页面

| 任务                                     | 阅读                                                                                        |
| ---------------------------------------- | ------------------------------------------------------------------------------------------- |
| 安装或调试 CLI/Gateway 网关服务          | [macOS 上的 Gateway 网关](/zh-CN/platforms/mac/bundled-gateway)                                   |
| 避免将状态存入云同步文件夹               | [macOS 上的 Gateway 网关](/zh-CN/platforms/mac/bundled-gateway#state-directory-on-macos)          |
| 调试应用发现和连接                       | [macOS 上的 Gateway 网关](/zh-CN/platforms/mac/bundled-gateway#debug-app-connectivity)            |
| 了解 launchd 行为                        | [Gateway 网关生命周期](/zh-CN/platforms/mac/child-process)                                        |
| 修复权限或签名/TCC 问题                  | [macOS 权限](/zh-CN/platforms/mac/permissions)                                                    |
| 检测你最近使用的 Mac                     | [活动计算机在线状态](/zh-CN/nodes/presence)                                                       |
| 连接到远程 Gateway 网关                  | [远程控制](/zh-CN/platforms/mac/remote)                                                           |
| 查看菜单栏状态和健康检查                 | [菜单栏](/zh-CN/platforms/mac/menu-bar)、[健康检查](/zh-CN/platforms/mac/health)                        |
| 使用嵌入式聊天 UI                        | [WebChat](/zh-CN/platforms/mac/webchat)                                                           |
| 使用语音唤醒或按键说话                   | [语音唤醒](/zh-CN/platforms/mac/voicewake)                                                        |
| 使用 Canvas 和 Canvas 深层链接           | [Canvas](/zh-CN/platforms/mac/canvas)                                                             |
| 托管 PeekabooBridge 以进行 UI 自动化     | [Peekaboo bridge](/zh-CN/platforms/mac/peekaboo)                                                  |
| 配置命令审批                             | [Exec 审批](/zh-CN/tools/exec-approvals)、[高级详情](/zh-CN/tools/exec-approvals-advanced)              |
| 检查 Mac 节点命令和应用 IPC              | [macOS IPC](/zh-CN/platforms/mac/xpc)                                                             |
| 捕获日志                                 | [macOS 日志](/zh-CN/platforms/mac/logging)                                                        |
| 从源代码构建                             | [macOS 开发设置](/zh-CN/platforms/mac/dev-setup)                                                  |

## 相关内容

- [平台](/zh-CN/platforms)
- [入门指南](/zh-CN/start/getting-started)
- [Gateway 网关](/zh-CN/gateway)
- [Exec 审批](/zh-CN/tools/exec-approvals)
