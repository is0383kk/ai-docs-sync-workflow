---
read_when:
    - 你想在 OpenClaw 中使用 Chutes
    - 你需要 OAuth 或 API key 设置路径
    - 你需要默认模型、别名或发现行为
summary: Chutes 设置（OAuth 或 API key、模型发现、别名）
title: Chutes
x-i18n:
    generated_at: "2026-07-26T06:25:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 57ea5112105f19028c1a348b4d7fec4cf7ef12de00b1b2de9c152057bf5033a9
    source_path: providers/chutes.md
    workflow: 16
---

[Chutes](https://chutes.ai) 通过 OpenAI 兼容 API 提供开源模型目录。OpenClaw 同时支持浏览器 OAuth 和 API key 身份验证。

| 属性             | 值                                                      |
| ---------------- | ------------------------------------------------------- |
| 提供商           | `chutes`                                      |
| 插件             | 官方外部软件包（`@openclaw/chutes-provider`）                    |
| API              | OpenAI 兼容                                             |
| 基础 URL         | `https://llm.chutes.ai/v1`                                      |
| 身份验证         | OAuth 或 API key（见下文）                              |
| 运行时环境变量   | `CHUTES_API_KEY`、`CHUTES_OAUTH_TOKEN`                  |

`CHUTES_OAUTH_TOKEN` 可直接提供已获取的 OAuth 访问令牌（例如在 CI 中），从而绕过下方的交互式浏览器流程。

## 安装插件

```bash
openclaw plugins install @openclaw/chutes-provider
openclaw gateway restart
```

## 入门指南

两种方式都会将默认模型设置为 `chutes/zai-org/GLM-5-TEE`，并注册 Chutes 目录。

<Tabs>
  <Tab title="OAuth">
    <Steps>
      <Step title="运行 OAuth 新手引导流程">
        ```bash
        openclaw onboard --auth-choice chutes
        ```
        OpenClaw 会在本地启动浏览器流程；在远程或无头主机上，则会显示 URL 和粘贴重定向结果的流程。OAuth 令牌会通过 OpenClaw 身份验证配置文件自动刷新。
      </Step>
    </Steps>
  </Tab>
  <Tab title="API key">
    <Steps>
      <Step title="获取 API key">
        在
        [chutes.ai/settings/api-keys](https://chutes.ai/settings/api-keys)
        创建密钥。
      </Step>
      <Step title="运行 API key 新手引导流程">
        ```bash
        openclaw onboard --auth-choice chutes-api-key
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

## 设备发现行为

当 Chutes 身份验证可用时，OpenClaw 会使用该凭据查询 `GET /v1/models`，并使用发现的模型；每个凭据的结果缓存 5 分钟。若密钥已过期或未经授权（HTTP 401），OpenClaw 会在不使用凭据的情况下重试一次。如果设备发现仍未返回任何行、失败或返回任何其他非 2xx 状态，则会回退到内置静态目录（API key 和 OAuth 设备发现使用相同路径）。如果启动时设备发现失败，则自动使用静态目录。

## 默认别名

OpenClaw 为 Chutes 目录注册了两个便捷别名：

| 别名                 | 目标模型                               |
| -------------------- | -------------------------------------- |
| `chutes-pro`   | `chutes/deepseek-ai/DeepSeek-V3.2-TEE`                     |
| `chutes-vision`   | `chutes/moonshotai/Kimi-K2.5-TEE`                     |

## 内置入门目录

内置回退目录包含以下五个当前提供服务的模型：

| 模型引用                               |
| -------------------------------------- |
| `chutes/zai-org/GLM-5-TEE`                     |
| `chutes/deepseek-ai/DeepSeek-V3.2-TEE`                     |
| `chutes/moonshotai/Kimi-K2.5-TEE`                     |
| `chutes/MiniMaxAI/MiniMax-M2.5-TEE`                     |
| `chutes/Qwen/Qwen3.5-397B-A17B-TEE`                     |

运行 `openclaw models list --all --provider chutes` 查看完整列表。

## 配置示例

```json5
{
  agents: {
    defaults: {
      model: { primary: "chutes/zai-org/GLM-5-TEE" },
      models: {
        "chutes/zai-org/GLM-5-TEE": { alias: "Chutes GLM 5" },
        "chutes/deepseek-ai/DeepSeek-V3.2-TEE": { alias: "Chutes DeepSeek V3.2" },
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="OAuth 覆盖设置">
    使用可选环境变量自定义 OAuth 流程：

    | 变量 | 用途 |
    | ---- | ---- |
    | `CHUTES_CLIENT_ID` | OAuth 客户端 ID（未设置时提示输入） |
    | `CHUTES_CLIENT_SECRET` | OAuth 客户端密钥 |
    | `CHUTES_OAUTH_REDIRECT_URI` | 重定向 URI（默认值为 `http://127.0.0.1:1456/oauth-callback`） |
    | `CHUTES_OAUTH_SCOPES` | 以空格分隔的作用域（默认值为 `openid profile chutes:invoke`） |

    有关重定向应用要求和帮助，请参阅 [Chutes OAuth 文档](https://chutes.ai/docs/sign-in-with-chutes/overview)。

  </Accordion>

  <Accordion title="注意事项">
    - Chutes 模型注册为 `chutes/<model-id>`。
    - Chutes 在流式传输期间不会报告令牌用量（`supportsUsageInStreaming: false`）；流式传输完成后仍会显示用量总计。

  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="模型选择" href="/zh-CN/concepts/model-providers" icon="layers">
    提供商规则、模型引用和故障转移行为。
  </Card>
  <Card title="配置参考" href="/zh-CN/gateway/configuration-reference" icon="gear">
    完整配置架构，包括提供商设置。
  </Card>
  <Card title="Chutes" href="https://chutes.ai" icon="arrow-up-right-from-square">
    Chutes 控制面板和 API 文档。
  </Card>
  <Card title="Chutes API key" href="https://chutes.ai/settings/api-keys" icon="key">
    创建和管理 Chutes API key。
  </Card>
</CardGroup>
