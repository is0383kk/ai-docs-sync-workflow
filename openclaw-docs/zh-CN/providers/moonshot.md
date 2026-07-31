---
read_when:
    - 你想要配置 Moonshot Kimi K3/K2（Moonshot 开放平台）还是 Kimi Coding
    - 你需要了解各自独立的端点、密钥和模型引用
    - 你需要任一提供商的可复制粘贴配置
summary: 配置 Moonshot Kimi 模型与 Kimi Coding（使用独立的提供商和密钥）
title: Moonshot AI
x-i18n:
    generated_at: "2026-07-26T07:00:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 213379bf88fec26b052184a920e112f0887d6485601bfb47f590cf37ef983e58
    source_path: providers/moonshot.md
    workflow: 16
---

Moonshot 提供具有 OpenAI 兼容端点的 Kimi API。选择
`moonshot/kimi-k3` 以使用 Kimi K3，保留新手引导默认值
`moonshot/kimi-k2.6`，或使用 `kimi/kimi-for-coding` 以使用 Kimi Coding。

<Warning>
Moonshot 和 Kimi Coding 是**独立的提供商**，分别作为独立的外部插件提供。两者的密钥不能互换，端点不同，模型引用也不同（`moonshot/...` 与 `kimi/...`）。
</Warning>

## 内置模型目录

[//]: # "moonshot-kimi-k2-ids:start"

| 模型引用                            | 名称                     | 推理       | 输入         | 上下文    | 最大输出   |
| ----------------------------------- | ------------------------ | ---------- | ------------ | --------- | ---------- |
| `moonshot/kimi-k2.6`                | Kimi K2.6                | 否         | 文本、图像   | 262,144   | 262,144    |
| `moonshot/kimi-k3`                  | Kimi K3                  | 始终为最大 | 文本、图像   | 1,048,576 | 1,048,576  |
| `moonshot/kimi-k2.7-code`           | Kimi K2.7 Code           | 始终开启   | 文本、图像   | 262,144   | 262,144    |
| `moonshot/kimi-k2.7-code-highspeed` | Kimi K2.7 Code HighSpeed | 始终开启   | 文本、图像   | 262,144   | 262,144    |
| `moonshot/kimi-k2.5`                | Kimi K2.5                | 否         | 文本、图像   | 262,144   | 262,144    |

[//]: # "moonshot-kimi-k2-ids:end"

目录中的成本估算采用 Moonshot 公布的按量付费费率。在做出成本决策前，请查看供应商的实时页面：[Kimi K3](https://platform.kimi.ai/docs/pricing/chat-k3)、
[Kimi K2.7 Code](https://platform.kimi.ai/docs/pricing/chat-k27-code)、
[Kimi K2.6](https://platform.kimi.ai/docs/pricing/chat-k26) 和
[Kimi K2.5](https://platform.kimi.ai/docs/pricing/chat-k25)。

Kimi K3 始终以 `reasoning_effort: "max"` 进行推理。OpenClaw 仅公开
`/think max`，省略仅适用于 K2 的 `thinking` 字段，并移除 K3 固定为提供商默认值的采样覆盖项（`temperature`、`top_p`、`n`、`presence_penalty` 和
`frequency_penalty`）。Kimi K2.7 Code 也始终使用原生思考，但要求同时省略 `thinking` 和
`reasoning_effort`；HighSpeed 变体使用相同的约定。
Kimi K2.6 仍是新手引导默认值。
请参阅 Moonshot 的 [Kimi K3 快速开始](https://platform.kimi.ai/docs/guide/kimi-k3-quickstart)。

## 入门指南

Moonshot 和 Kimi Coding 都是外部插件——请先安装其中一个，再进行新手引导。

<Tabs>
  <Tab title="Moonshot API">
    **最适合：**通过 Moonshot 开放平台使用 Kimi K3 和 K2 模型。

    <Steps>
      <Step title="安装插件">
        ```bash
        openclaw plugins install @openclaw/moonshot-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="选择端点区域">
        | 身份验证选项           | 端点                           | 区域          |
        | ---------------------- | ------------------------------ | ------------- |
        | `moonshot-api-key`     | `https://api.moonshot.ai/v1`   | 国际          |
        | `moonshot-api-key-cn`  | `https://api.moonshot.cn/v1`   | 中国          |
      </Step>
      <Step title="运行新手引导">
        ```bash
        openclaw onboard --auth-choice moonshot-api-key
        ```

        对于中国端点：

        ```bash
        openclaw onboard --auth-choice moonshot-api-key-cn
        ```
      </Step>
      <Step title="将 Kimi K3 设为默认模型">
        新手引导会保留 Kimi K2.6 作为初始默认模型。需要使用 Kimi K3 时，请显式切换：

        ```bash
        openclaw models set moonshot/kimi-k3
        ```
      </Step>
      <Step title="验证模型是否可用">
        ```bash
        openclaw models list --provider moonshot
        ```
      </Step>
      <Step title="运行实时冒烟测试">
        如果想在不影响正常会话的情况下验证模型访问和成本跟踪，请使用隔离的状态目录：

        ```bash
        OPENCLAW_CONFIG_PATH=/tmp/openclaw-kimi/openclaw.json \
        OPENCLAW_STATE_DIR=/tmp/openclaw-kimi \
        openclaw agent --local \
          --session-id live-kimi-cost \
          --message '请严格回复：KIMI_LIVE_OK' \
          --thinking max \
          --json
        ```

        JSON 响应应报告 `provider: "moonshot"` 和
        `model: "kimi-k3"`。当 Moonshot 返回用量元数据时，助手转录条目会在 `usage.cost` 下存储标准化的 token 用量和估算成本。
      </Step>
    </Steps>

    ### 配置示例

    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: {
            // moonshot-kimi-k2-aliases:start
            "moonshot/kimi-k2.6": { alias: "Kimi K2.6" },
            "moonshot/kimi-k3": { alias: "Kimi K3" },
            "moonshot/kimi-k2.7-code": { alias: "Kimi K2.7 Code" },
            "moonshot/kimi-k2.7-code-highspeed": { alias: "Kimi K2.7 Code HighSpeed" },
            "moonshot/kimi-k2.5": { alias: "Kimi K2.5" },
            // moonshot-kimi-k2-aliases:end
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              // moonshot-kimi-k2-models:start
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k3",
                name: "Kimi K3",
                reasoning: true,
                thinkingLevelMap: {
                  off: null,
                  minimal: null,
                  low: null,
                  medium: null,
                  high: null,
                  xhigh: "max",
                  max: "max",
                },
                input: ["text", "image"],
                cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 0 },
                contextWindow: 1048576,
                maxTokens: 1048576,
              },
              {
                id: "kimi-k2.7-code",
                name: "Kimi K2.7 Code",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.19, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.7-code-highspeed",
                name: "Kimi K2.7 Code HighSpeed",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 1.9, output: 8, cacheRead: 0.38, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.5",
                name: "Kimi K2.5",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.6, output: 3, cacheRead: 0.1, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              // moonshot-kimi-k2-models:end
            ],
          },
        },
      },
    }
    ```

  </Tab>

  <Tab title="Kimi Coding">
    **最适合：**通过 Kimi Coding 端点执行以代码为重点的任务。

    <Note>
    Kimi Coding 使用与 Moonshot（`moonshot/...`）不同的 API key 和提供商前缀（`kimi/...`）。当前引用包括：上下文为 256K 的 `kimi/k3`、1M 档位的 `kimi/k3[1m]`、`kimi/kimi-for-coding` 和 `kimi/kimi-for-coding-highspeed`。旧引用 `kimi/kimi-code` 和 `kimi/k2p5` 仍可接受，并会标准化为 `kimi/kimi-for-coding`。
    </Note>

    该编码服务同时接受 OpenAI 兼容的
    `https://api.kimi.com/coding/v1` 客户端和 Anthropic 兼容的
    `https://api.kimi.com/coding/` 客户端。此插件使用 Anthropic Messages。
    请在
    [Kimi Code 控制台](https://www.kimi.com/code/console)中创建会员密钥；当前会员价格请参阅 [Kimi 定价页面](https://www.kimi.com/membership/pricing)。

    <Steps>
      <Step title="安装插件">
        ```bash
        openclaw plugins install @openclaw/kimi-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="运行新手引导">
        ```bash
        openclaw onboard --auth-choice kimi-code-api-key
        ```
      </Step>
      <Step title="设置默认模型">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "kimi/kimi-for-coding" },
            },
          },
        }
        ```
      </Step>
      <Step title="验证模型是否可用">
        ```bash
        openclaw models list --provider kimi
        ```
      </Step>
    </Steps>

    Kimi Code K3 默认以 `max` 进行深度思考。`/think off` 会发送
    `thinking.type: "disabled"`；`/think max` 会发送 K3 的最大强度自适应思考请求。
    过时的较低思考级别会解析为受支持的 `max` 级别。1M 模型要求拥有 Allegretto 或更高级别的 Kimi 会员资格；Moderato 会员请使用 `kimi/k3`。

    有关当前方案的可用情况，请参阅官方 [Kimi Code 模型表](https://www.kimi.com/code/docs/en/kimi-code/models.html)。

    ### 配置示例

    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: {
            "kimi/kimi-for-coding": { alias: "Kimi" },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

## Kimi Web 搜索

Moonshot 插件还会将 **Kimi** 注册为 `web_search` 提供商，其后端由 Moonshot Web 搜索提供支持。

<Steps>
  <Step title="运行交互式 Web 搜索设置">
    ```bash
    openclaw configure --section web
    ```

    在 Web 搜索部分选择 **Kimi**，以存储
    `plugins.entries.moonshot.config.webSearch.*`。

  </Step>
  <Step title="配置 Web 搜索区域和模型">
    交互式设置会提示配置：

    | 设置                | 选项                                                                 |
    | ------------------- | -------------------------------------------------------------------- |
    | API 区域            | `https://api.moonshot.ai/v1`（国际）或 `https://api.moonshot.cn/v1`（中国） |
    | Web 搜索模型        | 默认为 `kimi-k2.6`                                             |

  </Step>
</Steps>

配置位于 `plugins.entries.moonshot.config.webSearch` 下：

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // 或使用 KIMI_API_KEY / MOONSHOT_API_KEY
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.6",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

## 高级配置

<AccordionGroup>
  <Accordion title="原生思考模式">
    Moonshot API Kimi K3 始终以最大强度进行推理。OpenClaw 仅公开
    `/think max`，发送 `reasoning_effort: "max"`，并忽略过时的较低级别或
    `off` 设置。

    Kimi Code K3 提供 `/think off|max`。其 Anthropic 兼容端点
    接收用于关闭的 `thinking.type: "disabled"`，或使用
    `output_config.effort: "max"` 表示最大值的自适应思考。这同时适用于 `kimi/k3` 和
    `kimi/k3[1m]`。
    Moonshot API K3 支持 `auto`、`none`、`required` 和固定工具选择，
    因此 OpenClaw 会保留请求的 `tool_choice`。对于多轮工具使用，
    OpenClaw 会保留 Moonshot 重放契约所需的助手推理内容。

    Kimi K2.7 Code 始终使用原生思考。Moonshot 要求客户端
    对此模型省略 `thinking` 字段，因此 OpenClaw 仅提供 `on`，
    并忽略过时的 `off` 设置。K2.7 还固定了 `temperature`、`top_p`、`n`、
    `presence_penalty` 和 `frequency_penalty`；OpenClaw 会省略这些字段已配置的
    覆盖值。

    其他 Moonshot Kimi 模型支持二元原生思考：

    - `thinking: { type: "enabled" }`
    - `thinking: { type: "disabled" }`

    通过 `agents.defaults.models.<provider/model>.params` 按模型配置：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "disabled" },
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw 会为这些模型映射运行时 `/think` 级别：

    | `/think` 级别       | Moonshot 行为          |
    | -------------------- | -------------------------- |
    | `/think off`         | `thinking.type=disabled`   |
    | 任何非关闭级别    | `thinking.type=enabled`    |

    <Warning>
    启用 Moonshot K2 思考时，`tool_choice` 必须为 `auto` 或 `none`。固定工具选择（`type: "tool"` 或 `type: "function"`）会改为强制将思考恢复为 `disabled`，以便请求的工具仍能运行；`tool_choice: "required"` 则会被规范化为 `auto`。Kimi K2.7 Code 无法禁用思考，因此其不兼容的 `tool_choice` 会被规范化为 `auto`。Kimi K3 使用独立的推理强度契约，并保留受支持的工具选择。
    </Warning>

    Kimi K2.6 还接受可选的 `thinking.keep` 字段，用于控制
    `reasoning_content` 的多轮保留。将其设为 `"all"` 可在各轮之间保留完整
    推理；省略该字段（或保留为 `null`）则使用服务器
    默认策略。OpenClaw 仅为
    `moonshot/kimi-k2.6` 转发 `thinking.keep`，并从其他模型中移除该字段。Kimi K2.7 Code
    默认保留完整推理历史记录，而 OpenClaw 会省略整个
    `thinking` 字段。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "enabled", keep: "all" },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="工具调用 ID 清理">
    Moonshot Kimi 提供形如 `functions.<name>:<index>` 的原生 tool_call ID。OpenClaw 会保留每个原生 Kimi ID 的首次出现，并将之后的重复项重写为确定性的 OpenAI 风格 `call_*` ID。匹配的工具结果会使用同一 ID 重新映射，因此重放仍保持唯一，同时不会移除 Kimi 的首个原生 ID。此行为已接入内置 Moonshot provider，无法由用户配置。
  </Accordion>

  <Accordion title="流式用量兼容性">
    原生 Moonshot 端点（`https://api.moonshot.ai/v1` 和
    `https://api.moonshot.cn/v1`）声明支持流式用量兼容性。
    OpenClaw 根据端点主机而非 provider ID 判断，因此指向相同原生
    Moonshot 主机的自定义 provider ID 会继承相同的
    流式用量行为。

    使用目录中的 K2.6 定价时，包含输入、输出和
    缓存读取 token 的流式用量还会转换为本地预估美元费用，用于
    `/status`、`/usage full`、`/usage cost` 和基于转录记录的会话
    计费。

  </Accordion>

  <Accordion title="端点和模型引用参考">
    | 提供商   | 模型引用前缀 | 端点                      | 身份验证环境变量        |
    | ---------- | ---------------- | ------------------------------ | ------------------- |
    | Moonshot   | `moonshot/`      | `https://api.moonshot.ai/v1`  | `MOONSHOT_API_KEY`  |
    | Moonshot CN| `moonshot/`      | `https://api.moonshot.cn/v1`  | `MOONSHOT_API_KEY`  |
    | Kimi Coding| `kimi/`          | Kimi Coding 端点           | `KIMI_API_KEY`      |
    | Web 搜索 | 不适用              | 与 Moonshot API 区域相同    | `KIMI_API_KEY` 或 `MOONSHOT_API_KEY` |

    - Kimi Web 搜索使用 `KIMI_API_KEY` 或 `MOONSHOT_API_KEY`，默认使用 `https://api.moonshot.ai/v1`，模型为 `kimi-k2.6`。
    - 如有需要，可在 `models.providers` 中覆盖定价和上下文元数据。
    - 如果 Moonshot 为某个模型发布了不同的上下文限制，请相应调整 `contextWindow`。

  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="模型选择" href="/zh-CN/concepts/model-providers" icon="layers">
    选择提供商、模型引用和故障转移行为。
  </Card>
  <Card title="Web 搜索" href="/zh-CN/tools/web" icon="magnifying-glass">
    配置包括 Kimi 在内的 Web 搜索提供商。
  </Card>
  <Card title="配置参考" href="/zh-CN/gateway/configuration-reference" icon="gear">
    提供商、模型和插件的完整配置模式。
  </Card>
  <Card title="Moonshot 开放平台" href="https://platform.moonshot.ai" icon="globe">
    Moonshot API 密钥管理和文档。
  </Card>
</CardGroup>
