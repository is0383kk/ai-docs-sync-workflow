---
read_when:
    - Bir ajanın, telefonunuzdan oturum açılmış gerçek Chrome tarayıcınızı yönetmesini istiyorsunuz
    - Masanın başında kimse yokken Chrome'un "Allow remote debugging?" istemiyle sürekli karşılaşıyorsunuz
    - Uzantı aracılığıyla tarayıcının ele geçirilmesine ilişkin güvenlik modelini anlamak istiyorsunuz
summary: 'Chrome uzantısı: OpenClaw''ın uzaktan hata ayıklama istemi olmadan oturum açtığınız Chrome''u yönetmesini sağlayın'
title: Chrome Uzantısı
x-i18n:
    generated_at: "2026-07-27T00:19:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3d974f62bb5697a23dd6a6852137ce6af5a8a4a2a8ff738eec0098f259e8faa0
    source_path: tools/chrome-extension.md
    workflow: 16
---

# Chrome uzantısı

OpenClaw Chrome uzantısı, bir aracının ayrı bir yönetilen tarayıcı başlatmadan ve Chrome'un engelleyici "Allow remote debugging?" istemi **olmadan**, **oturum açtığınız Chrome sekmelerini** denetlemesini sağlar.

Bu, OpenClaw'u bir telefondan (Telegram, WhatsApp vb.) kullandığınızda önemlidir:
[`user` profili](/tr/tools/browser#profiles-openclaw-user-chrome), Chrome'un uzaktan hata ayıklama bağlantı noktası üzerinden bağlanır; bu da uzaktayken kimsenin tıklayamayacağı bir masaüstü onay iletişim kutusu açar. Uzantı bunun yerine `chrome.debugger` API'sini kullanır; dolayısıyla sayfa içindeki tek belirti, Chrome'un kapatılabilir "OpenClaw started debugging this browser" banner'ıdır.

Bu, Anthropic'in Claude in Chrome ve OpenAI'ın Codex Chrome uzantılarının kullandığı yapıyla aynıdır.

## Nasıl çalışır?

Üç bölümden oluşur:

- **Tarayıcı denetim hizmeti** (Gateway veya node ana makinesi): `browser` aracının çağırdığı API.
- **Uzantı aktarıcısı** (geri döngü WebSocket'i): denetim hizmetinin `127.0.0.1` üzerinde başlattığı küçük bir sunucu. OpenClaw'a bir Chrome DevTools Protocol uç noktası sunar ve uzantıyla iletişim kurar. Her iki taraf da ana makineye yerel bir token ile kimlik doğrular (aşağıya bakın).
- **OpenClaw Chrome uzantısı** (MV3): `chrome.debugger` ile sekmelere bağlanır, CDP trafiğini iletir ve **OpenClaw sekme grubunu** yönetir.

OpenClaw yalnızca **OpenClaw sekme grubundaki** sekmeleri görür ve denetler. Grup, onay sınırıdır: paylaşmak için bir sekmeyi grubun içine, erişimi anında iptal etmek için dışına sürükleyin (veya araç çubuğu düğmesine tıklayın).

## Yükleme ve eşleştirme

1. Paketlenmemiş uzantının yolunu yazdırın:

   ```bash
   openclaw browser extension path
   ```

2. `chrome://extensions` sayfasını açın, **Developer mode** seçeneğini etkinleştirin, **Load
   unpacked** düğmesine tıklayın ve yazdırılan dizini seçin.

3. Eşleştirme dizesini yazdırın:

   ```bash
   openclaw browser extension pair
   ```

4. OpenClaw araç çubuğu simgesine tıklayın ve eşleştirme dizesini açılır pencereye yapıştırın.
   Uzantı aktarıcıya bağlandığında rozet **ON** durumuna geçer.

Eşleştirme token'ı, ilk kullanımda oluşturulan ve durum dizinindeki `credentials/` altında saklanan (mod `0600`) **ana makineye yerel bir gizli bilgidir**. Tarayıcı çalıştıran her makine — Gateway ana makinesi ve her tarayıcı node ana makinesi — kendi token'ına sahiptir; böylece kimlik bilgilerinin makineler arasında taşınması gerekmez. Token'ı döndürmek için `browser-extension-relay.secret` dosyasını silip yeniden eşleştirin.

## Kullanım

Bir `browser` aracı çağrısında yerleşik `chrome` profilini seçin veya varsayılan yapın:

```bash
openclaw config set browser.defaultProfile chrome
```

```json5
{
  browser: {
    profiles: {
      chrome: { driver: "extension", color: "#FF4500" },
    },
  },
}
```

- Bir sekmeyi paylaşın: o sekmedeki OpenClaw araç çubuğu düğmesine tıklayın (sekme OpenClaw sekme grubuna katılır) veya herhangi bir sekmeyi grubun içine sürükleyin.
- Aracı ayrıca yeni sekmeler de açabilir; bunlar gruba otomatik olarak eklenir.
- Erişimi iptal edin: düğmeye yeniden tıklayın, sekmeyi grubun dışına sürükleyin veya Chrome'un hata ayıklama banner'ını kapatın. Aracı, o sekmeye erişimini anında kaybeder.

### Sekme yardımcı pilotu yan paneli

Uzantıyı eşleştirdikten sonra araç çubuğu açılır penceresindeki **Open tab copilot** seçeneğine tıklayın.
OpenClaw, `sidepanel.html` öğesini tam olarak o Chrome sekmesi için yapılandırır; manifestte genel bir yan panel yolu yoktur. Bu nedenle her sekme ayrı bir panel belgesi, Gateway oturumu, mesaj aboneliği ve türü belirlenmiş tarayıcı aracı bağlaması alır.

Panel; sayfa URL'sini, başlığını, DOM'u veya görünür metni mesajınıza eklemez. Yalnızca yazdığınız metni gönderir. Tarayıcı eylemleri, Chrome sekmesini ve CDP hedefini içeren, Gateway tarafından kimliği doğrulanmış ayrı bir bağlama taşır; tarayıcı aracı bu hedefi değiştirme veya tarayıcı genelindeki eylemleri kullanma girişimlerini reddeder. Yanıtlar panelde (`deliver: false`) kalır; Telegram, Discord veya başka bir kanal rotasını devralmaz.

Yardımcı pilot, `operator.read` ve `operator.write` kapsamlarına sahip, eşleştirilmiş özel bir Gateway cihazıdır. İlk kullanımda isteğini inceleyip onaylayın:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Uzantı, bu cihaz kimliğini ve Gateway tarafından verilen cihaz token'ını, bunları veren standart Gateway uç noktası kapsamında saklar. Farklı bir Gateway'i eşleştirmek ayrı kimlik, token ve oturum gözetimi oluşturur; kimlik bilgileri ve oturumlar uç noktalar arasında hiçbir zaman yeniden kullanılmaz. Uzantı, Gateway paylaşılan gizli bilgisini kalıcı olarak saklamaz. Bir panel yalnızca kendi sekme oturumlarına abone olabilir ve Gateway bu olayları teslimattan önce filtreler.

Bir çalıştırma sırasında Gateway bağlantısı kesilirse uzantı, söz konusu çalıştırma kimliğinin kalıcı gözetimini korur. Yeniden bağlandığında herhangi bir paneli yeniden etkinleştirmeden önce çözümlenmemiş çalıştırmayı iptal eder, ardından transkript geçmişini yeniden yükler. Bu kapalı durumda başarısız olma adımı, bir teslimat kesintisi boyunca tarayıcı eylemlerinin görünmeden devam etmesini önler.

Bir sekmenin kapatılması canlı aboneliğini anında kaldırır, görünür tüm çalıştırmaları iptal eder ve o sekmenin oturumunu arşivlenmiş olarak işaretler. Gateway geçici olarak çevrimdışıysa uzantı, bekleyen arşivleme işlemini kalıcı olarak saklar ve yalnızca aynı Gateway uç noktası yeniden bağlandığında tekrar dener; arşivleme isteğini hiçbir zaman farklı bir Gateway'e göndermez. Bir tarayıcı çökmesinden sonra bir sonraki başlatma, önceki tarayıcı örneğinin bıraktığı oturumları arşivler. Arşivlenmiş oturumlar yeni işleri reddederken transkriptleri oturum geçmişinde kullanılabilir durumda kalır. Tarayıcı yardımcı pilotu anahtarları ileti dizisi oturumlarıdır; dolayısıyla normal yaş ve girdi sayısı bakımı bunları korur. Aracı başına oturum disk bütçesi yine geçerlidir (varsayılan `2gb`) ve baskı altında en eski oturumları çıkarabilir; [oturum bakımına](/tr/reference/session-management-compaction#store-maintenance-and-disk-controls) bakın.

Yan panel şu anda Gateway tarafından barındırılan bir uzantı aktarıcısı veya doğrudan uzak Gateway aktarıcısı gerektirir. Tarayıcı node'u üzerindeki geri döngü aktarıcısı, türü belirlenmiş sekme bağlamasının gerektirdiği node rotasını henüz sağlayamaz; bu nedenle panel, tarayıcı genelinde yönlendirmeye geri dönmek yerine bu topolojiyi reddeder.

## OpenClaw'a sayfa gönderme

Okunabilir sayfa metnini ana OpenClaw oturumunuzla paylaşmak için araç çubuğu açılır penceresindeki **Send page to OpenClaw** seçeneğini kullanın. İsteğe bağlı bir not ekleyebilir, sayfa veya seçim sağ tıklama menüsünü kullanabilir ya da `Alt+Shift+S` tuşlarına basabilirsiniz. OpenClaw, varsa geçerli seçiminizi tercih eder, paylaşımı bir sistem olayı olarak kuyruğa ekler ve ana oturumu hemen uyandırır.

Sekmenin OpenClaw sekme grubunda olması gerekmez. Bu, tek seferlik ve açık bir paylaşımdır: sayfadaki başka hiçbir şey açığa çıkarılmaz ve devam eden erişim izni verilmez. Google Docs, Google API kurulumu olmadan, oturum açtığınız tarayıcı oturumu kullanılarak düz metin olarak dışa aktarılır. X ve Twitter ileti dizileri, çevrelerindeki arayüz öğeleri olmadan ayıklanır.

Sayfa metni, OpenClaw'un harici içerik güvenliği sınırı içine alınır. İsteğe bağlı notunuz, kendi talimatınız olarak bu sınırın dışında kalır. Sayfa metni ve seçimler yaklaşık 120.000 karakterle sınırlandırılır ve kısaltıldıklarında bir kesilme işareti içerir.

Sayfa paylaşımı, uzantı aktarıcısı Gateway tarafından barındırıldığında, aynı ana makine eşleştirmesi veya doğrudan `wss://` Gateway eşleştirmesi kullanılarak çalışır. Node tarafından barındırılan aktarıcılar şimdilik açık bir hata döndürür. Klavye kısayolunu yeniden eşlemek için `chrome://extensions/shortcuts` sayfasını açın.

## Uzak / makineler arası

Chrome'un Gateway ana makinesinde çalışması gerekmez. Üç topoloji çalışır:

- **Aynı ana makine** (tek makinede Gateway + Chrome): o makinede `openclaw browser extension pair` ile eşleştirin. Aktarıcı yalnızca geri döngüdedir.
  Yerel Gateway TLS kullanıyorsa sertifikanın ana makine adını `--gateway-url wss://gateway-host.example` ile açıkça iletin; eşleştirme hiçbir zaman yerine bir geri döngü IP'si koymaz.
- **Doğrudan uzak bir Gateway'e** (dizüstü bilgisayarınızda Chrome, bir VPS'de Gateway ve
  **dizüstü bilgisayarda başka hiçbir şey yok**): Gateway üzerinde
  `openclaw browser extension pair --gateway-url wss://your-gateway.example.com` komutunu çalıştırın.
  Bu, bir `wss://…/browser/extension#<secret>` dizesi yazdırır; uzantıyı dizüstü bilgisayara yükleyip eşleştirin. Uzantı, `wss://` üzerinden **doğrudan Gateway'e** bağlanır — dizüstü bilgisayarda OpenClaw kurulumu, Node, CLI veya açık gelen bağlantı noktası gerekmez. Bu, yönetilen barındırma yoludur.
- **Bir tarayıcı node ana makinesi üzerinden** (OpenClaw node'u çalıştıran bir makinede Chrome): node üzerinde `pair` komutunu çalıştırıp yerel olarak eşleştirin; Gateway, tarayıcı eylemlerini mevcut kimliği doğrulanmış node bağlantısı üzerinden node'a vekâleten iletir.

Eşleştirme gizli bilgisi ana makine başınadır (doğrudan durumda Gateway'in gizli bilgisi) ve Gateway'in `/browser/extension` rotası tarafından doğrulanır. Doğrudan yol için Gateway'i TLS (`wss://`) üzerinden sunarak eşleştirme gizli bilgisinin ve CDP trafiğinin şifrelenmesini sağlayın. Gizli bilgi, eşleştirme dizesinin URL parçasında kalır ve WebSocket el sıkışması sırasında bir alt protokol kimlik bilgisi olarak sunulur; böylece normal proxy erişim günlükleri bunu istek URL'sinde almaz. Ters proxy'lerin standart `Sec-WebSocket-Protocol` üst bilgisini koruduğundan emin olun.

## Tanılama

```bash
openclaw browser status --browser-profile chrome
openclaw browser doctor --browser-profile chrome
```

`doctor`, uzantı açılır penceresi **Connected** gösterene kadar **Chrome uzantısı aktarıcısı** denetimini başarısız olarak bildirir.

## Güvenlik modeli

- Aktarıcı yalnızca geri döngüye bağlanır; her iki WebSocket tarafının kimliği türetilmiş token ile doğrulanır ve uzantı tarafının kaynağının `chrome-extension://` olduğu denetlenir.
- Doğrudan Gateway eşleştirmesi, aktarıcı token'ını istek URL'sinde kabul etmez; paketlenmiş uzantı bunun yerine token'ı WebSocket alt protokol listesinde taşır.
- Aracı yalnızca **OpenClaw sekme grubundaki** sekmeleri görebilir ve kullanabilir. Diğer sekmeleriniz gizli kalır.
- Yan panel çalıştırmaları iki kez kapsamlandırılır: Gateway teslimatı oturum başına bir izin verilenler listesi kullanır ve tarayıcı araçları istemin dışında taşınan Chrome sekmesi/hedef bağlamasını uygular.
- Uzaktan hata ayıklama istemini onayladığınızda oturum açtığınız tarayıcının tamamını açığa çıkaran `user` (Chrome MCP) profiliyle karşılaştırıldığında uzantı, paylaşılan yüzeyi bir bakışta denetleyebileceğiniz bir sekme grubuyla sınırlar.

Ayrıca bakın: tam profil modeli ile yönetilen `openclaw` ve Chrome MCP `user` profilleri için [Tarayıcı](/tr/tools/browser).
