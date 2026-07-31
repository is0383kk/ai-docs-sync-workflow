---
read_when:
    - 通过智能体生成音乐或音频
    - 配置音乐生成提供商和模型
    - 了解 `music_generate` 工具参数
sidebarTitle: Music generation
summary: 通过 music_generate 在 ComfyUI、fal、Google Lyria、MiniMax 和 OpenRouter 工作流中生成音乐
title: 音乐生成
x-i18n:
    generated_at: "2026-07-26T06:36:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f2a8a4a36e47839c7896046a556f7bf84f6c168492e2de46736635fe2a9358e
    source_path: tools/music-generation.md
    workflow: 16
---

`music_generate` 工具通过由 ComfyUI、fal、Google、MiniMax 和 OpenRouter 支持的共享音乐生成能力创建音乐或音频。

<Note>
仅当至少有一个音乐生成提供商可用时，`music_generate` 才会出现：存在显式的 `agents.defaults.mediaModels.music` 配置，或存在已配置身份验证的提供商（例如已设置 API key）。
</Note>

对于由会话支持的智能体运行，`music_generate` 会作为后台任务启动，在任务台账中跟踪进度，然后在曲目就绪时唤醒智能体，使其能够通知用户并附上完成的音频。完成处理智能体遵循会话的可见回复约定：配置后自动发送最终回复；如果会话要求使用消息工具，则使用 `message(action="send")`。如果请求方会话处于非活动状态或唤醒失败，并且回复中仍缺少已生成的音频，OpenClaw 会直接发送一条仅包含缺失音频的幂等回退消息。

## 快速开始

<Tabs>
  <Tab title="共享提供商支持">
    <Steps>
      <Step title="配置身份验证">
        为至少一个提供商设置 API key，例如 `GEMINI_API_KEY` 或 `MINIMAX_API_KEY`。
      </Step>
      <Step title="选择默认模型（可选）">
        ```json5
        {
          agents: {
            defaults: {
              musicGenerationModel: {
                primary: "google/lyria-3-clip-preview",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="向智能体提出请求">
        _“生成一首关于夜间驾车穿过霓虹都市的欢快合成器流行曲。”_

        智能体会自动调用 `music_generate`，无需加入工具允许列表。
      </Step>
    </Steps>

    如果没有由会话支持的智能体运行（直接/本地上下文），该工具会以内联方式运行，并在同一个工具结果中返回最终媒体路径。

  </Tab>
  <Tab title="ComfyUI 工作流">
    <Steps>
      <Step title="配置工作流">
        使用工作流 JSON 以及提示词/输出节点配置 `plugins.entries.comfy.config.music`。
      </Step>
      <Step title="云端身份验证（可选）">
        对于 Comfy Cloud，请设置 `COMFY_API_KEY` 或 `COMFY_CLOUD_API_KEY`。
      </Step>
      <Step title="调用工具">
        ```text
        /tool music_generate prompt="带有柔和磁带质感的温暖氛围合成器循环"
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

提示词示例：

```text
生成一首带有柔和弦乐、无人声的电影感钢琴曲。
```

```text
生成一段关于日出时发射火箭的动感芯片音乐循环。
```

使用 `action: "list"` 检查可用的提供商/模型，使用 `action: "status"` 检查当前由会话支持的音乐任务：

```text
/tool music_generate action=list
/tool music_generate action=status
```

直接生成示例：

```text
/tool music_generate prompt="带有黑胶质感和柔和雨声的梦幻 lo-fi 嘻哈音乐" instrumental=true
```

## 支持的提供商

| 提供商   | 默认模型                | 参考输入 | 支持的控制项                                    | 身份验证                                   |
| ---------- | ---------------------------- | ---------------- | ----------------------------------------------------- | -------------------------------------- |
| ComfyUI    | `workflow`                   | 最多 1 张图片    | 工作流定义的音乐或音频                       | `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY` |
| fal        | `fal-ai/minimax-music/v2.6`  | 无             | `lyrics`, `instrumental`, `durationSeconds`, `format` | `FAL_KEY` 或 `FAL_API_KEY`             |
| Google     | `lyria-3-clip-preview`       | 最多 10 张图片  | `lyrics`, `instrumental`, `format`                    | `GEMINI_API_KEY`, `GOOGLE_API_KEY`     |
| MiniMax    | `music-2.6`                  | 无             | `lyrics`, `instrumental`, `format`（仅 mp3）         | `MINIMAX_API_KEY` 或 MiniMax OAuth     |
| OpenRouter | `google/lyria-3-pro-preview` | 最多 1 张图片    | `lyrics`, `instrumental`, `durationSeconds`, `format` | `OPENROUTER_API_KEY`                   |

MiniMax 注册了两个共享相同模型的提供商 ID：`minimax` 用于 API key 身份验证，`minimax-portal` 用于 OAuth。模型引用遵循身份验证路径（`minimax/music-2.6` 与 `minimax-portal/music-2.6`）；请参阅 [MiniMax](/zh-CN/providers/minimax#music-generation)。

fal 除了其默认的 MiniMax 支持模型外，还提供 `fal-ai/ace-step/prompt-to-audio`（wav、无歌词、无纯器乐切换选项）和 `fal-ai/stable-audio-25/text-to-audio`（wav、仅提示词）。Google 的默认 `lyria-3-clip-preview` 仅输出 mp3；`lyria-3-pro-preview` 还支持 wav。MiniMax 还提供 `music-2.6-free`、`music-cover` 和 `music-cover-free`。OpenRouter 还提供 `google/lyria-3-clip-preview`。

### 能力矩阵

`music_generate`、约定测试和共享实时扫描使用的显式模式约定：

| 提供商   | `generate` | `edit` | 编辑限制 | 共享实时通道                                                         |
| ---------- | :--------: | :----: | ---------- | ------------------------------------------------------------------------- |
| ComfyUI    |     ✓      |   ✓    | 1 张图片    | 不在共享扫描中；由 `extensions/comfy/comfy.live.test.ts` 覆盖 |
| fal        |     ✓      |   —    | 无       | `generate`                                                                |
| Google     |     ✓      |   ✓    | 10 张图片  | `generate`, `edit`                                                        |
| MiniMax    |     ✓      |   —    | 无       | `generate`                                                                |
| OpenRouter |     ✓      |   ✓    | 1 张图片    | `generate`, `edit`                                                        |

## 工具参数

<ParamField path="prompt" type="string" required>
  音乐生成提示词。对于 `action: "generate"` 为必填项。
</ParamField>
<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  `"status"` 返回当前会话任务；`"list"` 检查提供商。
</ParamField>
<ParamField path="model" type="string">
  提供商/模型覆盖项（例如 `google/lyria-3-pro-preview`、`comfy/workflow`）。
</ParamField>
<ParamField path="lyrics" type="string">
  当提供商支持显式歌词输入时使用的可选歌词。
</ParamField>
<ParamField path="instrumental" type="boolean">
  当提供商支持时，请求仅生成纯器乐输出。
</ParamField>
<ParamField path="image" type="string">
  单个参考图片路径或 URL。
</ParamField>
<ParamField path="images" type="string[]">
  多个参考图片（支持的提供商最多允许 10 张）。
</ParamField>
<ParamField path="durationSeconds" type="number">
  当提供商支持时长提示时使用的目标时长（秒）。
</ParamField>
<ParamField path="format" type='"mp3" | "wav"'>
  当提供商支持时使用的输出格式提示。
</ParamField>
<ParamField path="filename" type="string">输出文件名提示。</ParamField>

<Note>
并非所有提供商都支持所有参数。OpenClaw 仍会在提交前验证输入数量等硬性限制。当提供商支持时长，但其最大值短于请求值时，OpenClaw 会将其限制为最接近的受支持时长。如果所选提供商或模型无法遵循确实不受支持的可选提示，这些提示会被忽略并产生警告。工具结果会报告已应用的设置；`details.normalization` 会记录从请求值到应用值的所有映射。
</Note>

提供商请求超时仅由操作员配置。配置后，OpenClaw 使用 `agents.defaults.mediaModels.music.timeoutMs`，将低于 120000ms 的值提高到 120000ms；否则，提供商请求默认超时为 300000ms。

## 异步行为

由会话支持的音乐生成会作为后台任务运行：

- **后台任务：**`music_generate` 会创建后台任务，立即返回已启动/任务响应，并稍后在智能体的后续消息中发布完成的曲目。
- **防止重复：**当任务处于 `queued` 或 `running` 状态时，同一会话中后续的 `music_generate` 调用会返回任务状态，而不是启动另一次生成。使用 `action: "status"` 进行显式检查。最近完成的相同请求也会在 2 分钟内去重。
- **状态查询：**`openclaw tasks list` 或 `openclaw tasks show <taskId>` 会检查已排队、运行中和终止状态。
- **完成唤醒：**OpenClaw 会将内部完成事件注入回同一会话，使模型能够自行编写面向用户的后续消息。
- **提示词提醒：**如果音乐任务已在进行中，同一会话中后续的用户/手动轮次会收到一条简短的运行时提示，使模型不会盲目地再次调用 `music_generate`。
- **无会话回退：**没有真实智能体会话的直接/本地上下文会以内联方式运行，并在同一轮次中返回最终音频结果。

### 任务生命周期

音乐任务会呈现与常规任务注册表相同的状态（完整状态机，包括 `timed_out`、`cancelled` 和 `lost`，请参阅[后台任务](/zh-CN/automation/tasks#task-lifecycle)）。大多数音乐运行会经历：

| 状态       | 含义                                                                                        |
| ----------- | ---------------------------------------------------------------------------------------------- |
| `queued`    | 任务已创建，正在等待提供商接受。                                           |
| `running`   | 提供商正在处理（通常为 30 秒到 3 分钟，具体取决于提供商和时长）。 |
| `succeeded` | 曲目已就绪；智能体会被唤醒并将其发布到对话中。                                 |
| `failed`    | 提供商错误或超时；智能体会被唤醒并收到错误详情。                                 |

从 CLI 检查状态：

```bash
openclaw tasks list
openclaw tasks show <taskId>
openclaw tasks cancel <taskId>
```

## 配置

### 模型选择

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
        fallbacks: ["fal/fal-ai/minimax-music/v2.6", "minimax/music-2.6"],
      },
    },
  },
}
```

### 提供商选择顺序

OpenClaw 按以下顺序尝试提供商：

1. 工具调用中的 `model` 参数（如果智能体指定）。
2. 配置中的 `musicGenerationModel.primary`。
3. 按顺序使用 `musicGenerationModel.fallbacks`。
4. 仅使用由身份验证支持的提供商默认值进行自动检测：
   - 如果当前默认文本模型提供商也提供音乐生成功能，则优先使用它；
   - 其余已注册的音乐生成提供商按提供商 ID 的字母顺序排列。

如果某个提供商失败，会自动尝试下一个候选提供商。如果全部失败，错误中会包含每次尝试的详细信息。

始终启用在已通过身份验证的提供商之间自动回退。每次调用的 `model` 仍具有最终决定权。

## 提供商说明

<AccordionGroup>
  <Accordion title="ComfyUI">
    由工作流驱动，并依赖所配置的图以及提示词/输出字段的节点映射。
    内置的 `comfy` 插件通过音乐生成提供商
    注册表接入共享的 `music_generate` 工具。
  </Accordion>
  <Accordion title="fal">
    通过共享的提供商身份验证路径使用 fal 模型端点。
    内置提供商默认使用 `fal-ai/minimax-music/v2.6`，并且还为
    提示词转音频请求提供 `fal-ai/ace-step/prompt-to-audio` 和
    `fal-ai/stable-audio-25/text-to-audio`。
    歌词和纯器乐模式仅适用于 MiniMax 模型；另外两个
    模型仅支持提示词。
  </Accordion>
  <Accordion title="Google (Lyria 3)">
    使用 Lyria 3 批量生成。当前内置流程支持
    提示词、可选歌词文本和可选参考图像。默认的
    `lyria-3-clip-preview` 模型仅输出 mp3；`lyria-3-pro-preview`
    模型还支持 wav。
  </Accordion>
  <Accordion title="MiniMax">
    使用批量 `music_generation` 端点。支持提示词、可选
    歌词、纯器乐模式和 mp3 输出，可使用 `minimax`
    API 密钥身份验证或 `minimax-portal` OAuth。还提供
    `music-2.6-free`、`music-cover` 和 `music-cover-free` 模型。
  </Accordion>
  <Accordion title="OpenRouter">
    使用已启用流式传输的 OpenRouter 聊天补全音频输出。
    内置提供商默认使用 `google/lyria-3-pro-preview`，并且还提供
    `openrouter/google/lyria-3-clip-preview`。
  </Accordion>
</AccordionGroup>

## 选择正确的路径

- 如果需要模型选择、提供商故障转移以及内置的异步任务/状态流程，请选择**共享提供商支持路径**。
- 如果需要自定义工作流图，或使用不属于共享内置音乐能力的提供商，请选择**插件路径（ComfyUI）**。

如果正在调试 ComfyUI 特有行为，请参阅
[ComfyUI](/zh-CN/providers/comfy)。如果正在调试共享提供商
行为，请从 [fal](/zh-CN/providers/fal)、[Google (Gemini)](/zh-CN/providers/google)、
[MiniMax](/zh-CN/providers/minimax) 或 [OpenRouter](/zh-CN/providers/openrouter) 开始。

## 提供商能力模式

共享音乐生成契约支持显式模式声明：

- `generate` 用于仅基于提示词的生成。
- 当请求包含一张或多张参考图像时，使用 `edit`。

新的提供商实现应优先使用显式模式块：

```typescript
capabilities: {
  generate: {
    maxTracks: 1,
    supportsLyrics: true,
    supportsFormat: true,
  },
  edit: {
    enabled: true,
    maxTracks: 1,
    maxInputImages: 1,
    supportsFormat: true,
  },
}
```

`maxInputImages`、`supportsLyrics` 和
`supportsFormat` 等旧版扁平字段**不足以**表明支持编辑。提供商
应显式声明 `generate` 和 `edit`，以便实时测试、契约
测试和共享的 `music_generate` 工具能够以确定性方式验证模式支持。

## 实时测试

共享内置提供商（fal、Google、MiniMax、
OpenRouter）的可选实时覆盖测试：

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts
```

等效的仓库封装命令，会运行同一个测试文件：

```bash
pnpm test:live:media:music
```

默认情况下，此实时测试文件会优先使用已导出的提供商环境变量，而不是存储的身份验证
配置文件；当提供商启用编辑模式时，它会同时运行 `generate` 和已声明的 `edit` 覆盖测试。当前覆盖范围：

- `google`：`generate` 加 `edit`
- `fal`：仅 `generate`
- `minimax`：仅 `generate`
- `openrouter`：`generate` 加 `edit`
- `comfy`：单独的 Comfy 实时覆盖测试，不属于共享提供商扫描

内置 ComfyUI 音乐路径的可选实时覆盖测试：

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

配置相应部分后，Comfy 实时测试文件还会覆盖 Comfy 图像和视频工作流。

## 相关内容

- [后台任务](/zh-CN/automation/tasks) — 跟踪分离式 `music_generate` 运行的任务
- [ComfyUI](/zh-CN/providers/comfy)
- [配置参考](/zh-CN/gateway/config-agents#agent-defaults) — `musicGenerationModel` 配置
- [Google (Gemini)](/zh-CN/providers/google)
- [MiniMax](/zh-CN/providers/minimax)
- [Models](/zh-CN/concepts/models) — 模型配置和故障转移
- [工具概览](/zh-CN/tools)
