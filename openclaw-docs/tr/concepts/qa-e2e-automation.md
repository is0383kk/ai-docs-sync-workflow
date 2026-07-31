---
doc-schema-version: 1
read_when:
    - QA yığınının nasıl bir araya geldiğini anlama
    - qa-lab, qa-channel veya bir aktarım bağdaştırıcısını genişletme
    - Depo destekli QA senaryoları ekleme
    - Gateway panosu etrafında daha yüksek gerçekçiliğe sahip QA otomasyonu oluşturma
summary: 'QA yığınına genel bakış: qa-lab, qa-channel, depo destekli senaryolar, canlı aktarım kanalları, aktarım bağdaştırıcıları ve raporlama.'
title: QA genel bakışı
x-i18n:
    generated_at: "2026-07-26T23:38:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91c34a50e6197195d57228d92b19caff1785ceaa5d82d7c88a1ec0ed76abd635
    source_path: concepts/qa-e2e-automation.md
    workflow: 16
---

Özel QA yığını, OpenClaw'u birim testinin yapamayacağı gerçekçi ve kanal biçimine uygun
bir şekilde sınar.

Parçalar:

- `extensions/qa-channel`: DM, kanal, iş parçacığı,
  tepki, düzenleme ve silme yüzeylerine sahip sentetik mesaj kanalı.
- `extensions/qa-lab`: transkripti gözlemlemek, gelen mesajları eklemek
  ve Markdown raporu dışa aktarmak için hata ayıklayıcı kullanıcı arayüzü, QA veri yolu,
  senaryo profilleri ve canlı aktarım bağdaştırıcıları.
- `qa/`: başlangıç görevi ve temel QA
  senaryoları için depo destekli başlangıç varlıkları.
- [Mantis](/tr/concepts/mantis): gerçek aktarımlar, tarayıcı ekran görüntüleri,
  VM durumu ve PR kanıtı gerektiren hatalar için canlı öncesi/sonrası doğrulama.

## Komut yüzeyi

Her QA akışı `pnpm openclaw qa <subcommand>` altında çalışır. Birçoğunun `pnpm qa:*`
betik takma adları vardır; her iki biçim de çalışır.

| Komut                                               | Amaç                                                                                                                                                                                                                                                                |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `qa run`                                            | `--qa-profile` olmadan paketlenmiş QA öz denetimi; `--qa-profile smoke-ci`, `--qa-profile release` veya `--qa-profile all` ile taksonomi destekli olgunluk profili çalıştırıcısı.                                                                                     |
| `qa suite`                                          | Depo destekli senaryoları QA gateway hattında çalıştırır. `--runner multipass`, ana makine yerine tek kullanımlık bir Linux VM kullanır.                                                                                                                              |
| `qa coverage`                                       | YAML senaryo kapsamı envanterini yazdırır (makine çıktısı için `--json`; dokunulan bir davranışa yönelik senaryoları bulmak için `--match <query>`; çalışma zamanı aracı fikstür kapsamı için `--tools`).                                           |
| `qa parity-report`                                  | Model ekseni eşlik kapısı için iki `qa-suite-summary.json` dosyasını karşılaştırır veya Codex ile OpenClaw çalışma zamanı eşliği ve token verimliliği raporlarını yazmak için `--runtime-axis --token-efficiency` kullanır.                                                               |
| `qa confidence-report`                              | QA kanıt yapıtlarını bir manifeste göre sınıflandırarak sıfır bilinmeyenli bir güven raporu oluşturur.                                                                                                                                                                  |
| `qa confidence-self-test`                           | Güven kapısının sapmayı algıladığını kanıtlayan başlangıç verileri yüklenmiş negatif kontrol kanaryaları yazar.                                                                                                                                                         |
| `qa jsonl-replay`                                   | Derlenmiş JSONL transkriptlerini çalışma zamanı eşliği yeniden oynatma düzeneğinde yeniden oynatır.                                                                                                                                                                    |
| `qa character-eval`                                 | Karakter QA senaryosunu birden çok canlı modelde değerlendirilmiş bir raporla çalıştırır. Bkz. [Raporlama](#reporting).                                                                                                                                                 |
| `qa manual`                                         | Seçilen sağlayıcı/model hattında tek seferlik bir istem çalıştırır.                                                                                                                                                                                                   |
| `qa ui`                                             | QA hata ayıklayıcı kullanıcı arayüzünü ve yerel QA veri yolunu başlatır (takma ad: `pnpm qa:lab:ui`).                                                                                                                                                                |
| `qa docker-build-image`                             | Önceden hazırlanmış QA Docker imajını oluşturur.                                                                                                                                                                                                                       |
| `qa docker-scaffold`                                | QA panosu + gateway hattı için bir docker-compose iskelesi yazar.                                                                                                                                                                                                     |
| `qa up`                                             | QA sitesini oluşturur, Docker destekli yığını başlatır ve URL'yi yazdırır (takma ad: `pnpm qa:lab:up`; `:fast` çeşidi `--use-prebuilt-image --bind-ui-dist --skip-ui-build` ekler).                                                                                                     |
| `qa aimock`                                         | Yalnızca AIMock sağlayıcı sunucusunu başlatır.                                                                                                                                                                                                                         |
| `qa mock-openai`                                    | Yalnızca senaryo duyarlı `mock-openai` sağlayıcı sunucusunu başlatır.                                                                                                                                                                                                |
| `qa credentials doctor` / `add` / `list` / `remove` | Paylaşılan Convex kimlik bilgisi havuzunu yönetir.                                                                                                                                                                                                                     |
| `qa discord`                                        | Gerçek bir özel Discord sunucusu kanalına karşı canlı aktarım hattı.                                                                                                                                                                                                  |
| `qa matrix`                                         | Tek kullanımlık bir Tuwunel ana sunucusuna karşı QA Lab Matrix profilleri. Bkz. [Matrix duman testi hatları](#matrix-smoke-lanes).                                                                                                                                   |
| `qa slack`                                          | Gerçek bir özel Slack kanalına karşı canlı aktarım hattı.                                                                                                                                                                                                              |
| `qa telegram`                                       | Gerçek bir özel Telegram grubuna karşı canlı aktarım hattı.                                                                                                                                                                                                            |
| `qa whatsapp`                                       | Gerçek WhatsApp Web hesaplarına karşı canlı aktarım hattı.                                                                                                                                                                                                              |
| `qa mantis`                                         | Discord durum tepkileri kanıtı, Crabbox masaüstü/tarayıcı duman testi ve VNC'de Slack duman testi içeren canlı aktarım hataları öncesi/sonrası doğrulama çalıştırıcısı. Bkz. [Mantis](/tr/concepts/mantis) ve [Mantis Slack Masaüstü Çalıştırma Kılavuzu](/tr/concepts/mantis-slack-desktop-runbook). |

### Profil destekli `qa run`

Profil destekli `qa run`, üyelik bilgilerini `taxonomy.yaml` kaynağından okur ve ardından
çözümlenen senaryoları `qa suite` üzerinden gönderir. `--surface` ve `--category`, ayrı
hatlar tanımlamak yerine seçilen profili filtreler. Ortaya çıkan
`qa-evidence.json`, seçilen kategori sayılarını ve eksik kapsam kimliklerini içeren
bir profil puan kartı özeti içerir; ayrı kanıt girdileri testler, kapsam rolleri ve
sonuçlar için doğruluk kaynağı olmaya devam eder. Taksonomi özellik kapsamı
kimlikleri takma ad değil, kesin kanıt hedefleridir: birincil senaryo kapsamı
eşleşen kimlikleri karşılarken ikincil kapsam yalnızca tavsiye niteliğinde kalır.
Her kapsam kimliği, `taxonomy.yaml` içindeki kısa yüzey kimliği kullanılarak tam olarak
`taxonomy-surface.feature` biçimindedir. Bir senaryonun ayrı `surface` alanı bir yürütme/raporlama
etiketidir (örneğin `channel` veya `runtime-tool`); taksonomi
sahipliğini tanımlamaz.

Daraltılmış kanıt, girdi başına `execution` değerini atlar ve `evidenceMode: "slim"` değerini ayarlar;
`smoke-ci` varsayılan olarak daraltılmış biçimi kullanır ve `--evidence-mode full` tam girdileri geri getirir:

```bash
pnpm openclaw qa run \
  --qa-profile smoke-ci \
  --category channels.conversation-routing-and-delivery \
  --provider-mode mock-openai \
  --output-dir .artifacts/qa-e2e/smoke-ci-profile-dispatch
```

Sahte model sağlayıcıları ve Crabline yerel sağlayıcı sunucularıyla deterministik profil kanıtı
için `smoke-ci` kullanın. Canlı kanallara karşı Stable/LTS kanıtı için
`release` kullanın. `all` yalnızca açıkça tam taksonomi kanıtı istenen
çalıştırmalarda kullanılmalıdır; tüm etkin olgunluk kategorilerini seçer ve `qa_profile=all` ile
`QA
Profile Evidence` GitHub Actions iş akışı üzerinden gönderilebilir. Bir
komut ayrıca bir OpenClaw kök profiline ihtiyaç duyduğunda, kök profilini QA
komutundan önce yerleştirin:

```bash
pnpm openclaw --profile work qa run --qa-profile smoke-ci
```

## Operatör akışı

Geçerli QA operatör akışı, iki bölmeli bir QA sitesidir:

- Sol: Aracının bulunduğu Gateway panosu (Control UI).
- Sağ: Slack benzeri transkripti ve senaryo planını gösteren QA Lab.

Şununla çalıştırın:

```bash
pnpm qa:lab:up
```

Bu işlem QA sitesini oluşturur, Docker destekli gateway hattını başlatır ve
bir operatörün veya otomasyon döngüsünün aracıya bir QA görevi verebileceği,
gerçek kanal davranışını gözlemleyebileceği ve nelerin çalıştığını, başarısız
olduğunu veya engellenmiş kaldığını kaydedebileceği QA Lab sayfasını kullanıma açar.

Docker imajını her seferinde yeniden oluşturmadan daha hızlı QA Lab kullanıcı arayüzü yinelemesi
için yığını, bağlama yoluyla bağlanmış bir QA Lab paketiyle başlatın:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast`, Docker hizmetlerini önceden oluşturulmuş bir imajda tutar ve
`extensions/qa-lab/web/dist` öğesini `qa-lab` kapsayıcısına bağlama yoluyla bağlar.
`qa:lab:watch`, bu paketi değişiklik olduğunda yeniden oluşturur ve QA Lab varlık
karması değiştiğinde tarayıcı otomatik olarak yeniden yüklenir.

### Gözlemlenebilirlik duman testleri

<Note>
Gözlemlenebilirlik QA'sı yalnızca kaynak kodu çalışma kopyasında kalır. npm tarball paketi,
QA Lab'i (ve `qa-channel`) bilerek dışarıda bırakır; bu nedenle paket Docker sürüm
hatları `qa` komutlarını çalıştırmaz. Tanılama enstrümantasyonu
değiştirilirken bunları oluşturulmuş bir kaynak kodu çalışma kopyasından çalıştırın.
</Note>

| Takma ad                                 | Çalıştırdığı işlem                                                                                                                       |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm qa:otel:smoke`                    | Yerel OpenTelemetry alıcısı ile `diagnostics-otel` etkinleştirilmiş `otel-trace-smoke` senaryosu.                                      |
| `pnpm qa:otel:collector-smoke`          | Gerçek bir OpenTelemetry Collector Docker kapsayıcısının arkasındaki aynı hat. Uç nokta bağlantılarını veya toplayıcı/OTLP uyumluluğunu değiştirirken kullanın. |
| `pnpm qa:prometheus:smoke`              | `diagnostics-prometheus` etkinleştirilmiş `docker-prometheus-smoke` senaryosu.                                                           |
| `pnpm qa:observability:smoke`           | `qa:otel:smoke` ve ardından `qa:prometheus:smoke`.                                                                                      |
| `pnpm qa:observability:collector-smoke` | `qa:otel:collector-smoke` ve ardından `qa:prometheus:smoke`.                                                                            |

`qa:otel:smoke` yerel bir OTLP/HTTP alıcısı başlatır, asgari düzeyde bir QA-channel
ajan turu çalıştırır ve ardından izlerin, metriklerin ve günlüklerin dışa aktarıldığını doğrular. Dışa aktarılan
protobuf iz yayılımlarının kodunu çözer ve sürüm açısından kritik yapıyı denetler:
`openclaw.run`, `openclaw.harness.run`, en güncel GenAI anlam kuralına uygun
model çağrısı yayılımı, `openclaw.context.assembled` ve `openclaw.message.delivery`
öğelerinin tümü mevcut olmalıdır. Smoke testi
`OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` kullanımını zorunlu kılar; bu nedenle model çağrısı
yayılımı `{gen_ai.operation.name} {gen_ai.request.model}` adını kullanmalıdır; başarılı turlardaki model
çağrıları `StreamAbandoned` değerini dışa aktarmamalıdır; ham tanılama
kimlikleri ve `openclaw.content.*` öznitelikleri izin dışında kalmalıdır. Senaryo
istemi, modelden sabit bir işaretçiyle yanıt vermesini ve sabit bir gizli
dizeyi açıklamamasını ister; ham OTLP yükleri bunlardan hiçbirini veya senaryo
kimliğinden türetilen QA oturum anahtarını içermemelidir. QA paketi yapıtlarının
yanına `otel-smoke-summary.json` yazar.

`qa:prometheus:smoke`, kimliği doğrulanmamış veri çekme isteklerinin reddedildiğini doğrular ve ardından
kimliği doğrulanmış veri çekme işleminin; istem içeriği, yanıt içeriği, ham tanılama tanımlayıcıları, kimlik doğrulama
token'ları veya yerel yollar olmadan sürüm açısından kritik metrik ailelerini içerdiğini denetler.

### Matrix smoke hatları

Model sağlayıcısı kimlik bilgileri gerektirmeyen, gerçek aktarımlı bir Matrix smoke hattı
için sürüm profilini deterministik sahte OpenAI sağlayıcısıyla çalıştırın:

```bash
pnpm openclaw qa matrix --provider-mode mock-openai --profile release
```

Canlı frontier sağlayıcı hattı için OpenAI uyumlu kimlik bilgilerini
açıkça sağlayın:

```bash
OPENCLAW_LIVE_OPENAI_KEY="${OPENAI_API_KEY}" \
  pnpm openclaw qa matrix --provider-mode live-frontier --profile release
```

Düz `pnpm openclaw qa matrix`, tam `all` profilini çalıştırır ve senaryo
hatalarından sonra devam eder. Daha kısa bir geri bildirim döngüsü için `--fail-fast` kullanın veya
tek tek senaryoları seçmek üzere `--scenario <id>` seçeneğini yineleyin; açık senaryo kimlikleri
`--profile` seçeneğine göre önceliklidir.

| Profil       | Senaryolar | Amaç                                                                                                                                     |
| ------------ | --------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `all`        | 93        | Eksiksiz katalog (varsayılan).                                                                                                           |
| `release`    | 2         | Sürüm açısından kritik kanal temel çizgisi ve canlı izin listesini yeniden yükleme.                                                       |
| `fast`       | 12        | Odaklanmış iş parçacığı, tepkiler, onaylar, politika, bot geçitleme ve şifreli yanıt kapsamı.                                             |
| `transport`  | 50        | İş parçacığı, DM/oda yönlendirme, otomatik katılım, onaylar, tepkiler, yeniden başlatmalar, bahsetme/izin listesi politikası, düzenlemeler ve çok aktörlü sıralama. |
| `media`      | 7         | Görsel, oluşturulan görsel, ses, ek, desteklenmeyen medya ve şifreli medya kapsamı.                                                       |
| `e2ee-smoke` | 8         | Asgari şifreli yanıt, iş parçacığı, önyükleme, kurtarma, yeniden başlatma, redaksiyon ve hata kapsamı.                                    |
| `e2ee-deep`  | 18        | Durum kaybı, yedekleme, anahtar kurtarma, cihaz hijyeni ve SAS/QR/DM doğrulaması.                                                         |
| `e2ee-cli`   | 9         | Harness üzerinden `openclaw matrix encryption setup`, kurtarma anahtarı, çoklu hesap, Gateway gidiş-dönüşü ve öz doğrulama komutları. |

Profil üyeliği ve kanal gereksinimleri, `qa/scenarios/channels/` altındaki bildirimsel Matrix
senaryolarıyla birlikte bulunur. Çalıştırma, kanal sürücüsünü seçer.
Bunların canlı uygulamaları
`extensions/qa-lab/src/live-transports/matrix/scenarios/` altında bulunur.

Bağdaştırıcı, Docker içinde tek kullanımlık bir Tuwunel ana sunucusu (varsayılan
imaj `ghcr.io/matrix-construct/tuwunel:v1.5.1`, sunucu adı `matrix-qa.test`,
port `28008`) hazırlar; geçici sürücü, SUT ve gözlemci kullanıcılarını kaydeder, gerekli
odaları oluşturur ve redakte edilmiş istek/yanıt sınırını kaydeder. Ardından
gerçek Matrix Plugin'ini bu aktarımla kapsamlandırılmış bir alt QA Gateway
içinde çalıştırır (`qa-channel` yoktur) ve ortamı kapatır.

Yaygın seçenekler:

| Bayrak                   | Varsayılan        | Amaç                                                                                 |
| ------------------------ | ----------------- | ------------------------------------------------------------------------------------ |
| `--profile <profile>`    | `all`             | Yukarıdaki profillerden birini seçin.                                                |
| `--scenario <id>`        | -                 | Bir senaryo seçin; yinelenebilir.                                                    |
| `--fail-fast`            | kapalı            | İlk başarısız denetimden veya senaryodan sonra durun.                                |
| `--allow-failures`       | kapalı            | Senaryo hataları için başarısız çıkış kodu döndürmeden yapıtları yazın.               |
| `--provider-mode <mode>` | `live-frontier`   | Deterministik dağıtım için `mock-openai`, canlı sağlayıcı için `live-frontier` kullanın. |
| `--model <ref>`          | sağlayıcı varsayılanı | Birincil `provider/model` referansını ayarlayın.                                      |
| `--alt-model <ref>`      | sağlayıcı varsayılanı | Model değiştiren senaryoların kullandığı alternatif modeli ayarlayın.                 |
| `--fast`                 | kapalı            | Desteklendiği yerlerde sağlayıcının hızlı modunu etkinleştirin.                       |
| `--output-dir <path>`    | oluşturulan       | Rapor dizinini seçin; göreli yollar `--repo-root` temel alınarak çözümlenir.         |
| `--repo-root <path>`     | geçerli dizin     | Tarafsız bir çalışma dizininden çalıştırın.                                           |
| `--sut-account <id>`     | `sut`             | Alt Gateway yapılandırmasında Matrix hesap kimliğini seçin.                           |

Matrix QA, paylaşılan Matrix kimlik bilgilerini kiralamaz: bağdaştırıcı tek kullanımlık
kullanıcıları yerel olarak oluşturduğundan `--credential-source` veya
`--credential-role` kabul etmez. Ana sunucu imajını
`OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE` ile geçersiz kılın; olumsuz yanıtsızlık doğrulamalarını
`OPENCLAW_QA_MATRIX_NO_REPLY_WINDOW_MS` ile ayarlayın (varsayılan `8000`, etkin
senaryo zaman aşımıyla sınırlandırılır). Tek seferlik komut normalde yapıtlar
temizlendikten sonra temiz bir çıkışı zorunlu kılar; çünkü Matrix kripto yerel tanıtıcıları temizlemeden daha uzun yaşayabilir.
Komutun bunun yerine geri dönmesini gerektiren doğrudan test harness'ı için
yalnızca `OPENCLAW_QA_MATRIX_DISABLE_FORCE_EXIT=1` ayarını belirleyin.

Her çalıştırma, seçilen çıktı dizini altında normal QA Lab yapıtlarını
yazar: `qa-suite-report.md`, `qa-suite-summary.json` ve
`qa-evidence.json`. Temizleme başarısız olursa yazdırılan
`docker compose ... down --remove-orphans` kurtarma komutunu çalıştırın. Yavaş çalıştırıcılarda
yanıtsızlık penceresini artırın; hızlı CI ortamında daha küçük bir pencere olumsuz
doğrulamaları kısaltabilir.

Senaryolar, birim testlerinin uçtan uca kanıtlayamadığı aktarım davranışlarını
kapsar: bahsetme geçitleme, botlara izin verme politikaları, izin listeleri, üst düzey ve iş parçacıklı
yanıtlar, DM yönlendirme, tepki işleme, gelen düzenlemeleri engelleme, yeniden başlatma
tekrarlarını tekilleştirme, ana sunucu kesintisinden kurtarma, onay meta verilerinin teslimi,
medya işleme ve Matrix E2EE önyükleme/kurtarma/doğrulama akışları.
E2EE CLI profili ayrıca Gateway
yanıtlarını denetlemeden önce `openclaw matrix encryption setup` ve doğrulama komutlarını aynı tek kullanımlık ana sunucu üzerinden yürütür.

`matrix-room-block-streaming` ve `subagent-thread-spawn`,
açık `--scenario` seçimiyle kullanılabilir durumda kalır ancak varsayılan `all` profilinin dışında tutulur.

CI aynı komut yüzeyini
`.github/workflows/qa-live-transports-convex.yml` içinde kullanır. Zamanlanmış ve sürüm çalıştırmaları
sürüm senaryolarını yürütür. Manuel `matrix_profile=all` dağıtımları
`transport`, `media`, `e2ee-smoke`, `e2ee-deep` ve `e2ee-cli` profillerine ayrılır;
odaklanmış dağıtımlar tek bir işte `fast`, `release` veya `transport` seçer.

### Discord Mantis senaryoları

Discord ayrıca hata yeniden üretimi için yalnızca Mantis'e özel, isteğe bağlı senaryolara sahiptir. Açık durum
tepki zaman çizelgesi için `--scenario discord-status-reactions-tool-only` veya gerçek bir Discord iş parçacığı
oluşturup `message.thread-reply` öğesinin `filePath` ekini
koruduğunu doğrulamak için `--scenario discord-thread-reply-filepath-attachment` kullanın. Bu senaryolar, geniş
smoke kapsamından ziyade öncesi/sonrası yeniden üretim probları oldukları için varsayılan
canlı Discord hattının dışında tutulur. İş parçacığı eki Mantis iş akışı,
QA ortamında `MANTIS_DISCORD_VIEWER_CHROME_PROFILE_DIR` veya
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_TGZ_B64` yapılandırılmışsa oturum açılmış
bir Discord Web tanık videosu da ekleyebilir. Bu görüntüleyici profili yalnızca görsel yakalama
içindir; başarılı/başarısız kararı yine Discord REST oracle'ından gelir.

Diğer gerçek aktarımlı smoke hatları için:

```bash
pnpm openclaw qa discord
pnpm openclaw qa slack
pnpm openclaw qa telegram
pnpm openclaw qa whatsapp
```

Bunlar iki bot veya hesaba (sürücü + SUT) sahip, önceden var olan gerçek bir kanalı hedefler.
Bu dört aktarım için gerekli ortam değişkenleri, senaryo listeleri, çıktı yapıtları ve Convex
kimlik bilgisi havuzu aşağıdaki
[Discord, Slack, Telegram ve WhatsApp QA referansı](#discord-slack-telegram-and-whatsapp-qa-reference)
bölümünde belgelenmiştir.

### Mantis Slack masaüstü ve görsel görev çalıştırıcıları

VNC kurtarmalı eksiksiz bir Slack masaüstü sanal makine çalıştırması için şunu çalıştırın:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

Bu komut bir Crabbox masaüstü/tarayıcı makinesi kiralar, Slack canlı
hattını VM içinde çalıştırır, VNC tarayıcısında Slack Web'i açar, masaüstünü
kaydeder ve `slack-qa/`, `slack-desktop-smoke.png` ile
`slack-desktop-smoke.mp4` dosyalarını (video kaydı kullanılabildiğinde)
Mantis yapıt dizinine geri kopyalar. Crabbox masaüstü/tarayıcı kiralamaları,
kayıt araçlarını ve tarayıcı/yerel derleme yardımcı paketlerini baştan sağlar;
bu nedenle senaryo yalnızca eski kiralamalarda yedek seçenekleri kurmalıdır.
Mantis, toplam ve aşama başına süreleri `mantis-slack-desktop-smoke-report.md` içinde bildirir;
böylece yavaş çalıştırmalarda sürenin kiralama hazırlığına, kimlik bilgisi
edinmeye, uzak kuruluma veya yapıt kopyalamaya mı harcandığı görülebilir.
VNC üzerinden Slack Web'de elle oturum açtıktan sonra `--lease-id <cbx_...>`
öğesini yeniden kullanın; yeniden kullanılan kiralamalar Crabbox'ın pnpm
depo önbelleğini de sıcak tutar. Varsayılan `--hydrate-mode source`, bir kaynak
çalışma kopyasından doğrulama yapar ve kurulumu/derlemeyi VM içinde çalıştırır.
`--hydrate-mode prehydrated` öğesini yalnızca yeniden kullanılan uzak çalışma alanında
zaten `node_modules` ve derlenmiş bir `dist/` bulunduğunda
kullanın; bu mod maliyetli kurulum/derleme adımını atlar ve çalışma alanı
hazır olmadığında güvenli biçimde başarısız olur. `--gateway-setup` ile
Mantis, VM içinde `38973` bağlantı noktasında kalıcı bir
OpenClaw Slack Gateway'i çalışır durumda bırakır; bu seçenek olmadan komut,
normal bottan bota Slack QA hattını çalıştırır ve yapıt kaydından sonra çıkar.

Masaüstü kanıtıyla yerel Slack onay kullanıcı arayüzünü doğrulamak için Mantis
onay denetim noktası modunu çalıştırın:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer
```

Bu mod, `--gateway-setup` ile birlikte kullanılamaz. Slack onay senaryolarını
çalıştırır, onay dışı senaryo kimliklerini reddeder, bekleyen ve çözümlenen her
onay durumunda bekler, gözlemlenen Slack API iletisini
`approval-checkpoints/<scenario>-pending.png` ve
`approval-checkpoints/<scenario>-resolved.png` içine işler; ardından herhangi bir denetim noktası,
ileti kanıtı, alındı bildirimi veya işlenmiş ekran görüntüsü eksik ya da
boşsa başarısız olur. Soğuk CI kiralamalarında
`slack-desktop-smoke.png` hâlâ Slack oturum açma ekranını gösterebilir; onay denetim
noktası görüntüleri bu hattın görsel kanıtıdır.

Varsayılan denetim noktası çalıştırması, iki standart Slack onay senaryosunu
korur. İsteğe bağlı Codex onay yollarından birini kaydetmek için
`--scenario slack-codex-approval-exec-native` veya
`--scenario slack-codex-approval-plugin-native` ile açıkça seçin; Mantis ikisini de kabul eder ve
aynı bekleyen/çözümlenen ekran görüntüsü çiftini oluşturur. Çalıştırıcı,
tam onay, aracının tamamlanması ve çözümlenen güncelleme dizisinin
bitirilebilmesi için seçilen her Codex yolunda denetim noktası ve uzak komut
zaman aşımı sürelerini uzatır.

Operatör kontrol listesi, GitHub iş akışı gönderim komutu, kanıt yorumu
sözleşmesi, hydrate modu karar tablosu, süre yorumlama ve hata işleme
adımları
[Mantis Slack Masaüstü Çalıştırma Kılavuzu](/tr/concepts/mantis-slack-desktop-runbook)
içinde yer alır.

Aracı/CV tarzı bir masaüstü görevi için şunu çalıştırın:

```bash
pnpm openclaw qa mantis visual-task \
  --browser-url https://example.net \
  --expect-text "Example Domain" \
  --vision-model openai/gpt-5.6-luna
```

`visual-task` bir Crabbox masaüstü/tarayıcı makinesi kiralar veya yeniden
kullanır, `crabbox record --while` başlatır, görünür tarayıcıyı iç içe geçmiş bir
`visual-driver` üzerinden yönetir, `visual-task.png` kaydeder,
`--vision-mode image-describe` seçildiğinde ekran görüntüsüne karşı
`openclaw infer image
describe` çalıştırır ve `visual-task.mp4`, `mantis-visual-task-summary.json`,
`mantis-visual-task-driver-result.json` ile
`mantis-visual-task-report.md` dosyalarını yazar. `--expect-text` ayarlandığında görsel
istemi yapılandırılmış bir JSON kararı (`visible`, `evidence`, `reason`)
ister ve yalnızca model, beklenen metne atıfta bulunan kanıtla birlikte
`visible: true` bildirdiğinde geçer; hedef metni yalnızca alıntılayan bir
`visible: false` yanıtı yine de doğrulamadan geçemez. Görüntü anlama
sağlayıcısını çağırmadan masaüstü, tarayıcı, ekran görüntüsü ve video
altyapısını doğrulayan modelsiz bir duman testi için `--vision-mode metadata`
kullanın. Kayıt, `visual-task` için zorunlu bir yapıttır; Crabbox boş
olmayan bir `visual-task.mp4` kaydetmezse, görsel sürücü başarılı olsa bile
görev başarısız olur. Başarısızlık durumunda görev zaten başarılı olmadıkça
ve `--keep-lease` ayarlanmamış olmadıkça Mantis, VNC için kiralamayı
korur.

### Kimlik bilgisi havuzu durum denetimi

Havuzdaki canlı kimlik bilgilerini kullanmadan önce şunu çalıştırın:

```bash
pnpm openclaw qa credentials doctor
```

Doctor, Convex aracısı ortam değişkenlerini (`OPENCLAW_QA_CONVEX_SITE_URL`,
`OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX`) denetler, uç nokta ayarlarını doğrular,
`OPENCLAW_QA_CONVEX_SECRET_CI` ve
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` için yalnızca ayarlı/eksik durumunu bildirir ve
bakımcı gizli anahtarı mevcut olduğunda yönetici/liste erişilebilirliğini
doğrular.

## Standart senaryo kapsamı

Kök `taxonomy.yaml`, anlamsal kapsam kimliklerini tanımlar.
`qa/scenarios/` altındaki senaryo YAML dosyaları, her senaryoyu bu
kimliklerle eşleştirir ve yürütme meta verilerinin sahibidir:
`channel` tek kanal gereksinimidir ve `profiles` adlandırılmış
çalıştırma üyeliğini bildirir. Kanal sürücüsü, çalıştırma düzeyinde birbirinin
yerine kullanılabilen bir uygulama tercihidir. TypeScript çalıştırıcıları bu
kataloğu sorgular; paralel senaryo veya kapsam envanterleri tutmaz.

Statik `qa coverage` çıktısı, taksonomiden senaryoya eşlemeyi bildirir.
Gerçek kanıt, yürütülen senaryoyu, kapsam kimliklerini, kanalı, gerçekten
kullanılan sürücüyü ve sonucu kaydeden `qa-evidence.json` kaynağından gelir.
Kanal ve sürücü, ek kapsam kimliği sözlükleri veya senaryo uygunluk eksenleri
değil, rapor boyutlarıdır.

Docker'ı QA yoluna katmadan tek kullanımlık bir Linux VM hattı için şunu
çalıştırın:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

Bu komut yeni bir Multipass konuk sistemi başlatır, bağımlılıkları kurar,
OpenClaw'ı konuk sistem içinde derler, `qa suite` çalıştırır ve
ardından normal QA raporunu ve özetini ana makinedeki `.artifacts/qa-e2e/...`
konumuna geri kopyalar. Ana makinedeki `qa suite` ile aynı senaryo
seçme davranışını yeniden kullanır.

Ana makine ve Multipass paket çalıştırmaları, varsayılan olarak birden fazla
seçili senaryoyu yalıtılmış Gateway çalışanlarıyla paralel yürütür.
`qa-channel`, seçili senaryo sayısıyla sınırlı olmak üzere varsayılan
olarak 4 eşzamanlılık kullanır. Çalışan sayısını ayarlamak için
`--concurrency
<count>`, seri yürütme içinse `--concurrency 1` kullanın.
Kişisel asistan karşılaştırma paketi (10 senaryo) için `--pack personal-agent`
kullanın. Paket seçici, yinelenen `--scenario` bayraklarına eklenir:
önce açıkça belirtilen senaryolar, ardından yinelenenler kaldırılarak paket
sırasındaki senaryolar çalıştırılır. Özel bir QA çalıştırıcısı OpenTelemetry
toplayıcı kurulumunu zaten sağlıyorsa `otel-trace-smoke` ve
`docker-prometheus-smoke` senaryolarını birlikte seçmek için `--pack observability`
kullanın.

Herhangi bir senaryo başarısız olduğunda komut sıfır olmayan bir kodla çıkar.
Başarısız bir çıkış kodu olmadan yapıtları almak istediğinizde
`--allow-failures` kullanın.

Canlı çalıştırmalar, konuk sistem için uygun olan desteklenen QA kimlik
doğrulama girdilerini iletir: ortam değişkeni tabanlı sağlayıcı anahtarları,
QA canlı sağlayıcı yapılandırma yolu ve mevcut olduğunda
`CODEX_HOME`. Konuk sistemin bağlı çalışma alanı üzerinden geri
yazabilmesi için `--output-dir` öğesini depo kökü altında tutun.

## Discord, Slack, Telegram ve WhatsApp QA başvurusu

Matrix bağdaştırıcısı, yukarıda belgelenen tek kullanımlık Docker destekli
hattı kullanır. Discord, Slack, Telegram ve WhatsApp önceden var olan gerçek
aktarımlara karşı çalıştığından bunların başvurusu burada yer alır.

### Paylaşılan CLI bayrakları

Bu hatlar
`extensions/qa-lab/src/live-transports/shared/live-transport-cli.ts` üzerinden kaydolur ve
aynı bayrakları kabul eder:

| Bayrak                                | Varsayılan                                         | Açıklama                                                                                                                                        |
| ------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `--scenario <id>`                     | -                                                  | Yalnızca bu senaryoyu çalıştırır. Tekrarlanabilir.                                                                                               |
| `--output-dir <path>`                 | `<repo>/.artifacts/qa-e2e/<transport>-<timestamp>` | Raporların, özetlerin, kanıtların, aktarıma özgü yapıtların ve çıktı günlüğünün yazıldığı yer. Göreli yollar `--repo-root` temel alınarak çözümlenir. |
| `--repo-root <path>`                  | `process.cwd()`                                    | Tarafsız bir çalışma dizininden çağırırken depo kökü.                                                                                            |
| `--sut-account <id>`                  | `sut`                                              | QA Gateway yapılandırmasındaki geçici hesap kimliği.                                                                                             |
| `--provider-mode <mode>`              | `live-frontier`                                    | `mock-openai`, `aimock` veya `live-frontier`.                                                                                                   |
| `--model <ref>` / `--alt-model <ref>` | sağlayıcı varsayılanı                                | Birincil/alternatif model başvuruları.                                                                                                           |
| `--fast`                              | kapalı                                             | Desteklendiği yerlerde sağlayıcının hızlı modu.                                                                                                  |
| `--credential-source <env\|convex>`   | `env`                                              | Bkz. [Convex kimlik bilgisi havuzu](#convex-credential-pool).                                                                                    |
| `--credential-role <maintainer\|ci>`  | CI'da `ci`, aksi durumda `maintainer`                 | `--credential-source convex` olduğunda kullanılan rol.                                                                                          |
| `--allow-failures`                    | kapalı                                             | Senaryolar başarısız olduğunda başarısız bir çıkış kodu döndürmeden yapıtları yazar.                                                             |

Her hat, herhangi bir senaryo başarısız olduğunda sıfır olmayan bir kodla
çıkar. `--allow-failures`, başarısız bir çıkış kodu ayarlamadan yapıtları
yazar. Telegram ayrıca kullanılabilir senaryo kimliklerini yazdırıp çıkmak
için `--list-scenarios` öğesini kabul eder; diğer hatlar bu bayrağı sunmaz.

### Telegram QA

```bash
pnpm openclaw qa telegram
```

İki farklı botun (sürücü + SUT) bulunduğu gerçek bir özel Telegram grubunu
hedefler. SUT botunun bir Telegram kullanıcı adı olmalıdır; bottan bota
gözlem, her iki botta da `@BotFather` içindeki
**Bot-to-Bot Communication Mode** etkinleştirildiğinde en iyi sonucu verir.

`--credential-source env` olduğunda gerekli ortam değişkenleri:

- `OPENCLAW_QA_TELEGRAM_GROUP_ID` - sayısal sohbet kimliği (dize).
- `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`

`release` profili, bakımı yapılan Telegram YAML senaryolarını seçer;
`all` ise isteğe bağlı oturum, kullanım, yanıt zinciri ve akış
stres denetimlerini ekler. Açık `--scenario` değerleri profili geçersiz
kılar.

- `channel-canary`
- `channel-mention-gating`
- `telegram-help-command`
- `telegram-commands-command`
- `telegram-tools-compact-command`
- `telegram-whoami-command`
- `telegram-status-command`
- `telegram-repeated-command-authorization`
- `telegram-other-bot-command-gating`
- `telegram-context-command`
- `telegram-current-session-status-tool`
- `telegram-tool-only-usage-footer`
- `telegram-reply-chain-exact-marker`
- `telegram-stream-final-single-message`
- `telegram-long-final-reuses-preview`
- `telegram-long-final-three-chunks`

`release` profili her zaman canary, bahsetme geçitlemesi, yerel komut
yanıtları, komut adresleme ve botlar arası grup yanıtlarını kapsar. `mock-openai`
ayrıca deterministik uzun son yanıt önizleme denetimini içerir.
`telegram-current-session-status-tool` ve
`telegram-tool-only-usage-footer` isteğe bağlı kalır: ilki yalnızca
canary'nin hemen ardından iş parçacığına bağlandığında kararlıdır; ikincisi ise yalnızca araç yanıtlarındaki
`/usage` altbilgisinin gerçek Telegram
kanıtıdır. Geçerli varsayılan/isteğe bağlı ayrımını regresyon referanslarıyla yazdırmak için `pnpm openclaw qa telegram
--list-scenarios --provider-mode mock-openai` kullanın.
Her Telegram canlı bağdaştırıcı senaryosu için `--profile all` kullanın.

Çıktı yapıtları:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - profil, kapsam, sağlayıcı, kanal, yapıtlar, sonuç ve RTT
  alanları dâhil olmak üzere canlı aktarım denetimlerine ilişkin kanıt girdileri.

Paket Telegram çalıştırmaları aynı Telegram kimlik bilgisi sözleşmesini kullanır. Tekrarlanan RTT
ölçümü, normal paket Telegram canlı hattının bir parçasıdır; RTT
dağılımı, seçilen RTT denetimi için `result.timing` altında `qa-evidence.json` içine
katılır.

```bash
OPENCLAW_QA_CREDENTIAL_SOURCE=convex \
pnpm test:docker:npm-telegram-live
```

`OPENCLAW_QA_CREDENTIAL_SOURCE=convex` ayarlandığında paket canlı sarmalayıcısı
bir `kind: "telegram"` kimlik bilgisi kiralar, kiralanan grup/sürücü/SUT
bot ortamını kurulu paket çalıştırmasına aktarır, kiralamaya Heartbeat gönderir ve kapanışta
kiralamayı serbest bırakır. Paket sarmalayıcısı varsayılan olarak
`channel-canary` için 20 RTT denetimi, 30s RTT zaman aşımı ve Convex seçildiğinde CI dışında
`maintainer` Convex rolünü kullanır. Ayrı bir RTT komutu veya Telegram'a özgü özet biçimi
oluşturmadan RTT ölçümünü ayarlamak için
`OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`, `OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS`
ya da `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES` değerini geçersiz kılın.

### Discord QA

```bash
pnpm openclaw qa discord
```

İki botun bulunduğu gerçek bir özel Discord sunucu kanalını hedefler: test düzeneği tarafından
denetlenen bir sürücü botu ve paketle birlikte gelen Discord plugin'i aracılığıyla alt OpenClaw Gateway
tarafından başlatılan bir SUT botu. Kanal bahsetme işlemesini, SUT botunun yerel
`/help` komutunu Discord'a kaydettiğini ve isteğe bağlı Mantis kanıt senaryolarını
doğrular.

`--credential-source env` olduğunda gerekli ortam değişkenleri:

- `OPENCLAW_QA_DISCORD_GUILD_ID`
- `OPENCLAW_QA_DISCORD_CHANNEL_ID`
- `OPENCLAW_QA_DISCORD_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_SUT_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_SUT_APPLICATION_ID` - Discord tarafından döndürülen SUT bot kullanıcı kimliğiyle
  eşleşmelidir (aksi takdirde hat hızla başarısız olur).

İsteğe bağlı:

- `OPENCLAW_QA_DISCORD_VOICE_CHANNEL_ID`, `discord-voice-autojoin` için ses/sahne kanalını seçer;
  bu olmadan senaryo, SUT botunun görebildiği ilk
  ses/sahne kanalını seçer.

Discord YAML modülü senaryoları (`qa/scenarios/channels/discord-*.yaml`):

- `discord-canary`
- `discord-mention-gating`
- `discord-native-help-command-registration`
- `discord-voice-autojoin` - isteğe bağlı ses senaryosu. Tek başına çalışır,
  `channels.discord.voice.autoJoin` özelliğini etkinleştirir ve SUT botunun geçerli
  Discord ses durumunun hedef ses/sahne kanalı olduğunu doğrular. Convex Discord
  kimlik bilgileri isteğe bağlı `voiceChannelId` içerebilir; aksi takdirde çalıştırıcı
  bağdaştırıcısı sunucudaki ilk görünür ses/sahne kanalını keşfeder.
- `discord-status-reactions-tool-only` - isteğe bağlı Mantis senaryosu. SUT'yi
  `messages.statusReactions.enabled=true` ile her zaman açık, yalnızca araç kullanan sunucu yanıtlarına
  geçirdiği için tek başına çalışır; ardından bir REST
  tepki zaman çizelgesinin yanı sıra HTML/PNG görsel yapıtlarını yakalar. Mantis öncesi/sonrası
  raporları, senaryo tarafından sağlanan MP4 yapıtlarını da `baseline.mp4`
  ve `candidate.mp4` olarak korur.
- `discord-thread-reply-filepath-attachment` - isteğe bağlı Mantis senaryosu; bkz.
  [Discord Mantis senaryoları](#discord-mantis-scenarios).

Discord ses kanalına otomatik katılma senaryosunu açıkça çalıştırın:

```bash
pnpm openclaw qa discord \
  --scenario discord-voice-autojoin \
  --provider-mode mock-openai
```

Mantis durum-tepki senaryosunu açıkça çalıştırın:

```bash
pnpm openclaw qa discord \
  --scenario discord-status-reactions-tool-only \
  --provider-mode live-frontier \
  --model openai/gpt-5.6-luna \
  --alt-model openai/gpt-5.6-luna \
  --fast
```

Çıktı yapıtları:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - canlı aktarım denetimlerine ilişkin kanıt girdileri.
- `discord-qa-reaction-timelines.json` ve durum-tepki
  senaryosu çalıştırıldığında `discord-status-reactions-tool-only-timeline.png`.

### Slack QA

```bash
pnpm openclaw qa slack
```

İki ayrı botun bulunduğu gerçek bir özel Slack kanalını hedefler: test düzeneği tarafından
denetlenen bir sürücü botu ve paketle birlikte gelen Slack plugin'i aracılığıyla alt OpenClaw Gateway
tarafından başlatılan bir SUT botu.

`--credential-source env` olduğunda gerekli ortam değişkenleri:

- `OPENCLAW_QA_SLACK_CHANNEL_ID`
- `OPENCLAW_QA_SLACK_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_APP_TOKEN`

İsteğe bağlı:

- `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_DIR`, Mantis için görsel onay
  kontrol noktalarını etkinleştirir. Bağdaştırıcı `<scenario>.pending.json` ve
  `<scenario>.resolved.json` dosyalarını yazar, ardından eşleşen `.ack.json` dosyalarını bekler.
- `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_TIMEOUT_MS`, kontrol noktası
  onay zaman aşımını geçersiz kılar. Varsayılan değer `120000` şeklindedir.

Slack canlı bağdaştırıcısı aracılığıyla sunulan standart YAML senaryoları:

- `thread-follow-up`
- `thread-isolation`

Slack YAML modülü senaryoları (`qa/scenarios/channels/slack-*.yaml`):

- `slack-canary`
- `slack-mention-gating`
- `slack-allowlist-block`
- `slack-channel-disabled-warning` - yapılandırılmış, devre dışı bir kanalın yanıt vermeden
  yapılandırılmış bir uyarı yaydığını doğrulayan isteğe bağlı gerçek Slack sondası.
- `slack-top-level-reply-shape`
- `slack-restart-resume`
- `slack-progress-commentary-true`, `slack-progress-commentary-false`,
  `slack-progress-commentary-omitted` ve
  `slack-progress-commentary-verbose-dedupe` - bağımsız yorum/araç ilerleme denetimleri, anahtarın atlandığı
  eski varsayılan ve kalıcı ayrıntılı ilerleme açıkken tek teslim davranışı için
  isteğe bağlı gerçek Slack sondaları.
- `slack-reaction-glyph-native` - isteğe bağlı canlı mesaj aracı tepki senaryosu.
  Aracı tam olarak `✅` glifini iletecek şekilde yönlendirir ve Slack'in hedef mesajda
  SUT botu için `white_check_mark` değerini depoladığını doğrular.
- `slack-chart-presentation-native` - yerel `data_visualization` bloğunu ve tam erişilebilir metni
  doğrulayan isteğe bağlı taşınabilir grafik senaryosu.
- `slack-table-presentation-native` - yerel `data_table` bloğunu, tam satırları ve erişilebilir metni
  doğrulayan isteğe bağlı taşınabilir tablo senaryosu.
- `slack-table-invalid-blocks-fallback` - 101 veri satırı ve başlığıyla birlikte yapısal olarak
  okunabilir, sınırı aşan ham bir tabloyu üretim Slack gönderim yolu üzerinden gönderen,
  Slack'in kendisinin `invalid_blocks` döndürdüğünü kanıtlayan ve depolanan biçimlendirmesi
  devre dışı bırakılmış geri dönüşün eksiksiz olduğunu ve yerel veri bloğu içermediğini doğrulayan
  isteğe bağlı doğrudan aktarım senaryosu. Senaryo ayrıntıları yalnızca güvenli hata kodu, sayı ve
  boole kanıtlarını tutar.
- `slack-approval-exec-native` - isteğe bağlı yerel Slack exec onay senaryosu.
  Gateway üzerinden bir exec onayı ister, Slack mesajının yerel onay düğmelerine
  sahip olduğunu doğrular, onayı çözümler ve çözümlenmiş Slack
  güncellemesini doğrular.
- `slack-approval-plugin-native` - isteğe bağlı yerel Slack plugin onay
  senaryosu. Plugin olaylarının exec onay yönlendirmesi tarafından bastırılmaması için exec ve plugin
  onay iletimini birlikte etkinleştirir, ardından aynı bekleyen/çözümlenmiş yerel Slack kullanıcı arayüzü
  yolunu doğrular.
- `slack-codex-approval-exec-native` - isteğe bağlı Codex Guardian komut onay
  senaryosu. Codex plugin'ini Guardian modunda etkinleştirir, Slack kaynaklı bir Gateway aracı
  dönüşünü Codex uygulama sunucusu test düzeneği üzerinden yönlendirir,
  `openclaw-codex-app-server` için yerel Slack plugin onay istemini bekler,
  onayı çözümler ve Codex dönüşünün beklenen komut çıktısı ve asistan işaretçileriyle
  tamamlandığını doğrular.
- `slack-codex-approval-plugin-native` - isteğe bağlı Codex Guardian dosya onay
  senaryosu. Codex'in uygulama sunucusu dosya değişikliği onay yolunu yayması için çalışma alanı dışındaki
  bir `apply_patch` talimatını kullanır; ardından aynı yerel Slack bekleyen/çözümlenmiş
  onay yolunu, son asistan işaretçisini ve temizlemeden önce tam dosya
  içeriklerini doğrular.

Codex onay senaryoları bir `openai/*` veya `codex/*` `--model`, normal
canlı model kimlik bilgileri ve Codex plugin'i tarafından kabul edilen Codex kimlik doğrulaması
veya API anahtarı kimlik doğrulaması gerektirir. Senaryo ayrıntıları, gizlenmiş Slack onay
meta verilerinin yanı sıra Codex uygulama sunucusu yöntemini, seçilen Codex model
anahtarını, son Codex dönüş durumunu ve işlem işaretçisi doğrulamasını içerir.

Çıktı yapıtları:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - canlı aktarım denetimlerine ilişkin kanıt girdileri.
- `approval-checkpoints/` - yalnızca Mantis
  `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_DIR` ayarladığında; kontrol noktası JSON'ını,
  onay JSON'ını ve bekleyen/çözümlenmiş ekran görüntülerini içerir.

#### Slack çalışma alanını ayarlama

Hat, tek bir çalışma alanında iki ayrı Slack uygulamasına ve her iki
botun da üyesi olduğu bir kanala ihtiyaç duyar:

- `channelId` - her iki botun da davet edildiği bir kanalın
  `Cxxxxxxxxxx` kimliği. Özel bir kanal kullanın; hat her çalıştırmada gönderi yayınlar.
- `driverBotToken` - **Sürücü** uygulamasının bot belirteci (`xoxb-...`).
- `sutBotToken` - **SUT** uygulamasının bot belirteci (`xoxb-...`);
  bot kullanıcı kimliğinin farklı olması için sürücüden ayrı bir Slack uygulaması olmalıdır.
- `sutAppToken` - SUT uygulamasının `connections:write` kapsamına sahip
  uygulama düzeyi belirteci (`xapp-...`); SUT uygulamasının olayları alabilmesi için
  Socket Mode tarafından kullanılır.

Üretim çalışma alanını yeniden kullanmak yerine QA'ya ayrılmış bir Slack çalışma alanını
tercih edin.

Aşağıdaki SUT manifesti, paketle birlikte gelen Slack plugin'inin üretim kurulumunu
(`extensions/slack/src/setup-shared.ts:12`) kasıtlı olarak canlı Slack QA paketinin kapsadığı
izinler ve olaylarla sınırlar. Kullanıcıların gördüğü üretim kanalı kurulumu için
[Slack kanalının hızlı kurulumuna](/tr/channels/slack#quick-setup) bakın; hat aynı çalışma alanında
iki ayrı bot kullanıcı kimliği gerektirdiği için QA Sürücü/SUT
çifti kasıtlı olarak ayrıdır.

**1. Sürücü uygulamasını oluşturun**

[api.slack.com/apps](https://api.slack.com/apps) → _Create New App_ →
_From a manifest_ bölümüne gidin → QA çalışma alanını seçin, aşağıdaki manifesti yapıştırın,
ardından _Install to Workspace_ seçeneğini kullanın:

```json
{
  "display_information": {
    "name": "OpenClaw QA Sürücüsü",
    "description": "OpenClaw QA Slack canlı hattı için test sürücüsü botu"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw QA Sürücüsü",
      "always_online": true
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": ["chat:write", "channels:history", "groups:history", "users:read"]
    }
  },
  "settings": {
    "socket_mode_enabled": false
  }
}
```

_Bot User OAuth Token_ (`xoxb-...`) değerini kopyalayın; bu değer
`driverBotToken` olur. Sürücünün yalnızca mesaj göndermesi ve kendisini tanımlaması
gerekir; olaylar ve Socket Mode gerekmez.

**2. SUT uygulamasını oluşturun**

Aynı çalışma alanında _Create New App → From a manifest_ işlemini tekrarlayın. Bu QA uygulaması,
paketle birlikte gelen Slack plugin'inin üretim manifestinin (`extensions/slack/src/setup-shared.ts:12`)
kasıtlı olarak daha dar bir sürümünü kullanır: canlı Slack QA paketi henüz tepki işlemeyi
kapsamadığından tepki kapsamları ve olayları atlanır.

```json
{
  "display_information": {
    "name": "OpenClaw QA SUT",
    "description": "OpenClaw QA SUT connector for OpenClaw"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw QA SUT",
      "always_online": true
    },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed"
      ]
    }
  }
}
```

Slack uygulamayı oluşturduktan sonra ayarlar sayfasında iki işlem yapın:

- _Install to Workspace_ → _Bot User OAuth Token_ değerini kopyalayın → bu,
  `sutBotToken` olur.
- _Basic Information → App-Level Tokens → Generate Token and Scopes_ → 
  `connections:write` kapsamını ekleyin → kaydedin → `xapp-...` değerini kopyalayın → bu,
  `sutAppToken` olur.

Her token üzerinde `auth.test` çağrısı yaparak iki botun farklı kullanıcı kimliklerine sahip olduğunu doğrulayın. Çalışma zamanı, sürücü ile SUT'yi kullanıcı kimliğine göre ayırt eder; aynı uygulamanın
her ikisi için yeniden kullanılması, bahsetme denetiminin hemen başarısız olmasına neden olur.

**3. Kanalı oluşturma**

QA çalışma alanında bir kanal (ör. `#openclaw-qa`) oluşturun ve kanalın içinden her iki
botu davet edin:

```text
/invite @OpenClaw QA Driver
/invite @OpenClaw QA SUT
```

`Cxxxxxxxxxx` kimliğini _channel info → About → Channel ID_ bölümünden kopyalayın; bu,
`channelId` olur. Herkese açık bir kanal kullanılabilir; özel bir kanal kullanırsanız
her iki uygulamada da zaten `groups:history` bulunduğundan, test düzeneğinin geçmiş okumaları
yine başarılı olur.

**4. Kimlik bilgilerini kaydetme**

İki seçenek vardır. Tek makineli hata ayıklama için ortam değişkenlerini kullanın (dört
`OPENCLAW_QA_SLACK_*` değişkenini ayarlayın ve `--credential-source env` iletin) veya CI ve diğer bakımcıların kiralayabilmesi için
paylaşılan Convex havuzunu başlangıç verileriyle doldurun.

Convex havuzu için dört alanı bir JSON dosyasına yazın:

```json
{
  "channelId": "Cxxxxxxxxxx",
  "driverBotToken": "xoxb-...",
  "sutBotToken": "xoxb-...",
  "sutAppToken": "xapp-..."
}
```

Kabuğunuzda `OPENCLAW_QA_CONVEX_SITE_URL` ve `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`
dışa aktarılmış durumdayken kaydedin ve doğrulayın:

```bash
pnpm openclaw qa credentials add \
  --kind slack \
  --payload-file slack-creds.json \
  --note "QA Slack pool seed"

pnpm openclaw qa credentials list --kind slack --status all --json
```

`count: 1`, `status: "active"` beklenir; `lease` alanı bulunmamalıdır.

**5. Uçtan uca doğrulama**

Her iki botun da aracı üzerinden birbiriyle iletişim kurabildiğini doğrulamak için hattı
yerel olarak çalıştırın:

```bash
pnpm openclaw qa slack \
  --credential-source convex \
  --credential-role maintainer \
  --output-dir .artifacts/qa-e2e/slack-local
```

Başarılı bir çalıştırma 30 saniyeden çok daha kısa sürede tamamlanır ve `qa-suite-report.md`,
hem `slack-canary` hem de `slack-mention-gating` için `pass` durumunu gösterir. Hat
yaklaşık 90 saniye boyunca takılı kalıp `Convex credential pool exhausted
for kind "slack"` ile sonlanırsa havuz boştur veya tüm satırlar kiralanmıştır; hangisinin geçerli olduğunu `qa
credentials list --kind slack --status all --json` bildirir.

### WhatsApp QA

```bash
pnpm openclaw qa whatsapp
```

İki özel WhatsApp Web hesabını hedefler: test düzeneği tarafından denetlenen bir sürücü hesabı
ve paketle gelen WhatsApp plugini üzerinden alt OpenClaw gateway'i tarafından başlatılan bir SUT hesabı.

`--credential-source env` kullanıldığında gerekli ortam değişkenleri:

- `OPENCLAW_QA_WHATSAPP_DRIVER_PHONE_E164`
- `OPENCLAW_QA_WHATSAPP_SUT_PHONE_E164`
- `OPENCLAW_QA_WHATSAPP_DRIVER_AUTH_ARCHIVE_BASE64`
- `OPENCLAW_QA_WHATSAPP_SUT_AUTH_ARCHIVE_BASE64`

İsteğe bağlı:

- `OPENCLAW_QA_WHATSAPP_GROUP_JID`; `whatsapp-mention-gating`, `whatsapp-group-pending-history-context`,
  `whatsapp-broadcast-group-fanout`, `whatsapp-group-activation-always`,
  `whatsapp-group-reply-to-bot-triggers`, grup eylemi/medya/anket senaryoları
  ve `whatsapp-group-allowlist-block` gibi grup senaryolarını etkinleştirir.

WhatsApp YAML senaryoları (`qa/scenarios/channels/whatsapp-*.yaml`):

- Temel davranış ve grup denetimi: `whatsapp-canary`, `whatsapp-pairing-block`,
  `whatsapp-mention-gating`, `whatsapp-group-pending-history-context`,
  `whatsapp-group-activation-always`, `whatsapp-group-reply-to-bot-triggers`,
  `whatsapp-top-level-reply-shape`, `whatsapp-restart-resume`,
  `whatsapp-group-allowlist-block`.
- Yerel komutlar: `whatsapp-help-command`, `whatsapp-status-command`,
  `whatsapp-commands-command`, `whatsapp-tools-compact-command`,
  `whatsapp-whoami-command`, `whatsapp-context-command`,
  `whatsapp-native-new-command`.
- Yanıt ve nihai çıktı davranışı: `whatsapp-tool-only-usage-footer`,
  `whatsapp-reply-to-message`, `whatsapp-group-reply-to-message`,
  `whatsapp-reply-to-mode-batched`, `whatsapp-reply-context-isolation`,
  `whatsapp-reply-delivery-shape`, `whatsapp-stream-final-message-accounting`.
- Kullanıcı yolu mesaj eylemleri: `whatsapp-agent-message-action-react`, gerçek bir sürücü DM'sinden
  başlar, modelin `message` aracını çağırmasına izin verir ve
  yerel WhatsApp tepkisini gözlemler. `whatsapp-agent-message-action-upload-file`,
  `message(action=upload-file)` için aynı yaklaşımı kullanır ve
  yerel WhatsApp medyasını gözlemler. `whatsapp-group-agent-message-action-react` ve
  `whatsapp-group-agent-message-action-upload-file`, aynı
  kullanıcı tarafından görülebilen eylemleri gerçek bir WhatsApp grubunda kanıtlar.
- Grup yayılımı: `whatsapp-broadcast-group-fanout`, bahsetme içeren tek bir
  WhatsApp grup mesajından başlar ve `main`
  ile `qa-second` tarafından verilen farklı görünür yanıtları doğrular.
- Grup etkinleştirme: `whatsapp-group-activation-always`, gerçek bir grup
  oturumunu `/activation always` olarak değiştirir, bahsetme içermeyen bir grup mesajının
  agent'ı uyandırdığını kanıtlar ve ardından `/activation mention` ayarını geri yükler.
  `whatsapp-group-reply-to-bot-triggers`, bir bot yanıtını başlangıç verisi olarak ekler, açık bir bahsetme olmadan
  bu yanıta yerel bir alıntılı yanıt gönderir ve agent'ın
  bu yanıt bağlamından uyandığını doğrular.
- Gelen medya ve yapılandırılmış mesajlar: `whatsapp-inbound-image-caption`,
  `whatsapp-audio-preflight`, `whatsapp-inbound-structured-messages`,
  `whatsapp-group-audio-gating`, `whatsapp-inbound-reaction-no-trigger`.
  Bunlar, gerçek WhatsApp görsel, ses, belge, konum, kişi,
  çıkartma ve tepki olaylarını sürücü üzerinden gönderir.
- Doğrudan Gateway sözleşmesi yoklamaları: `whatsapp-outbound-media-matrix`,
  `whatsapp-outbound-document-preserves-filename`, `whatsapp-outbound-poll`,
  `whatsapp-outbound-send-serialization`,
  `whatsapp-group-outbound-media`, `whatsapp-group-outbound-poll`,
  `whatsapp-message-actions`, `whatsapp-reply-context-isolation`,
  `whatsapp-reply-delivery-shape`. Bunlar model istemini bilinçli olarak atlar
  ve deterministik Gateway/kanal `send`, `poll` ve
  `message.action` sözleşmelerini kanıtlar.
- Erişim denetimi kapsamı: `whatsapp-access-control-dm-open`,
  `whatsapp-access-control-dm-disabled`, `whatsapp-access-control-group-open`,
  `whatsapp-access-control-group-disabled`, `whatsapp-group-allowlist-block`.
- Yerel onaylar: `whatsapp-approval-exec-deny-native`,
  `whatsapp-approval-exec-native`, `whatsapp-approval-exec-reaction-native`,
  `whatsapp-approval-exec-group-reaction-native`,
  `whatsapp-approval-plugin-native`.
- Durum tepkileri: `whatsapp-status-reactions`,
  `whatsapp-status-reaction-lifecycle`.

Katalog şu anda 52 senaryo içerir. `live-frontier` varsayılan hattı,
hızlı duman testi kapsamı için 8 senaryoyla küçük tutulur. `mock-openai`
varsayılan hattı, yalnızca model çıktısını taklit ederken gerçek WhatsApp
taşıması üzerinden 39 senaryoyu deterministik biçimde çalıştırır; onay senaryoları ve birkaç
daha ağır/engelleyici kontrol, senaryo kimliğiyle açıkça seçilmeye devam eder.

WhatsApp QA sürücüsü, yapılandırılmış canlı olayları (`text`, `media`,
`location`, `reaction` ve `poll`) gözlemler ve etkin olarak medya, anket,
kişi, konum ve çıkartma gönderebilir. QA Lab, özel
WhatsApp çalışma zamanı dosyalarına erişmek yerine bu sürücüyü
`@openclaw/whatsapp/api.js` paket yüzeyi üzerinden içe aktarır. Grup gözlemlerinde `fromJid` grup JID'siyken
`participantJid` ve `fromPhoneE164` katılımcı göndereni tanımlar.
Mesaj içeriği varsayılan olarak maskelenir. Doğrudan Gateway anket, dosya yükleme,
medya, grup anketi, grup medyası ve yanıt biçimi yoklamaları taşıma/API
sözleşmesi kontrolleridir; bunlar, bir kullanıcı isteminin agent'ın
aynı eylemi seçmesini sağladığının kanıtı olarak değerlendirilmez. Kullanıcı yolu eylem kanıtı,
`whatsapp-agent-message-action-react` ve
`whatsapp-group-agent-message-action-react` gibi senaryolardan gelir; bu senaryolarda sürücü normal bir
WhatsApp mesajı gönderir ve QA Lab sonuçta oluşan yerel WhatsApp yapıtını gözlemler.
WhatsApp senaryo ayrıntıları, kanıtın gerçekte kanıtladığından
daha güçlü bir sözleşme sanılmaması için her senaryonun yaklaşımını (`user-path`,
`direct-gateway` veya `native-approval`) içerir.

Çıktı yapıtları:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - canlı taşıma kontrollerinin kanıt girdileri.

### Convex kimlik bilgisi havuzu

Discord, Slack, Telegram ve WhatsApp hatları, yukarıdaki ortam değişkenlerini okumak yerine
paylaşılan bir Convex havuzundan kimlik bilgileri kiralayabilir.
`--credential-source convex` iletin (veya `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` ayarlayın);
QA Lab özel bir kiralama edinir, çalıştırma süresince Heartbeat gönderir
ve kapanışta kiralamayı serbest bırakır. Havuz türleri `"discord"`, `"slack"`,
`"telegram"` ve `"whatsapp"` şeklindedir.

Aracının `admin/add` sırasında doğruladığı yük biçimleri:

- Discord (`kind: "discord"`): `{ guildId: string, channelId: string,
driverBotToken: string, sutBotToken: string, sutApplicationId: string }`.
- Telegram (`kind: "telegram"`): `{ groupId: string, driverToken: string,
sutToken: string }` - `groupId` sayısal bir sohbet kimliği dizesi olmalıdır.
- Gerçek Telegram kullanıcısı (`kind: "telegram-user"`): `{ groupId: string, sutToken:
string, testerUserId: string, testerUsername: string, telegramApiId:
string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string,
tdlibArchiveBase64: string, tdlibArchiveSha256: string,
desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }` -
  yalnızca Mantis Telegram Desktop kanıtı içindir. Genel QA Lab hatları
  bu türü edinmemelidir.
- WhatsApp (`kind: "whatsapp"`): `{ driverPhoneE164: string, sutPhoneE164:
string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string,
groupJid?: string }` - telefon numaraları farklı E.164 dizeleri olmalıdır.

Mantis Telegram Desktop kanıt iş akışı, hem TDLib CLI sürücüsü hem de Telegram Desktop
tanığı için tek bir özel Convex `telegram-user` kiralamasını elinde tutar ve
kanıtı yayımladıktan sonra serbest bırakır.

Bir PR deterministik bir görsel fark gerektirdiğinde Mantis, Telegram biçimlendiricisi veya
teslimat katmanı değişirken `main` üzerinde ve PR başında aynı taklit
model yanıtını kullanabilir. Yakalama varsayılanları PR yorumları için ayarlanmıştır: standart
Crabbox sınıfı, 24fps masaüstü kaydı, 24fps hareketli GIF ve 1920px önizleme
genişliği. Önce/sonra yorumları, yalnızca amaçlanan GIF'leri içeren
temiz bir paket yayımlamalıdır.

Slack hatları da havuzu kullanabilir. Slack yük biçimi kontrolleri şu anda
aracı yerine Slack QA çalıştırıcısında bulunur; `{ channelId: string,
driverBotToken: string, sutBotToken: string, sutAppToken: string }` ve
`Cxxxxxxxxxx` gibi bir Slack kanal kimliği kullanın. Uygulama
ve kapsam sağlama için [Slack çalışma alanını ayarlama](#setting-up-the-slack-workspace) bölümüne bakın.

Operasyonel ortam değişkenleri ve Convex aracı uç noktası sözleşmesi,
[Test → Convex üzerinden paylaşılan Telegram kimlik bilgileri](/tr/help/testing#shared-telegram-credentials-via-convex-v1)
bölümünde bulunur (bölüm adı çok kanallı havuzdan önce belirlenmiştir; kiralama semantiği
türler arasında ortaktır).

## Depo destekli başlangıç verileri

Başlangıç varlıkları `qa/` içinde bulunur:

- `qa/scenarios/index.yaml`
- `qa/scenarios/<theme>/*.yaml`

QA planının hem insanlar hem de agent tarafından görülebilmesi için bunlar bilinçli olarak git içinde tutulur.

`qa-lab`, genel amaçlı bir YAML senaryo çalıştırıcısı olarak kalır. Her senaryo YAML dosyası,
tek bir test çalıştırmasının doğruluk kaynağıdır ve şunları tanımlamalıdır:

- üst düzey `title`
- `scenario` meta verileri
- `scenario` içinde isteğe bağlı kategori, yetenek, hat ve risk meta verileri
- `scenario` içinde belge ve kod referansları
- `scenario` içinde isteğe bağlı plugin gereksinimleri
- `scenario` içinde isteğe bağlı gateway yapılandırma yaması
- akış senaryoları için çalıştırılabilir üst düzey `flow` veya
  Vitest ve Playwright senaryoları için `scenario.execution.kind` / `scenario.execution.path`

`flow` öğesini destekleyen yeniden kullanılabilir çalışma zamanı yüzeyi genel ve
birden fazla alanı kapsayacak şekilde kalır. Örneğin YAML senaryoları, özel durumlu
bir çalıştırıcı eklemeden, gömülü Control UI'yi Gateway `browser.request` bağlantı
noktası üzerinden yöneten tarayıcı tarafı yardımcılarıyla taşıma tarafı
yardımcılarını birleştirebilir.

Senaryo dosyaları, kaynak ağaç klasörü yerine ürün yeteneğine göre
gruplandırılmalıdır. Dosyalar taşındığında senaryo kimliklerini sabit tutun;
uygulama izlenebilirliği için `docsRefs` ve
`codeRefs` kullanın.

Temel liste, aşağıdakileri kapsayacak kadar geniş kalmalıdır:

- DM ve kanal sohbeti
- ileti dizisi davranışı
- ileti eylemi yaşam döngüsü
- cron geri çağrıları
- bellekten geri çağırma
- model değiştirme
- alt ajan devri
- depo ve doküman okuma
- Lobster Invaders gibi küçük bir derleme görevi

## Sağlayıcı taklit kulvarları

`qa suite`, iki yerel sağlayıcı taklit kulvarına sahiptir:

- `mock-openai`, senaryodan haberdar OpenClaw taklididir. Depo destekli QA ve eşlik kapıları için varsayılan
  belirlenimci taklit kulvarı olarak kalır.
- `aimock`, deneysel protokol, sabit veri, kaydetme/yeniden oynatma ve kaos kapsamı için AIMock destekli bir sağlayıcı sunucusu başlatır. Ek niteliktedir ve
  `mock-openai` senaryo dağıtıcısının yerini almaz.

Sağlayıcı kulvarı uygulaması `extensions/qa-lab/src/providers/` altında bulunur.
Her sağlayıcı kendi varsayılanlarına, yerel sunucu başlatmasına, gateway model yapılandırmasına,
kimlik doğrulama profili hazırlama gereksinimlerine ve canlı/taklit yetenek bayraklarına sahip olur. Paylaşılan paket ve
gateway kodu, sağlayıcı adlarına göre dallanmak yerine sağlayıcı kayıt defteri üzerinden yönlendirilir.

## Taşıma bağdaştırıcıları

`qa-lab`, YAML QA senaryoları için genel bir taşıma bağlantı noktasına sahiptir. `qa-channel`,
yapay varsayılandır. `crabline`, yerel sağlayıcı biçimli sunucuları başlatır ve
OpenClaw'ın normal kanal pluginlerini bunlara karşı çalıştırır. `live`,
gerçek sağlayıcı kimlik bilgileri ve harici kanallar için ayrılmıştır.

Mimari düzeyde ayrım şöyledir:

- `qa-lab`; genel senaryo yürütme, çalışan eşzamanlılığı, çıktı
  yazma ve raporlamaya sahip olur.
- Taşıma bağdaştırıcısı; gateway yapılandırması, hazır olma durumu, gelen ve giden
  gözlem, taşıma eylemleri ve normalleştirilmiş taşıma durumuna sahip olur.
- `qa/scenarios/` altındaki YAML senaryo dosyaları test çalışmasını tanımlar; `qa-lab`
  ise bunları yürüten yeniden kullanılabilir çalışma zamanı yüzeyini sağlar.

### Kanal ekleme

YAML QA sistemine kanal eklemek, kanal uygulamasının yanı sıra
kanal sözleşmesini çalıştıran bir senaryo paketi gerektirir. Smoke CI
kapsamı için eşleşen Crabline yerel sağlayıcı sunucusunu ekleyin ve bunu
`crabline` sürücüsü üzerinden kullanıma açın.

Paylaşılan `qa-lab` ana bilgisayarı akışa sahip olabiliyorsa yeni bir üst düzey QA komut kökü
eklemeyin.

`qa-lab`, paylaşılan ana bilgisayar mekaniklerine sahip olur:

- `openclaw qa` komut kökü
- paket başlatma ve kapatma
- çalışan eşzamanlılığı
- çıktı yazma
- rapor oluşturma
- senaryo yürütme
- eski `qa-channel` senaryoları için uyumluluk takma adları

Çalıştırıcı pluginleri taşıma sözleşmesine sahip olur:

- `openclaw qa <runner>` öğesinin paylaşılan `qa` kökü altına nasıl bağlandığı
- gateway'in bu taşıma için nasıl yapılandırıldığı
- hazır olma durumunun nasıl denetlendiği
- gelen olayların nasıl eklendiği
- giden iletilerin nasıl gözlemlendiği
- transkriptlerin ve normalleştirilmiş taşıma durumunun nasıl kullanıma sunulduğu
- taşıma destekli eylemlerin nasıl yürütüldüğü
- taşımaya özgü sıfırlama veya temizliğin nasıl işlendiği

Yeni bir kanal için asgari benimseme eşiği:

1. Paylaşılan `qa` kökünün sahibi olarak `qa-lab` öğesini koruyun.
2. Taşıma çalıştırıcısını paylaşılan `qa-lab` ana bilgisayar bağlantı noktasında uygulayın.
3. Taşımaya özgü mekanikleri çalıştırıcı plugini veya kanal
   test düzeneği içinde tutun.
4. Çalıştırıcıyı rakip bir kök komut kaydetmek yerine
   `openclaw qa <runner>` olarak bağlayın. Çalıştırıcı pluginleri
   `openclaw.plugin.json` içinde `qaRunners` bildirmeli ve
   `runtime-api.ts` üzerinden eşleşen bir `qaRunnerCliRegistrations`
   dizisi dışa aktarmalıdır. `runtime-api.ts` öğesini hafif tutun; gecikmeli CLI ve
   çalıştırıcı yürütmesi ayrı giriş noktalarının arkasında kalmalıdır. İsteğe bağlı
   `adapterFactory`, komutun mevcut senaryo kataloğunu değiştirmeden taşımayı paylaşılan senaryolara açar.
   Fabrika her örneğin yalıtılmış kimlik bilgilerine veya
   atılabilir sunuculara, Gateway durumuna ve çıktı yollarına sahip olduğunu bildirmediği sürece aynı kanal bölümleri sıralı çalışır.
5. Temalı `qa/scenarios/` dizinleri altında YAML senaryoları yazın veya uyarlayın.
6. Yeni senaryolar için genel senaryo yardımcılarını kullanın.
7. Depoda kasıtlı bir geçiş yapılmıyorsa mevcut uyumluluk takma adlarını çalışır durumda tutun.

Karar kuralı kesindir:

- Davranış `qa-lab` içinde bir kez ifade edilebiliyorsa bunu `qa-lab` içine yerleştirin.
- Davranış tek bir kanal taşımasına bağlıysa bunu ilgili çalıştırıcı
  plugini veya plugin test düzeneğinde tutun.
- Bir senaryo, birden fazla kanalın kullanabileceği yeni bir yeteneğe ihtiyaç duyuyorsa
  `suite.ts` içinde kanala özgü bir dal yerine genel bir yardımcı ekleyin.
- Bir davranış yalnızca tek bir taşıma için anlamlıysa senaryoyu
  taşımaya özgü tutun ve bunu senaryo sözleşmesinde açıkça belirtin.

### Senaryo yardımcı adları

Yeni senaryolar için tercih edilen genel yardımcılar:

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

Uyumluluk takma adları mevcut senaryolar için kullanılabilir kalır:
`waitForQaChannelReady`, `waitForOutboundMessage`, `waitForNoOutbound`,
`formatConversationTranscript`, `resetBus`; ancak yeni senaryo yazımında
genel adlar kullanılmalıdır. Takma adlar, ileriye dönük model olarak değil,
bir kerede geçiş yapılmasını önlemek için vardır.

## Raporlama

`qa-lab`, gözlemlenen veri yolu zaman çizelgesinden bir Markdown protokol raporu dışa aktarır.
Rapor aşağıdakileri yanıtlamalıdır:

- Neler çalıştı
- Neler başarısız oldu
- Neler engellenmiş durumda kaldı
- Hangi takip senaryoları eklenmeye değer

Takip çalışmasının boyutunu belirlerken veya yeni bir taşıma bağlarken yararlı olan
kullanılabilir senaryoların envanteri için `pnpm openclaw qa coverage` komutunu çalıştırın
(makine tarafından okunabilir çıktı için `--json` ekleyin).
Değiştirilen bir davranış veya dosya yolu için odaklanmış kanıt seçerken
`pnpm openclaw qa coverage --match <query>` komutunu çalıştırın. Eşleşme raporu
senaryo meta verilerinde, doküman referanslarında, kod referanslarında, kapsam kimliklerinde,
pluginlerde ve sağlayıcı gereksinimlerinde arama yapar, ardından eşleşen
`qa suite
--scenario ...` hedeflerini yazdırır.

Her `qa suite` çalışması, seçilen senaryo kümesi için üst düzey
`qa-evidence.json`, `qa-suite-summary.json` ve `qa-suite-report.md`
çıktılarını yazar. `execution.kind: vitest` veya `execution.kind: playwright`
bildiren senaryolar eşleşen test yolunu çalıştırır ve ayrıca senaryo başına
günlükler yazar. `execution.kind: script` bildiren senaryolar,
`execution.path` konumundaki kanıt üreticisini `node --import tsx` üzerinden çalıştırır
(`${outputDir}` ve `${scenarioId}`, `execution.args` içinde genişletilir);
üretici kendi `qa-evidence.json` öğesini yazar, bunun girdileri paket çıktısına
aktarılır ve çıktı yolları ilgili üreticinin `qa-evidence.json` öğesine göre çözümlenir.
`qa suite` öğesine `qa run
--qa-profile` üzerinden ulaşıldığında aynı
`qa-evidence.json`, seçilen taksonomi kategorilerinin profil puan kartı özetini de içerir.

Kapsam çıktısını bir kapı yerine keşif yardımcısı olarak değerlendirin;
seçilen senaryo yine de test edilen davranış için doğru sağlayıcı moduna, canlı taşımaya,
Multipass'e, Testbox'a veya sürüm kulvarına ihtiyaç duyar. Puan kartı bağlamı için
[Olgunluk puan kartı](/tr/maturity/scorecard) sayfasına bakın.

Karakter ve stil denetimleri için aynı senaryoyu birden fazla canlı
model referansıyla çalıştırın ve değerlendirilmiş bir Markdown raporu yazın:

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.6-luna,thinking=medium,fast \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-8,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.6-sol,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-8,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

Komut Docker'ı değil, yerel QA gateway alt süreçlerini çalıştırır. Karakter
değerlendirme senaryoları kişiliği `SOUL.md` üzerinden ayarlamalı, ardından sohbet,
çalışma alanı yardımı ve küçük dosya görevleri gibi sıradan kullanıcı etkileşimlerini çalıştırmalıdır. Aday
modele değerlendirildiği söylenmemelidir. Komut her tam transkripti korur,
temel çalışma istatistiklerini kaydeder ve ardından doğal olma, genel his ve mizah açısından çalışmaları
sıralamaları için, desteklendiği durumlarda hızlı modda `xhigh` akıl yürütmesiyle
değerlendirici modellere sorar. Sağlayıcıları karşılaştırırken `--blind-judge-models`
kullanın: değerlendirici istemi yine her transkripti ve çalışma durumunu alır, ancak
aday referanslar `candidate-01` gibi tarafsız etiketlerle değiştirilir;
rapor, ayrıştırmadan sonra sıralamaları gerçek referanslarla eşleştirir.

Aday çalışmalar varsayılan olarak `high` düşünme düzeyini kullanır;
GPT-5.6 Luna için `medium`, bunu destekleyen eski OpenAI değerlendirme referansları için
`xhigh` kullanılır. Belirli bir adayı satır içinde
`--model provider/model,thinking=<level>` ile geçersiz kılın; satır içi seçenekler
`fast`, `no-fast` ve `fast=<bool>` öğelerini de destekler.
`--thinking
<level>` hâlâ genel bir geri dönüş değeri ayarlar ve eski
`--model-thinking
<provider/model=level>` biçimi uyumluluk için korunur. OpenAI aday referansları, sağlayıcının
desteklediği yerlerde öncelikli işlemenin kullanılması için varsayılan olarak hızlı modu kullanır.
Yalnızca her aday model için hızlı modu zorla etkinleştirmek istediğinizde
`--fast` iletin. Aday ve değerlendirici süreleri karşılaştırmalı değerlendirme analizi için
rapora kaydedilir, ancak değerlendirici istemleri hız temelinde sıralama yapılmamasını açıkça belirtir.
Hem aday hem de değerlendirici model çalışmaları varsayılan olarak 16 eşzamanlılık kullanır.
Sağlayıcı sınırları veya yerel gateway yükü bir çalışmayı fazla gürültülü hâle getirdiğinde
`--concurrency` ya da `--judge-concurrency` değerini düşürün.

Hiçbir aday `--model` iletilmediğinde karakter değerlendirmesi varsayılan olarak
`openai/gpt-5.6-luna`, `openai/gpt-5.2`, `openai/gpt-5`,
`anthropic/claude-opus-4-8`, `anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` ve `google/gemini-3.1-pro-preview` değerlerini kullanır. Hiçbir
`--judge-model` iletilmediğinde değerlendiriciler varsayılan olarak
`openai/gpt-5.6-sol,thinking=xhigh,fast` ve
`anthropic/claude-opus-4-8,thinking=high` değerlerini kullanır.

## İlgili dokümanlar

- [Olgunluk puan kartı](/tr/maturity/scorecard)
- [Kişisel ajan karşılaştırmalı değerlendirme paketi](/tr/concepts/personal-agent-benchmark-pack)
- [QA Kanalı](/tr/channels/qa-channel)
- [Test](/tr/help/testing)
- [Gösterge paneli](/tr/web/dashboard)
