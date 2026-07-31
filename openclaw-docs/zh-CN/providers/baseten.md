---
read_when:
    - 你想在 OpenClaw 中运行 Thinking Machines Lab 的 Inkling
    - 你希望通过一个兼容 OpenAI 的 API 使用 Baseten 托管的模型
summary: Baseten 的 Inkling 和托管 Model APIs 设置
title: Baseten
x-i18n:
    generated_at: "2026-07-26T06:59:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2ccc3b5cf64b01859f9f022d7bc15a69a1cb42c87d4f914c118276c1151020de
    source_path: providers/baseten.md
    workflow: 16
---

[Baseten Model APIs](https://docs.baseten.co/inference/model-apis/overview) 提供托管式、兼容 OpenAI 的前沿模型访问能力。官方外部插件使用经身份验证的模型发现，因此 OpenClaw 会采用为你的 Baseten 账户启用的完整模型集。其离线回退目录包含此 OpenClaw 版本构建时可用的所有 Model API。

| 属性            | 值                                                       |
| --------------- | -------------------------------------------------------- |
| 提供商 ID       | `baseten`                                       |
| 插件            | 官方外部软件包（`@openclaw/baseten-provider`）                     |
| 身份验证环境变量 | `BASETEN_API_KEY`                                       |
| 新手引导标志    | `--auth-choice baseten-api-key`                                       |
| 直接 CLI 标志   | `--baseten-api-key <key>`                                       |
| API             | 兼容 OpenAI（`openai-completions`）                        |
| 基础 URL        | `https://inference.baseten.co/v1`                                       |
| 默认模型        | `baseten/thinkingmachines/inkling`                                       |

## 安装插件

```bash
openclaw plugins install @openclaw/baseten-provider
openclaw gateway restart
```

## 入门指南

<Steps>
  <Step title="创建 Baseten 账户和 API 密钥">
    Baseten 的 Basic 方案不收取每月平台费用；Model API 调用按用量计费。在 [Baseten API 密钥设置](https://app.baseten.co/settings/api_keys)中创建密钥，并在[定价页面](https://www.baseten.co/pricing)查看当前费率。
  </Step>
  <Step title="运行新手引导">
    <CodeGroup>

```bash Onboarding
openclaw onboard --auth-choice baseten-api-key
```

```bash Direct flag
openclaw onboard --non-interactive \
  --auth-choice baseten-api-key \
  --baseten-api-key "$BASETEN_API_KEY"
```

```bash Env only
export BASETEN_API_KEY=...
```

    </CodeGroup>

  </Step>
  <Step title="验证实时目录">
    ```bash
    openclaw models list --provider baseten
    ```

    身份验证可用时，插件会请求 `GET /v1/models`，并列出为该账户返回的所有模型。没有身份验证时，插件会保持离线并使用内置回退目录。

  </Step>
</Steps>

## Inkling

[Thinking Machines Lab 的 Inkling](https://thinkingmachines.ai/news/introducing-inkling/) 是默认模型。在 OpenClaw 中，它支持文本和图像输入、工具调用、结构化工具架构、可配置的推理强度、1.048M token 的上下文窗口，以及最多 32k 输出 token：

```json5
{
  agents: {
    defaults: {
      model: { primary: "baseten/thinkingmachines/inkling" },
    },
  },
}
```

使用 `/model baseten/thinkingmachines/inkling` 切换现有聊天的模型。

## 内置回退目录

经身份验证的实时目录具有权威性。在模型发现成功前，以下条目可确保设置和模型选择功能仍然可用：

| 模型引用                                           | 输入        | 上下文 | 最大输出 |
| -------------------------------------------------- | ----------- | ------: | -------: |
| `baseten/deepseek-ai/DeepSeek-V4-Pro`              | 文本        |    262k |     262k |
| `baseten/zai-org/GLM-4.7`                          | 文本        |    200k |     200k |
| `baseten/zai-org/GLM-5`                            | 文本        |    202k |     202k |
| `baseten/zai-org/GLM-5.1`                          | 文本        |    202k |     202k |
| `baseten/zai-org/GLM-5.2`                          | 文本        |    202k |     202k |
| `baseten/thinkingmachines/inkling`                 | 文本、图像  |  1.048M |      32k |
| `baseten/moonshotai/Kimi-K2.5`                     | 文本、图像  |    262k |     262k |
| `baseten/moonshotai/Kimi-K2.6`                     | 文本、图像  |    262k |     262k |
| `baseten/moonshotai/Kimi-K2.7-Code`                | 文本、图像  |    262k |     262k |
| `baseten/nvidia/Nemotron-120B-A12B`                | 文本        |    202k |     202k |
| `baseten/nvidia/NVIDIA-Nemotron-3-Ultra-550B-A55B` | 文本        |    202k |     202k |
| `baseten/openai/gpt-oss-120b`                      | 文本        |    128k |     128k |

所有内置模型均支持工具调用和推理。OpenClaw 会将其思考级别映射到具有原生 `reasoning_effort` 的模型。Baseten 中可选择启用的 GLM、Kimi 和 Nemotron 模型默认关闭思考；大多数模型提供二元的关闭/开启控制，而 GLM 5.2 提供关闭、高和最大三个级别。OpenClaw 通过 Baseten 的 `chat_template_args.enable_thinking` 控制项发送这些选择；对于 GLM 5.2，还会发送经过验证的顶层 `reasoning_effort` 参数。

<Note>
Baseten 可以独立于 OpenClaw 版本添加、移除或更改 Model API。插件会通过经身份验证的 API 刷新模型 ID、上下文限制、输出限制，以及输入、缓存输入和输出定价，同时保留特定于模型的 OpenClaw 传输策略。
</Note>

## 手动配置

大多数设置只需要 API 密钥。若要显式固定提供商：

```json5
{
  env: { BASETEN_API_KEY: "..." },
  agents: {
    defaults: {
      model: { primary: "baseten/thinkingmachines/inkling" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      baseten: {
        baseUrl: "https://inference.baseten.co/v1",
        apiKey: "${BASETEN_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "thinkingmachines/inkling",
            name: "Inkling",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 1048000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<Note>
如果 Gateway 网关作为守护进程（launchd、systemd、Docker）运行，请确保 `BASETEN_API_KEY` 对该进程可用。仅在交互式 shell 中导出的密钥对已在运行的托管服务不可见。
</Note>

## 相关内容

<CardGroup cols={2}>
  <Card title="模型提供商" href="/zh-CN/concepts/model-providers" icon="layers">
    选择提供商、模型引用和故障转移行为。
  </Card>
  <Card title="思考模式" href="/zh-CN/tools/thinking" icon="brain">
    选择 OpenClaw 推理强度级别。
  </Card>
  <Card title="模型 CLI" href="/zh-CN/cli/models" icon="terminal">
    列出、检查和选择发现的模型。
  </Card>
  <Card title="模型常见问题" href="/zh-CN/help/faq-models" icon="circle-question">
    身份验证配置文件和模型选择故障排除。
  </Card>
</CardGroup>
