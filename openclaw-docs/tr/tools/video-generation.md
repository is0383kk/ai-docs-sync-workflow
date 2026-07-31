---
read_when:
    - Agent aracılığıyla video oluşturma
    - Video oluşturma sağlayıcılarını ve modellerini yapılandırma
    - video_generate aracı parametrelerini anlama
sidebarTitle: Video generation
summary: 16 sağlayıcı arka ucunda metin, görüntü veya video referanslarından video_generate aracılığıyla videolar oluşturun
title: Video oluşturma
x-i18n:
    generated_at: "2026-07-27T00:21:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4afc9338fdfdc269b50b949b6d1a186e3a2064ed4ee40a41722efea40ae791aa
    source_path: tools/video-generation.md
    workflow: 16
---

OpenClaw ajanları, metin istemlerinden, referans görsellerden veya
mevcut videolardan `video_generate` aracılığıyla videolar oluşturur. On altı sağlayıcı arka ucu
desteklenir; ajan, yapılandırmaya ve kullanılabilir API anahtarlarına göre
doğru olanı otomatik olarak seçer.

<Note>
`video_generate`, yalnızca en az bir video oluşturma sağlayıcısı
kullanılabilir olduğunda görünür. Ajan araçlarınızda yoksa bir sağlayıcı API anahtarı ayarlayın veya
`agents.defaults.mediaModels.video` öğesini yapılandırın.
</Note>

`video_generate`, çağrıdaki referans girdilerinden belirlenen üç çalışma zamanı
moduna sahiptir:

- `generate` - referans medya yok (metinden videoya).
- `imageToVideo` - bir veya daha fazla referans görsel.
- `videoToVideo` - bir veya daha fazla referans video.

Sağlayıcılar bu modların herhangi bir alt kümesini destekleyebilir. Araç, gönderimden
önce etkin modu doğrular ve desteklenen modları `action=list` içinde bildirir.

## Hızlı başlangıç

<Steps>
  <Step title="Kimlik doğrulamayı yapılandırın">
    Desteklenen herhangi bir sağlayıcı için bir API anahtarı ayarlayın:

    ```bash
    export GEMINI_API_KEY="your-key"
    ```

  </Step>
  <Step title="Varsayılan model seçin (isteğe bağlı)">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "google/veo-3.1-fast-generate-preview"
    ```
  </Step>
  <Step title="Ajandan isteyin">
    > Gün batımında sörf yapan sevimli bir ıstakozun 5 saniyelik sinematik videosunu oluştur.

    Ajan, `video_generate` öğesini otomatik olarak çağırır. Araç için izin verilenler listesi
    gerekmez.

  </Step>
</Steps>

## Eşzamansız oluşturma nasıl çalışır?

Video oluşturma eşzamansızdır:

1. OpenClaw, isteği sağlayıcıya gönderir ve hemen bir görev kimliği döndürür.
2. Sağlayıcı, işi arka planda işler (sağlayıcıya ve çözünürlüğe bağlı olarak genellikle 30 saniye ile birkaç dakika arasında; kuyruk destekli yavaş sağlayıcılar, yapılandırılmış zaman aşımına kadar çalışabilir).
3. Video hazır olduğunda OpenClaw, aynı oturumu dahili bir tamamlanma olayıyla uyandırır.
4. Ajan, bunu oturumun normal görünür yanıt modu aracılığıyla bildirir:
   otomatik son yanıt veya oturum mesaj aracını gerektiriyorsa `message(action="send")`.
   İstekte bulunan oturum etkin değilse ya da uyandırma işlemi başarısız olur ve oluşturulan
   medya tamamlanma yanıtında hâlâ yoksa OpenClaw, medyayı içeren
   eş etkili doğrudan bir geri dönüş gönderir.

Bir iş devam ederken aynı oturumdaki yinelenen `video_generate`
çağrıları, başka bir oluşturma başlatmak yerine mevcut görev durumunu
döndürür. Yeni bir oluşturmayı tetiklemeden kontrol etmek için `action: "status"`,
CLI üzerinden ise `openclaw tasks list` / `openclaw tasks show <lookup>`
kullanın (bkz. [Arka plan görevleri](/tr/automation/tasks)).

Oturum destekli ajan çalıştırmalarının dışında (örneğin doğrudan araç çağrılarında)
araç, satır içi oluşturmaya geri döner ve son medya yolunu
aynı turda döndürür.

Sağlayıcı bayt döndürdüğünde, oluşturulan video dosyaları OpenClaw tarafından yönetilen
medya depolamasına kaydedilir. Varsayılan üst sınır 16MB'dir (paylaşılan video medya
sınırı); `agents.defaults.mediaMaxMb`, daha büyük işlemeler için bu sınırı yükseltir. Bir
sağlayıcı barındırılan bir çıktı URL'si de döndürürse OpenClaw, yerel kalıcılık
fazla büyük bir dosyayı reddettiğinde görevi başarısız kılmak yerine bu URL'yi teslim eder.

### Görev yaşam döngüsü

| Durum       | Anlamı                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------ |
| `queued`    | Görev oluşturuldu; sağlayıcının kabul etmesi bekleniyor.                                                   |
| `running`   | Sağlayıcı işliyor (sağlayıcıya ve çözünürlüğe bağlı olarak genellikle 30 saniye ile birkaç dakika arasında). |
| `succeeded` | Video hazırdır; ajan uyanır ve videoyu konuşmaya gönderir.                                         |
| `failed`    | Sağlayıcı hatası veya zaman aşımı; ajan hata ayrıntılarıyla uyanır.                                         |

Durumu CLI üzerinden kontrol edin:

```bash
openclaw tasks list
openclaw tasks show <lookup>
openclaw tasks cancel <lookup>
```

## Desteklenen sağlayıcılar

| Sağlayıcı              | Varsayılan model                   | Metin | Görsel referansı                                            | Video referansı                                       | Kimlik doğrulama                                     |
| --------------------- | ------------------------------- | :--: | ---------------------------------------------------- | ----------------------------------------------- | ---------------------------------------- |
| Alibaba               | `wan2.6-t2v`                    |  ✓   | Evet (uzak URL)                                     | Evet (uzak URL)                                | `MODELSTUDIO_API_KEY`                    |
| BytePlus (paketle birlikte)    | `seedance-1-0-pro-250528`       |  ✓   | En fazla 2 görsel (ilk + son kare)                  | -                                               | `BYTEPLUS_API_KEY`                       |
| BytePlus 1.5 Plugin'i   | `seedance-1-5-pro-251215`       |  ✓   | En fazla 2 görsel (rol aracılığıyla ilk + son kare)         | -                                               | `BYTEPLUS_API_KEY`                       |
| BytePlus Seedance 2.0 | `dreamina-seedance-2-0-260128`  |  ✓   | En fazla 9 referans görsel                             | En fazla 3 video                                  | `BYTEPLUS_API_KEY`                       |
| ComfyUI               | `workflow`                      |  ✓   | 1 görsel                                              | -                                               | `COMFY_API_KEY` veya `COMFY_CLOUD_API_KEY` |
| DeepInfra             | `Pixverse/Pixverse-T2V`         |  ✓   | -                                                    | -                                               | `DEEPINFRA_API_KEY`                      |
| fal                   | `fal-ai/minimax/video-01-live`  |  ✓   | 1 görsel; Seedance referanstan videoya ile en fazla 9    | Seedance referanstan videoya ile en fazla 3 video | `FAL_KEY`                                |
| Google                | `veo-3.1-fast-generate-preview` |  ✓   | 1 görsel                                              | 1 video                                         | `GEMINI_API_KEY`                         |
| MiniMax               | `MiniMax-Hailuo-2.3`            |  ✓   | 1 görsel                                              | -                                               | `MINIMAX_API_KEY` veya MiniMax OAuth       |
| OpenAI                | `sora-2`                        |  ✓   | 1 görsel                                              | 1 video                                         | `OPENAI_API_KEY`                         |
| OpenRouter            | `google/veo-3.1-fast`           |  ✓   | En fazla 4 görsel (ilk/son kare veya referanslar)      | -                                               | `OPENROUTER_API_KEY`                     |
| Qwen                  | `wan2.6-t2v`                    |  ✓   | Evet (uzak URL)                                     | Evet (uzak URL)                                | `QWEN_API_KEY`                           |
| Runway                | `gen4.5`                        |  ✓   | 1 görsel                                              | 1 video                                         | `RUNWAYML_API_SECRET`                    |
| Together              | `Wan-AI/Wan2.2-T2V-A14B`        |  ✓   | Yalnızca `Wan-AI/Wan2.2-I2V-A14B`                        | -                                               | `TOGETHER_API_KEY`                       |
| Vydra                 | `veo3`                          |  ✓   | 1 görsel (`kling`)                                    | -                                               | `VYDRA_API_KEY`                          |
| xAI                   | `grok-imagine-video`            |  ✓   | Classic: 1 ilk kare veya 7 referans; 1.5: 1 kare | Classic: 1 video                                | `XAI_API_KEY`                            |

Bazı sağlayıcılar ek veya alternatif API anahtarı ortam değişkenlerini kabul eder. Ayrıntılar için
tek tek [sağlayıcı sayfalarına](#related) bakın.

Çalışma zamanında kullanılabilir sağlayıcıları, modelleri ve çalışma zamanı
modlarını incelemek için `video_generate action=list` komutunu çalıştırın.

### Yetenek matrisi

`video_generate`, sözleşme testleri ve
paylaşılan canlı tarama tarafından kullanılan açık mod sözleşmesi:

| Sağlayıcı   | `generate` | `imageToVideo` | `videoToVideo` | Bugünkü paylaşılan canlı hatlar                                                                                                                 |
| ---------- | :--------: | :------------: | :------------: | --------------------------------------------------------------------------------------------------------------------------------------- |
| Alibaba    |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; bu sağlayıcı uzak `http(s)` video URL'leri gerektirdiği için `videoToVideo` atlanır                              |
| BytePlus   |     ✓      |       ✓        |       -        | `generate`, `imageToVideo`                                                                                                              |
| ComfyUI    |     ✓      |       ✓        |       -        | Paylaşılan taramada yer almaz; iş akışına özgü kapsam Comfy testlerinde bulunur                                                              |
| DeepInfra  |     ✓      |       -        |       -        | `generate`; yerel DeepInfra video şemaları, Plugin sözleşmesinde metinden videoyadır                                                     |
| fal        |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; `videoToVideo` yalnızca Seedance referanstan videoya kullanılırken                                                  |
| Google     |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; mevcut arabellek destekli Gemini/Veo taraması bu girdiyi kabul etmediği için paylaşılan `videoToVideo` atlanır |
| MiniMax    |     ✓      |       ✓        |       -        | `generate`, `imageToVideo`                                                                                                              |
| OpenAI     |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; bu kuruluş/girdi yolu şu anda sağlayıcı tarafında video düzenleme erişimi gerektirdiği için paylaşılan `videoToVideo` atlanır   |
| OpenRouter |     ✓      |       ✓        |       -        | `generate`, `imageToVideo`                                                                                                              |
| Qwen       |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; bu sağlayıcı uzak `http(s)` video URL'leri gerektirdiği için `videoToVideo` atlanır                              |
| Runway     |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; `videoToVideo` yalnızca seçilen model `runway/gen4_aleph` olduğunda çalışır                                     |
| Together   |     ✓      |       ✓        |       -        | `generate`, `imageToVideo`                                                                                                              |
| Vydra      |     ✓      |       ✓        |       -        | `generate`; paketle birlikte gelen `veo3` yalnızca metin desteklediği ve paketle birlikte gelen `kling` uzak bir görsel URL'si gerektirdiği için paylaşılan `imageToVideo` atlanır           |
| xAI        |     ✓      |       ✓        |       ✓        | Classic tüm modları destekler; Video 1.5 yalnızca görselden videoya destekler; uzak MP4 girdisi `videoToVideo` öğesini paylaşılan taramanın dışında tutar             |

## Araç parametreleri

### Zorunlu

<ParamField path="prompt" type="string" required>
  Oluşturulacak videonun metinsel açıklaması. `action: "generate"` için gereklidir.
</ParamField>

### İçerik girdileri

<ParamField path="image" type="string">Tek referans görsel (yol veya URL).</ParamField>
<ParamField path="images" type="string[]">Birden fazla referans görsel (en fazla 9).</ParamField>
<ParamField path="imageRoles" type="string[]">
Birleştirilmiş görsel listesine paralel, konuma göre isteğe bağlı rol ipuçları.
Standart değerler: `first_frame`, `last_frame`, `reference_image`.
</ParamField>
<ParamField path="video" type="string">Tek referans video (yol veya URL).</ParamField>
<ParamField path="videos" type="string[]">Birden fazla referans video (en fazla 4).</ParamField>
<ParamField path="videoRoles" type="string[]">
Birleştirilmiş video listesine paralel, konuma göre isteğe bağlı rol ipuçları.
Standart değer: `reference_video`.
</ParamField>
<ParamField path="audioRef" type="string">
Tek referans ses (yol veya URL). Sağlayıcı ses girdilerini desteklediğinde
arka plan müziği veya ses referansı için kullanılır.
</ParamField>
<ParamField path="audioRefs" type="string[]">Birden fazla referans ses (en fazla 3).</ParamField>
<ParamField path="audioRoles" type="string[]">
Birleştirilmiş ses listesine paralel, konuma göre isteğe bağlı rol ipuçları.
Standart değer: `reference_audio`.
</ParamField>

<Note>
Rol ipuçları sağlayıcıya değiştirilmeden iletilir. Standart değerler
`VideoGenerationAssetRole` birleşiminden gelir, ancak sağlayıcılar ek
rol dizelerini kabul edebilir. `*Roles` dizileri, karşılık gelen
referans listesinden daha fazla giriş içermemelidir; bir basamaklık hatalar açık bir hata ile başarısız olur.
Bir yuvayı ayarlanmamış bırakmak için boş dize kullanın. xAI için, `reference_images`
oluşturma modunu kullanmak üzere her görsel rolünü `reference_image` olarak
ayarlayın; tek görselli görselden videoya dönüştürme için rolü atlayın veya
`first_frame` kullanın.
</Note>

### Stil denetimleri

<ParamField path="aspectRatio" type="string">
  `1:1`, `16:9`, `9:16`, `adaptive` gibi bir en-boy oranı ipucu veya sağlayıcıya özgü bir değer. OpenClaw, desteklenmeyen değerleri sağlayıcıya göre normalleştirir veya yok sayar.
</ParamField>
<ParamField path="resolution" type="string">`360P`, `480P`, `540P`, `720P`, `768P`, `1080P`, `4K` gibi bir çözünürlük ipucu veya sağlayıcıya özgü bir değer. OpenClaw, desteklenmeyen değerleri sağlayıcıya göre normalleştirir veya yok sayar.</ParamField>
<ParamField path="durationSeconds" type="number">
  Saniye cinsinden hedef süre (sağlayıcının desteklediği en yakın değere yuvarlanır).
</ParamField>
<ParamField path="size" type="string">Sağlayıcı desteklediğinde boyut ipucu.</ParamField>
<ParamField path="audio" type="boolean">
  Desteklendiğinde çıktıda oluşturulan sesi etkinleştirin. `audioRef*` (girdiler) ile aynı değildir.
</ParamField>
<ParamField path="watermark" type="boolean">Desteklendiğinde sağlayıcı filigranını açın veya kapatın.</ParamField>

`adaptive`, sağlayıcıya özgü bir nöbetçi değerdir: yeteneklerinde
`adaptive` bildiren sağlayıcılara değiştirilmeden iletilir (ör. BytePlus
Seedance, giriş görselinin boyutlarından oranı otomatik algılamak için bunu
kullanır). Bunu bildirmeyen sağlayıcılar, değerin elendiğinin görülebilmesi için
araç sonucunda `details.ignoredOverrides` üzerinden değeri gösterir.

### Gelişmiş

<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  `"status"` geçerli oturum görevini döndürür; `"list"` sağlayıcıları inceler.
</ParamField>
<ParamField path="model" type="string">Sağlayıcı/model geçersiz kılma değeri (ör. `runway/gen4.5`).</ParamField>
<ParamField path="filename" type="string">Çıktı dosya adı ipucu.</ParamField>
<ParamField path="timeoutMs" type="number">Milisaniye cinsinden isteğe bağlı sağlayıcı işlemi zaman aşımı. Atlandığında OpenClaw, yapılandırılmışsa `agents.defaults.mediaModels.video.timeoutMs` değerini; aksi takdirde varsa plugin yazarı tarafından belirlenen sağlayıcı varsayılanını kullanır.</ParamField>
<ParamField path="providerOptions" type="object">
  JSON nesnesi olarak sağlayıcıya özgü seçenekler (ör. `{"seed": 42, "draft": true}`).
  Türü belirlenmiş bir şema bildiren sağlayıcılar anahtarları ve türleri doğrular;
  bilinmeyen anahtarlar veya uyuşmazlıklar, geri dönüş sırasında adayın atlanmasına
  neden olur. Bildirilmiş şeması olmayan sağlayıcılar seçenekleri değiştirilmeden alır.
  Her sağlayıcının neleri kabul ettiğini görmek için `video_generate action=list` komutunu çalıştırın.
</ParamField>

<Note>
Tüm sağlayıcılar tüm parametreleri desteklemez. OpenClaw süreyi sağlayıcının
desteklediği en yakın değere normalleştirir ve geri dönüş sağlayıcısı farklı bir
denetim yüzeyi sunduğunda boyuttan en-boy oranına gibi çevrilmiş geometri
ipuçlarını yeniden eşler. Gerçekten desteklenmeyen geçersiz kılmalar, mümkün olan
en iyi çaba temelinde yok sayılır ve araç sonucunda uyarı olarak bildirilir. Kesin
yetenek sınırları (örneğin çok fazla referans girdisi) gönderimden önce başarısız
olur. Araç sonuçları uygulanan ayarları bildirir; `details.normalization`,
istenen değerden uygulanan değere yapılan tüm dönüşümleri yakalar.
</Note>

Referans girdileri çalışma zamanı modunu seçer:

- Referans medya yok -> `generate`
- Herhangi bir görsel referansı -> `imageToVideo`
- Herhangi bir video referansı -> `videoToVideo`
- Referans ses girdileri çözümlenen modu **değiştirmez**; görsel/video
  referanslarının seçtiği modun üzerine uygulanır ve yalnızca
  `maxInputAudios` bildiren sağlayıcılarla çalışır.

Karışık görsel ve video referansları, kararlı ve ortak bir yetenek yüzeyi değildir.
Her istek için tek bir referans türü tercih edin.

#### Geri dönüş ve türü belirlenmiş seçenekler

Bazı yetenek denetimleri araç sınırında değil, geri dönüş katmanında uygulanır;
bu nedenle birincil sağlayıcının sınırlarını aşan bir istek yine de yetenekli bir
geri dönüş sağlayıcısında çalışabilir:

- İstek ses referansları içerdiğinde, `maxInputAudios` (veya `0`) bildirmeyen
  etkin aday atlanır; sonraki aday denenir. Aynı koruma, görsel ve video referansı
  sayılarında `maxInputImages`/`maxInputVideos` değerlerine göre uygulanır.
- Etkin adayın `maxDurationSeconds` değeri istenen `durationSeconds`
  değerinden düşükse ve bildirilmiş bir `supportedDurationSeconds` listesi yoksa -> atlanır.
- İstek `providerOptions` içeriyorsa ve etkin aday açıkça türü belirlenmiş
  bir `providerOptions` şeması bildiriyorsa, sağlanan anahtarlar şemada bulunmadığında
  veya değer türleri eşleşmediğinde -> atlanır. Bildirilmiş şeması olmayan sağlayıcılar
  seçenekleri değiştirilmeden alır (geriye dönük uyumlu doğrudan aktarım).
  Bir sağlayıcı boş bir şema (`capabilities.providerOptions: {}`) bildirerek tüm sağlayıcı
  seçeneklerini devre dışı bırakabilir; bu, tür uyuşmazlığıyla aynı şekilde
  atlanmasına neden olur.

Bir istekteki ilk atlama nedeni `warn` düzeyinde günlüğe kaydedilir; böylece
operatörler birincil sağlayıcılarının ne zaman es geçildiğini görür. Uzun geri dönüş
zincirlerini sessiz tutmak için sonraki atlamalar `debug` düzeyinde günlüğe
kaydedilir. Her aday atlanırsa toplu hata, her birinin atlanma nedenini içerir.

## Eylemler

| Eylem     | Yaptığı işlem                                                                                             |
| ---------- | -------------------------------------------------------------------------------------------------------- |
| `generate` | Varsayılan. Verilen istemden ve isteğe bağlı referans girdilerinden bir video oluşturur.                             |
| `status`   | Başka bir oluşturma başlatmadan, geçerli oturum için devam eden video görevinin durumunu denetler. |
| `list`     | Kullanılabilir sağlayıcıları, modelleri ve yeteneklerini gösterir.                                                |

## Model seçimi

OpenClaw modeli şu sırayla çözümler:

1. **`model` araç parametresi** - çağrıda agent bir tane belirtirse.
2. Yapılandırmadaki **`videoGenerationModel.primary`**.
3. Sırayla **`videoGenerationModel.fallbacks`**.
4. **Otomatik algılama** - geçerli kimlik doğrulaması olan sağlayıcılar; önce
   geçerli varsayılan sağlayıcı, ardından kalan sağlayıcılar alfabetik sırayla.

Bir sağlayıcı başarısız olursa sonraki aday otomatik olarak denenir. Tüm
adaylar başarısız olursa hata, her denemeden ayrıntıları içerir.

Kimliği doğrulanmış sağlayıcılar arasında otomatik geri dönüş her zaman etkindir.
Çağrı başına `model` değeri belirleyici olmaya devam eder.

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
        fallbacks: ["runway/gen4.5", "qwen/wan2.6-t2v"],
        timeoutMs: 180000, // isteğe bağlı, araç başına sağlayıcı isteği zaman aşımını geçersiz kılma
      },
    },
  },
}
```

## Sağlayıcı notları

<AccordionGroup>
  <Accordion title="Alibaba">
    DashScope / Model Studio eşzamansız uç noktasını kullanır. Referans görseller
    ve videolar uzak `http(s)` URL'leri olmalıdır.
  </Accordion>
  <Accordion title="BytePlus (paketle birlikte)">
    Sağlayıcı kimliği: `byteplus`.

    Modeller: `seedance-1-0-pro-250528` (varsayılan),
    `seedance-1-5-pro-251215`.

    Birleşik `content[]` API'sini kullanır. En fazla 2 giriş görselini
    destekler (`first_frame` + `last_frame`). Görselleri konum sırasına
    göre iletin veya her görselin `role` değerini açıkça ayarlayın.

    Desteklenen `providerOptions` anahtarları: `seed` (sayı), `draft` (boolean -
    480p'yi zorunlu kılar), `camera_fixed` (boolean).

  </Accordion>
  <Accordion title="BytePlus Seedance 1.5 plugin'i">
    [`@openclaw/byteplus-modelark`](https://www.npmjs.com/package/@openclaw/byteplus-modelark)
    plugin'ini gerektirir (harici, paketle birlikte gelmez). Sağlayıcı kimliği: `byteplus-seedance15`. Model:
    `seedance-1-5-pro-251215`.

    Birleşik `content[]` API'sini kullanır. En fazla 2 giriş görselini
    destekler (`first_frame` + `last_frame`). Tüm girdiler uzak `https://`
    URL'leri olmalıdır. Her görselde `role: "first_frame"` / `"last_frame"`
    değerini ayarlayın veya görselleri konum sırasına göre iletin.

    `aspectRatio: "adaptive"`, oranı giriş görselinden otomatik algılar.
    `audio: true`, `generate_audio` değerine eşlenir. `providerOptions.seed`
    (sayı) iletilir.

  </Accordion>
  <Accordion title="BytePlus Seedance 2.0">
    [`@openclaw/byteplus-modelark`](https://www.npmjs.com/package/@openclaw/byteplus-modelark)
    plugin'ini gerektirir (harici, paketle birlikte gelmez). Sağlayıcı kimliği: `byteplus-seedance2`. Modeller:
    `dreamina-seedance-2-0-260128`,
    `dreamina-seedance-2-0-fast-260128`.

    Birleşik `content[]` API'sini kullanır. En fazla 9 referans görseli,
    3 referans videoyu ve 3 referans sesi destekler. Tüm girdiler uzak
    `https://` URL'leri olmalıdır. Her varlıkta `role` değerini ayarlayın - desteklenen değerler:
    `"first_frame"`, `"last_frame"`, `"reference_image"`,
    `"reference_video"`, `"reference_audio"`.

    `aspectRatio: "adaptive"`, oranı giriş görselinden otomatik algılar.
    `audio: true`, `generate_audio` değerine eşlenir. `providerOptions.seed`
    (sayı) iletilir.

  </Accordion>
  <Accordion title="ComfyUI">
    İş akışı odaklı yerel veya bulut yürütmesi. Yapılandırılmış grafik
    aracılığıyla metinden videoya ve görüntüden videoya dönüştürmeyi destekler.
  </Accordion>
  <Accordion title="fal">
    Uzun süren işler için kuyruk destekli bir akış kullanır. OpenClaw, devam eden
    bir fal kuyruk işini zaman aşımına uğramış saymadan önce varsayılan olarak 20
    dakikaya kadar bekler. Çoğu fal video modeli
    tek bir görüntü referansı kabul eder. Seedance 2.0 referanstan videoya
    modelleri, toplamda en fazla 12 referans dosyası olmak üzere en çok 9 görüntü,
    3 video ve 3 ses referansı kabul eder.
  </Accordion>
  <Accordion title="Google (Gemini / Veo)">
    Bir görüntü veya bir video referansını destekler. Gemini API yolu, mevcut Veo
    video üretiminde `generateAudio` parametresini reddettiği için oluşturulmuş
    ses istekleri bir uyarıyla yok sayılır.
  </Accordion>
  <Accordion title="MiniMax">
    Yalnızca tek görüntü referansı. MiniMax, `768P` ve `1080P`
    çözünürlüklerini kabul eder; `720P` gibi istekler gönderilmeden önce
    desteklenen en yakın değere normalleştirilir.
  </Accordion>
  <Accordion title="OpenAI">
    Yalnızca `size` geçersiz kılması iletilir. Diğer stil geçersiz
    kılmaları (`aspectRatio`, `resolution`, `audio`, `watermark`)
    bir uyarıyla yok sayılır.
  </Accordion>
  <Accordion title="OpenRouter">
    OpenRouter'ın eşzamansız `/videos` API'sini kullanır. OpenClaw işi
    gönderir, `polling_url` durumunu yoklar ve `unsigned_urls` ya da
    belgelenmiş iş içeriği uç noktasını indirir. Paketle gelen varsayılan
    `google/veo-3.1-fast`; 4/6/8 saniyelik süreleri, `720P`/`1080P`
    çözünürlüklerini ve `16:9`/`9:16` en-boy oranlarını bildirir.
  </Accordion>
  <Accordion title="Qwen">
    Alibaba ile aynı DashScope arka ucunu kullanır. Referans girdileri uzak
    `http(s)` URL'leri olmalıdır; yerel dosyalar en başta reddedilir.
  </Accordion>
  <Accordion title="Runway">
    Veri URI'leri aracılığıyla yerel dosyaları destekler. Videodan videoya
    dönüştürme `runway/gen4_aleph` gerektirir. Yalnızca metin içeren çalıştırmalar,
    `16:9` ve `9:16` en-boy oranlarını sunar.
  </Accordion>
  <Accordion title="Together">
    Yalnızca tek görüntü referansı.
  </Accordion>
  <Accordion title="Vydra">
    Kimlik doğrulamasını düşüren yönlendirmelerden kaçınmak için doğrudan
    `https://www.vydra.ai/api/v1` kullanır. `veo3` yalnızca metinden videoya olarak
    paketle gelir; `kling` uzak bir görüntü URL'si gerektirir.
  </Accordion>
  <Accordion title="xAI">
    Varsayılan `grok-imagine-video` modeli; metinden videoya, tek ilk kareli
    görüntüden videoya, xAI `reference_images` aracılığıyla en fazla 7
    `reference_image` girdisi ve uzak video düzenleme/uzatma akışlarını destekler.
    Üretim varsayılan olarak `480P` değerini kullanır; `aspectRatio`
    belirtilmediğinde tek görüntülü görüntüden videoya dönüştürme, kaynak oranını
    devralır. Video düzenleme/uzatma, girdi geometrisini devralır ve en-boy oranı
    veya çözünürlük geçersiz kılmalarını kabul etmez. Uzatma 2-10 saniye kabul eder.

    `grok-imagine-video-1.5` yalnızca görüntüden videoya dönüştürme içindir: tam olarak
    bir görüntü sağlayın. 1-15 saniyeyi ve `480P`, `720P`
    veya `1080P` değerlerini destekler; varsayılan değer
    `480P` olur. Kaynak görüntü oranını devralmak için
    `aspectRatio` değerini belirtmeyin. Önizleme ve tarihli 1.5 tanımlayıcıları
    aynı doğrulamadan geçirilir ve değiştirilmeden iletilir.

  </Accordion>
</AccordionGroup>

## Sağlayıcı yetenek modları

Paylaşılan video üretim sözleşmesi, yalnızca düz toplu sınırlar yerine
moda özgü yetenekleri destekler. Yeni sağlayıcı uygulamaları
açık mod bloklarını tercih etmelidir:

```typescript
capabilities: {
  generate: {
    maxVideos: 1,
    maxDurationSeconds: 10,
    supportsResolution: true,
  },
  imageToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputImages: 1,
    maxInputImagesByModel: { "provider/reference-to-video": 9 },
    maxDurationSeconds: 5,
  },
  videoToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputVideos: 1,
    maxDurationSeconds: 5,
  },
}
```

`maxInputImages` ve `maxInputVideos` gibi düz toplu alanlar, dönüştürme
modu desteğini bildirmek için **yeterli değildir**. Sağlayıcılar;
canlı testlerin, sözleşme testlerinin ve paylaşılan `video_generate` aracının
mod desteğini belirleyici biçimde doğrulayabilmesi için `generate`,
`imageToVideo` ve `videoToVideo` değerlerini açıkça bildirmelidir.

Bir sağlayıcıdaki tek bir model, diğerlerinden daha geniş referans girdisi
desteğine sahipse mod genelindeki sınırı yükseltmek yerine `maxInputImagesByModel`,
`maxInputVideosByModel` veya `maxInputAudiosByModel` kullanın.

## Canlı testler

Paylaşılan paketli sağlayıcılar için isteğe bağlı canlı kapsam:

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts
```

Repo sarmalayıcısı:

```bash
pnpm test:live:media video
```

Bu canlı dosya, varsayılan olarak önceden dışa aktarılmış sağlayıcı ortam
değişkenlerini saklanan kimlik doğrulama profillerinden önce kullanır ve
varsayılan olarak sürüm için güvenli bir hızlı test çalıştırır:

- `generate` taramadaki FAL dışındaki her sağlayıcı için.
- Bir saniyelik ıstakoz istemi.
- Sağlayıcı başına işlem sınırı:
  `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` (varsayılan olarak `180000`).

Sağlayıcı tarafındaki kuyruk gecikmesi sürüm süresine baskın gelebileceğinden
FAL isteğe bağlıdır:

```bash
pnpm test:live:media video --video-providers fal
```

Paylaşılan taramanın yerel medya ile güvenle çalıştırabildiği bildirilmiş
dönüştürme modlarını da çalıştırmak için `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1` ayarlayın:

- `capabilities.imageToVideo.enabled` olduğunda `imageToVideo`.
- `capabilities.videoToVideo.enabled` olduğunda ve sağlayıcı/model, paylaşılan
  taramada arabellek destekli yerel video girdisini kabul ettiğinde
  `videoToVideo`.

Günümüzde paylaşılan `videoToVideo` canlı hattı, yalnızca
`runway/gen4_aleph` seçildiğinde `runway` kapsamını sağlar.

## Yapılandırma

OpenClaw yapılandırmanızda varsayılan video üretim modelini ayarlayın:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-r2v-flash"],
      },
    },
  },
}
```

Veya CLI aracılığıyla:

```bash
openclaw config set agents.defaults.mediaModels.video.primary "qwen/wan2.6-t2v"
```

## İlgili

- [Alibaba Model Studio](/tr/providers/alibaba)
- [Arka plan görevleri](/tr/automation/tasks) - eşzamansız video üretimi için görev takibi
- [BytePlus](/tr/concepts/model-providers#byteplus-international)
- [ComfyUI](/tr/providers/comfy)
- [Yapılandırma başvurusu](/tr/gateway/config-agents#agent-defaults)
- [fal](/tr/providers/fal)
- [Google (Gemini)](/tr/providers/google)
- [MiniMax](/tr/providers/minimax)
- [Modeller](/tr/concepts/models)
- [OpenAI](/tr/providers/openai)
- [Qwen](/tr/providers/qwen)
- [Runway](/tr/providers/runway)
- [Together AI](/tr/providers/together)
- [Araçlara genel bakış](/tr/tools)
- [Vydra](/tr/providers/vydra)
- [xAI](/tr/providers/xai)
