---
read_when:
    - Aracı tarafından denetlenen tarayıcı otomasyonu ekleme
    - openclaw'ın kendi Chrome'unuza neden müdahale ettiğini hata ayıklama
    - macOS uygulamasında tarayıcı ayarlarını ve yaşam döngüsünü uygulama
summary: Entegre tarayıcı kontrol hizmeti + eylem komutları
title: Tarayıcı (OpenClaw tarafından yönetilen)
x-i18n:
    generated_at: "2026-07-26T23:42:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3afa2dda17520ae6c53fe3f1a7a12e7ca8a1414b2c12b79cf4a09ac8906bb3ca
    source_path: tools/browser.md
    workflow: 16
---

OpenClaw, aracının denetlediği **özel bir Chrome/Brave/Edge/Chromium profili** çalıştırabilir. Bu profil, Gateway içindeki küçük bir yerel denetim hizmeti (yalnızca geri döngü) üzerinden çalışır ve kişisel tarayıcınızdan yalıtılmıştır.

- Bunu **ayrı, yalnızca aracıya özel bir tarayıcı** olarak düşünün. `openclaw` profili kişisel tarayıcı profilinize asla dokunmaz.
- Aracı bu yalıtılmış kanalda sekmeleri açar, sayfaları okur, tıklar ve yazı yazar.
- Yerleşik `user` profili ise Chrome DevTools MCP aracılığıyla gerçek, oturum açılmış Chrome oturumunuza bağlanır.

## Elde ettikleriniz

- **openclaw** adlı ayrı bir tarayıcı profili (varsayılan olarak turuncu vurgu rengi).
- Belirlenimci sekme denetimi (listeleme/açma/odaklama/kapatma).
- Aracı eylemleri (tıklama/yazma/sürükleme/seçme), anlık görüntüler, ekran görüntüleri, PDF'ler.
- Playwright destekli profiller, doğrudan ek gezinmelerini yönetilen indirmeler dizinine kaydeder ve son URL ilkesi doğrulamasından sonra `{ url, suggestedFilename, path }` meta verilerini döndürür.
- Playwright destekli aracı eylemleri, eylem bir veya daha fazla indirmeyi hemen başlattığında aynı yönetilen meta verileri içeren bir `downloads` dizisi döndürür.
- Tarayıcı plugini etkinleştirildiğinde aracılara anlık görüntü,
  kararlı sekme, geçersiz başvuru ve manuel engelleyiciden kurtarma döngüsünü öğreten paketlenmiş
  bir `browser-automation` skill'i.
- İsteğe bağlı çoklu profil desteği (`openclaw`, `work`, `remote`, ...).

Bu tarayıcı günlük kullandığınız tarayıcı **değildir**. Aracı otomasyonu ve
doğrulaması için güvenli, yalıtılmış bir yüzeydir.

macOS'te, çerezleri Chrome ailesinden bir sistem profilinden ayrı bir yönetilen profile açıkça kopyalayabilirsiniz. Yönetilen tarayıcı yine kendi kullanıcı verileri dizinini kullanır; yalnızca seçilen çerezler kopyalanır, yerel depolama ve IndexedDB geride kalır. İçe aktarma komutları ve sınırlamalar için [Profiller](#profiles-multi-browser) veya [`openclaw browser` CLI başvurusu](/tr/cli/browser) bölümüne bakın.

## Hızlı başlangıç

```bash
openclaw browser --browser-profile openclaw doctor
openclaw browser --browser-profile openclaw doctor --deep
openclaw browser --browser-profile openclaw status
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

"Tarayıcı devre dışı", pluginin veya `browser.enabled` öğesinin kapalı olduğu anlamına gelir;
[Yapılandırma](#configuration) ve [Plugin denetimi](#plugin-control) bölümlerine bakın.

`openclaw browser` tamamen eksikse veya aracı tarayıcı aracının
kullanılamadığını söylüyorsa [Eksik tarayıcı komutu veya aracı](#missing-browser-command-or-tool) bölümüne geçin.

## Plugin denetimi

Varsayılan `browser` aracı paketlenmiş bir plugindir. Aynı `browser` araç adını kaydeden başka bir pluginle değiştirmek için bunu devre dışı bırakın:

```json5
{
  plugins: {
    entries: {
      browser: {
        enabled: false,
      },
    },
  },
}
```

Varsayılanlar hem `plugins.entries.browser.enabled` **hem de** `browser.enabled=true` gerektirir. Yalnızca plugini devre dışı bırakmak; `openclaw browser` CLI'ını, `browser.request` gateway yöntemini, aracı aracını ve denetim hizmetini tek bir birim olarak kaldırır; `browser.*` yapılandırmanız, yerine geçecek bir plugin için olduğu gibi kalır.

Tarayıcı yapılandırması değişiklikleri, pluginin hizmetini yeniden kaydedebilmesi için Gateway'in yeniden başlatılmasını gerektirir.

## Aracı yönergeleri

Araç profili notu: `tools.profile: "coding"`, `web_search` ve
`web_fetch` öğelerini içerir ancak tam `browser` aracını içermez. Aracının veya
oluşturulan bir alt aracının tarayıcı otomasyonunu kullanmasına izin vermek için profil
aşamasında browser öğesini ekleyin:

```json5
{
  tools: {
    profile: "coding",
    alsoAllow: ["browser"],
  },
}
```

Tek bir aracı için `agents.entries.*.tools.alsoAllow: ["browser"]` kullanın.
Alt aracı ilkesi profil filtrelemesinden sonra uygulandığından yalnızca
`tools.subagents.tools.allow: ["browser"]` yeterli değildir.

Tarayıcı plugini iki düzeyde aracı yönergesi sunar:

- `browser` araç açıklaması, her zaman etkin olan kısa sözleşmeyi taşır: doğru
  profili seçin, başvuruları aynı sekmede tutun, sekme hedefleme için `tabId`/etiketleri
  kullanın ve çok adımlı işler için tarayıcı skill'ini yükleyin.
- Paketlenmiş `browser-automation` skill'i daha uzun çalışma döngüsünü taşır:
  önce durumu/sekmeleri kontrol edin, görev sekmelerini etiketleyin, işlemden önce anlık görüntü alın, kullanıcı
  arayüzü değişikliklerinden sonra yeniden anlık görüntü alın, geçersiz başvuruları bir kez kurtarın ve
  oturum açma/2FA/captcha veya kamera/mikrofon engelleyicilerini tahmin yürütmek yerine manuel eylem
  olarak bildirin.

Pluginle paketlenmiş skill'ler, plugin etkinleştirildiğinde aracının kullanılabilir skill'leri
arasında listelenir. Tam skill yönergeleri isteğe bağlı olarak yüklenir; böylece rutin
işlemler tam token maliyetine katlanmaz.

## Eksik tarayıcı komutu veya aracı

Yükseltmeden sonra `openclaw browser` bilinmiyorsa, `browser.request` eksikse veya aracı tarayıcı aracının kullanılamadığını bildiriyorsa genel neden, `browser` öğesini içermeyen bir `plugins.allow` listesi ve kök düzeyinde `browser` yapılandırma bloğunun bulunmamasıdır. Şunu ekleyin:

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

Açık bir kök `browser` bloğu (`browser` altındaki herhangi bir anahtar; örneğin
`browser.enabled=true` veya `browser.profiles.<name>`), kısıtlayıcı bir `plugins.allow` altında bile paketlenmiş
tarayıcı pluginini etkinleştirir; bu, paketlenmiş kanal yapılandırması davranışıyla
eşleşir. `plugins.entries.browser.enabled=true` ve
`tools.alsoAllow: ["browser"]`, kendi başlarına izin listesi üyeliğinin
yerini tutmaz. `plugins.allow` öğesinin tamamen kaldırılması da varsayılanı geri getirir.

## Profiller: `openclaw`, `user`, `chrome`

- `openclaw`: yönetilen, yalıtılmış tarayıcı (uzantı gerekmez).
- `user`: **gerçek, oturum açılmış Chrome** oturumunuz için yerleşik Chrome DevTools MCP bağlantı profili. OpenClaw ilk kez bağlandığında Chrome, engelleyici bir "Allow remote debugging?"
  istemi gösterir; bu nedenle bilgisayarın başında birinin bulunması gerekir.
- `chrome`: **gerçek, oturum açılmış Chrome** oturumunuz için yerleşik [Chrome uzantısı](/tr/tools/chrome-extension) profili.
  Sekmeleri uzaktan hata ayıklama bağlantı noktası yerine OpenClaw tarayıcı uzantısı üzerinden
  yönettiği için "Allow remote debugging?" istemi gösterilmez ve masada kimse olmasa bile
  telefondan çalışır.

Aracı tarayıcı aracı çağrıları için:

- Varsayılan: yalıtılmış `openclaw` tarayıcısını kullanın.
- Mevcut oturum açılmış oturumlar önemliyse ve kullanıcı **bilgisayardan uzaktaysa**
  (Telegram, WhatsApp vb.) `profile="chrome"` (uzantı) tercih edin.
- Mevcut oturum açılmış oturumlar önemliyse ve kullanıcı bağlantı istemini onaylamak üzere
  **bilgisayarın başındaysa** `profile="user"` (Chrome MCP) tercih edin.
- Belirli bir tarayıcı modu istediğinizde `profile` açık geçersiz kılmadır.

Yönetilen modun varsayılan olmasını istiyorsanız `browser.defaultProfile: "openclaw"` ayarlayın.

## Yapılandırma

Tarayıcı ayarları `~/.openclaw/openclaw.json` içinde bulunur.

```json5
{
  browser: {
    enabled: true, // varsayılan: true
    evaluateEnabled: true, // varsayılan: true; false, act:evaluate işlevini (rastgele JS) devre dışı bırakır
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // yalnızca güvenilir özel ağ erişimi için kabul edin
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    // cdpUrl: "http://127.0.0.1:18792", // eski tek profil geçersiz kılması
    tabCleanup: {
      enabled: true, // varsayılan: true
    },
    // snapshotDefaults: { mode: "efficient" }, // çağıran belirtmediğinde varsayılan anlık görüntü modu
    defaultProfile: "openclaw",
    color: "#FF4500",
    headless: false,
    noSandbox: false,
    attachOnly: false,
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        headless: true,
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: {
        driver: "existing-session",
        attachOnly: true,
        color: "#00AA00",
      },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
  },
}
```

`browser.snapshotDefaults.mode: "efficient"`, çağıran açık bir `snapshotFormat` veya
`mode` iletmediğinde varsayılan `snapshot`
çıkarma modunu değiştirir; çağrı başına anlık görüntü seçenekleri için [Tarayıcı denetim API'sine](/tr/tools/browser-control)
bakın.

### Sekme temizleme sahipliği

Oturum sekmesi temizliği yalnızca OpenClaw tarayıcı aracı tarafından
`action: "open"` ile oluşturulan sekmelere uygulanır. OpenClaw önceden açık olan,
kullanıcı tarafından açılan veya sahipliği başka bir nedenle bilinmeyen sekmeleri sahiplenmez.
`browser.tabCleanup` bloğu, birincil oturumlar için düzenli boşta kalma ve sınır taramalarını
denetler; bunu devre dışı bırakmak, açık oturum yaşam döngüsü temizliğini devre dışı bırakmaz.

Ana makineye yerel açılışlarda, kararlı bir yerel CDP hedefi ve tarayıcı
kimliğiyle sahiplik, paylaşılan SQLite durumunda saklanır. Bu kayıtlar Gateway'in yeniden
başlatılmasından sonra da kalır ve `/new` ile diğer oturum yaşam döngüsü temizliği için uygun olmaya devam eder;
oturum yaşam döngüsü temizliği, alt aracı, cron ve ACP oturumlarının sonlandırılmasını içerir.
Araca gösterilen hedefi yerel CDP hedefi olan kayıtlar da yeniden başlatmadan sonra
boşta kalma ve oturum başına sınır taramaları için uygun olmaya devam eder. Chrome MCP hedef tanıtıcıları
işlem yereldir; bu nedenle soğuk mevcut oturum kayıtları, yeniden başlatmadan sonra güvenle
ilişkilendirilemeyen etkinliğe karşı boşta kalma taraması riski almak yerine yaşam döngüsü temizliğini
bekler. Bu kalıcı yol; OpenClaw tarafından yönetilen profilleri, normal uzak CDP profillerini
ve OpenClaw'ın hem yerel hedefi hem de kararlı bir tarayıcı kimliğini çözümleyebilmesi koşuluyla
açık bir `cdpUrl` içeren mevcut oturum profillerini kapsayabilir. OpenClaw, kalıcı bir kaydı
kapatmadan önce yapılandırılmış profil ile tarayıcı örneğinin hâlâ eşleştiğini doğrular.

Chrome MCP `--autoConnect`, `/json/version` yanıtında kararlı bir tarayıcı kimliği bulunmayan
CDP uç noktaları ve yerel hedefi çözümlenemeyen açılışlar,
işlem yerel en iyi çaba takibi olarak kalır. İlgili Gateway işlemi çalışırken
temizlenebilirler ancak Gateway yeniden başlatıldıktan sonra otomatik olarak
kapatılmazlar. Kalıcı izleme kullanıma sunulmadan önce açık bırakılan sekmeler
geriye dönük olarak sahiplenilmez; bu sekmeleri manuel olarak kapatın.

Temizleme en iyi çabayla yapılır; uygun her sekmenin hemen kapanacağı
garanti edilmez. Geçici bir sahiplik denetimi veya kapatma hatası, kalıcı
temizliği daha sonraki bir yeniden deneme için beklemede bırakır. Yeniden denemeler sınırsız değildir:
tarayıcıya erişilemez durumda kalırsa ve sekme bir günden uzun süredir kullanılmamışsa, bir daha
doğrulanamayacak sekmelerin kalıcı depoyu doldurmaması için izleme satırı
kaldırılır.

### Ekran görüntüsü görsel algısı (yalnızca metin model desteği)

Ana model yalnızca metinse (görsel/çok modlu destek yoksa), tarayıcı
ekran görüntüleri modelin okuyamayacağı görüntü blokları döndürür. Tarayıcı ekran görüntüleri,
mevcut görüntü anlama yapılandırmasını yeniden kullanır; dolayısıyla medya anlama için
yapılandırılmış bir görüntü modeli, tarayıcıya özgü model ayarları olmadan ekran görüntülerini
metin olarak açıklayabilir.

```json5
{
  tools: {
    media: {
      image: {
        models: [
          { provider: "bytedance", model: "doubao-seed-2.0-pro" },
          // Yedek adaylar ekleyin; ilk başarılı olan kazanır
          { provider: "openai", model: "gpt-4o" },
        ],
      },
      // Paylaşılan medya modelleri, görüntü desteği için etiketlendiklerinde de çalışır.
      // models: [{ provider: "openai", model: "gpt-4o", capabilities: ["image"] }],
    },
  },
  agents: {
    defaults: {
      // Mevcut görüntü modeli varsayılanları da dikkate alınır.
      // imageModel: { primary: "openai/gpt-4o" },
    },
  },
}
```

**Nasıl çalışır:**

1. Agent, `browser screenshot` çağrısını yapar ve her zamanki gibi diske bir görüntü kaydedilir.
2. Tarayıcı aracı, mevcut görüntü anlama çalışma zamanına; yapılandırılmış medya görüntü modellerini, paylaşılan medya
   modellerini, görüntü modeli varsayılanlarını veya kimlik doğrulama destekli bir görüntü sağlayıcısını kullanarak ekran görüntüsünü
   açıklayıp açıklayamayacağını sorar.
3. Görü modeli, `wrapExternalContent` (istem enjeksiyonu koruması) ile sarmalanıp
   görüntü bloğu yerine metin bloğu olarak agente döndürülen bir metin açıklaması
   sağlar.
4. Görüntü anlama kullanılamıyorsa, atlanırsa veya başarısız olursa tarayıcı,
   özgün görüntü bloğunu döndürmeye geri döner.

Ekran görüntüsü blokları özel araç sonuçlarıdır: agent bunları inceleyebilir,
ancak OpenClaw bunları kanal yanıtlarına otomatik olarak eklemez. Bir ekran
görüntüsünü paylaşmak için agentten bunu mesaj aracıyla açıkça göndermesini isteyin.

Model geri dönüşleri, zaman aşımları, bayt sınırları, profiller ve sağlayıcı
istek ayarları için mevcut `tools.media.image` / `tools.media.models` alanlarını kullanın.

Etkin ana model zaten görüntü işlemeyi destekliyorsa ve açıkça bir görüntü
anlama modeli yapılandırılmamışsa OpenClaw, ana modelin ekran görüntüsünü doğrudan
okuyabilmesi için normal görüntü sonucunu korur.

<AccordionGroup>

<Accordion title="Bağlantı noktaları ve erişilebilirlik">

- Denetim hizmeti, `gateway.port` değerinden türetilen bir bağlantı noktasında geri döngüye bağlanır (varsayılan `18791` = gateway + 2). `OPENCLAW_GATEWAY_PORT`, `gateway.port` değerinden önceliklidir; ikisi de türetilen bağlantı noktalarını aynı aile içinde kaydırır.
- Yerel `openclaw` profilleri, `cdpPort`/`cdpUrl` değerlerini denetim bağlantı noktasının 9 bağlantı noktası sonrasından başlayan bir aralıktan otomatik olarak atar (varsayılan `18800`-`18899`); bunları yalnızca
  uzak CDP profilleri veya mevcut oturum uç noktası bağlantısı için ayarlayın. `cdpUrl`, ayarlanmadığında
  yönetilen yerel CDP bağlantı noktasını varsayılan olarak kullanır.
- Uzak ve `attachOnly` CDP erişilebilirliği, WebSocket el sıkışmaları ve yerel
  yönetilen Chrome başlatma işlemi yerleşik süre sınırlarını kullanır.
- Yönetilen Chrome'un yinelenen başlatma/hazır olma hataları profil başına devre kesiciyle
  sınırlandırılır. Art arda birkaç hatadan sonra OpenClaw, her tarayıcı aracı çağrısında
  Chromium oluşturmak yerine yeni başlatma girişimlerini kısa süreliğine duraklatır. Başlatma
  sorununu düzeltin, gerekmiyorsa tarayıcıyı devre dışı bırakın veya onarımdan sonra
  Gateway'i yeniden başlatın.

</Accordion>

<Accordion title="SSRF politikası">

- Tarayıcı gezinme ve sekme açma istekleri ön kontrol işleminden geçirilir. Eylem sırasında ve sınırlı eylem sonrası ek süre boyunca korumalı Playwright etkileşimleri (tıklama, koordinata tıklama, üzerine gelme, sürükleme, kaydırma, seçme, tuşa basma, yazma, form doldurma ve değerlendirme), HTTP istek baytlarından önce politika tarafından reddedilen üst düzey ve alt çerçeve belge yüklemelerini engeller, ardından nihai `http(s)` URL'sini azami gayretle yeniden denetler.
- Her yeni OpenClaw yönetimli Chrome başlatmasından önce OpenClaw, ağ tahminini azami gayretle devre dışı bırakarak Chromium'da gözlemlenen bu reddedilmiş yüklemelere yönelik spekülatif ön bağlantıyı engeller. Bu, derinlemesine savunmadır; bir politika sınırı değildir: denetim hizmetinin yeniden başlatılması boyunca yeniden kullanılan bir tarayıcı ve diğer tarayıcı arka uçları bu sağlamlaştırmayı paylaşmayabilir. Playwright yönlendirmesi yine de bir ağ güvenlik duvarı değildir ve yönlendirme atlamalarını, açılır pencerenin ilk isteğini, Service Worker trafiğini, sınırlı koruma penceresinden sonra çalışan sayfa kodunu veya tüm arka plan/alt kaynak yollarını engellemez. Tam çıkış yalıtımı, sahip tarafında yalıtım veya politika uygulayan bir proxy gerektirir.
- Katı SSRF modunda uzak CDP uç noktası keşfi ve `/json/version` yoklamaları (`cdpUrl`) da denetlenir.
- Gateway/sağlayıcı `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY` ve `NO_PROXY` ortam değişkenleri, OpenClaw yönetimli tarayıcıyı otomatik olarak proxy üzerinden yönlendirmez. Sağlayıcı proxy ayarlarının tarayıcı SSRF denetimlerini zayıflatmaması için yönetilen Chrome varsayılan olarak doğrudan başlatılır.
- OpenClaw yönetimli yerel CDP hazır olma yoklamaları ve DevTools WebSocket bağlantıları, tam olarak başlatılan geri döngü uç noktası için yönetilen ağ proxy'sini atlar; böylece bir operatör proxy'si geri döngü çıkışını engellese bile `openclaw browser start` çalışmaya devam eder.
- Yönetilen tarayıcının kendisini proxy üzerinden yönlendirmek için `browser.extraArgs` aracılığıyla `--proxy-server=...` veya `--proxy-pac-url=...` gibi açık Chrome proxy bayrakları iletin. Katı SSRF modu, özel ağ tarayıcı erişimi kasıtlı olarak etkinleştirilmediği sürece açık tarayıcı proxy yönlendirmesini engeller.
- `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` varsayılan olarak kapalıdır; yalnızca özel ağ tarayıcı erişimine kasıtlı olarak güvenildiğinde etkinleştirin.
- `browser.ssrfPolicy.allowPrivateNetwork` eski bir diğer ad olarak desteklenmeye devam eder.

</Accordion>

<Accordion title="Profil davranışı">

- `attachOnly: true`, hiçbir zaman yerel bir tarayıcı başlatılmaması; yalnızca hâlihazırda çalışan bir tarayıcı varsa bağlanılması anlamına gelir.
- `headless` genel olarak veya yerel yönetilen profil başına ayarlanabilir. Profil başına değerler `browser.headless` değerini geçersiz kılar; böylece yerel olarak başlatılan bir profil başsız kalırken başka biri görünür kalabilir.
- `POST /start?headless=true` ve `openclaw browser start --headless`,
  `browser.headless` veya profil yapılandırmasını yeniden yazmadan yerel yönetilen profiller için
  tek seferlik başsız başlatma ister. Mevcut oturum, yalnızca bağlanma ve
  uzak CDP profilleri bu geçersiz kılmayı reddeder çünkü OpenClaw bu
  tarayıcı süreçlerini başlatmaz.
- `DISPLAY` veya `WAYLAND_DISPLAY` bulunmayan Linux ana makinelerinde, ne ortam ne de profil/genel
  yapılandırma açıkça pencereli modu seçtiğinde yerel yönetilen profiller
  otomatik olarak başsız modu varsayılan olarak kullanır. Açık ve net tarayıcı düzeyi biçimi
  `openclaw browser --json status` kullanın; sondaki `openclaw browser status --json` da
  çalışır çünkü `status` kendi `--json` seçeneğini tanımlamaz. Komut,
  `headlessSource` değerini `env`, `profile`, `config`,
  `request`, `linux-display-fallback` veya `default` olarak bildirir.
- `OPENCLAW_BROWSER_HEADLESS=1`, geçerli süreç için yerel yönetilen başlatmaları
  başsız olmaya zorlar. `OPENCLAW_BROWSER_HEADLESS=0`, sıradan başlatmalarda pencereli
  modu zorlar ve görüntü sunucusu bulunmayan Linux ana makinelerinde uygulanabilir bir hata döndürür;
  açık bir `start --headless` isteği, o tek başlatma için yine önceliklidir.
- Tarayıcı denetim rotası ve programatik istemci, görüntü bulunmaması hatasının
  insanlarca okunabilir `error` değerini korur ve kararlı neden
  `no_display_for_headed_profile` değerini sunar. Bunun `details` alanı yalnızca `profile`,
  `requestedHeadless`, `headlessSource` ve `displayPresent` içerir; böylece API istemcileri,
  ileti metniyle eşleştirme yapmadan doğru düzeltmeyi seçebilir.
- Çalışan bir yerel yönetilen profil için durum ve doctor, oluşturucu, arka uç, cihaz/sürücü, özellik
  durumu, sürücü geçici çözümleri ve hızlandırılmış video yetenekleri için Chrome'un
  tarayıcı düzeyindeki CDP uç noktasını sorgular. Sonuç, söz konusu tarayıcı süreci için
  önbelleğe alınır ve `openclaw browser --json status` tarafından eksiksiz biçimde sunulur.
  Pasif bir durum çağrısı Chrome'u başlatmaz.
  Mevcut oturum, uzantı, uzak CDP ve korumalı alan tarayıcıları ayrı kalır
  ve bu yönetilen ana makine yolu üzerinden incelenmez.
- Başsız yönetilen Chrome yine ölçülü `--disable-gpu` varsayılanını kullanır.
  Tanılama, hızlandırmayı etkinleştirmez, genel bir hızlandırma ayarı eklemez
  veya korumalı alan tarayıcısına cihaz erişimi vermez.
- `executablePath` genel olarak veya yerel yönetilen profil başına ayarlanabilir. Profil başına değerler `browser.executablePath` değerini geçersiz kılar; böylece farklı yönetilen profiller farklı Chromium tabanlı tarayıcıları başlatabilir. Her iki biçim de işletim sistemi ana dizininiz için `~` değerini kabul eder.
- `color` (üst düzeyde ve profil başına), hangi profilin etkin olduğunu görebilmeniz için tarayıcı arayüzünü renklendirir.
- Varsayılan profil `openclaw` şeklindedir (yönetilen bağımsız). Oturum açılmış kullanıcı tarayıcısını seçmek için `defaultProfile: "user"` kullanın.
- Otomatik algılama sırası: Chromium tabanlıysa sistemin varsayılan tarayıcısı; aksi takdirde Chrome, Brave, Edge, Chromium, Chrome Canary.
- `driver: "existing-session"`, ham CDP yerine Chrome DevTools MCP kullanır. Chrome MCP otomatik bağlantısı üzerinden veya çalışan tarayıcı için zaten bir DevTools uç noktanız varsa `cdpUrl` üzerinden bağlanabilir.
- `driver: "extension"`, [OpenClaw Chrome uzantısı](/tr/tools/chrome-extension) aracılığıyla oturum açılmış Chrome'unuzu yönetir. Aktarıcı kendi geri döngü uç noktasına sahip olduğundan bu profiller `cdpUrl` değerini kabul etmez. Bu, bilgisayarın başında kimse yokken çalışan tek oturum açılmış tarayıcı modudur.
- Mevcut oturum profilinin varsayılan olmayan bir Chromium kullanıcı profiline (Brave, Edge vb.) bağlanması gerektiğinde `browser.profiles.<name>.userDataDir` değerini ayarlayın. Bu yol, işletim sistemi ana dizininiz için `~` değerini de kabul eder.

</Accordion>

</AccordionGroup>

## Brave veya başka bir Chromium tabanlı tarayıcı kullanma

**Sisteminizin varsayılan** tarayıcısı Chromium tabanlıysa (Chrome/Brave/Edge/vb.),
OpenClaw bunu otomatik olarak kullanır. Otomatik algılamayı geçersiz kılmak için
`browser.executablePath` değerini ayarlayın. Üst düzey ve profil başına `executablePath`
değerleri, işletim sistemi ana dizininiz için `~` değerini kabul eder:

```bash
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set browser.profiles.work.executablePath "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
```

Alternatif olarak platforma göre yapılandırmada ayarlayın:

<Tabs>
  <Tab title="macOS">
```json5
{
  browser: {
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
  },
}
```
  </Tab>
  <Tab title="Windows">
```json5
{
  browser: {
    executablePath: "C:\\Program Files\\BraveSoftware\\Brave-Browser\\Application\\brave.exe",
  },
}
```
  </Tab>
  <Tab title="Linux">
```json5
{
  browser: {
    executablePath: "/usr/bin/brave-browser",
  },
}
```
  </Tab>
</Tabs>

Profil başına `executablePath`, yalnızca OpenClaw'un başlattığı yerel yönetilen
profilleri etkiler. `existing-session` profilleri bunun yerine hâlihazırda çalışan
bir tarayıcıya bağlanır ve uzak CDP profilleri `cdpUrl` arkasındaki tarayıcıyı kullanır.

## Yerel ve uzak denetim

- **Yerel denetim (varsayılan):** Gateway, geri döngü denetim hizmetini başlatır ve yerel bir tarayıcı başlatabilir.
- **Uzak denetim (Node ana makinesi):** tarayıcının bulunduğu makinede bir Node ana makinesi çalıştırın; Gateway, tarayıcı eylemlerini buna proxy üzerinden iletir.
- **Uzak CDP:** uzak bir Chromium tabanlı tarayıcıya bağlanmak için
  `browser.profiles.<name>.cdpUrl` (veya `browser.cdpUrl`) değerini ayarlayın. Bu durumda OpenClaw yerel bir tarayıcı başlatmaz.
- Geri döngüde haricen yönetilen CDP hizmetleri için (örneğin Docker'da
  `127.0.0.1` adresinde yayımlanan Browserless) ayrıca `attachOnly: true` değerini ayarlayın. `attachOnly` olmadan
  geri döngü CDP'si, yerel OpenClaw yönetimli tarayıcı profili olarak değerlendirilir.
- `headless` yalnızca OpenClaw'un başlattığı yerel yönetilen profilleri etkiler. Mevcut oturum veya uzak CDP tarayıcılarını yeniden başlatmaz ya da değiştirmez.
- `executablePath` aynı yerel yönetilen profil kuralını izler. Çalışan bir
  yerel yönetilen profilde bunun değiştirilmesi, bir sonraki başlatmanın yeni ikili dosyayı
  kullanması için profili yeniden başlatma/uzlaştırma işlemiyle işaretler.

Durdurma davranışı profil moduna göre farklılık gösterir:

- yerel yönetilen profiller: `openclaw browser stop`, OpenClaw'un
  başlattığı tarayıcı sürecini durdurur
- yalnızca bağlanma ve uzak CDP profilleri: OpenClaw tarafından
  herhangi bir tarayıcı süreci başlatılmamış olsa bile `openclaw browser stop`, etkin
  denetim oturumunu kapatır ve Playwright/CDP öykünme geçersiz kılmalarını
  (görüntü alanı, renk düzeni, yerel ayar, saat dilimi, çevrimdışı mod ve
  benzer durumlar) serbest bırakır

Uzak CDP URL'leri kimlik doğrulaması içerebilir:

- Sorgu belirteçleri (ör. `https://provider.example?token=<token>`)
- HTTP Temel kimlik doğrulaması (ör. `https://user:pass@provider.example`)

OpenClaw, `/json/*` uç noktalarını çağırırken ve CDP WebSocket'e bağlanırken
kimlik doğrulamasını korur. Tokenleri yapılandırma dosyalarına kaydetmek yerine
ortam değişkenlerini veya gizli bilgi yöneticilerini tercih edin.

## Node tarayıcı proxy'si (sıfır yapılandırmalı varsayılan)

Tarayıcınızın bulunduğu makinede bir **node ana makinesi** çalıştırırsanız OpenClaw,
ek tarayıcı yapılandırması gerektirmeden tarayıcı aracı çağrılarını bu node'a
otomatik olarak yönlendirebilir. Uzak gateway'ler için varsayılan yol budur.

Notlar:

- Node ana makinesi, yerel tarayıcı denetim sunucusunu bir **proxy komutu** aracılığıyla kullanıma sunar.
- Profiller, node'un kendi `browser.profiles` yapılandırmasından gelir (yerel yapılandırmayla aynıdır).
- Proxy komutu, `allowProfiles` değerinden bağımsız olarak kalıcı profil değişikliklerine (`create-profile`, `delete-profile`, `reset-profile`) hiçbir zaman izin vermez; bu değişiklikleri doğrudan node üzerinde yapın.
- `nodeHost.browserProxy.allowProfiles` isteğe bağlıdır. Eski/varsayılan davranış için boş bırakın: yapılandırılmış tüm profillere proxy üzerinden erişilebilir.
- `nodeHost.browserProxy.allowProfiles` ayarlanırsa OpenClaw, bunu proxy'nin hedefleyebileceği profil adlarını sınırlayan en az ayrıcalık sınırı olarak değerlendirir.
- Bunu istemiyorsanız devre dışı bırakın:
  - Node üzerinde: `nodeHost.browserProxy.enabled=false`
  - Gateway üzerinde: `gateway.nodes.browser.mode="off"` (bağlı tek bir tarayıcı node'u seçmek için `"auto"` veya açık bir node parametresi gerektirmek için `"manual"` değerini de kabul eder)

## Browserless (barındırılan uzak CDP)

[Browserless](https://browserless.io), CDP bağlantı URL'lerini HTTPS ve WebSocket
üzerinden kullanıma sunan, barındırılan bir Chromium hizmetidir. OpenClaw her iki
biçimi de kullanabilir ancak uzak bir tarayıcı profili için en basit seçenek,
Browserless bağlantı belgelerindeki doğrudan WebSocket URL'sidir.

Örnek:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    profiles: {
      browserless: {
        cdpUrl: "wss://production-sfo.browserless.io?token=<BROWSERLESS_API_KEY>",
        color: "#00AA00",
      },
    },
  },
}
```

Notlar:

- `<BROWSERLESS_API_KEY>` değerini gerçek Browserless tokeninizle değiştirin.
- Browserless hesabınızla eşleşen bölge uç noktasını seçin (belgelerine bakın).
- Browserless size bir HTTPS temel URL'si verirse bunu doğrudan CDP bağlantısı için
  `wss://` biçimine dönüştürebilir veya HTTPS URL'sini koruyup OpenClaw'un
  `/json/version` değerini keşfetmesini sağlayabilirsiniz.

### Aynı ana makinede Browserless Docker

Browserless, Docker'da kendi kendine barındırılıyor ve OpenClaw ana makinede
çalışıyorsa Browserless'ı haricen yönetilen bir CDP hizmeti olarak değerlendirin:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    profiles: {
      browserless: {
        cdpUrl: "ws://127.0.0.1:3000",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

`browser.profiles.browserless.cdpUrl` içindeki adrese OpenClaw işlemi tarafından erişilebilmelidir.
Browserless ayrıca eşleşen ve erişilebilir bir uç nokta duyurmalıdır; Browserless
`EXTERNAL` değerini `ws://127.0.0.1:3000`, `ws://browserless:3000` veya kararlı bir özel Docker
ağ adresi gibi, dışarıdan OpenClaw'a erişilebilen aynı WebSocket tabanına ayarlayın.
`/json/version`, OpenClaw'un erişemediği bir adresi gösteren `webSocketDebuggerUrl`
değerini döndürürse CDP HTTP sağlıklı görünebilir ancak WebSocket bağlantısı
yine de başarısız olur.

Geri döngü Browserless profili için `attachOnly` değerini ayarlanmamış
bırakmayın. `attachOnly` olmadan OpenClaw, geri döngü portunu yerel olarak
yönetilen bir tarayıcı profili sayar ve portun kullanımda olduğunu ancak
OpenClaw'a ait olmadığını bildirebilir.

## Doğrudan WebSocket CDP sağlayıcıları

Bazı barındırılan tarayıcı hizmetleri, standart HTTP tabanlı CDP keşfi
(`/json/version`) yerine **doğrudan WebSocket** uç noktası sunar. OpenClaw üç
CDP URL biçimini kabul eder ve doğru bağlantı stratejisini otomatik olarak seçer:

- **HTTP(S) keşfi** - `http://host[:port]` veya `https://host[:port]`.
  OpenClaw, WebSocket hata ayıklayıcı URL'sini keşfetmek için `/json/version` çağrısı
  yapar ve ardından bağlanır. WebSocket geri dönüşü yoktur.
- **Doğrudan WebSocket uç noktaları** - `/devtools/browser|page|worker|shared_worker|service_worker/<id>`
  yoluna sahip `ws://host[:port]/devtools/<kind>/<id>` veya `wss://...`.
  OpenClaw doğrudan WebSocket el sıkışmasıyla bağlanır ve
  `/json/version` adımını tamamen atlar.
- **Çıplak WebSocket kökleri** - `/devtools/...` yolu olmayan
  `ws://host[:port]` veya `wss://host[:port]` (ör. [Browserless](https://browserless.io),
  [Browserbase](https://www.browserbase.com)). OpenClaw önce HTTP
  `/json/version` keşfini dener (şemayı `http`/`https` olarak normalleştirir);
  keşif bir `webSocketDebuggerUrl` döndürürse bu kullanılır, aksi takdirde OpenClaw
  çıplak kökte doğrudan WebSocket el sıkışmasına geri döner. Duyurulan WebSocket
  uç noktası CDP el sıkışmasını reddeder ancak yapılandırılmış çıplak kök
  kabul ederse OpenClaw yine bu köke geri döner. Bu sayede yerel bir Chrome'u
  gösteren çıplak `ws://` bağlanmaya devam edebilir; çünkü Chrome,
  WebSocket yükseltmelerini yalnızca `/json/version` içindeki hedefe özel yolda
  kabul eder. Barındırılan sağlayıcılar ise keşif uç noktaları Playwright CDP
  için uygun olmayan kısa ömürlü bir URL duyurduğunda kök WebSocket uç
  noktalarını kullanmaya devam edebilir.

`openclaw browser doctor`, çalışma zamanı bağlantısıyla aynı önce keşif, ardından
WebSocket geri dönüşü mantığını kullanır; bu nedenle başarıyla bağlanan çıplak kök
URL'si, tanılama tarafından erişilemez olarak bildirilmez.

### Browserbase

[Browserbase](https://www.browserbase.com), yerleşik CAPTCHA çözme, gizlilik modu
ve konut proxy'leriyle başsız tarayıcı çalıştırmaya yönelik bir bulut platformudur.

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserbase",
    profiles: {
      browserbase: {
        cdpUrl: "wss://connect.browserbase.com?apiKey=<BROWSERBASE_API_KEY>",
        color: "#F97316",
      },
    },
  },
}
```

Notlar:

- [Kaydolun](https://www.browserbase.com/sign-up) ve **API Key** değerini
  [Overview dashboard](https://www.browserbase.com/overview) üzerinden kopyalayın.
- `<BROWSERBASE_API_KEY>` değerini gerçek Browserbase API anahtarınızla değiştirin.
- Browserbase, WebSocket bağlantısında otomatik olarak bir tarayıcı oturumu
  oluşturur; bu nedenle elle oturum oluşturma adımı gerekmez.
- Güncel ücretsiz katman sınırları ve ücretli planlar için [fiyatlandırmaya](https://www.browserbase.com/pricing) bakın.
- Eksiksiz API başvurusu, SDK kılavuzları ve entegrasyon örnekleri için
  [Browserbase belgelerine](https://docs.browserbase.com) bakın.

### Notte

[Notte](https://www.notte.cc), yerleşik gizlilik, konut proxy'leri ve CDP'ye özgü
WebSocket gateway'iyle başsız tarayıcı çalıştırmaya yönelik bir bulut platformudur.

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "notte",
    profiles: {
      notte: {
        cdpUrl: "wss://us-prod.notte.cc/sessions/connect?token=<NOTTE_API_KEY>",
        color: "#7C3AED",
      },
    },
  },
}
```

Notlar:

- [Kaydolun](https://console.notte.cc) ve **API Key** değerini
  konsol ayarları sayfasından kopyalayın.
- `<NOTTE_API_KEY>` değerini gerçek Notte API anahtarınızla değiştirin.
- Notte, WebSocket bağlantısında otomatik olarak bir tarayıcı oturumu
  oluşturur; bu nedenle elle oturum oluşturma adımı gerekmez. WebSocket bağlantısı
  kesildiğinde oturum yok edilir.
- Güncel ücretsiz katman sınırları ve ücretli planlar için [fiyatlandırmaya](https://www.notte.cc/#pricing) bakın.
- Eksiksiz API başvurusu, SDK kılavuzları ve entegrasyon örnekleri için
  [Notte belgelerine](https://docs.notte.cc) bakın.

## Güvenlik

Temel fikirler:

- Tarayıcı denetimi yalnızca geri döngü üzerinden kullanılabilir; erişim Gateway'in kimlik doğrulaması veya node eşleştirmesi üzerinden gerçekleşir.
- Bağımsız geri döngü tarayıcı HTTP API'si **yalnızca paylaşılan gizli bilgiyle kimlik doğrulaması** kullanır:
  gateway token taşıyıcı kimlik doğrulaması, `x-openclaw-password` veya yapılandırılmış
  gateway parolasıyla HTTP Temel kimlik doğrulaması.
- Tailscale Serve kimlik üstbilgileri ve `gateway.auth.mode: "trusted-proxy"`,
  bu bağımsız geri döngü tarayıcı API'sinde kimlik doğrulaması **yapmaz**.
- Tarayıcı denetimi etkinse ve paylaşılan gizli bilgiyle kimlik doğrulaması
  yapılandırılmamışsa OpenClaw, başlangıçta bir tarayıcı denetimi kimlik bilgisi
  oluşturup kalıcı olarak saklar: `gateway.auth.mode` değeri `none` olduğunda
  bir token, `trusted-proxy` olduğunda ise bir parola oluşturur (işlem dışı geri
  döngü istemcilerinin çözümleyebilmesi için `gateway.auth.password` aracılığıyla kalıcı
  olarak saklanır). İlgili mod için açık bir dize kimlik bilgisi zaten yapılandırılmışsa
  veya `gateway.auth.mode` değeri `password` ise otomatik oluşturma atlanır.
- Oluşturulan gizli bilgi yerine denetiminizdeki kararlı bir gizli bilgi
  istiyorsanız `gateway.auth.token`, `gateway.auth.password`, `OPENCLAW_GATEWAY_TOKEN` veya
  `OPENCLAW_GATEWAY_PASSWORD` değerini açıkça yapılandırın.

Uzak CDP ipuçları:

- Mümkün olduğunda şifrelenmiş uç noktaları (HTTPS veya WSS) ve kısa ömürlü tokenleri tercih edin.
- Uzun ömürlü tokenleri doğrudan yapılandırma dosyalarına yerleştirmekten kaçının.
- Gateway'i ve tüm node ana makinelerini özel bir ağda (Tailscale) tutun; herkese açık biçimde kullanıma sunmaktan kaçının.
- Uzak CDP URL'lerini/tokenlerini gizli bilgi olarak değerlendirin; ortam değişkenlerini veya bir gizli bilgi yöneticisini tercih edin.

## Profiller (çoklu tarayıcı)

OpenClaw birden fazla adlandırılmış profili (yönlendirme yapılandırmalarını)
destekler. Profiller şunlar olabilir:

- **OpenClaw tarafından yönetilen**: kendi kullanıcı veri dizinine ve CDP portuna sahip, Chromium tabanlı özel bir tarayıcı örneği
- **uzak**: açık bir CDP URL'si (başka bir yerde çalışan Chromium tabanlı tarayıcı)
- **mevcut oturum**: Chrome DevTools MCP otomatik bağlantısı üzerinden mevcut Chrome profiliniz

Varsayılanlar:

- `openclaw` profili yoksa otomatik olarak oluşturulur.
- `user` profili, Chrome MCP mevcut oturum bağlantısı için yerleşiktir.
- Mevcut oturum profilleri, `user` dışında isteğe bağlıdır; bunları `--driver existing-session` ile oluşturun.
- Yerel CDP portları varsayılan olarak **18800-18899** aralığından ayrılır.
- Bir profil silindiğinde yerel veri dizini Çöp Kutusu'na taşınır.

Tüm denetim uç noktaları `?profile=<name>` değerini kabul eder; CLI,
`--browser-profile` kullanır.

## Chrome DevTools MCP üzerinden mevcut oturum

OpenClaw, resmi Chrome DevTools MCP sunucusu aracılığıyla çalışan Chromium tabanlı
bir tarayıcı profiline de bağlanabilir. Bu yöntem, söz konusu tarayıcı profilinde
zaten açık olan sekmeleri ve oturum açma durumunu yeniden kullanır.

Resmî arka plan bilgileri ve kurulum başvuruları:

- [Chrome for Developers: Chrome DevTools MCP'yi tarayıcı oturumunuzla kullanma](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- [Chrome DevTools MCP README](https://github.com/ChromeDevTools/chrome-devtools-mcp)

Yerleşik profil: `user`. Farklı bir ad, renk veya tarayıcı veri dizini
istiyorsanız kendi özel mevcut oturum profilinizi oluşturun.

Varsayılan olarak yerleşik `user` profili, varsayılan yerel Google Chrome
profilini hedefleyen Chrome MCP otomatik bağlantısını kullanır. Brave, Edge, Chromium
veya varsayılan olmayan bir Chrome profili için `userDataDir` kullanın.
`~`, işletim sisteminizin ana dizinine genişletilir:

```json5
{
  browser: {
    profiles: {
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
    },
  },
}
```

Ardından eşleşen tarayıcıda:

1. Uzak hata ayıklama için bu tarayıcının inceleme sayfasını açın.
2. Uzak hata ayıklamayı etkinleştirin.
3. Tarayıcıyı çalışır durumda tutun ve OpenClaw bağlandığında bağlantı istemini onaylayın.

Yaygın inceleme sayfaları:

- Chrome: `chrome://inspect/#remote-debugging`
- Brave: `brave://inspect/#remote-debugging`
- Edge: `edge://inspect/#remote-debugging`

Canlı bağlantı duman testi:

```bash
openclaw browser --browser-profile user start
openclaw browser --browser-profile user status
openclaw browser --browser-profile user tabs
openclaw browser --browser-profile user snapshot --format ai
```

Başarılı sonuç şöyle görünür:

- `status`, `driver: existing-session` gösterir
- `status`, `transport: chrome-mcp` gösterir
- `status`, `running: true` gösterir
- `tabs`, zaten açık olan tarayıcı sekmelerinizi listeler
- `snapshot`, seçilen etkin sekmeden referansları döndürür

Bağlanma çalışmıyorsa kontrol edilecekler:

- hedef Chromium tabanlı tarayıcının sürümü `144+`
- uzaktan hata ayıklama, söz konusu tarayıcının inceleme sayfasında etkin
- tarayıcı, bağlanma onayı istemini gösterdi ve bu istem kabul edildi
- Chrome açık bir `--remote-debugging-port` ile başlatıldıysa Chrome MCP otomatik bağlantısına
  güvenmek yerine `browser.profiles.<name>.cdpUrl` değerini bu DevTools uç noktası olarak
  ayarlayın
- `openclaw doctor`, eski uzantı tabanlı tarayıcı yapılandırmasını taşır ve varsayılan
  otomatik bağlantı profilleri için Chrome'un yerel olarak yüklü olduğunu denetler, ancak
  tarayıcı tarafındaki uzaktan hata ayıklamayı sizin için etkinleştiremez

Ajan kullanımı:

- Kullanıcının oturum açmış tarayıcı durumuna ihtiyaç duyduğunuzda `profile="user"` kullanın.
- Özel bir mevcut oturum profili kullanıyorsanız bu profil adını açıkça iletin.
- Bu modu yalnızca kullanıcı bağlanma istemini onaylamak üzere bilgisayarın
  başındayken seçin.
- Gateway veya Node ana makinesi `npx chrome-devtools-mcp@latest --autoConnect` başlatabilir.

Notlar:

- Bu yol, oturum açmış tarayıcı oturumunuz içinde işlem yapabildiğinden yalıtılmış
  `openclaw` profilinden daha yüksek risk taşır.
- OpenClaw bu sürücü için tarayıcıyı başlatmaz; yalnızca tarayıcıya bağlanır.
- OpenClaw burada resmî Chrome DevTools MCP `--autoConnect` akışını kullanır. `userDataDir`
  ayarlanmışsa söz konusu kullanıcı verileri dizinini hedeflemek üzere doğrudan iletilir.
- Mevcut oturum, seçilen ana makinede veya bağlı bir tarayıcı Node'u üzerinden
  bağlanabilir. Chrome başka bir yerdeyse ve hiçbir tarayıcı Node'u bağlı değilse
  bunun yerine uzak CDP veya bir Node ana makinesi kullanın.
- Chrome MCP hedefleri ve anlık görüntü referansları tek bir MCP alt işlemiyle sınırlıdır.
  Bu işlem yeniden başladıktan sonra `browser tabs` komutunu tekrar çalıştırın, hedefe özgü
  çalışmalardan önce yeni bir hedefi açıkça seçin ve referansları kullanmadan önce yeni bir anlık
  görüntü alın. Her referans yalnızca kendi hedefi ve en son anlık görüntüsü için geçerlidir.
  URL'si eşleşse bile eski takma adlar yeni bir sekmeye aktarılmaz.
- Chrome DevTools MCP şu anda sayfa araçlarını işleme özgü sayısal bir sayfa
  kimliğine göre yönlendirir. İşlem kapsamlı tanıtıcılar, alt işlem değişimlerinde yeniden
  kullanımı önler; ancak bitişik araç çağrıları arasında işlem içi tarayıcı bağlamının
  değiştirilmesi yine de bir eylemi başka bir hedefe yönlendirebilir. Tamamen atomik
  yönlendirme, kararlı hedef kimlikleri için üst kaynak sayfa aracı desteği gerektirir.

### Özel Chrome MCP başlatma

Varsayılan `npx chrome-devtools-mcp@latest` akışı istediğiniz biçimde değilse (çevrimdışı ana makineler,
sabitlenmiş sürümler, projeye dahil edilmiş ikili dosyalar) başlatılan Chrome DevTools MCP
sunucusunu profil bazında geçersiz kılın:

| Alan         | İşlevi                                                                                                                     |
| ------------ | -------------------------------------------------------------------------------------------------------------------------- |
| `mcpCommand` | `npx` yerine başlatılacak yürütülebilir dosya. Olduğu gibi çözümlenir; mutlak yollar dikkate alınır.                     |
| `mcpArgs`    | `mcpCommand` öğesine değiştirilmeden iletilen bağımsız değişken dizisi. Varsayılan `chrome-devtools-mcp@latest --autoConnect` bağımsız değişkenlerinin yerini alır. |

Mevcut oturum profilinde `cdpUrl` ayarlandığında OpenClaw,
`--autoConnect` adımını atlar ve uç noktayı otomatik olarak Chrome MCP'ye iletir:

- `http(s)://...` → `--browserUrl <url>` (DevTools HTTP keşif uç noktası).
- `ws(s)://...` → `--wsEndpoint <url>` (doğrudan CDP WebSocket).

Uç nokta bayrakları ve `userDataDir` birlikte kullanılamaz: `cdpUrl` ayarlandığında,
Chrome MCP bir profil dizini açmak yerine uç noktanın arkasındaki
çalışan tarayıcıya bağlandığından, Chrome MCP başlatılırken `userDataDir`
yok sayılır.

<Accordion title="Mevcut oturum özellik sınırlamaları">

Yönetilen `openclaw` profiliyle karşılaştırıldığında mevcut oturum sürücüleri daha kısıtlıdır:

- **Ekran görüntüleri** - sayfa yakalamaları ve `--ref` öğe yakalamaları çalışır; CSS `--element` seçicileri çalışmaz. Sayfa veya referans tabanlı öğe ekran görüntüleri için Playwright gerekli değildir. (`--full-page`, yalnızca mevcut oturumda değil, hiçbir profilde `--ref` veya `--element` ile birlikte kullanılamaz.)
- **Eylemler** - `click`, `type`, `hover`, `scrollIntoView`, `drag` ve `select` anlık görüntü referansları gerektirir (CSS seçicileri kullanılamaz). `click-coords`, görünür görüntü alanı koordinatlarına tıklar ve anlık görüntü referansı gerektirmez. `click` yalnızca sol düğmeyi destekler (düğme geçersiz kılmaları veya değiştiriciler yoktur). `type`, `slowly=true` seçeneğini desteklemez; `fill` veya `press` kullanın. `press`, `delayMs` seçeneğini desteklemez. `type`, `hover`, `scrollIntoView`, `drag`, `select` ve `fill`, çağrı başına `timeoutMs` geçersiz kılmalarını desteklemez; `evaluate` destekler. `select` tek bir değer kabul eder. `batch` desteklenmez; eylemleri ayrı ayrı gönderin.
- **Bekleme / yükleme / iletişim kutusu** - `wait --url` tam eşleşme, alt dize ve glob kalıplarını destekler (yönetilen profille aynıdır); `wait --load networkidle` mevcut oturum profillerinde desteklenmez (yönetilen ve ham/uzak CDP profillerinde çalışır). Yükleme kancaları, her seferinde bir dosya olmak üzere `ref` veya `inputRef` gerektirir; CSS `element` kullanılamaz. İletişim kutusu kancaları zaman aşımı geçersiz kılmalarını veya `dialogId` seçeneğini desteklemez.
- **İletişim kutusu görünürlüğü** - Bir eylem kalıcı iletişim kutusu açtığında, yönetilen tarayıcı eylemi yanıtları `blockedByDialog` ve `browserState.dialogs.pending` içerir; anlık görüntüler de bekleyen iletişim kutusu durumunu içerir. Bir iletişim kutusu beklemedeyken `browser dialog --accept/--dismiss --dialog-id <id>` ile yanıt verin. OpenClaw dışında işlenen iletişim kutuları `browserState.dialogs.recent` altında görünür.
- **Yalnızca yönetilen profilde kullanılabilen özellikler** - PDF dışa aktarma, indirme yakalama ve `responsebody` hâlâ yönetilen tarayıcı yolunu gerektirir.

</Accordion>

## Yalıtım garantileri

- **Ayrılmış kullanıcı verileri dizini**: kişisel tarayıcı profilinize hiçbir zaman dokunmaz.
- **Ayrılmış bağlantı noktaları**: geliştirme iş akışlarıyla çakışmaları önlemek için `9222` kullanımından kaçınır.
- **Belirlenimci sekme denetimi**: `tabs` önce `suggestedTargetId`, ardından
  `t1` gibi kararlı `tabId` tanıtıcılarını, isteğe bağlı etiketleri ve ham `targetId` değerini döndürür.
  Aracılar `suggestedTargetId` değerini yeniden kullanmalıdır; ham kimlikler
  hata ayıklama ve uyumluluk için kullanılabilir olmaya devam eder.

## Tarayıcı seçimi

Yerel olarak başlatılırken OpenClaw kullanılabilir ilk tarayıcıyı seçer:

1. Chrome
2. Brave
3. Edge
4. Chromium
5. Chrome Canary

`browser.executablePath` ile geçersiz kılabilirsiniz.

Platformlar:

- macOS: `/Applications` ve `~/Applications` konumlarını denetler.
- Linux: `/usr/bin`,
  `/snap/bin`, `/opt/google`, `/opt/brave.com`, `/usr/lib/chromium` ve
  `/usr/lib/chromium-browser` altındaki yaygın Chrome/Brave/Edge/Chromium konumlarının yanı sıra
  `PLAYWRIGHT_BROWSERS_PATH` veya `~/.cache/ms-playwright` altındaki Playwright tarafından yönetilen
  Chromium'u denetler.
- Windows: yaygın kurulum konumlarını denetler.

## Denetim API'si (isteğe bağlı)

Betik oluşturma ve hata ayıklama için Gateway, küçük bir **yalnızca geri döngüden
erişilebilen HTTP denetim API'si** ve bununla eşleşen bir `openclaw browser` CLI
(anlık görüntüler, referanslar, bekleme geliştirmeleri, JSON çıktısı, hata ayıklama
iş akışları) sunar. Tam başvuru için
[Tarayıcı denetim API'si](/tr/tools/browser-control) bölümüne bakın.

## Sorun giderme

Linux'a özgü sorunlar (özellikle snap Chromium) için
[Tarayıcı sorunlarını giderme](/tr/tools/browser-linux-troubleshooting) bölümüne bakın.

WSL2 Gateway + Windows Chrome ayrık ana bilgisayar kurulumları için
[WSL2 + Windows + uzak Chrome CDP sorunlarını giderme](/tr/tools/browser-wsl2-windows-remote-cdp-troubleshooting) bölümüne bakın.

### CDP başlatma hatası ile gezinme SSRF engeli arasındaki fark

Bunlar farklı hata sınıflarıdır ve farklı kod yollarına işaret eder.

- **CDP başlatma veya hazır olma hatası**, OpenClaw'ın tarayıcı denetim düzleminin sağlıklı olduğunu doğrulayamadığı anlamına gelir.
- **Gezinme SSRF engeli**, tarayıcı denetim düzleminin sağlıklı olduğu ancak bir sayfa gezinme hedefinin ilke tarafından reddedildiği anlamına gelir.

Yaygın örnekler:

- CDP başlatma veya hazır olma hatası:
  - `Chrome CDP websocket for profile "openclaw" is not reachable after start`
  - `Remote CDP for profile "<name>" is not reachable at <cdpUrl>`
  - `attachOnly: true` olmadan bir
    geri döngü harici CDP hizmeti yapılandırıldığında `Port <port> is in use for profile "<name>" but not by openclaw`
- Gezinme SSRF engeli:
  - `start` ve `tabs` çalışmaya devam ederken `open`, `navigate`, anlık görüntü veya sekme açma akışları bir tarayıcı/ağ ilkesi hatasıyla başarısız olur

İkisini ayırt etmek için şu asgari komut dizisini kullanın:

```bash
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw tabs
openclaw browser --browser-profile openclaw open https://example.com
```

Sonuçları yorumlama:

- `start`, `not reachable after start` ile başarısız olursa önce CDP hazır olma sorununu giderin.
- `start` başarılı olur ancak `tabs` başarısız olursa denetim düzlemi hâlâ sağlıksızdır. Bunu sayfa gezinme sorunu olarak değil, CDP erişilebilirlik sorunu olarak değerlendirin.
- `start` ve `tabs` başarılı olur ancak `open` veya `navigate` başarısız olursa tarayıcı denetim düzlemi çalışıyordur ve hata gezinme ilkesinde ya da hedef sayfadadır.
- `start`, `tabs` ve `open` öğelerinin tümü başarılı olursa temel yönetilen tarayıcı denetim yolu sağlıklıdır.

Önemli davranış ayrıntıları:

- `browser.ssrfPolicy` yapılandırılmasa bile tarayıcı yapılandırması varsayılan olarak hatada kapalı bir SSRF ilkesi nesnesi kullanır.
- Yerel geri döngü `openclaw` yönetilen profili için CDP sistem durumu denetimleri, OpenClaw'ın kendi yerel denetim düzleminde tarayıcı SSRF erişilebilirlik zorlamasını kasıtlı olarak atlar.
- Gezinme koruması ayrıdır. Başarılı bir `start` veya `tabs` sonucu, daha sonraki bir `open` ya da `navigate` hedefine izin verildiği anlamına gelmez.

Güvenlik yönergeleri:

- Tarayıcı SSRF ilkesini varsayılan olarak **gevşetmeyin**.
- Geniş özel ağ erişimi yerine `hostnameAllowlist` veya `allowedHostnames` gibi dar kapsamlı ana bilgisayar istisnalarını tercih edin.
- `dangerouslyAllowPrivateNetwork: true` seçeneğini yalnızca özel ağ tarayıcı erişiminin gerekli olduğu ve incelendiği, bilinçli olarak güvenilen ortamlarda kullanın.

## Agent araçları + denetimin çalışma biçimi

Agent, tarayıcı otomasyonu için **tek bir araç** edinir:

- `browser` - doctor/status/start/stop/tabs/open/focus/close/snapshot/screenshot/navigate/act

Eşleştirilme biçimi:

- `browser snapshot` kararlı bir kullanıcı arayüzü ağacı (AI veya ARIA) döndürür.
- `browser act`, tıklamak/yazmak/sürüklemek/seçmek için anlık görüntüdeki `ref` kimliklerini kullanır.
- `browser screenshot` pikselleri yakalar (tam sayfa, öğe veya etiketli referanslar).
- `browser doctor`; Gateway, plugin, profil, tarayıcı ve sekmenin hazır olup olmadığını denetler.
- `browser` şunları kabul eder:
  - Adlandırılmış bir tarayıcı profili (openclaw, chrome veya uzak CDP) seçmek için `profile`.
  - Tarayıcının nerede çalışacağını seçmek için `target` (`sandbox` | `host` | `node`).
  - Korumalı alan oturumlarında `target: "host"`, `agents.defaults.sandbox.browser.allowHostControl=true` gerektirir.
  - `target` belirtilmezse korumalı alan oturumları varsayılan olarak `sandbox`, korumalı alan dışındaki oturumlar ise `host` kullanır.
  - Tarayıcı özellikli bir Node bağlıysa `target="host"` veya `target="node"` ile sabitlemediğiniz sürece araç otomatik olarak ona yönlendirilebilir.

Bu, ajanın belirlenimsel kalmasını sağlar ve kırılgan seçicileri önler.

## İlgili

- [Araçlara Genel Bakış](/tr/tools) - kullanılabilir tüm ajan araçları
- [Korumalı Alan](/tr/gateway/sandboxing) - korumalı alan ortamlarında tarayıcı denetimi
- [Güvenlik](/tr/gateway/security) - tarayıcı denetimi riskleri ve sağlamlaştırma
