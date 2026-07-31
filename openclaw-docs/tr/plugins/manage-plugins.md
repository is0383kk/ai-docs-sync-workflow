---
doc-schema-version: 1
read_when:
    - Control UI'da pluginlere göz atmak, onları yüklemek, etkinleştirmek veya devre dışı bırakmak istiyorsunuz
    - Hızlı Plugin listeleme, yükleme, güncelleme, inceleme veya kaldırma örnekleri istiyorsunuz
    - Bir plugin yükleme kaynağı seçmek istiyorsunuz
    - Plugin paketlerini yayımlamak için doğru referansı istiyorsunuz
sidebarTitle: Manage plugins
summary: OpenClaw pluginlerini Control UI veya CLI üzerinden yönetin
title: Pluginleri yönet
x-i18n:
    generated_at: "2026-07-27T00:09:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9101d5c3630b618a043f1e71fdf5fa083698cc23694ccdc773d295a37c4c1ef3
    source_path: plugins/manage-plugins.md
    workflow: 16
---

Control UI, yaygın keşfetme, yükleme, etkinleştirme ve devre dışı bırakma
iş akışını kapsar. CLI; güncelleme, kaldırma, gelişmiş yapılandırma ve açık
yükleme kaynağı denetimleri ekler. Komut sözleşmesinin, bayrakların, kaynak seçimi
kurallarının ve uç durumların tamamı için [`openclaw plugins`](/tr/cli/plugins) sayfasına bakın.

Tipik CLI iş akışı: bir paket bulun, ClawHub, npm, git veya yerel bir
yoldan yükleyin, yönetilen Gateway'in otomatik olarak yeniden başlamasını bekleyin (ya da
elle yeniden başlatın), ardından plugin'in çalışma zamanı kayıtlarını doğrulayın.

## Control UI'ı kullanma

Control UI'da **Plugin'ler** bölümünü açın veya yapılandırılmış Control UI
temel yoluna göre `/settings/plugins` yolunu kullanın. Örneğin, `/openclaw` temel yolu
`/openclaw/settings/plugins` yolunu kullanır. Sayfada iki sekme bulunur:

- **Yüklü** sekmesi, kategoriye göre gruplandırılmış tam yerel envanteri gösterir (kanallar,
  model sağlayıcıları, bellek, araçlar). Her satır bir ayrıntı görünümü açar; taşma
  (`…`) menüsü plugin'i etkinleştirir veya devre dışı bırakır ve haricen yüklenmiş
  plugin'ler için **Kaldır** seçeneğini sunar. Sekme ayrıca aynı menü tabanlı
  etkinleştirme, devre dışı bırakma ve kaldırma eylemleriyle yapılandırılmış
  [MCP sunucularını](/tr/cli/mcp) listeler ve Gateway yapılandırmasındaki `mcp.servers`
  değerini düzenler.
- **Keşfet**, mağazadır: OpenClaw ile birlikte gelen öne çıkan plugin'ler, resmî
  haricî plugin'ler ve seçilmiş bir bağlayıcı rafı. Bağlayıcı kartları, barındırılan bir
  MCP sunucusunu tek tıklamayla ekler (GitHub, Notion, Linear, Sentry,
  Home Assistant) veya önceden doldurulmuş bir ClawHub aramasına yönlendirir. Arama
  kutusuna yazıldığında [ClawHub](https://clawhub.ai/plugins) satır içinde sorgulanır ve indirme sayılarıyla
  kaynak doğrulama rozetlerini içeren bir **ClawHub'dan** bölümü eklenir.

Dahil edilen plugin'ler paket yüklemesi gerektirmez. Menü eylemleri **Etkinleştir**
veya **Devre Dışı Bırak** şeklindedir. Örneğin Workboard, OpenClaw ile birlikte gelir ve varsayılan
olarak devre dışıdır; açmak için **Etkinleştir** seçeneğini seçin. Paketle gelen plugin'ler
kaldırılamaz, yalnızca devre dışı bırakılabilir.

Katalog ve arama erişimi `operator.read` gerektirir. Yükleme, etkinleştirme, devre dışı bırakma,
kaldırma ve MCP sunucusu değişiklikleri `operator.admin` gerektirir. ClawHub yüklemesi
Gateway tarafından gerçekleştirilir ve güven, bütünlük ve plugin yükleme
politikası denetimlerini korur. Yüklü bir plugin'i yönetici olarak etkinleştirmek,
seçilen plugin'i mevcut kısıtlayıcı `plugins.allow` listesine ekleyerek
bu açık güveni de kaydeder. Açık bir `plugins.deny` girdisi belirleyici olmaya devam eder ve
plugin etkinleştirilmeden önce kaldırılmalıdır.

Plugin kodunu yüklemek veya kaldırmak Gateway'in yeniden başlatılmasını gerektirir. Etkinleştirme
değişiklikleri, yüklü plugin ve geçerli Gateway çalışma zamanı bunu desteklediğinde
yeniden başlatma olmadan uygulanabilir; aksi takdirde UI, yeniden başlatmanın gerekli olduğunu bildirir.
OAuth destekli MCP bağlayıcıları eklendikten sonra yine de CLI'dan bir defaya mahsus
`openclaw mcp login <name>` gerektirir.

Control UI; rastgele npm, git veya yerel yol kaynaklarından yükleme yapmaz,
plugin'leri güncellemez ya da kapsamlı plugin yapılandırmasını sunmaz. Bu işlemler için
aşağıdaki CLI iş akışlarını kullanın.

## Plugin'leri listeleme ve arama

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
openclaw plugins search "calendar"
```

Betikler için `--json`:

```bash
openclaw plugins list --json \
  | jq '.plugins[] | {id, enabled, format, source, dependencyStatus}'
```

`plugins list`, soğuk envanter denetimidir: OpenClaw'ın yapılandırmadan,
manifestlerden ve kalıcı plugin kayıt defterinden keşfedebildiklerini gösterir. Zaten çalışan bir
Gateway'in plugin çalışma zamanını içe aktardığını kanıtlamaz. JSON çıktısı,
kayıt defteri tanılamalarını ve her plugin'in `dependencyStatus` durumunu (bildirilen
`dependencies`/`optionalDependencies` öğelerinin diskte çözümlenip çözümlenmediğini) içerir.

`plugins search`, yüklenebilir plugin paketleri için ClawHub'ı sorgular ve
her sonuç için bir yükleme ipucu (`openclaw plugins install clawhub:<package>`) yazdırır.

## Plugin'leri etkinleştirme ve devre dışı bırakma

```bash
openclaw plugins enable <plugin-id>
openclaw plugins disable <plugin-id>
```

Yüklü dosyalara dokunmadan plugin'in yapılandırma girdisini değiştirir. Paketle
gelen bazı plugin'ler (paketle gelen model/konuşma sağlayıcıları ve paketle gelen tarayıcı plugin'i)
varsayılan olarak etkindir; diğerleri yüklemeden sonra `enable` gerektirir.

## Plugin'leri yükleme

```bash
# Plugin paketleri için ClawHub'da arama yapın.
openclaw plugins search "calendar"

# ClawHub'dan yükleyin.
openclaw plugins install clawhub:<package>
openclaw plugins install clawhub:<package>@1.2.3
openclaw plugins install clawhub:<package>@beta

# npm'den yükleyin.
openclaw plugins install npm:<package>
openclaw plugins install npm:@scope/openclaw-plugin@1.2.3
openclaw plugins install npm:@openclaw/codex

# Yerel bir npm-pack yapısından yükleyin.
openclaw plugins install npm-pack:<path.tgz>

# git'ten veya yerel bir geliştirme çalışma kopyasından yükleyin.
openclaw plugins install git:github.com/acme/openclaw-plugin@v1.0.0
openclaw plugins install ./my-plugin
openclaw plugins install --link ./my-plugin
```

Yalın paket belirtimleri, ad paketle gelen veya resmî bir plugin kimliğiyle
eşleşmediği sürece geçiş sırasında npm'den yüklenir; eşleşirse OpenClaw bunun
yerine ilgili yerel/resmî kopyayı kullanır. Belirleyici kaynak seçimi için `clawhub:`, `npm:`, `git:`
veya `npm-pack:` kullanın. OpenClaw'ın paketle gelen ve resmî katalog paketlerine,
ClawHub paketleriyle birlikte güvenilir. Yeni ve rastgele npm, git, yerel yol/arşiv,
`npm-pack:` veya pazar yeri kaynakları; kaynağı inceleyip
güvendikten sonra etkileşimsiz yüklemelerde `--force` gerektirir.

`--force`, ClawHub dışındaki bir kaynağı istem göstermeden onaylar ve
gerektiğinde mevcut yükleme hedefinin üzerine yazar. İzlenen bir npm,
ClawHub veya hook-pack yüklemesinin rutin yükseltmeleri için bunun yerine `openclaw plugins update` kullanın.
`--link` ile `--force` yalnızca kaynağı onaylar; bağlantılı dizin
kopyalanmaz veya üzerine yazılmaz.

Yeni yüklenen bir plugin henüz mevcut olmayan bir yapılandırma gerektiriyorsa
OpenClaw yüklemeyi kaydeder ancak plugin'i devre dışı bırakır. `plugins.entries.<id>.config`
öğesini yapılandırın, ardından `openclaw plugins enable <id>` komutunu çalıştırın. Mevcut bir
yapılandırma girdisi bulunuyor ancak geçersizse yükleme, girdiyi yeniden yazmadan başarısız olur.

## Yeniden başlatma ve inceleme

Yapılandırma yeniden yüklemesi etkin olan, çalışan ve yönetilen bir Gateway; plugin kodu
yüklendikten, güncellendikten veya kaldırıldıktan sonra otomatik olarak yeniden başlar. Gateway
yönetilmiyorsa veya yeniden yükleme devre dışıysa canlı çalışma zamanı yüzeylerini
denetlemeden önce kendiniz yeniden başlatın:

```bash
openclaw gateway restart
openclaw plugins inspect <plugin-id> --runtime --json
```

`inspect --runtime`, plugin modülünü yükler ve çalışma zamanı yüzeylerini
(araçlar, kancalar, hizmetler, Gateway yöntemleri, HTTP rotaları, plugin'e ait
CLI komutları) kaydettiğini kanıtlar. Düz `inspect` ve `list` yalnızca soğuk
manifest/yapılandırma/kayıt defteri denetimleridir.

## Plugin'leri güncelleme

```bash
openclaw plugins update <plugin-id>
openclaw plugins update <npm-package-or-spec>
openclaw plugins update --all
openclaw plugins update <plugin-id> --dry-run
```

Bir plugin kimliği iletildiğinde izlenen yükleme belirtimi yeniden kullanılır: saklanan dist-tag'ler
(`@beta`) ve tam sabitlenmiş sürümler sonraki `update <plugin-id>`
çalıştırmalarına aktarılır.

`openclaw plugins update --all`, toplu bakım yoludur. Olağan izlenen yükleme
belirtimlerine yine uyar ancak güvenilir resmî OpenClaw plugin kayıtları,
eski ve tam bir resmî pakete sabitlenmek yerine geçerli resmî katalog hedefiyle
eşitlenir; `update.channel` değeri `beta` olduğunda bu eşitleme beta
sürüm hattını tercih eder. Tam veya etiketlenmiş resmî bir belirtimi değiştirmeden
korumak için hedefli bir `update <plugin-id>` kullanın.

npm yüklemelerinde izlenen kaydı değiştirmek için açık bir paket belirtimi
iletin:

```bash
openclaw plugins update @scope/openclaw-plugin@beta
openclaw plugins update @scope/openclaw-plugin
```

İkinci komut, daha önce tam bir sürüme veya etikete sabitlenmiş plugin'i
kayıt defterinin varsayılan sürüm hattına geri taşır.

Tam geri dönüş ve sabitleme kuralları için
[`openclaw plugins`](/tr/cli/plugins#update) sayfasına bakın.

## Plugin'leri kaldırma

```bash
openclaw plugins uninstall <plugin-id> --dry-run
openclaw plugins uninstall <plugin-id>
openclaw plugins uninstall <plugin-id> --keep-files
```

Kaldırma işlemi; plugin'in yapılandırma girdisini, kalıcı plugin dizini kaydını,
izin/verme listesi girdilerini ve uygun olduğunda bağlantılı `plugins.load.paths`
girdilerini kaldırır. `--keep-files` iletmediğiniz sürece yönetilen yükleme
dizini kaldırılır. Kaldırma işlemi plugin kaynağını değiştirdiğinde, çalışan ve yönetilen bir
Gateway otomatik olarak yeniden başlatılır.

Nix modunda (`OPENCLAW_NIX_MODE=1`) plugin yükleme, güncelleme, kaldırma,
etkinleştirme ve devre dışı bırakma işlemlerinin tümü devre dışıdır; bunun yerine bu seçimleri
yüklemenin Nix kaynağında yönetin.

## Kaynak seçme

| Kaynak      | Kullanım durumu                                                                    | Örnek                                                        |
| ----------- | --------------------------------------------------------------------------- | -------------------------------------------------------------- |
| ClawHub     | OpenClaw'a özgü keşif, tarama özetleri, sürümler ve ipuçları istediğinizde     | `openclaw plugins install clawhub:<package>`                   |
| git         | Bir depodan dal, etiket veya işleme almak istediğinizde                         | `openclaw plugins install git:github.com/<owner>/<repo>@<ref>` |
| yerel yol  | Aynı makinede bir plugin geliştiriyor veya test ediyorsanız                  | `openclaw plugins install --link ./my-plugin`                  |
| pazar yeri | Claude uyumlu bir pazar yeri plugin'i yüklüyorsanız                   | `openclaw plugins install <plugin> --marketplace <source>`     |
| npm paketi    | Yerel bir paket yapısını npm yükleme semantiğiyle doğruluyorsanız      | `openclaw plugins install npm-pack:<path.tgz>`                 |
| npmjs.com   | Zaten JavaScript paketleri yayımlıyorsanız veya npm dist-tag'lerine/özel kayıt defterine ihtiyacınız varsa | `openclaw plugins install npm:@acme/openclaw-plugin`           |

Yönetilen yerel yol yüklemeleri plugin dizinleri veya arşivleri olmalıdır.
Bağımsız plugin dosyalarını `plugins install` ile yüklemek yerine
`plugins.load.paths` konumuna koyun.

## Plugin'leri yayımlama

ClawHub, OpenClaw plugin'leri için birincil herkese açık keşif yüzeyidir. Kullanıcıların
yüklemeden önce plugin meta verilerini, sürüm geçmişini, kayıt defteri tarama
sonuçlarını ve yükleme ipuçlarını bulmasını istediğinizde burada yayımlayın.

```bash
npm i -g clawhub
clawhub login
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
clawhub package publish your-org/your-plugin@v1.0.0
```

Yerel npm plugin'leri, yayımlanmadan önce bir plugin manifesti (`openclaw.plugin.json`) ve
`package.json` meta verileriyle birlikte sunulmalıdır:

```json package.json
{
  "name": "@acme/openclaw-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

```bash
npm publish --access public
openclaw plugins install npm:@acme/openclaw-plugin
openclaw plugins install npm:@acme/openclaw-plugin@beta
openclaw plugins install npm:@acme/openclaw-plugin@1.0.0
```

Bu sayfayı yayımlama başvurusu olarak kullanmak yerine yayımlama sözleşmesinin
tamamı için şu sayfaları kullanın:

- [ClawHub'da yayımlama](/tr/clawhub/publishing); sahipleri, kapsamları,
  sürümleri, incelemeyi, paket doğrulamasını ve paket aktarımını açıklar.
- [Plugin oluşturma](/tr/plugins/building-plugins), tam plugin
  paket yapısını (`openclaw.plugin.json` dahil) ve ilk yayımlama
  iş akışını gösterir.
- [Plugin manifesti](/tr/plugins/manifest), yerel plugin manifesti
  alanlarını tanımlar.

Aynı paket hem ClawHub'da hem de npm'de bulunuyorsa tek bir kaynağı zorunlu kılmak için
açık `clawhub:` veya `npm:` önekini kullanın.

## İlgili

- [Pluginler](/tr/tools/plugin) - yükleme, yapılandırma, yeniden başlatma ve sorun giderme
- [`openclaw plugins`](/tr/cli/plugins) - eksiksiz CLI referansı
- [Topluluk pluginleri](/tr/plugins/community) - herkese açık keşif ve ClawHub'da yayımlama
- [ClawHub](/tr/clawhub/cli) - kayıt defteri CLI işlemleri
- [Plugin oluşturma](/tr/plugins/building-plugins) - plugin paketi oluşturma
- [Plugin manifesti](/tr/plugins/manifest) - manifest ve paket meta verileri
