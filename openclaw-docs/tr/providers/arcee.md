---
read_when:
    - Arcee AI'ı OpenClaw ile kullanmak istiyorsunuz
    - API anahtarı ortam değişkeni veya CLI kimlik doğrulama seçeneği gereklidir
summary: Arcee AI kurulumu (kimlik doğrulama + model seçimi)
title: Arcee AI
x-i18n:
    generated_at: "2026-07-26T23:56:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a4c2fc7b8d86dd0d2a300dfc48951657cbcfcd9250016f52c1804777b2966e11
    source_path: providers/arcee.md
    workflow: 16
---

[Arcee AI](https://arcee.ai), OpenAI uyumlu bir API üzerinden uzmanlar karışımı Trinity model ailesini sunar. Tüm Trinity modelleri Apache 2.0 lisanslıdır. Arcee, çekirdekle birlikte paketlenmeyen resmî bir OpenClaw pluginidir; bu nedenle ilk kullanıma hazırlamadan önce bir kurulum adımı gerekir.

Arcee modellerine doğrudan Arcee platformu veya [OpenRouter](/tr/providers/openrouter) üzerinden erişin.

| Özellik   | Değer                                                                                 |
| --------- | ------------------------------------------------------------------------------------- |
| Sağlayıcı | `arcee`                                                                    |
| Kimlik doğrulama | `ARCEEAI_API_KEY` (doğrudan) veya `OPENROUTER_API_KEY` (OpenRouter üzerinden) |
| API       | OpenAI uyumlu                                                                         |
| Temel URL | `https://api.arcee.ai/api/v1` (doğrudan) veya `https://openrouter.ai/api/v1` (OpenRouter)                   |

## Plugini kurma

```bash
openclaw plugins install @openclaw/arcee-provider
openclaw gateway restart
```

## Başlarken

<Tabs>
  <Tab title="Doğrudan (Arcee platformu)">
    <Steps>
      <Step title="API anahtarı alma">
        [Arcee AI](https://chat.arcee.ai/) üzerinden bir API anahtarı oluşturun.
      </Step>
      <Step title="İlk kullanıma hazırlamayı çalıştırma">
        ```bash
        openclaw onboard --auth-choice arceeai-api-key
        ```
      </Step>
      <Step title="Varsayılan model ayarlama">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```
      </Step>
    </Steps>
  </Tab>

  <Tab title="OpenRouter üzerinden">
    <Steps>
      <Step title="API anahtarı alma">
        [OpenRouter](https://openrouter.ai/keys) üzerinden bir API anahtarı oluşturun.
      </Step>
      <Step title="İlk kullanıma hazırlamayı çalıştırma">
        ```bash
        openclaw onboard --auth-choice arceeai-openrouter
        ```
      </Step>
      <Step title="Varsayılan model ayarlama">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```

        Aynı model referansları hem doğrudan hem de OpenRouter kurulumlarında çalışır.
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Etkileşimsiz kurulum

<Tabs>
  <Tab title="Doğrudan (Arcee platformu)">
    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-api-key \
      --arceeai-api-key "$ARCEEAI_API_KEY"
    ```
  </Tab>

  <Tab title="OpenRouter üzerinden">
    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-openrouter \
      --openrouter-api-key "$OPENROUTER_API_KEY"
    ```
  </Tab>
</Tabs>

## Doğrudan Arcee kataloğu

| Model referansı                | Ad                     | Girdi | Bağlam  | Azami çıktı | Maliyet (1M başına giriş/çıkış) | Araçlar | Notlar                                    |
| ------------------------------ | ---------------------- | ----- | ------- | ---------- | -------------------------------- | ------- | ----------------------------------------- |
| `arcee/trinity-large-thinking` | Trinity Large Thinking | metin | 256K    | 80K        | $0.25 / $0.90                    | Hayır   | Varsayılan model; genişletilmiş düşünme   |
| `arcee/trinity-large-preview`  | Trinity Large Preview  | metin | 128K    | 16K        | $0.25 / $1.00                    | Evet    | Genel amaçlı; 400B parametre, 13B etkin   |
| `arcee/trinity-mini`           | Trinity Mini 26B       | metin | 128K    | 80K        | $0.045 / $0.15                   | Evet    | Hızlı ve uygun maliyetli; fonksiyon çağırma |

<Tip>
İlk kullanıma hazırlama ön ayarı, varsayılan model olarak `arcee/trinity-large-thinking` değerini ayarlar.
</Tip>

## OpenRouter kataloğu

OpenRouter ilk kullanıma hazırlama işlemi, `arcee/trinity-large-preview` ve `arcee/trinity-large-thinking` modellerini kullanıma sunar. OpenClaw, sağlayıcı nitelemeli bu model referanslarını yapılandırmada tutar ve OpenRouter'ın standart `arcee-ai/*` çalışma zamanı kimliklerini gönderir. Trinity Mini artık OpenRouter tarafından sunulmamaktadır; bu model için doğrudan Arcee API'sini kullanın.

## Desteklenen özellikler

| Özellik                                       | Destek                                      |
| --------------------------------------------- | ------------------------------------------- |
| Akış                                          | Evet                                        |
| Araç kullanımı / fonksiyon çağırma            | Evet (Trinity Mini, Trinity Large Preview)  |
| Yapılandırılmış çıktı (JSON modu ve JSON şeması) | Evet                                     |
| Genişletilmiş düşünme                         | Evet (Trinity Large Thinking; araçlar devre dışı) |

<AccordionGroup>
  <Accordion title="Ortam notu">
    Gateway bir arka plan hizmeti (launchd/systemd) olarak çalışıyorsa `ARCEEAI_API_KEY`
    (veya `OPENROUTER_API_KEY`) değerinin, örneğin `~/.openclaw/.env` içinde
    ya da `env.shellEnv` üzerinden bu işlem tarafından erişilebilir olduğundan emin olun.
  </Accordion>

  <Accordion title="OpenRouter yönlendirmesi">
    OpenRouter aynı `arcee/trinity-large-thinking` OpenClaw model referansını kullanır.
    OpenClaw, bunu standart `arcee-ai/trinity-large-thinking`
    OpenRouter çalışma zamanı kimliğiyle yönlendirir. OpenRouter'a özgü
    yapılandırma ayrıntıları için [OpenRouter sağlayıcı belgelerine](/tr/providers/openrouter) bakın.
  </Accordion>
</AccordionGroup>

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="OpenRouter" href="/tr/providers/openrouter" icon="shuffle">
    Tek bir API anahtarıyla Arcee modellerine ve diğer birçok modele erişin.
  </Card>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
</CardGroup>
