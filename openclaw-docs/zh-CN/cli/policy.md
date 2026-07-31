---
read_when:
    - 你想要根据编写的 `policy.jsonc` 检查 OpenClaw 设置
    - 你希望在 Doctor lint mode 中查看策略发现项
    - 你需要一个策略证明哈希作为审计证据
summary: '`openclaw policy` 一致性检查的 CLI 参考'
title: 策略
x-i18n:
    generated_at: "2026-07-26T05:44:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 63e4faeab8dd6535e3d517439d3f58cdc167b6b7fade808a6482742ec9b5acf1
    source_path: cli/policy.md
    workflow: 16
---

# `openclaw policy`

`openclaw policy` 由内置的 Policy 插件提供。它是基于现有 OpenClaw 设置的企业级合规层，而不是第二套配置系统。你在 `policy.jsonc` 中编写要求；OpenClaw 将活动工作区作为证据进行观测；Policy 通过 `doctor --lint` 报告偏差。Policy 不会在请求时强制执行工具调用或改写运行时行为，也不会证明 `auth-profiles.json` 等各智能体凭据存储的合规性。

Policy 检查已配置的渠道、MCP 服务器、模型提供商、网络 SSRF 防护态势、入口/渠道访问、Gateway 网关暴露情况和节点命令态势、编写的消息路由探针、Agent 工作区访问、沙箱态势、数据处理态势、机密信息提供商/身份验证配置文件态势，以及受治理的工具元数据（`TOOLS.md`）。当工作区需要“不得启用 Telegram”或“受治理的工具必须声明风险和所有者元数据”这类持久且可检查的声明时，请使用它。如果只需要本地行为，而不需要合规证明或偏差检测，普通配置就足够了。

## 快速开始

```bash
openclaw plugins enable policy
```

即使缺少 `policy.jsonc`，该插件也会保持启用，因此 Doctor 可以报告缺失的工件，而不是静默跳过检查。

请手动编写 `policy.jsonc`；它不会根据当前设置生成。每个顶层部分都是一个规则命名空间：只有其下存在具体规则时，检查才会运行（不支持的部分或键会以 `policy/policy-jsonc-invalid` 失败，而不是被静默忽略）。以下是覆盖所有受支持部分的最小示例：

```jsonc
{
  "channels": {
    "denyRules": [
      {
        "id": "no-telegram",
        "when": { "provider": "telegram" },
        "reason": "此工作区不允许使用 Telegram。",
      },
    ],
  },
  "mcp": {
    "servers": {
      "allow": ["docs"],
      "deny": ["untrusted"],
    },
  },
  "models": {
    "providers": {
      "allow": ["openai", "anthropic"],
      "deny": ["openrouter"],
    },
  },
  "network": {
    "privateNetwork": {
      "allow": false,
    },
  },
  "routing": {
    "requireBindings": true,
    "requireConfiguredChannels": true,
    "probes": [
      {
        "id": "family-dm",
        "route": {
          "channel": "imessage",
          "peer": { "kind": "direct", "id": "+15555550123" },
        },
        "expect": {
          "agentId": "family",
          "matchedBy": ["binding.peer"],
        },
      },
    ],
  },
  "ingress": {
    "session": {
      "requireDmScope": "per-channel-peer",
    },
    "channels": {
      "allowDmPolicies": ["pairing", "allowlist", "disabled"],
      "denyOpenGroups": true,
      "requireMentionInGroups": true,
    },
  },
  "gateway": {
    "exposure": {
      "allowNonLoopbackBind": false,
      "allowTailscaleFunnel": false,
    },
    "auth": {
      "requireAuth": true,
      "requireExplicitRateLimit": true,
    },
    "controlUi": {
      "allowInsecure": false,
    },
    "remote": {
      "allow": false,
    },
    "http": {
      "denyEndpoints": ["chatCompletions", "responses"],
      "requireUrlAllowlists": true,
    },
    "nodes": {
      "denyCommands": ["system.run"],
    },
  },
  "agents": {
    "workspace": {
      "allowedAccess": ["none", "ro"],
      "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
    },
  },
  "dataHandling": {
    "sensitiveLogging": {
      "requireRedaction": true,
    },
    "telemetry": {
      "denyContentCapture": true,
    },
    "retention": {
      "requireSessionMaintenance": true,
    },
    "memory": {
      "denySessionTranscriptIndexing": true,
    },
  },
  "secrets": {
    "requireManagedProviders": true,
    "denySources": ["exec"],
    "allowInsecureProviders": false,
  },
  "auth": {
    "profiles": {
      "requireMetadata": ["provider", "mode"],
      "allowModes": ["api_key", "token"],
    },
  },
  "execApprovals": {
    "requireFile": true,
    "defaults": { "allowSecurity": ["deny"] },
    "agents": {
      "allowSecurity": ["deny", "allowlist"],
      "allowAutoAllowSkills": false,
      "allowlist": { "expected": ["deploy", "status"] },
    },
  },
  "tools": {
    "requireMetadata": ["risk", "sensitivity", "owner"],
    "profiles": {
      "allow": ["messaging", "minimal"],
    },
    "fs": {
      "requireWorkspaceOnly": true,
    },
    "exec": {
      "allowSecurity": ["deny", "allowlist"],
      "requireAsk": ["always"],
      "allowHosts": ["sandbox"],
    },
    "elevated": {
      "allow": false,
    },
    "denyTools": ["group:runtime", "group:fs"],
  },
}
```

以下是下方规则表中未明确体现的跨领域注意事项：

- 在禁止非 local loopback 绑定时省略 `gateway.bind`，意味着你接受运行时默认值；如需严格合规，请设置 `gateway.bind: "loopback"`。
- 对于只读智能体，请在适用的默认设置/智能体中，将沙箱 `mode` 设置为 `all` 或 `non-main`，并将 `workspaceAccess` 设置为 `none` 或 `ro`。缺失或设为 `off` 的沙箱模式不符合只读策略。
- `agents.workspace.denyTools` 接受 `exec`、`process`、`write`、`edit`、`apply_patch`。配置中的工具拒绝组 `group:fs`（文件修改）和 `group:runtime`（shell/进程）可满足等效态势。
- 仅当存在 `execApprovals` 规则时，Exec 审批检查才会读取实时的 `exec-approvals.json` 工件；缺失或无效的工件属于无法观测的证据，不会被视为人为构造的通过结果。
- 机密信息和身份验证配置文件证据仅记录提供商/来源态势及 SecretRef 元数据，绝不记录原始值。Policy 不会读取或证明 `auth-profiles.json` 等各智能体凭据存储的合规性。
- 数据处理证据仅反映配置层面的态势（脱敏模式、遥测捕获开关、会话维护模式、转录文本索引设置）。它不会检查日志、遥测导出、转录文本或记忆文件；即使结果无异常，也不能证明其中不存在个人数据或机密信息。
- 路由探针复用 OpenClaw 的运行时绑定解析器。路由证据仅记录探针 ID、解析得到的智能体、匹配类型及已脱敏的绑定元数据。它绝不会记录对等方、账户、服务器、团队或角色标识符。添加路由部分会有意改变策略和证明哈希；不含路由的策略会保留其现有证据结构。

### Policy 规则参考

以下每条规则均为可选；只有规则存在时，检查才会运行。观测到的状态来自现有 OpenClaw 配置或工作区元数据。

#### 作用域覆盖层

当特定智能体或渠道需要比顶层基准更严格的策略时，请使用 `scopes.<scopeName>`。作用域名称只是标签；匹配使用作用域内的选择器。覆盖层是累加式的：全局规则仍会运行，而作用域规则可以针对同一证据添加自己的发现项。

| 选择器     | 支持的部分                                                             | 适用场景                                          |
| ------------ | ------------------------------------------------------------------------------ | ------------------------------------------------- |
| `agentIds`   | `tools`、`agents.workspace`、`sandbox`、`dataHandling.memory`、`execApprovals` | 一个或多个运行时智能体需要更严格的规则。   |
| `channelIds` | `ingress.channels`                                                             | 一个或多个渠道需要更严格的入口规则。 |

如果 `agentIds` 条目不在 `agents.entries.*` 中，OpenClaw 会依据该运行时智能体 ID 继承的全局/默认态势评估作用域规则，而不会跳过它。

```jsonc
{
  "tools": {
    "exec": {
      "allowHosts": ["sandbox", "node"],
    },
  },
  "sandbox": {
    "requireMode": ["all", "non-main"],
  },
  "scopes": {
    "release-workspace": {
      "agentIds": ["release-agent", "review-agent"],
      "agents": {
        "workspace": {
          "allowedAccess": ["none", "ro"],
        },
      },
    },
    "release-lockdown": {
      "agentIds": ["release-agent"],
      "tools": {
        "exec": {
          "allowHosts": ["sandbox"],
          "allowSecurity": ["deny", "allowlist"],
          "requireAsk": ["always"],
        },
        "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
      },
      "sandbox": {
        "requireMode": ["all"],
        "allowBackends": ["docker"],
      },
      "dataHandling": {
        "memory": {
          "denySessionTranscriptIndexing": true,
        },
      },
    },
    "shell-sandbox": {
      "agentIds": ["shell-agent"],
      "sandbox": {
        "allowBackends": ["openshell"],
        "containers": {
          "requireReadOnlyMounts": false,
        },
      },
    },
    "telegram-ingress": {
      "channelIds": ["telegram"],
      "ingress": {
        "channels": {
          "allowDmPolicies": ["pairing"],
          "denyOpenGroups": true,
          "requireMentionInGroups": true,
        },
      },
    },
  },
}
```

如上所示，如果每个作用域管理不同的字段，同一智能体可以出现在多个作用域中。对于同一智能体，重复的作用域字段必须具有同等或更严格的限制；较宽松的重复声明会被拒绝（允许列表必须是子集，拒绝列表必须是超集，必需的布尔值则是固定的）。

容器态势规则（`sandbox.containers.*`）仅针对匹配智能体的沙箱后端能够提供的证据进行检查。如果后端无法观测你为其启用的规则，Policy 会报告 `policy/sandbox-container-posture-unobservable`，而不是判定通过；请将容器规则的作用域限定为使用能够提供相应证据的后端的智能体组。

顶层 `ingress.session.requireDmScope` 保持全局有效；`session.dmScope` 不是可归因于渠道的证据，因此无法通过 `channelIds` 限定作用域。

`policy.jsonc` 中存在的每个作用域都必须有效且可执行。

#### 渠道

| 策略字段                         | 观测状态                          | 适用场景                                                     |
| ------------------------------------ | --------------------------------------- | ------------------------------------------------------------ |
| `channels.denyRules[].when.provider` | `channels.*` 提供商和启用状态 | 禁止来自 `telegram` 等提供商的已配置渠道。 |
| `channels.denyRules[].reason`        | 发现消息和修复提示上下文 | 说明为何禁止该提供商。                          |

#### MCP 服务器

| 策略字段        | 观测状态      | 适用场景                                                   |
| ------------------- | ------------------- | ---------------------------------------------------------- |
| `mcp.servers.allow` | `mcp.servers.*` ID | 要求每个已配置的 MCP 服务器都位于允许列表中。 |
| `mcp.servers.deny`  | `mcp.servers.*` ID | 禁止特定的已配置 MCP 服务器 ID。                   |

#### 模型提供商

| 策略字段             | 观测状态                                   | 适用场景                                                                        |
| ------------------------ | ------------------------------------------------ | ------------------------------------------------------------------------------- |
| `models.providers.allow` | `models.providers.*` ID 和选定的模型引用 | 要求已配置的提供商和选定的模型引用使用获批的提供商。 |
| `models.providers.deny`  | `models.providers.*` ID 和选定的模型引用 | 按提供商 ID 禁止已配置的提供商和选定的模型引用。               |

#### 网络

| 策略字段                   | 观测状态                      | 适用场景                                                           |
| ------------------------------ | ----------------------------------- | ------------------------------------------------------------------ |
| `network.privateNetwork.allow` | 专用网络 SSRF 绕过机制 | 设为 `false`，以要求专用网络访问保持禁用。 |

#### 消息路由

| 策略字段                        | 观测状态                                      | 适用场景                                                               |
| ----------------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------- |
| `routing.requireBindings`           | 渠道路由绑定，不包括 ACP 绑定      | 要求至少有一个消息路由绑定。                          |
| `routing.requireConfiguredChannels` | 绑定渠道 ID 和已配置的 `channels.*` ID | 检测过时或拼写错误的绑定渠道 ID。                        |
| `routing.probes[].route`            | OpenClaw 公共路由解析器                  | 描述一个有代表性的入站路由，而不发送消息。     |
| `routing.probes[].expect.agentId`   | 已解析的智能体 ID                                   | 要求路由到达已审核的智能体。                         |
| `routing.probes[].expect.matchedBy` | 解析器匹配类型                                 | 要求对等方、账户、渠道或其他已审核绑定具有明确的匹配范围。 |

探测 ID 必须唯一。路由支持 `channel`、可选的 `accountId`、
`peer`、`parentPeer`、`guildId`、`teamId` 和 `memberRoleIds`。对等方类型包括
`direct`、`group` 和 `channel`。`matchedBy` 可包含一个或多个运行时
匹配类型，包括 `binding.peer`、`binding.account`、`binding.channel`
或 `default`。

路由检查仅用于合规性检查。它们不会更改启动、
消息传递、绑定优先级或回退行为。发现项需要
操作员审核，因为自动更改绑定可能会将
私信重定向到其他位置。

#### 入口和渠道访问

| 策略字段                              | 观测状态                                                 | 适用场景                                                           |
| ----------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------ |
| `ingress.session.requireDmScope`          | `session.dmScope`                                              | 要求使用已审核的私信隔离范围。                 |
| `ingress.channels.allowDmPolicies`        | `channels.*.dmPolicy` 和旧版渠道私信策略字段      | 仅允许已审核的私信渠道策略。               |
| `ingress.channels.denyOpenGroups`         | 渠道、账户和群组入口策略                     | 拒绝已配置渠道和账户的开放群组入口。      |
| `ingress.channels.requireMentionInGroups` | 渠道、账户、群组、服务器和嵌套提及门控配置 | 当群组入口开放或受提及门控时，要求启用提及门控。 |

#### Gateway 网关

| 策略字段                            | 观测状态                                 | 适用场景                                                                             |
| --------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------ |
| `gateway.exposure.allowNonLoopbackBind` | `gateway.bind`                                 | 设为 `false`，以要求 Gateway 网关绑定到环回地址。                                  |
| `gateway.exposure.allowTailscaleFunnel` | Tailscale serve/funnel Gateway 网关安全态势         | 设为 `false`，以拒绝 Tailscale Funnel 暴露。                                    |
| `gateway.auth.requireAuth`              | `gateway.auth.mode`                            | 设为 `true`，以拒绝禁用 Gateway 网关身份验证。                                       |
| `gateway.auth.requireExplicitRateLimit` | `gateway.auth.rateLimit`                       | 设为 `true`，以要求显式配置身份验证速率限制。                            |
| `gateway.controlUi.allowInsecure`       | Control UI 不安全的身份验证、设备或来源开关 | 设为 `false`，以拒绝不安全的 Control UI 暴露开关。                         |
| `gateway.remote.allow`                  | 远程 Gateway 网关模式/配置                     | 设为 `false`，以拒绝远程 Gateway 网关模式。                                          |
| `gateway.http.denyEndpoints`            | Gateway 网关 HTTP API 端点                     | 拒绝 `chatCompletions` 或 `responses` 等端点 ID。                          |
| `gateway.http.requireUrlAllowlists`     | Gateway 网关 HTTP URL 获取输入                  | 设为 `true`，以要求 URL 获取输入使用 URL 允许列表。                         |
| `gateway.nodes.denyCommands`            | `gateway.nodes.commands.deny`                  | 要求在 OpenClaw 配置中明确拒绝 `system.run` 等节点命令 ID。 |

`gateway.nodes.denyCommands` 是一种精确且区分大小写的策略拒绝超集规则。
当策略必须证明 OpenClaw 配置显式拒绝了特权节点命令时，
使用此规则。如果部署有意允许某个特权
节点命令，应在审核后更新 `policy.jsonc`，而不是仅依赖
`gateway.nodes.commands.allow`。

#### Agent 工作区

| 策略字段                     | 观测状态                                                                           | 适用场景                                                                                 |
| -------------------------------- | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `agents.workspace.allowedAccess` | `agents.defaults.sandbox.workspaceAccess` 和 `agents.entries.*.sandbox.workspaceAccess` | 仅允许 `none` 或 `ro` 等沙箱工作区访问值。                       |
| `agents.workspace.denyTools`     | 全局和按智能体配置的工具拒绝设置                                                    | 要求拒绝变更工具（`exec`、`process`、`write`、`edit`、`apply_patch`）。 |

#### 沙箱安全态势

| 策略字段                                          | 观测状态                                          | 适用场景                                                       |
| ----------------------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------- |
| `sandbox.requireMode`                                 | `agents.defaults.sandbox.mode` 和按智能体配置的模式       | 仅允许 `all` 或 `non-main` 等已审核的沙箱模式。 |
| `sandbox.allowBackends`                               | `agents.defaults.sandbox.backend` 和按智能体配置的后端 | 仅允许 `docker` 等已审核的沙箱后端。         |
| `sandbox.containers.denyHostNetwork`                  | 容器支持的沙箱/浏览器网络模式           | 拒绝主机网络模式。                                        |
| `sandbox.containers.denyContainerNamespaceJoin`       | 容器支持的沙箱/浏览器网络模式           | 拒绝加入其他容器的网络命名空间。              |
| `sandbox.containers.requireReadOnlyMounts`            | 容器支持的沙箱/浏览器挂载模式             | 要求挂载为只读。                                |
| `sandbox.containers.denyContainerRuntimeSocketMounts` | 容器支持的沙箱/浏览器挂载目标          | 拒绝挂载容器运行时套接字。                          |
| `sandbox.containers.denyUnconfinedProfiles`           | 容器安全配置文件安全态势                      | 拒绝不受约束的容器安全配置文件。                   |
| `sandbox.browser.requireCdpSourceRange`               | 沙箱浏览器 CDP 来源范围                        | 要求浏览器 CDP 暴露声明来源范围。        |

策略将缺失的 `sandbox.mode` 视为其隐式默认值 `off`，因此
`sandbox.requireMode` 会将全新或未配置的沙箱报告为不在
`["all"]` 等允许列表中。

#### 数据处理

| 策略字段                                        | 观测状态                                                                                     | 适用场景                                                               |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `dataHandling.sensitiveLogging.requireRedaction`    | `logging.redactSensitive`                                                                          | 设为 `true`，以拒绝 `logging.redactSensitive: "off"`。              |
| `dataHandling.telemetry.denyContentCapture`         | `diagnostics.otel.captureContent`                                                                  | 设为 `true`，以拒绝捕获遥测内容。                     |
| `dataHandling.retention.requireSessionMaintenance`  | `session.maintenance.mode`                                                                         | 设为 `true`，以要求有效的会话维护模式为 `enforce`。 |
| `dataHandling.memory.denySessionTranscriptIndexing` | `memory.qmd.sessions.enabled`、`memory.search.experimental.sessionMemory` 和按智能体配置的覆盖值 | 设为 `true`，以拒绝将会话记录索引到记忆中。       |

#### 机密信息

| 策略字段                      | 观测状态                                           | 适用场景                                                                |
| --------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------- |
| `secrets.requireManagedProviders` | 配置 SecretRef 和 `secrets.providers.*` 声明 | 设为 `true`，以要求 SecretRef 指向已声明的提供商。     |
| `secrets.denySources`             | 机密信息提供商来源和 SecretRef 来源            | 拒绝 `exec`、`file` 或其他已配置来源名称等来源。 |
| `secrets.allowInsecureProviders`  | 不安全的机密信息提供商安全态势标志                   | 设为 `false`，以拒绝选择使用不安全安全态势的提供商。      |

#### Exec 审批

Exec 审批检查会读取运行时 `exec-approvals.json` 工件：
默认为 `~/.openclaw/exec-approvals.json`，设置 `OPENCLAW_STATE_DIR` 时则为
`$OPENCLAW_STATE_DIR/exec-approvals.json`。
`execApprovals.defaults.*` 或 `execApprovals.agents.*` 下的安全态势规则
要求提供可读的工件证据；缺失或无效的工件会报告为
无法观测的证据，而不是尽力而为地判定通过。工件可读后，省略的
字段将继承运行时默认值：缺失的 `defaults.security` 为 `full`，并且
缺失的智能体安全配置会继承该默认值。证据包括 `defaults`、
`agents.*`、`agents.*.allowlist[].pattern`、可选的 `argPattern`、有效的
`autoAllowSkills` 安全态势和条目来源，但绝不包括套接字路径/令牌、
`commandText`、`lastUsedCommand`、已解析路径或时间戳。

| 策略字段                                | 观测到的状态                                                                         | 适用场景                                                                                |
| ------------------------------------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `execApprovals.requireFile`                 | 活跃运行时 `exec-approvals.json` 路径                                              | 设置为 `true`，以要求审批工件存在且可解析。                     |
| `execApprovals.defaults.allowSecurity`      | `defaults.security`，默认为 `full`                                              | 仅允许已批准的默认审批安全模式。                                    |
| `execApprovals.agents.allowSecurity`        | `agents.*.security`，继承默认值                                               | 仅允许已批准的每智能体有效审批安全模式。                        |
| `execApprovals.agents.allowAutoAllowSkills` | `defaults.autoAllowSkills` 和 `agents.*.autoAllowSkills`，继承运行时默认值 | 设置为 `false`，以要求使用严格的手动允许列表，且不隐式批准 Skills CLI。 |
| `execApprovals.agents.allowlist.expected`   | 聚合的 `agents.*.allowlist[]` 模式和可选的 argPattern 条目               | 要求审批允许列表与已审查的模式集匹配。                      |

示例：要求审批工件存在、拒绝宽松的默认值，并且仅允许
所选智能体使用经过审查的 Exec 审批安全策略。

```jsonc
{
  "execApprovals": {
    "requireFile": true,
    "defaults": {
      // 安全模式："deny"、"allowlist" 或 "full"。
      // 此默认值仅允许锁定的拒绝安全策略。
      "allowSecurity": ["deny"],
    },
  },
  "scopes": {
    "restricted-shell": {
      "agentIds": ["family-agent", "groups-agent"],
      "execApprovals": {
        "agents": {
          // 所选智能体可以使用经过审查的允许列表安全策略，但不能使用 "full"。
          "allowSecurity": ["allowlist"],
          // false 表示 Skills CLI 必须出现在经过审查的允许列表中，而不是
          // 由 autoAllowSkills 隐式批准。
          "allowAutoAllowSkills": false,
          "allowlist": {
            "expected": [
              // 简单条目：经过审查的精确可执行文件模式，不含 argPattern。
              "travel-hub",
              // 受限条目：模式加上经过审查的参数正则表达式。
              { "pattern": "calendar-cli", "argPattern": "^sync\\b" },
              "/bin/date",
            ],
          },
        },
      },
    },
  },
}
```

#### 身份验证配置文件

| 策略字段                    | 观测到的状态                               | 适用场景                                                                                   |
| ------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `auth.profiles.requireMetadata` | `auth.profiles.*` 提供商和模式元数据 | 要求配置身份验证配置文件包含 `provider` 和 `mode` 等元数据键。               |
| `auth.profiles.allowModes`      | `auth.profiles.*.mode`                       | 仅允许 `api_key`、`aws-sdk`、`oauth` 或 `token` 等受支持的身份验证配置文件模式。 |

#### 工具元数据

| 策略字段            | 观测到的状态                   | 适用场景                                                                                   |
| ----------------------- | -------------------------------- | ------------------------------------------------------------------------------------------ |
| `tools.requireMetadata` | 受管控的 `TOOLS.md` 声明 | 要求受管控工具声明 `risk`、`sensitivity` 或 `owner` 等元数据键。 |

#### 工具安全策略

| 策略字段                    | 观测到的状态                                              | 适用场景                                                                                                 |
| ------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `tools.profiles.allow`          | `tools.profile` 和 `agents.entries.*.tools.profile`        | 仅允许 `minimal`、`messaging` 或 `coding` 等工具配置文件 ID。                                 |
| `tools.fs.requireWorkspaceOnly` | `tools.fs.workspaceOnly` 和每智能体的 `tools.fs` 覆盖项 | 设置为 `true`，以要求文件系统工具采用仅限工作区的安全策略。                                         |
| `tools.exec.allowSecurity`      | `tools.exec.security` 和每智能体的 Exec 安全模式           | 仅允许 `deny` 或 `allowlist` 等 Exec 安全模式。                                            |
| `tools.exec.requireAsk`         | `tools.exec.ask` 和每智能体的 Exec 询问模式                | 要求采用 `always` 等审批安全策略。                                                               |
| `tools.exec.allowHosts`         | `tools.exec.host` 和每智能体的 Exec 主机路由           | 仅允许 `sandbox` 等 Exec 主机路由模式。                                                    |
| `tools.elevated.allow`          | `tools.elevated.enabled` 和每智能体的提升权限安全策略     | 设置为 `false`，以要求提升权限的工具模式保持禁用。                                           |
| `tools.alsoAllow.expected`      | `tools.alsoAllow` 和每智能体的 `tools.alsoAllow`           | 要求精确匹配 `alsoAllow` 条目，并报告缺失或意外增加的工具授权。                 |
| `tools.denyTools`               | `tools.deny` 和 `agents.entries.*.tools.deny`              | 要求已配置的工具拒绝列表包含 `group:runtime` 和 `group:fs` 等工具 ID 或组。 |

## 运行检查

编写期间运行仅策略检查：

```bash
openclaw policy check
openclaw policy check --json
openclaw policy check --severity-min error
```

`policy check` 仅运行策略检查集，并生成证据、发现项
和证明哈希。启用 Policy 插件后，相同的发现项也会出现在
`openclaw doctor --lint` 中。

将操作员策略文件与编写的基准进行比较：

```bash
openclaw policy compare --baseline official.policy.jsonc
openclaw policy compare --baseline official.policy.jsonc --policy policy.jsonc --json
```

`policy compare` 根据策略文件语法检查策略文件语法；它
不会检查运行时状态、证据、凭据或密钥。它使用与作用域覆盖项相同的
规则元数据：允许列表必须保持相同或进一步收窄，
拒绝列表必须保持相同或进一步扩大，必需的布尔值必须保持
其值，有序字符串只能向已配置顺序中更严格的一端移动，
精确列表必须匹配。基准可以是
组织编写的策略；被检查的策略可以添加更严格的值或
额外规则。顶层被检查规则如果同样严格或更加严格，
可以满足有作用域的基准规则。文件之间的作用域名称无需
匹配；比较依据选择器（`agentIds`/`channelIds`）和字段进行。
对于路由探测，每个基准探测 ID 都必须保留相同的路由
和预期智能体。被检查的策略可以添加探测或收窄 `matchedBy`，但
移除探测、更改其路由或智能体，或扩大其接受的匹配
种类，都会使策略变弱。

无差异比较（`--json`）：

```json
{
  "ok": true,
  "baselinePath": "official.policy.jsonc",
  "policyPath": "policy.jsonc",
  "rulesChecked": 3,
  "findings": []
}
```

无发现项的 `policy check --json` 输出包含操作员或
监管者可记录的稳定哈希：

```json
{
  "ok": true,
  "attestation": {
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "checksRun": 5,
  "checksSkipped": 0,
  "findings": []
}
```

## 配置策略

策略配置位于 `plugins.entries.policy.config` 下。

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "enabled": true,
        "config": {
          "enabled": true,
          "path": "policy.jsonc",
          "workspaceRepairs": false,
          "expectedHash": "sha256:...",
          "expectedAttestationHash": "sha256:...",
        },
      },
    },
  },
}
```

| 设置                   | 用途                                                         |
| ------------------------- | --------------------------------------------------------------- |
| `enabled`                 | 即使 `policy.jsonc` 尚不存在，也启用策略检查。         |
| `workspaceRepairs`        | 允许 `doctor --fix` 编辑由策略管理的工作区设置。 |
| `expectedHash`            | 已批准策略工件的可选哈希锁。            |
| `expectedAttestationHash` | 最近一次获接受的无发现项策略检查的可选哈希锁。    |
| `path`                    | 策略工件相对于工作区的位置。             |

将 `plugins.entries.policy.config.enabled` 设置为 `false`，可在保留插件安装的同时
禁用工作区的策略检查。

## 接受策略状态

JSON 输出示例：

```json
{
  "ok": true,
  "attestation": {
    "checkedAt": "2026-05-10T20:00:00.000Z",
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "evidence": {
    "channels": [
      {
        "id": "telegram",
        "provider": "telegram",
        "source": "oc://openclaw.config/channels/telegram",
        "enabled": false
      }
    ],
    "mcpServers": [
      {
        "id": "docs",
        "transport": "stdio",
        "source": "oc://openclaw.config/mcp/servers/docs",
        "command": "npx"
      }
    ],
    "modelProviders": [
      {
        "id": "openai",
        "source": "oc://openclaw.config/models/providers/openai"
      }
    ],
    "modelRefs": [
      {
        "ref": "openai/gpt-5.6-sol",
        "provider": "openai",
        "model": "gpt-5.6-sol",
        "source": "oc://openclaw.config/agents/defaults/model"
      }
    ],
    "network": [
      {
        "id": "browser-private-network",
        "source": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
        "value": false
      }
    ],
    "gatewayExposure": [
      {
        "id": "gateway-bind",
        "kind": "bind",
        "source": "oc://openclaw.config/gateway/bind",
        "value": "loopback",
        "nonLoopback": false,
        "explicit": true
      }
    ],
    "agentWorkspace": [
      {
        "id": "agents-defaults-workspace-access",
        "kind": "workspaceAccess",
        "source": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
        "scope": "defaults",
        "value": "ro",
        "sandboxMode": "all",
        "sandboxModeSource": "oc://openclaw.config/agents/defaults/sandbox/mode",
        "sandboxEnabled": true,
        "explicit": true
      },
      {
        "id": "agents-defaults-tool-exec",
        "kind": "toolDeny",
        "source": "oc://openclaw.config/tools/deny",
        "scope": "defaults",
        "tool": "exec",
        "denied": true,
        "explicit": true
      }
    ],
    "secrets": [
      {
        "id": "vault",
        "kind": "provider",
        "source": "oc://openclaw.config/secrets/providers/vault",
        "providerSource": "env"
      },
      {
        "id": "oc://openclaw.config/models/providers/openai/apiKey",
        "kind": "input",
        "source": "oc://openclaw.config/models/providers/openai/apiKey",
        "provenance": "secretRef",
        "refSource": "env",
        "refProvider": "vault"
      }
    ],
    "authProfiles": [
      {
        "id": "github",
        "source": "oc://openclaw.config/auth/profiles/github",
        "validMetadata": true,
        "provider": "github",
        "mode": "token"
      }
    ],
    "tools": [
      {
        "id": "deploy",
        "source": "oc://TOOLS.md/tools/deploy",
        "line": 12,
        "risk": "critical",
        "sensitivity": "restricted",
        "capabilities": ["IRREVERSIBLE_EXTERNAL"]
      }
    ]
  },
  "checksRun": 30,
  "checksSkipped": 0,
  "findings": []
}
```

`attestation.policy.hash` 标识编写的规则工件。`evidence`
记录检查所使用的已观测 OpenClaw 状态，而
`workspace.hash` 标识该证据载荷。`findingsHash` 标识
确切的发现集。`checkedAt` 记录检查的运行时间。
`attestationHash` 标识稳定声明（策略哈希、证据哈希、
发现哈希以及干净/脏状态），并有意排除 `checkedAt`，
因此，相同的策略状态始终会生成相同的证明哈希。这四个值共同
构成一次策略检查的审计元组。

如果 Gateway 网关或监督程序使用策略来阻止、批准运行时操作或为其添加注释，
它应记录上一次干净检查的证明哈希。`checkedAt` 会保留在 JSON
输出中供审计日志使用，但不属于稳定哈希的一部分。

接受策略状态的生命周期：

1. 编写或审查 `policy.jsonc`。
2. 运行 `openclaw policy check --json`。
3. 如果检查干净，将 `attestation.policy.hash` 记录为 `expectedHash`。
4. 将 `attestation.attestationHash` 记录为 `expectedAttestationHash`。
5. 在 CI 或发布门禁中重新运行 `openclaw doctor --lint`。

如果有意更改策略规则，请使用一次干净检查的结果更新两个已接受的哈希。
如果仅工作区设置发生变化（策略保持不变），通常只有
`expectedAttestationHash` 会发生变化。

启用或升级 `agents.workspace` 规则会将 `agentWorkspace` 证据
添加到工作区哈希和证明哈希中；启用后，请审查新证据并
刷新已接受的证明哈希。启用或升级工具安全态势规则时，也会以相同方式
添加 `toolPosture` 证据。

`openclaw policy watch` 会重新运行检查，并在当前证据不再
匹配 `expectedAttestationHash` 时报告：

```bash
openclaw policy watch --json
```

在需要执行一次漂移评估的 CI 或脚本中使用 `--once`。如果未指定
`--once`，默认每两秒轮询一次；使用 `--interval-ms` 可更改
轮询间隔。

## 发现

| 检查 ID                                                 | 发现                                                                           |
| -------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `policy/policy-jsonc-missing`                            | 策略已启用，但缺少 `policy.jsonc`。                                  |
| `policy/policy-jsonc-invalid`                            | 无法解析策略，或策略包含格式错误的规则条目。                       |
| `policy/policy-hash-mismatch`                            | 策略与已配置的 `expectedHash` 不匹配。                                  |
| `policy/attestation-hash-mismatch`                       | 当前策略证据不再与已接受的证明匹配。               |
| `policy/policy-conformance-invalid`                      | 基准策略文件或受检策略文件包含无效的比较语法。                  |
| `policy/policy-conformance-missing`                      | 受检策略文件缺少基准策略文件要求的规则。     |
| `policy/policy-conformance-weaker`                       | 受检策略文件中的值弱于基准策略文件中的值。           |
| `policy/channels-denied-provider`                        | 已启用的渠道与渠道拒绝规则匹配。                                   |
| `policy/mcp-denied-server`                               | 已配置的 MCP 服务器被策略拒绝。                                      |
| `policy/mcp-unapproved-server`                           | 已配置的 MCP 服务器不在允许列表中。                                 |
| `policy/models-denied-provider`                          | 已配置的模型提供商或模型引用使用了被拒绝的提供商。                  |
| `policy/models-unapproved-provider`                      | 已配置的模型提供商或模型引用不在允许列表中。                |
| `policy/network-private-access-enabled`                  | 策略拒绝私有网络 SSRF 逃生开关，但该开关已启用。             |
| `policy/routing-bindings-required`                       | 策略要求配置渠道路由绑定，但当前未配置。                  |
| `policy/routing-binding-channel-unconfigured`            | 路由绑定指定了一个不存在于 `channels.*` 中的渠道。                         |
| `policy/routing-agent-mismatch`                          | 编写的路由解析到了不同的智能体。                                  |
| `policy/routing-match-kind-mismatch`                     | 编写的路由以非预期的绑定特异性匹配。                   |
| `policy/ingress-dm-policy-unapproved`                    | 渠道私信策略不在策略允许列表中。                              |
| `policy/ingress-dm-scope-unapproved`                     | `session.dmScope` 与策略要求的私信隔离范围不匹配。          |
| `policy/ingress-open-groups-denied`                      | 渠道群组策略为 `open`，但策略拒绝开放群组入口。          |
| `policy/ingress-group-mention-required`                  | 渠道或群组条目禁用了提及门控，但策略要求启用。       |
| `policy/gateway-non-loopback-bind`                       | 策略拒绝非回环暴露，但 Gateway 网关绑定配置允许这种暴露。         |
| `policy/gateway-auth-disabled`                           | 策略要求身份验证，但 Gateway 网关身份验证已禁用。                     |
| `policy/gateway-rate-limit-missing`                      | 策略要求明确配置 Gateway 网关身份验证速率限制，但当前配置不明确。          |
| `policy/gateway-control-ui-insecure`                     | Gateway 网关 Control UI 的不安全暴露开关已启用。                         |
| `policy/gateway-tailscale-funnel`                        | 策略拒绝 Gateway 网关 Tailscale Funnel 暴露，但该暴露已启用。               |
| `policy/gateway-remote-enabled`                          | 策略拒绝 Gateway 网关远程模式，但该模式处于活动状态。                              |
| `policy/gateway-http-endpoint-enabled`                   | 策略拒绝某个 Gateway 网关 HTTP API 端点，但该端点已启用。                    |
| `policy/gateway-http-url-fetch-unrestricted`             | Gateway 网关 HTTP URL 获取输入缺少必需的 URL 允许列表。                      |
| `policy/gateway-node-command-denied`                     | 策略拒绝的节点命令未被 OpenClaw 配置拒绝。                 |
| `policy/agents-workspace-access-denied`                  | 智能体沙箱模式或工作区访问权限不在策略允许列表中。           |
| `policy/agents-tool-not-denied`                          | 智能体配置或默认配置未拒绝策略要求拒绝的工具。               |
| `policy/tools-profile-unapproved`                        | 已配置的全局或按智能体工具配置文件不在允许列表中。           |
| `policy/tools-fs-workspace-only-required`                | 文件系统工具未配置为仅限工作区的路径安全策略。             |
| `policy/tools-exec-security-unapproved`                  | Exec 安全模式不在策略允许列表中。                               |
| `policy/tools-exec-ask-unapproved`                       | Exec 询问模式不在策略允许列表中。                                    |
| `policy/tools-exec-host-unapproved`                      | Exec 主机路由不在策略允许列表中。                                |
| `policy/tools-elevated-enabled`                          | 策略拒绝提升权限的工具模式，但该模式已启用。                              |
| `policy/tools-also-allow-missing`                        | 已配置的 `alsoAllow` 列表缺少策略要求的条目。             |
| `policy/tools-also-allow-unexpected`                     | 已配置的 `alsoAllow` 列表包含策略未预期的条目。           |
| `policy/tools-required-deny-missing`                     | 全局或按智能体工具拒绝列表未包含要求拒绝的工具。     |
| `policy/sandbox-mode-unapproved`                         | 沙箱模式不在策略允许列表中。                                     |
| `policy/sandbox-backend-unapproved`                      | 沙箱后端不在策略允许列表中。                                  |
| `policy/sandbox-container-posture-unobservable`          | 为无法观测容器安全策略规则的后端启用了该规则。         |
| `policy/sandbox-container-host-network-denied`           | 基于容器的沙箱或浏览器使用主机网络模式。                     |
| `policy/sandbox-container-namespace-join-denied`         | 基于容器的沙箱或浏览器加入了另一个容器的命名空间。          |
| `policy/sandbox-container-mount-mode-required`           | 基于容器的沙箱或浏览器挂载不是只读的。                     |
| `policy/sandbox-container-runtime-socket-mount`          | 基于容器的沙箱或浏览器挂载暴露了容器运行时套接字。 |
| `policy/sandbox-container-unconfined-profile`            | 策略拒绝不受约束的容器沙箱配置文件，但当前配置不受约束。                    |
| `policy/sandbox-browser-cdp-source-range-missing`        | 策略要求配置沙箱浏览器 CDP 源范围，但当前缺少该范围。             |
| `policy/data-handling-redaction-disabled`                | 策略要求进行敏感日志脱敏，但该功能已禁用。                  |
| `policy/data-handling-telemetry-content-capture`         | 策略拒绝遥测内容捕获，但该功能已启用。                       |
| `policy/data-handling-session-retention-not-enforced`    | 策略要求执行会话保留维护，但当前未强制执行。            |
| `policy/data-handling-session-transcript-memory-enabled` | 策略拒绝会话记录记忆索引，但该功能已启用。              |
| `policy/secrets-unmanaged-provider`                      | 配置中的 SecretRef 引用了未在 `secrets.providers` 下声明的提供商。  |
| `policy/secrets-denied-provider-source`                  | 配置中的密钥提供商或 SecretRef 使用了被策略拒绝的来源。             |
| `policy/secrets-insecure-provider`                       | 策略拒绝不安全配置，但密钥提供商选择启用了该配置。               |
| `policy/auth-profile-invalid-metadata`                   | 配置中的身份验证配置文件缺少有效的提供商或模式元数据。                 |
| `policy/auth-profile-unapproved-mode`                    | 配置中的身份验证配置文件模式不在策略允许列表中。                       |
| `policy/exec-approvals-missing`                          | 策略要求 `exec-approvals.json`，但缺少该工件。               |
| `policy/exec-approvals-invalid`                          | 无法解析已配置的 Exec 审批工件。                          |
| `policy/exec-approvals-default-security-unapproved`      | Exec 审批默认值使用了不在策略允许列表中的安全模式。          |
| `policy/exec-approvals-agent-security-unapproved`        | 某个智能体的有效 Exec 审批安全模式不在允许列表中。       |
| `policy/exec-approvals-auto-allow-skills-enabled`        | 某个 Exec 审批智能体隐式自动允许 Skills CLI，但策略拒绝这种行为。   |
| `policy/exec-approvals-allowlist-missing`                | 审批允许列表缺少策略要求的模式。                  |
| `policy/exec-approvals-allowlist-unexpected`             | 审批允许列表包含策略未预期的模式。                |
| `policy/tools-missing-risk-level`                        | 受管控的工具声明缺少风险元数据。                             |
| `policy/tools-unknown-risk-level`                        | 受管控的工具声明使用了未知的风险值。                           |
| `policy/tools-missing-sensitivity-token`                 | 受管控的工具声明缺少敏感性元数据。                      |
| `policy/tools-missing-owner`                             | 受管控的工具声明缺少所有者元数据。                            |
| `policy/tools-unknown-sensitivity-token`                 | 受管控的工具声明使用了未知的敏感性值。                    |

一个发现可以同时包含 `target`（观测到的不符合要求的工作区对象）
和 `requirement`（使其成为发现的已编写规则）。
目前两者都是 `oc://` 地址字符串，但字段名称描述的是策略角色，
而不是地址格式。

发现示例：

```json
{
  "checkId": "policy/channels-denied-provider",
  "severity": "error",
  "message": "渠道 'telegram' 使用了被拒绝的提供商 'telegram'。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/channels/telegram",
  "target": "oc://openclaw.config/channels/telegram",
  "requirement": "oc://policy.jsonc/channels/denyRules/#0",
  "fixHint": "此工作区未批准使用 Telegram。"
}
```

```json
{
  "checkId": "policy/tools-missing-risk-level",
  "severity": "error",
  "message": "TOOLS.md 工具 'deploy' 没有明确的风险分类。",
  "source": "policy",
  "path": "TOOLS.md",
  "line": 12,
  "ocPath": "oc://TOOLS.md/tools/deploy",
  "target": "oc://TOOLS.md/tools/deploy",
  "requirement": "oc://policy.jsonc/tools/requireMetadata"
}
```

```json
{
  "checkId": "policy/mcp-unapproved-server",
  "severity": "error",
  "message": "MCP 服务器 'remote' 不在策略允许列表中。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/mcp/servers/remote",
  "target": "oc://openclaw.config/mcp/servers/remote",
  "requirement": "oc://policy.jsonc/mcp/servers/allow"
}
```

```json
{
  "checkId": "policy/models-unapproved-provider",
  "severity": "error",
  "message": "模型引用 'anthropic/claude-sonnet-4.7' 使用了未获批准的提供商 'anthropic'。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "target": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "requirement": "oc://policy.jsonc/models/providers/allow"
}
```

```json
{
  "checkId": "policy/network-private-access-enabled",
  "severity": "error",
  "message": "网络设置 'browser-private-network' 允许访问私有网络。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "target": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "requirement": "oc://policy.jsonc/network/privateNetwork/allow"
}
```

```json
{
  "checkId": "policy/gateway-non-loopback-bind",
  "severity": "error",
  "message": "Gateway 网关绑定设置 'gateway-bind' 允许非环回暴露。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/bind",
  "target": "oc://openclaw.config/gateway/bind",
  "requirement": "oc://policy.jsonc/gateway/exposure/allowNonLoopbackBind"
}
```

```json
{
  "checkId": "policy/gateway-node-command-denied",
  "severity": "error",
  "message": "Gateway 网关节点命令 'system.run' 已被策略拒绝，但未被 OpenClaw 配置拒绝。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/nodes/commands/deny",
  "target": "oc://openclaw.config/gateway/nodes/commands/deny",
  "requirement": "oc://policy.jsonc/gateway/nodes/denyCommands",
  "fixHint": "将 'system.run' 添加到 gateway.nodes.commands.deny，或在审查后更新策略。"
}
```

```json
{
  "checkId": "policy/agents-workspace-access-denied",
  "severity": "error",
  "message": "策略不允许 agents.defaults 沙箱 workspaceAccess 使用 'rw'。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "target": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "requirement": "oc://policy.jsonc/agents/workspace/allowedAccess"
}
```

## 修复

`doctor --lint` 和 `policy check` 为只读。

仅当明确启用
`workspaceRepairs` 时，`doctor --fix` 才会编辑策略管理的工作区设置；否则，检查会报告其
将修复的内容，并保持设置不变。

在此版本中，修复功能可以禁用被 `channels.denyRules` 拒绝的渠道，并
应用下列自动收紧修复。仅应在审查策略文件后启用 `workspaceRepairs`，
因为有效的规则可能更改工作区配置：

- 当全局策略禁止提升权限的工具时，设置 `tools.elevated.enabled=false`
- 当策略要求拒绝相应工具时，将缺失的必需拒绝工具 ID 添加到 `tools.deny` 或
  `agents.entries.*.tools.deny`
- 将不安全的 `gateway.controlUi.*` 开关设置为 `false`
- 当策略拒绝远程 Gateway 网关模式时，设置 `gateway.mode=local`
- 当策略拒绝 Gateway 网关 HTTP API 端点时，将报告的 `gateway.http.endpoints.*.enabled` 路径设置为
  `false`
- 当策略拒绝开放群组入口时，将报告的渠道入口 `groupPolicy` 路径设置为
  `allowlist`
- 当策略要求群组提及时，将报告的渠道入口 `requireMention` 路径设置为
  `true`
- 当策略要求对敏感日志进行脱敏时，设置
  `logging.redactSensitive=tools`
- 当策略拒绝遥测内容采集时，设置 `diagnostics.otel.captureContent=false`，或针对对象形式的遥测
  采集设置设置 `diagnostics.otel.captureContent.enabled=false`

限定范围的提升权限工具修复仅进行检测。当发现项报告共享日志或遥测配置时，
限定范围的数据处理修复也会被跳过，因为更改共享设置将影响限定范围策略
目标之外的对象。

当发现项报告继承的根 `tools.deny` 时，会跳过限定范围的必需拒绝修复，
因为将必需工具添加到根配置会影响限定范围策略目标之外的对象。
智能体本地的必需拒绝修复可以更新报告的 `agents.entries.*.tools.deny` 路径。

当发现项报告继承的 `channels.defaults.*` 时，会跳过限定范围的渠道入口修复，
因为更改共享渠道默认值会影响限定范围策略目标之外的对象。Gateway 网关 HTTP URL 获取允许列表发现项
仍需手动处理，因为自动修复无法选择正确的端点 URL
允许列表值。

Gateway 网关绑定和节点命令发现项仍需审查。当
`policy/gateway-non-loopback-bind` 或 `policy/gateway-node-command-denied`
可以映射到配置路径时，`doctor --fix` 会将建议的
`gateway.bind` 或 `gateway.nodes.commands.deny` 更改报告为已跳过的预览
指导。它不会应用更改，并且在操作员审查并更新配置或策略之前，
该发现项不会计为已修复。

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "config": {
          "workspaceRepairs": true,
        },
      },
    },
  },
}
```

## 退出代码

| 命令          | `0`                                                    | `1`                                                                 | `2`                          |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------------- | ---------------------------- |
| `policy check`   | 没有达到阈值的发现项。                          | 一个或多个发现项达到阈值。                             | 参数或运行时失败。 |
| `policy compare` | 策略文件至少与基准同样严格。 | 策略文件无效、缺失或弱于基准规则。 | 参数或运行时失败。 |
| `policy watch`   | 没有发现项，且已接受的哈希为最新。              | 存在发现项或已接受的证明已过期。                    | 参数或运行时失败。 |

## 相关内容

- [Doctor lint mode](/zh-CN/cli/doctor#lint-mode)
- [Path CLI](/zh-CN/cli/path)
