---
read_when:
    - Bir kaynak kod deposu çalışma kopyasını güvenli bir şekilde güncellemek istiyorsunuz
    - '`openclaw update` çıktısında veya seçeneklerinde hata ayıklıyorsunuz'
    - '`--update` kısaltma davranışını anlamanız gerekir'
summary: '`openclaw update` için CLI başvurusu (nispeten güvenli kaynak güncellemesi + Gateway''in otomatik yeniden başlatılması)'
title: Güncelleme
x-i18n:
    generated_at: "2026-07-26T23:17:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b46696f6b9cba5c318f870bcb6c5ea8e0652940968da2ad85e86709fe4c11146
    source_path: cli/update.md
    workflow: 16
---

# `openclaw update`

OpenClaw'u güncelleyin ve stable/extended-stable/beta/dev kanalları arasında geçiş yapın.

**npm/pnpm/bun** aracılığıyla yüklediyseniz (genel yükleme, git meta verisi yok),
güncellemeler [Güncelleme](/tr/install/updating) bölümünde açıklanan
paket yöneticisi akışı üzerinden gerçekleştirilir.

## Kullanım

```bash
openclaw update
openclaw update status
openclaw update repair
openclaw update wizard
openclaw update --channel extended-stable
openclaw update --channel beta
openclaw update --channel dev
openclaw update --tag beta
openclaw update --tag main
openclaw update --dry-run
openclaw update --no-restart
openclaw update --yes
openclaw update --acknowledge-clawhub-risk
openclaw update --json
openclaw --update
```

`openclaw --update`, `openclaw update` olarak yeniden yazılır (kabuklar ve
başlatıcı betikleri için kullanışlıdır).

## Seçenekler

| Bayrak                                             | Açıklama                                                                                                                                                                                                                                                                                                                                  |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--no-restart`                                   | Başarılı bir güncellemeden sonra Gateway hizmetinin yeniden başlatılmasını atlayın. Yeniden başlatma yapan paket yöneticisi güncellemeleri, komut başarıyla tamamlanmadan önce yeniden başlatılan hizmetin beklenen sürümü bildirdiğini doğrular.                                                                                                                                                |
| `--channel <stable\|extended-stable\|beta\|dev>` | Güncelleme kanalını ayarlayın ve çekirdek güncellemesi başarıyla tamamlandıktan sonra kalıcı hâle getirin. Extended-stable yalnızca paketler içindir.                                                                                                                                                                                                                                            |
| `--tag <dist-tag\|version\|spec>`                | Yalnızca bu güncelleme için paket hedefini geçersiz kılın. Doğrulanmış kesin hedefi zorunlu olan etkin bir `extended-stable` kanalıyla birlikte kullanılamaz. Diğer paket yüklemelerinde `main`, `github:openclaw/openclaw#main` ile eşleşir; GitHub/git kaynak belirtimleri, aşamalı genel npm yüklemesinden önce geçici bir tarball olarak paketlenir. |
| `--dry-run`                                      | Yapılandırmayı yazmadan, yükleme yapmadan, plugin'leri eşitlemeden veya yeniden başlatmadan planlanan işlemleri (kanal/etiket/hedef/yeniden başlatma akışı) önizleyin.                                                                                                                                                                                                                |
| `--json`                                         | Makine tarafından okunabilir `UpdateRunResult` JSON çıktısı verin. Yönetilen bir plugin'in onarılması gerektiğinde `postUpdate.plugins.warnings`, beta kanalı plugin geri dönüş ayrıntıları ve güncelleme sonrası eşitleme sırasında npm plugin yapıtı sapması algılandığında `postUpdate.plugins.integrityDrifts` içerir.                                                                 |
| `--timeout <seconds>`                            | Adım başına zaman aşımı. Varsayılan: `1800`.                                                                                                                                                                                                                                                                                                            |
| `--yes`                                          | Onay istemlerini atlayın (örneğin sürüm düşürme onayı).                                                                                                                                                                                                                                                                              |
| `--acknowledge-clawhub-risk`                     | Güncelleme sonrası plugin eşitlemesinin, etkileşimli bir istem olmadan topluluk ClawHub güven uyarılarını geçerek devam etmesine izin verin. Bu seçenek olmadan, OpenClaw istem gösteremediğinde riskli topluluk sürümleri atlanır ve değiştirilmeden bırakılır. Resmî ClawHub paketleri ve paketle birlikte sunulan plugin kaynakları bu istemi atlar.                                                     |

`--verbose` bayrağı yoktur. Planlanan işlemleri önizlemek için `--dry-run`,
makine tarafından okunabilir sonuçlar için `--json` ve yalnızca
kanal/kullanılabilirlik için `openclaw update status --json` kullanın. Gateway konsol ayrıntı düzeyi
(`--verbose`) ve dosya günlük düzeyi (`logging.level: "debug"`/`"trace"`)
birbirinden bağımsız ayarlardır; bkz. [Gateway günlük kaydı](/tr/gateway/logging).

<Note>
Nix modunda (`OPENCLAW_NIX_MODE=1`), değişiklik yapan `openclaw update` çalıştırmaları devre dışıdır. Bunun yerine bu yüklemenin Nix kaynağını veya flake girdisini güncelleyin; nix-openclaw için önce ajan yaklaşımını kullanan [Hızlı Başlangıç](https://github.com/openclaw/nix-openclaw#quick-start) kılavuzunu kullanın. `openclaw update status` ve `openclaw update --dry-run` salt okunur kalır.
</Note>

<Warning>
Eski sürümler yapılandırmayı bozabileceğinden sürüm düşürme işlemleri onay gerektirir.
Yükleme oturumları zaten SQLite'a taşıdıysa, dosya destekli eski bir sürümü
başlatmadan önce arşivlenmiş eski transkript yapıtlarını geri yükleyin. Bkz.
[Doctor: Oturum SQLite geçişinden sonra sürüm düşürme](/tr/cli/doctor#downgrading-after-session-sqlite-migration).
</Warning>

## `update status`

Etkin güncelleme kanalını, git etiketini/dalını/SHA'sını (yalnızca kaynak
çalışma kopyalarında) ve güncelleme kullanılabilirliğini gösterin.

```bash
openclaw update status
openclaw update status --json
openclaw update status --timeout 10
```

| Bayrak                  | Varsayılan | Açıklama                         |
| --------------------- | ------- | ----------------------------------- |
| `--json`              | `false` | Makine tarafından okunabilir durum JSON'u çıktısı verin. |
| `--timeout <seconds>` | `3`     | Denetimler için zaman aşımı.                 |

Extended-stable paket yüklemelerinde durum, ön plan güncellemesiyle aynı genel
seçiciyi ve kesin paket doğrulamasını gerçekleştirir. Yüklü sürüm daha yeniyse
`ahead of extended-stable` bildirebilir. JSON hataları
`registry.reason` (`selector_missing`, `selector_query_failed`,
`exact_package_mismatch` veya `unsupported_git_channel`) içerir.

## `update repair`

Çekirdek paket zaten değiştirildikten ancak sonraki onarım çalışması düzgün
tamamlanmadıktan sonra güncelleme sonlandırmasını yeniden çalıştırın.
`openclaw update` yeni çekirdek paketi yüklediği hâlde çekirdek sonrası plugin
eşitlemesi, yönetilen npm plugin meta verileri, kayıt defteri yenilemesi veya
doctor onarımı yakınsamadığında desteklenen kurtarma yolu budur.

```bash
openclaw update repair
openclaw update repair --channel beta
openclaw update repair --acknowledge-clawhub-risk
openclaw update repair --json
```

| Bayrak                                             | Açıklama                                                                                                                                                                                                                                                         |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--channel <stable\|extended-stable\|beta\|dev>` | Onarımdan önce çekirdek güncelleme kanalını kalıcı hâle getirin. Extended-stable için, yalın/varsayılan veya `latest` amacını izleyen uygun resmî npm plugin'leri, yüklü kesin çekirdek sürümünü hedefler. Extended-stable onarımı, yapılandırmayı değiştirmeden Git çalışma kopyalarında reddedilir. |
| `--json`                                         | Makine tarafından okunabilir sonlandırma JSON'u çıktısı verin.                                                                                                                                                                                                                           |
| `--timeout <seconds>`                            | Onarım adımları için zaman aşımı. Varsayılan: `1800`.                                                                                                                                                                                                                           |
| `--yes`                                          | Onay istemlerini atlayın.                                                                                                                                                                                                                                          |
| `--acknowledge-clawhub-risk`                     | `openclaw update` üzerindekiyle aynı davranış.                                                                                                                                                                                                                              |
| `--no-restart`                                   | Eşdeğerlik için kabul edilir; onarım Gateway'i hiçbir zaman yeniden başlatmaz.                                                                                                                                                                                                             |

`update repair`, `openclaw doctor --fix` işlemini çalıştırır, onarılan
yapılandırmayı ve yükleme kayıtlarını yeniden yükler, izlenen plugin'leri etkin
güncelleme kanalı için eşitler, yönetilen npm plugin yüklemelerini günceller,
eksik yapılandırılmış plugin yüklerini onarır, plugin kayıt defterini yeniler
ve yakınsanmış yükleme kaydı meta verilerini yazar. Yeni bir çekirdek paket
yüklemez ve Gateway'i yeniden başlatmaz.

## `update wizard`

Bir güncelleme kanalı seçmek ve ardından Gateway'in yeniden başlatılıp
başlatılmayacağını onaylamak için etkileşimli akış (varsayılan olarak yeniden
başlatılır). Git çalışma kopyası olmadan `dev` seçildiğinde bir
tane oluşturulması önerilir.

| Bayrak                  | Varsayılan | Açıklama                   |
| --------------------- | ------- | ----------------------------- |
| `--timeout <seconds>` | `1800`  | Her güncelleme adımı için zaman aşımı. |

## Ne yapar?

Kanallar arasında açıkça geçiş yapılması (`--channel ...`), yükleme yöntemini
de uyumlu tutar:

- `dev` -> bir git çalışma kopyasının bulunmasını sağlar (varsayılan `~/openclaw` veya
  `OPENCLAW_HOME` ayarlandığında `$OPENCLAW_HOME/openclaw`; `OPENCLAW_GIT_DIR` ile
  geçersiz kılın), bunu günceller ve genel CLI'ı bu çalışma kopyasından
  yükler.
- `stable` -> `latest` kullanarak npm'den yükler.
- `extended-stable` -> genel npm `extended-stable` seçicisini çözümler,
  seçilen kesin paketi doğrular ve tam olarak bu sürümü yükler. Başka bir
  seçiciye geri dönmez ve Git çalışma kopyalarında reddedilir.
- `beta` -> npm dist-tag `beta` değerini tercih eder; beta eksikse
  veya mevcut kararlı sürümden daha eskiyse `latest` değerine geri döner.

### Yeniden başlatma devri

Gateway çekirdek otomatik güncelleyicisi (yapılandırma aracılığıyla
etkinleştirildiğinde), CLI güncelleme yolunu canlı Gateway istek işleyicisinin
dışında başlatır. Denetim düzlemi `update.run` paket yöneticisi
güncellemeleri ve denetlenen git çalışma kopyası güncellemeleri, canlı Gateway
işlemi içinde paket ağacını değiştirmek veya `dist/` öğesini yeniden
oluşturmak yerine aynı yönetilen hizmet devrini kullanır: Gateway ayrılmış bir
yardımcı başlatıp çıkar ve bu yardımcı `openclaw update --yes --json` işlemini Gateway işlem
ağacının dışından çalıştırır. Devir kullanılamıyorsa `update.run`, elle
çalıştırılacak güvenli kabuk komutunu içeren yapılandırılmış bir yanıt döndürür.

Saklanan extended-stable seçimler, `update.checkOnStart` etkinleştirildiğinde başlangıçta ve 24 saatlik aralıklarla salt okunur güncelleme
ipuçları alır. Bu kontroller hiçbir zaman güncelleme uygulamaz,
devir başlatmaz, Gateway'i yeniden başlatmaz, kararlı sürüm gecikmesi/jitter'ı kullanmaz veya beta
yoklama sıklığını kullanmaz. Açık ön plan güncellemeleri, saklanan
`update.channel: "extended-stable"` ile yalın ön plan güncellemeleri, isteğe bağlı durum ve bunların yönetilen
Gateway devri desteklenmeye devam eder.

Yerel bir yönetilen Gateway hizmeti kuruluysa ve yeniden başlatma etkinse,
paket yöneticisi ve git checkout güncellemeleri, paket ağacını değiştirmeden veya
checkout/derleme çıktısını değiştirmeden önce çalışan hizmeti durdurur. Güncelleyici
ardından hizmet meta verilerini yeniler, hizmeti yeniden başlatır ve
`Gateway: restarted and verified.` bildirmeden önce yeniden başlatılan Gateway'i doğrular.
Paket yöneticisi güncellemeleri ayrıca yeniden başlatılan Gateway'in beklenen
paket sürümünü bildirdiğini doğrular; git checkout güncellemeleri ise yeniden derlemeden sonra
gateway sağlığını ve hizmetin hazır olmasını doğrular.

Paket yöneticisi güncellemeleri normalde yönetilen hizmette kayıtlı Node ikilisini
kullanmaya devam eder. Bu Node hedef sürümü çalıştıramıyorsa ancak mevcut
CLI Node çalıştırabiliyorsa ve hizmetin güncellenen pakete ait olduğu
kanıtlanmışsa, yeniden başlatma etkin bir güncelleme sonlandırma için mevcut Node'u kullanır ve
hizmet meta verilerini bu çalışma zamanına göre yeniden yazar. `--no-restart` hizmet
meta verilerini onaramaz; bu nedenle aynı çalışma zamanı uyumsuzluğu paket değiştirilmeden önce işlemi durdurur.

macOS'ta güncelleme sonrası kontrol ayrıca LaunchAgent'ın etkin profil için
yüklü/çalışır durumda olduğunu ve yapılandırılmış geri döngü portunun
sağlıklı olduğunu doğrular. Plist kurulu ancak launchd tarafından denetlenmiyorsa OpenClaw,
LaunchAgent'ı otomatik olarak yeniden önyükler ve sağlık/sürüm/
kanal hazır olma kontrollerini yeniden çalıştırır (yeni bir önyükleme `RunAtLoad` işini doğrudan yükler,
bu nedenle kurtarma yeni oluşturulan Gateway'i hemen `kickstart -k` etmez). Gateway
yine de sağlıklı duruma gelmezse komut sıfır dışı kodla çıkar ve
yeniden başlatma günlük yoluyla birlikte yeniden başlatma, yeniden kurma ve paket geri alma
talimatlarını yazdırır.

Yeniden başlatma çalıştırılamazsa komut, manuel `openclaw gateway restart` ipucuyla birlikte
`Gateway: restart skipped (...)` veya `Gateway: restart failed: ...` yazdırır.
`--no-restart` ile paket değiştirme veya git yeniden derleme yine çalışır ancak
yönetilen hizmet durdurulmaz veya yeniden başlatılmaz; dolayısıyla çalışan Gateway, siz
manuel olarak yeniden başlatana kadar eski kodu kullanmaya devam eder.

### Kontrol düzlemi yanıt biçimi

`update.run`, bir paket yöneticisi kurulumu veya denetlenen git checkout üzerinde
Gateway kontrol düzlemi üzerinden çalıştığında işleyici, Gateway çıktıktan sonra
devam eden CLI güncellemesinden devir başlatmayı ayrı olarak bildirir:

- `ok: true`, `result.status: "skipped"`,
  `result.reason: "managed-service-handoff-started"` ve
  `handoff.status: "started"`: Gateway, yönetilen hizmet devrini oluşturdu
  ve ayrılmış yardımcının canlı hizmet işlemi dışında
  `openclaw update --yes --json` çalıştırabilmesi için kendi yeniden başlatmasını zamanladı.
- `ok: false`, `result.reason: "managed-service-handoff-unavailable"` ve
  `handoff.status: "unavailable"`: OpenClaw güvenli bir devir için denetleyici
  hizmet sınırı ve kalıcı hizmet kimliği bulamadı (örneğin
  systemd devri yalnızca ortam systemd işlem işaretçilerini değil,
  `OPENCLAW_SYSTEMD_UNIT` birim kimliğini gerektirir). Yanıt,
  Gateway dışından çalıştırılacak kabuk komutu olan `handoff.command` değerini içerir.
- `ok: false`, `result.reason: "managed-service-handoff-failed"`: Gateway
  devri oluşturmayı denedi ancak ayrılmış yardımcıyı başlatamadı.

`sentinel` yükü Gateway çıkmadan önce yazılır ve CLI
devri, yönetilen hizmetin yeniden başlatma sağlık kontrolleri tamamlandıktan sonra aynı yeniden başlatma
nöbetçisini günceller. Devir sırasında nöbetçi, başarı devamı olmadan
`stats.reason: "restart-health-pending"` taşıyabilir;
yeniden başlatılan Gateway bunu yoklar ve yalnızca CLI hizmet sağlığını
doğrulayıp nöbetçiyi nihai `ok` sonucuyla yeniden yazdıktan sonra devamı tetikler.
`openclaw status` ve `openclaw status --all`, bu nöbetçi beklemede veya başarısızken
bir `Update restart` satırı gösterir; `update.status` ise yenileyip
en son nöbetçiyi döndürür.

## Git checkout akışı

### Kanal seçimi

- `stable`: en son beta olmayan etiketi checkout yapar, ardından derler ve doctor'ı çalıştırır.
- `beta`: en son `-beta` etiketini tercih eder; beta yoksa veya daha eskiyse
  en son kararlı etikete geri döner.
- `dev`: `main` checkout yapar, ardından getirir ve rebase eder.
- `extended-stable`: Git checkout'ları için desteklenmez; checkout üzerinde
  değişiklik yapılmaz.

### Güncelleme adımları

<Steps>
  <Step title="Temiz çalışma ağacını doğrula">
    Kaydedilmemiş değişiklik bulunmamasını gerektirir.
  </Step>
  <Step title="Kanalı değiştir">
    Seçilen kanala (etiket veya dal) geçer.
  </Step>
  <Step title="Upstream'i getir">
    Yalnızca geliştirme.
  </Step>
  <Step title="Ön kontrol derlemesi (yalnızca geliştirme)">
    TypeScript derlemesini geçici bir çalışma ağacında çalıştırır. Uç başarısız olursa derlenebilen en yeni commit'i bulmak için en fazla 10 commit geriye gider. Bu ön kontrol sırasında lint'i de çalıştırmak için `OPENCLAW_UPDATE_PREFLIGHT_LINT=1` ayarlayın; kullanıcı güncelleme ana makineleri genellikle CI çalıştırıcılarından daha küçük olduğundan lint, kısıtlı seri modda çalışır.
  </Step>
  <Step title="Rebase et">
    Seçilen commit üzerine rebase eder (yalnızca geliştirme).
  </Step>
  <Step title="Bağımlılıkları kur">
    Repo paket yöneticisini kullanır. pnpm checkout'larında güncelleyici, bir pnpm çalışma alanı içinde `npm run build` çalıştırmak yerine gerektiğinde `pnpm` önyüklemesi yapar (önce `corepack`, ardından geçici bir `npm install pnpm@11` geri dönüşü aracılığıyla). pnpm önyüklemesi yine de başarısız olursa güncelleyici, checkout içinde `npm run build` denemek yerine paket yöneticisine özgü bir hatayla erkenden durur.
  </Step>
  <Step title="Control UI'ı derle">
    Gateway'i ve Control UI'ı derler.
  </Step>
  <Step title="Doctor'ı çalıştır">
    `openclaw doctor`, son güvenli güncelleme kontrolü olarak çalışır.
  </Step>
  <Step title="Plugin'leri eşitle">
    Plugin'leri etkin kanalla eşitler. Geliştirme, paketlenmiş Plugin'leri; kararlı ve beta ise npm'i kullanır. İzlenen Plugin kurulumlarını günceller.
  </Step>
</Steps>

### Plugin eşitleme ayrıntıları

Beta kanalında, varsayılan/en son hattı izleyen takip edilen npm ve ClawHub Plugin
kurulumları önce bir Plugin `@beta` sürümünü dener. Plugin'in beta sürümü yoksa
OpenClaw, kayıtlı varsayılan/en son tanıma geri döner ve
bir uyarı bildirir. OpenClaw, npm Plugin'lerinde beta
paketi mevcut olup kurulum doğrulamasında başarısız olduğunda da geri döner. Bu geri dönüş uyarıları
çekirdek güncellemesinin başarısız olmasına neden olmaz. Kesin sürümler ve açık etiketler hiçbir zaman yeniden yazılmaz.

<Warning>
Tam olarak sabitlenmiş bir npm Plugin güncellemesi, bütünlüğü saklanan kurulum kaydından farklı bir yapıta çözümlenirse `openclaw update`, bu Plugin yapıtı güncellemesini kurmak yerine iptal eder. Yeni yapıta güvendiğinizi doğruladıktan sonra Plugin'i açıkça yeniden kurun veya güncelleyin.
</Warning>

<Note>
Yönetilen bir Plugin ile sınırlı olan ve eşitleme yolunun etrafından dolaşabildiği güncelleme sonrası Plugin eşitleme hataları (örneğin temel olmayan bir Plugin için erişilemeyen bir npm kayıt defteri), çekirdek güncellemesi başarıyla tamamlandıktan sonra uyarı olarak bildirilir. JSON sonucu, üst düzey güncelleme `status: "ok"` değerini korur ve `openclaw update repair` ile `openclaw plugins inspect <id> --runtime --json` yönlendirmesini içeren `postUpdate.plugins.status: "warning"` değerini bildirir. Beklenmeyen güncelleyici veya eşitleme istisnaları yine de güncelleme sonucunu başarısız kılar. Plugin kurulum veya güncelleme hatasını düzeltin, ardından `openclaw update repair` komutunu yeniden çalıştırın. Başarısız bir güncelleme yönetilen bir Plugin'i kullanılamaz durumda bıraktığında OpenClaw, operatör tarafından yazılmış `plugins.allow` veya `plugins.deny` politikasını değiştirmeden çalışma zamanı girdisini devre dışı bırakır ve etkin yuvaları sıfırlar.

Plugin başına eşitleme adımından sonra `openclaw update`, gateway yeniden başlatılmadan önce zorunlu bir **çekirdek sonrası yakınsama** geçişi çalıştırır: eksik yapılandırılmış Plugin yüklerini onarır, diskteki her _etkin_ izlenen kurulum kaydını doğrular ve `package.json` değerinin ayrıştırılabilir olduğunu (ve açıkça bildirilmiş tüm `main` değerlerinin mevcut olduğunu) statik olarak doğrular. Bu geçişten kaynaklanan hatalar ve geçersiz bir yapılandırma anlık görüntüsü `postUpdate.plugins.status: "error"` döndürür ve üst düzey güncelleme `status` değerini `"error"` olarak değiştirir; böylece `openclaw update` sıfır dışı kodla çıkar ve gateway doğrulanmamış bir Plugin kümesiyle _yeniden başlatılmaz_. Hata, `openclaw update repair` ve `openclaw plugins inspect <id> --runtime --json` konumlarını gösteren yapılandırılmış `postUpdate.plugins.warnings[].guidance` satırlarını içerir. Devre dışı bırakılmış Plugin girdileri ve güvenilir kaynak bağlantılı resmî eşitleme hedefleri olmayan kayıtlar burada atlanır (eksik yük kontrolünün kullandığı `skipDisabledPlugins` politikası yansıtılır); böylece eski bir devre dışı Plugin kaydı, başka bakımdan geçerli bir güncellemeyi engelleyemez.

Güncellenen Gateway başladığında Plugin yükleme yalnızca doğrulama yapar: başlangıç, paket yöneticilerini çalıştırmaz veya bağımlılık ağaçlarını değiştirmez. Paket yöneticisi `update.run` yeniden başlatmaları CLI yönetilen hizmet yoluna devredilir; böylece paket değişimi eski Gateway işleminin dışında gerçekleşir ve güncellemenin tamamlanmış olarak bildirilip bildirilemeyeceğine hizmet sağlık kontrolleri karar verir.
</Note>

Bir extended-stable çekirdek güncellemesi başarıyla tamamlandıktan sonra çekirdek sonrası Plugin bütünlüğü ve
yakınsama, uygun resmî npm Plugin'lerini tam kurulu çekirdek
sürümünde hedefler. Varsayılan/`latest` amacı için OpenClaw, Plugin
`@extended-stable` değerini sorgulamaz veya npm `latest` değerine geri dönmez; paket sürümünü
kurulu çekirdekten türetir. Açık sürüm sabitlemeleri, açık `latest` olmayan etiketler,
üçüncü taraf paketler ve npm dışı kaynaklar mevcut amaçlarını korur.

Paket yöneticisi kurulumları için `openclaw update`, paket yöneticisini
çağırmadan önce hedef paket sürümünü çözümler. npm global kurulumları aşamalı
kurulum kullanır: OpenClaw yeni paketi geçici bir npm önekine kurar,
aday paketin `preinstall` sırasında ana makine Node sürümünü doğrulamasına izin verir
ve buradaki paketlenmiş `dist` envanterini doğrular. Paketlenmiş bir tamamlama koruması
`preinstall` başarılı olana kadar bu envanterin dışında kalır; böylece yaşam döngüsü betiklerini
atlayan paket yöneticileri de etkinleştirmeden önce durur. npm 12 ve daha yeni sürümlerde
güncelleyici yalnızca aday OpenClaw yaşam döngüsünü onaylar; geçişli
bağımlılık betikleri engellenmiş kalır. OpenClaw daha sonra temiz paket ağacını
gerçek global öneke geçirir. Doğrulama başarısız olursa güncelleme sonrası doctor, Plugin
eşitleme ve yeniden başlatma işlemleri şüpheli ağaçtan çalıştırılmaz. Kurulu
sürüm hedefle zaten eşleşse bile komut global paket kurulumunu
yeniler, ardından Plugin eşitlemeyi, çekirdek komut tamamlama yenilemesini
ve yeniden başlatma işlemlerini çalıştırır. Bu, paketlenmiş yan bileşenleri ve kanalın sahip olduğu
Plugin kayıtlarını kurulu OpenClaw derlemesiyle uyumlu tutarken tam
Plugin komutu tamamlama yeniden derlemelerini açık
`openclaw completion --write-state` çalıştırmalarına bırakır.

## İlgili

- `openclaw doctor` (git checkout'larında önce güncellemeyi çalıştırmayı önerir)
- [Geliştirme kanalları](/tr/install/development-channels)
- [Güncelleme](/tr/install/updating)
- [CLI başvurusu](/tr/cli)
