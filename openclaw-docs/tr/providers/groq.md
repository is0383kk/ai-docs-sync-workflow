---
read_when:
    - OpenClaw ile Groq kullanmak istiyorsunuz
    - API anahtarı ortam değişkeni veya CLI kimlik doğrulama seçeneği gereklidir
    - Groq üzerinde Whisper ses transkripsiyonunu yapılandırıyorsunuz
summary: Groq kurulumu (kimlik doğrulama + model seçimi + Whisper transkripsiyonu)
title: Groq
x-i18n:
    generated_at: "2026-07-26T22:58:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f04f9365127c72aa2f976f453e5d11657b19d6b4a57de1179b88924744db1dc1
    source_path: providers/groq.md
    workflow: 16
---

[Groq](https://groq.com), özel LPU donanımı kullanarak açık ağırlıklı modellerde (Llama, Gemma, Kimi, Qwen, GPT OSS ve daha fazlası) ultra hızlı çıkarım sağlar. Groq Plugin'i hem OpenAI uyumlu bir sohbet sağlayıcısını hem de ses medyası anlama sağlayıcısını kaydeder.

| Özellik                | Değer                                    |
| ---------------------- | ---------------------------------------- |
| Sağlayıcı kimliği      | `groq`                                   |
| Plugin                 | resmi harici paket                       |
| Kimlik doğrulama ortam değişkeni | `GROQ_API_KEY`                           |
| API                    | OpenAI uyumlu (`openai-completions`) |
| Temel URL              | `https://api.groq.com/openai/v1`         |
| Ses transkripsiyonu    | `whisper-large-v3-turbo` (varsayılan)       |
| Önerilen varsayılan sohbet modeli | `groq/llama-3.3-70b-versatile`           |

## Plugin'i yükleme

Resmi Plugin'i yükleyin, ardından Gateway'i yeniden başlatın:

```bash
openclaw plugins install @openclaw/groq-provider
openclaw gateway restart
```

## Başlarken

<Steps>
  <Step title="API anahtarı alma">
    [console.groq.com/keys](https://console.groq.com/keys) adresinde bir API anahtarı oluşturun.
  </Step>
  <Step title="API anahtarını ayarlama">
    ```bash
export GROQ_API_KEY=gsk_...
```
  </Step>
  <Step title="Varsayılan model ayarlama">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "groq/llama-3.3-70b-versatile" },
        },
      },
    }
    ```
  </Step>
  <Step title="Kataloğa erişilebildiğini doğrulama">
    ```bash
    openclaw models list --provider groq
    ```
  </Step>
</Steps>

### Yapılandırma dosyası örneği

```json5
{
  env: { GROQ_API_KEY: "gsk_..." },
  agents: {
    defaults: {
      model: { primary: "groq/llama-3.3-70b-versatile" },
    },
  },
}
```

## Yerleşik katalog

OpenClaw, hem akıl yürütme özellikli hem de akıl yürütme özelliği olmayan girdiler içeren, manifest destekli bir Groq kataloğuyla birlikte gelir. Yüklü sürümünüze ait statik satırları görmek için `openclaw models list --provider groq` komutunu çalıştırın veya Groq'nun yetkili listesi için [console.groq.com/docs/models](https://console.groq.com/docs/models) adresini kontrol edin.

| Model referansı                                  | Ad                      | Akıl yürütme | Girdi        | Bağlam  |
| ------------------------------------------------ | ----------------------- | ------------ | ------------ | ------- |
| `groq/llama-3.3-70b-versatile`                   | Llama 3.3 70B Versatile | hayır        | metin        | 131,072 |
| `groq/llama-3.1-8b-instant`                      | Llama 3.1 8B Instant    | hayır        | metin        | 131,072 |
| `groq/meta-llama/llama-4-scout-17b-16e-instruct` | Llama 4 Scout 17B       | hayır        | metin + görsel | 131,072 |
| `groq/openai/gpt-oss-120b`                       | GPT OSS 120B            | evet         | metin        | 131,072 |
| `groq/openai/gpt-oss-20b`                        | GPT OSS 20B             | evet         | metin        | 131,072 |
| `groq/openai/gpt-oss-safeguard-20b`              | Safety GPT OSS 20B      | evet         | metin        | 131,072 |
| `groq/qwen/qwen3-32b`                            | Qwen3 32B               | evet         | metin        | 131,072 |
| `groq/groq/compound`                             | Compound                | evet         | metin        | 131,072 |
| `groq/groq/compound-mini`                        | Compound Mini           | evet         | metin        | 131,072 |

<Tip>
  Katalog her OpenClaw sürümüyle birlikte gelişir. `openclaw models list --provider groq`, yüklü sürümünüzün bildiği satırları gösterir; yeni eklenen veya kullanımdan kaldırılan modeller için [console.groq.com/docs/models](https://console.groq.com/docs/models) adresiyle karşılaştırın.
</Tip>

## Akıl yürütme modelleri

Groq akıl yürütme modelleri (yukarıdaki tabloda `reasoning: true`), OpenClaw'un ortak `/think` düzeylerini `low`, `medium` veya `high` şeklindeki `reasoning_effort` değerlerine eşler. `/think off` veya `/think none`, devre dışı bırakılmış bir değer göndermek yerine `reasoning_effort` öğesini istekten çıkarır.

Ortak `/think` düzeyleri ve OpenClaw'un bunları her sağlayıcı için nasıl çevirdiği hakkında bilgi edinmek üzere [Düşünme modları](/tr/tools/thinking) bölümüne bakın.

## Ses transkripsiyonu

Groq Plugin'i ayrıca sesli mesajların ortak `tools.media.audio` yüzeyi üzerinden yazıya dökülebilmesi için bir **ses medyası anlama sağlayıcısı** kaydeder.

| Özellik            | Değer                                     |
| ------------------ | ----------------------------------------- |
| Ortak yapılandırma yolu | `tools.media.audio`                       |
| Varsayılan temel URL | `https://api.groq.com/openai/v1`          |
| Varsayılan model   | `whisper-large-v3-turbo`                  |
| Otomatik öncelik   | 20                                        |
| API uç noktası     | OpenAI uyumlu `/audio/transcriptions` |

Groq'yu varsayılan ses arka ucu yapmak için:

```json5
{
  tools: {
    media: {
      audio: {
        models: [{ provider: "groq" }],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Daemon için ortam kullanılabilirliği">
    Gateway yönetilen bir hizmet (launchd, systemd, Docker) olarak çalışıyorsa `GROQ_API_KEY`, yalnızca etkileşimli kabuğunuz tarafından değil, bu işlem tarafından da görülebilmelidir.

    <Warning>
      Yalnızca etkileşimli bir kabukta dışa aktarılan anahtar, söz konusu ortam oraya da aktarılmadıkça launchd veya systemd daemon'una yardımcı olmaz. Anahtarı gateway işlemi tarafından okunabilir hâle getirmek için `~/.openclaw/.env` içinde veya `env.shellEnv` aracılığıyla ayarlayın.
    </Warning>

  </Accordion>

  <Accordion title="Özel Groq model kimlikleri">
    OpenClaw çalışma zamanında herhangi bir Groq model kimliğini kabul eder. Groq tarafından gösterilen tam kimliği kullanın ve başına `groq/` ekleyin. Statik katalog yaygın durumları kapsar; katalogda bulunmayan kimlikler varsayılan OpenAI uyumlu şablona aktarılır.

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "groq/<your-model-id>" },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="Model sağlayıcıları" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Düşünme modları" href="/tr/tools/thinking" icon="brain">
    Akıl yürütme çabası düzeyleri ve sağlayıcı politikası etkileşimi.
  </Card>
  <Card title="Yapılandırma başvurusu" href="/tr/gateway/configuration-reference" icon="gear">
    Sağlayıcı ve ses ayarlarını içeren tam yapılandırma şeması.
  </Card>
  <Card title="Groq Console" href="https://console.groq.com" icon="arrow-up-right-from-square">
    Groq kontrol paneli, API belgeleri ve fiyatlandırma.
  </Card>
</CardGroup>
