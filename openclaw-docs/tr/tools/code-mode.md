---
read_when:
    - Bir ajan çalıştırması için OpenClaw kod modunu etkinleştirmek istiyorsunuz
    - Code Mode'un Codex Code Mode'dan neden farklı olduğunu açıklamanız gerekiyor
    - Kompakt araç sözleşmesini, QuickJS-WASI korumalı alanını, TypeScript dönüşümünü veya gizli araç kataloğu köprüsünü inceliyorsunuz
    - Dahili bir kod modu ad alanı kayıt defteri entegrasyonu ekliyor veya inceliyorsunuz
sidebarTitle: Code Mode
summary: Kompakt JavaScript veya TypeScript iş akışlarında geniş araç kataloglarını keşfetmek, çağırmak ve birleştirmek için OpenClaw Code Mode'u kullanın
title: Kod Modu
x-i18n:
    generated_at: "2026-07-26T23:03:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a21df3bcfb11668da6dde1f7c69adcc284a28dc491c95f95097ce7f41e5c45bf
    source_path: tools/code-mode.md
    workflow: 16
---

Kod modu deneysel, isteğe bağlı bir OpenClaw ajan çalışma zamanı özelliğidir. Etkinleştirildiğinde
model artık etkinleştirilmiş tüm araç şemalarını görmez; bunun yerine
`exec`, `wait` ve yapılandırılmış sonucu yalnızca JSON destekleyen konuk köprüsünden
geçemeyen, yalnızca doğrudan kullanılabilen araçları görür. Model, gizli araç kataloğunda
arama yapan, araçları açıklayan ve çağıran küçük bir JavaScript veya TypeScript
programı yazar.

Bu sayfa Codex Code Mode'u değil, OpenClaw kod modunu belgeler. İki özellik
aynı adı ve aynı denetim aracı adlarını (`exec`, `wait`) paylaşır, ancak bunlar
ayrı uygulamalardır:

- Codex Code Mode, Codex kodlama düzeneği içinde çalışır. `exec` aracı
  serbest biçimli dil bilgisine sahip bir araçtır: model, Codex'in süreç içi
  V8 Code Mode çalışma zamanında yürütülen ham JavaScript kaynak kodunu
  (isteğe bağlı olarak yürütme seçenekleri için bir `// @exec: {...}` pragma satırıyla başlayan) yazar.
- OpenClaw kod modu, genel OpenClaw ajan çalışma zamanında çalışır ve
  `tools.codeMode.enabled: true` yapılandırılmadıkça devre dışıdır. `exec`
  aracı, QuickJS-WASI iş parçacığında yürütülen bir JSON `{ code, language }` yükü
  alır.

Her ikisi de kabuk komutu yüzeyleri değil, JavaScript yürütme yüzeyleridir. Bunları
aynı adlara sahip `exec`/`wait` araçlarını açığa çıkaran,
birbirinden bağımsız ve farklı biçimde uygulanmış özellikler olarak değerlendirin.

## Ne yapar?

- Modelin görebildiği araç listesi `exec`, `wait` ve görüntü sonucu
  konuk köprüsünden geçemeyen `computer` ya da yerel görüntü `image` yükleyicisi
  gibi yalnızca doğrudan kullanılabilen araçlardan oluşur.
- `exec`, model tarafından oluşturulan JavaScript veya TypeScript'i yalıtılmış
  bir QuickJS-WASI işçi iş parçacığında değerlendirir.
- Kataloğa uygun etkinleştirilmiş her araç (OpenClaw çekirdeği, plugin, MCP, istemci), bağımsız
  bir model aracı olarak gizlenir ve konuk programında `ALL_TOOLS`
  ile `tools` üzerinden kullanıma sunulur.
- `exec` açıklaması; kesin OpenClaw/plugin katalog kimliklerinin, kompakt girdi
  ipuçlarının ve güvenilir bir araç çıktı şeması sağladığında kompakt bildirilmiş
  çıktı ipuçlarının sınırlı bir hızlı dizinini taşır. Açıklamaları, tam şemaları,
  MCP girdilerini ve kapasiteyi aşan girdileri içermez; konuk tarafındaki katalog araması yedek yöntem olarak kalır.
- Konuk kodu gizli katalogda arama yapar, bir aracın şemasını açıklar ve
  normal ajan turlarında kullanılan yürütme yoluyla bir aracı çağırır (politika,
  onaylar, kancalar ve telemetri uygulanmaya devam eder).
- MCP araçları `MCP` ad alanı altında gruplandırılır; kod modunda bunları
  çağırmanın desteklenen tek yolu budur.
- `wait`, iç içe araç çağrıları hâlâ beklemedeyken askıya alınmış bir kod modu
  çalışmasını sürdürür.

Kod modu yalnızca modele yönelik orkestrasyon yüzeyini değiştirir. Araçların,
plugin araçlarının, MCP araçlarının, kimlik doğrulamanın, onay politikasının, kanal
davranışının veya model seçiminin yerini almaz.

## Neden kullanılmalı?

- Daha küçük istem yüzeyi: sağlayıcılara düzinelerce veya yüzlerce tam araç
  şeması yerine iki denetim aracı, sınırlı bir yerel araç dizini ve yalnızca
  gerekli birkaç doğrudan araç sağlanır.
- Daha iyi orkestrasyon: model, tek bir kod hücresinde döngüler, birleştirmeler,
  küçük dönüşümler, koşullu mantık ve paralel iç içe araç çağrıları kullanabilir.
- Daha az model gidiş dönüşü: bildirilmiş bir çıktı sözleşmesi, modelin bir araç
  sonucunu tek bir `exec` içinde çağırıp dönüştürmesini sağlar; bilinmeyen çıktılarda önce ham veri kullanılır.
- Sağlayıcıdan bağımsız: sağlayıcıya özgü kod yürütmeye bağlı olmadan OpenClaw,
  plugin, MCP ve istemci araçlarıyla çalışır.
- Güvenli biçimde başarısız olur: kod modu etkinleştirilmiş ancak QuickJS-WASI çalışma zamanı
  kullanılamıyorsa çalışma, sessizce geniş kapsamlı doğrudan araç erişimine
  geri dönmek yerine başarısız olur.

En çok, etkinleştirilmiş araç kataloğu büyük olan ajanlar veya modelin yanıt
vermeden önce birkaç aracı araması, birleştirmesi ve çağırması gereken iş akışları için kullanışlıdır.

Küçük bir katalog veya kısa programları güvenilir biçimde yazamayan bir model için
doğrudan araç erişimini koruyun. Kompakt bir katalog istediğiniz ancak
QuickJS-WASI konuğu yerine yapılandırılmış arama/açıklama/çağırma denetimlerini
tercih ettiğiniz durumlarda [Araç Arama](/tr/tools/tool-search) özelliğini kullanın.

## Hızlı başlangıç

### Code Mode'u etkinleştirme

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

Kısa gösterim:

```json5
{
  tools: {
    codeMode: true,
  },
}
```

`tools.codeMode` belirtilmediğinde, `false` olduğunda veya `enabled: true` içermeyen
bir nesne olduğunda kod modu kapalı kalır.

Yapılandırılmış MCP sunucularına sahip korumalı alan ajanları kullanıyorsanız,
pakete dahil MCP plugin'ine korumalı alan araç politikasında da izin verin;
örneğin `tools.sandbox.tools.alsoAllow: ["bundle-mcp"]`. Bkz.
[Yapılandırma - araçlar ve özel sağlayıcılar](/tr/gateway/config-tools#mcp-and-plugin-tools-inside-sandbox-tool-policy).

Daha sıkı sınırlar için açık limitler belirleyin:

```json5
{
  tools: {
    codeMode: {
      enabled: true,
      timeoutMs: 10000,
      memoryLimitBytes: 67108864,
      maxOutputBytes: 65536,
      maxSnapshotBytes: 10485760,
      maxPendingToolCalls: 16,
      snapshotTtlSeconds: 900,
      searchDefaultLimit: 8,
      maxSearchLimit: 50,
    },
  },
}
```

### Modelin yaptığı işlemler

`Array<{ id: string; paid: boolean; tons: number }>` gibi bildirilmiş bir çıktıya sahip bir araç için
tek bir konuk programı aracı seçebilir, çağırabilir ve dönüştürebilir:

```javascript
const [shipmentTool] = await tools.search("list shipments");
const shipments = await tools.callValue(shipmentTool.id, {});
return shipments.filter((shipment) => !shipment.paid && shipment.tons > 10);
```

Hızlı dizin satırı `-> ?` ile bittiğinde çıktı şekli bilinmez. İlk
`exec`, `await tools.callValue(...)` değerini değiştirmeden döndürmelidir. Daha sonraki bir `exec`,
gözlemlenen değeri dönüştürebilir. Bu, ek bir model turuna mal olur ancak
modelin alan adlarını tahmin etmesini önler.

### Etkin yüzeyi doğrulama

Hata ayıklama sırasında model yükünün şeklini doğrulamak için Gateway'i
hedefli günlük kaydıyla çalıştırın:

```bash
OPENCLAW_DEBUG_CODE_MODE=1 \
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 \
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools \
openclaw gateway
```

Kod modu etkinken günlüğe kaydedilen, modele yönelik araç adları `exec` ve
`wait` olmalıdır. Düzenlenmiş tam sağlayıcı yükü için kısa bir hata ayıklama
oturumu boyunca `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted` ekleyin.

## Ajanları paralel çalıştırmak için Swarm kullanma

[Swarm](/tools/swarm), Code Mode betiklerinden eşzamanlı alt ajanları orkestre etmek için
`agents.run()`, `phase()` ve `log()` konuk globallerini ekler. `tools.codeMode`
ve `tools.swarm` özelliklerini etkinleştirin, ardından paralel çalıştırma,
karar kapıları ve yapılandırılmış toplama için normal JavaScript denetim akışını kullanın.
Swarm ayrı bir isteğe bağlı geçittir; yalnızca Code Mode'u etkinleştirmek
`agents.*` API'sini kullanıma sunmaz.

## Teknik inceleme

Bu sayfanın geri kalanı; bakımcılar, araç erişiminde hata ayıklayan plugin yazarları
ve yüksek riskli dağıtımları doğrulayan operatörler için çalışma zamanı sözleşmesini
ve uygulama ayrıntılarını ele alır.

## Çalışma zamanı durumu

|                     |                                                                                             |
| ------------------- | ------------------------------------------------------------------------------------------- |
| Çalışma zamanı      | [`quickjs-wasi`](https://github.com/vercel-labs/quickjs-wasi)                               |
| Varsayılan durum    | devre dışı                                                                                  |
| Kararlılık          | deneysel OpenClaw yüzeyi (Codex Code Mode, ayrı ve kararlı bir Codex düzenek yüzeyidir)      |
| Hedef yüzey         | genel OpenClaw ajan çalışmaları                                                              |
| Güvenlik yaklaşımı  | model kodu düşmanca kabul edilir                                                             |
| Kullanıcıya yönelik taahhüt | kod modunu etkinleştirmek hiçbir zaman sessizce geniş kapsamlı doğrudan araç erişimine geri dönmez |

## Kapsam

Kod modu, hazırlanmış bir çalışma için modele yönelik orkestrasyon biçimini yönetir.
Model seçimini, kanal davranışını, kimlik doğrulamayı, araç politikasını veya araç
uygulamalarını yönetmez.

Kapsam dahilinde: modelin görebildiği denetim/doğrudan araç tanımları, gizli araç
kataloğunun oluşturulması, JavaScript/TypeScript konuk yürütmesi, QuickJS-WASI işçi
çalışma zamanı, arama/açıklama/çağırma için ana makine geri çağırmaları, askıya alınmış
konuk programları için sürdürülebilir durum, çıktı/zaman aşımı/bellek/bekleyen çağrı/anlık
görüntü sınırları ve iç içe araç çağrıları için telemetri/yörünge izdüşümü.

Kapsam dışında: sağlayıcıya özgü uzaktan kod yürütme, kabuk yürütme
anlambilimi, mevcut araç yetkilendirmesini değiştirme, kullanıcı tarafından yazılmış
kalıcı betikler, konuk kodunda paket yöneticisi/dosya/ağ/modül erişimi ve
Codex Code Mode iç bileşenlerinin doğrudan yeniden kullanımı.

Uzak Python korumalı alanları gibi sağlayıcıya ait araçlar ayrı araçlardır. Bkz.
[Kod yürütme](/tr/tools/code-execution).

## Terimler

- **Kod modu**: kataloğa uyumlu model araçlarını gizleyen ve `exec`,
  `wait` ile gerekli yalnızca doğrudan araçları kullanıma sunan OpenClaw çalışma zamanı modu.
- **Konuk çalışma zamanı**: model kodunu değerlendiren QuickJS-WASI JavaScript sanal makinesi.
- **Ana makine köprüsü**: konuk kodundan OpenClaw'a geri dönen dar, JSON uyumlu
  geri çağırma yüzeyi.
- **Katalog**: normal araç politikası, plugin, MCP ve istemci aracı çözümlemesinden
  sonraki etkin araçların çalışma kapsamındaki listesi.
- **İç içe araç çağrısı**: konuk kodundan ana makine köprüsü üzerinden yapılan
  bir araç çağrısı.
- **Anlık görüntü**: `wait` aracının askıya alınmış bir kod modu çalışmasını
  sürdürebilmesi için kaydedilen serileştirilmiş QuickJS-WASI sanal makine durumu.

## Yapılandırma

Etkinleştirme geçidi `tools.codeMode.enabled` değeridir; diğer alanların ayarlanması
özelliği tek başına etkinleştirmez.

| Alan                  | Varsayılan                     | Sınırlandırma                                   |
| --------------------- | ------------------------------ | ----------------------------------------------- |
| `enabled`             | `false`                        | boolean; yalnızca `true` kod modunu etkinleştirir |
| `runtime`             | `"quickjs-wasi"`               | desteklenen tek değer                           |
| `mode`                | `"only"`                       | denetim/doğrudan araçlarını sunar, geri kalanını kataloglar |
| `languages`           | `["javascript", "typescript"]` | ikisinin herhangi bir alt kümesi                |
| `timeoutMs`           | `10000`                        | `100`-`60000`                                   |
| `memoryLimitBytes`    | `67108864`                     | `1048576`-`1073741824`                          |
| `maxOutputBytes`      | `65536`                        | `1024`-`10485760`                               |
| `maxSnapshotBytes`    | `10485760`                     | `1024`-`268435456`                              |
| `maxPendingToolCalls` | `16`                           | `1`-`128`                                       |
| `snapshotTtlSeconds`  | `900`                          | `1`-`86400`                                     |
| `searchDefaultLimit`  | `8`                            | `maxSearchLimit` ile sınırlandırılır             |
| `maxSearchLimit`      | `50`                           | `1`-`50`                                        |

Kod modu etkinleştirilmiş ancak QuickJS-WASI yüklenemiyorsa OpenClaw ilgili
çalışma için güvenli biçimde başarısız olur; yedek yöntem olarak normal araçları sessizce kullanıma sunmaz.

## Etkinleştirme

Kod modu, etkin araç politikası belirlendikten sonra ve nihai model isteği
oluşturulmadan önce değerlendirilir:

1. Ajanı, modeli, sağlayıcıyı, korumalı alanı, kanalı, göndereni ve çalıştırma
   politikasını çözümleyin.
2. Uygun plugin, MCP ve istemci araçlarını ekleyerek etkin OpenClaw araç
   listesini oluşturun.
3. İzin verme/reddetme politikasını uygulayın.
4. `tools.codeMode.enabled` false ise normal araç sunumuyla devam edin.
5. Etkinleştirilmişse ve araçlar çalıştırma için etkinse gerekli yalnızca doğrudan
   araçları koruyun ve katalog için uygun her etkin aracı kod modu
   kataloğuna kaydedin.
6. Kataloglanan araçları modelin görebildiği listeden kaldırın; korunan yalnızca doğrudan araçların
   yanına `exec` ve `wait` ekleyin.

Kasıtlı olarak hiç aracı olmayan çalıştırmalar (ham model çağrıları, `disableTools: true`
veya boş bir `tools.allow` listesi), `tools.codeMode.enabled: true` yapılandırılmış olsa bile
kod modu yüzeyini etkinleştirmez. Kod modu ile OpenClaw Araç
Araması bir çalıştırmada birbirini dışlar; kod modu etkinleşirse Araç Araması'nın
Compaction işlemi gerçekleşmez.

Kod modu kataloğu çalıştırma kapsamındadır ve başka bir
ajanın, oturumun, gönderenin veya çalıştırmanın araçlarını sızdırmamalıdır.

## Modelin görebildiği araçlar

Kod modu etkinken model `exec`, `wait` ve gerekli tüm
yalnızca doğrudan araçları görür. Etkinleştirilmiş diğer tüm araçlar modele yönelik
araç listesinden gizlenir ve kod modu kataloğuna kaydedilir.

Araç düzenleme, veri birleştirme, döngüler, paralel iç içe çağrılar
ve yapılandırılmış dönüşümler için `exec` kullanın. `wait` öğesini yalnızca `exec` sürdürülebilir
bir `waiting` sonucu döndürdüğünde kullanın.

## `exec`

`exec` bir kod modu hücresi başlatır ve tek bir sonuç döndürür. Girdi kodu model
tarafından oluşturulur ve düşmanca kabul edilmelidir.

Girdi:

```typescript
type CodeModeExecInput = {
  code?: string;
  command?: string;
  language?: "javascript" | "typescript";
};
```

Kurallar:

- `code` veya `command` öğelerinden biri boş olmamalıdır.
- `code`, belgelenmiş ve modele yönelik alandır.
- `command`, hook politikaları ve güvenilir yeniden yazımlar için exec uyumlu bir diğer ad
  olarak kabul edilir (normal OpenClaw kabuk exec aracı da bir `command`
  alanı kullanır); ikisi de mevcutsa değerler eşleşmelidir.
- `language` varsayılan olarak `"javascript"` değerini alır; bazı sağlayıcılar bu şekilleri
  reddettiğinden şema bunu bir `oneOf`/`anyOf` birleşimi
  olarak değil, düz bir dize numaralandırması (`"javascript" | "typescript"`) olarak sunar.
- `language`, `"typescript"` ise OpenClaw değerlendirmeden önce kodu dönüştürür.
- `exec`; `import`, `require`, dinamik içe aktarma ve modül yükleyici
  kalıplarını reddeder.
- `exec`, normal kabuk `exec` uygulamasını hiçbir zaman yinelemeli olarak sunmaz.
- Dış kod modu `exec` hook olayları, politikaların
  aynı araç adını paylaşan kod modu hücrelerini kabuk tarzı `exec` çağrılarından
  ayırt edebilmesi için `toolKind: "code_mode_exec"` ve
  `toolInputKind: "javascript" | "typescript"` (biliniyorsa) taşır.

Sonuç:

```typescript
type CodeModeResult = CodeModeCompletedResult | CodeModeWaitingResult | CodeModeFailedResult;

type CodeModeCompletedResult = {
  status: "completed";
  value: unknown;
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};

type CodeModeWaitingResult = {
  status: "waiting";
  runId: string;
  reason: "pending_tools" | "yield";
  pendingToolCalls?: CodeModePendingToolCall[];
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};

type CodeModeFailedResult = {
  status: "failed";
  error: string;
  code?: CodeModeErrorCode;
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};
```

`exec`, konuk hâlâ modelin görebildiği bir sürdürme gerektiren devam ettirilebilir durumla
askıya alındığında `waiting` döndürür: açık bir `yield_control(...)` veya
exec son tarihine kadar çözümlenmemiş bir köprü araç çağrısı. Sonuç,
`wait` için bir `runId` içerir. MCP ad alanı çağrıları dâhil olmak üzere
köprü araç çağrıları — `tools.search`/`describe`/
`call` ve ad alanı çağrıları — son tarih içinde çözümlendikleri sürece
aynı `exec`/`wait` çağrısının içinde otomatik olarak tüketilir; böylece birkaç aracı bekleyen
kompakt bir kod bloğu, her bekleme için bir model araç çağrısını zorunlu kılmak yerine tek bir model
turunda tamamlanır. Yeniden başlatmaya dayanıklı çalıştırmalar hiçbir zaman
otomatik tüketim yapmaz; bekleyen işleri yine yeniden oynatma güvenli denetimlerden geçer.

`exec`, yalnızca konuk VM'de bekleyen iş olmadığında ve
nihai değer OpenClaw'ın çıktı bağdaştırıcısı çalıştıktan sonra JSON uyumlu olduğunda `completed` döndürür.

## `wait`

`wait`, askıya alınmış bir kod modu VM'sini sürdürür.

Girdi:

```typescript
type CodeModeWaitInput = {
  runId: string;
};
```

Çıktı, `exec` tarafından döndürülen aynı `CodeModeResult` birleşimidir.

`wait`, iç içe OpenClaw araçları yavaş, etkileşimli, onay
geçitli olabildiği veya kısmi güncellemeler yayımlayabildiği için vardır; ana makine harici işi beklerken
modelin uzun bir `exec` çağrısını açık tutması gerekmemelidir.

Sürdürme mekanizması QuickJS-WASI anlık görüntüleme/geri yüklemedir:

1. `exec`, kodu tamamlanma, başarısızlık veya askıya alınma gerçekleşene kadar değerlendirir.
2. Askıya alınma sırasında OpenClaw, QuickJS VM'nin anlık görüntüsünü alır ve bekleyen ana makine
   işini kaydeder.
3. Bekleyen iş sonuçlandığında `wait`, VM anlık görüntüsünü geri yükler ve
   ana makine geri çağrılarını kararlı adlarla yeniden kaydeder.
4. OpenClaw, iç içe araç sonuçlarını geri yüklenen VM'ye iletir ve
   bekleyen QuickJS işlerini tüketir.
5. `wait`; `completed`, `failed` veya başka bir `waiting` sonucu döndürür.

Anlık görüntüler kullanıcı yapıtları değil, çalışma zamanı durumudur: yalnızca
işlem içi bir eşlemede tutulur (veritabanına veya diske yazılmaz), boyutları sınırlıdır, süreleri dolar ve
kendilerini oluşturan çalıştırma ile oturumun kapsamındadır.

`wait`, şu durumlarda (`failed` sonucu olarak) başarısız olur:

- `runId` bilinmiyorsa veya anlık görüntüsünün süresi zaten dolmuşsa.
- çağıran, askıya alınmış çalıştırmayla aynı çalıştırma/oturum kapsamında değilse.
- bu `runId` için zaten işlemde olan bir `wait` varsa.
- QuickJS-WASI geri yüklemesi başarısız olursa.
- sürdürme işlemi `maxOutputBytes` veya `maxSnapshotBytes` sınırını aşacaksa.

## Konuk çalışma zamanı API'si

```typescript
declare const ALL_TOOLS: ToolCatalogEntry[];
declare const tools: ToolCatalog;
declare const MCP: Record<string, unknown>;
declare const namespaces: Record<string, unknown>;

declare function text(value: unknown): void;
declare function json(value: unknown): void;
declare function yield_control(reason?: string): Promise<void>;
```

`ALL_TOOLS`, çalıştırma kapsamındaki katalog için kompakt meta verilerdir; varsayılan olarak
tam şemaları içermez. Modelin görebildiği `exec` açıklaması ayrıca tam OpenClaw/plugin kimliklerinin
sınırlı ve belirlenimci bir alt kümesini, kompakt girdi ipuçlarını
ve güvenilir bildirilmiş çıktı ipuçlarını içerir. Düşmanca katalog metninin
modeli yönlendirememesi için açıklamalar ertelenmiş olarak kalır. Bu dizin bir aracı içermediğinde
`ALL_TOOLS` öğesini okuyun veya konuk programın içinde `tools.search(...)` çağrısı yapın.

Her hızlı dizin satırındaki ok, `tools.callValue(...)` değerini açıklar.
`-> Array<{ id: string }>`, bildirilmiş bir çıktı ipucudur; `-> ?`, çıktının bilinmediği anlamına gelir.
Bilinmeyen çıktılar önce ham hâlde kalır: alan adlarını tahmin etmek yerine değeri değiştirmeden döndürün,
gözlemleyin, ardından daha sonraki bir `exec` içinde filtreleyin veya eşleyin. Bu,
bildirilmiş çıktı okumasının nihai bir `-> ?` çağrısını beslediği durumlarda da geçerlidir: bu
çağrının ham değerini, istenen yanıt şekline sarmadan döndürün.

```typescript
type ToolCatalogEntry = {
  id: string;
  name: string;
  label?: string;
  description: string;
  source: "openclaw" | "mcp" | "client";
  sourceName?: string;
  input: string;
  output?: string;
};
```

`input`, yaygın durum için sınırlı TypeScript tarzı bir imzadır. Tam şemaya hâlâ ihtiyaç duyulduğunda
`tools.describe(...)` kullanın. Uzak MCP
ve istemci girdileri `input: "unknown"` kullanır; böylece güvenilmeyen şemaları
`describe` zamanına kadar ertelenmiş olarak kalır. `output`,
yalnızca güvenilir bir OpenClaw çekirdeğinden veya plugin `outputSchema` öğesinden türetilmiş eksiksiz bir kompakt ipucu için
mevcuttur. MCP ve istemci çıktı şeması iddiaları bu güvenilir katalog ipucuna
yükseltilmez.

Plugin araçları, `sourceName` sahibi plugin kimliğine ayarlanmış olarak `source: "openclaw"` kullanır;
ayrı bir `"plugin"` kaynak değeri yoktur. `source: "mcp"`,
yalnızca `sourceName`/`mcp` meta verilerindeki MCP girdileri için kullanılır (ve
`ALL_TOOLS`/`tools.*` öğelerinden filtrelenir; aşağıya bakın).

Tam şema yalnızca talep üzerine yüklenir:

```typescript
type ToolCatalogEntryWithSchema = ToolCatalogEntry & {
  parameters: unknown;
  outputSchema?: unknown;
};
```

Katalog yardımcıları:

```typescript
type ToolCatalog = {
  search(query: string, options?: { limit?: number }): Promise<ToolCatalogEntry[]>;
  describe(id: string): Promise<ToolCatalogEntryWithSchema>;
  callValue(id: string, input?: unknown): Promise<unknown>;
  call(id: string, input?: unknown): Promise<unknown>;
  [safeToolName: string]: unknown;
};
```

Kolaylık sağlayan araç işlevleri yalnızca belirsizlik taşımayan güvenli adlar için kurulur:

```typescript
const files = await tools.search("read local file");
const fileRead = await tools.describe(files[0].id);
const content = await tools.callValue(fileRead.id, { path: "README.md" });

// Gizli katalogda belirsizlik taşımayan bir `web_search` girdisi varsa:
const hits = await tools.web_search({ query: "OpenClaw code mode" });
```

`tools.callValue(...)`, normal bir aracın JSON `details` değerini doğrudan döndürür.
`tools.call(...)`, içerik bloklarına veya diğer sonuç meta verilerine
ihtiyaç duyan çağıranlar için ham `{ tool, result }` zarfını korur.

## Bildirilmiş çıktı sözleşmeleri

OpenClaw araçları, `AgentToolResult.details` içine yerleştirilen yapılandırılmış değer için
`outputSchema` bildirebilir. Bu, Kod Modu ve Araç Araması için yararlıdır;
sağlayıcıya özgü bir araç yanıt şeması değildir ve doğrudan araç
sunumunu değiştirmez.

`defineToolPlugin` ile oluşturulan bir araç için şemayı
`parameters` yanında bildirin:

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

const Shipment = Type.Object(
  {
    id: Type.String(),
    paid: Type.Boolean(),
    tons: Type.Number(),
  },
  { additionalProperties: false },
);

export default defineToolPlugin({
  id: "shipping",
  name: "Shipping",
  description: "Shipment tools.",
  tools: (tool) => [
    tool({
      name: "shipping_list",
      description: "List shipments.",
      parameters: Type.Object({}),
      outputSchema: Type.Array(Shipment),
      execute: async () => loadShipments(),
    }),
  ],
});
```

`api.registerTool(...)` veya bir fabrika aracı için aynı `outputSchema`
özelliğini döndürülen `AnyAgentTool` nesnesine koyun.

Mevcut yerleşik sözleşmeler arasında `agents_list`, `apply_patch`,
`conversations_list`, `conversations_send`, `conversations_turn`, `edit`,
`openclaw`, `read`, `screen`,
`sessions_history`, `sessions_list`, `sessions_search`, `sessions_send`,
`session_status`, `spawn_task`, `terminal`, `web_fetch` ve `web_search` bulunur.
Tam geçişler, yalnızca modele özgü bir sözleşmeyi çoğaltmak yerine
sahip oldukları protokol şemasını yeniden kullanabilir. Örneğin, konuşma araçları
`conversations.list`, `conversations.send` ve `conversations.turn` tarafından kullanılan
aynı Gateway sonuç şemalarını sunar; `web_fetch`, ipucu kararlı meta verileri,
metni, önbellek durumunu ve iç içe taşma meta verilerini sunan araca özgü bir
şemaya sahiptir; `web_search`, tam bir hızlı dizin ipucu olarak kendi kesin
normalleştirilmiş sonuç/yanıt/hata/ham birleşimini bildirir. Dosya sistemi sözleşmeleri;
yapılandırılmış okuma metni, görüntü, kesilme ve isteğe bağlı bulunamadı sonuçlarını;
açık düzenleme değişiklik durumuyla birlikte diff/yama verilerini; ayrıca yama
uygulama yolu özetlerini döndürür. Hızlı dizin alanları bildirdiğinde, tek bir
hücre ayrı bir inceleme turu olmadan keşif ve teslimi birleştirebilir:

```javascript
const listed = await tools.conversations_list({ query: "derleme botu" });
const target = listed.conversations.find((item) => item.label === "Derleme botu");
if (!target) throw new Error("konuşma bulunamadı");
return await tools.conversations_send({
  conversationRef: target.conversationRef,
  message: "Derleme tamamlandı.",
});
```

İç içe çağrılar yine normal araç politikasını, kancaları ve onayları kullanır. Tam
bir sözleşme kesin ancak sınırlı hızlı dizin için fazla büyükse
`tools.describe(...)` üzerinden erişilebilir kalır ve ok `-> ?` olarak kalır.

Sözleşme kuralları katıdır:

- İşlenmiş `content` bloklarını veya bir sağlayıcı zarfını değil, tam olarak JSON uyumlu `details` değerini açıklayın.
- Fırlatma yapmayan her başarı veya hata varyantını ekleyin. Aracın kararlı yapılandırılmış bir sonucu yoksa
  `outputSchema` öğesini atlayın.
- Tam bir hızlı dizin ipucu için nesne katmanlarını
  `{ additionalProperties: false }` ile kapatın. Açık, aşırı büyük veya başka şekilde kısmi şemalara
  `tools.describe(...)` üzerinden erişilebilir; ancak bunlar alanların tek turda kullanımını etkinleştirmez.
- OpenClaw, aracı çalıştırmadan önce şemayı derler; ardından normal araç
  kancalarından sonra ve bir katalog çağrısı dönmeden önce nihai `details`
  değerini doğrular. Geçersiz bir şemayla araç çalıştırılamaz; uyumsuzluk, değer
  yazdırılmadan başarısız olur.
- Kompakt ipuçları deterministik ve sınırlıdır. Kompakt ipucu yetersiz olduğunda
  `tools.describe(...)` tam güvenilir şemayı sunar.
- Yüklü plugin kodu zaten güvenilir yerel koddur. Uzak MCP ve istemci
  meta verileri güvenilmez kalır ve bu hızlı dizin ipuçlarına katılamaz.

Plugin yazma ayrıntıları için [Araç pluginleri](/tr/plugins/tool-plugins#output-contracts) bölümüne bakın.

MCP katalog girdileri kod modunda `tools.callValue(...)`,
`tools.call(...)` veya kolaylık işlevleri üzerinden çağrılamaz; yalnızca
oluşturulan `MCP` ad alanı üzerinden sunulur. TypeScript tarzı bildirim dosyaları,
salt okunur `API` sanal dosya yüzeyi üzerinden kullanılabilir; böylece aracılar
istem istemine MCP şemaları eklemeden MCP imzalarını inceleyebilir:

```typescript
const files = await API.list("mcp");
const githubApi = await API.read("mcp/github.d.ts");

const issue = await MCP.github.createIssue({
  owner: "openclaw",
  repo: "openclaw",
  title: "Gateway günlüklerini araştır",
});

const snapshot = await MCP.chromeDevtools.takeSnapshot({ output: "markdown" });
const resource = await MCP.docs.resources.read({ uri: "memo://one" });
const prompt = await MCP.docs.prompts.get({
  name: "brief",
  arguments: { topic: "release" },
});
```

`API.read("mcp/<server>.d.ts")`, MCP araç meta verilerinden çıkarılan kompakt bildirimleri
döndürür:

```typescript
type McpToolResult = {
  content?: unknown[];
  structuredContent?: unknown;
  isError?: boolean;
  [key: string]: unknown;
};

declare namespace MCP.github {
  /** Bu TypeScript tarzı API üstbilgisini döndürür. */
  function $api(toolName?: string, options?: { schema?: boolean }): Promise<McpApiHeader>;

  /**
   * Bir GitHub sorunu oluşturur.
   * @param owner Depo sahibi
   * @param repo Depo adı
   * @param title Sorun başlığı
   */
  function createIssue(input: {
    owner: string;
    repo: string;
    title: string;
    body?: string;
  }): Promise<McpToolResult>;
}
```

Bildirim dosyaları sanaldır; çalışma alanı veya durum dizini altına
yazılmaz. OpenClaw, kod modundaki her `exec` çağrısı için çalıştırma kapsamlı araç
kataloğunu oluşturur, görünür MCP girdilerini korur, `mcp/index.d.ts` ile görünür
her sunucu için bir `mcp/<server>.d.ts` işler ve bu küçük salt okunur tabloyu
QuickJS çalışanına enjekte eder. Konuk kod yalnızca `API` nesnesini görür:
`API.list(prefix?)` dosya meta verilerini, `API.read(path)` ise seçilen
bildirim içeriğini döndürür. Bilinmeyen yollar ve `.`/`..` segmentleri
reddedilir.

Bu yaklaşım, büyük MCP şemalarını model isteminin dışında tutar: aracı,
`exec` araç açıklamasından sanal API'nin varlığını öğrenir, yalnızca gereken
bildirim dosyasını okur ve ardından tek bir nesne bağımsız değişkeniyle
`MCP.<server>.<tool>()` çağrısı yapar.
`MCP.<server>.$api()`, program içinde tek araçlık şema yanıtı için satır içi yedek olarak
kullanılabilir kalır.

Konuk çalışma zamanı ana makine nesnelerini hiçbir zaman doğrudan görmez. Girdiler ve çıktılar,
açık boyut sınırlarıyla JSON uyumlu değerler olarak köprüden geçer.

## Dahili ad alanları

Dahili ad alanları, modele görünür daha fazla araç eklemeden kod moduna kısa ve öz
bir etki alanı API'si sağlar. Yükleyiciye ait bir entegrasyon,
`Issues` veya `Calendar` gibi bir ad alanını kaydeder; ardından konuk kod
QuickJS programı içinde bu ad alanını çağırırken model kompakt denetim/doğrudan yüzeyi görmeye devam eder.

Ad alanları şimdilik dahilidir. Herkese açık bir plugin SDK ad alanı API'si yoktur:
harici plugin ad alanları, plugin kimliğinin, yüklü manifestlerin, kimlik doğrulama
durumunun ve önbelleğe alınmış katalog tanımlayıcılarının ad alanını destekleyen plugin
araçlarından sapmaması için yükleyiciye ait bir sözleşme gerektirir. Çekirdek kod modu
yalnızca korumalı alanın, serileştirmenin, katalog geçitlerinin ve köprü yönlendirmenin
sahibidir.

Konuk kod doğrudan globali veya `namespaces` eşlemesini kullanabilir:

```javascript
const open = await Issues.list({ state: "open" });
const alsoOpen = await namespaces.Issues.list({ state: "open" });
return { count: open.length, alsoCount: alsoOpen.length };
```

### Kayıt defteri yaşam döngüsü

Ad alanı kayıt defteri işlem yereldir ve ad alanı kimliğine göre anahtarlanır:

1. Güvenilir bir yükleyici `registerCodeModeNamespaceForPlugin(pluginId, registration)` çağrısını yapar.
2. Kod modu çalıştırma için gizli `ToolSearchRuntime` öğesini oluşturur ve
   çalıştırma kapsamlı kataloğunu okur.
3. `createCodeModeNamespaceRuntime(ctx, catalog)`, yalnızca tüm `requiredToolNames`
   öğeleri görünür olan ve aynı `pluginId` tarafından sahiplenilen kayıtları tutar.
4. Her görünür ad alanı mevcut çalıştırma için `createScope(ctx)` çağrısını yaparak
   `agentId`, `sessionKey`, `sessionId`,
   `runId`, yapılandırma ve iptal durumu gibi çalıştırma bağlamını alır.
5. Kapsam verileri düz bir tanımlayıcıya serileştirilir ve doğrudan globaller ile
   `namespaces.<globalName>` olarak QuickJS'e enjekte edilir.
6. Konuk çağrıları çalışan köprüsü üzerinden askıya alınır, ana makinedeki ad alanı yolu
   çözümlenir, çağrı bildirilmiş ve plugine ait bir katalog aracına eşlenir ve
   bu araç `ToolSearchRuntime.callExactId` üzerinden yürütülür.
7. Hazır ad alanı köprü çağrıları etkin `exec`/`wait`
   çağrısı içinde otomatik olarak boşaltılır; zaman aşımında ad alanı çalışması hâlâ bekliyorsa
   veya konuk açıkça yürütmeyi devrederse, `wait` daha sonra aynı ad alanı
   çalışma zamanını sürdürür.
8. Plugin geri alma veya kaldırma işlemi, başarısız bir plugin yüklemesinden sonra
   eski globallerin kalmaması için `clearCodeModeNamespacesForPlugin(pluginId)` çağrısını yapar.

Ad alanı çağrıları katalog aracı çağrılarıdır: `tools.call(...)` ile aynı politika kancalarını,
onayları, iptal işlemeyi, telemetriyi, transkript yansıtmayı ve
askıya alma/sürdürme davranışını kullanırlar.

### Kayıt biçimi

Ad alanlarını, destekleyici araçların sahibi olan entegrasyondan kaydedin. Kapsamı
küçük tutun ve yalnızca bildirilmiş katalog araçlarına eşlenen etki alanı fiillerini sunun.

```typescript
import {
  createCodeModeNamespaceTool,
  registerCodeModeNamespaceForPlugin,
} from "../agents/code-mode-namespaces.js";

const pluginId = "github";

registerCodeModeNamespaceForPlugin(pluginId, {
  id: "github-issues",
  globalName: "Issues",
  description: "Mevcut depo için GitHub sorunu yardımcıları.",
  requiredToolNames: ["github_list_issues", "github_update_issue"],
  prompt: "Issues.list(params) ve Issues.update(number, patch) kullanın.",
  createScope: (ctx) => ({
    repository: ctx.config,
    list: createCodeModeNamespaceTool("github_list_issues", ([params]) => params ?? {}),
    update: createCodeModeNamespaceTool("github_update_issue", ([number, patch]) => ({
      number,
      patch,
    })),
  }),
});
```

`createCodeModeNamespaceTool(toolName, inputMapper)`, bir kapsam üyesini çağrılabilir bir ad alanı işlevi olarak
işaretler. İsteğe bağlı `inputMapper`, konuk bağımsız değişkenlerini alır ve
destekleyici katalog aracı için girdi nesnesini döndürür; bu yoksa ilk konuk bağımsız
değişkeni, bağımsız değişken atlanmışsa `{}` kullanılır.

Ham ana makine işlevleri, konuk kod çalışmadan önce reddedilir:

```typescript
createScope: () => ({
  // Yanlış: bu, katalog aracı yaşam döngüsünü atlar ve reddedilir.
  list: async () => githubClient.listIssues(),
});
```

### Sahiplik ve görünürlük

Ad alanı sahipliği, kayıt çağrısını yapanın `pluginId` öğesine bağlanır.
`requiredToolNames` hem bir görünürlük geçidi hem de sahiplik denetimidir:

- gerekli her araç çalıştırma kataloğunda bulunmalıdır
- gerekli her araçta `sourceName === pluginId` bulunmalıdır
- gerekli herhangi bir araç yoksa veya başka bir plugin tarafından sahiplenilmişse
  ad alanı gizlenir
- çağrılabilir her yol yalnızca `requiredToolNames` içinde adı geçen bir aracı hedefleyebilir

Bu, başka bir pluginin aynı adlı bir aracı kaydederek ad alanı sunmasını engeller
ve ad alanlarını sıradan aracı politikasıyla uyumlu tutar: çalıştırma destekleyici
araçları göremiyorsa ad alanını da göremez.

Örneğin, bir GitHub ad alanı; GitHub kimlik doğrulamasının, REST/GraphQL istemcilerinin,
hız sınırlarının, yazma onaylarının ve testlerin sahibi olan GitHub'a ait bir pluginin
arkasında bulunmalıdır. Çekirdek kod modu GitHub'a özgü API'leri, token işlemeyi veya
sağlayıcı politikasını gömmemelidir.

### Kapsam serileştirme kuralları

`createScope(ctx)`; JSON uyumlu değerler, diziler, iç içe nesneler ve
`createCodeModeNamespaceTool(...)` çağrı işaretçileri içeren düz bir nesne döndürebilir.
Ana makine nesneleri hiçbir zaman doğrudan QuickJS'e girmez.

Serileştirici şunları reddeder:

- ham işlevler
- döngüsel nesne grafikleri
- güvenli olmayan yol segmentleri: `__proto__`, `constructor`, `prototype`, boş anahtarlar
  veya dahili yol ayırıcısını içeren anahtarlar
- JavaScript tanımlayıcıları olmayan `globalName` değerleri
- `globalName` değerinin `tools`,
  `namespaces`, `text`, `json`, `yield_control`, `MCP`, `API`, `ALL_TOOLS` veya
  `__openclaw*` gibi yerleşik kod modu globalleriyle çakışması

JSON olarak serileştirilemeyen değerler, köprüden geçmeden önce JSON için güvenli
yedek değerlere dönüştürülür. İkili veriler, tanıtıcılar, soketler, istemciler ve
sınıf örnekleri sıradan katalog araçlarının arkasında kalmalıdır.

### İstemler

Ad alanı `description` ve isteğe bağlı `prompt`, yalnızca ad alanı söz konusu
çalıştırmada görünür olduğunda modele görünür `exec` şemasına eklenir. Bunları
yararlı en küçük yüzeyi öğretmek için kullanın:

```typescript
{
  description: "Kurgu üretim hizmeti yardımcıları.",
  prompt:
    "Fictions.riskAudit(), Fictions.promoteIfReady(id, status) ve Fictions.unpaidOver(amount) kullanın.",
}
```

İstemleri kimlik doğrulama kurulumu, uygulama geçmişi veya ilgisiz Plugin
davranışlarıyla değil, ad alanı sözleşmesiyle ilgili tutun.

### Temizleme

Ad alanları, işlem yerelindeki kayıtlardır. Sahip olan Plugin devre dışı
bırakıldığında, kaldırıldığında veya geri alındığında bunları kaldırın:

```typescript
clearCodeModeNamespacesForPlugin(pluginId);
```

Kod modu temizliği Plugin tarafından yönetilir; yaşam döngüsü sona erdiğinde
ad alanı başına kapatma tanıtıcıları tutmak yerine Plugin'in ad alanı kayıtlarını
temizleyin. Testler, durumlar arasında kayıt sızıntısını önlemek için
`clearCodeModeNamespacesForTest()` çağrısı yapabilir.

### Test kontrol listesi

Ad alanı değişiklikleri güvenlik sınırını ve konuk davranışını kapsamalıdır:

- ad alanı istem metni yalnızca destekleyen araçlar görünür olduğunda gösterilir
- başka bir `sourceName` içindeki aynı adlı araçlar ad alanını açığa çıkarmaz
- ham kapsam işlevleri reddedilir
- sahte ad alanı kimlikleri ve sahte yollar reddedilir
- çağrılabilir yollar bildirilmemiş araçları hedefleyemez
- iç içe nesneler ve paylaşılan başvurular doğru şekilde serileştirilir
- ad alanı çağrıları katalog araçları üzerinden yürütülür ve JSON açısından güvenli ayrıntılar döndürür
- hatalar konuk kodu tarafından yakalanabilir
- askıya alınan ad alanı çağrıları `wait` üzerinden sürdürülür
- Plugin geri alma işlemi, sahip olunan ad alanı kayıtlarını temizler

Ad alanları genel `tools.search`/`tools.call` kataloğunu tamamlar: rastgele
etkin OpenClaw, Plugin ve istemci araçları için kataloğu; MCP araçları için
`MCP` öğesini; kısa kodun tekrarlanan şema aramalarından daha güvenilir
olduğu, Plugin tarafından yönetilen ve belgelenmiş etki alanı API'leri için
diğer ad alanlarını kullanın.

## Çıktı API'si

- `text(value)`, insan tarafından okunabilir çıktıyı `output` dizisine ekler.
- `json(value)`, JSON uyumlu serileştirmeden sonra yapılandırılmış bir çıktı
  öğesi ekler.
- Konuk kodunun döndürdüğü son değer, bir `completed` sonucunda
  `value` olur.

```typescript
type CodeModeOutput = { type: "text"; text: string } | { type: "json"; value: unknown };
```

Kurallar: çıktı sırası konuk çağrılarıyla eşleşir; çıktı
`maxOutputBytes` ile sınırlandırılır; serileştirilemeyen değerler düz dizelere veya
hatalara dönüştürülür; ikili değerler desteklenmez. Görüntüler ve dosyalar,
kod modu köprüsü üzerinden değil sıradan OpenClaw araçları üzerinden aktarılır.

## Araç kataloğu

Gizli katalog, etkili politika filtrelemesinden sonra araçları şu sırayla
içerir: OpenClaw çekirdek araçları, paketlenmiş Plugin araçları, harici Plugin
araçları, MCP araçları ve ardından geçerli çalıştırma için istemci tarafından sağlanan araçlar.

Katalog kimlikleri bir çalıştırma içinde kararlıdır ve mümkün olduğunda
eşdeğer araç kümelerinde belirlenimlidir. Gerçek biçim:

```text
<source>:<owner>:<tool-name>
```

burada `<source>`; `openclaw`, `mcp` veya `client` olur (Plugin araçları,
Plugin kimliğini `<owner>` olarak kullanarak `openclaw` kullanır; çekirdek araçları
`openclaw:core:*` kullanır).
Örnekler:

```text
openclaw:core:message
openclaw:browser:browser_request
mcp:github:create_issue
client:app:select_file
```

Katalog, kod modu denetim araçlarını (`exec`, `wait`, `tool_search_code`,
`tool_search`, `tool_describe`, `tool_call`) ve yalnızca doğrudan kullanılan araçları içermez.
Denetimler katalog üzerinden özyinelemeli olarak çağrılmamalıdır; yalnızca doğrudan kullanılan
araçlar, yapılandırılmış sonuçları QuickJS köprüsünden geçemediği için model tarafından
görünür kalır.

MCP girdileri, politika, onaylar, kancalar, telemetri, transkript yansıtma
ve tam araç kimliklerinin normal araç yürütmeyle ortak kalması için çalıştırma
kapsamlı katalogda kalır. Konuğa yönelik `ALL_TOOLS`, `tools.search(...)`,
`tools.describe(...)`, `tools.callValue(...)` ve `tools.call(...)` görünümleri MCP girdilerini içermez.
Oluşturulan `MCP.<server>.<tool>({ ...input })` ad alanı, tam katalog kimliğine
geri çözümlenir ve aynı yürütücü yolu üzerinden yönlendirir.

## Araç Arama etkileşimi

Kod modu, etkin olduğu çalıştırmalarda OpenClaw Araç Arama model yüzeyinin
yerini alır.

`tools.codeMode.enabled` true olduğunda ve kod modu etkinleştiğinde:

- OpenClaw; `tool_search_code`, `tool_search`, `tool_describe`
  veya `tool_call` öğelerini model tarafından görülebilen araçlar olarak sunmaz.
- Aynı kataloglama yaklaşımı konuk çalışma zamanının içine taşınır.
- Konuk çalışma zamanı, MCP dışı araçlar için kompakt `ALL_TOOLS`
  meta verileri ve arama/açıklama/çağrı yardımcıları alır.
- MCP çağrıları, `tools.call(...)` yerine oluşturulan `MCP`
  ad alanını ve onun `$api()` başlıklarını kullanır.
- İç içe çağrılar, Araç Arama'nın kullandığı aynı OpenClaw yürütücü yolu
  üzerinden yönlendirilir.

Kod modunun etkin çalıştırmalarda yerini aldığı OpenClaw kompakt katalog köprüsü
için [Araç Arama](/tr/tools/tool-search) bölümüne bakın.

## Araç adları ve çakışmalar

Model tarafından görülebilen `exec` aracı, kod modu aracıdır. Normal OpenClaw
kabuğu `exec` aracı etkinse modelden gizlenir ve diğer tüm araçlar gibi
kataloğa alınır.

Konuk çalışma zamanı içinde:

- `tools.call("openclaw:core:exec", input)`, politika izin veriyorsa kabuk yürütme aracını çağırabilir.
- `tools.exec(...)`, yalnızca kabuk yürütme katalog girdisinin
  kesin ve güvenli bir adı varsa kurulur.
- kod modu `exec` aracı, `tools` üzerinden hiçbir zaman
  özyinelemeli olarak kullanılamaz.

İki araç aynı güvenli kolaylık adına normalleştirilirse OpenClaw, kolaylık
işlevini dahil etmez ve `tools.call(id, input)` kullanılmasını zorunlu kılar.

## İç içe araç yürütme

Her iç içe araç çağrısı ana makine köprüsünü geçer ve OpenClaw'a yeniden girerek
şunları korur: etkin aracı kimliği, oturum kimliği ve anahtarı, gönderen ve kanal
bağlamı, korumalı alan politikası, onay politikası, Plugin `before_tool_call`
kancaları, iptal sinyali, kullanılabildiğinde akış güncellemeleri ve
yörünge/denetim olayları.

İç içe çağrılar, destek paketlerinin ne olduğunu gösterebilmesi için transkripte
gerçek araç çağrıları olarak yansıtılır; yansıtmada üst kod modu araç çağrısı
ve iç içe araç kimliği belirtilir.

`maxPendingToolCalls` sınırına kadar paralel iç içe çağrılara izin verilir.

## Çalıştırma ve anlık görüntü yaşam döngüsü

Her kod modu çalıştırması, `runId` ile anahtarlanan işlem içi bir eşlemde
izlenir (diske veya veritabanına kalıcı olarak kaydedilmez). `exec`/`wait`,
üç sonuç durumundan birini döndürür: `completed`, `waiting` veya `failed`.

- Bir `waiting` sonucu; QuickJS anlık görüntüsünü, bekleyen köprü isteklerini
  ve kapsam meta verilerini (aracı çalıştırma kimliği, oturum kimliği/anahtarı),
  `wait` onu sürdürene veya süresi dolana kadar saklar.
- Süresi dolmuş, yanlış oturumlu, yanlış çalıştırmalı ve bilinmeyen/zaten
  sürdürülen `runId` değerleri ayrı bir son durum oluşturmaz; `code mode
run is unavailable or expired.`
  veya `code mode run belongs to a different
session.` gibi bir iletiyle `failed` sonucu (`code: "invalid_input"`)
  olarak gösterilir.
- Bir çalıştırmanın anlık görüntüsü, `completed` veya `failed`
  durumuna yerleştiği anda eşlemden kaldırılır ya da Gateway kapatıldığında
  bırakılır (yeniden başlatmada hiçbir şey korunmaz: bu geçici çalışma zamanı durumudur).
- Salt okunur işler için `exec`, `restartSafe: true` değerini ayarlayabilir.
  OpenClaw daha sonra yan etkili katalog çağrılarını ve Plugin ad alanlarını
  yürütmeden önce reddeder ve askıya alınmış sonuçları yeniden oynatma açısından
  güvenli olarak işaretler. Yeniden başlatma `wait` işlemini kesintiye uğratırsa
  [yeniden başlatma kurtarması](/tr/gateway/restart-recovery), işlem yerelindeki anlık
  görüntüyü geri yüklemek yerine dönüşü transkriptten yeniden oluşturur. Kurtarma
  dönüşünün kendisi, denetlenmiş salt okunur çekirdek araçları ve açıkça yeniden
  oynatma açısından güvenli Plugin araçlarıyla sınırlı kalır.
- OpenClaw, işlem başına eşzamanlı olarak askıya alınmış çalıştırma sayısını (64)
  sınırlar ve bu sınırı aşan yeni askıya alma işlemlerini `too many suspended code mode
runs.` ile reddeder.

Anlık görüntü depolaması, çalıştırma başına `maxSnapshotBytes`, yukarıdaki işlem
başına askıya alınmış çalıştırma sınırı ve `snapshotTtlSeconds` ile sınırlandırılır.

## QuickJS-WASI çalışma zamanı

OpenClaw, `quickjs-wasi` öğesini sahibi olan pakette doğrudan bağımlılık olarak yükler;
ilgisiz bir bağımlılık için kurulmuş geçişli bir kopyaya dayanmaz.

Çalışma zamanı sorumlulukları: QuickJS-WASI WebAssembly modülünü derlemek/yüklemek;
her kod modu çalıştırması veya sürdürme işlemi için yalıtılmış bir VM oluşturmak;
ana makine geri çağrılarını kararlı adlarla kaydetmek; bellek ve kesme sınırlarını
ayarlamak; JavaScript'i değerlendirmek; bekleyen işleri boşaltmak; askıya alınmış
VM durumunun anlık görüntüsünü almak; `wait` için anlık görüntüleri geri
yüklemek; son durumlardan sonra VM tanıtıcılarını ve anlık görüntüleri elden çıkarmak.

Çalışma zamanı, OpenClaw'ın ana olay döngüsünün dışında bir Node.js iş parçacığında
çalışır. Konuğun sonsuz döngüsü Gateway işlemini süresiz olarak engellememelidir;
iş parçacığının kesme işleyicisi, konuk kodunun iş birliğinden bağımsız olarak
duvar saati zaman aşımını uygular.

## TypeScript

TypeScript desteği yalnızca bir kaynak dönüşümüdür: kabul edilen girdi tek bir
TypeScript kod dizesidir; çıktı, QuickJS-WASI tarafından değerlendirilen bir
JavaScript dizesidir. Tür denetimi, modül çözümleme ve
`import`/`require` yoktur. Tanılamalar `failed` sonuçları
olarak döndürülür.

TypeScript derleyicisi yalnızca TypeScript hücreleri için tembel olarak yüklenir;
düz JavaScript hücreleri ve devre dışı kod modu onu hiçbir zaman yüklemez.

## Güvenlik sınırı

Model kodu düşmancadır. Çalışma zamanı, çok katmanlı savunma kullanır:

- QuickJS-WASI'yi ana olay döngüsünün dışında, bir iş parçacığında çalıştırır
- `quickjs-wasi` öğesini Codex veya geçişli bir paket üzerinden değil,
  doğrudan bağımlılık olarak yükler
- konukta dosya sistemi, ağ, alt işlem, modül içe aktarma, ortam değişkenleri
  veya ana makine genel nesneleri bulunmaz
- QuickJS bellek ve kesme sınırlarının yanı sıra üst işlem duvar saati
  zaman aşımını kullanır
- çıktı, anlık görüntü, günlük ve bekleyen çağrı sınırlarını uygular
- ana makine köprüsü değerlerini dar kapsamlı bir JSON bağdaştırıcısı üzerinden serileştirir
- ana makine hatalarını düz konuk hatalarına dönüştürür; ana makine alanı nesnelerini hiçbir zaman aktarmaz
- zaman aşımı, iptal, oturum sonu veya süre dolumunda anlık görüntüleri bırakır
- `exec`, `wait` ve Araç Arama denetim araçlarına özyinelemeli erişimi reddeder
- kolaylık adı çakışmalarının katalog yardımcılarını gölgelemesini önler

Korumalı alan bir güvenlik katmanıdır; yüksek riskli dağıtımlar için
operatörlerin yine de işletim sistemi düzeyinde sağlamlaştırma yapması gerekebilir.

## Hata kodları

```typescript
type CodeModeErrorCode =
  | "invalid_input"
  | "runtime_unavailable"
  | "timeout"
  | "output_limit_exceeded"
  | "snapshot_limit_exceeded"
  | "internal_error";
```

`invalid_input`; hatalı `exec`/`wait` bağımsız değişkenlerini,
devre dışı dilleri, reddedilen modül erişimini, TypeScript dönüşüm hatalarını,
bilinmeyen/süresi dolmuş/yanlış kapsamlı `runId` değerlerini ve çok fazla
askıya alınmış çalıştırmayı kapsar. `runtime_unavailable`, başlatılamayan veya sıfır
dışında bir kodla çıkan QuickJS iş parçacığını kapsar.

Konuğa döndürülen hatalar düz verilerdir; ana makine `Error` örnekleri,
yığın nesneleri, prototipler ve ana makine işlevleri QuickJS'e geçmez.

## Telemetri

Her sonucun `telemetry` alanı şunları bildirir: gizli katalog boyutu ve kaynak
dağılımı (`openclaw`/`mcp`/`client` sayıları), çalıştırmanın
kataloğu için birikimli arama/açıklama/çağrı sayıları ve model tarafından görülebilen
araç adları (`exec`, `wait` ve korunan yalnızca doğrudan
kullanılan araçlar).

Telemetri; mevcut OpenClaw yörünge politikasının ötesinde gizli bilgileri,
ham ortam değerlerini veya sansürlenmemiş araç girdilerini içermemelidir.

## Hata ayıklama

Kod modu normal bir araç çalıştırmasından farklı davrandığında hedefli model
aktarım günlüklerini kullanın:

```bash
OPENCLAW_DEBUG_CODE_MODE=1 \
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 \
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools \
OPENCLAW_DEBUG_SSE=events \
openclaw gateway
```

Yük şekli hata ayıklaması için `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted` kullanın.
Bu, model isteğinin boyutu sınırlı ve hassas bilgileri çıkarılmış bir JSON anlık görüntüsünü günlüğe kaydeder; istemler ve ileti metni yine de görünebileceğinden bunu yalnızca
hata ayıklama sırasında kullanın.

Akış hata ayıklaması için, hassas bilgileri çıkarılmış ilk beş
SSE olayını günlüğe kaydetmek üzere `OPENCLAW_DEBUG_SSE=peek` kullanın. Kod modu yüzeyi etkinleştirildikten sonra nihai sağlayıcı
yükü tam olarak bir `exec`, bir `wait` ve yalnızca onaylı
yalnızca doğrudan araçları içermiyorsa Kod modu da kapalı biçimde başarısız olur.

## Uygulama düzeni

- yapılandırma sözleşmesi: `tools.codeMode`
- katalog oluşturucu: etkin araçları kompakt girdilere ve kimlik eşlemesine dönüştürür
- model yüzeyi bağdaştırıcısı: görünür araçları kontrol/doğrudan araçlarla değiştirir
- QuickJS-WASI çalışma zamanı bağdaştırıcısı: yükleme, değerlendirme, anlık görüntü oluşturma, geri yükleme, elden çıkarma
- çalışan gözetmeni: zaman aşımı, iptal, çökme yalıtımı
- köprü bağdaştırıcısı: JSON açısından güvenli ana makine geri çağırmaları ve sonuç teslimi
- TypeScript dönüştürme bağdaştırıcısı
- anlık görüntü deposu: TTL, boyut sınırları, çalıştırma/oturum kapsamı
- iç içe araç çağrıları için yörünge izdüşümü
- telemetri sayaçları ve tanılama

Uygulama, Araç Arama'daki katalog ve yürütücü kavramlarını yeniden kullanır ancak
korumalı alan olarak bir `node:vm` alt öğesi kullanmaz.

## Doğrulama kontrol listesi

Kod modu kapsamı şunları kanıtlamalıdır:

- devre dışı yapılandırma, mevcut araç görünürlüğünü değiştirmeden bırakır
- `enabled: true` içermeyen nesne yapılandırması, Kod modunu devre dışı bırakır
- etkin yapılandırma; araçlar çalıştırma için etkinken modele `exec`, `wait` ve yalnızca gerekli, yalnızca doğrudan araçları
  sunar
- ham araçsız çalıştırmalar, `disableTools` ve boş izin listeleri
  Kod modu yükü zorlamasını tetiklemez
- kataloğa uygun tüm etkin MCP dışı araçlar `ALL_TOOLS` içinde görünür
- yalnızca doğrudan araçlar model tarafından görünür kalır ve `ALL_TOOLS` içinde görünmez
- reddedilen araçlar `ALL_TOOLS` içinde görünmez
- `tools.search`, `tools.describe`, `tools.callValue` ve `tools.call` OpenClaw araçları için çalışır
- `API.list("mcp")` ve `API.read("mcp/<server>.d.ts")`, köprü/araç çağrısı olmadan TypeScript tarzı
  MCP bildirimlerini sunar
- MCP ad alanı `$api()`, şemalar için satır içi geri dönüş olarak kullanılabilir kalır
- MCP ad alanı çağrıları, tek nesne girdili görünür MCP araçları için çalışırken
  doğrudan MCP katalog girdileri `tools.*` içinde bulunmaz
- Araç Arama kontrol araçları hem model yüzeyinden hem de
  gizli katalogdan gizlenir
- iç içe çağrılar onay ve kanca davranışını korur
- kabuk `exec`, modelden gizlenir ancak izin verildiğinde katalog kimliğiyle
  çağrılabilir
- özyinelemeli Kod modu `exec` ve `wait`, konuk koddan çağrılamaz
- TypeScript girdisi, devre dışı veya yalnızca JavaScript yollarında TypeScript yüklenmeden
  dönüştürülür ve değerlendirilir
- `import`, `require`, dosya sistemi, ağ ve ortam erişimi başarısız olur
- sonsuz döngüler zaman aşımına uğrar ve Gateway'i engelleyemez
- bellek sınırı hataları konuk VM'yi sonlandırır
- çıktı ve anlık görüntü sınırları tamamlanan ve askıya alınan çağrılar için uygulanır
- `wait`, askıya alınmış bir anlık görüntüyü sürdürür ve nihai değeri döndürür
- süresi dolmuş, iptal edilmiş, yanlış oturuma ait ve bilinmeyen `runId` değerleri başarısız olur
- transkript yeniden oynatma ve kalıcılık, Kod modu kontrol çağrılarını korur
- transkript ve telemetri, iç içe araç çağrılarını açıkça gösterir

## Uçtan uca test planı

Çalışma zamanını değiştirirken bunları entegrasyon veya uçtan uca testler olarak çalıştırın:

1. `tools.codeMode.enabled: false` ile bir Gateway başlatın.
2. Küçük bir doğrudan araç kümesiyle bir aracı turu gönderin.
3. Model tarafından görülebilen araçların değişmediğini doğrulayın.
4. `tools.codeMode.enabled: true` ile yeniden başlatın.
5. OpenClaw, Plugin, MCP ve istemci test araçlarıyla bir aracı turu gönderin.
6. Model tarafından görülebilen araç listesinin `exec`, `wait` ve yalnızca yapılandırılmış
   yalnızca doğrudan araçlardan oluştuğunu doğrulayın.
7. `exec` içinde `ALL_TOOLS` öğesini okuyun ve kataloğa uygun etkin test
   araçlarının bulunduğunu, yalnızca doğrudan araçların ise bulunmadığını doğrulayın.
8. `exec` içinde OpenClaw/Plugin/istemci araçlarını `tools.search`,
   `tools.describe` ve `tools.callValue` (veya ham `tools.call`) üzerinden çağırın.
9. `exec` içinde `API.list("mcp")` ve `API.read("mcp/<server>.d.ts")` öğelerini çağırın ve
   bildirim dosyalarının görünür MCP araçlarını tanımladığını doğrulayın.
10. `exec` içinde MCP araçlarını `MCP.<server>.<tool>({ ...input })` üzerinden çağırın ve
    doğrudan MCP katalog girdilerinin `ALL_TOOLS` ve
    `tools.*` içinde bulunmadığını doğrulayın.
11. Reddedilen araçların bulunmadığını ve tahmin edilen kimlikle çağrılamadığını doğrulayın.
12. `exec`, `waiting` değerini döndürdükten sonra çözümlenen iç içe bir araç çağrısı başlatın.
13. `wait` öğesini çağırın ve geri yüklenen VM'nin araç sonucunu aldığını doğrulayın.
14. Nihai yanıtın geri yükleme sonrasında üretilen çıktıyı içerdiğini doğrulayın.
15. Zaman aşımı, iptal ve anlık görüntü süresinin dolmasının çalışma zamanı durumunu temizlediğini doğrulayın.
16. Yörüngeyi dışa aktarın ve iç içe çağrıların üst
    Kod modu çağrısı altında görünür olduğunu doğrulayın.

Bu sayfadaki yalnızca dokümantasyon değişikliklerinde bile `pnpm check:docs` çalıştırılmalıdır.

## İlgili

- Kod Modu betiklerinden dallanarak aracı düzenleme için [Swarm](/tools/swarm)
- [Araç Arama](/tr/tools/tool-search)
- [Aracı çalışma zamanları](/tr/concepts/agent-runtimes)
- [Exec aracı](/tr/tools/exec)
- [Kod yürütme](/tr/tools/code-execution)
