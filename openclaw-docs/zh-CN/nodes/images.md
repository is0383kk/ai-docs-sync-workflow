---
read_when:
    - 修改媒体管道或附件
summary: 发送、Gateway 网关和智能体回复的图像与媒体处理规则
title: 图像和媒体支持
x-i18n:
    generated_at: "2026-07-26T06:18:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71f5591f4268593c142056370802b702899787a79f9ca1fbde6ea8e422f34023
    source_path: nodes/images.md
    workflow: 16
---

WhatsApp 渠道基于 Baileys Web 运行。本页介绍发送、Gateway 网关和智能体回复中的媒体处理规则。

## 目标

- 通过 `openclaw message send --media` 发送媒体，并可选择添加说明文字。
- 允许来自 Web 收件箱的自动回复在文本之外附带媒体。
- 确保各类型的限制合理且可预测。

## CLI 接口

`openclaw message send --target <dest> --media <path-or-url> [--message <caption>]`

- `--media <path-or-url>` — 附加媒体（图像/音频/视频/文档）；接受本地路径或 URL。此项可选；对于仅含媒体的发送，说明文字可以为空。
- `--gif-playback` — 将视频媒体作为 GIF 播放（仅限 WhatsApp）。
- `--force-document` — 将媒体作为文档发送以避免渠道压缩（Telegram、WhatsApp）；适用于图像、GIF 和视频。
- `--reply-to <id>`、`--thread-id <id>`、`--pin`、`--silent` — 与纯文本发送共用的投递/线程选项。
- `--dry-run` — 输出解析后的有效载荷并跳过发送。
- `--json` — 将结果输出为 JSON：`{ action, channel, dryRun, handledBy, messageId?, payload }`（`payload` 包含特定于渠道的发送结果，包括任何媒体引用）。

## WhatsApp Web 渠道行为

- 输入：本地文件路径**或** HTTP(S) URL。
- 流程：加载到缓冲区中，检测媒体类型，然后按类型构建出站有效载荷：
  - **图像：**经过优化以保持在 `channels.whatsapp.mediaMaxMb`（默认 50MB）以内。不透明图像会重新压缩为 JPEG（默认边长梯度从 2048px 开始，在多次未达到大小要求时逐级缩小）；带透明度的图像保留为 PNG。如果源文件已经是符合要求的 JPEG/PNG/WebP，且未超出大小和边长限制，则原始字节保持不变，不会重新压缩。动画 GIF 永不重新编码，仅检查大小。
  - **音频/语音：**除非已经是原生语音音频（`.ogg`/`.opus` 或 `audio/ogg`/`audio/opus`），否则出站音频会先通过 `ffmpeg` 转码为 Opus/OGG（48kHz 单声道、64kbps、最长 20 分钟），再作为语音消息（`ptt: true`）发送。
  - **视频：**不超过 16MB 时直接传递。
  - **文档：**其他所有内容，最大 100MB；如有文件名则予以保留。
- WhatsApp GIF 风格播放：发送带有 `gifPlayback: true` 的 MP4（CLI：`--gif-playback`），以便移动客户端在消息中循环播放。
- MIME 检测优先使用探测到的魔数，其次是文件扩展名，最后是响应标头；探测到的通用容器（`application/octet-stream`、`zip`）绝不会覆盖更具体的扩展名映射（例如 XLSX 与 ZIP）。
- 说明文字来自 `--message` 或 `reply.text`；允许使用空说明文字。
- 日志：非详细模式显示 `↩️`/`✅`；详细模式还包括大小和源路径/URL。

<Note>
上述 16MB 音频/视频和 100MB 文档数值，是未传入显式字节上限时使用的各类型共用媒体默认值。WhatsApp 发送会从 `channels.whatsapp.mediaMaxMb` 设置显式上限（默认 50MB），该上限统一应用于此账户的所有类型。
</Note>

## 自动回复流水线

- `getReplyFromConfig` 返回一个回复有效载荷（或有效载荷数组），其中包含 `text?`、`mediaUrl?` 和 `mediaUrls?` 等字段。
- 存在媒体时，Web 发送器使用与 `openclaw message send` 相同的流水线解析本地路径或 URL。
- 如果提供了多个媒体条目，它们将依次发送。

## 将入站媒体传给命令

- 当入站 Web 消息包含媒体时，OpenClaw 会将其下载到临时文件，并公开以下模板变量：
  - `{{AttachmentUrl}}` — 当前附件的原始 URL 或提供商引用。
  - `{{AttachmentPath}}` — 运行命令前写入的本地临时路径。
  - `{{AttachmentContentType}}` — MIME 内容类型。
  - `{{AttachmentDir}}` — 包含本地路径的目录。
  - `{{AttachmentIndex}}` — 从零开始的源事实索引。
- 启用按会话配置的 Docker 沙箱后，入站媒体会复制到沙箱工作区中，附件路径/引用也会重写为类似 `media/inbound/<filename>` 的沙箱相对路径。
- 在插件 SDK 迁移窗口期间，`{{MediaPath}}`、`{{MediaUrl}}`、`{{MediaType}}` 和 `{{MediaDir}}` 仍作为已弃用的兼容性别名保留。
- 媒体理解（通过 `tools.media.*` 或共用的 `tools.media.models` 配置）在模板处理前运行，并可将 `[Image]`、`[Audio]` 和 `[Video]` 块插入 `Body`。
  - 音频会设置 `{{Transcript}}`，并使用转录文本解析命令，使斜杠命令仍可正常工作。
  - 视频和图像描述会保留任何说明文字，以用于命令解析。
  - 如果当前的主模型已原生支持视觉能力，OpenClaw 会跳过 `[Image]` 摘要块，改为将原始图像传给模型。
- 默认仅处理第一个匹配的图像/音频/视频附件；使用 `tools.media.<capability>.attachments` 可选择多个附件。

## 限制和错误

**出站发送上限（WhatsApp Web 发送）**

- 图像：优化后最大为 `channels.whatsapp.mediaMaxMb`（默认 50MB）。
- 音频/视频：上限为 16MB（共用默认值；通过 WhatsApp 发送时由 `mediaMaxMb` 覆盖）。
- 文档：上限为 100MB（共用默认值；通过 WhatsApp 发送时由 `mediaMaxMb` 覆盖）。
- 媒体过大或无法读取时，日志中会产生明确错误，并跳过该回复。

**媒体理解上限（转录/描述）**

- 图像默认值：10MB（使用 `tools.media.image.maxBytes` 覆盖，或在每个
  `tools.media.models[]` 条目中使用 `maxBytes` 覆盖）。
- 音频默认值：20MB（使用 `tools.media.audio.maxBytes` 覆盖，或按条目覆盖）。
- 视频默认值：50MB（使用 `tools.media.video.maxBytes` 覆盖，或按条目覆盖）。
- 媒体过大时会跳过理解，但仍会使用原始正文完成回复。

## 测试说明

- 覆盖图像、音频和文档场景的发送及回复流程。
- 验证图像优化后的大小限制，以及音频的语音消息标志。
- 确保多媒体回复分拆为依次发送的多条消息。

## 相关内容

- [相机拍摄](/zh-CN/nodes/camera)
- [媒体理解](/zh-CN/nodes/media-understanding)
- [音频和语音消息](/zh-CN/nodes/audio)
