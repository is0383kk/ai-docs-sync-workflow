---
read_when:
    - Gateway WS istemcilerini uygulama veya güncelleme
    - Protokol uyuşmazlıklarında veya bağlantı hatalarında hata ayıklama
    - Protokol şemasını/modellerini yeniden oluşturma
summary: 'Gateway WebSocket protokolü: el sıkışma, çerçeveler, sürümleme'
title: Gateway protokolü
x-i18n:
    generated_at: "2026-07-26T23:58:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89d637a9070bc6512a182fea0fd890b56287e0080515ba4fba9b2591c6247e0d
    source_path: gateway/protocol.md
    workflow: 16
---

Gateway WS protokolü, OpenClaw için tek denetim düzlemi ve Node aktarım mekanizmasıdır.
Operatör ve Node istemcileri (CLI, web kullanıcı arayüzü, macOS uygulaması, iOS/Android Node'ları,
başsız Node'lar) WebSocket üzerinden bağlanır ve el sıkışma sırasında bir **rol** ve **kapsam**
bildirir.

## npm paketleri

Bu paketler OpenClaw sürüm serileriyle birlikte sunulur. İlk kullanıma sunma sırasında,
paket içeren ilk sürüm yayımlanana kadar npm `E404` döndürebilir.

- [`@openclaw/gateway-protocol`](https://www.npmjs.com/package/@openclaw/gateway-protocol)
  şemaları, doğrulayıcıları, TypeScript türlerini, hafif çerçeve ve hata
  yardımcılarını ve sürüm sabitlerini yayımlar. Tarball dosyası, oluşturulan
  makine tarafından okunabilir [`protocol.schema.json`](https://unpkg.com/@openclaw/gateway-protocol/protocol.schema.json)
  sözleşmesini içerir.
- [`@openclaw/gateway-client`](https://www.npmjs.com/package/@openclaw/gateway-client)
  referans Node istemcisini ve `@openclaw/gateway-client/browser` konumunda
  tarayıcı açısından güvenli bir giriş noktasını yayımlar.

Uygulama yaşam döngüsü rehberliği için
[Gateway istemcisi oluşturma](https://docs.openclaw.ai/gateway/clients) bölümüne bakın. Gateway'i
alt süreç olarak yöneten uygulamalar için
[OpenClaw'u yerleştirme](https://docs.openclaw.ai/gateway/embedding) bölümüne bakın.

## Aktarım ve çerçeveleme

- WebSocket, metin çerçeveleri, JSON yükleri.
- İlk çerçeve **mutlaka** bir `connect` isteği olmalıdır.
- Bağlantı öncesi çerçeveler 64 KiB (`MAX_PREAUTH_PAYLOAD_BYTES`) ile sınırlıdır. El sıkışmadan
  sonra `hello-ok.policy.maxPayload` ve
  `hello-ok.policy.maxBufferedBytes` değerlerine uyun. Tanılama etkinleştirildiğinde, aşırı büyük
  gelen çerçeveler ve yavaş giden tamponlar, gateway bağlantıyı kapatmadan veya
  çerçeveyi bırakmadan önce `payload.large` olayları yayınlar. Bu olaylar
  `surface`, bayt boyutları, sınırlar ve güvenli bir neden kodu taşır;
  hiçbir zaman ileti gövdelerini, ek içeriklerini, ham çerçeve baytlarını,
  token'ları, çerezleri veya gizli bilgileri taşımaz.

Çerçeve biçimleri:

- İstek: `{type:"req", id, method, params}`
- Yanıt: `{type:"res", id, ok, payload|error}`
- Olay: `{type:"event", event, payload, seq?, stateVersion?}`

Yanıt hataları `{ code, message, details?, retryable?, retryAfterMs? }` kullanır.
İstemciler `code` ve `details.code` değerlerine göre dallanmalıdır; `message`,
bir uyumluluk notunda aksi belirtilmediği sürece insan tarafından okunabilir kalır
ve değişebilir. Yöntem düzeyindeki yetkilendirme hataları, yapılandırılmış
eksik kapsam ayrıntılarıyla üst düzey `code: "FORBIDDEN"` kullanır:

- Eksik kapsam: `{ code: "MISSING_SCOPE", missingScope, requiredScopes }`.
  `requiredScopes`, istenen işlem için bilinen kapsamların tam kümesidir.
  Eski `missing scope: <scope>` iletisi eski istemciler için korunur.

İstemciler önce `details` değerini okumalı ve eski iletiyi yalnızca uyumluluk
için geri dönüş seçeneği olarak kullanmalıdır. `readMissingScopeError` ve `readMissingScopeErrorDetails`,
`@openclaw/gateway-protocol/gateway-error-details` üzerinden dışa aktarılır; tarayıcı açısından güvenli
gateway istemcisi bunları `@openclaw/gateway-client/browser` üzerinden yeniden dışa aktarır.

Şemalar, `@openclaw/gateway-protocol/schema` üzerinden `GatewayErrorDetailsSchema`,
`MissingScopeErrorDetailsSchema` olarak dışa aktarılır.
HTTP kapsam hataları, `error.details` altında `MISSING_SCOPE` nesnesini yansıtır ve
`403` HTTP durumunu kullanır.

Yan etkiye sahip yöntemler idempotency anahtarları gerektirir (şemaya bakın).

## El sıkışma

Gateway, bağlantı öncesi bir sınama gönderir:

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

İstemci `connect` ile yanıt verir:

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 4,
    "maxProtocol": 4,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Gateway `hello-ok` ile yanıt verir:

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": {
    "type": "hello-ok",
    "protocol": 4,
    "server": { "version": "…", "connId": "…" },
    "features": { "methods": ["…"], "events": ["…"] },
    "snapshot": { "…": "…" },
    "auth": {
      "role": "operator",
      "scopes": ["operator.read", "operator.write"]
    },
    "policy": {
      "maxPayload": 26214400,
      "maxBufferedBytes": 52428800,
      "tickIntervalMs": 15000
    }
  }
}
```

`server`, `features`, `snapshot`, `policy` ve `auth`,
`HelloOkSchema` (`packages/gateway-protocol/src/schema/frames.ts`) tarafından zorunlu tutulur. `auth`,
hiçbir cihaz token'ı verilmediğinde bile anlaşmaya varılan rolü/kapsamları bildirir
(yukarıdaki biçim). `pluginSurfaceUrls` isteğe bağlıdır ve Plugin yüzeyi adlarını
(ör. `canvas`) kapsamlı barındırılan URL'lere eşler; süresi dolabileceğinden
Node'lar yeni bir giriş için `{ "surface": "canvas" }` ile `node.pluginSurface.refresh` çağrısı yapar.
Kullanımdan kaldırılan `canvasHostUrl` / `canvasCapability` / `node.canvas.capability.refresh`
yolu desteklenmez; Plugin yüzeylerini kullanın.
Anlık görüntünün isteğe bağlı `appliedConfigHash` değeri, etkin Gateway çalışma zamanı
tarafından kabul edilen çözümlenmiş kaynak yapılandırma revizyonudur. İstemciler,
daha yeni kaydedilmiş bir yapılandırmanın hâlâ yeniden başlatma gerektirip gerektirmediğini
belirlemek için bunu `config.get.configRevisionHash` ile karşılaştırabilir. `config.get.hash`,
yapılandırma yazma çakışması korumaları tarafından kullanılan ham kök dosya revizyonu
olarak kalır.

Gateway başlangıç yardımcı süreçlerini tamamlamaya devam ederken `connect`,
`details.reason: "startup-sidecars"` ve `retryAfterMs` içeren, yeniden denenebilir bir
`UNAVAILABLE` hatası döndürebilir. Bunu kalıcı bir el sıkışma hatası olarak
değerlendirmek yerine bağlantı bütçeniz içinde yeniden deneyin.

Bir cihaz token'ı verildiğinde `hello-ok.auth` bunu ekler:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

Yerleşik QR/kurulum kodu önyüklemesi, mobil cihazlara aktarım yoludur. Başarılı
bir temel kurulum kodu bağlantısı, birincil Node token'ının yanı sıra sınırlandırılmış
bir operatör token'ı döndürür:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

Bu operatör aktarımı bilerek sınırlandırılmıştır: Talk yapılandırması okumaları
için `operator.talk.secrets` dahil olmak üzere mobil operatör döngüsünü ve yerel kurulumu
başlatmaya yeterlidir, ancak eşleştirme değişikliği kapsamlarını ve `operator.admin`
değerini içermez. Daha geniş eşleştirme/yönetici erişimi için ayrı bir onaylı
eşleştirme veya token akışı gerekir. `hello-ok.auth.deviceTokens` değerini yalnızca önyükleme
kimlik doğrulaması güvenilir bir aktarım üzerinden (`wss://` veya
geri döngü/yerel eşleştirme) çalıştıysa kalıcı hâle getirin.

Aynı süreçteki güvenilir arka uç istemcileri (`client.id: "gateway-client"`,
`client.mode: "backend"`), paylaşılan gateway token'ı/parolasıyla kimlik doğrularken doğrudan
geri döngü bağlantılarında `device` değerini atlayabilir. Bu yol, dahili
denetim düzlemi RPC'lerine (ör. alt ajan oturum güncellemeleri) ayrılmıştır ve
eski CLI/cihaz eşleştirme temel değerlerinin yerel arka uç çalışmasını engellemesini
önler. Uzak, tarayıcı kaynaklı, Node ve açık cihaz token'ı/cihaz kimliği istemcileri
normal eşleştirme ve kapsam yükseltme kontrollerinden geçmeye devam eder.

### Çalışan rolü ve kapalı protokol

Bulut çalışanları, gateway'in sahip olduğu ve ana makine anahtarına sabitlenmiş
SSH tüneli üzerinden ayrılmış bir geri döngü girişini kullanır. Bu giriş yalnızca
çalışan kimliğini kabul eder; genel kimlik doğrulamasını, Node olaylarını, operatör
RPC'lerini veya Plugin yöntemlerini hiçbir zaman yönlendirmez. Katı bir `connect`,
ortama, paket karmasına, sahip dönemine, RPC kümesi sürümüne, sona erme zamanına
ve boş değer alabilen tek bir oturuma bağlı; saklama sırasında karmalanmış, kısa
ömürlü bir kimlik bilgisini doğrular ve ayrıca mevcut sürümü ve özellik kümesini
ayrı olarak denetler. Başarı durumunda asgari `worker-hello-ok` döndürülür; özellik
anlaşması genel protokol sürümünden bağımsızdır. Çerçeveler 64 KiB altında kalır;
ancak anlaşmaya varılmış bir `worker.inference.start` çerçevesi 25 MiB boyutuna kadar
olabilir. Kapalı izin listesi `worker.heartbeat`, `worker.transcript.commit`,
`worker.live-event`, `worker.inference.start` ve `worker.inference.cancel` değerlerini içerir.

Transkript işlemeleri sahip dönemi çitlemesini, gateway'in sahip olduğu bir oturum
bağlamasını, temel yaprak karşılaştırıp değiştirme işlemini ve kalıcı sıra yeniden
oynatmayı kullanır; gateway, normal oturum yazıcısı aracılığıyla transkript girdisi
ve üst öğe kimliklerini oluşturur. Sahiplik ve sona erme her RPC'de yeniden denetlenir.

### İstemci yetenekleri

Operatör istemcileri `connect.params.caps` içinde isteğe bağlı yetenekler bildirebilir:

- `tool-events`: yapılandırılmış araç yaşam döngüsü olaylarını kabul eder.
- `inline-widgets`: barındırılan satır içi widget araç sonuçlarını işleyebilir.

İstemci yetenekleri, yetkilendirmeyi değil bağlı istemciyi tanımlar. Ajan araçları gerekli yetenekleri bildirebilir; kaynak istemcinin `caps` değerinde her gereksinim bulunmadığı sürece Gateway bu araçları hariç tutar. Kanal kaynaklı çalıştırmaların Gateway istemci yetenekleri yoktur; bu nedenle araç ilkesi açıkça izin verse bile yetenek kısıtlamalı araçlar kullanılamaz.

### Node bağlantısı örneği

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 4,
    "maxProtocol": 4,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Node'lar bağlantı sırasında yetenek bildirimlerinde bulunur:

- `caps`: `camera`, `canvas`, `screen`,
  `location`, `voice`, `talk` gibi üst düzey kategoriler.
- `commands`: çağırma için komut izin listesi.
- `permissions`: ayrıntılı anahtarlar (ör. `screen.record`, `camera.capture`).

Gateway bunları bildirim olarak değerlendirir ve sunucu tarafı izin listelerini uygular.

## Roller ve kapsamlar

Tam operatör kapsam modeli, onay sırasındaki kontroller ve paylaşılan gizli bilgi
semantiği için [Operatör kapsamları](/tr/gateway/operator-scopes) bölümüne bakın.

Roller:

- `operator`: denetim düzlemi istemcisi (CLI/kullanıcı arayüzü/otomasyon).
- `node`: yetenek ana makinesi (kamera/ekran/tuval/system.run).
- `worker`: ayrılmış, kapalı çalışan protokolündeki bulut yürütme ana makinesi.

Operatör kapsamları (`src/gateway/operator-scopes.ts`), tam kapalı küme:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`includeSecrets: true` ile `talk.config`, `operator.talk.secrets` (veya
`operator.admin`) gerektirir. Gizli bilgiler dahil edildiğinde etkin Talk sağlayıcısının
kimlik bilgisini `talk.resolved.config.apiKey` üzerinden okuyun; `talk.providers.<id>.apiKey`
kaynak biçimini korur ve bir SecretRef nesnesi veya sansürlenmiş bir dize olabilir.

Plugin tarafından kaydedilen gateway RPC yöntemleri kendi operatör kapsamlarını
isteyebilir; ancak şu ayrılmış çekirdek ön ekleri her zaman `operator.admin`
(`src/shared/gateway-method-policy.ts`) olarak çözümlenir: `config.*`, `exec.approvals.*`,
`wizard.*`, `update.*`.

Yöntem kapsamı yalnızca ilk geçittir. `chat.send` üzerinden erişilen bazı
eğik çizgi komutları daha katı komut düzeyi kontroller uygular: kalıcı `/config set`
ve `/config unset` yazma işlemleri, daha düşük bir operatör kapsamına zaten sahip
gateway istemcileri için bile `operator.admin` gerektirir.

`node.pair.approve`, bekleyen isteğin bildirdiği `commands`
(`src/infra/node-pairing-authz.ts`) değerine göre temel yöntem kapsamına (`operator.pairing`) ek
olarak onay sırasında ilave bir kapsam kontrolüne sahiptir:

| Bildirilen komutlar                                                                                                           | Gerekli kapsamlar                      |
| ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| yok                                                                                                                           | `operator.pairing`                    |
| sıradan komutlar                                                                                                              | `operator.pairing` + `operator.write` |
| `system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `fs.listDir` veya `system.execApprovals.get/set` içerir | `operator.pairing` + `operator.admin` |

### Yetenekler/komutlar/izinler (Node)

Node'lar bağlantı sırasında yetenek beyanlarını bildirir:

- `caps`: `camera`, `canvas`, `screen`,
  `location`, `voice` ve `talk` gibi üst düzey yetenek kategorileri.
- `commands`: çağırma için komut izin listesi.
- `permissions`: ayrıntılı anahtarlar (ör. `screen.record`, `camera.capture`).

Gateway bunları **beyanlar** olarak değerlendirir ve sunucu tarafındaki izin listelerini uygular.
Bağlı Node'lar, başarılı bir bağlantı veya yeniden bağlantının ardından `node.pluginTools.update` ile
isteğe bağlı, aracı tarafından görülebilen Plugin ya da MCP araç tanımlayıcıları
yayımlayabilir. Başsız Node ana makineleri, bildirimsel MCP envanteri
değişikliklerini uygulamak için yeniden başlatılır. Bu güncelleme yöntemi tek yayımlama yoludur;
Plugin araç tanımlayıcıları `connect` parametrelerinde kabul edilmez.
Her tanımlayıcı, sağlayıcı açısından güvenli bir araç `name` kullanmalı ve
Node'un geçerli komut izin listesindeki bir `command` öğesini adlandırmalıdır.
Gateway, eşleştirilmiş Node'dan gelen tanımlayıcı meta verilerine güvenir,
onaylanan komut yüzeyinin dışındaki tanımlayıcıları filtreler, Node bağlantısı
kesildiğinde bunları kaldırır ve operatörlerin başka bir Node'un kataloğunu
değiştirme girişimlerini reddeder. Node tarafından yayımlanan tanımlayıcıları yok saymak için
`gateway.nodes.pluginTools.enabled: false` olarak ayarlayın.

Bağlı Node ana makineleri, eksiksiz Skills değiştirme kataloglarını
`node.skills.update` ile yayımlar. Bu Node rolü yöntemi, Node Skills yayımlamanın
tek yoludur; Skills, `connect` parametrelerinde kabul edilmez. Her tanımlayıcı;
güvenli bir ad, açıklama ve sınırlandırılmış `SKILL.md` içeriği barındırır.
Gateway bu içeriği normal Skills yükleyicisiyle ayrıştırır, Node bağlıyken
aracı Skills anlık görüntülerine ekler ve bağlantı kesildiğinde kaldırır.
Node tarafından yayımlanan Skills öğelerini yok saymak için
`gateway.nodes.allowSkills: false` olarak ayarlayın.

## Varlık

- `system-presence`, cihaz kimliğine göre anahtarlanmış ve
  `deviceId`, `roles` ile `scopes` içeren girdileri döndürür; böylece
  kullanıcı arayüzleri, cihaz hem operatör hem de Node olarak bağlansa bile cihaz başına bir satır gösterebilir.
- `node.list`, isteğe bağlı `lastSeenAtMs` ve `lastSeenReason` içerir. Bağlı
  Node'lar, geçerli bağlantı zamanını `connect` nedeniyle bildirir; eşleştirilmiş Node'lar
  güvenilir bir Node olayı aracılığıyla kalıcı arka plan varlığını da bildirebilir.

Yerel macOS Node'ları, sınırlandırılmış giriş boşta kalma süresiyle kimliği doğrulanmış
`node.presence.activity` olayları da gönderebilir. Gateway, etkinlik zaman damgalarını
kendi saatine göre türetir, en güncel bağlı Mac'i `node.list` ve
`node.describe` aracılığıyla sunar ve `node.presence` güncellemelerini okuma kapsamlı istemcilere yayımlar.
Kullanıcı etkinlik paylaşımını devre dışı bıraktığında uygulama `{ "action": "clear" }`
gönderir; Gateway zaman damgalarını yalnızca kimliği doğrulanmış bu tam Node bağlantısı için temizler.
Bu onaylanan eylemden daha eski Gateway sürümleri eylemi işlenmemiş olarak döndürür; bu nedenle Mac
Node bir kez yeniden bağlanır ve bağlantı kesme temizliğinin eski bağlantı durumunu kaldırmasını sağlar.
Seçim, gizlilik, model bağlamı ve bildirim yönlendirme davranışı için
[Etkin bilgisayar varlığı](/tr/nodes/presence) bölümüne bakın.

### Node arka planda canlı olayı

Node'lar, eşleştirilmiş bir Node'un arka planda uyanma sırasında canlı olduğunu
bağlı olarak işaretlemeden kaydetmek için `event: "node.presence.alive"` ile `node.event` çağrısı yapar:

```json
{
  "event": "node.presence.alive",
  "payloadJSON": "{\"trigger\":\"silent_push\",\"sentAtMs\":1737264000000,\"displayName\":\"Peter's iPhone\",\"version\":\"2026.4.28\",\"platform\":\"iOS 18.4.0\",\"deviceFamily\":\"iPhone\",\"modelIdentifier\":\"iPhone17,1\",\"pushTransport\":\"relay\"}"
}
```

`trigger` kapalı bir enum'dur: `background`, `silent_push`, `bg_app_refresh`,
`significant_location`, `manual`, `connect`. Bilinmeyen değerler
`background` (`src/shared/node-presence.ts`) olarak normalleştirilir. Olay yalnızca
kimliği doğrulanmış Node cihaz oturumları için kalıcılaştırılır; cihazı olmayan veya eşleştirilmemiş oturumlar
`handled: false` döndürür.

Başarılı Gateway'ler yapılandırılmış bir sonuç döndürür:

```json
{
  "ok": true,
  "event": "node.presence.alive",
  "handled": true,
  "reason": "persisted"
}
```

Eski Gateway'ler, `node.event` için yalnızca `{ "ok": true }` döndürebilir; bunu
kalıcı varlık kaydı olarak değil, onaylanmış bir RPC olarak değerlendirin.

## Yayın olayı kapsamlandırması

Sunucu tarafından gönderilen yayın olayları kapsamla sınırlandırılır; böylece eşleştirme kapsamlı veya yalnızca Node
oturumları, oturum içeriğini pasif olarak almaz
(`src/gateway/server-broadcast.ts`):

- Sohbet, aracı ve araç sonucu çerçeveleri (akışla iletilen `agent` olayları, araç sonucu
  olayları) en az `operator.read` gerektirir. Buna sahip olmayan oturumlar bu
  çerçeveleri tamamen atlar.
- Plugin tanımlı `plugin.*` yayınları varsayılan olarak `operator.write` veya
  `operator.admin` ile sınırlandırılır; `plugin.approval.requested` /
  `plugin.approval.resolved` gibi açık girdiler bunun yerine
  `operator.approvals` kullanır.
- Durum/taşıma olayları (`heartbeat`, `presence`, `tick`, bağlantı/bağlantı kesme
  yaşam döngüsü) sınırsız kalır; böylece taşıma sağlığı kimliği doğrulanmış her
  oturum tarafından gözlemlenebilir.
- Bilinmeyen yayın olayı aileleri, kayıtlı bir işleyici açıkça kısıtlamayı gevşetmediği sürece
  varsayılan olarak kapsamla sınırlandırılır (kapalı durumda başarısız olur).

Her istemci bağlantısı, kendine ait istemci bazlı sıra numarasını tutar; böylece farklı istemciler
olay akışının kapsamla filtrelenmiş farklı alt kümelerini görse bile yayınlar
ilgili sokette monoton biçimde sıralı kalır.

## RPC yöntem aileleri

`hello-ok.features.methods`, `src/gateway/server-methods-list.ts` ile yüklenen Plugin/kanal yöntem
dışa aktarımlarından oluşturulan tutucu bir keşif listesidir; her yöntemin
oluşturulmuş bir dökümü değildir ve bazı yöntemler (örneğin
`push.test`, `web.login.start`, `web.login.wait`, `sessions.usage`)
gerçek ve çağrılabilir yöntemler olmalarına rağmen keşif kapsamından bilinçli olarak çıkarılmıştır.
Bunu `src/gateway/server-methods/*.ts` öğesinin tam bir listesi değil, özellik keşfi olarak değerlendirin.

<AccordionGroup>
  <Accordion title="Sistem ve kimlik">
    - `health`, önbelleğe alınmış veya yeni yoklanmış Gateway sağlık anlık görüntüsünü döndürür.
    - `diagnostics.stability`, yakın zamandaki sınırlandırılmış tanılama kararlılığı kaydedicisini döndürür: olay adları, sayılar, bayt boyutları, bellek ölçümleri, kuyruk/oturum durumu, kanal/Plugin adları ve oturum kimlikleri. Sohbet metni, Webhook gövdeleri, araç çıktıları, ham istek/yanıt gövdeleri, token'lar, çerezler veya gizli bilgiler içermez. `operator.read` gerektirir.
    - `status`, `/status` tarzı Gateway özetini döndürür; hassas alanlar yalnızca yönetici kapsamlı operatör istemcilerine sunulur.
    - `gateway.identity.get`, aktarma ve eşleştirme akışlarında kullanılan Gateway cihaz kimliğini döndürür.
    - `system-presence`, bağlı operatör/Node cihazlarının geçerli varlık anlık görüntüsünü döndürür.
    - `system-event`, bir sistem olayı ekler ve varlık bağlamını güncelleyip yayımlayabilir.
    - `last-heartbeat`, kalıcılaştırılan en son Heartbeat olayını döndürür.
    - `set-heartbeats`, Gateway üzerindeki Heartbeat işlemeyi açar veya kapatır.
    - `gateway.suspend.prepare`, yalnızca izlenen Gateway işi boşta olduğunda kısa bir iş birlikçi askıya alma kirası oluşturur. `gateway.suspend.status` bu kirayı denetler ve `gateway.suspend.resume`, çözülmeden veya yarıda kesilen bir ana makine işleminden sonra kirayı serbest bırakır.

  </Accordion>

  <Accordion title="Modeller ve kullanım">
    - `models.list`, çalışma zamanında izin verilen model kataloğunu döndürür. Aşağıdaki "`models.list` görünümleri" bölümüne bakın.
    - `usage.status`, sağlayıcı kullanım pencerelerini/kalan kota özetlerini döndürür.
    - `usage.cost`, bir tarih aralığı için toplu maliyet kullanım özetlerini döndürür. Tek bir aracı için `agentId`, yapılandırılmış aracıları toplamak için `agentScope: "all"` iletin.
    - `doctor.memory.status`, etkin varsayılan aracı çalışma alanı için vektör belleği / önbelleğe alınmış gömme hazırlığını döndürür. Yalnızca açık bir canlı gömme sağlayıcısı pingi için `{ "probe": true }` veya `{ "deep": true }` iletin. Dreaming deposu istatistiklerini tek bir aracı çalışma alanıyla sınırlandırmak için `{ "agentId": "agent-id" }` iletin; belirtilmezse yapılandırılmış Dreaming çalışma alanları toplanır.
    - `doctor.memory.dreamDiary`, `doctor.memory.backfillDreamDiary`, `doctor.memory.resetDreamDiary`, `doctor.memory.resetGroundedShortTerm`, `doctor.memory.repairDreamingArtifacts` ve `doctor.memory.dedupeDreamDiary` isteğe bağlı `{ "agentId": "agent-id" }` kabul eder; belirtilmezse yapılandırılmış varsayılan aracı çalışma alanında çalışırlar.
    - `doctor.memory.remHarness`, çalışma alanı yolları, bellek parçacıkları, oluşturulmuş dayanaklı Markdown ve derin yükseltme adayları dahil olmak üzere uzak denetim düzlemi istemcileri için sınırlandırılmış, salt okunur bir REM donanım önizlemesi döndürür. `operator.read` gerektirir.
    - `sessions.usage`, oturum bazında kullanım özetlerini döndürür. Tek bir aracı için `agentId`, yapılandırılmış aracıları birlikte listelemek için `agentScope: "all"` iletin.
      Her iki kullanım yöntemi de yaz saati uygulamasını dikkate alan takvim günü sınırları ve grupları için IANA `timeZone` içeren `mode: "specific"` kabul eder. `utcOffset`, eski istemciler için ve Gateway çalışma zamanı istenen bölgeyi tanımadığında geri dönüş seçeneği olarak desteklenmeye devam eder.
    - `sessions.usage.timeseries`, tek bir oturum için zaman serisi kullanımını döndürür.
    - `sessions.usage.logs`, tek bir oturum için kullanım günlüğü girdilerini döndürür.

  </Accordion>

  <Accordion title="Kanallar ve oturum açma yardımcıları">
    - `channels.status`, yerleşik ve paketlenmiş kanal/Plugin durum özetlerini döndürür.
    - `channels.logout`, kanal destekliyorsa belirli bir kanal/hesap oturumunu kapatır.
    - `web.login.start`, QR özelliğine sahip geçerli web kanalı sağlayıcısı için QR/web oturum açma akışı başlatır.
    - `web.login.wait`, bu akışın tamamlanmasını bekler ve başarı durumunda kanalı başlatır.
    - `push.test`, kayıtlı bir iOS Node'una test amaçlı bir APNs anlık bildirimi gönderir.
    - `voicewake.get`, saklanan uyandırma sözcüğü tetikleyicilerini döndürür.
    - `voicewake.set`, uyandırma sözcüğü tetikleyicilerini günceller ve değişikliği yayımlar.

  </Accordion>

  <Accordion title="Plugin yönetimi">
    - `plugins.list` (`operator.read`), yüklü Plugin envanterinin yanı sıra yerel olarak derlenmiş resmî seçimleri, tanılamaları ve mevcut yükleme modunun değişikliklere izin verip vermediğini döndürür.
    - `plugins.search` (`operator.read`), yüklenebilir ClawHub kod Plugin'i ve paket Plugin'i ailelerini arar. Boş olmayan `query` ve 1 ile 100 arasında isteğe bağlı `limit` iletin.
    - `plugins.install` (`operator.admin`), `{ source: "official", pluginId }` ile resmî bir katalog girdisini veya `{ source: "clawhub", packageName, version?, acknowledgeClawHubRisk? }` ile bir ClawHub paketini yükler. ClawHub yüklemeleri Gateway güveni, bütünlük ve yükleme politikası denetimlerini korur. Başarılı yüklemeler Gateway'in yeniden başlatılmasını gerektirir.
    - `plugins.setEnabled` (`operator.admin`), yüklü bir Plugin'in etkinleştirme politikasını `{ pluginId, enabled }` ile değiştirir. Yanıt; güncellenmiş katalog girdisini, yeniden başlatma meta verilerini ve yuva seçimiyle ilgili uyarıları içerir.
    - `plugins.uninstall` (`operator.admin`), haricî olarak yüklenmiş bir Plugin'i `{ pluginId }` ile kaldırır: yapılandırma başvuruları, yükleme kaydı ve yönetilen dosyalar. Paketle birlikte gelen Plugin'ler kaldırılamaz, yalnızca devre dışı bırakılabilir. Yanıt, kaldırma işlemlerini listeler ve her zaman Gateway'in yeniden başlatılmasını gerektirir.

  </Accordion>

  <Accordion title="Mesajlaşma ve günlükler">
    - `send`, sohbet çalıştırıcısının dışında kanal/hesap/ileti dizisi hedefli gönderimler için doğrudan giden teslimat RPC'sidir.
    - `logs.tail`, yapılandırılmış Gateway dosya günlüğünün son bölümünü imleç/sınır ve azami bayt denetimleriyle döndürür.

  </Accordion>

  <Accordion title="Operatör terminali">
    - `terminal.open`, açıkça belirtilen bir `agentId` veya varsayılan ajan için ana makine PTY'si başlatır ve çözümlenen ajanı, çalışma dizinini, kabuğu ve yalıtım durumunu döndürür.
    - `terminal.input`, `terminal.resize` ve `terminal.close` yalnızca çağrıyı yapan bağlantının sahip olduğu oturumlar üzerinde çalışır.
    - `terminal.upload`, 16 MiB'a kadar bir base64 dosyasını kabul eder, oturumun Gateway veya eşleştirilmiş Node ana makinesindeki özel ve 24 saatlik geçici bir dizine yerleştirir ve mutlak yolu döndürür. Çağıranın bu yolu yine de yapıştırması veya başka bir şekilde kullanması gerekir; RPC hiçbir zaman terminal girdisi yazmaz veya komut çalıştırmaz.
    - `terminal.data` ve `terminal.exit` olayları yalnızca oturumun sahibi olan bağlantıya aktarılır.
    - Bağlantısı kesilen oturumlar sonlandırılmaz, bağlantıları ayrılır: son çıktılar sınırlı bir sunucu tarafı arabelleğinde birikirken `gateway.terminal.detachedSessionTimeoutSeconds` boyunca yeniden bağlanabilir durumda kalırlar (varsayılan 300; `0`, bağlantı kesildiğinde sonlandırma davranışını geri getirir).
    - `terminal.list`, bağlanılabilir oturumları döndürür; `terminal.attach`, canlı veya bağlantısı ayrılmış bir oturumu çağrıyı yapan bağlantıya yeniden bağlar ve yeniden oynatma arabelleğini döndürür (tmux tarzı devralma — önceki canlı sahip, nedeni `detached` olan `terminal.exit` alır); `terminal.text`, bağlanmadan arabelleği düz metin olarak okur.
    - Her terminal yöntemi `operator.admin` gerektirir; `gateway.terminal.enabled` açıkça true olmalıdır. Tamamen korumalı alan içinde çalışan ajanlar reddedilir ve ajan politikasındaki bir değişiklik, bağlantısı ayrılmış olanlar dâhil mevcut ve işlem hâlindeki PTY'leri kapatır.

  </Accordion>

  <Accordion title="Talk ve TTS">
    - `talk.catalog`; konuşma, akışlı transkripsiyon ve gerçek zamanlı ses için salt okunur Talk sağlayıcı kataloğunu döndürür: sağlayıcı sırlarını döndürmeden veya genel yapılandırmayı değiştirmeden kanonik sağlayıcı kimlikleri, kayıt defteri takma adları, etiketlar, yapılandırılma durumu, isteğe bağlı grup düzeyinde bir `ready` sonucu, kullanıma sunulan model/ses kimlikleri, kanonik modlar, taşıma yöntemleri, beyin stratejileri ve gerçek zamanlı ses/yetenek bayrakları. Güncel Gateway'ler, çalışma zamanı sağlayıcı seçimini uyguladıktan sonra `ready` değerini ayarlar; eski Gateway'lerde bunun bulunmamasını doğrulanmamış olarak değerlendirin.
    - `talk.config`, geçerli Talk yapılandırma yükünü döndürür; `includeSecrets`, `operator.talk.secrets` (veya `operator.admin`) gerektirir.
    - `talk.session.create`, `realtime/gateway-relay`, `transcription/gateway-relay` veya `stt-tts/managed-room` için Gateway'in sahip olduğu bir Talk oturumu oluşturur. `stt-tts/managed-room` için `sessionKey` ileten `operator.write` çağıranları, kapsamlı oturum anahtarı görünürlüğü için `spawnedBy` değerini de iletmelidir; kapsamsız `sessionKey` oluşturma ve `brain: "direct-tools"`, `operator.admin` gerektirir.
    - `talk.session.join`, yönetilen oda oturum belirtecini doğrular, gerektiğinde `session.ready` veya `session.replaced` yayar ve düz metin belirteci ya da karmasını hiçbir zaman döndürmeden oda/oturum meta verilerinin yanı sıra son Talk olaylarını döndürür.
    - `talk.session.appendAudio`, Gateway'in sahip olduğu gerçek zamanlı aktarma ve transkripsiyon oturumlarına base64 PCM giriş sesi ekler.
    - `talk.session.startTurn`, `talk.session.endTurn` ve `talk.session.cancelTurn`, durum temizlenmeden önce eski tur reddi uygulayarak yönetilen oda turu yaşam döngüsünü yürütür.
    - `talk.session.cancelOutput`, öncelikle Gateway aktarma oturumlarında VAD kapılı araya girme için asistan ses çıkışını durdurur.
    - `talk.session.submitToolResult`, Gateway'in sahip olduğu gerçek zamanlı aktarma oturumunun yaydığı sağlayıcı araç çağrısını tamamlar. İstek, sağlayıcı köprüsünün sunduğu tüm eşzamansız tamamlanma sinyallerini bekler; başarısız gönderimler bağlantılı çalıştırmayı etkin tutar ve başarılı bir araç sonucu olayı yaymaz. Ara araç çıktısı için `options: { willContinue: true }` veya sağlayıcı köprüsü bastırma desteğini bildirdiğinde ve sonucun başka bir yanıt başlatmaması gerektiğinde `options: { suppressResponse: true }` iletin.
    - `talk.session.steer`, Gateway'in sahip olduğu ajan destekli Talk oturumuna etkin çalıştırma ses denetimi gönderir: `{ sessionId, text, mode? }`; burada `mode`, `status`, `steer`, `cancel` veya `followup` olur; mod belirtilmezse konuşulan metne göre sınıflandırılır.
    - `talk.session.close`, Gateway'in sahip olduğu bir aktarma, transkripsiyon veya yönetilen oda oturumunu kapatır ve sonlandırıcı Talk olayları yayar.
    - `talk.mode`, WebChat/Control UI istemcileri için geçerli Talk modu durumunu ayarlar/yayınlar.
    - `talk.client.create`, Gateway kimlik bilgileri, talimatlar, araç politikası ve döndürülen `voiceSessionId` üzerinde sahipliğini korurken `webrtc` veya `provider-websocket` kullanarak istemcinin sahip olduğu gerçek zamanlı sağlayıcı oturumunu oluşturur veya sürdürür. İstemciler `sessionKey` iletir ve tek bir çağrı sırasında sağlayıcı taşıma yöntemini değiştirirken `voiceSessionId` değerini yeniden kullanır.
    - `talk.client.transcript`, normal ajan oturumuna tamamlanmış bir `{ role, text }` öğesi ekler. Gerekli `entryId`, `voiceSessionId` içinde eş etkili çalışır; yeniden denemeler transkript iletilerini çoğaltmaz.
    - `talk.client.close`, bekleyen transkript yazmalarından sonra mantıksal ses oturumunu kapatır. Kapatma eş etkilidir ve yalnızca değişiklik içeren bir çağrı özetini oturumun WebChat dışındaki son kanalına teslim edebilir.
    - `talk.client.toolCall`, istemcinin sahip olduğu gerçek zamanlı taşıma yöntemlerinin sağlayıcı araç çağrılarını Gateway politikasına iletmesini sağlar. Desteklenen ilk araç `openclaw_agent_consult`'dir; istemciler bir çalıştırma kimliği alır ve sağlayıcıya özgü araç sonucunu göndermeden önce normal sohbet yaşam döngüsü olaylarını bekler. Sese bağlı yüksek etkili eylemler, daha sonra tamamlanan bir kullanıcı ifadesi tam olarak bu eylemi açıkça onaylayana ve sonraki danışma `confirmationId` değerini sağlayana kadar `VOICE_CONFIRMATION_REQUIRED:<id>` döndürür.
    - `talk.client.steer`, istemcinin sahip olduğu gerçek zamanlı taşıma yöntemleri için etkin çalıştırma ses denetimi gönderir. Gateway, `sessionKey` üzerinden etkin gömülü çalıştırmayı çözümler ve yönlendirmeyi sessizce yok saymak yerine yapılandırılmış bir kabul/ret sonucu döndürür.
    - `talk.event`; gerçek zamanlı, transkripsiyon, STT/TTS, yönetilen oda, telefon ve toplantı bağdaştırıcıları için tek Talk olay kanalıdır.
    - `talk.speak`, etkin Talk konuşma sağlayıcısı üzerinden konuşma sentezler.
    - `tts.status`, TTS etkinlik durumunu, etkin sağlayıcıyı, yedek sağlayıcıları ve sağlayıcı yapılandırma durumunu döndürür.
    - `tts.providers`, görünür TTS sağlayıcı envanterini döndürür.
    - `tts.enable` ve `tts.disable`, TTS tercihleri durumunu açıp kapatır.
    - `tts.setProvider`, tercih edilen TTS sağlayıcısını günceller.
    - `tts.convert`, tek seferlik metinden konuşmaya dönüştürme işlemi çalıştırır.
    - `tts.speak` (`operator.write`), boş olmayan `text` değerini yapılandırılmış genel TTS sağlayıcı zinciriyle işler ve bir bütün klibi satır içi olarak `audioBase64` biçiminde, ayrıca `provider` ve isteğe bağlı `outputFormat`, `mimeType` ve `fileExtension` meta verileriyle döndürür. `tts.convert` aksine Gateway'e yerel bir yol döndürmez; `talk.speak` aksine Talk sağlayıcısı gerektirmez. `tts.maxTextLength` üzerindeki metin `INVALID_REQUEST` döndürür; sentez hataları `UNAVAILABLE` döndürür.

  </Accordion>

  <Accordion title="Gizli bilgiler, yapılandırma, güncelleme ve sihirbaz">
    - `secrets.reload` etkin SecretRef'leri yeniden çözümler ve sahip bilgisine duyarlı çalışma zamanı durumunu atomik olarak yayımlar. Uygun sahip hataları, `warningCount` ile soğuk veya eski durum indirgemesi olarak yayımlanabilir; katı ya da eşlenmemiş hatalar yeniden yüklemeyi reddeder ve etkin anlık görüntüyü korur.
    - `secrets.resolve` belirli bir komut/hedef kümesi için komut hedefi gizli bilgi atamalarını çözümler.
    - `config.get` diskteki geçerli yapılandırma anlık görüntüsünü, ham kök dosya `hash`, çözümlenmiş `configRevisionHash` ve etkin Gateway çalışma zamanı tarafından kabul edilen çözümlenmiş revizyon için isteğe bağlı `appliedConfigHash` değerini döndürür.
    - `config.set` doğrulanmış bir yapılandırma yükü yazar.
    - `config.patch` kısmi bir yapılandırma güncellemesini birleştirir. Yıkıcı dizi değiştirme işlemi, etkilenen yolun `replacePaths` içinde bulunmasını gerektirir; dizi girdileri altındaki iç içe diziler, `agents.entries.*.skills` gibi `[]` yollarını kullanır.
    - `config.apply` tam yapılandırma yükünü doğrular ve değiştirir.
    - `config.schema` Control UI ve CLI araçları tarafından kullanılan canlı yapılandırma şeması yükünü döndürür: şema, `uiHints`, sürüm, oluşturma meta verileri ve yüklenebildiklerinde plugin + kanal şeması meta verileri. Eşleşen alan belgelendirmesi mevcut olduğunda iç içe nesne, joker karakter, dizi öğesi ve `anyOf` / `oneOf` / `allOf` bileşim dalları da dahil olmak üzere, kullanıcı arayüzüyle aynı etiketlerden/yardım metninden alınan `title` / `description` meta verilerini içerir.
    - `config.schema.lookup` tek bir yapılandırma yolu için yol kapsamlı bir arama yükü döndürür: normalleştirilmiş yol, sığ bir şema düğümü, eşleşen ipucu + `hintPath`, isteğe bağlı `reloadKind` ve UI/CLI ayrıntı incelemesi için doğrudan alt öğe özetleri. `reloadKind`, `restart`, `hot` veya `none` (`src/config/schema.ts`) değerlerinden biridir ve istenen yol için Gateway yapılandırması yeniden yükleme planlayıcısını yansıtır. Arama şeması düğümleri, kullanıcıya yönelik belgeleri ve yaygın doğrulama alanlarını (`title`, `description`, `type`, `enum`, `const`, `format`, `pattern`, sayısal/dize/dizi/nesne sınırları, `additionalProperties`, `deprecated`, `readOnly`, `writeOnly`) korur. Alt öğe özetleri `key`, normalleştirilmiş `path`, `type`, `required`, `hasChildren`, isteğe bağlı `reloadKind` ve eşleşen `hint` / `hintPath` değerlerini sunar.
    - `update.run` Gateway güncelleme akışını çalıştırır ve yalnızca güncelleme başarılı olursa yeniden başlatma zamanlar; oturumu olan çağıranlar `continuationMessage` değerini ekleyebilir; böylece başlangıç, yeniden başlatma devam kuyruğu üzerinden bir takip ajan turunu sürdürür. Kontrol düzleminden yapılan paket yöneticisi güncellemeleri ve denetimli git çalışma kopyası güncellemeleri, canlı Gateway içinde paket ağacını değiştirmek veya çalışma kopyası/derleme çıktısını dönüştürmek yerine ayrılmış bir yönetilen hizmet devri kullanır. Başlatılmış bir devir, `result.reason: "managed-service-handoff-started"` ve `handoff.status: "started"` ile `ok: true` döndürür. Aynı Gateway işlemi tarafından ele alınan eşzamanlı ikinci bir `update.run`, `result.reason: "managed-service-handoff-already-running"` ve `handoff.status: "already-running"` ile `ok: false` döndürür; devam isteği kabul edilmez, böylece çağıran etkin güncelleme tamamlandıktan sonra yeniden deneyebilir. Bağımsız CLI güncelleyicileri ve yedek Gateway işlemleri bu işlem yerelindeki korumanın dışındadır. Kullanılamayan veya başarısız devirler, `managed-service-handoff-unavailable` ya da `managed-service-handoff-failed` ile `ok: false` ve manuel kabuk güncellemesi gerektiğinde ayrıca `handoff.command` döndürür. Kullanılamaz durumu, OpenClaw'ın systemd için `OPENCLAW_SYSTEMD_UNIT` gibi güvenli bir gözetmen sınırına veya kalıcı hizmet kimliğine sahip olmadığı anlamına gelir. Başlatılmış bir devir sırasında yeniden başlatma işaretçisi kısa süreliğine `stats.reason: "restart-health-pending"` bildirebilir; devam işlemi, CLI yeniden başlatılan Gateway'i doğrulayıp nihai `ok` işaretçisini yazana kadar geciktirilir.
    - `update.status`, mevcut olduğunda yeniden başlatma sonrası çalışan sürüm de dahil olmak üzere en son güncelleme yeniden başlatma işaretçisini yeniler ve döndürür.
    - `wizard.start`, `wizard.next`, `wizard.status` ve `wizard.cancel`, ilk katılım sihirbazını WS RPC üzerinden sunar.

  </Accordion>

  <Accordion title="Ajan ve çalışma alanı yardımcıları">
    - `agents.list`, etkin model/çalışma zamanı meta verileri ve isteğe bağlı anlamsal `kind` (`agent` veya `system`) dahil olmak üzere Gateway tarafından görülebilen ajan girdilerini döndürür. İstemciler, eksiksiz türlenmiş listeyi almak için `agent-kind` el sıkışma yeteneğini bildirir; bunu desteklemeyen istemciler, sistem satırları içermeyen eski ve seçicide güvenle kullanılabilen listeyi korur. Tür bilgisine duyarlı istemciler, `system` satırlarını tanılama görünümlerinde tutarken sıradan seçicilerden hariç tutar. Eski v4 Gateway'ler, `kind` içermeyen satırlar döndürebilir.
    - `agents.create`, `agents.update` ve `agents.delete`, ajan kayıtlarını ve çalışma alanı bağlantılarını yönetir.
    - `agents.files.list`, `agents.files.get` ve `agents.files.set`, bir ajan için sunulan başlangıç çalışma alanı dosyalarını yönetir.
    - `audit.activity.list`, sürümlendirilmiş yalnızca meta veri içeren etkinlik defterini döndürür; `audit.list` uyumluluk açısından güvenli çalıştırma/araç RPC'si olarak kalır.
    - `agents.workspace.list` ve `agents.workspace.get` (`operator.read`), [Operatör kapsamları](/tr/gateway/operator-scopes) bölümünde açıklanan güvenilir operatör etki alanındaki istemciler için bir ajanın çalışma alanı dizinine salt okunur, sayfalandırılmış göz atma erişimi sunar. İstekler yalnızca çalışma alanına göreli yolları kabul eder; okumalar gerçek yolu çözümlenmiş çalışma alanı köküyle sınırlı kalır (sembolik bağlantı ve sabit bağlantı üzerinden kaçışlar reddedilir), boyut sınırına tabidir ve UTF-8 metin ile yaygın görüntü türleriyle (base64) sınırlıdır. Yanıtlar ana makinedeki çalışma alanı yolunu açığa çıkarmaz. Bu ad alanında yazma işlemi yoktur.
    - `tasks.list`, `tasks.get` ve `tasks.cancel`, Gateway görev defterini SDK ve operatör istemcilerine sunar. Aşağıdaki [Görev defteri RPC'leri](#task-ledger-rpcs) bölümüne bakın.
    - `artifacts.list`, `artifacts.get` ve `artifacts.download`, açık bir `sessionKey`, `runId` veya `taskId` kapsamı için transkriptten türetilmiş yapıt özetlerini ve indirmeleri sunar. Çalıştırma ve görev sorguları, sahibi olan oturumu sunucu tarafında çözümler ve yalnızca eşleşen kökene sahip transkript medyasını döndürür; güvenli olmayan veya yerel URL kaynakları, sunucu tarafında getirilmek yerine desteklenmeyen indirmeler döndürür.
    - `environments.list` ve `environments.status`, Gateway'e yerel ortam ve Node ortamı keşfini korur. Yapılandırılmış bulut işçileri ve önceki profillerden kalan kalıcı kayıtlar; `providerId`, isteğe bağlı `leaseId`, `state`, `ageMs`, isteğe bağlı `idleMs` ve `attachedSessionIds` ile `worker` meta verilerini ekler. İşçi yaşam döngüsü durumları `requested`, `provisioning`, `bootstrapping`, `ready`, `attached`, `idle`, `draining`, `destroying`, `destroyed`, `failed` ve `orphaned` şeklindedir.
    - `environments.create` (`{ profileId, idempotencyKey }`), yapılandırılmış bir plugin sağlayıcı profilinden bir işçi hazırlar; aynı anahtarla yapılan yeniden denemeler kalıcı işlemi yeniden kullanır. `environments.destroy` (`{ environmentId }`), kalıcı bir işçi ortamının eş etkili olarak kaldırılmasını ister. Her ikisi de `operator.admin` gerektirir, kontrol düzlemi yazma işlemleridir ve durum yanıtlarında kullanılanla aynı ortam özeti biçimini döndürür.
    - `agent.identity.get`, bir ajan veya oturum için etkin asistan kimliğini döndürür.
    - `agent.wait`, bir çalıştırmanın tamamlanmasını bekler ve mevcut olduğunda son durum anlık görüntüsünü döndürür.

  </Accordion>

  <Accordion title="Oturum denetimi">
    - `sessions.list`, bir agent çalışma zamanı arka ucu yapılandırıldığında satır başına `agentRuntime` meta verileri dâhil olmak üzere geçerli oturum dizinini döndürür. Bulut çalışanı yerleşimi etkinleştirildiğinde veya kalıcı kurtarma durumu mevcut olduğunda, oturum satırları ayrıca kapalı bir `placement` durumu (`local`, `requested`, `provisioning`, `syncing`, `starting`, `active`, `draining`, `reconciling`, `reclaimed` veya `failed`) ile duruma özgü ortam, sahip dönemi, çalışma alanı, paket, ACK imleci veya kurtarma alanlarını içerir.
    - `sessions.subscribe` ve `sessions.unsubscribe`, geçerli WS istemcisi için oturum değişikliği olayı aboneliklerini açıp kapatır.
    - `sessions.messages.subscribe` ve `sessions.messages.unsubscribe`, tek bir oturum için transkript/mesaj olayı aboneliklerini açıp kapatır. Kalıcı hedef kitlesi tam olarak bu oturumu içeren ve inceleyici bağlaması abone olan istemciyi yetkilendiren onaylara ait temizlenmiş `session.approval` yaşam döngüsü olaylarını da almak için `includeApprovals: true` iletin. Ardından abonelik yanıtı, sınırlandırılmış bekleyen bir `approvalReplay` içerir; `truncated` false olduğunda bu değer yetkili kaynaktır. Etkinleştirme her abonelik çağrısına özeldir ve kalıcı değildir: aynı oturuma `includeApprovals: true` olmadan yeniden abone olmak, mevcut onay aboneliğini kaldırır. Normal oturum okuma yetkisine ek olarak bu etkinleştirme, eşleştirilmiş bir cihazda `operator.admin` veya `operator.approvals` gerektirir.
    - `sessions.preview`, belirli oturum anahtarları için sınırlandırılmış transkript önizlemeleri döndürür.
    - `sessions.describe`, tam bir oturum anahtarı için tek bir Gateway oturum satırı döndürür.
    - `sessions.resolve`, bir oturum hedefini çözümler veya standartlaştırır.
    - `sessions.create`, yeni bir oturum girdisi oluşturur. İsteğe bağlı `model` ve `thinkingLevel` değerleri, ilk model ve akıl yürütme geçersiz kılmalarını atomik olarak kalıcı hâle getirir. `worktree: true`, yönetilen bir çalışma ağacı hazırlar; isteğe bağlı `worktreeBaseRef`/`worktreeName`, temel referansı ve dal adını seçer; `execNode` (`operator.admin`) ise oturum yürütmesini bir Node ana makinesine bağlar. Oluşturulan çalışma ağacı sonuçta aynen döndürülür ve oturum satırında (`worktree: { id, branch, repoRoot }`) kalıcı hâle getirilir. Girdi oluşturulduğu hâlde iç içe geçmiş ilk `chat.send` reddedildiğinde başarılı sonuç, `runStarted: false` ve `runError` alanlarını içerir; istemciler istemi koruyup döndürülen oturum anahtarıyla yeniden deneyebilir. `parentSessionKey` ile birlikte `emitCommandHooks: true` ileten bir çağıran, ayrı bir alt öğenin yaşam döngüsü sonucunu da bildirmelidir: `succeedsParent: true`, üst öğeyi `session_end` ile sonlandırırken `false`, üst öğeyi etkin tutar ve yalnızca alt öğenin `session_start` olayını yayar. `succeedsParent` değerinin belirtilmemesi, mevcut istemciler için eski üst öğe devretme davranışını korur. Sonuç hem üst öğe bağlantısı hem de komut kancaları gerektirir; bir çatallanma üst öğesini başarılı olarak sonuçlandıramaz. Ayrı bir alt öğe oluşturulmadığından ana oturumun yerinde sıfırlama davranışı değişmez. Yeni satırlar, güvenilir oluşturma bağlantı noktasından gelen ve yalnızca bir kez yazılabilen oluşturma kaynağı bilgileriyle (`createdVia`, `createdActor`, `createdAt`) damgalanır; mevcut bir anahtarı benimsemek bu bilgileri hiçbir zaman yeniden damgalamaz. İnsan profili aktörleri için `createdActor.label`, satır yansıtılırken geçerli kullanıcı profilinden çözümlenir ve oturum girdisinde hiçbir zaman depolanmaz; böylece profil yeniden adlandırmalarında sapma oluşmaz. Oturum satırları ayrıca `parentSessionKey` (gezinme üst öğesi, kalıcı), `controlOwnerSessionKey` (canlıyken çalışma zamanı denetleyicisi), `forkSource` (çatallanmalar için tam kaynak anahtarı + transkript nesli) ve `previousSessionId` (aynı anahtar altındaki önceki transkript nesli) alanlarını taşır.
    - `sessions.dispatch` (`operator.admin`), oturuma ait yönetilen bir çalışma ağacı bulunan mevcut bir yerel OpenClaw oturumunu yapılandırılmış bir bulut çalışanı profiline taşır. `{ key, profileId, agentId? }` iletin. Hiçbir çalışan profili yapılandırılmadığında yöntem mevcut değildir; etkin işleri boşaltmadan önce yerel tur kabulünü kapatır ve yalnızca yerleşim `active` çalışan sahipliğine ulaştıktan sonra döner. Gönderim tek yönlüdür; çalışandan yerele geri çekme bu RPC'nin parçası değildir.
    - `sessions.groups.list`, `sessions.groups.put`, `sessions.groups.rename` ve `sessions.groups.delete`, Gateway'in sahip olduğu özel oturum grubu kataloğunu (adlar + görüntüleme sırası) yönetir. Üyelik, her oturumun `category` alanında kalır; yeniden adlandırma ve silme işlemleri üye oturumları sunucu tarafında günceller.
    - `sessions.send`, mevcut bir oturuma mesaj gönderir.
    - `sessions.steer`, etkin bir oturum için kesme ve yönlendirme çeşididir.
    - `sessions.abort`, bir oturumun etkin çalışmasını iptal eder. `key` ile isteğe bağlı `runId` değerini veya Gateway'in bir oturumla ilişkilendirebildiği etkin çalıştırmalar için yalnızca `runId` değerini iletin. `runId` sağlamak, iptali ilgili çalıştırmayla sınırlar. Yalnızca anahtar içeren, genel olmayan bir istekte `clearQueued: true` değerini ayarlayarak bu oturumun sahip olduğu takip ve hat kuyruklarını da atın. `clearQueued` değerini belirtmeyen mevcut çağıranlar bu kuyrukları korur. Değişmez `global` anahtarı, mevcut agent nitelikli `chat.abort` sahiplik kurallarını korur ve genel olmayan takip ya da hat temizliği gerçekleştirmez.
    - `sessions.patch`, oturum meta verilerini/geçersiz kılmalarını günceller ve çözümlenmiş standart model ile etkin `agentRuntime` değerini bildirir. Oluşturma kökeni (`spawnedBy`, `spawnedWorkspaceDir`, `spawnedCwd`, `spawnDepth`, `subagentRole`, `subagentControlScope`) artık herkese açık biçimde yamalanamaz; bu bilgiler güvenilir oluşturma yolları tarafından bir kez yazılır ve bunları hâlâ gönderen istekler reddedilir.
    - `sessions.reset`, `sessions.delete` ve `sessions.compact`, oturum bakımını gerçekleştirir.
    - `sessions.get`, depolanan tam oturum satırını döndürür.
    - Sohbet yürütmesi hâlâ `chat.history`, `chat.send`, `chat.abort` ve `chat.inject` kullanır. `chat.history`, UI istemcileri için görüntüleme amacıyla normalleştirilir: satır içi yönerge etiketleri görünür metinden çıkarılır; düz metin araç çağrısı XML yükleri (`<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` ve kesilmiş araç çağrısı blokları) ile sızmış ASCII/tam genişlikli model denetim belirteçleri çıkarılır; yalnızca sessiz belirteç içeren assistant satırları (tam olarak `NO_REPLY` / `no_reply`) atlanır ve aşırı büyük satırlar yer tutucularla değiştirilebilir.
    - `chat.message.get`, görünür tek bir transkript girdisi için eklemeli, sınırlandırılmış tam mesaj okuyucusudur. `sessionKey`, oturum seçimi agent kapsamlı olduğunda isteğe bağlı `agentId` ve daha önce `chat.history` üzerinden gösterilmiş bir transkript `messageId` değeri iletin; depolanan girdi hâlâ mevcutsa ve aşırı büyük değilse Gateway, hafif geçmiş kesme sınırı olmadan aynı görüntüleme için normalleştirilmiş yansıtmayı döndürür.
    - `chat.toolTitles`, Control UI'da işlenen araç çağrıları için kısa amaç başlıkları döndürür (toplu, sınırlandırılmış girdilerle en fazla 24 öğe). Özellik `gateway.controlUi.toolTitles` aracılığıyla etkinleştirilir (varsayılan olarak kapalıdır); devre dışı Gateway'ler, istemcilerin sormayı bırakması için `{ titles: {}, disabled: true }` yanıtını model çağrısı yapmadan verir. Etkinleştirildiğinde başlıklar standart yardımcı model yönlendirmesini kullanır: açıkça yapılandırılmış bir `utilityModel` (tüm yardımcı görevlerde olduğu gibi, sınırlandırılmış görev içeriğini seçilen sağlayıcıya gönderebilecek bir operatör kararı), aksi takdirde dolaylı olarak yeni bir çıkış hedefi oluşmaması için oturum sağlayıcısının bildirdiği küçük model varsayılanı; boş bir `utilityModel` ise bunları tamamen devre dışı bırakır. Başlıklar hiçbir zaman birincil modele geri dönmez. Sonuçlar, araç adı + girdi anahtarıyla agent başına durum veritabanında önbelleğe alınır; böylece tekrarlanan görünümler aynı çağrıları yeniden ücretlendirmez.
    - `chat.send`, otomatik kesme noktasından önce başlatılan model çağrılarında hızlı modu kullanmak, daha sonraki yeniden deneme, geri dönüş, araç sonucu veya devam çağrılarını ise hızlı mod olmadan başlatmak için tek turluk `fastMode: "auto"` değerini kabul eder. Kesme noktası varsayılan olarak 60 saniyedir (`DEFAULT_FAST_MODE_AUTO_ON_SECONDS`) ve model başına `agents.defaults.models["<provider>/<model>"].params.fastAutoOnSeconds` ile yapılandırılabilir. Bir `chat.send` çağıranı, bu istek için kesme noktasını geçersiz kılmak üzere tek turluk `fastAutoOnSeconds` iletebilir. Depolanan kuyruk modunu yalnızca bu istek için geçersiz kılmak üzere `queueMode` (`steer`, `followup`, `collect` veya `interrupt`) iletin; açık Control UI yönlendirme eylemleri `queueMode: "steer"` kullanır. Etkileşimli istemciler, görüntüledikleri etkin transkript dalı yaprağıyla birlikte `expectedLeafEntryId` veya yetkili bir boş transkript için `null` iletebilir; başka bir istemci önce dalları değiştirdiyse Gateway, gönderimi `details.reason: "active-leaf-changed"` ile reddeder.

  </Accordion>

  <Accordion title="Cihaz eşleştirme ve cihaz belirteçleri">
    - `device.pair.list`, bekleyen ve onaylanmış eşleştirilmiş cihazları döndürür.
    - `device.pair.setupCode`, bir mobil kurulum kodu ve varsayılan olarak PNG QR veri URL'si oluşturur. `operator.admin` gerektirir ve kasıtlı olarak duyurulan keşif bilgilerinden çıkarılmıştır. Sonuç; `setupCode`, isteğe bağlı `qrDataUrl`, `gatewayUrl`, gizli olmayan `auth` etiketi ve `urlSource` alanlarını içerir.
    - `device.pair.approve`, `device.pair.reject` ve `device.pair.remove`, cihaz eşleştirme kayıtlarını yönetir.
    - `device.pair.rename`, istemcinin bildirdiği görüntüleme adına tercih edilen ve cihaz onarımından veya yeniden onaylanmasından sonra da korunan bir operatör etiketi (`{ deviceId, label }`) atar.
    - `device.token.rotate`, eşleştirilmiş bir cihaz belirtecini onaylanmış rolü ve çağıran kapsamı sınırları içinde yeniler.
    - `device.token.revoke`, eşleştirilmiş bir cihaz belirtecini onaylanmış rolü ve çağıran kapsamı sınırları içinde iptal eder.

    Kurulum kodu, kısa ömürlü bir önyükleme kimlik bilgisi içerir. İstemciler bunu
    eşleştirme akışının ötesinde günlüğe kaydetmemeli veya kalıcı hâle getirmemelidir.

  </Accordion>

  <Accordion title="Node eşleştirme, çağırma ve bekleyen işler">
    - `node.pair.list`, `node.pair.approve`, `node.pair.reject` ve `node.pair.remove`, Node yetenek onaylarını kapsar. `node.pair.request` ve `node.pair.verify`, bağımsız Node eşleştirme deposuyla birlikte 2026.7 sürümünde kaldırılmıştır; bekleyen istekler Node bağlantıları sırasında Gateway tarafından oluşturulur.
    - `node.list` ve `node.describe`, bilinen/bağlı Node durumunu döndürür.
    - `node.rename`, eşleştirilmiş bir Node etiketini günceller.
    - `node.invoke`, bir komutu bağlı bir Node'a iletir.
    - `node.invoke.result`, bir çağırma isteğinin sonucunu döndürür.
    - `mcp.tools.call.v1`, yapılandırılmış, Node'a yerel bir MCP aracını çağırmaya yönelik başsız Node ana makinesi komutudur. `node.invoke` üzerinden taşınır, Node'un komutu bildirmesini gerektirir ve eşleştirme onayına ve `gateway.nodes.commands.deny` koşuluna tabi olmaya devam eder.
    - `node.event`, Node kaynaklı olayları Gateway'e geri taşır.
    - `node.pluginTools.update`, bağlı Node'un agent tarafından görülebilen Plugin/MCP araç tanımlayıcılarını değiştirmeye yönelik tek yayımlama yoludur; `connect` parametreleri bunları taşımaz.
    - `node.pending.pull` ve `node.pending.ack`, bağlı Node kuyruk API'leridir.
    - `node.pending.enqueue` ve `node.pending.drain`, çevrimdışı/bağlantısı kesilmiş Node'lar için kalıcı bekleyen işleri yönetir.

  </Accordion>

  <Accordion title="Onay aileleri">
    - `approval.history`, exec, plugin ve sistem aracısı istekleri için 30 gün boyunca tutulan, en yeniden en eskiye sıralanmış terminal onaylarını döndürür (kapsam `operator.approvals`). İmleçli sayfalandırmayı ve isteğe bağlı bir tür filtresini destekler; bekleyen onaylar geçmiş satırları değildir.
    - `approval.get` ve `approval.resolve`, türden bağımsız kalıcı onay yöntemleridir (kapsam `operator.approvals`). `approval.get`, kararlı bir `urlPath` ile arındırılmış bekleyen veya tutulan terminal projeksiyonunu döndürür; `approval.resolve`, standart onay kimliğini, açık bir `kind` değerini ve bir kararı kabul eder, ilk yanıtın kazandığı çözümlemeyi uygular ve her zaman kaydedilmiş standart sonucu döndürür.
    - `exec.approval.request`, `exec.approval.get`, `exec.approval.list` ve `exec.approval.resolve`, tek seferlik exec onay istekleri ile bekleyen onay aramasını/yeniden yürütmesini kapsar. Bunlar, aynı kalıcı onay kayıt defteri üzerindeki protokol sınırı bağdaştırıcılarıdır.
    - `exec.approval.waitDecision`, bekleyen tek bir exec onayını bekler ve nihai kararı (veya zaman aşımında `null`) döndürür.
    - `exec.approvals.get` ve `exec.approvals.set`, Gateway exec onay politikası anlık görüntülerini yönetir.
    - `exec.approvals.node.get` ve `exec.approvals.node.set`, Node aktarma komutları aracılığıyla Node'a yerel exec onay politikasını yönetir.
    - `plugin.approval.request`, `plugin.approval.list`, `plugin.approval.waitDecision` ve `plugin.approval.resolve`, Plugin tarafından tanımlanan onay akışlarını kapsar.

  </Accordion>

  <Accordion title="Control UI komutları">
    - `ui.command`, bir `operator.write` çağırıcısının `ui-commands` yeteneğini duyuran bağlı Control UI istemcilerine türü belirlenmiş yerleşim ve gezinme komutları göndermesine olanak tanır.
    - Komutlar; bölme panelini ayırma/kapatma/odaklama, kenar çubuğu görünürlüğü, terminal/tarayıcı paneli görünürlüğü ve sabitlemesi ile oturumlar arası gezinmeyi kapsar.
    - Protokol v1, komutları kasıtlı olarak bağlı ve yetenekli tüm Control UI istemcilerine dağıtır. Hiçbiri bağlı değilse istek, yerleşim değişmiş gibi davranmak yerine `UNAVAILABLE` ile başarısız olur.

  </Accordion>

  <Accordion title="Otomasyon, Skills ve araçlar">
    - Otomasyon: `wake`, hemen veya sonraki Heartbeat'te gerçekleştirilecek bir uyandırma metni eklemesi zamanlar; `cron.get`, `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`, `cron.run`, `cron.runs` zamanlanmış işleri yönetir.
    - `cron.run`, manuel çalıştırmalar için kuyruğa ekleme tarzı bir RPC olarak kalır. Tamamlanma semantiğine ihtiyaç duyan istemciler, döndürülen `runId` değerini okumalı ve `cron.runs` için yoklama yapmalıdır.
    - `cron.runs`, istemcilerin aynı işe ait diğer geçmiş girdileriyle yarışmadan kuyruğa alınmış tek bir manuel çalıştırmayı takip edebilmesi için isteğe bağlı, boş olmayan bir `runId` filtresini kabul eder.
    - Skills ve araçlar: `commands.list`, `skills.*`, `tools.catalog`, `tools.effective`, `tools.invoke`. Aşağıdaki [Operatör yardımcı yöntemleri](#operator-helper-methods) bölümüne bakın.

  </Accordion>
</AccordionGroup>

### Yaygın olay aileleri

- `chat`: `chat.inject` gibi UI sohbet güncellemeleri ve yalnızca döküme ait diğer sohbet
  olayları. Protokol v4'te fark yükleri `deltaText` taşır; `message`
  birikimli asistan anlık görüntüsü olarak kalır. Önek olmayan değiştirmeler
  `replace=true` değerini ayarlar ve değiştirme metni olarak `deltaText` kullanır.
- `session.message`, `session.operation`, `session.tool`: abone olunan bir oturum için döküm, sürmekte olan
  oturum işlemi ve olay akışı güncellemeleri.
- `session.approval`: açıkça katılım sağlanan tam oturum abonesi için arındırılmış bekleyen ve terminal
  onay gerçeği. Alt onaylar kalıcı üst öğe kitlesini kullanır; olaylar
  dökümleri hiçbir zaman değiştirmez veya aracıları uyandırmaz.
- `sessions.changed`: oturum dizini veya meta verileri değişti.
- `presence`: sistem mevcudiyeti anlık görüntüsü güncellemeleri.
- `tick`: periyodik bağlantıyı sürdürme/canlılık olayı.
- `health`: Gateway sağlık durumu anlık görüntüsü güncellemesi.
- `heartbeat`: Heartbeat olay akışı güncellemesi.
- `cron`: Cron çalıştırma/iş değişikliği olayı.
- `shutdown`: Gateway kapanış bildirimi.
- `node.pair.requested` / `node.pair.resolved`: Node eşleştirme yaşam döngüsü.
- `node.invoke.request`: Node çağırma isteği yayını.
- `device.pair.requested` / `device.pair.resolved`: eşleştirilmiş cihaz yaşam döngüsü.
- `voicewake.changed`: uyandırma sözcüğü tetikleyici yapılandırması değişti.
- `config.changed`: bir yapılandırma yazımı kalıcılaştırıldı (yük; yapılandırma yolunu,
  yeni anlık görüntü karmasını ve bir zaman damgasını taşır — yapılandırma içeriğini asla taşımaz). Operatör okuma
  kapsamındadır; istemciler `config.get` aracılığıyla yeniler.
- `exec.approval.requested` / `exec.approval.resolved`: exec onayı
  yaşam döngüsü.
- `plugin.approval.requested` / `plugin.approval.resolved`: Plugin onayı
  yaşam döngüsü.

### Node yardımcı yöntemleri

Node'lar, otomatik izin kontrolleri için geçerli Skills yürütülebilirleri listesini
almak üzere `skills.bins` çağrısı yapabilir.

## Denetim kayıt defteri RPC'si

`audit.activity.list`, operatör istemcilerine aracı çalıştırması, araç eylemi ve katılım gerektiren mesaj yaşam döngüsü
meta verilerinin kararlı, en yeniden en eskiye sıralanmış bir görünümünü sunar.
`operator.read` gerektirir. Sorgular 30 günden eski kayıtları hariç tutar ve paylaşılan
SQLite kayıt defteri 100.000 kayıtla sınırlıdır. Süresi dolan satırlar
Gateway başlangıcında, saatlik bakım sırasında ve sonraki yazmalarda silinir. Veri modeli ve gizlilik semantiği için
[Denetim geçmişi](/tr/gateway/audit) bölümüne bakın.

- Parametreler: isteğe bağlı tam `agentId`, `sessionKey` veya `runId`; isteğe bağlı `kind`
  (`"agent_run"`, `"tool_action"` veya `"message"`); isteğe bağlı `status`
  (`"started"`, `"succeeded"`, `"failed"`, `"cancelled"`, `"timed_out"`,
  `"blocked"` veya `"unknown"`); isteğe bağlı mesaj `direction` (`"inbound"` veya
  `"outbound"`) ve tam `channel`; isteğe bağlı kapsayıcı `after` / `before`
  Unix-milisaniye sınırları; `1` ile `500` arasında isteğe bağlı `limit`; ve önceki sayfadan
  isteğe bağlı dize `cursor`.
- Sonuç: `{ "events": AuditActivityEventV1[], "nextCursor"?: string }`.

Adlandırılmış V1 sonuç birleşimi; aracı çalıştırması, araç eylemi, gelen mesaj
ve giden mesaj için ayrı şemalara sahiptir. `eventType` ayrıştırıcısı sırasıyla
`agent_run`, `tool_action`, `inbound_message` veya `outbound_message` değeridir; `kind` ve
mesaj `direction`, filtreleme ve görüntüleme için kullanılabilir durumda kalır. Her olayda
tamsayı `schemaVersion: 1` bulunur. Mesaj kimliği başvuruları tam olarak
`hmac-sha256:v1:<32 hex key id>:<64 hex digest>` biçimini kullanır; kanal-gönderen aktör
kimliği de aynı biçimi kullanır.

Tüm varyantlar `eventType`, `schemaVersion`, `eventId`, `sequence`,
`sourceSequence`, `occurredAt`, `kind`, `action`, `status`, `actor` ve
`redaction` gerektirir. Varyant alanları şunlardır:

| `eventType`        | Zorunlu alanlar                                                   | İsteğe bağlı alanlar                                                                                                                 |
| ------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `agent_run`        | `agentId`, `runId`; `kind: "agent_run"`                           | `sessionKey`, `sessionId`, `errorCode`                                                                                          |
| `tool_action`      | `agentId`, `runId`; `kind: "tool_action"`                         | `sessionKey`, `sessionId`, `toolCallId`, `toolName`, `errorCode`                                                                |
| `inbound_message`  | `direction: "inbound"`, `channel`, `conversationKind`, `outcome`  | `agentId`, `runId`, `durationMs`, `resultCount`, kimlik başvuruları, `reasonCode`, `errorCode`                                 |
| `outbound_message` | `direction: "outbound"`, `channel`, `conversationKind`, `outcome` | `agentId`, `runId`, `durationMs`, `resultCount`, kimlik başvuruları, `reasonCode`, `deliveryKind`, `failureStage`, `errorCode` |

Kapalı mesaj numaralandırmaları şunlardır:

- `conversationKind`: `direct`, `group`, `channel` veya `unknown`.
- Gelen `outcome`: `completed`, `skipped` veya `failed`; isteğe bağlı
  `reasonCode`: `duplicate`, `reply_operation_active`,
  `reply_operation_aborted`, `fast_abort`, `plugin_bound_handled`,
  `plugin_bound_unavailable`, `plugin_bound_declined`, `plugin_bound_error`,
  `before_dispatch_handled`, `acp_dispatch_completed`, `acp_dispatch_failed`,
  `acp_dispatch_empty` veya `acp_dispatch_aborted`.
- Giden `outcome`: `sent`, `suppressed`, `failed` veya `unknown`; isteğe bağlı
  `reasonCode`: `cancelled_by_message_sending_hook`,
  `cancelled_by_reply_payload_sending_hook`,
  `empty_after_message_sending_hook`, `empty_after_reply_payload_sending_hook`
  veya `no_visible_payload`. Platform kimliği döndürmeyen bir bağdaştırıcı,
  harici yan etki çürütülemeyeceği için `unknown` değerindedir.
- `deliveryKind`: `text`, `media` veya `other`; `failureStage`:
  `platform_send`, `queue` veya `unknown`.

Terminal alanları birbirleriyle ilişkilidir; birbirinden bağımsız olarak isteğe bağlı değildir:

| Varyant          | Terminal eşlemesi                                                                                                                                                   |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Aracı çalıştırması        | `started` için `errorCode` bulunmaz; başarı dışındaki her tamamlanmış durum, eşleşen `run_*` kodunu gerektirir.                                                                 |
| Araç eylemi      | `started` ve başarılı durumda `errorCode` bulunmaz; tamamlanmış diğer her durum, eşleşen `tool_*` kodunu gerektirir.                                                       |
| Gelen mesaj  | başarılı = `completed`; engellendi = `skipped`; başarısız = `failed` artı `message_processing_failed`. `reasonCode` mevcut olduğunda bu terminal ailesine ait olmalıdır. |
| Giden mesaj | başarılı = `sent`; engellendi = `suppressed` artı `reasonCode`; başarısız = `failed` artı `errorCode` ve `failureStage`; bilinmiyor = `unknown` artı `failureStage`.      |

Her etkinlik olayı; kararlı bir olay kimliği, monoton kayıt defteri sırası,
kaynak olay sırası, zaman damgası, aktör, eylem, durum, tamsayı
`schemaVersion: 1` ve `redaction: "metadata_only"` içerir. Çalıştırma ve araç kayıtları,
aracı ve çalıştırma kökenini gerektirir ve oturum kökenini içerebilir. Mesaj
kayıtları aracı ve çalıştırma kimliklerini içerebilir ancak kasıtlı olarak hiçbir zaman
`sessionKey` veya `sessionId` içermez; dolayısıyla `sessionKey` sorgu filtresi
yalnızca çalıştırma ve araç satırlarına uygulanır. Araç olayları, araç çağrısı kimliğini ve araç adını içerebilir.

İleti kayıtları `message.inbound.processed` veya
`message.outbound.finished` kullanır ve yön, kanal, konuşma türü,
normalleştirilmiş sonuç ve isteğe bağlı teslimat türü, hata aşaması, süre,
sonuç sayısı, neden kodu ile kuruluma özel anahtarlanmış
hesap/konuşma/ileti/hedef takma adlarını ekler. Bu takma adlar
korelasyona yardımcı olur ancak anonimleştirme sağlamaz: durum veritabanı bunların anahtarını
içerirken RPC ve CLI dışa aktarımları içermez. Defter; istemleri, ileti
gövdelerini, araç bağımsız değişkenlerini, araç sonuçlarını, komut çıktısını veya ham hata metnini depolamaz.
Çalıştırma/araç `sessionKey` değerleri ham korelasyon meta verileri olarak kalır ve
platform hesabı ya da eş kimliklerini içerebilir; ileti kayıtları oturum anahtarlarını içermez.

Gelen satırlarda `durationMs`, çekirdek dağıtımından terminaline kadar geçen süreyi ölçer ve
`resultCount` sonlandırılmış, kuyruğa alınmış araç, blok ve yanıt yüklerini sayar. Giden
satırlarda `durationMs`, teslimat sahipliğinden onay,
teslim edilemeyen ileti veya uzlaştırmaya kadar olan süreyi (kuyrukta bekleme süresi dâhil) kapsar ve `resultCount`
tanımlanmış fiziksel platform gönderimlerini sayar. `deliveryKind`, mevcut olduğunda,
kancalar ve işleme sonrasındaki etkin yükü açıklar; engellenmiş veya
çökme nedeniyle belirsiz satırlar bunu içermez.

Geçerli ileti kapsamı, çekirdek
dağıtımına ulaşan kabul edilmiş gelen iletileri ve çekirdekteki yinelenen/terminal sonuçlarını içerir. Giden kapsamı,
paylaşılan kalıcı teslimata ulaşan her özgün mantıksal yanıt yükü için
bir terminal satırı yazar; parçalara ayırma ve bağdaştırıcı yayılımı `resultCount` içinde birleştirilir. Kuyruğa alınmış,
yeniden denenebilir veya belirsiz gönderimler yalnızca onay, teslim edilemeyen
ileti veya uzlaştırma sonrasında kaydedilir. Bu paylaşılan
sınırları atlayan Plugin'e özel ve doğrudan gönderim yolları henüz kapsam dâhilinde değildir. Sınırlı çalışan kuyruğu azami gayretle çalışır
ve hata ya da doygunluk durumunda kayıtları düşürebilir; dolayısıyla bu yüzey
kayıpsız bir uyumluluk arşivi değildir.

Kayıt varsayılan olarak açıktır ve
[`audit.enabled`](/tr/gateway/configuration-reference#audit) tarafından denetlenir. İleti kaydı
ayrıca `audit.messages` tarafından denetlenir ve varsayılan değeri `"off"` olur. Kayıt
devre dışı bırakıldığında `audit.activity.list`, daha önce yazılan kayıtları
süreleri dolana kadar sunmaya devam eder.

Yayımlanan `audit.list` istek, sonuç ve `AuditEvent` şemaları
değişmeden kalır ve yalnızca aracı çalıştırma ve araç eylemi kayıtlarını döndürür. Yeni operatör
istemcileri, Gateway bunu duyurduğunda `audit.activity.list` çağrısını yapmalıdır. Eski
Gateway'ler, salt okuma kapsamlı bir isteğe `unknown method: audit.activity.list` veya yayımlanan sürümlerde
yetkilendirme yöntem aramasından önce gerçekleştiği için
`missing scope:
operator.admin` bildirebilir. İkincisini yalnızca yöntem duyurulmamışsa
yöntemin bulunmaması olarak değerlendirin. Ardından istemci, yalnızca filtreleri ileti türü, yön veya kanal
desteği gerektirmiyorsa `audit.list` çağrısını yeniden deneyebilir.

Metin sorguları ve sınırlı JSON dışa aktarımları için [`openclaw audit`](/tr/cli/audit) kullanın.

## Görev defteri RPC'leri

Operatör istemcileri, Gateway arka plan görev kayıtlarını
görev defteri RPC'leri (`packages/gateway-protocol/src/schema/tasks.ts`) aracılığıyla inceler ve iptal eder. Bunlar,
ham çalışma zamanı durumunu değil, temizlenmiş görev özetlerini döndürür.

- `tasks.list`, `operator.read` gerektirir.
  - Parametreler: isteğe bağlı `status` (`"queued"`, `"running"`, `"completed"`,
    `"failed"`, `"cancelled"` veya `"timed_out"`) ya da bu durumların bir dizisi,
    isteğe bağlı `agentId`, isteğe bağlı `sessionKey`, `1` ile
    `500` arasında isteğe bağlı `limit` ve isteğe bağlı dize `cursor`.
  - Sonuç: `{ "tasks": TaskSummary[], "nextCursor"?: string }`.
- `tasks.get`, `operator.read` gerektirir.
  - Parametreler: `{ "taskId": string }`.
  - Sonuç: `{ "task": TaskSummary }`.
  - Eksik görev kimlikleri, Gateway'in bulunamadı hata biçimini döndürür.
- `tasks.cancel`, `operator.write` gerektirir.
  - Parametreler: `{ "taskId": string, "reason"?: string }`.
  - Sonuç: `{ "found": boolean, "cancelled": boolean, "reason"?: string, "task"?: TaskSummary }`.
  - `found`, defterde eşleşen bir görev bulunup bulunmadığını bildirir. `cancelled`,
    çalışma zamanının iptali kabul edip etmediğini veya kaydedip kaydetmediğini bildirir.

`TaskSummary`; `id`, `status` ve şu isteğe bağlı meta verileri içerir: `kind`,
`runtime`, `title`, `agentId`, `sessionKey`, `childSessionKey`, `ownerKey`,
`runId`, `taskId`, `flowId`, `parentTaskId`, `sourceId`, zaman damgaları, ilerleme,
terminal özeti ve temizlenmiş hata metni. `agentId`, görevi yürüten aracıyı
tanımlar; `sessionKey` ve `ownerKey`, istekte bulunanın ve denetimin
bağlamını korur.

## Operatör yardımcı yöntemleri

- `commands.list` (`operator.read`), bir aracı için çalışma zamanı komut envanterini
  getirir.
  - `agentId` isteğe bağlıdır; varsayılan aracı çalışma alanını okumak için bunu belirtmeyin.
  - `scope`, birincil `name` değerinin hangi yüzeyi hedeflediğini denetler: `text`,
    baştaki `/` olmadan birincil metin komutu belirtecini döndürür; `native` ve
    varsayılan `both` yolu, mevcut olduğunda sağlayıcıya duyarlı yerel adları döndürür.
  - `textAliases`, `/model` ve `/m` gibi tam eğik çizgi takma adlarını taşır.
  - `nativeName`, mevcut olduğunda sağlayıcıya duyarlı yerel komut adını taşır.
  - `provider` isteğe bağlıdır ve yalnızca yerel adlandırmayı ve yerel Plugin
    komutlarının kullanılabilirliğini etkiler.
  - `includeArgs=false`, serileştirilmiş bağımsız değişken meta verilerini yanıttan çıkarır.
- `tools.catalog` (`operator.read`), bir aracı için çalışma zamanı araç kataloğunu
  getirir. Yanıt, gruplandırılmış araçları ve kaynak meta verilerini içerir:
  - `source`: `core` veya `plugin`
  - `pluginId`: `source="plugin"` olduğunda Plugin sahibi
  - `optional`: bir Plugin aracının isteğe bağlı olup olmadığı
- `tools.effective` (`operator.read`), bir oturum için çalışma zamanında etkin araç
  envanterini getirir.
  - `sessionKey` gereklidir.
  - Gateway, çağıran tarafından sağlanan kimlik doğrulama veya teslimat bağlamını kabul etmek
    yerine güvenilir çalışma zamanı bağlamını sunucu tarafındaki oturumdan türetir.
  - Yanıt; çekirdek, Plugin, kanal ve önceden keşfedilmiş MCP
    sunucu araçları dâhil olmak üzere etkin envanterin, sunucu tarafından türetilmiş oturum kapsamlı bir izdüşümüdür.
  - `tools.effective`, MCP için salt okunurdur: sıcak bir oturumun MCP
    kataloğunu son araç politikası üzerinden yansıtabilir ancak MCP çalışma zamanları
    oluşturmaz, aktarımlara bağlanmaz veya `tools/list` göndermez. Eşleşen sıcak katalog
    yoksa yanıt, `mcp-not-yet-connected`,
    `mcp-not-yet-listed` veya `mcp-stale-catalog` gibi bir bildirim içerebilir.
  - Etkin araç girdileri `source="core"`, `source="plugin"`,
    `source="channel"` veya `source="mcp"` kullanır.
- `tools.invoke` (`operator.write`), kullanılabilir bir aracı
  `/tools/invoke` ile aynı Gateway politika yolu üzerinden çağırır.
  - `name` gereklidir. `args`, `sessionKey`, `agentId`, `confirm` ve
    `idempotencyKey` isteğe bağlıdır.
  - Hem `sessionKey` hem de `agentId` mevcutsa çözümlenen oturum aracısı
    `agentId` ile eşleşmelidir.
  - `cron`, `gateway` ve `nodes` gibi yalnızca sahip kullanımına açık çekirdek sarmalayıcıları,
    `tools.invoke` değerinin kendisi `operator.write` olsa bile
    sahip/yönetici kimliği (`operator.admin`) gerektirir.
  - Yanıt; `ok`, `toolName`, isteğe bağlı
    `output` ve türü belirlenmiş `error` alanlarını içeren, SDK'ya yönelik bir zarftır. Onay veya politika retleri,
    Gateway araç politikası işlem hattını atlamak yerine yük içinde
    `ok:false` döndürür.
- `skills.status` (`operator.read`), bir aracı için görünür Skills envanterini
  getirir.
  - `agentId` isteğe bağlıdır; varsayılan aracı çalışma alanını okumak için bunu belirtmeyin.
  - Yanıt; ham gizli değerleri açığa çıkarmadan uygunluk durumunu, eksik gereksinimleri, yapılandırma denetimlerini
    ve temizlenmiş kurulum seçeneklerini içerir.
- `skills.search` ve `skills.detail` (`operator.read`), ClawHub
  keşif meta verilerini döndürür.
- `skills.upload.begin`, `skills.upload.chunk` ve `skills.upload.commit`
  (`operator.admin`), özel bir Skills arşivini kurmadan önce hazırlar. Bu,
  güvenilir istemciler için ayrı bir yönetici yükleme yoludur; normal ClawHub
  Skills kurulum akışı değildir ve `skills.install.allowUploadedArchives` etkinleştirilmedikçe
  varsayılan olarak devre dışıdır.
  - `skills.upload.begin({ kind: "skill-archive", slug, sizeBytes, sha256?, force?, idempotencyKey? })`,
    bu kısa ada ve zorlama değerine bağlı bir yükleme oluşturur.
  - `skills.upload.chunk({ uploadId, offset, dataBase64 })`, baytları
    tam olarak kodu çözülmüş uzaklığa ekler.
  - `skills.upload.commit({ uploadId, sha256? })`, son boyutu ve
    SHA-256 değerini doğrular. Tamamlama yalnızca yüklemeyi kesinleştirir; Skills'i kurmaz.
  - Yüklenen Skills arşivleri, `SKILL.md` kökünü içeren zip arşivleridir. Arşivin
    iç dizin adı hiçbir zaman kurulum hedefini seçmez.
- `skills.install` (`operator.admin`) üç moda sahiptir:
  - ClawHub modu: `{ source: "clawhub", slug, version?, force? }`, bir
    Skills klasörünü varsayılan aracı çalışma alanının `skills/` dizinine kurar.
  - Yükleme modu: `{ source: "upload", uploadId, slug, force?, sha256?, timeoutMs? }`,
    tamamlanmış bir yüklemeyi varsayılan aracı çalışma alanının
    `skills/<slug>` dizinine kurar. Kısa ad ve zorlama değeri, özgün
    `skills.upload.begin` isteğiyle eşleşmelidir.
    `skills.install.allowUploadedArchives` etkinleştirilmedikçe reddedilir; bu ayar
    ClawHub kurulumlarını etkilemez.
  - Gateway yükleyici modu: `{ name, installId, timeoutMs? }`, Gateway ana makinesinde bildirilmiş bir
    `metadata.openclaw.install` eylemini çalıştırır. Eski istemciler
    hâlâ `dangerouslyForceUnsafeInstall` gönderebilir; bu alan kullanımdan kaldırılmıştır,
    yalnızca protokol uyumluluğu için kabul edilir ve yok sayılır. Operatörün sahip olduğu kurulum kararları için
    `security.installPolicy` kullanın.
- `skills.update` (`operator.admin`) iki moda sahiptir:
  - ClawHub modu, izlenen tek bir kısa adı veya varsayılan aracı çalışma alanındaki
    tüm izlenen ClawHub kurulumlarını günceller.
  - Yapılandırma modu, `enabled`,
    `apiKey` ve `env` gibi `skills.entries.<skillKey>` değerlerine yama uygular.

### `models.list` görünümleri

`models.list`, isteğe bağlı bir `view` parametresini
(`src/agents/model-catalog-visibility.ts`) kabul eder:

- Belirtilmemiş veya `"default"`: `agents.defaults.modelPolicy.allow` yapılandırılmışsa
  yanıt, `provider/*` girdileri için dinamik olarak keşfedilen modeller dâhil olmak üzere izin verilen katalogdur.
  Aksi takdirde yanıt, tam Gateway kataloğudur.
- `"configured"`: seçici boyutunda davranış. `agents.defaults.modelPolicy.allow`
  yapılandırılmışsa `provider/*` girdileri için sağlayıcı kapsamlı keşif dâhil olmak üzere yine önceliklidir.
  İzin verilenler listesi olmadan yanıt, açık
  `models.providers.<provider>.models` girdilerini kullanır ve yalnızca yapılandırılmış model satırı yoksa tam
  kataloğa geri döner.
- `"provider-config"`: seçici izin verilenler listelerinden bağımsız, kaynakta tanımlanmış
  `models.providers.*.models` envanteri. Satırlar genel model yeteneklerini ve
  rotaya duyarlı kullanılabilirliği içerir ancak sağlayıcı uç noktalarını, kimlik doğrulama malzemesini ve
  çalışma zamanı istek yapılandırmasını içermez.
- `"all"`: `agents.defaults.modelPolicy.allow` atlanarak tam Gateway kataloğu. Normal model seçicileri için değil,
  tanılama/keşif kullanıcı arayüzleri için kullanın.

## Yürütme onayları

- Bir exec isteği onay gerektirdiğinde Gateway şunu yayınlar:
  `exec.approval.requested`.
- Operatör istemcileri `exec.approval.resolve` çağrısını yaparak çözümler (şunu gerektirir:
  `operator.approvals`).
- `host=node` için `exec.approval.request`, `systemRunPlan` içermelidir
  (standart `argv`/`cwd`/`rawCommand`/oturum meta verileri). `systemRunPlan`
  içermeyen istekler reddedilir.
- Onaydan sonra iletilen `node.invoke system.run` çağrıları, yetkili komut/cwd/oturum bağlamı olarak bu
  standart `systemRunPlan` değerini yeniden kullanır.
- Bir çağıran, hazırlama ile onaylanan nihai `system.run` iletimi arasında
  `command`, `rawCommand`, `cwd`, `agentId` veya
  `sessionKey` değerini değiştirirse Gateway, değiştirilmiş yüke güvenmek yerine çalıştırmayı reddeder.

## Ajan teslimi için geri dönüş

- `agent` istekleri, giden teslimat istemek için `deliver=true` içerebilir.
- `bestEffortDeliver=false` (varsayılan) katı davranışı korur: çözümlenemeyen veya
  yalnızca dahili teslimat hedefleri `INVALID_REQUEST` döndürür.
- `bestEffortDeliver=true`, harici bir teslimat rotası çözümlenemediğinde
  (örneğin dahili/webchat oturumları veya belirsiz çok kanallı yapılandırmalar)
  yalnızca oturumda yürütmeye geri dönüşe izin verir.
- Nihai `agent` sonuçları, teslimat istendiğinde
  [`openclaw agent --json --deliver`](/tr/cli/agent#json-delivery-status) için belgelenen aynı
  `sent`, `suppressed`, `partial_failed` ve
  `failed` durumlarını kullanarak `result.deliveryStatus` içerebilir.

## Sürüm yönetimi

- `PROTOCOL_VERSION`, `MIN_CLIENT_PROTOCOL_VERSION`,
  `MIN_NODE_PROTOCOL_VERSION` ve `MIN_PROBE_PROTOCOL_VERSION`,
  `packages/gateway-protocol/src/version.ts` içinde bulunur.
- İstemciler `minProtocol` + `maxProtocol` gönderir. Operatör ve kullanıcı arayüzü istemcileri,
  bu aralıkta geçerli protokolü içermelidir; mevcut istemciler ve sunucular
  v4 protokolünü çalıştırır.
- Hem `role: "node"` hem de `client.mode: "node"` kullanan kimliği doğrulanmış istemciler,
  N-1 Node protokolünü (şu anda v3) kullanabilir. Hafif yeniden başlatma sondaları
  aynı N-1 aralığını kullanır. Cihaz kimlik doğrulaması, eşleştirme, kapsamlar,
  komut politikası ve exec onayları bu uyumluluk aralığından etkilenmez. Plugin
  tarafından yönetilen Node yetenekleri ve komutları, barındırılan yüzeyleri
  N-1 sözleşmesinin parçası olmadığından Node geçerli protokole yükseltilene kadar
  sunulmaz.
- Şemalar ve modeller TypeBox tanımlarından oluşturulur:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

### İstemci sabitleri

Referans istemci uygulaması `packages/gateway-client/src/` içinde bulunur
(OpenClaw bunu ince `src/gateway/client.ts` cephesi üzerinden sarmalar). Bu
varsayılanlar v4 protokolü genelinde kararlıdır ve üçüncü taraf istemciler için
beklenen temel değerlerdir.

| Sabit                                     | Varsayılan                                            | Kaynak                                                                                                                    |
| ----------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `PROTOCOL_VERSION`                        | `4`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_CLIENT_PROTOCOL_VERSION`             | `4`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_NODE_PROTOCOL_VERSION`               | `3`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_PROBE_PROTOCOL_VERSION`              | `3`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| İstek zaman aşımı (RPC başına)            | `30_000` ms                                           | `packages/gateway-client/src/client.ts` (`requestTimeoutMs`)                                                              |
| Ön kimlik doğrulama / bağlantı sınaması zaman aşımı | `15_000` ms                                           | `packages/gateway-client/src/timeouts.ts` (`OPENCLAW_HANDSHAKE_TIMEOUT_MS` ortam değişkeni eşleştirilmiş sunucu/istemci bütçesini artırabilir) |
| İlk yeniden bağlanma geri çekilmesi       | `1_000` ms                                            | `packages/gateway-client/src/client.ts` (`GATEWAY_RECONNECT_POLICY`)                                                      |
| Azami yeniden bağlanma geri çekilmesi     | `30_000` ms                                           | `packages/gateway-client/src/client.ts` (`GATEWAY_RECONNECT_POLICY`)                                                      |
| Cihaz belirteci kapanışından sonraki hızlı yeniden deneme sınırlaması | `250` ms                                              | `packages/gateway-client/src/client.ts`                                                                                   |
| `terminate()` öncesi zorla durdurma ek süresi | `250` ms                                              | `FORCE_STOP_TERMINATE_GRACE_MS`                                                                                           |
| `stopAndWait()` varsayılan zaman aşımı | `1_000` ms                                            | `STOP_AND_WAIT_TIMEOUT_MS`                                                                                                |
| Varsayılan tik aralığı (`hello-ok` öncesi) | `30_000` ms                                           | `packages/gateway-client/src/client.ts`                                                                                   |
| Tik zaman aşımı nedeniyle kapanma         | sessizlik `tickIntervalMs * 2` değerini aştığında kod `4000` | `packages/gateway-client/src/client.ts`                                                                                   |
| `MAX_PAYLOAD_BYTES`                       | `25 * 1024 * 1024` (25 MB)                            | `src/gateway/server-constants.ts`                                                                                         |

Sunucu, etkin `policy.tickIntervalMs`, `policy.maxPayload` ve
`policy.maxBufferedBytes` değerlerini `hello-ok` içinde bildirir; istemciler
el sıkışma öncesi varsayılanlar yerine bu değerlere uymalıdır.

Referans istemci, bekleyen her isteğin bir son tarihi olduğunda sonlu isteklerin
yapılandırılmış son tarihlerini kendilerinin yönetmesine izin verir. Sonlu bir
`timeoutMs` olmadan yapılan `expectFinal` isteği, `timeoutMs: null`
içeren herhangi bir istek veya sonlu ve sınırsız isteklerin bir karışımı, tik
gözetleyicisini etkin tutar. Gelen olaylar ve yanıtlar tik zaman aşımı eşiğinden
daha uzun süre sessiz kalırsa istemci soketi `4000` koduyla kapatır,
bekleyen tüm istekleri reddeder ve yeniden bağlanır. Reddedilen istekleri yeniden
bağlandıktan sonra tekrar yürütmez.

## Kimlik doğrulama

- Paylaşılan gizli anahtarlı Gateway kimlik doğrulaması, yapılandırılmış
  `gateway.auth.mode` değerine (`"none" | "token" | "password" | "trusted-proxy"`) bağlı olarak `connect.params.auth.token` veya
  `connect.params.auth.password` kullanır.
- Tailscale Serve (`gateway.auth.allowTailscale: true`) veya loopback olmayan
  `gateway.auth.mode: "trusted-proxy"` gibi kimlik taşıyan modlar, bağlantı kimlik doğrulaması
  denetimini `connect.params.auth.*` yerine istek üstbilgilerinden karşılar.
- Özel giriş `gateway.auth.mode: "none"`, paylaşılan gizli anahtarlı bağlantı kimlik
  doğrulamasını tamamen atlar; bu modu genel/güvenilmeyen girişlerde kullanıma açmayın.
- Eşleştirmeden sonra Gateway, bağlantı rolü + kapsamlarla sınırlandırılmış
  ve `hello-ok.auth.deviceToken` içinde döndürülen bir cihaz belirteci verir. İstemciler,
  her başarılı bağlantıdan sonra bunu kalıcı olarak saklamalıdır.
- Saklanan cihaz belirteciyle yeniden bağlanırken, bu belirteç için saklanan
  onaylı kapsam kümesi de yeniden kullanılmalıdır. Bu, daha önce verilmiş
  okuma/yoklama/durum erişimini korur ve yeniden bağlantıların sessizce yalnızca
  yöneticiye özgü daha dar bir örtük kapsama indirgenmesini önler.
- İstemci tarafı bağlantı kimlik doğrulaması oluşturma işlemi
  (`packages/gateway-client/src/client.ts` içindeki `selectConnectAuth`):
  - `auth.password` bağımsızdır ve ayarlandığında her zaman iletilir.
  - `auth.token` şu öncelik sırasıyla doldurulur: önce açıkça belirtilen
    paylaşılan belirteç, ardından açıkça belirtilen `deviceToken`, son olarak
    cihaz başına saklanan belirteç (`deviceId` + `role` ile anahtarlanır).
  - `auth.bootstrapToken` yalnızca yukarıdakilerin hiçbiri
    `auth.token` değerini çözümlemediğinde gönderilir. Paylaşılan bir belirteç
    veya çözümlemiş herhangi bir cihaz belirteci bunu engeller.
  - Tek seferlik `AUTH_TOKEN_MISMATCH` yeniden denemesinde saklanan bir cihaz
    belirtecinin otomatik olarak yükseltilmesi yalnızca güvenilir uç noktalarla
    sınırlandırılmıştır: loopback veya sabitlenmiş bir `tlsFingerprint` ile
    `wss://`. Sabitleme olmadan genel `wss://` uygun değildir.
- Yerleşik kurulum kodu önyüklemesi, güvenilir mobil aktarım için birincil
  Node `hello-ok.auth.deviceToken` değerini ve `hello-ok.auth.deviceTokens` içinde sınırlandırılmış
  bir operatör belirtecini döndürür. Operatör belirteci, yerel Talk yapılandırma
  okumaları için `operator.talk.secrets` kapsamını içerir ancak eşleştirme değişikliği
  kapsamlarını ve `operator.admin` kapsamını içermez.
- Temel olmayan bir kurulum kodu önyüklemesi onay beklerken,
  `PAIRING_REQUIRED` ayrıntıları `recommendedNextStep: "wait_then_retry"`, `retryable: true` ve
  `pauseReconnect: false` değerlerini içerir. İstek onaylanana veya belirteç geçersiz
  hâle gelene kadar aynı önyükleme belirteciyle yeniden bağlanmayı sürdürün.
- `hello-ok.auth.deviceTokens` değerini yalnızca bağlantı, `wss://` veya
  loopback/yerel eşleştirme gibi güvenilir bir aktarımda önyükleme kimlik
  doğrulamasını kullandıysa kalıcı olarak saklayın.
- Bir istemci açıkça `deviceToken` veya `scopes` sağlarsa,
  çağrıyı yapanın istediği bu kapsam kümesi belirleyici olmaya devam eder;
  önbelleğe alınmış kapsamlar yalnızca istemci, cihaz başına saklanan belirteci
  yeniden kullanırken yeniden kullanılır.
- Cihaz belirteçleri `device.token.rotate` ve `device.token.revoke` aracılığıyla
  döndürülebilir/iptal edilebilir (`operator.pairing` gerektirir). Bir Node veya
  operatör dışındaki başka bir rolün döndürülmesi ya da iptal edilmesi ayrıca
  `operator.admin` gerektirir.
- `device.token.rotate`, döndürme meta verilerini döndürür. Yedek taşıyıcı
  belirteci yalnızca aynı cihazdan gelen ve hâlihazırda bu cihaz belirteciyle
  kimliği doğrulanmış çağrılarda yineler; böylece yalnızca belirteç kullanan
  istemciler yeniden bağlanmadan önce yedek belirteçlerini kalıcı olarak
  saklayabilir. Paylaşılan/yönetici döndürmeleri taşıyıcı belirteci yinelemez.
- Belirteç verme, döndürme ve iptal etme işlemleri, ilgili cihazın eşleştirme
  girdisinde kayıtlı onaylanmış rol kümesiyle sınırlı kalır; belirteç değişikliği,
  eşleştirme onayının hiç vermediği bir cihaz rolünü genişletemez veya hedefleyemez.
- Eşleştirilmiş cihaz belirteci oturumlarında, çağrıyı yapan ayrıca
  `operator.admin` kapsamına sahip değilse cihaz yönetimi kendi cihazıyla
  sınırlıdır: yönetici olmayan çağrıcılar yalnızca kendi cihaz girdilerindeki
  operatör belirtecini yönetebilir. Node ve operatör dışındaki diğer belirteçlerin
  yönetimi, çağrıyı yapanın kendi cihazında bile yalnızca yöneticilere açıktır.
- `device.token.rotate` ve `device.token.revoke`, hedef operatör belirtecinin kapsam
  kümesini çağrıyı yapanın mevcut oturum kapsamlarına göre de denetler. Yönetici
  olmayan çağrıcılar, hâlihazırda sahip olduklarından daha geniş bir operatör
  belirtecini döndüremez veya iptal edemez.
- Kimlik doğrulama hataları, `error.details.code` ile birlikte kurtarma
  ipuçlarını içerir:
  - `error.details.canRetryWithDeviceToken` (boole)
  - `error.details.recommendedNextStep`: `retry_with_device_token`, `update_auth_configuration`,
    `update_auth_credentials`, `wait_then_retry`, `review_auth_configuration` değerlerinden biri
    (`packages/gateway-protocol/src/connect-error-details.ts`).
- `AUTH_TOKEN_MISMATCH` için istemci davranışı:
  - Güvenilir istemciler, önbelleğe alınmış cihaz başına belirteçle
    sınırlandırılmış tek bir yeniden deneme girişiminde bulunabilir.
  - Bu yeniden deneme başarısız olursa otomatik yeniden bağlantı döngülerini
    durdurun ve operatör eylemi yönergelerini gösterin.
- `AUTH_SCOPE_MISMATCH`, cihaz belirtecinin tanındığı ancak istenen rolü/kapsamları
  karşılamadığı anlamına gelir. Bunu hatalı bir belirteç olarak sunmayın;
  operatörden yeniden eşleştirme yapmasını veya daha dar/geniş kapsam sözleşmesini
  onaylamasını isteyin.

## Cihaz kimliği ve eşleştirme

- Node'lar, bir anahtar çifti parmak izinden türetilen kararlı bir cihaz kimliği
  (`device.id`) içermelidir.
- Gateway'ler cihaz + rol başına belirteç verir.
- Yerel otomatik onay etkinleştirilmedikçe yeni cihaz kimlikleri için
  eşleştirme onayı gerekir.
- Eşleştirme otomatik onayı, doğrudan yerel loopback bağlantılarını temel alır.
- OpenClaw ayrıca güvenilir, paylaşılan gizli anahtarlı yardımcı akışlar için
  dar kapsamlı bir arka uç/kapsayıcı içi kendi kendine bağlantı yoluna sahiptir.
- Aynı ana makinedeki tailnet veya LAN bağlantıları yine de eşleştirme açısından
  uzak kabul edilir ve onay gerektirir.
- WS istemcileri normalde `connect` sırasında `device`
  kimliğini içerir (operatör + Node). Cihazsız operatör için tek istisnalar,
  açıkça belirtilmiş güven yollarıdır:
  - başarılı `gateway.auth.mode: "trusted-proxy"` operatör Control UI kimlik doğrulaması.
  - ayrılmış dahili yardımcı yoldaki doğrudan loopback `gateway-client`
    arka uç RPC'leri.
- Cihaz kimliğinin atlanmasının kapsam sonuçları vardır. Açıkça belirtilmiş bir
  güven yolu üzerinden cihazsız operatör bağlantısına izin verildiğinde OpenClaw,
  söz konusu yolun adlandırılmış bir kapsam koruma istisnası olmadığı sürece,
  istemcinin kendi bildirdiği kapsamları yine boş kümeye temizler. Kapsamla
  sınırlandırılmış yöntemler daha sonra `missing scope` ile başarısız olur.
- Ayrılmış doğrudan loopback `gateway-client` arka uç yardımcı yolu, kapsamları
  yalnızca dahili yerel kontrol düzlemi RPC'leri için korur; özel arka uç kimlikleri
  bu istisnadan yararlanmaz.
- Tüm bağlantılar, sunucunun sağladığı `connect.challenge` tek kullanımlık
  değerini imzalamalıdır.

### Cihaz kimlik doğrulaması geçiş tanılamaları

Hâlâ doğrulama isteği öncesi imzalama davranışını kullanan eski istemciler için
`connect`, kararlı bir `error.details.reason` ile `error.details.code` altında
`DEVICE_AUTH_*` ayrıntı kodlarını döndürür.

Yaygın geçiş hataları:

| İleti                       | details.code                     | details.reason           | Anlamı                                             |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | İstemci `device.nonce` değerini atladı (veya boş gönderdi). |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | İstemci eski/yanlış bir tek kullanımlık değerle imzaladı. |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | İmza yükü v2 yüküyle eşleşmiyor.                   |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | İmzalanan zaman damgası izin verilen sapmanın dışında. |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id`, genel anahtar parmak iziyle eşleşmiyor. |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | Genel anahtar biçimi/standartlaştırması başarısız oldu. |

Geçiş hedefi:

- Her zaman `connect.challenge` değerini bekleyin.
- Sunucu tek kullanımlık değerini içeren v2 yükünü imzalayın.
- Aynı tek kullanımlık değeri `connect.params.device.nonce` içinde gönderin.
- Tercih edilen imza yükü `v3` değeridir
  (`packages/gateway-client/src/device-auth.ts` içindeki `buildDeviceAuthPayloadV3`); bu yük, cihaz/istemci/rol/
  kapsamlar/belirteç/tek kullanımlık değer alanlarına ek olarak
  `platform` ve `deviceFamily` değerlerini bağlar.
- Eski `v2` imzaları uyumluluk amacıyla kabul edilmeye devam
  eder ancak eşleştirilmiş cihaz meta verisi sabitlemesi, yeniden bağlantıda
  komut politikasını denetlemeyi sürdürür.

## TLS ve sabitleme

- WS bağlantıları için TLS desteklenir (`gateway.tls` yapılandırması).
- İstemciler isteğe bağlı olarak Gateway sertifikası parmak izini
  `gateway.remote.tlsFingerprint` veya CLI `--tls-fingerprint` aracılığıyla sabitleyebilir.

## Kapsam

Bu protokol; durum, kanallar, modeller, sohbet, ajan, oturumlar, Node'lar,
onaylar ve daha fazlası dâhil olmak üzere Gateway API'sinin tamamını kullanıma
açar. Kesin yüzey, `packages/gateway-protocol/src/schema.ts` üzerinden yeniden dışa aktarılan TypeBox
şemalarıyla tanımlanır.

## İlgili

- [Gateway istemcisi oluşturma](https://docs.openclaw.ai/gateway/clients)
- [OpenClaw'ı gömme](https://docs.openclaw.ai/gateway/embedding)
- [Köprü protokolü](/tr/gateway/bridge-protocol)
- [Gateway operasyon kılavuzu](/tr/gateway)
