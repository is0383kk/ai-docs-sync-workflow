---
read_when:
    - Düz MEMORY.md notlarının ötesinde kalıcı bilgi istiyorsunuz
    - Paketle birlikte gelen memory-wiki Plugin'ini yapılandırıyorsunuz
    - Tek bir Gateway'deki aracılar için ayrı wiki kasalarına ihtiyacınız var
    - wiki_search, wiki_get veya köprü modunu anlamak istiyorsunuz
summary: 'memory-wiki: kaynak bilgileri, iddialar, panolar ve köprü modu içeren derlenmiş bilgi kasası'
title: Bellek vikisi
x-i18n:
    generated_at: "2026-07-26T23:27:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fda3c801ae39b529a3f1fcaf8791b6dcb1d8116ba2e73e99cca62dca6c64140a
    source_path: plugins/memory-wiki.md
    workflow: 16
---

`memory-wiki`, kalıcı bilgiyi gezinilebilir bir wiki hâlinde derleyen paketlenmiş bir plugindir: belirlenimci sayfalar, kanıtlarla yapılandırılmış iddialar, kaynak geçmişi, panolar ve makine tarafından okunabilir özetler.

Active Memory plugininin yerini almaz. Hatırlama, yükseltme, indeksleme ve Dreaming, yapılandırılmış olan bellek arka ucunun
(`memory-core`, QMD, Honcho vb.) sorumluluğunda kalır. `memory-wiki` onun yanında yer alır ve bilgiyi bakımı yapılan bir wiki katmanı hâlinde derler.

CLI'sini, araçlarını veya çalışma zamanı entegrasyonunu kullanmadan önce plugini etkinleştirin:

```bash
openclaw plugins enable memory-wiki
openclaw gateway restart
```

| Katman               | Sorumluluğu                                                                       |
| -------------------- | --------------------------------------------------------------------------------- |
| Active Memory plugini | Hatırlama, anlamsal arama, yükseltme, Dreaming, bellek çalışma zamanı             |
| `memory-wiki`        | Derlenmiş wiki sayfaları, kaynak geçmişi açısından zengin sentezler, panolar, wiki arama/get/apply |

Pratik kural:

- Yapılandırılmış tüm derlemler genelinde tek bir geniş hatırlama geçişi için `memory_search`
- Wikiye özgü sıralama, kaynak geçmişi veya sayfa düzeyinde inanç yapısı istediğinizde `wiki_search` / `wiki_get`
- Active Memory plugini derlem seçimini desteklediğinde, tek çağrıda her iki katmanı da kapsamak için `memory_search corpus=all`

Yaygın bir yerel öncelikli kurulum: hatırlama için Active Memory arka ucu olarak QMD ve kalıcı sentezlenmiş sayfalar için `bridge` modunda
`memory-wiki`. [Yapılandırma](#configuration) altındaki QMD + köprü modu örneğine bakın.

Köprü modu dışa aktarılmış sıfır yapıt bildiriyorsa Active Memory plugini şu anda herkese açık köprü girdileri sunmuyor demektir. Önce `openclaw wiki doctor` komutunu çalıştırın, ardından Active Memory plugininin herkese açık yapıtları desteklediğini doğrulayın.

## Kasa modları

- `isolated` (varsayılan): kendi kasası, kendi kaynakları vardır; Active Memory pluginine bağımlılığı yoktur. Bağımsız ve özenle seçilmiş bir bilgi deposu için bunu kullanın.
- `bridge`: herkese açık plugin SDK bağlantı noktaları üzerinden Active Memory pluginindeki herkese açık bellek yapıtlarını ve olay günlüklerini okur. Bellek plugininin dışa aktardığı yapıtları, pluginin özel iç bileşenlerine erişmeden derlemek için bunu kullanın.
- `unsafe-local`: yerel özel yollar için açıkça etkinleştirilen, aynı makineye özgü bir kaçış yoludur. Bilerek deneysel ve taşınamazdır; yalnızca güven sınırını anladığınızda ve özellikle köprü modunun sağlayamadığı yerel dosya sistemi erişimine ihtiyaç duyduğunuzda kullanın.

Kasa modu ile kasa kapsamı ayrı seçimlerdir:

- `vaultMode`, wiki girdilerinin nereden geleceğini seçer.
- `vault.scope`, tüm ajanların tek kasa kullanacağını mı yoksa her ajanın bir alt kasa alacağını mı seçer.

`vault.scope: "global"` varsayılandır ve mevcut tek kasa davranışını korur. Ajanların wiki sayfalarını, derlenmiş özetleri, arama sonuçlarını veya yazma işlemlerini paylaşmaması gerektiğinde `isolated` ya da `bridge` moduyla birlikte `vault.scope: "agent"` kullanın. Ajan kapsamı, `unsafe-local` moduyla birleştirilemez çünkü yapılandırılmış bu özel yollar ajana ait girdiler değildir. Yapılandırma doğrulaması bu birleşimi reddeder.

Köprü modu, `bridge.*` yapılandırma anahtarına göre şunları indeksleyebilir:

- dışa aktarılmış bellek yapıtları (`indexMemoryRoot`)
- günlük notlar (`indexDailyNotes`)
- Dreaming raporları (`indexDreamReports`)
- bellek olay günlükleri (`followMemoryEvents`)

Köprü modu etkinken ve `bridge.readMemoryArtifacts` etkinleştirildiğinde,
`openclaw wiki status`, `openclaw wiki doctor` ve `openclaw wiki bridge
import`, ajan/çalışma zamanı belleğiyle aynı Active Memory plugini bağlamını görmeleri için çalışan Gateway üzerinden yönlendirilir. Köprü devre dışıysa veya yapıt okumaları kapalıysa bu komutlar yerel/çevrimdışı davranışını korur.

## Kasa düzeni

```text
<vault>/
  AGENTS.md
  WIKI.md
  index.md
  inbox.md
  entities/
  concepts/
  syntheses/
  sources/
  reports/
  _attachments/
  _views/
  .openclaw-wiki/
```

Yönetilen içerik, oluşturulan blokların içinde kalır; insan notu blokları yeniden oluşturma işlemleri boyunca korunur.

- `sources/`: içe aktarılmış ham materyal ile köprü/güvenli olmayan yerel destekli sayfalar
- `entities/`: kalıcı şeyler, kişiler, sistemler, projeler, nesneler
- `concepts/`: fikirler, soyutlamalar, örüntüler, politikalar (aynı zamanda OKF içe aktarımlarının hedef konumu)
- `syntheses/`: derlenmiş özetler ve bakımı yapılan toplu görünümler
- `reports/`: oluşturulan panolar

## Open Knowledge Format içe aktarımları

```bash
openclaw wiki okf import ./bundles/ga4
```

Paketinden çıkarılmış bir Open Knowledge Format paketini wiki kavram sayfalarına aktarın. Bir veri kataloğu, dokümantasyon tarayıcısı veya zenginleştirme ajanı zaten OKF üretiyorsa iyi bir seçimdir: OKF'yi taşınabilir değişim yapıtı olarak tutun, `memory-wiki` onu OpenClaw'a özgü kavram sayfalarına ve derlenmiş özetlere dönüştürsün.

- ayrılmamış `.md` dosyaları kavram belgeleridir
- içe aktarılan her kavram, boş olmayan bir `type` frontmatter alanı gerektirir; eksik `type`, bir `missing-type` uyarısı oluşturur ve dosya atlanır
- bilinmeyen `type` değerleri genel kavramlar olarak kabul edilir
- `index.md` ve `log.md` ayrılmıştır ve hiçbir zaman kavram olarak içe aktarılmaz
- bozuk veya harici Markdown bağlantıları değiştirilmeden bırakılır

İçe aktarılan sayfalar `concepts/` altında düzleştirilir; böylece mevcut derleme, arama, get ve pano akışları ikinci bir wiki ağacı olmadan bunları görür. Her sayfa özgün OKF kavram kimliğini, kaynak yolunu, `type`, `resource`, `tags`, zaman damgasını ve üreticinin tüm frontmatter verilerini korur. Dahili OKF bağlantıları, oluşturulan wiki kavram sayfalarına yeniden yazılır ve ayrıca `kind: okf-link` ile yapılandırılmış `relationships` girdileri oluşturur.

## Yapılandırılmış iddialar ve kanıtlar

Sayfalar yalnızca serbest biçimli metin değil, yapılandırılmış `claims` frontmatter verisi taşır. Her iddia `id`, `text`, `status`, `confidence`, `evidence[]` ve
`updatedAt` içerebilir. Her kanıt girdisi `kind`, `sourceId`, `path`,
`lines`, `weight`, `confidence`, `privacyTier`, `note` ve `updatedAt` içerebilir.

Bu, wikinin pasif bir not yığını gibi değil, bir inanç katmanı gibi davranmasını sağlar. İddialar izlenebilir, puanlanabilir, itiraz konusu yapılabilir ve kaynaklara göre çözümlenebilir.

## Ajana yönelik varlık meta verileri

Varlık sayfaları; kişiler, ekipler, sistemler, projeler veya diğer tüm varlık türleri için kullanılabilen genel yönlendirme meta verileri taşır:

- `entityType`: örneğin `person`, `team`, `system`, `project`
- `canonicalId`: takma adlar ve içe aktarımlar genelinde kararlı kimlik anahtarı
- `aliases`: aynı sayfaya çözümlenen adlar, kullanıcı adları veya etiketler
- `privacyTier`: serbest biçimli dize; `public` inceleme gerektirmeyen olarak değerlendirilir, diğer tüm değerler (örneğin `local-private`, `sensitive`, `confirm-before-use`) `reports/privacy-review.md` içinde işaretlenir
- `bestUsedFor` / `notEnoughFor`: kompakt yönlendirme ipuçları
- `lastRefreshedAt`: sayfa düzenleme zamanından ayrı kaynak yenileme zaman damgası
- `personCard`: isteğe bağlı, kişiye özgü yönlendirme kartı (kullanıcı adları, sosyal hesaplar, e-postalar, saat dilimi, kulvar, sorulacak konular, sorulmaması gereken konular, güven düzeyi, gizlilik katmanı)
- `relationships`: ilişkili sayfalara yönelen türü belirlenmiş kenarlar (hedef, tür, ağırlık, güven düzeyi, kanıt türü, gizlilik katmanı, not)

Bir kişi wikisi için `reports/person-agent-directory.md` ile başlayın, ardından iletişim bilgilerini veya çıkarılmış olguları kullanmadan önce kişi sayfasını `wiki_get` ile açın.

<Accordion title="Varlık sayfası örneği">
```yaml
pageType: entity
entityType: person
id: entity.example-person
canonicalId: maintainer.example-person
aliases:
  - Alex
  - example-handle
privacyTier: local-private
bestUsedFor:
  - Örnek ekosistem yönlendirmesi
notEnoughFor:
  - yasal onay
lastRefreshedAt: "2026-04-29T00:00:00.000Z"
personCard:
  handles:
    - "@example-handle"
  socials:
    - "https://x.example/example-handle"
  emails:
    - alex@example.com
  timezone: America/Chicago
  lane: Örnek ekosistem
  askFor:
    - Örnek kullanıma sunma soruları
  avoidAskingFor:
    - ilgisiz faturalandırma kararları
  confidence: 0.8
  privacyTier: confirm-before-use
relationships:
  - targetId: entity.other-person
    targetTitle: Diğer Kişi
    kind: collaborates-with
    confidence: 0.7
    evidenceKind: discrawl-stat
claims:
  - id: claim.example.routing
    text: Alex, örnek ekosistem yönlendirmesi için yararlıdır.
    status: supported
    confidence: 0.9
    evidence:
      - kind: maintainer-whois
        sourceId: source.maintainers
        privacyTier: local-private
```
</Accordion>

## Derleme işlem hattı

Derleme, wiki sayfalarını okur, özetleri normalleştirir ve makineye yönelik bir anlık görüntüyü OpenClaw'ın paylaşılan SQLite plugin durumunda kalıcı hâle getirir. Çalışma zamanı kodu, eşzamansız istem hazırlığı sırasında SQLite'ı yüklemek için yaşam döngüsüne ait sahip anlık görüntüsünü kullanır; eşzamanlı istem oluşturma hiçbir zaman Markdown'ı kazımaz veya önbellek dosyalarını okumaz. Derlenmiş çıktı ayrıca arama/get için ilk geçiş wiki indekslemesini, iddia kimliklerinin sahip sayfalara geri çözümlenmesini, kompakt istem eklerini ve rapor oluşturmayı destekler.

Kaynak düzenlemeleri ve kasa geri yüklemeleri, yalnızca sonraki derlemeden sonra makineye yönelik hâle gelir. Plugin yaşam döngüsünü yeniden başlatmak veya yenilemek, kasanın nedensel olarak zincirlenmiş derleme yayınını SQLite ile karşılaştırır ve daha yeni, geri alınmış bir durumdan gelen anlık görüntüyü reddeder. Geri almadan önce başlatılan bir derleyici, geri yüklenmiş öncül duruma karşı yayın yapamaz. İstem hazırlığı kasayı yoklamaz veya dosya izleyicileri kurmaz.
Geri alma karantinasından sonra, çalışan süreçte yapılan bir derleme sahibi hemen temizler; ayrı bir derleyici süreci ise daemon'ın yeni kalıcı yayını doğrulayabilmesi için plugin yaşam döngüsünün yenilenmesini gerektirir.
Derlenmiş önbellekler yeniden oluşturulabilir: yayın dönemlerinden önceki önbellek satırları isabetsiz kabul edilir ve sonraki derlemeyle değiştirilir; bunlar taşınmaz.

## Panolar ve sistem durumu raporları

`render.createDashboards` etkinleştirildiğinde derleme, `reports/` altında panoların bakımını yapar:

| Rapor                               | İzlediği                                            |
| ----------------------------------- | -------------------------------------------------- |
| `reports/open-questions.md`         | çözümlenmemiş soruları olan sayfalar                |
| `reports/contradictions.md`         | çelişki notu kümeleri                               |
| `reports/low-confidence.md`         | düşük güven düzeyli sayfalar ve iddialar            |
| `reports/claim-health.md`           | yapılandırılmış kanıtı olmayan iddialar             |
| `reports/stale-pages.md`            | eskimiş veya bilinmeyen güncellik                   |
| `reports/person-agent-directory.md` | kişi/varlık yönlendirme kartları                    |
| `reports/relationship-graph.md`     | yapılandırılmış ilişki kenarları                    |
| `reports/provenance-coverage.md`    | kanıt sınıfı kapsamı                                |
| `reports/privacy-review.md`         | kullanımdan önce incelenmesi gereken herkese açık olmayan gizlilik katmanları |

## Arama ve getirme

İki arama arka ucu:

- `shared`: kullanılabildiğinde paylaşılan bellek arama akışını kullanır
- `local`: wikiyi yerel olarak arar

Üç derlem: `wiki`, `memory`, `all`.

- `wiki_search` / `wiki_get`, mümkün olduğunda ilk geçiş olarak derlenmiş özetleri kullanır
- iddia kimlikleri sahip sayfaya geri çözümlenir
- itirazlı/eskimiş/güncel iddialar sıralamayı etkiler
- kaynak geçmişi etiketleri sonuçlarda korunur

Arama modları (`--mode` / araç `mode` parametresi):

| Mod               | Güçlendirdiği alanlar                                            |
| ----------------- | -------------------------------------------------------------- |
| `auto`            | dengeli varsayılan                                              |
| `find-person`     | kişi benzeri varlıklar, diğer adlar, kullanıcı adları, sosyal hesaplar, kanonik kimlikler |
| `route-question`  | ajan kartları, ne sorulacağına/en iyi kullanım alanına ilişkin ipuçları, ilişki bağlamı |
| `source-evidence` | kaynak sayfaları ve yapılandırılmış kanıt meta verileri         |
| `raw-claim`       | eşleşen yapılandırılmış iddialar; iddia/kanıt meta verilerini döndürür |

Bir sonuç yapılandırılmış bir iddiayla eşleştiğinde, `wiki_search` ayrıntılar
yükünde `matchedClaimId`, `matchedClaimStatus`, `matchedClaimConfidence`,
`evidenceKinds` ve `evidenceSourceIds` değerlerini döndürür. Metin çıktısı,
mevcut olduğunda kısa `Claim:` ve `Evidence:` satırlarını içerir.

## Ajan araçları

| Araç          | Amaç                                                                                                                                                          |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `wiki_status` | geçerli kasa modu ve kapsamı, çözümlenen ajan, sistem durumu, Obsidian CLI kullanılabilirliği                                                                  |
| `wiki_search` | wiki sayfalarında ve yapılandırıldığında paylaşılan bellek külliyatında arama yapar; kişi arama, soru yönlendirme, kaynak kanıtı veya ham iddia ayrıntılarına inme için `mode` kabul eder |
| `wiki_get`    | bir wiki sayfasını kimliğe/yola göre okur; paylaşılan arama etkinse ve arama sonuçsuz kalırsa paylaşılan bellek külliyatına geri döner                           |
| `wiki_apply`  | serbest biçimli sayfa düzenlemesi olmadan sınırlı sentez/meta veri değişiklikleri                                                                               |
| `wiki_lint`   | yapısal denetimler, kaynak kökeni boşlukları, çelişkiler, açık sorular                                                                                          |

Plugin ayrıca münhasır olmayan bir bellek külliyatı eklentisi kaydeder; böylece
etkin bellek plugin'i külliyat seçimini desteklediğinde paylaşılan
`memory_search` ve `memory_get` wiki'ye erişebilir.

## İstem ve bağlam davranışı

`context.includeCompiledDigestPrompt` etkinleştirildiğinde, bellek istemi bölümlerine
plugin durumundan derlenen kısa bir anlık görüntü eklenir: yalnızca en önemli
sayfalar, yalnızca en önemli iddialar, çelişki sayısı, soru sayısı ve
güven/güncellik niteleyicileri. İstem biçimini değiştirdiği için bu özellik
isteğe bağlıdır; esas olarak bellek eklentilerini açıkça kullanan bağlam
motorları veya istem oluşturma süreçleri açısından önem taşır.

## Yapılandırma

Yapılandırmayı `plugins.entries.memory-wiki.config` altına yerleştirin:

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "isolated",
          vault: {
            scope: "global",
            path: "~/.openclaw/wiki/main",
            renderMode: "obsidian",
          },
          obsidian: {
            enabled: true,
            useOfficialCli: true,
            vaultName: "OpenClaw Wiki",
            openAfterWrites: false,
          },
          bridge: {
            enabled: false,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          unsafeLocal: {
            allowPrivateMemoryCoreAccess: false,
            paths: [],
          },
          ingest: {
            autoCompile: true,
            maxConcurrentJobs: 1,
            allowUrlIngest: true,
          },
          search: {
            backend: "shared",
            corpus: "wiki",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
          render: {
            preserveHumanBlocks: true,
            createBacklinks: true,
            createDashboards: true,
          },
        },
      },
    },
  },
}
```

Temel anahtarlar:

| Anahtar                                    | Değerler / varsayılan                            | Notlar                                                                        |
| ------------------------------------------ | ------------------------------------------------ | ----------------------------------------------------------------------------- |
| `vaultMode`                                | `isolated` (varsayılan), `bridge`, `unsafe-local` | girdi ve entegrasyon davranışını seçer                                        |
| `vault.scope`                              | `global` (varsayılan), `agent`                    | tek bir paylaşılan kasa veya ajan başına bir alt kasa                         |
| `vault.path`                               | genel varsayılan `~/.openclaw/wiki/main`         | genel kapsamda tam kasa yolu; ajan kapsamındaki üst dizin varsayılan olarak `~/.openclaw/wiki` |
| `vault.renderMode`                         | `native` (varsayılan), `obsidian`                 |                                                                               |
| `bridge.readMemoryArtifacts`               | varsayılan `true`                                 | etkin bellek plugin'inin herkese açık yapıtlarını içe aktarır                 |
| `bridge.followMemoryEvents`                | varsayılan `true`                                 | köprü modunda olay günlüklerini dahil eder                                    |
| `unsafeLocal.allowPrivateMemoryCoreAccess` | varsayılan `false`                                | `unsafe-local` içe aktarmalarını çalıştırmak için gereklidir                  |
| `unsafeLocal.paths`                        | varsayılan `[]`                                   | `unsafe-local` modunda içe aktarılacak açıkça belirtilmiş yerel yollar       |
| `search.backend`                           | `shared` (varsayılan), `local`                    |                                                                               |
| `search.corpus`                            | `wiki` (varsayılan), `memory`, `all`              |                                                                               |
| `context.includeCompiledDigestPrompt`      | varsayılan `false`                                | seçili ajanın kısa özet anlık görüntüsünü bellek istemi bölümlerine ekler      |
| `render.createBacklinks`                   | varsayılan `true`                                 | belirlenimci ilişkili bloklar oluşturur                                       |
| `render.createDashboards`                  | varsayılan `true`                                 | pano sayfaları oluşturur                                                      |

### Ajan başına kasalar

Yapılandırılmış her ajana ayrı bir wiki vermek için `vault.scope` değerini
`agent` olarak ayarlayın. Bu kapsamda `vault.path` bir üst
dizindir ve OpenClaw normalleştirilmiş ajan kimliğini buna ekler:

```json5
{
  agents: {
    list: [{ id: "support" }, { id: "marketing" }],
  },
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "bridge",
          vault: {
            scope: "agent",
            path: "~/.openclaw/wiki",
          },
          bridge: {
            enabled: true,
            readMemoryArtifacts: true,
          },
        },
      },
    },
  },
}
```

Bu, `~/.openclaw/wiki/support` ve `~/.openclaw/wiki/marketing` olarak çözümlenir.
Ajan kapsamında `vault.path` belirtilmezse üst dizin varsayılan olarak
`~/.openclaw/wiki` olur. Bu nedenle varsayılan `main` ajanı mevcut
`~/.openclaw/wiki/main` yolunu korur.

Ajan araçları, derlenmiş istem özetleri ve `memory_search` /
`memory_get` üzerinden sunulan wiki eklentisi, kasayı etkin ajan
bağlamından çözümler. Birden çok yapılandırılmış ajanın bulunduğu bir kurulumda
CLI ve Gateway çağrıları için ajanı `openclaw wiki --agent <agentId> ...` veya Gateway isteğinin
`agentId` değeriyle açıkça belirtin. Kimlik sağlanmadığında tek bir
yapılandırılmış ajan varsayılan olarak kalır.

Köprü modunda, ajan kapsamlı içe aktarmalar yalnızca `agentIds` değeri
seçili ajanı içeriyorsa herkese açık bir bellek yapıtını kabul eder. Başka bir
ajana ait olan, sahiplik meta verileri bulunmayan veya sahibi bilinmeyen
yapıtlar atlanır. Genel kapsam, mevcut paylaşılan yapıt davranışını korur.

<Warning>
`vault.scope` değerini değiştirmek mevcut bir kasayı kopyalamaz veya bölmez.
Ajan kapsamında açıkça yapılandırılmış bir `vault.path` üst dizin hâline
gelir; bu nedenle üretim ajanlarını değiştirmeden önce mevcut sayfaları bilinçli
olarak taşıyın veya içe aktarın. Önce kasayı yedekleyin.

Ajan başına kasalar, işletim sistemi güvenlik sınırı değil, aynı süreç içinde
bir bilgi sınırıdır. Ana makinenin dosya sistemine erişimi olan plugin'ler ve
korumalı alanda çalışmayan araçlar başka bir ajanın dizinini yine de okuyabilir.
Ajanlar birbirine güvenmiyorsa [korumalı alan kullanımını](/tr/gateway/sandboxing)
veya [ayrı Gateway profillerini](/tr/gateway/multiple-gateways) kullanın.
</Warning>

### Örnek: QMD + köprü modu

Geri çağırma için QMD, sürdürülen bir bilgi katmanı için
`memory-wiki` kullanmak istediğinizde bunu kullanın. Her katman kendi
alanına odaklanır: QMD ham notları, oturum dışa aktarımlarını ve ek
koleksiyonları aranabilir tutarken `memory-wiki` kararlı varlıkları,
iddiaları, panoları ve kaynak sayfalarını derler.

```json5
{
  memory: {
    backend: "qmd",
  },
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "bridge",
          bridge: {
            enabled: true,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          search: {
            backend: "shared",
            corpus: "all",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
        },
      },
    },
  },
}
```

Bu, etkin bellek geri çağırma işleminin denetimini QMD'de tutar,
`memory-wiki` bileşenini derlenmiş sayfalara ve panolara odaklar ve
derlenmiş özet istemlerini bilinçli olarak etkinleştirene kadar istem biçimini
değiştirmez.

## CLI

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki get entity.alpha
openclaw wiki apply synthesis "Alpha Summary" --body "..." --source-id source.alpha
openclaw wiki bridge import
openclaw wiki obsidian status
```

`wiki okf import`, `wiki apply metadata`, `wiki unsafe-local import`,
`wiki chatgpt import` / `wiki chatgpt rollback` ve eksiksiz `wiki obsidian`
alt komut kümesi dâhil olmak üzere tüm komut başvurusu için
[CLI: wiki](/tr/cli/wiki) sayfasına bakın.

## Obsidian desteği

`vault.renderMode` değeri `obsidian` olduğunda plugin, Obsidian ile
uyumlu Markdown yazar ve isteğe bağlı olarak durum yoklama, kasa arama, sayfa
açma, komut çağırma ve günlük nota gitme işlemleri için resmî
`obsidian` CLI'ı kullanabilir. Bu isteğe bağlıdır; wiki, Obsidian
olmadan yerel modda çalışmaya devam eder.

Ajan kapsamlı kasalar Obsidian ile uyumlu Markdown kullanmaya devam edebilir,
ancak yapılandırma doğrulaması `obsidian.useOfficialCli: true` ile
`vault.scope: "agent"` birleşimini reddeder. Geçerli `obsidian.vaultName` ayarı
geneldir ve her ajan için ayrı bir Obsidian kasası seçemez. Bunun yerine wiki
araçlarını ve CLI işlemlerini kullanın veya Obsidian tarafından işletilen bir
wiki'yi genel kapsamda tutun.

## Önerilen iş akışı

<Steps>
<Step title="Geri çağırma için etkin bellek plugin'ini koruyun">
Geri çağırma, yükseltme ve Dreaming, yapılandırılmış bellek arka ucunun sorumluluğunda kalır.
</Step>
<Step title="memory-wiki'yi etkinleştirin">
Açıkça köprü modunu istemediğiniz sürece `isolated` moduyla başlayın.
</Step>
<Step title="Kaynak geçmişi önemli olduğunda wiki_search / wiki_get kullanın">
Wiki'ye özgü sıralama veya sayfa düzeyinde inanç yapısı istediğinizde bunları `memory_search` yerine tercih edin.
</Step>
<Step title="Dar kapsamlı sentezler veya meta veri güncellemeleri için wiki_apply kullanın">
Yönetilen, oluşturulmuş blokları elle düzenlemekten kaçının.
</Step>
<Step title="Anlamlı değişikliklerden sonra wiki_lint çalıştırın">
Çelişkileri, açık soruları ve kaynak geçmişindeki boşlukları yakalar.
</Step>
<Step title="Eskimişlik/çelişki görünürlüğü için panoları etkinleştirin">
`render.createDashboards: true` değerini ayarlayın (varsayılan).
</Step>
</Steps>

## İlgili belgeler

- [Belleğe Genel Bakış](/tr/concepts/memory)
- [CLI: bellek](/tr/cli/memory)
- [CLI: wiki](/tr/cli/wiki)
- [Plugin SDK'ya genel bakış](/tr/plugins/sdk-overview)
