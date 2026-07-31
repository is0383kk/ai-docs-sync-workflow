---
read_when:
    - 你想在 OpenClaw 中使用本地 ComfyUI 工作流
    - 你想将 Comfy Cloud 用于图像、视频或音乐工作流
    - 你需要内置 comfy 插件的配置键名
summary: OpenClaw 中的 ComfyUI 工作流图像、视频和音乐生成设置
title: ComfyUI
x-i18n:
    generated_at: "2026-07-26T06:20:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 74150d202a422de8e0f4b2b82d5d12bd42eb46991e8ef688832208e1a2ff7793
    source_path: providers/comfy.md
    workflow: 16
---

OpenClaw 内置了一个 `comfy` 插件，用于运行工作流驱动的 ComfyUI。该
插件完全由工作流驱动：OpenClaw 不会将通用的 `size`、
`aspectRatio`、`resolution`、`durationSeconds` 或 TTS 风格的控件映射到
你的图中。

| 属性         | 详情                                                                             |
| ------------ | -------------------------------------------------------------------------------- |
| 提供商       | `comfy`                                                               |
| 模型         | `comfy/workflow`                                                               |
| 共享工具     | `image_generate`、`video_generate`、`music_generate`                      |
| 身份验证     | 本地 ComfyUI 无需身份验证；Comfy Cloud 使用 `COMFY_API_KEY` 或 `COMFY_CLOUD_API_KEY` |
| API          | ComfyUI `/prompt` / `/history` / `/view`；Comfy Cloud `/api/*` |

## 支持的功能

- 使用工作流 JSON 生成和编辑图像（编辑需要 1 张上传的参考图像）
- 使用工作流 JSON 生成视频，支持文生视频或图生视频（1 张参考图像）
- 通过共享的 `music_generate` 工具生成音乐/音频，可选择提供 1 张参考图像
- 从已配置的节点下载输出；未配置节点时，从所有匹配的输出节点下载

## 入门指南

选择在自己的机器上运行 ComfyUI，或使用 Comfy Cloud。

<Tabs>
  <Tab title="本地">
    **最适合：** 在自己的机器或局域网上运行 ComfyUI 实例。

    <Steps>
      <Step title="在本地启动 ComfyUI">
        确保本地 ComfyUI 实例正在运行（默认为 `http://127.0.0.1:8188`）。
      </Step>
      <Step title="准备工作流 JSON">
        导出或创建 ComfyUI 工作流 JSON 文件。记下提示词输入节点以及希望 OpenClaw 读取的输出节点的节点 ID。
      </Step>
      <Step title="配置提供商">
        设置 `mode: "local"` 并指向你的工作流文件。最小图像配置示例：

        ```json5
        {
          plugins: {
            entries: {
              comfy: {
                config: {
                  mode: "local",
                  baseUrl: "http://127.0.0.1:8188",
                  image: {
                    workflowPath: "./workflows/flux-api.json",
                    promptNodeId: "6",
                    outputNodeId: "9",
                  },
                },
              },
            },
          },
        }
        ```
      </Step>
      <Step title="设置默认模型">
        将 OpenClaw 指向你所配置能力对应的 `comfy/workflow` 模型：

        ```json5
        {
          agents: {
            defaults: {
              imageGenerationModel: {
                primary: "comfy/workflow",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="验证">
        ```bash
        openclaw models list --provider comfy
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Comfy Cloud">
    **最适合：** 在 Comfy Cloud 上运行工作流，而无需管理本地 GPU 资源。

    <Steps>
      <Step title="获取 API key">
        在 [comfy.org](https://comfy.org) 注册，并从账户仪表板生成 API key。
      </Step>
      <Step title="设置 API key">
        通过以下任一方式提供你的密钥：

        ```bash
        # 新手引导标志
        openclaw onboard --comfy-api-key "your-key"

        # 环境变量（守护进程的首选方式）
        export COMFY_API_KEY="your-key"

        # 备用环境变量
        export COMFY_CLOUD_API_KEY="your-key"

        # 或在配置中内联设置
        openclaw config set plugins.entries.comfy.config.apiKey "your-key"
        ```
      </Step>
      <Step title="准备工作流 JSON">
        导出或创建 ComfyUI 工作流 JSON 文件。记下提示词输入节点和输出节点的节点 ID。
      </Step>
      <Step title="配置提供商">
        设置 `mode: "cloud"` 并指向你的工作流文件：

        ```json5
        {
          plugins: {
            entries: {
              comfy: {
                config: {
                  mode: "cloud",
                  image: {
                    workflowPath: "./workflows/flux-api.json",
                    promptNodeId: "6",
                    outputNodeId: "9",
                  },
                },
              },
            },
          },
        }
        ```

        <Tip>
        云模式默认将 `baseUrl` 设为 `https://cloud.comfy.org`。仅在使用自定义云端点时设置 `baseUrl`。
        </Tip>
      </Step>
      <Step title="设置默认模型">
        ```json5
        {
          agents: {
            defaults: {
              imageGenerationModel: {
                primary: "comfy/workflow",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="验证">
        ```bash
        openclaw models list --provider comfy
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## 配置

Comfy 支持共享的顶层连接设置，以及按能力划分的工作流部分（`image`、`video`、`music`）：

```json5
{
  plugins: {
    entries: {
      comfy: {
        config: {
          mode: "local",
          baseUrl: "http://127.0.0.1:8188",
          image: {
            workflowPath: "./workflows/flux-api.json",
            promptNodeId: "6",
            outputNodeId: "9",
          },
          video: {
            workflowPath: "./workflows/video-api.json",
            promptNodeId: "12",
            outputNodeId: "21",
          },
          music: {
            workflowPath: "./workflows/music-api.json",
            promptNodeId: "3",
            outputNodeId: "18",
          },
        },
      },
    },
  },
}
```

### 共享键

| 键                    | 类型                   | 说明                                                                                  |
| --------------------- | ---------------------- | ------------------------------------------------------------------------------------- |
| `mode`    | `"local"` 或 `"cloud"` | 连接模式。默认为 `"local"`。                                   |
| `baseUrl`    | 字符串                 | 本地模式默认为 `http://127.0.0.1:8188`，云模式默认为 `https://cloud.comfy.org`。     |
| `apiKey`    | 字符串                 | 可选的内联密钥，可替代 `COMFY_API_KEY` / `COMFY_CLOUD_API_KEY` 环境变量。 |
| `allowPrivateNetwork`    | 布尔值                 | 允许在云模式中使用私有/局域网 `baseUrl`，或使用本地私有 DNS FQDN。 |

<Note>
在 `local` 模式下，回环/私有 IP 字面量以及 `http://comfyui:8188` 等单标签服务名称无需 `allowPrivateNetwork` 即可工作。`https://comfy.local.example.com` 等外观类似公网域名的私有 DNS FQDN 需要 `allowPrivateNetwork: true`。对私有来源的信任范围仅限于已配置的协议、主机名和端口；本地重定向不能离开已配置的主机名，而指向公共 CDN 的云端重定向会使用默认 SSRF 策略进行检查。
</Note>

### 按能力配置的键

以下键适用于 `image`、`video` 或 `music` 部分：

| 键                           | 必需 | 默认值   | 说明                                                                        |
| ---------------------------- | ---- | -------- | --------------------------------------------------------------------------- |
| `workflow` 或 `workflowPath` | 是 | --       | 内联工作流 JSON，或 ComfyUI 工作流 JSON 文件的路径。                       |
| `promptNodeId`           | 是   | --       | 接收文本提示词的节点 ID。                                                   |
| `promptInputName`           | 否   | `"text"` | 提示词节点上的输入名称。                                       |
| `outputNodeId`           | 否   | --       | 要从中读取输出的节点 ID。省略时，将使用所有匹配的输出节点。                |
| `pollIntervalMs`           | 否   | `1500` | 等待任务完成的轮询间隔，以毫秒为单位。                         |
| `timeoutMs`           | 否   | `300000` | 工作流运行的超时时间，以毫秒为单位。                           |

`image` 和 `video` 部分还支持参考图像输入节点：

| 键                    | 必需                         | 默认值   | 说明                               |
| --------------------- | ---------------------------- | -------- | ---------------------------------- |
| `inputImageNodeId`    | 是（传入参考图像时）         | --       | 接收已上传参考图像的节点 ID。      |
| `inputImageInputName`    | 否                           | `"image"` | 图像节点上的输入名称。 |

`apiKey` 接受字面字符串或[密钥引用](/zh-CN/gateway/configuration-reference#secrets)对象。

## 工作流详情

<AccordionGroup>
  <Accordion title="图像工作流">
    将默认图像模型设置为 `comfy/workflow`：

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "comfy/workflow",
          },
        },
      },
    }
    ```

    **参考图像编辑示例：**

    要使用上传的参考图像启用图像编辑，请将 `inputImageNodeId` 添加到图像配置中：

    ```json5
    {
      plugins: {
        entries: {
          comfy: {
            config: {
              image: {
                workflowPath: "./workflows/edit-api.json",
                promptNodeId: "6",
                inputImageNodeId: "7",
                inputImageInputName: "image",
                outputNodeId: "9",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="视频工作流">
    将默认视频模型设置为 `comfy/workflow`：

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "comfy/workflow",
          },
        },
      },
    }
    ```

    Comfy 视频工作流通过已配置的图支持文生视频和图生视频。

    <Note>
    OpenClaw 不会将输入视频传入 Comfy 工作流。仅支持将文本提示词和单张参考图像作为输入。
    </Note>

  </Accordion>

  <Accordion title="音乐工作流">
    内置插件为工作流定义的音频或音乐输出注册了音乐生成提供商，并通过共享的 `music_generate` 工具提供。它可以接受一张可选的参考图像（最多 1 张）：

    ```text
    /tool music_generate prompt="带有柔和磁带质感的温暖氛围合成器循环"
    ```

    使用 `music` 配置部分指向你的音频工作流 JSON 和输出节点。

  </Accordion>

  <Accordion title="向后兼容性">
    现有的顶层图像配置（不含嵌套的 `image` 部分）仍然有效：

    ```json5
    {
      plugins: {
        entries: {
          comfy: {
            config: {
              workflowPath: "./workflows/flux-api.json",
              promptNodeId: "6",
              outputNodeId: "9",
            },
          },
        },
      },
    }
    ```

    OpenClaw 将该旧版结构视为图像工作流配置。你无需立即迁移，但对于新设置，建议使用嵌套的 `image` / `video` / `music` 部分。如果你只使用图像生成，则旧版扁平配置与新的嵌套 `image` 部分在功能上等效。

  </Accordion>

  <Accordion title="实时测试">
    内置插件提供可选择启用的实时测试覆盖：

    ```bash
    OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
    ```

    除非配置了对应的 Comfy 工作流部分，否则实时测试会跳过各个图像、视频或音乐测试用例。

  </Accordion>
</AccordionGroup>

## 相关内容

<CardGroup cols={2}>
  <Card title="图像生成" href="/zh-CN/tools/image-generation" icon="image">
    图像生成工具的配置和用法。
  </Card>
  <Card title="视频生成" href="/zh-CN/tools/video-generation" icon="video">
    视频生成工具的配置和用法。
  </Card>
  <Card title="音乐生成" href="/zh-CN/tools/music-generation" icon="music">
    音乐和音频生成工具的设置。
  </Card>
  <Card title="提供商目录" href="/zh-CN/providers/index" icon="layers">
    所有提供商和模型引用的概览。
  </Card>
  <Card title="配置参考" href="/zh-CN/gateway/config-agents#agent-defaults" icon="gear">
    完整的配置参考，包括智能体默认值。
  </Card>
</CardGroup>
