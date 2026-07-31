---
read_when:
    - OpenClaw'da açık modelleri ücretsiz kullanmak istiyorsunuz
    - NVIDIA_API_KEY ayarını yapmanız gerekir
    - Nemotron 3 Ultra'yı NVIDIA üzerinden kullanmak istiyorsunuz
summary: OpenClaw'da NVIDIA'nın OpenAI uyumlu API'sini kullanma
title: NVIDIA
x-i18n:
    generated_at: "2026-07-26T22:59:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b5ac7bcc19400a661b2f2861a1dd4d2306c94e445783929e342e9184003314e9
    source_path: providers/nvidia.md
    workflow: 16
---

NVIDIA, açık modelleri OpenAI uyumlu bir API üzerinden ücretsiz olarak
`https://integrate.api.nvidia.com/v1` adresinde sunar; kimlik doğrulama,
[build.nvidia.com](https://build.nvidia.com/settings/api-keys) üzerinden alınan bir API anahtarıyla yapılır. OpenClaw,
NVIDIA sağlayıcısı için varsayılan olarak uzun bağlamlı, ajan tabanlı çalışmalar için NVIDIA'nın toplam 550B / etkin 55B
akıl yürütme modeli Nemotron 3 Ultra'yı kullanır.

## Başlarken

<Steps>
  <Step title="API anahtarınızı alın">
    [build.nvidia.com](https://build.nvidia.com/settings/api-keys) üzerinden bir API anahtarı oluşturun.
  </Step>
  <Step title="Anahtarı dışa aktarın ve ilk kurulumu çalıştırın">
    ```bash
    export NVIDIA_API_KEY="nvapi-..."
    openclaw onboard --auth-choice nvidia-api-key
    ```
  </Step>
  <Step title="Bir NVIDIA modeli ayarlayın">
    ```bash
    openclaw models set nvidia/nvidia/nemotron-3-ultra-550b-a55b
    ```
  </Step>
</Steps>

Etkileşimsiz kurulum için anahtarı doğrudan iletin:

```bash
openclaw onboard --auth-choice nvidia-api-key --nvidia-api-key "nvapi-..."
```

<Warning>
`--nvidia-api-key`, anahtarın kabuk geçmişine ve `ps` çıktısına kaydedilmesine neden olur. Mümkün olduğunda
`NVIDIA_API_KEY` ortam değişkenini tercih edin.
</Warning>

## Yapılandırma örneği

```json5
{
  env: { NVIDIA_API_KEY: "nvapi-..." },
  models: {
    providers: {
      nvidia: {
        baseUrl: "https://integrate.api.nvidia.com/v1",
        api: "openai-completions",
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "nvidia/nvidia/nemotron-3-ultra-550b-a55b" },
    },
  },
}
```

## Öne çıkan katalog

Bir NVIDIA API anahtarı yapılandırıldığında, kurulum ve model seçimi yolları
NVIDIA'nın herkese açık öne çıkan model kataloğunu
`https://assets.ngc.nvidia.com/products/api-catalog/featured-models.json` üzerinden alır ve
sonucu 24 saat boyunca önbelleğe alır (ilk 32 girdi, ücretsiz metin girdili
satırlar olarak içe aktarılır). Böylece build.nvidia.com üzerindeki yeni öne çıkan modeller, bir OpenClaw sürümü beklenmeden kurulum ve
model seçimi yüzeylerinde görünür. Canlı akış kullanılabildiğinde, döndürülen ilk model
NVIDIA kurulumu sırasında önceden seçilmiş seçenek olur.

Alma işlemi, `assets.ngc.nvidia.com` için sabit bir HTTPS ana bilgisayar ilkesi kullanır. Herhangi bir
NVIDIA API anahtarı yapılandırılmamışsa ya da akış kullanılamıyor veya hatalı biçimlendirilmişse
OpenClaw, aşağıdaki paketlenmiş kataloğa ve paketlenmiş varsayılana geri döner.

## Nemotron 3 Ultra

Nemotron 3 Ultra, OpenClaw'daki varsayılan NVIDIA modelidir. NVIDIA'nın
[`nvidia/nemotron-3-ultra-550b-a55b`](https://build.nvidia.com/nvidia/nemotron-3-ultra-550b-a55b) için build sayfası,
onu 1M token bağlam özelliğine sahip, kullanılabilir ücretsiz bir uç nokta olarak listeler.

Paketlenmiş Ultra satırı, normal sohbet çıktısının akıl yürütme metnini
açığa çıkarmak yerine görünür yanıtta kalması için varsayılan olarak
`chat_template_kwargs: { enable_thinking: false, force_nonempty_content: true }`
gönderir.

En yüksek kapasiteli varsayılan NVIDIA seçeneği için Ultra'yı kullanın. Daha küçük Nemotron 3 seçeneğini
istediğinizde Super'ı seçili tutun veya bağlamı, gecikme süresi ya da davranışı daha uygun olduğunda
NVIDIA'nın kataloğunda barındırılan üçüncü taraf modellerden birini seçin.

## Paketlenmiş geri dönüş kataloğu

Seçilebilir paketlenmiş satırlar, NVIDIA'nın öne çıkan model kataloğunun anlık görüntüsüdür. Kullanımdan kaldırılmış
uyumluluk satırları tam referansla çözümlenebilir olmaya devam eder ancak model
seçicilerinde görünmez.

| Model referansı                            | Ad                    | Bağlam    | En fazla çıktı |
| ------------------------------------------ | --------------------- | --------- | -------------- |
| `nvidia/nvidia/nemotron-3-ultra-550b-a55b` | Nemotron 3 Ultra 550B | 1,048,576 | 8,192          |
| `nvidia/nvidia/nemotron-3-super-120b-a12b` | Nemotron 3 Super 120B | 1,000,000 | 8,192          |
| `nvidia/z-ai/glm-5.2`                      | GLM 5.2               | 202,752   | 8,192          |
| `nvidia/moonshotai/kimi-k2.6`              | Kimi K2.6             | 262,144   | 8,192          |
| `nvidia/minimaxai/minimax-m3`              | Minimax M3            | 196,608   | 8,192          |
| `nvidia/deepseek-ai/deepseek-v4-pro`       | DeepSeek V4 Pro       | 262,144   | 16,384         |
| `nvidia/qwen/qwen3.5-397b-a17b`            | Qwen3.5 397B A17B     | 262,144   | 16,384         |

Tam uyumluluk kataloğu, mevcut yapılandırmalar için yayımlanmış şu referansları da
korur: `nvidia/moonshotai/kimi-k2.5`, `nvidia/z-ai/glm-5.1`,
`nvidia/minimaxai/minimax-m2.5`, `nvidia/z-ai/glm5` ve
`nvidia/minimaxai/minimax-m2.7`. Bunlar tam referansla kullanılabilir olmaya devam eder ancak
ilk kurulumda veya model seçicilerinde hiçbir zaman görünmez.

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Otomatik etkinleştirme davranışı">
    `NVIDIA_API_KEY` ortam değişkeni ayarlandığında veya ilk kurulum sırasında bir anahtar
    kaydedildiğinde sağlayıcı otomatik olarak etkinleşir. Anahtar dışında açık bir sağlayıcı yapılandırması
    gerekmez.
  </Accordion>

  <Accordion title="Katalog ve fiyatlandırma">
    NVIDIA kimlik doğrulaması yapılandırıldığında OpenClaw, NVIDIA'nın herkese açık öne çıkan model kataloğunu
    tercih eder ve bunu 24 saat boyunca önbelleğe alır. Paketlenmiş seçilebilir geri dönüş, NVIDIA'nın
    öne çıkan model kataloğunun statik bir anlık görüntüsüdür; kullanımdan kaldırılmış tam referanslı
    uyumluluk satırları model seçicilerinden gizlenir. NVIDIA şu anda listelenen modeller için ücretsiz API erişimi sunduğundan,
    kaynakta maliyetler varsayılan olarak `0` değerine ayarlanır.
  </Accordion>

  <Accordion title="OpenAI uyumlu uç nokta">
    OpenClaw, standart `/v1` sohbet tamamlama rotasına karşı
    `openai-completions` bağdaştırıcısıyla NVIDIA ile iletişim kurar. OpenAI uyumlu tüm araçlar,
    NVIDIA temel URL'siyle doğrudan çalışmalıdır.
  </Accordion>

  <Accordion title="Nemotron 3 Ultra akıl yürütme parametreleri">
    NVIDIA'nın Ultra örnek isteği, akıl yürütme çıktısı için `chat_template_kwargs.enable_thinking`
    ve `reasoning_budget` kullanır. OpenClaw'ın paketlenmiş Ultra satırı,
    normal sohbet kullanımı için şablon düşünmesini varsayılan olarak devre dışı bırakır. NVIDIA akıl yürütme çıktısını
    etkinleştirmeniz veya NVIDIA'ya özgü diğer istek alanlarını zorlamanız gerekiyorsa
    model başına parametreleri ayarlayın ve sağlayıcıya özgü geçersiz kılmaları
    NVIDIA modeliyle sınırlı tutun:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "nvidia/nvidia/nemotron-3-ultra-550b-a55b": {
              params: {
                chat_template_kwargs: { enable_thinking: true },
                extra_body: { reasoning_budget: 16384 },
              },
            },
          },
        },
      },
    }
    ```

    `params.chat_template_kwargs`, nesnenin tamamını değiştirmek yerine istekte zaten bulunan
    `chat_template_kwargs` ile birleştirilir.
    `params.extra_body`, OpenAI uyumlu son istek gövdesi geçersiz kılmasıdır
    ve çakışan yük anahtarlarının üzerine yazar; bu nedenle yalnızca NVIDIA'nın
    seçilen uç nokta için belgelediği alanlarda kullanın.

  </Accordion>

  <Accordion title="Yavaş özel sağlayıcı yanıtları">
    NVIDIA'da barındırılan bazı özel modeller, ilk yanıt parçasını göndermeden önce varsayılan ~120s
    model boşta kalma gözetim süresinden daha uzun sürebilir. Özel
    NVIDIA sağlayıcı girdileri için tüm ajan çalışma zamanı zaman aşımı yerine sağlayıcı zaman aşımını artırın;
    `timeoutSeconds`, sağlayıcı HTTP isteklerini kapsar ve
    bu sağlayıcının boşta kalma/akış gözetimi üst sınırını yükseltir:

    ```json5
    {
      models: {
        providers: {
          "custom-integrate-api-nvidia-com": {
            baseUrl: "https://integrate.api.nvidia.com/v1",
            api: "openai-completions",
            apiKey: "NVIDIA_API_KEY",
            timeoutSeconds: 300,
          },
        },
      },
      agents: {
        defaults: {
          models: {
            "custom-integrate-api-nvidia-com/meta/llama-3.1-70b-instruct": {
              params: { thinking: "off" },
            },
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

<Tip>
NVIDIA modellerinin kullanımı şu anda ücretsizdir. En güncel kullanılabilirlik ve
hız sınırı ayrıntıları için [build.nvidia.com](https://build.nvidia.com/) adresini kontrol edin.
</Tip>

## İlgili

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Yapılandırma referansı" href="/tr/gateway/configuration-reference" icon="gear">
    Ajanlar, modeller ve sağlayıcılar için tam yapılandırma referansı.
  </Card>
</CardGroup>
