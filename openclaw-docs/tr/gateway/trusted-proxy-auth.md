---
read_when:
    - OpenClaw'u kimlik duyarlı bir proxy'nin arkasında çalıştırma
    - OpenClaw'ın önünde OAuth ile Pomerium, Caddy veya nginx kurulumu
    - Ters proxy kurulumlarında WebSocket 1008 yetkisiz hatalarını düzeltme
    - HSTS ve diğer HTTP sağlamlaştırma başlıklarının nerede ayarlanacağına karar verme
sidebarTitle: Trusted proxy auth
summary: Gateway kimlik doğrulamasını güvenilir bir ters proxy'ye devredin (Pomerium, Caddy, nginx + OAuth)
title: Güvenilir proxy kimlik doğrulaması
x-i18n:
    generated_at: "2026-07-26T22:47:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39bf8f12b3ae95f53b21bfed12deb1c8ed8f767711955bbee52c74538052a89f
    source_path: gateway/trusted-proxy-auth.md
    workflow: 16
---

<Warning>
**Güvenlik açısından hassas özellik.** Bu mod, kimlik doğrulamayı tamamen ters proxy'nize devreder. Yanlış yapılandırma, Gateway'inizi yetkisiz erişime açabilir. Etkinleştirmeden önce bu sayfayı dikkatle okuyun.
</Warning>

## Ne zaman kullanılmalı

- OpenClaw'ı **kimlik bilgisine duyarlı bir proxy** (Pomerium, Caddy + OAuth, nginx + oauth2-proxy, Traefik + iletilen kimlik doğrulama) arkasında çalıştırıyorsanız.
- Proxy'niz tüm kimlik doğrulama işlemlerini gerçekleştiriyor ve kullanıcı kimliğini üstbilgiler aracılığıyla iletiyorsa.
- Proxy'nin Gateway'e giden tek yol olduğu bir Kubernetes veya konteyner ortamındaysanız.
- Tarayıcılar WS yüklerinde belirteç iletemediği için WebSocket `1008 unauthorized` hataları alıyorsanız.

## Ne zaman KULLANILMAMALI

- Proxy'niz kullanıcıların kimliğini doğrulamıyorsa (yalnızca bir TLS sonlandırıcı veya yük dengeleyiciyse).
- Gateway'e proxy'yi atlayan herhangi bir yol varsa (güvenlik duvarı açıkları, dahili ağ erişimi).
- Proxy'nizin iletilen üstbilgileri doğru biçimde kaldırdığından/üzerine yazdığından emin değilseniz.
- Yalnızca kişisel, tek kullanıcılı erişime ihtiyacınız varsa (bunun yerine Tailscale Serve + geri döngüyü değerlendirin).

## Nasıl çalışır

<Steps>
  <Step title="Proxy kullanıcının kimliğini doğrular">
    Ters proxy'niz kullanıcıların kimliğini doğrular (OAuth, OIDC, SAML vb.).
  </Step>
  <Step title="Proxy bir kimlik üstbilgisi ekler">
    Proxy, kimliği doğrulanmış kullanıcı kimliğini içeren bir üstbilgi ekler (ör. `x-forwarded-user: nick@example.com`).
  </Step>
  <Step title="Gateway güvenilir kaynağı doğrular">
    OpenClaw, isteğin **güvenilir bir proxy IP'sinden** (`gateway.trustedProxies`) geldiğini ve Gateway'in kendi geri döngü veya yerel arayüz adresinden gelmediğini denetler.
  </Step>
  <Step title="Gateway kimliği çıkarır">
    OpenClaw gerekli üstbilgileri, ardından yapılandırılmış üstbilgiden kullanıcı kimliğini okur.
  </Step>
  <Step title="Yetkilendir">
    Tüm denetimler başarılı olursa ve kullanıcı `allowUsers` denetimini (ayarlandıysa) geçerse istek yetkilendirilir.
  </Step>
</Steps>

## Yapılandırma

```json5
{
  gateway: {
    // Güvenilir proxy kimlik doğrulaması, proxy'nin kaynak IP'sinin varsayılan olarak geri döngü olmamasını bekler
    bind: "lan",

    // KRİTİK: Buraya yalnızca proxy'nizin IP adreslerini ekleyin
    trustedProxies: ["10.0.0.1", "172.17.0.1"],

    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        // Kimliği doğrulanmış kullanıcı kimliğini içeren üstbilgi (gerekli)
        userHeader: "x-forwarded-user",

        // İsteğe bağlı: BULUNMASI ZORUNLU üstbilgiler (proxy doğrulaması)
        requiredHeaders: ["x-forwarded-proto", "x-forwarded-host"],

        // İsteğe bağlı: belirli kullanıcılarla sınırla (boş = tümüne izin ver)
        allowUsers: ["nick@example.com", "admin@company.org"],

        // İsteğe bağlı: açıkça etkinleştirildikten sonra aynı ana makinedeki geri döngü proxy'sine izin ver
        allowLoopback: false,

        // İsteğe bağlı: kimliği doğrulanmış proxy kullanıcılarının yeni tarayıcı cihazlarını kaydetmesine izin ver
        deviceAutoApprove: {
          enabled: false,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

<Warning>
**Değerlendirme sırasına göre çalışma zamanı kuralları**

1. İsteğin kaynak IP'si `gateway.trustedProxies` ile eşleşmelidir (CIDR desteklidir); aksi takdirde istek reddedilir (`trusted_proxy_untrusted_source`).
2. Geri döngü kaynaklı istekler (`127.0.0.1`, `::1`), `gateway.auth.trustedProxy.allowLoopback = true` etkin olmadığı ve geri döngü adresi ayrıca `trustedProxies` içinde bulunmadığı sürece reddedilir (`trusted_proxy_loopback_source`). Bu denetim üstbilgi denetimlerinden önce çalıştığından, gerekli üstbilgiler de eksik olsa bile geri döngü kaynağı bu nedenle başarısız olur.
3. Gateway ana makinesinin kendi yerel ağ arayüzü adreslerinden biriyle eşleşen geri döngü dışı kaynaklar, sahteciliğe karşı koruma amacıyla reddedilir (`trusted_proxy_local_interface_source`). Arayüz keşfinin kendisi başarısız olursa istek yine reddedilir (`trusted_proxy_local_interface_check_failed`).
4. `requiredHeaders` ve `userHeader` mevcut olmalı ve boş olmamalıdır.
5. `allowUsers` boş değilse çıkarılan kullanıcıyı içermelidir.

**İletilen üstbilgi kanıtı, yerel doğrudan geri dönüş için geri döngü yerelliğini geçersiz kılar.** Bir istek geri döngü üzerinden gelir ancak bir `Forwarded`, herhangi bir `X-Forwarded-*` veya `X-Real-IP` üstbilgisi taşırsa bu kanıt, istek geri döngü olması nedeniyle güvenilir proxy kimlik doğrulamasında yine başarısız olsa bile onu yerel doğrudan parola geri dönüşünden ve cihaz kimliği denetiminden diskalifiye eder.

`allowLoopback`, Gateway ana makinesindeki yerel işlemlere ters proxy ile aynı ölçüde güvenir. Bunu yalnızca Gateway doğrudan uzak erişime karşı güvenlik duvarıyla korunmaya devam ediyorsa ve yerel proxy istemci tarafından sağlanan kimlik üstbilgilerini kaldırıyor veya üzerlerine yazıyorsa etkinleştirin.

Ters proxy üzerinden geçmeyen dahili Gateway istemcileri, güvenilir proxy kimlik üstbilgileri yerine `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` kullanmalıdır. Geri döngü dışındaki Control UI dağıtımları için yine de açıkça `gateway.controlUi.allowedOrigins` gerekir.
</Warning>

### Yapılandırma başvurusu

<ParamField path="gateway.trustedProxies" type="string[]" required>
  Güvenilecek proxy IP adresleri (veya CIDR'ler) dizisi. Diğer IP'lerden gelen istekler reddedilir.
</ParamField>
<ParamField path="gateway.auth.mode" type="string" required>
  `"trusted-proxy"` olmalıdır.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.userHeader" type="string" required>
  Kimliği doğrulanmış kullanıcı kimliğini içeren üstbilgi adı.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.requiredHeaders" type="string[]">
  İsteğin güvenilir sayılması için mevcut olması gereken ek üstbilgiler.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowUsers" type="string[]">
  Kullanıcı kimliklerinin izin verilenler listesi. Boş olması, kimliği doğrulanmış tüm kullanıcılara izin verilmesi anlamına gelir.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowLoopback" type="boolean" default="false">
  Aynı ana makinedeki geri döngü ters proxy'leri için isteğe bağlı destek.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.enabled" type="boolean" default="false">
  Güvenilir proxy kimlik doğrulamasından sonra yeni Control UI ve WebChat cihaz kimliklerini otomatik olarak onaylar.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.scopes" type="string[]" default='["operator.read", "operator.write", "operator.approvals"]'>
  Otomatik olarak onaylanan bir tarayıcı cihazına verilen azami kapsamlar. `operator.admin` değerinin açıkça listelenmesi, proxy üzerinden kimliği doğrulanmış her kullanıcının otomatik tam yönetici cihaz yetkisi istemesine olanak tanır, kapsamsız isteklerin otomatik olarak tam yönetici yetkisi almasına neden olur ve KRİTİK `gateway.trusted_proxy_device_auto_approve_admin` güvenlik denetimi bulgusunu ve bir Gateway başlatma uyarısını tetikler.
</ParamField>

<Warning>
`allowLoopback` seçeneğini yalnızca yerel ters proxy amaçlanan güven sınırı olduğunda etkinleştirin. Gateway'e bağlanabilen herhangi bir yerel işlem proxy kimlik üstbilgileri göndermeyi deneyebilir; bu nedenle doğrudan Gateway erişimini ana makineye özel tutun ve `x-forwarded-proto` gibi proxy'nin sahip olduğu üstbilgileri ya da proxy'niz destekliyorsa imzalı bir beyan üstbilgisini zorunlu kılın.
</Warning>

## Otomatik cihaz onayı

Güvenilir proxy kimlik doğrulaması, isteğe bağlı olarak proxy kimliğini yeni tarayıcı cihazlarının onay sınırı olarak kullanabilir:

```json5
{
  gateway: {
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-forwarded-user",
        allowUsers: ["operator@example.com"],
        deviceAutoApprove: {
          enabled: true,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

Varsayılan değer `enabled: false`. Etkinleştirildiğinde şu kuralların tümü geçerlidir:

1. WebSocket, `trusted-proxy` yöntemi üzerinden, boş olmayan ve bir izin verilenler listesi yapılandırılmışsa `allowUsers` denetimini geçen bir kullanıcı kimliğiyle doğrulanmış olmalıdır. Belirteç, parola, Tailscale ve kimliği doğrulanmamış bağlantılar bu politikayı hiçbir zaman kullanmaz.
2. Yalnızca yeni bir Control UI veya WebChat tarayıcı cihazı otomatik olarak onaylanabilir. Kapsam yükseltme dahil olmak üzere mevcut bir cihaza yönelik tüm istekler, `openclaw devices approve <requestId>` ile elle onaylanmak üzere beklemede kalır.
3. Cihaz, `operator` rolüyle onaylanır. Bağlantı isteği kapsamlar içeriyorsa verilen yetki, istenen kapsamlarla `deviceAutoApprove.scopes` değerinin tam kesişimidir. İstekte kapsamlar belirtilmezse yapılandırılmış liste verilir; bu liste belirtilmediğinde varsayılan olarak `operator.read`, `operator.write` ve `operator.approvals` kullanılır. Ardından ortaya çıkan yetki, mevcut olduğunda bağlantının [`x-openclaw-scopes`](#control-ui-pairing-behavior) proxy üstbilgisiyle ayrıca sınırlandırılır; böylece bir kullanıcının kapsamlarını daraltan proxy yalnızca oturumu değil, **kalıcı** cihaz yetkisini de sınırlar; mevcut ancak boş bir üstbilgi hiçbir kapsam sağlamaz. Bu sınır, istemci kendi kapsam listesini belirtmediğinde da uygulanır.
4. `operator.admin` yalnızca `deviceAutoApprove.scopes` içinde açıkça listelenmesi yoluyla kullanılabilir. Listelendiğinde, proxy üzerinden kimliği doğrulanmış her kullanıcı yeni bir tarayıcı cihazında tam yönetici yetkisi isteyebilir ve bunu otomatik olarak alabilir; kapsamsız istekler otomatik olarak tam yönetici yetkisi alır. `openclaw security audit`, KRİTİK `gateway.trusted_proxy_device_auto_approve_admin` bulgusunu bildirir ve Gateway başlatma sırasında bir kez uyarı kaydeder. Kimlik başına roller kullanıma sunulana kadar `openclaw devices approve` veya `openclaw devices rotate` ile elle yönetici onayını tercih edin.

<Warning>
Bu seçeneğin etkinleştirilmesi, yeni tarayıcı cihazı kaydını tamamen ters proxy kimliğine devreder. Güvenliği ihlal edilmiş bir proxy hesabı, yapılandırılmış tüm kapsamları içeren kalıcı bir cihaz kaydedebilir. `operator.admin` değerinin listelenmesi, bu cihazı elle onay olmadan tam yönetici yapar. Gateway'e yalnızca proxy üzerinden erişilebilmesini sağlayın, güçlü proxy kimlik doğrulaması zorunlu kılın, kimlik üstbilgilerinin üzerine yazın ve dar bir `allowUsers` listesi kullanın.
</Warning>

## Control UI eşleştirme davranışı

`gateway.auth.mode = "trusted-proxy"` etkin olduğunda ve istek güvenilir proxy denetimlerini geçtiğinde, Control UI WebSocket oturumları cihaz eşleştirme kimliği olmadan bağlanabilir.

Kapsam sonuçları:

- Cihazsız Control UI WebSocket oturumları bağlanır ancak varsayılan olarak hiçbir operatör kapsamı almaz. OpenClaw, istenen kapsam listesini `[]` olarak temizler; böylece onaylanmış eşleştirilmiş bir cihaza/belirtece bağlı olmayan bir oturum izinlerini kendisi beyan edemez.
- Başarılı bir WebSocket bağlantısından sonra yöntemler `missing scope` hatasıyla başarısız olursa tarayıcının cihaz kimliği oluşturabilmesi ve eşleştirmeyi tamamlayabilmesi için HTTPS kullanın. Bkz. [Control UI güvenli olmayan HTTP](/tr/web/control-ui#insecure-http).
- Kullanımdan kaldırılmış
  `gateway.controlUi.dangerouslyDisableDeviceAuth=true` anahtarını hâlâ içeren eski yapılandırmalar, sınırlandırılmış
  [Control UI yükseltme geçişini](/tr/web/control-ui#device-pairing-first-connection) kullanır.

Ters proxy kapsam sınırlaması: Proxy'niz Control UI WebSocket yükseltme isteğinde `x-openclaw-scopes` gönderirse OpenClaw, oturum kapsamlarını istenen kapsamlarla beyan edilen kapsamların kesişimiyle sınırlar. Bu üstbilgi kapsam vermez; yalnızca oturumun sahip olabileceği kapsamları daraltır. `deviceAutoApprove.enabled` doğru olduğunda aynı sınır, [otomatik cihaz onayı](#automatic-device-approval) tarafından yazılan kalıcı cihaz yetkisine de uygulanır; böylece otomatik olarak onaylanan bir cihaz hiçbir zaman proxy'nin beyan ettiğinden daha fazla kapsama sahip olmaz.

Sonuçlar:

- Eşleştirme artık cihazsız Control UI erişimi için birincil geçit değildir. `deviceAutoApprove.enabled` doğru olduğunda proxy kimliği, yeni tarayıcı cihazı kaydı için de onay geçidi hâline gelir.
- Ters proxy kimlik doğrulama politikanız ve `allowUsers` etkin erişim denetimi hâline gelir.
- Gateway girişini yalnızca güvenilir proxy IP'leriyle sınırlı tutun (`gateway.trustedProxies` + güvenlik duvarı).

Özel WebSocket istemcileri Control UI oturumları değildir. Kullanımdan kaldırılmış Control UI
yükseltme girdisi, isteğe bağlı
`client.mode: "backend"` veya CLI biçimli istemcilere geçici erişim sağlamaz. Özel otomasyon;
cihaz kimliğini/eşleştirmeyi, ayrılmış doğrudan yerel `client.id: "gateway-client"`
arka uç yardımcı yolunu veya bir HTTP istek/yanıt yüzeyi daha uygunsa [yönetici HTTP RPC Plugin'ini](/tr/plugins/admin-http-rpc)
kullanmalıdır.

## Operatör kapsamları üstbilgisi

Güvenilir proxy kimlik doğrulaması, **kimlik taşıyan** bir HTTP modudur; bu nedenle çağıranlar, HTTP API isteklerinde `x-openclaw-scopes` ile isteğe bağlı olarak operatör kapsamları bildirebilir.

Not: WebSocket kapsamları, Gateway protokolü el sıkışması ve cihaz kimliği bağlaması tarafından belirlenir. Control UI WebSocket yükseltme isteklerinde `x-openclaw-scopes`, bir yetki verme değil, yalnızca üzerinde uzlaşılan oturum kapsamlarına uygulanan bir üst sınırdır. Bkz. [Control UI eşleştirme davranışı](#control-ui-pairing-behavior).

Örnekler:

- `x-openclaw-scopes: operator.read`
- `x-openclaw-scopes: operator.read,operator.write`
- `x-openclaw-scopes: operator.admin,operator.write`

Davranış:

- Üst bilgi mevcut olduğunda OpenClaw, bildirilen kapsam kümesini dikkate alır.
- Üst bilgi mevcut ancak boş olduğunda istek, **hiçbir** operatör kapsamı bildirmez.
- Üst bilgi olmadığında, normal kimlik taşıyan HTTP API'leri standart varsayılan operatör kapsamı kümesine geri döner (`operator.admin`, `operator.read`, `operator.write`, `operator.approvals`, `operator.pairing`, `operator.talk.secrets`).
- Gateway kimlik doğrulamalı **plugin HTTP rotaları** varsayılan olarak daha dardır: `x-openclaw-scopes` olmadığında çalışma zamanı kapsamları yalnızca `operator.write` kapsamına geri döner.
- Tarayıcı kaynaklı HTTP istekleri, güvenilir proxy kimlik doğrulaması başarılı olduktan sonra bile `gateway.controlUi.allowedOrigins` denetiminden (veya bilinçli Host üst bilgisi geri dönüş modundan) geçmelidir.

Pratik kural: Bir güvenilir proxy isteğinin varsayılanlardan daha dar olmasını istediğinizde veya Gateway kimlik doğrulamalı bir plugin rotası yazma kapsamından daha güçlü bir kapsam gerektirdiğinde `x-openclaw-scopes` değerini açıkça gönderin.

## TLS sonlandırma ve HSTS

Tek bir TLS sonlandırma noktası kullanın ve HSTS'yi orada uygulayın.

<Tabs>
  <Tab title="Proxy TLS sonlandırması (önerilen)">
    Ters proxy'niz `https://control.example.com` için HTTPS'yi işlediğinde, söz konusu alan adı için proxy'de `Strict-Transport-Security` ayarını yapın.

    - İnternete açık dağıtımlar için uygundur.
    - Sertifika ve HTTP sağlamlaştırma politikasını tek bir yerde tutar.
    - OpenClaw, proxy'nin arkasında geri döngü HTTP'sinde kalabilir.

    Örnek üst bilgi değeri:

    ```text
    Strict-Transport-Security: max-age=31536000; includeSubDomains
    ```

  </Tab>
  <Tab title="Gateway TLS sonlandırması">
    OpenClaw HTTPS'yi doğrudan kendisi sunuyorsa (TLS sonlandıran bir proxy yoksa) şunu ayarlayın:

    ```json5
    {
      gateway: {
        tls: { enabled: true },
        http: {
          securityHeaders: {
            strictTransportSecurity: "max-age=31536000; includeSubDomains",
          },
        },
      },
    }
    ```

    `strictTransportSecurity`, bir dize üst bilgi değerini veya açıkça devre dışı bırakmak için `false` değerini kabul eder.

  </Tab>
</Tabs>

### Kullanıma alma rehberi

- Trafiği doğrularken önce kısa bir maksimum süreyle (örneğin `max-age=300`) başlayın.
- Uzun süreli değerlere (örneğin `max-age=31536000`) yalnızca yeterince emin olduktan sonra geçin.
- Yalnızca tüm alt alan adları HTTPS'ye hazırsa `includeSubDomains` ekleyin.
- Ön yüklemeyi yalnızca tüm alan adı kümeniz için ön yükleme gereksinimlerini bilinçli olarak karşılıyorsanız kullanın.
- Yalnızca geri döngüye bağlı yerel geliştirme HSTS'den fayda sağlamaz.

## Proxy kurulum örnekleri

<AccordionGroup>
  <Accordion title="Pomerium">
    Pomerium, kimliği `x-pomerium-claim-email` (veya diğer talep üst bilgileri) içinde ve bir JWT'yi `x-pomerium-jwt-assertion` içinde iletir.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // Pomerium'un IP'si
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-pomerium-claim-email",
            requiredHeaders: ["x-pomerium-jwt-assertion"],
          },
        },
      },
    }
    ```

    Pomerium yapılandırma parçacığı:

    ```yaml
    routes:
      - from: https://openclaw.example.com
        to: http://openclaw-gateway:18789
        policy:
          - allow:
              or:
                - email:
                    is: nick@example.com
        pass_identity_headers: true
    ```

  </Accordion>
  <Accordion title="OAuth ile Caddy">
    `caddy-security` pluginiyle Caddy, kullanıcıların kimliğini doğrulayabilir ve kimlik üst bilgilerini iletebilir.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // Caddy/yardımcı proxy IP'si
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```

    Caddyfile parçacığı:

    ```caddy
    openclaw.example.com {
        authenticate with oauth2_provider
        authorize with policy1

        reverse_proxy openclaw:18789 {
            header_up X-Forwarded-User {http.auth.user.email}
        }
    }
    ```

  </Accordion>
  <Accordion title="nginx + oauth2-proxy">
    oauth2-proxy, kullanıcıların kimliğini doğrular ve kimliği `x-auth-request-email` içinde iletir.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // nginx/oauth2-proxy IP'si
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-auth-request-email",
          },
        },
      },
    }
    ```

    nginx yapılandırma parçacığı:

    ```nginx
    location / {
        auth_request /oauth2/auth;
        auth_request_set $user $upstream_http_x_auth_request_email;

        proxy_pass http://openclaw:18789;
        proxy_set_header X-Auth-Request-Email $user;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    ```

  </Accordion>
  <Accordion title="İleri kimlik doğrulamalı Traefik">
    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["172.17.0.1"], // Traefik kapsayıcı IP'si
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## Karma token yapılandırması

Paylaşılan bir token da yapılandırılmışsa (`gateway.auth.token` veya `OPENCLAW_GATEWAY_TOKEN`), Gateway başlangıcı güvenilir proxy kimlik doğrulamasını reddeder. Bu ikisi birbirini dışlar; çünkü paylaşılan bir token, aynı ana makinedeki çağıranların bu modun zorunlu kılmayı amaçladığı proxy tarafından doğrulanmış kimlikten tamamen farklı bir yolla kimlik doğrulamasına olanak tanır.

Başlangıç `gateway auth mode is trusted-proxy, but a shared token is also configured` benzeri bir hatayla başarısız olursa:

- Güvenilir proxy modunu kullanırken paylaşılan tokenı kaldırın veya
- Token tabanlı kimlik doğrulamayı amaçlıyorsanız `gateway.auth.mode` değerini `"token"` olarak değiştirin.

Geri döngü güvenilir proxy kimlik üst bilgileri yine güvenli biçimde başarısız olur: aynı ana makinedeki çağıranların kimliği sessizce proxy kullanıcıları olarak doğrulanmaz. Proxy'yi atlayan dahili OpenClaw çağıranları bunun yerine `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` ile kimlik doğrulayabilir. Token geri dönüşü, güvenilir proxy modunda bilinçli olarak desteklenmez.

## Güvenlik denetim listesi

Güvenilir proxy kimlik doğrulamasını etkinleştirmeden önce şunları doğrulayın:

- [ ] **Proxy tek yoldur**: Gateway portu, proxy'niz dışındaki her şeye karşı güvenlik duvarıyla korunur.
- [ ] **trustedProxies asgaridir**: Tüm alt ağlar değil, yalnızca gerçek proxy IP'leriniz.
- [ ] **Geri döngü proxy kaynağı bilinçli seçilmiştir**: Aynı ana makinedeki bir proxy için `gateway.auth.trustedProxy.allowLoopback` açıkça etkinleştirilmediği sürece, güvenilir proxy kimlik doğrulaması geri döngü kaynaklı isteklerde güvenli biçimde başarısız olur.
- [ ] **Proxy üst bilgileri temizler**: Proxy'niz, istemcilerden gelen `x-forwarded-*` üst bilgilerini sona eklemek yerine üzerine yazar.
- [ ] **TLS sonlandırma**: Proxy'niz TLS'yi işler; kullanıcılar HTTPS üzerinden bağlanır.
- [ ] **allowedOrigins açıkça belirtilmiştir**: Geri döngü dışındaki Control UI, açıkça belirtilmiş `gateway.controlUi.allowedOrigins` kullanır.
- [ ] **allowUsers ayarlanmıştır** (önerilen): Kimliği doğrulanmış herkese izin vermek yerine erişimi bilinen kullanıcılarla sınırlayın.
- [ ] **Karma token yapılandırması yoktur**: Hem `gateway.auth.token` hem de `gateway.auth.mode: "trusted-proxy"` ayarlamayın.
- [ ] **Yerel parola geri dönüşü özeldir**: Dahili doğrudan çağıranlar için `gateway.auth.password` yapılandırırsanız, proxy kullanmayan uzak istemcilerin doğrudan erişememesi için Gateway portunu güvenlik duvarıyla koruyun.
- [ ] **Cihazın otomatik onaylanması bilinçli bir tercihtir**: `deviceAutoApprove.enabled` true ise ters proxy hesap güvenliğini cihaz kaydı sınırı olarak değerlendirin ve verilen kapsam listesini yönetici olmayan kapsamlarla ve asgari düzeyde tutun.

## Güvenlik denetimi

`openclaw security audit`, güvenilir proxy kimlik doğrulamasını **kritik** önem derecesine sahip bir bulgu olarak işaretler. Bu bilinçli bir tercihtir; güvenliği proxy kurulumunuza devrettiğinizi hatırlatır.

Denetim şunları kontrol eder:

- Temel `gateway.trusted_proxy_auth` uyarısı/kritik hatırlatması.
- Eksik `trustedProxies` yapılandırması.
- Eksik `userHeader` yapılandırması.
- Boş `allowUsers` (kimliği doğrulanmış tüm kullanıcılara izin verir).
- Aynı ana makinedeki proxy kaynakları için etkinleştirilmiş `allowLoopback`.
- Etkinleştirilmiş tarayıcı cihazı otomatik onayı (yeni cihaz eşleştirmesini proxy kimliğine devreder).

Control UI kullanıma açıldığında, güvenilir proxy'ye özgü olmayan ayrı bulgular da geçerlidir: joker karakterli veya eksik `gateway.controlUi.allowedOrigins` ve Host üst bilgisi kaynak geri dönüşü.

## Sorun giderme

<AccordionGroup>
  <Accordion title="trusted_proxy_untrusted_source">
    İstek, `gateway.trustedProxies` içindeki bir IP'den gelmedi. Şunları kontrol edin:

    - Proxy IP'si doğru mu? (Docker kapsayıcı IP'leri değişebilir.)
    - Proxy'nizin önünde bir yük dengeleyici var mı?
    - Gerçek IP'leri bulmak için `docker inspect` veya `kubectl get pods -o wide` kullanın.

  </Accordion>
  <Accordion title="trusted_proxy_loopback_source">
    OpenClaw, geri döngü kaynaklı bir güvenilir proxy isteğini reddetti.

    Şunları kontrol edin:

    - Proxy `127.0.0.1` / `::1` adresinden mi bağlanıyor?
    - Güvenilir proxy kimlik doğrulamasını aynı ana makinedeki bir geri döngü ters proxy'siyle mi kullanmaya çalışıyorsunuz?

    Çözüm:

    - Proxy üzerinden geçmeyen aynı ana makinedeki dahili istemciler için token/parola kimlik doğrulamasını tercih edin veya
    - Trafiği geri döngü olmayan güvenilir bir proxy adresi üzerinden yönlendirin ve bu IP'yi `gateway.trustedProxies` içinde tutun veya
    - Bilinçli olarak kullanılan aynı ana makinedeki bir ters proxy için `gateway.auth.trustedProxy.allowLoopback = true` ayarını yapın, geri döngü adresini `gateway.trustedProxies` içinde tutun ve proxy'nin kimlik üst bilgilerini temizlediğinden veya üzerlerine yazdığından emin olun.

  </Accordion>
  <Accordion title="trusted_proxy_local_interface_source / trusted_proxy_local_interface_check_failed">
    İsteğin kaynak IP'si, Gateway ana makinesinin geri döngü olmayan kendi ağ arabirimi adreslerinden biriyle (proxy ile değil) eşleşti; bu, tailnet'lerde veya Docker köprü ağlarında aynı ana makineden gelen sahte trafiğe karşı bir korumadır. `..._check_failed`, arabirim keşfinin kendisinin hata verdiği anlamına gelir; bu nedenle OpenClaw güvenli biçimde başarısız olur.

    Şunları kontrol edin:

    - Gateway ana makinesinin kendisindeki bir işlem, proxy'yi atlayarak kimlik üst bilgilerini doğrudan mı gönderiyor?
    - Proxy, Gateway ile aynı ağ ad alanında ve yerel bir arabirim olarak da görünen bir IP ile mi çalışıyor?

    Çözüm: Proxy trafiğini Gateway ana makinesinde yerel olarak bağlı olmayan bir adres üzerinden yönlendirin veya `allowLoopback` seçeneğini yalnızca gerçek bir aynı ana makine proxy kurulumu için kullanın.

  </Accordion>
  <Accordion title="trusted_proxy_user_missing">
    Kullanıcı üst bilgisi boştu veya yoktu. Şunları kontrol edin:

    - Proxy'niz kimlik üst bilgilerini iletecek şekilde yapılandırılmış mı?
    - Üst bilgi adı doğru mu? (büyük/küçük harfe duyarsızdır, ancak yazım önemlidir)
    - Kullanıcının kimliği proxy'de gerçekten doğrulanmış mı?

  </Accordion>
  <Accordion title="trusted_proxy_missing_header_*">
    Gerekli bir üst bilgi yoktu. Şunları kontrol edin:

    - Söz konusu belirli üst bilgiler için proxy yapılandırmanızı.
    - Üst bilgilerin zincirin herhangi bir yerinde kaldırılıp kaldırılmadığını.

  </Accordion>
  <Accordion title="trusted_proxy_user_not_allowed">
    Kullanıcının kimliği doğrulandı ancak kullanıcı `allowUsers` içinde değil. Kullanıcıyı ekleyin veya izin verilenler listesini kaldırın.
  </Accordion>
  <Accordion title="trusted_proxy_no_proxies_configured / trusted_proxy_config_missing">
    `gateway.auth.mode`, `"trusted-proxy"` olarak ayarlanmış ancak `gateway.trustedProxies` boş veya `gateway.auth.trustedProxy` yapılandırmasının kendisi eksik. Her ikisi de ayarlanana kadar tüm istekler reddedilir.
  </Accordion>
  <Accordion title="trusted_proxy_origin_not_allowed">
    Güvenilir proxy kimlik doğrulaması başarılı oldu ancak tarayıcının `Origin` üstbilgisi, Control UI kaynak denetimlerinden geçemedi.

    Şunları denetleyin:

    - `gateway.controlUi.allowedOrigins`, tam tarayıcı kaynağını içeriyor.
    - Tümüne izin verme davranışını kasıtlı olarak istemediğiniz sürece joker karakterli kaynaklara güvenmiyorsunuz.
    - Host üstbilgisi geri dönüş modunu kasıtlı olarak kullanıyorsanız `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` bilinçli biçimde ayarlanmıştır.

  </Accordion>
  <Accordion title="Bağlantı başarılı ancak yöntemler eksik kapsam bildiriyor">
    WebSocket bağlanıyor ancak `chat.history`, `sessions.list` veya
    `models.list`, `missing scope: operator.read` hatasıyla başarısız oluyor.

    Yaygın nedenler:

    - Cihazsız Control UI oturumu: Güvenilir proxy kimlik doğrulaması, cihaz kimliği olmadan WebSocket bağlantısını kabul edebilir ancak OpenClaw tasarım gereği cihazsız oturumların kapsamlarını temizler.
    - Özel arka uç istemcisi: Kullanımdan kaldırılmış Control UI yükseltme girdisi, rastgele arka uç veya CLI biçimli WebSocket istemcilerine hiçbir zaman erişim vermez.
    - Aşırı dar `x-openclaw-scopes`: Proxy'niz bu üstbilgiyi Control UI WebSocket yükseltme isteğine ekliyorsa oturum kapsamları bu kümeyle sınırlandırılır. Boş bir üstbilgi değeri hiçbir kapsam sağlamaz.

    Düzeltme:

    - Control UI için tarayıcının cihaz kimliği oluşturabilmesi ve eşleştirmeyi tamamlayabilmesi amacıyla HTTPS kullanın.
    - Özel otomasyon için cihaz kimliği/eşleştirme, ayrılmış doğrudan yerel `gateway-client` arka uç yardımcı yolunu veya [yönetici HTTP RPC'sini](/tr/plugins/admin-http-rpc) kullanın.
    - Kullanımdan kaldırılmış `gateway.controlUi.dangerouslyDisableDeviceAuth` anahtarını mevcut yapılandırmaya eklemeyin. Eski kurulumlar tek seferlik kendiliğinden eşleştirme geçişini otomatik olarak kullanır.

  </Accordion>
  <Accordion title="WebSocket hâlâ başarısız oluyor">
    Proxy'nizin şunları yaptığından emin olun:

    - WebSocket yükseltmelerini destekler (`Upgrade: websocket`, `Connection: upgrade`).
    - Kimlik üstbilgilerini yalnızca HTTP'de değil, WebSocket yükseltme isteklerinde de iletir.
    - WebSocket bağlantıları için ayrı bir kimlik doğrulama yolu kullanmaz.

  </Accordion>
</AccordionGroup>

## Belirteç kimlik doğrulamasından geçiş

<Steps>
  <Step title="Proxy'yi yapılandırın">
    Proxy'nizi kullanıcıların kimliğini doğrulayacak ve üstbilgileri iletecek şekilde yapılandırın.
  </Step>
  <Step title="Proxy'yi bağımsız olarak test edin">
    Proxy kurulumunu bağımsız olarak test edin (üstbilgilerle curl).
  </Step>
  <Step title="OpenClaw yapılandırmasını güncelleyin">
    OpenClaw yapılandırmasını güvenilir proxy kimlik doğrulamasıyla güncelleyin.
  </Step>
  <Step title="Gateway'i yeniden başlatın">
    Gateway'i yeniden başlatın.
  </Step>
  <Step title="WebSocket'i test edin">
    Control UI üzerinden WebSocket bağlantılarını test edin.
  </Step>
  <Step title="Denetleyin">
    `openclaw security audit` komutunu çalıştırın ve bulguları inceleyin.
  </Step>
</Steps>

## İlgili

- [Yapılandırma](/tr/gateway/configuration) — yapılandırma başvurusu
- [Operatör kapsamları](/tr/gateway/operator-scopes) — roller, kapsamlar ve onay denetimleri
- [Uzaktan erişim](/tr/gateway/remote) — diğer uzaktan erişim kalıpları
- [Güvenlik](/tr/gateway/security) — kapsamlı güvenlik kılavuzu
- [Tailscale](/tr/gateway/tailscale) — yalnızca tailnet erişimi için daha basit alternatif
