---
read_when:
    - OpenClaw'da Z.AI / GLM modellerini kullanmak istiyorsunuz
    - Basit bir ZAI_API_KEY yapılandırmasına ihtiyacınız var
summary: OpenClaw ile Z.AI (GLM modelleri) kullanın
title: Z.AI
x-i18n:
    generated_at: "2026-07-27T00:15:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ca3e7ef743e908550f4d96ba6f78167e38cabd15b14044683b02493ebbf3025
    source_path: providers/zai.md
    workflow: 16
---

Z.AI, **GLM** modellerinin API platformudur. GLM için REST API'leri sağlar ve
kimlik doğrulama için API anahtarlarını kullanır. API anahtarınızı Z.AI konsolunda oluşturun.
OpenClaw, bir Z.AI API anahtarıyla `zai` sağlayıcısını kullanır.

| Özellik  | Değer                                        |
| -------- | -------------------------------------------- |
| Sağlayıcı | `zai`                                        |
| Paket    | `@openclaw/zai-provider`                     |
| Kimlik doğrulama | `ZAI_API_KEY` (eski diğer ad: `Z_AI_API_KEY`) |
| API      | Z.AI Chat Completions (Bearer kimlik doğrulaması)          |

## GLM modelleri

GLM ayrı bir sağlayıcı değil, bir model ailesidir. OpenClaw'da GLM modelleri
`zai/glm-5.2` gibi referanslar kullanır: sağlayıcı `zai`, model kimliği `glm-5.2`.

## Başlarken

Önce sağlayıcı pluginini yükleyin:

```bash
openclaw plugins install @openclaw/zai-provider
```

<Tabs>
  <Tab title="Uç noktayı otomatik algıla">
    **En uygun olduğu durum:** çoğu kullanıcı. OpenClaw, API anahtarınızla desteklenen Z.AI uç noktalarını yoklar ve doğru temel URL'yi otomatik olarak uygular.

    <Steps>
      <Step title="İlk kurulumu çalıştırın">
        ```bash
        openclaw onboard --auth-choice zai-api-key
        ```
      </Step>
      <Step title="Modelin listelendiğini doğrulayın">
        ```bash
        openclaw models list --all --provider zai
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Açıkça belirtilen bölgesel uç nokta">
    **En uygun olduğu durum:** belirli bir Coding Plan veya genel API yüzeyini kullanmaya zorlamak isteyen kullanıcılar.

    <Steps>
      <Step title="Doğru ilk kurulum seçeneğini belirleyin">
        ```bash
        # Coding Plan Global (Coding Plan kullanıcıları için önerilir)
        openclaw onboard --auth-choice zai-coding-global

        # Coding Plan CN (Çin bölgesi)
        openclaw onboard --auth-choice zai-coding-cn

        # Genel API
        openclaw onboard --auth-choice zai-global

        # Genel API CN (Çin bölgesi)
        openclaw onboard --auth-choice zai-cn
        ```
      </Step>
      <Step title="Modelin listelendiğini doğrulayın">
        ```bash
        openclaw models list --all --provider zai
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

### Uç noktalar

| İlk kurulum seçeneği | Temel URL                                     | Varsayılan model |
| -------------------- | --------------------------------------------- | ---------------- |
| `zai-global`        | `https://api.z.ai/api/paas/v4`                | `glm-5.1`     |
| `zai-cn`            | `https://open.bigmodel.cn/api/paas/v4`        | `glm-5.1`     |
| `zai-coding-global` | `https://api.z.ai/api/coding/paas/v4`         | `glm-5.2`     |
| `zai-coding-cn`     | `https://open.bigmodel.cn/api/coding/paas/v4` | `glm-5.2`     |

Z.AI ayrıca Anthropic uyumlu Coding Plan temel URL'si
`https://api.z.ai/api/anthropic` adresini yayımlar. OpenClaw'ın Z.AI seçenekleri yukarıda belgelenen
OpenAI Chat Completions uç noktalarını kullanır; Anthropic URL'si doğrudan
Anthropic Messages protokolünü kullanan istemciler içindir.

`zai-api-key`, anahtarınızı her uç noktanın chat-completions API'sinde yoklayarak
bu dört seçenekten birini otomatik algılar; önce genel uç noktaları (`zai-global`,
ardından `zai-cn`), sonra Coding Plan uç noktalarını (`zai-coding-global`, ardından
`zai-coding-cn`) denetler ve isteği kabul eden ilk uç noktada durur.
Anahtarınız her ikisinde de çalışıyorsa bir Coding Plan uç noktasını zorunlu kılmak için açıkça bir `--auth-choice` kullanın.

## Hız sınırları ve aşırı yüklenmeler

Z.AI, Coding Plan ve genel amaçlı aracı araçlarını kapasitesi
yönetilen hizmetler olarak belgeler. Z.AI'ın kendi belgelerine göre:

- [Genel amaçlı aracı araçları](https://docs.z.ai/devpack/tool/others),
  OpenClaw dahil olmak üzere, mümkün olan en iyi çaba esasına göre sunulur. Genellikle Singapur saatiyle
  14.00-18.00 civarındaki yüksek çıkarım yükü sırasında bazı istekler geçici
  hız sınırlarıyla karşılaşabilir.
- [Coding Plan hız ve eşzamanlılık sınırları](https://docs.z.ai/devpack/usage-policy)
  plan katmanına bağlıdır ve kaynak kullanılabilirliğine göre dinamik olarak
  ayarlanabilir. Yoğun olmayan saatlerde eşzamanlılık daha yüksek olabilir.
- [API hata kodu `1302`](https://docs.z.ai/api-reference/api-code), "İstekler için
  hız sınırına ulaşıldı" anlamına gelir. API hata kodu `1305`, "Hizmet geçici olarak
  aşırı yüklenmiş olabilir, lütfen daha sonra tekrar deneyin" anlamına gelir.

Yoğun bir dönemde geçici bir `429` veya `1305` yanıtı görürseniz bekleyip
isteği yeniden deneyin. Hatalar yoğun saatler dışında tekrarlanabiliyorsa veya yalnızca
tek bir uç nokta, model ya da istek biçiminde ortaya çıkıyorsa önce yapılandırılmış uç noktayı
ve modeli denetleyin:

```bash
openclaw models list --all --provider zai
openclaw config get models.providers.zai.baseUrl
```

Coding Plan anahtarları
`https://api.z.ai/api/coding/paas/v4` gibi bir Coding Plan uç noktası kullanmalıdır; genel API anahtarları
`https://api.z.ai/api/paas/v4` gibi bir genel API uç noktası kullanmalıdır. Aynı anahtar ve uç noktada
sürekli görülen hatalar, olağan yoğun yük kısıtlamasını değil, sağlayıcı tarafında reddedilmeyi
veya plan sınırlamasını gösterebilir.

## Yapılandırma örneği

<Tip>
`zai-api-key`, OpenClaw'ın anahtarla eşleşen Z.AI uç noktasını algılamasını ve
doğru temel URL'yi otomatik olarak uygulamasını sağlar. Belirli bir Coding Plan veya
genel API yüzeyini kullanmaya zorlamak istediğinizde açıkça belirtilen bölgesel seçenekleri kullanın.
</Tip>

```json5
{
  env: { ZAI_API_KEY: "sk-..." },
  models: {
    providers: {
      zai: {
        // GLM-5.2, Coding Plan uç noktasını kullanır.
        baseUrl: "https://api.z.ai/api/coding/paas/v4",
      },
    },
  },
  agents: { defaults: { model: { primary: "zai/glm-5.2" } } },
}
```

## Yerleşik katalog

`zai` sağlayıcı plugini, kataloğunu plugin bildiriminde sunar; bu nedenle salt okunur
listeleme, sağlayıcı çalışma zamanını yüklemeden bilinen GLM satırlarını gösterebilir:

```bash
openclaw models list --all --provider zai
```

Bildirim destekli katalog şu anda şunları içerir:

| Model referansı      | Notlar                          |
| -------------------- | ------------------------------- |
| `zai/glm-5.2`        | Coding Plan varsayılanı; 1M bağlam |
| `zai/glm-5.1`        | Genel API varsayılanı           |
| `zai/glm-5`          |                                 |
| `zai/glm-5-turbo`    |                                 |
| `zai/glm-5v-turbo`   |                                 |
| `zai/glm-4.7`        |                                 |
| `zai/glm-4.7-flash`  |                                 |
| `zai/glm-4.7-flashx` |                                 |
| `zai/glm-4.6`        |                                 |
| `zai/glm-4.6v`       |                                 |
| `zai/glm-4.5`        |                                 |
| `zai/glm-4.5-air`    |                                 |
| `zai/glm-4.5-flash`  |                                 |
| `zai/glm-4.5v`       |                                 |

Kataloğun token maliyeti meta verileri, Z.AI'ın güncel
[kullandıkça öde fiyatlandırmasını](https://docs.z.ai/guides/overview/pricing) izler. Coding Plan
abonelikleri token başına faturalandırma yerine plan kotasını kullanır; plan fiyatlandırması ve kullanılabilirliği için güncel
[abonelik sayfasına](https://z.ai/subscribe) bakın.

<Tip>
GLM modelleri `zai/<model>` olarak kullanılabilir (örnek: `zai/glm-5`).
</Tip>

<Note>
Coding Plan kurulumu varsayılan olarak `zai/glm-5.2` kullanır; genel API kurulumu
`zai/glm-5.1` değerini korur. Coding Plan uç noktalarında otomatik algılama, anahtar/plan GLM-5.2'yi sunmadığında
`glm-5.1` ve ardından `glm-4.7` modeline geri döner. GLM
sürümleri ve kullanılabilirliği değişebilir; yüklü sürümünüzün bildiği kataloğu görmek için
`openclaw models list --all --provider zai` komutunu çalıştırın.
</Note>

## Düşünme düzeyleri

<Tabs>
  <Tab title="GLM-5.2">
    Tam aralık: `off`, `low`, `high`, `max` (varsayılan `off`). OpenClaw,
    istek yükündeki `reasoning_effort` aracılığıyla `low` ve `high` değerlerini Z.AI'ın `high` akıl yürütme çabasına,
    `max` değerini ise Z.AI'ın `max` çabasına eşler.
  </Tab>
  <Tab title="Diğer GLM modelleri">
    Yalnızca ikili geçiş: `off` ve `low` (seçicilerde `on` olarak gösterilir), varsayılan
    `off`. Düşünmeyi `off` olarak ayarlamak `thinking: { type: "disabled" }` gönderir;
    diğer tüm düzeylerde istek yükü değiştirilmez (Z.AI'ın kendi varsayılan
    akıl yürütme davranışı uygulanır).
  </Tab>
</Tabs>

Düşünmeyi `off` olarak ayarlamak, görünür metinden önce çıktı bütçesini
`reasoning_content` için harcayan yanıtlardan kaçınır.

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Bilinmeyen GLM-5 modellerini ileriye dönük çözümleme">
    Bilinmeyen `glm-5*` kimlikleri, kimlik güncel GLM-5 ailesi biçimiyle eşleştiğinde
    `glm-4.7` şablonundan sağlayıcıya ait meta veriler sentezlenerek sağlayıcı yolunda
    ileriye dönük çözümlenmeye devam eder.
  </Accordion>

  <Accordion title="Araç çağrısı akışı">
    Z.AI araç çağrısı akışı için `tool_stream` varsayılan olarak etkindir. Devre dışı bırakmak için:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "zai/<model>": {
              params: { tool_stream: false },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Korunan düşünme">
    Z.AI, istem tokenlarını artıran tam geçmiş
    `reasoning_content` verisinin yeniden oynatılmasını gerektirdiğinden, korunan düşünme isteğe bağlıdır. Model başına etkinleştirin:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "zai/glm-5.2": {
              params: { preserveThinking: true },
            },
          },
        },
      },
    }
    ```

    Etkinleştirildiğinde ve düşünme açıkken OpenClaw,
    `thinking: { type: "enabled", clear_thinking: false }` gönderir ve aynı OpenAI uyumlu transkript için önceki
    `reasoning_content` verilerini yeniden oynatır. Alt çizgili
    `preserve_thinking` parametre anahtarı diğer ad olarak çalışır.

    İleri düzey kullanıcılar, sağlayıcının tam istek yükünü
    `params.extra_body.thinking` ile geçersiz kılmaya devam edebilir.

  </Accordion>

  <Accordion title="Görüntü anlama">
    Z.AI plugini görüntü anlamayı kaydeder.

    | Özellik       | Değer       |
    | ------------- | ----------- |
    | Model         | `glm-4.6v`  |

    Görüntü anlama, yapılandırılmış Z.AI kimlik doğrulamasından otomatik olarak çözümlenir;
    ek yapılandırma gerekmez.

  </Accordion>

  <Accordion title="Kimlik doğrulama ayrıntıları">
    - Z.AI, API anahtarınızla Bearer kimlik doğrulaması kullanır.
    - `zai-api-key` ilk kurulum seçeneği, anahtarınızla desteklenen uç noktaları yoklayarak eşleşen Z.AI uç noktasını otomatik olarak algılar.
    - Belirli bir API yüzeyini kullanmaya zorlamak istediğinizde açıkça belirtilen bölgesel seçenekleri (`zai-coding-global`, `zai-coding-cn`, `zai-global`, `zai-cn`) kullanın.
    - Eski `Z_AI_API_KEY` ortam değişkeni hâlâ kabul edilir; `ZAI_API_KEY` ayarlanmamışsa OpenClaw, başlangıçta bunu `ZAI_API_KEY` değişkenine kopyalar.

  </Accordion>
</AccordionGroup>

## İlgili kaynaklar

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Yapılandırma referansı" href="/tr/gateway/configuration-reference" icon="gear">
    Sağlayıcı ve model ayarları dahil tam OpenClaw yapılandırma şeması.
  </Card>
</CardGroup>
