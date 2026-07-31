---
read_when:
    - Sie möchten SenseAudio für die Spracherkennung bei Audioanhängen verwenden
    - Sie benötigen die Umgebungsvariable für den SenseAudio-API-Schlüssel oder den Pfad zur Audiokonfiguration.
summary: SenseAudio-Batch-Spracherkennung für eingehende Sprachnachrichten
title: SenseAudio
x-i18n:
    generated_at: "2026-07-26T19:12:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0ca4a31a32eed85c1d9dcd13ebc2eaea94be370d2b1013ae8b4677949bea91d
    source_path: providers/senseaudio.md
    workflow: 16
---

SenseAudio transkribiert eingehende Audio- und Sprachnotizanhänge über die gemeinsame `tools.media.audio`-Pipeline von OpenClaw. OpenClaw sendet Audiodaten als Multipart-Anfrage an den OpenAI-kompatiblen Transkriptionsendpunkt und fügt den zurückgegebenen Text als `{{Transcript}}` sowie als `[Audio]`-Block ein.

| Eigenschaft   | Wert                                             |
| ------------- | ------------------------------------------------ |
| Provider-ID   | `senseaudio`                               |
| Plugin        | gebündelt, `enabledByDefault: true`                    |
| Vertrag       | `mediaUnderstandingProviders` (Audio)                       |
| Auth-Umgebungsvariable | `SENSEAUDIO_API_KEY`                      |
| Standardmodell | `senseaudio-asr-pro-1.5-260319`                              |
| Standard-URL  | `https://api.senseaudio.cn/v1`                               |
| Website       | [senseaudio.cn](https://senseaudio.cn)           |
| Dokumentation | [senseaudio.cn/docs](https://senseaudio.cn/docs) |

## Erste Schritte

<Steps>
  <Step title="API-Schlüssel festlegen">
    ```bash
    export SENSEAUDIO_API_KEY="..."
    ```
  </Step>
  <Step title="Audio-Provider aktivieren">
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
  <Step title="Sprachnotiz senden">
    Senden Sie eine Audionachricht über einen beliebigen verbundenen Kanal. OpenClaw lädt die
    Audiodaten zu SenseAudio hoch und verwendet das Transkript in der Antwort-Pipeline.
  </Step>
</Steps>

## Optionen

| Option     | Pfad                            | Beschreibung                        |
| ---------- | ------------------------------- | ----------------------------------- |
| `model`    | `tools.media.models[].model`    | SenseAudio-ASR-Modell-ID            |
| `language` | `tools.media.models[].language` | Optionaler Sprachhinweis             |
| `prompt`   | `tools.media.models[].prompt`   | Optionale Transkriptionsanweisung    |
| `baseUrl`  | `tools.media.models[].baseUrl`  | OpenAI-kompatible Basis überschreiben |
| `headers`  | `tools.media.models[].headers`  | Zusätzliche Anfrage-Header           |

<Note>
SenseAudio unterstützt in OpenClaw ausschließlich Batch-STT. Die Echtzeittranskription für Sprachanrufe
verwendet weiterhin Provider mit Unterstützung für Streaming-STT.
</Note>

## Verwandte Themen

- [Medienverständnis (Audio)](/de/nodes/audio)
- [Modell-Provider](/de/concepts/model-providers)
