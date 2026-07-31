---
read_when:
    - 选择或切换模型、配置别名
    - 调试模型故障转移 / “所有模型均失败”
    - 了解身份验证配置文件及其管理方法
sidebarTitle: Models FAQ
summary: 常见问题：模型默认值、选择、别名、切换、故障转移和身份验证配置文件
title: 常见问题：模型和身份验证
x-i18n:
    generated_at: "2026-07-26T06:48:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0c46d99352c5e51af5917c6f62b897dfa4030cb0201b8235e28f2f81f2954544
    source_path: help/faq-models.md
    workflow: 16
---

模型和身份验证配置文件问答。有关设置、会话、Gateway 网关、渠道和故障排除，请参阅主[常见问题](/zh-CN/help/faq)。

## 模型：默认值、选择、别名和切换

<AccordionGroup>
  <Accordion title='什么是“默认模型”？'>
    通过以下配置设置：

    ```text
    agents.defaults.model.primary
    ```

    模型是 `provider/model` 引用（例如：`openai/gpt-5.5`、
    `anthropic/claude-sonnet-4-6`）。始终显式设置 `provider/model`。如果
    省略提供商，OpenClaw 会先尝试匹配别名，然后在已配置的提供商中查找
    具有该模型 ID 的唯一匹配项，最后回退到已配置的默认提供商
    （已弃用的兼容路径）。如果该提供商不再拥有已配置的默认模型，
    OpenClaw 会回退到第一个已配置的提供商/模型，而不是使用过时的默认值。

  </Accordion>

  <Accordion title="推荐使用什么模型？">
    使用你的提供商栈所提供的最新一代最强模型，尤其是对于启用了工具或
    接收不可信输入的智能体——较弱或过度量化的模型更容易受到提示词注入
    和不安全行为的影响（请参阅[安全](/zh-CN/gateway/security)）。根据智能体角色，
    将更便宜的模型分配给常规或低风险聊天。

    按智能体分配模型，并使用子智能体并行处理耗时任务（每个子智能体
    都会消耗自己的 token）。请参阅[Models](/zh-CN/concepts/models)、
    [子智能体](/zh-CN/tools/subagents)、[MiniMax](/zh-CN/providers/minimax)和
    [本地模型](/zh-CN/gateway/local-models)。

  </Accordion>

  <Accordion title="如何在不清空配置的情况下切换模型？">
    仅更改模型字段——避免替换完整配置。

    - 在聊天中使用 `/model`（按会话生效，请参阅[斜杠命令](/zh-CN/tools/slash-commands)）
    - `openclaw models set ...`（仅更新模型配置）
    - `openclaw configure --section model`（交互式）
    - 直接在 `~/.openclaw/openclaw.json` 中编辑 `agents.defaults.model`

    对于 RPC 编辑，先使用 `config.schema.lookup` 检查（规范化路径、
    浅层架构文档和子项摘要），然后优先使用 `config.patch`，
    而不是通过部分对象使用 `config.apply`。如果确实覆盖了配置，
    请从备份恢复，或运行 `openclaw doctor` 进行修复。

    文档：[Models](/zh-CN/concepts/models)、[配置](/zh-CN/cli/configure)、
    [配置](/zh-CN/cli/config)、[Doctor](/zh-CN/gateway/doctor)。

  </Accordion>

  <Accordion title="可以使用自托管模型（llama.cpp、vLLM、Ollama）吗？">
    可以——Ollama 是最简单的方案。快速设置：

    1. 从 `https://ollama.com/download` 安装 Ollama
    2. 拉取本地模型，例如 `ollama pull gemma4`
    3. 若还要使用云端模型，请运行 `ollama signin`
    4. 运行 `openclaw onboard`，选择 `Ollama`，然后选择 `Local` 或 `Cloud + Local`

    `Cloud + Local` 可同时提供云端模型和本地 Ollama 模型；
    `kimi-k2.5:cloud` 等云端模型无需在本地拉取。若要手动切换：
    先运行 `openclaw models list`，再运行 `openclaw models set ollama/<model>`。

    较小或高度量化的模型更容易受到提示词注入攻击。任何可访问工具的
    Bot 都应使用大型模型；如果仍要使用小型模型，请启用沙箱隔离和
    严格的工具允许列表。

    文档：[Ollama](/zh-CN/providers/ollama)、[本地模型](/zh-CN/gateway/local-models)、
    [模型提供商](/zh-CN/concepts/model-providers)、[安全](/zh-CN/gateway/security)、
    [沙箱隔离](/zh-CN/gateway/sandboxing)。

  </Accordion>

  <Accordion title="如何即时切换模型（无需重启）？">
    将 `/model <name>` 作为单独消息发送。完整命令列表请参阅
    [斜杠命令](/zh-CN/tools/slash-commands)，其中包括编号选择器
    （`/model`、`/model
    list`、`/model 3`）、
    用于清除会话覆盖的 `/model default`，以及用于查看端点/API 模式详情的
    `/model status`。

    使用 `@profile` 为每个会话强制指定身份验证配置文件：

    ```text
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    若要取消固定通过 `@profile` 设置的配置文件，请重新运行
    不带后缀的 `/model`（例如 `/model anthropic/claude-opus-4-6`），或从
    `/model` 中选择默认项。使用 `/model status`
    确认当前启用的身份验证配置文件。

  </Accordion>

  <Accordion title="如果两个提供商公开相同的模型 ID，/model 会使用哪一个？">
    `/model provider/model` 会选择该确切的提供商路由。例如，
    即使模型 ID 相同，`qianfan/deepseek-v4-flash` 和 `deepseek/deepseek-v4-flash`
    也是不同的引用——OpenClaw 不会仅因裸 ID 匹配而静默切换提供商。

    用户选择的 `/model` 引用采用严格回退策略：如果该
    提供商/模型不可用，回复会明确失败，而不会回退到
    `agents.defaults.model.fallbacks`。已配置的回退链仍适用于已配置的默认值、
    定时任务主模型和自动选择的回退状态。当允许没有会话覆盖的运行
    使用回退时，OpenClaw 会先尝试请求的提供商/模型，然后尝试已配置的
    回退项，最后尝试已配置的主模型——因此，重复的裸模型 ID 绝不会
    直接跳回默认提供商。

    请参阅[Models](/zh-CN/concepts/models)和[模型故障转移](/zh-CN/concepts/model-failover)。

  </Accordion>

  <Accordion title="可以将 GPT 5.5 用于日常任务，将 Codex 5.5 用于编码吗？">
    可以——模型选择和运行时选择彼此独立：

    - **原生 Codex 编码智能体：**将 `agents.defaults.model.primary` 设置为
      `openai/gpt-5.5`。使用 `openclaw models auth login --provider
      openai` 登录，以通过 ChatGPT/Codex 订阅进行身份验证。
    - **Agent loop 之外的直接 OpenAI API 任务：**为图像、嵌入、
      语音、实时处理和其他非智能体 OpenAI API 界面配置
      `OPENAI_API_KEY`。
    - **OpenAI 智能体 API 密钥身份验证：**使用 `/model openai/gpt-5.5`
      和一个有序的 `openai` API 密钥配置文件。
    - **子智能体：**将编码任务路由到专注于 Codex 的智能体，
      并为其配置独立的 `openai/gpt-5.5` 模型。

    请参阅[Models](/zh-CN/concepts/models)和[斜杠命令](/zh-CN/tools/slash-commands)。

  </Accordion>

  <Accordion title="如何为 GPT 5.5 配置快速模式？">
    - **按会话：**使用 `openai/gpt-5.5` 时发送 `/fast on`。
    - **按模型设置默认值：**将
      `agents.defaults.models["openai/gpt-5.5"].params.fastMode` 设置为 `true`。
    - **自动截止：**`/fast auto` 或 `params.fastMode: "auto"` 会让新的
      模型调用在截止时间前使用快速模式，截止后进行的重试、回退、
      工具结果或继续调用则不使用快速模式。截止时间默认为
      60 秒；可通过模型上的 `params.fastAutoOnSeconds` 覆盖。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: {
                fastMode: "auto",
                fastAutoOnSeconds: 30,
              },
            },
          },
        },
      },
    }
    ```

    在原生 OpenAI Responses 请求中，快速模式映射到
    `service_tier = "priority"`；现有的 `service_tier` 值会保留，并且快速模式
    不会重写 `reasoning` 或 `text.verbosity`。会话级
    `/fast` 覆盖优先于配置默认值。

    请参阅[思考和快速模式](/zh-CN/tools/thinking)，以及 [OpenAI](/zh-CN/providers/openai)
    提供商页面“高级配置”下的“快速模式”部分。

  </Accordion>

  <Accordion title='为什么会看到“Model ... is not allowed”，之后却没有回复？'>
    如果 `agents.defaults.modelPolicy.allow` 非空，它将成为 `/model`、
    会话覆盖和 `--model` 的**允许列表**。选择列表之外的模型时，
    会返回以下内容，而不是正常回复：

    ```text
    Model override "provider/model" is not allowed by agents.defaults.modelPolicy.allow.
    ```

    修复方法：将确切模型或 `"provider/*"` 等提供商通配符添加到指定的
    `modelPolicy.allow` 列表中；移除或清空该列表；或者从
    `/model list` 中选择模型。如果命令还包含
    `--runtime codex`，请先更新允许列表，然后重试相同的
    `/model provider/model --runtime codex` 命令。

  </Accordion>

  <Accordion title='为什么会看到“Unknown model: minimax/MiniMax-M3”？'>
    如果使用的是较旧版本的 OpenClaw，请先升级（或通过
    `main` 从源代码运行），然后重启 Gateway 网关——
    安装版本的目录中可能尚未包含 `MiniMax-M3`。否则，说明
    MiniMax 提供商尚未配置（未找到提供商条目或身份验证配置文件），
    因此无法解析该模型。完整的修复检查清单、提供商/模型 ID 表格和
    配置块示例，请参阅 [MiniMax](/zh-CN/providers/minimax) 提供商页面的
    “故障排查”部分。

  </Accordion>

  <Accordion title="可以将 MiniMax 设为默认模型，并使用 OpenAI 处理复杂任务吗？">
    可以。将 MiniMax 设为默认模型，并按会话切换模型——回退机制用于处理
    错误，而不是处理“困难任务”，因此请使用 `/model`
    或单独的智能体。

    **选项 A：按会话切换**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "minimax" },
            "openai/gpt-5.5": { alias: "gpt" },
          },
        },
      },
    }
    ```

    然后运行 `/model gpt`。

    **选项 B：使用不同的智能体**——智能体 A 默认使用 MiniMax，智能体 B
    默认使用 OpenAI；可按智能体路由，或使用 `/agent` 切换。

    文档：[Models](/zh-CN/concepts/models)、[多智能体路由](/zh-CN/concepts/multi-agent)、
    [MiniMax](/zh-CN/providers/minimax)、[OpenAI](/zh-CN/providers/openai)。

  </Accordion>

  <Accordion title="opus / sonnet / gpt 是内置快捷方式吗？">
    是——它们是内置简写，仅当目标模型存在于 `agents.defaults.models`
    中时才会应用：

    | 别名 | 解析为 |
    | --- | --- |
    | `opus` | `anthropic/claude-opus-5` |
    | `sonnet` | `anthropic/claude-sonnet-5` |
    | `gpt` | `openai/gpt-5.4` |
    | `gpt-mini` | `openai/gpt-5.4-mini` |
    | `gpt-nano` | `openai/gpt-5.4-nano` |
    | `gemini` | `google/gemini-3.1-pro-preview` |
    | `gemini-flash` | `google/gemini-3-flash-preview` |
    | `gemini-flash-lite` | `google/gemini-3.1-flash-lite` |

    同名的自定义别名会覆盖内置别名。

  </Accordion>

  <Accordion title="如何定义或覆盖模型快捷方式（别名）？">
    别名位于 `agents.defaults.models.<modelId>.alias`：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
          },
        },
      },
    }
    ```

    之后，`/model sonnet`（或在支持时使用 `/<alias>`）
    会解析为该模型 ID。

  </Accordion>

  <Accordion title="如何添加 OpenRouter 或 Z.AI 等其他提供商的模型？">
    OpenRouter（按 token 付费；提供多种模型）：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI（GLM 模型）：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5.1" },
          models: { "zai/glm-5.1": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    如果被引用的提供商/模型缺少提供商密钥，运行时会引发身份验证错误
    （例如 `No API key found for provider "zai"`）。

    **添加新智能体后找不到 API 密钥**

    新智能体的身份验证存储为空——身份验证按智能体独立管理，存储于：

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    修复方法：运行 `openclaw agents add <id>` 并在向导中配置身份验证，或者
    仅从主智能体的存储中复制可移植的静态 `api_key`/`token` 配置文件。
    对于 OAuth，当新智能体需要自己的账户时，请从该智能体登录。有关完整的
    `agentDir` 复用和凭据共享规则，请参阅[多智能体路由](/zh-CN/concepts/multi-agent)——绝不要在智能体之间复用
    `agentDir`。

  </Accordion>
</AccordionGroup>

## 模型故障转移和“All models failed”

<AccordionGroup>
  <Accordion title="故障转移如何工作？">
    分为两个阶段：

    1. 同一提供商内的**身份验证配置文件轮换**。
    2. **模型回退**到 `agents.defaults.model.fallbacks` 中的下一个模型。

    失败的配置文件会进入冷却期（指数退避），因此当提供商受到速率限制或暂时发生故障时，OpenClaw
    仍可继续响应。

    速率限制分类涵盖的不仅仅是普通的 `429`：`Too many concurrent
    requests`、`ThrottlingException`、`concurrency limit reached`、`workers_ai
    ... quota limit exceeded`、`resource exhausted` 以及周期性的
    使用窗口限制（`weekly/monthly limit reached`）都算作
    值得触发故障转移的速率限制。

    计费响应并不总是 `402`，有些 `402` 仍会归入
    瞬态/速率限制分类，而不是计费分类。`401`/`403` 中明确的
    计费文本仍可路由到计费分类；提供商特定的
    文本匹配器（例如 OpenRouter `Key limit exceeded`）仍仅限于其
    自身提供商。如果 `402` 看起来像可重试的使用窗口限制或
    组织/工作区支出限制（`daily limit reached, resets tomorrow`、
    `organization spending limit exceeded`），则会将其视为 `rate_limit`，而不是
    长期计费禁用。

    上下文溢出错误完全不会进入回退路径——
    `request_too_large`、`input exceeds the maximum number of tokens`、
    `input token count exceeds the maximum number of input tokens`、`input is
    too long for the model` 或 `ollama error: context length exceeded` 等特征会进入
    压缩/重试流程，而不是推进模型回退。

    通用服务器错误文本的范围比“任何包含 unknown/error
    的内容”更窄。以下提供商限定的瞬态形式会被视为故障转移
    信号：Anthropic 的纯 `An unknown error occurred`、OpenRouter 的纯
    `Provider returned error`、`Unhandled stop reason:
    error` 等停止原因错误、带有瞬态服务器文本（`internal
    server error`、`unknown error, 520`、`upstream error`、`backend error`）的 JSON `api_error` 载荷，
    以及提供商上下文匹配时类似 `ModelNotReadyException` 的提供商繁忙错误。
    `LLM request failed
    with an unknown error.` 等通用内部回退文本会保持保守，仅凭其本身不会触发回退。

  </Accordion>

  <Accordion title='"No credentials found for profile anthropic:default" 是什么意思？'>
    身份验证配置文件 ID `anthropic:default` 在
    预期的身份验证存储中没有凭据。

    **修复检查清单：**

    - 确认配置文件的存储位置——当前：
      `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`；旧版：
      `~/.openclaw/agent/*`（由 `openclaw doctor` 迁移）。
    - 确认 Gateway 网关已加载你的环境变量。仅在
      shell 中设置的 `ANTHROPIC_API_KEY` 不会传递给通过 systemd/launchd 运行的 Gateway 网关——请将其放入
      `~/.openclaw/.env`，或启用 `env.shellEnv`。
    - 确认正在编辑正确的智能体——多智能体设置中有
      多个 `auth-profiles.json` 文件。
    - 运行 `openclaw models status`，查看已配置的模型和提供商
      身份验证状态。

    **对于“No credentials found for profile anthropic”（没有电子邮件后缀）：**

    此次运行固定使用了 Gateway 网关找不到的 Anthropic 配置文件。

    - 使用 Claude CLI：在 Gateway 网关主机上运行 `openclaw models auth login --provider anthropic
      --method cli --set-default`。
    - 如果更倾向于使用 API key：请在 Gateway 网关主机上的
      `~/.openclaw/.env` 中放入 `ANTHROPIC_API_KEY`，然后清除任何强制使用缺失配置文件的固定顺序：

      ```bash
      openclaw models auth order clear --provider anthropic
      ```

    - 远程模式：身份验证配置文件位于 Gateway 网关计算机上，而不是你的
      笔记本电脑上——请确认是在该计算机上运行命令。

  </Accordion>

  <Accordion title="为什么它还尝试了 Google Gemini 并失败了？">
    如果模型配置将 Google Gemini 设为回退模型（或切换到了 Gemini 简写），OpenClaw
    会在回退期间尝试使用它。未配置 Google 凭据时会出现 `No API key found for provider
    "google"`。修复方法：添加 Google 身份验证，或从
    `agents.defaults.model.fallbacks`/别名中移除 Google 模型。

    **LLM 请求被拒绝：需要思考签名（Google Antigravity）**

    原因：会话历史中包含没有签名的思考块（通常源自中止或不完整的流）；
    Google Antigravity 要求思考块带有签名。OpenClaw 会为 Google
    Antigravity Claude 移除未签名的思考块；如果仍然出现此问题，请启动新会话，或为该智能体设置
    `/thinking off`。

  </Accordion>
</AccordionGroup>

## 身份验证配置文件：定义及管理方式

相关内容：[/concepts/oauth](/zh-CN/concepts/oauth)（OAuth 流程、令牌存储、多账户模式）

<AccordionGroup>
  <Accordion title="什么是身份验证配置文件？">
    与提供商关联的具名凭据记录（OAuth 或 API key），存储于：

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    在不输出机密的情况下检查已保存的配置文件：`openclaw models auth
    list`（可选使用 `--provider <id>` 或 `--json`）。请参阅
    [模型 CLI](/zh-CN/cli/models#auth-profiles)。

  </Accordion>

  <Accordion title="常见的配置文件 ID 有哪些？">
    以提供商为前缀：`anthropic:default`（没有电子邮件身份时常用）、
    用于 OAuth 身份的 `anthropic:<email>`，或你选择的自定义 ID
    （例如 `anthropic:work`）。

  </Accordion>

  <Accordion title="可以控制首先尝试哪个身份验证配置文件吗？">
    可以。`auth.order.<provider>` 配置用于设置每个提供商的轮换顺序
    （仅存储元数据，不存储机密）。

    OpenClaw 可能会跳过处于短暂**冷却**状态（速率限制、
    超时、身份验证失败）或较长时间**禁用**状态
    （计费/额度不足）的配置文件。使用 `openclaw models status
    --json` 检查，并查看 `auth.unusableProfiles`。速率限制冷却可以
    限定到模型——某个配置文件因一个模型而进入冷却期时，仍可为同一提供商的
    同级模型提供服务；计费/禁用窗口则会阻止整个配置文件。

    设置按智能体生效的顺序覆盖（存储在该智能体的 `auth-state.json` 中）：

    ```bash
    # 默认为已配置的默认智能体（省略 --agent）
    openclaw models auth order get --provider anthropic

    # 将轮换锁定为单个配置文件
    openclaw models auth order set --provider anthropic anthropic:default

    # 或设置明确的顺序（提供商内部回退）
    openclaw models auth order set --provider anthropic anthropic:work anthropic:default

    # 清除覆盖（回退到配置中的 auth.order / 轮询）
    openclaw models auth order clear --provider anthropic

    # 指定特定智能体
    openclaw models auth order set --provider anthropic --agent main anthropic:default
    ```

    验证实际将尝试的内容：`openclaw models status --probe`。显式顺序中遗漏的
    已存储配置文件会报告
    `excluded_by_auth_order`，而不会被静默尝试。

  </Accordion>

  <Accordion title="OAuth 和 API key 有什么区别？">
    - 在提供商支持的情况下，**OAuth / CLI 登录**通常使用订阅访问权限。
      对于 Anthropic，OpenClaw 的 Claude CLI 后端使用 Claude Code `claude -p`，Anthropic 目前将其视为
      使用订阅用量限制的 Agent SDK/编程式使用——
      有关当前暂停计费的状态和来源链接，请参阅 [Anthropic](/zh-CN/providers/anthropic)。
    - **API key** 采用按令牌计费。

    向导支持 Anthropic Claude CLI、OpenAI Codex OAuth 和 API
    key。

  </Accordion>
</AccordionGroup>

## 相关内容

- [常见问题](/zh-CN/help/faq)——主要常见问题
- [常见问题——快速开始和首次运行设置](/zh-CN/help/faq-first-run)
- [模型选择](/zh-CN/concepts/model-providers)
- [模型故障转移](/zh-CN/concepts/model-failover)
