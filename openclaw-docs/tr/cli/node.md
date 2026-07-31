---
read_when:
    - Başsız Node ana bilgisayarını çalıştırma
    - system.run için macOS dışı bir Node eşleştirme
summary: '`openclaw node` (başsız Node ana bilgisayarı) için CLI referansı'
title: Node
x-i18n:
    generated_at: "2026-07-26T22:41:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 341539d05545ddcbf6175c34af7dca49332ba55906283b9933b9c9b1732c0e4d
    source_path: cli/node.md
    workflow: 16
---

# `openclaw node`

Gateway WebSocket'e bağlanan ve bu makinede
`system.run` / `system.which` sunan bir **başsız node ana makinesi** çalıştırın.

macOS'te menü çubuğu uygulaması, bu node ana makinesi çalışma zamanını zaten kendi
node bağlantısına gömer ve yerel Mac yetenekleri ekler. Mac'te `openclaw node run` komutunu
yalnızca uygulama olmadan başsız bir node kullanmayı özellikle istediğinizde kullanın. Her
ikisini de çalıştırmak, aynı makine için iki node kimliği oluşturur.

## Neden bir node ana makinesi kullanılmalı?

Ağınızdaki diğer makinelere tam bir macOS yardımcı uygulaması yüklemeden
ajanların bu makinelerde **komut çalıştırmasını** istediğinizde bir node ana makinesi kullanın.

Yaygın kullanım alanları:

- Uzak Linux/Windows makinelerinde (derleme sunucuları, laboratuvar makineleri, NAS) komut çalıştırın.
- exec'i Gateway üzerinde **korumalı alanda** tutarken onaylanan çalıştırmaları diğer ana makinelere devredin.
- Otomasyon veya CI node'ları için hafif, başsız bir yürütme hedefi sağlayın.

Yürütme, node ana makinesindeki **exec onayları** ve ajan başına izin listeleriyle
korunmaya devam eder; böylece komut erişimini kapsamı belirlenmiş ve açık tutabilirsiniz.

`openclaw node run`, bağlandıktan sonra plugin veya MCP destekli araçlar yayımlayabilir.
Gateway, eşleştirilmiş node'dan gelen tanımlayıcılara varsayılan olarak güvenir; ancak
her tanımlayıcının komutunun node'un onaylanmış komut yüzeyinde kalmasını zorunlu tutar.
Ajan, kabul edilen her tanımlayıcıyı normal bir plugin aracı olarak görür; ancak yürütme yine
`node.invoke` üzerinden gerçekleşir, dolayısıyla node bağlantısının kesilmesi aracı yeni
ajan çalıştırmalarından kaldırır. Gateway operatörleri yayını
`gateway.nodes.pluginTools.enabled: false` ile devre dışı bırakabilir.

Bildirimsel MCP araçları için node makinesindeki `openclaw.json` içinde
`nodeHost.mcp.servers` altına normal MCP sunucusu yapısını ekleyin, ardından node ana makinesini
yeniden başlatın. Node, onay geçitli `mcp.tools.call.v1` komut ailesini bildirir
ve bağlandıktan sonra listelenen araçları yayımlar; sunucu listesini daha sonra
değiştirmek yeniden eşleştirme gerektirmez. Bkz.
[Node üzerinde barındırılan MCP sunucuları](/tr/nodes#node-hosted-mcp-servers).

## Tarayıcı proxy'si (sıfır yapılandırma)

Node ana makineleri, node üzerinde `browser.enabled` devre dışı bırakılmamışsa
otomatik olarak bir tarayıcı proxy'si duyurur. Bu, ajanın ek yapılandırma olmadan
o node üzerinde tarayıcı otomasyonu kullanmasını sağlar.

Proxy varsayılan olarak node'un normal tarayıcı profili yüzeyini sunar.
`nodeHost.browserProxy.allowProfiles` ayarlanırsa proxy kısıtlayıcı hâle gelir:
izin listesinde bulunmayan profilleri hedefleme reddedilir ve kalıcı profil
oluşturma/silme yolları proxy üzerinden engellenir.

Gerekirse node üzerinde devre dışı bırakın:

```json5
{
  nodeHost: {
    browserProxy: {
      enabled: false,
    },
  },
}
```

## Çalıştırma (ön planda)

```bash
openclaw node run --host <gateway-host> --port 18789
```

Seçenekler:

- `--host <host>`: Gateway WebSocket ana makinesi (varsayılan: `127.0.0.1`)
- `--port <port>`: Gateway WebSocket bağlantı noktası (varsayılan: `18789`)
- `--context-path <path>`: Gateway WebSocket bağlam yolu (ör. `/openclaw-gw`). WebSocket URL'sine eklenir.
- `--tls`: Gateway bağlantısı için TLS kullan
- `--no-tls`: Yerel Gateway yapılandırması TLS'yi etkinleştirse bile düz metin Gateway bağlantısını zorunlu kıl
- `--tls-fingerprint <sha256>`: Beklenen TLS sertifikası parmak izi (sha256)
- `--node-id <id>`: Paylaşılan SQLite durumunda saklanan istemci örneği kimliğini geçersiz kıl (eşleştirmeyi sıfırlamaz)
- `--display-name <name>`: Node görünen adını geçersiz kıl

## Node ana makinesi için Gateway kimlik doğrulaması

`openclaw node run` ve `openclaw node install`, Gateway kimlik doğrulamasını yapılandırmadan/ortamdan çözümler (node komutlarında `--token`/`--password` bayrakları yoktur):

- Önce `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD` kontrol edilir.
- Ardından yerel yapılandırma yedeği: `gateway.auth.token` / `gateway.auth.password`.
- Yerel modda node ana makinesi, `gateway.remote.token` / `gateway.remote.password` değerlerini kasıtlı olarak devralmaz.
- `gateway.auth.token` / `gateway.auth.password`, SecretRef aracılığıyla açıkça yapılandırılmış ancak çözümlenmemişse node kimlik doğrulaması çözümlemesi kapalı biçimde başarısız olur (uzak yedekleme bunu maskelemez).
- `gateway.mode=remote` içinde uzak istemci alanları (`gateway.remote.token` / `gateway.remote.password`) da uzak öncelik kurallarına göre kullanılabilir.
- Node ana makinesi kimlik doğrulaması çözümlemesi yalnızca `OPENCLAW_GATEWAY_*` ortam değişkenlerini dikkate alır.

Düz metin bir `ws://` Gateway'e bağlanan node için geri döngü, özel IP
değişmez değerleri, `.local` ve Tailnet `*.ts.net` ana makineleri kabul edilir. Diğer
güvenilir özel DNS adları için `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` ayarlayın; bu ayar olmadan
node başlatma kapalı biçimde başarısız olur ve `wss://`, bir SSH tüneli veya
Tailscale kullanmanızı ister. Bu, bir `openclaw.json` yapılandırma anahtarı değil,
işlem ortamında açıkça etkinleştirilen bir seçenektir.
`openclaw node install`, yükleme komutu ortamında mevcut olduğunda bunu denetlenen
node hizmetinde kalıcı hâle getirir.

## Hizmet (arka planda)

Başsız bir node ana makinesini kullanıcı hizmeti olarak yükleyin (macOS'te launchd,
Linux'ta systemd, Windows'ta Windows Task Scheduler).

```bash
openclaw node install --host <gateway-host> --port 18789
```

Seçenekler:

- `--host <host>`: Gateway WebSocket ana makinesi (varsayılan: `127.0.0.1`)
- `--port <port>`: Gateway WebSocket bağlantı noktası (varsayılan: `18789`)
- `--context-path <path>`: Gateway WebSocket bağlam yolu (ör. `/openclaw-gw`). WebSocket URL'sine eklenir.
- `--tls`: Gateway bağlantısı için TLS kullan
- `--tls-fingerprint <sha256>`: Beklenen TLS sertifikası parmak izi (sha256)
- `--node-id <id>`: Paylaşılan SQLite durumunda saklanan istemci örneği kimliğini geçersiz kıl (eşleştirmeyi sıfırlamaz)
- `--display-name <name>`: Node görünen adını geçersiz kıl
- `--runtime <runtime>`: Hizmet çalışma zamanı (`node`)
- `--force`: Zaten yüklüyse yeniden yükle/üzerine yaz

Hizmeti yönetin:

```bash
openclaw node status
openclaw node start
openclaw node stop
openclaw node restart
openclaw node uninstall
```

Ön plandaki bir node ana makinesi için `openclaw node run` kullanın (hizmet yok).

Hizmet komutları, makine tarafından okunabilir çıktı için `--json` kabul eder.

Node ana makinesi, Gateway yeniden başlatmalarını ve ağ kapanmalarını işlem içinde
yeniden dener. Gateway, terminal niteliğinde bir token/parola/bootstrap kimlik doğrulaması
duraklaması bildirirse node ana makinesi kapanış ayrıntısını günlüğe kaydeder ve sıfırdan
farklı bir kodla çıkar; böylece launchd/systemd/Task Scheduler onu güncel yapılandırma ve
kimlik bilgileriyle yeniden başlatabilir. Eşleştirme gerektiren duraklamalar, bekleyen
isteğin onaylanabilmesi için ön plan akışında kalır.

## Eşleştirme

İlk bağlantı, Gateway üzerinde bekleyen bir cihaz eşleştirme isteği (`role: node`) oluşturur.

Gateway ana makinesi node ana makinesine etkileşimsiz olarak SSH ile bağlanabiliyorsa
(aynı kullanıcı, güvenilir ana makine anahtarı), bekleyen istek otomatik olarak onaylanır:
Gateway, node ana makinesinde SSH üzerinden `openclaw node identity --json` çalıştırır ve
cihaz anahtarı tam olarak eşleştiğinde onay verir. Bu özellik varsayılan olarak açıktır;
gereksinimler ve nasıl devre dışı bırakılacağı (`gateway.nodes.pairing.sshVerify: false`) için
[SSH ile doğrulanan cihazın otomatik onayı](/tr/gateway/pairing#ssh-verified-device-auto-approval-default)
bölümüne bakın.

Aksi takdirde şu komutlarla manuel olarak onaylayın:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Gateway'in doğruladığı yerel node kimliğini inceleyin:

```bash
openclaw node identity --json
```

`state/openclaw.sqlite` içindeki `primary` satırından cihaz kimliğini ve ortak anahtarı
yazdırır; veritabanını veya yeni bir kimliği asla oluşturmaz.

Sıkı biçimde denetlenen node ağlarında Gateway operatörü, güvenilir CIDR'lerden
ilk kez yapılan node eşleştirmesinin otomatik onaylanmasını açıkça etkinleştirebilir:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

Bu özellik varsayılan olarak devre dışıdır (`autoApproveCidrs` ayarlanmamıştır). Yalnızca
istenen kapsamları olmayan yeni `role: node` eşleştirmelerine ve Gateway'in güvendiği
bir istemci IP'sinden gelen isteklere uygulanır. Operatör/tarayıcı istemcileri, Control UI,
WebChat ve rol, kapsam, meta veri veya ortak anahtar yükseltmeleri hâlâ manuel onay gerektirir.

Node, değiştirilmiş kimlik doğrulama ayrıntılarıyla (rol/kapsamlar/ortak anahtar) eşleştirmeyi
yeniden denerse önceki bekleyen isteğin yerini yeni bir `requestId` alır.
Onaylamadan önce `openclaw devices list` komutunu yeniden çalıştırın.

### Kimlik ve eşleştirme durumu

Başsız node, istemci örneği kimliğini Gateway'in eşleştirme ve yönlendirme için
kullandığı imzalı cihaz kimliğinden ayırır. Bu durum OpenClaw durum dizininde
(varsayılan olarak `~/.openclaw` veya ayarlandığında `$OPENCLAW_STATE_DIR`) bulunur:

| Durum                                                    | Amaç                                                                                                                          |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `state/openclaw.sqlite` (`node_host_config`)             | İstemci örneği kimliği, görünen ad ve Gateway bağlantı meta verileri. İstemci bu kimliği `instanceId` olarak gönderir.                     |
| `state/openclaw.sqlite` (`device_identities`, `primary`) | İmzalı Ed25519 anahtar çifti ve türetilmiş cihaz kimliği. İmzalı bağlantılarda bu cihaz kimliği, yönlendirilen node kimliği ve eşleştirme kimliğidir. |
| `state/openclaw.sqlite` (`device_auth_tokens`)           | Kriptografik cihaz kimliğine ve role göre anahtarlanmış eşleştirilmiş cihaz token'ları.                                                                 |

`--node-id`, yalnızca paylaşılan SQLite durumundaki istemci örneği kimliğini değiştirir.
Kriptografik cihaz kimliğini değiştirmez veya eşleştirme kimlik doğrulamasını temizlemez.
Kullanımdan kaldırılmış bir `node.json` öğesini `openclaw doctor --fix` ile taşımak da
eşleştirmeyi sıfırlamaz. Bir node'u iptal edip yeniden eşleştirmek için:

1. Gateway üzerinde `openclaw nodes remove --node <id|name|ip>` komutunu çalıştırın.
2. Node üzerinde yüklü hizmeti `openclaw node restart` ile yeniden başlatın veya
   ön plandaki `openclaw node run` komutunu durdurup yeniden çalıştırın. Bu işlem
   cihaz eşleştirme akışını başlatır. `openclaw devices list` bir istek göstermiyorsa
   ve node `AUTH_DEVICE_TOKEN_MISMATCH` bildiriyorsa node'u bir kez daha yeniden başlatın
   veya yeniden çalıştırın. Reddedilen deneme, artık iptal edilmiş yerel token'ı
   temizler; sonraki deneme eşleştirme isteyebilir.
3. Gateway üzerinde `openclaw devices list`, ardından
   `openclaw devices approve <deviceRequestId>` komutunu çalıştırın.
4. Node'u yeniden başlatın veya yeniden çalıştırın. Eşleştirme için duraklatılan
   bir istemci onaydan sonra otomatik olarak devam etmez; bu yeniden bağlantı ayrı
   komut yüzeyi isteğini oluşturur.
5. Gateway üzerinde `openclaw nodes pending`, ardından
   `openclaw nodes approve <nodeRequestId>` komutunu çalıştırın.

İki istek kimliği birbirinden farklıdır. Uygulanabilir bir güvenilir CIDR ilkesi,
ilk cihaz eşleştirme adımını otomatik olarak onaylayabilir; komut yüzeyi onayı ayrı
bir denetim olarak kalır.

Eski OpenClaw sürümleri node ana makinesi durumunu `node.json` içinde, imzalı
kimliği `identity/device.json` içinde ve eşleştirilmiş kimlik doğrulamasını
`identity/device-auth.json` içinde saklıyordu. Node ana makinesini durdurun ve
`openclaw doctor --fix` komutunu bir kez çalıştırın; Doctor kullanımdan kaldırılmış her kaynağı
sahiplenir, doğrular, standart SQLite satırını içe aktarıp doğrular ve ardından eski
dosyayı kaldırır. Kullanımdan kaldırılmış dosyalardan biri veya yarıda kesilmiş bir Doctor
sahiplenmesi mevcutken normal node komutları bu onarım talimatıyla kapalı biçimde başarısız olur.
`state/openclaw.sqlite` öğesini gizli tutun; cihaz anahtar çiftini ve kimlik doğrulama token'larını içerir.

## Exec onayları

`system.run`, yerel exec onaylarıyla geçitlenir:

- `$OPENCLAW_STATE_DIR/exec-approvals.json` veya
  değişken ayarlanmamışsa `~/.openclaw/exec-approvals.json`
- [Exec onayları](/tr/tools/exec-approvals)
- `openclaw approvals --node <id|name|ip>` (Gateway üzerinden düzenleyin)

Onaylanmış zaman uyumsuz node exec işlemi için OpenClaw, istemde bulunmadan önce standart
bir `systemRunPlan` hazırlar. Daha sonra onaylanan `system.run` iletimi,
saklanan bu planı yeniden kullanır; böylece onay isteği oluşturulduktan sonra komut/cwd/oturum
alanlarında yapılan düzenlemeler, node'un ne yürüteceğini değiştirmek yerine reddedilir.

## İlgili

- [CLI başvurusu](/tr/cli)
- [Node'lar](/tr/nodes)
