---
read_when:
    - OpenClaw ile Google Gemini modellerini kullanmak istiyorsunuz
    - API anahtarına veya OAuth kimlik doğrulama akışına ihtiyacınız var
summary: Google Gemini kurulumu (API anahtarı + OAuth, görüntü oluşturma, medya anlama, TTS, web araması)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-07-27T00:13:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fdf8db70bcebd425238e5f02ca12bdbcd75fa1c03d285ea127d4e3863892b3aa
    source_path: providers/google.md
    workflow: 16
---

Google plugini, Google AI Studio üzerinden Gemini modellerine erişimin yanı sıra görüntü oluşturma, medya anlama (görüntü/ses/video), metinden konuşmaya ve Gemini Grounding aracılığıyla web araması sağlar.

- Sağlayıcı: `google`
- Kimlik doğrulama: `GEMINI_API_KEY` veya `GOOGLE_API_KEY`
- API: Google Gemini API
- Çalışma zamanı seçeneği: `agentRuntime.id: "google-gemini-cli"`, model referanslarını `google/*` biçiminde standart tutarken Gemini CLI OAuth'u yeniden kullanır.

## Başlarken

Tercih edilen kimlik doğrulama yöntemini seçin ve kurulum adımlarını izleyin.

<Tabs>
  <Tab title="API anahtarı">
    **En uygun kullanım:** Google AI Studio üzerinden standart Gemini API erişimi.

    <Steps>
      <Step title="Bir API anahtarı alın">
        [Google AI Studio](https://aistudio.google.com/apikey) üzerinden ücretsiz bir anahtar oluşturun.
      </Step>
      <Step title="İlk katılımı çalıştırın">
        ```bash
        openclaw onboard --auth-choice gemini-api-key
        ```

        Alternatif olarak anahtarı doğrudan iletin:

        ```bash
        openclaw onboard --non-interactive \
          --mode local \
          --auth-choice gemini-api-key \
          --gemini-api-key "$GEMINI_API_KEY"
        ```
      </Step>
      <Step title="Varsayılan bir model ayarlayın">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "google/gemini-3.1-pro-preview" },
            },
          },
        }
        ```
      </Step>
      <Step title="Modelin kullanılabilir olduğunu doğrulayın">
        ```bash
        openclaw models list --provider google
        ```
      </Step>
    </Steps>

    <Tip>
    `GEMINI_API_KEY` ve `GOOGLE_API_KEY` değerlerinin ikisi de kabul edilir. Önceden yapılandırılmış olanı kullanın.
    </Tip>

    Bir API anahtarı yapılandırıldığında OpenClaw, Google AI Studio'nun metin modeli
    kataloğunu Gemini `models.list` API'sinden yeniler. Böylece yeni yayımlanan Gemini 3 Pro, Flash
    ve Flash-Lite varyantları bir OpenClaw sürümünü
    beklemeden `openclaw models list --provider google` içinde görünür. Keşif kullanılamıyorsa OpenClaw paketle birlikte gelen
    yedek kataloğu korur.

  </Tab>

  <Tab title="Gemini CLI (OAuth)">
    **En uygun kullanım:** ayrı bir API anahtarı kullanmak yerine Gemini CLI OAuth aracılığıyla Google hesabıyla oturum açma.

    <Warning>
    `google-gemini-cli` sağlayıcısı resmî olmayan bir entegrasyondur. Bazı kullanıcılar
    OAuth'u bu şekilde kullandıklarında hesap kısıtlamalarıyla karşılaştıklarını
    bildirmektedir. Riski size ait olmak üzere kullanın.
    </Warning>

    <Steps>
      <Step title="Gemini CLI'yi yükleyin">
        Yerel `gemini` komutu `PATH` üzerinde kullanılabilir olmalıdır.

        ```bash
        # Homebrew
        brew install gemini-cli

        # veya npm
        npm install -g @google/gemini-cli
        ```

        OpenClaw, yaygın Windows/npm düzenleri de dâhil olmak üzere hem Homebrew
        yüklemelerini hem de genel npm yüklemelerini destekler.
      </Step>
      <Step title="OAuth ile oturum açın">
        ```bash
        openclaw models auth login --provider google-gemini-cli --set-default
        ```
      </Step>
      <Step title="Modelin kullanılabilir olduğunu doğrulayın">
        ```bash
        openclaw models list --provider google
        ```
      </Step>
    </Steps>

    - Varsayılan model: `google/gemini-3.1-pro-preview`
    - Çalışma zamanı: `google-gemini-cli`
    - Takma ad: `gemini-cli`

    Gemini 3.1 Pro'nun Gemini API model kimliği `gemini-3.1-pro-preview` değeridir. OpenClaw, kolaylık sağlayan bir takma ad olarak daha kısa `google/gemini-3.1-pro` değerini kabul eder ve sağlayıcı çağrılarından önce bunu normalleştirir.

    **Ortam değişkenleri:**

    - `OPENCLAW_GEMINI_OAUTH_CLIENT_ID` / `GEMINI_CLI_OAUTH_CLIENT_ID`
    - `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET` / `GEMINI_CLI_OAUTH_CLIENT_SECRET`

    <Note>
    Gemini CLI OAuth istekleri oturum açtıktan sonra başarısız olursa Gateway ana makinesinde
    `GOOGLE_CLOUD_PROJECT` veya `GOOGLE_CLOUD_PROJECT_ID` değerini ayarlayıp yeniden deneyin.
    </Note>

    <Note>
    Oturum açma, tarayıcı akışı başlamadan önce başarısız olursa yerel `gemini`
    komutunun yüklü ve `PATH` üzerinde olduğundan emin olun.
    </Note>

    İlk katılımın otomatik algılama özelliği, mevcut bir Gemini CLI oturumunu listeler ancak
    Gemini CLI araçsız bir yoklama sağlamadığından bunu hiçbir zaman otomatik olarak
    test etmez. Devam etmek için Gemini CLI OAuth'u veya bir Gemini API anahtarını seçin.

    `google-gemini-cli/*` model referansları eski uyumluluk takma adlarıdır. Yeni
    yapılandırmalar, yerel Gemini CLI yürütmesi istediklerinde `google-gemini-cli`
    çalışma zamanıyla birlikte `google/*` model referanslarını kullanmalıdır.

  </Tab>
</Tabs>

<Note>
`google/gemini-3-pro-preview`, 2026-03-09 tarihinde kullanımdan kaldırıldı; bunun yerine `google/gemini-3.1-pro-preview` kullanın. Gemini API anahtarı kurulumunu yeniden çalıştırmak (`openclaw onboard --auth-choice gemini-api-key` veya `openclaw models auth login --provider google`), eski bir yapılandırılmış varsayılanı güncel modele yeniden yazar.
</Note>

## Yetenekler

| Yetenek                  | Destekleniyor                  |
| ------------------------ | ------------------------------ |
| Sohbet tamamlamaları     | Evet                           |
| Görüntü oluşturma        | Evet                           |
| Müzik oluşturma          | Evet                           |
| Metinden konuşmaya       | Evet                           |
| Gerçek zamanlı ses       | Evet (Google Live API)         |
| Görüntü anlama           | Evet                           |
| Ses transkripsiyonu      | Evet                           |
| Video anlama             | Evet                           |
| Web araması (Grounding)  | Evet                           |
| Düşünme/akıl yürütme     | Evet (Gemini 2.5+ / Gemini 3+) |
| Gemma 4 modelleri        | Evet                           |

## Web araması

Paketle birlikte gelen `gemini` web araması sağlayıcısı, Gemini Google Search grounding özelliğini kullanır.
`plugins.entries.google.config.webSearch` altında özel bir arama anahtarı yapılandırın
veya `GEMINI_API_KEY` sonrasında `models.providers.google.apiKey` değerini yeniden kullanmasına izin verin:

```json5
{
  plugins: {
    entries: {
      google: {
        config: {
          webSearch: {
            apiKey: "AIza...", // GEMINI_API_KEY veya models.providers.google.apiKey ayarlanmışsa isteğe bağlıdır
            baseUrl: "https://generativelanguage.googleapis.com/v1beta", // models.providers.google.baseUrl değerine geri döner
            model: "gemini-2.5-flash",
          },
        },
      },
    },
  },
}
```

Kimlik bilgisi önceliği sırasıyla özel `webSearch.apiKey`, ardından `GEMINI_API_KEY`
ve son olarak `models.providers.google.apiKey` biçimindedir. `webSearch.baseUrl` isteğe bağlıdır ve
operatör proxy'leri veya uyumlu Gemini API uç noktaları için bulunur; belirtilmediğinde
Gemini web araması `models.providers.google.baseUrl` değerini yeniden kullanır. Sağlayıcıya özgü araç davranışı için
[Gemini araması](/tr/tools/gemini-search) bölümüne bakın.

<Tip>
Gemini 3 modelleri `thinkingBudget` yerine `thinkingLevel` kullanır. OpenClaw,
Gemini 3, Gemini 3.1 ve `gemini-*-latest` takma adı akıl yürütme denetimlerini
`thinkingLevel` değerine eşler; böylece varsayılan/düşük gecikmeli çalıştırmalar devre dışı
`thinkingBudget` değerleri göndermez.

`/think adaptive`, sabit bir OpenClaw düzeyi seçmek yerine Google'ın dinamik düşünme
anlamını korur. Gemini 3 ve Gemini 3.1, düzeyi Google'ın
seçebilmesi için sabit bir `thinkingLevel` değerini atlar; Gemini 2.5 ise Google'ın dinamik belirteci
`thinkingBudget: -1` değerini gönderir.

Gemma 4 modelleri (örneğin `gemma-4-26b-a4b-it`) düşünme modunu destekler. OpenClaw,
Gemma 4 için `thinkingBudget` değerini desteklenen bir Google `thinkingLevel` değerine
yeniden yazar. Düşünmeyi `off` olarak ayarlamak, `MINIMAL`
değerine eşlemek yerine düşünmenin devre dışı kalmasını sağlar.

Gemini 2.5 Pro yalnızca düşünme modunda çalışır ve açık bir
`thinkingBudget: 0` değerini reddeder; OpenClaw, bunu göndermek yerine Gemini 2.5 Pro
isteklerinden bu değeri kaldırır.
</Tip>

## Görüntü oluşturma

Paketle birlikte gelen `google` görüntü oluşturma sağlayıcısı varsayılan olarak
`google/gemini-3.1-flash-image` kullanır.

- Ayrıca desteklenir: `google/gemini-3-pro-image`
- Oluşturma: istek başına en fazla 4 görüntü
- Düzenleme modu: etkin, en fazla 5 giriş görüntüsü
- Geometri denetimleri: `size`, `aspectRatio` ve `resolution`

Google'ı varsayılan görüntü sağlayıcısı olarak kullanmak için:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image",
      },
    },
  },
}
```

<Note>
Paylaşılan araç parametreleri, sağlayıcı seçimi ve yük devretme davranışı için [Görüntü Oluşturma](/tr/tools/image-generation) bölümüne bakın.
</Note>

## Video oluşturma

Paketle birlikte gelen `google` plugin ayrıca paylaşılan
`video_generate` aracı üzerinden video oluşturmayı kaydeder.

- Varsayılan video modeli: `google/veo-3.1-fast-generate-preview`
- Modlar: metinden videoya, görüntüden videoya ve tek video referanslı akışlar
- `aspectRatio` (`16:9`, `9:16`) ve `resolution` (`720P`, `1080P`) desteklenir; ses çıkışı şu anda Veo tarafından desteklenmemektedir
- Desteklenen süreler: **4, 6 veya 8 saniye** (diğer değerler izin verilen en yakın değere yuvarlanır)

Google'ı varsayılan video sağlayıcısı olarak kullanmak için:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

<Note>
Paylaşılan araç parametreleri, sağlayıcı seçimi ve yük devretme davranışı için [Video Oluşturma](/tr/tools/video-generation) bölümüne bakın.
</Note>

## Müzik oluşturma

Paketle birlikte gelen `google` plugin ayrıca paylaşılan
`music_generate` aracı üzerinden müzik oluşturmayı kaydeder.

- Varsayılan müzik modeli: `google/lyria-3-clip-preview`
- Ayrıca desteklenir: `google/lyria-3-pro-preview`
- İstem denetimleri: `lyrics` ve `instrumental`
- Çıkış biçimi: varsayılan olarak `mp3`, ayrıca `google/lyria-3-pro-preview` üzerinde `wav`
- Referans girdileri: en fazla 10 görüntü
- Oturum destekli çalıştırmalar, `action: "status"` dâhil olmak üzere paylaşılan görev/durum akışı üzerinden ayrılır

Google'ı varsayılan müzik sağlayıcısı olarak kullanmak için:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

<Note>
Paylaşılan araç parametreleri, sağlayıcı seçimi ve yük devretme davranışı için [Müzik Oluşturma](/tr/tools/music-generation) bölümüne bakın.
</Note>

## Metinden konuşmaya

Paketle birlikte gelen `google` konuşma sağlayıcısı, `gemini-3.1-flash-tts-preview` ile
Gemini API TTS yolunu kullanır.

- Varsayılan ses: `Kore`
- Kimlik doğrulama: `tts.providers.google.apiKey`, `models.providers.google.apiKey`, `GEMINI_API_KEY` veya `GOOGLE_API_KEY`
- Çıkış: normal TTS ekleri için WAV, sesli not hedefleri için Opus, Talk/telefon için PCM
- Sesli not çıkışı: Google PCM, WAV olarak sarmalanır ve `ffmpeg` ile 48 kHz Opus'a dönüştürülür

Google'ın toplu Gemini TTS yolu, oluşturulan sesi tamamlanmış
`generateContent` yanıtında döndürür. En düşük gecikmeli sesli görüşmeler için toplu
TTS yerine Gemini Live API destekli Google gerçek zamanlı ses sağlayıcısını
kullanın.

Google'ı varsayılan TTS sağlayıcısı olarak kullanmak için:

```json5
{
  tts: {
    auto: "always",
    provider: "google",
    providers: {
      google: {
        model: "gemini-3.1-flash-tts-preview",
        speakerVoice: "Kore",
        audioProfile: "Sakin bir tonla profesyonelce konuşun.",
      },
    },
  },
}
```

Gemini API TTS, stil denetimi için doğal dil istemleri kullanır. Konuşulan metinden önce
yeniden kullanılabilir bir stil istemi eklemek için `audioProfile` değerini ayarlayın. İstem
metni adlandırılmış bir konuşmacıya atıfta bulunuyorsa `speakerName` değerini ayarlayın.

Gemini API TTS ayrıca metin içinde `[whispers]` veya `[laughs]` gibi etkileyici
köşeli parantezli ses etiketlerini kabul eder. Etiketleri TTS'ye gönderirken görünür sohbet yanıtının
dışında tutmak için bunları bir `[[tts:text]]...[[/tts:text]]` bloğunun içine yerleştirin:

```text
İşte sade yanıt metni.

[[tts:text]][whispers] İşte seslendirilen sürüm.[[/tts:text]]
```

<Note>
Gemini API ile sınırlandırılmış bir Google Cloud Console API anahtarı bu
sağlayıcı için geçerlidir. Bu, ayrı Cloud Text-to-Speech API yolu değildir.
</Note>

## Gerçek zamanlı ses

Paketle birlikte gelen `google` plugin, Voice Call ve Google Meet gibi arka uç ses köprüleri için
Gemini Live API destekli bir gerçek zamanlı ses sağlayıcısı kaydeder.

| Ayar                  | Yapılandırma yolu                                                   | Varsayılan                                                                            |
| --------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Model                 | `plugins.entries.voice-call.config.realtime.providers.google.model` | `gemini-3.1-flash-live-preview`                                                       |
| Ses                   | `...google.voice`                                                   | `Kore`                                                                                |
| Sıcaklık              | `...google.temperature`                                             | (ayarlanmamış)                                                                        |
| VAD başlangıç hassasiyeti | `...google.startSensitivity`                                        | (ayarlanmamış)                                                                        |
| VAD bitiş hassasiyeti | `...google.endSensitivity`                                          | (ayarlanmamış)                                                                        |
| Sessizlik süresi      | `...google.silenceDurationMs`                                       | (ayarlanmamış)                                                                        |
| Etkinlik işleme       | `...google.activityHandling`                                        | Google varsayılanı, `start-of-activity-interrupts`                                        |
| Tur kapsamı           | `...google.turnCoverage`                                            | Google varsayılanı, `audio-activity-and-all-video`                                        |
| Otomatik VAD'yi devre dışı bırak | `...google.automaticActivityDetectionDisabled`                      | `false`                                                                               |
| Oturumu sürdürme      | `...google.sessionResumption`                                       | `true`                                                                                |
| Bağlam sıkıştırma     | `...google.contextWindowCompression`                                | `true`                                                                                |
| API anahtarı          | `...google.apiKey`                                                  | `models.providers.google.apiKey`, `GEMINI_API_KEY` veya `GOOGLE_API_KEY` değerine geri döner |

Örnek gerçek zamanlı Sesli Arama yapılandırması:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          realtime: {
            enabled: true,
            provider: "google",
            providers: {
              google: {
                model: "gemini-3.1-flash-live-preview",
                speakerVoice: "Kore",
                activityHandling: "start-of-activity-interrupts",
                turnCoverage: "audio-activity-and-all-video",
              },
            },
          },
        },
      },
    },
  },
}
```

<Note>
Google Live API, WebSocket üzerinden çift yönlü ses ve işlev çağrısı kullanır.
OpenClaw, telefon/Meet köprüsü sesini Gemini'nin PCM Live API akışına uyarlar ve
araç çağrılarını paylaşılan gerçek zamanlı ses sözleşmesinde tutar. Örnekleme
değişikliklerine ihtiyacınız olmadığı sürece `temperature` değerini ayarlamayın;
Google Live, `temperature: 0` için ses olmadan dökümler döndürebildiğinden OpenClaw
pozitif olmayan değerleri dahil etmez. Gemini API transkripsiyonu
`languageCodes` olmadan etkinleştirilir; mevcut Google SDK'sı bu API yolunda
dil kodu ipuçlarını reddeder.
</Note>

<Note>
Gemini 3.1 Live, gerçek zamanlı giriş üzerinden konuşma metnini kabul eder ve
sıralı işlev çağrısı kullanır. OpenClaw bu model için eski `NON_BLOCKING`,
işlev yanıtı zamanlaması ve duygusal diyalog alanlarını dahil etmez.
`thinkingLevel` tercih edin; yapılandırılmış pozitif `thinkingBudget`
değerleri desteklenen en yakın düzeye eşlenirken `-1`, Google'ın
varsayılanını korur. [Gemini Live yetenek karşılaştırmasına](https://ai.google.dev/gemini-api/docs/live-api/capabilities) bakın.
</Note>

<Note>
Control UI Talk, kısıtlı tek kullanımlık token'larla Google Live tarayıcı
oturumlarını destekler. Video Talk'ta tarayıcı, sınırlandırılmış JPEG karelerini
doğrudan Google Live'a, sağlayıcının saniyede en fazla bir kare sınırıyla gönderir.
`describe_view` işlevi, bu kamera akışının etkin olup olmadığını bildirir.
Kamera kareleri Gateway üzerinden geçmez. Yalnızca arka uçta çalışan gerçek
zamanlı ses sağlayıcıları da sağlayıcı kimlik bilgilerini Gateway üzerinde tutan
genel Gateway aktarma taşıması üzerinden çalışabilir.
</Note>

Bakım sorumlusu canlı doğrulaması için
`OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts` komutunu çalıştırın.
Hızlı kontrol ayrıca OpenAI arka uç/WebRTC yollarını da kapsar; Google ayağı,
Control UI Talk tarafından kullanılanla aynı kısıtlı Live API token biçimini
oluşturur, tarayıcı WebSocket uç noktasını açar, ilk kurulum yükünü bir JPEG
karesiyle birlikte gönderir ve bir metin yanıtını ve `describe_view` işlev
gidiş dönüşünü doğrular.

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Doğrudan Gemini önbelleğini yeniden kullanma">
    Doğrudan Gemini API çalıştırmalarında (`api: "google-generative-ai"`), OpenClaw
    yapılandırılmış bir `cachedContent` tanıtıcısını Gemini isteklerine aktarır.

    - Model başına veya genel parametreleri `cachedContent` ya da eski
      `cached_content` ile yapılandırın
    - Daha özel bir kapsamdaki parametreler (genel yerine model düzeyi) her zaman önceliklidir.
      Aynı kapsamda her iki anahtar da ayarlanmışsa `cached_content` önceliklidir.
      Beklenmedik sonuçlardan kaçınmak için kapsam başına yalnızca bir anahtar kullanın.
    - Örnek değer: `cachedContents/prebuilt-context`
    - Gemini önbellek isabeti kullanımı, üst kaynak `cachedContentTokenCount` değerinden
      OpenClaw `cacheRead` değerine normalleştirilir

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "google/gemini-2.5-pro": {
              params: {
                cachedContent: "cachedContents/prebuilt-context",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Gemini CLI kullanım notları">
    `google-gemini-cli` OAuth sağlayıcısını kullanırken OpenClaw, varsayılan olarak
    Gemini CLI `stream-json` çıktısını kullanır ve son `stats`
    yükündeki kullanımı normalleştirir. Eski `--output-format json` geçersiz kılmaları
    JSON ayrıştırıcısını kullanmaya devam eder.

    - Akışla iletilen yanıt metni, asistan `message` olaylarından gelir.
    - Eski JSON çıktısında yanıt metni, CLI JSON `response` alanından gelir.
    - CLI, `usage` değerini boş bıraktığında kullanım `stats` değerine geri döner.
    - `stats.cached`, OpenClaw `cacheRead` değerine normalleştirilir.
    - `stats.input` yoksa OpenClaw, giriş token'larını
      `stats.input_tokens - stats.cached` değerinden türetir.

  </Accordion>

  <Accordion title="Ortam ve daemon kurulumu">
    Gateway bir daemon (launchd/systemd) olarak çalışıyorsa `GEMINI_API_KEY`
    değerinin bu süreç tarafından kullanılabildiğinden emin olun (örneğin
    `~/.openclaw/.env` içinde veya `env.shellEnv` aracılığıyla).
  </Accordion>
</AccordionGroup>

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıları, model referanslarını ve yük devretme davranışını seçme.
  </Card>
  <Card title="Görsel oluşturma" href="/tr/tools/image-generation" icon="image">
    Paylaşılan görsel aracı parametreleri ve sağlayıcı seçimi.
  </Card>
  <Card title="Video oluşturma" href="/tr/tools/video-generation" icon="video">
    Paylaşılan video aracı parametreleri ve sağlayıcı seçimi.
  </Card>
  <Card title="Müzik oluşturma" href="/tr/tools/music-generation" icon="music">
    Paylaşılan müzik aracı parametreleri ve sağlayıcı seçimi.
  </Card>
</CardGroup>
