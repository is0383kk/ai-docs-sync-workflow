---
read_when:
    - 运行或调试 Gateway 网关进程
    - 调查单实例强制执行机制
summary: Gateway 网关单例保护：文件锁加 WebSocket/HTTP 绑定
title: Gateway 锁定
x-i18n:
    generated_at: "2026-07-26T06:14:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f5ac6d42c437b481c68a23a0aa4c00aeac9131acd76f3516ce3e949f325e265b
    source_path: gateway/gateway-lock.md
    workflow: 16
---

## 原因

- 一个状态目录只能由一个 Gateway 网关进程占用；运行其他 Gateway 网关时，请使用相互隔离的配置文件、状态目录、配置和端口。
- 即使发生崩溃或 SIGKILL，也不会留下过期的锁文件。
- 当另一个 Gateway 网关已占用端口时，立即失败并给出明确错误。

## 三层机制

启动过程按顺序通过三个步骤强制执行所有权：

1. **状态所有权锁**获取一个以规范状态目录为键的锁。每个 Gateway 网关都会参与，包括使用 `OPENCLAW_ALLOW_MULTI_GATEWAY=1` 启动的 Gateway 网关，因此破坏性的 SQLite 维护操作不会与仍在运行的所有者发生竞态。
2. **配置锁**获取历史沿用的每配置锁，并记录运行时端口。多 Gateway 网关模式会跳过此配置单例锁，但保留状态所有权锁。
3. **套接字绑定**将 HTTP/WebSocket 监听器（默认 `ws://127.0.0.1:18789`）绑定为独占 TCP 监听器。

每一层都可能独立失败，并抛出各自的 `GatewayLockError`。

### 状态锁和配置锁

- 锁的存活状态由记录的 PID、可用时的平台进程启动标识和 Gateway 网关进程标识共同确定。在端口开始监听之前的启动阶段，经过验证的所有者仍具有权威性。
- 专用 SQLite 协调器会串行执行元数据检查、过期所有者回收和锁替换。如果所属进程崩溃，其独占事务会自动释放。
- 如果锁文件缺失或记录的所有者进程已不存在，启动过程会回收该锁并继续。
- 如果任一锁正被占用，启动过程会重试最长 5 秒（默认值），之后放弃：

  ```text
  GatewayLockError("Gateway 网关已在运行（pid <pid>）；锁在 <ms>ms 后超时")
  ```

### 套接字绑定

- 遇到 `EADDRINUSE` 时，启动过程会以 500ms 的间隔重试绑定，最多 20 次（总计约 10 秒），以度过进程刚退出后可能出现的 `TIME_WAIT` 时间窗口。
- 如果重试后端口仍在使用中：

  ```text
  GatewayLockError("另一个 Gateway 网关实例已在 ws://127.0.0.1:<port> 上监听")
  ```

- 其他绑定失败：

  ```text
  GatewayLockError("无法在 ws://127.0.0.1:<port> 上绑定 Gateway 网关套接字：<cause>")
  ```

关闭时，Gateway 网关会关闭 HTTP/WebSocket 服务器，并删除其状态锁文件
和配置锁文件。

## 运维说明

- 如果端口被另一个非 Gateway 网关进程占用，错误相同；请释放该端口，或通过 `openclaw gateway --port <port>` 选择其他端口。
- `OPENCLAW_ALLOW_MULTI_GATEWAY=1` 允许多个配置/运行时实例，但不允许共享可变状态。每个实例仍需使用唯一的 `OPENCLAW_STATE_DIR`。
- 在服务监督程序下，如果新的 Gateway 网关进程遇到上述任一错误，它会先探测现有进程上的 `/healthz`。如果该进程健康，新进程会让其继续控制，而不是失败。在 systemd 上，新进程会以代码 `78` 退出；单元的 `RestartPreventExitStatus=78` 会阻止 `Restart=always` 因锁冲突或 `EADDRINUSE` 冲突而循环。如果现有进程始终未恢复健康，健康探测重试会在限定时间内结束，随后启动过程将以上述锁错误失败，而不会无限循环。
- macOS 应用在生成 Gateway 网关进程前会保留自身的轻量级 PID 防护；上述文件锁和套接字绑定才是实际的运行时强制机制。

## 相关内容

- [多个 Gateway 网关](/zh-CN/gateway/multiple-gateways) - 使用唯一端口运行多个实例
- [故障排查](/zh-CN/gateway/troubleshooting) - 诊断 `EADDRINUSE` 和端口冲突
