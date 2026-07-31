---
read_when:
    - 添加或修改后台 Exec 行为
    - 调试长时间运行的 Exec 任务
summary: 后台 Exec 执行和进程管理
title: 后台 Exec 和进程工具
x-i18n:
    generated_at: "2026-07-26T06:47:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 37cb65ddf67227e32be972e77d16b9835d592120ecd12e041d05c48536fd2204
    source_path: gateway/background-process.md
    workflow: 16
---

OpenClaw 通过 `exec` 工具运行 shell 命令，并将长时间运行的任务保存在内存中。`process` 工具用于管理这些后台会话。

## Exec 工具

参数：

| 参数    | 说明                                                                                                                                                |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`    | 必填。要运行的 shell 命令。                                                                                                                            |
| `workdir`    | 工作目录；省略时使用默认 cwd。                                                                                                            |
| `env`        | 命令所需的额外环境变量。                                                                                                               |
| `yieldMs`    | 转入后台前等待的毫秒数（默认 10000）。                                                                                                 |
| `background` | 立即在后台运行。                                                                                                                             |
| `timeout`    | 超时时间（秒，默认 `tools.exec.timeoutSeconds`）；到期时终止进程。设为 `timeout: 0` 可为该次调用禁用 Exec 进程超时。 |
| `pty`        | 可用时在伪终端中运行（需要 TTY 的 CLI、编码智能体）。                                                                                |
| `elevated`   | 如果提升权限模式已启用/允许，则在沙箱外运行（默认为 `gateway`；当 Exec 目标为 `node` 时为 `node`）。                              |
| `host`       | Exec 目标：`auto`、`sandbox`、`gateway` 或 `node`。                                                                                                      |
| `node`       | 节点 ID/名称，与 `host: "node"` 配合使用。                                                                                                                    |

行为：

- 前台运行会直接返回输出。
- 转入后台时（显式指定或因 `yieldMs` 超时），工具会返回 `status: "running"` + `sessionId` 以及一小段输出末尾内容。
- 后台运行和 `yieldMs` 运行会继承 `tools.exec.timeoutSeconds`，除非调用显式传入 `timeout`。
- 输出会保留在内存中，直到轮询或清除该会话。
- 如果不允许使用 `process` 工具，`exec` 会同步运行并忽略 `yieldMs`/`background`。
- 生成的 Exec 命令会接收 `OPENCLAW_SHELL=exec`，用于支持上下文感知的 shell/配置文件规则。
- 对于现在开始的长时间运行任务：只启动一次，并在命令产生输出或失败后依赖自动完成唤醒（启用时）。
- 如果自动完成唤醒不可用，或者需要确认一个无输出且正常退出的命令是否静默成功，请使用 `process` 进行轮询。
- 不要使用 `sleep` 循环或重复轮询来模拟提醒或延迟跟进——未来要执行的工作请使用 cron。

### 环境变量覆盖项

| 变量                                 | 作用                                                                                                           |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_BASH_YIELD_MS`                 | 转入后台前的默认让出时间（毫秒）。默认 10000，限制在 10-120000。                                       |
| `OPENCLAW_BASH_MAX_OUTPUT_CHARS`         | 内存中输出的字符数上限。                                                                                    |
| `OPENCLAW_BASH_PENDING_MAX_OUTPUT_CHARS` | 每个流的待处理 stdout/stderr 字符数上限。                                                                    |
| `OPENCLAW_BASH_JOB_TTL_MS`               | 已完成会话的 TTL（毫秒），限制在 1m-3h。                                                                |
| `OPENCLAW_PROCESS_INPUT_WAIT_IDLE_MS`    | 可写后台会话被标记为可能正在等待输入之前的输出空闲阈值。默认 15000。 |

### 配置（优先于环境变量覆盖项）

| 键                                   | 默认值 | 作用                                                                          |
| ------------------------------------- | ------- | ------------------------------------------------------------------------------- |
| `tools.exec.backgroundMs`             | 10000   | 与 `OPENCLAW_BASH_YIELD_MS` 相同。                                               |
| `tools.exec.timeoutSeconds`           | 1800    | 每次调用的默认超时时间。                                                       |
| `tools.exec.cleanupMs`                | 1800000 | 与 `OPENCLAW_BASH_JOB_TTL_MS` 相同。                                             |
| `tools.exec.notifyOnExit`             | true    | 后台 Exec 退出时，将系统事件加入队列并请求 Heartbeat。      |
| `tools.exec.notifyOnExitEmptySuccess` | false   | 对无输出且成功完成的后台运行也将完成事件加入队列。 |

## 子进程桥接

在 Exec/Process 工具之外生成长时间运行的子进程时（CLI 重新生成、Gateway 网关辅助程序），请附加子进程桥接辅助程序，以便转发终止信号，并在退出/出错时移除监听器。这样可以避免 systemd 上出现孤立进程，并使各平台的关闭行为保持一致。

## Process 工具

操作：

| 操作      | 作用                                                                        |
| ----------- | ----------------------------------------------------------------------------- |
| `list`      | 运行中和已完成的会话。                                                  |
| `poll`      | 提取会话的新输出（也会报告退出状态）。                    |
| `log`       | 读取聚合输出和输入恢复提示。支持 `offset` + `limit`。 |
| `write`     | 发送 stdin（`data`，可选 `eof`）。                                          |
| `send-keys` | 向基于 PTY 的会话发送显式按键标记或字节。                    |
| `submit`    | 向基于 PTY 的会话发送 Enter/回车。                           |
| `paste`     | 发送原样文本，可选择使用括号粘贴模式包装。                |
| `kill`      | 终止后台会话。                                               |
| `clear`     | 从内存中移除已完成的会话。                                        |
| `remove`    | 如果正在运行则终止，否则在已完成时清除。                                 |

说明：

- 仅列出/保留后台会话——只保存在内存中，不会写入磁盘。进程重启时会话将丢失。
- 在进程所有者确认后台会话已实际退出之前，仍在运行的后台会话会阻止协作式主机挂起和安全重启 Gateway 网关。
- `process remove` 可以在请求终止后立即隐藏正在运行的会话；在确认退出之前，挂起和重启仍会被阻止。
- 只有运行 `process poll`/`log` 且工具结果被记录时，会话日志才会保存到聊天历史记录中。
- `process` 的作用域按智能体划分；它只能看到由该智能体启动的会话。
- 当自动完成唤醒不可用时，使用 `poll`/`log` 获取状态、日志或完成确认。
- 恢复交互式 CLI 前，请使用 `log`，以便同时查看当前记录、stdin 状态和输入等待提示。
- 需要输入或干预时，请使用 `write`/`send-keys`/`submit`/`paste`/`kill`。
- `process list` 包含派生的 `name`（命令动词 + 目标），便于快速浏览。
- 只有当会话仍有可写的 stdin，并且空闲时间超过输入等待阈值（默认 15000 ms，`OPENCLAW_PROCESS_INPUT_WAIT_IDLE_MS`）时，`process list`、`poll` 和 `log` 才会报告 `waitingForInput`。
- `process log` 使用基于行的 `offset`/`limit`。两者都省略时，它会返回最后 200 行以及分页提示。设置 `offset` 且未设置 `limit` 时，它会返回从 `offset` 到末尾的内容（不限制为 200 行）。
- `poll` 的 `timeout` 在返回前最多等待指定的毫秒数；超过 30000 的值会被限制为 30000。
- 轮询用于按需获取状态，而不是安排等待循环。如果工作应在以后执行，请使用 cron。

## 示例

运行长时间任务并稍后轮询：

```json
{ "tool": "exec", "command": "sleep 5 && echo done", "yieldMs": 1000 }
```

```json
{ "tool": "process", "action": "poll", "sessionId": "<id>" }
```

发送输入前检查交互式会话：

```json
{ "tool": "process", "action": "log", "sessionId": "<id>" }
```

立即在后台启动：

```json
{ "tool": "exec", "command": "npm run build", "background": true }
```

发送 stdin：

```json
{ "tool": "process", "action": "write", "sessionId": "<id>", "data": "y\n" }
```

发送 PTY 按键：

```json
{ "tool": "process", "action": "send-keys", "sessionId": "<id>", "keys": ["C-c"] }
```

提交当前行：

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

粘贴原样文本：

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## 相关内容

- [Exec 工具](/zh-CN/tools/exec)
- [Exec 审批](/zh-CN/tools/exec-approvals)
