---
read_when:
    - Genel günlük kaydı düzeylerini yükseltmeden hedefe yönelik hata ayıklama günlüklerine ihtiyacınız var
    - Destek için alt sisteme özgü günlükleri yakalamanız gerekir
summary: Hedefli hata ayıklama günlükleri için tanılama bayrakları
title: Tanılama bayrakları
x-i18n:
    generated_at: "2026-07-26T22:45:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ad3bdab6ba1fd98ba58c99c93f9a12d31f57e2655cb0c1eb2de09e34b970f56c
    source_path: diagnostics/flags.md
    workflow: 16
---

Tanılama bayrakları, genel olarak `logging.level` düzeyini yükseltmeden tek bir alt sistem için ek günlük kaydını etkinleştirir. Bir alt sistem bayrağı denetlemediği sürece bayrağın hiçbir etkisi olmaz.

## Nasıl çalışır?

- Bayraklar büyük/küçük harfe duyarlı olmayan dizelerdir; yapılandırmadaki `diagnostics.flags` ile
  `OPENCLAW_DIAGNOSTICS` ortam değişkeni geçersiz kılmasından çözümlenir, yinelenenler kaldırılır ve küçük harfe dönüştürülür.
- `name.*`, doğrudan `name` ile ve `name.` altındaki her şeyle eşleşir (örneğin
  `telegram.*`, `telegram.http` ile eşleşir).
- `*` veya `all` tüm bayrakları etkinleştirir.
- Yapılandırmadaki `diagnostics.flags` değiştirildikten sonra Gateway'i yeniden başlatın; bu ayar
  çalışırken yeniden yüklenmez.

## Bilinen bayraklar

| Bayrak                | Etkinleştirdiği özellikler                                |
| --------------------- | --------------------------------------------------------- |
| `telegram.http`       | Telegram Bot API HTTP hata günlükleri                     |
| `brave.http`          | Brave Search istek/yanıt/önbellek günlükleri              |
| `profiler`            | Yanıt aşaması profil oluşturucusu ve Codex uygulama sunucusu profil oluşturucusu (ikisi de) |
| `reply.profiler`      | Yalnızca yanıt aşaması profil oluşturucusu                |
| `codex.profiler`      | Yalnızca Codex uygulama sunucusu profil oluşturucusu      |
| `health`              | Gateway sistem durumu yoklaması/hesap/bağlama hata ayıklama ayrıntıları |
| `ingress.timing`      | Oturum yükleme, model seçimi ve model kataloğu zamanlamaları |
| `plugin.load-profile` | Eşzamanlı Plugin modülü yükleme zamanlamaları              |
| `timeline`            | Yapılandırılmış JSONL zaman çizelgesi yapıtı (aşağıya bakın) |

## Yapılandırma aracılığıyla etkinleştirme

```json
{
  "diagnostics": {
    "flags": ["telegram.http"]
  }
}
```

Birden fazla bayrak:

```json
{
  "diagnostics": {
    "flags": ["telegram.http", "brave.http", "gateway.*"]
  }
}
```

## Ortam değişkeniyle geçersiz kılma (tek seferlik)

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,brave.http
```

Değerler virgül veya boşluklardan bölünür. Özel değerler:

| Değer                       | Etki                                     |
| --------------------------- | ---------------------------------------- |
| `0`, `false`, `off`, `none` | Yapılandırmayı da geçersiz kılarak tüm bayrakları devre dışı bırakır |
| `1`, `true`, `all`, `*`     | Tüm bayrakları etkinleştirir              |

`OPENCLAW_DIAGNOSTICS=0`, söz konusu işlem için hem ortam değişkenindeki hem de yapılandırmadaki bayrakları devre dışı bırakır;
bu, yapılandırmada açık bırakılmış bir profil oluşturucu bayrağını dosyayı
düzenlemeden geçici olarak susturmak için kullanışlıdır.

## Profil oluşturucu bayrakları

Profil oluşturucu bayrakları, hafif zamanlama aralıklarını denetler; kapalı olduklarında ek yük oluşturmazlar.

Bir Gateway çalıştırması için profil oluşturucu tarafından denetlenen tüm aralıkları etkinleştirin:

```bash
OPENCLAW_DIAGNOSTICS=profiler openclaw gateway run
```

Yalnızca yanıt gönderimi profil oluşturucu aralıklarını etkinleştirin:

```bash
OPENCLAW_DIAGNOSTICS=reply.profiler openclaw gateway run
```

Yalnızca Codex uygulama sunucusunun başlatma/araç/iş parçacığı profil oluşturucu aralıklarını etkinleştirin:

```bash
OPENCLAW_DIAGNOSTICS=codex.profiler openclaw gateway run
```

`profiler`, hem yanıt profil oluşturucusunu hem de Codex profil oluşturucusunu etkinleştirir; yalnızca birini
etkinleştirmek için kapsamlı bayrak adlarını kullanın.

Alternatif olarak yapılandırmada ayarlayın:

```json
{
  "diagnostics": {
    "flags": ["reply.profiler", "codex.profiler"]
  }
}
```

Yapılandırma bayraklarını değiştirdikten sonra Gateway'i yeniden başlatın. Bir profil oluşturucu bayrağını devre dışı bırakmak için
bayrağı `diagnostics.flags` içinden kaldırıp yeniden başlatın veya söz konusu çalıştırmada
tüm tanılama bayraklarını geçersiz kılmak için işlemi `OPENCLAW_DIAGNOSTICS=0` ile başlatın.

## Zaman çizelgesi yapıtları

`timeline` bayrağı (takma ad: `diagnostics.timeline`), harici QA düzenekleri için yapılandırılmış başlatma
ve çalışma zamanı zamanlama olaylarını JSONL olarak yazar:

```bash
OPENCLAW_DIAGNOSTICS=timeline \
OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=/tmp/openclaw-timeline.jsonl \
openclaw gateway run
```

Alternatif olarak yapılandırmada etkinleştirin:

```json
{
  "diagnostics": {
    "flags": ["timeline"]
  }
}
```

Bayrağın kendisi yapılandırmada ayarlanmış olsa bile çıktı yolu her zaman `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH` değerinden
alınır; yol için bir yapılandırma anahtarı yoktur.
`timeline` yalnızca yapılandırmadan etkinleştirildiğinde, OpenClaw henüz yapılandırmayı okumadığı için en erken yapılandırma yükleme aralıkları
eksik olur; sonraki başlatma aralıkları normal şekilde yakalanır.

`OPENCLAW_DIAGNOSTICS=1`, `=all` ve `=*` de tüm bayrakları etkinleştirdikleri için zaman çizelgesini etkinleştirir.
Yalnızca JSONL yapıtını isteyip diğer tüm tanılama bayraklarını istemediğinizde kapsamlı `timeline`
bayrağını tercih edin.

Zaman çizelgesindeki olay döngüsü gecikme örnekleri, `timeline` dışında bir ek açık onay daha gerektirir:
zaman çizelgesini etkinleştirmenin yanı sıra `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` (veya `on`/`true`/`yes`) ayarlayın.

Zaman çizelgesi kayıtları `openclaw.diagnostics.v1` zarfını kullanır ve
işlem kimliklerini, aşama adlarını, aralık adlarını, süreleri, Plugin kimliklerini, bağımlılık
sayılarını, olay döngüsü gecikme örneklerini, sağlayıcı işlem adlarını, alt işlem çıkış
durumunu ve başlatma hatası adlarını/iletilerini içerebilir. Zaman çizelgesi dosyalarını yerel
tanılama yapıtları olarak değerlendirin; makinenizin dışında paylaşmadan önce inceleyin.

## Günlüklerin kaydedildiği yer

Bayraklar, standart tanılama günlüğü dosyasına günlük kayıtları gönderir. Varsayılan olarak:

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log
```

Adlandırılmış profiller `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log` kullanır; örneğin
`--dev`, `openclaw-dev-YYYY-MM-DD.log` kullanır.

`logging.file` ayarlanırsa bunun yerine o yolu kullanın. Günlükler JSONL biçimindedir (satır başına bir JSON
nesnesi). `logging.redactSensitive` temelinde redaksiyon uygulanmaya devam eder.
Günlük yolu çözümleme, döndürme ve redaksiyon modelinin tamamı için [Günlük Kaydı](/tr/logging) bölümüne
bakın.

## Günlükleri çıkarma

Etkin profilin en son günlük dosyasını okuyun:

```bash
openclaw logs --plain
# Adlandırılmış profil örneği:
openclaw --profile work logs --plain
```

Telegram HTTP tanılamalarını filtreleyin:

```bash
openclaw logs --plain --limit 5000 | rg "telegram http error"
```

Brave Search HTTP tanılamalarını filtreleyin:

```bash
openclaw logs --plain --limit 5000 | rg "brave http"
```

Ya da sorunu yeniden oluştururken günlükleri takip edin:

```bash
openclaw logs --follow --plain | rg "telegram http error"
```

Uzak Gateway'ler için bunun yerine `openclaw logs --follow` kullanın (bkz.
[/cli/logs](/tr/cli/logs)).

## Notlar

- `logging.level`, `warn` değerinden daha yüksek ayarlanırsa bayrakla denetlenen günlükler
  bastırılabilir. Varsayılan `info` uygundur.
- `brave.http`, Brave Search istek URL'lerini/sorgu parametrelerini, yanıt
  durumunu/zamanlamasını ve önbellek isabet/ıskalama/yazma olaylarını günlüğe kaydeder. API anahtarını
  (istek üstbilgisi olarak gönderilir) veya yanıt gövdelerini günlüğe kaydetmez; ancak arama sorguları
  hassas olabilir.
- Bayrakları etkin bırakmak güvenlidir; yalnızca ilgili alt sistemin
  günlük hacmini etkilerler.
- Günlük hedeflerini, düzeylerini ve redaksiyonu değiştirmek için [/logging](/tr/logging) sayfasını kullanın.

## İlgili

- [Gateway tanılama](/tr/gateway/diagnostics)
- [Gateway sorun giderme](/tr/gateway/troubleshooting)
