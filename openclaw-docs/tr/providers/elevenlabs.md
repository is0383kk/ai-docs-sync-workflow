---
read_when:
    - OpenClaw'da ElevenLabs metinden konuşmaya özelliğini kullanmak istiyorsunuz
    - Ses ekleri için ElevenLabs Scribe konuşmadan metne dönüştürme özelliğini kullanmak istiyorsunuz
    - Voice Call veya Google Meet için ElevenLabs gerçek zamanlı transkripsiyonunu istiyorsunuz
summary: OpenClaw ile ElevenLabs konuşma, Scribe STT ve gerçek zamanlı transkripsiyon kullanın
title: ElevenLabs
x-i18n:
    generated_at: "2026-07-27T00:14:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5c570aab5fd3ca00e8ded8e3daa143cb199334d507461800ec0b6c1ab0b65c59
    source_path: providers/elevenlabs.md
    workflow: 16
---

OpenClaw, metinden konuşmaya, Scribe v2 ile toplu konuşmadan metne ve
Scribe v2 Realtime ile akışlı STT için ElevenLabs kullanır. Plugin paketle birlikte gelir ve
varsayılan olarak etkindir; `plugins install` adımı gerekmez.

| Yetenek                  | OpenClaw yüzeyi                                                      | Varsayılan               |
| ------------------------ | -------------------------------------------------------------------- | ------------------------ |
| Metinden konuşmaya       | `tts` / `talk`                                                       | `eleven_multilingual_v2` |
| Toplu konuşmadan metne   | `tools.media.audio`                                                  | `scribe_v2`              |
| Akışlı konuşmadan metne  | Sesli Arama akışı veya Google Meet `realtime.transcriptionProvider` | `scribe_v2_realtime`     |

## Kimlik doğrulama

Ortamda `ELEVENLABS_API_KEY` ayarlayın. Mevcut ElevenLabs araçlarıyla
uyumluluk için `XI_API_KEY` da kabul edilir.

```bash
export ELEVENLABS_API_KEY="..."
```

## Metinden konuşmaya

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

ElevenLabs v3 TTS kullanmak için `modelId` değerini `eleven_v3` olarak ayarlayın. OpenClaw, mevcut kurulumlar için
`eleven_multilingual_v2` değerini varsayılan olarak korur.

ElevenLabs, seçilen `voice.tts`/`tts` sağlayıcısı olduğunda Discord
ses kanalları ElevenLabs'in akışlı TTS uç noktasını kullanır: oynatma, OpenClaw'ın önce
ses dosyasının tamamını indirmesini beklemek yerine döndürülen ses akışından başlar.
`latencyTier`, bunu kabul eden modeller için ElevenLabs'in `optimize_streaming_latency`
sorgu parametresine eşlenir; OpenClaw, bunu reddeden `eleven_v3` için
bu parametreyi atlar.

## Konuşmadan metne

Gelen ses ekleri ve kısa kaydedilmiş ses parçaları için Scribe v2 kullanın:

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

OpenClaw, çok parçalı sesi `model_id: "scribe_v2"` ile ElevenLabs
`/v1/speech-to-text` konumuna gönderir. Dil ipuçları, mevcut olduğunda `language_code` değerine eşlenir.

## Akışlı STT

Paketle birlikte gelen `elevenlabs` Plugin, Sesli Arama ve Google Meet
aracı modu akışlı transkripsiyonu için Scribe v2 Realtime'ı kaydeder.

| Ayar            | Yapılandırma yolu                                                        | Varsayılan                                        |
| --------------- | ------------------------------------------------------------------------- | ------------------------------------------------- |
| API anahtarı    | `plugins.entries.voice-call.config.streaming.providers.elevenlabs.apiKey` | `ELEVENLABS_API_KEY` / `XI_API_KEY` değerlerine geri döner |
| Model           | `...elevenlabs.modelId`                                                   | `scribe_v2_realtime`                              |
| Ses biçimi      | `...elevenlabs.audioFormat`                                               | `ulaw_8000`                                       |
| Örnekleme hızı  | `...elevenlabs.sampleRate`                                                | `8000`                                            |
| İşleme stratejisi | `...elevenlabs.commitStrategy`                                            | `vad`                                             |
| Dil             | `...elevenlabs.languageCode`                                              | (ayarlanmamış)                                    |

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
Sesli Arama, Twilio medyasını 8 kHz G.711 u-law olarak alır. ElevenLabs gerçek zamanlı
sağlayıcısı varsayılan olarak `ulaw_8000` değerini kullanır; böylece telefon çerçeveleri
kod dönüştürme olmadan iletilebilir.
</Note>

Google Meet aracı modu için
`plugins.entries.google-meet.config.realtime.transcriptionProvider` değerini
`"elevenlabs"` olarak ayarlayın ve aynı sağlayıcı bloğunu
`plugins.entries.google-meet.config.realtime.providers.elevenlabs` altında yapılandırın.

## İlgili

- [Metinden konuşmaya](/tr/tools/tts)
- [Google Meet](/tr/plugins/google-meet)
- [Model seçimi](/tr/concepts/model-providers)
