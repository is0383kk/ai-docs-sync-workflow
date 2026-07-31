---
read_when: You are managing sandbox runtimes or debugging sandbox/tool-policy behavior.
status: active
summary: 管理沙箱运行时并检查生效的沙箱策略
title: 沙箱 CLI
x-i18n:
    generated_at: "2026-07-26T06:41:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea8311de7702222295f3ba8753304e30f6ed21958e2843f62db5d064f06e24ae
    source_path: cli/sandbox.md
    workflow: 16
---

管理用于隔离智能体执行的沙箱运行时：Docker 容器、SSH 目标或 OpenShell 后端。

## 命令

### `openclaw sandbox list`

列出沙箱运行时及其状态、后端、配置匹配情况、存在时长、空闲时间和关联的会话/智能体。

```bash
openclaw sandbox list
openclaw sandbox list --browser  # 仅浏览器容器
openclaw sandbox list --json
```

### `openclaw sandbox recreate`

移除沙箱运行时，以强制使用当前配置重新创建。下次使用智能体时，运行时会自动重新创建。

```bash
openclaw sandbox recreate --all
openclaw sandbox recreate --agent mybot        # 包括 agent:mybot:* 子会话
openclaw sandbox recreate --session "agent:main:main"
openclaw sandbox recreate --browser --all      # 仅浏览器容器
openclaw sandbox recreate --all --force        # 跳过确认
```

选项：

- `--all`：重新创建所有沙箱容器
- `--session <key>`：使用此精确作用域键（如 `sandbox list` 所示）重新创建运行时；不展开短名称
- `--agent <id>`：重新创建一个智能体的运行时（匹配 `agent:<id>` 和 `agent:<id>:*`）
- `--browser`：仅影响浏览器容器
- `--force`：跳过确认提示

必须且只能传入 `--all`、`--session` 或 `--agent` 之一。

对于 `ssh` 和 OpenShell `remote`，重新创建比使用 Docker 时更重要：初始播种后，远程工作区是规范副本；`recreate` 会删除所选作用域的该规范远程工作区，下一次运行会从当前本地工作区重新播种。

### `openclaw sandbox explain`

检查有效的沙箱模式/作用域/工作区访问权限、沙箱工具策略和提升权限工具的门控条件（并提供修复配置键路径）。

报告将 `workspaceRoot` 保留为已配置的沙箱根目录，并单独显示有效的主机工作区、后端运行时工作目录和 Docker 挂载表。对于 `workspaceAccess: "rw"`，有效的主机工作区是智能体工作区，而不是 `workspaceRoot` 下的目录。

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

与 `recreate --session` 不同，此命令接受短会话名称（例如 `main`），并根据解析出的智能体将其展开。

## 为什么需要重新创建

更新沙箱配置不会影响正在运行的容器：现有运行时会保留旧设置，而空闲运行时只有在 `prune.idleHours` 后才会被清理（默认 24h）。经常使用的智能体可能会无限期地保持过时运行时处于活动状态。`openclaw sandbox recreate` 会移除旧运行时，以便下次使用时根据当前配置重建。

<Tip>
优先使用 `openclaw sandbox recreate`，而不是手动执行特定于后端的清理。它使用 Gateway 网关的运行时注册表，可避免作用域或会话键发生变化时出现不匹配。
</Tip>

## 常见触发条件

| 更改                                                                                                                                                         | 命令                                                             |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Docker 镜像更新（`agents.defaults.sandbox.docker.image`）                                                                                                   | `openclaw sandbox recreate --all`                                   |
| 沙箱配置（`agents.defaults.sandbox.*`）                                                                                                                   | `openclaw sandbox recreate --all`                                   |
| SSH 目标/身份验证（`agents.defaults.sandbox.ssh.{target,workspaceRoot,identityFile,certificateFile,knownHostsFile,identityData,certificateData,knownHostsData}`） | `openclaw sandbox recreate --all`                                   |
| OpenShell 源/策略/模式（`plugins.entries.openshell.config.{from,mode,policy}`）                                                                           | `openclaw sandbox recreate --all`                                   |
| `setupCommand`                                                                                                                                                 | `openclaw sandbox recreate --all`（或对一个智能体使用 `--agent <id>`） |

<Note>
下次使用智能体时，运行时会自动重新创建。
</Note>

## 注册表迁移

沙箱运行时元数据存储在共享 SQLite 状态数据库中。旧版安装可能包含常规读取不再重写的旧版注册表文件：

- `~/.openclaw/sandbox/containers.json`
- `~/.openclaw/sandbox/browsers.json`
- 位于 `~/.openclaw/sandbox/containers/` 或 `~/.openclaw/sandbox/browsers/` 下的每个容器/浏览器一个 JSON 分片

运行 `openclaw doctor --fix`，将有效的旧版条目迁移到 SQLite。无效的旧版文件会被隔离，避免损坏的旧注册表隐藏当前运行时条目。

## 配置

沙箱设置位于 `~/.openclaw/openclaw.json` 的 `agents.defaults.sandbox` 下（每个智能体的覆盖配置位于 `agents.entries.*.sandbox`）：

```jsonc
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all", // 关闭、非主会话、全部
        "backend": "docker", // docker、ssh、openshell（由插件提供）
        "scope": "agent", // 会话、智能体、共享
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "containerPrefix": "openclaw-sbx-",
          // ... 更多 Docker 选项
        },
        "prune": {
          "idleHours": 24, // 空闲 24h 后自动清理
          "maxAgeDays": 7, // 7 天后自动清理
        },
      },
    },
  },
}
```

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [沙箱隔离](/zh-CN/gateway/sandboxing)
- [Agent 工作区](/zh-CN/concepts/agent-workspace)
- [Doctor](/zh-CN/gateway/doctor)：检查沙箱设置。
