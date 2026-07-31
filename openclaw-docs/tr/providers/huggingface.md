---
read_when:
    - OpenClaw ile Hugging Face Inference kullanmak istiyorsunuz
    - HF token ortam değişkenini veya CLI kimlik doğrulama seçeneğini kullanmanız gerekir
summary: Hugging Face Inference kurulumu (kimlik doğrulama + model seçimi)
title: Hugging Face (çıkarım)
x-i18n:
    generated_at: "2026-07-26T22:59:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 92c400b78c5ad2cc724ad4029560dccc5bc2006fdeae400fc6b58998e727e17c
    source_path: providers/huggingface.md
    workflow: 16
---

[Hugging Face Inference Providers](https://huggingface.co/docs/inference-providers), tek bir token altında barındırılan birçok modelin (DeepSeek, Llama ve daha fazlası) önünde OpenAI uyumlu bir sohbet tamamlama yönlendiricisi sunar. OpenClaw yalnızca **sohbet tamamlama uç noktasıyla** iletişim kurar; metinden görüntü oluşturma, gömme veya konuşma için doğrudan [HF çıkarım istemcilerini](https://huggingface.co/docs/api-inference/quicktour) kullanın.

| Özellik          | Değer                                                                                                                               |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Sağlayıcı kimliği | `huggingface`                                                                                                                  |
| Plugin           | paketle birlikte gelir (varsayılan olarak etkindir, kurulum adımı yoktur)                                                           |
| Kimlik doğrulama ortam değişkeni | `HUGGINGFACE_HUB_TOKEN` veya `HF_TOKEN` (ayrıntılı izinlere sahip token)                                            |
| API              | OpenAI uyumlu (`https://router.huggingface.co/v1`)                                                                                                  |
| Faturalandırma   | Tek HF token'ı; [fiyatlandırma](https://huggingface.co/docs/inference-providers/pricing), ücretsiz katmanla birlikte sağlayıcı ücretlerini izler |

## Başlarken

<Steps>
  <Step title="Ayrıntılı izinlere sahip bir token oluşturun">
    [Hugging Face Settings Tokens](https://huggingface.co/settings/tokens/new?ownUserPermissions=inference.serverless.write&tokenType=fineGrained) sayfasına gidin ve ayrıntılı izinlere sahip yeni bir token oluşturun.

    <Warning>
    Token için **Make calls to Inference Providers** izni etkinleştirilmiş olmalıdır; aksi takdirde API istekleri reddedilir.
    </Warning>

  </Step>
  <Step title="İlk kurulumu çalıştırın">
    Sağlayıcı açılır listesinden **Hugging Face** seçeneğini belirleyin, ardından istendiğinde API anahtarınızı girin:

    ```bash
    openclaw onboard --auth-choice huggingface-api-key
    ```

  </Step>
  <Step title="Varsayılan bir model seçin">
    **Default Hugging Face model** açılır listesinden bir model seçin. Token'ınız geçerliyse liste Inference API'den yüklenir; aksi takdirde OpenClaw aşağıdaki yerleşik kataloğu gösterir. Seçiminiz `agents.defaults.model.primary` olarak kaydedilir:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
        },
      },
    }
    ```

  </Step>
  <Step title="Modelin kullanılabilir olduğunu doğrulayın">
    ```bash
    openclaw models list --provider huggingface
    ```
  </Step>
</Steps>

### Etkileşimsiz kurulum

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice huggingface-api-key \
  --huggingface-api-key "$HF_TOKEN"
```

`huggingface/deepseek-ai/DeepSeek-R1` değerini varsayılan model olarak ayarlar.

## Model kimlikleri

Model referansları `huggingface/<org>/<model>` biçimini kullanır (Hub tarzı kimlikler). OpenClaw'ın yerleşik kataloğu:

| Model         | Referans (başına `huggingface/` ekleyin) |
| ------------- | -------------------------------------------- |
| DeepSeek R1   | `deepseek-ai/DeepSeek-R1`                           |
| DeepSeek V3.1 | `deepseek-ai/DeepSeek-V3.1`                           |
| GPT-OSS 120B  | `openai/gpt-oss-120b`                           |

<Tip>
Token'ınız geçerliyse OpenClaw, ilk kurulum sırasında ve Gateway başlatılırken **GET** `https://router.huggingface.co/v1/models` üzerinden diğer tüm modelleri de keşfeder; dolayısıyla kataloğunuz yukarıdaki üç modelden çok daha fazlasını içerebilir. Herhangi bir model kimliğine `:fastest` veya `:cheapest` ekleyebilirsiniz; HF yönlendiricisi isteği eşleşen çıkarım sağlayıcısına yönlendirir. Varsayılan sağlayıcı sıranızı [Inference Provider settings](https://hf.co/settings/inference-providers) bölümünde ayarlayın.
</Tip>

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Model keşfi ve ilk kurulum açılır listesi">
    OpenClaw modelleri şu istekle keşfeder:

    ```bash
    GET https://router.huggingface.co/v1/models
    Authorization: Bearer $HUGGINGFACE_HUB_TOKEN   # veya $HF_TOKEN
    ```

    Yanıt OpenAI tarzındadır: `{ "object": "list", "data": [ { "id": "Qwen/Qwen3-8B", "owned_by": "Qwen", ... }, ... ] }`.

    Yapılandırılmış bir anahtar varsa (ilk kurulum, `HUGGINGFACE_HUB_TOKEN` veya `HF_TOKEN`), etkileşimli kurulum sırasında **Default Hugging Face model** açılır listesi bu uç noktadan doldurulur. Gateway başlatılırken kataloğu yenilemek için aynı çağrı tekrarlanır. Keşfedilen modeller, yukarıdaki yerleşik katalogla birleştirilir (bir kimlik eşleştiğinde bağlam penceresi ve maliyet gibi meta veriler için kullanılır). İstek başarısız olursa, veri döndürmezse veya herhangi bir anahtar ayarlanmamışsa OpenClaw yalnızca yerleşik kataloğu kullanır.

    Sağlayıcıyı kaldırmadan keşfi devre dışı bırakın:

    ```bash
    openclaw config set plugins.entries.huggingface.config.discovery.enabled false
    ```

  </Accordion>

  <Accordion title="Model adları, diğer adlar ve ilke son ekleri">
    - **API'den gelen ad:** Keşfedilen modeller, mevcut olduğunda API'nin `name`, `title` veya `display_name` değerini kullanır; aksi takdirde OpenClaw model kimliğinden bir ad türetir (ör. `deepseek-ai/DeepSeek-R1`, "DeepSeek R1" olur).
    - **Görünen adı geçersiz kılma:** Yapılandırmada her model için özel bir etiket ayarlayın:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1 (fast)" },
            "huggingface/deepseek-ai/DeepSeek-R1:cheapest": { alias: "DeepSeek R1 (cheap)" },
          },
        },
      },
    }
    ```

    - **İlke son ekleri:** `:fastest` ve `:cheapest`, OpenClaw'ın yeniden yazdığı değerler değil, HF yönlendirici kurallarıdır: son ek, model kimliğinin bir parçası olarak aynen gönderilir ve HF yönlendiricisi eşleşen çıkarım sağlayıcısını seçer. Her son ek için farklı bir diğer ad istiyorsanız her çeşidi `models.providers.huggingface.models` altında (veya `model.primary` içinde) ayrı bir girdi olarak ekleyin.
    - **Yapılandırma birleştirme:** `models.providers.huggingface.models` içindeki mevcut girdiler (ör. `models.json` içinde) yapılandırma birleştirilirken korunur; dolayısıyla burada ayarladığınız özel `name`, `alias` veya model seçenekleri yeniden başlatmalarda kalıcı olur.

  </Accordion>

  <Accordion title="Ortam ve arka plan hizmeti kurulumu">
    Gateway bir arka plan hizmeti (launchd/systemd) olarak çalışıyorsa `HUGGINGFACE_HUB_TOKEN` veya `HF_TOKEN` değerinin bu süreç tarafından kullanılabildiğinden emin olun (örneğin `~/.openclaw/.env` içinde veya `env.shellEnv` aracılığıyla).

    <Note>
    OpenClaw hem `HUGGINGFACE_HUB_TOKEN` hem de `HF_TOKEN` değerini kabul eder. İkisi de ayarlanmışsa `HUGGINGFACE_HUB_TOKEN` önceliklidir.
    </Note>

  </Accordion>

  <Accordion title="Yapılandırma: Yedek model içeren DeepSeek R1">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "huggingface/deepseek-ai/DeepSeek-R1",
            fallbacks: ["huggingface/openai/gpt-oss-120b"],
          },
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1" },
            "huggingface/openai/gpt-oss-120b": { alias: "GPT-OSS 120B" },
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="Yapılandırma: En ucuz ve en hızlı çeşitleriyle DeepSeek">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1" },
            "huggingface/deepseek-ai/DeepSeek-R1:cheapest": { alias: "DeepSeek R1 (cheapest)" },
            "huggingface/deepseek-ai/DeepSeek-R1:fastest": { alias: "DeepSeek R1 (fastest)" },
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="Yapılandırma: Diğer adlarla DeepSeek + GPT-OSS">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "huggingface/deepseek-ai/DeepSeek-V3.1",
            fallbacks: ["huggingface/openai/gpt-oss-120b"],
          },
          models: {
            "huggingface/deepseek-ai/DeepSeek-V3.1": { alias: "DeepSeek V3.1" },
            "huggingface/openai/gpt-oss-120b": { alias: "GPT-OSS 120B" },
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Tüm sağlayıcılara, model referanslarına ve yük devretme davranışına genel bakış.
  </Card>
  <Card title="Model seçimi" href="/tr/concepts/models" icon="brain">
    Modellerin nasıl seçilip yapılandırılacağı.
  </Card>
  <Card title="Inference Providers belgeleri" href="https://huggingface.co/docs/inference-providers" icon="book">
    Resmî Hugging Face Inference Providers belgeleri.
  </Card>
  <Card title="Yapılandırma" href="/tr/gateway/configuration" icon="gear">
    Tam yapılandırma referansı.
  </Card>
</CardGroup>
