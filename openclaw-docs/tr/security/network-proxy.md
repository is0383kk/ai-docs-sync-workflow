---
read_when:
    - SSRF ve DNS yeniden bağlama saldırılarına karşı çok katmanlı savunma istiyorsunuz
    - OpenClaw çalışma zamanı trafiği için harici bir iletme proxy'si yapılandırma
summary: OpenClaw çalışma zamanı HTTP ve WebSocket trafiği operatör tarafından yönetilen bir filtreleme proxy'si üzerinden nasıl yönlendirilir?
title: Ağ proxy'si
x-i18n:
    generated_at: "2026-07-26T23:36:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e948189d691e2cfe32e911e24071fd77157397b510d606423ef738c2565071b5
    source_path: security/network-proxy.md
    workflow: 16
---

OpenClaw, çalışma zamanı HTTP ve WebSocket trafiğini operatör tarafından yönetilen bir ileri proxy üzerinden yönlendirebilir. Bu, isteğe bağlı bir derinlemesine savunma katmanıdır: merkezi çıkış kontrolü, daha güçlü SSRF koruması ve ağ sınırında hedef denetlenebilirliği sağlar. Proxy, hedefi DNS çözümlemesinden sonra ve yukarı akış bağlantısını açmadan hemen önce, bağlantı kurulurken değerlendirdiğinden, DNS yeniden bağlama saldırısının daha önceki bir uygulama düzeyi DNS denetimi ile gerçek giden bağlantı arasındaki zaman aralığına dayanma olanağını da daraltır. Tek bir proxy politikası ayrıca operatörlere OpenClaw'ı yeniden derlemeden hedef kurallarını, ağ segmentasyonunu, hız sınırlarını veya giden trafik izin listelerini uygulayabilecekleri tek bir yer sağlar.

OpenClaw bir proxy sunmaz, indirmez, başlatmaz, yapılandırmaz veya onaylamaz. Ortamınıza uygun proxy teknolojisini siz çalıştırırsınız; OpenClaw kendi HTTP ve WebSocket istemcilerini bunun üzerinden yönlendirir.

## Yapılandırma

```yaml
proxy:
  proxyUrl: http://127.0.0.1:3128
```

URL'yi ortam üzerinden de ayarlayabilirsiniz:

```bash
OPENCLAW_PROXY_URL=http://127.0.0.1:3128 openclaw gateway run
```

`proxy.proxyUrl`, `OPENCLAW_PROXY_URL` değerine göre önceliklidir. Yapılandırılmış bir URL, yönetilen proxy yönlendirmesini etkinleştirir; her iki URL'nin de kaldırılması bunu devre dışı bırakır.

| Anahtar                  | Tür                                  | Varsayılan     | Notlar                                                                                                                                 |
| -------------------- | ------------------------------------ | -------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `proxy.proxyUrl`     | string                               | ayarlanmamış   | `http://` veya `https://` ileri proxy URL'si. URL'ye gömülü kimlik bilgileri hassas kabul edilir ve anlık görüntülerden/günlüklerden sansürlenir. |
| `proxy.tls.caFile`   | string                               | ayarlanmamış   | Özel bir CA tarafından imzalanmış `https://` proxy uç noktasını doğrulamak için CA paketi.                                                          |
| `proxy.loopbackMode` | `gateway-only` \| `proxy` \| `block` | `gateway-only` | Geri döngü atlama davranışını denetler; aşağıya bakın.                                                                                         |

Yönetilen Gateway hizmetlerinde, ön plandaki ortam değişkenine güvenmek yerine yeniden kurulum sonrasında korunması için URL'yi yapılandırmada saklayın:

```bash
openclaw config set proxy.proxyUrl http://127.0.0.1:3128
openclaw gateway install --force
openclaw gateway start
```

`OPENCLAW_PROXY_URL` ortam yedeği, ön plandaki çalıştırmalar için en uygunudur. Kurulu bir hizmetle kullanmak için bunu hizmetin kalıcı ortamına (`$OPENCLAW_STATE_DIR/.env`, varsayılan `~/.openclaw/.env`) koyun, ardından launchd/systemd/Scheduled Tasks tarafından alınması için yeniden kurun.

### Özel CA kullanan HTTPS proxy uç noktası

```yaml
proxy:
  proxyUrl: https://proxy.corp.example:8443
  tls:
    caFile: /etc/openclaw/proxy-ca.pem
```

`proxy.tls.caFile`, proxy uç noktasının kendi TLS sertifikasını doğrular. Bu, hedefe yönelik bir MITM güven ayarı, istemci sertifikası veya proxy'nin hedef politikasının yerine geçen bir ayar değildir. Yalnızca tüm Node işleminin başlangıçtan itibaren ek bir CA'ya güvenmesi gerektiğinde (örneğin, her HTTPS hedef sertifikasını yeniden imzalayan kurumsal bir TLS inceleme sistemi) bunun yerine `NODE_EXTRA_CA_CERTS` kullanın — bu değişken işlem genelindedir ve Node başlamadan önce ayarlanmalıdır; dolayısıyla OpenClaw bunu `proxy.tls.caFile` ayarını uyguladığı gibi çalışma sırasında uygulayamaz. HTTPS proxy uç noktası güveni için `proxy.tls.caFile` tercih edin: tüm işlem yerine yönetilen proxy yönlendirmesiyle sınırlıdır.

```bash
openclaw config set proxy.proxyUrl https://proxy.corp.example:8443
openclaw config set proxy.tls.caFile /etc/openclaw/proxy-ca.pem
openclaw gateway run
```

## Yönlendirme nasıl çalışır?

Geçerli bir proxy URL'siyle, korunan çalışma zamanı işlemleri (`openclaw gateway run`, `openclaw node run`, `openclaw agent --local`) normal HTTP ve WebSocket çıkışını proxy üzerinden yönlendirir:

```text
OpenClaw işlemi
  fetch, node:http, node:https, WebSocket istemcileri  -> operatör proxy'si -> hedef
```

OpenClaw, dahili olarak işlem düzeyinde yönlendirme çalışma zamanı olarak [Proxyline](https://github.com/openclaw/proxyline) kurar. `fetch`, undici tabanlı istemciler, `node:http`/`node:https`, yaygın WebSocket istemcileri ve yardımcılar tarafından oluşturulan `CONNECT` tünellerini kapsar ve çağıran tarafından sağlanan Node HTTP aracılarının yerine geçerek açıkça belirtilmiş aracıların (`axios`, `got`, `node-fetch` ve benzer Node aracısı tabanlı istemciler dâhil) proxy'yi sessizce atlayamamasını sağlar.

Proxy URL şeması, OpenClaw'dan proxy'ye olan atlamayı tanımlar; nihai hedefe olanı değil:

- `http://proxy.example:3128` — proxy'ye düz TCP; OpenClaw, HTTPS hedefleri için `CONNECT` dâhil olmak üzere HTTP proxy istekleri gönderir.
- `https://proxy.example:8443` — OpenClaw, proxy'nin kendisine TLS bağlantısı açar (proxy'nin sertifikasını doğrulayarak), ardından bu oturumun içinde HTTP proxy istekleri gönderir.

Hedef TLS, proxy uç noktası TLS'sinden bağımsızdır: Bir HTTPS hedefi için OpenClaw her zaman proxy'den bir `CONNECT` tüneli ister ve hedef TLS'yi bu tünel üzerinden başlatır.

Proxy etkinken OpenClaw, `no_proxy`/`NO_PROXY` değerlerini temizler. Bu atlama listeleri hedef tabanlıdır; `localhost` veya `127.0.0.1` değerlerinin burada bırakılması, SSRF hedeflerinin proxy'yi tamamen atlamasına izin verir. Kapatma sırasında OpenClaw önceki proxy ortamını geri yükler ve önbelleğe alınmış yönlendirme durumunu sıfırlar.

Bazı plugin'ler, işlem düzeyinde yönlendirme etkin olsa bile kendi proxy bağlantı düzenine ihtiyaç duyan özel bir aktarımın sahibidir. Telegram'ın Bot API istemcisi kendi HTTP/1 undici dağıtıcısını kullanır ve işlem proxy ortamının yanı sıra `OPENCLAW_PROXY_URL` yedeğini de ayrıca dikkate alır.

### Gateway geri döngü modu

Yerel Gateway denetim düzlemi istemcileri normalde `ws://127.0.0.1:18789` gibi bir geri döngü WebSocket'ine bağlanır. `proxy.loopbackMode`, bu trafiğin yönetilen proxy'yi atlayıp atlamayacağını denetler:

```yaml
proxy:
  proxyUrl: http://127.0.0.1:3128
  loopbackMode: gateway-only # gateway-only, proxy veya block
```

Yapılandırılmış bir `proxyUrl` veya `OPENCLAW_PROXY_URL`, yönetilen yönlendirmeyi etkinleştirir. URL'yi etkinleştirmeden saklayan gelişmiş bir vazgeçme seçeneği olarak yalnızca
`proxy.enabled: false` ayarını kullanın.

| Mod                      | Davranış                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway-only` (varsayılan) | OpenClaw, etkin Gateway geri döngü yetkilisini doğrudan bağlantı istisnası olarak kaydeder; böylece yerel Gateway WebSocket trafiği proxy olmadan bağlanır. İstisna tam olarak yapılandırılmış ana bilgisayarı/bağlantı noktasını hedeflediğinden özel geri döngü bağlantı noktaları çalışır. Paketle gelen tarayıcı plugin'i, OpenClaw tarafından başlatılan yönetilen tarayıcıların tam yerel CDP hazırlık ve DevTools WebSocket URL'leri için aynı türde bir istisna kaydeder; paketle gelen Ollama bellek gömme sağlayıcısı ise tam olarak yapılandırılmış ana bilgisayar yerelindeki geri döngü gömme kaynağı için daha dar ve korumalı bir doğrudan yola sahiptir. |
| `proxy`                  | Hiçbir geri döngü istisnası kaydedilmez; Gateway ve Ollama geri döngü trafiği proxy üzerinden geçer. Uzak bir proxy, OpenClaw ana bilgisayarının geri döngü hizmetine geri yönlendirme yapabilmelidir (örneğin erişilebilir bir ana bilgisayar adı, IP veya tünel üzerinden) — standart bir uzak proxy, `127.0.0.1`/`localhost` değerlerini OpenClaw ana bilgisayarına göre değil, kendisine göre çözümler.                                                                                                                                                                                                                |
| `block`                  | OpenClaw, bir soket açmadan önce Gateway geri döngü denetim düzlemi bağlantılarını ve korumalı Ollama geri döngü gömme bağlantılarını reddeder.                                                                                                                                                                                                                                                                                                                                                                                                                               |

Gateway denetim düzlemi atlaması, `localhost` ve değişmez geri döngü IP URL'leriyle sınırlıdır — `ws://127.0.0.1:18789`, `ws://[::1]:18789` veya `ws://localhost:18789` kullanın. Diğer ana bilgisayar adları sıradan trafik gibi yönlendirilir.

### Kapsayıcılar

`openclaw --container ...` komutları için OpenClaw, ayarlandığında `OPENCLAW_PROXY_URL` değerini kapsayıcıyı hedefleyen alt CLI'a iletir. URL'ye kapsayıcının içinden erişilebilmelidir — oradaki `127.0.0.1`, ana bilgisayarı değil kapsayıcının kendisini ifade eder. Bu denetimi açıkça geçersiz kılmak için `OPENCLAW_CONTAINER_ALLOW_LOOPBACK_PROXY_URL=1` ayarını yapmadığınız sürece OpenClaw, kapsayıcıyı hedefleyen komutlarda geri döngü proxy URL'lerini reddeder.

## İlgili proxy terimleri

- `proxy.enabled` / `proxy.proxyUrl` — çalışma zamanı çıkışı için giden ileri proxy yönlendirmesi. Bu sayfa.
- `gateway.auth.mode: "trusted-proxy"` — Gateway erişimi için gelen, kimlik duyarlı ters proxy kimlik doğrulaması. [Güvenilen proxy kimlik doğrulaması](/tr/gateway/trusted-proxy-auth) bölümüne bakın.
- `openclaw proxy` — geliştirme ve destek için yerel hata ayıklama proxy'si ve yakalama denetleyicisi. [openclaw proxy](/tr/cli/proxy) bölümüne bakın.
- `tools.web.fetch.useTrustedEnvProxy` — varsayılan olarak katı DNS sabitlemesini ve ana bilgisayar adı politikasını korurken, operatör denetimindeki HTTP(S) ortam proxy'sinin DNS çözümlemesine izin vermek üzere `web_fetch` için isteğe bağlı etkinleştirme. [Web getirme](/tr/tools/web-fetch#trusted-env-proxy) bölümüne bakın.
- Kanala veya sağlayıcıya özgü proxy ayarları — tek bir aktarım için sahibine özgü geçersiz kılmalar. Çalışma zamanının tamamında merkezi çıkış denetimi için yönetilen ağ proxy'sini tercih edin.

## Proxy'yi doğrulama

Proxy'nin hedef politikası gerçek güvenlik sınırıdır; OpenClaw, proxy'nizin doğru hedefleri engellediğini doğrulayamaz. Proxy'yi şunları yapacak şekilde yapılandırın:

- Yalnızca geri döngüye veya güvenilir bir özel arayüze bağlayın; yalnızca OpenClaw işlemi/ana bilgisayarı/kapsayıcısı/hizmet hesabı tarafından erişilebilir olmalıdır.
- Hedefleri kendisi çözümlemeli ve hem düz HTTP hem de HTTPS `CONNECT` tünelleri için DNS çözümlemesinden sonra, bağlantı kurulurken IP'ye göre engellemelidir.
- Geri döngü, özel, bağlantı yerel, meta veri, çok noktaya yayın, ayrılmış ve dokümantasyon aralıklarına yönelik hedef tabanlı atlamaları reddetmelidir.
- DNS çözümleme yoluna tamamen güvenmiyorsanız ana bilgisayar adı izin listelerinden kaçının.
- Hedefi, kararı, durumu ve nedeni günlüğe kaydedin — istek gövdelerini, yetkilendirme üstbilgilerini, çerezleri veya diğer gizli bilgileri asla kaydetmeyin.
- Politikayı sürüm denetimi altında tutun ve değişiklikleri güvenlik açısından hassas olarak inceleyin.

OpenClaw'ı çalıştıran aynı ana bilgisayar/kapsayıcı/hizmet hesabından doğrulayın:

```bash
openclaw proxy validate --proxy-url http://127.0.0.1:3128
```

Özel CA kullanan bir HTTPS proxy uç noktasıyla:

```bash
openclaw proxy validate --proxy-url https://proxy.corp.example:8443 --proxy-ca-file /etc/openclaw/proxy-ca.pem
```

| Bayrak                   | Amaç                                                                 |
| ------------------------ | -------------------------------------------------------------------- |
| `--proxy-url <url>`      | Yapılandırmayı/ortamı çözümlemek yerine bu URL'yi doğrulayın.        |
| `--proxy-ca-file <path>` | Bir HTTPS proxy uç noktası için CA paketi.                            |
| `--allowed-url <url>`    | Başarılı olması beklenen hedef (yinelenebilir).                       |
| `--denied-url <url>`     | Engellenmesi beklenen hedef (yinelenebilir).                          |
| `--apns-reachable`       | Ayrıca proxy'nin doğrudan bir korumalı alan APNs HTTP/2 yoklamasını tünelleyebildiğini doğrulayın. |
| `--apns-authority <url>` | `--apns-reachable` ile yoklanan APNs yetkilisini geçersiz kılın.     |
| `--timeout-ms <ms>`      | İstek başına zaman aşımı.                                            |
| `--json`                 | Makine tarafından okunabilir çıktı.                                  |

Herhangi bir yapılandırma, ortam veya `--proxy-url` değeri yoksa komut bir yapılandırma sorunu bildirir; yapılandırmayı değiştirmeden önce tek seferlik bir ön kontrol için `--proxy-url` iletin.

`--allowed-url`/`--denied-url` belirtilmediğinde varsayılan kontroller şunlardır: `https://example.com/` başarılı olmalı ve proxy'nin erişememesi gereken geçici bir geri döngü kanarya sunucusu engellenmelidir. Geri döngü kontrolü, aktarım hatasında veya kanaryanın çalıştırmaya özgü belirtecini içermeyen 2xx dışı bir yanıtta başarılı olur; belirtecin bulunmadığı bir 2xx yanıtta (kanarya dışındaki bir kaynaktan gelen beklenmedik başarı) ve özellikle eşleşen belirteci taşıyan herhangi bir yanıtta başarısız olur; çünkü bu, proxy'nin reddetmesi gereken bir geri döngü hedefini gerçekten ilettiğini kanıtlar. Özel `--denied-url` hedeflerinde böyle bir kanarya belirteci yoktur, bu nedenle kapalı kalacak şekilde başarısız olurlar: Herhangi bir HTTP yanıtı hedefin erişilebilir olduğu anlamına gelir (başarısızlık) ve OpenClaw, proxy'nizin erişilebilir bir kaynağı reddettiğini başka bir şeyin ters gitmesinden ayırt edemediği için aktarım hatası, engellendiği kanıtlanmış sayılmak yerine sonuçsuz olarak bildirilir. `--apns-reachable` kasıtlı olarak geçersiz bir sağlayıcı belirteci gönderir; bu nedenle `403 InvalidProviderToken` yanıtı, tünelin Apple'a ulaştığının kanıtı sayılır. Komut, herhangi bir doğrulama hatasında `1` koduyla çıkar; proxy URL kimlik bilgileri hem metin hem de JSON çıktısında sansürlenir.

```json
{
  "ok": true,
  "config": {
    "enabled": true,
    "proxyUrl": "http://127.0.0.1:3128/",
    "source": "override",
    "errors": []
  },
  "checks": [
    { "kind": "allowed", "url": "https://example.com/", "ok": true, "status": 200 },
    { "kind": "apns", "url": "https://api.sandbox.push.apple.com", "ok": true, "status": 403 }
  ]
}
```

Elle `curl` kontrolü (genel istek başarılı olmalı; geri döngü ve meta veri istekleri proxy'nin kendisi tarafından engellenmelidir — `curl` tek başına, proxy reddini erişilemeyen bir kaynaktan `openclaw proxy validate`'ün yerleşik kanaryasının ayırt ettiği şekilde ayırt edemez):

```bash
curl -x http://127.0.0.1:3128 https://example.com/
curl -x http://127.0.0.1:3128 http://127.0.0.1/
curl -x http://127.0.0.1:3128 http://169.254.169.254/
```

## Önerilen engellenmiş hedefler

Herhangi bir ileri proxy, güvenlik duvarı veya çıkış ilkesi için başlangıç ret listesi. OpenClaw'ın kendi SSRF sınıflandırıcısı `src/infra/net/ssrf.ts` ve `packages/net-policy/src/ip.ts` içinde bulunur (`BLOCKED_HOSTNAMES`, `BLOCKED_IPV4_SPECIAL_USE_RANGES`, `BLOCKED_IPV6_SPECIAL_USE_RANGES`, RFC 2544 kıyaslama ön eki ve NAT64/6to4/Teredo/ISATAP/IPv4 eşlemeli biçimler için gömülü IPv4 işleme) — bunlar yararlı başvuru kaynaklarıdır ancak OpenClaw, harici proxy'nizde bu kuralları dışa aktarmaz veya uygulamaz.

| Aralık veya ana makine                                                               | Engelleme nedeni                                   |
| ------------------------------------------------------------------------------------ | ------------------------------------------------- |
| `127.0.0.0/8`, `localhost`, `localhost.localdomain`                                  | IPv4 geri döngü                                    |
| `::1/128`                                                                            | IPv6 geri döngü                                    |
| `0.0.0.0/8`, `::/128`                                                                | Belirtilmemiş / bu ağa ait adresler                |
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`                                      | RFC 1918 özel ağları                               |
| `169.254.0.0/16`, `fe80::/10`                                                        | Yaygın bulut meta veri yolları dâhil bağlantı yereli |
| `169.254.169.254`, `metadata.google.internal`                                        | Bulut meta veri hizmetleri                         |
| `100.64.0.0/10`                                                                      | Operatör sınıfı NAT paylaşımlı adres alanı         |
| `198.18.0.0/15`, `2001:2::/48`                                                       | Kıyaslama aralıkları                               |
| `192.0.0.0/24`, `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`, `2001:db8::/32` | Özel kullanım ve belgelendirme aralıkları          |
| `224.0.0.0/4`, `ff00::/8`                                                            | Çok noktaya yayın                                  |
| `240.0.0.0/4`                                                                        | Ayrılmış IPv4                                      |
| `fc00::/7`, `fec0::/10`                                                              | IPv6 yerel/özel aralıkları                         |
| `100::/64`, `2001:20::/28`                                                           | IPv6 atma ve ORCHIDv2 aralıkları                   |
| `64:ff9b::/96`, `64:ff9b:1::/48`                                                     | Gömülü IPv4 içeren NAT64 ön ekleri                 |
| `2002::/16`, `2001::/32`                                                             | Gömülü IPv4 içeren 6to4 ve Teredo                  |
| `::/96`, `::ffff:0:0/96`                                                             | IPv4 uyumlu ve IPv4 eşlemeli IPv6                  |

Bulut sağlayıcınızın veya ağ platformunuzun belgelediği diğer meta veri ana makinelerini ya da ayrılmış aralıkları ekleyin.

## Sınırlar

| Yüzey                                                       | Yönetilen proxy durumu                                                                                                                                   |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fetch`, `node:http`, `node:https`, yaygın WebSocket istemcileri | Yapılandırıldığında yönetilen proxy kancaları üzerinden yönlendirilir.                                                                                   |
| APNs doğrudan HTTP/2                                         | APNs tarafından yönetilen `CONNECT` yardımcısı üzerinden yönlendirilir.                                                                         |
| Gateway denetim düzlemi geri döngüsü                         | Yalnızca tam olarak yapılandırılmış yerel geri döngü Gateway URL'si için doğrudandır.                                                                     |
| Hata ayıklama proxy'sinin yukarı akış iletimi                | Yerel tanılama için açıkça etkinleştirilmediği sürece yönetilen proxy modu etkinken devre dışıdır.                                                        |
| IRC                                                          | Ham TCP/TLS; yönetilen HTTP proxy modu tarafından proxy üzerinden geçirilmez. Dağıtımınız tüm çıkış trafiğinin ileri proxy üzerinden geçmesini gerektiriyorsa `channels.irc.enabled: false` ayarlayın. |
| Diğer ham `net`, `tls` veya `http2` istemci çağrıları | Birleştirilmeden önce ham yuva koruması tarafından sınıflandırılmalıdır.                                                                                  |

- Bu, işletim sistemi düzeyinde bir ağ korumalı alanı değil, JavaScript HTTP/WebSocket istemcileri için işlem düzeyinde kapsama alanıdır.
- Ham `net`, `tls`, `http2` yuvaları, yerel eklentiler ve OpenClaw dışı alt işlemler, proxy ortam değişkenlerini devralıp bunlara uymadıkları sürece Node düzeyindeki yönlendirmeyi atlayabilir. Çatallanmış OpenClaw alt CLI'ları, yönetilen proxy URL'sini ve `proxy.loopbackMode` durumunu devralır.
- Kullanıcıya ait yerel WebUI'ler ve yerel model sunucuları genel bir yerel ağ atlaması kapsamında değildir — gerekirse bunları operatör proxy ilkesindeki izin listesine ekleyin. Bunun istisnası, paketle gelen Ollama bellek gömme sağlayıcısının korumalı doğrudan yoludur; bu yol, yapılandırılmış `baseUrl` değerindeki tam ana makine yerel geri döngü kaynağıyla sınırlıdır; LAN, tailnet, özel ağ ve genel Ollama ana makineleri yönetilen proxy'yi kullanmaya devam eder.
- Yerel hata ayıklama proxy'sinin doğrudan yukarı akış iletimi (proxy istekleri ve `CONNECT` tünelleri için), yönetilen proxy modu etkinken varsayılan olarak devre dışıdır; yalnızca onaylanmış yerel tanılama için etkinleştirin.
- OpenClaw, proxy ilkenizi incelemez, test etmez veya onaylamaz. Proxy ilkesi değişikliklerini güvenlik açısından hassas operasyonel değişiklikler olarak değerlendirin.
