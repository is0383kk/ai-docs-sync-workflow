---
read_when:
    - Her zaman açık grup veya kanal odalarını yapılandırma
    - Temsilcinin nihai metni otomatik olarak göndermeden odadaki konuşmaları izlemesini istiyorsunuz
    - Görünür oda mesajı olmadan yazma ve token kullanımı sorunlarını giderme
sidebarTitle: Ambient room events
summary: Desteklenen grup odalarının, agent ileti aracıyla gönderim yapmadıkça sessiz bağlam sağlamasına izin verin
title: Ortam odası etkinlikleri
x-i18n:
    generated_at: "2026-07-26T23:48:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 15c083c139058c9bd2c651794965bd8252d74691e536db2ad2a2ae0b4ac886e8
    source_path: channels/ambient-room-events.md
    workflow: 16
---

Ortam oda olayları, OpenClaw'un bahsedilmemiş grup veya kanal sohbetlerini sessiz bağlam olarak işlemesini sağlar. Aracı, belleği ve oturum durumunu güncelleyebilir; ancak aracı açıkça `message` aracını çağırmadıkça oda sessiz kalır.

Her zaman etkin grup sohbetleri için `messages.groupChat.unmentionedInbound: "room_event"` ile `messages.groupChat.visibleReplies: "message_tool"` ayarlarını birlikte kullanın. Aracı dinler, yanıtın ne zaman yararlı olacağına karar verir ve artık `NO_REPLY` yanıtını vermeye dayalı eski istem kalıbına ihtiyaç duymaz.

Bugün desteklenenler: Discord sunucu kanalları, Slack kanalları ve özel kanalları, Slack çok kişili doğrudan mesajları ve Telegram grupları veya süper grupları. Diğer grup kanalları, kanal sayfalarında ortam oda olaylarını destekledikleri belirtilmediği sürece mevcut grup davranışlarını korur.

## Önerilen kurulum

Genel grup sohbeti davranışını ayarlayın:

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
}
```

Ardından bu oda için bahsetme kısıtlamasını devre dışı bırakarak odayı her zaman etkin hâle getirin. Odanın yine de normal `groupPolicy`, oda izin listesi ve gönderici izin listesi denetimlerinden geçmesi gerekir.

Yapılandırmayı kaydettikten sonra Gateway, `messages` ayarlarını çalışırken uygular. Yalnızca dosya izleme veya yapılandırmayı yeniden yükleme devre dışıysa yeniden başlatın (`gateway.reload.mode: "off"`).

## Neler değişir?

`messages.groupChat.unmentionedInbound: "room_event"` ile:

- izin verilen ve bahsetme içermeyen grup veya kanal mesajları sessiz oda olaylarına dönüşür
- bahsetme içeren mesajlar kullanıcı istekleri olarak kalır
- metin denetim komutları ve yerel komutlar kullanıcı istekleri olarak kalır
- iptal veya durdurma istekleri kullanıcı istekleri olarak kalır
- doğrudan mesajlar kullanıcı istekleri olarak kalır

Oda olayları katı görünür teslim kullanır. Nihai aracı metni özeldir. Aracının odaya gönderi yapmak için `message(action=send)` çağrısını yapması gerekir.

Yazma ve yaşam döngüsü durumu tepkileri oda olayları için gösterilmez. Tek açık alındı istisnası, yapılandırılmış onay tepkisini gönderen `messages.ackReactionScope: "all"` ayarıdır; odanın tamamen sessiz kalması gerekiyorsa daha dar bir kapsam veya `"off"` kullanın.

## Discord örneği

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "<DISCORD_SERVER_ID>": {
          requireMention: false,
          users: ["<YOUR_DISCORD_USER_ID>"],
        },
      },
    },
  },
}
```

Yalnızca bir kanalın ortam kanalı olması gerekiyorsa kanal başına Discord yapılandırmasını kullanın. Kanalı `groupPolicy: "allowlist"` altında listelemek ona izin verilmesini sağlar (`enabled: false` bir girdiyi devre dışı bırakır):

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "<DISCORD_SERVER_ID>": {
          channels: {
            "<DISCORD_CHANNEL_ID_OR_NAME>": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

## Slack örneği

Slack kanal izin listelerinde öncelik kimliklerdedir. `#channel-name` yerine `C12345678` gibi kanal kimliklerini kullanın. Kanalı `channels.slack.channels` altında listelemek ona izin verilmesini sağlar (`enabled: false` bir girdiyi devre dışı bırakır):

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    slack: {
      groupPolicy: "allowlist",
      channels: {
        "<SLACK_CHANNEL_ID>": {
          requireMention: false,
        },
      },
    },
  },
}
```

## Telegram örneği

Telegram gruplarında botun normal grup mesajlarını görebilmesi gerekir. `requireMention: false` ise BotFather gizlilik modunu devre dışı bırakın veya tüm grup trafiğini bota ileten başka bir Telegram kurulumu kullanın.

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    telegram: {
      groups: {
        "<TELEGRAM_GROUP_CHAT_ID>": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

Telegram grup kimlikleri genellikle `-1001234567890` gibi negatif sayılardır. `openclaw logs --follow` içinden `chat.id` değerini okuyun, bir grup mesajını kimlik yardımcı botuna iletin veya Bot API `getUpdates` verilerini inceleyin.

## Aracıya özgü politika

Birden fazla aracı aynı odayı paylaşıyor ancak yalnızca birinin bahsetme içermeyen sohbetleri ortam bağlamı olarak değerlendirmesi gerekiyorsa aracı geçersiz kılma ayarı kullanın:

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "message_tool",
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          unmentionedInbound: "room_event",
          mentionPatterns: ["@openclaw", "openclaw"],
        },
      },
    ],
  },
}
```

Aracıya özgü `agents.entries.*.groupChat.unmentionedInbound` değeri, bu aracı için `messages.groupChat.unmentionedInbound` değerini geçersiz kılar.

## Görünür yanıt modları

`messages.groupChat.visibleReplies`, normal grup/kanal kullanıcı istekleri için varsayılan olarak `"automatic"` değerini kullanır. Nihai aracı metninin açık bir mesaj aracı çağrısı olmadan görünür biçimde gönderilmesi gerekiyorsa bu varsayılanı koruyun.

Her zaman etkin ortam odaları için, özellikle GPT-5.6 Sol gibi en yeni nesil ve araç kullanımında güvenilir modellerle birlikte `messages.groupChat.visibleReplies: "message_tool"` kullanılması yine önerilir. Bu ayar, aracının mesaj aracını çağırarak ne zaman konuşacağına karar vermesini sağlar. Model aracı çağırmadan nihai metin döndürürse OpenClaw bu nihai metni özel tutar ve engellenen teslim meta verilerini günlüğe kaydeder.

Diğer grup istekleri otomatik yanıtları kullandığında bile oda olayları katı kalır. Bahsetme içermeyen ortam oda olaylarında görünür çıktı için her zaman `message(action=send)` gerekir.

## Geçmiş

`messages.groupChat.historyLimit`, genel grup geçmişi varsayılanını belirler (ayarlanmadığında 50; pozitif bir tam sayı olmalıdır). Kanallar bunu `channels.<channel>.historyLimit` ile geçersiz kılabilir ve bazı kanallar hesap başına geçmiş sınırlarını da destekler. Bu kanalda grup geçmişi bağlamını devre dışı bırakmak için kanal düzeyindeki `historyLimit: 0` değerini ayarlayın.

Oda olaylarını destekleyen kanallar, yakın zamandaki ortam oda mesajlarını bağlam olarak tutar. Telegram, `historyLimit` ile sınırlanan, her grup için her zaman etkin kayan bir pencere tutar; kullanıcı isteği turları botun kaydedilen son yanıtından sonraki girdileri seçerken oda olayı turları, modelin kendi yakın zamandaki gönderilerini görebilmesi için yakın zamandaki pencerenin tamamını alır. Kullanımdan kaldırılmış Telegram `includeGroupHistoryContext` mod anahtarı, `openclaw doctor --fix` tarafından kaldırılır.

## Sorun giderme

Odada yazma veya token kullanımı görünüyor ancak görünür bir mesaj görünmüyorsa:

1. Kanal izin listesinin ve gönderici izin listesinin odaya izin verdiğini doğrulayın.
2. `requireMention: false` ayarının beklediğiniz oda düzeyinde ayarlandığını doğrulayın.
3. `messages.groupChat.unmentionedInbound` veya aracı geçersiz kılma ayarının `"room_event"` olup olmadığını kontrol edin.
4. Engellenen nihai yük meta verileri veya `didSendViaMessagingTool: false` için günlükleri inceleyin.
5. Normal grup isteklerinde nihai yanıtların otomatik olarak gönderilmesini istiyorsanız `messages.groupChat.visibleReplies: "automatic"` ayarını koruyun veya geri yükleyin. `message_tool` kullanan ortam odalarında araçları güvenilir biçimde çağıran bir model/çalışma zamanı kullanın.

Telegram ortam odaları hiç tetiklenmiyorsa BotFather gizlilik modunu kontrol edin ve Gateway'in normal grup mesajlarını aldığını doğrulayın.

Slack ortam odaları tetiklenmiyorsa kanal anahtarının Slack kanal kimliği olduğunu ve uygulamanın bu oda türü için geçmiş kapsamına sahip olduğunu doğrulayın: `channels:history` (herkese açık), `groups:history` (özel) veya `mpim:history` (çok kişili doğrudan mesajlar).

## İlgili

- [Gruplar](/tr/channels/groups)
- [Discord](/tr/channels/discord)
- [Slack](/tr/channels/slack)
- [Telegram](/tr/channels/telegram)
- [Kanal sorunlarını giderme](/tr/channels/troubleshooting)
- [Kanal yapılandırma başvurusu](/tr/gateway/config-channels)
