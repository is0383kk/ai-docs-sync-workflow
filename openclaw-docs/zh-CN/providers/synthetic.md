---
read_when:
    - 你想使用 Synthetic 作为模型提供商
    - 你需要设置 Synthetic API key 或基础 URL
summary: 在 OpenClaw 中使用 Synthetic 的 Anthropic 兼容 API
title: Synthetic
x-i18n:
    generated_at: "2026-07-26T06:31:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3f6cc89a7b837f57555d176ce78e62a39095d4ef0765c96b6b7b93ffebd7388
    source_path: providers/synthetic.md
    workflow: 16
---

[Synthetic](https://synthetic.new) 提供与 Anthropic 兼容的端点。
OpenClaw 将其内置为 `synthetic` 提供商，并使用 Anthropic
Messages API。

| 属性     | 值                                    |
| -------- | ------------------------------------- |
| 提供商   | `synthetic`                    |
| 身份验证 | `SYNTHETIC_API_KEY`                    |
| API      | Anthropic Messages                    |
| 基础 URL | `https://api.synthetic.new/anthropic`                    |

## 入门指南

<Steps>
  <Step title="获取 API key">
    从你的 Synthetic 账户获取 `SYNTHETIC_API_KEY`，或让新手引导
    提示你输入。
  </Step>
  <Step title="运行新手引导">
    ```bash
    openclaw onboard --auth-choice synthetic-api-key
    ```
  </Step>
  <Step title="验证默认模型">
    新手引导会将默认模型设置为：
    ```text
    synthetic/hf:MiniMaxAI/MiniMax-M3
    ```
  </Step>
</Steps>

<Warning>
OpenClaw 的 Anthropic 客户端会自动将 `/v1` 追加到基础 URL，因此请使用
`https://api.synthetic.new/anthropic`（而不是 `/anthropic/v1`）。如果 Synthetic
更改了基础 URL，请覆盖 `models.providers.synthetic.baseUrl`。
</Warning>

## 配置示例

```json5
{
  env: { SYNTHETIC_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
      models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "hf:MiniMaxAI/MiniMax-M3",
            name: "MiniMax M3",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

## 内置目录

所有 Synthetic 模型的费用均为 `0`（输入/输出/缓存）。有关服务可用性，请参阅 Synthetic 的
[当前模型列表](https://dev.synthetic.new/docs/api/models)。

| 模型 ID                                             | 上下文窗口 | 最大 token 数 | 推理 | 输入        |
| --------------------------------------------------- | ---------- | ------------ | ---- | ----------- |
| `hf:MiniMaxAI/MiniMax-M3`                                  | 262,144    | 65,536       | 是   | 文本 + 图像 |
| `hf:moonshotai/Kimi-K2.7-Code`                                  | 262,144    | 8,192        | 是   | 文本 + 图像 |
| `hf:nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4`                                  | 262,144    | 8,192        | 是   | 文本        |
| `hf:openai/gpt-oss-120b`                                  | 131,072    | 8,192        | 是   | 文本        |
| `hf:Qwen/Qwen3.6-27B`                                  | 262,144    | 81,920       | 是   | 文本 + 图像 |
| `hf:zai-org/GLM-4.7-Flash`                                  | 196,608    | 131,072      | 是   | 文本        |
| `hf:zai-org/GLM-5.2`                                  | 524,288    | 131,072      | 是   | 文本        |

<Tip>
模型引用采用 `synthetic/<modelId>` 格式。使用
`openclaw models list --provider synthetic` 可查看你的
账户可用的所有模型。
</Tip>

<AccordionGroup>
  <Accordion title="模型允许列表">
    如果启用模型允许列表（`agents.defaults.modelPolicy.allow`），请添加你计划使用的每个
    Synthetic 模型。不在允许列表中的模型会对智能体隐藏。
  </Accordion>

  <Accordion title="覆盖基础 URL">
    如果 Synthetic 更改了其 API 端点，请覆盖基础 URL：

    ```json5
    {
      models: {
        providers: {
          synthetic: {
            baseUrl: "https://new-api.synthetic.new/anthropic",
          },
        },
      },
    }
    ```

    OpenClaw 仍会自动追加 `/v1`。

  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="模型提供商" href="/zh-CN/concepts/model-providers" icon="layers">
    提供商规则、模型引用和故障转移行为。
  </Card>
  <Card title="配置参考" href="/zh-CN/gateway/configuration-reference" icon="gear">
    完整的配置架构，包括提供商设置。
  </Card>
  <Card title="Synthetic" href="https://synthetic.new" icon="arrow-up-right-from-square">
    Synthetic 控制面板和 API 文档。
  </Card>
</CardGroup>
