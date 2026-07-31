---
read_when:
    - 你希望使用云端托管的沙箱，而不是本地 Docker
    - 你正在设置 OpenShell 插件
    - 你需要在镜像工作区模式和远程工作区模式之间进行选择
summary: 将 OpenShell 用作 OpenClaw 智能体的托管式沙箱后端
title: OpenShell
x-i18n:
    generated_at: "2026-07-26T05:49:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bf5c33912bd0db759a01cf58ea26712a8ada68c0804bf16f69f1f7cdd496828c
    source_path: gateway/openshell.md
    workflow: 16
---

OpenShell 是一种托管式沙箱后端：OpenClaw 不在本地运行 Docker 容器，而是将沙箱生命周期委托给 `openshell` CLI，由它预配远程环境并通过 SSH 执行命令。

该插件复用与通用 [SSH 后端](/zh-CN/gateway/sandboxing#ssh-backend)相同的 SSH 传输和远程文件系统桥接，并增加 OpenShell 生命周期（`sandbox create/get/delete/ssh-config`）以及可选的 `mirror` 工作区同步模式。

## 前提条件

- 已安装 OpenShell 插件（`openclaw plugins install @openclaw/openshell-sandbox`）
- `openshell` CLI 位于 `PATH`（也可通过 `plugins.entries.openshell.config.command` 指定自定义路径）
- 拥有具备沙箱访问权限的 OpenShell 账户
- OpenClaw Gateway 网关正在主机上运行

## 快速开始

```bash
openclaw plugins install @openclaw/openshell-sandbox
```

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

重启 Gateway 网关。在智能体的下一个轮次中，OpenClaw 会创建 OpenShell 沙箱，并通过它路由工具执行。使用以下命令验证：

```bash
openclaw sandbox list
openclaw sandbox explain
```

## 工作区模式

这是使用 OpenShell 时最重要的决策。

### mirror（默认）

`plugins.entries.openshell.config.mode: "mirror"` 将**本地工作区作为规范来源**：

- 在 `exec` 之前，OpenClaw 将本地工作区同步到沙箱。
- 在 `exec` 之后，OpenClaw 将远程工作区同步回本地。
- 文件工具通过沙箱桥接运行，但在轮次之间，本地仍是事实来源。

最适合开发工作流：在 OpenClaw 外部进行的本地编辑会在下一次执行时显示，并且沙箱行为与 Docker 后端较为接近。

权衡：每个执行轮次都会产生上传和下载开销。

### remote

`mode: "remote"` 将 **OpenShell 工作区作为规范来源**：

- 首次创建沙箱时，OpenClaw 会从本地向远程工作区执行一次初始填充。
- 此后，`exec`、`read`、`write`、`edit` 和 `apply_patch` 将直接操作远程工作区。OpenClaw **不会**将远程更改同步回本地。
- 提示词处理期间的媒体读取仍然有效（文件/媒体工具通过沙箱桥接读取）。

最适合长时间运行的智能体和 CI：每轮开销更低，并且主机上的本地编辑无法在未察觉的情况下覆盖远程状态。

<Warning>
初始填充后，在主机上通过 OpenClaw 外部编辑文件，远程沙箱将无法看到这些更改。运行 `openclaw sandbox recreate` 重新填充。
</Warning>

### 选择模式

|                          | `mirror`                   | `remote`                  |
| ------------------------ | -------------------------- | ------------------------- |
| **规范工作区**  | 本地主机                 | 远程 OpenShell          |
| **同步方向**       | 双向（每次执行） | 一次性填充             |
| **每轮开销**    | 较高（上传 + 下载） | 较低（直接远程操作） |
| **本地编辑是否可见？** | 是，在下次执行时          | 否，直到重新创建        |
| **最适合**             | 开发工作流      | 长时间运行的智能体、CI   |

## 配置参考

所有 OpenShell 配置均位于 `plugins.entries.openshell.config` 下：

| 键                       | 类型                     | 默认值       | 描述                                                                            |
| ------------------------- | ------------------------ | ------------- | -------------------------------------------------------------------------------------- |
| `mode`                    | `"mirror"` 或 `"remote"` | `"mirror"`    | 工作区同步模式                                                                    |
| `command`                 | `string`                 | `"openshell"` | `openshell` CLI 的路径或名称                                                    |
| `from`                    | `string`                 | `"openclaw"`  | 首次创建时的沙箱来源                                                   |
| `gateway`                 | `string`                 | 未设置         | OpenShell Gateway 网关名称（顶层 `--gateway`）                                         |
| `gatewayEndpoint`         | `string`                 | 未设置         | OpenShell Gateway 网关端点（顶层 `--gateway-endpoint`）                            |
| `policy`                  | `string`                 | 未设置         | 用于创建沙箱的 OpenShell 策略 ID                                               |
| `providers`               | `string[]`               | `[]`          | 创建沙箱时附加的提供商名称（去重，每项对应一个 `--provider` 标志） |
| `gpu`                     | `boolean`                | `false`       | 请求 GPU 资源（`--gpu`）                                                        |
| `autoProviders`           | `boolean`                | `true`        | 创建时传递 `--auto-providers`（为 false 时传递 `--no-auto-providers`）            |
| `remoteWorkspaceDir`      | `string`                 | `"/sandbox"`  | 沙箱内的主要可写工作区                                          |
| `remoteAgentWorkspaceDir` | `string`                 | `"/agent"`    | Agent 工作区挂载路径（工作区访问权限不是 `rw` 时为只读）               |
| `timeoutSeconds`          | `number`                 | `120`         | `openshell` CLI 操作的超时时间                                                 |

`remoteWorkspaceDir` 和 `remoteAgentWorkspaceDir` 必须是绝对路径，并且位于托管根目录 `/sandbox` 或 `/agent` 下；其他绝对路径会被拒绝。

与其他后端一样，沙箱级设置（`mode`、`scope`、`workspaceAccess`）位于 `agents.defaults.sandbox` 下。完整矩阵请参阅[沙箱隔离](/zh-CN/gateway/sandboxing)。

## 示例

### 最小 remote 设置

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

### 带 GPU 的 mirror 模式

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "agent",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "mirror",
          gpu: true,
          providers: ["openai"],
          timeoutSeconds: 180,
        },
      },
    },
  },
}
```

### 使用自定义 Gateway 网关的按 Agent 配置 OpenShell

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "off" },
    },
    list: [
      {
        id: "researcher",
        sandbox: {
          mode: "all",
          backend: "openshell",
          scope: "agent",
          workspaceAccess: "rw",
        },
      },
    ],
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
          gateway: "lab",
          gatewayEndpoint: "https://lab.example",
          policy: "strict",
        },
      },
    },
  },
}
```

## 生命周期管理

```bash
# 列出所有沙箱运行时（Docker + OpenShell）
openclaw sandbox list

# 检查生效的策略
openclaw sandbox explain

# 重新创建（删除远程工作区，在下次使用时重新填充）
openclaw sandbox recreate --all
```

对于 `remote` 模式，重新创建尤为重要：它会删除该作用域的规范远程工作区，并在下次使用时从本地填充一个新的工作区。对于 `mirror` 模式，由于本地仍是规范来源，重新创建主要用于重置远程执行环境。

更改以下任一项后，请重新创建：

- `agents.defaults.sandbox.backend`
- `plugins.entries.openshell.config.from`
- `plugins.entries.openshell.config.mode`
- `plugins.entries.openshell.config.policy`

## 安全加固

mirror 模式文件系统桥接会固定本地工作区根目录，并在每次读取、写入、创建目录、删除和重命名前重新检查规范路径（通过 realpath），拒绝路径中间的符号链接。替换符号链接或重新挂载工作区都无法将文件访问重定向到镜像目录树之外。

## 当前限制

- OpenShell 后端不支持沙箱浏览器。
- `sandbox.docker.binds` 不适用于 OpenShell；如果配置了绑定挂载，沙箱创建将失败。
- `sandbox.docker.*` 下的 Docker 专用运行时选项（`env` 除外）仅适用于 Docker 后端。

## 工作原理

1. OpenClaw 针对沙箱名称运行 `sandbox get`（使用任何已配置的 `--gateway`/`--gateway-endpoint`）；如果失败，则使用 `sandbox create` 创建沙箱，并在设置时传递 `--name`、`--from`、`--policy`，启用时传递 `--gpu`，还会传递 `--auto-providers`/`--no-auto-providers`，以及为每个已配置提供商传递一个 `--provider` 标志。
2. OpenClaw 针对沙箱名称运行 `sandbox ssh-config`，以获取 SSH 连接详细信息。
3. 核心将 SSH 配置写入临时文件，并通过与通用 SSH 后端相同的远程文件系统桥接打开 SSH 会话。
4. 在 `mirror` 模式下：执行前将本地同步到远程，运行命令，之后再同步回来。
5. 在 `remote` 模式下：创建时填充一次，之后直接操作远程工作区。

## 相关内容

- [沙箱隔离](/zh-CN/gateway/sandboxing) - 模式、作用域和后端比较
- [沙箱、工具策略和提升权限](/zh-CN/gateway/sandbox-vs-tool-policy-vs-elevated) - 调试被阻止的工具
- [多 Agent 沙盒和工具](/zh-CN/tools/multi-agent-sandbox-tools) - 按 Agent 覆盖
- [沙箱 CLI](/zh-CN/cli/sandbox) - `openclaw sandbox` 命令
