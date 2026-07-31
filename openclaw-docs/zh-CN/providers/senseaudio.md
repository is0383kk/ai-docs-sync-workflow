---
read_when:
    - 你希望使用 SenseAudio 将音频附件转换为文本
    - 你需要设置 SenseAudio API key 环境变量或音频配置路径
summary: 用于入站语音留言的 SenseAudio 批量语音转文本
title: SenseAudio
x-i18n:
    generated_at: "2026-07-26T07:00:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0ca4a31a32eed85c1d9dcd13ebc2eaea94be370d2b1013ae8b4677949bea91d
    source_path: providers/senseaudio.md
    workflow: 16
---

SenseAudio 通过 OpenClaw 的共享 `tools.media.audio` 流程转录收到的音频和语音留言附件。OpenClaw 将多部分音频发送到兼容 OpenAI 的转录端点，并将返回的文本作为 `{{Transcript}}` 以及一个 `[Audio]` 块注入。

| 属性          | 值                                               |
| ------------- | ------------------------------------------------ |
| 提供商 ID     | `senseaudio`                               |
| 插件          | 内置，`enabledByDefault: true`                         |
| 契约          | `mediaUnderstandingProviders`（音频）                       |
| 身份验证环境变量 | `SENSEAUDIO_API_KEY`                            |
| 默认模型      | `senseaudio-asr-pro-1.5-260319`                               |
| 默认 URL      | `https://api.senseaudio.cn/v1`                               |
| 网站          | [senseaudio.cn](https://senseaudio.cn)            |
| 文档          | [senseaudio.cn/docs](https://senseaudio.cn/docs)  |

## 入门指南

<Steps>
  <Step title="设置 API 密钥">
    ```bash
    export SENSEAUDIO_API_KEY="..."
    ```
  </Step>
  <Step title="启用音频提供商">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [{ provider: "senseaudio", model: "senseaudio-asr-pro-1.5-260319" }],
          },
        },
      },
    }
    ```
  </Step>
  <Step title="发送语音留言">
    通过任意已连接的渠道发送音频消息。OpenClaw 会将音频上传到
    SenseAudio，并在回复流程中使用转录文本。
  </Step>
</Steps>

## 选项

| 选项       | 路径                            | 描述                         |
| ---------- | ------------------------------- | ---------------------------- |
| `model`    | `tools.media.models[].model`    | SenseAudio ASR 模型 ID       |
| `language` | `tools.media.models[].language` | 可选的语言提示               |
| `prompt`   | `tools.media.models[].prompt`   | 可选的转录提示词             |
| `baseUrl`  | `tools.media.models[].baseUrl`  | 覆盖兼容 OpenAI 的基础地址   |
| `headers`  | `tools.media.models[].headers`  | 额外的请求标头               |

<Note>
在 OpenClaw 中，SenseAudio 仅支持批量语音转文本。语音通话的实时转录
继续使用支持流式语音转文本的提供商。
</Note>

## 相关内容

- [媒体理解（音频）](/zh-CN/nodes/audio)
- [模型提供商](/zh-CN/concepts/model-providers)
