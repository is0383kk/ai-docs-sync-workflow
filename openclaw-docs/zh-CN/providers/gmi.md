---
read_when:
    - 你想使用 GMI Cloud 模型运行 OpenClaw
    - 你需要 GMI 提供商 ID、密钥或端点
summary: 通过 OpenClaw 使用 GMI Cloud 的 OpenAI 兼容 API
title: GMI Cloud
x-i18n:
    generated_at: "2026-07-26T06:56:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a21fd2a997f44e1f78d97a0fba24ca2bbc00dd193323da712d650ed4ba105355
    source_path: providers/gmi.md
    workflow: 16
---

GMI Cloud 是一个托管推理平台，通过兼容 OpenAI 的 API 提供前沿模型和开放权重模型。在 OpenClaw 中，它是官方外部提供商插件：安装一次，通过常规模型身份验证存储凭据，然后使用类似 `gmi/google/gemini-3.1-flash-lite` 的模型引用。

如果你希望使用一个 API key 访问多个托管模型系列，可以选择 GMI，其中包括 GMI 目录所提供的 Anthropic、DeepSeek、Google、Moonshot、OpenAI 和 Z.AI 路由。它可以作为模型回退的辅助提供商，用于比较不同厂商的托管路由，或在 GMI 比你的主要提供商更早提供某个模型时使用。OpenClaw 负责提供商 ID、身份验证配置文件、别名、模型目录种子和基础 URL；GMI 负责实时模型可用性、计费、速率限制以及任何提供商端路由策略。

| 属性          | 值                                       |
| ------------- | ---------------------------------------- |
| 提供商 ID     | `gmi`（别名：`gmi-cloud`、`gmicloud`） |
| 软件包        | `@openclaw/gmi-provider`                 |
| 身份验证环境变量 | `GMI_API_KEY`                            |
| API           | 兼容 OpenAI（`openai-completions`） |
| 基础 URL      | `https://api.gmi-serving.com/v1`         |
| 默认模型      | `gmi/google/gemini-3.1-flash-lite`       |

## 设置

安装插件，重启 Gateway 网关，然后在 GMI Cloud 中创建 API key
（`https://www.gmicloud.ai/`）：

```bash
openclaw plugins install @openclaw/gmi-provider
openclaw gateway restart
```

然后运行：

```bash
openclaw onboard --auth-choice gmi-api-key
```

非交互式设置可以传入 `--gmi-api-key <key>`，或设置：

```bash
export GMI_API_KEY="<your-gmi-api-key>" # pragma: allowlist secret
```

## 何时选择 GMI

- 你希望使用托管的兼容 OpenAI 的端点，而不是本地模型服务器。
- 你希望通过一个提供商账户试用多个商业模型系列和开放权重模型系列。
- 你希望使用一个上游路由不同于 DeepInfra、OpenRouter、Together 或厂商直连 API 的回退提供商。
- 你需要 GMI 特有的模型 ID、定价或账户控制功能。

如果你需要 GMI 未通过其兼容 OpenAI 的路由公开的厂商原生功能，请改用厂商直连提供商。如果数据本地性或本地 GPU 控制比托管服务的便利性更重要，请选择 LM Studio、Ollama、SGLang 或 vLLM 等本地提供商。

## 模型

插件目录预置了常见的 GMI Cloud 路由 ID：

| 模型引用                           | 输入          | 上下文    | 最大输出   |
| ---------------------------------- | ------------ | --------- | ---------- |
| `gmi/anthropic/claude-sonnet-4.6`  | 文本 + 图像 | 200,000   | 64,000     |
| `gmi/deepseek-ai/DeepSeek-V3.2`    | 文本         | 163,840   | 65,536     |
| `gmi/google/gemini-3.1-flash-lite` | 文本 + 图像 | 1,048,576 | 65,536     |
| `gmi/moonshotai/Kimi-K2.5`         | 文本 + 图像 | 262,144   | 65,536     |
| `gmi/openai/gpt-5.4`               | 文本 + 图像 | 400,000   | 128,000    |
| `gmi/zai-org/GLM-5.1-FP8`          | 文本         | 202,752   | 65,536     |

该目录只是种子数据，并不保证每个账户都能随时调用每个模型。列出已配置提供商在你的环境中报告的模型：

```bash
openclaw models list --provider gmi
```

## 故障排查

- `401` 或 `403`：检查运行 OpenClaw 的进程是否设置了 `GMI_API_KEY`，或重新运行新手引导，将 API key 存储在提供商身份验证配置文件中。
- 未知模型错误：确认该模型存在于你的 GMI 账户中，并使用 `openclaw models list --provider gmi` 显示的完整 `gmi/<route-id>` 引用。
- 间歇性提供商错误：尝试其他 GMI 路由，或将 GMI 配置为回退提供商，而不是唯一的主要模型提供商。

## 相关内容

- [模型提供商](/zh-CN/concepts/model-providers)
- [所有提供商](/zh-CN/providers/index)
