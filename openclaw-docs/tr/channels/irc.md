---
read_when:
    - OpenClaw'ı IRC kanallarına veya doğrudan mesajlara bağlamak istiyorsunuz
    - IRC izin listelerini, grup politikasını veya bahsetme geçidini yapılandırıyorsunuz
summary: IRC Plugin kurulumu, erişim denetimleri ve sorun giderme
title: IRC
x-i18n:
    generated_at: "2026-07-26T23:49:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 85c3da80b45d6611872ddbd10b3be4a5742b46e355e8bb554353a478f2a1702f
    source_path: channels/irc.md
    workflow: 16
---

Klasik kanallarda (`#room`) ve doğrudan mesajlarda OpenClaw kullanmak istediğinizde IRC kullanın.
Resmî IRC Plugin'ini yükleyin, ardından `channels.irc` altında yapılandırın.

## Hızlı başlangıç

1. Plugin'i yükleyin:

```bash
openclaw plugins install @openclaw/irc
```

2. `~/.openclaw/openclaw.json` içinde en az ana bilgisayarı, takma adı ve katılınacak kanalları ayarlayın:

```json5
{
  channels: {
    irc: {
      enabled: true,
      host: "irc.example.com",
      port: 6697,
      tls: true,
      nick: "openclaw-bot",
      channels: ["#openclaw"],
    },
  },
}
```

3. Gateway'i başlatın/yeniden başlatın:

```bash
openclaw gateway run
```

Bot koordinasyonu için özel bir IRC sunucusunu tercih edin. Bilerek genel bir IRC ağı kullanıyorsanız yaygın seçenekler arasında Libera.Chat, OFTC ve Snoonet bulunur. Bot veya sürü arka kanal trafiği için tahmin edilebilir genel kanallardan kaçının.

## Gelen iletilerin dayanıklılığı

OpenClaw, kabul edilen her IRC `PRIVMSG` öğesini normal politika kontrollerinden ve ajan yönlendirmesinden önce kalıcı giriş kuyruğuna yazar. Bekleyen veya yeniden denenebilir mesajlar Gateway yeniden başlatıldığında korunur ve kanal ya da doğrudan mesaj eşine göre sıralı kalır.

IRC, yeniden oynatılabilir bir teslimat kimliği sağlamaz veya bağlantısı kesilmiş bir istemcinin kaçırdığı mesajları yeniden göndermez. Bu nedenle OpenClaw, yalnızca mevcut TCP bağlantısı içinde kararlı olan yerel bir kimlik atar. Kuyruk, yerel kabulden yönlendirmeye kadar olan aralığı korur; OpenClaw'a hiç ulaşmamış bir mesajı kurtaramaz veya sunucunun bağlantılar arasında yeniden gönderdiği bir mesajı tekilleştiremez.

## Bağlantı ayarları

| Anahtar                       | Varsayılan                    | Notlar                                                      |
| ----------------------------- | ----------------------------- | ----------------------------------------------------------- |
| `host`                        | yok (gerekli)                 | IRC sunucusunun ana bilgisayar adı                           |
| `port`                        | TLS ile `6697`, düz metin için `6667` | 1-65535                                                     |
| `tls`                         | `true`                        | Yalnızca bilerek düz metin kullanmak için `false` olarak ayarlayın |
| `nick`                        | yok (gerekli)                 | Bot takma adı                                               |
| `username`                    | takma ad, yoksa `openclaw`    | IRC kullanıcı adı                                           |
| `realname`                    | `OpenClaw`                    | Gerçek ad/GECOS alanı                                       |
| `password` / `passwordFile`   | yok                           | Sunucu parolası; dosya normal bir dosya olmalıdır            |
| `channels`                    | yok                           | Katılınacak kanallar (`["#openclaw"]`)                   |
| `accounts` / `defaultAccount` | yok                           | Çok hesaplı kurulum; ortam değişkenleri yalnızca varsayılan hesabı doldurur |

## Güvenlik varsayılanları

- IRC, OpenClaw operatörü tarafından yönetilen ileri proxy yönlendirmesinin dışında ham TCP/TLS yuvaları kullanır. Tüm çıkış trafiğinin bu ileri proxy üzerinden geçmesini gerektiren dağıtımlarda, doğrudan IRC çıkışına açıkça onay verilmediği sürece `channels.irc.enabled=false` ayarını kullanın.
- `channels.irc.dmPolicy` varsayılan olarak `"pairing"` değerini kullanır: bilinmeyen doğrudan mesaj gönderenler, `openclaw pairing approve irc <code>` ile onaylayacağınız bir eşleştirme kodu alır.
- `channels.irc.groupPolicy` varsayılan olarak `"allowlist"` değerini kullanır.
- `groupPolicy="allowlist"` kullanırken izin verilen kanalları tanımlamak için `channels.irc.groups` ayarını belirleyin.
- Bilerek düz metin aktarımını kabul etmediğiniz sürece TLS (`channels.irc.tls=true`) kullanın.

## Erişim denetimi

IRC kanalları için iki ayrı "geçit" vardır:

1. **Kanal erişimi** (`groupPolicy` + `groups`): botun bir kanaldan gelen mesajları kabul edip etmeyeceği.
2. **Gönderen erişimi** (`groupAllowFrom` / kanal başına `groups["#channel"].allowFrom`): o kanal içinde botu kimlerin tetiklemesine izin verildiği.

Yapılandırma anahtarları:

- Doğrudan mesaj izin listesi (doğrudan mesaj gönderen erişimi): `channels.irc.allowFrom`
- Grup gönderen izin listesi (kanal gönderen erişimi): `channels.irc.groupAllowFrom`
- Kanal başına denetimler (kanal + gönderen + bahsetme kuralları): `requireMention`, `allowFrom`, `enabled`, `tools`, `toolsBySender`, `skills` ve `systemPrompt` ile `channels.irc.groups["#channel"]`
- `channels.irc.groupPolicy="open"`, yapılandırılmamış kanallara izin verir (**varsayılan olarak yine de bahsetme geçidi uygulanır**)

İzin listesi girdileri kararlı gönderen kimlikleri (`nick!user@host`) kullanmalıdır.
Yalın takma ad eşleştirmesi değişkendir ve yalnızca `channels.irc.dangerouslyAllowNameMatching: true` olduğunda etkinleştirilir.

### Yaygın sorun: `allowFrom` doğrudan mesajlar içindir, kanallar için değil

Şuna benzer günlükler görürseniz:

- `irc: drop group sender alice!ident@host (policy=allowlist)`

...bu, gönderenin **grup/kanal** mesajları için izinli olmadığı anlamına gelir. Şunlardan biriyle düzeltin:

- `channels.irc.groupAllowFrom` ayarını belirleyerek (tüm kanallar için genel) veya
- kanal başına gönderen izin listelerini belirleyerek: `channels.irc.groups["#channel"].allowFrom`

Örnek (`#openclaw` içindeki herkesin botla konuşmasına izin verin):

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": { allowFrom: ["*"] },
      },
    },
  },
}
```

## Yanıt tetikleme (bahsetmeler)

Bir kanala (`groupPolicy` + `groups` aracılığıyla) ve gönderene izin verilmiş olsa bile OpenClaw, grup bağlamlarında varsayılan olarak **bahsetme geçidi** uygular. Mesaj, bağlı botun takma adını içerdiğinde veya yapılandırılmış bahsetme kalıplarınızla eşleştiğinde botun adından bahsedilmiş sayılır.

Bu, mesaj botla eşleşen bir bahsetme kalıbı içermedikçe `drop channel … (missing-mention)` benzeri günlükler görebileceğiniz anlamına gelir.

Botun bir IRC kanalında **bahsetme gerektirmeden** yanıt vermesini sağlamak için o kanalın bahsetme geçidini devre dışı bırakın:

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": {
          requireMention: false,
          allowFrom: ["*"],
        },
      },
    },
  },
}
```

Veya **tüm** IRC kanallarına izin vermek (kanal başına izin listesi olmadan) ve yine de bahsetme olmadan yanıtlamak için:

```json5
{
  channels: {
    irc: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: false, allowFrom: ["*"] },
      },
    },
  },
}
```

## Güvenlik notu (genel kanallar için önerilir)

Genel bir kanalda `allowFrom: ["*"]` ayarına izin verirseniz herkes bota istem gönderebilir.
Riski azaltmak için o kanalın araçlarını kısıtlayın.

### Kanaldaki herkes için aynı araçlar

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          tools: {
            deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
          },
        },
      },
    },
  },
}
```

### Gönderene göre farklı araçlar (sahip daha fazla yetki alır)

`"*"` için daha katı, kendi takma adınız için daha gevşek bir politika uygulamak üzere `toolsBySender` kullanın:

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          toolsBySender: {
            "*": {
              deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
            },
            "id:alice": {
              deny: ["gateway", "nodes", "cron"],
            },
          },
        },
      },
    },
  },
}
```

Notlar:

- `toolsBySender` anahtarları açık önekler (`channel:`, `id:`, `e164:`, `username:`, `name:`) kullanmalıdır. IRC için gönderen kimliği değeriyle `id:` kullanın: daha güçlü eşleştirme için `id:alice` veya `id:alice!~alice@203.0.113.7`.
- Eski öneksiz anahtarlar hâlâ kabul edilir, yalnızca `id:` olarak eşleştirilir ve kullanımdan kaldırma uyarısı verir.
- İlk eşleşen gönderen politikası geçerli olur; `"*"` joker karakterli geri dönüş seçeneğidir.

Grup erişimi ile bahsetme geçidi (ve bunların etkileşimi) hakkında daha fazla bilgi için bkz. [/channels/groups](/tr/channels/groups).

## NickServ

Bağlantıdan sonra NickServ ile kimlik doğrulamak için:

```json5
{
  channels: {
    irc: {
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "your-nickserv-password",
      },
    },
  },
}
```

Bir parola ayarlandığında NickServ kimlik doğrulaması varsayılan olarak her zaman çalışır (devre dışı bırakmak için yalnızca `enabled` değerinin `false` olması gerekir). `service` varsayılan olarak `NickServ` değerini kullanır; `passwordFile`, satır içi `password` ayarına bir alternatiftir.

Bağlantıda isteğe bağlı tek seferlik kayıt (`register: true`, `registerEmail` gerektirir):

```json5
{
  channels: {
    irc: {
      nickserv: {
        register: true,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

Tekrarlanan REGISTER denemelerini önlemek için takma ad kaydedildikten sonra `register` ayarını devre dışı bırakın.

## Ortam değişkenleri

Varsayılan hesap şunları destekler:

- `IRC_HOST`
- `IRC_PORT`
- `IRC_TLS`
- `IRC_NICK`
- `IRC_USERNAME`
- `IRC_REALNAME`
- `IRC_PASSWORD`
- `IRC_CHANNELS` (virgülle ayrılmış)
- `IRC_NICKSERV_PASSWORD`
- `IRC_NICKSERV_REGISTER_EMAIL`

`IRC_HOST`, çalışma alanındaki bir `.env` üzerinden ayarlanamaz; bkz. [Çalışma alanı `.env` dosyaları](/tr/gateway/security).

## Sorun giderme

- Bot bağlanıyor ancak kanallarda hiç yanıt vermiyorsa `channels.irc.groups` ayarını **ve** bahsetme geçidinin mesajları düşürüp düşürmediğini (`missing-mention`) doğrulayın. Botun pingleme olmadan yanıt vermesini istiyorsanız kanal için `requireMention:false` ayarını belirleyin.
- Oturum açma başarısız olursa takma adın kullanılabilirliğini ve sunucu parolasını doğrulayın.
- Özel bir ağda TLS başarısız olursa ana bilgisayar/port ve sertifika kurulumunu doğrulayın.

## İlgili

- [Kanallara Genel Bakış](/tr/channels) — desteklenen tüm kanallar
- [Eşleştirme](/tr/channels/pairing) — doğrudan mesaj kimlik doğrulaması ve eşleştirme akışı
- [Gruplar](/tr/channels/groups) — grup sohbeti davranışı ve bahsetme geçidi
- [Kanal Yönlendirme](/tr/channels/channel-routing) — mesajların oturum yönlendirmesi
- [Güvenlik](/tr/gateway/security) — erişim modeli ve sağlamlaştırma
