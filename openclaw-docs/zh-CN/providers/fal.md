---
read_when:
    - 你想在 OpenClaw 中使用 fal 图像生成
    - 你需要使用 `FAL_KEY` 身份验证流程
    - 你想为 image_generate、video_generate 或 music_generate 设置 fal 默认值
summary: OpenClaw 中的 fal 图像、视频和音乐生成设置
title: Fal
x-i18n:
    generated_at: "2026-07-26T05:59:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9bd868aaf6771f6fa38bb8e2a83133460d150e2a5aa9e5b888e221c07f29e0ad
    source_path: providers/fal.md
    workflow: 16
---

OpenClaw 内置了一个 `fal` 提供商，用于托管式图像、视频和音乐生成。

| 属性     | 值                                                                              |
| -------- | ------------------------------------------------------------------------------- |
| 提供商   | `fal`                                                              |
| 身份验证 | `FAL_KEY`（规范配置；`FAL_API_KEY` 也可作为回退方案）             |
| API      | fal 模型端点（`https://fal.run`；视频任务使用 `https://queue.fal.run`）             |
| 基础 URL | 使用 `models.providers.fal.baseUrl` 覆盖                                                    |

## 入门指南

<Steps>
  <Step title="设置 API key">
    ```bash
    openclaw onboard --auth-choice fal-api-key
    ```

    非交互式设置可以传入 `--fal-api-key <key>` 或导出 `FAL_KEY`。
    如果尚未配置默认图像模型，新手引导还会将 `fal/fal-ai/flux/dev` 设置为默认图像模型。

  </Step>
  <Step title="设置默认图像模型">
    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "fal/fal-ai/flux/dev",
          },
        },
      },
    }
    ```
  </Step>
</Steps>

## 图像生成

内置的 `fal` 图像生成提供商默认使用
`fal/fal-ai/flux/dev`。

| 能力       | 值                                                                 |
| ---------- | ------------------------------------------------------------------ |
| 最大图像数 | 每个请求 4 张；Krea 2：每个请求 1 张                               |
| 尺寸覆盖值 | `1024x1024`、`1024x1536`、`1536x1024`、`1024x1792`、`1792x1024` |
| 宽高比     | 除 Flux 图生图外均支持                                             |
| 分辨率     | `1K`、`2K`、`4K`（各模型限制见下文） |
| 输出格式   | `png`（默认）或 `jpeg`；Krea 2 拒绝 `outputFormat` 覆盖值 |

编辑请求（通过共享的 `image` / `images` 参数传入参考图像）
会路由到各模型对应的编辑端点，并应用各模型的参考图像数量限制：

| 模型系列                  | `fal/` 后的模型引用    | 编辑端点          | 最大参考图像数 |
| ------------------------- | ---------------------------------- | ----------------- | -------------- |
| Flux 和其他 fal 模型      | `fal-ai/flux/dev`（默认）         | `/image-to-image` | 1              |
| GPT Image                 | `openai/gpt-image-*`                 | `/edit` | 10             |
| Grok Imagine              | `xai/grok-imagine-image`                 | `/edit` | 3              |
| Nano Banana（旧版）       | `fal-ai/nano-banana`                 | `/edit` | 3              |
| Nano Banana 2             | `fal-ai/nano-banana-*`                 | `/edit` | 14             |
| Nano Banana 2 Lite        | `google/nano-banana-2-lite`                 | `/edit` | 14             |
| Krea 2                    | `krea/v2/{medium,large}/text-to-image`                 | 无（风格参考）    | 10 个风格参考  |

<Warning>
Flux 图生图请求**不**支持 `aspectRatio` 覆盖值。GPT
Image 和 Nano Banana 2 编辑请求使用 fal 的 `/edit` 端点，并接受
宽高比提示。Nano Banana 2 还接受额外的原生横向/纵向比例，
例如 `4:1`、`1:4`、`8:1` 和 `1:8`；Krea 2 会验证其自身范围更小的
宽高比子集。Grok Imagine 有自己的比例列表（包括 `2:1`、
`20:9`、`19.5:9` 及其倒数），并且仅接受 `1K`/`2K` 分辨率；
旧版 Nano Banana 和 Nano Banana 2 Lite 拒绝 `resolution` 覆盖值。
</Warning>

Krea 2 模型使用 fal 的原生 Krea 负载架构。OpenClaw 发送
`aspect_ratio`、`creativity` 和 `image_style_references`，而不是 Flux 使用的
通用 `image_size` / 编辑端点负载。模型引用如下：

- `fal/krea/v2/medium/text-to-image`
- `fal/krea/v2/large/text-to-image`

如需速度更快、表现力更强的插画、动漫、绘画和艺术
风格，请使用 Medium。如需速度较慢的照片级写实、原始纹理、胶片颗粒和精细
效果，请使用 Large。Krea 默认使用 `fal.creativity: "medium"`；支持的值为
`raw`、`low`、`medium` 和 `high`。

在 fal 的请求架构中，Krea 2 提供宽高比，而不提供 `image_size`。优先使用
`aspectRatio`；OpenClaw 会将 `size` 映射到最接近的受支持 Krea 宽高比，
并拒绝 Krea 的 `resolution`，而不是将其丢弃。

当需要从提供 `output_format` 的 fal 模型获取 PNG 输出时，请使用 `outputFormat: "png"`。
fal 未在 OpenClaw 中声明明确的透明背景
控制，因此对于 fal 模型，`background: "transparent"` 会被报告为已忽略的
覆盖值。
Krea 2 端点未通过 fal 公开 `output_format` 请求字段，因此
OpenClaw 会拒绝 Krea 请求的 `outputFormat` 覆盖值。

要使用 Krea 2 Medium：

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

## 视频生成

内置的 `fal` 视频生成提供商默认使用
`fal/fal-ai/minimax/video-01-live`。

| 能力     | 值                                                                |
| -------- | ----------------------------------------------------------------- |
| 模式     | 文本生成视频、单图参考、Seedance 参考生成视频                      |
| 运行时   | 针对长时间运行任务、由队列支持的提交/状态/结果流程                 |
| 超时     | 每个任务默认 20 分钟；每 5 秒轮询一次状态                          |

<AccordionGroup>
  <Accordion title="可用的视频模型">
    **MiniMax（默认）：**

    - `fal/fal-ai/minimax/video-01-live`

    **HeyGen video-agent：**

    - `fal/fal-ai/heygen/v2/video-agent`

    **Kling 和 Wan：**

    - `fal/fal-ai/kling-video/v2.1/master/text-to-video`
    - `fal/fal-ai/wan/v2.2-a14b/text-to-video`
    - `fal/fal-ai/wan/v2.2-a14b/image-to-video`

    **Seedance 2.0：**

    - `fal/bytedance/seedance-2.0/fast/text-to-video`
    - `fal/bytedance/seedance-2.0/fast/image-to-video`
    - `fal/bytedance/seedance-2.0/fast/reference-to-video`
    - `fal/bytedance/seedance-2.0/text-to-video`
    - `fal/bytedance/seedance-2.0/image-to-video`
    - `fal/bytedance/seedance-2.0/reference-to-video`

    MiniMax Live 和 HeyGen 请求仅发送提示词以及可选的
    单张参考图像；不会转发其他覆盖值。Seedance 模型
    接受 `aspectRatio`、`size`、`resolution`、4-15 秒的时长以及
    音频开关。

  </Accordion>

  <Accordion title="Seedance 2.0 配置示例">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/bytedance/seedance-2.0/fast/text-to-video",
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="Seedance 2.0 参考生成视频配置示例">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/bytedance/seedance-2.0/fast/reference-to-video",
          },
        },
      },
    }
    ```

    参考生成视频可通过共享的 `video_generate` `images`、`videos` 和 `audioRefs`
    参数接受最多 9 张图像、3 个视频和 3 个音频参考，
    参考文件总数不得超过 12 个。音频参考要求同一请求中
    至少包含一个图像或视频参考。

  </Accordion>

  <Accordion title="HeyGen video-agent 配置示例">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/fal-ai/heygen/v2/video-agent",
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## 音乐生成

内置的 `fal` 插件还为共享的 `music_generate` 工具注册了音乐生成提供商。

| 能力         | 值                                                                                                                       |
| ------------ | ------------------------------------------------------------------------------------------------------------------------ |
| 默认模型     | `fal/fal-ai/minimax-music/v2.6`                                                                                                      |
| 模型         | `fal-ai/minimax-music/v2.6`（mp3）、`fal-ai/ace-step/prompt-to-audio`（wav）、`fal-ai/stable-audio-25/text-to-audio`（wav）                                        |
| 最长时长     | 240 秒                                                                                                                   |
| 运行时       | 同步请求并下载生成的音频                                                                                                 |

将 fal 用作默认音乐提供商：

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "fal/fal-ai/minimax-music/v2.6",
      },
    },
  },
}
```

`fal-ai/minimax-music/v2.6` 支持显式歌词和纯音乐模式，
但不能在同一请求中同时使用。ACE-Step 和 Stable Audio 是
提示词生成音频端点；当需要使用这些模型系列时，请通过 `model` 覆盖值选择它们。
ACE-Step 拒绝显式歌词；Stable Audio 同时拒绝歌词和纯音乐模式。

<Tip>
上面的表格和折叠面板涵盖了内置 fal
提供商进行特殊处理的模型系列。仍可选择其他 fal 图像端点 ID 作为
图像模型；它们会按 Flux 方式处理（使用通用 `image_size` 负载，并通过 `/image-to-image`
传入一张参考图像）。
</Tip>

## 相关内容

<CardGroup cols={2}>
  <Card title="图像生成" href="/zh-CN/tools/image-generation" icon="image">
    共享的图像工具参数和提供商选择。
  </Card>
  <Card title="视频生成" href="/zh-CN/tools/video-generation" icon="video">
    共享的视频工具参数和提供商选择。
  </Card>
  <Card title="音乐生成" href="/zh-CN/tools/music-generation" icon="music">
    共享的音乐工具参数和提供商选择。
  </Card>
  <Card title="配置参考" href="/zh-CN/gateway/config-agents#agent-defaults" icon="gear">
    Agent 默认配置，包括图像、视频和音乐模型选择。
  </Card>
</CardGroup>
