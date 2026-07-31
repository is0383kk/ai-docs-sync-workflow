---
read_when:
    - Hangi SDK alt yolundan içe aktarma yapmanız gerektiğini bilmeniz gerekir
    - OpenClawPluginApi üzerindeki tüm kayıt yöntemleri için bir başvuru kaynağı istiyorsunuz
    - Belirli bir SDK dışa aktarımını arıyorsunuz
sidebarTitle: Plugin SDK overview
summary: İçe aktarma haritası, kayıt API'si referansı ve SDK mimarisi
title: Plugin SDK'ya genel bakış
x-i18n:
    generated_at: "2026-07-26T22:57:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4f490aa8670c57cfc1a635fb1f5d9950fa1cabdb3d45abbc2295da796edcd52e
    source_path: plugins/sdk-overview.md
    workflow: 16
---

Plugin SDK, pluginler ile çekirdek arasındaki türü belirlenmiş sözleşmedir. Bu sayfa,
**nelerin içe aktarılacağı** ve **nelerin kaydedilebileceği** konusunda başvuru kaynağıdır.

<Note>
  Bu sayfa, OpenClaw içinde `openclaw/plugin-sdk/*` kullanan plugin yazarlarına
  yöneliktir. Gateway üzerinden agent çalıştırmak isteyen harici uygulamalar,
  betikler, panolar, CI işleri ve IDE uzantıları için bunun yerine
  [harici uygulamalara yönelik Gateway entegrasyonlarını](/tr/gateway/external-apps) kullanın.
</Note>

<Tip>
Bunun yerine bir nasıl yapılır kılavuzu mu arıyorsunuz? [Plugin oluşturma](/tr/plugins/building-plugins) ile başlayın. Kanallar için [Kanal pluginleri](/tr/plugins/sdk-channel-plugins), model sağlayıcıları için [Sağlayıcı pluginleri](/tr/plugins/sdk-provider-plugins), yerel yapay zekâ CLI arka uçları için [CLI arka uç pluginleri](/tr/plugins/cli-backend-plugins), yerel agent yürütücüleri için [Agent çalıştırma ortamı pluginleri](/tr/plugins/sdk-agent-harness) ve araç ya da yaşam döngüsü kancaları için [Plugin kancaları](/tr/plugins/hooks) sayfasını kullanın.
</Tip>

## İçe aktarma kuralı

Her zaman belirli bir alt yoldan içe aktarın:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Her alt yol küçük, bağımsız bir modüldür. Bu, başlatmayı hızlı tutar ve
döngüsel bağımlılık sorunlarını önler. Kanala özgü giriş/derleme yardımcıları için
`openclaw/plugin-sdk/channel-core` tercih edin; `openclaw/plugin-sdk/core` öğesini
daha geniş kapsamlı yüzey ve `buildChannelConfigSchema` gibi
paylaşılan yardımcılar için kullanın.

Kanal yapılandırması için kanalın sahip olduğu JSON Schema'yı
`openclaw.plugin.json#channelConfigs` üzerinden yayımlayın. `plugin-sdk/channel-config-schema`
alt yolu, paylaşılan şema temel öğeleri ve genel oluşturucu içindir. OpenClaw'ın
paketle birlikte sunulan pluginleri, korunan paket içi kanal şemaları için
`plugin-sdk/bundled-channel-config-schema` kullanır. Bu paket içi şema alt yolu,
yeni pluginler için örnek alınacak bir kalıp değildir.

<Warning>
  Sağlayıcı veya kanal markalı kolaylık katmanlarını içe aktarmayın (örneğin
  `openclaw/plugin-sdk/slack`, `.../discord`, `.../signal`, `.../whatsapp`).
  Paketle birlikte sunulan pluginler, genel SDK alt yollarını kendi `api.ts` /
  `runtime-api.ts` dışa aktarma dosyalarında birleştirir; çekirdek tüketicileri
  ya bu plugin yerelindeki dışa aktarma dosyalarını kullanmalı ya da gereksinim
  gerçekten kanallar arasıysa dar kapsamlı, genel bir SDK sözleşmesi eklemelidir.

İzlenen sahip kullanımları olduğunda, paketle birlikte sunulan pluginlere yönelik
az sayıdaki yardımcı katman oluşturulan dışa aktarma eşlemesinde görünmeye devam
eder. Bunlar yalnızca paketle birlikte sunulan pluginlerin bakımı için vardır ve
yeni üçüncü taraf pluginlere önerilen içe aktarma yolları değildir.

`openclaw/plugin-sdk/discord` ve `openclaw/plugin-sdk/telegram-account` ayrıca
izlenen sahip kullanımları için kullanımdan kaldırılmış uyumluluk cepheleri
olarak korunur. Bu içe aktarma yollarını yeni pluginlere kopyalamayın; bunun
yerine enjekte edilen çalışma zamanı yardımcılarını ve genel kanal SDK alt
yollarını kullanın.
</Warning>

## Alt yol başvurusu

Plugin SDK; plugin girişi, kanal, sağlayıcı, kimlik doğrulama, çalışma zamanı,
yetenek, bellek ve paketle birlikte sunulan pluginlere ayrılmış yardımcılar gibi
alanlara göre gruplandırılmış dar kapsamlı alt yollar kümesi olarak sunulur. Gruplandırılmış
ve bağlantıları verilmiş tam katalog için [Plugin SDK alt yolları](/tr/plugins/sdk-subpaths)
sayfasına bakın.

Derleyici giriş noktası envanteri
`scripts/lib/plugin-sdk-entrypoints.json` içinde bulunur; türü belirlenmiş genel dışa aktarımlar,
`scripts/lib/plugin-sdk-private-local-only-subpaths.json` içinde listelenen dahili alt yolları hariç
tutar. Bu listedeki üretim girişleri, ayrı yayımlanan resmî pluginler için
yalnızca JavaScript ana makine çalışma zamanı dışa aktarımlarını korurken,
yalnızca teste yönelik girişler dışa aktarılmaz. Genel dışa aktarma sayısını
denetlemek için `pnpm plugin-sdk:surface` çalıştırın. Yeterince eski olan ve paketle
birlikte sunulan uzantıların üretim kodunda kullanılmayan, kullanımdan kaldırılmış
genel alt yollar `scripts/lib/plugin-sdk-deprecated-public-subpaths.json` içinde; geniş kapsamlı, kullanımdan kaldırılmış
yeniden dışa aktarma dosyaları ise `scripts/lib/plugin-sdk-deprecated-barrel-subpaths.json` içinde izlenir.

## Kayıt API'si

`register(api)` geri çağırması, şu yöntemleri içeren bir
`OpenClawPluginApi` nesnesi alır:

Bir oturum için harici ekip sohbeti yüzeyi sağlayan pluginler,
`openclaw/plugin-sdk/session-discussion` tarafından dışa aktarılan, süreç genelindeki tek
sağlayıcıyı kaydedebilir. Sağlayıcının `info({ sessionKey })` yöntemi,
bir görüşmenin kullanılamadığını, açılmaya hazır olduğunu veya zaten açık
olduğunu bildirir; `open({ sessionKey })` görüşmeyi oluşturur ya da çözümler ve
gömme URL'si ile harici URL'sini döndürür. Başka bir sağlayıcının kaydedilmesi,
geçerli sağlayıcının yerini alır.

### Yetenek kaydı

| Yöntem                                           | Kaydettiği öğe                                                                                                                         |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerProvider(...)`                      | Metin çıkarımı (LLM)                                                                                                                      |
| `api.registerWorkerProvider(...)`                | Bulut çalışanı yaşam döngüsü kiralamaları                                                                                                             |
| `api.registerModelCatalogProvider(...)`          | Metin ve medya üretimine yönelik model kataloğu satırları                                                                                          |
| `api.registerAgentHarness(...)`                  | [Deneysel](/tr/plugins/sdk-agent-harness) yerel agent yürütücüsü (Codex, Copilot)                                                         |
| `api.registerCliBackend(...)`                    | Yerel CLI çıkarım arka ucu                                                                                                               |
| `api.registerChannel(...)`                       | Mesajlaşma kanalı                                                                                                                         |
| `api.registerEmbeddingProvider(...)`             | Yeniden kullanılabilir vektör gömme sağlayıcısı                                                                                                        |
| `api.registerSpeechProvider(...)`                | Metinden sese / STT sentezi                                                                                                            |
| `api.registerRealtimeTranscriptionProvider(...)` | Akışlı gerçek zamanlı yazıya döküm                                                                                                          |
| `api.registerRealtimeVoiceProvider(...)`         | Çift yönlü gerçek zamanlı sesli oturumlar                                                                                                            |
| `api.registerMediaUnderstandingProvider(...)`    | Görüntü/ses/video analizi                                                                                                                |
| `api.registerTranscriptSourceProvider(...)`      | Canlı veya içe aktarılmış toplantı dökümü kaynağı; toplantı pluginleri `plugin-sdk/transcripts` içindeki `createMeetingTranscriptSourceProvider` öğesini kullanabilir |
| `api.registerImageGenerationProvider(...)`       | Görüntü üretimi                                                                                                                          |
| `api.registerMusicGenerationProvider(...)`       | Müzik üretimi                                                                                                                          |
| `api.registerVideoGenerationProvider(...)`       | Video üretimi                                                                                                                          |
| `api.registerWebFetchProvider(...)`              | Web getirme / kazıma sağlayıcısı                                                                                                               |
| `api.registerWebSearchProvider(...)`             | Web araması                                                                                                                                |
| `api.registerCompactionProvider(...)`            | Takılabilir döküm sıkıştırma arka ucu                                                                                                   |

Çalışan sağlayıcıları ayrıca kimliklerini `contracts.workerProviders` içinde bildirmelidir.
Çekirdek, `provision(profile, operationId)` öncesinde kalıcı niyeti saklar. Sağlayıcılar, harici tahsis öncesinde ayarları doğrular ve profilin kalıcı olarak reddedilmesi durumunda `WorkerProviderError` fırlatır. İşlem kimliği tekrarlandığında `provision` aynı kiralamayı benimsemelidir.
Çekirdek, doğrulanmış profil ayarlarını kiralamayla birlikte saklar ve bu anlık görüntüyü eşgüçlü olması gereken `destroy({ leaseId, profile })` ile `active`, `destroyed` veya `unknown` döndüren `inspect({ leaseId, profile })` öğesine sağlar. Bu, sağlayıcıların bir Gateway yeniden başlatıldıktan veya adlandırılmış profil kaldırıldıktan sonra yaşam döngüsü çağrılarını yönlendirmesini sağlar. SSH uç noktaları, `keyRef` için satır içi anahtar malzemesi yerine bir `SecretRef` kullanır ve güvenilir sağlama çıktısından gelen bir `hostKey` değerini ana makine adı veya yorum olmadan tam olarak `algorithm base64` biçiminde içerir. Çekirdek `hostKey` değerini sabitler ve ilk bağlantıdan gelen bir anahtara asla güvenmez. Dinamik bir `keyRef` oluşturan sağlayıcı `resolveSshIdentity({ leaseId, profile, keyRef })` uygulayabilir; mevcut olduğunda bu çözümleyici yetkilidir, bunu sağlamayan sağlayıcılar ise yapılandırılmış genel gizli bilgi çözümleyicisini kullanır.
Yenilenebilir kiralamalara sahip sağlayıcılar ayrıca `renew(leaseId)` uygulayabilir.
`inspect`, geçici veya sonucu belirsiz hatalarda fırlatmalıdır; yalnızca yokluk kesin olarak doğrulandığında `unknown` döndürün. Çekirdek, etkin bir yerel kaydı sahipsiz olarak işaretler veya saklanmış bir yok etme isteğinin ardından yokluğu sökme işleminin tamamlanması olarak değerlendirir.

`api.registerEmbeddingProvider(...)` ile kaydedilen gömme sağlayıcıları,
plugin bildirimindeki `contracts.embeddingProviders` içinde de listelenmelidir. Bu,
yeniden kullanılabilir vektör üretimine yönelik genel gömme yüzeyidir. Bellek
araması bu genel sağlayıcı yüzeyini kullanabilir. Daha eski
`api.registerMemoryEmbeddingProvider(...)` ve `contracts.memoryEmbeddingProviders` katmanı, mevcut belleğe özgü
sağlayıcılar taşınırken kullanımdan kaldırılmış uyumluluk olarak kalır.

Çalışma zamanında hâlâ `batchEmbed(...)` sunan belleğe özgü sağlayıcılar,
çalışma zamanları `sourceWideBatchEmbed: true` değerini açıkça ayarlamadıkça dosya başına
mevcut toplu işleme sözleşmesini kullanmaya devam eder. Bu kabul, bellek ana
makinesinin birden fazla değiştirilmiş bellek dosyasından ve etkinleştirilmiş
kaynaktan gelen parçaları, ana makinenin toplu iş sınırlarına kadar tek bir
`batchEmbed(...)` çağrısında göndermesine olanak tanır. JSONL istek dosyaları
yükleyen toplu iş bağdaştırıcıları, sağlayıcı işlerini istek sayısı sınırının
yanı sıra yükleme boyutu üst sınırından önce de bölmelidir. Sağlayıcı,
`batch.chunks` ile aynı sırada her girdi parçası için bir gömme döndürmelidir;
sağlayıcı dosya yerelindeki toplu işleri bekliyorsa veya daha büyük, kaynak
genelindeki bir işte girdi sırasını koruyamıyorsa bayrağı kullanmayın.

### Araçlar ve komutlar

Sabit araç adlarına sahip, yalnızca basit araç pluginleri için
[`defineToolPlugin`](/tr/plugins/tool-plugins) kullanın. Karma pluginler veya
tamamen dinamik araç kaydı için doğrudan `api.registerTool(...)` kullanın.

| Yöntem                                 | Kaydettiği öğe                                                                                                                        |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerTool(tool, opts?)`        | Agent aracı (zorunlu veya `{ optional: true }`)                                                                                            |
| `api.registerCommand(def)`             | Özel komut (LLM'yi atlar)                                                                                                        |
| `api.registerNodeHostCommand(command)` | `openclaw node run` tarafından işlenen komut; isteğe bağlı `agentTool` meta verileri, Node bağlıyken bunu agent tarafından görülebilen bir araç olarak sunabilir |

Agent kısa, komuta ait bir yönlendirme ipucuna ihtiyaç duyduğunda plugin komutları
`agentPromptGuidance` ayarlayabilir. Bu metni komutun kendisiyle ilgili tutun;
çekirdek istem oluşturucularına sağlayıcıya veya plugine özgü politika eklemeyin.

Yönlendirme girdileri, her istem yüzeyine uygulanan eski dizeler veya
yapılandırılmış girdiler olabilir:

```ts
agentPromptGuidance: [
  "Genel komut ipucu.",
  { text: "Bunu yalnızca ana OpenClaw isteminde göster.", surfaces: ["openclaw_main"] },
];
```

Yapılandırılmış `surfaces`; `openclaw_main`, `codex_app_server`,
`cli_backend`, `acp_backend` veya `subagent` içerebilir. `pi_main`, `openclaw_main` için kullanımdan kaldırılmış bir takma ad
olarak kalır. Kasıtlı olarak tüm yüzeylere yönelik rehberlik için `surfaces` öğesini atlayın. Boş bir
`surfaces` dizisi geçirmeyin; kapsamın yanlışlıkla kaybolmasının
genel istem metnine dönüşmemesi için bu dizi reddedilir.

Yerel Codex uygulama sunucusu geliştirici talimatları, diğer istem
yüzeylerinden daha katıdır: yalnızca açıkça `codex_app_server` kapsamına alınmış rehberlik
bu daha yüksek öncelikli kanala yükseltilir. Eski dize rehberliği ve kapsamlandırılmamış yapılandırılmış
rehberlik, uyumluluk için Codex dışı istem yüzeylerinde kullanılabilir kalır.

Node ana bilgisayarı komutları Gateway
işleminin içinde değil, bağlı Node ana bilgisayarında çalışır. `agentTool` mevcutsa Node, başarılı bir
Gateway bağlantısından sonra bir tanımlayıcı yayımlar; Gateway bunu yalnızca söz konusu
Node bağlıyken ve yalnızca tanımlayıcının `command` değeri Node'un
onaylanmış komut yüzeyindeyse ajan çalıştırmalarına sunar. Tehlikeli olmayan bir komutu
varsayılan Node komut izin verilenler listesine dahil etmek için `agentTool.defaultPlatforms` değerini ayarlayın; aksi takdirde
açıkça `gateway.nodes.commands.allow` veya bir Node çağırma ilkesi gerektirin. `agentTool.name`
sağlayıcı açısından güvenli olmalıdır: bir harfle başlamalı, yalnızca harf, rakam,
alt çizgi veya kısa çizgi kullanmalı ve 64 karakteri aşmamalıdır. MCP destekli Node araçları,
katalog ve araç arama yüzeylerinin uzak MCP sunucusu/araç kimliğini gösterebilmesi için
`agentTool.mcp` meta verilerini ayarlayabilir, ancak yürütme yine de
duyurulan Node komutu üzerinden gerçekleşir.

### Altyapı

| Yöntem                                          | Kaydettiği öğe                                                      |
| ----------------------------------------------- | ---------------------------------------------------------------------- |
| `api.registerHook(events, handler, opts?)`      | Olay kancası                                                             |
| `api.registerHttpRoute(params)`                 | Gateway HTTP uç noktası                                                  |
| `api.registerGatewayMethod(name, handler)`      | Gateway RPC yöntemi                                                     |
| `api.registerGatewayDiscoveryService(service)`  | Yerel Gateway keşif duyurucusu                                     |
| `api.registerCli(registrar, opts?)`             | CLI alt komutu                                                         |
| `api.registerNodeCliFeature(registrar, opts?)`  | `openclaw nodes` altındaki Node özelliği CLI'si                                |
| `api.registerService(service)`                  | Arka plan hizmeti                                                     |
| `api.registerInteractiveHandler(registration)`  | Etkileşimli işleyici                                                    |
| `api.registerAgentToolResultMiddleware(...)`    | Çalışma zamanı araç sonucu ara yazılımı                                         |
| `api.registerMemoryPromptSupplement(builder)`   | Eklemeli, bellekle ilişkili istem bölümü                                |
| `api.registerMemoryPromptPreparation(prepare)`  | Bellekle ilişkili bir istem bölümü için eşzamansız hazırlık                 |
| `api.registerMemoryCorpusSupplement(adapter)`   | Eklemeli bellek arama/okuma külliyatı                                     |
| `api.registerHostedMediaResolver(resolver)`     | Tarayıcı tarzı barındırılan medya URL'leri için çözümleyici                           |
| `api.registerMcpServerConnectionResolver(...)`  | Statik bir sunucu adı için istek sahibi başına MCP aktarımı (`url`/`headers`) |
| `api.registerTextTransforms(transforms)`        | Plugin'in sahip olduğu istem/ileti uyumluluk metni yeniden yazımları                |
| `api.registerConfigMigration(migrate)`          | Plugin çalışma zamanı yüklenmeden önce çalışan hafif yapılandırma geçişi           |
| `api.registerMigrationProvider(provider)`       | `openclaw migrate` için içe aktarıcı                                        |
| `api.registerAutoEnableProbe(probe)`            | Bu Plugin'i otomatik etkinleştirebilen yapılandırma yoklaması                          |
| `api.registerReload(registration)`              | Yeniden yükleme işlemesi için yeniden başlatma/anında/işlemsiz yapılandırma öneki ilkesi              |
| `api.registerNodeHostCommand(command)`          | Eşleştirilmiş Node'lara sunulan komut işleyici                                |
| `api.registerNodeInvokePolicy(policy)`          | Node tarafından çağrılan komutlar için izin verilenler listesi/onay ilkesi                    |
| `api.registerSecurityAuditCollector(collector)` | `openclaw security audit` için bulgu toplayıcı                       |

#### Onay sonrası Webhook çalışması

İşleme tamamlanmadan önce bir isteği onaylayan Webhook rotaları, bu ayrılmış
çalışmayı kendi izlenen kabul köküne taşımalıdır:

```typescript
import { runDetachedWebhookWork } from "openclaw/plugin-sdk/webhook-request-guards";

void runDetachedWebhookWork(() => processWebhookEvent(event)).catch((error) => {
  runtime.error?.(`webhook gönderimi başarısız oldu: ${String(error)}`);
});
```

HTTP isteği hâlâ kabul edilmiş durumdayken `runDetachedWebhookWork(...)` öğesini eşzamanlı olarak çağırın.
Yardımcı hemen bağımsız bir kök ayırır, ardından istek işleyicisinin önce
onayını yazabilmesi için geri çağrıyı bir sonraki mikro görevde başlatır.
Döndürülen promise, geri çağrı sonucunu benimser; reddetme işlemesi yine
çağıranların sorumluluğundadır. Bu, onay sonrası kuyruk çalışmasının kabul edilmesini sağlar ve
yeniden başlatma veya askıya alma boşaltmalarının bunu beklemesini sağlar. Dönmeden önce tüm işlemeyi
bekleyen işleyicilerin bu yardımcıya ihtiyacı yoktur.

#### İstek sahibi kapsamlı MCP bağlantıları

MCP sunucusu **kimliğini** (ad, araç filtresi) `mcp.servers`, yerel bir
Plugin'in `mcpServers` manifest alanında veya bir paket manifestinde statik tutun. İsteğe bağlı olarak, güvenilir her
ileti istek sahibinin kendi aktarımına sahip olması için bir bağlantı çözümleyicisi kaydedin:

```ts
api.registerMcpServerConnectionResolver({
  serverName: "user-email",
  resolve: async (ctx) => {
    // ctx.requesterSenderId ana bilgisayar tarafından güvenilirdir; burada asla gönderen kimliği uydurmayın.
    const token = await lookupUserToken(ctx.requesterSenderId);
    if (!token) {
      return null; // mevcut çalıştırma için bu sunucuyu atla
    }
    return {
      url: "https://mcp.example.com/email",
      headers: { Authorization: `Bearer ${token}` },
    };
  },
});
```

Sözleşme notları:

- Çözümleyici bağlamı yalnızca güvenilir ana bilgisayar kimliğini taşır (`requesterSenderId`,
  isteğe bağlı `agentAccountId` / `messageChannel`). Gelecekteki güvenilir alanlar (örneğin
  Cron/alt ajan kullanıcı bağlamı) eklemeli olarak eklenebilir.
- Bir sunucu adına tek bir Plugin sahip olur: başka bir
  Plugin'den aynı `serverName` için yinelenen bir
  `registerMcpServerConnectionResolver`, hata tanılamasıyla reddedilir (ilk kayıt kazanır); böylece
  bağlantı sahipliği hiçbir zaman Plugin yükleme sırasına bağlı olmaz.
- Araç adları, kısmi çözümlemenin
  istek sahipleri veya turlar arasında güvenli sunucu adlarını hiçbir zaman değiştirmemesi için bildirilen tam sunucu kümesinden türetilir. Çekirdek,
  farklı istek sahibi uç noktalarının aynı araç şemalarını sunduğunu
  doğrulamaz; bir çözümleyici her istek sahibini aynı mantıksal hizmete yönlendirmelidir, aksi takdirde araç
  şemaları (ve istem önbelleği kararlılığı) istek sahibi başına farklılaşır.
- Güvenilir bir `requesterSenderId` olmadan yapılan çalıştırmalar (Cron, alt ajan, Heartbeat, genel
  Gateway) hiçbir zaman istek sahibi kapsamlı sunucuları somutlaştırmaz. Paylaşılan bir
  geri dönüş bağlantısı yoktur.
- `resolve`, sunucu başına 10 saniyeyle sınırlıdır; zaman aşımı veya hata fırlatılması,
  statik MCP'nin başarısız olmasına yol açmadan söz konusu sunucuyu çalıştırmadan çıkarır.
- Çözümlenen bağlantılar istek sahibi başına en fazla 5 dakikada bir yeniden doğrulanır:
  döndürme, aktarımı yeni kimlik bilgileriyle yeniden oluşturur ve bir `null` sonucu
  bunu iptal eder (önbelleğe alınmış çalışma zamanı oturumun ortasında bile imha edilir). Bu nedenle iptal edilmiş veya
  döndürülmüş bir kimlik bilgisi 5 dakikaya kadar kullanımda kalabilir.
- Çözümlenen `headers` hiçbir zaman günlüğe kaydedilmez veya kalıcılaştırılmaz; çekirdek, kimlik bilgisi döndürmesini algılamak için yalnızca
  geçici bir bellek içi anahtarlı özet (işlem yerel HMAC) tutar ve
  çözümlenen üst bilgi/URL kimlik bilgisi değerlerini günlük/hata ayıklama yakalama
  redaksiyon kayıt defterine kaydeder.
- İstek sahibi kapsamlı sunucular MCP Uygulaması görünümleri oluşturmaz: bir görünüm,
  istek sahibinin kimliğinin doğrulandığı çalıştırmadan daha uzun yaşar ve Gateway görünüm sınırında istek sahibi
  kimliği yoktur; bu nedenle uygulama önizlemeleri bu sunucular için hata durumunda kapalı kalır. Araç sonuçları
  etkilenmez.
- Çözümleyicisi olmayan statik sunucular mevcut oturum kapsamlı yaşam döngüsünü korur.
- **Çalıştırma ortamı teslim kuralı:** istek sahibi kapsamlı sunucular hiçbir zaman çalıştırma ortamına özgü
  MCP istemci yapılandırmasına (Codex iş parçacığı `mcp_servers`, CLI `-c mcp_servers=…` veya başka herhangi bir
  oturum paylaşımlı MCP projeksiyonu) girmez. Bunun yerine çalıştırma ortamları bunları çalıştırma kapsamlı
  araçlar olarak teslim eder:
  - Gömülü çalıştırıcı: oturum MCP çalışma zamanı + paket araçları (statik + kapsamlı).
  - Codex uygulama sunucusu:
    `materializeRequesterScopedMcpToolsForHarnessRun` aracılığıyla dinamik araçlar (yalnızca kapsamlı; statik
    sunucular Codex'in yerel MCP istemcisinde kalır).
- Kapsamlı araç **belirtimleri**, söz konusu oturumdaki ilk başarılı çözümlemeden sonra oturum boyunca kararlı kalır;
  böylece paylaşılan iş parçacıklı çalıştırma ortamları (Codex), gönderenler değiştiğinde
  iş parçacıklarını döndürmez. Herhangi bir istek sahibi çözümlemeden önce kapsamlı belirtimler duyurulmaz.
- Paylaşılan iş parçacıklı bir çalıştırma ortamındaki kimliği doğrulanmamış istek sahipleri yine de duyurulan
  kapsamlı araçları görür; bunlardan birini çağırmak, söz konusu istek sahibi için temiz bir bağlı-değil araç hatası
  döndürür. OpenClaw hiçbir zaman başka bir istek sahibinin kimlik bilgilerine geri dönmez.

Bellek istemi ek oluşturucuları isteğe bağlı `agentId`,
`agentSessionKey` ve `sandboxed` bağlamını alır. Bellek külliyatı eki `search`
ve `get` çağrıları, isteğe bağlı `agentId` ve `sandboxed` bağlamını alır. Ajanın sahip olduğu
depolamaya sahip Plugin'ler, kayıt sırasında tek bir genel yolu yakalamak yerine
her çağrı için bu depolamayı çözümlemelidir. Birden çok ajanlı bir işlemde ajan kimliği gerekliyse ancak
eksikse, keyfî bir ajan seçmek yerine hata durumunda kapalı kalın.

İstem metni eşzamansız Plugin durumuna bağlı olduğunda `registerMemoryPromptPreparation(...)` kullanın.
Geri çağrı, her tam ajan isteminden önce bir kez çalışır ve
eşzamanlı bellek istemi oluşturucularıyla aynı araç, ajan, oturum ve korumalı alan bağlamını alır.
Kalıcı durumu yüklemeden önce mevcut depolama sahibi örneğini doğrulayın, ardından yalnızca
o çalıştırmaya ait satırları döndürün. OpenClaw bu satırları dondurur ve
değişmez sonucu eşzamanlı istem derlemesine verir. Kalıcılaştırma,
atomik değiştirme ve sahip kaldırma sırasında silme işlemlerini sahip olan Plugin içinde tutun; bir istem oluşturucudan
dosyaları yoklamayın veya okumayın.

Telegram etkileşimli işleyicileri, işleyici başarıyla tamamlandıktan sonra metni
Telegram'ın normal gelen ajan yolu üzerinden yönlendirmek için `{ submitText }` döndürebilir. Gelen ilkesi metni atladığında veya
işleme başarısız olduğunda OpenClaw geri çağrı düğmesini korur; böylece
engelleyici koşul değiştikten sonra kullanıcı yeniden deneyebilir. Bu sonuç alanı
Telegram'a özeldir; diğer kanallar kendi etkileşimli sonuç sözleşmelerini korur.

### İş akışı Plugin'leri için ana bilgisayar kancaları

Ana bilgisayar kancaları, yalnızca bir sağlayıcı, kanal veya araç eklemek yerine ana bilgisayar
yaşam döngüsüne katılması gereken Plugin'ler için SDK bağlantı noktalarıdır. Bunlar
genel sözleşmelerdir; Plan Modu bunları kullanabilir, ancak onay iş akışları,
çalışma alanı ilkesi geçitleri, arka plan izleyicileri, kurulum sihirbazları ve kullanıcı arayüzü eşlikçi
Plugin'leri de kullanabilir.

| Yöntem                                                                               | Sahip olduğu sözleşme                                                                                                                                           |
| ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.session.state.registerSessionExtension(...)`                                    | Gateway oturumları aracılığıyla yansıtılan, Plugin'e ait, JSON uyumlu oturum durumu                                                                             |
| `api.session.workflow.enqueueNextTurnInjection(...)`                                 | Bir oturum için sonraki ajan turuna eklenen, kalıcı ve tam olarak bir kez kullanılan bağlam                                                                             |
| `api.registerTrustedToolPolicy(...)`                                                 | Araç parametrelerini engelleyebilen veya yeniden yazabilen, manifest tarafından denetlenen güvenilir Plugin öncesi araç politikası                                                                        |
| `api.registerToolMetadata(...)`                                                      | Araç uygulamasını değiştirmeden araç kataloğu görüntüleme meta verileri                                                                                     |
| `api.registerCommand(...)`                                                           | Kapsamlı Plugin komutları; komut sonuçları `continueAgent: true` veya `suppressReply: true` değerini ayarlayabilir; Discord yerel komutları `descriptionLocalizations` desteğine sahiptir |
| `api.session.controls.registerControlUiDescriptor(...)`                              | Oturum, araç, çalıştırma, ayarlar veya sekme yüzeyleri için Control UI katkı tanımlayıcıları                                                                      |
| `api.lifecycle.registerRuntimeLifecycle(...)`                                        | Sıfırlama/silme/yeniden yükleme yollarında Plugin'e ait çalışma zamanı kaynakları için temizleme geri çağrıları                                                                          |
| `api.agent.events.registerAgentEventSubscription(...)`                               | İş akışı durumu ve izleyiciler için arındırılmış olay abonelikleri                                                                                              |
| `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`  | Sonlandırıcı çalıştırma yaşam döngüsünde temizlenen, çalıştırma başına Plugin geçici durumu                                                                                             |
| `api.session.workflow.registerSessionSchedulerJob(...)`                              | Plugin'e ait zamanlayıcı işleri için temizleme meta verileri; işleri zamanlamaz veya görev kayıtları oluşturmaz                                                            |
| `api.session.workflow.sendSessionAttachment(...)`                                    | Yalnızca paketlenmiş Plugin'ler için, ana makine aracılı dosya eklerinin etkin doğrudan giden oturum rotasına teslimi                                                            |
| `api.session.workflow.scheduleSessionTurn(...)` / `unscheduleSessionTurnsByTag(...)` | Yalnızca paketlenmiş Plugin'ler için Cron destekli zamanlanmış oturum turları ve etiket tabanlı temizleme                                                                                    |
| `api.session.controls.registerSessionAction(...)`                                    | İstemcilerin Gateway üzerinden gönderebileceği türü belirlenmiş oturum eylemleri                                                                                             |

Bir `surface: "tab"` tanımlayıcısı, Control UI'a bir kenar çubuğu sekmesi ekler. Etkin
Plugin'lerin sekme tanımlayıcıları, gateway karşılama iletisinde
(`controlUiTabs`) pano istemcilerine duyurulur; böylece sekme yalnızca Plugin etkin durumdayken görünür.
Paketlenmiş Plugin'ler sekmeleri için birinci sınıf bir pano görünümü sunabilir; diğer
Plugin'ler `path` değerini panonun korumalı bir çerçevede
oluşturduğu bir Plugin HTTP rotasına (bkz.
`api.registerHttpRoute(...)`) ayarlayabilir.
`icon` bir pano simgesi adı ipucudur, `group` kenar çubuğu bölümünü
(`control` veya `agent`) seçer, `order` Plugin sekmelerini sıralar ve `requiredScopes`
bu operatör kapsamlarına sahip olmayan bağlantılarda sekmeyi gizler:

Gateway korumalı harici bir sekme için tanımlayıcıyı `path`, aynı Plugin'e ait
bir `auth: "gateway"` HTTP rotası altında kaydedin. Kimliği doğrulanmış önyüklemeden sonra tarayıcı,
Plugin ve rota köküyle sınırlandırılmış kısa ömürlü, HttpOnly bir yetki alır; böylece
korumalı çerçeve, Gateway taşıyıcı belirtecini URL'sine veya
JavaScript'e kopyalamadan yüklenebilir. Kimliği doğrulanmış üst öğe, harici sekme
etkinken ve gezinme ya da tarayıcının sürdürülmesinden sonra sekmeyi bağlamadan önce yetkiyi yeniler. Ayrıca
bağlamadan önce yetkiyi aynı opak korumalı alandan yoklar; böylece çerezi
engelleyen tarayıcı gizlilik modları, kullanılamayan bir panelle güvenli biçimde başarısız olur.
Çerçeve yetkisi yalnızca `GET` ve `HEAD` kabul eder ve her zaman
`operator.read` taşır; `requiredScopes` sekme görünürlüğünü denetler ancak
çerez yetkisini hiçbir zaman genişletmez. Değişiklik işlemleri, açıkça Gateway kimlik doğrulamalı üst öğe veya
taşıyıcı yüzeylerinde kalır. Harici sekmeler HTTPS/Tailscale Serve veya
tarayıcının güvendiği bir geri döngü kaynağı gerektirir; LAN ana makinesindeki düz HTTP,
kimlik doğrulayamayan bir paneli bağlamak yerine
güvenli bağlam hatasını gösterir.
Üçüncü taraf çerezlerinin tamamen engellenmesi de Gateway korumalı sekmeleri kullanılamaz hâle getirir.
Tüm yerel Plugin yüzeylerinde olduğu gibi çerçeve, kurulu
Plugin'in güven sınırı içinde kalır; OpenClaw, kurulu Plugin'leri karşılıklı olarak
yalıtılmış tarayıcı güvenlik sorumluları olarak değerlendirmez.
Çerez yetkileri tarayıcının ana makine adı sınırını kullanır, bağlantı noktası sınırını değil.
Karşılıklı olarak güvenilmeyen hizmetleri farklı bağlantı noktalarında bile Gateway ana makine adında
birlikte barındırmayın.
Plugin tarafından yönetilen kimlik doğrulamayla desteklenen sekmeler, doğrudan iframe davranışlarını korur ve
bu Gateway yetkisini istemez ya da gerektirmez.

```typescript
api.session.controls.registerControlUiDescriptor({
  surface: "tab",
  id: "logbook",
  label: "Günlük",
  description: "Ekran anlık görüntülerinden oluşturulan zaman çizelgesi biçimindeki gününüz.",
  icon: "sun",
  group: "control",
  requiredScopes: ["operator.write"],
});
```

Yeni Plugin kodu için gruplandırılmış ad alanlarını kullanın:

- `api.session.state.registerSessionExtension(...)`
- `api.session.workflow.enqueueNextTurnInjection(...)`
- `api.session.workflow.registerSessionSchedulerJob(...)`
- `api.session.workflow.sendSessionAttachment(...)`
- `api.session.workflow.scheduleSessionTurn(...)`
- `api.session.workflow.unscheduleSessionTurnsByTag(...)`
- `api.session.controls.registerSessionAction(...)`
- `api.session.controls.registerControlUiDescriptor(...)`
- `api.agent.events.registerAgentEventSubscription(...)`
- `api.agent.events.emitAgentEvent(...)`
- `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`
- `api.lifecycle.registerRuntimeLifecycle(...)`

Eşdeğer düz yöntemler, mevcut Plugin'ler için kullanımdan kaldırılmış uyumluluk
takma adları olarak kullanılabilir durumda kalır. Doğrudan
`api.registerSessionExtension`, `api.enqueueNextTurnInjection`,
`api.registerControlUiDescriptor`, `api.registerRuntimeLifecycle`,
`api.registerAgentEventSubscription`, `api.emitAgentEvent`,
`api.setRunContext`, `api.getRunContext`, `api.clearRunContext`,
`api.registerSessionSchedulerJob`, `api.registerSessionAction`,
`api.sendSessionAttachment`, `api.scheduleSessionTurn` veya
`api.unscheduleSessionTurnsByTag` çağıran yeni Plugin kodu eklemeyin.

`scheduleSessionTurn(...)`, Gateway
Cron zamanlayıcısı üzerinde oturum kapsamlı bir kolaylıktır. Cron zamanlamanın sahibidir ve
tur çalıştığında arka plan görev kaydını oluşturur; Plugin SDK yalnızca hedef oturumu, Plugin'e ait
adlandırmayı ve temizlemeyi sınırlar. İşin kendisi kalıcı, çok adımlı Task Flow durumu
gerektirdiğinde zamanlanmış turun içinde `api.runtime.tasks.managedFlows` kullanın.

Sözleşmeler yetkiyi kasıtlı olarak böler:

- Harici Plugin'ler oturum uzantılarının, UI tanımlayıcılarının, komutların, araç
  meta verilerinin, sonraki tur eklemelerinin ve normal kancaların sahibi olabilir.
- Güvenilir araç politikaları sıradan `before_tool_call` kancalarından önce çalışır ve
  ana makine tarafından güvenilir kabul edilir. Önce paketlenmiş politikalar çalışır; kurulu Plugin politikaları,
  açıkça etkinleştirilmelerinin yanı sıra yerel kimliklerinin
  `contracts.trustedToolPolicies` içinde bulunmasını gerektirir ve ardından Plugin yükleme sırasına göre çalışır. Politika kimlikleri,
  kaydeden Plugin ile sınırlıdır.
- Ayrılmış komut sahipliği yalnızca paketlenmiş Plugin'lere özeldir. Harici Plugin'ler kendi
  komut adlarını veya takma adlarını kullanmalıdır.
- `allowPromptInjection=false`; `agent_turn_prepare`, `before_prompt_build`, `heartbeat_prompt_contribution`
  ve `enqueueNextTurnInjection` dâhil olmak üzere istemi değiştiren kancaları devre dışı bırakır.

Plan dışı tüketici örnekleri:

| Plugin arketipi             | Kullanılan kancalar                                                                                                                             |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Onay iş akışı            | Oturum uzantısı, komut devamı, sonraki tur eklemesi, UI tanımlayıcısı                                                            |
| Bütçe/çalışma alanı politika geçidi | Güvenilir araç politikası, araç meta verileri, oturum yansıtması                                                                                 |
| Arka plan yaşam döngüsü izleyicisi | Çalışma zamanı yaşam döngüsü temizliği, ajan olay aboneliği, oturum zamanlayıcısı sahipliği/temizliği, Heartbeat istem katkısı, UI tanımlayıcısı |
| Kurulum veya ilk katılım sihirbazı   | Oturum uzantısı, kapsamlı komutlar, Control UI tanımlayıcısı                                                                              |

<Note>
  Ayrılmış çekirdek yönetici ad alanları (`config.*`, `exec.approvals.*`, `wizard.*`,
  `update.*`), bir Plugin daha dar bir gateway yöntem kapsamı atamaya çalışsa bile her zaman
  `operator.admin` olarak kalır. Plugin'e ait yöntemler için Plugin'e özgü ön ekleri
  tercih edin.
</Note>

<Accordion title="Araç sonucu ara yazılımı ne zaman kullanılmalı">
  Paketlenmiş Plugin'ler ve eşleşen manifest sözleşmelerine sahip, açıkça etkinleştirilmiş
  kurulu Plugin'ler; çalıştırmadan sonra ve çalışma zamanı bu sonucu modele
  geri beslemeden önce bir araç sonucunu yeniden yazmaları gerektiğinde
  `api.registerAgentToolResultMiddleware(...)` kullanabilir. Bu, tokenjuice gibi eşzamansız çıktı
  indirgeyicileri için güvenilir, çalışma zamanından bağımsız bağlantı noktasıdır.

Plugin'ler hedeflenen her çalışma zamanı için `contracts.agentToolResultMiddleware` bildirmelidir;
örneğin `["openclaw", "codex"]`. Bu sözleşmeye veya açık etkinleştirmeye sahip olmayan
kurulu Plugin'ler bu ara yazılımı kaydedemez; model öncesi araç sonucu zamanlaması
gerektirmeyen işler için normal OpenClaw Plugin kancalarını kullanın. Eski
yalnızca gömülü çalıştırıcıya özel uzantı fabrikası kayıt yolu kaldırılmıştır.
</Accordion>

### Gateway keşif kaydı

`api.registerGatewayDiscoveryService(...)`, bir Plugin'in etkin
Gateway'i mDNS/Bonjour gibi yerel bir keşif aktarımında duyurmasını sağlar. OpenClaw, yerel keşif
etkinken Gateway başlatma sırasında hizmeti çağırır, geçerli Gateway bağlantı noktalarını ve gizli olmayan
TXT ipucu verilerini iletir ve Gateway kapatma sırasında döndürülen
`stop` işleyicisini çağırır.

```typescript
api.registerGatewayDiscoveryService({
  id: "my-discovery",
  async advertise(ctx) {
    const handle = await startMyAdvertiser({
      gatewayPort: ctx.gatewayPort,
      tls: ctx.gatewayTlsEnabled,
      displayName: ctx.machineDisplayName,
    });
    return { stop: () => handle.stop() };
  },
});
```

Gateway keşif Plugin'leri, duyurulan TXT değerlerini gizli bilgi veya
kimlik doğrulama olarak değerlendirmemelidir. Keşif bir yönlendirme ipucudur; güvenin sahibi hâlâ Gateway kimlik doğrulaması ve TLS sabitlemesidir.

### CLI kayıt meta verileri

`api.registerCli(registrar, opts?)` iki tür komut meta verisi kabul eder:

- `commands`: kaydedenin sahip olduğu açık komut adları
- `descriptors`: CLI yardımı, yönlendirme ve gecikmeli Plugin CLI kaydı için kullanılan ayrıştırma zamanı komut tanımlayıcıları
- `parentPath`: `["nodes"]` gibi iç içe komut grupları için isteğe bağlı üst komut yolu

Eşleştirilmiş Node özellikleri için
`api.registerNodeCliFeature(registrar, opts?)` tercih edin. Bu, `api.registerCli(..., { parentPath: ["nodes"] })` etrafında küçük bir sarmalayıcıdır ve
`openclaw nodes canvas` gibi komutları açıkça Plugin'e ait Node özellikleri hâline getirir.

Bir Plugin komutunun normal kök CLI yolunda gecikmeli yüklenmiş olarak kalmasını
istiyorsanız, bu kaydeden tarafından sunulan her üst düzey komut kökünü kapsayan
`descriptors` sağlayın.

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "Matrix hesaplarını, doğrulamayı, cihazları ve profil durumunu yönetin",
        hasSubcommands: true,
      },
    ],
  },
);
```

İç içe komutlar, çözümlenmiş üst komutu `program` olarak alır:

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerNodesCanvasCommands } = await import("./src/cli.js");
    registerNodesCanvasCommands(program);
  },
  {
    parentPath: ["nodes"],
    descriptors: [
      {
        name: "canvas",
        description: "Eşleştirilmiş bir node'dan tuval içeriği yakalayın veya işleyin",
        hasSubcommands: true,
      },
    ],
  },
);
```

`commands` öğesini tek başına yalnızca gecikmeli kök CLI kaydına ihtiyacınız olmadığında kullanın.
Bu istekli uyumluluk yolu desteklenmeye devam eder, ancak ayrıştırma zamanında gecikmeli yükleme için
tanımlayıcı destekli yer tutucular kurmaz.

### CLI arka ucu kaydı

`api.registerCliBackend(...)`, bir plugin'in `claude-cli` veya `my-cli` gibi yerel
bir yapay zekâ CLI arka ucunun varsayılan yapılandırmasını sahiplenmesine olanak tanır.

- Arka ucun `id` değeri, `my-cli/gpt-5` gibi model başvurularında sağlayıcı öneki olur.
- Arka ucun `config` değeri yetkili komut bağdaştırıcısıdır: argv, ortam,
  ayrıştırıcı, oturum, görüntü ve güvenilirlik davranışı plugin kodunda bulunur.
- Kullanıcılar arka ucu model başvuruları veya model kapsamlı `agentRuntime.id` üzerinden seçer;
  `openclaw.json` bağdaştırıcıyı yeniden yazmaz.
- Kayıtlı statik alanlar çalışma zamanından haberdar bir
  normalleştirme geçişine ihtiyaç duyduğunda `normalizeConfig` kullanın.
- OpenClaw düşünme düzeylerini yerel bir efor bayrağıyla eşlemek gibi
  CLI lehçesine ait, istek kapsamlı argv yeniden yazımları için `resolveExecutionArgs` kullanın.
  Kanca `ctx.executionMode` alır; geçici `/btw` çağrılarına
  arka uca özgü yalıtım bayrakları eklemek için `"side-question"` kullanın. Bu bayraklar,
  aksi hâlde her zaman açık olan bir CLI için yerel araçları güvenilir biçimde devre dışı
  bırakıyorsa `sideQuestionToolMode: "disabled"` değerini de bildirin.
- Arka ucun sahip olduğu başlatma ortamı veya geçici
  kimlik doğrulama/yapılandırma köprüleri için `prepareExecution` kullanın. Bunun `ctx.contextTokenBudget` değeri,
  çalıştırma için seçilen etkin token sınırıdır; böylece yerel Compaction arka uçları,
  sağlayıcıya özgü çekirdek dalları olmadan kendi eşiklerini hizalayabilir. Ayrıca arka uç
  hazırlama işleminin paketlenmiş MCP ayarlarını genişletmesi gerektiğinde çekirdek tarafından
  hazırlanmış `ctx.env` değerini alır.
- Belirli bir çalıştırma için tüm yerel araçları devre dışı bırakabilen arka uçlar
  `nativeToolMode: "selectable"` bildirebilir. Kısıtlı çağrılar, tam bir
  `ctx.toolAvailability.native` listesiyle birlikte kurallı
  `ctx.toolAvailability.openClaw` adlarını geçirir.
  `toolAvailabilityEnforcement: "execution-args"` bildirin ve sözleşmeyi
  son yeni/devam ettirme argv'sinde uygulayın ya da `"prepare-execution"` bildirin, hazırlanan
  politikada uygulayın ve `toolAvailabilityEnforced: true` döndürün. OpenClaw,
  Cron `toolsAllow` gibi çalışma zamanı sınırları için yerel araçları devre dışı bırakır ve
  bildirilen uygulama yolu eksik olduğunda kapalı durumda başarısız olur.

Uçtan uca yazım kılavuzu için
[CLI arka uç plugin'leri](/tr/plugins/cli-backend-plugins) bölümüne bakın.

### Özel yuvalar

| Yöntem                                     | Kaydettiği öğe                                                                                                                                                                                  |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Bağlam motoru (aynı anda yalnızca biri etkin). Ana makine model/sağlayıcı/mod tanılaması sağlayabildiğinde yaşam döngüsü geri çağrıları `runtimeSettings` alır; eski katı motorlar bu anahtar olmadan yeniden denenir. |
| `api.registerMemoryCapability(capability)` | Birleşik bellek yeteneği                                                                                                                                                                          |

### Kullanımdan kaldırılmış bellek gömme bağdaştırıcıları

| Yöntem                                         | Kaydettiği öğe                              |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Etkin plugin için bellek gömme bağdaştırıcısı |

- `registerMemoryCapability`, özel bellek plugin'i API'sidir.
- `registerMemoryCapability`, ana makine tarafından yönetilen dışa aktarımlar için
  `publicArtifacts.listArtifacts(...)` öğesini de sunabilir. Bildirilen bu yapıtları listeleyen yardımcı
  plugin'ler, odaklanmış bir genel tüketici API'si oluşturulana kadar korunan
  `openclaw/plugin-sdk/memory-host-core` cephesindeki `listActiveMemoryPublicArtifacts(...)` öğesini kullanmaya devam eder;
  başka bir plugin'in özel düzenine erişmemelidir.
- `MemoryFlushPlan.model`, etkin geri dönüş zincirini devralmadan temizleme turunu
  `ollama/qwen3:8b` gibi tam bir `provider/model` başvurusuna sabitleyebilir.
- `registerMemoryEmbeddingProvider` kullanımdan kaldırılmıştır. Yeni gömme sağlayıcıları
  `api.registerEmbeddingProvider(...)` ve
  `contracts.embeddingProviders` kullanmalıdır.
- Mevcut belleğe özgü sağlayıcılar geçiş dönemi boyunca çalışmaya devam eder,
  ancak plugin incelemesi bunu paketlenmemiş plugin'ler için uyumluluk borcu olarak bildirir.

### Olaylar ve yaşam döngüsü

| Yöntem                                       | Yaptığı işlem                  |
| -------------------------------------------- | ----------------------------- |
| `api.on(hookName, handler, opts?)`           | Türü belirlenmiş yaşam döngüsü kancası          |
| `api.onConversationBindingResolved(handler)` | Konuşma bağlama geri çağrısı |

Örnekler, yaygın kanca adları ve koruma semantiği için
[Plugin kancaları](/tr/plugins/hooks) bölümüne bakın.

### Kanca karar semantiği

`before_install`, operatör kurulum politikası yüzeyi değil, plugin çalışma zamanı
yaşam döngüsü kancasıdır. İzin verme/engelleme kararının CLI ve Gateway destekli
kurulum ya da güncelleme yollarını kapsaması gerektiğinde `security.installPolicy` kullanın.

- `before_tool_call`: `{ block: true }` döndürmek sonlandırıcıdır. Herhangi bir işleyici bunu ayarladığında daha düşük öncelikli işleyiciler atlanır.
- `before_tool_call`: `{ block: false }` döndürmek, geçersiz kılma olarak değil, karar verilmemiş olarak değerlendirilir (`block` öğesini atlamakla aynıdır).
- `before_install`: `{ block: true }` döndürmek sonlandırıcıdır. Herhangi bir işleyici bunu ayarladığında daha düşük öncelikli işleyiciler atlanır.
- `before_install`: `{ block: false }` döndürmek, geçersiz kılma olarak değil, karar verilmemiş olarak değerlendirilir (`block` öğesini atlamakla aynıdır).
- `reply_dispatch`: `{ handled: true, ... }` döndürmek sonlandırıcıdır. Herhangi bir işleyici gönderimi üstlendiğinde daha düşük öncelikli işleyiciler ve varsayılan model gönderim yolu atlanır.
- `message_sending`: `{ cancel: true }` döndürmek sonlandırıcıdır. Herhangi bir işleyici bunu ayarladığında daha düşük öncelikli işleyiciler atlanır.
- `message_sending`: `{ cancel: false }` döndürmek, geçersiz kılma olarak değil, karar verilmemiş olarak değerlendirilir (`cancel` öğesini atlamakla aynıdır).
- `message_received`: gelen iş parçacığı/konu yönlendirmesine ihtiyaç duyduğunuzda türü belirlenmiş `threadId` alanını kullanın. Kanala özgü ek bilgiler için `metadata` öğesini koruyun.
- `message_sending`: kanala özgü `metadata` öğesine geri dönmeden önce türü belirlenmiş `replyToId` / `threadId` yönlendirme alanlarını kullanın.
- `gateway_start`: dahili `gateway:startup` kancalarına güvenmek yerine Gateway'in sahip olduğu başlangıç durumu için `ctx.config`, `ctx.workspaceDir` ve `ctx.getCron?.()` kullanın. Cron bu noktada hâlâ yükleniyor olabilir.
- `cron_reconciled`: başlangıçtan veya zamanlayıcının yeniden yüklenmesinden sonra tam bir harici Cron projeksiyonunu yeniden oluşturun. `ctx.getCron?.()` tam olarak uzlaştırılmış zamanlayıcıyı döndürürken bu, `reason` ve `enabled: false` dâhil etkin `enabled` durumunu içerir. Kalıcı projeksiyon çalışmalarına `ctx.abortSignal` geçirin; ilgili zamanlayıcı anlık görüntüsünün yerini yenisi aldığında veya Gateway kapandığında işlem iptal edilir.
- `cron_changed`: Gateway'in sahip olduğu Cron yaşam döngüsü değişikliklerini gözlemleyin. `scheduled` ve `removed` olayları, sıralı bir fark günlüğü değil, tamamlama sonrası uzlaştırma ipuçlarıdır. Zamanlanmış bir olayın `event.nextRunAtMs` değeri, işin bir sonraki uyanma zamanı olmadığında bulunmaz; kaldırılmış bir olay ise silinen işin anlık görüntüsünü taşımaya devam eder.

Harici uyandırma zamanlayıcıları `cron_changed` olaylarını geciktirmeli veya birleştirmeli,
ardından `cron_reconciled` tarafından en son yakalanan zamanlayıcıdan tam kalıcı görünümü
yeniden okumalıdır. Zamanlayıcıyı bir `cron_changed` bağlamından devralmayın: eski bir
zamanlayıcıdan ayrılmış bir ipucu, daha sonraki bir yeniden yüklemeyle çakışabilir.

Gateway başlangıcında veya zamanlayıcı değişiminde yüklenen kalıcı durum için tam anlık
görüntü tetikleyicisi olarak `cron_reconciled` kullanın. Yalnızca plugin'i etkileyen
çalışırken yeniden yükleme işleminde bu yeniden oynatılmaz. Gözlem işleyicileri paralel
çalışır ve çalıştır-unut gönderimleri çakışabilir; bu nedenle tüketiciler olayların
tamamlanma sırasına bağlı olmamalıdır. Zamanı gelen kontroller ve yürütme için doğruluk
kaynağı olarak OpenClaw'u koruyun.

Kalıcı değiştirme, yeniden deneme/geri çekilme ve temiz kapatma özelliklerine sahip
tek uçuşlu bir bağdaştırıcı için [Güvenli harici Cron projeksiyonu](/tr/plugins/hooks#safe-external-cron-projection) bölümüne bakın.

### API nesnesi alanları

| Alan                    | Tür                      | Açıklama                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Plugin kimliği                                                                                   |
| `api.name`               | `string`                  | Görünen ad                                                                                |
| `api.version`            | `string?`                 | Plugin sürümü (isteğe bağlı)                                                                   |
| `api.description`        | `string?`                 | Plugin açıklaması (isteğe bağlı)                                                               |
| `api.source`             | `string`                  | Plugin kaynak yolu                                                                          |
| `api.rootDir`            | `string?`                 | Plugin kök dizini (isteğe bağlı)                                                            |
| `api.config`             | `OpenClawConfig`          | Geçerli yapılandırma anlık görüntüsü (varsa etkin bellek içi çalışma zamanı anlık görüntüsü)                  |
| `api.pluginConfig`       | `Record<string, unknown>` | `plugins.entries.<id>.config` öğesinden plugin'e özgü yapılandırma                                   |
| `api.runtime`            | `PluginRuntime`           | [Çalışma zamanı yardımcıları](/tr/plugins/sdk-runtime)                                                     |
| `api.logger`             | `PluginLogger`            | Kapsamlı günlükçü (`debug`, `info`, `warn`, `error`)                                            |
| `api.registrationMode`   | `PluginRegistrationMode`  | Geçerli yükleme modu; `"setup-runtime"`, tam giriş öncesindeki hafif başlangıç/kurulum penceresidir |
| `api.resolvePath(input)` | `(string) => string`      | Yolu plugin köküne göre çözümle                                                        |

## Dahili modül kuralı

Plugin'iniz içinde dahili içe aktarımlar için yerel barrel dosyaları kullanın:

```text
my-plugin/
  api.ts            # Harici tüketiciler için genel dışa aktarımlar
  runtime-api.ts    # Yalnızca dahili çalışma zamanı dışa aktarımları
  index.ts          # Plugin giriş noktası
  setup-entry.ts    # Yalnızca kurulum için hafif giriş (isteğe bağlı)
```

<Warning>
  Üretim kodundan kendi plugin'inizi asla `openclaw/plugin-sdk/<your-plugin>`
  aracılığıyla içe aktarmayın. Dahili içe aktarmaları `./api.ts` veya
  `./runtime-api.ts` üzerinden yönlendirin. SDK yolu yalnızca harici sözleşmedir.
</Warning>

Facade aracılığıyla yüklenen paketli plugin genel yüzeyleri (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` ve benzeri genel giriş dosyaları), OpenClaw
zaten çalışıyorsa etkin çalışma zamanı yapılandırma anlık görüntüsünü tercih eder.
Henüz çalışma zamanı anlık görüntüsü yoksa diskteki çözümlenmiş yapılandırma dosyasına
geri dönerler. Paketlenmiş paketli plugin facade'ları, OpenClaw'ın plugin facade
yükleyicileri aracılığıyla yüklenmelidir; `dist/extensions/...` üzerinden doğrudan içe
aktarmalar, paketlenmiş kurulumların plugin'e ait kod için kullandığı manifest ve
çalışma zamanı sidecar denetimlerini atlar.

Sağlayıcı plugin'leri, bir yardımcı özellikle sağlayıcıya özgü olacak şekilde
tasarlanmışsa ve henüz genel bir SDK alt yoluna ait değilse dar kapsamlı, plugin'e
yerel bir sözleşme barrel'ı sunabilir. Paketli örnekler:

- **Anthropic**: Claude beta-header ve `service_tier` akış
  yardımcıları için genel `api.ts` / `contract-api.ts` bağlantı noktası.
- **`@openclaw/openai-provider`**: `api.ts`; sağlayıcı oluşturucularını,
  varsayılan model yardımcılarını ve gerçek zamanlı sağlayıcı oluşturucularını dışa aktarır.
- **`@openclaw/openrouter-provider`**: `api.ts`; sağlayıcı oluşturucusunu
  ve ilk katılım/yapılandırma yardımcılarını dışa aktarır.

<Warning>
  Uzantı üretim kodu da `openclaw/plugin-sdk/<other-plugin>` içe aktarmalarından
  kaçınmalıdır. Bir yardımcı gerçekten paylaşılıyorsa iki plugin'i birbirine
  bağlamak yerine onu `openclaw/plugin-sdk/speech`, `.../provider-model-shared` veya
  yetenek odaklı başka bir yüzey gibi tarafsız bir SDK alt yoluna yükseltin.
</Warning>

## İlgili

<CardGroup cols={2}>
  <Card title="Giriş noktaları" icon="door-open" href="/tr/plugins/sdk-entrypoints">
    `definePluginEntry` ve `defineChannelPluginEntry` seçenekleri.
  </Card>
  <Card title="Çalışma zamanı yardımcıları" icon="gears" href="/tr/plugins/sdk-runtime">
    Tam `api.runtime` ad alanı referansı.
  </Card>
  <Card title="Kurulum ve yapılandırma" icon="sliders" href="/tr/plugins/sdk-setup">
    Paketleme, manifestler ve yapılandırma şemaları.
  </Card>
  <Card title="Test" icon="vial" href="/tr/plugins/sdk-testing">
    Test yardımcı araçları ve lint kuralları.
  </Card>
  <Card title="SDK geçişi" icon="arrows-turn-right" href="/tr/plugins/sdk-migration">
    Kullanımdan kaldırılmış yüzeylerden geçiş.
  </Card>
  <Card title="Plugin iç yapısı" icon="diagram-project" href="/tr/plugins/architecture">
    Ayrıntılı mimari ve yetenek modeli.
  </Card>
</CardGroup>
