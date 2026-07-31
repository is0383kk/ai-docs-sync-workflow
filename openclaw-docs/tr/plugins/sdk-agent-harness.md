---
read_when:
    - Gömülü agent çalışma zamanını veya harness kayıt defterini değiştiriyorsunuz
    - Paketle birlikte gelen veya güvenilir bir pluginden bir ajan çalıştırma altyapısı kaydediyorsunuz
    - Codex Plugin'inin model sağlayıcılarıyla nasıl ilişkili olduğunu anlamanız gerekir
sidebarTitle: Agent Harness
summary: Düşük seviyeli gömülü ajan yürütücüsünün yerini alan pluginler için deneysel SDK yüzeyi
title: Ajan yürütme ortamı pluginleri
x-i18n:
    generated_at: "2026-07-27T00:13:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ff4e41a46ba0074fc6c8bf46da813b58d074f5e6c5c1d236d7ab78e824bdc02
    source_path: plugins/sdk-agent-harness.md
    workflow: 16
---

Bir **agent harness’ı**, hazırlanmış tek bir OpenClaw agent turunun düşük seviyeli yürütücüsüdür. Model sağlayıcısı, kanal veya araç kayıt defteri değildir. Kullanıcıya yönelik zihinsel model için [Agent çalışma zamanları](/tr/concepts/agent-runtimes) bölümüne bakın.

Bu yüzeyi yalnızca paketlenmiş veya güvenilir yerel plugin’ler için kullanın. Parametre türleri kasıtlı olarak mevcut gömülü çalıştırıcıyı yansıttığından sözleşme hâlâ deneyseldir.

## Harness ne zaman kullanılır?

Bir model ailesinin kendi yerel oturum çalışma zamanı varsa ve normal OpenClaw sağlayıcı aktarımı yanlış soyutlamaysa bir agent harness’ı kaydedin:

- iş parçacıklarının ve Compaction’ın sahibi olan yerel bir kodlama agent’ı sunucusu
- yerel planlama/akıl yürütme/araç olaylarını akışla iletmesi gereken yerel bir CLI veya daemon
- OpenClaw oturum transkriptine ek olarak kendi sürdürme kimliğine ihtiyaç duyan bir model çalışma zamanı

Yalnızca yeni bir LLM API’si eklemek için harness kaydetmeyin. Normal HTTP veya WebSocket model API’leri için bir [sağlayıcı plugin’i](/tr/plugins/sdk-provider-plugins) oluşturun.

## Core’un sahipliğinde kalanlar

Bir harness seçilmeden önce OpenClaw şunları zaten çözümlemiştir:

- sağlayıcı ve model
- harness kimlik doğrulama önyüklemesinin sahibi olduğunu bildirmediği sürece çalışma zamanı kimlik doğrulama durumu
- düşünme düzeyi ve bağlam bütçesi
- OpenClaw transkript/oturum dosyası
- çalışma alanı, korumalı alan ve araç politikası
- kanal yanıt geri çağrıları ve akış geri çağrıları
- model geri dönüşü ve canlı model değiştirme politikası

Harness hazırlanmış bir denemeyi çalıştırır; sağlayıcı seçmez, kanal teslimatının yerine geçmez veya sessizce model değiştirmez.

### Harness’ın sahip olduğu kimlik doğrulama önyüklemesi

Varsayılan olarak core, harness’ı çağırmadan önce sağlayıcı kimlik bilgilerini çözümler. Kendi yerel çalışma zamanı üzerinden kimlik doğrulaması yapabilen güvenilir bir harness, statik `AgentHarness` kaydında `authBootstrap: "harness"` ayarını belirleyebilir. Bu durumda core, söz konusu harness’ın üstlendiği her deneme için genel sağlayıcı kimlik bilgisi önyüklemesini ve eksik kimlik bilgisi hatasını atlar.

Core, mevcut olduğunda uyumlu, açıkça seçilmiş veya sıralanmış bir OpenClaw kimlik doğrulama profilini ve bunun kapsamlandırılmış deposunu yine iletir. Harness; model istekleri göndermeden önce bu profili veya yerel kimlik bilgilerini çözümlemeli, gizli bilgileri deneme kapsamında tutmalı ve eyleme dönüştürülebilir kimlik doğrulama hataları göstermelidir. Kimlik doğrulamanın sahipliğini yalnızca bazen üstlenen bir harness’ta bu yeteneği ayarlamayın.

### Doğrulanmış kurulum çalışma zamanı yapıtları

İlk çalıştırma kurulumunda çıkarım sağlayabilen yerel bir harness, yoklamayı tamamlayan uygulamayı tasdik etmelidir. `params.captureRuntimeArtifact` doğru olduğunda, kararlı kimliğe ve içerik parmak izine sahip opak bir `result.runtimeArtifact` döndürün. Farklı bir harness yüklemeden veya ilgisiz plugin’leri taramadan bu bağlamayı yeniden denetleyen eşleşen bir `runtimeArtifact.validate(...)` yeteneği kaydedin.

Doğrulanmış OpenClaw devam işlemleri ayrıca `params.expectedRuntimeArtifact` değerini iletir. Harness bunu edindiği kesin yerel süreçle karşılaştırmalı ve değerler farklıysa yerel bir iş parçacığını başlatmadan veya sürdürmeden önce başarısız olmalıdır. Sıradan agent turlarında her iki alan da atlanır; böylece içerik karma işlemi normal isteklerin yoğun yoluna girmez. Uzak/WebSocket harness’larının katılabilmesi için bir sunucu tasdik sözleşmesi gerekir; yalnızca sürüm dizesi, yapıt kimliği değildir.

Hazırlanmış deneme ayrıca OpenClaw ve yerel harness’lar arasında ortak kalması gereken çalışma zamanı kararları için OpenClaw’un sahip olduğu bir politika paketi olan `params.runtimePlan` değerini içerir:

- sağlayıcıya duyarlı araç şeması politikası için `runtimePlan.tools.normalize(...)` ve `runtimePlan.tools.logDiagnostics(...)`
- transkript temizleme ve araç çağrısı onarma politikası için `runtimePlan.transcript.resolvePolicy(...)`
- paylaşılan `NO_REPLY` ve medya teslimatı engellemesi için `runtimePlan.delivery.isSilentPayload(...)`
- model geri dönüşü sınıflandırması için `runtimePlan.outcome.classifyRunResult(...)`
- çözümlenmiş sağlayıcı/model/harness meta verileri için `runtimePlan.observability`

Harness’lar, OpenClaw davranışıyla eşleşmesi gereken kararlar için planı kullanabilir; ancak bunu ana makinenin sahip olduğu deneme durumu olarak ele alın: planı değiştirmeyin veya bir tur içinde sağlayıcı/model değiştirmek için kullanmayın.

### İstek aktarımı sözleşmesi

`supports(ctx)`, çözümlenmiş model aktarımını `ctx.modelProvider` içinde alır. Gizli bilgi içermeyen, sağlayıcının sahip olduğu iki olgu seçilen rotayı tanımlar:

- `runtimePolicy.compatibleIds`, sağlayıcının bu somut rotayla uyumlu olduğunu bildirdiği çalışma zamanı kimliklerini listeler. Politikanın bulunmaması, sağlayıcının rota düzeyinde uyumluluk bildirmediği anlamına gelir; destek varsayma izni değildir.
- `requestTransportOverrides: "none"`, yazılmış hiçbir sağlayıcı/model isteği geçersiz kılmasının yeniden üretilmesi gerekmediği anlamına gelir. `"present"`, yazılmış üstbilgilerin, kimlik doğrulama aktarımının, proxy’nin, TLS’nin, yerel hizmetin, özel ağ davranışının veya istek parametrelerinin mevcut olduğu anlamına gelir. Bu olgu söz konusu değerleri açığa çıkarmaz.

Harness hazırlanmış aktarımı yeniden üretemediğinde `{ supported: false, reason }` döndürün. Seçimden sonra ham yapılandırmayı okuyarak destek çıkarımı yapmayın. Kimlik doğrulama hazırlığı birden fazla yeniden deneme rotası oluşturduğunda, gönderimden önce tek bir harness bunların tümünü desteklemelidir. Hiçbir plugin tam kümenin sahipliğini üstlenemiyorsa örtük seçim OpenClaw’u kullanır; açıkça veya kalıcı olarak seçilmiş bir plugin ise kapalı biçimde başarısız olur.

## Harness kaydetme

**İçe aktarma:** `openclaw/plugin-sdk/agent-harness`

```typescript
import type { AgentHarness } from "openclaw/plugin-sdk/agent-harness";
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const myHarness: AgentHarness = {
  id: "my-harness",
  label: "My native agent harness",

  supports(ctx) {
    const routeSupportsHarness =
      ctx.modelProvider?.runtimePolicy?.compatibleIds.includes("my-harness") === true;
    const canReproduceRequest = ctx.modelProvider?.requestTransportOverrides !== "present";
    return ctx.provider === "my-provider" && routeSupportsHarness && canReproduceRequest
      ? { supported: true, priority: 100 }
      : { supported: false, reason: "effective route is not harness-compatible" };
  },

  async runAttempt(params) {
    // Yerel iş parçacığınızı başlatın veya sürdürün.
    // params.prompt, params.tools, params.images, params.onPartialReply,
    // params.onAgentEvent ve hazırlanmış denemenin diğer alanlarını kullanın.
    return await runMyNativeTurn(params);
  },
};

export default definePluginEntry({
  id: "my-native-agent",
  name: "My Native Agent",
  description: "Seçilen modelleri yerel bir agent daemon’u üzerinden çalıştırır.",
  register(api) {
    api.registerAgentHarness(myHarness);
  },
});
```

`authBootstrap` bu genel örnekte kasıtlı olarak yer almaz. `authBootstrap: "harness"` değerini yalnızca harness yukarıdaki sözleşmeyi karşıladığında ekleyin.

### Devredilmiş yürütme

Bir harness sahibi, ses aktarımının Codex destekli bir konuşmayı sürdürmesi gibi mevcut, modele kilitli bir oturumu yürütmesi gereken güvenilir plugin’lerin kimliklerini `delegatedExecutionPluginIds` olarak ayarlayabilir. Bu, core’a ait bir izin listesi değil, sahibin statik onayıdır. Kapsamı dar tutun.

Devralanlar yalnızca iş kabulü ve gömülü yürütme yetkisi alır. OpenClaw; depolanan oturum anahtarının, depo yolunun ve oturum kimliğinin tam olarak eşleşmesini; `modelSelectionLocked:
true`; ayrıca `agentHarnessId` ve `agentHarnessRuntimeOverride` değerlerinin eşleşmesini zorunlu kılar. Ardından çalıştırma harness sahibi üzerinden kapsamlandırılır. Oturum oluşturma, yamalama, sıfırlama, silme, arşivleme ve Gateway değişiklikleri yalnızca sahibin yetkisinde kalır.

## Seçim politikası

OpenClaw, sağlayıcı/model çözümlemesinden sonra bir harness seçer:

1. Model kapsamlı çalışma zamanı politikası önceliklidir.
2. Ardından sağlayıcı kapsamlı çalışma zamanı politikası gelir.
3. `auto`, kayıtlı harness’lara çözümlenmiş etkin rotayı destekleyip desteklemediklerini sorar. Sağlayıcı/model ön ekleri tek başına hiçbir zaman harness seçmez.
4. Kayıtlı hiçbir harness eşleşmezse OpenClaw gömülü çalışma zamanını kullanır.

Plugin harness hataları çalıştırma hataları olarak gösterilir. `auto` modunda gömülü geri dönüş yalnızca kayıtlı hiçbir plugin harness’ı çözümlenmiş sağlayıcı/modeli desteklemediğinde uygulanır. Bir plugin harness’ı çalıştırmayı üstlendikten sonra OpenClaw aynı turu başka bir çalışma zamanı üzerinden yeniden oynatmaz; çünkü bu, kimlik doğrulama/çalışma zamanı anlamlarını değiştirebilir veya yan etkileri çoğaltabilir.

Yapılandırılmış çalışma zamanı politikası, istenen çalışma zamanı konusunda belirleyici olmaya devam eder. Kalıcı bir oturum `agentHarnessId`, rota/kimlik doğrulama hazırlığı hâlâ beklemedeyken yerel transkriptinin sahipliğini korur. Bunların hiçbiri uyumsuz bir rotayı uyumlu hâle getirmez: hazırlanmış olgular mevcut olduğunda seçilen veya sabitlenen harness bunları desteklemelidir; aksi takdirde çalıştırma kapalı biçimde başarısız olur. `/status`, politika, kalıcı sahiplik ve rota desteğinden seçilen etkin çalışma zamanını gösterir. Hazırlanmış durum açıktır: eksik `runtimePolicy`, mevcut aktarım alanlarından çıkarılmak yerine bildirilmemiş olarak kalır. Harness’ın sahip olduğu kimlik doğrulama birden fazla fiziksel rotayı çözümlenmemiş bıraktığında, hazırlanmış destek olgusu bu rotaların uyumlu çalışma zamanı kimliklerinin kesişimidir ve herhangi bir adayda istek geçersiz kılmaları varsa bunu bildirir. Bu nedenle bildirilmemiş tek bir aday, yerel uyumluluğu boş hâle getirir; `preparedAuth.source: "harness"` bir kimlik doğrulama sahibidir, rota desteği çıkarımı yapma izni değildir.

Seçilen harness şaşırtıcıysa `agents/harness` hata ayıklama günlük kaydını etkinleştirin ve Gateway’in yapılandırılmış `agent harness selected` kaydını inceleyin: bu kayıt seçilen harness kimliğini, seçim nedenini, çalışma zamanı/geri dönüş politikasını ve `auto` modunda her plugin adayının destek sonucunu içerir.

Paketlenmiş Codex plugin’i, harness kimliği olarak `codex` değerini kaydeder. Core bunu sıradan bir plugin harness kimliği olarak ele alır; Codex’e özgü takma adlar paylaşılan çalışma zamanı seçicisinde değil, plugin’de veya operatör yapılandırmasında yer alır.

## Sağlayıcı ve harness eşleştirmesi

Çoğu harness ayrıca bir sağlayıcı kaydetmelidir. Sağlayıcı; model başvurularını, kimlik doğrulama durumunu, model meta verilerini ve `/model` seçimini OpenClaw’un geri kalanına görünür kılar. Harness daha sonra `supports(...)` içinde bu sağlayıcıyı üstlenir.

Paketlenmiş Codex plugin’i bu düzeni izler:

- tercih edilen kullanıcı model başvuruları: `openai/gpt-5.6-sol`
- uyumluluk başvuruları: eski `codex/gpt-*` başvuruları kabul edilmeye devam eder, ancak yeni yapılandırmalar bunları normal sağlayıcı/model başvuruları olarak kullanmamalıdır
- harness kimliği: `codex`
- kimlik doğrulama: Codex harness’ı yerel Codex oturum açma/oturumunun sahibi olduğundan yapay sağlayıcı kullanılabilirliği
- uygulama sunucusu isteği: OpenClaw, çıplak model kimliğini Codex’e gönderir ve harness’ın yerel uygulama sunucusu protokolüyle iletişim kurmasını sağlar

Codex plugin’i eklemelidir. Çalışma zamanı politikası ayarlanmamışsa veya `auto` ise OpenAI, Codex’i yalnızca sağlayıcının sahip olduğu rota sözleşmesi `codex` değerini uyumlu olarak bildirdiğinde seçebilir: hiçbir yazılmış istek geçersiz kılması bulunmayan, tam olarak resmî HTTPS Platform Responses veya ChatGPT Responses rotası. `openai/*` ön eki tek başına hiçbir zaman Codex’i seçmez. Özel uç noktalar, Completions bağdaştırıcıları ve yazılmış istek davranışı OpenClaw’da kalır. Düz metin kullanan resmî HTTP uç noktaları reddedilir. Eski `codex/gpt-*` başvuruları uyumluluk girdileri olarak kalır. Bkz. [OpenAI örtük agent çalışma zamanı](/tr/providers/openai#implicit-agent-runtime).

Operatör kurulumu, model ön eki örnekleri ve yalnızca Codex’e yönelik yapılandırmalar için [Codex Harness](/tr/plugins/codex-harness) bölümüne bakın.

Codex plugin’i, [Codex Harness](/tr/plugins/codex-harness) bölümünde belgelenen minimum uygulama sunucusu sürümünü zorunlu kılar. Başlatma el sıkışmasını denetler ve eski ya da sürümü belirtilmemiş sunucuları engeller; böylece OpenClaw yalnızca test ettiği protokol yüzeyine karşı çalışır.

### Araç sonucu ara yazılımı

Paketlenmiş plugin’ler ve eşleşen manifest sözleşmelerine sahip, açıkça etkinleştirilmiş kurulu plugin’ler; manifest’leri `contracts.agentToolResultMiddleware` içinde hedeflenen çalışma zamanı kimliklerini bildirdiğinde `api.registerAgentToolResultMiddleware(...)` üzerinden çalışma zamanından bağımsız araç sonucu ara yazılımı ekleyebilir. Bu güvenilir bağlantı noktası, OpenClaw veya Codex araç çıktısını yeniden modele vermeden önce çalışması gereken eşzamansız araç sonucu dönüşümleri içindir.

Eski paketlenmiş Plugin'ler, yalnızca Codex app-server
ara yazılımı için `api.registerCodexAppServerExtensionFactory(...)` kullanmaya devam edebilir,
ancak yeni sonuç dönüşümleri çalışma zamanından bağımsız API'yi kullanmalıdır. Yalnızca
gömülü çalıştırıcıya özgü `api.registerEmbeddedExtensionFactory(...)` kancası
kaldırılmıştır; gömülü araç sonucu dönüşümleri çalışma zamanından bağımsız ara yazılım kullanmalıdır.

### Terminal sonucu sınıflandırması

Kendi protokol projeksiyonunu yöneten yerel çalıştırma ortamları, tamamlanan bir turda
görünür asistan metni üretilmediğinde
`openclaw/plugin-sdk/agent-harness-runtime` içindeki
`classifyAgentHarnessTerminalOutcome(...)` öğesini kullanabilir. Yardımcı,
OpenClaw'ın geri dönüş politikasının farklı bir modelle yeniden deneme yapılıp yapılmayacağına
karar verebilmesi için `empty`, `reasoning-only` veya
`planning-only` döndürür. `planning-only`, çalıştırma ortamının açık
`planText` alanını gerektirir; OpenClaw bunu asistan metninden çıkarmaz. Yardımcı,
istem hatalarını, devam eden turları ve `NO_REPLY` gibi kasıtlı sessiz
yanıtları bilinçli olarak sınıflandırmadan bırakır.

### Aracı sonlandırma yan etkileri

Yerel çalıştırma ortamları, bir denemeyi sonlandırdıktan sonra
`openclaw/plugin-sdk/agent-harness-runtime` içindeki
`runAgentEndSideEffects(...)` öğesini çağırmalıdır. Bu, etkileşimli yanıtları geciktirmeden
taşınabilir `agent_end` kancasını ve OpenClaw'ın araştırma yakalamasını
çalıştırır. Denemenin bu yan etkiler tamamlanana kadar sonuçlanmaması gereken
yerel, etkileşimsiz çalıştırmalar için `awaitAgentEndSideEffects(...)` kullanın.
Her iki yardımcı da `runAgentHarnessAgentEndHook(...)` ile aynı
`{ event, ctx }` yükünü kabul eder; bunların başarısızlıkları tamamlanan
deneme sonucunu değiştirmez.

### Kullanıcı girişi ve araç yüzeyleri

Çalışma zamanı düzeyinde kullanıcı girişi isteği sunan yerel çalıştırma ortamları,
istemi biçimlendirmek, OpenClaw'ın engelleyici yanıt yolu üzerinden iletmek ve
seçim/serbest biçimli yanıtları çalışma zamanının yerel yanıt şekline geri
normalleştirmek için `openclaw/plugin-sdk/agent-harness-runtime` içindeki
kullanıcı girişi yardımcılarını kullanmalıdır. Yardımcı, her çalıştırma ortamı kendi
protokol ayrıştırmasını ve bekleyen istek yaşam döngüsünü korurken kanal/TUI
sunumunun tutarlı kalmasını sağlar.

Pi benzeri kompakt araç yönlendirmesine ihtiyaç duyan yerel çalıştırma ortamları,
`openclaw/plugin-sdk/agent-harness-tool-runtime` içindeki
`createAgentHarnessToolSurfaceRuntime(...)` öğesini kullanmalıdır. Bu öğe;
araç arama/kod modu denetimi seçimini, yerel model için yalın varsayılanları,
çalışma zamanıyla uyumlu şema filtrelemeyi, gizli katalog yürütmesini, dizin
hazırlamayı ve katalog temizliğini yönetir. Çalıştırma ortamları, SDK'larına özgü araç
dönüşümünü ve yerel yürütme geri çağrısını yönetmeye devam eder.

### Yerel Codex çalıştırma ortamı modu

Paketlenmiş `codex` çalıştırma ortamı, gömülü OpenClaw
aracı turları için yerel Codex modudur. Önce paketlenmiş `codex` Plugin'ini
etkinleştirin ve yapılandırmanız kısıtlayıcı bir izin listesi kullanıyorsa
`plugins.allow` içine `codex` ekleyin. Yerel app-server
yapılandırmaları `openai/gpt-*` kullanmalıdır; OpenAI aracı turları Codex çalıştırma ortamını
yalnızca etkin rota Codex uyumluluğunu bildirdiğinde seçer. Eski Codex model
referansları `openclaw doctor --fix` ile onarılmalıdır ve eski `codex/*`
model referansları, yerel çalıştırma ortamı için uyumluluk takma adları olarak kalır.

Bu mod çalıştığında yerel iş parçacığı kimliğini, sürdürme davranışını,
Compaction'ı ve app-server yürütmesini Codex yönetir. OpenClaw; sohbet kanalını,
görünür transkript yansısını, araç politikasını, onayları, medya teslimini ve oturum
seçimini yönetmeye devam eder. Çalıştırmayı yalnızca Codex app-server yolunun
üstlenebileceğini kanıtlamanız gerektiğinde sağlayıcı/model olarak
`agentRuntime.id: "codex"` kullanın. Açık Plugin çalışma zamanları kapalı biçimde başarısız olur;
Codex app-server seçim hataları ve çalışma zamanı hataları başka bir çalışma zamanı
üzerinden yeniden denenmez.

## Çalışma zamanı katılığı

OpenClaw varsayılan olarak `auto` sağlayıcı/model çalışma zamanı politikasını kullanır:
kayıtlı Plugin çalıştırma ortamları uyumlu etkin rotaları üstlenebilir ve hiçbiri eşleşmediğinde
gömülü çalışma zamanı turu işler. Tek başına bir sağlayıcı/model öneki hiçbir zaman bir
çalıştırma ortamı seçmez. Eksik çalıştırma ortamı seçiminin gömülü çalışma zamanı üzerinden
yönlendirilmek yerine başarısız olması gerekiyorsa `agentRuntime.id: "codex"` gibi açık bir
sağlayıcı/model Plugin çalışma zamanı kullanın. Açık seçim, uyumsuz bir rotayı uyumlu hâle
getirmez. Seçilen Plugin çalıştırma ortamı hataları her zaman kesin başarısızlıkla sonuçlanır.
Bu, açık bir sağlayıcı/model
`agentRuntime.id: "openclaw"` kullanımını engellemez.

Yalnızca Codex'e özgü gömülü çalıştırmalar için:

```json
{
  "models": {
    "providers": {
      "openai": {
        "agentRuntime": {
          "id": "codex"
        }
      }
    }
  },
  "agents": {
    "defaults": {
      "model": "openai/gpt-5.6-sol"
    }
  }
}
```

Tek bir kanonik model için CLI arka ucu istiyorsanız çalışma zamanını o
model girdisine yerleştirin:

```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-5",
      "models": {
        "anthropic/claude-opus-5": {
          "agentRuntime": {
            "id": "claude-cli"
          }
        }
      }
    }
  }
}
```

Aracı başına geçersiz kılmalar aynı model kapsamlı şekli kullanır:

```json
{
  "agents": {
    "list": [
      {
        "id": "codex-only",
        "model": "openai/gpt-5.6-sol",
        "models": {
          "openai/gpt-5.6-sol": {
            "agentRuntime": { "id": "codex" }
          }
        }
      }
    ]
  }
}
```

Bunun gibi eski, aracı genelindeki çalışma zamanı örnekleri yok sayılır:

```json
{
  "agents": {
    "defaults": {
      "agentRuntime": {
        "id": "codex"
      }
    }
  }
}
```

Açık bir Plugin çalışma zamanı kullanıldığında, istenen çalıştırma ortamı kayıtlı değilse,
çözümlenen sağlayıcı/modeli desteklemiyorsa veya tur yan etkilerini üretmeden önce
başarısız olursa oturum erkenden başarısız olur. Bu davranış, yalnızca Codex'e özgü
dağıtımlar ve Codex app-server yolunun gerçekten kullanımda olduğunu kanıtlaması gereken
canlı testler için kasıtlıdır.

Bu ayar yalnızca gömülü aracı çalıştırma ortamını denetler. Görüntü, video, müzik, TTS,
PDF veya sağlayıcıya özgü diğer model yönlendirmelerini devre dışı bırakmaz.

## Yerel oturumlar ve transkript yansısı

Bir çalıştırma ortamı yerel oturum kimliği, iş parçacığı kimliği veya daemon tarafı sürdürme
belirteci tutabilir. Bu bağlamayı OpenClaw oturumuyla açıkça ilişkilendirin ve
kullanıcı tarafından görülebilen asistan/araç çıktısını OpenClaw
transkriptine yansıtmaya devam edin.

OpenClaw transkripti şunlar için uyumluluk katmanı olmaya devam eder:

- kanalda görünen oturum geçmişi
- transkript araması ve dizinleme
- daha sonraki bir turda yerleşik OpenClaw çalıştırma ortamına geri dönme
- genel `/new`, `/reset` ve oturum silme davranışı

Çalıştırma ortamınız bir yardımcı bağlama depoluyorsa, sahip OpenClaw oturumu
sıfırlandığında OpenClaw'ın bunu temizleyebilmesi için `reset(...)` uygulayın.

## Araç ve medya sonuçları

Çekirdek, OpenClaw araç listesini oluşturur ve hazırlanmış denemeye aktarır.
Bir çalıştırma ortamı dinamik bir araç çağrısını yürüttüğünde, kanal medyasını kendiniz
göndermek yerine araç sonucunu çalıştırma ortamı sonuç şekli üzerinden geri döndürün.

Bu, metin, görüntü, video, müzik, TTS, onay ve mesajlaşma aracı
çıktılarını OpenClaw destekli çalıştırmalarla aynı teslim yolunda tutar.

`AgentHarnessAttemptResult.hostOwnedToolMediaUrls` öğesini yalnızca güvenilen çalıştırma ortamının
kendisinin oluşturduğu ve kalıcı hâle getirdiği yerel yapıtlar için ayarlayın. Her girdi
`toolMediaUrls` içinde de görünmelidir. Model tarafından seçilen dinamik araç veya
OpenClaw aracı medyasını hiçbir zaman dahil etmeyin. `message_tool_only` rotalarında
bu dar kaynak bilgisi, yerel çalışma zamanı yapıtlarının kaynak yanıtı engellemesini aşarak
korunmasını sağlar; normal gönderme politikası ve ortam odasına kabul kuralları uygulanmaya
devam eder.

### Terminal araç sonuçları

`AgentHarnessAttemptParams.observeToolTerminal`, ana makine tarafından yönetilen terminal
sonucu biriktiricisidir. OpenClaw dinamik araçlarını veya yerel araçları yürüten bir
çalıştırma ortamı, deneme sonucu sonlandırılmadan önce her araç bir terminal sonucuna
ulaştığında bunu çağırmalıdır. Araç yürütmeyen çalıştırma ortamlarının bunu çağırması gerekmez.

Yürütme sınırındaki olguları bildirin:

- Mevcutsa protokol çağrı kimliğini, kanonik araç adını ve hazırlık veya kanca
  yeniden yazımlarından sonra araca gerçekten ulaşan bağımsız değişkenleri aktarın.
- Doğrulama, onay veya başka bir koruma, araç uygulaması başlamadan önce çağrıyı
  durdurduğunda `executionStarted: false` ayarlayın. Gönderim gerçekleşmiş olabilecek duruma
  geldiğinde ihtiyatlı biçimde `true` bildirin.
- `outcome: "success"` veya `outcome: "failure"` bildirin. Görüntüleme metninden
  başarısızlık çıkarmak yerine çalışma zamanındaki yapılandırılmış başarısızlık alanlarını
  dahil edin.
- `nativeMutation` öğesini yalnızca OpenClaw araç tanımı kullanmayan yerel araçlar
  için kullanın. Protokol tarafından yönetilen değişiklik ve yeniden oynatma olgularını burada
  sağlayın; OpenClaw'ın değişiklik sınıflandırıcısını çalıştırma ortamına kopyalamayın.

Geri çağrı, bu çağrı için kanonik çözümlemeyi döndürür. Bunun
`lastToolError` değerini `AgentHarnessAttemptResult` içine taşıyın ve paralel
durum türetmek yerine çalıştırma ortamı projeksiyonunda bunun yürütme,
bağımsız değişken ve yan etki olgularını kullanın. Ana makine, çözümlenmemiş bir
değişiklik yapan başarısızlığı ilgisiz başarılı araçlar boyunca korur ve bunu yalnızca
eşleşen eylem başarılı olduktan sonra temizler.

Geri çağrı, eski deneysel çalıştırma ortamlarıyla kaynak uyumluluğu için isteğe bağlı kalır.
İsteğe bağlı olması, araç yürüten bir çalıştırma ortamı için göz ardı edilebilir olduğu
anlamına gelmez: terminal raporları olmadan OpenClaw, sessiz Heartbeat tamamlanması dahil
olmak üzere daha sonraki araç çağrıları boyunca değişiklik yapan araç başarısızlığı
gerçeğini koruyamaz.

### Karara bağlanmış araç sonlandırması

Bir çalıştırma ortamı tüm araç çağrılarını tamamladıktan sonra yerel turu asistan metni
olmadan sona erdiyse OpenClaw'ın son bir görünür yanıta ihtiyacı olabilir. Bir çalıştırma
ortamı, `finalizeSettledTurn({ attempt,
settledAttempt })` uygulayarak bu kurtarma işlemine katılabilir.

Geri çağrı, başka bir sıradan deneme değil, ayrı bir yetenektir. Şunları yapmalıdır:

- tam olarak kısıtlanmış yerel transkripti veya karara bağlanmış araç sonucu
  sınırına kadar dondurulmuş eksiksiz bir uygulama transkriptini kullanmak;
- hiçbir araç, izin verme veya kullanıcı girişi yeteneği, yerel yürütme
  kancası, aracı, Skills, bellek, zamanlama, uzantı ya da uzaktan denetim sunmamak;
- yalnızca ana makine tarafından sağlanan sonlandırma istemini göndermek; ve
- seçilen transkript/yalıtım stratejisi bu kısıtlamaları uygulayamıyorsa
  kapalı biçimde başarısız olmak.

OpenClaw, geri çağrıyı sıradan deneme ve yeniden deneme döngüsünün dışında terminal bir
alt işlem olarak bir kez çağırır. Başarısızlık, çalıştırmayı yan etki farkındalığına sahip
eksik tur uyarısıyla sonlandırır; sıradan kimlik doğrulama/profil döndürme, model geri
dönüşü, bağlam kurtarma, Compaction devamı veya kanca tarafından istenen revizyon
yollarına giremez. Sonlandırma ayrıca Plugin istem değişikliğini,
`before_agent_run`, LLM giriş/çıkışını, terminal revizyonunu ve
`agent_end` kancalarını atlar. Çekirdek tanılamaları işlemi ve başarısızlığını
kaydetmeye devam eder.

Geri çağrı sıradan bir deneme sonucu değil, `AgentHarnessSettledTurnFinalizationResult`
döndürür. Genel alanları tamamlanmış asistan mesajı, sonlandırma çağrısı kullanımı,
transkript sahipliği meta verileri ve tanılama iziyle sınırlıdır. Araç, teslim, medya,
oluşturma, yaşam döngüsü, yeniden oynatma, oturum ve geri dönüş durumu bu sonuç sınırını
geçemez. Bilinmeyen alanlar ve asistan araç çağrıları kapalı biçimde başarısız olur.

Dahili olarak tam deneme motorunu yeniden kullanan bir çalıştırma ortamı, dönmeden önce
`projectSettledTurnFinalizationAttemptResult(...)` çağırabilir. Yardımcı,
kanonik başarısızlık, araç, teslim, yeniden oynatma ve yaşam döngüsü kanıtlarını reddeder,
ardından yalnızca dar sonucu projekte eder. Bu, yerel yalıtımdan sonra derinlemesine
savunmadır; yerel yetenek yüzeyini kaldırmanın yerine geçmez.

Projeksiyon destekli bir çalıştırma ortamı, eksiksiz bağlamı
`source: "openclaw-transcript"` ile
`settledAttempt.settledTurnFinalizationContext` üzerine yerleştirmelidir. Karara bağlanmış tur yansıtıldıktan
sonra etkin dalı yakalamalı, mevcut istemin ve tüm mevcut araç çağrılarının/sonuçlarının bu
sınıra kadar mevcut olduğunu kanıtlamalı ve denemeyi döndürmeden önce elde edilen mesaj
dizisini dondurmalıdır. Sonlandırıcı eksik, desteklenmeyen, belirsiz veya aşırı büyük bir
bağlamı reddetmelidir. Mesajları kırpmamalı, önceki geçmişi bırakmamalı veya bu uygulama
transkriptini tam yerel geçmiş olarak tanımlamamalıdır. Kısıtlanmış tek bir yerel oturumu
sürdüren çalıştırma ortamlarının bu projeksiyon alanına ihtiyacı yoktur.

Bu geri çağrıyı, en iyi çabaya dayalı bir `disableTools` ipucuyla
`runAttempt` çağırarak uygulamayın. Çalıştırma ortamı sahibi eksiksiz yerel
yetenek sınırını uygulamalıdır. OpenClaw genel bir geri dönüş sağlamaz çünkü rastgele bir
yerel çalışma zamanının bu kısıtlamalara uyduğunu doğrulayamaz.

Geri çağırım, deneysel üçüncü taraf harness uyumluluğu için isteğe bağlı kalır. Seçilen harness bunu sağlamadığında OpenClaw, yan etkilerin tekrarlanması riskini almak yerine mevcut tamamlanmamış tur hatasını korur.

## Mevcut sınırlamalar

- Genel içe aktarma yolu geneldir, ancak bazı deneme/sonuç türü takma adları
  uyumluluk için hâlâ eski adları taşır.
- Üçüncü taraf harness kurulumu deneyseldir. Yerel bir oturum çalışma zamanına
  ihtiyaç duyana kadar sağlayıcı pluginlerini tercih edin.
- Turlar arasında harness değiştirme desteklenir. Yerel araçlar, onaylar,
  asistan metni veya ileti gönderimleri başladıktan sonra turun ortasında
  harness değiştirmeyin.

## İlgili

- [SDK'ya Genel Bakış](/tr/plugins/sdk-overview)
- [Çalışma Zamanı Yardımcıları](/tr/plugins/sdk-runtime)
- [Sağlayıcı Pluginleri](/tr/plugins/sdk-provider-plugins)
- [Codex Harness](/tr/plugins/codex-harness)
- [Model Sağlayıcıları](/tr/concepts/model-providers)
