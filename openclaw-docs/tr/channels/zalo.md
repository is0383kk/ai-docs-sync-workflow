---
read_when:
    - Zalo özellikleri veya webhook'lar üzerinde çalışma
summary: Zalo bot desteği durumu, yetenekleri ve yapılandırması
title: Zalo
x-i18n:
    generated_at: "2026-07-26T23:32:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f3e0bfe6003d3b2f38411fcc5a4e82266733b042693c7853d0b3c8a3864273c5
    source_path: channels/zalo.md
    workflow: 16
---

Durum: deneysel. Hem doğrudan mesajlar hem de grup sohbetleri uygulanmıştır; aşağıdaki [Yetenekler](#capabilities) tablosu, Zalo Bot Creator / Marketplace botlarında doğrulanmış davranışı yansıtır.

## Paketle birlikte gelen plugin

Zalo, güncel OpenClaw sürümlerinde paketle birlikte gelen bir plugin olarak sunulur; bu nedenle paketlenmiş derlemeler ayrı bir kurulum gerektirmez.

Daha eski bir derlemede veya Zalo'yu hariç tutan özel bir kurulumda npm paketini doğrudan yükleyin:

- Kurulum: `openclaw plugins install @openclaw/zalo`
- Sabitlenmiş sürüm: `openclaw plugins install @openclaw/zalo@2026.6.11`
- Yerel bir çalışma kopyasından: `openclaw plugins install ./path/to/local/zalo-plugin`
- Ayrıntılar: [Pluginler](/tr/tools/plugin)

## Hızlı kurulum

1. [https://bot.zaloplatforms.com](https://bot.zaloplatforms.com) adresinde bir bot token'ı oluşturun (oturum açın, bir bot oluşturun, ayarları yapılandırın). Token `numeric_id:secret`; Marketplace botlarında kullanılabilir çalışma zamanı token'ı botun karşılama mesajında görünebilir.
2. Token'ı yalnızca varsayılan hesap için `ZALO_BOT_TOKEN=...` ortam değişkeni olarak veya yapılandırmada ayarlayın.
3. Gateway'i yeniden başlatın.
4. İlk doğrudan mesaj iletişiminde eşleştirme kodunu onaylayın (varsayılan doğrudan mesaj politikası eşleştirmedir).

En küçük yapılandırma:

```json5
{
  channels: {
    zalo: {
      enabled: true,
      accounts: {
        default: {
          botToken: "12345689:abc-xyz",
          dmPolicy: "pairing",
        },
      },
    },
  },
}
```

Çoklu hesap: `channels.zalo.accounts.<id>` altına, her biri kendi `botToken`/`name` değerine sahip ek girdiler ekleyin. `channels.zalo.botToken` (`accounts` içermeyen düz biçim), eski tek hesaplı kısa gösterimdir; yeni yapılandırmalarda `accounts.<id>.*` biçimini tercih edin.

## Nedir?

Zalo, Vietnam odaklı bir mesajlaşma uygulamasıdır. Bot API'si, Gateway'in hem 1:1 görüşmeler hem de grup sohbetleri için bir bot çalıştırmasına ve yanıtları belirlenimsel olarak Zalo'ya yönlendirmesine olanak tanır (model hiçbir zaman kanalları seçmez).

Bu sayfa **Zalo Bot Creator / Marketplace botlarını** kapsar. **Zalo Official Account (OA) botları** farklı bir ürün yüzeyidir ve farklı davranabilir; bu sayfa bunları kapsamaz.

## Nasıl çalışır?

- Gelen mesajlar, medya yer tutucularıyla birlikte paylaşılan kanal zarfına normalleştirilir.
- Yanıtlar her zaman aynı Zalo sohbetine geri yönlendirilir; alıntılı yanıt kullanılmaz (`replyToMode` sabit olarak kapalıdır).
- Varsayılan olarak uzun yoklama (`getUpdates`) kullanılır; webhook modu `channels.zalo.webhookUrl` aracılığıyla kullanılabilir.
- Gruplarda botu tetiklemek için @bahsetme gerekir; bu, kanal başına yapılandırılamaz.

## Sınırlar

| Sınır                         | Değer                                                                    |
| ----------------------------- | ------------------------------------------------------------------------ |
| Giden metin parçası boyutu      | 2000 karakter (Zalo API sınırı)                                         |
| Medya boyutu (gelen/giden) | `channels.zalo.mediaMaxMb`, varsayılan `5` MB                               |
| Webhook istek gövdesi          | 1 MB, 30s okuma zaman aşımı                                                   |
| Webhook hız sınırı            | Yol+istemci IP'si başına 120 istek / 60s, ardından HTTP 429                     |
| Webhook yeniden oynatma mezar taşları     | 30 gün, hesap başına en fazla 20.000 tamamlanmış olay (mesaj kimliğine göre anahtarlanır) |

## Erişim denetimi

### Doğrudan mesajlar

- `channels.zalo.dmPolicy`: `pairing` (varsayılan) | `allowlist` | `open` | `disabled`.
- Eşleştirme: bilinmeyen gönderenler bir eşleştirme kodu alır; onaylanana kadar mesajlar yok sayılır. Kodların süresi 1 saat sonra dolar.
  - `openclaw pairing list zalo`
  - `openclaw pairing approve zalo <CODE>`
  - Ayrıntılar: [Eşleştirme](/tr/channels/pairing)
- `channels.zalo.allowFrom`, sayısal Zalo kullanıcı kimliklerini kabul eder (kullanıcı adı araması yoktur). `open`, `"*"` gerektirir.

### Gruplar

Grup sohbetleri plugin tarafından desteklenir (`chatTypes: ["direct", "group"]`) ve bahsetme ile grup politikası tarafından denetlenir:

- `channels.zalo.groupPolicy`: `open` | `allowlist` | `disabled`.
- `channels.zalo.groupAllowFrom`, gruplarda hangi gönderen kimliklerinin botu tetikleyebileceğini sınırlar; ayarlanmadığında `allowFrom` kullanılır.
- Varsayılan çözümleme: `channels.zalo` yapılandırıldığında, ayarlanmamış bir `groupPolicy` değeri `open` olarak çözümlenir. `channels.zalo` tamamen eksik olduğunda çalışma zamanı güvenli biçimde `allowlist` değerine kapanır.
- Gerçek kullanımda bildirilen sınırlama: bazı Marketplace botu kurulumlarında bot hiçbir şekilde bir gruba eklenememiştir. Bununla karşılaşırsanız botunuzun Zalo Bot Platform ayarlarını doğrulayın; bu, OpenClaw politikası değil, platform kaynaklı bir kısıtlamadır.

## Uzun yoklama ve webhook karşılaştırması

- Varsayılan: uzun yoklama (genel bir URL gerekmez).
- Webhook modu: `channels.zalo.webhookUrl` ve `channels.zalo.webhookSecret` değerlerini ayarlayın.
  - Webhook URL'si HTTPS kullanmalıdır.
  - Webhook gizli anahtarı 8-256 karakter olmalıdır.
  - Zalo, olayları sabit süreli karşılaştırmayla denetlenen bir `X-Bot-Api-Secret-Token` başlığıyla gönderir.
  - Gateway HTTP, webhook isteklerini `channels.zalo.webhookPath` konumunda işler (varsayılan olarak webhook URL'sinin yoludur).
  - İstekler `Content-Type: application/json` (veya bir `+json` medya türü) kullanmalıdır.
  - HTTP 200 yalnızca ham olay kalıcı olarak depolandıktan sonra döndürülür; depolama hataları HTTP 500 döndürür.
  - Zalo API belgelerine göre getUpdates yoklaması ile webhook birbirini dışlar.

## Desteklenen mesaj türleri

- Metin: tam destek, 2000 karakterlik parçalara bölünür.
- Medya: gelen/giden, `mediaMaxMb` ile sınırlandırılır.
- Tepkiler, ileti dizileri, anketler, yerel komutlar: plugin tarafından desteklenmez.
- Akış: plugin blok akışı yeteneğini bildirir ancak Zalo'nun özel giden kuyruğu/metin birleştirme ayarları yoktur (diğer bazı bölgesel kanalların aksine); bu kullanım durumunuz için önemliyse ortamınızdaki güncel davranışı doğrulayın.

## Yetenekler

| Özellik                  | Durum                            |
| ------------------------ | --------------------------------- |
| Doğrudan mesajlar          | Desteklenir                         |
| Gruplar                   | Desteklenir (bahsetme gerekli)         |
| Medya (gelen/giden) | Desteklenir, `mediaMaxMb` ile sınırlandırılır |
| Tepkiler                | Desteklenmez                     |
| İleti dizileri                  | Desteklenmez                     |
| Anketler                    | Desteklenmez                     |
| Yerel komutlar          | Desteklenmez                     |
| Yanıt / alıntı         | Kullanılmaz (sabit olarak kapalı)              |

## Teslim hedefleri (CLI/cron)

Hedef olarak bir sohbet kimliği kullanın:

```bash
openclaw message send --channel zalo --target 123456789 --message "hi"
```

## Sorun giderme

**Bot yanıt vermiyor:**

- Token'ı denetleyin: `openclaw channels status --probe`
- Gönderenin onaylandığını doğrulayın (eşleştirme veya `allowFrom`)
- Gateway günlüklerini denetleyin: `openclaw logs --follow`

**Webhook olayları almıyor:**

- Webhook URL'sinin HTTPS kullandığını doğrulayın
- Gizli anahtarın 8-256 karakter olduğunu doğrulayın
- Gateway HTTP uç noktasına yapılandırılmış yoldan erişilebildiğini doğrulayın
- getUpdates yoklamasının aynı anda çalışmadığını doğrulayın (birbirlerini dışlarlar)
- Ani bir istek yoğunluğu HTTP 429 döndürebilir (yol+IP başına 120 istek / 60s); bekleme süresini artırıp yeniden deneyin

## Yapılandırma referansı

Tam yapılandırma: [Yapılandırma](/tr/gateway/configuration)

| Ayar                                      | Açıklama                                       | Varsayılan               |
| -------------------------------------------- | ------------------------------------------------- | --------------------- |
| `channels.zalo.enabled`                      | Kanal başlatmayı etkinleştir/devre dışı bırak                    | `true`                |
| `channels.zalo.accounts.<id>.botToken`       | Zalo Bot Platform'dan alınan bot token'ı                  | -                     |
| `channels.zalo.accounts.<id>.tokenFile`      | Token'ı bir dosyadan oku (sembolik bağlantılar reddedilir)        | -                     |
| `channels.zalo.accounts.<id>.name`           | Görünen ad                                      | -                     |
| `channels.zalo.accounts.<id>.enabled`        | Bu hesabı etkinleştir/devre dışı bırak                       | `true`                |
| `channels.zalo.accounts.<id>.dmPolicy`       | Hesap başına doğrudan mesaj politikası                             | `pairing`             |
| `channels.zalo.accounts.<id>.allowFrom`      | Doğrudan mesaj izin listesi (kullanıcı kimlikleri)                           | -                     |
| `channels.zalo.accounts.<id>.groupPolicy`    | Hesap başına grup politikası                          | bkz. [Gruplar](#groups) |
| `channels.zalo.accounts.<id>.groupAllowFrom` | Grup gönderen izin listesi; `allowFrom` değerine geri döner | -                     |
| `channels.zalo.accounts.<id>.mediaMaxMb`     | Gelen/giden medya sınırı (MB)                   | `5`                   |
| `channels.zalo.accounts.<id>.webhookUrl`     | Webhook modunu etkinleştir (HTTPS gerekir)              | -                     |
| `channels.zalo.accounts.<id>.webhookSecret`  | Webhook gizli anahtarı (8-256 karakter)                      | -                     |
| `channels.zalo.accounts.<id>.webhookPath`    | Gateway HTTP sunucusundaki webhook yolu           | webhook URL yolu      |
| `channels.zalo.accounts.<id>.proxy`          | API istekleri için proxy URL'si                        | -                     |
| `channels.zalo.accounts.<id>.responsePrefix` | Giden yanıt ön eki geçersiz kılma değeri                 | -                     |
| `channels.zalo.defaultAccount`               | Birden fazla hesap yapılandırıldığında varsayılan hesap      | `default`             |

`channels.zalo.botToken`, `channels.zalo.dmPolicy` ve diğer düz üst düzey anahtarlar, yukarıdaki alanların eski tek hesaplı kısa gösterimidir; her iki biçim de desteklenir.

Ortam seçeneği: `ZALO_BOT_TOKEN=...` yalnızca varsayılan hesabın token'ını çözümler.

## İlgili

- [Kanallara Genel Bakış](/tr/channels) - desteklenen tüm kanallar
- [Eşleştirme](/tr/channels/pairing) - doğrudan mesaj kimlik doğrulaması ve eşleştirme akışı
- [Gruplar](/tr/channels/groups) - grup sohbeti davranışı ve bahsetme denetimi
- [Kanal Yönlendirme](/tr/channels/channel-routing) - mesajlar için oturum yönlendirmesi
- [Güvenlik](/tr/gateway/security) - erişim modeli ve sağlamlaştırma
