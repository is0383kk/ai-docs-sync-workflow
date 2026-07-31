---
read_when:
    - Gateway'e Tailscale üzerinden erişmek istiyorsunuz
    - Tarayıcıdaki Kontrol Arayüzünü ve yapılandırma düzenlemeyi istiyorsunuz
summary: 'Gateway web yüzeyleri: Kontrol kullanıcı arayüzü, bağlama modları ve güvenlik'
title: Web
x-i18n:
    generated_at: "2026-07-26T23:09:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 413fb029d95241f5c6043b28825727cdee52b2fa8cbe998fbbd6e3ff7b81467b
    source_path: web/index.md
    workflow: 16
---

Gateway, Gateway WebSocket ile aynı porttan küçük bir **tarayıcı Control UI** (Vite + Lit) sunar:

- varsayılan: `http://<host>:18789/`
- `gateway.tls.enabled: true` ile: `https://<host>:18789/`
- isteğe bağlı ön ek: `gateway.controlUi.basePath` değerini ayarlayın (ör. `/openclaw`)

Özellikler [Control UI](/tr/web/control-ui) bölümünde açıklanır. Bu sayfa bağlama modlarını, güvenliği ve web'e yönelik diğer yüzeyleri kapsar.

## Yapılandırma (varsayılan olarak açık)

Varlıklar mevcut olduğunda (`dist/control-ui`) Control UI **varsayılan olarak etkindir**:

```json5
{
  gateway: {
    controlUi: { enabled: true, basePath: "/openclaw" }, // basePath isteğe bağlıdır
  },
}
```

## Webhook'lar

`hooks.enabled=true` olduğunda Gateway, aynı HTTP sunucusunda bir webhook uç noktası da sunar. Kimlik doğrulama ve yükler için [Gateway yapılandırma referansındaki](/tr/gateway/configuration-reference#hooks) `hooks` bölümüne bakın.

## Yönetici HTTP RPC

`POST /api/v1/admin/rpc`, seçili Gateway kontrol düzlemi yöntemlerini HTTP üzerinden sunar. Varsayılan olarak kapalıdır; yalnızca `admin-http-rpc` plugin'i etkinleştirildiğinde kaydedilir. Kimlik doğrulama modeli, izin verilen yöntemler ve WebSocket API ile karşılaştırma için [Yönetici HTTP RPC](/tr/plugins/admin-http-rpc) bölümüne bakın.

## Tailscale erişimi

<Tabs>
  <Tab title="Tümleşik Serve (önerilir)">
    Gateway'i geri döngüde tutun ve Tailscale Serve'ün onu proxy'lemesini sağlayın:

    ```json5
    {
      gateway: {
        bind: "loopback",
        tailscale: { mode: "serve" },
      },
    }
    ```

    Gateway'i başlatın:

    ```bash
    openclaw gateway
    ```

    `https://<magicdns>/` adresini (veya yapılandırdığınız `gateway.controlUi.basePath` adresini) açın.

  </Tab>
  <Tab title="Tailnet bağlaması + belirteç">
    ```json5
    {
      gateway: {
        bind: "tailnet",
        controlUi: { enabled: true },
        auth: { mode: "token", token: "your-token" },
      },
    }
    ```

    Gateway'i başlatın (bu geri döngü dışı örnek, paylaşılan gizli belirteç kimlik doğrulaması kullanır):

    ```bash
    openclaw gateway
    ```

    `http://<tailscale-ip>:18789/` adresini (veya yapılandırdığınız `gateway.controlUi.basePath` adresini) açın.

  </Tab>
  <Tab title="Genel internet (Funnel)">
    ```json5
    {
      gateway: {
        bind: "loopback",
        tailscale: { mode: "funnel" },
        auth: { mode: "password" }, // veya OPENCLAW_GATEWAY_PASSWORD
      },
    }
    ```

    `tailscale.mode: "funnel"`, `gateway.auth.mode: "password"` gerektirir; hem Serve hem de Funnel, `gateway.bind: "loopback"` gerektirir.

  </Tab>
</Tabs>

## Güvenlik notları

- Gateway kimlik doğrulaması varsayılan olarak gereklidir: etkinleştirildiğinde belirteç, parola, güvenilir proxy veya Tailscale Serve kimlik üstbilgileri.
- Geri döngü dışı bağlamalar da gateway kimlik doğrulamasını **gerektirir**: belirteç/parola kimlik doğrulaması veya `gateway.auth.mode: "trusted-proxy"` kullanan kimlik farkındalıklı bir ters proxy.
- İlk katılım sihirbazı varsayılan olarak paylaşılan gizli kimlik doğrulaması oluşturur ve geri döngüde bile genellikle bir gateway belirteci üretir.
- Paylaşılan gizli modunda UI, WebSocket el sıkışması sırasında `connect.params.auth.token` veya `connect.params.auth.password` gönderir.
- `gateway.tls.enabled: true` ile yerel pano/durum yardımcıları `https://` URL'lerini ve `wss://` WebSocket URL'lerini oluşturur.
- Kimlik taşıyan modlarda (Tailscale Serve, `trusted-proxy`) WebSocket kimlik doğrulama denetimi, paylaşılan bir gizli yerine istek üstbilgileriyle karşılanır.
- Genel, geri döngü dışı Control UI dağıtımlarında `gateway.controlUi.allowedOrigins` değerini açıkça ayarlayın (tam kaynaklar). Geri döngü, RFC1918/bağlantı-yerel, `.local`, `.ts.net` ve Tailscale CGNAT ana makinelerinde özel aynı kaynak yüklemeleri bu değer olmadan kabul edilir.
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback: true`, Host üstbilgisi kaynak geri dönüşünü etkinleştirir; bu, güvenliği tehlikeli biçimde düşürür.
- Serve ile `gateway.auth.allowTailscale: true` olduğunda Tailscale kimlik üstbilgileri Control UI/WebSocket kimlik doğrulamasını karşılar (belirteç/parola gerekmez). HTTP API uç noktaları Tailscale kimlik üstbilgilerini kullanmaz; her zaman gateway'in normal HTTP kimlik doğrulama modunu izler. Serve üzerinden bile açık kimlik bilgileri gerektirmek için `gateway.auth.allowTailscale: false` değerini ayarlayın. Bu belirteçsiz akış, gateway ana makinesinin kendisinin güvenilir olduğunu varsayar. [Tailscale](/tr/gateway/tailscale) ve [Güvenlik](/tr/gateway/security) bölümlerine bakın.

## UI'yi derleme

Gateway, statik dosyaları `dist/control-ui` konumundan sunar:

```bash
pnpm ui:build
```
