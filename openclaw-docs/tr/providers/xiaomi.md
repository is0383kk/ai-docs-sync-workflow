---
read_when:
    - OpenClaw'da Xiaomi MiMo modellerini kullanmak istiyorsunuz
    - Xiaomi MiMo kimlik doğrulaması veya Token Planı kurulumu gerekiyor
summary: Xiaomi MiMo'nun kullandıkça öde ve Token Planı modellerini OpenClaw ile kullanın
title: Xiaomi MiMo
x-i18n:
    generated_at: "2026-07-26T23:59:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ef79dea8332903c726f076c91b3b458e2d98534d402a412e7c156c06b2912a69
    source_path: providers/xiaomi.md
    workflow: 16
---

Xiaomi MiMo, **MiMo** modellerine yönelik API platformudur. Birlikte gelen `xiaomi`
plugin (`enabledByDefault: true`, kurulum adımı yoktur), iki metin
sağlayıcısının yanı sıra bir konuşma (TTS) sağlayıcısını kaydeder:

- `xiaomi` - kullandıkça öde anahtarları (`sk-...`)
- `xiaomi-token-plan` - bölgesel uç nokta ön ayarlarına sahip Token Plan anahtarları (`tp-...`)

| Özellik          | Değer                                                                                                                                              |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Sağlayıcı kimlikleri | `xiaomi` (kullandıkça öde), `xiaomi-token-plan` (Token Plan)                                                                                         |
| Kimlik doğrulama ortam değişkenleri | `XIAOMI_API_KEY`, `XIAOMI_TOKEN_PLAN_API_KEY`                                                                                                      |
| İlk kurulum bayrakları | `--auth-choice xiaomi-api-key`, `--auth-choice xiaomi-token-plan-cn`, `--auth-choice xiaomi-token-plan-sgp`, `--auth-choice xiaomi-token-plan-ams` |
| Doğrudan CLI bayrakları | `--xiaomi-api-key <key>`, `--xiaomi-token-plan-api-key <key>`                                                                                      |
| API              | OpenAI uyumlu sohbet tamamlamaları (`openai-completions`)                                                                                          |
| Konuşma sözleşmesi | `speechProviders: ["xiaomi"]`                                                                                                                      |
| Temel URL'ler    | Kullandıkça öde: `https://api.xiaomimimo.com/v1`; Token Plan: `token-plan-{cn,sgp,ams}.xiaomimimo.com/v1`                                            |
| Varsayılan modeller | `xiaomi/mimo-v2.5`, `xiaomi-token-plan/mimo-v2.5-pro`                                                                                              |
| TTS varsayılanı  | `mimo-v2.5-tts`, ses `mimo_default`; ses tasarımı modeli `mimo-v2.5-tts-voicedesign`                                                               |

## Başlarken

<Steps>
  <Step title="Doğru anahtarı edinin">
    [Xiaomi MiMo konsolunda](https://platform.xiaomimimo.com/#/console/api-keys) bir kullandıkça öde anahtarı oluşturun veya Token Plan abonelik sayfanızı açıp bölgesel OpenAI uyumlu temel URL'yi ve eşleşen `tp-...` anahtarını kopyalayın.
  </Step>

  <Step title="İlk kurulumu çalıştırın">
    Kullandıkça öde:

    ```bash
    openclaw onboard --auth-choice xiaomi-api-key
    ```

    Token Plan:

    ```bash
    openclaw onboard --auth-choice xiaomi-token-plan-sgp
    ```

    Alternatif olarak anahtarları doğrudan iletin:

    ```bash
    openclaw onboard --auth-choice xiaomi-api-key --xiaomi-api-key "$XIAOMI_API_KEY"
    openclaw onboard --auth-choice xiaomi-token-plan-sgp --xiaomi-token-plan-api-key "$XIAOMI_TOKEN_PLAN_API_KEY"
    ```

  </Step>
  <Step title="Modelin kullanılabilir olduğunu doğrulayın">
    ```bash
    openclaw models list --provider xiaomi
    openclaw models list --provider xiaomi-token-plan
    ```
  </Step>
</Steps>

<Tip>
İlk kurulum, anahtar biçimini doğrular ve kullandıkça öde yoluna bir `tp-...` anahtarı ya da Token Plan yoluna bir `sk-...` anahtarı girildiğinde uyarı verir.
</Tip>

## Kullandıkça öde kataloğu

| Model başvurusu       | Girdi       | Bağlam    | Azami çıktı | Akıl yürütme | Notlar            |
| ---------------------- | ----------- | --------- | ---------- | --------- | ------------- |
| `xiaomi/mimo-v2.5`     | metin, görsel | 1,048,576 | 131,072    | Evet      | Varsayılan model |
| `xiaomi/mimo-v2.5-pro` | metin        | 1,048,576 | 131,072    | Evet      | Amiral gemisi   |

## Token Plan kataloğu

Xiaomi'nin abonelik kullanıcı arayüzünde gösterilen bölgesel temel URL ile eşleşen Token Plan kimlik doğrulama seçeneğini belirleyin:

| Kimlik doğrulama seçeneği | Temel URL                                  |
| ----------------------- | ------------------------------------------ |
| `xiaomi-token-plan-cn`  | `https://token-plan-cn.xiaomimimo.com/v1`  |
| `xiaomi-token-plan-sgp` | `https://token-plan-sgp.xiaomimimo.com/v1` |
| `xiaomi-token-plan-ams` | `https://token-plan-ams.xiaomimimo.com/v1` |

| Model başvurusu                  | Girdi       | Bağlam    | Azami çıktı | Akıl yürütme | Notlar            |
| --------------------------------- | ----------- | --------- | ---------- | --------- | ------------- |
| `xiaomi-token-plan/mimo-v2.5-pro` | metin        | 1,048,576 | 131,072    | Evet      | Varsayılan model |
| `xiaomi-token-plan/mimo-v2.5`     | metin, görsel | 1,048,576 | 131,072    | Evet      | Çok kipli         |

`xiaomi-token-plan` çözümlenmek için bölgesel bir temel URL gerektirir. Desteklenen yol,
birlikte gelen bir Token Plan ilk kurulum seçeneği veya `baseUrl` ayarlanmış açık bir
`models.providers.xiaomi-token-plan` yapılandırma bloğudur; bunlardan
biri olmadan sağlayıcı sunulmaz.

## Akıl yürütme modelleri

`mimo-v2.5` ve `mimo-v2.5-pro`,
OpenClaw'ın [`/think` yönergesini](/tr/tools/thinking) `off`,
`minimal`, `low`, `medium`, `high`, `xhigh` ve `max` düzeyleriyle (varsayılan `high`) destekler.

## Metinden konuşmaya

Birlikte gelen `xiaomi` plugin ayrıca Xiaomi MiMo'yu
`tts` için bir konuşma sağlayıcısı olarak kaydeder. Metni bir `assistant`
iletisi, isteğe bağlı biçem yönlendirmesini ise bir `user`
iletisi olarak kullanıp Xiaomi'nin sohbet tamamlamaları TTS sözleşmesini çağırır.

| Özellik | Değer                                    |
| -------- | ---------------------------------------- |
| TTS kimliği | `xiaomi` (`mimo` diğer adı)                  |
| Kimlik doğrulama | `XIAOMI_API_KEY`                         |
| API      | `audio` ile `POST /v1/chat/completions` |
| Varsayılan | `mimo-v2.5-tts`, ses `mimo_default`    |
| Çıktı    | Varsayılan olarak MP3; yapılandırıldığında WAV |

```json5
{
  tts: {
    auto: "always",
    provider: "xiaomi",
    providers: {
      xiaomi: {
        apiKey: "xiaomi_api_key",
        model: "mimo-v2.5-tts",
        speakerVoice: "mimo_default",
        format: "mp3",
        style: "Bright, natural, conversational tone.",
      },
    },
  },
}
```

Yerleşik sesler: `mimo_default`, `default_zh`, `default_en`, `Mia`, `Chloe`,
`Milo`, `Dean`. Ön ayarlı ses modeli `mimo-v2.5-tts`, `audio.voice` kullandığından
OpenClaw bu model için `speakerVoice` gönderir.

Ses tasarımı modeli `mimo-v2.5-tts-voicedesign`, ön ayarlı bir ses kimliği yerine
doğal dildeki bir biçem isteminden ses üretir. `style` değerini
istenen ses açıklamasına ayarlayın; OpenClaw bunu `user` iletisi olarak, seslendirilecek
metni `assistant` iletisi olarak gönderir ve bu
model için `audio.voice` değerini dahil etmez.

```json5
{
  tts: {
    provider: "xiaomi",
    providers: {
      xiaomi: {
        model: "mimo-v2.5-tts-voicedesign",
        format: "wav",
        style: "Warm, natural female voice with clear pronunciation.",
      },
    },
  },
}
```

Sesli not sentezi hedefi isteyen kanallar (Discord, Feishu,
Matrix, Telegram ve WhatsApp) için OpenClaw, Xiaomi çıktısını teslimattan önce
`ffmpeg` kullanarak 48kHz tek kanallı Opus biçimine dönüştürür.

## Yapılandırma örneği

```json5
{
  env: { XIAOMI_API_KEY: "your-key" },
  agents: { defaults: { model: { primary: "xiaomi/mimo-v2.5" } } },
  models: {
    mode: "merge",
    providers: {
      xiaomi: {
        baseUrl: "https://api.xiaomimimo.com/v1",
        api: "openai-completions",
        apiKey: "XIAOMI_API_KEY",
        models: [
          {
            id: "mimo-v2.5",
            name: "Xiaomi MiMo V2.5",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 1048576,
            maxTokens: 131072,
          },
          {
            id: "mimo-v2.5-pro",
            name: "Xiaomi MiMo V2.5 Pro",
            reasoning: true,
            input: ["text"],
            contextWindow: 1048576,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

Fiyatlandırma ve uyumluluk bayrakları birlikte gelen plugin bildiriminden alındığından, yapılandırma örneği çalışma zamanı davranışından sapmayı önlemek için `cost` ve `compat` değerlerini içermez.

Token Plan:

```json5
{
  env: { XIAOMI_TOKEN_PLAN_API_KEY: "tp-your-key" },
  agents: { defaults: { model: { primary: "xiaomi-token-plan/mimo-v2.5-pro" } } },
  models: {
    mode: "merge",
    providers: {
      "xiaomi-token-plan": {
        baseUrl: "https://token-plan-sgp.xiaomimimo.com/v1",
        api: "openai-completions",
        apiKey: "XIAOMI_TOKEN_PLAN_API_KEY",
        models: [
          {
            id: "mimo-v2.5-pro",
            name: "Xiaomi MiMo V2.5 Pro",
            reasoning: true,
            input: ["text"],
            contextWindow: 1048576,
            maxTokens: 131072,
          },
          {
            id: "mimo-v2.5",
            name: "Xiaomi MiMo V2.5",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 1048576,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

Fiyatlandırma birlikte gelen bildirimden alındığından (Token Plan modelleri kademeli önbellek okuma fiyatlandırması içerir), yapılandırma örneği `cost` değerini içermez.

<AccordionGroup>
  <Accordion title="Otomatik ekleme davranışı">
    Ortamınızda `XIAOMI_API_KEY` ayarlandığında veya bir kimlik doğrulama profili bulunduğunda `xiaomi` sağlayıcısı otomatik olarak etkinleştirilir. `xiaomi-token-plan` bölgesel bir temel URL gerektirdiğinden, desteklenen yol birlikte gelen Token Plan ilk kurulum seçeneği veya açık bir `models.providers.xiaomi-token-plan` yapılandırma bloğudur.
  </Accordion>

  <Accordion title="Model ayrıntıları">
    - **mimo-v2.5** - kullandıkça öde varsayılanı ve Token Plan çok kipli V2.5 rotası.
    - **mimo-v2.5-pro** - amiral gemisi akıl yürütme modeli ve Token Plan varsayılanı.

    <Note>
    Kullandıkça öde modelleri `xiaomi/` ön ekini kullanır. Token Plan modelleri `xiaomi-token-plan/` ön ekini kullanır.
    </Note>

  </Accordion>

  <Accordion title="Sorun giderme">
    - Modeller görünmüyorsa ilgili anahtar ortam değişkeninin veya kimlik doğrulama profilinin mevcut ve geçerli olduğunu doğrulayın.
    - Token Plan için seçilen ilk kurulum bölgesinin abonelik sayfasındaki temel URL ile eşleştiğini ve anahtarın `tp-` ile başladığını doğrulayın.
    - Gateway bir arka plan hizmeti olarak çalıştığında anahtarın bu işlem tarafından erişilebilir olduğundan emin olun (örneğin `~/.openclaw/.env` içinde veya `env.shellEnv` aracılığıyla).

    <Warning>
    Yalnızca etkileşimli kabuğunuzda ayarlanan anahtarlar, arka plan hizmeti tarafından yönetilen Gateway işlemleri tarafından görülemez. Kalıcı erişilebilirlik için `~/.openclaw/.env` veya `env.shellEnv` yapılandırmasını kullanın.
    </Warning>

  </Accordion>
</AccordionGroup>

## İlgili içerik

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıların, model başvurularının ve yük devretme davranışının seçilmesi.
  </Card>
  <Card title="Düşünme düzeyleri" href="/tr/tools/thinking" icon="brain">
    `/think` yönergesi söz dizimi ve düzey eşlemesi.
  </Card>
  <Card title="Yapılandırma başvurusu" href="/tr/gateway/configuration-reference" icon="gear">
    OpenClaw yapılandırmasının eksiksiz başvurusu.
  </Card>
  <Card title="Xiaomi MiMo konsolu" href="https://platform.xiaomimimo.com" icon="arrow-up-right-from-square">
    Xiaomi MiMo panosu ve API anahtarı yönetimi.
  </Card>
</CardGroup>
