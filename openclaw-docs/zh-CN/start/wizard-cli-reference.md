---
read_when:
    - 你需要了解某个特定 `openclaw onboard` 步骤的详细行为
    - 你正在调试新手引导结果或集成新手引导客户端
sidebarTitle: CLI reference
summary: openclaw onboard 的分步行为：每个步骤的作用、写入的配置及其内部机制
title: CLI 设置参考
x-i18n:
    generated_at: "2026-07-26T07:02:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 41bb9243ac7276b383274f4c27e3782b29e8ecf9d883229a44e3ab59aca5a34f
    source_path: start/wizard-cli-reference.md
    workflow: 16
---

本页介绍新手引导的分步行为、输出和内部机制。
如需演练，请参阅[新手引导（CLI）](/zh-CN/start/wizard)。如需完整的 CLI 标志
参考（每个 `--flag`、非交互式示例、提供商特定的
命令），请参阅 [`openclaw onboard`](/zh-CN/cli/onboard)。

## 向导的作用

本地模式（默认）将引导你完成：

- 模型和身份验证设置（Anthropic、OpenAI Code 订阅 OAuth、xAI、OpenCode、自定义端点以及更多由提供商负责的身份验证流程）
- 工作区位置和引导文件
- Gateway 网关设置（端口、绑定、身份验证、Tailscale）
- 渠道和提供商（Discord、Feishu、Google Chat、iMessage、Mattermost、Microsoft Teams、QQ Bot、Signal、Slack、Telegram、WhatsApp，以及其他内置或插件渠道）
- Web 搜索提供商（可选）
- 守护进程安装（LaunchAgent、systemd 用户单元，或原生 Windows 计划任务，并以启动文件夹作为后备方案）
- 健康检查
- Skills 设置

远程模式将此计算机配置为连接到其他位置的 Gateway 网关。它不会
在远程主机上安装或修改任何内容。

## 本地流程详情

<Steps>
  <Step title="检测现有配置">
    - 如果 `~/.openclaw/openclaw.json` 存在，请选择 **保留当前值**、**检查并更新** 或 **设置前重置**。
    - 重新运行向导不会清除任何内容，除非你明确选择 Reset（或传递 `--reset`）。
    - CLI `--reset` 默认为 `config+creds+sessions`；使用 `--reset-scope full` 还可移除工作区。
    - 如果配置无效或包含旧版键，向导将停止并要求你先运行 `openclaw doctor`，然后才能继续。
    - 重置会将状态移至废纸篓（绝不直接删除），并提供以下范围：
      - 仅配置
      - 配置 + 凭据 + 会话
      - 完全重置（同时移除工作区）

  </Step>
  <Step title="模型和身份验证">
    - 完整的选项矩阵位于[身份验证和模型选项](#auth-and-model-options)。

  </Step>
  <Step title="工作区">
    - 默认为 `~/.openclaw/workspace`（可配置）。
    - 生成首次运行引导所需的工作区文件。
    - 重新运行时，现有智能体名册会保留其整个智能体群的工作区，除非
      你明确确认移动。非交互式重新运行会发出警告并保留
      当前值。
    - 工作区布局：[Agent 工作区](/zh-CN/concepts/agent-workspace)。

  </Step>
  <Step title="Gateway 网关">
    - 提示输入端口、绑定、身份验证模式和 Tailscale 暴露方式。
    - 建议：即使使用环回，也应保持令牌身份验证启用，以便本地 WS 客户端必须进行身份验证。
    - 在令牌模式下，交互式设置提供：
      - **生成/存储明文令牌**（默认）
      - **使用 SecretRef**（选择启用）
    - 在密码模式下，交互式设置同样支持明文或 SecretRef 存储。
    - 非交互式令牌 SecretRef 路径：`--gateway-token-ref-env <ENV_VAR>`。
      - 要求新手引导进程的环境中存在非空环境变量。
      - 不能与 `--gateway-token` 结合使用。
    - 仅当你完全信任每个本地进程时才禁用身份验证。
    - 非环回绑定仍然需要身份验证。

  </Step>
  <Step title="渠道">
    - [WhatsApp](/zh-CN/channels/whatsapp)：可选的二维码登录
    - [Telegram](/zh-CN/channels/telegram)：Bot 令牌
    - [Discord](/zh-CN/channels/discord)：Bot 令牌
    - [Google Chat](/zh-CN/channels/googlechat)：服务账号 JSON + webhook 受众
    - [Mattermost](/zh-CN/channels/mattermost)：Bot 令牌 + 基础 URL
    - [Signal](/zh-CN/channels/signal)：可选的 `signal-cli` 安装 + 账号配置
    - [iMessage](/zh-CN/channels/imessage)：`imsg` CLI 路径 + Messages 数据库访问权限；当 Gateway 网关在非 Mac 设备上运行时，请使用 SSH 包装器
    - 私信安全：默认为配对。首次私信会发送一个代码；可通过
      `openclaw pairing approve <channel> <code>` 批准，或使用允许列表。
  </Step>
  <Step title="Web 搜索">
    - 选择一个提供商（Brave、DuckDuckGo、Exa、Firecrawl、Gemini、Grok、Kimi、MiniMax Search、Ollama Web 搜索、Perplexity、SearXNG、Tavily）或跳过。
    - 使用 `--skip-search` 跳过此步骤；之后可使用 `openclaw configure --section web` 重新配置。

  </Step>
  <Step title="安装守护进程">
    - macOS：LaunchAgent
      - 需要已登录的用户会话；对于无头环境，请使用自定义 LaunchDaemon（未随附）。
    - Linux 和通过 WSL2 运行的 Windows：systemd 用户单元
      - 向导会尝试执行 `loginctl enable-linger <user>`，使 Gateway 网关在注销后仍保持运行。
      - 可能会提示使用 sudo（写入 `/var/lib/systemd/linger`）；它会先尝试不使用 sudo。
    - 原生 Windows：优先使用计划任务
      - 如果创建任务被拒绝，OpenClaw 会回退到每用户的启动文件夹登录项，并立即启动 Gateway 网关。
      - 计划任务仍是首选，因为它们能提供更完善的监管器状态。
    - 运行时选择：必须使用 Node，因为 OpenClaw 的规范运行时状态存储使用 `node:sqlite`。

  </Step>
  <Step title="健康检查">
    - 启动 Gateway 网关（如有需要）并运行 `openclaw health`。
    - `openclaw status --deep` 会将实时 Gateway 健康探测添加到状态输出中，包括受支持时的渠道探测。

  </Step>
  <Step title="Skills">
    - 读取可用 Skills 并检查要求。
    - 允许你选择 Node 管理器：npm、pnpm 或 bun。
    - 当所需安装程序可用时，为受信任的内置 Skills 安装可选
      依赖项。
    - 跳过不可用的 Homebrew、uv 和 Go 安装程序，然后将受影响的
      Skills 与手动设置指南归组。安装缺少的
      前置条件后，运行 `openclaw doctor`。

  </Step>
  <Step title="完成">
    - 显示摘要和后续步骤，包括 iOS、Android 和 macOS 应用选项。

  </Step>
</Steps>

<Note>
如果未检测到 GUI，向导会输出 Control UI 的 SSH 端口转发说明，而不是打开浏览器。
如果缺少 Control UI 资源，向导会尝试构建它们；后备方案为 `pnpm ui:build`（自动安装 UI 依赖项）。
</Note>

## 远程模式详情

远程模式将此计算机配置为连接到其他位置的 Gateway 网关。它不会
在远程主机上安装或修改任何内容。

你需要设置：

- 远程 Gateway 网关 URL（`ws://...` 或 `wss://...`）
- 令牌、密码或无身份验证，需要与远程 Gateway 网关的配置一致

<Steps>
  <Step title="设备发现（可选）">
    如果 `dns-sd`（macOS）或 `avahi-browse`（Linux）可用，新手引导
    会先提供搜索 Bonjour/mDNS Gateway 网关信标的选项，然后才回退到
    手动输入 URL。配置后还会尝试广域 DNS-SD 设备发现。
    文档：[Gateway 网关设备发现](/zh-CN/gateway/discovery)、[Bonjour](/zh-CN/gateway/bonjour)。
  </Step>
  <Step title="连接方式">
    选择信标后，可选择直接 WebSocket 或 SSH 隧道：
    - **直接连接**：通过 `wss://` 连接，并提示信任发现的
      TLS 指纹（首次使用时信任并固定；仅在你接受后固定）。
    - **SSH 隧道**：输出一条需要先运行的 `ssh -N -L 18789:127.0.0.1:18789 <user>@<host>`
      命令，然后连接到本地隧道端点。
  </Step>
  <Step title="身份验证">
    选择令牌（推荐）、密码或无身份验证，然后可选择将其存储为
    SecretRef，而不是明文。
  </Step>
</Steps>

<Note>
如果 Gateway 网关仅绑定环回地址且无法被发现，请手动使用 SSH 隧道或 tailnet。
对于环回地址、私有 IP 字面量、`.local` 和 Tailnet `*.ts.net` URL，可以使用明文 `ws://`；其他私有 DNS 名称需要 `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1`。
</Note>

## 身份验证和模型选项

如果交互式新手引导中的提供商设置步骤失败（例如在本地未登录时选择复用 CLI），
向导会显示错误并返回提供商选择器，
而不是退出。显式 `--auth-choice` 运行仍会快速失败，以便自动化处理。

<AccordionGroup>
  <Accordion title="Anthropic API 密钥">
    如果存在，则使用 `ANTHROPIC_API_KEY`；否则提示输入密钥，然后保存以供守护进程使用。
  </Accordion>
  <Accordion title="Anthropic Claude CLI">
    交互式新手引导/配置中的首选本地路径；可用时复用现有 Claude CLI 登录。
  </Accordion>
  <Accordion title="OpenAI Code 订阅（OAuth）">
    浏览器流程；粘贴 `code#state`。

    在未设置主模型的全新设置中，通过 Codex 运行时将 `agents.defaults.model`
    设置为 `openai/gpt-5.6-sol`。

  </Accordion>
  <Accordion title="OpenAI Code 订阅（设备配对）">
    使用短期设备代码的浏览器配对流程。

    在未设置主模型的全新设置中，通过 Codex 运行时将 `agents.defaults.model`
    设置为 `openai/gpt-5.6-sol`。

  </Accordion>
  <Accordion title="OpenAI API 密钥">
    如果存在，则使用 `OPENAI_API_KEY`；否则提示输入密钥，然后将凭据存储在身份验证配置文件中。

    在未设置主模型的全新设置中，将 `agents.defaults.model` 设置为
    `openai/gpt-5.6`；不带前缀的直接 API 模型 ID 会解析到 Sol 层级。

    添加 OpenAI 或重新进行身份验证时，会保留现有的显式主
    模型，包括 `openai/gpt-5.5`。如果该账号未提供 GPT-5.6，
    请明确选择 `openai/gpt-5.5`；OpenClaw 不会静默降级它。

  </Accordion>
  <Accordion title="xAI (Grok) OAuth">
    适用于符合条件的 SuperGrok 或 X Premium 账户的浏览器登录方式。这是大多数用户
    推荐使用的 xAI 路径。OpenClaw 会为 Grok 模型、Grok `web_search`、
    `x_search` 和 `code_execution` 存储生成的身份验证配置文件。
  </Accordion>
  <Accordion title="xAI (Grok) 设备代码">
    使用短代码而非 localhost 回调、适合远程环境的浏览器登录方式。通过 SSH、Docker
    或 VPS 主机使用时，请选择此方式。
  </Accordion>
  <Accordion title="xAI (Grok) API 密钥">
    提示输入 `XAI_API_KEY`，并将 xAI 配置为模型提供商。当你想使用 xAI Console
    API 密钥而非订阅 OAuth 时，请选择此方式。
  </Accordion>
  <Accordion title="OpenCode">
    提示输入 `OPENCODE_API_KEY`（或 `OPENCODE_ZEN_API_KEY`），并允许你选择 Zen 或 Go 目录（一个 API 密钥同时适用于两者）。
    设置 URL：[opencode.ai/auth](https://opencode.ai/auth)。
  </Accordion>
  <Accordion title="API 密钥（通用）">
    为你存储密钥。
  </Accordion>
  <Accordion title="Vercel AI Gateway">
    提示输入 `AI_GATEWAY_API_KEY`。
    了解详情：[Vercel AI Gateway](/zh-CN/providers/vercel-ai-gateway)。
  </Accordion>
  <Accordion title="Cloudflare AI Gateway">
    提示输入账户 ID、Gateway 网关 ID 和 `CLOUDFLARE_AI_GATEWAY_API_KEY`。
    了解详情：[Cloudflare AI Gateway](/zh-CN/providers/cloudflare-ai-gateway)。
  </Accordion>
  <Accordion title="MiniMax">
    配置会自动写入。托管服务默认值为 `MiniMax-M3`；API 密钥设置使用
    `minimax/...`，OAuth 设置使用 `minimax-portal/...`。
    了解详情：[MiniMax](/zh-CN/providers/minimax)。
  </Accordion>
  <Accordion title="StepFun">
    系统会为中国或全球端点上的 StepFun 标准版或 Step Plan 自动写入配置。
    标准版目前包括 `step-3.5-flash`，Step Plan 还包括 `step-3.5-flash-2603`。
    了解详情：[StepFun](/zh-CN/providers/stepfun)。
  </Accordion>
  <Accordion title="Synthetic（兼容 Anthropic）">
    提示输入 `SYNTHETIC_API_KEY`。
    了解详情：[Synthetic](/zh-CN/providers/synthetic)。
  </Accordion>
  <Accordion title="Ollama（云端和本地开放模型）">
    首先提示选择 `Cloud + Local`、`Cloud only` 或 `Local only`。
    `Cloud only` 使用 `OLLAMA_API_KEY` 和 `https://ollama.com`。
    由主机支持的模式会提示输入基础 URL（默认值为 `http://127.0.0.1:11434`）、发现可用模型并建议默认值。
    `Cloud + Local` 还会检查该 Ollama 主机是否已登录以获取云端访问权限。
    了解详情：[Ollama](/zh-CN/providers/ollama)。
  </Accordion>
  <Accordion title="Moonshot 和 Kimi Coding">
    Moonshot（Kimi K2）和 Kimi Coding 配置会自动写入。
    了解详情：[Moonshot AI（Kimi + Kimi Coding）](/zh-CN/providers/moonshot)。
  </Accordion>
  <Accordion title="自定义提供商">
    适用于兼容 OpenAI、兼容 OpenAI Responses 和兼容 Anthropic 的端点。

    交互式新手引导支持与其他提供商 API 密钥流程相同的 API 密钥存储选项：
    - **立即粘贴 API 密钥**（明文）
    - **使用密钥引用**（环境变量引用或已配置的提供商引用，包含预检验证）

    新手引导会为常见视觉模型 ID（GPT-4o/4.1/5.x、Claude 3/4、Gemini、Qwen-VL、LLaVA、Pixtral 及类似模型）推断图像支持能力，仅在模型名称未知时才会询问。

    非交互式标志：
    - `--auth-choice custom-api-key`
    - `--custom-base-url`
    - `--custom-model-id`
    - `--custom-api-key`（可选；回退到 `CUSTOM_API_KEY`）
    - `--custom-provider-id`（可选）
    - `--custom-compatibility <openai|openai-responses|anthropic>`（可选；默认值为 `openai`）
    - `--custom-image-input` / `--custom-text-input`（可选；覆盖推断出的模型输入能力）

  </Accordion>
  <Accordion title="跳过">
    不配置身份验证。
  </Accordion>
</AccordionGroup>

模型行为：

- 从检测到的选项中选择默认模型，或手动输入提供商和模型。
- 当新手引导从提供商身份验证选项开始时，模型选择器会自动优先选择
  该提供商。对于 Volcengine 和 BytePlus，同一偏好还会匹配其编码套餐变体
  （`volcengine-plan/*`、`byteplus-plan/*`）。
- 如果按首选提供商筛选后结果为空，选择器会回退到
  完整目录，而不是不显示任何模型。
- 向导会运行模型检查，并在配置的模型未知或缺少身份验证时发出警告。

凭据和配置文件路径：

- 身份验证配置文件（API 密钥 + OAuth）：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- 旧版 OAuth 导入：`~/.openclaw/credentials/oauth.json`

凭据存储模式：

- 默认的新手引导行为会将 API 密钥作为明文值持久化到身份验证配置文件中。
- `--secret-input-mode ref` 会启用引用模式，而不是存储明文密钥。
  在交互式设置中，你可以选择以下任一方式：
  - 环境变量引用（例如 `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`）
  - 已配置的提供商引用（`file` 或 `exec`），包含提供商别名和 ID
- 交互式引用模式会在保存前运行快速预检验证。
  - 环境变量引用：验证当前新手引导环境中的变量名称和非空值。
  - 提供商引用：验证提供商配置并解析请求的 ID。
  - 如果预检失败，新手引导会显示错误并允许你重试。
- 在非交互模式下，`--secret-input-mode ref` 仅支持环境变量。
  - 在新手引导进程的环境中设置提供商环境变量。
  - 内联密钥标志（例如 `--openai-api-key`）要求已设置该环境变量；否则新手引导会立即失败。
  - 对于自定义提供商，非交互式 `ref` 模式会将 `models.providers.<id>.apiKey` 存储为 `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`。
  - 在该自定义提供商场景中，`--custom-api-key` 要求已设置 `CUSTOM_API_KEY`；否则新手引导会立即失败。
- Gateway 网关身份验证凭据在交互式设置中支持明文和 SecretRef 选项：
  - 令牌模式：**生成/存储明文令牌**（默认）或**使用 SecretRef**。
  - 密码模式：明文或 SecretRef。
- 非交互式令牌 SecretRef 路径：`--gateway-token-ref-env <ENV_VAR>`。
- 现有明文设置将继续正常工作，无需更改。

<Note>
无界面和服务器提示：在配有浏览器的计算机上完成 OAuth，然后将
该智能体的 `auth-profiles.json`（例如
`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`，或对应的
`$OPENCLAW_STATE_DIR/...` 路径）复制到 Gateway 网关主机。`credentials/oauth.json`
仅用作旧版导入源。
</Note>

## 输出和内部机制

`~/.openclaw/openclaw.json` 中的典型字段：

- `agents.defaults.workspace`
- 传入 `--skip-bootstrap` 时的 `agents.defaults.skipBootstrap`
- `agents.defaults.model` / `models.providers`（如果选择 Minimax）
- `tools.profile`（未设置时，本地新手引导默认为 `"coding"`；保留现有的显式值）
- `gateway.*`（模式、绑定、身份验证、Tailscale）
- `session.dmScope`（新手引导会保留显式值，否则保持未设置，因此 `main` 默认值会将所有渠道中的私信保留在该智能体的滚动主会话中——这是个人智能体的默认设置。对于共享或多用户收件箱，请使用 `per-channel-peer`；当 `openclaw security audit` 检测到多用户私信流量时，会建议进行隔离）
- `channels.telegram.botToken`、`channels.discord.token`、`channels.matrix.*`、`channels.signal.*`、`channels.imessage.*`
- 在提示过程中选择启用时的渠道允许列表（Discord、iMessage、Signal、Slack、Telegram、WhatsApp）；Discord 和 Slack 还会将输入的名称解析为 ID
- `skills.install.nodeManager`
  - `setup --node-manager` 标志接受 `npm`、`pnpm` 或 `bun`。
  - 之后仍可通过手动配置设置 `skills.install.nodeManager: "yarn"`。
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`
- `wizard.securityAcknowledgedAt`

`openclaw agents add` 会写入 `agents.entries.*` 和可选的 `bindings`。

WhatsApp 凭据位于 `~/.openclaw/credentials/whatsapp/<accountId>/` 下。
活动会话和转录记录存储在
`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` 中。
`~/.openclaw/agents/<agentId>/sessions/` 目录用于旧版迁移
输入以及归档/支持工件。

<Note>
某些渠道以插件形式提供。在设置期间选择这些渠道时，向导会先提示安装插件
（npm 或本地路径），然后再进行频道配置。
</Note>

### 已安装应用推荐

模型访问检查成功后，macOS 上的经典交互式新手引导会扫描应用程序名称和软件包 ID，而无需请求 macOS 隐私权限。它会搜索官方插件目录和 ClawHub，然后要求配置的模型排除错误的名称匹配，并推荐相关插件或 Skills。推荐的匹配项默认处于选中状态；可选匹配项需要显式选择。

结果屏幕会列出检测到的应用程序，并显示：“应用程序名称通过你配置的模型和 ClawHub 搜索进行匹配。”将 `wizard.appRecommendations` 设置为 `false`，可同时禁用此新手引导步骤以及 Gateway 网关对节点应用清单的访问。快速开始或非 macOS 新手引导不会使用此扫描。

## 非交互式设置

`--non-interactive` 要求使用 `--accept-risk`（确认智能体功能强大，
授予完整系统访问权限存在风险）：

```bash
openclaw onboard --non-interactive --accept-risk \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY"
```

完整标志参考和特定提供商示例：[`openclaw onboard`](/zh-CN/cli/onboard)、[CLI 自动化](/zh-CN/start/wizard-cli-automation)。

## Gateway 网关向导 RPC

- `wizard.start`
- `wizard.next`
- `wizard.cancel`
- `wizard.status`

客户端（macOS 应用和 Control UI）无需重新实现新手引导逻辑即可渲染步骤。

## Signal 设置行为

- 从官方 `signal-cli` GitHub 发行版下载适当的发行资源（原生构建，仅支持 Linux x86-64）
- 在其他平台（macOS、非 x64 Linux）上，改为通过 Homebrew 安装
- 将发行资源安装存储在 `~/.openclaw/tools/signal-cli/<version>/` 下
- 在配置中写入包含 `kind: "managed-native"` 的 `channels.signal.transport.cliPath`
- 尚不支持原生 Windows；请在 WSL2 中运行新手引导，以使用 Linux 安装路径

## 相关文档

- 新手引导中心：[新手引导（CLI）](/zh-CN/start/wizard)
- 自动化和脚本：[CLI 自动化](/zh-CN/start/wizard-cli-automation)
- 命令参考：[`openclaw onboard`](/zh-CN/cli/onboard)
