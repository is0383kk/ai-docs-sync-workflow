---
read_when:
    - Bir kanal için kişi/grup/kendi kimliklerinizi aramak istiyorsunuz
    - Bir kanal dizini bağdaştırıcısı geliştiriyorsunuz
summary: '`openclaw directory` (kendisi, eşler, gruplar) için CLI başvurusu'
title: Dizin
x-i18n:
    generated_at: "2026-07-26T23:35:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 33f1cabd0954f2e6e6affbfbff9f8e1f543bffebc54baff7c1ffaa21778744a0
    source_path: cli/directory.md
    workflow: 16
---

# `openclaw directory`

Destekleyen kanallar için dizin aramaları: kişiler/eşler, gruplar ve "ben" (kendi).

Sonuçlar, özellikle `openclaw message send --target ...` olmak üzere diğer komutlara yapıştırılmak üzere tasarlanmıştır.

## Ortak bayraklar

- `--channel <name>`: kanal kimliği/takma adı (birden fazla kanal yapılandırıldığında gereklidir; yalnızca bir kanal yapılandırıldığında otomatik seçilir)
- `--account <id>`: hesap kimliği (varsayılan: kanal varsayılanı)
- `--json`: JSON çıktısı

Varsayılan (JSON olmayan) çıktı, sekmeyle ayrılmış `id` (ve bazen `name`) şeklindedir.

## Notlar

- Birçok kanalda sonuçlar, canlı bir sağlayıcı dizini yerine yapılandırmaya (izin listeleri / yapılandırılmış gruplar) dayanır.
- WhatsApp grup listeleme işlemi canlıdır. Gateway aramaları, Gateway'in sahip olduğu bağlantıyı yeniden kullanır; bağımsız bir komut, yalnızca başka hiçbir işlem söz konusu hesabın sahibi değilse bağlı oturumu açar, aksi takdirde canlı grupların kullanılamadığını bildirir.
- Önceden yüklenmiş bir kanal Plugin'i dizin desteğinden yoksun olabilir. Bu durumda komut, desteklenmeyen işlemi bildirir; destek eklemek için Plugin'i yeniden yüklemeye veya yükseltmeye çalışmaz.

## Sonuçları `message send` ile kullanma

```bash
openclaw directory peers list --channel slack --query "U0"
openclaw message send --channel slack --target user:U012ABCDEF --message "hello"
```

## Kanala göre kimlik biçimleri

| Kanal                             | Hedef kimlik biçimi                                                                                                            |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| WhatsApp                            | `+15551234567` (DM), `1234567890-1234567890@g.us` (grup), `120363123456789@newsletter` (Kanal/Bülten, yalnızca giden) |
| Signal                              | Yapılandırılmış takma adlar, E.164/UUID DM hedeflerine veya `group:<id>` grup hedeflerine çözümlenir                                           |
| Telegram                            | `@username` veya sayısal sohbet kimliği; gruplar sayısal kimlikler kullanır                                                                      |
| Slack                               | `user:U…` ve `channel:C…`                                                                                                  |
| Discord                             | `user:<id>` ve `channel:<id>`                                                                                              |
| Matrix (Plugin)                     | `user:@user:server`, `room:!roomId:server` veya `#alias:server`                                                              |
| Microsoft Teams (Plugin)            | `user:<id>` ve `conversation:<id>`                                                                                         |
| Zalo (Plugin)                       | Kullanıcı kimliği (Bot API)                                                                                                           |
| Zalo Personal / `zalouser` (Plugin) | `zca` kaynağından iş parçacığı kimliği (DM/grup) (`me`, `friend list`, `group list`)                                                        |

## Kendi ("ben")

```bash
openclaw directory self --channel zalouser
```

## Eşler (kişiler/kullanıcılar)

```bash
openclaw directory peers list --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory peers list --channel zalouser --limit 50
```

## Gruplar

```bash
openclaw directory groups list --channel zalouser
openclaw directory groups list --channel zalouser --query "work"
openclaw directory groups members --channel zalouser --group-id <id>
```

## İlgili

- [CLI başvurusu](/tr/cli)
