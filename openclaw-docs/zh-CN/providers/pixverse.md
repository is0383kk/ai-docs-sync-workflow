---
read_when:
    - 你想在 OpenClaw 中使用 PixVerse 视频生成服务
    - 你需要设置 PixVerse API key/环境变量
    - 你想将 PixVerse 设为默认视频提供商
summary: OpenClaw 中的 PixVerse 视频生成设置
title: PixVerse
x-i18n:
    generated_at: "2026-07-26T06:31:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3dba881e877e3da4677a40dff736cb46de114337a1e0338ef8220dcd8e616f46
    source_path: providers/pixverse.md
    workflow: 16
---

OpenClaw 提供 `pixverse`，作为用于托管式 PixVerse 视频生成的官方外部插件。该插件按照 `videoGenerationProviders` 契约注册 `pixverse` 提供商。

| 属性               | 值                                                                   |
| ------------------ | -------------------------------------------------------------------- |
| 提供商 ID          | `pixverse`                                                   |
| 插件包             | `@openclaw/pixverse-provider`                                                   |
| 身份验证环境变量   | `PIXVERSE_API_KEY`                                                   |
| 新手引导标志       | `--auth-choice pixverse-api-key`                                                   |
| 直接 CLI 标志      | `--pixverse-api-key <key>`                                                   |
| API                | PixVerse Platform API v2（`video_id` 提交并轮询结果）        |
| 默认模型           | `pixverse/v6`                                                   |
| 默认 API 区域      | 国际版                                                               |

## 入门指南

<Steps>
  <Step title="安装插件">
    ```bash
    openclaw plugins install @openclaw/pixverse-provider
    openclaw gateway restart
    ```
  </Step>
  <Step title="设置 API 密钥">
    ```bash
    openclaw onboard --auth-choice pixverse-api-key
    ```

    向导会先提示选择国际版或中国版端点（参见下方的 API 区域），然后将 `region` 和 `baseUrl` 写入提供商配置。
    非交互式运行（密钥来自 `--pixverse-api-key` 或 `PIXVERSE_API_KEY`）
    默认使用国际版。

    如果尚未配置默认视频模型，新手引导还会将 `agents.defaults.mediaModels.video.primary` 设置为
    `pixverse/v6`。

  </Step>
  <Step title="切换现有的默认视频提供商（可选）">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "pixverse/v6"
    ```
  </Step>
  <Step title="生成视频">
    让智能体生成视频。系统会自动使用 PixVerse。
  </Step>
</Steps>

## 支持的模式和模型

该提供商通过 OpenClaw 的共享视频工具提供 PixVerse 生成模型。

| 模式         | 模型                                            | 参考输入                 |
| ------------ | ----------------------------------------------- | ------------------------ |
| 文生视频     | `v6`（默认）、`c1` | 无                       |
| 图生视频     | `v6`（默认）、`c1` | 1 张本地或远程图像       |

在发出图生视频请求前，本地图像引用会上传到 PixVerse。远程图像 URL 会作为 `image_url` 传递给 PixVerse 图像上传端点。

| 选项         | 支持的值                                                                                                                         |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| 时长         | 1-15 秒（默认 5 秒）                                                                                                            |
| 分辨率       | `360P`、`540P`、`720P`、`1080P`（默认 `540P`；`480P` 请求映射到 `540P`） |
| 宽高比       | `16:9`（默认）、`4:3`、`1:1`、`3:4`、`9:16`、`2:3`、`3:2`、`21:9`；仅适用于文生视频，图生视频沿用源图像的宽高比 |
| 生成音频     | `audio: true`                                                                                                               |

<Note>
PixVerse 图像模板生成功能尚未通过 `image_generate` 提供。该 API 由模板 ID 驱动，而 OpenClaw 的共享图像生成契约目前没有 PixVerse 专用的类型化选项包。
</Note>

## 提供商选项

视频提供商接受以下可选的提供商专用键：

| 选项                                 | 类型   | 效果                                              |
| ------------------------------------ | ------ | ------------------------------------------------- |
| `seed`                   | number | 确定性种子，范围为 0 到 2147483647                |
| `negativePrompt` / `negative_prompt` | string | 负面提示词                                    |
| `quality`                   | string | PixVerse 质量，例如 `720p`            |
| `motionMode` / `motion_mode` | string | 图生视频运动模式（默认 `normal`）   |
| `cameraMovement` / `camera_movement` | string | PixVerse 镜头运动预设                          |
| `templateId` / `template_id` | number | 已启用的 PixVerse 模板 ID                      |

## 配置

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "pixverse/v6",
      },
    },
  },
}
```

## 高级配置

<AccordionGroup>
  <Accordion title="API 区域">
    | 区域值          | PixVerse API 基础 URL                          |
    | --------------- | --------------------------------------------- |
    | `international` | `https://app-api.pixverse.ai/openapi/v2`                         |
    | `cn` | `https://app-api.pixverseai.cn/openapi/v2`                         |

    当密钥属于特定 PixVerse 平台区域时，请手动设置 `models.providers.pixverse.region`，或者运行
    `openclaw onboard --auth-choice pixverse-api-key`，在设置向导中选择区域：

    ```json5
    {
      models: {
        providers: {
          pixverse: {
            region: "cn", // "international" 或 "cn"
            baseUrl: "https://app-api.pixverseai.cn/openapi/v2",
            models: [],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="自定义基础 URL">
    仅当通过可信的兼容代理进行路由时，才设置 `models.providers.pixverse.baseUrl`。
    `baseUrl` 的优先级高于 `region`。

    ```json5
    {
      models: {
        providers: {
          pixverse: {
            baseUrl: "https://app-api.pixverse.ai/openapi/v2",
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="任务轮询">
    PixVerse 会从生成请求返回 `video_id`。OpenClaw 每隔 5 秒轮询
    `/openapi/v2/video/result/{video_id}`，直到任务成功、失败或超时（默认 5 分钟；可通过
    `agents.defaults.mediaModels.video.timeoutMs` 覆盖）。
  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="视频生成" href="/zh-CN/tools/video-generation" icon="video">
    共享工具参数、提供商选择和异步行为。
  </Card>
  <Card title="配置参考" href="/zh-CN/gateway/config-agents#agent-defaults" icon="gear">
    Agent 默认设置，包括视频生成模型。
  </Card>
</CardGroup>
