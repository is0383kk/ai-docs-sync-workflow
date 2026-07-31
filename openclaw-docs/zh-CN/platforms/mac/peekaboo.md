---
read_when:
    - 在 OpenClaw.app 中托管 PeekabooBridge
    - 通过 Swift Package Manager 集成 Peekaboo
    - 更改 PeekabooBridge 协议/路径
    - 在 PeekabooBridge、Codex Computer Use 和 cua-driver MCP 之间进行选择
summary: 用于 macOS UI 自动化的 PeekabooBridge 集成
title: Peekaboo 桥接器
x-i18n:
    generated_at: "2026-07-26T06:48:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 24d4187b2f5c5f11f44a24e25b350adaa3b068f24dce640ec695d52eb61f8e9a
    source_path: platforms/mac/peekaboo.md
    workflow: 16
---

OpenClaw 可将 **PeekabooBridge** 作为本地、权限感知的 UI 自动化代理托管（`PeekabooBridgeHostCoordinator`，由 `steipete/Peekaboo` Swift 软件包提供支持）。这样，`peekaboo` CLI 就能在复用 macOS 应用的 TCC 权限的同时驱动 UI 自动化。

## 这是什么（以及不是什么）

- **主机**：OpenClaw.app 可充当 PeekabooBridge 主机。
- **客户端**：`peekaboo` CLI（没有单独的 `openclaw ui ...` 界面）。
- **UI**：视觉叠加层保留在 Peekaboo.app 中；OpenClaw 只是一个轻量代理主机。

## 与其他桌面控制路径的关系

OpenClaw 有四条有意保持独立的桌面控制路径：

- **PeekabooBridge 主机**：OpenClaw.app 托管本地 PeekabooBridge 套接字。`peekaboo` CLI 是客户端，它使用 OpenClaw.app 的 macOS 权限执行屏幕截图、点击、菜单、对话框、Dock 操作和窗口管理。
- **智能体驱动的计算机使用（`computer.act`）**：Gateway 网关智能体的内置 `computer` 工具通过 `screen.snapshot` 捕获屏幕截图，并通过危险的 `computer.act` 节点命令控制指针和键盘。macOS 节点使用此桥接所公开的嵌入式 Peekaboo 自动化服务以及有限的 CoreGraphics 原语，在进程内执行 `computer.act`，无需经过 PeekabooBridge 套接字或 `peekaboo` CLI。请参阅[计算机使用](/zh-CN/nodes/computer-use)。
- **Codex Computer Use**：内置的 `codex` 插件会检查 Codex 的 `computer-use` MCP 插件（`extensions/codex/src/app-server/computer-use.ts`），并可安装该插件；随后，在 Codex 模式轮次中，由 Codex 负责原生桌面控制工具调用。OpenClaw 不会通过 PeekabooBridge 代理这些操作。
- **直接使用 `cua-driver` MCP**：OpenClaw 可将 TryCua 的上游 `cua-driver mcp` 服务器注册为普通 MCP 服务器，使智能体能够使用 CUA 驱动程序自身的架构以及 pid/窗口/元素索引工作流，而无需经由 Codex 市场或 PeekabooBridge 套接字进行路由。

如需通过 OpenClaw.app 的权限感知桥接主机使用广泛的 macOS 自动化功能，请使用 Peekaboo。如需让 Gateway 网关智能体通过任何视觉模型都能驱动的统一 `computer.act` 节点命令查看并控制桌面，请使用智能体驱动的计算机使用。如需让 Codex 模式智能体依赖 Codex 的原生插件，请使用 Codex Computer Use。如需将 CUA 驱动程序作为普通 MCP 服务器公开给任何由 OpenClaw 管理的运行时，请直接使用 `cua-driver mcp`。

## 启用桥接

在 macOS 应用中：**Settings -> Enable Peekaboo Bridge**。由于两者都会授予本地 UI 自动化能力，因此仅当 **Allow Computer Control** 已开启时才能使用此开关；Computer Control 关闭时，该开关会被禁用，主机也不会运行。若要在不启用 Computer Control 的情况下驱动 Peekaboo，请改为运行 Peekaboo 自己的 Mac 应用作为主机。

启用后（且 Computer Control 已开启），OpenClaw 会在 `~/Library/Application Support/OpenClaw/<socket-name>` 启动本地 UNIX 套接字服务器。如果禁用，主机会停止，`peekaboo` 将回退到其他可用主机。协调器还会维护旧版套接字符号链接（Application Support 下的 `clawdbot`、`clawdis`、`moltbot`），这些链接指向当前套接字，以支持较旧的 `peekaboo` 安装。

## 客户端发现顺序

Peekaboo 客户端通常按以下顺序尝试主机：

1. Peekaboo.app（完整用户体验）
2. Claude.app（如果已安装）
3. OpenClaw.app（轻量代理）

使用 `peekaboo bridge status --verbose` 查看当前活跃的主机及其使用的套接字路径。可通过以下方式覆盖：

```bash
export PEEKABOO_BRIDGE_SOCKET=/path/to/bridge.sock
```

## 安全与权限

- 桥接会验证**调用方代码签名**；系统强制执行 TeamID 允许列表（Peekaboo 主机 TeamID 加上正在运行的应用自身的 TeamID）。
- 对于辅助功能权限，应优先使用已签名的桥接/应用身份，而不是通用的 `node` 运行时。向 `node` 授予辅助功能权限，会使该 Node 可执行文件启动的任何软件包继承 GUI 自动化访问权限；请参阅 [macOS 权限](/zh-CN/platforms/mac/permissions#accessibility-grants-for-node-and-cli-runtimes)。
- 请求将在 10 秒后超时（`requestTimeoutSec: 10`）。
- 如果缺少所需权限，桥接会返回清晰的错误消息，而不是启动 System Settings。

## 快照行为（自动化）

快照存储在内存中，有效期为 10 分钟，上限为 50 个快照（`InMemorySnapshotManager`）；清理时不会删除工件。如果需要保留更长时间，请从客户端重新捕获。

## 故障排查

- 如果 `peekaboo` 报告“bridge client is not authorized”，请确保客户端已正确签名，或仅在 **debug** 模式下使用 `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` 运行主机。
- 如果找不到主机，请打开其中一个主机应用（Peekaboo.app 或 OpenClaw.app），并确认已授予权限。

## 相关内容

- [macOS 应用](/zh-CN/platforms/macos)
- [macOS 权限](/zh-CN/platforms/mac/permissions)
