---
read_when:
    - Bonjour keşfini/duyurusunu uygulama veya değiştirme
    - Uzak bağlantı modlarını ayarlama (doğrudan ve SSH)
    - Uzak node'lar için node keşfi + eşleştirme tasarımı
summary: Gateway'i bulmak için Node keşfi ve aktarımlar (Bonjour, Tailscale, SSH)
title: Keşif ve taşıma mekanizmaları
x-i18n:
    generated_at: "2026-07-26T23:56:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3a3f1a6a1212ab0bc7021e77c88de059edcb8e09eff90d3e1e59451b9b20876b
    source_path: gateway/discovery.md
    workflow: 16
---

OpenClaw'ın birbiriyle ilişkili ancak birbirinden farklı iki keşif sorunu vardır:

1. **Operatör uzaktan denetimi**: başka bir yerde çalışan Gateway'i denetleyen macOS menü çubuğu uygulaması.
2. **Node eşleştirme**: iOS/Android'in (ve gelecekteki Node'ların) bir Gateway bulması ve güvenli şekilde eşleşmesi.

Tüm ağ keşfi/reklamı **Node Gateway** içinde gerçekleşir
(`openclaw gateway`); istemciler (Mac uygulaması, iOS) yalnızca tüketicidir.

## Terimler

- **Gateway**: duruma (oturumlar,
  eşleştirme, Node kayıt defteri) sahip olan ve kanalları çalıştıran, uzun süre çalışan tek bir süreç. Çoğu kurulumda ana makine başına bir tane kullanılır;
  yalıtılmış çoklu Gateway kurulumları mümkündür.
- **Gateway WS (denetim düzlemi)**: varsayılan olarak `127.0.0.1:18789`
  üzerindeki WebSocket uç noktasıdır; `gateway.bind` aracılığıyla LAN/tailnet'e bağlayın.
- **Doğrudan WS aktarımı**: LAN/tailnet'e dönük bir Gateway WS uç noktasıdır (SSH yoktur).
- **SSH aktarımı (geri dönüş seçeneği)**: `127.0.0.1:18789` bağlantı noktasını
  SSH üzerinden yönlendirerek uzaktan denetim.
- **Eski TCP köprüsü (kaldırıldı)**: eski Node aktarımıdır (bkz.
  [Köprü protokolü](/tr/gateway/bridge-protocol)); artık keşif için duyurulmaz
  ve güncel derlemelerin parçası değildir.

Protokol ayrıntıları: [Gateway protokolü](/tr/gateway/protocol),
[Köprü protokolü (eski)](/tr/gateway/bridge-protocol).

## Neden hem doğrudan bağlantı hem de SSH vardır?

- **Doğrudan WS**, aynı ağda ve bir tailnet içinde en iyi kullanıcı deneyimini sunar: Bonjour üzerinden LAN
  otomatik keşfi, Gateway'in yönettiği eşleştirme token'ları ve ACL'ler
  ve kabuk erişimi gerektirmemesi.
- **SSH** evrensel geri dönüş seçeneğidir: ilgisiz ağlar arasında bile SSH erişiminizin olduğu her yerde
  çalışır, çok noktaya yayın/mDNS sorunlarından etkilenmez ve SSH dışında yeni bir
  gelen bağlantı noktası gerektirmez.

## Keşif girdileri

### 1) Bonjour / DNS-SD

Çok noktaya yayın Bonjour, mümkün olan en iyi şekilde çalışır ve ağlar arasında geçiş yapmaz. OpenClaw ayrıca
aynı Gateway işaretinin yapılandırılmış geniş alan DNS-SD
alan adı üzerinden taranmasını destekler; böylece keşif hem aynı LAN'daki `local.` öğesini hem de ağlar arası keşif için yapılandırılmış
tek noktaya yayın DNS-SD alan adını kapsayabilir.

Paketle birlikte gelen `bonjour` Plugin etkinleştirildiğinde **Gateway**, WS uç noktasını Bonjour üzerinden duyurur;
istemciler tarama yapıp bir "Gateway seçin" listesi gösterir,
ardından seçilen uç noktayı saklar.

Sorun giderme ve işaret ayrıntıları: [Bonjour](/tr/gateway/bonjour).

#### Hizmet işareti ayrıntıları

- Hizmet türü: `_openclaw-gw._tcp` (Gateway aktarım işareti).
- TXT anahtarları (gizli değildir):

  | Anahtar                     | Notlar                                                                                                                                                           |
  | --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `role=gateway`              | Her zaman bulunur.                                                                                                                                               |
  | `transport=gateway`         | Her zaman bulunur.                                                                                                                                               |
  | `displayName=<name>`        | Operatör tarafından yapılandırılan görünen ad.                                                                                                                    |
  | `lanHost=<hostname>.local`  | Yalnızca LAN mDNS duyurucusu; geniş alan DNS-SD tarafından yazılmaz.                                                                                              |
  | `gatewayPort=18789`         | Gateway WS + HTTP bağlantı noktası.                                                                                                                               |
  | `gatewayTls=1`              | Yalnızca TLS etkinleştirildiğinde.                                                                                                                                |
  | `gatewayTlsSha256=<sha256>` | Yalnızca TLS etkinleştirildiğinde ve parmak izi kullanılabilir olduğunda.                                                                                         |
  | `tailnetDns=<magicdns>`     | İsteğe bağlı ipucu; Tailscale kullanılabilir olduğunda otomatik olarak algılanır.                                                                                 |
  | `sshPort=<port>`            | Yalnızca `discovery.mdns.mode="full"` olduğunda bulunur; hem LAN duyurucusunda hem de geniş alan DNS-SD'de varsayılan `"minimal"` modunda atlanır (SSH varsayılan olarak `22` kullanır). |
  | `cliPath=<path>`            | `sshPort` ile aynı `discovery.mdns.mode="full"` koşulu; CLI yolu için uzaktan kurulum ipucu.                                                                     |

  Gelecekteki bir tuval ana makinesi bağlantı noktası için Plugin keşif sözleşmesinde bir `canvasPort` TXT anahtarı tanımlanmıştır,
  ancak güncel hiçbir kod yolu bir değer ayarlamadığından
  bugün hiçbir zaman yayınlanmaz.

Güvenlik notları:

- Bonjour/mDNS TXT kayıtlarının **kimliği doğrulanmaz**. İstemciler TXT
  değerlerini yalnızca kullanıcı deneyimi ipuçları olarak değerlendirmelidir.
- Yönlendirme (ana makine/bağlantı noktası), TXT tarafından sağlanan `lanHost`, `tailnetDns` veya `gatewayPort` yerine
  **çözümlenmiş hizmet uç noktasını** (SRV + A/AAAA) tercih etmelidir.
- TLS sabitleme, duyurulan bir `gatewayTlsSha256` değerinin önceden saklanmış bir
  sabitlemeyi geçersiz kılmasına asla izin vermemelidir.
- Seçilen yol güvenli/TLS tabanlı olduğunda iOS/Android Node'ları ilk kez bir sabitleme kaydetmeden önce
  açık bir "bu parmak izine güven" onayı (bant dışı doğrulama)
  istemelidir.

Etkinleştirme, devre dışı bırakma ve geçersiz kılma:

- `openclaw plugins enable bonjour`, LAN çok noktaya yayın duyurusunu etkinleştirir.
- `openclaw.json` içindeki `discovery.mdns.mode`, mDNS yayınını denetler:
  `"minimal"` (varsayılan), `"full"` (hem LAN
  işaretine hem de tüm geniş alan DNS-SD bölgelerine `cliPath`/`sshPort` ekler) veya `"off"` (mDNS'yi devre dışı bırakır).
- `OPENCLAW_DISABLE_BONJOUR=1`, duyuruyu zorla devre dışı bırakır; `discovery.mdns.mode="off"`
  bunu bağımsız olarak devre dışı bırakır. `OPENCLAW_DISABLE_BONJOUR=0`, algılanan bir kapsayıcı
  (Docker, containerd, Kubernetes, LXC) içinde Plugin'in otomatik devre dışı bırakmasını geçersiz kılan açık bir
  katılımdır; `discovery.mdns.mode="off"` değerini geçersiz kılmaz. Paketle birlikte gelen `bonjour` Plugin,
  macOS ana makinelerinde (`enabledByDefaultOnPlatforms: ["darwin"]`) otomatik başlatılır ve algılanan
  kapsayıcıların içinde otomatik devre dışı bırakılır; Linux, Windows ve diğer kapsayıcılı
  dağıtımlar açıkça `plugins enable bonjour` gerektirir.
- `~/.openclaw/openclaw.json` içindeki `gateway.bind`, Gateway bağlama modunu denetler.
- `OPENCLAW_SSH_PORT`, duyurulan SSH bağlantı noktasını geçersiz kılar (yalnızca
  `discovery.mdns.mode="full"` olduğunda etkili olur).
- `OPENCLAW_TAILNET_DNS`, bir `tailnetDns` ipucu (MagicDNS) yayınlar.
- `OPENCLAW_CLI_PATH`, duyurulan CLI yolunu geçersiz kılar.

### 2) Tailnet (ağlar arası)

Farklı fiziksel ağlardaki Gateway'ler için Bonjour yardımcı olmaz. Önerilen
doğrudan hedef, bir Tailscale MagicDNS adı (tercih edilir) veya
kararlı bir tailnet IP adresidir.

Gateway, Tailscale altında çalıştığını algılarsa istemciler için isteğe bağlı bir ipucu olarak
`tailnetDns` yayınlar (geniş alan işaretleri dâhil).
macOS uygulaması, Gateway keşfinde ham Tailscale IP'leri yerine MagicDNS adlarını tercih eder;
MagicDNS otomatik olarak güncel IP'ye çözümlendiği için tailnet IP'leri değiştiğinde (Node yeniden başlatmaları,
CGNAT yeniden ataması) bu yöntem güvenilirliğini korur.

Mobil Node eşleştirmesinde keşif ipuçları, tailnet/genel yollarda aktarım güvenliğini
asla gevşetmez:

- iOS/Android yine de ilk tailnet/genel bağlantı için güvenli bir yol
  (`wss://` veya Tailscale Serve/Funnel) gerektirir.
- Keşfedilen ham bir tailnet IP'si, düz metin uzak
  `ws://` kullanım izni değil, bir yönlendirme ipucudur.
- Özel LAN üzerinden doğrudan bağlantı `ws://` desteği sürer.
- Mobil Node'larda en basit Tailscale yolu için Tailscale Serve kullanın;
  böylece hem keşif hem de kurulum aynı güvenli MagicDNS uç noktasına çözümlenir.

### 3) Manuel / SSH hedefi

Doğrudan bir yol olmadığında (veya doğrudan bağlantı devre dışı bırakıldığında), istemciler geri döngü Gateway bağlantı noktasını yönlendirerek
her zaman SSH üzerinden bağlanabilir. Bkz.
[Uzaktan erişim](/tr/gateway/remote).

## Aktarım seçimi (istemci politikası)

1. Eşleştirilmiş bir doğrudan uç nokta yapılandırılmış ve erişilebilir durumdaysa onu kullanın.
2. Aksi takdirde keşif, `local.` veya yapılandırılmış geniş alan
   alan adında bir Gateway bulursa tek dokunuşla "Bu Gateway'i kullan" seçeneğini sunun ve bunu
   doğrudan uç nokta olarak kaydedin.
3. Aksi takdirde bir tailnet DNS/IP yapılandırılmışsa doğrudan bağlantıyı deneyin. Tailnet/genel yollardaki mobil Node'lar için
   doğrudan bağlantı, düz metin uzak `ws://` değil, güvenli bir uç nokta anlamına gelir.
4. Aksi takdirde SSH'ye geri dönün.

## Eşleştirme ve kimlik doğrulama (doğrudan aktarım)

Node/istemci kabulünde doğruluk kaynağı Gateway'dir:

- Eşleştirme istekleri Gateway'de oluşturulur/onaylanır/reddedilir (bkz.
  [Gateway eşleştirmesi](/tr/gateway/pairing)).
- Gateway, kimlik doğrulamayı (token/anahtar çifti), kapsamları/ACL'leri (her yönteme yönelik ham
  bir proxy değildir) ve hız sınırlarını uygular.

## Bileşenlere göre sorumluluklar

- **Gateway**: keşif işaretlerini duyurur, eşleştirme kararlarının sahibidir ve
  WS uç noktasını barındırır.
- **macOS uygulaması**: Gateway seçmenize yardımcı olur, eşleştirme istemlerini gösterir ve SSH'yi
  yalnızca geri dönüş seçeneği olarak kullanır.
- **iOS/Android Node'ları**: kolaylık sağlamak için Bonjour'u tarar ve eşleştirilmiş
  Gateway WS'ye bağlanır.

## İlgili konular

- [Uzaktan erişim](/tr/gateway/remote)
- [Tailscale](/tr/gateway/tailscale)
- [Bonjour keşfi](/tr/gateway/bonjour)
