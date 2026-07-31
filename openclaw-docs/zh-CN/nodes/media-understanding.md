---
read_when:
    - 设计或重构媒体理解功能
    - 调整入站音频、视频和图像预处理
sidebarTitle: Media understanding
summary: 入站图像/音频/视频理解（可选），支持提供商和 CLI 回退方案
title: 媒体理解
x-i18n:
    generated_at: "2026-07-26T06:52:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 38e9a0f89607bb9c4af85689ef0fbd3df9234b36e06d86c129e0d823d6e05143
    source_path: nodes/media-understanding.md
    workflow: 16
---

OpenClaw 可在回复管线运行前汇总入站媒体（图像/音频/视频），因此命令解析和路由处理的是简短文本，而非原始字节。媒体理解会自动检测本地工具或提供商密钥，你也可以配置明确的模型。原始媒体始终会像往常一样传递给模型；当媒体理解失败或被禁用时，回复流程将保持不变并继续运行。

供应商插件会注册能力元数据（哪个提供商支持哪种媒体类型、默认模型和优先级）。OpenClaw 核心负责共享的 `tools.media` 配置、回退顺序和回复管线集成。

## 工作原理

<Steps>
  <Step title="收集附件">
    收集有序的入站媒体信息（`path`、`url`、`contentType` 和 `kind`）。
  </Step>
  <Step title="按能力选择">
    对于每项已启用的能力（图像/音频/视频），根据 `attachments` 策略选择附件（默认：仅选择第一个附件）。
  </Step>
  <Step title="选择模型">
    选择第一个符合条件的模型条目（大小、能力和身份验证均可用）。
  </Step>
  <Step title="失败时回退">
    如果模型出错、超时或媒体超过 `maxBytes`，则尝试下一个条目。
  </Step>
  <Step title="成功时应用">
    `Body` 将成为 `[Image]`、`[Audio]` 或 `[Video]` 块。音频还会设置 `{{Transcript}}`；存在说明文字时，命令解析会使用说明文字，否则使用转录文本。说明文字会在块内保留为 `User text:`。
  </Step>
</Steps>

## 配置

`tools.media` 包含一个带能力标签的模型列表，以及少量按能力设置的控制项：

```json5
{
  tools: {
    media: {
      concurrency: 2, // 最大并发能力运行数（默认值）
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["image", "video"] },
      ],
      image: { preferredModel: "google/gemini-3-flash-preview" },
      audio: { enabled: true },
      video: { enabled: true },
    },
  },
}
```

按能力（`image`/`audio`/`video`）设置的键：

| 键              | 类型      | 默认值                                | 说明                                                                |
| ---------------- | --------- | -------------------------------------- | -------------------------------------------------------------------- |
| `enabled`        | `boolean` | 自动（`false` 表示禁用）                | 设置 `false` 可关闭此能力的自动检测              |
| `preferredModel` | `string`  | 第一个兼容条目                 | 优先选择 `provider/model`、模型 ID、`provider:<id>` 或 `cli:command` |
| `prompt`         | `string`  | 能力默认值                     | 条目未覆盖时使用的默认提示词                    |
| `maxChars`       | `number`  | 图像/视频为 `500`，音频未设置         | 默认输出限制                                                 |
| `maxBytes`       | `number`  | 图像 10MB、音频 20MB、视频 50MB     | 默认输入限制                                                  |
| `timeoutSeconds` | `number`  | 图像/音频为 `60`，视频为 `120`          | 默认请求超时时间                                              |
| `language`       | `string`  | 未设置                                  | 音频转录提示                                             |
| `scope`          | 对象    | 未设置                                  | 按渠道/聊天类型/来源键设限                                 |
| `attachments`    | 对象    | `{ mode: "first", maxAttachments: 1 }` | 选择要处理的匹配附件                      |
| `echoTranscript` | `boolean` | `false`                                | 仅限音频：在智能体处理前回显转录文本              |
| `echoFormat`     | `string`  | `'📝 "{transcript}"'`                  | 仅限音频：回显转录文本的格式                         |

提示词、限制、语言提示、请求覆盖项和提供商选项既可设为能力默认值，也可在各个 `tools.media.models[]` 条目中覆盖。未配置明确模型时，能力默认值也适用于自动检测到的提供商。

### 模型条目

每个 `models[]` 条目都是**提供商**条目（默认）或 **CLI** 条目：

<Tabs>
  <Tab title="提供商条目">
    ```json5
    {
      type: "provider", // 省略时的默认值
      provider: "openai",
      model: "gpt-5.6-sol",
      prompt: "用不超过 500 个字符描述图像。",
      maxChars: 500,
      maxBytes: 10485760,
      timeoutSeconds: 60,
      capabilities: ["image"],
      profile: "vision-profile",
      preferredProfile: "vision-fallback",
    }
    ```
  </Tab>
  <Tab title="CLI 条目">
    ```json5
    {
      type: "cli",
      command: "gemini",
      args: [
        "-m",
        "gemini-3-flash",
        "--allowed-tools",
        "read_file",
        "读取 {{AttachmentPath}} 处的媒体，并用不超过 {{MaxChars}} 个字符进行描述。",
      ],
      maxChars: 500,
      maxBytes: 52428800,
      timeoutSeconds: 120,
      capabilities: ["video", "image"],
    }
    ```

    CLI 模板还可以使用 `{{AttachmentUrl}}`、`{{AttachmentContentType}}`、`{{AttachmentDir}}`、`{{AttachmentIndex}}`、`{{OutputDir}}`（为本次运行创建的临时目录）和 `{{OutputBase}}`（临时文件的基础路径，不含扩展名）。较旧的 `{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}` 和 `{{MediaDir}}` 名称仍作为已弃用的兼容别名保留。

  </Tab>
</Tabs>

### 提供商凭据

提供商媒体理解使用与普通模型调用相同的身份验证解析顺序：身份验证配置文件、环境变量，然后是 `models.providers.<providerId>.apiKey`。`tools.media.models[]` 条目不接受内联的 `apiKey` 字段。

```json5
{
  models: {
    providers: {
      openai: { apiKey: "<OPENAI_API_KEY>" },
      moonshot: { apiKey: "<MOONSHOT_API_KEY>" },
    },
  },
}
```

有关配置文件、环境变量和自定义基础 URL，请参阅[工具和自定义提供商](/zh-CN/gateway/config-tools)。

## 规则和行为

- 超过 `maxBytes` 的媒体会跳过该模型并尝试下一个模型。
- 小于 1024 字节的音频文件会被视为空文件或损坏文件，并在转录前跳过；智能体会收到确定性的占位转录文本。
- 如果当前主图像模型原生支持视觉能力，OpenClaw 会跳过 `[Image]` 摘要块，直接将原始图像传入模型。MiniMax 是一个例外：`minimax`、`minimax-cn`、`minimax-portal` 和 `minimax-portal-cn` 始终通过插件所有的 `MiniMax-VL-01` 媒体提供商路由图像理解，即使旧版 MiniMax M2.x 聊天元数据声称支持图像输入（只有 `MiniMax-M3` 及更高版本才被视为原生支持视觉能力）。
- 如果 Gateway 网关/WebChat 主模型仅支持文本，图像附件会保留为已卸载的 `media://inbound/*` 引用，以便图像/PDF 工具或已配置的图像模型仍可检查它们，而不会丢失附件。
- 明确设置的 `openclaw infer image describe --file <path> --model <provider/model>`（别名：`openclaw capability image describe`）会直接运行该支持图像的提供商/模型，包括 `ollama/qwen2.5vl:7b` 等 Ollama 引用，前提是在 `models.providers.ollama.models[]` 下配置了匹配且支持图像的模型。
- 如果 `<capability>.enabled` 不是 `false`，但未配置模型，则当当前回复模型的提供商支持相应能力时，OpenClaw 会尝试使用该模型。

### 自动检测（默认）

当 `tools.media.<capability>.enabled` 不是 `false` 且未配置模型时，OpenClaw 会按以下顺序尝试，并在遇到第一个可用选项时停止：

<Steps>
  <Step title="已配置的图像模型（仅限图像）">
    `agents.defaults.imageModel` 主引用/回退引用，但当前回复模型已原生支持视觉能力时除外。优先选择 `provider/model` 引用；只有在匹配结果唯一时，才会根据已配置的支持图像的提供商模型条目补全裸引用。
  </Step>
  <Step title="当前回复模型">
    当前回复模型，前提是其提供商支持相应能力。
  </Step>
  <Step title="提供商身份验证（仅限音频，位于本地 CLI 之前）">
    支持音频的已配置 `models.providers.*` 条目会在本地 CLI 之前尝试。内置提供商优先级顺序（优先级相同时按提供商 ID 的字母顺序决定）：Groq/OpenAI &rarr; xAI &rarr; Deepgram &rarr; OpenRouter &rarr; Google/SenseAudio &rarr; Deepinfra/ElevenLabs &rarr; Mistral。
  </Step>
  <Step title="本地 CLI（仅限音频）">
    已就绪的本地二进制文件会组成有序回退列表：
    - 仅当当前进程中先前的模型调用观察到 Metal 或 CUDA 后，才首先使用 `whisper-cli`
    - 默认使用 CPU 的 `sherpa-onnx-offline`（需要带有 `tokens.txt`/`encoder.onnx`/`decoder.onnx`/`joiner.onnx` 的 `SHERPA_ONNX_MODEL_DIR`）
    - 当加速仅为构建能力或尚未观察到时，使用 `whisper-cli`
    - 在 Apple Silicon 上使用 `parakeet-mlx`（支持 MLX，但尚未观察到设备使用情况）
    - `whisper`（Python CLI；默认使用 `turbo` 模型，并自动下载）

    后端能力检查会被缓存，且不会加载模型。构建能力、请求的后端标志和实际调用中观察到的后端相互独立。自动检测到的 whisper.cpp 会保持模型运行日志处于启用状态，以便记录上游选择后端的日志行。明确配置的 CLI 条目会保留其配置顺序、后端标志和输出标志。

  </Step>
  <Step title="提供商身份验证（图像/视频）">
    支持相应能力的已配置 `models.providers.*` 条目会在内置回退顺序之前尝试。仅用于图像配置且拥有支持图像模型的提供商会自动注册用于媒体理解，即使它们不是内置供应商插件。

    内置提供商优先级顺序（优先级相同时按提供商 ID 的字母顺序决定）：
    - 图像：Anthropic/OpenAI &rarr; Google &rarr; MiniMax &rarr; Deepinfra &rarr; MiniMax Portal &rarr; Z.AI
    - 视频：Google &rarr; Qwen &rarr; Moonshot

  </Step>
  <Step title="Antigravity CLI（仅限图像/视频）">
    使用第一个已安装的 `agy` 或 `antigravity` 二进制文件（可通过 `OPENCLAW_ANTIGRAVITY_CLI` 覆盖），并将其沙箱隔离范围限制在媒体所在目录。
  </Step>
</Steps>

要禁用某项能力的自动检测：

```json5
{
  tools: {
    media: {
      audio: {
        enabled: false,
      },
    },
  },
}
```

<Note>
在 macOS/Linux/Windows 上，二进制文件检测均为尽力而为；请确保 CLI 位于 `PATH` 中（会展开 `~`），或使用完整命令路径设置明确的 CLI 模型条目。
</Note>

### 代理支持（音频/视频提供商调用）

基于提供商的**音频**和**视频**理解遵循标准的出站代理环境变量，包括 `NO_PROXY`/`no_proxy` 绕过规则：`HTTPS_PROXY`、`HTTP_PROXY`、`ALL_PROXY`、`https_proxy`、`http_proxy`、`all_proxy`。小写变量的优先级高于大写变量。如果均未设置，媒体理解会直接访问外部网络；如果代理值格式错误，OpenClaw 会记录警告并回退到直接获取。图像理解不经过此代理路径。

## 能力

在 `models[]` 条目上设置 `capabilities`，可将其限制为特定媒体类型。对于共享列表，OpenClaw 会按内置提供商推断默认值：

| 提供商                                                                 | 能力                  |
| ------------------------------------------------------------------------ | --------------------- |
| `openai`, `anthropic`, `minimax`                                         | 图像                 |
| `minimax-portal`                                                         | 图像                 |
| `moonshot`                                                               | 图像 + 视频         |
| `openrouter`                                                             | 图像 + 音频         |
| `google`（Gemini API）                                                    | 图像 + 音频 + 视频 |
| `qwen`                                                                   | 图像 + 视频         |
| `deepinfra`                                                              | 图像 + 音频         |
| `mistral`                                                                | 音频                 |
| `zai`                                                                    | 图像                 |
| `groq`, `xai`, `deepgram`, `senseaudio`                                  | 音频                 |
| 任何包含支持图像模型的 `models.providers.<id>.models[]` 目录 | 图像                 |

对于 CLI 条目，请显式设置 `capabilities`，以避免意外匹配；如果省略，该条目将适用于它出现的每个能力列表。

## 提供商支持矩阵

| 能力 | 提供商                                                                                                                                               | 说明                                                                                                                                                                                   |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 图像      | Anthropic、Codex app-server、Deepinfra、Google、MiniMax、MiniMax Portal、Moonshot、OpenAI、OpenAI Codex OAuth、OpenRouter、Qwen、Z.AI、配置提供商 | 供应商插件注册图像支持；`openai/*` 可使用 API 密钥或 Codex OAuth 路由；`codex/*` 使用有界的 Codex app-server 轮次；支持图像的配置提供商会自动注册。 |
| 音频      | Deepgram、Deepinfra、ElevenLabs、Google、Groq、Mistral、OpenAI、OpenRouter、SenseAudio、xAI                                                             | 提供商转录（Whisper/Groq/xAI/Deepgram/OpenRouter STT/Gemini/SenseAudio/Scribe/Voxtral）。                                                                                     |
| 视频      | Google、Moonshot、Qwen                                                                                                                                  | 通过供应商插件实现提供商视频理解；Qwen 视频理解使用标准 DashScope 端点。                                                                        |

<Note>
**MiniMax 说明**：`minimax`、`minimax-cn`、`minimax-portal` 和 `minimax-portal-cn` 的图像理解始终由插件自有的 `MiniMax-VL-01` 媒体提供商提供，即使旧版 MiniMax M2.x 聊天元数据声称支持图像输入。
</Note>

## 模型选择指南

- 当质量和安全性至关重要时，请为每种媒体能力优先选择最强的当前一代模型。
- 对于处理不受信任输入且启用了工具的智能体，请避免使用较旧或较弱的媒体模型。
- 为确保可用性，请为每种能力至少保留一个后备模型（高质量模型 + 更快或更便宜的模型）。
- 当提供商 API 不可用时，CLI 后备方案（`whisper-cli`、`whisper`、`gemini`）可提供帮助。
- 已知的文件输出模式具有权威性：推断出的转录文件为空或缺失时，不会生成转录，而不会回退到 CLI 进度输出。
- `parakeet-mlx`：将 `--output-format txt`（或 `all`）与 `--output-dir` 和默认的 `{filename}` 输出模板配合使用。上游的 `PARAKEET_OUTPUT_FORMAT` 和 `PARAKEET_OUTPUT_TEMPLATE` 环境变量也会生效。OpenClaw 读取 `<output-dir>/<media-basename>.txt`；默认的 `srt` 格式、其他格式和自定义输出模板仍使用 stdout。

## 附件策略

每种能力的 `attachments` 控制处理哪些附件：

<ParamField path="mode" type='"first" | "all"' default="first">
  仅处理第一个选中的附件，或处理所有附件。
</ParamField>
<ParamField path="maxAttachments" type="number" default="1">
  限制处理的附件数量。
</ParamField>
<ParamField path="prefer" type='"first" | "last" | "path" | "url"'>
  候选附件之间的选择偏好。
</ParamField>

当 `mode: "all"` 时，输出会标记为 `[Image 1/2]`、`[Audio 2/2]` 等。

### 文件附件提取

- 提取的文件文本在附加到媒体提示词之前，会包装为不受信任的外部内容，并使用 `<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>` 等边界标记以及一行 `Source: External` 元数据。
- 此路径有意省略较长的 `SECURITY NOTICE:` 横幅，以保持媒体提示词简短；边界标记和元数据仍然适用。
- 没有可提取文本的文件会获得 `[No extractable text]`。
- 如果 PDF 回退到渲染后的页面图像，OpenClaw 会将这些图像转发给支持视觉能力的回复模型，并在文件块中保留占位符 `[PDF content rendered to images]`。

## 配置示例

<Tabs>
  <Tab title="共享模型 + 覆盖">
    ```json5
    {
      tools: {
        media: {
          models: [
            { provider: "openai", model: "gpt-5.6-sol", capabilities: ["image"] },
            {
              provider: "google",
              model: "gemini-3-flash-preview",
              capabilities: ["image", "audio", "video"],
            },
            {
              type: "cli",
              command: "gemini",
              args: [
                "-m",
                "gemini-3-flash",
                "--allowed-tools",
                "read_file",
                "读取 {{AttachmentPath}} 处的媒体，并用不超过 {{MaxChars}} 个字符描述它。",
              ],
              capabilities: ["image", "video"],
            },
          ],
          audio: {
            attachments: { mode: "all", maxAttachments: 2 },
          },
          video: {
            maxChars: 500,
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="仅音频 + 视频">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [
              { provider: "openai", model: "gpt-4o-mini-transcribe" },
              {
                type: "cli",
                command: "whisper",
                args: ["--model", "base", "{{AttachmentPath}}"],
              },
            ],
          },
          video: {
            enabled: true,
            maxChars: 500,
            models: [
              { provider: "google", model: "gemini-3-flash-preview" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "读取 {{AttachmentPath}} 处的媒体，并用不超过 {{MaxChars}} 个字符描述它。",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="仅图像">
    ```json5
    {
      tools: {
        media: {
          image: {
            enabled: true,
            maxBytes: 10485760,
            maxChars: 500,
            models: [
              { provider: "openai", model: "gpt-5.6-sol" },
              { provider: "anthropic", model: "claude-opus-5" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "读取 {{AttachmentPath}} 处的媒体，并用不超过 {{MaxChars}} 个字符描述它。",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="单个多模态条目">
    ```json5
    {
      tools: {
        media: {
          image: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          audio: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          video: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## 状态输出

运行媒体理解时，`/status` 会包含一行按能力汇总的信息：

```
📎 媒体：图像正常（openai/gpt-5.6-sol）· 音频正常（whisper-cli 检测到=metal）
```

如需预检清单，请运行 `openclaw capability audio providers`。本地行会将本地后备优胜项与全局提供商选择、就绪状态以及独立的可用/请求/检测后端字段分别显示。同一本地选择也可作为信息性 Doctor 发现获取：

```bash
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

## 注意事项

- 理解以尽力而为为原则。错误不会阻止回复。
- 即使禁用了理解功能，附件仍会传递给模型。
- 使用 `scope` 限制运行理解功能的位置（例如，仅限私信）。

## 相关内容

- [配置](/zh-CN/gateway/configuration)
- [图像与媒体支持](/zh-CN/nodes/images)
