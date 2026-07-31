---
read_when:
    - OpenClaw günlük kaydı hakkında başlangıç düzeyine uygun bir genel bakışa ihtiyacınız var
    - Günlük düzeylerini, biçimlerini veya gizlemeyi yapılandırmak istiyorsunuz
    - Sorun gideriyorsunuz ve günlükleri hızlıca bulmanız gerekiyor
summary: Dosya günlükleri, konsol çıktısı, CLI ile günlük takibi ve Kontrol Arayüzü Günlükler sekmesi
title: Günlük Kaydı
x-i18n:
    generated_at: "2026-07-26T23:24:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2c9303c4bc9c0797ca9c5775a281dce95229661b61d710425b2f7bec182b2e75
    source_path: logging.md
    workflow: 16
---

OpenClaw'ın iki ana günlük yüzeyi vardır:

- Gateway tarafından yazılan **dosya günlükleri** (JSON satırları).
- Gateway'in çalıştığı terminaldeki **konsol çıktısı**.

Control UI'daki **Günlükler** sekmesi, Gateway dosya günlüğünü canlı olarak izler. Bu sayfa günlüklerin
nerede bulunduğunu, nasıl okunacağını ve günlük düzeyleri ile biçimlerinin nasıl yapılandırılacağını açıklar.

## Günlüklerin bulunduğu yer

Gateway, varsayılan olarak her gün için dönüşümlü bir günlük dosyası yazar. Varsayılan profil
geçmişten gelen yolu korur:

`/tmp/openclaw/openclaw-YYYY-MM-DD.log`

Adlandırılmış profiller, aynı dizinde profil niteleyicili bir dosya adı kullanır:

`/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`

Dosya adındaki profil bölümü küçük harflidir ve yalnızca harf, rakam ve
tire içerebilir. Basit küçük harfli adlar okunabilir kalır; bu nedenle `--dev` kısaltması
`openclaw-dev-YYYY-MM-DD.log` dosyasına yazar. Büyük/küçük harf, alt çizgi ve değişmez tireler,
farklı profil adlarının hiçbir zaman aynı günlük dosyasını paylaşmaması için
tersine çevrilebilir bir tire kaçışı kullanır. Doğrudan ortam üzerinden ayarlanan aşırı büyük değerler,
dosya sistemi dosya adı sınırları içinde kalmak için sınırlı bir karma son eki kullanır. Açıkça belirtilen bir `logging.file`,
bu varsayılanları geçersiz kılar.

Tarih, Gateway ana makinesinin yerel saat dilimini kullanır. `/tmp/openclaw` güvenli
veya kullanılabilir olmadığında (ve Windows'ta her zaman), OpenClaw bunun yerine işletim sistemi geçici dizini altında
kullanıcı kapsamlı bir `openclaw-<uid>` dizini kullanır. Tarihli günlük dosyaları
24 saat sonra temizlenir.

Bir sonraki yazma işlemi `logging.maxFileBytes` değerini
(varsayılan: 100 MB) aşacaksa her dosya döndürülür. OpenClaw, etkin dosyanın yanında
`openclaw-YYYY-MM-DD.1.log` veya `openclaw-dev-YYYY-MM-DD.1.log` gibi en fazla beş numaralı arşiv
tutar ve tanılamaları engellemek yerine yeni bir etkin günlüğe
yazmayı sürdürür.

`~/.openclaw/openclaw.json` içindeki yolu geçersiz kılabilirsiniz:

```json
{
  "logging": {
    "file": "/path/to/openclaw.log"
  }
}
```

## Günlükleri okuma

### CLI: canlı izleme (önerilir)

Gateway günlük dosyasını RPC üzerinden canlı izleyin:

```bash
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
```

Kök profil seçici, yerel RPC kullanılamadığında CLI geri dönüş okumaları da dahil olmak üzere
Gateway'in kullandığı profil özelindeki aynı dosyayı çözümler.

Seçenekler:

| Bayrak              | Varsayılan | Davranış                                                                              |
| ------------------- | ---------- | ------------------------------------------------------------------------------------- |
| `--follow`          | kapalı     | İzlemeyi sürdürür; bağlantı kesildiğinde artan beklemeyle yeniden bağlanır             |
| `--limit <n>`       | `200`    | Her getirme işlemi için azami satır sayısı                                             |
| `--max-bytes <n>`   | `250000` | Her getirme işleminde okunacak azami bayt sayısı                                       |
| `--interval <ms>`   | `1000`   | İzleme sırasındaki yoklama aralığı                                                      |
| `--json`            | kapalı     | Satırla ayrılmış JSON (satır başına bir olay)                                          |
| `--plain`           | kapalı     | TTY oturumlarında düz metni zorunlu kılar                                              |
| `--no-color`        | —          | ANSI renklerini devre dışı bırakır                                                     |
| `--utc`             | kapalı     | Zaman damgalarını UTC olarak işler (varsayılan yerel saattir)                          |
| `--local-time`      | kapalı     | Yerel saat varsayılanı için kabul edilen uyumluluk yazımıdır; bunun dışında etkisi yoktur |
| `--url` / `--token` | —          | Standart Gateway RPC bayrakları                                                        |
| `--timeout <ms>`    | `30000`  | Gateway RPC zaman aşımı                                                               |
| `--expect-final`    | kapalı     | Aracı destekli RPC son yanıt bekleme bayrağı (paylaşılan istemci katmanı üzerinden burada kabul edilir) |

Çıktı kipleri:

- **TTY oturumları**: düzenli, renkli ve yapılandırılmış günlük satırları.
- **TTY olmayan oturumlar**: düz metin.

Açık bir `--url` ilettiğinizde CLI, yapılandırma veya
ortam kimlik bilgilerini otomatik olarak uygulamaz; `--token` değerini kendiniz ekleyin, aksi takdirde çağrı
`gateway url override requires explicit credentials` hatasıyla başarısız olur.

JSON kipinde CLI, `type` etiketli nesneler yayar:

- `meta`: akış meta verileri (dosya, kaynak, kaynak türü, hizmet, imleç, boyut)
- `log`: ayrıştırılmış günlük girdisi
- `notice`: kesilme / döndürme ipuçları
- `raw`: ayrıştırılmamış günlük satırı
- `error`: Gateway bağlantı hataları (stderr'e yazılır)

Örtük yerel geri döngü Gateway'i eşleştirme isterse, bağlantı sırasında kapanırsa
veya `logs.tail` yanıt vermeden önce zaman aşımına uğrarsa `openclaw logs`,
yapılandırılmış Gateway dosya günlüğüne otomatik olarak geri döner. Açık `--url` hedefleri
bu geri dönüşü kullanmaz. `openclaw logs --follow` daha katıdır: Linux'ta kullanılabildiğinde
etkin kullanıcı systemd Gateway günlüğünü PID'ye göre kullanır; aksi takdirde güncelliğini yitirmiş olabilecek
yan yana bir dosyayı izlemek yerine canlı Gateway'e artan beklemeyle
yeniden bağlanmayı dener.

Gateway'e ulaşılamıyorsa CLI, şunu çalıştırmanızı belirten kısa bir ipucu yazdırır:

```bash
openclaw doctor
```

### Control UI (web)

Control UI'daki **Günlükler** sekmesi, `logs.tail` kullanarak aynı dosyayı canlı izler.
Nasıl açılacağını öğrenmek için [Control UI](/tr/web/control-ui) bölümüne bakın.

### Yalnızca kanal günlükleri

Kanal etkinliğini (WhatsApp/Telegram/vb.) filtrelemek için şunu kullanın:

```bash
openclaw channels logs --channel whatsapp
```

`--channel` varsayılan olarak `all` değerini kullanır; `--lines <n>` (varsayılan 200) ve `--json` de
kullanılabilir.

## Günlük biçimleri

### Dosya günlükleri (JSONL)

Günlük dosyasındaki her satır bir JSON nesnesidir. CLI ve Control UI,
yapılandırılmış çıktı (zaman, düzey, alt sistem, ileti) oluşturmak için bu girdileri ayrıştırır.

Dosya günlüğü JSONL kayıtları, kullanılabildiğinde makineyle filtrelenebilir
üst düzey alanları da içerir:

- `hostname`: Gateway ana makine adı.
- `message`: tam metin araması için düzleştirilmiş günlük iletisi metni.
- `agent_id`: günlük çağrısı aracı bağlamı taşıdığında etkin aracı kimliği.
- `session_id`: günlük çağrısı oturum bağlamı taşıdığında etkin oturum kimliği/anahtarı.
- `channel`: günlük çağrısı kanal bağlamı taşıdığında etkin kanal.

OpenClaw, numaralı tslog bağımsız değişken anahtarlarını okuyan mevcut ayrıştırıcıların
çalışmayı sürdürmesi için özgün yapılandırılmış günlük bağımsız değişkenlerini bu alanlarla birlikte korur.

Talk, gerçek zamanlı ses ve yönetilen oda etkinliği, aynı dosya günlüğü işlem hattı üzerinden
sınırlı yaşam döngüsü günlük kayıtları yayar. Bu kayıtlar, kullanılabildiğinde olay türü,
kip, aktarım, sağlayıcı ve boyut/zamanlama ölçümlerini içerir; ancak
transkript metnini, ses yüklerini, tur kimliklerini, çağrı kimliklerini ve sağlayıcı öğe kimliklerini içermez.

### Konsol çıktısı

Konsol günlükleri **TTY'ye duyarlıdır** ve okunabilirlik için biçimlendirilir:

- Alt sistem ön ekleri (ör. `gateway/channels/whatsapp`)
- Düzey renklendirmesi (bilgi/uyarı/hata)
- İsteğe bağlı kompakt veya JSON kipi

Konsol biçimlendirmesi `logging.consoleStyle` tarafından denetlenir.

### Gateway WebSocket günlükleri

`openclaw gateway`, RPC trafiği için WebSocket protokolü günlüğüne de sahiptir:

- normal kip: yalnızca kayda değer sonuçlar (hatalar, ayrıştırma hataları, yavaş çağrılar)
- `--verbose`: tüm istek/yanıt trafiği
- `--ws-log auto|compact|full`: ayrıntılı işleme biçemini seçer
- `--compact`: `--ws-log compact` için diğer ad

Örnekler:

```bash
openclaw gateway
openclaw gateway --verbose --ws-log compact
openclaw gateway --verbose --ws-log full
```

## Günlük kaydını yapılandırma

Tüm günlük yapılandırması, `~/.openclaw/openclaw.json` içindeki `logging` altında bulunur.

```json
{
  "logging": {
    "level": "info",
    "file": "/path/to/openclaw.log",
    "consoleLevel": "info",
    "consoleStyle": "pretty",
    "redactSensitive": "tools",
    "redactPatterns": ["sk-.*"]
  }
}
```

### Günlük düzeyleri

Düzeyler: `silent`, `fatal`, `error`, `warn`, `info`, `debug`, `trace`.

- `logging.level`: **dosya günlükleri** (JSONL) düzeyi (varsayılan: `info`).
- `logging.consoleLevel`: **konsol** ayrıntı düzeyi.

Her ikisini de **`OPENCLAW_LOG_LEVEL`** ortam değişkeniyle geçersiz kılabilirsiniz (ör. `OPENCLAW_LOG_LEVEL=debug`). Ortam değişkeni yapılandırma dosyasından önceliklidir; böylece `openclaw.json` dosyasını düzenlemeden tek bir çalıştırma için ayrıntı düzeyini artırabilirsiniz. Ayrıca ortam değişkenini ilgili komut için geçersiz kılan genel CLI seçeneği **`--log-level <level>`**'ü de iletebilirsiniz (örneğin `openclaw --log-level debug gateway run`).

`--verbose` yalnızca konsol çıktısını ve WS günlüğü ayrıntı düzeyini etkiler; dosya
günlüğü düzeylerini değiştirmez.

### Hedefli model aktarımı tanılamaları

Sağlayıcı çağrılarında hata ayıklarken tüm günlükleri `debug` düzeyine yükseltmek yerine
hedefli ortam bayraklarını kullanın:

```bash
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 openclaw gateway
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools OPENCLAW_DEBUG_SSE=events openclaw gateway
```

Kullanılabilir bayraklar:

- `OPENCLAW_DEBUG_MODEL_TRANSPORT=1`: istek başlangıcını, fetch yanıtını, SDK
  üst bilgilerini, ilk akış olayını, akış tamamlanmasını ve aktarım hatalarını
  `info` düzeyinde yayar.
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=summary`: model istek günlüklerine sınırlı bir istek yükü
  özeti ekler.
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=tools`: yük özetine modele yönelik tüm araç adlarını
  ekler.
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted`: gizli bilgileri ayıklanmış ve boyutu sınırlandırılmış bir JSON
  yükü anlık görüntüsü ekler. Yalnızca hata ayıklarken kullanın; gizli bilgiler ayıklanır ancak istemler
  ve ileti metni yine de bulunabilir.
- `OPENCLAW_DEBUG_SSE=events`: ilk olay ve akış tamamlanması zamanlamasını yayar.
- `OPENCLAW_DEBUG_SSE=peek`: ayrıca her olay için sınırlandırılmış, gizli bilgileri ayıklanmış ilk beş SSE olay
  yükünü yayar.
- `OPENCLAW_DEBUG_CODE_MODE=1`: kod kipi araç yüzeyinin sahibi olduğu için yerel sağlayıcı araçlarının
  gizlendiği durumlar dahil olmak üzere kod kipi model yüzeyi tanılamalarını
  yayar.

Bu bayraklar normal OpenClaw günlük kaydı üzerinden günlük oluşturur; dolayısıyla `openclaw logs --follow`
ve Control UI'daki Günlükler sekmesi bunları gösterir. Bayraklar olmadan aynı tanılamalar
`debug` düzeyinde kullanılabilir kalır.

`[model-fetch]` başlangıç ve yanıt meta verileri (sağlayıcı, API, model, durum,
gecikme ve yöntem, URL, zaman aşımı, proxy ve ilke gibi istek alanları),
`OPENCLAW_DEBUG_MODEL_TRANSPORT` değerinden bağımsız olarak her zaman `info` düzeyinde
yayılır; böylece temel model aktarımı düzeni, hata ayıklama bayrakları olmadan
görülebilir.

### İz bağıntısı

Dosya günlükleri JSONL biçimindedir. Bir günlük çağrısı geçerli bir tanılama iz bağlamı taşıdığında
OpenClaw, harici günlük işleyicilerinin satırı OTEL yayılımları ve sağlayıcı
`traceparent` yayılımıyla ilişkilendirebilmesi için iz alanlarını üst düzey JSON anahtarları
(`traceId`, `spanId`, `parentSpanId`, `traceFlags`) olarak yazar.

Gateway HTTP istekleri ve Gateway WebSocket çerçeveleri, dahili bir istek
iz kapsamı oluşturur. Bu eşzamansız kapsam içinde yayılan günlükler ve tanılama olayları,
açık bir iz bağlamı iletmediklerinde istek izini devralır. Aracı çalıştırma ve
model çağrısı izleri, etkin istek izinin alt öğeleri olur; böylece yerel günlükler,
tanılama anlık görüntüleri, OTEL yayılımları ve güvenilir sağlayıcı `traceparent` üst bilgileri,
ham istek veya model içeriği günlüğe kaydedilmeden `traceId` aracılığıyla birleştirilebilir.

Talk yaşam döngüsü günlük kayıtları da OpenTelemetry günlük dışa aktarımı
etkinleştirildiğinde, dosya günlükleriyle aynı sınırlı öznitelikleri kullanarak diagnostics-otel günlük dışa aktarımına
akar. OTLP, stdout JSONL veya her iki hedefi seçmek için `diagnostics.otel.logsExporter` yapılandırmasını kullanın.

### Model çağrısı boyutu ve zamanlaması

Model çağrısı tanılamaları, ham istem veya yanıt içeriğini
yakalamadan sınırlı istek/yanıt ölçümlerini kaydeder:

- `requestPayloadBytes`: nihai model isteği yükünün UTF-8 bayt boyutu
- `responseStreamBytes`: akışla iletilen model yanıtı parçası
  yüklerinin UTF-8 bayt boyutu. Yüksek frekanslı metin, düşünme ve araç çağrısı fark olayları,
  tam `partial` anlık görüntüleri yerine yalnızca artımlı `delta` baytlarını sayar.
- `timeToFirstByteMs`: akışla iletilen ilk yanıt olayından önce geçen süre
- `durationMs`: toplam model çağrısı süresi

Bu alanlar; tanılama anlık görüntüleri, model çağrısı Plugin kancaları ve
tanılama dışa aktarımı etkinleştirildiğinde OTEL model çağrısı span'leri/metrikleri tarafından kullanılabilir.

### Konsol stilleri

`logging.consoleStyle`:

- `pretty`: kullanıcı dostu, renkli ve zaman damgalı.
- `compact`: daha sıkı çıktı (uzun oturumlar için en iyisi).
- `json`: her satırda JSON (günlük işleyicileri için).

### Maskeleme

OpenClaw; hassas belirteçleri konsol çıktısına, dosya günlüklerine,
OTLP günlük kayıtlarına, kalıcı oturum transkripti metnine veya Control UI araç
olayı yüklerine (araç başlatma argümanları, kısmi/nihai sonuç yükleri, türetilmiş
çalıştırma çıktısı ve yama özetleri) ulaşmadan önce maskeleyebilir:

- Hassas değerlerin maskelenmesi her zaman etkindir.
- `logging.redactPatterns`: günlük/transkript çıktısı için varsayılan kümenin yerini alan regex dizelerinin listesi. Control UI araç yüklerinde özel kalıplar yerleşik varsayılanlara ek olarak uygulanır; dolayısıyla bir kalıp eklemek, varsayılanların zaten yakaladığı değerlerin maskelenmesini hiçbir zaman zayıflatmaz.

Dosya günlükleri ve oturum transkriptleri JSONL olarak kalır ancak eşleşen gizli
değerler, satır veya ileti diske yazılmadan önce maskelenir. Maskeleme en iyi
gayretle uygulanır: her tanımlayıcıya veya ikili yük alanına değil, metin içeren
ileti içeriğine ve günlük dizelerine uygulanır.

Yerleşik varsayılanlar; JSON alanları, URL parametreleri, CLI bayrakları veya
atamalar olarak göründüklerinde kart numarası, CVC/CVV, paylaşılan ödeme belirteci
ve ödeme kimlik bilgisi gibi yaygın API kimlik bilgilerini ve ödeme kimlik bilgisi alan adlarını kapsar.

OpenClaw ayrıca UI istemcilerine, destek paketlerine, tanılama gözlemcilerine,
onay istemlerine veya ajan araçlarına gösterilen güvenlik sınırı yüklerini de
maskeler. Özel `logging.redactPatterns`, bu yüzeylere projeye özgü kalıplar ekleyebilir.

## Tanılama ve OpenTelemetry

Tanılamalar, model çalıştırmaları ve ileti akışı telemetrisi (Webhook'lar,
kuyruğa alma, oturum durumu) için yapılandırılmış, makine tarafından okunabilir
olaylardır. Günlüklerin yerini **almazlar** — metrikleri, izleri ve dışa
aktarıcıları beslerler. Olaylar varsayılan olarak işlem içinde yayımlanır
(kapatmak için `diagnostics.enabled: false` ayarını yapın); bunların dışa aktarılması ayrı
bir işlemdir.

Birbiriyle ilişkili iki yüzey:

- **OpenTelemetry dışa aktarımı** — metrikleri, izleri ve günlükleri OTLP/HTTP üzerinden
  OpenTelemetry uyumlu herhangi bir toplayıcıya veya arka uca (Datadog, Grafana,
  Honeycomb, New Relic, Tempo vb.) gönderir. Tam yapılandırma, sinyal kataloğu,
  metrik/span adları, ortam değişkenleri ve gizlilik modeli özel bir sayfada yer alır:
  [OpenTelemetry dışa aktarımı](/tr/gateway/opentelemetry).
- **Tanılama bayrakları** — `logging.level` düzeyini yükseltmeden ek günlükleri
  `logging.file` hedefine yönlendiren hedefli hata ayıklama günlüğü bayraklarıdır. Bayraklar büyük/küçük
  harfe duyarlı değildir ve joker karakterleri (`telegram.*`, `*`) destekler. `diagnostics.flags`
  altında veya `OPENCLAW_DIAGNOSTICS=...` ortam geçersiz kılması aracılığıyla yapılandırın. Tam kılavuz:
  [Tanılama bayrakları](/tr/diagnostics/flags).

Bir toplayıcıya OTLP dışa aktarımı için [OpenTelemetry dışa aktarımı](/tr/gateway/opentelemetry) sayfasına bakın.

## Sorun giderme ipuçları

- **Gateway'e ulaşılamıyor mu?** Önce `openclaw doctor` komutunu çalıştırın.
- **Günlükler boş mu?** Gateway'in çalıştığını ve `logging.file` içindeki dosya yoluna
  yazdığını doğrulayın.
- **Daha fazla ayrıntı mı gerekiyor?** `logging.level` ayarını `debug` veya `trace` olarak ayarlayıp yeniden deneyin.

## İlgili konular

- [OpenTelemetry dışa aktarımı](/tr/gateway/opentelemetry) — OTLP/HTTP dışa aktarımı, metrik/span kataloğu, gizlilik modeli
- [Tanılama bayrakları](/tr/diagnostics/flags) — hedefli hata ayıklama günlüğü bayrakları
- [Gateway günlük kaydı iç işleyişi](/tr/gateway/logging) — WS günlük stilleri, alt sistem önekleri ve konsol yakalama
- [Yapılandırma referansı](/tr/gateway/configuration-reference#diagnostics) — tam `diagnostics.*` alan referansı
