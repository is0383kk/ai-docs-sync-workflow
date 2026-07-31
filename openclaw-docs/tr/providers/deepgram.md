---
read_when:
    - Ses ekleri için Deepgram konuşmayı metne dönüştürme özelliğini kullanmak istiyorsunuz
    - Voice Call için Deepgram akışlı transkripsiyonunu kullanmak istiyorsunuz
    - Hızlı bir Deepgram yapılandırma örneğine ihtiyacınız var
summary: Gelen sesli notlar için Deepgram transkripsiyonu
title: Deepgram
x-i18n:
    generated_at: "2026-07-26T23:36:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c00473762c3bede1f6de9230043827d90daefd68d05e67ed4b3e3026b9d6ba4f
    source_path: providers/deepgram.md
    workflow: 16
---

Deepgram bir konuşmayı metne dönüştürme API'sidir. OpenClaw bunu `tools.media.audio` üzerinden gelen ses/sesli not
transkripsiyonu ve `plugins.entries.voice-call.config.streaming` üzerinden Sesli Arama akış STT'si
için kullanır.

Toplu transkripsiyon, ses dosyasının tamamını Deepgram'a yükler ve
transkripti yanıt işlem hattına ekler (`{{Transcript}}` + `[Audio]` bloğu).
Sesli Arama akışı, canlı G.711 u-law karelerini Deepgram'ın
WebSocket `listen` uç noktası üzerinden iletir ve Deepgram bunları
döndürdükçe kısmi/nihai transkriptleri yayımlar.

| Ayrıntı       | Değer                                                      |
| ------------- | ---------------------------------------------------------- |
| Web sitesi    | [deepgram.com](https://deepgram.com)                       |
| Belgeler      | [developers.deepgram.com](https://developers.deepgram.com) |
| Kimlik doğrulama | `DEEPGRAM_API_KEY`                                      |
| Varsayılan model | `nova-3`                                      |

## Başlarken

<Steps>
  <Step title="API anahtarınızı ayarlayın">
    ```bash
    DEEPGRAM_API_KEY=dg_...
    ```
  </Step>
  <Step title="Ses sağlayıcısını etkinleştirin">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [{ provider: "deepgram", model: "nova-3" }],
          },
        },
      },
    }
    ```
  </Step>
  <Step title="Sesli not gönderin">
    Bağlı herhangi bir kanal üzerinden sesli mesaj gönderin. OpenClaw bunu
    Deepgram aracılığıyla metne dönüştürür ve transkripti yanıt işlem hattına ekler.
  </Step>
</Steps>

## Yapılandırma seçenekleri

| Seçenek    | Yol                             | Açıklama                              |
| ---------- | ------------------------------- | ------------------------------------- |
| `model`    | `tools.media.models[].model`    | Deepgram model kimliği (varsayılan: `nova-3`) |
| `language` | `tools.media.models[].language` | Dil ipucu (isteğe bağlı)              |

`providerOptions.deepgram`, ek sorgu parametrelerini doğrudan
Deepgram `/listen` isteğiyle birleştirir; dolayısıyla Deepgram'ın desteklediği tüm parametre adları kullanılabilir
(örneğin `detect_language`, `punctuate`, `smart_format`):

<Tabs>
  <Tab title="Dil ipucuyla">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [{ provider: "deepgram", model: "nova-3", language: "en" }],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Deepgram seçenekleriyle">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            providerOptions: {
              deepgram: {
                detect_language: true,
                punctuate: true,
                smart_format: true,
              },
            },
            models: [{ provider: "deepgram", model: "nova-3" }],
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## Sesli Arama akış STT'si

Paketle birlikte gelen `deepgram` Plugin'i, Sesli Arama Plugin'i için
gerçek zamanlı bir transkripsiyon sağlayıcısı da kaydeder.

| Ayar            | Yapılandırma yolu                                                      | Varsayılan                                   |
| --------------- | ----------------------------------------------------------------------- | -------------------------------------------- |
| API anahtarı    | `plugins.entries.voice-call.config.streaming.providers.deepgram.apiKey` | `DEEPGRAM_API_KEY` değerine geri döner       |
| Temel URL       | `...deepgram.baseUrl`                                                   | `DEEPGRAM_BASE_URL` veya Deepgram'ın genel API'si |
| Model           | `...deepgram.model`                                                     | `nova-3`                           |
| Dil             | `...deepgram.language`                                                  | (ayarlanmamış)                               |
| Kodlama         | `...deepgram.encoding`                                                  | `mulaw`                           |
| Örnekleme hızı  | `...deepgram.sampleRate`                                                | `8000`                           |
| Uç noktalama    | `...deepgram.endpointingMs`                                             | `800`                           |
| Ara sonuçlar    | `...deepgram.interimResults`                                            | `true`                           |

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          streaming: {
            enabled: true,
            provider: "deepgram",
            providers: {
              deepgram: {
                apiKey: "${DEEPGRAM_API_KEY}",
                model: "nova-3",
                endpointingMs: 800,
                language: "en-US",
              },
            },
          },
        },
      },
    },
  },
}
```

Bir [Deepgram özel uç noktası](https://developers.deepgram.com/reference/custom-endpoints) için
`baseUrl` değerini, temel yollar dâhil ancak `/listen` hariç olmak üzere uç nokta köküne ayarlayın.
Gerçek zamanlı uç noktalar `http://`, `https://`, `ws://` ve `wss://` kabul eder. HTTP
WS'ye, HTTPS WSS'ye eşlenir ve açık WebSocket şemaları değişmeden kalır.
Hatalı biçimlendirilmiş URL'ler ve diğer şemalar oturum kurulumu sırasında başarısız olur.

<Note>
Sesli Arama, telefon sesini 8 kHz G.711 u-law olarak alır. Deepgram
akış sağlayıcısının varsayılanları `encoding: "mulaw"` ve `sampleRate: 8000` olduğundan
Twilio medya kareleri doğrudan iletilebilir.
</Note>

## Notlar

<AccordionGroup>
  <Accordion title="Kimlik doğrulama">
    Kimlik doğrulama, standart sağlayıcı kimlik doğrulama sırasını izler. `DEEPGRAM_API_KEY`
    en basit yoldur.
  </Accordion>
  <Accordion title="Proxy ve özel uç noktalar">
    Proxy kullanırken Deepgram `tools.media.models[]` girdisindeki uç noktaları veya üstbilgileri geçersiz kılın.
  </Accordion>
  <Accordion title="Çıktı davranışı">
    Çıktı, diğer sağlayıcılarla aynı ses kurallarını izler (boyut sınırları, zaman aşımları,
    transkript ekleme).
  </Accordion>
</AccordionGroup>

## İlgili

<CardGroup cols={2}>
  <Card title="Medya araçları" href="/tr/tools/media-overview" icon="photo-film">
    Ses, görüntü ve video işleme hattına genel bakış.
  </Card>
  <Card title="Yapılandırma" href="/tr/gateway/configuration" icon="gear">
    Medya aracı ayarları dâhil tam yapılandırma başvurusu.
  </Card>
  <Card title="Sorun giderme" href="/tr/help/troubleshooting" icon="wrench">
    Yaygın sorunlar ve hata ayıklama adımları.
  </Card>
  <Card title="SSS" href="/tr/help/faq" icon="circle-question">
    OpenClaw kurulumu hakkında sıkça sorulan sorular.
  </Card>
</CardGroup>
