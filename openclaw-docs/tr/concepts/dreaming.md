---
read_when:
    - Belleğin otomatik olarak yükseltilmesini istiyorsunuz
    - Her Dreaming aşamasının ne yaptığını anlamak istiyorsunuz
    - MEMORY.md dosyasını gereksiz içerikle doldurmadan birleştirmeyi ayarlamak istiyorsunuz
sidebarTitle: Dreaming
summary: Hafif, derin ve REM evreleri ile bir Rüya Günlüğü içeren arka planda bellek pekiştirme
title: Dreaming
x-i18n:
    generated_at: "2026-07-26T22:43:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 501ab42cfdfa0216c308896aa8c1719b06b49d64a62afdb004e097102a376eac
    source_path: concepts/dreaming.md
    workflow: 16
---

Dreaming, `memory-core` içindeki arka plan bellek birleştirme sistemidir. Süreci açıklanabilir ve incelenebilir tutarken güçlü kısa süreli sinyalleri kalıcı belleğe taşır.

<Note>
Dreaming **isteğe bağlıdır** ve varsayılan olarak devre dışıdır.
</Note>

## Dreaming neler yazar?

- `memory/.dreams/` içindeki **makine durumu** (geri çağırma deposu, aşama sinyalleri, alım kontrol noktaları, kilitler).
- `DREAMS.md` (veya mevcut bir `dreams.md`) içindeki **insanlar tarafından okunabilir çıktı** ve `memory/dreaming/<phase>/YYYY-MM-DD.md` altındaki isteğe bağlı aşama raporu dosyaları.

Uzun süreli yükseltme hâlâ yalnızca `MEMORY.md` konumuna yazar.

## Aşama modeli

Dreaming, her taramada sırasıyla üç iş birliğine dayalı aşama çalıştırır: hafif -> REM -> derin. Bunlar ayrı ayrı kullanıcı tarafından yapılandırılan modlar değil, dahili uygulama aşamalarıdır.

| Aşama | Amaç                                      | Kalıcı yazma      |
| ----- | ----------------------------------------- | ----------------- |
| Hafif | Yakın tarihli kısa süreli materyali sıralayıp hazırlamak | Hayır             |
| REM   | Temalar ve yinelenen fikirler üzerine düşünmek | Hayır             |
| Derin | Kalıcı adayları puanlayıp yükseltmek      | Evet (`MEMORY.md`) |

<AccordionGroup>
  <Accordion title="Hafif aşama">
    - Yakın tarihli kısa süreli geri çağırma durumunu, günlük bellek dosyalarını ve mevcut olduğunda hassas bilgileri çıkarılmış oturum dökümlerini okur.
    - Sinyallerin yinelenenlerini kaldırır ve aday satırları hazırlar.
    - Depolama satır içi çıktı içerdiğinde yönetilen bir `## Light Sleep` bloğu yazar.
    - Daha sonraki derin sıralama için pekiştirme sinyallerini kaydeder.
    - `MEMORY.md` konumuna asla yazmaz.

  </Accordion>
  <Accordion title="REM aşaması">
    - Yakın tarihli kısa süreli izlerden tema ve değerlendirme özetleri oluşturur.
    - Depolama satır içi çıktı içerdiğinde yönetilen bir `## REM Sleep` bloğu yazar.
    - Derin sıralamanın kullandığı REM pekiştirme sinyallerini kaydeder.
    - `MEMORY.md` konumuna asla yazmaz.

  </Accordion>
  <Accordion title="Derin aşama">
    - Adayları ağırlıklı puanlama ve eşik geçitleriyle sıralar (`minScore`, `minRecallCount`, `minUniqueQueries` değerlerinin tümü geçmelidir).
    - Yazmadan önce parçacıkları etkin günlük dosyalardan yeniden oluşturur; böylece eski veya silinmiş parçacıklar atlanır.
    - Yükseltilen girdileri `MEMORY.md` konumuna ekler.
    - `DREAMS.md` içine ve isteğe bağlı olarak `memory/dreaming/deep/YYYY-MM-DD.md` içine bir `## Deep Sleep` özeti yazar.

  </Accordion>
</AccordionGroup>

## Oturum dökümü alımı

Dreaming, hassas bilgileri çıkarılmış oturum dökümlerini Dreaming külliyatına alabilir. Dökümler mevcut olduğunda günlük bellek sinyalleri ve geri çağırma izleriyle birlikte hafif aşamayı besler. Kişisel ve hassas içerik alımdan önce çıkarılır.

## Rüya Günlüğü

Dreaming, `DREAMS.md` içinde anlatı biçiminde bir **Rüya Günlüğü** tutar. Her aşama yeterli materyale ulaştıktan sonra `memory-core`, en iyi çaba esaslı bir arka plan alt aracı çalıştırır ve `dreaming.model` yapılandırılmadığı sürece varsayılan çalışma zamanı modelini kullanarak kısa bir günlük girdisi ekler. Yapılandırılan model kullanılamıyorsa günlük çalışması, oturumun varsayılan modeliyle bir kez daha denenir; güven veya izin verilenler listesi hataları yeniden denenmez ve sessizce genel bir günlük girdisine geri dönmek yerine günlüklerde görünür kalır.

<Note>
Günlük, Dreams kullanıcı arayüzünde insanlar tarafından okunmak içindir; yükseltme kaynağı değildir. Günlük/rapor yapıtları kısa süreli yükseltmenin dışında tutulur; yalnızca dayanaklı bellek parçacıkları `MEMORY.md` içine yükseltilmeye uygundur.
</Note>

İnceleme ve kurtarma çalışmaları için dayanaklı bir geçmiş doldurma hattı da vardır:

<AccordionGroup>
  <Accordion title="Geçmiş doldurma komutları">
    - `memory rem-harness --path ... --grounded`, geçmiş `YYYY-MM-DD.md` notlarından dayanaklı günlük çıktısının önizlemesini gösterir.
    - `memory rem-backfill --path ...`, geri alınabilir dayanaklı günlük girdilerini `DREAMS.md` içine yazar.
    - `memory rem-backfill --path ... --stage-short-term`, dayanaklı kalıcı adayları normal derin aşamanın kullandığı aynı kısa süreli kanıt deposunda hazırlar.
    - `memory rem-backfill --rollback` ve `--rollback-short-term`, sıradan günlük girdilerine veya etkin kısa süreli geri çağırmaya dokunmadan bu hazırlanmış geçmiş doldurma yapıtlarını kaldırır.

  </Accordion>
</AccordionGroup>

Control UI, aracının Memory sekmesinde (Agents sayfası) aynı günlük geçmiş doldurma/sıfırlama akışını sunar; böylece dayanaklı adayların yükseltmeyi hak edip etmediğine karar vermeden önce sonuçları rüya sahnesinde inceleyebilirsiniz. Ayrı bir dayanaklı Scene hattı, hazırlanmış kısa süreli girdilerden hangilerinin geçmiş yeniden oynatmadan geldiğini ve yükseltilen öğelerden hangilerinin dayanaklı kaynaklı olduğunu gösterir; ayrıca etkin kısa süreli duruma dokunmadan yalnızca dayanaklı hazırlanmış girdileri temizlemenizi sağlar.

## Derin sıralama sinyalleri

Derin sıralama, aşama pekiştirmesine ek olarak altı ağırlıklı temel sinyal kullanır:

| Sinyal              | Ağırlık | Açıklama                                       |
| ------------------- | ------- | ---------------------------------------------- |
| İlgi                | 0.30    | Girdinin ortalama getirme kalitesi             |
| Sıklık              | 0.24    | Girdinin biriktirdiği kısa süreli sinyal sayısı |
| Sorgu çeşitliliği   | 0.15    | Girdiyi ortaya çıkaran farklı sorgu/gün bağlamları |
| Güncellik           | 0.15    | Zamana bağlı azalan güncellik puanı            |
| Birleştirme         | 0.10    | Birden çok gündeki yinelenme gücü              |
| Kavramsal zenginlik | 0.06    | Parçacık/yoldaki kavram etiketi yoğunluğu      |

Hafif ve REM aşaması eşleşmeleri, `memory/.dreams/phase-signals.json` kaynağından güncelliği zamanla azalan küçük bir destek ekler.

Gölge deneme sonuçları, herhangi bir kalıcı yazma işleminden önce inceleme sinyali olarak temel puanın üzerine eklenebilir: yararlı bir deneme adaya küçük ve sınırlı bir destek verir, nötr bir deneme adayın ertelenmiş kalmasını sağlar ve zararlı bir deneme adayı o puanlama geçişinde reddedilmiş olarak işaretler. Bu sinyal yalnızca rapor amaçlıdır; aday sıralamasını veya inceleme meta verilerini değiştirebilir ancak `MEMORY.md` konumuna asla yazmaz ve tek başına bir adayı yükseltmez.

### QA gölge denemesi raporu kapsamı

QA Lab, gelecekteki bir Dreaming gölge denemesinin aday belleği yükseltmeden önce nasıl inceleyebileceğini araştırmaya yönelik, yalnızca rapor amaçlı bir senaryo içerir: bir araç, temel yanıtı aday belleği kullanabilen bir yanıtla karşılaştırır ve ardından karar, gerekçe ve risk işaretlerini içeren yerel bir rapor yazar. Bu kapsam QA ile sınırlıdır; rapor yapıtının `MEMORY.md` öğesinden ayrı kaldığını ve aracın adayın yükseltildiğini asla iddia etmediğini doğrular. Üretim ortamına gölge deneme davranışı eklemez veya derin aşama yükseltme motorunu değiştirmez.

`memory-core` gölge denemesi çalıştırıcısı, kararlı bir yapıta ihtiyaç duyan kod yolları için aynı yalnızca rapor amaçlı sözleşmeyi korur. Adayı, deneme istemini, temel sonucu, aday sonucunu, kararı, gerekçeyi, risk işaretlerini ve kanıt başvurularını kabul eder; ardından `promotion action: report-only` ile bir rapor yazar. Yararlı kararlar bir `promote` önerisiyle, nötr kararlar `defer` ile ve zararlı kararlar `reject` ile eşleştirilir; bunların hiçbiri `MEMORY.md` konumuna yazmaz veya derin aşama yükseltmesini uygulamaz.

## Zamanlama

Etkinleştirildiğinde `memory-core`, tam bir Dreaming taraması için bir Cron işini otomatik olarak yönetir; bu iş, birincil çalışma zamanı çalışma alanı ve yapılandırılmış tüm araç çalışma alanları genelinde yinelenenlerden arındırılır. Böylece alt araç çalışma alanlarının dallanması, ana aracın `DREAMS.md` ve bellek durumunu dışlamaz.

| Ayar                 | Varsayılan    |
| -------------------- | ------------- |
| `dreaming.frequency` | `0 3 * * *`   |
| `dreaming.model`     | varsayılan model |

## Hızlı başlangıç

<Tabs>
  <Tab title="Dreaming'i etkinleştir">
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
  </Tab>
  <Tab title="Özel tarama sıklığı">
    ```json
    {
      "plugins": {
        "entries": {
          "memory-core": {
            "config": {
              "dreaming": {
                "enabled": true,
                "timezone": "America/Los_Angeles",
                "frequency": "0 */6 * * *"
              }
            }
          }
        }
      }
    }
    ```
  </Tab>
</Tabs>

## Eğik çizgi komutu

```text
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

`/dreaming on` ve `/dreaming off`, kanal çağıranları için sahip durumu veya Gateway istemcileri için `operator.admin` gerektirir. `/dreaming status` ve `/dreaming help` salt okunurdur.

## CLI iş akışı

<Tabs>
  <Tab title="Yükseltme önizlemesi / uygulama">
    ```bash
    openclaw memory promote
    openclaw memory promote --apply
    openclaw memory promote --limit 5
    openclaw memory status --deep
    ```

    Manuel `memory promote`, CLI bayraklarıyla geçersiz kılınmadığı sürece varsayılan olarak derin aşama eşiklerini kullanır.

  </Tab>
  <Tab title="Yükseltmeyi açıkla">
    Belirli bir adayın neden yükseltileceğini veya yükseltilmeyeceğini açıklayın:

    ```bash
    openclaw memory promote-explain "router vlan"
    openclaw memory promote-explain "router vlan" --json
    ```

  </Tab>
  <Tab title="REM düzeneği önizlemesi">
    Hiçbir şey yazmadan REM değerlendirmelerini, aday doğruları ve derin yükseltme çıktısını önizleyin:

    ```bash
    openclaw memory rem-harness
    openclaw memory rem-harness --json
    ```

  </Tab>
</Tabs>

## Temel varsayılanlar

Tüm ayarlar `plugins.entries.memory-core.config.dreaming` altında bulunur.

<ParamField path="enabled" type="boolean" default="false">
  Dreaming taramasını etkinleştirin veya devre dışı bırakın.
</ParamField>
<ParamField path="frequency" type="string" default="0 3 * * *">
  Tam Dreaming taramasının Cron sıklığı.
</ParamField>
<ParamField path="model" type="string">
  İsteğe bağlı Rüya Günlüğü alt araç modeli geçersiz kılma değeri. Ayrıca bir alt araç `allowedModels` izin verilenler listesi ayarlarken standart bir `provider/model` değeri kullanın.
</ParamField>
<ParamField path="phases.deep.maxPromotedSnippetTokens" type="number" default="160">
  `MEMORY.md` içine yükseltilen her kısa süreli geri çağırma parçacığından korunan tahmini en yüksek token sayısı. Sıralama kaynağı görünür kalır.
</ParamField>

<Warning>
`dreaming.model`, `plugins.entries.memory-core.subagent.allowModelOverride: true` gerektirir. Kısıtlamak için ayrıca `plugins.entries.memory-core.subagent.allowedModels` değerini ayarlayın. Otomatik yeniden deneme yalnızca model kullanılamıyor hatalarını kapsar; güven veya izin verilenler listesi hataları sessizce geri dönmek yerine günlüklerde görünür kalır.
</Warning>

<Note>
Aşama politikasının, eşiklerin ve depolama davranışının çoğu dahili uygulama ayrıntılarıdır. Tüm anahtarların listesi için [Bellek yapılandırması başvurusu](/tr/reference/memory-config#dreaming) bölümüne bakın.
</Note>

## Dreams kullanıcı arayüzü

Etkinleştirildiğinde Gateway **Dreams** sekmesi şunları gösterir:

- Dreaming'in geçerli etkinlik durumu
- aşama düzeyinde durum ve yönetilen taramanın varlığı
- kısa süreli, dayanaklı, sinyal ve bugün yükseltilen öğe sayıları
- bir sonraki zamanlanmış çalışmanın zamanı
- hazırlanmış geçmiş yeniden oynatma girdileri için ayrı bir dayanaklı Scene hattı
- `doctor.memory.dreamDiary` tarafından desteklenen, genişletilebilir bir Rüya Günlüğü okuyucusu

## İlgili

- [Bellek](/tr/concepts/memory)
- [Bellek CLI'si](/tr/cli/memory)
- [Bellek yapılandırması başvurusu](/tr/reference/memory-config)
- [Bellek araması](/tr/concepts/memory-search)
