---
read_when:
    - 调整提升权限模式的默认设置、允许列表或斜杠命令行为
    - 了解沙箱隔离的智能体如何访问主机
summary: 提升权限的 Exec 模式：从沙箱隔离的智能体在沙箱外运行命令
title: 提升权限模式
x-i18n:
    generated_at: "2026-07-26T07:02:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 40627217acf56122acfc48b689be1b9e2c61d889fe698e9c3c8fd91270d4a6cf
    source_path: tools/elevated.md
    workflow: 16
---

当智能体在沙箱内运行时，其 `exec` 命令会被限制在沙箱环境中。**提升权限模式**允许智能体突破这一限制，改为在沙箱外运行命令，并可配置审批关卡。

<Info>
  提升权限模式仅在智能体处于**沙箱隔离**状态时改变行为。对于未进行沙箱隔离的智能体，exec 已经在宿主机上运行。
</Info>

## 指令

使用斜杠命令按会话控制提升权限模式：

| 指令             | 作用                                                                                                         |
| ---------------- | ------------------------------------------------------------------------------------------------------------ |
| `/elevated on`   | 在配置的宿主机路径上于沙箱外运行，同时保留审批                                                              |
| `/elevated ask`  | 与 `on` 相同（别名）                                                                                                            |
| `/elevated full` | 在配置的宿主机路径上于沙箱外运行，并在模式/宿主机审批策略已经宽松时跳过审批                                  |
| `/elevated off`  | 恢复仅限沙箱内执行                                                                                           |

也可使用 `/elev on|off|ask|full`。

发送不带参数的 `/elevated` 可查看当前级别。

## 工作原理

<Steps>
  <Step title="检查可用性">
    必须在配置中启用提升权限，并且发送者必须在允许列表中：

    ```json5
    {
      tools: {
        elevated: {
          enabled: true,
          allowFrom: {
            discord: ["user-id-123"],
            whatsapp: ["+15555550123"],
          },
        },
      },
    }
    ```

  </Step>

  <Step title="设置级别">
    发送一条仅包含指令的消息，以设置会话默认值：

    ```
    /elevated full
    ```

    也可以内联使用（仅应用于该消息）：

    ```
    /elevated on 运行部署脚本
    ```

  </Step>

  <Step title="命令在沙箱外运行">
    启用提升权限后，`exec` 调用会离开沙箱。有效宿主机默认是
    `gateway`；当配置的/会话的 exec 目标为
    `node` 时，则为 `node`。在 `full` 模式下，如果解析后的 exec
    模式/宿主机审批策略已经完全宽松（security `full`、
    ask `off`），则会跳过 exec 审批；否则仍应用正常审批策略。在
    `on`/`ask` 模式下，始终应用配置的审批规则。
  </Step>
</Steps>

## 解析顺序

1. **内联指令**，位于消息中（仅应用于该消息）
2. **会话覆盖值**（通过发送仅包含指令的消息设置）
3. **全局默认值**（配置中的 `agents.defaults.elevatedDefault`）

## 可用性和允许列表

- **全局关卡**：`tools.elevated.enabled`（必须为 `true`）
- **发送者允许列表**：`tools.elevated.allowFrom`，包含各渠道的列表
- **每智能体关卡**：`agents.entries.*.tools.elevated.enabled`（只能进一步限制；全局关卡和每智能体关卡都必须为 `true`）
- **每智能体允许列表**：`agents.entries.*.tools.elevated.allowFrom`（发送者必须同时匹配全局和每智能体允许列表）
- **渠道提供的后备允许列表**：渠道插件可以选择通过 SDK 适配器钩子提供后备允许列表，在未配置 `tools.elevated.allowFrom.<provider>` 时使用。目前没有内置渠道实现此钩子，因此实际使用中，每个提供商目前都需要显式的 `tools.elevated.allowFrom.<provider>` 条目。
- **所有关卡都必须通过**；否则提升权限将被视为不可用

允许列表条目格式：

| 前缀                    | 匹配项                          |
| ----------------------- | ------------------------------- |
| （无）                  | 发送者 ID、E.164 或 From 字段   |
| `name:`                 | 发送者显示名称                  |
| `username:`             | 发送者用户名                    |
| `tag:`                  | 发送者标签                      |
| `id:`、`from:`、`e164:` | 显式指定身份                    |

## 提升权限不控制的内容

- **工具策略**：如果工具策略拒绝了 `exec`，提升权限无法将其覆盖。
- **宿主机选择策略**：提升权限不会将 `auto` 变成可随意跨宿主机覆盖的设置。它使用配置的/会话的 exec 目标规则，仅当目标已经是 `node` 时才选择 `node`。
- **与 `/exec` 分离**：`/exec` 指令可供已获授权的发送者调整每会话 exec 默认值（host、security、ask、node），且不需要提升权限模式。

<Note>
  bash 聊天命令（`!` 前缀；`/bash` 别名）使用独立关卡，除其自身的 `tools.bash.enabled` 标志外，还要求启用 `tools.elevated`。禁用提升权限也会禁止 `!` shell 命令。
</Note>

## 相关内容

<CardGroup cols={2}>
  <Card title="Exec 工具" href="/zh-CN/tools/exec" icon="terminal">
    从智能体执行 shell 命令。
  </Card>
  <Card title="Exec 审批" href="/zh-CN/tools/exec-approvals" icon="shield">
    `exec` 的审批和允许列表系统。
  </Card>
  <Card title="沙箱隔离" href="/zh-CN/gateway/sandboxing" icon="box">
    Gateway 网关级沙箱配置。
  </Card>
  <Card title="沙箱、工具策略和提升权限" href="/zh-CN/gateway/sandbox-vs-tool-policy-vs-elevated" icon="scale-balanced">
    三个关卡在工具调用期间如何组合。
  </Card>
</CardGroup>
