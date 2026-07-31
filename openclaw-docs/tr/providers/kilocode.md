---
read_when:
    - Birçok LLM için tek bir API anahtarı istiyorsunuz
    - OpenClaw'da modelleri Kilo Gateway üzerinden çalıştırmak istiyorsunuz
summary: OpenClaw'da birçok modele erişmek için Kilo Gateway'in birleşik API'sini kullanın
title: Kilo Gateway
x-i18n:
    generated_at: "2026-07-27T00:14:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0246a1a77f4265168b213e0167360e1cd89dc2ca864997f08cae5331037f9e89
    source_path: providers/kilocode.md
    workflow: 16
---

Kilo Gateway, istekleri tek bir OpenAI uyumlu uç nokta ve API anahtarı arkasındaki birçok modele yönlendirir.

| Özellik   | Değer                              |
| --------- | ---------------------------------- |
| Sağlayıcı | `kilocode`                 |
| Kimlik doğrulama | `KILOCODE_API_KEY`         |
| API       | OpenAI uyumlu                      |
| Temel URL | `https://api.kilo.ai/api/gateway/`                 |

## Plugini yükleme

```bash
openclaw plugins install @openclaw/kilocode-provider
openclaw gateway restart
```

## Kurulum

<Steps>
  <Step title="Hesap oluşturma">
    [app.kilo.ai](https://app.kilo.ai) adresine gidin, oturum açın veya bir hesap oluşturun, ardından bir API anahtarı oluşturun.
  </Step>
  <Step title="İlk kurulumu çalıştırma">
    ```bash
    openclaw onboard --auth-choice kilocode-api-key
    ```

    Alternatif olarak ortam değişkenini doğrudan ayarlayın:

    ```bash
    export KILOCODE_API_KEY="<your-kilocode-api-key>" # pragma: allowlist secret
    ```

  </Step>
  <Step title="Modelin kullanılabilir olduğunu doğrulama">
    ```bash
    openclaw models list --provider kilocode
    ```
  </Step>
</Steps>

## Varsayılan model ve katalog

Varsayılan model, Kilo Gateway'in dengeli akıllı yönlendirme katmanı olan `kilocode/kilo-auto/balanced` modelidir.
OpenClaw bunun için görevden üst akış modeline eşleme yayımlamaz; `kilo-auto/balanced`
arkasındaki yönlendirme Kilo Gateway tarafından yönetilir.

OpenClaw başlangıçta `GET https://api.kilo.ai/api/gateway/models` sorgusu yapar ve keşfedilen modelleri
statik bir yedek katalogdan önce birleştirir. Statik yedek yalnızca
`kilocode/kilo-auto/balanced` (`Auto Balanced`, `input: ["text", "image"]`, `reasoning: true`,
`contextWindow: 1000000`, `maxTokens: 65536`) içerir.

Gateway üzerindeki herhangi bir modele `kilocode/<upstream-id>` olarak erişilebilir (örneğin
`kilocode/anthropic/claude-sonnet-4`, `kilocode/openai/gpt-5.5`). Keşfedilen listenin tamamını görmek için
`/models kilocode` veya `openclaw models list --provider kilocode` komutunu çalıştırın.

## Yapılandırma örneği

```json5
{
  env: { KILOCODE_API_KEY: "<your-kilocode-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "kilocode/kilo-auto/balanced" },
    },
  },
}
```

## Davranış notları

<AccordionGroup>
  <Accordion title="Aktarım ve uyumluluk">
    Kilo Gateway, OpenRouter ile uyumludur; bu nedenle yerel OpenAI istek biçimlendirmesi yerine
    proxy tarzı OpenAI uyumlu istek yolunu kullanır (`store` yoktur, OpenAI muhakeme eforu yükü yoktur).

    - Gemini destekli Kilo referansları proxy-Gemini yolunda kalır: OpenClaw burada Gemini düşünce
      imzalarını temizler ancak yerel Gemini yeniden oynatma doğrulamasını veya önyükleme yeniden yazımlarını etkinleştirmez.
    - İstekler, API anahtarınızdan oluşturulan bir Bearer belirteci kullanır.

  </Accordion>

  <Accordion title="Akış sarmalayıcısı ve muhakeme">
    Kilo akış sarmalayıcısı, `X-KILOCODE-FEATURE` istek üst bilgisi ekler (varsayılan `openclaw`,
    `KILOCODE_FEATURE` ortam değişkeniyle geçersiz kılınabilir) ve bunu destekleyen modeller için
    muhakeme eforu yüklerini normalleştirir.

    <Warning>
    `kilocode/kilo-auto/balanced` ve `x-ai/*` referansları muhakeme eforu eklemeyi atlar. Muhakeme
    desteğine ihtiyacınız varsa `kilocode/anthropic/claude-sonnet-4` gibi somut bir model referansı kullanın.
    </Warning>

  </Accordion>

  <Accordion title="Sorun giderme">
    - Başlangıçta model keşfi başarısız olursa OpenClaw, `kilocode/kilo-auto/balanced` içeren statik yedek kataloğa döner.
    - API anahtarınızın geçerli olduğunu ve Kilo hesabınızda istenen modellerin etkinleştirildiğini doğrulayın.
    - Gateway bir arka plan hizmeti olarak çalıştığında, `KILOCODE_API_KEY` değerinin bu süreç tarafından kullanılabildiğinden emin olun (örneğin `~/.openclaw/.env` içinde veya `env.shellEnv` aracılığıyla).

  </Accordion>
</AccordionGroup>

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Yapılandırma referansı" href="/tr/gateway/configuration-reference" icon="gear">
    OpenClaw yapılandırmasının tam referansı.
  </Card>
  <Card title="Kilo Gateway" href="https://app.kilo.ai" icon="arrow-up-right-from-square">
    Kilo Gateway panosu, API anahtarları ve hesap yönetimi.
  </Card>
</CardGroup>
