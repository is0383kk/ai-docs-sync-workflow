---
read_when:
    - 查找特定的新手引导步骤或标志位
    - 使用非交互模式自动完成新手引导
    - 调试新手引导行为
sidebarTitle: Onboarding Reference
summary: CLI 新手引导完整参考：每个步骤、标志和配置字段
title: 新手引导参考
x-i18n:
    generated_at: "2026-07-26T06:34:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e5e7e42fa3fc1a6d85ad422d0d28dfeda233c89a4d7e97eee4fb974831816372
    source_path: reference/wizard.md
    workflow: 16
---

这是 `openclaw onboard` 的完整参考。
如需高层概览，请参阅[新手引导（CLI）](/zh-CN/start/wizard)。如需了解分步行为和输出，
请参阅 [CLI 设置参考](/zh-CN/start/wizard-cli-reference)。

## 流程详情（本地模式）

<Steps>
  <Step title="重置（可选）">
    - `--reset` 会在设置运行前重置状态；如果不使用它，重新运行新手引导
      将保留现有配置，并将其复用为默认值。
    - `--reset-scope` 控制 `--reset` 删除的内容：`config`（仅配置文件）、
      `config+creds+sessions`（默认），或 `full`（还会删除
      工作区）。
    - 如果配置文件无效，新手引导将停止，并提示你先运行
      `openclaw doctor`，然后重新运行设置。
    - 重置会将状态移至废纸篓（绝不会直接删除）。

  </Step>
  <Step title="风险确认">
    - 首次运行（或设置 `wizard.securityAcknowledgedAt` 之前的任何运行）
      会要求你确认已了解智能体功能强大，授予其完整的
      系统访问权限存在风险。
    - `--non-interactive` 要求显式指定 `--accept-risk`；如果未指定，
      新手引导将报错退出，而不会显示提示。
    - 交互式运行会显示确认提示而不使用该标志；拒绝确认将
      取消设置。

  </Step>
  <Step title="模型/身份验证">
    - **Anthropic API 密钥**：如果存在 `ANTHROPIC_API_KEY`，则使用它，否则提示输入密钥，然后保存该密钥供守护进程使用。
    - **Anthropic Claude CLI**：已有 Claude CLI 登录时的首选本地路径；OpenClaw 仍支持 Anthropic 设置令牌身份验证作为替代方案。
    - **OpenAI Code (Codex) 订阅（OAuth）**：浏览器流程；粘贴 `code#state`。
      - 在没有主模型的全新设置中，通过 Codex 运行时将 `agents.defaults.model` 设置为 `openai/gpt-5.6-sol`。
    - **OpenAI Code (Codex) 订阅（设备配对）**：使用短期设备代码的浏览器配对流程。
      - 在没有主模型的全新设置中，通过 Codex 运行时将 `agents.defaults.model` 设置为 `openai/gpt-5.6-sol`。
    - **OpenAI API 密钥**：如果存在 `OPENAI_API_KEY`，则使用它，否则提示输入密钥，然后将其存储在身份验证配置文件中。
      - 在没有主模型的全新设置中，将 `agents.defaults.model` 设置为 `openai/gpt-5.6`；不带限定的直接 API 模型 ID 会解析为 Sol 层级。
    - 添加 OpenAI 或重新对其进行身份验证时，会保留现有的显式主模型，包括 `openai/gpt-5.5`。如果该账户不提供 GPT-5.6，请显式选择 `openai/gpt-5.5`；OpenClaw 不会静默降级模型。
    - **xAI OAuth**：使用设备代码通过浏览器登录，无需 localhost 回调，因此也可通过 SSH/Docker/VPS 使用（`--auth-choice xai-oauth`）。
    - **xAI API 密钥**：提示输入 `XAI_API_KEY`（`--auth-choice xai-api-key`）。
    - `--auth-choice xai-device-code` 仍作为仅供手动使用的兼容性别名，执行相同的 xAI OAuth 设备代码流程；新脚本请使用 `xai-oauth`。
    - **OpenCode**：提示输入 `OPENCODE_API_KEY`（或 `OPENCODE_ZEN_API_KEY`，可在 https://opencode.ai/auth 获取），并允许你选择 Zen 或 Go 目录。
    - **Ollama**：首先提供 **Cloud + Local**、**Cloud only** 或 **Local only** 选项。`Cloud only` 会提示输入 `OLLAMA_API_KEY` 并使用 `https://ollama.com`；基于主机的模式会提示输入 Ollama 基础 URL（默认值为 `http://127.0.0.1:11434`），发现可用模型，并在需要时自动拉取所选的本地模型；`Cloud + Local` 还会检查该 Ollama 主机是否已登录以访问云端。
    - 更多详情：[Ollama](/zh-CN/providers/ollama)
    - **API 密钥**：为你存储密钥。
    - **Vercel AI Gateway（多模型代理）**：提示输入 `AI_GATEWAY_API_KEY`。
    - 更多详情：[Vercel AI Gateway](/zh-CN/providers/vercel-ai-gateway)
    - **Cloudflare AI Gateway**：提示输入 Account ID、Gateway ID 和 `CLOUDFLARE_AI_GATEWAY_API_KEY`。
    - 更多详情：[Cloudflare AI Gateway](/zh-CN/providers/cloudflare-ai-gateway)
    - **MiniMax**：自动写入配置；托管服务的默认值为 `MiniMax-M3`。
      API 密钥设置使用 `minimax/...`，OAuth 设置使用
      `minimax-portal/...`。
    - 更多详情：[MiniMax](/zh-CN/providers/minimax)
    - **StepFun**：针对中国或全球端点的 StepFun 标准版或 Step Plan 自动写入配置。
    - 标准版当前默认为 `step-3.5-flash`；Step Plan 还包括 `step-3.5-flash-2603`。
    - 更多详情：[StepFun](/zh-CN/providers/stepfun)
    - **Synthetic（兼容 Anthropic）**：提示输入 `SYNTHETIC_API_KEY`。
    - 更多详情：[Synthetic](/zh-CN/providers/synthetic)
    - **Moonshot（Kimi K2）**：自动写入配置。
    - **Kimi Coding**：自动写入配置。
    - 更多详情：[Moonshot AI（Kimi + Kimi Coding）](/zh-CN/providers/moonshot)
    - **自定义提供商**：适用于兼容 OpenAI、兼容 OpenAI Responses 或兼容 Anthropic 的端点。非交互式标志：`--auth-choice custom-api-key`、`--custom-base-url`、`--custom-model-id`、`--custom-api-key`（可选；回退到 `CUSTOM_API_KEY`）、`--custom-provider-id`（可选；根据基础 URL 自动派生）、`--custom-compatibility openai|openai-responses|anthropic`（默认为 `openai`）、`--custom-image-input` / `--custom-text-input`（覆盖推断的视觉模型检测）。
    - **跳过**：暂不配置身份验证。
    - 从检测到的选项中选择默认模型（或手动输入提供商/模型）。为获得最佳质量并降低提示注入风险，请选择提供商堆栈中可用的最强最新一代模型。
    - 新手引导会运行模型检查，并在配置的模型未知或缺少身份验证时发出警告。
    - API 密钥存储模式默认使用身份验证配置文件中的明文值。使用 `--secret-input-mode ref` 可改为存储由环境变量支持的引用（例如 `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`）；引用的环境变量必须已经设置，否则新手引导会立即失败。
    - 身份验证配置文件位于 `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（API 密钥 + OAuth）。`~/.openclaw/credentials/oauth.json` 仅用于旧版导入。
    - 更多详情：[OAuth](/zh-CN/concepts/oauth)
    <Note>
    无头/服务器提示：在有浏览器的计算机上完成 OAuth，然后将
    该智能体的 `auth-profiles.json`（例如
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`，或对应的
    `$OPENCLAW_STATE_DIR/...` 路径）复制到 Gateway 网关主机。`credentials/oauth.json`
    仅是旧版导入来源。
    </Note>
  </Step>
  <Step title="工作区">
    - 默认值为 `~/.openclaw/workspace`（可配置）。
    - 预置智能体引导初始化流程所需的工作区文件。
    - 完整的工作区布局和备份指南：[Agent 工作区](/zh-CN/concepts/agent-workspace)

  </Step>
  <Step title="Gateway 网关">
    - 端口（默认 **18789**）、绑定、身份验证模式、Tailscale 暴露。
    - 身份验证建议：即使使用环回地址，也应保留 **Token**，以便本地 WS 客户端必须进行身份验证。
    - 在令牌模式下，交互式设置提供：
      - **生成/存储明文令牌**（默认）
      - **使用 SecretRef**（选择启用）
      - 快速开始会在 `env`、`file` 和 `exec` 提供商之间复用现有的 `gateway.auth.token` SecretRef，用于新手引导探测/仪表板引导初始化。
      - 如果配置了该 SecretRef 但无法解析，新手引导会提前失败并显示明确的修复消息，而不是静默降低运行时身份验证的安全性。
    - 在密码模式下，交互式设置也支持明文或 SecretRef 存储。
    - 非交互式令牌 SecretRef 路径：`--gateway-token-ref-env <ENV_VAR>`。
      - 要求新手引导进程环境中存在非空环境变量。
      - 不能与 `--gateway-token` 结合使用。
    - 仅当你完全信任每个本地进程时，才可禁用身份验证。
    - 非环回绑定仍要求身份验证。

  </Step>
  <Step title="渠道">
    - [WhatsApp](/zh-CN/channels/whatsapp)：可选的二维码登录。
    - [Telegram](/zh-CN/channels/telegram)：Bot 令牌。
    - [Discord](/zh-CN/channels/discord)：Bot 令牌。
    - [Google Chat](/zh-CN/channels/googlechat)：服务账户 JSON + webhook 受众。
    - [Mattermost](/zh-CN/channels/mattermost)（插件）：Bot 令牌 + 基础 URL。
    - [Signal](/zh-CN/channels/signal)（插件）：可选安装 `signal-cli` + 账户配置。
    - [iMessage](/zh-CN/channels/imessage)：`imsg` CLI 路径 + Messages 数据库访问权限；当 Gateway 网关不在 Mac 上运行时，请使用 SSH 包装器。
    - Discord、Feishu、Microsoft Teams、QQ Bot、Slack 和其他渠道以
      插件形式提供，新手引导可为你安装。完整目录：[渠道](/zh-CN/channels)。
    - 私信安全：默认为配对。首条私信会发送代码；通过 `openclaw pairing approve <channel> <code>` 批准，或使用允许列表。

  </Step>
  <Step title="Web 搜索">
    - 选择受支持的提供商，例如 Brave、Codex（托管搜索）、DuckDuckGo、Exa、Firecrawl、Gemini、Grok、Kimi、MiniMax Search、Ollama Web 搜索、Parallel、Perplexity、SearXNG 或 Tavily（也可以跳过）。
    - 由 API 支持的提供商可以使用环境变量或现有配置进行快速设置；无密钥提供商则使用其特定于提供商的前置条件。
    - 使用 `--skip-search` 跳过。
    - 稍后配置：`openclaw configure --section web`。

  </Step>
  <Step title="安装守护进程">
    - macOS：LaunchAgent
      - 需要已登录的用户会话；对于无头环境，请使用自定义 LaunchDaemon（未随附）。
    - Linux（以及通过 WSL2 运行的 Windows）：systemd 用户单元
      - 新手引导会尝试通过 `loginctl enable-linger <user>` 启用 lingering，以便 Gateway 网关在注销后继续运行。
      - 可能会提示使用 sudo（写入 `/var/lib/systemd/linger`）；它会先尝试不使用 sudo。
    - 原生 Windows：优先使用计划任务；如果任务创建被拒绝，OpenClaw 会回退到每用户的“启动”文件夹登录项，并立即启动 Gateway 网关。
    - **运行时选择：**必须使用 Node，因为规范运行时状态存储使用 `node:sqlite`。修复期间会将旧版 Bun 服务迁移到 Node。
    - 如果令牌身份验证要求提供令牌，并且 `gateway.auth.token` 由 SecretRef 管理，则守护进程安装会验证它，但不会将解析后的明文令牌值持久保存到监管程序服务的环境元数据中。
    - 如果令牌身份验证要求提供令牌，而配置的令牌 SecretRef 无法解析，则会阻止守护进程安装并提供可操作的指导。
    - 如果同时配置了 `gateway.auth.token` 和 `gateway.auth.password`，且未设置 `gateway.auth.mode`，则在显式设置模式之前会阻止守护进程安装。

  </Step>
  <Step title="健康检查">
    - 启动 Gateway 网关（如有需要）并运行 `openclaw health`。
    - 提示：`openclaw status --deep` 会将实时 Gateway 健康探测添加到状态输出中，并在支持时包含渠道探测（要求 Gateway 网关可访问）。

  </Step>
  <Step title="Skills（推荐）">
    - 读取可用 Skills 并检查要求。
    - 允许你选择节点管理器：**npm / pnpm / bun**。
    - 自动安装受信任的内置 Skills 的可选依赖项（部分依赖项在 macOS 上使用 Homebrew）。
    - 跳过无法满足 Homebrew、uv 或 Go 安装程序前置条件的 Skills，将其归组并提供手动设置指南，并提示你在安装前置条件后使用 `openclaw doctor`。

  </Step>
  <Step title="完成">
    - 提供摘要和后续步骤，包括 **How do you want to hatch your agent?** 提示，可选择 Terminal、Browser 或稍后处理。

  </Step>
</Steps>

<Note>
如果未检测到 GUI，新手引导将输出用于 Control UI 的 SSH 端口转发说明，而不是打开浏览器。
如果缺少 Control UI 资源，新手引导会尝试构建它们；回退方案为 `pnpm ui:build`（自动安装 UI 依赖项）。
</Note>

## 非交互模式

使用 `--non-interactive --accept-risk` 自动执行新手引导或编写新手引导脚本（该
标志是必需的风险确认；如果没有该标志，新手引导将报错
退出）：

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

添加 `--json` 可获取机器可读的摘要。

非交互模式下的 Gateway 网关令牌 SecretRef：

```bash
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice skip \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN
```

`--gateway-token` 和 `--gateway-token-ref-env` 互斥。

<Note>
`--json` **并不**表示非交互模式。脚本应使用 `--non-interactive --accept-risk`（以及 `--workspace`）。
</Note>

特定于提供商的命令示例位于 [CLI 自动化](/zh-CN/start/wizard-cli-automation#provider-specific-examples)。
有关标志语义和步骤顺序，请使用此参考页面。

### 添加智能体（非交互）

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.6-sol \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

`main` 是保留的智能体 ID，不能用于 `openclaw agents add`。

## Gateway 网关向导 RPC

Gateway 网关通过 RPC（`wizard.start`、`wizard.next`、`wizard.cancel`、`wizard.status`）公开新手引导流程。
客户端（macOS 应用、Control UI）无需重新实现新手引导逻辑即可呈现各个步骤。

## Signal 设置（signal-cli）

新手引导会检测 `signal-cli` 是否位于 `PATH` 中；如果缺失，则会提供安装选项：

- Linux x86-64：从 `signal-cli` GitHub 发布页面下载官方原生 GraalVM 构建，并将其存储在 `~/.openclaw/tools/signal-cli/<version>/` 下。
- macOS 和其他架构：改为通过 Homebrew 安装。
- 原生 Windows：尚不支持；请在 WSL2 中运行新手引导以使用 Linux 安装路径。
- 无论采用哪种方式，都会写入包含 `kind: "managed-native"` 的 `channels.signal.transport.cliPath`。

## 向导写入的内容

`~/.openclaw/openclaw.json` 中的典型字段：

- `agents.defaults.workspace`
- 传入 `--skip-bootstrap` 时的 `agents.defaults.skipBootstrap`
- `agents.defaults.model` / `models.providers`（如果选择 Minimax）
- `tools.profile`（未设置时，本地新手引导默认为 `"coding"`；现有的显式值会予以保留）
- `gateway.*`（模式、绑定、身份验证、Tailscale）
- `session.dmScope`（新手引导会保留显式值，否则保持未设置，因此 `"main"` 默认值会将所有渠道的私信保留在智能体滚动的主会话中——这是个人智能体的默认设置。对于共享或多用户收件箱，请使用 `"per-channel-peer"`；当 `openclaw security audit` 检测到多用户私信流量时，会建议进行隔离。详情：[CLI 设置参考](/zh-CN/start/wizard-cli-reference#outputs-and-internals)）
- `channels.telegram.botToken`、`channels.discord.token`、`channels.matrix.*`、`channels.signal.*`、`channels.imessage.*`
- 在渠道提示过程中选择启用时配置的渠道私信允许列表。Discord、Matrix、Microsoft Teams 和 Slack 会尽可能将名称解析为 ID；其他渠道则直接接受 ID（例如 Telegram 发件人的数字 ID 或 WhatsApp 电话号码）。
- `skills.install.nodeManager`
  - `setup --node-manager` 接受 `npm`、`pnpm` 或 `bun`。
  - 手动配置仍可通过直接设置 `skills.install.nodeManager` 来使用 `yarn`。
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`
- `wizard.securityAcknowledgedAt`

`openclaw agents add` 会写入 `agents.entries.*` 和可选的 `bindings`。

WhatsApp 凭据位于 `~/.openclaw/credentials/whatsapp/<accountId>/` 下。
活动会话和转录文本存储在
`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` 中。
`~/.openclaw/agents/<agentId>/sessions/` 目录用于旧版迁移
输入以及归档/支持工件。

某些渠道以插件形式提供。在设置期间选择其中一个时，新手引导
会提示先安装该插件（通过 npm 或本地路径），然后才能进行配置。

## 相关文档

- 新手引导概览：[新手引导（CLI）](/zh-CN/start/wizard)
- CLI 设置参考：[CLI 设置参考](/zh-CN/start/wizard-cli-reference)
- macOS 应用新手引导：[新手引导](/zh-CN/start/onboarding)
- 配置参考：[Gateway 配置](/zh-CN/gateway/configuration)
- 提供商：[WhatsApp](/zh-CN/channels/whatsapp)、[Telegram](/zh-CN/channels/telegram)、[Discord](/zh-CN/channels/discord)、[Google Chat](/zh-CN/channels/googlechat)、[Signal](/zh-CN/channels/signal)、[iMessage](/zh-CN/channels/imessage)
- Skills：[Skills](/zh-CN/tools/skills)、[Skills 配置](/zh-CN/tools/skills-config)
