---
read_when:
    - Tlon/Urbit kanal özellikleri üzerinde çalışma
summary: Tlon/Urbit destek durumu, özellikleri ve yapılandırması
title: Tlon
x-i18n:
    generated_at: "2026-07-26T23:50:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d742628d6cf9aaf82d79a8d96b1685229905e9452c9fc4d3a494d2dee8d69943
    source_path: channels/tlon.md
    workflow: 16
---

Tlon, Urbit üzerine kurulmuş merkeziyetsiz bir mesajlaşma uygulamasıdır. OpenClaw, Urbit geminize bağlanır ve
DM'lere ve grup sohbeti mesajlarına yanıt verir. Grup yanıtları varsayılan olarak @ ile bahsetme gerektirir;
bunun üzerine yetkilendirme kuralları ve bir sahip onay akışı uygulanır.

Durum: paketle birlikte gelen plugin. DM'ler, grup bahsetmeleri, ileti dizileri, zengin metin, görsel yükleme/indirme ve bir
sahip onay sistemi desteklenir. Tepkiler ve anketler desteklenmez.

## Paketle birlikte gelen plugin

Tlon, güncel OpenClaw sürümleriyle birlikte gelir; paketlenmiş derlemeler ayrı bir kurulum gerektirmez.

Tlon'u içermeyen eski bir derlemede veya özel kurulumda npm'den yükleyin:

```bash
openclaw plugins install @openclaw/tlon
```

Güncel sürüm etiketini izlemek için yalnızca paket adını kullanın. Yalnızca tekrarlanabilir kurulumlar için bir sürümü (`@openclaw/tlon@x.y.z`)
sabitleyin.

Yerel bir çalışma kopyasından:

```bash
openclaw plugins install ./path/to/local/tlon-plugin
```

Ayrıntılar: [Plugin'ler](/tr/tools/plugin)

## Kurulum

```bash
openclaw channels add --channel tlon --ship ~sampel-palnet --url https://your-ship-host --code lidlut-tabwed-pillex-ridrup
```

Veya yapılandırmayı doğrudan düzenleyin:

```json5
{
  channels: {
    tlon: {
      enabled: true,
      ship: "~sampel-palnet",
      url: "https://your-ship-host",
      code: "lidlut-tabwed-pillex-ridrup",
      ownerShip: "~your-main-ship", // önerilen: geminiz, her zaman yetkilidir
    },
  },
}
```

Yapılandırmayı doğrudan düzenledikten sonra Gateway'i yeniden başlatın. Ardından bota DM gönderin veya bir grup
kanalında @ ile bottan bahsedin.

## Gelen iletilerin dayanıklılığı

OpenClaw, kabul edilen Tlon DM ve grup sohbeti olaylarını aracıya göndermeden önce kalıcı hâle getirir. Bekleyen veya yeniden denenebilir işlemler Gateway yeniden başlatıldığında korunur ve işler grup kanalı ya da doğrudan eş başına sıralı kalır. Kararlı Urbit mesaj kimlikleri de kuyruk kaydı veya saklanan tamamlanma kaydı mevcut olduğu sürece yeniden teslim edilen bir olayı engeller.

Kuyruktan aracıya geçiş sınırında teslimat en az bir kez gerçekleşir: devir sırasında oluşan bir çökme, bir işlemin yeniden yürütülmesine neden olabilir. Bu nedenle dış yan etkiler oluşturan aracı eylemleri, uygulanabilir olduğunda idempotent kalmalıdır.

## Özel/LAN gemileri

OpenClaw, SSRF koruması için özel/dahili ana bilgisayar adlarını ve IP aralıklarını varsayılan olarak engeller. Geminiz
özel bir ağda (localhost, LAN IP'si, dahili ana bilgisayar adı) çalışıyorsa açıkça izin verin:

```json5
{
  channels: {
    tlon: {
      url: "http://localhost:8080",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
    },
  },
}
```

`http://localhost:8080`, `http://192.168.x.x:8080` ve
`http://my-ship.local:8080` gibi hedefler için geçerlidir. Bunu yalnızca güvendiğiniz bir gemi URL'si için etkinleştirin; ilgili hesabın
HTTP isteklerinde SSRF korumasını devre dışı bırakır.

<Note>
`channels.tlon.allowPrivateNetwork` (düz anahtar) kullanımdan kaldırılmıştır. `openclaw doctor --fix` bunu otomatik olarak
`channels.tlon.network.dangerouslyAllowPrivateNetwork` konumuna taşır.
</Note>

## Grup kanalları

Kanalları elle sabitleyin veya otomatik keşfi açın:

```json5
{
  channels: {
    tlon: {
      groupChannels: ["chat/~host-ship/general", "chat/~host-ship/support"],
      autoDiscoverChannels: true,
    },
  },
}
```

`autoDiscoverChannels`, yapılandırmada ayarlanmadığında varsayılan olarak `false` değerini alır; kurulum sihirbazı
istem için varsayılan olarak evet seçeneğini sunar ve `true` değerini açıkça yazar. Bu özellik açıkken OpenClaw, başlangıçta katılınan grupları sorgular,
grup davetleri kabul edildikçe yeni kanalları izler ve her 2 dakikada bir yeniden kontrol eder.

## Erişim denetimi

DM izin listesi (boş = gönderen `ownerShip` olmadığı sürece hiçbir DM'ye izin verilmez):

```json5
{
  channels: {
    tlon: {
      dmAllowlist: ["~zod", "~nec"],
    },
  },
}
```

Grup yetkilendirmesi kanal başına varsayılan olarak `restricted` değerini kullanır. Bir
temel değer için `defaultAuthorizedShips` ayarını yapın ve kanal yuvası başına geçersiz kılın:

```json5
{
  channels: {
    tlon: {
      defaultAuthorizedShips: ["~zod"],
      authorization: {
        channelRules: {
          "chat/~host-ship/general": {
            mode: "restricted",
            allowedShips: ["~zod", "~nec"],
          },
          "chat/~host-ship/announcements": {
            mode: "open",
          },
        },
      },
    },
  },
}
```

Bot bir ileti dizisinde yanıt verdikten sonra, yeniden bahsedilmesi gerekmeden o ileti dizisindeki sonraki mesajlara
yanıt vermeyi sürdürür.

Bu takip yanıtlarında yeniden açıkça bahsedilmesini zorunlu kılmak için `channels.tlon.implicitMentions.threadParticipation: false` ayarını yapın.
Hesap geçersiz kılmaları `channels.tlon.accounts.<id>.implicitMentions` kullanır. Tlon şu anda
`replyToBot` veya `quotedBot` olgularını üretmediğinden bu bayrakların burada hiçbir etkisi yoktur.

## Sahip ve onay sistemi

```json5
{
  channels: {
    tlon: {
      ownerShip: "~your-main-ship",
    },
  },
}
```

Sahip gemi her yerde yetkilidir: DM davetleri her zaman otomatik kabul edilir, grup davetleri
her zaman otomatik kabul edilir ve kanal mesajları her zaman yetkilendirmeden geçer. Sahibin
`dmAllowlist`, `defaultAuthorizedShips` veya `groupInviteAllowlist` içinde bulunması gerekmez.

`ownerShip` ayarlandığında yetkisiz istekler yalnızca yok sayılmaz; bekleyen
bir onay kuyruğuna alınır ve sahibine DM gönderilir:

- `dmAllowlist` üzerinde olmayan gemilerden gelen DM istekleri
- Gönderenin yetkilendirmeyi geçemediği kanallardaki bahsetmeler
- `groupInviteAllowlist` üzerinde olmayan gemilerden gelen grup davetleri (otomatik kabul kapalıysa veya açık olmasına rağmen
  davet eden izin listesinde değilse)

Sahip, bir istek üzerinde işlem yapmak için DM ile yanıt verir:

| Sahip yanıtı                  | Etki                                               |
| ---------------------------- | ---------------------------------------------------- |
| `approve` / `deny` / `block` | En son bekleyen onay üzerinde işlem yapar             |
| `approve <id>` / `deny <id>` | Belirli bir onay üzerinde kimliğine göre işlem yapar                    |
| `block`                      | Ayrıca gemiyi yerel olarak engelleyerek yeniden bağlanmasını önler |
| `unblock ~ship`              | Yerel engellemeyi geri alır                              |
| `blocked`                    | Şu anda engellenmiş gemileri listeler                        |
| `pending`                    | Bekleyen onay isteklerini listeler                      |

`ownerShip` yapılandırılmadığında yetkisiz DM'ler ve kanal bahsetmeleri yalnızca yok sayılır ve günlüğe kaydedilir;
onay istemi gösterilmez.

## Otomatik kabul ayarları

`dmAllowlist` üzerinde bulunan gemilerden gelen DM davetlerini otomatik kabul edin (bu bayraktan bağımsız olarak
sahip her zaman otomatik kabul edilir):

```json5
{
  channels: {
    tlon: {
      autoAcceptDmInvites: true,
    },
  },
}
```

Bir izin listesindeki grup davetlerini otomatik kabul edin (güvenli biçimde reddeder: `autoAcceptGroupInvites: true` etkin ve
`groupInviteAllowlist` boş olduğunda sahip dışındaki hiçbir davet kabul edilmez):

```json5
{
  channels: {
    tlon: {
      autoAcceptGroupInvites: true,
      groupInviteAllowlist: ["~zod"],
    },
  },
}
```

## Urbit ayarlar deposu üzerinden çalışırken yeniden yükleme

Yukarıdaki ayarların çoğu (`dmAllowlist`, `groupInviteAllowlist`, `groupChannels`,
`defaultAuthorizedShips`, `autoDiscoverChannels`, `autoAcceptDmInvites`,
`autoAcceptGroupInvites`, `ownerShip`, `showModelSignature`) ilk çalıştırmada geminin
`%settings` aracısına (masa `moltbot`, demet `tlon`) yansıtılır ve ardından doğrudan buradan okunur;
bu nedenle bir Landscape istemcisi veya paketle birlikte gelen skill'in ayar komutlarıyla yapılan değişiklikler
Gateway yeniden başlatılmadan uygulanır. `channelRules` ve bekleyen onaylar da burada JSON olarak kalıcı hâle getirilir. Dosya
yapılandırması, ayarlar deposuna hiç yazılmamış değerlerin doğruluk kaynağı olmaya devam eder.

## Teslimat hedefleri (CLI/cron)

`openclaw message send` veya cron teslimatıyla kullanın:

- DM: `~sampel-palnet` veya `dm/~sampel-palnet`
- Grup: `chat/~host-ship/channel` veya `group:~host-ship/channel`

## Paketle birlikte gelen skill

Plugin, doğrudan Urbit işlemleri için bir CLI olan [`@tloncorp/tlon-skill`](https://github.com/tloncorp/tlon-skill) aracını paketle birlikte sunar;
plugin yüklendikten sonra otomatik olarak kullanılabilir:

- **Etkinlik**: bahsetmeler, yanıtlar, okunmamışlar
- **Kanallar**: listeleme, oluşturma, yeniden adlandırma
- **Kişiler**: profilleri listeleme/alma/güncelleme
- **Gruplar**: oluşturma, katılma, davet/istek akışları, roller
- **Kancalar**: kanal kancalarını yönetme
- **Mesajlar**: geçmiş, arama
- **DM'ler**: gönderme, tepki verme, kabul etme/reddetme
- **Gönderiler**: tepki verme, silme
- **Not Defteri**: günlük kanallarına gönderi yayımlama
- **Ayarlar**: yukarıdaki ayarlar deposu üzerinden plugin yapılandırmasını çalışırken yeniden yükleme

## Yetenekler

| Özellik         | Durum                                        |
| --------------- | --------------------------------------------- |
| Doğrudan mesajlar | Desteklenir                                     |
| Gruplar/kanallar | Desteklenir (varsayılan olarak bahsetme gerektirir)          |
| İleti dizileri         | Desteklenir (katıldıktan sonra yanıt vermeyi sürdürür) |
| Zengin metin       | Markdown, Tlon'un yerel biçimine dönüştürülür    |
| Görseller          | Gelenler indirilir, gidenler yüklenir         |
| Tepkiler       | Yalnızca [paketle birlikte gelen skill](#bundled-skill) üzerinden  |
| Anketler           | Desteklenmez                                 |
| Yerel komutlar | Varsayılan olarak yalnızca sahip tarafından kullanılabilir                         |

## Sorun giderme

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
```

Yaygın hatalar:

- **DM'ler yok sayılıyor**: gönderen `dmAllowlist` içinde değil ve onay akışı için `ownerShip` yapılandırılmamış.
- **Grup mesajları yok sayılıyor**: kanal keşfedilmemiş/sabitlenmemiş veya gönderen yetkilendirmeyi geçemiyor ve
  onayı kuyruğa almak için `ownerShip` bulunmuyor.
- **Bağlantı hataları**: gemi URL'sine erişilebildiğini kontrol edin; yerel gemiler için
  `network.dangerouslyAllowPrivateNetwork` ayarını yapın.
- **Kimlik doğrulama hataları**: giriş kodları dönüşümlüdür; güncel kodu geminizden kopyalayın.

## Yapılandırma referansı

Tam yapılandırma: [Yapılandırma](/tr/gateway/configuration)

| Anahtar                                                    | Anlam                                                        |
| ------------------------------------------------------ | -------------------------------------------------------------- |
| `channels.tlon.enabled`                                | Kanal başlatmayı etkinleştirir/devre dışı bırakır.                                |
| `channels.tlon.ship`                                   | Botun Urbit gemi adı (ör. `~sampel-palnet`).                 |
| `channels.tlon.url`                                    | Gemi URL'si (ör. `https://sampel-palnet.tlon.network`).          |
| `channels.tlon.code`                                   | Gemi giriş kodu.                                               |
| `channels.tlon.network.dangerouslyAllowPrivateNetwork` | Localhost/LAN gemi URL'lerine izin verir (SSRF için açıkça izin verme).                   |
| `channels.tlon.ownerShip`                              | Sahip gemi: her zaman yetkilidir, onay isteklerini alır.     |
| `channels.tlon.dmAllowlist`                            | DM göndermesine izin verilen gemiler (boş = sahip dışında hiçbiri).              |
| `channels.tlon.autoAcceptDmInvites`                    | `dmAllowlist` içindeki gemilerden gelen DM'leri otomatik kabul eder.                   |
| `channels.tlon.autoAcceptGroupInvites`                 | `groupInviteAllowlist` içindeki grup davetlerini otomatik kabul eder.         |
| `channels.tlon.groupInviteAllowlist`                   | Grup davetleri otomatik kabul edilen gemiler.                   |
| `channels.tlon.autoDiscoverChannels`                   | Katılınan grup kanallarını otomatik keşfeder (varsayılan: `false`).        |
| `channels.tlon.implicitMentions.threadParticipation`   | Katılım sağlanan ileti dizilerindeki takip mesajlarının bahsetme denetimini atlamasına izin verir.      |
| `channels.tlon.groupChannels`                          | Elle sabitlenen kanal yuvaları.                                 |
| `channels.tlon.defaultAuthorizedShips`                 | Tüm kanallar için yetkilendirilen gemiler (hiçbir kural eşleşmediğinde kullanılır). |
| `channels.tlon.authorization.channelRules`             | Kanal yuvası başına kimlik doğrulama modu + izin listesi.                        |
| `channels.tlon.showModelSignature`                     | Yanıtlara `_[Generated by <model>]_` ekler.                  |
| `channels.tlon.responsePrefix`                         | Giden yanıtların başına eklenen statik önek.                   |
| `channels.tlon.accounts.<id>`                          | Ek adlandırılmış hesaplar (çok gemili kurulumlar).                 |

## Notlar

- Bot ilgili ileti dizisine zaten katılmadıysa grup yanıtları bir @ bahsi (ör. `~your-bot-ship`) gerektirir.
- İleti dizisi yanıtları ilgili ileti dizisine gönderilir; ayrıca ileti dizisi bağlamındaki son 10 mesaj
  agent için başa eklenerek bota iletilir.
- Zengin metin (kalın, italik, kod, başlıklar, listeler) Tlon'un yerel biçimine dönüştürülür.
- Kanal özeti isteyen bir gelen mesajın (örneğin "bu kanalı
  özetle") gönderilmesi, normal yanıt akışı yerine yerleşik geçmiş özetlemeyi tetikler.

## İlgili

- [Kanallara Genel Bakış](/tr/channels) — desteklenen tüm kanallar
- [Eşleştirme](/tr/channels/pairing) — DM kimlik doğrulaması ve eşleştirme akışı
- [Gruplar](/tr/channels/groups) — grup sohbeti davranışı ve bahsetme kısıtlaması
- [Kanal Yönlendirme](/tr/channels/channel-routing) — mesajlar için oturum yönlendirmesi
- [Güvenlik](/tr/gateway/security) — erişim modeli ve güvenliği güçlendirme
