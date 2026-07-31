---
read_when:
    - OpenClaw ajanlarının her araç şemasını isteme eklemeden geniş bir araç kataloğu kullanmasını istiyorsunuz
    - OpenClaw araçlarının, MCP araçlarının ve istemci araçlarının tek bir kompakt çalışma zamanı yüzeyi üzerinden kullanıma sunulmasını istiyorsunuz
    - OpenClaw çalıştırmaları için araç keşfini uyguluyor veya hata ayıklıyorsunuz
summary: 'Araç Arama: büyük OpenClaw araç kataloglarını arama, açıklama ve çağırma işlevlerinin arkasında kompakt hâle getirin'
title: Araç Arama
x-i18n:
    generated_at: "2026-07-26T23:40:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d31322d5ef108c52fd14d48771cc3c6c43fcfbc4bfb95652bc29a55fd706c903
    source_path: tools/tool-search.md
    workflow: 16
---

Araç Arama, deneysel bir OpenClaw ajan çalışma zamanı özelliğidir. Ajanlara büyük
araç kataloglarını keşfetmek ve çağırmak için tek ve kompakt bir yol sunar. Çalıştırmada
çok sayıda kullanılabilir araç olduğunda, ancak modelin bunlardan yalnızca birkaçına ihtiyaç
duyması beklendiğinde kullanışlıdır.

Bu sayfa OpenClaw Araç Arama özelliğini belgeler. Codex'e özgü araç
arama veya dinamik araçlar yüzeyi değildir. Codex'e özgü kod modu, araç arama, ertelenmiş
dinamik araçlar ve iç içe araç çağrıları kararlı Codex çalıştırma altyapısı yüzeyleridir ve
`tools.toolSearch` öğesine bağlı değildir.

Araç Arama denetimleri yerine QuickJS-WASI `exec`/`wait`
yüzeyi sunan genel OpenClaw çalışma zamanı için [Kod Modu](/tools/code-mode) bölümüne bakın.

OpenClaw çalıştırmaları için etkinleştirildiğinde model, varsayılan olarak tek bir
`tool_search_code` aracı ile yapılandırılmış sonuçları kompakt köprüden geçemeyen tüm
yalnızca doğrudan araçları alır. Kod aracı, yalıtılmış bir Node alt sürecinde
`openclaw.tools` köprüsüyle kısa bir JavaScript gövdesi çalıştırır:

```js
const hits = await openclaw.tools.search("create a GitHub issue");
const tool = await openclaw.tools.describe(hits[0].id);
return await openclaw.tools.call(tool.id, {
  title: "Crash on startup",
  body: "Steps to reproduce...",
});
```

Katalog; kataloğa uygun OpenClaw araçlarını, plugin araçlarını, MCP
araçlarını ve istemci tarafından sağlanan araçları içerebilir. Model, kataloglanan her şemayı
başlangıçta görmez. Bunun yerine kompakt tanımlayıcılarda arama yapar, tam şemaya ihtiyaç
duyduğunda seçilen bir aracı açıklar ve bu aracı OpenClaw üzerinden çağırır.
Yalnızca doğrudan araçlar model tarafından görünür kalır ve kataloğa eklenmez.

Codex çalıştırma altyapısı çalıştırmaları bu deneysel OpenClaw Araç Arama
denetimlerini almaz. OpenClaw, ürün yeteneklerini Codex'e dinamik araçlar olarak aktarır;
kararlı yerel kod modu, yerel araç arama, ertelenmiş dinamik
araçlar ve iç içe araç çağrılarının sahibi Codex'tir.

## Bir turun çalışma şekli

Planlama sırasında OpenClaw yerleşik çalıştırıcısı, çalıştırma için geçerli kataloğu
oluşturur:

1. Ajan, profil, korumalı alan ve oturum için etkin araç politikasını çözümleyin.
2. Uygun OpenClaw ve plugin araçlarını listeleyin.
3. Oturum MCP çalışma zamanı üzerinden uygun MCP araçlarını listeleyin.
4. Geçerli çalıştırma için sağlanan uygun istemci araçlarını ekleyin.
5. Yalnızca doğrudan araçları model tarafından görünür tutun ve
   kataloğa uygun kalan araçların kompakt tanımlayıcılarını dizine ekleyin.
6. OpenClaw kod köprüsünü, yapılandırılmış geri dönüş araçlarını veya
   kompakt dizin yüzeyini bu yalnızca doğrudan araçların yanında sunun.

Yürütme sırasında her gerçek araç çağrısı OpenClaw'a geri döner. Yalıtılmış Node
çalışma zamanı plugin uygulamalarını, MCP istemci nesnelerini veya gizli bilgileri
barındırmaz. `openclaw.tools.call(...)`, köprüden Gateway'e geri geçer; burada
normal politika, onay, kanca, günlükleme ve sonuç işleme süreçleri uygulanmaya devam eder.

## Modlar

`tools.toolSearch`, modele yönelik üç moda sahiptir:

- `code`: yalnızca doğrudan araçların yanında varsayılan kompakt JavaScript
  köprüsü olan `tool_search_code` öğesini sunar.
- `tools`: kod almaması gereken sağlayıcılar için yalnızca doğrudan
  araçların yanında `tool_search`, `tool_describe` ve `tool_call` öğelerini düz
  yapılandırılmış araçlar olarak sunar.
- `directory`: her tam şemayı görmeden araç adlarını görmesi gereken
  sağlayıcılar için `tool_search`, `tool_describe` ve `tool_call` öğelerinin yanı sıra
  kullanılabilir araç adları ve açıklamalarından oluşan sınırlı bir istem dizini sunar.
  OpenClaw ayrıca geçerli tur için muhtemel veya gerekli araç şemalarından oluşan küçük,
  sınırlı bir kümeyi doğrudan sunabilir. Yalnızca doğrudan araçlar bu modda da görünür kalır.

Tüm modlar, aynı politika filtreli kataloğu ve normal OpenClaw yürütme
yolunu kullanır. `catalogMode: "direct-only"` olarak işaretlenen araçlar bu kataloğun dışında
kalır ve model tarafından görünür olmaya devam eder. Geçerli çalışma zamanı yalıtılmış Node
kod modu alt sürecini başlatamıyorsa varsayılan `code` modu, katalog
sıkıştırmasından önce `tools` moduna geri döner. `directory` modunda,
OpenClaw araçları, plugin araçları ve MCP araçları dizin kataloğunun arkasında
sıkıştırılabilirken istemci tarafından sağlanan araçlar geçerli çalıştırma için doğrudan görünür
kalır. Tam bir gizli dizin adına yapılan doğrudan çağrı, yürütmeden önce aynı yetkili
katalogdan yüklenir.

Tüm modlar deneyseldir. Küçük OpenClaw araç katalogları için doğrudan araç sunumunu,
Codex çalıştırma altyapısı çalıştırmaları içinse Codex'e özgü kararlı yüzeyleri tercih edin.

Ayrı bir kaynak seçimi yapılandırması yoktur. Araç Arama etkinleştirildiğinde
katalog, normal politika filtrelemesinden sonra kataloğa uygun OpenClaw, MCP ve istemci
araçlarını içerir; yalnızca doğrudan araçlar ayrı tutulur.

## Neden var?

Büyük kataloglar kullanışlıdır ancak maliyetlidir. Her araç şemasını modele göndermek
isteği büyütür, planlamayı yavaşlatır ve yanlışlıkla araç
seçilmesi olasılığını artırır.

Araç Arama yapıyı değiştirir:

- doğrudan araçlar: model, ilk token'dan önce seçilen her şemayı görür
- Araç Arama kod modu: model tek bir kompakt kod aracını, kısa bir API
  sözleşmesini ve tüm yalnızca doğrudan araçları görür
- Araç Arama araç modu: model üç kompakt yapılandırılmış geri dönüş
  aracını ve tüm yalnızca doğrudan araçları görür
- Araç Arama dizin modu: model sınırlı bir dizini,
  arama/açıklama/çağrı denetimlerini, muhtemel veya gerekli şemalardan oluşan küçük ve
  sınırlı bir kümeyi ve tüm yalnızca doğrudan araçları görür
- tur sırasında: model kalan şemaları gerektiğinde yükleyebilir

Doğrudan araç sunumu, küçük kataloglar için hâlâ doğru varsayılandır. Araç Arama,
özellikle MCP sunucularından veya istemci tarafından sağlanan uygulama araçlarından
çok sayıda aracın tek bir çalıştırmada görülebildiği durumlarda en uygundur.

## API

`openclaw.tools.search(query, options?)`

Geçerli çalıştırmanın etkin kataloğunda arama yapar. Sonuçlar kompakttır ve istem
bağlamına güvenle geri eklenebilir. Her eşleşme, `{ id: string; mode?: "drip" | "flood" }` gibi sınırlı,
TypeScript tarzında bir `input` imzası içerir; böylece bu imza yeterliyse
model `describe` işlemini atlayabilir. Güvenilir bir OpenClaw çekirdek veya plugin
aracı, `Array<{ id: string; paid: boolean }>` gibi kompakt bir `output` ipucu da içerebilir.
MCP ve istemci çıktı şeması iddiaları bu güvenilir ipucuna yükseltilmez.
Güvenilmeyen giriş şemaları da `input: "unknown"` olarak ertelenir; bunları çağırmadan
önce `describe` kullanın. Açık, aşırı büyük veya başka bir şekilde kısmi çıktı
şemaları ipucunu içermez ve bunun yerine `describe` üzerinden kullanılabilir
olmaya devam eder.

```js
const hits = await openclaw.tools.search("calendar event", { limit: 5 });
```

`openclaw.tools.describe(id)`

Tam giriş şeması ve araç tarafından bildirilmişse güvenilir tam `outputSchema`
dâhil olmak üzere bir arama sonucunun tüm meta verilerini yükler.

```js
const calendarCreate = await openclaw.tools.describe("mcp:calendar:create_event");
```

`openclaw.tools.call(id, args)`

Seçilen bir aracı OpenClaw üzerinden çağırır ve ham `{ tool, result }`
zarfını döndürür. JSON döndüren araçlar normalde değerlerini
`result.details` içine yerleştirir. Güvenilir bir araç `outputSchema` bildirirse OpenClaw,
yürütmeden önce şemayı derler ve katalog çağrısını döndürmeden önce normal araç
kancalarının ardından nihai `details` değerini doğrular.

```js
await openclaw.tools.call(calendarCreate.id, {
  summary: "Planning",
  start: "2026-05-09T14:00:00Z",
});
```

Araç yazarları çıktı sözleşmelerini aracın `outputSchema` özelliğinde bildirir.
Bu özellik, işlenmiş içerik bloklarını değil `AgentToolResult.details` öğesini açıklar.
Hata oluşturmayan tüm varyantları dâhil edin veya kararsız sonuçlar için bunu atlayın.
[Kod Modu çıktı sözleşmeleri](/tools/code-mode#declared-output-contracts) ve
[Araç pluginleri](/tr/plugins/tool-plugins#output-contracts) bölümlerine bakın.

Yapılandırılmış geri dönüş modu, aynı işlemleri araçlar olarak sunar:

- `tool_search`
- `tool_describe`
- `tool_call`

Dizin modu şunları sunar:

- `tool_search`
- `tool_describe`
- `tool_call`

Ayrıca istemci tarafından sağlanan araçları ve tüm yalnızca doğrudan araçları doğrudan görünür
tutar; geçerli tur için muhtemel veya gerekli katalog araç şemalarından oluşan küçük,
sınırlı bir kümeyi de doğrudan sunabilir. Sınırlı dizinde bazı girdiler yoksa
bunları bulmak için `tool_search` kullanın. Model tam bir gizli dizin aracı adını
doğrudan isterse OpenClaw, normal yürütmeden önce bu aracı yetkili katalogdan yükler.
Tam ertelenmiş gönderim bu adları kullandığından dizin modundaki istemci aracı adları
OpenClaw, plugin veya MCP araç adlarıyla çakışmamalıdır.

## Çalışma zamanı sınırı

Kod köprüsü kısa ömürlü bir Node alt sürecinde çalışır. Alt süreç;
Node izin modu etkin, boş bir ortamla, dosya sistemi veya ağ izni olmadan ve
alt süreç ya da worker izni olmadan başlar. OpenClaw, üst süreçte gerçek zamanlı
bir zaman aşımı uygular ve zaman uyumsuz devam işlemlerinden sonra bile zaman aşımında
alt süreci sonlandırır.

Çalışma zamanı yalnızca şunları sunar:

- `console.log`, `console.warn` ve `console.error`
- `openclaw.tools.search`
- `openclaw.tools.describe`
- `openclaw.tools.call`

Normal OpenClaw davranışı nihai çağrılar için uygulanmaya devam eder:

- araç izin ve reddetme politikaları
- ajan ve korumalı alan başına araç kısıtlamaları
- kanal/çalışma zamanı araç politikası
- onay kancaları
- plugin `before_tool_call` kancaları
- oturum kimliği, günlükler ve telemetri

## Yapılandırma

OpenClaw çalıştırmaları için Araç Arama'yı varsayılan kod köprüsüyle etkinleştirin:

```bash
openclaw config set tools.toolSearch true
```

Eşdeğer JSON:

```json5
{
  tools: {
    toolSearch: true,
  },
}
```

OpenClaw çalıştırmaları için bunun yerine yapılandırılmış geri dönüş araçlarını kullanın:

```json5
{
  tools: {
    toolSearch: {
      mode: "tools",
    },
  },
}
```

OpenClaw çalıştırmaları için bunun yerine kompakt dizin yüzeyini kullanın:

```json5
{
  tools: {
    toolSearch: {
      mode: "directory",
    },
  },
}
```

Kod modu zaman aşımını ve arama sonucu sınırlarını ayarlayın (gösterilen değerler varsayılanlardır):

```json5
{
  tools: {
    toolSearch: {
      mode: "code",
      codeTimeoutMs: 10000,
      searchDefaultLimit: 8,
      maxSearchLimit: 20,
    },
  },
}
```

Çalışma zamanı `codeTimeoutMs` değerini 1000-60000, `maxSearchLimit` değerini 1-50 ve
`searchDefaultLimit` değerini 1..`maxSearchLimit` aralığıyla sınırlar.

Devre dışı bırakın:

```json5
{
  tools: {
    toolSearch: false,
  },
}
```

## İstem ve telemetri

Araç Arama, doğrudan araç sunumuyla karşılaştırma yapmaya yetecek kadar telemetri kaydeder:

- çalıştırma altyapısına gönderilen toplam serileştirilmiş araç ve istem baytı
- katalog boyutu ve kaynak dağılımı
- arama, açıklama ve çağrı sayıları
- OpenClaw üzerinden yürütülen nihai araç çağrıları
- seçilen araç kimlikleri ve kaynakları

Oturum günlükleri şu soruların yanıtlanmasını mümkün kılmalıdır:

- modelin başlangıçta kaç araç şeması gördüğü
- kaç arama ve açıklama işlemi gerçekleştirdiği
- hangi nihai aracın çağrıldığı
- sonucun OpenClaw, MCP veya bir istemci aracından gelip gelmediği

## E2E doğrulaması

QA Lab Gateway senaryosu, OpenClaw çalışma zamanı ile her iki yolu da doğrular:

```bash
pnpm openclaw qa suite --provider-mode mock-openai --scenario tool-search-gateway-e2e
```

Büyük bir araç kataloğuna sahip geçici bir sahte plugin oluşturur, sahte
OpenAI sağlayıcısını başlatır, Gateway'i bir kez doğrudan modda ve bir kez Araç Arama
etkin olarak başlatır, ardından sağlayıcı istek yüklerini ve oturum günlüklerini karşılaştırır.

Regresyon şunları doğrular:

1. Doğrudan mod, sahte plugin aracını çağırabilir.
2. Araç Arama, aynı sahte plugin aracını çağırabilir.
3. Doğrudan mod, sahte plugin aracı şemalarını doğrudan sağlayıcıya sunar.
4. Araç Arama, yalnızca kompakt köprüyü ve doğrudan moda özel araçları sunar.
5. Araç Arama isteği yükü, büyük sahte katalog için daha küçüktür.
6. Oturum günlükleri, beklenen araç çağrısı sayılarını ve köprülenen çağrı telemetrisini gösterir.

## Hata davranışı

Araç Arama, hata durumunda erişimi engellemelidir:

- bir araç etkin politikada yer almıyorsa arama bu aracı döndürmemelidir
- seçilen bir araç kullanılamaz hâle gelirse `tool_call` başarısız olmalıdır
- politika veya onay yürütmeyi engelliyorsa çağrı sonucu, engeli atlamak yerine
  bu engeli bildirmelidir
- kod köprüsü yalıtılmış bir çalışma zamanı oluşturamıyorsa `mode: "tools"` kullanın veya
  bu dağıtım için Araç Arama'yı devre dışı bırakın

## İlgili

- [Araçlar ve pluginler](/tr/tools)
- [Çok aracılı korumalı alan ve araçlar](/tr/tools/multi-agent-sandbox-tools)
- [Exec aracı](/tr/tools/exec)
- [ACP aracılarını ayarlama](/tr/tools/acp-agents-setup)
- [Plugin geliştirme](/tr/plugins/building-plugins)
