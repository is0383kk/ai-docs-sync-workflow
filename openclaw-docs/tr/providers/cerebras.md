---
read_when:
    - Cerebras'ı OpenClaw ile kullanmak istiyorsunuz
    - Cerebras API anahtarı ortam değişkeni veya CLI kimlik doğrulama seçeneği gereklidir
summary: Cerebras kurulumu (kimlik doğrulama + model seçimi)
title: Cerebras
x-i18n:
    generated_at: "2026-07-26T23:35:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 716eef83155ef80d9aa61bd55ed83e3e38ad22720ae055bce7eb9c2cbfb6cf41
    source_path: providers/cerebras.md
    workflow: 16
---

[Cerebras](https://www.cerebras.ai), özel çıkarım donanımında yüksek hızlı, OpenAI uyumlu çıkarım sağlar. Plugin, statik iki modelli bir katalogla sunulur (canlı keşif yoktur).

| Özellik              | Değer                                                     |
| -------------------- | --------------------------------------------------------- |
| Sağlayıcı kimliği    | `cerebras`                                        |
| Plugin               | resmi harici paket (`@openclaw/cerebras-provider`)                   |
| Kimlik doğrulama ortam değişkeni | `CEREBRAS_API_KEY`                            |
| İlk kurulum bayrağı  | `--auth-choice cerebras-api-key`                                        |
| Doğrudan CLI bayrağı | `--cerebras-api-key <key>`                                        |
| API                  | OpenAI uyumlu (`openai-completions`)                        |
| Temel URL            | `https://api.cerebras.ai/v1`                                        |
| Varsayılan model     | `cerebras/zai-glm-4.7`                                        |

## Plugin'i yükleme

```bash
openclaw plugins install @openclaw/cerebras-provider
openclaw gateway restart
```

## Başlarken

<Steps>
  <Step title="Bir API anahtarı alma">
    [Cerebras Cloud Console](https://cloud.cerebras.ai) içinde bir API anahtarı oluşturun.
  </Step>
  <Step title="İlk kurulumu çalıştırma">
    <CodeGroup>

```bash İlk kurulum
openclaw onboard --auth-choice cerebras-api-key
```

```bash Doğrudan bayrak
openclaw onboard --non-interactive \
  --auth-choice cerebras-api-key \
  --cerebras-api-key "$CEREBRAS_API_KEY"
```

```bash Yalnızca ortam değişkeni
export CEREBRAS_API_KEY=csk-...
```

    </CodeGroup>

  </Step>
  <Step title="Modellerin kullanılabilir olduğunu doğrulama">
    ```bash
    openclaw models list --provider cerebras
    ```

    Her iki statik modeli de listeler. `CEREBRAS_API_KEY` çözümlenmemişse `openclaw models status --json`, eksik kimlik bilgisini `auth.unusableProfiles` altında bildirir.

  </Step>
</Steps>

## Etkileşimsiz kurulum

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice cerebras-api-key \
  --cerebras-api-key "$CEREBRAS_API_KEY"
```

## Yerleşik katalog

Her iki model de 128k bağlam penceresini ve en fazla 8.192 çıktı token'ını destekler.

| Model referansı          | Ad           | Akıl yürütme | Notlar                                      |
| ------------------------ | ------------ | ------------ | ------------------------------------------- |
| `cerebras/zai-glm-4.7`       | Z.ai GLM 4.7 | evet         | Varsayılan model; önizleme akıl yürütme modeli |
| `cerebras/gpt-oss-120b`       | GPT OSS 120B | evet         | Üretim akıl yürütme modeli                  |

## Manuel yapılandırma

Çoğu kurulum yalnızca API anahtarını gerektirir. Model meta verilerini geçersiz kılmak veya statik kataloğa karşı `mode: "merge"` içinde çalışmak için açık `models.providers.cerebras` yapılandırmasını kullanın:

```json5
{
  env: { CEREBRAS_API_KEY: "csk-..." },
  agents: {
    defaults: {
      model: { primary: "cerebras/zai-glm-4.7" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      cerebras: {
        baseUrl: "https://api.cerebras.ai/v1",
        apiKey: "${CEREBRAS_API_KEY}",
        api: "openai-completions",
        models: [
          { id: "zai-glm-4.7", name: "Z.ai GLM 4.7" },
          { id: "gpt-oss-120b", name: "GPT OSS 120B" },
        ],
      },
    },
  },
}
```

<Note>
Gateway bir daemon (launchd, systemd, Docker) olarak çalışıyorsa `CEREBRAS_API_KEY` değerinin bu işlem tarafından kullanılabildiğinden emin olun; örneğin `~/.openclaw/.env` içinde veya `env.shellEnv` aracılığıyla. Yalnızca etkileşimli bir kabukta dışa aktarılan anahtar, ortam ayrı olarak içe aktarılmadıkça yönetilen bir hizmete yardımcı olmaz.
</Note>

## İlgili

<CardGroup cols={2}>
  <Card title="Model sağlayıcıları" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Düşünme modları" href="/tr/tools/thinking" icon="brain">
    Akıl yürütme özelliğine sahip iki Cerebras modeli için akıl yürütme eforu düzeyleri.
  </Card>
  <Card title="Yapılandırma referansı" href="/tr/gateway/config-agents#agent-defaults" icon="gear">
    Agent varsayılanları ve model yapılandırması.
  </Card>
  <Card title="Modeller hakkında SSS" href="/tr/help/faq-models" icon="circle-question">
    Kimlik doğrulama profilleri, model değiştirme ve "profil yok" hatalarını çözme.
  </Card>
</CardGroup>
