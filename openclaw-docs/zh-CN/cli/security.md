---
read_when:
    - 你想对配置/状态运行快速安全审计
    - 你希望应用安全的“修复”建议（权限、收紧默认设置）
summary: '`openclaw security` 的 CLI 参考（审计并修复常见的安全隐患）'
title: 安全性
x-i18n:
    generated_at: "2026-07-26T05:45:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b5f9ea5cb746bfd29ff4d096062e81595abe99a883fc3b1113b45a3527d42d9
    source_path: cli/security.md
    workflow: 16
---

# `openclaw security`

安全工具：审计以及可选的安全修复。相关内容：[安全](/zh-CN/gateway/security)。

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --deep --password <password>
openclaw security audit --deep --token <token>
openclaw security audit --auth password --password <password>
openclaw security audit --fix
openclaw security audit --json
```

## 审计模式

普通 `security audit` 始终使用冷配置、文件系统和只读路径：它不会发现插件运行时安全收集器，因此例行审计不会加载每个已安装插件的运行时。`--deep` 会添加尽力而为的实时 Gateway 网关探测和插件所有的安全审计收集器（显式内部调用方在已有适当运行时作用域时，也可以选择启用这些收集器）。

如果仅在启动时提供 Gateway 网关密码身份验证，请通过 `--auth password --password <password>` 传入相同的值，以便审计可以根据 `hooks.token` 对其进行检查。

## 检查内容

**私信/信任模型**

- 当多个私信发送者共享主会话时发出警告，并建议共享收件箱使用安全私信模式：`session.dmScope="per-channel-peer"`（多账号渠道则使用 `per-account-channel-peer`）。这是针对协作式共享收件箱的安全强化，并不能隔离彼此不信任的操作员；在这种情况下，应使用不同的 Gateway 网关（或不同的操作系统用户/主机）来划分信任边界。
- 当配置表明可能存在共享用户入口（例如开放的私信/群组策略、已配置的群组目标或通配符发送者规则）时，生成 `security.trust_model.multi_user_heuristic`——OpenClaw 的默认信任模型是个人助理（单个操作员），而不是抵御恶意行为的多租户隔离。对于有意设置的共享用户环境：对所有会话启用沙箱隔离，将文件系统访问限制在工作区范围内，并且不要在该运行时中存放个人/私有身份或凭据。
- 当使用小型模型（`<=300B` 参数）、未启用沙箱隔离且启用了 Web/浏览器工具时发出警告。

**Webhook/钩子**

启动日志会记录非致命安全警告，审计也会标记对有效 Gateway 网关共享密钥身份验证值（`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN`、`gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`）的 `hooks.token` 重用。此外，以下情况也会发出警告：

- `hooks.token` 过短
- `hooks.path="/"`
- 未设置 `hooks.defaultSessionKey`
- `hooks.allowedAgentIds` 不受限制
- 启用了请求 `sessionKey` 覆盖
- 启用了覆盖，但未设置 `hooks.allowedSessionKeyPrefixes`

运行 `openclaw doctor --fix` 轮换已持久化且被重复使用的 `hooks.token`，然后更新外部钩子发送方以使用新令牌。

**沙箱/工具**

- 当配置了沙箱 Docker 设置，但沙箱模式处于关闭状态时发出警告。
- 当 `gateway.nodes.commands.deny` 使用无效的类模式条目或未知条目时发出警告（仅对节点命令名称进行精确匹配，不会过滤 shell 文本）。
- 当 `gateway.nodes.commands.allow` 显式启用危险节点命令时发出警告。
- 当 Agent 工具配置文件覆盖全局 `tools.profile="minimal"` 时发出警告。
- 当写入/编辑工具已禁用，但 `exec` 仍可用且没有约束性的沙箱文件系统边界时发出警告。
- 当开放的私信或群组在没有沙箱/工作区防护的情况下暴露运行时/文件系统工具时发出警告。
- 当已安装插件的工具在宽松的工具策略下可能可访问时发出警告。

**沙箱浏览器**

- 当沙箱浏览器使用 Docker `bridge` 网络但未设置 `sandbox.browser.cdpSourceRange` 时发出警告。
- 标记危险的沙箱 Docker 网络模式，包括 `host` 和 `container:*` 命名空间加入模式。
- 当现有沙箱浏览器 Docker 容器缺少哈希标签或标签已过时时（例如迁移前的容器缺少 `openclaw.browserConfigEpoch`）发出警告，并建议使用 `openclaw sandbox recreate --browser --all`。

**网络/设备发现**

- 标记 `gateway.allowRealIpFallback=true`（代理配置错误时存在请求头伪造风险）。
- 标记 `discovery.mdns.mode="full"`（通过 mDNS TXT 记录泄露元数据）。
- 当 `gateway.auth.mode="none"` 使 Gateway 网关 HTTP API 在没有共享密钥的情况下可访问时发出警告（`/tools/invoke` 加上任何已启用的 `/v1/*` 端点）。

**插件/渠道**

- 当基于 npm 的插件/钩子安装记录未固定版本、缺少完整性元数据或与当前已安装的软件包版本不一致时发出警告。
- 当渠道允许列表依赖可变的名称/电子邮件/标签，而不是稳定 ID 时发出警告（适用于 Discord、Slack、Google Chat、Microsoft Teams、Mattermost 和 IRC 的相关作用域）。

以 `dangerous`/`dangerously` 为前缀的设置是操作员显式启用的紧急绕过项；启用其中一项本身并不构成安全漏洞报告。有关危险参数的完整清单，请参阅[安全](/zh-CN/gateway/security)中的“不安全或危险标志摘要”。

## SecretRef 行为

`security audit` 会以只读模式为其目标路径解析受支持的 SecretRef。如果当前命令路径中无法使用某个 SecretRef，审计将继续执行并报告 `secretDiagnostics`，而不是崩溃。`--token` 和 `--password` 仅覆盖该次命令调用的深度探测身份验证；它们不会重写配置或 SecretRef 映射。

## 抑制项

使用 `security.audit.suppressions` 接受有意保留的长期发现项。每个抑制项会精确匹配一个 `checkId`，并可通过不区分大小写的 `titleIncludes` 和/或 `detailIncludes` 子字符串缩小范围：

```json
{
  "security": {
    "audit": {
      "suppressions": [
        {
          "checkId": "plugins.tools_reachable_permissive_policy",
          "detailIncludes": "Enabled extension plugins: gbrain",
          "reason": "trusted local operator plugin"
        }
      ]
    }
  }
}
```

被抑制的发现项会从有效的 `summary` 和 `findings` 列表中移除。JSON 输出会将其保留在 `suppressedFindings` 下，以便审计。当配置了抑制项时，有效输出还会保留一条无法抑制的 `security.audit.suppressions.active` 信息发现项，以便读者判断审计结果是否经过筛选。危险配置标志会按每个标志一条发现项的方式输出，因此接受一个危险标志不会隐藏共享同一 `config.insecure_or_dangerous_flags` checkId 的其他已启用标志。

由于抑制项可以隐藏长期风险，因此通过 Agent 运行的 shell 命令添加或移除抑制项需要 Exec 审批，除非 Exec 已针对受信任的本地自动化使用 `security="full"` 和 `ask="off"` 运行。

## JSON 输出

```bash
openclaw security audit --json | jq '.summary'
openclaw security audit --deep --json | jq '.findings[] | select(.severity=="critical") | .checkId'
```

使用 `--fix --json` 时，输出会同时包含修复操作和最终报告：

```bash
openclaw security audit --fix --json | jq '{fix: .fix.ok, summary: .report.summary}'
```

## `--fix` 更改的内容

应用安全且确定性的修复措施：

- 将常见的 `groupPolicy="open"` 切换为 `groupPolicy="allowlist"`（包括受支持渠道中的账号变体）
- 当 WhatsApp 群组策略切换为 `allowlist` 时，如果已存储的 `allowFrom` 文件中存在列表，且配置尚未定义 `allowFrom`，则使用该列表初始化 `groupAllowFrom`
- 将 `logging.redactSensitive` 从 `"off"` 设置为 `"tools"`
- 收紧状态/配置和常见敏感文件（`credentials/*.json`、`auth-profiles.json`、`openclaw-agent.sqlite` 以及旧版会话工件）的权限
- 同时收紧 `openclaw.json` 引用的配置包含文件的权限
- 在 POSIX 主机上使用 `chmod`，在 Windows 上使用 `icacls` 重置

`--fix` **不会**：

- 轮换令牌/密码/API 密钥
- 禁用工具（`gateway`、`cron`、`exec` 等）
- 更改 Gateway 网关绑定/身份验证/网络暴露选项
- 移除或重写插件/Skills

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [安全审计](/zh-CN/gateway/security)
