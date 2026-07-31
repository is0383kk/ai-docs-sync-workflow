---
read_when:
    - 你想将 Arcee AI 与 OpenClaw 搭配使用
    - 你需要 API key 环境变量或 CLI 身份验证选项
summary: Arcee AI 设置（身份验证 + 模型选择）
title: Arcee AI
x-i18n:
    generated_at: "2026-07-26T06:55:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a4c2fc7b8d86dd0d2a300dfc48951657cbcfcd9250016f52c1804777b2966e11
    source_path: providers/arcee.md
    workflow: 16
---

[Arcee AI](https://arcee.ai) 通过兼容 OpenAI 的 API 提供 Trinity 混合专家模型系列。所有 Trinity 模型均采用 Apache 2.0 许可证。Arcee 是 OpenClaw 官方插件，未内置于核心，因此需要在新手引导前完成安装。

可以直接通过 Arcee 平台或通过 [OpenRouter](/zh-CN/providers/openrouter) 访问 Arcee 模型。

| 属性 | 值                                                                                 |
| -------- | ------------------------------------------------------------------------------------- |
| 提供商 | `arcee`                                                                               |
| 身份验证     | `ARCEEAI_API_KEY`（直接访问）或 `OPENROUTER_API_KEY`（通过 OpenRouter）                   |
| API      | 兼容 OpenAI                                                                     |
| 基础 URL | `https://api.arcee.ai/api/v1`（直接访问）或 `https://openrouter.ai/api/v1`（OpenRouter） |

## 安装插件

```bash
openclaw plugins install @openclaw/arcee-provider
openclaw gateway restart
```

## 入门指南

<Tabs>
  <Tab title="直接访问（Arcee 平台）">
    <Steps>
      <Step title="获取 API key">
        在 [Arcee AI](https://chat.arcee.ai/) 创建 API key。
      </Step>
      <Step title="运行新手引导">
        ```bash
        openclaw onboard --auth-choice arceeai-api-key
        ```
      </Step>
      <Step title="设置默认模型">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```
      </Step>
    </Steps>
  </Tab>

  <Tab title="通过 OpenRouter">
    <Steps>
      <Step title="获取 API key">
        在 [OpenRouter](https://openrouter.ai/keys) 创建 API key。
      </Step>
      <Step title="运行新手引导">
        ```bash
        openclaw onboard --auth-choice arceeai-openrouter
        ```
      </Step>
      <Step title="设置默认模型">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```

        直接访问和 OpenRouter 设置使用相同的模型引用。
      </Step>
    </Steps>

  </Tab>
</Tabs>

## 非交互式设置

<Tabs>
  <Tab title="直接访问（Arcee 平台）">
    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-api-key \
      --arceeai-api-key "$ARCEEAI_API_KEY"
    ```
  </Tab>

  <Tab title="通过 OpenRouter">
    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-openrouter \
      --openrouter-api-key "$OPENROUTER_API_KEY"
    ```
  </Tab>
</Tabs>

## Arcee 直接访问目录

| 模型引用                      | 名称                   | 输入 | 上下文 | 最大输出 | 成本（每 100 万输入/输出 token） | 工具 | 说明                                     |
| ------------------------------ | ---------------------- | ----- | ------- | ---------- | -------------------- | ----- | ----------------------------------------- |
| `arcee/trinity-large-thinking` | Trinity Large Thinking | 文本  | 256K    | 80K        | $0.25 / $0.90        | 否    | 默认模型；扩展思考          |
| `arcee/trinity-large-preview`  | Trinity Large Preview  | 文本  | 128K    | 16K        | $0.25 / $1.00        | 是   | 通用；400B 参数，13B 激活参数  |
| `arcee/trinity-mini`           | Trinity Mini 26B       | 文本  | 128K    | 80K        | $0.045 / $0.15       | 是   | 快速且经济高效；函数调用 |

<Tip>
新手引导预设将 `arcee/trinity-large-thinking` 设置为默认模型。
</Tip>

## OpenRouter 目录

OpenRouter 新手引导提供 `arcee/trinity-large-preview` 和 `arcee/trinity-large-thinking`。OpenClaw 在配置中保留这些包含提供商限定信息的模型引用，并发送 OpenRouter 的规范 `arcee-ai/*` 运行时 ID。OpenRouter 不再提供 Trinity Mini；如需使用该模型，请使用 Arcee 直接 API。

## 支持的功能

| 功能                                       | 支持情况                                    |
| --------------------------------------------- | -------------------------------------------- |
| 流式传输                                     | 是                                          |
| 工具使用 / 函数调用                   | 是（Trinity Mini、Trinity Large Preview）    |
| 结构化输出（JSON 模式和 JSON schema） | 是                                          |
| 扩展思考                             | 是（Trinity Large Thinking；工具已禁用） |

<AccordionGroup>
  <Accordion title="环境说明">
    如果 Gateway 网关作为守护进程（launchd/systemd）运行，请确保 `ARCEEAI_API_KEY`
    （或 `OPENROUTER_API_KEY`）可供该进程使用，例如将其放在
    `~/.openclaw/.env` 中或通过 `env.shellEnv` 提供。
  </Accordion>

  <Accordion title="OpenRouter 路由">
    OpenRouter 使用相同的 `arcee/trinity-large-thinking` OpenClaw 模型引用。
    OpenClaw 使用规范的 `arcee-ai/trinity-large-thinking`
    OpenRouter 运行时 ID 进行路由。有关 OpenRouter 特有的
    配置详情，请参阅 [OpenRouter 提供商文档](/zh-CN/providers/openrouter)。
  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="OpenRouter" href="/zh-CN/providers/openrouter" icon="shuffle">
    使用一个 API key 访问 Arcee 模型及许多其他模型。
  </Card>
  <Card title="模型选择" href="/zh-CN/concepts/model-providers" icon="layers">
    选择提供商、模型引用和故障转移行为。
  </Card>
</CardGroup>
