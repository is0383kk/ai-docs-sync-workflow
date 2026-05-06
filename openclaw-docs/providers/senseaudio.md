> ## Documentation Index
> Fetch the complete documentation index at: https://docs.openclaw.ai/llms.txt
> Use this file to discover all available pages before exploring further.

# SenseAudio

SenseAudio can transcribe inbound audio and voice-note attachments through OpenClaw's shared `tools.media.audio` pipeline. OpenClaw posts multipart audio to the OpenAI-compatible transcription endpoint and injects the returned text as `{{Transcript}}` plus an `[Audio]` block.

| Property      | Value                                            |
| ------------- | ------------------------------------------------ |
| Provider id   | `senseaudio`                                     |
| Plugin        | bundled, `enabledByDefault: true`                |
| Contract      | `mediaUnderstandingProviders` (audio)            |
| Auth env var  | `SENSEAUDIO_API_KEY`                             |
| Default model | `senseaudio-asr-pro-1.5-260319`                  |
| Default URL   | `https://api.senseaudio.cn/v1`                   |
| Website       | [senseaudio.cn](https://senseaudio.cn)           |
| Docs          | [senseaudio.cn/docs](https://senseaudio.cn/docs) |

## Getting started

<Steps>
  <Step title="Set your API key">
    ```bash theme={"theme":{"light":"min-light","dark":"min-dark"}}
    export SENSEAUDIO_API_KEY="..."
    ```
  </Step>

  <Step title="Enable the audio provider">
    ```json5 theme={"theme":{"light":"min-light","dark":"min-dark"}}
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

  <Step title="Send a voice note">
    Send an audio message through any connected channel. OpenClaw uploads the
    audio to SenseAudio and uses the transcript in the reply pipeline.
  </Step>
</Steps>

## Options

| Option     | Path                                  | Description                         |
| ---------- | ------------------------------------- | ----------------------------------- |
| `model`    | `tools.media.audio.models[].model`    | SenseAudio ASR model id             |
| `language` | `tools.media.audio.models[].language` | Optional language hint              |
| `prompt`   | `tools.media.audio.prompt`            | Optional transcription prompt       |
| `baseUrl`  | `tools.media.audio.baseUrl` or model  | Override the OpenAI-compatible base |
| `headers`  | `tools.media.audio.request.headers`   | Extra request headers               |

<Note>
  SenseAudio is batch STT only in OpenClaw. Voice Call realtime transcription
  continues to use providers with streaming STT support.
</Note>

## Related

* [Media understanding (audio)](/nodes/audio)
* [Model providers](/concepts/model-providers)
