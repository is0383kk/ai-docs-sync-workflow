---
read_when:
    - OpenClaw'u yerel bir vLLM sunucusuyla çalıştırmak istiyorsunuz
    - Kendi modellerinizle OpenAI uyumlu /v1 uç noktaları istiyorsunuz
summary: OpenClaw'ı vLLM ile çalıştırma (OpenAI uyumlu yerel sunucu)
title: vLLM
x-i18n:
    generated_at: "2026-07-26T22:59:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98d1044c0a82efb6c9937e961d765d0cfcea8664cbaa043168921b457756512c
    source_path: providers/vllm.md
    workflow: 16
---

vLLM, açık kaynaklı (ve bazı özel) modelleri **OpenAI uyumlu** bir HTTP API aracılığıyla sunar. OpenClaw, `openai-completions` API'sini kullanarak bağlanır ve `VLLM_API_KEY` ile etkinleştirdiğinizde modelleri **otomatik olarak keşfedebilir**.

| Özellik           | Değer                                      |
| ----------------- | ------------------------------------------ |
| Sağlayıcı kimliği | `vllm`                         |
| API               | `openai-completions` (OpenAI uyumlu)         |
| Kimlik doğrulama  | `VLLM_API_KEY` ortam değişkeni          |
| Varsayılan temel URL | `http://127.0.0.1:8000/v1`                      |
| Akış kullanımı    | Desteklenir (`stream_options.include_usage`)           |

## Başlarken

<Steps>
  <Step title="vLLM'yi OpenAI uyumlu bir sunucuyla başlatın">
    Temel URL'niz `/v1` uç noktalarını (`/v1/models`, `/v1/chat/completions`) sunmalıdır. vLLM genellikle şu adreste çalışır:

    ```text
    http://127.0.0.1:8000/v1
    ```

  </Step>
  <Step title="API anahtarı ortam değişkenini ayarlayın">
    Sunucunuz kimlik doğrulamayı zorunlu kılmıyorsa boş olmayan herhangi bir değer kullanılabilir:

    ```bash
    export VLLM_API_KEY="vllm-local"
    ```

  </Step>
  <Step title="Bir model seçin">
    Aşağıdakini vLLM model kimliklerinizden biriyle değiştirin:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "vllm/your-model-id" },
        },
      },
    }
    ```

  </Step>
  <Step title="Modelin kullanılabilir olduğunu doğrulayın">
    ```bash
    openclaw models list --provider vllm
    ```
  </Step>
</Steps>

<Tip>
Etkileşimsiz kurulumda (CI, betik çalıştırma) temel URL'yi, anahtarı ve modeli doğrudan iletin:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice vllm \
  --custom-base-url "http://127.0.0.1:8000/v1" \
  --custom-api-key "vllm-local" \
  --custom-model-id "your-model-id"
```

</Tip>

## Model keşfi (örtük sağlayıcı)

`VLLM_API_KEY` ayarlandığında (veya bir kimlik doğrulama profili bulunduğunda) ve `models.providers.vllm` **tanımlı olmadığında**, OpenClaw `GET http://127.0.0.1:8000/v1/models` sorgusu yapar ve döndürülen kimlikleri model girdilerine dönüştürür.

<Note>
`models.providers.vllm` değerini açıkça ayarlarsanız OpenClaw yalnızca bildirdiğiniz modelleri kullanır. OpenClaw'ın yapılandırılmış sağlayıcının `/models` uç noktasını da sorgulaması ve duyurulan tüm vLLM modellerini dahil etmesi için `agents.defaults.models` öğesine `"vllm/*": {}` ekleyin.
</Note>

## Açık yapılandırma

vLLM farklı bir ana makine veya bağlantı noktasında çalışıyorsa, `contextWindow`/`maxTokens` değerlerini sabitlemek istiyorsanız, sunucunuz gerçek bir API anahtarı gerektiriyorsa ya da güvenilir bir geri döngü, LAN veya Tailscale uç noktasına bağlanıyorsanız açıkça yapılandırın:

```json5
{
  models: {
    providers: {
      vllm: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "${VLLM_API_KEY}",
        api: "openai-completions",
        timeoutSeconds: 300, // İsteğe bağlı: yavaş yerel modeller için istek zaman aşımını uzatır
        models: [
          {
            id: "your-model-id",
            name: "Yerel vLLM Modeli",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Her modeli listelemeden sağlayıcıyı dinamik tutmak için görünür model kataloğuna bir joker karakter ekleyin:

```json5
{
  agents: {
    defaults: {
      models: {
        "vllm/*": {},
      },
    },
  },
}
```

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Proxy tarzı davranış">
    vLLM, yerel bir OpenAI uç noktası olarak değil, proxy tarzı OpenAI uyumlu bir `/v1` arka ucu olarak ele alınır:

    | Davranış                               | Uygulanıyor mu?                          |
    | -------------------------------------- | ---------------------------------------- |
    | Yerel OpenAI istek biçimlendirmesi     | Hayır                                    |
    | `service_tier`                     | Gönderilmez                              |
    | Responses `store`           | Gönderilmez                              |
    | İstem önbelleği ipuçları                | Gönderilmez                              |
    | OpenAI akıl yürütme uyumluluk yükü biçimlendirmesi | Uygulanmaz                  |
    | Gizli OpenClaw ilişkilendirme üstbilgileri | Özel temel URL'lere eklenmez          |

  </Accordion>

  <Accordion title="Qwen düşünme denetimleri">
    Qwen modellerinde, sunucu Qwen sohbet şablonu anahtar sözcük bağımsız değişkenlerini bekliyorsa model satırında `compat.thinkingFormat: "qwen-chat-template"` ayarlayın. Qwen sohbet şablonundaki düşünme özelliği OpenAI tarzı bir çaba kademesi değil, açık/kapalı bayrağı olduğundan bu modeller ikili bir `/think` profili (`off`, `on`) sunar.

    ```json5
    {
      models: {
        providers: {
          vllm: {
            models: [
              {
                id: "Qwen/Qwen3-8B",
                name: "Qwen3 8B",
                reasoning: true,
                compat: { thinkingFormat: "qwen-chat-template" },
              },
            ],
          },
        },
      },
    }
    ```

    OpenClaw, `/think off` değerini şuna eşler:

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "preserve_thinking": true
      }
    }
    ```

    `off` dışındaki düşünme düzeyleri `enable_thinking: true` gönderir. Uç noktanız bunun yerine DashScope tarzı üst düzey bayraklar bekliyorsa istek kökünde `enable_thinking` göndermek için `compat.thinkingFormat: "qwen"` kullanın.

  </Accordion>

  <Accordion title="Nemotron 3 düşünme denetimleri">
    Düşünme kapalıyken `vllm/nemotron-3-*` modelleri için paketle gelen plugin şunu gönderir:

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "force_nonempty_content": true
      }
    }
    ```

    Bu değerleri özelleştirmek için model parametreleri altında `chat_template_kwargs` ayarlayın. `params.extra_body.chat_template_kwargs` değerini de ayarlarsanız `extra_body` son istek gövdesi geçersiz kılması olduğundan bu değer öncelikli olur.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/nemotron-3-super": {
              params: {
                chat_template_kwargs: {
                  enable_thinking: false,
                  force_nonempty_content: true,
                },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Qwen araç çağrıları metin olarak görünüyor">
    Öncelikle vLLM'nin model için doğru araç çağrısı ayrıştırıcısı ve sohbet şablonuyla başlatıldığını doğrulayın. vLLM, Qwen2.5 modelleri için `hermes`, Qwen3-Coder modelleri için ise `qwen3_xml` kullanımını belgeler.

    Belirtiler: Skills/araçlar hiç çalışmaz, asistan `{"name":"read","arguments":...}` gibi ham JSON/XML yazdırır veya OpenClaw `tool_choice: "auto"` gönderdiğinde vLLM boş bir `tool_calls` dizisi döndürür.

    Bazı Qwen/vLLM birleşimleri yalnızca istek `tool_choice: "required"` kullandığında yapılandırılmış araç çağrıları döndürür. `params.extra_body` ile model başına zorunlu kılın:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/Qwen-Qwen2.5-Coder-32B-Instruct": {
              params: {
                extra_body: {
                  tool_choice: "required",
                },
              },
            },
          },
        },
      },
    }
    ```

    Model kimliğini `openclaw models list --provider vllm` çıktısındaki tam kimlikle değiştirin veya aynı geçersiz kılmayı CLI üzerinden uygulayın:

    ```bash
    openclaw config set agents.defaults.models '{"vllm/Qwen-Qwen2.5-Coder-32B-Instruct":{"params":{"extra_body":{"tool_choice":"required"}}}}' --strict-json --merge
    ```

    Bu, isteğe bağlı bir geçici çözümdür: araçların bulunduğu her turu bir araç çağrısı yapmaya zorlar; bu nedenle yalnızca bunun kabul edilebilir olduğu özel bir model girdisinde kullanın. Bunu tüm vLLM modelleri için genel varsayılan olarak ayarlamayın ve rastgele asistan metnini yürütülebilir araç çağrılarına dönüştüren bir proxy ile birlikte kullanmayın.

  </Accordion>

  <Accordion title="Özel temel URL">
    vLLM sunucunuz varsayılan olmayan bir ana makine veya bağlantı noktasında çalışıyorsa açık sağlayıcı yapılandırmasında `baseUrl` ayarlayın:

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:9000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [
              {
                id: "my-custom-model",
                name: "Uzak vLLM Modeli",
                reasoning: false,
                input: ["text"],
                contextWindow: 64000,
                maxTokens: 4096,
              },
            ],
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## Sorun giderme

<AccordionGroup>
  <Accordion title="İlk yanıt yavaş veya uzak sunucu zaman aşımına uğruyor">
    Büyük yerel modeller, uzak LAN ana makineleri veya tailnet bağlantıları için sağlayıcı kapsamlı bir istek zaman aşımı ayarlayın:

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:8000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [{ id: "your-model-id", name: "Yerel vLLM Modeli" }],
          },
        },
      },
    }
    ```

    `timeoutSeconds` yalnızca vLLM model HTTP isteklerine uygulanır: bağlantı kurulumu, yanıt üstbilgileri, gövde akışı ve korumalı getirme işleminin toplam iptali. Ayrıca LLM boşta kalma/akış gözetleyicisi üst sınırını bu sağlayıcı için örtük ~120s varsayılanının üzerine çıkarır. Tüm ajan çalışmasını denetleyen `agents.defaults.timeoutSeconds` değerini artırmak yerine bunu tercih edin.

  </Accordion>

  <Accordion title="Sunucuya erişilemiyor">
    vLLM sunucusunun çalıştığını ve erişilebilir olduğunu denetleyin:

    ```bash
    curl http://127.0.0.1:8000/v1/models
    ```

    Bağlantı hatası görürseniz ana makineyi, bağlantı noktasını ve vLLM'nin OpenAI uyumlu sunucu modunda başlatıldığını doğrulayın. OpenClaw, geri döngü, LAN ve Tailscale uç noktalarındaki korumalı model istekleri için tam olarak yapılandırılmış `models.providers.vllm.baseUrl` kaynağına güvenir. Meta veri/yerel bağlantı kaynakları açıkça etkinleştirilmedikçe engellenmeye devam eder. Yalnızca vLLM isteklerinin başka bir özel kaynağa ulaşması gerektiğinde `models.providers.vllm.request.allowPrivateNetwork: true`, tam kaynak güveninden vazgeçmek için ise `false` ayarlayın.

  </Accordion>

  <Accordion title="İsteklerde kimlik doğrulama hataları">
    İstekler kimlik doğrulama hatalarıyla başarısız oluyorsa sunucu yapılandırmanızla eşleşen gerçek bir `VLLM_API_KEY` ayarlayın veya sağlayıcıyı `models.providers.vllm` altında açıkça yapılandırın.

    <Tip>
    vLLM sunucunuz kimlik doğrulamayı zorunlu kılmıyorsa `VLLM_API_KEY` için boş olmayan herhangi bir değer, OpenClaw açısından etkinleştirme sinyali olarak çalışır.
    </Tip>

  </Accordion>

  <Accordion title="Hiçbir model keşfedilmedi">
    Otomatik keşif için `VLLM_API_KEY` ayarlanmış olmalıdır. `models.providers.vllm` tanımladıysanız `agents.defaults.models` içinde `"vllm/*": {}` bulunmadığı sürece OpenClaw yalnızca bildirdiğiniz modelleri kullanır.
  </Accordion>

  <Accordion title="Araçlar ham metin olarak işleniyor">
    Bir Qwen modeli Skill çalıştırmak yerine JSON/XML araç söz dizimini yazdırıyorsa:

    - vLLM'yi bu model için doğru ayrıştırıcı/şablonla başlatın.
    - Tam model kimliğini `openclaw models list --provider vllm` ile doğrulayın.
    - Yalnızca `tool_choice: "auto"` hâlâ boş veya yalnızca metin içeren araç çağrıları döndürüyorsa modele özel bir `params.extra_body.tool_choice: "required"` geçersiz kılması ekleyin.

  </Accordion>
</AccordionGroup>

<Warning>
Daha fazla yardım: [Sorun giderme](/tr/help/troubleshooting) ve [SSS](/tr/help/faq).
</Warning>

## İlgili

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="OpenAI" href="/tr/providers/openai" icon="bolt">
    Yerel OpenAI sağlayıcısı ve OpenAI uyumlu rota davranışı.
  </Card>
  <Card title="OAuth ve kimlik doğrulama" href="/tr/gateway/authentication" icon="key">
    Kimlik doğrulama ayrıntıları ve kimlik bilgilerini yeniden kullanma kuralları.
  </Card>
  <Card title="Sorun giderme" href="/tr/help/troubleshooting" icon="wrench">
    Yaygın sorunlar ve bunların nasıl çözüleceği.
  </Card>
</CardGroup>
