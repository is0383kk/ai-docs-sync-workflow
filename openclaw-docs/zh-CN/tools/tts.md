---
read_when:
    - 为回复启用文本转语音功能
    - 配置 TTS 提供商、回退链或角色设定
    - 使用 /tts 命令或指令
sidebarTitle: Text to speech (TTS)
summary: 出站回复的文本转语音——提供商、角色、斜杠命令和按渠道输出
title: 文本转语音
x-i18n:
    generated_at: "2026-07-26T07:05:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2ae9d0cc6f77c6a8b1b379c3712fd92fbbc22dae694ecdd46a0bb35cec0d29e7
    source_path: tools/tts.md
    workflow: 16
---

OpenClaw 通过 **14 个语音提供商**将出站回复转换为音频：
在 Feishu、Matrix、Telegram 和 WhatsApp 上使用原生语音消息；在其他所有渠道使用音频
附件；并为电话和 Talk 提供 PCM/Ulaw 流。

TTS 是 Talk 的 `stt-tts` 模式中的语音输出部分（`talk.speak` 调用相同的
合成路径）。提供商原生的 `realtime` Talk 会话改为在实时提供商内部
合成语音；`transcription` 会话从不
合成助手语音回复。

## 快速开始

<Steps>
  <Step title="选择提供商">
    OpenAI 和 ElevenLabs 是最可靠的托管选项。Microsoft 和
    Local CLI 无需 API key 即可工作。完整列表请参阅[提供商矩阵](#supported-providers)。
  </Step>
  <Step title="设置 API key">
    导出你的提供商所需的环境变量（例如 `OPENAI_API_KEY`、
    `ELEVENLABS_API_KEY`）。Microsoft 和 Local CLI 无需 API key。
  </Step>
  <Step title="在配置中启用">
    设置 `tts.auto: "always"` 和 `tts.provider`：

    ```json5
    {
      tts: {
        auto: "always",
        provider: "elevenlabs",
      },
    }
    ```

  </Step>
  <Step title="在聊天中试用">
    `/tts status` 显示当前状态。`/tts audio Hello from OpenClaw`
    发送一次性音频回复。
  </Step>
</Steps>

<Note>
自动 TTS 默认**关闭**。未设置 `tts.provider` 时，
OpenClaw 会按照注册表自动选择顺序选取第一个已配置的提供商。
内置的 `tts` Agent 工具仅用于明确意图：普通聊天仍使用
文本，除非用户请求音频、使用 `/tts`，或启用自动 TTS/指令
语音。
</Note>

## 支持的提供商

| 提供商          | 身份验证                                                                                                             | 说明                                                                                       |
| ----------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Azure Speech**  | `AZURE_SPEECH_KEY` + `AZURE_SPEECH_REGION`（也支持 `AZURE_SPEECH_API_KEY`、`SPEECH_KEY`、`SPEECH_REGION`）          | 原生 Ogg/Opus 语音留言输出和电话功能。                                            |
| **DeepInfra**     | `DEEPINFRA_API_KEY`                                                                                              | 兼容 OpenAI 的 TTS。默认为 `hexgrad/Kokoro-82M`。                                    |
| **ElevenLabs**    | `ELEVENLABS_API_KEY` 或 `XI_API_KEY`                                                                             | 支持语音克隆、多语言，并可通过 `seed` 实现确定性；以流式传输用于 Discord 语音播放。 |
| **Google Gemini** | `GEMINI_API_KEY` 或 `GOOGLE_API_KEY`                                                                             | Gemini API 批量 TTS；通过 `promptTemplate: "audio-profile-v1"` 感知角色设定。               |
| **Gradium**       | `GRADIUM_API_KEY`                                                                                                | 语音留言和电话输出。                                                            |
| **Inworld**       | `INWORLD_API_KEY`                                                                                                | 流式 TTS API。原生 Opus 语音留言和 PCM 电话音频。                                |
| **Local CLI**     | 无                                                                                                             | 运行已配置的本地 TTS 命令。                                                        |
| **Microsoft**     | 无                                                                                                             | 通过 `node-edge-tts` 使用公共 Edge 神经网络 TTS。尽力而为，不提供 SLA。                            |
| **MiniMax**       | `MINIMAX_API_KEY`（或 Token Plan：`MINIMAX_OAUTH_TOKEN`、`MINIMAX_CODE_PLAN_KEY`、`MINIMAX_CODING_API_KEY`）      | T2A v2 API。默认为 `speech-2.8-hd`。                                                    |
| **OpenAI**        | `OPENAI_API_KEY`                                                                                                 | 也用于自动摘要；支持角色设定 `instructions`。                                |
| **OpenRouter**    | `OPENROUTER_API_KEY`（可复用 `models.providers.openrouter.apiKey`）                                            | 默认模型为 `hexgrad/kokoro-82m`。                                                         |
| **Volcengine**    | `VOLCENGINE_TTS_API_KEY` 或 `BYTEPLUS_SEED_SPEECH_API_KEY`（旧版 AppID/token：`VOLCENGINE_TTS_APPID`/`_TOKEN`） | BytePlus Seed Speech HTTP API。                                                              |
| **Vydra**         | `VYDRA_API_KEY`                                                                                                  | 共享的图像、视频和语音提供商。                                                   |
| **xAI**           | `XAI_API_KEY`                                                                                                    | xAI 批量 TTS。**不**支持原生 Opus 语音留言。                                 |
| **Xiaomi MiMo**   | `XIAOMI_API_KEY`                                                                                                 | 通过 Xiaomi 聊天补全提供 MiMo TTS。                                                   |

如果配置了多个提供商，将优先使用选定的提供商，其他提供商则作为回退选项。
自动摘要使用 `summaryModel`（或
`agents.defaults.model.primary`），因此如果保持启用摘要，
也必须对该提供商进行身份验证。

<Warning>
内置的 **Microsoft** 提供商通过 `node-edge-tts` 使用 Microsoft Edge 的在线神经网络 TTS
服务。这是一项没有公开
SLA 或配额的公共 Web 服务——应将其视为尽力而为的服务。旧版提供商 ID `edge`
会被规范化为 `microsoft`，而 `openclaw doctor --fix` 会重写持久化
配置；新配置应始终使用 `microsoft`。
</Warning>

## 配置

TTS 配置位于 `~/.openclaw/openclaw.json` 中的 `tts` 下。选择一个
预设并调整提供商配置块。下方显示的 `speakerVoice`/`speakerVoiceId`
字段是规范字段；每个提供商自己的 `voice`/`voiceId`/
`voiceName` 字段名仍可作为旧版别名使用。

<Tabs>
  <Tab title="Azure Speech">
```json5
{
  tts: {
    auto: "always",
    provider: "azure-speech",
    providers: {
      "azure-speech": {
        apiKey: "${AZURE_SPEECH_KEY}",
        region: "eastus",
        speakerVoice: "en-US-JennyNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
        voiceNoteOutputFormat: "ogg-24khz-16bit-mono-opus",
      },
    },
  },
}
```
  </Tab>
  <Tab title="ElevenLabs">
```json5
{
  tts: {
    auto: "always",
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        apiKey: "${ELEVENLABS_API_KEY}",
        model: "eleven_multilingual_v2",
        speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Google Gemini">
```json5
{
  tts: {
    auto: "always",
    provider: "google",
    providers: {
      google: {
        apiKey: "${GEMINI_API_KEY}",
        model: "gemini-3.1-flash-tts-preview",
        speakerVoice: "Kore",
        // 可选的自然语言风格提示：
        // audioProfile: "以平静的播客主持人口吻说话。",
        // speakerName: "Alex",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Gradium">
```json5
{
  tts: {
    auto: "always",
    provider: "gradium",
    providers: {
      gradium: {
        apiKey: "${GRADIUM_API_KEY}",
        speakerVoiceId: "YTpq7expH9539ERJ",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Inworld">
```json5
{
  tts: {
    auto: "always",
    provider: "inworld",
    providers: {
      inworld: {
        apiKey: "${INWORLD_API_KEY}",
        modelId: "inworld-tts-1.5-max",
        speakerVoiceId: "Sarah",
        temperature: 0.7,
      },
    },
  },
}
```
  </Tab>
  <Tab title="Local CLI">
```json5
{
  tts: {
    auto: "always",
    provider: "tts-local-cli",
    providers: {
      "tts-local-cli": {
        command: "say",
        args: ["-o", "{{OutputPath}}", "{{Text}}"],
        outputFormat: "wav",
        timeoutMs: 120000,
      },
    },
  },
}
```
  </Tab>
  <Tab title="Microsoft（无需 API key）">
```json5
{
  tts: {
    auto: "always",
    provider: "microsoft",
    providers: {
      microsoft: {
        enabled: true,
        speakerVoice: "en-US-MichelleNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
        rate: "+0%",
        pitch: "+0%",
      },
    },
  },
}
```
  </Tab>
  <Tab title="MiniMax">
```json5
{
  tts: {
    auto: "always",
    provider: "minimax",
    providers: {
      minimax: {
        apiKey: "${MINIMAX_API_KEY}",
        model: "speech-2.8-hd",
        speakerVoiceId: "English_expressive_narrator",
        speed: 1.0,
        vol: 1.0,
        pitch: 0,
      },
    },
  },
}
```
  </Tab>
  <Tab title="OpenAI + ElevenLabs">
```json5
{
  tts: {
    auto: "always",
    provider: "openai",
    summaryModel: "openai/gpt-4.1-mini",
    modelOverrides: { enabled: true },
    providers: {
      openai: {
        apiKey: "${OPENAI_API_KEY}",
        model: "gpt-4o-mini-tts",
        speakerVoice: "alloy",
      },
      elevenlabs: {
        apiKey: "${ELEVENLABS_API_KEY}",
        model: "eleven_multilingual_v2",
        speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
        voiceSettings: { stability: 0.5, similarityBoost: 0.75, style: 0.0, useSpeakerBoost: true, speed: 1.0 },
        applyTextNormalization: "auto",
        languageCode: "en",
      },
    },
  },
}
```
  </Tab>
  <Tab title="OpenRouter">
```json5
{
  tts: {
    auto: "always",
    provider: "openrouter",
    providers: {
      openrouter: {
        apiKey: "${OPENROUTER_API_KEY}",
        model: "hexgrad/kokoro-82m",
        speakerVoice: "af_alloy",
        responseFormat: "mp3",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Volcengine">
```json5
{
  tts: {
    auto: "always",
    provider: "volcengine",
    providers: {
      volcengine: {
        apiKey: "${VOLCENGINE_TTS_API_KEY}",
        resourceId: "seed-tts-1.0",
        speakerVoice: "en_female_anna_mars_bigtts",
      },
    },
  },
}
```
  </Tab>
  <Tab title="xAI">
```json5
{
  tts: {
    auto: "always",
    provider: "xai",
    providers: {
      xai: {
        apiKey: "${XAI_API_KEY}",
        speakerVoiceId: "eve",
        language: "en",
        responseFormat: "mp3",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Xiaomi MiMo">
```json5
{
  tts: {
    auto: "always",
    provider: "xiaomi",
    providers: {
      xiaomi: {
        apiKey: "${XIAOMI_API_KEY}",
        model: "mimo-v2.5-tts",
        speakerVoice: "mimo_default",
        format: "mp3",
      },
    },
  },
}
```
  </Tab>
</Tabs>

对于 Xiaomi `mimo-v2.5-tts-voicedesign`，请省略 `speakerVoice`，并将 `style` 设置为
语音设计提示。OpenClaw 会将该提示作为 TTS 的 `user` 消息发送，
并且不会为 voicedesign 模型发送 `audio.voice`。

### 每个 Agent 的语音覆盖配置

当某个 Agent 应使用不同的提供商、语音、模型、角色设定或自动 TTS 模式时，
请使用 `agents.entries.*.tts`。Agent 配置块会深度合并并覆盖
`tts`，因此提供商凭据可以保留在全局提供商配置中：

```json5
{
  tts: {
    auto: "always",
    provider: "elevenlabs",
    providers: {
      elevenlabs: { apiKey: "${ELEVENLABS_API_KEY}", model: "eleven_multilingual_v2" },
    },
  },
  agents: {
    list: [
      {
        id: "reader",
        tts: {
          providers: {
            elevenlabs: { speakerVoiceId: "EXAVITQu4vr4xnSDxMaL" },
          },
        },
      },
    ],
  },
}
```

要为每个智能体固定一个 persona，请将 `agents.entries.*.tts.persona` 与提供商配置一同设置——它仅为该智能体覆盖全局 `tts.persona`。

自动回复、`/tts audio`、`/tts status` 和
`tts` 智能体工具的优先级顺序：

1. `tts`
2. 活动的 `agents.entries.*.tts`
3. 渠道覆盖（当渠道支持 `channels.<channel>.tts` 时）
4. 账号覆盖（当渠道传递 `channels.<channel>.accounts.<id>.tts` 时）
5. 此主机的本地 `/tts` 偏好设置
6. 启用[模型驱动指令](#model-driven-directives)时的内联 `[[tts:...]]` 指令

渠道和账号覆盖使用与 `tts` 相同的结构，并在之前的层级之上进行深度合并，因此共享的提供商凭据可以保留在
`tts` 中，而渠道或 Bot 账号只需更改说话声音、模型、persona
或自动模式：

```json5
{
  tts: {
    provider: "openai",
    providers: {
      openai: { apiKey: "${OPENAI_API_KEY}", model: "gpt-4o-mini-tts" },
    },
  },
  channels: {
    feishu: {
      accounts: {
        english: {
          tts: {
            providers: {
              openai: { speakerVoice: "shimmer" },
            },
          },
        },
      },
    },
  },
}
```

## Persona

**Persona** 是一种稳定的语音身份，可以确定性地应用于不同提供商。
它可以首选某个提供商、定义与提供商无关的提示意图，并携带声音、模型、提示模板、种子和声音设置等提供商专属绑定。

### 最小 persona

```json5
{
  tts: {
    auto: "always",
    persona: "narrator",
    personas: {
      narrator: {
        label: "Narrator",
        provider: "elevenlabs",
        providers: {
          elevenlabs: {
            speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
            modelId: "eleven_multilingual_v2",
          },
        },
      },
    },
  },
}
```

### 完整 persona（提供商专属塑形）

```json5
{
  tts: {
    auto: "always",
    persona: "alfred",
    personas: {
      alfred: {
        label: "Alfred",
        description: "Dry, warm British butler narrator.",
        provider: "google",
        fallbackPolicy: "preserve-persona",
        providers: {
          google: {
            model: "gemini-3.1-flash-tts-preview",
            speakerVoice: "Algieba",
            promptTemplate: "audio-profile-v1",
          },
          openai: { model: "gpt-4o-mini-tts", speakerVoice: "cedar" },
          elevenlabs: {
            speakerVoiceId: "voice_id",
            modelId: "eleven_multilingual_v2",
            seed: 42,
            voiceSettings: {
              stability: 0.65,
              similarityBoost: 0.8,
              style: 0.25,
              useSpeakerBoost: true,
              speed: 0.95,
            },
          },
        },
      },
    },
  },
}
```

### Persona 解析

活动 persona 按以下确定性顺序选择：

1. `/tts persona <id>` 本地偏好设置（如果已设置）。
2. `tts.persona`（如果已设置）。
3. 无 persona。

提供商选择遵循显式设置优先：

1. 直接覆盖（CLI、Gateway 网关、Talk、允许的 TTS 指令）。
2. `/tts provider <id>` 本地偏好设置。
3. 活动 persona 的 `provider`。
4. `tts.provider`。
5. 注册表自动选择。

每次尝试提供商时，OpenClaw 按以下顺序合并配置：

1. `tts.providers.<id>`
2. `tts.personas.<persona>.providers.<id>`
3. 可信请求覆盖
4. 允许的模型生成 TTS 指令覆盖

### 自定义 persona 塑形

与提供商无关的 `personas.<id>.prompt.*` 配置已停用。Doctor 会移除这些字段，并指向语音提供商接缝。请将内置提供商设置放在 `personas.<id>.providers.<provider>` 下（例如 Google
`personaPrompt` 或 OpenAI `instructions`）。如需自定义塑形，请使用 `prepareSynthesis(ctx)` 实现语音提供商插件，并在 `synthesize()` 运行前返回调整后的文本、提供商配置或覆盖。这样可将富有表现力的提示构建保留在了解请求语义的提供商代码中。

### 回退策略

`fallbackPolicy` 控制 persona 对尝试的提供商**没有绑定**时的行为：

| 策略                 | 行为                                                                                                                                             |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `preserve-persona`  | **默认。**与提供商无关的提示字段仍然可用；提供商可以使用或忽略这些字段。                                                                                 |
| `provider-defaults` | 此次尝试的提示准备过程会省略 persona；提供商使用其中性默认值，同时继续回退到其他提供商。                                                                  |
| `fail`              | 使用 `reasonCode: "not_configured"` 和 `personaBinding: "missing"` 跳过此次提供商尝试。仍会尝试回退提供商。              |

只有在尝试的**所有**提供商均被跳过或失败时，整个 TTS 请求才会失败。

Talk 会话的提供商选择仅作用于会话范围。Talk 客户端应从 `talk.catalog` 中选择提供商 ID、模型 ID、声音 ID 和区域设置，并通过 Talk 会话或交接请求传递这些值。打开语音会话不应修改 `tts` 或全局 Talk 提供商默认值。

## 模型驱动指令

默认情况下，助手**可以**生成 `[[tts:...]]` 指令，为单次回复覆盖声音、模型或速度，还可以附加可选的
`[[tts:text]]...[[/tts:text]]` 块，用于仅应出现在音频中的表现力提示：

```text
给你。

[[tts:speakerVoiceId=pMsXgVXv3BLzUgSXRplE model=eleven_v3 speed=1.1]]
[[tts:text]]（笑）再读一遍这首歌。[[/tts:text]]
```

当 `tts.auto` 为 `"tagged"` 时，必须使用**指令**才能触发音频。分块流式传输会在渠道看到可见文本之前移除其中的指令，即使指令被拆分到相邻的块中也是如此。

除非 `modelOverrides.allowProvider: true`，否则 `provider=...` 会被忽略。当回复声明 `provider=...` 时，该指令中的其他键仅由该提供商解析；不支持的键会被移除，并报告为 TTS 指令警告。

**可用的指令键：**

- `provider`（已注册的提供商 ID；需要 `allowProvider: true`）
- `speakerVoice` / `speakerVoiceId`（旧版别名：`voice`、`voiceName`、`voice_name`、`google_voice`、`voiceId`）
- `model` / `google_model`
- `stability`、`similarityBoost`、`style`、`speed`、`useSpeakerBoost`
- `vol` / `volume`（MiniMax 音量，`(0, 10]`）
- `pitch`（MiniMax 整数音高，−12 到 12；小数值会被截断）
- `emotion`（Volcengine 情感标签）
- `applyTextNormalization`（`auto|on|off`）
- `languageCode`（ISO 639-1）
- `seed`

**完全禁用模型覆盖：**

```json5
{ messages: { tts: { modelOverrides: { enabled: false } } } }
```

**允许切换提供商，同时保持其他参数可配置：**

```json5
{ messages: { tts: { modelOverrides: { enabled: true, allowProvider: true, allowSeed: false } } } }
```

## 斜杠命令

单个命令 `/tts`。在 Discord 上，OpenClaw 还会注册 `/voice`，因为
`/tts` 是 Discord 内置命令——文本 `/tts ...` 仍然有效。

```text
/tts off | on | status
/tts chat on | off | default
/tts latest
/tts provider <id>
/tts persona <id> | off
/tts limit <chars>
/tts summary off
/tts audio <text>
```

<Note>
命令要求发送者已获授权（适用允许列表/所有者规则），并且必须启用
`commands.text` 或原生命令注册。
</Note>

行为说明：

- `/tts on` 将本地 TTS 偏好设置写入 `always`；`/tts off` 将其写入 `off`。
- `/tts chat on|off|default` 为当前聊天写入会话范围的自动 TTS 覆盖。
- `/tts persona <id>` 写入本地 persona 偏好设置；`/tts persona off` 将其清除。
- `/tts latest` 从当前会话记录中读取最新的助手回复，并将其作为音频发送一次。它仅在会话条目中存储该回复的哈希值，以避免重复发送语音。
- `/tts audio` 生成一次性音频回复（**不会**开启 TTS）。
- `/tts limit <chars>` 接受 **100–4096**（4096 是 Telegram 说明文字/消息的最大值）；超出此范围的值会被拒绝。
- `limit` 和 `summary` 存储在**本地偏好设置**中，而不是主配置中。
- `/tts status` 包含最近一次尝试的回退诊断信息——`Fallback: <primary> -> <used>`、`Attempts: ...`，以及每次尝试的详细信息（`provider:outcome(reasonCode) latency`）。
- `/status` 会在启用 TTS 时显示活动 TTS 模式，以及已配置的提供商、模型、声音和经过清理的自定义端点元数据。

## 每用户偏好设置

斜杠命令会将本地覆盖写入 TTS 偏好设置路径。默认路径为
`~/.openclaw/settings/tts.json`；可使用 `OPENCLAW_TTS_PREFS` 覆盖。Doctor
会将已停用的全局 `tts.prefsPath` 值移入共享机器状态。
在有意让智能体使用独立偏好设置存储的高级多智能体配置中，仍可设置 `agents.entries.<id>.tts.prefsPath`。

| 存储字段 | 效果                                                                           |
| ------------ | -------------------------------------------------------------------------------- |
| `auto`       | 本地自动 TTS 覆盖（`always`、`off`，……）                                     |
| `provider`   | 本地主提供商覆盖                                                  |
| `persona`    | 本地 persona 覆盖                                                           |
| `maxLength`  | 摘要/截断阈值（默认 `1500` 个字符，`/tts limit` 范围为 100–4096） |
| `summarize`  | 摘要开关（默认为 `true`）                                                  |

这些设置会覆盖由 `tts` 加上该主机活动的
`agents.entries.*.tts` 块所产生的有效配置。

## 输出格式

TTS 语音传递由渠道能力驱动。渠道插件会声明语音式 TTS 是否应要求提供商生成原生 `voice-note` 目标，还是继续使用普通的 `audio-file` 合成，以及渠道是否会在发送前对非原生输出进行转码。

| 目标                                  | 格式                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Feishu / Matrix / Telegram / WhatsApp | 语音留言回复优先使用 **Opus**（ElevenLabs 使用 `opus_48000_64`，OpenAI 使用 `opus`）。48 kHz / 64 kbps 可在清晰度与大小之间取得平衡。 |
| 其他渠道                              | **MP3**（ElevenLabs 使用 `mp3_44100_128`，OpenAI 使用 `mp3`）。44.1 kHz / 128 kbps 是语音的默认平衡设置。                  |
| Talk / 电话                           | 提供商原生 **PCM**（Inworld 为 22050 Hz，Google 为 24 kHz），或电话场景使用 Gradium 的 `ulaw_8000`。                                 |

各提供商说明：

- **Feishu / WhatsApp 转码：**当语音留言回复以 MP3/WebM/WAV/M4A 或其他可能的音频文件形式生成时，渠道插件会在发送原生语音消息之前，使用 `ffmpeg`（`libopus`，64 kbps）将其转码为 48 kHz Ogg/Opus。WhatsApp 通过 Baileys `audio` 载荷发送结果，并设置 `ptt: true` 和 `audio/ogg; codecs=opus`。转码失败时：Feishu 会捕获错误并回退为将原始文件作为普通附件发送；WhatsApp 没有回退机制，因此发送本身会失败，而不会发布不兼容的 PTT 载荷。
- **MiniMax：**普通音频附件使用 MP3（`speech-2.8-hd` 模型，32 kHz 采样率）；对于渠道声明支持的语音留言目标，使用 `ffmpeg` 转码为 48 kHz Opus。
- **Xiaomi MiMo：**默认使用 MP3，也可配置为 WAV；对于渠道声明支持的语音留言目标，使用 `ffmpeg` 转码为 48 kHz Opus。
- **本地 CLI：**使用已配置的 `outputFormat`。语音留言目标会转换为 Ogg/Opus，电话输出则使用 `ffmpeg` 转换为原始 16 kHz 单声道 PCM。
- **Google Gemini：**返回原始 24 kHz PCM。OpenClaw 将其封装为 WAV 以用作音频附件，为语音留言目标转码为 48 kHz Opus，并为 Talk/电话直接返回 PCM。
- **Gradium：**音频附件使用 WAV，语音留言目标使用 Opus，电话场景使用 8 kHz 的 `ulaw_8000`。
- **Inworld：**普通音频附件使用 MP3，语音留言目标使用原生 `OGG_OPUS`，Talk/电话使用 22050 Hz 的原始 `PCM`。
- **xAI：**默认使用 MP3；音频文件合成可为缓冲输出和流式输出使用 `mp3`、`wav`、`pcm`、`mulaw` 或 `alaw`。语音留言目标在流式输出和缓冲回退时使用 MP3，因为 xAI 的 `pcm`、`mulaw` 和 `alaw` 输出是不含标头的原始音频。缓冲合成使用 xAI 的批量 REST `/v1/tts` 端点；`textToSpeechStream` 使用原生 `wss://api.x.ai/v1/tts`。这不是实时语音契约。不支持原生 Opus 语音留言格式。
- **Microsoft：**使用 `microsoft.outputFormat`（默认为 `audio-24khz-48kbitrate-mono-mp3`）。
  - 内置传输层接受 `outputFormat`，但服务并不提供所有格式。
  - 输出格式值遵循 Microsoft Speech 输出格式（包括 Ogg/WebM Opus）。
  - Telegram `sendVoice` 接受 OGG/MP3/M4A；如果需要确保生成 Opus 语音消息，请使用 OpenAI/ElevenLabs。
  - 如果配置的 Microsoft 输出格式失败，OpenClaw 会使用 MP3 重试。
  - 如果未设置明确的语音覆盖项，并且使用默认英语语音，当回复文本主要由 CJK 字符组成时，OpenClaw 会自动切换到中文神经网络语音（`zh-CN-XiaoxiaoNeural`，`zh-CN` 区域设置）。

OpenAI 和 ElevenLabs 的输出格式按上述列表针对各渠道固定。

## 自动 TTS 行为

启用 `tts.auto` 后，OpenClaw 会：

- 如果回复已包含结构化媒体，则跳过 TTS。
- 跳过非常短的回复（少于 10 个字符）。
- 启用摘要时，使用
  `summaryModel`（或 `agents.defaults.model.primary`）概括较长的回复。
- 将生成的音频附加到回复中。
- 在 `mode: "final"` 中，文本流完成后，仍会为流式最终回复发送纯音频 TTS；
  生成的媒体会像普通回复附件一样经过相同的
  渠道媒体标准化处理。

如果回复超过 `maxLength`，OpenClaw 绝不会直接跳过音频：

- **启用摘要**（默认）且摘要模型可用：将文本概括为大约
  `maxLength` 个字符，然后合成摘要。
- **关闭摘要**、摘要生成失败，或摘要模型没有可用的 API key：
  将文本截断为 `maxLength` 个字符，然后合成
  截断后的文本。

```text
回复 -> 已启用 TTS？
  否  -> 发送文本
  是 -> 包含媒体 / 过短？
          是 -> 发送文本
          否  -> 长度 > 限制？
                   否  -> TTS -> 附加音频
                   是 -> 摘要已启用且可用？
                            否  -> 截断 -> TTS -> 附加音频
                            是 -> 生成摘要 -> TTS -> 附加音频
```

## 字段参考

<AccordionGroup>
  <Accordion title="顶层 tts.*">
    <ParamField path="auto" type='"off" | "always" | "inbound" | "tagged"'>
      自动 TTS 模式。`inbound` 仅在收到入站语音消息后发送音频；`tagged` 仅在回复包含 `[[tts:...]]` 指令或 `[[tts:text]]` 块时发送音频。
    </ParamField>
    <ParamField path="enabled" type="boolean" deprecated>
      旧版开关。`openclaw doctor --fix` 会将其迁移到 `auto`。
    </ParamField>
    <ParamField path="mode" type='"final" | "all"' default="final">
      除最终回复外，`"all"` 还包括工具/块回复。
    </ParamField>
    <ParamField path="provider" type="string">
      语音提供商 ID。未设置时，OpenClaw 使用注册表自动选择顺序中的第一个已配置提供商。旧版 `provider: "edge"` 会由 `openclaw doctor --fix` 重写为 `"microsoft"`。
    </ParamField>
    <ParamField path="persona" type="string">
      来自 `personas` 的活动角色 ID。标准化为小写。
    </ParamField>
    <ParamField path="personas.<id>" type="object">
      稳定的口语身份。字段：`label`、`description`、`provider`、`fallbackPolicy`、`prompt`、`providers.<provider>`。请参阅[角色](#personas)。
    </ParamField>
    <ParamField path="summaryModel" type="string">
      用于自动摘要的低成本模型；默认为 `agents.defaults.model.primary`。接受 `provider/model` 或已配置的模型别名。
    </ParamField>
    <ParamField path="modelOverrides" type="object">
      允许模型发出 TTS 指令。`enabled` 默认为 `true`；`allowProvider` 默认为 `false`。
    </ParamField>
    <ParamField path="providers.<id>" type="object">
      按语音提供商 ID 设置键名的提供商自有设置。旧版直接配置块（`tts.openai`、`.elevenlabs`、`.microsoft`、`.edge`）会由 `openclaw doctor --fix` 重写；仅提交 `tts.providers.<id>`。
    </ParamField>
    <ParamField path="maxTextLength" type="number" default="4096">
      TTS 输入字符数的硬性上限。超出时，`/tts audio`、`tts.convert` 和 `tts.speak` 会失败。
    </ParamField>
    <ParamField path="timeoutMs" type="number" default="30000">
      请求超时时间，以毫秒为单位。设置后，每次调用的 `timeoutMs`（智能体工具、Gateway 网关）优先；否则，明确配置的 `tts.timeoutMs` 优先于任何插件定义的提供商默认值。
    </ParamField>
  </Accordion>

提供商的 `apiKey` 字段可以是原始字符串或 SecretRef。在 Gateway 网关
冷启动期间，不可用的 TTS SecretRef 会将内置 TTS 能力标记为
“已配置但不可用”，而不会阻止 Gateway 网关启动。随后，`tts.speak` 返回
`UNAVAILABLE`，原因为 `SECRET_SURFACE_UNAVAILABLE`，并且不会
发送提供商请求。状态和 Doctor 会列出降级的 TTS 所有者及其配置路径。显式
引用会保留在运行时快照中，因此环境或配置文件中的
凭据无法静默选择其他账户。重新加载和配置写入
预检会应用可感知所有者的降级策略：未更改且符合条件的 TTS
所有者可以继续使用其最后已知有效的凭据作为过期凭据，而新的或已更改的
故障会转为冷状态，且不会阻止正常的所有者。结构无效的引用
和解析后的值仍会导致启动失败或更新被拒绝。

  <Accordion title="Azure Speech">
    <ParamField path="apiKey" type="string">环境变量：`AZURE_SPEECH_KEY`、`AZURE_SPEECH_API_KEY` 或 `SPEECH_KEY`。</ParamField>
    <ParamField path="region" type="string">Azure Speech 区域（例如 `eastus`）。环境变量：`AZURE_SPEECH_REGION` 或 `SPEECH_REGION`。</ParamField>
    <ParamField path="endpoint" type="string">可选的 Azure Speech 端点覆盖项（别名为 `baseUrl`）。</ParamField>
    <ParamField path="speakerVoice" type="string">Azure 语音 ShortName。默认为 `en-US-JennyNeural`。旧版别名：`voice`。</ParamField>
    <ParamField path="lang" type="string">SSML 语言代码。默认为 `en-US`。</ParamField>
    <ParamField path="outputFormat" type="string">用于标准音频的 Azure `X-Microsoft-OutputFormat`。默认为 `audio-24khz-48kbitrate-mono-mp3`。</ParamField>
    <ParamField path="voiceNoteOutputFormat" type="string">用于语音留言输出的 Azure `X-Microsoft-OutputFormat`。默认为 `ogg-24khz-16bit-mono-opus`。</ParamField>
  </Accordion>

  <Accordion title="ElevenLabs">
    <ParamField path="apiKey" type="string">回退到 `ELEVENLABS_API_KEY` 或 `XI_API_KEY`。</ParamField>
    <ParamField path="model" type="string">模型 ID。默认为 `eleven_multilingual_v2`。旧版 ID `eleven_turbo_v2_5`/`eleven_turbo_v2` 会标准化为匹配的 `flash` 模型。</ParamField>
    <ParamField path="speakerVoiceId" type="string">ElevenLabs 语音 ID。默认为 `pMsXgVXv3BLzUgSXRplE`。旧版别名：`voiceId`。</ParamField>
    <ParamField path="voiceSettings" type="object">
      `stability`、`similarityBoost`、`style`（每项均为 `0..1`，默认值分别为 `0.5`/`0.75`/`0`）、`useSpeakerBoost`（`true|false`，默认为 `true`）、`speed`（`0.5..2.0`，默认为 `1.0`）。
    </ParamField>
    <ParamField path="applyTextNormalization" type='"auto" | "on" | "off"'>文本标准化模式。</ParamField>
    <ParamField path="languageCode" type="string">双字母 ISO 639-1 代码（例如 `en`、`de`）。</ParamField>
    <ParamField path="seed" type="number">用于尽力实现确定性的整数 `0..4294967295`。</ParamField>
    <ParamField path="baseUrl" type="string">覆盖 ElevenLabs API 基础 URL。</ParamField>
  </Accordion>

  <Accordion title="Google Gemini">
    <ParamField path="apiKey" type="string">回退到 `GEMINI_API_KEY` / `GOOGLE_API_KEY`。如果省略，TTS 可以在回退到环境变量之前复用 `models.providers.google.apiKey`。</ParamField>
    <ParamField path="model" type="string">Gemini TTS 模型。默认值为 `gemini-3.1-flash-tts-preview`。</ParamField>
    <ParamField path="speakerVoice" type="string">Gemini 预构建语音名称。默认值为 `Kore`。旧版别名：`voiceName`、`voice`。</ParamField>
    <ParamField path="audioProfile" type="string">添加在朗读文本之前的自然语言风格提示词。</ParamField>
    <ParamField path="speakerName" type="string">可选的说话者标签；当提示词使用具名说话者时，添加在朗读文本之前。</ParamField>
    <ParamField path="promptTemplate" type='"audio-profile-v1"'>设为 `audio-profile-v1`，以确定性的 Gemini TTS 提示词结构封装当前角色提示词字段。</ParamField>
    <ParamField path="personaPrompt" type="string">附加到模板中 Director's Notes 的 Google 专用额外角色提示词文本。</ParamField>
    <ParamField path="baseUrl" type="string">仅接受 `https://generativelanguage.googleapis.com`。</ParamField>
  </Accordion>

  <Accordion title="Gradium">
    <ParamField path="apiKey" type="string">环境变量：`GRADIUM_API_KEY`。</ParamField>
    <ParamField path="baseUrl" type="string">位于 `api.gradium.ai` 的 HTTPS Gradium API URL。默认值为 `https://api.gradium.ai`。</ParamField>
    <ParamField path="speakerVoiceId" type="string">默认值为 Emma（`YTpq7expH9539ERJ`）。旧版别名：`voiceId`。</ParamField>
  </Accordion>

  <Accordion title="Inworld">
    ### Inworld 主配置

    <ParamField path="apiKey" type="string">环境变量：`INWORLD_API_KEY`。</ParamField>
    <ParamField path="baseUrl" type="string">默认值为 `https://api.inworld.ai`。</ParamField>
    <ParamField path="modelId" type="string">默认值为 `inworld-tts-1.5-max`。还支持：`inworld-tts-1.5-mini`、`inworld-tts-1-max`、`inworld-tts-1`。</ParamField>
    <ParamField path="speakerVoiceId" type="string">默认值为 `Sarah`。旧版别名：`voiceId`。</ParamField>
    <ParamField path="temperature" type="number">采样温度 `0..2`（不含 0）。</ParamField>

  </Accordion>

  <Accordion title="本地 CLI（tts-local-cli）">
    <ParamField path="command" type="string">用于 CLI TTS 的本地可执行文件或命令字符串。</ParamField>
    <ParamField path="args" type="string[]">命令参数。支持 `{{Text}}`、`{{OutputPath}}`、`{{OutputDir}}`、`{{OutputBase}}` 占位符。</ParamField>
    <ParamField path="outputFormat" type='"mp3" | "opus" | "wav"'>预期的 CLI 输出格式。音频附件的默认值为 `mp3`。</ParamField>
    <ParamField path="timeoutMs" type="number">命令超时时间（毫秒）。默认值为 `120000`。</ParamField>
    <ParamField path="cwd" type="string">可选的命令工作目录。</ParamField>
    <ParamField path="env" type="Record<string, string>">可选的命令环境变量覆盖项。</ParamField>

    命令标准输出以及生成或转换后的音频限制为 50 MiB。诊断标准错误输出限制为 1 MiB。超过任一限制时，OpenClaw 会终止命令并使合成失败。

  </Accordion>

  <Accordion title="Microsoft（无需 API key）">
    <ParamField path="enabled" type="boolean" default="true">允许使用 Microsoft 语音。</ParamField>
    <ParamField path="speakerVoice" type="string">Microsoft 神经语音名称（例如 `en-US-MichelleNeural`）。旧版别名：`voice`。如果正在使用默认英语语音，并且回复文本以 CJK 字符为主，OpenClaw 会自动切换到 `zh-CN-XiaoxiaoNeural`。</ParamField>
    <ParamField path="lang" type="string">语言代码（例如 `en-US`）。</ParamField>
    <ParamField path="outputFormat" type="string">Microsoft 输出格式。默认值为 `audio-24khz-48kbitrate-mono-mp3`。内置的 Edge 后端传输并不支持所有格式。</ParamField>
    <ParamField path="rate / pitch / volume" type="string">百分比字符串（例如 `+10%`、`-5%`）。</ParamField>
    <ParamField path="saveSubtitles" type="boolean">在音频文件旁写入 JSON 字幕。</ParamField>
    <ParamField path="proxy" type="string">Microsoft 语音请求使用的代理 URL。</ParamField>
    <ParamField path="timeoutMs" type="number">请求超时覆盖值（毫秒）。</ParamField>
    <ParamField path="edge.*" type="object" deprecated>旧版别名。运行 `openclaw doctor --fix`，将持久化配置重写为 `providers.microsoft`。</ParamField>
  </Accordion>

  <Accordion title="MiniMax">
    <ParamField path="apiKey" type="string">回退到 `MINIMAX_API_KEY`。通过 `MINIMAX_OAUTH_TOKEN`、`MINIMAX_CODE_PLAN_KEY` 或 `MINIMAX_CODING_API_KEY` 进行 Token Plan 身份验证。</ParamField>
    <ParamField path="baseUrl" type="string">默认值为 `https://api.minimax.io`。环境变量：`MINIMAX_API_HOST`。</ParamField>
    <ParamField path="model" type="string">默认值为 `speech-2.8-hd`。环境变量：`MINIMAX_TTS_MODEL`。</ParamField>
    <ParamField path="speakerVoiceId" type="string">默认值为 `English_expressive_narrator`。环境变量：`MINIMAX_TTS_VOICE_ID`。旧版别名：`voiceId`。</ParamField>
    <ParamField path="speed" type="number">`0.5..2.0`。默认值为 `1.0`。</ParamField>
    <ParamField path="vol" type="number">`(0, 10]`。默认值为 `1.0`。</ParamField>
    <ParamField path="pitch" type="number">整数 `-12..12`。默认值为 `0`。发送请求前会截断小数值。</ParamField>
  </Accordion>

  <Accordion title="OpenAI">
    <ParamField path="apiKey" type="string">回退到 `OPENAI_API_KEY`。</ParamField>
    <ParamField path="model" type="string">OpenAI TTS 模型 ID。默认值为 `gpt-4o-mini-tts`。</ParamField>
    <ParamField path="speakerVoice" type="string">语音名称（例如 `alloy`、`cedar`）。默认值为 `coral`。旧版别名：`voice`。</ParamField>
    <ParamField path="instructions" type="string">显式的 OpenAI `instructions` 字段。设置后，角色提示词字段**不会**自动映射。</ParamField>
    <ParamField path="extraBody / extra_body" type="Record<string, unknown>">生成 OpenAI TTS 字段后，合并到 `/audio/speech` 请求体中的额外 JSON 字段。对于 Kokoro 等需要 `lang` 这类提供商专用键的 OpenAI 兼容端点，请使用此字段；不安全的原型键会被忽略。</ParamField>
    <ParamField path="baseUrl" type="string">
      覆盖 OpenAI TTS 端点。解析顺序：配置 → `OPENAI_TTS_BASE_URL` → `https://api.openai.com/v1`。非默认值会被视为 OpenAI 兼容的 TTS 端点，因此接受自定义模型和语音名称，并且 `speed` 不再进行 `0.25..4.0` 范围检查。
    </ParamField>
  </Accordion>

  <Accordion title="OpenRouter">
    <ParamField path="apiKey" type="string">环境变量：`OPENROUTER_API_KEY`。可以复用 `models.providers.openrouter.apiKey`。</ParamField>
    <ParamField path="baseUrl" type="string">默认值为 `https://openrouter.ai/api/v1`。旧版 `https://openrouter.ai/v1` 会被规范化。</ParamField>
    <ParamField path="model" type="string">默认值为 `hexgrad/kokoro-82m`。别名：`modelId`。</ParamField>
    <ParamField path="speakerVoice" type="string">默认值为 `af_alloy`。旧版别名：`voice`、`voiceId`。</ParamField>
    <ParamField path="responseFormat" type='"mp3" | "pcm"'>默认值为 `mp3`。</ParamField>
    <ParamField path="speed" type="number">提供商原生速度覆盖值。</ParamField>
  </Accordion>

  <Accordion title="Volcengine（BytePlus Seed Speech）">
    <ParamField path="apiKey" type="string">环境变量：`VOLCENGINE_TTS_API_KEY` 或 `BYTEPLUS_SEED_SPEECH_API_KEY`。</ParamField>
    <ParamField path="resourceId" type="string">默认值为 `seed-tts-1.0`。环境变量：`VOLCENGINE_TTS_RESOURCE_ID`。当项目具有 TTS 2.0 权限时，使用 `seed-tts-2.0`。</ParamField>
    <ParamField path="appKey" type="string">App key 请求头。默认值为 `aGjiRDfUWi`。环境变量：`VOLCENGINE_TTS_APP_KEY`。</ParamField>
    <ParamField path="baseUrl" type="string">覆盖 Seed Speech TTS HTTP 端点。环境变量：`VOLCENGINE_TTS_BASE_URL`。</ParamField>
    <ParamField path="speakerVoice" type="string">语音类型。默认值为 `en_female_anna_mars_bigtts`。环境变量：`VOLCENGINE_TTS_VOICE`。旧版别名：`voice`。</ParamField>
    <ParamField path="speedRatio" type="number">提供商原生速度比率，`0.2..3`。</ParamField>
    <ParamField path="emotion" type="string">提供商原生情感标签。</ParamField>
    <ParamField path="appId / token / cluster" type="string" deprecated>旧版 Volcengine Speech Console 字段。环境变量：`VOLCENGINE_TTS_APPID`、`VOLCENGINE_TTS_TOKEN`、`VOLCENGINE_TTS_CLUSTER`（默认值为 `volcano_tts`）。</ParamField>
  </Accordion>

  <Accordion title="xAI">
    <ParamField path="apiKey" type="string">环境变量：`XAI_API_KEY`。</ParamField>
    <ParamField path="baseUrl" type="string">默认值为 `https://api.x.ai/v1`。环境变量：`XAI_BASE_URL`。</ParamField>
    <ParamField path="speakerVoiceId" type="string">默认值为 `eve`。进行身份验证后，`openclaw infer tts voices --provider xai` 会获取当前内置目录；未进行身份验证时，它会列出离线回退项 `ara`、`eve`、`leo`、`rex` 和 `sal`。即使账号自定义语音 ID 不在内置列表中，也会照常转发。旧版别名：`voiceId`。</ParamField>
    <ParamField path="language" type="string">BCP-47 语言代码或 `auto`。默认值为 `en`。</ParamField>
    <ParamField path="responseFormat" type='"mp3" | "wav" | "pcm" | "mulaw" | "alaw"'>默认值为 `mp3`。</ParamField>
    <ParamField path="speed" type="number">提供商原生速度覆盖值，`0.7..1.5`。</ParamField>
  </Accordion>

  <Accordion title="Xiaomi MiMo">
    <ParamField path="apiKey" type="string">环境变量：`XIAOMI_API_KEY`。</ParamField>
    <ParamField path="baseUrl" type="string">默认值为 `https://api.xiaomimimo.com/v1`。环境变量：`XIAOMI_BASE_URL`。</ParamField>
    <ParamField path="model" type="string">默认值为 `mimo-v2.5-tts`。环境变量：`XIAOMI_TTS_MODEL`。还支持 `mimo-v2.5-tts-voicedesign`。</ParamField>
    <ParamField path="speakerVoice" type="string">预设语音模型的默认值为 `mimo_default`。环境变量：`XIAOMI_TTS_VOICE`。旧版别名：`voice`。使用 `mimo-v2.5-tts-voicedesign` 时不发送此字段。</ParamField>
    <ParamField path="format" type='"mp3" | "wav"'>默认值为 `mp3`。环境变量：`XIAOMI_TTS_FORMAT`。</ParamField>
    <ParamField path="style" type="string">作为用户消息发送的可选自然语言风格指令；不会被朗读。对于 `mimo-v2.5-tts-voicedesign`，这是语音设计提示词；省略时，OpenClaw 会提供默认值。</ParamField>
  </Accordion>
</AccordionGroup>

## Agent 工具

`tts` 工具将文本转换为语音，并返回音频附件用于
发送回复。在 Feishu、Matrix、Telegram 和 WhatsApp 上，音频会
作为语音消息而非文件附件发送。当 `ffmpeg` 可用时，Feishu 和
WhatsApp 可以在此路径上对非 Opus TTS 输出进行转码。

WhatsApp 通过 Baileys 将音频作为 PTT 语音便笺发送（`audio`，并带有
`ptt: true`），并将可见文本与 PTT 音频**分开发送**，因为
客户端无法始终如一地显示语音便笺的说明文字。

该工具接受可选的 `channel` 和 `timeoutMs` 字段；`timeoutMs` 是
每次调用的提供商请求超时时间（毫秒）。每次调用的值会覆盖
`tts.timeoutMs`；已配置的 TTS 超时时间会覆盖插件指定的任何
提供商默认值。

## Gateway RPC 参考

| 方法              | 用途                                         |
| ----------------- | -------------------------------------------- |
| `tts.status`      | 读取当前 TTS 状态和上次尝试结果。            |
| `tts.enable`      | 将本地自动偏好设置为 `always`。    |
| `tts.disable`     | 将本地自动偏好设置为 `off`。       |
| `tts.convert`     | 一次性将文本转换为音频。                     |
| `tts.setProvider` | 设置本地提供商偏好。                         |
| `tts.personas`    | 列出已配置的角色及当前启用的角色。           |
| `tts.setPersona`  | 设置本地角色偏好。                           |
| `tts.providers`   | 列出已配置的提供商及其状态。                 |

## 服务链接

- [OpenAI 文本转语音指南](https://platform.openai.com/docs/guides/text-to-speech)
- [OpenAI Audio API 参考](https://platform.openai.com/docs/api-reference/audio)
- [Azure Speech REST 文本转语音](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech)
- [Azure Speech provider](/zh-CN/providers/azure-speech)
- [ElevenLabs 文本转语音](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [ElevenLabs 身份验证](https://elevenlabs.io/docs/api-reference/authentication)
- [Gradium](/zh-CN/providers/gradium)
- [Inworld TTS API](https://docs.inworld.ai/tts/tts)
- [MiniMax T2A v2 API](https://platform.minimaxi.com/document/T2A%20V2)
- [Volcengine TTS HTTP API](/zh-CN/providers/volcengine#text-to-speech)
- [小米 MiMo 语音合成](/zh-CN/providers/xiaomi#text-to-speech)
- [node-edge-tts](https://github.com/SchneeHertz/node-edge-tts)
- [Microsoft Speech 输出格式](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech#audio-outputs)
- [xAI 文本转语音](https://docs.x.ai/developers/rest-api-reference/inference/voice#text-to-speech-rest)

## 相关内容

- [媒体概览](/zh-CN/tools/media-overview)
- [音乐生成](/zh-CN/tools/music-generation)
- [视频生成](/zh-CN/tools/video-generation)
- [斜杠命令](/zh-CN/tools/slash-commands)
- [语音通话插件](/zh-CN/plugins/voice-call)
