---
read_when:
    - Birden fazla mesaj kanalında aynı izin verilenler listesini yapılandırma
    - DM ve grup gönderen erişim kurallarını paylaşma
    - Mesaj kanalı erişim denetimini inceleme
summary: Mesaj kanalları için yeniden kullanılabilir gönderici izin listeleri
title: Erişim grupları
x-i18n:
    generated_at: "2026-07-26T23:29:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 099abc95e90d9a7b7006d19062c46b4ffdb2aecb1e8e714454a3182131a786d0
    source_path: channels/access-groups.md
    workflow: 16
---

Erişim grupları, `accessGroups` altında bir kez tanımladığınız ve kanal izin listelerinden `accessGroup:<name>` ile başvurduğunuz adlandırılmış gönderen listeleridir.

Aynı kişilere birden fazla mesaj kanalında izin verilmesi gerektiğinde veya tek bir güvenilir kümenin hem DM'ler hem de grup gönderen yetkilendirmesi için geçerli olması gerektiğinde bunları kullanın.

Bir grup tek başına hiçbir izin vermez. Yalnızca bir izin listesi alanı ona başvurduğunda önem taşır.

## Statik mesaj gönderen grupları

Statik gönderen grupları `type: "message.senders"` kullanır. `members`, mesaj kanalı kimliğine göre anahtarlanır ve her kanalın paylaştığı girdiler için ayrıca `"*"` bulunur:

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        "*": ["global-owner-id"],
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
        whatsapp: ["+15551234567"],
      },
    },
  },
}
```

| Anahtar                    | Anlam                                                                                  |
| -------------------------- | -------------------------------------------------------------------------------------- |
| `"*"`         | Gruba başvuran her mesaj kanalı için denetlenen paylaşılan girdiler.                    |
| `discord`, `telegram`, ... | Yalnızca o kanalın izin listesi eşleştirmesi için denetlenen girdiler. |

Girdiler, hedef kanalın normal `allowFrom` kurallarıyla eşleştirilir. OpenClaw, gönderen kimliklerini kanallar arasında dönüştürmez: Alice'in bir Telegram kimliği ve bir Discord kimliği varsa her iki kimliği de eşleşen kanal anahtarlarının altında listeleyin.

## İzin listelerinden gruplara başvurma

Mesaj kanalı yolunun gönderen izin listelerini desteklediği her yerde `accessGroup:<name>` ile bir gruba başvurun.

DM izin listesi örneği:

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
      },
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
    telegram: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
  },
}
```

Grup gönderen izin listesi örneği:

```json5
{
  accessGroups: {
    oncall: {
      type: "message.senders",
      members: {
        whatsapp: ["+15551234567"],
        googlechat: ["users/1234567890"],
      },
    },
  },
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["accessGroup:oncall"],
    },
    googlechat: {
      groups: {
        "spaces/AAA": {
          users: ["accessGroup:oncall"],
        },
      },
    },
  },
}
```

Grupları ve doğrudan girdileri birlikte kullanabilirsiniz:

```json5
{
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators", "discord:123456789012345678"],
    },
  },
}
```

## Desteklenen mesaj kanalı yolları

Erişim grupları, paylaşılan mesaj kanalı yetkilendirme yollarında çalışır:

- `channels.<channel>.allowFrom` gibi DM gönderen izin listeleri
- `channels.<channel>.groupAllowFrom` gibi grup gönderen izin listeleri
- aynı gönderen eşleştirme kurallarını kullanan kanala özgü oda başına gönderen izin listeleri (örneğin Google Chat `groups.<space>.users`)
- mesaj kanalı gönderen izin listelerini yeniden kullanan komut yetkilendirme yolları

Kanal desteği, ilgili kanalın paylaşılan OpenClaw gönderen yetkilendirme yardımcılarına bağlanıp bağlanmadığına bağlıdır. Mevcut paketli destek ClickClack, Discord, Feishu, Google Chat, iMessage, IRC, LINE, Mattermost, Microsoft Teams, Nextcloud Talk, Nostr, QQ Bot, Signal, Slack, SMS, Telegram, WhatsApp, Zalo ve Zalo Personal'ı kapsar. Statik `message.senders` grupları kanaldan bağımsızdır; dolayısıyla yeni mesaj kanalları, özel izin listesi genişletmesi yerine paylaşılan plugin SDK giriş yardımcılarını kullanarak bunları edinir.

## Discord kanal kitleleri

Discord ayrıca dinamik bir erişim grubu türünü destekler:

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
      membership: "canViewChannel",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers"],
    },
  },
}
```

`discord.channelAudience`, "şu anda bu sunucu kanalını görüntüleyebilen Discord DM gönderenlerine izin ver" anlamına gelir. OpenClaw, yetkilendirme sırasında göndereni Discord üzerinden çözümler ve Discord `ViewChannel` izin kurallarını uygular. `membership` isteğe bağlıdır ve varsayılan olarak `canViewChannel` değerini alır.

Bir Discord kanalı, `#maintainers` veya `#on-call` gibi bir ekip için zaten doğruluk kaynağı olduğunda bunu kullanın.

Gereksinimler ve hata davranışı:

- Botun sunucuya ve kanala erişmesi gerekir.
- Botun Discord Developer Portal **Server Members Intent** özelliğine ihtiyacı vardır.
- Discord `Missing Access` döndürdüğünde, gönderen bir sunucu üyesi olarak çözümlenemediğinde veya kanal başka bir sunucuya ait olduğunda erişim grubu güvenli biçimde erişimi reddeder.

Discord'a özgü diğer örnekler: [Discord erişim denetimi](/tr/channels/discord#access-control-and-routing)

## Plugin tanılamaları

Plugin yazarları, yapılandırılmış erişim grubu durumunu yeniden düz bir izin listesine genişletmeden inceleyebilir:

```typescript
import { resolveAccessGroupAllowFromState } from "openclaw/plugin-sdk/access-groups";

const state = await resolveAccessGroupAllowFromState({
  accessGroups: cfg.accessGroups,
  allowFrom: channelConfig.allowFrom,
  channel: "my-channel",
  accountId: "default",
  senderId,
  isSenderAllowed,
});
```

Sonuç; başvurulan, eşleşen, eksik, desteklenmeyen ve başarısız grupları bildirir. Bunu tanılama veya uygunluk testleri için kullanın. `expandAllowFromWithAccessGroups(...)` öğesini yalnızca hâlâ düz bir `allowFrom` dizisi bekleyen uyumluluk yolları için kullanın.

## Güvenlik notları

- Erişim grupları rol değil, izin listesi diğer adlarıdır. Tek başlarına sahip oluşturmaz, eşleştirme isteklerini onaylamaz veya araç izinleri vermezler.
- `dmPolicy: "open"`, etkin DM izin listesinde yine de `"*"` gerektirir. Bir erişim grubuna başvurmak, genel erişimle aynı değildir.
- Eksik grup adları güvenli biçimde erişimi reddeder. `allowFrom`, `accessGroup:operators` içeriyor ve `accessGroups.operators` mevcut değilse bu girdi hiç kimseyi yetkilendirmez.
- Kanal kimliklerini kararlı tutun. Kanal her ikisini de destekliyorsa görünen adlar yerine sayısal/kullanıcı kimliklerini tercih edin.

## Sorun giderme

Bir gönderenin eşleşmesi gerektiği hâlde erişimi engelleniyorsa:

1. İzin listesi alanının tam `accessGroup:<name>` başvurusunu içerdiğini doğrulayın.
2. `accessGroups.<name>.type` değerinin doğru olduğunu doğrulayın.
3. Gönderen kimliğinin eşleşen kanal anahtarının veya `"*"` öğesinin altında listelendiğini doğrulayın.
4. Girdinin ilgili kanalın normal izin listesi söz dizimini kullandığını doğrulayın.
5. Discord kanal kitleleri için botun sunucu kanalını görebildiğini ve Server Members Intent özelliğinin etkinleştirildiğini doğrulayın.

Erişim denetimi yapılandırmasını düzenledikten sonra `openclaw doctor` komutunu çalıştırın. Bu komut, birçok geçersiz izin listesi ve ilke birleşimini çalışma zamanından önce yakalar.
