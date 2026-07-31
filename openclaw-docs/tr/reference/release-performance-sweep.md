---
read_when:
    - Mayıs 2026 performans ve paket boyutu temizliğini doğruluyorsunuz
    - OpenClaw performansı ve bağımlılıkları hakkındaki blog gönderisinin temelini oluşturan sayılara ihtiyacınız var
    - Sürüm geçitlerini, paket shrinkwrap'ini veya plugin bağımlılık sınırlarını değiştiriyorsunuz
summary: Mayıs 2026 performans, paket boyutu, bağımlılık ve shrinkwrap temizliğine ilişkin görsel özet ve teknik kanıtlar
title: Sürüm performansı taraması
x-i18n:
    generated_at: "2026-07-26T23:00:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9e98ffc9d63e14e078a19368917eb4278695e1426048dc21942f928af145d5e1
    source_path: reference/release-performance-sweep.md
    workflow: 16
---

Bu sayfa, Mayıs 2026 OpenClaw performans, paket boyutu, bağımlılık ve shrinkwrap temizliğinin arkasındaki kanıtları sunar. Kamuya açık blog yazısının teknik tamamlayıcısıdır.

Burada iki denetim birleştirilmiştir:

- **Sürüm performansı taraması:** `v2026.5.28` sürümünden kararlı
  `v2026.4.23` sürümüne kadar GitHub Releases; `OpenClaw Performance` iş akışının
  `profile=smoke` mock sağlayıcı hattı kullanılarak. Çoğu etiket satırı tek bir örnektir;
  `v2026.5.27` ve `v2026.5.28` satırları, en son 3 tekrarlı sürüm dalı
  yapıtlarını kullanır.
- **Nisan ayının önceki bağlamı:** `v2026.4.1` ile
  `v2026.5.2` arasındaki yayımlanmış `clawgrit-reports` mock sağlayıcı
  temel değerleri; yalnızca bozuk Nisan sonu sürümlerini kamuya açık performans
  temel değeri olarak değerlendirmemek için kullanılmıştır.
- **Kurulum alanı taraması:** Geçici paketlere yeni
  `npm install --ignore-scripts` kurulumları; boyut için `du -sk node_modules` ve paket örneği
  sayıları için bir `node_modules` taraması kullanılarak.
- **npm paket boyutu taraması:** Yayımlanmış sürümler için
  `npm pack openclaw@<version> --dry-run --json`; sıkıştırılmış tarball boyutu, açılmış boyut ve dosya
  sayısı kaydedilerek.

<Warning>
Ana performans taraması, en son 3 tekrarlı sürüm dalı yapıtlarını kullanan
`v2026.5.27` ve `v2026.5.28` satırları dışında etiket başına bir
smoke örneği kullanır. Nisan ayının önceki bağlamı, `clawgrit-reports` kaynağındaki
yayımlanmış 3 tekrarlı medyanları kullanır. Sayıları sürüm geçidi istatistikleri
olarak değil, eğilim kanıtı ve regresyon arama sinyali olarak değerlendirin.
</Warning>

## Anlık Görünüm

Performans kapsamı: **77 istenen sürüm**, **yapıtlarla desteklenen 74 nokta**
ve **kullanılamayan 3 CI çalıştırması**. Ölçülen en son kararlı nokta:
`v2026.5.28`.

<CardGroup cols={2}>
  <Card title="Kararlı ajan turu" icon="gauge">
    **5,1 kat daha hızlı soğuk tur**

    - `v2026.4.14`: 9,8 sn
    - `v2026.5.28`: 1,9 sn

  </Card>
  <Card title="Yayımlanmış paket" icon="package">
    **17,9 MB tarball**

    En son kararlı paket; Mart ayındaki 43,3 MB'lık paket boyutu zirvesinden daha düşük.

  </Card>
  <Card title="En son kararlı kurulum" icon="hard-drive">
    **361,7 MiB yeni kurulum**

    İç içe geçmiş OpenClaw bağımlılık ağacını `2026.5.22`
    shrinkwrap kullanıma alma zirvesinden itibaren önemli ölçüde küçültür; ancak
    yerel kurulum denetiminde daha küçük, 259,7 MiB'lık iç içe ağaç hâlâ kalır.

  </Card>
  <Card title="Bağımlılık grafiği" icon="boxes">
    **300 kurulu paket**

    Betikler devre dışıyken yapılan yeni bir kurulumda benzersiz paket adı/sürüm
    kökleri olarak ölçülmüştür; önceki kararlı sürümden 71 kök daha azdır.

  </Card>
</CardGroup>

## 5.28 Sürümünde Neler Değişti?

`v2026.5.27` ile `v2026.5.28` arasındaki temizlik, yeteneklerin kendilerini
kaldırmak yerine varsayılan kurulum grafiğini küçülttü.

<CardGroup cols={2}>
  <Card title="Kök varsayılan grafiği" icon="git-branch">
    Benzersiz paket adı/sürüm kökleri **371**'den **300**'e düştü. Paket
    örnekleri **372**'den **301**'e düştü.
  </Card>
  <Card title="İç içe ağaç" icon="unplug">
    İç içe `openclaw/node_modules`, aynı yerel kurulum denetiminde **656,1 MiB**'dan
    **259,7 MiB**'a düştü.
  </Card>
  <Card title="İsteğe bağlı yerel koniler" icon="cpu">
    Tüm platformlara yönelik `@napi-rs/canvas` yerel paket konisi artık
    varsayılan kuruluma eklenmiyor.
  </Card>
  <Card title="Tedarik zinciri yüzeyi" icon="shield">
    Daha az varsayılan paket; varsayılan olarak güvenilmesi gereken daha az
    tarball, bakımcı, yerel ikili dosya, kurulum zamanı davranışı ve geçişli
    güncelleme yolu anlamına gelir.
  </Card>
</CardGroup>

<Tip>
Sorun tek başına shrinkwrap değildi. Kötü paket biçimiydi.
`v2026.5.28` hâlâ shrinkwrap ile sunulur, ancak yerel denetimde iç içe
bağımlılık ağacı çok daha küçüktür ve tüm platformlara yönelik canvas
dağılımı kaldırılmıştır.
</Tip>

## Öne Çıkan Sayılar

Nisan sonundaki bozuk satırları kamuya açık performans temel değerleri olarak
kullanmayın. `v2026.4.23` ve `v2026.4.29` faydalı regresyon kanıtlarıdır,
ancak `14x` tarzındaki büyük farklar çoğunlukla kötü bir sürüm
serisinden toparlanmayı gösterir.

Blog anlatısı için ölçek olarak Nisan ayının önceki yayımlanmış temel değerini
kullanın. Temel değer, yayımlanmış `clawgrit-reports` mock sağlayıcı çalışmasındaki
`v2026.4.14` değeridir (3 tekrar; bu çalışma yalnızca tanılama zaman çizelgesi
oluşturulmadığı için başarısız oldu, dolayısıyla soğuk, sıcak ve RSS medyanları
yaklaşık ölçek olarak hâlâ kullanışlıdır). Bunu sürüm geçidi istatistiği olarak
değil, anlatı bağlamı olarak değerlendirin.

| Metrik          | Nisan ayının önceki temel değeri | `v2026.5.28` |                    Fark |
| --------------- | ---------------------: | -----------: | -----------------------: |
| Soğuk ajan turu |                9.819 ms |      1.908 ms | %80,6 daha düşük, 5,1 kat daha hızlı |
| Sıcak ajan turu |                7.458 ms |      1.870 ms | %74,9 daha düşük, 4,0 kat daha hızlı |
| Ajan zirve RSS değeri  |                686,2 MB |      581,0 MB |              %15,3 daha düşük |

Mayıs taraması içinde, en son sürüm dalı satırı `v2026.5.2` değerinden
önemli ölçüde değişti:

| Metrik          | `v2026.5.2` | `v2026.5.28` |       Fark |
| --------------- | ----------: | -----------: | ----------: |
| Soğuk ajan turu |     3.897 ms |      1.908 ms | %51,0 daha düşük |
| Sıcak ajan turu |     3.610 ms |      1.870 ms | %48,2 daha düşük |
| Ajan zirve RSS değeri  |     613,7 MB |      581,0 MB |  %5,3 daha düşük |

Önceki kararlı sürümle karşılaştırıldığında:

| Metrik          | `v2026.5.27` | `v2026.5.28` |       Fark |
| --------------- | -----------: | -----------: | ----------: |
| Soğuk ajan turu |      2.231 ms |      1.908 ms | %14,5 daha düşük |
| Sıcak ajan turu |      2.226 ms |      1.870 ms | %16,0 daha düşük |
| Ajan zirve RSS değeri  |      649,0 MB |      581,0 MB | %10,5 daha düşük |

### Kurulum alanı

| Metrik                                          |  Temel değer | `v2026.5.28` |       Fark |
| ----------------------------------------------- | --------: | -----------: | ----------: |
| `2026.5.22` zirvesinden itibaren kurulum boyutu              | 1.020,6 MB |     361,7 MiB | %64,6 daha düşük |
| En son `2026.5.27` sürümünden itibaren kurulum boyutu    |  767,1 MiB |     361,7 MiB | %52,8 daha düşük |
| Aylık `2026.2.26` zirvesinden itibaren bağımlılıklar      |       645 |          300 | %53,5 daha düşük |
| En son `2026.5.27` sürümünden itibaren bağımlılıklar    |       371 |          300 | %19,1 daha düşük |
| `2026.5.22` sürümünden itibaren iç içe `openclaw/node_modules` |   911,8 MB |     259,7 MiB | %71,5 daha düşük |
| `2026.5.27` sürümünden itibaren iç içe `openclaw/node_modules` |  656,1 MiB |     259,7 MiB | %60,4 daha düşük |

### npm paket boyutu

| Sürüm     | Sıkıştırılmış tarball | Açılmış paket |  Dosyalar | Notlar                             |
| ----------- | -----------------: | ---------------: | -----: | --------------------------------- |
| `2026.1.30` |             12,8 MB |           33,5 MB |  4.607 | yeniden markalanan ilk paket           |
| `2026.2.26` |             23,6 MB |           82,9 MB | 10.125 | özellik büyümesi                    |
| `2026.3.31` |             43,3 MB |          182,6 MB | 21.037 | paket boyutu zirvesi           |
| `2026.4.29` |             22,9 MB |           74,6 MB |  9.309 | paket budaması görünür           |
| `2026.5.12` |             23,4 MB |           80,1 MB | 12.035 | büyük harici Plugin ayrımı       |
| `2026.5.22` |             17,2 MB |           76,9 MB | 12.386 | belgeler/varlıklar paketten hariç tutuldu |
| `2026.5.27` |             17,8 MB |           79,0 MB | 12.509 | önceki kararlı paket           |
| `2026.5.28` |             17,9 MB |           81,0 MB |  9.082 | en son kararlı paket             |

`2026.5.12`, değişiklik günlüğündeki görünür Plugin çıkarma dönüm noktasıdır:
Amazon Bedrock, Bedrock Mantle, Slack, OpenShell sandbox, Anthropic Vertex,
Matrix ve WhatsApp, bağımlılık konilerinin her çekirdek kurulumla değil bu
Plugin'lerle birlikte kurulması için çekirdek bağımlılık yolundan çıkarıldı.

## Kova ajan turu özeti

Nisan ayının kararlı sürüm serisi iki farklı hikâye içerir. Nisan ayının
başları yavaştı ancak tanınabilir durumdaydı. Nisan sonuysa bir regresyon
uçurumuna dönüştü. `v2026.5.2`, mock sağlayıcı hattının ilk kez 3-5 sn
aralığına düştüğü ve sağlanan taramada tutarlı biçimde başarılı olmaya
başladığı noktadır.

Daha önce yayımlanan bağlam:

| Sürüm      | Kova | Soğuk tur | Sıcak tur | Ajan zirve RSS değeri |
| ------------ | ---- | --------: | --------: | -------------: |
| `v2026.4.10` | BAŞARISIZ |  11.031 ms |   7.962 ms |        679,0 MB |
| `v2026.4.12` | BAŞARISIZ |  11.965 ms |   8.289 ms |        713,5 MB |
| `v2026.4.14` | BAŞARISIZ |   9.819 ms |   7.458 ms |        686,2 MB |
| `v2026.4.20` | BAŞARISIZ |  22.314 ms |  18.811 ms |        810,8 MB |
| `v2026.4.22` | BAŞARISIZ |   9.630 ms |   7.459 ms |        743,0 MB |

Sağlanan tarama:

| Sürüm             | Kova | Soğuk tur | Sıcak tur | Ajan zirve RSS değeri |
| ------------------- | ---- | --------: | --------: | -------------: |
| `v2026.4.23`        | BAŞARISIZ |  47.847 ms |   8.010 ms |      1.082,7 MB |
| `v2026.4.24`        | BAŞARISIZ |  48.264 ms |  25.483 ms |        996,0 MB |
| `v2026.4.25`        | BAŞARISIZ |  81.080 ms |  59.172 ms |      1.113,9 MB |
| `v2026.4.26`        | BAŞARISIZ |  76.771 ms |  54.941 ms |      1.140,8 MB |
| `v2026.4.27`        | BAŞARISIZ |  60.902 ms |  33.699 ms |      1.156,0 MB |
| `v2026.4.29`        | BAŞARISIZ |  94.031 ms |  57.334 ms |      3.613,7 MB |
| `v2026.5.2`         | BAŞARILI |   3.897 ms |   3.610 ms |        613,7 MB |
| `v2026.5.7`         | BAŞARILI |   3.923 ms |   3.693 ms |        654,1 MB |
| `v2026.5.12`        | BAŞARILI |   7.248 ms |   6.629 ms |        834,8 MB |
| `v2026.5.18`        | BAŞARILI |   3.301 ms |   2.913 ms |        630,3 MB |
| `v2026.5.20`        | BAŞARILI |   3.413 ms |   2.952 ms |        643,2 MB |
| `v2026.5.22`        | BAŞARILI |   4.494 ms |   4.093 ms |        654,3 MB |
| `v2026.5.26`        | BAŞARILI |   2.626 ms |   2.282 ms |        660,4 MB |
| `v2026.5.27-beta.1` | BAŞARILI |   2.575 ms |   2.217 ms |        635,3 MB |
| `v2026.5.27`        | BAŞARILI |   2.231 ms |   2.226 ms |        649,0 MB |
| `v2026.5.28`        | BAŞARILI |   1.908 ms |   1.870 ms |        581,0 MB |

## Kaynak probları

Gerekli prob giriş noktaları bu kaynak ağaçlarında henüz bulunmadığından,
başarılı 17 eski ref için kaynak probları atlandı. Bu refler için ajan turu
metrikleri yine de mevcuttur.

Temsilî kaynak probu noktaları:

| Sürüm             | Varsayılan `readyz` p50 | 50 Plugin `readyz` p50 | CLI sağlığı p50 | Plugin azami RSS değeri |
| ------------------- | -------------------: | ----------------------: | -------------: | -------------: |
| `v2026.4.29`        |              2.819 ms |                 2.618 ms |        1.679 ms |        389,0 MB |
| `v2026.5.2`         |              2.324 ms |                 2.013 ms |        1.384 ms |        377,2 MB |
| `v2026.5.7`         |              1.649 ms |                 1.540 ms |        1.175 ms |        387,6 MB |
| `v2026.5.18`        |              1.942 ms |                 1.927 ms |          607 ms |        426,5 MB |
| `v2026.5.20`        |              1.966 ms |                 1.987 ms |          621 ms |        455,0 MB |
| `v2026.5.22`        |              2.081 ms |                 1.884 ms |        5.095 ms |        444,2 MB |
| `v2026.5.26`        |              1.546 ms |                 1.634 ms |          656 ms |        400,4 MB |
| `v2026.5.27-beta.1` |              1.462 ms |                 1.548 ms |          548 ms |        394,0 MB |
| `v2026.5.27`        |              1.491 ms |                 1.571 ms |          553 ms |        401,5 MB |
| `v2026.5.28`        |              1.457 ms |                 1.474 ms |          623 ms |        386,1 MB |

Ajan turu hattı başarılı olsa da `v2026.5.22` CLI sağlığı sıçraması bu
tabloda görülebilir. Hedefli CLI veya Gateway regresyonlarını araştırırken
kaynak problarını koruyun.

## Kurulum alanı denetimi

Bağımlılık örnekleri, ayda bir kararlı sürümün yanı sıra
`2026.5.22` shrinkwrap'ı kullanıma sunma olayını ve en son `2026.5.28` sürümünü kullanır.

| Nokta              | Kurulu bağımlılıklar | Temiz kurulum | OpenClaw paketi | İç içe `openclaw/node_modules` | Kök shrinkwrap | Canvas kurulum davranışı                   |
| ------------------ | -------------: | ------------: | ---------------: | -----------------------------: | --------------- | ----------------------------------------- |
| Oca `2026.1.30`    |            605 |       438.4MB |           45.8MB |                          2.4MB | hayır              | üst düzey sarmalayıcı + `darwin-arm64`        |
| Şub `2026.2.26`    |            645 |       575.7MB |          110.1MB |                          3.5MB | hayır              | üst düzey sarmalayıcı + `darwin-arm64`        |
| Mar `2026.3.31`    |            438 |       584.1MB |          234.8MB |                            0MB | hayır              | üst düzey sarmalayıcı + `darwin-arm64`        |
| Nis `2026.4.29`    |            392 |       335.0MB |           97.4MB |                            0MB | hayır              | hiçbiri kurulmadı                            |
| `2026.5.22`        |            401 |     1,020.6MB |        1,020.4MB |                        911.8MB | evet             | iç içe: 12 `@napi-rs/canvas` paketinin tümü |
| May `2026.5.26`    |            371 |       767.5MB |          767.4MB |                        656.4MB | evet             | iç içe: 12 `@napi-rs/canvas` paketinin tümü |
| `2026.5.27`        |            371 |      767.1MiB |         766.9MiB |                       656.1MiB | evet             | iç içe: 12 `@napi-rs/canvas` paketinin tümü |
| En son `2026.5.28` |            300 |      361.7MiB |         361.6MiB |                       259.7MiB | evet             | hiçbiri kurulmadı                            |

### Shrinkwrap sınırı

`2026.5.20`, kök shrinkwrap ve büyük bir iç içe OpenClaw
bağımlılık ağacı olmadan yayımlandı. `2026.5.22` kök shrinkwrap'ı kullanıma sundu ve iç içe `openclaw/node_modules`
altına 911.8MB kurdu. `2026.5.28` shrinkwrap'ı korur ve iç içe `openclaw/node_modules`
altına hâlâ 259.7MiB kurar, ancak yerel temiz kurulum denetiminde artık
hiçbir `@napi-rs/canvas` paketi kurulmaz.

Yayımlanan tarball incelemesi sınırı doğrular:

| Sürüm     | Kararlı olarak yayımlandı mı? | Kök `npm-shrinkwrap.json` | Notlar                                 |
| ----------- | ----------------- | -------------------------- | ------------------------------------- |
| `2026.5.20` | evet               | hayır                         | shrinkwrap öncesindeki son kararlı sürüm |
| `2026.5.21` | hayır                | uygulanamaz                        | kararlı npm sürümü yok                 |
| `2026.5.22` | evet               | evet                        | shrinkwrap kullanıma sunuldu                 |
| `2026.5.23` | hayır                | uygulanamaz                        | kararlı npm sürümü yok                 |
| `2026.5.24` | hayır                | uygulanamaz                        | kararlı npm sürümü yok                 |
| `2026.5.25` | hayır                | uygulanamaz                        | kararlı npm sürümü yok                 |
| `2026.5.26` | evet               | evet                        | iç içe bağımlılık ağacı hâlâ mevcut  |
| `2026.5.27` | evet               | evet                        | iç içe bağımlılık ağacı hâlâ mevcut  |
| `2026.5.28` | evet               | evet                        | iç içe bağımlılık ağacı çok daha küçük   |

Önemli ayrım şudur: **sorun shrinkwrap'ın kendisi değildir**.
`v2026.5.28` hâlâ kök shrinkwrap ile yayımlanır. Sorun, npm'in büyük bir iç içe
OpenClaw bağımlılık ağacını ve 12 `@napi-rs/canvas` platform paketinin tümünü
oluşturmasına neden olan paket yapısıydı. İç içe ağaç `v2026.5.28` sürümünde
daha küçüktür ve canvas platform dağılımı artık yerel denetime dahil olmaz.

Shrinkwrap'ın sade bir dille açıklaması ve bakımcı düzeyindeki paket
kontrolleri için [npm shrinkwrap](/tr/gateway/security/shrinkwrap) bölümüne bakın.

## Tedarik zinciri yorumu

Bağımlılık sayısı yalnızca bir kurulum boyutu metriği değil, operasyonel bir
güvenlik metriğidir. Her paket, operatörlerin güvenmesi gereken bakımcıların,
tarball'ların, geçişli güncellemelerin, isteğe bağlı yerel ikililerin ve kurulum
sırasındaki davranışların kapsamını genişletir.

Temizleme yönü şöyledir:

- ağır ve isteğe bağlı yetenekleri varsayılan çekirdek kurulumun dışında tutmak
- plugin paketlerinin kendi çalışma zamanı bağımlılık grafiğinin sahibi olmasını sağlamak
- Gateway başlatılırken çalışma zamanı paket yöneticisi onarımından kaçınmak
- tüm platformlara ait yerel paketlerin oluşturulmasına neden olmadan deterministik
  kurulumları korumak
- paket kabulü ve ölçüm yollarında kurulum betiklerini devre dışı tutmak
- iç içe bağımlılık ağaçlarını ve yerel isteğe bağlı bağımlılık patlamalarını
  yayımlamadan önce yakalamak

İlgili belgeler:

- [Plugin bağımlılık çözümlemesi](/tr/plugins/dependency-resolution)
- [Plugin envanteri](/tr/plugins/plugin-inventory)
- [Tam sürüm doğrulaması](/tr/reference/full-release-validation)
