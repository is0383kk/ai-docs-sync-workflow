---
read_when:
    - Kanal aktarımı bağlı olduğunu bildiriyor ancak yanıtlar başarısız oluyor
    - Ayrıntılı sağlayıcı belgelerinden önce kanala özgü kontroller yapmanız gerekir
summary: Kanal başına hata belirtileri ve düzeltmelerle hızlı kanal düzeyinde sorun giderme
title: Kanal sorunlarını giderme
x-i18n:
    generated_at: "2026-07-26T23:51:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3891595e4b5aca9de7997a6e908fa1c9246579032bfdfa1656a6992d644c3ecc
    source_path: channels/troubleshooting.md
    workflow: 16
---

Bir kanal bağlandığı hâlde davranış hatalıysa bu sayfayı kullanın.

## Komut sıralaması

Önce bunları sırayla çalıştırın:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Sağlıklı temel durum:

- `Runtime: running`
- `Connectivity probe: ok`
- `Capability: read-only`, `write-capable` veya `admin-capable`
- Kanal yoklaması, aktarımın bağlı olduğunu ve desteklenen yerlerde `works` veya `audit ok` gösterir

## Güncellemeden sonra

Güncellemeden sonra Telegram, iMessage, BlueBubbles dönemi yapılandırmaları veya başka bir Plugin kanalı kaybolursa bunu kullanın.

```bash
openclaw status --all
openclaw doctor --fix
openclaw gateway restart
openclaw status --all
```

`openclaw
status --all` içinde `plugin load failed: dependency tree corrupted; run openclaw doctor --fix` arayın. Bu, kanalın yapılandırıldığı ancak Plugin kurulumunun/yüklemesinin kanalı kaydetmek yerine bozuk bir bağımlılık ağacına takıldığı anlamına gelir. `openclaw doctor --fix`, eski Plugin çalışma zamanı bağımlılık sembolik bağlantılarını ve eski kimlik doğrulama gölgelerini temizler; ardından `openclaw gateway restart` temiz durumu yeniden yükler.

## WhatsApp

### WhatsApp hata belirtileri

| Belirti                             | En hızlı kontrol                                       | Düzeltme                                                                                                                              |
| ----------------------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Bağlı ancak DM yanıtı yok         | `openclaw pairing list whatsapp`                    | Gönderene onay verin veya DM politikasını/izin listesini değiştirin.                                                                                    |
| Grup mesajları yok sayılıyor              | Yapılandırmada `requireMention` + bahsetme kalıplarını kontrol edin | Bottan bahsedin veya bu grup için bahsetme politikasını gevşetin.                                                                          |
| QR oturum açma işlemi 408 ile zaman aşımına uğruyor         | Gateway `HTTPS_PROXY` / `HTTP_PROXY` ortamını kontrol edin      | Erişilebilir bir proxy ayarlayın; `NO_PROXY` öğesini yalnızca atlamalar için kullanın.                                                                         |
| Rastgele bağlantı kesilmesi/yeniden oturum açma döngüleri     | `openclaw channels status --probe` + günlükler           | Yakın zamandaki yeniden bağlantılar şu anda bağlı olunsa bile işaretlenir; günlükleri izleyin, Gateway'i yeniden başlatın, ardından dalgalanma devam ederse yeniden bağlayın. |
| `status=408 Request Time-out` döngüsü  | Yoklama, günlükler, doctor, ardından Gateway durumu            | Önce ana makine bağlantısını/zamanlamasını düzeltin; kimlik doğrulama verilerini yedekleyin ve döngü sürerse hesabı yeniden bağlayın.                                   |
| Yanıtlar saniyeler/dakikalar sonra geliyor | `openclaw doctor --fix`                             | Doctor, Gateway olay döngüsünün performansını düşürdüğü doğrulanan eski yerel TUI istemcilerini durdurur.                                    |

Tüm sorun giderme bilgileri: [WhatsApp sorun giderme](/tr/channels/whatsapp#troubleshooting)

## Telegram

### Telegram hata belirtileri

| Belirti                              | En hızlı kontrol                                    | Düzeltme                                                                                                                    |
| ------------------------------------ | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `/start` ancak kullanılabilir yanıt akışı yok    | `openclaw pairing list telegram`                 | Eşleştirmeyi onaylayın veya DM politikasını değiştirin.                                                                                   |
| Bot çevrimiçi ancak grup sessiz kalıyor    | Bahsetme gereksinimini ve bot gizlilik modunu doğrulayın  | Grup görünürlüğü için gizlilik modunu devre dışı bırakın veya bottan bahsedin.                                                              |
| Ağ hatalarıyla gönderim başarısız oluyor    | Telegram API çağrısı hataları için günlükleri inceleyin      | `api.telegram.org` için DNS/IPv6/proxy yönlendirmesini düzeltin.                                                                      |
| Başlangıçta `getMe returned 401` bildiriliyor | Yapılandırılmış token kaynağını kontrol edin                    | BotFather token'ını yeniden kopyalayın veya oluşturun ve `botToken`, `tokenFile` ya da varsayılan hesap `TELEGRAM_BOT_TOKEN` değerini güncelleyin. |
| Yoklama takılıyor veya yeniden bağlantı yavaş gerçekleşiyor  | Yoklama tanılaması için `openclaw logs --follow` | Yükseltin; kalıcı takılmalar genellikle proxy/DNS/IPv6 sorununa işaret eder.                                                            |
| `setMyCommands` başlangıçta reddediliyor  | Günlüklerde `BOT_COMMANDS_TOO_MUCH` arayın         | Plugin/skill/özel Telegram komutlarını azaltın veya yerel menüleri devre dışı bırakın.                                                  |
| Yükseltmeden sonra izin listesi sizi engelliyor    | `openclaw security audit` ve yapılandırma izin listeleri  | `openclaw doctor --fix` komutunu çalıştırın veya `@username` değerini sayısal gönderici kimlikleriyle değiştirin.                                            |

Tüm sorun giderme bilgileri: [Telegram sorun giderme](/tr/channels/telegram#troubleshooting)

## Discord

### Discord hata belirtileri

| Belirti                                   | En hızlı kontrol                                                                                                                | Düzeltme                                                                                                                                                                                                                                                                   |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bot çevrimiçi ancak sunucuda yanıt yok           | `openclaw channels status --probe`                                                                                           | Sunucuya/kanala izin verin ve mesaj içeriği intent'ini doğrulayın.                                                                                                                                                                                                                |
| Grup mesajları yok sayılıyor                    | Bahsetme geçidi nedeniyle bırakılan iletiler için günlükleri kontrol edin                                                                                          | Bottan bahsedin veya sunucu/kanal `requireMention: false` değerini ayarlayın.                                                                                                                                                                                                             |
| Yazma/token kullanımı var ancak Discord mesajı yok | Bunun bir ortam odası olayı mı yoksa modelin `message(action=send)` öğesini atladığı, katılımın etkinleştirildiği bir `message_tool` odası mı olduğunu kontrol edin | Bastırılan son yük meta verileri için ayrıntılı Gateway günlüğünü inceleyin, `messages.groupChat.unmentionedInbound` değerini doğrulayın, [Ortam odası olayları](/tr/channels/ambient-room-events) sayfasını okuyun veya normal grup istekleri için `messages.groupChat.visibleReplies: "automatic"` değerini koruyun. |
| DM yanıtları eksik                        | `openclaw pairing list discord`                                                                                              | DM eşleştirmesini onaylayın veya DM politikasını ayarlayın.                                                                                                                                                                                                                               |

Tüm sorun giderme bilgileri: [Discord sorun giderme](/tr/channels/discord#troubleshooting)

## Slack

### Slack hata belirtileri

| Belirti                                | En hızlı kontrol                             | Düzeltme                                                                                                                                                  |
| -------------------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Soket modu bağlı ancak yanıt yok | `openclaw channels status --probe`        | Uygulama token'ını, bot token'ını ve gerekli kapsamları doğrulayın; SecretRef destekli kurulumlarda `botTokenStatus` / `appTokenStatus = configured_unavailable` durumlarını izleyin. |
| DM'ler engelleniyor                            | `openclaw pairing list slack`             | Eşleştirmeyi onaylayın veya DM politikasını gevşetin.                                                                                                                  |
| Kanal mesajı yok sayılıyor                | `groupPolicy` ve kanal izin listesini kontrol edin | Kanala izin verin veya politikayı `open` olarak değiştirin.                                                                                                        |

Tüm sorun giderme bilgileri: [Slack sorun giderme](/tr/channels/slack#troubleshooting)

## iMessage

### iMessage hata belirtileri

| Belirti                              | En hızlı kontrol                                           | Düzeltme                                                                   |
| ------------------------------------ | ------------------------------------------------------- | --------------------------------------------------------------------- |
| `imsg` eksik veya macOS dışındaki sistemlerde başarısız oluyor | `openclaw channels status --probe --channel imessage`   | OpenClaw'u Messages'ın bulunduğu Mac'te çalıştırın veya `cliPath` için bir SSH sarmalayıcısı kullanın. |
| macOS'te gönderilebiliyor ancak alınamıyor     | Messages otomasyonu için macOS gizlilik izinlerini kontrol edin | TCC izinlerini yeniden verin ve kanal işlemini yeniden başlatın.                 |
| DM göndericisi engelleniyor                    | `openclaw pairing list imessage`                        | Eşleştirmeyi onaylayın veya izin listesini güncelleyin.                                  |

Tüm sorun giderme bilgileri: [iMessage sorun giderme](/tr/channels/imessage#troubleshooting)

## Signal

### Signal hata belirtileri

| Belirti                         | En hızlı kontrol                              | Düzeltme                                                      |
| ------------------------------- | ------------------------------------------ | -------------------------------------------------------- |
| Daemon erişilebilir ancak bot sessiz | `openclaw channels status --probe`         | `signal-cli` daemon URL'sini/hesabını ve alım modunu doğrulayın. |
| DM engelleniyor                      | `openclaw pairing list signal`             | Gönderene onay verin veya DM politikasını ayarlayın.                      |
| Grup yanıtları tetiklenmiyor    | Grup izin listesini ve bahsetme kalıplarını kontrol edin | Göndericiyi/grubu ekleyin veya geçidi gevşetin.                       |

Tüm sorun giderme bilgileri: [Signal sorun giderme](/tr/channels/signal#troubleshooting)

## QQ Bot

### QQ Bot hata belirtileri

| Belirti                         | En hızlı kontrol                               | Düzeltme                                                             |
| ------------------------------- | ------------------------------------------- | --------------------------------------------------------------- |
| Bot "Mars'a gitti" yanıtını veriyor      | Yapılandırmada `appId` ve `clientSecret` değerlerini doğrulayın | Kimlik bilgilerini ayarlayın veya Gateway'i yeniden başlatın.                         |
| Gelen mesaj yok             | `openclaw channels status --probe`          | QQ Open Platform'da kimlik bilgilerini doğrulayın.                     |
| Ses yazıya dökülmüyor           | STT sağlayıcı yapılandırmasını kontrol edin                   | `channels.qqbot.stt` veya `tools.media.audio` yapılandırın.          |
| Proaktif mesajlar ulaşmıyor | QQ platformu etkileşim gereksinimlerini kontrol edin  | QQ, yakın zamanda etkileşim olmadığında bot tarafından başlatılan mesajları engelleyebilir. |

Tüm sorun giderme bilgileri: [QQ Bot sorun giderme](/tr/channels/qqbot#troubleshooting)

## Matrix

### Matrix hata belirtileri

| Belirti                                  | En hızlı kontrol                         | Düzeltme                                                                                     |
| ---------------------------------------- | ---------------------------------------- | -------------------------------------------------------------------------------------------- |
| Oturum açık ancak oda mesajlarını yok sayıyor | `openclaw channels status --probe`     | `groupPolicy`, oda izin listesini ve bahsetme kısıtlamasını kontrol edin.                  |
| DM'ler işlenmiyor                        | `openclaw pairing list matrix`         | Göndereni onaylayın veya DM politikasını ayarlayın.                                           |
| Şifrelenmiş odalar çalışmıyor            | `openclaw matrix verify status`        | Cihazı yeniden doğrulayın, ardından `openclaw matrix verify backup status` öğesini kontrol edin.  |
| Yedekten geri yükleme bekliyor/bozuk     | `openclaw matrix verify backup status` | `openclaw matrix verify backup restore` komutunu çalıştırın veya bir kurtarma anahtarıyla yeniden çalıştırın. |
| Çapraz imzalama/önyükleme yanlış görünüyor | `openclaw matrix verify bootstrap`     | Gizli veri depolamasını, çapraz imzalamayı ve yedekleme durumunu tek seferde onarın.           |

Tam kurulum ve yapılandırma: [Matrix](/tr/channels/matrix)

## İlgili

- [Eşleştirme](/tr/channels/pairing)
- [Kanal yönlendirme](/tr/channels/channel-routing)
- [Gateway sorun giderme](/tr/gateway/troubleshooting)
