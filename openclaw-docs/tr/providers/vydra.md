---
read_when:
    - OpenClaw'da Vydra ile medya oluşturmak istiyorsunuz
    - Vydra API anahtarı kurulum rehberine ihtiyacınız var
summary: OpenClaw'da Vydra görüntü, video ve konuşma özelliklerini kullanın
title: Vydra
x-i18n:
    generated_at: "2026-07-26T23:38:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cc3856c2dd740e87d70d7eedefd9eae7905ab547aa0d68a1c479a305c59b2982
    source_path: providers/vydra.md
    workflow: 16
---

Paketle gelen Vydra Plugin'i şunları ekler:

- `vydra/grok-imagine` aracılığıyla görsel oluşturma
- `vydra/veo3` (metinden videoya) ve `vydra/kling` (görselden videoya) aracılığıyla video oluşturma
- Vydra'nın ElevenLabs destekli TTS rotası aracılığıyla konuşma sentezi

OpenClaw, üç özelliğin tümü için aynı `VYDRA_API_KEY` değerini kullanır.

| Özellik         | Değer                                                                     |
| --------------- | ------------------------------------------------------------------------- |
| Sağlayıcı kimliği | `vydra`                                                                   |
| Plugin          | paketle gelen, `enabledByDefault: true`                                         |
| Kimlik doğrulama ortam değişkeni | `VYDRA_API_KEY`                                                           |
| İlk kurulum bayrağı | `--auth-choice vydra-api-key`                                             |
| Doğrudan CLI bayrağı | `--vydra-api-key <key>`                                                   |
| Sözleşmeler     | `imageGenerationProviders`, `videoGenerationProviders`, `speechProviders` |
| Temel URL       | `https://www.vydra.ai/api/v1` (`www` ana makinesini kullanın)                        |

<Warning>
Temel URL olarak `https://www.vydra.ai/api/v1` kullanın. Vydra'nın kök ana makinesi (`https://vydra.ai/api/v1`) şu anda `www` adresine yönlendiriyor. Bazı HTTP istemcileri bu farklı ana makineye yönlendirme sırasında `Authorization` değerini çıkarır; bu da geçerli bir API anahtarının yanıltıcı bir kimlik doğrulama hatasına dönüşmesine neden olur. Paketle gelen Plugin, bunu önlemek için yapılandırılmış tüm `vydra.ai` temel URL'lerini `www.vydra.ai` olarak normalleştirir.
</Warning>

## Kurulum

<Steps>
  <Step title="Etkileşimli ilk kurulumu çalıştırın">
    ```bash
    openclaw onboard --auth-choice vydra-api-key
    ```

    Alternatif olarak ortam değişkenini doğrudan ayarlayın:

    ```bash
    export VYDRA_API_KEY="vydra_live_..."
    ```

  </Step>
  <Step title="Varsayılan bir özellik seçin">
    Aşağıdaki özelliklerden birini veya birkaçını (görsel, video ya da konuşma) seçin ve eşleşen yapılandırmayı uygulayın.
  </Step>
</Steps>

## Özellikler

<AccordionGroup>
  <Accordion title="Görsel oluşturma">
    Varsayılan ve paketle gelen tek görsel modeli:

    - `vydra/grok-imagine`

    Bunu varsayılan görsel sağlayıcısı olarak ayarlayın:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "vydra/grok-imagine",
          },
        },
      },
    }
    ```

    Paketle gelen destek yalnızca metinden görsele dönüştürme içindir ve istek başına en fazla bir görsel oluşturur. Vydra'nın barındırılan düzenleme rotaları uzak görsel URL'leri bekler ve paketle gelen Plugin, Vydra'ya özgü bir yükleme köprüsü eklemez.

    <Note>
    Paylaşılan araç parametreleri, sağlayıcı seçimi ve yük devretme davranışı için [Görsel Oluşturma](/tr/tools/image-generation) bölümüne bakın.
    </Note>

  </Accordion>

  <Accordion title="Video oluşturma">
    Kayıtlı video modelleri:

    - Metinden videoya dönüştürme için `vydra/veo3` (görsel referansı girdilerini reddeder)
    - Görselden videoya dönüştürme için `vydra/kling` (tam olarak bir uzak görsel URL'si gerektirir)

    Vydra'yı varsayılan video sağlayıcısı olarak ayarlayın:

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "vydra/veo3",
          },
        },
      },
    }
    ```

    Notlar:

    - `vydra/kling` yerel dosya yüklemelerini baştan reddeder; yalnızca uzak bir görsel URL'si referansı çalışır.
    - Vydra'nın `kling` HTTP rotası, `image_url` veya `video_url` gerektirip gerektirmediği konusunda tutarsız davranmıştır; paketle gelen sağlayıcı, her iki alanda da aynı uzak görsel URL'sini gönderir.
    - Paketle gelen Plugin ihtiyatlı davranır ve en-boy oranı, çözünürlük, filigran veya oluşturulan ses gibi belgelenmemiş stil ayarlarını iletmez.

    <Note>
    Paylaşılan araç parametreleri, sağlayıcı seçimi ve yük devretme davranışı için [Video Oluşturma](/tr/tools/video-generation) bölümüne bakın.
    </Note>

  </Accordion>

  <Accordion title="Canlı video testleri">
    Sağlayıcıya özgü canlı test kapsamı:

    ```bash
    OPENCLAW_LIVE_TEST=1 \
    OPENCLAW_LIVE_VYDRA_VIDEO=1 \
    pnpm test:live -- extensions/vydra/vydra.live.test.ts
    ```

    Paketle gelen Vydra canlı test dosyası şunları kapsar:

    - `vydra/veo3` ile metinden videoya dönüştürme
    - Uzak bir görsel URL'si kullanarak `vydra/kling` ile görselden videoya dönüştürme

    Gerektiğinde uzak görsel test verisini geçersiz kılın:

    ```bash
    export OPENCLAW_LIVE_VYDRA_KLING_IMAGE_URL="https://example.com/reference.png"
    ```

  </Accordion>

  <Accordion title="Konuşma sentezi">
    Vydra'yı konuşma sağlayıcısı olarak ayarlayın:

    ```json5
    {
      tts: {
        provider: "vydra",
        providers: {
          vydra: {
            apiKey: "${VYDRA_API_KEY}",
            voiceId: "21m00Tcm4TlvDq8ikWAM",
          },
        },
      },
    }
    ```

    Varsayılanlar:

    - Model: `elevenlabs/tts`
    - Ses kimliği: `21m00Tcm4TlvDq8ikWAM` ("Rachel")

    Paketle gelen Plugin, çalıştığı bilinen bu tek varsayılan sesi sunar ve MP3 ses dosyaları döndürür.

  </Accordion>
</AccordionGroup>

## İlgili

<CardGroup cols={2}>
  <Card title="Sağlayıcı dizini" href="/tr/providers/index" icon="list">
    Kullanılabilir tüm sağlayıcılara göz atın.
  </Card>
  <Card title="Görsel oluşturma" href="/tr/tools/image-generation" icon="image">
    Paylaşılan görsel aracı parametreleri ve sağlayıcı seçimi.
  </Card>
  <Card title="Video oluşturma" href="/tr/tools/video-generation" icon="video">
    Paylaşılan video aracı parametreleri ve sağlayıcı seçimi.
  </Card>
  <Card title="Yapılandırma referansı" href="/tr/gateway/config-agents#agent-defaults" icon="gear">
    Aracı varsayılanları ve model yapılandırması.
  </Card>
</CardGroup>
