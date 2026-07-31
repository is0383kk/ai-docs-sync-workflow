---
read_when:
    - Mattermost'u ayarlama
    - Mattermost yönlendirmesinde hata ayıklama
sidebarTitle: Mattermost
summary: Mattermost bot kurulumu ve OpenClaw yapılandırması
title: Mattermost
x-i18n:
    generated_at: "2026-07-26T23:49:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea41fb9a7e4e9ea6bd8d04a4f2c6d2d7f2e43cf71830e445f1e28e2e8737f3cb
    source_path: channels/mattermost.md
    workflow: 16
---

Durum: indirilebilir plugin (bot tokeni + WebSocket olayları). Kanallar, özel kanallar, grup DM'leri ve DM'ler desteklenir. Mattermost, kendi sunucunuzda barındırılabilen bir ekip mesajlaşma platformudur ([mattermost.com](https://mattermost.com)).

## Kurulum

<Tabs>
  <Tab title="npm kayıt defteri">
    ```bash
    openclaw plugins install @openclaw/mattermost
    ```
  </Tab>
  <Tab title="Yerel çalışma kopyası">
    ```bash
    openclaw plugins install ./path/to/local/mattermost-plugin
    ```
  </Tab>
</Tabs>

Ayrıntılar: [Plugin'ler](/tr/tools/plugin)

## Hızlı kurulum

<Steps>
  <Step title="Plugin'in kullanılabilir olduğundan emin olun">
    Yukarıdaki komutla `@openclaw/mattermost` öğesini kurun, ardından Gateway zaten çalışıyorsa yeniden başlatın.
  </Step>
  <Step title="Bir Mattermost botu oluşturun">
    Bir Mattermost bot hesabı oluşturun, **bot tokenini** kopyalayın ve botu okuması gereken ekiplere ve kanallara ekleyin.
  </Step>
  <Step title="Temel URL'yi kopyalayın">
    Mattermost **temel URL'sini** (ör. `https://chat.example.com`) kopyalayın. Sondaki `/api/v4` otomatik olarak kaldırılır.
  </Step>
  <Step title="OpenClaw'u yapılandırın ve gateway'i başlatın">
    Asgari yapılandırma:

    ```json5
    {
      channels: {
        mattermost: {
          enabled: true,
          botToken: "mm-token",
          baseUrl: "https://chat.example.com",
          dmPolicy: "pairing",
        },
      },
    }
    ```

    Etkileşimsiz alternatif:

    ```bash
    openclaw channels add --channel mattermost --bot-token <token> --http-url https://chat.example.com
    ```

  </Step>
</Steps>

<Note>
Özel/LAN/tailnet adresindeki, kendi sunucunuzda barındırılan Mattermost: giden Mattermost API istekleri, özel ve dahili IP'leri varsayılan olarak engelleyen bir SSRF korumasından geçer. `channels.mattermost.network.dangerouslyAllowPrivateNetwork: true` ile açıkça etkinleştirin (hesap başına: `channels.mattermost.accounts.<id>.network.dangerouslyAllowPrivateNetwork`).
</Note>

## Yerel eğik çizgi komutları

Yerel eğik çizgi komutları isteğe bağlıdır. Etkinleştirildiğinde OpenClaw, botun üyesi olduğu her ekipte `oc_*` eğik çizgi komutlarını kaydeder ve gateway HTTP sunucusunda geri çağırma POST isteklerini alır.

```json5
{
  channels: {
    mattermost: {
      commands: {
        native: true,
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // Mattermost gateway'e doğrudan erişemediğinde kullanın (ters proxy/genel URL).
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
    },
  },
}
```

Kaydedilen komutlar: `/oc_status`, `/oc_model`, `/oc_models`, `/oc_new`, `/oc_help`, `/oc_think`, `/oc_reasoning`, `/oc_verbose`, `/oc_queue`. `nativeSkills: true` ile beceri komutları da `/oc_<skill>` olarak kaydedilir.

<AccordionGroup>
  <Accordion title="Davranış notları">
    - `native` ve `nativeSkills` varsayılan olarak `"auto"` değerini kullanır; bu değer Mattermost için devre dışı olarak çözümlenir. Bunları açıkça `true` olarak ayarlayın.
    - `callbackPath` varsayılan olarak `/api/channels/mattermost/command` değerini kullanır.
    - `callbackUrl` belirtilmezse OpenClaw, `http://<gateway.customBindHost or localhost>:<gateway.port, default 18789><callbackPath>` değerini türetir. Joker karakterli bağlama ana makineleri (`0.0.0.0`, `::`) `localhost` değerine geri döner.
    - Çok hesaplı kurulumlarda `commands`, üst düzeyde veya `channels.mattermost.accounts.<id>.commands` altında ayarlanabilir (hesap değerleri üst düzey alanları geçersiz kılar).
    - Diğer entegrasyonlar tarafından aynı tetikleyiciyle oluşturulan mevcut eğik çizgi komutlarına dokunulmaz (kayıt işlemi bunları atlar); botun oluşturduğu komutlar, geri çağırma URL'si değiştiğinde güncellenir veya yeniden oluşturulur.
    - Komut geri çağırmaları, OpenClaw `oc_*` komutlarını kaydettiğinde Mattermost tarafından döndürülen komut başına tokenlerle doğrulanır.
    - OpenClaw, her geri çağırmayı kabul etmeden önce mevcut Mattermost komut kaydını yeniler; böylece silinen veya yeniden oluşturulan eğik çizgi komutlarının eski tokenleri, gateway yeniden başlatılmadan kabul edilmemeye başlar.
    - Mattermost API komutun hâlâ güncel olduğunu doğrulayamazsa geri çağırma doğrulaması kapalı kalacak şekilde başarısız olur; başarısız doğrulamalar kısa süreliğine önbelleğe alınır, eşzamanlı aramalar birleştirilir ve tekrar oynatma baskısını sınırlamak için yeni arama başlangıçları komut başına hız sınırına tabi tutulur.
    - Kayıt başarısız olduğunda, başlangıç kısmi kaldığında veya geri çağırma tokeni çözümlenen komutun kayıtlı tokeniyle eşleşmediğinde eğik çizgi geri çağırmaları kapalı kalacak şekilde başarısız olur (bir komut için geçerli olan token, farklı bir komutun üst akış doğrulamasına ulaşamaz).
    - Kabul edilen geri çağırmalar, geçici bir "İşleniyor..." yanıtıyla onaylanır; gerçek yanıt normal bir mesaj olarak gelir.

  </Accordion>
  <Accordion title="Erişilebilirlik gereksinimi">
    Geri çağırma uç noktasına Mattermost sunucusundan erişilebilmelidir.

    - Mattermost, OpenClaw ile aynı ana makine/ağ ad alanında çalışmadığı sürece `callbackUrl` değerini `localhost` olarak ayarlamayın.
    - Bu URL, `/api/channels/mattermost/command` yolunu OpenClaw'a ters proxy ile yönlendirmediği sürece `callbackUrl` değerini Mattermost temel URL'nize ayarlamayın.
    - Hızlı bir kontrol için `curl https://<gateway-host>/api/channels/mattermost/command` kullanılabilir; bir GET isteği, `404` değil, OpenClaw'dan `405 Method Not Allowed` döndürmelidir.

  </Accordion>
  <Accordion title="Mattermost giden trafik izin listesi">
    Geri çağırmanız özel/tailnet/dahili adresleri hedefliyorsa Mattermost `ServiceSettings.AllowedUntrustedInternalConnections` ayarını geri çağırma ana makinesini/alan adını içerecek şekilde yapılandırın.

    Tam URL'ler yerine ana makine/alan adı girdileri kullanın.

    - Doğru: `gateway.tailnet-name.ts.net`
    - Yanlış: `https://gateway.tailnet-name.ts.net`

  </Accordion>
</AccordionGroup>

## Ortam değişkenleri (varsayılan hesap)

Ortam değişkenlerini tercih ediyorsanız bunları gateway ana makinesinde ayarlayın:

- `MATTERMOST_BOT_TOKEN=...`
- `MATTERMOST_URL=https://chat.example.com`

<Note>
Ortam değişkenleri yalnızca **varsayılan** hesaba (`default`) uygulanır. Diğer hesaplar yapılandırma değerlerini kullanmalıdır.

`MATTERMOST_URL`, bir çalışma alanı `.env` dosyasından ayarlanamaz; bkz. [Çalışma alanı .env dosyaları](/tr/gateway/security).
</Note>

## Sohbet modları

Mattermost doğrudan mesajlara otomatik olarak yanıt verir. Kanal davranışı `chatmode` tarafından denetlenir:

<Tabs>
  <Tab title="oncall (varsayılan)">
    Kanallarda yalnızca @bahsedildiğinde yanıt ver.
  </Tab>
  <Tab title="onmessage">
    Her kanal mesajına yanıt ver.
  </Tab>
  <Tab title="onchar">
    Bir mesaj tetikleyici önekle başladığında yanıt ver.
  </Tab>
</Tabs>

Yapılandırma örneği:

```json5
{
  channels: {
    mattermost: {
      chatmode: "onchar",
      oncharPrefixes: [">", "!"], // varsayılan
    },
  },
}
```

Notlar:

- `onchar` açık @bahsetmelerine yanıt vermeye devam eder.
- `channels.mattermost.requireMention` hâlâ dikkate alınır, ancak `chatmode` tercih edilir. Kanal başına `groups.<channelId>.requireMention` ayarları her ikisinden de önceliklidir.
- Bot bir kanal ileti dizisinde görünür bir yanıt gönderdikten sonra, aynı ileti dizisindeki sonraki mesajlar yeni bir @bahsetme veya `onchar` öneki olmadan yanıtlanır; böylece çok turlu ileti dizisi konuşmaları kesintisiz sürer. Katılım, botun söz konusu ileti dizisindeki son yanıtından itibaren 7 gün boyunca hatırlanır ve Gateway yeniden başlatmalarında korunur. Botun yalnızca gözlemlediği ileti dizileri bundan etkilenmez; yeniden açık bir bahsetme gerektirmek için yeni bir üst düzey mesaj başlatın.
- Katılım sağlanan ileti dizilerindeki takip mesajlarının bahsetme denetimini atlamasını önlemek için `channels.mattermost.implicitMentions.threadParticipation: false` ayarını belirleyin. Hesap geçersiz kılmaları `channels.mattermost.accounts.<id>.implicitMentions` kullanır. Mattermost şu anda `replyToBot` veya `quotedBot` bilgilerini üretmediğinden bu bayrakların burada hiçbir etkisi yoktur.

## İleti dizileri ve oturumlar

Kanal ve grup yanıtlarının ana kanalda kalacağını mı yoksa tetikleyici gönderinin altında bir ileti dizisi mi başlatacağını denetlemek için `channels.mattermost.replyToMode` kullanın.

- `off` (varsayılan): yalnızca gelen gönderi zaten bir ileti dizisindeyse ileti dizisinde yanıt verilir.
- `first`: üst düzey kanal/grup gönderileri için gönderinin altında bir ileti dizisi başlatılır ve konuşma ileti dizisi kapsamındaki bir oturuma yönlendirilir.
- `all` ve `batched`: Mattermost'ta bugün `first` ile aynı davranışı gösterir; çünkü Mattermost'ta bir ileti dizisi kökü oluşturulduktan sonra takip parçaları ve medya aynı ileti dizisinde devam eder.
- Doğrudan mesajlar, `replyToMode` ayarlanmış olsa bile varsayılan olarak `off` kullanır.

`direct`, `group` veya `channel` sohbetlerinin modunu geçersiz kılmak için `channels.mattermost.replyToModeByChatType` kullanın. Doğrudan mesajlarda ileti dizilerini etkinleştirmek için `direct` ayarını belirleyin:

- `off` (varsayılan): doğrudan mesajlar, sürekli ilerleyen tek bir oturumda ileti dizisi olmadan kalır.
- `first`, `all` veya `batched`: her üst düzey doğrudan mesaj, yeni ve bağımsız bir oturum tarafından desteklenen bir Mattermost ileti dizisi başlatır.

```json5
{
  channels: {
    mattermost: {
      replyToMode: "all",
      replyToModeByChatType: {
        direct: "first",
      },
    },
  },
}
```

Notlar:

- İleti dizisi kapsamındaki oturumlar, tetikleyici gönderi kimliğini ileti dizisi kökü olarak kullanır.
- `first` ve `all` şu anda eşdeğerdir; çünkü Mattermost'ta bir ileti dizisi kökü oluşturulduktan sonra takip parçaları ve medya aynı ileti dizisinde devam eder.
- Sohbet türü başına geçersiz kılmalar `replyToMode` ayarından önceliklidir. Bir `direct` geçersiz kılması olmadan mevcut dağıtımlar düz, ileti dizisiz doğrudan mesajları korur.

## Erişim denetimi (doğrudan mesajlar)

- Varsayılan: `channels.mattermost.dmPolicy = "pairing"` (bilinmeyen gönderenlere bir eşleştirme kodu verilir). Diğer değerler: `allowlist`, `open`, `disabled`.
- Şunlar aracılığıyla onaylayın:
  - `openclaw pairing list mattermost`
  - `openclaw pairing approve mattermost <CODE>`
- Herkese açık doğrudan mesajlar: `channels.mattermost.dmPolicy="open"` ile birlikte `channels.mattermost.allowFrom=["*"]` (yapılandırma şeması joker karakteri zorunlu kılar).
- `channels.mattermost.allowFrom`, kullanıcı kimliklerini (önerilir) ve `accessGroup:<name>` girdilerini kabul eder. Bkz. [Erişim grupları](/tr/channels/access-groups).

## Kanallar (gruplar)

- Varsayılan: `channels.mattermost.groupPolicy = "allowlist"` (bahsetme gerektirir).
- Gönderenleri `channels.mattermost.groupAllowFrom` ile izin verilenler listesine ekleyin (kullanıcı kimlikleri önerilir).
- `channels.mattermost.groupAllowFrom`, `accessGroup:<name>` girdilerini kabul eder. Bkz. [Erişim grupları](/tr/channels/access-groups).
- Kanal başına bahsetme geçersiz kılmaları `channels.mattermost.groups.<channelId>.requireMention` altında, varsayılan ayar ise `channels.mattermost.groups["*"].requireMention` altında bulunur.
- `@username` eşleştirmesi değişkendir ve yalnızca `channels.mattermost.dangerouslyAllowNameMatching: true` olduğunda etkinleştirilir.
- Açık kanallar: `channels.mattermost.groupPolicy="open"` (bahsetme gerektirir).
- Çözümleme sırası: `channels.mattermost.groupPolicy`, ardından `channels.defaults.groupPolicy`, ardından `"allowlist"`.
- Çalışma zamanı notu: `channels.mattermost` bölümü tamamen eksikse çalışma zamanı, grup denetimleri için (`channels.defaults.groupPolicy` ayarlanmış olsa bile) güvenli biçimde `groupPolicy="allowlist"` durumuna geçer ve bir defalık uyarı kaydeder.

Örnek:

```json5
{
  channels: {
    mattermost: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
    },
  },
}
```

## Giden teslimat hedefleri

`openclaw message send` veya cron/webhook'larla şu hedef biçimlerini kullanın:

| Hedef                               | Teslim edildiği yer                                           |
| ----------------------------------- | ------------------------------------------------------------- |
| `channel:<id>`                      | Kimliğe göre kanal                                            |
| `channel:<name>` veya `#channel-name` | Ada göre kanal; botun üyesi olduğu takımlarda aranır          |
| `user:<id>` veya `mattermost:<id>`    | İlgili kullanıcıyla doğrudan mesaj                            |
| `@username`                         | Doğrudan mesaj (kullanıcı adı Mattermost API üzerinden çözümlenir) |

Giden gönderimler mesaj başına en fazla bir eki destekler; birden fazla dosyayı ayrı gönderimlere bölün.

<Warning>
Yalın belirsiz kimlikler (`64ifufp...` gibi) Mattermost'ta **muğlaktır** (kullanıcı kimliği veya kanal kimliği).

OpenClaw bunları **önce kullanıcı** olarak çözümler:

- Kimlik bir kullanıcı olarak mevcutsa (`GET /api/v4/users/<id>` başarılı olursa), OpenClaw doğrudan kanalı `/api/v4/channels/direct` aracılığıyla çözümleyerek bir **DM** gönderir.
- Aksi takdirde kimlik bir **kanal kimliği** olarak değerlendirilir.

Belirleyici davranış gerekiyorsa her zaman açık önekleri (`user:<id>` / `channel:<id>`) kullanın.
</Warning>

## DM kanalı yeniden denemesi

OpenClaw bir Mattermost DM hedefine gönderim yaparken önce doğrudan kanalı çözümlemesi gerektiğinde, geçici doğrudan kanal oluşturma hatalarını varsayılan olarak yeniden dener.

Bu davranışı Mattermost plugin'i genelinde ayarlamak için `channels.mattermost.dmChannelRetry`, tek bir hesap için ise `channels.mattermost.accounts.<id>.dmChannelRetry` kullanın. Varsayılanlar:

```json5
{
  channels: {
    mattermost: {
      dmChannelRetry: {
        maxRetries: 3,
        initialDelayMs: 1000,
        maxDelayMs: 10000,
        timeoutMs: 30000,
      },
    },
  },
}
```

Notlar:

- Bu, her Mattermost API çağrısına değil, yalnızca DM kanalı oluşturmaya (`/api/v4/channels/direct`) uygulanır.
- Yeniden denemelerde değişken gecikmeli üstel geri çekilme kullanılır ve hız sınırları, 5xx yanıtları ile ağ veya zaman aşımı hataları gibi geçici hatalara uygulanır.
- `429` dışındaki 4xx istemci hataları kalıcı kabul edilir ve yeniden denenmez.

## Önizleme akışı

Mattermost; düşünme sürecini, araç etkinliğini ve kısmi yanıt metnini, nihai yanıtın gönderilmesi güvenli olduğunda yerinde son hâline getirilen bir **taslak önizleme gönderisine** aktarır. `partial` modunda önizleme, kanalı her parça için ayrı mesajlarla doldurmak yerine aynı gönderi kimliği üzerinde güncellenir. `block` modunda önizleme, tamamlanmış metin ile araç etkinliği blokları arasında dönüşümlü ilerler; böylece önceki bloklar bir sonraki blok tarafından üzerlerine yazılmak yerine kendi gönderileri olarak görünür kalır. Medya/hata nihai sonuçları, bekleyen önizleme düzenlemelerini iptal eder ve geçici bir önizleme gönderisini tamamlamak yerine normal teslimatı kullanır.

Önizleme akışı, `partial` modunda **varsayılan olarak açıktır**. `channels.mattermost.streaming.mode` üzerinden yapılandırın (eski skaler/boolean `streaming` değerleri `openclaw doctor --fix` tarafından taşınır):

```json5
{
  channels: {
    mattermost: {
      streaming: { mode: "partial" }, // off | partial | block | progress
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Akış modları">
    - `partial` (varsayılan): yanıt büyüdükçe düzenlenen ve ardından eksiksiz yanıtla son hâline getirilen tek bir önizleme gönderisi.
    - `block`, önizlemeyi tamamlanmış metin ile araç etkinliği blokları arasında dönüşümlü ilerletir; böylece her blok yerinde üzerine yazılmak yerine kendi gönderisi olarak görünür kalır. Paralel ve ardışık araç güncellemeleri mevcut araç etkinliği gönderisini paylaşır.
    - `progress`, oluşturma sırasında bir durum önizlemesi gösterir ve nihai yanıtı yalnızca tamamlandığında gönderir.
    - `off`, önizleme akışını devre dışı bırakır. `streaming.block.enabled: true` ile tamamlanmış asistan blokları, birleştirilmiş tek bir nihai gönderi yerine normal blok yanıtları (ayrı gönderiler) olarak teslim edilmeye devam eder.

  </Accordion>
  <Accordion title="Akış davranışı notları">
    - Akış yerinde son hâline getirilemezse (örneğin gönderi akış sırasında silinirse), yanıtın hiçbir zaman kaybolmaması için OpenClaw yeni bir nihai gönderi göndermeye geri döner.
    - Yalnızca düşünme içeren yüklerin kanal gönderilerinde gösterilmesi engellenir; buna `> Thinking` blok alıntısı olarak gelen metinler de dahildir. Düşünme sürecini diğer yüzeylerde görmek için `/reasoning on` ayarını kullanın; Mattermost nihai gönderisi yalnızca yanıtı içerir.
    - Kanal eşleme matrisi için [Akış](/tr/concepts/streaming#preview-streaming-modes) bölümüne bakın.

  </Accordion>
</AccordionGroup>

## Tepkiler (mesaj aracı)

- `message action=react` öğesini `channel=mattermost` ile kullanın.
- `messageId`, Mattermost gönderi kimliğidir.
- `emoji`, `thumbsup` veya `:+1:` gibi adları kabul eder (iki nokta üst üste isteğe bağlıdır).
- Bir tepkiyi kaldırmak için `remove=true` (boolean) ayarını kullanın.
- Tepki ekleme/kaldırma olayları, mesajlarla aynı DM/grup politikası kontrollerine tabi olarak yönlendirilmiş agent oturumuna sistem olayları şeklinde iletilir.

Örnekler:

```text
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup remove=true
```

Yapılandırma:

- `channels.mattermost.actions.reactions`: tepki eylemlerini etkinleştirin/devre dışı bırakın (varsayılan true).
- Hesap başına geçersiz kılma: `channels.mattermost.accounts.<id>.actions.reactions`.

## Etkileşimli düğmeler (mesaj aracı)

Tıklanabilir düğmeler içeren mesajlar gönderin. Bir kullanıcı bir düğmeye tıkladığında agent seçimi alır ve yanıt verebilir.

Düğmeler, anlamsal `presentation` yükünden gelir (normal agent yanıtlarında ve `message action=send` içinde). OpenClaw, değer düğmelerini Mattermost etkileşimli düğmeleri olarak işler, URL düğmelerini mesaj metninde görünür tutar ve seçim menülerini okunabilir metne indirger.

```text
message action=send channel=mattermost target=channel:<channelId> presentation={"blocks":[{"type":"buttons","buttons":[{"label":"Yes","value":"yes"},{"label":"No","value":"no"}]}]}
```

Sunum düğmesi alanları:

<ParamField path="label" type="string" required>
  Görünen etiket (diğer ad: `text`).
</ParamField>
<ParamField path="value" type="string">
  Tıklamada geri gönderilen ve eylem kimliği olarak kullanılan değer (diğer adlar: `callback_data`, `callbackData`). `url` ayarlanmadıkça tıklanabilir bir düğme için gereklidir.
</ParamField>
<ParamField path="url" type="string">
  Bağlantı düğmesi; etkileşimli bir düğme yerine mesaj gövdesinde `label: url` metni olarak işlenir.
</ParamField>
<ParamField path="style" type='"primary" | "secondary" | "success" | "danger"'>
  Düğme stili. Mattermost, desteklemediği değerlere varsayılan stili uygular.
</ParamField>

Agent sistem isteminde düğme desteğini bildirmek için kanal yeteneklerine `inlineButtons` ekleyin:

```json5
{
  channels: {
    mattermost: {
      capabilities: ["inlineButtons"],
    },
  },
}
```

Bir kullanıcı bir düğmeye tıkladığında:

<Steps>
  <Step title="Erişim kontrolü">
    Tıklayan kişi, mesaj gönderenle aynı DM/grup politikası kontrollerinden geçmelidir; yetkisiz tıklamalar geçici bir bildirim alır ve yok sayılır.
  </Step>
  <Step title="Düğmelerin onayla değiştirilmesi">
    Tüm düğmeler bir onay satırıyla değiştirilir (ör. "✓ **Yes** selected by @user").
  </Step>
  <Step title="Agent'ın seçimi alması">
    Agent, seçimi gelen bir mesaj (ve ayrıca bir sistem olayı) olarak alır ve yanıt verir.
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Uygulama notları">
    - Düğme geri çağrıları HMAC-SHA256 doğrulaması kullanır (otomatik; yapılandırma gerekmez).
    - Tıklamada ek bloğunun tamamı değiştirilir; bu nedenle tüm düğmeler birlikte kaldırılır ve kısmi kaldırma mümkün değildir.
    - Kısa çizgi veya alt çizgi içeren eylem kimlikleri otomatik olarak arındırılır (Mattermost yönlendirme sınırlaması).
    - `action_id` değeri özgün gönderideki bir eylemle eşleşmeyen tıklamalar, `403` ("Bilinmeyen eylem") ile reddedilir.

  </Accordion>
  <Accordion title="Yapılandırma ve erişilebilirlik">
    - `channels.mattermost.capabilities`: yetenek dizeleri dizisi. Agent sistem isteminde düğmeler aracı açıklamasını etkinleştirmek için `"inlineButtons"` ekleyin.
    - `channels.mattermost.interactions.callbackBaseUrl`: düğme geri çağrıları için isteğe bağlı harici temel URL (örneğin `https://gateway.example.com`). Mattermost, bağlama ana makinesindeki gateway'e doğrudan erişemediğinde bunu kullanın.
    - Çok hesaplı kurulumlarda aynı alanı `channels.mattermost.accounts.<id>.interactions.callbackBaseUrl` altında da ayarlayabilirsiniz.
    - `interactions.callbackBaseUrl` atlanırsa OpenClaw, geri çağrı URL'sini `gateway.customBindHost` + `gateway.port` (varsayılan 18789) üzerinden türetir, ardından `http://localhost:<port>` seçeneğine geri döner. Geri çağrı yolu `/mattermost/interactions/<accountId>` şeklindedir.
    - Erişilebilirlik kuralı: Düğme geri çağrı URL'sine Mattermost sunucusundan erişilebilmelidir. `localhost`, yalnızca Mattermost ve OpenClaw aynı ana makinede/ağ ad alanında çalıştığında işe yarar.
    - `channels.mattermost.interactions.allowedSourceIps`: düğme geri çağrıları için kaynak IP izin listesi. Bu ayar olmadan yalnızca geri döngü kaynakları (`127.0.0.1`, `::1`) kabul edilir; bu nedenle uzak bir Mattermost sunucusu burada izin listesine eklenmelidir, aksi takdirde tıklamaları `403` ile reddedilir. Ters proxy arkasında, gerçek istemci IP'sinin iletilen üst bilgilerden türetilmesi için ayrıca `gateway.trustedProxies` ayarını kullanın.
    - Geri çağrı hedefiniz özel/tailnet/dahili ise ana makinesini/alan adını Mattermost `ServiceSettings.AllowedUntrustedInternalConnections` öğesine ekleyin.

  </Accordion>
</AccordionGroup>

### Doğrudan API entegrasyonu (harici betikler)

Harici betikler ve webhook'lar, agent'ın `message` aracından geçmek yerine Mattermost REST API üzerinden doğrudan düğmeler gönderebilir. OpenClaw'ın `message` aracını tercih edin. Doğrudan entegrasyonlar için `buildButtonAttachments` öğesini `@openclaw/mattermost/api.js` içinden içe aktarın; ham JSON gönderiyorsanız şu kurallara uyun:

**Yük yapısı:**

```json5
{
  channel_id: "<channelId>",
  message: "Bir seçenek belirleyin:",
  props: {
    attachments: [
      {
        actions: [
          {
            id: "mybutton01", // yalnızca alfasayısal - aşağıya bakın
            type: "button", // gereklidir, aksi takdirde tıklamalar sessizce yok sayılır
            name: "Onayla", // görünen etiket
            style: "primary", // isteğe bağlı: "default", "primary", "danger"
            integration: {
              url: "https://gateway.example.com/mattermost/interactions/default",
              context: {
                action_id: "mybutton01", // düğme kimliğiyle eşleşmelidir
                action: "approve",
                // ... özel alanlar ...
                _token: "<hmac>", // aşağıdaki HMAC bölümüne bakın
              },
            },
          },
        ],
      },
    ],
  },
}
```

<Warning>
**Kritik kurallar**

1. Ekler, üst düzey `attachments` içine değil, `props.attachments` içine yerleştirilir (aksi takdirde sessizce yok sayılır).
2. Her eylem için `type: "button"` gereklidir; bu olmadan tıklamalar sessizce yutulur.
3. Her eylem için bir `id` alanı gereklidir; Mattermost, kimliği olmayan eylemleri yok sayar.
4. Eylem `id` değeri **yalnızca alfasayısal** olmalıdır (`[a-zA-Z0-9]`). Kısa çizgiler ve alt çizgiler Mattermost'un sunucu tarafı eylem yönlendirmesini bozar (404 döndürür). Kullanmadan önce bunları kaldırın.
5. `context.action_id`, düğmenin `id` değeriyle eşleşmelidir; gateway, `action_id` değeri gönderide bulunmayan tıklamaları reddeder.
6. `context.action_id` gereklidir; etkileşim işleyicisi bu olmadan 400 döndürür.
7. Geri çağrı kaynak IP'sine izin verilmelidir (yukarıdaki `interactions.allowedSourceIps` bölümüne bakın).

</Warning>

**HMAC belirteci oluşturma**

Gateway, düğme tıklamalarını HMAC-SHA256 ile doğrular. Harici betikler, gateway'in doğrulama mantığıyla eşleşen belirteçler oluşturmalıdır:

<Steps>
  <Step title="Gizli anahtarı bot belirtecinden türetme">
    `HMAC-SHA256(key="openclaw-mattermost-interactions", data=botToken)`, onaltılık olarak kodlanmış.
  </Step>
  <Step title="Bağlam nesnesini oluşturma">
    Bağlam nesnesini `_token` **hariç** tüm alanlarla oluşturun.
  </Step>
  <Step title="Sıralanmış anahtarlarla serileştirme">
    **Özyinelemeli olarak sıralanmış anahtarlarla** ve **boşluksuz** serileştirin (gateway, iç içe nesneleri de standartlaştırır ve kompakt JSON üretir).
  </Step>
  <Step title="Yükü imzalama">
    `HMAC-SHA256(key=secret, data=serializedContext)`
  </Step>
  <Step title="Belirteci ekleme">
    Ortaya çıkan onaltılık özeti bağlama `_token` olarak ekleyin.
  </Step>
</Steps>

Python örneği:

```python
import hmac, hashlib, json

secret = hmac.new(
    b"openclaw-mattermost-interactions",
    bot_token.encode(), hashlib.sha256
).hexdigest()

ctx = {"action_id": "mybutton01", "action": "approve"}
payload = json.dumps(ctx, sort_keys=True, separators=(",", ":"))
token = hmac.new(secret.encode(), payload.encode(), hashlib.sha256).hexdigest()

context = {**ctx, "_token": token}
```

<AccordionGroup>
  <Accordion title="Yaygın HMAC sorunları">
    - Python'ın `json.dumps` işlevi varsayılan olarak boşluk ekler (`{"key": "val"}`). JavaScript'in kompakt çıktısıyla (`{"key":"val"}`) eşleşmek için `separators=(",", ":")` kullanın.
    - Her zaman **tüm** bağlam alanlarını (`_token` hariç) imzalayın. Gateway, `_token` alanını kaldırır ve ardından kalan her şeyi imzalar. Yalnızca bir alt kümeyi imzalamak, herhangi bir belirti vermeden doğrulamanın başarısız olmasına neden olur.
    - `sort_keys=True` kullanın; Gateway imzalamadan önce anahtarları sıralar ve Mattermost, yükü depolarken bağlam alanlarını yeniden sıralayabilir.
    - Gizli anahtarı rastgele baytlardan değil, bot token'ından türetin (belirlenimsel olarak). Gizli anahtar, düğmeleri oluşturan işlem ile doğrulamayı yapan Gateway genelinde aynı olmalıdır.

  </Accordion>
</AccordionGroup>

## Dizin bağdaştırıcısı

Mattermost Plugin'i, Mattermost API'si aracılığıyla kanal ve kullanıcı adlarını çözümleyen bir dizin bağdaştırıcısı içerir. Bu, `openclaw message send` içindeki `#channel-name` ve `@username` hedeflerinin yanı sıra cron/webhook teslimatlarını etkinleştirir.

Herhangi bir yapılandırma gerekmez; bağdaştırıcı, hesap yapılandırmasındaki bot token'ını kullanır.

## Çoklu hesap

Mattermost, `channels.mattermost.accounts` altında birden fazla hesabı destekler:

```json5
{
  channels: {
    mattermost: {
      accounts: {
        default: { name: "Primary", botToken: "mm-token", baseUrl: "https://chat.example.com" },
        alerts: { name: "Alerts", botToken: "mm-token-2", baseUrl: "https://alerts.example.com" },
      },
    },
  },
}
```

Hesap değerleri üst düzey alanları geçersiz kılar; herhangi bir hesap belirtilmediğinde hangi hesabın kullanılacağını `channels.mattermost.defaultAccount` seçer.

## Sorun giderme

<AccordionGroup>
  <Accordion title="Kanallarda yanıt yok">
    Botun kanalda olduğundan emin olun ve bottan bahsedin (oncall), bir tetikleyici öneki kullanın (onchar) veya `chatmode: "onmessage"` ayarını yapın.
  </Accordion>
  <Accordion title="Kimlik doğrulama veya çoklu hesap hataları">
    - Bot token'ını, temel URL'yi ve hesabın etkin olup olmadığını kontrol edin.
    - Çoklu hesap sorunları: ortam değişkenleri yalnızca `default` hesabına uygulanır.
    - Özel/LAN Mattermost sunucuları `network.dangerouslyAllowPrivateNetwork: true` gerektirir (SSRF koruması varsayılan olarak özel IP'leri engeller).

  </Accordion>
  <Accordion title="Yerel eğik çizgi komutları başarısız oluyor">
    - `Unauthorized: invalid command token.`: OpenClaw geri çağırma token'ını kabul etmedi. Yaygın nedenler:
      - eğik çizgi komutu kaydı başlangıçta başarısız oldu veya yalnızca kısmen tamamlandı
      - geri çağırma yanlış Gateway'e/hesaba ulaşıyor
      - Mattermost'ta hâlâ önceki bir geri çağırma hedefini gösteren eski komutlar bulunuyor
      - Gateway, eğik çizgi komutlarını yeniden etkinleştirmeden yeniden başlatıldı
    - Yerel eğik çizgi komutları çalışmayı durdurursa günlüklerde `mattermost: failed to register slash commands` veya `mattermost: native slash commands enabled but no commands could be registered` olup olmadığını kontrol edin.
    - `callbackUrl` belirtilmemişse ve günlükler geri çağırmanın `http://localhost:18789/...` gibi bir geri döngü URL'sine çözümlendiği konusunda uyarıyorsa bu URL'ye büyük olasılıkla yalnızca Mattermost, OpenClaw ile aynı ana bilgisayarda/ağ ad alanında çalıştığında erişilebilir. Bunun yerine dışarıdan erişilebilen açık bir `commands.callbackUrl` ayarlayın.

  </Accordion>
  <Accordion title="Düğme sorunları">
    - Düğmeler beyaz kutular olarak görünüyor veya hiç görünmüyor: düğme verileri hatalı biçimlendirilmiştir. Her sunum düğmesi bir `label` ve bir `value` gerektirir (bunlardan herhangi biri eksik olan düğmeler kaldırılır).
    - Düğmeler görüntüleniyor ancak tıklamalar hiçbir şey yapmıyor: Gateway'e Mattermost sunucusundan erişilebildiğini, Mattermost sunucusunun IP'sinin `channels.mattermost.interactions.allowedSourceIps` içinde bulunduğunu (bu ayar olmadan yalnızca geri döngü kabul edilir) ve özel hedefler için `ServiceSettings.AllowedUntrustedInternalConnections` değerinin geri çağırma ana bilgisayarını içerdiğini doğrulayın.
    - Düğmeler tıklandığında 404 döndürüyor: düğmenin `id` değeri muhtemelen kısa çizgi veya alt çizgi içeriyor. Mattermost'un eylem yönlendiricisi alfasayısal olmayan kimliklerde bozulur. Yalnızca `[a-zA-Z0-9]` kullanın.
    - Gateway günlüklerinde `rejected callback source` görülüyor: tıklama, `interactions.allowedSourceIps` dışındaki bir IP'den geldi. Mattermost sunucusunu veya giriş noktanızı izin verilenler listesine ekleyin ve ters proxy arkasında `gateway.trustedProxies` ayarını yapın.
    - Gateway günlüklerinde `invalid _token` görülüyor: HMAC uyuşmazlığı. Tüm bağlam alanlarını (yalnızca bir alt kümeyi değil) imzaladığınızı, sıralanmış anahtarlar ve kompakt JSON (boşluksuz) kullandığınızı kontrol edin. Yukarıdaki HMAC bölümüne bakın.
    - Gateway günlüklerinde `missing _token in context` görülüyor: `_token` alanı düğmenin bağlamında bulunmuyor. Entegrasyon yükü oluşturulurken bu alanın eklendiğinden emin olun.
    - Gateway, tıklamayı `Unknown action` ile reddediyor: `context.action_id`, gönderideki hiçbir eylem `id` değeriyle eşleşmiyor. Her ikisini de aynı arındırılmış değere ayarlayın.
    - Agent düğmeler sunmuyor: Mattermost kanal yapılandırmasına `capabilities: ["inlineButtons"]` ekleyin.

  </Accordion>
</AccordionGroup>

## İlgili konular

- [Kanal Yönlendirme](/tr/channels/channel-routing) - iletiler için oturum yönlendirme
- [Kanallara Genel Bakış](/tr/channels) - desteklenen tüm kanallar
- [Gruplar](/tr/channels/groups) - grup sohbeti davranışı ve bahsetme denetimi
- [Eşleştirme](/tr/channels/pairing) - DM kimlik doğrulaması ve eşleştirme akışı
- [Güvenlik](/tr/gateway/security) - erişim modeli ve sağlamlaştırma
