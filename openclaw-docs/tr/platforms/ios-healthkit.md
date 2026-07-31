---
read_when:
    - Bir iOS Node'unda HealthKit özetlerini etkinleştirme
    - health.summary çağrısı veya eksik sistem durumu metriklerinde sorun giderme
    - Bir iOS aygıtından hangi sağlık verilerinin çıkabileceğini inceleme
summary: Bir iOS Node'undan gizlilik iznine tabi HealthKit özetlerini etkinleştirme ve çağırma
title: HealthKit özetleri
x-i18n:
    generated_at: "2026-07-26T22:51:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b8ac13d2870c55e2083a5e3a14c3d04238c2780a9e83d091f31923eb738476af
    source_path: platforms/ios-healthkit.md
    workflow: 16
---

# HealthKit özetleri

OpenClaw, bağlı bir iPhone veya iPad Node'undan geçerli takvim gününün salt okunur bir özetini isteyebilir. Cihaz, toplam değerleri cihaz üzerinde hesaplar ve yalnızca adım sayısını, uyku süresini, ortalama dinlenme kalp atış hızını ve antrenman sayısını/süresini döndürür. Tekil HealthKit örnekleri, kaynaklar, meta veriler, klinik kayıtlar, arka planda veri alımı ve yazma işlemleri desteklenmez.

Bu özellik varsayılan olarak kapalıdır. iOS cihazında ayrı onay ve Gateway'de yetkilendirme gerektirir.

## Gereksinimler

- HealthKit'in sağlık verilerini kullanılabilir olarak bildirdiği OpenClaw iOS uygulamasını çalıştıran bir iPhone veya iPad.
- Bağlı ve onaylanmış bir iOS Node'u. Bkz. [iOS uygulaması kurulumu](/tr/platforms/ios).
- iOS Node'una erişebilen güncel bir Gateway.
- Görmeyi beklediğiniz tüm metrikler için okunabilir Sağlık verileri. Bir Apple Watch, Apple Sağlık deposuna veri sağlayabilir ancak HealthKit özetleri için OpenClaw watchOS uygulaması gerekli değildir.

## Erişimi etkinleştirme

### 1. Gateway komutunu yetkilendirme

`openclaw.json` içindeki mevcut `gateway.nodes.commands.allow` dizisine `health.summary` ekleyin. Zaten mevcut olan komutları koruyun:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["health.summary"] },
    },
  },
}
```

`health.summary` gizlilik açısından hassas olarak sınıflandırılır ve iOS platformunun varsayılan ayarlarında hiçbir zaman izin verilmez. `gateway.nodes.commands.deny` içindeki bir girdi, izin girdisini geçersiz kılar. Bkz. [Node komut ilkesi](/tr/nodes#command-policy).

### 2. iOS cihazında paylaşımı etkinleştirme

iOS uygulamasında:

1. **Settings -> Permissions** öğesini açın ve her zaman görünür olan **Apple Health** bölümünde **Apple Health Summaries** öğesini bulun.
2. **Enable Apple Health Summaries** öğesine dokunun.
3. Açıklamayı okuyun, ardından Apple'ın izin ekranında OpenClaw'un hangi Sağlık kategorilerini okuyabileceğini seçin.

Anahtar, açık OpenClaw paylaşım tercihinizi kaydeder. Apple'ın istenen her kategoriye izin verdiğini göstermez.

Sağlık özetlerini etkinleştirmek, Node'un bildirdiği komut yüzeyine `health.summary` ekler. Ortaya çıkan Node eşleştirme güncellemesini onaylayın:

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

Ardından bağlı iOS cihazının etkin bir `health.summary` komutu sunduğunu doğrulayın:

```bash
openclaw nodes describe --node "<iOS device name>"
```

## Bugünün özetini isteme

Yalnızca `today` desteklenir. iOS cihazının geçerli takvimi ve saat dilimi kullanılarak yerel gece yarısından istek zamanına kadar olan süreyi kapsar.

```bash
openclaw nodes invoke \
  --node "<iOS device name>" \
  --command health.summary \
  --params '{"period":"today"}' \
  --json
```

Ajanlar aynı komutu `nodes` aracıyla çağırabilir:

```json
{
  "action": "invoke",
  "node": "<iOS device name>",
  "invokeCommand": "health.summary",
  "invokeParamsJson": "{\"period\":\"today\"}"
}
```

Özet yükü şunları içerir:

| Alan                     | Anlamı                                        |
| ------------------------ | --------------------------------------------- |
| `period`                 | Her zaman `today`                             |
| `startISO`               | ISO zaman anı olarak kodlanmış yerel gün başlangıcı |
| `endISO`                 | ISO zaman anı olarak kodlanmış istek zamanı   |
| `timeZoneIdentifier`     | iOS cihazının saat dilimi tanımlayıcısı        |
| `stepCount`              | Yuvarlanmış toplam adım sayısı                 |
| `sleepDurationMinutes`   | Tekilleştirilmiş ve bugünle sınırlandırılmış uyku süresi |
| `restingHeartRateBpm`    | Ortalama dinlenme kalp atış hızı               |
| `workoutCount`           | Bugün başlayan antrenmanlar                    |
| `workoutDurationMinutes` | Bu antrenmanların toplam süresi                 |

Metrik alanları isteğe bağlıdır ve HealthKit okunabilir bir değer döndürmediğinde atlanır. Süre hesaplanmadan önce uyku evreleri ve örtüşen kaynaklar birleştirilir; böylece aynı dakika iki kez sayılmaz.

## Gizlilik davranışı

- Toplama işlemi iOS cihazında gerçekleşir. Ham örnekler cihazdan ayrılmaz.
- İstenen toplam değer, Gateway'iniz üzerinden cihazdan ayrılır. Bir ajan bunu istediğinde toplam değer, yapılandırılmış yapay zekâ sağlayıcısına ulaşır ve sohbet geçmişinde kalabilir. Doğrudan CLI çağrısı, değeri CLI operatörüne döndürür.
- OpenClaw yalnızca okuma erişimi ister. Sağlık verisi ekleyemez veya değiştiremez.
- OpenClaw, HealthKit'i yalnızca `health.summary` çağrıldığında okur. Arka planda sağlık verisi alımı yapılmaz.
- HealthKit, okuma erişiminin reddedilip reddedilmediğini kasıtlı olarak açıklamaz. Eksik bir metrik; erişimin reddedildiği, eşleşen örnek bulunmadığı veya veri türünün kullanılamadığı anlamına gelebilir. OpenClaw bu durumları birbirinden ayıramaz.
- Özet, teşhis veya tıbbi tavsiye için değil, kişisel sağlık ve fitness bağlamı içindir.

Paylaşımı durdurmak için **Apple Health Summaries** öğesine dönün ve **Turn Off Summaries** öğesine dokunun. Ardından iOS cihazı, Sağlık yeteneğini ve `health.summary` komutunu Node yüzeyinden kaldırır. Geçidin Gateway tarafını kapatmak için `gateway.nodes.commands.allow` içinden `health.summary` öğesini de kaldırabilirsiniz.

## Sorun giderme

### Komut Node tarafından bildirilmemiş

Apple Sağlık özetlerinin iOS uygulamasında etkinleştirildiğini ve cihazın bağlı olduğunu doğrulayın. `openclaw nodes pending` komutunu çalıştırın ve tüm yetenek güncellemelerini onaylayın, ardından `openclaw nodes describe --node "<iOS device name>"` öğesini yeniden inceleyin.

### Komut açıkça etkinleştirme gerektiriyor

`gateway.nodes.commands.allow` içine `health.summary` ekleyin. Ayrıca `gateway.nodes.commands.deny` öğesinin bunu içermediğini kontrol edin; engelleme listesi önceliklidir.

### `HEALTH_ACCESS_DISABLED`

Uygulama tarafındaki paylaşım anahtarı kapalıdır. iOS cihazında **Settings -> Permissions -> Apple Health** altında **Apple Health Summaries** öğesini etkinleştirin.

### Özet başarılı ancak metrikler eksik

Apple'ın Sağlık uygulamasını açın ve bugün için veri bulunduğunu doğrulayın. Apple'ın Sağlık ayarlarında OpenClaw erişimini inceleyin ancak boş bir sonucu erişimin reddedildiğinin kanıtı olarak değerlendirmeyin: HealthKit bu ayrımı kasıtlı olarak gizler.

### Daha eski aralıklar başarısız oluyor

Komut yalnızca `{"period":"today"}` kabul eder. Birden çok günü kapsayan ve geçmişe dönük özetler desteklenmez.

## İlgili

- [iOS uygulaması](/tr/platforms/ios)
- [Node'lar](/tr/nodes)
- [Gateway yapılandırma referansı](/tr/gateway/configuration-reference#gateway)
- [Güvenlik denetimi](/tr/gateway/security)
