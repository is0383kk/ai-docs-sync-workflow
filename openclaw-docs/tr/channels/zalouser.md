---
read_when:
    - OpenClaw için Zalo Personal'ı ayarlama
    - Zalo Personal oturum açma veya mesaj akışında hata ayıklama
summary: Yerel zca-js (QR ile oturum açma) üzerinden Zalo kişisel hesap desteği, özellikleri ve yapılandırması
title: Zalo kişisel
x-i18n:
    generated_at: "2026-07-26T23:52:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09cecad1a9a5b34b932c5e68e2b3164b360fb6af1dcd2fd5b5979d1b2a1bd62b
    source_path: channels/zalouser.md
    workflow: 16
---

Durum: deneysel. Bu entegrasyon, harici bir CLI ikili dosyası olmadan, işlem içinde yerel `zca-js` aracılığıyla bir **kişisel Zalo hesabını** otomatikleştirir.

<Warning>
Bu resmî olmayan bir entegrasyondur ve hesabın askıya alınmasına veya yasaklanmasına yol açabilir. Riski size ait olmak üzere kullanın.
</Warning>

## Kurulum

Zalo Personal, çekirdekle birlikte sunulmayan resmî bir harici plugindir. Kullanmadan önce yükleyin:

```bash
openclaw plugins install @openclaw/zalouser
```

- Bir sürümü sabitleyin: `openclaw plugins install @openclaw/zalouser@<version>`
- Kaynak kod deposundan: `openclaw plugins install ./path/to/local/zalouser-plugin`
- Ayrıntılar: [Pluginler](/tr/tools/plugin)

## Hızlı kurulum

1. Plugini yükleyin (yukarıda).
2. Oturum açın (QR ile, Gateway makinesinde):
   - `openclaw channels login --channel zalouser`
   - QR kodunu Zalo mobil uygulamasıyla tarayın.
3. Kanalı etkinleştirin:

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

4. Gateway'i yeniden başlatın (veya kurulumu tamamlayın).
5. DM erişimi varsayılan olarak eşleştirme kullanır; ilk iletişimde eşleştirme kodunu onaylayın.

## Nedir?

- Tamamen `zca-js` kitaplığı aracılığıyla işlem içinde çalışır (harici `zca`/`openzca` ikili dosyası yoktur).
- Gelen mesajları almak için yerel olay dinleyicilerini (`message`, `error`) kullanır.
- Yanıtları doğrudan JS API üzerinden gönderir (metin/medya/bağlantı).
- Zalo Bot API'nin kullanılamadığı "kişisel hesap" kullanım durumları için tasarlanmıştır.

## Adlandırma

Kanal kimliği, bunun **kişisel bir Zalo kullanıcı hesabını** (resmî olmayan şekilde) otomatikleştirdiğini açıkça belirtmek için `zalouser` olarak belirlenmiştir. `zalo`, gelecekteki olası bir resmî Zalo API entegrasyonu için ayrılmıştır.

## Kimlikleri bulma (dizin)

```bash
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "work"
```

## Sınırlar

- Giden metin, 2000 karakterlik parçalara bölünür (Zalo istemci sınırı).
- Akış desteklenmez.
- Tamamlanan gelen mesaj kimlikleri 30 gün boyunca saklanır ve hesap başına en son 1000 kayıtla sınırlandırılır.

## Gelen mesaj dayanıklılığı

OpenClaw, her ham `zca-js` mesaj geri çağrısını işlemeden önce saklar. Bekleyen mesajlar bir Gateway yeniden başlatmasından sonra hesap kuyruğundan devam eder ve işleme, doğrudan sohbet veya grup başına sıralı kalır.

`zca-js` soket dinleyicisi, teslimat onayı sunmaz veya yeniden bağlandıktan sonra eski mesajları otomatik olarak yeniden oynatmaz. Bu nedenle dayanıklı kuyruk, bir geri çağrı OpenClaw'a ulaştıktan sonraki yerel çökme aralığını korur; soketin hiç teslim etmediği bir mesajı kurtaramaz. Yeniden oynatma mezar taşları, çoğunlukla aynı Zalo mesaj kimliğine sahip tekrarlanan bir geri çağrıya karşı koruma sağlar.

## Erişim denetimi (DM'ler)

`channels.zalouser.dmPolicy`: `pairing | allowlist | open | disabled` (varsayılan: `pairing`).

`channels.zalouser.allowFrom`, kararlı Zalo kullanıcı kimliklerini kullanmalıdır. Ayrıca statik gönderen erişim gruplarına (`accessGroup:<name>`) başvurabilir. Etkileşimli kurulum sırasında girilen adlar, pluginin işlem içi kişi araması kullanılarak kimliklere çözümlenebilir.

Yapılandırmada ham bir ad kalırsa başlangıçta yalnızca `channels.zalouser.dangerouslyAllowNameMatching: true` etkinleştirildiğinde çözümlenir. Bu açık onay olmadan çalışma zamanı gönderen kontrolleri yalnızca kimlikleri kullanır ve ham adlar yetkilendirme sırasında yok sayılır.

Şu yollarla onaylayın:

- `openclaw pairing list zalouser`
- `openclaw pairing approve zalouser <code>`

## Grup erişimi (isteğe bağlı)

- Varsayılan: `channels.zalouser.groupPolicy = "allowlist"` (gruplar açık bir izin listesi girdisi gerektirir).
- Tüm grupları açın: `channels.zalouser.groupPolicy = "open"`.
- Tüm grupları engelleyin: `channels.zalouser.groupPolicy = "disabled"`.
- `groupPolicy = "allowlist"` ile:
  - `channels.zalouser.groups` anahtarları kararlı grup kimlikleri olmalıdır; adlar yalnızca `channels.zalouser.dangerouslyAllowNameMatching: true` etkinleştirildiğinde başlangıçta kimliklere çözümlenir.
  - `channels.zalouser.groupAllowFrom`, izin verilen gruplardaki hangi gönderenlerin botu tetikleyebileceğini denetler; statik gönderen erişim gruplarına `accessGroup:<name>` ile başvurulabilir.
- Yapılandırma sihirbazı, grup izin listelerini sorabilir.
- Grup izin listesi eşleştirmesi varsayılan olarak yalnızca kimlikleri kullanır. `channels.zalouser.dangerouslyAllowNameMatching: true` etkinleştirilmediği sürece çözümlenmemiş adlar yetkilendirme sırasında yok sayılır.
- `channels.zalouser.dangerouslyAllowNameMatching: true`, değiştirilebilir başlangıç adı çözümlemesini ve çalışma zamanı grup adı eşleştirmesini yeniden etkinleştiren acil durum uyumluluk modudur.
- `groupAllowFrom`, normal grup mesajları için `allowFrom` değerine **geri dönmez**: izin listesindeki bir grupta bunu boş bırakmak, grubu tüm gönderenlere açar. Yetkili denetim komutları (örneğin `/new`) istisnadır; komut gönderen kontrolleri, `groupAllowFrom` boş olduğunda `allowFrom` değerine geri döner.

Örnek:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["1471383327500481391"],
      groups: {
        "123456789": { enabled: true },
        "Work Chat": { enabled: true },
      },
    },
  },
}
```

<Note>
`channels.zalouser.groups.<id>.allow` eski bir alan adıdır; güncel yapılandırma `enabled` kullanır. `openclaw doctor --fix`, `allow` değerini otomatik olarak `enabled` biçimine taşır.
</Note>

### Grup bahsetme geçidi

- `channels.zalouser.groups.<group>.requireMention`, grup yanıtlarının bahsetme gerektirip gerektirmediğini denetler.
- Çözümleme sırası: grup kimliği -> `group:<id>` diğer adı -> grup adı/kısa adı (ada dayalı adaylar yalnızca `dangerouslyAllowNameMatching: true` olduğunda uygulanır) -> `*` -> varsayılan (`true`).
- Hem izin listesindeki gruplara hem de açık grup moduna uygulanır.
- Bir bot mesajını alıntılamak, grup etkinleştirmesi için örtük bir bahsetme sayılır.
- Yetkili denetim komutları (örneğin `/new`) bahsetme geçidini atlayabilir.
- Bir grup mesajı bahsetme gerektiği için atlandığında OpenClaw, mesajı bekleyen grup geçmişi olarak saklar ve bir sonraki işlenen grup mesajına ekler.
- Grup geçmişi sınırı: `channels.zalouser.historyLimit`, ardından `messages.groupChat.historyLimit`, ardından `50` yedek değeri.

Örnek:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groups: {
        "*": { enabled: true, requireMention: true },
        "Work Chat": { enabled: true, requireMention: false },
      },
    },
  },
}
```

## Çoklu hesap

Hesaplar, OpenClaw durumundaki `zalouser` profilleriyle eşlenir. Örnek:

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      defaultAccount: "default",
      accounts: {
        work: { enabled: true, profile: "work" },
      },
    },
  },
}
```

## Ortam değişkenleri

Profil seçimi ortam değişkenlerinden de gelebilir:

| Değişken                | Amaç                                                                    |
| ------------------ | -------------------------------------------------------------------------- |
| `ZALOUSER_PROFILE` | Kanal veya hesap yapılandırmasında `profile` ayarlanmadığında kullanılacak profil adı. |
| `ZCA_PROFILE`      | Yalnızca `ZALOUSER_PROFILE` ayarlanmadığında kullanılan eski yedek değer.             |

Profil adları, OpenClaw durumunda kayıtlı Zalo oturum açma kimlik bilgilerini seçer. Çözümleme sırası:

1. Yapılandırmadaki açık `profile`.
2. `ZALOUSER_PROFILE`.
3. `ZCA_PROFILE`.
4. Varsayılan olmayan hesaplar için hesap kimliği veya varsayılan hesap için `default`.

Çoklu hesap kurulumlarında, tek bir ortam değişkeninin birden fazla hesabın aynı oturum açma oturumunu paylaşmasına yol açmaması için yapılandırmadaki her hesapta `profile` ayarlamayı tercih edin.

## Yazma göstergesi, tepkiler ve teslimat onayları

- OpenClaw, bir yanıtı göndermeden önce yazma olayı gönderir (mümkün olan en iyi şekilde).
- Mesaj tepkisi eylemi `react`, kanal eylemlerindeki `zalouser` için desteklenir.
  - Bir mesajdan belirli bir tepki emojisini kaldırmak için `remove: true` kullanın.
  - Tepki semantiği: [Tepkiler](/tr/tools/reactions)
- OpenClaw, olay meta verilerini içeren gelen mesajlar için teslim edildi + görüldü onayları gönderir (mümkün olan en iyi şekilde).

## Sorun giderme

**Oturum açık kalmıyor:**

- `openclaw channels status --probe`
- Yeniden oturum açın: `openclaw channels logout --channel zalouser && openclaw channels login --channel zalouser`

**İzin listesi/grup adı çözümlenmedi:**

- `allowFrom`/`groupAllowFrom` içinde sayısal kimlikler, `groups` içinde ise kararlı grup kimlikleri kullanın. Tam arkadaş/grup adlarını özellikle kullanmanız gerekiyorsa `channels.zalouser.dangerouslyAllowNameMatching: true` seçeneğini etkinleştirin.

**Eski bir harici `zca`/CLI tabanlı kurulumdan yükseltme yapıldı:**

- Harici `zca` işlemine ilişkin tüm varsayımları kaldırın; kanal artık harici bir CLI ikili dosyası olmadan tamamen `zca-js` aracılığıyla işlem içinde çalışır.

## İlgili içerikler

- [Kanallara Genel Bakış](/tr/channels) - desteklenen tüm kanallar
- [Eşleştirme](/tr/channels/pairing) - DM kimlik doğrulaması ve eşleştirme akışı
- [Gruplar](/tr/channels/groups) - grup sohbeti davranışı ve bahsetme geçidi
- [Kanal Yönlendirme](/tr/channels/channel-routing) - mesajlar için oturum yönlendirmesi
- [Güvenlik](/tr/gateway/security) - erişim modeli ve güçlendirme
