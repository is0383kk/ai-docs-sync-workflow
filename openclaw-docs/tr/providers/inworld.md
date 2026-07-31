---
read_when:
    - Giden yanıtlar için Inworld konuşma sentezi istiyorsunuz
    - Inworld'den PCM telefon sesi veya OGG_OPUS sesli not çıktısına ihtiyacınız var
summary: OpenClaw yanıtları için Inworld akışlı metinden konuşmaya dönüştürme
title: Inworld
x-i18n:
    generated_at: "2026-07-26T23:58:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09560f5beda3b40d9c67f9408d34446f28ecddb8235fc0725c4265c813302946
    source_path: providers/inworld.md
    workflow: 16
---

Inworld, akışlı bir metinden konuşmaya (TTS) sağlayıcısıdır. OpenClaw'da giden yanıt sesini (varsayılan olarak MP3, sesli notlar için OGG_OPUS) ve Voice Call gibi telefon kanalları için ham PCM sesini sentezler.

OpenClaw, Inworld'ün akışlı TTS uç noktasına istek gönderir, döndürülen base64 ses parçalarını tek bir arabellekte birleştirir ve sonucu standart yanıt sesi işlem hattına aktarır.

| Özellik       | Değer                                                           |
| ------------- | --------------------------------------------------------------- |
| Sağlayıcı kimliği | `inworld`                                                       |
| Plugin        | resmî harici paket (`@openclaw/inworld-speech`)          |
| Sözleşme      | `speechProviders` (yalnızca TTS)                                    |
| Kimlik doğrulama ortam değişkeni | `INWORLD_API_KEY` (HTTP Basic, Base64 pano kimlik bilgisi)     |
| Temel URL     | `https://api.inworld.ai`                                        |
| Varsayılan ses | `Sarah`                                                         |
| Varsayılan model | `inworld-tts-1.5-max`                                           |
| Çıktı         | MP3 (varsayılan), OGG_OPUS (sesli notlar), PCM 22050 Hz (telefon) |
| Web sitesi    | [inworld.ai](https://inworld.ai)                                |
| Belgeler      | [docs.inworld.ai/tts/tts](https://docs.inworld.ai/tts/tts)      |

## Plugin'i yükleme

```bash
openclaw plugins install @openclaw/inworld-speech
openclaw gateway restart
```

## Başlarken

<Steps>
  <Step title="API anahtarınızı ayarlayın">
    Kimlik bilgisini Inworld panonuzdan (Workspace > API Keys) kopyalayın ve ortam değişkeni olarak ayarlayın. Değer, HTTP Basic kimlik bilgisi olarak olduğu gibi gönderilir; bu nedenle yeniden Base64 ile kodlamayın veya bearer token'a dönüştürmeyin.

    ```bash
    INWORLD_API_KEY=<base64-credential-from-dashboard>
    ```

  </Step>
  <Step title="tts içinde Inworld'ü seçin">
    ```json5
    {
      tts: {
        auto: "always",
        provider: "inworld",
        providers: {
          inworld: {
            voiceId: "Sarah",
            modelId: "inworld-tts-1.5-max",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="Mesaj gönderin">
    Bağlı herhangi bir kanal üzerinden yanıt gönderin. OpenClaw, sesi Inworld ile sentezler ve MP3 olarak (veya kanal sesli not beklediğinde OGG_OPUS olarak) iletir.
  </Step>
</Steps>

## Yapılandırma seçenekleri

| Seçenek       | Yol                                 | Açıklama                                                            |
| ------------- | ----------------------------------- | ------------------------------------------------------------------- |
| `apiKey`      | `tts.providers.inworld.apiKey`      | Base64 pano kimlik bilgisi. `INWORLD_API_KEY` değerine geri döner.       |
| `baseUrl`     | `tts.providers.inworld.baseUrl`     | Inworld API temel URL'sini geçersiz kılar (varsayılan `https://api.inworld.ai`).   |
| `voiceId`     | `tts.providers.inworld.voiceId`     | Ses tanımlayıcısı (varsayılan `Sarah`). Eski diğer ad: `speakerVoiceId`. |
| `modelId`     | `tts.providers.inworld.modelId`     | TTS model kimliği (varsayılan `inworld-tts-1.5-max`).                       |
| `temperature` | `tts.providers.inworld.temperature` | Örnekleme sıcaklığı, `0` (hariç) ile `2` arasında (isteğe bağlı).            |

## Notlar

<AccordionGroup>
  <Accordion title="Kimlik doğrulama">
    Inworld, Base64 ile kodlanmış tek bir kimlik bilgisi dizesiyle HTTP Basic kimlik doğrulaması kullanır. Bunu Inworld panosundan olduğu gibi kopyalayın. Sağlayıcı, herhangi bir ek kodlama yapmadan bunu `Authorization: Basic <apiKey>` olarak gönderir; bu nedenle kendiniz Base64 ile kodlamayın ve bearer tarzı bir token iletmeyin. Aynı uyarı için [TTS kimlik doğrulama notlarına](/tr/tools/tts#inworld-primary) bakın.
  </Accordion>
  <Accordion title="Modeller">
    Desteklenen model kimlikleri: `inworld-tts-1.5-max` (varsayılan), `inworld-tts-1.5-mini`, `inworld-tts-1-max`, `inworld-tts-1`.
  </Accordion>
  <Accordion title="Ses çıktıları">
    Yanıtlar varsayılan olarak MP3 kullanır. Kanal hedefi `voice-note` olduğunda OpenClaw, sesin yerel bir ses balonu olarak oynatılması için Inworld'den `OGG_OPUS` ister. Telefon sentezi, telefon köprüsünü beslemek için 22050 Hz'de ham `PCM` kullanır.
  </Accordion>
  <Accordion title="Özel uç noktalar">
    `tts.providers.inworld.baseUrl` ile API ana makinesini geçersiz kılın. İstekler gönderilmeden önce sondaki eğik çizgiler kaldırılır.
  </Accordion>
</AccordionGroup>

## İlgili

<CardGroup cols={2}>
  <Card title="Metinden konuşmaya" href="/tr/tools/tts" icon="waveform-lines">
    TTS genel bakışı, sağlayıcılar ve `tts` yapılandırması.
  </Card>
  <Card title="Yapılandırma" href="/tr/gateway/configuration" icon="gear">
    `tts` ayarlarını içeren tam yapılandırma başvurusu.
  </Card>
  <Card title="Sağlayıcılar" href="/tr/providers" icon="grid">
    Desteklenen tüm OpenClaw sağlayıcıları.
  </Card>
  <Card title="Sorun giderme" href="/tr/help/troubleshooting" icon="wrench">
    Yaygın sorunlar ve hata ayıklama adımları.
  </Card>
</CardGroup>
