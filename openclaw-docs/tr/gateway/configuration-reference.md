---
read_when:
    - Tam alan düzeyinde yapılandırma semantiğine veya varsayılan değerlere ihtiyacınız var
    - Kanal, model, Gateway veya araç yapılandırma bloklarını doğruluyorsunuz
summary: Temel OpenClaw anahtarları, varsayılan değerleri ve özel alt sistem referanslarına bağlantılar için Gateway yapılandırma referansı
title: Yapılandırma referansı
x-i18n:
    generated_at: "2026-07-26T23:18:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7135554fda444fd1b8c072af5768c53a165f7be2dcd12a7909fc7fd4bd864428
    source_path: gateway/configuration-reference.md
    workflow: 16
---

`~/.openclaw/openclaw.json` için alan düzeyinde başvuru: anahtarlar, varsayılanlar ve daha ayrıntılı alt sistem sayfalarına bağlantılar. Göreve yönelik kurulum rehberliği için [Yapılandırma](/tr/gateway/configuration) bölümüne bakın. Kanal ve plugin sahipliğindeki komut katalogları ile ayrıntılı bellek/QMD ayarları burada değil, kendi sayfalarında bulunur.

Yapılandırma biçimi **JSON5**'tir (yorumlara ve sondaki virgüllere izin verilir). Tüm alanlar isteğe bağlıdır; atlandıklarında OpenClaw güvenli varsayılanları kullanır.

Kod gerçeği bu sayfadan üstündür:

- `openclaw config schema`, doğrulama ve Control UI için kullanılan canlı JSON Şemasını, paketlenmiş/plugin/kanal meta verileri birleştirilmiş olarak yazdırır.
- Aracılar, yapılandırmayı düzenlemeden önce yol kapsamındaki tek bir kesin şema düğümü için `gateway` araç eylemi `config.schema.lookup` çağrısını yapmalıdır.
- `pnpm config:docs:check` / `pnpm config:docs:gen`, bu belgenin temel karma değerini geçerli şema yüzeyine göre doğrular.

Şema `uiHints`, her yol için çözümlenmiş bir `advanced` boole değeri de taşır.
Control UI bunu, sık kullanılan alanları önce göstermek ve gelişmiş alanları her
bölümde daraltmak için kullanır; arama yine de her iki katmanı kapsar. Katman meta verileri yalnızca sunum amaçlıdır.
Bir anahtar eklerken katmanını yaprakta bildirin veya en yakın
üst öğe bildiriminden devralmasına izin verin. Bildirilmiş bir üst öğesi olmayan yollar varsayılan olarak gelişmiş kabul edilir.

Özel ayrıntılı başvurular:

- `memory.search.*`, `memory.qmd.*`, `memory.citations` ve `plugins.entries.memory-core.config.dreaming` altındaki dreaming yapılandırması için [Bellek yapılandırması başvurusu](/tr/reference/memory-config).
- Geçerli yerleşik + paketlenmiş komut kataloğu için [Eğik çizgi komutları](/tr/tools/slash-commands).
- Kanala özgü komut yüzeyleri için bunların sahibi olan kanal/plugin sayfaları.

---

## Kanallar

Kanal başına yapılandırma anahtarları [Yapılandırma - kanallar](/tr/gateway/config-channels) bölümünde bulunur: Slack, Discord, Telegram, WhatsApp, Matrix, iMessage ve diğer paketlenmiş kanallar için `channels.*` (kimlik doğrulama, erişim denetimi, çoklu hesap, bahsetme geçidi).

## Aracı varsayılanları, çoklu aracı, oturumlar ve mesajlar

Şunlar için [Yapılandırma - aracılar](/tr/gateway/config-agents) bölümüne bakın:

- `agents.defaults.*` (çalışma alanı, model, düşünme, Heartbeat, bellek, medya, Skills, korumalı alan)
- `multiAgent.*` (çoklu aracı yönlendirmesi ve bağlamaları)
- `session.*` (oturum yaşam döngüsü, Compaction, budama)
- `messages.*` (mesaj teslimi, TTS, markdown oluşturma)
- `talk.*` (Konuşma modu)
  - `talk.consultThinkingLevel`: Control UI gerçek zamanlı Konuşma danışmalarının arkasındaki OpenClaw aracı çalışmasının tamamı için düşünme düzeyi geçersiz kılması
  - `talk.consultFastMode`: Control UI gerçek zamanlı Konuşma danışmaları için tek seferlik hızlı mod geçersiz kılması
  - `talk.speechLocale`: Android, iOS ve macOS'ta Konuşma ses tanıması için isteğe bağlı BCP 47 yerel ayar kimliği
  - `talk.silenceTimeoutMs`: ayarlanmadığında Konuşma, dökümü göndermeden önce platformun varsayılan duraklama penceresini korur (`700 ms on macOS and Android, 900 ms on iOS`)
  - `talk.realtime.consultRouting`: `openclaw_agent_consult` adımını atlayan tamamlanmış gerçek zamanlı Konuşma dökümleri için Gateway aktarma yedeği

## Araçlar ve özel sağlayıcılar

Araç politikası, deneysel geçişler, sağlayıcı destekli araç yapılandırması ve özel
sağlayıcı / temel URL kurulumu
[Yapılandırma - araçlar ve özel sağlayıcılar](/tr/gateway/config-tools) bölümünde bulunur.

## Modeller

Sağlayıcı tanımları, model izin listeleri ve özel sağlayıcı kurulumu
[Yapılandırma - araçlar ve özel sağlayıcılar](/tr/gateway/config-tools#custom-providers-and-base-urls) bölümünde bulunur.
`models` kökü, genel model kataloğu davranışının da sahibidir.

```json5
{
  models: {
    // İsteğe bağlı. Varsayılan: true. Değiştirildiğinde Gateway'in yeniden başlatılmasını gerektirir.
    pricing: { enabled: false },
  },
}
```

- `models.mode`: sağlayıcı kataloğu davranışı (`merge` veya `replace`).
- `models.providers`: sağlayıcı kimliğiyle anahtarlanmış özel sağlayıcı eşlemesi.
- `models.providers.*.localService`: yerel model sunucuları için isteğe bağlı, talep üzerine çalışan süreç yöneticisi. OpenClaw, yapılandırılmış sağlık uç noktasını yoklar, gerektiğinde mutlak
  `command` yolunu başlatır, hazır olmasını bekler ve ardından model isteğini
  gönderir. [Yerel model hizmetleri](/tr/gateway/local-model-services) bölümüne bakın.
- `models.pricing.enabled`: yan süreçler ve kanallar Gateway'in hazır yoluna
  ulaştıktan sonra başlayan arka plan fiyatlandırma önyüklemesini denetler. `false` olduğunda
  Gateway, OpenRouter ve LiteLLM fiyatlandırma kataloğu getirmelerini atlar; yapılandırılmış
  `models.providers.*.models[].cost` değerleri yerel maliyet tahminleri için çalışmaya devam eder.

## MCP

OpenClaw tarafından yönetilen MCP sunucu tanımları `mcp.servers` altında bulunur ve
gömülü OpenClaw ile diğer çalışma zamanı bağdaştırıcıları tarafından kullanılır. `openclaw mcp list`,
`show`, `set` ve `unset` komutları, yapılandırma düzenlemeleri sırasında hedef sunucuya bağlanmadan bu bloğu yönetir.

```json5
{
  mcp: {
    servers: {
      docs: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-fetch"],
      },
      remote: {
        url: "https://example.com/mcp",
        transport: "streamable-http", // streamable-http | sse
        requestTimeoutMs: 20000,
        connectionTimeoutMs: 5000,
        supportsParallelToolCalls: true,
        headers: {
          Authorization: "Bearer ${MCP_REMOTE_TOKEN}",
        },
        auth: "oauth",
        oauth: {
          scope: "docs.read",
        },
        sslVerify: true,
        clientCert: "/path/to/client.crt",
        clientKey: "/path/to/client.key",
        toolFilter: {
          include: ["search_*"],
          exclude: ["admin_*"],
        },
        // İsteğe bağlı Codex uygulama sunucusu yansıtma denetimleri.
        codex: {
          agents: ["main"],
          defaultToolsApprovalMode: "approve", // auto | prompt | approve
        },
      },
    },
  },
}
```

- `mcp.servers`: yapılandırılmış MCP araçlarını sunan çalışma zamanları için
  adlandırılmış stdio veya uzak MCP sunucu tanımları.
  Uzak girdiler `transport: "streamable-http"` veya `transport: "sse"` kullanır;
  `type: "http"`, `openclaw mcp set` ve
  `openclaw doctor --fix` tarafından standart `transport` alanına normalleştirilen CLI'ye özgü bir diğer addır.
- `mcp.servers.<name>.enabled`: kayıtlı bir sunucu tanımını korurken
  onu gömülü OpenClaw MCP keşfi ve araç yansıtmasının dışında bırakmak için `false` olarak ayarlayın.
- `mcp.servers.<name>.requestTimeoutMs`: milisaniye cinsinden sunucu başına MCP istek zaman aşımı.
- `mcp.servers.<name>.connectionTimeoutMs`: milisaniye cinsinden sunucu başına bağlantı zaman aşımı.
- `mcp.servers.<name>.supportsParallelToolCalls`: paralel MCP araç çağrıları yapıp yapmamayı
  seçebilen bağdaştırıcılar için isteğe bağlı eşzamanlılık ipucu.
- `mcp.servers.<name>.auth`: OAuth gerektiren HTTP MCP sunucuları için
  `"oauth"` olarak ayarlayın. Belirteçleri OpenClaw durumu altında saklamak için `openclaw mcp login <name>` komutunu çalıştırın.
- `mcp.servers.<name>.oauth`: isteğe bağlı OAuth kapsamı, yönlendirme URL'si ve istemci
  meta verisi URL'si geçersiz kılmaları.
- `mcp.servers.<name>.sslVerify`, `clientCert`, `clientKey`: özel uç noktalar ve karşılıklı TLS için HTTP TLS denetimleri.
- `mcp.servers.<name>.toolFilter`: isteğe bağlı sunucu başına araç seçimi. `include`,
  keşfedilen MCP araçlarını eşleşen adlarla sınırlar; `exclude` eşleşen
  adları gizler. Girdiler, tam MCP araç adları veya basit `*` glob kalıplarıdır. Kaynaklara veya istemlere sahip sunucular ayrıca yardımcı araç adları (`resources_list`,
  `resources_read`, `prompts_list`, `prompts_get`) üretir ve bu adlar aynı
  filtreyi kullanır.
- `mcp.servers.<name>.codex`: isteğe bağlı Codex uygulama sunucusu yansıtma denetimleri.
  Bu blok yalnızca Codex uygulama sunucusu iş parçacıkları için OpenClaw meta verisidir; ACP oturumlarını,
  genel Codex çalıştırma düzeneği yapılandırmasını veya diğer çalışma zamanı bağdaştırıcılarını etkilemez.
  Boş olmayan `codex.agents`, sunucuyu listelenen OpenClaw aracı kimlikleriyle sınırlar.
  Boş, yalnızca boşluk içeren veya geçersiz kapsamlı aracı listeleri, genel hâle gelmek yerine yapılandırma doğrulaması tarafından reddedilir
  ve çalışma zamanı yansıtma yolundan çıkarılır.
  `codex.defaultToolsApprovalMode`, söz konusu sunucu için Codex'in yerel
  `default_tools_approval_mode` değerini üretir. OpenClaw, yerel `mcp_servers` yapılandırmasını Codex'e iletmeden önce `codex`
  bloğunu kaldırır. Sunucunun, Codex'in
  varsayılan MCP onay davranışıyla her Codex uygulama sunucusu aracısına yansıtılmasını korumak için bloğu atlayın.
- Oturum kapsamındaki paketlenmiş MCP çalışma zamanları, yerleşik 10 dakikalık boşta kalma TTL'si kullanır.
  Tek seferlik gömülü çalışmalar, çalışma sonu temizliği ister; TTL, uzun ömürlü oturumlar ve gelecekteki çağıranlar için son güvencedir.
- `mcp.*` altındaki değişiklikler, önbelleğe alınmış oturum MCP çalışma zamanlarını elden çıkararak çalışırken uygulanır.
  Sonraki araç keşfi/kullanımı bunları yeni yapılandırmadan yeniden oluşturur; böylece kaldırılan
  `mcp.servers` girdileri boşta kalma TTL'sini beklemek yerine hemen temizlenir.
- Çalışma zamanı keşfi, söz konusu oturumun önbelleğe alınmış kataloğunu bırakarak MCP araç listesi değişiklik bildirimlerini de dikkate alır.
  Kaynak veya istem bildiren sunucular; kaynakları listelemek/okumak ve
  istemleri listelemek/getirmek için yardımcı araçlar edinir. Tekrarlanan araç çağrısı hataları, başka bir çağrı denenmeden önce
  etkilenen sunucuyu kısa süreliğine duraklatır.

Çalışma zamanı davranışı için [MCP](/tr/cli/mcp#openclaw-as-an-mcp-client-registry) ve
[CLI arka uçları](/tr/gateway/cli-backends#bundle-mcp-overlays) bölümlerine bakın.

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
      allowUploadedArchives: false,
    },
    workshop: {
      allowSymlinkTargetWrites: false,
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // veya düz metin dizesi
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: yalnızca paketlenmiş Skills için isteğe bağlı izin listesi (yönetilen/çalışma alanı Skills etkilenmez).
- `load.extraDirs`: ek paylaşılan Skill kökleri (en düşük öncelik).
- `load.allowSymlinkTargets`: bağlantı, yapılandırılmış kaynak kökünün dışında bulunduğunda Skill sembolik bağlantılarının
  çözümleyebileceği güvenilir gerçek hedef kökleri.
- `workshop.allowSymlinkTargetWrites`: Skill Workshop uygulamasının önceden güvenilen sembolik bağlantı hedefleri üzerinden
  yazmasına izin verir (varsayılan: false).
- `install.preferBrew`: true olduğunda, diğer yükleyici türlerine geri dönmeden önce `brew`
  kullanılabiliyorsa Homebrew yükleyicilerini tercih eder.
- `install.nodeManager`: `metadata.openclaw.install` belirtimleri için Node yükleyici tercihi
  (`npm` | `pnpm` | `yarn` | `bun`).
- `install.allowUploadedArchives`: güvenilir `operator.admin` Gateway
  istemcilerinin `skills.upload.*` üzerinden hazırlanan özel zip arşivlerini yüklemesine izin verir
  (varsayılan: false). Bu yalnızca yüklenen arşiv yolunu etkinleştirir; normal ClawHub
  yüklemeleri bunu gerektirmez.
- `entries.<skillKey>.enabled: false`, paketlenmiş/yüklenmiş olsa bile bir Skill'i devre dışı bırakır.
- `entries.<skillKey>.apiKey`: birincil ortam değişkeni bildiren Skills için kolaylık (düz metin dizesi veya SecretRef nesnesi).
- `limits.maxCandidatesPerRoot`, `limits.maxSkillsLoadedPerSource`, `limits.maxSkillsInPrompt`, `limits.maxSkillsPromptChars`, `limits.maxSkillFileBytes`: Skill keşfini ve modele sunulan Skills istemini sınırlar.
- Skill Workshop özerklik/onay ayarları (`workshop.autonomous.enabled`, `workshop.approvalPolicy`, `workshop.maxPending`, `workshop.maxSkillBytes`) [Skills yapılandırması](/tr/tools/skills-config) bölümünde belgelenmiştir.

---

## Pluginler

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-plugin"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- `~/.openclaw/extensions` ve `<workspace>/.openclaw/extensions` altındaki paket veya paket grubu dizinlerinden ve ayrıca `plugins.load.paths` içinde listelenen dosya veya dizinlerden yüklenir.
- Bağımsız plugin dosyalarını `plugins.load.paths` içine yerleştirin; otomatik keşfedilen uzantı kökleri, bu köklerdeki yardımcı betiklerin başlatmayı engellememesi için üst düzey `.js`, `.mjs` ve `.ts` dosyalarını yok sayar.
- Keşif; yerel OpenClaw pluginlerinin yanı sıra, manifest içermeyen varsayılan Claude yerleşimli paket grupları dahil olmak üzere uyumlu Codex ve Claude paket gruplarını kabul eder.
- **Yapılandırma değişiklikleri Gateway'in yeniden başlatılmasını gerektirir.**
- `allow`: isteğe bağlı izin listesi (yalnızca listelenen pluginler yüklenir). `deny` önceliklidir.
- `plugins.entries.<id>.apiKey`: plugin düzeyinde API anahtarı kolaylık alanı (plugin tarafından desteklendiğinde).
- `plugins.entries.<id>.env`: plugin kapsamlı ortam değişkeni eşlemesi.
- `plugins.entries.<id>.hooks.allowPromptInjection`: `false` olduğunda çekirdek, `before_prompt_build` gibi istemi değiştiren kancaları engeller. Yerel plugin kancalarına ve desteklenen paket gruplarının sağladığı kanca dizinlerine uygulanır.
- `plugins.entries.<id>.hooks.allowConversationAccess`: `true` olduğunda güvenilen ve paket grubuna dahil olmayan pluginler, `llm_input`, `llm_output`, `before_model_resolve`, `before_agent_reply`, `before_agent_run`, `before_agent_finalize` ve `agent_end` gibi türü belirlenmiş kancalardan ham konuşma içeriğini okuyabilir.
- `plugins.entries.<id>.subagent.allowModelOverride`: arka plan alt ajan çalıştırmaları için çalışma başına `provider` ve `model` geçersiz kılmaları istemesi amacıyla bu plugine açıkça güvenin.
- `plugins.entries.<id>.subagent.allowedModels`: güvenilen alt ajan geçersiz kılmaları için kurallı `provider/model` hedeflerinden oluşan isteğe bağlı izin listesi. Yalnızca herhangi bir modele bilerek izin vermek istediğinizde `"*"` kullanın.
- `plugins.entries.<id>.llm.allowModelOverride`: `api.runtime.llm.complete` için model geçersiz kılmaları istemesi amacıyla bu plugine açıkça güvenin.
- `plugins.entries.<id>.llm.allowedModels`: güvenilen plugin LLM tamamlama geçersiz kılmaları için kurallı `provider/model` hedeflerinden oluşan isteğe bağlı izin listesi. Yalnızca herhangi bir modele bilerek izin vermek istediğinizde `"*"` kullanın.
- `plugins.entries.<id>.llm.allowAgentIdOverride`: varsayılan olmayan bir ajan kimliğine karşı `api.runtime.llm.complete` çalıştırması amacıyla bu plugine açıkça güvenin.
- `plugins.entries.<id>.config`: plugin tarafından tanımlanan yapılandırma nesnesi (mevcut olduğunda yerel OpenClaw plugin şeması tarafından doğrulanır).
- Kanal plugini hesabı/çalışma zamanı ayarları `channels.<id>` altında bulunur ve merkezi bir OpenClaw seçenek kayıt defteri tarafından değil, sahibi olan pluginin manifestindeki `channelConfigs` meta verileriyle açıklanmalıdır.

### Codex çalıştırma çerçevesi plugin yapılandırması

Paket grubuna dahil `codex` plugini, yerel Codex uygulama sunucusu çalıştırma çerçevesi ayarlarını
`plugins.entries.codex.config` altında yönetir. Yapılandırma yüzeyinin tamamı için
[Codex çalıştırma çerçevesi başvurusuna](/tr/plugins/codex-harness-reference), çalışma zamanı modeli için
[Codex çalıştırma çerçevesine](/tr/plugins/codex-harness) bakın.

`codexPlugins` yalnızca yerel Codex çalıştırma çerçevesini seçen oturumlara uygulanır.
OpenClaw sağlayıcı çalıştırmaları, ACP konuşma bağlamaları veya Codex dışındaki
herhangi bir çalıştırma çerçevesi için Codex pluginlerini etkinleştirmez.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
                allow_destructive_actions: false,
              },
            },
          },
        },
      },
    },
  },
}
```

- `plugins.entries.codex.config.codexPlugins.enabled`: Codex çalıştırma çerçevesi için yerel Codex
  plugin/uygulama desteğini etkinleştirir. Varsayılan: `false`.
- `plugins.entries.codex.config.codexPlugins.allow_all_plugins`: kimliği doğrulanmış Codex hesabına bağlı,
  o anda erişilebilir olan her uygulamayı her yeni yerel Codex iş parçacığında
  kullanıma sunar. Varsayılan: `false`.
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions`:
  yapılandırılmış plugin uygulaması istemleri için varsayılan yıkıcı eylem politikası.
  Güvenli Codex onay şemalarını istem göstermeden kabul etmek için `true`, bunları
  reddetmek için `false`, Codex'in gerektirdiği onayları OpenClaw
  plugin onayları üzerinden yönlendirmek için `"auto"` veya kalıcı onay olmadan
  her plugin yazma/yıkıcı eylemi için istem göstermek üzere `"ask"` kullanın.
  `"ask"` modu, etkilenen uygulamanın araç başına kalıcı Codex onayı
  geçersiz kılmalarını temizler ve Codex iş parçacığı başlamadan önce o uygulama için
  insan onayları inceleyicisini seçer.
  Varsayılan: `true`.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.enabled`: genel `codexPlugins.enabled` değeri de doğru olduğunda
  yapılandırılmış bir plugin girdisini etkinleştirir.
  Açık girdiler için varsayılan: `true`.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.marketplaceName`:
  her çözümlenen girdi için `pluginName` ile birlikte gerekli olan kararlı pazar yeri
  kimliği. `"openai-curated"` ve `"workspace-directory"` desteklenir. Kimlik
  alanlarından biri eksik olan girdiler yok sayılır.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.pluginName`: `marketplaceName` ile birlikte gerekli olan
  kararlı Codex plugin kimliği. Bir
  `workspace-directory` girdisi, `plugin/list` tarafından döndürülen pazar yeri nitelemeli
  `summary.id` değerini tam olarak kullanmalıdır; örneğin
  `"example-plugin@workspace-directory"`.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.allow_destructive_actions`:
  plugin başına yıkıcı eylem geçersiz kılması. Belirtilmediğinde genel
  `allow_destructive_actions` değeri kullanılır. Plugin başına değer, aynı
  `true`, `false`, `"auto"` veya `"ask"` politikalarını kabul eder.

`"ask"` kullanan, kabul edilmiş her plugin uygulaması; o uygulamanın onay isteklerini
insan inceleyiciye yönlendirir. Diğer uygulamalar ve uygulama dışı iş parçacığı onayları
yapılandırılmış inceleyicilerini korur; böylece karma plugin politikaları
`"ask"` davranışını devralmaz.

`codexPlugins.enabled` genel etkinleştirme yönergesidir. Geçiş tarafından yazılan açık plugin
girdileri, kalıcı ve seçilmiş kurulum ve onarım uygunluğu kümesidir. Elle yapılandırılan
`workspace-directory` girdileri önceden kurulmuş ve etkinleştirilmiş olmalı, sahibi oldukları
uygulamalar da erişilebilir olmalıdır; OpenClaw bunları kurmaz veya kimliklerini
doğrulamaz. Codex açık çalışma alanı katalog isteğini reddederse etkin çalışma alanı
girdileri `marketplace_missing` ile kapalı durumda başarısız olurken varsayılan katalogdaki
seçilmiş girdiler kullanılabilir kalır. `plugins["*"]` desteklenmez, `install`
anahtarı yoktur ve yerel `marketplacePath` değerleri ana makineye özgü oldukları için
bilerek yapılandırma alanı olarak tanımlanmamıştır. Uygulama sunucusu sürümü ve hazırlık
gereksinimleri için [Yerel Codex pluginleri](/tr/plugins/codex-native-plugins) bölümüne bakın.

`app/list` hazırlık denetimleri bir saat boyunca önbelleğe alınır ve eskidiğinde
eşzamansız olarak yenilenir. Codex iş parçacığı uygulama yapılandırması her turda değil,
Codex çalıştırma çerçevesi oturumu oluşturulurken hesaplanır; yerel plugin yapılandırmasını
değiştirdikten sonra `/new`, `/reset` veya Gateway'i yeniden başlatma
seçeneklerinden birini kullanın.

`codexPlugins.allow_all_plugins`, o anda erişilebilir olan her hesap uygulamasını her yeni yerel Codex
iş parçacığına anlık görüntü olarak ekler. Pluginleri veya uygulamaları kurmaz ve
erişilemeyen uygulamalar hariç tutulmaya devam eder. Hesap uygulamaları genel
`codexPlugins.allow_destructive_actions` politikasını kullanır. Aynı uygulama her iki yolda da mevcutsa açık
plugin girdileri önceliklidir. `app/list` okunamazsa hesap genelinde kullanıma
sunma işlemi kapalı durumda başarısız olur.

- `plugins.entries.firecrawl.config.webFetch`: Firecrawl web getirme sağlayıcısı ayarları.
  - `apiKey`: Daha yüksek sınırlar için isteğe bağlı Firecrawl API anahtarı (SecretRef kabul eder). `plugins.entries.firecrawl.config.webSearch.apiKey` veya `FIRECRAWL_API_KEY` ortam değişkenine geri döner.
  - `baseUrl`: Firecrawl API temel URL'si (varsayılan: `https://api.firecrawl.dev`; kendi barındırdığınız geçersiz kılmalar özel/dahili uç noktaları hedeflemelidir).
  - `onlyMainContent`: sayfalardan yalnızca ana içeriği çıkarır (varsayılan: `true`).
  - `maxAgeMs`: milisaniye cinsinden azami önbellek yaşı (varsayılan: `172800000` / 2 gün).
  - `timeoutSeconds`: saniye cinsinden kazıma isteği zaman aşımı (varsayılan: `60`).
- `plugins.entries.xai.config.xSearch`: xAI X Search (Grok web araması) ayarları.
  - `enabled`: X Search sağlayıcısını etkinleştirir.
  - `model`: arama için kullanılacak Grok modeli (ör. `"grok-4.3"`).
- `plugins.entries.memory-core.config.dreaming`: bellek Dreaming ayarları. Aşamalar ve eşikler için [Dreaming](/tr/concepts/dreaming) bölümüne bakın.
  - `enabled`: ana Dreaming anahtarı (varsayılan `false`).
  - `frequency`: her tam Dreaming taraması için Cron sıklığı (varsayılan olarak `"0 3 * * *"`).
  - `model`: isteğe bağlı Dream Diary alt ajan modeli geçersiz kılması. `plugins.entries.memory-core.subagent.allowModelOverride: true` gerektirir; hedefleri kısıtlamak için `allowedModels` ile eşleştirin. Modelin kullanılamadığı hatalarda oturumun varsayılan modeliyle bir kez daha denenir; güven veya izin listesi hatalarında sessizce geri dönüş yapılmaz.
  - aşama politikası ve eşikler uygulama ayrıntılarıdır (kullanıcıya yönelik yapılandırma anahtarları değildir).
- Bellek yapılandırmasının tamamı [Bellek yapılandırması başvurusunda](/tr/reference/memory-config) bulunur:
  - `memory.search.*`
  - ajan başına geçersiz kılmalar için `agents.entries.*.memory.search.*`
  - `memory.backend`
  - `memory.citations`
  - `memory.qmd.*`
  - `plugins.entries.memory-core.config.dreaming`
- Etkin Claude paket grubu pluginleri, `settings.json` üzerinden gömülü OpenClaw varsayılanları da sağlayabilir; OpenClaw bunları ham OpenClaw yapılandırma yamaları olarak değil, arındırılmış ajan ayarları olarak uygular.
- `plugins.slots.memory`: etkin bellek plugini kimliğini seçin veya bellek pluginlerini devre dışı bırakmak için `"none"` kullanın.
- `plugins.slots.contextEngine`: etkin bağlam motoru plugini kimliğini seçin; başka bir motor kurup seçmediğiniz sürece varsayılan değer `"legacy"` olur.

[Pluginler](/tr/tools/plugin) bölümüne bakın.

---

## Tarayıcı

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // opt in only for trusted private-network access
      // allowPrivateNetwork: true, // legacy alias
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    tabCleanup: {
      enabled: true,
      idleMinutes: 120,
      maxTabsPerSession: 8,
      sweepMinutes: 5,
    },
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: { driver: "existing-session", attachOnly: true, color: "#00AA00" },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
    color: "#FF4500",
    // headless: false,
    // noSandbox: false,
    // extraArgs: [],
    // executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    // attachOnly: false,
  },
}
```

- `evaluateEnabled: false`, `act:evaluate` ve `wait --fn` özelliklerini devre dışı bırakır.
- `tabCleanup`, boşta kalma süresinden sonra veya bir oturum sınırını aştığında izlenen birincil ajan
  sekmelerinin periyodik olarak imkânlar ölçüsünde temizlenmesini denetler. İzleme yalnızca
  tarayıcı aracı `action: "open"` tarafından oluşturulan sekmelere uygulanır; kullanıcı tarafından açılan veya
  sahipliği bilinmeyen sekmeler hiçbir zaman devralınmaz. `tabCleanup` özelliğinin devre dışı bırakılması, açık oturum yaşam döngüsü temizliğini devre dışı bırakmaz.
- Kararlı bir yerel CDP hedefi ve tarayıcı kimliğiyle ana makine üzerinde yerel olarak açılan sekmeler,
  paylaşılan SQLite durumunda saklanır ve Gateway yeniden başlatmaları boyunca
  `/new` ve oturum yaşam döngüsü temizliği için uygun kalır. Yerel araçlara yönelik CDP hedefleri de
  yeniden başlatmanın ardından boşta kalma ve sınır temizliği için uygun kalır. Chrome MCP,
  işlem yerelindeki hedef tanıtıcılarını kullanır; bu nedenle mevcut soğuk oturum kayıtları, yeniden başlatma sonrası
  ilişkilendirilemeyen etkinliğe karşı boşta kalma taraması yapma riskine girmek yerine
  yaşam döngüsü temizliğini bekler. OpenClaw, kapatmadan önce profili ve tarayıcı örneğini
  doğrular. Chrome MCP otomatik bağlantısı, eksik `/json/version` tarayıcı
  kimliği ve çözümlenmemiş yerel hedefler tamamen işlem yerelinde kalır; dolayısıyla
  yeniden başlatmanın ardından otomatik olarak kapatılmaz. İzlenmeyen eski sekmelerin
  elle kapatılması gerekir. Geçici hatalar, daha sonra yeniden denenmek üzere beklemede kalır. Bkz.
  [Sekme temizleme sahipliği](/tr/tools/browser#tab-cleanup-ownership).
- `ssrfPolicy.dangerouslyAllowPrivateNetwork` ayarlanmadığında devre dışıdır; böylece tarayıcı gezinmesi varsayılan olarak katı kalır.
- `ssrfPolicy.dangerouslyAllowPrivateNetwork: true` değerini yalnızca özel ağdaki tarayıcı gezinmesine bilinçli olarak güvendiğinizde ayarlayın.
- Katı modda, uzak CDP profil uç noktaları (`profiles.*.cdpUrl`) erişilebilirlik/keşif denetimleri sırasında aynı özel ağ engellemesine tabidir.
- `ssrfPolicy.allowPrivateNetwork`, eski bir takma ad olarak desteklenmeye devam eder.
- Katı modda, açık istisnalar için `ssrfPolicy.hostnameAllowlist` ve `ssrfPolicy.allowedHostnames` kullanın.
- Uzak profiller yalnızca bağlanmaya yöneliktir (başlatma/durdurma/sıfırlama devre dışıdır).
- `profiles.*.cdpUrl`; `http://`, `https://`, `ws://` ve `wss://` değerlerini kabul eder.
  OpenClaw'ın `/json/version` öğesini keşfetmesini istediğinizde HTTP(S) kullanın; sağlayıcınız
  doğrudan bir DevTools WebSocket URL'si verdiğinde WS(S) kullanın.
- Haricen yönetilen bir CDP hizmetine geri döngü üzerinden erişilebiliyorsa, söz konusu
  profilin `attachOnly: true` değerini ayarlayın; aksi takdirde OpenClaw geri döngü portunu
  yerel olarak yönetilen bir tarayıcı profili olarak değerlendirir ve yerel port sahipliği hataları bildirebilir.
- `existing-session` profilleri CDP yerine Chrome MCP kullanır ve
  seçilen ana makinede veya bağlı bir tarayıcı Node'u üzerinden bağlanabilir.
- `existing-session` profilleri, Brave veya Edge gibi belirli bir
  Chromium tabanlı tarayıcı profilini hedeflemek için `userDataDir` değerini ayarlayabilir.
- `existing-session` profilleri, Chrome zaten bir DevTools HTTP(S) keşif uç noktası
  veya doğrudan WS(S) uç noktası arkasında çalışıyorsa `cdpUrl` değerini ayarlayabilir. Bu
  modda OpenClaw, otomatik bağlantıyı kullanmak yerine uç noktasını Chrome MCP'ye iletir;
  Chrome MCP başlatma bağımsız değişkenleri için `userDataDir` yok sayılır.
- `existing-session` profilleri mevcut Chrome MCP rota sınırlarını korur:
  CSS seçicisiyle hedefleme yerine anlık görüntü/referans odaklı eylemler, tek dosyalı yükleme
  kancaları, iletişim kutusu zaman aşımı geçersiz kılmaları yoktur, `wait --load networkidle` yoktur ve
  `responsebody`, PDF dışa aktarma, indirme müdahalesi veya toplu eylemler yoktur.
- Yerel olarak yönetilen `openclaw` profilleri `cdpPort` ve `cdpUrl` değerlerini otomatik olarak atar;
  `cdpUrl` değerini yalnızca uzak CDP profilleri veya mevcut oturum uç noktasına
  bağlanma için açıkça ayarlayın.
- Yerel olarak yönetilen profiller, söz konusu profil için genel
  `browser.executablePath` değerini geçersiz kılmak üzere `executablePath` değerini ayarlayabilir. Bunu bir profili
  Chrome'da, diğerini Brave'de çalıştırmak için kullanın.
- Otomatik algılama sırası: Chromium tabanlıysa varsayılan tarayıcı → Chrome → Brave → Edge → Chromium → Chrome Canary.
- `browser.executablePath` ve `browser.profiles.<name>.executablePath`, Chromium başlatılmadan önce işletim sistemi giriş dizininiz için
  hem `~` hem de `~/...` değerlerini kabul eder.
  `existing-session` profillerindeki profil başına `userDataDir` değerinde de tilde genişletmesi yapılır.
- Denetim hizmeti: yalnızca geri döngü (`gateway.port` değerinden türetilen port, varsayılan `18791`).
- `extraArgs`, yerel Chromium başlangıcına ek başlatma bayrakları ekler (örneğin
  `--disable-gpu`, pencere boyutlandırma veya hata ayıklama bayrakları).

---

## Kullanıcı Arayüzü

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // emoji, kısa metin, görüntü URL'si veya veri URI'si
    },
    prefs: {
      theme: "claw", // claw | knot | dash | custom
      themeMode: "system", // light | dark | system
      locale: "en",
      chatShowThinking: true,
      chatShowToolCalls: true,
      chatPersistCommentary: true, // Çalıştırmalardan sonra açıklamaları Control UI'da tutar; kanallara iletmez
      chatSendShortcut: "enter", // enter | modifier-enter
      chatFollowUpMode: "steer", // steer | queue; sunucu kuyruk modunu kullanmak için atlayın
      showAdvancedSettings: false, // Settings içindeki tüm Advanced gruplarını genişletir
    },
  },
}
```

- `seamColor`: yerel uygulama kullanıcı arayüzü çerçevesinin vurgu rengi (Talk Mode balon tonu vb.).
- `assistant`: Control UI kimliğini geçersiz kılma. Etkin ajan kimliğine geri döner.
- `prefs`: cihazlar arası operatör tercihleri. Bu, ajanların
  onay geçidi üzerinden bunları değiştirebilmesi ve tüm Control UI istemcilerinin eşzamanlı
  kalması için standart konumdur; tarayıcılar anında başlatma için değerleri yerel depolamaya yansıtır ve
  yapılandırmaya yazamadıklarında (görüntüleyici kapsamı, çevrimdışı) cihaz yerelinde bir kopya
  tutar. `chatPersistCommentary` varsayılan olarak `true` değerini kullanır. Bunu `false` olarak ayarlamak, çalıştırma
  sırasında canlı açıklamaları görünür tutar ancak tamamlandığında kaldırır ve yeni
  Codex açıklamalarının kalıcı transkript yansısına girmesini engeller. Mesajlaşma kanalı
  üzerinden iletim ayrı kalır ve değişmez.
  `showAdvancedSettings` varsayılan olarak `false` değerini kullanır; Settings araması, bu tercihi
  değiştirmeden eşleşen bir gelişmiş grubu geçici olarak açabilir.
  Metin ölçeği, sohbet genişliği ve canlı kenar çubuğu etkinliği gibi yalnızca sunuma yönelik
  tercihler tarayıcı yerelinde kalır ve Settings içinde yapılandırılır.
  Bağlı istemciler sunucu tarafındaki değişiklikleri canlı olarak uygular: Gateway, kalıcılaştırılan
  her yapılandırma yazımından sonra yalnızca karma değeri içeren bir `config.changed` olayı yayınlar ve
  istemciler anlık görüntülerini yeniler (yerel bir ayar taslağında kaydedilmemiş
  düzenlemeler varken atlanır). Yeniden bağlanan istemciler bağlantı sırasında uzlaştırılır.

---

## Gateway

```json5
{
  gateway: {
    mode: "local", // local | remote
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token", // none | token | password | trusted-proxy
      token: "your-token",
      // password: "your-password", // veya OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // mode=trusted-proxy için; bkz. /gateway/trusted-proxy-auth
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,
      },
    },
    tailscale: {
      mode: "off", // off | serve | funnel
      resetOnExit: false,
    },
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      // root: "dist/control-ui",
      // toolTitles: false, // araç çağrıları için isteğe bağlı yapay zekâ amaç başlıkları (yardımcı model token'ları harcar)
      // embedSandbox: "scripts", // strict | scripts | trusted
      // allowExternalEmbedUrls: false, // tehlikeli: mutlak harici http(s) yerleştirme URL'lerine izin verir
      // allowedOrigins: ["https://control.example.com"], // geri döngü dışındaki Control UI için gereklidir
      // dangerouslyAllowHostHeaderOriginFallback: false, // tehlikeli Host üst bilgisi kaynak geri dönüş modu
    },
    terminal: {
      enabled: false,
      // shell: "/bin/zsh",
    },
    remote: {
      url: "ws://127.0.0.1:18789",
      transport: "ssh", // ssh | direct
      token: "your-token",
      // password: "your-password",
    },
    trustedProxies: ["10.0.0.1"],
    // İsteğe bağlı. Varsayılan değer false.
    allowRealIpFallback: false,
    nodes: {
      pairing: {
        // İsteğe bağlı. Varsayılan olarak ayarlanmamış/devre dışıdır.
        autoApproveCidrs: ["192.168.1.0/24", "fd00:1234:5678::/64"],
        // SSH ile doğrulanan otomatik onay. Varsayılan: etkin (true).
        // Yalnızca SSH doğrulamasını devre dışı bırakmak için false olarak ayarlayın; bu,
        // yukarıdaki autoApproveCidrs değerini etkilemez. Yalnızca elle Node eşleştirmesi için false olarak ayarlayın VE
        // autoApproveCidrs değerini kaldırın. İnce ayar yapmak için bir nesne iletin: { user, identity,
        // timeoutMs, cidrs }.
        sshVerify: true,
      },
      commands: {
        allow: ["canvas.navigate"],
        deny: ["system.run"],
      },
    },
    tools: {
      // Ek /tools/invoke HTTP engellemeleri
      deny: ["browser"],
      // Sahip/yönetici çağıranlar için araçları varsayılan HTTP engelleme listesinden kaldırır
      allow: ["gateway"],
    },
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000,
        },
      },
    },
  },
}
```

<Accordion title="Gateway alan ayrıntıları">

- `mode`: `local` (gateway'i çalıştır) veya `remote` (uzak gateway'e bağlan). `local` olmadığı sürece Gateway başlatılmayı reddeder.
- `port`: WS + HTTP için tek çoklanmış bağlantı noktası. Öncelik: `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`.
- `bind`: `auto`, `loopback` (varsayılan), `lan` (`0.0.0.0`), `tailnet` (varsa Tailscale IPv4, aksi hâlde geri döngü) veya `custom` (bir IPv4 adresi). Çözümlenmiş bir `tailnet` adresi ve `127.0.0.1` ya da `0.0.0.0` dışındaki herhangi bir `custom` adresi, aynı ana makinedeki istemciler için aynı bağlantı noktasında `127.0.0.1` gerektirir; dinleyicilerden biri bağlanamazsa başlatma başarısız olur. Geri döngü dışına açılma, seçilen arayüzle sınırlı kalır.
- **Eski bağlama takma adları**: ana makine takma adları (`0.0.0.0`, `127.0.0.1`, `localhost`, `::`, `::1`) yerine `gateway.bind` içindeki bağlama modu değerlerini (`auto`, `loopback`, `lan`, `tailnet`, `custom`) kullanın.
- **Docker notu**: varsayılan `loopback` bağlaması, kapsayıcı içinde `127.0.0.1` üzerinde dinler. Docker köprü ağıyla (`-p 18789:18789`) trafik `eth0` üzerinden gelir, dolayısıyla gateway'e erişilemez. `--network host` kullanın veya tüm arayüzlerde dinlemek için `bind: "lan"` (ya da `customBindHost: "0.0.0.0"` ile `bind: "custom"`) ayarlayın.
- **Kimlik doğrulama**: varsayılan olarak gereklidir. Geri döngü dışındaki bağlamalar gateway kimlik doğrulaması gerektirir. Uygulamada bu, paylaşılan bir belirteç/parola veya `gateway.auth.mode: "trusted-proxy"` içeren kimlik bilgisine duyarlı bir ters proxy anlamına gelir. İlk katılım sihirbazı varsayılan olarak bir belirteç oluşturur.
- Hem `gateway.auth.token` hem de `gateway.auth.password` yapılandırılmışsa (SecretRef'ler dâhil), `gateway.auth.mode` değerini açıkça `token` veya `password` olarak ayarlayın. Her ikisi de yapılandırılmışken mod ayarlanmamışsa başlatma ve hizmet yükleme/onarım akışları başarısız olur.
- `gateway.auth.mode: "none"`: açık kimlik doğrulamasız mod. Yalnızca güvenilir yerel geri döngü kurulumlarında kullanın; bu seçenek ilk katılım istemlerinde kasıtlı olarak sunulmaz.
- `gateway.auth.mode: "trusted-proxy"`: tarayıcı/kullanıcı kimlik doğrulamasını kimlik bilgisine duyarlı bir ters proxy'ye devredin ve `gateway.trustedProxies` kaynaklı kimlik başlıklarına güvenin (bkz. [Güvenilir Proxy Kimlik Doğrulaması](/tr/gateway/trusted-proxy-auth)). Bu mod varsayılan olarak **geri döngü dışındaki** bir proxy kaynağı bekler; aynı ana makinedeki geri döngü ters proxy'leri açıkça `gateway.auth.trustedProxy.allowLoopback = true` gerektirir. Aynı ana makinedeki dâhilî çağıranlar, yerel doğrudan geri dönüş olarak `gateway.auth.password` kullanabilir; `gateway.auth.token` güvenilir proxy moduyla birbirini dışlamaya devam eder.
- `gateway.auth.allowTailscale`: `true` olduğunda Tailscale Serve kimlik başlıkları Control UI/WebSocket kimlik doğrulamasını karşılayabilir (`tailscale whois` aracılığıyla doğrulanır). HTTP API uç noktaları bu Tailscale başlık kimlik doğrulamasını **kullanmaz**; bunun yerine gateway'in normal HTTP kimlik doğrulama modunu izler. Bu belirteçsiz akış, gateway ana makinesinin güvenilir olduğunu varsayar. `tailscale.mode = "serve"` olduğunda varsayılan değer `true` olur.
- `gateway.auth.rateLimit`: isteğe bağlı başarısız kimlik doğrulama sınırlayıcısı. İstemci IP'si ve kimlik doğrulama kapsamı başına uygulanır (paylaşılan gizli bilgi ve cihaz belirteci bağımsız olarak izlenir). Engellenen denemeler `429` + `Retry-After` döndürür.
  - Eşzamansız Tailscale Serve Control UI yolunda, aynı `{scope, clientIp}` için başarısız denemeler, başarısızlık yazılmadan önce sıralı hâle getirilir. Bu nedenle aynı istemciden gelen eşzamanlı hatalı denemeler, her ikisinin de sıradan uyuşmazlıklar olarak yarışıp geçmesi yerine ikinci istekte sınırlayıcıyı tetikleyebilir.
  - `gateway.auth.rateLimit.exemptLoopback` varsayılan olarak `true` değerindedir; localhost trafiğinin de kasıtlı olarak hız sınırlamasına tabi olmasını istediğinizde (test kurulumları veya katı proxy dağıtımları için) `false` ayarlayın.
- Tarayıcı kaynaklı WS kimlik doğrulama denemeleri, geri döngü muafiyeti devre dışı bırakılarak her zaman sınırlandırılır (tarayıcı tabanlı localhost kaba kuvvet saldırılarına karşı derinlemesine savunma).
- Geri döngüde bu tarayıcı kaynaklı kilitlemeler, normalleştirilmiş `Origin`
  değeri başına yalıtılır; böylece bir localhost kaynağından gelen tekrarlanan başarısızlıklar
  farklı bir kaynağı otomatik olarak kilitlemez.
- `tailscale.mode`: `serve` (yalnızca tailnet, geri döngü bağlaması) veya `funnel` (herkese açık, kimlik doğrulama gerektirir).
- `tailscale.serviceName`: Serve modu için isteğe bağlı Tailscale Service adı; örneğin
  `svc:openclaw`. Ayarlandığında OpenClaw bunu `tailscale serve
--service` öğesine iletir; böylece Control UI, cihaz ana makine adı
  yerine adlandırılmış bir Service üzerinden kullanıma sunulabilir. Değer, Tailscale'in `svc:<dns-label>`
  Service adı biçimini kullanmalıdır; başlatma işlemi türetilen Service URL'sini bildirir.
- `tailscale.preserveFunnel`: `true` ve `tailscale.mode = "serve"` olduğunda OpenClaw,
  başlatma sırasında Serve'ü yeniden uygulamadan önce `tailscale funnel status` öğesini denetler ve
  harici olarak yapılandırılmış bir Funnel rotası gateway bağlantı noktasını zaten kapsıyorsa işlemi atlar.
  Varsayılan `false`.
- `controlUi.allowedOrigins`: Gateway WebSocket bağlantıları için açık tarayıcı kaynağı izin listesi. Herkese açık geri döngü dışı tarayıcı kaynakları için gereklidir. Geri döngü, RFC1918/bağlantı-yerel, `.local`, `.ts.net` veya Tailscale CGNAT ana makinelerinden gelen özel aynı kaynaklı LAN/Tailnet UI yüklemeleri, Host başlığı geri dönüşü etkinleştirilmeden kabul edilir.
- `controlUi.toolTitles`: Control UI sohbetindeki araç çağrıları için yapay zekâ tarafından oluşturulan amaç başlıklarını etkinleştirin. Varsayılan: `false` (araçların görüntülenmesi, arka planda model çağrısı olmadan tamamen belirlenimsel kalır). Etkinleştirildiğinde `chat.toolTitles` yöntemi, karmaşık çağrıları standart yardımcı model yönlendirmesi üzerinden etiketler — ajanın `utilityModel` değeri (her yardımcı görevde olduğu gibi sınırlı araç bağımsız değişkenlerini seçilen sağlayıcıya gönderebilecek bir operatör kararı) veya oturum sağlayıcısının bildirdiği küçük model varsayılanı (OpenAI → `gpt-5.6-luna`, Anthropic → `claude-haiku-4-5`) — ve sonuçları ajan başına durum veritabanında önbelleğe alır; böylece yinelenen görüntülemeler hiçbir zaman yeniden ücretlendirilmez. `utilityModel: \"\"`, diğer tüm yardımcı görevlerde olduğu gibi başlıkları devre dışı bırakır; başlıklar hiçbir zaman birincil modele geri dönmez.
- `controlUi.dangerouslyAllowHostHeaderOriginFallback`: kasıtlı olarak Host başlığı kaynak politikasına dayanan dağıtımlar için Host başlığı kaynak geri dönüşünü etkinleştiren tehlikeli mod.
- `terminal.enabled`: yönetici kapsamlı operatör terminalini etkinleştirin. Varsayılan: `false`. Terminal, seçilen ajan çalışma alanında bir ana makine PTY'si başlatır, Gateway işleminin ortamını devralır ve `sandbox.mode: "all"` değerine sahip ajanlar için reddedilir. Bunu yalnızca güvenilir operatör dağıtımlarında etkinleştirin; değiştirilmesi Gateway'i yeniden başlatır ve Control UI içerik güvenliği politikasını günceller.
- `terminal.shell`: isteğe bağlı kabuk yürütülebilir dosyası. Ayarlanmadığında OpenClaw, Unix'te `$SHELL` ve Windows'ta `%ComSpec%` kullanır.
- `terminal.detachedSessionTimeoutSeconds`: bir terminal oturumunun bağlantısı kesildikten sonra (sayfanın yeniden yüklenmesi, dizüstü bilgisayarın uykuya geçmesi) ne kadar süreyle yaşamaya devam edeceği; bu süre boyunca son çıktısı yeniden oynatılarak `terminal.attach` aracılığıyla yeniden bağlanılabilir durumda kalır. Varsayılan: `300`. Bağlantıları kesildiği anda oturumları sonlandırmak için `0` ayarlayın. Ayrılmış oturumlar komutlarını çalıştırmaya devam eder; bu nedenle paylaşılan veya dışa açık ana makinelerde bu süreyi kısaltın.
- `remote.transport`: `ssh` (varsayılan) veya `direct` (ws/wss). `direct` için herkese açık ana makinelerde `remote.url`, `wss://` olmalıdır; düz metin `ws://` yalnızca geri döngü, LAN, bağlantı-yerel, `.local`, `.ts.net` ve Tailscale CGNAT ana makinelerinde kabul edilir.
- `remote.remotePort`: uzak SSH ana makinesindeki gateway bağlantı noktası. Varsayılan değer `18789`; yerel tünel bağlantı noktası uzak gateway bağlantı noktasından farklı olduğunda bunu kullanın.
- `remote.tlsFingerprint`: uzak bir `wss://` Gateway için beklenen SHA-256 sertifika parmak izi. macOS uygulaması bunu hem operatör/denetim hem de eşlikçi Node bağlantılarına uygular. Açık bir değer olmadığında macOS, yalnızca normal sistem güven doğrulaması başarılı olduktan sonra ilk kullanım sabitlemesini kaydeder.
- `remote.sshHostKeyPolicy`: macOS SSH tüneli ana makine anahtarı politikası. `strict` varsayılandır ve önceden güvenilen bir anahtar gerektirir. `openssh`, yönetilen takma adlar için geçerli OpenSSH yapılandırmasına açıkça katılım sağlar; kullanmadan önce eşleşen kullanıcı ve sistem SSH ayarlarını inceleyin. macOS uygulaması ve `configure-remote`, hedefler değiştirilirken yeniden açıkça etkinleştirilmediği sürece bu politikayı `strict` değerine sıfırlar.
- `gateway.remote.token` / `.password`, uzak istemci kimlik bilgisi alanlarıdır. Bunlar tek başlarına gateway kimlik doğrulamasını yapılandırmaz.
- `gateway.push.apns.relay.baseUrl`: aktarıcı destekli iOS derlemeleri kayıtları gateway'de yayımladıktan sonra kullanılan harici APNs aktarıcısının temel HTTPS URL'si. Herkese açık App Store derlemeleri, barındırılan OpenClaw aktarıcısını kullanır. Özel aktarıcı URL'leri, aktarıcı URL'si söz konusu aktarıcıyı gösteren ve kasıtlı olarak ayrı tutulan bir iOS derleme/dağıtım yoluyla eşleşmelidir.
- `gateway.push.apns.relay.timeoutMs`: gateway'den aktarıcıya gönderim zaman aşımı, milisaniye cinsinden. Varsayılan değer `10000`.
- Aktarıcı destekli kayıtlar belirli bir gateway kimliğine devredilir. Eşleştirilmiş iOS uygulaması `gateway.identity.get` öğesini alır, bu kimliği aktarıcı kaydına dâhil eder ve kayıt kapsamlı bir gönderim iznini gateway'e iletir. Başka bir gateway, saklanan bu kaydı yeniden kullanamaz.
- `OPENCLAW_APNS_RELAY_BASE_URL` / `OPENCLAW_APNS_RELAY_TIMEOUT_MS`: yukarıdaki aktarıcı yapılandırması için geçici ortam geçersiz kılmaları.
- `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true`: geri döngü HTTP aktarıcı URL'leri için yalnızca geliştirmeye yönelik kaçış yolu. Üretim aktarıcı URL'leri HTTPS üzerinde kalmalıdır.
- `OPENCLAW_HANDSHAKE_TIMEOUT_MS`: yerleşik kimlik doğrulama öncesi Gateway WebSocket el sıkışma zaman aşımı için isteğe bağlı ortam geçersiz kılması.
- `channels.<provider>.healthMonitor.enabled`: genel izleyiciyi etkin tutarken sistem durumu izleyicisinin yeniden başlatmalarını kanal bazında devre dışı bırakma.
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: çok hesaplı kanallar için hesap bazında geçersiz kılma. Ayarlandığında kanal düzeyindeki geçersiz kılmaya göre önceliklidir.
- Yerel gateway çağrı yolları, yalnızca `gateway.auth.*` ayarlanmamışsa `gateway.remote.*` öğesini geri dönüş olarak kullanabilir.
- `gateway.auth.token` / `gateway.auth.password`, SecretRef aracılığıyla açıkça yapılandırılmış ve çözümlenmemişse çözümleme güvenli biçimde başarısız olur (uzak geri dönüşle maskeleme yapılmaz).
- `trustedProxies`: TLS'yi sonlandıran veya iletilen istemci başlıkları ekleyen ters proxy IP'leri. Yalnızca denetiminizdeki proxy'leri listeleyin. Geri döngü girdileri aynı ana makinedeki proxy/yerel algılama kurulumları (örneğin Tailscale Serve veya yerel bir ters proxy) için hâlâ geçerlidir, ancak geri döngü isteklerini `gateway.auth.mode: "trusted-proxy"` için uygun **hâle getirmez**.
- `allowRealIpFallback`: `true` olduğunda gateway, `X-Forwarded-For` eksikse `X-Real-IP` öğesini kabul eder. Güvenli biçimde başarısız olma davranışı için varsayılan `false`.
- `gateway.nodes.pairing.autoApproveCidrs`: kapsam istenmeden ilk kez yapılan Node cihaz eşleştirmesini otomatik olarak onaylamak için isteğe bağlı CIDR/IP izin listesi. Ayarlanmadığında devre dışıdır. Bu, operatör/tarayıcı/Control UI/WebChat eşleştirmesini otomatik olarak onaylamaz ve rol, kapsam, meta veri veya ortak anahtar yükseltmelerini otomatik olarak onaylamaz.
- `gateway.nodes.pairing.sshVerify`: ilk kez yapılan Node cihaz eşleştirmesi için SSH ile doğrulanan otomatik onay (varsayılan: etkin). Gateway, eşleştirme ana makinesine SSH ile geri bağlanır (BatchMode, katı ana makine anahtarları) ve yalnızca tam bir `openclaw node identity` cihaz anahtarı eşleşmesinde onay verir. `autoApproveCidrs` ile aynı asgari uygunluk koşulları geçerlidir; `cidrs` bunları geçersiz kılmadığı sürece yoklamalar özel/CGNAT kaynak adresleriyle sınırlıdır. Devre dışı bırakmak için `false`, ayarlamak için `{ user, identity, timeoutMs, cidrs }` kullanın. Bkz. [Node eşleştirmesi](/tr/gateway/pairing#ssh-verified-device-auto-approval-default).
- `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny`: eşleştirme ve platform izin listesi değerlendirmesinden sonra bildirilen Node komutları için genel izin verme/reddetme biçimlendirmesi. `camera.snap`, `camera.clip`, `screen.record`, `health.summary`, `sms.search` ve `sms.send` gibi tehlikeli Node komutlarını etkinleştirmek için `commands.allow` kullanın; `commands.deny`, bir platform varsayılanı veya açık izin normalde komutu dahil edecek olsa bile komutu kaldırır. iOS Health izni, Android SMS izni ve Gateway komut yetkilendirmesi birbirinden bağımsızdır. Bir Node, bildirdiği komut listesini değiştirdikten sonra Gateway'in güncellenmiş komut anlık görüntüsünü saklaması için söz konusu cihaz eşleştirmesini reddedip yeniden onaylayın.
- `gateway.tools.deny`: HTTP `POST /tools/invoke` için engellenen ek araç adları (varsayılan ret listesini genişletir).
- `gateway.tools.allow`: sahip/yönetici çağıranlar için araç adlarını varsayılan HTTP ret listesinden kaldırır. Bu, kimlik bilgisi taşıyan `operator.write` çağıranlara sahip/yönetici erişimi kazandırmaz; `cron`, `gateway` ve `nodes`, izin listesine alınmış olsalar bile sahip olmayan çağıranlar tarafından kullanılamaz.

</Accordion>

### OpenAI uyumlu uç noktalar

- Yönetici HTTP RPC: `admin-http-rpc` plugin'i olarak varsayılan şekilde kapalıdır. `POST /api/v1/admin/rpc` kaydını yapmak için plugin'i etkinleştirin. Bkz. [Yönetici HTTP RPC](/tr/plugins/admin-http-rpc).
- Sohbet Tamamlamaları: varsayılan olarak devre dışıdır. `gateway.http.endpoints.chatCompletions.enabled: true` ile etkinleştirin.
- Yanıtlar API'si: `gateway.http.endpoints.responses.enabled`.
- Yanıtlar URL girdisi sağlamlaştırması:
  - `gateway.http.endpoints.responses.maxUrlParts`
  - `gateway.http.endpoints.responses.files.urlAllowlist`
  - `gateway.http.endpoints.responses.images.urlAllowlist`
    Boş izin listeleri ayarlanmamış kabul edilir; URL getirmeyi devre dışı bırakmak için
    `gateway.http.endpoints.responses.files.allowUrl=false` ve/veya `gateway.http.endpoints.responses.images.allowUrl=false` kullanın.
- İsteğe bağlı yanıt sağlamlaştırma üstbilgisi:
  - `gateway.http.securityHeaders.strictTransportSecurity` (yalnızca denetiminizdeki HTTPS kaynakları için ayarlayın; bkz. [Güvenilir Proxy Kimlik Doğrulaması](/tr/gateway/trusted-proxy-auth#tls-termination-and-hsts))

### Çoklu örnek yalıtımı

Benzersiz bağlantı noktaları ve durum dizinleriyle tek bir ana makinede birden fazla Gateway çalıştırın:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json \
OPENCLAW_STATE_DIR=~/.openclaw-a \
openclaw gateway --port 19001
```

Kolaylık bayrakları: `--dev` (`~/.openclaw-dev` + `19001` bağlantı noktasını kullanır), `--profile <name>` (`~/.openclaw-<name>` kullanır).

Bkz. [Birden Fazla Gateway](/tr/gateway/multiple-gateways).

### `gateway.tls`

```json5
{
  gateway: {
    tls: {
      enabled: false,
      autoGenerate: false,
      certPath: "/etc/openclaw/tls/server.crt",
      keyPath: "/etc/openclaw/tls/server.key",
      caPath: "/etc/openclaw/tls/ca-bundle.crt",
    },
  },
}
```

- `enabled`: Gateway dinleyicisinde TLS sonlandırmasını (HTTPS/WSS) etkinleştirir (varsayılan: `false`).
- `autoGenerate`: açık dosyalar yapılandırılmadığında yerel, kendinden imzalı bir sertifika/anahtar çifti otomatik olarak oluşturur; yalnızca yerel/geliştirme kullanımı içindir.
- `certPath`: TLS sertifika dosyasının dosya sistemi yolu.
- `keyPath`: TLS özel anahtar dosyasının dosya sistemi yolu; izinlerini kısıtlı tutun.
- `caPath`: istemci doğrulaması veya özel güven zincirleri için isteğe bağlı CA paketi yolu.

### `gateway.reload`

```json5
{
  gateway: {
    reload: {
      mode: "hybrid", // kapalı | yeniden başlat | sıcak | karma
      debounceMs: 500,
      deferralTimeoutMs: 300000,
    },
  },
}
```

- `mode`: yapılandırma düzenlemelerinin çalışma zamanında nasıl uygulandığını denetler.
  - `"off"`: canlı düzenlemeleri yok sayar; değişiklikler açıkça yeniden başlatma gerektirir.
  - `"restart"`: yapılandırma değiştiğinde Gateway işlemini her zaman yeniden başlatır.
  - `"hot"`: değişiklikleri yeniden başlatmadan işlem içinde uygular.
  - `"hybrid"` (varsayılan): önce sıcak yeniden yüklemeyi dener; gerekirse yeniden başlatmaya geri döner.
- `debounceMs`: yapılandırma değişikliklerinin uygulanmasından önceki ms cinsinden bekletme penceresi (negatif olmayan tam sayı; varsayılan: `300`).
- `deferralTimeoutMs`: yeniden başlatmayı veya kanalın sıcak yeniden yüklenmesini zorlamadan önce sürmekte olan işlemleri beklemek için isteğe bağlı azami süre (ms). Varsayılan sınırlı beklemeyi (`300000`) kullanmak için bunu atlayın; süresiz beklemek ve hâlâ bekleyen işlemler için düzenli uyarılar günlüğe kaydetmek üzere `0` olarak ayarlayın.

---

## Bulut çalışanı ortamları

Bulut çalışanları isteğe bağlıdır. `cloudWorkers` yoksa veya `profiles` boşsa OpenClaw yeni çalışan oluşturulmasını kabul etmez. Daha önce oluşturulan kalıcı kayıtlar uzlaştırılmaya devam eder ve görünür kalır; mevcut Gateway/Node izdüşümü değişmez.

Her çalışan sağlayıcısı, güvenilir hazırlama çıktısından bir SSH `hostKey` değerini ana makine adı veya yorum olmadan tam olarak `algorithm base64` biçiminde döndürmelidir. Önyükleme bu anahtarı yalıtılmış bir `known_hosts` dosyasına yazar, `StrictHostKeyChecking=yes` kullanır ve sağlayıcı bunu atladığında bağlantı açmadan önce başarısız olur. İlk kullanımda güvenme yedeği yoktur.

Tünel kurulumu, hazırlamanın bir parçası olmak yerine isteğe bağlı olarak yapılır. Başlatıldığında Gateway, çalışana yerel bir Unix soketini kendi geri döngü WebSocket uç noktasına ters yönlendirir. Soket, rastgele tahsis edilen ve yalnızca sahibinin erişebildiği uzak bir dizinde bulunur; geri döngü TCP bağlantı noktasından farklı olarak çok kullanıcılı bir çalışandaki diğer hesaplar tarafından erişilemez ve başka bir ortamın bağlantı noktasıyla çakışamaz. SSH canlı tutma sinyalleri ve sınırlandırılmış yeniden bağlanma geri çekilmesi yalnızca tünel sahibi güncel kaldığı sürece çalışır. Tünelin durdurulması, SSH işlemini kapatmadan önce yeniden bağlantıları engeller.

Denetim trafiği ile çalışma alanı aktarımı ayrı SSH bağlantıları kullanır. Her ikisi de aynı çözümlenmiş kimliği ve yalıtılmış, sabitlenmiş `known_hosts` dosyasını yeniden kullanır; ancak çalışma alanı aktarımı, uzun ömürlü tünelle SSH bağlantısı çoğullamasını paylaşmaz, dolayısıyla rsync denetim trafiğini engelleyemez.

### Crabbox profili

Paketle gelen `crabbox` sağlayıcısı, yerel Crabbox CLI aracılığıyla SSH özellikli bir kiralama hazırlar. İçteki `settings.provider` Crabbox arka ucunu seçer; dıştaki OpenClaw sağlayıcı kimliğinden ayrıdır.

```json5
{
  cloudWorkers: {
    profiles: {
      production: {
        provider: "crabbox",
        install: "bundle", // Varsayılan; yalnızca yayımlanmış bir Gateway sürümü için "npm" kullanın.
        settings: {
          provider: "aws",
          class: "standard",
          ttl: "24h",
          idleTimeout: "60m",
          // İsteğe bağlı mutlak yol. Varsayılan: kardeş ../crabbox/bin/crabbox, ardından PATH.
          binary: "/usr/local/bin/crabbox",
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `settings.provider` (zorunlu): `--provider` aracılığıyla geçirilen Crabbox arka ucu. İnceleme çıktısı bir SSH uç noktası içeren bir arka uç kullanın; `aws` doğrudan AWS arka ucunu seçer.
- `settings.class` (zorunlu): `--class` öğesine geçirilen Crabbox makine sınıfı.
- `settings.ttl` ve `settings.idleTimeout` (zorunlu): `--ttl` ve `--idle-timeout` öğelerine geçirilen pozitif Go süre dizeleri. Sağlayıcı tarafındaki bu güvenlik önlemleri, aşağıdaki OpenClaw tarafından depolanan `lifetime` ilkesinden ayrıdır.
- `settings.binary`: isteğe bağlı mutlak Crabbox yürütülebilir dosya yolu. Bu olmadan OpenClaw önce kardeş Crabbox çıkışını, ardından `PATH` üzerindeki yürütülebilir girdileri denetler ve son olarak `crabbox` komutunu çağırır; böylece eksik bir CLI görünür bir sağlayıcı hatası olarak kalır.

Bilinmeyen ayarlar reddedilir. Crabbox kimlik bilgileri ve arka uca özgü hesap yapılandırması Crabbox'ın mülkiyetinde kalır; bunları `settings` içine koymayın. OpenClaw yalnızca yerel CLI'ı çağırır ve bu plugin'den hiçbir sağlayıcı ağ çağrısı yapmaz. Hazırlama her zaman `--keep=true` geçirir; dış yaşam döngüsünün sahibi OpenClaw'dur ve kiralamayı `crabbox stop` ile yok eder.

<Note>
  OpenClaw, Crabbox'ın kiralamaya yerel `sshKey` yolunu sağlayıcıya ait gizli değer çözümleyicisi aracılığıyla çözümler ve `crabbox inspect --json` tarafından döndürülen yetkili `sshHostKey` değerini sabitler. AWS kabulü ayrıca `providerMetadata.instanceProfileAttached` gerektirir. Bu kapalı inceleme sözleşmesi için Crabbox 0.38.1 veya daha yeni bir sürümünü yükleyin.
</Note>

### Statik SSH geliştirme profili

```json5
{
  cloudWorkers: {
    profiles: {
      development: {
        provider: "static-ssh",
        settings: {
          host: "worker.example.test",
          port: 22,
          user: "openclaw",
          hostKey: "ssh-ed25519 <base64-public-host-key>",
          keyRef: {
            source: "env",
            provider: "default",
            id: "OPENCLAW_WORKER_SSH_KEY",
          },
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `profiles`: boş olmayan, çevresindeki boşlukları kırpılmış kimliklere sahip adlandırılmış çalışan profilleri. Her profil, bir plugin tarafından kaydedilmiş bir sağlayıcı seçer.
- `provider`: boş olmayan çalışan sağlayıcısı kimliği. Örneklerde paketle gelen `crabbox` sağlayıcısı ve QA Lab `static-ssh` sağlayıcısı kullanılır.
- `install`: çalışan yükleme yöntemi. `"bundle"` (varsayılan), Gateway'in yüklü derlemesinin içerik karmalı bir paketini aktarır ve yayımlanmış, geliştirme aşamasındaki ve yayımlanmamış sürümleri destekler. `"npm"`, değiştirilmemiş paketlenmiş bir sürüm için isteğe bağlı bir optimizasyondur; genel npm kayıt defterinden `openclaw@<exact gateway version>` yükler ve hiçbir zaman `latest` yüklemez.
- Paketle gelen sağlayıcı plugin'leri yapılandırıldıklarında otomatik olarak seçilir; ancak açık devre dışı bırakmalar ve `plugins.allow` yine geçerlidir. Bir izin listesi yapılandırıldığında sağlayıcı kimliğini (örneğin `crabbox`) dahil edin. Harici sağlayıcı plugin'leri de yüklenmeli ve açıkça etkinleştirilmelidir.
- `settings`: sağlayıcıya ait sınırlı JSON. Seçilen plugin anahtarlarını tanımlar ve doğrular; gizli değer taşıyan değerler için [SecretRef nesnelerini](/tr/gateway/secrets) kullanın. Statik SSH sağlayıcısı `host`, `user`, `hostKey` ve `keyRef` gerektirir; `port` varsayılan olarak `22` değerini alır. `hostKey`, bilinen ana makineden veya başka bir güvenilir kanaldan elde edilmiş, seçenek öneki bulunmayan tek bir OpenSSH genel ana makine anahtarı satırı (`algorithm base64`) olmalıdır.
- `lifetime.idleTimeoutMinutes`: daha sonraki boşta kalma geri kazanım ilkesi için depolanan pozitif tam sayı dakika değeri.
- `lifetime.maxLifetimeMinutes`: daha sonraki yaşam döngüsü ilkesi için depolanan pozitif tam sayı dakika değeri.

WAL sıfırlaması güvenli SQLite'a sahip desteklenen bir Node çalışma zamanı (22.22.3+, 24.15+ veya 25.9+) çalışanda önceden yüklü olmalıdır. İsteğe bağlı `"npm"` yöntemi ayrıca `npm` ve genel npm kayıt defterine giden HTTPS erişimi gerektirir. Ağ bağlantılı araç zinciri kurulumu sağlayıcı ilkesidir; önyükleme, araç zincirlerini kendi başına yüklemek yerine uygulanabilir bir hata bildirir.

Bu temel, Gateway derlemesini yükleyip doğrular ve tünel başlatma/durdurma yaşam döngüsünü sağlar; ancak genel OpenClaw CLI'ını başlatmaz. Bağımsız çalışan giriş noktası ve döngüsü bir sonraki bulut çalışanı dönüm noktasında sunulur.

Her kalıcı ortam kaydı, doğrulanmış sağlayıcı ayarlarını, çözümlenmiş yükleme yöntemini ve yaşam süresi ilkesini oluşturulma zamanındaki profil anlık görüntüsünde saklar. Adlandırılmış bir profilin değiştirilmesi veya kaldırılması yeni oluşturmaları etkiler; sahibi olan plugin kullanılabilir durumda olduğu sürece mevcut kayıtlar bu anlık görüntüyle yaşam döngüsü uzlaştırmasına devam eder.

İlk bulut çalışanı sürümünde yaşam süresi değerleri yalnızca veridir; otomatik uygulama daha sonraki yaşam döngüsü çalışmalarıyla sunulur. Profil değişiklikleri Gateway'in yeniden başlatılmasını gerektirir.

<Warning>
  `static-ssh` sağlayıcısı, kaynak ağacı QA Lab geliştirme test düzeneğidir ve paketlenmiş dağıtımlara dahil edilmez. Paylaşılan ana makinesinde çalışan bir çalışan, ilgisiz ana makine verilerini okuyabilir; bu nedenle bu sağlayıcıyı üretim yalıtım sınırı olarak kullanmayın.
  İşletmecisi beklenen `hostKey` değerini sağlamalıdır; OpenClaw ilk bağlantıdan bir anahtar öğrenmez veya kabul etmez.
  Kiralamasını yok etmek yalnızca OpenClaw'un mantıksal kaydını serbest bırakır; ana makineyi durdurmaz veya temizlemez.
</Warning>

---

## Kancalar

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: true,
    allowedSessionKeyPrefixes: ["hook:", "hook:gmail:"],
    allowedAgentIds: ["hooks", "main"],
    presets: ["gmail"],
    transformsDir: "~/.openclaw/hooks/transforms",
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        agentId: "hooks",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "Gönderen: {{messages[0].from}}\nKonu: {{messages[0].subject}}\n{{messages[0].snippet}}",
        deliver: true,
        channel: "last",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

Kimlik doğrulaması: `Authorization: Bearer <token>` veya `x-openclaw-token: <token>`.
Sorgu dizesi kanca belirteçleri reddedilir.

Doğrulama ve güvenlik notları:

- `hooks.enabled=true`, boş olmayan bir `hooks.token` gerektirir.
- `hooks.token`, etkin Gateway paylaşılan gizli anahtar kimlik doğrulamasından (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` veya `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`) farklı olmalıdır; başlangıç sırasında yeniden kullanım algılanırsa ölümcül olmayan bir güvenlik uyarısı günlüğe kaydedilir.
- `openclaw security audit`, yalnızca denetim sırasında sağlanan Gateway parola kimlik doğrulaması (`--auth password --password <password>`) dâhil olmak üzere hook/Gateway kimlik doğrulamasının yeniden kullanımını kritik bir bulgu olarak işaretler. Kalıcı olarak saklanan ve yeniden kullanılan bir `hooks.token` değerini döndürmek için `openclaw doctor --fix` komutunu çalıştırın, ardından harici hook göndericilerini yeni hook token'ını kullanacak şekilde güncelleyin.
- `hooks.path`, `/` olamaz; `/hooks` gibi ayrılmış bir alt yol kullanın.
- `hooks.allowRequestSessionKey=true` ise `hooks.allowedSessionKeyPrefixes` değerini kısıtlayın (örneğin `["hook:"]`).
- Bir eşleme veya ön ayar, şablonlu bir `sessionKey` kullanıyorsa `hooks.allowedSessionKeyPrefixes` ve `hooks.allowRequestSessionKey=true` değerlerini ayarlayın. Statik eşleme anahtarları bu açık katılımı gerektirmez.

**Uç noktalar:**

- `POST /hooks/wake` → `{ text, mode?: "now"|"next-heartbeat" }`
- `POST /hooks/agent` → `{ message, name?, agentId?, sessionKey?, wakeMode?, deliver?, channel?, to?, model?, thinking?, timeoutSeconds? }`
  - İstek yükündeki `sessionKey`, yalnızca `hooks.allowRequestSessionKey=true` olduğunda kabul edilir (varsayılan: `false`).
- `POST /hooks/<name>` → `hooks.mappings` aracılığıyla çözümlenir
  - Şablonla işlenen eşleme `sessionKey` değerleri harici olarak sağlanmış kabul edilir ve ayrıca `hooks.allowRequestSessionKey=true` gerektirir.

<Accordion title="Eşleme ayrıntıları">

- `match.path`, `/hooks` sonrasındaki alt yolla eşleşir (ör. `/hooks/gmail` → `gmail`).
- `match.source`, genel yollar için bir yük alanıyla eşleşir.
- `{{messages[0].subject}}` gibi şablonlar yükten okur.
- `transform`, bir hook eylemi döndüren JS/TS modülüne işaret edebilir.
  - `transform.module` göreli bir yol olmalı ve `hooks.transformsDir` içinde kalmalıdır (mutlak yollar ve dizin geçişi reddedilir).
  - `hooks.transformsDir` değerini `~/.openclaw/hooks/transforms` altında tutun; çalışma alanı skill dizinleri reddedilir. `openclaw doctor` bu yolu geçersiz olarak bildirirse dönüştürme modülünü hooks dönüştürme dizinine taşıyın veya `hooks.transformsDir` değerini kaldırın.
- `agentId`, belirli bir agent'a yönlendirir; bilinmeyen kimlikler varsayılan agent'a geri döner.
- `allowedAgentIds`: `agentId` atlandığında varsayılan agent yolu dâhil olmak üzere etkin agent yönlendirmesini kısıtlar (`*` veya atlanmış = tümüne izin ver, `[]` = tümünü reddet).
- `defaultSessionKey`: açık bir `sessionKey` olmadan hook agent çalıştırmaları için isteğe bağlı sabit oturum anahtarı.
- `allowRequestSessionKey`: `/hooks/agent` çağıranlarının ve şablon güdümlü eşleme oturum anahtarlarının `sessionKey` değerini ayarlamasına izin verir (varsayılan: `false`).
- `allowedSessionKeyPrefixes`: açık `sessionKey` değerleri (istek + eşleme) için isteğe bağlı ön ek izin listesi; ör. `["hook:"]`. Herhangi bir eşleme veya ön ayar şablonlu bir `sessionKey` kullandığında zorunlu hâle gelir.
- `deliver: true`, son yanıtı bir kanala gönderir; `channel` varsayılan olarak `last` değerini kullanır.
- `model`, bu hook çalıştırması için LLM'yi geçersiz kılar (model kataloğu ayarlanmışsa izin verilmiş olmalıdır).

</Accordion>

### Gmail entegrasyonu

- Yerleşik Gmail ön ayarı `sessionKey: "hook:gmail:{{messages[0].id}}"` kullanır.
- Mesaj başına bu anahtar, araçları veya çalışma alanı erişimini değil, konuşma bağlamını yalıtır. `agentId` değerini ayarlayan özel bir eşleme olmadan ön ayar varsayılan agent'ı kullanır.
- Güvenilmeyen gelen kutuları için Gmail'i ayrılmış bir okuyucu agent'a yönlendirin ve bu agent'ı [agent başına sandbox ve araç politikası](/tr/tools/multi-agent-sandbox-tools) ile kısıtlayın. Okuyucunun ana agent'ı bilgilendirmesi gerekiyorsa devri [`tools.agentToAgent`](/tr/gateway/config-tools#toolsagenttoagent) ile kısıtlayın. Önerilen tehdit modeli ve model katmanı için [İstem enjeksiyonu](/tr/gateway/security#prompt-injection) bölümüne bakın.
- Mesaj başına bu yönlendirmeyi korursanız `hooks.allowRequestSessionKey: true` değerini ayarlayın ve `hooks.allowedSessionKeyPrefixes` değerini Gmail ad alanıyla eşleşecek şekilde kısıtlayın; örneğin `["hook:", "hook:gmail:"]`.
- `hooks.allowRequestSessionKey: false` gerekiyorsa şablonlu varsayılan yerine statik bir `sessionKey` ile ön ayarı geçersiz kılın.

```json5
{
  hooks: {
    gmail: {
      account: "openclaw@gmail.com",
      topic: "projects/<project-id>/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,
      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

- Yapılandırıldığında Gateway, açılış sırasında `gog gmail watch serve` öğesini otomatik olarak başlatır. Devre dışı bırakmak için `OPENCLAW_SKIP_GMAIL_WATCHER=1` değerini ayarlayın.
- Gateway ile birlikte ayrı bir `gog gmail watch serve` çalıştırmayın.

---

## Canvas plugin ana makinesi

```json5
{
  plugins: {
    entries: {
      canvas: {
        config: {
          host: {
            root: "~/.openclaw/workspace/canvas",
            liveReload: true,
            // etkin: false, // veya OPENCLAW_SKIP_CANVAS_HOST=1
          },
        },
      },
    },
  },
}
```

- Agent tarafından düzenlenebilir HTML/CSS/JS ve A2UI'yi Gateway portu altında HTTP üzerinden sunar:
  - `http://<gateway-host>:<gateway.port>/__openclaw__/canvas/`
  - `http://<gateway-host>:<gateway.port>/__openclaw__/a2ui/`
- Yalnızca yerel: `gateway.bind: "loopback"` değerini koruyun (varsayılan).
- Geri döngü olmayan bağlamalar: canvas yolları, diğer Gateway HTTP yüzeyleriyle aynı şekilde Gateway kimlik doğrulaması (token/parola/güvenilir proxy) gerektirir.
- Node WebView'ları genellikle kimlik doğrulama üst bilgileri göndermez; bir node eşleştirilip bağlandıktan sonra Gateway, canvas/A2UI erişimi için node kapsamlı yetenek URL'lerini duyurur.
- Yetenek URL'leri etkin node WS oturumuna bağlıdır ve hızla sona erer. IP tabanlı geri dönüş kullanılmaz.
- Sunulan HTML'ye canlı yeniden yükleme istemcisi ekler.
- Boş olduğunda başlangıç `index.html` öğesini otomatik oluşturur.
- A2UI'yi ayrıca `/__openclaw__/a2ui/` konumunda sunar.
- Değişiklikler Gateway'in yeniden başlatılmasını gerektirir.
- Büyük dizinler veya `EMFILE` hataları için canlı yeniden yüklemeyi devre dışı bırakın.

---

## Keşif

### mDNS (Bonjour)

```json5
{
  discovery: {
    mdns: {
      mode: "minimal", // minimal | full | off
    },
  },
}
```

- `minimal` (varsayılan): TXT kayıtlarından `cliPath` + `sshPort` değerlerini çıkarır.
- `full`: `cliPath` + `sshPort` değerlerini içerir; LAN çok noktaya yayın duyurusu için paketle gelen `bonjour` plugin'inin yine de etkinleştirilmesi gerekir.
- `off`: plugin etkinleştirme durumunu değiştirmeden LAN çok noktaya yayın duyurusunu engeller.
- Paketle gelen `bonjour` plugin'i macOS ana makinelerinde otomatik başlar; Linux, Windows ve konteyner tabanlı Gateway dağıtımlarında açık katılım gerektirir.
- Ana makine adı, geçerli bir DNS etiketi olduğunda varsayılan olarak sistem ana makine adını kullanır; aksi hâlde `openclaw` değerine geri döner. `OPENCLAW_MDNS_HOSTNAME` ile geçersiz kılın.
- `OPENCLAW_DISABLE_BONJOUR=1`, `discovery.mdns.mode` değerini geçersiz kılarak mDNS duyurusunu tamamen devre dışı bırakır.

### Geniş alan (DNS-SD)

```json5
{
  discovery: {
    wideArea: { enabled: true },
  },
}
```

`~/.openclaw/dns/` altında tek noktaya yayın DNS-SD bölgesi yazar. Ağlar arası keşif için bir DNS sunucusu (CoreDNS önerilir) + Tailscale bölünmüş DNS ile birlikte kullanın.

Kurulum: `openclaw dns setup --apply`.

---

## Ortam

### `env` (satır içi ortam değişkenleri)

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

- Satır içi ortam değişkenleri yalnızca işlem ortamında anahtar eksikse uygulanır.
- `.env` dosyaları: CWD `.env` + `~/.openclaw/.env` (hiçbiri mevcut değişkenleri geçersiz kılmaz).
- `shellEnv`: eksik beklenen anahtarları oturum açma kabuğu profilinizden içe aktarır.
- Tam öncelik sırası için [Ortam](/tr/help/environment) bölümüne bakın.

### Ortam değişkeni ikamesi

Herhangi bir yapılandırma dizesindeki ortam değişkenlerine `${VAR_NAME}` ile başvurun:

```json5
{
  gateway: {
    auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" },
  },
}
```

- Yalnızca büyük harfli adlar eşleştirilir: `[A-Z_][A-Z0-9_]*`.
- Eksik/boş değişkenler yapılandırma yüklenirken hata oluşturur.
- Değişmez bir `${VAR}` için `$${VAR}` ile kaçış uygulayın.
- `$include` ile çalışır.

---

## Gizli değerler

Gizli değer başvuruları eklemelidir: düz metin değerler çalışmaya devam eder.

### `SecretRef`

Tek bir nesne biçimi kullanın:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

Doğrulama:

- `provider` deseni: `^[a-z][a-z0-9_-]{0,63}$`
- `source: "env"` kimlik deseni: `^[A-Z][A-Z0-9_]{0,127}$`
- `source: "file"` kimliği: mutlak JSON işaretçisi (örneğin `"/providers/openai/apiKey"`)
- `source: "exec"` kimlik deseni: `^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$` (AWS tarzı `secret#json_key` seçicilerini destekler)
- `source: "exec"` kimlikleri, eğik çizgiyle ayrılmış yol segmentlerinde `.` veya `..` içermemelidir (örneğin `a/../b` reddedilir)

### Desteklenen kimlik bilgisi yüzeyi

- Standart matris: [SecretRef Kimlik Bilgisi Yüzeyi](/tr/reference/secretref-credential-surface)
- `secrets apply`, desteklenen `openclaw.json` kimlik bilgisi yollarını hedefler.
- `auth-profiles.json` başvuruları, çalışma zamanı çözümlemesine ve denetim kapsamına dâhildir.

### Gizli değer sağlayıcıları yapılandırması

```json5
{
  secrets: {
    providers: {
      default: { source: "env" }, // isteğe bağlı açık ortam sağlayıcısı
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",
        timeoutMs: 5000,
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        passEnv: ["PATH", "VAULT_ADDR"],
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

Notlar:

- `file` sağlayıcısı, `mode: "json"` ve `mode: "singleValue"` değerlerini destekler (singleValue modunda `id`, `"value"` olmalıdır).
- Windows ACL doğrulaması kullanılamadığında dosya ve exec sağlayıcı yolları güvenli biçimde başarısız olur. `allowInsecurePath: true` değerini yalnızca doğrulanamayan güvenilir yollar için ayarlayın.
- `exec` sağlayıcısı mutlak bir `command` yolu gerektirir ve stdin/stdout üzerinde protokol yüklerini kullanır.
- Varsayılan olarak sembolik bağlantı komut yolları reddedilir. Çözümlenen hedef yolu doğrularken sembolik bağlantı yollarına izin vermek için `allowSymlinkCommand: true` değerini ayarlayın.
- `trustedDirs` yapılandırılmışsa güvenilir dizin denetimi çözümlenen hedef yola uygulanır.
- `exec` alt işlem ortamı varsayılan olarak en az düzeydedir; gerekli değişkenleri `passEnv` ile açıkça geçirin.
- Gizli değer başvuruları etkinleştirme sırasında bellek içi bir anlık görüntüye çözümlenir; ardından istek yolları yalnızca bu anlık görüntüyü okur.
- Etkin yüzey filtrelemesi etkinleştirme sırasında uygulanır: etkin yüzeylerde çözümlenemeyen başvurular başlatma/yeniden yüklemeyi başarısız kılarken etkin olmayan yüzeyler tanılama bilgileriyle atlanır.

---

## Kimlik doğrulama depolaması

```json5
{
  auth: {
    profiles: {
      "anthropic:default": { provider: "anthropic", mode: "api_key" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai:personal": { provider: "openai", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:default", "anthropic:work"],
      openai: ["openai:personal"],
    },
  },
}
```

- Aracı başına profiller `<agentDir>/auth-profiles.json` konumunda saklanır.
- `auth-profiles.json`, statik kimlik bilgisi modları için değer düzeyinde referansları (`api_key` için `keyRef`, `token` için `tokenRef`) destekler.
- `{ "provider": { "apiKey": "..." } }` gibi eski düz `auth-profiles.json` eşlemeleri bir çalışma zamanı biçimi değildir; `openclaw doctor --fix` bunları bir `.legacy-flat.*.bak` yedeğiyle kurallı `provider:default` API anahtarı profillerine dönüştürür.
- OAuth modundaki profiller (`auth.profiles.<id>.mode = "oauth"`), SecretRef destekli kimlik doğrulama profili kimlik bilgilerini desteklemez.
- Statik çalışma zamanı kimlik bilgileri, bellekte çözümlenmiş anlık görüntülerden gelir; eski statik `auth.json` girdileri algılandığında temizlenir.
- `~/.openclaw/credentials/oauth.json` konumundan eski OAuth içe aktarımları.
- Bkz. [OAuth](/tr/concepts/oauth).
- Gizli bilgilerin çalışma zamanı davranışı ve `audit/configure/apply` araçları: [Gizli Bilgi Yönetimi](/tr/gateway/secrets).

---

## Denetim

```json5
{
  audit: {
    enabled: true,
    messages: "off", // off | direct | all
  },
}
```

Gateway, aracı çalıştırmaları ve araç eylemleri için **yalnızca meta veri**
içeren denetim olaylarını paylaşılan durum veritabanına kaydeder. Mesaj yaşam
döngüsü meta verileri, ayrıca etkinleştirilmesi gereken bir seçenektir. Kayıt
defteri kimliği, zamanlamayı, araç adlarını ve normalleştirilmiş sonuçları
saklar; ancak istemleri, mesaj gövdelerini, araç bağımsız değişkenlerini,
sonuçları veya ham hata metnini asla saklamaz. Mesaj satırları ham platform
hesabı, konuşma, mesaj ve hedef kimliklerini saklamaz. Çalıştırma/araç oturum
anahtarları ilişkilendirme için kullanılabilir kalır ve kendileri de platform
hesabı veya eş kimlikleri içerebilir. Kayıtların süresi 30 gün sonra dolar ve
kayıt defteri 100.000 satırla sınırlıdır. Bunları
[`openclaw audit`](/tr/cli/audit) veya
[`audit.activity.list`](/tr/gateway/protocol#audit-ledger-rpc) Gateway RPC'siyle
sorgulayın. Tam veri modeli, gizlilik anlamları ve kapsam sınırları için
[Denetim geçmişi](/tr/gateway/audit) bölümüne bakın.

- `enabled`: yeni denetim olaylarını kaydeder (varsayılan: `true`). Yalnızca bir olaydan sonra etkinleştirilen denetim izi olayı açıklayamayacağı için kayıt defteri varsayılan olarak açıktır. `false` ayarı, Gateway yeniden başlatıldıktan sonra yeni olay eklemelerini durdurur; mevcut kayıtlar süreleri dolana kadar okunabilir kalır. Tekrar açılması, kaydı o noktadan itibaren sürdürür; aradaki boşluk geriye dönük olarak doldurulmaz.
- `messages`: mesaj meta verisi kapsamı (varsayılan: `"off"`). `"direct"` yalnızca bilinen doğrudan konuşmaları kaydeder. `"all"` ayrıca grup, kanal ve bilinmeyen konuşma türlerini kaydeder. Her iki mod da içerik barındırmaz ve ilişkilendirmenin mümkün olduğu yerlerde ham tanımlayıcıları kuruluma özgü, anahtarlı takma adlarla değiştirir. Bunlar anonimleştirme değil, ilişkilendirme yardımcılarıdır; durum veritabanı türetme anahtarını saklarken RPC ve CLI dışa aktarımları saklamaz.

Çalışan Gateway, başlangıçta `audit.enabled` ve `audit.messages` değerlerini yakalar; ayarlardan birini değiştirdikten sonra yeniden başlatın. Mesaj kapsamı şu anda çekirdek yönlendirmeye ulaşan kabul edilmiş gelen mesajları ve paylaşılan kalıcı teslimata ulaşan her özgün mantıksal giden yanıt yükü için bir sonlandırma satırını içerir. Bu paylaşılan sınırları atlayan Plugin yerelindeki ve doğrudan gönderim yolları henüz kapsanmaz. Sınırlı arka plan yazıcısı en iyi çaba esasına göre çalışır; kayıpsız bir uyumluluk arşivi değildir.

---

## Günlük Kaydı

```json5
{
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty", // pretty | compact | json
    redactSensitive: "tools", // off | tools
    redactPatterns: ["\\bTOKEN\\b\\s*[=:]\\s*([\"']?)([^\\s\"']+)\\1"],
  },
}
```

- Varsayılan günlük dosyası: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`; adlandırılmış profiller `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log` kullanır.
- Kararlı bir yol için `logging.file` değerini ayarlayın.
- `--verbose` olduğunda `consoleLevel`, `debug` düzeyine yükselir.
- `maxFileBytes`: döndürmeden önce etkin günlük dosyasının bayt cinsinden azami boyutu (pozitif tam sayı; varsayılan: `104857600` = 100 MB). OpenClaw, etkin dosyanın yanında en fazla beş numaralandırılmış arşiv tutar.
- `redactSensitive` / `redactPatterns`: konsol çıktısı, dosya günlükleri, OTLP günlük kayıtları ve kalıcı oturum dökümü metni için en iyi çaba esaslı maskeleme. `redactSensitive: "off"` yalnızca bu genel günlük/döküm politikasını devre dışı bırakır; kullanıcı arayüzü/araç/tanılama güvenlik yüzeyleri gizli bilgileri yayımlamadan önce karartmaya devam eder.

---

## Tanılama

```json5
{
  diagnostics: {
    enabled: true,
    flags: ["telegram.*"],

    otel: {
      enabled: false,
      endpoint: "https://otel-collector.example.com:4318",
      tracesEndpoint: "https://traces.example.com/v1/traces",
      metricsEndpoint: "https://metrics.example.com/v1/metrics",
      logsEndpoint: "https://logs.example.com/v1/logs",
      protocol: "http/protobuf", // http/protobuf | grpc
      headers: { "x-tenant-id": "my-org" },
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: false,
      logsExporter: "otlp",
      sampleRate: 1.0,
      flushIntervalMs: 5000,
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },

    cacheTrace: {
      enabled: false,
      filePath: "~/.openclaw/logs/cache-trace.jsonl",
      includeMessages: true,
      includePrompt: true,
      includeSystem: true,
    },
  },
}
```

- `enabled`: araçlandırma çıktısının ana anahtarı (varsayılan: `true`).
- `flags`: hedefli günlük çıktısını etkinleştiren bayrak dizeleri dizisi (`"telegram.*"` veya `"*"` gibi joker karakterleri destekler).
- `otel.enabled`: OpenTelemetry dışa aktarma işlem hattını etkinleştirir (varsayılan: `false`). Tam yapılandırma, sinyal kataloğu ve gizlilik modeli için [OpenTelemetry dışa aktarma](/tr/gateway/opentelemetry) bölümüne bakın.
- `otel.endpoint`: OTel dışa aktarımı için toplayıcı URL'si.
- `otel.tracesEndpoint` / `otel.metricsEndpoint` / `otel.logsEndpoint`: isteğe bağlı, sinyale özgü OTLP uç noktaları. Ayarlandıklarında yalnızca ilgili sinyal için `otel.endpoint` değerini geçersiz kılarlar.
- `otel.protocol`: `"http/protobuf"` (varsayılan) veya `"grpc"`.
- `otel.headers`: OTel dışa aktarma istekleriyle gönderilen ek HTTP/gRPC meta veri başlıkları.
- `otel.serviceName`: kaynak öznitelikleri için hizmet adı.
- `otel.traces` / `otel.metrics` / `otel.logs`: iz, metrik veya günlük dışa aktarımını etkinleştirir.
- `otel.logsExporter`: günlük dışa aktarma hedefi: `"otlp"` (varsayılan), stdout'taki her satır için bir JSON nesnesi amacıyla `"stdout"` veya `"both"`.
- `otel.sampleRate`: `0`-`1` iz örnekleme oranı.
- `otel.flushIntervalMs`: ms cinsinden dönemsel telemetri boşaltma aralığı.
- `otel.captureContent`: OTEL span öznitelikleri için isteğe bağlı ham içerik yakalama. Varsayılan olarak kapalıdır. Boolean `true` sistem dışı mesaj/araç içeriğini yakalar; nesne biçimi `inputMessages`, `outputMessages`, `toolInputs`, `toolOutputs`, `systemPrompt` ve `toolDefinitions` seçeneklerini açıkça etkinleştirmenizi sağlar.
- `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`: `{gen_ai.operation.name} {gen_ai.request.model}` span adları, `CLIENT` span türü ve eski `gen_ai.system` yerine `gen_ai.provider.name` dâhil olmak üzere en son deneysel GenAI çıkarım span biçimi için ortam anahtarı. Varsayılan olarak span'ler uyumluluk amacıyla `openclaw.model.call` ve `gen_ai.system` değerlerini korur; GenAI metrikleri sınırlı anlamsal öznitelikler kullanır.
- `OPENCLAW_OTEL_PRELOADED=1`: küresel bir OpenTelemetry SDK'sını zaten kaydetmiş ana makineler için ortam anahtarı. Bu durumda OpenClaw, tanılama dinleyicilerini etkin tutarken Plugin tarafından yönetilen SDK başlatma/kapatma işlemlerini atlar.
- `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`, `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` ve `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`: eşleşen yapılandırma anahtarı ayarlanmamış olduğunda kullanılan sinyale özgü uç nokta ortam değişkenleri.
- `cacheTrace.enabled`: gömülü çalıştırmalar için önbellek izi anlık görüntülerini günlüğe kaydeder (varsayılan: `false`).
- `cacheTrace.filePath`: önbellek izi JSONL için çıktı yolu (varsayılan: `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl`).
- `cacheTrace.includeMessages` / `includePrompt` / `includeSystem`: önbellek izi çıktısına nelerin dâhil edileceğini denetler (tümünün varsayılanı: `true`).

---

## Güncelleme

```json5
{
  update: {
    channel: "stable", // stable | extended-stable | beta | dev
    checkOnStart: true,

    auto: {
      enabled: false,
    },
  },
}
```

- `channel`: sürüm kanalı — `"stable"`, `"extended-stable"`, `"beta"` veya `"dev"`. Extended-stable yalnızca pakete yöneliktir: ön plan komutları kurulumu yönetirken Gateway salt okunur güncelleme ipuçları yayımlayabilir.
- `checkOnStart`: Gateway başlatıldığında npm güncellemelerini denetler (varsayılan: `true`). Saklanan extended-stable seçimleri aynı salt okunur ipucunu ve 24 saatlik ipucu zamanlamasını kullanır.
- `auto.enabled`: stable ve beta paket kurulumları için arka planda otomatik güncellemeyi etkinleştirir (varsayılan: `false`). Extended-stable hiçbir zaman otomatik uygulanmaz.

---

## ACP

```json5
{
  acp: {
    enabled: true,
    dispatch: { enabled: true },
    backend: "acpx",
    fallbacks: ["acpx-secondary"],
    defaultAgent: "main",
    allowedAgents: ["main", "ops"],
    stream: {
      repeatSuppression: true,
      deliveryMode: "live", // live | final_only
    },
  },
}
```

- `enabled`: küresel ACP özellik geçidi (varsayılan: `true`; ACP yönlendirme ve oluşturma seçeneklerini gizlemek için `false` olarak ayarlayın).
- `dispatch.enabled`: ACP oturum turu yönlendirmesi için bağımsız geçit (varsayılan: `true`). Yürütmeyi engellerken ACP komutlarını kullanılabilir tutmak için `false` olarak ayarlayın.
- `backend`: varsayılan ACP çalışma zamanı arka uç kimliği (kayıtlı bir ACP çalışma zamanı Plugin'iyle eşleşmelidir).
  Önce arka uç Plugin'ini kurun ve `plugins.allow` ayarlanmışsa arka uç Plugin kimliğini (örneğin `acpx`) ekleyin; aksi takdirde ACP arka ucu yüklenmez.
- `fallbacks`: birincil arka uç herhangi bir çıktı üretmeden önce geçici görünen bir hatayla (kullanılamıyor, hız sınırına takılmış, kotası tükenmiş veya aşırı yüklenmiş) erken başarısız olduğunda denenen yedek ACP arka uç kimliklerinin sıralı listesi. Her girdi, kayıtlı bir ACP çalışma zamanı Plugin arka ucuyla eşleşmelidir.
- `defaultAgent`: oluşturma işlemleri açık bir hedef belirtmediğinde kullanılan yedek ACP hedef aracı kimliği.
- `allowedAgents`: ACP çalışma zamanı oturumlarına izin verilen aracı kimliklerinin izin listesi; boş olması ek kısıtlama olmadığı anlamına gelir.
- `stream.repeatSuppression`: tur başına yinelenen durum/araç satırlarını engeller (varsayılan: `true`).
- `stream.deliveryMode`: `"live"` artımlı olarak akış yapar; `"final_only"` tur sonlandırma olaylarına kadar arabelleğe alır.
- `stream.tagVisibility`: akış olayları için etiket adlarından Boolean görünürlük geçersiz kılmalarına uzanan kayıt.
- `runtime.installCommand`: bir ACP çalışma zamanı ortamı önyüklenirken çalıştırılacak isteğe bağlı kurulum komutu.

---

## Sihirbaz

CLI rehberli kurulum akışlarının davranışı ve meta verileri (`onboard`, `configure`, `doctor`):

```json5
{
  wizard: {
    accessMode: "full",
    appRecommendations: true,
    lastRunAt: "2026-01-01T00:00:00.000Z",
    lastRunVersion: "2026.1.4",
    lastRunCommit: "abc1234",
    lastRunCommand: "configure",
    lastRunMode: "local",
    securityAcknowledgedAt: "2026-01-01T00:00:00.000Z",
  },
}
```

- `wizard.accessMode`: yönlendirmeli ilk katılımın başında seçilen keşif onayı. `"full"` (önerilen), kurulumun yapay zekâ uygulamalarını, anahtarları ve yerel çalışma zamanlarını otomatik olarak aramasına izin verir; `"guarded"`, kurulumun çevreyi incelemeden önce bir kez sormasını ve bunun yerine elle yapılandırma sunmasını sağlar.

- `wizard.appRecommendations` varsayılan olarak `true` değerindedir. Yönlendirmeli veya klasik ilk katılım sırasında yüklü uygulama önerilerini devre dışı bırakmak ve Gateway `device.apps` erişimini engellemek için bunu `false` olarak ayarlayın. Node ana makinelerinin komutu duyurmadan önce ayrıca varsayılan olarak kapalı olan yüklü uygulama paylaşım bayraklarının etkinleştirilmesi gerekir.

---

## Kimlik

[Agent varsayılanları](/tr/gateway/config-agents#agent-defaults) altındaki `agents.entries` kimlik alanlarına bakın.

---

## Köprü (eski, kaldırıldı)

Güncel derlemeler artık TCP köprüsünü içermez. Node'lar Gateway WebSocket üzerinden bağlanır. `bridge.*` anahtarları artık yapılandırma şemasının parçası değildir (kaldırılana kadar doğrulama başarısız olur; `openclaw doctor --fix` bilinmeyen anahtarları kaldırabilir).

<Accordion title="Eski köprü yapılandırması (tarihsel başvuru)">

```json
{
  "bridge": {
    "enabled": true,
    "port": 18790,
    "bind": "tailnet",
    "tls": {
      "enabled": true,
      "autoGenerate": true
    }
  }
}
```

</Accordion>

---

## Cron

```json5
{
  cron: {
    enabled: true,
    webhook: "https://example.invalid/legacy", // saklanan notify:true işleri için kullanımdan kaldırılmış geri dönüş
    webhookToken: "replace-with-dedicated-token", // giden webhook kimlik doğrulaması için isteğe bağlı bearer token
    sessionRetention: "24h", // süre dizesi veya false
  },
}
```

- `sessionRetention`: SQLite oturum satırlarını budamadan önce tamamlanmış yalıtılmış Cron çalıştırma oturumlarının ne kadar süre tutulacağı. Arşivlenmiş, silinmiş Cron dökümlerinin temizlenmesini de denetler. Varsayılan: `24h`; devre dışı bırakmak için `false` olarak ayarlayın.
- Çalıştırma geçmişi, her iş için en yeni 2000 terminal satırını otomatik olarak tutar. Kaybolan satırlar 24 saatlik temizleme aralığını korur.
- `webhookToken`: Cron webhook POST teslimatı (`delivery.mode = "webhook"`) için kullanılan bearer token; atlanırsa kimlik doğrulama üstbilgisi gönderilmez.
- `webhook`: hâlâ `notify: true` içeren saklanmış işleri taşımak için `openclaw doctor --fix` tarafından kullanılan, kullanımdan kaldırılmış eski geri dönüş webhook URL'si (http/https); çalışma zamanı teslimatı iş başına `delivery.mode="webhook"` ile `delivery.to` değerlerini veya duyuru teslimatı korunurken `delivery.completionDestination` değerini kullanır.

### `cron.failureAlert`

```json5
{
  cron: {
    failureAlert: {
      enabled: false,
      after: 3,
      cooldownMs: 3600000,
      includeSkipped: false,
      mode: "announce",
      accountId: "main",
    },
  },
}
```

- `enabled`: Cron işleri için başarısızlık uyarılarını etkinleştirir (varsayılan: `false`).
- `after`: bir uyarı tetiklenmeden önceki ardışık başarısızlık sayısı (pozitif tam sayı, en az: `1`).
- `cooldownMs`: aynı iş için yinelenen uyarılar arasındaki en az milisaniye (negatif olmayan tam sayı).
- `includeSkipped`: ardışık atlanan çalıştırmaları uyarı eşiğine dahil eder (varsayılan: `false`). Atlanan çalıştırmalar ayrı izlenir ve yürütme hatası geri çekilmesini etkilemez.
- `mode`: teslimat modu - `"announce"` bir kanal iletisi üzerinden gönderir; `"webhook"` yapılandırılmış webhook'a gönderir.
- `accountId`: uyarı teslimatının kapsamını belirleyen isteğe bağlı hesap veya kanal kimliği.

### `cron.failureDestination`

```json5
{
  cron: {
    failureDestination: {
      mode: "announce",
      channel: "last",
      to: "channel:C1234567890",
      accountId: "main",
    },
  },
}
```

- Tüm işlerdeki Cron başarısızlık bildirimleri için varsayılan hedef.
- `mode`: `"announce"` veya `"webhook"`; yeterli hedef verisi mevcutsa varsayılan olarak `"announce"` kullanılır.
- `channel`: duyuru teslimatı için kanal geçersiz kılması. `"last"`, bilinen son teslimat kanalını yeniden kullanır.
- `to`: açık duyuru hedefi veya webhook URL'si. Webhook modu için zorunludur.
- `accountId`: teslimat için isteğe bağlı hesap geçersiz kılması.
- İş başına `delivery.failureDestination`, bu genel varsayılanı geçersiz kılar.
- Ne genel ne de iş başına başarısızlık hedefi ayarlanmışsa, zaten `announce` üzerinden teslimat yapan işler başarısızlık durumunda bu birincil duyuru hedefine geri döner.
- `delivery.failureDestination`, işin birincil `delivery.mode` değeri `"webhook"` olmadığı sürece yalnızca `sessionTarget="isolated"` işleri için desteklenir.

[Cron İşleri](/tr/automation/cron-jobs) bölümüne bakın. Yalıtılmış Cron yürütmeleri [arka plan görevleri](/tr/automation/tasks) olarak izlenir.

## Medya modeli şablon değişkenleri

`tools.media.models[].args` içinde genişletilen şablon yer tutucuları:

| Değişken                    | Açıklama                                          |
| --------------------------- | ------------------------------------------------- |
| `{{Body}}`                  | Gelen iletinin tam gövdesi                        |
| `{{RawBody}}`               | Ham gövde (geçmiş/gönderen sarmalayıcıları olmadan) |
| `{{BodyStripped}}`          | Grup bahsetmeleri kaldırılmış gövde               |
| `{{From}}`                  | Gönderen tanımlayıcısı                            |
| `{{To}}`                    | Hedef tanımlayıcısı                               |
| `{{MessageSid}}`            | Kanal iletisi kimliği                             |
| `{{SessionId}}`             | Geçerli oturum UUID'si                            |
| `{{IsNewSession}}`          | Yeni oturum oluşturulduğunda `"true"`   |
| `{{AttachmentUrl}}`         | Geçerli ek URL'si veya sağlayıcı başvurusu         |
| `{{AttachmentPath}}`        | Geçerli ekin yerel yolu                            |
| `{{AttachmentContentType}}` | Geçerli ekin MIME içerik türü                      |
| `{{AttachmentDir}}`         | `AttachmentPath` öğesini içeren dizin            |
| `{{AttachmentIndex}}`       | Sıfır tabanlı kaynak olgusu dizini                 |
| `{{Transcript}}`            | Ses dökümü                                         |
| `{{Prompt}}`                | CLI girdileri için çözümlenmiş medya istemi        |
| `{{MaxChars}}`              | CLI girdileri için çözümlenmiş en fazla çıktı karakteri |
| `{{ChatType}}`              | `"direct"` veya `"group"`         |
| `{{GroupSubject}}`          | Grup konusu (mümkün olan en iyi şekilde)           |
| `{{GroupMembers}}`          | Grup üyeleri önizlemesi (mümkün olan en iyi şekilde) |
| `{{SenderName}}`            | Gönderenin görünen adı (mümkün olan en iyi şekilde) |
| `{{SenderE164}}`            | Gönderenin telefon numarası (mümkün olan en iyi şekilde) |
| `{{Provider}}`              | Sağlayıcı ipucu (whatsapp, telegram, discord vb.)  |

Eski `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` ve `{{MediaDir}}`
adları, Plugin SDK uyumluluk aralığı boyunca kullanılabilir olmaya devam eder ancak
kullanımdan kaldırılmıştır. Yeni yapılandırmalar `Attachment*` değişkenlerini kullanmalıdır.

---

## Yapılandırma eklemeleri (`$include`)

Yapılandırmayı birden çok dosyaya bölün:

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: {
    $include: ["./clients/mueller.json5", "./clients/schmidt.json5"],
  },
}
```

**Birleştirme davranışı:**

- Tek dosya: kapsayan nesnenin yerini alır.
- Dosya dizisi: sırayla derinlemesine birleştirilir (sonrakiler öncekileri geçersiz kılar).
- Eş düzey anahtarlar: eklemelerden sonra birleştirilir (eklenen değerleri geçersiz kılar).
- İç içe eklemeler: en fazla 10 düzey derinlik.
- Yollar: ekleyen dosyaya göre çözümlenir ancak en üst düzey yapılandırma dizininin (`openclaw.json` öğesinin `dirname` değeri) içinde kalmalıdır. Mutlak/`../` biçimlerine yalnızca hâlâ bu sınırın içinde çözümleniyorlarsa izin verilir. Yapılandırma dizini dışındaki ek köklere izin vermek için `OPENCLAW_INCLUDE_ROOTS` (mutlak yollar) değerini ayarlayın.
- Sınırlar: yollar null baytları içermemeli ve çözümlemeden önce ve sonra kesinlikle 4096 karakterden kısa olmalıdır; eklenen her dosya en fazla 2 MB olabilir.
- Yalnızca tek dosyalı bir ekleme tarafından desteklenen bir üst düzey bölümü değiştiren OpenClaw'a ait yazma işlemleri, doğrudan bu eklenen dosyaya yazar. Örneğin `plugins install`, `plugins.json5` içindeki `plugins: { $include: "./plugins.json5" }` değerini günceller ve `openclaw.json` değerini olduğu gibi bırakır.
- Kök eklemeler, ekleme dizileri ve eş düzey geçersiz kılmalara sahip eklemeler, OpenClaw'a ait yazma işlemleri için salt okunurdur; bu yazma işlemleri yapılandırmayı düzleştirmek yerine güvenli biçimde başarısız olur.
- Hatalar: eksik dosyalar, ayrıştırma hataları, döngüsel eklemeler, geçersiz yol biçimi ve aşırı uzunluk için açık iletiler.

---

## İlgili

- [Yapılandırma](/tr/gateway/configuration)
- [Yapılandırma örnekleri](/tr/gateway/configuration-examples)
- [Doctor](/tr/gateway/doctor)
