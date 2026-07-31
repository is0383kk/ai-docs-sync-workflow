---
read_when:
    - Volcano Engine veya Doubao modellerini OpenClaw ile kullanmak istiyorsunuz
    - Volcengine API anahtarı kurulumunu yapmanız gerekir
    - Volcengine Speech metinden konuşmaya özelliğini kullanmak istiyorsunuz
summary: Volcano Engine kurulumu (Doubao modelleri, kodlama uç noktaları ve Seed Speech TTS)
title: Volcengine (Doubao)
x-i18n:
    generated_at: "2026-07-26T23:00:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89538772b704499547ecf0274c5bb9bf8f68cc267dc7f484d3236921a9c89681
    source_path: providers/volcengine.md
    workflow: 16
---

Volcengine sağlayıcısı, genel ve kodlama iş yükleri için ayrı uç noktalarla Doubao modellerine ve Volcano Engine üzerinde barındırılan üçüncü taraf modellere erişim sağlar. Aynı paketlenmiş plugin, Volcengine Speech'ü bir TTS sağlayıcısı olarak da kaydeder.

| Ayrıntı          | Değer                                                      |
| ---------------- | ---------------------------------------------------------- |
| Sağlayıcılar     | `volcengine` (genel + TTS), `volcengine-plan` (kodlama)   |
| Model kimlik doğrulaması | `VOLCANO_ENGINE_API_KEY`                                   |
| TTS kimlik doğrulaması   | `VOLCENGINE_TTS_API_KEY` veya `BYTEPLUS_SEED_SPEECH_API_KEY` |
| API              | OpenAI uyumlu modeller, BytePlus Seed Speech TTS           |

## Başlarken

<Steps>
  <Step title="API anahtarını ayarlayın">
    Etkileşimli ilk kurulumu çalıştırın:

    ```bash
    openclaw onboard --auth-choice volcengine-api-key
    ```

    Bu işlem, tek bir API anahtarından hem genel (`volcengine`) hem de kodlama (`volcengine-plan`) sağlayıcılarını kaydeder.

  </Step>
  <Step title="Varsayılan bir model ayarlayın">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "volcengine-plan/ark-code-latest" },
        },
      },
    }
    ```
  </Step>
  <Step title="Modelin kullanılabilir olduğunu doğrulayın">
    ```bash
    openclaw models list --provider volcengine
    openclaw models list --provider volcengine-plan
    ```
  </Step>
</Steps>

<Tip>
Etkileşimsiz kurulum (CI, betik çalıştırma) için anahtarı doğrudan iletin:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice volcengine-api-key \
  --volcengine-api-key "$VOLCANO_ENGINE_API_KEY"
```

</Tip>

## Sağlayıcılar ve uç noktalar

| Sağlayıcı         | Uç nokta                                  | Kullanım alanı    |
| ----------------- | ----------------------------------------- | ----------------- |
| `volcengine`      | `ark.cn-beijing.volces.com/api/v3`        | Genel modeller    |
| `volcengine-plan` | `ark.cn-beijing.volces.com/api/coding/v3` | Kodlama modelleri |

<Note>
Her iki sağlayıcı da tek bir API anahtarından yapılandırılır. Kurulum her ikisini de otomatik olarak kaydeder ve kodlama sağlayıcısının model seçicisi de genel sağlayıcının kimlik doğrulamasını yeniden kullanır (`volcengine-plan`, `volcengine` için bir kimlik doğrulama diğer adıdır).
</Note>

## Yerleşik katalog

<Tabs>
  <Tab title="Genel (volcengine)">
    | Model başvurusu                              | Ad                              | Girdi       | Bağlam  |
    | -------------------------------------------- | ------------------------------- | ----------- | ------- |
    | `volcengine/deepseek-v3-2-251201`            | DeepSeek V3.2                   | metin, görsel | 128,000 |
    | `volcengine/doubao-seed-1-8-251228`          | Doubao Seed 1.8                 | metin, görsel | 256,000 |
    | `volcengine/doubao-seed-code-preview-251028` | doubao-seed-code-preview-251028 | metin, görsel | 256,000 |
    | `volcengine/glm-4-7-251222`                  | GLM 4.7                         | metin, görsel | 200,000 |
    | `volcengine/kimi-k2-5-260127`                | Kimi K2.5                       | metin, görsel | 256,000 |
  </Tab>
  <Tab title="Kodlama (volcengine-plan)">
    | Model başvurusu                                   | Ad                       | Girdi | Bağlam  |
    | ------------------------------------------------- | ------------------------ | ----- | ------- |
    | `volcengine-plan/ark-code-latest`                 | Ark Coding Plan          | metin | 256,000 |
    | `volcengine-plan/doubao-seed-code`                | Doubao Seed Code         | metin | 256,000 |
  </Tab>
</Tabs>

Her iki katalog da statiktir (`/models` keşif çağrısı yoktur) ve OpenAI uyumlu akışlı kullanım hesaplamasını destekler. Volcengine araç çağrısı API'si bunları reddettiğinden, her iki sağlayıcının araç şemaları `minLength`, `maxLength`, `minItems`, `maxItems`, `minContains` ve `maxContains` anahtar sözcüklerini otomatik olarak çıkarır.

## Metinden konuşmaya

Volcengine TTS, BytePlus Seed Speech HTTP API'sini (`voice.ap-southeast-1.bytepluses.com`) kullanır ve OpenAI uyumlu Doubao model API anahtarından ayrı olarak yapılandırılır. BytePlus konsolunda Seed Speech > Settings > API Keys yolunu açın, API anahtarını kopyalayın ve ardından şunları ayarlayın:

```bash
export VOLCENGINE_TTS_API_KEY="byteplus_seed_speech_api_key"
export VOLCENGINE_TTS_RESOURCE_ID="seed-tts-1.0"
```

Ardından `openclaw.json` içinde etkinleştirin:

```json5
{
  tts: {
    auto: "always",
    provider: "volcengine",
    providers: {
      volcengine: {
        apiKey: "byteplus_seed_speech_api_key",
        voice: "en_female_anna_mars_bigtts",
        speedRatio: 1.0,
      },
    },
  },
}
```

`tts.providers.volcengine` altındaki kullanılabilir alanlar: `apiKey`, `voice`, `speedRatio` (0.2-3.0), `emotion`, `cluster`, `resourceId`, `appKey` ve `baseUrl`. Ses ayarı geçersiz kılmalarına izin verildiğinde `!emotion=<value>`, satır içi ses yönergesi olarak da çalışır.

Sesli not hedefleri için OpenClaw, sağlayıcının yerel `ogg_opus` biçimini ister. Normal ses ekleri için `mp3` biçimini ister. `bytedance` ve `doubao` sağlayıcı diğer adları da bu konuşma sağlayıcısına çözümlenir.

Varsayılan kaynak kimliği, BytePlus'ın yeni oluşturulan Seed Speech API anahtarlarına varsayılan olarak verdiği yetki olan `seed-tts-1.0` değeridir. Projenizin TTS 2.0 yetkisi varsa `VOLCENGINE_TTS_RESOURCE_ID=seed-tts-2.0` değerini ayarlayın.

<Warning>
`VOLCANO_ENGINE_API_KEY`, ModelArk/Doubao model uç noktaları içindir ve bir Seed Speech API anahtarı değildir. TTS için BytePlus Speech Console'dan alınan bir Seed Speech API anahtarı veya eski bir Speech Console AppID/token çifti gerekir.
</Warning>

Eski Speech Console uygulamaları için eski AppID/token kimlik doğrulaması desteklenmeye devam eder:

```bash
export VOLCENGINE_TTS_APPID="speech_app_id"
export VOLCENGINE_TTS_TOKEN="speech_access_token"
export VOLCENGINE_TTS_CLUSTER="volcano_tts"
```

Diğer isteğe bağlı TTS ortam değişkenleri: `VOLCENGINE_TTS_VOICE`, `VOLCENGINE_TTS_APP_KEY` ve `VOLCENGINE_TTS_BASE_URL`; ayarlandıklarında karşılık gelen `tts.providers.volcengine` yapılandırma alanlarını geçersiz kılar.

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="İlk kurulumdan sonraki varsayılan model">
    `openclaw onboard --auth-choice volcengine-api-key`, genel `volcengine` kataloğunu da kaydederken `volcengine-plan/ark-code-latest` değerini varsayılan model olarak ayarlar.
  </Accordion>

  <Accordion title="Model seçicinin geri dönüş davranışı">
    İlk kurulum/yapılandırma sırasında model seçilirken Volcengine kimlik doğrulama seçeneği hem `volcengine/*` hem de `volcengine-plan/*` satırlarını tercih eder. Bu modeller henüz yüklenmemişse OpenClaw, sağlayıcı kapsamındaki boş bir seçici göstermek yerine filtrelenmemiş kataloğa geri döner.
  </Accordion>

  <Accordion title="Daemon süreçleri için ortam değişkenleri">
    Gateway bir daemon (launchd/systemd) olarak çalışıyorsa `VOLCANO_ENGINE_API_KEY`, `VOLCENGINE_TTS_API_KEY`, `BYTEPLUS_SEED_SPEECH_API_KEY`, `VOLCENGINE_TTS_APPID` ve `VOLCENGINE_TTS_TOKEN` gibi model ve TTS ortam değişkenlerinin bu süreç tarafından kullanılabildiğinden emin olun (örneğin `~/.openclaw/.env` içinde veya `env.shellEnv` aracılığıyla).
  </Accordion>
</AccordionGroup>

<Warning>
OpenClaw bir arka plan hizmeti olarak çalıştırıldığında, etkileşimli kabuğunuzda ayarlanan ortam değişkenleri otomatik olarak devralınmaz. Yukarıdaki daemon notuna bakın.
</Warning>

## İlgili

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model başvurularını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Yapılandırma" href="/tr/gateway/configuration" icon="gear">
    Ajanlar, modeller ve sağlayıcılar için tam yapılandırma başvurusu.
  </Card>
  <Card title="Sorun giderme" href="/tr/help/troubleshooting" icon="wrench">
    Yaygın sorunlar ve hata ayıklama adımları.
  </Card>
  <Card title="SSS" href="/tr/help/faq" icon="circle-question">
    OpenClaw kurulumu hakkında sıkça sorulan sorular.
  </Card>
</CardGroup>
