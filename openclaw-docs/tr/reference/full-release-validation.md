---
doc-schema-version: 1
read_when:
    - Tam Sürüm Doğrulamasını çalıştırma veya yeniden çalıştırma
    - Kararlı ve tam sürüm doğrulama profillerinin karşılaştırılması
    - Sürüm doğrulama aşaması hatalarında hata ayıklama
summary: Tam Sürüm Doğrulama aşamaları, alt iş akışları, sürüm profilleri, yeniden çalıştırma tanıtıcıları ve kanıtlar
title: Tam sürüm doğrulaması
x-i18n:
    generated_at: "2026-07-26T23:38:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf165d5515f4b9bb11d239382649d332d20bb8a32bd4492ae99092fb5ee2216
    source_path: reference/full-release-validation.md
    workflow: 16
---

`Full Release Validation`, sürüm ürün doğrulamasını kapsayan üst iş akışıdır. Çalışmaların çoğu,
başarısız olan bir ortamın tüm sürüm yeniden başlatılmadan tekrar çalıştırılabilmesi için
alt iş akışlarında gerçekleşir. Code SHA'yı sabitlemeden önce sürüm hazırlığını çalıştırın;
arka plan botu Control UI yerel ayar çıktısını henüz birleştirmediyse bu işlem çıktıyı
yeniler, ardından sürüm CI işlem hattında kullanılan aynı katı sıfır geri dönüş denetimini uygular.

Ürün açısından tamamlanmış, değişiklik günlüğü öncesi commit'i **Code SHA** olarak sabitleyin, ardından şunu çalıştırın:

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

`provider`, işletim sistemleri arası ilk katılım ve uçtan uca ajan turu için
`anthropic` veya `minimax` değerlerini de kabul eder. Yardımcı, alfa/beta
paket sürümlerinden `beta` profilini, aksi durumda `stable` profilini çıkarır.
Alternatif iş akışı girdilerini `-f key=value` ile iletin; `-f release_profile=full` seçeneğini yalnızca
geniş kapsamlı danışma taraması için kullanın.

Yardımcı, güvenilir tek bir `origin/main` iş akışı SHA'sına sabitlenmiş geçici bir
`release-ci/*` ref'i oluşturur, hedef SHA'yı yalnızca aday `ref` olarak iletir
ve doğrulamadan sonra geçici ref'i siler. Tetiklenen her alt iş akışı aynı iş akışı SHA'sını
bildirmelidir. Yeni bir çalıştırmayı zorlamak için
`-f reuse_evidence=false`, güncel `origin/main` üzerinden hâlâ erişilebilen
daha eski bir iş akışı commit'ini seçmek için `--workflow-sha <trusted-main-sha>`
iletin. İş akışının kendisi hiçbir zaman depo ref'leri oluşturmaz veya güncellemez.

## Extended-stable istisnası

Extended-stable yayımlama, hem iş akışının hem de hedefin standart dal olduğu
bir çalıştırma gerektirir:

```bash
gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

`pnpm ci:full-release` veya `release-ci/*` kullanmayın. Yayımlama; çalıştırmanın
dalını, baş/hedef SHA'sını, manifest `workflowRef` değerini, kimliğini ve deneme
numarasını standart dala ve sürüm commit'ine bağlar.

Ürün hatalarını backport edin; sabitlenmiş hedef araçları için davranışı koruyan
en küçük onarımı yapın; sağlayıcı, onay veya çalıştırıcı hatalarını kaynak değişikliği
olmadan yeniden deneyin. Her dal değişikliği tamamen yeni bir çalıştırma gerektirir.
Hedef eski olduğu için gerekli paket, yükleyici, güncelleme, kanal veya canlı davranışı
atlamayın.

Normal bir sürümde Code SHA yeşil olduğunda yalnızca
`CHANGELOG.md` dosyasını oluşturup commit edin. Bu yeni commit **Release SHA** olur.
Aynı yardımcıyı Release SHA için çalıştırın. Ürün kanıtı yalnızca GitHub, Release SHA'nın
Code SHA'nın soyundan geldiğini ve değişen yolların eksiksiz kümesinin tam olarak
`CHANGELOG.md` olduğunu kanıtladığında yeniden kullanılır; npm ön kontrolü ile
paket/yükleme kabulü yine de Release SHA üzerinde çalışır.

`release_profile=stable` ve `release_profile=full` her zaman kapsamlı
canlı/Docker dayanıklılık testini çalıştırır. Aynı dayanıklılık testi hatlarını
`beta` profiliyle dahil etmek için `run_release_soak=true` iletin.
Kararlı yayımlama, bu dayanıklılık testi ve engelleyici ürün performansı kanıtı
bulunmayan bir doğrulama manifestini reddeder.

Package Acceptance normalde aday tarball'u, `pnpm ci:full-release` ile tetiklenen tam SHA
çalıştırmaları dâhil olmak üzere çözümlenen `ref` değerinden oluşturur. Bir
beta yayımlamasından sonra, yayımlanmış npm paketini sürüm denetimleri, Package Acceptance,
işletim sistemleri arası denetimler, sürüm yolu Docker ve paket Telegram genelinde yeniden
kullanmak için `release_package_spec=openclaw@YYYY.M.PATCH-beta.N` iletin. `package_acceptance_package_spec` seçeneğini
yalnızca Package Acceptance'ın kasıtlı olarak farklı bir paketi kanıtlaması gerektiğinde
kullanın. Codex Plugin canlı paket hattı aynı durumu izler: yayımlanmış
`release_package_spec` değerlerinden `codex_plugin_spec=npm:@openclaw/codex@<version>` türetilir;
SHA/artefakt çalıştırmaları seçilen ref'ten `extensions/codex` paketler ve operatörler
`npm:`, `npm-pack:` veya `git:` Plugin kaynakları için
`codex_plugin_spec` değerini doğrudan ayarlayabilir. Hat, bu Plugin'in gerektirdiği açık
Codex CLI yükleme onayını verir, ardından Codex CLI ön kontrolünü ve aynı oturumdaki
OpenAI ajan turlarını çalıştırır. Sıfır yeniden denemeli, orta düzey düşünmeli son turu;
Codex `final` atlanmış olarak görünür ilerleme gönderir, rastgeleleştirilmiş
çalışma alanı girdilerini okur, bunların birebir artefaktını yazar ve açık bir tamamlanma
iletisi gönderir. Bu, sıradan bir ilerleme gönderiminin turu sonlandırdığı v2026.7.1
regresyonunu yakalar.

## Üst düzey aşamalar

`rerun_group=all` için önce bir `Check for reusable validation evidence` işi çalışır.
Aynı sürüm profiline, etkin dayanıklılık testi ayarına ve doğrulama girdilerine sahip,
önceki en yeni yeşil tam doğrulamayı arar. Tam hedef yeniden çalıştırmaları
`exact-target-full-validation-v1` kullanır. Eksiksiz farkı tam olarak
`CHANGELOG.md` olan bir alt commit `changelog-only-release-v1` kullanır; tüm ürün hatları
atlanır ve doğrulayıcı GitHub commit karşılaştırmasını, değişmez üst artefaktı, alt
çalıştırmaları ve tetikleme günlüklerini bağımsız olarak yeniden denetler. Diğer tüm hedef
değişiklikleri yeni bir Code SHA doğrulaması gerektirir. Yeni bir tam çalıştırmayı
zorlamak için `reuse_evidence=false` iletin. Kanıt yeniden kullanımı yalnızca
`main` veya iş akışı commit'i güvenilir `main` soyunda kalan,
standart ve SHA'ya sabitlenmiş bir `release-ci/*` ref'inden çalışır;
diğer iş akışı ref'leri seçilen hatları yeniden çalıştırır.

Yeni paket odaklı doğrulama, Plugin Prerelease ve OpenClaw Release Checks'i tetiklemeden
önce değişmez tek bir tarball ile tek bir Docker imajı artefaktı hazırlar.
Her iki alt iş akışı da kullanımdan önce aynı paket SHA'sını, artefakt kimliklerini,
hizmet özetlerini, üretici çalıştırma denemesini ve Docker arşivi özetini doğrular.
Paketten bağımsız yalın Docker katmanı, içerik adresli bir GHCR önbelleği kullanır;
adaya özgü imajlar değişmez GitHub artefaktları olarak kalır. Açıkça belirtilmiş,
yayımlanmış paket tanımına sahip odaklanmış çalıştırmalar ise mevcut paket yolunu korur.

Ayrıca `rerun_group=all` için bir `Verify Docker runtime image assets` işi,
`runtime-assets` Docker hedefini
`OPENCLAW_EXTENSIONS=diagnostics-otel,codex` ile oluşturur. Diğer aşamalarla paralel çalışır
ve üst doğrulayıcı tarafından zorunlu tutulur; hatlar artık tetiklenmeden önce
bunun tamamlanmasını beklemez. Daha dar kapsamlı bir `rerun_group` bu ön kontrolü atlar.

| Aşama                   | Ayrıntılar                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Hedef çözümleme       | **İş:** `Resolve target ref`<br />**Alt iş akışı:** yok<br />**Kanıtladığı:** sürüm dalını, etiketini veya tam commit SHA'sını çözümler ve seçilen girdileri kaydeder.<br />**Yeniden çalıştırma:** bu başarısız olursa üst iş akışını yeniden çalıştırın.                                                                                                                                                                                                                                                                                                            |
| Paylaşılan aday        | **İş:** `Prepare shared release candidate`<br />**Alt iş akışı:** `OpenClaw Live And E2E Checks (Reusable)`<br />**Kanıtladığı:** tam SHA'lı tek bir paketi paketleyip doğrular, işlevsel tek bir Docker imajı oluşturur ve paket odaklı iki alt iş akışı için değişmez paket ve imaj artefaktı demetlerini kaydeder.<br />**Yeniden çalıştırma:** etkilenen paket, plugin-prerelease, işletim sistemleri arası veya canlı/E2E grubunu yeniden çalıştırın.                                                                                                                 |
| Docker varlıkları ön kontrolü | **İş:** `Verify Docker runtime image assets`<br />**Alt iş akışı:** yok<br />**Kanıtladığı:** `runtime-assets` Docker oluşturma hedefinin, diğer aşamalar tetiklenmeden önce hâlâ başarıyla tamamlandığını kanıtlar. Yalnızca `rerun_group=all` için çalışır.<br />**Yeniden çalıştırma:** üst iş akışını `rerun_group=all` ile yeniden çalıştırın.                                                                                                                                                                                                                                         |
| Vitest ve normal CI    | **İş:** `Run normal full CI`<br />**Alt iş akışı:** `CI`<br />**Kanıtladığı:** Linux Node hatları, paketlenmiş Plugin parçaları, Plugin ve kanal sözleşmesi parçaları, Node 22 uyumluluğu, `check-*`, `check-additional-*`, oluşturulmuş artefakt hızlı kontrolleri, dokümantasyon denetimleri, Python Skills, Windows, macOS, Control UI i18n ve üst iş akışı üzerinden Android dâhil olmak üzere hedef ref'e karşı manuel tam CI grafiğini çalıştırır.<br />**Yeniden çalıştırma:** `rerun_group=ci`.                                                                                          |
| Plugin ön sürümü       | **İş:** `Run plugin prerelease validation`<br />**Alt iş akışı:** `Plugin Prerelease`<br />**Kanıtladığı:** yalnızca sürüme özgü Plugin statik denetimleri, ajansal Plugin kapsamı, tam Plugin toplu parçaları, Plugin ön sürüm Docker hatları ve uyumluluk triyajı için engelleyici olmayan bir `plugin-inspector-advisory` artefaktı.<br />**Yeniden çalıştırma:** `rerun_group=plugin-prerelease`.                                                                                                                                                          |
| Sürüm denetimleri          | **İş:** `Run release/live/Docker/QA validation`<br />**Alt iş akışı:** `OpenClaw Release Checks`<br />**Kanıtladığı:** yükleme hızlı kontrolü, işletim sistemleri arası paket denetimleri, Package Acceptance, QA Lab eşdeğerliği, canlı Matrix ve Telegram ile geçitli danışma Discord, WhatsApp ve Slack hatları. Kararlı ve tam profiller ayrıca kapsamlı canlı/E2E paketlerini ve Docker sürüm yolu parçalarını çalıştırır; beta, `run_release_soak=true` ile bunları etkinleştirebilir.<br />**Yeniden çalıştırma:** `rerun_group=release-checks` veya daha dar kapsamlı bir sürüm denetimleri tanıtıcısı.              |
| Paket Telegram        | **İş:** `Run package Telegram E2E`<br />**Alt iş akışı:** `NPM Telegram Beta E2E`<br />**Kanıtladığı:** `release_package_spec` veya `npm_telegram_package_spec` ayarlandığında, yayımlanmış pakete odaklı bir Telegram E2E. Tam aday doğrulaması bunun yerine standart Package Acceptance Telegram E2E'yi kullanır.<br />**Yeniden çalıştırma:** `release_package_spec` veya `npm_telegram_package_spec` ile `rerun_group=npm-telegram`.                                                                                                              |
| Ürün performansı     | **İş:** `Run product performance evidence`<br />**Alt iş akışı:** `OpenClaw Performance`<br />**Kanıtladığı:** hedef SHA'ya karşı sürüm profili performans çalıştırması (`profile=release`, `repeat=3`, `fail_on_regression=true`, `publish_reports=false`). Kova çıktısı iş akışı artefaktlarında kalır ve alt iş akışı, rapor yayımlayıcısının atlandığını kanıtlamalıdır. Yalnızca `rerun_group=all` veya `rerun_group=performance` için gereklidir (engelleyicidir); daha dar kapsamlı yeniden çalıştırma grupları için gerekli değildir.<br />**Yeniden çalıştırma:** `rerun_group=performance`. |
| Üst doğrulayıcı       | **İş:** `Verify full validation`<br />**Alt iş akışı:** yok<br />**Kanıtladığı:** kaydedilmiş alt iş akışı sonuçlarını yeniden denetler ve alt iş akışlarındaki en yavaş işlerin tablolarını ekler.<br />**Yeniden çalıştırma:** başarısız bir alt iş akışını yeşile dönecek şekilde yeniden çalıştırdıktan sonra yalnızca bu işi yeniden çalıştırın.                                                                                                                                                                                                                                                                 |

Üst iş akışı, ürün performansını her zaman yalnızca artefakt modunda tetikler.
`OpenClaw Performance`, rapor yayımlamaya yalnızca zamanlanmış çalıştırmalar veya
`publish_reports=true` değerini açıkça ayarlayan manuel bir tetikleme için izin verir.
Yalnızca artefakt koruması başarıyla tamamlanmalı ve yayımlayıcı işinin atlanmış
kaldığını kanıtlamalıdır. Yeni ve yeniden kullanılan kanıtlar
`controls.performanceReportPublication=artifact-only` değerini kaydeder; doğrulayıcı ve yeniden kullanım seçici,
eşleşen normalleştirilmiş performans alt iş akışı kanıtı bulunmayan kanıtları reddeder.

Doğrulayıcı, kurallı manifesti
`full-release-validation-<run-id>-<run-attempt>` olarak yükler. Kanıt araçları, tam olarak bu
yapıt kimliğini indirmeden önce yapıt kimliğini, özetini, üretici çalıştırmasını ve denemesini doğrular. İndirilen ZIP'in boyutunu sınırlar, baytlarını REST
`sha256:` özetiyle karşılaştırarak doğrular ve arşivi
çıkarmadan izin verilen tek sınırlı manifest girdisini akış olarak işler. Eski
yayınlama tüketicileri için kararlı adlı bir diğer ad geçici olarak kalır.
Doğrulayıcı her zaman deneme niteleyicili yapıtı tercih eder; geçiş sürecinde
kararlı adı yalnızca deneme-1 manifest v2 üreticisi için kabul eder. Sonraki
denemelerde ve manifest v3 için bu eski adı reddeder.

`rerun_group=all` ile `ref=main`, `release/*` referansları ve Tideclaw
alfa referansları için, daha yeni bir şemsiye çalıştırma aynı referansa ve
yeniden çalıştırma grubuna sahip eski bir çalıştırmanın yerini alır. Üst öğe iptal
edildiğinde izleyicisi, daha önce gönderdiği tüm alt iş akışlarını iptal eder.
Etiket ve sabitlenmiş SHA doğrulama çalıştırmaları birbirini iptal etmez.

## Sürüm denetimi aşamaları

`OpenClaw Release Checks` en büyük alt iş akışıdır. Hedefi
bir kez çözümler ve kullanılabilir olduğunda şemsiyenin paylaşılan paket yapıtını doğrular.
Doğrudan veya odaklanmış bir gönderim, paket ya da Docker'a yönelik aşamalar gerektirdiğinde kendi `release-package-under-test`
yapıtını hazırlar.

| Aşama                    | Ayrıntılar                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Sürüm hedefi           | **İş:** `Resolve target ref`<br />**Destekleyen iş akışı:** yok<br />**Testler:** seçilen referans, isteğe bağlı beklenen SHA, profil, yeniden çalıştırma grubu ve odaklanmış canlı paket filtresi.<br />**Yeniden çalıştırma:** `rerun_group=release-checks`.                                                                                                                                                                                                                                                                                                                                                             |
| Paket yapıtı         | **İş:** `Prepare release package artifact`<br />**Destekleyen iş akışı:** yok<br />**Testler:** şemsiyenin değişmez paket demetini doğrular veya doğrudan/odaklanmış bir Sürüm Denetimleri gönderimi için bir aday tarball paketler, ardından bunu aşağı akıştaki pakete yönelik denetimlere sunar.<br />**Yeniden çalıştırma:** etkilenen paket, işletim sistemleri arası veya canlı/E2E grubu.                                                                                                                                                                                                                                |
| Kurulum duman testi            | **İş:** `Run install smoke`<br />**Destekleyen iş akışı:** `Install Smoke`<br />**Testler:** kök Dockerfile duman testi imgesinin yeniden kullanımı, QR paket kurulumu, kök ve Gateway Docker duman testleri, yükleyici Docker testleri ve Bun genel kurulum imge sağlayıcısı duman testiyle tam kurulum yolu.<br />**Yeniden çalıştırma:** `rerun_group=install-smoke`.                                                                                                                                                                                                                                                           |
| İşletim sistemleri arası                 | **İş:** `cross_os_release_checks`<br />**Destekleyen iş akışı:** `OpenClaw Cross-OS Release Checks (Reusable)`<br />**Testler:** aday tarball ile bir temel paket kullanılarak seçilen sağlayıcı ve mod için Linux, Windows ve macOS üzerinde temiz kurulum ve yükseltme hatları.<br />**Yeniden çalıştırma:** `rerun_group=cross-os`.                                                                                                                                                                                                                                                                 |
| Depo ve canlı E2E        | **İş:** `Run repo/live E2E validation`<br />**Destekleyen iş akışı:** `OpenClaw Live And E2E Checks (Reusable)`<br />**Testler:** depo E2E'si, canlı önbellek, OpenAI websocket akışı, yerel canlı sağlayıcı ve Plugin parçaları ile `release_profile` tarafından seçilen Docker destekli canlı model/arka uç/Gateway test düzenekleri.<br />**Çalıştırmalar:** `run_release_soak=true`, `release_profile=full` veya odaklanmış `rerun_group=live-e2e`.<br />**Yeniden çalıştırma:** isteğe bağlı olarak `live_suite_filter` ile birlikte `rerun_group=live-e2e`.                                                                                |
| Docker sürüm yolu      | **İş:** `Run Docker release-path validation`<br />**Destekleyen iş akışı:** `OpenClaw Live And E2E Checks (Reusable)`<br />**Testler:** paylaşılan paket yapıtına karşı sürüm yolu Docker parçaları.<br />**Çalıştırmalar:** `run_release_soak=true`, `release_profile=full` veya odaklanmış `rerun_group=live-e2e`.<br />**Yeniden çalıştırma:** `rerun_group=live-e2e`.                                                                                                                                                                                                                                     |
| Paket Kabulü       | **İş:** `Run package acceptance`<br />**Destekleyen iş akışı:** `Package Acceptance`<br />**Testler:** çevrimdışı Plugin paket fikstürleri, Plugin güncellemesi, kurallı sahte-OpenAI Telegram paket E2E'si ve aynı tarball'a karşı yayımlanmış yükseltmeden sağ çıkma denetimleri. Engelleyici sürüm denetimleri varsayılan olarak yayımlanmış en son temeli kullanır; yük testleri (`run_release_soak=true`), bildirilen sorun yükseltme fikstürlerine karşı çalıştırılmak üzere son 4 kararlı npm sürümü ile sabitlenmiş 3 geçmiş sürüme (`2026.4.23`, `2026.5.2`, `2026.4.15`) genişler.<br />**Yeniden çalıştırma:** `rerun_group=package`. |
| Olgunluk puan kartı       | **İş:** `Render maturity scorecard release docs`<br />**Destekleyen iş akışı:** `maturity-scorecard.yml`<br />**Testler:** danışma amaçlı olgunluk puan kartı belgelerini hedef referansa göre işler. Yalnızca `run_maturity_scorecard=true` geçirildiğinde çalışır.<br />**Yeniden çalıştırma:** `run_maturity_scorecard=true` ile `rerun_group=qa`.                                                                                                                                                                                                                                                           |
| QA eşliği                | **İş:** `Run QA Lab parity lane` ve `Run QA Lab parity report`<br />**Destekleyen iş akışı:** doğrudan işler<br />**Testler:** aday ve temel ajan tabanlı eşlik paketleri, ardından eşlik raporu.<br />**Yeniden çalıştırma:** `rerun_group=qa-parity` veya `rerun_group=qa`.                                                                                                                                                                                                                                                                                                                         |
| QA çalışma zamanı eşliği        | **İş:** `Verify QA Lab runtime-pair lanes`<br />**Destekleyen iş akışı:** doğrudan iş<br />**Testler:** kurallı çekirdek `openclaw`/`codex` hattı (`pnpm openclaw qa suite --runtime-pair openclaw,codex --runtime-pair-lane core`) ve `run_release_soak=true` ile yük hattı. Danışma amaçlı: tek tek hat işleri sürüm denetimi doğrulayıcısını engellemez.<br />**Yeniden çalıştırma:** `rerun_group=qa-parity` veya `rerun_group=qa`.                                                                                                                                                             |
| QA çalışma zamanı araç kapsamı | **İş:** `Enforce QA Lab runtime tool coverage`<br />**Destekleyen iş akışı:** doğrudan iş<br />**Testler:** kurallı çekirdek çalışma zamanı çifti hattında (`pnpm openclaw qa coverage --tools`), bu hattın çıktısını kullanarak `openclaw` ile `codex` arasındaki dinamik araç sapması. Engelleyici: bu iş danışma amaçlı olarak geçersiz kılınamaz.<br />**Yeniden çalıştırma:** `rerun_group=qa-parity` veya `rerun_group=qa`.                                                                                                                                                                                                     |
| QA canlı Matrix           | **İş:** `Run QA Live Matrix profile`<br />**Destekleyen iş akışı:** `QA-Lab - All Lanes` yeniden kullanılabilir iş akışı<br />**Testler:** `qa-live-shared` ortamındaki paylaşılan Matrix canlı bağdaştırıcısı üzerinden eşliği kanıtlanmış YAML senaryoları.<br />**Yeniden çalıştırma:** `rerun_group=qa-live` veya `rerun_group=qa`; odaklanmış bir Matrix yeniden çalıştırması için `live_suite_filter=qa-live-matrix` kullanın.                                                                                                                                                                                                                    |
| QA canlı Telegram         | **İş:** `Run QA Lab live Telegram lane`<br />**Destekleyen iş akışı:** güvenilir `OpenClaw Release Telegram QA` gönderimi<br />**Testler:** Convex CI kimlik bilgisi kiralamalarıyla canlı Telegram QA.<br />**Yeniden çalıştırma:** `rerun_group=qa-live` veya `rerun_group=qa`.                                                                                                                                                                                                                                                                                                                                 |
| QA canlı Discord          | **İş:** `Run QA Lab live Discord lane`<br />**Destekleyen iş akışı:** doğrudan danışma amaçlı iş<br />**Testler:** `OPENCLAW_RELEASE_QA_DISCORD_LIVE_CI_ENABLED` etkinleştirildiğinde Convex CI kimlik bilgisi kiralamalarıyla canlı Discord QA.<br />**Yeniden çalıştırma:** `live_suite_filter=qa-live-discord` ile `rerun_group=qa-live`.                                                                                                                                                                                                                                                                            |
| QA canlı WhatsApp         | **İş:** `Run QA Lab live WhatsApp lane`<br />**Destekleyen iş akışı:** doğrudan danışma amaçlı iş<br />**Testler:** `OPENCLAW_RELEASE_QA_WHATSAPP_LIVE_CI_ENABLED` etkinleştirildiğinde Convex CI kimlik bilgisi kiralamalarıyla canlı WhatsApp QA.<br />**Yeniden çalıştırma:** `live_suite_filter=qa-live-whatsapp` ile `rerun_group=qa-live`.                                                                                                                                                                                                                                                                        |
| QA canlı Slack            | **İş:** `Run QA Lab live Slack lane`<br />**Destekleyen iş akışı:** doğrudan danışma amaçlı iş<br />**Testler:** `OPENCLAW_RELEASE_QA_SLACK_LIVE_CI_ENABLED` etkinleştirildiğinde Convex CI kimlik bilgisi kiralamalarıyla canlı Slack QA.<br />**Yeniden çalıştırma:** `live_suite_filter=qa-live-slack` ile `rerun_group=qa-live`.                                                                                                                                                                                                                                                                                    |
| Sürüm doğrulayıcısı         | **İş:** `Verify release checks`<br />**Destekleyen iş akışı:** yok<br />**Testler:** seçilen yeniden çalıştırma grubu için gerekli sürüm denetimi işleri.<br />**Yeniden çalıştırma:** odaklanmış alt işler geçtikten sonra yeniden çalıştırın.                                                                                                                                                                                                                                                                                                                                                                                   |

## Docker sürüm yolu parçaları

Docker sürüm yolu aşaması, `live_suite_filter` boş olduğunda şu parçaları
çalıştırır:

| Parça                                                           | Kapsam                                                                                                                                     |
| --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `core`                                                          | Temel Docker sürüm yolu smoke hatları.                                                                                                        |
| `package-update-openai`                                         | OpenAI paket kurulum/güncelleme davranışı, isteğe bağlı Codex kurulumu, Codex plugin canlı ilerleme takibi ve Chat Completions araç çağrıları. |
| `package-update-anthropic`                                      | Anthropic paket kurulum ve güncelleme davranışı.                                                                                               |
| `package-update-core`                                           | Sağlayıcıdan bağımsız paket ve güncelleme davranışı.                                                                                                |
| `plugins-runtime-plugins`                                       | Plugin davranışını çalıştıran plugin çalışma zamanı hatları.                                                                                          |
| `plugins-runtime-services`                                      | Hizmet destekli ve canlı plugin çalışma zamanı hatları.                                                                                                |
| `plugins-runtime-install-a` ile `plugins-runtime-install-h` arası | Paralel sürüm doğrulaması için bölünmüş plugin kurulum/çalışma zamanı grupları.                                                                        |
| `openwebui`                                                     | İstendiğinde özel bir büyük diskli çalıştırıcıda yalıtılmış OpenWebUI uyumluluk smoke testi.                                                      |

Yalnızca bir Docker hattı başarısız olduğunda yeniden kullanılabilir canlı/E2E iş akışında
hedefli `docker_lanes=<lane[,lane]>` kullanın. Sürüm yapıtları, mevcut olduğunda paket
yapıtı ve görüntü yeniden kullanım girdilerini içeren hat başına yeniden çalıştırma
komutlarını içerir.

## Sürüm profilleri

`release_profile`, sürüm denetimlerindeki canlı/sağlayıcı kapsamını büyük ölçüde kontrol eder.
Normal tam CI'ı, Plugin Ön Sürümünü, kurulum smoke testini, paket
kabulünü veya QA Lab'i kaldırmaz. Kararlı ve tam profiller her zaman kapsamlı depo/canlı
E2E ve Docker sürüm yolu dayanıklılık kapsamını çalıştırır. Beta profili
`run_release_soak=true` ile bunu etkinleştirebilir. Paket Kabulü, her tam aday
için standart paket Telegram E2E'sini sağladığından, çatı iş akışı bu
canlı yoklayıcıyı tekrarlamaz.

| Profil  | Amaçlanan kullanım                      | Dahil edilen canlı/sağlayıcı kapsamı                                                                                                                                                                            |
| -------- | --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `beta`   | En hızlı, sürüm açısından kritik smoke testi.   | OpenAI/temel canlı yol, OpenAI için Docker canlı modelleri, yerel gateway temeli, yerel OpenAI gateway profili, yerel OpenAI plugin'i ve Docker canlı gateway OpenAI.                                            |
| `stable` | Varsayılan sürüm onay profili. | `beta` artı Anthropic smoke testi, Google, MiniMax, arka uç, yerel canlı test düzeneği, Docker canlı CLI arka ucu, Docker ACP bağlama, Docker Codex düzeneği, Docker alt ajan duyurusu ve bir OpenCode Go smoke parçası. |
| `full`   | Geniş danışma amaçlı tarama.             | `stable` artı danışma amaçlı sağlayıcılar, plugin canlı parçaları ve medya canlı parçaları.                                                                                                                               |

## Yalnızca tam profile eklenenler

Bu test paketleri `stable` tarafından atlanır ve `full` tarafından dahil edilir:

| Alan                             | Yalnızca tam profil kapsamı                                                                                                          |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Docker canlı modelleri               | OpenCode Go, OpenRouter, xAI, Z.ai ve Fireworks.                                                                          |
| Docker canlı gateway              | DeepSeek/Fireworks, OpenCode Go/OpenRouter ve xAI/Z.ai parçalarına bölünmüş danışma amaçlı sağlayıcılar.                              |
| Yerel gateway sağlayıcı profilleri | Tam Anthropic Opus ve Sonnet/Haiku parçaları, Fireworks, DeepSeek, tam OpenCode Go model parçaları, OpenRouter, xAI ve Z.ai. |
| Yerel plugin canlı parçaları        | Plugin'ler A-K, L-N, O-Z diğerleri, Moonshot ve xAI.                                                                             |
| Yerel medya canlı parçaları         | Ses, Google müzik, MiniMax müzik ve A-D video grupları.                                                                   |

`stable`, `native-live-src-gateway-profiles-anthropic-smoke` ve
`native-live-src-gateway-profiles-opencode-go-smoke` öğelerini içerir; `full` ise bunun yerine daha geniş
Anthropic ve OpenCode Go model parçalarını kullanır. Odaklı yeniden çalıştırmalar yine
toplu `native-live-src-gateway-profiles-anthropic` veya
`native-live-src-gateway-profiles-opencode-go` tanıtıcılarını kullanabilir.

## Odaklı yeniden çalıştırmalar

İlgisiz sürüm kutularını yinelemekten kaçınmak için `rerun_group` kullanın:

| Tanıtıcı              | Kapsam                                                                                           |
| ------------------- | ----------------------------------------------------------------------------------------------- |
| `all`               | Tüm Tam Sürüm Doğrulama aşamaları.                                                             |
| `ci`                | Yalnızca manuel tam CI alt iş akışı.                                                                      |
| `plugin-prerelease` | Yalnızca Plugin Ön Sürümü alt iş akışı.                                                                   |
| `release-checks`    | Tüm OpenClaw Sürüm Denetimleri aşamaları.                                                             |
| `install-smoke`     | Kurulum Smoke Testinden sürüm denetimlerine kadar.                                                           |
| `cross-os`          | İşletim sistemleri arası sürüm denetimleri.                                                                        |
| `live-e2e`          | Depo/canlı E2E ve Docker sürüm yolu doğrulaması.                                               |
| `package`           | Paket Kabulü.                                                                             |
| `qa`                | QA eşdeğerliği ve QA canlı hatları.                                                                   |
| `qa-parity`         | Yalnızca QA eşdeğerlik hatları ve raporu.                                                                |
| `qa-live`           | QA canlı Matrix/Telegram ile etkinleştirildiğinde kapılı Discord, WhatsApp ve Slack hatları.             |
| `npm-telegram`      | Yayımlanmış paket Telegram E2E'si; `release_package_spec` veya `npm_telegram_package_spec` gerektirir. |
| `performance`       | Yalnızca ürün performansı kanıtı.                                                              |

Bir canlı test paketi başarısız olduğunda `rerun_group=live-e2e` ile `live_suite_filter` kullanın.
Geçerli filtre kimlikleri, yeniden kullanılabilir canlı/E2E iş akışında tanımlanır ve
`docker-live-models`, `live-gateway-docker`,
`live-gateway-anthropic-docker`, `live-gateway-google-docker`,
`live-gateway-minimax-docker`, `live-gateway-advisory-docker`,
`live-cli-backend-docker`, `live-acp-bind-docker` ve
`live-codex-harness-docker` değerlerini içerir.

Odaklı bir QA aktarım yeniden çalıştırması için `rerun_group=qa-live` ayarlayın ve
standart seçici `qa-live-matrix`, `qa-live-telegram`, `qa-live-discord`,
`qa-live-whatsapp` veya `qa-live-slack` değerini kullanın.

`live-gateway-advisory-docker` tanıtıcısı, üç sağlayıcı parçası için toplu bir yeniden çalıştırma
tanıtıcısıdır; bu nedenle yine tüm danışma amaçlı Docker gateway işlerine yayılır.

İşletim sistemleri arası hatlardan biri başarısız olduğunda `rerun_group=cross-os` ile `cross_os_suite_filter` kullanın.
Filtre bir işletim sistemi kimliği, test paketi kimliği veya işletim sistemi/test paketi çifti kabul eder;
örneğin `windows/packaged-upgrade`, `windows` veya `packaged-fresh`. İşletim sistemleri arası
özetler, paketlenmiş yükseltme hatları için aşama başına zamanlamaları içerir ve uzun süren
komutlar, takılı kalan bir güncellemenin iş zaman aşımından önce görünür olması için
Heartbeat satırları yazdırır.

QA sürüm denetimi hataları, normal sürüm doğrulamasını yalnızca seçili
Matrix, Telegram ve QA çalışma zamanı araç kapsamı hatları için engeller. QA eşdeğerliği, çalışma zamanı
eşdeğerliği ve kapılı Discord, WhatsApp ve Slack canlı hatları danışma amaçlıdır ve
sürüm doğrulayıcıyı engellemeden durum yapıtları yayımlar. Tideclaw
alfa çalıştırmaları, paket güvenliği dışındaki sürüm denetimi hatlarını yine danışma amaçlı kabul edebilir. 
`release_profile=beta` ile `Run repo/live E2E validation` canlı sağlayıcı test paketleri
danışma amaçlıdır: üçüncü taraf model dağıtımları bir sürümün altında değiştiğinden,
beta bunların hatalarını uyarı olarak gösterirken kararlı ve tam profiller bunları
engelleyici tutar.
`live_suite_filter`, Discord, WhatsApp veya Slack gibi kapılı bir QA canlı hattını
açıkça istediğinde eşleşen `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED` depo
değişkeni etkinleştirilmiş olmalıdır; aksi takdirde hat sessizce atlanmak yerine girdi yakalama başarısız olur.
Yeni QA kanıtı gerektiğinde `rerun_group=qa`, `qa-parity` veya `qa-live` öğesini
yeniden çalıştırın.

## Saklanacak kanıtlar

Sürüm düzeyi dizin olarak `Full Release Validation` özetini saklayın. Bu özet
alt çalıştırma kimliklerine bağlantı verir ve en yavaş iş tablolarını içerir. Hatalarda önce alt
iş akışını inceleyin, ardından yukarıdaki en küçük eşleşen tanıtıcıyı yeniden çalıştırın.

Normal bir sürüm için hem Kod SHA'sını hem Sürüm SHA'sını, yeniden kullanım politikasını
ve değiştirilen yollar kümesini, yeşil Kod SHA üst çalıştırmasını ve hafif Sürüm
SHA üst çalıştırmasını kaydedin. Uzatılmış kararlı sürüm için standart dalı, tam sürüm
SHA'sını, yeni üst çalıştırma kimliğini ve denemesini, iş akışı referansını, her alt çalıştırmayı ve
dondurulmuş hedefe yönelik uyumluluk onarımlarını veya kasıtlı atlamaları kaydedin.

Yararlı yapıtlar:

- `OpenClaw Release Checks` kaynağından `release-package-under-test`
- `.artifacts/docker-tests/` altındaki Docker sürüm yolu yapıtları
- Paket Kabulü `package-under-test` ve Docker kabul yapıtları
- Her işletim sistemi ve test paketi için işletim sistemleri arası sürüm denetimi yapıtları
- QA eşdeğerliği, çalışma zamanı eşdeğerliği ve seçili Matrix, Telegram, Discord, WhatsApp
  veya Slack yapıtları

## İş akışı dosyaları

- `.github/workflows/full-release-validation.yml`
- `.github/workflows/openclaw-release-checks.yml`
- `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml`
- `.github/workflows/plugin-prerelease.yml`
- `.github/workflows/install-smoke.yml`
- `.github/workflows/install-smoke-reusable.yml`
- `.github/workflows/openclaw-cross-os-release-checks-reusable.yml`
- `.github/workflows/package-acceptance.yml`
- `.github/workflows/openclaw-performance.yml`
- `.github/workflows/npm-telegram-beta-e2e.yml`
