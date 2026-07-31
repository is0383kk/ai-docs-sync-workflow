---
read_when:
    - Konum Node desteği veya izinler kullanıcı arayüzü ekleme
    - Android konum izinlerini veya ön plan davranışını tasarlama
summary: Node'lar için konum komutu, platform izin modları ve Linux GeoClue kurulumu
title: Konum komutu
x-i18n:
    generated_at: "2026-07-27T00:03:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 644229c1eafc8fc7b59bc23ba01d4ba95687ea66c4f9bd4a4cda98a87f2b6085
    source_path: nodes/location-command.md
    workflow: 16
---

## Özet

- `location.get`, `node.invoke` veya `openclaw nodes location get` aracılığıyla çağrılan bir Node komutudur.
- Varsayılan olarak kapalıdır.
- Android üçüncü taraf derlemeleri bir seçici kullanır: Kapalı / Kullanırken / Her Zaman. Play derlemelerinde Kapalı / Kullanırken seçenekleri bulunmaya devam eder.
- Kesin Konum ayrı bir açma/kapatma seçeneğidir.

## Neden yalnızca anahtar değil, seçici kullanılıyor?

İşletim sistemi konum izinleri birden fazla düzeye sahiptir. Kesin konum da ayrı bir işletim sistemi iznidir (iOS 14+ sürümünde "Precise", Android'de "fine" ve "coarse"). Uygulama içi seçici, istenen modu belirler ancak verilecek gerçek izne yine işletim sistemi karar verir.

## Ayarlar modeli

Her Node cihazı için:

- `location.enabledMode`: `off | whileUsing | always`
- `location.preciseEnabled`: bool

Kullanıcı arayüzü davranışı:

- `whileUsing` seçildiğinde ön plan izni istenir.
- Android üçüncü taraf derlemesinde `always` seçildiğinde önce ön plan izni istenir, arka plan erişimi açıklanır ve ardından ayrı **Allow all the time** izni için Android uygulama ayarları açılır.
- Android Play derlemeleri arka plan konum iznini bildirmez veya `always` seçeneğini göstermez.
- İşletim sistemi istenen düzeyi reddederse uygulama, izin verilen en yüksek düzeye geri döner ve durumu gösterir.

## İzin eşlemesi (node.permissions)

İsteğe bağlıdır. macOS Node'u, `node.list`/`node.describe` üzerindeki `permissions` eşlemesi aracılığıyla `location` değerini bildirir; iOS/Android bunu dahil etmeyebilir.

## Komut: `location.get`

`node.invoke` veya CLI yardımcısı aracılığıyla çağrılır:

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

Parametreler:

```json
{
  "timeoutMs": 10000,
  "maxAgeMs": 15000,
  "desiredAccuracy": "coarse|balanced|precise"
}
```

CLI bayrakları doğrudan eşlenir: `--location-timeout` -> `timeoutMs`, `--max-age` -> `maxAgeMs`, `--accuracy` -> `desiredAccuracy`.

Yanıt yükü:

```json
{
  "lat": 48.20849,
  "lon": 16.37208,
  "accuracyMeters": 12.5,
  "altitudeMeters": 182.0,
  "speedMps": 0.0,
  "headingDeg": 270.0,
  "timestamp": "2026-01-03T12:34:56.000Z",
  "isPrecise": true,
  "source": "gps|wifi|cell|unknown"
}
```

Hatalar (kararlı kodlar):

- `LOCATION_DISABLED`: seçici kapalıdır.
- `LOCATION_PERMISSION_REQUIRED`: istenen mod için izin eksiktir.
- `LOCATION_BACKGROUND_UNAVAILABLE`: uygulama arka plandadır ancak yalnızca Kullanırken izni verilmiştir.
- `LOCATION_TIMEOUT`: zamanında konum belirlenemedi.
- `LOCATION_UNAVAILABLE`: sistem hatası oluştu veya sağlayıcı yok.

## Arka plan davranışı

- Android üçüncü taraf derlemeleri, arka plandaki `location.get` isteklerini yalnızca kullanıcı `Always` seçeneğini belirlediyse ve Android arka plan konum izni verdiyse kabul eder. Mevcut kalıcı Node hizmeti, `location` hizmet türünü ekler ve etkinken `Location: Always` bilgisini açıkça gösterir.
- Android Play derlemeleri ve `While Using` modu, uygulama arka plandayken `location.get` isteklerini reddeder.
- Diğer Node platformları farklı davranabilir.

## Linux Node ana makinesi

Paketle gelen Linux Node Plugin'i, Linux masaüstü uygulaması bulunmayan başsız ana makineler de dahil olmak üzere CLI `openclaw node` hizmetine `location.get` ekler. Konum varsayılan olarak kapalıdır. Plugin girdisi altında etkinleştirin ve ardından Node hizmetini yeniden başlatın:

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          location: { enabled: true },
        },
      },
    },
  },
}
```

GeoClue2'yi ve `where-am-i` demosunu (Debian ve Ubuntu'da `geoclue-2-demo`) yükleyin. Node hizmeti kullanıcısına, ana makinenin GeoClue ilkesi ve yetkilendirme aracısı tarafından izin verilmelidir.

Plugin, bir dizi `busctl` çağrısı yerine `where-am-i` kullanır. GeoClue; istemci oluşturma, özellikler, başlatma, güncellemeler ve durdurma işlemlerini tek bir D-Bus istemci bağlantısına bağlar. Demo bu yaşam döngüsünü bir arada tutarken ayrı `busctl` alt süreçleri bunu yapmaz. Herhangi bir npm bağımlılığı eklenmez.

Linux; `coarse`, `balanced` ve `precise` değerlerini sırasıyla `4`, `6` ve `8` GeoClue doğruluk düzeyleriyle eşler. `maxAgeMs` değerini döndürülen zaman damgasına göre doğrular. GeoClue demosu seçilen sağlayıcıyı göstermediğinden `source` değeri `unknown` olur; `isPrecise` yalnızca bildirilen doğruluk 100 metre veya daha iyiyse true olur.

Linux aynı kararlı hataları kullanır: `LOCATION_DISABLED`, `LOCATION_TIMEOUT` ve `LOCATION_UNAVAILABLE`.

## Model/araç entegrasyonu

- Aracı aracı: `nodes` aracının `location_get` eylemi (Node gereklidir).
- CLI: `openclaw nodes location get --node <id>`.
- Aracı yönergeleri: yalnızca kullanıcı konumu etkinleştirdiyse ve kapsamını anlıyorsa çağırın.

## Kullanıcı deneyimi metni (önerilen)

- Kapalı: "Konum paylaşımı devre dışı."
- Kullanırken: "Yalnızca OpenClaw açıkken."
- Her Zaman: "OpenClaw arka plandayken istenen konum kontrollerine izin ver."
- Kesin: "Kesin GPS konumunu kullan. Yaklaşık konumu paylaşmak için kapatın."

## İlgili konular

- [Node'lara genel bakış](/tr/nodes)
- [Kanal konumu ayrıştırma](/tr/channels/location)
- [Kamerayla çekim](/tr/nodes/camera)
- [Konuşma modu](/tr/nodes/talk)
