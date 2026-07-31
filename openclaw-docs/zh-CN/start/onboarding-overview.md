---
read_when:
    - 选择新手引导路径
    - 设置新环境
sidebarTitle: Onboarding Overview
summary: OpenClaw 新手引导选项和流程概览
title: 新手引导概览
x-i18n:
    generated_at: "2026-07-26T06:29:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4bcda1dcfb91f388ca6bef59f9bdf5177571d93c0d89c45025ef837628fa7ba0
    source_path: start/onboarding-overview.md
    workflow: 16
---

OpenClaw 提供终端和 macOS 应用新手引导。两者都会先建立推理能力：
它们会检测现有的 AI 访问方式，要求成功完成一次实时补全，然后才启动
OpenClaw 以配置其余设置。如果可访问且已配置的 Gateway 网关中，
默认智能体已经配置模型，则会跳过新手引导和 OpenClaw，直接打开
常规智能体 UI。终端流程还提供完整的经典向导，用于
详细设置。

## 应该使用哪种方式？

|                | CLI 新手引导                           | macOS 应用新手引导             |
| -------------- | -------------------------------------- | ------------------------------ |
| **平台**       | macOS、Linux、Windows（原生或 WSL2）   | 仅限 macOS                     |
| **界面**       | 先设置推理，再进入 OpenClaw            | 先设置推理，再进入 OpenClaw    |
| **最适合**     | 服务器、无头环境、完全控制             | Mac 桌面设备、可视化设置       |
| **自动化**     | 脚本使用 `--non-interactive`            | 仅支持手动操作                 |
| **命令**       | `openclaw onboard`                     | 启动应用                       |

大多数用户应从 **CLI 新手引导** 开始——它适用于所有环境，并能让
你获得最大的控制权。

## 新手引导会配置什么

引导式推理阶段仅建立以下内容：

1. **模型提供商和身份验证**——检测到的访问方式，或已验证的提供商登录、
   API key 或令牌
2. **已验证的推理**——使用默认智能体的实际生效模型进行一次真实补全

该补全通过后，OpenClaw 可以配置工作区、Gateway 网关、
Gateway 网关服务、渠道、智能体、插件及其他可选功能。

经典 CLI 向导还可以配置：

1. **渠道**（可选）——内置和随附的聊天渠道，例如
   Discord、Feishu、Google Chat、iMessage、Mattermost、Microsoft Teams、
   Telegram、WhatsApp 等
2. **高级 Gateway 网关控制**——远程模式、网络设置和守护进程选项

## CLI 新手引导

在任意终端中运行：

```bash
openclaw onboard
```

引导流程会检测现有的 AI 访问方式，按顺序对候选项进行实时测试，
并在失败时继续尝试下一项。如果检测完所有候选项仍未成功，它会优先显示 OpenAI、
Anthropic、xAI (Grok)、Google 和 OpenRouter。**More…** 的第二级菜单中
包含按提供商分组的其他提供商，以及区域、套餐和支持的
浏览器、设备、API key 或令牌方式。只有在补全测试通过后，它才会保存模型
和凭据，然后启动 OpenClaw 以
配置工作区、Gateway 网关、渠道、智能体、插件及其他可选
功能。选择 **Skip for now** 会退出且不启动 OpenClaw。流程中
不会转入经典向导；如果要改用经典向导，请退出并运行
`openclaw onboard --classic`。

推理通过后，OpenClaw 可以将渠道设置交给使用掩码输入的终端
向导。该向导不会打开引导式或经典提供商设置；如需更改模型提供商
或其身份验证，请退出 OpenClaw 并运行 `openclaw onboard`。

使用 `openclaw onboard --classic` 可进行详细的模型/身份验证、渠道、技能、
远程 Gateway 网关或导入设置。添加 `--install-daemon` 还会选择
经典流程，并一步安装后台服务。使用 `openclaw
openclaw` 可通过对话方式进行非推理设置和修复。`openclaw
onboard --modern` 是使用同一实时推理
门槛的兼容性别名。

完整参考：[新手引导（CLI）](/zh-CN/start/wizard)
CLI 命令文档：[`openclaw onboard`](/zh-CN/cli/onboard)

## macOS 应用新手引导

打开 OpenClaw 应用。如果其已配置的本地或远程 Gateway 网关可访问，
且默认智能体已经配置模型，应用会跳过新手引导
和 OpenClaw，并立即打开常规智能体 UI。

对于全新或配置不完整的 Gateway 网关，首次运行流程会检测现有的 AI
访问方式（Claude Code、Codex 或 API key），实时测试最佳
选项，并仅在获得真实回复后保存——若失败则自动尝试后备选项，
如果未找到任何访问方式，则提供经过验证的手动 API key 设置步骤。敏感
凭据使用掩码输入。推理通过后，OpenClaw 会启动并
协助配置其余内容。

设置完成后，Gemini CLI 仍可供常规智能体使用，但不会用于
此推理门槛，因为它无法强制执行不使用工具的探测。

完整参考：[新手引导（macOS 应用）](/zh-CN/start/onboarding)

## 自定义或未列出的提供商

如果你的提供商未列出，请运行 `openclaw onboard --classic`，选择
**Custom Provider**，然后输入：

- 端点兼容性：OpenAI 兼容（`/chat/completions`）、OpenAI Responses 兼容（`/responses`）、Anthropic 兼容（`/messages`）或未知（探测全部三种并自动检测）
- 基础 URL 和 API key（如果端点不要求 API key，则可以不填）
- 模型 ID 和可选的模型别名

可以同时使用多个自定义端点——每个端点都有自己的端点 ID。

## 相关内容

- [入门指南](/zh-CN/start/getting-started)
- [CLI 设置参考](/zh-CN/start/wizard-cli-reference)
