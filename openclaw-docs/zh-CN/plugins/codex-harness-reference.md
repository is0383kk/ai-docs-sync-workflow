---
read_when:
    - 你需要 Codex harness 的每个配置字段
    - 你正在更改 app-server 的传输、身份验证、设备发现或超时行为
    - 你正在调试 Codex harness 启动、模型发现或环境隔离问题
summary: Codex harness 的配置、身份验证、发现和应用服务器参考
title: Codex harness reference
x-i18n:
    generated_at: "2026-07-26T06:54:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 149f065f5bef18d0f491c97facc4b5991afc3f7e1077abdc7a4b49f506eac3e0
    source_path: plugins/codex-harness-reference.md
    workflow: 16
---

此参考涵盖官方 `codex` 插件的详细配置。
有关设置和路由决策，请先参阅
[Codex harness](/zh-CN/plugins/codex-harness)。

## 插件配置界面

所有 Codex harness 设置都位于 `plugins.entries.codex.config` 下。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
          appServer: {
            mode: "guardian",
          },
        },
      },
    },
  },
}
```

顶层字段：

| 字段                       | 默认值                   | 含义                                                                                                                                           |
| -------------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `discovery`                | 已启用                   | Codex app-server `model/list` 的模型发现设置。                                                                                    |
| `appServer`                | 托管式 stdio app-server | 传输、命令、身份验证、审批、沙箱和超时设置。普通 harness 默认使用 Agent 范围的状态。                        |
| `codexDynamicToolsLoading` | `"searchable"`           | 使用 `"direct"` 将 OpenClaw 动态工具直接放入初始 Codex 工具上下文。                                                       |
| `codexDynamicToolsExclude` | `[]`                     | 要从 Codex app-server 轮次中省略的其他 OpenClaw 动态工具名称。                                                                    |
| `codexPlugins`             | 已禁用                   | Native Codex plugins/app 支持，包括选择启用对已连接账户应用的访问权限。请参阅 [Native Codex plugins](/zh-CN/plugins/codex-native-plugins)。 |
| `computerUse`              | 已禁用                   | Codex Computer Use 设置。请参阅 [Codex Computer Use](/zh-CN/plugins/codex-computer-use)。                                                               |
| `sessionCatalog`           | 已启用                   | 侧边栏的原生 Codex 会话发现。将 `enabled: false` 设置为禁用可关闭发现，而不禁用提供商或 harness。           |
| `supervision`              | 已禁用                   | 面向智能体的原生会话转录和写入控制策略。请参阅 [Codex 监督](/zh-CN/plugins/codex-supervision)。                          |

## 监督

默认情况下，原生会话发现会列出 Gateway 网关计算机和已选择加入的配对节点中未归档的 Codex 会话。仅通过以下配置禁用该目录：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          sessionCatalog: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

`supervision` 单独控制面向智能体的工具：

| 字段                  | 默认值                  | 含义                                                                                                                                                                                                                                    |
| --------------------- | ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`             | `false`                 | 启用面向智能体的 Codex 监督工具。这不会控制已通过身份验证的操作员会话目录。                                                                                                                            |
| `endpoints`           | 内置本地端点            | 为保留的 Codex 监督智能体和独立 MCP 工具提供兼容性及高级端点目标。人工目录和分支流程会忽略这些目标，并使用从 `appServer` 解析的监督 App Server。       |
| `allowRawTranscripts` | `false`                 | 启用监督后，允许自主智能体或独立 MCP 读取转录内容和从转录派生的列表字段。仅包含 `codex_threads` 元数据的读取仍然可用。不会控制已通过身份验证的 Control UI 续接。     |
| `allowWriteControls`  | `false`                 | 启用监督后，允许自主执行 `codex_threads` 分叉、重命名、归档和取消归档变更，以及独立 MCP 的发送、Steer 和中断操作。不会绕过其他绑定、主机、状态或确认检查。 |

端点条目接受以下字段：

| 字段           | 适用范围      | 含义                                                                  |
| -------------- | ------------- | --------------------------------------------------------------------- |
| `id`           | 全部          | 稳定的端点 ID。                                                   |
| `label`        | 全部          | 可选的显示标签。                                               |
| `transport`    | 全部          | `"stdio-proxy"` 或 `"websocket"`。                                     |
| `command`      | `stdio-proxy` | 可选的 App Server 命令。                                          |
| `args`         | `stdio-proxy` | 可选的命令参数。                                           |
| `cwd`          | `stdio-proxy` | 可选的子进程工作目录。                             |
| `url`          | `websocket`   | 必需的 WebSocket 或受支持的本地套接字 URL。                     |
| `authTokenEnv` | `websocket`   | 可选的环境变量，其值用于对端点进行身份验证。 |

**Codex 会话**页面使用插件的监督 App Server，并且仅显示未归档的会话。如果没有明确的 `appServer` 连接设置，
该连接将作为托管式用户主目录 stdio 运行。已存储或空闲的本地行可以通过截至最后一个
终止并已持久化的源轮次的有限用户和助手历史记录，创建模型锁定的聊天。其私有绑定可确保快照分叉、
规范 `appServer` 源分支、历史记录注入和后续轮次都保留在该连接上。
首次规范启动使用分叉返回的配对。后续恢复会省略 OpenClaw 模型和提供商覆盖，
以便 Codex 恢复规范线程持久化的配对；单独的原生变更可以更新该配对，
但外层模型和回退链绝不会替换它。确认没有其他运行器后，可以归档已存储和空闲的行，
除非另一个活跃的 OpenClaw 绑定拥有完全相同的目标，或拥有其某个未归档的派生后代。
OpenClaw 遵循 Codex 的后代分页机制，并在枚举错误、循环或安全限制耗尽时以失败关闭。
确认仍会涵盖未知的原生客户端以及从状态检查到归档操作之间的竞态条件。
受监督且模型锁定的聊天在保护原生绑定时无法删除。
活跃源无法创建分支或被归档，但仍然可以打开现有的受监督聊天。
每个配对节点行均保持只读；节点传输尚未提供 harness 所需的流式生命周期。

只有 `appServer.homeScope: "user"` 会更改托管 harness 进程使用的 Codex 主目录；
它不会发布设备群目录。启用监督不会更改 harness 默认值。
相反，当不存在明确的 `appServer` 连接设置时，
单独的监督连接默认使用托管式用户主目录 stdio。该连接会采用明确设置。
待处理和已提交的受监督绑定会在每个轮次中保留该连接；
监督被禁用或连接/生命周期发生漂移时会以失败关闭，而不会回退到 Agent 主目录 harness。
默认连接与原生 Codex 客户端共享已存储的会话，但不共享其进程本地活动状态。

旧版 `plugins.entries.codex-supervisor` 设置已弃用。运行
`openclaw doctor --fix`，将旧条目、端点定义、策略标志和插件允许/拒绝引用迁移到此配置块中。
发生冲突时，明确的规范 `codex.config.supervision` 值优先。

## App-server 传输

对于普通 harness 轮次，OpenClaw 会启动随官方插件提供的托管 Codex 二进制文件
（当前为 `@openai/codex` `0.145.0`）：

```bash
codex app-server --listen stdio://
```

这样可让 app-server 版本与官方 `codex` 插件保持一致，
而不是取决于本地恰好安装的其他 Codex CLI。仅当有意使用其他可执行文件时，
才设置 `appServer.command`。即使已安装 macOS 桌面应用包，
采用默认隔离 Agent 主目录的普通托管轮次也会优先使用这个固定版本的软件包。
启用[计算机使用](/zh-CN/plugins/codex-computer-use)后，或者当 `homeScope` 为
`"user"` 且能够加载原生计算机使用状态时，托管启动会改为优先使用
拥有所需 macOS 权限的桌面应用二进制文件。当隔离 Agent 主目录的有效 Codex 配置
启用原生计算机使用时，同样适用桌面应用优先规则。如果未安装桌面应用包，
OpenClaw 会回退到固定版本的软件包二进制文件。

可执行文件交接和原生配置隔离会协调同一运行中 Gateway 网关进程内的客户端。
其他进程更改原生 Codex 插件配置后，请重启 Gateway 网关。

监督会解析单独的连接。当没有明确的
`appServer` 连接设置时，它会使用带有 `homeScope: "user"` 的托管式 stdio；
普通 harness 仍使用带有 `homeScope: "agent"` 的托管式 stdio。
两个路径都会采用明确的连接设置。当普通 harness 应与原生客户端共享
`$CODEX_HOME`（或 `~/.codex`）时，请明确设置 `homeScope: "user"`。
无论普通 harness 的默认值如何，私有的受监督绑定都会使用监督连接。
独立的 App Server 进程分别保留各自的实时状态和审批状态。

对于针对已运行 app-server 的非生产测试，可以使用 WebSocket 传输：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
            requestTimeoutMs: 60000,
          },
        },
      },
    },
  },
}
```

Codex 将 WebSocket 传输归类为实验性且不受支持。对于生产工作负载，
请优先使用托管式 stdio 或本地 Unix 控制套接字。

`appServer` 字段：

| 字段                                          | 默认值                                                  | 含义                                                                                                                                                                                                                                                                                                                                                                                         |
| --------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                                   | `"stdio"`                                              | `"stdio"` 会启动 Codex；显式指定 `"unix"` 会连接到本地控制套接字；`"websocket"` 会连接到 `url`。                                                                                                                                                                                                                                                                                |
| `homeScope`                                   | `"agent"`                                              | `"agent"` 会按 OpenClaw 智能体隔离普通 harness 状态。`"user"` 是显式选择启用项，会共享原生 `$CODEX_HOME` 或 `~/.codex`，使用原生身份验证，并启用仅限所有者的线程管理。用户作用域支持本地 stdio 或 Unix 传输。对于单独的监管连接，未设置的值在 stdio 或 Unix 下解析为 `"user"`，在 WebSocket 下解析为 `"agent"`。     |
| `command`                                     | 托管的 Codex 二进制文件                                   | stdio 传输使用的可执行文件。保持未设置以使用托管二进制文件。                                                                                                                                                                                                                                                                                                                          |
| `args`                                        | `["app-server", "--listen", "stdio://"]`               | stdio 传输的参数。                                                                                                                                                                                                                                                                                                                                                                  |
| `url`                                         | 未设置                                                  | WebSocket App Server URL 或 `unix://` URL。显式指定空的 Unix 路径会选择用户主目录下的规范控制套接字。                                                                                                                                                                                                                                                                          |
| `authToken`                                   | 未设置                                                  | WebSocket 传输的 Bearer 令牌。接受字面字符串或 SecretInput，例如 `${CODEX_APP_SERVER_TOKEN}`。                                                                                                                                                                                                                                                                              |
| `headers`                                     | `{}`                                                   | 额外的 WebSocket 标头。标头值接受字面字符串或 SecretInput 值，例如 `x-codex-client-session-token: "${CODEX_CLIENT_SESSION_TOKEN}"`。                                                                                                                                                                                                                               |
| `clearEnv`                                    | `[]`                                                   | OpenClaw 构建继承环境后，从启动的 stdio app-server 进程中移除的额外环境变量名称。                                                                                                                                                                                                                                                             |
| `remoteWorkspaceRoot`                         | 未设置                                                  | 远程 Codex app-server 工作区根目录。设置后，OpenClaw 会根据解析后的 OpenClaw 工作区推断本地工作区根目录，在此远程根目录下保留当前 cwd 后缀，并且只将最终的 app-server cwd 发送给 Codex。如果 cwd 位于解析后的 OpenClaw 工作区根目录之外，OpenClaw 会以关闭方式失败，而不会将 Gateway 网关本地路径发送给远程 app-server。 |
| `loopDetectionPreToolUseRelay`                | `true`                                                 | 安装仅用于 OpenClaw 循环检测及其显式无策略标记的 Codex `PreToolUse` 子进程。设置 `false` 可减少每个工具的进程扇出。工具执行前的插件钩子和可信工具策略仍会安装其所需的中继。                                                                                                                                         |
| `requestTimeoutMs`                            | `60000`                                                | app-server 控制平面调用的超时时间。                                                                                                                                                                                                                                                                                                                                                     |
| `turnCompletionIdleTimeoutMs`                 | `60000`                                                | Codex 接受一个轮次后，或发出轮次作用域的 app-server 请求后，OpenClaw 等待 `turn/completed` 时使用的静默窗口。                                                                                                                                                                                                                                                                    |
| `turnAssistantCompletionIdleTimeoutMs`        | `10000`                                                | 最终/非 commentary 助手项或工具执行前的原始助手完成结果触发助手输出释放后，OpenClaw 仍在等待 `turn/completed` 时使用的静默窗口。提高该值可让 Codex 有更多时间发出 `turn/completed`，然后 OpenClaw 才会中断并释放会话通道。                                                                                            |
| `postToolRawAssistantCompletionIdleTimeoutMs` | `300000`                                               | 工具交接、原生工具完成、工具执行后的原始助手进度、原始推理完成或推理进度之后，OpenClaw 等待 `turn/completed` 时使用的完成空闲和进度保护机制。对于可信或繁重的工作负载，如果工具执行后的综合处理可以合理地保持静默，且静默时间长于最终助手释放预算，请使用此项。                                |
| `mode`                                        | `"yolo"`，除非本地 Codex 要求不允许 YOLO | 用于 YOLO 或经 guardian 审查执行的预设。                                                                                                                                                                                                                                                                                                                                                 |
| `approvalPolicy`                              | `"never"` 或允许的 guardian 审批策略       | 在线程启动、恢复和轮次时发送的原生 Codex 审批策略。                                                                                                                                                                                                                                                                                                                            |
| `sandbox`                                     | `"danger-full-access"` 或允许的 guardian 沙箱  | 在线程启动和恢复时发送的原生 Codex 沙箱模式。启用的 OpenClaw 沙箱会将 `danger-full-access` 轮次收窄为 Codex `workspace-write`；轮次网络标志遵循 OpenClaw 沙箱出站设置。                                                                                                                                                                                       |
| `approvalsReviewer`                           | `"user"` 或允许的 guardian 审查器               | 在允许的情况下，使用 `"auto_review"` 让 Codex 审查原生审批提示。                                                                                                                                                                                                                                                                                                                   |
| `defaultWorkspaceDir`                         | 当前进程目录                              | 省略 `--cwd` 时，`/codex bind` 使用的工作区。                                                                                                                                                                                                                                                                                                                                        |
| `serviceTier`                                 | 未设置                                                  | 可选的 Codex app-server 服务层级。`"priority"` 启用快速模式路由，`"flex"` 请求弹性处理，`null` 清除覆盖值。旧版 `"fast"` 会按 `"priority"` 接受。                                                                                                                                                                                                 |
| `networkProxy`                                | 已禁用                                               | 选择启用 Codex 权限配置文件网络功能，以供 app-server 命令使用。OpenClaw 会定义选定的 `permissions.<profile>.network` 配置，并使用 `default_permissions` 选择该配置，而不是发送 `sandbox`。                                                                                                                                                                             |
| `experimental.sandboxExecServer`              | `false`                                                | 预览版选择启用功能，可向受支持的 Codex app-server 注册由 OpenClaw 沙箱支持的 Codex 环境，使原生 Codex 执行能够在当前 OpenClaw 沙箱内运行。                                                                                                                                                                                                            |

`appServer.networkProxy` 是显式配置，因为它会更改 Codex 沙箱契约。启用后，OpenClaw 还会在 Codex 线程配置中设置 `features.network_proxy.enabled` 和 `default_permissions`，以便生成的权限配置文件可以启动由 Codex 管理的网络。默认情况下，OpenClaw 会根据配置文件正文生成抗冲突的 `openclaw-network-<fingerprint>` 配置文件名称；仅当需要稳定的本地名称时，才使用 `profileName`。

```js
export default {
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            sandbox: "workspace-write",
            networkProxy: {
              enabled: true,
              domains: {
                "api.openai.com": "allow",
                "blocked.example.com": "deny",
              },
              allowUpstreamProxy: true,
              proxyUrl: "http://127.0.0.1:3128",
            },
          },
        },
      },
    },
  },
};
```

如果正常的应用服务器运行时原本为 `danger-full-access`，启用 `networkProxy` 后，生成的权限配置文件将改用工作区式文件系统访问。由 Codex 管理的网络强制策略属于沙箱网络，因此完全访问权限配置文件无法保护出站流量。

该插件会阻止使用较旧版本、尚未验证的较新版本、预发布版本、带构建后缀版本或未提供版本的应用服务器握手。Codex 应用服务器必须报告从 `0.143.0` 到内置 `0.145.0` 范围内的稳定版本。

OpenClaw 将非 loopback WebSocket 应用服务器 URL 视为远程地址，并要求通过 `appServer.authToken` 或 `Authorization` 标头进行包含身份信息的 WebSocket 身份验证。`appServer.authToken` 和每个 `appServer.headers.*` 值都可以是 SecretInput；在 OpenClaw 构建应用服务器启动选项之前，密钥运行时会解析 SecretRef 和环境变量简写，任何未解析的结构化 SecretRef 都会在发送任何令牌或标头之前导致失败。配置 Native Codex plugins 后，OpenClaw 会使用已连接应用服务器的插件控制平面安装或刷新这些插件，然后刷新应用清单，使归插件所有的应用对 Codex 线程可见。`app/list` 仍是权威的清单和元数据来源，但即使 Codex 当前将某个已列出且可访问的应用标记为已禁用，OpenClaw 策略仍会决定 `thread/start` 是否发送 `config.apps[appId].enabled = true`。未知或缺失的应用 ID 仍会以关闭方式失败；此路径仅通过 `plugin/install` 激活市场插件并刷新清单。仅将 OpenClaw 连接到可信的远程应用服务器，确保其可以接受由 OpenClaw 管理的插件安装和应用清单刷新。

## 审批和沙箱模式

本地 stdio 应用服务器会话默认使用 YOLO 模式：`approvalPolicy: "never"`、`approvalsReviewer: "user"` 和 `sandbox: "danger-full-access"`。这种受信任的本地操作员模式允许无人值守的 OpenClaw 轮次和 Heartbeat 持续推进，而不会出现无人可以响应的原生审批提示。

如果 Codex 的本地系统要求文件不允许隐式使用 YOLO 审批、审查者或沙箱值，OpenClaw 会改为将隐式默认值视为 guardian，并选择允许的 guardian 权限。`tools.exec.mode: "auto"` 也会强制使用 guardian 审查的 Codex 审批，并且不会保留不安全的旧版 `approvalPolicy: "never"` 或 `sandbox: "danger-full-access"` 覆盖；如需有意采用无需审批的模式，请设置 `tools.exec.mode: "full"`。在同一要求文件中，与主机名匹配的 `[[remote_sandbox_config]]` 条目会用于沙箱默认值决策。

设置 `appServer.mode: "guardian"` 以使用 Codex guardian 审查的审批：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            mode: "guardian",
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

当这些值被允许时，`guardian` 预设会展开为 `approvalPolicy: "on-request"`、`approvalsReviewer: "auto_review"` 和 `sandbox: "workspace-write"`。各个策略字段会覆盖 `mode`。较旧的 `guardian_subagent` 审查者值仍作为兼容别名被接受，但新配置应使用 `auto_review`。

当 OpenClaw 沙箱处于活动状态时，本地 Codex 应用服务器进程仍在 Gateway 网关主机上运行。因此，OpenClaw 会为该轮次禁用 Codex 原生代码模式、用户 MCP 服务器和由应用支持的插件执行，而不会将 Codex 主机侧的沙箱隔离视为等同于 OpenClaw 沙箱后端。当正常的 Exec/进程工具可用时，Shell 访问会通过由 OpenClaw 沙箱支持的动态工具（例如 `sandbox_exec` 和 `sandbox_process`）公开。

<Note>
在由 Docker 支持的 OpenClaw 沙箱主机上（`agents.defaults.sandbox.mode` 设置为 Docker 后端），`openclaw doctor` 会探测主机是否允许非特权用户命名空间，以及当 Docker 沙箱网络出口被禁用时是否允许网络命名空间；沙箱容器内嵌套的 Codex `bwrap` 需要这些命名空间来执行 `workspace-write` Shell。探测失败通常会在 Ubuntu/AppArmor 主机上显示为 `bwrap: setting up uid map: Permission denied` 或 `bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted`。请为 OpenClaw 服务用户修复报告的主机命名空间策略并重启 Gateway 网关；优先为服务进程使用限定范围的 AppArmor 配置文件，而不是使用作用于整个主机的 `kernel.apparmor_restrict_unprivileged_userns=0` 后备方案，并且不要仅为满足嵌套 `bwrap` 的要求而授予更广泛的 Docker 容器权限。
</Note>

## 沙箱化原生执行

稳定的默认行为是以关闭方式失败：启用 OpenClaw 沙箱隔离后，会禁用原本将在 Codex 应用服务器主机上运行的原生 Codex 执行界面。仅当需要尝试将 Codex 的远程环境支持与 OpenClaw 沙箱后端配合使用时，才使用 `appServer.experimental.sandboxExecServer: true`。此预览路径适用于所有受支持的 Codex 应用服务器版本。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            experimental: {
              sandboxExecServer: true,
            },
          },
        },
      },
    },
  },
}
```

启用此标志且当前 OpenClaw 会话处于沙箱隔离状态时，OpenClaw 会启动一个由活动沙箱支持的 local loopback Exec 服务器，将其注册到 Codex 应用服务器，并使用这个由 OpenClaw 所有的环境启动 Codex 线程和轮次。如果应用服务器无法注册该环境，运行将以关闭方式失败，而不会悄然回退到主机执行。

此预览路径仅限本地使用。除非远程 WebSocket 应用服务器在同一主机上运行，否则无法访问 loopback Exec 服务器，因此 OpenClaw 会拒绝这种组合。

## 身份验证和环境隔离

在默认的按 Agent 配置主目录中，身份验证按以下顺序选择：

1. 为该 Agent 显式配置的 OpenClaw Codex 身份验证配置文件。
2. 该 Agent 的 Codex 主目录中应用服务器的现有账户。
3. 仅对于本地 stdio 应用服务器启动，如果不存在应用服务器账户且仍需要 OpenAI 身份验证，则依次使用 `CODEX_API_KEY` 和 `OPENAI_API_KEY`。

当 OpenClaw 检测到 ChatGPT 订阅式 Codex 身份验证配置文件（OAuth 或令牌凭证类型）时，会从生成的 Codex 子进程中移除 `CODEX_API_KEY` 和 `OPENAI_API_KEY`。这样既能让 Gateway 网关级 API 密钥继续用于嵌入或直接调用 OpenAI 模型，又能避免原生 Codex 应用服务器轮次意外通过 API 计费。

显式 Codex API 密钥配置文件和本地 stdio 环境密钥后备方案会使用应用服务器登录，而不是继承子进程环境。WebSocket 应用服务器连接不会接收 Gateway 网关环境中的 API 密钥后备值；请使用显式身份验证配置文件或远程应用服务器自己的账户。

默认情况下，stdio 应用服务器启动会继承 OpenClaw 的进程环境。OpenClaw 负责 Codex 应用服务器账户桥接，并将 `CODEX_HOME` 设置为该 Agent 的 OpenClaw 状态目录下的按 Agent 配置目录。这样，Codex 配置、账户、插件缓存/数据和线程状态都会限定在 OpenClaw Agent 范围内，而不会从操作员的个人 `~/.codex` 主目录泄漏进来。

设置 `appServer.homeScope: "user"` 可与 Codex Desktop 和 CLI 共享原生 Codex 状态。此本地用户主目录模式支持托管 stdio 和显式 Unix 传输。设置 `$CODEX_HOME` 时使用该值，否则使用 `~/.codex`，包括原生身份验证、配置、插件和线程。OpenClaw 会跳过其针对应用服务器的身份验证配置文件桥接。经过验证的所有者轮次可以使用 `codex_threads` 列出（可选使用 `search` 筛选器）、读取、派生、重命名、归档和取消归档这些线程。在 OpenClaw 中继续线程之前，请先派生该线程；独立的 Codex 进程不会协调对同一线程的并发写入。

该 `homeScope` 选择加入项适用于普通 harness 会话。通过 Codex Sessions 创建的 Chat 则使用其专用监管连接，该连接会为规范分支和未来的恢复保留原生连接的身份验证和提供商配置。

在锁定模型的受监管 Chat 中，`codex_threads` 无法附加不同的派生线程，也无法归档 Chat 绑定的原生线程。列表和仅元数据读取仍然可用。读取原始转录内容需要 `allowRawTranscripts`；禁用后，列表搜索也会被拒绝，因为原生搜索可能匹配转录内容预览。重命名、取消归档、分离派生和归档不属于其他 OpenClaw Chat 的无关线程，需要 `allowWriteControls`。这两个选项都无法绕过锁定绑定。

OpenClaw 不会为正常的本地应用服务器启动重写 `HOME`。由 Codex 运行的子进程（例如 `openclaw`、`gh`、`git`、云 CLI 和 Shell 命令）会看到正常的进程主目录，并可以找到用户主目录中的配置和令牌。Codex 还可能发现 `$HOME/.agents/skills` 和 `$HOME/.agents/plugins/marketplace.json`；该 `.agents` 发现机制有意与操作员主目录共享，并且独立于隔离的 `~/.codex` 状态。

在默认的 Agent 范围内，OpenClaw 插件和 OpenClaw Skills 快照仍通过 OpenClaw 自己的插件注册表和 Skills 加载器流转；个人 Codex `~/.codex` 资产不会如此处理。如果 Codex 主目录中有实用的 Codex CLI Skills 或插件，需要将其纳入隔离的 OpenClaw Agent，请明确清点这些资产：

```bash
openclaw migrate codex --dry-run
openclaw migrate apply codex --yes
```

如果部署需要额外的环境隔离，请将这些变量添加到 `appServer.clearEnv`：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            clearEnv: ["CODEX_API_KEY", "OPENAI_API_KEY"],
          },
        },
      },
    },
  },
}
```

`appServer.clearEnv` 仅影响生成的 Codex 应用服务器子进程。OpenClaw 会在本地启动规范化期间从此列表中移除 `CODEX_HOME` 和 `HOME`：`CODEX_HOME` 会继续指向选定的 Agent 或用户范围，`HOME` 则继续被继承，使子进程可以使用正常的用户主目录状态。

## 动态工具

Codex 动态工具默认采用 `searchable` 加载方式，通过 `openclaw` 命名空间和 `deferLoading: true` 公开。OpenClaw 通常不会公开与 Codex 原生工作区操作或 Codex 自身工具搜索界面重复的动态工具：

- `read`
- `write`
- `edit`
- `apply_patch`
- `exec`
- `process`
- `update_plan`
- `tool_call`
- `tool_describe`
- `tool_search`
- `tool_search_code`

当有限的运行时允许列表禁用原生代码模式时，OpenClaw 会发送空的执行环境选择。在这种直接且非沙箱隔离的情况下，OpenClaw 会保留经过策略筛选的 `exec` 和 `process` 工具作为 Shell 后备方案。运行时允许列表和 `codexDynamicToolsExclude` 仍然适用。

其余大多数 OpenClaw 集成工具，例如消息传递、媒体、定时任务、
浏览器、节点、Gateway 网关、`heartbeat_respond` 和 `web_search`，均可
通过该命名空间下的 Codex 工具搜索使用。这样可以缩小初始模型
上下文。无论 `codexDynamicToolsLoading` 如何，仍有一小部分工具可直接
调用，因为 Codex 工具搜索可能不可用，或仅解析出由连接器组成的工具集合：
`agents_list`、`sessions_spawn` 和
`sessions_yield`。开发者指令仍会引导常规 Codex 子智能体
在处理 Codex 原生子智能体工作时使用原生 `spawn_agent`，而
`sessions_spawn` 仍可用于显式的 OpenClaw 或 ACP 委派。
仅含消息工具的源回复也仍采用直接方式，因为这是一个
轮次控制契约。

Codex 代码模式会将通用 OpenClaw 动态工具结果呈现为文本。读取字段前，请先解析
JSON 结果。嵌套动态调用由 Codex 运行时串行执行，因此
`Promise.all` 不会并发提交这些调用；启动收集器子项时，应使用
有界的顺序启动循环。

标记为 `catalogMode: "direct-only"` 的工具（包括 OpenClaw `computer`
工具）归入 `openclaw_direct`。OpenClaw 会将该命名空间添加到
Codex 的 `code_mode.direct_only_tool_namespaces` 列表中，而不会替换
操作员提供的条目。因此，在普通线程和仅限代码模式的线程中，Codex 会将这些工具公开为
`DirectModelOnly`，而不是通过嵌套的代码模式 `tools.*`
调用来路由。含图像的结果需要此边界：嵌套代码模式序列化会将图像输出扁平化为
文本，从而丢弃下一次计算机操作所需的屏幕截图。

仅在连接到无法搜索延迟动态工具的自定义
Codex app-server，或调试完整工具载荷时，才设置 `codexDynamicToolsLoading: "direct"`。

## 超时

OpenClaw 所有的动态工具调用具有独立于
`appServer.requestTimeoutMs` 的时限。每个 Codex `item/tool/call` 请求按以下
顺序使用第一个可用的超时：

- 每次调用中为正数的 `timeoutMs` 参数。
- 对于 `image_generate`，使用 `agents.defaults.mediaModels.image.timeoutMs`。
- 对于未配置超时的 `image_generate`，使用 120 秒的
  图像生成默认值。
- 对于媒体理解 `image` 工具，使用所选支持图像的 `tools.media.models[]` 条目的 `timeoutSeconds`
  转换后的毫秒值，或 60 秒的媒体默认值。对于图像
  理解，此值适用于请求本身，不会因之前的准备工作而缩短。
- 对于 `message` 工具，使用固定的 600 秒外层预算，其中涵盖 Gateway 网关投递和有界的同键协调。
- 使用 90 秒的动态工具默认值。

此看门狗是动态 `item/tool/call` 的外层预算。提供商特定的
请求超时在该调用内运行，并保留各自的超时语义。
动态工具预算上限为 600000 ms。`agents_wait` 会增加 30000 ms 的
外层完成宽限时间，而 app-server 客户端允许 660000 ms，以便
结构化等待结果能够传递给 Codex。发生超时时，OpenClaw 会在支持的情况下中止工具
信号，并向 Codex 返回失败的动态工具响应，使
轮次能够继续，而不是让会话停留在 `processing`。

Codex 接受一个轮次后，以及 OpenClaw 响应轮次范围内的
app-server 请求后，该 harness 会期望 Codex 在当前轮次中取得进展，
并最终以 `turn/completed` 完成原生轮次。如果
app-server 在 `appServer.turnCompletionIdleTimeoutMs` 期间保持静默，OpenClaw
会尽力中断 Codex 轮次，记录诊断超时，并
释放 OpenClaw 会话通道，避免后续聊天消息排在
已失效的原生轮次之后。

同一轮次的大多数非终止通知都会解除该短时看门狗，因为 Codex 已证明
该轮次仍处于活动状态。工具交接使用更长的
工具执行后空闲预算：在 OpenClaw 返回 `item/tool/call` 响应后、
在 `commandExecution` 等原生工具项完成后、在原始
`custom_tool_call_output` 完成后，以及在工具执行后的原始智能体
进度、原始推理完成或推理进度之后。若已配置，
该保护机制使用 `appServer.postToolRawAssistantCompletionIdleTimeoutMs`，
否则默认为五分钟。这一工具执行后预算还会延长
静默综合窗口的进度看门狗，直至 Codex 发出下一个
当前轮次事件。推理完成、commentary `agentMessage`
完成，以及工具执行前的原始推理或智能体进度之后，都可能跟随
自动最终回复，因此它们使用进度后回复保护机制，
而不是立即释放会话通道。只有最终/非 commentary
完成的 `agentMessage` 项和工具执行前的原始智能体完成会启动
智能体输出释放机制：如果 Codex 随后保持静默且未发出 `turn/completed`，
OpenClaw 会尽力中断原生轮次并释放会话
通道。可安全重放的 stdio app-server 故障，包括没有智能体、
工具、活动项或副作用证据的轮次完成空闲
超时，会在全新的 app-server 尝试中重试一次。不安全的超时仍会停用
卡住的 app-server 客户端并释放 OpenClaw 会话通道。它们还会
清除过期的原生线程绑定，而不会自动
重放。完成监视超时会显示 Codex 特定的超时文本：
可安全重放的情况会说明响应可能不完整，而不安全的情况会告知
用户在重试前验证当前状态。公开的超时诊断
包含结构化字段，例如上一个 app-server 通知方法、
原始智能体响应项的 id/type/role、活动请求/项数量，以及
已启用的监视状态。当最后一个通知是原始智能体响应
项时，诊断还会包含长度受限的智能体文本预览，但不会
包含原始提示词或工具内容。

## 模型发现

默认情况下，Codex 插件会向 app-server 查询可用模型。模型
可用性由 Codex app-server 管理，因此当
OpenClaw 升级内置的 `@openai/codex` 版本，或部署将
`appServer.command` 指向其他 Codex 二进制文件时，列表可能发生变化。可用性也可能
因账户而异。在运行中的 Gateway 网关上使用 `/codex models`，可查看该 harness 和账户的实时
目录。

如果发现失败或超时，OpenClaw 会使用内置的回退目录：

| 模型 id       | 显示名称 | 推理强度                 |
| -------------- | ------------ | ------------------------ |
| `gpt-5.5`      | gpt-5.5      | low, medium, high, xhigh |
| `gpt-5.4-mini` | GPT-5.4-Mini | low, medium, high, xhigh |

<Note>
当前内置的 harness 是 `@openai/codex` `0.145.0`。针对该内置 app-server 的 `model/list` 探测
返回了以下公开选择器行：

| 模型 id        | 输入模态 | 推理强度                             |
| --------------- | ---------------- | ------------------------------------ |
| `gpt-5.6-sol`   | text, image      | low, medium, high, xhigh, max, ultra |
| `gpt-5.6-terra` | text, image      | low, medium, high, xhigh, max, ultra |
| `gpt-5.6-luna`  | text, image      | low, medium, high, xhigh, max        |
| `gpt-5.5`       | text, image      | low, medium, high, xhigh             |
| `gpt-5.2`       | text, image      | low, medium, high, xhigh             |

app-server 目录可以报告 `ultra`；OpenClaw 推理控制目前
公开到 `max` 级别。

实时选择器行因账户而异，并可能随账户、Codex
目录或内置版本而变化；请运行 `/codex models` 获取当前列表，而不要
依赖任何特定时间点的表格。隐藏模型也可能出现在
app-server 目录中，用于内部或专用流程，而不是常规模型选择器选项。
</Note>

在 `plugins.entries.codex.config.discovery` 下调整发现设置：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
        },
      },
    },
  },
}
```

如果希望启动时避免探测 Codex，并且仅使用
回退目录，请禁用发现：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

## 工作区引导文件

Codex 通过原生项目文档发现自行处理 `AGENTS.md`。
OpenClaw 不会写入合成的 Codex 项目文档文件，也不依赖 Codex
回退文件名来获取角色文件，因为 Codex 回退仅在
`AGENTS.md` 缺失时适用。

为了与 OpenClaw 工作区保持一致，Codex harness 会将其他
引导文件作为开发者指令转发，但方式并不完全相同：

- `TOOLS.md` 会作为**继承的** Codex 开发者指令转发，因此
  在该轮次中生成的原生 Codex 子智能体也能看到它。
- `SOUL.md`、`IDENTITY.md` 和 `USER.md` 会作为**轮次范围内的**
  协作指令转发。原生 Codex 子智能体不会继承这些指令，
  从而避免子智能体轮次获取父智能体的角色设定和
  用户资料。
- 精简后的已加载 OpenClaw Skills 列表也会作为轮次范围内的
  协作开发者指令转发，因此原生 Codex 子智能体同样
  不会继承它。
- 不会注入 `HEARTBEAT.md` 内容；当该文件存在且
  非空时，Heartbeat 轮次会收到一条协作模式指针，要求读取该文件。
- 当该工作区可以使用记忆工具时，不会将已配置 Agent 工作区中的
  `MEMORY.md` 内容粘贴到原生 Codex 轮次输入中；如果该文件存在，harness 会向轮次范围内的协作开发者指令添加一个简短的工作区记忆
  指针，并且当持久记忆相关时，Codex
  应使用 `memory_search` 或 `memory_get`。
  如果工具已禁用、记忆搜索不可用，或活动
  工作区与 Agent 记忆工作区不同，则 `MEMORY.md` 会改用
  常规的有界轮次上下文路径。
- 如果存在 `BOOTSTRAP.md`，则会将其作为 OpenClaw 轮次输入参考
  上下文转发。

## 环境变量覆盖

环境变量覆盖仍可用于本地测试：

- `OPENCLAW_CODEX_APP_SERVER_BIN`
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_MODE=yolo|guardian`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`

当 `appServer.command` 未设置时，
`OPENCLAW_CODEX_APP_SERVER_BIN` 会绕过托管二进制文件。

`OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1` 已移除。请改用
`plugins.entries.codex.config.appServer.mode: "guardian"`，或使用
`OPENCLAW_CODEX_APP_SERVER_MODE=guardian` 进行一次性本地测试。对于可重复部署，
首选配置，因为它能将插件行为与 Codex harness 的其余设置保存在
同一个经过审查的文件中。

## 相关内容

- [Codex harness](/zh-CN/plugins/codex-harness)
- [Codex harness runtime](/zh-CN/plugins/codex-harness-runtime)
- [Codex 监管](/zh-CN/plugins/codex-supervision)
- [Native Codex plugins](/zh-CN/plugins/codex-native-plugins)
- [Codex Computer Use](/zh-CN/plugins/codex-computer-use)
- [OpenAI provider](/zh-CN/providers/openai)
- [配置参考](/zh-CN/gateway/configuration-reference)
