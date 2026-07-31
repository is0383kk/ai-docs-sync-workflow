---
read_when:
    - WebChat erişiminde hata ayıklama veya erişimi yapılandırma
summary: Sohbet kullanıcı arayüzü için loopback WebChat statik ana bilgisayarı ve Gateway WS kullanımı
title: WebChat
x-i18n:
    generated_at: "2026-07-27T00:10:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 19c301af1eb1b28650849cdd90924805dd0f5189516693505d9b75f62197007f
    source_path: web/webchat.md
    workflow: 16
---

Durum: macOS/iOS SwiftUI sohbet kullanıcı arayüzü doğrudan Gateway WebSocket ile iletişim kurar. Gömülü tarayıcı ve yerel statik sunucu yoktur.

## Nedir

- Gateway için yerel bir sohbet kullanıcı arayüzü.
- Diğer kanallarla aynı oturumları ve yönlendirme kurallarını kullanır.
- Belirlenimci yönlendirme: yanıtlar her zaman WebChat'e geri gönderilir.
- Geçmiş her zaman Gateway'den alınır (yerel dosya izleme yoktur). Gateway'e ulaşılamıyorsa WebChat salt okunurdur.

## Hızlı başlangıç

1. Gateway'i başlatın.
2. WebChat kullanıcı arayüzünü (macOS/iOS uygulaması) veya Control UI sohbet sekmesini açın.
3. Geçerli bir Gateway kimlik doğrulama yolunun yapılandırıldığından emin olun (geri döngüde bile varsayılan olarak paylaşılan gizli anahtar).

## Nasıl çalışır

- Kullanıcı arayüzü Gateway WebSocket'e bağlanır ve `chat.history`, `chat.send`, `chat.inject` ve `chat.message.get` RPC yöntemlerini kullanır.
- `chat.history` kararlılık için sınırlandırılmıştır: Gateway uzun metin alanlarını kısaltabilir, ağır meta verileri atlayabilir ve aşırı büyük girdileri `[chat.history omitted: message too large]` ile değiştirebilir. API istemcileri, tek bir çağrıda varsayılan sınırı geçersiz kılmak için istek başına `maxChars` gönderebilir.
- Görünür bir asistan mesajı `chat.history` içinde kısaltıldığında Control UI, varsayılan geçmiş yükünü artırmadan bir yan okuyucu açabilir ve `chat.message.get` üzerinden isteğe bağlı olarak görüntüleme için tamamen normalleştirilmiş girdiyi alabilir. `chat.message.get`, `chat.history` ile aynı transkript dalını ve görüntüleme kurallarını kullanır ancak `messageId` aracılığıyla tek bir girdiyi hedefler ve içeriğin tamamı artık döndürülemiyorsa gerçek durumu yansıtan bir kullanılamama nedeni döndürür.
- `chat.history`, yalnızca eklemeli oturum dosyalarında etkin transkript dalını izler; böylece terk edilmiş yeniden yazma dalları ve yerini daha yeni sürümlerin aldığı istem kopyaları WebChat'te işlenmez.
- Compaction girdileri, sıkıştırılmış transkriptin bir denetim noktası olarak korunduğunu açıklayan bir "Sıkıştırılmış geçmiş" ayırıcısı olarak işlenir ve oturum denetim noktalarını açmaya yönelik bir eylem sunar (izinler elverdiğinde dallandırma veya geri yükleme).
- Control UI, `chat.history` tarafından döndürülen temel Gateway `sessionId` değerini hatırlar ve bunu sonraki `chat.send` çağrılarına ekler; böylece kullanıcı bir oturum başlatmadığı veya sıfırlamadığı sürece yeniden bağlantılar ve sayfa yenilemeleri aynı kayıtlı konuşmayı sürdürür.
- Ön plan gönderimleri ayrıca işlenmiş geçmişteki görüntülenen dalın yaprağını `expectedLeafEntryId` olarak içerir; başka bir istemci önce dal değiştirdiyse Control UI, mesajı yeni dala göndermek yerine incelenmek üzere beklemeye alır ve transkripti yeniler. Yeniden bağlantı ve geri yüklenen giden kutusu tekrarları, mevcut geçmiş uzlaştırıldıktan sonra bu ön koşulu bilinçli olarak atlar.
- `chat.send` bir eş etkisizlik anahtarı alır (Control UI çalıştırma kimliğini kullanır); Gateway, aynı anahtarı yeniden kullanan tekrarlanan isteklerin yinelenmesini önler, böylece aynı oturum/mesaj/ekler için yeniden denenen veya aktarım sırasında yinelenen gönderimler ikinci bir çalıştırma oluşturmaz.
- Belirli bir mesajı yanıtlamak (sağ tıklama → Reply), hedefin transkript kimliğini `chat.send` üzerinde `replyToId` olarak gönderir. Gateway bu mesajı oturum geçmişinden çözümler ve Discord yanıtlarının kullandığı, kanaldan bağımsız aynı yanıt bağlamı meta verilerini doldurur: aracılar `has_reply_context` ile birlikte gönderici etiketi ve gövdeyi içeren güvenilmeyen "Geçerli kullanıcı mesajının yanıt hedefi" bloğunu görür. (WebChat istemlerinde `reply_to_id` gibi geçici konuşma kimlikleri, doğrudan WebChat oturumlarına yönelik mevcut bayt kararlı istem politikası uyarınca gizli tutulur.) Kalıcı bir transkript kimliği olmayan yanıt hedeflerinde (örneğin bekleyen gönderimler) mesaj gövdesindeki satır içi alıntıya geri dönülür.
- Çalışma alanı başlangıç dosyaları ve bekleyen `BOOTSTRAP.md` talimatları, WebChat kullanıcı mesajına kopyalanmak yerine aracı sistem isteminin `# Project Context` bölümü üzerinden sağlanır. Önyükleme içeriği kısaltılırsa sistem istemine bunun yerine kısa bir "Önyükleme Bağlamı Bildirimi" eklenir; ayrıntılı sayımlar ve yapılandırma ayarları tanılama yüzeylerinde kalır.
- `chat.history` üzerindeki görüntüleme normalleştirmesi şunları kaldırır: yalnızca çalışma zamanına yönelik OpenClaw bağlamı, gelen zarf sarmalayıcıları, `[[reply_to_current]]`, `[[reply_to:<id>]]` ve `[[audio_as_voice]]` gibi satır içi teslimat yönergesi etiketleri, düz metin araç çağrısı XML yükleri (`<tool_call>`, `<function_call>`, `<tool_calls>`, `<function_calls>`; kısaltılmış bloklar dâhil) ve sızmış ASCII/tam genişlikli model kontrol belirteçleri. Görünür metninin tamamı yalnızca sessiz `NO_REPLY` belirtecinden oluşan asistan girdileri (büyük/küçük harfe duyarsız olarak) atlanır.
- Akıl yürütme bayrağı taşıyan yanıt yükleri (`isReasoning: true`) WebChat asistan içeriğinden, transkript tekrar oynatma metninden ve ses içeriği bloklarından çıkarılır; böylece yalnızca düşünme içeren yükler görünür asistan mesajları veya oynatılabilir ses olarak gösterilmez.
- `chat.inject`, doğrudan transkripte bir asistan notu ekler ve bunu kullanıcı arayüzüne yayınlar (aracı çalıştırması yoktur).
- Durdurulan çalıştırmalarda kısmi asistan çıktısı kullanıcı arayüzünde görünür kalabilir. Arabelleğe alınmış çıktı varsa Gateway bu kısmi metni transkript geçmişinde kalıcı hâle getirir ve girdiyi durdurma meta verileriyle işaretler.

### Transkript ve teslimat modeli

WebChat'in iki ayrı veri yolu vardır:

- SQLite transkript satırları, kalıcı model/çalışma zamanı transkriptidir. Normal aracı çalıştırmalarında gömülü OpenClaw çalışma zamanı, model tarafından görülebilen `user`, `assistant` ve `toolResult` mesajlarını oturum erişimcisi üzerinden kalıcı hâle getirir. WebChat bu transkripte rastgele teslimat, durum veya yardımcı metin yazmaz.
- Gateway `ReplyPayload` olayları canlı teslimat izdüşümüdür: WebChat/kanal görüntülemesi, blok akışı, yönerge etiketleri, medya gömme, TTS/ses bayrakları ve kullanıcı arayüzü geri dönüş davranışı için normalleştirilir. Bunların kendisi kurallı oturum günlüğü değildir.
- `tools.message` üzerinden görünür yanıtlar gerektiren test düzenekleri, WebChat'i hâlâ mevcut çalıştırmaya ait dahili kaynak yanıt havuzu olarak kullanır. Bu etkin WebChat çalıştırmasından gelen hedefsiz bir `message.send`, aynı sohbete yansıtılır ve oturum transkriptine kopyalanır; WebChat yeniden kullanılabilir bir giden kanal hâline gelmez ve hiçbir zaman `lastChannel` değerini devralmaz.
- WebChat, yalnızca Gateway normal bir gömülü aracı dönüşünün dışında görüntülenen bir mesajın sahibi olduğunda asistan transkript girdileri ekler: `chat.inject`, aracı dışı komut yanıtları, durdurulmuş kısmi çıktı ve WebChat tarafından yönetilen medya transkript ekleri.
- Canlı asistan metni bir çalıştırma sırasında görünüyor ancak geçmiş yeniden yüklendikten sonra kayboluyorsa sırasıyla şunları denetleyin: SQLite transkriptinin asistan metnini içerip içermediği, `chat.history` görüntüleme izdüşümünün bunu kaldırıp kaldırmadığı ve ardından Control UI iyimser kuyruk birleştirmesinin yerel teslimat durumunu kalıcı anlık görüntüyle değiştirip değiştirmediği.

Normal aracı çalıştırması nihai yanıtları kalıcı olmalıdır; çünkü gömülü çalışma zamanı asistan `message_end` değerini yazar. Teslim edilmiş bir nihai yükü transkripte yansıtan herhangi bir geri dönüş, önce gömülü çalışma zamanının zaten yazdığı bir asistan dönüşünü çoğaltmaktan kaçınmalıdır.

## Control UI aracı araçları paneli

- Control UI `/agents` Araçlar panelinde, `tools.effective(sessionKey=...)` tarafından desteklenen bir "Available Right Now" görünümü bulunur: çekirdek, Plugin, kanalın sahip olduğu ve zaten keşfedilmiş MCP sunucusu araçları dâhil olmak üzere geçerli oturumun araç envanterinin sunucu tarafından türetilmiş, salt okunur izdüşümü.
- Ayrı bir yapılandırma düzenleme görünümü (`tools.catalog` tarafından desteklenir) profilleri, aracı başına geçersiz kılmaları ve katalog anlamlarını kapsar.
- Çalışma zamanı kullanılabilirliği oturum kapsamındadır. Aynı aracıdaki oturumlar arasında geçiş yapmak "Available Right Now" listesini değiştirebilir. Yapılandırılmış MCP sunucularına son keşiften bu yana bağlanılmadıysa veya bunlar değiştiyse panel, okuma yolundan MCP aktarımlarını sessizce başlatmak yerine bir bildirim gösterir.
- Yapılandırma düzenleyicisi çalışma zamanı kullanılabilirliği anlamına gelmez; etkin erişim yine politika önceliğini izler (`allow`/`deny`, aracı başına ve sağlayıcı/kanal geçersiz kılmaları).

## Uzaktan kullanım

- Uzak mod, Gateway WebSocket'i SSH/Tailscale üzerinden tüneller.
- Ayrı bir WebChat sunucusu çalıştırmanız gerekmez.

## Yapılandırma başvurusu (WebChat)

Tam yapılandırma: [Yapılandırma](/tr/gateway/configuration)

WebChat'in kalıcı bir yapılandırma bölümü yoktur. Gateway yerleşik `chat.history` görüntüleme sınırını kullanır; API istemcileri tek bir çağrıda bunu geçersiz kılmak için istek başına `maxChars` gönderebilir. Eski `channels.webchat` ve `gateway.webchat` yapılandırması kullanımdan kaldırılmıştır; bunu kaldırmak için `openclaw doctor --fix` komutunu çalıştırın.

İlgili genel seçenekler:

- `gateway.port`, `gateway.bind`: WebSocket ana makinesi/bağlantı noktası.
- `gateway.auth.mode`, `gateway.auth.token`, `gateway.auth.password`:
  paylaşılan gizli anahtarlı WebSocket kimlik doğrulaması.
- `gateway.auth.allowTailscale`: tarayıcıdaki Control UI sohbet sekmesi, etkinleştirildiğinde Tailscale
  Serve kimlik üst bilgilerini kullanabilir.
- `gateway.auth.mode: "trusted-proxy"`: kimliğe duyarlı bir **geri döngü dışı** proxy kaynağının arkasındaki tarayıcı istemcileri için ters proxy kimlik doğrulaması (bkz. [Güvenilir Proxy Kimlik Doğrulaması](/tr/gateway/trusted-proxy-auth)).
- `gateway.remote.url`, `gateway.remote.token`, `gateway.remote.password`: uzak Gateway hedefi.
- `session.*`: oturum depolama ve ana anahtar varsayılanları.

## İlgili

- [Control UI](/tr/web/control-ui)
- [Gösterge Paneli](/tr/web/dashboard)
