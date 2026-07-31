---
read_when:
    - 你想在 OpenClaw 中使用 ElevenLabs 文本转语音功能
    - 你希望使用 ElevenLabs Scribe 将音频附件转换为文本
    - 你希望为语音通话或 Google Meet 使用 ElevenLabs 实时转录
summary: 通过 OpenClaw 使用 ElevenLabs 语音、Scribe 语音转文本和实时转录
title: ElevenLabs
x-i18n:
    generated_at: "2026-07-26T06:59:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5c570aab5fd3ca00e8ded8e3daa143cb199334d507461800ec0b6c1ab0b65c59
    source_path: providers/elevenlabs.md
    workflow: 16
---

OpenClaw 使用 ElevenLabs 进行文本转语音、通过 Scribe v2 进行批量语音转文本，以及通过 Scribe v2 Realtime 进行流式 STT。该插件已内置且默认启用；无需执行 `plugins install` 步骤。

| 能力                     | OpenClaw 功能界面                                                     | 默认值                   |
| ------------------------ | -------------------------------------------------------------------- | ------------------------ |
| 文本转语音               | `tts` / `talk`                                                       | `eleven_multilingual_v2` |
| 批量语音转文本           | `tools.media.audio`                                                  | `scribe_v2`              |
| 流式语音转文本           | 语音通话流式传输或 Google Meet `realtime.transcriptionProvider` | `scribe_v2_realtime`     |

## 身份验证

在环境中设置 `ELEVENLABS_API_KEY`。为兼容现有 ElevenLabs 工具，也接受 `XI_API_KEY`。

```bash
export ELEVENLABS_API_KEY="..."
```

## 文本转语音

```json5
{
  tts: {
    providers: {
      elevenlabs: {
        apiKey: "${ELEVENLABS_API_KEY}",
        voiceId: "pMsXgVXv3BLzUgSXRplE",
        modelId: "eleven_multilingual_v2",
      },
    },
  },
}
```

将 `modelId` 设置为 `eleven_v3` 即可使用 ElevenLabs v3 TTS。对于现有安装，OpenClaw 仍将 `eleven_multilingual_v2` 保持为默认值。

当 ElevenLabs 是所选的 `voice.tts`/`tts` 提供商时，Discord 语音频道会使用 ElevenLabs 的流式 TTS 端点：播放直接从返回的音频流开始，而不是先等待 OpenClaw 下载完整音频文件。对于支持该参数的模型，`latencyTier` 会映射到 ElevenLabs 的 `optimize_streaming_latency` 查询参数；对于会拒绝该参数的 `eleven_v3`，OpenClaw 会省略此参数。

## 语音转文本

对入站音频附件和短时录制语音片段使用 Scribe v2：

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "elevenlabs", model: "scribe_v2" }],
      },
    },
  },
}
```

OpenClaw 使用 `model_id: "scribe_v2"` 将 multipart 音频发送到 ElevenLabs `/v1/speech-to-text`。如果存在语言提示，则会映射到 `language_code`。

## 流式 STT

内置的 `elevenlabs` 插件为语音通话和 Google Meet Agent 模式流式转录注册 Scribe v2 Realtime。

| 设置            | 配置路径                                                                  | 默认值                                            |
| --------------- | ------------------------------------------------------------------------- | ------------------------------------------------- |
| API 密钥        | `plugins.entries.voice-call.config.streaming.providers.elevenlabs.apiKey` | 回退到 `ELEVENLABS_API_KEY` / `XI_API_KEY` |
| 模型            | `...elevenlabs.modelId`                                                   | `scribe_v2_realtime`                              |
| 音频格式        | `...elevenlabs.audioFormat`                                               | `ulaw_8000`                                       |
| 采样率          | `...elevenlabs.sampleRate`                                                | `8000`                                            |
| 提交策略        | `...elevenlabs.commitStrategy`                                            | `vad`                                             |
| 语言            | `...elevenlabs.languageCode`                                              | （未设置）                                        |

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          streaming: {
            enabled: true,
            provider: "elevenlabs",
            providers: {
              elevenlabs: {
                apiKey: "${ELEVENLABS_API_KEY}",
                audioFormat: "ulaw_8000",
                commitStrategy: "vad",
                languageCode: "en",
              },
            },
          },
        },
      },
    },
  },
}
```

<Note>
语音通话以 8 kHz G.711 μ-law 格式接收 Twilio 媒体。ElevenLabs 实时提供商默认为 `ulaw_8000`，因此可以直接转发电话音频帧，无需转码。
</Note>

对于 Google Meet Agent 模式，将 `plugins.entries.google-meet.config.realtime.transcriptionProvider` 设置为 `"elevenlabs"`，并在 `plugins.entries.google-meet.config.realtime.providers.elevenlabs` 下配置相同的提供商块。

## 相关内容

- [文本转语音](/zh-CN/tools/tts)
- [Google Meet](/zh-CN/plugins/google-meet)
- [模型选择](/zh-CN/concepts/model-providers)
