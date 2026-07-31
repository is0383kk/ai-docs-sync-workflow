---
read_when: You want a dedicated explanation of sandboxing or need to tune agents.defaults.sandbox.
sidebarTitle: Sandboxing
status: active
summary: OpenClaw 沙箱隔离的工作原理：模式、范围、工作区访问和镜像
title: 沙箱隔离
x-i18n:
    generated_at: "2026-07-26T06:16:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a3668dc512a8ff30732290ee68e9dd29a3a2e9c106e6e39077a97bfbd90098f7
    source_path: gateway/sandboxing.md
    workflow: 16
---

OpenClaw 可以在沙箱后端内执行工具，以减小影响范围。沙箱隔离默认关闭，由 `agents.defaults.sandbox`（全局）或 `agents.entries.*.sandbox`（按 Agent）控制。Gateway 网关进程始终留在主机上；启用后，只有工具执行会移入沙箱。

<Note>
这并非完美的安全边界，但当模型做出不当操作时，它可以切实限制对文件系统和进程的访问。
</Note>

## 哪些内容会被沙箱隔离

- 工具执行：`exec`、`read`、`write`、`edit`、`apply_patch`、`process` 等。
- 可选的沙箱浏览器（`agents.defaults.sandbox.browser`）。

不进行沙箱隔离的内容：

- Gateway 网关进程本身。
- 通过 `tools.elevated` 明确允许在沙箱外运行的任何工具。提升权限的 Exec 会绕过沙箱隔离，并在配置的逃逸路径上运行（默认为 `gateway`；当 Exec 目标为 `node` 时，则为 `node`）。如果沙箱隔离已关闭，`tools.elevated` 不会产生任何变化，因为 Exec 已经在主机上运行。参阅[提升权限模式](/zh-CN/tools/elevated)。

## 模式、范围和后端

三个相互独立的设置控制沙箱行为：

| 设置 | 键                               | 值                       | 默认值  |
| ------- | --------------------------------- | ---------------------------- | -------- |
| 模式    | `agents.defaults.sandbox.mode`    | `off`、`non-main`、`all`     | `off`    |
| 范围   | `agents.defaults.sandbox.scope`   | `agent`、`session`、`shared` | `agent`  |
| 后端 | `agents.defaults.sandbox.backend` | `docker`、`ssh`、`openshell` | `docker` |

**模式**控制何时应用沙箱隔离：

- `off`：不进行沙箱隔离。
- `non-main`：除 Agent 主会话外，对每个会话进行沙箱隔离。主会话键始终为 `agent:<agentId>:main`（当 `session.scope` 为 `"global"` 时则为 `global`）；不可配置。群组/渠道会话使用各自的键，因此始终被视为非主会话并进行沙箱隔离。
- `all`：每个会话都在沙箱中运行。

**范围**控制创建多少个容器/环境：

- `agent`：每个 Agent 使用一个容器。
- `session`：每个会话使用一个容器。
- `shared`：所有沙箱会话共享一个容器（在此范围下，会忽略按 Agent 配置的 `docker`/`ssh`/`browser` 覆盖项）。

**后端**控制由哪个运行时执行沙箱工具。SSH 专用配置位于 `agents.defaults.sandbox.ssh` 下；OpenShell 专用配置位于 `plugins.entries.openshell.config` 下。

|                     | Docker                           | SSH                            | OpenShell                                           |
| ------------------- | -------------------------------- | ------------------------------ | --------------------------------------------------- |
| **运行位置**   | 本地容器                  | 任何可通过 SSH 访问的主机        | OpenShell 托管沙箱                           |
| **设置**           | `scripts/sandbox-setup.sh`       | SSH 密钥 + 目标主机          | 已启用 OpenShell 插件                            |
| **工作区模型** | 绑定挂载或复制               | 以远程为准（一次性初始化）   | `mirror` 或 `remote`                                |
| **网络控制** | `docker.network`（默认：无） | 取决于远程主机         | 取决于 OpenShell                                |
| **浏览器沙箱** | 支持                        | 不支持                  | 暂不支持                                   |
| **绑定挂载**     | `docker.binds`                   | 不适用                            | 不适用                                                 |
| **最适合**        | 本地开发、完全隔离        | 将工作卸载到远程计算机 | 具有可选双向同步功能的托管远程沙箱 |

## Docker 后端

启用沙箱隔离后，Docker 是默认后端。它通过 Docker 守护进程套接字（`/var/run/docker.sock`）在本地运行工具和沙箱浏览器；隔离由 Docker 命名空间提供。

默认值：`network: "none"`（无出站访问）、`readOnlyRoot: true`、`capDrop: ["ALL"]`，镜像为 `openclaw-sandbox:bookworm-slim`。

要向容器暴露主机 GPU，请将 `agents.defaults.sandbox.docker.gpus`（或按 Agent 配置的覆盖项）设置为类似 `"all"` 或 `"device=GPU-uuid"` 的值。该值会传递给 Docker 的 `--gpus` 标志，并且需要兼容的主机运行时，例如 NVIDIA Container Toolkit。

<Warning>
**Docker 外运行 Docker（DooD）的限制**

如果将 OpenClaw Gateway 网关本身部署为 Docker 容器，它会使用主机的 Docker 套接字编排同级沙箱容器（DooD）。这会带来路径映射限制：

- **配置需要主机路径**：`openclaw.json` `workspace` 必须包含**主机的绝对路径**（例如 `/home/user/.openclaw/workspaces`），而不是 Gateway 网关容器内部的路径。Docker 守护进程相对于主机操作系统命名空间（而非 Gateway 网关自身的命名空间）解析路径。
- **需要匹配的卷映射**：Gateway 网关进程还会将心跳和桥接文件写入该 `workspace` 路径。请为 Gateway 网关容器提供相同的卷映射（`-v /home/user/.openclaw:/home/user/.openclaw`），使同一主机路径也能从 Gateway 网关容器内正确解析。映射不匹配时，Gateway 网关尝试写入心跳会出现 `EACCES`。
- **Codex 代码模式**：当 OpenClaw 沙箱处于活动状态时，OpenClaw 会为该轮次禁用 Codex app-server 原生代码模式、用户 MCP 服务器以及由应用支持的插件执行（这些功能从 Gateway 网关主机上的 app-server 进程运行，而不是从 OpenClaw 沙箱后端运行），除非沙箱工具策略公开了所需工具，并且你选择启用实验性沙箱 Exec 服务器路径。之后，Shell 访问会通过 OpenClaw 基于沙箱后端的工具（例如 `sandbox_exec` 和 `sandbox_process`）进行路由。不要将主机 Docker 套接字挂载到 Agent 沙箱容器或自定义 Codex 沙箱中。有关完整行为，请参阅 [Codex harness](/zh-CN/plugins/codex-harness)。

在启用了 Docker 沙箱模式的 Ubuntu/AppArmor 主机上，Codex app-server 的 `workspace-write` Shell 执行需要沙箱容器内的非特权用户命名空间；当服务用户无法创建这些命名空间时，执行可能会在 Shell 启动前失败。禁用 Docker 沙箱出站访问时（`network: "none"`，默认值），还需要一个非特权网络命名空间。常见症状包括：`bwrap: setting up uid map: Permission denied` 和 `bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted`。运行 `openclaw doctor`；如果它报告 Codex bwrap 命名空间探测失败，建议使用允许 OpenClaw 服务进程创建所需命名空间的 AppArmor 配置文件。`kernel.apparmor_restrict_unprivileged_userns=0` 是一种会产生安全权衡的主机级回退方案；仅当该主机的安全策略可以接受时才使用。
</Warning>

### 沙箱浏览器

- 当浏览器工具需要沙箱浏览器时，它会自动启动（确保 CDP 可访问）。通过 `agents.defaults.sandbox.browser.autoStart`（默认为 `true`）和 `autoStartTimeoutMs`（默认为 12 秒）进行配置。
- 沙箱浏览器容器使用专用 Docker 网络（`openclaw-sandbox-browser`），而不是全局 `bridge` 网络。通过 `agents.defaults.sandbox.browser.network` 进行配置。
- `agents.defaults.sandbox.browser.cdpSourceRange` 使用 CIDR 允许列表（例如 `172.21.0.1/32`）限制容器边缘的 CDP 入站访问。
- 默认情况下，noVNC 观察器访问受密码保护；OpenClaw 会生成一个短期有效的令牌 URL，该 URL 提供本地引导页面，并使用 URL 片段中的密码打开 noVNC（密码不在查询字符串或请求头日志中）。
- `agents.defaults.sandbox.browser.allowHostControl`（默认为 `false`）允许沙箱会话明确以主机浏览器为目标。
- 可选允许列表用于限制 `target: "custom"`：`allowedControlUrls`、`allowedControlHosts`、`allowedControlPorts`。

## SSH 后端

使用 `backend: "ssh"` 在任意可通过 SSH 访问的计算机上对 `exec`、文件工具和媒体读取进行沙箱隔离。

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        scope: "session",
        workspaceAccess: "rw",
        ssh: {
          target: "user@gateway-host:22",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // 或者使用 SecretRefs / 内联内容代替本地文件：
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

默认值：`command: "ssh"`、`workspaceRoot: "/tmp/openclaw-sandboxes"`、`strictHostKeyChecking: true`、`updateHostKeys: true`。

- **生命周期**：OpenClaw 会在 `sandbox.ssh.workspaceRoot` 下为每个范围创建远程根目录。创建或重新创建后首次使用时，它会从本地工作区一次性初始化该远程工作区。此后，`exec`、`read`、`write`、`edit`、`apply_patch`、提示词媒体读取和入站媒体暂存都会通过 SSH 直接针对远程工作区运行。OpenClaw 不会自动将远程更改同步回本地工作区。
- **身份验证材料**：`identityFile`/`certificateFile`/`knownHostsFile` 引用现有本地文件。`identityData`/`certificateData`/`knownHostsData` 接受内联字符串或 SecretRefs；它们通过常规密钥运行时快照解析，以 `0600` 模式写入临时文件，并在 SSH 会话结束时删除。如果同一项同时设置了 `*File` 和 `*Data` 变体，则该会话优先使用 `*Data`。
- **以远程为准的影响**：初始初始化后，远程 SSH 工作区会成为真正的沙箱状态。在初始化步骤之后于 OpenClaw 外部进行的主机本地编辑，在重新创建沙箱之前不会在远程可见。`openclaw sandbox recreate` 会删除每个范围的远程根目录，并在下次使用时再次从本地初始化。此后端不支持浏览器沙箱隔离，`sandbox.docker.*` 设置也不适用于它。

## OpenShell 后端

使用 `backend: "openshell"` 在 OpenShell 管理的远程环境中对工具进行沙箱隔离。OpenShell 复用与通用 SSH 后端相同的 SSH 传输和远程文件系统桥接，并添加 OpenShell 生命周期（`sandbox create/get/delete/ssh-config`）以及可选的 `mirror` 工作区同步模式。

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
          mode: "remote", // 镜像 | 远程
        },
      },
    },
  },
}
```

`mode: "mirror"`（默认）将本地工作区保持为规范来源：OpenClaw 在 `exec` 之前将本地内容同步到沙箱，并在之后同步回来。`mode: "remote"` 仅从本地初始化远程工作区一次，随后直接针对远程工作区运行 `exec`/`read`/`write`/`edit`/`apply_patch`，而不会同步回来；初始化后的本地编辑在你执行 `openclaw sandbox recreate` 之前不可见。在 `scope: "agent"` 或 `scope: "shared"` 下，该远程工作区会在相同范围内共享。当前限制：尚不支持沙箱浏览器，并且 `sandbox.docker.binds` 不适用于此后端。

`openclaw sandbox list`/`recreate`/prune 对 OpenShell 运行时和 Docker 运行时一视同仁；清理逻辑可感知后端。

有关完整的前提条件、配置参考、工作区模式比较和生命周期详情，请参阅 [OpenShell](/zh-CN/gateway/openshell)。

## 工作区访问

`agents.defaults.sandbox.workspaceAccess` 控制沙箱可以看到的内容：

| 值            | 行为                                                                                  |
| ---------------- | ----------------------------------------------------------------------------------------- |
| `none`（默认） | 工具可看到 `~/.openclaw/sandboxes` 下的隔离沙箱工作区。                    |
| `ro`             | 将 Agent 工作区以只读方式挂载到 `/agent`（禁用 `write`/`edit`/`apply_patch`）。 |
| `rw`             | 将 Agent 工作区以读写方式挂载到 `/workspace`。                                    |

使用 OpenShell 后端时，`mirror` 模式仍在两次 exec 轮次之间将本地工作区用作规范来源；`remote` 模式在初次初始化后将远程 OpenShell 工作区用作规范来源；`workspaceAccess: "ro"`/`"none"` 仍以相同方式限制写入行为。

入站媒体会复制到当前使用的沙箱工作区（`media/inbound/*`）。

<Note>
**Skills**：`read` 工具以沙箱根目录为根。使用 `workspaceAccess: "none"` 时，OpenClaw 会将符合条件的 Skills 镜像到沙箱工作区（`.../skills`），以便读取。使用 `"rw"` 时，可从 `/workspace/skills` 读取工作区 Skills，而符合条件的托管、内置或插件 Skills 会具现化到生成的只读路径 `/workspace/.openclaw/sandbox-skills/skills` 中。
</Note>

## 一个 Agent 使用多个文件夹

当一个沙箱隔离的 Agent 需要访问主工作区之外的更多文件夹时，请使用 Docker 绑定挂载。每个条目都使用明确的访问模式，将一个主机文件夹映射到一个容器路径：

```text
host-directory:container-directory:ro
host-directory:container-directory:rw
```

- `ro` 使挂载的文件夹在沙箱内为只读。
- `rw` 允许沙箱隔离的工具和进程更改主机文件夹。
- 容器路径是 Agent 使用的路径。主机路径不会自动暴露。

此示例为 `research` Agent 提供一个可写的主工作区、位于 `/reference` 的只读参考资料，以及位于 `/drafts` 的独立可写输出文件夹：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        scope: "agent",
      },
    },
    list: [
      {
        id: "research",
        workspace: "/srv/openclaw/research-workspace",
        sandbox: {
          workspaceAccess: "rw",
          docker: {
            binds: ["/srv/shared/reference:/reference:ro", "/srv/shared/drafts:/drafts:rw"],
            // 必需，因为这些源位于 Agent 工作区之外。
            dangerouslyAllowExternalBindSources: true,
          },
        },
      },
    ],
  },
}
```

`workspaceAccess` 与绑定模式彼此独立：

| 设置                          | 控制内容                                                                    |
| -------------------------------- | --------------------------------------------------------------------------- |
| `workspaceAccess: "none"`        | 使用隔离的沙箱工作区；不暴露 Agent 工作区。    |
| `workspaceAccess: "ro"`          | 将 Agent 工作区以只读方式挂载到 `/agent`。                           |
| `workspaceAccess: "rw"`          | 将 Agent 工作区以读写方式挂载到 `/workspace`。                      |
| `docker.binds` 条目 `:ro`/`:rw` | 仅控制该额外主机文件夹在其所配置容器路径上的访问方式。 |

更改 `workspaceAccess` 不会将额外绑定从 `ro` 改为 `rw`，反之亦然。全局和按 Agent 配置的 `docker.binds` 会合并。对按 Agent 配置的绑定使用 `scope: "agent"` 或 `"session"`；`scope: "shared"` 会忽略所有按 Agent 配置的 Docker 覆盖项，仅使用全局绑定。

绑定挂载是受支持的多文件夹边界，因为 Docker 使用挂载隔离构建容器的文件系统视图，而 `ro`/`rw` 模式适用于沙箱中的每个进程。该边界涵盖 `exec`、文件系统工具、子进程和库，无需在 OpenClaw 的每条代码路径中重复路径授权检查。当获准使用的 shell 或依赖项可以直接访问文件时，主机端路径允许列表无法提供同样完整的边界。

选择启用的 `dangerouslyAllowExternalBindSources` 仅允许使用工作区根目录之外的源。它不会禁用 OpenClaw 对系统路径、凭证、Docker 套接字、符号链接父级或保留目标的阻止检查。应优先使用范围最小的文件夹；除非需要写入，否则请使用 `ro`；更改挂载后请重新创建沙箱：

```bash
openclaw sandbox recreate --agent research
```

### 其他绑定行为

`agents.defaults.sandbox.docker.binds` 配置全局挂载。格式为相同的 `host:container:mode` 形式（例如 `"/home/user/source:/source:rw"`）。

`agents.defaults.sandbox.browser.binds` 仅将额外的主机目录挂载到**沙箱浏览器**容器。设置时（包括 `[]`），它会替换浏览器容器的 `docker.binds`；省略时，浏览器容器会回退到 `docker.binds`。

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          binds: ["/home/user/source:/source:ro", "/var/data/myapp:/data:ro"],
        },
      },
    },
    list: [
      {
        id: "build",
        sandbox: {
          docker: {
            binds: ["/mnt/cache:/cache:rw"],
          },
        },
      },
    ],
  },
}
```

<Warning>
**绑定安全**

- 绑定会绕过沙箱文件系统：它们会按照你设置的模式（`:ro` 或 `:rw`）暴露主机路径。
- 默认情况下，OpenClaw 会阻止危险的绑定源：系统路径（`/etc`、`/proc`、`/sys`、`/dev`、`/root`、`/boot`）、Docker 套接字目录（`/run`、`/var/run` 及其 `docker.sock` 变体），以及常见的主目录凭证根目录（`~/.aws`、`~/.cargo`、`~/.config`、`~/.docker`、`~/.gnupg`、`~/.netrc`、`~/.npm`、`~/.ssh`）。
- 验证会先规范化源路径，再通过最深层的现有祖先重新解析该路径，然后重新检查被阻止的路径和允许的根目录。因此，即使最终叶节点尚不存在，通过符号链接父级进行的越界也会以拒绝方式失败（例如，如果 `run-link` 指向该处，`/workspace/run-link/new-file` 仍会解析为 `/var/run/...`）。
- 默认情况下，也会阻止遮蔽容器保留挂载点（`/workspace`、`/agent`）的绑定目标；可使用 `agents.defaults.sandbox.docker.dangerouslyAllowReservedContainerTargets: true` 覆盖此行为。
- 默认情况下，会阻止工作区/Agent 工作区允许列表根目录之外的绑定源；可使用 `agents.defaults.sandbox.docker.dangerouslyAllowExternalBindSources: true` 覆盖此行为。允许的根目录会以相同方式进行规范化，因此，如果一个路径只是在解析符号链接之前看起来位于允许列表内，仍会因位于允许根目录之外而被拒绝。
- 除非绝对必要，否则敏感挂载（密钥、SSH 密钥、服务凭证）应使用 `:ro`。
- 如果只需要对工作区进行读取访问，请与 `workspaceAccess: "ro"` 结合使用；绑定模式仍彼此独立。
- 有关绑定如何与工具策略和提升权限的 Exec 交互，请参阅[沙箱、工具策略和提升权限](/zh-CN/gateway/sandbox-vs-tool-policy-vs-elevated)。

</Warning>

## 镜像和设置

默认 Docker 镜像：`openclaw-sandbox:bookworm-slim`

<Note>
**源代码检出与 npm 安装**

仅在从[源代码检出](https://github.com/openclaw/openclaw)运行时，才可使用 `scripts/sandbox-setup.sh`、`scripts/sandbox-common-setup.sh` 和 `scripts/sandbox-browser-setup.sh` 辅助脚本。npm 软件包中不包含这些脚本。

如果通过 `npm install -g openclaw` 安装了 OpenClaw，请改用下方显示的内联 `docker build` 命令。
</Note>

<Steps>
  <Step title="构建默认镜像">
    从源代码检出运行：

    ```bash
    scripts/sandbox-setup.sh
    ```

    从 npm 安装运行（无需检出源代码）：

    ```bash
    docker build -t openclaw-sandbox:bookworm-slim - <<'DOCKERFILE'
    FROM debian:bookworm-slim
    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt-get update && apt-get install -y --no-install-recommends \
      bash ca-certificates curl git jq python3 ripgrep \
      && rm -rf /var/lib/apt/lists/*
    RUN useradd --create-home --shell /bin/bash sandbox
    USER sandbox
    WORKDIR /home/sandbox
    CMD ["sleep", "infinity"]
    DOCKERFILE
    ```

    默认镜像**不**包含 Node。如果某个 Skill 需要 Node（或其他运行时），请构建自定义镜像，或通过 `sandbox.docker.setupCommand` 安装（需要网络出口、可写根目录和 root 用户）。

    当缺少 `openclaw-sandbox:bookworm-slim` 时，OpenClaw 不会静默改用普通的 `debian:bookworm-slim`。针对默认镜像的沙箱运行会快速失败，并给出构建说明，直到你构建该镜像为止，因为内置镜像包含沙箱写入/编辑辅助工具所需的 `python3`。

  </Step>
  <Step title="可选：构建通用镜像">
    如需包含常用工具、功能更齐全的沙箱镜像（例如 `curl`、`jq`、Node 24、pnpm、`python3` 和 `git`）：

    从源代码检出运行：

    ```bash
    scripts/sandbox-common-setup.sh
    ```

    从 npm 安装运行时，请先构建默认镜像（见上文），然后使用仓库中的 [`scripts/docker/sandbox/Dockerfile.common`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.common)，在默认镜像之上构建通用镜像。

    然后将 `agents.defaults.sandbox.docker.image` 设置为 `openclaw-sandbox-common:bookworm-slim`。

  </Step>
  <Step title="可选：构建沙箱浏览器镜像">
    从源代码检出运行：

    ```bash
    scripts/sandbox-browser-setup.sh
    ```

    从 npm 安装运行时，请使用仓库中的 [`scripts/docker/sandbox/Dockerfile.browser`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.browser) 进行构建。

  </Step>
</Steps>

默认情况下，Docker 沙箱容器在**无网络**环境下运行。可通过 `agents.defaults.sandbox.docker.network` 覆盖此设置。

<AccordionGroup>
  <Accordion title="沙箱浏览器 Chromium 默认设置">
    内置的沙箱浏览器镜像针对容器化工作负载应用了保守的 Chromium 启动标志：

    - `--remote-debugging-address=127.0.0.1`
    - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
    - `--user-data-dir=${HOME}/.chrome`
    - `--no-first-run`
    - `--no-default-browser-check`
    - `--disable-dev-shm-usage`
    - `--disable-background-networking`
    - `--disable-breakpad`
    - `--disable-crash-reporter`
    - `--no-zygote`
    - `--metrics-recording-only`
    - `--password-store=basic`
    - `--use-mock-keychain`
    - 启用 `browser.headless` 时为 `--headless=new`。
    - 启用 `browser.noSandbox` 时为 `--no-sandbox --disable-setuid-sandbox`。
    - 默认使用 `--disable-3d-apis`、`--disable-gpu`、`--disable-software-rasterizer`；这些图形强化标志有助于不支持 GPU 的容器。如果你的工作负载需要 WebGL 或其他 3D 功能，请设置 `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0`。
    - 默认为 `--disable-extensions`；对于依赖扩展程序的流程，请设置 `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0`。
    - 默认为 `--renderer-process-limit=2`；由 `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>` 控制，其中 `0` 保留 Chromium 的默认设置。

    如果需要不同的运行时配置文件，请使用自定义浏览器镜像并提供自己的入口点。对于本地（非容器）Chromium 配置文件，请使用 `browser.extraArgs` 追加额外的启动标志。

  </Accordion>
  <Accordion title="网络安全默认设置">
    - 已阻止 `network: "host"`。
    - 默认阻止 `network: "container:<id>"`（存在加入命名空间绕过风险）。
    - 紧急覆盖：`agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`。

  </Accordion>
</AccordionGroup>

Docker 安装和容器化 Gateway 网关位于此处：[Docker](/zh-CN/install/docker)

对于 Docker Gateway 网关部署，`scripts/docker/setup.sh` 可以引导沙箱配置。设置 `OPENCLAW_SANDBOX=1`（或 `true`/`yes`/`on`）可启用该路径。使用 `OPENCLAW_DOCKER_SOCKET` 覆盖套接字位置。完整设置和环境变量参考：[Docker](/zh-CN/install/docker#agent-sandbox)。

## setupCommand（一次性容器设置）

`setupCommand` 在沙箱容器创建后**运行一次**（不会在每次运行时执行）。它通过 `sh -lc` 在容器内执行。

路径：

- 全局：`agents.defaults.sandbox.docker.setupCommand`
- 按智能体：`agents.entries.*.sandbox.docker.setupCommand`

<AccordionGroup>
  <Accordion title="常见陷阱">
    - 默认 `docker.network` 为 `"none"`（无出站访问），因此软件包安装会失败。
    - `docker.network: "container:<id>"` 需要 `dangerouslyAllowContainerNamespaceJoin: true`，仅供紧急情况使用。
    - `readOnlyRoot: true` 会阻止写入；请设置 `readOnlyRoot: false` 或构建自定义镜像。
    - 安装软件包时，`user` 必须为 root（省略 `user` 或设置 `user: "0:0"`）。
    - 沙箱 Exec **不会**继承主机的 `process.env`。对于 Skills API 密钥，请使用 `agents.defaults.sandbox.docker.env`（或自定义镜像）。
    - `agents.defaults.sandbox.docker.env` 中的值会作为显式 Docker 容器环境变量传递。任何有权访问 Docker 守护进程的人都可以使用 `docker inspect` 等 Docker 元数据命令检查这些值。如果这种元数据暴露不可接受，请使用自定义镜像、挂载的密钥文件或其他密钥交付路径。

  </Accordion>
</AccordionGroup>

## 工具策略和逃生通道

工具允许/拒绝策略仍会先于沙箱规则应用。如果某个工具在全局或按智能体被拒绝，沙箱隔离不会将其恢复。

`tools.elevated` 是一个显式逃生通道，可在沙箱外运行 `exec`（默认为 `gateway`，当 Exec 目标为 `node` 时则为 `node`）。`/exec` 指令仅对已授权的发送者生效，并按会话持久保留；要硬性禁用 `exec`，请使用工具策略拒绝（参见[沙箱、工具策略和提升权限](/zh-CN/gateway/sandbox-vs-tool-policy-vs-elevated)）。

调试：

- `openclaw sandbox list` 显示沙箱容器、状态、镜像匹配情况、存续时间、空闲时间以及关联的会话/智能体。
- `openclaw sandbox explain [--session <key>] [--agent <id>]` 检查生效的沙箱模式、主机工作区、运行时工作目录、Docker 挂载、工具策略以及修复配置键。其 `workspaceRoot` 字段仍是配置的沙箱根目录；`effectiveHostWorkspaceRoot` 显示活动工作区的实际位置。
- `openclaw sandbox recreate [--all | --session <key> | --agent <id>] [--browser] [--force]` 删除容器/环境，以便下次使用时根据当前配置重新创建。
- 有关“为什么这被阻止？”的思维模型，请参阅[沙箱、工具策略和提升权限](/zh-CN/gateway/sandbox-vs-tool-policy-vs-elevated)。

## 多智能体覆盖设置

每个智能体都可以覆盖沙箱和工具设置：`agents.entries.*.sandbox` 和 `agents.entries.*.tools`（以及用于沙箱工具策略的 `agents.entries.*.tools.sandbox.tools`）。有关优先级，请参阅[多 Agent 沙盒和工具](/zh-CN/tools/multi-agent-sandbox-tools)。

## 最小启用示例

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
      },
    },
  },
}
```

## 相关内容

- [多 Agent 沙盒和工具](/zh-CN/tools/multi-agent-sandbox-tools) -- 按智能体覆盖设置和优先级
- [OpenShell](/zh-CN/gateway/openshell) -- 托管沙箱后端设置、工作区模式和配置参考
- [沙箱配置](/zh-CN/gateway/config-agents#agentsdefaultssandbox)
- [沙箱、工具策略和提升权限](/zh-CN/gateway/sandbox-vs-tool-policy-vs-elevated) -- 调试“为什么这被阻止？”
- [安全性](/zh-CN/gateway/security)
