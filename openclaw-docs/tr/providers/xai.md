---
read_when:
    - OpenClaw'da Grok modellerini kullanmak istiyorsunuz
    - xAI kimlik doğrulamasını veya model kimliklerini yapılandırıyorsunuz
summary: OpenClaw'da xAI Grok modellerini kullanma
title: xAI
x-i18n:
    generated_at: "2026-07-26T23:45:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71ae7b049649b08b6508b8331714fec3464628814629256ad23b584f0f8ca8b7
    source_path: providers/xai.md
    workflow: 16
---

OpenClaw, Grok modelleri için paketle birlikte gelen bir `xai` sağlayıcı Plugin'i sunar. Önerilen yöntem, uygun bir SuperGrok veya X Premium aboneliğiyle Grok OAuth kullanmaktır. Gateway, yapılandırma, yönlendirme ve araçlar yerel kalır; yalnızca Grok istekleri xAI'ın API'sine gider.

OAuth, xAI API anahtarı veya Grok Build uygulaması gerektirmez. OpenClaw, xAI'ın paylaşılan OAuth istemcisini kullandığından xAI, onay ekranında yine de Grok Build'i gösterebilir.

## Kurulum

<Steps>
  <Step title="Yeni kurulum">
    Arka plan hizmeti kurulumuyla ilk yapılandırmayı çalıştırın, ardından model/kimlik doğrulama adımında xAI/Grok OAuth'u seçin:

    ```bash
    openclaw onboard --install-daemon
    ```

    Bir VPS'de veya SSH üzerinden xAI OAuth'u doğrudan seçin; cihaz koduyla doğrulama kullanır ve localhost geri çağrısı gerektirmez:

    ```bash
    openclaw onboard --install-daemon --auth-choice xai-oauth
    ```

  </Step>
  <Step title="Mevcut kurulum">
    Yalnızca xAI'da oturum açın; sadece Grok'u bağlamak için ilk yapılandırmanın tamamını yeniden çalıştırmayın:

    ```bash
    openclaw models auth login --provider xai --method oauth
    ```

    Grok'u varsayılan model olarak ayrıca ayarlayın:

    ```bash
    openclaw models set xai/grok-4.3
    ```

    İlk yapılandırmanın tamamını yalnızca Gateway, arka plan hizmeti, kanal, çalışma alanı veya diğer kurulum seçimlerini bilerek değiştirmek istiyorsanız yeniden çalıştırın.

  </Step>
  <Step title="API anahtarı yöntemi">
    API anahtarıyla kurulum, xAI Console anahtarları ve anahtara dayalı sağlayıcı yapılandırması gerektiren medya yüzeyleri için çalışmaya devam eder:

    ```bash
    openclaw models auth login --provider xai --method api-key
    export XAI_API_KEY=xai-...
    ```

  </Step>
  <Step title="Model seçme">
    ```json5
    {
      agents: { defaults: { model: { primary: "xai/grok-4.3" } } },
    }
    ```
  </Step>
</Steps>

<Note>
OpenClaw, paketle birlikte gelen xAI aktarımı olarak xAI Responses API'yi kullanır. `openclaw models auth login --provider xai --method oauth` veya `--method api-key` kaynağındaki aynı kimlik bilgisi; `web_search` (sağlayıcı kimliği `grok`), `x_search`, `code_execution`, konuşma/transkripsiyon ve xAI görüntü/video üretimini de çalıştırır. Bir xAI anahtarını `plugins.entries.xai.config.webSearch.apiKey` altında saklarsanız paketle birlikte gelen xAI model sağlayıcısı bunu yedek seçenek olarak da yeniden kullanır.
</Note>

## OAuth sorunlarını giderme

- SSH, Docker, VPS veya diğer uzak kurulumlar için `openclaw models auth login --provider xai --method oauth` kullanın; bu yöntem localhost geri çağrısı yerine cihaz koduyla doğrulama kullanır.
- Oturum açma başarılı olduğu hâlde Grok varsayılan model değilse `openclaw models set xai/grok-4.3` komutunu çalıştırın.
- Kaydedilmiş xAI kimlik doğrulama profillerini inceleyin:

  ```bash
  openclaw models auth list --provider xai
  openclaw models status
  ```

- Hangi hesapların OAuth API belirteçlerini alabileceğine xAI karar verir. Bir hesap uygun değilse API anahtarı yöntemini kullanın veya xAI tarafındaki aboneliği kontrol edin.

<Tip>
SSH, Docker veya bir VPS'den oturum açarken `xai-oauth` kullanın. OpenClaw bir URL ve kısa kod yazdırır; uzak işlem, tamamlanan belirteç alışverişi için xAI'ı yoklarken herhangi bir yerel tarayıcıda oturum açma işlemini tamamlayın.
</Tip>

## Yerleşik katalog

Model seçicilerde seçilebilen kimlikler. Plugin, mevcut yapılandırmalar için eski Grok 3, Grok 4, Grok 4 Fast, Grok 4.1 Fast ve Grok Code kimliklerini çözümlemeye devam eder; bkz. [eski sürüm uyumluluğu ve değişken takma adlar](#legacy-compatibility-and-moving-aliases).

| Aile           | Model kimlikleri                                              |
| -------------- | ------------------------------------------------------------ |
| Grok 4.5       | `grok-4.5` (takma adlar: `grok-4.5-latest`, `grok-build-latest`) |
| Grok Build 0.1 | `grok-build-0.1`                                             |
| Grok 4.3       | `grok-4.3` (takma adlar: `grok-4.3-latest`, `grok-latest`)       |
| Grok 4.20      | `grok-4.20-0309-reasoning`, `grok-4.20-0309-non-reasoning`   |

<Tip>
Kullanılabildiği yerlerde genel sohbet, kodlama ve aracılı çalışma için `grok-4.5` kullanın. Grok 4.3, bölgesel açıdan güvenli kurulum varsayılanı olmaya devam eder; `grok-build-0.1` ve tarihli iki Grok 4.20 çeşidi seçilebilir durumda kalır.
</Tip>

Katalog bağlamı ve belirteç maliyeti meta verileri, xAI'ın güncel [model sayfalarını](https://docs.x.ai/developers/models) ve [fiyatlandırma sayfasını](https://docs.x.ai/developers/pricing) izler. Bir istek belgelenen uzun bağlam eşiğini aştığında xAI daha yüksek ücretler uygular; OpenClaw'ın sabit katalog maliyeti alanları kısa bağlam ücretlerini kaydeder. xAI'ın ayrı kodlama aracısı CLI'ı olan Grok Build, [x.ai/cli](https://x.ai/cli) adresinden edinilebilir ve şu anda Grok 4.5 kullanır.

## Özellik kapsamı

Paketle birlikte gelen Plugin, desteklenen xAI API'lerini OpenClaw'ın paylaşılan sağlayıcı ve araç sözleşmeleriyle eşler. Paylaşılan sözleşmeye uymayan yetenekler aşağıda veya bilinen sınırlamalar bölümünde listelenmiştir.

| xAI yeteneği                | OpenClaw yüzeyi                         | Durum                                                |
| -------------------------- | --------------------------------------- | ---------------------------------------------------- |
| Sohbet / Responses         | `xai/<model>` model sağlayıcısı            | Evet                                                 |
| Sunucu tarafında web araması | `web_search` sağlayıcısı `grok`            | Evet                                                 |
| Sunucu tarafında X araması | `x_search` aracı                         | Evet                                                 |
| Sunucu tarafında kod yürütme | `code_execution` aracı                   | Evet                                                 |
| Görüntüler                 | `image_generate`                        | Evet                                                 |
| Videolar                   | `video_generate`                        | Evet                                                 |
| Toplu metinden konuşmaya   | `tts.provider: "xai"` / `tts`           | Evet                                                 |
| Akışlı TTS                 | `textToSpeechStream`                    | `wss://api.x.ai/v1/tts` üzerinden evet (gerçek zamanlı ses değil) |
| Toplu konuşmadan metne     | `tools.media.audio` medya anlama | Evet                                                 |
| Akışlı konuşmadan metne    | Voice Call `streaming.provider: "xai"`  | Evet                                                 |
| Gerçek zamanlı ses         | Talk `talk.realtime.provider: "xai"`    | Evet; yerel Talk Node'ları için gateway aktarımı     |
| Dosyalar / toplu işlemler  | Yalnızca genel model API uyumluluğu      | Birinci sınıf bir OpenClaw aracı değil               |

<Note>
OpenClaw; medya üretimi ve toplu transkripsiyon için xAI'ın REST görüntü/video/TTS/STT API'lerini, canlı sesli arama transkripsiyonu için xAI'ın akışlı STT WebSocket'ini, Talk gerçek zamanlı oturumları için xAI'ın Grok Voice Agent WebSocket'ini ve sohbet, arama ve kod yürütme araçları için Responses API'yi kullanır.
</Note>

### Eski hızlı mod uyumluluğu

`/fast on` veya `agents.defaults.models["xai/<model>"].params.fastMode: true` eski xAI yapılandırmalarını aşağıdaki şekilde yeniden yazmaya devam eder. Bu hedef kimlikler yalnızca uyumluluk için korunur; yeni yapılandırmalar için güncel seçilebilir modelleri kullanın.

| Kaynak model  | Hızlı mod hedefi   |
| ------------- | ------------------ |
| `grok-3`      | `grok-3-fast`      |
| `grok-3-mini` | `grok-3-mini-fast` |
| `grok-4`      | `grok-4-fast`      |
| `grok-4-0709` | `grok-4-fast`      |

### Eski sürüm uyumluluğu ve değişken takma adlar

Eski takma adlar aşağıdaki şekilde normalleştirilir:

| Eski takma ad                                                 | Normalleştirilmiş kimlik |
| ------------------------------------------------------------- | ---------------- |
| `grok-code-fast-1`, `grok-code-fast`, `grok-code-fast-1-0825` | `grok-build-0.1` |

0309 tarihli kimlikler, seçilebilir katalog girdileridir. OpenClaw, diğer tüm güncel Grok 4.20 takma adlarını olduğu gibi gönderir; böylece xAI kararlı, en yeni, beta, deneysel ve tarihli takma adların anlamları üzerindeki denetimini korur. Genel `grok-latest` takma adı da olduğu gibi korunur.

xAI aşağıdaki tam kimlikleri kullanımdan kaldırdı. OpenClaw bunları, yayımlanmış yapılandırmalar için mevcut yönlendirme hedeflerinin sınırları ve fiyatlandırmasıyla gizli uyumluluk satırları olarak korur:

| Kullanımdan kaldırılan kimlikler                                    | Güncel davranış                  |
| -------------------------------------------------------------------- | -------------------------------- |
| `grok-4-1-fast-reasoning`, `grok-4-fast-reasoning`, `grok-4-0709`    | `low` akıl yürütmeli Grok 4.3    |
| `grok-4-1-fast-non-reasoning`, `grok-4-fast-non-reasoning`, `grok-3` | Akıl yürütmesi devre dışı Grok 4.3 |
| `grok-code-fast-1`                                                   | Grok Build 0.1                   |
| `grok-imagine-image-pro`                                             | Grok Imagine Görüntü Kalitesi    |

`openclaw doctor --fix`, kalıcı xAI sunucu aracı varsayılanlarını ve kullanımdan kaldırılmış kaliteli görüntü kısa adını günceller, eski oluşturulmuş katalog satırlarını kaldırır ve etkin 4.20 satırlarındaki eski bağlam meta verilerini onarır. Etkin 4.20 `beta-latest` takma adlarını tarihli bir anlık görüntüye sabitlemez.

## Özellikler

<Warning>
  `x_search` ve `code_execution` xAI'ın sunucularında çalışır. xAI, modelin giriş ve çıkış belirteçlerine ek olarak 1.000 araç çağrısı başına $5 ücretlendirir. Her aracın `enabled` ayarı belirtilmediğinde OpenClaw aracı yalnızca etkin bir xAI modeli için sunar. Bilinen xAI dışı bir model sağlayıcısı, araç başına açık bir `enabled: true` gerektirir; eksik veya çözümlenemeyen sağlayıcı kapalı durumda başarısız olur. xAI kimlik doğrulaması her zaman gereklidir ve `enabled: false` aracı tüm sağlayıcılar için devre dışı bırakır.
</Warning>

<AccordionGroup>
  <Accordion title="Web araması">
    Paketle birlikte gelen `grok` web araması sağlayıcısı önce xAI OAuth'u tercih eder, ardından `XAI_API_KEY` veya bir Plugin web araması anahtarını yedek seçenek olarak kullanır:

    ```bash
    openclaw models auth login --provider xai --method oauth
    openclaw config set tools.web.search.provider grok
    ```

  </Accordion>

  <Accordion title="Video üretimi">
    Paketle birlikte gelen `xai` Plugin'i, paylaşılan `video_generate` aracı üzerinden video üretimini kaydeder.

    - Varsayılan model: `xai/grok-imagine-video`
    - Ek model: `xai/grok-imagine-video-1.5`
    - Klasik modlar: metinden videoya, görüntüden videoya, referans görüntü üretimi, uzak video düzenleme ve uzak video uzatma
    - Video 1.5 modu: tam olarak bir ilk kare görüntüsüyle yalnızca görüntüden videoya
    - En-boy oranları: `1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `3:2`, `2:3`; belirtilmediğinde klasik ve Video 1.5 görüntüden videoya modları kaynak görüntünün oranını devralır
    - Çözünürlükler: klasik `480P`/`720P`; Video 1.5 ayrıca `1080P` destekler; tüm üretim modlarının varsayılanı `480P` değeridir
    - Süre: üretim/görüntüden videoya için 1-15 saniye, klasik `reference_image` rolleri kullanılırken 1-10 saniye, klasik uzatma için 2-10 saniye
    - Referans görüntü üretimi: sağlanan her görüntü için `imageRoles` değerini `reference_image` olarak ayarlayın; xAI bu tür en fazla 7 görüntü kabul eder
    - Video düzenleme/uzatma, giriş videosunun en-boy oranını ve çözünürlüğünü devralır; bu işlemler geometri geçersiz kılmalarını kabul etmez
    - Varsayılan işlem zaman aşımı: `video_generate.timeoutMs` veya `agents.defaults.mediaModels.video.timeoutMs` ayarlanmadıkça 600 saniye

    <Warning>
    Yerel video arabellekleri kabul edilmez. Video düzenleme/uzatma girişleri için uzak `http(s)` URL'lerini kullanın. Görüntüden videoya, yerel görüntü arabelleklerini kabul eder; çünkü OpenClaw bunları xAI için veri URL'leri olarak kodlar.
    </Warning>

    Video 1.5 ayrıca xAI'ın `grok-imagine-video-1.5-preview` ve `grok-imagine-video-1.5-2026-05-30` tanımlayıcılarını da tanır. OpenClaw seçilen tanımlayıcıyı değiştirmeden iletir, ancak yalnızca görüntü doğrulamasını aynı şekilde uygular.

    xAI'ı varsayılan video sağlayıcısı olarak kullanmak için:

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "xai/grok-imagine-video",
          },
        },
      },
    }
    ```

    <Note>
    Paylaşılan araç parametreleri, sağlayıcı seçimi ve yük devretme davranışı için
    [Video Oluşturma](/tr/tools/video-generation) bölümüne bakın.
    </Note>

  </Accordion>

  <Accordion title="Görsel oluşturma">
    Birlikte gelen `xai` Plugin'i, paylaşılan
    `image_generate` aracı üzerinden görsel oluşturmayı kaydeder.

    - Varsayılan görsel modeli: `xai/grok-imagine-image`
    - Ek model: `xai/grok-imagine-image-quality`
    - Modlar: metinden görsele ve referans görsel düzenleme
    - Referans girdileri: bir `image` veya en fazla üç `images`
    - En-boy oranları: `1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `3:2`, `2:3`, `2:1`,
      `1:2`, `19.5:9`, `9:19.5`, `20:9`, `9:20`
    - Çözünürlükler: `1K`, `2K`
    - Sayı: en fazla 4 görsel
    - Varsayılan işlem zaman aşımı: `image_generate.timeoutMs`
      veya `agents.defaults.mediaModels.image.timeoutMs` ayarlanmadığı sürece 600 saniye

    OpenClaw, oluşturulan medyanın normal kanal eki yolu üzerinden depolanıp
    teslim edilebilmesi için xAI'dan `b64_json` görsel yanıtları ister. Yerel
    referans görseller veri URL'lerine dönüştürülür; uzak `http(s)` referansları
    değiştirilmeden iletilir.

    xAI'ı varsayılan görsel sağlayıcısı olarak kullanmak için:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "xai/grok-imagine-image",
          },
        },
      },
    }
    ```

    <Note>
    xAI ayrıca `quality`, `mask`, `user` ve bir `auto` en-boy oranını belgeler.
    OpenClaw günümüzde yalnızca sağlayıcılar arasında paylaşılan görsel denetimlerini iletir;
    yalnızca yerel olan bu ayarlar `image_generate` üzerinden sunulmaz.
    </Note>

  </Accordion>

  <Accordion title="Metinden konuşmaya">
    Birlikte gelen `xai` Plugin'i, paylaşılan `tts`
    sağlayıcı yüzeyi üzerinden metinden konuşmaya özelliğini kaydeder.

    - Sesler: xAI'dan kimliği doğrulanmış canlı katalog; şu komutla listeleyin:
      `openclaw infer tts voices --provider xai`
    - Çevrimdışı yedek sesler: `ara`, `eve`, `leo`, `rex`, `sal`
    - Varsayılan ses: `eve`
    - Hesaba özel ses kimlikleri, yerleşik katalog yanıtında bulunmasalar bile
      iletilir
    - Biçimler: `mp3`, `wav`, `pcm`, `mulaw`, `alaw`
    - Dil: BCP-47 kodu veya `auto`
    - Hız: sağlayıcıya özgü hız geçersiz kılması
    - Yerel Opus sesli not biçimi desteklenmez

    xAI'ı varsayılan TTS sağlayıcısı olarak kullanmak için:

    ```json5
    {
      tts: {
        provider: "xai",
        providers: {
          xai: {
            voiceId: "eve",
          },
        },
      },
    }
    ```

    <Note>
    OpenClaw, arabelleğe alınmış sentez için xAI'ın toplu `/v1/tts` uç noktasını,
    kimliği doğrulanmış `/v1/tts/voices` katalog keşfini ve akışlı sentez için yerel
    `wss://api.x.ai/v1/tts` kullanır. Akış, yerel `api.x.ai` ana makinesiyle sınırlıdır;
    bu nedenle özel `baseUrl` değerleri bu yolda reddedilir. Mevcut dil, ses,
    codec ve hız denetimleri kullanılır; örnekleme hızı ve bit hızı için xAI
    varsayılanları geçerlidir. Ses dosyası sentezi, yapılandırılmış tüm codec'lere
    uyar. Sesli not hedefleri, xAI'ın ham codec'leri codec/hız meta verilerini
    taşımadığı için akışta ve arabelleğe alınmış yedekte MP3 kullanır. Akış önce
    `text.delta`, ardından
    `text.done` gönderir; `audio.delta`, `audio.done` veya `error` alır ve her
    ses parçasında yenilenen bir boşta kalma `timeoutMs` uygular. Bu, gerçek
    zamanlı ses oturumlarından ayrıdır. xAI'ın [Akışlı TTS API'si](https://docs.x.ai/developers/rest-api-reference/inference/voice) sözleşmesine bakın.
    </Note>

  </Accordion>

  <Accordion title="Konuşmadan metne">
    Birlikte gelen `xai` Plugin'i, OpenClaw'ın medya anlama transkripsiyon
    yüzeyi üzerinden toplu konuşmadan metne özelliğini kaydeder.

    - Uç nokta: xAI REST `/v1/stt`
    - Girdi yolu: çok parçalı ses dosyası yüklemesi
    - Model seçimi: xAI transkripsiyon modelini dahili olarak seçer;
      uç noktada model seçici yoktur
    - Discord ses kanalı bölümleri ve kanal ses ekleri dâhil olmak üzere,
      gelen ses transkripsiyonunun `tools.media.audio` okuduğu her yerde kullanılır

    Gelen ses transkripsiyonu için xAI kullanımını zorunlu kılmak üzere:

    ```json5
    {
      tools: {
        media: {
          audio: {
            models: [
              {
                type: "provider",
                provider: "xai",
              },
            ],
          },
        },
      },
    }
    ```

    Dil, paylaşılan ses medyası yapılandırması veya çağrı başına transkripsiyon
    isteği üzerinden sağlanabilir. İstem ipuçları paylaşılan OpenClaw yüzeyi
    tarafından kabul edilir; ancak xAI REST STT entegrasyonu yalnızca dosya ve
    dili iletir, çünkü bunlar mevcut genel xAI uç noktasıyla eşleşir.

  </Accordion>

  <Accordion title="Akışlı konuşmadan metne">
    Birlikte gelen `xai` Plugin'i, canlı sesli arama sesi için gerçek zamanlı
    bir transkripsiyon sağlayıcısı da kaydeder.

    - Uç nokta: xAI WebSocket `wss://api.x.ai/v1/stt`
    - Varsayılan kodlama: `mulaw`
    - Varsayılan örnekleme hızı: `8000`
    - Varsayılan uç noktalama: `800ms`
    - Ara transkriptler: varsayılan olarak etkin

    Voice Call'un Twilio medya akışı G.711 mu-law ses kareleri gönderir; bu nedenle
    xAI sağlayıcısı bu kareleri kod dönüştürmesi yapmadan doğrudan iletir:

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              streaming: {
                enabled: true,
                provider: "xai",
                providers: {
                  xai: {
                    apiKey: "${XAI_API_KEY}",
                    endpointingMs: 800,
                    language: "en",
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

    Sağlayıcıya ait yapılandırma
    `plugins.entries.voice-call.config.streaming.providers.xai` altında bulunur. Desteklenen
    anahtarlar `apiKey`, `baseUrl`, `sampleRate`, `encoding` (`pcm`, `mulaw` veya
    `alaw`), `interimResults`, `endpointingMs` ve `language` şeklindedir.

    <Note>
    Bu akış sağlayıcısı, Voice Call'un gerçek zamanlı transkripsiyon yolu içindir.
    Discord kısa bölümler kaydeder ve bunun yerine toplu
    `tools.media.audio` transkripsiyon yolunu kullanır.
    </Note>

  </Accordion>

  <Accordion title="Gerçek zamanlı ses (Talk)">
    Birlikte gelen `xai` Plugin'i, paylaşılan `registerRealtimeVoiceProvider` sözleşmesi
    üzerinden Talk modu için Grok Voice Agent gerçek zamanlı oturumlarını kaydeder.

    - Uç nokta: `wss://api.x.ai/v1/realtime?model=<voice-model>`
    - Varsayılan model: `grok-voice-latest`
    - Varsayılan ses: `eve`
    - Aktarım: `gateway-relay` (iOS, Android ve Control UI aktarma yolları)
    - Ses: PCM16 24 kHz veya G.711 µ-law 8 kHz
    - Araya girme: xAI sunucu VAD'si yanıtı keser; OpenClaw sıraya alınmış oynatmayı
      temizler ve oynatılmamış sağlayıcı geçmişini kısaltır

    Talk'u Gateway üzerinde yapılandırın:

    ```json5
    {
      talk: {
        realtime: {
          provider: "xai",
          mode: "realtime",
          transport: "gateway-relay",
          brain: "agent-consult",
          providers: {
            xai: {
              model: "grok-voice-latest",
              voice: "eve",
              // Yalnızca sağlayıcı tarafında oturumun yeniden oynatılması kabul edilebilirse etkinleştirin.
              sessionResumption: false,
            },
          },
        },
      },
      env: { XAI_API_KEY: "xai-..." },
    }
    ```

    Voice Call veya paylaşılan gerçek zamanlı seçiciler aynı sağlayıcı eşlemesini
    yeniden kullandığında, sağlayıcıya ait yapılandırma
    `plugins.entries.voice-call.config.realtime.providers.xai` üzerinden de çözümlenir. Desteklenen anahtarlar
    `apiKey`, `baseUrl`, `model`, `voice`, `vadThreshold`, `silenceDurationMs`,
    `prefixPaddingMs`, `reasoningEffort` ve `sessionResumption` şeklindedir.
    `reasoningEffort`, xAI Voice Agent API'siyle eşleşecek şekilde yalnızca `high` veya `none` kabul eder.

    xAI'ın sunucu VAD'si her zaman yanıtlar oluşturur ve ses kesintisini işler.
    `consultRouting: "provider-direct"` kullanın; zorunlu transkript yönlendirmesi ve
    giriş sesi kesintisinin devre dışı bırakılması xAI Voice Agent protokolü tarafından desteklenmez.

    <Note>
    xAI OAuth veya `XAI_API_KEY`, gerçek zamanlı sesin kimliğini doğrulayabilir. Tarayıcı
    tarafından yönetilen WebRTC henüz bu sağlayıcı yüzeyinin parçası değildir; yerel
    Node'larda gateway-relay Talk'u veya Control UI aktarma yolunu kullanın.
    </Note>

    <Note>
    `sessionResumption` varsayılan olarak `false` değerini alır. `true` olarak ayarlandığında OpenClaw,
    yeniden bağlantıdan sonra aynı konuşmayı sürdürebilmek için xAI'dan yeterli
    oturum durumunu saklamasını ister ve ardından döndürülen konuşma kimliğiyle
    yeniden bağlanır. Sağlayıcı tarafında yeniden oynatma/saklama kabul edilemezse
    bunu devre dışı bırakın; böylece kesintiye uğrayan soketler sessizce yeni bir
    konuşma başlatmak yerine kapalı biçimde başarısız olur.
    </Note>

  </Accordion>

  <Accordion title="x_search yapılandırması">
    Birlikte gelen xAI Plugin'i, Grok aracılığıyla X (eski adıyla Twitter) içeriğinde
    arama yapmak için `x_search` aracını bir OpenClaw aracı olarak sunar.

    Yapılandırma yolu: `plugins.entries.xai.config.xSearch`

    | Anahtar           | Tür     | Varsayılan                | Açıklama                                         |
    | ----------------- | ------- | ------------------------- | ------------------------------------------------ |
    | `enabled`         | boolean | xAI modelleri için otomatik | Devre dışı bırakın veya bilinen bir xAI dışı sağlayıcı için etkinleştirin |
    | `model`           | string  | `grok-4.3`                | x_search istekleri için kullanılan model         |
    | `baseUrl`         | string  | -                         | xAI Responses temel URL geçersiz kılması          |
    | `inlineCitations` | boolean | -                         | Sonuçlara satır içi alıntıları dâhil et           |
    | `maxTurns`        | number  | -                         | En fazla konuşma turu                             |
    | `timeoutSeconds`  | number  | `30`                      | Saniye cinsinden istek zaman aşımı                |
    | `cacheTtlMinutes` | number  | `15`                      | Dakika cinsinden önbellek yaşam süresi            |

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              xSearch: {
                enabled: true,
                model: "grok-4.3",
                baseUrl: "https://api.x.ai/v1",
                inlineCitations: true,
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Kod yürütme yapılandırması">
    Birlikte gelen xAI Plugin'i, xAI'ın korumalı alan ortamında uzaktan kod
    yürütmek için `code_execution` aracını bir OpenClaw aracı olarak sunar.

    Yapılandırma yolu: `plugins.entries.xai.config.codeExecution`

    | Anahtar          | Tür     | Varsayılan                 | Açıklama                                                    |
    | ---------------- | ------- | -------------------------- | ----------------------------------------------------------- |
    | `enabled`        | boolean | xAI modelleri için otomatik | Devre dışı bırakın veya bilinen bir xAI dışı sağlayıcı için etkinleştirin |
    | `model`          | string  | `grok-4.3`               | Kod yürütme istekleri için kullanılan model                 |
    | `maxTurns`       | number  | -                          | Maksimum konuşma turu                                       |
    | `timeoutSeconds` | number  | `30`                     | Saniye cinsinden istek zaman aşımı                          |

    <Note>
    Bu, yerel [`exec`](/tr/tools/exec) değil, uzak xAI korumalı alan yürütmesidir.
    </Note>

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              codeExecution: {
                enabled: true,
                model: "grok-4.3",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Bilinen sınırlamalar">
    - xAI kimlik doğrulaması bir API anahtarı, ortam değişkeni, plugin yapılandırması
      yedeği veya uygun bir xAI hesabıyla OAuth kullanabilir. OAuth, localhost
      geri çağrısı olmadan cihaz kodu doğrulaması kullanır. Hangi hesapların
      OAuth API belirteçleri alabileceğine xAI karar verir ve OpenClaw, Grok Build
      uygulamasını gerektirmese de onay sayfasında Grok Build gösterilebilir.
    - OpenClaw şu anda xAI çok aracılı model ailesini kullanıma sunmaz. xAI,
      bu modelleri Responses API üzerinden sunar ancak modeller, OpenClaw'ın
      paylaşılan aracı döngüsünün kullandığı istemci tarafı veya özel araçları
      kabul etmez. Bkz.
      [xAI çok aracılı sistem sınırlamaları](https://docs.x.ai/developers/model-capabilities/text/multi-agent#limitations).
    - xAI Realtime ses şu anda yalnızca gateway aktarmalı Talk taşımasını sunar.
      Tarayıcının yönettiği sağlayıcı WebSocket oturumları henüz Control UI'a
      bağlanmamıştır.
    - xAI görüntü `quality`, görüntü `mask` ve yalnızca yerel kullanıma özgü ek en-boy oranları,
      paylaşılan `image_generate` aracında bunlara karşılık gelen
      sağlayıcılar arası denetimler bulunana kadar kullanıma sunulmaz.
  </Accordion>

  <Accordion title="İleri düzey notlar">
    - OpenClaw, paylaşılan çalıştırıcı yolunda xAI'a özgü araç şeması ve araç çağrısı uyumluluk
      düzeltmelerini otomatik olarak uygular.
    - Yerel xAI isteklerinde `tool_stream: true` varsayılan olarak etkindir. Devre dışı bırakmak için
      `agents.defaults.models["xai/<model>"].params.tool_stream` değerini `false`
      olarak ayarlayın.
    - Paketle gelen xAI sarmalayıcısı, yerel xAI isteklerini göndermeden önce desteklenmeyen
      contains-count şema sınırlarını ve desteklenmeyen akıl yürütme *çabası* yük anahtarlarını
      kaldırır. Grok 4.5 düşük, orta ve yüksek çabayı destekler
      (varsayılan yüksek). Grok 4.3 çabasız, düşük, orta ve yüksek
      çabayı destekler (varsayılan düşük). Akıl yürütme yeteneğine sahip diğer xAI modelleri,
      yapılandırılabilir bir çaba denetimi sunmaz ancak önceki şifrelenmiş akıl yürütmenin
      sonraki turlarda yeniden oynatılabilmesi için yine de
      `include: ["reasoning.encrypted_content"]` ister.
    - `web_search`, `x_search` ve `code_execution`, OpenClaw
      araçları olarak sunulur. OpenClaw, her yerel aracı her sohbet turuna eklemek
      yerine yalnızca her aracın ihtiyaç duyduğu belirli yerleşik xAI aracını
      o aracın isteğine ekler.
    - Grok `web_search`, `plugins.entries.xai.config.webSearch.baseUrl` değerini okur.
      `x_search`, `plugins.entries.xai.config.xSearch.baseUrl` değerini okur ve ardından
      Grok web araması temel URL'sine geri döner.
    - `x_search` ve `code_execution`, çekirdek model çalışma zamanına
      sabit kodlanmak yerine paketle gelen xAI plugin'i tarafından yönetilir.
    - `code_execution`, yerel
      [`exec`](/tr/tools/exec) değil, uzak xAI korumalı alan yürütmesidir.
  </Accordion>
</AccordionGroup>

## Canlı test

xAI medya yolları, birim testleri ve isteğe bağlı canlı test paketleri kapsamındadır. Canlı
yoklamaları çalıştırmadan önce işlem ortamına `XAI_API_KEY` aktarın.

```bash
pnpm test extensions/xai
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_TEST_QUIET=1 pnpm test:live -- extensions/xai/xai.live.test.ts
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "classic Grok Imagine"
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "Grok Imagine Video 1.5"
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_TEST_QUIET=1 pnpm test:live -- extensions/xai/x-search.live.test.ts
OPENCLAW_LIVE_GATEWAY_MODELS="xai/grok-4.5,xai/grok-build-0.1,xai/grok-4.3,xai/grok-4.20-0309-reasoning,xai/grok-4.20-0309-non-reasoning" OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0 OPENCLAW_LIVE_GATEWAY_SMOKE=0 pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_TEST_QUIET=1 OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS=xai pnpm test:live -- test/image-generation.runtime.live.test.ts
```

Sağlayıcıya özgü canlı dosya; normal TTS ve telefon kullanımına uygun PCM TTS sentezler,
xAI toplu STT aracılığıyla ses yazıya döker, aynı PCM'yi xAI gerçek zamanlı STT üzerinden
aktarır, metinden görüntü çıktısı oluşturur ve bir referans görüntüyü düzenler.
Paylaşılan görüntü canlı dosyası, aynı xAI sağlayıcısını OpenClaw'ın çalışma zamanı
seçimi, yedek mekanizması, normalleştirme ve medya eki yolu üzerinden doğrular. İsteğe
bağlı Video 1.5 örneği, oluşturulan bir ilk kare görüntüsünü 1080P çözünürlükte gönderir
ve tamamlanan videonun indirildiğini doğrular.

## İlgili

<CardGroup cols={2}>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcıların, model referanslarının ve yük devretme davranışının seçilmesi.
  </Card>
  <Card title="Video oluşturma" href="/tr/tools/video-generation" icon="video">
    Paylaşılan video aracı parametreleri ve sağlayıcı seçimi.
  </Card>
  <Card title="Tüm sağlayıcılar" href="/tr/providers/index" icon="grid-2">
    Daha kapsamlı sağlayıcı genel bakışı.
  </Card>
  <Card title="Sorun giderme" href="/tr/help/troubleshooting" icon="wrench">
    Yaygın sorunlar ve düzeltmeler.
  </Card>
</CardGroup>
