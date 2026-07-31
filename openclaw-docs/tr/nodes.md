---
read_when:
    - iOS/watchOS/Android Node'larını bir Gateway ile eşleştirme
    - Agent bağlamı için Node tuvalini/kamerasını kullanma
    - Yeni Node komutları veya CLI yardımcıları ekleme
summary: 'Node''lar: canvas/kamera/ekran/cihaz/bildirimler/sistem için eşleştirme, yetenekler, izinler ve CLI yardımcıları'
title: Node'lar
x-i18n:
    generated_at: "2026-07-26T23:45:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b4f7c80491d713777e1ba5b8f55c88bd9fa48be602b504e6ac6ba00cd12a4313
    source_path: nodes/index.md
    workflow: 16
---

Bir **node**, Gateway'e `role: "node"` ile bağlanan ve `node.invoke` üzerinden bir komut yüzeyi (ör. `canvas.*`, `camera.*`, `device.*`, `notifications.*`, `system.*`) sunan yardımcı bir cihazdır (macOS/iOS/watchOS/Android/başsız). Çoğu node, operatör bağlantı noktasındaki Gateway WebSocket'i kullanır. İsteğe bağlı doğrudan Apple Watch node'u, watchOS sıradan uygulamaların genel amaçlı düşük düzey ağ iletişimini engellediğinden aynı bağlantı noktasında imzalı HTTPS yoklaması kullanır. Protokol ayrıntıları: [Gateway protokolü](/tr/gateway/protocol).

Eski aktarım: [Bridge protokolü](/tr/gateway/bridge-protocol) (TCP JSONL; mevcut node'lar için yalnızca geçmişe yöneliktir).

macOS ayrıca **node modunda** çalışabilir: Menü çubuğu uygulaması Gateway'in
WS sunucusuna bir node olarak bağlanır (böylece `openclaw nodes …` bu Mac üzerinde çalışır). Uygulama,
`openclaw node run` tarafından kullanılan aynı node ana makinesi komut yüzeyine yerel Canvas, kamera, ekran, bildirim ve bilgisayar denetimi komutları
ekler. Bu Mac'te
ikinci bir CLI node'u başlatmayın; uygulama, eşleşen CLI node ana makinesi çalışma zamanını
dahili bir çalışan olarak yürütür ve tek Gateway bağlantısı ve node kimliği olarak kalır.

Node'lar gateway değil, **çevre birimleridir**: Gateway hizmetini çalıştırmazlar ve kanal mesajları (Telegram, WhatsApp vb.) node'lara değil gateway'e ulaşır.

Sorun giderme çalışma kılavuzu: [/nodes/troubleshooting](/tr/nodes/troubleshooting)

## Eşleştirme + durum

Node'lar **cihaz eşleştirmesi** kullanır. Bir node, bağlantı sırasında imzalı bir cihaz kimliği sunar; Gateway, `role: node` için bir cihaz eşleştirme isteği oluşturur. Cihazlar CLI'si (veya kullanıcı arayüzü) üzerinden onaylayın. Doğrudan Apple Watch kurulumu, sabit ve düşük riskli komut yüzeyini onaylamak için yönetici tarafından oluşturulan kısa ömürlü, yalnızca node'a yönelik bir kurulum kodu kullanır; daha sonraki yetenek genişletmeleri yine normal onay gerektirir.

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
```

Bekleyen eşleştirme istekleri, cihazın son yeniden denemesinden 5 dakika sonra sona erer — yeniden bağlanmayı sürdüren bir cihaz, birkaç dakikada bir yeni istem oluşturmak yerine tek bekleyen isteğini (ve `requestId`) etkin tutar; tam istek/onay yaşam döngüsü için [Node eşleştirmesi](/tr/gateway/pairing) bölümüne bakın. Bir node değiştirilmiş kimlik doğrulama ayrıntılarıyla (rol/kapsamlar/açık anahtar) yeniden denerse önceki bekleyen isteğin yerini yeni bir istek alır ve yeni bir `requestId` oluşturulur — istemciler, yerini başka bir isteğin aldığı istek için bir `device.pair.resolved` olayı alır ve onaylamadan önce `openclaw devices list` komutunu yeniden çalıştırmanız gerekir.

- `nodes status`, cihaz eşleştirme rolü `node` içerdiğinde bir node'u **eşleştirilmiş** olarak işaretler.
- Bağlı bir yerel Mac, birleştirilmiş fiziksel giriş etkinliğini
  **Settings -> Permissions -> Active computer detection** üzerinden etkinleştirebilir. Erişilebilirlik izni de
  gereklidir. Gateway, uygun en güncel Mac'i
  `active` olarak işaretler, agente kararlı bir node kimliği ipucu verir ve gecikmeli bir geri dönüşten
  önce node bağlantı uyarılarını buraya yönlendirir. Kurulum, gizlilik, zamanlama ve
  sorun giderme için [Etkin bilgisayar varlığı](/tr/nodes/presence) bölümüne bakın.
- Cihaz eşleştirme kaydı, kalıcı onaylanmış rol sözleşmesidir. Token döndürme bu sözleşmenin içinde kalır; eşleştirilmiş bir node'u, eşleştirme onayının hiçbir zaman vermediği bir role yükseltemez.
- `node.pair.*` (CLI: `openclaw nodes pending/approve/reject/remove/rename`), node'un yeniden bağlantılar boyunca onaylanmış komut/yetenek yüzeyini izleyen, gateway'e ait ayrı bir node eşleştirme deposudur. Aktarım kimlik doğrulamasını **denetlemez** — bunu cihaz eşleştirmesi yapar.
- `openclaw nodes remove --node <id|name|ip>`, bir node eşleştirmesini kaldırır. Cihaz destekli bir node için eşleştirilmiş cihaz deposundaki cihazın `node` rolünü iptal eder ve o cihazın node rolü oturumlarının bağlantısını keser: birden çok role sahip cihaz satırını korur ve yalnızca `node` rolünü kaybederken, yalnızca node rolüne sahip cihaz satırı silinir. Ayrıca ayrı node eşleştirme deposundaki eşleşen tüm girdileri temizler. `operator.pairing`, diğer cihazlardaki operatör olmayan node satırlarını kaldırabilir; birden çok role sahip bir cihazda kendi node rolünü iptal eden cihaz token'lı bir çağıran ayrıca `operator.admin` gerektirir.
- Onay kapsamı, bekleyen isteğin bildirdiği komutlara göre belirlenir:
  - komutsuz istek: `operator.pairing`
  - exec dışı node komutları: `operator.pairing` + `operator.write`
  - `system.run` / `system.run.prepare` / `system.which`: `operator.pairing` + `operator.admin`

## Sürüm uyumsuzluğu ve yükseltme sırası

Gateway WebSocket, N-1 protokol aralığındaki kimliği doğrulanmış node istemcilerini kabul eder.
Bu nedenle mevcut v4 Gateway, bağlantı hem `role: "node"` hem de
`client.mode: "node"` bildirdiğinde v3 node'ları kabul eder. Operatör ve kullanıcı arayüzü oturumları
yine de mevcut protokolü kullanmalıdır.

Aşamalı filo yükseltmelerinde önce Gateway'i, ardından her node'u yükseltin.
Bir N-1 node yükseltilirken görünür ve yönetilebilir kalır; Gateway,
bir yükseltme önerisiyle birlikte `legacy node protocol accepted` günlüğünü kaydeder. Eşleştirme,
cihaz kimlik doğrulaması, komut izin listeleri ve exec onayları geçerliliğini korur.
Plugin'e ait yetenekler ve komutlar, node mevcut protokole yükseltilene kadar
gizli kalır. N-1'den eski node'lar yeniden bağlanmadan önce
bant dışı bir yükseltme gerektirir.

Doğrudan watchOS HTTPS aktarımı mevcut protokol sürümünü gerektirir; doğrudan modu
etkinleştirmeden önce watch uygulamasını Gateway ile birlikte güncelleyin.

## Uzak node ana makinesi (system.run)

Gateway'iniz bir makinede çalışırken komutları başka bir makinede yürütmek istediğinizde bir **node ana makinesi** kullanın. Model yine **gateway** ile iletişim kurar; `host=node` seçildiğinde gateway, `exec` çağrılarını **node ana makinesine** iletir.

| Rol                  | Sorumluluk                                                           |
| -------------------- | -------------------------------------------------------------------- |
| Gateway ana makinesi | Mesajları alır, modeli çalıştırır, araç çağrılarını yönlendirir.      |
| Node ana makinesi    | Node makinesinde `system.run`/`system.which` yürütür.      |
| Onaylar              | Node ana makinesinde `~/.openclaw/exec-approvals.json` aracılığıyla uygulanır.      |

Onay notu:

- Onay destekli node çalıştırmaları, tam istek bağlamına bağlanır. Exec yolu, onaydan önce kurallı bir `systemRunPlan` hazırlar; onay verildikten sonra gateway, daha sonra çağıran tarafından düzenlenmiş komut/cwd/oturum alanlarını değil, depolanan bu planı iletir ve çalıştırmadan önce çalışma dizinini yeniden doğrular.
- Doğrudan kabuk/çalışma zamanı dosyası yürütmelerinde OpenClaw ayrıca mümkün olan en iyi şekilde tek bir somut yerel dosya işlenenini bağlar ve bu dosya yürütmeden önce değişirse çalıştırmayı reddeder.
- OpenClaw bir yorumlayıcı/çalışma zamanı komutu için tam olarak bir somut yerel dosya belirleyemezse tam çalışma zamanı kapsamı varmış gibi davranmak yerine onay destekli yürütme reddedilir. Daha geniş yorumlayıcı semantiği için korumalı alan, ayrı ana makineler veya açık bir güvenilir izin listesi/tam iş akışı kullanın.

### Bir node ana makinesi başlatma (ön plan)

Node makinesinde:

```bash
openclaw node run --host <gateway-host> --port 18789 --display-name "Build Node"
```

`node run` ayrıca `--context-path` (Gateway WS bağlam yolu), `--tls`, `--tls-fingerprint <sha256>` ve `--node-id` (eski istemci örneği kimliğini geçersiz kılar; bu işlem eşleştirmeyi sıfırlamaz) seçeneklerini kabul eder. macOS'te `device.apps` duyurmak için `--share-installed-apps` geçirin; paylaşım varsayılan olarak kapalıdır. Önceden kaydedilmiş bir etkinleştirmeyi devre dışı bırakmak için `--no-share-installed-apps` kullanın.

### SSH tüneli üzerinden uzak gateway (geri döngü bağlama)

Gateway geri döngüye bağlanırsa (`gateway.bind=loopback`, yerel modda varsayılan), uzak node ana makineleri doğrudan bağlanamaz. Bir SSH tüneli oluşturun ve node ana makinesini tünelin yerel ucuna yönlendirin.

Örnek (node ana makinesi -> gateway ana makinesi):

```bash
# Terminal A (çalışır durumda tutun): yerel 18790'ı gateway 127.0.0.1:18789'a iletin
ssh -N -L 18790:127.0.0.1:18789 user@gateway-host

# Terminal B: gateway token'ını dışa aktarın ve tünel üzerinden bağlanın
export OPENCLAW_GATEWAY_TOKEN="<gateway-token>"
openclaw node run --host 127.0.0.1 --port 18790 --display-name "Build Node"
```

Notlar:

- `openclaw node run`, token veya parola kimlik doğrulamasını destekler.
- Ortam değişkenleri tercih edilir: `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`.
- Yapılandırma geri dönüşü: `gateway.auth.token` / `gateway.auth.password`.
- Yerel modda node ana makinesi, `gateway.remote.token` / `gateway.remote.password` değerlerini kasıtlı olarak yok sayar.
- Uzak modda `gateway.remote.token` / `gateway.remote.password`, uzak öncelik kurallarına göre kullanılabilir.
- Etkin yerel `gateway.auth.*` SecretRef'leri yapılandırılmış ancak çözümlenmemişse node ana makinesi kimlik doğrulaması güvenli biçimde başarısız olur.
- Node ana makinesi kimlik doğrulama çözümlemesi yalnızca `OPENCLAW_GATEWAY_*` ortam değişkenlerini dikkate alır.

### Bir node ana makinesi başlatma (hizmet)

```bash
openclaw node install --host <gateway-host> --port 18789 --display-name "Build Node"
openclaw node start
openclaw node restart
```

`node install` ayrıca `--context-path`, `--tls`, `--tls-fingerprint`, `--node-id` (yalnızca eski istemci örneği kimliği), `--share-installed-apps` / `--no-share-installed-apps`, `--runtime <node>` (varsayılan: node) ve yeniden kurulum için `--force` seçeneklerini kabul eder. `node status`, `node stop` ve `node uninstall` da kullanılabilir.

### Eşleştirme + adlandırma

Gateway ana makinesinde:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Node değiştirilmiş kimlik doğrulama ayrıntılarıyla yeniden denerse `openclaw devices list` komutunu yeniden çalıştırın ve mevcut `requestId` değerini onaylayın.

Adlandırma seçenekleri:

- `openclaw node run` / `openclaw node install` üzerindeki `--display-name` (istemci örneği kimliği ve Gateway bağlantı meta verileriyle birlikte paylaşılan `node_host_config` SQLite satırında kalıcı olur).
- `openclaw nodes rename --node <id|name|ip> --name "Build Node"` (gateway geçersiz kılması).

### Node tarafından barındırılan MCP sunucuları

MCP sunucularını Gateway'de değil, node makinesindeki
`openclaw.json` içinde yapılandırın:

```json5
{
  nodeHost: {
    mcp: {
      servers: {
        localDocs: {
          command: "npx",
          args: ["-y", "@modelcontextprotocol/server-filesystem", "/srv/docs"],
          toolFilter: {
            include: ["read_*", "search"],
          },
        },
        internalApi: {
          url: "https://mcp.internal.example/mcp",
          transport: "streamable-http",
          headers: {
            Authorization: "Bearer ${INTERNAL_MCP_TOKEN}",
          },
        },
      },
    },
  },
}
```

Başsız node ana makinesi bu sunucuları başlatır, araçlarını listeler ve bağlandıktan sonra
tanımlayıcıları yayımlar. Araç çağrıları `mcp.tools.call.v1` üzerinden
o node'a geri döner; Gateway'in eşleşen bir MCP yapılandırmasına veya JS
Plugin'ine ihtiyacı yoktur. OAuth MCP sunucuları, node tarafından barındırılan bu v1 yolunda desteklenmez.

Mevcut node ana makineleri, hiçbir MCP sunucusu yapılandırılmamış olsa bile ilk
eşleştirmeleri sırasında yerleşik `mcp.tools.call.v1` komut ailesini bildirir. Daha
eski bir OpenClaw sürümünde eşleştirilmiş bir node, node ana makinesi
güncellendikten sonra bir defaya mahsus komut yüzeyi yükseltmesi isteyebilir. Onaylanan komut ailesi değişmediğinden
bundan sonra sunucu eklemek, kaldırmak veya filtrelemek yeniden eşleştirme
gerektirmez. Node MCP yapılandırma değişikliklerini uygulamak için
`openclaw node run` veya `openclaw node restart` yeniden başlatın;
node ana makinesi bu yapılandırmayı izlemez.

Gateway operatörleri, node tarafından barındırılan MCP araçları dahil olmak üzere eşleştirilmiş node'ların yayımladığı
tüm agent tarafından görülebilen araçları
`gateway.nodes.pluginTools.enabled: false` ile yok sayabilir. `gateway.nodes.commands.deny: ["mcp.tools.call.v1"]` gibi
tam komut retleri de yürütmeyi engeller.

### Node tarafından barındırılan Skills

Becerileri düğüm makinesinin etkin OpenClaw becerileri dizinine yükleyin;
varsayılan olarak `~/.openclaw/skills`. `OPENCLAW_HOME`, `OPENCLAW_STATE_DIR` ve
`OPENCLAW_CONFIG_PATH` bu etkin profili taşır. Beceriler için `OPENCLAW_STATE_DIR`
önceliklidir; aksi takdirde `skills/`, `openclaw config file` tarafından
yazdırılan yolun yanındadır. Başsız düğüm ana makinesi, bağlandıktan sonra geçerli
`SKILL.md` dosyalarını yayımlar ve Gateway bunları yalnızca söz konusu
düğüm bağlı kaldığı sürece aracı beceri anlık görüntülerine ekler. Soyut düğüm
konumlandırıcının başka bir protokol alanı eklemeden tek bir girişle eşleşmesi
için her beceri dizininin adı, `name` frontmatter alanıyla eşleşmelidir.

İlk düğüm-rol eşleştirmesi, beceri yayımlamayı onaylar. Becerilerin eklenmesi,
kaldırılması veya değiştirilmesi için başka bir eşleştirme ya da Gateway
yapılandırma değişikliği gerekmez. Düğüm beceri dosyalarını değiştirdikten sonra
`openclaw node run` veya `openclaw node restart` öğesini yeniden başlatın; düğüm ana
makinesi beceriler dizinini izlemez.

Düğümde barındırılan beceri girişleri, düğümlerini tanımlar ve yürütme
konumlarını taşır. Beceri dosyaları, başvurulan göreli yollar ve ikili dosyalar
o düğümde kalır. Aracı, duyurulan `node://.../SKILL.md` konumunu normal
`read` aracıyla okur. `file_fetch`, düğüm beceri
konumlandırıcılarını değil, operatör tarafından onaylanmış mutlak düğüm yollarını
kabul eder; normal okuma aracı olmayan çalışma zamanları bunun yerine duyurulan
`node://.../skills/<name>` dizinini `workdir` olarak kullanarak
`exec host=node node=<node-id>` üzerinden `cat SKILL.md` çalıştırabilir. Başvurulan
dosyalar ve ikili dosyalar aynı yürütme hedefini ve çalışma dizinini kullanır.
Düğüm ana makinesi bu konumlandırıcıyı etkin OpenClaw durum dizinine göre
çözümler; dolayısıyla göreli yollar Gateway makinesinde değil, düğümde çözümlenir.
Yayımlayan düğümde `system.run` onaylanmış olmalı ve aracının yürütme
ilkesi `host=node` öğesine izin vermelidir; aksi takdirde beceri, söz
konusu aracının anlık görüntüsüne dahil edilmez.

Yayımlamayı durdurmak için düğümde `nodeHost.skills.enabled: false` ayarını yapın. Gateway
operatörleri, `gateway.nodes.allowSkills: false` ile eşleştirilmiş tüm düğümlerdeki becerileri
yok sayabilir.

### Başsız kimlik durumu

Başsız düğüm, paylaşılan SQLite'ta üç ayrı durum kaydı tutar:

- `~/.openclaw/state/openclaw.sqlite` (`node_host_config`): istemci örneği kimliği, görünen ad ve Gateway bağlantı meta verileri.
- `~/.openclaw/state/openclaw.sqlite` (`device_identities`, anahtar `primary`): imzalı cihaz anahtar çifti ve bundan türetilen kriptografik cihaz kimliği.
- `~/.openclaw/state/openclaw.sqlite` (`device_auth_tokens`): kriptografik cihaz kimliğine ve role göre anahtarlanmış eşleştirilmiş cihaz kimlik doğrulama belirteçleri.

İmzalı bir düğüm için Gateway, eşleştirme ve düğüm yönlendirmesinde kriptografik
cihaz kimliğini kullanır. İstemci örneği kimliği yalnızca bağlantı meta
verisidir. Bu nedenle `--node-id` öğesinin değiştirilmesi veya kullanımdan
kaldırılmış bir `node.json` öğesinin taşınması eşleştirmeyi sıfırlamaz.
Desteklenen iptal edip yeniden eşleştirme akışı ve yükseltme notları için
[Kimlik ve eşleştirme durumu](/tr/cli/node#identity-and-pairing-state) bölümüne bakın.

Kullanımdan kaldırılmış `identity/device.json` ve `identity/device-auth.json` dosyaları,
Doctor tarafından yönetilen taşıma girdileridir. Düğüm ana makinesini durdurun
ve `openclaw doctor --fix` komutunu çalıştırın; Doctor eski dosyaları kaldırmadan önce
satırlarını SQLite'a aktarır ve doğrular.

### Komutları izin verilenler listesine ekleme

Yürütme onayları **her düğüm ana makinesi için ayrıdır**. Gateway üzerinden izin
verilenler listesi girişleri ekleyin:

```bash
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/uname"
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/sw_vers"
```

Onaylar düğüm ana makinesinde `~/.openclaw/exec-approvals.json` konumunda bulunur.

### Yürütmeyi düğüme yönlendirme

Varsayılanları yapılandırın (Gateway yapılandırması):

```bash
openclaw config set tools.exec.host node
openclaw config set tools.exec.mode allowlist
openclaw config set tools.exec.node "<id-or-name>"
```

Veya oturum başına:

```text
/exec host=node security=allowlist node=<id-or-name>
```

Ayarlandıktan sonra, `host=node` içeren her `exec` çağrısı
düğüm ana makinesinde çalışır (düğümün izin verilenler listesine/onaylarına
tabidir).

`host=auto` düğümü kendi başına örtük olarak seçmez; ancak
`auto` üzerinden çağrı başına açık bir `host=node` isteğine
izin verilir. Düğüm yürütmesinin oturum için varsayılan olmasını istiyorsanız
`tools.exec.host=node` veya `/exec host=node ...` ayarını açıkça yapın.

İlgili:

- [Düğüm ana makinesi CLI'si](/tr/cli/node)
- [Yürütme aracı](/tr/tools/exec)
- [Yürütme onayları](/tr/tools/exec-approvals)

### Yerel model çıkarımı

Bir masaüstü veya sunucu düğümü, o düğümde çalışan bir Ollama sunucusundaki sohbet
özellikli modelleri kullanıma sunabilir. Aracılar, yüklü modelleri keşfetmek ve
sınırlandırılmış bir istemi uzaktan çalıştırmak için Ollama Plugin'inin
`node_inference` aracını kullanır; Gateway'in Ollama'ya doğrudan ağ erişimine
ihtiyacı yoktur. Kurulum, model filtreleme ve doğrudan doğrulama komutları için
[Ollama düğüm-yerel çıkarımı](/tr/providers/ollama#node-local-inference) bölümüne bakın.

### Codex oturumları ve dökümleri

Resmî `codex` Plugin'i, başsız bir düğüm ana makinesindeki veya yerel
macOS düğümündeki arşivlenmemiş Codex oturumlarını kullanıma sunabilir. Katalog
kaydı artık `supervision.enabled` öğesine bağlı değildir; bu seçenek, aracıya yönelik
denetim araçlarını kontrol eder. Sağlayıcıyı veya donanımı devre dışı bırakmadan
operatör kataloğunu ve eşleştirilmiş düğüm katalog komutlarını devre dışı bırakmak
için Codex Plugin yapılandırmasında `sessionCatalog.enabled: false` ayarını yapın.
Plugin her iki bilgisayarda da etkin olmalıdır ve düğüm ayarı yerel onay olmaya
devam eder: yalnızca Gateway'in etkinleştirilmesi, başka bir bilgisayarın Codex
durumunun okunmasına izin vermez.

Düğüm, sürümlendirilmiş salt okunur `codex.appServer.threads.list.v1` ve
`codex.appServer.thread.turns.list.v1` komutlarını duyurur. Codex CLI'ın kullanılabildiği yerel bir
düğüm ana makinesi ayrıca `codex.terminal.resume.v1` komutunu duyurur. Bu komutlar ilk
kez göründüğünde düğüm eşleştirme yükseltmesini onaylayın. Gateway bunları normal
Plugin düğüm ilkesi üzerinden çağırır ve hataları ana makineye göre yalıtır.

Eşleştirilmiş düğüm satırları normal oturumlar kenar çubuğunda bir **Codex**
grubu olarak görünür. Her ana makinede satırlar varsayılan olarak proje klasörüne
göre gruplanır; `.claude/worktrees/<name>` altındaki bir çalışma dizini kaynak deposuyla
birleştirilir ve proje grupları diğer kenar çubuğu bölümleri gibi daraltılır.
Proje gruplarını düzleştirmek veya geri yüklemek için katalog başlığındaki klasör
simgesini kullanın. Aynı gruplandırma Claude oturumları kataloğuna da uygulanır.
Varsayılan olarak bir satır seçildiğinde normal Sohbet bölmesi açılır ve kalıcı
dökümü, tam öğe projeksiyonuna sahip sınırlandırılmış, imleçle sayfalandırılan
`thread/turns/list` çağrıları üzerinden okunur. Oturumun sahibi olan bilgisayardaki
operatör terminalinde `codex resume <thread-id>` öğesini başlatmak için satır menüsünü,
görüntüleyici başlığını veya **Codex/Claude oturumlarını şurada aç** tercihini
kullanın. Eşleştirilmiş düğüm terminal yolu, Codex Plugin'inin yönettiği izin
verilenler listesine alınmış bir PTY aktarıcısıdır; rastgele düğüm komutu yürütme
mekanizması değildir.

Aktarıcı, OpenClaw donanımının tam devam ettirme ve arşiv sahipliği sözleşmelerini
sağlamaz. Bu nedenle uzak satırlar için **Devam et** ve **Arşivle** kullanılamaz.
Gateway bilgisayarında depolanmış ve boşta olan satırlar, ayrı ve modele kilitli
bir Sohbet dalı başlatabilir. Her ikisi de yalnızca operatör başka hiçbir Codex
istemcisinin bunu kullanmadığını onayladıktan sonra arşivlenebilir; depolanmış bir
satırın canlı etkinliği bilinmez. Etkin satırlar dallanamaz veya arşivlenemez.

Kurulum, sayfalama, yerel devam ettirme ve meta veri güvenlik sınırı için
[Codex oturumlarını denetleme](/plugins/codex-supervision) bölümüne bakın.

### Claude oturumları ve dökümleri

Paketle birlikte gelen `anthropic` Plugin'i, varsayılan olarak Gateway'deki
ve eşleştirilmiş düğümlerdeki arşivlenmemiş Claude CLI ve Claude Desktop
oturumlarını keşfeder. Anthropic modellerini veya Claude CLI arka ucunu devre
dışı bırakmadan operatör kataloğunu ve eşleştirilmiş düğüm katalog komutlarını
devre dışı bırakmak için `plugins.entries.anthropic.config.sessionCatalog.enabled: false` ayarını yapın.
Uzak bir macOS uygulama düğümü, Anthropic Plugin etkinleştirildiğinde ve
`~/.claude/projects/` mevcut olduğunda `anthropic.claude.sessions.list.v1` ve
`anthropic.claude.sessions.read.v1` komutlarını duyurur. Bu komutlar ilk kez göründüğünde düğüm
eşleştirme yükseltmesini onaylayın.

Claude CLI'ın kullanılabildiği yerel bir düğüm ana makinesi ayrıca
`anthropic.claude.terminal.resume.v1` komutunu duyurur. Uygun CLI ve Desktop satırları, sahibi olan
ana makinedeki operatör terminalinde `claude --resume <session-id>` öğesini açabilir.
Bu, yerel oturumun devralınmasıdır; OpenClaw benimsemesinin aksine, önce Claude
oturumunu çatallamaz.

Katalog, geçerli Claude CLI proje dizini kayıtlarını, dizine eklenmemiş JSONL
dökümleri için sınırlandırılmış bir meta veri geri dönüşüyle birleştirir. Bu geri
dönüş, eş zamanlı ve yan zincir olmayan etkileşimli (`cli`) oturumları
ve başsız Agent SDK CLI (`sdk-cli`) oturumlarını tanır. Claude Desktop'ın
yerel meta verileri, Desktop başlıklarını ve arşiv durumunu sağlar. Her iki kaynak
da aynı Claude Code oturum kimliğine başvurduğunda Desktop meta verileri
önceliklidir; CLI'da arşiv işareti bulunmadığı için yalnızca CLI'a ait dökümler
görünür kalır. Döküm okumaları, opak bayt uzaklığı imleçlerini ve sınırlandırılmış
geriye doğru dosya okumalarını kullanır; böylece büyük bir oturum seçildiğinde
veya eski bir sayfa yüklendiğinde JSONL geçmişinin tamamı tek bir Gateway
yanıtına okunmaz.

Listeleme ve okuma komutları salt okunurdur. Katalog meta verilerini ve döküm
içeriğini yalnızca `operator.write` özelliğine sahip kimliği doğrulanmış bir
operatör bağlantısına genel `sessions.catalog.list` ve `sessions.catalog.read` yöntemleri
üzerinden sunarlar. Gateway'e yerel bir Claude CLI satırı normal Sohbet
düzenleyicisinden benimsenebilir: OpenClaw sınırlandırılmış görünür geçmişi içe
aktarır, ilk turda `--fork-session` ile devam eder ve kaynak dökümü olduğu gibi
bırakır.

Başsız bir düğüm ana makinesi aynı devam ettirme akışını etkinleştirebilir:

```json5
{
  nodeHost: {
    agentRuns: {
      claude: { enabled: true },
    },
  },
}
```

Düğüm yalnızca bu düğüme yerel ayar etkinleştirildiğinde ve
`claude` yürütülebilir dosyası o düğümde çözümlendiğinde
`agent.cli.claude.run.v1` komutunu duyurur. Gateway bunu uzaktan etkinleştiremez. Komut
ayrıca düğümün mevcut yürütme onayı ilkesinden geçer. Üç Claude komutunun tümü
duyurulduğunda ve Gateway'in düğüm komutu ilkesi tarafından izin verildiğinde,
o düğümdeki bir Claude CLI satırı devam ettirilebilir hâle gelir: OpenClaw
sınırlandırılmış geçmişi içe aktarır, benimsenen oturumu düğüme ve katalogda
bildirilen çalışma dizinine bağlar ve her tek seferlik `claude -p` turunu
orada çalıştırır. İlk tur yine `--fork-session` kullanarak kaynak dökümü korur.

Düğüme yerleştirilen turlar, düğümün Claude varsayılanlarını kullanır. v1'de
Gateway geri döngü MCP yapılandırmasını veya Gateway becerileri Plugin'ini almaz,
bir Gateway dökümünden yeniden tohumlanamaz ve eklerle görselleri reddeder.
Claude Desktop satırları ve çalıştırma komutunu duyurmayan düğümler yalnızca
görüntülenebilir. macOS uygulama düğümü henüz bu komutu duyurmadığından satırları
yalnızca görüntülenebilir olarak kalır.

Control UI davranışı ve depolama kaynakları için
[Anthropic: Bilgisayarlar arası Claude oturumları](/tr/providers/anthropic#claude-sessions-across-computers)
bölümüne bakın.

### OpenCode ve Pi oturumları

Paketle birlikte gelen OpenCode ve ACPX Plugin'leri de Gateway'deki ve
eşleştirilmiş düğümlerdeki salt okunur yerel oturum kataloglarını keşfeder.
Bir düğüm, `opencode` CLI yüklendiğinde `opencode.sessions.list.v1` /
`opencode.sessions.read.v1`; Pi'ın oturum dizini mevcut olduğunda ise
`acpx.pi.sessions.list.v1` / `acpx.pi.sessions.read.v1` komutlarını duyurur. Yeni komutlar ilk
kez göründüğünde düğüm eşleştirme yükseltmesini onaylayın. Eşleşen CLI da
kullanılabilir olduğunda düğüm `opencode.terminal.resume.v1` veya `acpx.pi.terminal.resume.v1`
komutunu ekler; mevcut satır menüsü ve görüntüleyici başlığı, seçilen oturumu
`opencode --session <id>` veya `pi --session <id>` ile sahibi olan terminalde yeniden
açabilir.

OpenCode, resmî CLI JSON/dışa aktarma yüzeyi üzerinden okur. Pi, proje ve genel
`settings.json` oturum dizinlerinin yanı sıra `PI_CODING_AGENT_DIR` ve
`PI_CODING_AGENT_SESSION_DIR` geçersiz kılmalarını da içeren belgelenmiş JSONL oturum
deposunu okur. Her iki katalog da varsayılan olarak etkindir; bunları Web UI'da
**Config > Plugins** altında kapatın.

Terminalde devam ettirme, depolanmış oturum çalışma dizinini ve Codex ile Claude
ile aynı, izin verilenler listesine alınmış çift yönlü PTY aktarıcısını kullanır.
Rastgele düğüm komutu yürütmeyi kullanıma sunmaz.

### Terminal dosya yüklemeleri

Control UI, dosyaları açık bir eşleştirilmiş node terminaline sürükleyebilir. Yerel node ana bilgisayarı, yalnızca yöneticilere açık `terminal.upload` komutunu duyurur; ilk göründüğünde eşleştirme yükseltmesini onaylayın. Her dosya 16 MiB ile sınırlıdır, söz konusu node üzerindeki özel bir geçici dizinde hazırlanır ve çalıştırılmadan, kabuk için tırnaklanmış bir yol olarak terminale döndürülür.

Yol ekleme; PowerShell, `cmd.exe` ve tanınan POSIX kabuklarını (`sh`, Bash, Dash, Ash, Ksh, Zsh ve Fish), Windows üzerinde Git Bash dahil olmak üzere destekler. Tırnaklama kuralları güvenli biçimde çıkarılamadığından diğer kabuk geçersiz kılmaları reddedilir; yerel WSL yolları için node ana bilgisayarını WSL içinde çalıştırın. `%` veya `!` içeren `cmd.exe` yolları da reddedilir; çünkü bu kabuk, söz konusu karakterleri çift tırnak içinde bile genişletir.

## Komutları çağırma

Düşük düzeyli (ham RPC):

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command canvas.eval --params '{"javaScript":"location.href"}'
```

`nodes invoke`, `system.run` ve `system.run.prepare` komutlarını engeller; bu komutlar yalnızca `host=node` ile `exec` aracı üzerinden çalışır (yukarıya bakın). Yaygın "aracıya bir MEDIA eki sağla" iş akışları için daha yüksek düzeyli yardımcılar bulunur (Canvas, kamera, ekran, konum; aşağıya bakın).

Uzun süre çalışan akışlı node komutları, eklemeli `node.invoke.progress`
olaylarını kullanır. Her olay çağrı kimliğini, sıfır tabanlı bir sıra numarasını ve
sınırlı bir UTF-8 metin parçasını taşır; Gateway, parçaları çağırana iletmeden önce
sıralar. Mevcut `node.invoke.result`, tek terminal
yanıtı olarak kalır. Akış çağıranları, ilk ilerleme olayıyla başlayan ve sonraki
ilerlemelerden sonra sıfırlanan bir hareketsizlik son tarihi ayarlayabilir; bu sırada
çağrının onay ve yürütme aşamalarındaki ayrı kesin zaman aşımı korunur. Sonuç, kesin
zaman aşımı, hareketsizlik zaman aşımı ve node bağlantısının kesilmesi, bekleyen tüm akış
durumunu temizler. Çağıranın iptali `node.invoke.cancel` olayını yayar; ardından node ana bilgisayarı
eşleşen işlem ağacını sonlandırır. Mevcut istek/yanıt komutları değişmemiştir.

## Komut politikası

Node komutlarının çağrılabilmesi için iki geçidi aşması gerekir:

1. Node, komutu kimliği doğrulanmış bağlantı meta verilerinde (`connect.commands`) bildirmelidir.
2. Gateway'in platformdan ve onaydan türetilen izin listesi, bildirilen komutu içermelidir.

Platforma göre varsayılan izin listeleri (Plugin varsayılanlarından ve `commands.allow`/`commands.deny` geçersiz kılmalarından önce):

| Platform | Varsayılan olarak izin verilen komutlar                                                                                                                                                                                                                                                                                           |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| iOS      | `camera.list`, `location.get`, `device.info`, `device.status`, `contacts.search`, `calendar.events`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer`, `system.notify`                                                                                                                        |
| watchOS  | `device.info`, `device.status`, `system.notify`                                                                                                                                                                                                                                                                       |
| Android  | `camera.list`, `location.get`, `notifications.list`, `notifications.actions`, `system.notify`, `device.info`, `device.status`, `device.permissions`, `device.health`, `device.apps`, `contacts.search`, `calendar.events`, `callLog.search`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer` |
| macOS    | `camera.list`, `location.get`, `device.info`, `device.status`, `device.apps`, `contacts.search`, `calendar.events`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer`, `system.notify`                                                                                                         |
| Windows  | `camera.list`, `location.get`, `device.info`, `device.status`, `system.notify`                                                                                                                                                                                                                                        |
| Linux    | `system.notify` (node ana bilgisayarı komutları, örneğin `system.run`, onaya tabidir; aşağıya bakın)                                                                                                                                                                                                                                  |

Bu satırlar, her node uygulamasının uyguladığı komutları değil, Gateway politikasının üst sınırını açıklar. Bir komut yalnızca bağlı node da bunu bildiriyorsa kullanılabilir. Özellikle mevcut macOS uygulaması, macOS politika satırında listelenen cihaz ve kişisel veri ailelerini bildirmez.

`canvas.*` komutları (`canvas.present`, `canvas.hide`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot`, `canvas.a2ui.*`) iOS, Android, macOS, Windows, Linux ve bilinmeyen platformlarda bir Plugin varsayılanıdır. Linux node'ları bunları yalnızca masaüstü uygulamasının yerel Canvas yuvası mevcut olduğunda bildirir. Tüm Canvas komutları iOS'ta ön planla sınırlıdır.

`talk.ptt.start`, `talk.ptt.stop`, `talk.ptt.cancel` ve `talk.ptt.once`; platform etiketinden bağımsız olarak, `talk` yeteneğini duyuran veya `talk.*` komutlarını bildiren herhangi bir node için varsayılan olarak izinlidir.

Masaüstü ana bilgisayar komutları (macOS/Windows/Linux üzerinde `system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `mcp.tools.call.v1` ve `screen.snapshot`) yukarıdaki statik platform varsayılanları tablosunun parçası değildir. Operatör bunları bildiren bir eşleştirme isteğini onayladığında kullanılabilir hâle gelirler; bundan sonra node'un onaylanmış komut kümesi, yeniden bağlantıda bunları korur.

Tehlikeli veya yoğun gizlilik etkisi olan komutlar, bir node bunları bildirse bile `gateway.nodes.commands.allow` ile açıkça etkinleştirilmelidir: `camera.snap`, `camera.clip`, `screen.record`, `computer.act`, `contacts.add`, `calendar.add`, `reminders.add`, `health.summary`, `sms.send`, `sms.search`. `gateway.nodes.commands.deny`, varsayılanlara ve ek izin listesi girdilerine her zaman üstün gelir. iPhone izin geçidi için [HealthKit özetleri](/tr/platforms/ios-healthkit), masaüstü girdisi etrafındaki ek yetenek, araç politikası, etkinleştirme ve platform yürütücüsü geçitleri için [Bilgisayar kullanımı](/tr/nodes/computer-use) bölümüne bakın.

Plugin'in sahip olduğu node komutları, bir Gateway node çağırma politikası ekleyebilir. Bu politika izin listesi denetiminden sonra ve node'a iletilmeden önce çalışır; böylece ham `node.invoke`, CLI yardımcıları ve özel aracı araçları aynı Plugin izin sınırını paylaşır. Tehlikeli Plugin node komutları yine de açık `gateway.nodes.commands.allow` etkinleştirmesi gerektirir.

Bir node bildirilen komut listesini değiştirdikten sonra, Gateway'in güncellenmiş komut anlık görüntüsünü saklaması için eski cihaz eşleştirmesini reddedin ve yeni isteği onaylayın.

## Yapılandırma (`openclaw.json`)

Node ile ilgili ayarlar `gateway.nodes` ve `tools.exec` altında bulunur:

```json5
{
  gateway: {
    nodes: {
      // Güvenilen ağlardan ilk node eşleştirmesini otomatik olarak onaylayın (CIDR listesi).
      // Ayarlanmadığında devre dışıdır. Yalnızca istenen kapsamları olmayan ilk role:node
      // isteklerine uygulanır; yükseltmeleri otomatik olarak onaylamaz.
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
        // SSH ile doğrulanmış otomatik onay (varsayılan: etkin). SSH üzerinden geri okunan
        // tam cihaz anahtarı eşleşmesinde ilk node eşleştirmesini onaylar.
        sshVerify: true,
      },
      // Eşleştirilmiş node'ların yayımladığı aracıya görünür Plugin araçlarına güvenin (varsayılan: true).
      pluginTools: {
        enabled: true,
      },
      // Tehlikeli/yoğun gizlilik etkisi olan node komutlarını etkinleştirin (camera.snap vb.).
      commands: {
        allow: ["camera.snap", "screen.record"],
        // Varsayılanlar veya commands.allow bunları içerse bile tam komut adlarını engelleyin.
        deny: ["camera.clip"],
      },
    },
  },
  tools: {
    exec: {
      // Varsayılan exec ana bilgisayarı: "node", tüm exec çağrılarını eşleştirilmiş bir node'a yönlendirir.
      host: "node",
      // Node exec için güvenlik modu: yalnızca onaylanmış/izin listesine alınmış komutlara izin verin.
      security: "allowlist",
      // Exec'i belirli bir node'a sabitleyin (kimlik veya ad). Herhangi bir node'a izin vermek için atlayın.
      node: "build-node",
    },
  },
}
```

Tam node komut adlarını kullanın. `commands.deny`, bir platform varsayılanı veya `commands.allow` girdisi normalde izin verse bile komutu kaldırır. Eşleştirilmiş node'lar varsayılan olarak aracıya görünür Plugin aracı tanımlayıcıları yayımlayabilir, ancak her tanımlayıcının komutu yine de node'un onaylanmış komut yüzeyinde bulunmalıdır. Bu tür tüm tanımlayıcıları yok saymak için `gateway.nodes.pluginTools.enabled: false` ayarını kullanın. Gateway node eşleştirmesi ve komut politikası alanlarının ayrıntıları için [Gateway yapılandırma başvurusu](/tr/gateway/configuration-reference#gateway) bölümüne bakın.

Aracı başına exec node geçersiz kılması:

```json5
{
  agents: {
    list: [
      {
        id: "main",
        tools: { exec: { node: "build-node" } },
      },
    ],
  },
}
```

## Ekran görüntüleri (Canvas anlık görüntüleri)

Node Canvas'ı (WebView) gösteriyorsa `canvas.snapshot`, `{ format, base64 }` döndürür.

CLI yardımcısı (geçici bir dosyaya yazar ve kaydedilen yolu yazdırır):

```bash
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format png
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format jpg --max-width 1200 --quality 0.9
```

### Canvas denetimleri

```bash
openclaw nodes canvas present --node <idOrNameOrIp> --target https://example.com
openclaw nodes canvas hide --node <idOrNameOrIp>
openclaw nodes canvas navigate https://example.com --node <idOrNameOrIp>
openclaw nodes canvas eval --node <idOrNameOrIp> --js "document.title"
```

Notlar:

- `canvas present`, yerel yolları destekleyen node'larda URL'leri veya yerel dosya yollarını (`--target`) ve ayrıca konumlandırma için isteğe bağlı `--x/--y/--width/--height` değerini kabul eder. Linux Canvas, HTTP(S) URL'lerini veya paketlenmiş A2UI işleyicisini kabul eder.
- `canvas eval`, satır içi JS (`--js`) veya konumsal bir argüman kabul eder.

### A2UI (Canvas)

```bash
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --text "Hello"
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --jsonl ./payload.jsonl
openclaw nodes canvas a2ui reset --node <idOrNameOrIp>
```

Notlar:

- Mobil ve Linux masaüstü node'ları, eylem özellikli işleme için uygulamaya ait paketlenmiş bir A2UI sayfası kullanır.
- Yalnızca A2UI v0.8 JSONL desteklenir (v0.9/createSurface reddedilir).
- iOS ve Android, uzak Gateway Canvas sayfalarını işler; ancak A2UI düğme eylemleri yalnızca uygulamaya ait paketlenmiş A2UI sayfasından gönderilir. Gateway tarafından barındırılan HTTP/HTTPS A2UI sayfaları, bu mobil istemcilerde yalnızca görüntülenir.
- macOS, uygulamanın seçtiği tam yetenek kapsamlı Gateway A2UI sayfasından eylemler gönderebilir. Diğer HTTP/HTTPS sayfaları yalnızca görüntülenmeye devam eder.
- Linux, eylemleri yalnızca paketlenmiş A2UI sayfasından gönderir. Diğer HTTP/HTTPS sayfaları yalnızca görüntülenmeye devam eder ve masaüstü uygulaması olmayan başsız bir Linux node'u Canvas'ı duyurmaz.

## Fotoğraflar + videolar (node kamerası)

Fotoğraflar (`jpg`):

```bash
openclaw nodes camera list --node <idOrNameOrIp>
openclaw nodes camera snap --node <idOrNameOrIp>            # varsayılan: her iki yön (2 MEDIA satırı)
openclaw nodes camera snap --node <idOrNameOrIp> --facing front
openclaw nodes camera snap --node <idOrNameOrIp> --device-id <id> --max-width 1200 --quality 0.9 --delay-ms 2000
```

Video klipleri (`mp4`):

```bash
openclaw nodes camera clip --node <idOrNameOrIp> --duration 10s
openclaw nodes camera clip --node <idOrNameOrIp> --duration 3000 --no-audio
```

Notlar:

- Node, `canvas.*` ve `camera.*` için **ön planda** olmalıdır (arka plan çağrıları `NODE_BACKGROUND_UNAVAILABLE` döndürür).
- Node'lar, base64 yükünü yönetilebilir tutmak için klip süresini sınırlar (platforma özgü kesin sınırlar için [Kamera yakalama](/tr/nodes/camera) bölümüne bakın). `nodes` agent aracı ayrıca çağrıyı iletmeden önce istenen `durationMs` değerini 300000 (5 dakika) ile sınırlar; daha sıkı sınırı Node'un kendisi uygular.
- Android, mümkün olduğunda `CAMERA`/`RECORD_AUDIO` izinlerini ister; reddedilen izinler `*_PERMISSION_REQUIRED` hatasıyla sonuçlanır.

## Ekran kayıtları (Node'lar)

Desteklenen Node'lar `screen.record` (mp4) sunar. Örnek:

```bash
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10 --no-audio
```

Notlar:

- `screen.record` kullanılabilirliği Node platformuna bağlıdır.
- `nodes` agent aracı, istenen `durationMs` değerini 300000 (5 dakika) ile sınırlar; Node, döndürülen yükü sınırlamak için daha sıkı bir sınır uygulayabilir.
- `--no-audio`, desteklenen platformlarda mikrofon yakalamayı devre dışı bırakır.
- Birden fazla ekran kullanılabilir olduğunda ekran seçmek için `--screen <index>` kullanın (0 = birincil).

## Konum (Node'lar)

Ayarlar'da Konum etkinleştirildiğinde Node'lar `location.get` sunar.

CLI yardımcısı:

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

Notlar:

- Konum **varsayılan olarak kapalıdır**.
- "Always" sistem izni gerektirir; arka planda alma işlemi mümkün olan en iyi şekilde gerçekleştirilir.
- Yanıt; enlem/boylamı, doğruluğu (metre) ve zaman damgasını içerir.
- Parametrelerin/yanıtın tam yapısı ve hata kodları: [Konum komutu](/tr/nodes/location-command).

## SMS (Android Node'ları)

Kullanıcı **SMS** izni verdiğinde ve cihaz telefon özelliğini desteklediğinde Android Node'ları `sms.send` ve `sms.search` sunabilir. Her iki komut da varsayılan olarak tehlikelidir: çağrılabilmeleri için Gateway operatörünün bunları ayrıca `gateway.nodes.commands.allow` listesine eklemesi gerekir (bkz. [Komut ilkesi](#command-policy)).

Salt okunur SMS araması için `openclaw.json` içinde açıkça etkinleştirin:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["sms.search"] },
    },
  },
}
```

Yalnızca Node'un mesaj da gönderebilmesi gerektiğinde `sms.send` değerini ayrıca ekleyin. Android izni ile Gateway komut yetkilendirmesi birbirinden bağımsızdır; telefon izninin verilmesi Gateway ilkesini düzenlemez.

Düşük düzeyli çağrı:

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command sms.send --params '{"to":"+15555550123","message":"OpenClaw'dan merhaba"}'
```

Notlar:

- `sms.search`, `READ_SMS` verilmeden önce bildirilebilir; böylece bir çağrı izin tanılaması döndürebilir. Mesajları okumak için yine de bu Android izni gerekir.
- Telefon özelliği olmayan, yalnızca Wi-Fi kullanan cihazlar `sms.send` bildirmez.
- `requires explicit gateway.nodes.commands.allow opt-in` hatası, telefonun komutu bildirdiği ancak Gateway operatörünün komutu yetkilendirmediği anlamına gelir.

## Cihaz ve kişisel veri komutları

iOS ve Android Node'ları varsayılan olarak birkaç salt okunur veri komutu bildirir ([Komut ilkesi](#command-policy) tablosuna bakın); Android ayrıca kendi uygulama içi ayarlarıyla denetlenen daha geniş bir komut ailesi sunar. macOS veya başsız Mac TypeScript Node ana bilgisayarı, `--share-installed-apps` ile yüklü uygulama paylaşımı operatör tarafından etkinleştirildikten sonra yalnızca `device.apps` bildirir.

Kullanılabilir aileler:

- `device.status`, `device.info` — iOS, Android, Windows.
- `device.permissions`, `device.health` — yalnızca Android.
- `device.apps` — Android, macOS ve başsız Mac Node'ları. Android, Ayarlar'da Yüklü Uygulamalar paylaşımının etkinleştirilmesini gerektirir ve varsayılan olarak başlatıcıda görünen uygulamaları döndürür. TypeScript Node ana bilgisayarları paylaşımı varsayılan olarak kapalı tutar ve `query`, `limit` ve `includeSystem` değerlerini kabul eder; macOS sonuçları `label`, `bundleId`, `path` ve `system` içerir.
- `notifications.list`, `notifications.actions` — yalnızca Android.
- `photos.latest` — iOS, Android.
- `contacts.search` — iOS, Android (varsayılan olarak salt okunur); `contacts.add` tehlikelidir ve `gateway.nodes.commands.allow` gerektirir.
- `calendar.events` — iOS, Android (varsayılan olarak salt okunur); `calendar.add` tehlikelidir ve `gateway.nodes.commands.allow` gerektirir.
- `reminders.list` — iOS, Android (varsayılan olarak salt okunur); `reminders.add` tehlikelidir ve `gateway.nodes.commands.allow` gerektirir.
- `callLog.search` — yalnızca Android.
- `motion.activity`, `motion.pedometer` — iOS, Android; kullanılabilir sensörlerin yetenekleriyle sınırlandırılır.

Örnek çağrılar:

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command device.status --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command device.apps --params '{"limit":10}'
openclaw nodes invoke --node <idOrNameOrIp> --command notifications.list --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command photos.latest --params '{"limit":1}'
```

## Sistem komutları (Node ana bilgisayarı / Mac Node'u)

macOS Node'u `system.run`, `system.which`, `system.notify` ve `system.execApprovals.get/set` sunar. Başsız Node ana bilgisayarı `system.run.prepare`, `system.run`, `system.which` ve `system.execApprovals.get/set` sunar.

Örnekler:

```bash
openclaw nodes notify --node <idOrNameOrIp> --title "Ping" --body "Gateway hazır"
openclaw nodes invoke --node <idOrNameOrIp> --command system.which --params '{"bins":["git"]}'
```

Notlar:

- `system.run`, yük içinde standart çıktıyı/standart hatayı/çıkış kodunu döndürür.
- Kabuk yürütme artık `host=node` ile `exec` aracı üzerinden gerçekleştirilir; `nodes`, açık Node komutları için doğrudan RPC yüzeyi olarak kalır.
- `nodes invoke`, `system.run` veya `system.run.prepare` sunmaz; bunlar yalnızca exec yolunda kalır.
- Exec yolu, onaydan önce standart bir `systemRunPlan` hazırlar. Onay verildikten sonra Gateway, çağıran tarafından daha sonra düzenlenmiş komut/cwd/oturum alanlarını değil, depolanan bu planı iletir.
- `system.notify`, macOS uygulamasındaki bildirim izni durumuna uyar; `--priority <passive|active|timeSensitive>` ve `--delivery <system|overlay|auto>` destekler.
- Tanınmayan Node `platform` / `deviceFamily` meta verileri, `system.run` ve `system.which` öğelerini hariç tutan ihtiyatlı bir varsayılan izin listesi kullanır. Bilinmeyen bir platform için bu komutlara bilinçli olarak ihtiyaç duyuluyorsa bunları `gateway.nodes.commands.allow` aracılığıyla açıkça ekleyin.
- `system.run`; `--cwd`, `--env KEY=VAL`, `--command-timeout` ve `--needs-screen-recording` destekler.
- Kabuk sarmalayıcıları (`bash|sh|zsh ... -c/-lc`) için istek kapsamındaki `--env` değerleri açık bir izin listesine (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`) indirgenir.
- İzin listesi modundaki her zaman izin ver kararlarında, bilinen dağıtım sarmalayıcıları (`env`, `flock`, `nice`, `nohup`, `stdbuf`, `timeout`) sarmalayıcı yolları yerine iç yürütülebilir dosya yollarını kalıcı hâle getirir. Sarmalayıcı güvenli biçimde açılamıyorsa otomatik olarak hiçbir izin listesi girdisi kalıcı hâle getirilmez.
- İzin listesi modundaki Windows Node ana bilgisayarlarında, `cmd.exe /c` üzerinden gerçekleştirilen kabuk sarmalayıcı çalıştırmaları onay gerektirir (tek başına izin listesi girdisi sarmalayıcı biçimine otomatik olarak izin vermez).
- Node ana bilgisayarları, `--env` içindeki `PATH` geçersiz kılmalarını yok sayar ve bir komutu çalıştırmadan önce geniş ve bakımı sürdürülen bir yorumlayıcı/kabuk başlangıç değişkenleri kümesini (örneğin `NODE_OPTIONS`, `PYTHONPATH`, `BASH_ENV`, `DYLD_*`, `LD_*`) kaldırır. Ek PATH girdileri gerekiyorsa `PATH` değerini `--env` üzerinden geçirmek yerine Node ana bilgisayarı hizmet ortamını yapılandırın (veya araçları standart konumlara yükleyin).
- macOS Node modunda `system.run`, macOS uygulamasındaki exec onaylarıyla denetlenir (Settings → Exec approvals). Sor/izin listesi/tam seçenekleri başsız Node ana bilgisayarıyla aynı şekilde davranır; reddedilen istemler `SYSTEM_RUN_DENIED` döndürür.
- Başsız Node ana bilgisayarında `system.run`, exec onaylarıyla (`~/.openclaw/exec-approvals.json`) denetlenir; özellikle macOS için aşağıdaki [Başsız Node ana bilgisayarı](#headless-node-host-cross-platform) bölümünde exec ana bilgisayarı yönlendirme ortam değişkenlerine bakın.

## Exec Node bağlama

Birden fazla Node kullanılabilir olduğunda exec belirli bir Node'a bağlanabilir. Bu, `exec host=node` için varsayılan Node'u ayarlar (ve agent başına geçersiz kılınabilir).

Genel varsayılan:

```bash
openclaw config set tools.exec.node "node-id-or-name"
```

Agent başına geçersiz kılma:

```bash
openclaw config get agents.entries
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
```

Herhangi bir Node'a izin vermek için ayarı kaldırın:

```bash
openclaw config unset tools.exec.node
openclaw config unset 'agents.entries.main.tools.exec.node'
```

## İzinler eşlemesi

Node'lar, izin adına göre anahtarlanmış (ör. `screenRecording`, `accessibility`, `location`) ve boole değerleri (`true` = verildi) içeren bir `permissions` eşlemesini `node.list` / `node.describe` içine ekleyebilir.

## Başsız Node ana bilgisayarı (platformlar arası)

OpenClaw, Gateway WebSocket'e bağlanan ve `system.run` / `system.which` sunan bir **başsız Node ana bilgisayarı** (kullanıcı arayüzü olmadan) çalıştırabilir. Bu, Linux/Windows üzerinde veya bir sunucunun yanında asgari bir Node çalıştırmak için kullanışlıdır.

Başlatın:

```bash
openclaw node run --host <gateway-host> --port 18789
```

Notlar:

- Eşleştirme yine de gereklidir (Gateway bir cihaz eşleştirme istemi gösterir).
- İstemci örneği meta verileri, imzalı cihaz kimliği ve eşleştirme kimlik doğrulaması ayrı durum kayıtları kullanır; bkz. [Başsız kimlik durumu](#headless-identity-state).
- Exec onayları `~/.openclaw/exec-approvals.json` aracılığıyla yerel olarak uygulanır (bkz. [Exec onayları](/tr/tools/exec-approvals)).
- macOS'ta başsız Node ana bilgisayarı varsayılan olarak `system.run` öğesini yerel olarak yürütür. `system.run` öğesini yardımcı uygulamanın exec ana bilgisayarı üzerinden yönlendirmek için `OPENCLAW_NODE_EXEC_HOST=app` ayarlayın; uygulama ana bilgisayarını zorunlu kılmak ve kullanılamadığında güvenli biçimde başarısız olmak için `OPENCLAW_NODE_EXEC_FALLBACK=0` ekleyin.
- Gateway WS, TLS kullandığında `--tls` / `--tls-fingerprint` ekleyin.

## Mac Node modu

- macOS menü çubuğu uygulaması Gateway WS sunucusuna bir Node olarak bağlanır (böylece `openclaw nodes …` bu Mac üzerinde çalışır).
- Uzak modda uygulama, Gateway portu için bir SSH tüneli açar ve `localhost` adresine bağlanır.
