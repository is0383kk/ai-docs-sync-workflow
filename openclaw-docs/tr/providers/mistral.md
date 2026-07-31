---
read_when:
    - OpenClaw'da Mistral modellerini kullanmak istiyorsunuz
    - Sesli Arama için Voxtral gerçek zamanlı transkripsiyonunu kullanmak istiyorsunuz
    - Mistral API anahtarı ilk katılımına ve model referanslarına ihtiyacınız var
summary: OpenClaw ile Mistral modellerini ve Voxtral transkripsiyonunu kullanın
title: Mistral
x-i18n:
    generated_at: "2026-07-26T23:58:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 23f0ebb664a37cadefb65b7f531cecd3bdfaa4ff5426cb665e305f8f03f0b0ab
    source_path: providers/mistral.md
    workflow: 16
---

Paketle birlikte gelen `mistral` Plugin'i dört sözleşme kaydeder: sohbet tamamlamaları, medya anlama (Voxtral toplu transkripsiyon), Voice Call için gerçek zamanlı STT (Voxtral Realtime) ve bellek gömmeleri (`mistral-embed`).

| Özellik          | Değer                                       |
| ---------------- | ------------------------------------------- |
| Sağlayıcı kimliği | `mistral`                                   |
| Plugin           | paketle birlikte gelir, varsayılan olarak etkindir |
| Kimlik doğrulama ortam değişkeni | `MISTRAL_API_KEY`                           |
| İlk kurulum bayrağı | `--auth-choice mistral-api-key`             |
| Doğrudan CLI bayrağı | `--mistral-api-key <key>`                   |
| API              | OpenAI uyumlu (`openai-completions`)    |
| Temel URL        | `https://api.mistral.ai/v1`                 |
| Varsayılan model | `mistral/mistral-large-latest`              |
| Gömme modeli     | `mistral-embed`                             |
| Voxtral toplu    | `voxtral-mini-latest` (ses transkripsiyonu) |
| Voxtral gerçek zamanlı | `voxtral-mini-transcribe-realtime-2602`     |

## Başlarken

<Steps>
  <Step title="API anahtarınızı alın">
    [Mistral Console](https://console.mistral.ai/) içinde bir API anahtarı oluşturun.
  </Step>
  <Step title="İlk kurulumu çalıştırın">
    ```bash
    openclaw onboard --auth-choice mistral-api-key
    ```

    Alternatif olarak anahtarı doğrudan iletin:

    ```bash
    openclaw onboard --mistral-api-key "$MISTRAL_API_KEY"
    ```

  </Step>
  <Step title="Varsayılan bir model ayarlayın">
    ```json5
    {
      env: { MISTRAL_API_KEY: "sk-..." },
      agents: { defaults: { model: { primary: "mistral/mistral-large-latest" } } },
    }
    ```
  </Step>
  <Step title="Modelin kullanılabilir olduğunu doğrulayın">
    ```bash
    openclaw models list --provider mistral
    ```
  </Step>
</Steps>

## Yerleşik LLM kataloğu

| Model referansı                   | Girdi       | Bağlam  | Maksimum çıktı | Notlar                                                |
| -------------------------------- | ----------- | ------- | -------------- | ----------------------------------------------------- |
| `mistral/mistral-large-latest`   | metin, görüntü | 262,144 | 16,384     | Varsayılan model                                      |
| `mistral/mistral-medium-2508`    | metin, görüntü | 262,144 | 8,192      | Mistral Medium 3.1                                    |
| `mistral/mistral-medium-3-5`     | metin, görüntü | 262,144 | 8,192      | Mistral Medium 3.5; ayarlanabilir akıl yürütme        |
| `mistral/mistral-small-latest`   | metin, görüntü | 262,144 | 16,384     | En son Mistral Small 4; ayarlanabilir `reasoning_effort` |
| `mistral/mistral-small-2603`     | metin, görüntü | 262,144 | 16,384     | Sabitlenmiş Mistral Small 4; ayarlanabilir `reasoning_effort` |
| `mistral/pixtral-large-latest`   | metin, görüntü | 128,000 | 32,768     | Pixtral                                               |
| `mistral/codestral-latest`       | metin        | 256,000 | 4,096      | Kodlama                                               |
| `mistral/devstral-medium-latest` | metin        | 262,144 | 32,768     | Devstral 2                                            |
| `mistral/magistral-small`        | metin        | 128,000 | 40,000     | Akıl yürütme etkin                                    |

Yapılandırmayı değiştirmeden önce paketle birlikte gelen katalog satırına göz atın:

```bash
openclaw models list --all --provider mistral --plain
```

Gateway'i başlatmadan bir model için temel işlev testi gerçekleştirin:

```bash
openclaw infer model run --local \
  --model mistral/mistral-medium-3-5 \
  --prompt "Tam olarak şu yanıtı ver: mistral-ok" \
  --json
```

## Ses transkripsiyonu (Voxtral)

Medya anlama işlem hattı üzerinden toplu ses transkripsiyonu için Voxtral'ı kullanın:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "mistral", model: "voxtral-mini-latest" }],
      },
    },
  },
}
```

<Tip>
Medya transkripsiyonu yolu `/v1/audio/transcriptions` kullanır. Mistral için varsayılan ses modeli `voxtral-mini-latest` şeklindedir.
</Tip>

## Voice Call akış STT'si

Paketle birlikte gelen `mistral` Plugin'i, Voxtral Realtime'ı bir Voice Call akış STT sağlayıcısı olarak kaydeder.

| Ayar         | Yapılandırma yolu                                                     | Varsayılan                              |
| ------------ | ---------------------------------------------------------------------- | --------------------------------------- |
| API anahtarı | `plugins.entries.voice-call.config.streaming.providers.mistral.apiKey` | `MISTRAL_API_KEY` değerine geri döner   |
| Model        | `...mistral.model`                                                     | `voxtral-mini-transcribe-realtime-2602` |
| Kodlama      | `...mistral.encoding`                                                  | `pcm_mulaw`                             |
| Örnekleme hızı | `...mistral.sampleRate`                                                | `8000`                                  |
| Hedef gecikme | `...mistral.targetStreamingDelayMs`                                    | `800`                                   |

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          streaming: {
            enabled: true,
            provider: "mistral",
            providers: {
              mistral: {
                apiKey: "${MISTRAL_API_KEY}",
                targetStreamingDelayMs: 800,
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
OpenClaw, Voice Call'un Twilio medya karelerini doğrudan iletebilmesi için Mistral gerçek zamanlı STT'yi varsayılan olarak 8 kHz'de `pcm_mulaw` değerine ayarlar. Yalnızca yukarı akışınız zaten ham PCM ise `encoding: "pcm_s16le"` ve eşleşen bir `sampleRate` kullanın.
</Note>

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Ayarlanabilir akıl yürütme">
    `mistral/mistral-small-latest`, `mistral/mistral-small-2603` ve `mistral/mistral-medium-3-5`, Chat Completions API üzerinde `reasoning_effort` aracılığıyla [ayarlanabilir akıl yürütmeyi](https://docs.mistral.ai/studio-api/conversations/reasoning/adjustable) destekler (`none`, çıktıdaki ek düşünmeyi en aza indirir; `high`, nihai yanıttan önce tüm düşünme izlerini gösterir).

    OpenClaw, oturumun **düşünme** düzeyini Mistral API'sine şu şekilde eşler:

    | OpenClaw düşünme düzeyi                                              | Mistral `reasoning_effort` |
    | ----------------------------------------------------------------------- | --------------------------- |
    | **kapalı** / **en az**                                                | `none`                      |
    | **düşük** / **orta** / **yüksek** / **çok yüksek** / **uyarlanabilir** / **maksimum** | `high`                       |

    <Warning>
    Medium 3.5 akıl yürütme modunu `temperature: 0` ile birleştirmekten kaçının; Mistral HTTP API'nin `reasoning_effort="high"` ile `temperature: 0` birleşimini 400 yanıtıyla reddettiği bildirilmiştir. Sıcaklığı ayarlamadan bırakın veya OpenClaw'ın düşük bir sıcaklık ayarlamadan önce `reasoning_effort: "none"` göndermesi için düşünmeyi kapatın/en aza indirin.
    </Warning>

    Medium 3.5 akıl yürütmesi için model kapsamlı örnek yapılandırma:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "mistral/mistral-medium-3-5" },
          models: {
            "mistral/mistral-medium-3-5": {
              params: { thinking: "high" },
            },
          },
        },
      },
    }
    ```

    <Note>
    Paketle birlikte gelen diğer Mistral katalog modelleri bu parametreyi kullanmaz. Mistral'ın yerel, akıl yürütme öncelikli davranışını istediğinizde `magistral-*` modellerini kullanmaya devam edin.
    </Note>

  </Accordion>

  <Accordion title="Bellek gömmeleri">
    Mistral, `/v1/embeddings` aracılığıyla bellek gömmeleri sunabilir (varsayılan model: `mistral-embed`):

    ```json5
    {
      memory: {
        search: { provider: "mistral" },
      },
    }
    ```

  </Accordion>

  <Accordion title="Kimlik doğrulama ve temel URL">
    - Mistral kimlik doğrulaması `MISTRAL_API_KEY` kullanır (Bearer üstbilgisi).
    - Sağlayıcının temel URL'si varsayılan olarak `https://api.mistral.ai/v1` değerini kullanır ve standart OpenAI uyumlu sohbet tamamlamaları istek biçimini kabul eder.
    - İlk kurulumun varsayılan modeli `mistral/mistral-large-latest` şeklindedir.
    - Temel URL'yi `models.providers.mistral.baseUrl` altında yalnızca Mistral ihtiyaç duyduğunuz bölgesel bir uç noktayı açıkça yayımladığında geçersiz kılın.

  </Accordion>
</AccordionGroup>

## İlgili

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Medya anlama" href="/tr/nodes/media-understanding" icon="microphone">
    Ses transkripsiyonu kurulumu ve sağlayıcı seçimi.
  </Card>
</CardGroup>
