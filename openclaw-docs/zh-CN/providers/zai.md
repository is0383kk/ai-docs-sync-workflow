---
read_when:
    - 你想在 OpenClaw 中使用 Z.AI / GLM 模型
    - 你需要简单设置一下 ZAI_API_KEY
summary: 在 OpenClaw 中使用 Z.AI (GLM) 模型
title: Z.AI
x-i18n:
    generated_at: "2026-07-26T06:59:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ca3e7ef743e908550f4d96ba6f78167e38cabd15b14044683b02493ebbf3025
    source_path: providers/zai.md
    workflow: 16
---

Z.AI 是 **GLM** 模型的 API 平台。它为 GLM 提供 REST API，并使用 API key 进行身份验证。请在 Z.AI 控制台中创建 API key。
OpenClaw 将 Z.AI API key 与 `zai` 提供商配合使用。

| 属性 | 值                                           |
| ---- | -------------------------------------------- |
| 提供商 | `zai`                                        |
| 软件包 | `@openclaw/zai-provider`                     |
| 身份验证 | `ZAI_API_KEY`（旧版别名：`Z_AI_API_KEY`） |
| API  | Z.AI Chat Completions（Bearer 身份验证）     |

## GLM 模型

GLM 是一个模型系列，而非独立的提供商。在 OpenClaw 中，GLM 模型使用
`zai/glm-5.2` 之类的引用：提供商为 `zai`，模型 ID 为 `glm-5.2`。

## 入门指南

请先安装提供商插件：

```bash
openclaw plugins install @openclaw/zai-provider
```

<Tabs>
  <Tab title="自动检测端点">
    **最适合：**大多数用户。OpenClaw 会使用你的 API key 探测受支持的 Z.AI 端点，并自动应用正确的基础 URL。

    <Steps>
      <Step title="运行新手引导">
        ```bash
        openclaw onboard --auth-choice zai-api-key
        ```
      </Step>
      <Step title="验证模型已列出">
        ```bash
        openclaw models list --all --provider zai
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="显式指定区域端点">
    **最适合：**希望强制使用特定 Coding Plan 或通用 API 接口的用户。

    <Steps>
      <Step title="选择正确的新手引导选项">
        ```bash
        # Coding Plan 全球端点（建议 Coding Plan 用户使用）
        openclaw onboard --auth-choice zai-coding-global

        # Coding Plan 中国区端点
        openclaw onboard --auth-choice zai-coding-cn

        # 通用 API
        openclaw onboard --auth-choice zai-global

        # 通用 API 中国区端点
        openclaw onboard --auth-choice zai-cn
        ```
      </Step>
      <Step title="验证模型已列出">
        ```bash
        openclaw models list --all --provider zai
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

### 端点

| 新手引导选项        | 基础 URL                                      | 默认模型      |
| ------------------- | --------------------------------------------- | ------------- |
| `zai-global`        | `https://api.z.ai/api/paas/v4`                | `glm-5.1`     |
| `zai-cn`            | `https://open.bigmodel.cn/api/paas/v4`        | `glm-5.1`     |
| `zai-coding-global` | `https://api.z.ai/api/coding/paas/v4`         | `glm-5.2`     |
| `zai-coding-cn`     | `https://open.bigmodel.cn/api/coding/paas/v4` | `glm-5.2`     |

Z.AI 还发布了与 Anthropic 兼容的 Coding Plan 基础 URL：
`https://api.z.ai/api/anthropic`。OpenClaw 的 Z.AI 选项使用上面列出的
OpenAI Chat Completions 端点；Anthropic URL 适用于直接使用 Anthropic Messages
协议的客户端。

`zai-api-key` 会通过使用你的 key 逐一探测这四个端点的
Chat Completions API 来自动检测其中之一。它会先检查通用端点（`zai-global`，
然后是 `zai-cn`），再检查 Coding Plan 端点（`zai-coding-global`，然后是
`zai-coding-cn`），并在找到第一个接受请求的端点时停止。如果你的 key
在两类端点上都能使用，请使用显式的 `--auth-choice` 强制指定 Coding Plan 端点。

## 速率限制和过载

Z.AI 文档将 Coding Plan 和通用智能体工具描述为实行容量管理的服务。根据 Z.AI 自己的文档：

- [通用智能体工具](https://docs.z.ai/devpack/tool/others)
  （包括 OpenClaw）以尽力而为的方式提供服务。在推理负载较高期间（通常为新加坡时间下午 2 点至 6 点），
  某些请求可能会受到临时速率限制。
- [Coding Plan 速率和并发限制](https://docs.z.ai/devpack/usage-policy)
  与套餐等级相关，并可根据资源可用性动态调整。非高峰时段的并发量可能更高。
- [API 错误代码 `1302`](https://docs.z.ai/api-reference/api-code) 表示“请求已达到
  速率限制”。API 错误代码 `1305` 表示“服务可能暂时过载，请稍后重试”。

如果在繁忙时段看到临时的 `429` 或 `1305` 响应，请等待后重试请求。
如果故障在非高峰时段仍可重复出现，或仅发生在某个端点、模型或请求结构上，
请先检查所配置的端点和模型：

```bash
openclaw models list --all --provider zai
openclaw config get models.providers.zai.baseUrl
```

Coding Plan key 应使用 `https://api.z.ai/api/coding/paas/v4` 等 Coding Plan 端点；
通用 API key 应使用 `https://api.z.ai/api/paas/v4` 等通用 API 端点。相同 key
和端点持续发生故障，可能表明请求被提供商拒绝或受到套餐限制，
而非普通的高峰负载限流。

## 配置示例

<Tip>
`zai-api-key` 可让 OpenClaw 根据 key 检测匹配的 Z.AI 端点，并自动应用正确的基础 URL。
如果希望强制使用特定 Coding Plan 或通用 API 接口，请使用显式的区域选项。
</Tip>

```json5
{
  env: { ZAI_API_KEY: "sk-..." },
  models: {
    providers: {
      zai: {
        // GLM-5.2 使用 Coding Plan 端点。
        baseUrl: "https://api.z.ai/api/coding/paas/v4",
      },
    },
  },
  agents: { defaults: { model: { primary: "zai/glm-5.2" } } },
}
```

## 内置目录

`zai` 提供商插件在插件清单中附带目录，因此只读列表可以在不加载提供商运行时的情况下显示已知的 GLM 条目：

```bash
openclaw models list --all --provider zai
```

基于清单的目录当前包括：

| 模型引用             | 说明                            |
| -------------------- | ------------------------------- |
| `zai/glm-5.2`        | Coding Plan 默认模型；1M 上下文 |
| `zai/glm-5.1`        | 通用 API 默认模型               |
| `zai/glm-5`          |                                 |
| `zai/glm-5-turbo`    |                                 |
| `zai/glm-5v-turbo`   |                                 |
| `zai/glm-4.7`        |                                 |
| `zai/glm-4.7-flash`  |                                 |
| `zai/glm-4.7-flashx` |                                 |
| `zai/glm-4.6`        |                                 |
| `zai/glm-4.6v`       |                                 |
| `zai/glm-4.5`        |                                 |
| `zai/glm-4.5-air`    |                                 |
| `zai/glm-4.5-flash`  |                                 |
| `zai/glm-4.5v`       |                                 |

目录中的 token 成本元数据遵循 Z.AI 当前的
[按量付费定价](https://docs.z.ai/guides/overview/pricing)。Coding Plan
订阅使用套餐配额，而非按 token 计费；有关套餐定价和可用性，请参阅实时
[订阅页面](https://z.ai/subscribe)。

<Tip>
GLM 模型以 `zai/<model>` 的形式提供（示例：`zai/glm-5`）。
</Tip>

<Note>
Coding Plan 设置默认为 `zai/glm-5.2`；通用 API 设置则保留
`zai/glm-5.1`。在 Coding Plan 端点上，当 key 或套餐未提供 GLM-5.2 时，
自动检测会依次回退到 `glm-5.1` 和 `glm-4.7`。GLM
版本和可用性可能会发生变化；运行 `openclaw models list --all --provider zai`
可查看已安装版本所知的目录。
</Note>

## 思考级别

<Tabs>
  <Tab title="GLM-5.2">
    完整范围：`off`、`low`、`high`、`max`（默认为 `off`）。OpenClaw 通过请求载荷中的
    `reasoning_effort`，将 `low` 和 `high` 映射到 Z.AI 的 `high` 推理强度，并将 `max` 映射到 Z.AI 的
    `max` 强度。
  </Tab>
  <Tab title="其他 GLM 模型">
    仅支持二元切换：`off` 和 `low`（在选择器中显示为 `on`），默认为
    `off`。将思考级别设置为 `off` 会发送 `thinking: { type: "disabled" }`；
    其他任何级别都不会修改请求载荷（应用 Z.AI 自身的默认推理行为）。
  </Tab>
</Tabs>

将思考级别设置为 `off`，可避免响应在显示可见文本之前将输出预算消耗在
`reasoning_content` 上。

## 高级配置

<AccordionGroup>
  <Accordion title="前向解析未知的 GLM-5 模型">
    当 ID 符合当前 GLM-5 系列的格式时，未知的 `glm-5*` ID 仍会在提供商路径上进行前向解析，
    即根据 `glm-4.7` 模板合成由提供商拥有的元数据。
  </Accordion>

  <Accordion title="工具调用流式传输">
    Z.AI 的工具调用流式传输默认启用 `tool_stream`。要将其禁用：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "zai/<model>": {
              params: { tool_stream: false },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="保留思考内容">
    保留思考内容需要主动启用，因为 Z.AI 要求重放完整的历史
    `reasoning_content`，这会增加提示词 token 数量。可按模型启用：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "zai/glm-5.2": {
              params: { preserveThinking: true },
            },
          },
        },
      },
    }
    ```

    启用且思考功能开启时，OpenClaw 会发送
    `thinking: { type: "enabled", clear_thinking: false }`，并为同一个 OpenAI 兼容对话记录重放先前的
    `reasoning_content`。snake_case 形式的 `preserve_thinking` 参数键也可用作别名。

    高级用户仍可使用 `params.extra_body.thinking` 覆盖确切的提供商载荷。

  </Accordion>

  <Accordion title="图像理解">
    Z.AI 插件会注册图像理解功能。

    | 属性          | 值          |
    | ------------- | ----------- |
    | 模型          | `glm-4.6v`  |

    图像理解功能会根据所配置的 Z.AI 身份验证自动解析，无需额外配置。

  </Accordion>

  <Accordion title="身份验证详情">
    - Z.AI 使用你的 API key 进行 Bearer 身份验证。
    - `zai-api-key` 新手引导选项会使用你的 key 探测受支持的端点，以自动检测匹配的 Z.AI 端点。
    - 如果希望强制使用特定 API 接口，请使用显式的区域选项（`zai-coding-global`、`zai-coding-cn`、`zai-global`、`zai-cn`）。
    - 旧版环境变量 `Z_AI_API_KEY` 仍受支持；如果未设置 `ZAI_API_KEY`，OpenClaw 会在启动时将其复制到 `ZAI_API_KEY`。

  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="模型选择" href="/zh-CN/concepts/model-providers" icon="layers">
    选择提供商、模型引用和故障转移行为。
  </Card>
  <Card title="配置参考" href="/zh-CN/gateway/configuration-reference" icon="gear">
    完整的 OpenClaw 配置架构，包括提供商和模型设置。
  </Card>
</CardGroup>
