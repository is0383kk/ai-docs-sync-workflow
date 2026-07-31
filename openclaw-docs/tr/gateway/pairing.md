---
read_when:
    - macOS kullanıcı arayüzü olmadan Node eşleştirme onaylarını uygulama
    - Uzak düğümleri onaylamak için CLI akışları ekleme
    - Gateway protokolünü Node yönetimiyle genişletme
summary: 'Node yetenek onayları: cihaz eşleştirmesinden sonra Node''ların komutları nasıl erişime açtığı'
title: Node eşleştirme
x-i18n:
    generated_at: "2026-07-26T23:21:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 25e4016657379573ddb7e9027899afd8b97b16709da6e73ed44d4016b99e715a
    source_path: gateway/pairing.md
    workflow: 16
---

Node eşleştirmenin iki katmanı vardır; her ikisi de Gateway'in SQLite durum
veritabanındaki eşleştirilmiş cihaz kaydında saklanır:

- **Cihaz eşleştirme** (`node` rolü), `connect` el sıkışmasını denetler. Aşağıdaki
  [Güvenilir CIDR cihaz otomatik onayı](#trusted-cidr-device-auto-approval)
  ve [Kanal eşleştirme](/tr/channels/pairing) bölümlerine bakın.
- **Node yetenek onayı** (`node.pair.*`), bağlı bir Node'un hangi beyan edilmiş
  yetenekleri/komutları sunabileceğini denetler. Doğruluk kaynağı Gateway'dir;
  kullanıcı arayüzleri (macOS uygulaması, Control UI), bekleyen istekleri
  onaylayan veya reddeden ön yüzlerdir.

Önceki bağımsız Node eşleştirme deposu (Node başına bir belirteç içeren
`nodes/paired.json`; Ocak 2026'da bağlantı yolundan kaldırıldı) artık yoktur:
Gateway'ler kalan tüm satırları başlangıçta bir kez cihaz kayıtlarına aktarır
ve eski dosyaları `.migrated` son ekiyle arşivler. Eski TCP köprüsü
desteği kaldırılmıştır.

## Yetenek onayı nasıl çalışır?

1. Bir Node, Gateway WS'ye bağlanır (cihaz eşleştirme bu adımı denetler).
2. Gateway, beyan edilen yetenek/komut yüzeyini onaylanmış yüzeyle
   karşılaştırır; yeni veya genişletilmiş yüzeyler, cihaz kaydında bir
   **bekleyen istek** saklar ve `node.pair.requested` olayını yayınlar.
3. İsteği onaylar veya reddedersiniz (CLI ya da kullanıcı arayüzü).
4. Onay verilene kadar Node komutları filtrelenmiş olarak kalır; onay,
   normal komut politikasına tabi olmak üzere beyan edilen yüzeyi kullanıma açar.

Bekleyen isteklerin süresi, **Node'un son yeniden denemesinden 5 dakika sonra**
otomatik olarak dolar. Etkin biçimde yeniden bağlanan bir Node, her denemede
yeni bir istek (ve onay istemi) oluşturmak yerine tek bekleyen isteğini canlı
tutar.

## CLI iş akışı (ekransız kullanıma uygun)

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes status
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name "Living Room iPad"
```

`nodes status`, eşleştirilmiş/bağlı Node'ları ve yeteneklerini gösterir.

## API yüzeyi (Gateway protokolü)

Olaylar:

- `node.pair.requested` - yeni bir bekleyen istek oluşturulduğunda yayınlanır.
- `node.pair.resolved` - bir istek onaylandığında, reddedildiğinde veya
  süresi dolduğunda yayınlanır.

Yöntemler:

- `node.pair.list` - bekleyen ve eşleştirilmiş Node'ları listeler (`operator.pairing`).
- `node.pair.approve` - bekleyen bir isteği onaylar.
- `node.pair.reject` - bekleyen bir isteği reddeder.
- `node.pair.remove` - eşleştirilmiş bir Node'u kaldırır. Bu işlem, eşleştirilmiş cihaz
  deposunda cihazın `node` rolünü iptal eder, onaylanmış Node yüzeyini
  onunla birlikte kaldırır ve bu cihazın Node rolü oturumlarını geçersiz
  kılar/bağlantılarını keser. **Karma rollü** bir cihaz (örneğin aynı zamanda
  `operator` rolüne de sahip olan bir cihaz) satırını korur ve yalnızca
  `node` rolünü kaybeder; yalnızca Node olan bir cihazın satırı silinir.
  Yetkilendirme: `operator.pairing`, operatör olmayan Node satırlarını kaldırabilir;
  karma rollü bir cihazda **kendi** Node rolünü iptal eden cihaz belirteçli bir
  çağıranın ayrıca `operator.admin` yetkisine ihtiyacı vardır.
- `node.rename` - eşleştirilmiş bir Node'un operatöre gösterilen görüntüleme adını değiştirir.

2026.7 sürümünde kaldırıldı: `node.pair.request` ve `node.pair.verify`. Bekleyen
istekler, Node bağlantıları sırasında Gateway tarafından oluşturulur ve bu
yöntemlerin hizmet ettiği bağımsız Node başına belirteç artık mevcut değildir;
Node kimlik doğrulamasında cihaz eşleştirme belirteci kullanılır.

Notlar:

- Değişmemiş bir yüzeyle yeniden bağlanıldığında bekleyen istek yeniden
  kullanılır; yinelenen istekler, operatör görünürlüğü için saklanan Node
  meta verilerini ve izin listesine alınmış en son beyan edilen komut anlık
  görüntüsünü yeniler.
- Operatör kapsam düzeyleri ve onay sırasındaki denetimler
  [Operatör kapsamları](/tr/gateway/operator-scopes) bölümünde özetlenmiştir.
- `node.pair.approve`, ek onay kapsamlarını zorunlu kılmak için bekleyen isteğin
  beyan edilmiş komutlarını kullanır:
  - komutsuz istek: `operator.pairing`
  - normal komut isteği: `operator.pairing` + `operator.write`
  - `system.run`, `system.run.prepare`,
    `system.which`, `browser.proxy`, `fs.listDir` veya
    `system.execApprovals.get/set` içeren yönetici açısından hassas istek:
    `operator.pairing` + `operator.admin`

<Warning>
Node eşleştirme onayı, güvenilen yetenek yüzeyini kaydeder. Canlı Node komut yüzeyini Node bazında sabitlemez.

- Canlı Node komutları, Node'un bağlantı sırasında beyan ettiklerinden gelir ve
  Gateway'in genel Node komut politikası (`gateway.nodes.commands.allow` ve
  `gateway.nodes.commands.deny`) tarafından filtrelenir.
- Node bazındaki `system.run` izin verme ve sorma politikası,
  eşleştirme kaydında değil, `exec.approvals.node.*` içindeki Node'da bulunur.

</Warning>

## Node komutu denetimi (2026.3.31+)

<Warning>
**Uyumsuz değişiklik:** `2026.3.31` sürümünden itibaren Node eşleştirme onaylanana kadar Node komutları devre dışıdır. Beyan edilen Node komutlarını kullanıma açmak için artık yalnızca cihaz eşleştirme yeterli değildir.
</Warning>

Bir Node ilk kez bağlandığında eşleştirme otomatik olarak istenir.
Bu istek onaylanana kadar söz konusu Node'dan gelen tüm bekleyen Node komutları
filtrelenir ve yürütülmez. Eşleştirme onaylandıktan sonra Node'un beyan ettiği
komutlar, normal komut politikasına tabi olarak kullanılabilir hâle gelir.

Bunun anlamı şudur:

- Daha önce komutları kullanıma açmak için yalnızca cihaz eşleştirmeye dayanan
  Node'lar artık Node eşleştirmeyi de tamamlamalıdır.
- Eşleştirme onayından önce kuyruğa alınan komutlar ertelenmez, bırakılır.

## Node olayı güven sınırları (2026.3.31+)

<Warning>
**Uyumsuz değişiklik:** Node kaynaklı çalıştırmalar artık daraltılmış bir güvenilen yüzeyde kalır.
</Warning>

Node kaynaklı özetler ve ilgili oturum olayları, amaçlanan güvenilen yüzeyle
sınırlandırılmıştır. Daha önce daha geniş ana makine veya oturum aracı
erişimine dayanan bildirim güdümlü ya da Node tarafından tetiklenen akışların
ayarlanması gerekebilir. Bu sağlamlaştırma, Node olaylarının Node'un güven
sınırının izin verdiğinin ötesinde ana makine düzeyinde araç erişimine
yükselmesini engeller.

Kalıcı Node mevcudiyeti güncellemeleri aynı kimlik sınırını izler:
`node.presence.alive` olayı yalnızca kimliği doğrulanmış Node cihazı
oturumlarından kabul edilir ve eşleştirme meta verilerini yalnızca cihaz/Node
kimliği zaten eşleştirilmişse günceller. Kendi beyan ettiği bir
`client.id` değeri, son görülme durumunu yazmak için yeterli değildir.

## SSH ile doğrulanmış cihaz otomatik onayı (varsayılan)

Özel/CGNAT adresinden yapılan ilk `role: node` cihaz eşleştirmesi,
Gateway **SSH üzerinden makine sahipliğini kanıtlayabildiğinde** otomatik
olarak onaylanır: eşleştirme ana makinesine (`BatchMode`,
`StrictHostKeyChecking=yes`) geri bağlanır, orada `openclaw node identity --json` komutunu çalıştırır
ve yalnızca uzak cihaz kimliğiyle açık anahtar bekleyen istekle tam olarak
eşleştiğinde onay verir. Bunu güvenli kılan anahtar eşleşmesidir: yalnızca
erişilebilirlik hiçbir zaman onay için yeterli değildir; bu nedenle aynı NAT'ı
kullanan diğer kişiler, paylaşımlı bir ana makinedeki diğer kullanıcılar ve
LAN sahteciliği normal istem akışına geçer.

Varsayılan olarak etkindir. Tetiklenmesi için gerekenler:

- Gateway işleminin kullanıcısı (veya `sshVerify.user`), Node ana makinesine
  etkileşimsiz olarak SSH ile bağlanabilir (anahtarlar/aracı; Tailscale SSH de
  çalışır) ve ana makine anahtarı zaten güvenilirdir.
- `openclaw`, etkileşimsiz `sh -lc` için uzak
  `PATH` üzerinde çözümlenir.
- Bağlanan IP, doğrudan (proxy kullanılmayan ve geri döngü olmayan) bir özel,
  ULA, bağlantı-yerel veya CGNAT adresidir ya da ayarlanmışsa
  `sshVerify.cidrs` ile eşleşir.
- Güvenilir CIDR onayıyla aynı uygunluk alt sınırı geçerlidir: yalnızca yeni ve
  kapsamsız Node eşleştirme; yükseltmeler, tarayıcılar, Control UI ve WebChat
  her zaman istem gösterir.

Bir yoklama çalışırken Node istemcisine, manuel onay için duraklamak yerine
yeniden denemeyi sürdürmesi (`wait_then_retry`) bildirilir; yoklama başarısız
olursa sonraki deneme normal istem akışına geri döner. Başarısız hedeflere kısa
bir bekleme süresi uygulanır (anahtar uyuşmazlığından sonra 5 dakika).

Onaylanan cihazlar `approvedVia: "ssh-verified"` değerini kaydeder ve ilk beyan edilen
yetenek yüzeyleri aynı adımda onaylanır. Anahtar eşleşmesi, Node'un operatörün
hesabı altında, operatörün sahip olduğu bir makinede çalıştığını zaten
kanıtlar; bu, manuel yetenek onayının ileri sürdüğü iddiayla aynıdır. Daha
sonraki yüzey yükseltmeleri yine istem gösterir.

Güvenliği artırma veya devre dışı bırakma:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        // Tamamen devre dışı bırak:
        sshVerify: false,
        // ...veya yoklamanın kapsamını/ayarlarını belirle:
        // sshVerify: { user: "me", identity: "~/.ssh/probe", timeoutMs: 7000, cidrs: ["10.0.0.0/8"] },
      },
    },
  },
}
```

## Otomatik onay (macOS uygulaması)

macOS uygulaması, aşağıdaki durumlarda Node yeteneği isteklerini **sessizce
onaylamayı** deneyebilir:

- istek `silent` olarak işaretlenmişse (cihaz eşleştirme
  etkileşimsiz olarak onaylandığında Gateway ilk yetenek yüzeyini sessiz olarak
  işaretler) ve
- uygulama, aynı kullanıcıyı kullanarak Gateway ana makinesine SSH
  bağlantısını doğrulayabiliyorsa.

Sessiz onay başarısız olursa normal Approve/Reject istemine geri döner.

## Güvenilir CIDR cihaz otomatik onayı

`role: node` için WS cihaz eşleştirmesi varsayılan olarak manuel kalır.
Gateway'in ağ yoluna zaten güvendiği özel Node ağlarında operatörler açık CIDR
değerleri veya tam IP'lerle bunu etkinleştirebilir:

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

Güvenlik sınırı:

- `gateway.nodes.pairing.autoApproveCidrs` ayarlanmamışsa devre dışıdır.
- Genel bir LAN veya özel ağ otomatik onay modu yoktur; SSH ile doğrulanmış
  otomatik onay (yukarıda) kriptografik bir cihaz anahtarı eşleşmesi gerektirir,
  yalnızca ağ yakınlığı hiçbir zaman yeterli değildir.
- Yalnızca istenen kapsamı olmayan yeni bir `role: node` cihaz eşleştirme
  isteği uygundur.
- Operatör, tarayıcı, Control UI ve WebChat istemcileri manuel kalır.
- Rol, kapsam, meta veri ve açık anahtar yükseltmeleri manuel kalır.
- Aynı ana makinedeki geri döngü güvenilir proxy üst bilgisi yolları uygun
  değildir; çünkü bu yol yerel çağıranlar tarafından taklit edilebilir.

## Sessiz eşleştirmede yerine geçme temizliği

Etkileşimsiz onaylar kaynaklarını eşleştirilmiş cihaz satırına kaydeder:
aynı ana makine yerel politika onayları `silent`, güvenilir CIDR Node
onayları `trusted-cidr`, SSH ile doğrulanmış Node onayları ise
`ssh-verified` olarak kaydedilir. Durum dizini geçici olan istemciler
(geçici ana dizinler, kapsayıcılar, çalıştırma başına korumalı alanlar) her
çalıştırmada yeni bir cihaz anahtar çifti oluşturur ve her çalıştırma tamamen
yeni bir cihaz olarak sessizce yeniden eşleşir; temizlik yapılmazsa
eşleştirilmiş cihazlar listesi her çalıştırmada bir eski satır büyür.

Gateway **yerel** bir cihaz eşleştirmesini sessizce onayladığında, aynı istemci
kümesine ait olan (`clientId`, `clientMode` ve görüntüleme adı
eşleşen) ve o anda bağlı olmayan eski `silent` onaylı kayıtları
kullanımdan kaldırır. Yerel istemciler doğrudan Gateway ana makinesinde
çalıştığından küme anahtarı farklı bir makineyle eşleşemez. Kullanımdan
kaldırılan satırların belirteçleri hemen geçersiz olur; eşleşen tüm eski Node
eşleştirme girdileri temizlenir ve bir `node.pair.resolved` kaldırma olayı
yayınlanır.

Sınırlar:

- Yalnızca en son onayı aynı ana makinede yerel (`silent`) olarak verilen kayıtlar
  hem tetikleyici hem de hedef olarak uygundur. Güvenilir CIDR ve SSH ile doğrulanmış eşleştirmeler,
  görüntüleme meta verilerinin bir makine kimliği olmadığı ana makineler arasında gerçekleştiğinden
  hiçbir zaman otomatik olarak kaldırılmaz — bunlar için Control UI temizliğini veya
  `openclaw nodes remove` kullanın.
- Sahibi tarafından onaylanan ve QR/kurulum koduyla (önyükleme) yapılan eşleştirmeler hiçbir zaman
  otomatik olarak kaldırılmaz. Kaynak bilgisi mevcut olmadan önce onaylanan kayıtlar,
  aynı cihaz kimliği daha sonra sessizce yeniden onaylansa bile korunmaya devam eder.
- O anda bağlı olan cihazlar atlanır; böylece ayrı durum dizinlerine sahip
  eşzamanlı yerel oturumlar, etkin oldukları sürece token'larını korur. Son bir dakika içinde
  onaylanan kayıtlar da atlanır; böylece eşzamanlı eşleştirme el sıkışmaları, bağlantıları
  kaydedilmeden önce birbirini devre dışı bırakamaz.
- Etkilenen istemciler yapıları gereği yereldir; bu nedenle bir sonraki
  bağlantılarında sessizce yeniden eşleşirler.

## Meta veri yükseltmesini otomatik onaylama

Önceden eşleştirilmiş bir cihaz yalnızca hassas olmayan meta veri
değişiklikleriyle (örneğin görüntüleme adı veya istemci platformu ipuçları) yeniden
bağlandığında, OpenClaw bunu `metadata-upgrade` olarak değerlendirir. Sessiz otomatik onayın
kapsamı dardır: yalnızca işletim sistemi sürümü meta verisi değişikliklerinden sonra
aynı ana makinedeki yerel uygulamaların yeniden bağlanmaları da dahil olmak üzere,
yerel veya paylaşılan kimlik bilgilerine sahip olduğunu daha önce kanıtlamış güvenilir,
tarayıcı dışı yerel yeniden bağlantılara uygulanır. Tarayıcı/Control UI istemcileri ve
uzak istemciler açık yeniden onay akışını kullanmaya devam eder. Kapsam yükseltmeleri
(okumadan yazma/yönetici kapsamına) ve ortak anahtar değişiklikleri meta veri yükseltmesini
otomatik onaylama için **uygun değildir**; bunlar açık yeniden onay istekleri olarak kalır.

## QR eşleştirme yardımcıları

`/pair qr`, mobil ve tarayıcı istemcilerinin doğrudan tarayabilmesi için eşleştirme
yükünü yapılandırılmış medya olarak işler.

Bir cihazın silinmesi, o cihaz kimliği için bekleyen eski eşleştirme isteklerini de
temizler; böylece `nodes pending`, iptal işleminden sonra sahipsiz satırlar göstermez.

## Yerellik ve iletilen üstbilgiler

Gateway eşleştirmesi, bir bağlantıyı yalnızca hem ham soket hem de yukarı akış
proxy kanıtı aynı sonuca vardığında geri döngü olarak değerlendirir. Bir istek geri
döngü üzerinden gelir ancak `Forwarded`, herhangi bir `X-Forwarded-*` veya
`X-Real-IP` üstbilgi kanıtı taşırsa, iletilen üstbilgi kanıtı geri döngü yerelliği
iddiasını geçersiz kılar ve eşleştirme yolu, isteği sessizce aynı ana makineden gelen
bir bağlantı olarak değerlendirmek yerine açık onay gerektirir. Operatör kimlik
doğrulamasındaki eşdeğer kural için
[Güvenilir Proxy Kimlik Doğrulaması](/tr/gateway/trusted-proxy-auth) bölümüne bakın.

## Depolama (yerel, özel)

Eşleştirme durumu, Gateway durum dizini altındaki paylaşılan SQLite durum
veritabanında eşleştirilmiş cihaz kayıtlarında bulunur (varsayılan `~/.openclaw`):

- `~/.openclaw/state/openclaw.sqlite` (cihaz kimlik doğrulamasına sahip eşleştirilmiş cihazlar,
  onaylanmış Node yüzeyleri, bekleyen yüzey istekleri, bekleyen cihaz eşleştirme
  istekleri ve önyükleme token'ları)

`OPENCLAW_STATE_DIR` değerini geçersiz kılarsanız veritabanı da onunla birlikte taşınır. JSON
depolarını kullanan sürümlerden yükseltilen Gateway'ler, başlangıçta bunları içe aktarır ve
geride `devices/*.json.migrated` ile `nodes/*.json.migrated` arşivlerini bırakır.

Güvenlik notları:

- Cihaz token'ları gizlidir; durum veritabanını hassas kabul edin.
- Bir cihaz token'ını döndürmek için `openclaw devices rotate` /
  `device.token.rotate` kullanılır.

## Aktarım davranışı

- Aktarım **durumsuzdur**; üyelik bilgilerini depolamaz.
- Gateway çevrimdışıysa veya eşleştirme devre dışıysa Node'lar eşleşemez.
- Uzak modda eşleştirme, uzak Gateway'in deposunda gerçekleştirilir.

## İlgili konular

- [Kanal eşleştirmesi](/tr/channels/pairing)
- [Node CLI](/tr/cli/nodes)
- [Cihazlar CLI](/tr/cli/devices)
