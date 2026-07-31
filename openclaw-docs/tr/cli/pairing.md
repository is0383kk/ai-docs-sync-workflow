---
read_when:
    - Eşleştirme modundaki DM'leri kullanıyorsunuz ve gönderenleri onaylamanız gerekiyor
summary: '`openclaw pairing` için CLI referansı (eşleştirme isteklerini onaylama/listeleme)'
title: Eşleştirme
x-i18n:
    generated_at: "2026-07-26T23:36:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e4c6c53f1a3eefe50b4b7a45fa535e9a05faabb50df1ba5195a7635ee13d9da0
    source_path: cli/pairing.md
    workflow: 16
---

# `openclaw pairing`

Eşleştirmeyi destekleyen kanallar için DM eşleştirme isteklerini onaylayın veya inceleyin (yalnızca sohbet DM'leri; Node/cihaz eşleştirmesi `openclaw devices` kullanır).

İlgili: [Eşleştirme akışı](/tr/channels/pairing)

Aynı bekleyen istekler, Kontrol Arayüzünde **Settings →
Channels → DM access requests** altında incelenebilir. Kontrol Arayüzü; onaylama, isteğe bağlı olarak
istek sahibini bilgilendirme ve kapatma işlemlerini destekler. Kapatma, mevcut isteği kaldırır ancak
göndereni kalıcı olarak engellemez.

## Komutlar

```bash
openclaw pairing list telegram
openclaw pairing list --channel telegram --account work
openclaw pairing list telegram --json

openclaw pairing approve <code>
openclaw pairing approve telegram <code>
openclaw pairing approve --channel telegram --account work <code> --notify
```

## `pairing list`

Bir kanal için bekleyen eşleştirme isteklerini listeleyin.

| Seçenek                 | Açıklama                              |
| ----------------------- | ------------------------------------- |
| `[channel]`      | konumsal kanal kimliği                |
| `--channel <channel>`      | açık kanal kimliği                    |
| `--account <accountId>`      | çok hesaplı kanallar için hesap kimliği |
| `--json`      | makine tarafından okunabilir çıktı    |

Eşleştirme özelliğine sahip birden fazla kanal yapılandırılmışsa bir kanalı konumsal olarak veya `--channel` ile belirtin. Kanal kimliği geçerli olduğu sürece uzantı kanalları da çalışır.

## `pairing approve`

Bekleyen bir eşleştirme kodunu onaylayın ve ilgili gönderene izin verin.

Kullanım:

- `openclaw pairing approve <channel> <code>`
- `openclaw pairing approve --channel <channel> <code>`
Tam olarak bir eşleştirme özellikli kanal yapılandırıldığında
- `openclaw pairing approve <code>`

Seçenekler: `--channel <channel>`, `--account <accountId>`, `--notify` (aynı kanaldan istek sahibine bir onay gönderir).

### Sahip başlangıç yapılandırması

Bir eşleştirme kodunu onayladığınızda `commands.ownerAllowFrom` boşsa CLI, onaylanan göndereni `telegram:123456789` gibi kanal kapsamlı bir girdi kullanarak komut sahibi olarak da kaydeder. Bu yalnızca ilk sahibin başlangıç yapılandırmasını yapar; sonraki eşleştirme onayları `commands.ownerAllowFrom` değerini hiçbir zaman değiştirmez veya genişletmez. Kontrol Arayüzü, bu yetki yükseltmesini otomatik olarak uygulamak yerine ayrı bir `operator.admin` korumalı onay kutusu olarak sunar.

Komut sahibi; yalnızca sahibin kullanabildiği komutları çalıştırmasına ve `/diagnostics`, `/export-session`, `/export-trajectory`, `/config` ve exec onayları gibi tehlikeli eylemleri onaylamasına izin verilen insan operatör hesabıdır. Eşleştirme yalnızca bir gönderenin agent ile iletişim kurmasını sağlar; bu tek seferlik başlangıç yapılandırması dışında tek başına sahip ayrıcalıkları vermez.

Bir göndereni bu başlangıç yapılandırması eklenmeden önce onayladıysanız `openclaw doctor` komutunu çalıştırın; komut sahibi yapılandırılmamışsa uyarır ve sorunu düzeltmek için kullanılacak tam `openclaw config set commands.ownerAllowFrom ...` komutunu gösterir.

## İlgili

- [CLI referansı](/tr/cli)
- [Kanal eşleştirmesi](/tr/channels/pairing)
