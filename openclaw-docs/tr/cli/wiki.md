---
read_when:
    - memory-wiki CLI'yi kullanmak istiyorsunuz
    - '`openclaw wiki` öğesini belgeliyor veya değiştiriyorsunuz'
summary: '`openclaw wiki` için CLI referansı (memory-wiki kasası durumu, arama, derleme, lint, uygulama, köprü, ChatGPT içe aktarma ve Obsidian yardımcıları)'
title: Viki
x-i18n:
    generated_at: "2026-07-26T23:55:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1f793d52de270068cf3a06b13f52242bb66738235718639486e090a2de213e73
    source_path: cli/wiki.md
    workflow: 16
---

# `openclaw wiki`

`memory-wiki` kasasını inceleyin ve yönetin. Birlikte sunulan isteğe bağlı `memory-wiki` plugin'i tarafından sağlanır. İlk kullanımdan önce etkinleştirin:

```bash
openclaw plugins enable memory-wiki
openclaw gateway restart
```

İlgili: [Memory Wiki plugin'i](/tr/plugins/memory-wiki), [Belleğe Genel Bakış](/tr/concepts/memory), [CLI: bellek](/tr/cli/memory)

## Yaygın komutlar

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki okf import ./knowledge-catalog/okf/bundles/ga4
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki search "Teams hakkında kime danışmalıyım?" --mode route-question
openclaw wiki get entity.alpha --from 1 --lines 80

openclaw wiki apply synthesis "Alpha Özeti" \
  --body "Kısa sentez metni" \
  --source-id source.alpha

openclaw wiki apply metadata entity.alpha \
  --source-id source.alpha \
  --status review \
  --question "Hâlâ etkin mi?"

openclaw wiki bridge import
openclaw wiki unsafe-local import
openclaw wiki chatgpt import --export ./chatgpt-export --dry-run
openclaw wiki chatgpt rollback <run-id>

openclaw wiki obsidian status
openclaw wiki obsidian search "alpha"
openclaw wiki obsidian open syntheses/alpha-summary.md
openclaw wiki obsidian command workspace:quick-switcher
openclaw wiki obsidian daily
```

## Aracı seçimi

`plugins.entries.memory-wiki.config.vault.scope`, `agent` olduğunda kasayı
üst düzey `--agent <id>` seçeneğiyle seçin:

```bash
openclaw wiki --agent support status
openclaw wiki --agent support search "iade politikası"
openclaw wiki --agent marketing ingest ./campaign-notes.md
```

Birden fazla yapılandırılmış aracının bulunduğu bir kurulumda, bir komutun herhangi bir varsayılan kasayı okuyamaması veya kasaya yazamaması için CLI
işlemlerinde `--agent` gereklidir. Yalnızca
bir aracı yapılandırılmışsa bu aracı varsayılan olarak kalır. Bilinmeyen aracı kimlikleri,
kasa işlemi başlamadan önce hataya neden olur. `vault.scope`, `global` olduğunda seçenek, seçili
yolu değiştirmez.

Gateway istemcileri de aynı kurala uyar: aracı kapsamlı çok aracılı bir kurulumda kasa destekli `wiki.*`
isteklerinde `agentId` iletin. Eksik veya bilinmeyen kimlik
hatadır. Aracı turları, wiki araçları, bellek külliyatı ekleri ve derlenmiş istem
özetleri etkin çalışma zamanı aracısı bağlamını zaten taşır.

## Komutlar

### `wiki status`

Kasa modunu ve kapsamını, çözümlenen aracıyı, sistem durumunu ve Obsidian CLI kullanılabilirliğini gösterir. Amaçlanan kasanın başlatılıp başlatılmadığını, köprü modunun sağlıklı olup olmadığını veya Obsidian entegrasyonunun kullanılabilirliğini kontrol etmek için önce bunu kullanın.

Köprü modu etkinken ve bellek yapıtlarını okuyacak şekilde yapılandırıldığında bu komut, aracı/çalışma zamanı belleğiyle aynı etkin bellek plugin'i bağlamını görebilmek için çalışan Gateway'i sorgular.

### `wiki doctor`

Wiki sistem durumu kontrollerini çalıştırır ve uygulanabilir düzeltmeleri bildirir. Sistem sağlıksızsa sıfır olmayan kodla çıkar.

Köprü modu etkinken ve bellek yapıtlarını okuyacak şekilde yapılandırıldığında bu komut, raporu oluşturmadan önce çalışan Gateway'i sorgular. Devre dışı bırakılan köprü içe aktarımları ve bellek yapıtlarını okumayan köprü yapılandırmaları yerel/çevrimdışı kalır.

Tipik sorunlar:

- herkese açık bellek yapıtları olmadan köprü modunun etkinleştirilmesi
- geçersiz veya eksik kasa düzeni
- Obsidian modu beklendiğinde harici Obsidian CLI'ın eksik olması

### `wiki init`

Üst düzey dizinler ve önbellek dizinleri dâhil olmak üzere wiki kasa düzenini ve başlangıç sayfalarını oluşturur.

### `wiki ingest <path>`

Yerel bir Markdown veya metin dosyasını kaynak sayfası olarak wiki `sources/` klasörüne aktarır. `<path>` yerel bir dosya yolu olmalıdır; şu anda URL'den veri alma desteklenmez. İkili dosyaları reddeder.

İçe aktarılan kaynak sayfalar köken bilgisi frontmatter'ı (`sourceType: local-file`, `sourcePath`, `ingestedAt`) taşır. Veri alma işlemi sonrasında kasa her zaman yeniden derlenir.

Bayraklar: `--title <title>` kaynak başlığını geçersiz kılar (varsayılan: dosya adından türetilir).

### `wiki okf import <path>`

Paketinden çıkarılmış bir Open Knowledge Format paketini wiki kavram sayfalarına aktarır.

İçe aktarıcı, OKF dizin ağacındaki ayrılmış olmayan her `.md` kavram belgesini okur, boş olmayan bir `type` alanı gerektirir ve bilinmeyen OKF `type` değerlerini genel kavramlar olarak ele alır. Ayrılmış OKF `index.md` ve `log.md` dosyaları kavram olarak içe aktarılmaz.

İçe aktarılan sayfalar `concepts/` altında düzleştirilir; böylece mevcut wiki derleme, arama, alma, özet ve pano akışları bunları hemen görür. Özgün OKF kavram kimliği, `type`, `resource`, `tags`, zaman damgası, kaynak yolu ve frontmatter'ın tamamı sayfa frontmatter'ında korunur. Dahili OKF Markdown bağlantıları oluşturulan wiki sayfalarına yönlendirilir; bozuk veya harici bağlantılar değiştirilmeden bırakılır. İçe aktarma sonrasında kasa her zaman yeniden derlenir.

Örnekler:

```bash
openclaw wiki okf import ./bundles/ga4
openclaw wiki okf import ./bundles/ga4 --json
openclaw wiki search "BigQuery Tablosu" --mode source-evidence --json
openclaw wiki get <path-from-json-result>
```

### `wiki compile`

Dizinleri, ilgili blokları, panoları ve derlenmiş sorgu/istem anlık görüntüsünü yeniden oluşturur. Anlık görüntü, OpenClaw'ın paylaşılan SQLite plugin durumunda kalıcılaştırılır ve eşzamanlı istem izdüşümü için bellekte tutulur; kasada önbellek dosyaları oluşturmaz.

`render.createDashboards` etkinse derleme, rapor sayfalarını da yeniler.

### `wiki lint`

Kasayı denetler ve aşağıdakileri kapsayan bir rapor yazar:

- yapısal sorunlar (bozuk bağlantılar, eksik/yinelenen kimlikler, eksik sayfa türü veya başlığı, geçersiz frontmatter)
- köken bilgisi eksiklikleri (eksik kaynak kimlikleri, eksik içe aktarma köken bilgisi)
- çelişkiler (işaretlenmiş çelişkiler, birbiriyle uyuşmayan iddialar)
- açık sorular
- düşük güven düzeyli sayfalar ve iddialar
- güncelliğini yitirmiş sayfalar ve iddialar

Bunu önemli wiki güncellemelerinden sonra çalıştırın.

### `wiki search <query>`

Wiki içeriğinde arama yapar. Davranış yapılandırmaya bağlıdır:

- `search.backend`: `shared` veya `local`
- `search.corpus`: `wiki`, `memory` veya `all`
- `--mode`: `auto`, `find-person`, `route-question`, `source-evidence` veya `raw-claim`

Wiki'ye özgü sıralama ve köken bilgisi için `wiki search` kullanın. Etkin bellek plugin'i paylaşılan arama sunuyorsa tek ve geniş kapsamlı bir ortak hatırlama geçişi için `openclaw memory search` tercih edin.

Arama modları:

- `find-person`: diğer adlar, kullanıcı adları, sosyal hesaplar, standart kimlikler ve kişi sayfaları
- `route-question`: danışılacak/en uygun kullanım ipuçları ve ilişki bağlamı
- `source-evidence`: kaynak sayfaları ve yapılandırılmış kanıt alanları
- `raw-claim`: iddia/kanıt meta verileriyle yapılandırılmış iddia metni

Örnekler:

```bash
openclaw wiki search "bgroux" --mode find-person
openclaw wiki search "Teams dağıtımını kim biliyor?" --mode route-question
openclaw wiki search "maintainer-whois" --mode source-evidence
openclaw wiki search "güçlü rota Teams" --mode raw-claim --json
```

Bir sonuç yapılandırılmış bir iddiayla eşleştiğinde metin çıktısı `Claim:` ve `Evidence:` satırlarını içerir. JSON çıktısı, aracı tarafında ayrıntılı inceleme için ayrıca `matchedClaimId`, `matchedClaimStatus`, `matchedClaimConfidence`, `evidenceKinds` ve `evidenceSourceIds` değerlerini sunar.

### `wiki get <lookup>`

Bir wiki sayfasını kimliğe veya göreli yola göre okur.

```bash
openclaw wiki get entity.alpha
openclaw wiki get syntheses/alpha-summary.md --from 1 --lines 80
```

### `wiki apply`

Serbest biçimli sayfa düzenlemesi yapmadan sınırlı değişiklikler uygular:

- `apply synthesis <title>`: yönetilen özet gövdesine sahip bir sentez sayfası oluşturur veya yeniler
- `apply metadata <lookup>`: mevcut bir sayfadaki meta verileri günceller

Her ikisi de `--source-id`, `--contradiction`, `--question` (her biri yinelenebilir), `--confidence <n>` (0-1) ve `--status <status>` kabul eder. `apply metadata`, saklanan bir güven değerini kaldırmak için ayrıca `--clear-confidence` kabul eder. Yönetilen ve oluşturulmuş blokları bozmadan wiki sayfalarını geliştirmek için desteklenen yöntem budur.

### `wiki bridge import`

Etkin bellek plugin'indeki herkese açık bellek yapıtlarını köprü destekli kaynak sayfalarına aktarır. En son dışa aktarılan bellek yapıtlarını wiki kasasına çekmek için bunu `bridge` modunda kullanın.

Etkin köprü yapıtı okumalarında CLI, çalışma zamanı bellek plugin'i bağlamını kullanması için içe aktarmayı Gateway RPC üzerinden yönlendirir. Köprü içe aktarımları devre dışıysa veya yapıt okumaları kapalıysa komut, yerel/çevrimdışı sıfır içe aktarma davranışını korur. İçe aktarma sonrasındaki dizin yenilemesi `ingest.autoCompile` tarafından denetlenir.

### `wiki unsafe-local import`

`unsafe-local` modunda açıkça yapılandırılmış yerel yollardan (`unsafeLocal.paths`) içe aktarır. Bilinçli olarak deneyseldir ve yalnızca aynı makinede çalışır. İçe aktarma sonrasındaki dizin yenilemesi `ingest.autoCompile` tarafından denetlenir.

### `wiki chatgpt import`

Bir ChatGPT dışa aktarımını taslak wiki kaynak sayfalarına aktarır.

```bash
openclaw wiki chatgpt import --export ./chatgpt-export
openclaw wiki chatgpt import --export ./conversations.json --dry-run
```

| Bayrak              | Varsayılan    | Açıklama                                                   |
| ----------------- | ---------- | ------------------------------------------------------------- |
| `--export <path>` | (gerekli) | ChatGPT dışa aktarma dizini veya `conversations.json` yolu.        |
| `--dry-run`       | `false`    | Sayfaları yazmadan oluşturulan/güncellenen/atlanan sayılarını önizler. |

Herhangi bir sayfayı değiştiren deneme dışı bir içe aktarma, geri alma için gereken ve özette yazdırılan bir içe aktarma çalıştırma kimliği kaydeder.

### `wiki chatgpt rollback <run-id>`

Daha önce uygulanmış bir ChatGPT içe aktarma çalıştırmasını geri alır; oluşturduğu sayfaları kaldırır ve üzerine yazdığı sayfaları geri yükler. Çalıştırma zaten geri alınmışsa işlem yapmaz (ve `alreadyRolledBack` bildirir).

### `wiki obsidian ...`

Obsidian uyumlu modda çalışan kasalar için Obsidian yardımcı komutları: `status`, `search`, `open`, `command`, `daily`. `obsidian.useOfficialCli` etkinken bunlar, `PATH` üzerinde resmî `obsidian` CLI'ı gerektirir.

Yapılandırma doğrulaması, `vault.scope`, `agent` olduğunda
`obsidian.useOfficialCli: true` değerini reddeder; çünkü `obsidian.vaultName` aracı başına bir eşleme değil,
tek bir genel ayardır. Obsidian uyumlu Markdown oluşturma kullanılabilir
kalmaya devam eder.

## Pratik kullanım kılavuzu

- Köken bilgisi ve sayfa kimliği önemli olduğunda `wiki search` + `wiki get` kullanın.
- Yönetilen ve oluşturulmuş bölümleri elle düzenlemek yerine `wiki apply` kullanın.
- Çelişkili veya düşük güven düzeyli içeriğe güvenmeden önce `wiki lint` kullanın.
- Yeni panoları ve derlenmiş özetleri hemen istediğinizde toplu içe aktarmalardan veya kaynak değişikliklerinden sonra `wiki compile` kullanın.
- Bir veri kataloğu, belge dışa aktarımı veya aracı zenginleştirme işlem hattı zaten OKF Markdown paketleri üretiyorsa `wiki okf import` kullanın.
- Köprü modu yeni dışa aktarılan bellek yapıtlarına bağlı olduğunda `wiki bridge import` kullanın.

## Yapılandırma bağlantıları

`openclaw wiki` davranışı aşağıdakiler tarafından belirlenir:

- `plugins.entries.memory-wiki.config.vaultMode`
- `plugins.entries.memory-wiki.config.vault.scope`
- `plugins.entries.memory-wiki.config.vault.path`
- `plugins.entries.memory-wiki.config.search.backend`
- `plugins.entries.memory-wiki.config.search.corpus`
- `plugins.entries.memory-wiki.config.bridge.*`
- `plugins.entries.memory-wiki.config.obsidian.*`
- `plugins.entries.memory-wiki.config.ingest.autoCompile`
- `plugins.entries.memory-wiki.config.render.*`
- `plugins.entries.memory-wiki.config.context.includeCompiledDigestPrompt`

Tam yapılandırma modeli için [Memory Wiki plugin'i](/tr/plugins/memory-wiki) bölümüne bakın.

## İlgili

- [CLI referansı](/tr/cli)
- [Bellek wiki'si](/tr/plugins/memory-wiki)
