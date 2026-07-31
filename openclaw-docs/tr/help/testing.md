---
read_when:
    - Testleri yerel olarak veya CI'da çalıştırma
    - Model/sağlayıcı hataları için regresyon testleri ekleme
    - Gateway + aracı davranışında hata ayıklama
summary: 'Test araç seti: birim/e2e/canlı test paketleri, Docker çalıştırıcıları ve her testin kapsadığı alanlar'
title: Test Etme
x-i18n:
    generated_at: "2026-07-26T23:43:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 20e0aa22bf16561334f83342abffabb387ed0b41b901773939123ecfbc0ae330
    source_path: help/testing.md
    workflow: 16
---

OpenClaw, Docker çalıştırıcılarına ek olarak üç Vitest paketine (birim/entegrasyon, e2e, canlı) sahiptir. Bu sayfa her paketin neleri kapsadığını, belirli bir iş akışı için hangi komutun çalıştırılacağını, canlı testlerin kimlik bilgilerini nasıl bulduğunu ve gerçek dünyadaki sağlayıcı/model hataları için regresyon testlerinin nasıl ekleneceğini açıklar.

<Note>
**QA yığını (qa-lab, qa-channel, canlı aktarım kulvarları)** ayrı olarak belgelenmiştir:

- [QA genel bakışı](/tr/concepts/qa-e2e-automation) - mimari, komut yüzeyi, senaryo yazımı ve Matrix profilleri.
- [Olgunluk puan kartı](/tr/maturity/scorecard) - sürüm QA kanıtlarının kararlılık ve LTS kararlarını nasıl desteklediği.
- [QA kanalı](/tr/channels/qa-channel) - depo destekli senaryolar tarafından kullanılan sentetik aktarım plugini.

Bu sayfa normal test paketlerini ve Docker/Parallels çalıştırıcılarını kapsar. Aşağıdaki [QA'ya özgü çalıştırıcılar](#qa-specific-runners), somut `qa` çağrılarını listeler ve yukarıdaki referanslara yönlendirir.
</Note>

## Hızlı başlangıç

Çoğu gün:

- Tam geçit (göndermeden önce beklenir): `pnpm build && pnpm check && pnpm check:test-types && pnpm test`
- Geniş kaynaklı bir makinede daha hızlı yerel tam paket çalıştırması: `pnpm test:max`
- Doğrudan Vitest izleme döngüsü: `pnpm test:watch`
- Doğrudan dosya hedefleme, plugin/kanal yollarını da yönlendirir: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Tek bir hata üzerinde yineleme yaparken önce hedefli çalıştırmaları tercih edin.
- Docker destekli QA sitesi: `pnpm qa:lab:up`
- Linux VM destekli QA kulvarı: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

Testlere dokunduğunuzda veya ek güvence istediğinizde:

- Bilgilendirici V8 kapsam raporu: `pnpm test:coverage`
- E2E paketi: `pnpm test:e2e`

## Test Geçici Dizinleri

Sahipliğin açık olması ve temizliğin test yaşam döngüsü içinde kalması için testlerin sahip olduğu geçici dizinlerde `test/helpers/temp-dir.ts` içindeki paylaşılan yardımcıları kullanın:

```ts
import { afterEach } from "vitest";
import { useAutoCleanupTempDirTracker } from "../helpers/temp-dir.js";

const tempDirs = useAutoCleanupTempDirTracker(afterEach);

it("geçici bir çalışma alanı kullanır", () => {
  const workspace = tempDirs.make("openclaw-example-");
  // çalışma alanını kullan
});
```

`useAutoCleanupTempDirTracker(afterEach)` kasıtlı olarak elle temizleme yöntemi sunmaz; her testten sonraki temizliğin sahibi Vitest'tir. Henüz taşınmamış testler için eski düşük seviyeli yardımcılar (`makeTempDir`, `cleanupTempDirs`, `createTempDirTracker`) hâlâ mevcuttur; bunların yeni kullanımlarından ve bir test ham geçici dizin davranışını açıkça doğrulamıyorsa yeni yalın `fs.mkdtemp*` çağrılarından kaçının. Yalın bir geçici dizin gerçekten gerektiğinde, nedeniyle birlikte denetlenebilir bir izin yorumu ekleyin:

```ts
// openclaw-temp-dir: allow ham fs temizleme davranışını doğrular
const workspace = fs.mkdtempSync(prefix);
```

`node scripts/report-test-temp-creations.mjs`, mevcut temizleme biçimlerini engellemeden eklenen diff satırlarındaki yeni yalın geçici dizin oluşturma işlemlerini ve paylaşılan yardımcının yeni elle kullanımlarını bildirir. `scripts/changed-lanes.mjs` ile aynı test yolu sınıflandırmasını izler ve paylaşılan yardımcı uygulamasının kendisini atlar. `check:changed`, bu raporu değiştirilen test yolları için yalnızca uyarı veren bir CI sinyali olarak çalıştırır (hatalar değil, GitHub uyarı ek açıklamaları).

## Canlı ve Docker/Parallels iş akışları

Gerçek sağlayıcılarda/modellerde hata ayıklarken (gerçek kimlik bilgileri gerektirir):

- Canlı paket (modeller + Gateway araç/görüntü yoklamaları): `pnpm test:live`
- Tek bir canlı dosyayı sessizce hedefleme: `pnpm test:live -- src/agents/models.profiles.live.test.ts`
- Çalışma zamanı performans raporları: gerçek bir `openai/gpt-5.6-luna` aracı dönüşü için `live_openai_candidate=true` veya Kova CPU/heap/izleme yapıtları için `deep_profile=true` ile `OpenClaw Performance` dağıtımını başlatın. Günlük zamanlanmış çalıştırmalar, sahte sağlayıcı, derin profil ve GPT-5.6 Luna kulvarı raporlarını ayrı bir yapıt tüketen yayımlayıcı işinden `openclaw/clawgrit-reports` hedefine yayımlar; eksik veya geçersiz yayımlayıcı kimlik doğrulaması, zamanlanmış ve `profile=release` çalıştırmalarını başarısız kılar. Sürüm dışı elle başlatılan dağıtımlar GitHub yapıtlarını korur ve rapor yayımlamayı tavsiye niteliğinde değerlendirir. Sahte sağlayıcı raporu ayrıca kaynak düzeyinde Gateway başlatma, bellek, plugin baskısı, tekrarlanan sahte model merhaba döngüsü ve CLI başlatma sayılarını içerir.
- Docker canlı model taraması: `pnpm test:docker:live-models`
  - Seçilen her model bir metin dönüşü ve küçük bir dosya okuma tarzı yoklama çalıştırır.
    Meta verileri `image` girdisini bildiren modeller ayrıca küçük bir görüntü dönüşü çalıştırır.
    Sağlayıcı hatalarını yalıtırken ek yoklamaları `OPENCLAW_LIVE_MODEL_FILE_PROBE=0` veya
    `OPENCLAW_LIVE_MODEL_IMAGE_PROBE=0` ile devre dışı bırakın.
  - CI kapsamı: günlük `OpenClaw Scheduled Live And E2E Checks` ve elle başlatılan
    `OpenClaw Release Checks`, sağlayıcıya göre parçalanmış Docker canlı model matrisi işlerini içeren
    `include_live_suites: true` ile yeniden kullanılabilir canlı/E2E iş akışını çağırır.
  - Odaklı CI yeniden çalıştırmaları için `OpenClaw Live And E2E Checks (Reusable)` dağıtımını
    `include_live_suites: true` ve `live_models_only: true` ile başlatın.
  - Yeni yüksek sinyalli sağlayıcı gizli bilgilerini `scripts/ci-hydrate-live-auth.sh`,
    `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml` ve bunun
    zamanlanmış/sürüm çağırıcılarına ekleyin.
- Yerel Codex bağlı sohbet duman testi: `pnpm test:docker:live-codex-bind`
  - Codex uygulama sunucusu yoluna karşı bir Docker canlı kulvarı çalıştırır, `/codex bind` ile sentetik bir Slack DM bağlar, `/codex fast` ve `/codex permissions` işlemlerini uygular; ardından düz bir yanıt ile görüntü ekinin ACP yerine yerel plugin bağlaması üzerinden yönlendirildiğini doğrular.
- Codex uygulama sunucusu test düzeneği duman testi: `pnpm test:docker:live-codex-harness`
  - Gateway aracı dönüşlerini pluginin sahip olduğu Codex uygulama sunucusu test düzeneğinden geçirir, `/codex status` ve `/codex models` değerlerini doğrular ve varsayılan olarak görüntü, cron MCP, alt aracı ve Guardian yoklamalarını uygular. Diğer hataları yalıtırken alt aracı yoklamasını `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=0` ile devre dışı bırakın. Odaklı bir alt aracı kontrolü için diğer yoklamaları devre dışı bırakın:
    `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 pnpm test:docker:live-codex-harness`.
    `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_ONLY=0` ayarlanmadığı sürece bu işlem alt aracı yoklamasından sonra çıkar.
- Codex isteğe bağlı kurulum duman testi: `pnpm test:docker:codex-on-demand`
  - Paketlenmiş OpenClaw tarball dosyasını Docker'a kurar, OpenAI API anahtarıyla ilk kurulumu çalıştırır ve Codex plugini ile `@openai/codex` bağımlılığının isteğe bağlı olarak yönetilen npm proje köküne indirildiğini doğrular.
- Codex npm plugini canlı paket duman testi: `pnpm test:docker:live-codex-npm-plugin`
  - Aday OpenClaw paketini ve tam Codex pluginini Docker'a kurar, ardından CLI ön kontrolü ve aynı oturumdaki dönüşler için gerçek bir OpenAI anahtarı kullanır.
  - Sıfır yeniden denemeli orta düzey düşünme içeren devam dönüşü ilerleme göndermeli, rastgele çalışma alanı okumaları ve tam bir yapıt yazımı boyunca çalışmayı sürdürmeli, ardından tamamlanma bildirmelidir. Yalnızca ilerleme bildiren son dönüş kulvarı başarısız kılar.
- Canlı plugin aracı bağımlılığı duman testi: `pnpm test:docker:live-plugin-tool`
  - Gerçek bir `slugify` bağımlılığı içeren sabit bir plugini paketler, bunu `npm-pack:` üzerinden kurar, yönetilen npm proje kökü altındaki bağımlılığı doğrular ve ardından canlı bir OpenAI modelinden plugin aracını çağırıp gizli kısa adı döndürmesini ister.
- OpenClaw kurtarma komutu duman testi: `pnpm test:live:system-agent-rescue-channel`
  - Mesaj kanalı kurtarma komutu yüzeyi için isteğe bağlı, çok katmanlı güvenlik kontrolü. `/openclaw status` işlemini uygular, kalıcı bir model değişikliğini kuyruğa alır, `/openclaw yes` yanıtını verir ve denetim/yapılandırma yazma yolunu doğrular.
- OpenClaw ilk çalıştırma Docker duman testi: `pnpm test:docker:system-agent-first-run`
  - Boş bir OpenClaw durum dizininden başlar ve önce paketlenmiş `openclaw setup` CLI'nin çıkarım olmadan kapalı biçimde başarısız olduğunu kanıtlar. Ardından paketlenmiş etkinleştirme modülü aracılığıyla sahte Claude'u test eder ve etkinleştirir. Ancak bundan sonra belirsiz bir paketlenmiş CLI isteği planlayıcıya ulaşır ve tipli kuruluma çözümlenir; bunu tek seferlik model, aracı, Discord yapılandırması ve SecretRef işlemleri izler. Yapılandırmayı ve denetim girdilerini doğrular. Bu, destekleyici geçit/işlem kanıtıdır; etkileşimli ilk kurulum veya OpenClaw aracı/araç/onay kanıtı değildir. Aynı kulvar QA Lab'da `pnpm openclaw qa suite --scenario system-agent-ring-zero-setup` ile sunulur.
- Moonshot/Kimi maliyet duman testi: `MOONSHOT_API_KEY` ayarlıyken `openclaw models list --provider moonshot --json` komutunu çalıştırın, ardından `moonshot/kimi-k2.6` hedefine karşı yalıtılmış bir `openclaw agent --local --session-id live-kimi-cost --message 'Reply exactly: KIMI_LIVE_OK' --thinking off --json` çalıştırın. JSON'un Moonshot/K2.6 bildirdiğini ve asistan dökümünün normalleştirilmiş `usage.cost` değerini sakladığını doğrulayın.

<Tip>
Yalnızca tek bir başarısız duruma ihtiyacınız olduğunda, aşağıda açıklanan izin listesi ortam değişkenleriyle canlı testleri daraltmayı tercih edin.
</Tip>

## QA'ya özgü çalıştırıcılar

QA-lab gerçekçiliğine ihtiyaç duyduğunuzda bu komutlar ana test paketlerinin yanında yer alır.

CI, QA Lab'ı özel iş akışlarında çalıştırır. Aracı eşliği, bağımsız bir PR iş akışında değil, `QA-Lab - All Lanes` ve sürüm doğrulaması altında iç içe bulunur. Geniş doğrulama, `rerun_group=qa-parity` ile `Full Release Validation` veya sürüm kontrollerinin QA grubunu kullanmalıdır. Kararlı/varsayılan sürüm kontrolleri, kapsamlı canlı/Docker dayanıklılık testini `run_release_soak=true` arkasında tutar; `full` profili dayanıklılık testini zorunlu kılar. `QA-Lab - All Lanes`, `main` üzerinde gecelik olarak ve elle başlatılan dağıtımdan; sahte eşlik kulvarı, canlı Matrix kulvarı, Convex tarafından yönetilen canlı Telegram kulvarı ve Convex tarafından yönetilen canlı Discord kulvarıyla paralel işler olarak çalışır. Zamanlanmış QA ve sürüm kontrolleri, Matrix sürüm profilini paylaşılan canlı bağdaştırıcı üzerinden çalıştırır. Matrix CLI ve elle çalıştırılan iş akışı girdisinin varsayılanı `all` olarak kalır; elle başlatılan `all` dağıtımları aktarım, medya ve E2EE profillerine dallanırken odaklı dağıtımlar `fast`, `release` veya `transport` seçebilir. `OpenClaw Release Checks`, sürüm onayından önce eşliği, yeniden kullanılabilir Matrix canlı bağdaştırıcı profilini ve Telegram kulvarını çalıştırır. Sürüm aktarım kontrolleri, deterministik kalmaları ve normal sağlayıcı plugini başlangıcından kaçınmaları için `mock-openai/gpt-5.6-luna` kullanır. Bu canlı aktarım Gateway'leri bellek aramasını devre dışı bırakır; bellek davranışı QA eşlik paketleri tarafından kapsanmaya devam eder.

Tam sürüm canlı medya parçaları, zaten `ffmpeg` ve `ffprobe` içeren `ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04` kullanır. Docker canlı model/arka uç parçaları, seçilen her commit için bir kez oluşturulan paylaşılan `ghcr.io/openclaw/openclaw-live-test:<sha>` görüntüsünü kullanır ve her parça içinde yeniden oluşturmak yerine bunu `OPENCLAW_SKIP_DOCKER_BUILD=1` ile çeker.

- `pnpm openclaw qa suite`
  - Depo destekli QA senaryolarını doğrudan ana makinede çalıştırır.
  - Seçilen senaryo kümesi için karma akış, Vitest ve Playwright senaryo
    seçimleri dahil olmak üzere üst düzey `qa-evidence.json`, `qa-suite-summary.json` ve
    `qa-suite-report.md` yapıtlarını yazar.
  - `pnpm openclaw qa run --qa-profile <profile>` tarafından tetiklendiğinde seçilen taksonomi profili
    puan kartını aynı `qa-evidence.json` içine gömer.
    `smoke-ci` sınırlı kanıt yazar (`evidenceMode: "slim"`, giriş başına
    `execution` yoktur). `release` özenle seçilmiş sürüme hazır olma
    bölümünü kapsar; `all` tüm etkin olgunluk kategorilerini seçer ve tam
    puan kartı yapıtı gerektiğinde açık QA Profile Evidence iş akışı
    tetiklemelerini hedefler.
  - Varsayılan olarak birden fazla seçili senaryoyu yalıtılmış gateway
    çalışanlarıyla paralel olarak çalıştırır. `qa-channel` varsayılan olarak 4
    eşzamanlılık kullanır (seçili senaryo sayısıyla sınırlıdır). Çalışan
    sayısını ayarlamak için `--concurrency <count>`, eski seri hat için ise
    `--concurrency 1` kullanın.
  - Herhangi bir senaryo başarısız olduğunda sıfır olmayan kodla çıkar.
    Başarısızlık çıkış kodu olmadan yapıtlar için `--allow-failures` kullanın.
  - `live-frontier`, `mock-openai` ve `aimock` sağlayıcı
    modlarını destekler. `aimock`, senaryo duyarlı `mock-openai` hattının
    yerini almadan deneysel fikstür ve protokol sahtesi kapsamı için AIMock
    destekli yerel bir sağlayıcı sunucusu başlatır.
- `pnpm openclaw qa coverage --match <query>`
  - Senaryo kimliklerini, başlıkları, yüzeyleri, kapsam kimliklerini, belge
    referanslarını, kod referanslarını, plugin'leri ve sağlayıcı gereksinimlerini
    arar, ardından eşleşen paket hedeflerini yazdırır.
  - Etkilenen davranışı veya dosya yolunu bildiğiniz ancak en küçük senaryoyu
    bilmediğiniz durumlarda bunu QA Lab çalıştırmasından önce kullanın. Yalnızca
    öneri amaçlıdır; yine de değiştirilen davranışa göre sahte, canlı,
    Multipass, Matrix veya taşıma kanıtını seçin.
- `pnpm test:plugins:kitchen-sink-live`
  - Canlı OpenAI Kitchen Sink plugin sınama serisini QA Lab üzerinden çalıştırır.
    Harici Kitchen Sink paketini kurar, plugin SDK yüzey envanterini doğrular,
    `/healthz` ve `/readyz` için yoklama yapar, gateway CPU/RSS
    kanıtını kaydeder, canlı bir OpenAI turu çalıştırır ve hasmane tanılamaları
    denetler. `OPENAI_API_KEY` gibi canlı OpenAI kimlik doğrulaması gerektirir.
    Hazırlanmış Testbox oturumlarında `openclaw-testbox-env` yardımcısı mevcutsa Testbox
    canlı kimlik doğrulama profilini otomatik olarak kaynak olarak kullanır.
- `pnpm test:gateway:cpu-scenarios`
  - Gateway başlangıç karşılaştırmasını ve küçük bir sahte QA Lab senaryo
    paketini (`channel-chat-baseline`, `memory-failure-fallback`,
    `gateway-restart-inflight-run`) çalıştırır ve `.artifacts/gateway-cpu-scenarios/` altında birleşik bir CPU
    gözlem özeti yazar.
  - Varsayılan olarak yalnızca sürekli yüksek CPU gözlemlerini işaretler
    (`--cpu-core-warn`, varsayılan `0.9`; `--hot-wall-warn-ms`,
    varsayılan `30000`); böylece kısa başlangıç sıçramaları, dakikalarca süren
    gateway yoğun kullanım regresyonu gibi görünmeden metrik olarak kaydedilir.
  - Derlenmiş `dist` yapıtları üzerinde çalışır; çalışma kopyasında
    güncel çalışma zamanı çıktısı yoksa önce bir derleme çalıştırın.
- `pnpm openclaw qa suite --runner multipass`
  - Aynı QA paketini tek kullanımlık bir Multipass Linux sanal makinesi içinde,
    `qa suite` ile aynı senaryo seçimi ve sağlayıcı/model bayraklarını
    koruyarak çalıştırır.
  - Canlı çalıştırmalar konuk için uygulanabilir QA kimlik doğrulama girdilerini
    iletir: ortam tabanlı sağlayıcı anahtarları, QA canlı sağlayıcı yapılandırma
    yolu ve mevcut olduğunda `CODEX_HOME`.
  - Konuğun bağlı çalışma alanı üzerinden geri yazabilmesi için çıktı dizinleri
    depo kökü altında kalmalıdır.
  - Normal QA raporu ve özetine ek olarak Multipass günlüklerini
    `.artifacts/qa-e2e/...` altına yazar.
- `pnpm qa:lab:up`
  - Operatör tarzı QA çalışmaları için Docker destekli QA sitesini başlatır.
- `pnpm test:docker:npm-onboard-channel-agent`
  - Geçerli çalışma kopyasından bir npm tarball'ı oluşturur, bunu Docker'a
    genel olarak kurar, etkileşimsiz OpenAI API anahtarı ilk katılımını çalıştırır,
    varsayılan olarak Telegram'ı yapılandırır, paketlenmiş plugin çalışma zamanının
    başlangıç bağımlılığı onarımı olmadan yüklendiğini doğrular, doctor'ı çalıştırır
    ve sahte bir OpenAI uç noktasına karşı bir yerel ajan turu çalıştırır.
  - Aynı paketlenmiş kurulum hattını Discord ile çalıştırmak için
    `OPENCLAW_NPM_ONBOARD_CHANNEL=discord` kullanın.
- `pnpm test:docker:session-runtime-context`
  - Gömülü çalışma zamanı bağlamı transkriptleri için deterministik, derlenmiş
    uygulamalı bir Docker duman testi çalıştırır. Gizli OpenClaw çalışma zamanı
    bağlamının görünür kullanıcı turuna sızmak yerine görüntülenmeyen özel bir mesaj
    olarak kalıcı olduğunu doğrular, ardından etkilenmiş bozuk bir oturum JSONL'si
    hazırlar ve `openclaw doctor --fix` öğesinin bunu bir yedekle etkin dala yeniden
    yazdığını doğrular.
- `pnpm test:docker:npm-telegram-live`
  - Bir OpenClaw paket adayını Docker'a kurar, kurulu paket ilk katılımını
    çalıştırır, kurulu CLI üzerinden Telegram'ı yapılandırır, ardından SUT Gateway
    olarak bu kurulu paketi kullanarak canlı Telegram QA hattını yeniden kullanır.
  - Sarmalayıcı çalışma kopyasından yalnızca `qa-lab` test düzeneği
    kaynağını bağlar; `dist`, `openclaw/plugin-sdk` ve paketlenmiş plugin çalışma
    zamanı kurulu pakete aittir, böylece hat geçerli çalışma kopyasındaki plugin'leri
    test edilen pakete karıştırmaz.
  - Varsayılan değer `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@beta` şeklindedir; kayıt defterinden kurmak
    yerine çözümlenmiş yerel bir tarball'ı test etmek için
    `OPENCLAW_NPM_TELEGRAM_PACKAGE_TGZ=/path/to/openclaw-current.tgz` veya
    `OPENCLAW_CURRENT_PACKAGE_TGZ` ayarlayın.
  - Varsayılan olarak `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES=20` ile `qa-evidence.json` içinde yinelenen
    RTT zamanlaması üretir. Çalıştırmayı ayarlamak için
    `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`,
    `OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS` veya
    `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES` değerlerini geçersiz kılın.
    `OPENCLAW_NPM_TELEGRAM_RTT_CHECKS`, örneklenecek Telegram QA senaryosunu seçer;
    desteklenen RTT hedefi `channel-canary` şeklindedir.
  - `pnpm openclaw qa telegram` ile aynı Telegram ortam kimlik bilgilerini veya Convex
    kimlik bilgisi kaynağını kullanır. CI/sürüm otomasyonu için
    `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex` ile
    `OPENCLAW_QA_CONVEX_SITE_URL` ve bir rol sırrını ayarlayın.
    CI ortamında `OPENCLAW_QA_CONVEX_SITE_URL` ve bir Convex rol sırrı mevcutsa Docker
    sarmalayıcısı Convex'i otomatik olarak seçer.
  - Sarmalayıcı, Docker derleme/kurulum çalışmasından önce ana makinedeki
    Telegram veya Convex kimlik bilgisi ortamını doğrular.
    `OPENCLAW_NPM_TELEGRAM_SKIP_CREDENTIAL_PREFLIGHT=1` değerini yalnızca kimlik bilgisi öncesi
    kurulumu bilinçli olarak hata ayıklarken ayarlayın.
  - `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci|maintainer`, yalnızca bu hat için paylaşılan `OPENCLAW_QA_CREDENTIAL_ROLE`
    değerini geçersiz kılar. Convex kimlik bilgileri seçildiğinde ve hiçbir rol
    ayarlanmadığında sarmalayıcı CI içinde `ci`, CI dışında ise
    `maintainer` kullanır.
  - GitHub Actions bu hattı `NPM Telegram Beta E2E` adlı manuel bakımcı iş akışı
    olarak sunar. Birleştirme sırasında çalışmaz. İş akışı
    `qa-live-shared` ortamını ve Convex CI kimlik bilgisi kiralamalarını kullanır.
- GitHub Actions ayrıca tek bir aday pakete karşı yan çalıştırma ürün kanıtı
  için `Package Acceptance` sunar. Bir Git referansını, yayımlanmış npm belirtimini,
  HTTPS tarball URL'si ve SHA-256 değerini, güvenilir URL politikasını veya başka
  bir çalıştırmadan alınan tarball yapıtını (`source=ref|npm|url|trusted-url|artifact`) kabul eder,
  normalleştirilmiş `openclaw-current.tgz` öğesini `package-under-test` olarak yükler,
  ardından mevcut Docker E2E zamanlayıcısını `smoke`, `package`,
  `product`, `full` veya `custom` hat profilleriyle
  çalıştırır. Telegram QA iş akışını aynı `package-under-test` yapıtına karşı
  çalıştırmak için `telegram_mode=mock-openai` veya `live-frontier` ayarlayın.
  - En son beta ürün kanıtı:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai
```

- Tam tarball URL'si kanıtı bir özet değeri gerektirir ve genel URL
güvenlik politikasını kullanır:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=url \
  -f package_url=https://registry.npmjs.org/openclaw/-/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

- Kurumsal/özel tarball yansıları açık bir güvenilir kaynak politikası kullanır:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

`source=trusted-url`, güvenilir iş akışı referansından `.github/package-trusted-sources.json` öğesini okur ve URL kimlik bilgilerini ya da iş akışı girdisiyle özel ağ atlamasını kabul etmez. Adlandırılmış politika taşıyıcı kimlik doğrulaması bildiriyorsa sabit `OPENCLAW_TRUSTED_PACKAGE_TOKEN` sırrını yapılandırın.

- Yapıt kanıtı, başka bir Actions çalıştırmasından bir tarball yapıtı indirir:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=<artifact-name> \
  -f suite_profile=smoke
```

- `pnpm test:docker:plugins`
  - Geçerli OpenClaw derlemesini Docker'da paketler ve kurar, OpenAI
    yapılandırılmış olarak Gateway'i başlatır, ardından yapılandırma düzenlemeleriyle
    paketlenmiş kanalları/plugin'leri etkinleştirir.
  - Kurulum keşfinin yapılandırılmamış indirilebilir plugin'leri mevcut
    bırakmadığını, ilk yapılandırılmış doctor onarımının eksik indirilebilir her
    plugin'i açıkça kurduğunu ve ikinci yeniden başlatmanın gizli bağımlılık
    onarımı çalıştırmadığını doğrular.
  - Ayrıca bilinen eski bir npm temel sürümünü kurar, `openclaw update --tag <candidate>`
    çalıştırılmadan önce Telegram'ı etkinleştirir ve adayın güncelleme sonrası
    doctor işleminin eski plugin bağımlılığı kalıntılarını test düzeneği taraflı
    bir postinstall onarımı olmadan temizlediğini doğrular.
- `pnpm test:parallels:npm-update`
  - Yerel paketlenmiş kurulum güncelleme duman testini Parallels konuklarında
    çalıştırır. Seçilen her platform önce istenen temel paketi kurar, ardından aynı
    konukta kurulu `openclaw update` komutunu çalıştırır ve kurulu sürümü,
    güncelleme durumunu, gateway hazır olma durumunu ve bir yerel ajan turunu doğrular.
  - Tek bir konuk üzerinde yineleme yaparken `--platform macos`,
    `--platform windows` veya `--platform linux` kullanın. Özet yapıt yolu ve hat
    başına durum için `--json` kullanın.
  - OpenAI hattı, canlı ajan turu kanıtı için varsayılan olarak
    `openai/gpt-5.6-luna` kullanır. Başka bir OpenAI modelini doğrulamak için
    `--model <provider/model>` iletin veya `OPENCLAW_PARALLELS_OPENAI_MODEL` ayarlayın.
  - Parallels taşıma takılmalarının test süresinin geri kalanını tüketememesi
    için uzun yerel çalıştırmaları ana makine zaman aşımıyla sarmalayın:

    ```bash
    timeout --foreground 150m pnpm test:parallels:npm-update -- --json
    timeout --foreground 90m pnpm test:parallels:npm-update -- --platform windows --json
    ```

  - Betik iç içe hat günlüklerini `/tmp/openclaw-parallels-npm-update.*` altına yazar. Dış
    sarmalayıcının takıldığını varsaymadan önce `windows-update.log`,
    `macos-update.log` veya `linux-update.log` öğesini inceleyin.
  - Windows güncellemesi, soğuk bir konukta güncelleme sonrası doctor ve paket
    güncelleme çalışmalarında 10 ila 15 dakika harcayabilir; iç içe npm hata
    ayıklama günlüğü ilerlediği sürece bu durum normaldir.
  - Bu toplu sarmalayıcıyı tekil Parallels macOS, Windows veya Linux duman
    testi hatlarıyla paralel çalıştırmayın. Bunlar sanal makine durumunu paylaşır
    ve anlık görüntü geri yükleme, paket sunma veya konuk gateway durumu üzerinde
    çakışabilir.
  - Güncelleme sonrası kanıt normal paketlenmiş plugin yüzeyini çalıştırır;
    çünkü konuşma, görüntü oluşturma ve medya anlama gibi yetenek cepheleri,
    ajan turunun kendisi yalnızca basit bir metin yanıtını denetlese bile paketlenmiş
    çalışma zamanı API'leri üzerinden yüklenir.

- `pnpm openclaw qa aimock`
  - Doğrudan protokol duman
    testi için yalnızca yerel AIMock sağlayıcı sunucusunu başlatır.
- `pnpm openclaw qa matrix`
  - Matrix canlı QA hattını, tek kullanımlık Docker destekli bir Tuwunel
    homeserver üzerinde çalıştırır. Yalnızca kaynak kod deposunda kullanılabilir; paketlenmiş kurulumlar
    `qa-lab` içermez.
  - Tam CLI, profil/senaryo kataloğu, ortam değişkenleri ve yapıt düzeni:
    [Matrix duman testi hatları](/tr/concepts/qa-e2e-automation#matrix-smoke-lanes).
- `pnpm openclaw qa telegram`
  - Telegram canlı QA hattını, ortamdaki sürücü ve test edilen sistem (SUT)
    bot token'larını kullanarak gerçek bir özel grupta çalıştırır.
  - `OPENCLAW_QA_TELEGRAM_GROUP_ID`,
    `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` ve
    `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN` gerektirir. Grup kimliği, sayısal
    Telegram sohbet kimliği olmalıdır.
  - Paylaşılan havuz kimlik bilgileri için `--credential-source convex` destekler.
    Varsayılan olarak ortam modunu kullanın veya havuzlanmış kiralamaları etkinleştirmek için
    `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` ayarlayın.
  - Varsayılanlar canary, bahsetme geçidi, komut adresleme, `/status`,
    bottan bota bahsetmeli yanıtlar ve temel yerel komut yanıtlarını kapsar.
    `mock-openai` varsayılanları ayrıca belirlenimci yanıt zinciri ve
    Telegram son mesaj akışı gerilemelerini kapsar. `session_status` gibi
    isteğe bağlı yoklamalar için `--list-scenarios` kullanın.
  - Herhangi bir senaryo başarısız olduğunda sıfır olmayan bir kodla çıkar. Başarısız
    çıkış kodu olmadan yapıtlar almak için `--allow-failures` kullanın.
  - Aynı özel grupta iki farklı bot bulunmasını ve SUT botunun
    bir Telegram kullanıcı adı sunmasını gerektirir.
  - Kararlı bottan bota gözlem için her iki botta
    `@BotFather` içindeki Bot-to-Bot Communication Mode seçeneğini etkinleştirin ve sürücü botun
    grup bot trafiğini gözlemleyebildiğinden emin olun.
  - `.artifacts/qa-e2e/...` altında bir Telegram QA raporu, özet ve
    `qa-evidence.json` yazar. Yanıt içeren senaryolar, sürücünün gönderme
    isteğinden gözlemlenen SUT yanıtına kadar olan RTT'yi içerir.

`Mantis Telegram Live`, bu hattın PR kanıtı sarmalayıcısıdır. Aday referansı
Convex tarafından kiralanan Telegram kimlik bilgileriyle çalıştırır, düzeltilmiş
QA raporu/kanıt paketini bir Crabbox masaüstü tarayıcısında işler, MP4
kanıtı kaydeder, hareket açısından kırpılmış bir GIF oluşturur, yapıt paketini yükler ve
`pr_number` ayarlandığında Mantis GitHub App aracılığıyla satır içi PR kanıtı
gönderir. Bakımcılar bunu Actions kullanıcı arayüzünden `Mantis Scenario`
(`scenario_id: telegram-live`) üzerinden veya doğrudan bir pull request yorumundan başlatabilir:

```text
@openclaw-mantis telegram
@openclaw-mantis telegram scenario=telegram-status-command
@openclaw-mantis telegram scenarios=telegram-status-command,channel-canary
```

`Mantis Telegram Desktop Proof`, PR görsel kanıtı için aracılı yerel Telegram Desktop
öncesi/sonrası sarmalayıcısıdır. Bunu Actions kullanıcı arayüzünden serbest biçimli
`instructions` ile, `Mantis Scenario` (`scenario_id:
telegram-desktop-proof`) üzerinden veya bir PR yorumundan başlatın:

```text
@openclaw-mantis telegram desktop proof
```

Mantis aracısı PR'ı okur, hangi Telegram'da görünür davranışın
değişikliği kanıtladığına karar verir, temel ve aday referanslarda gerçek kullanıcı
Crabbox Telegram Desktop kanıt hattını çalıştırır, yerel GIF'ler kullanışlı olana kadar
yineleme yapar, eşleştirilmiş bir `motionPreview` manifesti yazar ve `pr_number`
ayarlandığında Mantis GitHub App aracılığıyla aynı 2 sütunlu GIF tablosunu gönderir.

- `pnpm openclaw qa mantis telegram-desktop-builder`
  - Bir Crabbox Linux masaüstünü kiralar veya yeniden kullanır, yerel Telegram
    Desktop'ı kurar, OpenClaw'ı kiralanmış bir Telegram SUT bot token'ıyla
    yapılandırır, Gateway'i başlatır ve görünür VNC masaüstünden
    ekran görüntüsü/MP4 kanıtı kaydeder.
  - Varsayılan olarak `--credential-source convex` kullanır; böylece iş akışları yalnızca
    Convex aracı sırrına ihtiyaç duyar. `pnpm openclaw qa telegram` ile aynı
    `OPENCLAW_QA_TELEGRAM_*` değişkenlerini kullanarak `--credential-source env` seçeneğini kullanın.
  - Telegram Desktop yine de bir kullanıcı oturumu/profili gerektirir. Bot token'ı
    yalnızca OpenClaw'ı yapılandırır. Base64 biçimindeki bir `.tgz` profil arşivi
    için `--telegram-profile-archive-env <name>` kullanın veya `--keep-lease` kullanıp VNC üzerinden
    bir kez manuel olarak oturum açın.
  - Çıktı dizini altına `mantis-telegram-desktop-builder-report.md`,
    `mantis-telegram-desktop-builder-summary.json`,
    `telegram-desktop-builder.png` ve `telegram-desktop-builder.mp4`
    yazar.

Yeni taşıma mekanizmalarının birbirinden sapmaması için canlı taşıma hatları tek bir standart
sözleşmeyi paylaşır; hat başına kapsam matrisi
[QA genel bakışı - Canlı taşıma kapsamı](/tr/concepts/qa-e2e-automation#live-transport-coverage)
bölümündedir.
`qa-channel` geniş kapsamlı sentetik pakettir ve bu matrisin parçası değildir.

### Convex aracılığıyla paylaşılan Telegram kimlik bilgileri (v1)

Canlı taşıma QA'sı için `--credential-source convex` (veya `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`)
etkinleştirildiğinde QA laboratuvarı, Convex destekli bir havuzdan özel bir kiralama
alır, hat çalışırken bu kiralamaya Heartbeat gönderir ve kapanışta
kiralamayı serbest bırakır. Bölüm adı Discord, Slack ve WhatsApp desteğinden
önceye dayanır; kiralama sözleşmesi türler arasında paylaşılır.

Referans Convex proje iskelesi: `qa/convex-credential-broker/`

Gerekli ortam değişkenleri:

- `OPENCLAW_QA_CONVEX_SITE_URL` (örneğin `https://your-deployment.convex.site`)
- Seçilen rol için bir sır:
  - `maintainer` için `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`
  - `ci` için `OPENCLAW_QA_CONVEX_SECRET_CI`
- Kimlik bilgisi rolü seçimi:
  - CLI: `--credential-role maintainer|ci`
  - Ortam varsayılanı: `OPENCLAW_QA_CREDENTIAL_ROLE` (CI'da varsayılan `ci`, aksi durumda `maintainer`)

İsteğe bağlı ortam değişkenleri:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS` (varsayılan `1200000`)
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS` (varsayılan `30000`)
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS` (varsayılan `90000`)
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS` (varsayılan `15000`)
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX` (varsayılan `/qa-credentials/v1`)
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID` (isteğe bağlı izleme kimliği)
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1`, yalnızca yerel geliştirme için geri döngü `http://` Convex URL'lerine izin verir.

Normal çalışma sırasında `OPENCLAW_QA_CONVEX_SITE_URL`, `https://` kullanmalıdır.

Bakımcı yönetici komutları (havuz ekleme/kaldırma/listeleme) özellikle
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` gerektirir.

Bakımcılar için CLI yardımcıları:

```bash
pnpm openclaw qa credentials doctor
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

Canlı çalıştırmalardan önce gizli değerleri yazdırmadan Convex site URL'sini, aracı sırlarını,
uç nokta önekini, HTTP zaman aşımını ve yönetici/liste erişilebilirliğini denetlemek için
`doctor` kullanın. Betiklerde ve CI yardımcı programlarında makinece okunabilir
çıktı için `--json` kullanın.

Varsayılan uç nokta sözleşmesi (`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`).
İstekler bir `Authorization: Bearer <role secret>` üst bilgisiyle kimlik doğrular;
aşağıdaki gövdelerde bu üst bilgi gösterilmemiştir:

- `POST /acquire`
  - İstek: `{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - Başarılı: `{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - Tükenmiş/yeniden denenebilir: `{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /payload-chunk`
  - İstek: `{ kind, ownerId, actorRole, credentialId, leaseToken, index }`
  - Başarılı: `{ status: "ok", index, data }`
- `POST /heartbeat`
  - İstek: `{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - Başarılı: `{ status: "ok" }` (veya boş `2xx`)
- `POST /release`
  - İstek: `{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - Başarılı: `{ status: "ok" }` (veya boş `2xx`)
- `POST /admin/add` (yalnızca bakımcı sırrı)
  - İstek: `{ kind, actorId, payload, note?, status? }`
  - Başarılı: `{ status: "ok", credential }`
- `POST /admin/remove` (yalnızca bakımcı sırrı)
  - İstek: `{ credentialId, actorId }`
  - Başarılı: `{ status: "ok", changed, credential }`
  - Etkin kiralama koruması: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list` (yalnızca bakımcı sırrı)
  - İstek: `{ kind?, status?, includePayload?, limit? }`
  - Başarılı: `{ status: "ok", credentials, count }`

Telegram türü için yük biçimi:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId`, sayısal bir Telegram sohbet kimliği dizesi olmalıdır.
- `admin/add`, `kind: "telegram"` için bu biçimi doğrular ve hatalı biçimlendirilmiş yükleri reddeder.

Telegram gerçek kullanıcı türü için yük biçimi:

- `{ groupId: string, sutToken: string, testerUserId: string, testerUsername: string, telegramApiId: string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string, tdlibArchiveBase64: string, tdlibArchiveSha256: string, desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }`
- `groupId`, `testerUserId` ve `telegramApiId` sayısal dizeler olmalıdır.
- `tdlibArchiveSha256` ve `desktopTdataArchiveSha256`, SHA-256 onaltılık dizeleri olmalıdır.
- `kind: "telegram-user"`, Mantis Telegram Desktop kanıt iş akışı için ayrılmıştır. Genel QA Laboratuvarı hatları bunu kiralamamalıdır.

Aracı tarafından doğrulanan çok kanallı yükler:

- Discord: `{ guildId: string, channelId: string, driverBotToken: string, sutBotToken: string, sutApplicationId: string, voiceChannelId?: string }`
- WhatsApp: `{ driverPhoneE164: string, sutPhoneE164: string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string, groupJid?: string }`

Slack hatları da havuzdan kiralama yapabilir ancak Slack yük doğrulaması
şu anda aracı yerine Slack QA çalıştırıcısında bulunur. Slack satırları için
`{ channelId: string, driverBotToken: string, sutBotToken: string, sutAppToken: string }`
kullanın.

### QA'ya kanal ekleme

Yeni kanal bağdaştırıcılarının mimarisi ve senaryo yardımcısı adları
[QA genel bakışı - Kanal ekleme](/tr/concepts/qa-e2e-automation#adding-a-channel)
bölümündedir.
Asgari gereklilikler: taşıma çalıştırıcısını paylaşılan `qa-lab` ana bilgisayar
bağlantı noktasında uygulayın, paylaşılan senaryolar için bir `adapterFactory` ekleyin,
Plugin manifestinde `qaRunners` bildirin, `openclaw qa <runner>` olarak bağlayın ve
`qa/scenarios/` altında senaryolar yazın.

## Test paketleri (hangisi nerede çalışır)

Paketleri "artan gerçekçilik" (ve artan kararsızlık/maliyet) olarak düşünün.

### Birim / entegrasyon (varsayılan)

- Komut: `pnpm test`
- Yapılandırma: hedeflenmemiş çalıştırmalar `vitest.full-*.config.ts` parça kümesini kullanır ve
  paralel zamanlama için çok projeli parçaları proje başına yapılandırmalara
  genişletebilir
- Dosyalar: `src/**/*.test.ts`,
  `packages/**/*.test.ts` ve `test/**/*.test.ts` altındaki temel/birim envanterleri; kullanıcı arayüzü birim testleri,
  özel `unit-ui` parçasında çalışır
- Kapsam:
  - Saf birim testleri
  - İşlem içi entegrasyon testleri (Gateway kimlik doğrulaması, yönlendirme, araçlar, ayrıştırma, yapılandırma)
  - Bilinen hatalar için belirlenimci gerileme testleri
- Beklentiler:
  - CI'da çalışır
  - Gerçek anahtarlar gerekmez
  - Hızlı ve kararlı olmalıdır
  - Çözümleyici ve genel yüzey yükleyici testleri, gerçek paketlenmiş Plugin kaynak API'leriyle değil,
    oluşturulmuş küçük Plugin fikstürleriyle geniş kapsamlı `api.js` ve
    `runtime-api.js` geri dönüş davranışını kanıtlamalıdır. Gerçek Plugin API yüklemeleri
    Plugin'in sahip olduğu sözleşme/entegrasyon paketlerinde yer almalıdır.

Yerel bağımlılık politikası:

- Varsayılan test kurulumları isteğe bağlı yerel Discord opus derlemelerini atlar. Discord
  sesi paketlenmiş `libopus-wasm` kullanır ve yerel testler ile Testbox hatlarının yerel
  eklentiyi derlememesi için `@discordjs/opus`, `allowBuilds` içinde devre dışı kalır.
- Yerel opus performansını varsayılan OpenClaw kurulum/test döngülerinde değil,
  `libopus-wasm` kıyaslama deposunda karşılaştırın. Varsayılan `allowBuilds` içinde
  `@discordjs/opus` değerini `true` olarak ayarlamayın; bu, ilgisiz kurulum/test
  döngülerinin yerel kod derlemesine neden olur.

<AccordionGroup>
  <Accordion title="Projeler, parçalar ve kapsamlı hatlar">

    - Hedef belirtilmeyen `pnpm test`, tek bir dev yerel kök proje işlemi yerine on üç küçük parça yapılandırmasını (`core-unit-fast`, `core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-tooling`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) çalıştırır. Bu, yüklü makinelerde en yüksek RSS değerini düşürür ve otomatik yanıt/Plugin işlerinin ilgisiz paketleri kaynaklardan mahrum bırakmasını önler.
    - `pnpm test --watch`, çok parçalı bir izleme döngüsü pratik olmadığından hâlâ yerel kök `vitest.config.ts` proje grafiğini kullanır.
    - `pnpm test`, `pnpm test:watch` ve `pnpm test:perf:imports`, açık dosya/dizin hedeflerini önce kapsamlı hatlardan geçirir; böylece `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`, tam kök proje başlatma maliyetini üstlenmekten kaçınır.
    - `pnpm test:changed`, değişen git yollarını varsayılan olarak düşük maliyetli kapsamlı hatlara genişletir: doğrudan test düzenlemeleri, kardeş `*.test.ts` dosyaları, açık kaynak eşlemeleri ve yerel içe aktarma grafiği bağımlıları. `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` açıkça kullanılmadıkça yapılandırma/kurulum/paket düzenlemeleri testlerin geniş kapsamlı çalıştırılmasına neden olmaz.
    - `pnpm check:changed`, dar kapsamlı çalışmalar için olağan akıllı yerel denetim kapısıdır. Farkı çekirdek, çekirdek testleri, uzantılar, uzantı testleri, uygulamalar, belgeler, sürüm meta verileri, canlı Docker araçları ve araçlar olarak sınıflandırır; ardından eşleşen tür denetimi, lint ve koruma komutlarını çalıştırır. Vitest testlerini çalıştırmaz; test kanıtı için `pnpm test:changed` veya açık `pnpm test <target>` çağrısı yapın. Yalnızca sürüm meta verilerini değiştiren sürüm artışları, üst düzey sürüm alanı dışındaki paket değişikliklerini reddeden bir korumayla hedefli sürüm/yapılandırma/kök bağımlılık denetimlerini çalıştırır.
    - Canlı Docker ACP test düzeneği düzenlemeleri, canlı Docker kimlik doğrulama betikleri için kabuk söz dizimi denetimini ve canlı Docker zamanlayıcısının deneme çalıştırmasını içeren odaklı denetimleri çalıştırır. `package.json` değişiklikleri yalnızca fark `scripts["test:docker:live-*"]` ile sınırlıysa dahil edilir; bağımlılık, dışa aktarma, sürüm ve diğer paket yüzeyi düzenlemeleri yine daha geniş korumaları kullanır.
    - Aracılardan, komutlardan, Plugin'lerden, otomatik yanıt yardımcılarından, `plugin-sdk` ve benzer saf yardımcı alanlardan gelen içe aktarma yükü düşük birim testleri, `test/setup-openclaw-runtime.ts` öğesini atlayan `unit-fast` hattından geçirilir; durum bilgisi tutan/çalışma zamanı yükü ağır dosyalar mevcut hatlarda kalır.
    - Seçili `plugin-sdk` ve `commands` yardımcı kaynak dosyaları da değişiklik modu çalıştırmalarını bu hafif hatlardaki açık kardeş testlerle eşler; böylece yardımcı düzenlemeleri, söz konusu dizinin ağır paketinin tamamını yeniden çalıştırmaz.
    - `auto-reply`; üst düzey çekirdek yardımcıları, üst düzey `reply.*` entegrasyon testleri ve `src/auto-reply/reply/**` alt ağacı için ayrılmış gruplara sahiptir. CI ayrıca yanıt alt ağacını aracı çalıştırıcı, dağıtım ve komutlar/durum yönlendirme parçalarına böler; böylece içe aktarma yükü ağır tek bir grup, Node kuyruğunun tamamını üstlenmez.
    - Olağan PR/main CI, paketlenmiş Plugin toplu taramasını ve yalnızca sürüme yönelik `agentic-plugins` parçasını kasıtlı olarak atlar. Tam Sürüm Doğrulaması, sürüm adaylarındaki Plugin yükü ağır paketler için ayrı `Plugin Prerelease` alt iş akışını başlatır.

  </Accordion>

  <Accordion title="Gömülü çalıştırıcı kapsamı">

    - İleti aracı keşif girdilerini veya Compaction çalışma zamanı
      bağlamını değiştirdiğinizde her iki kapsam düzeyini de koruyun.
    - Saf yönlendirme ve normalleştirme sınırları için odaklı yardımcı
      regresyonları ekleyin.
    - Gömülü çalıştırıcı entegrasyon paketlerini sağlıklı tutun:
      `src/agents/embedded-agent-runner/compact.hooks.test.ts`,
      `src/agents/embedded-agent-runner/run.overflow-compaction.test.ts` ve
      `src/agents/embedded-agent-runner/run.overflow-compaction.loop.test.ts`.
    - Bu paketler, kapsamlı kimliklerin ve Compaction davranışının gerçek
      `run.ts` / `compact.ts` yollarından geçmeye devam ettiğini doğrular; yalnızca yardımcı testleri,
      bu entegrasyon yollarının yerine geçmek için yeterli değildir.

  </Accordion>

  <Accordion title="Vitest havuzu ve yalıtım varsayılanları">

    - Temel Vitest yapılandırmasının varsayılanı `threads` şeklindedir.
    - Paylaşılan Vitest yapılandırması `isolate: false` değerini sabitler ve
      kök projeler, e2e ve canlı yapılandırmalar genelinde yalıtılmamış çalıştırıcıyı kullanır.
    - Kök kullanıcı arayüzü hattı, `jsdom` kurulumunu ve iyileştiricisini korur ancak
      paylaşılan yalıtılmamış çalıştırıcıda da çalışır.
    - Her `pnpm test` parçası, paylaşılan Vitest yapılandırmasından aynı `threads` + `isolate: false`
      varsayılanlarını devralır.
    - `scripts/run-vitest.mjs`, büyük yerel çalıştırmalarda V8 derleme tekrarını azaltmak için
      varsayılan olarak Vitest alt Node işlemlerine `--no-maglev` ekler.
      Standart V8 davranışıyla karşılaştırmak için `OPENCLAW_VITEST_ENABLE_MAGLEV=1` ayarını kullanın.
    - `scripts/run-vitest.mjs`, stdout veya stderr çıktısı olmadan
      5 dakika geçen açık ve izleme dışı Vitest çalıştırmalarını sonlandırır. Kasıtlı olarak
      sessiz bir inceleme için gözetleyiciyi devre dışı bırakmak üzere
      `OPENCLAW_VITEST_NO_OUTPUT_TIMEOUT_MS=0` ayarını kullanın.

  </Accordion>

  <Accordion title="Hızlı yerel yineleme">

    - `pnpm changed:lanes`, bir farkın hangi mimari hatları tetiklediğini gösterir.
    - Ön işleme kancası yalnızca biçimlendirme yapar. Biçimlendirilmiş dosyaları
      yeniden hazırlar ve lint, tür denetimi veya testleri çalıştırmaz.
    - Akıllı yerel denetim kapısına ihtiyaç duyduğunuzda teslim veya gönderim öncesinde
      `pnpm check:changed` komutunu açıkça çalıştırın.
    - `pnpm test:changed`, varsayılan olarak düşük maliyetli kapsamlı hatlardan geçirilir. `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` öğesini yalnızca aracı,
      bir test düzeneği, yapılandırma, paket veya sözleşme düzenlemesinin gerçekten
      daha geniş Vitest kapsamına ihtiyaç duyduğuna karar verdiğinde kullanın.
    - `pnpm test:max` ve `pnpm test:changed:max`, yalnızca daha yüksek çalışan sınırıyla
      aynı yönlendirme davranışını korur.
    - Yerel çalışanların otomatik ölçeklendirilmesi kasıtlı olarak ihtiyatlıdır ve
      ana makinenin yük ortalaması zaten yüksek olduğunda geri çekilir; böylece eşzamanlı
      birden fazla Vitest çalıştırması varsayılan olarak daha az zarar verir.
    - Temel Vitest yapılandırması, test bağlantıları değiştiğinde değişiklik modu yeniden çalıştırmalarının
      doğru kalması için projeleri/yapılandırma dosyalarını
      `forceRerunTriggers` olarak işaretler.
    - Yapılandırma, desteklenen ana makinelerde `OPENCLAW_VITEST_FS_MODULE_CACHE` özelliğini etkin tutar;
      doğrudan profil çıkarma için tek bir açık önbellek konumu belirlemek üzere
      `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` ayarını kullanın.

  </Accordion>

  <Accordion title="Performans hata ayıklaması">

    - `pnpm test:perf:imports`, Vitest içe aktarma süresi raporlamasını ve
      içe aktarma dökümü çıktısını etkinleştirir.
    - `pnpm test:perf:imports:changed`, aynı profil çıkarma görünümünü
      `origin/main` tarihinden beri değişen dosyalarla sınırlar.
    - Parça zamanlama verileri `.artifacts/vitest-shard-timings.json` konumuna yazılır.
      Tam yapılandırma çalıştırmaları anahtar olarak yapılandırma yolunu kullanır; dahil etme desenli CI
      parçaları, filtrelenmiş parçaların ayrı ayrı izlenebilmesi için parça adını ekler.
    - Yoğun bir test hâlâ zamanının çoğunu başlangıç içe aktarmalarında harcıyorsa
      ağır bağımlılıkları dar bir yerel `*.runtime.ts` bağlantısının arkasında tutun ve
      çalışma zamanı yardımcılarını yalnızca `vi.mock(...)` üzerinden geçirmek için
      derinden içe aktarmak yerine bu bağlantının doğrudan sahtesini oluşturun.
    - `pnpm test:perf:changed:bench -- --ref <git-ref>`, işlenen fark için yönlendirilmiş
      `test:changed` ile yerel kök proje yolunu karşılaştırır ve geçen sürenin
      yanı sıra macOS en yüksek RSS değerini yazdırır.
    - `pnpm test:perf:changed:bench -- --worktree`, değişen dosya listesini
      `scripts/test-projects.mjs` ve kök Vitest yapılandırması üzerinden yönlendirerek
      mevcut kirli ağacın performansını ölçer.
    - `pnpm test:perf:profile:main`, Vitest/Vite başlangıcı ve dönüştürme ek yükü için
      ana iş parçacığı CPU profili yazar.
    - `pnpm test:perf:profile:runner`, dosya paralelliği devre dışıyken
      birim paketi için çalıştırıcı CPU+yığın profilleri yazar.

  </Accordion>
</AccordionGroup>

### Kararlılık (Gateway)

- Komut: `pnpm test:stability:gateway`
- Yapılandırma: Her biri tek çalışana zorlanan `test/vitest/vitest.gateway.config.ts`, `test/vitest/vitest.logging.config.ts` ve `test/vitest/vitest.infra.config.ts`
- Kapsam:
  - Tanılamalar varsayılan olarak etkin biçimde gerçek bir geri döngü Gateway'i başlatır
  - Sentetik Gateway iletisi, bellek ve büyük veri yükü hareketini tanılama olay yolu üzerinden yürütür
  - Gateway WS RPC üzerinden `diagnostics.stability` sorgusu yapar
  - Tanılama kararlılığı paketi kalıcılık yardımcılarını kapsar
  - Kaydedicinin sınırlı kaldığını, sentetik RSS örneklerinin baskı bütçesinin altında kaldığını ve oturum başına kuyruk derinliklerinin yeniden sıfıra indiğini doğrular
- Beklentiler:
  - CI için güvenlidir ve anahtar gerektirmez
  - Kararlılık regresyonu takibi için dar kapsamlı bir hattır; tam Gateway paketinin yerine geçmez

### E2E (depo toplamı)

- Komut: `pnpm test:e2e`
- Kapsam:
  - Gateway temel kontrolü E2E hattını çalıştırır
  - Sahte Control UI tarayıcı E2E hattını çalıştırır
- Beklentiler:
  - CI için güvenlidir ve anahtar gerektirmez
  - Playwright Chromium'un kurulu olmasını gerektirir

### E2E (Gateway temel kontrolü)

- Komut: `pnpm test:e2e:gateway`
- Yapılandırma: `test/vitest/vitest.e2e.config.ts`
- Dosyalar: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts` ve `extensions/` altındaki paketlenmiş Plugin E2E testleri
- Çalışma zamanı varsayılanları:
  - Deponun geri kalanıyla eşleşecek şekilde Vitest `threads` ile `isolate: false` kullanır.
  - Uyarlanabilir çalışanlar kullanır (CI: en fazla 2, yerel: varsayılan olarak 1).
  - Konsol G/Ç ek yükünü azaltmak için varsayılan olarak sessiz modda çalışır.
- Yararlı geçersiz kılmalar:
  - Çalışan sayısını zorlamak için `OPENCLAW_E2E_WORKERS=<n>` (üst sınır 16).
  - Ayrıntılı konsol çıktısını yeniden etkinleştirmek için `OPENCLAW_E2E_VERBOSE=1`.
- Kapsam:
  - Birden fazla örneğe sahip Gateway'in uçtan uca davranışı
  - WebSocket/HTTP yüzeyleri, Node eşleştirmesi ve daha ağır ağ iletişimi
- Beklentiler:
  - CI'da çalışır (işlem hattında etkinleştirildiğinde)
  - Gerçek anahtarlar gerekmez
  - Birim testlerinden daha fazla hareketli parça içerir (daha yavaş olabilir)

### E2E (Control UI sahte tarayıcısı)

- Komut: `pnpm test:ui:e2e`
- Yapılandırma: `test/vitest/vitest.ui-e2e.config.ts`
- Dosyalar: `ui/src/**/*.e2e.test.ts`
- Kapsam:
  - Vite Control UI'ı başlatır
  - Playwright üzerinden gerçek bir Chromium sayfasını çalıştırır
  - Gateway WebSocket'i tarayıcı içindeki deterministik sahtelerle değiştirir
- Beklentiler:
  - `pnpm test:e2e` parçası olarak CI'da çalışır
  - Gerçek Gateway, aracılar veya sağlayıcı anahtarları gerekmez
  - Tarayıcı bağımlılığı mevcut olmalıdır (`pnpm --dir ui exec playwright install chromium`)

### E2E: OpenShell arka uç temel kontrolü

- Komut: `pnpm test:e2e:openshell`
- Dosya: `extensions/openshell/src/backend.e2e.test.ts`
- Kapsam:
  - Etkin bir yerel OpenShell Gateway'ini yeniden kullanır
  - Geçici bir yerel Dockerfile'dan korumalı alan oluşturur
  - OpenClaw'ın OpenShell arka ucunu gerçek `sandbox ssh-config` + SSH exec üzerinden çalıştırır
  - Korumalı alan dosya sistemi köprüsü üzerinden uzak sistemin kurallarına uygun dosya sistemi davranışını doğrular
- Beklentiler:
  - Yalnızca isteğe bağlıdır; varsayılan `pnpm test:e2e` çalıştırmasının parçası değildir
  - Yerel bir `openshell` CLI ve çalışan bir Docker daemon gerektirir
  - Etkin bir yerel OpenShell Gateway'i ve onun yapılandırma kaynağını gerektirir
  - Yalıtılmış `HOME` / `XDG_CONFIG_HOME` kullanır, ardından test korumalı alanını yok eder
- Yararlı geçersiz kılmalar:
  - Daha geniş e2e paketi elle çalıştırılırken testi etkinleştirmek için `OPENCLAW_E2E_OPENSHELL=1`
  - Varsayılan olmayan bir CLI ikili dosyasına veya sarmalayıcı betiğe işaret etmek için `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`
  - Kayıtlı Gateway yapılandırmasını yalıtılmış teste sunmak için `OPENCLAW_E2E_OPENSHELL_CONFIG_HOME=/path/to/config`
  - Ana makine ilkesi fikstürünün kullandığı Docker Gateway IP'sini geçersiz kılmak için `OPENCLAW_E2E_OPENSHELL_HOST_IP=172.18.0.1`

### Canlı (gerçek sağlayıcılar + gerçek modeller)

- Komut: `pnpm test:live`
- Yapılandırma: `test/vitest/vitest.live.config.ts`
- Dosyalar: `src/**/*.live.test.ts`, `test/**/*.live.test.ts` ve `extensions/` altındaki paketlenmiş Plugin canlı testleri
- Varsayılan: `pnpm test:live` tarafından **etkin** ( `OPENCLAW_LIVE_TEST=1` ayarlanır)
- Kapsam:
  - "Bu sağlayıcı/model gerçek kimlik bilgileriyle _bugün_ gerçekten çalışıyor mu?"
  - Sağlayıcı biçimi değişikliklerini, araç çağırma kendine özgü davranışlarını, kimlik doğrulama sorunlarını ve hız sınırı davranışını yakalar
- Beklentiler:
  - Tasarım gereği CI açısından kararlı değildir (gerçek ağlar, gerçek sağlayıcı politikaları, kotalar, kesintiler)
  - Maliyetlidir / hız sınırlarını kullanır
  - "Her şeyi" çalıştırmak yerine daraltılmış alt kümeleri çalıştırmayı tercih edin
- Canlı çalıştırmalar, önceden dışa aktarılmış API anahtarlarını ve hazırlanmış kimlik doğrulama profillerini kullanır.
- Canlı çalıştırmalar varsayılan olarak yine `HOME` öğesini yalıtır ve birim testi fikstürlerinin gerçek `~/.openclaw` öğenizi değiştirememesi için yapılandırma/kimlik doğrulama materyalini geçici bir test ana dizinine kopyalar.
- Yalnızca canlı testlerin gerçek ana dizininizi kullanmasına kasıtlı olarak ihtiyaç duyduğunuzda `OPENCLAW_LIVE_USE_REAL_HOME=1` ayarlayın.
- `pnpm test:live` varsayılan olarak daha sessiz bir mod kullanır: `[live] ...` ilerleme çıktısını korur ve Gateway önyükleme günlüklerini/Bonjour bildirimlerini susturur. Tüm başlangıç günlüklerini geri istiyorsanız `OPENCLAW_LIVE_TEST_QUIET=0` ayarlayın.
- API anahtarı rotasyonu (sağlayıcıya özgü): virgül/noktalı virgül biçimiyle `*_API_KEYS` veya `*_API_KEY_1`, `*_API_KEY_2` (örneğin `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) ayarlayın ya da `OPENCLAW_LIVE_*_KEY` aracılığıyla canlı çalıştırma başına geçersiz kılma kullanın; testler hız sınırı yanıtlarında yeniden dener.
- İlerleme/Heartbeat çıktısı:
  - Canlı test paketleri stderr'e ilerleme satırları gönderir; böylece Vitest konsol yakalaması sessizken bile uzun sağlayıcı çağrılarının etkin olduğu görünür.
  - `test/vitest/vitest.live.config.ts`, Vitest konsol müdahalesini devre dışı bırakır; böylece sağlayıcı/Gateway ilerleme satırları canlı çalıştırmalar sırasında anında akar.
  - Doğrudan model Heartbeat'lerini `OPENCLAW_LIVE_HEARTBEAT_MS` ile ayarlayın.
  - Gateway/sonda Heartbeat'lerini `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` ile ayarlayın.

## Hangi test paketini çalıştırmalıyım?

Bu karar tablosunu kullanın:

- Mantık/testler düzenleniyorsa: `pnpm test` çalıştırın (çok fazla değişiklik yaptıysanız `pnpm test:coverage` de)
- Gateway ağı / WS protokolü / eşleştirme değiştiriliyorsa: `pnpm test:e2e` ekleyin
- "Botum çalışmıyor" durumu / sağlayıcıya özgü hatalar / araç çağırma hata ayıklaması yapılıyorsa: daraltılmış bir `pnpm test:live` çalıştırın

## Canlı (ağa erişen) testler

Canlı model matrisi, CLI arka uç duman testleri, ACP duman testleri, Codex uygulama sunucusu
test düzeneği ve tüm medya sağlayıcısı canlı testleri (Deepgram, BytePlus, ComfyUI,
görüntü, müzik, video, medya test düzeneği) ile canlı çalıştırmaların kimlik bilgisi yönetimi için

- [Canlı test paketlerini test etme](/tr/help/testing-live) bölümüne bakın. Özel güncelleme ve
  Plugin doğrulama kontrol listesi için
  [Güncellemeleri ve Pluginleri test etme](/tr/help/testing-updates-plugins) bölümüne bakın.

## Docker çalıştırıcıları (isteğe bağlı "Linux'ta çalışıyor" kontrolleri)

Bu Docker çalıştırıcıları iki gruba ayrılır:

- Canlı model çalıştırıcıları: `test:docker:live-models` ve `test:docker:live-gateway`, yerel yapılandırma dizininizi, çalışma alanınızı ve isteğe bağlı profil ortam dosyanızı bağlayarak yalnızca eşleşen profil anahtarı canlı dosyasını depo Docker görüntüsü (`src/agents/models.profiles.live.test.ts` ve `src/gateway/gateway-models.profiles.live.test.ts`) içinde çalıştırır. Eşleşen yerel giriş noktaları `test:live:models-profiles` ve `test:live:gateway-profiles` öğeleridir.
- Docker canlı çalıştırıcıları, gerektiğinde kendi pratik sınırlarını korur:
  `test:docker:live-models` varsayılan olarak özenle seçilmiş, desteklenen ve yüksek sinyalli kümeyi; 
  `test:docker:live-gateway` ise varsayılan olarak `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` ve
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000` öğelerini kullanır. Açıkça daha küçük bir sınır veya daha büyük bir tarama istediğinizde `OPENCLAW_LIVE_MAX_MODELS`
  ya da Gateway ortam değişkenlerini ayarlayın.
- `test:docker:all`, canlı Docker görüntüsünü `test:docker:live-build` aracılığıyla bir kez oluşturur, OpenClaw'ı `scripts/package-openclaw-for-docker.mjs` üzerinden bir npm tar arşivi olarak bir kez paketler ve ardından iki `scripts/e2e/Dockerfile` görüntüsünü oluşturur/yeniden kullanır. Yalın görüntü yalnızca kurulum/güncelleme/Plugin bağımlılığı hatları için Node/Git çalıştırıcısıdır; bu hatlar önceden oluşturulmuş tar arşivini bağlar. İşlevsel görüntü, derlenmiş uygulama işlevselliği hatları için aynı tar arşivini `/app` içine kurar. Docker hattı tanımları `scripts/lib/docker-e2e-scenarios.mjs` içinde; planlayıcı mantığı `scripts/lib/docker-e2e-plan.mjs` içinde bulunur; `scripts/test-docker-all.mjs` seçilen planı yürütür. Toplu çalıştırma ağırlıklı bir yerel zamanlayıcı kullanır: `OPENCLAW_DOCKER_ALL_PARALLELISM` işlem yuvalarını denetlerken kaynak sınırları ağır canlı, npm kurulumu ve çok hizmetli hatların tümünün aynı anda başlamasını önler. Tek bir hat etkin sınırlardan daha ağırsa zamanlayıcı, havuz boşken yine de onu başlatabilir ve kapasite yeniden kullanılabilir olana kadar tek başına çalışmasını sürdürür. Varsayılanlar 10 yuva, `OPENCLAW_DOCKER_ALL_LIVE_LIMIT=9`, `OPENCLAW_DOCKER_ALL_NPM_LIMIT=5` ve `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT=7` şeklindedir; `OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` veya `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT` (ve diğer `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT` geçersiz kılmalarını) yalnızca Docker ana makinesinde daha fazla kapasite olduğunda ayarlayın. Çalıştırıcı varsayılan olarak bir Docker ön kontrolü gerçekleştirir, eski OpenClaw E2E kapsayıcılarını kaldırır, her 30 saniyede bir durum bilgisi yazdırır, başarılı hat sürelerini `.artifacts/docker-tests/lane-timings.json` içinde saklar ve sonraki çalıştırmalarda daha uzun hatları önce başlatmak için bu süreleri kullanır. Docker'ı oluşturmadan veya çalıştırmadan ağırlıklı hat bildirimini yazdırmak için `OPENCLAW_DOCKER_ALL_DRY_RUN=1`; seçilen hatlar, paket/görüntü gereksinimleri ve kimlik bilgileri için CI planını yazdırmak üzere `node scripts/test-docker-all.mjs --plan-json` kullanın.
- `Package Acceptance`, "bu kurulabilir tar arşivi bir ürün olarak çalışıyor mu?" sorusuna yönelik GitHub'a özgü paket geçididir. `source=npm`, `source=ref`, `source=url`, `source=trusted-url` veya `source=artifact` kaynaklarından tek bir aday paketi çözümler, bunu `package-under-test` olarak yükler ve ardından seçilen referansı yeniden paketlemek yerine yeniden kullanılabilir Docker E2E hatlarını tam olarak bu tar arşivine karşı çalıştırır. Profiller kapsam genişliğine göre sıralanır: `smoke`, `package`, `product` ve `full` (ayrıca açık bir hat listesi için `custom`). Paket/güncelleme/Plugin sözleşmesi, yayımlanmış yükseltmeden sonra çalışmayı sürdürenler matrisi, sürüm varsayılanları ve hata triyajı için [Güncellemeleri ve Pluginleri test etme](/tr/help/testing-updates-plugins) bölümüne bakın.
- Derleme ve sürüm kontrolleri, tsdown sonrasında `scripts/check-cli-bootstrap-imports.mjs` çalıştırır. Koruma, `dist/entry.js` ve `dist/cli/run-main.js` kaynaklarından başlayarak statik derlenmiş grafiği dolaşır ve bu dağıtım öncesi önyükleme grafiği komut dağıtımından önce herhangi bir haricî paketi (Commander, istem kullanıcı arabirimi, undici, günlük kaydı ve benzeri başlangıç açısından ağır bağımlılıkların tümü buna dahildir) statik olarak içe aktarırsa başarısız olur; ayrıca paketlenmiş Gateway çalıştırma parçasını 70 KB ile sınırlar ve bu parçadan bilinen soğuk Gateway yollarının (`control-ui-assets`, `diagnostic-stability-bundle`, `onboard-helpers`, `process-respawn`, `restart-sentinel`, `server-close`, `server-reload-handlers`) statik olarak içe aktarılmasını reddeder. `scripts/release-check.ts`, paketlenmiş CLI'yi ayrıca `--help`, `onboard --help`, `doctor --help`, `status --json --timeout 1`, `config schema` ve `models list --provider openai` ile duman testinden geçirir.
- Paket Kabulü eski sürüm uyumluluğu `2026.4.25` ile sınırlıdır (`2026.4.25-beta.*` dahildir). Bu sınır dâhilinde test düzeneği yalnızca yayımlanmış paket meta verisi eksiklerini tolere eder: atlanmış özel QA envanter girdileri, eksik `gateway install --wrapper`, tar arşivinden türetilen git fikstüründeki eksik yama dosyaları, eksik kalıcı `update.channel`, eski Plugin kurulum kaydı konumları, eksik pazar yeri kurulum kaydı kalıcılığı ve `plugins update` sırasında yapılandırma meta verisi geçişi. `2026.4.25` sonrasındaki paketlerde bu yollar kesin hata olarak değerlendirilir.
- Kapsayıcı duman testi çalıştırıcıları: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:npm-onboard-channel-agent`, `test:docker:release-user-journey`, `test:docker:release-typed-onboarding`, `test:docker:release-media-memory`, `test:docker:release-upgrade-user-journey`, `test:docker:release-plugin-marketplace`, `test:docker:skill-install`, `test:docker:update-channel-switch`, `test:docker:upgrade-survivor`, `test:docker:published-upgrade-survivor`, `test:docker:session-runtime-context`, `test:docker:agents-delete-shared-workspace`, `test:docker:gateway-network`, `test:docker:browser-cdp-snapshot`, `test:docker:mcp-channels`, `test:docker:agent-bundle-mcp-tools`, `test:docker:cron-mcp-cleanup`, `test:docker:plugins`, `test:docker:plugin-update`, `test:docker:plugin-lifecycle-matrix` ve `test:docker:config-reload`, bir veya daha fazla gerçek kapsayıcıyı başlatır ve üst düzey entegrasyon yollarını doğrular.
- Paketlenmiş OpenClaw tar arşivini `scripts/lib/openclaw-e2e-instance.sh` aracılığıyla kuran Docker/Bash E2E hatları, `npm install` değerini `OPENCLAW_E2E_NPM_INSTALL_TIMEOUT` ile sınırlar (varsayılan `600s`; hata ayıklamak amacıyla sarmalayıcıyı devre dışı bırakmak için `0` ayarlayın).

Canlı model Docker çalıştırıcıları ayrıca yalnızca gereken CLI kimlik doğrulama ana dizinlerini
(çalıştırma daraltılmamışsa desteklenenlerin tümünü) bağlar ve ardından haricî CLI OAuth'un
ana makine kimlik doğrulama deposunu değiştirmeden tokenları yenileyebilmesi için bunları
çalıştırmadan önce kapsayıcı ana dizinine kopyalar:

- Doğrudan modeller: `pnpm test:docker:live-models` (betik: `scripts/test-live-models-docker.sh`)
- ACP bağlama duman testi: `pnpm test:docker:live-acp-bind` (betik: `scripts/test-live-acp-bind-docker.sh`; varsayılan olarak Claude, Codex ve Gemini'yi, `pnpm test:docker:live-acp-bind:droid` ve `pnpm test:docker:live-acp-bind:opencode` aracılığıyla da kesin Droid/OpenCode kapsamını içerir)
- CLI arka uç duman testi: `pnpm test:docker:live-cli-backend` (betik: `scripts/test-live-cli-backend-docker.sh`)
- Codex uygulama sunucusu test düzeneği duman testi: `pnpm test:docker:live-codex-harness` (betik: `scripts/test-live-codex-harness-docker.sh`)
- Gateway + geliştirme aracısı: `pnpm test:docker:live-gateway` (betik: `scripts/test-live-gateway-models-docker.sh`)
- Gözlemlenebilirlik duman testleri: `pnpm qa:otel:smoke`, `pnpm qa:prometheus:smoke` ve `pnpm qa:observability:smoke`, özel QA kaynak kodu teslim alma hatlarıdır. npm tar arşivi QA Lab'i içermediğinden bunlar kasıtlı olarak paket Docker sürüm hatlarının parçası değildir.
- Open WebUI canlı duman testi: `pnpm test:docker:openwebui` (betik: `scripts/e2e/openwebui-docker.sh`)
- İlk katılım sihirbazı (TTY, tam iskele oluşturma): `pnpm test:docker:onboard` (betik: `scripts/e2e/onboard-docker.sh`)
- Npm tar arşivi ilk katılım/kanal/aracı duman testi: `pnpm test:docker:npm-onboard-channel-agent`, paketlenmiş OpenClaw tar arşivini Docker'a genel olarak kurar, varsayılan olarak ortam referanslı ilk katılım aracılığıyla OpenAI'ı ve Telegram'ı yapılandırır, doctor'ı çalıştırır ve taklit edilmiş bir OpenAI aracı turu çalıştırır. Önceden oluşturulmuş bir tar arşivini `OPENCLAW_CURRENT_PACKAGE_TGZ=/path/to/openclaw-*.tgz` ile yeniden kullanın, ana makine yeniden derlemesini `OPENCLAW_NPM_ONBOARD_HOST_BUILD=0` ile atlayın veya kanalı `OPENCLAW_NPM_ONBOARD_CHANNEL=discord` ya da `OPENCLAW_NPM_ONBOARD_CHANNEL=slack` ile değiştirin.

- Sürüm kullanıcı yolculuğu duman testi: `pnpm test:docker:release-user-journey`, paketlenmiş OpenClaw tarball'ını temiz bir Docker ana dizinine global olarak kurar, ilk katılımı çalıştırır, sahte bir OpenAI sağlayıcısı yapılandırır, bir ajan turu çalıştırır, harici pluginleri kurar/kaldırır, ClickClack'i yerel bir fixture'a karşı yapılandırır, giden/gelen mesajlaşmayı doğrular, Gateway'i yeniden başlatır ve doctor'ı çalıştırır.
- Sürüm tür belirtilmiş ilk katılım duman testi: `pnpm test:docker:release-typed-onboarding`, paketlenmiş tarball'ı kurar, `openclaw onboard` sürecini gerçek bir TTY üzerinden yürütür, OpenAI'ı env-ref sağlayıcısı olarak yapılandırır, ham anahtarın kalıcı olarak saklanmadığını doğrular ve sahte bir ajan turu çalıştırır.
- Sürüm medya/bellek duman testi: `pnpm test:docker:release-media-memory`, paketlenmiş tarball'ı kurar; bir PNG ekinden görüntü anlamayı, OpenAI uyumlu görüntü oluşturma çıktısını, bellek aramasıyla hatırlamayı ve Gateway yeniden başlatıldıktan sonra hatırlamanın korunmasını doğrular.
- Sürüm yükseltme kullanıcı yolculuğu duman testi: `pnpm test:docker:release-upgrade-user-journey`, varsayılan olarak aday tarball'dan eski, yayımlanmış en yeni temel sürümü kurar; yayımlanmış pakette sağlayıcı/plugin/ClickClack durumunu yapılandırır, aday tarball'a yükseltir ve ardından temel ajan/plugin/kanal yolculuğunu yeniden çalıştırır. Yayımlanmış daha eski bir temel sürüm yoksa aday sürümü yeniden kullanır. Temel sürümü `OPENCLAW_RELEASE_UPGRADE_BASELINE_SPEC=openclaw@<version>` ile geçersiz kılın.
- Sürüm plugin pazaryeri duman testi: `pnpm test:docker:release-plugin-marketplace`, yerel bir fixture pazaryerinden kurulum yapar, kurulu plugini günceller, kaldırır ve kurulum meta verileri temizlendiğinde plugin CLI'ının kaybolduğunu doğrular.
- Skill kurulum duman testi: `pnpm test:docker:skill-install`, paketlenmiş OpenClaw tarball'ını Docker'da global olarak kurar, yapılandırmada yüklenen arşiv kurulumlarını devre dışı bırakır, arama yoluyla mevcut canlı ClawHub skill slug'ını çözümler, `openclaw skills install` ile kurar ve kurulu skill'i ve `.clawhub` kaynak/kilit meta verilerini doğrular.
- Güncelleme kanalı değiştirme duman testi: `pnpm test:docker:update-channel-switch`, paketlenmiş OpenClaw tarball'ını Docker'da global olarak kurar, paket `stable` kanalından git `dev` kanalına geçer, kalıcı kanalın ve güncelleme sonrası pluginin çalıştığını doğrular, ardından paket `stable` kanalına geri döner ve güncelleme durumunu denetler.
- Yükseltmeden sağ çıkma duman testi: `pnpm test:docker:upgrade-survivor`, paketlenmiş OpenClaw tarball'ını; ajanlar, kanal yapılandırması, plugin izin listeleri, eski plugin bağımlılık durumu ve mevcut çalışma alanı/oturum dosyaları içeren değiştirilmiş bir eski kullanıcı fixture'ının üzerine kurar. Canlı sağlayıcı veya kanal anahtarları olmadan paket güncellemesini ve etkileşimsiz doctor'ı çalıştırır, ardından bir geri döngü Gateway'i başlatır ve yapılandırma/durum korumasının yanı sıra başlangıç/durum bütçelerini denetler.
- Yayımlanmış yükseltmeden sağ çıkma duman testi: `pnpm test:docker:published-upgrade-survivor`, varsayılan olarak `openclaw@latest` sürümünü kurar, gerçekçi mevcut kullanıcı dosyalarını yerleştirir, yerleşik bir komut tarifiyle bu temel sürümü yapılandırır, ortaya çıkan yapılandırmayı doğrular, yayımlanmış kurulumu aday tarball'a günceller, etkileşimsiz doctor'ı çalıştırır, `.artifacts/upgrade-survivor/summary.json` yazar, ardından bir geri döngü Gateway'i başlatır ve yapılandırılmış amaçları, durum korumasını, başlangıcı, `/healthz`, `/readyz` ve RPC durum bütçelerini denetler. Tek bir temel sürümü `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC` ile geçersiz kılın; toplu zamanlayıcıdan `openclaw@2026.5.2 openclaw@2026.4.23 openclaw@2026.4.15` gibi tam yerel temel sürümleri `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` ile genişletmesini ve `reported-issues` gibi sorun biçimli fixture'ları `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS` ile genişletmesini isteyin; bildirilen sorunlar kümesi, otomatik harici OpenClaw plugin kurulum onarımı için `configured-plugin-installs` öğesini içerir. Paket Kabulü bunları `published_upgrade_survivor_baseline`, `published_upgrade_survivor_baselines` ve `published_upgrade_survivor_scenarios` olarak sunar; `last-stable-4` veya `all-since-2026.4.23` gibi meta temel sürüm belirteçlerini çözümler ve Tam Sürüm Doğrulaması, sürüm dayanıklılık paketi geçidini `last-stable-4 2026.4.23 2026.5.2 2026.4.15` ile `reported-issues` olarak genişletir.
- Oturum çalışma zamanı bağlamı duman testi: `pnpm test:docker:session-runtime-context`, gizli çalışma zamanı bağlamı transkriptinin kalıcılığını ve etkilenen yinelenmiş istem yeniden yazma dallarının doctor tarafından onarılmasını doğrular.
- Bun global kurulum duman testi: `bash scripts/e2e/bun-global-install-smoke.sh`, mevcut ağacı paketler, yalıtılmış bir ana dizine `bun install -g` ile kurar ve `openclaw infer image providers --json` komutunun takılmak yerine paketle gelen görüntü sağlayıcılarını döndürdüğünü doğrular. Önceden oluşturulmuş bir tarball'ı `OPENCLAW_BUN_GLOBAL_SMOKE_PACKAGE_TGZ=/path/to/openclaw-*.tgz` ile yeniden kullanın, ana makine derlemesini `OPENCLAW_BUN_GLOBAL_SMOKE_HOST_BUILD=0` ile atlayın veya `dist/` öğesini oluşturulmuş bir Docker görüntüsünden `OPENCLAW_BUN_GLOBAL_SMOKE_DIST_IMAGE=openclaw-dockerfile-smoke:local` ile kopyalayın.
- Kurucu Docker duman testi: `bash scripts/test-install-sh-docker.sh`, kök, güncelleme ve doğrudan npm container'ları arasında tek bir npm önbelleği paylaşır. Güncelleme duman testi, aday tarball'a yükseltmeden önce kararlı temel sürüm olarak varsayılan biçimde npm `latest` kullanır. Yerel olarak `OPENCLAW_INSTALL_SMOKE_UPDATE_BASELINE=2026.4.22` ile veya GitHub'da Install Smoke iş akışının `update_baseline_version` girdisiyle geçersiz kılın. Kök olmayan kurucu denetimleri, köke ait önbellek girdilerinin kullanıcıya yerel kurulum davranışını maskelememesi için yalıtılmış bir npm önbelleği tutar. Yerel yeniden çalıştırmalarda kök/güncelleme/doğrudan npm önbelleğini yeniden kullanmak için `OPENCLAW_INSTALL_SMOKE_NPM_CACHE_DIR=/path/to/cache` ayarlayın.
- Install Smoke CI, yinelenen doğrudan npm global güncellemesini `OPENCLAW_INSTALL_SMOKE_SKIP_NPM_GLOBAL=1` ile atlar; doğrudan `npm install -g` kapsamı gerektiğinde betiği yerel olarak bu ortam değişkeni olmadan çalıştırın.
- Ajanların paylaşılan çalışma alanını silme CLI duman testi: `pnpm test:docker:agents-delete-shared-workspace` (betik: `scripts/e2e/agents-delete-shared-workspace-docker.sh`), varsayılan olarak kök Dockerfile görüntüsünü oluşturur, yalıtılmış bir container ana dizininde tek bir çalışma alanına sahip iki ajan yerleştirir, `agents delete --json` komutunu çalıştırır ve geçerli JSON ile çalışma alanının korunması davranışını doğrular. Install-smoke görüntüsünü `OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_IMAGE=openclaw-dockerfile-smoke:local OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_SKIP_BUILD=1` ile yeniden kullanın.
- Gateway ağı ve ana makine yaşam döngüsü: `pnpm test:docker:gateway-network` (betik: `scripts/e2e/gateway-network-docker.sh`), iki container'lı LAN WebSocket kimlik doğrulama/sağlık duman testini korur, ardından hazırlama çitini, korunmuş denetim erişimini, sürdürme kurtarmasını ve hazırlanmış aynı container durdurma/başlatmasını kanıtlamak için geri döngü Admin HTTP'yi kullanır. Yeniden başlatma denetimi, özgün kiralama süresi dolmadan tamamlanmalı; askıya alma durumunun sürece yerel olduğunu, kalıcı Gateway yapılandırması ile container kimliğinin ise korunduğunu doğrulamalı ve makine tarafından okunabilir aşama zamanlaması JSON'u üretmelidir.
- Tarayıcı CDP anlık görüntü duman testi: `pnpm test:docker:browser-cdp-snapshot` (betik: `scripts/e2e/browser-cdp-snapshot-docker.sh`), kaynak E2E görüntüsünü ve bir Chromium katmanını oluşturur, Chromium'u ham CDP ile başlatır, `browser doctor --deep` komutunu çalıştırır ve CDP rol anlık görüntülerinin bağlantı URL'lerini, imleçle tıklanabilir hâle getirilen öğeleri, iframe başvurularını ve çerçeve meta verilerini kapsadığını doğrular.
- OpenAI Responses web_search minimal akıl yürütme regresyonu: `pnpm test:docker:openai-web-search-minimal` (betik: `scripts/e2e/openai-web-search-minimal-docker.sh`), Gateway üzerinden sahte bir OpenAI sunucusu çalıştırır, `web_search` öğesinin `reasoning.effort` değerini `minimal` değerinden `low` değerine yükselttiğini doğrular, ardından sağlayıcı şeması reddini zorlar ve ham ayrıntının Gateway günlüklerinde göründüğünü denetler.
- MCP kanal köprüsü (yerleştirilmiş Gateway + stdio köprüsü + ham Claude bildirim çerçevesi duman testi): `pnpm test:docker:mcp-channels` (betik: `scripts/e2e/mcp-channels-docker.sh`)
- OpenClaw paket MCP araçları (gerçek stdio MCP sunucusu + gömülü OpenClaw profili izin verme/reddetme duman testi): `pnpm test:docker:agent-bundle-mcp-tools` (betik: `scripts/e2e/agent-bundle-mcp-tools-docker.sh`)
- Cron/alt ajan MCP temizliği (yalıtılmış cron ve tek seferlik alt ajan çalıştırmalarından sonra gerçek Gateway + stdio MCP alt sürecini sonlandırma): `pnpm test:docker:cron-mcp-cleanup` (betik: `scripts/e2e/cron-mcp-cleanup-docker.sh`)
- Pluginler (yerel yol, `file:`, yukarı taşınmış bağımlılıklara sahip npm kayıt defteri, hatalı biçimlendirilmiş npm paketi meta verileri, hareketli git başvuruları, kapsamlı ClawHub, pazaryeri güncellemeleri ve Claude paketi etkinleştirme/inceleme için kurulum/güncelleme duman testi): `pnpm test:docker:plugins` (betik: `scripts/e2e/plugins-docker.sh`)
  ClawHub bloğunu atlamak için `OPENCLAW_PLUGINS_E2E_CLAWHUB=0` ayarlayın veya varsayılan kapsamlı paket/çalışma zamanı çiftini `OPENCLAW_PLUGINS_E2E_CLAWHUB_SPEC` ve `OPENCLAW_PLUGINS_E2E_CLAWHUB_ID` ile geçersiz kılın. `OPENCLAW_CLAWHUB_URL`/`CLAWHUB_URL` olmadan test, hermetik bir yerel ClawHub fixture sunucusu kullanır.
- Plugin güncellemesi değişiklik yok duman testi: `pnpm test:docker:plugin-update` (betik: `scripts/e2e/plugin-update-unchanged-docker.sh`)
- Plugin yaşam döngüsü matrisi duman testi: `pnpm test:docker:plugin-lifecycle-matrix`, paketlenmiş OpenClaw tarball'ını yalın bir container'a kurar, bir npm plugini kurar, etkinleştirme/devre dışı bırakma durumunu değiştirir, yerel bir npm kayıt defteri üzerinden yükseltir ve düşürür, kurulu kodu siler, ardından her yaşam döngüsü aşaması için RSS/CPU metriklerini günlüğe kaydederken kaldırma işleminin eski durumu hâlâ temizlediğini doğrular.
- Yapılandırma yeniden yükleme meta verisi duman testi: `pnpm test:docker:config-reload` (betik: `scripts/e2e/config-reload-source-docker.sh`)
- Pluginler: `pnpm test:docker:plugins`; yerel yol, `file:`, yukarı taşınmış bağımlılıklara sahip npm kayıt defteri, hareketli git başvuruları, ClawHub fixture'ları, pazaryeri güncellemeleri ve Claude paketi etkinleştirme/inceleme için kurulum/güncelleme duman testini kapsar. `pnpm test:docker:plugin-update`, kurulu pluginlerin değişiklik olmayan güncelleme davranışını kapsar. `pnpm test:docker:plugin-lifecycle-matrix`, kaynakları izlenen npm plugin kurulumunu, etkinleştirmeyi, devre dışı bırakmayı, yükseltmeyi, düşürmeyi ve kodu eksik kaldırma işlemini kapsar.

Paylaşılan işlevsel görüntüyü elle önceden oluşturmak ve yeniden kullanmak için:

```bash
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local pnpm test:docker:e2e-build
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local OPENCLAW_SKIP_DOCKER_BUILD=1 pnpm test:docker:mcp-channels
```

`OPENCLAW_GATEWAY_NETWORK_E2E_IMAGE` gibi pakete özgü görüntü geçersiz kılmaları ayarlandıklarında yine önceliklidir. `OPENCLAW_SKIP_DOCKER_BUILD=1` uzak bir paylaşılan görüntüyü gösterdiğinde, görüntü yerelde zaten yoksa betikler onu çeker. QR ve kurucu Docker testleri, paylaşılan oluşturulmuş uygulama çalışma zamanı yerine paket/kurulum davranışını doğruladıkları için kendi Dockerfile'larını kullanmayı sürdürür.

Canlı model Docker çalıştırıcıları ayrıca mevcut çalışma dizinini salt okunur olarak bağlar
ve container içindeki geçici bir çalışma dizinine hazırlar. Böylece Vitest tam yerel
kaynağınıza/yapılandırmanıza karşı çalışmaya devam ederken çalışma zamanı görüntüsü küçük
kalır. Hazırlama adımı, Docker canlı çalıştırmalarının makineye özgü yapıtları
kopyalamak için dakikalar harcamaması amacıyla `.pnpm-store`, `.worktrees`, `__openclaw_vitest__` ve
uygulamaya yerel `.build` veya Gradle çıktı dizinleri gibi büyük, yalnızca yerel önbellekleri ve uygulama derleme
çıktılarını atlar. Ayrıca Gateway canlı yoklamalarının container içinde gerçek
Telegram/Discord/vb. kanal işçilerini başlatmaması için
`OPENCLAW_SKIP_CHANNELS=1` ayarlarlar.
`test:docker:live-models` yine de `pnpm test:live` çalıştırır; bu nedenle söz konusu Docker kulvarındaki Gateway
canlı kapsamını daraltmanız veya hariç tutmanız gerektiğinde
`OPENCLAW_LIVE_GATEWAY_*` öğesini de aktarın.

`test:docker:openwebui`, daha üst düzey bir uyumluluk duman testidir: OpenAI uyumlu HTTP
uç noktaları etkinleştirilmiş bir OpenClaw Gateway container'ı başlatır,
bu Gateway'e karşı sabitlenmiş bir Open WebUI container'ı başlatır, Open WebUI
üzerinden oturum açar, `/api/models` öğesinin `openclaw/default` sunduğunu doğrular ve ardından
Open WebUI'ın `/api/chat/completions` proxy'si üzerinden gerçek bir sohbet isteği gönderir. Canlı model
tamamlamasını beklemeden Open WebUI oturum açma ve model keşfinden
sonra durması gereken sürüm yolu CI denetimleri için
`OPENWEBUI_SMOKE_MODE=models` ayarlayın. Docker'ın Open WebUI görüntüsünü
çekmesi ve Open WebUI'ın kendi soğuk başlangıç kurulumunu tamamlaması gerekebileceğinden ilk çalıştırma belirgin biçimde daha yavaş olabilir.
Bu kulvar; süreç ortamı, hazırlanmış kimlik doğrulama profilleri veya açıkça belirtilmiş
bir `OPENCLAW_PROFILE_FILE` üzerinden sağlanan kullanılabilir bir canlı model anahtarı bekler.
Başarılı çalıştırmalar `{ "ok": true, "model": "openclaw/default", ... }` gibi küçük bir JSON yükü yazdırır.

`test:docker:mcp-channels` kasıtlı olarak deterministiktir ve gerçek bir
Telegram, Discord veya iMessage hesabı gerektirmez. Yerleştirilmiş bir Gateway
container'ı başlatır, `openclaw mcp serve` oluşturan ikinci bir container başlatır, ardından
yönlendirilmiş konuşma keşfini, transkript okumalarını, ek
meta verilerini, canlı olay kuyruğu davranışını, giden gönderim yönlendirmesini ve gerçek stdio MCP köprüsü üzerinden Claude tarzı
kanal + izin bildirimlerini doğrular.
Bildirim denetimi, ham stdio MCP çerçevelerini doğrudan inceler; böylece duman testi
yalnızca belirli bir istemci SDK'sının yüzeye çıkardıklarını değil, köprünün gerçekten ne ürettiğini
doğrular.

`test:docker:agent-bundle-mcp-tools` deterministiktir ve canlı bir model anahtarına ihtiyaç duymaz.
Repo Docker imajını oluşturur, konteyner içinde gerçek bir stdio MCP
yoklama sunucusu başlatır, gömülü OpenClaw paket MCP çalışma zamanı aracılığıyla
bu sunucuyu somutlaştırır, aracı çalıştırır ve ardından
`coding` ile `messaging` öğelerinin `bundle-mcp` araçlarını koruduğunu, `minimal` ile
`tools.deny: ["bundle-mcp"]` öğelerinin ise bunları filtrelediğini doğrular.

`test:docker:cron-mcp-cleanup` deterministiktir ve canlı bir
model anahtarına ihtiyaç duymaz. Gerçek bir stdio MCP yoklama sunucusuyla önceden verileri yüklenmiş bir Gateway
başlatır, yalıtılmış bir cron turu ve tek seferlik bir `sessions_spawn` alt turu çalıştırır, ardından
MCP alt işleminin her çalıştırmadan sonra sonlandığını doğrular.

Manuel ACP doğal dil iş parçacığı duman testi (CI değil):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Bu betiği regresyon/hata ayıklama iş akışları için saklayın. ACP iş parçacığı yönlendirme doğrulaması için yeniden gerekebilir; bu nedenle silmeyin.

Kullanışlı ortam değişkenleri:

- `OPENCLAW_CONFIG_DIR=...` (varsayılan: `~/.openclaw`) `/home/node/.openclaw` konumuna bağlanır
- `OPENCLAW_WORKSPACE_DIR=...` (varsayılan: `~/.openclaw/workspace`) `/home/node/.openclaw/workspace` konumuna bağlanır
- `OPENCLAW_PROFILE_FILE=...` bağlanır ve testler çalıştırılmadan önce kaynak olarak yüklenir
- `OPENCLAW_DOCKER_PROFILE_ENV_ONLY=1`, yalnızca `OPENCLAW_PROFILE_FILE` kaynağından yüklenen ortam değişkenlerini geçici yapılandırma/çalışma alanı dizinleri kullanarak ve harici CLI kimlik doğrulama bağlamaları olmadan doğrulamak için
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (varsayılan: çalıştırma zaten bir CI/yönetilen bağlama dizini kullanmıyorsa `~/.cache/openclaw/docker-cli-tools`) Docker içindeki önbelleğe alınmış CLI kurulumları için `/home/node/.npm-global` konumuna bağlanır
- `$HOME` altındaki harici CLI kimlik doğrulama dizinleri/dosyaları, `/host-auth...` altında salt okunur olarak bağlanır ve ardından testler başlamadan önce `/home/node/...` içine kopyalanır
  - Varsayılan dizinler (çalıştırma belirli sağlayıcılarla sınırlandırılmadığında kullanılır): `.factory`, `.gemini`, `.minimax`
  - Varsayılan dosyalar: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Sınırlandırılmış sağlayıcı çalıştırmaları, yalnızca `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` üzerinden çıkarılan gerekli dizinleri/dosyaları bağlar
  - `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` veya `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex` gibi virgülle ayrılmış bir listeyle manuel olarak geçersiz kılın
- Çalıştırmayı sınırlandırmak için `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`
- Konteyner içindeki sağlayıcıları filtrelemek için `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`
- Yeniden oluşturma gerektirmeyen tekrar çalıştırmalarda mevcut bir `openclaw:local-live` imajını yeniden kullanmak için `OPENCLAW_SKIP_DOCKER_BUILD=1`
- Kimlik bilgilerinin ortamdan değil profil deposundan geldiğinden emin olmak için `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`
- Open WebUI duman testi için gateway tarafından sunulan modeli seçmek üzere `OPENCLAW_OPENWEBUI_MODEL=...`
- Open WebUI duman testinde kullanılan nonce denetimi istemini geçersiz kılmak için `OPENCLAW_OPENWEBUI_PROMPT=...`
- Sabitlenmiş Open WebUI imaj etiketini geçersiz kılmak için `OPENWEBUI_IMAGE=...`

## Doküman tutarlılığı

Doküman düzenlemelerinden sonra doküman denetimlerini çalıştırın: `pnpm check:docs`.
Sayfa içi başlık denetimlerine de ihtiyaç duyduğunuzda tam Mintlify bağlantı noktası doğrulamasını çalıştırın: `pnpm docs:check-links:anchors`.

## Çevrimdışı regresyon (CI için güvenli)

Bunlar, gerçek sağlayıcılar olmadan yapılan "gerçek işlem hattı" regresyonlarıdır:

- Gateway araç çağırma (sahte OpenAI, gerçek gateway + ajan döngüsü): `src/gateway/gateway.test.ts` (senaryo: "gateway ajan döngüsü aracılığıyla uçtan uca sahte bir OpenAI araç çağrısı çalıştırır")
- Gateway sihirbazı (WS `wizard.start`/`wizard.next`, yapılandırmayı yazar + kimlik doğrulamayı zorunlu kılar): `src/gateway/gateway.test.ts` (senaryo: "sihirbazı ws üzerinden çalıştırır ve kimlik doğrulama belirteci yapılandırmasını yazar")

## Ajan güvenilirliği değerlendirmeleri (Skills)

Halihazırda "ajan güvenilirliği değerlendirmeleri" gibi davranan, CI için güvenli birkaç testimiz var:

- Gerçek gateway + ajan döngüsü üzerinden sahte araç çağırma (`src/gateway/gateway.test.ts`).
- Oturum bağlantılarını ve yapılandırma etkilerini doğrulayan uçtan uca sihirbaz akışları (`src/gateway/gateway.test.ts`).

Skills için hâlâ eksik olanlar (bkz. [Skills](/tr/tools/skills)):

- **Karar verme:** Skills istemde listelendiğinde ajan doğru skill'i seçiyor mu (veya ilgisiz olanlardan kaçınıyor mu)?
- **Uyumluluk:** ajan kullanımdan önce `SKILL.md` öğesini okuyup gerekli adımları/argümanları izliyor mu?
- **İş akışı sözleşmeleri:** araç sırasını, oturum geçmişinin aktarımını ve korumalı alan sınırlarını doğrulayan çok turlu senaryolar.

Gelecekteki değerlendirmeler öncelikle deterministik kalmalıdır:

- Araç çağrılarını + sırasını, skill dosyası okumalarını ve oturum bağlantılarını doğrulamak için sahte sağlayıcılar kullanan bir senaryo çalıştırıcısı.
- Skill odaklı küçük bir senaryo paketi (kullanma veya kaçınma, geçitleme, istem enjeksiyonu).
- İsteğe bağlı canlı değerlendirmeler (katılım isteğe bağlı, ortam değişkenleriyle geçitlenmiş) yalnızca CI için güvenli paket oluşturulduktan sonra.

## Sözleşme testleri (plugin ve kanal yapısı)

Sözleşme testleri, kayıtlı her plugin ve kanalın kendi arayüz
sözleşmesine uyduğunu doğrular. Keşfedilen tüm plugin'ler üzerinde yinelenir ve bir
yapı ve davranış doğrulama paketi çalıştırır. Varsayılan `pnpm test` birim hattı,
bu paylaşılan bağlantı noktası ve duman testi dosyalarını kasıtlı olarak atlar; paylaşılan kanal
veya sağlayıcı yüzeylerine dokunduğunuzda sözleşme komutlarını açıkça çalıştırın.

### Komutlar

- Tüm sözleşmeler: `pnpm test:contracts`
- Yalnızca kanal sözleşmeleri: `pnpm test:contracts:channels`
- Yalnızca sağlayıcı sözleşmeleri: `pnpm test:contracts:plugins`

### Kanal sözleşmeleri

`src/channels/plugins/contracts/*.contract.test.ts` konumundadır. Mevcut
üst düzey kategoriler:

- **channel-catalog** - paketlenmiş/kayıt defteri kanal kataloğu girdi meta verileri
- **plugin** (kayıt defteri destekli, parçalanmış) - temel plugin kayıt yapısı
- **surfaces-only** (kayıt defteri destekli, parçalanmış) - `actions`, `setup`, `status`, `outbound`, `messaging`, `threading`, `directory` ve `gateway` için yüzey başına yapı denetimleri
- **session-binding** (kayıt defteri destekli) - oturum bağlama davranışı
- **outbound-payload** - mesaj yükü yapısı ve normalleştirmesi
- **group-policy** (geri dönüş) - kanal başına varsayılan grup politikasının uygulanması
- **threading** (kayıt defteri destekli, parçalanmış) - iş parçacığı kimliği işleme
- **directory** (kayıt defteri destekli, parçalanmış) - dizin/kadro API'si
- **registry** ve **plugins-core.\*** - kanal plugin kayıt defteri, yükleyici ve yapılandırma yazma yetkilendirmesi iç işleyişi

Bu paketlerin kullandığı gelen gönderim yakalama ve giden yük test düzeneği yardımcıları,
`src/plugin-sdk/channel-contract-testing.ts` üzerinden dahili olarak sunulur
(npm dışında bırakılmıştır, genel bir SDK alt yolu değildir); bu dizinde bağımsız bir
`inbound.contract.test.ts` dosyası yoktur.

### Sağlayıcı sözleşmeleri

`src/plugins/contracts/*.contract.test.ts` konumundadır. Mevcut kategoriler
şunları içerir:

- **shape** - plugin manifesti, API ve çalışma zamanı dışa aktarma yapısı
- **plugin-registration** (+ paralel) - manifest kayıt senaryoları
- **package-manifest** - paket manifesti gereksinimleri
- **loader** - plugin yükleyici kurulum/kapatma davranışı
- **registry** - plugin sözleşme kayıt defteri içerikleri ve arama
- **providers** - paketlenmiş sağlayıcılar ve web araması sağlayıcıları genelinde paylaşılan sağlayıcı davranışı
- **auth-choice** - kimlik doğrulama seçimi meta verileri ve kurulum davranışı
- **provider-catalog-deprecation** - kullanımdan kaldırılmış sağlayıcı kataloğu meta verileri
- **wizard.choice-resolution**, **wizard.model-picker**, **wizard.setup-options** - sağlayıcı kurulum sihirbazı sözleşmeleri
- **embedding-provider**, **memory-embedding-provider**, **web-fetch-provider**, **tts** - yeteneğe özgü sağlayıcı sözleşmeleri
- **session-actions**, **session-attachments**, **session-entry-projection** - plugin'e ait oturum durumu sözleşmeleri
- **scheduled-turns** - plugin zamanlanmış tur meta verileri ve zaman damgası sınırları
- **host-hooks**, **run-context-lifecycle**, **runtime-import-side-effects**, **runtime-seams** - plugin ana makine/çalışma zamanı yaşam döngüsü ve içe aktarma sınırı sözleşmeleri
- **extension-runtime-dependencies** - eklentiler için çalışma zamanı bağımlılığı yerleşimi

### Ne zaman çalıştırılmalı

- plugin-sdk dışa aktarımları veya alt yolları değiştirildikten sonra
- Bir kanal veya sağlayıcı plugin'i eklendikten ya da değiştirildikten sonra
- Plugin kaydı veya keşfi yeniden düzenlendikten sonra

Sözleşme testleri CI'da çalışır ve gerçek API anahtarları gerektirmez.

## Regresyon ekleme (rehber)

Canlı ortamda keşfedilen bir sağlayıcı/model sorununu düzelttiğinizde:

- Mümkünse CI için güvenli bir regresyon ekleyin (sahte/taslak sağlayıcı veya tam istek yapısı dönüşümünü yakalayın)
- Sorun doğası gereği yalnızca canlı ortamda oluşuyorsa (hız sınırları, kimlik doğrulama politikaları), canlı testi dar kapsamlı tutun ve ortam değişkenleri aracılığıyla isteğe bağlı yapın
- Hatayı yakalayan en küçük katmanı hedeflemeyi tercih edin:
  - sağlayıcı istek dönüştürme/yeniden oynatma hatası -> doğrudan model testi
  - gateway oturum/geçmiş/araç işlem hattı hatası -> gateway canlı duman testi veya CI için güvenli gateway sahte testi
- SecretRef dolaşım koruması:
  - `src/secrets/exec-secret-ref-id-parity.test.ts`, kayıt defteri meta verilerinden (`listSecretTargetRegistryEntries()`) SecretRef sınıfı başına örneklenmiş bir hedef türetir ve ardından dolaşım segmenti exec kimliklerinin reddedildiğini doğrular.
  - `src/secrets/target-registry-data.ts` içine yeni bir `includeInPlan` SecretRef hedef ailesi eklerseniz bu testteki `classifyTargetClass` öğesini güncelleyin. Test, yeni sınıfların sessizce atlanamaması için sınıflandırılmamış hedef kimliklerinde kasıtlı olarak başarısız olur.

## İlgili

- [Canlı test](/tr/help/testing-live)
- [Güncellemeleri ve plugin'leri test etme](/tr/help/testing-updates-plugins)
- [CI](/tr/ci)
