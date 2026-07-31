---
read_when:
    - OpenClaw'u LINE'a bağlamak istiyorsunuz
    - LINE webhook + kimlik bilgileri kurulumu gereklidir
    - LINE'a özgü mesaj seçenekleri istiyorsunuz
summary: LINE Messaging API Plugin kurulumu, yapılandırması ve kullanımı
title: LINE
x-i18n:
    generated_at: "2026-07-26T23:49:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa160970278e0899637307136139f7d2fc83bf57defc30771d77649060f77274
    source_path: channels/line.md
    workflow: 16
---

LINE, LINE Messaging API aracılığıyla OpenClaw'a bağlanır. Plugin, Gateway üzerinde bir Webhook
alıcısı olarak çalışır ve kimlik doğrulama için kanal erişim belirtecinizi + kanal gizli anahtarınızı
kullanır.

Durum: Resmî Plugin, ayrı olarak kurulur. Doğrudan mesajlar, grup sohbetleri, medya,
konumlar, Flex mesajları, şablon mesajları ve hızlı yanıtlar desteklenir.
Tepkiler ve ileti dizileri desteklenmez.

## Kurulum

Kanalı yapılandırmadan önce LINE'ı kurun:

```bash
openclaw plugins install @openclaw/line
```

Yerel çalışma kopyası (bir git deposundan çalıştırırken):

```bash
openclaw plugins install ./path/to/local/line-plugin
```

## Ayarlama

1. Bir LINE Developers hesabı oluşturun ve Console'u açın:
   [https://developers.line.biz/console/](https://developers.line.biz/console/)
2. Bir Provider oluşturun (veya seçin) ve bir **Messaging API** kanalı ekleyin.
3. Kanal ayarlarından **Channel access token** ve **Channel secret** değerlerini kopyalayın.
4. Messaging API ayarlarında **Use webhook** seçeneğini etkinleştirin.
5. Webhook URL'sini Gateway uç noktanıza ayarlayın (HTTPS gereklidir):

```text
https://gateway-host/line/webhook
```

Gateway, LINE'ın Webhook doğrulamasını (GET) yanıtlar. İmzalı gelen olaylarda
(POST), `200` döndürmeden önce her olayı kalıcı giriş kuyruğuna yazar;
ajan işlemesi eşzamansız olarak devam eder. Başarısız teslimat, Gateway yeniden
başlatıldıktan sonra da kuyruktan yeniden denenir ve sorunlu olaylar sınırlı sayıda
yeniden denemeden sonra başarısız kuyruk kayıtlarına dönüşür. Kalıcı depolama başarısız
olursa istek, kaybolabilecek bir olayı onaylamak yerine `500` döndürür.
Kuyruktan ajana geçiş sınırında teslimat en az bir kez gerçekleşir: etkin bir teslimat
sırasında Gateway'in kapanması veya çökmesi, turun yeniden oynatılmasına neden olabilir.
Mesaj olaylarının yinelenmesi LINE mesaj kimliğine göre önlenir; diğer olay türleri
`webhookEventId` kullanır. Saklanan tamamlanma kayıtları sıradan yinelenen Webhook'ları
engeller, ancak harici yan etkiler gerçekleştiren işleyiciler yine de eş etkili olmalıdır.
Özel bir yol gerekiyorsa `channels.line.webhookPath` veya
`channels.line.accounts.<id>.webhookPath` ayarını belirleyin ve URL'yi buna göre güncelleyin.

Güvenlik notları:

- LINE imza doğrulaması gövdeye bağlıdır (ham gövde üzerinden HMAC); bu nedenle OpenClaw, doğrulamadan önce katı bir kimlik doğrulama öncesi gövde sınırı (64 KB) ve okuma zaman aşımı uygular.
- OpenClaw, Webhook olaylarını doğrulanmış ham istek baytlarından işler. İmza bütünlüğü güvenliği için üst katman ara yazılımı tarafından dönüştürülen `req.body` değerleri yok sayılır.

## Yapılandırma

Asgari yapılandırma:

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "LINE_CHANNEL_ACCESS_TOKEN",
      channelSecret: "LINE_CHANNEL_SECRET",
      dmPolicy: "pairing",
    },
  },
}
```

Herkese açık DM yapılandırması:

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "LINE_CHANNEL_ACCESS_TOKEN",
      channelSecret: "LINE_CHANNEL_SECRET",
      dmPolicy: "open",
      allowFrom: ["*"],
    },
  },
}
```

Ortam değişkenleri (yalnızca varsayılan hesap):

- `LINE_CHANNEL_ACCESS_TOKEN`
- `LINE_CHANNEL_SECRET`

Belirteç/gizli anahtar dosyaları:

```json5
{
  channels: {
    line: {
      tokenFile: "/path/to/line-token.txt",
      secretFile: "/path/to/line-secret.txt",
    },
  },
}
```

`tokenFile` ve `secretFile` normal dosyalara işaret etmelidir. Sembolik bağlantılar reddedilir.
Satır içi yapılandırma değerleri dosyalara göre önceliklidir; ortam değişkenleri varsayılan hesap için son geri dönüş seçeneğidir.

Birden fazla hesap:

```json5
{
  channels: {
    line: {
      accounts: {
        marketing: {
          channelAccessToken: "...",
          channelSecret: "...",
          webhookPath: "/line/marketing",
        },
      },
    },
  },
}
```

## Erişim denetimi

Doğrudan mesajlar varsayılan olarak eşleştirmeyi kullanır. Bilinmeyen gönderenler bir
eşleştirme kodu alır ve onaylanana kadar mesajları yok sayılır:

```bash
openclaw pairing list line
openclaw pairing approve line <CODE>
```

İzin listeleri ve ilkeler:

- `channels.line.dmPolicy`: `pairing | allowlist | open | disabled` (varsayılan `pairing`)
- `channels.line.allowFrom`: DM'ler için izin verilen LINE kullanıcı kimlikleri; `dmPolicy: "open"`, `["*"]` gerektirir
- `channels.line.groupPolicy`: `allowlist | open | disabled` (varsayılan `allowlist`)
- `channels.line.groupAllowFrom`: gruplar için izin verilen LINE kullanıcı kimlikleri; DM `allowFrom` girdileri grup gönderenlerine izin vermez
- Grup başına geçersiz kılmalar: `channels.line.groups.<groupId>.allowFrom` (ayrıca `enabled`, `requireMention`, `systemPrompt`, `skills`). `groupPolicy: "allowlist"` ile
  `groupAllowFrom` veya grup başına `allowFrom` değerini ayarlayın; boş bir grup izin listesi, DM'ler açık olsa bile grup mesajlarını engeller.
- Statik gönderen erişim gruplarına `allowFrom`, `groupAllowFrom` ve grup başına `allowFrom` içinden `accessGroup:<name>` ile başvurulabilir; bkz. [Erişim grupları](/tr/channels/access-groups).
- Çalışma zamanı notu: `channels.line` tamamen eksikse çalışma zamanı, `channels.defaults.groupPolicy` ayarlanmış olsa bile grup denetimleri için `groupPolicy="allowlist"` değerine geri döner.

LINE kimlikleri büyük/küçük harfe duyarlıdır. Geçerli kimlikler şöyle görünür:

- Kullanıcı: `U` + 32 onaltılık karakter
- Grup: `C` + 32 onaltılık karakter
- Oda: `R` + 32 onaltılık karakter

## Mesaj davranışı

- Metin, 5000 karakterlik parçalara bölünür.
- Markdown biçimlendirmesi kaldırılır; kod blokları ve tablolar mümkün olduğunda Flex
  kartlarına dönüştürülür.
- Akış yanıtları arabelleğe alınır; ajan çalışırken LINE, yükleme
  animasyonuyla birlikte tam parçalar alır.
- Medya indirmeleri `channels.line.mediaMaxMb` ile sınırlandırılır (varsayılan 10).
- Gelen medya, ajana aktarılmadan önce `~/.openclaw/media/inbound/` altına
  kaydedilir; bu, diğer kanal Plugin'lerinin kullandığı ortak medya deposuyla aynıdır.

## Kanal verileri (zengin mesajlar)

Hızlı yanıtlar, konumlar, Flex kartları veya şablon mesajları göndermek için
`channelData.line` kullanın.

```json5
{
  text: "Buyurun",
  channelData: {
    line: {
      quickReplies: ["Durum", "Yardım"],
      location: {
        title: "Ofis",
        address: "123 Ana Cadde",
        latitude: 35.681236,
        longitude: 139.767125,
      },
      flexMessage: {
        altText: "Durum kartı",
        contents: {/* Flex yükü */},
      },
      templateMessage: {
        type: "confirm",
        text: "Devam edilsin mi?",
        confirmLabel: "Evet",
        confirmData: "yes",
        cancelLabel: "Hayır",
        cancelData: "no",
      },
    },
  },
}
```

LINE Plugin'i ayrıca Flex mesajı ön ayarları için bir `/card` komutuyla birlikte gelir:

```text
/card info "Hoş geldiniz" "Katıldığınız için teşekkürler!"
```

## ACP desteği

LINE, ACP (Ajan İletişim Protokolü) konuşma bağlamalarını destekler:

- `/acp spawn <agent> --bind here`, alt ileti dizisi oluşturmadan mevcut LINE sohbetini bir ACP oturumuna bağlar.
- Yapılandırılmış ACP bağlamaları ve konuşmaya bağlı etkin ACP oturumları, LINE'da diğer konuşma kanallarındaki gibi çalışır.

Ayrıntılar için [ACP ajanları](/tr/tools/acp-agents) bölümüne bakın.

## Giden medya

LINE Plugin'i, ajan mesaj aracılığıyla görüntü, video ve ses gönderir:

- **Görüntüler**: LINE görüntü mesajları olarak gönderilir; önizleme görüntüsü varsayılan olarak medya URL'sidir.
- **Videolar**: bir önizleme görüntüsü gerektirir; `channelData.line.previewImageUrl` değerini bir görüntü URL'sine ayarlayın.
- **Ses**: LINE ses mesajları olarak gönderilir; `channelData.line.durationMs` ayarlanmadığı sürece süre varsayılan olarak 60 saniyedir.

Medya türü, ayarlandığında `channelData.line.mediaKind` değerinden alınır; aksi takdirde
diğer LINE seçeneklerinden veya URL dosya uzantısından çıkarılır ve geri dönüş
seçeneği görüntüdür.

Giden medya URL'leri, en fazla 2000 karakter uzunluğunda herkese açık HTTPS URL'leri olmalıdır. OpenClaw,
URL'yi LINE'a iletmeden önce hedef ana bilgisayar adını doğrular ve geri döngü,
bağlantı-yerel ve özel ağ hedeflerini reddeder.

LINE'a özgü seçenekler içermeyen genel medya gönderimleri görüntü yolunu kullanır.

## Sorun giderme

- **Webhook doğrulaması başarısız oluyor:** Webhook URL'sinin HTTPS olduğundan ve
  `channelSecret` değerinin LINE Console ile eşleştiğinden emin olun.
- **Gelen olay yok:** Webhook yolunun `channels.line.webhookPath` ile eşleştiğini
  ve Gateway'e LINE tarafından erişilebildiğini doğrulayın.
- **Medya indirme hataları:** Medya varsayılan sınırı aşıyorsa
  `channels.line.mediaMaxMb` değerini artırın.

## İlgili

- [Kanallara Genel Bakış](/tr/channels) — desteklenen tüm kanallar
- [Eşleştirme](/tr/channels/pairing) — DM kimlik doğrulaması ve eşleştirme akışı
- [Gruplar](/tr/channels/groups) — grup sohbeti davranışı ve bahsetme denetimi
- [Kanal Yönlendirme](/tr/channels/channel-routing) — mesajlar için oturum yönlendirmesi
- [Güvenlik](/tr/gateway/security) — erişim modeli ve güvenliği sağlamlaştırma
