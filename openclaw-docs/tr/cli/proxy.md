---
read_when:
    - Dağıtımdan önce operatör tarafından yönetilen proxy yönlendirmesini doğrulamanız gerekir
    - Hata ayıklama için OpenClaw aktarım trafiğini yerel olarak yakalamanız gerekiyor
    - Hata ayıklama proxy oturumlarını, blob’ları veya yerleşik sorgu ön ayarlarını incelemek istiyorsunuz
summary: Operatör tarafından yönetilen proxy doğrulaması ve yerel hata ayıklama proxy yakalama inceleyicisi dâhil olmak üzere `openclaw proxy` için CLI başvurusu
title: Vekil Sunucu
x-i18n:
    generated_at: "2026-07-26T23:13:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91583f785032bfffe455a1963804108550f6fbb735ac4de1dd91d0ca5ae0df35
    source_path: cli/proxy.md
    workflow: 16
---

# `openclaw proxy`

Operatör tarafından yönetilen proxy yönlendirmesini doğrulayın veya yerel açık hata ayıklama proxy'sini çalıştırıp yakalanan trafiği inceleyin.

```bash
openclaw proxy validate [--json] [--proxy-url <url>] [--proxy-ca-file <path>] [--allowed-url <url>] [--denied-url <url>] [--apns-reachable] [--apns-authority <url>] [--timeout-ms <ms>]
openclaw proxy start [--host <host>] [--port <port>]
openclaw proxy run [--host <host>] [--port <port>] -- <cmd...>
openclaw proxy coverage
openclaw proxy sessions [--limit <count>]
openclaw proxy query --preset <name> [--session <id>]
openclaw proxy blob --id <blobId>
openclaw proxy purge
```

`validate`, operatör tarafından yönetilen bir ileri proxy için ön kontrol gerçekleştirir. Diğerleri, aktarım düzeyinde inceleme amaçlı hata ayıklama araçlarıdır: yerel bir yakalama proxy'si başlatır, bunun üzerinden bir alt komut çalıştırır, yakalama oturumlarını listeler, trafik örüntülerini sorgular, yakalanan blob'ları okur ve yerel yakalama verilerini temizler.

## Doğrulama

Geçerli operatör yönetimli proxy URL'sini şu öncelik sırasıyla denetler: `--proxy-url`, yapılandırma (`proxy.proxyUrl`) veya `OPENCLAW_PROXY_URL`. Hiçbir proxy etkinleştirilip yapılandırılmamışsa bir yapılandırma sorunu bildirir; yapılandırmaya dokunmadan tek seferlik ön kontrol gerçekleştirmek için `--proxy-url` iletin.

Yönetilen proxy URL'leri, düz bir ileri proxy dinleyicisi için `http://`; OpenClaw'ın proxy isteklerini göndermeden önce proxy uç noktasına TLS bağlantısı açması gerektiğinde ise `https://` kullanır. Bu TLS bağlantısı için özel bir CA'ya güvenmek üzere `--proxy-ca-file` kullanın.

Varsayılan olarak şunları çalıştırır:

- `https://example.com/` için bir **izin verilen** denetim (`--allowed-url` ile geçersiz kılın veya ekleyin; yinelenebilir)
- geçici bir geri döngü kanaryası için bir **reddedilen** denetim (`--denied-url` ile geçersiz kılın; yinelenebilir)

Özel `--denied-url` hedefleri kapalı durumda başarısız olur: dağıtıma özgü bir ret sinyalini bağımsız olarak doğrulayamadığınız sürece hem HTTP yanıtları hem de belirsiz aktarım hataları başarısızlık sayılır. Aktarım hatasının engelleme kanıtı olarak kabul edildiği tek hedef, yerleşik geri döngü kanaryasıdır.

Proxy üzerinden bir APNs HTTP/2 CONNECT tüneli açıp korumalı alan APNs hizmetinin yanıt verdiğini doğrulamak için ayrıca `--apns-reachable` ekleyin. Yoklama, kasıtlı olarak geçersiz bir sağlayıcı token'ı gönderir; dolayısıyla APNs `403 InvalidProviderToken` yanıtı başarılı bir erişilebilirlik sinyali sayılır (başarısızlık değil).

### Seçenekler

| Bayrak                     | Etki                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `--json`                 | makine tarafından okunabilir JSON yazdırır                                                                                        |
| `--proxy-url <url>`      | yapılandırma veya ortam yerine bu `http://`/`https://` proxy URL'sini doğrular                                              |
| `--proxy-ca-file <path>` | bir HTTPS proxy uç noktasının TLS doğrulaması için bu PEM CA dosyasına güvenir                                             |
| `--allowed-url <url>`    | proxy üzerinden başarılı olması beklenen hedef (yinelenebilir)                                                     |
| `--denied-url <url>`     | proxy tarafından engellenmesi beklenen hedef (yinelenebilir)                                                       |
| `--apns-reachable`       | ayrıca korumalı alan APNs HTTP/2 hizmetine proxy üzerinden erişilebildiğini doğrular                                                     |
| `--apns-authority <url>` | yoklanacak APNs yetki alanı (varsayılan `https://api.sandbox.push.apple.com`; üretim `https://api.push.apple.com`) |
| `--timeout-ms <ms>`      | istek başına zaman aşımı                                                                                                |

Proxy yapılandırması veya hedef denetimleri başarısız olduğunda 1 koduyla çıkar.

Dağıtım rehberliği ve ret semantiği için [Ağ Proxy'si](/tr/security/network-proxy) bölümüne bakın.

## Hata ayıklama proxy'si

`start` yerel bir yakalama proxy'si başlatır ve URL'sini, CA sertifikası yolunu ve yakalama veritabanı yolunu yazdırır; Ctrl+C ile durdurun. `--host` ayarlanmadığı sürece varsayılan olarak `127.0.0.1` adresine bağlanır.

`run` yerel bir hata ayıklama proxy'si başlatır, ardından proxy ortamı uygulanmış şekilde ve kendi yakalama oturumunda `<cmd...>` komutunu (`--` sonrasında) çalıştırır.

Hata ayıklama proxy'sinin doğrudan yukarı akış iletimi, tanılama amacıyla yukarı akış soketlerini açar. OpenClaw yönetilen proxy modu etkin olduğunda, proxy istekleri ve CONNECT tünelleri için doğrudan iletim varsayılan olarak devre dışıdır; `OPENCLAW_DEBUG_PROXY_ALLOW_DIRECT_CONNECT_WITH_MANAGED_PROXY=1` yalnızca onaylanmış yerel tanılamalar için ayarlayın.

`coverage`, hangi aktarımların yakalandığını, yalnızca proxy üzerinden geçtiğini veya kapsam dışında kaldığını gösteren bir JSON raporu (`summary` + aktarım başına `entries`) yazdırır.

`sessions` son yakalama oturumlarını listeler (`--limit`, varsayılan 20).

`query --preset <name>`, yakalanan trafik üzerinde isteğe bağlı olarak `--session <id>` kapsamıyla sınırlandırılmış yerleşik bir sorgu çalıştırır. Ön ayarlar:

- `double-sends`
- `retry-storms`
- `cache-busting`
- `ws-duplicate-frames`
- `missing-ack`
- `error-bursts`

`blob --id <blobId>`, yakalanmış bir yük blob'unun ham içeriğini yazdırır.

`purge`, yakalanan tüm trafik meta verilerini ve blob'ları siler. Yakalamalar yerel hata ayıklama verileridir; işiniz bittiğinde temizleyin.

## İlgili

- [CLI referansı](/tr/cli)
- [Ağ Proxy'si](/tr/security/network-proxy)
- [Güvenilir proxy kimlik doğrulaması](/tr/gateway/trusted-proxy-auth)
