---
read_when:
    - 设计 macOS 新手引导助手
    - 实现身份验证或身份设置
sidebarTitle: 'Onboarding: macOS App'
summary: OpenClaw 首次运行设置流程（macOS 应用）
title: 新手引导（macOS 应用）
x-i18n:
    generated_at: "2026-07-26T07:01:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 55154774886c530de92b2110d367af24e2142fac48b901f288582d8552a6ca10
    source_path: start/onboarding.md
    workflow: 16
---

macOS 应用的首次运行流程：选择 Gateway 网关的运行位置，连接已验证的 AI 后端，授予权限，然后交由智能体执行自己的引导初始化流程。
有关 CLI 新手引导以及两种路径的对比，请参阅[新手引导概览](/zh-CN/start/onboarding-overview)。

<Steps>
<Step title="批准 macOS 警告">
<Frame>
<img src="/assets/macos-onboarding/01-macos-warning.jpeg" alt="" />
</Frame>
</Step>
<Step title="允许查找本地网络">
<Frame>
<img src="/assets/macos-onboarding/02-local-networks.jpeg" alt="" />
</Frame>
</Step>
<Step title="欢迎和安全通知">
<Frame caption="阅读显示的安全通知，并据此决定">
<img src="/assets/macos-onboarding/03-security-notice.png" alt="" />
</Frame>

安全信任模型：

- 默认情况下，OpenClaw 是个人智能体：仅有一个受信任的操作员边界。
- 共享/多用户设置需要严格限制：拆分信任边界，将工具访问权限保持在最低限度，并遵循[安全](/zh-CN/gateway/security)指南。
- 本地新手引导默认将新配置设为 `tools.profile: "coding"`，以便全新设置保留文件系统/运行时工具，而不使用不受限制的 `full` 配置文件。
- 如果启用了 hooks/webhooks 或其他不受信任的内容源，请使用强大的现代模型层级，并保持严格的工具策略/沙箱隔离。

</Step>
<Step title="本地与远程">
<Frame>
<img src="/assets/macos-onboarding/04-choose-gateway.png" alt="" />
</Frame>

**Gateway 网关**在哪里运行？

- **此 Mac（仅限本地）：**新手引导会配置身份验证，并在本地写入凭据。
- **远程（通过 SSH/Tailnet）：**新手引导**不会**配置本地身份验证；
  凭据必须已存在于 Gateway 网关主机上。远程 Gateway 网关令牌
  字段存储 macOS 应用用于连接该 Gateway 网关的令牌；
  现有的 `gateway.remote.token` SecretRef 值会保留，直到你
  将其替换。
- **稍后配置：**跳过设置，并让应用保持未配置状态。

<Tip>
**Gateway 网关身份验证提示：**

- 即使绑定到回环地址，Gateway 网关身份验证模式也默认为 `token`，因此本地 WS 客户端必须进行身份验证。
- 设置 `gateway.auth.mode: "none"` 后，任何本地进程都可以连接；仅在完全受信任的机器上使用此设置。
- 对于多机器访问或非回环地址绑定，请使用令牌。

</Tip>
</Step>
<Step title="CLI">
  本地设置通过 npm、pnpm 或 bun 安装全局 `openclaw` CLI，
  并优先使用 npm。对于 Gateway 网关本身，Node 仍是推荐的运行时。
  现有的兼容安装会被复用。
</Step>
<Step title="连接你的 AI">
  如果已连接的 Gateway 网关已经配置了智能体模型，则会完全跳过此
  页面并打开正常的智能体 UI。仅当 Gateway 网关为全新或配置不完整时，
  才会运行 OpenClaw 和提供商设置。

Gateway 网关就绪后，新手引导会查找你已有的 AI 访问方式：
Claude Code 或 Codex 登录、`OPENAI_API_KEY` / `ANTHROPIC_API_KEY`，或者
已安装在可访问的 Ollama 或 LM Studio 服务器中、具备工具能力且实测有效上下文
至少为 16K 的模型。检测在 Gateway 网关主机上运行，包括 macOS 应用连接到
Linux Gateway 网关时。系统会使用真实补全测试最佳选项，并且仅在
成功响应后才保存；测试失败时，应用会自动尝试下一个选项，
并显示上一个选项失败的原因。如果找到多个选项，你可以在继续之前
切换选择。自动本地发现绝不会拉取或下载模型。

如果要在 Gateway 网关主机没有 Claude CLI 登录的情况下使用 Claude 订阅，请在
任何已安装 Claude Code 的机器上运行 `claude setup-token`，然后将输出的
令牌粘贴到 **Connect with an API key or
token** 下的 **Anthropic setup-token** 中。

如果已安装的 Gemini CLI、Antigravity、Pi 和 OpenCode CLI 无法被选作可复用的引导式设置推理路径，
系统仍会显示它们以供参考。Gemini 和 Antigravity 无法强制执行无工具推理探测。
Pi 和 OpenCode 是完整的智能体 harness，而不是设置推理路径；它们的
会话集成需要单独配置运行时和插件。

你也可以通过提供商自己的 OAuth 或设备配对流程登录。
内置选项包括 OpenAI/ChatGPT、OpenRouter、GitHub Copilot、Google
Gemini CLI、xAI、MiniMax 全球版和中国版，以及 Chutes。该列表来自
Gateway 网关当前启用的文本推理提供商插件，而非固定的应用列表，
因此其他提供商无需添加特定于 macOS 的代码即可选择加入。

手动密钥/令牌选择器使用相同的提供商注册表。在每种路径中，
提供商都会提供其初始模型和配置；OpenClaw 会使用相同的实时测试
验证凭据，然后再存储其身份验证配置文件。在一个后端通过测试之前，
下一步始终处于锁定状态，因此首次智能体聊天无法在推理不可用的情况下
启动。实时检查通过后，OpenClaw 即可帮助配置其余工作区、Gateway 网关、渠道及
其他可选功能。当 OpenClaw 提供简短的选项列表时，应用会显示原生选项卡片；
选择其中一项会发送该选择，而 **Skip for
now** 始终会让该选择保持可选。稍后也可在
Settings → OpenClaw 下使用 OpenClaw。
</Step>
<Step title="导入记忆（检测到时显示）">
对于本地 Gateway 网关，新手引导会检查 Mac 上受支持的 AI
工具所生成的记忆：Claude Code 自动记忆、Codex 整合记忆和 Hermes 记忆
文件。找到任何内容后，此页面会列出各个来源及其记忆数量，
并允许你将选定来源导入智能体工作区的
`memory/imports/` 下，以供建立索引后回忆。已导入的文件会被跳过，
如果没有可导入的内容，则绝不会显示此页面。可以安全地跳过；之后可在
控制面板的记忆导入页面执行相同的导入，并逐个控制文件。
</Step>
<Step title="权限">

<Frame caption="选择要授予 OpenClaw 的权限">
<img src="/assets/macos-onboarding/05-permissions.png" alt="" />
</Frame>

新手引导会请求以下项目的 TCC 权限：自动化（AppleScript）、通知、辅助功能、屏幕录制、麦克风、语音识别、摄像头和位置。

</Step>
<Step title="完成">
  推理测试通过后，OpenClaw 会负责其余可选设置，并可
  将你转到正常的智能体聊天。完成权限引导流程
  会打开同一个聊天；在 OpenClaw 之前，应用不会创建工作区，也不会启动单独的
  智能体设置对话。请参阅
  [引导初始化](/zh-CN/start/bootstrapping)，了解智能体第一次真正轮次期间
  Gateway 网关主机上发生的情况。
</Step>
</Steps>

## 相关内容

- [新手引导概览](/zh-CN/start/onboarding-overview)
- [入门指南](/zh-CN/start/getting-started)
