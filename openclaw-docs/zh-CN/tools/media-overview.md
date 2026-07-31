---
read_when:
    - 想了解 OpenClaw 媒体功能的概览
    - 确定要配置哪个媒体提供商
    - 了解异步媒体生成的工作原理
sidebarTitle: Media overview
summary: 图像、视频、音乐、语音和媒体理解能力一览
title: 媒体概览
x-i18n:
    generated_at: "2026-07-26T07:03:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 18eb79e6915c5dc8d705bf5cadfcdddecaf7d21a037f102696d4f2bcd41e5bea
    source_path: tools/media-overview.md
    workflow: 16
---

OpenClaw 可生成图像、视频和音乐，理解入站媒体
（图像、音频、视频），并通过文本转语音朗读回复。所有
媒体能力均由工具驱动：智能体根据对话决定何时使用它们，
且每个工具仅在配置了至少一个后端
提供商时才会出现。

实时语音使用 Talk 会话契约，而非一次性媒体工具
路径。Talk 有三种模式：提供商原生的 `realtime`、本地或流式
`stt-tts`，以及用于仅观察式语音捕获的 `transcription`。这些模式
与电话、会议、浏览器实时通信和原生按键通话客户端
共享提供商目录、事件信封和取消语义。

## 能力

<CardGroup cols={2}>
  <Card title="图像生成" href="/zh-CN/tools/image-generation" icon="image">
    通过 `image_generate`，根据文本提示词或参考图像创建和编辑图像。
    在聊天会话中异步执行——在后台运行，并在准备就绪后
    发布结果。
  </Card>
  <Card title="视频生成" href="/zh-CN/tools/video-generation" icon="video">
    通过 `video_generate` 实现文本生成视频、图像生成视频和视频转换视频。
    异步执行——在后台运行，并在准备就绪后发布结果。
  </Card>
  <Card title="音乐生成" href="/zh-CN/tools/music-generation" icon="music">
    通过 `music_generate` 生成音乐或音轨。在聊天
    会话中基于共享的媒体生成任务生命周期异步执行。
  </Card>
  <Card title="文本转语音" href="/zh-CN/tools/tts" icon="microphone">
    通过 `tts` 工具和 `tts` 配置，
    将出站回复转换为语音音频。同步执行。
  </Card>
  <Card title="媒体理解" href="/zh-CN/nodes/media-understanding" icon="eye">
    使用具备视觉能力的模型提供商和专用媒体理解插件，
    总结入站图像、音频和视频。
  </Card>
  <Card title="语音转文本" href="/zh-CN/nodes/audio" icon="ear-listen">
    通过批量 STT 或语音通话流式 STT 提供商
    转录入站语音消息。
  </Card>
</CardGroup>

## 提供商能力矩阵

<Note>
此表涵盖专用的媒体生成、TTS 和 STT 插件。许多
聊天模型提供商（Anthropic、Google、OpenAI 等）也能通过其回复模型
理解入站媒体；完整提供商列表请参阅
[媒体理解](/zh-CN/nodes/media-understanding#provider-support-matrix)。
</Note>

| 提供商            | 图像 | 视频 | 音乐 | TTS | STT | 实时语音 | 媒体理解 |
| ----------------- | :---: | :---: | :---: | :-: | :-: | :------------: | :-----------------: |
| Alibaba           |       |   ✓   |       |     |     |                |                     |
| Azure Speech      |       |       |       |  ✓  |     |                |                     |
| BytePlus          |       |   ✓   |       |     |     |                |                     |
| ComfyUI           |   ✓   |   ✓   |   ✓   |     |     |                |                     |
| Deepgram          |       |       |       |     |  ✓  |                |                     |
| DeepInfra         |   ✓   |   ✓   |       |  ✓  |  ✓  |                |          ✓          |
| ElevenLabs        |       |       |       |  ✓  |  ✓  |                |                     |
| fal               |   ✓   |   ✓   |   ✓   |     |     |                |                     |
| Google            |   ✓   |   ✓   |   ✓   |  ✓  |  ✓  |       ✓        |          ✓          |
| Gradium           |       |       |       |  ✓  |     |                |                     |
| Inworld           |       |       |       |  ✓  |     |                |                     |
| LiteLLM           |   ✓   |       |       |     |     |                |                     |
| Local CLI         |       |       |       |  ✓  |     |                |                     |
| Microsoft         |       |       |       |  ✓  |     |                |                     |
| Microsoft Foundry |   ✓   |       |       |     |     |                |                     |
| MiniMax           |   ✓   |   ✓   |   ✓   |  ✓  |     |                |                     |
| Mistral           |       |       |       |     |  ✓  |                |                     |
| OpenAI            |   ✓   |   ✓   |       |  ✓  |  ✓  |       ✓        |          ✓          |
| OpenRouter        |   ✓   |   ✓   |   ✓   |  ✓  |  ✓  |                |          ✓          |
| PixVerse          |       |   ✓   |       |     |     |                |                     |
| Qwen              |       |   ✓   |       |     |     |                |          ✓          |
| Runway            |       |   ✓   |       |     |     |                |                     |
| SenseAudio        |       |       |       |     |  ✓  |                |                     |
| Together          |       |   ✓   |       |     |     |                |                     |
| Volcengine        |       |       |       |  ✓  |     |                |                     |
| Vydra             |   ✓   |   ✓   |       |  ✓  |     |                |                     |
| xAI               |   ✓   |   ✓   |       |  ✓  |  ✓  |                |          ✓          |
| Xiaomi MiMo       |       |       |       |  ✓  |     |                |                     |

<Note>
此处的**实时语音**是指提供商原生的双向实时通信（Talk
`realtime` 模式，例如 Gemini Live 或 OpenAI Realtime API）——目前仅 Google
和 OpenAI 注册了该能力。Deepgram、ElevenLabs、Mistral、OpenAI 和 xAI
另行注册了语音通话流式 STT（单向音频转文本）；请参阅下文的
[语音转文本和语音通话](#speech-to-text-and-voice-call)。
xAI 实时语音是一项上游能力，但在共享实时语音契约能够表示它之前，
不会在 OpenClaw 中注册。
</Note>

## 异步与同步

| 能力         | 模式     | 原因                                                                                                 |
| ------------ | -------- | ---------------------------------------------------------------------------------------------------- |
| 图像         | 异步     | 提供商处理可能超出一个聊天轮次；生成的附件使用共享完成路径。                                         |
| 文本转语音   | 同步     | 提供商响应会在数秒内返回；附加到回复音频。                                                           |
| 视频         | 异步     | 提供商处理需要 30 s 到数分钟；较慢的队列最长可运行到配置的超时时间。                                 |
| 音乐         | 异步     | 与视频具有相同的提供商处理特性。                                                                     |

对于异步工具，OpenClaw 将请求提交给提供商，立即返回任务
ID，并在任务账本中跟踪该作业。作业运行期间，智能体会继续
响应其他消息。提供商完成后，
OpenClaw 会使用生成的媒体路径唤醒智能体，使其可通过会话的正常可见回复模式告知
用户：配置后自动发送最终回复，
或在会话要求使用消息工具时使用 `message(action="send")`。
如果请求方会话处于非活动状态或其主动唤醒失败，
且完成回复中仍缺少部分生成的媒体，
OpenClaw 会发送一次幂等的直接回退，其中仅包含缺失的媒体。已通过
完成回复发送的媒体不会再次发布。

## 语音转文本和语音通话

配置后，Deepgram、DeepInfra、ElevenLabs、Google、Groq、Mistral、OpenAI、OpenRouter、
SenseAudio 和 xAI 均可通过批量
`tools.media.audio` 路径转录入站音频。为提及门控或命令解析预检
语音消息的渠道插件会在入站上下文中标记已转录的
附件，因此共享媒体理解流程会
复用该转录文本，而不会对同一段
音频再次发起 STT 调用。

Deepgram、ElevenLabs、Mistral、OpenAI 和 xAI 还注册了语音通话
流式 STT 提供商，因此实时电话音频无需等待录音完成，
即可转发到选定的供应商。

对于实时用户对话，优先使用 [Talk 模式](/zh-CN/nodes/talk)。批量音频
附件仍使用媒体路径；浏览器实时通信、原生按键通话、
电话和会议音频应使用 Talk 事件，以及 Gateway 网关返回的会话范围
目录。

## 提供商映射（供应商如何分布于各个功能面）

<AccordionGroup>
  <Accordion title="Google">
    图像、视频、音乐、批量 TTS、批量 STT、后端实时语音和
    媒体理解功能面。
  </Accordion>
  <Accordion title="OpenAI">
    图像、视频、批量 TTS、批量 STT、语音通话流式 STT、后端
    实时语音和记忆嵌入功能面。
  </Accordion>
  <Accordion title="DeepInfra">
    聊天/模型路由、图像生成/编辑、文本生成视频、批量 TTS、
    批量 STT、图像媒体理解和记忆嵌入功能面。
    DeepInfra 还提供重排序、分类、对象检测和
    其他原生模型类型；OpenClaw 尚未针对这些
    类别提供提供商契约，因此此插件不会注册它们。
  </Accordion>
  <Accordion title="xAI">
    图像、视频、搜索、代码执行、批量 TTS、批量 STT 和语音
    通话流式 STT。xAI 实时语音是一项上游能力，但在
    共享实时语音契约能够表示它之前，不会在 OpenClaw 中
    注册。
  </Accordion>
</AccordionGroup>

## 相关内容

- [图像生成](/zh-CN/tools/image-generation)
- [视频生成](/zh-CN/tools/video-generation)
- [音乐生成](/zh-CN/tools/music-generation)
- [文本转语音](/zh-CN/tools/tts)
- [媒体理解](/zh-CN/nodes/media-understanding)
- [音频节点](/zh-CN/nodes/audio)
- [Talk 模式](/zh-CN/nodes/talk)
