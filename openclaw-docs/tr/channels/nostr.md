---
read_when:
    - OpenClaw'ın Nostr üzerinden doğrudan mesaj almasını istiyorsunuz
    - Merkeziyetsiz mesajlaşmayı ayarlıyorsunuz
summary: NIP-04 şifreli mesajlar üzerinden Nostr DM kanalı
title: Nostr
x-i18n:
    generated_at: "2026-07-26T22:38:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 31fa283f706036a37795ddad71602058ba94388a9cb01044927c4bb2d83ba4a8
    source_path: channels/nostr.md
    workflow: 16
---

Nostr, OpenClaw'ın Nostr aktarıcıları üzerinden NIP-04 ile şifrelenmiş doğrudan mesajları alıp yanıtlamasını sağlayan, indirilebilir bir kanal pluginidir (`@openclaw/nostr`). Gateway başına bir hesap; yalnızca DM'ler.

## Kurulum

```bash
openclaw plugins install @openclaw/nostr
```

Güncel resmî sürüm etiketini izlemek için yalnızca paket belirtimini kullanın. Yalnızca yeniden üretilebilir bir kurulum gerektiğinde tam bir sürümü sabitleyin.

Yerel bir çalışma kopyasından (geliştirme iş akışları):

```bash
openclaw plugins install --link <path-to-local-nostr-plugin>
```

Pluginleri kurduktan veya etkinleştirdikten sonra Gateway'i yeniden başlatın. Plugin kurulduktan sonra ilk katılım (`openclaw onboard`) ve `openclaw channels add`, Nostr'ı paylaşılan kanal kataloğunda gösterir.

### Etkileşimsiz kurulum

```bash
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY" --relay-urls "wss://relay.damus.io,wss://relay.primal.net"
```

Anahtarı yapılandırmada saklamak yerine `NOSTR_PRIVATE_KEY` ortamda tutmak için `--use-env` kullanın (yalnızca varsayılan hesap).

## Hızlı kurulum

1. Bir Nostr anahtar çifti oluşturun (gerekirse):

```bash
# nak kullanarak
nak key generate
```

2. Yapılandırmaya ekleyin:

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
    },
  },
}
```

3. Anahtarı dışa aktarın:

```bash
export NOSTR_PRIVATE_KEY="nsec1..."
```

4. Gateway'i yeniden başlatın.

## Yapılandırma başvurusu

| Anahtar          | Tür     | Varsayılan                                     | Açıklama                                              |
| ------------ | -------- | ------------------------------------------- | -------------------------------------------------------- |
| `privateKey` | string   | gerekli                                    | `nsec` veya onaltılık biçimde özel anahtar; gizli değer başvurularına izin verilir |
| `relays`     | string[] | `['wss://relay.damus.io', 'wss://nos.lol']` | Aktarıcı URL'leri (WebSocket)                                   |
| `dmPolicy`   | string   | `pairing`                                   | DM erişim ilkesi                                         |
| `allowFrom`  | string[] | `[]`                                        | İzin verilen gönderen açık anahtarları                                   |
| `enabled`    | boolean  | `true`                                      | Kanalı etkinleştir/devre dışı bırak                                   |
| `name`       | string   | -                                           | Görünen ad                                             |
| `profile`    | object   | -                                           | NIP-01 profil meta verileri                                  |

## Profil meta verileri

Profil verileri, bir NIP-01 `kind:0` olayı olarak yayımlanır. Bunları Control UI'dan (Channels -> Nostr -> Profile) yönetebilir veya doğrudan yapılandırmada ayarlayabilirsiniz.

Örnek:

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      profile: {
        name: "openclaw",
        displayName: "OpenClaw",
        about: "Kişisel asistan DM botu",
        picture: "https://example.com/avatar.png",
        banner: "https://example.com/banner.png",
        website: "https://example.com",
        nip05: "openclaw@example.com",
        lud16: "openclaw@example.com",
      },
    },
  },
}
```

Notlar:

- Profil URL'leri `https://` kullanmalıdır.
- Aktarıcılardan içe aktarma, alanları birleştirir ve yerel geçersiz kılmaları korur.

## Erişim denetimi

### DM ilkeleri

- **eşleştirme** (varsayılan): bilinmeyen gönderenler bir eşleştirme kodu alır.
- **izin listesi**: yalnızca `allowFrom` içindeki açık anahtarlar DM gönderebilir.
- **açık**: herkese açık gelen DM'ler (`allowFrom: ["*"]` gerektirir).
- **devre dışı**: gelen DM'leri yok sayar.

Uygulama notları:

- Gelen olay imzaları, gönderen ilkesi ve NIP-04 şifre çözme işleminden önce doğrulanır; böylece sahte olaylar erkenden reddedilir.
- Eşleştirme yanıtları, özgün DM gövdesinin şifresi çözülmeden veya gövde işlenmeden gönderilir.
- Gelen DM'lere hız sınırı uygulanır (genel olarak ve gönderen başına) ve aşırı büyük yükler şifre çözme işleminden önce bırakılır.

### İzin listesi örneği

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      dmPolicy: "allowlist",
      allowFrom: ["npub1abc...", "npub1xyz..."],
    },
  },
}
```

## Anahtar biçimleri

Kabul edilen biçimler:

- **Özel anahtar:** `nsec...` veya 64 karakterlik onaltılık değer
- **Açık anahtarlar (`allowFrom`):** `npub...` veya onaltılık değer

## Aktarıcılar

Varsayılanlar: `relay.damus.io` ve `nos.lol`.

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      relays: ["wss://relay.damus.io", "wss://relay.primal.net", "wss://nostr.wine"],
    },
  },
}
```

İpuçları:

- Yedeklilik için 2-3 aktarıcı kullanın.
- Çok fazla aktarıcı kullanmaktan kaçının (gecikme, yinelenme).
- Ücretli aktarıcılar güvenilirliği artırabilir.
- Yerel aktarıcılar test için uygundur (`ws://localhost:7777`).

## Protokol desteği

| NIP    | Durum    | Açıklama                           |
| ------ | --------- | ------------------------------------- |
| NIP-01 | Destekleniyor | Temel olay biçimi + profil meta verileri |
| NIP-04 | Destekleniyor | Şifrelenmiş DM'ler (`kind:4`)              |
| NIP-17 | Planlanıyor   | Hediye paketiyle sarmalanmış DM'ler                      |
| NIP-44 | Planlanıyor   | Sürümlendirilmiş şifreleme                  |

## Test

### Yerel aktarıcı

```bash
# strfry'ı başlat
docker run -p 7777:7777 ghcr.io/hoytech/strfry
```

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      relays: ["ws://localhost:7777"],
    },
  },
}
```

### Manuel test

1. Botun açık anahtarını Gateway günlüklerinden veya `openclaw channels status` üzerinden not edin (onaltılık; gerekirse istemcinizde npub'a dönüştürün).
2. Bir Nostr istemcisi açın (Amethyst, Damus vb.).
3. Botun açık anahtarına DM gönderin.
4. Yanıtı doğrulayın.

## Sorun giderme

### Mesajlar alınmıyor

- Özel anahtarın geçerli olduğunu doğrulayın.
- Aktarıcı URL'lerine erişilebildiğinden ve `wss://` (yerel için `ws://`) kullandıklarından emin olun.
- `enabled` değerinin `false` olmadığını doğrulayın.
- Aktarıcı bağlantı hataları için Gateway günlüklerini kontrol edin.

### Yanıtlar gönderilmiyor

- Aktarıcının yazma işlemlerini kabul ettiğini kontrol edin.
- Giden bağlantıyı doğrulayın.
- Aktarıcı hız sınırlarını izleyin.

### Yinelenen yanıtlar

- Birden fazla aktarıcı kullanılırken bu beklenen bir durumdur.
- Mesajların yinelenenleri olay kimliğine göre ayıklanır; yalnızca ilk teslimat bir yanıtı tetikler.

## Güvenlik

- Özel anahtarları asla commit etmeyin.
- Anahtarlar için ortam değişkenlerini kullanın.
- Üretim botları için `allowlist` kullanmayı değerlendirin.
- İmzalar gönderen ilkesinden önce doğrulanır ve gönderen ilkesi şifre çözme işleminden önce uygulanır; böylece sahte olaylar erkenden reddedilir ve bilinmeyen gönderenler tüm kriptografik işlemlerin gerçekleştirilmesini zorlayamaz.

## Sınırlamalar (MVP)

- Yalnızca doğrudan mesajlar (grup sohbeti yok).
- Medya eki yok.
- Yalnızca NIP-04 (NIP-17 hediye paketiyle sarmalama planlanıyor).

## İlgili

- [Kanallara Genel Bakış](/tr/channels) — desteklenen tüm kanallar
- [Eşleştirme](/tr/channels/pairing) — DM kimlik doğrulaması ve eşleştirme akışı
- [Gruplar](/tr/channels/groups) — grup sohbeti davranışı ve bahsetme denetimi
- [Kanal Yönlendirme](/tr/channels/channel-routing) — mesajlar için oturum yönlendirmesi
- [Güvenlik](/tr/gateway/security) — erişim modeli ve güçlendirme
