---
read_when:
    - Kontrol paneli kimlik doğrulamasını veya erişime açma modlarını değiştirme
summary: Gateway panosuna (Kontrol Arayüzü) erişim ve kimlik doğrulama
title: Kontrol Paneli
x-i18n:
    generated_at: "2026-07-27T00:21:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ca531ad2943dfdee1cd90a4efdc1fb69c4517780e2be52237fd558b8638e7cd0
    source_path: web/dashboard.md
    workflow: 16
---

Gateway panosu, varsayılan olarak `/` adresinde sunulan tarayıcı Control UI'sidir (`gateway.controlUi.basePath` ile geçersiz kılınabilir).

Hızlı açma (yerel Gateway):

- [http://127.0.0.1:18789/](http://127.0.0.1:18789/) (veya [http://localhost:18789/](http://localhost:18789/))
- `gateway.tls.enabled: true` ile WebSocket uç noktası için `https://127.0.0.1:18789/` ve `wss://127.0.0.1:18789` kullanın.

Temel başvurular:

- Kullanım ve UI özellikleri için [Control UI](/tr/web/control-ui).
- Serve/Funnel otomasyonu için [Tailscale](/tr/gateway/tailscale).
- Bağlama modları ve güvenlik notları için [Web yüzeyleri](/tr/web).

Kimlik doğrulama, yapılandırılmış gateway kimlik doğrulama yolu üzerinden WebSocket el sıkışması sırasında zorunlu tutulur:

- `connect.params.auth.token`
- `connect.params.auth.password`
- `gateway.auth.allowTailscale: true` olduğunda Tailscale Serve kimlik üst bilgileri
- `gateway.auth.mode: "trusted-proxy"` olduğunda güvenilir proxy kimlik üst bilgileri

[Gateway yapılandırması](/tr/gateway/configuration) içindeki `gateway.auth` bölümüne bakın.

<Warning>
Control UI bir **yönetici yüzeyidir** (sohbet, yapılandırma, çalıştırma onayları). Herkese açık hâle getirmeyin. UI, pano URL belirteçlerini geçerli tarayıcı sekmesi ve seçili gateway URL'si için sessionStorage'da tutar ve yüklemeden sonra bunları URL'den kaldırır. Localhost, Tailscale Serve veya SSH tünelini tercih edin.
</Warning>

## Hızlı yol (önerilen)

- İlk kurulumdan sonra CLI, panoyu otomatik olarak açar ve temiz (belirteç içermeyen) bir bağlantı yazdırır.
- İstediğiniz zaman yeniden açın: `openclaw dashboard` (bağlantıyı kopyalar, mümkünse tarayıcı açar, grafik arabirim yoksa SSH ipucu yazdırır).
- Hem panoya kopyalama hem de tarayıcıya iletme başarısız olursa `openclaw dashboard`, temiz URL'yi yine de yazdırır ve belirtecinizi (`OPENCLAW_GATEWAY_TOKEN` veya `gateway.auth.token` kaynağından) `token` URL parçası anahtarı olarak eklemenizi söyler; belirteç değerini günlüklere asla yazdırmaz.
- UI, paylaşılan gizli anahtarla kimlik doğrulama isterse yapılandırılmış belirteci veya parolayı Control UI ayarlarına yapıştırın.

## Kimlik doğrulama temelleri (yerel ve uzak)

- **Localhost**: `http://127.0.0.1:18789/` adresini açın.
- **Gateway TLS**: `gateway.tls.enabled: true` olduğunda pano/durum bağlantıları `https://`, Control UI WebSocket bağlantıları ise `wss://` kullanır.
- **Paylaşılan gizli anahtar belirtecinin kaynağı**: `gateway.auth.token` (veya `OPENCLAW_GATEWAY_TOKEN`). `openclaw dashboard`, tek seferlik önyükleme için bunu URL parçası üzerinden iletebilir; Control UI bunu localStorage'da değil, geçerli sekme ve seçili gateway URL'si için sessionStorage'da tutar.
- **Yapılandırma eksikken çalışma zamanı belirteci**: başlangıç iletisi bir çalışma zamanı belirteci oluşturulduğunu söylüyorsa bu belirteç geçicidir ve `openclaw config get gateway.auth.token` üzerinden kullanılamaz. Geri döngü bağlantısı da kimlik doğrulaması gerektirir. `openclaw doctor --generate-gateway-token` komutunu çalıştırın, Gateway'i yeniden başlatın ve ardından yapılandırılmış belirteci Control UI ayarlarına yapıştırın.
- `gateway.auth.token`, SecretRef tarafından yönetiliyorsa `openclaw dashboard`, haricî olarak yönetilen belirteçlerin kabuk günlüklerinde, pano geçmişinde veya tarayıcı başlatma bağımsız değişkenlerinde açığa çıkmasını önlemek amacıyla tasarım gereği belirteç içermeyen bir URL yazdırır/kopyalar/açar. Başvuru geçerli kabuğunuzda çözümlenemiyorsa yine de belirteç içermeyen URL'yi ve uygulanabilir kimlik doğrulama kurulumu yönergelerini yazdırır.
- **Paylaşılan gizli anahtar parolası**: yapılandırılmış `gateway.auth.password` (veya `OPENCLAW_GATEWAY_PASSWORD`) değerini kullanın. Pano, parolaları yeniden yüklemeler arasında saklamaz.
- **Kimlik taşıyan modlar**: `gateway.auth.allowTailscale: true` olduğunda Tailscale Serve, kimlik üst bilgileri aracılığıyla Control UI/WebSocket kimlik doğrulamasını karşılar; geri döngü dışı, kimlikten haberdar bir ters proxy ise `gateway.auth.mode: "trusted-proxy"` koşulunu karşılar. WebSocket için ikisi de paylaşılan gizli anahtar yapıştırılmasını gerektirmez.
- **Localhost değilse**: Tailscale Serve, paylaşılan gizli anahtarla geri döngü dışı bir bağlama, `gateway.auth.mode: "trusted-proxy"` ile kimlikten haberdar geri döngü dışı bir ters proxy veya SSH tüneli kullanın. Özel giriş `gateway.auth.mode: "none"` ya da güvenilir proxy HTTP kimlik doğrulamasını bilinçli olarak çalıştırmadığınız sürece HTTP API'leri paylaşılan gizli anahtarla kimlik doğrulama kullanmaya devam eder. [Web yüzeyleri](/tr/web) bölümüne bakın.

## Telegram'da açma

Telegram botları, `/dashboard` ile panoyu Telegram Mini App olarak açabilir.

Gereksinimler:

- Telegram'ın bir HTTPS Mini App URL'si alabilmesi için `gateway.tailscale.mode: "serve"` veya `"funnel"`.
- Telegram göndericisi botun sahibi olmalıdır: `commands.ownerAllowFrom` içinde sayısal bir Telegram kullanıcı kimliği veya seçili hesabın etkin `channels.telegram.allowFrom` değeri.
- Botla yapılan bir DM içinde `/dashboard` komutunu çalıştırın. Grup çağrıları yalnızca komutu DM içinde açmanızı söyler ve düğme içermez.
- Docker kurulumları: Serve/Funnel modları, gateway'in `tailscaled` yanında geri döngüye bağlanmasını gerektirir; yayımlanmış bağlantı noktaları kullanan köprü ağı bunu karşılayamaz. Gateway kapsayıcısını `network_mode: host` ile çalıştırın ve ana makinenin `tailscaled` yuvasını (`/var/run/tailscale`) ve `tailscale` CLI'sini kapsayıcıya bağlayın.

Mini App, tek seferlik sahip aktarımı gerçekleştirir ve kısa ömürlü bir önyükleme belirteciyle Control UI'ye yönlendirir. URL'de paylaşılan bir gateway belirtecini açığa çıkarmaz.

v1 kapsamı dışında olanlar:

- Telegram Web iframe desteklenmez.
- Tailscale Serve/Funnel, desteklenen tek yayımlanmış URL yoludur.

<a id="if-you-see-unauthorized-1008"></a>

## "unauthorized" / 1008 görürseniz

- Gateway'e erişilebildiğini doğrulayın: yerelde `openclaw status`; uzakta `ssh -N -L 18789:127.0.0.1:18789 user@gateway-host` SSH tünelini açın, ardından `http://127.0.0.1:18789/` adresini açın.
- `AUTH_TOKEN_MISMATCH` için gateway yeniden deneme ipuçları döndürdüğünde istemciler, önbelleğe alınmış bir cihaz belirteciyle tek bir güvenilir yeniden deneme yapabilir; bu yeniden deneme, belirtecin önbelleğe alınmış onaylı kapsamlarını yeniden kullanır (açık `deviceToken`/`scopes` çağırıcıları istedikleri kapsam kümesini korur). Bu yeniden denemeden sonra kimlik doğrulama hâlâ başarısız olursa belirteç sapmasını manuel olarak giderin.
- `AUTH_SCOPE_MISMATCH` için cihaz belirteci tanındı ancak istenen kapsamları taşımıyor; paylaşılan gateway belirtecini döndürmek yerine yeniden eşleştirin veya yeni kapsam kümesini onaylayın.
- Bu yeniden deneme yolunun dışında bağlantı kimlik doğrulaması önceliği şöyledir: açıkça belirtilen paylaşılan belirteç/parola, ardından açıkça belirtilen `deviceToken`, ardından saklanan cihaz belirteci, ardından önyükleme belirteci.
- Zaman uyumsuz Tailscale Serve yolunda aynı `{scope, ip}` için başarısız denemeler, başarısız kimlik doğrulama sınırlayıcısı bunları kaydetmeden önce sıralı hâle getirilir; bu nedenle ikinci bir eşzamanlı hatalı yeniden deneme zaten `retry later` gösterebilir.
- Belirteç sapmasını giderme adımları için [Belirteç sapması kurtarma denetim listesi](/tr/cli/devices#token-drift-recovery-checklist) bölümüne bakın.
- Paylaşılan gizli anahtarı gateway ana makinesinden alın veya sağlayın:
  - Belirteç: `openclaw config get gateway.auth.token`
  - Parola: yapılandırılmış `gateway.auth.password` veya `OPENCLAW_GATEWAY_PASSWORD` değerini çözümleyin
  - SecretRef tarafından yönetilen belirteç: haricî gizli anahtar sağlayıcısını çözümleyin veya bu kabukta `OPENCLAW_GATEWAY_TOKEN` değerini dışa aktarıp `openclaw dashboard` komutunu yeniden çalıştırın
  - Paylaşılan gizli anahtar yapılandırılmadığı için çalışma zamanı belirteci oluşturulduysa: `openclaw doctor --generate-gateway-token` komutunu çalıştırın, Gateway'i yeniden başlatın ve ardından yapılandırılmış belirteci kullanın
- Pano ayarlarında belirteci veya parolayı kimlik doğrulama alanına yapıştırın, ardından bağlanın.
- UI dil seçici Appearance altında değil, **Settings -> General -> Language** konumundadır.

## İlgili

- [Control UI](/tr/web/control-ui)
- [WebChat](/tr/web/webchat)
