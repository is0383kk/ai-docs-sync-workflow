---
read_when:
    - 通过智能体生成或编辑图像
    - 配置图像生成提供商和模型
    - 了解 image_generate 工具参数
sidebarTitle: Image generation
summary: 通过 image_generate 在 OpenAI、Google、fal、Microsoft Foundry、MiniMax、ComfyUI、DeepInfra、OpenRouter、LiteLLM、xAI、Vydra 中生成和编辑图像
title: 图像生成
x-i18n:
    generated_at: "2026-07-26T06:03:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9688b1bc649713d8ed345a69a28d20b36ecd768b6a6d28a2d6c022d65b081862
    source_path: tools/image-generation.md
    workflow: 16
---

`image_generate` 工具通过你配置的提供商创建和编辑图像。在聊天会话中，它以异步方式运行：OpenClaw 会记录一个后台任务，立即返回任务 ID，并在提供商完成处理后唤醒智能体。完成任务的智能体遵循会话的常规可见回复模式：配置后自动发送最终回复；如果会话要求使用消息工具，则使用 `message(action="send")`。如果请求方会话处于非活动状态或其主动唤醒失败，OpenClaw 会发送包含所生成图像的幂等直接回退消息，确保结果不会丢失。

<Note>
仅当至少有一个图像生成提供商可用时，此工具才会出现。如果在智能体的工具中看不到 `image_generate`，请配置 `agents.defaults.mediaModels.image`、设置提供商 API key，或使用 OpenAI ChatGPT/Codex OAuth 登录。
</Note>

## 快速开始

<Steps>
  <Step title="配置身份验证">
    为至少一个提供商设置 API key（例如 `OPENAI_API_KEY`、`GEMINI_API_KEY`、`OPENROUTER_API_KEY`），或使用 OpenAI Codex OAuth 登录。
  </Step>
  <Step title="选择默认模型（可选）">
    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "openai/gpt-image-2",
            timeoutMs: 180_000,
          },
        },
      },
    }
    ```

    ChatGPT/Codex OAuth 使用相同的 `openai/gpt-image-2` 模型引用。配置 `openai` OAuth 配置文件后，OpenClaw 会通过该 OAuth 配置文件路由图像请求，而不是先尝试 `OPENAI_API_KEY`。显式设置 `models.providers.openai` 配置（API key、自定义/Azure 基础 URL）后，将重新使用直接调用 OpenAI Images API 的路由。

  </Step>
  <Step title="向智能体提出请求">
    _“生成一张友好机器人吉祥物的图像。”_

    智能体会自动调用 `image_generate`。无需将工具加入允许列表——当提供商可用时，默认启用此工具。该工具会返回后台任务 ID，任务就绪后，完成任务的智能体会通过 `message` 工具发送生成的附件。

  </Step>
</Steps>

<Warning>
对于 LocalAI 等兼容 OpenAI 的局域网端点，请保留自定义 `models.providers.openai.baseUrl`，并通过 `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: true` 显式选择启用。默认情况下，私有和内部图像端点仍会被阻止。
</Warning>

## 常用路由

| 目标                                                 | 模型引用                                          | 身份验证                                   |
| ---------------------------------------------------- | -------------------------------------------------- | -------------------------------------- |
| 使用 API 计费的 OpenAI 图像生成             | `openai/gpt-image-2`                               | `OPENAI_API_KEY`                       |
| 使用 Codex 订阅身份验证的 OpenAI 图像生成 | `openai/gpt-image-2`                               | OpenAI ChatGPT/Codex OAuth             |
| OpenAI 透明背景 PNG/WebP               | `openai/gpt-image-1.5`                             | `OPENAI_API_KEY` 或 OpenAI Codex OAuth |
| DeepInfra 图像生成                           | `deepinfra/black-forest-labs/FLUX-1-schnell`       | `DEEPINFRA_API_KEY`                    |
| fal Krea 2 表现力/风格定向生成      | `fal/krea/v2/medium/text-to-image`                 | `FAL_KEY`                              |
| OpenRouter 图像生成                          | `openrouter/google/gemini-3.1-flash-image-preview` | `OPENROUTER_API_KEY`                   |
| LiteLLM 图像生成                             | `litellm/gpt-image-2`                              | `LITELLM_API_KEY`                      |
| Microsoft Foundry MAI 图像生成               | `microsoft-foundry/<deployment-name>`              | `AZURE_OPENAI_API_KEY` 或 Entra ID     |
| Google Gemini 图像生成                       | `google/gemini-3.1-flash-image`                    | `GEMINI_API_KEY` 或 `GOOGLE_API_KEY`   |

同一工具同时处理文本生成图像和参考图像编辑。单张参考图像使用 `image`，多张参考图像使用 `images`。对于 fal 上的 Krea 2 模型，这些参考图像会作为风格参考发送，而不是作为编辑输入发送。
提供商支持的输出提示（例如 `quality`、`outputFormat` 和 `background`）会在可用时转发；如果提供商未声明支持，则会报告为已忽略。内置透明背景支持仅适用于 OpenAI；如果其他提供商的后端输出包含 PNG Alpha 通道，也可能保留透明度。

## 支持的提供商

| 提供商          | 默认模型                           | 编辑支持                       | 身份验证                                                  |
| ----------------- | --------------------------------------- | ---------------------------------- | ----------------------------------------------------- |
| ComfyUI           | `workflow`                              | 是（1 张图像，由工作流配置） | `COMFY_API_KEY`，云端使用 `COMFY_CLOUD_API_KEY`    |
| DeepInfra         | `black-forest-labs/FLUX-1-schnell`      | 是（1 张图像）                      | `DEEPINFRA_API_KEY`                                   |
| fal               | `fal-ai/flux/dev`                       | 是（模型特定限制）        | `FAL_KEY`                                             |
| Google            | `gemini-3.1-flash-image`                | 是（最多 5 张图像）               | `GEMINI_API_KEY` 或 `GOOGLE_API_KEY`                  |
| LiteLLM           | `gpt-image-2`                           | 是（最多 5 张输入图像）         | `LITELLM_API_KEY`                                     |
| Microsoft Foundry | `<deployment-name>`                     | 是（仅限 MAI-Image-2.5 模型）    | `AZURE_OPENAI_API_KEY` 或 Entra ID（`az login`）       |
| MiniMax           | `image-01`                              | 是（主体参考）            | `MINIMAX_API_KEY` 或 MiniMax OAuth（`minimax-portal`） |
| OpenAI            | `gpt-image-2`                           | 是（最多 5 张图像）               | `OPENAI_API_KEY` 或 OpenAI ChatGPT/Codex OAuth        |
| OpenRouter        | `google/gemini-3.1-flash-image-preview` | 是（最多 5 张输入图像）         | `OPENROUTER_API_KEY`                                  |
| Vydra             | `grok-imagine`                          | 否                                 | `VYDRA_API_KEY`                                       |
| xAI               | `grok-imagine-image`                    | 是（最多 3 张图像）               | `XAI_API_KEY`                                         |

使用 `action: "list"` 在运行时检查可用的提供商和模型：

```text
/tool image_generate action=list
```

使用 `action: "status"` 检查当前会话的活动图像生成任务：

```text
/tool image_generate action=status
```

## 提供商能力

| 能力            | ComfyUI            | DeepInfra | fal                                            | Google         | Microsoft Foundry | MiniMax               | OpenAI         | Vydra | xAI            |
| --------------------- | ------------------ | --------- | ---------------------------------------------- | -------------- | ----------------- | --------------------- | -------------- | ----- | -------------- |
| 生成（最大数量）  | 1                  | 4         | 4                                              | 4              | 1                 | 9                     | 4              | 1     | 4              |
| 编辑/参考      | 1 张图像（工作流） | 1 张图像   | Flux：1；GPT：10；Krea 风格参考：10；NB2：14 | 最多 5 张图像 | 1 张图像           | 1 张图像（主体参考） | 最多 5 张图像 | -     | 最多 3 张图像 |
| 尺寸控制          | -                  | ✓         | ✓                                              | ✓              | ✓                 | -                     | 最高 4K       | -     | -              |
| 宽高比          | -                  | -         | ✓                                              | ✓              | -                 | ✓                     | -              | -     | ✓              |
| 分辨率（1K/2K/4K） | -                  | -         | ✓                                              | ✓              | -                 | -                     | -              | -     | 1K、2K         |

## 工具参数

<ParamField path="prompt" type="string" required>
  图像生成提示词。`action: "generate"` 必须提供此参数。
</ParamField>
<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  使用 `"status"` 检查活动会话任务，或使用 `"list"` 在运行时检查可用的提供商和模型。
</ParamField>
<ParamField path="model" type="string">
  提供商/模型覆盖（例如 `openai/gpt-image-2`）。如需透明的 OpenAI 背景，请使用 `openai/gpt-image-1.5`。
</ParamField>
<ParamField path="image" type="string">
  用于编辑模式的单张参考图像路径或 URL。
</ParamField>
<ParamField path="images" type="string[]">
  用于编辑模式或风格参考模型的多张参考图像（通过共享工具最多可传递 14 张；仍需遵守提供商特定的限制）。
</ParamField>
<ParamField path="size" type="string">
  尺寸提示：`1024x1024`、`1536x1024`、`1024x1536`、`2048x2048`、`3840x2160`。
</ParamField>
<ParamField path="aspectRatio" type="string">
  宽高比：`1:1`、`2:1`、`20:9`、`19.5:9`、`2:3`、`3:2`、`2.35:1`、`3:4`、
  `4:3`、`4:5`、`5:4`、`9:16`、`9:19.5`、`9:20`、`16:9`、`21:9`、`1:2`、`4:1`、
  `1:4`、`8:1`、`1:8`。提供商会验证其模型特定的子集。
</ParamField>
<ParamField path="resolution" type='"1K" | "2K" | "4K"'>分辨率提示。</ParamField>
<ParamField path="quality" type='"low" | "medium" | "high" | "auto"'>
  提供商支持时使用的质量提示。
</ParamField>
<ParamField path="outputFormat" type='"png" | "jpeg" | "webp"'>
  提供商支持时使用的输出格式提示。
</ParamField>
<ParamField path="background" type='"transparent" | "opaque" | "auto"'>
  提供商支持时使用的背景提示。对于支持透明度的提供商，请将 `transparent` 与 `outputFormat: "png"` 或 `"webp"` 配合使用。
</ParamField>
<ParamField path="count" type="number">要生成的图像数量（1-4）。</ParamField>
<ParamField path="timeoutMs" type="number">
  可选的提供商请求超时时间，以毫秒为单位。当 Codex 通过动态工具调用 `image_generate` 时，此单次调用值仍会覆盖已配置的默认值，且上限为 600000 ms。
</ParamField>
<ParamField path="filename" type="string">输出文件名提示。</ParamField>
<ParamField path="openai" type="object">
  仅适用于 OpenAI 的提示：`background`、`moderation`、`outputCompression` 和 `user`。
</ParamField>
<ParamField path="fal.creativity" type='"raw" | "low" | "medium" | "high"'>
  fal Krea 2 创意程度控制。默认为 `medium`。
</ParamField>

<Note>
并非所有提供商都支持全部参数。当回退提供商支持与请求选项相近的几何选项，而不支持完全一致的选项时，OpenClaw 会在提交前重新映射到最接近的受支持尺寸、宽高比或分辨率。对于未声明支持的提供商，不受支持的输出提示会被丢弃，并在工具结果中报告。工具结果会报告实际应用的设置；`details.normalization` 会记录从请求值到应用值的转换。
</Note>

## 配置

### 模型选择

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-2",
        timeoutMs: 180_000,
        fallbacks: [
          "openrouter/google/gemini-3.1-flash-image-preview",
          "google/gemini-3.1-flash-image",
          "fal/fal-ai/flux/dev",
        ],
      },
    },
  },
}
```

### 提供商选择顺序

OpenClaw 按以下顺序尝试提供商：

1. 工具调用中的 **`model` 参数**（如果智能体指定）。
2. 配置中的 **`imageGenerationModel.primary`**。
3. 按顺序使用 **`imageGenerationModel.fallbacks`**。
4. **自动检测**——仅限有身份验证支持的提供商默认值：
   - 首先使用当前默认提供商；
   - 然后按提供商 ID 顺序使用其余已注册的图像生成提供商。

如果提供商失败（身份验证错误、速率限制等），系统会自动尝试下一个已配置的
候选项。如果全部失败，错误中会包含每次尝试的详细信息。

<AccordionGroup>
  <Accordion title="每次调用的模型覆盖值均为精确指定">
    每次调用的 `model` 覆盖值只会尝试该提供商/模型，
    不会继续尝试已配置的主要提供商/后备提供商或自动检测到的提供商。
  </Accordion>
  <Accordion title="自动检测会感知身份验证状态">
    只有当 OpenClaw 确实能够对提供商进行身份验证时，该提供商的默认值
    才会进入候选列表。始终启用经过身份验证的提供商之间的自动回退；
    每次调用的 `model` 始终具有最终决定权。
  </Accordion>
  <Accordion title="超时">
    对于较慢的图像后端，请设置 `agents.defaults.mediaModels.image.timeoutMs`。
    每次调用的 `timeoutMs` 工具参数会覆盖已配置的默认值，
    已配置的默认值又会覆盖插件提供商定义的默认值。Google 和 OpenRouter
    托管的图像提供商默认使用 180 秒；Microsoft Foundry MAI、xAI 和
    Azure OpenAI 图像生成默认使用 600 秒。Codex 动态工具调用使用
    120 秒的 `image_generate` 桥接默认值，并在配置后遵循相同的超时预算，
    但上限为 OpenClaw 动态工具桥接的最大值 600000 ms。
  </Accordion>
  <Accordion title="在运行时检查">
    使用 `action: "list"` 检查当前已注册的提供商、
    它们的默认模型以及身份验证环境变量提示。
  </Accordion>
</AccordionGroup>

### 图像编辑

OpenAI、OpenRouter、Google、DeepInfra、fal、Microsoft Foundry、MiniMax、
ComfyUI 和 xAI 支持编辑参考图像。fal 上的 Krea 2 模型将相同的
`image` / `images` 字段用作风格参考，而不是编辑输入。
传入参考图像路径或 URL：

```text
“生成这张照片的水彩版本” + image: "/path/to/photo.jpg"
```

OpenAI、OpenRouter 和 Google 通过 `images` 参数支持最多 5 张参考图像；
xAI 最多支持 3 张。fal 对 Flux 图生图支持 1 张参考图像，对 GPT Image 2 编辑
最多支持 10 张，对 Krea 2 最多支持 10 张风格参考图像，对 Nano Banana 2 编辑
最多支持 14 张。Microsoft Foundry、MiniMax 和 ComfyUI 支持 1 张。

## 提供商深入解析

<AccordionGroup>
  <Accordion title="OpenAI gpt-image-2（以及 gpt-image-1.5）">
    OpenAI 图像生成默认使用 `openai/gpt-image-2`。如果配置了
    `openai` OAuth 配置文件，OpenClaw 会复用 Codex 订阅聊天模型
    所用的同一个 OAuth 配置文件，并通过 Codex Responses 后端发送图像请求。
    对于图像请求，`https://chatgpt.com/backend-api` 等旧版 Codex 基础 URL 会被规范化为
    `https://chatgpt.com/backend-api/codex`。OpenClaw **不会**为该请求静默回退到
    `OPENAI_API_KEY`——若要强制直接通过 OpenAI Images API 路由，
    请使用 API key、自定义基础 URL 或 Azure 端点显式配置
    `models.providers.openai`。

    仍可显式选择 `openai/gpt-image-1.5`、`openai/gpt-image-1` 和
    `openai/gpt-image-1-mini` 模型。若要输出透明背景的 PNG/WebP，请使用
    `gpt-image-1.5`；当前 `gpt-image-2` API 会拒绝
    `background: "transparent"`。

    `gpt-image-2` 通过同一个 `image_generate` 工具同时支持文生图
    和参考图像编辑。OpenClaw 会将 `prompt`、`count`、
    `size`、`quality`、`outputFormat` 以及参考图像
    转发给 OpenAI。OpenAI 不会直接接收 `aspectRatio` 或
    `resolution`；OpenClaw 会尽可能将其映射到受支持的
    `size`，否则工具会将其报告为被忽略的覆盖值。

    OpenAI 专属选项位于 `openai` 对象下：

    ```json
    {
      "quality": "low",
      "outputFormat": "jpeg",
      "openai": {
        "background": "opaque",
        "moderation": "low",
        "outputCompression": 60,
        "user": "end-user-42"
      }
    }
    ```

    `openai.background` 接受 `transparent`、`opaque` 或
    `auto`；透明输出需要 `outputFormat`
    `png` 或 `webp`，以及支持透明度的 OpenAI
    图像模型。OpenClaw 会将默认的 `gpt-image-2` 透明背景请求路由到
    `gpt-image-1.5`。`openai.outputCompression` 适用于 JPEG/WebP 输出，
    对 PNG 输出会被忽略。

    顶层 `background` 提示与提供商无关；选择 OpenAI provider 时，
    当前会映射到相同的 OpenAI `background` 请求字段。
    未声明支持背景的提供商会在 `ignoredOverrides` 中返回该值，
    而不会接收不受支持的参数。

    若要通过 Azure OpenAI 部署路由 OpenAI 图像生成，而不是使用
    `api.openai.com`，请参阅
    [Azure OpenAI 端点](/zh-CN/providers/openai#azure-openai-endpoints)。

  </Accordion>
  <Accordion title="Microsoft Foundry MAI 图像模型">
    Microsoft Foundry 图像生成在 `microsoft-foundry/` 提供商前缀下使用
    已部署的 MAI 图像部署名称。由于 MAI API 要求在 `model`
    字段中提供你的部署名称，因此没有提供商级默认模型：

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "microsoft-foundry/<deployment-name>",
            timeoutMs: 600_000,
          },
        },
      },
    }
    ```

    该提供商使用 Microsoft Foundry 的 MAI API，而不是 OpenAI Images API：

    - 生成端点：`/mai/v1/images/generations`
    - 编辑端点：`/mai/v1/images/edits`
    - 身份验证：`AZURE_OPENAI_API_KEY` / 提供商 API key，或通过 `az login` 使用 Entra ID
    - 输出：一张 PNG 图像
    - 尺寸：默认为 `1024x1024`；宽度和高度均须至少为 768 px，
      总像素数不得超过 1,048,576
    - 编辑：一张 PNG 或 JPEG 参考图像，仅
      `MAI-Image-2.5-Flash` 和 `MAI-Image-2.5` 部署支持

    仅使用提示词生成时，只需配置 Foundry 端点即可使用自定义部署名称。
    使用自定义部署名称进行编辑时，需要新手引导/模型元数据，以便 OpenClaw
    验证该部署是否由 `MAI-Image-2.5-Flash` 或 `MAI-Image-2.5` 提供支持。

    当前 MAI 图像模型包括 `MAI-Image-2.5-Flash`、`MAI-Image-2.5`、
    `MAI-Image-2e` 和 `MAI-Image-2`。有关设置和聊天模型行为，
    请参阅 [Microsoft Foundry 插件](/zh-CN/plugins/reference/microsoft-foundry)。

  </Accordion>
  <Accordion title="OpenRouter 图像模型">
    OpenRouter 图像生成使用相同的 `OPENROUTER_API_KEY`，
    并通过 OpenRouter 的聊天补全图像 API 路由。使用
    `openrouter/` 前缀选择 OpenRouter 图像模型：

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "openrouter/google/gemini-3.1-flash-image-preview",
          },
        },
      },
    }
    ```

    OpenClaw 会将 `prompt`、`count`、参考图像以及
    与 Gemini 兼容的 `aspectRatio` / `resolution` 提示转发给
    OpenRouter。当前内置的 OpenRouter 图像模型快捷方式包括
    `google/gemini-3.1-flash-image`、`google/gemini-3-pro-image` 和 `openai/gpt-5.4-image-2`。
    使用 `action: "list"` 查看已配置插件公开的内容。

  </Accordion>
  <Accordion title="fal Krea 2">
    fal 上的 Krea 2 模型使用 fal 原生 Krea 架构，而不是 Flux 使用的通用
    `image_size` 架构。OpenClaw 会发送：

    - `aspect_ratio`，用于宽高比提示
    - `creativity`，默认为 `medium`
    - 提供 `image` 或 `images` 时发送 `image_style_references`

    如需更快且富有表现力的插画，请选择 Krea 2 Medium；如需速度较慢、
    细节更丰富的照片级真实效果和纹理风格，请选择 Krea 2 Large：

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "fal/krea/v2/medium/text-to-image",
          },
        },
      },
    }
    ```

    Krea 2 当前每次请求返回一张图像。对于 Krea，建议使用
    `aspectRatio`；OpenClaw 会将 `size` 映射到最接近的
    Krea 支持宽高比，并会拒绝 Krea 的 `resolution`，而不是将其丢弃。
    如需使用 Krea 原生创意级别，请使用 `fal.creativity`：

    ```json
    {
      "model": "fal/krea/v2/medium/text-to-image",
      "prompt": "带有孔版印刷纹理的赛博杂志肖像",
      "aspectRatio": "9:16",
      "fal": {
        "creativity": "high"
      }
    }
    ```

  </Accordion>
  <Accordion title="MiniMax 双重身份验证">
    可通过两种内置 MiniMax 身份验证路径使用 MiniMax 图像生成：

    - `minimax/image-01`，用于 API key 设置
    - `minimax-portal/image-01`，用于 OAuth 设置

  </Accordion>
  <Accordion title="xAI grok-imagine-image">
    内置 xAI 提供商对仅含提示词的请求使用 `/v1/images/generations`，
    当存在 `image` 或 `images` 时使用
    `/v1/images/edits`。

    - 模型：`xai/grok-imagine-image`、`xai/grok-imagine-image-quality`
    - 数量：最多 4 张
    - 参考图像：一个 `image` 或最多三个 `images`
    - 宽高比：`1:1`、`16:9`、`9:16`、`4:3`、`3:4`、`3:2`、`2:3`、`2:1`、
      `1:2`、`19.5:9`、`9:19.5`、`20:9`、`9:20`
    - 分辨率：`1K`、`2K`
    - 输出：以 OpenClaw 管理的图像附件形式返回

    OpenClaw 有意不公开 xAI 原生的 `quality`、`mask`、
    `user` 或 `auto` 宽高比，直到这些控制项纳入共享的
    跨提供商 `image_generate` 合约。

  </Accordion>
</AccordionGroup>

## 示例

<Tabs>
  <Tab title="生成（4K 横向）">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="OpenClaw 图像生成的简洁编辑风格海报" size=3840x2160 count=1
```
  </Tab>
  <Tab title="生成（透明 PNG）">
```text
/tool image_generate action=generate model=openai/gpt-image-1.5 prompt="透明背景上的简洁红色圆形贴纸" outputFormat=png background=transparent
```

等效 CLI：

```bash
openclaw infer image generate \
  --model openai/gpt-image-1.5 \
  --output-format png \
  --background transparent \
  --prompt "透明背景上的简洁红色圆形贴纸" \
  --json
```

  </Tab>
  <Tab title="生成（OpenAI 低质量）">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="安静高效应用的低成本海报草稿" quality=low openai='{"moderation":"low"}'
```

等效 CLI：

```bash
openclaw infer image generate \
  --model openai/gpt-image-2 \
  --quality low \
  --openai-moderation low \
  --prompt "用于安静高效应用的低成本海报草稿" \
  --json
```

  </Tab>
  <Tab title="生成（两个正方形图像）">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="为一款专注平和体验的效率应用图标提供两个视觉方向" size=1024x1024 count=2
```
  </Tab>
  <Tab title="编辑（一个参考图像）">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="保留主体，将背景替换为明亮的摄影棚布景" image=/path/to/reference.png size=1024x1536
```
  </Tab>
  <Tab title="编辑（多个参考图像）">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="将第一张图像中的角色特征与第二张图像中的调色板相结合" images='["/path/to/character.png","/path/to/palette.jpg"]' size=1536x1024
```
  </Tab>
  <Tab title="Krea 风格参考">
```text
/tool image_generate action=generate model=fal/krea/v2/medium/text-to-image prompt="使用此调色板和印刷纹理创作一幅富有表现力的编辑风格肖像" images='["/path/to/palette.png","/path/to/texture.jpg"]' aspectRatio=9:16 fal='{"creativity":"high"}'
```
  </Tab>
</Tabs>

`openclaw infer image edit` 也支持相同的 `--output-format`、`--background`、`--quality` 和
`--openai-moderation` 标志；`--openai-background` 仍是 OpenAI 专用别名。除 OpenAI
以外的内置提供商目前未声明显式背景控制，因此对这些提供商使用
`background: "transparent"` 时，会报告该标志已被忽略。

## 相关内容

- [工具概览](/zh-CN/tools) - 所有可用的智能体工具
- [ComfyUI](/zh-CN/providers/comfy) - 本地 ComfyUI 和 Comfy Cloud 工作流设置
- [fal](/zh-CN/providers/fal) - fal 图像和视频提供商设置
- [Google (Gemini)](/zh-CN/providers/google) - Gemini 图像提供商设置
- [Microsoft Foundry 插件](/zh-CN/plugins/reference/microsoft-foundry) - Microsoft Foundry 聊天和 MAI 图像设置
- [MiniMax](/zh-CN/providers/minimax) - MiniMax 图像提供商设置
- [OpenAI](/zh-CN/providers/openai) - OpenAI Images 提供商设置
- [Vydra](/zh-CN/providers/vydra) - Vydra 图像、视频和语音设置
- [xAI](/zh-CN/providers/xai) - Grok 图像、视频、搜索、代码执行和 TTS 设置
- [配置参考](/zh-CN/gateway/config-agents#agent-defaults) - `imageGenerationModel` 配置
- [Models](/zh-CN/concepts/models) - 模型配置和故障转移
