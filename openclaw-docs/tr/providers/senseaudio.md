---
read_when:
    - Ses ekleri için SenseAudio konuşmayı metne dönüştürme özelliğini istiyorsunuz
    - SenseAudio API anahtarı ortam değişkeni veya ses yapılandırma yolu gereklidir
summary: Gelen sesli notlar için SenseAudio toplu konuşmadan metne dönüştürme
title: SenseAudio
x-i18n:
    generated_at: "2026-07-27T00:15:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0ca4a31a32eed85c1d9dcd13ebc2eaea94be370d2b1013ae8b4677949bea91d
    source_path: providers/senseaudio.md
    workflow: 16
---

SenseAudio, gelen sesleri ve sesli not eklerini OpenClaw'ın paylaşılan `tools.media.audio` işlem hattı üzerinden yazıya döker. OpenClaw, çok parçalı sesi OpenAI uyumlu transkripsiyon uç noktasına gönderir ve döndürülen metni `{{Transcript}}` olarak, ayrıca bir `[Audio]` bloğu içinde ekler.

| Özellik       | Değer                                            |
| ------------- | ------------------------------------------------ |
| Sağlayıcı kimliği | `senseaudio`                                     |
| Plugin        | paketle birlikte gelen, `enabledByDefault: true`                |
| Sözleşme      | `mediaUnderstandingProviders` (ses)            |
| Kimlik doğrulama ortam değişkeni | `SENSEAUDIO_API_KEY`                             |
| Varsayılan model | `senseaudio-asr-pro-1.5-260319`                  |
| Varsayılan URL | `https://api.senseaudio.cn/v1`                   |
| Web sitesi    | [senseaudio.cn](https://senseaudio.cn)           |
| Belgeler      | [senseaudio.cn/docs](https://senseaudio.cn/docs) |

## Başlarken

<Steps>
  <Step title="API anahtarınızı ayarlayın">
    ```bash
    export SENSEAUDIO_API_KEY="..."
    ```
  </Step>
  <Step title="Ses sağlayıcısını etkinleştirin">
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
  <Step title="Sesli not gönderin">
    Bağlı herhangi bir kanal üzerinden sesli mesaj gönderin. OpenClaw, sesi
    SenseAudio'ya yükler ve transkripti yanıt işlem hattında kullanır.
  </Step>
</Steps>

## Seçenekler

| Seçenek    | Yol                             | Açıklama                            |
| ---------- | ------------------------------- | ----------------------------------- |
| `model`    | `tools.media.models[].model`    | SenseAudio ASR model kimliği        |
| `language` | `tools.media.models[].language` | İsteğe bağlı dil ipucu              |
| `prompt`   | `tools.media.models[].prompt`   | İsteğe bağlı transkripsiyon istemi  |
| `baseUrl`  | `tools.media.models[].baseUrl`  | OpenAI uyumlu tabanı geçersiz kılma |
| `headers`  | `tools.media.models[].headers`  | Ek istek üst bilgileri              |

<Note>
SenseAudio, OpenClaw'da yalnızca toplu STT olarak kullanılabilir. Voice Call gerçek zamanlı transkripsiyonu,
akışlı STT desteğine sahip sağlayıcıları kullanmaya devam eder.
</Note>

## İlgili

- [Medya anlama (ses)](/tr/nodes/audio)
- [Model sağlayıcıları](/tr/concepts/model-providers)
