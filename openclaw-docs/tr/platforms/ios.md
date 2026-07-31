---
read_when:
    - iOS Node'unu eşleştirme veya yeniden bağlama
    - Doğrudan Apple Watch Node'unu etkinleştirme veya sorunlarını giderme
    - iOS uygulamasını kaynak koddan çalıştırma
    - Gateway keşfinde veya canvas komutlarında hata ayıklama
summary: 'iOS Node uygulaması: Gateway''e bağlanma, eşleştirme, tuval ve sorun giderme'
title: iOS uygulaması
x-i18n:
    generated_at: "2026-07-26T22:51:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2b01a63fa1e2c445f7fb35843536f7f5918e94bfe885dac19c852d7d52d86342
    source_path: platforms/ios.md
    workflow: 16
---

Kullanılabilirlik: iPhone uygulaması derlemeleri, bir sürüm için etkinleştirildiğinde Apple kanalları üzerinden dağıtılır. Yerel geliştirme derlemeleri kaynak koddan da çalıştırılabilir.

## İşlevi

- WebSocket üzerinden (LAN veya tailnet) bir Gateway'e bağlanır.
- Node yeteneklerini kullanıma sunar: Canvas, ekran anlık görüntüsü, kamera çekimi, konum, konuşma modu, sesle uyandırma ve isteğe bağlı Sağlık özetleri.
- `node.invoke` komutlarını alır ve node durum olaylarını bildirir.
- Agents yüzeyinden (Files) seçilen aracının çalışma alanına salt okunur olarak göz atar: dizinlerde ayrıntılı gezinme, sözdizimi vurgulu metin önizlemeleri, görüntü önizlemeleri ve paylaşım sayfası üzerinden dışa aktarma. Yazma işlemi yoktur; önizlemelerin boyutu gateway tarafından sınırlandırılır.
- Eşleştirilmiş her gateway için son sohbet oturumlarının ve transkriptlerin küçük, salt okunur bir çevrimdışı önbelleğini tutar: soğuk açılışlarda bilinen son transkript hemen görüntülenir ve gateway yanıt verdiğinde yenilenir, bağlantı kesikken son sohbetlere göz atılabilir ve sıfırlama/unutma işlemi korumalı yerel önbelleği temizler.
- Bağlantı kesikken gönderilen metin mesajlarını gateway başına kalıcı bir giden kutusunda sıraya alır (en fazla 50): sıraya alınan balonlar transkriptte gösterilir, yeniden bağlanıldığında idempotent yeniden denemelerle sırayla gönderilir, standart geçmiş gönderimi doğrulayana kadar kalıcı tutulur, yeniden deneme/silme eylemi gösterilmeden önce geri çekilmeli olarak yeniden denenir ve çevrimdışı geçen 48 saatten sonra gönderilmek yerine zaman aşımına uğrar; sıfırlama/unutma işlemi önbellekle birlikte kuyruğu da temizler.
- Chat, tek metin ve ses yüzeyidir. Chat eylemleri Chat'ten ayrılmadan tam Sessions ekranını açabilir ve asistan akıl yürütmesini ve araç etkinliğini gösterebilir veya gizleyebilir. Taslak dikte için mikrofona dokunun, sesli not kaydetmek için menüsünü açın veya gerçek zamanlı ses için satır içi Talk denetimini kullanın; Talk denetimi dinleme veya konuşma sırasında canlı mikrofon ya da oynatma düzeyine göre hareketlenir.
- **Settings -> OpenClaw**, operatör bağlantısında `operator.admin` bulunduğunda ve Gateway `openclaw.chat` desteğine sahip olduğunda özel bir Gateway ayarları asistanı açar. Kurulum görüşmesi sıradan Chat'ten ayrı tutulur, gizli yanıtlar yerel olarak sansürlenir ve yalnızca **Open Chat** öğesine dokunduktan sonra Chat'e geçilir.
- Asistan mesajlarını isteğe bağlı olarak seslendirir: Chat'te bir mesaja uzun basın ve **Listen** öğesini seçin. Uygulama, yapılandırılmış TTS sağlayıcısıyla desteklenen gateway `tts.speak` kliplerini oynatır ve gateway sesi kullanılamadığında veya oynatılamadığında cihaz üzerindeki konuşma sentezine geri döner. Oturum değiştirildiğinde veya uygulama arka plana alındığında oynatma durur.

## Gereksinimler

- Başka bir cihazda çalışan Gateway (macOS, Linux veya WSL2 üzerinden Windows).
- Ağ yolu:
  - Bonjour üzerinden aynı LAN, **veya**
  - Tek noktaya yayın DNS-SD üzerinden tailnet (örnek alan adı: `openclaw.internal.`), **veya**
  - Manuel ana makine/bağlantı noktası (geri dönüş seçeneği).

## Hızlı başlangıç (eşleştirme + bağlanma)

İlk açılışta uygulama, kısa bir eşleştirme açıklaması ve
izinler sayfası (bildirimler, kamera, mikrofon, fotoğraflar, kişiler,
takvim, anımsatıcılar, konum) boyunca yönlendirir. Her izin isteğe bağlıdır ve
daha sonra **Settings** -> **Permissions** bölümünden veya iOS Settings uygulamasından
değiştirilebilir.

1. Telefonunuzun erişebileceği bir rotaya sahip kimliği doğrulanmış bir Gateway başlatın. Tailscale
   Serve, önerilen uzak bağlantı yoludur:

```bash
openclaw gateway --port 18789 --tailscale serve
```

Güvenilir bir aynı LAN kurulumu için bunun yerine kimliği doğrulanmış bir `gateway.bind: "lan"`
kullanın. Varsayılan geri döngü bağlamasına telefondan erişilemez. Gateway
henüz yapılandırılmadıysa kurulum kodu oluşturma işleminin bir token veya parola
kimlik doğrulama yoluna sahip olması için önce `openclaw onboard` komutunu çalıştırın.

2. [Control UI](/tr/web/control-ui) arayüzünü açın, **Nodes** öğesini seçin ve
   **Devices** sayfasında **Pair mobile device** öğesine tıklayın. Tam erişim önerilir
   ve varsayılan olarak seçilidir; yalnızca yönetsel Gateway denetimlerini hariç
   bırakmak istediğinizde Limited access seçeneğini belirleyin, ardından **Create setup code** öğesine tıklayın.

3. iOS uygulamasında **Settings** -> **Gateway** bölümünü açın, QR kodunu tarayın (veya
   kurulum kodunu yapıştırın) ve bağlanın.

   Kurulum kodu hem LAN hem de Tailscale Serve rotalarını içeriyorsa uygulama
   bunları sırayla yoklar ve erişilebilen ilk uç noktayı kaydeder.

   Eşleştirilmiş gateway'ler **Gateways** listesinde kalır. Onay işareti
   odaklanılan gateway'i belirtir; başka bir satırdaki şimşek denetimini kullanarak
   operatör oturumunun aynı anda bağlı kalmasını sağlayın. Odağı değiştirmek,
   etkin diğer gateway'lerin bağlantısını kesmez. Yalnızca odaklanılan gateway,
   iPhone'un yetenek taşıyan node oturumunu alır; böylece kamera, ekran, konum ve
   diğer cihaz komutlarının her zaman belirsizliğe yer vermeyen tek bir sahibi olur. Uygulama
   arka plana geçtiğinde iOS bu ön plan bağlantılarını askıya alabilir.

4. Resmî uygulama otomatik olarak bağlanır. **Pending approval** bir
   istek gösterirse onaylamadan önce rolünü ve kapsamlarını inceleyin.

   **Settings → Gateway**, kaydedilen operatör bağlantısının
   **Full** veya **Limited** erişime sahip olup olmadığını gösterir. Düz metin LAN `ws://` kurulumu,
   bearer token güvenliği için otomatik olarak sınırlandırılır. Sınırlandırılmışsa `wss://`
   veya Tailscale Serve yapılandırın, Control UI ya da `openclaw qr` üzerinden yeni bir tam erişim
   kodunu tarayın ve ardından ayarları ve yükseltmeleri etkinleştirmek için yeniden bağlanın.

Control UI düğmesi, `operator.admin` içeren önceden eşleştirilmiş bir oturum gerektirir.
Terminal geri dönüş seçeneği olarak iOS uygulamasında keşfedilmiş bir gateway seçin (veya
Manual Host seçeneğini etkinleştirip ana makine/bağlantı noktasını girin), ardından isteği Gateway ana makinesinde onaylayın:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Uygulama değişen kimlik doğrulama ayrıntılarıyla (rol/kapsamlar/açık anahtar) eşleştirmeyi yeniden denerse önceki bekleyen isteğin yerini yeni bir `requestId` alır. Onaylamadan önce `openclaw devices list` komutunu yeniden çalıştırın.

İsteğe bağlı: iOS node'u her zaman sıkı denetlenen bir alt ağdan bağlanıyorsa açık CIDR'ler veya tam IP'lerle ilk node eşleştirmesinin otomatik onaylanmasını etkinleştirebilirsiniz:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

Bu özellik varsayılan olarak devre dışıdır. Yalnızca kapsam istenmeyen yeni `role: node` eşleştirmelerine uygulanır. Operatör/tarayıcı eşleştirmesi ile rol, kapsam, meta veri veya açık anahtar değişiklikleri yine de manuel onay gerektirir.

5. Bağlantıyı doğrulayın:

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

## Sağlık özetleri

iOS node'u, geçerli takvim günü için isteğe bağlı, salt okunur bir HealthKit
toplamı döndürebilir. iOS cihaz izni ile açık Gateway komut yetkilendirmesi
birbirinden bağımsız geçitlerdir. Kurulum, çağırma, yük alanları, gizlilik davranışı
ve sorun giderme için [HealthKit özetleri](/tr/platforms/ios-healthkit) bölümüne bakın.

Varsayılan olarak Apple Watch eşlikçi uygulaması mevcut iPhone aktarımını kullanmaya
devam eder ve ayrı bir Gateway eşleştirmesi gerektirmez. Apple'ın Watch uygulamasında Watch'u
iPhone ile eşleştirin, OpenClaw'ı **Watch app -> My Watch -> Available
Apps** üzerinden yükleyin, ardından OpenClaw'ı her iki cihazda da bir kez açın.

## Komut onaylarını inceleme

`operator.admin` içeren bir operatör bağlantısı veya Gateway tarafından açıkça
hedeflenen eşleştirilmiş bir `operator.approvals` bağlantısı, iPhone'daki bekleyen
exec isteklerini inceleyebilir. Onay kartı Gateway'in temizlenmiş komut
önizlemesini, uyarısını, ana makine bağlamını, sona erme süresini ve yalnızca
o isteğin sunduğu kararları gösterir. Eşleştirilmiş Apple Watch, aynı
incelemeci için güvenli istemi mevcut iPhone aktarımı üzerinden alır ve kompakt
bir kereliğine izin ver/reddet karar alt kümesini sunar. Doğrudan Watch Gateway modu
onay istemlerini taşımaz.

Onay durumu Control UI ve desteklenen sohbet yüzeyleriyle paylaşılır. İlk
kaydedilen yanıt geçerli olur. Başka bir yüzey isteği çözdükten, uzaktan
çözüldü bildirimi geldikten ve çözüm onayının kaybolmuş olabileceği her durumda
iPhone ve Watch, Gateway'in standart terminal kaydını getirir.
Bu geri okuma isteğin beklemede kalıp kalmadığını doğrulayana kadar eylemler kullanılamaz.

Onay sahipliği seçilen Gateway'e bağlıdır. Gateway'ler arasında geçiş yapmak
eski bir istemi yeni bağlantıya uygulayamaz. Birleşik onay yöntemlerinden
önceki Gateway'ler, dağıtılmış exec'e özgü yöntemlere geri döner;
korunan terminal durumu ve daha zengin yüzeyler arası sonuçlar için güncellenmiş bir
Gateway gerekir.

## Aracı sorularını yanıtlama

Chat, `operator.questions` (veya `operator.admin`) içeren operatör bağlantılarında
bekleyen Gateway sorularını yerel kartlar olarak gösterir. Kartlar tekli ve
çoklu seçim seçeneklerini, seçenek açıklamalarını, serbest metinli **Other** yanıtlarını ve
sona erme geri sayımını destekler. Yeniden bağlantılar, bekleyen soruları Gateway'den yeniden yükler. Bir kart,
bu cihaz yanıtladığında, başka bir yüzey önce yanıtladığında veya
sorunun süresi dolduğunda ya da soru iptal edildiğinde kilitlenir.

## İsteğe bağlı doğrudan Apple Watch node'u

Doğrudan mod, saate kendi imzalı node kimliğini ve Gateway bağlantısını verir.
OpenClaw etkin olduğu sürece, eşleştirilmiş iPhone kullanılamadığında bile
desteklenen node komutları saatin Wi-Fi veya hücresel bağlantısı üzerinden çalışmaya devam eder.

Gereksinimler:

- iPhone, `operator.admin` kapsamıyla Gateway'e bağlıdır.
- Kurulum kodu, watchOS tarafından güvenilen bir sertifikaya sahip `wss://` Gateway uç noktasını
  duyurur; saat, karşılık gelen `https://` kaynağını yoklar. Düz HTTP ve
  kendinden imzalı ya da yalnızca parmak izine dayalı güven desteklenmez. Uç nokta yapılandırması için
  [Gateway sahipliğinde eşleştirme](/tr/gateway/pairing) bölümüne bakın. Geri döngü, yalnızca iPhone'a yönelik
  ve yalnızca tailnet rotalarına saat tarafından bağımsız olarak erişilemez.
- Hücresel kullanım, etkin hizmete sahip hücresel özellikli bir Apple Watch gerektirir.
- OpenClaw saatte etkindir. Apple, sıradan watchOS uygulamalarının
  genel WebSocket/TCP bağlantılarını sürdürmesine izin vermez; bu nedenle doğrudan node kısa HTTPS
  yoklamaları kullanır ve uygulama ön plana döndüğünde yeniden bağlanır. Apple'ın
  [watchOS düşük düzeyli ağ iletişimi kılavuzuna](https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchOS) bakın.

Kurulum:

1. iPhone'da **Settings -> Apple Watch** bölümünü açın.
2. __Enable Direct Gateway Connection** öğesine dokunun.
3. Kısa ömürlü kurulum kodunun süresi dolmadan saatte OpenClaw'ı açın.
4. Ayrı Apple Watch satırını `openclaw nodes status` ile doğrulayın.

Kurulum kodu kısa ömürlü, yalnızca node'a yönelik bir önyükleme kimlik bilgisi içerir; süresi dolana kadar
parola gibi değerlendirin. Kod hiçbir zaman iPhone'un kayıtlı Gateway
parolasını veya token'ını içermez. Eşleştirmeden sonra saat kendi cihaz token'ını saklar ve
önyükleme kimlik bilgisini siler. Doğrudan mod yalnızca aşağıdaki komutları kapsar.
Chat, Talk, onaylar ve mevcut `watch.*` bildirim akışı
iPhone aktarım özellikleri olarak kalır ve eşleştirilmiş iPhone'u gerektirmeye devam eder.

Doğrudan watchOS node komutları:

| Yüzey        | Komutlar                       | Notlar                                                   |
| ------------- | ------------------------------ | ------------------------------------------------------- |
| Cihaz         | `device.info`, `device.status` | Watch kimliği, pil, termal durum, depolama ve ağ. |
| Bildirimler | `system.notify`                | Uygulama etkinken; Watch izni gerektirir.     |

watchOS, WebKit'i üçüncü taraf uygulamaların kullanımına sunmaz; bu nedenle doğrudan Watch node'u
Canvas komutlarını duyurmaz.

## Resmî derlemeler için aktarım destekli anlık bildirim

Resmî olarak dağıtılan iOS derlemeleri, ham APNs token'ını gateway'de yayımlamak yerine harici bir anlık bildirim aktarımı kullanır. Herkese açık sürüm kanalındaki resmî App Store derlemeleri, `https://ios-push-relay.openclaw.ai` konumundaki barındırılan aktarımı kullanır; bu temel URL App Store dağıtımı için sabit kodlanmıştır ve hiçbir geçersiz kılma değerini okumaz.

Özel aktarım dağıtımları, aktarım URL'si gateway aktarım URL'siyle eşleşen bilinçli olarak ayrı bir iOS derleme/dağıtım yolu gerektirir. App Store sürüm kanalı hiçbir zaman özel aktarım URL'sini kabul etmez. Özel bir aktarım derlemesi kullanıyorsanız eşleşen gateway aktarım URL'sini ayarlayın:

```json5
{
  gateway: {
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
        },
      },
    },
  },
}
```

Akışın çalışma şekli:

- iOS uygulaması, App Attest ve bir StoreKit uygulama işlemi JWS'si kullanarak aktarıcıya kaydolur.
- Aktarıcı, opak bir aktarıcı tanıtıcısının yanı sıra kayıt kapsamlı bir gönderim yetkisi döndürür.
- iOS uygulaması, eşleştirilmiş Gateway kimliğini (`gateway.identity.get`) alır ve aktarıcı kaydına dahil eder; böylece aktarıcı destekli kayıt, söz konusu Gateway'e devredilir.
- Uygulama, aktarıcı destekli bu kaydı `push.apns.register` ile eşleştirilmiş Gateway'e iletir.
- Gateway, `push.test`, arka plan uyandırmaları ve uyandırma dürtmeleri için saklanan bu aktarıcı tanıtıcısını kullanır.
- Uygulama daha sonra farklı bir Gateway'e veya farklı bir aktarıcı temel URL'sine sahip bir derlemeye bağlanırsa eski bağlamayı yeniden kullanmak yerine aktarıcı kaydını yeniler.

Gateway'in bu yol için **ihtiyaç duymadığı** öğeler: dağıtım genelinde bir aktarıcı belirteci ve resmî App Store aktarıcı destekli gönderimleri için doğrudan bir APNs anahtarı gerekmez.

Beklenen operatör akışı:

1. Resmî iOS uygulamasını yükleyin.
2. İsteğe bağlı: yalnızca kasıtlı olarak ayrı bir özel aktarıcı derlemesi kullanırken Gateway üzerinde `gateway.push.apns.relay.baseUrl` ayarını yapın.
3. Uygulamayı Gateway ile eşleştirin ve bağlantının tamamlanmasını bekleyin.
4. Uygulama; bir APNs belirteci edindikten, operatör oturumu bağlandıktan ve aktarıcı kaydı başarıyla tamamlandıktan sonra `push.apns.register` yayımlar.
5. Bundan sonra `push.test`, yeniden bağlanma uyandırmaları ve uyandırma dürtmeleri, saklanan aktarıcı destekli kaydı kullanabilir.

## Arka planda etkinlik sinyalleri

iOS uygulamayı sessiz anlık bildirim, arka plan yenilemesi veya önemli konum olayı için uyandırdığında uygulama kısa bir Node yeniden bağlantısı kurmayı dener ve ardından `event: "node.presence.alive"` ile `node.event` çağrısını yapar. Gateway, bunu yalnızca kimliği doğrulanmış Node cihaz kimliği bilindikten sonra eşleştirilmiş Node/cihaz meta verilerine `lastSeenAtMs`/`lastSeenReason` olarak kaydeder.

Uygulama, arka plan uyandırmasının başarıyla kaydedildiğini yalnızca Gateway yanıtı `handled: true` içerdiğinde kabul eder. Eski Gateway sürümleri, `node.event` çağrısını `{ "ok": true }` ile onaylayabilir; bu yanıt uyumludur ancak kalıcı bir son görülme güncellemesi olarak sayılmaz.

Uyumluluk notu:

- `OPENCLAW_APNS_RELAY_BASE_URL`, Gateway için geçici bir ortam geçersiz kılması olarak hâlâ çalışır (`gateway.push.apns.relay.baseUrl`, öncelikle yapılandırmayı kullanan yoldur).
- App Store sürüm derlemesinin anlık bildirim modu, barındırılan aktarıcı ana makinesini sabit kodlar ve hiçbir zaman aktarıcı URL'si geçersiz kılmasını okumaz. Derleme zamanı ortam değişkeni `OPENCLAW_PUSH_RELAY_BASE_URL` yalnızca yerel/korumalı alan iOS derleme modlarını etkiler.

## Kimlik doğrulama ve güven akışı

Aktarıcı, doğrudan Gateway üzerinden APNs kullanımının resmî iOS derlemeleri için sağlayamayacağı iki kısıtlamayı uygulamak üzere vardır:

- Barındırılan aktarıcıyı yalnızca Apple aracılığıyla dağıtılan gerçek OpenClaw iOS derlemeleri kullanabilir.
- Bir Gateway, yalnızca söz konusu Gateway ile eşleştirilmiş iOS cihazlarına aktarıcı destekli anlık bildirimler gönderebilir.

Adım adım:

1. `iOS app -> gateway`: uygulama, normal Gateway kimlik doğrulama akışı üzerinden Gateway ile eşleşir ve kimliği doğrulanmış bir Node oturumunun yanı sıra kimliği doğrulanmış bir operatör oturumu edinir. Operatör oturumu `gateway.identity.get` çağrısını yapar.
2. `iOS app -> relay`: uygulama, App Attest kanıtı ve bir StoreKit uygulama işlemi JWS'si ile HTTPS üzerinden aktarıcı kayıt uç noktalarını çağırır. Aktarıcı; paket kimliğini, App Attest kanıtını ve Apple dağıtım kanıtını doğrular ve resmî/üretim dağıtım yolunu zorunlu tutar. Yerel bir derleme resmî Apple dağıtım kanıtını sağlayamayacağından, yerel Xcode/geliştirme derlemelerinin barındırılan aktarıcıyı kullanmasını engelleyen budur.
3. `gateway identity delegation`: uygulama, aktarıcı kaydından önce eşleştirilmiş Gateway kimliğini `gateway.identity.get` üzerinden alır ve aktarıcı kayıt yüküne dahil eder. Aktarıcı, söz konusu Gateway kimliğine devredilmiş bir aktarıcı tanıtıcısı ve kayıt kapsamlı bir gönderim yetkisi döndürür.
4. `gateway -> relay`: Gateway, `push.apns.register` üzerinden gelen aktarıcı tanıtıcısını ve gönderim yetkisini saklar. `push.test`, yeniden bağlanma uyandırmaları ve uyandırma dürtmelerinde Gateway, gönderim isteğini kendi cihaz kimliğiyle imzalar; aktarıcı hem saklanan gönderim yetkisini hem de Gateway imzasını, kayıt sırasında devredilen Gateway kimliğine göre doğrular. Başka bir Gateway, tanıtıcıyı bir şekilde ele geçirse bile saklanan bu kaydı yeniden kullanamaz.
5. `relay -> APNs`: aktarıcı, üretim APNs kimlik bilgilerini ve resmî derlemenin ham APNs belirtecini yönetir. Gateway, aktarıcı destekli resmî derlemeler için ham APNs belirtecini hiçbir zaman saklamaz; aktarıcı, eşleştirilmiş Gateway adına son anlık bildirimi APNs'ye gönderir.

Bu tasarımın oluşturulma nedeni: üretim APNs kimlik bilgilerini kullanıcı Gateway'lerinden uzak tutmak, resmî derlemelerin ham APNs belirteçlerini Gateway üzerinde saklamaktan kaçınmak, barındırılan aktarıcının yalnızca resmî OpenClaw iOS derlemeleri tarafından kullanılmasına izin vermek ve bir Gateway'in farklı bir Gateway'e ait iOS cihazlarına uyandırma bildirimleri göndermesini önlemek.

Yerel/manuel derlemeler doğrudan APNs kullanmaya devam eder. Bu derlemeleri aktarıcı olmadan test ediyorsanız Gateway hâlâ doğrudan APNs kimlik bilgilerine ihtiyaç duyar:

```bash
export OPENCLAW_APNS_TEAM_ID="TEAMID"
export OPENCLAW_APNS_KEY_ID="KEYID"
export OPENCLAW_APNS_PRIVATE_KEY_P8="$(cat /path/to/AuthKey_KEYID.p8)"
```

Bunlar Fastlane ayarları değil, Gateway ana makinesi çalışma zamanı ortam değişkenleridir. `apps/ios/fastlane/.env` yalnızca `APP_STORE_CONNECT_KEY_ID` ve `APP_STORE_CONNECT_ISSUER_ID` gibi App Store Connect kimlik doğrulama bilgilerini saklar; yerel iOS derlemeleri için doğrudan APNs teslimatını yapılandırmaz.

`~/.openclaw/credentials/` altındaki diğer sağlayıcı kimlik bilgileriyle tutarlı, önerilen Gateway ana makinesi depolama düzeni:

```bash
mkdir -p ~/.openclaw/credentials/apns
chmod 700 ~/.openclaw/credentials/apns
mv /path/to/AuthKey_KEYID.p8 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
chmod 600 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
export OPENCLAW_APNS_PRIVATE_KEY_PATH="$HOME/.openclaw/credentials/apns/AuthKey_KEYID.p8"
```

`.p8` dosyasını işlemeye dahil etmeyin veya depo çalışma kopyasının altına yerleştirmeyin.

## Keşif yolları

### Bonjour (LAN)

iOS uygulaması, `local.` üzerinde `_openclaw-gw._tcp` hizmetini ve yapılandırıldığında aynı geniş alan DNS-SD keşif alanını tarar. Aynı LAN'daki Gateway'ler `local.` üzerinden otomatik olarak görünür; ağlar arası keşif, sinyal türünü değiştirmeden yapılandırılmış geniş alan etki alanını kullanabilir.

### Tailnet (ağlar arası)

mDNS engellenmişse tek noktaya yayın DNS-SD bölgesi (bir etki alanı seçin; örnek: `openclaw.internal.`) ve Tailscale bölünmüş DNS kullanın. CoreDNS örneği için [Bonjour](/tr/gateway/bonjour) sayfasına bakın.

### Manuel ana makine/bağlantı noktası

Settings bölümünde **Manual Host** seçeneğini etkinleştirin ve Gateway ana makinesi ile bağlantı noktasını girin (varsayılan: `18789`).

## Birden fazla Gateway

Uygulama, eşleştirildiği her Gateway'in kaydını tutar; böylece yeniden eşleştirme yapmadan aralarında geçiş yapabilirsiniz:

- **Settings -> Gateway**, etkin Gateway'in işaretlendiği bir **Paired Gateways** listesi gösterir. Geçiş yapmak için bir girdiye dokunun; uygulama mevcut oturumları sonlandırır ve seçilen Gateway'e yeniden bağlanır. Birden fazla Gateway eşleştirildiğinde bağlantı satırının yanında bir hızlı geçiş menüsü görünür.
- Kimlik bilgileri, TLS güven kararları, Gateway'e özgü tercihler ve önbelleğe alınmış sohbet geçmişi her Gateway için ayrı saklanır. Geçiş yapıldığında Gateway'ler arasındaki durum hiçbir zaman karışmaz ve anlık bildirim kaydı etkin Gateway'i izler.
- Eşleştirilmiş bir Gateway'i kaydırarak (veya bağlam menüsünü kullanarak) **Forget** seçeneğini uygulayın; bu işlem kimlik bilgilerini, cihaz belirteçlerini, TLS sabitlemesini ve önbelleğe alınmış sohbetleri kaldırır.
- Keşfedilen Gateway'lere geçiş yapılabilmesi için bunların ağda görünür olması gerekir; manuel Gateway'ler kaydedilmiş ana makine ve bağlantı noktası üzerinden yeniden bağlanır.

## Canvas + A2UI

iOS Node, bir WKWebView Canvas'ı işler. Bunu yönetmek için `node.invoke` kullanın:

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.navigate --params '{"url":"http://<gateway-host>:18789/__openclaw__/canvas/"}'
```

Notlar:

- Gateway Canvas ana makinesi, Gateway HTTP sunucusundan (`gateway.port` ile aynı bağlantı noktası; varsayılan: `18789`) `/__openclaw__/canvas/` ve `/__openclaw__/a2ui/` sunar.
- iOS Node, yerleşik iskeleti bağlı durumdaki varsayılan görünüm olarak korur. `canvas.a2ui.push` ve `canvas.a2ui.reset`, paketlenmiş ve uygulamaya ait A2UI sayfasını kullanır.
- Uzak Gateway A2UI sayfaları iOS'ta yalnızca işleme amaçlıdır; yerel A2UI düğme eylemleri yalnızca paketlenmiş ve uygulamaya ait sayfalardan kabul edilir.
- `canvas.navigate` ve `{"url":""}` ile yerleşik iskelete dönün.

## Computer Use ilişkisi

iOS uygulaması bir mobil Node yüzeyidir, Codex Computer Use arka ucu değildir. Codex Computer Use ve `cua-driver mcp`, MCP araçları üzerinden yerel bir macOS masaüstünü kontrol eder; iOS uygulaması ise `canvas.*`, `camera.*`, `screen.*`, `location.*` ve `talk.*` gibi OpenClaw Node komutları üzerinden iPhone özelliklerini kullanıma sunar.

Aracılar, Node komutlarını çağırarak OpenClaw üzerinden iOS uygulamasını yine de çalıştırabilir; ancak bu çağrılar Gateway Node protokolü üzerinden geçer ve iOS ön plan/arka plan sınırlarına tabidir. Yerel masaüstü kontrolü için [Codex Computer Use](/tr/plugins/codex-computer-use), iOS Node özellikleri için bu sayfayı kullanın.

### Canvas değerlendirmesi / anlık görüntüsü

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.eval --params '{"javaScript":"(() => { const {ctx} = window.__openclaw; ctx.clearRect(0,0,innerWidth,innerHeight); ctx.lineWidth=6; ctx.strokeStyle=\"#ff2d55\"; ctx.beginPath(); ctx.moveTo(40,40); ctx.lineTo(innerWidth-40, innerHeight-40); ctx.stroke(); return \"ok\"; })()"}'
```

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.snapshot --params '{"maxWidth":900,"format":"jpeg"}'
```

## Sesle uyandırma + konuşma modu

- Sesle uyandırma ve konuşma modu Settings bölümünde kullanılabilir.
- `talk.realtime.transport`, `webrtc` olduğunda OpenAI gerçek zamanlı Talk, istemcinin yönettiği WebRTC'yi kullanır; açık bir `gateway-relay` yapılandırması Gateway tarafından yönetilmeye devam eder. Bkz. [Talk modu](/tr/nodes/talk).
- Talk özellikli iOS Node'ları `talk` yeteneğini duyurur ve `talk.ptt.start`, `talk.ptt.stop`, `talk.ptt.cancel` ile `talk.ptt.once` bildirebilir; Gateway, güvenilir Talk özellikli Node'lar için bu bas-konuş komutlarına varsayılan olarak izin verir.
- iOS arka plan sesini askıya alabilir; uygulama etkin değilken ses özelliklerini en iyi çaba esasına göre çalışan özellikler olarak değerlendirin.

## Yaygın hatalar

- `NODE_BACKGROUND_UNAVAILABLE`: iOS uygulamasını ön plana getirin (Canvas/kamera/ekran komutları bunu gerektirir).
- `A2UI_HOST_UNAVAILABLE`: paketlenmiş A2UI sayfasına uygulamanın WebView'ından erişilemedi; uygulamayı Screen sekmesinde ön planda tutun ve yeniden deneyin.
- Eşleştirme istemi hiç görünmüyor: `openclaw devices list` komutunu çalıştırın ve manuel olarak onaylayın.
- Watch, iPhone durumunu göstermiyor: iPhone'un `watch.status` içinde `watchPaired: true`
  ve `watchAppInstalled: true` bildirdiğini doğrulayın. Eşleştirme yanlışsa Watch'u
  Apple'ın Watch uygulamasında eşleştirin. Yükleme yanlışsa yardımcı uygulamayı
  **My Watch -> Available Apps** üzerinden yükleyin. Her iki değişiklikten sonra da
  OpenClaw'ı Watch üzerinde bir kez açın; anında erişilebilirlik için her iki uygulamanın
  da çalışıyor olması gerekirken sıraya alınmış güncellemeler daha sonra arka planda ulaşabilir.
- Yeniden yüklemenin ardından yeniden bağlantı başarısız oluyor: Keychain eşleştirme belirteci temizlenmiştir; Node'u yeniden eşleştirin.

## İlgili belgeler

- [Eşleştirme](/tr/channels/pairing)
- [Keşif](/tr/gateway/discovery)
- [Bonjour](/tr/gateway/bonjour)
