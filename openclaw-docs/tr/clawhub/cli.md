---
read_when:
    - ClawHub CLI'yi Kullanma
    - Yükleme, güncelleme veya yayımlama sorunlarını ayıklama
summary: 'CLI başvurusu: komutlar, bayraklar, yapılandırma ve kilit dosyası davranışı.'
x-i18n:
    generated_at: "2026-07-26T23:33:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eba91a83c5542c4b570bd22a526911633e43d0b4e921c013e6fd29451193f2a7
    source_path: clawhub/cli.md
    workflow: 16
---

# CLI

CLI paketi: `clawhub`, ikili dosya: `clawhub`.

npm veya pnpm ile genel olarak yükleyin:

```bash
npm i -g clawhub
# veya
pnpm add -g clawhub
```

Ardından doğrulayın:

```bash
clawhub --help
clawhub login
clawhub whoami
```

## Genel bayraklar

- `--workdir <dir>`: çalışma dizini (varsayılan: cwd; yapılandırılmışsa Clawdbot çalışma alanına geri döner)
- `--dir <dir>`: workdir altındaki yükleme dizini (varsayılan: `skills`)
- `--site <url>`: tarayıcı oturum açma işleminin temel URL'si (varsayılan: `https://clawhub.ai`)
- `--registry <url>`: API temel URL'si (varsayılan: keşfedilen, aksi hâlde `https://clawhub.ai`)
- `--no-input`: istemleri devre dışı bırakır

Eşdeğer ortam değişkenleri:

- `CLAWHUB_SITE` (eski `CLAWDHUB_SITE`)
- `CLAWHUB_REGISTRY` (eski `CLAWDHUB_REGISTRY`)
- `CLAWHUB_WORKDIR` (eski `CLAWDHUB_WORKDIR`)

### HTTP proxy'si

CLI, kurumsal proxy'lerin veya kısıtlı ağların arkasındaki sistemler için
standart HTTP proxy ortam değişkenlerine uyar:

- `HTTPS_PROXY` / `https_proxy`
- `HTTP_PROXY` / `http_proxy`
- `NO_PROXY` / `no_proxy`

Bu değişkenlerden herhangi biri ayarlandığında CLI, giden istekleri belirtilen
proxy üzerinden yönlendirir. HTTPS istekleri için `HTTPS_PROXY`, düz HTTP için
`HTTP_PROXY` kullanılır. Belirli ana makineler veya etki alanları için proxy'yi
atlamak üzere `NO_PROXY` / `no_proxy` değerine uyulur.

Doğrudan giden bağlantıların engellendiği sistemlerde bu gereklidir
(ör. Docker kapsayıcıları, yalnızca proxy üzerinden internete erişebilen Hetzner VPS,
kurumsal güvenlik duvarları).

Örnek:

```bash
export HTTPS_PROXY=http://proxy.example.com:3128
export NO_PROXY=localhost,127.0.0.1
clawhub search "my query"
```

Hiçbir proxy değişkeni ayarlanmadığında davranış değişmez (doğrudan bağlantılar).

## Yapılandırma dosyası

API belirtecinizi ve önbelleğe alınmış kayıt defteri URL'sini saklar.

- macOS: `~/Library/Application Support/clawhub/config.json`
- Linux/XDG: `$XDG_CONFIG_HOME/clawhub/config.json` veya `~/.config/clawhub/config.json`
- Windows: `%APPDATA%\\clawhub\\config.json`
- Eski geri dönüş: `clawhub/config.json` henüz mevcut değil ancak `clawdhub/config.json` mevcutsa CLI eski yolu yeniden kullanır
- geçersiz kılma: `CLAWHUB_CONFIG_PATH` (eski `CLAWDHUB_CONFIG_PATH`)

## Komutlar

### `login` / `auth login`

- Varsayılan: tarayıcıyı `<site>/cli/auth` adresinde açar ve geri döngü geri çağrısı aracılığıyla tamamlar.
- Başsız: `clawhub login --token clh_...`
- Uzak/başsız etkileşimli: `clawhub login --device` bir kod yazdırır ve siz `<site>/cli/device` adresinde yetkilendirirken bekler.

### `whoami`

- Saklanan belirteci `/api/v1/whoami` aracılığıyla doğrular.

### `token`

- Saklanan API belirtecini stdout'a yazdırır.
- Yerel oturum açma belirtecini CI gizli dizi kurulum komutlarına aktarmak için kullanışlıdır.

### `star <skill>` / `unstar <skill>`

- Yer İmlerinize bir skill ekler veya kaldırır. Komut adları uyumluluk için
  `star` ve `unstar` olarak kalır.
- `POST /api/v1/stars/<slug>` ve `DELETE /api/v1/stars/<slug>` çağrılarını yapar.
- `--yes` onayı atlar.

### `search <query...>`

- `/api/v1/search?q=...` çağrısını yapar.
- Çıktı; skill kısa adını, sahip kullanıcı adını, görünen adı ve alaka puanını içerir.
- Arama, indirme popülerliğinden önce kısa ad/ad belirteçlerinin tam eşleşmelerini tercih eder. `map` gibi bağımsız bir kısa ad belirteci, `amap` içindeki alt diziye kıyasla `personal-map` ile daha güçlü eşleşir.
- Popülerlik küçük bir sıralama önceliğidir; en üstte yer alma garantisi değildir.
- Bir skill görünmesi gerektiği hâlde görünmüyorsa meta veriyi yeniden adlandırmadan önce, sahip tarafından görülebilen moderasyon tanılamalarını denetlemek için oturum açmış durumdayken `clawhub inspect @owner/slug` komutunu çalıştırın.

### `explore`

- En yeni skill'leri `/api/v1/skills?limit=...&sort=createdAt` aracılığıyla listeler (`createdAt` azalan düzende sıralanır).
- Bayraklar:
  - `--limit <n>` (1-200, varsayılan: 25)
  - `--sort newest|updated|rating|downloads|trending` (varsayılan: en yeni). Eski yükleme sıralama diğer adları uyumluluk için çalışmaya devam eder.
  - `--json` (makine tarafından okunabilir çıktı)
- Çıktı: `<slug>  v<version>  <age>  <summary>` (özet 50 karaktere kısaltılır).

### `inspect @owner/slug`

- Yüklemeden skill meta verilerini ve sürüm dosyalarını getirir.
- `--version <version>`: belirli bir sürümü inceleyin (varsayılan: en son).
- `--tag <tag>`: etiketlenmiş bir sürümü inceleyin (ör. `latest`).
- `--versions`: sürüm geçmişini listeleyin (ilk sayfa).
- `--limit <n>`: listelenecek azami sürüm sayısı (1-200).
- `--files`: seçilen sürümün dosyalarını listeleyin.
- `--file <path>`: ham dosya baytlarını getirin (10MB sınırı).
- `--json`: makine tarafından okunabilir çıktı; `--file`, kullanılabilir olduğunda tam baytları base64 ve UTF-8 metni olarak içerir.

### `install @owner/slug`

- Adlandırılan sahip ve skill için en son sürümü çözümler.
- Zip dosyasını `/api/v1/download` aracılığıyla indirir.
- `<workdir>/<dir>/<slug>` içine ayıklar.
- Sabitlenmiş skill'lerin üzerine yazmayı reddeder; önce `clawhub unpin <skill>` komutunu çalıştırın.
- Şunları yazar:
  - `<workdir>/.clawhub/lock.json` (eski `.clawdhub`)
  - `<skill>/.clawhub/origin.json` (eski `.clawdhub`)

### `uninstall <skill>`

- `<workdir>/<dir>/<slug>` öğesini kaldırır ve kilit dosyası girdisini siler.
- Geçerli yükleme sayılarının devre dışı bırakılabilmesi için oturum açıkken
  mümkün olan en iyi şekilde telemetri gönderir.
- Etkileşimli: onay ister.
- Etkileşimsiz (`--no-input`): `--yes` gerektirir.

### `list`

- `<workdir>/.clawhub/lock.json` dosyasını okur (eski `.clawdhub`).
- `clawhub pin` ile dondurulan skill'lerin yanında, isteğe bağlı neden dâhil olmak üzere `pinned` gösterir.

### `pin <skill>`

- Yüklü bir skill'i kilit dosyasında sabitlenmiş olarak işaretler.
- `--reason <text>`, skill'in neden dondurulduğunu kaydeder.
- Sabitlenmiş skill'ler `update --all` tarafından atlanır ve doğrudan `update <skill>` tarafından reddedilir.
- Yerel baytların yanlışlıkla değiştirilememesi için sabitlenmiş skill'ler `install --force` işlemini de reddeder.

### `unpin <skill>`

- Gelecekteki güncellemelerin skill'i değiştirebilmesi için yüklü bir skill'in kilit dosyası sabitlemesini kaldırır.

### `update [@owner/slug]` / `update --all`

- Yerel dosyalardan parmak izi hesaplar.
- Parmak izi bilinen bir sürümle eşleşirse istem gösterilmez.
- Parmak izi eşleşmezse:
  - varsayılan olarak reddeder
  - `--force` ile üzerine yazar (veya etkileşimliyse istem gösterir)
- Sabitlenmiş skill'ler `--force` tarafından hiçbir zaman güncellenmez.
- `update <skill>`, sabitlenmiş skill'ler için hemen başarısız olur ve önce `clawhub unpin <skill>` komutunu çalıştırmanızı söyler.
- `update --all`, sabitlenmiş kısa adları atlar ve nelerin donmuş kaldığının özetini yazdırır.

### `skill publish <path>`

- Yerel paket parmak izini ClawHub ile karşılaştırır ve içerik zaten
  yayımlanmışsa başarıyla çıkar.
- Yeni skill'ler varsayılan olarak `1.0.0`; değiştirilmiş skill'ler ise varsayılan olarak sonraki yama
  sürümünü kullanır.
- `--version <version>` açıkça bir sürüm seçer ve içerik mevcut bir sürümle
  eşleşse bile yayımlar.
- `--dry-run`, yüklemeden yayımlama işlemini çözümler; `--json`
  makine tarafından okunabilir bir sonuç yazdırır.
- `--owner <handle>`, aktörün yayımcı erişimi olduğunda bir kuruluş/kullanıcı
  yayımcı kullanıcı adı altında yayımlar.
- `--migrate-owner`, yeni bir sürüm yayımlarken mevcut bir skill'i
  `--owner` konumuna taşır. Her iki yayımcı için de yönetici/sahip erişimi gerektirir.
- Sahip ve inceleme davranışı `docs/publishing.md` içinde açıklanmaktadır.
- Bir skill'i yayımlamak, onun ClawHub üzerinde `MIT-0` kapsamında kullanıma sunulması anlamına gelir.
- Yayımlanan skill'ler atıf gerektirmeden ücretsiz olarak kullanılabilir, değiştirilebilir ve yeniden dağıtılabilir.
- ClawHub ücretli skill'leri veya skill başına fiyatlandırmayı desteklemez.
- Eski diğer ad: `publish <path>`.

```bash
clawhub skill publish ./my-skill --dry-run
clawhub skill publish ./my-skill
clawhub skill publish ./my-skill --version 2.0.0
```

#### GitHub Actions

ClawHub'ın yeniden kullanılabilir
[`skill-publish.yml`](https://github.com/openclaw/clawhub/blob/main/.github/workflows/skill-publish.yml)
iş akışı, bir `skill_path` için veya `root` altındaki her doğrudan skill
klasörü için (varsayılan: `skills`) `skill publish` çağrısını yapar. Değişmeyen skill'leri atlar ve
aynı otomatik yama sürümü davranışını kullanır.

Belirteç olmadan önizleme yapmak için `dry_run: true` ayarlayın. Gerçek yayımlama işlemleri
`clawhub_token` gizli dizisini gerektirir.

### `sync`

- Geçerli workdir'i, yapılandırılmış skills dizinini ve `SKILL.md` veya
  `skill.md` içeren yerel skill klasörleri için tüm `--root <dir>`
  klasörlerini tarar.
- Her yerel skill parmak izini ClawHub ile karşılaştırır ve yalnızca yeni veya
  değiştirilmiş skill'leri yayımlar.
- Yeni skill'ler `1.0.0` olarak yayımlanır; değiştirilmiş skill'ler varsayılan olarak sonraki yama sürümünü
  yayımlar. Daha büyük bir semver adımıyla ilerlemesi gereken güncelleme grupları için
  `--bump minor|major` kullanın.
- `--dry-run`, yüklemeden yayımlama planını gösterir; `--json`
  makine tarafından okunabilir bir plan yazdırır.
- `--all`, her yeni veya değiştirilmiş skill'i istem göstermeden yayımlar.
  `--all` olmadan etkileşimli terminaller, yayımlanacak skill'leri seçmenize olanak tanır.
- `--owner <handle>`, aktörün yayımcı erişimi olduğunda bir kuruluş/kullanıcı
  yayımcı kullanıcı adı altında yayımlar.
- `sync` yalnızca tek yönlü yayımlama yapar. Yüklemez, güncellemez, indirmez
  veya yükleme/indirme telemetrisi bildirmez.

```bash
clawhub sync --all --dry-run
clawhub sync --all
clawhub sync --root ./skills --owner openclaw --bump minor
```

### `scan --slug <slug>`

- `clawhub login` gerektirir.
- `POST /api/v1/skills/-/scan` aracılığıyla ClawHub ClawScan'i çalıştırır, ardından tarama son duruma ulaşana kadar yoklar.
- Taramalar eşzamansızdır ve tamamlanması zaman alabilir. Kuyruktayken terminal döndürücüsü, geçerli öncelikli tarama konumunu ve önde kaç tarama olduğunu gösterir.
- Yayımlanmış taramalar sahiplik veya yayımcı yönetim erişimi gerektirir. Moderatörler/yöneticiler aynı arka ucu `clawhub-admin` aracılığıyla kullanabilir.
- `--update` yalnızca `--slug` ile geçerlidir; başarılı yayımlanmış tarama sonuçlarını seçilen sürüme geri yazar.
- `--output <file.zip>`, tam rapor arşivini `manifest.json`, `clawscan.json`, `skillspector.json`, `static-analysis.json`, `virustotal.json` ve `README.md` ile indirir.
- `--json`, otomasyon için tam yoklama yanıtını yazdırır.
- Yerel yol taramaları artık desteklenmemektedir. Yeni bir sürüm yükleyin, ardından gönderilen bu sürümün saklanan tarama sonuçlarını almak için `scan download` kullanın.

```bash
clawhub scan --slug gifgrep
clawhub scan --slug gifgrep --version 1.2.3
clawhub scan --slug gifgrep --update --output report.zip
```

### `scan download <name>`

- `clawhub login` gerektirir.
- ClawHub güvenlik denetimleri tarafından engellenen veya gizlenen sürümler dâhil, gönderilmiş bir skill veya Plugin sürümü için saklanan tarama raporu ZIP dosyasını indirir.
- Skill indirmeleri skill kısa adını kullanır ve varsayılan olarak `--kind skill` değerini alır.
- Plugin indirmeleri paket adını kullanır ve `--kind plugin` gerektirir.
- Yazarların ClawHub tarafından engellenen gönderimin tam sürümünü incelemesi için `--version` gereklidir.
- `--output <file.zip>` hedef yolu seçer.

```bash
clawhub scan download gifgrep --version 1.2.3
clawhub scan download @scope/demo --version 2.0.0 --kind plugin --output report.zip
```

#### GitHub Actions

ClawHub, skill depoları ve katalog depoları için
[`/.github/workflows/skill-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/skill-publish.yml)
adresinde resmî, yeniden kullanılabilir bir iş akışı sunar.

Tipik katalog kurulumu:

```yaml
name: Skill Publish

on:
  pull_request:
  workflow_dispatch:

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

Notlar:

- Katalog depolarında `root` varsayılan olarak `skills` değerini alır.
- Tek bir skill klasörünü işlemek için `skill_path: skills/review-helper` iletin.
- `owner`, CLI `--owner` bayrağına karşılık gelir; kimliği doğrulanmış kullanıcı olarak yayımlamak için bunu atlayın.
- V1 skill yayımlama `clawhub_token` kullanır; GitHub OIDC güvenilir yayımlama şimdilik yalnızca paketler içindir.

### `delete <skill>`

- `--version` olmadan bir skill'i geçici olarak siler (sahip, moderatör veya yönetici).
- `DELETE /api/v1/skills/{slug}` çağrısını yapar.
- Sahibin başlattığı geçici silme işlemleri kısa adı 30 gün boyunca rezerve eder; komut sona erme zamanını yazdırır.
- `--version <version>`, sahip olunan ve en son olmayan tek bir sürümü, hata durumunda kapalı davranan sürüme özgü bir rota üzerinden geri çeker. Sürüm numarası rezerve kalır ve farklı içerikle yeniden yayımlanamaz. Geçerli en son sürümü silmeden önce yerine geçecek bir sürüm yayımlayın. Platform personeli yalnızca bu sürüme yönelik akışta sahiplik denetimini atlayamaz.
- `--reason <text>`, skill'in tamamının geçici olarak silinmesine ve denetim günlüğüne bir moderasyon notu kaydeder.
- `--note <text>`, `--reason` için bir diğer addır.
- `--yes` onayı atlar.

### `undelete <skill>`

- Gizli bir skill'i geri yükler (sahip, moderatör veya yönetici).
- `POST /api/v1/skills/{slug}/undelete` çağrısını yapar.
- `--version <version>`, yalnızca aynı sahip aktör tarafından daha önce geri çekilmiş ve saklanmış olan tam yapıyı geri yükler. Geri yüklenen sürümü en son sürüm yapmaz veya kaldırılan etiketleri yeniden oluşturmaz.
- Sürüm geri yükleme, `POST /api/v1/skills/{slug}/versions/{version}/restore` çağrısını yapar.
- `--reason <text>`, skill'e ve denetim günlüğüne bir moderasyon notu kaydeder.
- `--note <text>`, `--reason` için bir diğer addır.
- `--yes` onayı atlar.

### `hide <skill>`

- Bir skill'i gizler (sahip, moderatör veya yönetici).
- `delete` için diğer ad.

### `unhide <skill>`

- Bir skill'in gizliliğini kaldırır (sahip, moderatör veya yönetici).
- `undelete` için diğer ad.

### `skill rename <skill> <new-name>`

- Sahip olunan bir skill'i yeniden adlandırır ve önceki kısa adı yönlendirme diğer adı olarak tutar.
- `POST /api/v1/skills/{slug}/rename` çağrısını yapar.
- `--yes` onayı atlar.

### `skill merge <source> <target>`

- Sahip olunan bir skill'i, sahip olunan başka bir skill ile birleştirir.
- Kaynak kısa ad artık herkese açık olarak listelenmez ve hedefe yönlendiren bir diğer ada dönüşür.
- `POST /api/v1/skills/{sourceSlug}/merge` çağrısını yapar.
- `--yes` onayı atlar.

### `transfer`

- Sahiplik aktarımı iş akışı.
- Kullanıcı tanıtıcılarına yapılan aktarımlar, alıcının kabul edeceği bekleyen bir istek oluşturur.
- Kuruluş/yayımcı tanıtıcılarına yapılan aktarımlar, yalnızca aktör hem mevcut sahibin hem de hedef yayımcının yönetici erişimine sahip olduğunda hemen uygulanır.
- Alt komutlar:
  - `transfer request <skill> <handle> [--message "..."] [--yes]`
  - `transfer list [--outgoing]`
  - `transfer accept <skill> [--yes]`
  - `transfer reject <skill> [--yes]`
  - `transfer cancel <skill> [--yes]`
- Uç noktalar:
  - `POST /api/v1/skills/{slug}/transfer`
  - `POST /api/v1/skills/{slug}/transfer/accept`
  - `POST /api/v1/skills/{slug}/transfer/reject`
  - `POST /api/v1/skills/{slug}/transfer/cancel`
  - `GET /api/v1/transfers/incoming`
  - `GET /api/v1/transfers/outgoing`

### `package explore [query...]`

- `GET /api/v1/packages` ve `GET /api/v1/packages/search` aracılığıyla birleşik paket kataloğuna göz atar veya katalogda arama yapar.
- Bunu Plugin'ler ve diğer paket ailesi girdileri için kullanın; üst düzey `search` skill arama yüzeyi olarak kalır.
- Bayraklar:
  - `--family skill|code-plugin|bundle-plugin`
  - `--official`
  - `--executes-code`
  - `--target <target>`, `--os <os>`, `--arch <arch>`, `--libc <libc>`
  - `--requires-browser`, `--requires-desktop`, `--requires-native-deps`
  - `--requires-external-service`, `--external-service <name>`
  - `--binary <name>`, `--os-permission <name>`
  - `--artifact-kind legacy-zip|npm-pack`
  - `--npm-mirror`
  - `--limit <n>` (1-100, varsayılan: 25)
  - `--json`

Örnekler:

```bash
clawhub package explore --family code-plugin
clawhub package explore --family code-plugin --os darwin --requires-desktop
clawhub package explore --family code-plugin --artifact-kind npm-pack
clawhub package explore --npm-mirror
clawhub package explore episodic-claw --family code-plugin
```

### `package inspect <name>`

- Kurulum yapmadan paket meta verilerini getirir.
- Bunu Plugin meta verileri, uyumluluk, doğrulama, kaynak ve sürüm/dosya incelemesi için kullanın.
- `--version <version>`: belirli bir sürümü inceler (varsayılan: en son).
- `--tag <tag>`: etiketli bir sürümü inceler (ör. `latest`).
- `--versions`: sürüm geçmişini listeler (ilk sayfa).
- `--limit <n>`: listelenecek azami sürüm sayısı (1-100).
- `--files`: seçilen sürümün dosyalarını listeler.
- `--file <path>`: sınırlı bir UTF-8 metin önizlemesi getirir (200KB sınırı).
- `--json`: makine tarafından okunabilir çıktı.

### `package download <name>`

- Bir paket sürümünü `GET /api/v1/packages/{name}/versions/{version}/artifact` üzerinden çözümler.
- Yapıyı çözümleyicinin `downloadUrl` konumundan indirir.
- Tüm yapılar için ClawHub SHA-256 değerini doğrular.
- ClawPack npm-pack yapıları için ayrıca npm `sha512` bütünlüğünü, npm shasum değerini ve tarball dosyasının `package.json` adını/sürümünü doğrular.
- Eski ZIP sürümleri, eski ZIP rotası üzerinden indirilir.
- Bayraklar:
  - `--version <version>`: belirli bir sürümü indirir.
  - `--tag <tag>`: etiketli bir sürümü indirir (varsayılan: `latest`).
  - `-o, --output <path>`: çıktı dosyası veya dizini.
  - `--force`: mevcut bir çıktı dosyasının üzerine yazar.
  - `--json`: makine tarafından okunabilir çıktı.

Örnekler:

```bash
clawhub package download @openclaw/example-plugin --tag latest
clawhub package download @openclaw/example-plugin --version 1.2.3 -o artifacts/
```

### `package verify <file>`

- Yerel bir yapı için ClawHub SHA-256, npm `sha512` bütünlüğü ve npm shasum değerini hesaplar.
- `--package` ile beklenen meta verileri ClawHub'dan çözümler ve yerel dosyayı yayımlanmış yapı meta verileriyle karşılaştırır.
- Doğrudan özet bayraklarıyla ağ sorgusu yapmadan doğrulama gerçekleştirir.
- Bayraklar:
  - `--package <name>`: beklenen yapı meta verilerinin çözümleneceği paket adı.
  - `--version <version>` veya `--tag <tag>`: beklenen paket sürümü.
  - `--sha256 <hex>`: beklenen ClawHub SHA-256.
  - `--npm-integrity <sri>`: beklenen npm bütünlüğü.
  - `--npm-shasum <sha1>`: beklenen npm shasum.
  - `--json`: makine tarafından okunabilir çıktı.

Örnekler:

```bash
clawhub package verify ./example-plugin-1.2.3.tgz --package @openclaw/example-plugin --version 1.2.3
clawhub package verify ./example-plugin-1.2.3.tgz --sha256 <hex>
```

### `package validate <source>`

- ClawHub CLI ile birlikte sunulan Plugin Inspector'ı yerel bir Plugin paket klasöründe çalıştırır.
- Yerel bir OpenClaw çalışma kopyasını bulmadan veya içe aktarmadan varsayılan olarak çevrimdışı/statik doğrulama yapar.
- Kritik uyumluluk hataları sıfırdan farklı bir kodla çıkar. Yalnızca uyarı niteliğindeki bulgular yazdırılır ancak sıfır koduyla çıkılır.
- Bayraklar:
  - `--out <dir>`: Plugin Inspector raporlarını bu dizine yazar.
  - `--openclaw <path>`: açıkça belirtilen yerel bir OpenClaw çalışma kopyasına göre inceleme yapar.
  - `--runtime`: çalışma zamanı yakalamayı etkinleştirir; Plugin kodunu içe aktarır.
  - `--allow-execute`: yalıtılmış bir çalışma alanında çalışma zamanı yakalamaya izin verir.
  - `--no-mock-sdk`: çalışma zamanı yakalama sırasında taklit OpenClaw SDK'sını devre dışı bırakır.
  - `--json`: makine tarafından okunabilir çıktı.

Örnek:

```bash
clawhub package validate ./example-plugin
```

Doğrulama bir paket, bildirim, SDK içe aktarma veya yapı bulgusu bildirirse [Plugin doğrulama düzeltmeleri](/tr/clawhub/plugin-validation-fixes) bölümüne bakın ve ardından komutu yeniden çalıştırın.

### `package delete <name>`

- `--version` olmadan bir paketi ve tüm sürümlerini geçici olarak siler.
- `--version <version>`, sahip olunan ve en son olmayan tek bir sürümü, hata durumunda kapalı davranan sürüme özgü bir rota üzerinden geri çeker. Sürüm numarası rezerve kalır ve farklı içerikle yeniden yayımlanamaz. Geçerli en son sürümü silmeden önce yerine geçecek bir sürüm yayımlayın. Yalnızca bu sürüme yönelik akış, paket sahibini veya kuruluş yayımcısının yöneticisini gerektirir; platform personeli paket sahipliğini atlayamaz.
- Paketin tamamının geçici olarak silinmesi paket sahibini, kuruluş yayımcısının sahibini/yöneticisini, platform moderatörünü veya platform yöneticisini gerektirir.
- Bayraklar:
  - `--version <version>`: en son olmayan tek bir sürümü geri çeker.
  - `--yes`: onayı atlar.
  - `--json`: makine tarafından okunabilir çıktı.

Örnek:

```bash
clawhub package delete @openclaw/example-plugin --yes
clawhub package delete @openclaw/example-plugin --version 1.2.3 --yes
```

### `package undelete <name>`

- Geçici olarak silinmiş bir paketi ve sürümlerini geri yükler.
- Paket sahibini, kuruluş yayımcısının sahibini/yöneticisini, platform moderatörünü veya platform yöneticisini gerektirir.
- `POST /api/v1/packages/{name}/undelete` çağrısını yapar.
- `--version <version>`, yalnızca aynı sahip aktör tarafından daha önce geri çekilmiş ve saklanmış olan tam sürümü geri yükler. Sürümü en son sürüm yapmaz veya kaldırılan paket etiketlerini/dist-tag'leri yeniden oluşturmaz.
- Sürüm geri yükleme, `POST /api/v1/packages/{name}/versions/{version}/restore` çağrısını yapar.
- Bayraklar:
  - `--version <version>`: sahibi tarafından geri çekilmiş tek bir sürümü geri yükler.
  - `--yes`: onayı atlar.
  - `--json`: makine tarafından okunabilir çıktı.

Örnek:

```bash
clawhub package undelete @openclaw/example-plugin --yes
```

### `package transfer <name>`

- Bir paketi başka bir yayıncıya aktarır.
- Bir platform yöneticisi tarafından gerçekleştirilmediği sürece hem mevcut paket sahibine hem de hedef
  yayıncıya yönetici erişimi gerektirir.
- Kapsamlı paket adları, eşleşen kapsam sahibine aktarılmalıdır.
- `POST /api/v1/packages/{name}/transfer` çağrısını yapar.
- Bayraklar:
  - `--to <owner>`: hedef yayıncı tanıtıcısı.
  - `--reason <text>`: isteğe bağlı denetim gerekçesi.
  - `--json`: makine tarafından okunabilir çıktı.

Örnek:

```bash
clawhub package transfer @openclaw/example-plugin --to openclaw
```

### `package report`

- Bir paketi moderatörlere bildirmek için kimliği doğrulanmış komut.
- `POST /api/v1/packages/{name}/report` çağrısını yapar.
- Bildirimler paket düzeyindedir, isteğe bağlı olarak bir sürümle ilişkilendirilebilir ve incelenmek üzere
  moderatörlere görünür hâle gelir.
- Bildirimler tek başına paketleri otomatik olarak gizlemez veya indirmeleri engellemez.
- Bayraklar:
  - `--version <version>`: bildirime eklenecek isteğe bağlı paket sürümü.
  - `--reason <text>`: gerekli bildirim gerekçesi.
  - `--json`: makine tarafından okunabilir çıktı.

Örnek:

```bash
clawhub package report @openclaw/example-plugin --version 1.2.3 --reason "şüpheli yerel yük"
```

### `package moderation-status`

- Paket moderasyon görünürlüğünü kontrol etmek için sahip komutu.
- `GET /api/v1/packages/{name}/moderation` çağrısını yapar.
- Mevcut paket tarama durumunu, açık bildirim sayısını, en son sürümün manuel
  moderasyon durumunu, indirme engeli durumunu ve moderasyon gerekçelerini gösterir.
- Bayraklar:
  - `--json`: makine tarafından okunabilir çıktı.

Örnek:

```bash
clawhub package moderation-status @openclaw/example-plugin
```

### `package readiness <name>`

- Bir paketin OpenClaw tarafından gelecekte kullanılmaya hazır olup olmadığını kontrol eder.
- `GET /api/v1/packages/{name}/readiness` çağrısını yapar.
- Resmî durum, ClawPack kullanılabilirliği, yapıt özeti,
  kaynak kökeni, OpenClaw uyumluluğu, ana makine hedefleri, ortam meta verileri
  ve tarama durumuna ilişkin engelleri bildirir.
- Bayraklar:
  - `--json`: makine tarafından okunabilir çıktı.

Örnek:

```bash
clawhub package readiness @openclaw/example-plugin
```

### `package migration-status <name>`

- Paketle birlikte sunulan bir OpenClaw plugin'inin yerini alabilecek bir paket için
  operatör odaklı geçiş durumunu gösterir.
- `package readiness` ile aynı hesaplanan hazırlık uç noktasını çağırır ancak
  geçiş odaklı durumu, en son sürümü, resmî paket durumunu, kontrolleri ve
  engelleri yazdırır.
- Bayraklar:
  - `--json`: makine tarafından okunabilir çıktı.

Örnek:

```bash
clawhub package migration-status @openclaw/example-plugin
```

### `publisher create <handle>`

- Kimliği doğrulanmış kullanıcının sahip olduğu bir kuruluş yayıncısı oluşturur.
- Tanıtıcı küçük harfe dönüştürülerek normalleştirilir ve `@` ile veya onsuz iletilebilir.
- Yeni oluşturulan kuruluş yayıncıları varsayılan olarak güvenilir/resmî değildir.
- Tanıtıcı mevcut bir yayıncı, kullanıcı veya ayrılmış rota tarafından zaten kullanılıyorsa başarısız olur.

```bash
clawhub publisher create opik --display-name "Opik"
```

### `package publish <source>`

- `POST /api/v1/packages` aracılığıyla bir kod plugin'i veya paket plugin'i yayımlar.
- `<source>` şunları kabul eder:
  - Yerel klasör yolu: `./my-plugin`
  - Yerel ClawPack npm-pack tarball dosyası: `./my-plugin-1.2.3.tgz`
  - GitHub deposu: `owner/repo` veya `owner/repo@ref`
  - GitHub URL'si: `https://github.com/owner/repo`
- Meta veriler `package.json`, `openclaw.plugin.json` ve
  `.codex-plugin/plugin.json`, `.claude-plugin/plugin.json` ve `.cursor-plugin/plugin.json`
  gibi gerçek OpenClaw paket işaretçilerinden otomatik olarak algılanır.
- `.tgz` kaynakları ClawPack olarak değerlendirilir. CLI, tam npm-pack
  baytlarını yükler ve ayıklanan `package/` içeriğini yalnızca doğrulama ve
  meta verileri önceden doldurmak için kullanır.
- Kod plugin'i klasörleri yüklemeden önce bir ClawPack npm tarball dosyası olarak paketlenir; böylece
  OpenClaw kurulumları tam yapıtı doğrulayabilir. Paket plugin'i klasörleri ise
  ayıklanan dosyalarla yayımlama yolunu kullanmaya devam eder.
- GitHub kaynaklarında kaynak atfı; depo, çözümlenen commit, ref ve alt yoldan otomatik olarak doldurulur.
- Yerel klasörlerde, origin uzak deposu GitHub'a işaret ettiğinde kaynak atfı yerel git'ten otomatik olarak algılanır.
- Harici kod plugin'leri `openclaw.compat.pluginApi` ve
  `openclaw.build.openclawVersion` değerlerini açıkça bildirmelidir.
  Üst düzey `package.json.version`, yayımlama doğrulaması için yedek olarak kullanılmaz.
- `--dry-run`, yükleme yapmadan çözümlenen yayımlama yükünün önizlemesini gösterir.
- `--json`, CI için makine tarafından okunabilir çıktı üretir.
- `--owner <handle>`, aktörün yayıncı erişimi olduğunda bir kullanıcı veya kuruluş yayıncı tanıtıcısı altında yayımlar.
- Kapsamlı paket adları seçilen sahiple eşleşmelidir. Bkz. `docs/publishing.md`.
- Mevcut bayraklar (`--family`, `--name`, `--version`, `--source-repo`, `--source-commit`, `--source-ref`, `--source-path`) geçersiz kılma olarak çalışmaya devam eder.
- Özel GitHub depoları `GITHUB_TOKEN` gerektirir.

```bash
clawhub package publish ./plugin.tgz --owner openclaw
```

#### Önerilen yerel akış

Canlı bir sürüm oluşturmadan önce çözümlenen paket meta verilerini ve
kaynak atfını doğrulayabilmek için önce `--dry-run` kullanın:

```bash
npm pack
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin --dry-run
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin
```

#### Yerel klasör akışı

Kod plugin'lerinde klasörden yayımlama, paket klasöründen bir ClawPack yapıtı
oluşturup yükler:

```bash
clawhub package publish ./my-plugin --family code-plugin --dry-run
clawhub package publish ./my-plugin --family code-plugin
```

#### `--family code-plugin` için minimum `package.json`

Harici kod plugin'leri `package.json` içinde az miktarda OpenClaw
meta verisine ihtiyaç duyar. Bu minimum bildirim başarılı bir yayımlama için yeterlidir:

```json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2"
    }
  }
}
```

Gerekli alanlar:

- `openclaw.compat.pluginApi`
- `openclaw.build.openclawVersion`

Notlar:

- `package.json.version` paket sürümünüzdür ancak OpenClaw
  uyumluluk/derleme doğrulaması için yedek olarak kullanılmaz.
- `openclaw.hostTargets` ve `openclaw.environment` isteğe bağlı meta verilerdir.
  ClawHub mevcut olduklarında bunları gösterebilir ancak yayımlama için gerekli değildir.
- `openclaw.compat.minGatewayVersion` ve
  `openclaw.build.pluginSdkVersion`, daha ayrıntılı uyumluluk meta verileri yayımlamak
  isterseniz kullanabileceğiniz isteğe bağlı ek alanlardır.
- Daha eski bir `clawhub` CLI sürümü kullanıyorsanız yükleme öncesinde
  yerel ön kontrollerin çalışması için yayımlamadan önce yükseltin.
- Doğrulama bir düzeltme kodu bildirirse
  [Plugin doğrulama düzeltmeleri](/tr/clawhub/plugin-validation-fixes) bölümüne bakın.

#### GitHub Actions

ClawHub ayrıca plugin depoları için
[`/.github/workflows/package-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/package-publish.yml)
adresinde resmî, yeniden kullanılabilir bir iş akışı sunar.

Tipik çağıran yapılandırması:

```yaml
name: Paket Yayımlama

on:
  pull_request:
  workflow_dispatch:
  push:
    tags:
      - "v*"

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch' || startsWith(github.ref, 'refs/tags/')
    permissions:
      contents: read
      id-token: write
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

Notlar:

- Yeniden kullanılabilir iş akışı, `source` için varsayılan olarak çağıran depoyu kullanır.
- Tek depolu çoklu projelerde, iş akışının plugin paket klasörünü yayımlaması için
  `source_path` iletin; örneğin `source_path: extensions/codex`.
- Yeniden kullanılabilir iş akışını kararlı bir etikete veya tam commit SHA'sına sabitleyin. Sürüm yayımlamayı `@main` üzerinden çalıştırmayın.
- CI'ın değişiklik oluşturmaması için `pull_request`, `dry_run: true` kullanmalıdır.
- Gerçek yayımlamalar `workflow_dispatch` veya etiket gönderimleri gibi güvenilir olaylarla sınırlandırılmalıdır.
- Gizli anahtar olmadan güvenilir yayımlama yalnızca `workflow_dispatch` üzerinde çalışır; etiket gönderimleri yine de `clawhub_token` gerektirir.
- İlk yayımlama, güvenilmeyen paketler veya acil durum yayımlamaları için `clawhub_token` kullanılabilir durumda tutulmalıdır.
- İş akışı JSON sonucunu bir yapıt olarak yükler ve iş akışı çıktıları olarak sunar.

### `package trusted-publisher get <name>`

- Bir paket için GitHub Actions güvenilir yayıncı yapılandırmasını gösterir.
- Depoyu, iş akışı dosya adını ve isteğe bağlı ortam sabitlemesini
  doğrulamak için yapılandırmayı ayarladıktan sonra bunu kullanın.
- Bayraklar:
  - `--json`: makine tarafından okunabilir çıktı.

Örnek:

```bash
clawhub package trusted-publisher get @openclaw/example-plugin
```

### `package trusted-publisher set <name>`

- Mevcut bir pakete GitHub Actions güvenilir yayıncı yapılandırması ekler
  veya mevcut yapılandırmayı değiştirir.
- Paket önce normal manuel veya token ile kimliği doğrulanmış
  `clawhub package publish` aracılığıyla oluşturulmalıdır.
- Yapılandırma ayarlandıktan sonra gelecekte desteklenen GitHub Actions yayımlamaları,
  uzun ömürlü bir ClawHub token'ı olmadan OIDC/güvenilir yayımlamayı kullanabilir.
- `--repository <repo>`, `owner/repo` olmalıdır.
- `--workflow-filename <file>`, `.github/workflows/` içindeki iş akışı
  dosya adıyla eşleşmelidir.
- `--environment <name>` isteğe bağlıdır. Yapılandırıldığında OIDC talebindeki
  GitHub Actions ortamı tam olarak eşleşmelidir.
- ClawHub, bu komut çalıştırıldığında yapılandırılan GitHub deposunu doğrular.
  Herkese açık depolar, herkese açık GitHub meta verileri aracılığıyla doğrulanabilir. Özel
  depolar için ClawHub'ın söz konusu depoya GitHub erişimi olması gerekir;
  örneğin gelecekteki bir ClawHub GitHub App kurulumu veya başka bir yetkilendirilmiş
  GitHub entegrasyonu aracılığıyla.
- Bayraklar:
  - `--repository <repo>`: GitHub deposu, örneğin `openclaw/example-plugin`.
  - `--workflow-filename <file>`: iş akışı dosya adı, örneğin `package-publish.yml`.
  - `--environment <name>`: isteğe bağlı, tam eşleşmeli GitHub Actions ortamı.
  - `--json`: makine tarafından okunabilir çıktı.

Örnek:

```bash
clawhub package trusted-publisher set @openclaw/example-plugin \
  --repository openclaw/example-plugin \
  --workflow-filename package-publish.yml \
  --environment release
```

### `package trusted-publisher delete <name>`

- Bir paketten güvenilir yayıncı yapılandırmasını kaldırır.
- İş akışı, depo veya ortam sabitlemesinin devre dışı bırakılması ya da yeniden
  oluşturulması gerekiyorsa bunu geri alma işlemi olarak kullanın.
- Yapılandırma yeniden ayarlanana kadar gelecekteki gerçek yayımlamalar normal kimliği
  doğrulanmış yayımlamayı kullanmalıdır.
- Bayraklar:
  - `--json`: makine tarafından okunabilir çıktı.

Örnek:

```bash
clawhub package trusted-publisher delete @openclaw/example-plugin
```

### Kurulum telemetrisi

- Oturum açıldığında, `CLAWHUB_DISABLE_TELEMETRY=1` ayarlanmadığı sürece
  `clawhub install <slug>` sonrasında gönderilir.
- Bildirim en iyi çaba esasına dayanır. Telemetri kullanılamıyorsa kurulum
  komutları başarısız olmaz.
- Ayrıntılar: `docs/telemetry.md`.
