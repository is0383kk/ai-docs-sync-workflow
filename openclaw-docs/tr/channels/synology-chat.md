---
read_when:
    - OpenClaw ile Synology Chat'i ayarlama
    - Synology Chat Webhook yönlendirmesinde hata ayıklama
summary: Synology Chat webhook kurulumu ve OpenClaw yapılandırması
title: Synology Chat
x-i18n:
    generated_at: "2026-07-26T23:13:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3c03379944ee4187260a7287f6d2aed1ad8fdd1c22b5581c8a5d55515bbb6ad5
    source_path: channels/synology-chat.md
    workflow: 16
---

Synology Chat, OpenClaw'a bir Webhook çifti aracılığıyla bağlanır: Synology Chat giden Webhook'u gelen doğrudan mesajları Gateway'e gönderir ve yanıtlar bir Synology Chat gelen Webhook'u üzerinden geri iletilir.

Durum: resmi Plugin, ayrı olarak kurulur. Yalnızca doğrudan mesajlar; metin ve URL tabanlı dosya gönderimleri desteklenir.

## Kurulum

```bash
openclaw plugins install @openclaw/synology-chat
```

Yerel çalışma kopyası (bir git deposundan çalıştırırken):

```bash
openclaw plugins install ./path/to/local/synology-chat-plugin
```

Ayrıntılar: [Pluginler](/tr/tools/plugin)

## Hızlı kurulum

1. Plugin'i kurun (yukarıda).
2. Synology Chat entegrasyonlarında:
   - Bir gelen Webhook oluşturun ve URL'sini kopyalayın.
   - Gizli token'ınızla bir giden Webhook oluşturun.
3. Giden Webhook URL'sini OpenClaw Gateway'inize yönlendirin:
   - Varsayılan olarak `https://gateway-host/webhook/synology`.
   - Ya da özel `channels.synology-chat.webhookPath` değeriniz.
4. Kurulumu OpenClaw'da tamamlayın. Synology Chat, her iki akışta da aynı kanal kurulum listesinde görünür:
   - Kılavuzlu: `openclaw onboard` veya `openclaw channels add`
   - Doğrudan: `openclaw channels add --channel synology-chat --token <token> --url <incoming-webhook-url>`
5. Gateway'i yeniden başlatın ve Synology Chat botuna bir doğrudan mesaj gönderin.

Webhook kimlik doğrulama ayrıntıları:

- OpenClaw, giden Webhook token'ını önce `body.token`, ardından
  `?token=...`, son olarak üstbilgilerden kabul eder.
- Kabul edilen üstbilgi biçimleri:
  - `x-synology-token`
  - `x-webhook-token`
  - `x-openclaw-token`
  - `Authorization: Bearer <token>`
- Boş veya eksik token'lar güvenli biçimde reddedilir.
- Yükler `application/x-www-form-urlencoded` veya `application/json` olabilir; `token`, `user_id` ve `text` zorunludur.

## Gelen verilerin kalıcılığı

Token, gönderen politikası ve hız sınırı denetimleri geçtikten sonra OpenClaw, Webhook token'ını depolanan zarftan kaldırır ve olayı onaylamadan önce kalıcı olarak kuyruğa alır. Rota, yalnızca bu ekleme başarılı olduktan sonra `204` döndürür; kalıcı depolama hatası `503` döndürür, böylece Synology Chat mesajı sessizce kaybetmek yerine yeniden deneyebilir.

Bekleyen veya yeniden denenebilir olaylar Gateway yeniden başlatıldığında korunur. Synology'nin kararlı `post_id` değeri, ilgili etkin veya saklanan tamamlanma kaydı var olduğu sürece yinelenen kuyruk girdilerini engeller. Kuyruktan aracıya devir boyunca teslimat en az bir kez yapılmaya devam eder; dolayısıyla bu sınırdaki bir çökme, bir etkileşimin yeniden oynatılmasına neden olabilir.

Asgari yapılandırma:

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      token: "synology-outgoing-token",
      incomingUrl: "https://nas.example.com/webapi/entry.cgi?api=SYNO.Chat.External&method=incoming&version=2&token=...",
      webhookPath: "/webhook/synology",
      dmPolicy: "allowlist",
      allowedUserIds: ["123456"],
      rateLimitPerMinute: 30,
      allowInsecureSsl: false,
    },
  },
}
```

## Ortam değişkenleri

Varsayılan hesap için ortam değişkenlerini kullanabilirsiniz:

- `SYNOLOGY_CHAT_TOKEN`
- `SYNOLOGY_CHAT_INCOMING_URL`
- `SYNOLOGY_NAS_HOST`
- `SYNOLOGY_ALLOWED_USER_IDS` (virgülle ayrılmış)
- `SYNOLOGY_RATE_LIMIT`
- `OPENCLAW_BOT_NAME`

Yapılandırma değerleri ortam değişkenlerini geçersiz kılar.

`SYNOLOGY_CHAT_INCOMING_URL` ve `SYNOLOGY_NAS_HOST`, çalışma alanındaki bir `.env` üzerinden ayarlanamaz; bkz. [Çalışma alanı `.env` dosyaları](/tr/gateway/security#workspace-env-files).

## Doğrudan mesaj politikası ve erişim denetimi

- Desteklenen `dmPolicy` değerleri: `allowlist` (varsayılan), `open` ve `disabled`. Synology Chat'te eşleştirme akışı yoktur; gönderenleri, sayısal Synology kullanıcı kimliklerini `allowedUserIds` alanına ekleyerek onaylayın.
- `allowedUserIds`, Synology kullanıcı kimliklerinin bir listesini (veya virgülle ayrılmış dizesini) kabul eder.
- `allowlist` modunda boş bir `allowedUserIds` listesi hatalı yapılandırma olarak değerlendirilir ve Webhook rotası başlatılmaz.
- `dmPolicy: "open"`, yalnızca `allowedUserIds` içinde `"*"` bulunduğunda herkese açık doğrudan mesajlara izin verir; kısıtlayıcı girdiler varsa yalnızca eşleşen kullanıcılar sohbet edebilir. Boş bir `allowedUserIds` listesiyle `open` de rotayı başlatmayı reddeder.
- `dmPolicy: "disabled"` doğrudan mesajları engeller.
- Yanıt alıcısı bağlaması varsayılan olarak kararlı sayısal `user_id` üzerinde kalır. `channels.synology-chat.dangerouslyAllowNameMatching: true`, yanıt teslimatı için değiştirilebilir kullanıcı adı/takma ad aramasını yeniden etkinleştiren acil durum uyumluluk modudur.

## Giden teslimat

Hedef olarak sayısal Synology Chat kullanıcı kimliklerini kullanın. `synology-chat:`, `synology_chat:` ve `synology:` önekleri kabul edilir.

Örnekler:

```bash
openclaw message send --channel synology-chat --target 123456 --message "Hello from OpenClaw"
openclaw message send --channel synology-chat --target synology-chat:123456 --message "Hello again"
openclaw message send --channel synology-chat --target synology:123456 --message "Short prefix"
```

Giden metin 2000 karakterlik parçalara bölünür. Medya gönderimleri URL tabanlı dosya teslimatıyla desteklenir: NAS dosyayı indirir ve ekler (en fazla 32 MB). Giden dosya URL'leri `http` veya `https` kullanmalıdır; özel ya da başka bir nedenle engellenen ağ hedefleri, OpenClaw URL'yi NAS Webhook'una iletmeden önce reddedilir.

## Çoklu hesap

`channels.synology-chat.accounts` altında birden fazla Synology Chat hesabı desteklenir.
Her hesap token'ı, gelen URL'yi, Webhook yolunu, doğrudan mesaj politikasını ve sınırları geçersiz kılabilir.
Doğrudan mesaj oturumları hesap ve kullanıcı başına yalıtılır; dolayısıyla iki farklı Synology hesabındaki aynı sayısal `user_id`
transkript durumunu paylaşmaz.
Etkinleştirilen her hesaba farklı bir `webhookPath` verin. OpenClaw, tamamen aynı yolları reddeder
ve çoklu hesap kurulumlarında yalnızca paylaşılan bir Webhook yolunu devralan adlandırılmış hesapları başlatmayı reddeder.
Adlandırılmış bir hesap için eski davranış devralmasını bilinçli olarak kullanmanız gerekiyorsa söz konusu hesapta veya `channels.synology-chat` konumunda
`dangerouslyAllowInheritedWebhookPath: true` ayarlayın;
ancak tamamen aynı yollar yine güvenli biçimde reddedilir. Hesap başına açıkça belirtilmiş yolları tercih edin.

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      accounts: {
        default: {
          token: "token-a",
          incomingUrl: "https://nas-a.example.com/...token=...",
        },
        alerts: {
          token: "token-b",
          incomingUrl: "https://nas-b.example.com/...token=...",
          webhookPath: "/webhook/synology-alerts",
          dmPolicy: "allowlist",
          allowedUserIds: ["987654"],
        },
      },
    },
  },
}
```

## Güvenlik notları

- `token` değerini gizli tutun ve sızarsa yenileyin.
- Kendinden imzalı yerel bir NAS sertifikasına açıkça güvenmiyorsanız `allowInsecureSsl: false` değerini koruyun.
- Gelen Webhook isteklerinin token'ı doğrulanır ve gönderen başına hız sınırı uygulanır (`rateLimitPerMinute`, varsayılan 30).
- Geçersiz token denetimleri sabit süreli gizli değer karşılaştırması kullanır ve güvenli biçimde reddedilir; yinelenen geçersiz token denemeleri kaynak IP'yi geçici olarak kilitler.
- Gelen mesaj metni, bilinen istem enjeksiyonu kalıplarına karşı temizlenir ve 4000 karakterde kesilir.
- Üretim için `dmPolicy: "allowlist"` tercih edin.
- Eski kullanıcı adı tabanlı yanıt teslimatına açıkça ihtiyacınız yoksa `dangerouslyAllowNameMatching` kapalı kalsın.
- Çoklu hesap kurulumunda paylaşılan yol yönlendirme riskini açıkça kabul etmiyorsanız `dangerouslyAllowInheritedWebhookPath` kapalı kalsın.

## Sorun giderme

- `Missing required fields (token, user_id, text)`:
  - giden Webhook yükünde zorunlu alanlardan biri eksik
  - Synology token'ı üstbilgilerde gönderiyorsa Gateway'in/proxy'nin bu üstbilgileri koruduğundan emin olun
- `Invalid token`:
  - giden Webhook gizli değeri `channels.synology-chat.token` ile eşleşmiyor
  - istek yanlış hesap/Webhook yoluna ulaşıyor
  - ters proxy, istek OpenClaw'a ulaşmadan önce token üstbilgisini kaldırdı
- `Rate limit exceeded`:
  - aynı kaynaktan çok fazla geçersiz token denemesi, söz konusu kaynağı geçici olarak kilitleyebilir
  - kimliği doğrulanmış gönderenler için ayrıca kullanıcı başına ayrı bir mesaj hız sınırı vardır
- `Allowlist is empty. Configure allowedUserIds or use dmPolicy=open with allowedUserIds=["*"].`:
  - `dmPolicy="allowlist"` etkinleştirilmiş ancak hiçbir kullanıcı yapılandırılmamış
- `User not authorized`:
  - gönderenin sayısal `user_id` değeri `allowedUserIds` içinde değil

## İlgili

- [Kanallara Genel Bakış](/tr/channels) — desteklenen tüm kanallar
- [Gruplar](/tr/channels/groups) — grup sohbeti davranışı ve bahsetme kısıtlaması
- [Kanal Yönlendirme](/tr/channels/channel-routing) — mesajlar için oturum yönlendirme
- [Güvenlik](/tr/gateway/security) — erişim modeli ve güvenliği güçlendirme
