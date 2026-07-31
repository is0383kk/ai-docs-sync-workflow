---
read_when:
    - Bir CI işinin neden çalıştığını veya çalışmadığını anlamanız gerekiyor
    - Başarısız olan bir GitHub Actions denetiminde hata ayıklıyorsunuz
    - Bir sürüm doğrulama çalışmasını veya yeniden çalıştırmasını koordine ediyorsunuz
    - ClawSweeper yönlendirmesini veya GitHub etkinliği iletimini değiştiriyorsunuz
summary: CI iş grafiği, kapsam kapıları, sürüm şemsiyeleri ve yerel komut eşdeğerleri
title: CI işlem hattı
x-i18n:
    generated_at: "2026-07-26T22:35:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9de5b527354f3cc9eed3813e961116f3834c61bd72b29c92f762c46722815df
    source_path: ci.md
    workflow: 16
---

OpenClaw CI, `main` dalına yapılan göndermelerde (Markdown ve `docs/**` yolları tetikleyicide yok sayılır), taslak olmayan her pull request'te ve manuel tetiklemede çalışır.
Kanonik `main` göndermeleri tek seferde bir tane çalışır: `CI` eşzamanlılık grubu, GitHub yalnızca bekleyen en yeni göndermeyi tutarken bir tam entegrasyon döngüsünün çalışmasına izin verir.
Yeni birleştirmeler, Blacksmith matrisini zaten kaydetmiş çalışmayı iptal etmek yerine bekleyen bu çalışmanın yerini alır.
Pull request'ler geçersiz kılınan head'leri iptal etmeye devam eder ve manuel tetiklemeler yalıtılmış gruplar kullanır. `preflight` farkı sınıflandırır ve yalnızca ilgisiz alanlar değiştiğinde maliyetli iş hatlarını kapatır. Manuel
`workflow_dispatch` çalışmaları, akıllı kapsam belirlemeyi kasıtlı olarak atlar ve sürüm adayları ile geniş kapsamlı doğrulama için
tam grafiği paralel kollara ayırır. Android iş hatları
`include_android` (veya `release_gate` girdisi) aracılığıyla isteğe bağlı kalır. Yalnızca sürüme yönelik
plugin kapsamı ayrı
[`Plugin Prerelease`](#plugin-prerelease) iş akışında bulunur ve yalnızca
[`Full Release Validation`](#full-release-validation) üzerinden veya açık bir manuel
tetiklemeyle çalışır.

## İşlem hattına genel bakış

| İş                                | Amaç                                                                                                                                                                                                               | Çalışma koşulu                                   |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| `preflight`                        | Değişen kapsamları algılar ve CI bildirimini oluşturur; Node ile ilgili kanonik `main` üzerinde, paralel kollara ayrılmadan önce bağımlılık anlık görüntüsünü yeniler ve sürdürür                                                                        | Taslak olmayan gönderme ve PR'lerde her zaman             |
| `security-fast`                    | Özel anahtar algılama, `zizmor` aracılığıyla değişen iş akışlarını denetleme ve üretim kilit dosyasını denetleme                                                                                                                             | Taslak olmayan gönderme ve PR'lerde her zaman             |
| `pnpm-store-warmup`                | Linux Node parçalarını engellemeden pull request'ler ve manuel çalışmalar için kilit dosyasına sabitlenmiş Actions önbelleğini hazırlar                                                                                                           | Ana dal dışında Node veya doküman denetimi iş hatları seçildiğinde |
| `build-artifacts`                  | `dist/`, Control UI, derlenmiş CLI işleyiş denetimleri, başlangıç belleği ve gömülü derleme yapıtı denetimlerini derler                                                                                                                 | Node ile ilgili değişikliklerde                          |
| `control-ui-i18n`                  | Oluşturulan Control UI yerel ayar paketlerini, meta verileri ve çeviri belleğini doğrular; otomatik çalışmalarda bilgilendiricidir, manuel sürüm CI'ında engelleyicidir                                                                               | Control UI i18n ile ilgili değişikliklerde ve manuel CI'da |
| `checks-fast-core`                 | Hızlı Linux doğruluk iş hatları: bastırma temel çizgisi azami satır sayısı mandalı, paketlenmiş bileşenler + protokol, Bun başlatıcısı ve CI yönlendirme hızlı görevi                                                                                  | Node ile ilgili değişikliklerde                          |
| `qa-smoke-ci-profile`              | Sınırlı otomatik QA Smoke temsili kümesinin kendi kendine yeterli, dengeli iki parçası; taksonominin tam kapsamı açık QA profilleri aracılığıyla kullanılabilir olmaya devam eder                                                         | Node ile ilgili değişikliklerde                          |
| `checks-fast-contracts-plugins-*`  | Ağırlıklandırılmış iki plugin sözleşmesi parçası                                                                                                                                                                                   | Node ile ilgili değişikliklerde                          |
| `checks-fast-contracts-channels-*` | Ağırlıklandırılmış iki kanal sözleşmesi parçası                                                                                                                                                                                  | Node ile ilgili değişikliklerde                          |
| `checks-node-*`                    | Pull request'lerde değişen hedeflere yönelik Node testleri; `main`, manuel, sürüm ve geniş kapsamlı geri dönüş çalışmalarında tüm çekirdek parçaları                                                                                                      | Node ile ilgili değişikliklerde                          |
| `check-*`                          | Parçalara ayrılmış ana yerel geçit eşdeğeri: korumalar, shrinkwrap, paketlenmiş kanal yapılandırma meta verileri, üretim türleri, lint, bağımlılıklar, test türleri                                                                                   | Node ile ilgili değişikliklerde                          |
| `check-additional-*`               | Sınır denetimi şeritleri (istem anlık görüntüsü sapması dâhil), oturum erişimcisi/transkript okuyucusu/SQLite işlem sınırları, uzantı lint grupları, paket sınırı derleme/canary ve çalışma zamanı topolojisi mimarisi | Node ile ilgili değişikliklerde                          |
| `checks-node-compat-node22`        | Node 22 uyumluluk derlemesi ve işleyiş denetimi iş hattı                                                                                                                                                                            | Sürümler için manuel CI tetiklemesinde                |
| `check-docs`                       | Doküman biçimlendirme, lint ve bozuk bağlantı denetimleri                                                                                                                                                                         | Dokümanlar değiştiğinde (PR'ler ve manuel tetikleme)         |
| `native-i18n`                      | Kaynak PR'lerde yerel kaynak çıkarma ve yerelleştirme güvenliğini doğrular; oluşturulan PR'lerde ve manuel CI'da çevrilmiş/platform tarafından oluşturulmuş tam eşitliği zorunlu kılar                                                               | Yerel i18n ile ilgili değişikliklerde                   |
| `skills-python`                    | Python destekli skills için Ruff + pytest                                                                                                                                                                                | Python skill'leriyle ilgili değişikliklerde                  |
| `checks-windows`                   | Windows'a özgü süreç/yol testleri ve paylaşılan çalışma zamanı içe aktarma belirteci regresyonları                                                                                                                                  | Windows ile ilgili değişikliklerde                       |
| `macos-node`                       | Odaklı macOS TypeScript testleri: launchd, Homebrew, çalışma zamanı yolları, paketleme betikleri, süreç grubu sarmalayıcısı                                                                                                            | macOS ile ilgili değişikliklerde                         |
| `macos-swift`                      | macOS uygulaması için Swift lint ve derlemenin yanı sıra uygulama ve paylaşılan OpenClawKit paketi testleri                                                                                                                         | macOS ile ilgili değişikliklerde                         |
| `ios-build`                        | Xcode proje oluşturma ve iOS uygulaması simülatör derlemesi                                                                                                                                                             | iOS uygulaması, paylaşılan uygulama kiti veya Swabble değişikliklerinde    |
| `android`                          | Her iki çeşide yönelik Android birim testleri ve bir hata ayıklama APK derlemesi                                                                                                                                                          | Android ile ilgili değişikliklerde                       |
| `openclaw/ci-gate`                 | Nihai toplama: ön kontrol ve güvenlik gerektirir; atlamaları yalnızca bildirim tarafından devre dışı bırakılan aşağı akış iş hatları için kabul eder                                                                                                           | Taslak olmayan her CI çalışmasında                         |
| `test-performance-agent`           | Ayrı iş akışı: güvenilir etkinlikten sonra günlük Codex yavaş test optimizasyonu                                                                                                                                          | Ana CI başarısında veya manuel tetiklemede             |
| `openclaw-performance`             | Ayrı iş akışı: sahte sağlayıcı, derin profil ve GPT 5.6 canlı iş hatlarıyla günlük/isteğe bağlı Kova çalışma zamanı performans raporları                                                                                          | Zamanlanmış ve manuel tetiklemede                  |

Bağımsız Periphery iş akışları, iOS ve macOS uygulamalarında ölü kod bulgusu olmamasını zorunlu kılar. Paylaşılan OpenClawKit iş akışı her iki tüketiciyi paralel olarak tarar ve bir bildirimi yalnızca Periphery her iki derlemeden aynı Swift USR'yi ürettiğinde raporlar. Oluşturulan `OpenClawProtocol/GatewayModels.swift` şema sözleşmesi, uygulamaya özgü ölü kod olarak değerlendirilmek yerine oluşturucuya ait kod olarak tutulur.

## Hızlı başarısızlık sırası

1. `preflight`, hangi iş hatlarının var olacağına karar verir. `docs-scope` ve `changed-scope` mantığı bağımsız işler değil, bu işin içindeki adımlardır. Kanonik `main` hemen başlar ancak eşzamanlılık grubu yalnızca bir tam çalışmayı kabul eder ve sonraki göndermeleri bekleyen en yeni tek çalışma hâlinde birleştirir. Node ile ilgili ana dal göndermeleri ayrıca, aşağı akış işleri anahtarı bağlamadan önce tek bağımlılık diski yazıcısını ve bunun boyut bakımını burada sıralı hâle getirir; Blacksmith yeni bir commit'i yalnızca sonraki bir iş akışı çalışmasına sunabilir, bu nedenle aynı çalışmadaki tüketiciler işaretçiyle denetlenen yerel geri dönüşü korur.
2. `security-fast`, `check-*`, `check-additional-*`, `check-docs` ve `skills-python`, daha ağır yapıt ve platform matrisi işlerini beklemeden hızla başarısız olur.
3. `build-artifacts` ve yerel ayar denetimleri, hızlı Linux iş hatlarıyla çakışır. Control UI ve yerel uygulama kaynak PR'leri, oluşturulan yerel ayar anlık görüntülerini/kaynaklarını hariç tutar; bunların sıralı yenileme iş akışları, yalıtılmış oluşturulan PR'leri arka planda onarır ve otomatik olarak birleştirir. Kaynak CI, güncelliğini yitirmiş kaynak envanterlerini ve güvenli olmayan yerelleştirme çağrılarını yine de engeller. Oluşturulan PR'ler, manuel CI ve sürüm hazırlığı; çevrilmiş/platform tarafından oluşturulmuş tam eşitliği zorunlu kılar. Kanonik `release/YYYY.M.PATCH` dalları, oluşturulan diğer sürüm çıktılarıyla birlikte sürüm hazırlığı yerel ayar onarımlarını içerebilir.
4. Bundan sonra daha ağır platform ve çalışma zamanı iş hatları paralel kollara ayrılır: `checks-fast-core`, `checks-fast-contracts-plugins-*`, `checks-fast-contracts-channels-*`, `checks-node-*`, `checks-windows`, `macos-node`, `macos-swift`, `ios-build` ve `android`.
5. `openclaw/ci-gate`, seçilen her iş hattını bekler. Ön kontrol ve güvenlik başarılı olmalıdır; aşağı akış işleri yalnızca bildirim onları seçmediyse atlanabilir. Başarısız olan veya iptal edilen seçili bir iş hattı, toplamayı başarısız kılar.

Birleştirme koordinatörü, aynı pull request head'i için kimliği doğrulanmış başarılı bir `openclaw/ci-gate`
sonucunu 24 saate kadar yeniden kullanabilir. Bu, ilgisiz `main` değişikliklerinden sonra
katkıda bulunan dalının yeniden yazılmasını önler. Yeniden kullanılabilir sonuç,
mevcut `main` ile karşılaştırılan ayrı, katı ve uygulamaya ait test birleştirme denetiminin yerini almaz.
Daha sonra bekleyen veya başarısız olan bir yeniden çalışma, yenilik süresi içinde
değişmemiş bu head için daha önce alınmış başarılı sonucu silmez.

Varsayılan dal kural kümesi, GitHub Actions'a ait `openclaw/ci-gate` denetimini gerektirir. Depo bakımcıları ve yöneticileri, yalnızca imzalı doğrudan hızlı ileri alma yoluyla birleştirmeler için tasarlanmış, denetlenen bir acil durum atlama yetkisine sahiptir; kuruluş kural kümesi silme ve hızlı ileri alma olmayan güncellemeleri engellemeye devam eder. Normal pull request birleştirmeleri, başarısız CI'ı atlamak yerine geçidi kullanmaya devam etmelidir. Ayrı katı, App'e ait test birleştirme denetimi, head'i hâlâ geçerli `main` değerine bağlar.

GitHub, daha yeni bir head birleştirildiğinde yerini yenisine bırakan pull request işlerini `cancelled` olarak işaretleyebilir. Aynı PR'ın en yeni çalıştırması da başarısız değilse bunu CI gürültüsü olarak değerlendirin. Kanonik `main` çalıştırmaları kabulden sonra iptal edilmez; birleştirme trafiği geldiğinde GitHub, yalnızca bekleyen eski çalıştırmayı en yeni uçla değiştirir. Matris işleri `fail-fast: false` kullanır ve `build-artifacts`, küçük doğrulayıcı işleri kuyruğa almak yerine gömülü kanal, çekirdek destek sınırı ve gateway izleme hatalarını doğrudan bildirir. Otomatik CI eşzamanlılık anahtarı sürümlüdür (`CI-v7-*`); böylece eski bir kuyruk grubundaki GitHub taraflı bir zombi, daha yeni main çalıştırmalarını süresiz olarak engelleyemez. Manuel tam paket çalıştırmaları `CI-manual-v1-*` kullanır ve devam eden çalıştırmaları iptal etmez. Plugin listesi başlangıç belleği koruması, kendi barındırılan Blacksmith Linux'ta 350 MiB tavanını korur ve aynı derlenmiş CLI için RSS taban değeri daha yüksek olan GitHub tarafından barındırılan Linux'ta 425 MiB'a izin verir.

GitHub Actions'tan geçen duvar saati süresini, kuyruk süresini, en yavaş işleri, hataları ve `pnpm-store-warmup` yayılım engelini özetlemek için `pnpm ci:timings`, `pnpm ci:timings:recent` veya `node scripts/ci-run-timings.mjs <run-id>` kullanın. İş akışı içindeki `ci-timings-summary` işi `ci.yml` içinde bulunur ancak şu anda devre dışıdır (`if: false`); bunun yerine zamanlama yardımcısını yerel olarak çalıştırın. Derleme zamanlaması için `build-artifacts` işinin `Build dist` adımını kontrol edin: `pnpm build:ci-artifacts`, `[build-all] phase timings:` çıktısını verir ve `ui:build` öğesini içerir; iş ayrıca `startup-memory` yapıtını yükler.

## PR bağlamı ve kanıt

Harici katkıcı PR'ları, şuradan bir PR bağlamı ve kanıt geçidi çalıştırır:
`.github/workflows/real-behavior-proof.yml`. İş akışı, güvenilir iş akışı revizyonunu
(`github.workflow_sha`) kullanıma alır ve yalnızca PR gövdesini değerlendirir;
katkıcı dalındaki kodu çalıştırmaz.

Geçit; depo sahibi, üyesi, işbirlikçisi veya bot olmayan PR yazarlarına
uygulanır. PR gövdesi, yazar tarafından oluşturulmuş `What Problem This Solves` ve
`Evidence` bölümlerini içerdiğinde geçer. Kanıt; odaklı bir test, CI
sonucu, ekran görüntüsü, kayıt, terminal çıktısı, canlı gözlem, hassas bilgileri
gizlenmiş günlük veya yapıt bağlantısı olabilir. Gövde, amacı ve yararlı
doğrulamayı sunar; inceleyenler doğruluğu değerlendirmek için kodu, testleri ve
CI'ı inceler.

Denetim başarısız olduğunda başka bir kod commit'i göndermek yerine PR gövdesini güncelleyin.

## Kapsam ve yönlendirme

Kapsam mantığı `scripts/ci-changed-scope.mjs` içinde bulunur ve `src/scripts/ci-changed-scope.test.ts` içindeki birim testleriyle kapsanır. Manuel tetikleme, değişen kapsam algılamasını atlar ve ön kontrol manifestinin, kapsamlı alanların tümü değişmiş gibi davranmasını sağlar.

Ayrı iOS ve macOS Periphery iş akışları, sıfır bulgulu bir ölü kod politikası uygular. Her biri yalnızca taslak olmayan bir pull request kendi yerel tarama kapsamına dokunduğunda veya manuel olarak tetiklendiğinde çalışır.

- **CI iş akışı düzenlemeleri**, Node CI grafiğini, iş akışı lint denetimini ve Windows kulvarını doğrular (`ci.yml` bunu çalıştırır), ancak tek başlarına iOS, Android veya macOS yerel derlemelerini zorunlu kılmaz; bu platform kulvarları, platform kaynak değişiklikleriyle sınırlı kalır.
- **İş Akışı Sağlamlık Denetimi**, tüm iş akışı YAML dosyalarında `actionlint` ve `zizmor`, bileşik eylem enterpolasyon koruması ve çakışma işareti korumasını çalıştırır. PR kapsamlı `security-fast` işi ayrıca değişen iş akışı dosyalarında `zizmor` çalıştırır; böylece iş akışı güvenliği bulguları ana CI grafiğinde erkenden başarısız olur.
- `main` gönderimlerindeki **belgeler**, CI'ın kullandığı aynı ClawHub belge yansısıyla bağımsız `Docs` iş akışı tarafından denetlenir; böylece karma kod+belge gönderimleri ayrıca CI `check-docs` parçasını kuyruğa almaz. Pull request'ler ve manuel CI, belgeler değiştiğinde `check-docs` öğesini CI'dan çalıştırmaya devam eder.
- **TUI PTY**, TUI değişiklikleri için `checks-node-core-runtime-tui-pty` Linux Node parçasında çalışır. Parça, `test/vitest/vitest.tui-pty.config.ts` öğesini `OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1` ile çalıştırır; böylece hem belirlenimci `TuiBackend` sabit veri kulvarını hem de yalnızca harici model uç noktasını taklit eden daha yavaş `tui --local` duman testini kapsar.
- **Yalnızca CI yönlendirmesi düzenlemeleri, hızlı görevin doğrudan çalıştırdığı küçük çekirdek test sabit verileri kümesi ve dar kapsamlı Plugin sözleşmesi yardımcısı düzenlemeleri**, hızlı, yalnızca Node kullanan bir manifest yolu kullanır: `preflight`, `security-fast` ve yalnızca değişikliğin dokunduğu hızlı kulvarlar — tek bir `checks-fast-core` CI yönlendirme görevi, iki Plugin sözleşmesi parçası veya her ikisi. Bu yol; derleme yapıtlarını, Node 22 uyumluluğunu, kanal sözleşmelerini, tam çekirdek parçalarını, paketlenmiş Plugin parçalarını ve ek koruma matrislerini atlar.
- **Windows Node denetimleri**, Windows'a özgü süreç/yol sarmalayıcıları, npm/pnpm/UI çalıştırıcı yardımcıları, paket yöneticisi yapılandırması ve bu kulvarı çalıştıran CI iş akışı yüzeyleriyle sınırlıdır; ilgisiz kaynak, Plugin, kurulum duman testi ve yalnızca test değişiklikleri Linux Node kulvarlarında kalır.

En yavaş Node test aileleri, her işin çalıştırıcıları gereğinden fazla ayırmadan küçük kalması için bölünür veya dengelenir:

- Plugin sözleşmeleri ve kanal sözleşmelerinin her biri, standart GitHub çalıştırıcısı geri dönüşüyle birlikte Blacksmith destekli, ağırlıklandırılmış iki parça olarak çalışır.
- Çekirdek birim hızlı/destek hatları ayrı ayrı çalışır; çekirdek çalışma zamanı altyapısı süreç, paylaşılan, kancalar, gizli bilgiler ve üç cron alan parçasına bölünür.
- Otomatik yanıt, dengeli işçiler olarak çalışır; yanıt alt ağacı agent-runner, komutlar, dağıtım, oturum ve durum yönlendirme parçalarına bölünür.
- Aracılı gateway/sunucu (kontrol düzlemi) yapılandırmaları, oluşturulmuş yapıtları beklemek yerine sohbet, kimlik doğrulama, model, HTTP/plugin, çalışma zamanı ve başlatma hatlarına bölünür.
- Normal CI, yalnızca yalıtılmış altyapı include-pattern parçalarını en fazla 64 test dosyasından oluşan belirlenimci paketlerde toplar; böylece yalıtılmamış komut/cron, durum bilgili agents-core veya gateway/sunucu paketlerini birleştirmeden Node matrisi küçültülür. Ağır sabit paketler 8 vCPU'da kalırken paketlenmiş ve daha düşük ağırlıklı hatlar 4 vCPU kullanır.
- Standart depodaki pull request'ler, sentetik birleştirilmiş ağaç farkına karşı değiştirilmiş test çözümleyicisini yeniden kullanır. Kesin değişiklikler tek bir hedefli Node işi çalıştırır; durum bilgili paket yalıtımının bozulmaması için seçilen her test dosyası kendi sürecini alır. Planlayıcı, kardeş testleri içe aktarma grafiğindeki bağımlılarla birleştirir ve çalışma alanı paketi, paket/kilit dosyası, paylaşılan test düzeneği, bölünmüş yapılandırma, yeniden adlandırılmış veya silinmiş değişiklikler, herkese açık uzantı sözleşmesi değişiklikleri, özel parça kurulumu olan testler, kısmen çözümlenmiş veya boş hedefler, aşırı büyük yol ya da hedef planları ve planlayıcı hataları için mevcut 14 işlik kompakt tam paket planına geri döner. Hedefli planlar, depo tarayıcıları içe aktarmalardan türetilemediği için oluşturulmuş yapıt sınırı geçidinin tamamını her zaman korur. `main` göndermeleri aynı tam kompakt paketi çalıştırır: bekleyen ara gönderme olayları birleştirilebildiğinden, ayakta kalan en yeni çalıştırma yalnızca son tek gönderme farkını değil, eksiksiz tümleştirme ağacını doğrulamalıdır. Manuel çalıştırmalar ve sürüm geçitleri, parça başına tam adlandırılmış matrisi korur.
- Tam Node matrisi, sürekli yavaş olan seri araçları, otomatik yanıt komut parçalarını ve geniş core-fast önbellek yazıcısını önce kabul eder. Bu, kritik yol çalışmasının ve sonraki çalıştırmanın dönüşüm çekirdeğinin sonraki bir dalgaya kaymasını önlerken 28 iş sınırını korur.
- Geniş tarayıcı, QA, medya ve çeşitli plugin testleri, paylaşılan genel plugin yapılandırması yerine kendilerine ayrılmış Vitest yapılandırmalarını kullanır. Include-pattern parçaları zamanlama girdilerini CI parçasının adını kullanarak kaydeder; böylece `.artifacts/vitest-shard-timings.json` bütün bir yapılandırmayı filtrelenmiş bir parçadan ayırt edebilir.
- Linux Node parça işleri, Vitest'in deneysel dosya sistemi modül önbelleğini, Blacksmith'in kendi çalıştırıcılarında şeffaf biçimde hızlandırdığı üst kaynak Actions önbellek API'si aracılığıyla kalıcı tutar. Her CI parçası yalnızca geri yükleme yapar ve korumalı çekirdeği kendi çalıştırıcı yerel köküne açar; ardından parça sarmalayıcısı eşzamanlı Vitest süreçlerine ayrı canlı alt dizinler verir. Yalnızca iptal edilmeyen günlük veya açıkça çalıştırılan ısıtıcı yeni bir değişmez arşiv kaydeder; böylece pull request'ler dönüşüm yayımlayamaz veya PR başına önbellek aileleri oluşturamaz. Dönüşüm girdisi parmak izi, uyumsuz kilit dosyası, paket, tsconfig ve Vitest yapılandırması nesillerini temizler. Korumalı yazıcı, geri yüklenen önbelleği tarar ve 2 GiB'yi aştıktan sonra %75'e düşürür. Vitest; modül kimliği, kaynak içeriği, ortam ve çözümlenmiş dönüşüm yapılandırmasının karmasını alır; böylece sıradan kısmi kaynak değişikliklerinde değişmeyen girdiler sıcak kalırken değiştirilen modüller güvenli biçimde önbellek ıskalaması yaşar. Kaba geri yükleme önekleri iş akışı çalıştırmaları arasında köprü kurar; normal Actions önbelleği LRU ve hareketsizlik tahliyesi eski değişmez arşivleri sınırlar.
- Güvenilir Linux Node işleri ayrıca pnpm deposunu ve `node_modules` öğesini, desteklenen her Node hattı için tek bir korumalı bağımlılık diskinden bağlar. Paket manifestoları, kurulum ayarları, çalıştırıcı platformu ve tam Node yama sürümü disk anahtarının dışında kalır; bir işin ağacı yeniden kullanacağına mı yoksa yeniden kurup aynı diski yenileyeceğine mi tam çalışma zamanı ve kurulum girdisi parmak izi karar verir. Manifestolar karma alınmadan önce standartlaştırılır. Denetlenmiş doğrudan kök kancalar yalnızca pnpm'in kurulum yaşam döngüsü betiklerini korur; böylece biçimlendirme ve sıradan test/derleme betiği düzenlemeleri sıcak bağımlılık ağacını korur; denetlenmemiş yaşam döngüsü kancası sapmaları ise kaynak girdileri parmak izi sözleşmesine katılana kadar güvenli biçimde başarısız olur. Bağımlılık, paket yöneticisi, kanca kaynağı ve kilit dosyası değişiklikleri anlık görüntüyü her zaman geçersiz kılar. Eşleşen bir parmak izi gereklidir ancak yeterli değildir: kurulum ayrıca içe aktarıcı arşivini ve manifesto sağlama toplamlarını denetler, ardından postinstall tarafından tutulan kayıt defteri destekli kilit dosyası bağımlılıklarını, Node'un içe aktarıcılarından çözümlediği paket manifestolarına karşı doğrular. Eksik veya eski içe aktarıcı içeriği, kök hoist'i sunmak yerine yeni bir kuruluma geri döner. Salt okunur anlık görüntüsü kullanılamayan bir pull request, çalışma alanı bağını ayırır ve yayımlayamayacağı bir klona yavaş yazma işlemlerinden kaçınmak için çalıştırıcı yerel depolamasına kurulum yapar. Yapışkan soğuk kurulumlar, pnpm'in iç getirme yeniden denemelerini devre dışı bırakır ve aşamalı olarak ısınan depodan en fazla üç sınırlı tam kurulum denemesi gerçekleştirir; zaman aşımı başarısızlık olarak kalır. İçeriği doğrulanmış bir geri yükleme veya frozen-lockfile kurulumundan sonra kurulum, pnpm'in çalıştırma öncesi gereksiz bağımlılık denetimini devre dışı bırakır: depo kasıtlı olarak plugin'e yerel `node_modules` öğelerini budar; aksi hâlde pnpm bunları eski kabul eder ve parça yayılımı sırasında güvenli olmayan eşzamanlı örtük kurulumlarla onarır. Standart main ön denetimi tek yazıcıdır ve her yenilemede deponun boyutunu ölçerek `pnpm store prune` komutunu yalnızca kullanımdan kaldırılmış paket sürümleri boyutu 8 GiB'nin üzerine çıkardığında çalıştırır. Blacksmith anlık görüntü yayımı, yazıcı işi tamamlandıktan sonra bile eşzamansızdır; bu nedenle yeni bir anahtar veya parmak izinden sonraki ilk çalıştırma soğuk kalabilir; sonraki, içeriği doğrulanmış tam işaretçi geri yüklemeleri kullanıma sunma kanıtıdır. Zorunlu CI işleri ve pull request'ler tek kullanımlık klonlar alır; böylece bağımlılık değişiklikleri yeni diskler, rakip anlık görüntüler veya derlemeleri iptal edebilecek bir önbellek kilidi oluşturmaz.
- Node parçası ve derleme yapıtı işleri ayrıca Node'un taşınabilir disk üzerindeki derleme önbelleğini değişmez Actions önbellekleri aracılığıyla geri yükler. Bağımsız `test` ve `build` ad alanları, yazıcılarının birbirlerinin arşivlerinin yerini almasını önler: zamanlanmış test ısıtıcısı korumalı test çekirdeğinin sahibiyken `build-artifacts`, güvenilir `main` göndermelerinden UTC günü başına en fazla bir korumalı derleme arşivi yayımlayabilir. PR ve sıradan test işleri yalnızca korumalı anlık görüntüleri okur; böylece özellik dalı bayt kodu hiçbir zaman paylaşılan çekirdeğe girmez ve PR trafiği önbellek arşivleri oluşturmaz. Bu, kaynak grafiğinin yalnızca bir kısmı değiştiğinde bile Node tarafından yüklenen düzenleme, derleme araçları ve harici bağımlılıklar için V8 bayt kodunu farklı teslim alma yolları arasında yeniden kullanır. Vitest alt süreçleri, dinamik yapılandırmalarda kapsam etkinleştirilebildiği ve betikler bayt kodundan seri durumdan çıkarıldığında V8 kapsamı kaynak konumu hassasiyetini kaybedebildiği için devralınan derleme önbelleğini devre dışı bırakır.
- Derleme yapıtı işi ayrıca içerik parmak izi alınmış `build-all` adımı çıktılarını kalıcı tutar. CI'nin kendi oluşturduğu plugin SDK bildirimleri, depoya ait TypeScript/JSON kaynak grafiğinin tamamının karmasını alır, kurulu ve oluşturulmuş dizinleri hariç tutar ve `tsdown` öğesi `dist` öğesini temizledikten sonra hem düz bildirimleri hem de paket köprülerini geri yükler. Bu grafiğin dışındaki dokümantasyon, iş akışı, plugin ve diğer değişiklikler bildirim anlık görüntüsünü yeniden kullanabilir; kaynak değişiklikleri ise dışa aktarma geçidi çalışmadan önce bunu yeniden oluşturur.
- Tam bildirim derlemeleri `tsdown` öğesini AI, çalışma alanı paketi ve birleşik gruplara ayırır. Her grup yalnızca bildirimleri önbelleğe alır, ardından bu bildirimleri geri yüklemeden önce çalışma zamanı JavaScript'ini yine de yeniden oluşturur. Bu nedenle çekirdek veya plugin değişiklikleri yalnızca büyük birleşik grafiği geçersiz kılarken çalışma alanı paketi değişiklikleri bağımlı her bildirim grubunu ihtiyatlı biçimde geçersiz kılar. Herkese açık tam derlemeler genellikle değişmez bir Actions önbelleği kullanır; kaba geri yükleme anahtarları kısmi değişiklikleri besler, grup başına içerik parmak izleri eski verileri reddeder ve GitHub'ın önbellek kotası eski nesilleri tahliye eder. Bunun yerine haftalık Node 22 hattı, başarılı `main` çalıştırmalarından sonra 14 günlük bir yapıt yayımlar ve yalnızca değişmez üretici kimliği `main` üzerindeki bu iş akışına çözümlenen yapıtları geri yükler; böylece PR kodunun paylaşılan bir önbelleğe yazmasına izin vermeden kota hareketliliğini önler. Private-QA bildirimleri, önbellek ad alanları gizlilik sınırı olmadığından Actions önbelleklerinde hiçbir zaman kalıcı tutulmaz.
- `check-additional-*`, tamamlayıcı sınır koruması listesini (`scripts/run-additional-boundary-checks.mjs`) istem ağırlıklı bir parçaya (`check-additional-boundaries-a`, Codex istem anlık görüntüsü sapma denetimini içerir) ve kalan şeritler için bir birleşik parçaya (`check-additional-boundaries-bcd`) dağıtır; her biri bağımsız korumaları eşzamanlı çalıştırır ve denetim başına zamanlamaları yazdırır. Paket sınırı derleme/canary çalışması birlikte kalır ve çalışma zamanı topolojisi mimarisi, `build-artifacts` içine gömülü Gateway izleme kapsamından ayrı çalışır.
- 32 vCPU'lu kendi barındırılan derleme çalıştırıcısında Gateway izleme, kanal testleri ve çekirdek destek sınırı parçası, `dist/` ve `dist-runtime/` zaten oluşturulduktan sonra `build-artifacts` içinde birlikte başlar. GitHub tarafından barındırılan geri dönüş çalıştırmaları Gateway izlemeyi seri tutar; böylece düşük çekirdek çekişmesi hazırlık son tarihini tüketemez.

Kabul edildikten sonra standart Linux CI, en fazla 28 eşzamanlı Node test işine ve
daha küçük hızlı/denetim hatları için 12 işe izin verir; Windows ve Android,
çalıştırıcı havuzları daha dar olduğu için ikide kalır. Kompakt tam yapılandırma
grupları 120 dakikalık grup zaman aşımıyla çalışırken include-pattern grupları aynı
sınırlı iş bütçesini paylaşır.

Android CI hem `testPlayDebugUnitTest` hem de `testThirdPartyDebugUnitTest` görevini çalıştırır ve ardından Play hata ayıklama APK'sını oluşturur. Üçüncü taraf çeşidinin ayrı bir kaynak kümesi veya manifestosu yoktur; birim testi hattı, SMS/çağrı günlüğü BuildConfig bayraklarıyla çeşidi derlemeye devam ederken Android ile ilgili her göndermede yinelenen hata ayıklama APK paketleme işinden kaçınır. Her güncel Gradle görevinin bir korumalı yapışkan diski vardır; PR işleri tek kullanımlık klonlar kullanırken korumalı çalıştırmalar içerik adresli Gradle girdilerini yerinde yeniler.

Blacksmith yapışkan disk anahtarları kasıtlı olarak desteklenen çalışma zamanı veya görev boyutlarıyla sınırlandırılır; hiçbir zaman PR numarası, commit, çalıştırma, dal veya bağımlılık karmasıyla değil. Çalışma zamanı dönüşüm ve derleme önbellekleri, değişmez arşivler doğrulanabilir geri yükleme/kaydetme sonuçları sunduğu ve değiştirilebilir anlık görüntü yükseltme hatalarını önlediği için yapışkan diskler yerine Actions önbelleğini kullanır. Yapışkan anahtar sürümü geçişinden sonra `.github/retired-sticky-disks.json` öğesine yalnızca tam eski anahtar, mimari ve bölge kimliklerini ekleyin; aynı boyutlar ve onayla `main` üzerinden `Sticky Disk Cleanup` öğesini çalıştırın, silme işlemini doğrulayın, ardından bu girdileri kaldırın. İş akışı ARM kimliklerini bir ARM çalıştırıcısına yönlendirir, çalıştırıcı-bölge uyuşmazlıklarını reddeder, Blacksmith'in tam anahtar silme eylemini kullanır ve Docker oluşturucu önbelleklerini veya joker öneklerini hiçbir zaman silmez. Actions önbellek arşivleri normal LRU ve hareketsizlik tahliyesini kullanır.

`check-dependencies` parçası üretim Knip bağımlılık, kullanılmayan dosya ve kullanılmayan dışa aktarma denetimlerini çalıştırır. Kullanılmayan dosya koruması, bir PR yeni ve incelenmemiş kullanılmayan bir dosya eklediğinde veya eski bir izin listesi girdisi bıraktığında başarısız olurken Knip'in statik olarak çözümleyemediği kasıtlı dinamik plugin, oluşturulmuş, derleme, canlı test ve paket köprüsü yüzeylerini korur. Kullanılmayan dışa aktarma koruması test desteği dosyalarını hariç tutar ve kullanılmayan her üretim dışa aktarmasında başarısız olur; kasıtlı dinamik tüketiciler `config/knip.config.ts` içinde modellenmelidir. Geçmiş hedefler, sağladıklarında dışa aktarma korumasını çalıştırır; aksi hâlde eski ölü kod geri dönüşlerini korur.

## ClawSweeper etkinlik yönlendirmesi

`.github/workflows/clawsweeper-dispatch.yml`, OpenClaw deposu etkinliğinden ClawSweeper'a uzanan hedef tarafı köprüsüdür. Güvenilmeyen pull request kodunu kullanıma almaz veya çalıştırmaz. İş akışı, `CLAWSWEEPER_APP_PRIVATE_KEY` kaynağından bir GitHub App token'ı oluşturur, ardından kompakt `repository_dispatch` yüklerini `openclaw/clawsweeper` hedefine gönderir.

İş akışının dört hattı vardır:

- `clawsweeper_item`: tam issue ve pull request inceleme istekleri için;
- `clawsweeper_comment`: issue yorumlarındaki açık ClawSweeper komutları için;
- `clawsweeper_commit_review`: `main` push'larındaki commit düzeyi inceleme istekleri için;
- `github_activity`: ClawSweeper aracısının inceleyebileceği genel GitHub etkinliği için.

`github_activity` hattı yalnızca normalleştirilmiş meta verileri iletir: etkinlik türü, eylem, aktör, depo, öğe numarası, URL, başlık, durum ve mevcut olduğunda yorumlar veya incelemelerden kısa alıntılar. Tam webhook gövdesini iletmekten bilerek kaçınır. `openclaw/clawsweeper` içindeki alıcı iş akışı `.github/workflows/github-activity.yml` olup normalleştirilmiş etkinliği ClawSweeper aracısının OpenClaw Gateway hook'una gönderir.

Genel etkinlik gözlem amaçlıdır, varsayılan olarak teslimat amaçlı değildir. ClawSweeper aracısı, isteminde Discord hedefini alır ve yalnızca etkinlik şaşırtıcı, eyleme geçirilebilir, riskli veya operasyonel açıdan yararlı olduğunda `#clawsweeper` hedefine gönderi yayımlamalıdır. Rutin açmalar, düzenlemeler, bot hareketliliği, yinelenen webhook gürültüsü ve normal inceleme trafiği `NO_REPLY` ile sonuçlanmalıdır.

GitHub başlıklarını, yorumlarını, gövdelerini, inceleme metinlerini, dal adlarını ve commit mesajlarını bu yolun tamamında güvenilmeyen veri olarak ele alın. Bunlar, özetleme ve triyaj için girdidir; iş akışı veya aracı çalışma zamanı için talimat değildir.

## Manuel gönderimler

Manuel CI gönderimleri, normal CI ile aynı iş grafiğini çalıştırır ancak Android dışındaki kapsama alınmış her hattı zorunlu olarak etkinleştirir: Linux Node parçaları, paketlenmiş Plugin parçaları, Plugin ve kanal sözleşmesi parçaları, Node 22 uyumluluğu, `check-*`, `check-additional-*`, derlenmiş yapıt duman kontrolleri, dokümantasyon kontrolleri, Python Skills, Windows, macOS, iOS derlemesi ve Control UI/yerel uygulama i18n. Otomatik kaynak PR'leri, aynı PR'de çevrilmiş veya platform tarafından oluşturulmuş çıktı gerektirmeden yerel ayıklama envanterini ve Android/Apple yerelleştirme güvenliğini doğrular. Seri hâle getirilmiş Native App Locale Refresh iş akışı, bu yapıtları yalıtılmış tek bir PR'de yeniden oluşturur ve gerekli kontroller geçtikten sonra tam HEAD için otomatik birleştirmeyi etkinleştirir. Tam yerel eşitlik; oluşturulmuş yapıt PR'leri, manuel CI, Full Release Validation ve sürüm hazırlığı için engelleyici olmaya devam eder. Control UI yerel ayar eşitliği, otomatik PR ve `main` çalıştırmalarında danışma niteliğinde, manuel/sürüm CI'da ise engelleyicidir. Bağımsız manuel CI gönderimleri Android'i yalnızca `include_android=true` ile çalıştırır (`release_gate` girdisi de Android'i zorunlu kılar); tam sürüm şemsiyesi, `include_android=true` geçirerek Android'i etkinleştirir. Plugin ön sürüm statik kontrolleri, yalnızca sürüme özgü `agentic-plugins` parçası, tam uzantı toplu taraması ve Plugin ön sürüm Docker hatları CI kapsamı dışındadır. Docker ön sürüm paketi yalnızca `Full Release Validation`, ayrı `Plugin Prerelease` iş akışını sürüm doğrulama geçidi etkin şekilde gönderdiğinde çalışır.

PR maksimum satır kontrolleri, temel çizgiyi kullanıma alınmış sentetik birleştirme ağacından türetir ve bunun HEAD üst öğesini etkinlik HEAD'iyle karşılaştırarak doğrular. Manuel çalıştırmalar benzersiz bir eşzamanlılık grubu kullanır; böylece sürüm adayı tam paketi, aynı ref üzerindeki başka bir push veya PR çalıştırması tarafından iptal edilmez. İsteğe bağlı `target_ref` girdisi, güvenilir bir çağırıcının seçilen gönderim ref'indeki iş akışı dosyasını kullanırken bu grafiği bir dal, etiket veya tam commit SHA'sı üzerinde çalıştırmasına olanak tanır; maksimum satır temel çizgisi, hedefin o çalıştırma için çözümlenen varsayılan dal HEAD'iyle olan birleştirme tabanıyla karşılaştırılır. `release_gate` girdisi, kapasite nedeniyle bekleyen PR CI için tam SHA'lı bir bakım sorumlusu geri dönüşüdür: `target_ref` değerinin gönderilen dal HEAD'iyle eşleşen tam bir commit SHA'sı olmasını ve `pull_request_number` değerinin birleştirme ağacı doğrulanan açık PR'yi tanımlamasını gerektirir.

```bash
gh workflow run ci.yml --ref release/YYYY.M.PATCH
gh workflow run ci.yml --ref main -f target_ref=<branch-or-sha> -f include_android=true
gh workflow run full-release-validation.yml --ref main -f ref=<branch-or-sha>
```

Gateway extended-stable çalıştırmaları npm ön kontrolünü, Full Release Validation'ı ve `extended-stable/YYYY.M.33` kaynağından Plugin
npm sürümünü çalıştırır; çekirdek yayımlama bu üç
çalıştırma kimliğini ve doğrulama denemesini kullanır. Yayımlama
her çalıştırmayı standart dala ve sürüm SHA'sına bağladığı için `release-ci/*` kanıtı geçersizdir. Etiket,
Gateway imajlarını ve yalnızca `extended-stable*` takma adlarını
yayımlar; bu yol normal orkestratörü ve onun ClawHub, yerel uygulama, GitHub Release, web sitesi
ve özel dist-tag yüzeylerini atlar. Komutlar ve kurtarma için [Aylık Gateway extended-stable
yayınına](/tr/reference/RELEASING#monthly-gateway-extended-stable-publication)
bakın.

## Çalıştırıcılar

| Çalıştırıcı                          | İşler                                                                                                                                                                                                                                                                              |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ubuntu-24.04`                  | `security-fast`, manuel CI gönderimi ve standart olmayan depo geri dönüşleri, QA Smoke toplaması, CodeQL güvenlik ve kalite taramaları, iş akışı tutarlılık kontrolü, etiketleyici, otomatik yanıt, bağımsız Docs iş akışı ve Install Smoke iş akışının tamamı                                |
| `blacksmith-4vcpu-ubuntu-2404`  | `preflight`, `pnpm-store-warmup`, `native-i18n`, QA Smoke CI dışında `checks-fast-core`, Plugin/kanal sözleşmesi parçaları, paketlenmiş/daha düşük ağırlıklı Linux Node parçalarının çoğu, `check-lint` dışında `check-*` hatları, seçili `check-additional-*` parçaları, `check-docs` ve `skills-python` |
| `blacksmith-8vcpu-ubuntu-2404`  | Korunan ağır Linux Node paketleri, sınır/uzantı ağırlıklı `check-additional-*` parçaları ve `android`                                                                                                                                                                             |
| `blacksmith-16vcpu-ubuntu-2404` | Otomatik QA Smoke CI parçaları, CI ve Testbox'ta `build-artifacts` ve `check-lint` (CPU'ya karşı yeterince hassastır; 8 vCPU sağladığı tasarruftan daha pahalıya mal olmuştur)                                                                                                                                  |
| `blacksmith-8vcpu-windows-2025` | `checks-windows`                                                                                                                                                                                                                                                                  |
| `blacksmith-6vcpu-macos-15`     | `openclaw/openclaw` üzerinde `macos-node`; fork'lar `macos-15` seçeneğine geri döner                                                                                                                                                                                                                |
| `blacksmith-12vcpu-macos-26`    | `openclaw/openclaw` üzerinde `macos-swift` ve `ios-build`; fork'lar `macos-26` seçeneğine geri döner                                                                                                                                                                                               |

## Çalıştırıcı kayıt bütçesi

OpenClaw'ın mevcut GitHub çalıştırıcı kayıt grubu, `ghx api rate_limit` içinde 5 dakikada 10.000 kendi kendine barındırılan
çalıştırıcı kaydı bildirir. GitHub
bu grubu değiştirebileceği için her ayarlama geçişinden önce `actions_runner_registration` değerini yeniden kontrol edin.
Sınır, `openclaw` kuruluşundaki tüm Blacksmith çalıştırıcı kayıtları tarafından paylaşılır; dolayısıyla başka bir Blacksmith kurulumu eklemek
yeni bir grup eklemez.

Blacksmith etiketlerini ani yük kontrolü için kıt kaynak olarak değerlendirin. Yalnızca
yönlendirme, bildirim, özetleme, parça seçimi yapan veya kısa CodeQL taramaları çalıştıran işler,
ölçülmüş Blacksmith'e özgü gereksinimleri olmadığı sürece GitHub tarafından barındırılan çalıştırıcılarda
kalmalıdır. Her yeni Blacksmith matrisi, daha büyük `max-parallel` veya yüksek sıklıklı
iş akışı, en kötü durum kayıt sayısını göstermeli ve kuruluş düzeyindeki
hedefi canlı grubun yaklaşık %60'ının altında tutmalıdır. Mevcut 10.000 kayıtlık
grupta bu, eşzamanlı
depolar, yeniden denemeler ve ani yük çakışmaları için pay bırakan 6.000 kayıtlık bir işletim hedefi anlamına gelir.

Değişen hedefli PR planı, yaygın Node test ani yükünü 14 Blacksmith kaydından bire düşürür. Geniş riskli PR'ler 14 kayıtlık kompakt geri dönüşü korur; dolayısıyla en kötü durum artmaz.

Standart depo CI'ı, normal push ve pull request çalıştırmaları için varsayılan çalıştırıcı yolu olarak Blacksmith'i korur. `workflow_dispatch` ve standart olmayan depo çalıştırmaları GitHub tarafından barındırılan çalıştırıcıları kullanır; ancak normal standart çalıştırmalar şu anda Blacksmith kuyruk durumunu yoklamaz veya Blacksmith kullanılamadığında otomatik olarak GitHub tarafından barındırılan etiketlere geri dönmez.

## Yüzey mandalları

Yalnızca küçülmeye izin veren iki bütçe, yapılandırma yüzeyini korur. Her ikisi de
bütçe dosyası aynı PR'de bilinçli olarak güncellenene kadar büyüme durumunda CI'ı
başarısız kılar ve temizlik gerçek sayıyı düşürdüğünde
mandalın aşağı çekilmesini gerektirir.

- `config/env-var-count-budget.txt`, `src/`, `packages/` ve `extensions/`
altındaki üretim kaynağında bulunan farklı `OPENCLAW_*` adlarının sayısını sınırlar
(testler ve QA Lab hariç). `node scripts/check-env-var-count.mjs` tarafından kontrol edilir.
  Ortam değişkenlerini kaldırırken: aynı PR'de sayıyı düşürün. Yeni bir tane eklemek
  yapılandırma yüzeyi kararıdır — bunu PR gövdesinde gerekçelendirin.
- `docs/.generated/config-baseline.counts.json`, tür başına
  (çekirdek/kanal/Plugin) `openclaw.json` şema girdisi sayılarını sınırlar. `pnpm config:docs:check` tarafından
  kontrol edilir; herhangi bir şema değişikliğinden sonra `pnpm config:docs:gen` ile yeniden oluşturun.

## Yerel eşdeğerler

```bash
pnpm changed:lanes                            # origin/main...HEAD için yerel değişen şerit sınıflandırıcısını incele
pnpm check:changed                            # akıllı yerel denetim kapısı: sınır şeridine göre değişen biçimlendirme/tür denetimi/lint/korumalar
pnpm check                                    # hızlı yerel kapı: üretim tsgo + parçalı lint + paralel hızlı korumalar
pnpm check:test-types
pnpm check:timed                              # aşama başına zamanlamalarla aynı kapı
pnpm build:strict-smoke
pnpm check:architecture
pnpm test:gateway:watch-regression
OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1 node scripts/run-vitest.mjs run --config test/vitest/vitest.tui-pty.config.ts
pnpm test                                     # vitest testleri
pnpm test:changed                             # düşük maliyetli, akıllı ve değişikliğe dayalı Vitest hedefleri
pnpm test:ui                                  # Control UI birim/tarayıcı paketi
pnpm ui:i18n:check                            # oluşturulan Control UI yerel ayar eşliği (sürüm kapısı)
pnpm native:i18n:baseline                     # kaynak tarafından yönetilen yerel ayıklama envanterini güncelle
pnpm native:i18n:verify                       # kaynak envanteri + Android/Apple yerelleştirme güvenliği
pnpm native:i18n:check                        # katı çevrilmiş/platform tarafından oluşturulan eşlik (sürüm kapısı)
pnpm test:channels
pnpm test:contracts:channels
pnpm check:docs                               # belge biçimi + lint + bozuk bağlantılar
pnpm build                                    # CI yapıtı/duman denetimleri önemli olduğunda dist'i derle
pnpm ios:build                                # iOS uygulama projesini oluştur ve derle
pnpm ci:timings                               # en son origin/main gönderim CI çalışmasını özetle
pnpm ci:timings:recent                        # yakın tarihli başarılı ana CI çalışmalarını karşılaştır
node scripts/ci-run-timings.mjs <run-id>      # toplam süreyi, kuyruk süresini ve en yavaş işleri özetle
node scripts/ci-run-timings.mjs --latest-main # sorun/yorum gürültüsünü yok say ve origin/main gönderim CI'ını seç
node scripts/ci-run-timings.mjs --recent 10   # yakın tarihli başarılı ana CI çalışmalarını karşılaştır
pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json
pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json
pnpm test:startup:memory
pnpm test:extensions:memory -- --json .artifacts/openclaw-performance/source/mock-provider/extension-memory.json
pnpm perf:kova:summary --report .artifacts/kova/reports/mock-provider/report.json --output .artifacts/kova/summary.md
```

## OpenClaw Performansı

`OpenClaw Performance`, ürün/çalışma zamanı performans iş akışıdır. Her gün `main` üzerinde çalışır ve manuel olarak başlatılabilir:

```bash
gh workflow run openclaw-performance.yml --ref main -f profile=diagnostic -f repeat=3
gh workflow run openclaw-performance.yml --ref main -f profile=smoke -f repeat=1 -f deep_profile=true -f live_openai_candidate=true
gh workflow run openclaw-performance.yml --ref main -f target_ref=v2026.5.2 -f profile=diagnostic -f repeat=3
```

Manuel başlatma normalde iş akışı referansını karşılaştırmalı teste tabi tutar. Geçerli iş akışı uygulamasıyla bir sürüm etiketini veya başka bir dalı karşılaştırmalı teste tabi tutmak için `target_ref` ayarlayın. Yayımlanan rapor yolları ve en son işaretçiler, test edilen referansa göre anahtarlanır ve her `index.md`; test edilen referansı/SHA'yı, iş akışı referansını/SHA'yı, Kova referansını, profili, şerit kimlik doğrulama modunu, modeli, tekrar sayısını ve senaryo filtrelerini kaydeder.

İş akışı, OCM'yi sabitlenmiş bir sürümden ve Kova'yı sabitlenmiş `kova_ref` girdisindeki `openclaw/Kova` kaynağından yükler, ardından üç şerit çalıştırır:

- `mock-provider`: Belirlenimci sahte OpenAI uyumlu kimlik doğrulamasıyla yerel derleme çalışma zamanına karşı Kova tanılama senaryoları.
- `mock-deep-profile`: Başlatma, gateway ve ajan dönüşü yoğun noktaları için CPU/heap/iz profili çıkarma. Zamanlamaya göre veya `deep_profile=true` ile başlatıldığında çalışır.
- `live-openai-candidate`: Gerçek bir OpenAI `openai/gpt-5.6-luna` ajan dönüşü; `OPENAI_API_KEY` kullanılamadığında atlanır. Zamanlamaya göre veya `live_openai_candidate=true` ile başlatıldığında çalışır.

Sahte sağlayıcı şeridi, Kova geçişinden sonra OpenClaw'a özgü kaynak problarını da çalıştırır: varsayılan, atlanan kanal, dahili kanca ve elli Plugin'li başlatma durumlarında gateway önyükleme zamanlaması ve belleği; paketlenmiş Plugin içe aktarma RSS'i, yinelenen sahte OpenAI `channel-chat-baseline` merhaba döngüleri, önyüklenmiş gateway'e karşı CLI başlatma komutları ve SQLite durum duman performansı probu. Test edilen referans için daha önce yayımlanmış sahte sağlayıcı kaynak raporu kullanılabiliyorsa kaynak özeti, geçerli RSS ve heap değerlerini bu temel değerle karşılaştırır ve büyük RSS artışlarını `watch` olarak işaretler. Kaynak probu Markdown özeti, rapor paketindeki `source/index.md` konumunda bulunur; ham JSON da yanındadır.

Her şerit; CPU, heap, iz ve sıkıştırılmış tanılama paketleri dâhil eksiksiz GitHub yapıtını yükler. Ayrı bir yayımcı işi bu yapıtları indirip doğrular, ardından yalnızca `openclaw/clawgrit-reports` içerikleriyle kapsamlandırılmış kısa ömürlü bir ClawSweeper GitHub App belirteci oluşturur ve bunu yalnızca Git gönderim adımına iletir. `report.json`, `report.md`, `index.md`, kaynak probu yapıtları ve paket meta verileri/sağlama toplamlarını `openclaw-performance/<tested-ref>/<run-id>-<attempt>/<lane>/` altında kaydeder; tam tanılama arşivi bağlantılı Actions yapıtında kalır. Yayımcı, gönderim girişiminden önce 50 MB üzerindeki tüm rapor dosyalarını reddeder. Geçerli test edilen referans işaretçisi `openclaw-performance/<tested-ref>/latest-<lane>.json` değeridir. Zamanlanmış çalışmalar ve `profile=release` başlatmaları, uygulama belirteci oluşturma veya rapor yayımlama başarısız olursa başarısız olur. Sürüm dışı manuel başlatmalarda yayımlama öneri niteliğinde kalır ve kimlik doğrulama veya yayımlama başarısız olduğunda GitHub yapıtları korunur. Önceki kaynak temel değeri, herkese açık rapor deposundan anonim olarak alınır; dolayısıyla temel değerin başarıyla alınması yayımcı kimlik doğrulamasını kanıtlamaz.

## Tam Sürüm Doğrulaması

`Full Release Validation`, "sürümden önce her şeyi çalıştır" için manuel şemsiye iş akışıdır. Bir dalı, etiketi veya tam commit SHA'sını kabul eder; manuel `CI` iş akışını bu hedefle (Android dâhil) başlatır, yalnızca sürüme özgü Plugin/paket/statik/Docker kanıtı için `Plugin Prerelease` iş akışını başlatır, hedef SHA'ya karşı `OpenClaw Performance` iş akışını başlatır ve yükleme duman testi, paket kabulü, işletim sistemleri arası paket denetimleri, QA Lab eşliği, Matrix, Telegram ve kapılı Discord, WhatsApp ve Slack şeritleri için `OpenClaw Release Checks` iş akışını başlatır (öneri niteliğindeki olgunluk puan kartı oluşturma, `run_maturity_scorecard` aracılığıyla isteğe bağlıdır). Kararlı ve tam profiller her zaman kapsamlı canlı/E2E ve Docker sürüm yolu dayanıklılık kapsamını içerir; beta profili `run_release_soak=true` ile bunu etkinleştirebilir. Standart paket Telegram E2E, Package Acceptance içinde çalışır; dolayısıyla tam bir aday yinelenen bir canlı yoklayıcı başlatmaz. Yayımlamadan sonra, yeniden derleme yapmadan gönderilmiş npm paketini sürüm denetimleri, Package Acceptance, Docker, işletim sistemleri arası denetimler ve Telegram genelinde yeniden kullanmak için `release_package_spec` iletin. `npm_telegram_package_spec` değerini yalnızca yayımlanmış pakete yönelik odaklı bir Telegram yeniden çalıştırması için kullanın. Codex Plugin canlı paket şeridi varsayılan olarak aynı seçili durumu kullanır: yayımlanan `release_package_spec=openclaw@<tag>`, `codex_plugin_spec=npm:@openclaw/codex@<tag>` değerini türetirken SHA/yapıt çalışmaları seçili referanstan `extensions/codex` paketler. `npm:`, `npm-pack:` veya `git:` belirtimleri gibi özel Plugin kaynakları için `codex_plugin_spec` değerini açıkça ayarlayın. Canlı ajan kanıtı görünür ilerleme gönderir, rastgeleleştirilmiş çalışma alanı okumaları ve tam bir yapıt yazımı boyunca devam eder, ardından tamamlanma bildirimi gönderir.

Aşama matrisi, tam iş akışı işi adları, profil farklılıkları, yapıtlar ve
odaklı yeniden çalıştırma tanıtıcıları için [Tam sürüm doğrulaması](/tr/reference/full-release-validation)
sayfasına bakın.

`OpenClaw Release Publish`, durumu değiştiren manuel sürüm iş akışıdır. Sürüm etiketi
oluşturulduktan ve OpenClaw npm ön denetimi başarıyla tamamlandıktan sonra
güvenilir `main` kaynağından normal beta ve kararlı yayımları başlatın
(ön denetim, denetimleri arasında `pnpm plugins:sync:check` çalıştırır). Etiket,
`release/YYYY.M.PATCH` üzerindeki bir commit dâhil olmak üzere tam sürüm commit'ini
seçmeye devam eder; Tideclaw alfa yayımları eşleşen alfa dallarını kullanmayı
sürdürür. Kaydedilmiş `preflight_run_id` ile başarılı bir
`full_release_validation_run_id` ve onun tam
`full_release_validation_run_attempt` değerini gerektirir; yayımlanabilir tüm
Plugin paketleri için `Plugin NPM Release` iş akışını başlatır, aynı
sürüm SHA'sı için `Plugin ClawHub Release` iş akışını başlatır ve ancak bundan
sonra `OpenClaw NPM Release` iş akışını başlatır. Kararlı yayımlama ayrıca tam bir
`windows_node_tag` gerektirir; iş akışı herhangi bir alt yayımlama işleminden
önce Windows kaynak sürümünü doğrular ve x64/ARM64 yükleyicilerini aday tarafından
onaylanan `windows_node_installer_digests` girdisiyle karşılaştırır, ardından GitHub sürüm
taslağını yayımlamadan önce aynı sabitlenmiş yükleyici özetlerini, tam eşlikçi
yapıtı ve sağlama toplamı sözleşmesini yükseltir ve doğrular.
Odaklı, yalnızca Plugin'e yönelik onarımlar, boş olmayan bir paket listesiyle
`plugin_publish_scope=selected` kullanır. Yalnızca Plugin'e yönelik `all-publishable`
çalışmaları, çekirdek yayımlamayla aynı değişmez npm ön denetimini ve Tam Sürüm
Doğrulaması kanıtını gerektirir.

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH-beta.N \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

Hızla değişen bir dalda sabitlenmiş commit kanıtı için
`gh workflow run ... --ref main -f ref=<sha>` yerine yardımcıyı kullanın:

```bash
pnpm ci:full-release --sha <full-sha>
```

GitHub iş akışı başlatma referansları ham commit SHA'ları değil, dallar veya
etiketler olmalıdır. Yardımcı, güvenilir bir `main` iş akışı SHA'sında
geçici bir `release-ci/<sha>-...` dalı gönderir, istenen hedef SHA'yı iş akışının
`ref` girdisi üzerinden iletir, mevcut olduğunda katı tam hedef
kanıtını yeniden kullanır, her alt iş akışının `headSha` değerinin
güvenilir iş akışı SHA'sıyla eşleştiğini doğrular ve çalışma tamamlandığında
geçici dalı siler. Yeni doğrulamayı zorlamak için `-f reuse_evidence=false` iletin.
Şemsiye doğrulayıcı, herhangi bir alt iş akışı farklı bir iş akışı SHA'sında
çalıştıysa da başarısız olur.

`release_profile`, sürüm denetimlerine iletilen canlı/sağlayıcı kapsamını
denetler. Manuel sürüm iş akışları varsayılan olarak `stable` kullanır;
geniş, öneri niteliğindeki sağlayıcı/medya matrisini bilinçli olarak istediğinizde
yalnızca `full` kullanın. Kararlı ve tam sürüm denetimleri her zaman
kapsamlı canlı/E2E ve Docker sürüm yolu dayanıklılık testini çalıştırır; beta
profili `run_release_soak=true` ile bunu etkinleştirebilir.

- `beta`, en hızlı ve sürüm açısından kritik OpenAI/çekirdek şeritlerini korur.
- `stable`, kararlı sağlayıcı/arka uç kümesini ekler.
- `full`, geniş ve öneri niteliğindeki sağlayıcı/medya matrisini çalıştırır.

Şemsiye, başlatılan alt çalışma kimliklerini kaydeder ve son `Verify full validation` işi, geçerli alt çalışma sonuçlarını yeniden denetleyip her alt çalışma için en yavaş işler tablolarını ekler. Bir alt iş akışı yeniden çalıştırılarak başarılı duruma gelirse şemsiye sonucunu ve zamanlama özetini yenilemek için yalnızca üst doğrulayıcı işini yeniden çalıştırın.

Kurtarma için hem `Full Release Validation` hem de `OpenClaw Release Checks`, `rerun_group` kabul eder. Bir sürüm adayı için `all`, yalnızca normal tam CI alt iş akışı için `ci`, yalnızca plugin ön sürüm alt iş akışı için `plugin-prerelease`, yalnızca OpenClaw Performance alt iş akışı için `performance`, tüm sürüm alt iş akışları için `release-checks` veya şemsiye iş akışında daha dar bir grup kullanın: `install-smoke`, `cross-os`, `live-e2e`, `package`, `qa`, `qa-parity`, `qa-live` ya da `npm-telegram`. Bu, odaklı bir düzeltmeden sonra başarısız olan bir sürüm kutusunun yeniden çalıştırılmasını sınırlı tutar. Başarısız olan tek bir işletim sistemleri arası hat için `rerun_group=cross-os` ile `cross_os_suite_filter` öğelerini birleştirin; örneğin `windows/packaged-upgrade`. Uzun işletim sistemleri arası komutlar Heartbeat satırları üretir ve paketlenmiş yükseltme özetleri aşama başına süreleri içerir. Seçili Matrix ve Telegram QA hatları ile temel çalışma zamanı çifti araç kapsamı geçidi normal sürüm doğrulamasını engeller. QA eşdeğerliği, çalışma zamanı eşdeğerliği ve geçitli Discord, WhatsApp ve Slack canlı hatları öneri niteliğindedir.

`OpenClaw Release Checks`, seçilen başvuruyu bir kez `release-package-under-test` tarball'ına çözümlemek için güvenilen iş akışı başvurusunu kullanır; ardından bu yapıtı işletim sistemleri arası kontrollere ve Package Acceptance'a, ayrıca bekletme kapsamı çalıştırıldığında canlı/E2E sürüm yolu Docker iş akışına iletir. Bu, paket baytlarını sürüm kutuları arasında tutarlı tutar ve aynı adayın birden fazla alt işte yeniden paketlenmesini önler. Codex npm-plugin canlı hattı için sürüm kontrolleri ya `release_package_spec` üzerinden türetilmiş eşleşen bir yayımlanmış plugin belirtimi ya da operatörün sağladığı `codex_plugin_spec` değerini iletir; ayrıca Docker betiğinin seçilen çalışma kopyasındaki Codex plugin'ini paketlemesi için girdiyi boş bırakabilir.

`ref=main` ve `rerun_group=all` için yinelenen `Full Release Validation` çalıştırmaları
eski şemsiye iş akışının yerini alır. Üst izleyici, üst iş akışı iptal
edildiğinde daha önce başlattığı tüm alt iş akışlarını iptal eder; böylece daha
yeni ana dal doğrulaması, eskimiş iki saatlik bir sürüm kontrolü çalıştırmasının
arkasında beklemez. Sürüm dalı/etiketi doğrulaması ve odaklı yeniden çalıştırma
grupları `cancel-in-progress: false` değerini korur.

## Canlı ve E2E parçaları

Sürümün canlı/E2E alt iş akışı geniş yerel `pnpm test:live` kapsamını korur, ancak bunu tek bir sıralı iş yerine `scripts/test-live-shard.mjs` aracılığıyla adlandırılmış parçalar olarak çalıştırır:

- `native-live-src-agents` ve `native-live-src-agents-zai-coding`
- `native-live-src-gateway-core`
- sağlayıcıya göre filtrelenmiş `native-live-src-gateway-profiles` işleri
- `native-live-src-gateway-backends`
- `native-live-src-infra`
- `native-live-test`
- `native-live-extensions-a-k`
- `native-live-extensions-l-n`
- `native-live-extensions-moonshot`
- `native-live-extensions-openai`
- `native-live-extensions-o-z-other`
- `native-live-extensions-xai`
- ayrılmış medya ses/video parçaları ve sağlayıcıya göre filtrelenmiş müzik parçaları

Bu, aynı dosya kapsamını korurken yavaş canlı sağlayıcı hatalarının yeniden çalıştırılmasını ve tanılanmasını kolaylaştırır. Toplu `native-live-src-gateway`, `native-live-extensions-o-z`, `native-live-extensions-media` ve `native-live-extensions-media-music` parça adları, tek seferlik manuel yeniden çalıştırmalar için geçerli kalır.

Yerel canlı medya parçaları, `Live Media Runner Image` iş akışının oluşturduğu `ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04` içinde çalışır. Bu imaj `ffmpeg` ve `ffprobe` öğelerini önceden yükler; medya işleri kurulumdan önce yalnızca ikili dosyaları doğrular. Docker destekli canlı paketleri normal Blacksmith çalıştırıcılarında tutun; kapsayıcı işleri, iç içe Docker testlerini başlatmak için yanlış yerdir.

Docker destekli canlı model/arka uç parçaları, seçilen her commit için ayrı ve paylaşılan bir `ghcr.io/openclaw/openclaw-live-test:<sha>-<extensions>` imajı kullanır. Canlı sürüm iş akışı bu imajı bir kez oluşturup gönderir; ardından Docker canlı model, sağlayıcıya göre parçalanmış Gateway, CLI arka ucu, ACP bağlama ve Codex test düzeneği parçaları `OPENCLAW_SKIP_DOCKER_BUILD=1` ile çalışır. Gateway Docker parçaları, takılan bir kapsayıcının veya temizleme yolunun tüm sürüm kontrolü bütçesini tüketmek yerine hızla başarısız olması için iş akışı iş zaman aşımının altında açık betik düzeyi `timeout` sınırları taşır. Bu parçalar tam kaynak Docker hedefini bağımsız olarak yeniden oluşturursa sürüm çalıştırması yanlış yapılandırılmıştır ve yinelenen imaj oluşturmaları için geçen süreyi boşa harcar.

## Package Acceptance

Soru "bu kurulabilir OpenClaw paketi bir ürün olarak çalışıyor mu?" olduğunda `Package Acceptance` kullanın. Bu, normal CI'dan farklıdır: normal CI kaynak ağacını doğrularken paket kabulü, tek bir tarball'ı kullanıcıların kurulum veya güncelleme sonrasında kullandığı aynı Docker E2E test düzeneği üzerinden doğrular.

### İşler

1. `resolve_package`, `workflow_ref` öğesini kullanıma alır, tek bir paket adayını çözümler, `.artifacts/docker-e2e-package/openclaw-current.tgz` ve `.artifacts/docker-e2e-package/package-candidate.json` dosyalarını yazar, ikisini de `package-under-test` yapıtı olarak yükler ve GitHub adımı özetinde kaynağı, iş akışı başvurusunu, paket başvurusunu, sürümü, SHA-256 değerini ve profili yazdırır.
2. `package_integrity`, `package-under-test` yapıtını indirir ve `scripts/check-openclaw-package-tarball.mjs` ile herkese açık paket tarball sözleşmesini zorunlu kılar.
3. `docker_acceptance`, çözümlenmiş paket kaynak SHA'sı (`workflow_ref` değerine geri dönerek) ve `package_artifact_name=package-under-test` ile `openclaw-live-and-e2e-checks-reusable.yml` öğesini çağırır. Yeniden kullanılabilir iş akışı bu yapıtı indirir, tarball envanterini doğrular, gerektiğinde paket özeti Docker imajlarını hazırlar ve seçilen Docker hatlarını iş akışı çalışma kopyasını paketlemek yerine bu paket üzerinde çalıştırır. Bir profil birden fazla hedefli `docker_lanes` seçtiğinde, yeniden kullanılabilir iş akışı paketi ve paylaşılan imajları bir kez hazırlar; ardından bu hatları benzersiz yapıtlara sahip paralel hedefli Docker işleri olarak dağıtır.
4. `package_telegram`, isteğe bağlı olarak `NPM Telegram Beta E2E` öğesini çağırır. `telegram_mode`, `none` olmadığında çalışır ve Package Acceptance bir paket çözümlediyse aynı `package-under-test` yapıtını kurar; bağımsız Telegram gönderimi yine de yayımlanmış bir npm belirtimini kurabilir.
5. `summary`, paket çözümleme, bütünlük, Docker kabulü veya isteğe bağlı Telegram hattı başarısız olursa iş akışını başarısız kılar. `advisory` girdisi, öneri niteliğindeki çağıranlar için kabul hatalarını uyarılara indirger.

### Aday kaynakları

- `source=npm`, yalnızca `openclaw@extended-stable`, `openclaw@beta`, `openclaw@latest` veya `openclaw@2026.4.27-beta.2` gibi tam bir OpenClaw sürümünü kabul eder. Bunu yayımlanmış genişletilmiş kararlı, ön sürüm veya kararlı sürüm kabulü için kullanın.
- `source=ref`, güvenilen bir `package_ref` dalını, etiketini veya tam commit SHA'sını paketler. Çözümleyici OpenClaw dallarını/etiketlerini getirir, seçilen commit'in depo dalı geçmişinden veya bir sürüm etiketinden erişilebilir olduğunu doğrular, bağımlılıkları ayrılmış bir çalışma ağacında kurar ve bunu `scripts/package-openclaw-for-docker.mjs` ile paketler.
- `source=url`, herkese açık bir HTTPS `.tgz` öğesini indirir; `package_sha256` zorunludur. Bu yol URL kimlik bilgilerini, varsayılan olmayan HTTPS bağlantı noktalarını, özel/dahili/özel amaçlı ana bilgisayar adlarını veya çözümlenmiş IP'leri ve aynı genel güvenlik politikası dışındaki yönlendirmeleri reddeder.
- `source=trusted-url`, `.github/package-trusted-sources.json` içindeki adlandırılmış bir güvenilen kaynak politikasından bir HTTPS `.tgz` öğesi indirir; `package_sha256` ve `trusted_source_id` zorunludur. Bunu yalnızca yapılandırılmış ana bilgisayarlara, bağlantı noktalarına, yol öneklerine, yönlendirme ana bilgisayarlarına veya özel ağ çözümlemesine ihtiyaç duyan, bakımcıların sahip olduğu kurumsal aynalar ya da özel paket depoları için kullanın. Politika taşıyıcı kimlik doğrulaması tanımlıyorsa iş akışı sabit `OPENCLAW_TRUSTED_PACKAGE_TOKEN` gizli bilgisini kullanır; URL içine gömülü kimlik bilgileri yine de reddedilir.
- `source=artifact`, `artifact_run_id` ve `artifact_name` üzerinden bir `.tgz` indirir; `package_sha256` isteğe bağlıdır ancak dışarıdan paylaşılan yapıtlar için sağlanmalıdır.

`workflow_ref` ile `package_ref` öğelerini ayrı tutun. `workflow_ref`, testi çalıştıran güvenilen iş akışı/test düzeneği kodudur. `package_ref`, `source=ref` olduğunda paketlenen kaynak commit'idir. Bu, güncel test düzeneğinin eski iş akışı mantığını çalıştırmadan daha eski güvenilen kaynak commit'lerini doğrulamasını sağlar.

### Paket profilleri

- `smoke` — `npm-onboard-channel-agent`, `gateway-network`, `config-reload`
- `package` — `npm-onboard-channel-agent`, `doctor-switch`, `update-channel-switch`, `skill-install`, `update-corrupt-plugin`, `upgrade-survivor`, `published-upgrade-survivor`, `root-managed-vps-upgrade`, `update-restart-auth`, `plugins-offline`, `plugin-update`
- `product` — `plugins-offline` yerine canlı `plugins` kapsamına sahip `package` kümesi, ayrıca `mcp-channels`, `cron-mcp-cleanup`, `openai-web-search-minimal`, `openwebui`
- `full` — OpenWebUI ile tam Docker sürüm yolu parçaları
- `custom` — tam olarak `docker_lanes`; `suite_profile=custom` olduğunda zorunludur

`package` profili çevrimdışı plugin kapsamı kullanır; böylece yayımlanmış paket doğrulaması canlı ClawHub kullanılabilirliğine bağlı olmaz. İsteğe bağlı Telegram hattı `NPM Telegram Beta E2E` içindeki `package-under-test` yapıtını yeniden kullanırken yayımlanmış npm belirtimi yolu bağımsız gönderimler için korunur.

Yerel komutlar, Docker hatları, Package Acceptance girdileri, sürüm varsayılanları ve hata triyajı dâhil olmak üzere özel güncelleme ve plugin testi politikası için
bkz. [Güncellemeleri ve plugin'leri test etme](/tr/help/testing-updates-plugins).

Sürüm kontrolleri Package Acceptance'ı `source=artifact`, hazırlanmış sürüm paketi yapıtı, `suite_profile=custom`, `docker_lanes='doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape'` ve `telegram_mode=mock-openai` ile çağırır. Bu, paket taşıma, güncelleme, canlı ClawHub Skills kurulumu, eski plugin bağımlılığı temizliği, yapılandırılmış plugin kurulum onarımı, çevrimdışı plugin, plugin güncellemesi ve Telegram kanıtının aynı çözümlenmiş paket tarball'ında kalmasını sağlar. Aynı matrisi yeniden oluşturmadan yayımlanmış npm paketi üzerinde çalıştırmak için bir beta yayımladıktan sonra Full Release Validation veya OpenClaw Release Checks üzerinde `release_package_spec` ayarlayın; yalnızca Package Acceptance'ın sürüm doğrulamasının geri kalanından farklı bir pakete ihtiyacı olduğunda `package_acceptance_package_spec` ayarlayın. İşletim sistemleri arası sürüm kontrolleri işletim sistemine özgü ilk kullanım, yükleyici ve platform davranışını kapsamaya devam eder; paket/güncelleme ürün doğrulaması Package Acceptance ile başlamalıdır.

`published-upgrade-survivor` Docker hattı, engelleyici sürüm yolunda çalıştırma başına yayımlanmış tek bir paket temelini doğrular. Package Acceptance'ta çözümlenen `package-under-test` tarball'ı her zaman adaydır ve `published_upgrade_survivor_baseline`, varsayılan olarak `openclaw@latest` olan yedek yayımlanmış temeli seçer; başarısız hat yeniden çalıştırma komutları bu temeli korur. `run_release_soak=true` veya `release_profile=full` ile Full Release Validation; en son dört kararlı npm sürümüne, sabitlenmiş plugin uyumluluk sınırı sürümlerine ve Feishu yapılandırması, korunmuş önyükleme/kişilik dosyaları, yapılandırılmış OpenClaw plugin kurulumları, yaklaşık işaretli günlük yolları ve eski plugin bağımlılığı kökleri için sorun biçimli fikstürlere genişlemek üzere `published_upgrade_survivor_baselines='last-stable-4 2026.4.23 2026.5.2 2026.4.15'` ve `published_upgrade_survivor_scenarios=reported-issues` değerlerini ayarlar. Çok temelli yayımlanmış yükseltme sağ kalım seçimleri, temele göre ayrı hedefli Docker çalıştırıcı işlerine parçalanır. Ayrı `Update Migration` iş akışı; soru normal Full Release CI kapsamı değil, kapsamlı yayımlanmış güncelleme temizliği olduğunda `all-since-2026.4.23` temelleri ve `plugin-deps-cleanup` senaryolarıyla `update-migration` Docker hattını kullanır. Yerel toplu çalıştırmalar `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` ile tam paket belirtimleri iletebilir, `openclaw@2026.4.15` gibi `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC` ile tek bir hattı koruyabilir veya senaryo matrisi için `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS` ayarlayabilir. Yayımlanmış hat, temeli yerleşik bir `openclaw config set` komut tarifiyle yapılandırır, tarif adımlarını `summary.json` içinde kaydeder ve Gateway başlatıldıktan sonra `/healthz`, `/readyz` ile RPC durumunu yoklar. Windows paketlenmiş ve yükleyici temiz kurulum hatları ayrıca kurulu bir paketin ham mutlak Windows yolundan bir tarayıcı denetimi geçersiz kılmasını içe aktarabildiğini doğrular. OpenAI işletim sistemleri arası agent dönüşü duman testi, ayarlandığında varsayılan olarak `OPENCLAW_CROSS_OS_OPENAI_MODEL`, aksi takdirde `openai/gpt-5.6-luna` kullanır; böylece kurulum ve Gateway kanıtı daha düşük maliyetli GPT-5.6 test katmanını kullanır.

### Eski uyumluluk dönemleri

Paket Kabulü, daha önce yayımlanmış paketler için sınırlandırılmış eski uyumluluk pencerelerine sahiptir. `2026.4.25-beta.*` dahil olmak üzere `2026.4.25` sürümüne kadarki paketler uyumluluk yolunu kullanabilir:

- `dist/postinstall-inventory.json` içindeki bilinen özel QA girdileri, tarball'dan çıkarılmış dosyalara işaret edebilir;
- paket bu bayrağı sunmadığında `doctor-switch`, `gateway install --wrapper` kalıcılık alt durumunu atlayabilir;
- `update-channel-switch`, tarball'dan türetilen sahte git fikstüründeki eksik pnpm `patchedDependencies` öğelerini ayıklayabilir ve eksik kalıcı `update.channel` öğelerini günlüğe kaydedebilir;
- plugin duman testleri eski kurulum kaydı konumlarını okuyabilir veya eksik pazar yeri kurulum kaydı kalıcılığını kabul edebilir;
- `plugin-update`, kurulum kaydının ve yeniden kurmama davranışının değişmeden kalmasını hâlâ zorunlu tutarken yapılandırma meta verisi geçişine izin verebilir.

Yayımlanmış `2026.4.26` paketi, daha önce dağıtılmış yerel derleme meta verisi damga dosyaları için de uyarı verebilir ve `2026.5.20` sürümüne kadarki paketler, `npm-shrinkwrap.json` eksik olduğunda başarısız olmak yerine uyarı verebilir. Daha sonraki paketler modern sözleşmeleri karşılamalıdır; aynı koşullar uyarı vermek veya atlamak yerine başarısız olur.

### Örnekler

```bash
# Geçerli beta paketini ürün düzeyinde kapsamla doğrulayın.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai

# Yayımlanmış extended-stable paketini paket kapsamıyla doğrulayın.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@extended-stable \
  -f suite_profile=package \
  -f telegram_mode=mock-openai

# Bir sürüm dalını geçerli test düzeneğiyle paketleyip doğrulayın.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=ref \
  -f package_ref=release/YYYY.M.PATCH \
  -f suite_profile=package \
  -f telegram_mode=mock-openai

# Bir tarball URL'sini doğrulayın. source=url için SHA-256 zorunludur.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=url \
  -f package_url=https://example.com/openclaw-current.tgz \
  -f package_sha256=<64-char-sha256> \
  -f suite_profile=smoke

# Adlandırılmış güvenilir bir özel yansıtıcı ilkesinden gelen tarball'ı doğrulayın.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-current.tgz \
  -f package_sha256=<64-char-sha256> \
  -f suite_profile=smoke

# Başka bir Actions çalıştırması tarafından yüklenen tarball'ı yeniden kullanın.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=package-under-test \
  -f suite_profile=custom \
  -f docker_lanes='install-e2e plugin-update'
```

Başarısız bir paket kabulü çalıştırmasında hata ayıklarken paket kaynağını, sürümünü ve SHA-256 değerini doğrulamak için `resolve_package` özetinden başlayın. Ardından `docker_acceptance` alt çalıştırmasını ve Docker yapıtlarını inceleyin: `.artifacts/docker-tests/**/summary.json`, `failures.json`, hat günlükleri, aşama zamanlamaları ve yeniden çalıştırma komutları. Tam sürüm doğrulamasını yeniden çalıştırmak yerine başarısız paket profilini veya tam Docker hatlarını yeniden çalıştırmayı tercih edin.

## Kurulum duman testi

`Install Smoke` iş akışı artık pull request'lerde veya `main` gönderimlerinde çalışmaz. Gecelik/manuel sarmalayıcısı ve sürüm doğrulaması, salt okunur `install-smoke-reusable.yml` çekirdeğini çağırır ve her çalıştırma GitHub tarafından barındırılan çalıştırıcılarda tam kurulum duman testi yolunu izler:

- Kök Dockerfile duman testi görüntüsü hedef SHA başına bir kez oluşturulur, değiştirilemez bir yapıtta iş akışı revizyonuna ve üretici denemesine bağlanır; ardından CLI duman testi, aracıların paylaşılan çalışma alanını sildiği CLI duman testi, konteyner Gateway ağ E2E testi ve paketlenmiş `matrix` plugin derleme argümanı duman testi tarafından yüklenir. Plugin duman testi, çalışma zamanı bağımlılığı kurulumunun yansıtılmasını ve plugin'in giriş noktası dışına çıkma tanılamaları olmadan yüklendiğini doğrular.
- QR paket kurulumu ve yükleyici/güncelleme Docker duman testleri (Rocky Linux yükleyici hatları ve yapılandırılabilir bir `update_baseline_version` npm temel çizgisine karşı güncelleme hattı dahil), yükleyici çalışmasının kök görüntü duman testlerinin arkasında beklememesi için ayrı işler olarak çalışır.

Yavaş Bun genel kurulum görüntü sağlayıcısı duman testi ayrıca `run_bun_global_install_smoke` tarafından denetlenir. Gecelik zamanlamada çalışır, sürüm kontrollerinden gelen iş akışı çağrılarında varsayılan olarak açıktır ve manuel `Install Smoke` tetiklemeleri bunu etkinleştirebilir. Normal PR CI, Node ile ilgili değişiklikler için hızlı Bun başlatıcı regresyon hattını çalıştırmaya devam eder. QR ve yükleyici Docker testleri kendi kurulum odaklı Dockerfile'larını korur.

## Yerel Docker E2E

`pnpm test:docker:all`, paylaşılan tek bir canlı test görüntüsünü önceden oluşturur, OpenClaw'ı bir npm tarball'ı olarak bir kez paketler ve iki paylaşılan `scripts/e2e/Dockerfile` görüntüsü oluşturur:

- yükleyici/güncelleme/plugin bağımlılığı hatları için yalın bir Node/Git çalıştırıcısı;
- normal işlevsellik hatları için aynı tarball'ı `/app` içine kuran işlevsel bir görüntü.

Docker hattı tanımları `scripts/lib/docker-e2e-scenarios.mjs` içinde, planlayıcı mantığı `scripts/lib/docker-e2e-plan.mjs` içinde bulunur ve çalıştırıcı yalnızca seçilen planı yürütür. Zamanlayıcı, `OPENCLAW_DOCKER_E2E_BARE_IMAGE` ve `OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE` ile hat başına görüntüyü seçer, ardından hatları `OPENCLAW_SKIP_DOCKER_BUILD=1` ile çalıştırır.

### Ayarlanabilir Değerler

| Değişken                               | Varsayılan | Amaç                                                                                       |
| -------------------------------------- | ------- | --------------------------------------------------------------------------------------------- |
| `OPENCLAW_DOCKER_ALL_PARALLELISM`      | 10      | Normal hatlar için ana havuz yuvası sayısı.                                                        |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM` | 10      | Sağlayıcıya duyarlı kuyruk havuzu yuvası sayısı.                                                      |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT`       | 9       | Sağlayıcıların hız sınırlaması uygulamaması için eşzamanlı canlı hat sınırı.                                        |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT`        | 5       | Eşzamanlı npm kurulum hattı sınırı.                                                              |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT`    | 7       | Eşzamanlı çok hizmetli hat sınırı.                                                            |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS` | 2000    | Docker daemon oluşturma fırtınalarını önlemek için hat başlangıçları arasındaki gecikme; gecikmeyi kaldırmak için `0` ayarlayın.     |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS`  | 7200000 | Hat başına yedek zaman aşımı (120 dakika); seçilen canlı/kuyruk hatları daha sıkı sınırlar kullanır.           |
| `OPENCLAW_DOCKER_ALL_DRY_RUN`          | ayarlanmamış   | `1`, hatları çalıştırmadan zamanlayıcı planını yazdırır.                                          |
| `OPENCLAW_DOCKER_ALL_LANES`            | ayarlanmamış   | Virgülle ayrılmış tam hat listesi; aracıların tek bir başarısız hattı yeniden üretebilmesi için temizleme duman testini atlar. |

Geçerli sınırından daha ağır bir hat boş bir havuzdan yine de başlayabilir, ardından kapasiteyi serbest bırakana kadar tek başına çalışır. Yerel toplu işlem Docker için ön kontrolleri yapar, eski OpenClaw E2E konteynerlerini kaldırır, etkin hat durumunu yayınlar, en uzundan en kısaya sıralama için hat zamanlamalarını kalıcılaştırır ve varsayılan olarak ilk başarısızlıktan sonra yeni havuzlanmış hatları zamanlamayı durdurur.

### Yeniden kullanılabilir canlı/E2E iş akışı

Yeniden kullanılabilir canlı/E2E iş akışı; hangi paketin, görüntü türünün, canlı görüntünün, hattın ve kimlik bilgisi kapsamının gerekli olduğunu `scripts/test-docker-all.mjs --plan-json` öğesine sorar. Ardından `scripts/docker-e2e.mjs`, bu planı GitHub çıktılarına ve özetlerine dönüştürür. OpenClaw'ı `scripts/package-openclaw-for-docker.mjs` aracılığıyla paketler, geçerli çalıştırmaya ait bir paket yapıtını indirir veya `package_artifact_run_id` kaynağından bir paket yapıtı indirir ve ardından tarball envanterini doğrular. Varsayılan `no-push-artifact` yolu, Blacksmith'in Docker katmanı önbelleği üzerinden paket özetiyle etiketlenmiş yalın/işlevsel görüntüler oluşturur, tam görüntü baytlarını değiştirilemez bir iş akışı yapıtında paketler ve her tüketicinin bu yapıtı doğrulayıp yüklemesini sağlar. Buna karşılık `existing-only`, açıkça belirtilmiş `docker_e2e_bare_image`/`docker_e2e_functional_image` GHCR referansları gerektirir ve hiçbir zaman derleme veya gönderme yapmaz. Bu kayıt defteri çekme işlemleri, takılı kalan bir akışın CI kritik yolunun çoğunu tüketmek yerine hızla yeniden denenmesi için deneme başına sınırlandırılmış 180 saniyelik zaman aşımı kullanır. Başarılı zamanlanmış doğrulamadan sonra `openclaw-scheduled-live-checks.yml`, değiştirilemez test edilmiş görüntü bildirimini ayrı paket yazma yayımcısına iletir; salt okunur sürüm ve ön sürüm çağırıcıları hiçbir zaman bu yazıcıdan geçmez.

### Sürüm yolu parçaları

Sürüm Docker kapsamı, `OPENCLAW_SKIP_DOCKER_BUILD=1` ile daha küçük parçalı işler çalıştırır; böylece her parça yalnızca ihtiyaç duyduğu yapıt destekli görüntü türünü doğrulayıp yükler (veya açık `existing-only` yeniden kullanımı kapsamında çeker) ve aynı ağırlıklı zamanlayıcı üzerinden birden fazla hattı yürütür:

- `OPENCLAW_DOCKER_ALL_PROFILE=release-path`
- `OPENCLAW_DOCKER_ALL_CHUNK=core | package-update-openai | package-update-anthropic | package-update-core | plugins-runtime-plugins | plugins-runtime-services | plugins-runtime-install-a..h | openwebui`

Geçerli sürüm Docker parçaları `core`, `package-update-openai`, `package-update-anthropic`, `package-update-core`, `plugins-runtime-plugins`, `plugins-runtime-services`, `plugins-runtime-install-a` ile `plugins-runtime-install-h` arası ve `openwebui` öğeleridir. `package-update-openai`, aday OpenClaw paketini kuran, Codex plugin'ini `codex_plugin_spec` kaynağından veya açık Codex CLI kurulum onayıyla aynı referansa ait bir tarball'dan kuran, Codex CLI ön kontrolünü ve aynı oturumdaki aracı turlarını çalıştıran, ardından ilerleme gönderen, rastgele seçilmiş çalışma alanı girdilerini okuyan, bunların birebir yapıtını yazan ve tamamlanma bildirimi gönderen sıfır yeniden denemeli orta düzey düşünme turunu çalıştıran canlı Codex plugin paket hattını içerir. `plugins-runtime-core`, `plugins-runtime` ve `plugins-integrations` toplu plugin/çalışma zamanı takma adları olarak kalır. `install-e2e` hat takma adı, her iki sağlayıcı yükleyici hattı için toplu manuel yeniden çalıştırma takma adı olarak kalır.

OpenWebUI, kararlı veya tam sürüm yolu kapsamı bunu istediğinde, yeniden kullanılabilir iş akışı desteklenen işleri GitHub tarafından barındırılan çalıştırıcılara yönlendirse bile özel bir büyük diskli Blacksmith çalıştırıcısında bağımsız bir `openwebui` parçası olarak çalışır. Harici görüntü çekme işlemini ayrı tutmak, büyük görüntünün `plugins-runtime-services` içindeki paylaşılan paket ve plugin görüntüleriyle rekabet etmesini önler; eski toplu plugin/çalışma zamanı parçaları, uyumlu manuel yeniden çalıştırmalar için OpenWebUI'yi içermeye devam eder. Paketlenmiş kanal güncelleme hatları, geçici npm ağ hataları için bir kez yeniden dener.

Her parça; hat günlükleri, zamanlamalar, `summary.json`, `failures.json`, aşama zamanlamaları, zamanlayıcı planı JSON'u, yavaş hat tabloları ve hat başına yeniden çalıştırma komutlarıyla birlikte `.artifacts/docker-tests/` öğesini yükler. İş akışının `docker_lanes` girdisi, seçilen hatları parça işleri yerine o çalıştırma için hazırlanmış görüntülere karşı çalıştırır; bu da başarısız hat hata ayıklamasını hedeflenmiş tek bir Docker işiyle sınırlar. Seçilen hat canlı bir Docker hattıysa hedeflenmiş iş, söz konusu yeniden çalıştırma için canlı test görüntüsünü yerel olarak oluşturur. Yeniden çalıştırma yardımcısı, başarısızlık yapıtının tam olarak seçilmiş hedef SHA değerini doğrular ve manuel tetikleme bu referansı yeniden paketler; çünkü dahili yeniden kullanılabilir iş akışı paket demeti `workflow_dispatch` şemasının parçası değildir. Oluşturulan komutlar, hazırlanmış görüntü girdilerini ve yalnızca bu girdiler GHCR destekliyse `shared_image_policy=existing-only` öğesini içerir; yeni bir çalıştırıcının bunları yeniden oluşturması için çalıştırıcıya yerel yapıt etiketleri atlanır. Açık bir hedef geçersiz kılması, yapıt bunların geçersiz kılmayla eşleştiğini kanıtlamadıkça kurtarılan GHCR görüntü referanslarını kaldırır. Tam sürüme ait geçici dallar silindiği için yapıt tarafından oluşturulan iş akışı tanımı referansları da atlanır; operatör açıkça geçersiz kılmadıkça tetikleme, deponun varsayılan dalını kullanır.

```bash
pnpm test:docker:rerun <run-id>      # Docker yapıtlarını indirin ve birleştirilmiş/hat başına hedeflenmiş yeniden çalıştırma komutlarını yazdırın
pnpm test:docker:timings <summary>   # yavaş hat ve aşama kritik yol özetleri
```

Zamanlanmış canlı/E2E iş akışı, tam sürüm yolu Docker paketini günlük olarak çalıştırır ve başarılı olduktan sonra tam olarak test edilen görüntü yapıtları için açık yayımcıyı çağırır.

## Plugin Ön Sürümü

`Plugin Prerelease` daha maliyetli ürün/paket kapsamıdır; bu nedenle `Full Release Validation` tarafından veya açık bir operatör işlemiyle başlatılan ayrı bir iş akışıdır. Normal pull request'ler, `main` push'ları ve bağımsız manuel CI başlatmaları bu paketi devre dışı tutar. Birlikte sunulan plugin testlerini sekiz eklenti worker'ı arasında dengeler; içe aktarma ağırlıklı plugin gruplarının ek CI işleri oluşturmaması için bu eklenti shard işleri, grup başına bir Vitest worker'ı ve daha büyük bir Node heap'i ile aynı anda en fazla iki plugin yapılandırma grubunu çalıştırır. Yalnızca sürüme yönelik Docker ön sürüm yolu (`full_release_validation` girdisiyle etkinleştirilir), bir ila üç dakikalık işler için düzinelerce runner ayırmaktan kaçınmak üzere hedeflenen Docker şeritlerini dörtlü gruplar hâlinde toplu olarak çalıştırır. İş akışı ayrıca `@openclaw/plugin-inspector` kaynağından bilgilendirme amaçlı bir `plugin-inspector-advisory` artefaktı yükler; denetleyici bulguları triyaj girdisidir ve engelleyici Plugin Ön Sürüm kapısını değiştirmez.

## QA Lab

QA Lab'in ana akıllı kapsamlı iş akışının dışında özel CI şeritleri vardır. Aracı eşdeğerliği, bağımsız bir PR iş akışı olarak değil, geniş QA ve sürüm düzeneklerinin altında iç içe yer alır. Eşdeğerliğin geniş bir doğrulama çalışmasıyla birlikte yürütülmesi gerektiğinde `Full Release Validation` ile `rerun_group=qa-parity` kullanın.

- `QA-Lab - All Lanes` iş akışı, her gece `main` üzerinde ve manuel başlatmayla çalışır; sahte eşdeğerlik ile canlı Matrix, Telegram, Discord, WhatsApp ve Slack işlerini paralel olarak dağıtır. Canlı işler `qa-live-shared` ortamını kullanır; Telegram, Discord, WhatsApp ve Slack Convex kiralamalarını kullanırken Matrix tek kullanımlık yerel kimlik bilgileri sağlar.

Sürüm kontrolleri, kanal sözleşmesini canlı model gecikmesinden ve normal sağlayıcı-plugin başlangıcından yalıtmak için deterministik sahte sağlayıcı ve sahte olarak nitelendirilmiş modellerle (`mock-openai/gpt-5.6-luna` ve `mock-openai/gpt-5.6-luna-alt`) Matrix ve Telegram canlı taşıma şeritlerini çalıştırır. Canlı taşıma Gateway'i bellek aramasını devre dışı bırakır; çünkü QA eşdeğerliği bellek davranışını ayrı olarak kapsar. Sağlayıcı bağlantısı ise ayrı canlı model, yerel sağlayıcı ve Docker sağlayıcı paketleri tarafından kapsanır.

Zamanlanmış ve sürüm Matrix kapıları, paylaşılan QA Lab paket barındırıcısını ve canlı bağdaştırıcıyı sürüm senaryolarıyla kullanır. CLI varsayılanı ve manuel iş akışı girdisi `all` olarak kalır; manuel `all` başlatmaları, 93 senaryoluk kanıtın iş başına zaman aşımları içinde kalması için `transport`, `media`, `e2ee-smoke`, `e2ee-deep` ve `e2ee-cli` profillerini paralel olarak dağıtır. Odaklanmış manuel başlatmalar, tek bir işte `fast`, `release` veya `transport` seçer.

`OpenClaw Release Checks` ayrıca sürüm onayından önce sürüm açısından kritik QA Lab şeritlerini çalıştırır; QA eşdeğerlik kapısı, aday ve temel paketleri paralel şerit işleri olarak çalıştırır, ardından son eşdeğerlik karşılaştırması için her iki artefaktı küçük bir rapor işine indirir.

Normal PR'lerde eşdeğerliği zorunlu bir durum olarak değerlendirmek yerine kapsamlı CI/kontrol kanıtlarını izleyin.

## CodeQL

`CodeQL` iş akışı, tam depo taraması değil, bilinçli olarak dar kapsamlı bir ilk geçiş güvenlik tarayıcısıdır. Günlük, manuel, `main` push ve taslak olmayan pull request koruma çalışmaları; Actions iş akışı kodunu ve en yüksek riskli JavaScript/TypeScript yüzeylerini, yüksek/kritik `security-severity` düzeylerine filtrelenmiş yüksek güvenilirlikli güvenlik sorgularıyla tarar.

Pull request koruması hafif kalır: yalnızca `.github/actions`, `.github/codeql`, `.github/workflows`, `packages`, `scripts`, `src` altındaki veya süreç sahibi birlikte sunulan plugin çalışma zamanı yollarındaki değişiklikler için başlar ve zamanlanmış iş akışıyla aynı yüksek güvenilirlikli güvenlik matrisini çalıştırır. Android ve macOS CodeQL, PR varsayılanlarının dışında kalır.

### Güvenlik kategorileri

| Kategori                                          | Yüzey                                                                                                                             |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-security-high/core-auth-secrets`         | Kimlik doğrulama, gizli bilgiler, korumalı alan, cron ve gateway temel çizgisi                                                                                  |
| `/codeql-security-high/channel-runtime-boundary`  | Temel kanal uygulama sözleşmeleri ile kanal plugin çalışma zamanı, gateway, Plugin SDK, gizli bilgiler ve denetim temas noktaları              |
| `/codeql-security-high/network-ssrf-boundary`     | Temel SSRF, IP ayrıştırma, ağ koruması, web'den getirme ve Plugin SDK SSRF politikası yüzeyleri                                                |
| `/codeql-security-high/mcp-process-tool-boundary` | MCP sunucuları, süreç yürütme yardımcıları, giden teslimat ve aracı araç yürütme kapıları                                           |
| `/codeql-security-high/process-exec-boundary`     | Yerel kabuk, süreç başlatma yardımcıları, alt süreç sahibi birlikte sunulan plugin çalışma zamanları ve iş akışı betiği bağlayıcı kodu                             |
| `/codeql-security-high/plugin-trust-boundary`     | Plugin kurulumu, yükleyici, bildirim, kayıt defteri, paket yöneticisi kurulumu, kaynak yükleme ve Plugin SDK paket sözleşmesi güven yüzeyleri |

### Platforma özgü güvenlik shard'ları

- `CodeQL Android Critical Security` — zamanlanmış Android güvenlik shard'ı. Android uygulamasını CodeQL için iş akışı sağlamlık kontrolünün kabul ettiği en küçük Blacksmith Linux runner'ında manuel olarak derler. `/codeql-critical-security/android` altında yükler.
- `CodeQL macOS Critical Security` — haftalık/manuel macOS güvenlik shard'ı. macOS uygulamasını CodeQL için Blacksmith macOS üzerinde manuel olarak derler, bağımlılık derleme sonuçlarını yüklenen SARIF'ten filtreler ve `/codeql-critical-security/macos` altında yükler. macOS derlemesi temiz olduğunda bile çalışma süresine hâkim olduğu için günlük varsayılanların dışında tutulur.

### Kritik Kalite kategorileri

`CodeQL Critical Quality`, buna karşılık gelen güvenlik dışı shard'dır. Kalite taramalarının Blacksmith runner kayıt bütçesini tüketmemesi için GitHub tarafından barındırılan Linux runner'larında, dar kapsamlı ve yüksek değerli yüzeyler üzerinde yalnızca hata önem dereceli, güvenlik dışı JavaScript/TypeScript kalite sorgularını çalıştırır. Pull request koruması bilinçli olarak zamanlanmış profilden daha küçüktür: taslak olmayan PR'ler, PR'ye yönlendirilebilir on üç shard arasından yalnızca dokundukları yüzeylerle eşleşen shard'ları çalıştırır — `agent-runtime-boundary`, `channel-runtime-boundary`, `config-boundary`, `core-auth-secrets`, `gateway-runtime-boundary`, `mcp-process-runtime-boundary`, `memory-runtime-boundary`, `network-runtime-boundary`, `plugin-boundary`, `plugin-sdk-package-contract`, `plugin-sdk-reply-runtime`, `provider-runtime-boundary` ve `session-diagnostics-boundary`. `ui-control-plane` ve `web-media-runtime-boundary`, PR çalışmalarının dışında kalır. CodeQL yapılandırması ve kalite iş akışı değişiklikleri, tam PR shard kümesini çalıştırır (ağ çalışma zamanı shard'ı kendi CodeQL yapılandırma dosyalarına ve ağ sahibi kaynak yollarına göre tetiklenir).

Manuel başlatma şunları kabul eder:

```text
profile=all|agent-runtime-boundary|config-boundary|core-auth-secrets|channel-runtime-boundary|gateway-runtime-boundary|memory-runtime-boundary|mcp-process-runtime-boundary|network-runtime-boundary|plugin-boundary|plugin-sdk-package-contract|plugin-sdk-reply-runtime|provider-runtime-boundary|session-diagnostics-boundary
```

Dar profiller, tek bir kalite shard'ını yalıtılmış biçimde çalıştırmak için öğretim/yineleme kancalarıdır.

| Kategori                                                | Yüzey                                                                                                                                                           |
| ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-critical-quality/core-auth-secrets`            | Kimlik doğrulama, gizli bilgiler, korumalı alan, cron ve gateway güvenlik sınırı kodu                                                                                                  |
| `/codeql-critical-quality/config-boundary`              | Yapılandırma şeması, geçiş, normalleştirme ve G/Ç sözleşmeleri                                                                                                         |
| `/codeql-critical-quality/gateway-runtime-boundary`     | Gateway protokol şemaları ve sunucu yöntemi sözleşmeleri                                                                                                              |
| `/codeql-critical-quality/channel-runtime-boundary`     | Temel kanal ve birlikte sunulan kanal plugin'i uygulama sözleşmeleri                                                                                                  |
| `/codeql-critical-quality/agent-runtime-boundary`       | Komut yürütme, model/sağlayıcı yönlendirme, otomatik yanıt yönlendirmesi ve kuyrukları ile ACP kontrol düzlemi çalışma zamanı sözleşmeleri                                               |
| `/codeql-critical-quality/mcp-process-runtime-boundary` | MCP sunucuları ve araç köprüleri, süreç denetimi yardımcıları ve giden teslimat sözleşmeleri                                                                        |
| `/codeql-critical-quality/memory-runtime-boundary`      | Bellek barındırıcı SDK'sı, bellek çalışma zamanı cepheleri, bellek Plugin SDK diğer adları, bellek çalışma zamanı etkinleştirme bağlayıcı kodu ve bellek doctor komutları                                    |
| `/codeql-critical-quality/network-runtime-boundary`     | Ağ politikası paketi, ham soket ve proxy yakalama çalışma zamanı, SSH tüneli, gateway kilidi, JSONL soketi ve push taşıma yüzeyleri                                 |
| `/codeql-critical-quality/session-diagnostics-boundary` | Yanıt kuyruğu iç bileşenleri, oturum teslimat kuyrukları, giden oturum bağlama/teslimat yardımcıları, tanılama olayı/günlük paketi yüzeyleri ve oturum doctor CLI sözleşmeleri |
| `/codeql-critical-quality/plugin-sdk-reply-runtime`     | Plugin SDK gelen yanıt yönlendirmesi, yanıt yükü/parçalama/çalışma zamanı yardımcıları, kanal yanıt seçenekleri, teslimat kuyrukları ve oturum/iş parçacığı bağlama yardımcıları             |
| `/codeql-critical-quality/provider-runtime-boundary`    | Model kataloğu normalleştirmesi, sağlayıcı kimlik doğrulaması ve keşfi, sağlayıcı çalışma zamanı kaydı, sağlayıcı varsayılanları/katalogları ve web/arama/getirme/gömme kayıt defterleri    |
| `/codeql-critical-quality/ui-control-plane`             | Kontrol kullanıcı arayüzü önyüklemesi, yerel kalıcılık, gateway kontrol akışları ve görev kontrol düzlemi çalışma zamanı sözleşmeleri                                                          |
| `/codeql-critical-quality/web-media-runtime-boundary`   | Temel web getirme/arama, medya G/Ç, medya anlama, görüntü oluşturma ve medya oluşturma çalışma zamanı sözleşmeleri                                                    |
| `/codeql-critical-quality/plugin-boundary`              | Yükleyici, kayıt defteri, genel yüzey ve Plugin SDK giriş noktası sözleşmeleri                                                                                             |
| `/codeql-critical-quality/plugin-sdk-package-contract`  | Yayımlanan paket tarafı Plugin SDK kaynağı ve plugin paket sözleşmesi yardımcıları                                                                                      |

Kalite, güvenlik sinyalini belirsizleştirmeden kalite bulgularının zamanlanabilmesi, ölçülebilmesi, devre dışı bırakılabilmesi veya genişletilebilmesi için güvenlikten ayrı tutulur. Swift, Python ve birlikte sunulan plugin CodeQL genişletmesi, yalnızca dar profiller kararlı çalışma süresine ve sinyale ulaştıktan sonra kapsamlı veya shard'lara bölünmüş takip çalışması olarak yeniden eklenmelidir.

## Bakım iş akışları

### Doküman Aracısı

`Docs Agent` iş akışı, mevcut dokümanları yakın zamanda birleştirilen değişikliklerle uyumlu tutmaya yönelik olay güdümlü bir Codex bakım şerididir. Yalnızca zamanlamaya dayalı bir çalışması yoktur: `main` üzerindeki başarılı, bot dışı bir push CI çalışması bunu tetikleyebilir ve manuel başlatma doğrudan çalıştırabilir. İş akışı çalıştırmaları, `main` ilerlemişse veya son bir saat içinde atlanmamış başka bir Doküman Aracısı çalışması oluşturulmuşsa atlanır. Çalıştığında, önceki atlanmamış Doküman Aracısı kaynak SHA'sından geçerli `main` değerine kadar olan commit aralığını inceler; böylece saatlik tek bir çalışma, son doküman geçişinden bu yana biriken tüm ana dal değişikliklerini kapsayabilir.

### Test Performansı Aracısı

`Test Performance Agent` iş akışı, yavaş testlere yönelik olay güdümlü bir Codex bakım hattıdır. Salt bir zamanlaması yoktur: `main` üzerindeki bot dışı başarılı bir push CI çalıştırması bunu tetikleyebilir, ancak aynı UTC gününde başka bir iş akışı çalıştırma çağrısı zaten çalıştıysa veya çalışıyorsa atlanır. Manuel çalıştırma, bu günlük etkinlik kapısını atlar. Hat, tam paket için gruplandırılmış bir Vitest performans raporu oluşturur, Codex'in geniş kapsamlı yeniden düzenlemeler yerine yalnızca kapsamı koruyan küçük test performansı düzeltmeleri yapmasına izin verir, ardından tam paket raporunu yeniden çalıştırır ve geçen temel test sayısını azaltan değişiklikleri reddeder. Gruplandırılmış rapor, Linux ve macOS'ta yapılandırma başına geçen süreyi ve maksimum RSS'yi kaydeder; böylece önce/sonra karşılaştırması, süre farklarının yanında test belleği farklarını da gösterir. Temel durumda başarısız testler varsa Codex yalnızca bariz hataları düzeltebilir ve herhangi bir şey commit edilmeden önce ajan sonrası tam paket raporunun geçmesi gerekir. Bot push'u uygulanmadan önce `main` ilerlerse hat, doğrulanmış yamayı rebase eder, `pnpm check:changed` öğesini yeniden çalıştırır ve push'u tekrar dener; çakışan eski yamalar atlanır. Codex eyleminin doküman ajanıyla aynı sudo kaldırma güvenlik yaklaşımını sürdürebilmesi için GitHub tarafından barındırılan Ubuntu kullanır.

### Birleştirme Sonrası Yinelenen PR'ler

`Duplicate PRs After Merge` iş akışı, birleştirme sonrası yinelenenleri temizlemeye yönelik manuel bir bakımcı iş akışıdır. Varsayılan olarak deneme çalıştırması yapar ve yalnızca `apply=true` olduğunda açıkça listelenen PR'leri kapatır. GitHub'da değişiklik yapmadan önce, birleştirilen PR'nin birleştirilmiş olduğunu ve her yinelenenin ya ortak bir başvurulan soruna ya da örtüşen değiştirilmiş parçalara sahip olduğunu doğrular.

```bash
gh workflow run duplicate-after-merge.yml \
  -f landed_pr=70532 \
  -f duplicate_prs='70530,70592' \
  -f apply=true
```

## Yerel kontrol kapıları ve değişiklik yönlendirmesi

### Yapılandırma temel sayı mandalı

`pnpm config:docs:check`, belgelenmemiş yapılandırma yüzeyi büyümesini ve bozuk ya da güncelliğini yitirmiş sayı anlık görüntülerini reddeder. İncelenmiş bir ürün değişikliği kasıtlı olarak şema yolları eklediğinde `pnpm config:docs:gen` öğesini çalıştırın, çekirdek/kanal/plugin sayı farklarını ve oluşturulan SHA-256 dosyalarını inceleyin; bilinçli temel artışını şema, yardım, etiketler, geçiş ve testlerle birlikte commit edin. Mandalı atlamak için sayı dosyasını elle düzenlemeyin.

Yapılandırma yazarları ayrıca Settings için yeni yapraklara katman atamalıdır. Yaprağa `advanced: false` veya
`advanced: true` ekleyin ya da anahtarı, katmanını tüm alt öğelerin
devralması gereken bir üst öğenin altına yerleştirin. Sınıflandırılmamış kökler, kopyalanıp yapıştırılabilir taslaklarla şema kalitesi
testinde başarısız olur; üst öğesi olmayan yollar varsayılan olarak gelişmiş kabul edilir.
Özenle seçilmiş ortak yaprak anlık görüntüsü, kasıtlı katman değişikliklerini
incelemede görünür kılar.

Yerel değişiklik hattı mantığı `scripts/changed-lanes.mjs` içinde bulunur ve `scripts/check-changed.mjs` tarafından yürütülür. Bu yerel kontrol kapısı, mimari sınırlar konusunda geniş CI platformu kapsamından daha katıdır:

- çekirdek üretim değişiklikleri; çekirdek üretim ve çekirdek test tür denetiminin yanı sıra çekirdek lint/koruma denetimlerini çalıştırır;
- yalnızca çekirdek test değişiklikleri; yalnızca çekirdek test tür denetiminin yanı sıra çekirdek lint denetimini çalıştırır;
- eklenti üretim değişiklikleri; eklenti üretim ve eklenti test tür denetiminin yanı sıra eklenti lint denetimini çalıştırır;
- yalnızca eklenti test değişiklikleri; eklenti test tür denetiminin yanı sıra eklenti lint denetimini çalıştırır;
- herkese açık Plugin SDK veya plugin sözleşmesi değişiklikleri, eklentiler bu çekirdek sözleşmelere bağımlı olduğundan eklenti tür denetimini kapsayacak şekilde genişler (Vitest eklenti taramaları açık test çalışması olarak kalır);
- yalnızca sürüm meta verilerini değiştiren sürüm artışları, hedefli sürüm/yapılandırma/kök bağımlılık kontrollerini çalıştırır;
- bilinmeyen kök/yapılandırma değişiklikleri güvenli biçimde tüm kontrol hatlarına yönlendirilir.

Yerel değişiklik testi yönlendirmesi `scripts/test-projects.test-support.mjs` içinde bulunur ve kasıtlı olarak `check:changed` öğesinden daha az maliyetlidir: doğrudan test düzenlemeleri kendi testlerini çalıştırır; kaynak düzenlemeleri önce açık eşlemeleri, ardından kardeş testleri ve içe aktarma grafiğindeki bağımlıları tercih eder. Paylaşılan grup odası teslimat yapılandırması açık eşlemelerden biridir: grubun görünür yanıt yapılandırmasındaki, kaynak yanıt teslimat modundaki veya mesaj aracı sistem istemindeki değişiklikler; çekirdek yanıt testlerinin yanı sıra Discord ve Slack teslimat regresyonlarından geçirilir; böylece paylaşılan bir varsayılan değişikliği ilk PR push'undan önce başarısız olur. `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` öğesini yalnızca değişiklik, düşük maliyetli eşlenmiş kümenin güvenilir bir vekil olamayacağı kadar test altyapısı genelindeyse kullanın.

## Testbox doğrulaması

Crabbox, bakımcı Linux kanıtı için repoya ait uzak kutu sarmalayıcısıdır. Ajan
oturumları, mevcut bağımlılık kurulumu hazır olduğunda yalnızca güvenilir kaynaklar için
birkaç odaklı testi ve düşük maliyetli statik kontrolü yerelde tutar. Derlemeler, tür denetimleri, lint dağıtımı,
Docker, paket hatları, E2E, canlı kanıt ve CI eşdeğerliği dahil olmak üzere daha büyük paketler ve
hesaplama açısından yoğun çalışmalar için Crabbox kullanırlar. Güvenilir bakımcı ağır
kanıtı varsayılan olarak `blacksmith-testbox` kullanır ve `.crabbox.yaml` artık varsayılan olarak bunu kullanır. Yapılandırılmış
iş akışı sağlayıcı ve ajan kimlik bilgilerini yükler; bu nedenle güvenilmeyen katkıcı veya
fork kodu bunun yerine secretsız fork CI ya da temizlenmiş doğrudan AWS Crabbox kullanmalıdır.
Temizlenmiş AWS çalıştırmaları `CRABBOX_ENV_ALLOW=CI` değerini ayarlar,
`--no-hydrate` geçirir ve yeni bir geçici uzak `HOME` kullanır; bu, repo
`OPENCLAW_*` izin listesinin ve mevcut kimlik doğrulama profillerinin güvenilmeyen koda ulaşmasını önler.
Bu güvenilmeyen kaynağa ayrılmış, yeni ısıtılmış bir kiralama kullanırlar; güvenilir veya
daha önce yüklenmiş bir kiralamayı asla kullanmazlar. Kurulu güvenilir bir Crabbox
ikilisini temiz ve güvenilir bir `main` çalışma kopyasından başlatın ve yalnızca uzak PR'yi
`--fresh-pr` ile getirin; güvenilmeyen çalışma kopyasının sarmalayıcısını veya yapılandırmasını asla yerelde çalıştırmayın.
`CRABBOX_AWS_INSTANCE_PROFILE` ayarını kaldırın ve çözümlenen
`aws.instanceProfile` boş değilse güvenli biçimde başarısız olun. Herhangi bir kurulum/testten önce, IMDSv2 belirteci gerektirmek,
IAM kimlik bilgileri uç noktasının 404 döndürdüğünü kanıtlamak ve uzak `git rev-parse HEAD` değerini
incelenmiş PR başlığının tam SHA'sı ile karşılaştırmak için güvenilir mutlak yollu araçlar kullanın. Kiralamayı bu SHA'ya bağlayın ve başlık değiştiğinde durdurup yeniden ısıtın.
Güvenilir `scripts/crabbox-untrusted-bootstrap.sh` dosyasını temiz `main` üzerinden
`--fresh-pr` ile birlikte yükleyin; bu, sabitlenmiş Node/pnpm'yi kurar, SHA'yı ve
paket yöneticisi sabitlemesini doğrular, `HOME` ortamını yalıtır, bağımlılıkları kurar ve ardından
istenen testi yürütür.
Tüm `CRABBOX_TAILSCALE*` geçersiz kılmalarını kaldırın, `--network public
--tailscale=false` kullanımını zorunlu kılın, çıkış düğümü/LAN bayraklarını temizleyin ve herhangi bir betiği yüklemeden önce `crabbox inspect` öğesinin
Tailscale durumu olmadan genel ağ bağlantısı bildirmesini zorunlu tutun.
Sahip olunan AWS/Hetzner kapasitesi de Blacksmith kesintileri,
kota sorunları veya açıkça sahip olunan kapasite testi için yedek seçenek olarak kalır.

Ajanlar öngörülen işler için önceden ısıtma yapmaz. İlk ağır komut
hazır olduğunda bir Testbox'ı gerektiği anda edinin, döndürülen `tbx_...` kimliğini sonraki ağır
komutlar için yeniden kullanın, her çalıştırmada mevcut çalışma kopyasını eşitleyin ve devretmeden önce durdurun.

Crabbox destekli Blacksmith çalıştırmaları, tek kullanımlık Testbox'ları ısıtır, sahiplenir, eşitler, çalıştırır, raporlar ve temizler.
Yerleşik eşitleme sağlamlık kontrolü, eşitlenen kutudaki
`git status --short` en az 200 izlenen silme gösterdiğinde hızla başarısız olur;
bu, `pnpm-lock.yaml` gibi kaybolan kök dosyaları yakalar. Kasıtlı
büyük silme PR'leri için uzak komutta `CRABBOX_ALLOW_MASS_DELETIONS=1` değerini ayarlayın.

Crabbox ayrıca eşitleme sonrası çıktı olmadan beş dakikadan uzun süre
eşitleme aşamasında kalan yerel Blacksmith CLI çağrısını sonlandırır. Bu korumayı devre dışı bırakmak için
`CRABBOX_BLACKSMITH_SYNC_TIMEOUT_MS=0` değerini ayarlayın veya alışılmadık derecede büyük yerel farklar için daha büyük bir
milisaniye değeri kullanın.

İlk çalıştırmadan önce sarmalayıcıyı repo kökünden kontrol edin:

```bash
pnpm crabbox:run -- --help | sed -n '1,120p'
```

Repo sarmalayıcısı, seçili sağlayıcıyı duyurmayan eski bir Crabbox ikilisini reddeder ve Blacksmith destekli çalıştırmalar, sarmalayıcının güncel Testbox eşitleme, kuyruk ve temizleme davranışını alabilmesi için Crabbox 0.22.0 veya daha yenisini gerektirir. Codex çalışma ağaçlarında ya da bağlı/seyrek çalışma kopyalarında yerel `pnpm crabbox:run` betiğinden kaçının; çünkü pnpm, Crabbox başlamadan önce bağımlılıkları uzlaştırabilir. Bunun yerine node sarmalayıcısını doğrudan çağırın:

```bash
node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox --timing-json --shell -- "pnpm test <path-or-filter>"
```

Kardeş çalışma kopyasını kullanırken zamanlama veya kanıt çalışmasından önce yok sayılan yerel ikiliyi yeniden derleyin:

```bash
version="$(git -C ../crabbox describe --tags --always --dirty | sed 's/^v//')" \
  && go build -C ../crabbox -trimpath -ldflags "-s -w -X github.com/openclaw/crabbox/internal/cli.version=${version}" -o bin/crabbox ./cmd/crabbox
```

`.crabbox.yaml` içindeki `blacksmith:` bloğu kuruluş, iş akışı, iş ve ref varsayılanlarını zaten sabitler; dolayısıyla aşağıdaki açık bayraklar isteğe bağlıdır. Değişiklik kapısı:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --blacksmith-org openclaw \
  --blacksmith-workflow .github/workflows/ci-check-testbox.yml \
  --blacksmith-job check \
  --blacksmith-ref main \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm check:changed"
```

Yerel bağımlılıklar kullanılamadığında veya hedef dağıldığında Testbox üzerinde
odaklı testin yeniden çalıştırılması:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm test <path-or-filter>"
```

Tam paket:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm test"
```

Son JSON özetini okuyun. Yararlı alanlar `provider`, `leaseId`,
`syncDelegated`, `exitCode`, `commandMs` ve `totalMs` alanlarıdır. Devredilmiş
Blacksmith Testbox çalıştırmalarında Crabbox sarmalayıcısının çıkış kodu ve JSON özeti
komut sonucudur. Bağlı GitHub Actions çalıştırması yükleme ve canlı tutma işlemlerini yönetir; SSH
komutu zaten döndükten sonra Testbox dışarıdan durdurulursa
`cancelled` olarak tamamlanabilir. Sarmalayıcı `exitCode` sıfırdan farklı olmadığı veya komut çıktısı başarısız bir test göstermediği sürece
bunu bir temizleme/durum artifaktı olarak değerlendirin.
Tek kullanımlık Blacksmith destekli Crabbox çalıştırmaları Testbox'ı otomatik olarak durdurmalıdır;
bir çalıştırma kesintiye uğrarsa veya temizleme durumu belirsizse canlı kutuları inceleyin ve yalnızca
oluşturduğunuz kutuları durdurun:

```bash
blacksmith testbox list --all
blacksmith testbox status --id <tbx_id>
blacksmith testbox stop --id <tbx_id>
```

Yeniden kullanımı yalnızca aynı yüklenmiş kutuda kasıtlı olarak birden fazla komuta ihtiyacınız olduğunda kullanın:

```bash
node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox --id <tbx_id> --timing-json --shell -- "corepack pnpm test <path-or-filter>"
pnpm crabbox:stop -- <tbx_id>
```

Eski kaynağı değil, kiralamayı yeniden kullanın. Her çalıştırmanın mevcut çalışma kopyasını yüklemesi için
`--no-sync` değerini atlayın; bunu yalnızca değişmemiş, zaten eşitlenmiş bir ağacı kasıtlı olarak
yeniden çalıştırmak için kullanın. Güvenilmeyen katkıcı/fork kodu her komut için
`CRABBOX_ENV_ALLOW=CI`, `--provider aws --no-hydrate` ve yeni bir
geçici uzak `HOME` kullanmalıdır; testten önce bağımlılıkları bu temizlenmiş komutun içinde kurun.
Yalnızca aynı güvenilmeyen kaynağa ayrılmış, yeni ısıtılmış bir kiralamayı yeniden kullanın;
güvenilir veya daha önce yüklenmiş bir kiralamayı asla kullanmayın. Güvenilmeyen çalışma kopyasının
sarmalayıcısını veya yapılandırmasını asla yerelde çalıştırmayın: kurulu güvenilir Crabbox ikilisini temiz ve güvenilir
`main` üzerinden başlatın ve her çalıştırmada `--fresh-pr` geçirin. `CRABBOX_AWS_INSTANCE_PROFILE` ayarını kaldırılmış
tutun, boş olmayan çözümlenmiş bir örnek profilini reddedin, güvenilir bir uzak IMDS rol yokluğu kanıtını zorunlu tutun ve
kurulum/testten önce incelenmiş başlık SHA'sını doğrulayın. Kiralamayı bu SHA'ya bağlayın; herhangi bir başlık değişikliğinden sonra durdurup
yeniden ısıtın. Uzak PR yoksa secretsız fork CI kullanın.
Güvenilmeyen kaynaklar için `hydrate-github` veya kimlik bilgileri yüklenmiş Blacksmith iş akışını
asla seçmeyin.

Bozuk katman Crabbox ise ancak Blacksmith çalışıyorsa, doğrudan
Blacksmith'i yalnızca `list`, `status` ve temizleme gibi tanı işlemleri için kullanın.
Doğrudan bir Blacksmith çalıştırmasını bakımcı kanıtı olarak değerlendirmeden önce
Crabbox yolunu düzeltin.

`blacksmith testbox list --all` ve `blacksmith testbox status` çalışıyor ancak yeni
ısınmalar birkaç dakika sonra IP veya Actions çalıştırma URL'si olmadan `queued` durumunda bekliyorsa,
bunu Blacksmith sağlayıcısı, kuyruk, faturalandırma veya kuruluş sınırı baskısı olarak değerlendirin. Oluşturduğunuz
kuyruğa alınmış kimlikleri durdurun, daha fazla Testbox başlatmaktan kaçının ve biri Blacksmith panosunu,
faturalandırmayı ve kuruluş sınırlarını kontrol ederken kanıtı aşağıdaki kuruluşa ait Crabbox kapasitesi yoluna taşıyın.

Yalnızca Blacksmith devre dışı, kota sınırlı veya gerekli ortamdan yoksun olduğunda ya da açıkça kuruluşa ait kapasitenin kullanılması amaçlandığında kuruluşa ait Crabbox kapasitesine geçin:

```bash
CRABBOX_CAPACITY_REGIONS=eu-west-1,eu-west-2,eu-central-1,us-east-1,us-west-2 \
  pnpm crabbox:warmup -- --provider aws --class standard --market on-demand --idle-timeout 90m
pnpm crabbox:hydrate -- --provider aws --id <cbx_id-or-slug>
pnpm crabbox:run -- --provider aws --id <cbx_id-or-slug> --timing-json --shell -- "pnpm check:changed"
pnpm crabbox:stop -- --provider aws <cbx_id-or-slug>
```

AWS baskısı altında, görev gerçekten 48xlarge sınıfı CPU gerektirmediği sürece `class=beast` kullanmaktan kaçının. Bir `beast` isteği 192 vCPU ile başlar ve bölgesel EC2 Spot veya On-Demand Standard kotasını aşmanın en kolay yoludur. Depoya ait `.crabbox.yaml` varsayılan olarak `class: standard`, isteğe bağlı pazar ve `capacity.hints: true` kullanır; böylece aracılı AWS kiralamaları seçilen bölgeyi/pazarı, kota baskısını, Spot geri dönüşünü ve yüksek baskılı sınıf uyarılarını yazdırır. Daha ağır ve kapsamlı kontroller için `fast`, yalnızca standard/fast yeterli olmadığında `large` ve yalnızca tam paket veya tüm Plugin Docker matrisleri, açık sürüm/engelleyici doğrulaması ya da yüksek çekirdekli performans profili çıkarma gibi olağanüstü CPU ağırlıklı hatlar için `beast` kullanın. `pnpm check:changed`, odaklanmış testler, yalnızca belge çalışmaları, olağan lint/tür denetimi, küçük E2E yeniden üretimleri veya Blacksmith kesintisi triyajı için `beast` kullanmayın. Spot pazarındaki dalgalanmanın sinyale karışmaması için kapasite tanılamasında `--market on-demand` kullanın.

`.crabbox.yaml`; sağlayıcı, eşitleme ve GitHub Actions hidrasyon varsayılanlarının sahibidir. Crabbox eşitlemesi hiçbir zaman `.git` aktarmaz; böylece hidrate edilmiş Actions çalışma kopyası, bakımcıya ait yerel uzak depoları ve nesne depolarını eşitlemek yerine kendi uzak Git meta verilerini korur. Ayrıca depo yapılandırması, hiçbir zaman aktarılmaması gereken yerel çalışma zamanı/derleme yapılarını (`.artifacts` ve test raporları gibi) hariç tutar. `.github/workflows/crabbox-hydrate.yml`; çalışma kopyası alma, Node/pnpm kurulumu, `origin/main` getirme ve kuruluşa ait buluttaki `crabbox run --id <cbx_id>` komutları için gizli olmayan ortam aktarımının sahibidir.

## İlgili

- [Kuruluma genel bakış](/tr/install)
- [Geliştirme kanalları](/tr/install/development-channels)
