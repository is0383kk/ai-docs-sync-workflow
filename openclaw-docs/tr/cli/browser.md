---
read_when:
    - '`openclaw browser` kullanıyorsunuz ve yaygın görevler için örnekler istiyorsunuz'
    - Başka bir makinede çalışan tarayıcıyı bir Node ana makinesi üzerinden denetlemek istiyorsunuz
    - Chrome MCP aracılığıyla yerel olarak oturum açtığınız Chrome'a bağlanmak istiyorsunuz
summary: '`openclaw browser` için CLI başvurusu (yaşam döngüsü, profiller, sekmeler, eylemler, durum ve hata ayıklama)'
title: Tarayıcı
x-i18n:
    generated_at: "2026-07-26T23:51:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 62eb41248cda87cef96be7b0dfe3e0d36a9d3e1ee55c165bd8e3efd68d1e9a5e
    source_path: cli/browser.md
    workflow: 16
---

# `openclaw browser`

OpenClaw'ın tarayıcı kontrol yüzeyini yönetin ve tarayıcı eylemlerini çalıştırın: yaşam döngüsü, profiller, sekmeler, anlık görüntüler, ekran görüntüleri, gezinme, giriş, durum emülasyonu ve hata ayıklama.

İlgili: [Tarayıcı aracı](/tr/tools/browser)

## Ortak bayraklar

- `--url <gatewayWsUrl>`: Gateway WebSocket URL'si (varsayılan olarak yapılandırmayı kullanır).
- `--token <token>`: Gateway belirteci (gerekiyorsa).
- `--timeout <ms>`: ms cinsinden istek zaman aşımı (varsayılan: `30000`).
- `--expect-final`: nihai bir Gateway yanıtını bekler.
- `--browser-profile <name>`: bir tarayıcı profili seçer (varsayılan: `openclaw` veya `browser.defaultProfile`).
- `--json`: makine tarafından okunabilir çıktı (desteklendiği yerlerde). Bu, tarayıcı düzeyinde bir seçenektir; bu nedenle
  belirsizliği önleyen bir biçim için alt komuttan önce yerleştirin, örneğin
  `openclaw browser --json status`. Sonda yerleştirme, örneğin
  `openclaw browser status --json`, seçilen alt komut kendi
  `--json` seçeneğini tanımlamadığında da çalışır.

## Hızlı başlangıç (yerel)

```bash
openclaw browser profiles
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

Ajanlar aynı hazırlık denetimini `browser({ action: "doctor" })` ile çalıştırabilir.

## Hızlı sorun giderme

`start`, `not reachable after start` hatasıyla başarısız olursa önce CDP hazırlığını giderin. `start` ve `tabs` başarılı olduğu hâlde `open` veya `navigate` başarısız olursa tarayıcı kontrol düzlemi sağlıklıdır ve hata genellikle gezinme SSRF ilkesi engelinden kaynaklanır.

Asgari sıra:

```bash
openclaw browser --browser-profile openclaw doctor
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw tabs
openclaw browser --browser-profile openclaw open https://example.com
```

Ayrıntılı rehber: [Tarayıcı sorunlarını giderme](/tr/tools/browser#cdp-startup-failure-vs-navigation-ssrf-block)

## Yaşam döngüsü

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep
openclaw browser start
openclaw browser start --headless
openclaw browser stop
openclaw browser --browser-profile openclaw reset-profile
```

- `doctor --deep` canlı bir anlık görüntü araştırması ekler: temel CDP hazırlığı sorunsuzken geçerli sekmenin incelenebildiğine dair kanıt istediğinizde kullanışlıdır.
- Çalışan bir yerel yönetilen profil için `status` ve `doctor`, Chrome'dan önbelleğe alınmış
  grafik tanılama bilgilerini bildirir: donanım/yazılım sınıflandırması, işleyici,
  arka uç, cihaz/sürücü, özellik ve devre dışı durum ayrıntıları ile hızlandırılmış
  video yetenekleri. `openclaw browser --json status` yapılandırılmış yükün tamamını döndürür.
  Pasif durum, yalnızca bu bilgileri toplamak için Chrome'u hiçbir zaman başlatmaz.
- `stop`, OpenClaw'ın tarayıcı işlemini kendisinin başlatmadığı `attachOnly` ve uzak CDP profillerinde bile etkin kontrol oturumunu kapatır ve geçici emülasyon geçersiz kılmalarını temizler. Yerel yönetilen profillerde `stop` ayrıca başlatılan tarayıcı işlemini durdurur.
- `start --headless` yalnızca söz konusu başlatma isteği için ve yalnızca OpenClaw yerel yönetilen bir tarayıcı başlattığında geçerlidir. `browser.headless` veya profil yapılandırmasını yeniden yazmaz ve zaten çalışan bir tarayıcıda hiçbir işlem yapmaz.
- `DISPLAY` veya `WAYLAND_DISPLAY` bulunmayan Linux ana makinelerinde, `OPENCLAW_BROWSER_HEADLESS=0`, `browser.headless=false` veya `browser.profiles.<name>.headless=false` görünür bir tarayıcıyı açıkça istemediği sürece yerel yönetilen profiller otomatik olarak başsız çalışır.

## Komut eksikse

`openclaw browser` bilinmeyen bir komutsa `~/.openclaw/openclaw.json` içindeki `plugins.allow` öğesini denetleyin. `plugins.allow` mevcutsa, yapılandırmada zaten kök düzeyinde bir `browser` bloğu olmadığı sürece paketlenmiş tarayıcı Plugin'ini açıkça listeleyin:

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

Açık bir kök `browser` bloğu (örneğin `browser.enabled=true` veya `browser.profiles.<name>`) da kısıtlayıcı bir Plugin izin listesi altında paketlenmiş tarayıcı Plugin'ini etkinleştirir.

İlgili: [Tarayıcı aracı](/tr/tools/browser#missing-browser-command-or-tool)

## Profiller

Profiller, adlandırılmış tarayıcı yönlendirme yapılandırmalarıdır:

- `openclaw` (varsayılan): OpenClaw tarafından yönetilen özel bir Chrome örneğini başlatır veya ona bağlanır (yalıtılmış kullanıcı verileri dizini).
- `user`: Chrome DevTools MCP aracılığıyla mevcut oturum açılmış Chrome oturumunuzu kontrol eder.
- özel CDP profilleri: yerel veya uzak bir CDP uç noktasını gösterir.

```bash
openclaw browser profiles
openclaw browser system-profiles
openclaw browser system-profiles --browser brave
openclaw browser import-profile --browser chrome --system Default --into imported
openclaw browser import-profile --system "Profile 1" --into work --domains google.com,youtube.com
openclaw browser create-profile --name work --color "#FF5A36"
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name remote --cdp-url https://browser-host.example.com
openclaw browser delete-profile --name work
```

Herhangi bir alt komutta `--browser-profile <name>` ile belirli bir profil kullanın; örneğin `openclaw browser --browser-profile work tabs`.

macOS'te `system-profiles`, ana makinede kullanılabilen gerçek Chrome, Brave, Edge veya Chromium profillerini listeler. `import-profile`, bir macOS Keychain/Touch ID onay isteminden sonra bunların çerezlerinin şifresini çözer ve çerezleri yeni bir OpenClaw tarafından yönetilen profile ekler. Yalnızca çerezleri içe aktarır; yerel depolama ve IndexedDB değişmeden kalır. Bazı Google oturumları cihaza bağlı oturum kimlik bilgilerini (DBSC) kullanır ve içe aktarmadan sonra yine de yeniden kimlik doğrulama gerektirebilir.

macOS uygulaması yerel bir Gateway kullandığında bu içe aktarmayı bir kez sunabilir ve yalıtılmış içe aktarılmış profili ajan taraması için varsayılan hâle getirebilir. İçe aktarma her zaman açık bir tıklama gerektirir; başarılı içe aktarma veya iletişim kutusunun kapatılması sonraki otomatik istemleri engeller ve yeniden içe aktarma için **Settings → General → Browser login** kullanılabilir durumda kalır.

Sistem profili içe aktarma varsayılan olarak etkindir. Hem CLI hem de ajan tarafından tetiklenen içe aktarmaları devre dışı bırakmak için `browser.allowSystemProfileImport=false` ayarını kullanın. İçe aktarma ana makineye özeldir ve tarayıcı Node proxy'si üzerinden çalıştırılamaz.

## Sekmeler

```bash
openclaw browser tabs
openclaw browser tab new --label docs
openclaw browser tab label t1 docs
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://docs.openclaw.ai --label docs
openclaw browser focus docs
openclaw browser close t1
```

`tabs` önce `suggestedTargetId`, ardından kararlı `tabId` (örneğin `t1`), isteğe bağlı etiketi ve ham `targetId` değerini döndürür. `suggestedTargetId` değerini `focus`, `close`, anlık görüntüler ve eylemlere geri iletin. `open --label`, `tab new --label` veya `tab label` ile bir etiket atayın; etiketler, sekme kimlikleri, ham hedef kimlikleri ve benzersiz hedef kimliği öneklerinin tümü kabul edilir. İstek alanı uyumluluk için hâlâ `targetId` olarak adlandırılır ancak bu sekme başvurularının herhangi birini kabul eder.

Ham hedef kimlikleri kalıcı ajan belleği değil, geçici tanılama tanıtıcılarıdır: Chromium, gezinme veya form gönderimi sırasında temeldeki ham hedefi değiştirdiğinde OpenClaw eşleşmeyi kanıtlayabiliyorsa kararlı `tabId`/etiketi yeni sekmeye bağlı tutar. `suggestedTargetId` tercih edin.

## Anlık görüntü / ekran görüntüsü / eylemler

Anlık görüntü:

```bash
openclaw browser snapshot
openclaw browser snapshot --urls
```

Ekran görüntüsü:

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref e12
openclaw browser screenshot --labels
```

- `--full-page` yalnızca sayfa yakalamaları içindir; `--ref` veya `--element` ile birlikte kullanılamaz.
- `existing-session` / `user` profilleri sayfa ekran görüntülerini ve anlık görüntü çıktısındaki `--ref` ekran görüntülerini destekler ancak CSS `--element` ekran görüntülerini desteklemez.
- `--labels`, geçerli anlık görüntü başvurularını ekran görüntüsünün üzerine bindirir. Playwright destekli profillerde `--full-page` (tam sayfa bindirmesi), `--ref` (ARIA başvurusuna göre öğe kırpma bindirmesi) ve `--element` (CSS seçicisine göre öğe kırpma bindirmesi) ile çalışır; öğe kırpma modlarında etiketler öğeye göre yansıtılır. Yanıt ayrıca, yakalanan görüntünün koordinat alanında (görünüm alanı / tam sayfa / öğeye göre) her başvurunun sınırlayıcı kutusunu içeren bir `annotations` dizisi (boş olduğunda atlanır) içerir: `ref`, `number`, `role`, isteğe bağlı `name` ve `box: {x, y, width, height}`.
  `existing-session` profilleri sayfa ekran görüntülerinde bir chrome-mcp bindirmesi oluşturur ancak Playwright yansıtma yardımcısını kullanmaz ve `annotations` içermez; CSS `--element` ekran görüntüleri burada desteklenmez. Playwright veya chrome-mcp olmadan etiketli ekran görüntüleri kullanılamaz.
- `snapshot --urls`, keşfedilen bağlantı hedeflerini yapay zekâ anlık görüntülerine ekler; böylece ajanlar yalnızca bağlantı metninden tahminde bulunmak yerine doğrudan gezinme hedeflerini seçebilir.

Gezinme/tıklama/yazma (başvuru tabanlı kullanıcı arayüzü otomasyonu):

```bash
openclaw browser navigate https://example.com
openclaw browser click <ref>
openclaw browser click-coords 120 340
openclaw browser type <ref> "hello"
openclaw browser press Enter
openclaw browser hover <ref>
openclaw browser scrollintoview <ref>
openclaw browser drag <startRef> <endRef>
openclaw browser select <ref> OptionA OptionB
openclaw browser fill --fields '[{"ref":"1","value":"Ada"}]'
openclaw browser wait --text "Done"
openclaw browser evaluate --fn '(el) => el.textContent' --ref <ref>
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
```

`evaluate --fn` bir işlev kaynağını, ifadeyi veya deyim gövdesini kabul eder. Deyim gövdeleri zaman uyumsuz işlevler olarak sarmalanır; bu nedenle geri almak istediğiniz değer için `return` kullanın. Sayfa tarafındaki işlev varsayılan değerlendirme zaman aşımından daha uzun sürebilecekse `--timeout-ms` kullanın. `browser.evaluateEnabled=false` (varsayılan: `true`) hem `evaluate` hem de `wait --fn` öğelerini devre dışı bırakır.

OpenClaw yeni sekmeyi kanıtlayabildiğinde eylem yanıtları, eylemin tetiklediği sayfa değişiminden sonra geçerli ham `targetId` değerini döndürür. Betikler, uzun ömürlü iş akışları için yine de `suggestedTargetId`/etiketleri depolayıp iletmelidir.

Dosya ve iletişim kutusu yardımcıları:

```bash
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref <ref>
openclaw browser upload media://inbound/file.pdf --ref <ref>
openclaw browser waitfordownload
openclaw browser download <ref> report.pdf
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
```

Yönetilen Chrome profilleri, sıradan tıklamayla tetiklenen indirmeleri OpenClaw indirmeler dizinine (varsayılan olarak `/tmp/openclaw/downloads` veya yapılandırılmış geçici kök) kaydeder. Ajanın belirli bir dosyayı bekleyip yolunu döndürmesi gerektiğinde `waitfordownload` veya `download` kullanın; sonraki indirme bu açık bekleyicilere aittir. Yüklemeler, `media://inbound/<id>` ve korumalı alanla göreli `media/inbound/<id>` başvuruları dâhil olmak üzere OpenClaw geçici yüklemeler kökündeki ve OpenClaw tarafından yönetilen gelen medyadaki dosyaları kabul eder. İç içe medya başvuruları, dizin geçişi ve rastgele yerel yollar reddedilir.

Bir eylem kalıcı bir iletişim kutusu açtığında eylem yanıtı, `browserState.dialogs.pending` ile birlikte `blockedByDialog` döndürür; doğrudan yanıtlamak için `--dialog-id` iletin. OpenClaw dışında işlenen iletişim kutuları `browserState.dialogs.recent` altında görünür.

Toplu eylemler:

```bash
openclaw browser batch --actions '[{"kind":"wait","timeMs":500},{"kind":"click","ref":"12"},{"kind":"type","ref":"23","text":"hello"}]'
openclaw browser batch --actions-file plan.json
openclaw browser batch --actions-file - --continue
```

`openclaw browser batch`, iç içe `BrowserActRequest` eylemleriyle (`wait`, `click`, `type`, `evaluate`, ...) bir `kind="batch"` `/act` isteği gönderir — CLI alt komutları olan `open`/`navigate`/`snapshot`/`screenshot` değil; bunlar `/act` türleri değildir. `--continue`, `stopOnError=false` değerini ayarlar (varsayılan olarak ilk hatada durur); `--target-id`, tüm toplu işlemi tek bir sekmeyle sınırlar. İç içe bir eylemin başarısız olması, komutun sıfır olmayan bir kodla sonlanmasına neden olur; sıralı `results` yanıtını korumak için `--json` kullanın. Tam sözleşme (ref yaşam döngüsü, hedef kimliği çakışmaları, hata özeti) için [Tarayıcı toplu işlem CLI'si](/tr/tools/browser-control#browser-batch-cli) bölümüne bakın. `batch`, `profile="user"` / mevcut oturum profillerinde desteklenmez.

## Durum ve depolama

Görünüm alanı + emülasyon:

```bash
openclaw browser resize 1280 720
openclaw browser set viewport 1280 720
openclaw browser set offline on
openclaw browser set media dark
openclaw browser set timezone Europe/London
openclaw browser set locale en-GB
openclaw browser set geo 51.5074 -0.1278 --accuracy 25
openclaw browser set device "iPhone 14"
openclaw browser set headers '{"x-test":"1"}'
openclaw browser set credentials myuser mypass
```

Çerezler + depolama:

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url https://example.com
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set token abc123
openclaw browser storage session clear
```

## Hata ayıklama

```bash
openclaw browser console --level error
openclaw browser pdf
openclaw browser responsebody "**/api"
openclaw browser highlight <ref>
openclaw browser errors --clear
openclaw browser requests --filter api
openclaw browser trace start
openclaw browser trace stop --out trace.zip
```

## MCP aracılığıyla mevcut Chrome

Yerleşik `user` profilini kullanın veya kendi `existing-session` profilinizi oluşturun:

```bash
openclaw browser --browser-profile user tabs
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name brave-live --driver existing-session --user-data-dir "~/Library/Application Support/BraveSoftware/Brave-Browser"
openclaw browser create-profile --name chrome-port --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser --browser-profile chrome-live tabs
```

Varsayılan mevcut oturum yolu, yalnızca ana makinede çalışan Chrome MCP otomatik bağlantısıdır. Tarayıcı zaten bir DevTools uç noktasıyla çalışıyorsa Chrome MCP'nin bunun yerine bu uç noktaya bağlanması için `--cdp-url` iletin. Chrome MCP semantiğinin gerekli olmadığı Docker, Browserless veya diğer uzak kurulumlarda bunun yerine bir CDP profili kullanın.

Mevcut oturumun güncel sınırlamaları:

- Anlık görüntü odaklı eylemler CSS seçicileri yerine ref'leri kullanır.
- Desteklenen `act` istekleri, çağıranlar `timeoutMs` değerini atladığında yerleşik 60000 ms varsayılanını kullanır; çağrı başına `timeoutMs` yine de önceliklidir.
- `click` yalnızca sol tıklamayı destekler.
- `type`, `slowly=true` desteği sunmaz.
- `press`, `delayMs` desteği sunmaz.
- `hover`, `scrollintoview`, `drag`, `select` ve `fill`, çağrı başına zaman aşımı geçersiz kılmalarını reddeder; `evaluate`, `--timeout-ms` değerini kabul eder.
- `select` yalnızca tek bir değeri destekler.
- `wait --load networkidle` desteklenmez (yönetilen ve ham/uzak CDP profillerinde çalışır).
- Dosya yüklemeleri `--ref` / `--input-ref` gerektirir, CSS `--element` desteği sunmaz ve aynı anda yalnızca tek bir dosyayı destekler.
- İletişim kutusu kancaları `--timeout` desteği sunmaz.
- Ekran görüntüleri sayfa yakalamalarını ve `--ref` destekler, ancak CSS `--element` desteği sunmaz.
- `responsebody`, indirme engelleme, PDF dışa aktarma ve toplu eylemler için hâlâ yönetilen bir tarayıcı veya ham CDP profili gerekir.

## Uzak tarayıcı denetimi (node ana makine proxy'si)

Gateway tarayıcıdan farklı bir makinede çalışıyorsa Chrome/Brave/Edge/Chromium'un bulunduğu makinede bir **node ana makinesi** çalıştırın. Gateway, tarayıcı eylemlerini bu node'a proxy üzerinden iletir; ayrı bir tarayıcı denetim sunucusu gerekmez.

Otomatik yönlendirmeyi denetlemek için `gateway.nodes.browser.mode`, birden fazla node bağlıysa belirli bir node'u sabitlemek için `gateway.nodes.browser.node` kullanın.

Güvenlik + uzak kurulum: [Tarayıcı aracı](/tr/tools/browser), [Uzak erişim](/tr/gateway/remote), [Tailscale](/tr/gateway/tailscale), [Güvenlik](/tr/gateway/security)

## İlgili

- [CLI referansı](/tr/cli)
- [Tarayıcı](/tr/tools/browser)
