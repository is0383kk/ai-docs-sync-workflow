---
read_when:
    - Bir Yuanbao botuna bağlanmak istiyorsunuz
    - Yuanbao kanalını yapılandırıyorsunuz
summary: Yuanbao botuna genel bakış, özellikler ve yapılandırma
title: Yuanbao
x-i18n:
    generated_at: "2026-07-26T23:13:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 43488834f588530206b290cb0fb185fd1fe2e1f214ab4a4ccccc49b9b549b6ac
    source_path: channels/yuanbao.md
    workflow: 16
---

Tencent Yuanbao, Tencent'in yapay zekâ asistan platformudur. Topluluk tarafından bakımı yapılan `openclaw-plugin-yuanbao` plugini, doğrudan mesajlar ve grup sohbetleri için Yuanbao botlarını WebSocket üzerinden OpenClaw'a bağlar.

**Durum:** bot DM'leri ve grup sohbetleri için üretime hazırdır. Desteklenen tek bağlantı modu WebSocket'tir. Bu plugin, çekirdek OpenClaw tarafından değil, harici bir katalog girdisi olarak Tencent Yuanbao ekibi tarafından yönetilir; aşağıdaki yapılandırma/davranış ayrıntıları (kurulum ve genel CLI yüzeyi dışındakiler) pluginin kendi belgelerinden alınmıştır ve OpenClaw çekirdek kaynağına göre doğrulanmamıştır.

## Hızlı başlangıç

OpenClaw 2026.4.10 veya üzerini gerektirir. `openclaw --version` ile denetleyin; `openclaw update` ile yükseltin.

<Steps>
  <Step title="Kimlik bilgilerinizle Yuanbao kanalını ekleyin">
  ```bash
  openclaw channels add --channel yuanbao --token "appKey:appSecret"
  ```
  `--token`, iki nokta üst üste ile ayrılmış `appKey:appSecret` kullanır. Bunları, uygulama ayarlarınızda bir bot oluşturarak Yuanbao uygulamasından edinin.
  </Step>

  <Step title="Değişikliği uygulamak için Gateway'i yeniden başlatın">
  ```bash
  openclaw gateway restart
  ```
  </Step>
</Steps>

### Etkileşimli kurulum (alternatif)

```bash
openclaw channels login --channel yuanbao
```

App ID ve App Secret bilgilerinizi girmek için istemleri izleyin.

## Erişim denetimi

### Doğrudan mesajlar

`channels.yuanbao.dm.policy`:

| Değer            | Davranış                                          |
| ---------------- | ------------------------------------------------- |
| `open` (varsayılan) | Tüm kullanıcılara izin ver                                   |
| `pairing`        | Bilinmeyen kullanıcılar bir eşleştirme kodu alır; CLI üzerinden onaylayın |
| `allowlist`      | Yalnızca `allowFrom` içindeki kullanıcılar sohbet edebilir                |
| `disabled`       | Tüm DM'leri devre dışı bırak                                   |

Bir eşleştirme isteğini onaylayın:

```bash
openclaw pairing list yuanbao
openclaw pairing approve yuanbao <CODE>
```

### Grup sohbetleri

`channels.yuanbao.requireMention` (varsayılan `true`): botun bir grupta yanıt vermesinden önce @bahsetme gerektirir. Botun kendi mesajına verilen yanıt, örtük bir bahsetme olarak değerlendirilir.

## Yapılandırma örnekleri

Temel kurulum, açık DM politikası:

```json5
{
  channels: {
    yuanbao: {
      appKey: "your_app_key",
      appSecret: "your_app_secret",
      dm: {
        policy: "open",
      },
    },
  },
}
```

DM'leri belirli kullanıcılarla sınırlandırın:

```json5
{
  channels: {
    yuanbao: {
      appKey: "your_app_key",
      appSecret: "your_app_secret",
      dm: {
        policy: "allowlist",
        allowFrom: ["user_id_1", "user_id_2"],
      },
    },
  },
}
```

Gruplarda @bahsetme gereksinimini devre dışı bırakın:

```json5
{
  channels: {
    yuanbao: {
      requireMention: false,
    },
  },
}
```

Giden teslimatı ayarlama:

```json5
{
  channels: {
    yuanbao: {
      outboundQueueStrategy: "merge-text",
      minChars: 2800, // bu karakter sayısına ulaşana kadar arabelleğe al
      maxChars: 3000, // bu sınırın üzerinde bölmeyi zorunlu kıl
      idleMs: 5000, // boşta kalma zaman aşımından (ms) sonra otomatik gönder
    },
  },
}
```

Her parçayı arabelleğe almadan göndermek için `outboundQueueStrategy: "immediate"` olarak ayarlayın.

## Yaygın komutlar

| Komut    | Açıklama                 |
| ---------- | --------------------------- |
| `/help`    | Kullanılabilir komutları göster     |
| `/status`  | Bot durumunu göster             |
| `/new`     | Yeni bir oturum başlat         |
| `/stop`    | Geçerli çalıştırmayı durdur        |
| `/restart` | OpenClaw'ı yeniden başlat            |
| `/compact` | Oturum bağlamını sıkıştır |

Yuanbao yerel eğik çizgi komutu menülerini destekler; komutlar Gateway başlatıldığında platformla otomatik olarak eşitlenir.

## Sorun giderme

**Bot grup sohbetlerinde yanıt vermiyor:**

1. Botun gruba eklendiğini doğrulayın
2. Bottan @bahsedildiğini doğrulayın (varsayılan olarak gereklidir)
3. Günlükleri denetleyin: `openclaw logs --follow`

**Bot mesajları almıyor:**

1. Botun Yuanbao uygulamasında oluşturulduğunu ve onaylandığını doğrulayın
2. `appKey` ve `appSecret` değerlerinin doğru yapılandırıldığını doğrulayın
3. Gateway'in çalıştığını doğrulayın: `openclaw gateway status`
4. Günlükleri denetleyin: `openclaw logs --follow`

**Bot boş yanıtlar veya yedek yanıtlar gönderiyor:**

1. Yapay zekâ modelinin geçerli içerik döndürüp döndürmediğini denetleyin
2. Varsayılan yedek yanıt: "暂时无法解答，你可以换个问题问问我哦"
3. `channels.yuanbao.fallbackReply` ile özelleştirin

**App Secret sızdırıldı:**

1. Yuanbao uygulamasındaki App Secret değerini sıfırlayın
2. Yapılandırmanızdaki değeri güncelleyin
3. Gateway'i yeniden başlatın: `openclaw gateway restart`

## Gelişmiş yapılandırma

### Birden çok hesap

```json5
{
  channels: {
    yuanbao: {
      defaultAccount: "main",
      accounts: {
        main: {
          appKey: "key_xxx",
          appSecret: "secret_xxx",
          name: "Birincil bot",
        },
        backup: {
          appKey: "key_yyy",
          appSecret: "secret_yyy",
          name: "Yedek bot",
          enabled: false,
        },
      },
    },
  },
}
```

`defaultAccount`, giden API'ler bir `accountId` belirtmediğinde hangi hesabın kullanılacağını denetler.

### Mesaj sınırları

- `maxChars`: tek mesaj için en fazla karakter sayısı (varsayılan `3000`)
- `mediaMaxMb`: medya yükleme/indirme sınırı (varsayılan `20` MB)
- `overflowPolicy`: bir mesaj sınırı aştığındaki davranış, `"split"` (varsayılan) veya `"stop"`

### Akış

Yuanbao blok düzeyinde akış çıktısını destekler; bot metni oluştururken parçalar hâlinde gönderir.

```json5
{
  channels: {
    yuanbao: {
      disableBlockStreaming: false, // blok akışı etkin (varsayılan)
    },
  },
}
```

Yanıtın tamamını tek bir mesajda göndermek için `disableBlockStreaming: true` olarak ayarlayın.

### Grup sohbeti geçmişi bağlamı

```json5
{
  channels: {
    yuanbao: {
      historyLimit: 100, // varsayılan: 100, devre dışı bırakmak için 0 olarak ayarlayın
    },
  },
}
```

Grup sohbetleri için yapay zekâ bağlamına kaç geçmiş mesajın ekleneceğini denetler.

### Yanıtlama modu

```json5
{
  channels: {
    yuanbao: {
      replyToMode: "first", // "off" | "first" | "all" (varsayılan: "first")
    },
  },
}
```

| Değer   | Davranış                                                 |
| ------- | -------------------------------------------------------- |
| `off`   | Alıntılı yanıt yok                                           |
| `first` | Her gelen mesaj için yalnızca ilk yanıtı alıntıla (varsayılan) |
| `all`   | Her yanıtı alıntıla                                        |

### Markdown ipucu ekleme

Bot, varsayılan olarak modelin yanıtın tamamını bir markdown kod bloğuna sarmasını önlemek için sistem istemine bir talimat ekler.

```json5
{
  channels: {
    yuanbao: {
      markdownHintEnabled: true, // varsayılan: true
    },
  },
}
```

### Hata ayıklama modu

```json5
{
  channels: {
    yuanbao: {
      debugBotIds: ["bot_user_id_1", "bot_user_id_2"],
    },
  },
}
```

Listelenen bot kimlikleri için temizlenmemiş günlük çıktısını etkinleştirir.

### Çok aracılı yönlendirme

Yuanbao DM'lerini veya gruplarını farklı aracılara yönlendirmek için `bindings` kullanın:

```json5
{
  agents: {
    list: [
      { id: "main" },
      { id: "agent-a", workspace: "/home/user/agent-a" },
      { id: "agent-b", workspace: "/home/user/agent-b" },
    ],
  },
  bindings: [
    {
      agentId: "agent-a",
      match: {
        channel: "yuanbao",
        peer: { kind: "direct", id: "user_xxx" },
      },
    },
    {
      agentId: "agent-b",
      match: {
        channel: "yuanbao",
        peer: { kind: "group", id: "group_zzz" },
      },
    },
  ],
}
```

- `match.channel`: `"yuanbao"`
- `match.peer.kind`: `"direct"` (DM) veya `"group"` (grup sohbeti)
- `match.peer.id`: kullanıcı kimliği veya grup kodu

## Yapılandırma başvurusu

Tam yapılandırma: [Gateway yapılandırması](/tr/gateway/configuration)

| Ayar                                    | Açıklama                                       | Varsayılan                                |
| ------------------------------------------ | ------------------------------------------------- | -------------------------------------- |
| `channels.yuanbao.enabled`                 | Kanalı etkinleştir/devre dışı bırak                        | `true`                                 |
| `channels.yuanbao.defaultAccount`          | Giden yönlendirme için varsayılan hesap              | `default`                              |
| `channels.yuanbao.accounts.<id>.appKey`    | App Key (imzalama + bilet oluşturma)             | -                                      |
| `channels.yuanbao.accounts.<id>.appSecret` | App Secret (imzalama)                              | -                                      |
| `channels.yuanbao.accounts.<id>.token`     | Önceden imzalanmış belirteç (otomatik bilet imzalamayı atlar) | -                                      |
| `channels.yuanbao.accounts.<id>.name`      | Hesabın görünen adı                              | -                                      |
| `channels.yuanbao.accounts.<id>.enabled`   | Belirli bir hesabı etkinleştir/devre dışı bırak                 | `true`                                 |
| `channels.yuanbao.dm.policy`               | DM politikası                                         | `open`                                 |
| `channels.yuanbao.dm.allowFrom`            | DM izin listesi (kullanıcı kimliği listesi)                       | -                                      |
| `channels.yuanbao.requireMention`          | Gruplarda @bahsetme gerektir                        | `true`                                 |
| `channels.yuanbao.overflowPolicy`          | Uzun mesaj işleme (`split` veya `stop`)         | `split`                                |
| `channels.yuanbao.replyToMode`             | Grup yanıtlama stratejisi (`off`, `first`, `all`)   | `first`                                |
| `channels.yuanbao.outboundQueueStrategy`   | Giden strateji (`merge-text` veya `immediate`)   | `merge-text`                           |
| `channels.yuanbao.minChars`                | Metin birleştirme: göndermeyi tetikleyecek en az karakter sayısı             | `2800`                                 |
| `channels.yuanbao.maxChars`                | Metin birleştirme: mesaj başına en fazla karakter sayısı                 | `3000`                                 |
| `channels.yuanbao.idleMs`                  | Metin birleştirme: otomatik göndermeden önce boşta kalma zaman aşımı (ms)   | `5000`                                 |
| `channels.yuanbao.mediaMaxMb`              | Medya boyutu sınırı (MB)                             | `20`                                   |
| `channels.yuanbao.historyLimit`            | Grup sohbeti geçmiş bağlamı girdileri                | `100`                                  |
| `channels.yuanbao.disableBlockStreaming`   | Blok düzeyinde akış çıktısını devre dışı bırak              | `false`                                |
| `channels.yuanbao.fallbackReply`           | Model içerik döndürmediğinde kullanılacak yedek yanıt  | `暂时无法解答，你可以换个问题问问我哦` |
| `channels.yuanbao.markdownHintEnabled`     | Markdown sarmalamayı önleme talimatları ekle        | `true`                                 |
| `channels.yuanbao.debugBotIds`             | Hata ayıklama izin listesindeki bot kimlikleri (temizlenmemiş günlükler)        | `[]`                                   |

## Desteklenen mesaj türleri

**Alma:** metin, görseller, dosyalar, ses/sesli mesaj, video, çıkartmalar/özel emojiler, özel öğeler (bağlantı kartları).

**Gönderme:** metin (markdown), görseller, dosyalar, ses, video, çıkartmalar.

**İş parçacıkları ve yanıtlar:** alıntılı yanıtlar (`replyToMode` aracılığıyla yapılandırılabilir); iş parçacığı yanıtları platform tarafından desteklenmez.

## İlgili

- [Kanallara Genel Bakış](/tr/channels) - desteklenen tüm kanallar
- [Eşleştirme](/tr/channels/pairing) - DM kimlik doğrulaması ve eşleştirme akışı
- [Gruplar](/tr/channels/groups) - grup sohbeti davranışı ve bahsetme denetimi
- [Kanal Yönlendirme](/tr/channels/channel-routing) - iletiler için oturum yönlendirmesi
- [Güvenlik](/tr/gateway/security) - erişim modeli ve sağlamlaştırma
