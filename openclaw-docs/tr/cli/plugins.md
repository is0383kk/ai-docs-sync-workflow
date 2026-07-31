---
read_when:
    - Gateway Pluginlerini veya uyumlu paketleri yüklemek ya da yönetmek istiyorsunuz
    - Basit bir araç Plugin'i için iskelet oluşturmak veya bunu doğrulamak istiyorsunuz
    - Plugin yükleme hatalarında hata ayıklamak istiyorsunuz
sidebarTitle: Plugins
summary: '`openclaw plugins` için CLI başvurusu (başlatma, derleme, doğrulama, listeleme, yükleme, pazar yeri, kaldırma, etkinleştirme/devre dışı bırakma, doctor)'
title: Pluginler
x-i18n:
    generated_at: "2026-07-26T23:54:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a1acba76fb1bc0ddae75e51fe573d3c2ac8f694607836e0c072ec7ca8fc0e262
    source_path: cli/plugins.md
    workflow: 16
---

Gateway pluginlerini, hook paketlerini ve uyumlu paketleri yönetin.

<CardGroup cols={2}>
  <Card title="Plugin sistemi" href="/tr/tools/plugin">
    Pluginleri yükleme, etkinleştirme ve sorunlarını giderme hakkında son kullanıcı kılavuzu.
  </Card>
  <Card title="Pluginleri yönetin" href="/tr/plugins/manage-plugins">
    Yükleme, listeleme, güncelleme, kaldırma ve yayımlama için kısa örnekler.
  </Card>
  <Card title="Plugin paketleri" href="/tr/plugins/bundles">
    Paket uyumluluk modeli.
  </Card>
  <Card title="Plugin manifesti" href="/tr/plugins/manifest">
    Manifest alanları ve yapılandırma şeması.
  </Card>
  <Card title="Güvenlik" href="/tr/gateway/security">
    Plugin yüklemeleri için güvenlik güçlendirmesi.
  </Card>
</CardGroup>

## Komutlar

```bash
openclaw plugins list [--enabled] [--verbose] [--json]
openclaw plugins search <query> [--limit <n>] [--json]
openclaw plugins install <path-or-spec> [--link] [--force] [--pin] [--marketplace <source>]
openclaw plugins inspect <id> [--runtime] [--json]
openclaw plugins inspect --all [--runtime] [--json]
openclaw plugins info <id>                    # inspect için diğer ad
openclaw plugins enable <id>
openclaw plugins disable <id>
openclaw plugins uninstall <id> [--dry-run] [--keep-files] [--force]
openclaw plugins update <id-or-npm-spec> | --all [--dry-run]
openclaw plugins registry [--refresh] [--json]
openclaw plugins doctor
openclaw plugins init <id> [--name <name>] [--type tool|provider] [--directory <path>]
openclaw plugins build [--entry <path>] [--check]
openclaw plugins validate [--entry <path>]
openclaw plugins marketplace entries [--offline] [--feed-profile <name>] [--json]
openclaw plugins marketplace list <source> [--json]
openclaw plugins marketplace refresh [--feed-profile <name>] [--expected-sha256 <sha256>] [--json]
```

Yavaş yükleme, inceleme, kaldırma veya kayıt yenileme işlemlerini araştırmak için
komutu `OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1` ile çalıştırın. İz, aşama sürelerini stderr'e
yazar ve JSON çıktısının ayrıştırılabilir kalmasını sağlar. Bkz. [Hata Ayıklama](/tr/help/debugging#plugin-lifecycle-trace).

<Note>
Nix modunda (`OPENCLAW_NIX_MODE=1`), `openclaw.json` değiştirilemez. `install`, `update`, `uninstall`, `enable` ve `disable` komutlarının tümü çalışmayı reddeder. Bunun yerine bu yüklemeye ait Nix kaynağını (nix-openclaw için `programs.openclaw.config` veya `instances.<name>.config`) düzenleyip yeniden derleyin. Önce ajan yaklaşımını kullanan [Hızlı Başlangıç](https://github.com/openclaw/nix-openclaw#quick-start) bölümüne bakın.
</Note>

<Note>
Birlikte gelen pluginler OpenClaw ile dağıtılır. Bazıları varsayılan olarak etkindir (örneğin birlikte gelen model sağlayıcıları, konuşma sağlayıcıları ve tarayıcı plugini); diğerleri `plugins enable` gerektirir.

Yerel OpenClaw pluginleri, satır içi JSON Schema (`configSchema`, boş olsa bile) içeren `openclaw.plugin.json` dosyasını sunar. Uyumlu paketler bunun yerine kendi paket manifestlerini kullanır.

`plugins list`, `Format: openclaw` veya `Format: bundle` değerini gösterir. Ayrıntılı liste/bilgi çıktısı ayrıca paket alt türünü (`codex`, `claude` veya `cursor`) ve algılanan paket yeteneklerini gösterir.
</Note>

## Yazarlık

```bash
openclaw plugins init stock-quotes --name "Stock Quotes"
cd stock-quotes
npm run plugin:build
npm run plugin:validate
```

`plugins init` varsayılan olarak asgari bir TypeScript araç plugini oluşturur. İlk
bağımsız değişken plugin kimliğidir; `--name` görünen adı ayarlar. OpenClaw,
varsayılan çıktı dizini ve paket adlandırması için kimliği kullanır. Araç iskeletleri
`defineToolPlugin` kullanır ve önce derleyip ardından `openclaw plugins build`/`validate`
çağıran `package.json` betiklerini, yani `plugin:build` ve
`plugin:validate` betiklerini oluşturur.

`plugins build` derlenmiş giriş noktasını içe aktarır, statik araç meta verilerini okur,
`openclaw.plugin.json` dosyasını yazar ve `package.json` içindeki `openclaw.extensions`
değerini uyumlu tutar. `plugins validate`, oluşturulan manifestin, paket meta verilerinin ve
geçerli giriş noktası dışa aktarımının hâlâ birbiriyle uyumlu olduğunu denetler. Eksiksiz
yazarlık iş akışı için [Araç Pluginleri](/tr/plugins/tool-plugins) bölümüne bakın.

İskelet TypeScript kaynak kodu yazar ancak meta verileri derlenmiş
`./dist/index.js` giriş noktasından oluşturur; bu nedenle iş akışı yayımlanmış CLI ile de
çalışır. Giriş noktası varsayılan paket giriş noktası değilse `--entry <path>` kullanın.
Dosyaları yeniden yazmadan oluşturulan meta veriler güncel olmadığında işlemin başarısız
olması için CI'da `plugins build --check` kullanın.

### Sağlayıcı iskeleti

```bash
openclaw plugins init acme-models --name "Acme Models" --type provider
cd acme-models
npm install
npm run build
npm test
npm run validate
```

Sağlayıcı iskeletleri; API anahtarı kimlik doğrulama tesisatı, `clawhub package validate`
çalıştıran bir `npm run validate` betiği, ClawHub paket meta verileri ve gelecekte
GitHub OIDC üzerinden güvenilir yayımlama için elle tetiklenen bir GitHub Actions iş
akışı içeren, OpenAI uyumlu genel bir model sağlayıcı plugini oluşturur. Sağlayıcı
iskeletleri Skills oluşturmaz ve `openclaw plugins build`/`validate` kullanmaz;
bu komutlar araç iskeletinin oluşturulan meta veri yolu içindir.

Yayımlamadan önce yer tutucu API temel URL'sini, model kataloğunu, doküman
rotasını, kimlik bilgisi metnini ve README metnini gerçek sağlayıcı ayrıntılarıyla
değiştirin. İlk ClawHub yayını ve güvenilir yayımcı kurulumu için oluşturulan
README'yi kullanın.

## Yükleme

```bash
openclaw plugins search "calendar"                      # ClawHub pluginlerinde ara
openclaw plugins install @openclaw/<package>            # güvenilir resmî katalog
openclaw plugins install <package>                       # herhangi bir npm paketi
openclaw plugins install clawhub:<package>                # yalnızca ClawHub
openclaw plugins install npm:<package>                    # yalnızca npm
openclaw plugins install npm-pack:<path.tgz>               # yerel npm-pack tarball dosyası
openclaw plugins install git:github.com/<owner>/<repo>     # git deposu
openclaw plugins install git:github.com/<owner>/<repo>@<ref>
openclaw plugins install <path>                            # yerel yol veya arşiv
openclaw plugins install -l <path>                         # kopyalamak yerine bağla
openclaw plugins install <plugin>@<marketplace>             # marketplace kısa gösterimi
openclaw plugins install <plugin> --marketplace <name>      # marketplace (açıkça belirtilmiş)
openclaw plugins install <package> --force                  # kaynağı onayla / mevcut olanın üzerine yaz
openclaw plugins install <package> --pin                    # çözümlenen npm sürümünü sabitle
openclaw plugins install clawhub:<package> --acknowledge-clawhub-risk
openclaw plugins install <package> --dangerously-force-unsafe-install
```

Kurulum sırasında yapılan yüklemeleri test eden bakımcılar, korumalı ortam
değişkenleriyle otomatik plugin yükleme kaynaklarını geçersiz kılabilir. Bkz.
[Plugin yükleme geçersiz kılmaları](/tr/plugins/install-overrides).

<Warning>
Geçiş sürecinde yalın paket adları varsayılan olarak npm'den yüklenir; ancak birlikte gelen veya resmî bir plugin kimliğiyle eşleşirlerse OpenClaw, npm kayıt defterine erişmek yerine bu yerel/resmî kopyayı kullanır. Bilerek haricî bir npm paketi istediğinizde `npm:<package>` kullanın. ClawHub için `clawhub:<package>` kullanın. Plugin yüklemelerini kod çalıştırmak gibi değerlendirin; sabitlenmiş sürümleri tercih edin.
</Warning>

<Warning>
ClawHub paketleri ile OpenClaw'ın birlikte gelen/resmî kataloğu güvenilir yükleme
kaynaklarıdır. Yeni ve rastgele seçilmiş bir npm, `npm-pack:`, git, yerel yol/arşiv
veya marketplace kaynağı, devam etmeden önce uyarı gösterir ve onay ister. Etkileşimsiz
rastgele kaynak yüklemelerinde, kaynağı inceleyip güvenilir bulduktan sonra
`--force` iletilmelidir. Aynı bayrak, gerektiğinde mevcut yükleme hedefinin
üzerine yazar. Zaten izlenen bir yüklemenin normal güncellemeleri bunu gerektirmez.
Bu onay, yalnızca riskli ClawHub sürümü güven uyarıları için geçerli olan
`--acknowledge-clawhub-risk` seçeneğinden ayrıdır. `--force`, `security.installPolicy`
veya kalan yükleme güvenliği denetimlerini atlamaz.
</Warning>

`plugins search`, yüklenebilir `code-plugin` ve `bundle-plugin`
paketleri için ClawHub'ı sorgular (Skills için değildir; bunlar için
`openclaw skills search` kullanın). Varsayılan `--limit` değeri 20'dir ve üst
sınır 100'dür. Yalnızca uzak kataloğu okur: yerel durum incelemesi, yapılandırma
değişikliği, paket yüklemesi veya plugin çalışma zamanı yüklemesi yapmaz. Sonuçlar;
ClawHub paket adını, ailesini, kanalını, sürümünü, özetini ve
`openclaw plugins install clawhub:<package>` gibi bir yükleme ipucunu içerir.

<Note>
ClawHub, çoğu plugin için birincil dağıtım ve keşif yüzeyidir. Npm, desteklenen
bir yedek ve doğrudan yükleme yolu olmaya devam eder. OpenClaw'a ait
`@openclaw/*` plugin paketleri yeniden npm'de yayımlanmaktadır; güncel liste
için [npmjs.com/org/openclaw](https://www.npmjs.com/org/openclaw) veya
[plugin envanteri](/tr/plugins/plugin-inventory) sayfasına bakın. Kararlı yüklemeler
`latest` kullanır. Beta kanalı yüklemeleri ve güncellemeleri,
mevcut olduğunda npm `beta` dağıtım etiketini tercih eder; aksi hâlde
`latest` kullanılır. Genişletilmiş kararlı kanalda yalın/varsayılan veya
`latest` amacı taşıyan resmî npm pluginleri, yüklü çekirdeğin tam sürümüne
çözümlenir. Tam sabitlemeler ve açıkça belirtilen `latest` dışı etiketler,
üçüncü taraf paketleri ve npm dışı kaynaklar yeniden yazılmaz.
</Note>

<AccordionGroup>
  <Accordion title="Yapılandırma eklemeleri ve geçersiz yapılandırma onarımı">
    `plugins` bölümünüz tek dosyalı bir `$include` tarafından destekleniyorsa `plugins install/update/enable/disable/uninstall`, eklenen bu dosyaya yazar ve `openclaw.json` dosyasına dokunmaz. Kök eklemeler, ekleme dizileri ve kardeş geçersiz kılmalar içeren eklemeler düzleştirilmek yerine güvenli biçimde başarısız olur. Desteklenen biçimler için [Yapılandırma eklemeleri](/tr/gateway/configuration) bölümüne bakın.

    Yüklemeden önce yapılandırma geçersizse `plugins install` normalde güvenli biçimde başarısız olur ve önce `openclaw doctor --fix` çalıştırılmasını bildirir. Gateway başlangıcı ve çalışırken yeniden yükleme sırasında geçersiz plugin yapılandırması, diğer tüm geçersiz yapılandırmalar gibi güvenli biçimde başarısız olur; `openclaw doctor --fix` geçersiz plugin girdisini karantinaya alabilir. Önceden mevcut yapılandırmaya ilişkin tek istisna, açıkça `openclaw.install.allowInvalidConfigRecovery` seçeneğini etkinleştiren pluginlere yönelik dar kapsamlı bir birlikte gelen plugin kurtarma yoludur.

    Mevcut ana makine yapılandırması geçerli ancak yeni yüklenen pluginin kendi yapılandırması yoksa OpenClaw, geçersiz ve etkin bir girdi yazmak yerine yüklemeyi devre dışı olarak kaydeder. `plugins.entries.<id>.config` değerini yapılandırın, ardından `openclaw plugins enable <id>` çalıştırın. Mevcut bir plugin yapılandırma girdisi varsa ancak geçersizse yükleme, girdiyi yeniden yazmadan başarısız olur.

  </Accordion>
  <Accordion title="--force onayı ve yeniden yükleme ile güncelleme arasındaki fark">
    `--force`, ClawHub dışı bir kaynağı istem göstermeden onaylar. `security.installPolicy` veya kalan yükleme güvenliği denetimlerini atlamaz. Plugin veya hook paketi zaten yüklüyse mevcut hedefi yeniden kullanır ve yerinde üzerine yazar. Herhangi bir npm, yerel, arşiv, git veya marketplace kaynağını inceledikten sonra ya da aynı kimliği bilerek yeniden yüklerken kullanın. Zaten izlenen bir npm plugininin rutin yükseltmeleri için `openclaw plugins update <id-or-npm-spec>` tercih edin.

    Zaten yüklü olan bir plugin kimliği için `plugins install` çalıştırırsanız OpenClaw durur ve normal yükseltme için `plugins update <id-or-npm-spec>`, mevcut yüklemeyi gerçekten farklı bir kaynaktan gelen sürümle değiştirmek istediğinizde ise `plugins install <package> --force` kullanmanızı söyler. Rastgele kaynaklarda etkileşimli kaynak kökeni uyarısı gösterilmeye devam eder; etkileşimsiz yüklemelerde inceleme sonrasında `--force` iletilmelidir. Güvenilir ClawHub ve OpenClaw katalog kaynakları bunu gerektirmez. `--link` kullanıldığında `--force` kaynağı onaylar ancak bağlantılı yol yükleme modunu değiştirmez.

  </Accordion>
  <Accordion title="--pin kapsamı">
    `--pin` yalnızca npm kurulumları için geçerlidir ve çözümlenen tam `<name>@<version>` değerini kaydeder. `git:` kurulumlarında (bunun yerine ref'i belirtimde sabitleyin; ör. `git:github.com/acme/plugin@v1.2.3`) veya `--marketplace` ile desteklenmez (marketplace kurulumları, npm belirtimi yerine marketplace kaynak meta verilerini kalıcı olarak saklar).
  </Accordion>
  <Accordion title="--dangerously-force-unsafe-install">
    `--dangerously-force-unsafe-install` kullanımdan kaldırılmıştır ve artık hiçbir işlem yapmaz. OpenClaw, Plugin kurulumları için yerleşik kurulum zamanı tehlikeli kod engellemesini artık çalıştırmaz.

    Ana makineye özgü kurulum politikası gerektiğinde operatörün yönettiği `security.installPolicy` yüzeyini kullanın. Plugin `before_install` kancaları, Plugin çalışma zamanı yaşam döngüsü kancalarıdır; CLI kurulumları için birincil politika sınırı değildir.

    ClawHub'da yayımladığınız bir Plugin, kayıt defteri taraması tarafından gizlenir veya engellenirse [ClawHub'da yayımlama](/tr/clawhub/publishing) bölümündeki yayımcı adımlarını kullanın. `--dangerously-force-unsafe-install`, ClawHub'dan Plugin'i yeniden taramasını veya engellenen bir sürümü herkese açık hâle getirmesini istemez.

  </Accordion>
  <Accordion title="--acknowledge-clawhub-risk">
    Topluluk ClawHub kurulumları, indirmeden önce seçilen sürümün güven kaydını denetler. ClawHub sürümün indirilmesini devre dışı bırakırsa, kötü amaçlı tarama bulguları bildirirse veya sürümü engelleyici bir moderasyon durumuna (karantinaya alınmış, iptal edilmiş) getirirse OpenClaw, bu bayraktan bağımsız olarak sürümü doğrudan reddeder. Engelleyici olmayan riskli tarama durumlarında veya moderasyon durumlarında OpenClaw güven ayrıntılarını gösterir ve devam etmeden önce onay ister.

    `--acknowledge-clawhub-risk` seçeneğini yalnızca ClawHub uyarısını inceledikten ve etkileşimli bir istem olmadan devam etmeye karar verdikten sonra kullanın. Bekleyen veya eski (henüz temiz olmayan) tarama sonuçları uyarı verir ancak onay gerektirmez. Resmî ClawHub paketleri ve paketle birlikte sunulan OpenClaw Plugin kaynakları bu sürüm güven denetimini tamamen atlar.

  </Accordion>
  <Accordion title="Kanca paketleri ve npm belirtimleri">
    `plugins install`, `package.json` içinde `openclaw.hooks` sunan kanca paketlerinin de kurulum yüzeyidir. Paket kurulumu için değil, filtrelenmiş kanca görünürlüğü ve kanca başına etkinleştirme için `openclaw hooks` kullanın.

    Npm belirtimleri **yalnızca kayıt defterine yöneliktir** (paket adı ile isteğe bağlı **tam sürüm** veya **dist-tag**). Git/URL/dosya belirtimleri ve semver aralıkları reddedilir. Bağımlılık kurulumları, kabuğunuzda genel npm kurulum ayarları olsa bile güvenlik amacıyla her Plugin için `--ignore-scripts` ile yönetilen tek bir npm projesinde çalışır. Yönetilen Plugin npm projeleri, OpenClaw'ın paket düzeyindeki npm `overrides` ayarını devralır; böylece ana makine güvenlik sabitlemeleri yukarı taşınan Plugin bağımlılıklarına da uygulanır.

    npm çözümlemesini açık hâle getirmek için `npm:<package>` kullanın. Çıplak paket belirtimleri de resmî bir Plugin kimliğiyle eşleşmedikleri sürece geçiş sırasında doğrudan npm'den kurulur.

    Paketle birlikte sunulan Plugin'lerle eşleşen ham `@openclaw/*` belirtimleri, npm geri dönüşünden önce imajın sahip olduğu paketlenmiş kopyaya çözümlenir. Örneğin `openclaw plugins install @openclaw/discord@2026.5.20 --pin`, yönetilen bir npm geçersiz kılması oluşturmak yerine mevcut OpenClaw derlemesindeki paketlenmiş Discord Plugin'ini kullanır. Haricî npm paketini zorlamak için `openclaw plugins install npm:@openclaw/discord@2026.5.20 --pin` kullanın.

    Çıplak belirtimler ve `@latest` kararlı kanalda kalır. `2026.5.3-1` gibi OpenClaw tarih damgalı düzeltme sürümleri bu denetimde kararlı kabul edilir. npm bu biçimlerden birini bir ön sürüme çözümlerse OpenClaw durur ve bir ön sürüm etiketiyle (`@beta`/`@rc`) veya tam bir ön sürüm sürümüyle (`@1.2.3-beta.4`) açıkça katılmanızı ister.

    Tam sürüm içermeyen npm kurulumlarında (`npm:<package>` veya `npm:<package>@latest`) OpenClaw, kurulumdan önce çözümlenen paket meta verilerini denetler. En son kararlı paket daha yeni bir OpenClaw Plugin API'si veya daha yüksek bir asgari ana makine sürümü gerektiriyorsa OpenClaw eski kararlı sürümleri inceler ve bunun yerine uyumlu en yeni sürümü kurar. Tam sürümler ve açık dist-tag'ler katı kalır: uyumsuz bir seçim başarısız olur ve OpenClaw'ı yükseltmenizi veya uyumlu bir sürüm seçmenizi ister.

    Çıplak bir kurulum belirtimi resmî bir Plugin kimliğiyle eşleşirse (örneğin `diffs`) OpenClaw katalog girdisini doğrudan kurar. Aynı ada sahip bir npm paketini kurmak için açık kapsamlı bir belirtim kullanın (örneğin `@scope/diffs`).

  </Accordion>
  <Accordion title="Git depoları">
    Doğrudan bir git deposundan kurmak için `git:<repo>` kullanın. Desteklenen biçimler: `git:github.com/owner/repo`, `git:owner/repo`, tam `https://`, `ssh://`, `git://`, `file://` ve `git@host:owner/repo.git` klonlama URL'leri. Kurulumdan önce bir dalı, etiketi veya commit'i teslim almak için `@<ref>` ya da `#<ref>` ekleyin.

    Git kurulumları geçici bir dizine klonlar, varsa istenen ref'i teslim alır ve ardından normal Plugin dizini yükleyicisini kullanır; böylece manifest doğrulaması, operatör kurulum politikası, paket yöneticisi kurulum işlemleri ve kurulum kayıtları npm kurulumlarındaki gibi davranır. Kaydedilen git kurulumları, kaynak URL'sini/ref'ini ve çözümlenen commit'i içerir; böylece `openclaw plugins update` kaynağı daha sonra yeniden çözümleyebilir.

    Git'ten kurulum yaptıktan sonra Gateway yöntemleri ve CLI komutları gibi çalışma zamanı kayıtlarını doğrulamak için `openclaw plugins inspect <id> --runtime --json` kullanın. Plugin, `api.registerCli` ile bir CLI kökü kaydettiyse bu komutu doğrudan OpenClaw kök CLI'si üzerinden çalıştırın; örneğin `openclaw demo-plugin ping`.

  </Accordion>
  <Accordion title="Arşivler">
    Desteklenen arşivler: `.zip`, `.tgz`, `.tar.gz`, `.tar`. Yerel OpenClaw Plugin arşivleri, çıkarılan Plugin kökünde geçerli bir `openclaw.plugin.json` içermelidir; yalnızca `package.json` içeren arşivler, OpenClaw kurulum kayıtlarını yazmadan önce reddedilir.

    Dosya bir npm-pack tarball'ı olduğunda ve kayıt defteri kurulumları tarafından kullanılan
    Plugin başına yönetilen npm projesi yolunun aynısını; `package-lock.json` doğrulaması,
    yukarı taşınan bağımlılık taraması ve npm kurulum kayıtları dâhil olmak üzere kullanmak istediğinizde
    `npm-pack:<path.tgz>` kullanın. Düz arşiv yolları, Plugin uzantıları kökü altına yerel
    arşivler olarak kurulmaya devam eder.

    Claude marketplace kurulumları da desteklenir.

  </Accordion>
</AccordionGroup>

ClawHub kurulumları açık bir `clawhub:<package>` konumlandırıcısı kullanır:

```bash
openclaw plugins install clawhub:openclaw-codex-app-server
openclaw plugins install clawhub:openclaw-codex-app-server@1.2.3
```

Npm açısından güvenli çıplak Plugin belirtimleri, resmî bir Plugin kimliğiyle eşleşmedikleri sürece geçiş sırasında varsayılan olarak npm'den kurulur:

```bash
openclaw plugins install openclaw-codex-app-server
```

Yalnızca npm çözümlemesini açık hâle getirmek için `npm:` kullanın:

```bash
openclaw plugins install npm:openclaw-codex-app-server
openclaw plugins install npm:@openclaw/discord@2026.5.20
openclaw plugins install npm:@scope/plugin-name@1.0.1
```

OpenClaw, kurulumdan önce duyurulan Plugin API'si / asgari Gateway uyumluluğunu denetler. Seçilen ClawHub sürümü bir ClawPack yapıtı yayımladığında OpenClaw sürümlendirilmiş npm-pack `.tgz` dosyasını indirir, ClawHub özet üst bilgisini ve yapıt özetini doğrular, ardından normal arşiv yolu üzerinden kurar. ClawPack meta verileri bulunmayan eski ClawHub sürümleri, eski paket arşivi doğrulama yolu üzerinden kurulmaya devam eder. Kaydedilen kurulumlar daha sonraki güncellemeler için ClawHub kaynak meta verilerini, yapıt türünü, npm bütünlük değerini, npm shasum değerini, tarball adını ve ClawPack özet bilgilerini korur.
Sürümlendirilmemiş ClawHub kurulumları, `openclaw plugins update` daha yeni ClawHub sürümlerini takip edebilsin diye sürümlendirilmemiş bir kayıtlı belirtimi korur; `clawhub:pkg@1.2.3` ve `clawhub:pkg@beta` gibi açık sürüm veya etiket seçicileri söz konusu seçiciye sabitlenmiş olarak kalır.

### Marketplace kısa gösterimi

Marketplace adı Claude'un `~/.claude/plugins/known_marketplaces.json` konumundaki yerel kayıt defteri önbelleğinde bulunduğunda `plugin@marketplace` kısa gösterimini kullanın:

```bash
openclaw plugins marketplace list <marketplace-name>
openclaw plugins install <plugin-name>@<marketplace-name>
```

Marketplace kaynağını açıkça iletmek için `--marketplace` kullanın:

```bash
openclaw plugins install <plugin-name> --marketplace <marketplace-name>
openclaw plugins install <plugin-name> --marketplace <owner/repo>
openclaw plugins install <plugin-name> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <plugin-name> --marketplace ./my-marketplace
```

<Tabs>
  <Tab title="Marketplace kaynakları">
    - `~/.claude/plugins/known_marketplaces.json` içindeki, Claude tarafından bilinen bir marketplace adı
    - yerel bir marketplace kökü veya `marketplace.json` yolu
    - `owner/repo` gibi bir GitHub deposu kısa gösterimi
    - `https://github.com/owner/repo` gibi bir GitHub deposu URL'si
    - bir git URL'si

  </Tab>
  <Tab title="Uzak marketplace kuralları">
    GitHub veya git'ten yüklenen uzak marketplace'lerde Plugin girdileri klonlanan marketplace deposunun içinde kalmalıdır. OpenClaw bu depodaki göreli yol kaynaklarını kabul eder; uzak manifestlerdeki HTTP(S), mutlak yol, git, GitHub ve yol olmayan diğer Plugin kaynaklarını reddeder.
  </Tab>
</Tabs>

OpenClaw, yerel yollar ve arşivler için şunları otomatik olarak algılar:

- yerel OpenClaw Plugin'leri (`openclaw.plugin.json`)
- Codex uyumlu paketler (`.codex-plugin/plugin.json`)
- Claude uyumlu paketler (`.claude-plugin/plugin.json` veya bu manifest dosyası yoksa varsayılan Claude bileşen düzeni)
- Cursor uyumlu paketler (`.cursor-plugin/plugin.json`)

Yönetilen yerel kurulumlar Plugin dizinleri veya arşivleri olmalıdır. Bağımsız `.js`,
`.mjs`, `.cjs` ve `.ts` Plugin dosyaları `plugins install` tarafından yönetilen Plugin
köküne kopyalanmaz ve doğrudan
`~/.openclaw/extensions` veya `<workspace>/.openclaw/extensions` içine yerleştirilerek yüklenmez; bu
otomatik olarak keşfedilen kökler Plugin paketi veya paket dizinlerini yükler ve
üst düzey betik dosyalarını yerel yardımcılar olarak atlar. Bunun yerine bağımsız dosyaları
`plugins.load.paths` içinde açıkça listeleyin.

<Note>
Uyumlu paketler normal Plugin köküne kurulur ve aynı listeleme/bilgi/etkinleştirme/devre dışı bırakma akışına katılır. Şu anda paket Skills'ları, Claude komut Skills'ları, Claude `settings.json` varsayılanları, Claude `.lsp.json` / manifestte bildirilen `lspServers` varsayılanları, Cursor komut Skills'ları ve uyumlu Codex kanca dizinleri desteklenir; algılanan diğer paket yetenekleri tanılama/bilgi bölümünde gösterilir ancak henüz çalışma zamanı yürütmesine bağlanmamıştır.
</Note>

Yerel bir Plugin dizinini kopyalamadan göstermek için `-l`/`--link` kullanın
(`plugins.load.paths` öğesine ekler):

```bash
openclaw plugins install -l ./my-plugin
```

`--link`, `--marketplace` veya `git:` kurulumlarıyla desteklenmez ve
hâlihazırda var olan yerel bir yol gerektirir. Etkileşimsiz bir yerel bağlantı için,
kaynağı inceledikten sonra `--force` iletin; bu seçenek kaynağı doğrular ancak
bağlanan dizini kopyalamaz veya üzerine yazmaz.

<Note>
Bir çalışma alanı uzantıları kökünden keşfedilen çalışma alanı kaynaklı Plugin'ler,
açıkça etkinleştirilene kadar içe aktarılmaz veya yürütülmez. Yerel geliştirme için
`openclaw plugins enable <plugin-id>` çalıştırın ya da
`plugins.entries.<plugin-id>.enabled: true` ayarlayın; yapılandırmanız
`plugins.allow` kullanıyorsa aynı Plugin kimliğini oraya da ekleyin. Bu kapalı hata verme kuralı,
kanal kurulumu yalnızca kurulum amaçlı yükleme için çalışma alanı kaynaklı bir Plugin'i açıkça hedeflediğinde de
geçerlidir; dolayısıyla söz konusu çalışma alanı Plugin'i devre dışı veya izin listesinin dışında kaldığı sürece
yerel kanal Plugin'i kurulum kodu çalışmaz. Bağlantılı kurulumlar
ve açık `plugins.load.paths` girdileri, çözümlenen Plugin kaynakları için normal politikayı
izler. Bkz.
[Plugin politikasını yapılandırma](/tr/tools/plugin#configure-plugin-policy)
ve [Yapılandırma referansı](/tr/gateway/configuration-reference#plugins).

Varsayılan davranışı sabitlenmemiş durumda tutarken çözümlenen tam belirtimi (`name@version`) yönetilen Plugin dizinine kaydetmek için npm kurulumlarında `--pin` kullanın.
</Note>

## Listeleme

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
```

<ParamField path="--enabled" type="boolean">
  Yalnızca etkin Plugin'leri gösterir.
</ParamField>
<ParamField path="--verbose" type="boolean">
  Tablo görünümünden biçim/kaynak/köken/sürüm/etkinleştirme meta verilerini içeren Plugin başına ayrıntı satırlarına geçer.
</ParamField>
<ParamField path="--json" type="boolean">
  Makine tarafından okunabilir envanterin yanı sıra kayıt defteri tanılamaları ve paket bağımlılığı yükleme durumu.
</ParamField>

<Note>
`plugins list` önce kalıcı yerel Plugin kayıt defterini okur; kayıt defteri eksik veya geçersiz olduğunda yalnızca manifestten türetilen bir geri dönüş kullanır. Bir Plugin'in yüklü, etkin ve soğuk başlatma planlamasında görünür olup olmadığını denetlemek için kullanışlıdır ancak hâlihazırda çalışan bir Gateway işleminin canlı çalışma zamanı yoklaması değildir. Plugin kodunu, etkinleştirmeyi, kanca politikasını veya `plugins.load.paths` öğesini değiştirdikten sonra yeni `register(api)` kodunun ya da kancalarının çalışmasını beklemeden önce kanala hizmet veren Gateway'i yeniden başlatın. Uzak/konteyner dağıtımlarında yalnızca bir sarmalayıcı işlemi değil, gerçek `openclaw gateway run` alt işlemini yeniden başlattığınızı doğrulayın.

`plugins list --json`, her Plugin'in `package.json`
`dependencies` ve `optionalDependencies` içindeki `dependencyStatus` öğesini içerir. OpenClaw, bu paket
adlarının Plugin'in normal Node `node_modules` arama yolunda bulunup bulunmadığını
denetler; Plugin çalışma zamanı kodunu içe aktarmaz, paket yöneticisi çalıştırmaz veya
eksik bağımlılıkları onarmaz.
</Note>

Başlatma günlüklerinde `plugins.allow is empty; discovered non-bundled plugins may auto-load: ...` görünürse Plugin
kimliklerini doğrulamak ve güvenilir kimlikleri `openclaw.json` içindeki
`plugins.allow` öğesine kopyalamak için listelenen bir Plugin kimliğiyle
`openclaw plugins list --enabled --verbose` veya `openclaw plugins inspect <id>` komutunu çalıştırın. Uyarı keşfedilen
tüm Plugin'leri listeleyebildiğinde, bu kimlikleri zaten içeren ve doğrudan
yapıştırılmaya hazır bir `plugins.allow` parçacığı yazdırır. Bir Plugin
yükleme/yükleme-yolu köken bilgisi olmadan yüklenirse bu Plugin kimliğini inceleyin,
ardından güvenilir kimliği `plugins.allow` içinde sabitleyin veya OpenClaw'un
yükleme köken bilgisini kaydetmesi için Plugin'i güvenilir bir kaynaktan yeniden yükleyin.

Paketlenmiş bir Docker imajı içindeki paketle birlikte gelen Plugin çalışmaları için
Plugin kaynak dizinini `/app/extensions/synology-chat` gibi eşleşen paketlenmiş kaynak yolunun
üzerine bağlama noktası olarak bağlayın. OpenClaw, bağlanan bu kaynak katmanını
`/app/dist/extensions/synology-chat` öğesinden önce keşfeder; yalnızca kopyalanmış bir kaynak dizini
etkisiz kalır, dolayısıyla normal paketlenmiş yüklemeler derlenmiş dist'i kullanmaya devam eder.

Çalışma zamanı kancası hata ayıklaması için:

- `openclaw plugins inspect <id> --runtime --json`, modülün yüklendiği bir inceleme geçişinden kaydedilmiş kancaları ve tanılamaları gösterir. Çalışma zamanı incelemesi hiçbir zaman bağımlılık yüklemez; eski bağımlılık durumunu temizlemek veya yapılandırmada başvurulan eksik indirilebilir Plugin'leri kurtarmak için `openclaw doctor --fix` kullanın.
- `openclaw gateway status --deep --require-rpc`; erişilebilir Gateway URL'sini/profilini, hizmet/işlem ipuçlarını, yapılandırma yolunu ve RPC durumunu doğrular.
- Paketle birlikte gelmeyen konuşma kancaları (`llm_input`, `llm_output`, `before_model_resolve`, `before_agent_reply`, `before_agent_run`, `before_agent_finalize`, `agent_end`) `plugins.entries.<id>.hooks.allowConversationAccess=true` gerektirir.

### Plugin dizini

Plugin yükleme meta verileri, kullanıcı yapılandırması değil, makine tarafından yönetilen durumdur. Yüklemeler ve güncellemeler bunları etkin OpenClaw durum dizini altındaki paylaşılan SQLite durum veritabanına yazar. `installed_plugin_index` satırı; bozuk veya eksik Plugin manifestlerinin kayıtları ile `openclaw plugins update`, kaldırma, tanılama ve soğuk Plugin kayıt defteri tarafından kullanılan manifestten türetilmiş soğuk kayıt defteri önbelleği dâhil olmak üzere kalıcı `installRecords` meta verilerini saklar.

`plugins.installs`, kullanımdan kaldırılmış bir yazılmış-yapılandırma yüzeyidir. Çalışma zamanı ve güncelleme komutları yalnızca SQLite yüklü-Plugin dizinini okur. Normal çalışma zamanı kullanımından önce eski yapılandırma kayıtlarını dizine aktarmak ve kullanımdan kaldırılmış anahtarı kaldırmak için `openclaw doctor --fix` komutunu çalıştırın.

## Kaldırma

```bash
openclaw plugins uninstall <id>
openclaw plugins uninstall <id> --dry-run
openclaw plugins uninstall <id> --keep-files
openclaw plugins uninstall <id> --force
```

`uninstall`; Plugin kayıtlarını `plugins.entries` öğesinden, kalıcı Plugin dizininden, Plugin izin/ret listesi girdilerinden ve uygulanabildiğinde bağlantılı `plugins.load.paths` girdilerinden kaldırır. `--keep-files` ayarlanmadığı sürece kaldırma işlemi, izlenen yönetilen yükleme dizinini de kaldırır; ancak yalnızca bu dizin OpenClaw'un Plugin uzantıları kökü içinde çözümleniyorsa. Plugin şu anda `memory` veya `contextEngine` yuvasının sahibiyse bu yuva varsayılanına sıfırlanır (bellek için `memory-core`, bağlam motoru için `legacy`).

`uninstall`, kaldırılacakların önizlemesini yazdırır ve değişiklik yapmadan önce `Uninstall plugin "<id>"?` istemini gösterir. Onay istemini atlamak için `--force` geçirin (betikler ve etkileşimsiz çalıştırmalar için kullanışlıdır); bu seçenek olmadan kaldırma işlemi etkileşimli bir TTY gerektirir. `--dry-run` aynı önizlemeyi yazdırır ve istem göstermeden veya herhangi bir değişiklik yapmadan çıkar.

<Note>
`--keep-config`, `--keep-files` için kullanımdan kaldırılmış bir diğer ad olarak desteklenir.
</Note>

## Güncelleme

```bash
openclaw plugins update <id-or-npm-spec>
openclaw plugins update --all
openclaw plugins update <id-or-npm-spec> --dry-run
openclaw plugins update @openclaw/voice-call
openclaw plugins update @acme/demo
openclaw plugins update openclaw-codex-app-server --acknowledge-clawhub-risk
openclaw plugins update openclaw-codex-app-server --dangerously-force-unsafe-install
```

Güncellemeler, yönetilen Plugin dizinindeki izlenen Plugin yüklemelerine ve paylaşılan SQLite durumundaki izlenen kanca paketi yüklemelerine uygulanır. Kullanıcının Plugin'i yüklerken zaten seçtiği kaynağı yeniden kullandıkları için ikinci bir kaynak onayı gerektirmezler.

<AccordionGroup>
  <Accordion title="Plugin kimliğinin npm belirtiminden ayrıştırılması">
    Bir Plugin kimliği geçirdiğinizde OpenClaw, bu Plugin için kayıtlı yükleme belirtimini yeniden kullanır. Bu, `@beta` gibi önceden saklanan dist-tag'lerin ve tam olarak sabitlenmiş sürümlerin sonraki `update <id>` çalıştırmalarında kullanılmaya devam edeceği anlamına gelir.

    `update <id> --dry-run` sırasında tam olarak sabitlenmiş npm yüklemeleri sabit kalır. OpenClaw ayrıca paketin kayıt defteri varsayılan hattını çözümleyebiliyorsa ve bu varsayılan hat yüklü sabitlenmiş sürümden daha yeniyse deneme çalıştırması sabitlemeyi bildirir ve kayıt defteri varsayılan hattını izlemek için açık `@latest` paket güncelleme komutunu yazdırır.

    Bu hedefli güncelleme kuralı, toplu `openclaw plugins update --all` bakım yolundan farklıdır. Toplu güncellemeler normal izlenen yükleme belirtimlerine uymaya devam eder ancak güvenilir resmî OpenClaw Plugin kayıtları, eski bir tam resmî pakette kalmak yerine güncel resmî katalog hedefine eşitlenebilir. Tam veya etiketlenmiş resmî bir belirtimi kasıtlı olarak değiştirmeden korumak istediğinizde hedefli `update <id>` kullanın.

    npm yüklemelerinde dist-tag veya tam sürüm içeren açık bir npm paket belirtimi de geçirebilirsiniz. OpenClaw bu paket adını izlenen Plugin kaydına geri çözümler, yüklü Plugin'i günceller ve gelecekteki kimlik tabanlı güncellemeler için yeni npm belirtimini kaydeder.

    npm paket adını sürüm veya etiket olmadan geçirmek de izlenen Plugin kaydına geri çözümlenir. Bir Plugin tam bir sürüme sabitlenmişse ve onu kayıt defterinin varsayılan sürüm hattına geri taşımak istiyorsanız bunu kullanın.

  </Accordion>
  <Accordion title="Beta kanalı güncellemeleri">
    Hedefli `openclaw plugins update <id-or-npm-spec>`, yeni bir belirtim geçirmediğiniz sürece izlenen Plugin belirtimini yeniden kullanır. Toplu `openclaw plugins update --all`, güvenilir resmî Plugin kayıtlarını resmî katalog hedefine eşitlerken yapılandırılmış `update.channel` öğesini kullanır; böylece beta kanalı yüklemeleri sessizce kararlı/latest sürümüne normalleştirilmek yerine beta sürüm hattında kalabilir.

    `openclaw update` etkin OpenClaw güncelleme kanalını da bilir: beta kanalında varsayılan hat npm ve ClawHub Plugin kayıtları önce `@beta` öğesini dener. Hiçbir Plugin beta sürümü yoksa kayıtlı varsayılan/latest belirtimine geri dönerler; npm Plugin'leri ayrıca beta paketi mevcut olduğu hâlde yükleme doğrulaması başarısız olduğunda da geri döner. Bu geri dönüş uyarı olarak bildirilir ve çekirdek güncellemesinin başarısız olmasına neden olmaz. Tam sürümler ve açık etiketler, hedefli güncellemelerde bu seçiciye sabitlenmiş olarak kalır.

  </Accordion>
  <Accordion title="Sürüm denetimleri ve bütünlük sapması">
    Canlı bir npm güncellemesinden önce OpenClaw, yüklü paket sürümünü npm kayıt defteri meta verilerine göre denetler. Yüklü sürüm ve kayıtlı yapıt kimliği çözümlenen hedefle zaten eşleşiyorsa güncelleme; indirme, yeniden yükleme veya `openclaw.json` öğesini yeniden yazma yapılmadan atlanır.

    Saklanan bir bütünlük karması mevcutken getirilen yapıtın karması değişirse OpenClaw bunu npm yapıt sapması olarak değerlendirir. Etkileşimli `openclaw plugins update` komutu beklenen ve gerçek karmaları yazdırır ve devam etmeden önce onay ister. Etkileşimsiz güncelleme yardımcıları, çağıran açık bir devam politikası sağlamadığı sürece güvenli biçimde başarısız olur.

  </Accordion>
  <Accordion title="Güncellemede --dangerously-force-unsafe-install">
    `--dangerously-force-unsafe-install`, uyumluluk amacıyla `plugins update` üzerinde de kabul edilir ancak kullanımdan kaldırılmıştır ve artık Plugin güncelleme davranışını değiştirmez. Operatör `security.installPolicy` öğesi güncellemeleri hâlâ engelleyebilir; Plugin `before_install` kancaları yalnızca Plugin kancalarının yüklendiği işlemlerde uygulanır.
  </Accordion>
  <Accordion title="Güncellemede --acknowledge-clawhub-risk">
    Topluluk ClawHub destekli Plugin güncellemeleri, yerine geçecek paketi indirmeden önce yüklemelerle aynı tam sürüm güven denetimini çalıştırır. Seçilen ClawHub sürümünde riskli bir güven uyarısı bulunduğunda devam etmesi gereken incelenmiş otomasyon için `--acknowledge-clawhub-risk` kullanın. Resmî ClawHub paketleri ve paketle birlikte gelen OpenClaw Plugin kaynakları bu sürüm güven istemini atlar.
  </Accordion>
</AccordionGroup>

## İnceleme

```bash
openclaw plugins inspect <id>
openclaw plugins inspect <id> --runtime
openclaw plugins inspect <id> --json
openclaw plugins inspect --all
```

İnceleme, varsayılan olarak Plugin çalışma zamanını içe aktarmadan kimliği, yükleme durumunu, kaynağı, manifest yeteneklerini, politika bayraklarını, tanılamaları, yükleme meta verilerini, paket yeteneklerini ve algılanan tüm MCP veya LSP sunucusu desteğini gösterir. JSON çıktısı, `contracts.agentToolResultMiddleware` ve `contracts.trustedToolPolicies` gibi Plugin manifest sözleşmelerini içerir; böylece operatörler bir Plugin'i etkinleştirmeden veya yeniden başlatmadan önce güvenilir yüzey bildirimlerini denetleyebilir. Plugin modülünü yüklemek ve kayıtlı kancaları, araçları, komutları, hizmetleri, Gateway yöntemlerini ve HTTP rotalarını dâhil etmek için `--runtime` ekleyin. Çalışma zamanı incelemesi eksik Plugin bağımlılıklarını doğrudan bildirir; yüklemeler ve onarımlar `openclaw plugins install`, `openclaw plugins update` ve `openclaw doctor --fix` içinde kalır.

Plugin'e ait CLI komutları genellikle kök `openclaw` komut grupları olarak yüklenir ancak Plugin'ler `openclaw nodes` gibi bir çekirdek üst öğenin altına iç içe komutlar da kaydedebilir. `inspect --runtime` bir komutu `cliCommands` altında gösterdikten sonra komutu listelenen yolda çalıştırın; örneğin `demo-git` kaydeden bir Plugin, `openclaw demo-git ping` ile doğrulanabilir.

Her Plugin, çalışma zamanında gerçekten kaydettiği öğelere göre sınıflandırılır:

| Biçim              | Anlamı                                                            |
| ------------------- | ----------------------------------------------------------------- |
| `plain-capability`  | tam olarak bir yetenek türü (ör. yalnızca sağlayıcı Plugin'i)     |
| `hybrid-capability` | birden fazla yetenek türü (ör. metin + konuşma + görseller)       |
| `hook-only`         | yalnızca kancalar; yetenek, araç, komut, hizmet veya rota yok |
| `non-capability`    | araçlar/komutlar/hizmetler var ancak yetenek yok                  |

Yetenek modeli hakkında daha fazla bilgi için [Plugin biçimleri](/tr/plugins/architecture#plugin-shapes) bölümüne bakın.

<Note>
`--json` bayrağı, betik oluşturma ve denetim için uygun, makine tarafından okunabilir bir rapor çıktısı verir. `inspect --all`; biçim, yetenek türleri, uyumluluk bildirimleri, paket yetenekleri ve kanca özeti sütunlarını içeren filo genelinde bir tablo oluşturur. `info`, `inspect` için bir diğer addır.
</Note>

## Doctor

```bash
openclaw plugins doctor
```

`doctor` Plugin yükleme hatalarını, manifest/keşif tanılamalarını, uyumluluk bildirimlerini ve eksik Plugin yuvaları gibi eski Plugin yapılandırma referanslarını bildirir. Kurulum ağacı ve Plugin yapılandırması temiz olduğunda `No plugin issues detected.` yazdırır. Eski yapılandırma kalmış ancak kurulum ağacı bunun dışında sağlıklıysa özet, Plugin'lerin tamamen sağlıklı olduğunu ima etmek yerine bunu belirtir.

Yapılandırılmış bir Plugin diskte mevcut ancak yükleyicinin yol güvenliği denetimleri tarafından engelleniyorsa yapılandırma doğrulaması Plugin girdisini korur ve bunu `present but blocked` olarak bildirir. `plugins.entries.<id>` veya `plugins.allow` yapılandırmasını kaldırmak yerine yol sahipliği ya da herkesçe yazılabilir izinler gibi, öncesinde bildirilen engellenmiş Plugin tanılamasını düzeltin.

Eksik `register`/`activate` dışa aktarımları gibi modül biçimi hatalarında, tanılama çıktısına kısa bir dışa aktarım biçimi özeti eklemek için `OPENCLAW_PLUGIN_LOAD_DEBUG=1` ile yeniden çalıştırın.

## Kayıt Defteri

```bash
openclaw plugins registry
openclaw plugins registry --refresh
openclaw plugins registry --json
```

Yerel Plugin kayıt defteri, kurulu Plugin kimliği, etkinleştirme durumu, kaynak meta verileri ve katkı sahipliği için OpenClaw'ın kalıcı soğuk okuma modelidir. Normal başlangıç, sağlayıcı sahibi araması, kanal kurulumu sınıflandırması ve Plugin envanteri, Plugin çalışma zamanı modüllerini içe aktarmadan bu kayıt defterini okuyabilir.

Kalıcı kayıt defterinin mevcut, güncel veya eski olup olmadığını incelemek için `plugins registry` kullanın. Kalıcı Plugin dizininden, yapılandırma politikasından ve manifest/paket meta verilerinden kayıt defterini yeniden oluşturmak için `--refresh` kullanın. Bu bir onarım yoludur, çalışma zamanı etkinleştirme yolu değildir.

`openclaw doctor --fix` ayrıca kayıt defteriyle ilişkili yönetilen npm sapmalarını da onarır. Yönetilen bir Plugin npm projesi veya eski düz yönetilen npm kökü altındaki yetim ya da kurtarılmış bir `@openclaw/*` paketi, paketle gelen bir Plugin'i gölgeliyorsa Doctor bu eski paketi kaldırır ve başlangıcın paketle gelen manifest üzerinden doğrulama yapması için kayıt defterini yeniden oluşturur. Yetkili bir kurulum kaydı yönetilen bir nesli seçtiği hâlde eski düz veya nesil dizinleri kalmışsa Doctor, Gateway yeniden başlatıldıktan sonra budanmaları için bu eski ağaçları kullanımdan kaldırır. Doctor ayrıca ana makinenin `openclaw` paketini, `peerDependencies.openclaw` bildiren yönetilen npm Plugin'lerine yeniden bağlar; böylece `openclaw/plugin-sdk/*` gibi pakete yerel çalışma zamanı içe aktarımları güncellemelerden veya npm onarımlarından sonra çözümlenir.

## Pazar Yeri

```bash
openclaw plugins marketplace entries
openclaw plugins marketplace entries --offline
openclaw plugins marketplace entries --json
openclaw plugins marketplace entries --feed-profile <name>
openclaw plugins marketplace entries --feed-url <url>
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json
openclaw plugins marketplace refresh
openclaw plugins marketplace refresh --feed-profile <name>
openclaw plugins marketplace refresh --feed-url <url>
openclaw plugins marketplace refresh --expected-sha256 <sha256> --json
```

`plugins marketplace entries`, yapılandırılmış OpenClaw pazar yeri akışındaki girdileri listeler. Varsayılan olarak barındırılan akışı kullanmayı dener ve başarısız olursa kabul edilen en son anlık görüntüye veya paketle gelen verilere geri döner. Belirli bir yapılandırılmış profili okumak için `--feed-profile <name>`, açıkça belirtilen barındırılmış bir akış URL'sini okumak için `--feed-url <url>` ve akışı getirmeden kabul edilen en son anlık görüntüyü okumak için `--offline` kullanın.

`plugins marketplace refresh`, yapılandırılmış barındırılmış akışın anlık görüntüsünü yeniler ve OpenClaw'ın barındırılan verileri, barındırılmış bir anlık görüntüyü veya paketle gelen geri dönüş verilerini kabul edip etmediğini bildirir. Bir çağıranın, yeni bir barındırılmış yük sabitlenmiş bir sağlama toplamıyla eşleşmediği sürece komutun başarısız olmasını istediği durumlarda `--expected-sha256` kullanın.

Pazar yeri `list`, yerel bir pazar yeri yolunu, bir `marketplace.json` yolunu, `owner/repo` gibi bir GitHub kısaltmasını, bir GitHub depo URL'sini veya bir git URL'sini kabul eder. `--json`, çözümlenen kaynak etiketinin yanı sıra ayrıştırılmış pazar yeri manifestini ve Plugin girdilerini yazdırır.

Pazar yeri yenilemesi, barındırılan bir OpenClaw pazar yeri akışını yükler ve
doğrulanmış yanıtı yerel barındırılmış akış anlık görüntüsü olarak kalıcı hâle getirir. Seçenek
belirtilmediğinde yapılandırılmış varsayılan akış profilini kullanır. Belirli bir
yapılandırılmış profili yenilemek için `--feed-profile <name>`, açıkça belirtilen barındırılmış
bir akış URL'sini yenilemek için `--feed-url <url>`, eşleşen bir yük sağlama toplamı
(`sha256:<hex>` veya 64 karakterlik yalın bir onaltılık özet) zorunlu kılmak için
`--expected-sha256 <sha256>` ve makine tarafından okunabilir çıktı için `--json`
kullanın. Açıkça belirtilen barındırılmış akış URL'leri kimlik bilgileri,
sorgu dizeleri veya parçalar içermemelidir. Sabitlenmemiş yenilemeler, komutu
başarısız kılmadan barındırılmış bir anlık görüntü veya paketle gelen geri dönüş sonucu
bildirebilir. Sabitlenmiş yenilemeler, yeni bir barındırılmış yükü kabul etmedikçe
başarısız olur; başarılı barındırılmış yenilemeler ise OpenClaw doğrulanmış anlık
görüntüyü kalıcı hâle getiremezse başarısız olur.

Yerleşik `clawhub-public` profili, `clawhub-official` yük kimliğini
bekler. ClawHub bu anahtarı oluşturup devrettikten sonra OpenClaw, ClawHub'ın
üretim ortak anahtarını paketine dahil edecektir. O zamana kadar yerleşik profil,
imzalı akıştan kurulum yetkisi vermez. Ortak anahtarlar, akış ana makinesindeki
bir anahtar uç noktasından değil, güvenilir bir sürüm veya operatör kanalından gelmelidir.

OpenClaw, DSSE zarfını doğrular ve bir profil `feedId` bildirdiğinde
kodu çözülmüş yük kimliğinin bununla eşleşmesini zorunlu kılar. Yerleşik
`clawhub-public` profili her zaman kendi kimliğini bildirerek başka bir
akışa ait geçerli bir belgenin bu profil üzerinden yeniden oynatılmasını önler.

Aşamalı kullanıma sunma sırasında, `feedId` belirtmeyen mevcut özel imzalı
profiller, yük kimliği bağlaması olmadan imza doğrulamasını korur. Yeni özel
profiller `feedId` bildirmelidir. Akış profili yapılandırma yüzeyi,
Control UI'ın ihtiyaç duyduğu sunum meta verileriyle birlikte ayrıca kullanıma
sunulmaktadır; bunun Doctor tanılaması operatörden eksik kimliği sağlamasını istemeli
ve akış URL'sinden bir kimlik çıkarsamamalıdır. Bu güven bağlaması, kullanımdan kaldırılan
kök `marketplaces` anahtarını geri getirmez.

## İlgili

- [Plugin oluşturma](/tr/plugins/building-plugins)
- [CLI başvurusu](/tr/cli)
- [ClawHub](/clawhub)
