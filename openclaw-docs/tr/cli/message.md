---
read_when:
    - Mesaj CLI eylemleri ekleme veya değiştirme
    - Giden kanal davranışını değiştirme
summary: '`openclaw message` için CLI başvurusu (gönderme + kanal eylemleri)'
title: İleti
x-i18n:
    generated_at: "2026-07-26T23:36:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e2d1cca9be7cfa7625cac3e440ecb5847d9fab9c545c9267a41a2f99c26c514b
    source_path: cli/message.md
    workflow: 16
---

# `openclaw message`

Discord, Google Chat, iMessage, Matrix, Mattermost (plugin), Microsoft Teams,
Signal, Slack, Telegram ve WhatsApp genelinde mesaj ve kanal eylemleri göndermek için
tek bir giden komut.

```bash
openclaw message <subcommand> [flags]
```

## Kanal seçimi

- Birden fazla kanal yapılandırılmışsa `--channel <name>` gereklidir;
  tam olarak bir kanal yapılandırılmışsa bu kanal varsayılandır.
- Değerler: `discord|googlechat|imessage|matrix|mattermost|msteams|signal|slack|telegram|whatsapp`
  (Mattermost için plugin gereklidir).
- Kanal önekli hedefler (örneğin `discord:channel:123`), açıkça bir
  `--channel` belirtilmeden sahip plugin'i çözümler.

## Hedef biçimleri (`-t, --target`)

| Kanal               | Biçim                                                                                                      |
| ------------------- | ---------------------------------------------------------------------------------------------------------- |
| Discord             | `channel:<id>`, `user:<id>`, `<@id>` bahsetmesi veya yalnızca sayısal bir kimlik (kanal kimliği olarak değerlendirilir) |
| Google Chat         | `spaces/<spaceId>` veya `users/<userId>`                                                                     |
| iMessage            | tanıtıcı, `chat_id:<id>`, `chat_guid:<guid>` veya `chat_identifier:<id>`                                      |
| Mattermost (plugin) | `channel:<id>`, `user:<id>`, `@username` veya yalnızca bir kimlik (kanal olarak değerlendirilir) |
| Matrix              | `@user:server`, `!room:server` veya `#alias:server`                                                         |
| Microsoft Teams     | `conversation:<id>` (`19:...@thread.tacv2`), yalnızca bir konuşma kimliği veya `user:<aad-object-id>`             |
| Signal              | `+E.164`, `group:<id>`, `uuid:<id>`, `username:<name>`/`u:<name>` veya bunlardan herhangi birinin `signal:` önekli hâli |
| Slack               | `channel:<id>` veya `user:<id>` (yalnızca bir kimlik kanal olarak değerlendirilir)                                          |
| Telegram            | sohbet kimliği, `@username` veya bir forum konusu hedefi: `<chatId>:topic:<topicId>` (ya da `--thread-id <topicId>`)     |
| WhatsApp            | E.164, grup JID'si (`...@g.us`) veya Kanal/Bülten JID'si (`...@newsletter`)                                |

Kanal adı araması: dizini olan sağlayıcılarda (Discord/Slack/vb.) `Help`
veya `#help` gibi adlar dizin önbelleği aracılığıyla çözümlenir; önbellekte
bulunamadığında ve sağlayıcı desteklediğinde canlı dizin aramasına başvurulur.

## Ortak bayraklar

Her eylem şunları kabul eder: `--channel <name>`, `--account <id>`, `--json`,
`--dry-run`, `--verbose`. Hedef alan eylemler ayrıca
`-t, --target <dest>` değerini de kabul eder.

## SecretRef çözümlemesi

`openclaw message`, eylemi çalıştırmadan önce kanal SecretRef'lerini mümkün
olduğunca dar bir kapsamda çözümler:

- `--channel` ayarlandığında (veya önekli bir hedeften çıkarıldığında) kanal kapsamlı
- `--account` da ayarlandığında hesap kapsamlı
- hiçbiri ayarlanmadığında yapılandırılmış tüm kanallar

İlgisiz kanallardaki çözümlenmemiş SecretRef'ler hedeflenmiş bir eylemi asla
engellemez; seçilen kanal/hesaptaki çözümlenmemiş bir SecretRef, eylemin güvenli biçimde başarısız olmasına neden olur.

## Eylemler

### Temel

| Eylem           | Kanallar                                                                                                        | Gerekli                                                        | Notlar                                                                                                                                                                                                                                                                                                  |
| --------------- | --------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `send`          | Discord, Google Chat, iMessage, Matrix, Mattermost (plugin), Microsoft Teams, Signal, Slack, Telegram, WhatsApp | `--target` ve `--message`/`--media`/`--presentation` seçeneklerinden biri | Aşağıdaki [Gönderme](#send) bölümüne bakın.                                                                                                                                                                                                                                                                               |
| `poll`          | Discord, Matrix, Microsoft Teams, Telegram, WhatsApp                                                            | `--target`, `--poll-question`, `--poll-option` (tekrarlanabilir)        | Aşağıdaki [Anket](#poll) bölümüne bakın.                                                                                                                                                                                                                                                                               |
| `react`         | Discord, Matrix, Nextcloud Talk, Signal, Slack, Telegram, WhatsApp                                              | `--message-id`, `--target`                                     | `--emoji`, `--remove` (`--emoji` gerektirir; desteklendiği yerlerde kendi tepkilerinizi temizlemek için bunu atlayın, bkz. [Tepkiler](/tr/tools/reactions)). WhatsApp: `--participant`, `--from-me`. Signal grup tepkileri `--target-author` veya `--target-author-uuid` gerektirir. Nextcloud Talk yalnızca tepki ekler; `--remove` hata verir. |
| `reactions`     | Discord, Matrix, Microsoft Teams, Slack                                                                         | `--message-id`, `--target`                                     | `--limit`.                                                                                                                                                                                                                                                                                             |
| `read`          | Discord, Matrix, Microsoft Teams, Slack                                                                         | `--target`                                                     | `--limit`, `--message-id`, `--before`, `--after`. Discord: `--around`, `--include-thread`. Slack: `--message-id` belirli bir zaman damgasını okur; tam bir ileti dizisi yanıtı için `--thread-id` ile birleştirin.                                                                                                     |
| `edit`          | Discord, Matrix, Microsoft Teams, Slack, Telegram                                                               | `--message-id`, `--message`, `--target`                        | Telegram forum ileti dizileri `--thread-id` kullanır.                                                                                                                                                                                                                                                              |
| `delete`        | Discord, Matrix, Microsoft Teams, Slack, Telegram                                                               | `--message-id`, `--target`                                     |                                                                                                                                                                                                                                                                                                        |
| `pin` / `unpin` | Discord, Matrix, Microsoft Teams, Slack                                                                         | `--message-id`, `--target`                                     | `unpin` ayrıca `--pinned-message-id` değerini de kabul eder (Microsoft Teams: sohbet mesajı kimliği değil, sabitleme/sabitlemeleri listeleme kaynağı kimliği).                                                                                                                                                                                  |
| `pins` (liste)   | Discord, Matrix, Microsoft Teams, Slack                                                                         | `--target`                                                     | `--limit`.                                                                                                                                                                                                                                                                                             |
| `permissions`   | Discord, Matrix                                                                                                 | `--target`                                                     | Matrix: yalnızca şifreleme etkinleştirildiğinde ve doğrulama eylemlerine izin verildiğinde kullanılabilir.                                                                                                                                                                                                                |
| `search`        | Discord                                                                                                         | `--guild-id`, `--query`                                        | `--channel-id`, `--channel-ids` (tekrarlanabilir), `--author-id`, `--author-ids` (tekrarlanabilir), `--limit`.                                                                                                                                                                                                           |
| `member info`   | Discord, Matrix, Microsoft Teams, Slack                                                                         | `--user-id`                                                    | `--guild-id` (Discord).                                                                                                                                                                                                                                                                                |

### Gönderme

```bash
openclaw message send --channel discord \
  --target channel:123 --message "hi" --reply-to 456
```

- `--media <path-or-url>`: görüntü/ses/video/belge ekler (yerel yol veya
  URL).
- `--presentation <json>`: `text`, `context`, `divider`,
  `chart`, `table`, `buttons` ve `select` bloklarını içeren, kanal
  yeteneğine göre işlenen paylaşılan yük. Bkz. [Mesaj Sunumu](/tr/plugins/message-presentation).
- `--delivery <json>`: genel teslim tercihleri; örneğin `{"pin":
true}`. `--pin`, kanal desteklediğinde sabitlenmiş teslimin kısa gösterimidir.
- `--reply-to <id>`, `--thread-id <id>` (Telegram forum konusu; Slack ileti dizisi
  zaman damgası, `--reply-to` ile aynı alan).
- `--force-document` (Telegram, WhatsApp): kanal sıkıştırmasını önlemek için
  görüntüleri/GIF'leri/videoları belge olarak gönderir.
- `--silent` (Telegram, Discord): bildirim olmadan gönderir.
- `--gif-playback` (yalnızca WhatsApp): video medyasını GIF oynatımı olarak değerlendirir.

```bash
openclaw message send --channel discord \
  --target channel:123 --message "Seçin:" \
  --presentation '{"blocks":[{"type":"buttons","buttons":[{"label":"Onayla","value":"approve","style":"success"},{"label":"Reddet","value":"decline","style":"danger"}]}]}'
```

```bash
openclaw message send --channel telegram --target @mychat --message "Seçin:" \
  --presentation '{"blocks":[{"type":"buttons","buttons":[{"label":"Evet","value":"cmd:yes"},{"label":"Hayır","value":"cmd:no"}]}]}'
```

Slack, desteklenen grafik bloklarını yerel olarak işler; diğer kanallar aynı
verileri okunabilir metin olarak alır:

```bash
openclaw message send --channel slack --target channel:C123 \
  --presentation '{"blocks":[{"type":"chart","chartType":"bar","title":"Üç aylık gelir","categories":["Q1","Q2"],"series":[{"name":"Gelir","values":[120,145]}],"xLabel":"Çeyrek"}]}'
```

Slack ayrıca açık tablo bloklarını yerel olarak işler. Diğer kanallar başlığı
ve her satırı belirlenimci metin olarak alır:

```bash
openclaw message send --channel slack --target channel:C123 \
  --presentation '{"title":"İşlem hattı raporu","blocks":[{"type":"table","caption":"Açık işlem hattı","headers":["Hesap","Aşama","ARR"],"rows":[["Acme","Won",125000],["Globex","Review",82000]],"rowHeaderColumnIndex":0}]}'
```

Telegram Mini App düğmeleri `webApp` kullanır (`web_app` eski
JSON için ayrıştırılmaya devam eder) ve yalnızca kullanıcı ile bot arasındaki özel sohbetlerde işlenir:

```bash
openclaw message send --channel telegram --target 123456789 --message "Uygulamayı açın:" \
  --presentation '{"blocks":[{"type":"buttons","buttons":[{"label":"Başlat","webApp":{"url":"https://example.com/app"}}]}]}'
```

```bash
openclaw message send --channel telegram --target @mychat \
  --media ./diagram.png --force-document
```

```bash
openclaw message send --channel msteams \
  --target conversation:19:abc@thread.tacv2 \
  --presentation '{"title":"Durum güncellemesi","blocks":[{"type":"text","text":"Derleme tamamlandı"}]}'
```

### Anket

```bash
openclaw message poll --channel discord \
  --target channel:123 \
  --poll-question "Atıştırmalık?" \
  --poll-option Pizza --poll-option Suşi \
  --poll-multi --poll-duration-hours 48
```

- `--poll-option <choice>`: 2-12 kez tekrarlayın.
- `--poll-multi`: birden fazla seçime izin verin.
- Discord: `--poll-duration-hours`, `--silent`, `--message`.
- Telegram: `--poll-duration-seconds <n>` (5-600), `--silent`,
  `--poll-anonymous` / `--poll-public`, `--thread-id`.

```bash
openclaw message poll --channel telegram \
  --target @mychat \
  --poll-question "Öğle yemeği?" \
  --poll-option Pizza --poll-option Suşi \
  --poll-duration-seconds 120 --silent
```

```bash
openclaw message poll --channel msteams \
  --target conversation:19:abc@thread.tacv2 \
  --poll-question "Öğle yemeği?" \
  --poll-option Pizza --poll-option Suşi
```

### İleti dizileri

- `thread create`: Discord kanalları. Gerekli: `--thread-name`, `--target`
  (kanal kimliği). İsteğe bağlı: `--message-id`, `--message`, `--auto-archive-min`.
- `thread list`: Discord kanalları. Gerekli: `--guild-id`. İsteğe bağlı:
  `--channel-id`, `--include-archived`, `--before`, `--limit`.
- `thread reply`: Discord kanalları. Gerekli: `--target` (ileti dizisi kimliği),
  `--message`. İsteğe bağlı: `--media`, `--reply-to`.

### Emojiler

- `emoji list`: Discord (`--guild-id`), Slack (ek bayrak yok).
- `emoji upload`: Discord. Gerekli: `--guild-id`, `--emoji-name`, `--media`.
  İsteğe bağlı: `--role-ids` (tekrarlanabilir).

### Çıkartmalar

- `sticker send`: Discord. Gerekli: `--target`, `--sticker-id` (tekrarlanabilir).
  İsteğe bağlı: `--message`.
- `sticker upload`: Discord. Gerekli: `--guild-id`, `--sticker-name`,
  `--sticker-desc`, `--sticker-tags`, `--media`.

### Roller, kanallar, ses ve etkinlikler (Discord)

- `role info`: `--guild-id`.
- `role add` / `role remove`: `--guild-id`, `--user-id`, `--role-id`.
- `channel info`: `--target`.
- `channel list`: `--guild-id`.
- `voice status`: `--guild-id`, `--user-id`.
- `event list`: `--guild-id`.
- `event create`: gerekli `--guild-id`, `--event-name`, `--start-time`;
  isteğe bağlı `--end-time`, `--desc`, `--channel-id`, `--location`,
  `--event-type`, `--image <url-or-path>`.

### Moderasyon (Discord)

- `timeout`: `--guild-id`, `--user-id`; isteğe bağlı `--duration-min` veya
  `--until` (zaman aşımını temizlemek için ikisini de atlayın), `--reason`.
- `kick`: `--guild-id`, `--user-id`, `--reason`.
- `ban`: `--guild-id`, `--user-id`, `--delete-days`, `--reason`.

### Yayın

```bash
openclaw message broadcast --targets <target...> [--channel all] [--message <text>] [--media <url>] [--dry-run]
```

Bir yükü birden fazla hedefe gönderir. `--targets`, boşlukla ayrılmış bir
liste alır. Yapılandırılmış tüm sağlayıcıları hedeflemek için `--channel all` kullanın.

## İlgili

- [CLI başvurusu](/tr/cli)
- [Agent gönderimi](/tr/tools/agent-send)
- [Mesaj Sunumu](/tr/plugins/message-presentation)
