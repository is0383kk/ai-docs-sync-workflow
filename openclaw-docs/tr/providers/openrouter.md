---
read_when:
    - Birçok LLM için tek bir API anahtarı istiyorsunuz
    - OpenClaw'da modelleri OpenRouter üzerinden çalıştırmak istiyorsunuz
    - Görsel oluşturmak için OpenRouter kullanmak istiyorsunuz
    - Müzik üretimi için OpenRouter kullanmak istiyorsunuz
    - Video oluşturmak için OpenRouter kullanmak istiyorsunuz
summary: OpenClaw'da birçok modele erişmek için OpenRouter'ın birleşik API'sini kullanın
title: OpenRouter
x-i18n:
    generated_at: "2026-07-26T22:59:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0936a10222f44f376dee081b7ee0678cddc3bc4579ac0006321dc1012d59bcf
    source_path: providers/openrouter.md
    workflow: 16
---

OpenRouter, istekleri tek bir API ve tek bir anahtarın arkasındaki birçok modele yönlendirir.
OpenAI uyumlu olduğundan OpenClaw, diğer proxy sağlayıcıları için kullanılan aynı
`openai-completions` tarzı aktarım üzerinden onunla iletişim kurar.

## Başlarken

<Tabs>
  <Tab title="OAuth">
    <Steps>
      <Step title="OAuth ilk katılımını çalıştırın">
        ```bash
        openclaw onboard --auth-choice openrouter-oauth
        ```

        OpenClaw, OpenRouter'ın tarayıcıda oturum açma akışını (PKCE) açar, kodu
        bir OpenRouter API anahtarıyla değiştirir ve varsayılan
        OpenRouter kimlik doğrulama profiline kaydeder. Uzak/başsız ana makinelerde OpenClaw,
        oturum açma URL'sini yazdırır ve oturum açtıktan sonra yönlendirme URL'sini yapıştırmanızı ister.
      </Step>
      <Step title="(İsteğe bağlı) Belirli bir modele geçin">
        İlk katılım varsayılan olarak `openrouter/auto` kullanır. Daha sonra belirli bir model seçin:

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
  <Tab title="API anahtarı">
    <Steps>
      <Step title="API anahtarınızı alın">
        [openrouter.ai/keys](https://openrouter.ai/keys) adresinde bir API anahtarı oluşturun.
      </Step>
      <Step title="API anahtarıyla ilk katılımı çalıştırın">
        ```bash
        openclaw onboard --auth-choice openrouter-api-key
        ```
      </Step>
      <Step title="(İsteğe bağlı) Belirli bir modele geçin">
        İlk katılım varsayılan olarak `openrouter/auto` kullanır. Daha sonra belirli bir model seçin:

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## Yapılandırma örneği

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/auto" },
    },
  },
}
```

## Model referansları

<Note>
Model referansları `openrouter/<provider>/<model>` kalıbını izler. Kullanılabilir sağlayıcıların ve
modellerin tam listesi için [/concepts/model-providers](/tr/concepts/model-providers) sayfasına bakın.
</Note>

Canlı katalog keşfi kullanılamadığında kullanılan paketle birlikte gelen yedek modeller:

| Model referansı                   | Notlar                       |
| --------------------------------- | ---------------------------- |
| `openrouter/auto`                 | OpenRouter otomatik yönlendirmesi |
| `openrouter/moonshotai/kimi-k2.6` | MoonshotAI üzerinden Kimi K2.6     |
| `openrouter/moonshotai/kimi-k2.5` | MoonshotAI üzerinden Kimi K2.5     |

`openrouter/openrouter/fusion` ([Fusion yönlendiricisi](#fusion-router) bölümüne bakın) dahil diğer tüm
`openrouter/<provider>/<model>` referansları, OpenRouter'ın canlı model kataloğuna göre
dinamik olarak çözümlenir.

## Görüntü oluşturma

OpenRouter, `image_generate` aracını destekleyebilir. `agents.defaults.mediaModels.image` altında
bir OpenRouter görüntü modeli ayarlayın:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openrouter/google/gemini-3.1-flash-image-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

OpenClaw, görüntü isteklerini `modalities: ["image", "text"]` ile OpenRouter'ın sohbet tamamlama
görüntü API'sine gönderir. Gemini görüntü modelleri ayrıca OpenRouter'ın
`image_config` üzerinden `aspectRatio` ve `resolution` ipuçlarını alır; diğer
görüntü modelleri almaz. Daha yavaş modeller için `agents.defaults.mediaModels.image.timeoutMs` kullanın;
`image_generate` aracının çağrı başına `timeoutMs` değeri yine önceliklidir.

## Video oluşturma

OpenRouter, eşzamansız `/videos` API'si üzerinden
`video_generate` aracını destekleyebilir. `agents.defaults.mediaModels.video` altında
bir OpenRouter video modeli ayarlayın:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openrouter/google/veo-3.1-fast",
      },
    },
  },
}
```

OpenClaw, metinden videoya ve görüntüden videoya işleri gönderir, döndürülen
`polling_url` değerini yoklar ve tamamlanan videoyu OpenRouter'ın
`unsigned_urls` uç noktasından veya iş içeriği uç noktasından indirir. Referans görüntüler varsayılan olarak
ilk/son kare görüntüleri şeklinde kullanılır; `reference_image` olarak etiketlenen görüntüler bunun yerine girdi
referansları olarak gönderilir. Paketle birlikte gelen `google/veo-3.1-fast` varsayılanı, 4/6/8
saniyelik süreleri, `720P`/`1080P` çözünürlüklerini ve `16:9`/`9:16` en-boy oranlarını destekler.
Videodan videoya dönüştürme desteklenmez: üst kaynak API yalnızca metin ve görüntü
referanslarını kabul eder.

## Müzik oluşturma

OpenRouter, sohbet tamamlama ses çıkışı üzerinden `music_generate` aracını
destekleyebilir. `agents.defaults.mediaModels.music` altında
bir OpenRouter ses modeli ayarlayın:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "openrouter/google/lyria-3-pro-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

Paketle birlikte gelen OpenRouter müzik sağlayıcısı varsayılan olarak `google/lyria-3-pro-preview`
kullanır ve ayrıca `google/lyria-3-clip-preview` seçeneğini sunar. OpenClaw, `modalities:
["text", "audio"]` gönderir, yanıtı akış halinde alır, ses parçalarını toplar ve
sonucu kanala teslim edilmek üzere oluşturulan medya olarak kaydeder. Lyria modelleri, paylaşılan
`music_generate image=...` parametresi üzerinden tek bir referans görüntü kabul eder.
Ses akışı, transkript saklama ve türetilmiş SSE olay zarfı
`agents.defaults.mediaMaxMb` ile sınırlandırılır (varsayılan ses sınırı 16 MB'dir).

## Metinden konuşmaya

OpenRouter, OpenAI uyumlu `/audio/speech` uç noktası üzerinden bir TTS sağlayıcısı olarak çalışabilir.

```json5
{
  tts: {
    auto: "always",
    provider: "openrouter",
    providers: {
      openrouter: {
        model: "hexgrad/kokoro-82m",
        speakerVoice: "af_alloy",
        responseFormat: "mp3",
      },
    },
  },
}
```

`tts.providers.openrouter.apiKey` belirtilmezse TTS sırasıyla
`models.providers.openrouter.apiKey` ve ardından `OPENROUTER_API_KEY` seçeneklerine geri döner.

## Konuşmadan metne dönüştürme (gelen ses)

OpenRouter, paylaşılan `tools.media.audio` yolu üzerinden STT uç noktasını
(`/audio/transcriptions`) kullanarak gelen sesli mesajların/ses dosyalarının yazıya
dökümünü yapabilir. Bu, gelen sesli mesajları/ses dosyalarını medya anlama ön
kontrolüne ileten tüm kanal pluginleri için geçerlidir.

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "openrouter", model: "openai/whisper-large-v3-turbo" }],
      },
    },
  },
}
```

OpenClaw, OpenRouter STT isteklerini çok parçalı OpenAI form yüklemeleri olarak
değil, `input_audio` altında base64 ses içeren JSON biçiminde
(OpenRouter'ın STT sözleşmesi) gönderir.

## Fusion yönlendiricisi

OpenRouter Fusion, bir OpenClaw model başvurusunu paralel olarak birden fazla
OpenRouter modeline gönderir, yanıtları OpenRouter'a değerlendirtir ve normal
OpenRouter uç noktası üzerinden tek bir nihai yanıt döndürür. Üst model slug'ı
`openrouter/fusion` olduğundan OpenClaw model başvurusu hem OpenClaw sağlayıcı
önekini hem de üst OpenRouter ad alanını taşır:

```bash
openclaw models set openrouter/openrouter/fusion
```

Fusion'ın panelini ve değerlendiricisini modelin `params.extraBody` alanı
üzerinden yapılandırın; bu alanlar doğrudan OpenRouter sohbet tamamlama isteği
gövdesine iletilir. Fusion, hem OAuth hem de API anahtarıyla ilk kurulumla
çalışır; OAuth kullanıyorsanız aşağıdaki `env.OPENROUTER_API_KEY` satırını atlayın.

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/openrouter/fusion" },
      models: {
        "openrouter/openrouter/fusion": {
          params: {
            extraBody: {
              plugins: [
                {
                  id: "fusion",
                  analysis_models: [
                    "google/gemini-3.5-flash",
                    "moonshotai/kimi-k2.6",
                    "deepseek/deepseek-v4-pro",
                  ],
                  model: "google/gemini-3.5-flash",
                },
              ],
            },
          },
        },
      },
    },
  },
}
```

`analysis_models` paralel paneldir; Fusion plugin yapılandırmasındaki
`model` ise değerlendirici modeldir. Normal ajan/sohbet
etkileşimlerinde Fusion'ı zorlamak amacıyla üst düzey `tool_choice`
değerini `"required"` olarak ayarlamayın: OpenClaw etkileşimleri kendi
araç tanımlarını içerebilir ve üst düzey zorunlu araç seçimi, Fusion
yönlendiricisi yerine bunlardan birini seçebilir. Bu Fusion plugin
yapılandırması mevcut olduğunda OpenClaw, yapılandırılmış analiz modellerini ve
değerlendirici modeli listeleyen, temizlenmiş bir sistem istemi notu ekler;
böylece ajan kendi Fusion paneli hakkındaki soruları yanıtlayabilir. Diğer
`extraBody` alanları isteme kopyalanmaz.

Fusion tasarım gereği daha yavaştır: OpenRouter istemi birden fazla analiz
modeline dağıtır ve ardından bir değerlendirme/sentez adımı çalıştırır; bu
nedenle gecikme, doğrudan tek modelli bir istekten daha yüksektir. Gecikmeye
duyarlı bir varsayılan olarak değil, özenli ve yüksek kaliteli yanıtlar ya da
yükseltme yolları için kullanın. Daha hızlı yanıtlar için paneli küçük tutun ve
daha hızlı analiz/değerlendirici modeller seçin.

Yapılandırılmış bir başvuruyu tek seferlik yerel çağrıyla test edin:

```bash
openclaw infer model run --local \
  --model openrouter/openrouter/fusion \
  --prompt "Tam olarak şununla yanıt ver: FUSION_OK" \
  --json
```

## Kimlik doğrulama ve üstbilgiler

OpenRouter, API anahtarınızdaki bir Bearer belirteci kullanır. OpenRouter OAuth, bir OpenRouter API anahtarı veren bir PKCE oturum açma akışıdır; bu nedenle OpenClaw, sonucu manuel API anahtarı kurulumunda kullanılan aynı `openrouter:default` API anahtarı kimlik doğrulama profilinde saklar.

Tam ilk katılım sürecini yeniden çalıştırmadan mevcut bir kurulumda oturum açmak veya saklanan anahtarı değiştirmek için:

```bash
openclaw models auth login --provider openrouter --method oauth
openclaw models auth login --provider openrouter --method api-key
```

Doğrulanmış OpenRouter isteklerinde (`https://openrouter.ai/api/v1`) OpenClaw, OpenRouter'ın belgelenmiş uygulama ilişkilendirme üstbilgilerini ekler:

| Üstbilgi                  | Değer                                                                                                  |
| ------------------------- | ------------------------------------------------------------------------------------------------------ |
| `HTTP-Referer`        | `https://openclaw.ai`                                                                                     |
| `X-OpenRouter-Title`        | `OpenClaw`                                                                                     |
| `X-OpenRouter-Categories`        | `cli-agent,cloud-agent,programming-app,creative-writing,writing-assistant,general-chat,personal-agent`                                                                                     |

<Warning>
OpenRouter sağlayıcısını başka bir proxy'ye veya temel URL'ye yönlendirirseniz OpenClaw, OpenRouter'a özgü bu üstbilgileri veya Anthropic önbellek işaretleyicilerini eklemez.
</Warning>

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Yanıt önbelleğe alma">
    OpenRouter yanıt önbelleğe alma özelliği isteğe bağlıdır. Bunu model başına etkinleştirin:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/auto": {
              params: {
                responseCache: true,
                responseCacheTtlSeconds: 300,
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw, `X-OpenRouter-Cache: true` ve yapılandırıldığında `X-OpenRouter-Cache-TTL` gönderir. `responseCacheClear: true`, geçerli istek için yenilemeyi zorlar ve yerine geçen yanıtı saklar. Alt çizgili adlandırma diğer adları (`response_cache`, `response_cache_ttl_seconds`, `response_cache_clear`) ile `Seconds` son eki olmadan `responseCacheTtl` / `response_cache_ttl` kabul edilir.

    Bu, sağlayıcı istem önbelleğe alma özelliğinden ve OpenRouter'ın Anthropic `cache_control` işaretleyicilerinden ayrıdır. Özel proxy temel URL'lerinde değil, yalnızca doğrulanmış `openrouter.ai` rotalarında geçerlidir.

  </Accordion>

  <Accordion title="Anthropic önbellek işaretleyicileri">
    Doğrulanmış OpenRouter rotalarında Anthropic model referansları, sistem/geliştirici istem bloklarında istem önbelleğinin daha iyi yeniden kullanılması için OpenRouter'ın Anthropic `cache_control` işaretleyicilerini korur.
  </Accordion>

  <Accordion title="Anthropic akıl yürütme ön dolgusu">
    Doğrulanmış OpenRouter rotalarında, akıl yürütmenin etkin olduğu Anthropic model referansları, istek OpenRouter'a ulaşmadan önce sondaki asistan ön dolgu turlarını kaldırır. Bu, Anthropic'in akıl yürütme konuşmalarının bir kullanıcı turuyla sona ermesi gereksinimine uygundur.
  </Accordion>

  <Accordion title="Düşünme / akıl yürütme ekleme">
    Desteklenen `auto` dışı rotalarda OpenClaw, seçilen düşünme düzeyini
    OpenRouter proxy akıl yürütme yüklerine eşler. `openrouter/auto` ve desteklenmeyen
    model ipuçlarında bu ekleme atlanır. Güncelliğini yitirmiş `openrouter/hunter-alpha` referanslarında da
    bu işlem atlanır; çünkü OpenRouter, kullanımdan kaldırılan bu rotada akıl yürütme
    alanlarında nihai yanıt metni döndürebilir.
  </Accordion>

  <Accordion title="DeepSeek V4 akıl yürütmesini yeniden oynatma">
    Doğrulanmış OpenRouter rotalarında `openrouter/deepseek/deepseek-v4-flash` ve
    `openrouter/deepseek/deepseek-v4-pro`, yeniden oynatılan asistan dönüşlerinde eksik
    `reasoning_content` alanlarını doldurarak düşünme/araç konuşmalarını DeepSeek
    V4'ün gerektirdiği takip biçiminde tutar. OpenClaw, bu rotalar için OpenRouter tarafından desteklenen
    `reasoning.effort` değerlerini gönderir: `xhigh`/`max`, `xhigh` değerine eşlenir;
    kapalı dışındaki diğer tüm düzeyler `high` değerine eşlenir.
  </Accordion>

  <Accordion title="Yalnızca OpenAI'a özel istek biçimlendirme">
    OpenRouter, proxy tarzı OpenAI uyumlu yol üzerinden çalışır; bu nedenle
    `serviceTier`, Responses `store`, OpenAI akıl yürütme uyumluluğu yükleri
    ve istem önbelleği ipuçları gibi yalnızca yerel OpenAI'a özgü istek biçimlendirmeleri iletilmez.
  </Accordion>

  <Accordion title="Gemini destekli rotalar">
    Gemini destekli OpenRouter referansları proxy-Gemini yolunda kalır: OpenClaw,
    Gemini düşünce imzası temizliğini burada sürdürür ancak yerel
    Gemini yeniden oynatma doğrulamasını veya başlangıç yeniden yazımlarını etkinleştirmez.
  </Accordion>

  <Accordion title="Sağlayıcı yönlendirme meta verileri">
    OpenRouter, temel sağlayıcı yönlendirmesi için bir `provider` istek nesnesini
    destekler. Tüm OpenRouter metin modeli istekleri için varsayılan bir politikayı
    `models.providers.openrouter.params.provider` ile yapılandırın:

    ```json5
    {
      models: {
        providers: {
          openrouter: {
            params: {
              provider: {
                sort: "latency",
                require_parameters: true,
                data_collection: "deny",
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw bu nesneyi, istek `provider` yükü olarak OpenRouter'a
    iletir. `sort`, `only`, `ignore`, `order`,
    `allow_fallbacks`, `require_parameters`, `data_collection`, `quantizations`,
    `max_price`, `preferred_max_latency`, `preferred_min_throughput`, `zdr` ve
    `enforce_distillable_text` dâhil olmak üzere OpenRouter'ın belgelenmiş snake_case alanlarını kullanın.

    Model başına parametreler, sağlayıcı genelindeki yönlendirme nesnesini geçersiz kılar:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/anthropic/claude-sonnet-4-6": {
              params: {
                provider: {
                  order: ["anthropic"],
                  allow_fallbacks: false,
                },
              },
            },
          },
        },
      },
    }
    ```

    Bu yalnızca OpenRouter sohbet tamamlama rotalarında geçerlidir. Doğrudan Anthropic,
    Google, OpenAI veya özel sağlayıcı rotaları OpenRouter yönlendirme parametrelerini yok sayar.

  </Accordion>
</AccordionGroup>

## İlgili

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıların, model referanslarının ve yük devretme davranışının seçilmesi.
  </Card>
  <Card title="Yapılandırma referansı" href="/tr/gateway/configuration-reference" icon="gear">
    Aracılar, modeller ve sağlayıcılar için tam yapılandırma referansı.
  </Card>
</CardGroup>
