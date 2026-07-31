---
read_when:
    - Bot tarafından oluşturulan kanal mesajlarını yapılandırma
    - Botlar arası döngü korumasını ayarlama
sidebarTitle: Bot loop protection
summary: Botlar arası döngü koruması varsayılanları ve kanal geçersiz kılmaları
title: Bot döngüsü koruması
x-i18n:
    generated_at: "2026-07-26T23:49:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d59d3b48dd5506e774282b880334df8970b05c4d001261ff7107e8e1678894db
    source_path: channels/bot-loop-protection.md
    workflow: 16
---

OpenClaw, `allowBots` desteği sunan kanallarda diğer botlar tarafından yazılan mesajları kabul edebilir. Bu yol etkinleştirildiğinde, bot çifti döngü koruması iki bot kimliğinin birbirine süresiz olarak yanıt vermesini önler.

Koruma, çekirdek gelen yanıt yürütücüsü tarafından uygulanır. Destekleyen her kanal, gelen olayını genel olgulara eşler: hesap veya kapsam, konuşma kimliği, gönderen bot kimliği ve alıcı bot kimliği. Çekirdek, katılımcı çiftini her iki yönde izler (A'dan B'ye ve B'den A'ya aynı çift sayılır), kayan pencere bütçesi uygular ve bütçe aşıldıktan sonra bekleme süresi boyunca çifti engeller.

## Varsayılanlar

Bir kanal botlar tarafından yazılan mesajların yönlendirmeye ulaşmasına izin verdiğinde bot çifti döngü koruması etkin olur. Yerleşik varsayılanlar:

| Anahtar              | Varsayılan | Anlam                                               |
| -------------------- | ---------- | --------------------------------------------------- |
| `enabled`   | `true` | Destekleyen kanallar için koruma etkindir.           |
| `maxEventsPerWindow`   | `20` | Bir bot çiftinin pencere içinde değiş tokuş edebileceği olaylar. |
| `windowSeconds`   | `60` | Kayan pencerenin uzunluğu.                           |
| `cooldownSeconds`   | `60` | Çift bütçeyi aştıktan sonraki engelleme süresi.      |

Koruma; insanlar tarafından yazılan mesajları, tek botlu dağıtımları, kendi mesajını filtrelemeyi veya bütçenin altında kalan bot yanıtlarını etkilemez.

## Paylaşılan varsayılanları yapılandırma

Destekleyen her kanala aynı temel ayarı vermek için `channels.defaults.botLoopProtection` değerini bir kez ayarlayın. Kanallar daha dar kapsamlı geçersiz kılmalar da sunabilir; Feishu kasıtlı olarak yalnızca bu paylaşılan temel ayarı kullanır.

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
        windowSeconds: 60,
        cooldownSeconds: 60,
      },
    },
  },
}
```

`enabled: false` değerini yalnızca kanal politikanız otomatik engelleme olmadan botlar arası konuşmalara kasıtlı olarak izin veriyorsa ayarlayın.

## Kanal, hesap veya oda bazında geçersiz kılma

Destekleyen kanallar, kendi yapılandırmalarını paylaşılan varsayılanın üzerine anahtar bazında uygular. Öncelik sırası, en dar kapsamdan başlayarak:

1. `channels.<channel>.<room-or-space>.botLoopProtection`, kanal konuşma bazında geçersiz kılmaları desteklediğinde
2. `channels.<channel>.accounts.<account>.botLoopProtection`, kanal hesapları desteklediğinde
3. `channels.<channel>.botLoopProtection`, kanal üst düzey varsayılanları desteklediğinde
4. `channels.defaults.botLoopProtection`
5. yerleşik varsayılanlar

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
      },
    },
    discord: {
      botLoopProtection: {
        maxEventsPerWindow: 8,
      },
      accounts: {
        secondary: {
          allowBots: true,
          botLoopProtection: {
            maxEventsPerWindow: 5,
            cooldownSeconds: 90,
          },
        },
      },
    },
    googlechat: {
      allowBots: true,
      groups: {
        "spaces/AAAA": {
          botLoopProtection: {
            maxEventsPerWindow: 5,
          },
        },
      },
    },
    matrix: {
      allowBots: "mentions",
      groups: {
        "!roomid:example.org": {
          botLoopProtection: {
            maxEventsPerWindow: 5,
          },
        },
      },
    },
    slack: {
      allowBots: "mentions",
      botLoopProtection: {
        maxEventsPerWindow: 8,
      },
    },
  },
}
```

## Kanal desteği

- Discord: Discord hesabı, kanal ve bot çiftine göre anahtarlanan yerel `author.bot` olguları.
- Feishu: Kabul edilen ve botlar tarafından yazılan grup mesajları için Feishu hesabı, sohbet ve bot çiftine göre anahtarlanan yerel `sender_type=bot` olguları. Feishu yalnızca `channels.defaults.botLoopProtection` kullanır.
- Google Chat: Kabul edilen ve botlar tarafından yazılan mesajlar için hesap, alan ve bot çiftine göre anahtarlanan yerel `sender.type=BOT` olguları.
- Matrix: Matrix hesabı, oda ve yapılandırılmış bot çiftine göre anahtarlanan, yapılandırılmış Matrix bot hesapları.
- Slack: Kabul edilen ve botlar tarafından yazılan mesajlar için Slack hesabı, kanal ve bot çiftine göre anahtarlanan yerel `bot_id` olguları.

Güvenilir bir gelen bot kimliği sunmayan kanallar, normal kendi mesajını ve erişim politikası filtrelerini kullanmayı sürdürür. Bot çiftindeki her iki katılımcıyı da tanımlayabilene kadar bu korumaya katılmamalıdırlar.

Plugin uygulama ayrıntıları için [SDK çalışma zamanı](/tr/plugins/sdk-runtime#reusable-runtime-utilities) bölümüne bakın.
