---
read_when:
    - Control UI'da gününüzün Dayflow tarzı bir zaman çizelgesini istiyorsunuz
    - Paketle birlikte gelen Logbook pluginini etkinleştiriyor veya yapılandırıyorsunuz
    - Ekran etkinliğine dayalı günlük toplantı özetleri veya gün içi anımsatma istiyorsunuz
summary: Periyodik ekran görüntülerinden oluşturulan isteğe bağlı otomatik çalışma günlüğü
title: Günlük Plugin'i
x-i18n:
    generated_at: "2026-07-27T00:06:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 19197e580421dfe81f82f8599578e4c68a15004813bb2b6c3de761c14f426b08
    source_path: plugins/logbook.md
    workflow: 16
---

Logbook Plugin'i, ekran etkinliğini otomatik bir çalışma günlüğüne dönüştürür. Eşleştirilmiş bir Node'dan düzenli aralıklarla ekran görüntüleri alır, bunları zaman damgalı gözlemler hâlinde özetler ve [Kontrol Arayüzü](/tr/web/control-ui) içinde zaman çizelgesi kartları oluşturur. Ayrıca günlük durum toplantısı notları oluşturabilir ve izlenen bir gün hakkındaki soruları yanıtlayabilir.

OpenClaw'a ait durum, Gateway üzerinde `<state-dir>/logbook/` altında kalır ancak model işleme işlemi mutlaka yerel değildir. Örneklenen ekran görüntüleri yapılandırılmış görüntü rotasına; gözlemler ve zaman çizelgesi metni ise varsayılan ajan modeline gider. Ekran içeriğinin ve bundan türetilen etkinlik metninin makinede kalması gerekiyorsa her iki aşama için de yerel model rotalarını kullanın.

Logbook paketle birlikte gelir ve varsayılan olarak devre dışıdır. `captureEnabled` varsayılan olarak `true` olduğundan Plugin'in etkinleştirilmesi, Gateway'in ekran yakalamayı kabul etmesini sağlar.

## Başlamadan önce

Gereksinimler:

- `screen.snapshot` veya `logbook.snapshot` sunan bağlı bir Node. macOS uygulama Node'u Ekran Kaydı iznine ihtiyaç duyar. Başsız bir macOS Node ana bilgisayarı (`openclaw node host run`), sistemdeki `screencapture` aracı tarafından desteklenen ve Plugin'in sağladığı `logbook.snapshot` komutunu alır.
- Paketle gelen Codex Plugin'inin etkinleştirilmiş ve kimliği doğrulanmış olması. Codex şu anda Logbook'un ihtiyaç duyduğu yapılandırılmış görüntü çıkarma sözleşmesini sağlar. `openclaw models auth login --provider openai` ile oturum açın; diğer kimlik doğrulama yolları için [Codex donanımına](/tr/plugins/codex-harness) bakın.
- Çalışan bir varsayılan ajan modeli. Logbook, görüntü geçişinden sonra kartları, durum toplantısı notlarını ve günlük soru-cevapları sentezlemek için bunu kullanır.

## Hızlı başlangıç

Codex ve Logbook Plugin'lerini etkinleştirin:

```bash
openclaw plugins enable codex
openclaw plugins enable logbook
```

Belirlenimci başlatma için açık bir görüntü modeli yapılandırın:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          visionModel: "codex/gpt-5.6-sol",
        },
      },
    },
  },
}
```

`plugins.allow` kullanıyorsanız hem `codex` hem de `logbook` değerlerini ekleyin. Plugin yapılandırmasını değiştirdikten sonra Gateway'i yeniden başlatın, ardından kayıtları inceleyip gösterge panelini açın:

```bash
openclaw gateway restart
openclaw plugins inspect logbook --runtime --json
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw dashboard
```

Node açıklaması `screen.snapshot` veya `logbook.snapshot` içermelidir. Başsız Node'lar `logbook.snapshot` değerini yalnızca Plugin etkinleştirildikten sonra duyurur. Komut eksikse [Node sorun giderme](/tr/nodes/troubleshooting) sayfasına bakın.

Logbook sekmesi yalnızca etkinleştirilmiş bir Plugin ve `operator.write` Control UI oturumu için görünür. Durum satırında hata olmadan **Yakalanıyor** gösterilmelidir. Analiz penceresi kapandığında bir zaman çizelgesi kartı görünür veya etkinlik yakalandıktan sonra **Şimdi analiz et** seçeneğini belirleyebilirsiniz.

## Nasıl çalışır?

1. **Yakalama**: Logbook, her `captureIntervalSeconds` aralığında (varsayılan 30 sn.) seçilen Node'un yakalama komutunu çağırır ve ölçeklendirilmiş bir JPEG karesi depolar. Birbirinin aynısı olan ardışık kareler boşta olarak işaretlenir ve analizin dışında bırakılır.
2. **Gözlemleme**: Bir analiz penceresi (varsayılan 15 dakika) dolduğunda Plugin, en fazla 16 etkin kareyi örnekleyip görüntü modeline gönderir. Model, zaman damgalı etkinlik gözlemleri döndürür ("VS Code: store.ts düzenleniyor, bir tür hatası düzeltiliyor"). İki dakikadan uzun bir yakalama boşluğu veya yerel gece yarısı da geçerli pencereyi kapatır.
3. **Sentezleme**: Gözlemler ve mevcut kartların son 45 dakikası; başlık, özet, kategori, ana uygulama ve kısa süreli dikkat dağıtıcıları içeren zaman çizelgesi kartları (her biri 10-60 dakika) hâlinde yeniden düzenlenir.
4. **Temizleme**: `retentionDays` günden (varsayılan 14) eski kareler silinir. Kartlar, gözlemler ve önbelleğe alınmış durum toplantısı notları korunur.

Gün sınırları ve zaman çizelgesi saatleri, tarayıcının saat dilimini değil Gateway'in yerel saat dilimini kullanır. Kareler ve SQLite zaman çizelgesi veritabanı `<state-dir>/logbook/` altında bulunur.

## Model ve veri akışı

Logbook iki ayrı model rotası kullanır:

| Aşama            | Gönderilen veriler                                                 | Model rotası                                                       |
| ---------------- | --------------------------------------------------------- | ----------------------------------------------------------------- |
| Gözlemleme          | En fazla 16 örneklenmiş JPEG karesi ve bunların yakalanma zamanları     | `visionModel` veya uyumlu bir ödünç `tools.media` Codex girdisi |
| Kartları sentezleme | Zaman damgalı gözlemler ve son zaman çizelgesi kartları        | Plugin LLM çalışma zamanı üzerinden varsayılan ajan modeli                |
| Durum toplantısı oluşturma | Seçilen ve önceki güne ait kartlar               | Plugin LLM çalışma zamanı üzerinden varsayılan ajan modeli                |
| Gününüze soru sorun     | Soru, seçilen güne ait kartlar ve son gözlemler | Plugin LLM çalışma zamanı üzerinden varsayılan ajan modeli                |

SQLite veritabanının tamamı hiçbir modele gönderilmez. Ham ekran görüntüleri yalnızca gözlem aşamasına gider; kart sentezleme, durum toplantısı ve soru-cevap işlemleri türetilmiş metni alır.

## Yapılandırma

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          captureEnabled: true,
          captureIntervalSeconds: 30,
          analysisIntervalMinutes: 15,
          nodeId: "my-mac",
          screenIndex: 0,
          maxWidth: 1440,
          visionModel: "codex/gpt-5.6-sol",
          retentionDays: 14,
        },
      },
    },
  },
}
```

Tüm Logbook yapılandırma anahtarları isteğe bağlıdır. Sayısal değerler tam sayıya yuvarlanır ve desteklenen aralıkla sınırlandırılır.

| Anahtar                       | Varsayılan | Aralık veya değerler         | Davranış                                                                                     |
| ------------------------- | ------- | ----------------------- | -------------------------------------------------------------------------------------------- |
| `captureEnabled`          | `true`  | Boole                  | Yeni anlık görüntüler için kalıcı ana anahtar; `false` olduğunda zaman çizelgesi kullanılabilir durumda kalır      |
| `captureIntervalSeconds`  | `30`    | `5`-`600`               | Yakalama denemeleri arasındaki gecikme                                                               |
| `analysisIntervalMinutes` | `15`    | `3`-`120`               | Hedef gözlem penceresi; boşluklar ve gece yarısı pencereyi daha erken kapatabilir                            |
| `nodeId`                  | ayarlanmamış   | Node kimliği veya görünen ad | Yakalamayı bağlı tek bir Node'a sabitler; eşleştirme büyük/küçük harfe duyarlı değildir                             |
| `screenIndex`             | `0`     | `0`-`16`                | Sıfır tabanlı ekran dizini                                                                     |
| `maxWidth`                | `1440`  | `480`-`3840`            | İstenen yakalama boyutu sınırı; başsız macOS bunu en büyük boyuta uygular               |
| `visionModel`             | ayarlanmamış   | `provider/model`        | Açık yapılandırılmış rota; hatalı biçimlendirilmiş başvurular analizi duraklatır, desteklenmeyen sağlayıcılar toplu işlemleri başarısız kılar |
| `retentionDays`           | `14`    | `1`-`365`               | Eski kareleri siler; kartlar, gözlemler ve durum toplantısı notları kalır                                 |

`nodeId` olmadan Logbook, `screen.snapshot` sunan bağlı bir uygulama Node'unu tercih eder, ardından `logbook.snapshot` sunan başsız bir Node'a geri döner. Sabitlenmemiş bir kurulumda başarısız olan Node, diğer uygun Node'ların arkasına alınır. Gösterge panelindeki duraklatma anahtarı yalnızca oturum boyunca geçerlidir ve Gateway yeniden başlatıldığında sıfırlanır; kalıcı olarak durdurmak için `captureEnabled: false` kullanın.

### Görüntü modeli seçimi

Logbook gözlem modelini şu sırayla çözümler:

1. `plugins.entries.logbook.config.visionModel`
2. `tools.media.models` altındaki görüntü destekli ilk Codex girdisi

Diğer medya sağlayıcıları, Logbook'un ihtiyaç duyduğu yapılandırılmış çıkarma sözleşmesini şu anda sunmadıkları için atlanır. `tools.media.image.enabled: false` değerinin ayarlanması ödünç alınan medya varsayılanlarını devre dışı bırakır ancak açık bir Logbook `visionModel` değeri yine de uygulanır.

## Gösterge paneli sekmesi

- **Zaman çizelgesi**: Kategori renkleri, ana uygulama, dikkat dağıtıcı etiketleri ve bir anlık görüntü ana karesiyle her etkinlik için genişletilebilir kartlar.
- **Güne genel bakış**: Odaklanma oranı, kategori dağılımı ve en çok kullanılan uygulamalar.
- **Günlük durum toplantısı**: Dün ile bugünü yapıştırmaya hazır bir güncellemeye dönüştürür.
- **Gününüze soru sorun**: İzlenen zaman çizelgesinden yanıtlanan doğal dil soruları ("Gateway PR'ını ne zaman inceledim?").
- **Şimdi analiz et**: Analiz aralığını beklemek yerine geçerli yakalama penceresini hemen kapatır.

## Gateway yöntemleri

Logbook şu Gateway RPC yöntemlerini kaydeder:

| Yöntem                | Parametreler               | Kapsam            | Sonuç                                                                   |
| --------------------- | ------------------------ | ---------------- | ------------------------------------------------------------------------ |
| `logbook.status`      | yok                     | `operator.read`  | Yakalama, analiz, model, Node, Gateway günü ve Gateway saat dilimi durumu |
| `logbook.days`        | yok                     | `operator.read`  | Zaman çizelgesi kartı sayıları ve kart zaman sınırları bulunan günler                      |
| `logbook.timeline`    | `{ day?: "YYYY-MM-DD" }` | `operator.read`  | Türetilmiş kartlar ve gün istatistikleri; varsayılan olarak Gateway'in geçerli gününü kullanır  |
| `logbook.frames`      | `{ startMs, endMs }`     | `operator.write` | İstenen epoch-milisaniye aralığındaki kare meta verileri                  |
| `logbook.frame`       | `{ frameId }`            | `operator.write` | Base64 biçiminde bir ham JPEG karesi                                             |
| `logbook.standup`     | `{ day?, refresh? }`     | `operator.write` | Bir gün için önbelleğe alınmış veya yeniden oluşturulmuş durum toplantısı metni                             |
| `logbook.ask`         | `{ day?, question }`     | `operator.write` | Bir gün için zaman çizelgesine dayalı yanıt                                       |
| `logbook.capture.set` | `{ paused }`             | `operator.write` | Yalnızca oturuma ait duraklatma durumu ve güncellenmiş durum                              |
| `logbook.analyze.now` | yok                     | `operator.write` | Bekleyen analizi başlatır veya neden başlatılamadığını döndürür          |

Okuma yöntemleri operasyonel durumu veya türetilmiş metni döndürür. Ham ekran görüntüsü pikselleri, model harcaması gerektiren eylemler ve çalışma zamanı değişiklikleri `operator.write` gerektirir. Control UI sekmesi, bu eylemleri ve ham kare önizlemelerini sunduğu için ayrıca `operator.write` gerektirir; salt okunur bir istemci türetilmiş metin yöntemlerini doğrudan çağırmaya devam edebilir.

## Gizlilik notları

- Anlık görüntüler, gizli bilgiler dâhil ekrandaki her şeyi içerebilir. Kareler, yapılandırılmış gözlem modeline örneklenmiş girdi olarak gönderilmesi dışında hiçbir zaman makineden ayrılmaz.
- Gözlemler, son kartlar ve sorular; kart sentezi, durum toplantısı oluşturma veya soru-cevap sırasında varsayılan ajan modeli aracılığıyla makineden ayrılabilir. Sağlayıcının veri işleme politikasını her iki model rotasına da uygulayın.
- Tamamen yerel bir işlem hattına ihtiyaç duyduğunuzda hem yapılandırılmış gözlem modeli hem de varsayılan ajan modeli için yerel rotaları kullanın.
- Kareler, zaman çizelgesi veritabanı ve geçici yakalamalar yalnızca sahibin erişebileceği dosya izinleriyle yazılır.
- `gateway.nodes.commands.deny` öğesine `screen.snapshot` eklemek, ekran yakalama acil durdurma anahtarıdır: hem uygulama Node'u yakalamasını hem de Logbook'un kendi `logbook.snapshot` komutunu engeller.
- `tools.media.image.enabled: false` ayarlandığında Logbook'un analiz için medya görüntü modellerini ödünç alması da durdurulur; bu durumda yalnızca Plugin yapılandırmasında açıkça belirtilen bir `visionModel` kullanılır.

## Sorun giderme

### Logbook sekmesi görünmüyor

Üç geçidi de kontrol edin:

1. `openclaw plugins list --enabled`, `logbook` içerir.
2. Plugin veya izin verilenler listesi değişikliğinden sonra Gateway yeniden başlatıldı.
3. Control UI bağlantısında `operator.write` bulunur; salt okunur oturumlar etkileşimli sekme tanımlayıcısını almaz.

`plugins.allow` ayarlanmışsa önerilen yapılandırma için hem `logbook` hem de `codex` içermelidir.

### Yakalama bir hata bildiriyor

```bash
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw logs --follow
```

- Node'un `screen.snapshot` veya `logbook.snapshot` sunduğunu doğrulayın.
- Yakalama Mac'inde Screen Recording izni verin.
- `nodeId` yapılandırılmışsa Node kimliği veya görünen adıyla eşleştiğini doğrulayın.
- `gateway.nodes.commands.deny` değerinin `screen.snapshot` içermediğini kontrol edin.

Art arda üç hatadan sonra Logbook, on yakalama döngüsü boyunca geri çekilir ve ardından yeniden dener. Sabitlenmemiş bir kurulum, uygun başka bir Node'a geçebilir.

### Yakalamalar başarılı ancak kartlar görünmüyor

- **Model eksik** durumu, uyumlu bir yapılandırılmış görsel rota bulunamadığı anlamına gelir. Codex Plugin'ini etkinleştirip kimliğini doğrulayın veya geçerli ve açık bir `visionModel` ayarlayın. Model eksikken yakalanan kareler beklemede kalır ve yapılandırma düzeltildikten sonra analiz edilebilir.
- `analysisIntervalMinutes` için bekleyin veya etkinlik yakalandıktan sonra **Şimdi analiz et** seçeneğini belirleyin.
- Art arda gelen aynı kareler, boşta olunduğuna dair kanıttır ve analiz gruplarına dâhil edilmez. Test etmeden önce görünür ekranı değiştirin.
- En son grup bir hata gösteriyorsa model veya kimlik doğrulama sorununu düzeltin ve **Şimdi analiz et** seçeneğini belirleyin. Model için tekrar tekrar harcama yapılmasını önlemek amacıyla başarısız gruplar yalnızca bu açık eylemle yeniden denenir.

## İlgili

- [Plugin'leri yönetme](/tr/plugins/manage-plugins)
- [Codex çalışma düzeneği](/tr/plugins/codex-harness)
- [Medya anlama](/tr/nodes/media-understanding)
- [Node'lar](/tr/nodes)
- [Node sorunlarını giderme](/tr/nodes/troubleshooting)
- [Control UI](/tr/web/control-ui)
