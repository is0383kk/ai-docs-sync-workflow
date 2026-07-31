---
read_when:
    - WhatsApp/web kanal davranışı veya gelen kutusu yönlendirmesi üzerinde çalışma
summary: WhatsApp kanal desteği, erişim denetimleri, teslimat davranışı ve operasyonlar
title: WhatsApp
x-i18n:
    generated_at: "2026-07-26T23:10:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7489b37f91775868d0694daea8a0958ee000d1411674d1800bb1e77df5961e68
    source_path: channels/whatsapp.md
    workflow: 16
---

Durum: WhatsApp Web (Baileys) üzerinden üretime hazır. Gateway, bağlı oturumların sahibidir; ayrı bir Twilio WhatsApp kanalı yoktur.

## Kurulum

`openclaw onboard` ve `openclaw channels add --channel whatsapp`, ilk kez seçtiğinizde plugini kurmanızı ister; plugin eksikse `openclaw channels login --channel whatsapp` aynı kurulum akışını sunar. Geliştirme çalışma kopyaları yerel plugin yolunu kullanır; kararlı/beta kurulumları önce ClawHub'dan `@openclaw/whatsapp` kurar, başarısız olursa npm'e geri döner. WhatsApp çalışma zamanı, temel OpenClaw npm paketinin dışında sunulduğundan çalışma zamanı bağımlılıkları harici pluginle birlikte kalır. Manuel kurulum:

```bash
openclaw plugins install clawhub:@openclaw/whatsapp
```

Çıplak npm paketini (`@openclaw/whatsapp`) yalnızca kayıt defteri geri dönüşü için kullanın; tam bir sürümü yalnızca tekrarlanabilir bir kurulum için sabitleyin.

<CardGroup cols={3}>
  <Card title="Eşleştirme" icon="link" href="/tr/channels/pairing">
    Bilinmeyen gönderenler için varsayılan DM politikası eşleştirmedir.
  </Card>
  <Card title="Kanal sorunlarını giderme" icon="wrench" href="/tr/channels/troubleshooting">
    Kanallar arası tanılama ve onarım çalışma kitapları.
  </Card>
  <Card title="Gateway yapılandırması" icon="settings" href="/tr/gateway/configuration">
    Eksiksiz kanal yapılandırma kalıpları ve örnekleri.
  </Card>
</CardGroup>

## Hızlı kurulum

<Steps>
  <Step title="Erişim politikasını yapılandırın">

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

  </Step>

  <Step title="WhatsApp'ı bağlayın (QR)">

```bash
openclaw channels login --channel whatsapp
```

    Oturum açma yalnızca QR ile yapılır. Uzak veya başsız ana makinelerde oturum açmayı başlatmadan önce canlı QR'ı telefona iletecek güvenilir bir yol bulunduğundan emin olun; terminalde oluşturulan QR'ların, ekran görüntülerinin veya sohbet eklerinin süresi aktarım sırasında dolabilir.

    Belirli bir hesap için:

```bash
openclaw channels login --channel whatsapp --account work
```

    Oturum açmadan önce mevcut/özel bir kimlik doğrulama dizini eklemek için:

```bash
openclaw channels add --channel whatsapp --account work --auth-dir /path/to/wa-auth
openclaw channels login --channel whatsapp --account work
```

  </Step>

  <Step title="Gateway'i başlatın">

```bash
openclaw gateway
```

  </Step>

  <Step title="İlk DM erişim isteğini onaylayın (eşleştirme modu)">

    **Settings → Channels → DM access requests** öğesini açın, WhatsApp hesabını bulun
    ve göndereni onaylayın. CLI'yi tercih ederseniz:

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

    DM erişim isteklerinin süresi 1 saat sonra dolar; bekleyen istekler hesap
    başına 3 ile sınırlıdır. Bu onay, hesabın kendisini bağlamak için kullanılan
    WhatsApp oturum açma QR'ından ayrıdır.

  </Step>
</Steps>

<Note>
Ayrı bir WhatsApp numarası önerilir (kurulum ve meta veriler buna göre optimize edilmiştir), ancak kişisel numara/kendi kendine sohbet kurulumları tam olarak desteklenir.
</Note>

## Dağıtım kalıpları

<AccordionGroup>
  <Accordion title="Özel numara (önerilen)">
    - OpenClaw için ayrı WhatsApp kimliği
    - daha anlaşılır DM izin listeleri ve yönlendirme sınırları
    - kendi kendine sohbet karışıklığı olasılığının daha düşük olması

    ```json5
    {
      channels: {
        whatsapp: {
          dmPolicy: "allowlist",
          allowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Kişisel numaraya geri dönüş">
    İlk katılım, kişisel numara modunu destekler ve kendi kendine sohbete uygun bir temel yapılandırma yazar: `dmPolicy: "allowlist"`, kendi numaranızı içeren `allowFrom`, `selfChatMode: true`. Çalışma zamanındaki kendi kendine sohbet korumaları, bağlı kişisel numarayı ve `allowFrom` değerini anahtar olarak kullanır.
  </Accordion>
</AccordionGroup>

## Çalışma zamanı modeli

- Gateway, WhatsApp soketinin ve yeniden bağlanma döngüsünün sahibidir.
- Bir gözetleyici iki sinyali bağımsız olarak izler: ham WhatsApp Web aktarım etkinliği ve uygulama mesajı etkinliği. Sessiz ancak bağlı bir oturum, yalnızca yakın zamanda mesaj gelmediği için yeniden başlatılmaz; yalnızca aktarım çerçeveleri sabit bir dahili süre boyunca (kullanıcı tarafından yapılandırılamaz) gelmeyi durdurduğunda veya uygulama mesajları normal mesaj zaman aşımının 4 katını aşacak kadar sessiz kaldığında yeniden bağlanmayı zorlar. Yakın zamanda etkin olan bir oturum yeniden bağlandıktan hemen sonra bu ilk süre, 4 katlık süre yerine daha kısa olan normal mesaj zaman aşımını kullanır. OpenClaw, Baileys'in bu yeniden bağlanmanın erken aşamasında ilettiği çevrimdışı mesajlara, gelen mesaj kimliği yinelenenleri ayıklama ömrüyle sınırlı olarak otomatik yanıt verebilir; ilk başlatma kısa eski geçmiş korumasını sürdürür.
- Giden gönderimler, hedef hesap için etkin bir WhatsApp dinleyicisi gerektirir; aksi takdirde gönderimler hızla başarısız olur.
- Grup gönderimleri, belirteç mevcut katılımcı meta verileriyle eşleştiğinde `@+<digits>` ve `@<digits>` belirteçleri için (metin ve medya açıklamalarında) LID destekli gruplar dahil olmak üzere yerel bahsetme meta verilerini ekler.
- Durum ve yayın sohbetleri (`@status`, `@broadcast`) yok sayılır.
- Doğrudan sohbetler DM oturum kurallarını kullanır (`session.dmScope`; varsayılan `main`, DM'leri aracının ana oturumunda birleştirir). Grup oturumları her JID için yalıtılır (`agent:<agentId>:whatsapp:group:<jid>`).
- WhatsApp Kanalları/Bültenleri, DM semantiği yerine kanal oturumu meta verilerini (`agent:<agentId>:whatsapp:channel:<jid>`) kullanarak yerel `@newsletter` JID'leri aracılığıyla açık giden hedefler olabilir.
- WhatsApp Web aktarımı, Gateway ana makinesindeki standart proxy ortam değişkenlerine (`HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY`, küçük harfli çeşitleri) uyar. Kanal başına ayarlar yerine ana makine düzeyinde proxy yapılandırmasını tercih edin.

## Mevcut istekte bulunanı MeowCaller ile arayın (deneysel)

Plugin, WhatsApp kaynaklı aracı dönüşlerinde `whatsapp_call` sunabilir. Mevcut yetkili istekte bulunana WhatsApp sesli araması yapmak ve yanıt verdikten sonra bir OpenClaw TTS mesajı oynatmak için [MeowCaller](https://github.com/purpshell/meowcaller) kullanır. Aracın hedef numara parametresi yoktur, bu nedenle bir istem çağrıyı yeniden yönlendiremez. Varsayılan olarak devre dışıdır.

<Warning>
MeowCaller deneyseldir, etiketlenmiş bir sürümü yoktur ve ayrıca eşleştirilmiş bir whatsmeow bağlı cihaz oturumu kullanır; pluginin Baileys kimlik bilgilerini yeniden kullanamaz. Eşleştirme, aynı WhatsApp hesabına başka bir bağlı cihaz ekler; OpenClaw tarafından kullanılan kimlikle tarayın. Kişisel numara/kendi kendine sohbet modu kendisini arayamaz; kişisel numaranızı aramak için özel bir OpenClaw numarası kullanın.
</Warning>

<Steps>
  <Step title="Deneysel aramaları etkinleştirin">

    WhatsApp kanal yapılandırmasına `actions.calls: true` ekleyin ve Gateway'i yeniden başlatın:

```json
{
  "channels": {
    "whatsapp": {
      "actions": {
        "calls": true
      }
    }
  }
}
```

    Olmadığında veya `false` olduğunda OpenClaw, `whatsapp_call` aracını sunmaz.

  </Step>

  <Step title="İncelenen MeowCaller CLI'yi kurun">

    Bağdaştırıcı, Gateway ana makinesinin `PATH` konumunda bir `meowcaller` yürütülebilir dosyası bekler. [MeowCaller PR #7](https://github.com/purpshell/meowcaller/pull/7) birleştirilene kadar incelenen dalı derleyin:

```bash
git clone --branch feat/send-only-notify https://github.com/steipete/meowcaller.git
cd meowcaller
git checkout 752050471fc2bf7a8cdfbf7dbd3cd4e865d85d3f
mkdir -p "$HOME/.local/bin"
go build -o "$HOME/.local/bin/meowcaller" ./cmd/meowcaller
```

    `$HOME/.local/bin` öğesinin Gateway hizmetinin `PATH` konumunda olduğundan emin olun. Bu revizyonda açık `pair` ve yalnızca gönderime yönelik `notify` komutları bulunur; `notify` hiçbir mikrofonu, hoparlörü, video cihazını veya tanılama yakalamasını açmaz. Bunun yerine yukarı akış örnek CLI'sinin `play` komutunu kullanmayın.

  </Step>

  <Step title="MeowCaller bağlı cihazını eşleştirin">

    WhatsApp aracısından arama kurulumunu denetlemesini isteyin (`whatsapp_call` durum eylemi, hesaba özgü durum dizinini ve eşleştirme komutunu bildirir). Varsayılan hesap için:

```bash
state_dir="$HOME/.openclaw/credentials/whatsapp-calls/default"
mkdir -p "$state_dir"
chmod 700 "$state_dir"
meowcaller pair --store "$state_dir/wa-voip.db"
```

    Bunu etkileşimli olarak çalıştırın, **WhatsApp > Linked devices** bölümünden QR'ı tarayın ve `MeowCaller linked device ready` için bekleyin. `wa-voip.db` öğesini gizli tutun; bu, MeowCaller oturumudur. Varsayılan olmayan hesaplar durum eyleminden kendi depo yollarını alır; Windows'ta bunun PowerShell komutunu çalıştırın.

  </Step>

  <Step title="TTS'yi yapılandırın ve WhatsApp'tan arama yapın">

    Telefon kullanımına uygun bir [TTS sağlayıcısı](/tr/tools/tts) yapılandırın, Gateway'i yeniden başlatın ve ardından `Call me and say the build finished.` gibi bir istek gönderin. Araç, göndereni güvenilir gelen bağlamdan çözümler, geçici bir özel WAV dosyası sentezler, MeowCaller'ı sınırlı bir çağrı süresi boyunca çalıştırır ve ardından ses dosyasını siler. OpenClaw, hesabın deposunu açıkça iletir; yanıtlama/oynatma/kapatma sonrasında sıfır çıkış durumu bekler ve zaman aşımını veya sıfır dışı çıkışı başarısız bir araç çağrısı olarak değerlendirir.

  </Step>
</Steps>

Sınırlar: yalnızca bire bir giden sesli aramalar; rastgele hedef numara yok; sohbet bağlantısıyla paylaşılan kimlik doğrulama yok; kişisel numara/kendi kendine sohbet modundan kendi kendini arama yok; sentezlenen ses 60 saniyeyle sınırlıdır; MeowCaller'ın yanıtlama/oynatma/kapatma tamamlanması dışında telefon tarafında duyulabilirlik alındısı yoktur ve OpenClaw, yardımcı işlemi sınırlı bir 115-175 saniyelik süreden sonra durdurur (MeowCaller'ın bağlantı, yanıtlama, oynatma ve kapanma aşamalarını kapsar).

## Onay istemleri

WhatsApp, yürütme ve plugin onay istemlerini üst düzey onay iletme yapılandırmasıyla denetlenen `👍`/`👎` tepkileri olarak işleyebilir:

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session",
    },
    plugin: {
      enabled: true,
      mode: "targets",
      targets: [{ channel: "whatsapp", to: "+15551234567" }],
    },
  },
}
```

`approvals.exec` ve `approvals.plugin` birbirinden bağımsızdır; WhatsApp'ı kanal olarak etkinleştirmek yalnızca aktarımı bağlar ve eşleşen onay ailesi etkinleştirilip oraya yönlendirilmedikçe hiçbir şey göndermez. Oturum modu, yalnızca WhatsApp'tan kaynaklanan onaylar için yerel emoji onayları iletir. Hedef modu, açık hedefler için paylaşılan iletme işlem hattını kullanır ve ayrı onaylayıcı DM dağıtımı oluşturmaz.

WhatsApp onay tepkileri, `allowFrom` (veya `"*"`) içinde açık onaylayıcılar gerektirir. `defaultTo`, onaylayıcı listesi değil, sıradan varsayılan mesaj hedeflerini ayarlar. Manuel `/approve` komutları, onay çözümlenmeden önce yine normal WhatsApp gönderen yetkilendirme yolundan geçer.

## Soru tepkileri

Gizli olmayan, tek seçimli bir soru ve bir ile dört seçenek içeren `ask_user` istemi için WhatsApp, seçenek etiketlerinin yanında `1️⃣` ile `4️⃣` arasındaki değerleri gösterir. Yanıtlamak için iletilen isteme eşleşen numarayla tepki verin. OpenClaw, numarayı Gateway aracılığıyla kurallı seçeneğe eşler; eski veya yinelenen dokunuşlar yok sayılır. Çok sorulu, çok seçimli ve serbest metin istemleri yalnızca metin yanıtlı olarak kalır. Normal WhatsApp DM/grup kabul kuralları, tepki veren göndereni yetkilendirir.

## Plugin kancaları ve gizlilik

Gelen WhatsApp mesajları kişisel içerik, telefon numaraları, grup tanımlayıcıları, gönderen adları ve oturum ilişkilendirme alanları taşıyabilir. WhatsApp, siz etkinleştirmedikçe gelen `message_received` kanca yüklerini pluginlere yayınlamaz:

```json5
{
  channels: {
    whatsapp: {
      pluginHooks: {
        messageReceived: true,
      },
    },
  },
}
```

Etkinleştirme kapsamını `channels.whatsapp.accounts.<id>.pluginHooks.messageReceived` altında tek bir hesapla sınırlayın. Bunu yalnızca gelen WhatsApp içeriği ve tanımlayıcıları konusunda güvendiğiniz pluginler için etkinleştirin.

## Erişim denetimi ve etkinleştirme

<Tabs>
  <Tab title="DM politikası">
    `channels.whatsapp.dmPolicy`:

    | Değer | Davranış |
    | --- | --- |
    | `pairing` (varsayılan) | Bilinmeyen gönderenler eşleştirme ister; sahip onaylar |
    | `allowlist` | Yalnızca `allowFrom` gönderenleri kabul edilir |
    | `open` | `allowFrom` öğesinin `"*"` içermesini gerektirir |
    | `disabled` | Tüm DM'leri engeller |

    `allowFrom`, E.164 biçimli numaraları kabul eder (dahili olarak normalleştirilir). Bu yalnızca DM göndereni için bir erişim denetimi listesidir — grup JID'lerine veya `@newsletter` kanal JID'lerine açıkça yapılan giden gönderimleri kısıtlamaz.

    Çoklu hesap geçersiz kılması: `channels.whatsapp.accounts.<id>.dmPolicy` (ve `.allowFrom`), ilgili hesap için kanal düzeyindeki varsayılanlardan önceliklidir.

    Çalışma zamanı notları:

    - eşleştirmeler kanal izin deposunda kalıcıdır ve yapılandırılmış `allowFrom` ile birleştirilir
    - zamanlanmış otomasyon ve heartbeat alıcısı yedeği, açık teslimat hedeflerini veya yapılandırılmış `allowFrom` değerini kullanır; DM eşleştirme onayları örtük cron/heartbeat alıcıları değildir
    - hiçbir izin listesi yapılandırılmamışsa bağlı öz numaraya varsayılan olarak izin verilir
    - OpenClaw, giden `fromMe` DM'lerini (bağlı cihazdan kendinize gönderdiğiniz mesajları) hiçbir zaman otomatik olarak eşleştirmez

  </Tab>

  <Tab title="Grup politikası ve izin listeleri">
    Grup erişiminin iki katmanı vardır:

    1. **Grup üyeliği izin listesi** (`channels.whatsapp.groups`): `groups` belirtilmezse tüm gruplar uygundur; mevcutsa grup izin listesi görevi görür (`"*"` tümünü kabul eder).
    2. **Grup göndereni politikası** (`channels.whatsapp.groupPolicy` + `groupAllowFrom`): `open` gönderen izin listesini atlar, `allowlist` bir `groupAllowFrom` (veya `*`) eşleşmesi gerektirir, `disabled` gelen tüm grup iletilerini engeller.

    `groupAllowFrom` ayarlanmamışsa ve `allowFrom` girdiler içeriyorsa gönderen denetimleri ona geri döner. Gönderen izin listeleri, bahsetme/yanıt etkinleştirmesinden önce değerlendirilir.

    Hiç `channels.whatsapp` bloğu yoksa çalışma zamanı, `channels.defaults.groupPolicy` başka bir değere ayarlanmış olsa bile `groupPolicy: "allowlist"` değerine geri döner (bir uyarı günlüğüyle).

    <Note>
    Grup üyeliği çözümlemesinde tek hesaplı bir güvenlik ağı bulunur: yalnızca bir WhatsApp hesabı yapılandırılmışsa ve hesabın `accounts.<id>.groups` değeri açıkça boş bir nesneyse (`{}`), bu değer "ayarlanmamış" kabul edilir ve her grubu sessizce engellemek yerine kök `channels.whatsapp.groups` eşlemesine geri dönülür. 2+ hesap yapılandırıldığında açıkça boş olan hesap eşlemesi boş kalır ve geri dönüş yapılmaz — böylece bir hesap, diğer hesapları etkilemeden tüm grupları bilinçli olarak devre dışı bırakabilir.
    </Note>

  </Tab>

  <Tab title="Bahsetmeler ve /activation">
    Grup yanıtları varsayılan olarak bahsetme gerektirir. Bahsetme algılaması şunları içerir:

    - bot kimliğinden açıkça bahseden WhatsApp bahsetmeleri
    - yapılandırılmış bahsetme regex kalıpları (`agents.entries.*.groupChat.mentionPatterns`, geri dönüş `messages.groupChat.mentionPatterns`)
    - yetkili grup mesajlarının gelen sesli not dökümleri
    - örtük bota-yanıt algılaması (yanıtı gönderen bot kimliğiyle eşleşir)

    Güvenlik: alıntı/yanıt yalnızca bahsetme kısıtlamasını karşılar — gönderen yetkisi **vermez**. `groupPolicy: "allowlist"` ile izin listesinde bulunmayan gönderenler, izin listesindeki bir kullanıcının mesajına yanıt verseler bile engellenmeye devam eder.

    Oturum düzeyinde etkinleştirme komutu: `/activation mention` veya `/activation always`. Bu, oturum durumunu günceller (genel yapılandırmayı değil) ve sahip denetimine tabidir.

  </Tab>
</Tabs>

## Yapılandırılmış ACP bağlamaları

WhatsApp, üst düzey `bindings[]` aracılığıyla kalıcı ACP bağlamalarını destekler:

```json5
{
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "whatsapp",
        accountId: "work",
        peer: { kind: "direct", id: "+15555550123" },
      },
    },
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "whatsapp",
        accountId: "work",
        peer: { kind: "group", id: "120363424282127706@g.us" },
      },
    },
  ],
}
```

Doğrudan sohbetler E.164 numaralarıyla, gruplar ise WhatsApp grup JID'leriyle eşleşir. Grup izin listeleri, gönderen politikası ve bahsetme/etkinleştirme kısıtlaması, OpenClaw bağlı ACP oturumunun varlığını sağlamadan önce çalışır. Eşleşen bir bağlama rotanın sahibidir — yayın grupları bu etkileşimi sıradan WhatsApp oturumlarına dağıtmaz.

## Kişisel numara ve kendi kendine sohbet davranışı

Bağlı öz numara `allowFrom` içinde de yer aldığında kendi kendine sohbet güvenlik önlemleri etkinleşir: kendi kendine sohbet etkileşimleri için okundu bilgilerini atlar, kendinize bildirim gönderecek bahsetme JID'si otomatik tetikleme davranışını yok sayar ve kanal/hesap `responsePrefix` değeri ayarlanmamışsa yanıtları varsayılan olarak `[{identity.name}]` (veya `[openclaw]`) değerine yönlendirir.

## Mesaj normalleştirme ve bağlam

<AccordionGroup>
  <Accordion title="Gelen zarf ve yanıt bağlamı">
    Gelen mesajlar paylaşılan gelen zarfla sarmalanır. Alıntılanmış bir yanıt, bağlamı şu biçimde ekler:

    ```text
    [Replying to <sender> id:<stanzaId>]
    <quoted body or media placeholder>
    [/Replying]
    ```

    Yanıt meta verileri (`ReplyToId`, `ReplyToBody`, `ReplyToSender`, gönderen JID'si/E.164) mevcut olduğunda doldurulur. Alıntılanan hedef indirilebilir bir medyaysa OpenClaw bunu normal gelen medya deposu aracılığıyla kaydeder ve `MediaPath`/`MediaType` değerlerini sunar; böylece ajan yalnızca `<media:image>` görmek yerine medyayı doğrudan inceleyebilir.

  </Accordion>

  <Accordion title="Medya yer tutucuları ve konum/kişi çıkarımı">
    Yalnızca medya içeren mesajlar şu yer tutuculara normalleştirilir: `<media:image>`, `<media:video>`, `<media:audio>`, `<media:document>`, `<media:sticker>`.

    Gövde yalnızca `<media:audio>` olduğunda yetkili grup sesli notları bahsetme kısıtlamasından önce yazıya dökülür; böylece sesli notta bottan bahsedilmesi yanıtı tetikleyebilir. Dökümde hâlâ bottan bahsedilmiyorsa ham yer tutucu yerine bekleyen grup geçmişinde kalır.

    Konum gövdeleri kısa koordinat metni olarak işlenir. Konum etiketleri/yorumları ve kişi/vCard ayrıntıları, satır içi istem metni olarak değil, çitle çevrili güvenilmeyen meta veriler olarak işlenir.

  </Accordion>

  <Accordion title="Bekleyen grup geçmişinin eklenmesi">
    İşlenmemiş grup mesajları arabelleğe alınır ve bot nihayet tetiklendiğinde bağlam olarak eklenir.

    - varsayılan sınır: `50`
    - yapılandırma: `channels.whatsapp.historyLimit`, geri dönüş `messages.groupChat.historyLimit`
    - `0` devre dışı bırakır

    Ekleme işaretçileri: `[Chat messages since your last reply - for context]` ve `[Current message - respond to this]`.

  </Accordion>

  <Accordion title="Okundu bilgileri">
    Kabul edilen gelen mesajlar için varsayılan olarak etkindir. Genel olarak devre dışı bırakmak için:

    ```json5
    { channels: { whatsapp: { sendReadReceipts: false } } }
    ```

    Hesap başına geçersiz kılma: `channels.whatsapp.accounts.<id>.sendReadReceipts`. Kendi kendine sohbet etkileşimleri, genel olarak etkin olsa bile okundu bilgilerini atlar.

  </Accordion>
</AccordionGroup>

## Teslimat, parçalara ayırma ve medya

<AccordionGroup>
  <Accordion title="Metni parçalara ayırma">
    - varsayılan parça sınırı: `channels.whatsapp.textChunkLimit = 4000`
    - `channels.whatsapp.streaming.chunkMode = "length" | "newline"`; `newline` paragraf sınırlarını (boş satırları) tercih eder, ardından uzunluk açısından güvenli parçalara ayırmaya geri döner

  </Accordion>

  <Accordion title="Giden medya davranışı">
    - görsel, video, ses (PTT sesli notu) ve belge yüklerini destekler
    - ses, `ptt: true` ile Baileys `audio` yükü olarak gönderilir ve bas-konuş sesli notu şeklinde işlenir; TTS sesli not çıktısının sağlayıcının kaynak biçiminden bağımsız olarak bu yolda kalması için `audioAsVoice` yanıt yüklerinde korunur
    - yerel Ogg/Opus sesi `audio/ogg; codecs=opus` olarak gönderilir; diğer her şey (Microsoft Edge TTS MP3/WebM çıktısı dâhil), PTT teslimatından önce `ffmpeg` ile 48 kHz mono Ogg/Opus biçimine dönüştürülür
    - `/tts latest`, en son asistan yanıtını tek bir sesli not olarak gönderir ve aynı yanıtın yinelenen gönderimlerini engeller; `/tts chat on|off|default` mevcut sohbet için otomatik TTS'yi denetler
    - videolarda `gifPlayback: true` gönderilmesi animasyonlu GIF oynatmayı etkinleştirir
    - `forceDocument`/`asDocument`, WhatsApp'ın medya sıkıştırmasını önlemek için giden görselleri, GIF'leri ve videoları Baileys belge yükü üzerinden yönlendirerek çözümlenen dosya adını ve MIME türünü korur
    - altyazılar, çok medyalı bir yanıttaki ilk medya öğesine uygulanır; PTT sesli notları istisnadır: ses önce altyazısız gönderilir, ardından altyazı ayrı bir metin mesajı olarak gönderilir (WhatsApp istemcileri sesli not altyazılarını tutarlı biçimde işlemez)
    - medya kaynağı HTTP(S), `file://` veya yerel bir yol olabilir

  </Accordion>

  <Accordion title="Medya boyutu sınırları ve geri dönüş davranışı">
    - gelen kaydetme üst sınırı ve giden gönderme üst sınırı: `channels.whatsapp.mediaMaxMb` (varsayılan `50`)
    - hesap başına geçersiz kılma: `channels.whatsapp.accounts.<id>.mediaMaxMb`
    - `forceDocument`/`asDocument` belge teslimatı istemediği sürece görseller sınırlara sığmak için otomatik olarak optimize edilir (yeniden boyutlandırma/kalite taraması)
    - medya gönderimi başarısız olduğunda ilk öğe geri dönüşü, yanıtı sessizce bırakmak yerine bir metin uyarısı gönderir

  </Accordion>
</AccordionGroup>

## Yanıt alıntılama

`channels.whatsapp.replyToMode`, yerel yanıt alıntılamayı denetler (giden yanıtlar gelen mesajı görünür şekilde alıntılar):

| Değer             | Davranış                                                       |
| ----------------- | -------------------------------------------------------------- |
| `"off"` (varsayılan) | Hiçbir zaman alıntılama; düz mesaj olarak gönder                           |
| `"first"`         | Yalnızca ilk giden yanıt parçasını alıntıla                      |
| `"all"`           | Her giden yanıt parçasını alıntıla                               |
| `"batched"`       | Kuyruğa alınmış toplu yanıtları alıntıla; anlık yanıtları alıntısız bırak |

Hesap başına geçersiz kılma: `channels.whatsapp.accounts.<id>.replyToMode`.

```json5
{ channels: { whatsapp: { replyToMode: "first" } } }
```

## Tepki düzeyi

`channels.whatsapp.reactionLevel`, ajanın emoji tepkilerini ne ölçüde kullandığını denetler:

| Düzey                 | Onay tepkileri | Ajan tarafından başlatılan tepkiler  |
| --------------------- | ------------- | -------------------------- |
| `"off"`               | Hayır            | Hayır                         |
| `"ack"`               | Evet           | Hayır                         |
| `"minimal"` (varsayılan) | Evet           | Evet, ölçülü yönlendirme |
| `"extensive"`         | Evet           | Evet, teşvik edilen yönlendirme   |

Hesap başına geçersiz kılma: `channels.whatsapp.accounts.<id>.reactionLevel`.

```json5
{ channels: { whatsapp: { reactionLevel: "ack" } } }
```

## Alındı onayı tepkileri

`channels.whatsapp.ackReaction`, gelen ileti alındığında `reactionLevel` tarafından kısıtlanan anlık bir tepki gönderir (`"off"` olduğunda engellenir):

```json5
{
  channels: {
    whatsapp: {
      ackReaction: {
        emoji: "👀",
        direct: true,
        group: "mentions", // her zaman | bahsetmeler | hiçbir zaman
      },
    },
  },
}
```

Notlar: gelen ileti kabul edildikten hemen sonra (yanıttan önce) gönderilir; `ackReaction`, `emoji` olmadan mevcutsa WhatsApp, yönlendirilen ajanın kimlik emojisini kullanır ve mevcut değilse "👀" değerine geri döner (onay tepkisi olmaması için `ackReaction` değerini atlayın veya `emoji: ""` olarak ayarlayın); hatalar günlüğe kaydedilir ancak yanıt teslimatını engellemez; grup modu `mentions` yalnızca bahsetmeyle tetiklenen etkileşimlerde tepki verirken grup etkinleştirmesi `always` bu denetimi atlar; WhatsApp yalnızca `channels.whatsapp.ackReaction` kullanır (eski `messages.ackReaction` burada geçerli değildir).

## Yaşam döngüsü durumu tepkileri

WhatsApp'ın bir etkileşim sırasında statik bir alındı emojisi bırakmak yerine onay tepkisini değiştirmesini ve kuyrukta, düşünüyor, araç etkinliği, Compaction, tamamlandı ve hata gibi durumlar arasında geçiş yapmasını sağlamak için `messages.statusReactions.enabled: true` değerini ayarlayın:

```json5
{
  messages: {
    statusReactions: {
      enabled: true,
    },
  },
}
```

Notlar: `channels.whatsapp.ackReaction`, doğrudan mesajlar ve gruplar için uygunluğu denetlemeye devam eder; kuyrukta durumu, düz onay tepkileriyle aynı etkin emojiyi kullanır; WhatsApp'ta mesaj başına bir bot tepki yuvası bulunur, bu nedenle yaşam döngüsü güncellemeleri mevcut tepkiyi yerinde değiştirir ve son tamamlandı/hata durumundan sonra onay tepkisini geri yükler.

## Çoklu hesap ve kimlik bilgileri

<AccordionGroup>
  <Accordion title="Hesap seçimi ve varsayılanlar">
    Hesap kimlikleri `channels.whatsapp.accounts` kaynağından gelir. Varsa varsayılan hesap seçimi `default`, aksi takdirde yapılandırılmış ilk hesap kimliğidir (alfabetik olarak sıralanır). Hesap kimlikleri, arama için dahili olarak normalleştirilir.
  </Accordion>

  <Accordion title="Kimlik bilgisi yolları ve eski sürüm uyumluluğu">
    - geçerli kimlik doğrulama yolu: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json` (yedek: `creds.json.bak`)
    - `~/.openclaw/credentials/` içindeki eski varsayılan kimlik doğrulama, varsayılan hesap akışları için hâlâ tanınır/taşınır

  </Accordion>

  <Accordion title="Oturumu kapatma davranışı">
    `openclaw channels logout --channel whatsapp [--account <id>]`, söz konusu hesabın WhatsApp kimlik doğrulama durumunu temizler. Bir gateway erişilebilir olduğunda oturum kapatma işlemi önce o hesabın canlı dinleyicisini durdurur; böylece bağlı oturum, bir sonraki yeniden başlatmadan önce mesaj almayı bırakır. `openclaw channels remove --channel whatsapp` ayrıca hesap yapılandırmasını devre dışı bırakmadan veya silmeden önce canlı dinleyiciyi durdurur.

    Eski kimlik doğrulama dizinlerinde Baileys kimlik doğrulama dosyaları kaldırılırken `oauth.json` korunur.

  </Accordion>
</AccordionGroup>

## Araçlar, eylemler ve yapılandırma yazma işlemleri

- Agent araç desteği, WhatsApp tepki eylemini (`react`) içerir.
- Eylem geçitleri: `channels.whatsapp.actions.reactions`, `channels.whatsapp.actions.polls` (mevcut eylemlerin varsayılanı `true`), `channels.whatsapp.actions.calls` (varsayılan `false`, yukarıdaki MeowCaller bölümüne bakın).
- Kanal tarafından başlatılan yapılandırma yazma işlemleri varsayılan olarak etkindir; `channels.whatsapp.configWrites: false` aracılığıyla devre dışı bırakın.

## Sorun giderme

<AccordionGroup>
  <Accordion title="Bağlı değil (QR gerekli)">
    Belirti: kanal durumu, bağlı olmadığını bildirir.

```bash
openclaw channels login --channel whatsapp
openclaw channels status
```

  </Accordion>

  <Accordion title="Bağlı ancak bağlantı kesilmiş / yeniden bağlanma döngüsü">
    Belirti: bağlı hesapta tekrarlanan bağlantı kesilmeleri veya yeniden bağlanma girişimleri.

    Etkin olmayan hesaplar normal mesaj zaman aşımını aşarak bağlı kalabilir; watchdog yalnızca WhatsApp Web aktarım etkinliği durduğunda, soket kapandığında veya uygulama düzeyindeki etkinlik daha uzun güvenlik aralığının ötesinde sessiz kaldığında yeniden başlatır (yukarıdaki Çalışma zamanı modeli bölümüne bakın).

    Düzeltme:

    ```bash
    openclaw channels status --probe
    openclaw doctor
    openclaw logs --follow
    openclaw gateway status
    ```

    Ana makine bağlantısı ve zamanlama düzeltildikten sonra döngü devam ederse hesap kimlik doğrulama dizinini yedekleyip yeniden bağlayın:

    ```bash
    cp -a ~/.openclaw/credentials/whatsapp/<accountId> \
      ~/.openclaw/credentials/whatsapp/<accountId>.bak
    openclaw channels logout --channel whatsapp --account <accountId>
    openclaw channels login --channel whatsapp --account <accountId>
    ```

    `~/.openclaw/logs/whatsapp-health.log`, `Gateway inactive` diyorsa ancak `openclaw gateway status` ve `openclaw channels status --probe` sağlıklı görünüyorsa `openclaw doctor` komutunu çalıştırın. Linux'ta doctor, kullanımdan kaldırılan `~/.openclaw/bin/ensure-whatsapp.sh` betiğini çağıran eski crontab girdileri hakkında uyarır; bu girdileri `crontab -e` ile kaldırın — cron, systemd kullanıcı veri yolu ortamından yoksun olabilir ve bu eski betiğin gateway durumunu yanlış bildirmesine neden olabilir.

  </Accordion>

  <Accordion title="Proxy arkasında QR ile oturum açma zaman aşımına uğruyor">
    Belirti: `openclaw channels login --channel whatsapp`, kullanılabilir bir QR göstermeden önce `status=408 Request Time-out` veya bir TLS soket bağlantı kesintisiyle başarısız olur.

    WhatsApp Web ile oturum açma, gateway ana makinesinin standart proxy ortamını (`HTTPS_PROXY`, `HTTP_PROXY`, küçük harfli türevleri, `NO_PROXY`) kullanır. Gateway işleminin proxy ortam değişkenlerini devraldığını ve `NO_PROXY` değerinin `mmg.whatsapp.net` ile eşleşmediğini doğrulayın.

  </Accordion>

  <Accordion title="Gönderim sırasında etkin dinleyici yok">
    Hedef hesap için etkin bir gateway dinleyicisi olmadığında giden gönderimler hızla başarısız olur. Gateway'in çalıştığını ve hesabın bağlı olduğunu doğrulayın.
  </Accordion>

  <Accordion title="Yanıt dökümde görünüyor ancak WhatsApp'ta görünmüyor">
    Döküm satırları, agent tarafından oluşturulan içeriği kaydeder; WhatsApp teslimatı ayrı olarak denetlenir. OpenClaw, yalnızca Baileys en az bir görünür metin veya medya gönderimi için bir giden mesaj kimliği döndürdükten sonra otomatik yanıtı gönderilmiş sayar.

    Onay tepkileri, yanıttan bağımsız ön alındı bildirimleridir — başarılı bir tepki, daha sonraki metin/medya yanıtının kabul edildiğini kanıtlamaz. Gateway günlüklerinde `auto-reply delivery failed` veya `auto-reply was not accepted by WhatsApp provider` ifadelerini denetleyin.

  </Accordion>

  <Accordion title="Grup mesajları beklenmedik şekilde yok sayılıyor">
    Şu sırayla denetleyin: `groupPolicy`, `groupAllowFrom`/`allowFrom`, `groups` izin listesi girdileri, bahsetme geçidi (`requireMention` + bahsetme kalıpları) ve `openclaw.json` içindeki yinelenen anahtarlar (JSON5'te sonraki girdiler öncekileri geçersiz kılar — kapsam başına yalnızca bir `groupPolicy` tutun).

    `channels.whatsapp.groups` mevcutsa WhatsApp diğer gruplardan gelen mesajları yine de gözlemleyebilir, ancak OpenClaw bunları oturum yönlendirmesinden önce bırakır. Grup JID'sini `channels.whatsapp.groups` öğesine ekleyin veya gönderen yetkilendirmesini `groupPolicy`/`groupAllowFrom` denetimi altında tutarken tüm grupları kabul etmek için `groups["*"]` ekleyin.

  </Accordion>

  <Accordion title="Bun çalışma zamanı uyarısı">
    OpenClaw gateway'leri Node gerektirir. Bun, standart durum deposunun kullandığı `node:sqlite` API'sini sağlamaz ve doctor eski Bun hizmetlerini Node'a taşır.
  </Accordion>
</AccordionGroup>

## Sistem istemleri

WhatsApp, `groups` ve `direct` eşlemeleri aracılığıyla gruplar ve doğrudan sohbetler için Telegram tarzı sistem istemlerini destekler.

Grup mesajlarının çözümlenmesi: Önce etkin `groups` eşlemesi belirlenir — hesap kendi `groups` anahtarını herhangi bir şekilde tanımlıyorsa kök `groups` eşlemesini tamamen değiştirir (derin birleştirme yapılmaz). Ardından istem araması, ortaya çıkan bu tek eşleme üzerinde çalışır:

1. **Gruba özgü istem** (`groups["<groupId>"].systemPrompt`): grup girdisi mevcut **ve** `systemPrompt` anahtarı tanımlı olduğunda kullanılır. Boş bir dize (`""`) joker karakteri bastırır ve hiçbir istem uygulamaz.
2. **Grup joker karakteri istemi** (`groups["*"].systemPrompt`): belirli grup girdisi mevcut olmadığında veya `systemPrompt` anahtarı olmadan mevcut olduğunda kullanılır.

Doğrudan mesajların çözümlenmesi, `direct` eşlemesi ve `direct["*"]` üzerinde aynı kalıbı izler.

<Note>
`dms`, DM başına hafif geçmiş geçersiz kılma bölümü (`dms.<id>.historyLimit`) olarak kalır. İstem geçersiz kılmaları `direct` altında bulunur.
</Note>

<Note>
İstem çözümlemesindeki bu hesabın kökü değiştirmesi davranışı, basit ve sığ bir geçersiz kılmadır: açıkça belirtilmiş boş bir nesne dâhil olmak üzere herhangi bir hesap `groups`/`direct` anahtarı kök eşlemenin yerini alır. Bu, yanlışlıkla boş bırakılmış bir `groups: {}` için tek hesaplı bir güvenlik ağına sahip olan, yukarıda açıklanan grup üyeliği izin listesi denetiminden farklıdır.
</Note>

**Telegram'dan farkı:** Telegram, bir botun üyesi olmadığı gruplardan grup mesajları almasını önlemek için çok hesaplı bir kurulumdaki her hesapta kök `groups` değerini bastırır (kendi `groups` değeri olmayan hesaplarda bile). WhatsApp bu korumayı uygulamaz — kök `groups`/`direct`, hesap sayısından bağımsız olarak kendi geçersiz kılması bulunmayan her hesap tarafından devralınır. Çok hesaplı bir WhatsApp kurulumunda hesap başına istemler istiyorsanız tam eşlemeyi her hesabın altında açıkça tanımlayın.

Önemli davranış:

- `channels.whatsapp.groups` hem grup başına yapılandırma eşlemesi hem de sohbet düzeyindeki grup izin listesidir. Kök veya hesap kapsamında `groups["*"]`, söz konusu kapsam için "tüm gruplar kabul edilir" anlamına gelir.
- Yalnızca söz konusu kapsamın tüm grupları kabul etmesini zaten istiyorsanız bir `systemPrompt` joker karakteri ekleyin. Yalnızca sabit bir grup kimliği kümesini uygun tutmak için `groups["*"]` kullanmak yerine istemi açıkça izin verilen her girdide tekrarlayın.
- Grup kabulü ve gönderen yetkilendirmesi ayrı denetimlerdir. `groups["*"]`, hangi grupların grup işlemeye ulaşacağını genişletir; bu gruplardaki her gönderene yetki vermez — bu işlem `groupPolicy`/`groupAllowFrom` tarafından denetlenmeye devam eder.
- `channels.whatsapp.direct`, DM'ler için eşdeğer bir yan etkiye sahip değildir: `direct["*"]`, yalnızca bir DM `dmPolicy` ile birlikte `allowFrom` veya eşleştirme deposu kuralları tarafından zaten kabul edildikten sonra varsayılan bir yapılandırma sağlar.

Örnek:

```json5
{
  channels: {
    whatsapp: {
      groups: {
        // Yalnızca tüm grupların kök kapsamında kabul edilmesi gerekiyorsa kullanın.
        // Kendi groups eşlemesini tanımlamayan tüm hesaplara uygulanır.
        "*": { systemPrompt: "Tüm gruplar için varsayılan istem." },
      },
      direct: {
        // Kendi direct eşlemesini tanımlamayan tüm hesaplara uygulanır.
        "*": { systemPrompt: "Tüm doğrudan sohbetler için varsayılan istem." },
      },
      accounts: {
        work: {
          groups: {
            // Bu hesap kendi groups eşlemesini tanımlar, dolayısıyla kök groups eşlemesi tamamen
            // değiştirilir. Joker karakteri korumak için "*" değerini burada da açıkça tanımlayın.
            "120363406415684625@g.us": {
              requireMention: false,
              systemPrompt: "Proje yönetimine odaklan.",
            },
            // Yalnızca bu hesapta tüm grupların kabul edilmesi gerekiyorsa kullanın.
            "*": { systemPrompt: "İş grupları için varsayılan istem." },
          },
          direct: {
            // Bu hesap kendi direct eşlemesini tanımlar, dolayısıyla kök direct girdileri
            // tamamen değiştirilir. Joker karakteri korumak için "*" değerini burada da açıkça tanımlayın.
            "+15551234567": { systemPrompt: "Belirli bir doğrudan iş sohbeti için istem." },
            "*": { systemPrompt: "Doğrudan iş sohbetleri için varsayılan istem." },
          },
        },
      },
    },
  },
}
```

## Yapılandırma başvurusu bağlantıları

Birincil başvuru: [Yapılandırma başvurusu - WhatsApp](/tr/gateway/config-channels#whatsapp)

| Alan             | Alanlar                                                                                                         |
| ---------------- | -------------------------------------------------------------------------------------------------------------- |
| Erişim           | `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`                                             |
| Teslimat         | `textChunkLimit`, `streaming.chunkMode`, `mediaMaxMb`, `sendReadReceipts`, `ackReaction`, `reactionLevel`      |
| Çoklu hesap      | `accounts.<id>.enabled`, `accounts.<id>.authDir` ve diğer hesap başına geçersiz kılmalar                              |
| İşlemler         | `configWrites`, `enabled`                                                                                      |
| Gelen toplu işleme | `messages.inbound.debounceMs`, `messages.inbound.byChannel.whatsapp`                                           |
| Oturum davranışı | `session.dmScope`, `historyLimit`, `dmHistoryLimit`, `dms.<id>.historyLimit`                                   |
| İstemler         | `groups.<id>.systemPrompt`, `groups["*"].systemPrompt`, `direct.<id>.systemPrompt`, `direct["*"].systemPrompt` |

## İlgili

- [Eşleştirme](/tr/channels/pairing)
- [Gruplar](/tr/channels/groups)
- [Güvenlik](/tr/gateway/security)
- [Kanal yönlendirme](/tr/channels/channel-routing)
- [Çoklu agent yönlendirme](/tr/concepts/multi-agent)
- [Sorun giderme](/tr/channels/troubleshooting)
