---
read_when:
    - 你需要使用插件钩子或工具，以便在执行有副作用的操作前进行询问
    - 你需要配置插件审批提示的发送位置
    - 你正在选择可选工具、Exec 审批和插件审批
sidebarTitle: Permission requests
summary: 请求用户批准插件工具调用和插件自有的权限提示
title: 插件权限请求
x-i18n:
    generated_at: "2026-07-26T06:16:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 675534212e70cc7b2e7bdc801955929c6a8156b08d620483edf0133afc3bfdaa
    source_path: plugins/plugin-permission-requests.md
    workflow: 16
---

插件权限请求允许插件代码暂停工具调用或插件自有操作，直到用户批准或拒绝。它们使用 Gateway 网关
`plugin.approval.*` 流程，以及处理聊天批准按钮和 `/approve` 命令的相同批准 UI 界面。

将插件权限请求用于插件/应用权限。它们不能取代主机 Exec 审批、可选工具允许列表或 Codex 的原生权限审查。

## 选择正确的门控

选择与你所需决策点匹配的门控：

| 门控                             | 适用场景                                                              | 控制内容                                                                                                  |
| -------------------------------- | ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| 可选工具                   | 在用户选择启用之前，不应让模型看到某个工具。        | 通过 `tools.allow` 控制工具暴露。                                                                              |
| 插件权限请求       | 插件钩子或插件自有操作必须在执行某个动作前询问用户。 | 通过 `plugin.approval.*` 进行运行时批准。                                                                     |
| Exec 审批                   | 主机命令或类 Shell 工具需要操作员批准。               | 主机 Exec 策略和持久 Exec 允许列表。                                                                     |
| Codex 原生权限请求 | Codex 在执行原生 Shell、文件、MCP 或应用服务器操作前询问用户。        | Codex 应用服务器或原生钩子批准处理；当 OpenClaw 拥有提示词时，通过插件批准进行路由。 |
| MCP 批准征询        | Codex MCP 服务器请求批准工具调用。                    | 通过 OpenClaw 插件批准桥接 MCP 批准响应。                                                 |

可选工具是发现时门控。插件权限请求是逐调用门控。如果敏感工具在模型可见前应要求明确选择启用，并且在动作执行前还需批准，请同时使用两者。

## 在工具调用前请求批准

大多数插件编写的提示词应从 `before_tool_call` 钩子开始。该钩子在模型选择工具之后、OpenClaw 执行工具之前运行：

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "deploy-policy",
  name: "Deploy Policy",
  register(api) {
    api.on("before_tool_call", async (event) => {
      if (event.toolName !== "deploy_service") {
        return;
      }

      const environment =
        typeof event.params.environment === "string" ? event.params.environment : "unknown";

      return {
        requireApproval: {
          title: "Deploy service",
          description: `Deploy service to ${environment}.`,
          severity: environment === "production" ? "critical" : "warning",
          allowedDecisions:
            environment === "production"
              ? ["allow-once", "deny"]
              : ["allow-once", "allow-always", "deny"],
          timeoutMs: 120_000,
          onResolution(decision) {
            console.log(`deploy approval resolved: ${decision}`);
          },
        },
      };
    });
  },
});
```

为批准此操作的人编写提示文本：

- 保持 `title` 简短并聚焦于操作；Gateway 网关将其限制为 80 个字符。
- 保持 `description` 具体且范围明确；Gateway 网关将其限制为 512 个字符。
- 包含操作、目标和风险。不要包含不应出现在聊天批准界面中的秘密、令牌或私有载荷。
- 省略时，`severity` 默认为 `"warning"`。仅对错误决策可能造成生产环境损坏或数据丢失的操作使用 `"critical"`。
- 省略时，`allowedDecisions` 默认为 `["allow-once", "allow-always", "deny"]`。当持久信任对于该操作并不安全时，传入 `["allow-once", "deny"]`。
- `timeoutMs` 默认为 120000（2 分钟），无论请求值是多少，上限均为 600000（10 分钟）。

## 决策行为

OpenClaw 使用 `plugin:` ID 创建待处理的批准，将其发送到可用的批准界面，然后等待决策。

| 决策          | 结果                                                                    |
| ----------------- | ------------------------------------------------------------------------- |
| `allow-once`      | 当前调用继续执行。                                               |
| `allow-always`    | 当前调用继续执行，并将决策传递给插件。      |
| `deny`            | 调用被阻止，并返回遭拒的工具结果。                            |
| 超时           | 调用被阻止。                                                      |
| 取消      | 运行中止时，调用被阻止。                              |
| 无批准路由 | 调用被阻止，因为没有已连接的批准界面能够处理该请求。 |

只有请求允许的确切 `allow-once` 和 `allow-always` 决策才能允许执行。未知、格式错误、不匹配、缺失和超时的决策均会以关闭方式失败。为兼容插件，旧版 `timeoutBehavior` 字段仍会被接受，但已弃用且会被忽略；不要在新钩子中设置它。

仅当发起请求的插件或运行时实现了相应持久化时，`allow-always` 才是持久的。对于普通 `before_tool_call.requireApproval` 钩子，OpenClaw 将 `allow-once` 和 `allow-always` 视为当前调用的批准决策，并将解析后的值传递给 `onResolution`。如果你的插件提供 `allow-always`，请准确记录并实现它信任哪些未来调用。

如果钩子还返回 `params`，OpenClaw 仅在批准成功后应用这些参数更改。即使优先级较高的钩子请求了批准，优先级较低的钩子仍可阻止调用。

`allowedDecisions` 限制向用户显示的按钮和命令。对于请求未提供的任何决策，Gateway 网关都会拒绝解析尝试。

## 路由批准提示词

批准提示词可以在本地 UI 界面中解析，也可以在支持批准处理的聊天渠道中解析。要将插件批准提示词转发到明确的聊天目标，请配置 `approvals.plugin`：

```json5
{
  approvals: {
    plugin: {
      enabled: true,
      mode: "targets",
      agentFilter: ["main"],
      targets: [{ channel: "slack", to: "U12345678" }],
    },
  },
}
```

`approvals.plugin` 独立于 `approvals.exec`。启用 Exec 审批转发不会路由插件批准提示词，启用插件批准转发也不会更改主机 Exec 策略。

当提示词包含手动批准文本时，请使用所提供的决策之一进行解析：

```text
/approve <id> allow-once
/approve <id> allow-always
/approve <id> deny
```

有关完整的转发模型、同一聊天批准行为、原生渠道投递和特定渠道批准者规则，请参阅[高级 Exec 审批](/zh-CN/tools/exec-approvals-advanced#plugin-approval-forwarding)。

## Codex 原生权限

Codex 原生权限提示词也可以通过插件批准传递，但其所有权与插件编写的钩子不同。

- Codex 应用服务器批准请求在 Codex 审查后通过 OpenClaw 路由。
- 启用原生钩子 `permission_request` 中继后，该中继可以通过 `plugin.approval.request` 发起询问。
- 当 Codex 将 `_meta.codex_approval_kind` 标记为 `"mcp_tool_call"` 时，MCP 工具批准征询会通过插件批准进行路由。

有关 Codex 特有的行为和回退规则，请参阅 [Codex harness runtime](/zh-CN/plugins/codex-harness-runtime#native-permissions-and-mcp-elicitations)。

## 故障排查

**工具提示插件批准不可用。** 没有批准 UI 或已配置的批准路由接受该请求。连接支持批准的客户端，使用支持同一聊天 `/approve` 的渠道，或配置 `approvals.plugin`。

**出现 `allow-always`，但下一次调用再次提示。** 通用插件批准流程不会自动为任意钩子持久化信任。在 `onResolution("allow-always")` 后，在你的插件中持久化插件自有信任；或者仅提供 `allow-once` 和 `deny`。

**`/approve` 拒绝该决策。** 请求限制了 `allowedDecisions`。请使用提示词中列出的决策之一。

**Discord、Matrix、Slack 或 Telegram 提示词的路由方式与 Exec 审批不同。** 插件批准和 Exec 审批使用不同的配置，并且可能使用不同的授权检查。请验证 `approvals.plugin` 和该渠道的插件批准支持，而不是只检查 `approvals.exec`。

## 相关内容

- [插件钩子](/zh-CN/plugins/hooks#tool-call-policy)
- [Building Plugins](/zh-CN/plugins/building-plugins#registering-tools)
- [高级 Exec 审批](/zh-CN/tools/exec-approvals-advanced#plugin-approval-forwarding)
- [Gateway 网关协议](/zh-CN/gateway/protocol)
- [Codex harness runtime](/zh-CN/plugins/codex-harness-runtime#native-permissions-and-mcp-elicitations)
