---
read_when:
    - OpenClaw deposu dışında bir operatör, gösterge paneli veya WebChat istemcisi oluşturma
    - Gateway yeniden bağlantısını, geçmişini, onaylarını veya cihaz eşleştirmesini uygulama
    - Yeni bir Gateway kablo protokolü sürümü için üçüncü taraf istemciyi güncelleme
summary: Gateway WebSocket protokolü için üçüncü taraf bir operatör veya WebChat istemcisi oluşturun
title: Gateway istemcisi oluşturma
x-i18n:
    generated_at: "2026-07-26T23:19:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fa24b196ff1fa28fb3b64d49ac25597f22cf1945aea56029e78e4375f1bdddb7
    source_path: gateway/clients.md
    workflow: 16
---

Yayınlanmış Gateway paketlerini kullanarak operatör panoları, WebChat istemcileri
ve diğer üçüncü taraf uygulamaları oluşturun. Bu kılavuz, istemci yaşam döngüsünü
kablo protokolü sözleşmesi bağlamında ele alır: kimlik doğrulama, yetenekler, yeniden bağlantı sonrası kurtarma, geçmiş,
abonelikler ve sürüm yükseltmeleri.

Çerçeve biçimleri, el sıkışma, hatalar ve eksiksiz yöntem yüzeyi için
[Gateway protokolü belirtimini](https://docs.openclaw.ai/gateway/protocol) okuyun.

## Paketleri yükleme

```bash
npm install @openclaw/gateway-client @openclaw/gateway-protocol
```

<Note>
Bu paketler OpenClaw sürüm serileriyle birlikte dağıtılır. İlk kullanıma sunma sırasında, paketleri içeren ilk OpenClaw sürümü yayımlanana kadar npm
`E404` döndürebilir;
bunları yalnızca aşağıdaki kayıt sayfaları erişilebilir hâle geldikten sonra yükleyin.
</Note>

- [`@openclaw/gateway-protocol`](https://www.npmjs.com/package/@openclaw/gateway-protocol)
  şemalar, çalışma zamanı doğrulayıcıları, TypeScript türleri, istemci kimliği ve
  yetenek kayıtları, yapılandırılmış hata okuyucuları ve protokol sürümü sabitleri
  sağlar. npm tarball paketi ayrıca oluşturulmuş, makine tarafından okunabilir
  [`protocol.schema.json`](https://unpkg.com/@openclaw/gateway-protocol/protocol.schema.json)
  sözleşmesini içerir.
- [`@openclaw/gateway-client`](https://www.npmjs.com/package/@openclaw/gateway-client)
  başvuru bağlantı uygulamasıdır. Node istemcisi için paket kökünü; tarayıcı açısından güvenli protokol,
  cihaz kimlik doğrulaması ve yeniden bağlantı yardımcıları için `@openclaw/gateway-client/browser` öğesini
  içe aktarın.

Node giriş noktası kendi WebSocket aktarımını yönetir. Tarayıcı ana makinesi, cihaz kimliği ve
cihaz belirteci için kalıcı depolama ve imzalama geri çağırmalarına ek olarak bir WebSocket
bağdaştırıcısı sağlar.

## Kapsamları seçme ve cihazı eşleştirme

Onay istemlerini de oluşturan tam etkileşimli bir sohbet istemcisi,
şu kapsamlarla `role: "operator"` istemelidir:

| Kapsam               | Kullanım amacı                                                                            |
| -------------------- | ----------------------------------------------------------------------------------------- |
| `operator.read`      | `chat.history`, `sessions.list`, `sessions.subscribe`, model durumu ve salt okunur olaylar |
| `operator.write`     | `chat.send` ve sıradan oturum değişiklikleri                                              |
| `operator.approvals` | Çalıştırma veya plugin onaylarını listeleme, görüntüleme ve çözümleme                      |

Yalnızca istemci etkileşimli soruları işliyorsa `operator.questions`,
yalnızca eşleştirilmiş cihazları veya Node öğelerini yönetiyorsa `operator.pairing` ve
yalnızca `config.patch` gibi yönetim işlemleri için `operator.admin` ekleyin.
[Operatör kapsamları başvurusu](https://docs.openclaw.ai/gateway/operator-scopes),
tüm yöntemleri ve onay zamanı kurallarını tanımlar.

`openclaw.json` öğesini elle düzenleyerek istemci başına bir bearer belirteci oluşturmayın.
Gateway'in paylaşılan önyükleme kimlik doğrulamasını `openclaw configure --section
gateway` veya `openclaw onboard --gateway-auth ...` seçenekleriyle yapılandırın, ardından istemci belirtecini cihaz
eşleştirmesinin üretmesine izin verin:

1. İstemcide bir Ed25519 cihaz kimliğini kalıcı olarak saklayın.
2. `connect.challenge` için bekleyin, sınamaya bağlı cihaz yükünü imzalayın ve
   istenen operatör rolü, kapsamlar ile önyükleme kimlik doğrulaması için paylaşılan Gateway belirtecini
   veya parolasını içeren `connect` öğesini gönderin.
3. Gateway yapılandırılmış `PAIRING_REQUIRED` ayrıntıları döndürürse istek
   kimliğini gösterin ve `error.details.recommendedNextStep` uyarınca duraklatın veya yeniden deneyin.
4. Gateway ana makinesinde isteği `openclaw devices list` ile inceleyin, ardından
   tam olarak bu güncel isteği `openclaw devices approve <requestId>` ile onaylayın.
5. Yeniden bağlanın ve `hello-ok.auth.deviceToken` öğesini üzerinde anlaşmaya varılan rol ve
   kapsamlarla kalıcı olarak saklayın. Sonraki bağlantılarda bu cihaz belirtecini kullanın.

Kapsam veya rol yükseltmeleri yeni bir bekleyen eşleştirme isteği oluşturur. Belirteç döndürme,
onaylanmış eşleştirme sözleşmesini genişletemez. Onay, döndürme ve
iptal komutları için [Cihazlar CLI](https://docs.openclaw.ai/cli/devices) bölümüne bakın.

## İstemci yeteneklerini bildirme

`connect.params.caps`, istemcinin kullanabileceği isteğe bağlı davranışı açıklar.
Yetkilendirme sağlamaz. Dize değişmezlerini çoğaltmak yerine adları
`GATEWAY_CLIENT_CAPS` öğesinden içe aktarın:

```ts
import { GATEWAY_CLIENT_CAPS } from "@openclaw/gateway-protocol/client-info";

const caps = [GATEWAY_CLIENT_CAPS.TOOL_EVENTS];
```

Geçerli kayıt `approvals`, `exec-approvals`, `inline-widgets`,
`run-tool-bindings`, `session-scoped-events`, `plugin-approvals`,
`task-suggestions`, `terminal-offset-seq`, `tool-events` ve `ui-commands` öğelerini içerir.
Yalnızca istemcinin gerçekten uyguladığı yetenekleri bildirin.

<Warning>
`tool-events`, canlı araç yürütme akışını denetler. Gateway, yalnızca
bu yeteneği bildiren bağlantıları bir çalıştırmanın yapılandırılmış araç olaylarının
alıcısı olarak kaydeder. Bu yetenek olmadan bağlantı canlı araç olayları almaz ve
el sıkışma bir hata bildirmez.
</Warning>

Yetenekle denetlenen aracı araçları, aynı bildirimin ayrı bir kullanım alanıdır. Bir
aracı aracı istemci yeteneği gerektiriyorsa Gateway, kaynak istemci gerekli tüm
yetenekleri bildirmediği sürece bu aracı çıkarır.

## Yeniden bağlantıdan sonra durumu kurtarma

Her başarılı yeniden bağlantıyı, kalıcı geçmiş ve bellekteki güncel çalıştırma durumu
üzerinde yeni bir izdüşüm olarak değerlendirin:

1. `sessions.subscribe` ve seçilen oturumun
   `sessions.messages.subscribe` aboneliğini yeniden oluşturun.
2. Seçilen `sessionKey` için `chat.history` çağrısı yapın ve yerel olarak kalıcılaştırılmış
   satırları döndürülen `messages` izdüşümüyle değiştirin.
3. `inFlightRun` mevcutsa onun `runId`, arabelleğe alınmış `text` ve isteğe bağlı
   `plan` öğelerini benimseyin. `text` boş olsa bile çalıştırmayı benimseyin.
4. `sessionInfo.hasActiveRun` ve `sessionInfo.activeRunIds` değerlerini okuyun. Tutulan bir çalıştırmanın hâlâ
   akış kullanıcı arayüzüne sahip olup olmadığına karar verirken `activeRunIds` içindeki tam
   üyeliği tercih edin. Listelenmiş kimliği olmayan doğru bir `hasActiveRun`, başka bir
   etkin çalışma zamanı izdüşümünü temsil edebilir.
5. Sonraki `agent` olaylarını `payload.runId` ve `payload.seq` değerlerine göre uzlaştırın.
   Her çalıştırma için kabul edilen en yüksek sıra değerini bağımsız olarak koruyun, daha önce
   görülmüş veya daha düşük bir sırayı yok sayın ve ileri yöndeki bir boşluğu yetkili geçmişi
   yeniden yüklemek için bir neden olarak değerlendirin.

Dış olay çerçevesi ayrıca geçerli WebSocket bağlantısındaki olayları sıralayan isteğe bağlı bir
`seq` içerir. Yeni bağlantıyla sıfırlanır. Bir `agent` olay yükünün içindeki
`seq`, çalıştırma başına atanır ve bu çalıştırmanın yaşam döngüsü, asistan, plan,
araç ve diğer akış olaylarını sıralar.

## Geçmiş meta verilerini ve kararlı sabitleyicileri kullanma

`chat.history` tarafından döndürülen satırlar bir `__openclaw` meta veri zarfı taşıyabilir:

- `id`, döküm girdisinin kimliğidir. Bunu sabitlenmiş geçmiş istekleri için kullanın,
  ancak benzersiz bir görüntüleme satırı anahtarı olarak kullanmayın.
- `seq`, pozitif döküm kaydı sırasıdır. Depolanan tek bir kayıt
  birden fazla görüntüleme satırına yansıtılabilir; bu nedenle aynı `id` ve sıraya
  sahip kardeşleri birlikte tutun.
- `kind`, sentetik satırları tanımlar. Bir Compaction sınırı
  `kind: "compaction"` kullanır ve eşleşen bir denetim noktası bu ölçümleri kaydettiğinde
  `tokensBefore` ile `tokensAfter` öğelerini içerebilir.

Yanıtın `hasMore` ve `nextOffset` değerleriyle geriye doğru sayfalayın. Sayısal
uzaklıklar geçerli döküm izdüşümünü açıklar; bu nedenle bunları sıfırlama veya Compaction
boyunca uzun ömürlü yer imleri olarak kalıcılaştırmayın. Bunun yerine `__openclaw.id` öğesini
kalıcılaştırın. Bilinen bir satırın çevresini geri yüklemek için `chat.history` öğesini
`messageId` ve onu döndüren `sessionId` ile çağırın. Gateway bu sabitleyiciyi
sıfırlama arşivi geçmişinden çözümleyebilir; sabitlenmiş yanıtlar kasıtlı olarak sayısal
sayfalama meta verilerini içermez.

## Kullanımı yoklamak yerine abone olma

İlk kataloğu `sessions.list` ile yükleyin, ardından bağlantı başına bir kez
`sessions.subscribe` çağrısı yapın. `sessions.changed` olaylarını `sessionKey` değerine göre birleştirin. Oturum değişikliği
yükleri canlı `inputTokens`, `outputTokens`, `totalTokens`,
`totalTokensFresh`, `contextTokens`, `estimatedCostUsd`, yanıt kullanım ayarları
ve etkin çalıştırma durumunu taşıyabilir.

Bazı değişiklik bildirimleri yalnızca geçersiz kılma sinyalleridir. Bir olay,
görünümünüzün ihtiyaç duyduğu satır alanlarını içermiyorsa `sessions.list` öğesini yenileyin. Canlı oturum listesini
güncel tutmak için `usage.cost` veya `sessions.usage` öğesini yoklamayın; bu yöntemleri
isteğe bağlı toplu veya ayrıntılı raporlar için ayırın.

## Çalıştırma onaylarını geriye dönük doldurma

`operator.approvals` kapsamına sahip bir istemci, `hello-ok` tamamlanır tamamlanmaz
olay dinleyicisini kurmalı, ardından bağlantıdan önceki istekleri geriye dönük doldurmak için
`exec.approval.list` çağrısı yapmalıdır. Listeyi ve canlı
`exec.approval.requested` / `exec.approval.resolved` olaylarını onay kimliğine göre
uzlaştırın; böylece liste isteğiyle yarışan bir geçiş ne kaybolur ne de yeniden ortaya çıkar.

## Protokol sürümlerini izleme

Geçerli kablo protokolü sürümü `4` değeridir. Genel operatör ve WebChat istemcileri,
`minProtocol: 4` ve `maxProtocol: 4` ile tam olarak geçerli sürüm üzerinde anlaşmalıdır.
Yalnızca kimliği doğrulanmış Node istemcileri ve hafif sondalar, şu anda
`3` ile `4` arasındaki protokolleri kapsayan N-1 kabul
aralığına sahiptir.

Protokol değişiklikleri öncelikle eklemeli yapılır. `protocol.schema.json`, çekirdek yöntemler için
`since` sürüm dönemi meta verilerini ve gerekli kapsam meta verilerini içerir; ancak bir kablo
protokolü sürümü artışı, üçüncü taraf istemciler için yine de açık bir uyumluluk bozucu olaydır. Test ettiğiniz
paket sürümlerini sabitleyin, kablo protokolü sürümü değiştiğinde istemciyi ve Gateway'i birlikte
yükseltin ve her yükseltmeden önce
[OpenClaw değişiklik günlüğünü](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)
inceleyin.

## İlgili konular

- [Gateway protokolü](https://docs.openclaw.ai/gateway/protocol)
- [OpenClaw'u gömme](https://docs.openclaw.ai/gateway/embedding)
- [Gateway RPC başvurusu](https://docs.openclaw.ai/reference/rpc)
- [Harici uygulamalar için Gateway entegrasyonları](https://docs.openclaw.ai/gateway/external-apps)
