---
read_when:
    - 你想通过 CLI 编辑 Exec 审批
    - 你需要在 Gateway 网关或节点主机上管理允许列表
    - 你需要在没有聊天界面的情况下列出或处理待审批请求
summary: '`openclaw approvals` 和 `openclaw exec-policy` 的 CLI 参考'
title: 审批
x-i18n:
    generated_at: "2026-07-26T06:04:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8b6f198af718d7b058498dbb960a1eb68ced601e1cd9205070b7199688552d2
    source_path: cli/approvals.md
    workflow: 16
---

# `openclaw approvals`

管理**本地主机**、**Gateway 网关主机**或**节点主机**的 Exec 审批。未指定目标标志时，命令会读取或写入磁盘上的本地审批文件。使用 `--gateway` 指定 Gateway 网关，或使用 `--node <id|name|ip>` 指定特定节点。

别名：`openclaw exec-approvals`

相关内容：[Exec 审批](/zh-CN/tools/exec-approvals)、[节点](/zh-CN/nodes)

## `openclaw exec-policy`

`openclaw exec-policy` 是一个**仅限本地**的便捷命令，可一步同步所请求的 `tools.exec.*` 配置与本地主机审批文件：

```bash
openclaw exec-policy show
openclaw exec-policy show --json

openclaw exec-policy preset yolo
openclaw exec-policy preset cautious --json

openclaw exec-policy set --host gateway --security full --ask off --ask-fallback full
```

预设（`yolo`、`cautious`、`deny-all`）会同时应用 `host`、`security`、`ask` 和 `askFallback`。`set` 仅应用传入的标志；系统会验证每个接受的值（`--host auto|sandbox|gateway|node`、`--security deny|allowlist|full`、`--ask off|on-miss|always`、`--ask-fallback deny|allowlist|full`）。

作用范围：

- 同时更新本地配置文件和本地审批文件；不会将策略推送到 Gateway 网关或节点主机。
- 系统会拒绝 `--host node`：节点 Exec 审批在运行时从节点获取，因此本地 `exec-policy` 无法同步这些审批。请改用 `openclaw approvals set --node <id|name|ip>`。
- `exec-policy show` 会将 `host=node` 作用域标记为在运行时由节点管理，而不是根据本地审批文件推导有效策略。

对于远程主机审批，请直接使用 `openclaw approvals set --gateway` 或 `openclaw approvals set --node <id|name|ip>`。

## 常用命令

```bash
openclaw approvals get
openclaw approvals get --node <id|name|ip>
openclaw approvals get --gateway
openclaw approvals pending
openclaw approvals resolve <id> <allow-once|allow-always|deny>
```

`get` 显示目标的有效 Exec 策略：所请求的 `tools.exec` 策略、主机审批文件策略以及合并后的有效结果。具有主机原生策略的节点（例如 Windows 配套应用）会直接显示该策略，而不会应用 OpenClaw 审批文件的策略计算方式。

对于基于文件的节点，合并视图需要主机解析的策略快照。较旧的节点会将有效策略显示为不可用，而不会假定 Gateway 网关请求的策略也适用于该主机。

<Note>
不包括每会话的 `/exec` 覆盖项。请在相关会话中运行 `/exec`，以检查其当前默认值。
</Note>

优先级：

- 主机审批文件是可强制执行的事实来源。
- 所请求的 `tools.exec` 策略可以缩小或扩大意图范围，但有效结果由主机规则推导。
- `--node` 会将节点主机审批文件与 Gateway 网关的 `tools.exec` 策略结合使用（两者都会在运行时生效）。
- 如果 Gateway 网关配置不可用，CLI 会回退到节点审批快照，并注明无法计算最终运行时策略。

## 待处理的审批

列出 Gateway 网关中待处理的 Exec、插件和 OpenClaw 系统智能体审批：

```bash
openclaw approvals pending
openclaw approvals pending --json
```

完整枚举及相应的操作员范围 `resolve` 流程使用 `operator.admin`，因为否则审批记录会保留请求者/审查者筛选条件。解决审批还会请求专用的 `operator.approvals` 作用域。标准 CLI 操作员授权包含这两个作用域；受限的第三方客户端不应仅为模拟此命令而请求管理员权限。

人类可读输出会显示审批类型、智能体/会话归属、请求已存在时长、距离到期的时间、缩短后的命令或摘要，以及与 shell 无关的 `id64_<base64url>` ID 令牌。紧凑表格后始终会有一个 `Full request text` 块，其中包含所有完整令牌和以无损方式转义的请求，因此终端宽度导致的缩短不会隐藏后缀或解决审批所需的令牌。请将完整令牌复制到 `resolve`。其他字段中的不安全终端字符会显示为可见的 Unicode 转义序列。JSON 输出在 `approvals` 下返回规范化条目，并为脚本保留原始的 `id`、`summary`、`createdAtMs` 和 `expiresAtMs`；`resolve` 仍接受原始 ID，除非它们使用保留的 `id64_` 显示令牌前缀。

如果提供的 `id64_` 值既匹配某个字面原始 ID，又匹配另一项审批解码后的显示令牌，CLI 会因其存在歧义而拒绝该值，以免解决错误的请求。

使用完整 ID 解决一项审批：

```bash
openclaw approvals resolve <id> allow-once
openclaw approvals resolve <id> allow-always
openclaw approvals resolve <id> deny --reason "Not expected during maintenance"
```

CLI 会读取统一审批记录以确定其类型，检查请求的决定是否在该记录允许的决定范围内，然后调用统一解决器。首次成功作出决定时以 `0` 退出。重复已记录的决定也会以 `0` 退出，并报告 `already resolved (same decision)`。如果决定冲突、审批不存在、审批已过期，或该审批类型不支持所请求的决定，则会显示明确错误并以非零状态退出。

`--reason` 会在 CLI 确认信息中添加本地备注。当前 Gateway 网关审批记录没有自由文本形式的解决原因字段，因此不会持久保存此备注，也不会将其发送到其他审批界面。

## 从文件替换审批配置

```bash
openclaw approvals set --file ./exec-approvals.json
openclaw approvals set --stdin <<'EOF'
{ version: 1, defaults: { security: "full", ask: "off", askFallback: "full" } }
EOF
openclaw approvals set --node <id|name|ip> --file ./exec-approvals.json
openclaw approvals set --gateway --file ./exec-approvals.json
```

`set` 接受 JSON5，而不仅限于严格 JSON。请使用 `--file` 或 `--stdin` 中的一个，不要同时使用。

采用主机原生策略的 Windows 节点使用其自己的策略结构：

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  defaultAction: "deny",
  rules: [{ pattern: "hostname", action: "allow" }]
}
EOF
```

CLI 会先读取节点的当前哈希值，并随更新一起发送，因此并发的本地编辑会被拒绝，而不会遭到覆盖。此操作会替换节点的完整规则列表，因此必须提供 `rules`；`defaultAction` 为可选项。报告其原生策略已禁用的节点无法进行远程配置；请先在该主机上启用或配置策略。主机原生策略不支持 `allowlist add|remove` 辅助命令。

## “永不提示”/YOLO 示例

对于不应因 Exec 审批而停止的主机，将其主机审批默认值设置为 `full` + `off`：

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

对于公开 OpenClaw 审批文件的节点，请将同一内容与 `openclaw approvals set --node <id|name|ip> --stdin` 配合使用。主机原生节点需要使用上文所示的所有者专用结构。

这只会更改**主机审批文件**。要同时使所请求的 OpenClaw 策略保持一致，还应设置：

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.mode full
```

这里明确指定 `tools.exec.host=gateway`，因为 `host=auto` 的含义仍然是“沙箱可用时使用沙箱，否则使用 Gateway 网关”：YOLO 针对的是审批，而不是路由。当你希望即使配置了沙箱也在主机上执行时，请使用 `gateway`（或 `/exec host=gateway`）。

省略 `askFallback` 时默认为 `deny`。升级应继续保持永不提示行为的无 UI 主机时，请显式设置 `askFallback: "full"`。

仅在本地机器上实现相同意图的快捷方式：

```bash
openclaw exec-policy preset yolo
```

## 允许列表辅助命令

```bash
openclaw approvals allowlist add "~/Projects/**/bin/rg"
openclaw approvals allowlist add --agent main --node <id|name|ip> "/usr/bin/uptime"
openclaw approvals allowlist add --agent "*" "/usr/bin/uname"

openclaw approvals allowlist remove "~/Projects/**/bin/rg"
```

## 常用选项

`get`、`set` 和 `allowlist add|remove` 均支持：

- `--node <id|name|ip>`（解析 ID、名称、IP 或 ID 前缀；与 `openclaw nodes` 使用相同的解析器）
- `--gateway`
- 共享节点 RPC 选项：`--url`、`--token`、`--timeout`、`--json`

未指定目标标志表示使用磁盘上的本地审批文件。

`allowlist add|remove` 还支持 `--agent <id>`（默认为 `"*"`，适用于所有智能体）。

`pending` 和 `resolve` 始终使用 Gateway 网关，因为待处理请求是 Gateway 网关的实时状态。它们支持共享的 Gateway 网关连接选项 `--url`、`--token` 和 `--timeout`；`pending` 还支持 `--json`。

## 备注

- 节点主机必须公告 `system.execApprovals.get/set`（macOS 应用、无头节点主机或 Windows 配套应用）。
- 审批文件按主机存储在 OpenClaw 状态目录中：`$OPENCLAW_STATE_DIR/exec-approvals.json`；未设置该变量时则存储在 `~/.openclaw/exec-approvals.json`。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [Exec 审批](/zh-CN/tools/exec-approvals)
