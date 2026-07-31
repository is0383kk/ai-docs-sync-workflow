---
read_when:
    - Chutes'u OpenClaw ile kullanmak istiyorsunuz
    - OAuth veya API anahtarı kurulum yoluna ihtiyacınız var
    - Varsayılan modeli, diğer adları veya keşif davranışını istiyorsunuz
summary: Chutes kurulumu (OAuth veya API anahtarı, model keşfi, takma adlar)
title: Chutes
x-i18n:
    generated_at: "2026-07-26T23:32:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 57ea5112105f19028c1a348b4d7fec4cf7ef12de00b1b2de9c152057bf5033a9
    source_path: providers/chutes.md
    workflow: 16
---

[Chutes](https://chutes.ai), açık kaynaklı model kataloglarını
OpenAI uyumlu bir API üzerinden sunar. OpenClaw hem tarayıcı OAuth'unu hem de API anahtarıyla kimlik doğrulamayı destekler.

| Özellik                  | Değer                                                   |
| ------------------------ | ------------------------------------------------------- |
| Sağlayıcı                | `chutes`                                      |
| Plugin                   | resmî harici paket (`@openclaw/chutes-provider`)                 |
| API                      | OpenAI uyumlu                                           |
| Temel URL                | `https://llm.chutes.ai/v1`                                      |
| Kimlik doğrulama         | OAuth veya API anahtarı (aşağıya bakın)                 |
| Çalışma zamanı ortam değişkenleri | `CHUTES_API_KEY`, `CHUTES_OAUTH_TOKEN`         |

`CHUTES_OAUTH_TOKEN`, önceden alınmış bir OAuth erişim belirtecini doğrudan sağlar
(örneğin CI ortamında) ve aşağıdaki etkileşimli tarayıcı akışını atlar.

## Plugin'i yükleme

```bash
openclaw plugins install @openclaw/chutes-provider
openclaw gateway restart
```

## Başlarken

Her iki yol da varsayılan modeli `chutes/zai-org/GLM-5-TEE` olarak ayarlar ve
Chutes kataloğunu kaydeder.

<Tabs>
  <Tab title="OAuth">
    <Steps>
      <Step title="OAuth ilk katılım akışını çalıştırma">
        ```bash
        openclaw onboard --auth-choice chutes
        ```
        OpenClaw, tarayıcı akışını yerel olarak başlatır veya uzak/ekransız
        ana makinelerde bir URL ve yönlendirme adresi yapıştırma akışı gösterir. OAuth belirteçleri,
        OpenClaw kimlik doğrulama profilleri aracılığıyla otomatik olarak yenilenir.
      </Step>
    </Steps>
  </Tab>
  <Tab title="API anahtarı">
    <Steps>
      <Step title="API anahtarı alma">
        [chutes.ai/settings/api-keys](https://chutes.ai/settings/api-keys)
        adresinde bir anahtar oluşturun.
      </Step>
      <Step title="API anahtarı ilk katılım akışını çalıştırma">
        ```bash
        openclaw onboard --auth-choice chutes-api-key
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

## Keşif davranışı

Chutes kimlik doğrulaması kullanılabildiğinde OpenClaw, bu kimlik bilgisiyle
`GET /v1/models` adresini sorgular ve keşfedilen modelleri kullanır; bunlar
kimlik bilgisi başına 5 dakika önbelleğe alınır. Süresi dolmuş/yetkisiz bir anahtarda
(HTTP 401) OpenClaw, kimlik bilgileri olmadan bir kez daha dener. Keşif yine de
hiç satır döndürmezse, başarısız olursa veya 2xx dışındaki başka bir durum kodu
döndürürse paketle birlikte gelen statik kataloğa geri döner (hem API anahtarı
hem de OAuth keşfi aynı yolu kullanır). Keşif başlangıçta başarısız olursa
statik katalog otomatik olarak kullanılır.

## Varsayılan takma adlar

OpenClaw, Chutes kataloğu için iki kullanışlı takma ad kaydeder:

| Takma ad              | Hedef model                            |
| --------------------- | -------------------------------------- |
| `chutes-pro`    | `chutes/deepseek-ai/DeepSeek-V3.2-TEE`                     |
| `chutes-vision`    | `chutes/moonshotai/Kimi-K2.5-TEE`                     |

## Yerleşik başlangıç kataloğu

Paketle birlikte gelen yedek katalog, şu anda sunulan bu beş modeli içerir:

| Model referansı                       |
| ------------------------------------- |
| `chutes/zai-org/GLM-5-TEE`                    |
| `chutes/deepseek-ai/DeepSeek-V3.2-TEE`                    |
| `chutes/moonshotai/Kimi-K2.5-TEE`                    |
| `chutes/MiniMaxAI/MiniMax-M2.5-TEE`                    |
| `chutes/Qwen/Qwen3.5-397B-A17B-TEE`                    |

Tam liste için `openclaw models list --all --provider chutes` komutunu çalıştırın.

## Yapılandırma örneği

```json5
{
  agents: {
    defaults: {
      model: { primary: "chutes/zai-org/GLM-5-TEE" },
      models: {
        "chutes/zai-org/GLM-5-TEE": { alias: "Chutes GLM 5" },
        "chutes/deepseek-ai/DeepSeek-V3.2-TEE": { alias: "Chutes DeepSeek V3.2" },
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="OAuth geçersiz kılmaları">
    OAuth akışını isteğe bağlı ortam değişkenleriyle özelleştirin:

    | Değişken | Amaç |
    | -------- | ---- |
    | `CHUTES_CLIENT_ID` | OAuth istemci kimliği (ayarlanmamışsa sorulur) |
    | `CHUTES_CLIENT_SECRET` | OAuth istemci sırrı |
    | `CHUTES_OAUTH_REDIRECT_URI` | Yönlendirme URI'si (varsayılan `http://127.0.0.1:1456/oauth-callback`) |
    | `CHUTES_OAUTH_SCOPES` | Boşlukla ayrılmış kapsamlar (varsayılan `openid profile chutes:invoke`) |

    Yönlendirme uygulaması gereksinimleri ve yardım için
    [Chutes OAuth belgelerine](https://chutes.ai/docs/sign-in-with-chutes/overview) bakın.

  </Accordion>

  <Accordion title="Notlar">
    - Chutes modelleri `chutes/<model-id>` olarak kaydedilir.
    - Chutes, akış sırasında belirteç kullanımını bildirmez (`supportsUsageInStreaming: false`); kullanım toplamları akış tamamlandığında yine gösterilir.

  </Accordion>
</AccordionGroup>

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcı kuralları, model referansları ve yük devretme davranışı.
  </Card>
  <Card title="Yapılandırma referansı" href="/tr/gateway/configuration-reference" icon="gear">
    Sağlayıcı ayarlarını içeren tam yapılandırma şeması.
  </Card>
  <Card title="Chutes" href="https://chutes.ai" icon="arrow-up-right-from-square">
    Chutes kontrol paneli ve API belgeleri.
  </Card>
  <Card title="Chutes API anahtarları" href="https://chutes.ai/settings/api-keys" icon="key">
    Chutes API anahtarlarını oluşturun ve yönetin.
  </Card>
</CardGroup>
