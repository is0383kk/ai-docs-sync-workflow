---
read_when:
    - Gateway'i bir tarayıcıdan yönetmek istiyorsunuz
    - SSH tünelleri olmadan Tailnet erişimi istiyorsunuz
sidebarTitle: Control UI
summary: Gateway için tarayıcı tabanlı kontrol kullanıcı arayüzü (sohbet, etkinlik, Node'lar, yapılandırma)
title: Kontrol Arayüzü
x-i18n:
    generated_at: "2026-07-27T00:10:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 069bad7f3c8fce46759893e16d2dac86047c0929d6d866d25ce3b080204c1180
    source_path: web/control-ui.md
    workflow: 16
---

Control UI, Gateway tarafından sunulan küçük bir **Vite + Lit** tek sayfalı uygulamadır:

- varsayılan: `http://<host>:18789/`
- isteğe bağlı önek: `gateway.controlUi.basePath` olarak ayarlayın (ör. `/openclaw`)

Aynı port üzerinden **doğrudan Gateway WebSocket ile** iletişim kurar.

Çalışan bir oturumu izlerken Gateway, kısa bir durum özeti oluşturmak için söz konusu aracının yardımcı modelini kullanabilir. Sohbet, bunu değerlendirme, plan ilerlemesi, pull request'ler ve geçen süreyi içeren bir karta genişleyen tek satırlık bir durum rozeti olarak gösterir. Bir çalışma takıldığında veya girdi gerektirdiğinde kart bir kez genişleyebilir; `/btw` yan sohbeti, genişletilmiş karta göre önceliklidir.

Genişletilmiş kart, çalışma hakkında kısa soruları da kabul eder. Yanıtlar yalnızca gözlemcinin mevcut özetini ve arındırılmış, sınırlandırılmış notları kullanır; o oturum boyunca tarayıcıda kalır ve ana aracı çalışmasına hiçbir zaman girmez veya onu kesintiye uğratmaz. Gözlemler yanıtı içermiyorsa gözlemci bunu bilemeyeceğini belirtir.

İlk özet geldikten sonra sezgisel canlı etkinlik yerine söz konusu çalışmanın kenar çubuğu alt başlığını yönetir. Tamamlandı veya başarısız şeklindeki son özet, oturum okunmamış durumdayken görünür kalır; ardından satır normal çalışma alt başlığına döner.

Oturum gözlemi varsayılan olarak etkindir. **Settings > Appearance > Sidebar** bölümünde bunu Gateway genelinde kapatabilir, çözümlenen küçük modeli ve kaynağını inceleyebilir ya da otomatik yönlendirmeyi seçebilir, yardımcı görevleri devre dışı bırakabilir veya açık bir `agents.defaults.utilityModel` seçebilirsiniz. Eşdeğer yapılandırma denetimleri `gateway.controlUi.sessionObserver: false` ve `agents.defaults.utilityModel: ""` şeklindedir.

## Hızlı açma (yerel)

Gateway aynı bilgisayarda çalışıyorsa [http://127.0.0.1:18789/](http://127.0.0.1:18789/) (veya [http://localhost:18789/](http://localhost:18789/)) adresini açın.

Sayfa yüklenmezse önce Gateway'i başlatın: `openclaw gateway`.

<Note>
Yerel Windows LAN bağlamalarında, Gateway ana bilgisayarında `127.0.0.1` çalışsa bile Windows Firewall veya kuruluş tarafından yönetilen Group Policy, duyurulan LAN URL'sini engelleyebilir. Windows ana bilgisayarında `openclaw gateway status --deep` komutunu çalıştırın; bu komut engellenmiş olabilecek portları, profil uyuşmazlıklarını ve ilkenin yok sayabileceği yerel güvenlik duvarı kurallarını bildirir.
</Note>

Kimlik doğrulama, WebSocket el sıkışması sırasında şu yollarla sağlanır:

- `connect.params.auth.token`
- `connect.params.auth.password`
- `gateway.auth.allowTailscale: true` olduğunda Tailscale Serve kimlik üstbilgileri
- `gateway.auth.mode: "trusted-proxy"` olduğunda güvenilir proxy kimlik üstbilgileri

Gateway kimlik doğrulaması cihaz eşleştirmesinden önce çalışır. Doğrudan geri döngü bağlantısı, belirteç veya parola kimlik doğrulamasını atlamaz. Pano ayarları paneli, geçerli tarayıcı sekmesi oturumu ve seçili Gateway URL'si için bir belirteci saklar; parolalar kalıcı olarak saklanmaz. Eşleştirmeden sonra tarayıcı, sonraki bağlantılarda cihaz başına saklanan belirtecini kullanabilir.

İlk katılım genellikle paylaşılan gizli anahtar kimlik doğrulaması için bir Gateway belirteci yapılandırır. Gateway, yapılandırılmış bir belirteç olmadan belirteç modunda başlarsa bunun yerine söz konusu işlem için geçici bir çalışma zamanı belirteci oluşturur. Çalışma zamanı belirteci yapılandırmaya yazılmaz; bu nedenle `openclaw config get gateway.auth.token` bunu alamaz ve bu belirtece sahip olmayan bir geri döngü tarayıcısı reddedilir. `openclaw doctor --generate-gateway-token` komutunu çalıştırın, Gateway'i yeniden başlatın ve ardından yapılandırılmış belirteci Control UI ayarlarına yapıştırın. `gateway.auth.mode`, `"password"` olduğunda bunun yerine parola kimlik doğrulaması çalışır.

## Cihaz eşleştirme (ilk bağlantı)

Gateway kimlik doğrulaması başarılı olduktan sonra yeni bir tarayıcıdan veya cihazdan bağlanmak genellikle `disconnected (1008): pairing required` olarak gösterilen **tek seferlik bir eşleştirme onayı** gerektirir.

<Warning>
Kullanımdan kaldırılmış
`gateway.controlUi.dangerouslyDisableDeviceAuth=true` acil durum ayarını kullanan bir sürümden doğrudan yükseltme yapılırken
OpenClaw, belirteç/parola veya güvenilir proxy ile kimliği doğrulanan Control UI erişimini
yalnızca eşleştirme düzeltmesi için kullanılabilir durumda tutar. Tarayıcı düz HTTP üzerindeyse ve cihaz kimliği oluşturamıyorsa
önce HTTPS veya localhost üzerinden yeniden açın. Ardından uyarı bandında **Bu tarayıcıyı güvenli hâle getir** seçeneğine tıklayın.
Gateway yalnızca imzalı bir tarayıcı açıkça eşleştirildikten
sonra normal cihaz kimlik doğrulaması uygulamasına döner; cihaz kimliği olmayan bir tarayıcı için hiçbir zaman
kimlik oluşturmaz veya onaylamaz. Başka bir operatör cihazı zaten eşleştirilmişse
geçiş kullanılamaz. Gateway başlangıcı ve
`openclaw doctor --fix`, eski anahtarı sessizce
yok saymak yerine bu geçişi açıkça bildirir.
</Warning>

<Steps>
  <Step title="Bekleyen istekleri listeleyin">
    ```bash
    openclaw devices list
    ```
  </Step>
  <Step title="İstek kimliğine göre onaylayın">
    ```bash
    openclaw devices approve <requestId>
    ```
  </Step>
</Steps>

Tarayıcı, değiştirilmiş kimlik doğrulama ayrıntılarıyla (rol/kapsamlar/açık anahtar) eşleştirmeyi yeniden denerse önceki bekleyen isteğin yerini yenisi alır ve yeni bir `requestId` oluşturulur; onaylamadan önce `openclaw devices list` komutunu yeniden çalıştırın.

Önceden eşleştirilmiş uzak bir tarayıcının okuma erişiminden yazma/yönetici erişimine geçirilmesi, sessiz bir yeniden bağlantı değil, onay yükseltmesi olarak değerlendirilir: OpenClaw eski onayı etkin tutar, daha geniş kapsamlı yeniden bağlantıyı engeller ve yeni kapsam kümesini açıkça onaylamanızı ister. Gerekli koşulları karşılayan doğrudan geri döngü Control UI bağlantısı, kimliği doğrulandıktan sonra yükseltmeyi sessizce onaylayabilir.

Onaylandıktan sonra cihaz hatırlanır ve `openclaw devices revoke --device <id> --role <role>` ile iptal etmediğiniz sürece yeniden onay gerektirmez. Belirteç döndürme, iptal ve Paperclip / `openclaw_gateway` ilk çalıştırma onay akışı için [Cihazlar CLI'sine](/tr/cli/devices) bakın.

<Note>
- İletilen/proxy üstbilgileri olmayan bir geri döngü TCP eşinden (`127.0.0.1` veya `::1`, genellikle `localhost` olarak erişilir) gelen doğrudan yerel Control UI bağlantıları, yalnızca Gateway kimlik doğrulaması başarılı olduktan ve tarayıcı cihaz kimliğini sunduktan sonra cihaz eşleştirmesini otomatik olarak onaylayabilir. Belirteç/parola modunda ilk bağlantı yine de yapılandırılmış paylaşılan gizli anahtarı gerektirir; bu otomatik onay, belirteci atlama yöntemi değildir.
- Doğrudan geri döngü yalnızca `gateway.auth.mode: "none"` açıkça yapılandırıldığında paylaşılan gizli anahtar gerektirmez. Bu, Gateway kimlik doğrulamasını devre dışı bırakır ve önerilen Control UI kurulumu değildir. Tailscale Serve ve güvenilir proxy modları, yalnızca ilgili kimlik denetimleri başarılı olduğunda paylaşılan gizli anahtarın yapıştırılmasını gerektirmeyebilir.
- Tailscale Serve; `gateway.auth.allowTailscale: true` olduğunda, Tailscale kimliği doğrulandığında ve tarayıcı cihaz kimliğini sunduğunda Control UI operatör oturumları için eşleştirme gidiş dönüşünü atlayabilir. Cihaz kimliği olmayan tarayıcılar ve Node rolü bağlantıları normal cihaz denetimlerini izlemeye devam eder.
- Doğrudan Tailnet bağlamaları ve LAN tarayıcı bağlantıları yine açık onay gerektirir. Cihaz kimliği olmayan tarayıcı profilleri geri döngü otomatik onayını kullanamaz.
- Her tarayıcı profili benzersiz bir cihaz kimliği oluşturur; bu nedenle tarayıcı değiştirmek veya tarayıcı verilerini temizlemek yeniden eşleştirme gerektirir.

</Note>

## Bir mobil cihazı eşleştirme

Önceden eşleştirilmiş bir yönetici, terminal açmadan iOS/Android bağlantı QR kodunu oluşturabilir:

<Steps>
  <Step title="Mobil eşleştirmeyi açın">
    **Devices** seçeneğini belirleyin, ardından **Devices** kartında **Pair mobile device** seçeneğine tıklayın.
  </Step>
  <Step title="Telefonu bağlayın">
    OpenClaw mobil uygulamasında **Settings** → **Gateway** bölümünü açın ve QR kodunu tarayın. Bunun yerine kurulum kodunu kopyalayıp yapıştırabilirsiniz.
  </Step>
  <Step title="Bağlantıyı onaylayın">
    Resmî iOS/Android uygulaması otomatik olarak bağlanır. **Pending approval** bir istek gösteriyorsa onaylamadan önce rolünü ve kapsamlarını inceleyin.
  </Step>
</Steps>

Kurulum kodu oluşturmak `operator.admin` gerektirir; buna sahip olmayan oturumlarda düğme devre dışıdır. Kurulum kodu kısa ömürlü bir önyükleme kimlik bilgisi içerir; bu nedenle QR kodunu ve kopyalanan kodu geçerli oldukları sürece parola gibi değerlendirin. Uzak eşleştirme için Gateway, `wss://` olarak çözümlenmelidir (örneğin Tailscale Serve/Funnel üzerinden); düz `ws://`, geri döngü ve özel LAN adresleriyle sınırlıdır. Tüm güvenlik ve geri dönüş ayrıntıları için [Eşleştirme](/tr/channels/pairing#pair-from-the-control-ui-recommended) bölümüne bakın.

## Kişisel kimlik (tarayıcıya özel)

Control UI, paylaşılan oturumlarda ilişkilendirme amacıyla giden iletilere eklenen, tarayıcı başına kişisel kimliği (görünen ad ve avatar) destekler. Tarayıcı depolama alanında bulunur, geçerli tarayıcı profiliyle sınırlıdır ve diğer cihazlarla eşitlenmez ya da gönderdiğiniz iletilerdeki normal transkript yazarlık meta verileri dışında sunucu tarafında kalıcı olarak saklanmaz. Site verilerinin temizlenmesi veya tarayıcı değiştirilmesi kimliği boş duruma sıfırlar.

Asistan avatarı geçersiz kılma seçeneği de aynı tarayıcıya özel modeli izler: yüklenen geçersiz kılmalar, Gateway tarafından çözümlenen kimliğin üzerine yerel olarak uygulanır ve hiçbir zaman `config.patch` üzerinden gidiş dönüş yapmaz. Paylaşılan `ui.assistant.avatar` yapılandırma alanı, alana doğrudan yazan UI dışı istemciler için kullanılabilir olmaya devam eder.

## Çalışma zamanı yapılandırma uç noktası

Control UI, çalışma zamanı ayarlarını Gateway'in Control UI temel yoluna göre çözümlenen `/control-ui-config.json` adresinden alır (örneğin `/__openclaw__/` temel yolu altında `/__openclaw__/control-ui-config.json`). Bu uç nokta, HTTP yüzeyinin geri kalanıyla aynı Gateway kimlik doğrulamasıyla korunur: kimliği doğrulanmamış tarayıcılar bunu alamaz ve başarılı bir alma işlemi için geçerli bir Gateway belirteci/parolası, Tailscale Serve kimliği veya güvenilir proxy kimliği gerekir.

## Gateway ana bilgisayar durumu

Gateway makinesi, LAN adresi, işletim sistemi, çalışma zamanı, çalışma süresi, CPU yükü, bellek ve durum birimi disk alanını içeren **Gateway Host** kartını görmek için **Settings → General** bölümünü açın. Kart, görünür olduğu sürece `operator.read` kapsamını gerektiren `system.info` Gateway RPC üzerinden her 10 saniyede bir yenilenir. Eski Gateway sürümleri ve bu kapsama sahip olmayan bağlantılar kartı göstermez.

## Dil desteği

Control UI, ilk yüklemede tarayıcı yerel ayarınıza göre kendini yerelleştirir. Bunu daha sonra geçersiz kılmak için **Settings -> General -> Language** bölümünü açın (seçici Appearance altında değil, General sayfasındadır).

- Desteklenen yerel ayarlar: `en`, `ar`, `de`, `es`, `fa`, `fr`, `hi`, `id`, `it`, `ja-JP`, `ko`, `nl`, `pl`, `pt-BR`, `ru`, `th`, `tr`, `uk`, `vi`, `zh-CN`, `zh-TW`
- İngilizce dışındaki çeviriler tarayıcıda tembel yüklenir.
- Seçilen yerel ayar tarayıcı depolama alanına kaydedilir ve sonraki ziyaretlerde yeniden kullanılır.
- Eksik çeviri anahtarları için İngilizce kullanılır.

Belge çevirileri aynı İngilizce dışı yerel ayar kümesi için oluşturulur ancak belge sitesinin yerleşik Mintlify dil seçicisi yalnızca Mintlify'ın kabul ettiği yerel ayar kodlarını listeler. Tayca (`th`) ve Farsça (`fa`) belgeleri yayın deposunda yine oluşturulur; Mintlify bu kodları destekleyene kadar söz konusu seçicide görünmeyebilirler.

## Görünüm temaları

Appearance panelinde yerleşik Claw, Knot ve Dash temaları (varsayılan Claw'dır) ile tarayıcıya özel bir tweakcn içe aktarma yuvası bulunur. Bir temayı içe aktarmak için [tweakcn düzenleyicisini](https://tweakcn.com/editor/theme) açın, bir tema seçin veya oluşturun, **Share** seçeneğine tıklayın ve kopyalanan bağlantıyı Appearance bölümüne yapıştırın. İçe aktarıcı ayrıca `https://tweakcn.com/r/themes/<id>` kayıt defteri URL'lerini, `https://tweakcn.com/editor/theme?theme=amethyst-haze` gibi düzenleyici URL'lerini, göreli `/themes/<id>` yollarını, ham tema kimliklerini ve `amethyst-haze` gibi varsayılan tema adlarını kabul eder.

İçe aktarılan temalar yalnızca geçerli tarayıcı profilinde saklanır; Gateway yapılandırmasına yazılmaz ve cihazlar arasında eşitlenmez. İçe aktarılan temanın değiştirilmesi tek yerel yuvayı günceller; içe aktarılan tema etkinken bu yuvanın temizlenmesi Claw temasına geri döner.

Appearance bölümünde ayrıca bir Metin boyutu ayarı bulunur. Bu ayar sohbet metnine, oluşturucu metnine, araç kartlarına ve sohbet kenar çubuklarına uygulanır; ayrıca mobil Safari'nin odaklanıldığında otomatik yakınlaştırma yapmaması için metin girişlerini en az 16px boyutunda tutar.

Tema, tema modu, metin boyutu, dil ve sohbet görüntüleme tercihleri Gateway yapılandırması (`ui.prefs`) üzerinden eşitlenir; böylece cihazlar arasında sizi takip eder ve aracılar bunları onay geçidi üzerinden değiştirebilir — bağlı istemciler, Gateway'in `config.changed` bildirimi aracılığıyla değişiklikleri anında uygular. Her tarayıcı, anında başlatma için yerel bir kopya tutar; yapılandırmaya yazamayan istemciler (görüntüleyici kapsamı, çevrimdışı) değişiklikleri yalnızca cihazda tutar. Bkz. [Yapılandırma başvurusu](/tr/gateway/configuration-reference#ui).

## OpenClaw sistem bakımı

Sistem kurulum ve onarım aracısıyla konuşmak için **Settings → Ask OpenClaw** öğesini açın. İlk katılım dışında bu sayfa, ziyaret başına en fazla bir kapatılabilir olay etiketi gösterebilir. Rutin Gateway trafiğinde sessiz kalır ve yalnızca devre dışı bir yapılandırma yeniden yükleyicisi, yapılandırılmış bir kanal bağlantısının kesilmesi/bozulması, başarısız bir kanal yoklaması veya kullanılamayan kanal kimlik bilgileri bildiren durum anlık görüntülerine tepki verir. Daha yeni bir olay, yalnızca daha ciddi olduğunda bekleyen etiketin yerini alır; etiketi kapatmak veya kullanmak, o ziyaret boyunca olay istemlerini susturur. Etikete tıklamak tanılama sorusunu gerçek bir `openclaw.chat` mesajı olarak gönderir; böylece transkript isteği kaydeder ve OpenClaw tanılamayı gerçekleştirir. İlk katılım sırasında bu olay etiketleri hiçbir zaman gösterilmez.

## Pluginleri yönetme

Control UI'dan ayrılmadan pluginlere göz atmak ve onları yönetmek için kenar çubuğundaki **Pluginler** öğesini açın veya yapılandırılmış Control UI temel yoluna göre `/settings/plugins` yolunu kullanın. Örneğin, `/openclaw` temel yolu `/openclaw/settings/plugins` yolunu kullanır. Tüm isteğe bağlı pluginler devre dışı bırakılmış olsa bile sayfa her zaman kullanılabilir.

Pluginler, dört sekmeli bir merkezdir: **Yüklü** ve **Keşfet**, `/settings/plugins` konumundaki plugin kodunu yönetir; **Skills**, `/skills` konumundaki aracı başına beceri yöneticisini barındırır ve **Atölye**, `/skills/workshop` konumundaki Skill Workshop öneri incelemesini barındırır. Her sekme kendi URL'sini korur ve kenar çubuğu bunların tümü için tek bir Pluginler girdisi gösterir.

**Yüklü** sekmesi, genel sayımlarla birlikte kategoriye göre gruplandırılmış eksiksiz yerel envanteri gösterir. Her satır bir ayrıntı görünümü açar; taşma (`…`) menüsü plugini etkinleştirir veya devre dışı bırakır ve haricen yüklenmiş pluginler için **Kaldır** seçeneğini sunar. Ayrıca yapılandırılmış [MCP sunucularını](/tr/cli/mcp) listeler ve bunların satır içinde eklenmesini, devre dışı bırakılmasını ve kaldırılmasını destekler. Aynı sunucu denetimleri **Settings → MCP** altında bulunur. **Keşfet** sekmesi mağazadır: OpenClaw ile birlikte gelen öne çıkan pluginler, resmî haricî pluginler ve popüler hizmetler için tek tıklamalı MCP bağlayıcıları. Arama kutusuna yazmak, [ClawHub](https://clawhub.ai/plugins) üzerinde satır içi sorgu yapar ve indirme sayıları ile kaynak doğrulama rozetlerini içeren bir **ClawHub'dan** bölümü ekler. Derin bağlantılar `/settings/plugins?tab=discover` ile doğrudan mağazayı hedefleyebilir.

**Skills** sekmesi, seçili aracı kapsamında beceri durum raporunu, etkinleştirme/devre dışı bırakma anahtarlarını, API anahtarı girişini ve satır içi ClawHub beceri aramasını tutar. **Atölye** sekmesi, [beceri önerileri](/tr/tools/skill-workshop) için Skill Workshop panosunu ve Bugün inceleme akışını tutar. **Beceri fikirleri bul**, önemli oturumlardan oluşan sınırlı bir aralığı en yeniden en eskiye doğru inceler ve tüm sonuçları bekleyen öneriler olarak bırakır. Panel, birikimli kapsamı gösterir; **Önceki çalışmaları tara**, kalıcı imleçten devam eder ve eski geçmiş tükendikten sonra **Yeni çalışmaları tara** olur. Manuel geçmiş incelemesi, özerk kendi kendine öğrenme devre dışıyken çalışır ve seçili aracının yapılandırılmış modelini kullanır.

Dâhil edilen pluginler Gateway'de zaten mevcuttur ve **Yükle** yerine **Etkinleştir** veya **Devre dışı bırak** gösterir. Örneğin Workboard, OpenClaw ile birlikte gelir ancak varsayılan olarak devre dışıdır; dolayısıyla eylemi **Etkinleştir** olur. Birlikte gelen pluginler kaldırılamaz, yalnızca devre dışı bırakılabilir.

Kataloğu okumak ve ClawHub'da arama yapmak `operator.read` gerektirir. Bir plugini yüklemek, etkinleştirmek, devre dışı bırakmak veya kaldırmak ve MCP sunucularını değiştirmek `operator.admin` gerektirir; bu eylemler salt okunur operatörler için devre dışı kalır.

ClawHub yüklemeleri Gateway üzerinden çalışır ve Gateway aracılı diğer yüklemelerle aynı güven, bütünlük ve plugin yükleme politikası denetimlerini korur. Plugin kodunu yüklemek veya kaldırmak Gateway'in yeniden başlatılmasını gerektirir. Yüklü bir plugini etkinleştirmek veya devre dışı bırakmak, plugin ve mevcut Gateway çalışma zamanı bunu desteklediğinde yeniden başlatma olmadan uygulanabilir; aksi hâlde UI, yeniden başlatma gerektiğini bildirir. OAuth destekli MCP bağlayıcıları, eklendikten sonra CLI'dan bir defalık `openclaw mcp login <name>` gerektirir.

Sayfa bilinçli olarak envanter, keşif, yükleme, etkinleştirme ve kaldırmaya odaklanır. İsteğe bağlı npm, git veya yerel yol kaynakları, güncellemeler ve gelişmiş plugin yapılandırması için [`openclaw plugins`](/tr/cli/plugins) kullanın.

## Uygulamalar ve uzantılar

Kenar çubuğundaki **Daha Fazla** menüsünden, komut paletinden veya kenar çubuğundaki aracı menüsünden (**Uygulamaları edinin**) **Uygulamalar** öğesini açın ya da yapılandırılmış Control UI temel yoluna göre `/apps` yolunu kullanın. Sayfa, tüm OpenClaw eşlikçi yüzeyleri için yükleme bağlantılarını toplar: [iOS](/tr/platforms/ios) ve [Android](/tr/platforms/android) uygulamaları, bunlarla birlikte gelen Apple Watch ve Wear OS eşlikçileri, [macOS](/tr/platforms/macos), [Windows](/tr/platforms/windows) ve [Linux](/tr/platforms/linux) masaüstü uygulamaları, [Chrome uzantısı](/tr/tools/chrome-extension), [ClawHub](https://clawhub.ai) içeren uygulama içi Pluginler merkezi ile Discord topluluğu ve belgeleri.

## Kenar çubuğunda gezinme

Kenar çubuğu her şeyi aracının etrafında düzenler. Üstteki kimlik satırı etkin aracıdır; altındaki **Sayfalar** bölümü, okunmamış veya çalışıyor durumunu gösteren rozetiyle aracının kesintisiz ana oturumu olan **Ana Sayfa** ile başlar ve ardından sabitlenmiş hedefler (varsayılan olarak **Otomasyonlar** ve **Pluginler**) gelir. Sayfalar başlığındaki özelleştirme denetimi, **Kullanım** ve pluginler tarafından sağlanan sekmeler dâhil diğer tüm hedefleri ve **Sabitlenmiş öğeleri düzenle** seçeneğini içeren bir menü açar; gezinme alanına sağ tıklamak sabitleme düzenleyicisini doğrudan açar. Aşağıdaki oturum listesi bölgelere ayrılır: aracının sohbet oturumları için **İş Parçacıkları** (ana oturum Ana Sayfa'nın arkasında kalır; oluşturduğu oturumlar burada üst düzey iş parçacıkları olarak görünür ve adlandırılmış iş parçacıkları tür öneki olmadan gösterilir), grup ve oda konuşmaları için **Gruplar** ve yönetilen bir worktree'ye veya yürütme Node'una bağlı oturumlar (`repo ⎇ branch` satırına ek olarak Node ana makinesini gösterir), ACP destekli düzenek oturumları ve Codex/Claude CLI katalogları için **Kodlama**. Kodlama ilk çalıştırmada daraltılmış olarak başlar ve tercihinizi hatırlar; daraltılmış başlığı gerçek sayıyı korur ve içerdiği oturumlar çalışırken bir çalışma göstergesi gösterir. Özel gruplar (oturum `category`) ve **Sabitlenmiş** satırları İş Parçacıkları'nın üzerinde yer alır ve bir oturumu özel bir gruba atamak her zaman otomatik bölge sınıflandırmasına üstün gelir. İş Parçacıkları başlığı sıralama denetimini (Oluşturulma veya Son güncelleme, Gruplandırma ölçütü ve Etkin, Arşivlenmiş veya Tümü için kalıcı bir **Durum** filtresi) ve Yeni oturum sayfasını açan **+** işaretini içerir. Arşivlenmiş satırlar, bir arşiv simgesiyle soluklaştırılmış olarak satır içinde kalır; okunmamış veya dikkat durumuna katkıda bulunmaz ve soy yükseltmenin dışında kalır. Bir oturumu açmak, satırları yeniden sıralamadan seçim vurgusunu taşır. Yakın zamanda alt çalıştırmaları bulunan üst oturumlar bir açma denetimi ve alt öğe sayısı gösterir; kenar çubuğundan ayrılmadan iç içe alt oturumları, canlı veya sonlandırılmış durumu ve çalışma zamanını incelemek için bunu genişletin. Bir alt öğeyi seçmek, sohbetini açar ve üst öğe yolunu otomatik olarak görünür kılar. Alt satırlar kök gruplandırmanın, sabitlemenin, sürüklemenin, çoklu seçimin ve sayfalamanın dışında kalır; daraltılmış bölgeler görünür sayfa bütçesini tüketmez. Son okunduklarından beri yeni etkinlik bulunan oturumlar okunmamış noktası gösterir ve bunlardan birini açmak onu okunmuş olarak işaretler. Bir aracı ayrıca kısa, süresi dolan bir durum satırı yayımlayabilir ve isteğe bağlı olarak seçilmiş kehribar renkli bir simgeyle dikkat isteyebilir; bu bildirim oturumu açtığınızda, sonraki mesajı gönderdiğinizde, açıkça temizlediğinizde veya TTL süresi dolduğunda temizlenir. Bulut çalışanı yaşam döngüsü durumları küre rozeti kullanır; yerel ve geri alınmış oturumlar yerleşim rozeti göstermez çünkü yerel yürütme varsayılandır. Her kök oturum satırında Sabitle/Sabitlemeyi kaldır, Okunmamış/okunmuş olarak işaretle, Yeniden adlandır, Çatalla, Gruba taşı (Yeni grup ve Gruptan kaldır dâhil), Arşivle veya Arşivden çıkar ve Sil seçeneklerini içeren bir bağlam menüsü (üç nokta düğmesi veya sağ tıklama) bulunur; dokunmatik düzenler doğrudan sabitleme ve menü denetimlerini görünür tutar. Cmd/Ctrl-tıklama kök satırların çoklu seçimini açıp kapatır ve Shift-tıklama seçimi görünür sıra boyunca genişletir; seçili bir satırda menüyü açmak, seçili her oturuma uygulanan toplu eylemleri (N tanesini okunmamış/okunmuş olarak işaretle, N tanesini gruba taşı, N tanesini arşivle, N tanesini sil) sunar ve toplu silme için tek bir onay ister. Bir kök oturumu sabitlemek için **Sabitlenmiş** üzerine veya taşımak için özel bir grup üzerine sürükleyin. Özel grup başlıkları daraltılabilir, genişletilebilir veya yeniden sıralamak için sürüklenebilir; grup adları ve sıraları Gateway'de (`sessions.groups.*`) tutulur, böylece tarayıcılar arasında sizi takip eder; daraltılmış durum ise tarayıcı profilinde kalır. Grup başlıklarında ayrıca Grubu yeniden adlandır, Yeni grup ve Grubu sil seçeneklerini içeren bir menü (üç nokta düğmesi veya sağ tıklama) bulunur; bir grubu yeniden adlandırmak veya silmek, arşivlenmiş olanlar dâhil tüm üye oturumlarını sunucu tarafında günceller. Bir grubu silmek oturumlarını korur ve onları İş Parçacıkları'na geri taşır.

## Yeni oturum sayfası

Kenar çubuğundaki oturum listesi başlığında bulunan **+**, `/new` konumunda tam sayfa bir taslak açar: ilk mesajı gönderene kadar hiçbir şey oluşturulmaz. Birleşik **Yer** seçici, çalışma klasörünü ve yönetici operatörler için yürütme hedefini seçer: **Gateway · yerel**, `system.run` sunan eşleştirilmiş bir Node veya kullanılabilir bir bulut profili. Klasör varsayılan olarak aracı çalışma alanıdır; başka bir mutlak Gateway yolu `operator.admin` gerektirir ancak Git checkout'u olmadan doğrudan çalışabilir. Seçili Gateway klasörü bir Git checkout'u olduğunda aynı seçici, `worktrees.branches` tarafından desteklenen bir temel dal seçici (fetch yapılmaz) ve isteğe bağlı bir worktree adıyla (dal `openclaw/<name>` olur) isteğe bağlı **Worktree** yalıtımı sunar. Bulut çalışanları bu yönetilen worktree yolunu gerektirir; eşleştirilmiş Node'lar bunu hiçbir zaman sunmaz. Düzenleyici altbilgisi, yeni oturumun modelini ve akıl yürütme düzeyini seçer. **Gizli** anahtarı, oturum girdisi, transkripti ve Compaction durumu Gateway yeniden başlatılana kadar bellekte kalan, yalnızca web'e özgü bir iş parçacığı oluşturur; OpenClaw ayrıca otomatik bellek boşaltmasını atlar. Aracı normal araçlarını korur; bu nedenle açık bir kaydetme isteği veya araç güdümlü dosya yazımı yine de verileri kalıcılaştırabilir. Model sağlayıcısı mesajları işlemeye devam eder ve içerik içermeyen denetim meta verileri yine kaydedilir. Bulut başlatmaları, oturumu çalışanına göndermeden önce model ve akıl yürütme tercihlerini kalıcılaştırır.

Çok kullanıcılı Gateway'lerde yalnızca yönetici kapsamlı bağlantılar gizli iş parçacıkları oluşturabilir veya görüntüleyebilir ve diğer oturumlar, aracı oturum araçları veya transkript araması aracılığıyla bunlara erişemez. Gizli mod, depolamaya ve Gateway aracılı diğer kullanıcılara karşı koruma sağlar; canlı oturumları her zaman gözlemleyebilen Gateway sahibi veya işlem operatörüne karşı koruma sağlamaz.

**Klasörlere göz at**, Yer seçicinin satır içi dizin tarayıcısını açar; bu tarayıcı yalnızca yöneticilere açık `fs.listDir` yöntemi tarafından desteklenir ve seçili Gateway veya Node ile sınırlıdır. Gateway ve göz atma özelliğine sahip Node'lar dosya sistemlerini listeler; `fs.listDir` olmadan yürütme özelliğine sahip bir Node yine de yazılan mutlak yolu kabul eder. Son kullanılan yerler, yolları ana makineler arasında taşımadan bir klasörü ve sahibi olan Node'u birlikte geri yükleyebilir. Gönderme işlemi, ilk mesajla birlikte `sessions.create` çağrısı yapar; böylece çalıştırma aynı gidiş dönüşte başlar ve UI yeni oturumun sohbetine geçer. Gateway oturumu oluşturur ancak ilk gönderimi reddederse sohbet, istemi ve hatayı yeniden yüklemeler arasında korur; **Yeniden dene**, başka bir oturum oluşturmak yerine mesajı zaten oluşturulmuş oturum üzerinden gönderir.

**Settings** içinde, özel kenar çubuğu **Ask OpenClaw** öğesini içerir ve ayar bölümlerini hızla bulmak için bir **Ayarları ara** alanıyla başlar.

Masaüstü web'de, içerik alanının sol üstünde sabit bir denetim kümesi — macOS başlık çubuğu şeridinin web karşılığı — kenar çubuğunu daraltma düğmesini (⌘B) ve komut paleti arama düğmesini (⌘K) barındırır. Kenar çubuğunun üst kısmındaki ajan kimliği satırına tıklamak ajan menüsünü açar; **Ana Sayfa** ana oturumu açar. Bir şey işlem gerektirdiğinde — başarısız veya süresi geçmiş Cron işleri, süresi dolmak üzere olan veya dolmuş model kimlik doğrulaması — kenar çubuğu alt bilgisinin üzerinde kompakt dikkat göstergeleri belirir ve tıklandığında ilgili sayfayı açar. Kimlik satırı ajanın avatarını (kimlik görseli veya emoji), adını, bağlantı noktasını ve canlı bir alt başlığı gösterir. Ajan kapsamlı menüsü satır içi ajan değiştiriciyi (çok ajanlı kurulumlar), **Yeni ajan**, "Bu ajan ne yapabilir?" ve **Ajan ayarları** seçeneklerini içerir. Ondan fazla ajan içeren listelerde bir filtre alanı bulunur ve sabitlenmiş ajanlar önce listelenir; ajanları Ajanlar ayarları sayfasından sabitleyin veya sabitlemesini kaldırın; sabitlenen küme tarayıcı profilinde saklanır. Bir ajan seçildiğinde Sohbet'in yanı sıra Kullanım, Otomasyonlar, Görevler, Çalışma Panosu ve Oturumlar da o ajanla sınırlandırılır. Kapsamlı her sayfa, kapsamdan çıkmak için **Tüm ajanlar** seçeneğini içeren bir **Ajan** denetimi sunar; bu, somut sohbet ajanını değiştirmeden paylaşılan sayfa kapsamını genişletir, doğrudan oturum bağlantıları ise hedeflerini açmaya devam eder. Ajanlar ayarları sayfası kendi `?agent=` seçimini korur ve paylaşılan sayfa kapsamını izlemez. Alt bilgi, çevrimdışıyken de kullanılabilen ve bilinen son hesap adının altında **Yeniden bağlanılıyor…** ifadesini gösteren tam genişlikte tek bir kimlik kartıdır. Bu kart, profil kimliği başlığının ardından **Ayarlar**, **Kullanım**, mobil eşleştirme, **Uygulamaları edinin**, **Yardım** (yardım, Discord, Belgeler ve değişiklik günlüğü), gerektiğinde çevrimdışı yeniden deneme işlemi, sürüm/derleme göstergesi ve renk modu düğmesinin yer aldığı uygulama/hesap menüsünü açar. Derleme göstergesi Hakkında sayfasını açar. Gateway, kaynak kod deposundan `main` dışındaki bir dalda çalıştırıldığında, sürüm dışı bir Gateway'in ilk bakışta anlaşılabilmesi için alt bilgi bu dal adını kırmızı renkte de gösterir (sürüm kurulumlarında hiçbir zaman gösterilmez). Apple platformlarında Shift-Command-Comma, diğer platformlarda ise Ctrl-Shift-Comma, tarayıcının yalın Command-Comma kısayolunu geçersiz kılmadan **Ayarlar**'ı açar. Kenar çubuğunu daraltmak (⌘B veya kümedeki düğme) tam genişlikte bir çalışma alanı için çubuğu tamamen gizler; daraltılmış durumdayken sol üst küme genişletme düğmesini ve aramayı korur, ayrıca macOS uygulamasının başlık çubuğunda yerel olarak barındırdıklarını yansıtan yeni bir ileti dizisi düğmesi kazanır. Kenar çubuğu, masaüstündeki tek gezinme arayüzüdür; üst çubuk yoktur. Dar görüntü alanlarında kenar çubuğunun yerini, çekmece düğmesini, markayı ve komut paleti aramasını içeren kompakt bir başlık satırının arkasındaki kayar çekmece alır; telefonlarda Sohbet bu gezinme satırını kendi başlık çubuğuna alır ve menü ile arama denetimleri oturum başlığının yanında yer alır. macOS uygulamasında ayrı başlık satırı, başlık çubuğu boşluğunu pencere denetimlerinin yanındaki tek bir kompakt şeritte birleştirir. Gezinme normal tarayıcı geçmişini kullanır; böylece tarayıcının geri/ileri düğmeleri geçmişte ilerler. macOS uygulaması ayrıca pencere denetimlerinin yanına yerel bir kenar çubuğu düğmesi ve izleme dörtgeni kaydırma hareketleri ekler; kenar çubuğu genişletilmişken sağ kenarında geri/ileri düğmeleri, daraltılmışken ise yerel arama (komut paleti) ve yeni oturum düğmeleri bulunur.

Bekleyen onaylar da kenar çubuğu alt bilgisinin üzerinde bir dikkat göstergesi oluşturur;
ilgili Onaylar sayfasını açmak için bunu seçin.

## Neler yapabilir (bugün)

<AccordionGroup>
  <Accordion title="Sohbet ve Konuşma">
    - Gateway WS üzerinden modelle sohbet edin (`chat.history`, `chat.send`, `chat.abort`, `chat.inject`). Arşivlenmiş oturumlarda düzenleyici devre dışı kalır ve konuşmanın devam edebilmesi için **Arşivden çıkar** işlemini içeren bir başlık gösterilir.
    - Sohbet geçmişi yenilemeleri, ileti başına metin sınırları olan sınırlı bir yakın zaman aralığı ister; böylece büyük oturumlar, sohbet kullanılabilir hâle gelmeden önce tarayıcıyı tam bir döküm yükünü oluşturmaya zorlamaz.
    - Herkese açık bir GitHub sorun veya pull request bağlantısının üzerine gelindiğinde ya da klavyeyle odaklanıldığında bağlantının durumu, başlığı, yazarı, son etkinliği, yorumları ve değişiklik istatistikleri gösterilir. Bağlı Gateway, kullanıcı arayüzü uzak bir Gateway kullandığında da bağlantı hedefini değiştirmeden herkese açık meta verileri getirir ve önbelleğe alır. Gateway, deponun herkese açık olduğunu doğruladıktan sonra mevcutsa `GH_TOKEN` veya `GITHUB_TOKEN` kullanır; aksi takdirde GitHub'ın anonim API'sini daha uzun bir önbellekle kullanır.
    - Tarayıcıdaki gerçek zamanlı oturumlar üzerinden konuşun. OpenAI doğrudan WebRTC kullanır, Google Live WebSocket üzerinden kısıtlı ve tek kullanımlık bir tarayıcı belirteci kullanır, yalnızca arka uçta çalışan gerçek zamanlı ses Plugin'leri ise Gateway aktarma taşımasını kullanır. Video özellikli tarayıcı oturumları Ayarlar'dan cihazdaki bir kamerayı seçebilir veya canlı önizlemeden kameralar arasında geçiş yapabilir; tarayıcı, kamera videosunu Gateway üzerinden yayımlamadan gerçek zamanlı sağlayıcı için JPEG kareleri yakalar. İstemcinin sahip olduğu sağlayıcı oturumları `talk.client.create` ile; Gateway aktarma oturumları ise `talk.session.create` ile başlar. Aktarma, sağlayıcı kimlik bilgilerini Gateway'de tutarken tarayıcı mikrofon PCM verisini `talk.session.appendAudio` üzerinden yayımlar, `openclaw_agent_consult` sağlayıcı araç çağrılarını Gateway politikası ve yapılandırılmış daha büyük OpenClaw modeli için `talk.client.toolCall` üzerinden iletir ve etkin çalıştırmadaki sesli yönlendirmeyi `talk.client.steer` veya `talk.session.steer` üzerinden yönlendirir.
    - Sohbet'te araç çağrılarını ve canlı araç çıktısı kartlarını (ajan olayları) akış hâlinde görüntüleyin. Araç etkinliği, türe duyarlı satırlar hâlinde oluşturulur: kabuk komutları, sözdizimi vurgulanmış komutu terminal tarzı çıktıyla gösterir; desteklenen düzenleme ve yazma çağrıları sınırlı satır içi farkları, mevcut olduğunda satır numaralarını ve `+added -removed` istatistiklerini gösterir; art arda gelen çağrılar ise "13 komut çalıştırıldı, 6 dosya okundu, 9 dosya düzenlendi" gibi bir özette daraltılır. Bir çalıştırma canlıyken en yeni çalışan çağrı grubun başlığını belirler. Kalan bağımsız değişkenleri ve ham çıktıyı incelemek için bir satırı genişletin.
    - Karmaşık araç çağrıları (uzun kabuk komutları, çok sayıda bağımsız değişken içeren Plugin araçları) için isteğe bağlı yapay zekâ amaç başlıkları; `gateway.controlUi.toolTitles: true` ile etkinleştirilir (varsayılan olarak kapalıdır). Başlıklar, standart yardımcı model yönlendirmesi üzerinden toplu `chat.toolTitles` yönteminden gelir — açıkça belirtilmiş bir `utilityModel` (diğer yardımcı görevlerde olduğu gibi operatör tarafından seçilen sağlayıcı) veya bunun yokluğunda oturum sağlayıcısının bildirdiği küçük model varsayılanı — ve Gateway tarafında ajan başına önbelleğe alınır. Katılım seçeneği kapalı olduğunda veya kullanılabilir düşük maliyetli bir model bulunmadığında satırlar belirlenimci etiketlerini korur ve model çağrısı yapılmaz.
    - Modelin önerdiği geçici takip görevlerini başlatın veya kapatın; kabul edilen öneriler, önerilen istemi içeren yeni bir yönetilen çalışma ağacı oturumu açar.
    - Mevcut `session.tool` / araç olayı teslimatından gelen canlı araç etkinliğinin tarayıcıya yerel, önce redaksiyon uygulanan özetlerini içeren Etkinlik sekmesi.

  </Accordion>
  <Accordion title="Kanallar, oturumlar, bellek">
    - Kanallar: yerleşik ve paketlenmiş/harici Plugin kanallarının durumu, QR ile oturum açma ve kanal başına yapılandırma (`channels.status`, `web.login.*`, `config.patch`).
    - Kanal yoklaması yenilemeleri, yavaş sağlayıcı kontrolleri tamamlanırken önceki anlık görüntüyü görünür tutar ve bir yoklama veya denetim kullanıcı arayüzü bütçesini aştığında kısmi anlık görüntüleri etiketler.
    - İleti Dizileri (`/sessions` adresindeki bir çalışma alanı sayfası; yanında bir **Çalışma ağaçları** sekmesi bulunur): varsayılan olarak yapılandırılmış ajan oturumlarını listeler, sık kullanılan oturumları sabitler, yeniden adlandırır, etkin olmayan oturumları arşivler veya geri yükler, eski yapılandırılmamış ajan oturumu anahtarlarından geri dönüş yapar ve oturum başına model/düşünme/hızlı/ayrıntılı/izleme/akıl yürütme geçersiz kılmalarını uygular (`sessions.list`, `sessions.patch`). Üç seçenekli **Etkin / Arşivlenmiş / Tümü** filtresi hem bu sayfayı hem de kenar çubuğunu denetler; Tümü seçeneği arşivlenmiş satırları soluklaştırır ve açıkça etiketler. Arşivlenmiş oturumlar dökümlerini korur, hiçbir zaman otomatik olarak budanmaz ve açıkça arşivden çıkarılana veya silinene kadar rafa kaldırılmış durumda kalır. Satırlar, son okunmalarından bu yana etkinlik gerçekleşen etkin oturumlarda okunmamış noktasını ve okunmadı/okundu olarak işaretleme işlemlerini gösterir (`sessions.patch { unread }`); Çatallandır işlemi ise dökümü yeni bir oturuma dallandırır (`sessions.create { parentSessionKey, fork: true }`). Tablonun üzerindeki genel bakış kutucukları yüklenen listeyi özetler (oturum sayısı, canlı çalıştırmalar, okunmamış oturumlar, toplam belirteçler ve mevcut olduğunda arşivlenmiş oturum sayısı); her satır canlı çalıştırma noktası içeren bir tür simgesi taşır, durum yalın bir nokta ve etiket olarak oluşturulur ve oturum belirteç ile bağlam boyutlarını bildirdiğinde Belirteçler sütunu bir bağlam penceresi kullanım ölçeri gösterir. Satır yönetimi işlemleri, kenar çubuğunun oturum menüsünü yansıtan satır başına menüde (üç nokta düğmesi veya sağ tıklama) bulunur; satır çekmecesi ise diğer oturum ayrıntılarının yanında ajan çalışma zamanını ve çalıştırma süresini içerir.
    - Yerel Claude ve Codex kenar çubuğu katalogları bir seferde tek bir ana makineden akış sağlar; ardından Node bağlantısı değişikliklerinden sonra, sayfaya odaklanıldığında ve görünür durumdayken en fazla her 30 saniyede bir uzlaştırılır. Katalog değişiklikleri daha hızlı bir takip geçişini tetikler; böylece yerel araçlarda oluşturulan oturumlar Control UI yeniden yüklenmeden görünür. Claude Desktop satırları, mevcut olduğunda yerel özel grup etiketlerini de korur; OpenClaw bu eşlemeyi Desktop'ın yerel deposundan okur ve hiçbir zaman yazmaz.
    - Oturum gruplandırma: Gruplandırma ölçütü denetimi, oturumlar tablosunu özel gruplara, kanala, türe, ajana veya tarihe göre bölümler hâlinde düzenler. Özel gruplar `sessions.patch` (`category`) aracılığıyla oturum başına kalıcı olur; böylece ileti kanallarından (Discord, Telegram, WhatsApp, ...) başlatılan oturumlar da kategorilere ayrılabilir. Satırları bir bölümün üzerine sürükleyerek veya satır başına grup seçiciyi kullanarak gruplar atayın; Yeni grup işlemiyle gruplar oluşturun.
    - Bellek (Ajanlar sayfasında, seçilen ajanla sınırlandırılmış bir sekme): Dreaming durumu, etkinleştirme/devre dışı bırakma düğmesi ve Rüya Günlüğü okuyucusu (`doctor.memory.status`, `doctor.memory.dreamDiary`, `config.patch`).
    - Belleği İçe Aktar (`/memory-import`, Ajanlar sayfasının Bellek sekmesinden erişilir): yerel Claude Code otomatik belleğini, Codex birleştirilmiş belleğini veya Hermes bellek dosyalarını önizleyip seçilen ajan çalışma alanına kopyalayın (`migrations.memory.plan`, `migrations.memory.apply`).
    - İlk katılım bellek teklifi: Control UI ilk katılım modunda açıldığında (`?onboarding=1`, Linux yardımcı uygulaması tarafından ilk çalıştırma kurulumundan sonra kullanılır), tek sayfalık bir iletişim kutusu algılanan bellekleri aynı planlama/uygulama akışıyla içe aktarmayı önerir; bu adımın atlanması, ayarlar sayfasını daha sonraki giriş noktası olarak bırakır.

  </Accordion>
  <Accordion title="Cron, görevler, plugin'ler, Skills, cihazlar, çalıştırma onayları">
    - Otomasyonlar (cron işleri): Otomasyonlar/Çalıştırma geçmişi sekme geçişinin üzerinde istatistik kartları (otomasyon sayısı, başarısız olanların sayısı, zamanlayıcı durumu, sonraki uyanma); Otomasyonlar sekmesi işleri filtrelenebilir bir tabloda (Tümü/Etkin/Duraklatıldı, arama, zamanlama ve son çalıştırma filtreleri, satır başına eylem menüsü) listeler ve altında başlangıç önerileri sunar; Çalıştırma geçmişi sekmesi ise tüm otomasyonlardaki son çalıştırmaları gösterir (`cron.*`).
    - Görevler: Bağlantılı oturumlar ve iptal özelliğiyle etkin ve yakın tarihli arka plan görevlerinin canlı kaydı (`tasks.*`). Sohbet'in Arka plan görevleri bölmesi, çalışan ve tamamlanan işleri gruplandırır; sınırlandırılmış istemini ve çıktısını ya da hata özetini incelemek için bir satır seçin.
    - Plugin'ler: Yüklü envantere ve seçilmiş mağazaya göz atın, ClawHub'da arayın, plugin kodunu yükleyip kaldırın ve yüklü plugin'leri etkinleştirin ya da devre dışı bırakın (`plugins.*`); MCP sunucusu satırları, yapılandırma yöntemleri aracılığıyla `mcp.servers` öğesini düzenler.
    - Skills: durum, etkinleştirme/devre dışı bırakma, yükleme, API anahtarı güncellemeleri (`skills.*`).
    - Cihazlar: Tek bir envanter; eşleştirilmiş cihaz kayıtlarını, Node kataloğunu ve canlı mevcudiyet durumunu bir araya getirir (`device.pair.list`, `node.list`, `system-presence`). Gateway ana makinesi ilk sıraya sabitlenir; eşleştirilmiş istemciler bağlantı durumunu, rolleri, token'ları, yetenekleri ve komutları gösterir. Yinelenen eşleştirmeler genişletilebilir bir grupta birleştirilir ve **Eski N kaydı temizle**, yönetici tarafından onaylanan; otomatik olarak onaylanmış (sessiz yerel, güvenilir CIDR veya SSH ile doğrulanmış) ya da onay kaynağı kaydından önce oluşturulmuş çevrimdışı yinelenen kayıtları topluca kaldırır. Girdiler kaldırılabilir (`node.pair.remove`, `device.pair.remove`), cihaz eşleştirme ve Node yeniden onayları satır içinde gerçekleştirilir (`device.pair.*`, `node.pair.approve`/`reject`) ve mobil kurulum kodları aynı karttan oluşturulur.
    - Çalıştırma onayları: `exec host=gateway/node` için Gateway veya Node izin listelerini ve sorma politikasını düzenleyin (`exec.approvals.*`).

  </Accordion>
  <Accordion title="Yapılandırma">
    - `~/.openclaw/openclaw.json` öğesini görüntüleyin/düzenleyin (`config.get`, `config.set`).
    - Ayarlar gezintisi Ask OpenClaw ile başlar, ardından sayfaları ilgi gereksinimine göre gruplandırır: üstte Genel, Görünüm ve Bildirimler; Bağlantılar (Bağlantı, Kanallar, İletişim, Cihazlar); Aracılar ve Araçlar (Aracılar, Yapay Zekâ ve Aracılar, Model Sağlayıcıları, MCP, Otomasyon, Laboratuvarlar); Gizlilik ve Güvenlik (Güvenlik, Onaylar); ve Sistem (Altyapı, Gelişmiş, Hata Ayıklama, Günlükler, Hakkında). Genel; model varsayılanları, dil ve Gateway ana makine istatistiklerini içeren sade bir merkezdir; diğer tüm ayarlar yalnızca tek bir sayfada bulunur.
    - Gizlilik ve Güvenlik: Şema destekli `security`/`approvals` bölümlerinin üzerinde Gateway kimlik doğrulaması, çalıştırma politikası, tarayıcı etkinleştirme, araç profili, cihaz kimlik doğrulaması ve mobil eşleştirme için seçilmiş satırlar.
    - Onaylar; çözümlenmiş çalıştırma, plugin ve sistem aracısı istekleri için en yeniden eskiye sıralanmış 30 günlük geçmişi içerir. Gateway tarafından kaydedilen kararı, nedeni, kaynak oturumu ve çözümleyici bilgisini incelemek için türe göre filtreleyin veya eski satırlar arasında sayfalama yapın.
    - Laboratuvarlar, yayımlanmış deneysel anahtarları sunar. Code Mode ve Swarm mevcut girdilerdir ve `tools.codeMode.enabled` ile `tools.swarm.enabled` değerlerini hemen kaydeder; yayımlanmamış deneyler görünmez veya varsayımsal yapılandırma anahtarları yazmaz.
    - Bildirimler: Tarayıcı web anlık bildirim durumu, abone olma/abonelikten çıkma ve test gönderimi.
    - Gelişmiş: Özel bir ana sayfası bulunmayan tüm yapılandırma bölümleri ve ham JSON5 düzenleyicisi (önceden Genel sayfasının Gelişmiş moduydu).
    - Model Kurulumu (`/settings/model-setup`), Model Sağlayıcıları'nın başlığından açılan bir alt sayfasıdır.
    - Aracılar: Aracı başına sekmeler (Genel Bakış, Dosyalar, Araçlar, Skills, Kanallar, Otomasyonlar, Bellek) içeren bir ayarlar sayfası (**Ayarlar → Aracılar**, `/settings/agents`). Genel Bakış sekmesi aracının kimliğini — görünen ad, emoji ve `agents.update` öncesinde tarayıcıda küçültülüp boyutu sınırlandırılan avatar görselini — düzenler. Kaydetme işlemi, yapılandırılmış kimlik alanlarını depolar ve bunları çalışma alanındaki `IDENTITY.md` dosyasına yansıtır; yapılandırılmış değerler, aynı dosya alanlarında yapılan manuel düzenlemelere göre önceliklidir.
    - Profil: Varsayılan aracının kimliğini tüm zamanlara ait kullanım istatistikleriyle — toplam token sayısı, en yoğun gün, en uzun oturum, etkinlik serileri, bir yıllık token ısı haritası, en çok kullanılan araçlar ve kanal öne çıkanları — gösteren bir ayarlar sayfası (`usage.cost`, `sessions.usage`).
    - MCP'nin; sunucu satırları (aktarım, etkinlik, OAuth/filtre/paralellik özetleri), doğrudan ekleme/etkinleştirme/devre dışı bırakma/kaldırma denetimleri, yaygın operatör komutları ve kapsamlı `mcp` yapılandırma düzenleyicisi içeren özel bir ayarlar sayfası vardır. Plugin'ler sayfası, tek tıklamalı bağlayıcıların ve keşfin ana sayfası olmaya devam eder.
    - Model Sağlayıcıları: Yapılandırılmış her model sağlayıcısını marka simgesi, kimlik doğrulama durumu (`models.authStatus`), model kullanılabilirliği (`models.list`), sağlayıcının bildirdiği yerlerde canlı plan/kota/faturalandırma verileri (`usage.status`) ve son 30 güne ait yerel oturum harcamalarıyla (`sessions.usage`) listeleyen bir ayarlar sayfası. Yenile eylemi, kimlik bilgisi durumunu ve sağlayıcı kullanımını yeniden okur.
    - Bağlantı: Panonun kendi Gateway bağlantısını — WebSocket URL'si, Gateway token'ı, parola ve varsayılan oturum anahtarı — ve en son el sıkışma anlık görüntüsünü (durum, çalışma süresi, tik aralığı, son kanal yenilemesi) yöneten, **Bağlantılar** altındaki bir ayarlar sayfası. Çevrimdışı oturum açma geçidi bağlantının kesik olduğu durumu yönetir; bu sayfa bağlantı kuruluyken bağlantıyı düzenler.
    - Doğrulamayla uygulayın ve yeniden başlatın (`config.apply`), ardından son etkin oturumu uyandırın.
    - Yazma işlemleri, eşzamanlı düzenlemelerin üzerine yazılmasını önlemek için temel karma koruması içerir.
    - Yazma işlemleri (`config.set`/`config.apply`/`config.patch`), gönderilen yapılandırma yükündeki başvurular için etkin SecretRef çözümlemesini önceden denetler; gönderilen ve çözümlenemeyen etkin başvurular yazma işleminden önce reddedilir.
    - Form kaydetme işlemleri, kaydedilmiş yapılandırmadan geri yüklenemeyen eski redakte edilmiş yer tutucuları atarken hâlâ kaydedilmiş gizli bilgilere eşlenen redakte edilmiş değerleri korur.
    - Şema ve form oluşturma; alan `title`/`description`, eşleşen kullanıcı arayüzü ipuçları, doğrudan alt öğe özetleri, iç içe nesne/joker karakter/dizi/bileşim düğümlerindeki dokümantasyon meta verileri ve kullanılabilir olduğunda plugin ile kanal şemaları dâhil olmak üzere `config.schema` / `config.schema.lookup` kaynaklarından gelir. Ham JSON düzenleyicisi yalnızca anlık görüntü güvenli bir ham gidiş dönüşe sahipse kullanılabilir; aksi hâlde Control UI Form modunu zorunlu kılar.
    - Ham JSON düzenleyicisindeki "Kaydedilene sıfırla", düzleştirilmiş bir anlık görüntüyü yeniden oluşturmak yerine ham olarak yazılmış biçimi (biçimlendirme, yorumlar, `$include` düzeni) korur; böylece anlık görüntü güvenli biçimde gidiş dönüş yapabildiğinde dış düzenlemeler sıfırlamadan sonra da korunur.
    - Yapılandırılmış SecretRef nesne değerleri, nesneden dizeye yanlışlıkla bozulmayı önlemek için form metin girişlerinde salt okunur olarak oluşturulur.

  </Accordion>
  <Accordion title="Kullanım">
    - Oturumdan türetilen token ve tahmini maliyet analizi, sağlayıcı faturalandırmasından ayrı kalır.
    - Sağlayıcı kartları `usage.status` çağrısını yapar ve yapılandırılmış sağlayıcı plugin'leri tarafından bildirilen canlı plan adlarını, kota dönemlerini, bakiyeleri, harcamaları ve bütçeleri gösterir.
    - Sağlayıcı kullanım hatası, oturum/maliyet panosunu engellemez; kullanılamayan sağlayıcı kartları kendi hata durumlarını gösterir.

  </Accordion>
  <Accordion title="Hata ayıklama, günlükler, güncelleme">
    - Hata ayıklama: durum/sağlık/modeller anlık görüntüleri, olay günlüğü ve manuel RPC çağrıları (`status`, `health`, `models.list`).
    - Olay günlüğü; Control UI yenileme/RPC sürelerini, yavaş sohbet/yapılandırma oluşturma sürelerini ve tarayıcı bu PerformanceObserver girdi türlerini sunduğunda uzun animasyon kareleri veya uzun görevler için tarayıcı yanıt verebilirlik girdilerini içerir.
    - Günlükler: Filtreleme/dışa aktarma özelliğiyle Gateway dosya günlüklerinin canlı akışı (`logs.tail`).
    - Güncelleme: Yeniden başlatma raporuyla birlikte paket/git güncellemesi ve yeniden başlatma işlemi (`update.run`) çalıştırın, ardından çalışan Gateway sürümünü doğrulamak için yeniden bağlantı sonrasında `update.status` öğesini yoklayın.

  </Accordion>
  <Accordion title="Otomasyonlar paneli notları">
    - Bir satır seçildiğinde, başlığında Etkin/Duraklatıldı anahtarı ve Şimdi çalıştır bulunan tam sayfa ayrıntı görünümü açılır (menüsünde zamanı geldiyse çalıştır, klonla ve kaldır seçenekleri bulunur); Ayarlar sekmesi otomasyonu satır içinde düzenler (istem, ayrıntılar, sıklık, gelişmiş geçersiz kılmalar) ve Çalıştırma geçmişi sekmesi bu otomasyonun çalıştırmalarını gösterir.
    - Tablonun altındaki başlangıç otomasyonları, oluşturma formunu düzenlenebilir bir istem ve zamanlamayla önceden doldurur.
    - Yalıtılmış görevlerde teslimat varsayılan olarak özeti duyurur; yalnızca dahili çalıştırmalar için hiçbiri seçeneğine geçin.
    - Duyuru seçildiğinde kanal/hedef alanları görünür.
    - Webhook modu, `delivery.to` geçerli bir HTTP(S) webhook URL'sine ayarlanmış şekilde `delivery.mode = "webhook"` kullanır.
    - Ana oturum görevlerinde webhook ve teslimat yok modları kullanılabilir.
    - Gelişmiş düzenleme denetimleri; çalıştırma sonrasında silme, aracı geçersiz kılmasını temizleme, cron kesin/kademeli seçenekleri, aracı modeli/düşünme geçersiz kılmaları ve mümkün olan en iyi teslimat anahtarlarını içerir.
    - Form doğrulaması alan düzeyindeki hatalarla satır içinde yapılır; geçersiz değerler düzeltilene kadar kaydetme düğmesini devre dışı bırakır.
    - Özel bir bearer token göndermek için `cron.webhookToken` ayarını yapın; atlanırsa webhook kimlik doğrulama üstbilgisi olmadan gönderilir.
    - `cron.webhook`, mevcut yapılandırma doğrulaması tarafından reddedilen kullanımdan kaldırılmış eski bir geri dönüş mekanizmasıdır. Hâlâ `notify: true` kullanan depolanmış işleri açık iş başına webhook veya tamamlanma teslimatına taşımak ve eski anahtarı kaldırmak için `openclaw doctor --fix` çalıştırın.

  </Accordion>
</AccordionGroup>

## Yardımcı belleğini içe aktarma

Yerel Codex veya Claude Code belleğini bir OpenClaw aracısına aktarmak için **Ayarlar** → **Belleği İçe Aktar** bölümünü açın. Gateway, desteklenen yerel belleği kendi ana makinesinde keşfeder; bu nedenle uzaktaki bir Control UI, tarayıcı bilgisayarı yerine Gateway bilgisayarından içe aktarır.

1. Hedef aracıyı seçin.
2. Algılanan kaynak koleksiyonlarını ve Markdown dosya adlarını inceleyin. Dosya içerikleri
   plan yanıtında gönderilmez veya sayfada görüntülenmez.
3. İçe aktarılacak koleksiyonları seçip onaylayın. Uygulama işlemi, yazmadan önce planı yeniden
   oluşturur; böylece eski seçimler güvenli biçimde başarısız olur.
4. Dosyalar zaten varsa **Mevcut içe aktarımları değiştir** seçeneğini etkinleştirin, önizlemeyi
   yenileyin ve değiştirme işlemini onaylayın.

Codex yalnızca birleştirilmiş `MEMORY.md` ve `memory_summary.md` öğelerini içe aktarır. Claude
Code, proje otomatik bellek dizinlerinden ve yapılandırılmış bir
`autoMemoryDirectory` konumundan Markdown içe aktarır; bu sayfa aracılığıyla oturumları, ayarları, talimatları veya
kimlik bilgilerini içe aktarmaz. Dosyalar, seçilen çalışma alanındaki `memory/imports/` altına
kopyalanır; etkin bellek plugin'i burada bunları dizine ekleyebilir. Kaynaklar
asla değiştirilmez.

Planlama ve uygulama için `operator.admin` gerekir. Her uygulama, durum mevcut olduğunda doğrulanmış bir
OpenClaw yedeği oluşturur, redakte edilmiş bir taşıma raporu yazar ve mevcut hedef dosyaları değiştirmeden önce
öğe düzeyinde yedekler tutar. Yollar ve
anımsama davranışı için [Belleğe genel bakış](/tr/concepts/memory#import-from-coding-assistants) sayfasına bakın.

## MCP sayfası

Özel MCP sayfası, `mcp.servers` altındaki OpenClaw tarafından yönetilen MCP sunucuları için bir operatör görünümüdür. MCP aktarımlarını kendisi başlatmaz; kayıtlı yapılandırmayı incelemek ve düzenlemek için bu sayfayı, canlı sunucu kanıtına ihtiyaç duyduğunuzda ise `openclaw mcp doctor --probe` öğesini kullanın.

Tipik iş akışı:

1. Kenar çubuğundan **MCP**'yi açın.
2. Toplam, etkin, OAuth ve filtrelenmiş sunucu sayılarını görmek için özet kartlarını kontrol edin.
3. Her sunucu satırında aktarımı, etkinleştirme durumunu, kimlik doğrulamayı, filtreleri, zaman aşımlarını ve komut ipuçlarını inceleyin.
4. Sunucuları doğrudan MCP sayfasından ekleyin, etkinleştirin, devre dışı bırakın veya kaldırın. Streamable HTTP, SSE ya da stdio seçeneklerinden birini açıkça seçin; stdio komut satırları, boşluk içeren yollar gibi tırnak içine alınmış bağımsız değişkenleri kabul eder. Tek tıklamalı bağlayıcılar ve keşif için **Plugins** sayfasını kullanın.
5. Ortam değişkenleri, çalışma dizinleri, üstbilgiler, TLS/mTLS yolları, OAuth meta verileri, araç filtreleri ve Codex projeksiyon meta verileri gibi gelişmiş sunucu alanları için kapsamlı `mcp` yapılandırma bölümünü düzenleyin.
6. Yapılandırmayı yazmak için **Save** seçeneğini, çalışan Gateway'in değiştirilen yapılandırmayı uygulaması gerektiğinde ise **Save & Publish** seçeneğini kullanın.
7. Statik tanılama, canlı doğrulama veya önbelleğe alınmış çalışma zamanını kaldırma işlemleri için bir terminalden `openclaw mcp status --verbose`, `openclaw mcp doctor --probe` ya da `openclaw mcp reload` komutunu çalıştırın.

Sayfa, görüntülemeden önce kimlik bilgileri içeren URL benzeri değerleri sansürler ve kopyalanan komutların boşluklar veya kabuk meta karakterleriyle de çalışması için komut parçacıklarında sunucu adlarını tırnak içine alır. Tam CLI ve yapılandırma başvurusu: [MCP](/tr/cli/mcp).

## Etkinlik sekmesi

Etkinlik sekmesi, **Settings › System** içinde Logs ve Debug seçeneklerinin yanında yer alır. Chat araç kartlarını destekleyen aynı Gateway `session.tool` / araç olay akışından türetilen, canlı araç etkinliğine yönelik geçici ve tarayıcıya yerel bir gözlem aracıdır. Başka bir Gateway olay ailesi, uç nokta, kalıcı etkinlik deposu, metrik akışı veya harici gözlemci akışı eklemez.

Etkinlik girdileri yalnızca temizlenmiş özetleri ve sansürlenmiş, kısaltılmış çıktı önizlemelerini tutar. Araç bağımsız değişkenlerinin değerleri Etkinlik durumunda saklanmaz; kullanıcı arayüzü bağımsız değişkenlerin gizlendiğini gösterir ve yalnızca bağımsız değişken alanı sayısını kaydeder. Bellek içi liste geçerli tarayıcı sekmesini izler, Control UI içindeki gezinmelerde korunur ve sayfa yeniden yüklendiğinde, oturum değiştirildiğinde veya **Clear** seçildiğinde sıfırlanır.

## Operatör terminali

Yerleştirilebilir operatör terminali varsayılan olarak devre dışıdır. Etkinleştirmek için `gateway.terminal.enabled: true` değerini ayarlayın ve Gateway'i yeniden başlatın. Terminal, bir `operator.admin` bağlantısı gerektirir ve etkin aracının çalışma alanında bir ana makine PTY'si açar. Yeni sekmeler, o anda seçili sohbet aracısını izler.

<Warning>
Terminal, sınırlandırılmamış bir ana makine kabuğudur ve Gateway işleminin ortamını devralır. Yalnızca güvenilir operatör dağıtımları için etkinleştirin. OpenClaw, `sandbox.mode: "all"` kullanan aracılar için terminal oturumlarını reddeder; etkin bir aracının bu moda geçirilmesi, mevcut ve devam eden terminal oturumlarını kapatır.
</Warning>

Yerleştirme alanını açıp kapatmak için **Ctrl + backtick** kullanın. Düzen, alta ve sağa yerleştirmeyi destekler, tarayıcı görüntü alanıyla birlikte yeniden boyutlanır ve birden fazla kabuk sekmesini korur. `gateway.terminal.enabled` ve isteğe bağlı `gateway.terminal.shell` geçersiz kılma ayarı için [Gateway yapılandırması](/tr/gateway/configuration-reference#gateway) bölümüne bakın.

Sahibi tarafından yetkilendirilmiş, sandbox dışında çalışan aracılar, operatörün izlemesi gereken uzun veya etkileşimli işler için `terminal` aracını kullanabilir. Her araç çağrısı, aracının kendi Gateway PTY'lerini açabilir, okuyabilir, bunlara yazabilir, yeniden boyutlandırabilir, kapatabilir veya listeleyebilir. Yeni oturumlar varsayılan olarak ortak bağlı bir Control UI sekmesi açar; böylece aracı ve operatör çıktıyı paylaşır, her ikisi de yazabilir veya yeniden boyutlandırabilir. Aracı erişimi tam oturumla sınırlıdır: Bir aracı, operatör tarafından oluşturulan terminalleri veya başka bir aracı oturumunun açtığı terminalleri okuyamaz ya da denetleyemez.

Bir veya daha fazla dosyayı etkin terminale sürükleyin ya da dosya seçmek için ataç düğmesini kullanın. OpenClaw her dosyayı PTY'nin sahibi olan makinede hazırlar ve kabuk için tırnak içine alınmış mutlak yolları imleç konumuna yapıştırır; hiçbir zaman Enter tuşuna basmaz veya girdiyi çalıştırmaz. Kompakt bir toplu işlem göstergesi geçerli dosyayı ve tamamlanan dosya sayısını gösterir. İptal işlemi, yolları yapıştırmadan kalan toplu işlemi durdurur; başarısız bir aktarım görünür kalır, böylece tamamlanan dosyaları yeniden yüklemeden o dosyadan itibaren yeniden deneyebilirsiniz. Görüntüler, PDF'ler, arşivler ve diğer dosya türleri dosya başına 16 MiB'a kadar kabul edilir. Hazırlanan dosyalar, POSIX ana makinelerinde özel bir sistem geçici dizini (dizin modu `0700`, dosya modu `0600`) veya Windows'ta kullanıcı profili ACL sınırı altındaki bir dizini ve 24 saatlik temizleme zamanlayıcısını kullanır; bu nedenle saklamanız gereken her şeyi taşıyın veya kopyalayın.

Yol ekleme; PowerShell, `cmd.exe` ve tanınan POSIX kabuklarını (`sh`, Bash, Dash, Ash, Ksh, Zsh ve Fish), Windows'taki Git Bash dâhil olmak üzere destekler. Tırnaklama kuralları güvenli biçimde çıkarılamadığı için diğer kabuk geçersiz kılmaları reddedilir; yerel bir WSL terminali ve Linux yükleme yolları için Gateway'i WSL içinde çalıştırın. `%` veya `!` içeren `cmd.exe` yolları da reddedilir; çünkü bu kabuk, bu karakterleri çift tırnak içinde bile genişletir.

Oturumlar kenar çubuğunda keşfedilen Codex ve Claude Code oturumları, aynı terminal panelinde kendi yerel CLI'larıyla açılabilir. Normal bir satır tıklamasının `codex resume` veya `claude --resume` açmasını sağlamak için **Settings › Chat** bölümünde **Open Codex/Claude threads in** ayarını **Terminal** olarak belirleyin; varsayılan, salt okunur OpenClaw görüntüleyicisi olmaya devam eder. Bir satırın sağ tıklama veya kebap menüsü her zaman iki seçeneği de sunar ve oturum uygunsa görüntüleyici üstbilgisinde **Open in terminal** seçeneği bulunur.

Uygunluk, oturum ve ana makine bazında belirlenir. Gateway'e yerel oturumlar, sağlayıcının sahip olduğu sürdürme komutunu Gateway ana makinesinde başlatır. Eşleştirilmiş Node oturumları, izin verilenler listesindeki bir sağlayıcı komutunu sahibi olan Node üzerinde başlatır ve yalnızca ilgili PTY'nin çıktı, girdi ve yeniden boyutlandırma olaylarını aktarır; bu, genel bir Node kabuğunu kullanıma açmaz veya tarayıcı tarafından sağlanan komutları kabul etmez. Dosya yüklemeleri ayrı, boyutu sınırlandırılmış `terminal.upload` Node komutunu kullanır ve açık durumdaki terminal oturumuna bağlı kalır. Bu komut ilk kez göründüğünde Node eşleştirme yükseltmesini onaylayın. Çift yönlü akış desteği olmayan gömülü çalışan köprüleri dâhil, eşleşen terminal sürdürme komutunu duyurmayan Node'lar görüntüleyiciyi kullanılabilir tutar ve terminal açma işlemini kullanılamaz olarak gösterir; eski Node'lar terminali çalıştırmaya devam edebilir ancak sürüklenen dosyaları alamaz.

Bağlantıya ait oturumlar, bağlantı kesintilerinde çalışmaya devam eder: Sayfanın yeniden yüklenmesi, dizüstü bilgisayarın uykuya geçmesi veya kısa süreli ağ kesintisi oturumu sonlandırmak yerine Gateway'de ayırır ve aynı tarayıcı sekmesi yeniden bağlandığında son çıktılar yeniden oynatılarak oturuma tekrar bağlanır. Ayrılmış, bağlantıya ait oturumlar `gateway.terminal.detachedSessionTimeoutSeconds` sonrasında sonlandırılır (varsayılan 300 saniye; `0` bağlantı kesildiğinde sonlandırma davranışını geri getirir). Bu oturumlardan birine bağlanmak, tmux tarzı devralma işlemi olmaya devam eder.

Aracıya ait oturumlar bir tarayıcı bağlantısına bağlı değildir. `terminal.attach`, sahipliği devralmadan her tarayıcıyı görüntüleyici olarak ekler ve bir görüntüleyici sekmesinin kapatılması yalnızca o tarayıcının bağlantısını ayırır. PTY; sahibi olan aracı kapatana, işlemi sonlanana, ilke tarafından devre dışı bırakılana veya Gateway kapanana kadar çalışmaya devam eder. `terminal.list` her girdiyi bağlantıya veya aracıya ait olarak işaretler ve `terminal.text` bir yönetici bağlantısının bağlanmadan yakın zamandaki düz metin çıktısını okumasına olanak tanır.

Terminal ayrıca `/?view=terminal` konumunda tam ekran, yalnızca terminal içeren bir belge olarak kullanılabilir. iOS ve Android uygulamaları, kayıtlı Gateway kimlik bilgilerini yeniden kullanarak bu sayfayı Terminal ekranlarına gömer; kullanılabilirlik aynı `gateway.terminal.enabled` ve `operator.admin` geçidine bağlıdır ve bağlı Gateway terminal sunmadığında sayfada bir bildirim gösterilir.

## Tarayıcı paneli

Control UI, Gateway tarafından denetlenen tarayıcıyı (aracıların [tarayıcı aracı](/tr/tools/browser-control) üzerinden kullandığı tarayıcıyla aynı) herhangi bir normal web tarayıcısında işleyen yerleştirilebilir bir tarayıcı paneliyle birlikte gelir; yerel web görünümü gerekmez. Bağlı Gateway bir `operator.admin` bağlantısına `browser.request` duyurduğunda görünür; iş parçacığı çalışma alanı rayındaki küre düğmesi paneli açıp kapatır. Panel; sekmeler, düzenlenebilir bir URL çubuğu, geri/ileri/yeniden yükleme ve kendi tarayıcınızda açma özellikleriyle canlı sayfa anlık görüntüsünü gösterir, sağa veya alta yerleştirilebilir ve tıklamaları, fare tekerleğiyle kaydırmayı ve temel yazma işlemlerini uzak sayfaya iletir.

İki yakalama modu, sayfa bağlamını aracı için paketler:

- **Açıklama ekle (kalem)**: Sayfa üzerine serbest biçimli işaretler çizin. **Sohbete gönder**, çizgileri ekran görüntüsüyle birleştirir, görüntüyü etkin sohbet oluşturucusuna ekler ve sayfa URL'sini, başlığını ve işaretlenen her bölgeyi açıklayan bir istemi önceden doldurur; böylece aracı tam olarak nereyi daire içine aldığınızı bilir.
- **İncele (işaretçi)**: İmlecin altındaki öğeyi (seçici, erişilebilir ad, rol, boyut) görmek için üzerine gelin; bu öğenin ayrıntılarını ve vurgulanmış ekran görüntüsünü aynı oluşturucu akışı üzerinden göndermek için tıklayın. İnceleme, fare tekerleğiyle kaydırma ve geri/ileri işlemleri `browser.evaluateEnabled` gerektirir (varsayılan olarak etkindir).

macOS uygulaması, panoda tıklanan bağlantılar için yerel bağlantı tarayıcısı kenar çubuğunu korur; tarayıcı paneli burada da çalışır ve diğer tüm platformlarda sayfalara açıklama eklemenin yoludur.

## Sohbet davranışı

<AccordionGroup>
  <Accordion title="Send and history semantics">
    - `chat.send` **engellemesizdir**: `{ runId, status: "started" }` ile hemen alındığını onaylar ve yanıt `chat` olayları aracılığıyla akış hâlinde iletilir. Güvenilir Control UI istemcileri, yerel tanılama için isteğe bağlı ACK zamanlama meta verilerini de alabilir.
    - Sohbet yüklemeleri, görüntülerin yanı sıra video olmayan dosyaları da kabul eder. Görüntüler yerel görüntü yolunu korur; diğer dosyalar yönetilen medya olarak depolanır ve geçmişte ek bağlantıları olarak gösterilir.
    - Aynı `idempotencyKey` ile yeniden gönderme, çalışırken `{ status: "in_flight" }`, tamamlandıktan sonra ise `{ status: "ok" }` döndürür.
    - `chat.history` yanıtlarının boyutu, UI güvenliği için sınırlandırılmıştır. Transkript girdileri çok büyük olduğunda Gateway, uzun metin alanlarını kısaltabilir, ağır meta veri bloklarını atlayabilir ve aşırı büyük iletileri bir yer tutucuyla (`[chat.history omitted: message too large]`) değiştirebilir.
    - Görünür bir asistan iletisi `chat.history` içinde kısaltıldığında yan okuyucu; `sessionKey`, gerektiğinde etkin `agentId` ve transkript `messageId` aracılığıyla `chat.message.get` üzerinden, görüntüleme için normalleştirilmiş tam transkript girdisini isteğe bağlı olarak getirebilir. Gateway yine de daha fazlasını döndüremiyorsa okuyucu, kısaltılmış önizlemeyi sessizce yinelemek yerine açık bir kullanılamıyor durumu gösterir.
    - Asistan tarafından oluşturulan görüntüler, yönetilen medya referansları olarak kalıcı hâle getirilir ve kimliği doğrulanmış Gateway medya URL'leri üzerinden yeniden sunulur; böylece yeniden yüklemeler, ham base64 görüntü yüklerinin sohbet geçmişi yanıtında kalmasına bağlı olmaz.
    - Control UI, `chat.history` oluşturulurken yalnızca görüntülemeye yönelik satır içi direktif etiketlerini (örneğin `[[reply_to_*]]` ve `[[audio_as_voice]]`), düz metin araç çağrısı XML yüklerini (`<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` ve kısaltılmış araç çağrısı blokları dâhil) ve sızmış ASCII/tam genişlikli model kontrol belirteçlerini görünür asistan metninden çıkarır. Görünür metninin tamamı yalnızca tam sessizlik belirteci `NO_REPLY` / `no_reply` veya Heartbeat alındı belirteci `HEARTBEAT_OK` olan asistan girdilerini atlar.
    - Etkin bir gönderim ve son geçmiş yenilemesi sırasında, `chat.history` kısa süreliğine daha eski bir anlık görüntü döndürürse sohbet görünümü yerel iyimser kullanıcı/asistan iletilerini görünür tutar; Gateway geçmişi güncellendiğinde standart transkript bu yerel iletilerin yerini alır.
    - Canlı `chat` olayları teslimat durumudur; `chat.history` ise kalıcı oturum transkriptinden yeniden oluşturulur. Araç sonlandırma olaylarından sonra Control UI geçmişi yeniden yükler ve yalnızca küçük bir iyimser kuyruğu birleştirir; transkript sınırı [WebChat](/tr/web/webchat) bölümünde belgelenmiştir.
    - `chat.inject`, oturum transkriptine bir asistan notu ekler ve yalnızca UI güncellemeleri için bir `chat` olayı yayınlar (ajan çalıştırılmaz, kanal teslimatı yapılmaz).
    - Kenar çubuğu, yüklenmiş her etkin oturumu ajan bölümüne ve sabitlenmiş/kanal/iş/özel/Sohbetler gruplarına göre listeler; tek bir Yeni Oturum eylemi taslak iletişim kutusunu açar. Görünür bir satırı açmak yalnızca vurguyu taşır. Oturumlar sabitlemek için Sabitlenmiş grubuna veya taşımak için özel bir gruba ya da Sohbetler'e sürüklenebilir; özel gruplar daraltılabilir ve sürüklenerek yeniden sıralanabilir, grup adları ve sırası Gateway üzerinden eşitlenir ve daraltılmış durum tarayıcıda kalır. Yeni bir pano oturumu, komut olmayan ilk iletisinden eşzamansız olarak kısa ve öz bir başlık oluşturur; açıkça belirtilen adlar ile kimliği doğrulanmış gönderici kimliği ayrı tutulur, dolayısıyla hesap adları hiçbir zaman oluşturulan başlık olarak kullanılmaz. Bu ayrı model çağrısını daha düşük maliyetli bir modele yönlendirmek için `agents.defaults.utilityModel` (veya `agents.entries.*.utilityModel`) ayarlayın; bu farklı model başarısız olursa başlık oluşturma bir kez birincil modelle yeniden denenir. Başka bir ajan bölümünü genişletmek, açık sohbetten ayrılmadan o ajanın oturumlarına göz atılmasını sağlar.
    - İleti dizisi araması komut paletinde bulunur (⌘K veya sol üstteki kontrol kümesindeki arama düğmesi): bir sorgu yazıldığında ajanlar genelinde eşleşen sınırlı sayıda sayfa izlenir, dâhilî alt/Cron satırları filtrelenir ve görünür eşleşmeler gezinme komutlarının yanında listelenir. İleti Dizileri sayfası, filtrelerle birlikte kapsamlı ve aranabilir listeyi korur.
    - Her kenar çubuğu satırı, doğrudan sabitleme erişiminin yanı sıra okunmamış durumu, yeniden adlandırma, çatallama, gruplama, arşivleme ve silme için tam bir bağlam menüsü içerir. Birden çok seçili satır (Cmd/Ctrl-tıklama, aralıklar için Shift-tıklama) okunmamış durumu, gruplama, arşivleme ve silmeyi kapsayan bir toplu işlem menüsü alır; seçili her oturum arşivlenebilir olmadığı sürece toplu arşivleme/silme devre dışı kalır. Etkin bir çalışma ve bir ajanın ana oturumu arşivlenemez. Seçili durumdaki oturum arşivlendiğinde veya silindiğinde Sohbet, o ajanın ana oturumuna geri döner.
    - macOS uygulamasında OpenClaw işareti, bir kenar çubuğu satırını kullanmak yerine pencere denetimlerinin yanındaki normalde boş olan yerel başlık çubuğu şeridini kullanır.
    - Masaüstü genişliklerinde sohbet denetimleri tek bir kompakt satırda kalır ve transkriptte aşağı kaydırılırken daralır; yukarı kaydırmak, en üste dönmek veya en alta ulaşmak denetimleri geri getirir.
    - Aynı oturumu başka kişiler de görüntülerken oturum başlığı, çalışma alanı rozetinin yanında küçük bir üst üste avatar grubu gösterir; en fazla dört görüntüleyici avatarını ve fazlası için bir sayı gösterir, yalnız olduğunuzda ise kaybolur.
    - Art arda gelen aynı, yalnızca metin içeren iletiler, sayı rozeti bulunan tek bir baloncuk olarak oluşturulur. Görüntü, ek, araç çıktısı veya tuval önizlemesi içeren iletiler birleştirilmez.
    - Kullanıcı iletisi baloncukları transkript eylemleri içerir: üzerine gelindiğinde görünen bir geri sarma düğmesi ("Don't ask again" seçeneği bulunan onay açılır penceresi) ile sağ tıklama menüsündeki **Buraya geri sar** ve **Buradan çatalla** seçenekleri. Geri sarma, oturumu ilgili iletiden hemen önceki duruma yönlendirir ve metni düzenlenip yeniden gönderilmesi için düzenleyiciye döndürür (`sessions.rewind`, `operator.admin`); çatallama, iletiden önceki etkin yol önekinden yeni bir oturum oluşturur, bu oturumu açar ve düzenleyicisini aynı metinle doldurur (`sessions.fork`, `operator.write`). Ajan çalışırken her iki eylem de açıklayıcı bir araç ipucuyla devre dışı bırakılır, yalnızca kalıcı kullanıcı iletilerine uygulanır ve konuşması haricî bir ajan çalıştırma ortamına ait oturumlarda reddedilir. Geri sarma yalnızca sohbet bağlamını taşır; dosyalar ve araçların diğer yan etkileri geri alınmaz. Geri sarma öncesindeki transkript ise yalnızca eklemeli oturum deposunda korunur. Bu depo birden fazla transkript dalı içerdiğinde sohbet başlık çubuğu, her dalın en son iletisini, ileti sayısını ve güncelliğini içeren bir dal menüsü gösterir; etkin olmayan bir dal seçildiğinde mevcut oturum korunmuş olan bu yola geri geçirilir (`sessions.branches.list`, `operator.read`; `sessions.branches.switch`, `operator.admin`). Ajan çalışırken dal değiştirme de kullanılamaz ve zaten etkin olan dalın seçilmesi RPC sınırında türü belirlenmiş bir etkisiz işlem hatasıdır. Kullanıcı baloncuklarındaki ayrı gizleme eylemi, bir iletiyi yalnızca mevcut tarayıcıda gizler; ileti transkriptte kalır ve ajan onu görmeye devam eder.
    - Bir oturumun çalışma kopyası, GitHub deposunun varsayılan olmayan bir dalındaysa sohbet görünümü düzenleyicinin üzerine pull request rozetlerini sabitler: PR numarası, depo, dal, fark sayıları, bir CI rozeti ve taslak/birleştirilmiş/kapatılmış durumu; her biri PR'a bağlantı verir. Satır en fazla iki rozet gösterir — önce canlı (açık/taslak) PR'lar — ve "Daha fazlasını göster" düğmesi daraltılmış birleştirilmiş/kapatılmış geçmişi ortaya çıkarır. CI rozeti; başarılı/başarısız/çalışan/atlanan kontrol sayılarını ve PR'ın kontroller sayfasına bir bağlantıyı içeren küçük bir CI izleme açılır penceresi açar. Algılama, ayarlandıklarında Gateway'in `GH_TOKEN`/`GITHUB_TOKEN` değerlerini yeniden kullanan `controlUi.sessionPullRequests` aracılığıyla sunucu tarafında çalışır. GitHub API hız sınırına ulaşıldığında rozetler bilinen son durumu korur ve durumun güncel olmayabileceğini belirten bir uyarı gösterir; bir rozeti kapatmak, mevcut tarayıcı profilinde bu oturum için onu gizler. Henüz bir PR yokken satır dalın kendisini gösterir: depo, dal adı ve varsayılan dal birleştirme tabanına göre farkın +/− boyutu (işlenmiş ve işlenmemiş çalışma). Gönderilmiş dalda karşılaştırılacak commit'ler bulunduğunda satır, GitHub'ın yeni pull request sayfasını açan bir PR Oluştur düğmesi ekler; bundan önce de değiştirilmiş dosyaları (işlenmiş, işlenmemiş veya izlenmeyen) bulunan bir oturumda satır gösterilir ancak düğme bulunmaz. Açık veya taslak bir PR varken satır kendisini gizler. Dal satırı yalnızca yerel git verilerinden gelir; bu nedenle GitHub hız sınırındayken de kullanılabilir kalır ve aynı eski durum uyarısını taşır, çünkü sınır sıfırlanana kadar "PR bulunamadı" sonucuna güvenilemez.
    - Oturum fark paneli, bir oturumun çalışma kopyasının gerçekte neleri değiştirdiğini gösterir: çalışma alanı çubuğundaki veya sohbet başlık çubuğundaki dal düğmesi; dal, işlenmemiş ve izlenmeyen çalışmanın çalışma kopyasının varsayılan dal birleştirme tabanına göre dosya başına farkını içeren ayrıntı panelini açar — durum noktası, yeniden adlandırma oku, dosya başına +/− sayıları, daraltılabilir dosyalar ve parçalar arasında "N değiştirilmemiş satır" işaretleri. Farklar, `sessions.diff` Gateway yöntemi (`operator.read` kapsamı) aracılığıyla sunucu tarafında hesaplanır; ikili ve aşırı büyük dosyalar yalnızca istatistik içeren girdilere indirgenir ve düğme yalnızca bağlı Gateway `sessions.diff` özelliğini duyurduğunda görünür.
    - Her Sohbet bölmesinin bir başlık çubuğu vardır. Yeniden adlandırmak için oturum başlığına tıklayın; çalışma alanı rozeti çalışma kopyası yolunu veya dalını kopyalar ve yerel Gateway çalışma alanlarını ana makinenin dosya yöneticisinde gösterebilir. Uzak ve yürütme düğümü oturumları kopyalama eylemlerini korur ancak gösterme eylemini gizler.
    - Her Sohbet bölmesindeki ileti dizisi çalışma alanı çubuğu, ileti dizisi dosyalarını, proje dosyalarını ve yapıtları listeler. Varsayılan olarak bölmenin sağ kenarına sabitlenir; alta taşımak için başlığını sürükleyin (veya sabitleme düğmesini kullanın). Seçim mevcut tarayıcı profilinde saklanır. Daraltılmış bir çubuk hiç yer kaplamaz: ⇧⌘B ile veya başlık çubuğundaki, değiştirilmiş dosya sayısı rozeti taşıyan dosyalar düğmesiyle yeniden açın. Ayrı dosya, araç ve Canvas ayrıntı paneli bundan etkilenmez.
    - Sohbetteki bir dosya referansına, genişletilmiş bir okuma/düzenleme/yazma araç kartındaki dosya yoluna veya çalışma alanı çubuğundaki bir dosya satırına tıklamak dosya ayrıntı panelini açar: sözdizimi vurgulama, satır numaraları, satıra atlama, dosya içi arama, kopyalama eylemleri ve haricî düzenleyicide açma menüsü içeren CodeMirror tabanlı bir kod görünümü. Gateway, bir `operator.admin` bağlantısına `sessions.files.set` özelliğini duyurduğunda panel; değişiklik takibi ve Cmd/Ctrl-S ile kaydetme özelliklerine sahip bir Düzenleme modu ekler. Kaydedilmemiş taslaklar, açıkça kaydedilene veya atılana kadar mevcut tarayıcı sekmesinde dosya, panel ve oturum gezinmeleri arasında korunur. Kaydetme işlemleri, `sessions.files.get` tarafından döndürülen içerik karması üzerinde karşılaştır ve değiştir yöntemiyle yapılır: dosya yüklendiğinden beri diskte değiştiyse (örneğin ajan çalışmaya devam ettiği için) panel, Yeniden Yükle (en son içeriği al) ve Üzerine Yaz (yerel düzenlemeyi koru) eylemlerini içeren bir çakışma bildirimi gösterir. Yazma işlemleri, okumalarla aynı dosya sistemi güvenli çalışma alanı korumalarından geçer — yol kapsamı, sembolik bağlantı/sabit bağlantı reddi ve 256 KB UTF-8 sınırı — ve yalnızca mevcut dosyaların üzerine yazar; düzenleyici hiçbir zaman dosya oluşturmaz veya silmez.
    - Her Sohbet bölmesindeki arka plan görevleri çubuğu, mevcut ajanın arka plan görevlerini ve alt ajanlarını listeler (`tasks.list` ajan kapsamındadır ve `task` olaylarıyla canlı tutulur): çalışan işler canlı bir geçen süre sayacı, araç kullanım sayısı, o anda kullanılan araç ve durdurma denetimi gösterir; daraltılabilir tamamlananlar bölümü çalışma sürelerini ekler; Transkripti görüntüle bağlantısı ise görevin alt oturumunu bölmede açar. Başlık çubuğundaki etkinlik düğmesiyle açın; görev anlık görüntüsü önceden yüklenir, bu nedenle çubuk açılmadan da çalışan görev sayısı rozetini gösterir. Görevler sayfası, ajanlar genelindeki tam kayıt defteri olmaya devam eder.
    - Çalışma alanı şeridi, arka plan görevleri şeridi ve ayrıntı paneli pencerenin değil, her bölmenin kendi genişliğine uyum sağlar: dar bir bölmede veya kompakt bir pencerede her iki şerit de alt şeritler olarak görünür (bölme genişleyene kadar yan kenara sabitleme denetimleri gizlenir; yalnızca bir sütun sığdığında yan yuvadaki öncelik çalışma alanı şeridindedir) ve ayrıntı paneli, iş parçacığıyla aynı satırı paylaşmak yerine yatay yeniden boyutlandırma tutamacıyla iş parçacığının altına yığılır. Telefon boyutundaki görünüm alanlarında ayrıntı paneli hâlâ tam ekran açılır.
    - Sohbet başlığındaki model ve düşünme seçicileri, etkin oturumu `sessions.patch` aracılığıyla hemen günceller; bunlar yalnızca tek gönderime özgü seçenekler değil, kalıcı oturum geçersiz kılmalarıdır.
    - **Bölünmüş görünüm:** sohbet başlık çubuğundan (iş parçacığı farkı, arka plan görevleri ve iş parçacığı dosyaları açma/kapama denetimlerinin yanında) açın, ardından etkin bölmeyi sığabilecek sayıda bölme oluşturacak şekilde sağa veya aşağıya bölün. Her bölmenin kendi iş parçacığı, dökümü, ileti oluşturucusu ve araç akışı vardır.
    - `screen` aracına sahip ajanlar, yetenekli bir Denetim Arayüzü bağlıyken aynı bölme, kenar çubuğu, terminal, tarayıcı, odak ve gezinme değişikliklerini isteyebilir. Protokol v1, komutu bağlı tüm yetenekli Denetim Arayüzlerine uygular; bkz. [Ekran](/tools/screen).
    - Bir oturumu bölmede açmak için kenar çubuğundan sohbete sürükleyin. Animasyonlu bırakma önizlemesi bölgeler arasında kayarak sonucu etiketler — yeni bir bölmenin kaplayacağı tam yarının üzerinde "Böl", bütün bir bölmenin üzerinde "Burada aç" — ve bırakma işlemleri tek bölmeli modda da çalışır.
    - Etkin bölünmüş bölme, kenar çubuğu seçimini ve URL'yi belirler. Başlık çubuğuna bölme ve kapatma denetimleri eklenir; ayırıcılar sütunları ve yığılmış bölmeleri yeniden boyutlandırır, tarayıcı da düzeni yeniden yüklemeler arasında yerel olarak saklar.
    - Dar ekranlarda bölünmüş görünüm düzeni korur ancak kapatma denetimini içeren başlığıyla birlikte yalnızca etkin bölmeyi işler.
    - Aynı oturuma yönelik bir model seçici değişikliği hâlâ kaydedilirken ileti gönderirseniz ileti oluşturucu, `chat.send` çağrısından önce bu oturum güncellemesinin tamamlanmasını bekler; böylece gönderimde seçilen model kullanılır.
    - `/new` yazıldığında, New Chat ile aynı yeni pano oturumu oluşturulur ve bu oturuma geçilir; ancak `session.dmScope: "main"` yapılandırılmışsa ve mevcut üst oturum ajanın ana oturumuysa ana oturum yerinde sıfırlanır. `/reset` yazıldığında, Gateway'in mevcut oturum için açıkça tanımlanmış yerinde sıfırlama işlemi korunur.
    - Sohbet model seçicisi, Gateway'in yapılandırılmış model görünümünü ister. `agents.defaults.modelPolicy.allow` boş değilse seçiciyi bu politika belirler; buna sağlayıcı kapsamlı katalogları dinamik tutan `provider/*` girdileri de dahildir. Aksi takdirde seçici, yapılandırılmış girdilerin yanı sıra kullanılabilir kimlik doğrulaması bulunan sağlayıcıları gösterir; `agents.defaults.models` altındaki takma adlar ve ayarlar bunu kısıtlamaz. Tam katalog, `view: "all"` ile hata ayıklama amaçlı `models.list` RPC üzerinden kullanılabilir olmaya devam eder.
    - Yeni Gateway oturum kullanım raporları güncel bağlam tokenlarını içerdiğinde, sohbet ileti oluşturucu araç çubuğu kullanılan yüzdeyi gösteren küçük bir bağlam kullanım halkası görüntüler. Geçerli bağlam penceresini, son çalıştırmanın token sayılarını ve tahmini toplam maliyetini, sağlayıcı/model kimliğini ve bildirildiğinde en son sağlayıcı yanıtının girdi/çıktı/önbellek maliyeti dökümünü görmek için halkayı açın. Bağlam baskısı yükseldiğinde halka uyarı stiline geçer ve önerilen Compaction düzeylerinde normal oturum Compaction yolunu çalıştıran kompakt bir düğme gösterir. Gateway yeniden güncel kullanım bildirene kadar eski token anlık görüntüleri gizlenir.

  </Accordion>
  <Accordion title="Konuşma modu (tarayıcıda gerçek zamanlı)">
    Konuşma modu, kayıtlı bir gerçek zamanlı ses sağlayıcısı kullanır. OpenAI'ı `talk.realtime.provider: "openai"` ve bir `openai` API anahtarı profili, `talk.realtime.providers.openai.apiKey` veya `OPENAI_API_KEY` ile yapılandırın. OpenAI Realtime, herkese açık Platform API'sini kullanır ve bir Platform API anahtarı gerektirir; Codex OAuth oturumu bu yüzey için yeterli değildir. Google'ı `talk.realtime.provider: "google"` ve `talk.realtime.providers.google.apiKey` ile yapılandırın. Tarayıcı hiçbir zaman standart bir sağlayıcı API anahtarı almaz: OpenAI, WebRTC için geçici bir Realtime istemci sırrı; Google Live ise tarayıcı WebSocket oturumu için tek kullanımlık, kısıtlı bir Live API kimlik doğrulama belirteci alır ve talimatlar ile araç bildirimleri Gateway tarafından belirtece kilitlenir. Yalnızca arka uç gerçek zamanlı köprüsü sunan sağlayıcılar Gateway aktarma taşıması üzerinden çalışır; böylece kimlik bilgileri ve sağlayıcı soketleri sunucu tarafında kalırken tarayıcı sesi kimliği doğrulanmış Gateway RPC'leri üzerinden taşınır. Realtime oturum istemi Gateway tarafından oluşturulur; `talk.client.create` çağıranın sağladığı talimat geçersiz kılmalarını kabul etmez.

    Kalıcı sağlayıcı, model, ses, taşıma, akıl yürütme çabası, tam VAD eşiği, sessizlik süresi ve önek dolgusu varsayılanları **Ayarlar → İletişim → Konuşma** bölümünde bulunur; bunları değiştirmek `operator.admin` erişimi gerektirir. Gateway aktarmasını yapılandırmak arka uç aktarma yolunu zorunlu kılar; WebRTC'yi yapılandırmak oturumu istemciye ait tutar ve sağlayıcı bir tarayıcı oturumu oluşturamazsa sessizce aktarmaya geri dönmek yerine başarısız olur.

    Konuşma denetiminin kendisi, düzenleyici araç çubuğundaki mikrofon düğmesidir. Açılır menüsünde **Sistem varsayılanı** ve USB, Bluetooth ve sanal girişler dâhil olmak üzere tarayıcının sunduğu tüm mikrofonlar listelenir. Seçilen cihaz kimliği tarayıcıda yerel olarak kalır ve hiçbir zaman Gateway'e gönderilmez; söz konusu cihaz kaybolursa Konuşma, sessizce farklı bir mikrofondan kayıt yapmak yerine başka bir giriş seçmenizi ister. Konuşma etkinken mikrofon düğmesi, canlı giriş düzeyi ölçerini gösteren bir kapsüle dönüşür; düğmeye tıklamak ses girişini durdurur, üzerine gelmek ise durdurma simgesini gösterir. Gerçek zamanlı bir araç çağrısı, `talk.client.toolCall` üzerinden yapılandırılmış daha büyük modele danışırken ekran okuyucular `Connecting voice input...`, `Listening...` veya `Asking OpenClaw...` duyurusunu yapar. Çalışan bir aracı yanıtını durdurmak için kapsülün yanında ayrı bir kare **Durdur** denetimi bulunur.

    **Görüntülü Konuşma**, OpenAI Realtime WebRTC ve Google Live tarayıcı oturumlarında kullanılabilir. Kamera düğmesine tıklayın, kamera ve mikrofon erişimine izin verin ve yerel önizlemeyi onaylayın. OpenAI, `describe_view` görsel bağlam istediğinde tarayıcı veri kanalı üzerinden sınırlandırılmış tek bir JPEG karesi gönderir. Google Live, sınırlandırılmış JPEG karelerini tarayıcıdan doğrudan sağlayıcıya desteklenen saniyede en fazla bir kare hızıyla gönderir ve `describe_view` işlev çağrılarını kamera akışı durumuyla yanıtlar. Kamera kareleri hiçbir zaman Gateway'den geçmez. Konuşmayı durdurmak önizlemeyi kapatır ve her iki medya izini de serbest bırakır. Sağlayıcının kablo protokolü sözleşmeleri için Google'ın [Live API yetenekleri](https://ai.google.dev/gemini-api/docs/live-api/capabilities#video) ve [işlev çağırma kılavuzuna](https://ai.google.dev/gemini-api/docs/live-api/tools) bakın.

    Bakımcı canlı duman testi: `OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts`; OpenAI arka uç WebSocket köprüsünü, OpenAI tarayıcı WebRTC SDP değişimini, bir JPEG karesi ve `describe_view` işlev gidiş dönüşüyle Google Live kısıtlı belirteçli tarayıcı kurulumunu ve sahte mikrofon medyasıyla Gateway aktarma tarayıcı bağdaştırıcısını doğrular. Komut yalnızca sağlayıcı durumunu yazdırır ve sırları günlüğe kaydetmez.

  </Accordion>
  <Accordion title="Durdurma ve iptal etme">
    - **Durdur**'a tıklayın. Tam bir yerel çalıştırma kimliğine sahip çalıştırmalar `chat.abort` çağrısını yapar; seçili oturum durumu etkin çalışma bildirirken Denetim Arayüzünde yerel çalıştırma kimliği yoksa bunun yerine `sessions.abort` çağrılır. Genel olmayan oturumlarda bu seçili oturum yolu, durdurmadan sonra çalışmayı yeniden başlatamamaları için kuyruğa alınmış takip iletilerini de atar.
    - Bir çalıştırma etkinken normal takip iletileri Gateway'in etkin `messages.queue` modunu kullanır. `steer` çalışan tura ekleme yapar; diğer modlar tarayıcının kalıcı kuyruklu teslimatını korur. Yönlendirme reddedilirse de bu kuyruğa geri dönülür. Kuyruğa alınmış bir iletiyi elle eklemek için üzerindeki **Yönlendir** düğmesine tıklayın.
    - **Ayarlar → Görünüm → Sohbet → Aracı çalışırken takip iletileri**, geçerli tarayıcı için bu sunucu varsayılanını geçersiz kılabilir. Sayfa, geçersiz kılmayı açıkça işaretler ve **Sunucu varsayılanına sıfırla** seçeneğini sunar. `Steer into the active run` takip iletilerini hemen gönderirken `Queue until the run ends` çalıştırma bitene kadar bekletir.
    - Bant dışı iptal etmek için `/stop` (veya `stop`, `stop action`, `stop run`, `stop openclaw`, `please stop` gibi bağımsız iptal ifadeleri) yazın.
    - `chat.abort`, söz konusu oturumdaki tüm etkin çalıştırmaları iptal etmek için `{ sessionKey }` seçeneğini (`runId` olmadan) destekler. Denetim Arayüzü yerel çalıştırma kimliğine sahip olmadığında `sessions.abort` kullanır.

  </Accordion>
  <Accordion title="İptal edilen kısmi içeriğin korunması">
    - Bir çalıştırma iptal edildiğinde kısmi yardımcı metni yine de kullanıcı arayüzünde gösterilebilir.
    - Arabelleğe alınmış çıktı varsa Gateway, iptal edilen kısmi yardımcı metnini transkript geçmişinde kalıcı hâle getirir.
    - Kalıcı hâle getirilen girdiler, transkript tüketicilerinin iptal edilmiş kısmi çıktıları normal tamamlanma çıktısından ayırt edebilmesi için iptal meta verileri içerir.

  </Accordion>
</AccordionGroup>

## Bağlantı kaybı ve yeniden bağlanma

Bir oturum kurulduktan sonra Gateway bağlantısının kesilmesi oturumunuzu kapatmaz. İstemci, geri çekilmeli olarak (800 ms'den 15 sn'ye kadar) otomatik biçimde yeniden denerken pano görünür kalır ve üst çubuğun altında yüzen sarı renkli bir "Gateway bağlantısı kesildi — Yeniden bağlanılıyor…" kapsülü gösterilir. Canlı güncellemeler ve gerçek zamanlı/oturum eylemleri bağlantı geri gelene kadar duraklatılır; kapsüldeki **Şimdi yeniden dene** seçeneği anında bir denemeyi zorunlu kılar. Sohbet düzenlenebilir durumda kalır: sıradan metin ve ek gönderimleri geçerli sekmenin Gateway/oturum kapsamlı tarayıcı depolamasında tutulur, yeniden bağlanmayı bekliyor olarak gösterilir ve Gateway geri döndüğünde otomatik biçimde gönderilir. Bağlantı yokken canlı denetimler ve eğik çizgi komutları kullanılamaz; ancak **Durdur**, yeniden oynatılmak üzere tam bir yerel çalıştırma kimliğini kuyruğa alabilir. Yalnızca oturuma yönelik durdurma yeniden oynatılmaz; çünkü bağlantı geri gelmeden önce bu oturumda daha yeni bir çalışma başlayabilir.

Bu tarayıcıda zaten kimlik bilgileri (yapılandırılmış bir belirteç/parola veya onaylanmış bir cihaz belirteci) bulunduğunda, ilk açılışlarda ve yeniden yüklemelerde oturum açma geçidinin kısa süre görünmesi yerine bağlantı kurulurken küçük, hareketli bir OpenClaw işareti gösterilir. Oturum açma geçidi yalnızca henüz hiçbir kimlik bilgisi depolanmadığında veya Gateway bunları etkin biçimde reddettiğinde (hatalı belirteç/parola, iptal edilmiş eşleştirme) görünür; bunlar beklemek yerine girdinizi gerektiren durumlardır.

## PWA kurulumu ve web anlık bildirimleri

Denetim Arayüzü, bir `manifest.webmanifest` ve hizmet çalışanıyla birlikte sunulur; böylece modern tarayıcılar onu bağımsız bir PWA olarak kurabilir. Web Push, sekme veya tarayıcı penceresi açık olmasa bile Gateway'in bildirimlerle kurulu PWA'yı uyandırmasına olanak tanır.

macOS uygulamasında Bildirimler ayarları sayfası, uygulama bildirimleri yerel olarak ilettiği için tarayıcı anlık bildirim izni yerine uygulamanın yerel bildirim iznini gösterir.

Sayfa bir OpenClaw güncellemesinden hemen sonra **Protokol uyuşmazlığı** gösterirse önce panoyu `openclaw dashboard` ile yeniden açın ve tam yenileme yapın. Sorun devam ederse pano kaynağının site verilerini temizleyin veya gizli bir tarayıcı penceresinde deneyin; eski bir sekme veya tarayıcı hizmet çalışanı önbelleği, güncelleme öncesi bir Denetim Arayüzü paketini daha yeni Gateway'e karşı çalıştırmaya devam edebilir.

| Yüzey                                             | İşlevi                                                                       |
| -------------------------------------------------- | ---------------------------------------------------------------------------- |
| `ui/public/manifest.webmanifest`                   | PWA manifesti. Erişilebilir olduğunda tarayıcılar "Install app" seçeneğini sunar. |
| `ui/public/sw.js`                                  | `push` olaylarını ve bildirim tıklamalarını işleyen hizmet çalışanı. |
| `state/openclaw.sqlite` → `web_push_vapid_keys`    | Web Push yüklerini imzalamak için kullanılan, otomatik oluşturulmuş VAPID anahtar çifti. |
| `state/openclaw.sqlite` → `web_push_subscriptions` | Kalıcı tarayıcı abonelik uç noktaları, anahtarları ve kayıt zaman damgaları. |

Kullanımdan kaldırılan `push/vapid-keys.json` ve `push/web-push-subscriptions.json` depolarından yapılan yükseltmeler `openclaw doctor --fix` tarafından içe aktarılır. Daha eski bir sürecin içe aktarma sırasında kullanımdan kaldırılmış durumu yeniden oluşturamaması için bu onarımı çalıştırmadan önce Gateway'i durdurun. Yükseltmeden sonra Web Push'ı kullanmadan önce onarımı çalıştırın; kullanımdan kaldırılmış kaynaklardan biri veya kesintiye uğramış bir Doctor talebi kaldığı sürece kayıt, teslimat, silme ve anahtar çözümleme işlemleri devam etmeyi reddeder. Gateway çalışma zamanı yalnızca SQLite okur ve yazar.

Anahtarları sabitlemek istediğinizde (çok sunuculu dağıtımlar, sır döndürme veya testler) Gateway sürecindeki ortam değişkenleri aracılığıyla VAPID anahtar çiftini geçersiz kılın:

- `OPENCLAW_VAPID_PUBLIC_KEY`
- `OPENCLAW_VAPID_PRIVATE_KEY`
- `OPENCLAW_VAPID_SUBJECT` (varsayılanı `https://openclaw.ai`)

Denetim Arayüzü, tarayıcı aboneliklerini kaydetmek ve test etmek için kapsamla sınırlandırılmış şu Gateway yöntemlerini kullanır:

- `push.web.vapidPublicKey` etkin VAPID ortak anahtarını getirir.
- `push.web.subscribe`, bir `endpoint` ile `keys.p256dh`/`keys.auth` değerlerini kaydeder.
- `push.web.unsubscribe` kayıtlı bir uç noktayı kaldırır.
- `push.web.test` çağıranın aboneliğine bir test bildirimi gönderir.

<Note>
Web Push, iOS APNS aktarma yolundan (aktarma destekli anlık bildirim için [Yapılandırma](/tr/gateway/configuration) bölümüne bakın) ve yerel mobil eşleştirmeyi hedefleyen `push.test` yönteminden bağımsızdır.
</Note>

## Barındırılan yerleştirmeler

Yardımcı iletileri, `[embed ...]` kısa koduyla barındırılan web içeriğini satır içinde oluşturabilir. iframe korumalı alan politikası `gateway.controlUi.embedSandbox` tarafından denetlenir:

Temel [`show_widget`](/tr/tools/show-widget) aracı, kendi kendine yeten SVG veya HTML'yi doğrudan bir araç çağrısından oluşturur. Tarayıcı ve desteklenen yerel sohbet istemcileri `inline-widgets` Gateway yeteneğini bildirir ve ortaya çıkan Canvas belgesi, sohbet geçmişi yeniden yüklendiğinde kullanılabilir kalır. Discord Activities, Discord üzerinde aynı araç adını sağlar; diğer kanallardan kaynaklanan çalıştırmalar bu aracı almaz.

<Tabs>
  <Tab title="katı">
    Barındırılan yerleştirmelerin içinde betik yürütmeyi devre dışı bırakır.
  </Tab>
  <Tab title="betikler (varsayılan)">
    Kaynak yalıtımını korurken etkileşimli yerleştirmelere izin verir; genellikle kendi kendine yeten tarayıcı oyunları/araçları için yeterlidir.
  </Tab>
  <Tab title="güvenilir">
    Bilerek daha güçlü ayrıcalıklara ihtiyaç duyan aynı site belgeleri için `allow-scripts` üzerine `allow-same-origin` ekler.
  </Tab>
</Tabs>

```json5
{
  gateway: {
    controlUi: {
      embedSandbox: "scripts",
    },
  },
}
```

<Warning>
`trusted` seçeneğini yalnızca yerleştirilmiş belge gerçekten aynı kaynak davranışına ihtiyaç duyduğunda kullanın. Aracı tarafından oluşturulan çoğu oyun ve etkileşimli tuval için `scripts` daha güvenli seçenektir.
</Warning>

Mutlak harici `http(s)` yerleştirme URL'leri varsayılan olarak engellenmiş kalır. `[embed url="https://..."]` öğesinin üçüncü taraf sayfalarını yüklemesine izin vermek için `gateway.controlUi.allowExternalEmbedUrls: true` ayarını belirleyin.

## Sohbet transkripti düzeni

Sohbet dökümü, ileti oluşturucuyla hizalanmış, ortalanmış ve okunabilir bir çerçeve kullanır. Asistan ve araç çıktıları sola hizalı kalırken kendi iletileriniz bu çerçevenin içinde sağa hizalı kalır. Çok kullanıcılı oturumlarda (örneğin bir kanal plugin'inden aktarılan grup sohbetinde), kimliği belirtilen diğer katılımcıların iletileri yazarın avatarı, adı ve kimlik başına sabit bir renkle sola hizalı olarak görüntülenir; böylece yalnızca oturum açmış görüntüleyicinin iletileri "benim" olarak algılanır. Kimliği belirtilen iki veya daha fazla katılımcı bulunduğunda, asistan yanıtları, dönüşü tetikleyen iletinin sahibini belirten küçük bir "Ad adlı kişiye yanıt veriliyor" işareti taşır. Yerel eğik çizgi komutu çıktısı gibi sistem girdileri, avatar olmadan ortalanmış bildirim satırları olarak görüntülenir.

## Sohbet iletisi genişliği

Geniş monitör kullanıcıları, döküm genişliğini **Ayarlar → Sohbet →
İleti genişliği** altında geçersiz kılabilir. Tercih, ilgili tarayıcının yerel depolamasında kalır. Desteklenen
biçimler arasında `960px` veya `82%` gibi sade uzunluklar ve yüzdelerin yanı sıra
kısıtlanmış `min(...)`, `max(...)`, `clamp(...)`, `calc(...)` ve
`fit-content(...)` genişlik ifadeleri bulunur.

## Tailnet erişimi (önerilir)

<Tabs>
  <Tab title="Tümleşik Tailscale Serve (tercih edilir)">
    Gateway'i geri döngüde tutun ve Tailscale Serve'ün HTTPS ile proxy görevi görmesini sağlayın:

    ```bash
    openclaw gateway --tailscale serve
    ```

    `https://<magicdns>/` adresini (veya yapılandırdığınız `gateway.controlUi.basePath` adresini) açın.

    Varsayılan olarak, `gateway.auth.allowTailscale` değeri `true` olduğunda Control UI/WebSocket Serve istekleri Tailscale kimlik üstbilgileri (`tailscale-user-login`) aracılığıyla kimlik doğrulayabilir. OpenClaw, `x-forwarded-for` adresini `tailscale whois` ile çözümleyip üstbilgiyle eşleştirerek kimliği doğrular ve bunları yalnızca istek Tailscale'in `x-forwarded-*` üstbilgileriyle geri döngüye ulaştığında kabul eder. Tarayıcı cihaz kimliği bulunan Control UI operatör oturumlarında, doğrulanmış bu Serve yolu cihaz eşleştirme gidiş dönüşünü de atlar; cihaz kimliği bulunmayan tarayıcılar ve node rolü bağlantıları normal cihaz kontrollerini izlemeye devam eder. Serve trafiği için bile açıkça paylaşılan gizli bilgi kimlik bilgileri gerektirmek istiyorsanız `gateway.auth.allowTailscale: false` ayarını belirleyin, ardından `gateway.auth.mode: "token"` veya `"password"` kullanın.

    Bu eşzamansız Serve kimlik yolu için, aynı istemci IP'si ve kimlik doğrulama kapsamındaki başarısız kimlik doğrulama girişimleri, hız sınırı yazımlarından önce serileştirilir. Bu nedenle aynı tarayıcıdan eşzamanlı hatalı yeniden denemelerde, paralel olarak yarışan iki sade uyuşmazlık yerine ikinci istekte `retry later` gösterilebilir.

    <Warning>
    Token'sız Serve kimlik doğrulaması, gateway ana makinesinin güvenilir olduğunu varsayar. Bu ana makinede güvenilmeyen yerel kod çalışabiliyorsa token/parola kimlik doğrulamasını zorunlu kılın.
    </Warning>

  </Tab>
  <Tab title="Tailnet'e bağla + token">
    ```bash
    openclaw gateway --bind tailnet --token "$(openssl rand -hex 32)"
    ```

    `http://<tailscale-ip>:18789/` adresini (veya yapılandırdığınız `gateway.controlUi.basePath` adresini) açın.

    Eşleşen paylaşılan gizli bilgiyi UI ayarlarına yapıştırın (`connect.params.auth.token` veya `connect.params.auth.password` olarak gönderilir).

  </Tab>
</Tabs>

## Güvenli olmayan HTTP

Kontrol panelini düz HTTP üzerinden (`http://<lan-ip>` veya `http://<tailscale-ip>`) açarsanız tarayıcı **güvenli olmayan bir bağlamda** çalışır ve WebCrypto'yu engeller. OpenClaw varsayılan olarak cihaz kimliği bulunmayan Control UI bağlantılarını **engeller**.

Desteklenen cihaz kimliği olmayan istisna, `gateway.auth.mode: "trusted-proxy"` üzerinden başarılı
Control UI operatör kimlik doğrulamasıdır. Cihaz kimliğini devre dışı bırakan kalıcı bir yapılandırma
anahtarı yoktur.

**Önerilen düzeltme:** HTTPS (Tailscale Serve) kullanın veya UI'yi yerel olarak `https://<magicdns>/` (Serve) ya da `http://127.0.0.1:18789/` (gateway ana makinesinde) adresinden açın.

<AccordionGroup>
  <Accordion title="Güvenilir proxy notu">
    - Başarılı güvenilir proxy kimlik doğrulaması, cihaz kimliği bulunmayan **operatör** Control UI oturumlarına izin verebilir.
    - Bu, node rolündeki Control UI oturumlarını **kapsamaz**.
    - Aynı ana makinedeki geri döngü ters proxy'leri yine de güvenilir proxy kimlik doğrulamasını karşılamaz; bkz. [Güvenilir proxy kimlik doğrulaması](/tr/gateway/trusted-proxy-auth).

  </Accordion>
</AccordionGroup>

HTTPS kurulum rehberliği için [Tailscale](/tr/gateway/tailscale) bölümüne bakın.

## İçerik güvenliği politikası

Control UI sıkı bir `img-src` politikasıyla sunulur: yalnızca **aynı kökenli** varlıklara, `data:` URL'lerine ve yerel olarak oluşturulan `blob:` URL'lerine izin verilir. Uzak `http(s)` ve protokole göreli görüntü URL'leri tarayıcı tarafından reddedilir ve hiçbir zaman ağ getirme isteği oluşturmaz.

Uygulamada:

- Göreli yollar altında sunulan avatarlar ve görüntüler (örneğin `/avatars/<id>`), UI'nin getirip yerel `blob:` URL'lerine dönüştürdüğü kimliği doğrulanmış avatar rotaları da dahil olmak üzere görüntülenmeye devam eder.
- Satır içi `data:image/...` URL'leri görüntülenmeye devam eder.
- Control UI tarafından oluşturulan yerel `blob:` URL'leri görüntülenmeye devam eder.
- GitHub bağlantı önizleme avatarları, Gateway tarafından GitHub'ın sabit avatar ana makinesinden getirilir ve sınırlandırılmış `data:` URL'leri olarak döndürülür; operatör tarayıcısı uzak avatar ana makinesiyle hiçbir zaman iletişim kurmaz.
- Kanal meta verilerinin ürettiği uzak avatar URL'leri, Control UI'nin avatar yardımcılarında kaldırılıp yerleşik logo/rozetle değiştirilir; böylece ele geçirilmiş veya kötü amaçlı bir kanal, operatör tarayıcısını rastgele uzak görüntü getirme istekleri yapmaya zorlayamaz.

Bu her zaman etkindir ve yapılandırılamaz.

## Avatar rotası kimlik doğrulaması

Gateway kimlik doğrulaması yapılandırıldığında, Control UI avatar uç noktası API'nin geri kalanıyla aynı gateway token'ını gerektirir:

- `GET /avatar/<agentId>`, avatar görüntüsünü yalnızca kimliği doğrulanmış çağıranlara döndürür. `GET /avatar/<agentId>?meta=1`, aynı kurala göre avatar meta verilerini döndürür.
- Her iki rotaya yapılan kimliği doğrulanmamış istekler reddedilir (kardeş asistan medya rotasıyla eşleşecek şekilde); böylece avatar rotası, diğer yönlerden korunan ana makinelerde ajan kimliğini sızdıramaz.
- Control UI, avatarları getirirken gateway token'ını bearer üstbilgisi olarak iletir ve görüntünün kontrol panellerinde görüntülenmeye devam etmesi için kimliği doğrulanmış blob URL'leri kullanır.

Gateway kimlik doğrulamasını devre dışı bırakırsanız (paylaşılan ana makinelerde önerilmez), gateway'in geri kalanıyla uyumlu olarak avatar rotası da kimlik doğrulamasız hâle gelir.

## Asistan medya rotası kimlik doğrulaması

Gateway kimlik doğrulaması yapılandırıldığında, asistanın yerel medya önizlemeleri iki adımlı bir rota kullanır:

- `GET /__openclaw__/assistant-media?meta=1&source=<path>`, normal Control UI operatör kimlik doğrulamasını gerektirir; tarayıcı, kullanılabilirliği denetlerken gateway token'ını bearer üstbilgisi olarak gönderir.
- Başarılı meta veri yanıtları, tam olarak ilgili kaynak yoluyla kapsamlandırılmış kısa ömürlü bir `mediaTicket` içerir.
- Tarayıcıda görüntülenen görüntü, ses, video ve belge URL'leri, etkin gateway token'ı veya parolası yerine `mediaTicket=<ticket>` kullanır. Biletin süresi hızla dolar ve farklı bir kaynağı yetkilendiremez.

Bu, yeniden kullanılabilir gateway kimlik bilgilerini görünür medya URL'lerine koymadan medya görüntülemeyi tarayıcıya özgü medya öğeleriyle uyumlu tutar.

## Onay bağlantıları

Operatör onay bildirimleri, ayrılmış `${controlUiBasePath}/approve/{approvalId}` ad alanı altında sunulan bağımsız bir onay belgesine derin bağlantı verebilir (örneğin `/approve/<approvalId>` veya yapılandırılmış bir temel yolla `/openclaw/approve/<approvalId>`). URL, onayın ömrü boyunca sabittir ve kendi cihazlarınız arasında güvenle iletilebilir: onayı tanımlar, hiçbir zaman yetkilendirme sağlamaz.

- Tek segmentli `/approve/<approvalId>` ad alanı, **tüm** HTTP yöntemleri için plugin HTTP rotalarından önce Gateway tarafından ayrılır; böylece bir plugin rotası hiçbir zaman onay belgesini gölgeleyemez veya engelleyemez.
- Bir onay belgesini açmak, Control UI'nin geri kalanıyla aynı gateway kimlik doğrulamasını (token/parola, Tailscale Serve kimliği veya güvenilir proxy kimliği) gerektirir; kimlik bilgileri hiçbir zaman onay URL'sinin parçası olmaz.
- Control UI sunumu devre dışı bırakıldığında, ad alanına yapılan istekler plugin işleyicilerine geçmek yerine `404` döndürür.
- Bir onay belgesinde oturum açılması yalnızca ilgili sayfa için geçicidir: aynı tarayıcıda tam Control UI tarafından kaydedilen gateway seçiminin veya ayarların üzerine yazmaz.

Gateway, statik dosyaları `dist/control-ui` konumundan sunar:

```bash
pnpm ui:build
```

İsteğe bağlı mutlak taban (sabit varlık URL'leri):

```bash
OPENCLAW_CONTROL_UI_BASE_PATH=/openclaw/ pnpm ui:build
```

Yerel geliştirme (ayrı geliştirme sunucusu):

```bash
pnpm ui:dev
```

Ardından UI'yi Gateway WS URL'nize yönlendirin (ör. `ws://127.0.0.1:18789`).

## Boş Control UI sayfası

Tarayıcı boş bir kontrol paneli yüklüyor ve DevTools yararlı bir hata göstermiyorsa bir uzantı veya erken çalışan içerik betiği, JavaScript modül uygulamasının değerlendirilmesini engellemiş olabilir. Statik sayfa, başlangıçtan sonra `<openclaw-app>` kaydedilmemişse görünen sade bir HTML kurtarma paneli içerir.

Tarayıcı ortamını değiştirdikten sonra paneldeki **Tekrar dene** eylemini kullanın veya şu kontrollerden sonra elle yeniden yükleyin:

- Tüm sayfalara içerik ekleyen uzantıları, özellikle `<all_urls>` içerik betiklerine sahip uzantıları devre dışı bırakın.
- Gizli bir pencereyi, temiz bir tarayıcı profilini veya başka bir tarayıcıyı deneyin.
- Gateway'i çalışır durumda tutun ve tarayıcı değişikliğinden sonra aynı kontrol paneli URL'sini doğrulayın.

## Hata ayıklama/test: geliştirme sunucusu + uzak Gateway

Control UI statik dosyalardan oluşur; WebSocket hedefi yapılandırılabilir ve HTTP kökeninden farklı olabilir. Vite geliştirme sunucusunu yerel olarak kullanmak, ancak Gateway'i başka bir yerde çalıştırmak istediğinizde bu kullanışlıdır.

<Steps>
  <Step title="UI geliştirme sunucusunu başlatın">
    ```bash
    pnpm ui:dev
    ```
  </Step>
  <Step title="gatewayUrl ile açın">
    ```text
    http://localhost:5173/?gatewayUrl=ws%3A%2F%2F<gateway-host>%3A18789
    ```

    İsteğe bağlı tek seferlik kimlik doğrulama (gerekirse):

    ```text
    http://localhost:5173/?gatewayUrl=wss%3A%2F%2F<gateway-host>%3A18789#token=<gateway-token>
    ```

  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Notlar">
    - `gatewayUrl`, yüklemeden sonra localStorage'da saklanır ve URL'den kaldırılır.
    - `gatewayUrl` aracılığıyla tam bir `ws://` veya `wss://` uç noktası geçirirseniz tarayıcının sorgu dizesini doğru ayrıştırması için değeri URL olarak kodlayın.
    - `token`, mümkün olduğunda URL parçası (`#token=...`) aracılığıyla geçirilmelidir. Parçalar sunucuya gönderilmez; bu da istek günlüğüne ve Referer'a sızmayı önler. Eski `?token=` sorgu parametreleri uyumluluk amacıyla bir kez içe aktarılmaya devam eder, ancak yalnızca geri dönüş olarak kullanılır ve önyüklemeden hemen sonra kaldırılır.
    - `password` yalnızca bellekte tutulur.
    - `gatewayUrl` ayarlandığında UI, yapılandırma veya ortam kimlik bilgilerine geri dönmez. `token` (veya `password`) değerini açıkça sağlayın; açık kimlik bilgilerinin eksik olması hatadır.
    - Gateway TLS arkasındayken (Tailscale Serve, HTTPS proxy vb.) `wss://` kullanın.
    - `gatewayUrl`, tıklama tuzağını önlemek için yalnızca üst düzey bir pencerede (gömülü değilken) kabul edilir.
    - Geri döngü dışında herkese açık Control UI dağıtımları, `gateway.controlUi.allowedOrigins` değerini açıkça (tam kökenler olarak) ayarlamalıdır. Geri döngüden, RFC1918/bağlantı yerelinden, `.local`, `.ts.net` veya Tailscale CGNAT ana makinelerinden yapılan aynı kökenli özel LAN/Tailnet yüklemeleri, Host üstbilgisi geri dönüşü etkinleştirilmeden kabul edilir.
    - Gateway başlangıcı, etkili çalışma zamanı bağlantısı ve bağlantı noktasından `http://localhost:<port>` ve `http://127.0.0.1:<port>` gibi yerel kökenleri başlangıçta ekleyebilir; ancak uzak tarayıcı kökenleri yine de açık girdiler gerektirir.
    - Sıkı denetlenen yerel testler dışında `gateway.controlUi.allowedOrigins: ["*"]` kullanmayın; bu, "kullandığım ana makineyle eşleştir" değil, tüm tarayıcı kökenlerine izin ver anlamına gelir.
    - `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true`, Host üstbilgisi köken geri dönüşü modunu etkinleştirir, ancak bu tehlikeli bir güvenlik modudur.

  </Accordion>
</AccordionGroup>

```json5
{
  gateway: {
    controlUi: {
      allowedOrigins: ["http://localhost:5173"],
    },
  },
}
```

Uzaktan erişim kurulumu ayrıntıları: [Uzaktan erişim](/tr/gateway/remote).

## İlgili

- [Gösterge Paneli](/tr/web/dashboard) — gateway gösterge paneli
- [Durum Denetimleri](/tr/gateway/health) — gateway durum izleme
- [TUI](/tr/web/tui) — terminal kullanıcı arayüzü
- [WebChat](/tr/web/webchat) — tarayıcı tabanlı sohbet arayüzü
