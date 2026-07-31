---
read_when:
    - DM erişim denetimini ayarlama
    - Yeni bir iOS/Android Node'unu eşleştirme
    - OpenClaw güvenlik duruşunu inceleme
summary: 'Eşleştirmeye genel bakış: Size kimlerin DM gönderebileceğini ve hangi Node''ların katılabileceğini onaylayın'
title: Eşleştirme
x-i18n:
    generated_at: "2026-07-26T23:10:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dc874d660509f59bc26795c8b3ce13f5d238cd61154c717637f5d545b995fb08
    source_path: channels/pairing.md
    workflow: 16
---

"Eşleştirme", OpenClaw'ın açık erişim onayı adımıdır.
İki yerde kullanılır:

1. **DM eşleştirmesi** (botla kimlerin konuşmasına izin verildiği)
2. **Node eşleştirmesi** (hangi cihazların/Node'ların Gateway ağına katılmasına izin verildiği)

Güvenlik bağlamı: [Güvenlik](/tr/gateway/security)

## 1) DM eşleştirmesi (gelen sohbet erişimi)

Bir kanalın DM politikası `pairing` olarak yapılandırıldığında, bilinmeyen göndericiler kısa bir kod alır ve siz onaylayana kadar mesajları **işlenmez**.

Varsayılan DM politikaları burada belgelenmiştir: [Güvenlik](/tr/gateway/security)

`dmPolicy: "open"`, yalnızca geçerli DM izin listesi `"*"` içerdiğinde herkese açıktır.
Kurulum ve doğrulama, herkese açık yapılandırmalar için bu joker karakteri gerektirir. Mevcut
durum, belirli `allowFrom` girdileriyle birlikte `open` içeriyorsa çalışma zamanı yine
yalnızca bu göndericileri kabul eder ve eşleştirme deposundaki onaylar `open` erişimini genişletmez.

Eşleştirme kodları:

- 8 karakterden oluşur, büyük harflidir ve belirsiz karakter içermez (`0O1I`).
- **1 saat sonra geçerliliğini yitirir**. Bot, eşleştirme mesajını yalnızca yeni bir istek oluşturulduğunda gönderir (gönderici başına yaklaşık saatte bir).
- Bekleyen DM eşleştirme istekleri **kanal hesabı başına 3** ile sınırlıdır; ek istekler, bunlardan biri zaman aşımına uğrayana veya onaylanana kadar yok sayılır.

### Control UI'dan onaylama

**Ayarlar → Kanallar → DM erişim istekleri** bölümünü açın. Kuyruk, DM politikası `pairing`
olan tüm yapılandırılmış kanal hesaplarındaki bekleyen istekleri birleştirir.
Kanala veya hesaba göre filtreleyin, gönderici kimliğini ve meta verileri inceleyin, ardından
**Onayla** seçeneğini belirleyin.

Onay yalnızca doğrudan mesaj erişimi verir. Grup erişimi vermez. Desteklendiğinde,
onay iletişim kutusu şu açık seçenekleri de sunar:

- **Onaydan sonra istekte bulunanı bilgilendir**
- **Bu göndericiyi aynı zamanda ilk komut sahibi yap**; yalnızca komut
  sahibi yoksa ve Control UI oturumunda `operator.admin` varsa gösterilir

Bekleyen bir isteği onaylamadan kaldırmak için **Reddet** seçeneğini belirleyin. Reddetme
kalıcı bir engelleme değildir; gönderici daha sonra yeniden erişim isteyebilir.

### CLI'dan onaylama

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

İstekte bulunanı aynı kanalda bilgilendirmek için `--notify` ekleyin. Çok hesaplı kanallar
`--account <id>` kabul eder.

Control UI'daki açık onay kutusunun aksine CLI, herhangi bir komut sahibi yapılandırılmadığında
`commands.ownerAllowFrom` için `telegram:123456789` gibi bir girdi kullanarak otomatik olarak
başlangıç kurulumu yapar. Bu, ilk kurulumlara ayrıcalıklı komutlar ve çalıştırma onayı
istemleri için açık bir sahip verir. Bir sahip oluşturulduktan sonra sonraki
eşleştirme onayları yalnızca DM erişimi verir; başka sahipler eklemez.

<Note>
WhatsApp'ın oturum açma QR kodu, bir WhatsApp hesabını OpenClaw'a bağlar. DM erişim istekleri,
bu hesaba mesaj gönderen kişileri onaylar. Bunlar ayrı akışlardır.
</Note>

Desteklenen kanallar (eşleştirme bildiren herhangi bir yüklü kanal Plugin'i; `openclaw-weixin` gibi harici Plugin'ler daha fazlasını ekleyebilir): `discord`, `feishu`, `googlechat`, `imessage`, `irc`, `line`, `matrix`, `mattermost`, `msteams`, `nextcloud-talk`, `nostr`, `signal`, `slack`, `sms`, `synology-chat`, `telegram`, `twitch`, `whatsapp`, `zalo`, `zalouser`.

### Yeniden kullanılabilir gönderici grupları

Aynı güvenilir gönderici kümesinin birden çok mesajlaşma kanalına veya hem DM hem de
grup izin listelerine uygulanması gerektiğinde üst düzey `accessGroups` kullanın.

Statik gruplar `type: "message.senders"` kullanır ve kanal izin listelerinden
`accessGroup:<name>` ile başvurulur:

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
        whatsapp: ["+15551234567"],
      },
    },
  },
  channels: {
    telegram: { dmPolicy: "allowlist", allowFrom: ["accessGroup:operators"] },
    whatsapp: { groupPolicy: "allowlist", groupAllowFrom: ["accessGroup:operators"] },
  },
}
```

Erişim grupları burada ayrıntılı olarak belgelenmiştir: [Erişim grupları](/tr/channels/access-groups)

### Durumun saklandığı yer

Paylaşılan SQLite durum veritabanında
`~/.openclaw/state/openclaw.sqlite` konumunda saklanır:

- `channel_pairing_requests` içindeki bekleyen istekler
- `channel_pairing_allow_entries` içindeki onaylanmış göndericiler

Hesap kapsamı davranışı:

- her istek ve onaylanmış gönderici, kanal ve hesaba göre anahtarlanır
- çalışma zamanı yalnızca standart SQLite satırlarını okur; eski dosyaları birleştirmez

Eski Gateway'ler, `~/.openclaw/credentials/` altında `<channel>-pairing.json` ve
`<channel>-<accountId>-allowFrom.json` yazıyordu.
Başlangıç geçişi ve `openclaw doctor --fix`, bu dosyaları SQLite'a aktarır ve
başarılı bir aktarımdan sonra her kaynak dosyayı kaldırır. Bu satırlar asistanınıza
erişimi denetlediği için SQLite veritabanını hassas olarak değerlendirin.

<Note>
Eşleştirme izin listesi deposu DM erişimi içindir. Grup yetkilendirmesi ayrıdır.
Bir DM eşleştirme kodunu onaylamak, bu göndericinin grup komutlarını çalıştırmasına
veya gruplarda botu denetlemesine otomatik olarak izin vermez. İlk sahip başlangıç kurulumu,
`commands.ownerAllowFrom` içindeki ayrı bir yapılandırma durumudur ve grup sohbeti teslimi yine
kanalın grup izin listelerini izler (örneğin kanala bağlı olarak `groupAllowFrom`, `groups`
veya grup ya da konu başına geçersiz kılmalar).
</Note>

## 2) Node cihaz eşleştirmesi (iOS/Android/macOS/başsız Node'lar)

Node'lar, `role: node` ile **cihazlar** olarak Gateway'e bağlanır. Gateway,
onaylanması gereken bir cihaz eşleştirme isteği oluşturur.

### Control UI'dan eşleştirme (önerilir)

`operator.admin` erişimine sahip, zaten bağlı bir Control UI oturumu kullanın:

1. Control UI'ı açın ve **Ayarlar → Cihazlar** bölümüne gidin.
2. **Cihazlar** sayfasında **Mobil cihaz eşleştir** seçeneğine tıklayın.
3. **Tam erişim (önerilir)** seçeneğini koruyun veya yönetimsel Gateway
   denetimlerini hariç tutmak için **Sınırlı erişim** seçeneğini belirleyin.
4. **Kurulum kodu oluştur** seçeneğine tıklayın.
5. Telefonunuzda OpenClaw uygulamasını açın → **Ayarlar** → **Gateway**.
6. QR kodunu tarayın veya kurulum kodunu yapıştırın, ardından bağlanın.

Resmî OpenClaw iOS ve Android uygulamaları, kurulum kodu meta verileri eşleştiğinde
otomatik olarak onaylanır. **Onay bekliyor** bölümünde bir istek görünürse (örneğin
resmî olmayan bir istemci veya eşleşmeyen meta veriler için), onaylamadan önce rolünü ve
kapsamlarını inceleyin.

Geçerli Control UI oturumunun yönetici erişimi olmadığında düğme devre dışı bırakılır.
Bu durumda Gateway ana makinesinden aşağıdaki CLI onay akışını kullanın.

### Telegram üzerinden eşleştirme

`device-pair` Plugin'ini kullanıyorsanız ilk cihaz eşleştirmesini tamamen Telegram üzerinden yapabilirsiniz:

1. Telegram'da botunuza şu mesajı gönderin: `/pair`
2. Bot iki mesajla yanıt verir: bir talimat mesajı ve ayrı bir **kurulum kodu** mesajı (Telegram'da kolayca kopyalanıp yapıştırılabilir).
3. Telefonunuzda OpenClaw iOS uygulamasını açın → Ayarlar → Gateway.
4. QR kodunu (`/pair qr`) tarayın veya kurulum kodunu yapıştırıp bağlanın.
5. Resmî mobil uygulama otomatik olarak bağlanır. `/pair pending` bir
   istek gösterirse onaylamadan önce rolünü ve kapsamlarını inceleyin.

Kurulum kodu, şunları içeren base64 kodlu bir JSON yüküdür:

- `url`: Gateway WebSocket URL'si (`ws://...` veya `wss://...`)
- `urls`: kullanılabildiğinde, mobil uygulamanın deneyebileceği sıralı LAN/Tailnet rotaları
- `bootstrapToken`: ilk eşleştirme el sıkışması için tek kullanımlık başlangıç token'ı; Gateway bunun süresini 10 dakika sonra sona erdirir

Eşleştirme tamamlandıktan sonra kullanılmamış kurulum kodlarını geçersiz kılmak için `/pair cleanup` çalıştırın.

Bu başlangıç token'ı, yerleşik eşleştirme başlangıç profilini taşır:

- güvenli bir `wss://` kurulumu (veya aynı ana makinedeki geri döngü), varsayılan olarak `node` ile birlikte tam
  yerel mobil `operator` erişimi kullanır
- devredilen `node` token'ı `scopes: []` olarak kalır
- varsayılan olarak devredilen `operator` token'ı `operator.admin`,
  `operator.approvals`, `operator.read`, `operator.talk.secrets` ve
  `operator.write` içerir
- Control UI **Sınırlı erişim** ve `openclaw qr --limited`,
  diğer operatör kapsamlarını korurken `operator.admin` kapsamını hariç tutar
- düz metin LAN `ws://` kurulumu otomatik olarak aynı sınırlı profili kullanır;
  tam erişim için `wss://` veya Tailscale Serve yapılandırın ve yeni bir kod oluşturun
- sonraki token döndürme/iptal işlemleri hem cihazın onaylanmış
  rol sözleşmesiyle hem de çağıran oturumun operatör kapsamlarıyla sınırlı kalır

Kurulum kodu geçerli olduğu sürece ona parola gibi davranın.

iOS ve Android **Ayarlar → Gateway** sayfaları **Tam** veya **Sınırlı**
erişimi gösterir. Sınırlı bir telefonun erişimini yükseltmek için önce güvenli bir `wss://`
veya Tailscale Serve rotası yapılandırın, ardından yeni bir tam erişimli kurulum kodu oluşturun,
bu ayarlar sayfasında tarayın ya da yapıştırın ve yeniden bağlanın.

Tailscale, herkese açık veya diğer uzak mobil eşleştirmeler için Tailscale Serve/Funnel
ya da başka bir `wss://` Gateway URL'si kullanın. Düz metin `ws://` kurulum kodları yalnızca
geri döngü, özel LAN adresleri, `.local` Bonjour ana makineleri ve Android
emülatör ana makinesi için kabul edilir. Geri döngü dışındaki düz metin rotalarına sınırlı erişim verilir. Tailnet
CGNAT adresleri, `.ts.net` adları ve genel ana makineler, QR/kurulum kodu
verilmeden önce yine güvenli biçimde reddedilir.

`gateway.bind=lan` kurulum URL'leri için OpenClaw, etkin Gateway'in geri döngü portuna
proxy uygulayan kalıcı Tailscale Serve HTTPS köklerini algılar ve bunları
LAN rotasıyla birlikte duyurur. Kurulum komutu bu yedek rotayı yalnızca
`lan` için ekler; `custom` ve `tailnet` açıkça duyurulan rotalarını korur. iOS
uygulaması duyurulan rotaları sırayla yoklar ve erişilebilen ilk
uç noktayı kaydeder.

### Bir Node cihazını onaylama

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

Açık bir onay, onaylayan eşleştirilmiş cihaz oturumu yalnızca eşleştirme kapsamıyla
açıldığı için reddedilirse CLI aynı isteği `operator.admin` ile yeniden dener.
Bu, yönetici yeteneğine sahip mevcut bir eşleştirilmiş cihazın, eşleştirme deposunu
elle düzenlemeden yeni bir Control UI/tarayıcı eşleştirmesini kurtarmasını sağlar.
Gateway yeniden denenen bağlantıyı yine doğrular; `operator.admin` ile kimlik doğrulaması
yapamayan token'lar engellenmeye devam eder.

Aynı cihaz farklı kimlik doğrulama ayrıntılarıyla yeniden denerse (örneğin farklı
rol/kapsamlar/açık anahtar), önceki bekleyen isteğin yerini yenisi alır ve yeni bir
`requestId` oluşturulur.

<Note>
Zaten eşleştirilmiş bir cihaza sessizce daha geniş erişim verilmez. Daha fazla kapsam veya daha geniş bir rol isteyerek yeniden bağlanırsa OpenClaw mevcut onayı olduğu gibi korur ve yeni bir bekleyen yükseltme isteği oluşturur. Onaylamadan önce mevcut onaylı erişim ile yeni istenen erişimi karşılaştırmak için `openclaw devices list` kullanın.
</Note>

### İsteğe bağlı güvenilir CIDR Node otomatik onayı

Cihaz eşleştirmesi varsayılan olarak manuel kalır. Sıkı biçimde denetlenen Node ağları için
açık CIDR'ler veya tam IP'lerle ilk Node otomatik onayını etkinleştirebilirsiniz:

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

Bu yalnızca istenen kapsamı olmayan yeni `role: node` eşleştirme istekleri için
geçerlidir. Operatör, tarayıcı, Control UI ve WebChat istemcileri yine manuel
onay gerektirir. Rol, kapsam, meta veri ve açık anahtar değişiklikleri de yine manuel
onay gerektirir.

### Node eşleştirme durumu depolaması

Paylaşılan SQLite durum veritabanında `~/.openclaw/state/openclaw.sqlite` konumunda saklanır:

- bekleyen cihaz eşleştirme istekleri (kısa ömürlüdür; 5 dakika sonra geçerliliklerini yitirir)
- eşleştirilmiş cihazlar + token'lar

Eski gateway'ler bu durumu `~/.openclaw/devices/*.json` içinde tutuyordu; bu dosyalar
gateway başlatılırken SQLite'a aktarılır ve `.migrated` son ekiyle arşivlenir.

### Notlar

- `node.pair.*` API'si (CLI: `openclaw nodes pending|approve|reject|remove|rename`), aynı eşleştirilmiş cihaz kayıtlarında depolanan
  Node yetenek onaylarını yönetir. WS Node'ları için cihaz eşleştirmesi
  hâlâ gereklidir; bkz. [Node eşleştirme](/tr/gateway/pairing).
- Eşleştirme kaydı, onaylanan roller için kalıcı doğruluk kaynağıdır. Etkin
  cihaz belirteçleri bu onaylanmış rol kümesiyle sınırlı kalır; onaylanmış rollerin
  dışındaki başıboş bir belirteç girdisi yeni erişim oluşturmaz.

## İlgili belgeler

- Güvenlik modeli + istem enjeksiyonu: [Güvenlik](/tr/gateway/security)
- Güvenli güncelleme (doctor'ı çalıştırın): [Güncelleme](/tr/install/updating)
- Kanal yapılandırmaları:
  - Telegram: [Telegram](/tr/channels/telegram)
  - WhatsApp: [WhatsApp](/tr/channels/whatsapp)
  - Signal: [Signal](/tr/channels/signal)
  - iMessage: [iMessage](/tr/channels/imessage)
  - Discord: [Discord](/tr/channels/discord)
  - Slack: [Slack](/tr/channels/slack)
