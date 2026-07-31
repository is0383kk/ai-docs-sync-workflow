---
read_when:
    - 你希望使用 Gradium 进行文本转语音
    - 你需要配置 Gradium API key、语音或指令令牌
summary: 在 OpenClaw 中使用 Gradium 文本转语音
title: Gradium
x-i18n:
    generated_at: "2026-07-26T06:59:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5536426eb6d3c8f24c04643b033ebb519a1f2f9df9d97c917ced1c7e23ad180d
    source_path: providers/gradium.md
    workflow: 16
---

[Gradium](https://gradium.ai) 是 OpenClaw 的文本转语音提供商。它可生成标准音频回复（WAV）、兼容语音消息的 Opus 输出，以及用于电话通信界面的 8 kHz u-law 音频。

| 属性          | 值                                   |
| ------------- | ------------------------------------ |
| 提供商 ID     | `gradium`                            |
| 身份验证      | `GRADIUM_API_KEY` 或配置 `apiKey` |
| 基础 URL      | `https://api.gradium.ai`（默认）   |
| 默认语音      | `Emma`（`YTpq7expH9539ERJ`）          |

## 安装插件

Gradium 是官方外部插件。安装后，重启 Gateway 网关：

```bash
openclaw plugins install @openclaw/gradium-speech
openclaw gateway restart
```

## 设置

创建 Gradium API key，然后通过环境变量或配置键提供该密钥。配置的优先级高于环境变量。

<Tabs>
  <Tab title="环境变量">
    ```bash
    export GRADIUM_API_KEY="gsk_..."
    ```
  </Tab>

  <Tab title="配置键">
    ```json5
    {
      tts: {
        auto: "always",
        provider: "gradium",
        providers: {
          gradium: {
            apiKey: "${GRADIUM_API_KEY}",
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## 配置

```json5
{
  tts: {
    auto: "always",
    provider: "gradium",
    providers: {
      gradium: {
        speakerVoiceId: "YTpq7expH9539ERJ",
        // apiKey: "${GRADIUM_API_KEY}",
        // baseUrl: "https://api.gradium.ai",
      },
    },
  },
}
```

| 键                                     | 类型   | 描述                                                                                                    |
| -------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------- |
| `tts.providers.gradium.apiKey`         | string | 解析后的 API key。支持 `${ENV}` 和密钥引用。                                                    |
| `tts.providers.gradium.baseUrl`        | string | `api.gradium.ai` 上的 Gradium HTTPS API URL。会移除末尾的斜杠。默认值为 `https://api.gradium.ai`。 |
| `tts.providers.gradium.speakerVoiceId` | string | 不存在指令覆盖时使用的默认语音 ID。                                            |

输出格式会根据目标界面自动选择（参阅[输出](#output)），无法在 `openclaw.json` 中配置。

## 语音

| 名称               | 语音 ID            |
| ------------------ | ------------------ |
| Arthur             | `3jUdJyOi9pgbxBTK` |
| Christina          | `2H4HY2CBNyJHBCrP` |
| Emma **（默认）** | `YTpq7expH9539ERJ` |
| John               | `KWJiFWu2O9nMPYcR` |
| Kent               | `LFZvm12tW_z0xfGo` |
| Sydney             | `jtEKaLYNn6iif5PR` |
| Tiffany            | `Eu9iL_CYe8N-Gkx_` |

### 按消息覆盖语音

当生效的语音策略允许覆盖语音时，可使用指令令牌在消息内切换语音（以下指令效果相同，均接受提供商原生语音 ID）：

```text
/voice:LFZvm12tW_z0xfGo
/voice_id:LFZvm12tW_z0xfGo
/voiceid:LFZvm12tW_z0xfGo
/gradium_voice:LFZvm12tW_z0xfGo
/gradiumvoice:LFZvm12tW_z0xfGo
```

如果语音策略禁用了语音覆盖，该指令仍会被消费，但会被忽略。

## 输出

输出格式由目标界面决定；提供商不会合成其他格式。

| 目标           | 格式        | 文件扩展名 | 采样率      | 语音兼容标志 |
| -------------- | ----------- | ---------- | ----------- | ------------ |
| 标准音频       | `wav`       | `.wav`   | 提供商      | 否           |
| 语音消息       | `opus`      | `.opus`  | 提供商      | 是           |
| 电话通信       | `ulaw_8000` | 不适用     | 8 kHz       | 不适用       |

## 自动选择顺序

在已配置的 TTS 提供商中，Gradium 的自动选择顺序为 `30`。有关未固定 `tts.provider` 时 OpenClaw 如何选择生效提供商的信息，请参阅[文本转语音](/zh-CN/tools/tts)。

## 相关内容

- [文本转语音](/zh-CN/tools/tts)
- [媒体概览](/zh-CN/tools/media-overview)
