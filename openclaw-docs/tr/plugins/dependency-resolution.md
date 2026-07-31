---
read_when:
    - Plugin paketi kurulumlarında hata ayıklıyorsunuz
    - Plugin başlatma, doctor veya paket yöneticisi kurulum davranışını değiştiriyorsunuz
    - Paketlenmiş OpenClaw kurulumlarının veya paketle birlikte sunulan plugin manifestlerinin bakımını yapıyorsunuz
sidebarTitle: Dependencies
summary: OpenClaw plugin paketlerini nasıl kurar ve plugin bağımlılıklarını nasıl çözümler
title: Plugin bağımlılık çözümleme
x-i18n:
    generated_at: "2026-07-26T22:52:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ae24a82568e275399cb7b68729d2805956792852612f84d6918850305f0eb243
    source_path: plugins/dependency-resolution.md
    workflow: 16
---

OpenClaw, plugin bağımlılıklarını yalnızca kurulum/güncelleme sırasında işler. Çalışma zamanı
yüklemesi hiçbir zaman bir paket yöneticisi çalıştırmaz, bağımlılık ağacını onarmaz veya
OpenClaw paket dizinini değiştirmez.

## Sorumlulukların ayrımı

Plugin paketleri kendi bağımlılık grafiğinden sorumludur:

- Çalışma zamanı bağımlılıkları, plugin paketinin `dependencies` veya
  `optionalDependencies` dosyasında bulunur.
- SDK/çekirdek içe aktarımları, eş bağımlılıklar veya OpenClaw tarafından sağlanan içe aktarımlardır.
- Yerel geliştirme pluginleri, önceden kurulmuş kendi bağımlılıklarını getirir.
- npm ve git pluginleri, OpenClaw'un sahip olduğu paket köklerine kurulur.

OpenClaw yalnızca plugin yaşam döngüsünden sorumludur:

- Plugin kaynağını keşfeder.
- Açıkça istendiğinde paketi kurar veya günceller.
- Kurulum meta verilerini kaydeder.
- Plugin giriş noktasını yükler.
- Bağımlılıklar eksik olduğunda uygulanabilir bir hata ile başarısız olur.

## Kurulum kökleri

OpenClaw, kaynak başına kararlı kökler kullanır:

- npm paketleri, `~/.openclaw/npm/projects/<encoded-package>` altındaki plugin başına projelere
  kurulur.
- git paketleri `~/.openclaw/git` altına klonlanır.
- Yerel/yol/arşiv kurulumları, bağımlılık onarımı yapılmadan kopyalanır
  veya referans olarak kullanılır.

npm kurulumları, plugin başına proje kökünde şu komutla çalışır:

```bash
cd ~/.openclaw/npm/projects/<encoded-package>
npm install --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts --no-audit --no-fund
```

`openclaw plugins install npm-pack:<path.tgz>`, yerel bir npm-pack tar arşivi için aynı plugin başına npm
proje kökünü kullanır: OpenClaw tar arşivinin npm meta verilerini okur,
yönetilen projeye kopyalanmış bir `file:` bağımlılığı olarak ekler, yukarıdaki
normal npm kurulumunu çalıştırır ve ardından plugine güvenmeden önce kurulan
kilit dosyası meta verilerini doğrular. Bu yol, yerel paketleme yapıtının
taklit ettiği kayıt defteri yapıtı gibi davranması gereken paket kabulü ve
sürüm adayı kanıtı için vardır.

Resmî veya harici plugin paketlerini yayımlamadan önce test ederken `npm-pack:`
kullanın. Ham arşiv veya yol kurulumu, yerel hata ayıklama için kullanışlıdır ancak
kurulu bir npm veya ClawHub paketiyle aynı bağımlılık yolunu kanıtlamaz.
`npm-pack:`, yönetilen paket kurulum biçimini kanıtlar; tek başına
pluginin katalog bağlantılı resmî içerik olduğunu kanıtlamaz.

Davranış, paketle birlikte gelen plugin veya güvenilir resmî plugin durumuna
bağlı olduğunda, yerel paket kanıtını katalog destekli resmî bir kurulumla ya da
resmî güveni kaydeden yayımlanmış bir paket yoluyla eşleştirin. Ayrıcalıklı yardımcı
erişimi ve güvenilir-resmî kapsam işleme davranışı, yerel tar arşivi kurulumundan
çıkarılmak yerine bu güvenilir kurulum yolunda doğrulanmalıdır.

Bir plugin çalışma zamanında eksik içe aktarma nedeniyle başarısız olursa,
yönetilen projeyi elle onarmak yerine paket bildirimini düzeltin. Çalışma zamanı
içe aktarımları, plugin paketinin `dependencies` veya `optionalDependencies` dosyasına
aittir; `devDependencies`, yönetilen çalışma zamanı projeleri için kurulmaz.
`~/.openclaw/npm/projects/<encoded-package>` içindeki yerel bir `npm install`,
geçici bir tanılama işleminin önünü açabilir ancak sonraki kurulum veya
güncelleme projeyi paket meta verilerinden yeniden oluşturduğu için paket kabulü
kanıtı değildir.

npm, geçişli bağımlılıkları plugin paketinin yanındaki plugin başına projenin
`node_modules` dizinine yükseltebilir. OpenClaw, kuruluma güvenmeden önce
yönetilen proje kökünü tarar ve kaldırma sırasında bu projeyi siler; dolayısıyla
yükseltilmiş çalışma zamanı bağımlılıkları ilgili pluginin temizleme sınırı içinde kalır.

Yayımlanmış npm plugin paketleri `npm-shrinkwrap.json` ile gönderilebilir; npm,
kurulum sırasında yayımlanabilir bu kilit dosyasını kullanır ve OpenClaw'un
yönetilen npm proje kökü bunu normal kurulum yolu üzerinden destekler.
OpenClaw'a ait yayımlanabilir plugin paketleri, ilgili paketin yayımlanmış
bağımlılık grafiğinden oluşturulmuş, pakete yerel bir shrinkwrap içermelidir:

```bash
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check
```

Oluşturucu, plugin `devDependencies` değerlerini çıkarır, çalışma alanı geçersiz
kılma politikasını uygular ve `openclaw.release.publishToNpm: true` içeren her plugin için
`extensions/<id>/npm-shrinkwrap.json` yazar. Üçüncü taraf plugin paketleri de bir shrinkwrap ile
gönderilebilir; OpenClaw topluluk paketleri için bunu zorunlu tutmaz ancak
mevcut olduğunda npm buna uyar.

Yerel bir paketi sürüm adayı kanıtı olarak değerlendirmeden önce kurulacak
tar arşivini inceleyin:

```bash
npm pack --pack-destination /tmp
tar -xOf /tmp/<plugin-package>.tgz package/package.json
tar -tf /tmp/<plugin-package>.tgz | grep '^package/dist/'
```

Bağımlılık değişikliklerinde ayrıca üretim kurulumunun, geliştirme bağımlılıkları
olmadan çalışma zamanı paketlerini çözümleyebildiğini doğrulayın:

```bash
tmpdir=$(mktemp -d)
(
  cd "$tmpdir"
  npm init -y >/dev/null
  npm install --package-lock-only --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts /tmp/<plugin-package>.tgz
)
rm -rf "$tmpdir"
```

OpenClaw'a ait npm plugin paketleri açık bir `bundledDependencies` ile de
yayımlanabilir. npm yayımlama yolu, çalışma zamanı bağımlılık adları listesini
yerleştirir, yalnızca geliştirmeye yönelik çalışma alanı meta verilerini
yayımlanan bildirimden çıkarır, pakete yerel çalışma zamanı bağımlılıkları için
betiksiz bir npm kurulumu çalıştırır ve ardından bu bağımlılık dosyalarını içeren
plugin tar arşivini paketler veya yayımlar. Yoğun yerel bileşen kullanan paketler
(Codex, ACPX, Copilot, llama.cpp, memory-lancedb, Tlon)
`openclaw.release.bundleRuntimeDependencies: false` ile bu işlemin dışında kalır;
yine shrinkwrap ile gönderilirler ancak npm, her platform ikili dosyasını plugin
tar arşivine gömmek yerine çalışma zamanı bağımlılıklarını kurulum sırasında çözümler.
Kök `openclaw` paketi, tam bağımlılık ağacını paketlemez.

`openclaw/plugin-sdk/*` içe aktaran pluginler, `openclaw` öğesini eş
bağımlılık olarak bildirir. OpenClaw, npm'in ana makine paketinin ayrı bir kayıt
defteri kopyasını yönetilen projeye kurmasına izin vermez; çünkü eski bir ana
makine paketi, npm'in ilgili plugin içindeki eş bağımlılık çözümlemesini etkileyebilir.
Yönetilen npm kurulumları, npm eş bağımlılık çözümlemesini/materyalleştirmesini
atlar ve OpenClaw, ana makine eş bağımlılığını bildiren kurulu paketler için
kurulum veya güncellemeden sonra plugine yerel `node_modules/openclaw` bağlantılarını
yeniden oluşturur.

git kurulumları depoyu klonlar veya yeniler, ardından şunu çalıştırır:

```bash
npm install --omit=dev --ignore-scripts --no-audit --no-fund
```

Kurulan plugin daha sonra bu paket dizininden yüklenir; böylece pakete yerel
ve üst `node_modules` çözümlemesi, normal bir Node paketindekiyle aynı şekilde
çalışır.

## Yerel pluginler

Yerel pluginler, geliştirici tarafından denetlenen dizinlerdir. OpenClaw bunlar için
asla `npm install`, `pnpm install` veya bağımlılık onarımı çalıştırmaz;
yerel bir pluginin bağımlılıkları varsa plugini yüklemeden önce bunları ilgili
plugine kurun.

Üçüncü taraf TypeScript yerel pluginleri, acil durum yolu olarak Jiti üzerinden
yüklenir. Paketlenmiş JavaScript pluginleri ve paketle birlikte gelen dahili
pluginler ise yerel import/require üzerinden yüklenir.

## Başlatma ve yeniden yükleme

Gateway başlatma ve yapılandırmayı yeniden yükleme işlemleri, plugin bağımlılıklarını
asla kurmaz. Plugin kurulum kayıtlarını okur, giriş noktasını hesaplar ve onu yükler.

Çalışma zamanında eksik bir bağımlılık, operatörü açık bir düzeltmeye yönlendiren
bir hatayla plugin yüklemesinin başarısız olmasına neden olur:

```bash
openclaw plugins update <id>
openclaw plugins install <source>
openclaw doctor --fix
```

`doctor --fix`, OpenClaw tarafından oluşturulmuş eski bağımlılık durumunu
temizler ve yapılandırma hâlâ bunlara başvuruyorsa yerel kurulum kayıtlarında
eksik olan indirilebilir pluginleri kurtarabilir. Doctor, önceden kurulmuş yerel
bir pluginin bağımlılıklarını onarmaz.

## Paketle birlikte gelen pluginler

Hafif ve çekirdek açısından kritik paketle birlikte gelen pluginler, OpenClaw'un
parçası olarak sunulur. Bunlar ya ağır bir çalışma zamanı bağımlılık ağacı
taşımamalı ya da ClawHub/npm üzerinde indirilebilir bir pakete taşınmalıdır.

Çekirdek pakette sunulan, harici olarak kurulan veya yalnızca kaynakta kalan
pluginlerin güncel oluşturulmuş listesi için
[Plugin envanteri](/tr/plugins/plugin-inventory) sayfasına bakın.

Paketle birlikte gelen plugin bildirimleri, bağımlılık hazırlama talep etmemelidir.
Büyük veya isteğe bağlı plugin işlevleri normal bir plugin olarak paketlenmeli ve
üçüncü taraf pluginlerle aynı npm/git/ClawHub yolu üzerinden kurulmalıdır.

Kaynak çalışma kopyalarında OpenClaw, depoyu bir pnpm monoreposu olarak ele alır.
`pnpm install` sonrasında paketle birlikte gelen pluginler
`extensions/<id>` üzerinden yüklenir; böylece pakete yerel çalışma alanı
bağımlılıkları kullanılabilir olur ve düzenlemeler doğrudan uygulanır. Kaynak
çalışma kopyası geliştirmesi yalnızca pnpm kullanır; depo kökündeki düz
`npm install`, paketle birlikte gelen plugin bağımlılıklarını hazırlamaz.

| Kurulum biçimi                    | Paketle birlikte gelen plugin konumu               | Bağımlılık sahibi                                                     |
| -------------------------------- | ------------------------------------- | -------------------------------------------------------------------- |
| `npm install -g openclaw`        | Paket içindeki derlenmiş çalışma zamanı ağacı | OpenClaw paketi ve açık plugin kurulum/güncelleme/doctor akışları     |
| Git çalışma kopyası ve `pnpm install` | `extensions/<id>` çalışma alanı paketleri  | Her plugin paketinin kendi bağımlılıkları dâhil pnpm çalışma alanı |
| `openclaw plugins install ...`   | Yönetilen npm projesi/git/ClawHub kökü  | Plugin kurulum/güncelleme akışı                                       |

## Eski durum temizliği

Eski OpenClaw sürümleri, başlatma sırasında veya doctor onarımı esnasında
paketle birlikte gelen plugin bağımlılık kökleri oluşturuyordu. Güncel doctor
temizliği, eski `plugin-runtime-deps` kökleri, kaldırılmış
`plugin-runtime-deps` hedeflerini gösteren genel Node öneki paket sembolik bağlantıları,
`.openclaw-runtime-deps*` bildirimleri, oluşturulmuş plugin `node_modules` öğeleri,
kurulum hazırlama dizinleri ve pakete yerel pnpm depoları dâhil olmak üzere bu eski
dizinleri ve sembolik bağlantıları `--fix` ile kaldırır. Paketlenmiş
postinstall işlemi de eski hedef köklerini budamadan önce bu genel sembolik
bağlantıları kaldırır; böylece yükseltmeler askıda ESM paket içe aktarımları bırakmaz.

Eski npm kurulumları ayrıca paylaşılan bir `~/.openclaw/npm/node_modules` kökü kullanıyordu.
Güncel kurulum, güncelleme, kaldırma ve doctor akışları, bu eski düz kökü yalnızca
kurtarma ve temizleme amacıyla tanımaya devam eder. Yeni npm kurulumları bunun
yerine plugin başına proje kökleri oluşturur.
