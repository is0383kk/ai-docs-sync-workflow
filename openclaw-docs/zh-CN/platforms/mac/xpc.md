---
read_when:
    - 编辑 IPC 契约或菜单栏应用 IPC
summary: OpenClaw 应用、Gateway 网关节点传输和 PeekabooBridge 的 macOS IPC 架构
title: macOS 进程间通信
x-i18n:
    generated_at: "2026-07-26T06:50:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39e11af2bb9348d1c1f6e4fe6be95e825d23d5c1aa66e32dae713a89afb12b4f
    source_path: platforms/mac/xpc.md
    workflow: 16
---

# OpenClaw macOS IPC 架构

本地 Unix 套接字将节点主机服务连接到 macOS 应用，用于 Exec 审批和 `system.run`。另有一个用于设备发现/连接检查的 `openclaw-mac` 调试 CLI（`apps/macos/Sources/OpenClawMacCLI`）；智能体操作仍通过 Gateway 网关 WebSocket 和 `node.invoke` 传递。由节点支持的 `computer.act` 路径会在进程内运行嵌入式 Peekaboo 自动化；独立的 Peekaboo 客户端则使用 PeekabooBridge。

## 目标

- 由单个 GUI 应用实例负责所有面向 TCC 的工作（通知、屏幕录制、麦克风、语音、AppleScript）。
- 提供精简的自动化接口：Gateway 网关 + 节点命令、进程内 `computer.act`，以及供独立 UI 自动化客户端使用的 PeekabooBridge。
- 可预测的权限：始终使用同一个签名 bundle ID 并由 launchd 启动，从而使 TCC 授权持续有效。

## 工作原理

### Gateway 网关 + 节点传输

- 应用运行 Gateway 网关（本地模式），并作为节点连接到该网关。
- 智能体操作通过 `node.invoke` 执行（例如 `system.run`、`system.notify`、`canvas.*`）。
- 节点命令包括 `canvas.*`、`camera.snap`、`camera.clip`、`screen.snapshot`、`screen.record`、`computer.act`、`system.run` 和 `system.notify`。
- 节点会报告一个 `permissions` 映射，以便智能体了解屏幕、摄像头、麦克风、语音、自动化或辅助功能访问是否可用。

### 节点服务 + 应用 IPC

- 无头节点主机服务连接到 Gateway 网关 WebSocket。
- `system.run` 请求通过本地 Unix 套接字（`ExecApprovalsSocket.swift`）转发到 macOS 应用。
- 应用在 UI 上下文中执行 Exec，必要时提示用户，并返回输出。

图示（SCI）：

```text
智能体 -> Gateway 网关 -> 节点服务（WS）
                             |  IPC（UDS + 令牌 + HMAC + TTL）
                             v
                         Mac 应用（UI + TCC + system.run）
```

### PeekabooBridge（UI 自动化）

- 内置的智能体 `computer` 工具**不**使用此套接字。已配对的 macOS 节点通过嵌入式 Peekaboo 服务在应用进程中执行 `computer.act`。
- UI 自动化使用单独的 UNIX 套接字（`~/Library/Application Support/OpenClaw/<socket>`）和 PeekabooBridge JSON 协议。
- 主机优先顺序（客户端侧）：Peekaboo.app -> Claude.app -> OpenClaw.app -> 本地执行。
- 安全性：桥接主机必须具有允许列表中的 TeamID（内置的 `PeekabooBridgeHostCoordinator` 允许一个固定团队以及应用自身的签名团队）；仅限 DEBUG 的同 UID 例外通道由 `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` 保护（Peekaboo 约定）。
- 详情请参阅：[PeekabooBridge 用法](/zh-CN/platforms/mac/peekaboo)。

## 操作流程

- 重启/重新构建：`scripts/restart-mac.sh` 会终止现有实例、通过 Swift 重新构建、重新打包并重新启动。它会自动检测可用的签名身份，如果未找到则回退到 `--no-sign`；传入 `--sign` 可强制要求签名（没有可用密钥时失败），或传入 `--no-sign` 强制使用未签名路径。在签名路径上，环境中设置的 `SIGN_IDENTITY` 会被取消设置，以便 `scripts/codesign-mac-app.sh` 自身的身份自动检测选择证书。
- 单实例：应用通过 `NSWorkspace.runningApplications` 检查是否存在相同 bundle ID 的重复实例；如果发现多个实例，则退出（`MenuBar.swift` 中的 `isDuplicateInstance()`）。

## 加固说明

- 对于所有特权接口，建议要求 TeamID 匹配。
- PeekabooBridge：`PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1`（仅限 DEBUG）可允许同 UID 调用方进行本地开发。
- 所有通信均仅限本地；不会暴露任何网络套接字。
- TCC 提示仅由 GUI 应用 bundle 发起；在重新构建过程中应保持已签名 bundle ID 稳定。
- Exec 审批套接字加固：文件模式 `0600`、共享令牌、对端 UID 检查（`getpeereid`）、HMAC-SHA256 质询/响应，以及较短的请求 TTL。

## 相关内容

- [macOS 应用](/zh-CN/platforms/macos)
- [macOS IPC 流程（Exec 审批）](/zh-CN/tools/exec-approvals-advanced#macos-ipc-flow)
