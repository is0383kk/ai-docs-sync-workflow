---
read_when:
    - Bir agent için GitHub Copilot SDK donanımını kullanmak istiyorsunuz
    - '`copilot` çalışma zamanı için yapılandırma örneklerine ihtiyacınız var'
    - Bir agenti abonelik Copilot'a (github / openclaw / copilot) bağlıyorsunuz ve Copilot CLI üzerinden çalışmasını istiyorsunuz
summary: Harici GitHub Copilot SDK çalıştırma ortamı üzerinden OpenClaw yerleşik ajan turlarını çalıştırın
title: Copilot SDK test düzeneği
x-i18n:
    generated_at: "2026-07-27T00:07:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4b67959c2c72bda97a81d0b45bc32ba363373064ec40c54f9709705dd15dd9fc
    source_path: plugins/copilot.md
    workflow: 16
---

Harici `@openclaw/copilot` plugin'i, yerleşik abonelik Copilot
ajan turlarını OpenClaw'ın yerleşik altyapısı yerine GitHub Copilot CLI
(`@github/copilot-sdk`) üzerinden çalıştırır. Copilot CLI oturumu, düşük seviyeli
ajan döngüsünün sahibidir: yerel araç yürütme, yerel Compaction (`infiniteSessions`) ve
`copilotHome` altındaki CLI tarafından yönetilen iş parçacığı durumu. OpenClaw; sohbet
kanallarının, oturum dosyalarının, model seçiminin, dinamik araçların (köprülenmiş), onayların,
medya tesliminin, görünür transkript yansısının, `/btw` yan sorularının (bkz.
[Yan sorular (`/btw`)](#side-questions-btw)) ve `openclaw doctor` öğesinin sahibi olmaya devam eder.

Daha geniş model/sağlayıcı/çalışma zamanı ayrımı için
[Ajan çalışma zamanları](/tr/concepts/agent-runtimes) ile başlayın.

## Gereksinimler

- `@openclaw/copilot` plugin'i yüklü OpenClaw.
- Yapılandırmanız `plugins.allow` kullanıyorsa `copilot` değerini (plugin'in
  bildirdiği manifest kimliği) ekleyin. npm paket adı
  `@openclaw/copilot` için bir izin listesi girdisi eşleşmez ve
  `agentRuntime.id: "copilot"` ayarlanmış olsa bile plugin'in engellenmiş kalmasına neden olur.
- Copilot CLI'ı çalıştırabilen bir GitHub Copilot aboneliği veya
  başsız ya da Cron çalıştırmaları için bir `gitHubToken` ortam değişkeni / kimlik doğrulama profili girdisi.
- Yazılabilir bir `copilotHome` dizini. OpenClaw bir ajan dizini
  sağladığında varsayılan değer `<agentDir>/copilot`, aksi takdirde
  `~/.openclaw/agents/<agentId>/copilot` olur.

`openclaw doctor`, oturum durumu sahipliği ve gelecekteki yapılandırma
geçişleri için plugin'in [doctor sözleşmesini](#doctor) çalıştırır. Copilot CLI
ortamını yoklamaz.

## Kurulum

Copilot çalışma zamanı, çekirdek `openclaw` paketinin
`@github/copilot-sdk` veya platforma özgü `@github/copilot-<platform>-<arch>` CLI ikili dosyasını
(birlikte yaklaşık 260 MB) taşımaması için harici bir plugin olarak sunulur.
Yalnızca bu çalışma zamanını tercih eden ajanlar için kurun:

```bash
openclaw plugins install @openclaw/copilot
```

Kurulum sihirbazı, ilk kez bir `github-copilot/*` modeli seçtiğinizde **ve**
yapılandırmanız bu modeli (veya sağlayıcısını) `agentRuntime: { id: "copilot" }` aracılığıyla
Copilot çalışma zamanına yönlendirdiğinde plugin'i otomatik olarak kurar; bkz.
[Hızlı başlangıç](#quickstart). Bu tercih olmadan OpenClaw, yerleşik GitHub
Copilot sağlayıcısını kullanır ve bu plugin'i hiçbir zaman kurmaz.

Çalışma zamanı SDK'yı şu sırayla çözümler:

1. Kurulu `@openclaw/copilot` paketindeki `import("@github/copilot-sdk")`.
2. Geri dönüş dizini `~/.openclaw/npm-runtime/copilot/` (eski isteğe bağlı
   kurulum hedefi).

Eksik bir SDK, `COPILOT_SDK_MISSING` koduyla tek bir hata ve yukarıdaki yeniden
kurulum komutunu gösterir.

## Hızlı başlangıç

Bir modeli (veya bir sağlayıcıyı) altyapıya sabitleyin:

```json5
{
  agents: {
    defaults: {
      model: "github-copilot/auto",
      models: {
        "github-copilot/auto": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
}
```

Yalnızca o modeli altyapı üzerinden yönlendirmek için tek bir model girdisinde
`agentRuntime.id` ayarlayın veya söz konusu sağlayıcının altındaki tüm modelleri
yönlendirmek için bunu bir sağlayıcıda ayarlayın.

`github-copilot/auto`, taşınabilir başlangıç noktasıdır. Adlandırılmış Copilot
modelleri hesaba ve kuruluş politikasına bağlıdır; bir modeli sabitlemeden önce
kimliği doğrulanmış Copilot CLI'ınızın o modeli gerçekten sunduğunu doğrulayın.

## Desteklenen sağlayıcılar

Altyapı, `extensions/github-copilot` tarafından sahip olunan standart
`github-copilot` sağlayıcısını ve modelde boş olmayan bir
`baseUrl` ile aşağıdaki `api` biçimlerinden biri
bulunduğunda özel `models.providers` girdilerini destekler:

- `anthropic-messages`
- `azure-openai-responses`
- `ollama` (OpenAI uyumlu tamamlamalar)
- `openai-completions`
- `openai-responses`

Yerel sağlayıcı kimlikleri (`openai`, `anthropic`, `google`, `ollama`) kendi
yerel çalışma zamanlarının sahipliğinde kalır. Bunun yerine bir uç noktayı
Copilot BYOK üzerinden yönlendirmek için ayrı bir özel sağlayıcı kimliği kullanın.

Copilot BYOK uç noktaları genel HTTPS URL'leri olmalıdır. Altyapı, Copilot
SDK'ya her deneme için bir geri döngü proxy'si verir ve ardından sağlayıcı
trafiğini OpenClaw'ın korumalı fetch yolu üzerinden iletir; böylece DNS
sabitleme ve SSRF politikası OpenClaw'ın sahipliğinde kalır. Yerel Ollama, LM
Studio veya LAN model sunucuları için yerel OpenClaw çalışma zamanını kullanın.

## BYOK

Copilot BYOK, SDK'nın oturum düzeyindeki özel sağlayıcı sözleşmesini kullanır.
OpenClaw; çözümlenmiş model uç noktasını, API anahtarını, bearer token modunu,
üstbilgileri, model kimliğini ve bağlam/çıktı sınırlarını aktarır; sağlayıcı
taşıma mantığı çekirdekte değil SDK'da kalır.

```json5
{
  agents: {
    defaults: {
      model: "custom-proxy/llama-3.1-8b",
      models: {
        "custom-proxy/llama-3.1-8b": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      "custom-proxy": {
        baseUrl: "https://api.example.com/v1",
        apiKey: "${CUSTOM_PROXY_API_KEY}",
        api: "openai-responses",
        authHeader: true,
        models: [{ id: "llama-3.1-8b", name: "Llama 3.1 8B" }],
      },
    },
  },
}
```

BYOK oturumları, abonelik oturumlarından ve diğer BYOK uç noktalarından veya
kimlik bilgilerinden ayrı anahtarlanır. Anahtarın, üstbilgilerin, modelin veya
uç noktanın döndürülmesi, uyumsuz durumu sürdürmek yerine yeni bir Copilot SDK
oturumu başlatır.

## Kimlik doğrulama

`runCopilotAttempt` sırasında ajan başına uygulanan öncelik sırası:

1. Deneme girdisindeki **açık `useLoggedInUser: true`** — ajanın
   `copilotHome` altındaki Copilot CLI'da oturum açmış kullanıcıyı kullanır.
2. Deneme girdisindeki **açık `gitHubToken`** (`profileId` +
   `profileVersion` gerektirir). Kimlik doğrulama profili çözümlemesini
   atlaması gereken doğrudan CLI çağrıları ve testler içindir.
3. **Sözleşme tarafından çözümlenen `resolvedApiKey` + `authProfileId`** — üretimdeki
   ana yol. Çekirdek, altyapıyı çağırmadan önce ajanın yapılandırılmış
   `github-copilot` kimlik doğrulama profilini (`src/infra/provider-usage.auth.ts:resolveProviderAuths`)
   çözümler; böylece bir `github-copilot:<profile>` kimlik doğrulama profili, ortam
   değişkenleri olmadan başsız, Cron veya çok profilli kurulumlarda uçtan uca çalışır.
4. **Ortam değişkeni geri dönüşü**, şu sırayla denetlenir (ilk boş olmayan
   değer kazanır, boş dizeler yok sayılır; `extensions/github-copilot/auth.ts` içindeki sunulan
   `github-copilot` sağlayıcı önceliğini yansıtır):
   1. `OPENCLAW_GITHUB_TOKEN` — altyapıya özgü geçersiz kılma; sistem genelindeki
      `gh` / Copilot CLI yapılandırmasını bozmadan OpenClaw
      altyapısı için bir token sabitlemenizi sağlar.
   2. `COPILOT_GITHUB_TOKEN` — standart Copilot SDK / CLI ortam değişkeni.
   3. `GH_TOKEN` — standart `gh` CLI ortam değişkeni.
   4. `GITHUB_TOKEN` — genel GitHub token geri dönüşü.

   Sentezlenen havuz profili kimliği `env:<NAME>` değeridir; profil sürümü
   token'ın geri döndürülemez bir sha256 parmak izidir, dolayısıyla ortam
   değerinin döndürülmesi istemci havuzunu temiz biçimde geçersiz kılar.

5. Token sinyali olmadığında **varsayılan `useLoggedInUser`**.

Her ajan kendi `copilotHome` değerini alır; böylece Copilot CLI token'ları,
oturumları ve yapılandırması aynı makinedeki ajanlar arasında hiçbir zaman
sızmaz. Varsayılan:
`<agentDir>/copilot` (SDK durumunu OpenClaw'ın `models.json` /
`auth-profiles.json` diziniyle aynı dizinin dışında tutar) veya ajan dizini
sağlanmadığında `~/.openclaw/agents/<agentId>/copilot`.
Özel bir konum için (örneğin geçiş amacıyla paylaşılan bir bağlama noktası)
deneme girdisinde `copilotHome: <path>` ile geçersiz kılın.

Canlı altyapı testleri, doğrudan token için `OPENCLAW_COPILOT_AGENT_LIVE_TOKEN` kullanır.
Paylaşılan canlı test kurulumu, gerçek kimlik doğrulama profillerini yalıtılmış
test ana dizinine hazırladıktan sonra `COPILOT_GITHUB_TOKEN`, `GH_TOKEN` ve
`GITHUB_TOKEN` değerlerini temizler; böylece özel değişken üzerinden iletilen
bir `gh auth token` değeri, ilgisiz test paketlerine sızmadan hatalı atlamaları
önler.

## Yapılandırma yüzeyi

Altyapı, ajan başına deneme girdisinden (`runCopilotAttempt({...})`) ve
`extensions/copilot/src/` içindeki küçük bir ortam varsayılanları kümesinden
yapılandırmayı okur:

| Alan                     | Amaç                                                                                                                                                                                                                                                                                            |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `copilotHome`            | Ajan başına CLI durum dizini (varsayılanlar yukarıdadır).                                                                                                                                                                                                                                       |
| `model`                  | Dize veya `{ provider, id, api?, baseUrl?, headers?, authHeader? }`. Ajanın normal model seçimini kullanmak için atlayın; altyapı, çözümlenen sağlayıcının desteklendiğini doğrular.                                                                                                               |
| `reasoningEffort`        | `"low" \| "medium" \| "high" \| "xhigh"`. `auto-reply/thinking.ts` içindeki OpenClaw `ThinkLevel` / `ReasoningLevel` çözümlemesinden eşlenir.                                                                                                                                                   |
| `infiniteSessionConfig`  | `harness.compact` tarafından yönlendirilen SDK `infiniteSessions` bloğu için isteğe bağlı geçersiz kılma. Olduğu gibi bırakılması güvenlidir.                                                                                                                               |
| `hooksConfig`            | Araç/MCP, kullanıcı istemi, oturum ve hata geri çağırmaları için isteğe bağlı yerel Copilot SDK `SessionHooks` yapılandırması. OpenClaw'ın taşınabilir yaşam döngüsü kancalarından ayrıdır.                                                                                  |
| `permissionPolicy`       | Yerleşik SDK araç türleri (`shell`, `write`, `read`, `url`, `mcp`, `memory`, `hook`) için SDK'nın `onPermissionRequest` işleyicisini isteğe bağlı olarak geçersiz kılar. Güvenlik ağı olarak varsayılan değer `rejectAllPolicy`; neden gerçekte hiçbir zaman tetiklenmediği için [İzinler ve ask_user](#permissions-and-ask_user) bölümüne bakın. |
| `enableSessionTelemetry` | İsteğe bağlı SDK oturum telemetrisi bayrağı.                                                                                                                                                                                                                                                      |

OpenClaw plugin kancaları, Copilot'a özgü deneme yapılandırması gerektirmez.
Altyapı; `before_prompt_build`, `llm_input`, `llm_output` ve `agent_end` öğelerini standart
altyapı yardımcıları üzerinden çalıştırır. Başarılı SDK Compaction işlemleri
ayrıca `before_compaction` ve `after_compaction` öğelerini çalıştırır.
Köprülenmiş OpenClaw araçları `before_tool_call` öğesini çalıştırır ve
`after_tool_call` öğesini bildirir; `hooksConfig`, taşınabilir eşdeğeri
olmayan yalnızca SDK'ya özgü geri çağırmalar için kalır.

OpenClaw'daki başka hiçbir şeyin bu alanlar hakkında bilgi sahibi olması
gerekmez. Diğer plugin'ler, kanallar ve çekirdek kod yalnızca standart
`AgentHarnessAttemptParams` / `AgentHarnessAttemptResult` biçimini görür.

## Compaction

`harness.compact` çalıştığında Copilot SDK altyapısı:

1. Bekleyen işi sürdürmeden izlenen SDK oturumunu devam ettirir.
2. SDK'nın oturum kapsamlı geçmiş Compaction RPC'sini çağırır.
3. Çalışma alanının altına uyumluluk işaretleyici dosyaları yazmadan SDK
   Compaction sonucunu döndürür.

OpenClaw tarafındaki transkript yansısı (aşağıda), Compaction sonrası iletileri
almaya devam eder; böylece kullanıcıya yönelik sohbet geçmişi tutarlı kalır.

## Transkript yansıtma

`runCopilotAttempt`, her turun yansıtılabilir mesajlarını
`extensions/copilot/src/dual-write-transcripts.ts` aracılığıyla OpenClaw denetim transkriptine
çift yazar. Yansıtma oturum başına kapsamlandırılır
(`copilot:${sessionId}`) ve mesaj başına anahtarlanır
(`${role}:${sha256_16(role,content)}`); böylece yeniden yayımlanan önceki tur girdileri,
yinelenmek yerine diskteki mevcut anahtarlarla çakışır.

İki hata sınırlama katmanı yansıtmayı sarar; böylece bir transkript yazma
hatası hiçbir zaman denemenin başarısız olmasına yol açmaz: dahili bir en iyi çaba
sarmalayıcısı ve deneme düzeyinde derinlemesine savunma sağlayan
`.catch(...)`. Hatalar günlüğe kaydedilir, kullanıcıya
yansıtılmaz.

## Yan sorular (`/btw`)

`/btw` bu harness'te yerel **değildir**. `createCopilotAgentHarness()`,
`harness.runSideQuestion` değerini kasıtlı olarak tanımsız bırakır
(`extensions/copilot/harness.test.ts`, `describe("runSideQuestion")` içinde doğrulanır);
böylece OpenClaw'ın `/btw` dağıtıcısı (`src/agents/btw.ts`),
Codex dışındaki tüm çalışma zamanlarında kullandığı aynı yola geçer:
yapılandırılmış model sağlayıcısı kısa bir yan soru istemiyle doğrudan
çağrılır ve `streamSimple` üzerinden akışla geri döndürülür
(CLI oturumu ve ek havuz yuvası yoktur).

Bu, Copilot CLI oturumlarını ajanın ana tur döngüsüne ayırır ve
`/btw` davranışını Codex dışındaki diğer çalışma zamanlarıyla
aynı tutar.

## Doctor

`extensions/copilot/doctor-contract-api.ts`,
`src/plugins/doctor-contract-registry.ts` tarafından otomatik olarak yüklenir. Şunları sağlar:

- Boş bir `legacyConfigRules` (henüz kullanımdan kaldırılmış alan yoktur).
- İşlem yapmayan bir `normalizeCompatibilityConfig` (gelecekteki alan kullanımdan kaldırma
  işlemlerinin ağaç içinde kararlı bir yeri olması için korunur).
- Bir `sessionRouteStateOwners` girdisi: sağlayıcı `github-copilot`, çalışma zamanı
  `copilot`, CLI oturum anahtarı `copilot`, kimlik doğrulama profili ön eki `github-copilot:`.

## Sınırlamalar

- Harness, `github-copilot` ile sahipsiz özel BYOK sağlayıcı kimliklerini üstlenir.
  Manifest sahibi yerel sağlayıcı kimlikleri, `agentRuntime.id`
  `copilot` olarak zorlansa bile kendi çalışma zamanlarında kalır.
- TUI yüzeyi yoktur; eş yüzeyi bulunmayan çalışma zamanları için PI'ın TUI'ı
  geri dönüş seçeneği olmaya devam eder.
- Bir ajan `copilot` çalışma zamanına geçtiğinde PI oturum durumu taşınmaz.
  Seçim deneme bazındadır; mevcut PI oturumları geçerliliğini korur.
- `ask_user`, sağlayıcıdan bağımsız Gateway soru çalışma zamanını kullanır. Control
  UI, diğer OpenClaw sorularıyla aynı soru kartını gösterir; desteklenen
  kanallar seçim düğmelerini işler ve sıradaki düz metin mesajı,
  SDK isteği dönmeden önce bu Gateway kaydını çözümler.

## İzinler ve ask_user

Köprülenen OpenClaw araçları için izin uygulaması, SDK'nın
`onPermissionRequest` geri çağrısı üzerinden değil, **araç sarmalayıcısının
içinde** gerçekleşir. PI'ın kullandığı aynı
`wrapToolWithBeforeToolCallHook`
(`src/agents/agent-tools.before-tool-call.ts`), `createOpenClawCodingTools`
tarafından tüm kodlama araçlarına uygulanır: döngü algılama, güvenilir
plugin politikaları, araç çağrısı öncesi kancalar ve Gateway üzerinden
iki aşamalı plugin onaylarının (`plugin.approval.request`) tümü, yerel PI
denemeleriyle tamamen aynı kod yolundan geçer.

Copilot araç köprüsünün döndürdüğü her SDK aracı şunlarla işaretlenir:

- `overridesBuiltInTool: true` — aynı adlı yerleşik Copilot CLI aracının
  (edit, read, write, bash, ...) yerini alır; böylece her araç çağrısı yeniden
  OpenClaw'a yönlendirilir.
- `skipPermission: true` — SDK'ya, aracı çağırmadan önce
  `onPermissionRequest({kind: "custom-tool"})` çalıştırmamasını bildirir.
  Sarmalanmış `execute()` zaten daha kapsamlı OpenClaw politika denetimini
  gerçekleştirir; SDK düzeyindeki bir istem OpenClaw'ın uygulamasını ya
  kısa devre eder (tümüne izin ver) ya da her araç çağrısını engeller
  (tümünü reddet) — bunların hiçbiri PI ile eşdeğer değildir.

Ağaç içindeki Codex harness'i aynı ayrımı kullanır: köprülenen OpenClaw araçları
sarmalanır (`extensions/codex/src/app-server/dynamic-tools.ts`) ve
codex-app-server'ın kendi yerel onay türleri
(`item/commandExecution/requestApproval`, `item/fileChange/requestApproval`,
`item/permissions/requestApproval`), `plugin.approval.request` üzerinden yönlendirilir
(`extensions/codex/src/app-server/approval-bridge.ts`). Copilot SDK'daki
eşdeğeri — `onPermissionRequest` öğesine ulaşan `custom-tool` dışındaki herhangi bir tür
için kapalı hata veren `rejectAllPolicy` — aynı güvenlik ağıdır ve
`overridesBuiltInTool: true` her yerleşik aracın yerini aldığı için pratikte
hiçbir zaman tetiklenmez.

Sarmalanmış araç katmanının PI ile eşdeğer politika kararları verebilmesi için
harness, PI'ın eksiksiz deneme-aracı bağlamını
`createOpenClawCodingTools` öğesine iletir: kimlik (`senderIsOwner`, `memberRoleIds`,
`ownerOnlyToolAllowlist`, ...), kanal/yönlendirme (`groupId`,
`currentChannelId`, `replyToMode`, mesaj aracı geçişleri), kimlik doğrulama
(`authProfileStore`), çalıştırma kimliği (`sandboxSessionKey`, `runId`
değerlerinden türetilen `sessionKey` / `runSessionKey`), model bağlamı
(`modelApi`, `modelContextWindowTokens`, `modelCompat`, `modelHasVision`)
ve çalıştırma kancaları (`onToolOutcome`, `onYield`). Bu alanlar olmadan
yalnızca sahibe özel izin listeleri varsayılan olarak sessizce reddeder,
plugin güven politikaları doğru kapsamı çözümleyemez ve
`session_status: "current"` eski bir korumalı alan anahtarına çözümlenir.
Köprü oluşturucu `extensions/copilot/src/tool-bridge.ts` olup PI'ın
`src/agents/embedded-agent-runner/run/attempt.ts:1262` konumundaki yetkili çağrısını yansıtır.
`runAttempt`, paylaşılan
`resolveSandboxContext` bağlantı noktası üzerinden korumalı alan bağlamını çözümler,
SDK'ya etkin bir çalışma dizini geçirir ve `sandbox` ile alt ajan
oluşturma çalışma alanını araç köprüsüne iletir. Köprü ayrıca SDK sınırında
uygulayabildiği sınırlı araç oluşturma denetimlerini iletir:
`includeCoreTools`, çalışma zamanı araç izin listesi ve `toolConstructionPlan`.

Köprü ayrıca PI ile eşdeğerlik için
`openclaw/plugin-sdk/agent-harness-tool-runtime` içindeki paylaşılan harness araç yüzeyi yardımcısını
kullanır. Araç arama etkinleştirildiğinde SDK, tüm OpenClaw araç şemaları
yerine kompakt denetim araçlarını ve gizli bir katalog yürütücüsünü görür.
Kod modu etkinleştirildiğinde yardımcı, diğer ajan harness'leri tarafından
kullanılan aynı kod modu denetim yüzeyini ve katalog yaşam döngüsünü
oluşturur. Yerel model yalın varsayılanları, çalışma zamanıyla uyumlu şema
filtreleme, dizin hazırlama ve katalog temizleme işlemlerinin tümü paylaşılan
yardımcıda kalır; böylece Copilot ile Codex'e komşu harness'ler birbirinden
sapmaz.

### Oturum düzeyinde GitHub token'ı

Copilot SDK sözleşmesi, **istemci düzeyindeki** GitHub token'ını
(`CopilotClientOptions.gitHubToken`, CLI işleminin kendisinin kimliğini doğrular)
**oturum düzeyindeki** token'dan (`SessionConfig.gitHubToken`, o oturum için
içerik dışlamayı, model yönlendirmeyi ve kotayı belirler;
hem `createSession` hem de `resumeSession` üzerinde dikkate alınır) ayırır.
Harness, kimlik doğrulamayı `resolveCopilotAuth` üzerinden bir kez çözümler ve
kimlik doğrulama modu `gitHubToken` olduğunda her iki alanı da ayarlar
(açık bir `auth.gitHubToken` veya yapılandırılmış bir
`github-copilot` kimlik doğrulama profilinden sözleşmeye göre çözümlenen
`resolvedApiKey`). Çözümlenen mod `useLoggedInUser` olduğunda,
SDK'nın oturum açmış kimlikten kimlik türetmeye devam etmesi için oturum
düzeyindeki alan atlanır.

`ask_user`, `SessionConfig.onUserInputRequest` kullanır. Köprü, SDK
seçeneklerini veya seçeneksiz serbest metin istemlerini Gateway soruları
olarak kaydeder; sabit seçenekli istekler için seçim indekslerini ya da
etiketlerini, SDK isteği izin verdiğinde ise serbest biçimli yanıtları kabul
eder. OpenClaw denemesinin iptal edilmesi Gateway kaydını iptal eder ve boş
bir SDK yanıtı döndürür.

## İlgili

- [Ajan çalışma zamanları](/tr/concepts/agent-runtimes)
- [Codex harness'i](/tr/plugins/codex-harness)
- [Ajan harness pluginleri (SDK referansı)](/tr/plugins/sdk-agent-harness)
