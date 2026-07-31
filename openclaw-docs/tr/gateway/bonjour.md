---
read_when:
    - macOS/iOS'te Bonjour keşif sorunlarını ayıklama
    - mDNS hizmet türlerini, TXT kayıtlarını veya keşif kullanıcı deneyimini değiştirme
summary: Bonjour/mDNS keşfi + hata ayıklama (Gateway işaretleri, istemciler ve yaygın hata modları)
title: Bonjour keşfi
x-i18n:
    generated_at: "2026-07-26T22:44:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f43ef71b323b59362655c390a4df621c2571abbe3b2c1cd2728918c6f76d6f99
    source_path: gateway/bonjour.md
    workflow: 16
---

OpenClaw, etkin bir Gateway'i (WebSocket uç noktası) keşfetmek için Bonjour'u (mDNS/DNS-SD) kullanabilir. Çok noktaya yayın `local.` taraması **yalnızca LAN'a yönelik bir kolaylıktır**: paketle gelen `bonjour` Plugin'i LAN duyurularını yönetir; macOS ana makinelerinde otomatik olarak başlar, Linux, Windows ve konteynerleştirilmiş Gateway dağıtımlarında ise isteğe bağlıdır. Aynı işaret, ağlar arası keşif için yapılandırılmış bir geniş alan DNS-SD etki alanı üzerinden de yayın yapabilir. Keşif, mümkün olan en iyi şekilde çalışır ve SSH ya da Tailnet tabanlı bağlantının yerini **almaz**.

## Tailscale üzerinden geniş alan Bonjour (Tek Noktaya Yayın DNS-SD)

Node ile Gateway farklı ağlardaysa çok noktaya yayın mDNS bu sınırı aşamaz. Tailscale üzerinden **tek noktaya yayın DNS-SD**'ye ("Geniş Alan Bonjour") geçerek aynı keşif kullanıcı deneyimini koruyun:

1. Gateway ana makinesinde Tailnet üzerinden erişilebilen bir DNS sunucusu çalıştırın.
2. Özel bir bölge altında (örnek: `openclaw.internal.`) `_openclaw-gw._tcp` için DNS-SD kayıtları yayınlayın.
3. iOS dahil istemcilerde seçtiğiniz etki alanının bu DNS sunucusu üzerinden çözümlenmesi için Tailscale **bölünmüş DNS**'ini yapılandırın.

Yukarıdaki `openclaw.internal.` yalnızca bir örnektir — OpenClaw tüm keşif etki alanlarını destekler. iOS/Android Node'ları hem `local.` hem de yapılandırdığınız geniş alan etki alanını tarar.

### Gateway yapılandırması

```json5
{
  gateway: { bind: "tailnet" }, // yalnızca tailnet (önerilir)
  discovery: { wideArea: { enabled: true, domain: "openclaw.internal" } },
}
```

`discovery.wideArea.domain`, ayarlanmamışsa yedek olarak `OPENCLAW_WIDE_AREA_DOMAIN` ortam değişkenini de kabul eder.

### Tek seferlik DNS sunucusu kurulumu (Gateway ana makinesi, yalnızca macOS)

```bash
openclaw dns setup --apply
```

Bu komut yalnızca macOS'ta çalışır ve Homebrew ile etkin bir Tailscale bağlantısı gerektirir. CoreDNS'i (`brew install coredns`) kurar ve şu şekilde yapılandırır:

- yalnızca Gateway'in Tailscale arayüzlerinde 53 numaralı bağlantı noktasını dinler
- seçtiğiniz etki alanını (örnek: `openclaw.internal.`) `~/.openclaw/dns/<domain>.db` üzerinden sunar

Hiçbir şey kurmadan planı (etki alanı, bölge dosyası yolu, algılanan Tailnet IP'si, önerilen yapılandırma) önizlemek için önce `--apply` olmadan çalıştırın.

Tailnet'e bağlı bir makineden doğrulayın:

```bash
dns-sd -B _openclaw-gw._tcp openclaw.internal.
dig @<TAILNET_IPV4> -p 53 _openclaw-gw._tcp.openclaw.internal PTR +short
```

### Tailscale DNS ayarları

Tailscale yönetici konsolunda:

- Gateway'in Tailnet IP'sini gösteren bir ad sunucusu ekleyin (UDP/TCP 53).
- Keşif etki alanınızın bu ad sunucusunu kullanması için bölünmüş DNS ekleyin.

İstemciler Tailnet DNS'ini kabul ettikten sonra iOS Node'ları ve CLI keşfi, keşif etki alanınızdaki `_openclaw-gw._tcp` öğesini çok noktaya yayın olmadan tarayabilir.

### Gateway dinleyicisi güvenliği

Gateway WS bağlantı noktası (varsayılan `18789`) varsayılan olarak geri döngü arayüzüne bağlanır. LAN/Tailnet erişimi için bağlantıyı açıkça yapılandırın ve kimlik doğrulamayı etkin tutun. Yalnızca Tailnet kurulumları için `~/.openclaw/openclaw.json` içinde `gateway.bind: "tailnet"` ayarını yapın ve Gateway'i (veya macOS menü çubuğu uygulamasını) yeniden başlatın.

## Neler duyurulur?

Yalnızca Gateway, `_openclaw-gw._tcp` öğesini duyurur. LAN çok noktaya yayın duyuruları etkinleştirildiğinde paketle gelen `bonjour` Plugin'inden gelir; geniş alan DNS-SD yayınlama işlemi Gateway'in sorumluluğunda kalır.

## Hizmet türleri

- `_openclaw-gw._tcp` - macOS/iOS/Android Node'ları tarafından kullanılan Gateway aktarım işareti.

## TXT anahtarları (gizli olmayan ipuçları)

| Anahtar                       | Bulunduğu durum                                                                |
| ----------------------------- | ------------------------------------------------------------------------------ |
| `role=gateway`                | Her zaman.                                                                     |
| `displayName=<friendly name>` | Her zaman.                                                                     |
| `lanHost=<hostname>.local`    | Her zaman.                                                                     |
| `gatewayPort=<port>`          | Her zaman (Gateway WS + HTTP).                                                 |
| `transport=gateway`           | Her zaman.                                                                     |
| `gatewayTls=1`                | Yalnızca TLS etkinleştirildiğinde.                                             |
| `gatewayTlsSha256=<sha256>`   | Yalnızca TLS etkinleştirildiğinde ve bir parmak izi mevcut olduğunda.          |
| `gatewayDirectReachable=1`    | Yalnızca Gateway'e doğrudan erişilebildiğinde (yalnızca aktarıcı/proxy yolu üzerinden değil). |
| `canvasPort=<port>`           | Yalnızca tuval ana makinesi etkinleştirildiğinde; şu anda `gatewayPort` ile aynıdır. |
| `tailnetDns=<magicdns>`       | Yalnızca mDNS tam modu; Tailnet kullanılabilir olduğunda isteğe bağlı ipucu.   |
| `sshPort=<port>`              | Yalnızca tam mod; minimal ve kapalı modlarda dahil edilmez.                    |
| `cliPath=<path>`              | Yalnızca tam mod; minimal ve kapalı modlarda dahil edilmez.                    |

Güvenlik notları:

- Bonjour/mDNS TXT kayıtlarının **kimliği doğrulanmaz**. İstemciler TXT'yi yetkili yönlendirme kaynağı olarak kabul etmemelidir.
- İstemciler çözümlenen hizmet uç noktasını (SRV + A/AAAA) kullanarak yönlendirme yapmalıdır. `lanHost`, `tailnetDns`, `gatewayPort` ve `gatewayTlsSha256` değerlerini yalnızca ipucu olarak değerlendirin.
- SSH otomatik hedefleme de yalnızca TXT ipuçlarını değil, çözümlenen hizmet ana makinesini kullanmalıdır.
- TLS sabitlemesi, duyurulan bir `gatewayTlsSha256` değerinin önceden saklanmış bir sabitlemeyi geçersiz kılmasına asla izin vermemelidir.
- iOS/Android Node'ları, keşif tabanlı doğrudan bağlantıları **yalnızca TLS** olarak değerlendirmeli ve ilk kez karşılaşılan bir parmak izine güvenmeden önce açık kullanıcı onayı istemelidir.

## macOS'ta hata ayıklama

Yerleşik araçlar:

```bash
# Örnekleri tara
dns-sd -B _openclaw-gw._tcp local.

# Bir örneği çözümle (<instance> öğesini değiştirin)
dns-sd -L "<instance>" _openclaw-gw._tcp local.
```

Tarama çalışıyor ancak çözümleme başarısız oluyorsa genellikle bir LAN politikası veya mDNS çözümleyicisi sorunuyla karşılaşıyorsunuzdur.

## Gateway günlüklerinde hata ayıklama

Gateway, dönüşümlü bir günlük dosyası yazar (başlangıçta `gateway log file: ...` olarak yazdırılır). Özellikle aşağıdakiler olmak üzere `bonjour:` satırlarını arayın:

- `bonjour: advertise failed ...`
- `bonjour: suppressing ciao netmask assertion ...`
- `bonjour: ... name conflict resolved` / `hostname conflict resolved`

OpenClaw, her Bonjour hizmetini bir kez başlatır ve yoklama, yeniden deneme, ad çakışması çözümleme ve arayüz değişikliğinde yeniden yayınlama işlemlerini mDNS yanıtlayıcısına bırakır. Bu, normal ağ değişimleri sırasında çakışan yayın girişimlerini önler. Tekrarlanan dahili öz yoklama iletileri, Gateway günlüğünü doldurmamaları için bastırılır.

Aynı ana makineden birden fazla OpenClaw Gateway'i duyuru yaptığında Bonjour, hizmet örneği adlarını benzersiz tutmak için `(2)` veya `(3)` gibi son ekler ekleyebilir. Bu son ekler normal çakışma çözümlemesidir ve yinelenen OCM denetimini göstermez.

Bonjour, geçerli bir DNS etiketi olduğunda duyurulan `.local` ana makinesi için sistem ana makine adını kullanır. Sistem ana makine adı boşluk, alt çizgi veya DNS etiketinde geçersiz başka bir karakter içeriyorsa OpenClaw, `openclaw.local` değerine geri döner. Açık bir ana makine etiketi gerektiğinde Gateway'i başlatmadan önce `OPENCLAW_MDNS_HOSTNAME=<name>` ayarını yapın.

## iOS Node'unda hata ayıklama

iOS Node'u, `_openclaw-gw._tcp` öğesini keşfetmek için `NWBrowser` kullanır.

Günlükleri yakalamak için: Settings -> Gateway -> Advanced -> **Discovery Debug Logs**, ardından Settings -> Gateway -> Advanced -> **Discovery Logs** -> yeniden oluşturun -> **Copy**. Günlük, tarayıcı durum geçişlerini ve sonuç kümesi değişikliklerini içerir.

## Bonjour ne zaman etkinleştirilmeli?

Yerel uygulama ve yakındaki iOS/Android Node'ları çoğunlukla aynı LAN'da keşfe dayandığından, macOS ana makinelerinde boş yapılandırmayla Gateway başlatıldığında Bonjour otomatik olarak başlar.

Aynı LAN'da otomatik keşif Linux, Windows veya macOS dışındaki başka bir ana makinede yararlıysa Bonjour'u açıkça etkinleştirin:

```bash
openclaw plugins enable bonjour
```

Bonjour etkinleştirildiğinde ne kadar TXT meta verisi yayınlanacağını belirlemek için `discovery.mdns.mode` kullanır; aynı mod, geniş alan DNS-SD kayıtlarındaki isteğe bağlı TXT ipuçlarını da denetler. Modlar:

| Mod                 | Davranış                                                                                                                                 |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `minimal` (varsayılan) | Yalnızca temel TXT anahtarları; `sshPort`, `cliPath`, `tailnetDns` dahil edilmez.                                                        |
| `full`              | `sshPort`, `cliPath`, `tailnetDns` ekler — istemciler bu ipuçlarına ihtiyaç duyduğunda kullanın.                                       |
| `off`               | Plugin'in etkinliğini değiştirmeden LAN çok noktaya yayınını bastırır; `discovery.wideArea.domain` ayarlandığında geniş alan DNS-SD yine yayın yapabilir. |

## Bonjour ne zaman devre dışı bırakılmalı?

LAN çok noktaya yayın duyurusu gereksiz, kullanılamaz veya zararlı olduğunda Bonjour'u devre dışı bırakın — macOS dışı sunucular, Docker köprü ağı, WSL veya mDNS çok noktaya yayınını engelleyen bir ağ politikası yaygın örneklerdir. Gateway, yayınlanan URL'si, SSH, Tailnet veya geniş alan DNS-SD üzerinden erişilebilir kalır; yalnızca LAN otomatik keşfi güvenilmez olur.

Dağıtım kapsamındaki sorunlar için ortam geçersiz kılmasını kullanın (Docker görüntüleri, hizmet dosyaları, başlatma betikleri ve tek seferlik hata ayıklama için güvenlidir — ortam ortadan kalktığında bu ayar da kaybolur):

```bash
OPENCLAW_DISABLE_BONJOUR=1
```

Bu OpenClaw yapılandırması için paketle gelen LAN keşif Plugin'ini kasıtlı olarak kapatmak istediğinizde Plugin yapılandırmasını kullanın:

```bash
openclaw plugins disable bonjour
```

## Docker'da dikkat edilmesi gerekenler

Paketle gelen Bonjour Plugin'i, `OPENCLAW_DISABLE_BONJOUR` ayarlanmamışsa algılanan konteynerlerde LAN çok noktaya yayın duyurusunu otomatik olarak devre dışı bırakır. Docker köprü ağları genellikle mDNS çok noktaya yayınını (`224.0.0.251:5353`) konteyner ile LAN arasında iletmez; bu nedenle konteynerden duyuru yapmak keşfin çalışmasını nadiren sağlar.

Dikkat edilmesi gerekenler:

- Bonjour, macOS ana makinelerinde otomatik olarak başlar ve diğer ortamlarda isteğe bağlıdır. Devre dışı bırakılması Gateway'i durdurmaz; yalnızca LAN çok noktaya yayın duyurusunu atlar.
- Bonjour'u devre dışı bırakmak `gateway.bind` değerini değiştirmez; yayınlanan ana makine bağlantı noktasının çalışması için Docker varsayılan olarak `OPENCLAW_GATEWAY_BIND=lan` kullanmaya devam eder.
- Bonjour'u devre dışı bırakmak geniş alan DNS-SD'yi devre dışı bırakmaz. Gateway ile Node aynı LAN'da değilse geniş alan keşfini veya Tailnet'i kullanın.
- Aynı `OPENCLAW_CONFIG_DIR` değerini Docker dışında yeniden kullanmak, konteynerin otomatik devre dışı bırakma politikasını kalıcı hâle getirmez.
- `OPENCLAW_DISABLE_BONJOUR=0` ayarını yalnızca ana makine ağı, macvlan veya mDNS çok noktaya yayınının geçtiği bilinen başka bir ağ için yapın; zorla devre dışı bırakmak için `1` değerine ayarlayın.

## Devre dışı Bonjour sorunlarını giderme

Docker kurulumundan sonra bir Node artık Gateway'i otomatik olarak keşfetmiyorsa:

1. Gateway'in otomatik, zorla açık veya zorla kapalı modda çalışıp çalışmadığını doğrulayın:

   ```bash
   docker compose config | grep OPENCLAW_DISABLE_BONJOUR
   ```

2. Gateway'in kendisine yayınlanan bağlantı noktası üzerinden erişilebildiğini doğrulayın:

   ```bash
   curl -fsS http://127.0.0.1:18789/healthz
   ```

3. Bonjour devre dışıyken doğrudan bir hedef kullanın:
   - Denetim kullanıcı arayüzü veya yerel araçlar: `http://127.0.0.1:18789`
   - LAN istemcileri: `http://<gateway-host>:18789`
   - Ağlar arası istemciler: Tailnet MagicDNS, Tailnet IP'si, SSH tüneli veya geniş alan DNS-SD

4. Docker'da Bonjour Plugin'ini bilerek etkinleştirdiyseniz ve `OPENCLAW_DISABLE_BONJOUR=0` ile duyuruyu zorladıysanız çok noktaya yayını ana makineden test edin:

   ```bash
   dns-sd -B _openclaw-gw._tcp local.
   ```

   Tarama boşsa veya Gateway günlükleri yinelenen ciao yoklama hataları gösteriyorsa `OPENCLAW_DISABLE_BONJOUR=1` ayarını geri yükleyin ve doğrudan bir rota ya da Tailnet rotası kullanın.

## Yaygın hata modları

- **Bonjour ağlar arasında çalışmaz**: Tailnet veya SSH kullanın.
- **Çok noktaya yayın engellenmiş**: bazı Wi-Fi ağları mDNS'i devre dışı bırakır.
- **Duyurucu yoklama/duyuru aşamasında takılmış**: çok noktaya yayının engellendiği ana makineler, konteyner köprüleri, WSL veya arayüz değişimleri, yanıtlayıcıyı duyuru yapılmamış bir durumda bırakabilir. Gateway'e doğrudan, SSH, Tailnet veya geniş alan DNS-SD yolları üzerinden erişilmeye devam edilebilir; çok noktaya yayın kullanılamadığında `discovery.mdns.mode: "off"` veya `OPENCLAW_DISABLE_BONJOUR=1` ile LAN Bonjour'u devre dışı bırakın.
- **Docker köprü ağı**: Bonjour, algılanan konteynerlerde otomatik olarak devre dışı bırakılır. `OPENCLAW_DISABLE_BONJOUR=0` değerini yalnızca ana makine, macvlan veya mDNS özellikli başka bir ağ için ayarlayın.
- **Uyku/arayüz değişimi**: macOS, mDNS sonuçlarını geçici olarak kaybedebilir; yeniden deneyin.
- **Tarama çalışıyor ancak çözümleme başarısız oluyor**: makine adlarını basit tutun (emoji veya noktalama işaretlerinden kaçının), ardından Gateway'i yeniden başlatın. Hizmet örneği adı ana makine adından türetildiğinden, aşırı karmaşık adlar bazı çözümleyicilerin kafasını karıştırabilir.

## Kaçış dizili örnek adları (`\032`)

Bonjour/DNS-SD, hizmet örneği adlarındaki baytları genellikle ondalık `\DDD` dizileri olarak kaçış dizisine dönüştürür (boşluklar `\032` olur). Bu, protokol düzeyinde normaldir; kullanıcı arayüzleri görüntülemek için bunları çözmelidir (iOS, `BonjourEscapes.decode` kullanır).

## Etkinleştirme / devre dışı bırakma / yapılandırma

| Ayar                                                | Etki                                                                              |
| ---------------------------------------------------- | --------------------------------------------------------------------------------- |
| `openclaw plugins enable bonjour`                    | Varsayılan olarak etkinleştirilmediği ana makinelerde paketle gelen LAN keşif Plugin'ini etkinleştirir. |
| `openclaw plugins disable bonjour`                   | Paketle gelen Plugin'i devre dışı bırakarak LAN çok noktaya yayın duyurusunu devre dışı bırakır. |
| `OPENCLAW_DISABLE_BONJOUR=1` (veya `true`/`yes`/`on`)  | Plugin yapılandırmasını değiştirmeden LAN çok noktaya yayın duyurusunu devre dışı bırakır. |
| `OPENCLAW_DISABLE_BONJOUR=0` (veya `false`/`no`/`off`) | Algılanan konteynerlerin içi dahil olmak üzere LAN çok noktaya yayın duyurusunu zorla etkinleştirir. |
| `discovery.mdns.mode`                                | `off` \| `minimal` (varsayılan) \| `full` — yukarıdaki modlara bakın. |
| `gateway.bind`                                       | `~/.openclaw/openclaw.json` içindeki Gateway bağlama modunu denetler. |
| `OPENCLAW_SSH_PORT`                                  | `sshPort` duyurulduğunda SSH bağlantı noktasını geçersiz kılar (tam mod). |
| `OPENCLAW_TAILNET_DNS`                               | mDNS tam modu etkinleştirildiğinde TXT içinde bir MagicDNS ipucu yayımlar. |
| `OPENCLAW_CLI_PATH`                                  | Duyurulan CLI yolunu geçersiz kılar (tam mod). |

macOS ana makineleri, paketle gelen LAN keşif Plugin'ini varsayılan olarak otomatik başlatır. Bonjour Plugin'i etkinleştirildiğinde ve `OPENCLAW_DISABLE_BONJOUR` ayarlanmamışsa Bonjour, normal ana makinelerde duyuru yapar ve algılanan konteynerlerde (Docker, Fly.io makineleri ve yaygın konteyner çalışma zamanları) otomatik olarak devre dışı bırakılır.

## İlgili belgeler

- Keşif ilkesi ve aktarım seçimi: [Keşif](/tr/gateway/discovery)
- Node eşleştirme + onaylar: [Gateway eşleştirme](/tr/gateway/pairing)
