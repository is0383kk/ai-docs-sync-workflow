---
read_when:
    - Mac WebChat görünümünde veya geri döngü bağlantı noktasında hata ayıklama
summary: Mac uygulamasının Gateway WebChat'i nasıl yerleşik olarak sunduğu ve hata ayıklama yöntemleri
title: WebChat (macOS)
x-i18n:
    generated_at: "2026-07-26T23:49:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b5e5983954e12d8546a01d089eda54e7eb0c60b4c92eff670f91797cd022c9fd
    source_path: platforms/mac/webchat.md
    workflow: 16
---

macOS menü çubuğu uygulaması, WebChat kullanıcı arayüzünü yerel bir SwiftUI görünümü olarak içerir. Gateway'e bağlanır ve seçilen agent için birincil oturumu varsayılan olarak kullanır (`main` veya `session.scope`, `global` olduğunda `global`).

Tam sohbet penceresi, yerel bir bölünmüş görünümdür:

- **Oturumlar kenar çubuğu**: sabitlenmiş, Gateway destekli grup ve son kullanılanlar bölümlerini içeren, aranabilir oturum listesi. Oluşturulan alt oturumlar, her bölümde üst oturumlarının altında iç içe yer alır; daraltılmış üst oturumlar çalışan, başarısız ve okunmamış alt oturumları özetler. Bağlam menüleri; oturum bilgilerini görüntülemeyi, yeniden adlandırmayı, sabitlemeyi, çatallamayı, okundu/okunmadı olarak işaretlemeyi, arşivlemeyi/geri yüklemeyi, oturum anahtarını kopyalamayı ve silmeyi destekler. Birincil yeni oturum eylemi (veya Shift-Cmd-N), `sessions.create` aracılığıyla hemen oluşturur; yanındaki seçenekler açılır penceresinden bir agent seçilebilir ve isteğe bağlı bir temel referansla yönetilen bir çalışma ağacı talep edilebilir.
- **Pencere araç çubuğu**: bağlam kullanımı halkası (belirteçler ve oturum maliyeti, kompakt bir eylemle), model denetimleri ve oturum eylemleri menüsü. Modeller, varsayılan sağlayıcı ilk sırada olacak şekilde sağlayıcıya göre gruplandırılırken sabitlenmiş ve son kullanılan modeller en üstte kalır. Denetimler, modelin düşünme düzeyini devralabilir veya geçersiz kılabilir, araç çağrısı ayrıntı düzeyini seçebilir ve Hızlı yanıtları açıp kapatabilir. Menü, geçerli oturumu yeniden adlandırabilir veya çatallayabilir ve sabitlenme, okunma veya arşivlenme durumunu güncelleyebilir. **Oturumlar…** (Shift-Cmd-S), Gateway araması, grup yönetimi, oturum inceleme, yeniden adlandırma, sabitleme, arşivleme ve geri yükleme için Etkin/Arşivlenmiş yöneticisini açar. Seçim modu, tek tek hataları görünür tutarken birden fazla etkin oturuma sabitleme, sabitlemeyi kaldırma, arşivleme veya silme uygular. Ayrı menü onay işaretleri, assistant akıl yürütmesini ve araç etkinliğini gösterir veya gizler; her ikisi de varsayılan olarak açıktır ve başlatmalar arasında hatırlanır.
- **Döküm ve oluşturucu**: assistant mesajları avatarla birlikte düz metin, kullanıcı mesajları ise vurgu renkli baloncuklar olarak görüntülenir. Bekleyen agent soruları; tek veya çoklu seçim seçenekleri, serbest metin **Diğer** yanıtları, sona erme geri sayımları ve paylaşılan terminal durumuyla yerel kartlar olarak görüntülenir. Boş sohbetler masaüstü başlangıç istemleri sunar. `/` yazıldığında, `commands.list` destekli eğik çizgi komutu otomatik tamamlama açılır; ok/Tab/Return/Escape tuşlarıyla klavye gezintisi sağlanır. Gizli akıl yürütmeyi içermeyen görünür Markdown'ını kopyalamak için bir mesaja sağ tıklayın. Kesilmiş assistant mesajları ayrıca seçilebilir bir Markdown okuyucusu yükleyen **Tam Mesajı Aç** seçeneğini sunar. Yerel konuşma yedeğiyle Gateway TTS için **Dinle** seçeneğini kullanın.
- **Ses denetimleri**: oluşturucu, menü çubuğu katmanını değiştirmeden mevcut macOS Konuşma Modu'nu başlatabilir veya durdurabilir. Konuşma Modu etkinken oluşturucu; dinleme/düşünme/konuşma durumunu, canlı ses etkinliğini ve genişletilebilir kayan dökümü gösterir. **System Default** veya bağlı bir mikrofon seçmek için Konuş düğmesine sağ tıklayın; bu, Sesle Uyandırma ve bas-konuş tarafından kullanılan mikrofon seçimiyle aynıdır. Seçilen mikrofonun bağlantısı kesilirse etkin Konuşma oturumu sistem varsayılanına geri döner ve Konuşma Modu bir sonraki başlatıldığında seçimi yeniden dener. Konuşma Modu ses yakalamayı kullanmıyorsa ayrı bir mikrofon eylemi sesli not kaydeder.

Menü çubuğundaki sabitlenmiş kompakt sohbet paneli; aynı model, düşünme, ayrıntı düzeyi ve Hızlı denetimleri satır içinde olacak şekilde kompakt tek sütunlu düzeni, ayrıca başlangıç istemlerini, Konuşma Modu'nu, sesli notları ve Dinle seçeneğini korur. Assistant akıl yürütmesi ve araç etkinliği bu kompakt yüzeyde gizli kalır.

## Birden Fazla Gateway penceresi

Yeniden kullanılabilir Gateway profilleri eklemek veya kaldırmak için **Settings → Gateways** bölümünü açın. Her
profil, bir özel ağ `ws://` veya güvenli `wss://` uç noktası ile
isteğe bağlı belirtecini ya da parolasını içerir; kimlik bilgileri macOS Anahtar Zinciri'nde saklanır.
Güvenli profiller, sistem güveniyle denetlenen kendi ilk kullanım sertifika sabitlemelerini
korur ve birincil Gateway'den `gateway.remote.tlsFingerprint` devralmaz.
Bir profil kaldırıldığında açık pencereleri de kapatılır ve ikincil
bağlantısı sonlandırılır.

**File → New Gateway Window…** seçeneğini belirleyin veya Cmd-N tuşlarına basın, ardından bu
kayıtlı profillerden birini seçin. Seçici, en son kullanılan profili hatırlar. Her
seçim yeni ve bağımsız bir pencere oluşturur; böylece aynı Gateway, farklı etkin
oturumlar ve gezinme durumlarıyla birden fazla pencerede görünebilir.

Her kayıtlı profil; bir paylaşılan Gateway bağlantısına, cihaz kimlik doğrulama kapsamına,
döküm önbelleğine, çevrimdışı giden kutusuna ve rota kiralarına sahiptir. O profile ait pencereler,
bağımsız olarak gezilebilir kalırken bu kaynakları yeniden kullanır. Farklı
profillere ait pencereler bağlı kalır ve sohbetleri eşzamanlı olarak çalıştırır.

Menü çubuğu uygulamasında yapılandırılan Gateway, Mac Node
yeteneklerinin ve Konuşma Modu'nun sahibi olarak kalır. Ek Gateway pencereleri yalnızca operatöre yöneliktir; böylece
ikinci bir Gateway, genel mikrofon veya cihaz denetimlerini sessizce yeniden hedefleyemez.
Dinle/TTS ve normal sohbet eylemleri, pencerenin kendi Gateway bağlantısını kullanır.

## Hızlı Sohbet çubuğu

Ana oturum için kayan bir oluşturucu açmak üzere Option-Space (⌥Space) tuşlarına basın veya menü çubuğu menüsünden **Hızlı Sohbet** seçeneğini belirleyin. Genel kısayolu **Settings → General → Quick Chat shortcut** bölümündeki kaydediciyle değiştirin.

Hızlı Sohbet, hedeflenen agent'ı (yer tutucu olarak agent'ın adıyla birlikte avatar veya emoji) gösterir ve bu agent'ın ana oturumuna gönderir. Return tuşuyla gönderim kabul edildikten sonra çubuk açık kalır ve akış halinde gelen Markdown yanıtı ile son dökümü göstermek üzere aşağı doğru genişler. Çubuk girdisi, oluşturucu olarak kalır. Göndermek ve aynı hedefi tam sohbet penceresinde açmak için Command-Return, yeni satır için Shift-Return tuşlarına basın ya da çubuğun tamamını ve yanıt alanını kapatmak için Escape tuşuna basın. Dışarıya tıklamak da pencereyi kapatır. İlgili macOS izinleri eksik olduğunda iliştirilmiş bir şerit **İzin Ver** ve **Şimdi değil** eylemlerini sunar.

Oluşturucuya dikte etmek için mikrofon düğmesini kullanın. Kısmi konuşma sonuçları, oluşturucuda önceden bulunan metni korurken dikte edilen bölümü canlı olarak değiştirir. Durdurmak için düğmeye tekrar basın ya da Return veya Escape tuşuna basın; Hızlı Sohbet'i göndermek, gizlemek veya odağından çıkarmak da mikrofonu serbest bırakır. İlk kullanımda macOS Mikrofon ve Konuşma Tanıma erişimi istenir. Hızlı Sohbet, Apple Speech kullanır ve ağ hizmetlerinden yararlanabilir; yalnızca pasif Sesle Uyandırma, cihaz üzerinde tanıma gerektirir.

Kompakt model denetimi, hedef oturumun geçerli modelini ve akıl yürütme düzeyini gösterir. Model seçimi bu oturumu günceller ve dolayısıyla orada kalıcı olurken akıl yürütme seçimi yalnızca geçerli Hızlı Sohbet sunumundan gönderilen her mesaja uygulanır. Çubuk gizlendiğinde yerel seçimler sıfırlanır. Agent'lar arasında geçiş yapmak veya son kullanılan bir oturumu seçmek, açık seçimleri korur ancak yeni hedeflenen oturumun temel model durumunu yeniden yükler.

Son güncellenen beş oturum arasından seçim yapmak veya **&lt;agent&gt; için yeni mesaj** seçeneğine dönmek için geçmiş düğmesine tıklayın. Son kullanılanlardan yapılan bir seçim, tam olarak o oturuma gönderir ve yer tutucuyu **&lt;session&gt; oturumunda yanıtla** olarak değiştirir. Hızlı Sohbet'i gizlemek, bu geçici hedefi seçili agent'ın ana oturumuna sıfırlar; avatar menüsünden agent değiştirmek de hedefi temizler.

Command-Return, oturum kapsamı genel olduğunda da gönderimi alan agent'ın konuşmasını açar.

Kamera düğmesi, **Capture Window…** veya **Capture Area…** seçeneklerini içeren bir menü açar. Pencere yakalama, görünür her pencereyi etiketler; alan yakalama ise bir bölgeyi sürüklerken her ekranı karartır ve bölgenin canlı boyutunu gösterir. Seçilen ekran görüntüsü, yazılan metin varsa açıklama olarak eklenip seçilen agent'a gönderilir. İlk kullanımda macOS Ekran Kaydı erişimi istenir. Escape tuşuna basmak, boş alana tıklamak veya anlamlı bir alan sürüklemeden tıklamak işlemi iptal eder.

Odaktaki uygulamanın odaklanmış penceresinden metin eklemek için belge-metin düğmesini kullanın. Hızlı Sohbet, sonucu yakalanan metni oluşturucuya yerleştirmek yerine kaldırılabilir bir bağlam çipi olarak gösterir; gönderim, çipteki metni giden mesaja ekler ve ardından çipi temizler. Bunun için macOS Erişilebilirlik izni gerekir. Eklenen metin, Hızlı Sohbet her kapandığında da temizlenir; böylece bir sunumdaki bağlam sonraki bir gönderime sızamaz.

Yanıt tamamlandıktan sonra görünür assistant metnini, gizli akıl yürütme hariç olmak üzere genel panoya kopyalamak ve ön planda olan uygulamaya yapıştırmak için **&lt;app&gt; uygulamasına yapıştır** seçeneğini belirleyin. Bunun için macOS Erişilebilirlik izni gerekir. Eylem, geçerli pano içeriğini değiştirir ve ardından Hızlı Sohbet'i gizler.

Özelliği **Settings → General → Quick Chat** ile tamamen devre dışı bırakın; kısayol kaydedici de aynı bölümdedir.

- **Yerel mod**: doğrudan yerel Gateway WebSocket'e bağlanır.
- **Uzak mod**: veri düzlemi olarak yapılandırılmış doğrudan `ws://`/`wss://` rotasını veya uygulama tarafından yönetilen SSH tünelini kullanır.

## Başlatma ve hata ayıklama

- Elle: Lobster menüsü -> "Sohbeti Aç".
- Test için otomatik açma:

  ```bash
  dist/OpenClaw.app/Contents/MacOS/OpenClaw --chat
  ```

  (`--webchat` eski bir diğer ad olarak kabul edilir.)

- Günlükler: `./scripts/clawlog.sh` (alt sistem `ai.openclaw`, kategori `WebChatSwiftUI`).

## Bağlantı yapısı

- Veri düzlemi: Gateway WS yöntemleri `chat.history`, `chat.message.get`, `chat.send`, `chat.abort`, `chat.inject`; ayrıca `question.list` ve `question.resolve`; olaylar ise `chat`, `agent`, `presence`, `tick`, `health`; soru kartları `question.requested` ve `question.resolved` olaylarını izler ve yeniden bağlantılardan sonra `question.list` üzerinden yenilenir.
- `chat.history`, görüntüleme için normalleştirilmiş bir döküm döndürür: satır içi yönerge etiketleri görünür metinden çıkarılır; düz metin araç çağrısı XML yükleri (`<tool_call>`, `<function_call>`, `<tool_calls>`, `<function_calls>`, kesilmiş bloklar dâhil) ve sızdırılmış model denetim belirteçleri kaldırılır; tam olarak `NO_REPLY`/`no_reply` gibi yalnızca sessiz belirteç içeren assistant satırları atlanır ve aşırı büyük satırlar kesilmiş bir yer tutucuyla değiştirilebilir.
- Oturum: yukarıdaki gibi varsayılan olarak birincil oturumu kullanır; kullanıcı arayüzü oturumlar arasında geçiş yapabilir.
- Oturum grupları: `sessions.groups.list`, `sessions.groups.put`, `sessions.groups.rename` ve `sessions.groups.delete` grup kataloğunun sahibidir. Üyelik, `sessions.patch` üzerinden güncellenen oturum `category` değeridir.
- Okunmamış durumu: bir oturum etkinleştikten ve canlı geçmişi başarıyla yüklendikten sonra uygulama, o oturumun okunmamış işaretini temizler. Başarısız geçmiş yüklemeleri bu işareti temizlemez; geçici bir yama hatası bir sonraki etkinleştirmede yeniden denenir.
- İlk kullanım kurulumu, başlangıç ayarlarını ayrı tutmak için özel bir oturum kullanır.
- Çevrimdışı önbellek: uygulama, Gateway başına son sohbet oturumları ve dökümlerinden oluşan küçük, salt okunur bir önbellek tutar (`~/Library/Application Support/OpenClaw/chat-cache.sqlite`): soğuk açılışlar bilinen son dökümü hemen görüntüler ve Gateway yanıt verdiğinde yeniler; son sohbetler bağlantı kesikken de gözden geçirilebilir durumda kalır (bağlantı geri gelene kadar gönderim devre dışı kalır).

## Güvenlik yüzeyi

- Uzak mod, SSH üzerinden yalnızca Gateway WebSocket denetim bağlantı noktasını iletir.

## Bilinen sınırlamalar

- Kullanıcı arayüzü, tam bir tarayıcı korumalı alanı için değil, sohbet oturumları için optimize edilmiştir.

## İlgili

- [WebChat](/tr/web/webchat)
- [macOS uygulaması](/tr/platforms/macos)
