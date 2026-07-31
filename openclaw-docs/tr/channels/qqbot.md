---
read_when:
    - OpenClaw'u QQ'ya bağlamak istiyorsunuz
    - QQ Bot kimlik bilgilerini ayarlamanız gerekiyor
    - QQ Bot grup veya özel sohbet desteği istiyorsunuz
summary: QQ Bot kurulumu, yapılandırması ve kullanımı
title: QQ botu
x-i18n:
    generated_at: "2026-07-26T23:30:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b185a2b1182471bbec3688b40fb72b671bdf3a2e8351aa6e2f7918f4f5936825
    source_path: channels/qqbot.md
    workflow: 16
---

QQ Bot, resmi QQ Bot API'si (WebSocket gateway) aracılığıyla OpenClaw'a bağlanır.
C2C özel sohbet ve grup `@`-bahsetmeleri, zengin
medya (görüntüler, ses, video, dosyalar) desteğiyle başlıca sohbet türleridir. Guild kanal mesajlarında
yalnızca metin ve uzak URL görüntüleri desteklenir; ses, video, dosya yüklemeleri ve yerel/Base64
görüntüler guild kanallarında kullanılamaz. Tepkiler ve ileti dizileri hiçbir yerde
desteklenmez.

Durum: resmi indirilebilir plugin.

## Kurulum

```bash
openclaw plugins install @openclaw/qqbot
```

## Ayarlama

1. [QQ Open Platform](https://q.qq.com/) adresine gidin ve kaydolmak / oturum açmak için QR kodunu
   telefonunuzdaki QQ ile tarayın.
2. Yeni bir QQ botu oluşturmak için **Create Bot** seçeneğine tıklayın.
3. Botun ayarlar sayfasında **AppID** ve **AppSecret** değerlerini bulup kopyalayın.

<Note>
AppSecret düz metin olarak saklanmaz. Kaydetmeden sayfadan ayrılırsanız yeni bir tane oluşturmanız gerekir.
</Note>

4. Kanalı ekleyin:

```bash
openclaw channels add --channel qqbot --token "AppID:AppSecret"
```

5. Gateway'i yeniden başlatın.

## Gelen olayların dayanıklılığı

QQ gateway tur olaylarında OpenClaw, kaydedilmiş gateway sürdürme sırasını ilerletmeden önce ham olayı kalıcı olarak saklar. Bekleyen veya yeniden denenebilir turlar Gateway yeniden başlatıldığında korunur, konuşma başına sıralı kalır ve etkin ya da saklanan tamamlanma kaydı var olduğu sürece yinelenen kuyruk girdilerini engellemek için sağlayıcı olay kimliğini kullanır.

Dayanıklı kabul başarısız olursa OpenClaw, sırayı ilerletmeden mevcut gateway soketini sonlandırır. Yeniden bağlanma/sürdürme yolu daha sonra kaydedilmemiş olayı tekrar isteyebilir. Kuyruktan agente aktarım sınırında teslimat yine en az bir kez gerçekleşir; dolayısıyla aktarım sırasında oluşan bir çökme, bir turun yeniden yürütülmesine neden olabilir.

Etkileşimli ayarlama:

```bash
openclaw channels add
```

Sihirbaz, AppID/AppSecret değerlerini elle yazmaya alternatif olarak QR koduyla bağlama seçeneği de sunar:
bağlamayı tamamlamak için kodu hedef QQ Bot'a bağlı telefon uygulamasıyla tarayın.
OpenClaw, döndürülen kimlik bilgilerini hesabın yapılandırma
kapsamında kalıcı olarak saklar.

## Yapılandırma

Asgari yapılandırma:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecret: "YOUR_APP_SECRET",
    },
  },
}
```

Varsayılan hesap ortam değişkenleri (yalnızca üst düzey hesap):

- `QQBOT_APP_ID`
- `QQBOT_CLIENT_SECRET`

Dosya tabanlı AppSecret:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecretFile: "/path/to/qqbot-secret.txt",
    },
  },
}
```

Ortam SecretRef AppSecret:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecret: { source: "env", provider: "default", id: "QQBOT_CLIENT_SECRET" },
    },
  },
}
```

Notlar:

- `openclaw channels add --channel qqbot --token-file ...` yalnızca AppSecret değerini ayarlar;
  `appId` yapılandırmada veya `QQBOT_APP_ID` içinde önceden ayarlanmış olmalıdır.
- `clientSecret` düz metin dizesini, dosya yolunu (`clientSecretFile`)
  veya yapılandırılmış bir SecretRef nesnesini kabul eder.
- Eski `secretref:...` / `secretref-env:...` işaretleyici dizeleri
  `clientSecret` için reddedilir; bunun yerine yapılandırılmış bir SecretRef nesnesi kullanın.

### Akış

```json5
{
  channels: {
    qqbot: {
      streaming: {
        mode: "partial", // blok akışı: "partial" (varsayılan) veya "off"
        nativeTransport: true, // DM'ler için QQ'nun resmi C2C stream_messages API'sini kullan
      },
    },
  },
}
```

- `streaming.mode: "off"` hesap için blok akışını devre dışı bırakır.
- `streaming.nativeTransport: true`, C2C (DM) yanıtlarını QQ'nun
  resmi `stream_messages` API'si üzerinden akıtır; grup/kanal hedefleri bundan etkilenmez.
- Eski `streaming: true|false` skalerleri ve `streaming.c2cStreamApi` anahtarı
  `openclaw doctor --fix` aracılığıyla bu biçime geçirilir.
- `/bot-streaming on|off`, bir DM'den aynı yapılandırmayı değiştirir.

### Erişim politikası

- `allowFrom` / `groupAllowFrom`, C2C /
  grup bağlamlarında botla kimlerin sohbet edebileceğini sınırlar. `dmPolicy` / `groupPolicy` (`open` | `allowlist` | `disabled`)
  uygulama modunu denetler. `allowFrom` somut (joker olmayan) bir girdi içerdiğinde
  `dmPolicy` varsayılan olarak `allowlist` olur; aksi takdirde `open` olur.
  `groupAllowFrom` veya `allowFrom` somut bir girdi içerdiğinde
  `groupPolicy` varsayılan olarak `allowlist` olur; aksi takdirde `open` olur.
- "Auth: allowlist" eğik çizgi komutları, `dmPolicy` / `groupPolicy`
  değerlerinden bağımsız olarak `allowFrom` içinde (veya grup çağrıları için `groupAllowFrom` içinde)
  açıkça belirtilmiş joker olmayan bir girdi gerektirir — bkz. [Eğik çizgi komutları](#slash-commands).

### Çok hesaplı ayarlama

Tek bir OpenClaw örneği altında birden fazla QQ botu çalıştırın:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "111111111",
      clientSecret: "secret-of-bot-1",
      accounts: {
        bot2: {
          enabled: true,
          appId: "222222222",
          clientSecret: "secret-of-bot-2",
        },
      },
    },
  },
}
```

Her hesap, `appId` ile anahtarlanan yalıtılmış bir WebSocket bağlantısına, API istemcisine ve token
önbelleğine sahiptir. Tek bir Gateway altında birden fazla bot çalıştırıldığında
tanılamaların ayrı kalması için günlük satırları sahip hesabın kimliğiyle etiketlenir.

CLI aracılığıyla ikinci bir bot ekleyin:

```bash
openclaw channels add --channel qqbot --account bot2 --token "222222222:secret-of-bot-2"
```

### Grup sohbetleri

Grup desteği, görünen adları değil QQ grup OpenID'lerini kullanır. Botu bir
gruba ekleyin, ardından bottan bahsedin veya grubu bahsetme olmadan çalışacak şekilde yapılandırın.

```json5
{
  channels: {
    qqbot: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["member_openid"],
      groups: {
        "*": {
          requireMention: true,
          commandLevel: "all",
          historyLimit: 50,
          tools: { deny: ["exec", "read", "write"] },
        },
        GROUP_OPENID: {
          name: "Release room",
          requireMention: false,
          ignoreOtherMentions: true,
          commandLevel: "safety",
          historyLimit: 20,
          prompt: "Keep replies short and operational.",
        },
      },
    },
  },
}
```

`groups["*"]` her grup için varsayılanları belirler; somut bir `groups.GROUP_OPENID`
girdisi, bir grup için bu varsayılanları geçersiz kılar. Grup ayarları:

| Alan                  | Varsayılan       | Açıklama                                                                                           |
| --------------------- | ---------------- | -------------------------------------------------------------------------------------------------- |
| `requireMention`      | `true`           | Bot yanıt vermeden önce bir `@`-bahsetmesi gerektirir.                                              |
| `commandLevel`        | `all`            | Grupta hangi yerleşik eğik çizgi komutlarının çalışabileceğini belirler (aşağıya bakın).             |
| `ignoreOtherMentions` | `false`          | Bottan değil başka birinden bahseden mesajları bırakır.                                             |
| `historyLimit`        | `50`             | Sonraki bahsetmeli tur için bağlam olarak tutulan son bahsetmesiz mesajlar. `0` geçmişi devre dışı bırakır. |
| `tools`               | —                | Tüm grup için araçlara izin verir veya araçları reddeder.                                           |
| `toolsBySender`       | —                | Gönderen başına araç geçersiz kılmaları; bkz. [Gruplar](/tr/channels/groups#groupchannel-tool-restrictions-optional). |
| `name`                | openid öneki     | Günlüklerde ve grup bağlamında kullanılan kolay anlaşılır etiket.                                   |
| `prompt`              | yerleşik varsayılan | Agent bağlamına eklenen grup başına davranış istemi.                                               |

`commandLevel` şunları kabul eder:

| Düzey    | Davranış                                                                                                                                      |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `all`    | Mevcut yerleşik komutlar kullanılabilir durumda kalır. Bazıları menülerde gizli kalır ancak yetkili kullanıcılar bunları grupta çalıştırmaya devam edebilir. |
| `safety` | `/help`, `/btw`, `/stop` grupta görünür kalır; hassas komutlar (`/config`, `/tools`, `/bash` vb.) özel sohbette çalıştırılmalıdır. |
| `strict` | Yalnızca katı çalışma için gereken grup oturumu denetimlerine izin verilir. Yetkili bir gönderenin etkin bir çalışmayı kesebilmesi için `/stop` çalışmaya devam eder. |

Eski QQBot `toolPolicy` girdileri kullanımdan kaldırılmıştır. Bunları `tools` biçimine geçirmek için `openclaw doctor --fix` komutunu çalıştırın.

Etkinleştirme modları `mention` ve `always` şeklindedir. `requireMention: true`,
`mention` ile; `requireMention: false` ise `always` ile eşleşir. Mevcut olduğunda oturum düzeyindeki etkinleştirme
geçersiz kılması yapılandırmaya göre önceliklidir.

Gelen kuyruk eş başınadır. Grup eşleri daha büyük bir kuyruk sınırına sahiptir (doğrudan
eşler için 20 yerine 50); kuyruk dolduğunda insan mesajlarından önce bot tarafından yazılan mesajları çıkarır
ve normal grup mesajlarının ani kümelerini, gönderenlerin belirtildiği tek bir turda birleştirir. Eğik çizgi
komutları, herhangi bir birleştirme toplu işleminden bağımsız olarak birer birer çalışır.

### Ses (STT / TTS)

STT ve TTS, öncelikli geri dönüşle iki düzeyli yapılandırmayı destekler:

| Ayar    | Plugin'e özgü                                            | Çerçeve geri dönüşü                                  |
| ------- | -------------------------------------------------------- | ---------------------------------------------------- |
| STT     | `channels.qqbot.stt`                                     | ses destekli ilk `tools.media.models[]` girdisi |
| TTS     | `channels.qqbot.tts`, `channels.qqbot.accounts.<id>.tts` | `tts`                                            |

```json5
{
  channels: {
    qqbot: {
      stt: {
        provider: "your-provider",
        model: "your-stt-model",
      },
      tts: {
        provider: "your-provider",
        model: "your-tts-model",
        voice: "your-voice",
      },
      accounts: {
        "qq-main": {
          tts: {
            providers: {
              openai: { voice: "shimmer" },
            },
          },
        },
      },
    },
  },
}
```

Devre dışı bırakmak için herhangi birinde `enabled: false` değerini ayarlayın. Hesap düzeyindeki TTS geçersiz kılmaları,
`tts` ile aynı biçimi kullanır ve kanal/genel TTS yapılandırması üzerinde derin birleştirme uygular.

STT istekleri varsayılan olarak 60 saniye sonra zaman aşımına uğrar. Plugin'e özgü STT, seçili
`models.providers.<id>.timeoutSeconds` geçersiz kılmasını kullanır. Çerçeve ses STT'si,
seçili ses destekli `tools.media.models[]` girdisinin `timeoutSeconds` değerini, ardından seçili sağlayıcı geçersiz kılmasını kullanır.

Gelen QQ ses ekleri, ham ses dosyaları genel `MediaPaths` dışında tutulurken
agentlere ses medyası meta verileri olarak sunulur. Düz metin yanıtındaki `[[audio_as_voice]]`,
TTS yapılandırılmışsa TTS sentezler ve yerel bir QQ sesli mesajı gönderir.

Giden ses yükleme/dönüştürme davranışı
`channels.qqbot.audioFormatPolicy` ile de ayarlanabilir:

- `sttDirectFormats`
- `uploadDirectFormats`
- `transcodeEnabled`

## Hedef biçimleri

| Biçim                      | Açıklama           |
| -------------------------- | ------------------ |
| `qqbot:c2c:OPENID`         | Özel sohbet (C2C)  |
| `qqbot:group:GROUP_OPENID` | Grup sohbeti        |
| `qqbot:channel:CHANNEL_ID` | Guild kanalı        |

<Note>
Her botun kendine ait kullanıcı OpenID kümesi vardır. Bot A tarafından alınan bir OpenID, Bot B üzerinden mesaj göndermek için **kullanılamaz**.
</Note>

## Eğik çizgi komutları

AI kuyruğundan önce yakalanan yerleşik komutlar:

| Komut                | Kimlik Doğrulama | Kapsam        | Açıklama                                                                       |
| -------------------- | ----------------- | ------------- | ------------------------------------------------------------------------------ |
| `/bot-ping`          | —                 | herhangi biri | Gecikme testi                                                                  |
| `/bot-help`          | —                 | herhangi biri | Tüm komutları listele                                                          |
| `/bot-me`            | —                 | yalnızca özel | `allowFrom` / `groupAllowFrom` kurulumu için gönderenin QQ kullanıcı kimliğini (openid) göster |
| `/bot-version`       | —                 | yalnızca özel | OpenClaw çerçeve sürümünü ve Plugin sürümünü göster                            |
| `/bot-upgrade`       | —                 | yalnızca özel | QQBot yükseltme kılavuzunun bağlantısını göster                                |
| `/bot-approve`       | izin listesi     | yalnızca özel | Komut yürütme onayı yapılandırmasını yönet (on / off / always / reset / status) |
| `/bot-logs`          | izin listesi     | yalnızca özel | Son gateway günlüklerini dosya olarak dışa aktar                               |
| `/bot-clear-storage` | izin listesi     | yalnızca özel | QQBot medya dizini altındaki önbelleğe alınmış indirmeleri sil                  |
| `/bot-streaming`     | izin listesi     | yalnızca özel | C2C akış yanıtlarını aç veya kapat                                              |
| `/bot-group-allways` | izin listesi     | yalnızca özel | Varsayılan grup etkinleştirme modunu değiştir (bahsetme gerekli veya her zaman açık) |

Kullanım yardımı için herhangi bir komuta `?` ekleyin (örneğin `/bot-upgrade ?`).

"Kimlik Doğrulama: izin listesi" komutları ayrıca gönderenin openid değerinin
açık bir joker karakter içermeyen `allowFrom` listesinde bulunmasını gerektirir (gruptan
verilen komutlarda `groupAllowFrom` önceliklidir; bulunmazsa `allowFrom` kullanılır).
Joker karakterli `allowFrom: ["*"]` sohbete izin verir ancak bu komutlara izin vermez.
Bunlardan birini özel sohbet dışında veya yetkisiz çalıştırmak, iletiyi sessizce
yok saymak yerine bir ipucu döndürür.

`/bot-me`, `/bot-version` ve `/bot-upgrade` yalnızca özel sohbette kullanılabilir ancak
izin listesi gerektirmez; herhangi bir C2C göndereni bunları çalıştırabilir.

QQ Bot yürütme onayları varsayılan aynı sohbet yedeğini kullandığında, yerel onay
düğmesi tıklamaları da aynı açık, joker karakter içermeyen komut izin listesini izler.
Daha geniş komut erişimi vermeden yalnızca onay erişimi vermek için
`channels.qqbot.execApprovals.approvers` yapılandırın. Yerel yürütme onayları varsayılan
olarak etkindir.

## Medya ve depolama

- Gelen, giden ve gateway köprüsü medyası, `~/.openclaw/media/qqbot` altında tek bir yük kökünü
  paylaşır (`OPENCLAW_HOME` ayarlandığında buna uyulur); böylece yüklemeler,
  indirmeler ve kod dönüştürme önbellekleri korumalı tek bir dizin altında kalır.
- C2C ve grup hedeflerine zengin medya teslimatı tek bir `sendMedia`
  yolu üzerinden gerçekleştirilir. 5&nbsp;MiB veya daha büyük yerel dosyalar ve bellek içi
  arabellekler QQ'nun parçalı yükleme uç noktalarını; daha küçük yükler ve uzak URL/Base64
  kaynakları ise tek seferlik yükleme API'sini kullanır.
- Bir çalışırken yükseltme, `openclaw.json` yazımı tamamlanmadan Gateway'i kesintiye
  uğratırsa Plugin, sonraki başlatmada söz konusu hesap için bilinen son
  `appId` / `clientSecret` değerlerini dahili bir anlık görüntüden geri yükler
  (kasıtlı bir yapılandırma değişikliğinin üzerine asla yazmaz); böylece QR kodunun
  yeniden taranması gerekmez.

## Sorun giderme

- **Gateway başlamıyor / gelen ileti yok:** `appId` ve
  `clientSecret` değerlerinin doğru olduğunu ve botun QQ Open Platform'da etkinleştirildiğini
  doğrulayın. Eksik bir kimlik bilgisi "QQBot not configured (missing appId or
  clientSecret)" olarak gösterilir.
- **`--token-file` ile kurulum hâlâ yapılandırılmamış görünüyor:** `--token-file`
  yalnızca AppSecret değerini ayarlar. `appId` yine de yapılandırmada veya `QQBOT_APP_ID`
  içinde ayarlanmalıdır.
- **Ani grup yanıtları çakışıyor:** bir eşin kuyruğu dolduğunda gelen ileti kuyruğu,
  bot tarafından yazılmış iletileri insan iletilerinden önce çıkarır ve normal (komut olmayan)
  grup iletilerinin ani yığınlarını ilişkilendirilmiş tek bir etkileşimde birleştirir; böylece
  yoğun bot konuşmaları insan iletilerinin işlenmesini engellememelidir.
- **Proaktif iletiler ulaşmıyor:** kullanıcı yakın zamanda etkileşimde bulunmadıysa
  QQ, bot tarafından başlatılan iletileri engelleyebilir.
- **Ses metne dönüştürülmüyor:** STT'nin yapılandırıldığından ve sağlayıcıya
  erişilebildiğinden emin olun.

## İlgili

- [Eşleştirme](/tr/channels/pairing)
- [Gruplar](/tr/channels/groups)
- [Kanal sorunlarını giderme](/tr/channels/troubleshooting)
