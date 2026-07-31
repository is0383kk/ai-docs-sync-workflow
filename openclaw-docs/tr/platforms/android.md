---
read_when:
    - Android Node'unu eşleştirme veya yeniden bağlama
    - Android Gateway keşfi veya kimlik doğrulamasında hata ayıklama
    - Uzak bir Mac'ten Android cihazını yansıtma veya kontrol etme
    - İstemciler arasında sohbet geçmişi tutarlılığını doğrulama
summary: 'Android uygulaması (Node): bağlantı çalışma kılavuzu + Bağlan/Sohbet/OpenClaw/Ses/Canvas komut yüzeyi'
title: Android uygulaması
x-i18n:
    generated_at: "2026-07-26T23:25:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a134a678e26924abc24dd107c3feaad9d09e83e3829eef73514c7ef078d578f1
    source_path: platforms/android.md
    workflow: 16
---

<Note>
Resmî Android uygulaması [Google Play](https://play.google.com/store/apps/details?id=ai.openclaw.app&hl=en_IN) üzerinden ve desteklenen [GitHub Sürümlerinde](https://github.com/openclaw/openclaw/releases) imzalı bağımsız APK olarak sunulur. Bir eşlikçi Node'dur ve çalışan bir OpenClaw Gateway gerektirir. Kaynak: [apps/android](https://github.com/openclaw/openclaw/tree/main/apps/android) ([derleme talimatları](https://github.com/openclaw/openclaw/blob/main/apps/android/README.md)).
</Note>

## Destek özeti

- Rol: eşlikçi Node uygulaması (Android, Gateway'i barındırmaz).
- Gateway gerekli: evet (macOS, Linux veya WSL2 aracılığıyla Windows üzerinde çalıştırın).
- Kurulum: [Google Play](https://play.google.com/store/apps/details?id=ai.openclaw.app&hl=en_IN) veya desteklenen bir [GitHub Sürümünden](https://github.com/openclaw/openclaw/releases) `OpenClaw-Android.apk`; Gateway için [Başlarken](/tr/start/getting-started), ardından [Eşleştirme](/tr/channels/pairing).
- Gateway: [Çalıştırma kılavuzu](/tr/gateway) + [Yapılandırma](/tr/gateway/configuration).
  - Protokoller: [Gateway protokolü](/tr/gateway/protocol) (Node'lar + kontrol düzlemi).
- **Settings → OpenClaw**, operatör bağlantısında `operator.admin` bulunduğunda ve Gateway `openclaw.chat` desteklediğinde özel bir Gateway ayarları yardımcısını açar. Kurulum görüşmesi sıradan Sohbetten ayrı kalır, gizli yanıtları yerel olarak karartır ve yalnızca **Open Chat** seçeneğine dokunulduktan sonra Sohbete geçer.

Sistem denetimi (launchd/systemd) Gateway ana makinesinde bulunur — bkz. [Gateway](/tr/gateway).

## Eş zamanlı Gateway oturumları

Her Gateway'i bir kez eşleştirin, ardından **Settings → Gateway** bölümünü açın. Onay işareti odaktaki Gateway'i belirtir ve her anahtar, odakta olmayan bir Gateway'in operatör oturumunun bağlı kalıp kalmayacağını denetler. Etkin Gateway'ler, uygulama ön plandayken bağımsız olarak yeniden bağlanır; bu nedenle odağı değiştirmek diğerlerinin bağlantısını kesmez. Yalnızca odaktaki Gateway, Android Node oturumuna ve cihaz yeteneklerine sahip olur; bu, eş zamanlı Gateway'lerin aynı telefona kamera, konum, ekran veya bildirim komutları vermesini engeller. Uygulama ön plandan ayrıldıktan sonra Android ikincil bağlantıları askıya alabilir.

## Wear OS eşlikçisi

Wear OS eşlikçisi, eşleştirilmiş Android telefonun kimliği doğrulanmış Gateway bağlantısını kullanır; saat hiçbir zaman Gateway kimlik bilgilerini almaz veya saklamaz. Aracıları ve oturumları seçebilir, sınırlandırılmış dökümleri okuyabilir, metin ya da dikte edilmiş yanıtlar gönderebilir, etkin bir çalıştırmayı iptal edebilir, seçilen oturumda gerçek zamanlı Konuşmayı başlatabilir ve eşleştirilmiş telefonun Gateway bağlantısını kurabilir veya kesebilir. Ayrıca yerel yanıt bildirimleri, koyu veya açık görünüm ve yanıtların isteğe bağlı olarak otomatik seslendirilmesini sunar. Aracı ve Gateway denetimleri, telefon ve saat güncellemelerinin farklı zamanlarda yapılabilmesi için yetenek uzlaşmasıyla belirlenir. Gerçek zamanlı Konuşma, mikrofon ve oynatma sesini geçici bir Wear OS Data Layer kanalı üzerinden aktarır ve seçili telefon, Gateway bağlantısı veya ses kanalı kaybedildiğinde durur.

## Google Play dışından kurulum

Olağan nihai ve düzeltme GitHub Sürümleri, evrensel bir `OpenClaw-Android.apk` ve `OpenClaw-Android-SHA256SUMS.txt` içerir. APK, sürüm etiketinden derlenir, OpenClaw Android sürüm anahtarıyla imzalanır ve GitHub Actions kaynak kanıtı taşır.

Her iki varlığı da listeleyen bir [sürüm](https://github.com/openclaw/openclaw/releases) seçin, ardından dışarıdan yüklemeden önce tam olarak bu etiketi indirip doğrulayın:

```bash
release_tag=vYYYY.M.PATCH
gh release download "$release_tag" \
  --repo openclaw/openclaw \
  --pattern OpenClaw-Android.apk \
  --pattern OpenClaw-Android-SHA256SUMS.txt
sha256sum --check OpenClaw-Android-SHA256SUMS.txt
gh attestation verify OpenClaw-Android.apk \
  --repo openclaw/openclaw \
  --signer-workflow openclaw/openclaw/.github/workflows/android-release.yml \
  --source-ref "refs/tags/${release_tag}" \
  --deny-self-hosted-runners
```

<Warning>
Google Play ve bağımsız APK kurulumları farklı güncelleme kanalları kullanır ve farklı imzalama kimliklerine sahip olabilir. Android, kanallar arasında geçiş yapmadan önce mevcut uygulamanın kaldırılmasını gerektirebilir; bu işlem uygulamanın yerel verilerini siler. Olağan güncellemeler için tek bir kanalda kalın.
</Warning>

## Android'i uzak bir Mac'ten yansıtma ve denetleme

[scrcpy](https://github.com/Genymobile/scrcpy), Android ekranını bir macOS penceresine yansıtır ve
klavye ile işaretçi girdisini Android Debug Bridge (ADB) üzerinden iletir. Bu, OpenClaw Node
bağlantısından ayrı, operatör taraflı bir iş akışıdır. Android cihaz ile Mac farklı konumlarda
bulunmasına rağmen özel bir Tailscale ağını paylaşıyorsa kullanışlıdır.

### Başlamadan önce

- Android cihaza ve Mac'e Tailscale yükleyin ve her ikisini de aynı tailnet'e bağlayın.
- Android'de **Developer options** ve **USB debugging** seçeneklerini etkinleştirin. Android 16, **Wireless
  debugging** seçeneğini **Settings > System > Developer options** altında bulundurur. Bkz. [Android geliştirici
  seçenekleri](https://developer.android.com/studio/debug/dev-options).
- Mac'e scrcpy ve ADB yükleyin:

  ```bash
  brew install scrcpy
  brew install --cask android-platform-tools
  ```

- İlk bağlantı için Android cihazı erişilebilir durumda tutun. Android, ilgili Mac'in cihazı denetleyebilmesi
  için her Mac'in ADB anahtarını onaylamalıdır.

### TCP üzerinden ADB'yi etkinleştirme

İlk kurulum için Android cihazı USB üzerinden güvenilir bir bilgisayara bağlayın ve hata ayıklama
istemini onaylayın. Ardından şunları çalıştırın:

```bash
adb devices
adb tcpip 5555
```

Artık USB bağlantısını kesebilirsiniz. Cihaz yeniden başlatıldıktan veya hata ayıklama sıfırlandıktan
sonra 5555 numaralı bağlantı noktası dinlemeyi durdurursa bu yerel kurulum adımını tekrarlayın.
Android 11 ve sonraki sürümler ilk güven ilişkisini **Wireless debugging > Pair device with pairing code**
ve `adb pair` ile de kurabilir.

### Yalnızca denetleyici Mac'e izin verme

Kısıtlayıcı izinlere sahip tailnet'ler, denetleyici Mac'in Android cihazdaki TCP 5555 numaralı
bağlantı noktasına erişmesine açıkça izin vermelidir. Örnek adresleri iki cihazın sabit Tailscale
IP'leriyle değiştirerek tailnet politikasına dar kapsamlı bir kural ekleyin:

```json5
{
  grants: [
    {
      src: ["<remote-mac-tailnet-ip>"],
      dst: ["<android-tailnet-ip>"],
      ip: ["tcp:5555"],
    },
  ],
}
```

Ana makine takma adları ve diğer seçiciler için [Tailscale izinleri](https://tailscale.com/docs/reference/syntax/grants)
bölümüne bakın. Bu bağlantı noktasını genel internete açmayın veya Funnel ile yayımlamayın:
yetkilendirilmiş bir ADB istemcisi cihaz üzerinde geniş denetime sahiptir.

### Bağlanma ve yansıtmayı başlatma

Uzak Mac'te:

```bash
adb connect <android-tailnet-ip>:5555
adb devices
scrcpy --serial <android-tailnet-ip>:5555
```

Bu Mac'ten yapılan ilk `adb connect`, Android'de bir yetkilendirme iletişim kutusu gösterir.
Cihazın kilidini açın, anahtar parmak izini onaylayın ve yalnızca Mac güvenilir olduğunda
**Always allow from this computer** seçeneğini belirleyin. Başarılı bir `adb devices` girdisi
`device` ile biter; `unauthorized`, cihazdaki istemin onaylanmadığı anlamına gelir.

scrcpy penceresi açıldıktan sonra pencereyi doğrudan kullanın veya
[Peekaboo](https://peekaboo.sh/) gibi bir macOS ekran otomasyonu aracıyla hedefleyin. scrcpy,
görüntüyü ve girdiyi taşır; Tailscale yalnızca özel ağ yolunu sağlar.

### Sorun giderme

- `Connection timed out`: TCP 5555 için tailnet iznini doğrulayın. Başarılı bir
  `tailscale ping`, eşler arası erişilebilirliği kanıtlar; politikanın bu TCP bağlantı noktasına
  izin verdiğini kanıtlamaz. Mac'ten `nc -vz <android-tailnet-ip> 5555` ile test edin.
- `unauthorized`: Android'in kilidini açıp uzak Mac'in ADB anahtarını onaylayın
  veya **Wireless debugging > Paired devices** altındaki eski iş istasyonunu kaldırıp yeniden eşleştirin.
- `Connection refused`: yerel olarak yeniden bağlanın ve `adb tcpip 5555` komutunu tekrar çalıştırın.
- Birden fazla cihaz listelendi: açık `--serial <android-tailnet-ip>:5555` bağımsız değişkenini kullanmaya devam edin.

İşiniz bittiğinde scrcpy'yi kapatın ve ADB bağlantısını kesin:

```bash
adb disconnect <android-tailnet-ip>:5555
```

## Bağlantı çalıştırma kılavuzu

Android Node uygulaması ⇄ (mDNS/NSD + WebSocket) ⇄ **Gateway**

Android doğrudan Gateway WebSocket'e bağlanır ve cihaz eşleştirmesini (`role: node`) kullanır.

Tailscale veya genel ana makineler için Android güvenli bir uç nokta gerektirir:

- Tercih edilen: `https://<magicdns>` / `wss://<magicdns>` ile Tailscale Serve / Funnel
- Ayrıca desteklenir: gerçek bir TLS uç noktasına sahip diğer tüm `wss://` Gateway URL'leri
- Şifrelenmemiş `ws://`; özel LAN adreslerinde / `.local` ana makinelerinde, ayrıca `localhost`, `127.0.0.1` ve Android emülatör köprüsünde (`10.0.2.2`) desteklenmeye devam eder; geri döngü dışı kurulum otomatik olarak sınırlı operatör erişimini kullanır

### Ön koşullar

- Gateway başka bir makinede çalışıyor (veya SSH aracılığıyla erişilebilir).
- Android cihazı/emülatörü Gateway WebSocket'e erişebiliyor:
  - mDNS/NSD ile aynı LAN'da, **veya**
  - Wide-Area Bonjour / tek noktaya yayın DNS-SD kullanılarak aynı Tailscale tailnet'inde (aşağıya bakın), **veya**
  - Elle girilen Gateway ana makinesi/bağlantı noktası (geri dönüş)
- Tailnet/genel mobil eşleştirme, ham tailnet IP `ws://` uç noktalarını **kullanmaz**. Bunun yerine Tailscale Serve veya başka bir `wss://` URL'si kullanın.
- Eşleştirme isteklerini onaylamak için Gateway makinesinde (veya SSH aracılığıyla) `openclaw` CLI bulunmalıdır.

### 1. Gateway'i başlatma

```bash
openclaw gateway --port 18789 --verbose
```

Günlüklerde şuna benzer bir satır gördüğünüzü doğrulayın:

- `listening on ws://0.0.0.0:18789`

Tailscale üzerinden uzak Android erişimi için ham tailnet bağlaması yerine Serve/Funnel'ı tercih edin:

```bash
openclaw gateway --tailscale serve
```

Bu, Android'e güvenli bir `wss://` / `https://` uç noktası sağlar. TLS'yi ayrıca sonlandırmadığınız sürece düz bir `gateway.bind: "tailnet"` kurulumu, ilk kez uzak Android eşleştirmesi için yeterli değildir.

### 2. Keşfi doğrulama (isteğe bağlı)

Gateway makinesinden:

```bash
dns-sd -B _openclaw-gw._tcp local.
```

Daha fazla hata ayıklama notu: [Bonjour](/tr/gateway/bonjour).

Geniş alan keşif etki alanını da yapılandırdıysanız şununla karşılaştırın:

```bash
openclaw gateway discover --json
```

Bu, yalnızca TXT ipuçları yerine çözümlenmiş hizmet uç noktasını kullanarak `local.`
ile yapılandırılmış geniş alan etki alanını tek geçişte gösterir.

#### Tek noktaya yayın DNS-SD ile ağlar arası keşif

Android NSD/mDNS keşfi ağlar arasında çalışmaz. Android Node'u ve Gateway farklı ağlarda olup
Tailscale üzerinden bağlıysa bunun yerine Wide-Area Bonjour / tek noktaya yayın DNS-SD kullanın.
Tailnet/genel Android eşleştirmesi için yalnızca keşif yeterli değildir — keşfedilen rota yine de
güvenli bir uç nokta (`wss://` veya Tailscale Serve) gerektirir:

1. Gateway ana makinesinde bir DNS-SD bölgesi (örnek: `openclaw.internal.`) kurun ve `_openclaw-gw._tcp` kayıtlarını yayımlayın.
2. Seçtiğiniz etki alanı için bu DNS sunucusunu işaret eden Tailscale bölünmüş DNS'i yapılandırın.

Ayrıntılar ve örnek CoreDNS yapılandırması: [Bonjour](/tr/gateway/bonjour).

### 3. Android'den bağlanma

Android uygulamasında:

- Uygulama, Gateway bağlantısını bir **ön plan hizmeti** (kalıcı bildirim) aracılığıyla etkin tutar.
- **Connect** sekmesini açın.
- **Setup Code** veya **Manual** modunu kullanın.
- Keşif engelleniyorsa **Advanced controls** bölümünde ana makineyi/bağlantı noktasını elle girin. Özel LAN ana makinelerinde `ws://` çalışmaya devam eder. Tailscale/genel ana makinelerde TLS'yi açın ve bir `wss://` / Tailscale Serve uç noktası kullanın.

İlk başarılı eşleştirmenin ardından Android, başlatıldığında etkin eşleştirilmiş Gateway'e otomatik olarak yeniden bağlanır (ağda görünür olması gereken keşfedilmiş Gateway'ler için mümkün olduğunca).

Resmî kurulum kodları Android'i bir Node olarak bağlar ve varsayılan olarak `wss://` üzerinden tam Gateway operatör erişimi verir. Döngüsel olmayan düz metin `ws://` kurulumu, bearer token güvenliği için otomatik olarak sınırlı erişim kullanır. **Ayarlar → Gateway**, **Tam** veya **Sınırlı** erişimi gösterir. Sınırlı bir bağlantı için
`wss://` veya Tailscale Serve'ü yapılandırın, Control UI'da ya da
`openclaw qr` ile yeni bir tam erişim kodu oluşturun, ardından bu sayfada kodu tarayın veya yapıştırın ve yeniden bağlanın. Kısıtlı profili kullanmak isteyen operatörler Control UI'da **Sınırlı erişim** seçeneğini belirleyebilir veya
`openclaw qr --limited` komutunu çalıştırabilir.

### Eşleştirilmiş Gateway'leri yönetme

Uygulama, eşleştirildiği her Gateway'in kaydını tutar; böylece operatör oturumlarını bağlı tutabilir ve yeniden eşleştirme yapmadan odağı değiştirebilirsiniz:

- **Ayarlar → Gateway**, odaklanmış olan işaretlenmiş şekilde eşleştirilmiş Gateway'leri listeler. Odaklamak için bir girdiye dokunun; etkin olan diğer operatör oturumları bağlı kalır.
- Her anahtar, uygulama ön plandayken odaklanılmayan ilgili Gateway'in bağlı kalıp kalmayacağını denetler. Odaklanmış Gateway etkin kalır ve telefonun Node bağlantısıyla cihaz yeteneklerinin sahibi olur.
- Birden fazla Gateway eşleştirildiğinde **Bağlan** sekmesi hızlı bir değiştirici gösterir.
- Kimlik bilgileri, cihaz token'ları, TLS güveni, sohbet geçmişi ve kuyruğa alınmış çevrimdışı mesajlar Gateway başına saklanır. Odağı değiştirmek Gateway'ler arasındaki durumları asla karıştırmaz ve çevrimdışıyken kuyruğa alınan mesajlar yalnızca yazıldıkları Gateway'e teslim edilir.
- **Unut**, bir Gateway'in kayıt girdisini kimlik bilgileri, cihaz token'ları, TLS sabitlemesi ve önbelleğe alınmış sohbetleriyle birlikte kaldırır.

### Canlılık bildirim sinyalleri

Kimliği doğrulanmış Node oturumu bağlandıktan sonra ve ön plan hizmeti hâlâ bağlıyken uygulama arka plana geçtiğinde Android, `event: "node.presence.alive"` ile `node.event` çağrısını yapar. Gateway, bunu yalnızca kimliği doğrulanmış Node cihaz kimliği bilindikten sonra eşleştirilmiş Node/cihaz meta verilerinde `lastSeenAtMs`/`lastSeenReason` olarak kaydeder.

Uygulama, bildirim sinyalini yalnızca Gateway yanıtı `handled: true` içerdiğinde başarıyla kaydedilmiş sayar. Eski Gateway'ler `node.event` isteğini `{ "ok": true }` ile onaylayabilir; bu yanıt uyumludur ancak kalıcı bir son görülme güncellemesi sayılmaz.

### 4. Eşleştirmeyi onaylama (CLI)

Gateway makinesinde:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

Eşleştirme ayrıntıları: [Eşleştirme](/tr/channels/pairing).

İsteğe bağlı: Android Node her zaman sıkı biçimde denetlenen bir alt ağdan bağlanıyorsa açık CIDR'ler veya tam IP'lerle ilk Node eşleştirmesinin otomatik onaylanmasını etkinleştirebilirsiniz:

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

Bu, varsayılan olarak devre dışıdır. Yalnızca talep edilen kapsam bulunmayan yeni `role: node` eşleştirmelerine uygulanır. Operatör/tarayıcı eşleştirmesi ve her türlü rol, kapsam, meta veri veya açık anahtar değişikliği yine manuel onay gerektirir.

### 5. Node'un bağlı olduğunu doğrulama

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

### 6. Sohbet + geçmiş

Android Sohbet sekmesi, oturum seçimini destekler (varsayılan `main` ve mevcut diğer oturumlar):

- Geçmiş: `chat.history` (görüntüleme için normalleştirilmiştir — satır içi yönerge etiketleri, düz metin araç çağrısı XML yükleri (`<tool_call>`, `<function_call>`, `<tool_calls>`, `<function_calls>` ve kesilmiş varyantları) ve sızmış ASCII/tam genişlikli model kontrol token'ları kaldırılır; tam olarak `NO_REPLY` / `no_reply` olan sessiz token'lı asistan satırları atlanır; aşırı büyük satırlar yer tutucularla değiştirilebilir)
- Gönderme: `chat.send`
- Kalıcı gönderim: her gönderim (metin, seçilmiş görseller ve sesli notlar), herhangi bir ağ denemesinden önce Gateway başına cihaz üzerindeki bir giden kutusuna kaydedilir; böylece uygulamanın sonlandırılması gönderilmiş girdilerin kaybolmasına neden olmaz. Çevrimdışıyken kuyruğa alınan gönderimler, kararlı idempotency anahtarlarıyla yeniden bağlanıldığında sırayla teslim edilir ve bir gönderim yalnızca tur, standart `chat.history` içinde görünür olduktan sonra tamamlanmış sayılır; tek başına onay, teslimat kanıtı olarak değerlendirilmez. Belirsiz sonuçlar (kaybolan onay, gönderim sırasında uygulamanın kapatılması, döküm yazılmadan önce Gateway'in yeniden başlatılması), otomatik yeniden gönderim yerine açık **Yeniden dene**/**Sil** seçenekleri bulunan görünür satırlar olarak gösterilir. Eğik çizgi komutları yeniden bağlantı sonrasında asla otomatik olarak yeniden yürütülmez; açıkça yeniden denenmek üzere bekletilir. Kuyruk sınırlıdır (Gateway başına 50 mesaj ve 48 MB ek baytı) ve gönderilmemiş satırların süresi 48 saat sonra dolar. Hiç gönderilmemiş düzenleyici taslakları süreçler arasında kalıcı değildir.
- Anlık güncellemeler (en iyi çaba): `chat.subscribe` -> `event:"chat"`
- Dinleme: dinlemek için bir asistan mesajına uzun basın ve **Dinle** seçeneğini belirleyin; ses, yapılandırılmış TTS sağlayıcı zinciriyle Gateway `tts.speak` üzerinden oluşturulur ve Gateway ses oluşturamadığında cihazdaki sistem TTS'si kullanılır. Oturum değiştirildiğinde, yeni sohbet başlatıldığında, uygulama arka plana geçtiğinde veya sohbet kapatıldığında oynatma durur.

### 7. Canvas + kamera

#### Gateway Canvas Sunucusu (web içeriği için önerilir)

Node'un, ajanın diskte düzenleyebileceği gerçek HTML/CSS/JS göstermesini sağlamak için Node'u Gateway Canvas sunucusuna yönlendirin.

<Note>
Node'lar Canvas'ı Gateway HTTP sunucusundan yükler (`gateway.port` ile aynı bağlantı noktası, varsayılan `18789`).
</Note>

1. Gateway ana makinesinde `~/.openclaw/workspace/canvas/index.html` oluşturun.
2. Node'u buna yönlendirin (LAN):

```bash
openclaw nodes invoke --node "<Android Node>" --command canvas.navigate --params '{"url":"http://<gateway-hostname>.local:18789/__openclaw__/canvas/"}'
```

Tailnet (isteğe bağlı): her iki cihaz da Tailscale üzerindeyse `.local` yerine bir MagicDNS adı veya tailnet IP'si kullanın; ör. `http://<gateway-magicdns>:18789/__openclaw__/canvas/`.

Bu sunucu HTML'e canlı yeniden yükleme istemcisi ekler ve dosya değişikliklerinde sayfayı yeniden yükler. Gateway ayrıca `/__openclaw__/a2ui/` sunar ancak Android uygulaması uzak A2UI sayfalarını yalnızca görüntülenebilir olarak ele alır. Eylem özellikli A2UI komutları, paketle gelen ve uygulamaya ait A2UI sayfasını kullanır.

Canvas komutları (yalnızca ön planda):

- `canvas.eval`, `canvas.snapshot`, `canvas.navigate` (varsayılan iskelete dönmek için `{"url":""}` veya `{"url":"/"}` kullanın). `canvas.snapshot`, `{ format, base64 }` döndürür (varsayılan `format="jpeg"`).
- A2UI: `canvas.a2ui.push`, `canvas.a2ui.reset` (`canvas.a2ui.pushJSONL` eski diğer adıdır). Bunlar eylem özellikli görüntüleme için paketle gelen ve uygulamaya ait A2UI sayfasını kullanır.

Kamera komutları (yalnızca ön planda; izne tabidir): `camera.snap` (jpg), `camera.clip` (mp4). Parametreler ve CLI yardımcıları için [Kamera Node'u](/tr/nodes/camera) bölümüne bakın.

### 8. Ses + genişletilmiş Android komut yüzeyi

- Android kabuk gezintisi **Ana Sayfa**, **Sohbet** ve **Ayarlar** bölümlerinden oluşur. Sesli giriş
  Sohbet düzenleyicisine aittir; ayrı bir Ses sekmesi yoktur.
- Taslağa döküm ekleyen cihaz üzerindeki konuşma tanımayı kullanmak için düzenleyici mikrofonuna dokunun. Sesli not
  eki kaydetmek için mikrofona uzun basın. Kullanıcı arayüzü, denemeyi sessizce yok saymak yerine kullanılamayan tanımayı, eksik izni,
  meşgul/ağ hatalarını ve konuşma algılanmaması sonuçlarını bildirir.
- Sohbet dalga biçiminden sürekli **Konuşma** modunu başlatın. Dikte, sesli not
  kaydı ve Konuşma, birbirini dışlayan mikrofon yollarıdır.
- Konuşma Modu, yakalama başlamadan önce mevcut ön plan hizmetini `connectedDevice` düzeyinden `connectedDevice|microphone` düzeyine yükseltir ve Konuşma Modu durduğunda eski düzeyine indirir. Node hizmeti, `CHANGE_NETWORK_STATE` ile `FOREGROUND_SERVICE_CONNECTED_DEVICE` bildirir; Android 14+ ayrıca `FOREGROUND_SERVICE_MICROPHONE` bildirimini, `RECORD_AUDIO` çalışma zamanı iznini ve çalışma zamanında mikrofon hizmeti türünü gerektirir.
- Android Konuşma varsayılan olarak yerel konuşma tanımayı, Gateway sohbetini ve yapılandırılmış Gateway Konuşma sağlayıcısı aracılığıyla `talk.speak` kullanır. Yerel sistem TTS'si yalnızca `talk.speak` kullanılamadığında kullanılır.
- Android Konuşma, gerçek zamanlı Gateway aktarımını yalnızca `talk.realtime.mode` değeri `realtime` ve `talk.realtime.transport` değeri `gateway-relay` olduğunda kullanır.
- Android, `voiceWake` yeteneğini duyurmaz. Sesli giriş için Sohbet diktesini,
  sesli notu veya Konuşma'yı kullanın.
- Ek Android komut aileleri (kullanılabilirlik cihaza, izinlere ve kullanıcı ayarlarına bağlıdır):
  - `device.status`, `device.info`, `device.permissions`, `device.health`
  - `device.apps` yalnızca **Ayarlar > Telefon Yetenekleri > Yüklü Uygulamalar** etkinleştirildiğinde kullanılabilir; varsayılan olarak başlatıcıda görünen uygulamaları listeler (tam liste için `includeNonLaunchable` geçirin).
  - `notifications.list`, `notifications.actions` (aşağıdaki [Bildirim yönlendirme](#notification-forwarding) bölümüne bakın)
  - `photos.latest`
  - `contacts.search`, `contacts.add`
  - `calendar.events`, `calendar.add`
  - `callLog.search`
  - `sms.search`
  - `motion.activity`, `motion.pedometer`

### 9. Çalışma alanı dosyaları (salt okunur)

Ana Sayfa genel görünümü, etkin ajanın çalışma alanında salt okunur `agents.workspace.list` / `agents.workspace.get` Gateway RPC'leri aracılığıyla gezinmeyi sağlayan bir **Dosyalar** kartı içerir: dizinlerde ayrıntılı gezinme, metin ve görsel önizlemeleri ve Android paylaşım sayfası üzerinden dışa aktarma. Yazma işlemi yoktur ve önizlemelerin boyutu Gateway tarafından sınırlandırılır.

## Komut onaylarını inceleme

`operator.admin` özellikli bir operatör bağlantısı veya Gateway tarafından açıkça hedeflenen eşleştirilmiş
bir `operator.approvals` bağlantısı, bekleyen yürütme isteklerini **Ayarlar -> Onaylar** altında inceleyebilir. Uygulama, düğmelerini etkinleştirmeden önce
Gateway'in temizlenmiş onay kaydını yükler, tüm güvenlik uyarılarını ve bu isteğin sunduğu kararları eksiksiz biçimde gösterir ve
onay kimliğiyle sahip türünü Gateway'e geri gönderir.

Onay durumu Control UI ve desteklenen sohbet yüzeyleriyle paylaşılır. İlk kaydedilen
yanıt geçerli olur; başka bir yüzey daha önce yanıtlamış olsa bile Android bu standart sonucu gösterir. Çözüm yanıtı kaybolursa veya Gateway
bağlantısı kesilirse uygulama eylemi kilitli tutar ve başka bir karar sunmadan önce
onayı yeniden okur.

Birleştirilmiş onay yöntemlerinden önceki Gateway'ler, yayımlanmış
yürütmeye özgü yöntemlere geri döner. Bekleyen inceleme çalışmaya devam eder ancak kalıcı terminal durumu
ve daha kapsamlı yüzeyler arası sonuç, güncel bir Gateway gerektirir.

## Ajan sorularını yanıtlama

Sohbet, `operator.questions` (veya `operator.admin`) özellikli operatör bağlantılarında bekleyen Gateway sorularını yerel kartlar olarak gösterir. Kartlar tekli ve
çoklu seçim seçeneklerini, seçenek açıklamalarını, serbest metin **Diğer** yanıtlarını ve
süre sonu geri sayımını destekler. Yeniden bağlantılar, bekleyen soruları Gateway'den yeniden yükler. Bir kart;
bu cihaz yanıtladığında, başka bir yüzey önce yanıtladığında veya
sorunun süresi dolduğunda ya da soru iptal edildiğinde kilitlenir.

## Asistan giriş noktaları

Android, OpenClaw'ın sistem asistanı tetikleyicisinden (Google Assistant) başlatılmasını destekler. Ana ekran düğmesini basılı tutmak (veya başka bir `ACTION_ASSIST` tetikleyicisi) uygulamayı açar; "Hey Google, ask OpenClaw `<prompt>`" demek, uygulamanın bildirdiği App Actions sorgu kalıbıyla eşleşir ve istemi otomatik göndermeden sohbet düzenleyicisine aktarır.

Bu, uygulama manifestinde bildirilen Android **App Actions** (`shortcuts.xml` yeteneği) özelliğini kullanır. Gateway tarafında yapılandırma gerekmez — asistan niyeti tamamen Android uygulaması tarafından işlenir.

<Note>
App Actions'ın kullanılabilirliği cihaza, Google Play Services sürümüne ve kullanıcının OpenClaw'ı varsayılan asistan uygulaması olarak ayarlayıp ayarlamadığına bağlıdır.
</Note>

## Bildirim yönlendirme

Android, cihaz bildirimlerini Gateway'e `node.event` öğeleri olarak yönlendirebilir. Bu, Gateway/`openclaw.json` yapılandırmasında değil, uygulamanın Ayarlar sayfasında **cihaz üzerinde** yapılandırılır.

| Ayar                         | Açıklama                                                                                                                                                                                                    |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bildirim Olaylarını İlet     | Ana anahtar. Varsayılan olarak kapalıdır; önce Bildirim Dinleyicisi Erişimi verilmelidir.                                                                                                                    |
| Paket Filtresi               | **İzin listesi** (yalnızca listelenen paket kimlikleri iletilir) veya **Engelleme listesi** (varsayılan: listelenen kimlikler dışındaki tüm paketler). İletim döngülerini önlemek için OpenClaw'ın kendi paketi Engelleme listesi modunda her zaman hariç tutulur. |
| Sessiz Saatler               | İletimi engelleyen yerel HH:mm başlangıç/bitiş zaman aralığı. Varsayılan olarak devre dışıdır; etkinleştirildiğinde varsayılan değerler `22:00`-`07:00` olur.                           |
| Dakika Başına En Fazla Olay  | İletilen bildirimler için cihaz başına hız sınırı. Varsayılan 20'dir.                                                                                                                                        |
| Yönlendirme Oturumu Anahtarı | İsteğe bağlıdır. İletilen bildirim olaylarını cihazın varsayılan bildirim rotası yerine belirli bir oturuma sabitler.                                                                                        |

<Note>
Bildirim iletimi, Android Bildirim Dinleyicisi iznini gerektirir. Uygulama, kurulum sırasında bu izni vermenizi ister.
</Note>

WhatsApp, WhatsApp Business, Telegram, Telegram X, Discord ve Signal bildirimleri her zaman hariç tutulur. Bunların mesajları zaten yerel OpenClaw kanal oturumlarına aittir; Android bildiriminin ayrı bir Node olayı olarak iletilmesi, yanıtı yanlış görüşme üzerinden yönlendirebilir.

## İlgili

- [iOS uygulaması](/tr/platforms/ios)
- [Node'lar](/tr/nodes)
- [Android Node sorun giderme](/tr/nodes/troubleshooting)
