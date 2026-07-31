---
read_when:
    - Etkin bir oturumun yanıtlarını Telegram'dan Discord, Slack, Mattermost veya başka bir bağlı kanala taşımak istiyorsunuz
    - Kanallar arası doğrudan mesajlar için session.identityLinks'i yapılandırıyorsunuz
    - Bir /dock komutu, gönderenin bağlı olmadığını veya etkin bir oturumun bulunmadığını belirtiyor
summary: Bir OpenClaw oturumunun yanıt rotasını bağlı sohbet kanalları arasında taşıma
title: Kanal yerleştirme
x-i18n:
    generated_at: "2026-07-26T22:43:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d7af3a59b95b2c73cb74a9529584e51caed055719db2df8aad2ba8e8c9b0593
    source_path: concepts/channel-docking.md
    workflow: 16
---

Kanal kenetleme, tek bir OpenClaw oturumu için çağrı yönlendirmedir. Aynı
konuşma bağlamını korur ancak bu oturumun gelecekteki yanıtlarının nereye
iletileceğini değiştirir. Kenetleme yalnızca doğrudan sohbetten çalışır; grup
sohbetinden çalışmaz.

## Örnek

Alice, Telegram ve Discord üzerinden OpenClaw'a mesaj gönderebilir:

```json5
{
  session: {
    identityLinks: {
      alice: ["telegram:123", "discord:456"],
    },
  },
}
```

Alice bunu bir Telegram doğrudan sohbetinden gönderirse:

```text
/dock_discord
```

OpenClaw mevcut oturum bağlamını korur ve yanıt rotasını değiştirir:

| Kenetlemeden önce               | `/dock_discord` sonrasında       |
| ---------------------------- | --------------------------- |
| Yanıtlar Telegram'a gider `123` | Yanıtlar Discord'a gider `456` |

Oturum yeniden oluşturulmaz. Transkript geçmişi aynı oturuma bağlı kalır.

## Neden kullanılır?

Bir görev bir sohbet uygulamasında başladığında ancak sonraki yanıtların başka
bir yere ulaşması gerektiğinde kenetlemeyi kullanın.

Yaygın akış:

1. Telegram'dan bir aracı görevi başlatın.
2. Çalışmayı koordine ettiğiniz Discord'a geçin.
3. Telegram doğrudan sohbetinden `/dock_discord` gönderin.
4. Aynı OpenClaw oturumunu koruyun ancak gelecekteki yanıtları Discord'da alın.

## Gerekli yapılandırma

Kenetleme için `session.identityLinks` gereklidir. Kaynak gönderen ve hedef eş
aynı kimlik grubunda olmalıdır:

```json5
{
  session: {
    identityLinks: {
      alice: ["telegram:123", "discord:456", "slack:U123"],
    },
  },
}
```

Değerler, kanal ön ekli eş kimlikleridir:

| Değer          | Anlamı                      |
| -------------- | ---------------------------- |
| `telegram:123` | Telegram gönderen kimliği `123`     |
| `discord:456`  | Discord doğrudan eş kimliği `456` |
| `slack:U123`   | Slack kullanıcı kimliği `U123`         |

Kanonik anahtar (yukarıdaki `alice`) yalnızca paylaşılan kimlik grubunun adıdır.
Kenetleme komutları, kaynak gönderen ile hedef eşin aynı kişi olduğunu kanıtlamak
için kanal ön ekli değerleri kullanır.

## Komutlar

OpenClaw, yerel komutları destekleyen yüklü her kanal plugini için bir
`/dock-<channel>` komutu oluşturur; dolayısıyla pluginler eklendikçe liste büyür.
Şu anda bunu destekleyen paketlenmiş pluginler:

| Hedef kanal | Komut            | Diğer ad              |
| -------------- | ------------------ | ------------------ |
| Discord        | `/dock-discord`    | `/dock_discord`    |
| Mattermost     | `/dock-mattermost` | `/dock_mattermost` |
| Slack          | `/dock-slack`      | `/dock_slack`      |
| Telegram       | `/dock-telegram`   | `/dock_telegram`   |

Alt çizgili biçim, eğik çizgi komutlarını doğrudan sunan Telegram gibi
yüzeylerde aynı zamanda yerel komut adıdır.

## Neler değişir?

Kenetleme, etkin oturumun teslimat alanlarını günceller:

| Oturum alanı   | `/dock_discord` sonrasındaki örnek            |
| --------------- | ---------------------------------------- |
| `lastChannel`   | `discord`                                |
| `lastTo`        | `456`                                    |
| `lastAccountId` | hedef kanal hesabı veya `default` |

Bu alanlar oturum deposunda kalıcı hâle getirilir ve söz konusu oturumun
sonraki yanıtlarının teslimatında kullanılır.

## Neler değişmez?

Kenetleme şunları yapmaz:

- kanal hesapları oluşturmaz
- yeni bir Discord, Telegram, Slack veya Mattermost botuna bağlanmaz
- bir kullanıcıya erişim izni vermez
- kanal izin listelerini veya DM politikalarını atlamaz
- transkript geçmişini başka bir oturuma taşımaz
- ilgisiz kullanıcıların bir oturumu paylaşmasını sağlamaz

Yalnızca mevcut oturumun teslimat rotasını değiştirir.

## Sorun giderme

**Komut, gönderenin bağlantılı olmadığını söylüyor.**

Hem mevcut göndereni hem de hedef eşi aynı `session.identityLinks` grubuna ekleyin.
Örneğin Telegram göndereni `123`, Discord eşi `456` ile
kenetlenecekse hem `telegram:123` hem de `discord:456` değerlerini ekleyin.

**Komut, kenetlemenin yalnızca doğrudan sohbetlerde kullanılabildiğini söylüyor.**

Kenetleme komutunu grup sohbetinden değil, OpenClaw ile doğrudan sohbetten gönderin.

**Komut, etkin bir oturumun bulunmadığını söylüyor.**

Mevcut bir doğrudan sohbet oturumundan kenetleyin. Komutun yeni rotayı kalıcı
hâle getirebilmesi için etkin bir oturum girdisine ihtiyacı vardır.

**Yanıtlar hâlâ eski kanala gidiyor.**

Komutun bir başarı mesajıyla yanıt verdiğini kontrol edin ve hedef eş kimliğinin
söz konusu kanalın kullandığı kimlikle eşleştiğini doğrulayın. Kenetleme yalnızca
etkin oturum rotasını değiştirir; başka bir oturum hâlâ farklı bir yere yönlendirilebilir.

**Geri dönmem gerekiyor.**

Bağlantılı bir gönderenden, özgün kanal için `/dock_telegram` veya
`/dock-telegram` gibi eşleşen komutu gönderin.
