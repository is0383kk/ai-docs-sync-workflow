---
read_when:
    - OpenClaw'u Twilio aracılığıyla SMS'e bağlamak istiyorsunuz
    - SMS webhook'u veya izin listesi kurulumu gerekiyor
summary: Twilio SMS kanalı kurulumu, erişim denetimleri ve webhook yapılandırması
title: SMS
x-i18n:
    generated_at: "2026-07-26T23:10:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 99a76b2f2d66858f8eb699939084104e620af9bc024053bbe1c1d7350530bff0
    source_path: channels/sms.md
    workflow: 16
---

OpenClaw, bir Twilio telefon numarası veya Messaging Service aracılığıyla SMS alır ve gönderir. Gateway bir gelen Webhook rotası (varsayılan `/webhooks/sms`) kaydeder, varsayılan olarak Twilio istek imzalarını doğrular ve yanıtları Twilio'nun Messages API'si aracılığıyla geri gönderir.

Durum: resmî Plugin, ayrı olarak kurulur. Yalnızca metin: MMS/medya yoktur, yalnızca doğrudan mesajlar desteklenir.

<CardGroup cols={3}>
  <Card title="Eşleştirme" icon="link" href="/tr/channels/pairing">
    SMS için varsayılan DM ilkesi eşleştirmedir.
  </Card>
  <Card title="Gateway güvenliği" icon="shield" href="/tr/gateway/security">
    Webhook erişimini ve gönderen erişim denetimlerini gözden geçirin.
  </Card>
  <Card title="Kanal sorunlarını giderme" icon="wrench" href="/tr/channels/troubleshooting">
    Kanallar arası tanılama ve onarım çalışma planları.
  </Card>
</CardGroup>

## Başlamadan önce

Gerekenler:

- `openclaw plugins install @openclaw/sms` ile kurulmuş resmî SMS Plugin'i.
- SMS özellikli bir telefon numarasına veya Twilio Messaging Service'e sahip bir Twilio hesabı.
- Twilio Account SID ve Auth Token.
- OpenClaw Gateway'inize ulaşan herkese açık bir HTTPS URL'si.
- Bir gönderen ilkesi seçimi: özel kullanım için `pairing` (varsayılan), önceden onaylanmış telefon numaraları için `allowlist` veya yalnızca bilerek herkese açık SMS erişimi sağlamak için `open`.

Tek bir Twilio numarası, her iki özelliğe de sahipse hem SMS hem de [Sesli Arama](/tr/plugins/voice-call) için kullanılabilir. SMS Webhook'u ve Ses Webhook'u Twilio'da ayrı ayrı yapılandırılır ve ayrı Gateway yolları kullanır; bu sayfa yalnızca SMS Webhook'unu kapsar.

## Hızlı Kurulum

<Steps>
  <Step title="Plugin'i kurun">
    ```bash
    openclaw plugins install @openclaw/sms
    ```
  </Step>
  <Step title="Bir Twilio göndereni oluşturun veya seçin">
    Twilio'da **Phone Numbers > Manage > Active numbers** bölümünü açın ve SMS özellikli bir numara seçin. Şunları kaydedin:

    - Account SID, örneğin `ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
    - Auth Token
    - Gönderen telefon numarası, örneğin `+15551234567`

    Sabit bir gönderen numarası yerine Messaging Service kullanıyorsanız Messaging Service SID'yi kaydedin; örneğin `MGxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`.

  </Step>

  <Step title="SMS kanalını yapılandırın">

Bunu `sms.patch.json5` olarak kaydedin ve yer tutucuları değiştirin:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

Uygulayın:

```bash
openclaw config patch --file ./sms.patch.json5 --dry-run
openclaw config patch --file ./sms.patch.json5
```

  </Step>

  <Step title="Twilio'yu Gateway Webhook'una yönlendirin">
    Twilio telefon numarası ayarlarında **Messaging** bölümünü açın ve **A message comes in** değerini şuna ayarlayın:

```text
https://gateway.example.com/webhooks/sms
```

    HTTP `POST` kullanın. Varsayılan yerel yol `/webhooks/sms` şeklindedir; farklı bir rota gerekiyorsa `channels.sms.webhookPath` değerini değiştirin.

  </Step>

  <Step title="Tam SMS Webhook yolunu kullanıma açın">
    Herkese açık URL'niz, SMS yolunu Gateway işlemine yönlendirmelidir (varsayılan port `18789`). Yerel test için Tailscale Funnel kullanıyorsanız `/webhooks/sms` yolunu açıkça kullanıma açın:

```bash
tailscale funnel --bg --set-path /webhooks/sms http://127.0.0.1:<gateway-port>/webhooks/sms
tailscale funnel status
```

    Sesli Arama ve SMS ayrı Webhook yolları kullanır. Aynı Twilio numarası her ikisini de işliyorsa her iki rotayı da Twilio'da ve tünelinizde yapılandırılmış olarak tutun.

  </Step>

  <Step title="Gateway'i başlatın ve ilk göndereni onaylayın">

```bash
openclaw gateway
```

Twilio numarasına bir kısa mesaj gönderin. İlk mesaj bir eşleştirme isteği oluşturur. İsteği onaylayın:

```bash
openclaw pairing list sms
openclaw pairing approve sms <CODE>
```

    Eşleştirme kodlarının süresi 1 saat sonra dolar.

  </Step>
</Steps>

## Yapılandırma Örnekleri

Tüm anahtarlar `channels.sms` altında (ve hesap başına `channels.sms.accounts.<id>` altında) bulunur:

| Anahtar                                 | Varsayılan      | Amaç                                                                |
| --------------------------------------- | --------------- | ------------------------------------------------------------------- |
| `enabled`                               | `true`          | Kanalı/hesabı etkinleştirin veya devre dışı bırakın.                 |
| `accountSid`                            | —               | Twilio Account SID (`AC...`).                                       |
| `authToken`                             | —               | Twilio Auth Token; düz metin dizesi veya SecretRef.                  |
| `fromNumber`                            | —               | E.164 gönderen numarası.                                             |
| `messagingServiceSid`                   | —               | Hiçbir `fromNumber` çözümlenmediğinde kullanılan Messaging Service SID (`MG...`). |
| `defaultTo`                             | —               | Gönderme akışı açık bir hedef belirtmediğinde varsayılan hedef.      |
| `webhookPath`                           | `/webhooks/sms` | Gelen Twilio Webhook'ları için Gateway HTTP yolu.                    |
| `publicWebhookUrl`                      | —               | Twilio'da yapılandırılan herkese açık URL; imza doğrulaması için gereklidir. |
| `dangerouslyDisableSignatureValidation` | `false`         | `X-Twilio-Signature` denetimlerini atlayın; yalnızca yerel tünel testi içindir. |
| `dmPolicy`                              | `"pairing"`     | `pairing`, `allowlist`, `open` veya `disabled`.                      |
| `allowFrom`                             | `[]`            | E.164 biçiminde izin verilen gönderen numaraları veya `dmPolicy: "open"` ile `"*"`. |
| `textChunkLimit`                        | `1500`          | Giden SMS parçası başına azami karakter sayısı.                      |
| `accounts`, `defaultAccount`            | —               | Çok hesaplı eşleme ve varsayılan hesap kimliği.                      |

### Yapılandırma dosyası

Kanal tanımının Gateway yapılandırmasıyla birlikte taşınmasını istediğinizde yapılandırma dosyasıyla kurulumu kullanın:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

### Ortam değişkenleri

Ortam değişkenleri yalnızca varsayılan hesaba uygulanır; yapılandırma değerleri ortam değerlerinden önceliklidir.

| Değişken                                        | Eşlendiği değer                                     |
| ----------------------------------------------- | -------------------------------------------------- |
| `TWILIO_ACCOUNT_SID`                            | `accountSid`                                       |
| `TWILIO_AUTH_TOKEN`                             | `authToken`                                        |
| `TWILIO_PHONE_NUMBER` (`TWILIO_SMS_FROM` diğer adıyla) | `fromNumber`                                       |
| `TWILIO_MESSAGING_SERVICE_SID`                  | `messagingServiceSid`                              |
| `SMS_PUBLIC_WEBHOOK_URL`                        | `publicWebhookUrl`                                 |
| `SMS_WEBHOOK_PATH`                              | `webhookPath`                                      |
| `SMS_ALLOWED_USERS`                             | `allowFrom` (virgülle ayrılmış)                      |
| `SMS_TEXT_CHUNK_LIMIT`                          | `textChunkLimit`                                   |
| `SMS_DANGEROUSLY_DISABLE_SIGNATURE_VALIDATION`  | `dangerouslyDisableSignatureValidation` (`"true"`) |

```bash
export TWILIO_ACCOUNT_SID="ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
export TWILIO_AUTH_TOKEN="<twilio-auth-token>"
export TWILIO_PHONE_NUMBER="+15551234567"
export SMS_PUBLIC_WEBHOOK_URL="https://gateway.example.com/webhooks/sms"
```

Ardından kanalı yapılandırmada etkinleştirin:

```json5
{
  channels: {
    sms: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

### SecretRef kimlik doğrulama belirteci

`authToken` bir SecretRef (`source: "env" | "file" | "exec"`) olabilir. Gateway'in düz metin yapılandırmayı depolamak yerine Twilio Auth Token'ı OpenClaw gizli bilgiler çalışma zamanından çözümlemesi gerektiğinde bunu kullanın:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: { source: "env", provider: "default", id: "TWILIO_AUTH_TOKEN" },
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

Başvurulan ortam değişkeni veya gizli bilgi sağlayıcısı Gateway çalışma zamanı tarafından görülebilmelidir. Ana makine ortam değişkenlerini değiştirdikten sonra yönetilen Gateway işlemlerini yeniden başlatın.

### Messaging Service göndereni

Twilio'nun göndereni bir Messaging Service aracılığıyla seçmesi gerektiğinde `fromNumber` yerine `messagingServiceSid` kullanın:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      messagingServiceSid: "MGxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

Yapılandırma ve ortam çözümlemesinden sonra hem `fromNumber` hem de `messagingServiceSid` mevcutsa `fromNumber` kullanılır.

### Varsayılan giden hedef

Bir gönderme akışı açık bir hedef belirtmediğinde otomasyon veya aracı tarafından başlatılan teslimatın varsayılan bir hedefi olması gerekiyorsa `defaultTo` değerini ayarlayın:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      defaultTo: "+15557654321",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
    },
  },
}
```

## Erişim denetimi

`channels.sms.dmPolicy` doğrudan SMS erişimini denetler:

- `pairing` (varsayılan): bilinmeyen gönderenler bir eşleştirme kodu alır; `openclaw pairing approve sms <CODE>` ile onaylayın.
- `allowlist`: yalnızca `allowFrom` içindeki gönderenler işlenir. Boş bir `allowFrom` her göndereni reddeder (Gateway bir başlangıç uyarısı günlüğe kaydeder).
- `open`: yapılandırma doğrulaması, `allowFrom` değerinin `"*"` içermesini gerektirir. Joker karakter olmadan yalnızca listelenen numaralar sohbet edebilir.
- `disabled`: gelen tüm DM'ler bırakılır.

`allowFrom` girdileri `+15551234567` gibi E.164 telefon numaraları olmalıdır. `sms:` ve `twilio-sms:` ön ekleri kabul edilir ve normalleştirilir. Özel bir asistan için açık telefon numaralarıyla `dmPolicy: "allowlist"` kullanmayı tercih edin:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "allowlist",
      allowFrom: ["+15557654321"],
    },
  },
}
```

## SMS gönderme

SMS kanalı seçildiğinde hedefler, yalın E.164 numaralarını veya `sms:` ön ekini kabul eder:

```bash
openclaw message send --channel sms --target sms:+15551234567 --message "hello"
```

Kanal seçimi örtük olduğunda `twilio-sms:` ön eki, iMessage'ın kendi hedefleri için operatör SMS teslimatını seçmek üzere kullandığı `sms:` hizmet ön ekini devralmadan bu kanalı seçer:

```bash
openclaw message send --target twilio-sms:+15551234567 --message "hello"
```

CLI açık bir `--target` gerektirir. `defaultTo`, hedefin kanal yapılandırmasından çözümlenebildiği otomasyon ve aracı tarafından başlatılan teslimat yolları içindir.

Gelen SMS görüşmelerine verilen aracı yanıtları, yapılandırılmış Twilio göndericisi üzerinden otomatik olarak gönderene geri iletilir.

SMS çıktısı düz metindir. OpenClaw; markdown biçimlendirmesini kaldırır, çitle çevrili kod bloklarını düzleştirir, bağlantıları `label (url)` olarak yeniden yazar ve uzun yanıtları Twilio üzerinden göndermeden önce en fazla `textChunkLimit` karakterlik (varsayılan 1500) parçalara böler.

## Kurulumu Doğrulama

Gateway başladıktan sonra:

1. Gateway günlüğünde SMS Webhook yolunun gösterildiğini doğrulayın.
2. Twilio tarafında bir yoklama çalıştırın (yapılandırılmış Twilio Webhook URL'sini/yöntemini ve son gelen ileti hatalarını denetler):

```bash
openclaw channels capabilities --channel sms
openclaw channels status --channel sms --probe --json
```

3. Telefonunuzdan Twilio numarasına bir SMS gönderin.
4. `openclaw pairing list sms` komutunu çalıştırın.
5. Eşleştirme kodunu `openclaw pairing approve sms <CODE>` ile onaylayın.
6. Başka bir SMS gönderin ve aracının yanıt verdiğini doğrulayın.

Yalnızca giden ileti testi için şunu kullanın:

```bash
openclaw message send --channel sms --target sms:+15557654321 --message "OpenClaw SMS test"
```

### macOS iMessage/SMS üzerinden uçtan uca test

Messages üzerinden operatör SMS'i gönderebilen bir Mac'te, telefonunuza dokunmadan gönderici tarafını yönetmek için `imsg` kullanabilirsiniz:

```bash
imsg send --to "+15551234567" --service sms --text "OpenClaw SMS E2E $(date -u +%Y%m%dT%H%M%SZ)" --json
openclaw pairing list sms
openclaw pairing approve sms <CODE>
imsg send --to "+15551234567" --service sms --text "reply exactly SMS pong" --json
```

İlk ileti bir eşleştirme isteği oluşturmalıdır. İkinci ileti, aracı yanıtını Twilio üzerinden almalıdır.

## Webhook güvenliği

OpenClaw varsayılan olarak `X-Twilio-Signature` değerini `publicWebhookUrl` ve `authToken` kullanarak doğrular. `publicWebhookUrl` değerinin uç nokta bölümünü; şema, ana bilgisayar, yol ve sorgu dizesi dâhil olmak üzere Twilio'da yapılandırılan URL ile bayt bayt aynı tutun. OpenClaw, Twilio'nun gerektirdiği şekilde Twilio [bağlantı geçersiz kılma](https://www.twilio.com/docs/usage/webhooks/webhooks-connection-overrides) parçalarını (`#...`) imza hesaplamasının dışında bırakır.

Webhook yolu, imza doğrulamasından bağımsız olarak şunları da uygular:

- Yalnızca `POST`.
- SMS hesabı, Webhook yolu ve çözümlenmiş istemci adresi başına dakikada 300 istekten oluşan başarısız istek bütçesi. Tüm istekler bu bütçeye dâhildir ancak HTTP 429 yalnızca bir istek gövde ayrıştırma, Twilio doğrulaması veya AccountSid eşleştirmesinde başarısız olduktan sonra uygulanır.
- Bu denetimler geçildikten sonra SMS hesabı, Webhook yolu ve çözümlenmiş istemci adresi başına dakikada 30 kabul edilmiş geri çağırmadan oluşan işlenebilir geri çağırma hız sınırı (bunun üzerinde HTTP 429). İmza doğrulaması devre dışıysa bu 30/dakika sınırı, kimliği doğrulanmamış işleme üst sınırıdır.
- İstemci adresleri, paylaşılan Gateway güvenilir proxy kuralları üzerinden çözümlenir. `gateway.trustedProxies`, Twilio geri çağırmalarını ileten ters proxy'yi içeriyorsa OpenClaw bu sınırları iletilen istemci adresine göre belirler; aksi takdirde doğrudan soket adresine geri döner.
- Yükteki `AccountSid`, yapılandırılmış `accountSid` ile eşleşmelidir (aksi takdirde HTTP 403).
- Yeniden oynatılan `MessageSid` değerlerinin yinelenenleri 10 dakika boyunca ayıklanır.
- Her SMS hesabının yeniden oynatma önbelleği en fazla 10.000 etkin ileti SID'sini saklar. Tüm yuvalar etkin olduğunda, o hesaba yönelik yeni Webhook'lar en eski yuvanın süresi dolana kadar HTTP 429 ve bir `Retry-After` üst bilgisiyle kapalı biçimde başarısız olur.
- 32 KB üzerindeki istek gövdeleri reddedilir.

Twilio varsayılan olarak HTTP 429 isteklerini yeniden denemez ve `Retry-After` desteğini belgelememektedir. `#rp=4xx` ve `#rp=all` bağlantı geçersiz kılmaları 4xx yeniden denemelerini etkinleştirir ancak Twilio tüm yeniden deneme işlemini 15 saniyeyle sınırlar; bu nedenle yeniden denemeler, bir yeniden oynatma önbelleği yuvasının süresi dolmadan önce tamamlanabilir. Başarısız teslimatları başka bir işleyicinin alması gerekiyorsa bir yedek URL yapılandırın; 429 yanıtını güvenilir geri basınç olarak değil, kapalı biçimde başarısız olan bir ret olarak değerlendirin.

Yalnızca yerel tünel testi için şunu ayarlayabilirsiniz:

```json5
{
  channels: {
    sms: {
      dangerouslyDisableSignatureValidation: true,
    },
  },
}
```

Herkese açık bir Gateway'de devre dışı bırakılmış imza doğrulaması kullanmayın.

## Çok hesaplı yapılandırma

Birden fazla Twilio numarası işletiyorsanız `accounts` kullanın:

```json5
{
  channels: {
    sms: {
      accounts: {
        support: {
          enabled: true,
          accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
          authToken: "twilio-auth-token",
          fromNumber: "+15551234567",
          publicWebhookUrl: "https://gateway.example.com/webhooks/sms/support",
          webhookPath: "/webhooks/sms/support",
          dmPolicy: "allowlist",
          allowFrom: ["+15557654321"],
        },
      },
    },
  },
}
```

Her hesap ayrı bir `webhookPath` kullanmalıdır; Gateway, yolu zaten başka bir hesaba ait olan bir Webhook yolunu kaydetmeyi reddeder. `TWILIO_*`/`SMS_*` ortam değişkeni geri dönüşleri yalnızca varsayılan hesaba uygulanır; bunun hangi hesap olduğunu değiştirmek için `defaultAccount` ayarlayın.

## Sorun Giderme

### Twilio 403 döndürüyor veya OpenClaw Webhook'u reddediyor

`publicWebhookUrl` değerinin şema, ana bilgisayar, yol ve sorgu dizesi dâhil olmak üzere Twilio'da yapılandırılan URL ile tam olarak eşleştiğini denetleyin. Twilio herkese açık URL dizesini imzalar; bu nedenle proxy yeniden yazımları ve alternatif ana bilgisayar adları imza doğrulamasını bozabilir.

`Invalid account` içeren bir 403 yanıtı, gelen yükün `AccountSid` değerinin yapılandırılmış `accountSid` ile eşleşmediği anlamına gelir; Webhook'un numaranın sahibi olan hesaba yönlendirildiğini denetleyin.

### Eşleştirme isteği görünmüyor

Twilio numarasının **Messaging** Webhook URL'sini ve yöntemini denetleyin. SMS Webhook URL'sine yönelmeli ve `POST` kullanmalıdır. Ayrıca Gateway'e herkese açık internetten veya tüneliniz üzerinden erişilebildiğini doğrulayın.

Twilio ileti günlüğünde `11200` hatası gösteriliyorsa Twilio gelen SMS'i kabul etmiş ancak Webhook'unuza ulaşamamıştır. Şunları denetleyin:

- Twilio **Messaging > A message comes in**, `publicWebhookUrl` adresine yöneliyor.
- Yöntem `POST`.
- Tünel veya ters proxy tam `webhookPath` yolunu kullanıma açıyor; Tailscale Funnel için `tailscale funnel status` çalıştırın ve `/webhooks/sms` değerinin listelendiğini doğrulayın.
- `publicWebhookUrl`, Twilio'nun gönderdiği şema, ana bilgisayar, yol ve sorgu dizesinin aynısını kullanıyor; böylece imza doğrulaması imzalanmış URL'yi yeniden oluşturabiliyor.

`openclaw channels status --channel sms --probe`, hem eşleşmeyen Twilio Webhook ayarlarını hem de son `11200` hatalarını gösterir.

### Giden iletiler gönderilemiyor

`accountSid`, `authToken` ve `fromNumber` ya da `messagingServiceSid` değerlerinden birinin çözümlendiğini doğrulayın. Deneme sürümü bir Twilio hesabı kullanıyorsanız giden SMS'in gönderilebilmesi için hedef numaranın önce Twilio'da doğrulanması gerekebilir.

### İletiler geliyor ancak aracı yanıt vermiyor

`dmPolicy` ve `allowFrom` değerlerini denetleyin. Varsayılan `pairing` ilkesi kullanıldığında, normal aracı turlarının işlenebilmesi için gönderenin onaylanmış olması gerekir.
