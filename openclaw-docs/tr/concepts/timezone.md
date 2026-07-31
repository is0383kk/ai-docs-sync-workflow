---
read_when:
    - Saat dilimi yönetimi için hızlı bir zihinsel model istiyorsunuz
    - Bir saat diliminin nerede ayarlanacağına veya geçersiz kılınacağına karar veriyorsunuz
summary: OpenClaw'da saat dilimlerinin göründüğü yerler — zarflar, araç yükleri, sistem istemi
title: Saat dilimleri
x-i18n:
    generated_at: "2026-07-26T23:46:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d1620b4b2cedba89bd6ab4392018cd48d0ef92a6abc1744011d482557e2c4fc
    source_path: concepts/timezone.md
    workflow: 16
---

OpenClaw, modelin sağlayıcıya özgü saatlerin bir karışımı yerine **tek bir referans zamanı** görmesi için zaman damgalarını standartlaştırır. Üç yüzey saat dilimlerini gösterir ve her birinin kendi amacı vardır:

## Üç saat dilimi yüzeyi

| Yüzey             | Gösterdiği                                                                                                 | Varsayılan                            | Yapılandırma yöntemi                                    |
| ----------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------- | ------------------------------------------------------- |
| İleti zarfları    | Gelen kanal iletilerini sarmalar: `[Signal +1555 Sun 2026-01-18 00:19:42 PST] hello`                                                       | Ana makine yereli                     | `agents.defaults.envelopeTimezone`                                      |
| Araç yükleri      | Kanal `readMessages` tarzı araçlar, ham sağlayıcı zamanını ve normalleştirilmiş `timestampMs` / `timestampUtc` değerlerini döndürür | UTC alanları her zaman bulunur        | Yapılandırılamaz; sağlayıcıya özgü zaman damgalarını korur |
| Sistem istemi     | **Yalnızca saat dilimini** içeren küçük bir `Current Date & Time` bloğu (önbellek kararlılığı için saat değeri yoktur) | `userTimezone` ayarlanmamışsa ana makine saat dilimi | `agents.defaults.userTimezone`                                      |

Sistem istemi, istem önbelleğe alımını turlar arasında kararlı tutmak için güncel saati kasıtlı olarak içermez. Aracının güncel saate ihtiyacı olduğunda `session_status` çağrısını yapar.

## Kullanıcının saat dilimini ayarlama

```json5
{
  agents: {
    defaults: {
      userTimezone: "America/Chicago",
    },
  },
}
```

`userTimezone` ayarlanmamışsa OpenClaw, çalışma zamanında ana makinenin saat dilimini `Intl.DateTimeFormat().resolvedOptions().timeZone` aracılığıyla çözümler (yapılandırmaya yazmaz). `agents.defaults.timeFormat` (`auto` | `12` | `24`), sistem istemi bölümünde değil, zarflarda ve sonraki yüzeylerde 12 saatlik/24 saatlik gösterimi denetler.

## Zarf saat dilimi değerleri

`agents.defaults.envelopeTimezone` şunları kabul eder:

- `"local"` (varsayılan) veya `"host"` - ana makinenin saat dilimi.
- `"utc"` veya `"gmt"` - UTC.
- `"user"` - çözümlenen `agents.defaults.userTimezone` (ayarlanmamışsa ana makinenin saat dilimine geri döner).
- Herhangi bir açık IANA saat dilimi dizesi, ör. `"Europe/Vienna"`.

## Ne zaman geçersiz kılınmalı

- Farklı bölgelerdeki ana makineler arasında tutarlı zaman damgaları sağlamak veya UTC ile hizalanmış tanılama/günlük çıktısıyla eşleşmek için **`"utc"` kullanın**.
- Gateway ana makinesinin çalıştığı saat diliminden bağımsız olarak zarfları yapılandırılmış kullanıcı saat dilimiyle hizalı tutmak için **`"user"` kullanın**.
- Gateway ana makinesi bir saat dilimindeyken, ana makine geçişinden bağımsız olarak zarfın her zaman başka bir saat diliminde gösterilmesi gerektiğinde **sabit bir IANA saat dilimi kullanın**.
- Zaman damgası bağlamı konuşma için yararlı olmadığında **`envelopeTimestamp: "off"` değerini ayarlayın**. Bu, zarflardan, doğrudan aracı istemi öneklerinden ve gömülü model girdisi öneklerinden mutlak zaman damgalarını kaldırır.

Davranışın tam başvurusu, sağlayıcı başına örnekler ve geçen süre biçimlendirmesi için [Tarih ve Saat](/tr/date-time) bölümüne bakın.

## İlgili

- [Tarih ve Saat](/tr/date-time) - zarf/araç/istem davranışının tamamı ve örnekler.
- [Heartbeat](/tr/gateway/heartbeat) - etkin saatler zamanlama için saat dilimini kullanır.
- [Cron İşleri](/tr/automation/cron-jobs) - cron ifadeleri zamanlama için saat dilimini kullanır.
