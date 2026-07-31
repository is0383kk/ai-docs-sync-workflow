---
read_when:
    - Signal desteğini ayarlama
    - Signal gönderme/alma sorunlarını ayıklama
summary: signal-cli (yerel daemon veya bbernhard container) üzerinden Signal desteği, kurulum yolları ve numara modeli
title: Signal
x-i18n:
    generated_at: "2026-07-26T22:35:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 744f817e425d378e9f3e160df534019a6fc865227eb3fc68959a12ad46c0b714
    source_path: channels/signal.md
    workflow: 16
---

Signal, indirilebilir bir kanal pluginidir (`@openclaw/signal`). Gateway, `signal-cli` ile HTTP üzerinden iletişim kurar: yerel daemon (JSON-RPC + SSE) veya [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) konteyneri (REST + WebSocket). OpenClaw, libsignal'ı yerleşik olarak içermez.

## Numara modeli (önce bunu okuyun)

- Gateway bir **Signal cihazına** bağlanır: `signal-cli` hesabı.
- Botu **kişisel Signal hesabınızda** çalıştırmak, kendi mesajlarınızı yok saymasına neden olur (döngü koruması).
- “Bota mesaj gönderdiğimde yanıt versin” kullanım biçimi için **ayrı bir bot numarası** kullanın.

## Kurulum

```bash
openclaw plugins install @openclaw/signal
```

Yalın plugin belirtimleri önce ClawHub'ı, ardından yedek olarak npm'i dener. `openclaw plugins install clawhub:@openclaw/signal` veya `npm:@openclaw/signal` ile bir kaynağı zorunlu kılın. `plugins install`, plugini kaydeder ve etkinleştirir; ayrı bir `enable` adımı gerekmez. Genel kurulum kuralları için [Pluginler](/tr/tools/plugin) bölümüne bakın.

## Hızlı yapılandırma

<Steps>
  <Step title="Bir numara seçin">
    Bot için **ayrı bir Signal numarası** kullanın (önerilir).
  </Step>
  <Step title="Plugini kurun">
    ```bash
    openclaw plugins install @openclaw/signal
    ```
  </Step>
  <Step title="Yönlendirmeli yapılandırmayı çalıştırın">
    ```bash
    openclaw channels add
    ```
    Sihirbaz, `signal-cli` öğesinin `PATH` üzerinde bulunup bulunmadığını algılar ve bulunmadığında yüklemeyi önerir: Linux x86-64'te resmi yerel GraalVM derlemesini indirir; macOS'te ve diğer mimarilerde ise Homebrew aracılığıyla yükler. Ardından bot numarasını ve `signal-cli` yolunu ister.

    Etkileşimsiz yapılandırma için `openclaw channels add --channel signal`, bot telefon numarası amacıyla `--signal-number <e164>` seçeneğini ve Signal daemon uç noktası (varsayılan `127.0.0.1:8080`) için `--http-host <host>` ile `--http-port <port>` seçeneklerini de kabul eder.

  </Step>
  <Step title="Hesabı bağlayın veya kaydedin">
    - **QR ile bağlama (en hızlı):** `signal-cli link -n "OpenClaw"`, ardından Signal ile tarayın. [A Yoluna](#setup-path-a-link-existing-signal-account-qr) bakın.
    - **SMS ile kayıt:** captcha + SMS doğrulaması bulunan özel numara. [B Yoluna](#setup-path-b-register-dedicated-bot-number-sms-linux) bakın.

  </Step>
  <Step title="Doğrulayın ve eşleştirin">
    ```bash
    openclaw gateway call channels.status --params '{"probe":true}'
    ```
    İlk DM'yi gönderin ve eşleştirmeyi onaylayın: `openclaw pairing approve signal <CODE>`.
  </Step>
</Steps>

Asgari yapılandırma:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "managed-native",
        cliPath: "signal-cli",
      },
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

| Alan        | Açıklama                                          |
| ----------- | ------------------------------------------------- |
| `account`   | E.164 biçimindeki bot telefon numarası (`+15551234567`) |
| `transport` | Hesabın sahip olduğu Signal bağlantısı ve işlem modu |
| `dmPolicy`  | DM erişim politikası (`pairing` önerilir)          |
| `allowFrom` | DM göndermesine izin verilen telefon numaraları veya `uuid:<id>` değerleri |

Çoklu hesap desteği: hesap başına yapılandırma ve isteğe bağlı `name` ile `channels.signal.accounts` kullanın. Adlandırılmış her hesap kendi `transport` öğesinin sahibidir; üst düzey taşıma yapılandırmasını devralmaz. Üst düzey taşıma yapılandırması yalnızca örtük `default` hesabına aittir. Ortak kalıp için [Çoklu hesap kanalları](/tr/gateway/config-channels#multi-account-all-channels) bölümüne bakın.

## Nedir?

- Belirlenimci yönlendirme: yanıtlar her zaman Signal'a geri gönderilir.
- DM'ler aracının ana oturumunu paylaşır; gruplar yalıtılmıştır (`agent:<agentId>:signal:group:<groupId>`).
- Signal varsayılan olarak `/config set|unset` tarafından tetiklenen yapılandırma güncellemelerini yazabilir (`commands.config: true` gerektirir). `channels.signal.configWrites: false` ile devre dışı bırakın.

## Yapılandırma yolu A: mevcut Signal hesabını bağlama (QR)

1. `signal-cli` öğesini (JVM veya yerel derleme) yükleyin ya da `openclaw channels add` öğesinin sizin için yüklemesine izin verin.
2. Bir bot hesabını bağlayın: `signal-cli link -n "OpenClaw"`, ardından Signal'da QR kodunu tarayın.
3. Signal'ı yapılandırın ve Gateway'i başlatın.

## Yapılandırma yolu B: özel bot numarasını kaydetme (SMS, Linux)

Mevcut bir Signal uygulaması hesabını bağlamak yerine özel bir bot numarası için bunu kullanın. Aşağıdaki akış Ubuntu 24 üzerinde test edilmiştir.

1. SMS (veya sabit hatlar için sesli doğrulama) alabilen bir numara edinin. Özel bir bot numarası, hesap/oturum çakışmalarını önler.
2. Gateway ana makinesine `signal-cli` yükleyin:

```bash
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}-Linux-native.tar.gz"
sudo tar xf "signal-cli-${VERSION}-Linux-native.tar.gz" -C /opt
sudo ln -sf /opt/signal-cli /usr/local/bin/
signal-cli --version
```

JVM derlemesini (`signal-cli-${VERSION}.tar.gz`) kullanıyorsanız önce bir JRE yükleyin. `signal-cli` öğesini güncel tutun; üst kaynak, Signal sunucu API'leri değiştikçe eski sürümlerin çalışmayı durdurabileceğini belirtir.

3. Numarayı kaydedin ve doğrulayın:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register
```

Captcha gerekiyorsa (bu adımı tamamlamak için tarayıcı erişimi gerekir):

1. `https://signalcaptchas.org/registration/generate.html` öğesini açın.
2. Captcha'yı tamamlayın ve “Open Signal” öğesindeki `signalcaptcha://...` bağlantı hedefini kopyalayın.
3. Mümkün olduğunda komutu tarayıcı oturumuyla aynı harici IP'den çalıştırın (captcha belirteçlerinin süresi hızla dolar).
4. Hemen kaydedin ve doğrulayın:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register --captcha '<SIGNALCAPTCHA_URL>'
signal-cli -a +<BOT_PHONE_NUMBER> verify <VERIFICATION_CODE>
```

4. OpenClaw'ı yapılandırın, Gateway'i yeniden başlatın ve kanalı doğrulayın:

```bash
# Gateway'i bir kullanıcı systemd hizmeti olarak çalıştırıyorsanız:
systemctl --user restart openclaw-gateway.service

# Ardından doğrulayın:
openclaw doctor
openclaw channels status --probe
```

5. DM göndericinizi eşleştirin:
   - Bot numarasına herhangi bir mesaj gönderin.
   - Sunucuda onaylayın: `openclaw pairing approve signal <PAIRING_CODE>`.
   - “Unknown contact” ifadesini önlemek için bot numarasını telefonunuza kişi olarak kaydedin.

<Warning>
Bir telefon numarası hesabını `signal-cli` ile kaydetmek, bu numaraya ait ana Signal uygulaması oturumunun kimlik doğrulamasını kaldırabilir. Özel bir bot numarasını tercih edin veya mevcut telefon uygulaması yapılandırmanızı korumak için QR ile bağlama modunu kullanın.
</Warning>

Üst kaynak başvuruları:

- `signal-cli` README: `https://github.com/AsamK/signal-cli`
- Captcha akışı: `https://github.com/AsamK/signal-cli/wiki/Registration-with-captcha`
- Bağlama akışı: `https://github.com/AsamK/signal-cli/wiki/Linking-other-devices-(Provisioning)`

## Harici yerel daemon modu

`signal-cli` öğesini kendiniz yönetmek için (yavaş JVM soğuk başlatmaları, konteyner başlatma işlemi, paylaşımlı CPU'lar) daemon'ı ayrı olarak çalıştırın ve OpenClaw'ı ona yönlendirin:

Etkileşimsiz yapılandırmada gerektiğinde uç nokta türünü açıkça seçin:

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://127.0.0.1:8080 --signal-transport external-native
```

```json5
{
  channels: {
    signal: {
      transport: {
        kind: "external-native",
        url: "http://127.0.0.1:8080",
      },
    },
  },
}
```

Bu, otomatik başlatmayı ve OpenClaw'ın başlangıç beklemesini atlar. Yavaş başlayan yönetilen bir daemon için `channels.signal.transport.startupTimeoutMs` ayarlayın.

## Konteyner modu (bbernhard/signal-cli-rest-api)

`signal-cli` öğesini yerel olarak çalıştırmak yerine, `signal-cli` öğesini bir REST + WebSocket arayüzünün arkasında sarmalayan [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) Docker konteynerini kullanın.

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://signal-cli:8080 --signal-transport container
```

Gereksinimler:

- Gerçek zamanlı mesaj alımı için konteyner **mutlaka** `MODE=json-rpc` ile çalışmalıdır.
- OpenClaw'ı bağlamadan önce Signal hesabınızı konteynerin içinde kaydedin veya bağlayın.

Örnek `docker-compose.yml` hizmeti:

```yaml
signal-cli:
  image: bbernhard/signal-cli-rest-api:latest
  environment:
    MODE: json-rpc
  ports:
    - "8080:8080"
  volumes:
    - signal-cli-data:/home/.local/share/signal-cli
```

OpenClaw yapılandırması:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "container",
        url: "http://signal-cli:8080",
      },
    },
  },
}
```

`transport.kind`, OpenClaw'ın hangi protokolü ve işlem yaşam döngüsünü kullandığını denetler:

| Değer               | Davranış                                                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `"managed-native"`  | Yerel signal-cli'ı başlatır ve `/api/v1/rpc` üzerinde JSON-RPC ile `/api/v1/events` üzerinde SSE kullanır; `url`, daemon bağlama adresinden farklı bir bağlantı uç noktası seçebilir |
| `"external-native"` | Hâlihazırda çalışan yerel signal-cli daemon'ına bağlanır                                                                                                       |
| `"container"`       | `/v2/send` üzerindeki bbernhard REST'e ve `/v1/receive/{account}` üzerindeki WebSocket'e bağlanır                                                                             |

Yapılandırma ve `openclaw doctor --fix`, somut türünü belirlemek için mevcut bir uç noktayı bir kez yoklayabilir. Çalışma zamanı işlemleri protokolleri otomatik olarak algılamaz veya değiştirmez.

Konteyner modu, konteynerin eşleşen API'leri sunduğu durumlarda yerel modla aynı Signal işlemlerini destekler: gönderme, alma, ekler, yazma göstergeleri, okundu/görüldü alındıları, tepkiler, gruplar ve biçemlendirilmiş metin. OpenClaw, `group.{base64(internal_id)}` grup kimlikleri ve biçimlendirilmiş metin için `text_mode: "styled"` dâhil olmak üzere yerel Signal RPC çağrılarını konteynerin REST yüklerine dönüştürür.

İşletim notları:

- Alım için `MODE=json-rpc` kullanın. `MODE=normal`, `/v1/about` öğesinin sağlıklı görünmesini sağlayabilir; ancak `/v1/receive/{account}` WebSocket yükseltmesi yapmaz, bu nedenle konteyner alım akışının yoklaması başarısız olur.
- bbernhard REST API için `kind: "container"`, yerel `signal-cli` JSON-RPC/SSE içinse `kind: "external-native"` ayarlayın.
- Konteyner eki indirmeleri, yerel modla aynı medya bayt sınırlarına uyar. Sunucu `Content-Length` gönderdiğinde aşırı büyük yanıtlar tamamen arabelleğe alınmadan önce, aksi durumda ise akış sırasında reddedilir.

## Erişim denetimi (DM'ler + gruplar)

DM'ler:

- Varsayılan: `channels.signal.dmPolicy = "pairing"`.
- Bilinmeyen göndericiler bir eşleştirme kodu alır; onaylanana kadar mesajlar yok sayılır (kodların süresi 1 saat sonra dolar).
- `openclaw pairing list signal` ve `openclaw pairing approve signal <CODE>` aracılığıyla onaylayın.
- Eşleştirme, Signal DM'leri için varsayılan belirteç değişimidir. Ayrıntılar: [Eşleştirme](/tr/channels/pairing)
- Yalnızca UUID içeren göndericiler (`sourceUuid` kaynağından), `channels.signal.allowFrom` içinde `uuid:<id>` olarak depolanır.

Gruplar:

- `channels.signal.groupPolicy = open | allowlist | disabled`.
- `channels.signal.groupAllowFrom`, `allowlist` ayarlandığında hangi grupların veya gönderenlerin grup yanıtlarını tetikleyebileceğini denetler; girdiler Signal grup kimlikleri (ham, `group:<id>` veya `signal:group:<id>`), gönderen telefon numaraları, `uuid:<id>` değerleri veya `*` olabilir.
- `channels.signal.groups["<group-id>" | "*"]`, `requireMention`, `tools` ve `toolsBySender` ile grup davranışını geçersiz kılabilir.
- Çok hesaplı kurulumlarda hesap başına geçersiz kılmalar için `channels.signal.accounts.<id>.groups` kullanın.
- Bir Signal grubunu `groupAllowFrom` aracılığıyla izin verilenler listesine eklemek, bahsetme denetimini tek başına devre dışı bırakmaz. Özel olarak yapılandırılmış bir `channels.signal.groups["<group-id>"]` girdisi, `requireMention=true` ayarlanmadığı sürece her grup mesajını işler.
- `requireMention=true` ile Signal'ın yerel @bahsetmeleri, yapılandırılmış bahsetme meta verilerinden bot hesabının telefonu veya `accountUuid` ile eşleştirilir. Yapılandırılmış `mentionPatterns`, düz metin geri dönüşü olarak kalır.
- Çalışma zamanı notu: `channels.signal` tamamen eksikse çalışma zamanı, grup denetimleri için (`channels.defaults.groupPolicy` ayarlanmış olsa bile) `groupPolicy="allowlist"` değerine geri döner.

Sınırlı bağlama sahip, bahsetme denetimli grup:

```json5
{
  channels: {
    signal: {
      account: "+15551234567",
      accountUuid: "bot-signal-uuid",
      groupPolicy: "allowlist",
      groupAllowFrom: ["group:<signal-group-id>"],
      historyLimit: 8,
      groups: {
        "<signal-group-id>": { requireMention: true },
      },
    },
  },
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b"],
    },
  },
}
```

Bottan bahsetmeyen izinli grup mesajları sessiz kalır ve yalnızca sınırlı bekleyen geçmiş penceresinde tutulur. Daha sonraki bir yerel @bahsetme veya geri dönüş metni bahsetmesi botu tetiklediğinde OpenClaw bu yakın tarihli bağlamı ekler ve aynı gruba yanıt verir. Atlanan eklerin gövdeleri indirilmez; bekleyen bağlamda yalnızca kısa medya yer tutucuları olarak görünebilirler.

## Nasıl çalışır (davranış)

- Yerel mod: `signal-cli` bir arka plan hizmeti olarak çalışır; Gateway olayları SSE üzerinden okur.
- Konteyner modu: Gateway REST API aracılığıyla gönderir ve WebSocket aracılığıyla alır.
- Gelen mesajlar, paylaşılan kanal zarfına normalleştirilir.
- Yanıtlar her zaman aynı numaraya veya gruba geri yönlendirilir.
- Arka uç gelen zaman damgasını ve yazarı kabul ettiğinde, gelen mesajlara verilen yanıtlar yerel Signal alıntı meta verilerini içerir; alıntı meta verileri eksikse veya reddedilirse OpenClaw yanıtı normal bir mesaj olarak gönderir.
- Yerel alıntı kullanımını `channels.signal.replyToMode = off | first | all | batched` ile veya sohbet türü başına geçersiz kılmalar için `channels.signal.replyToModeByChatType.direct/group` ile yapılandırın. `channels.signal.accounts.<id>` altındaki hesap düzeyi değerler önceliklidir.

## Medya + sınırlar

- Giden metin `channels.signal.textChunkLimit` değerine göre parçalara ayrılır (varsayılan 4000).
- İsteğe bağlı yeni satıra göre parçalama: uzunluğa göre parçalamadan önce boş satırlarda (paragraf sınırlarında) bölmek için `channels.signal.streaming.chunkMode="newline"` ayarını yapın.
- Ekler desteklenir (`signal-cli` üzerinden base64 olarak alınır).
- Sesli not ekleri, `contentType` eksik olduğunda MIME geri dönüşü olarak `signal-cli` dosya adını kullanır; böylece ses dökümü AAC sesli notlarını sınıflandırmaya devam edebilir.
- Varsayılan medya sınırı: `channels.signal.mediaMaxMb` (varsayılan 8).
- Herhangi bir aktarım için medya indirmeyi atlamak üzere `channels.signal.ignoreAttachments` kullanın.
- Grup geçmişi bağlamı, `messages.groupChat.historyLimit` değerine geri dönerek `channels.signal.historyLimit` (veya `channels.signal.accounts.*.historyLimit`) kullanır. Devre dışı bırakmak için `0` ayarını yapın (varsayılan 50).

## Yazıyor göstergeleri + okundu bilgileri

- **Yazıyor göstergeleri**: OpenClaw, `signal-cli sendTyping` aracılığıyla yazıyor sinyalleri gönderir ve bir yanıt çalışırken bunları yeniler.
- **Okundu bilgileri**: `channels.signal.sendReadReceipts` true olduğunda OpenClaw, izin verilen doğrudan mesajlar için okundu bilgilerini iletir.
- `signal-cli`, gruplar için okundu bilgilerini sunmaz.

## Yaşam döngüsü durum tepkileri

Signal'ın gelen turlarda paylaşılan sırada/bekliyor/araç/compaction/tamamlandı/hata tepki yaşam döngüsünü göstermesi için `messages.statusReactions.enabled: true` ayarını yapın. Signal, tepki hedefi olarak gelen mesajın zaman damgasını kullanır; grup tepkileri, Signal grup kimliği ve hedef yazar olarak asıl gönderen ile gönderilir.

Durum tepkileri ayrıca bir onay tepkisi ve eşleşen bir `messages.ackReactionScope` (`direct`, `group-all`, `group-mentions` veya `all`) gerektirir. Signal durum tepkilerini devre dışı bırakmak için `channels.signal.reactionLevel: "off"` ayarını yapın.

Signal, nihai tamamlandı/hata durumundan sonra ilk onay tepkisini geri yükler.

## Tepkiler (mesaj aracı)

`message action=react` öğesini `channel=signal` ile kullanın.

- Hedefler: gönderenin E.164 numarası veya UUID'si (eşleştirme çıktısındaki `uuid:<id>` değerini kullanın; yalın UUID de çalışır).
- `messageId`, tepki verdiğiniz mesajın Signal zaman damgasıdır.
- Grup tepkileri `targetAuthor` veya `targetAuthorUuid` gerektirir.

```text
message action=react channel=signal target=uuid:123e4567-e89b-12d3-a456-426614174000 messageId=1737630212345 emoji=🔥
message action=react channel=signal target=+15551234567 messageId=1737630212345 emoji=🔥 remove=true
message action=react channel=signal target=signal:group:<groupId> targetAuthor=uuid:<sender-uuid> messageId=1737630212345 emoji=✅
```

Yapılandırma:

- `channels.signal.actions.reactions`: tepki eylemlerini etkinleştirir/devre dışı bırakır (varsayılan true).
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive` (varsayılan `minimal`).
  - `off`/`ack`, aracı tepkilerini devre dışı bırakır (mesaj aracı `react` hata verir).
  - `minimal`/`extensive`, aracı tepkilerini etkinleştirir ve yönlendirme düzeyini ayarlar.
- Hesap başına geçersiz kılmalar: `channels.signal.accounts.<id>.actions.reactions`, `channels.signal.accounts.<id>.reactionLevel`.

## Onay tepkileri

Signal yürütme ve plugin onay istemleri, üst düzey `approvals.exec` ve `approvals.plugin` yönlendirme bloklarını kullanır. Signal'ın `channels.signal.execApprovals` bloğu yoktur.

- `👍`, bir kez onaylar.
- `👎`, reddeder.
- Bir istek kalıcı onay sunduğunda `/approve <id> allow-always` kullanın.

Onay tepkisinin çözümlenmesi, `channels.signal.allowFrom`, `channels.signal.defaultTo` veya eşleşen hesap düzeyi alanlarından açık Signal onaylayıcıları gerektirir. Aynı sohbetteki doğrudan yürütme onayı istemleri, açık onaylayıcılar olmadan da yinelenen yerel `/approve` geri dönüşünü gizleyebilir; onaylayıcısı olmayan grup onaylarında yerel geri dönüş görünür kalır.

## Soru tepkileri

Gizli olmayan, tek seçimli bir soru ve bir ila dört seçenek içeren `ask_user` isteminde Signal, seçenek etiketlerinin yanında `1️⃣` ile `4️⃣` arasındaki değerleri gösterir. Yanıtlamak için teslim edilen isteme eşleşen numarayla tepki verin. OpenClaw, tepkinin bot tarafından yazılmış mesajı hedeflediğini doğrular ve ardından numarayı Gateway aracılığıyla standart seçeneğe eşler. Eski veya yinelenen dokunuşlar yok sayılır. Çok sorulu, çok seçimli ve serbest metin istemleri yalnızca metin yanıtıyla kullanılmaya devam eder; normal Signal doğrudan mesaj/grup kabul kuralları göndereni yetkilendirir.

## Teslimat hedefleri (CLI/cron)

- Doğrudan mesajlar: `signal:+15551234567` (veya yalın E.164).
- UUID doğrudan mesajları: `uuid:<id>` (veya yalın UUID).
- Gruplar: `signal:group:<groupId>`.
- Kullanıcı adları: `username:<name>` (Signal hesabınız destekliyorsa).

## Takma adlar

Yinelenen Signal hedeflerinde kararlı adlar kullanmak için takma adları yapılandırın. Takma adlar yalnızca OpenClaw tarafındaki yapılandırmadır; Signal kişileri oluşturmaz veya düzenlemez.

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
        jane: "uuid:123e4567-e89b-12d3-a456-426614174000",
        ops: "group:<groupId>",
      },
      defaultTo: "signal:me",
    },
  },
}
```

Signal teslimat hedeflerinin kabul edildiği her yerde takma adları kullanın:

```bash
openclaw message send --channel signal --target signal:ops --message "Dağıtım tamamlandı"
```

Hesap başına takma adlar, üst düzey takma adları devralır ve ad ekleyebilir veya mevcut adları geçersiz kılabilir:

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
      },
      accounts: {
        work: {
          aliases: {
            ops: "group:<workGroupId>",
          },
        },
      },
    },
  },
}
```

`openclaw directory peers list --channel signal` ve `openclaw directory groups list --channel signal`, yapılandırılmış takma adları listeler. Signal dizini yapılandırma desteklidir; Signal kişilerini canlı olarak sorgulamaz veya Signal hesabını değiştirmez.

## Sorun giderme

Önce şu adımları çalıştırın:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Ardından gerekirse doğrudan mesaj eşleştirme durumunu doğrulayın:

```bash
openclaw pairing list signal
```

Yaygın hatalar:

- Arka plan hizmetine erişilebiliyor ancak yanıt yok: `account`, `transport.kind`, aktarım URL'si ve alma modunu doğrulayın.
- Doğrudan mesajlar yok sayılıyor: gönderen eşleştirme onayı bekliyor.
- Grup mesajları yok sayılıyor: grup göndereni/bahsetme denetimi teslimatı engelliyor.
- Düzenlemelerden sonra yapılandırma doğrulama hataları: `openclaw doctor --fix` komutunu çalıştırın.
- Tanılamada Signal eksik: `channels.signal.enabled: true` değerini doğrulayın.

Ek denetimler:

```bash
openclaw pairing list signal
pgrep -af signal-cli
openclaw logs --plain --limit 500 | grep -i "signal" | tail -20
```

Sorun sınıflandırma akışı için: [Kanal Sorunlarını Giderme](/tr/channels/troubleshooting).

## Güvenlik notları

- `signal-cli`, hesap anahtarlarını yerel olarak saklar (genellikle `~/.local/share/signal-cli/data/`).
- Sunucu taşıma veya yeniden oluşturma işleminden önce Signal hesap durumunu yedekleyin.
- Açıkça daha geniş doğrudan mesaj erişimi istemediğiniz sürece `channels.signal.dmPolicy: "pairing"` değerini koruyun.
- SMS doğrulaması yalnızca kayıt veya kurtarma akışları için gereklidir, ancak numaranın/hesabın denetimini kaybetmek yeniden kaydı karmaşıklaştırabilir.

## Yapılandırma referansı (Signal)

Tam yapılandırma: [Yapılandırma](/tr/gateway/configuration)

Sağlayıcı seçenekleri:

- `channels.signal.enabled`: kanal başlangıcını etkinleştirir/devre dışı bırakır.
- `channels.signal.account`: bot hesabı için E.164.
- `channels.signal.accountUuid`: yerel @bahsetme algılama ve döngü koruması için isteğe bağlı bot hesabı UUID'si.
- `channels.signal.transport`: hesaba ait taşıma katmanı. Yönetilen yerel varsayılanlar için bunu atlayın.
- `channels.signal.transport.kind`: `managed-native | external-native | container`.
- `channels.signal.transport.url`: `external-native` ve `container` için gereklidir; bağlantı uç noktası daemon bağlama adresinden farklı olduğunda `managed-native` için isteğe bağlıdır.
- `channels.signal.transport.cliPath`: `signal-cli` için yönetilen yerel yol.
- `channels.signal.transport.configPath`: isteğe bağlı yönetilen yerel `signal-cli --config` dizini.
- `channels.signal.transport.httpHost`, `channels.signal.transport.httpPort`: yönetilen yerel daemon bağlama adresi (varsayılan `127.0.0.1:8080`).
- `channels.signal.transport.startupTimeoutMs`: ms cinsinden yönetilen yerel başlatma bekleme süresi (min 1000, üst sınır 120000; varsayılan 30000).
- `channels.signal.transport.receiveMode`: yönetilen yerel `on-start | manual`.
- `channels.signal.ignoreAttachments`: bu hesap için gelen eklerin indirilmesini atlar.
- `channels.signal.transport.ignoreStories`: yönetilen yerel hikâye açma/kapatma seçeneği.
- `channels.signal.sendReadReceipts`: okundu bilgilerini iletir.
- `channels.signal.dmPolicy`: `pairing | allowlist | open | disabled` (varsayılan: eşleştirme).
- `channels.signal.allowFrom`: DM izin listesi (E.164 veya `uuid:<id>`). `open`, `"*"` gerektirir. Signal'de kullanıcı adları yoktur; telefon/UUID kimliklerini kullanın.
- `channels.signal.aliases`: DM veya grup teslim hedefleri için OpenClaw tarafındaki takma adlar.
- `channels.signal.groupPolicy`: `open | allowlist | disabled` (varsayılan: izin listesi).
- `channels.signal.groupAllowFrom`: grup izin listesi; Signal grup kimliklerini (ham, `group:<id>` veya `signal:group:<id>`), gönderenin E.164 numaralarını ya da `uuid:<id>` değerlerini kabul eder.
- `channels.signal.groups`: Signal grup kimliğine (veya `"*"`) göre anahtarlanmış grup başına geçersiz kılmalar. Desteklenen alanlar: `requireMention`, `tools`, `toolsBySender`.
- `channels.signal.accounts.<id>.groups`: çok hesaplı kurulumlar için `channels.signal.groups` seçeneğinin hesap başına sürümü.
- `channels.signal.accounts.<id>.aliases`: üst düzey takma adlarla birleştirilen hesap başına takma adlar.
- `channels.signal.replyToMode`: yerel yanıt alıntısı modu, `off | first | all | batched` (varsayılan: `all`).
- `channels.signal.replyToModeByChatType.direct`, `channels.signal.replyToModeByChatType.group`: sohbet türü başına yerel yanıt alıntısı geçersiz kılmaları.
- `channels.signal.accounts.<id>.replyToMode`, `channels.signal.accounts.<id>.replyToModeByChatType.direct`, `channels.signal.accounts.<id>.replyToModeByChatType.group`: hesap başına yanıt alıntısı geçersiz kılmaları.
- `channels.signal.historyLimit`: bağlam olarak eklenecek azami grup mesajı sayısı (0 devre dışı bırakır).
- `channels.signal.dmHistoryLimit`: kullanıcı turları cinsinden DM geçmişi sınırı. Kullanıcı başına geçersiz kılmalar: `channels.signal.dms["<phone_or_uuid>"].historyLimit`.
- `channels.signal.textChunkLimit`: karakter cinsinden giden parça boyutu (varsayılan 4000).
- `channels.signal.streaming.chunkMode`: uzunluğa göre parçalara ayırmadan önce boş satırlarda (paragraf sınırlarında) bölmek için `length` (varsayılan) veya `newline`.
- `channels.signal.mediaMaxMb`: MB cinsinden gelen/giden medya üst sınırı (varsayılan 8).
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive` (varsayılan `minimal`). Bkz. [Tepkiler](#reactions-message-tool).
- `channels.signal.reactionNotifications`: `off | own | all | allowlist` (varsayılan `own`) - temsilcinin başkalarından gelen tepkiler konusunda ne zaman bilgilendirileceği.
- `channels.signal.reactionAllowlist`: `reactionNotifications: "allowlist"` olduğunda tepkileri temsilciyi bilgilendiren göndericiler.
- `channels.signal.streaming.block.enabled`, `channels.signal.streaming.block.coalesce`: kanallar arasında paylaşılan blok modu akış denetimleri. Bkz. [Akış](/tr/concepts/streaming).

İlgili genel seçenekler:

- `agents.entries.*.groupChat.mentionPatterns` (düz metin geri dönüşü; bot hesabı kimliği yapılandırıldığında Signal'in yerel @bahsetmeleri yapılandırılmış meta verilerden algılanır).
- `messages.groupChat.mentionPatterns` (genel geri dönüş).
- `channels.signal.responsePrefix` veya hesap düzeyinde bir `responsePrefix`.

## İlgili

- [Kanallara Genel Bakış](/tr/channels) - desteklenen tüm kanallar
- [Eşleştirme](/tr/channels/pairing) - DM kimlik doğrulama ve eşleştirme akışı
- [Gruplar](/tr/channels/groups) - grup sohbeti davranışı ve bahsetme geçidi
- [Kanal Yönlendirme](/tr/channels/channel-routing) - mesajlar için oturum yönlendirmesi
- [Güvenlik](/tr/gateway/security) - erişim modeli ve sağlamlaştırma
