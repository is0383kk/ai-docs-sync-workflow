---
read_when:
    - Kanal yönlendirmesini veya gelen kutusu davranışını değiştirme
summary: Kanal başına yönlendirme kuralları (WhatsApp, Telegram, Discord, Slack) ve paylaşılan bağlam
title: Kanal yönlendirme
x-i18n:
    generated_at: "2026-07-26T22:37:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa03f04a55015bf17e0fe1f3a9bc422875124bb64af5891c898a98bc6917d9e8
    source_path: channels/channel-routing.md
    workflow: 16
---

# Kanallar ve yönlendirme

OpenClaw, yanıtları **mesajın geldiği kanala geri** yönlendirir. Model bir kanal
seçmez; yönlendirme deterministiktir ve ana makine yapılandırması tarafından
kontrol edilir. Varsayılan DM kapsamında, her kanaldan gelen doğrudan mesajlar
aracının [ana oturumunda](/concepts/main-session) birleşir.

## Temel terimler

- **Kanal**: `discord`, `googlechat`, `imessage`, `irc`, `line`, `signal`, `slack`, `telegram` veya `whatsapp` gibi paketle birlikte gelen bir kanal plugini ve ayrıca kurulu plugin kanalları. `webchat`, dahili WebChat kullanıcı arayüzü kanalıdır ve yapılandırılabilir bir giden kanal değildir.
- **AccountId**: kanal başına hesap örneği (destekleniyorsa).
- İsteğe bağlı varsayılan kanal hesabı: `channels.<channel>.defaultAccount`, bir giden yol `accountId` belirtmediğinde
  hangi hesabın kullanılacağını seçer.
  - Çok hesaplı kurulumlarda, iki veya daha fazla hesap yapılandırıldığında açık bir varsayılan (`defaultAccount` veya `default` adlı bir hesap) belirleyin. Bu olmadan geri dönüş yönlendirmesi, normalleştirilmiş ilk hesap kimliğini seçebilir.
- **AgentId**: yalıtılmış bir çalışma alanı + oturum deposu ("beyin").
- **SessionKey**: bağlamı depolamak ve eşzamanlılığı kontrol etmek için kullanılan bölüm anahtarı.

## Giden hedef önekleri

Açık giden hedefler, `telegram:123` veya `tg:123` gibi bir sağlayıcı öneki içerebilir. Çekirdek, bu öneki yalnızca seçilen kanal `last` olduğunda veya başka şekilde çözümlenemediğinde ve yalnızca yüklenen plugin bu öneki bildirdiğinde kanal seçimi ipucu olarak değerlendirir. Çağıran zaten açık bir kanal seçmişse sağlayıcı öneki bu kanalla eşleşmelidir; `telegram:123` hedefine WhatsApp üzerinden teslimat gibi kanallar arası birleşimler, plugine özgü hedef normalleştirmesinden önce başarısız olur.

`channel:<id>`, `user:<id>`, `room:<id>`, `thread:<id>`, `imessage:<handle>` ve `sms:<number>` gibi hedef türü ve hizmet önekleri, seçilen kanalın söz dizimi içinde kalır. Bunlar sağlayıcıyı kendi başlarına seçmez.

## Oturum anahtarı biçimleri (örnekler)

Doğrudan mesajlar varsayılan olarak aracının **ana** oturumunda birleştirilir:

- `agent:<agentId>:<mainKey>` (varsayılan: `agent:main:main`)

`session.dmScope`, DM birleştirmesini kontrol eder: `main` (varsayılan) tek bir ana
oturumu paylaşırken `per-peer`, `per-channel-peer` ve `per-account-channel-peer`
DM'leri ayrı oturumlarda tutar. Bir yönlendirme bağlaması, eşleşen eşler için kapsamı
`bindings[].session.dmScope` aracılığıyla geçersiz kılabilir.

Doğrudan mesaj konuşma geçmişi ana oturumla paylaşılsa bile korumalı alan ve
araç politikası, harici DM'ler için hesap başına türetilmiş bir doğrudan sohbet çalışma zamanı anahtarı kullanır;
böylece kanaldan kaynaklanan mesajlar yerel ana oturum çalıştırmaları gibi değerlendirilmez.

Gruplar ve kanallar, kanal başına yalıtılmış kalır:

- Gruplar: `agent:<agentId>:<channel>:group:<id>`
- Kanallar/odalar: `agent:<agentId>:<channel>:channel:<id>`

İleti dizileri:

- Slack/Discord ileti dizileri, temel anahtara `:thread:<threadId>` ekler.
- Telegram forum konuları, grup anahtarına `:topic:<topicId>` yerleştirir.

Örnekler:

- `agent:main:telegram:group:-1001234567890:topic:42`
- `agent:main:discord:channel:123456:thread:987654`

## Ana DM yönlendirmesini sabitleme

`session.dmScope`, `main` olduğunda doğrudan mesajlar tek bir ana oturumu paylaşabilir.
Oturumun `lastRoute` değerinin sahip olmayan kişilerin DM'leri tarafından üzerine yazılmasını önlemek için
OpenClaw, aşağıdakilerin tümü doğru olduğunda `allowFrom` üzerinden sabitlenmiş bir sahip çıkarımı yapar:

- `allowFrom` tam olarak bir joker karakter içermeyen girdiye sahiptir.
- Girdi, bu kanal için somut bir gönderen kimliğine normalleştirilebilir.
- Gelen DM'nin göndereni, sabitlenmiş bu sahiple eşleşmez.

Bu eşleşmeme durumunda OpenClaw, gelen oturum meta verilerini yine kaydeder ancak
ana oturumun `lastRoute` değerini güncellemeyi atlar.

## Korumalı gelen kayıt

Kanal pluginleri, korumalı bir yolun yeni bir OpenClaw oturumu oluşturmaması gerektiğinde
gelen oturum kaydını `createIfMissing: false` olarak işaretleyebilir. Bu modda
OpenClaw, mevcut bir oturumun meta verilerini ve `lastRoute` değerini güncelleyebilir ancak
yalnızca bir mesaj gözlemlendi diye yalnızca yönlendirmeye özgü bir oturum girdisi oluşturmaz.

## Yönlendirme kuralları (bir araç nasıl seçilir)

Yönlendirme, her gelen mesaj için **bir araç** seçer:

1. **Tam eş eşleşmesi** (`bindings` ile `peer.kind` + `peer.id`).
2. **Üst eş eşleşmesi** (ileti dizisi devralımı).
3. **Eş joker karakter eşleşmesi** (bir eş türü için `peer.id: "*"`).
4. **Sunucu + roller eşleşmesi** (Discord), `guildId` + `roles` aracılığıyla.
5. **Sunucu eşleşmesi** (Discord), `guildId` aracılığıyla.
6. **Ekip eşleşmesi** (Slack), `teamId` aracılığıyla.
7. **Hesap eşleşmesi** (kanaldaki `accountId`).
8. **Kanal eşleşmesi** (o kanaldaki herhangi bir hesap, `accountId: "*"`).
9. **Varsayılan araç** (`agents.entries.*.default`; yoksa listedeki ilk girdi, son çare olarak `main`).

Bir bağlama birden fazla eşleştirme alanı (`peer`, `guildId`, `teamId`, `roles`) içerdiğinde, bağlamanın uygulanması için **sağlanan tüm alanların eşleşmesi gerekir**.

Eşleşen araç, hangi çalışma alanının ve oturum deposunun kullanılacağını belirler.

## Yayın grupları (birden fazla araç çalıştırma)

Yayın grupları, **OpenClaw'ın normalde yanıt vereceği durumlarda** aynı eş için **birden fazla araç** çalıştırmanıza olanak tanır (örneğin WhatsApp gruplarında, bahsetme/etkinleştirme denetiminden sonra).

Yapılandırma:

```json5
{
  broadcast: {
    strategy: "parallel",
    "120363403215116621@g.us": ["alfred", "baerbel"],
    "+15555550123": ["support", "logger"],
  },
}
```

Bkz. [Yayın Grupları](/tr/channels/broadcast-groups).

## Yapılandırmaya genel bakış

- `agents.entries`: adlandırılmış araç tanımları (çalışma alanı, model vb.).
- `bindings`: gelen kanalları/hesapları/eşleri araçlarla eşleştirir.

Örnek:

```json5
{
  agents: {
    list: [{ id: "support", name: "Support", workspace: "~/.openclaw/workspace-support" }],
  },
  bindings: [
    { match: { channel: "slack", teamId: "T123" }, agentId: "support" },
    { match: { channel: "telegram", peer: { kind: "group", id: "-100123" } }, agentId: "support" },
  ],
}
```

## Oturum depolama

Çalışma zamanı oturum satırları, durum dizini altındaki her aracının SQLite veritabanında
bulunur (varsayılan `~/.openclaw`):

- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`

Eski kurulumlarda, `~/.openclaw/agents/<agentId>/sessions/` altında eski transkript JSONL dosyaları ve bir `sessions.json` satır
deposu bulunabilir. Gateway başlatma işlemi ve
`openclaw doctor --fix`, etkin eski satırları/geçmişi otomatik olarak SQLite'a aktarır.
Açık geçiş kanıtına ihtiyaç duyduğunuzda `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` ve
[Doctor](/tr/cli/doctor#session-sqlite-migration) doğrulama sırasını kullanın.
Geçiş ve çevrimdışı bakım iş akışları için `session.store` ve `{agentId}`
şablonlaması aracılığıyla eski bir depo yolu seçmeye devam edebilirsiniz.

Gateway ve ACP oturum keşfi ayrıca varsayılan `agents/` kökü ve şablonlanmış
`session.store` kökleri altındaki disk destekli araç depolarını tarar. Keşfedilen
depolar, çözümlenen araç kökü içinde kalmalı ve normal bir eski
`sessions.json` dosyası kullanmalıdır. Sembolik bağlantılar ve kök dışındaki yollar yok sayılır.

## WebChat davranışı

WebChat, **seçilen araca** bağlanır ve varsayılan olarak aracın ana
oturumunu kullanır. Bu nedenle WebChat, söz konusu aracın kanallar arası bağlamını
tek bir yerde görmenizi sağlar.

## Yanıt bağlamı

Gelen yanıtlar şunları içerir:

- Kullanılabilir olduğunda `ReplyToId`, `ReplyToBody` ve `ReplyToSender`.
- Alıntılanan bağlam, `Body` öğesine bir `[Replying to ...]` bloğu olarak eklenir.

Bu, tüm kanallarda tutarlıdır.

## İlgili

- [Gruplar](/tr/channels/groups)
- [Yayın grupları](/tr/channels/broadcast-groups)
- [Eşleştirme](/tr/channels/pairing)
