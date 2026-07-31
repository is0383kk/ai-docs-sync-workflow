---
read_when:
    - Çıkarım kurulumunu tamamladınız ve geri kalanını OpenClaw'ın yapılandırmasını istiyorsunuz
    - OpenClaw'u yerel kurulum aracısıyla incelemeniz veya onarmanız gerekiyor
    - Mesaj kanalı kurtarma modunu tasarlıyor veya etkinleştiriyorsunuz
summary: Çıkarım destekli OpenClaw kurulum ve onarım yardımcısı için CLI referansı ve güvenlik modeli
title: OpenClaw kurulum aracısı
x-i18n:
    generated_at: "2026-07-26T23:15:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9578d1493ff514ea6dd07dae995bf83443e9e17f2c2134bc801faa45254615bf
    source_path: cli/openclaw.md
    workflow: 16
---

# `openclaw setup`

OpenClaw, yerel kurulum, onarım ve yapılandırma için "OpenClaw" olarak konuşan yerleşik bir sistem aracısıyla (önceki adıyla Crestodian) birlikte gelir. Yalnızca geçerli varsayılan model gerçek bir turu tamamladıktan sonra başlar.
Yeni kurulumlar önce çıkarım işlevini kurar; hatalı biçimlendirilmiş yapılandırmalar klasik doctor yolunda kalır.

## Ne zaman başlar

Alt komut olmadan `openclaw` çalıştırıldığında yönlendirme, yapılandırma durumuna göre yapılır:

- Yapılandırma yoksa veya mevcut olup kullanıcı tarafından yazılmış ayar içermiyorsa (boşsa ya da yalnızca `$schema`/`meta` anahtarlarını içeriyorsa): canlı yapay zekâ doğrulamasıyla yönlendirmeli ilk katılımı başlatır.
- Yapılandırma mevcut ancak doğrulamadan geçemiyorsa: sorunları bildiren ve `openclaw doctor` komutuna yönlendiren klasik ilk katılımı başlatır.
- Yapılandırma mevcut ve geçerliyse: normal aracı TUI'sini açar. Varsayılan aracısında model bulunan ve erişilebilen yapılandırılmış bir Gateway, ilk katılım veya OpenClaw olmadan doğrudan bu kullanıcı arayüzüne gider. OpenClaw'a daha sonra ulaşmak için TUI içinde `/openclaw` komutunu kullanın veya doğrudan `openclaw setup` komutunu çalıştırın.

`openclaw setup` çalıştırıldığında önce yapılandırılmış varsayılan model canlı olarak test edilir. Başarılı bir tur OpenClaw'ı başlatır. Etkileşimli bir hata, yönlendirmeli çıkarım kurulumunu açar ve bir aday başarılı olduktan sonra denetimi OpenClaw'a devreder. Tek seferlik, JSON ve diğer etkileşimsiz istekler, çıkarım kullanılamadığında `openclaw onboard` komutunun çalıştırılması talimatıyla başarısız olur. `openclaw --help` ve `openclaw --version` normal hızlı yollarını korur.

Etkileşimsiz yalın `openclaw` (TTY yokken), kök yardımı yazdırmak yerine kısa bir iletiyle çıkar: yeni veya geçersiz bir kurulumda etkileşimsiz ilk katılıma, yapılandırma geçerliyse `openclaw agent --local ...` komutuna yönlendirir.

`openclaw onboard --modern`, OpenClaw için uyumluluk takma adı olarak kalır ancak aynı çıkarım geçidini kullanır: çalışan çıkarım sohbeti açar, etkileşimli hatalar yönlendirmeli çıkarım kurulumunu başlatır ve etkileşimsiz hatalar ilk katılım yönlendirmesiyle çıkar. `openclaw onboard --classic` tam adım adım sihirbazı açar.

## OpenClaw ne gösterir

Etkileşimli OpenClaw, OpenClaw sohbet arka ucuyla `openclaw tui` ile aynı TUI kabuğunu açar. Başlangıç karşılaması şunları kapsar:

- yapılandırmanın geçerliliği ve varsayılan aracı
- OpenClaw'ın kullandığı doğrulanmış model
- ilk başlangıç yoklamasından Gateway erişilebilirliği
- önerilen bir sonraki hata ayıklama eylemi

Yalnızca başlamak için gizli bilgileri dökmez veya Plugin CLI komutlarını yüklemez.

Ayrıntılı envanter için `status` komutunu kullanın: yapılandırma yolu, belge/kaynak yolları, yerel CLI yoklamaları, anahtar/token varlığı, aracılar, model ve Gateway ayrıntıları.

OpenClaw, normal aracılarla aynı referans keşfini kullanır: bir Git çalışma kopyasında yerel `docs/` ve kaynak ağacını gösterir; bir npm kurulumunda paketlenmiş belgeleri kullanır ve [https://github.com/openclaw/openclaw](https://github.com/openclaw/openclaw) bağlantısını sunar; belgeler yeterli olmadığında kaynağı kontrol etme yönlendirmesi sağlar.

## Örnekler

```bash
openclaw
openclaw setup
openclaw setup --json
openclaw setup --message "modeller"
openclaw setup --message "yapılandırmayı doğrula"
openclaw setup --message "~/Projects/work çalışma alanını kur" --yes
openclaw setup --message "varsayılan modeli openai/gpt-5.6 olarak ayarla" --yes
openclaw onboard --modern
```

OpenClaw TUI içinde:

```text
durum
sağlık
doctor
yapılandırmayı doğrula
kurulum
~/Projects/work çalışma alanını kur
config set gateway.port 19001
config set-ref gateway.auth.token env OPENCLAW_GATEWAY_TOKEN
gateway durumu
gateway'i yeniden başlat
aracılar
~/Projects/work çalışma alanıyla work aracısını oluştur
modeller
model sağlayıcısını yapılandır
varsayılan modeli openai/gpt-5.6 olarak ayarla
kanallar
slack kanal bilgisi
slack'e bağlan
slack için kanal sihirbazını aç
pluginleri listele
slack pluginlerini ara
plugin install clawhub:openclaw-codex-app-server
work aracısıyla konuş
~/Projects/work için aracıyla konuş
denetim
çık
```

## İşlemler ve onay

OpenClaw, yapılandırmayı geçici biçimde düzenlemek yerine tür belirtilmiş işlemleri kullanır.

Salt okunur işlemler hemen çalışır: genel bakışı gösterme, aracıları listeleme, kurulu Pluginleri listeleme, ClawHub Pluginlerini arama, model/arka uç durumunu gösterme, durum/sağlık denetimleri çalıştırma, Gateway erişilebilirliğini kontrol etme, etkileşimli düzeltmeler olmadan doctor çalıştırma, yapılandırmayı doğrulama ve denetim günlüğü yolunu gösterme.

Yönlendirmeli kanal kurulumunu (`connect telegram`) başlatma da hemen çalışır. Sihirbazı açık yanıtları toplar ve ortaya çıkan yazma işlemlerini yönetir.

Kalıcı işlemler, konuşma yoluyla onay (veya doğrudan bir komut için `--yes`) gerektirir: yapılandırma yazma, `config set`, `config set-ref`, kurulum/ilk katılım önyüklemesi, varsayılan modeli değiştirme, Gateway'i başlatma/durdurma/yeniden başlatma, aracı oluşturma ve Plugin yükleme.

Doctor onarımları OpenClaw içinde kullanılamaz; çünkü oturuma güç veren sağlayıcı, kimlik doğrulama veya varsayılan aracı çıkarım rotasını yeniden yazabilirler. OpenClaw'dan çıkın ve bir terminalde `openclaw doctor --fix` komutunu çalıştırın. Salt okunur `doctor`, OpenClaw içinde kullanılabilir durumda kalır.

Yeni aracılar, canlı olarak doğrulanmış varsayılan çıkarım rotasını devralır. `openclaw` ve `crestodian` aracı kimlikleri sistem aracısı için ayrılmıştır ve normal aracılar olarak oluşturulamaz. Kullanımdan kaldırılmış kimlik, eski bir yapılandırmanın onu sahiplenememesi için engellenmiş durumda kalır.

`config set` ve `config set-ref`, kullanıcıların değiştirebildiği tüm ayarları değiştirebilir;
yalnızca insanlara yönelik kısa bir ret listesi vardır: `$include`, `auth.*`, `env.*`, `models.*`
ve `secrets.*`; kimlik bilgisi materyali, alternatif yapılandırma ekleme veya çıkarım yönlendirmesini besleyen sağlayıcı/katalog tanımları taşıdıkları için reddedilmeye devam eder.
Çıkarım yönlendirmesinin kendisi de korunur: varsayılan model rotaları (`agents.defaults` model/parametreler/çalışma zamanı alanları) ile etkin varsayılan rotayı destekleyen aracının yönlendirme alanları ve aracı kimliği/topolojisi alanları (`id`, `agentDir`, `default`) reddedilir. Diğer aracıların yönlendirme alanları onayla yazılabilir durumda kalır. Gateway ve kanal kimlik doğrulaması normal yapılandırma yüzeyleri olarak kalır. Önceden yapılandırılmış bir rota için `set default model <provider/model>` komutunu kullanın; rota kaydedilmeden önce canlı olarak test edilir. Sağlayıcı/kimlik doğrulama erişimini yapılandırmak veya onarmak için OpenClaw'dan çıkın ve `openclaw onboard` komutunu çalıştırın.

`plugins.entries.<id>.*` yazma işlemlerine (kurulu Pluginlerin etkinleştirilmesi/devre dışı bırakılması/yapılandırılması), ilgili Plugin etkin çıkarım rotasını desteklemediği sürece izin verilir. Plugin yükleme kaynakları ve yükleme ilkesi, güven sınırlarını tür belirtilmiş Plugin yükleme iş akışında korur. Rotayı destekleyen Pluginin kaldırılması da aynı nedenle reddedilir; OpenClaw'dan çıkın ve terminalden `openclaw plugins uninstall <id>` komutunu çalıştırın.

Onay kendi ifadelerinizle verilir: açık yanıtlar ("evet", "elbette", "devam et", "şimdi değil") kapalı ve belirlenimci bir listeden çözümlenir. Yapılandırılmış rota ayrı bir tamamlama çağrısını desteklediğinde diğer yanıtlar yalnızca iletiniz ve bekleyen teklif kullanılarak sınıflandırılabilir; kendi kendini onaylayamayan konuşma modeli tarafından asla sınıflandırılmaz. Sınıflandırılamayan veya belirsiz yanıtlar teklifi beklemede tutar ve konuşma yeniden sorar.

### Değişiklik geçmişi

OpenClaw'a Sor sayfası; yakın zamanda uygulanmış sistem aracısı işlemlerini, Doctor geçişlerini, Ayarlar ve CLI yapılandırma yazma işlemlerini ve `openclaw.json` üzerinde yapılan elle düzenlemeleri gösterebilir. Yapılandırma günlüğü; Gateway izleme yaparken, OpenClaw'a ait bir yazma işlemi sırasında veya çevrimdışı bir düzenlemeden sonraki ilk başlangıçta dış düzenlemeleri algılar.

Geçmiş, paylaşılan `~/.openclaw/state/openclaw.sqlite` veritabanının `diagnostic_events` tablosunda, `system-agent-audit` ve `config-audit` kapsamları altında saklanır. Her kapsam en son 50,000 kaydı tutar. Keşif ve salt okunur işlemler buna dahil değildir. Gizli bilgiler değişiklik geçmişinde asla görünmez; yapılandırma günlüğü kayıtları yapılandırma değerleri yerine değiştirilen yolları içerir ve değer karşılaştırması korumalı parmak izlerini kullanır.

Kanal kurulumu, gizli bir bilgiye ulaşana kadar barındırılan bir konuşma olarak çalışabilir. Yerel OpenClaw TUI, terminal sohbet girdisi görünür olduğu için hassas sihirbaz yanıtlarını kabul etmez. Seçilen kanalı maskelenmiş terminal sihirbazına taşıyarak `open channel wizard` seçeneğini hemen sunar; `openclaw channels add --channel <channel>` komutunu daha sonra da çalıştırabilirsiniz.

### Maskelenmiş kanal kurulumuna geçme

Yerel sohbet, denetimi maskelenmiş kanal sihirbazına devredebilir:

```text
slack için kanal sihirbazını aç
slack kanal bilgisi
```

`open channel wizard for <channel>`, sohbet TUI'si kapandıktan sonra maskelenmiş kanal kurulumunu açar. Kanal etiketi, kurulum durumu, ön koşullar özeti ve belge bağlantısı için önce `channel info <channel>` komutunu kullanın.

OpenClaw kendi oturumunun içinden sağlayıcı/kimlik doğrulama erişimini asla değiştirmez: oturum zaten bu çıkarım rotasına bağlıdır. Model sağlayıcısı kurulumu veya onarımı için `configure model provider`, sihirbaz başlatmadan veya yapılandırma yazmadan çıkış/ilk katılım yönlendirmesi döndürür. OpenClaw'dan çıkın ve `openclaw
onboard` komutunu çalıştırın; ilk katılım kimlik bilgilerini hazırlar ve yalnızca gerçek bir canlı turu tamamlayan rotayı kaydeder. İlk katılım başarıyla tamamlandıktan sonra OpenClaw'ı yeniden başlatın.

## Kurulum önyüklemesi

`setup`, yönlendirmeli ilk katılım çıkarımı zaten kurduktan sonra kalan çalışma alanı ve Gateway durumunu yapılandırır. Yalnızca tür belirtilmiş yapılandırma işlemleri üzerinden yazar ve önce onay ister.

```text
kurulum
~/Projects/work çalışma alanını kur
```

`setup`, doğrulanmış geçerli modeli korur. Çıkarımı yapılandırmaz veya değiştirmez.

Çıkarım yoksa veya canlı denetimi başarısız olursa OpenClaw'dan çıkın ve `openclaw onboard` komutunu çalıştırın. Yönlendirmeli ilk katılım önce yapılandırılmış modeli, ardından kimliği doğrulanmış abonelik CLI'larını, API anahtarlarını ve desteklenen diğer CLI'ları dener; her adaydan gerçek bir yanıt ister ve yalnızca başarılı bir rotayı kalıcılaştırır. OpenClaw bu sınırdan hemen sonra başlar ve ardından çalışma alanını, Gateway'i, kanalları, aracıları, Pluginleri ve diğer isteğe bağlı özellikleri yapılandırabilir.

macOS uygulaması, varsayılan aracısında zaten yapılandırılmış bir model bulunan yapılandırılmış bir Gateway'e ulaştığında bu basamakların tamamını atlar; normal aracı kullanıcı arayüzünü açar.
Yeni veya tamamlanmamış bir Gateway için uygulama, çıkarım basamaklarını `openclaw.setup.detect` ve `openclaw.setup.activate` Gateway yöntemleri üzerinden yürütür:
algılama, bulduğu her aday arka ucu listeler; etkinleştirme, bir adayı canlı olarak test eder (gerçek bir "OK ile yanıt ver" tamamlaması) ve yalnızca test başarılı olduktan sonra o rota için gereken model, kimlik bilgisi ve sağlayıcı/çalışma zamanı durumunu kalıcılaştırır. Çalışma alanı ve Gateway varsayılanları OpenClaw'a bırakılır. Başarısız bir aday yapılandırmayı asla değiştirmez; uygulama basamaklarda otomatik olarak ilerler ve son olarak Gateway'in etkin metin çıkarımı sağlayıcısı Pluginlerinden doldurulan elle anahtar/token girme adımını sunar. Seçilen sağlayıcı kendi başlangıç modelini ve yapılandırmasını yönetir; kimlik bilgisi de kaydedilmeden önce aynı şekilde doğrulanır.

Codex denetimi ve diğer isteğe bağlı Plugin özellikleri bu çıkarım etkinleştirme işleminin dışında kalır. Bunları yalnızca çıkarım çalıştıktan ve OpenClaw başladıktan sonra yapılandırın; mevcut Plugin ilkesi ve açık denetimden çıkma tercihleri çıkarım kurulumu sırasında değiştirilmez.

## Yapay zekâ konuşması

Etkileşimli OpenClaw'ın serbest biçimli konuşması, normal OpenClaw aracılarının kullandığı aracı döngüsü üzerinden çalışır ve tür belirtilmiş işlemleri sarmalayan tek bir sıfırıncı halka OpenClaw yetki aracı olan `openclaw` ile sınırlandırılır. Okuma eylemleri serbestçe çalışır, değişiklikler tam olarak ilgili işlem için konuşma yoluyla onayınızı gerektirir (bkz. İşlemler ve onay) ve uygulanan her yazma işlemi denetlenip yeniden doğrulanır. Aracı oturumu kalıcıdır; dolayısıyla OpenClaw gerçek çok turlu belleğe sahiptir. Doğrulanmış çıkarım rotası daha sonra çalışmayı durdurursa devam etmeden önce `openclaw onboard` konumuna dönün ve rotayı onarın.

Ana bilgisayar, doğal dil isteklerini işlemlere ayrıştırmaz. Komut gibi görünen metinler ve "gateway'im neden durdu?" gibi sorular dahil serbest biçimli iletiler, isteği `openclaw` aracı üzerinden tür belirtilmiş bir işleme eşleyebilen yapay zekâya gider.

Bir mutasyon beklemedeyken, yalnızca kapalı bir listedeki anlamı kesin onay veya ret ifadeleri çıkarım yapılmadan çözümlenir. Belirsiz rıza, yapılandırılmış ayrı bir tamamlama çağrısına gönderilir; bu mümkün değilse işlem güvenli biçimde reddedilir. Yapılandırılmış sihirbaz alanları ve kesin ana makine gezinme adımları, doğal dilde işlem ayrıştırma değil, kullanıcı arayüzü denetimleridir. Gizli bilgi hijyeniyle ilgili bir istisna özellikle önemlidir: hassas bir yoldaki (token'lar, anahtarlar, parolalar) tam bir `config set` hiçbir zaman bir modele ulaşmaz. Ana makine, redakte edilmiş bir öneri oluşturur ve değer, yapay zekânın görebildiği geçmişte maskelenir. Gizli bilgiler için `config set-ref <path> env <ENV_VAR>` tercih edin.

Mesaj kanalı kurtarma modu, model destekli planlayıcıyı hiçbir zaman kullanmaz. Uzaktan kurtarma deterministik kalır; böylece bozuk veya ele geçirilmiş normal bir ajan yolu, yapılandırma düzenleyicisi olarak kullanılamaz.

### CLI yürütme düzeneği güven modeli

Gömülü çalışma zamanları ve Codex uygulama sunucusu yürütme düzeneği, sıfırıncı halka kısıtlamasını doğrudan uygular: çalıştırma, yalnızca `openclaw` aracını içeren bir OpenClaw araç izin listesi taşır. OpenClaw, Codex için ayrıca bu çalıştırmadaki ortam, yerel yürütme, çoklu ajan, hedef, uygulama/plugin, beceri/MCP, web araması ve `request_user_input` yüzeylerini devre dışı bırakır. Codex yine de etkisiz yerel `update_plan` yardımcı programını ekler; bu program modelin geçici kontrol listesini güncelleyebilir ancak dosyalara veya OpenClaw yapılandırmasına yazamaz. CLI yürütme düzenekleri OpenClaw'ın izin listesini kullanmaz; bu nedenle OpenClaw yalnızca kendi araç seçimi sözleşmesiyle aynı kısıtlamayı kanıtlayabilen arka uçları kabul eder:

- Claude Code dâhil seçilebilir arka uçlar, boş bir yerel araç seçimi ve tek bir MCP aracı olan `openclaw` ile başlatılır. Claude'un oluşturduğu MCP yapılandırması `--strict-mcp-config` ile uygulanır; böylece başka hiçbir MCP sunucusu yüklenmez.
- Yerel araç bildirmeyen arka uçlar da aynı özel OpenClaw MCP sunucusunu alır.
- Her zaman etkin veya bilinmeyen yerel araçlara sahip arka uçlar, çıkarımdan önce güvenli biçimde reddedilir; bir OpenClaw oturumunu barındıramazlar.

openclaw MCP sunucusunu yalnızca OpenClaw oturumları alır; normal ajan çalıştırmaları bu aracı hiçbir zaman görmez. Dolayısıyla seçilebilir/yerel araçsız CLI arka uçları ve API anahtarlı modeller, gerçek anlamda tek araçlı döngüyü uygular. Codex uygulama sunucusu modelleri, tek bir OpenClaw yetki aracıyla birlikte etkisiz yerel planlama yardımcı programını uygular. Üç durumda da kurulum yazma işlemleri, OpenClaw'ın denetlenen onay sözleşmesiyle sınırlı kalır.

Gemini CLI normal ajanlar için kullanılabilir olmaya devam eder ancak çıkarım geçidinin gerektirdiği araçsız yoklamayı uygulayamaz; dolayısıyla OpenClaw'ı barındıramaz.

## Bir ajana geçiş

OpenClaw'dan çıkıp normal TUI'yi açmak için doğal dilde bir seçici kullanın:

```text
ajanla konuş
iş ajanıyla konuş
ana ajana geç
```

`openclaw tui`, `openclaw chat` ve `openclaw terminal` normal ajan TUI'sini doğrudan açar; OpenClaw'ı başlatmaz. Normal TUI'ye geçtikten sonra `/openclaw`, isteğe bağlı bir devam isteğiyle OpenClaw'a döner:

```text
/openclaw
/openclaw gateway'i yeniden başlat
```

## Mesaj kurtarma modu

Mesaj kurtarma modu, OpenClaw'ın mesaj kanalı giriş noktasıdır: normal ajanınız çalışmıyorken ancak güvenilir bir kanal (örneğin WhatsApp) hâlâ komut alıyorsa bunu kullanın.

Bu, konuşmaya dayalı OpenClaw ajanı değil, deterministik bir acil durum komut işleyicisidir. Yeni bir kurulumu başlatmaz veya OpenClaw sohbeti için çıkarım geçidini gevşetmez.

Desteklenen komut: `/openclaw <request>`. Kurtarma yalnızca tam olarak yazılan komut dil bilgisini kabul eder — doğal dil bir ipucuyla reddedilir, hiçbir zaman tahmin yoluyla bir işleme dönüştürülmez ve hiçbir modele başvurulmaz.

```text
Siz, güvenilir bir sahip DM'sinde: /openclaw status
OpenClaw: OpenClaw kurtarma modu. Gateway erişilebilir: hayır. Yapılandırma geçerli: hayır.
Siz: /openclaw restart gateway
OpenClaw: Plan: Gateway'i yeniden başlat. Uygulamak için /openclaw yes yanıtını verin.
Siz: /openclaw yes
OpenClaw: Uygulandı. Denetim kaydı yazıldı.
```

Ajan oluşturma işlemi yerel olarak veya kurtarma aracılığıyla da kuyruğa alınabilir:

```text
create agent work workspace ~/Projects/work model openai/gpt-5.6-sol
/openclaw create agent work workspace ~/Projects/work
```

Ajan oluşturulurken yalnızca canlı olarak doğrulanmış mevcut varsayılan model belirtilebilir. Bu rotayı devralmak için modeli belirtmeyin.

Uzaktan kurtarma bir yönetici yüzeyidir ve normal sohbet gibi değil, uzaktan yapılandırma onarımı gibi ele alınmalıdır.

Uzaktan kurtarma güvenlik sözleşmesi:

- Ajan/oturum için korumalı alan etkin olduğunda devre dışıdır; OpenClaw uzaktan kurtarmayı reddeder ve yerel CLI onarımına yönlendirir.
- Varsayılan etkin durum `auto` şeklindedir: uzaktan kurtarmaya yalnızca çalışma zamanının zaten korumalı alansız yerel yetkiye sahip olduğu güvenilir YOLO işleminde izin verilir (`tools.exec.security`, `full` olarak ve `tools.exec.ask`, `off` olarak çözümlenir; korumalı alan modu `off` değerindedir).
- Açıkça belirtilmiş bir sahip kimliği gerektirir; joker karakterli gönderen kurallarına, açık grup politikasına, kimliği doğrulanmamış Webhook'lara veya anonim kanallara izin verilmez.
- Kurtarma yalnızca sahip DM'leriyle sınırlıdır.
- Plugin arama ve listeleme salt okunurdur. Plugin kurulumu, yürütülebilir kod indirdiği için her zaman yalnızca yerelde yapılabilir (başka koşullarda etkin olsa bile kurtarmada engellenir). Plugin kaldırma hem yerel OpenClaw'da hem de kurtarmada reddedilir; bir terminalden `openclaw plugins uninstall <id>` komutunu çalıştırın.
- Uzaktan kurtarma yerel TUI'yi açamaz veya etkileşimli bir ajan oturumuna geçemez; ajan devri için yerel `openclaw` kullanın.
- Kalıcı yazma işlemleri kurtarma modunda bile onay gerektirir.
- Bekleyen onaylar tek kullanımlıktır. Aynı hesap, kanal ve gönderen için gönderilen daha yeni herhangi bir kurtarma komutu eski planı iptal eder; başarısız yürütme de onayı tüketir, bu nedenle yeniden denemek için komutu tekrar gönderin.
- Uygulanan her kurtarma işlemi denetlenir. Mesaj kanalı kurtarması; kanal, hesap, gönderen ve kaynak adresi meta verilerini kaydeder. Yapılandırmayı değiştiren işlemler ayrıca değişiklikten önceki ve sonraki yapılandırma karmalarını kaydeder.
- Gizli bilgiler hiçbir zaman geri yansıtılmaz. SecretRef incelemesi değerleri değil, kullanılabilirliği bildirir.
- Gateway çalışıyorsa kurtarma, Gateway'in türü belirlenmiş işlemlerini tercih eder; çalışmıyorsa yalnızca normal ajan döngüsüne bağlı olmayan asgari yerel onarım yüzeyini kullanır.

Kurtarma politikası yerleşiktir: yalnızca etkin çalışma zamanı YOLO olduğunda, korumalı alan kapalı olduğunda ve istek bir sahip DM'si olduğunda kullanılabilir. Bekleyen yazma onaylarının süresi 15 dakika sonra dolar. `openclaw doctor --fix`, kullanımdan kaldırılmış `systemAgent` ve `crestodian` yapılandırma bloklarını kaldırır.

Uzaktan kurtarma, Docker hattı kapsamındadır:

```bash
pnpm test:docker:system-agent-rescue
```

İsteğe bağlı canlı kanal komut yüzeyi duman testi, `/openclaw status` ile kurtarma işleyicisi üzerinden kalıcı bir onay gidiş dönüşünü denetler:

```bash
pnpm test:live:system-agent-rescue-channel
```

Çıkarım geçitli, paketlenmiş tek seferlik kurulum şu test kapsamındadır:

```bash
pnpm test:docker:system-agent-first-run
```

Bu paketlenmiş CLI hattı boş bir durum diziniyle başlar ve OpenClaw'ın çıkarım olmadan güvenli biçimde reddettiğini kanıtlar. Ardından paketlenmiş etkinleştirme modülü üzerinden sahte Claude'u test eder ve etkinleştirir. Ancak bundan sonra belirsiz bir istek planlayıcıya ulaşır ve türü belirlenmiş kuruluma çözümlenir; bunu ek bir ajan oluşturan, bir plugin etkinleştirmesi ve token SecretRef'i aracılığıyla Discord'u yapılandıran, yapılandırmayı doğrulayan ve denetim günlüğünü kontrol eden tek seferlik komutlar izler. Bu hat, geçit/işlem için destekleyici kanıttır; etkileşimli ilk katılımı veya OpenClaw ajanı/araç/onay konuşmasını yürütmez. Aşağıdaki QA Lab senaryosu aynı Docker hattına yönlendirir:

```bash
pnpm openclaw qa suite --scenario system-agent-ring-zero-setup
```

## İlgili

- [CLI referansı](/tr/cli)
- [Doctor](/tr/cli/doctor)
- [TUI](/tr/cli/tui)
- [Korumalı alan](/tr/cli/sandbox)
- [Güvenlik](/tr/cli/security)
