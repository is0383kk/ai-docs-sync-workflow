---
read_when:
    - Çalışmayı birden fazla aracıya dağıtacak bir Code Mode betiği istiyorsunuz
    - Yapılandırılmış alt sonuçlara, karar geçitlerine veya ilk tamamlanma işlem hatlarına ihtiyacınız var
    - tools.swarm sınırlarını etkinleştiriyor veya ayarlıyorsunuz
    - Oturum panosunda toplayıcı alt süreçlerini gözlemlemek istiyorsunuz
sidebarTitle: Swarm
summary: Yapılandırılmış sonuçlar, sınırlı dağılım ve canlı ilerlemeyle Code Mode betiklerinden eşzamanlı alt ajanları yönetin
title: Sürüü
x-i18n:
    generated_at: "2026-07-27T00:21:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0bec17da7a2e144df35189a65d9b35d829815b545a4bb89652e6a681ca971a9
    source_path: tools/swarm.md
    workflow: 16
---

Swarm, bir [Code Mode](/tools/code-mode) betiğinden çok sayıda alt aracıyı düzenlemek için deneysel ve isteğe bağlı bir yöntemdir. Çalışmaları dağıtmak, sonuçları toplamak ve kararlar almak için `Promise.all`, `while` ve `if` gibi normal JavaScript veya TypeScript kontrol akışlarını kullanın.

Graf DSL'si veya ayrı bir iş akışı biçimi yoktur. Düzenleme programın kendisidir. Swarm bu programa beklenebilir toplayıcı alt öğeler, yapılandırılmış sonuçlar, sınırlı eşzamanlılık ve ilerleme raporlaması ekler.

## Swarm'ı etkinleştirme

Önerilen yol, Control UI içindeki **Ayarlar → Laboratuvarlar → Swarm** seçeneğidir. Anahtar hemen etkinleşir ve yapılandırmanıza `tools.swarm.enabled` yazar.

Swarm'ı doğrudan `openclaw.json` içinde de etkinleştirebilirsiniz:

```json5
{
  tools: {
    swarm: {
      enabled: true,
      maxConcurrent: 8,
      maxChildrenPerGroup: 50,
      maxTotalPerGroup: 200,
      waitTimeoutSecondsMax: 600,
      defaultAgentId: "",
    },
  },
}
```

Boole kısaltması, diğer tüm değerler varsayılanlarında kalacak şekilde özelliği etkinleştirir veya devre dışı bırakır:

```json5
{
  tools: {
    swarm: true,
  },
}
```

| Alan                    | Varsayılan | Açıklama                                                                                                                       |
| ----------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `enabled`               | `false` | Toplayıcı modundaki başlatma seçeneklerini, `agents_wait` ve Code Mode `agents.*` konuk API'sini kullanıma sunar.              |
| `maxConcurrent`         | `8`     | Tek bir swarm grubunda eşzamanlı çalışan azami toplayıcı alt öğe sayısı. Kabul edilen ek alt öğeler FIFO sırasıyla kuyruğa girer. |
| `maxChildrenPerGroup`   | `50`    | Tek bir gruptaki azami canlı toplayıcı alt öğe sayısı.                                                                          |
| `maxTotalPerGroup`      | `200`   | Bir grubun yaşam süresi boyunca başlatabileceği azami toplayıcı alt öğe sayısı. Bu, denetimsiz başlatmaları engelleyen son önlemdir. |
| `waitTimeoutSecondsMax` | `600`   | Tek bir `agents_wait` çağrısının kabul ettiği azami zaman aşımı. Çağrının varsayılanı 30 saniyedir.                              |
| `defaultAgentId`        | `""`    | Bir başlatma işleminde `agentId` belirtilmediğinde kullanılan hedef aracı. Boş değer, istekte bulunan aracıyı kullanır. Mevcut alt aracı izin listeleri geçerlidir. |

Sayısal değerler pozitif tam sayılar olmalıdır. OpenClaw, `maxConcurrent` değerini `1`–`1000`, `maxChildrenPerGroup` değerini `1`–`10000`, `maxTotalPerGroup` değerini `1`–`100000` ve `waitTimeoutSecondsMax` değerini `1`–`86400` aralıklarıyla sınırlar.

Yapılandırılmış tek bir aracı için Swarm'ı `agents.entries.*.tools.swarm` ile geçersiz kılabilirsiniz. Aracıya özgü nesne, üst düzey `tools.swarm` nesnesinin üzerine birleştirilir.

## Gereksinimler

`agents.run`, `phase` ve `log` konuk global değişkenleri hem Swarm'ı hem de OpenClaw Code Mode'u gerektirir:

```json5
{
  tools: {
    codeMode: true,
    swarm: true,
  },
}
```

Code Mode'un ayrıca `sessions_spawn` için etkin erişimi olmalıdır. Araç profilleri, izin verme/reddetme politikası, sağlayıcı kuralları ve korumalı alan politikası bu aracı kaldırabilir. Bir betik `sessions_spawn` kullanılamıyor şeklinde bildirimde bulunursa [Code Mode etkinleştirmesi](/tools/code-mode#activation) ve [Alt aracılar](/tr/tools/subagents) bölümlerine bakın.

`defaultAgentId` ve çalıştırma başına `agentId` değerleri, istekte bulunanın `subagents.allowAgents` politikasının izin verdiği yapılandırılmış bir hedefi adlandırmalıdır. OpenClaw, başka bir aracıya geri dönmek yerine bilinmeyen veya izin verilmeyen bir hedefi reddeder.

## Swarm betiği yazma

Swarm etkinleştirildiğinde Code Mode şu konuk API'sini kullanıma sunar:

```typescript
type AgentRunOptions = {
  label?: string;
  model?: string;
  thinking?: string;
  fastMode?: boolean | "auto";
  agentId?: string;
  schema?: Record<string, unknown>;
  phase?: string;
};

agents.run(prompt: string, options?: AgentRunOptions & { schema?: undefined }): Promise<string>;
agents.run<T>(prompt: string, options: AgentRunOptions & { schema: Record<string, unknown> }): Promise<T>;
phase(title: string): void;
log(message: string): void;
```

`schema` olmadan `agents.run()`, alt öğenin nihai metnine çözümlenir. JSON Schema ile alt öğenin `structured_output` aracı üzerinden gönderdiği değere çözümlenir. Başarısız olan, sonlandırılan, zaman aşımına uğrayan veya şemaya uymayan bir alt öğe, promise'i bir `SwarmAgentError` ile reddeder. Tam olarak oluşturulan bildirimleri ve kısa düzenleme kalıplarını Code Mode içindeki `API.read("agents.d.ts")` kaynağından okuyun.

Kontrol panelinde ve kenar çubuğunda tanınabilir bir alt öğe adı için `label` kullanın. Alt öğe başlamadan hemen önce bir aşama yayımlamak için seçeneklerde `phase` kullanın veya birkaç alt öğe aynı aşamaya aitse `phase()` çağrısını yapın. `log()` kısa bir ilerleme notu yayımlar. İlerleme çağrıları başlatılıp unutulur; kullanıcı arayüzü kullanılamıyorsa betiği geciktirmezler.

### Yapılandırılmış sonuçlarla paralel dağıtma

Bu örnek, her konu için bir araştırmacı başlatır, hepsini bekler ve ardından son bir alt öğeden yapılandırılmış raporlarını sentezlemesini ister:

```javascript
const reportSchema = {
  type: "object",
  properties: {
    finding: { type: "string" },
    evidence: { type: "array", items: { type: "string" } },
    confidence: { type: "number" },
  },
  required: ["finding", "evidence", "confidence"],
  additionalProperties: false,
};

const topics = ["kimlik doğrulama", "depolama", "kurtarma"];
phase("Bağımsız inceleme");

const reports = await Promise.all(
  topics.map((topic) =>
    agents.run(`${topic} yolunu incele. Kanıtlarla birlikte tek bir bulgu döndür.`, {
      label: `inceleme-${topic}`,
      thinking: "high",
      fastMode: "auto",
      schema: reportSchema,
    }),
  ),
);

phase("Sentez");
log(`${reports.length} bağımsız rapor toplandı.`);

return await agents.run(
  `Bu raporları uzlaştır ve anlaşmazlıkları açıkla:\n${JSON.stringify(reports)}`,
  { label: "sentez" },
);
```

`Promise.all`, dağıtma ve birleştirme sınırıdır. OpenClaw grup için en fazla `maxConcurrent` alt öğe başlatır ve kalanları gönderim sırasıyla kuyruğa alır.

Code Mode, eşzamanlı konuk köprüsü çağrılarını ayrıca `tools.codeMode.maxPendingToolCalls` ile sınırlar (varsayılan `16`, azami `128`). Çok büyük gruplarda, bu sınırın altında kalan sınırlı toplu işlemler başlatın ve `phase()`, `log()` ve alt öğe bekleme geçişleri için boşluk bırakın. `maxConcurrent` çalışan alt öğeleri sınırlar; konuk köprüsü çağrı sınırını yükseltmez.

### Karar kapısında döngü oluşturma

Her geçişin başka bir geçiş gerekip gerekmediğine karar verdiği durumlarda sınırlı bir `while` döngüsü kullanın:

```javascript
const gateSchema = {
  type: "object",
  properties: {
    ready: { type: "boolean" },
    reason: { type: "string" },
    nextAction: { type: "string" },
  },
  required: ["ready", "reason", "nextAction"],
  additionalProperties: false,
};

let pass = 0;
let decision = { ready: false, reason: "Kontrol edilmedi", nextAction: "İncele" };

while (!decision.ready && pass < 4) {
  pass += 1;
  phase(`Karar geçişi ${pass}`);
  decision = await agents.run(
    `Sürüm kanıtlarının eksiksiz olup olmadığını kontrol et. Önceki karar: ${JSON.stringify(decision)}`,
    {
      label: `sürüm-kapısı-${pass}`,
      schema: gateSchema,
    },
  );
  log(decision.reason);
}

if (!decision.ready) {
  throw new Error(`${pass} geçişten sonra kapı hâlâ kapalı: ${decision.nextAction}`);
}

return decision;
```

Karar döngülerini her zaman sınırlayın. `maxTotalPerGroup` nihai güvenlik önlemidir; açık bir durma koşulunun yerine geçmez.

### İlk tamamlanan alt öğeyi işleme

`agents.run()` sıradan bir promise döndürdüğü için `Promise.race` ilk Code Mode alt öğesine tepki verebilir. Daha düşük düzeyli araçları çağıran test düzeneklerinde `agents_wait` aynı ilk tamamlanma sınırını sağlar: istenen çalıştırmalardan en az biri tamamlanır tamamlanmaz veya sınırlı zaman aşımı sona erdiğinde döner. Tam boşaltma döngüsü için [Swarm'ı diğer test düzeneklerinden kullanma](#use-swarm-from-other-harnesses) bölümüne bakın.

## Toplayıcı alt öğelerin davranışı

Toplayıcı alt öğeler, farklı bir tamamlanma yoluna sahip sıradan ve yalıtılmış alt aracı oturumlarıdır. Ana oturuma yanıt duyurmak veya yönlendirmek yerine, üst öğenin beklemesi için kalıcı bir toplayıcı sonucu yazarlar.

Hedef aracı şu sırayla çözümlenir:

1. Başlatma veya `agents.run()` çağrısındaki `agentId`.
2. `tools.swarm.defaultAgentId`.
3. İstekte bulunan aracı.

Swarm alt öğelerinin daha küçük bir araç yüzeyine, daha ucuz bir modele veya daha sıkı bir korumalı alan politikasına ihtiyaç duyduğu durumlarda özel, yalın bir çalışan aracı yararlıdır. OpenClaw yerleşik bir `worker` aracı kimliğiyle gelmez; bunu varsayılan olarak adlandırmadan önce yapılandırın. Bu çalışanı, başlatılabilmesi ancak kendi üst düzey oturumlarından swarm başlatamaması için aracıya özgü yapılandırmasında `tools.swarm: false` ile sıkılaştırın:

```json5
{
  tools: { swarm: { enabled: true, defaultAgentId: "worker" } },
  agents: {
    list: [
      {
        id: "main",
        default: true,
        subagents: { allowAgents: ["worker"] },
      },
      { id: "worker", tools: { swarm: false } },
    ],
  },
}
```

Toplayıcı onayları güvenli biçimde başarısız olur. Bir alt öğe hiçbir zaman operatör onay istemi açmaz. Onay gerektirecek bir araç eylemi reddedilir ve alt öğe bu reddi sonucunda bildirebilir; böylece betik bir sonraki adımda ne yapılacağına karar verebilir.

OpenClaw, yapılandırılmış çıktı için alt öğeye sentetik bir `structured_output` aracı ekler ve yükünü sağlanan JSON Schema'ya göre doğrular. Geçersiz veya eksik bir yük için bir düzeltme uyarısı verilir. Yeniden deneme de doğrulanmazsa toplayıcı tamamlanması alt öğenin ham metnini korur, `structured` değerini ayarlamadan bırakır ve `schemaError` içerir. Düşük düzeyli `agents_wait` sonucu, açık kurtarma mantığı için bu alanları kullanıma sunar.

### Alt öğeler yapraktır

Swarm alt öğeleri varsayılan olarak yapraktır. Evrensel `agents.defaults.subagents.maxSpawnDepth` koruması, bir alt öğenin varsayılan `1` derinliğinde kendi alt öğelerini başlatmasını önler. Olağan düzenleme kalıbı, bir alt öğeden daha fazla çalışma başlatmak değil, çalışmayı üst öğeye döndürmektir:

```javascript
const plan = await agents.run("Bu işi bağımsız görevler olarak planla.", {
  schema: {
    type: "object",
    properties: { tasks: { type: "array", items: { type: "string" } } },
    required: ["tasks"],
    additionalProperties: false,
  },
});
return await Promise.all(plan.tasks.map((task) => agents.run(task)));
```

İç içe alt aracılar, `agents.defaults.subagents.maxSpawnDepth` üzerinden operatörün isteğe bağlı olarak etkinleştirdiği bir özelliktir ve Swarm için önerilmez. Grup sınırları, bütçeler ve gözlemlenebilirlik düz toplayıcı grupları varsayar.

Her alt öğenin bir kabul sahibi vardır. Duyuru ve etkileşimli alt öğeler `agents.defaults.subagents.maxChildrenPerAgent` (varsayılan `5`) kullanır ve toplayıcı alt öğeleri saymaz. Toplayıcı alt öğeler yalnızca `maxChildrenPerGroup` ve `maxTotalPerGroup` kullanır; oturum başına alt öğe bütçesini tüketmezler. Başlatma derinliği koruması her iki mod için de geçerlidir.

Kabulden sonra `maxConcurrent` üzerindeki alt öğeler, global alt aracı şeridinin içinde kendi swarm grupları dâhilinde FIFO sırasıyla kuyruğa girer. Bu eşzamanlılık katmanları çalışmayı reddetmek yerine kuyruğa alır. Grup sınırlarından herhangi birini aşan bir toplayıcı başlatma işlemi, hatadaki ilgili yapılandırma anahtarıyla reddedilir.

## Swarm'ı gözlemleme

Bir swarm etkinken üst oturumun kontrol panelini Control UI içinde açın. Swarm pencere öğesi, her etkin toplayıcı grubunu alt öğe başına bir nokta olarak; kuyrukta, çalışıyor, tamamlandı veya başarısız durumuyla görüntüler. Etiketler nokta araç ipuçlarında görünür; bu nedenle kısa ve kararlı etiketler daha büyük swarm'ların okunmasını kolaylaştırır.

Oturum kenar çubuğu normal üst/alt öğe ağacını korur. Swarm hiyerarşisini kaybetmeden bir toplayıcı alt öğeyi incelemek veya dökümünü açmak için üst öğe satırını genişletin.

Toplayıcı sonuçları, grupları arşivlenene kadar beklenebilir durumda kalır. Her
üye saklama süresinin sonuna ulaştıktan sonra OpenClaw, tamamlanmış sürülerin
canlı oturum ağacında kalmaması için grubun alt öğelerini toplu olarak arşivler.

## Swarm'ı diğer yürütme ortamlarından kullanma

Swarm'ı OpenClaw Code Mode olmadan kullanabilirsiniz. Temel araçları yürütme
ortamından bağımsızdır: toplayıcı alt öğelerini `sessions_spawn({ collect: true })` ile başlatın
ve sınırlı `agents_wait` çağrılarıyla sonuçlarını alın.

Codex Code Mode, uygun dinamik OpenClaw araçlarını otomatik olarak
`tools.*` altında kullanıma sunar. OpenClaw'ın QuickJS konuk API'sini
kullanmaz veya `tools.codeMode` gerektirmez; ancak `tools.swarm` yine de
etkinleştirilmiş olmalıdır. Codex yürütme ortamının `agents_wait` çağrıları
600 saniyelik zaman aşımının tamamını destekler.

Şu anda desteklenen Codex çalışma zamanında, dinamik OpenClaw araç sonuçları
Code Mode'a JSON metni olarak ulaşır. Alanları okumadan önce her sonucu ayrıştırın.
Codex ayrıca dinamik araç çağrılarını seri hâle getirir; bu nedenle
`Promise.all`, birden fazla `sessions_spawn` çağrısını eşzamanlı olarak
göndermez. Toplayıcıları sınırlı bir döngüde başlatın; daha sonraki başlatmalar
gönderilirken önceden kabul edilmiş alt öğeler çalışmaya devam edebilir.

```javascript
function parseToolResult(value) {
  if (typeof value !== "string") return value;
  return JSON.parse(value);
}

const tasks = [
  "Kimlik doğrulama yolunu kontrol edin.",
  "Depolama yolunu kontrol edin.",
  "Kurtarma yolunu kontrol edin.",
];
const launches = [];

for (const [index, task] of tasks.entries()) {
  const launch = parseToolResult(
    await tools.sessions_spawn({
      task,
      collect: true,
      label: `review-${index + 1}`,
    }),
  );
  if (launch.status !== "accepted") {
    throw new Error(launch.error ?? "Toplayıcının başlatılması kabul edilmedi.");
  }
  launches.push(launch);
}

const pending = new Set(launches.map((launch) => launch.runId));
const completed = [];

while (pending.size > 0) {
  const ids = [...pending].slice(0, 1000);
  const batch = parseToolResult(
    await tools.agents_wait({
      ids,
      timeoutSeconds: 30,
    }),
  );

  // Bu sınırlı pencereyi henüz kontrol edilmemiş kimliklerin arkasına döndürün.
  for (const runId of ids) {
    if (pending.delete(runId)) pending.add(runId);
  }

  for (const item of batch.completed) {
    pending.delete(item.runId);
    if (item.status !== "done") {
      throw new Error(item.schemaError ?? item.result ?? `${item.runId}: ${item.status}`);
    }
    completed.push(item); // Her sonucu tamamlanır tamamlanmaz işleyin.
  }

  for (const failure of batch.errors ?? []) {
    pending.delete(failure.runId);
    throw new Error(`${failure.runId}: ${failure.error}`);
  }
}

return completed;
```

Her `agents_wait` çağrısı 1–1000 çalıştırma kimliği kabul eder. Şunu döndürür:

```typescript
type AgentsWaitResult = {
  completed: Array<{
    runId: string;
    status: "done" | "failed" | "killed" | "timeout";
    result: string;
    structured?: unknown;
    schemaError?: string;
    sessionKey: string;
    label?: string;
    usage?: { inputTokens: number; outputTokens: number };
  }>;
  pending: string[];
  errors?: Array<{
    runId: string;
    error: "not_found" | "not_owner";
  }>;
};
```

İstenen alt öğelerden herhangi biri zaten tamamlanmışsa, bekleyen alt öğelerden
en az biri tamamlandığında, geçerli bekleyen kimlik kalmadığında veya zaman aşımı
dolduğunda çağrı hemen döner. Tamamlanmış kayıtlar idempotenttir; dolayısıyla
zaten tamamlanmış bir çalıştırma kimliğini geçirmek, sonucunu yeniden döndürür.
Yalnızca başlatan oturum veya onun yetkilendirilmiş üst zinciri bir toplayıcıyı
bekleyebilir.

Bu, yoğun bir durum döngüsü değil, sınırlı uzun yoklamadır. `pending`
boşalana kadar yalnızca kalan çalıştırma kimliklerini geçirmeye devam edin.
Toplayıcı modu, yerel OpenClaw alt ajanlarını destekler; ACP çalışma zamanını,
iş parçacığı bağlamayı, görünür oturumları veya kalıcı oturum modunu desteklemez.

## Sınırlar ve yol haritası

Swarm v1, tek seferlik toplayıcı alt öğeleri çalıştırır; planlanan
`agents.session()` API'si durum bilgisi taşıyan çok turlu çalışanlar ekleyecektir.
Alt öğeler şu anda yerel Gateway'in alt ajan hattında çalışır; bulut yerleşimi,
açık bir başlatma seçeneği olarak planlanmaktadır. Kaydedilmiş iş akışı tanımları
ve bir grafik DSL'si, Swarm'ın mevcut yöneliminin parçası değildir.

## İlgili

- [Code Mode](/tools/code-mode): QuickJS konuk çalışma zamanı ve etkinleştirme kuralları için
- [Alt ajanlar](/tr/tools/subagents): alt öğe politikası, yalıtım ve oturum davranışı için
- [Çok ajanlı korumalı alan araçları](/tr/tools/multi-agent-sandbox-tools): ajan başına kısıtlamalar için
- [Araçlara genel bakış](/tr/tools): araç profilleri ve politika yönlendirmesi için
