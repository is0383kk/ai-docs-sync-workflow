---
read_when:
    - 你希望使用一个 API 密钥访问多个 LLM
    - 你想在 OpenClaw 中通过 Kilo Gateway 运行模型
summary: 使用 Kilo Gateway 网关的统一 API 在 OpenClaw 中访问多种模型
title: Kilo Gateway 网关
x-i18n:
    generated_at: "2026-07-26T06:58:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0246a1a77f4265168b213e0167360e1cd89dc2ca864997f08cae5331037f9e89
    source_path: providers/kilocode.md
    workflow: 16
---

Kilo Gateway 网关通过单个兼容 OpenAI 的端点和 API key，将请求路由到多个模型。

| 属性 | 值                                 |
| -------- | ---------------------------------- |
| 提供商 | `kilocode`                         |
| 身份验证     | `KILOCODE_API_KEY`                 |
| API      | 兼容 OpenAI                  |
| 基础 URL | `https://api.kilo.ai/api/gateway/` |

## 安装插件

```bash
openclaw plugins install @openclaw/kilocode-provider
openclaw gateway restart
```

## 设置

<Steps>
  <Step title="创建账户">
    前往 [app.kilo.ai](https://app.kilo.ai)，登录或创建账户，然后生成 API key。
  </Step>
  <Step title="运行新手引导">
    ```bash
    openclaw onboard --auth-choice kilocode-api-key
    ```

    或直接设置环境变量：

    ```bash
    export KILOCODE_API_KEY="<your-kilocode-api-key>" # pragma: allowlist secret
    ```

  </Step>
  <Step title="验证模型是否可用">
    ```bash
    openclaw models list --provider kilocode
    ```
  </Step>
</Steps>

## 默认模型和目录

默认模型为 `kilocode/kilo-auto/balanced`，即 Kilo Gateway 网关的均衡智能路由层级。
OpenClaw 不会发布其任务到上游模型的映射关系；`kilo-auto/balanced`
背后的路由由 Kilo Gateway 网关负责。

OpenClaw 启动时会查询 `GET https://api.kilo.ai/api/gateway/models`，并将发现的模型合并到
静态回退目录之前。静态回退目录仅包含
`kilocode/kilo-auto/balanced`（`Auto Balanced`、`input: ["text", "image"]`、`reasoning: true`、
`contextWindow: 1000000`、`maxTokens: 65536`）。

Gateway 网关上的任何模型都可通过 `kilocode/<upstream-id>` 寻址（例如
`kilocode/anthropic/claude-sonnet-4`、`kilocode/openai/gpt-5.5`）。运行 `/models kilocode` 或
`openclaw models list --provider kilocode` 可查看发现的完整列表。

## 配置示例

```json5
{
  env: { KILOCODE_API_KEY: "<your-kilocode-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "kilocode/kilo-auto/balanced" },
    },
  },
}
```

## 行为说明

<AccordionGroup>
  <Accordion title="传输和兼容性">
    Kilo Gateway 网关兼容 OpenRouter，因此它使用代理式的 OpenAI 兼容请求
    路径，而不是原生 OpenAI 请求结构（无 `store`，无 OpenAI 推理强度载荷）。

    - 由 Gemini 支持的 Kilo 引用仍使用代理 Gemini 路径：OpenClaw 会在那里清理 Gemini 思考
      签名，但不会启用原生 Gemini 重放验证或引导重写。
    - 请求使用根据你的 API key 构建的 Bearer token。

  </Accordion>

  <Accordion title="流包装器和推理">
    Kilo 流包装器会添加 `X-KILOCODE-FEATURE` 请求标头（默认为 `openclaw`，
    可通过 `KILOCODE_FEATURE` 环境变量覆盖），并为支持的模型规范化推理强度载荷。

    <Warning>
    `kilocode/kilo-auto/balanced` 和 `x-ai/*` 引用会跳过推理强度注入。如果需要推理支持，
    请使用具体的模型引用，例如 `kilocode/anthropic/claude-sonnet-4`。
    </Warning>

  </Accordion>

  <Accordion title="故障排查">
    - 如果启动时模型发现失败，OpenClaw 会回退到包含 `kilocode/kilo-auto/balanced` 的静态目录。
    - 确认你的 API key 有效，并且你的 Kilo 账户已启用所需模型。
    - 当 Gateway 网关作为守护进程运行时，请确保该进程可以使用 `KILOCODE_API_KEY`（例如位于 `~/.openclaw/.env` 中，或通过 `env.shellEnv` 提供）。

  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="模型选择" href="/zh-CN/concepts/model-providers" icon="layers">
    选择提供商、模型引用和故障转移行为。
  </Card>
  <Card title="配置参考" href="/zh-CN/gateway/configuration-reference" icon="gear">
    完整的 OpenClaw 配置参考。
  </Card>
  <Card title="Kilo Gateway 网关" href="https://app.kilo.ai" icon="arrow-up-right-from-square">
    Kilo Gateway 网关仪表板、API key 和账户管理。
  </Card>
</CardGroup>
