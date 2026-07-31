---
read_when:
    - 使用或修改 Exec 工具
    - 调试 stdin 或 TTY 行为
summary: Exec 工具用法、stdin 模式和 TTY 支持
title: Exec 工具
x-i18n:
    generated_at: "2026-07-26T06:35:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9c16b5122c527c069a4d1a0c1649726073339e95b9084100c1a0f45ebcae759d
    source_path: tools/exec.md
    workflow: 16
---

在工作区中运行 shell 命令。`exec` 是一个可修改内容的 shell 接口：只要所选主机或沙箱文件系统允许，命令就可以在任何位置创建、编辑或删除文件。禁用 OpenClaw 文件系统工具（如 `write`、`edit` 或 `apply_patch`）并不会使 `exec` 变为只读。

支持通过 `process` 在前台和后台执行。如果不允许使用 `process`，`exec` 将同步运行，并忽略 `yieldMs`/`background`。后台会话按智能体隔离；`process` 只能看到同一智能体的会话。

## 参数

<ParamField path="command" type="string" required>
要运行的 shell 命令。
</ParamField>

<ParamField path="workdir" type="string" default="cwd">
命令的工作目录。
</ParamField>

<ParamField path="env" type="object">
合并到继承环境之上的键值环境覆盖项。
</ParamField>

<ParamField path="yieldMs" type="number" default="10000">
在此延迟（毫秒）后自动将命令转入后台。
</ParamField>

<ParamField path="background" type="boolean" default="false">
立即将命令转入后台，而不是等待 `yieldMs`。
</ParamField>

<ParamField path="timeout" type="number" default="tools.exec.timeoutSeconds">
覆盖此次调用所配置的 Exec 超时时间，单位为秒。适用于前台、后台、`yieldMs`、Gateway 网关、沙箱和节点 `system.run` 执行。`timeout: 0` 会禁用此次调用的 Exec 进程超时。
</ParamField>

<ParamField path="pty" type="boolean" default="false">
可用时在伪终端中运行。适用于仅支持 TTY 的 CLI、编码智能体和终端 UI。
</ParamField>

<ParamField path="host" type="'auto' | 'sandbox' | 'gateway' | 'node'" default="auto">
执行位置。当沙箱运行时处于活动状态时，`auto` 解析为 `sandbox`，否则解析为 `gateway`。
</ParamField>

<ParamField path="security" type="'deny' | 'allowlist' | 'full'">
普通工具调用会忽略此项。`gateway`/`node` 安全策略由 `tools.exec.mode` 和主机审批文件决定；仅当操作员明确授予提升权限时，提升权限模式才能强制使用完全访问权限。
</ParamField>

<ParamField path="ask" type="'off' | 'on-miss' | 'always'">
基准询问模式由 `tools.exec.mode` 和主机审批配置决定。对于源自渠道的模型调用，当主机的有效询问模式为 `off` 时，会忽略每次调用的 `ask`；否则，它只能收紧为更严格的模式。
</ParamField>

<ParamField path="node" type="string">
使用 `host=node` 时的节点 ID/名称。
</ParamField>

<ParamField path="elevated" type="boolean" default="false">
请求提升权限模式：脱离沙箱并进入配置的主机路径。仅当提升权限解析为 `full` 时，才会强制使用 `security=full`。
</ParamField>

注意：

- `host` 仅接受 `auto`、`sandbox`、`gateway` 或 `node`。它不是主机名选择器；类似主机名的值会在命令运行前被拒绝。
- 允许从 `auto` 按调用指定 `host=node`；仅当没有活动的沙箱运行时时，才允许按调用指定 `host=gateway`。
- 即使没有额外配置，`host=auto` 仍然“开箱即用”：没有沙箱时，它解析为 `gateway`；存在活动沙箱时，它仍在沙箱中运行。
- `elevated` 会脱离沙箱并进入配置的主机路径：默认使用 `gateway`；当 `tools.exec.host=node`（或会话默认值为 `host=node`）时，使用 `node`。仅当当前会话/提供商启用了提升权限访问时，此功能才可用。
- `gateway`/`node` 审批由主机审批文件控制。
- `node` 需要已配对的节点（配套应用或无界面节点主机）。如果有多个可用节点，请设置 `exec.node` 或 `tools.exec.node` 以选择其中一个。
- `exec host=node` 是节点执行 shell 的唯一途径；旧版 `nodes.run` 包装器已被移除。
- 在非 Windows 主机上，如果设置了 `SHELL`，Exec 会使用它；如果 `SHELL` 为 `fish`，则会优先使用 `PATH` 中的 `bash`（或 `sh`），以避免与 fish 不兼容的 bash 语法；如果两者都不存在，则回退到 `SHELL`。
- 在 Windows 主机上，Exec 优先查找 PowerShell 7（`pwsh`）（依次搜索 Program Files、ProgramW6432 和 PATH），然后回退到 Windows PowerShell 5.1。
- 在非 Windows Gateway 网关主机上，bash 和 zsh Exec 命令使用启动快照。OpenClaw 从 shell 启动文件中捕获可加载的别名/函数和一小组安全环境变量，将其保存到 `$OPENCLAW_STATE_DIR/cache/shell-snapshots/`，然后在每次执行 Exec 命令前加载该快照。疑似密钥的变量会被排除；沙箱和节点 Exec 不使用此快照。在 Gateway 网关进程环境中设置 `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` 可禁用此快照路径。
- 主机执行（`gateway`/`node`）会拒绝 `env.PATH` 和加载器覆盖项（`LD_*`/`DYLD_*`），以防止二进制劫持或代码注入。
- OpenClaw 会在生成的命令环境（包括 PTY 和沙箱执行）中设置 `OPENCLAW_SHELL=exec`，以便 shell/profile 规则检测 Exec 工具上下文。
- 对于源自渠道的运行，如果渠道提供了相关 ID，OpenClaw 还会在 `OPENCLAW_CHANNEL_CONTEXT` 中公开一个范围受限的发送者/聊天身份 JSON 载荷。
- `exec` 无法运行 `openclaw channels login` 或 `/approve` shell 命令：`openclaw channels login` 是交互式渠道身份验证流程，而 `/approve` 必须通过审批命令处理程序执行，而不能通过 shell 执行。请在 Gateway 网关主机的终端中运行渠道登录，或在存在渠道专用登录智能体工具时使用该工具（例如 `whatsapp_login`）。
- 重要提示：沙箱隔离**默认关闭**。如果沙箱隔离关闭，隐式 `host=auto` 会解析为 `gateway`。显式 `host=sandbox` 仍会以安全方式失败，而不会静默改为在 Gateway 网关主机上运行。请启用沙箱隔离，或配合审批使用 `host=gateway`。
- 脚本预检（针对常见的 Python/Node shell 语法错误）仅检查有效 `workdir` 边界内的文件。如果脚本路径解析到 `workdir` 之外，则会跳过对该文件的预检。当 `host=gateway` 且有效策略为带有 `ask=off` 的 `security=full` 时，也会完全跳过预检。
- 对于现在启动的长时间运行任务，只需启动一次；当自动完成唤醒已启用且命令产生输出或失败时，依靠该机制唤醒。使用 `process` 查看日志、状态、提供输入或进行干预；不要使用休眠循环、超时循环或重复轮询来模拟调度。
- 智能体启动的后台命令在完成之前会显示在 Web、iOS 和 Android 的后台任务视图中。任务账本会在完成 Heartbeat 再次唤醒智能体之前完成最终记录。
- 对于应在稍后或按计划执行的工作，请使用 cron，而不是 `exec` 休眠/延迟模式。

## 配置

| 键                                   | 默认值                   | 说明                                                                                                                                                    |
| ------------------------------------ | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tools.exec.timeoutSeconds`          | `1800`                   | 每条命令的默认 Exec 超时时间，单位为秒。按调用指定的 `timeout` 会覆盖它；按调用指定的 `timeout: 0` 会禁用 Exec 进程超时。                  |
| `tools.exec.host`                    | `auto`                   | 当沙箱运行时处于活动状态时解析为 `sandbox`，否则解析为 `gateway`。                                                                            |
| `tools.exec.mode`                    | 由主机决定               | 规范策略选项。请参阅下方的[模式](#modes)。                                                                                                              |
| `tools.exec.reviewer.model`          | 配置的智能体主模型       | 用于 `mode=auto` 审查的可选提供商/模型覆盖项。                                                                                                |
| `tools.exec.reviewer.timeoutMs`      | `30000`                  | 在回退到人工处理之前，审查模型准备和完成阶段各自的超时时间。                                                                  |
| `tools.exec.node`                    | 未设置                   |                                                                                                                                                         |
| `tools.exec.notifyOnExit`            | `true`                   | 为 true 时，转入后台的 Exec 会话会在退出时将系统事件加入队列并请求一次 Heartbeat。                                                           |
| `tools.exec.approvalRunningNoticeMs` | `10000`                  | 当需要审批的 Exec 运行时间超过此值时，发出一次 "running" 通知（`0` 表示禁用）。                                                        |
| `tools.exec.strictInlineEval`        | `false`                  | 请参阅[内联求值](#inline-eval-strictinlineeval)。                                                                                                       |
| `tools.exec.commandHighlighting`     | `false`                  | 为 true 时，审批提示可在命令文本中突出显示由解析器识别的命令片段。可全局或按智能体设置；不会改变审批策略。 |
| `tools.exec.pathPrepend`             | 未设置                   | 要添加到 Exec 运行的 `PATH` 前面的目录列表（仅限 Gateway 网关 + 沙箱）。                                                                        |
| `tools.exec.safeBins`                | 未设置                   | 仅从 stdin 读取且无需显式允许列表条目即可运行的安全二进制文件。请参阅[安全二进制文件](/zh-CN/tools/exec-approvals-advanced#safe-bins-stdin-only)。         |
| `tools.exec.safeBinTrustedDirs`      | `/bin`, `/usr/bin`       | 在 `safeBins` 路径检查中明确受信任的其他目录。`PATH` 条目永远不会被自动信任。                                              |
| `tools.exec.safeBinProfiles`         | 未设置                   | 每个安全二进制文件的可选自定义 argv 策略（`minPositional`、`maxPositional`、`allowedValueFlags`、`deniedFlags`）。                                        |

Gateway 网关和节点默认采用无需审批的主机 Exec（`mode=full`）——这来自主机策略默认值，而不是 `host=auto`。如果需要审批/允许列表行为，请设置 `tools.exec.mode` 并收紧主机审批文件；请参阅 [Exec 审批](/zh-CN/tools/exec-approvals#yolo-mode-no-approval)。若要无论沙箱状态如何都强制路由到 Gateway 网关或节点，请设置 `tools.exec.host` 或使用 `/exec host=...`。

示例：

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
  },
}
```

### 模式

`tools.exec.mode` 是规范的持久化策略选项。运行时安全和审批行为由它派生。

| 模式        | security    | ask       | 行为                                                                                                                       |
| ----------- | ----------- | --------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `deny`      | `deny`      | `off`     | Exec 被拒绝。                                                                                                                |
| `allowlist` | `allowlist` | `off`     | 仅运行允许列表中或安全二进制文件中的命令；其他命令均不询问。                                                                 |
| `ask`       | `allowlist` | `on-miss` | 与允许列表匹配的命令直接运行；其他所有命令均询问人工审批。                                                                  |
| `auto`      | `allowlist` | `on-miss` | 与允许列表或安全二进制文件匹配的命令直接运行；其他所有命令先经过 OpenClaw 的原生自动审查器，再询问人工审批。 |
| `full`      | `full`      | `off`     | 无审批关卡。                                                                                                              |

无论持久化模式如何，每个会话的 `/exec ask=always` 仍会每次都请求人工审批。

自动审查审批仅供单次使用。在 Gateway 网关上，OpenClaw 会将解析后的可执行文件路径提供给审查器，并将执行固定到同一路径。对于无法归约为单个可强制执行计划的命令（例如 heredoc、shell 展开或不受支持的包装器引号），即使模型原本会允许，也会回退到人工审批。

对于尚未由显式运行时策略或原生策略决定的 Codex app-server 命令审批，将使用人工审批路径。OpenClaw 不会为这些请求运行其配置的 Exec 审查器，因为 Codex 不会公开一个可强制执行的已解析可执行文件，因而无法将审查决定绑定到 Codex 实际运行的命令。

### 内联求值（`strictInlineEval`）

当 `tools.exec.strictInlineEval` 为 `true` 时，内联解释器求值形式需要审查器或显式审批：`python -c`、`node -e`、`ruby -e`、`perl -e`、`php -r`、`lua -e`、`osascript -e`，以及其他受支持解释器和命令载体中的类似形式（`awk`、`find -exec`、`make`、`sed`、`xargs` 等）。在 `mode=auto` 中，常规 Exec 审批路径可以让原生自动审查器允许明显低风险的一次性命令；直接调用节点主机 `system.run` 仍需要显式审批，因为它们无法将命令交给人工审批路径。如果审查器要求审批，请求将转交人工处理。`allow-always` 仍可持久信任无害的解释器或脚本调用，但内联求值形式不会成为持久允许规则。

### PATH 处理

- `host=gateway`：将登录 shell 的 `PATH` 合并到 Exec 环境中。主机执行会拒绝 `env.PATH` 覆盖。守护进程本身仍使用最小化的 `PATH`：
  - macOS：`/opt/homebrew/bin`、`/usr/local/bin`、`/usr/bin`、`/bin`
  - Linux：`/usr/local/bin`、`/usr/bin`、`/bin`
  - 为防止用户 shell 配置（如 `~/.zshenv` 或 `/etc/zshenv`）在启动期间覆盖优先路径，`tools.exec.pathPrepend` 条目会在执行前，在 shell 命令内部安全地添加到最终 `PATH` 的开头。
- `host=sandbox`：在容器内运行 `sh -lc`（登录 shell），因此 `/etc/profile` 可能会重置 `PATH`。OpenClaw 会在加载配置文件后通过内部环境变量添加 `env.PATH`（不进行 shell 插值）；`tools.exec.pathPrepend` 在此处同样适用。
- `host=node`：仅将你传入且未被阻止的环境变量覆盖发送到节点。主机执行会拒绝 `env.PATH` 覆盖，节点主机也会忽略它们。如果需要在节点上添加 PATH 条目，请配置节点主机服务的环境（systemd/launchd），或将工具安装到标准位置。

按 Agent 绑定节点（在配置中使用带键的 Agent ID）：

```bash
openclaw config get agents.entries
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
```

Control UI：**设备**页面包含一个小型“Exec 节点绑定”面板，用于配置相同的设置。

## 会话覆盖（`/exec`）

使用 `/exec` 设置 `host`、`security`、`ask` 和 `node` 的**每会话**默认值。发送不带参数的 `/exec` 可显示当前值。

示例：

```text
/exec host=auto security=allowlist ask=on-miss node=mac-1
```

只有通过渠道允许列表/配对和访问组验证的**授权发送者**才能使用 `/exec`。访问组强制执行始终启用。它仅更新**会话状态**，不会写入配置。已授权的外部渠道发送者可以设置这些会话默认值。内部 Gateway 网关/Webchat 客户端需要 `operator.admin` 才能持久化这些设置。

要彻底禁用 Exec，请通过工具策略（`tools.deny: ["exec"]` 或按 Agent 配置）拒绝它。除非显式设置 `security=full` 和 `ask=off`，否则主机审批仍然适用。

## Exec 审批（配套应用/节点主机）

在 `exec` 于 Gateway 网关或节点主机上运行之前，沙箱隔离的 Agent 可以要求逐请求审批。有关策略、允许列表和 UI 流程，请参阅 [Exec 审批](/zh-CN/tools/exec-approvals)。

需要人工审批时，节点主机和非原生 Gateway 网关流程会立即返回 `status: "approval-pending"` 和一个审批 ID。原生聊天和 Web UI Gateway 网关流程则可以内联等待，并在审批后返回最终命令结果。`approval-pending` 结果表示命令尚未启动，因此仅当获批命令实际以内联方式运行时，才会显示前台回退警告。获批的异步运行会发出命令进度和完成系统事件（`Exec running` / `Exec finished`）；被拒绝或超时的审批是终态，不会通过拒绝系统事件唤醒 Agent 会话。

在具有原生审批卡片/按钮的渠道上，Agent 应优先使用该原生 UI；只有当工具结果明确表示聊天审批不可用，或手动审批是唯一途径时，才应包含手动 `/approve` 命令。

## 允许列表 + 安全二进制文件

手动允许列表强制执行会匹配解析后的二进制文件路径 glob 和裸命令名 glob。裸名称仅匹配通过 PATH 调用的命令，因此当命令为 `rg` 时，`rg` 可以匹配 `/opt/homebrew/bin/rg`，但不能匹配 `./rg` 或 `/tmp/rg`。

当 `security=allowlist` 时，仅当管道的每个分段都在允许列表中或属于安全二进制文件时，shell 命令才会被自动允许。在允许列表模式下，除非每个顶层分段（包括安全二进制文件）都满足允许列表，否则会拒绝命令串联（`;`、`&&`、`||`）和重定向。重定向仍不受支持。持久化的 `allow-always` 信任不会绕过此规则：串联命令仍要求每个顶层分段都匹配。

`autoAllowSkills` 是 Exec 审批中单独的便利路径，与手动路径允许列表条目不同。若需要严格的显式信任，请保持禁用 `autoAllowSkills`。

将这两类控制用于不同用途：

- `tools.exec.safeBins`：小型、仅使用 stdin 的流过滤器。
- `tools.exec.safeBinTrustedDirs`：为安全二进制可执行文件路径显式添加的额外可信目录。
- `tools.exec.safeBinProfiles`：自定义安全二进制文件的显式 argv 策略。
- 允许列表：对可执行文件路径的显式信任。

不要将 `safeBins` 当作通用允许列表，也不要添加解释器/运行时二进制文件（例如 `python3`、`node`、`ruby`、`bash`）。如果需要这些程序，请使用显式允许列表条目，并保持启用审批提示。

当解释器/运行时 `safeBins` 条目缺少显式配置文件时，`openclaw security audit` 会发出警告；`openclaw doctor --fix` 可以为缺失的自定义 `safeBinProfiles` 条目搭建基础配置。当你显式将 `jq` 等行为宽泛的二进制文件重新添加到 `safeBins` 时，`openclaw security audit` 和 `openclaw doctor` 也会发出警告（`jq` 可以读取环境数据，并从模块或启动文件加载 jq 代码，因此应改用显式允许列表条目或需要审批的运行）。即使显式列出，`jq` 也会被拒绝作为安全二进制文件。如果显式将解释器加入允许列表，请启用 `tools.exec.strictInlineEval`，以确保内联代码求值形式仍需审查器或显式审批。

完整策略详情和示例请参阅 [Exec 审批](/zh-CN/tools/exec-approvals-advanced#safe-bins-stdin-only)和[安全二进制文件与允许列表对比](/zh-CN/tools/exec-approvals-advanced#safe-bins-versus-allowlist)。

## 示例

前台：

```json
{ "tool": "exec", "command": "ls -la" }
```

后台 + 轮询：

```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

轮询用于按需获取状态，而不是等待循环。如果启用了自动完成唤醒，命令在产生输出或失败时可以唤醒会话。

发送按键（tmux 风格）：

```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

提交（仅发送 CR）：

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

粘贴（默认使用括号粘贴模式）：

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## apply_patch

`apply_patch` 是 `exec` 的子工具，用于结构化的多文件编辑。它默认启用，适用于任何模型提供商；`allowModels` 可以对其进行限制。仅当你想禁用它或将其限制为特定模型时才使用配置：

```json5
{
  tools: {
    exec: {
      applyPatch: { workspaceOnly: true, allowModels: ["gpt-5.6-sol"] },
    },
  },
}
```

注意：

- 工具策略仍然适用；`allow: ["write"]` 会隐式允许 `apply_patch`。
- `deny: ["write"]` 不会拒绝 `apply_patch`；请显式拒绝 `apply_patch`，或在还应阻止补丁写入时使用 `deny: ["group:fs"]`。
- 配置位于 `tools.exec.applyPatch` 下。
- `tools.exec.applyPatch.enabled` 默认为 `true`；将其设为 `false` 可禁用此工具。
- `tools.exec.applyPatch.workspaceOnly` 默认为 `true`（限制在工作区内）。仅当你有意允许 `apply_patch` 在工作区目录之外写入/删除时，才将其设为 `false`。
- `tools.exec.applyPatch.allowModels` 是可选的模型 ID 允许列表（可以是 `gpt-5.4` 这样的原始 ID，也可以是 `openai/gpt-5.4` 这样的完整 ID）。设置后，仅匹配的模型可使用该工具；未设置时，所有模型均可使用。

## 相关内容

- [Exec 审批](/zh-CN/tools/exec-approvals) — shell 命令的审批关卡
- [沙箱隔离](/zh-CN/gateway/sandboxing) — 在沙箱隔离环境中运行命令
- [后台进程](/zh-CN/gateway/background-process) — 长时间运行的 Exec 和进程工具
- [安全性](/zh-CN/gateway/security) — 工具策略和提升权限的访问
