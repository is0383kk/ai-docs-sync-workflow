---
read_when:
    - Bir mobil Node uygulamasını bir Gateway ile hızlıca eşleştirmek istiyorsunuz
    - Uzakta/elle paylaşım için kurulum kodu çıktısına ihtiyacınız var
summary: '`openclaw qr` için CLI referansı (mobil eşleştirme QR kodu + kurulum kodu oluşturma)'
title: QR
x-i18n:
    generated_at: "2026-07-26T23:16:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9d60a58126eae7eec5979f28bb511a09fa52b68cdd73727fca0b2de74efa84a
    source_path: cli/qr.md
    workflow: 16
---

# `openclaw qr`

Geçerli Gateway yapılandırmanızdan mobil eşleştirme QR kodu ve kurulum kodu oluşturun.

```bash
openclaw qr
openclaw qr --setup-code-only
openclaw qr --json
openclaw qr --remote
openclaw qr --limited
openclaw qr --url wss://gateway.example/ws
```

Resmî OpenClaw iOS ve Android uygulamaları, kurulum kodu meta verileri eşleştiğinde otomatik olarak bağlanır. Bir istek beklemede kalırsa (örneğin resmî olmayan bir istemci veya eşleşmeyen meta veriler nedeniyle), isteği inceleyip onaylayın:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

## Seçenekler

- `--remote`: `gateway.remote.url` tercih edilir; bu URL ayarlanmamışsa `gateway.tailscale.mode=serve|funnel` değerine geri döner. `device-pair` Plugin `publicUrl` değerini yok sayar.
- `--url <url>`: yükte kullanılan Gateway URL'sini geçersiz kıl
- `--public-url <url>`: yükte kullanılan genel URL'yi geçersiz kıl
- `--token <token>`: önyükleme akışının kimlik doğrulamasında kullandığı Gateway belirtecini geçersiz kıl
- `--password <password>`: önyükleme akışının kimlik doğrulamasında kullandığı Gateway parolasını geçersiz kıl
- `--limited`: devredilen operatör belirtecine yönetimsel Gateway erişimini dahil etme
- `--setup-code-only`: yalnızca kurulum kodunu yazdır
- `--no-ascii`: ASCII QR oluşturmayı atla
- `--json`: JSON çıktısı üret (`setupCode`, `gatewayUrl`, isteğe bağlı `gatewayUrls`, `auth`, `access`, isteğe bağlı `accessDowngraded`, `urlSource`)

`--token` ve `--password` birlikte kullanılamaz.

## Kurulum kodunun içeriği

Kurulum kodu, paylaşılan Gateway belirteci/parolası yerine opak ve kısa ömürlü bir `bootstrapToken` taşır. Bir `wss://` uç noktası (veya aynı ana makinedeki geri döngü) için varsayılan önyükleme akışı şunları oluşturur:

- `scopes: []` içeren birincil bir `node` belirteci
- `operator.admin`, `operator.approvals`, `operator.read`, `operator.talk.secrets` ve `operator.write` içeren tam bir yerel mobil `operator` devir belirteci

Operatör devrinden `operator.admin` değerini çıkarırken aynı Node belirtecini korumak için `--limited` kullanın. Eşleştirmeyi değiştirme kapsamı hiçbir zaman kurulum koduyla devredilmez.

Düz metin LAN `ws://` kurulumu kullanılabilir olmaya devam eder ancak ağdaki bir gözlemci taşıyıcı önyükleme belirtecini yakalayıp ondan önce davranabileceğinden OpenClaw sınırlı profili otomatik olarak kullanır. Tam erişim elde etmek için `wss://` veya Tailscale Serve'ü yapılandırıp yeni bir kod oluşturun.

## Gateway URL'sinin çözümlenmesi

Mobil eşleştirme, Tailscale/genel `ws://` Gateway URL'leri için güvenli biçimde başarısız olur: bunlar için Tailscale Serve/Funnel veya bir `wss://` Gateway URL'si kullanın. Özel LAN adresleri ve `.local` Bonjour ana makineleri, yukarıda açıklandığı gibi sınırlı operatör erişimiyle düz `ws://` üzerinden desteklenmeye devam eder.

Seçilen Gateway URL'si `gateway.bind=lan` kaynağından geldiğinde OpenClaw, kalıcı `tailscale serve status --json` rotalarını da denetler. Etkin Gateway'in geri döngü bağlantı noktasına vekâlet eden tüm HTTPS Serve kökleri, geri dönüş seçeneği olarak eklenir. QR komutu bu geri dönüşü yalnızca `lan` için ekler; `custom` ve `tailnet` açıkça duyurulan rotalarını korur. Geçerli iOS istemcileri duyurulan rotaları sırayla yoklar ve erişilebilen ilk rotayı kaydeder; eski istemciler için eski `url` alanı değiştirilmeden kalır.

`--remote` ile `gateway.remote.url` veya `gateway.tailscale.mode=serve|funnel` değerlerinden biri gereklidir.

## Kimlik doğrulama çözümlemesi (`--remote` olmadan)

CLI kimlik doğrulama geçersiz kılma seçeneği aktarılmadığında, yerel Gateway kimlik doğrulama SecretRef'leri şu şekilde çözümlenir:

| Koşul                                                                                                                        | Çözümlenen değer                          |
| ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `gateway.auth.mode="token"` veya kazanan bir parola kaynağı olmayan çıkarılmış mod                                                    | `gateway.auth.token`                        |
| `gateway.auth.mode="password"` veya kimlik doğrulama/ortam kaynaklı kazanan bir belirteç olmayan çıkarılmış mod                          | `gateway.auth.password`                        |
| Hem `gateway.auth.token` hem de `gateway.auth.password` yapılandırılmışsa (SecretRef'ler dâhil) ve `gateway.auth.mode` ayarlanmamışsa | başarısız olur; `gateway.auth.mode` değerini açıkça ayarlayın |

## Kimlik doğrulama çözümlemesi (`--remote`)

Etkin biçimde kullanılan uzak kimlik bilgileri SecretRef olarak yapılandırılmışsa ve ne `--token` ne de `--password` aktarılmışsa komut, bunları etkin Gateway anlık görüntüsünden çözümler. Gateway kullanılamıyorsa komut hemen başarısız olur.

<Note>
Bu komut yolu, `secrets.resolve` RPC yöntemini destekleyen bir Gateway gerektirir. Eski Gateway'ler bilinmeyen yöntem hatası döndürür.
</Note>

## İlgili içerikler

- [CLI başvurusu](/tr/cli)
- [Cihazlar](/tr/cli/devices)
- [Eşleştirme](/tr/cli/pairing)
