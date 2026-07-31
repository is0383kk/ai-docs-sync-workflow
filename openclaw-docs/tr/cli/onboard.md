---
read_when:
    - Çıkarım hizmetini kurmak, ardından kurulumu OpenClaw ile tamamlamak istiyorsunuz
summary: '`openclaw onboard` için CLI başvurusu (etkileşimli ilk kurulum)'
title: İlk Kurulum
x-i18n:
    generated_at: "2026-07-26T23:13:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ec5cfc564aa14041d1aa67a978a4661e6105b7119a942940f71197c695e788b
    source_path: cli/onboard.md
    workflow: 16
---

# `openclaw onboard`

Önce çıkarımı kuran yönlendirmeli kurulum: mevcut yapay zekâ erişimini algılar,
canlı bir tamamlamayı zorunlu kılar, yalnızca çalışan rotayı kalıcı hâle getirir ve ardından
geri kalanını yapılandırmak için OpenClaw'ı başlatır. `openclaw setup`, yeni
sistemlerde veya bir ilk katılım seçeneği bulunduğunda bu akışa ulaşır; yapılandırılmış sistemler,
sistem aracısı sohbeti için doğrudan `openclaw setup` kullanır. `openclaw setup --baseline` yalnızca
temel yapılandırmayı/çalışma alanını yazar.

<CardGroup cols={2}>
  <Card title="CLI ilk katılım merkezi" href="/tr/start/wizard" icon="rocket">
    Etkileşimli CLI akışının adım adım açıklaması.
  </Card>
  <Card title="İlk katılıma genel bakış" href="/tr/start/onboarding-overview" icon="map">
    OpenClaw ilk katılım bileşenlerinin birlikte nasıl çalıştığı.
  </Card>
  <Card title="CLI kurulum başvurusu" href="/tr/start/wizard-cli-reference" icon="book">
    Çıktılar, iç işleyiş ve adım başına davranış.
  </Card>
  <Card title="CLI otomasyonu" href="/tr/start/wizard-cli-automation" icon="terminal">
    Etkileşimsiz bayraklar ve betikli kurulumlar.
  </Card>
  <Card title="macOS uygulaması ilk katılımı" href="/tr/start/onboarding" icon="apple">
    macOS menü çubuğu uygulamasının ilk katılım akışı.
  </Card>
</CardGroup>

## Örnekler

```bash
openclaw onboard
openclaw onboard --tui
openclaw onboard --classic
openclaw onboard --modern
openclaw onboard --flow quickstart
openclaw onboard --flow manual
openclaw onboard --flow import
openclaw onboard --import-from hermes --import-source ~/.hermes
openclaw onboard --skip-bootstrap
openclaw onboard recommendations --json
openclaw onboard recommendations acknowledge
openclaw onboard recommendations acknowledge --retry "<failed-id>"
openclaw onboard recommendations refresh
openclaw onboard --mode remote --remote-url wss://gateway-host:18789
```

`openclaw onboard recommendations`, ilk katılım sırasında depolanan bekleyen uygulama önerisi
eşleşmelerini okur. İlk çalıştırma önyüklemesinin kullandığı makine tarafından okunabilir liste için
`--json` ekleyin. Komut, yüklü uygulamaları yeniden taramaz veya bir
modeli çağırmaz. Çıktısı yalnızca doğrulanmış kurulum kimliklerini, kaynağı ve katmanı içerir;
güvenilmeyen pazar yeri metinlerini, model gerekçelerini ve yerel uygulama
etiketlerini kasıtlı olarak dışarıda bırakır. Öneri teklifi yanıtlandıktan sonra komut
boş bir liste döndürür ve gelecekteki ilk katılım çalıştırmaları bu adımı tamamen atlar.
`openclaw onboard recommendations refresh`, depolanan teklifi temizler; böylece bir sonraki
ilk katılım çalıştırması yüklü uygulamaları yeniden tarar ve yeni bir teklif oluşturur.

Yeni çalışma alanları, öneri seçimini önyükleme görüşmesine erteler.
Bu görüşme kullanıcının seçimlerini işledikten sonra
`openclaw onboard recommendations acknowledge`, depolanan teklifi yanıtlanmış olarak işaretler.
Onay işlemi idempotenttir. Seçilen bir kurulum başarısız olursa başarısız olan her
opak kimliği `--retry <id...>` ile iletin; başarılı ve reddedilen eşleşmeler tüketilir,
başarısız eşleşmeler ise daha sonraki bir ilk katılım çalıştırması için beklemede kalır. Bilinmeyen kimlikler,
depolanan teklifi değiştirmeden başarısız olur. Kesintiye uğramış bir ClawHub beceri
kurulumundan sonra mevcut bir hedef, yalnızca aynı
yayıncı nitelikli öneri kimliği için `openclaw skills verify "@owner/slug"`
başarılı olduğunda ve JSON çıktısı
`openclaw.resolution.source: "installed"` bildirdiğinde başarılı sayılır. Yalnızca kayıt defteri doğrulaması,
yerel kurulumun kanıtı değildir. Aksi hâlde bu kimliği `--retry` ile beklemede tutun ve
mevcut becerinin üzerine yazmayın.

- `--classic`: adım adım tam sihirbazı açar. Bu seçenek
  `--non-interactive` ile birlikte kullanılamaz; otomatik kurulum için `--classic` seçeneğini kullanmayın.
- `--flow quickstart`: en az sayıda istem içeren klasik sihirbazı açar, varsayılan olarak
  belirteç kimlik doğrulamasını kullanır ve depolanmış ya da açıkça belirtilmiş
  bir kimlik bilgisi geçerli değilse belirteç oluşturur. `--gateway-port`,
  `--gateway-bind`, `--gateway-auth` ve `--tailscale` gibi açıkça belirtilmiş yerel Gateway bayrakları,
  karşılık gelen depolanmış veya varsayılan hızlı başlangıç değerlerini geçersiz kılar; belirtilmeyen
  seçenekler mevcut değerlerini korur.
- `--flow manual` (`advanced` diğer adıyla): bağlantı noktası, bağlama ve kimlik doğrulama için
  tüm istemleri içeren klasik sihirbazı açar.
- `--flow import`: algılanan bir geçiş sağlayıcısını (örneğin `--import-from hermes` üzerinden Hermes) yeni bir kurulum üzerinde çalıştırır. Onaydan sonra ilk katılım; yapılandırmayı, kimlik bilgilerini, çalışma alanı dosyalarını, belleği ve becerileri özel geçici hedeflerde hazırlar; içe aktarılan çıkarım, çalışma alanı ve aracı durumu kullanıma alınmadan ve yapılandırma kaydedilmeden önce canlı bir tamamlamadan geçmelidir. Kullanıma alma öncesindeki başarısızlık veya iptal, etkin hedefi değiştirmeden bırakır. Codex eklentisi kurulumu gibi geri alınamayan harici etkinleştirme adımları daha sonra çalışır ve geçiş raporundan yeniden denenebilir. Mevcutlarsa önce yapılandırmayı, kimlik bilgilerini, oturumları ve çalışma alanı durumunu sıfırlayın. Deneme çalıştırması planları, üzerine yazma modu, doğrulanmış yedeklemeler, raporlar ve tam eşlemeler için [`openclaw migrate`](/tr/cli/migrate) kullanın.
- `--remote-url` ve `--remote-token`: klasik uzak Gateway adımını önceden doldurur ve bu çalıştırma için depolanmış uzak değerleri geçersiz kılar. URL'nin değiştirilmesi, ayrıca bir belirteç iletmediğiniz sürece depolanmış kimlik bilgilerini yeniden kullanmaz. Belirteç istemlerde maskeli kalır ve sihirbazın mevcut düz metin veya SecretRef depolama seçimini izler.
- `--tailscale-reset-on-exit` ve `--no-tailscale-reset-on-exit`: Gateway kapandığında Tailscale Serve veya Funnel yapılandırmasının sıfırlanıp sıfırlanmayacağını açıkça denetler. Her ikisinin de belirtilmemesi, etkileşimsiz yeniden çalıştırmalar sırasında mevcut ayarı korur.
- `--modern`, OpenClaw konuşmalı kurulum
  yardımcısının uyumluluk diğer adıdır. `openclaw setup` ile aynı canlı çıkarım geçidini kullanır ve
  yalnızca `--workspace`, `--accept-risk`,
  `--non-interactive` ve `--json` seçeneklerini kabul eder. Diğer kurulum bayrakları sessizce
  yok sayılmak yerine reddedilir.

## Yönlendirmeli akış

Doğrudan `openclaw onboard`, yönlendirmeli akışı başlatır. Güvenlik bildirimini gösterir,
ardından başlangıçta tek bir soru sorar: **tam erişim** (önerilir — kurulum yapay zekâ
uygulamalarını, anahtarlarını ve yerel çalışma zamanlarını otomatik olarak arar) veya **önce sor** (kurulum
çevreyi incelemeden önce bir kez sorar ya da elle yapılandırmanıza olanak tanır).
Seçim, `wizard.accessMode` olarak kalıcı hâle getirilir. Keşfe izin verildiğinde ilk katılım;
yapılandırılmış modeller, API anahtarı ortam değişkenleri ve desteklenen yerel CLI'lar
üzerinden zaten kullanılabilen yapay zekâ erişimini algılar, ardından önerilen
adayı gerçek bir tamamlamayla test eder. Bir aday başarısız olursa ilk katılım sessizce
bir sonraki kullanılabilir adayı dener ve yanıt vermeyenleri tek
satırda özetler; çalışan rota, diğer her şeyi görmek için tek tuşla kullanılabilen
bir seçenekle birlikte duyurulur.

Otomatik algılama seçenekleri tükendiğinde sağlayıcı seçici önce OpenAI,
Anthropic, xAI (Grok), Google ve OpenRouter'ı gösterir. Desteklenen diğer tüm
sağlayıcıları sağlayıcıya göre gruplandırılmış biçimde görmek için **Diğer…** seçeneğini belirleyin;
bölgeler, planlar ve kimlik doğrulama yöntemleri daha sonra ikinci bir menüde görünür.
Desteklenen tarayıcı veya cihazla oturum açma ve maskeli API anahtarı ya da belirteç yöntemleri,
aynı canlı tamamlama yolunu kullanır. OpenClaw, yalnızca test başarılı olduktan sonra
doğrulanmış model rotasını ve kimlik bilgisini kalıcı hâle getirir; başarısız bir aday,
yapılandırılmış modelin yerini almaz veya denenen kimlik bilgisini kaydetmez.
OpenClaw'ı başlatmadan çıkmak için **Şimdilik atla** seçeneğini belirleyin ve hazır olduğunuzda
`openclaw onboard` komutunu yeniden çalıştırın. Çalışma alanı ve Gateway kurulumu,
OpenClaw başlayana kadar değişmeden kalır.

Yönlendirmeli modda `--workspace <dir>`, OpenClaw'ın önerdiği çalışma alanını
ve yalıtılmış çıkarım bağlamını sağlar. OpenClaw kurulum önerisini onaylayana kadar
kalıcı hâle getirilmez. Klasik ve etkileşimsiz ilk katılım, çalışma alanlarını
normal kurulum akışları üzerinden kalıcı hâle getirir. Mevcut bir aracı kadrosuyla yeniden
çalıştırıldığında ilk katılım, yapılandırılmış filo çalışma alanını korur: klasik
sihirbaz her iki yolu da gösterir ve taşımadan önce açık onay gerektirir;
etkileşimsiz kurulum ise uyarır ve mevcut değeri korur.

Çıkarım başarılı olduktan sonra ilk katılım, desteklenen yerel yapay zekâ
araçlarındaki bellekleri denetler: Claude Code otomatik belleği, Codex birleştirilmiş bellekleri
ve Hermes bellek dosyaları. Herhangi birini bulduğunda tek bir sayfa, dizinlenmiş geri çağırma için
bunları aracı çalışma alanındaki `memory/imports/` altına kopyalamayı önerir.
Onay olmadan hiçbir şey içe aktarılmaz, daha önce içe aktarılmış dosyalar atlanır ve daha sonra
aynı yalnızca bellek kapsamını sunan Control UI [Bellek içe aktarma sayfasından](/tr/web/control-ui)
istediğiniz zaman içe aktarabilirsiniz. (Tam bir [`openclaw migrate`](/tr/cli/migrate)
çalıştırması daha geniş kapsamlıdır: yapılandırmayı, becerileri ve kimlik bilgilerini de içe
aktarabilir.) Klasik sihirbaz, çalışma alanını hazırladıktan sonra aynı sayfayı gösterir.

Çıkarım başarılı olduktan sonra (ve bellek içe aktarma teklifinin ardından) yönlendirmeli ilk katılım,
standart kurulumu otomatik olarak uygular — çalışma alanı, Gateway ve oturumlar;
konuşmalı `openclaw setup` sohbetinin "evet" yanıtı üzerine uygulayacağı planın aynısıdır —
ardından yüklü uygulamalardan eklenti ve beceri önerileri sunar; uygulama adları,
yapılandırılmış modeliniz ve ClawHub araması aracılığıyla eşleştirilir ve bu adım
[`wizard.appRecommendations`](/tr/gateway/configuration-reference#wizard) ile devre dışı bırakılabilir.
Bir macOS, Linux veya Windows masaüstü oturumunda daha sonra kimliği doğrulanmış
Control UI panosunu açar ve tarayıcı istemcisinin bağlanması için 60 saniyeye kadar
bekler. Ekransız Linux'ta veya SSH üzerinden, geri döngü Gateway'i için bir SSH bağlantı noktası
yönlendirme komutu da içeren, belirgin ve kopyalanıp yapıştırılabilir bir pano URL'si yazdırır
ve beş dakikaya kadar bekler. Başarılı bir bağlantı tarayıcıda devam eder;
erişilemeyen bir Gateway veya zaman aşımı, öncekiyle aynı terminal çıkış yoluna geri döner.
Tarayıcıya devretmeyi atlamak ve bu terminal çıkış yolunu zorlamak için `--tui` iletin.
Kurulumun uygulanması başarısız olursa ilk katılım, etkileşimli biçimde tamamlamak için
konuşmalı OpenClaw sohbetine geri döner. Kanallar, aracılar,
eklentiler ve diğer isteğe bağlı özellikler OpenClaw sohbetinin alanında kalır:
`openclaw` komutunu çalıştırın ve kanal kimlik bilgisi toplamayı
maskeli bir terminal sihirbazına devretmek için `open channel wizard for <channel>` kullanın. Model
sağlayıcısını veya kimlik doğrulamasını değiştirmek için OpenClaw'dan çıkın ve
`openclaw onboard` komutunu çalıştırın; OpenClaw, yönlendirmeli veya klasik sağlayıcı akışlarını açmaz.

Yapılandırılmış bir kurulumda `openclaw onboard` yeniden çalıştırıldığında önce mevcut
varsayılan model doğrulanır; böylece aynı akış bir doğrulama ve onarım geçişi görevi görür —
kurulumu yeniden uygulamaz, yeniden yüklemez veya Gateway hizmetini yeniden başlatmaz.
Bu denetim başarısız olursa yapılandırılmış model hiçbir zaman otomatik olarak değiştirilmez —
ilk katılım durur ve nasıl devam edileceğini sorar. Denetim çalışma alanınızın
dışında çalışır; bu nedenle çalışma alanı eklentisinin sağladığı bir model, aracıda çalışmaya
devam etse bile burada başarısız olabilir.
Sağlayıcıya özgü kimlik doğrulama, kanallar, beceriler,
uzak Gateway kurulumu, içe aktarmalar veya eksiksiz Gateway denetimleri için `openclaw onboard --classic` kullanın.
Çıkarım dışı konuşmalı kurulum ve onarım için `openclaw setup` komutunu çalıştırın;
`openclaw onboard
--modern`, aynı çıkarım geçidinden geçen bir uyumluluk diğer adıdır. Klasik
sihirbaz, isteğe bağlı olarak varsayılan modeli canlı bir tamamlamayla doğrulayabilir; ancak
OpenClaw kendi canlı çıkarım denetimini geçene kadar başlamaz.

Etkileşimli bir terminalde doğrudan `openclaw` (alt komut olmadan), yapılandırma
durumuna göre yönlendirme yapar:

- Etkin yapılandırma dosyası eksikse veya yazılmış ayar içermiyorsa (boş ya da
  yalnızca meta veri içeriyorsa) yönlendirmeli ilk katılımı başlatır.
- Yapılandırma dosyası mevcut ancak doğrulamada başarısızsa
  `openclaw doctor` yönlendirmesiyle klasik ilk katılım yolunu başlatır. OpenClaw'ın çalışan
  çıkarıma ihtiyacı vardır ve bu çıkarım öncesi durumu onarmak için kullanılmaz.
- Yapılandırma dosyası geçerliyse normal aracı TUI'sini açar. Bir aracıya
  ve modele sahip, erişilebilir ve yapılandırılmış bir Gateway; ilk katılım veya OpenClaw olmadan
  doğrudan bu kullanıcı arayüzüne gider. Yapılandırılmış bir kurulumda OpenClaw'a TUI içindeki
  `/openclaw` veya `openclaw setup` ile ulaşın.

Düz metin `ws://`; geri döngü, özel IP sabit değerleri, `.local` ve Tailnet `*.ts.net` Gateway URL'leri için kabul edilir. Diğer güvenilir özel DNS adları için ilk katılım işlemi ortamında `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` ayarlayın.

## Sıfırlama

```bash
openclaw onboard --reset
openclaw onboard --reset --reset-scope full
```

`--reset`, kurulumu çalıştırmadan önce durumu temizler. `--reset-scope`, ne kadarının temizleneceğini denetler: `config` (yalnızca yapılandırma), `config+creds+sessions` (`--reset` kapsam belirtilmeden iletildiğinde varsayılan) veya `full` (çalışma alanını da sıfırlar). Çalışma alanı yalnızca `--reset-scope full` ile sıfırlanır.

## Yerel ayar

Etkileşimli ilk katılım, sabit kurulum metinleri için CLI sihirbazının yerel ayarını kullanır. Şu sıradaki ilk boş olmayan değeri kullanır:

1. `OPENCLAW_LOCALE`
2. `LC_ALL`
3. `LC_MESSAGES`
4. `LANG`
5. İngilizceye geri dönüş

Desteklenen sihirbaz yerel ayarları `en`, `zh-CN` ve `zh-TW` şeklindedir. Yerel ayar değerleri, `zh_CN.UTF-8` gibi alt çizgili veya POSIX son ekli biçimleri kullanabilir. Ürün adları, komut adları, yapılandırma anahtarları, URL'ler, sağlayıcı kimlikleri, model kimlikleri ve plugin/kanal etiketleri değişmeden kalır.

```bash
OPENCLAW_LOCALE=zh-CN openclaw onboard
OPENCLAW_LOCALE=en openclaw onboard # Açık İngilizce geçersiz kılma
```

## Etkileşimsiz kurulum

`--non-interactive`, `--accept-risk` gerektirir (ajanların güçlü olduğunu ve tam sistem erişiminin riskli olduğunu kabul eder). `--mode` varsayılan olarak `local` değerini kullanır.

```bash
openclaw onboard --non-interactive \
  --auth-choice custom-api-key \
  --custom-base-url "https://llm.example.com/v1" \
  --custom-model-id "foo-large" \
  --custom-api-key "$CUSTOM_API_KEY" \
  --secret-input-mode plaintext \
  --custom-compatibility openai \
  --custom-image-input
```

`--custom-api-key` isteğe bağlıdır; atlanırsa ilk katılım, ortamda `CUSTOM_API_KEY` değerini denetler. OpenClaw, yaygın görüntü modellerinin kimliklerini (GPT-4o/4.1/5.x, Claude 3/4, Gemini, Qwen-VL, LLaVA, Pixtral ve benzerleri) otomatik olarak görüntü destekli olarak işaretler. Bilinmeyen özel görüntü modeli kimlikleri için `--custom-image-input` iletin veya yalnızca metin meta verisini zorunlu kılmak için `--custom-text-input` kullanın. `/v1/responses` destekleyen ancak `/v1/chat/completions` desteklemeyen OpenAI uyumlu uç noktalar için `--custom-compatibility openai-responses` kullanın; geçerli değerler `openai` (varsayılan), `openai-responses`, `anthropic` şeklindedir.

LM Studio ayrıca sağlayıcıya özgü bir anahtar bayrağına sahiptir:

```bash
openclaw onboard --non-interactive \
  --auth-choice lmstudio \
  --custom-base-url "http://localhost:1234/v1" \
  --custom-model-id "qwen/qwen3.5-9b" \
  --lmstudio-api-key "$LM_API_TOKEN" \
  --accept-risk
```

Etkileşimsiz Ollama:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

`--custom-base-url` varsayılan olarak `http://127.0.0.1:11434` değerini kullanır. `--custom-model-id` isteğe bağlıdır; atlanırsa ilk katılım, Ollama'nın önerdiği varsayılanları kullanır. `kimi-k2.5:cloud` gibi bulut modeli kimlikleri de burada çalışır.

Sağlayıcı anahtarlarını düz metin yerine başvuru olarak saklayın:

```bash
openclaw onboard --non-interactive \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --accept-risk
```

`--secret-input-mode ref` ile ilk katılım, düz metin anahtar değerleri yerine ortam destekli başvurular yazar: kimlik doğrulama profili destekli sağlayıcılarda `keyRef: { source: "env", provider: "default", id: <envVar> }`; özel sağlayıcılarda ise aynı biçimde `models.providers.<id>.apiKey` yazar (örneğin `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`). Sözleşme: ilk katılım işleminin ortamında sağlayıcı ortam değişkenini ayarlayın (örneğin `OPENAI_API_KEY`) ve bu ortam değişkeni ayarlı olmadığı sürece satır içi anahtar bayrağını ayrıca iletmeyin; eşleşen ortam değişkeni olmadan bir bayrak değeri iletilirse yönlendirmeyle birlikte hızla hata oluşur.

### Gateway kimlik doğrulaması (etkileşimsiz)

- `--gateway-auth token --gateway-token <token>`, düz metin bir belirteç saklar. `token` varsayılan kimlik doğrulama modudur.
- `--gateway-auth token --gateway-token-ref-env <name>`, `gateway.auth.token` değerini bir ortam SecretRef'i olarak saklar. İlk katılım işleminin ortamında bu adda boş olmayan bir ortam değişkeni gerektirir.
- `--gateway-token` ve `--gateway-token-ref-env` birbirini dışlar.
- `--install-daemon` ile: SecretRef tarafından yönetilen bir `gateway.auth.token` doğrulanır ancak gözetmen hizmeti ortamı meta verilerinde çözümlenmiş düz metin olarak kalıcılaştırılmaz; başvuru çözümlenemezse kurulum, düzeltme yönlendirmesiyle güvenli biçimde başarısız olur. Hem `gateway.auth.token` hem de `gateway.auth.password` yapılandırılmışsa ve `gateway.auth.mode` ayarlanmamışsa mod açıkça ayarlanana kadar kurulum engellenir.
- Yerel ilk katılım, yapılandırmaya `gateway.mode="local"` yazar. Daha sonraki bir yapılandırma dosyasında `gateway.mode` eksikse bu, geçerli bir yerel mod kısayolu değil, yapılandırma hasarı veya tamamlanmamış bir elle düzenleme anlamına gelir.
- Yerel ilk katılım, seçilen kurulum yolunun gerektirdiği indirilebilir pluginleri kurar (örneğin bu kimlik doğrulama seçenekleri için bir Codex veya Copilot çalışma zamanı plugini). Uzak ilk katılım yalnızca uzak Gateway için bağlantı bilgilerini yazar; yerel plugin paketlerini hiçbir zaman kurmaz.
- `--allow-unconfigured` ayrı bir `openclaw gateway run` kaçış yoludur; ilk katılımın `gateway.mode` adımını atlamasına izin vermez.

```bash
export OPENAI_API_KEY="your-provider-key"
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN \
  --accept-risk
```

### Yerel Gateway durumu

- `--skip-health` iletmediğiniz sürece ilk katılım, başarıyla çıkmadan önce erişilebilir bir yerel Gateway bekler.
- `--install-daemon`, önce yönetilen Gateway kurulum yolunu başlatır. Bu olmadan yerel bir Gateway zaten çalışıyor olmalıdır (örneğin `openclaw gateway run`).
- Otomasyonda yalnızca yapılandırma/çalışma alanı/önyükleme yazımlarını istiyorsanız `--skip-health` beklemeyi atlar.
- `--skip-bootstrap`, `agents.defaults.skipBootstrap: true` değerini ayarlar ve `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md` ve `BOOTSTRAP.md` oluşturulmasını atlar.
- Yerel Windows'ta `--install-daemon`, önce Zamanlanmış Görevler'i dener; görev oluşturma reddedilirse kullanıcı başına Başlangıç klasörü oturum açma öğesine geri döner.

### Etkileşimli başvuru modu

- İstendiğinde **Gizli bilgi başvurusu kullan** seçeneğini, ardından **Ortam değişkeni** veya yapılandırılmış bir gizli bilgi sağlayıcısını (`file` ya da `exec`) seçin.
- İlk katılım, başvuruyu kaydetmeden önce hızlı bir ön kontrol doğrulaması çalıştırır ve başarısızlık durumunda yeniden denemenize olanak tanır.

### Z.AI uç noktası seçenekleri

<Note>
`--auth-choice zai-api-key`, anahtarınız için en iyi Z.AI uç noktasını ve modelini otomatik olarak algılar: Coding Plan uç noktaları `zai/glm-5.2` seçeneğini tercih eder (kullanılamıyorsa `glm-5.1` seçeneğine geri döner); genel API uç noktaları varsayılan olarak `zai/glm-5.1` değerini kullanır. Bir Coding Plan uç noktasını zorunlu kılmak için doğrudan `zai-coding-global` veya `zai-coding-cn` seçin.
</Note>

```bash
# İstemsiz uç nokta seçimi
openclaw onboard --non-interactive \
  --auth-choice zai-coding-global \
  --zai-api-key "$ZAI_API_KEY"

# Diğer Z.AI uç noktası seçenekleri: zai-coding-cn, zai-global, zai-cn
```

Mistral:

```bash
openclaw onboard --non-interactive \
  --auth-choice mistral-api-key \
  --mistral-api-key "$MISTRAL_API_KEY"
```

## Ek etkileşimsiz bayraklar

Belirteç tabanlı model kimlik doğrulaması (`--auth-choice token` ile kullanılır):

| Bayrak                            | Açıklama                                                                                                                 |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `--token-provider <id>`         | Belirteci veren belirteç sağlayıcısının kimliği                                                                                         |
| `--token <token>`               | Model kimlik doğrulaması için belirteç değeri                                                                                        |
| `--token-profile-id <id>`       | Kimlik doğrulama profili kimliği (varsayılan `<provider>:manual`; sağlayıcıya ait bazı akışlar `anthropic:default` gibi kendi varsayılanlarını kullanır) |
| `--token-expires-in <duration>` | İsteğe bağlı belirteç süre sonu süresi (ör. `365d`, `12h`)                                                                         |

Cloudflare AI Gateway: `--cloudflare-ai-gateway-account-id <id>`, `--cloudflare-ai-gateway-gateway-id <id>`.

Arka plan programı kurulum denetimi: `--no-install-daemon` / `--skip-daemon` (takma adlar; Gateway hizmeti kurulumunu atlar), `--daemon-runtime <node>`.

Skills: `--node-manager <npm|pnpm|bun>` (varsayılan `npm`), `--skip-skills`.

Kullanıcı arayüzü ve kanca kurulumu: `--skip-ui` (Control UI/TUI istemlerini atlar), `--skip-hooks` (Webhook/kanca kurulumunu atlar), `--skip-channels`, `--skip-search`.

Çıktı: `--suppress-gateway-token-output`, belirteç içeren Gateway/kullanıcı arayüzü çıktısını (belirteç ipuçları, gömülü belirteçli otomatik oturum açma URL'si ve Control UI'ın otomatik başlatılması) engeller; paylaşılan terminallerde ve CI'da kullanışlıdır.

<Note>
`--json`, yönlendirmeli veya klasik ilk katılımda etkileşimsiz modu ifade etmez.
`--modern` ile JSON, tek seferlik bir OpenClaw genel bakışıdır ve bu
tek sonuçtan sonra çıkar. Diğer betikler için `--non-interactive` kullanın.
</Note>

## Sağlayıcı ön filtrelemesi

Bir kimlik doğrulama seçeneği tercih edilen bir sağlayıcıyı ifade ettiğinde ilk katılım, varsayılan model ve izin listesi seçicilerini o sağlayıcının modelleriyle önceden filtreler. Filtre ayrıca aynı pluginin sahip olduğu diğer sağlayıcılarla da eşleşir; bu, `volcengine`/`volcengine-plan` ve `byteplus`/`byteplus-plan` gibi Coding Plan varyantlarını kapsar. Tercih edilen sağlayıcı filtresi yüklenmiş model döndürmezse ilk katılım, seçiciyi boş bırakmak yerine filtrelenmemiş kataloğa geri döner.

## Web araması takip adımları

Bazı web araması sağlayıcıları, ilk katılım sırasında sağlayıcıya özgü takip istemlerini tetikler:

- **Grok**, aynı xAI kimlik doğrulaması ve bir `x_search` model seçeneğiyle isteğe bağlı `x_search` kurulumu sunabilir.
- **Kimi**, Moonshot API bölgesini (`api.moonshot.ai` veya `api.moonshot.cn`) ve varsayılan Kimi web araması modelini sorabilir.

## Diğer davranışlar

- Yerel ilk katılımın DM kapsamı davranışı: [CLI kurulum başvurusu](/tr/start/wizard-cli-reference#outputs-and-internals).
- En hızlı ilk sohbet: `openclaw dashboard` (Control UI, kanal kurulumu yok).
- Özel sağlayıcı: listelenmeyen barındırılan sağlayıcılar dâhil olmak üzere OpenAI veya Anthropic uyumlu herhangi bir uç noktaya bağlanın. Canlı bir yoklama aracılığıyla otomatik algılama için **Bilinmeyen** uyumluluğunu kullanın.
- Hermes durumu algılanırsa ilk katılım bir geçiş akışı sunar (yukarıdaki `--flow import` bölümüne bakın).

## Yaygın takip komutları

Daha sonra hedefli ve çıkarım dışı değişiklikler için `openclaw configure`, yalnızca kanal kurulumu için `openclaw
channels add` kullanın. Model sağlayıcısı veya kimlik doğrulama rotası değişiklikleri için
bunun yerine `openclaw onboard` çalıştırın.

```bash
openclaw channels add
openclaw configure
openclaw agents add <name>
```
