---
read_when:
    - Anlamsal belleği indekslemek veya aramak istiyorsunuz
    - Bellek kullanılabilirliği veya indeksleme sorunlarını ayıklıyorsunuz
    - Hatırlanan kısa süreli belleği `MEMORY.md` içine yükseltmek istiyorsunuz
summary: '`openclaw memory` için CLI başvurusu (status/index/search/promote/promote-explain/rem-harness/rem-backfill)'
title: Belle​k
x-i18n:
    generated_at: "2026-07-26T23:15:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6354745f8622ee80345325fa6f3e7d6c5f280cb63b9cdb100a766cf9e300af59
    source_path: cli/memory.md
    workflow: 16
---

# `openclaw memory`

Anlamsal bellek indekslemeyi, aramayı ve `MEMORY.md` içine yükseltmeyi yönetin.
Paketle birlikte gelen `memory-core` plugin'i tarafından sağlanır ve
`plugins.slots.memory`, `memory-core` seçtiğinde (varsayılan) kullanılabilir. Diğer bellek
plugin'leri kendi CLI ad alanlarını sunar.

İlgili: [Bellek](/tr/concepts/memory) kavramı, [Dreaming](/tr/concepts/dreaming),
[Bellek yapılandırma referansı](/tr/reference/memory-config), [Bellek Wiki'si](/tr/plugins/memory-wiki),
[wiki](/tr/cli/wiki), [Plugin'ler](/tr/tools/plugin).

## `memory status`

```bash
openclaw memory status [--agent <id>] [--deep] [--index] [--fix] [--json] [--verbose]
```

`--agent` olmadan, `agents.entries` içindeki her aracı için çalışır; hiçbir aracı listesi
yapılandırılmamışsa varsayılan aracıya geri döner.

| Bayrak      | Etki                                                                                                                                                                                                                                                                                                      |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--deep`    | Vektör deposunun, gömme sağlayıcısının ve anlamsal aramanın hazır olma durumunu yoklar (ek sağlayıcı çağrıları gerektirir). Düz `memory status` hızlı kalır ve bunu atlar; bilinmeyen vektör/anlamsal durumu, yoklama yapılmadığı anlamına gelir. QMD sözcüksel `searchMode: "search"`, `--deep` ile bile anlamsal vektör yoklamalarını her zaman atlar. |
| `--index`   | Depo kirliyse yeniden indeksler. `--deep` anlamına gelir.                                                                                                                                                                                                                                                 |
| `--fix`     | Eski geri çağırma kilitlerini onarır ve yükseltme meta verilerini normalleştirir.                                                                                                                                                                                                                           |
| `--json`    | JSON yazdırır.                                                                                                                                                                                                                                                                                             |
| `--verbose` | Her aşama için ayrıntılı günlükler oluşturur.                                                                                                                                                                                                                                                              |

`Dreaming` satırı `dreaming.enabled: true` ile bile `off` olarak kalıyorsa veya
zamanlanmış taramalar hiç çalışmıyor gibi görünüyorsa yönetilen Dreaming Cron'u,
uzlaştırmayı tetiklemek için varsayılan aracının Heartbeat'inin çalışmasına bağlıdır. Zamanlama
ayrıntıları için [Dreaming](/tr/concepts/dreaming) bölümüne bakın.

Durum ayrıca `memory.search.extraPaths` içindeki ek arama yollarını da listeler.

## `memory index`

```bash
openclaw memory index [--agent <id>] [--force] [--verbose]
```

`status` ile aynı aracı başına kapsamlandırmayı kullanır. `--force`, artımlı
indeksleme yerine tam yeniden indeksleme çalıştırır. `--verbose`, indeksleme ilerlemesini
göstermeden önce aracı başına sağlayıcı, model, kaynaklar ve ek yol ayrıntılarını yazdırır.

## `memory search`

```bash
openclaw memory search [query] [--query <text>] [--agent <id>] [--max-results <n>] [--min-score <n>] [--json]
```

- Sorgu: konumsal `[query]` veya `--query <text>`. Her ikisi de ayarlanmışsa `--query`
  önceliklidir. Hiçbiri ayarlanmamışsa komut hata verir.
- `--agent <id>`: varsayılan olarak varsayılan aracıyı kullanır (tam aracı listesini değil).
- `--max-results <n>`: sonuç sayısını sınırlar (pozitif tam sayı).
- `--min-score <n>`: bu puanın altındaki eşleşmeleri filtreler.

## `memory promote`

`memory/YYYY-MM-DD.md` içindeki kısa vadeli adayları sıralayın ve isteğe bağlı olarak
en iyi girdileri `MEMORY.md` içine ekleyin.

```bash
openclaw memory promote [--agent <id>] [--limit <n>] [--min-score <n>] \
  [--min-recall-count <n>] [--min-unique-queries <n>] [--apply] [--include-promoted] [--json]
```

| Bayrak                     | Varsayılan        | Etki                                                                  |
| -------------------------- | ----------------- | --------------------------------------------------------------------- |
| `--limit <n>`              |                   | Döndürülecek/uygulanacak azami aday sayısı.                            |
| `--min-score <n>`          | `0.75`            | Asgari ağırlıklı yükseltme puanı.                                      |
| `--min-recall-count <n>`   | `3`               | Gerekli asgari geri çağırma sayısı.                                    |
| `--min-unique-queries <n>` | `2`               | Gerekli asgari farklı sorgu sayısı.                                    |
| `--apply`                  | yalnızca önizleme | Seçili adayları `MEMORY.md` içine ekler ve yükseltilmiş olarak işaretler. |
| `--include-promoted`       |                   | Önceki döngülerde zaten yükseltilmiş adayları dahil eder.              |
| `--json`                   |                   | JSON yazdırır.                                                         |

Bu CLI varsayılanları, zamanlanmış Dreaming taramasının derin aşama eşiklerinden
farklıdır (aşağıdaki [Dreaming](#dreaming) bölümüne bakın); tek seferlik manuel bir çalıştırmada
tarama davranışıyla eşleşmek için açık bayraklar geçirin.

Sıralama sinyalleri: hem bellek geri çağırmalarından hem de günlük alma
geçişlerinden elde edilen geri çağırma sıklığı, getirme ilgisi, sorgu çeşitliliği,
zamansal yakınlık, günler arası pekiştirme ve türetilmiş kavram zenginliğinin yanı sıra
tekrarlanan Dreaming ziyaretleri için hafif/REM aşaması pekiştirme artışı. Yükseltme,
yazmadan önce etkin günlük notu yeniden okur; böylece sıralamadan bu yana kısa vadeli
parçacıklarda yapılan düzenlemeler veya silmeler dikkate alınır ve eski bir anlık görüntüden
yükseltme yapılmaz.

## `memory promote-explain`

Bir yükseltme adayının puan dökümünü açıklayın.

```bash
openclaw memory promote-explain <selector> [--agent <id>] [--include-promoted] [--json]
```

`<selector>`, adayın anahtarıyla (tam veya alt dize), yoluyla ya da parçacık
metniyle eşleşir.

## `memory rem-harness`

Hiçbir şey yazmadan REM yansımalarını, aday doğruları ve derin aşama yükseltme
çıktısını önizleyin.

```bash
openclaw memory rem-harness [--agent <id>] [--path <file-or-dir>] [--grounded] [--include-promoted] [--json]
```

- `--path <file-or-dir>`: donanımı canlı çalışma alanı yerine geçmiş `YYYY-MM-DD.md`
  günlük dosyalarından başlatır.
- `--grounded`: geçmiş notlardan ayrıca temellendirilmiş bir `What Happened` / `Reflections` /
  `Possible Lasting Updates` önizlemesi oluşturur.

## `memory rem-backfill`

Kullanıcı arayüzü incelemesi için temellendirilmiş geçmiş REM özetlerini `DREAMS.md` içine yazın.
Geri alınabilir.

```bash
openclaw memory rem-backfill --path <file-or-dir> [--agent <id>] [--stage-short-term] [--json]
openclaw memory rem-backfill --rollback [--rollback-short-term] [--json]
```

- `--path <file-or-dir>`: `--rollback`/`--rollback-short-term`
  ayarlanmadığı sürece gereklidir. Geri doldurmanın yapılacağı geçmiş günlük bellek dosyaları veya dizini.
- `--stage-short-term`: normal derin aşamanın sıralayabilmesi için temellendirilmiş kalıcı adayları ayrıca canlı
  kısa vadeli yükseltme deposuna ekler.
- `--rollback`: önceden yazılmış temellendirilmiş günlük girdilerini
  `DREAMS.md` içinden kaldırır.
- `--rollback-short-term`: önceden hazırlanmış temellendirilmiş kısa vadeli
  adayları kaldırır.

## Dreaming

Dreaming, tek bir zamanlamada sırayla çalışan üç işbirlikçi aşamadan oluşan arka plan
bellek pekiştirme sistemidir: **hafif** (kısa vadeli materyali sırala/hazırla),
**REM** (yansıt ve temaları ortaya çıkar), **derin** (kalıcı olguları
`MEMORY.md` içine yükselt). Yalnızca derin aşama `MEMORY.md` içine yazar.

- `plugins.entries.memory-core.config.dreaming.enabled: true` ile etkinleştirin
  (varsayılan `false`); `memory-core` tarama Cron işini otomatik yönetir, manuel
  `openclaw cron add` gerekmez.
- Sohbetten `/dreaming on|off` ile açıp kapatın; `/dreaming status`
  (veya `/dreaming`/`/dreaming help`) ile inceleyin. `on`/`off`, kanal sahibi durumu
  veya Gateway `operator.admin` gerektirir; `status` ve yardım, komutu
  çalıştırabilen herkes tarafından kullanılabilir.
- İnsan tarafından okunabilir aşama çıktısı `DREAMS.md` içine (veya mevcut bir `dreams.md` içine) gider.
  Varsayılan olarak (`dreaming.storage.mode: "separate"`) her aşama ayrıca
  `memory/dreaming/<phase>/YYYY-MM-DD.md` içine bağımsız bir rapor yazar; raporları günlük bellek dosyasıyla birleştirmek için `mode:
"inline"`, her ikisi için `"both"`
  olarak ayarlayın.
- Zamanlanmış ve manuel `memory promote` çalıştırmaları aynı derin aşama
  sıralama sinyallerini paylaşır; yalnızca varsayılan eşikler farklıdır (yukarıdaki tablo ile
  aşağıdaki zamanlanmış varsayılanlara bakın).
- Zamanlanmış çalıştırmalar, yapılandırılmış her aracının bellek çalışma alanına yayılır.

Zamanlanmış varsayılanlar (`plugins.entries.memory-core.config.dreaming`):

| Anahtar                                | Varsayılan  |
| -------------------------------------- | ----------- |
| `frequency`                            | `0 3 * * *` |
| `phases.deep.minScore`                 | `0.8`       |
| `phases.deep.minRecallCount`           | `3`         |
| `phases.deep.minUniqueQueries`         | `3`         |
| `phases.deep.recencyHalfLifeDays`      | `14`        |
| `phases.deep.maxAgeDays`               | `30`        |
| `phases.deep.maxPromotedSnippetTokens` | `160`       |

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

Tam anahtar listesi ve aşama ayrıntıları: [Dreaming](/tr/concepts/dreaming),
[Bellek yapılandırma referansı](/tr/reference/memory-config#dreaming).

## SecretRef Gateway bağımlılığı

Active Memory uzak API anahtarı alanları SecretRef olarak yapılandırılmışsa `memory`
komutları bunları etkin Gateway anlık görüntüsünden çözümler; Gateway
kullanılamıyorsa komut hızla başarısız olur. Bunun için `secrets.resolve`
yöntemini destekleyen bir Gateway gerekir; eski Gateway'ler bilinmeyen yöntem hatası döndürür.

## İlgili

- [CLI referansı](/tr/cli)
- [Belleğe genel bakış](/tr/concepts/memory)
