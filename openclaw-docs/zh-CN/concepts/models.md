---
read_when:
    - 更改模型回退行为或选择体验
    - 调试“模型不被允许”或默认提供商回退过时问题
    - 处理 models.json 的合并/密钥行为
sidebarTitle: Models CLI
summary: OpenClaw 如何解析提供商/模型引用、配置键和 `/model` 聊天命令
title: 模型 CLI
x-i18n:
    generated_at: "2026-07-26T06:13:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2cd13a2aae6575bdfeefb477b7fe8be740b77c66cb76454b07d82481f6612152
    source_path: concepts/models.md
    workflow: 16
---

<CardGroup cols={2}>
  <Card title="模型故障转移" href="/zh-CN/concepts/model-failover">
    身份验证配置文件轮换、冷却机制及其与回退的交互方式。
  </Card>
  <Card title="模型提供商" href="/zh-CN/concepts/model-providers">
    提供商快速概览和示例。
  </Card>
  <Card title="模型 CLI 参考" href="/zh-CN/cli/models">
    完整的 `openclaw models` 命令和标志参考。
  </Card>
  <Card title="配置参考" href="/zh-CN/gateway/config-agents#agent-defaults">
    模型配置键、默认值和示例。
  </Card>
</CardGroup>

模型引用（`provider/model`）用于选择提供商和模型，而不是底层
Agent 运行时。当未设置运行时策略或策略为 `auto` 时，仅当路由是完全匹配的官方 HTTPS Platform
Responses 或 ChatGPT Responses 路由，且没有用户编写的请求覆盖时，OpenAI provider 自有的
路由策略才可能选择 Codex；仅有
`openai/*` 前缀绝不会选择 Codex。Completions 适配器、自定义
端点以及用户编写的请求行为仍使用 OpenClaw。官方明文
HTTP 端点会被拒绝。请参阅 [OpenAI 隐式 Agent 运行时](/zh-CN/providers/openai#implicit-agent-runtime)。

订阅版 Copilot 引用（`github-copilot/*`）可以选择使用外部
GitHub Copilot agent runtime 插件，但该路径始终需要显式选择（绝不会
由 `auto` 选择）。运行时覆盖应配置在提供商/模型策略上，而不是配置在
整个智能体或会话上。运行时选择并不决定计费方式：
OpenAI API 密钥凭据与 ChatGPT/Codex 订阅凭据仍相互独立。请参阅
[Agent Runtimes](/zh-CN/concepts/agent-runtimes) 和
[GitHub Copilot agent runtime](/zh-CN/plugins/copilot)。

## 选择顺序

<Steps>
  <Step title="主模型">
    `agents.defaults.model.primary`（或以普通字符串形式使用 `agents.defaults.model`）。
  </Step>
  <Step title="回退模型">
    `agents.defaults.model.fallbacks`，按顺序尝试。
  </Step>
  <Step title="身份验证故障转移">
    在 OpenClaw 转移到下一个回退模型之前，会先在提供商内部轮换身份验证配置文件。
  </Step>
</Steps>

相关模型配置项：

- `agents.defaults.models` 存储别名和每个模型的设置。添加条目不会限制模型覆盖。
- `agents.defaults.modelPolicy.allow` 是可选的覆盖允许列表。使用精确引用或尾部前缀通配符，例如 `provider/*` 和 `provider/namespace/*`；省略该项或将 `[]` 设置为允许任意模型。每个智能体的 `agents.entries.*.modelPolicy.allow` 会替换该智能体的默认策略。
- `agents.defaults.utilityModel` 是用于简短内部任务的可选低成本模型，例如生成的仪表板会话标题、受支持渠道的线程/主题标题以及进度叙述。每个智能体的 `agents.entries.*.utilityModel` 会覆盖它。未设置时，如果主提供商声明了小模型默认值，OpenClaw 会使用该值（OpenAI → `gpt-5.6-luna`，Anthropic → `claude-haiku-4-5`）；否则使用智能体的主模型。将其设置为空字符串可禁用实用模型路由。使用不同的实用模型生成标题失败时，会使用主模型重试一次。对于仪表板标题，自动实用模型派生和常规回退会遵循有效会话提供商及身份验证配置文件；显式配置的实用模型则保留其已配置的提供商/身份验证。空实用模型只会跳过备用小模型路由，不会跳过仪表板标题生成。实用任务是独立的模型调用，可能会将有限的任务内容发送给所选模型提供商。
- `agents.defaults.imageModel` 仅在主模型无法接受图像时使用。
- `agents.defaults.pdfModel` 由 `pdf` 工具使用。如果未设置，该工具会回退到 `imageModel`，然后回退到解析后的会话/默认模型。
- `agents.defaults.mediaModels.{image,music,video}` 为共享媒体生成工具提供支持。如果未设置，每个工具会推断具有身份验证支持的提供商默认值：先使用当前默认提供商，然后按照提供商 ID 顺序尝试其余为该能力注册的提供商。跨提供商回退是固定的默认行为。
- 每个智能体的 `agents.entries.*.model`（以及绑定）会覆盖 `agents.defaults.model` —— 请参阅[多智能体路由](/zh-CN/concepts/multi-agent)。

完整键参考、默认值和 JSON5 示例：[配置参考](/zh-CN/gateway/config-agents#agent-defaults)。

## 选择来源和回退严格性

同一个 `provider/model` 会因其来源不同而表现不同：

| 来源                                                                  | 行为                                                                                                                                                                                                                                                       |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 已配置的默认值（`agents.defaults.model.primary`、每个智能体的主模型） | 正常起点；使用 `agents.defaults.model.fallbacks`。                                                                                                                                                                                                 |
| 自动回退                                                           | 临时恢复状态，存储为 `modelOverrideSource: "auto"`。OpenClaw 会定期重新探测原始主模型，在恢复时清除自动选择，并在每次状态变化时通知一次回退/恢复转换。                              |
| 用户会话选择                                                  | 精确且严格。`/model`、模型选择器、`session_status(model=...)` 和 `sessions.patch` 会存储 `modelOverrideSource: "user"`。如果该提供商/模型变得不可访问，运行会明确失败，而不会继续尝试另一个已配置模型。 |
| 定时任务 `--model` / 负载 `model`                                        | 每个任务的主模型。除非任务提供自己的负载 `fallbacks`，否则仍使用已配置的回退模型（`fallbacks: []` 会强制严格运行）。                                                                                                                    |

其他选择规则：

- 更改 `agents.defaults.model.primary` 不会重写现有会话固定设置。如果状态报告 `This session is pinned to X; config primary Y will apply to new/unpinned sessions.`，请运行 `/model default` 清除固定设置。
- CLI 默认模型和允许列表选择器会遵循 `models.mode: "replace"`，仅列出 `models.providers.*.models`，而不是完整的内置目录。
- Control UI 模型选择器会向 Gateway 网关请求其已配置的模型视图。显式的 `modelPolicy.allow` 会对其进行筛选，包括尾部前缀通配符条目；否则，它会显示已配置的模型以及具有可用身份验证的提供商。完整内置目录仅用于显式浏览视图（带有 `view: "all"` 的 `models.list`，或 `openclaw models list --all`）。
- 提供商清单 UI 使用带有 `view: "provider-config"` 的 `models.list`，显示由来源编写的 `models.providers.*.models` 行，而不应用选择器允许列表。

完整机制：[模型故障转移](/zh-CN/concepts/model-failover)。

## 快速模型策略

- 将主模型设置为你可用的最强最新一代模型。
- 对于对成本/延迟敏感的任务和风险较低的聊天，使用回退模型。
- 对于启用了工具的智能体或不受信任的输入，避免使用较旧/较弱的模型层级。

## 新手引导

```bash
openclaw onboard
```

无需手动编辑配置，即可为常用提供商设置模型和身份验证，包括 OpenAI Codex 订阅 OAuth 和 Anthropic（API 密钥或复用 Claude CLI）。

如果未配置主模型，全新 OpenAI API 密钥设置会选择
`openai/gpt-5.6`；不带限定的直接 API ID 会解析到 Sol 层级。全新
ChatGPT/Codex OAuth 设置会选择精确的 `openai/gpt-5.6-sol` 目录引用。
重新进行身份验证时会保留现有的显式主模型，包括
`openai/gpt-5.5`。如果账户无法使用 GPT-5.6，请显式选择
`openai/gpt-5.5`；OpenClaw 不会在不提示的情况下将其降级。

## “不允许使用该模型”（以及回复为何停止）

如果 `agents.defaults.modelPolicy.allow` 非空，它会成为 `/model`、会话覆盖和 `--model` 的允许列表。选择允许列表之外的模型时，系统会在生成任何正常回复之前返回。每个智能体的 `agents.entries.*.modelPolicy.allow` 会替换该智能体的默认策略。

```text
agents.defaults.modelPolicy.allow 不允许模型覆盖“provider/model”。
请将“provider/model”“provider/*”或更窄的“provider/namespace/*”前缀添加到 agents.defaults.modelPolicy.allow，或者删除/清空该列表以允许任意模型。
```

修复方法是将模型或提供商通配符添加到指定的 `modelPolicy.allow` 键、删除/清空该列表，或从 `/model list` 中选择模型。如果被拒绝的命令包含运行时覆盖（例如 `/model openai/gpt-5.5 --runtime codex`），请先修复允许列表，然后重试同一命令。

对于本地/GGUF 模型，允许列表需要完整的提供商前缀引用，例如 `ollama/gemma4:26b` 或 `lmstudio/Gemma4-26b-a4-it-gguf` —— 请查看 `openclaw models list --provider <provider>` 以获取精确字符串。允许列表启用后，仅使用文件名或显示名称是不够的。

要限制提供商而不逐一列出每个模型，请使用尾部前缀通配符条目。提供商范围的 `provider/*` 会匹配该提供商下的所有模型；更窄的前缀（例如 `clawrouter/anthropic/*`）仅匹配该命名空间：

```json5
{
  agents: {
    defaults: {
      modelPolicy: {
        allow: ["openai/*", "vllm/*"],
      },
    },
  },
}
```

随后，`/model`、`/models` 和模型选择器只会显示这些提供商的已发现目录，并且新模型无需编辑允许列表即可出现。将精确的 `provider/model` 条目与 `provider/*` 条目混合使用，可引入另一个提供商的某个特定模型。

包含别名和每个模型设置的允许列表示例：

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-sonnet-4-6" },
      modelPolicy: {
        allow: ["anthropic/claude-sonnet-4-6", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
}
```

<Accordion title="显式编辑允许列表">
直接设置完整列表：

```bash
openclaw config set agents.defaults.modelPolicy.allow '["openai/gpt-5.4","anthropic/*"]' --strict-json
```

`openclaw models set`、提供商设置和 `openclaw models aliases add` 可以在 `agents.defaults.models` 下添加条目，但绝不会更改 `modelPolicy.allow`。这样可使模型元数据和别名独立于覆盖策略。
</Accordion>

## 聊天中的 `/model`

```text
/model
/model list
/model 3
/model openai/gpt-5.4
/model default
/model status
```

- `/model` 和 `/model list` 会显示一个紧凑的编号选择器（模型系列 + 可用提供商）；`/model <#>` 从中进行选择。在 Discord 上，这会打开提供商/模型下拉菜单，并包含一个 Submit 步骤；在 Telegram 上，选择器中的选择仅作用于会话，绝不会改写 `openclaw.json` 中智能体的持久默认值。`/models add` 已弃用，它会返回一条消息，而不是从聊天中注册模型。
- `/model` 会立即持久化新的会话选择。如果智能体处于空闲状态，下一次运行会立即使用它；如果已有运行正在进行，切换会排队等到下一个可安全重试的节点（如果工具活动或回复输出已经开始，则等待之后的节点）。
- `/model default` 会清除会话选择，使其重新继承已配置的主模型。
- 用户选择的 `/model` 引用对该会话是严格的：如果它变得不可访问，回复会明确失败，而不是通过 `agents.defaults.model.fallbacks` 静默回退。已配置的默认值和定时任务的主模型仍会使用回退链。
- `/model status` 是详细视图：显示每个提供商的身份验证候选项，以及（配置后）提供商端点 `baseUrl` 和 `api` 模式。
- 模型引用通过在第一个 `/` 处拆分来解析；请输入 `provider/model`。如果模型 ID 本身包含 `/`（OpenRouter 风格），请包含提供商前缀，例如 `/model openrouter/moonshotai/kimi-k2`。如果省略提供商，OpenClaw 会依次尝试：(1) 匹配别名；(2) 对该未加前缀的确切模型 ID 进行唯一的已配置提供商匹配；(3) 使用已配置的默认提供商（已弃用的回退）——如果该提供商已不再提供已配置的默认模型，则改用第一个已配置的提供商/模型，以免暴露已被移除提供商的过时默认值。
- 模型引用会规范化为小写；提供商 ID 在其他方面则要求完全匹配，因此请使用插件公布的 ID。

完整命令行为和配置：[斜杠命令](/zh-CN/tools/slash-commands)。

## CLI

```bash
openclaw models status
openclaw models list
openclaw models set <provider/model>
openclaw models set-image <provider/model>
openclaw models scan
openclaw models aliases list|add|remove
openclaw models fallbacks list|add|remove|clear
openclaw models image-fallbacks list|add|remove|clear
openclaw models auth list|add|login|paste-api-key|paste-token|setup-token|order
```

不带子命令的 `openclaw models` 是 `models status` 的快捷方式，后者还会显示身份验证存储配置文件的 OAuth 到期状态（默认在 24h 内到期时发出警告）。完整的标志、JSON 结构和身份验证配置文件子命令：[模型 CLI 参考](/zh-CN/cli/models)。

<AccordionGroup>
  <Accordion title="扫描（OpenRouter 免费模型）">
    `openclaw models scan` 会检查 OpenRouter 的公共免费模型目录，并可实时探测候选模型是否支持工具和图像。目录本身是公开的，因此仅扫描元数据（`--no-probe`）不需要密钥；实时探测以及 `--set-default`/`--set-image` 需要 OpenRouter API 密钥（身份验证配置文件或 `OPENROUTER_API_KEY`），如果没有密钥，则会以闭锁方式仅输出元数据。

    结果排序依据依次为：图像支持、工具延迟、上下文大小、参数数量。在 TTY 中，探测结果会提示以交互方式选择回退项；非交互模式需要使用 `--yes` 接受默认值。

  </Accordion>
</AccordionGroup>

## 模型注册表（`models.json`）

在 `models.providers` 下配置的自定义提供商会写入智能体目录下的 `models.json`（默认值为 `~/.openclaw/agents/<agentId>/agent/models.json`）。提供商插件目录会单独存储为由插件拥有的已生成目录分片，并自动加载。默认情况下，此文件会与配置合并；将 `models.mode: "replace"` 设置为仅使用你配置的提供商。

<AccordionGroup>
  <Accordion title="合并模式优先级">
    对于匹配的提供商 ID：

    - 智能体 `models.json` 中已存在的非空 `baseUrl` 优先。
    - `models.json` 中的非空 `apiKey` 仅在当前配置/身份验证配置文件上下文中该提供商不受 SecretRef 管理时优先。
    - 受 SecretRef 管理的 `apiKey` 值会从来源标记刷新，而不会持久化已解析的机密：环境引用使用环境变量名称，文件/exec 引用使用 `secretref-managed`。
    - 受 SecretRef 管理的标头值会以相同方式刷新，环境引用使用 `secretref-env:ENV_VAR_NAME`。
    - `models.json` 中为空或缺失的 `apiKey`/`baseUrl` 会回退到配置中的 `models.providers`。
    - 其他提供商字段会从配置和规范化后的目录数据中刷新。

  </Accordion>
</AccordionGroup>

标记持久化以来源为准：每当 OpenClaw 重新生成 `models.json` 时（包括 `openclaw agent` 等由命令驱动的路径），都会从活动来源配置快照（解析前）写入标记，而不是从运行时已解析的机密值写入。

## 相关内容

- [Agent Runtimes](/zh-CN/concepts/agent-runtimes) — OpenClaw、Codex 和其他 Agent loop 运行时
- [配置参考](/zh-CN/gateway/config-agents#agent-defaults) — 模型配置键
- [图像生成](/zh-CN/tools/image-generation) — 图像模型配置
- [模型故障转移](/zh-CN/concepts/model-failover) — 回退链
- [模型提供商](/zh-CN/concepts/model-providers) — 提供商路由和身份验证
- [模型 CLI 参考](/zh-CN/cli/models) — 完整的命令和标志参考
- [音乐生成](/zh-CN/tools/music-generation) — 音乐模型配置
- [视频生成](/zh-CN/tools/video-generation) — 视频模型配置
