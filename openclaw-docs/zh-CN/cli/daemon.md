---
read_when:
    - 你仍在脚本中使用 `openclaw daemon ...`
    - 你需要服务生命周期命令（安装/启动/停止/重启/状态）
summary: '`openclaw daemon` 的 CLI 参考（Gateway 网关服务管理的旧版别名）'
title: 守护进程
x-i18n:
    generated_at: "2026-07-26T06:04:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 629852ebf3efe86dedc4c84f6ddc9349b25ddde832df5d78521641fe4b137658
    source_path: cli/daemon.md
    workflow: 16
---

# `openclaw daemon`

Gateway 网关服务管理的旧版别名。`openclaw daemon ...` 映射到与 `openclaw gateway ...` 相同的服务控制命令。当前文档和示例请优先参阅 [`openclaw gateway`](/zh-CN/cli/gateway)。

## 用法

```bash
openclaw daemon status
openclaw daemon install
openclaw daemon start
openclaw daemon stop
openclaw daemon restart
openclaw daemon uninstall
```

## 子命令和选项

| 子命令  | 选项                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------ |
| `status`    | `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json` |
| `install`   | `--port`, `--runtime <node>`, `--token`, `--wrapper <path>`, `--force`, `--json`                 |
| `uninstall` | `--json`                                                                                         |
| `start`     | `--json`                                                                                         |
| `stop`      | `--json`, `--disable`（仅限 launchd：在下次启动前持续禁止 KeepAlive/RunAtLoad） |
| `restart`   | `--force`, `--safe`, `--skip-deferral`, `--wait <duration>`, `--json`                            |

- `status`：显示服务安装状态（launchd/systemd/schtasks）并探测 Gateway 健康。
- `install`：安装服务；`--force` 会重新安装或覆盖现有安装。
- `restart --safe`：请求正在运行的 Gateway 网关预检当前活动任务，并在任务全部完成后安排一次合并重启，最长等待 5 分钟。超过此时间限制后，仍会强制重启。普通的 `restart` 直接使用服务管理器；`--force` 是立即执行的覆盖选项。
- `restart --safe --skip-deferral`：绕过活动任务延迟门控，因此即使报告了阻塞项，Gateway 网关也会立即重启。需要 `--safe`。

## 说明

- `status` 会尽可能解析已配置的身份验证 SecretRef，以用于探测身份验证。如果所需的 SecretRef 未解析，`status --json` 会报告 `rpc.authWarning`；请显式传入 `--token`/`--password`，或先解析密钥来源。一旦探测在其他方面成功，就会抑制未解析身份验证警告。
- `status --deep` 会额外执行尽力而为的系统级扫描，以查找其他类似 Gateway 网关的服务（输出清理提示；仍建议每台机器仅运行一个 Gateway 网关），并以插件感知模式运行配置验证，显示快速默认路径会跳过的插件清单警告。
- 在 Linux systemd 安装中，令牌漂移检查会检查 `Environment=` 和 `EnvironmentFile=` 这两个单元来源。
- 令牌漂移检查使用合并后的运行时环境解析 `gateway.auth.token` SecretRef（先使用服务命令环境，再使用进程环境）。如果令牌身份验证实际上未启用（`password`/`none`/`trusted-proxy` 的 `gateway.auth.mode`，或未设置且密码可以优先），则跳过配置令牌解析。
- `install` 会验证由 SecretRef 管理的 `gateway.auth.token` 是否可解析，但绝不会将解析后的值持久化到服务环境元数据中；如果无法解析，安装将以关闭方式失败。
- 如果同时配置了 `gateway.auth.token` 和 `gateway.auth.password`，但未设置 `gateway.auth.mode`，则 `install` 会阻止继续执行，直到你显式设置模式。
- 在 macOS 上，`install` 会将 LaunchAgent plist 和生成的环境文件/包装器设为仅所有者可访问（模式 `0600`/`0700`），而不是将密钥嵌入 `EnvironmentVariables`。
- 在一台主机上运行多个 Gateway 网关：请隔离端口、配置/状态和工作区。请参阅[多个 Gateway 网关](/zh-CN/gateway#multiple-gateways-same-host)。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [Gateway 网关运行手册](/zh-CN/gateway)
