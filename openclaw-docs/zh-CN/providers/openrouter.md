---
read_when:
    - 你希望使用一个 API key 访问多个 LLM
    - 你想通过 OpenClaw 中的 OpenRouter 运行模型
    - 你想使用 OpenRouter 生成图像
    - 你想使用 OpenRouter 生成音乐
    - 你想使用 OpenRouter 生成视频
summary: 使用 OpenRouter 的统一 API 在 OpenClaw 中访问多种模型
title: OpenRouter
x-i18n:
    generated_at: "2026-07-26T05:59:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0936a10222f44f376dee081b7ee0678cddc3bc4579ac0006321dc1012d59bcf
    source_path: providers/openrouter.md
    workflow: 16
---

OpenRouter 通过一个 API 和一个密钥将请求路由到许多模型。它兼容
OpenAI，因此 OpenClaw 使用与其他代理提供商相同的
`openai-completions` 风格传输与其通信。

## 入门指南

<Tabs>
  <Tab title="OAuth">
    <Steps>
      <Step title="运行 OAuth 新手引导">
        ```bash
        openclaw onboard --auth-choice openrouter-oauth
        ```

        OpenClaw 会打开 OpenRouter 的浏览器登录流程（PKCE），用授权码换取
        OpenRouter API key，并将其存储在默认的 OpenRouter 身份验证配置文件中。
        在远程/无头主机上，OpenClaw 会输出登录 URL，并要求你在登录后粘贴
        重定向 URL。
      </Step>
      <Step title="（可选）切换到特定模型">
        新手引导默认使用 `openrouter/auto`。之后可以选择一个具体模型：

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
  <Tab title="API key">
    <Steps>
      <Step title="获取你的 API key">
        在 [openrouter.ai/keys](https://openrouter.ai/keys) 创建 API key。
      </Step>
      <Step title="运行 API key 新手引导">
        ```bash
        openclaw onboard --auth-choice openrouter-api-key
        ```
      </Step>
      <Step title="（可选）切换到特定模型">
        新手引导默认使用 `openrouter/auto`。之后可以选择一个具体模型：

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## 配置示例

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/auto" },
    },
  },
}
```

## 模型引用

<Note>
模型引用遵循 `openrouter/<provider>/<model>` 模式。有关可用提供商和模型的完整列表，
请参阅 [/concepts/model-providers](/zh-CN/concepts/model-providers)。
</Note>

当实时目录发现不可用时，使用以下内置回退模型：

| 模型引用                          | 说明                         |
| --------------------------------- | ---------------------------- |
| `openrouter/auto`                 | OpenRouter 自动路由          |
| `openrouter/moonshotai/kimi-k2.6` | 通过 MoonshotAI 使用 Kimi K2.6 |
| `openrouter/moonshotai/kimi-k2.5` | 通过 MoonshotAI 使用 Kimi K2.5 |

任何其他 `openrouter/<provider>/<model>` 引用，包括
`openrouter/openrouter/fusion`（参阅 [Fusion 路由器](#fusion-router)），都会根据 OpenRouter
的实时模型目录动态解析。

## 图像生成

OpenRouter 可以为 `image_generate` 工具提供支持。在
`agents.defaults.mediaModels.image` 下设置 OpenRouter 图像模型：

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openrouter/google/gemini-3.1-flash-image-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

OpenClaw 使用 `modalities: ["image", "text"]` 将图像请求发送到 OpenRouter 的
chat-completions 图像 API。Gemini 图像模型还会通过 OpenRouter 的
`image_config` 接收 `aspectRatio` 和 `resolution` 提示；其他
图像模型不会接收这些提示。对于速度较慢的模型，请使用
`agents.defaults.mediaModels.image.timeoutMs`；但 `image_generate` 工具每次调用的
`timeoutMs` 仍具有更高优先级。

## 视频生成

OpenRouter 可以通过其异步 `/videos` API 为
`video_generate` 工具提供支持。在 `agents.defaults.mediaModels.video` 下设置
OpenRouter 视频模型：

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openrouter/google/veo-3.1-fast",
      },
    },
  },
}
```

OpenClaw 会提交文生视频和图生视频任务，轮询返回的
`polling_url`，并从 OpenRouter 的 `unsigned_urls` 或任务内容端点
下载完成的视频。参考图像默认用作首帧/末帧图像；标记为
`reference_image` 的图像则作为输入参考发送。内置的
`google/veo-3.1-fast` 默认模型支持 4/6/8 秒时长、
`720P`/`1080P` 分辨率，以及
`16:9`/`9:16` 宽高比。
不支持视频转视频：上游 API 仅接受文本和图像引用。

## 音乐生成

OpenRouter 可以通过 chat-completions 音频输出为
`music_generate` 工具提供支持。在 `agents.defaults.mediaModels.music` 下设置
OpenRouter 音频模型：

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "openrouter/google/lyria-3-pro-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

内置的 OpenRouter 音乐提供商默认使用 `google/lyria-3-pro-preview`，
同时还提供 `google/lyria-3-clip-preview`。OpenClaw 会发送
`modalities:
["text", "audio"]`，以流式方式接收响应、收集音频分块，并将结果保存为
生成的媒体，以便投递到渠道。Lyria 模型通过共享的
`music_generate image=...` 参数接受一张参考图像。
流式音频、转录文本保留以及派生的 SSE 事件信封均受
`agents.defaults.mediaMaxMb` 限制（默认音频上限为 16 MB）。

## 文本转语音

OpenRouter 可通过其兼容 OpenAI 的
`/audio/speech` 端点充当 TTS 提供商。

```json5
{
  tts: {
    auto: "always",
    provider: "openrouter",
    providers: {
      openrouter: {
        model: "hexgrad/kokoro-82m",
        speakerVoice: "af_alloy",
        responseFormat: "mp3",
      },
    },
  },
}
```

如果省略 `tts.providers.openrouter.apiKey`，TTS 会回退到
`models.providers.openrouter.apiKey`，然后再回退到 `OPENROUTER_API_KEY`。

## 语音转文本（入站音频）

OpenRouter 可通过共享的 `tools.media.audio` 路径，使用其 STT 端点
（`/audio/transcriptions`）转录入站语音/音频附件。
这适用于任何将入站语音/音频转发到媒体理解预检的渠道插件。

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "openrouter", model: "openai/whisper-large-v3-turbo" }],
      },
    },
  },
}
```

OpenClaw 按照 OpenRouter 的 STT 契约，将 base64 音频置于
`input_audio` 下，以 JSON 格式发送 OpenRouter STT 请求，而不是采用
multipart OpenAI 表单上传。

## Fusion 路由器

OpenRouter Fusion 会将一个 OpenClaw 模型引用并行发送到多个 OpenRouter 模型，
由 OpenRouter 评判这些模型的回答，然后通过常规 OpenRouter 端点返回一个最终响应。
上游模型 slug 为 `openrouter/fusion`，因此 OpenClaw 模型引用同时包含 OpenClaw
提供商前缀和上游 OpenRouter 命名空间：

```bash
openclaw models set openrouter/openrouter/fusion
```

通过模型的 `params.extraBody` 配置 Fusion 的模型组和评判模型；
这些字段会直接转发到 OpenRouter chat-completions 请求体中。
Fusion 可配合 OAuth 或 API key 新手引导使用；如果使用 OAuth，
请省略下面的 `env.OPENROUTER_API_KEY` 行。

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/openrouter/fusion" },
      models: {
        "openrouter/openrouter/fusion": {
          params: {
            extraBody: {
              plugins: [
                {
                  id: "fusion",
                  analysis_models: [
                    "google/gemini-3.5-flash",
                    "moonshotai/kimi-k2.6",
                    "deepseek/deepseek-v4-pro",
                  ],
                  model: "google/gemini-3.5-flash",
                },
              ],
            },
          },
        },
      },
    },
  },
}
```

`analysis_models` 是并行模型组；Fusion 插件配置中的 `model`
是评判模型。在正常的智能体/聊天轮次中，请勿将顶层 `tool_choice` 设置为
`"required"` 来尝试强制使用 Fusion：OpenClaw 轮次可能包含自身的工具定义，
而顶层必选工具选项可能会选择其中某个工具，而不是 Fusion 路由器。
存在此 Fusion 插件配置时，OpenClaw 会添加一条经过清理的系统提示说明，
其中列出已配置的分析模型和评判模型，使智能体能够回答有关其自身 Fusion 模型组的问题。
其他 `extraBody` 字段不会复制到提示词中。

Fusion 的设计本身就较慢：OpenRouter 会将提示词分发给多个分析模型，
然后执行评判/综合步骤，因此其延迟高于直接的单模型请求。
应将其用于需要审慎处理的高质量回答或升级处理路径，而不要将其用作对延迟敏感的默认选项。
保持模型组规模较小，并选择速度更快的分析模型和评判模型，以缩短响应时间。

使用一次性本地调用测试已配置的引用：

```bash
openclaw infer model run --local \
  --model openrouter/openrouter/fusion \
  --prompt "Reply with exactly: FUSION_OK" \
  --json
```

## 身份验证和请求头

OpenRouter 使用来自 API key 的 Bearer 令牌。OpenRouter OAuth 是一种 PKCE
登录流程，会签发 OpenRouter API key，因此 OpenClaw 将结果存储在与手动
API key 设置所用相同的 `openrouter:default` API key 身份验证配置文件中。

若要在现有安装中登录或轮换已存储的密钥，而不重新运行完整的新手引导：

```bash
openclaw models auth login --provider openrouter --method oauth
openclaw models auth login --provider openrouter --method api-key
```

对于经过验证的 OpenRouter 请求（`https://openrouter.ai/api/v1`），OpenClaw 会添加
OpenRouter 文档中规定的应用归属请求头：

| 请求头                    | 值                                                                                                  |
| ------------------------- | ------------------------------------------------------------------------------------------------------ |
| `HTTP-Referer`            | `https://openclaw.ai`                                                                                  |
| `X-OpenRouter-Title`      | `OpenClaw`                                                                                             |
| `X-OpenRouter-Categories` | `cli-agent,cloud-agent,programming-app,creative-writing,writing-assistant,general-chat,personal-agent` |

<Warning>
如果将 OpenRouter 提供商重新指向其他代理或基础 URL，OpenClaw
**不会**注入这些 OpenRouter 专用请求头或 Anthropic 缓存标记。
</Warning>

## 高级配置

<AccordionGroup>
  <Accordion title="响应缓存">
    OpenRouter 响应缓存需主动启用。请按模型启用：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/auto": {
              params: {
                responseCache: true,
                responseCacheTtlSeconds: 300,
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw 会发送 `X-OpenRouter-Cache: true`，并在已配置时发送
    `X-OpenRouter-Cache-TTL`。`responseCacheClear: true` 会强制刷新当前请求，
    并存储替换后的响应。也接受 snake_case 别名
    （`response_cache`、`response_cache_ttl_seconds`、
    `response_cache_clear`），以及不带 `Seconds` 后缀的
    `responseCacheTtl` / `response_cache_ttl`。

    此功能与提供商提示词缓存以及 OpenRouter 的 Anthropic
    `cache_control` 标记相互独立。它仅适用于经过验证的
    `openrouter.ai` 路由，不适用于自定义代理基础 URL。

  </Accordion>

  <Accordion title="Anthropic 缓存标记">
    在经过验证的 OpenRouter 路由上，Anthropic 模型引用会保留 OpenRouter 的
    Anthropic `cache_control` 标记，以便在系统/开发者提示词块中更好地复用提示词缓存。
  </Accordion>

  <Accordion title="Anthropic 推理预填充">
    在经过验证的 OpenRouter 路由上，启用推理的 Anthropic 模型引用会在请求到达
    OpenRouter 之前移除末尾的助手预填充轮次，以满足 Anthropic 对推理对话必须以用户轮次结束的要求。
  </Accordion>

  <Accordion title="思考 / 推理注入">
    在支持的非 `auto` 路由上，OpenClaw 会将所选思考级别
    映射到 OpenRouter 代理推理载荷。`openrouter/auto` 和不支持的
    模型提示会跳过该注入。过时的 `openrouter/hunter-alpha` 引用也会
    跳过该注入，因为 OpenRouter 在该已停用路由上可能会在推理
    字段中返回最终答案文本。
  </Accordion>

  <Accordion title="DeepSeek V4 推理重放">
    在已验证的 OpenRouter 路由上，`openrouter/deepseek/deepseek-v4-flash` 和
    `openrouter/deepseek/deepseek-v4-pro` 会在重放的助手轮次中补全缺失的 `reasoning_content`，
    从而使思考/工具对话保持 DeepSeek V4 所要求的后续交互格式。OpenClaw 会为
    这些路由发送 OpenRouter 支持的 `reasoning.effort` 值：`xhigh`/`max` 映射为 `xhigh`，
    其他所有非关闭级别均映射为 `high`。
  </Accordion>

  <Accordion title="仅限 OpenAI 的请求塑形">
    OpenRouter 通过代理式 OpenAI 兼容路径运行，因此不会转发
    仅限原生 OpenAI 的请求塑形，例如 `serviceTier`、Responses `store`、
    OpenAI 推理兼容载荷和提示词缓存提示。
  </Accordion>

  <Accordion title="由 Gemini 支持的路由">
    由 Gemini 支持的 OpenRouter 引用仍使用代理 Gemini 路径：OpenClaw 会在此处保留
    Gemini 思考签名清理，但不会启用原生
    Gemini 重放验证或引导重写。
  </Accordion>

  <Accordion title="提供商路由元数据">
    OpenRouter 支持用于底层提供商
    路由的 `provider` 请求对象。使用 `models.providers.openrouter.params.provider` 为所有 OpenRouter 文本模型请求
    配置默认策略：

    ```json5
    {
      models: {
        providers: {
          openrouter: {
            params: {
              provider: {
                sort: "latency",
                require_parameters: true,
                data_collection: "deny",
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw 会将该对象作为请求的 `provider`
    载荷转发给 OpenRouter。请使用 OpenRouter 文档中说明的 snake_case 字段，包括 `sort`、
    `only`、`ignore`、`order`、`allow_fallbacks`、`require_parameters`、
    `data_collection`、`quantizations`、`max_price`、`preferred_max_latency`、
    `preferred_min_throughput`、`zdr` 和 `enforce_distillable_text`。

    按模型设置的参数会覆盖提供商范围的路由对象：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/anthropic/claude-sonnet-4-6": {
              params: {
                provider: {
                  order: ["anthropic"],
                  allow_fallbacks: false,
                },
              },
            },
          },
        },
      },
    }
    ```

    这仅适用于 OpenRouter 聊天补全路由。直接使用 Anthropic、
    Google、OpenAI 或自定义提供商的路由会忽略 OpenRouter 路由参数。

  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="模型选择" href="/zh-CN/concepts/model-providers" icon="layers">
    选择提供商、模型引用和故障转移行为。
  </Card>
  <Card title="配置参考" href="/zh-CN/gateway/configuration-reference" icon="gear">
    智能体、模型和提供商的完整配置参考。
  </Card>
</CardGroup>
