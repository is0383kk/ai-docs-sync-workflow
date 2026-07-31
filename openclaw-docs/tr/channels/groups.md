---
read_when:
    - Grup sohbeti davranışını veya bahsetme denetimini değiştirme
    - mentionPatterns'ı belirli grup konuşmalarıyla sınırlandırma
sidebarTitle: Groups
summary: Platformlar genelinde grup sohbeti davranışı (Discord/iMessage/Matrix/Microsoft Teams/QQBot/Signal/Slack/Telegram/WhatsApp/Zalo)
title: Gruplar
x-i18n:
    generated_at: "2026-07-26T22:37:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 146378f0fc31e129b6504df6778ab8633c048cd4d02af02a5e6da1bfef640d3f
    source_path: channels/groups.md
    workflow: 16
---

OpenClaw; Discord, iMessage, Matrix, Microsoft Teams, QQBot, Signal, Slack, Telegram, WhatsApp ve Zalo dahil olmak üzere grup destekli kanalların tümünde aynı grup kurallarını uygular.

Temsilci açıkça görünür bir mesaj göndermedikçe sessiz bağlam sağlaması gereken sürekli etkin odalar için [Ortam odası etkinlikleri](/tr/channels/ambient-room-events) bölümüne bakın.

## Başlangıç tanıtımı (2 dakika)

OpenClaw kendi mesajlaşma hesaplarınızda "yaşar". Ayrı bir WhatsApp bot kullanıcısı yoktur: bir grupta **siz** bulunuyorsanız OpenClaw bu grubu görebilir ve orada yanıt verebilir.

Varsayılan davranış:

- Gruplar kısıtlanmıştır (`groupPolicy: "allowlist"`); grup gönderenleri izin verilenler listesine eklenene kadar engellenir.
- Bir grup için bahsetme denetimini devre dışı bırakmadığınız sürece yanıtlar bir bahsetme gerektirir.
- Son yanıt metni odaya otomatik olarak gönderilir (`visibleReplies: "automatic"`).

Başka bir deyişle: izin verilenler listesindeki gönderenler, OpenClaw'dan bahsederek onu tetikleyebilir.

<Note>
**Kısaca**

- **DM erişimi**, `*.allowFrom` tarafından denetlenir.
- **Grup erişimi**, `*.groupPolicy` + izin verilenler listeleri (`*.groups`, `*.groupAllowFrom`) tarafından denetlenir.
- **Yanıt tetikleme**, bahsetme denetimi (`requireMention`, `/activation`) tarafından denetlenir.

</Note>

Hızlı akış (bir grup mesajına ne olur):

```text
groupPolicy? disabled -> bırak
groupPolicy? allowlist -> gruba izin veriliyor mu? hayır -> bırak
requireMention? evet -> bahsedildi mi? hayır -> yalnızca bağlam için sakla
bahsetme/yanıt/komut/DM -> kullanıcı isteği
sürekli etkin grup konuşması -> kullanıcı isteği veya yapılandırıldığında oda etkinliği
```

## Görünür yanıtlar

Normal grup/kanal isteklerinde OpenClaw varsayılan olarak `messages.groupChat.visibleReplies: "automatic"` kullanır: son asistan metni görünür yanıt olarak odaya gönderilir.

Paylaşılan bir odada temsilcinin ne zaman konuşacağına `message(action=send)` çağrısı yaparak karar vermesi gerekiyorsa `messages.groupChat.visibleReplies: "message_tool"` kullanın. Bu, araçları güvenilir biçimde kullanan modellerle (örneğin GPT-5.6 Sol) en iyi şekilde çalışır. Model aracı kullanmaz ve anlamlı bir son metin döndürürse OpenClaw bu metni odaya göndermek yerine gizli tutar.

Yalnızca araçla teslim talimatını güvenilir biçimde izlemeyen modeller veya çalışma zamanları için `"automatic"` kullanın: normal son metinler doğrudan odaya gönderilir ve temsilci, son metinle birlikte gönderilemeyen dosyalar, görseller veya diğer ekler için yine de `message(action=send)` çağrısı yapabilir.

Mesaj aracı etkin araç politikası kapsamında kullanılamıyorsa OpenClaw yanıtı sessizce bastırmak yerine otomatik görünür yanıtlara geri döner. `openclaw doctor` bu uyumsuzluk hakkında uyarır.

Doğrudan sohbetler ve diğer tüm kaynak etkinlikleri için `messages.visibleReplies: "message_tool"` aynı yalnızca araç davranışını genel olarak uygular; `messages.groupChat.visibleReplies`, grup/kanal odaları için daha özel geçersiz kılma olarak kalır. Dahili WebChat doğrudan turları varsayılan olarak otomatik son yanıt teslimini kullanır; böylece Pi ve Codex aynı görünür yanıt sözleşmesini alır.

Yalnızca araç modu, çoğu sessiz izleme modu turunda modeli `NO_REPLY` yanıtını vermeye zorlama şeklindeki eski kalıbın yerini alır. Yalnızca araç modunda istem bir `NO_REPLY` sözleşmesi tanımlamaz; görünür hiçbir şey yapmamak, yalnızca mesaj aracını çağırmamak anlamına gelir.

Plugin'e ait konuşma bağlamaları istisnadır. Bir Plugin, bir iş parçacığını bağlayıp gelen turun sahipliğini aldıktan sonra Plugin'in döndürdüğü yanıt görünür bağlama yanıtıdır; `message(action=send)` gerektirmez. Bu yanıt gizli model son metni değil, Plugin çalışma zamanı çıktısıdır.

Doğrudan grup istekleri için yazıyor göstergeleri yine gönderilir. Etkinleştirildiğinde ortam niteliğindeki sürekli etkin oda etkinlikleri, temsilci mesaj aracını çağırmadıkça kesinlikle sessiz kalır.

Oturumlar varsayılan olarak ayrıntılı araç/ilerleme özetlerini bastırır. Hata ayıklama sırasında geçerli oturum için bunları göstermek üzere `/verbose on` (veya `/verbose full`), yalnızca son yanıt davranışına dönmek için `/verbose off` kullanın. Ayrıntı durumu oturum başınadır ve doğrudan sohbetlerde, gruplarda, kanallarda ve forum konularında aynı şekilde çalışır.

Bahsedilmemiş sürekli etkin grup konuşmalarını kullanıcı istekleri yerine sessiz oda bağlamı olarak göndermek için [Ortam odası etkinlikleri](/tr/channels/ambient-room-events) özelliğini kullanın:

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
    },
  },
}
```

Varsayılan değer `unmentionedInbound: "user_request"` şeklindedir. Bahsetme içeren mesajlar, komutlar, iptal istekleri ve DM'ler kullanıcı isteği olarak kalır.

Grup/kanal isteklerinde görünür çıktının mesaj aracı üzerinden gönderilmesini zorunlu tutmak için:

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "message_tool",
    },
  },
}
```

Bunu her kaynak sohbet için zorunlu tutmak üzere:

```json5
{
  messages: {
    visibleReplies: "message_tool",
  },
}
```

Gateway, dosya kaydedildikten sonra `messages` yapılandırma değişikliklerini yeniden başlatılmadan uygular. Yalnızca yapılandırmayı yeniden yükleme devre dışı bırakıldığında (`gateway.reload.mode: "off"`) yeniden başlatın.

Komut turları `visibleReplies: "message_tool"` ayarını atlar ve her zaman görünür biçimde yanıt verir: yerel eğik çizgi komutları (Discord, Telegram ve yerel komut desteği sunan diğer yüzeyler) ile yetkilendirilmiş metin `/...` komutlarının ikisi de yanıtlarını kaynak sohbete gönderir. Gruplardaki yetkilendirilmemiş metin `/...` turları yalnızca mesaj aracı modunda kalır; sıradan sohbet turları yapılandırılan varsayılanı izler.

## Bağlam görünürlüğü ve izin verilenler listeleri

Grup güvenliğinde iki farklı denetim kullanılır:

- **Tetikleme yetkilendirmesi**: temsilciyi kimlerin tetikleyebileceği (`groupPolicy`, `groups`, `groupAllowFrom`, kanala özgü izin verilenler listeleri).
- **Bağlam görünürlüğü**: modele hangi ek bağlamın eklendiği (yanıt/alıntı metni, iş parçacığı geçmişi, iletilen meta veriler).

OpenClaw varsayılan olarak bağlamı alındığı hâliyle korur: izin verilenler listeleri modelin hangi alıntılanmış veya geçmiş parçacıkları gördüğünü değil, kimlerin eylemleri tetikleyebileceğini belirler. Ek bağlamı da filtrelemek için `contextVisibility` ayarını belirleyin:

| Mod                | Davranış                                                                         |
| ------------------- | -------------------------------------------------------------------------------- |
| `"all"` (varsayılan)   | Ek bağlamı alındığı hâliyle korur.                                           |
| `"allowlist"`       | Yalnızca izin verilenler listesindeki gönderenlerden gelen geçmiş/iş parçacığı/alıntı/iletilmiş bağlamı ekler.     |
| `"allowlist_quote"` | `allowlist`; ayrıca herhangi bir gönderenden açıkça alıntılanan/yanıtlanan mesajı korur. |

Bunu kanal başına (`channels.<channel>.contextVisibility`), hesap başına (`channels.<channel>.accounts.<accountId>.contextVisibility`) veya genel olarak (`channels.defaults.contextVisibility`) ayarlayın. Ek bağlamı getiren kanallar (Discord, Feishu, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp), gelen bağlamı oluştururken politikayı uygular; bilinmeyen politika birleşimleri güvenli biçimde başarısız olur ve bağlamı dışarıda bırakır.

Bu modlar yalnızca kanal tarafından sağlanan ek bağlamı filtreler. Araç politikası ve yalnızca sahibin kullanabildiği araç envanteri, istemde temsil edilen her gönderene göre değil, yine geçerli turun kaynak istekte bulunan kişisine göre seçilir. [İstekte bulunan kişi kapsamlı denetimler ve istem bağlamı](/tr/gateway/security#requester-scoped-controls-and-prompt-context) bölümüne bakın.

![Grup mesajı akışı](/images/groups-flow.svg)

İstediğiniz...

| Hedef                                         | Ayarlanacak değer                                                |
| -------------------------------------------- | ---------------------------------------------------------- |
| Tüm gruplara izin verip yalnızca @bahsetmelerine yanıt verme | `groups: { "*": { requireMention: true } }`                |
| Tüm grup yanıtlarını devre dışı bırakma                    | `groupPolicy: "disabled"`                                  |
| Yalnızca belirli gruplar                         | `groups: { "<group-id>": { ... } }` (`"*"` anahtarı olmadan)         |
| Gruplarda yalnızca sizin tetikleyebilmeniz               | `groupPolicy: "allowlist"`, `groupAllowFrom: ["+1555..."]` |
| Güvenilen tek bir gönderen kümesini kanallar arasında yeniden kullanma | `groupAllowFrom: ["accessGroup:operators"]`                |

Yeniden kullanılabilir gönderen izin listeleri için [Erişim grupları](/tr/channels/access-groups) bölümüne bakın.

## Oturum anahtarları

- Grup oturumları `agent:<agentId>:<channel>:group:<id>` oturum anahtarlarını kullanır (odalar/kanallar `agent:<agentId>:<channel>:channel:<id>` kullanır).
- Telegram forum konuları, her konunun kendi oturumu olması için grup kimliğine `:topic:<threadId>` ekler.
- Doğrudan sohbetler ana oturumu kullanır (veya `session.dmScope` yapılandırılmışsa gönderen başına oturumları).
- Heartbeat'ler yapılandırılan Heartbeat oturumunda çalışır (varsayılan: temsilcinin ana oturumu); grup oturumları kendi Heartbeat'lerini çalıştırmaz.

<a id="pattern-personal-dms-public-groups-single-agent"></a>

## Kalıp: kişisel DM'ler + herkese açık gruplar (tek temsilci)

Evet — "kişisel" trafiğiniz **DM'lerden**, "herkese açık" trafiğiniz ise **gruplardan** oluşuyorsa bu yaklaşım iyi çalışır.

Nedeni: tek temsilci modunda DM'ler genellikle **ana** oturum anahtarına (`agent:main:main`) ulaşırken gruplar her zaman **ana olmayan** oturum anahtarlarını (`agent:main:<channel>:group:<id>`) kullanır. Korumalı alan kullanımını `mode: "non-main"` ile etkinleştirirseniz bu grup oturumları yapılandırılan korumalı alan arka ucunda çalışırken ana DM oturumunuz ana makinede kalır. Bir arka uç seçmezseniz varsayılan arka uç Docker'dır.

Böylece tek bir temsilci "beyniniz" (paylaşılan çalışma alanı + bellek), ancak iki farklı yürütme biçiminiz olur:

- **DM'ler**: tüm araçlar (ana makine)
- **Gruplar**: korumalı alan + kısıtlı araçlar

<Note>
Gerçekten ayrı çalışma alanlarına/kişiliklere ihtiyacınız varsa ("kişisel" ve "herkese açık" hiçbir zaman karışmamalıysa), ikinci bir temsilci + bağlamalar kullanın. [Çok Temsilcili Yönlendirme](/tr/concepts/multi-agent) bölümüne bakın.
</Note>

<Tabs>
  <Tab title="DM'ler ana makinede, gruplar korumalı alanda">
    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main", // gruplar/kanallar ana değildir -> korumalı alanda
            scope: "session", // en güçlü yalıtım (grup/kanal başına bir kapsayıcı)
            workspaceAccess: "none",
          },
        },
      },
      tools: {
        sandbox: {
          tools: {
            // allow boş değilse diğer her şey engellenir (deny yine önceliklidir).
            allow: ["group:messaging", "group:sessions"],
            deny: ["group:runtime", "group:fs", "group:ui", "nodes", "cron", "gateway"],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Gruplar yalnızca izin verilenler listesindeki bir klasörü görür">
    "Ana makine erişimi yok" yerine "gruplar yalnızca X klasörünü görebilir" mi istiyorsunuz? `workspaceAccess: "none"` değerini koruyun ve korumalı alana yalnızca izin verilenler listesindeki yolları bağlayın:

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",
            scope: "session",
            workspaceAccess: "none",
            docker: {
              binds: [
                // hostPath:containerPath:mode
                "/home/user/FriendsShared:/data:ro",
              ],
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

İlgili:

- Yapılandırma anahtarları ve varsayılanlar: [Gateway yapılandırması](/tr/gateway/config-agents#agentsdefaultssandbox)
- Bir aracın neden engellendiğini hata ayıklama: [Korumalı Alan ile Araç Politikası ile Yükseltilmiş Yetki Karşılaştırması](/tr/gateway/sandbox-vs-tool-policy-vs-elevated)
- Bağlama noktası ayrıntıları: [Korumalı alan kullanımı](/tr/gateway/sandboxing#custom-bind-mounts)

## Görünen etiketler

- Kullanılabilir olduğunda kullanıcı arabirimi etiketleri `displayName` değerini, `<channel>:<token>` biçiminde kullanır.
- `#room`, odalar/kanallar için ayrılmıştır; grup sohbetleri `g-<slug>` kullanır (küçük harf, boşluklar -> `-`, `#@+._-` değerini koruyun). Çok uzun ve anlaşılmaz kimlikler, tam rota kimliklerini kullanıcı arabirimine sızdırmak yerine kararlı bir belirteç olarak kısaltılır.

## Grup politikası

Grup/oda mesajlarının kanal başına nasıl işleneceğini denetleyin:

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "disabled", // "open" | "disabled" | "allowlist"
      groupAllowFrom: ["+15551234567"],
    },
    telegram: {
      groupPolicy: "disabled",
      groupAllowFrom: ["123456789"], // sayısal Telegram kullanıcı kimliği (kurulum @username değerini çözümler)
    },
    signal: {
      groupPolicy: "disabled",
      groupAllowFrom: ["+15551234567"],
    },
    imessage: {
      groupPolicy: "disabled",
      groupAllowFrom: ["chat_id:123"],
    },
    msteams: {
      groupPolicy: "disabled",
      groupAllowFrom: ["user@org.com"],
    },
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        GUILD_ID: { channels: { help: { enabled: true } } },
      },
    },
    slack: {
      groupPolicy: "allowlist",
      channels: { "#general": { enabled: true } },
    },
    matrix: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["@owner:example.org"],
      groups: {
        "!roomId:example.org": { enabled: true },
        "#alias:example.org": { enabled: true },
      },
    },
  },
}
```

| İlke          | Davranış                                                        |
| ------------- | --------------------------------------------------------------- |
| `"open"`      | Gruplar izin listelerini atlar; bahsetme geçidi uygulanmaya devam eder. |
| `"disabled"`  | Tüm grup mesajlarını tamamen engeller.                          |
| `"allowlist"` | Yalnızca yapılandırılmış izin listesiyle eşleşen gruplara/odalara izin verir. |

<AccordionGroup>
  <Accordion title="Kanal bazında notlar">
    - `groupPolicy`, bahsetme geçidinden (@bahsetmeler gerektirir) ayrıdır.
    - WhatsApp/Telegram/Signal/iMessage/Microsoft Teams/Zalo: `groupAllowFrom` kullanın (geri dönüş: açıkça belirtilmiş `allowFrom`).
    - Signal: `groupAllowFrom`, gelen Signal grup kimliğiyle veya gönderenin telefon numarası/UUID değeriyle eşleşebilir.
    - DM eşleştirme onayları (`*-allowFrom` deposu girdileri) yalnızca DM erişimi için geçerlidir; grup göndereni yetkilendirmesi açıkça grup izin listeleriyle belirlenmeye devam eder.
    - Discord: izin listesi `channels.discord.guilds.<id>.channels` kullanır.
    - Slack: izin listesi `channels.slack.channels` kullanır.
    - Matrix: izin listesi `channels.matrix.groups` kullanır. Oda kimliklerini (`!room:server`) veya diğer adları (`#alias:server`) kullanın; oda adı anahtarları yalnızca `channels.matrix.dangerouslyAllowNameMatching: true` ile eşleşir ve çözümlenemeyen girdiler çalışma zamanında yok sayılır. Gönderenleri kısıtlamak için `channels.matrix.groupAllowFrom` kullanın; oda bazında `users` izin listeleri de desteklenir.
    - Grup DM'leri ayrı olarak denetlenir (`channels.discord.dm.*`, `channels.slack.dm.*`: `groupEnabled`, `groupChannels`).
    - Telegram: gönderen izin listeleri yalnızca sayısal kullanıcı kimliklerini kabul eder (`"123456789"`; `telegram:`/`tg:` önekleri büyük/küçük harfe duyarsız biçimde kaldırılır). `@username` girdileri çalışma zamanında eşleşmez ve bir uyarı günlüğe kaydedilir; kurulum `@username` değerlerini kimliklere çözümler. Negatif sohbet kimlikleri gönderen izin listelerinde değil, `channels.telegram.groups` altında yer almalıdır.
    - Varsayılan değer `groupPolicy: "allowlist"`; grup izin listeniz boşsa grup mesajları engellenir.
    - Çalışma zamanı güvenliği: bir sağlayıcı bloğu tamamen eksik olduğunda (`channels.<provider>` yoksa), grup ilkesi `channels.defaults.groupPolicy` değerini devralmak yerine güvenli biçimde `allowlist` olarak kapanır ve Gateway geri dönüşü hesap başına bir kez günlüğe kaydeder.

  </Accordion>
</AccordionGroup>

Hızlı zihinsel model (grup mesajları için değerlendirme sırası):

<Steps>
  <Step title="groupPolicy">
    `groupPolicy` (açık/devre dışı/izin listesi).
  </Step>
  <Step title="Grup izin listeleri">
    Grup izin listeleri (`*.groups`, `*.groupAllowFrom`, kanala özgü izin listesi).
  </Step>
  <Step title="Bahsetme geçidi">
    Bahsetme geçidi (`requireMention`, `/activation`).
  </Step>
</Steps>

## Bahsetme geçidi (varsayılan)

Grup bazında geçersiz kılınmadığı sürece grup mesajları bir bahsetme gerektirir. Varsayılanlar her alt sistem için `*.groups."*"` altında bulunur.

Desteklenen örtük bahsetme olguları kanala özgüdür:

| Olgu                    | Mevcut yerleşik üreticiler                       |
| ----------------------- | ------------------------------------------------ |
| Bota verilen yanıt      | Discord, Microsoft Teams, QQBot, Slack, Telegram |
| Bottan yapılan alıntı   | WhatsApp, Zalo personal                          |
| Bot ileti dizisine katıldı | Mattermost, Slack, Tlon                       |

Kanal ilgili olguyu ürettiğinde her olgu varsayılan olarak etkinleştirilir. İlgili olgunun bahsetme geçidini atlamasını engellemek için karşılık gelen `implicitMentions` bayrağını `false` olarak ayarlayın; yerel açık bahsetmeler bundan etkilenmez. Bir bayrak, ilgili olguyu üretmeyen kanalları etkilemez.

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
        "123@g.us": { requireMention: false },
      },
    },
    telegram: {
      groups: {
        "*": { requireMention: true },
        "123456789": { requireMention: false },
      },
    },
    imessage: {
      groups: {
        "*": { requireMention: true },
        "123": { requireMention: false },
      },
    },
  },
  agents: {
    entries: {
      main: {
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw", "\\+15555550123"],
          historyLimit: 50,
        },
      },
    },
  },
}
```

## Yapılandırılmış bahsetme kalıplarının kapsamı

Yapılandırılmış `mentionPatterns`, regex geri dönüş tetikleyicileridir. Platform
yerel bir bot bahsetmesi sunmadığında veya `openclaw:` gibi düz metnin
bahsetme olarak sayılmasını istediğinizde bunları kullanın. Yerel platform bahsetmeleri ayrıdır:
Discord, Slack, Telegram, Matrix, Signal veya başka bir kanal mesajda
bottan açıkça bahsedildiğini kanıtlayabildiğinde, yapılandırılmış regex kalıpları
reddedilmiş olsa bile bu yerel bahsetme tetiklemeye devam eder.

Varsayılan olarak yapılandırılmış bahsetme kalıpları, kanalın sağlayıcı ve konuşma olgularını bahsetme algılamasına ilettiği her yerde uygulanır. Geniş kapsamlı kalıpların agent'ı her grupta uyandırmasını önlemek için bunları `channels.<channel>.mentionPatterns` ile kanal bazında kapsamlandırın.

Regex bahsetme kalıplarının bir kanal için varsayılan olarak kapalı olması gerektiğinde `mode: "deny"` kullanın, ardından belirli odaları `allowIn` ile etkinleştirin:

```json5
{
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b", "\\bops bot\\b"],
    },
  },
  channels: {
    slack: {
      mentionPatterns: {
        mode: "deny",
        allowIn: ["C0123OPS"],
      },
    },
  },
}
```

Regex bahsetme kalıplarının geniş kapsamda uygulanması gerektiğinde varsayılan `mode: "allow"` değerini kullanın (veya `mode` değerini atlayın), ardından gürültülü odalarda bunları `denyIn` ile kapatın:

```json5
{
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b"],
    },
  },
  channels: {
    telegram: {
      mentionPatterns: {
        denyIn: ["-1001234567890", "-1001234567890:topic:42"],
      },
    },
  },
}
```

İlke çözümleme:

| Alan            | Etki                                                                                                                  |
| --------------- | --------------------------------------------------------------------------------------------------------------------- |
| `mode: "allow"` | Konuşma kimliği `denyIn` içinde olmadığı sürece regex bahsetme kalıpları etkindir. Bu varsayılan davranıştır. |
| `mode: "deny"`  | Konuşma kimliği `allowIn` içinde olmadığı sürece regex bahsetme kalıpları devre dışıdır.                      |
| `allowIn`       | Reddetme modunda regex bahsetme kalıplarının etkin olduğu konuşma kimlikleri.                                         |
| `denyIn`        | Regex bahsetme kalıplarının devre dışı olduğu konuşma kimlikleri. Her ikisi de aynı kimliği içeriyorsa `denyIn`, `allowIn` değerine üstün gelir. |

Bugün desteklenen kapsamlı regex ilkesi:

| Kanal    | `allowIn` / `denyIn` içinde kullanılan kimlikler |
| -------- | ------------------------------------------------------------ |
| Discord  | Discord kanal kimlikleri.                                    |
| Matrix   | Matrix oda kimlikleri.                                       |
| Slack    | Slack kanal kimlikleri.                                      |
| Telegram | Grup sohbeti kimlikleri veya forum konuları için `chatId:topic:threadId`. |
| WhatsApp | `123@g.us` gibi WhatsApp konuşma kimlikleri.         |

Hesap düzeyindeki kanal yapılandırmaları, ilgili kanal birden fazla hesabı desteklediğinde aynı ilkeyi `channels.<channel>.accounts.<accountId>.mentionPatterns` altında ayarlayabilir. Hesap ilkesi, ilgili hesap için üst düzey kanal ilkesine göre önceliklidir.

<AccordionGroup>
  <Accordion title="Bahsetme geçidi notları">
    - `mentionPatterns`, büyük/küçük harfe duyarsız güvenli regex kalıplarıdır; geçersiz kalıplar ve güvenli olmayan iç içe yineleme biçimleri yok sayılır (bir uyarıyla).
    - Kalıp önceliği: `agents.entries.*.groupChat.mentionPatterns` (birden fazla agent aynı grubu paylaştığında kullanışlıdır), `messages.groupChat.mentionPatterns` değerini geçersiz kılar; hiçbiri ayarlanmadığında kalıplar agent kimliğinin adından/emojisinden türetilir.
    - Bahsetme geçidi yalnızca bahsetme algılaması mümkün olduğunda (yerel bahsetmeler veya `mentionPatterns` yapılandırıldığında) uygulanır.
    - Bir grubu veya göndereni izin listesine eklemek bahsetme geçidini devre dışı bırakmaz; tüm mesajların tetiklemesi gerektiğinde ilgili grubun `requireMention` değerini `false` olarak ayarlayın.
    - Otomatik grup sohbeti istem bağlamı, çözümlenmiş sessiz yanıt talimatını her turda taşır; çalışma alanı dosyaları `NO_REPLY` işleyişini yinelememelidir.
    - Otomatik sessiz yanıtlara izin verilen gruplar, temiz boş veya yalnızca akıl yürütme içeren model turlarını `NO_REPLY` ile eşdeğer biçimde sessiz kabul eder. Doğrudan sohbetler hiçbir zaman `NO_REPLY` yönlendirmesi almaz ve yalnızca mesaj aracı kullanan grup yanıtları `message(action=send)` çağrılmayarak sessiz kalır.
    - Ortamda sürekli etkin grup sohbetleri varsayılan olarak kullanıcı isteği semantiğini kullanır. Bunun yerine sessiz bağlam olarak göndermek için `messages.groupChat.unmentionedInbound: "room_event"` değerini ayarlayın. Kurulum örnekleri için [Ortam oda olayları](/tr/channels/ambient-room-events) bölümüne bakın.
    - Oda olayları sahte kullanıcı istekleri olarak saklanmaz ve mesaj aracı kullanılmayan oda olaylarındaki özel asistan metni sohbet geçmişi olarak yeniden oynatılmaz.
    - Discord varsayılanları `channels.discord.guilds."*"` içinde bulunur (sunucu/kanal bazında geçersiz kılınabilir).
    - Grup geçmişi bağlamı tüm kanallarda tek tip olarak sarmalanır. Bahsetme geçitli gruplar bekleyen atlanmış mesajları tutar; sürekli etkin gruplar, kanal desteklediğinde yakın zamanda işlenmiş oda mesajlarını da tutabilir. Genel varsayılan için `messages.groupChat.historyLimit`, geçersiz kılmalar için `channels.<channel>.historyLimit` (veya `channels.<channel>.accounts.*.historyLimit`) kullanın. Devre dışı bırakmak için `0` olarak ayarlayın.

  </Accordion>
</AccordionGroup>

## Grup/kanal aracı kısıtlamaları (isteğe bağlı)

Bazı kanal yapılandırmaları, **belirli bir grup/oda/kanal içinde** hangi araçların kullanılabileceğinin kısıtlanmasını destekler.

- `tools`: grubun tamamı için araçlara izin verin/reddedin (`allow`, `alsoAllow`, `deny`; reddetme üstün gelir).
- `toolsBySender`: grup içinde gönderen bazında geçersiz kılmalar. Açık anahtar önekleri kullanın: `channel:<channelId>:<senderId>`, `id:<senderId>`, `e164:<phone>`, `username:<handle>`, `name:<displayName>` ve `"*"` joker karakteri. Kanal kimlikleri, standart OpenClaw kanal kimliklerini kullanır; `teams` gibi diğer adlar `msteams` değerine normalleştirilir. Öneksiz eski anahtarlar hâlâ kabul edilir, yalnızca `id:` olarak eşleştirilir ve kullanımdan kaldırma uyarısı günlüğe kaydedilir.

Çözümleme sırası (en belirgin olan üstün gelir):

<Steps>
  <Step title="Grup toolsBySender">
    Grup/kanal `toolsBySender` eşleşmesi.
  </Step>
  <Step title="Grup araçları">
    Grup/kanal `tools`.
  </Step>
  <Step title="Varsayılan toolsBySender">
    Varsayılan (`"*"`) `toolsBySender` eşleşmesi.
  </Step>
  <Step title="Varsayılan araçlar">
    Varsayılan (`"*"`) `tools`.
  </Step>
</Steps>

Örnek (Telegram):

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { tools: { deny: ["exec"] } },
        "-1001234567890": {
          tools: { deny: ["exec", "read", "write"] },
          toolsBySender: {
            "id:123456789": { alsoAllow: ["exec"] },
          },
        },
      },
    },
  },
}
```

<Note>
Grup/kanal araç kısıtlamaları, genel/ajan araç politikasına ek olarak uygulanır (reddetme yine önceliklidir). Bazı kanallar, odalar/kanallar için farklı iç içe yerleştirme kullanır (ör. Discord `guilds.*.channels.*`, Slack `channels.*`, Microsoft Teams `teams.*.channels.*`).
</Note>

## Grup izin listeleri

`channels.whatsapp.groups`, `channels.telegram.groups` veya `channels.imessage.groups` yapılandırıldığında, anahtarlar bir grup izin listesi işlevi görür. Varsayılan bahsetme davranışını ayarlamaya devam ederken tüm gruplara izin vermek için `"*"` kullanın.

<Warning>
Yaygın karışıklık: DM eşleştirme onayı, grup yetkilendirmesiyle aynı değildir. DM eşleştirmeyi destekleyen kanallarda eşleştirme deposu yalnızca DM'lerin kilidini açar. Grup komutları yine de `groupAllowFrom` gibi yapılandırma izin listelerinden veya ilgili kanal için belgelenmiş yapılandırma geri dönüşünden açık grup gönderen yetkilendirmesi gerektirir.
</Warning>

Yaygın amaçlar (kopyala/yapıştır):

<Tabs>
  <Tab title="Tüm grup yanıtlarını devre dışı bırak">
    ```json5
    {
      channels: { whatsapp: { groupPolicy: "disabled" } },
    }
    ```
  </Tab>
  <Tab title="Yalnızca belirli gruplara izin ver (WhatsApp)">
    ```json5
    {
      channels: {
        whatsapp: {
          groups: {
            "123@g.us": { requireMention: true },
            "456@g.us": { requireMention: false },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Tüm gruplara izin ver ancak bahsetmeyi zorunlu tut">
    ```json5
    {
      channels: {
        whatsapp: {
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Yalnızca sahip tetikleyicileri (WhatsApp)">
    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## Etkinleştirme (yalnızca sahip)

Grup sahipleri, bağımsız bir mesajla grup başına etkinleştirmeyi açıp kapatabilir:

- `/activation mention`
- `/activation always`

`/activation`, sahip denetimine tabi bir çekirdek komuttur ve yalnızca grup sohbetlerinde geçerlidir. Sahip, gönderenin `commands.ownerAllowFrom` ile eşleşmesi anlamına gelir; kanal `allowFrom` listeleri yalnızca sıradan kanal ve komut erişimini denetler. Depolanan mod, bunu dikkate alan kanallarda (Google Chat, QQBot, Telegram, WhatsApp) ilgili grubun `requireMention` değerini geçersiz kılar ve grup sistem istemi girişi her yerde etkin modu yansıtır.

## Bağlam alanları

Gelen grup yükleri şunları ayarlar:

- `ChatType=group`
- `GroupSubject` (biliniyorsa)
- `GroupMembers` (biliniyorsa)
- `WasMentioned` (bahsetme geçidi sonucu)
- Telegram forum konuları ayrıca `MessageThreadId` ve `IsForum` içerir.

Ajan sistem istemi, yeni bir grup oturumunun ilk turunda (ve `/activation` değiştikten sonra) bir grup girişi içerir. Modele bir insan gibi yanıt vermesini, boş satırları en aza indirip normal sohbet aralıklarına uymasını ve değişmez `\n` dizilerini yazmaktan kaçınmasını hatırlatır. Bildirilen tablo modu yerel veya ham tabloları korumayan kanallar, Markdown tablolarının kullanılmasını da önermez. Kanal kaynaklı grup adları ve katılımcı etiketleri, satır içi sistem talimatları olarak değil, kod çiti içinde güvenilmeyen meta veriler olarak işlenir.

## iMessage ayrıntıları

- Yönlendirme veya izin listesine ekleme sırasında `chat_id:<id>` tercih edin.
- Sohbetleri listeleme: `imsg chats --limit 20`.
- Grup yanıtları her zaman aynı `chat_id` öğesine geri gönderilir.

## WhatsApp sistem istemleri

Grup ve doğrudan istem çözümleme, joker karakter davranışı ve hesap geçersiz kılma anlamları dâhil olmak üzere standart WhatsApp sistem istemi kuralları için [WhatsApp](/tr/channels/whatsapp#system-prompts) bölümüne bakın.

## WhatsApp ayrıntıları

Yalnızca WhatsApp'a özgü davranışlar (geçmiş ekleme, bahsetme işleme ayrıntıları) için [Grup mesajları](/tr/channels/group-messages) bölümüne bakın.

## İlgili

- [Yayın grupları](/tr/channels/broadcast-groups)
- [Kanal yönlendirme](/tr/channels/channel-routing)
- [Grup mesajları](/tr/channels/group-messages)
- [Eşleştirme](/tr/channels/pairing)
