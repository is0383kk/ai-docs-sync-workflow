---
read_when:
    - 你需要一个适用于 Gateway 网关的终端 UI（便于远程使用）
    - 你想从脚本传入 url/token/session
    - 你希望在不使用 Gateway 网关的情况下，以本地嵌入模式运行 TUI
    - 你想使用 openclaw chat 或 openclaw tui --local
summary: '`openclaw tui` 的 CLI 参考（由 Gateway 网关支持或本地嵌入式终端 UI）'
title: TUI
x-i18n:
    generated_at: "2026-07-26T06:06:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5406f25bbd22c64867296c15112fafcaf8e1580c759e5fdc81fccfb62ae1e318
    source_path: cli/tui.md
    workflow: 16
---

# `openclaw tui`

打开连接到 Gateway 网关的终端 UI，或以本地嵌入模式运行。

相关指南：[TUI](/zh-CN/web/tui)

## 选项

| 标志                         | 默认值                                   | 描述                                                                        |
| ---------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------- |
| `--local`                    | `false`                                   | 使用本地嵌入式智能体运行时，而不是 Gateway 网关。                 |
| `--url <url>`                | 配置中的 `gateway.remote.url`          | Gateway 网关 WebSocket URL。                                                             |
| `--token <token>`            | （无）                                    | Gateway 网关令牌（如果需要）。                                                         |
| `--password <pass>`          | （无）                                    | Gateway 网关密码（如果需要）。                                                      |
| `--tls-fingerprint <sha256>` | `gateway.remote.tlsFingerprint`           | 固定 `wss://` Gateway 网关的预期 TLS 证书指纹。                |
| `--session <key>`            | `main`（范围为全局时使用 `global`） | 会话键。在 Agent 工作区内，除非添加前缀，否则会自动选择该智能体。 |
| `--deliver`                  | `false`                                   | 通过已配置的渠道发送助手回复。                             |
| `--thinking <level>`         | （模型默认值）                           | 覆盖思考级别。                                                           |
| `--message <text>`           | （无）                                    | 连接后发送初始消息。                                          |
| `--timeout-ms <ms>`          | `agents.defaults.timeoutSeconds`          | 智能体超时时间。无效值会记录警告并被忽略。                       |
| `--history-limit <n>`        | `200`                                     | 连接时加载的历史记录条目数。                                                 |

别名：`openclaw chat` 和 `openclaw terminal` 会调用此命令，并隐含使用
`--local`。

## 注意事项

- `--local` 不能与 `--url`、`--token`、`--password` 或 `--tls-fingerprint` 组合使用。
- 在可能的情况下，`tui` 会解析为令牌/密码身份验证配置的 Gateway 网关身份验证 SecretRefs
  （`env`/`file`/`exec` 提供商）。
- 未显式指定 URL 或端口时，`tui` 会采用正在运行的 Gateway 网关所记录的活动本地 Gateway 网关端口。
  显式的 `--url`、`OPENCLAW_GATEWAY_URL`、
  `OPENCLAW_GATEWAY_PORT` 和远程 Gateway 网关配置仍具有优先权。
- 从已配置的 Agent 工作区目录内启动时，TUI 会自动选择
  该智能体作为会话键的默认值（除非将 `--session` 显式设为
  `agent:<id>:...`）。
- 本地模式直接使用嵌入式智能体运行时。大多数本地工具都可使用，
  但仅限 Gateway 网关的功能不可用。
- 本地模式会将 `/auth [provider]` 添加到 TUI 命令界面。
- 插件审批门控在本地模式下仍然适用：需要审批的工具
  会在终端中提示作出决定，不会静默自动批准任何操作。
- 会话[目标](/zh-CN/tools/goal)显示在页脚中，并可使用
  `/goal` 进行管理。

## 示例

```bash
openclaw chat
openclaw tui --local
openclaw tui
openclaw tui --url ws://127.0.0.1:18789 --token <token>
openclaw tui --session main --deliver
openclaw chat --message "将我的配置与文档进行比较，并告诉我需要修复什么"
# 在智能体工作区内运行时，会自动推断该智能体
openclaw tui --session bugfix
```

## 配置修复循环

使用本地模式，让嵌入式智能体检查当前配置，将其与文档进行比较，
并协助在同一个终端中修复配置。

如果 `openclaw config validate` 已经失败，请先运行 `openclaw configure` 或
`openclaw doctor --fix`；`openclaw chat` 不会绕过
无效配置保护。

```bash
openclaw chat
```

然后在 TUI 内：

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

使用 `openclaw config set` 或 `openclaw configure` 应用针对性修复，然后
重新运行 `openclaw config validate`。请参阅 [TUI](/zh-CN/web/tui) 和
[配置](/zh-CN/cli/config)。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [TUI](/zh-CN/web/tui)
- [目标](/zh-CN/tools/goal)
