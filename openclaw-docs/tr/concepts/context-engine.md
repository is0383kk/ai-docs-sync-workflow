---
read_when:
    - OpenClaw'ın model bağlamını nasıl oluşturduğunu anlamak istiyorsunuz
    - Eski motor ile bir plugin motoru arasında geçiş yapıyorsunuz
    - Bir bağlam motoru plugini oluşturuyorsunuz
sidebarTitle: Context engine
summary: 'Bağlam motoru: takılabilir bağlam oluşturma, Compaction ve alt ajan yaşam döngüsü'
title: Bağlam motoru
x-i18n:
    generated_at: "2026-07-26T23:37:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 721780790dacebec44e3c7540b225bd853ee66bf5ae066b84df4344614d93a62
    source_path: concepts/context-engine.md
    workflow: 16
---

Bir **bağlam motoru**, OpenClaw'ın her çalıştırma için model bağlamını nasıl oluşturduğunu denetler: hangi iletilerin dahil edileceği, eski geçmişin nasıl özetleneceği ve alt ajan sınırları arasında bağlamın nasıl yönetileceği.

OpenClaw, yerleşik bir `legacy` motoruyla gelir ve varsayılan olarak bunu kullanır. Yalnızca farklı bir birleştirme, Compaction veya oturumlar arası hatırlama davranışı istendiğinde bir Plugin motoru kurup seçin.

## Hızlı başlangıç

<Steps>
  <Step title="Hangi motorun etkin olduğunu denetleyin">
    ```bash
    openclaw doctor
    # veya yapılandırmayı doğrudan inceleyin:
    cat ~/.openclaw/openclaw.json | jq '.plugins.slots.contextEngine'
    ```
  </Step>
  <Step title="Bir Plugin motoru kurun">
    Bağlam motoru Pluginleri, diğer tüm OpenClaw Pluginleri gibi kurulur.

    <Tabs>
      <Tab title="npm üzerinden">
        ```bash
        openclaw plugins install @martian-engineering/lossless-claw
        ```
      </Tab>
      <Tab title="Yerel bir yoldan">
        ```bash
        openclaw plugins install -l ./my-context-engine
        ```
      </Tab>
    </Tabs>

  </Step>
  <Step title="Motoru etkinleştirip seçin">
    ```json5
    // openclaw.json
    {
      plugins: {
        slots: {
          contextEngine: "lossless-claw", // Pluginin kayıtlı motor kimliğiyle eşleşmelidir
        },
        entries: {
          "lossless-claw": {
            enabled: true,
            // Plugine özgü yapılandırma buraya yazılır (Plugin belgelerine bakın)
          },
        },
      },
    }
    ```

    Kurulum ve yapılandırmadan sonra Gateway'i yeniden başlatın.

  </Step>
  <Step title="Eski motora geri dönün (isteğe bağlı)">
    `contextEngine` değerini `"legacy"` olarak ayarlayın (veya anahtarı tamamen kaldırın; varsayılan değer `"legacy"` değeridir).
  </Step>
</Steps>

## Nasıl çalışır?

OpenClaw bir model istemini her çalıştırdığında bağlam motoru, yaşam döngüsünün dört noktasına katılır:

<AccordionGroup>
  <Accordion title="1. Alma">
    Oturuma yeni bir ileti eklendiğinde çağrılır. Motor, iletiyi kendi veri deposunda saklayabilir veya dizine ekleyebilir.
  </Accordion>
  <Accordion title="2. Birleştirme">
    Her model çalıştırmasından önce çağrılır. Motor, belirteç bütçesine sığan sıralı bir ileti kümesi (ve isteğe bağlı bir `systemPromptAddition`) döndürür.
  </Accordion>
  <Accordion title="3. Compaction">
    Bağlam penceresi dolduğunda veya kullanıcı `/compact` komutunu çalıştırdığında çağrılır. Motor, alan açmak için eski geçmişi özetler.
  </Accordion>
  <Accordion title="4. Tur sonrası">
    Bir çalıştırma tamamlandıktan sonra çağrılır. Motor durumu kalıcı hâle getirebilir, arka planda Compaction tetikleyebilir veya dizinleri güncelleyebilir.
  </Accordion>
</AccordionGroup>

Motorlar ayrıca önyüklemeden, başarılı bir turdan veya Compaction işleminden sonra transkript bakımı (`runtimeContext.rewriteTranscriptEntries()` aracılığıyla güvenli yeniden yazma) için isteğe bağlı bir `maintain()` yöntemi uygulayabilir. Yanıtı engellemek yerine ertelenmiş iş olarak çalıştırmak için `info.turnMaintenanceMode: "background"` değerini ayarlayın.

Paketle birlikte gelen ACP dışı Codex koşum takımı için OpenClaw, birleştirilmiş bağlamı Codex geliştirici talimatlarına ve mevcut turun istemine yansıtarak aynı yaşam döngüsünü uygular. Codex, yerel iş parçacığı geçmişini ve yerel Compaction bileşenini yönetmeye devam eder.

### Alt ajan yaşam döngüsü (isteğe bağlı)

OpenClaw, isteğe bağlı iki alt ajan yaşam döngüsü kancası çağırır:

<ParamField path="prepareSubagentSpawn" type="method">
  Alt çalıştırma başlamadan önce paylaşılan bağlam durumunu hazırlar. Kanca; üst/alt oturum anahtarlarını, `contextMode` (`isolated` veya `fork`), kullanılabilir transkript kimliklerini/dosyalarını ve isteğe bağlı TTL değerini alır. Bir geri alma tanıtıcısı döndürürse OpenClaw, hazırlık başarılı olduktan sonra başlatma başarısız olduğunda bunu çağırır. `lightContext` isteyen ve `contextMode="isolated"` olarak çözümlenen yerel alt ajan başlatmaları, alt çalıştırmanın bağlam motoru tarafından yönetilen başlatma öncesi durum olmadan hafif önyükleme bağlamından başlaması için bu kancayı kasıtlı olarak atlar.
</ParamField>
<ParamField path="onSubagentEnded" type="method">
  Bir alt ajan oturumu tamamlandığında veya süpürüldüğünde temizleme yapar.
</ParamField>

### Sistem istemine ekleme

`assemble` yöntemi bir `systemPromptAddition` dizesi döndürebilir. OpenClaw bunu çalıştırmanın sistem isteminin başına ekler. Böylece motorlar, statik çalışma alanı dosyaları gerektirmeden dinamik hatırlama yönlendirmeleri, erişim talimatları veya bağlama duyarlı ipuçları ekleyebilir.

## Eski motor

Yerleşik `legacy` motoru, OpenClaw'ın özgün davranışını korur:

- **Alma**: işlem yapmaz (ileti kalıcılığını doğrudan oturum yöneticisi işler).
- **Birleştirme**: değişiklik yapmadan geçirir (çalışma zamanındaki mevcut temizle → doğrula → sınırla işlem hattı bağlam birleştirmeyi işler).
- **Compaction**: eski iletilerin tek bir özetini oluşturan ve son iletileri olduğu gibi koruyan yerleşik özetleme Compaction işlemine devreder.
- **Tur sonrası**: işlem yapmaz.

Eski motor araç kaydetmez veya bir `systemPromptAddition` sağlamaz.

Hiçbir `plugins.slots.contextEngine` ayarlanmadığında (veya `"legacy"` olarak ayarlandığında) bu motor otomatik olarak kullanılır.

## Plugin motorları

Bir Plugin, Plugin API'sini kullanarak bir bağlam motoru kaydedebilir:

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function register(api) {
  api.registerContextEngine("my-engine", (ctx) => ({
    info: {
      id: "my-engine",
      name: "My Context Engine",
      ownsCompaction: true,
    },

    async ingest({ sessionId, message, isHeartbeat }) {
      // İletiyi veri deponuzda saklayın
      return { ingested: true };
    },

    async assemble({
      sessionId,
      sessionKey,
      messages,
      tokenBudget,
      availableTools,
      citationsMode,
    }) {
      // Bütçeye sığan iletileri döndürün
      return {
        messages: buildContext(messages, tokenBudget),
        estimatedTokens: countTokens(messages),
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },

    async compact({ sessionId, force }) {
      // Eski bağlamı özetleyin
      return { ok: true, compacted: true };
    },
  }));
}
```

`ctx` fabrikası, Pluginlerin ilk yaşam döngüsü çağrısından önce
ajan veya çalışma alanı başına durumu başlatabilmesi için isteğe bağlı `config`, `agentDir` ve `workspaceDir`
değerlerini içerir. Eski olmayan bir `assemble()` çağrısından önce ana makine,
kayıtlı eşzamansız bellek istemi hazırlığını tamamlar. Eşzamanlı
`buildMemorySystemPromptAddition(...)` yardımcısı bu değişmez çalıştırma anlık görüntüsünü okur;
sağlanan araç, alıntı, ajan ve oturum bağlamını değiştirmeden aktarın.

Ardından yapılandırmada etkinleştirin:

```json5
{
  plugins: {
    slots: {
      contextEngine: "my-engine",
    },
    entries: {
      "my-engine": {
        enabled: true,
      },
    },
  },
}
```

### ContextEngine arayüzü

Gerekli üyeler:

| Üye               | Tür      | Amaç                                                        |
| ------------------ | -------- | ----------------------------------------------------------- |
| `info`             | Özellik  | Motor kimliği, adı, sürümü ve Compaction'ı yönetip yönetmediği |
| `ingest(params)`   | Yöntem   | Tek bir iletiyi saklama                                    |
| `assemble(params)` | Yöntem   | Bir model çalıştırması için bağlam oluşturma (`AssembleResult` döndürür) |
| `compact(params)`  | Yöntem   | Bağlamı özetleme/azaltma                                   |

`assemble`, aşağıdakileri içeren bir `AssembleResult` döndürür:

<ParamField path="messages" type="Message[]" required>
  Modele gönderilecek sıralı iletiler.
</ParamField>
<ParamField path="estimatedTokens" type="number" required>
  Motorun, birleştirilmiş bağlamdaki toplam belirteç sayısına ilişkin tahmini. OpenClaw bunu Compaction eşiği kararları ve tanılama raporlaması için kullanır.
</ParamField>
<ParamField path="systemPromptAddition" type="string">
  Sistem isteminin başına eklenir.
</ParamField>
<ParamField path="promptAuthority" type='"assembled" | "preassembly_may_overflow"'>
  Çalıştırıcının önleyici taşma ön denetimleri için hangi belirteç tahminini
  kullanacağını denetler. Varsayılan değer `"assembled"` değeridir; bu,
  Compaction'ı yönetmeyen motorlarda yalnızca birleştirilmiş istemin tahmininin
  denetlendiği anlamına gelir. `ownsCompaction: true` değerini ayarlayan motorlar
  kendi istem kabul işlemlerini yönetir; bu nedenle OpenClaw varsayılan olarak
  genel istem öncesi ön denetimi atlar. Yalnızca birleştirilmiş görünümünüz
  temel transkriptteki taşma riskini gizleyebiliyorsa `"preassembly_may_overflow"`
  değerini ayarlayın; bu durumda çalıştırıcı genel ön denetimi etkin tutar ve
  önleyici Compaction yapılıp yapılmayacağına karar verirken birleştirilmiş
  tahmin ile birleştirme öncesi (pencerelenmemiş) oturum geçmişi tahmininden
  büyük olanını kullanır. Her iki durumda da döndürdüğünüz iletiler modelin
  gördüğü iletilerdir; `promptAuthority` yalnızca ön denetimi etkiler.
</ParamField>
<ParamField path="contextProjection" type="ContextEngineProjection">
  Kalıcı arka uç iş parçacıklarına sahip ana makineler (örneğin Codex app-server) için isteğe bağlı yansıtma yaşam döngüsü. Kararlı bir `epoch` ile `mode: "thread_bootstrap"`, ana makineden birleştirilmiş bağlamı dönem başına bir kez eklemesini ve her turda yeniden yansıtmak yerine dönem değişene kadar arka uç iş parçacığını yeniden kullanmasını ister. Normal tur başına yansıtma için bu alanı dahil etmeyin.
</ParamField>

`compact`, bir `CompactResult` döndürür. Compaction etkin oturum
kimliğini değiştirdiğinde, `result.sessionTarget` (oturum kimliğini ve depo
kapsamını taşıyan türü belirlenmiş bir `ContextEngineSessionTarget`), sonraki yeniden
denemenin veya turun kullanması gereken ardıl oturumu tanımlar; `result.sessionId`
ardıl kimliği yansıtır.

İsteğe bağlı üyeler:

| Üye                           | Tür    | Amaç                                                                                                                                         |
| ----------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `bootstrap(params)`            | Yöntem | Bir oturum için motor durumunu başlatır. Motor bir oturumu ilk kez gördüğünde bir kez çağrılır (ör. geçmişi içe aktarma).                    |
| `maintain(params)`             | Yöntem | Önyükleme, başarılı bir tur veya Compaction sonrasında transkript bakımı. Güvenli yeniden yazmalar için `runtimeContext.rewriteTranscriptEntries()` kullanın. |
| `ingestBatch(params)`          | Yöntem | Tamamlanmış bir turu toplu olarak alır. Bir çalıştırma tamamlandıktan sonra o turdaki tüm iletilerle birlikte tek seferde çağrılır.             |
| `afterTurn(params)`            | Yöntem | Çalıştırma sonrası yaşam döngüsü işleri (durumu kalıcı hâle getirme, arka planda Compaction tetikleme).                                       |
| `prepareSubagentSpawn(params)` | Yöntem | Alt oturum başlamadan önce bu oturum için paylaşılan durumu ayarlar.                                                                          |
| `onSubagentEnded(params)`      | Yöntem | Bir alt ajan sona erdikten sonra temizleme yapar.                                                                                             |
| `dispose()`                    | Yöntem | Kaynakları serbest bırakır. Oturum başına değil, Gateway kapatılırken veya Plugin yeniden yüklenirken çağrılır.                              |

### Çalışma zamanı ayarları

OpenClaw içinde çalışan yaşam döngüsü kancaları, isteğe bağlı bir
`runtimeSettings` nesnesi alır. Bu, sürümlendirilmiş ve salt okunur bir
dahili üretici/tüketici API yüzeyidir: OpenClaw bunu seçilen bağlam motoru için
üretir ve bağlam motoru bunu yaşam döngüsü kancaları içinde tüketir. Doğrudan
kullanıcılara gösterilmez ve özel bir raporlama yüzeyi oluşturmaz.

- `schemaVersion`: şu anda `1`
- `runtime`: OpenClaw ana bilgisayarı, çalışma zamanı modu (`normal`, `fallback` veya
  `degraded`) ve isteğe bağlı test donanımı/çalışma zamanı kimlikleri
- `contextEngineSelection`: seçilen bağlam motoru kimliği ve seçim kaynağı
- `executionHost`: kancayı çağıran yüzeyin ana bilgisayar kimliği ve etiketi
- `model`: istenen model, çözümlenen model, sağlayıcı ve isteğe bağlı model ailesi
- `limits`: biliniyorsa istem belirteci bütçesi ve maksimum çıktı belirteci sayısı
- `diagnostics`: biliniyorsa kapalı geri dönüş ve düşürülmüş çalışma nedeni kodları

Bilinmeyebilen alanlar `null` olarak gösterilir; çalışma zamanı modu ve seçim kaynağı gibi
ayırt edici alanlar null değer kabul etmez. Eski motorlar uyumlu kalır:
katı bir eski motor `runtimeSettings` özelliğini bilinmeyen bir özellik olarak reddederse
OpenClaw, motoru karantinaya almak yerine yaşam döngüsü çağrısını bu özellik olmadan
yeniden dener.

### Ana bilgisayar gereksinimleri

Bağlam motorları, `info.hostRequirements` üzerinde ana bilgisayar yeteneği gereksinimleri bildirebilir.
OpenClaw, işleme başlamadan önce bu gereksinimleri denetler ve seçilen çalışma zamanı
bunları karşılayamadığında açıklayıcı bir hatayla kapalı durumda başarısız olur.

Motorun gerçek model istemini `assemble()` aracılığıyla denetlemesi gerektiğinde
ajan çalıştırmaları için `assemble-before-prompt` bildirin:

```ts
info: {
  id: "my-context-engine",
  name: "Bağlam Motorum",
  hostRequirements: {
    "agent-run": {
      requiredCapabilities: ["assemble-before-prompt"],
      unsupportedMessage:
        "Yerel Codex veya OpenClaw gömülü çalışma zamanını kullanın ya da eski bağlam motorunu seçin.",
    },
  },
}
```

Yerel Codex ve OpenClaw gömülü ajan çalıştırmaları `assemble-before-prompt` gereksinimini karşılar.
Genel CLI arka uçları bunu karşılamaz; bu nedenle bu yeteneği gerektiren motorlar
CLI işlemi başlamadan önce reddedilir.

### Hata yalıtımı

OpenClaw, seçilen plugin motorunu çekirdek yanıt yolundan yalıtır. Eski olmayan bir
motor eksikse, sözleşme doğrulamasını geçemezse, fabrika oluşturulurken hata fırlatırsa
veya bir yaşam döngüsü yönteminden hata fırlatırsa OpenClaw, söz konusu motoru mevcut
Gateway işlemi için karantinaya alır ve bağlam motoru işlemlerini yerleşik
`legacy` motoruna düşürür. Hata, başarısız işlemle birlikte günlüğe kaydedilir;
böylece operatör, ajan sessiz kalmadan plugin'i onarabilir, güncelleyebilir veya devre dışı
bırakabilir.

Ana bilgisayar gereksinimi hataları farklıdır: Bir motor, çalışma zamanında gerekli bir
yeteneğin bulunmadığını bildirdiğinde OpenClaw, çalıştırmayı başlatmadan önce kapalı
durumda başarısız olur. Bu, desteklenmeyen bir ana bilgisayarda çalışmaları hâlinde
durumu bozacak motorları korur.

### ownsCompaction

`ownsCompaction`, OpenClaw çalışma zamanının yerleşik deneme içi otomatik Compaction özelliğinin çalıştırma için etkin kalıp kalmayacağını denetler:

<AccordionGroup>
  <Accordion title="ownsCompaction: true">
    Compaction davranışının sahibi motordur. OpenClaw, söz konusu çalıştırma için OpenClaw çalışma zamanının yerleşik otomatik Compaction özelliğini ve istem öncesi genel taşma ön denetimini devre dışı bırakır; motorun `compact()` uygulaması `/compact`, sağlayıcı taşması kurtarma Compaction'ı ve `afterTurn()` içinde gerçekleştirmek istediği tüm proaktif Compaction işlemlerinden sorumludur. Motor, `assemble()` sonucunda `promptAuthority: "preassembly_may_overflow"` döndürdüğünde OpenClaw yine de istem öncesi taşma korumasını çalıştırır.
  </Accordion>
  <Accordion title="ownsCompaction: false veya ayarlanmamış">
    OpenClaw çalışma zamanının yerleşik otomatik Compaction özelliği istem yürütme sırasında çalışmaya devam edebilir; ancak etkin motorun `compact()` yöntemi `/compact` ve taşma kurtarma için yine de çağrılır.
  </Accordion>
</AccordionGroup>

<Warning>
`ownsCompaction: false`, OpenClaw'ın otomatik olarak eski motorun Compaction yoluna geri döneceği anlamına **gelmez**.
</Warning>

Bu, iki geçerli plugin kalıbı olduğu anlamına gelir:

<Tabs>
  <Tab title="Sahiplik modu">
    Kendi Compaction algoritmanızı uygulayın ve `ownsCompaction: true` değerini ayarlayın.
  </Tab>
  <Tab title="Yetkilendirme modu">
    `ownsCompaction: false` değerini ayarlayın ve OpenClaw'ın yerleşik Compaction davranışını kullanmak için `compact()` yönteminin `openclaw/plugin-sdk/core` üzerinden `delegateCompactionToRuntime(...)` çağrısı yapmasını sağlayın.
  </Tab>
</Tabs>

Hiçbir işlem yapmayan bir `compact()`, etkin ve sahip olmayan bir motor için güvenli değildir; çünkü söz konusu motor yuvasının normal `/compact` ve taşma kurtarma Compaction yolunu devre dışı bırakır.

## Yapılandırma referansı

```json5
{
  plugins: {
    slots: {
      // Etkin bağlam motorunu seçin. Varsayılan: "legacy".
      // Bir plugin motoru kullanmak için bir plugin kimliğine ayarlayın.
      contextEngine: "legacy",
    },
  },
}
```

<Note>
Yuva, çalışma zamanında özeldir: Belirli bir çalıştırma veya Compaction işlemi için yalnızca bir kayıtlı bağlam motoru çözümlenir. Etkinleştirilmiş diğer `kind: "context-engine"` plugin'leri yine de yüklenip kayıt kodlarını çalıştırabilir; `plugins.slots.contextEngine`, yalnızca OpenClaw bir bağlam motoruna ihtiyaç duyduğunda hangi kayıtlı motor kimliğini çözümleyeceğini seçer.
</Note>

<Note>
**Plugin'i kaldırma:** `plugins.slots.contextEngine` olarak seçili plugin'i kaldırdığınızda OpenClaw, yuvayı varsayılan değere (`legacy`) sıfırlar. Aynı sıfırlama davranışı `plugins.slots.memory` için de geçerlidir. Yapılandırmanın elle düzenlenmesi gerekmez.
</Note>

## Compaction ve bellekle ilişkisi

<AccordionGroup>
  <Accordion title="Compaction">
    Compaction, bağlam motorunun sorumluluklarından biridir. Eski motor, işlemi OpenClaw'ın yerleşik özetleme özelliğine devreder. Plugin motorları herhangi bir Compaction stratejisi (DAG özetleri, vektör erişimi vb.) uygulayabilir.
  </Accordion>
  <Accordion title="Bellek plugin'leri">
    Bellek plugin'leri (`plugins.slots.memory`) bağlam motorlarından ayrıdır. Bellek plugin'leri arama/erişim sağlar; bağlam motorları ise modelin ne göreceğini denetler. Birlikte çalışabilirler: Bir bağlam motoru, birleştirme sırasında bellek plugini verilerini kullanabilir. Etkin bellek istem yolunu kullanmak isteyen plugin motorları, bellek plugini düzenini açığa çıkarmadan ana bilgisayar tarafından hazırlanmış bellek istemi bölümlerini başa eklenmeye hazır bir `systemPromptAddition` biçimine dönüştüren `openclaw/plugin-sdk/core` içindeki `buildMemorySystemPromptAddition(...)` öğesini kullanmalıdır.
  </Accordion>
  <Accordion title="Oturum budama">
    Bellekteki eski araç sonuçlarını kırpma işlemi, hangi bağlam motorunun etkin olduğundan bağımsız olarak çalışmaya devam eder.
  </Accordion>
</AccordionGroup>

## İpuçları

- Motorunuzun doğru şekilde yüklendiğini doğrulamak için `openclaw doctor` kullanın.
- Motorlar arasında geçiş yapıldığında mevcut oturumlar geçerli geçmişleriyle devam eder. Yeni motor, gelecekteki çalıştırmaları devralır.
- Motor hataları günlüğe kaydedilir ve seçilen plugin motoru mevcut Gateway işlemi için karantinaya alınır. Yanıtların devam edebilmesi için OpenClaw, kullanıcı turlarında `legacy` seçeneğine geri döner; ancak bozuk plugin'i yine de onarmalı, güncellemeli, devre dışı bırakmalı veya kaldırmalısınız.
- Geliştirme için yerel bir plugin dizinini kopyalamadan bağlamak üzere `openclaw plugins install -l ./my-engine` kullanın.

## İlgili

- [Compaction](/tr/concepts/compaction) - uzun konuşmaları özetleme
- [Bağlam](/tr/concepts/context) - ajan turları için bağlamın nasıl oluşturulduğu
- [Plugin Mimarisi](/tr/plugins/architecture) - bağlam motoru plugin'lerini kaydetme
- [Plugin manifesti](/tr/plugins/manifest) - plugin manifesti alanları
- [Plugin'ler](/tr/tools/plugin) - plugin'e genel bakış
