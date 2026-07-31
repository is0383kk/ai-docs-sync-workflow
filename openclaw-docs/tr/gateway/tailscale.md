---
read_when:
    - Gateway Kontrol Kullanıcı Arayüzünü localhost dışından erişime açma
    - Tailnet veya herkese açık pano erişimini otomatikleştirme
summary: Gateway panosu için entegre Tailscale Serve/Funnel
title: Tailscale
x-i18n:
    generated_at: "2026-07-27T00:00:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e201a64ac427994401fae1b934d94e0c5afe976b4acd34d45b059978f5f1807e
    source_path: gateway/tailscale.md
    workflow: 16
---

OpenClaw, Gateway panosu ve WebSocket portu için Tailscale **Serve** (tailnet) veya **Funnel** (genel) yapılandırmasını otomatik olarak yapabilir. Böylece gateway geri döngüye bağlı kalırken Tailscale; HTTPS, yönlendirme ve (Serve için) kimlik üst bilgilerini sağlar.

## Modlar

`gateway.tailscale.mode`:

| Mod             | Davranış                                                                    |
| --------------- | --------------------------------------------------------------------------- |
| `serve`         | `tailscale serve` üzerinden yalnızca tailnet'e açık Serve. Gateway, `127.0.0.1` üzerinde kalır. |
| `funnel`        | `tailscale funnel` üzerinden genel HTTPS. Paylaşılan parola gerektirir.            |
| `off` (varsayılan) | Tailscale otomasyonu yoktur.                                                    |

Durum ve denetim çıktısı, bu OpenClaw Serve/Funnel modu için **Tailscale erişimini** kullanır. `off`, OpenClaw'ın Serve veya Funnel'ı yönetmediği anlamına gelir; yerel Tailscale arka plan programının durdurulduğu veya oturumunun kapatıldığı anlamına gelmez.

## Yapılandırma örnekleri

### Yalnızca tailnet (Serve)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" },
  },
}
```

Açın: `https://<magicdns>/` (veya yapılandırdığınız `gateway.controlUi.basePath`)

Control UI'yi cihaz ana bilgisayar adı yerine adlandırılmış bir Tailscale Hizmeti üzerinden kullanıma açmak için `gateway.tailscale.serviceName` değerini Hizmet adına ayarlayın:

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve", serviceName: "svc:openclaw" },
  },
}
```

Başlatma işlemi daha sonra cihaz ana bilgisayar adı yerine Hizmet URL'sini `https://openclaw.<tailnet-name>.ts.net/` olarak bildirir. Tailscale Hizmetleri, ana bilgisayarın tailnet'inizde onaylı ve etiketlenmiş bir Node olmasını gerektirir. Bunu etkinleştirmeden önce Tailscale'de etiketi yapılandırıp Hizmeti onaylayın; aksi takdirde gateway başlatılırken `tailscale serve --service=...` başarısız olur.

### Yalnızca tailnet (Tailnet IP'sine bağlanma)

Gateway'in Serve/Funnel olmadan doğrudan Tailnet IP'sini dinlemesi için bunu kullanın:

```json5
{
  gateway: {
    bind: "tailnet",
    auth: { mode: "token", token: "your-token" },
  },
}
```

Başka bir Tailnet cihazından bağlanın:

- Control UI: `http://<tailscale-ip>:18789/`
- WebSocket: `ws://<tailscale-ip>:18789`

<Note>
Bağlanılabilir bir Tailnet IPv4 adresi mevcut olduğunda Gateway, kimliği doğrulanmış aynı ana bilgisayardaki istemciler için ayrıca `http://127.0.0.1:18789` gerektirir. Başlatma sırasında kullanılabilir bir Tailnet adresi yoksa yalnızca geri döngüye geri döner; doğrudan Tailnet erişimi eklemek için Tailscale kullanılabilir hâle geldikten sonra yeniden başlatın. Her iki yol da LAN veya genel erişim eklemez.
</Note>

### Genel internet (Funnel + paylaşılan parola)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "funnel" },
    auth: { mode: "password", password: "replace-me" },
  },
}
```

Parolayı diske kaydetmek yerine `OPENCLAW_GATEWAY_PASSWORD` kullanmayı tercih edin.

## CLI örnekleri

```bash
openclaw gateway --tailscale serve
openclaw gateway --tailscale funnel --auth password
```

## Kimlik doğrulama

`gateway.auth.mode`, el sıkışmayı denetler:

| Mod                                                    | Kullanım alanı                                                                       |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------ |
| `none`                                                 | Yalnızca özel giriş                                                                 |
| `token` (`OPENCLAW_GATEWAY_TOKEN` ayarlandığında varsayılan) | Paylaşılan belirteç                                                                        |
| `password`                                             | `OPENCLAW_GATEWAY_PASSWORD` veya yapılandırma üzerinden paylaşılan gizli değer                             |
| `trusted-proxy`                                        | Kimlik duyarlı ters proxy; bkz. [Güvenilir Proxy Kimlik Doğrulaması](/tr/gateway/trusted-proxy-auth) |

### Tailscale kimlik üst bilgileri (yalnızca Serve)

`tailscale.mode: "serve"` olduğunda ve `gateway.auth.allowTailscale`, `true` olarak ayarlandığında Control UI/WebSocket kimlik doğrulaması, belirteç/parola yerine Tailscale kimlik üst bilgilerini (`tailscale-user-login`) kullanabilir. OpenClaw, isteğin `x-forwarded-for` adresini yerel Tailscale arka plan programı (`tailscale whois`) üzerinden çözümleyip kabul etmeden önce üst bilgi oturum açma bilgisiyle eşleştirerek üst bilgiyi doğrular. Bir istek yalnızca geri döngüden gelip Tailscale'in `x-forwarded-for`, `x-forwarded-proto` ve `x-forwarded-host` üst bilgilerini taşıdığında bu yolu kullanmaya hak kazanır.

Bu belirteçsiz akış, gateway ana bilgisayarının güvenilir olduğunu varsayar. Güvenilmeyen yerel kod aynı ana bilgisayarda çalışabiliyorsa `gateway.auth.allowTailscale: false` ayarını yapın ve bunun yerine belirteç/parola kimlik doğrulamasını zorunlu kılın.

Atlamanın kapsamı:

- Yalnızca Control UI WebSocket kimlik doğrulama yüzeyi için geçerlidir. HTTP API uç noktaları (`/v1/*`, `/tools/invoke`, `/api/channels/*` vb.) Tailscale kimlik üst bilgisi kimlik doğrulamasını hiçbir zaman kullanmaz; her zaman gateway'in normal HTTP kimlik doğrulama modunu izler.
- Tarayıcı cihaz kimliğini zaten taşıyan Control UI operatör oturumlarında, doğrulanmış bir Tailscale kimliği önyükleme belirteci/QR eşleştirme gidiş dönüşünü atlar.
- Cihaz kimliğinin kendisini atlamaz: cihazsız istemciler yine reddedilir ve Node rolü bağlantıları yine normal eşleştirme ve kimlik doğrulama kontrollerinden geçer.

## Notlar

- Tailscale Serve/Funnel, `tailscale` CLI'nin kurulu ve oturum açmış olmasını gerektirir.
- `tailscale.mode: "funnel"`, genel erişimi önlemek için kimlik doğrulama modu `password` olmadığı sürece başlatılmayı reddeder.
- `gateway.tailscale.serviceName` yalnızca Serve modu için geçerlidir ve `tailscale serve --service=<name>` öğesine iletilir. Değer, Tailscale'in `svc:<dns-label>` biçimini kullanmalıdır; örneğin `svc:openclaw`. Tailscale, Hizmet ana bilgisayarlarının etiketlenmiş Node'lar olmasını gerektirir ve Serve bunu yayımlamadan önce Hizmetin yönetici konsolunda onaylanması gerekebilir.
- `gateway.tailscale.resetOnExit`, kapatma sırasında `tailscale serve`/`tailscale funnel` yapılandırmasını geri alır.
- `gateway.tailscale.preserveFunnel: true`, haricî olarak yapılandırılmış bir `tailscale funnel` rotasını gateway yeniden başlatmaları boyunca etkin tutar. `mode: "serve"` ile OpenClaw, Serve'i yeniden uygulamadan önce `tailscale funnel status` denetimi yapar ve bir Funnel rotası gateway portunu zaten kapsıyorsa bunu atlar. OpenClaw tarafından yönetilen Funnel'ın yalnızca parola ilkesi değişmez.
- `gateway.bind: "tailnet"`, bir Tailnet IPv4 adresi kullanılabilir olduğunda doğrudan Tailnet bağlantısını (HTTPS ve Serve/Funnel olmadan) ve gerekli yerel `127.0.0.1` değerini kullanır; aksi takdirde yalnızca geri döngüye geri döner.
- `gateway.bind: "auto"`, geri döngüyü tercih eder; aynı ana bilgisayardaki geri döngü erişimini korurken ağ erişimini Tailnet ile sınırlamak için `tailnet` kullanın.
- Serve/Funnel yalnızca **Gateway Control UI + WS** öğelerini kullanıma açar. Node'lar aynı Gateway WS uç noktası üzerinden bağlandığından Serve, Node erişimi için de çalışır.

### Tailscale ön koşulları ve sınırları

- Serve, tailnet'iniz için HTTPS'nin etkinleştirilmesini gerektirir; eksikse CLI istemde bulunur.
- Serve, Tailscale kimlik üst bilgilerini ekler; Funnel eklemez.
- Funnel; Tailscale v1.38.3+, MagicDNS, etkinleştirilmiş HTTPS ve bir funnel Node özniteliği gerektirir.
- Funnel, TLS üzerinden yalnızca `443`, `8443` ve `10000` portlarını destekler.
- macOS'te Funnel, Tailscale uygulamasının açık kaynaklı varyantını gerektirir.

## Tarayıcı denetimi (uzak Gateway + yerel tarayıcı)

Gateway'i bir makinede çalıştırıp başka bir makinedeki tarayıcıyı yönetmek için tarayıcı makinesinde bir **Node ana bilgisayarı** çalıştırın ve ikisini de aynı tailnet'te tutun. Gateway, tarayıcı eylemlerini Node'a proxy üzerinden iletir; ayrı bir denetim sunucusu veya Serve URL'si gerekmez.

Tarayıcı denetimi için Funnel'dan kaçının; Node eşleştirmesini operatör erişimi gibi değerlendirin.

## Daha fazla bilgi

- Tailscale Serve genel bakışı: [https://tailscale.com/kb/1312/serve](https://tailscale.com/kb/1312/serve)
- `tailscale serve` komutu: [https://tailscale.com/kb/1242/tailscale-serve](https://tailscale.com/kb/1242/tailscale-serve)
- Tailscale Funnel genel bakışı: [https://tailscale.com/kb/1223/tailscale-funnel](https://tailscale.com/kb/1223/tailscale-funnel)
- `tailscale funnel` komutu: [https://tailscale.com/kb/1311/tailscale-funnel](https://tailscale.com/kb/1311/tailscale-funnel)

## İlgili

- [Uzaktan erişim](/tr/gateway/remote)
- [Keşif](/tr/gateway/discovery)
- [Kimlik doğrulama](/tr/gateway/authentication)
