---
read_when:
    - 你想更改默认模型或查看提供商身份验证状态
    - 你想扫描可用的模型/提供商并调试身份验证配置文件
summary: '`openclaw models` 的 CLI 参考（状态/列出/设置/扫描、别名、回退、身份验证）'
title: Models
x-i18n:
    generated_at: "2026-07-26T06:44:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f7405c25694f04afe9c3029a8af64ae3ae7e1bdcf4c4ac31b8b84ff512d6a90e
    source_path: cli/models.md
    workflow: 16
---

# `openclaw models`

模型发现、扫描和配置（默认模型、回退模型、身份验证配置文件）。

相关内容：

- 提供商和模型：[模型](/zh-CN/providers/models)
- 模型选择概念和 `/models` 斜杠命令：[模型概念](/zh-CN/concepts/models)
- 提供商身份验证设置：[入门指南](/zh-CN/start/getting-started)

## 常用命令

```bash
openclaw models status
openclaw models list
openclaw models set <model-or-alias>
openclaw models set-image <model-or-alias>
openclaw models scan
```

`status` 和 `auth` 子命令接受 `--agent <id>`，用于指定已配置的智能体；`list`、`scan`、`aliases` 以及 `fallbacks`/`image-fallbacks` 始终使用已配置的默认智能体，而 `set`/`set-image` 会直接拒绝 `--agent`。省略时，支持 `--agent` 的命令会在已设置的情况下使用 `OPENCLAW_AGENT_DIR`，否则使用已配置的默认智能体。

### 状态

`openclaw models status` 显示解析后的默认模型和回退模型，以及身份验证概览。对于 Codex 等由插件拥有的 Agent Runtimes，它还会检查所属插件是否已启用并通过启动载荷验证。凭据有效但运行时不可用的路由会报告 `status: unavailable`，而不是 `usable`；JSON 输出包含相互独立的 `authStatus`、`runtimeStatus` 和有界的运行时诊断信息。当提供商用量快照可用时，OAuth/API 密钥状态部分会包含提供商用量窗口和配额快照。目前支持用量窗口的提供商有：Anthropic、GitHub Copilot、Gemini CLI、OpenAI、MiniMax、Xiaomi 和 z.ai。如果提供商提供了专用钩子，则通过该钩子获取用量身份验证信息；否则，OpenClaw 会回退到从身份验证配置文件、环境变量或配置中匹配 OAuth/API 密钥凭据。

在 `--json` 输出中，`auth.providers` 是感知环境变量、配置和存储的提供商概览，而 `auth.oauth` 仅表示身份验证存储中的配置文件健康状况。

选项：

| 标志                      | 作用                                                                                                                                   |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `--json`                  | JSON 输出；身份验证配置文件、提供商和启动诊断信息会发送到 stderr，以便 stdout 仍可通过管道传给 `jq`。                            |
| `--plain`                 | 纯文本输出。                                                                                                                       |
| `--check`                 | 如果身份验证即将过期或已过期，或者选定的智能体运行时不可用，则以非零状态退出：`1` = 不可用/已过期/缺失，`2` = 即将过期。 |
| `--probe`                 | 对已配置的身份验证配置文件进行实时探测。会发出真实请求；可能消耗 token 并触发速率限制。                                       |
| `--probe-provider <name>` | 仅探测一个提供商。                                                                                                                 |
| `--probe-profile <id>`    | 探测指定的身份验证配置文件 ID（可重复指定或用逗号分隔）。                                                                             |
| `--probe-timeout <ms>`    | 每次探测的超时时间。                                                                                                                       |
| `--probe-concurrency <n>` | 并发探测数。                                                                                                                       |
| `--probe-max-tokens <n>`  | 探测的最大 token 数（尽力而为）。                                                                                                          |
| `--agent <id>`            | 已配置的智能体 ID；覆盖 `OPENCLAW_AGENT_DIR`。                                                                                     |

探测行可能来自身份验证配置文件、环境变量凭据或 `models.json`。探测状态分类：`ok`、`auth`、`rate_limit`、`billing`、`timeout`、`format`、`unknown`、`no_model`。

当探测始终未能发起模型调用时，可能出现以下探测详情/原因代码：

- `excluded_by_auth_order`：存在已存储的配置文件，但显式指定的 `auth.order.<provider>` 将其省略，因此探测会报告该排除情况，而不是尝试使用它。
- `missing_credential`、`invalid_expires`、`expired`、`unresolved_ref`：配置文件存在，但不符合使用条件或无法解析。
- `ineligible_profile`：配置文件因其他原因与提供商配置不兼容。
- `no_model`：存在提供商身份验证信息，但 OpenClaw 无法为该提供商解析出可探测的候选模型。

对于 OpenAI ChatGPT/Codex OAuth 故障排除，`openclaw models status`、`openclaw models auth list --provider openai` 和 `openclaw config get agents.defaults.model --json` 是确认智能体是否拥有可通过原生 Codex 运行时用于 `openai/*` 的 `openai` OAuth 配置文件的最快方式。请参阅 [OpenAI provider 设置](/zh-CN/providers/openai#check-and-recover-codex-oauth-routing)。

### 列出

`openclaw models list` 是只读的：它会读取配置、身份验证配置文件、现有目录状态和提供商拥有的目录行，但绝不会重写 `models.json`。

选项：`--all`（完整目录）、`--local`（仅筛选本地模型）、`--provider <id>`、`--json`、`--plain`。

注意：

- `Auth` 列是只读的。对于 OpenAI 等由提供商拥有的模型路由，它会将每一行的 API/基础 URL 路由与有效 `auth.order` 中符合条件的配置文件、环境变量/配置凭据以及已解析的命令作用域 SecretRef 进行匹配。当具体 OpenAI 行的路由策略不可用时，其状态会保持未知，而不会借用提供商级身份验证；仅提供商级的旧式检查及其他提供商仍保留提供商级行为。插件的合成身份验证元数据只是一项运行时能力提示，并不能证明原生账户身份验证，因此依赖账户的路由在没有明确注册表证据时仍为未知。该命令不会加载提供商运行时、读取钥匙串机密、调用提供商 API，也不会证明确切的执行就绪状态。
- `models list --all --provider <id>` 可以包含来自插件清单或内置提供商目录元数据、由提供商拥有的静态目录行，即使你尚未向该提供商进行身份验证。这些行在配置匹配的身份验证信息之前仍会显示为不可用。
- `models list` 可在提供商目录发现速度较慢时保持控制平面的响应能力。默认视图和配置视图会在短暂等待后回退到已配置或合成的模型行，并允许发现过程在后台完成。当你需要确切、完整的已发现目录，并愿意等待提供商发现完成时，请使用 `--all`。
- 宽泛的 `models list --all` 会在不加载提供商运行时补充钩子的情况下，将清单目录行合并到注册表行之上。按提供商筛选的清单快速路径仅用于标记为 `static` 的提供商；标记为 `refreshable` 的提供商仍以注册表/缓存为基础，并将清单行作为补充追加；标记为 `runtime` 的提供商则继续使用注册表/运行时发现。
- `models list` 会将原生模型元数据与运行时上限区分开来。在表格输出中，当有效运行时上限与原生上下文窗口不同时，`Ctx` 会显示 `contextTokens/contextWindow`；如果提供商公开了该上限，JSON 行会包含 `contextTokens`。
- 对于提供商拥有的路由，`models list` 会将一个逻辑提供商/模型行投影到选定路由上。`Input` 和 `Ctx` 仅来自完全匹配的物理路由目录行，显式配置的逻辑覆盖项最后应用；当路由选择无法解析时，能力字段会显示为未知，而不会借用同级路由的元数据。
- `models list --provider <id>` 按提供商 ID 筛选，例如 `moonshot` 或 `openai`。它不接受交互式提供商选择器中的显示标签，例如 `Moonshot AI`。
- 解析模型引用时，会在**第一个** `/` 处拆分。如果模型 ID 包含 `/`（OpenRouter 风格），请包含提供商前缀（示例：`openrouter/moonshotai/kimi-k2`）。
- 如果省略提供商，OpenClaw 会先将输入解析为别名，然后尝试与该确切模型 ID 在已配置提供商中进行唯一匹配，最后才会回退到已配置的默认提供商，并显示弃用警告。如果该提供商不再提供已配置的默认模型，OpenClaw 会回退到第一个已配置的提供商/模型，而不是显示已移除提供商的过时默认值。
- `models status` 在身份验证输出中，对于非机密占位符（例如 `OPENAI_API_KEY`、`secretref-managed`、`minimax-oauth`、`oauth:chutes`、`ollama-local`）可能会显示 `marker(<value>)`，而不是将其遮蔽为机密。

### 设置默认模型/图像模型

```bash
openclaw models set <model-or-alias>
openclaw models set-image <model-or-alias>
```

`set` 写入 `agents.defaults.model.primary`；`set-image` 写入 `agents.defaults.imageModel.primary`。两者都接受 `provider/model` 或已配置的别名。当新选择的模型需要 Codex/Copilot 运行时插件时，`set` 还会修复其插件安装；`set-image` 则不会。两个命令都不接受 `--agent`；它们始终写入智能体默认值。

### 扫描

`models scan` 会读取 OpenRouter 的公共 `:free` 目录，并对候选项进行排序，以供回退使用。该目录本身是公开的，因此仅扫描元数据不需要 OpenRouter 密钥。

默认情况下，OpenClaw 会尝试通过实时模型调用探测工具和图像支持。如果未配置 OpenRouter 密钥，该命令会回退到仅输出元数据，并说明 `:free` 模型仍需要 `OPENROUTER_API_KEY` 才能进行探测和推理。

选项：

- `--no-probe`（仅元数据；不查找配置/机密）
- `--min-params <b>`
- `--max-age-days <days>`
- `--provider <name>`
- `--max-candidates <n>`
- `--timeout <ms>`（目录请求和每次探测的超时时间）
- `--concurrency <n>`
- `--yes`
- `--no-input`
- `--set-default`
- `--set-image`
- `--json`

`--set-default` 和 `--set-image` 需要实时探测；仅元数据的扫描结果只供参考，不会应用到配置中。

## 别名

```bash
openclaw models aliases list [--json] [--plain]
openclaw models aliases add <alias> <model-or-alias>
openclaw models aliases remove <alias>
```

别名按模型条目存储为 `agents.defaults.models.<key>.alias`。`add` 会先将 `<model-or-alias>` 解析为规范的提供商/模型键，因此为一个别名设置别名会将其重新指向目标，而不是形成别名链。
添加别名不会更改 `agents.defaults.modelPolicy.allow`，也不会限制模型覆盖。

## 回退模型

```bash
openclaw models fallbacks list [--json] [--plain]
openclaw models fallbacks add <model-or-alias>
openclaw models fallbacks remove <model-or-alias>
openclaw models fallbacks clear
```

管理 `agents.defaults.model.fallbacks`。`openclaw models image-fallbacks list|add|remove|clear` 使用相同的子命令形式管理并行的 `agents.defaults.imageModel.fallbacks` 列表。

## 身份验证配置文件

```bash
openclaw models auth add
openclaw models auth list [--provider <id>] [--json]
openclaw models auth login --provider <id>
openclaw models auth login --provider openai --profile-id openai:work
openclaw models auth login-github-copilot
openclaw models auth paste-api-key --provider <id>
openclaw models auth setup-token --provider <id>
openclaw models auth paste-token --provider <id>
openclaw models auth order get --provider <id>
openclaw models auth order set --provider <id> <profileIds...>
openclaw models auth order clear --provider <id>
```

`models auth add` 是交互式身份验证助手。它可以启动提供商身份验证流程（OAuth/API key），也可以引导你手动粘贴令牌，具体取决于所选的提供商。

`models auth list` 会列出所选智能体已保存的身份验证配置文件，但不会输出令牌、API key 或 OAuth 机密材料。使用 `--provider <id>` 可筛选单个提供商，例如 `openai`；使用 `--json` 可用于脚本。

`models auth login` 会运行提供商插件的身份验证流程（OAuth/API key）。使用 `openclaw plugins list` 可查看已安装的提供商。对于登录期间支持命名配置文件的提供商，`login` 接受 `--profile-id <id>`（用它将同一提供商的多个登录相互分开）；使用 `--method <id>` 可选择特定身份验证方法；`--device-code` 是 `--method device-code` 的快捷方式；使用 `--set-default` 可应用提供商推荐的默认模型；使用 `--force` 可先删除该提供商的现有配置文件（当缓存的 OAuth 配置文件卡住或你想切换账户时使用）。

`models auth login-github-copilot` 是 `models auth login --provider github-copilot --method device`（GitHub 设备流程）的快捷方式；它接受 `--yes`，可在不提示的情况下覆盖现有配置文件。

使用 `openclaw models auth --agent <id> <subcommand>` 可将身份验证结果写入特定的已配置智能体存储。父级 `--agent` 标志受 `add`、`list`、`login`、`paste-api-key`、`setup-token`、`paste-token`、`login-github-copilot` 以及 `order get`/`set`/`clear` 支持。

对于 OpenAI 模型，`--provider openai` 默认使用 ChatGPT/Codex 账户登录。仅当你想添加 OpenAI API key 配置文件时才使用 `--method api-key`，通常将其用作 Codex 订阅限额的备用方案。运行 `openclaw doctor --fix` 可将旧版 OpenAI Codex 前缀的身份验证/配置文件状态迁移到 `openai`。

示例：

```bash
openclaw models auth login --provider openai --set-default
openclaw models auth login --provider openai --method api-key
openclaw models auth paste-api-key --provider openai
openclaw models auth list --provider openai
```

注意：

- `paste-api-key` 接受在其他位置生成的 API key，提示输入密钥值，并将其写入默认配置文件 ID `<provider>:manual`，除非传入 `--profile-id`。在自动化中，通过标准输入传入密钥，例如 `printf "%s\n" "$OPENAI_API_KEY" | openclaw models auth paste-api-key --provider openai`。
- `setup-token` 和 `paste-token` 仍是通用令牌命令，适用于公开令牌身份验证方法的提供商。
- `setup-token` 需要交互式 TTY，并运行提供商的令牌身份验证方法（如果该提供商公开了 `setup-token` 方法，则默认使用该方法）。
- `paste-token` 需要 `--provider`，默认提示输入令牌值，并将其写入默认配置文件 ID `<provider>:manual`，除非传入 `--profile-id`。在自动化中，应通过标准输入传入令牌，而不是将其作为参数传递，以免提供商凭据出现在 shell 历史记录或进程列表中。
- `paste-token --expires-in <duration>` 根据 `365d` 或 `12h` 等相对时长存储绝对令牌过期时间。
- 对于 `openai`，OpenAI API key 与 ChatGPT/OAuth 令牌材料采用不同的身份验证结构。对 `sk-...` OpenAI API key 使用 `paste-api-key`；仅对令牌身份验证材料使用 `paste-token`。
- Anthropic：`setup-token`/`paste-token` 是 OpenClaw 为 `anthropic` 支持的身份验证路径，但当主机上有 Claude CLI（`claude -p`）可用时，OpenClaw 更倾向于复用它。
- `auth order get/set/clear` 管理某个提供商按智能体设置的身份验证配置文件顺序覆盖，该设置存储在 `auth-state.json` 中（与 `auth.order.<provider>` 配置键分开）。`set` 按优先级顺序接受一个或多个配置文件 ID；`clear` 会回退到配置/轮询顺序。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [模型选择](/zh-CN/concepts/model-providers)
- [模型故障转移](/zh-CN/concepts/model-failover)
