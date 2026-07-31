---
read_when:
    - Je wilt SenseAudio-spraak-naar-tekst voor audiobijlagen
    - Je hebt de omgevingsvariabele voor de SenseAudio-API-sleutel of het pad naar de audioconfiguratie nodig
summary: SenseAudio-batchspraak-naar-tekst voor inkomende spraakberichten
title: SenseAudio
x-i18n:
    generated_at: "2026-07-27T06:31:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0ca4a31a32eed85c1d9dcd13ebc2eaea94be370d2b1013ae8b4677949bea91d
    source_path: providers/senseaudio.md
    workflow: 16
---

SenseAudio transcribeert inkomende audio- en spraaknotitiebijlagen via de gedeelde `tools.media.audio`-pijplijn van OpenClaw. OpenClaw verzendt multipart-audio naar het OpenAI-compatibele transcriptie-eindpunt en voegt de geretourneerde tekst in als `{{Transcript}}` plus een `[Audio]`-blok.

| Eigenschap    | Waarde                                           |
| ------------- | ------------------------------------------------ |
| Provider-id   | `senseaudio`                               |
| Plugin        | meegeleverd, `enabledByDefault: true`                  |
| Contract      | `mediaUnderstandingProviders` (audio)                       |
| Auth-omgevingsvariabele | `SENSEAUDIO_API_KEY`                     |
| Standaardmodel | `senseaudio-asr-pro-1.5-260319`                              |
| Standaard-URL | `https://api.senseaudio.cn/v1`                               |
| Website       | [senseaudio.cn](https://senseaudio.cn)           |
| Documentatie  | [senseaudio.cn/docs](https://senseaudio.cn/docs) |

## Aan de slag

<Steps>
  <Step title="Stel je API-sleutel in">
    ```bash
    export SENSEAUDIO_API_KEY="..."
    ```
  </Step>
  <Step title="Schakel de audioprovider in">
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
  <Step title="Stuur een spraaknotitie">
    Stuur een audiobericht via een verbonden kanaal. OpenClaw uploadt de
    audio naar SenseAudio en gebruikt de transcriptie in de antwoordpijplijn.
  </Step>
</Steps>

## Opties

| Optie      | Pad                             | Beschrijving                        |
| ---------- | ------------------------------- | ----------------------------------- |
| `model`    | `tools.media.models[].model`    | SenseAudio ASR-model-id             |
| `language` | `tools.media.models[].language` | Optionele taalaanwijzing            |
| `prompt`   | `tools.media.models[].prompt`   | Optionele transcriptieprompt        |
| `baseUrl`  | `tools.media.models[].baseUrl`  | Overschrijf de OpenAI-compatibele basis |
| `headers`  | `tools.media.models[].headers`  | Extra aanvraagheaders               |

<Note>
SenseAudio ondersteunt in OpenClaw alleen batch-STT. Realtime transcriptie voor Voice Call
blijft providers met ondersteuning voor streaming-STT gebruiken.
</Note>

## Gerelateerd

- [Media begrijpen (audio)](/nl/nodes/audio)
- [Modelproviders](/nl/concepts/model-providers)
