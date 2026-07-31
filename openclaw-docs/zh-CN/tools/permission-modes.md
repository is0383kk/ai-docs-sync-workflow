---
read_when:
    - 为命令权限选择 auto、ask、allowlist、full 或 deny
    - 通过 tools.exec.mode 配置经 Codex Guardian 审查的审批流程
    - OpenClaw Exec 审批与 ACPX harness 权限对比
summary: 主机 Exec 权限模式、Codex Guardian 审批和 ACPX harness 会话
title: 权限模式
x-i18n:
    generated_at: "2026-07-26T07:03:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f580e66508c1f69e868ed26a62d88a675f86a4d1ca738650dc5af82e967f3ac3
    source_path: tools/permission-modes.md
    workflow: 16
---

权限模式决定智能体在运行主机命令、写入文件或请求后端 harness 提供额外访问权限之前拥有多大的权限。

<Note>
  权限模式与 `tools.exec.host=auto` 相互独立。`tools.exec.host`
  决定命令在哪里运行。`tools.exec.mode` 决定如何批准主机 Exec。
</Note>

## 推荐默认设置

对于需要实用主机访问权限、但又不希望每次未命中都提示人工处理的编码智能体，请使用 `auto`：

```bash
openclaw config set tools.exec.mode auto
openclaw approvals get
openclaw gateway restart
```

然后验证生效的策略：

```bash
openclaw exec-policy show
```

## OpenClaw 主机 Exec 模式

`tools.exec.mode` 是主机 `exec` 的标准化策略界面。每种模式都会解析为底层的 `security`（允许列表严格程度）和 `ask`（未命中时提示）组合：

| 模式        | security / ask          | 行为                                                                                      | 适用场景                                              |
| ----------- | ----------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `deny`      | `deny` / `off`          | 完全阻止主机 Exec。                                                                     | 不允许执行任何主机命令。                         |
| `allowlist` | `allowlist` / `off`     | 仅运行允许列表中的命令；静默拒绝未命中的命令。                                          | 你有一组已知安全的命令。                    |
| `ask`       | `allowlist` / `on-miss` | 运行与允许列表匹配的命令；未命中时请求人工批准。                                                 | 每种新命令都应由人工审核。              |
| `auto`      | `allowlist` / `on-miss` | 运行与允许列表匹配的命令；未命中的命令先经过自动审核，再回退到人工批准。 | 编码会话需要实用且受保护的访问权限。        |
| `full`      | `full` / `off`          | 无需提示即可运行主机 Exec。                                                                | 此受信任的主机/会话应跳过审批关卡。 |

`ask` 和 `auto` 使用相同的允许列表/询问设置；`auto` 还会启用原生自动审核器，由它自行裁决未命中的命令，只有在无法安全批准时，才交由已配置的人工审批路径处理。

有关完整的主机 Exec 策略、本地审批文件、允许列表架构、安全二进制文件和转发行为，请参阅 [Exec 审批](/zh-CN/tools/exec-approvals)。

## Codex Guardian 映射

对于原生 Codex app-server 会话，如果本地 Codex 要求允许，`tools.exec.mode: "auto"` 会引导 Codex 使用由 Guardian 审核的审批。通常产生以下值：

| Codex 字段         | 典型值     |
| ------------------- | ----------------- |
| `approvalPolicy`    | `on-request`      |
| `approvalsReviewer` | `auto_review`     |
| `sandbox`           | `workspace-write` |

`auto` 模式会强制使用此策略，覆盖所有已配置的 Codex 沙箱/审批设置，因此不会保留 `approvalPolicy: "never"` 与 `sandbox: "danger-full-access"` 等旧版不安全组合。`tools.exec.mode: "deny"` 和 `"allowlist"` 会完全阻止 Codex app-server 在本地执行。仅当你有意采用无需审批的安全态势时，才使用 `tools.exec.mode: "full"`。

有关 app-server 设置、身份验证顺序和原生 Codex 运行时的详细信息，请参阅 [Codex harness](/zh-CN/plugins/codex-harness)。

## ACPX harness 权限

ACPX 会话是非交互式的，因此无法点击 TTY 权限提示。ACPX 使用 `plugins.entries.acpx.config` 下单独的 harness 级设置：

| 设置                     | 值          | 含义                                     |
| --------------------------- | --------------- | ------------------------------------------- |
| `permissionMode`            | `approve-reads` | 仅自动批准读取操作。                    |
| `permissionMode`            | `approve-all`   | 自动批准写入操作和 shell 命令。     |
| `permissionMode`            | `deny-all`      | 拒绝所有权限提示。                |
| `nonInteractivePermissions` | `fail`          | 需要提示时中止。      |
| `nonInteractivePermissions` | `deny`          | 拒绝提示，并在可能的情况下继续。 |

ACPX 权限应与 OpenClaw Exec 审批分开设置：

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
openclaw gateway restart
```

将 `approve-all` 用作 ACPX 的紧急解锁设置，等同于无需提示的 harness 会话。有关设置详情和失败模式，请参阅 [ACP Agents 设置](/zh-CN/tools/acp-agents-setup#permission-configuration)。

## 选择模式

| 目标                                          | 配置                                                   |
| --------------------------------------------- | ----------------------------------------------------------- |
| 完全阻止主机命令                | `tools.exec.mode: "deny"`                                   |
| 仅允许运行已知安全的命令              | `tools.exec.mode: "allowlist"`                              |
| 每种新命令形式都请求人工批准       | `tools.exec.mode: "ask"`                                    |
| 在人工审核前使用 Codex/OpenClaw 自动审核  | `tools.exec.mode: "auto"`                                   |
| 完全跳过主机 Exec 审批             | `tools.exec.mode: "full"` 加上匹配的主机审批文件 |
| 允许非交互式 ACPX 会话进行写入/执行 | `plugins.entries.acpx.config.permissionMode: "approve-all"` |

如果更改模式后命令仍然提示或失败，请检查这两个层面：

```bash
openclaw approvals get
openclaw exec-policy show
```

主机 Exec 会采用 OpenClaw 配置与主机本地审批文件中更严格的结果。ACPX harness 权限不会放宽主机 Exec 审批，主机 Exec 审批也不会放宽 ACPX harness 提示。

## 相关内容

- [Exec 审批](/zh-CN/tools/exec-approvals)
- [Exec 审批 - 高级](/zh-CN/tools/exec-approvals-advanced)
- [Codex harness](/zh-CN/plugins/codex-harness)
- [ACP Agents 设置](/zh-CN/tools/acp-agents-setup#permission-configuration)
