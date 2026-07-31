---
read_when:
    - OpenClaw güncelleme, doctor, paket kabulü veya plugin yükleme davranışını değiştirme
    - Bir sürüm adayını hazırlama veya onaylama
    - Paket güncellemesi, plugin bağımlılığı temizliği veya plugin yükleme gerilemelerinde hata ayıklama
sidebarTitle: Update and plugin tests
summary: OpenClaw güncelleme yollarını, paket geçişlerini ve plugin yükleme/güncelleme davranışını nasıl doğrular
title: 'Test: güncellemeler ve pluginler'
x-i18n:
    generated_at: "2026-07-26T23:22:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 96a11fe42472f758d4fd1cc568486e301f7460982fdb547cab8b39de04a8dabe
    source_path: help/testing-updates-plugins.md
    workflow: 16
---

Güncelleme ve plugin doğrulaması için kontrol listesi: kurulabilir paketin gerçek kullanıcı durumunu
güncelleyebildiğini, eski kalmış legacy durumu `doctor` aracılığıyla onarabildiğini ve yine de
desteklenen her kaynaktan plugin kurabildiğini, yükleyebildiğini, güncelleyebildiğini ve kaldırabildiğini kanıtlayın.

Daha kapsamlı test çalıştırıcısı haritası için [Test](/tr/help/testing) bölümüne bakın. Canlı sağlayıcı
anahtarları ve ağa erişen paketler için [Canlı test](/tr/help/testing-live) bölümüne bakın.

## Neleri koruyoruz

- Paket tarball'ı eksiksizdir, geçerli bir `dist/postinstall-inventory.json` içerir
  ve paketlenmemiş depo dosyalarına bağımlı değildir.
- Kullanıcı; yapılandırmasını, ajanlarını, oturumlarını, çalışma alanlarını, plugin izin listelerini veya
  kanal yapılandırmasını kaybetmeden yayımlanmış eski bir paketten aday pakete
  geçebilir.
- `openclaw doctor --fix --non-interactive`, legacy temizleme ve onarım
  yollarının sahibidir. Başlatma, eski kalmış plugin durumu için gizli uyumluluk
  geçişleri oluşturmamalıdır.
- Plugin kurulumları yerel dizinlerden, git depolarından, npm paketlerinden ve
  ClawHub kayıt yolu üzerinden çalışır.
- Plugin npm bağımlılıkları plugin başına tek bir yönetilen npm projesine kurulur,
  güvenilmeden önce taranır ve plugin kaldırılırken `npm uninstall` aracılığıyla
  kaldırılır; böylece yukarı taşınmış bağımlılıklar geride kalmaz.
- Hiçbir şey değişmediğinde plugin güncellemesi işlem yapmaz: kurulum kayıtları, çözümlenen
  kaynak, kurulu bağımlılık düzeni ve etkinlik durumu değişmeden kalır.

## Geliştirme sırasında yerel kanıt

Dar kapsamla başlayın:

```bash
pnpm changed:lanes --json
pnpm check:changed
pnpm test:changed
```

Plugin kurulumu, kaldırılması, bağımlılıkları veya paket envanteri değişiklikleri için,
düzenlenen bağlantı noktasını kapsayan odaklı testleri de çalıştırın:

```bash
pnpm test src/plugins/uninstall.test.ts src/infra/package-dist-inventory.test.ts test/scripts/package-acceptance-workflow.test.ts
```

Herhangi bir paket Docker hattı bir tarball kullanmadan önce paket yapıtını kanıtlayın:

```bash
pnpm release:check
```

`release:check`; yapılandırma/belgeler/API sapması denetimlerini (yapılandırma şeması, yapılandırma belgeleri
temel çizgisi, plugin SDK API sözleşmesi manifesti ve dışa aktarımları, plugin sürümleri/envanteri)
çalıştırır, paket dağıtım envanterini yazar, `npm pack --dry-run` komutunu çalıştırır, yasaklanmış
paketlenmiş dosyaları reddeder, tarball'ı geçici bir ön eke kurar, postinstall işlemini çalıştırır ve
paketle gelen kanal giriş noktalarında smoke testi yapar.

## Docker hatları

Docker hatları ürün düzeyindeki kanıttır. Linux kapsayıcıları içinde gerçek bir
paketi kurar veya günceller ve CLI komutları, Gateway başlatma, HTTP yoklamaları,
RPC durumu ve dosya sistemi durumu üzerinden davranışı doğrular.

Yineleme sırasında odaklı hatları kullanın:

```bash
pnpm test:docker:plugins
pnpm test:docker:plugin-lifecycle-matrix
pnpm test:docker:plugin-update
pnpm test:docker:upgrade-survivor
pnpm test:docker:published-upgrade-survivor
pnpm test:docker:update-restart-auth
pnpm test:docker:update-migration
```

Önemli hatlar:

- `test:docker:plugins`; plugin kurulum smoke testini, yerel klasör kurulumlarını,
  yerel klasör güncellemesinin atlanma davranışını, önceden kurulmuş bağımlılıkları olan
  yerel klasörleri, `file:` paket kurulumlarını, CLI yürütmeli git kurulumlarını, git
  hareketli referans güncellemelerini, yukarı taşınmış geçişli bağımlılıkları olan npm kayıt
  defteri kurulumlarını, işlem yapmayan npm güncellemelerini, hatalı biçimlendirilmiş npm paket
  meta verilerinin reddedilmesini, yerel ClawHub fikstürü kurulumlarını ve işlem yapmayan
  güncellemeleri, pazar yeri güncelleme davranışını ve Claude paketini etkinleştirme/incelemeyi
  kapsar. ClawHub bloğunu hermetik/çevrimdışı tutmak için `OPENCLAW_PLUGINS_E2E_CLAWHUB=0` değerini ayarlayın.
- `test:docker:plugin-lifecycle-matrix`, aday paketi boş bir
  kapsayıcıya kurar; bir npm pluginini kurma, inceleme, devre dışı bırakma, etkinleştirme,
  açık yükseltme, açık düşürme ve plugin kodunu sildikten sonra kaldırma adımlarından
  geçirir. Her aşama için RSS ve CPU metriklerini günlüğe kaydeder.
- `test:docker:plugin-update`, değişmemiş kurulu bir pluginin
  `openclaw plugins update` sırasında yeniden kurulmadığını veya kurulum meta verilerini
  kaybetmediğini doğrular.
- `test:docker:upgrade-survivor`, aday tarball'ı değiştirilmiş
  bir eski kullanıcı fikstürünün üzerine kurar, paket güncellemesini ve etkileşimsiz doctor işlemini
  çalıştırır, ardından bir geri döngü Gateway'i başlatır ve durumun korunduğunu denetler.
- `test:docker:published-upgrade-survivor`, önce yayımlanmış bir temel sürümü kurar,
  bunu yerleşik bir `openclaw config set` tarifi üzerinden yapılandırır, aday tarball'a
  günceller, doctor işlemini çalıştırır, legacy temizliğini denetler, Gateway'i başlatır ve
  `/healthz`, `/readyz` ile RPC durumunu yoklar.
- `test:docker:update-restart-auth`, aday paketi kurar, yönetilen
  token kimlik doğrulamalı bir Gateway başlatır, `openclaw update --yes --json` için
  çağıranın Gateway kimlik doğrulama ortam değişkenini kaldırır ve normal yoklamalardan önce
  aday güncelleme komutunun Gateway'i yeniden başlatmasını zorunlu kılar.
- `test:docker:update-migration`, temizleme ağırlıklı yayımlanmış güncelleme hattıdır.
  Yapılandırılmış Discord/Telegram tarzı bir kullanıcı durumundan başlar, yapılandırılmış plugin
  bağımlılıklarının oluşturulmasına fırsat vermek için temel sürümün doctor işlemini çalıştırır,
  yapılandırılmış paketli bir plugin için legacy plugin bağımlılığı kalıntıları ekler, aday
  tarball'a günceller ve güncelleme sonrası doctor işleminin legacy bağımlılık köklerini
  kaldırmasını zorunlu kılar.

Yararlı yayımlanmış yükseltme dayanıklılığı varyantları:

```bash
OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC=openclaw@2026.4.23 \
OPENCLAW_UPGRADE_SURVIVOR_SCENARIO=versioned-runtime-deps \
pnpm test:docker:published-upgrade-survivor

OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC=openclaw@latest \
OPENCLAW_UPGRADE_SURVIVOR_SCENARIO=bootstrap-persona \
pnpm test:docker:published-upgrade-survivor
```

Kullanılabilir senaryolar: `base`, `acpx-openclaw-tools-bridge`, `feishu-channel`,
`bootstrap-persona`, `channel-post-core-restore`, `plugin-deps-cleanup`,
`configured-plugin-installs`, `stale-source-plugin-shadow`, `tilde-log-path`
ve `versioned-runtime-deps`. Toplu çalıştırmalarda `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS=reported-issues`
(`far-reaching` diğer adıyla), yapılandırılmış plugin kurulum geçişi de dahil olmak üzere
tüm senaryolara genişler.

Tam güncelleme geçişi, Tam Sürüm CI'dan bilinçli olarak ayrıdır. Sürüm sorusu "2026.4.23'ten
itibaren yayımlanan her kararlı sürüm bu adaya güncellenebilir ve plugin bağımlılığı
kalıntılarını temizleyebilir mi?" olduğunda manuel `Update Migration` iş akışını kullanın:

```bash
gh workflow run update-migration.yml \
  --ref main \
  -f workflow_ref=main \
  -f package_ref=main \
  -f baselines=all-since-2026.4.23 \
  -f scenarios=plugin-deps-cleanup
```

## Paket Kabulü

Paket Kabulü, GitHub'a özgü paket geçididir. Bir aday paketi
`package-under-test` tarball'ına çözümler, sürümü ve SHA-256 değerini kaydeder, ardından
yeniden kullanılabilir Docker E2E hatlarını tam olarak bu tarball'a karşı çalıştırır. İş akışı test
düzeneği referansı paket kaynak referansından ayrıdır; böylece güncel test mantığı eski güvenilir
sürümleri doğrulayabilir.

Aday kaynaklar:

- `source=npm`: `openclaw@extended-stable`, `openclaw@beta`,
  `openclaw@latest` veya tam bir yayımlanmış sürümü doğrular.
- `source=ref`: seçili güncel
  test düzeneğiyle güvenilir bir dalı, etiketi veya commit'i paketler.
- `source=url`: zorunlu `package_sha256` ile herkese açık bir HTTPS tarball'ını doğrular.
  Bu yol; URL kimlik bilgilerini, varsayılan olmayan HTTPS portlarını, özel/dahili
  ana bilgisayar adlarını veya DNS/IP sonuçlarını, özel kullanımlı IP alanını ve güvenli olmayan yönlendirmeleri reddeder.
- `source=trusted-url`: zorunlu
  `package_sha256` ve `trusted_source_id` ile bir HTTPS tarball'ını, bakımcıların sahip olduğu
  `.github/package-trusted-sources.json` ilkesine göre doğrular. Bunu, `source=url` seçeneğini girdi düzeyinde
  özel erişime izin veren bir anahtarla zayıflatmak yerine kurumsal/özel yansılar için kullanın.
  Taşıyıcı kimlik doğrulaması, ilke tarafından yapılandırıldığında sabit
  `OPENCLAW_TRUSTED_PACKAGE_TOKEN` gizli bilgisini kullanır.
- `source=artifact`: başka bir Actions çalıştırmasının yüklediği tarball'ı yeniden kullanır.

Tam Sürüm Doğrulaması varsayılan olarak çözümlenen sürüm SHA'sından oluşturulan
`source=artifact` değerini kullanır. Yayım sonrası kanıt için
`package_acceptance_package_spec=openclaw@YYYY.M.PATCH` değerini geçirin; böylece aynı yükseltme matrisi
gönderilmiş npm paketini hedefler.

Sürüm denetimleri, Paket Kabulünü paket/güncelleme/yeniden başlatma/plugin kümesiyle çağırır:

```text
doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape
```

Sürüm bekletme testi etkinleştirildiğinde (`release_profile=stable` ve
`full` için zorunlu), ayrıca şunları geçirir:

```text
published_upgrade_survivor_baselines=last-stable-4 2026.4.23 2026.5.2 2026.4.15
published_upgrade_survivor_scenarios=reported-issues
telegram_mode=mock-openai
```

Bu, varsayılan sürüm paketi geçidini yayımlanmış her sürümde dolaştırmadan paket geçişini,
güncelleme kanalı değiştirmeyi, bozuk yönetilen plugin toleransını, eski kalmış plugin bağımlılığı
temizliğini, çevrimdışı plugin kapsamını, plugin güncelleme davranışını ve Telegram paket
kalite güvencesini aynı çözümlenmiş yapıt üzerinde tutar.

`last-stable-4`, npm'de yayımlanmış en son dört kararlı OpenClaw
sürümüne çözümlenir. Sürüm paketi kabulü, ilk plugin güncelleme uyumluluğu sınırı olarak
`2026.4.23` değerini, plugin mimarisi değişim sınırı olarak `2026.5.2`
değerini ve daha eski bir 2026.4.1x yayımlanmış güncelleme temel sürümü olarak
`2026.4.15` değerini sabitler; çözümleyici, zaten son dört sürüm arasında bulunan sabitleri
yinelenenlerden arındırır. Kapsamlı yayımlanmış güncelleme geçişi kapsamı için Tam Sürüm CI
yerine ayrı Güncelleme Geçişi iş akışında `all-since-2026.4.23` değerini kullanın.
Legacy tarih öncesi bağlantı noktasını da istediğiniz manuel geniş örnekleme için
`release-history` kullanılabilir durumda kalır.

Birden fazla yayımlanmış yükseltme dayanıklılığı temel sürümü seçildiğinde, yeniden kullanılabilir
Docker iş akışı her temel sürümü kendi hedefli çalıştırıcı işine böler. Her temel sürüm parçası
seçili senaryo kümesini çalıştırmaya devam eder; ancak günlükler ve yapıtlar temel sürüm başına
ayrı kalır ve duvar saati süresi tek bir büyük seri iş yerine en yavaş parçayla sınırlanır.

Sürüm öncesinde bir adayı doğrularken paket profilini manuel olarak çalıştırın:

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=package \
  -f published_upgrade_survivor_baselines="last-stable-4 2026.4.23 2026.5.2 2026.4.15" \
  -f published_upgrade_survivor_scenarios=reported-issues \
  -f telegram_mode=mock-openai
```

Yayımlanmış bir extended-stable canary için
`package_spec=openclaw@extended-stable` değerini ayarlayın. Paket Kabulü, Docker hatları çalışmadan önce bu
seçiciyi tam bir tarball'a çözümler.

Sürüm sorusu MCP kanallarını, cron/alt ajan temizliğini, OpenAI web aramasını veya OpenWebUI'ı
içerdiğinde `suite_profile=product` değerini kullanın. `suite_profile=full` değerini yalnızca
tam Docker sürüm yolu kapsamına ihtiyaç duyduğunuzda kullanın.

## Sürüm varsayılanı

Sürüm adayları için varsayılan kanıt yığını şöyledir:

1. Kaynak düzeyindeki regresyonlar için `pnpm check:changed` ve `pnpm test:changed`.
2. Paket yapıtı bütünlüğü için `pnpm release:check`.
3. Kurulum/güncelleme/yeniden başlatma/plugin sözleşmeleri için Paket Kabulü
   `package` profili veya sürüm denetiminin özel paket hatları.
4. İşletim sistemine özgü yükleyici, ilk katılım ve platform
   davranışı için işletim sistemleri arası sürüm denetimleri.
5. Canlı paketler yalnızca değiştirilen yüzey sağlayıcı veya barındırılan hizmet
   davranışına dokunduğunda çalıştırılır.

Bakımcı makinelerinde, açıkça yerel kanıt yürütülmediği sürece geniş geçitler ve Docker/paket
ürün kanıtı Testbox içinde çalıştırılmalıdır.

## Legacy uyumluluk

Uyumluluk esnekliği dar kapsamlı ve zaman sınırlıdır:

- `2026.4.25-beta.*` dahil olmak üzere `2026.4.25` sürümüne kadarki paketler,
  Paket Kabulünde daha önce gönderilmiş paket meta verisi eksiklerini tolere edebilir.
- Yayımlanmış `2026.4.26` paketi, daha önce gönderilmiş yerel derleme meta verisi
  damga dosyaları için uyarı verebilir.
- Daha sonraki paketler modern sözleşmeleri karşılamalıdır. Aynı eksikler
  uyarı verilmesi veya atlanması yerine başarısızlığa neden olur.

Bu eski şekiller için yeni başlatma geçişleri eklemeyin. Bir doctor onarımı ekleyin veya mevcut
onarımı genişletin; ardından yeniden başlatma güncelleme komutuna ait olduğunda bunu
`upgrade-survivor`, `published-upgrade-survivor` veya `update-restart-auth` ile kanıtlayın.

## Kapsam ekleme

Güncelleme veya plugin davranışını değiştirirken, doğru nedenle başarısız olabilecek
en alt katmana kapsam ekleyin:

- Saf yol veya meta veri mantığı: kaynağın yanında birim testi.
- Paket envanteri veya paketlenmiş dosya davranışı: `package-dist-inventory` ya da tarball
  denetleyici testi.
- CLI yükleme/güncelleme davranışı: Docker hattı doğrulaması veya fikstürü.
- Yayımlanmış sürüm geçişi davranışı: `published-upgrade-survivor` senaryosu.
- Güncellemenin yönettiği yeniden başlatma davranışı: `update-restart-auth`.
- Kayıt defteri/paket kaynağı davranışı: `test:docker:plugins` fikstürü veya ClawHub
  fikstür sunucusu.
- Bağımlılık yerleşimi veya temizleme davranışı: hem çalışma zamanı yürütmesini hem de
  dosya sistemi sınırını doğrulayın. npm bağımlılıkları, plugin'in
  yönetilen npm projesi içinde üst seviyeye taşınabilir; bu nedenle testler yalnızca plugin paketine özgü
  `node_modules` ağacını varsaymak yerine bu projenin tarandığını/temizlendiğini kanıtlamalıdır.

Yeni Docker fikstürlerini varsayılan olarak dış etkenlerden yalıtılmış tutun. Testin amacı canlı kayıt defteri
davranışı olmadığı sürece yerel fikstür kayıt defterleri ve sahte paketler kullanın.

## Hata triyajı

Artefakt kimliğiyle başlayın:

- Package Acceptance `resolve_package` özeti: kaynak, sürüm, SHA-256 ve
  artefakt adı.
- Docker artefaktları: `.artifacts/docker-tests/**/summary.json`,
  `failures.json`, hat günlükleri ve yeniden çalıştırma komutları.
- Yükseltmeden sağ kalanlar özeti: temel sürüm, aday sürüm, senaryo, aşama süreleri ve
  yapılandırma tarifi kapsamı dâhil olmak üzere `.artifacts/upgrade-survivor/summary.json`.

Tüm sürüm şemsiyesini yeniden çalıştırmak yerine, başarısız olan tam hattı aynı paket artefaktıyla
yeniden çalıştırmayı tercih edin.
