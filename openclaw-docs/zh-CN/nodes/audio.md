---
read_when:
    - 更改音频转写或媒体处理
summary: 入站音频/语音留言如何下载、转录并注入回复中
title: 音频和语音留言
x-i18n:
    generated_at: "2026-07-26T05:52:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4076e3e55eb5c6dcc94cfdd842619697c8d756b924956d7b266d18446b4dd9be
    source_path: nodes/audio.md
    workflow: 16
---

## 功能说明

启用（或自动检测到）音频理解后，OpenClaw 会：

1. 查找第一个音频附件（本地路径或 URL），并在需要时下载。
2. 在发送给每个模型条目之前强制执行 `maxBytes`。
3. 按顺序运行第一个符合条件的模型条目（提供商或 CLI）；如果某个条目失败或被跳过（大小/超时），则尝试下一个条目。
4. 成功后，将 `Body` 替换为 `[Audio]` 块，并设置 `{{Transcript}}`。

转录成功后，`CommandBody`/`RawBody` 也会设置为转录文本，以便斜杠命令仍可正常工作。使用 `--verbose` 时，日志会显示转录何时运行以及何时替换正文。

## 自动检测（默认）

如果你尚未配置模型，并且 `tools.media.audio.enabled` 不是 `false`，OpenClaw 会按以下顺序自动检测，并在找到第一个可用选项时停止：

1. **当前回复模型**，前提是其提供商支持音频理解。
2. **已配置的提供商身份验证** — 任何已有可用身份验证且提供商支持音频转录的 `models.providers.*` 条目。此项会在本地 CLI 之前检查，因此已配置的 API key 始终优先于 `PATH` 上的本地二进制文件。
   配置了多个提供商时的优先级：Groq、OpenAI、xAI、Deepgram、Google、SenseAudio、ElevenLabs、Mistral。
3. **本地 CLI**（仅当未解析到提供商身份验证时）。OpenClaw 会构建一个有序的回退列表：
   - `whisper-cli`，仅当当前进程中较早的模型调用观察到 Metal 或 CUDA 时，才排在 CPU 默认选项之前
   - `sherpa-onnx-offline`，使用其默认 CPU 提供商（需要包含 `tokens.txt`、`encoder.onnx`、`decoder.onnx` 和 `joiner.onnx` 的 `SHERPA_ONNX_MODEL_DIR`）
   - `whisper-cli`，当 Metal/CUDA 仅具备构建能力，或所选后端尚未通过其他方式观察到时
   - `parakeet-mlx`，用于 Apple Silicon（支持 MLX；设备使用情况仍未观察到）
   - `whisper`（Python CLI；自动下载模型）

安装/链接来源可作为能力证据，而非执行证据。仅凭这一点绝不会将候选项排到 CPU sherpa 之前。OpenClaw 不会仅为了探测后端而在设置或状态检查期间加载模型。
自动检测到的 whisper.cpp 会保持其常规模型运行日志处于启用状态，以便 OpenClaw 记录上游的 `using … backend` 行。显式 CLI 条目会保留其配置的输出标志。

用于媒体理解的 Gemini CLI 自动检测已替换为沙箱隔离的 Antigravity CLI（`agy`），作为图像/视频的回退选项；除上述本地二进制文件外，音频不使用 CLI 回退。

要禁用自动检测，请设置 `tools.media.audio.enabled: false`。要进行自定义，请向 `tools.media.models` 添加带能力标签的条目。

<Note>
二进制文件检测在 macOS/Linux/Windows 上均为尽力而为。请确保 CLI 位于 `PATH` 中（会展开 `~`），或使用完整命令路径设置显式 CLI 模型。
</Note>

无需转录音频即可检查本地选择：

```bash
openclaw capability audio providers
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

提供商清单会将本地回退选用项与全局提供商选择分别报告，并列出具备能力、已请求和已观察到的后端字段。转录运行后，`/status` 会在媒体行中报告已请求或已观察到的后端。显式支持音频的 `tools.media.models` CLI 条目仍会绕过自动选择；请使用其后端专用标志，例如 sherpa 的 `--provider=cuda` 或 whisper.cpp 的 `--no-gpu`/`--device`。

## 配置示例

### 提供商 + CLI 回退（OpenAI + Whisper CLI）

```json5
{
  tools: {
    media: {
      models: [
        { provider: "openai", model: "gpt-4o-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          timeoutSeconds: 45,
          capabilities: ["audio"],
        },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-transcribe" },
    },
  },
}
```

### 仅提供商（Deepgram）

```json5
{
  tools: {
    media: {
      models: [{ provider: "deepgram", model: "nova-3", capabilities: ["audio"] }],
      audio: { enabled: true },
    },
  },
}
```

### 仅提供商（Mistral Voxtral）

```json5
{
  tools: {
    media: {
      models: [{ provider: "mistral", model: "voxtral-mini-latest", capabilities: ["audio"] }],
      audio: { enabled: true },
    },
  },
}
```

### 仅提供商（SenseAudio）

```json5
{
  tools: {
    media: {
      models: [
        {
          provider: "senseaudio",
          model: "senseaudio-asr-pro-1.5-260319",
          capabilities: ["audio"],
        },
      ],
      audio: { enabled: true },
    },
  },
}
```

### 将转录文本回显到聊天（选择启用）

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        echoTranscript: true,
        echoFormat: '📝 "{transcript}"',
      },
    },
  },
}
```

## 注意事项和限制

- 提供商身份验证遵循标准模型身份验证顺序（身份验证配置文件、环境变量、`models.providers.*.apiKey`）。
- Groq 设置详情：[Groq](/zh-CN/providers/groq)。
- 使用 `provider: "deepgram"` 时，Deepgram 会读取 `DEEPGRAM_API_KEY`。设置详情：[Deepgram](/zh-CN/providers/deepgram)。
- Mistral 设置详情：[Mistral](/zh-CN/providers/mistral)。
- 使用 `provider: "senseaudio"` 时，SenseAudio 会读取 `SENSEAUDIO_API_KEY`。设置详情：[SenseAudio](/zh-CN/providers/senseaudio)。
- 音频提供商可以使用 `tools.media.audio` 下的默认值，也可以在其 `tools.media.models[]` 条目中覆盖 `baseUrl`、`headers`、`providerOptions` 和限制。
- 内置音频大小上限为 20MB。条目级 `maxBytes` 覆盖可以更改此值；对于该模型，超出大小限制的音频会被跳过，并尝试下一个条目。
- 小于 1024 字节的音频文件会在提供商/CLI 转录前被跳过。
- 音频的默认 `maxChars` 为**未设置**（完整转录文本）。设置 `tools.media.audio.maxChars` 或每个条目的 `maxChars` 可截短输出。
- OpenAI 自动检测默认值为 `gpt-4o-transcribe`；设置 `model: "gpt-4o-mini-transcribe"` 可使用成本更低、速度更快的选项。
- 模板可通过 `{{Transcript}}` 访问转录文本。
- `tools.media.audio.echoTranscript` 默认关闭；`echoFormat` 接受 `{transcript}` 占位符。
- CLI 标准输出上限为 5MB；请保持 CLI 输出简洁。
- CLI `args` 应使用 `{{AttachmentPath}}` 表示本地音频文件路径。运行 `openclaw doctor --fix`，以迁移旧版 `audio.transcription.command` 配置中已弃用的 `{input}` 占位符（已停用的键：`audio.transcription`，由 `tools.media.models` 取代）。`{{MediaPath}}` 仍作为已弃用的兼容别名保留。
- `tools.media.concurrency` 限制媒体任务；它不是 GPU 调度器。

### 常驻本地 STT

自动检测到的本地 STT 仍采用每个请求一个进程的方式。OpenClaw 目前不管理常驻 whisper.cpp 服务器，因为标准 Homebrew `whisper-cpp` 软件包禁用了该服务器，而上游示例未配置有界准入队列。由插件所有的常驻生命周期需要具备受维护的打包工作进程，包含健康检查/启动、模型常驻、有界队列、取消/超时、仅 local loopback 的无身份验证运行且无云端回退，之后才能安全启用。

### 代理环境支持

基于提供商的音频转录遵循标准出站代理环境变量，与 undici 的 `EnvHttpProxyAgent` 语义一致：

- `HTTPS_PROXY` / `https_proxy`
- `HTTP_PROXY` / `http_proxy`
- `ALL_PROXY` / `all_proxy`

小写变量优先于大写变量；`NO_PROXY`/`no_proxy` 条目（主机名、`*.suffix` 或 `host:port`）会绕过代理。如果未设置代理环境变量，则使用直接出站连接。如果代理设置失败（URL 格式错误），OpenClaw 会记录警告并回退到直接获取。

## 群组中的提及检测

在支持音频预检的渠道上，如果为群聊设置了 `requireMention: true`，OpenClaw 会在检查提及**之前**转录音频。这样，当无说明文字的语音消息的转录文本包含已配置的提及模式时，它就能通过提及关卡。特定渠道的文档会说明哪些传输方式要求键入提及。

**工作原理：**

1. 如果语音消息没有文本正文，并且群组要求提及，OpenClaw 会对第一个音频附件执行预检转录。
2. 检查转录文本中是否存在提及模式（例如 `@BotName`、表情符号触发器）。
3. 如果找到提及，消息会继续进入完整的回复流水线。

**回退行为：**如果预检转录失败（超时、API 错误等），消息会回退到仅文本提及检测，因此混合消息（文本 + 音频）绝不会被丢弃。

**按 Telegram 群组/话题选择停用：**

- 设置 `channels.telegram.groups.<chatId>.disableAudioPreflight: true`，以跳过该群组的预检转录文本提及检查。
- 设置 `channels.telegram.groups.<chatId>.topics.<threadId>.disableAudioPreflight`，以按话题覆盖（`true` 表示跳过，`false` 表示强制启用）。
- 默认值为 `false`（当符合提及关卡条件时启用预检）。

**示例：**在设置了 `requireMention: true` 的 Telegram 群组中，用户发送一条语音消息，说“嘿 @Claude，天气怎么样？”。该语音消息会被转录，系统检测到提及，随后智能体进行回复。

## 注意事项

- 作用域规则采用首次匹配优先；`chatType` 会规范化为 `direct`、`group` 或 `channel`。
- 确保 CLI 以状态码 0 退出并输出纯文本；JSON 输出需要通过 `jq -r .text` 进行处理。
- 已知的文件输出模式具有权威性：如果推断的转录文件为空或缺失，则不会生成转录文本，也不会回退到 CLI 进度输出。
- 对于 `parakeet-mlx`，请将 `--output-format txt`（或 `all`）与 `--output-dir` 及默认 `{filename}` 输出模板搭配使用。上游的 `PARAKEET_OUTPUT_FORMAT` 和 `PARAKEET_OUTPUT_TEMPLATE` 环境变量也会生效。OpenClaw 读取 `<output-dir>/<media-basename>.txt`；默认 `srt` 格式、其他格式和自定义输出模板仍使用标准输出。
- 请设置合理的超时（`timeoutSeconds`，默认 60s），以避免阻塞回复队列。
- 预检转录仅处理**第一个**音频附件以进行提及检测。其他音频附件会在主要媒体理解阶段处理。

## 相关内容

- [媒体理解](/zh-CN/nodes/media-understanding)
- [Talk 模式](/zh-CN/nodes/talk)
- [语音唤醒](/zh-CN/nodes/voicewake)
