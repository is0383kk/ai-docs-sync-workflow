---
read_when:
    - Ajan varsayılanlarını ayarlama (modeller, düşünme, çalışma alanı, Heartbeat, medya, Skills)
    - Çoklu ajan yönlendirmesini ve bağlamalarını yapılandırma
    - Oturum, mesaj teslimi ve konuşma modu davranışını ayarlama
summary: Ajan varsayılanları, çoklu ajan yönlendirmesi, oturum, mesajlar ve konuşma yapılandırması
title: Yapılandırma — aracılar
x-i18n:
    generated_at: "2026-07-26T23:56:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7a161d65b02e3333c15a2d998421419ee37d36be4d02ebb3a86e66282df06adb
    source_path: gateway/config-agents.md
    workflow: 16
---

`agents.*`, `multiAgent.*`, `session.*`,
`messages.*` ve `talk.*` altındaki ajan kapsamlı yapılandırma anahtarları.
Kanallar, araçlar, Gateway çalışma zamanı ve diğer üst düzey anahtarlar için
[Yapılandırma referansı](/tr/gateway/configuration-reference) bölümüne bakın.

## Ajan varsayılanları

### `agents.defaults.workspace`

Varsayılan: ayarlandığında `OPENCLAW_WORKSPACE_DIR`; aksi takdirde `~/.openclaw/workspace` (veya `OPENCLAW_PROFILE` varsayılan olmayan bir profile ayarlandığında `~/.openclaw/workspace-<profile>`).

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

Açıkça belirtilen bir `agents.defaults.workspace` değeri,
`OPENCLAW_WORKSPACE_DIR` değerine göre önceliklidir. Bu yolu yapılandırmaya yazmak
istemediğinizde varsayılan ajanları bağlanmış bir çalışma alanına yönlendirmek için ortam değişkenini kullanın.

### `agents.defaults.repoRoot`

Sistem isteminin Runtime satırında gösterilen isteğe bağlı depo kökü. Ayarlanmazsa OpenClaw, çalışma alanından yukarı doğru ilerleyerek bunu otomatik algılar.

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

`agents.entries.*.skills` ayarını belirtmeyen ajanlar için isteğe bağlı varsayılan skill izin listesi.

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" }, // github ve weather değerlerini devralır
      { id: "docs", skills: ["docs-search"] }, // varsayılanların yerini alır
      { id: "locked-down", skills: [] }, // skill yok
    ],
  },
}
```

- Varsayılan olarak sınırsız Skills için `agents.defaults.skills` değerini belirtmeyin.
- Varsayılanları devralmak için `agents.entries.*.skills` değerini belirtmeyin.
- Skills olmaması için `agents.entries.*.skills: []` değerini ayarlayın.
- Boş olmayan bir `agents.entries.*.skills` listesi, ilgili ajan için nihai kümedir;
  varsayılanlarla birleştirilmez.

### `agents.defaults.skipBootstrap`

Çalışma alanı önyükleme dosyalarının (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `BOOTSTRAP.md`) otomatik oluşturulmasını devre dışı bırakır.

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.skipOptionalBootstrapFiles`

Gerekli önyükleme dosyalarını (`AGENTS.md`, `TOOLS.md`, `BOOTSTRAP.md`) yazmaya devam ederken seçili isteğe bağlı çalışma alanı dosyalarının oluşturulmasını atlar. Geçerli değerler: `SOUL.md`, `USER.md` ve `IDENTITY.md` (`HEARTBEAT.md` kabul edilir ancak Heartbeat bağlamı Cron izleyicisinin geçici alanına taşındığından herhangi bir etkisi yoktur).

```json5
{
  agents: {
    defaults: {
      skipOptionalBootstrapFiles: ["SOUL.md", "USER.md"],
    },
  },
}
```

### `agents.defaults.contextInjection`

Çalışma alanı önyükleme dosyalarının sistem istemine ne zaman ekleneceğini denetler. Varsayılan: `"always"`.

- `"continuation-skip"`: güvenli devam turlarında (tamamlanmış bir asistan yanıtından sonra) çalışma alanı önyüklemesinin yeniden eklenmesi atlanarak istem boyutu küçültülür. Heartbeat çalıştırmaları ve Compaction sonrası yeniden denemeler bağlamı yine yeniden oluşturur.
- `"never"`: her turda çalışma alanı önyüklemesini ve bağlam dosyası eklemeyi devre dışı bırakır. Bunu yalnızca istem yaşam döngüsünün tamamına sahip olan ajanlar (özel bağlam motorları, kendi bağlamını oluşturan yerel çalışma zamanları veya önyüklemesiz özel iş akışları) için kullanın. Heartbeat ve Compaction kurtarma turlarında da ekleme atlanır.

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

Ajan başına geçersiz kılma: `agents.entries.*.contextInjection`. Belirtilmeyen değerler
`agents.defaults.contextInjection` değerini devralır.

### `agents.defaults.bootstrapMaxChars`

Kesilmeden önce çalışma alanı önyükleme dosyası başına en fazla karakter sayısı. Varsayılan: `20000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

Ajan başına geçersiz kılma: `agents.entries.*.bootstrapMaxChars`. Belirtilmeyen değerler
`agents.defaults.bootstrapMaxChars` değerini devralır.

### `agents.defaults.bootstrapTotalMaxChars`

Tüm çalışma alanı önyükleme dosyalarından eklenen toplam karakterlerin üst sınırı. Varsayılan: `60000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 60000 } },
}
```

Ajan başına geçersiz kılma: `agents.entries.*.bootstrapTotalMaxChars`. Belirtilmeyen değerler
`agents.defaults.bootstrapTotalMaxChars` değerini devralır.

### Ajan başına önyükleme profili geçersiz kılmaları

Bir ajanın paylaşılan varsayılanlardan farklı istem ekleme davranışına ihtiyaç
duyduğu durumlarda ajan başına önyükleme profili geçersiz kılmalarını kullanın. Belirtilmeyen alanlar
`agents.defaults` değerinden devralınır.

```json5
{
  agents: {
    defaults: {
      contextInjection: "continuation-skip",
      bootstrapMaxChars: 20000,
      bootstrapTotalMaxChars: 60000,
    },
    list: [
      {
        id: "strict-worker",
        contextInjection: "always",
        bootstrapMaxChars: 50000,
        bootstrapTotalMaxChars: 300000,
      },
    ],
  },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

Önyükleme bağlamı kesildiğinde ajanın görebildiği sistem istemi bildirimini denetler.
Varsayılan: `"always"`.

- `"off"`: kesilme bildirimi metnini sistem istemine hiçbir zaman eklemez.
- `"once"`: her benzersiz kesilme imzası için bir kez kısa bir bildirim ekler.
- `"always"`: kesilme olduğunda her çalıştırmada kısa bir bildirim ekler (önerilir).

Ayrıntılı ham/eklenen sayımlar ve yapılandırma ayarlama alanları, bağlam/durum
raporları ve günlükler gibi tanılamalarda kalır; rutin WebChat kullanıcı/çalışma zamanı bağlamına yalnızca
kısa kurtarma bildirimi eklenir.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "always" } }, // off | once | always
}
```

### Bağlam bütçesi sahiplik haritası

OpenClaw birden fazla yüksek hacimli istem/bağlam bütçesine sahiptir ve bunlar,
tümünün tek bir genel ayar üzerinden akması yerine kasıtlı olarak alt sistemlere
ayrılmıştır.

| Bütçe                                                         | Kapsam                                                                                                                                                          |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agents.defaults.bootstrapMaxChars` / `bootstrapTotalMaxChars` | Normal çalışma alanı önyükleme eklemesi                                                                                                                            |
| `agents.defaults.startupContext.*`                             | Son günlük `memory/*.md` dosyaları dahil, tek seferlik sıfırlama/başlatma model çalıştırması ön bölümü. Yalın sohbet `/new` ve `/reset` komutları model çağrılmadan onaylanır |
| `skills.limits.*`                                              | Sistem istemine eklenen kompakt Skills listesi                                                                                                         |
| `agents.defaults.contextLimits.*`                              | Sınırlandırılmış çalışma zamanı alıntıları ve eklenen çalışma zamanı sahipliğindeki bloklar                                                                                                      |
| `memory.qmd.limits.*`                                          | Dizinlenmiş bellek arama parçacığı ve ekleme boyutlandırması                                                                                                              |

Eşleşen ajan başına geçersiz kılmalar:

- `agents.entries.*.skillsLimits.maxSkillsPromptChars`
- `agents.entries.*.contextInjection`
- `agents.entries.*.bootstrapMaxChars`
- `agents.entries.*.bootstrapTotalMaxChars`
- `agents.entries.*.contextLimits.*`

#### `agents.defaults.startupContext`

Sıfırlama/başlatma model çalıştırmalarında ilk tura eklenen başlangıç ön bölümünü denetler.
Yalın sohbet `/new` ve `/reset` komutları modeli çağırmadan sıfırlamayı onayladığından
bu ön bölümü yüklemez.

```json5
{
  agents: {
    defaults: {
      startupContext: {
        enabled: true,
        applyOn: ["new", "reset"],
        dailyMemoryDays: 2,
        maxFileBytes: 16384,
        maxFileChars: 1200,
        maxTotalChars: 2800,
      },
    },
  },
}
```

#### `agents.defaults.contextLimits`

Sınırlandırılmış çalışma zamanı bağlam yüzeyleri için paylaşılan varsayılanlar.

```json5
{
  agents: {
    defaults: {
      contextLimits: {
        memoryGetMaxChars: 12000,
        postCompactionMaxChars: 1800,
      },
    },
  },
}
```

- `memoryGetMaxChars`: kesilme meta verileri ve devam bildirimi eklenmeden önce varsayılan `memory_get` alıntı sınırı.
- `memory_get`, `lines` değerini içermediğinde OpenClaw yerleşik 120 satırlık bir pencere kullanır ve
  ardından `memoryGetMaxChars` değerini uygular.
- Canlı araç sonuçları model bağlamına göre otomatik bir sınır kullanır: 100K token'ın altında `16000` karakter, 100K+ token'da `32000` karakter ve 200K+ token'da `64000` karakter.
- `postCompactionMaxChars`: Compaction sonrası yenileme eklemesi sırasında kullanılan AGENTS.md alıntı sınırı.

#### `agents.entries.*.contextLimits`

Paylaşılan `contextLimits` ayarları için ajan başına geçersiz kılma. Belirtilmeyen alanlar
`agents.defaults.contextLimits` değerinden devralınır.

```json5
{
  agents: {
    defaults: {
      contextLimits: { memoryGetMaxChars: 12000 },
    },
    list: [
      {
        id: "tiny-local",
        contextLimits: {
          memoryGetMaxChars: 6000,
        },
      },
    ],
  },
}
```

#### `skills.limits.maxSkillsPromptChars`

Sistem istemine eklenen kompakt Skills listesinin genel üst sınırı. Bu,
`SKILL.md` dosyalarının gerektiğinde okunmasını etkilemez.

```json5
{
  skills: { limits: { maxSkillsPromptChars: 18000 } },
}
```

#### `agents.entries.*.skillsLimits.maxSkillsPromptChars`

Skills istem bütçesi için ajan başına geçersiz kılma.

```json5
{
  agents: {
    list: [{ id: "tiny-local", skillsLimits: { maxSkillsPromptChars: 6000 } }],
  },
}
```

### `agents.defaults.imageMaxDimensionPx`

Sağlayıcı çağrılarından önce transkript/araç görüntü bloklarındaki görüntünün en uzun kenarı için en fazla piksel boyutu.
Varsayılan: `1200`.

Daha düşük değerler, ekran görüntüsü yoğun çalıştırmalarda genellikle görsel token kullanımını ve istek yükü boyutunu azaltır.
Daha yüksek değerler daha fazla görsel ayrıntıyı korur.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.imageQuality`

Dosya yollarından, URL'lerden ve medya referanslarından yüklenen görüntüler için görüntü aracı sıkıştırma/ayrıntı tercihi.
Varsayılan: `auto`.

OpenClaw, yeniden boyutlandırma kademelerini seçilen görüntü modeline uyarlar. Örneğin Claude Opus 4.8, OpenAI GPT-5.6 Sol, Qwen VL ve barındırılan Llama 4 görüntü modelleri, eski/varsayılan yüksek ayrıntılı görüntü yollarından daha büyük görüntüler kullanabilir; çok görüntülü turlar ise token ve gecikme maliyetini denetlemek için `auto` modunda daha agresif biçimde sıkıştırılır.

Değerler:

- `auto`: model sınırlarına ve görüntü sayısına uyarlanır.
- `efficient`: daha düşük token ve bayt kullanımı için daha küçük görüntüler tercih edilir.
- `balanced`: standart dengeli kademe kullanılır.
- `high`: ekran görüntüleri, diyagramlar ve belge görüntüleri için daha fazla ayrıntı korunur.

```json5
{
  agents: { defaults: { imageQuality: "auto" } },
}
```

### `agents.defaults.userTimezone`

Sistem istemi bağlamının saat dilimi (mesaj zaman damgalarının değil). Ana makinenin saat dilimine geri döner.

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

Sistem istemindeki saat biçimi. Varsayılan: `auto` (işletim sistemi tercihi).

```json5
{
  agents: { defaults: { timeFormat: "auto" } }, // auto | 12 | 24
}
```

### `agents.defaults.model`

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
      utilityModel: "openai/gpt-5.4-mini",
      imageModel: {
        primary: "openrouter/qwen/qwen-2.5-vl-72b-instruct:free",
        fallbacks: ["openrouter/google/gemini-2.0-flash-vision:free"],
      },
      mediaModels: {
        image: {
          primary: "openai/gpt-image-2",
          fallbacks: ["google/gemini-3.1-flash-image"],
        },
        video: {
          primary: "qwen/wan2.6-t2v",
          fallbacks: ["qwen/wan2.6-i2v"],
        },
      },
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      params: { cacheRetention: "long" }, // genel varsayılan sağlayıcı parametreleri
      pdfMaxMb: 10,
      pdfMaxPages: 20,
      thinkingDefault: "low",
      verboseDefault: "off",
      toolProgressDetail: "explain",
      reasoningDefault: "off",
      elevatedDefault: "on",
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      contextTokens: 200000,
      maxConcurrent: 4,
    },
  },
}
```

- `model`: bir dizeyi (`"provider/model"`) veya bir nesneyi (`{ primary, fallbacks }`) kabul eder.
  - Dize biçimi yalnızca birincil modeli ayarlar.
  - Nesne biçimi, birincil modeli ve sıralı yük devretme modellerini ayarlar.
- `utilityModel`: kısa dahili görevler için isteğe bağlı `provider/model` başvurusu veya diğer adı. Şu anda oluşturulan Control UI oturum başlıklarını, Telegram DM konu başlıklarını, Discord otomatik ileti dizisi başlıklarını ve [ilerleme taslağı anlatımını](/tr/concepts/progress-drafts#narrated-status) destekler. Ayarlanmadığında OpenClaw, varsa birincil sağlayıcının bildirdiği küçük model varsayılanını türetir (OpenAI → `gpt-5.6-luna`, Anthropic → `claude-haiku-4-5`); aksi hâlde başlık görevleri aracının birincil modelini kullanır ve anlatım kapalı kalır. Ayrı bir yardımcı model oluşturulan başlığı hazırlayamaz veya tamamlayamazsa OpenClaw, bu başlığı birincil modelle bir kez daha dener. Pano başlıklarında otomatik yardımcı model türetme ve normal geri dönüş, etkin oturum sağlayıcısını ve kimlik doğrulama profilini kullanır; açıkça belirtilen bir yardımcı model ise yapılandırılmış sağlayıcısını/kimlik doğrulamasını korur. Alternatif yardımcı rotayı atlamak için `utilityModel: ""` değerini ayarlayın; pano başlığı oluşturma yine doğrudan normal oturum modeliyle devam eder. `agents.entries.*.utilityModel` varsayılanı geçersiz kılar, işleme özgü bir model geçersiz kılması ise her ikisine de üstün gelir. Yardımcı görevler ayrı model çağrıları yapar ve göreve özgü içeriği seçilen model sağlayıcısına gönderir. Pano başlığı oluşturma, komut olmayan ilk mesajın en fazla ilk 1.000 karakterini gönderir; anlatım ise gelen isteği ve kısa, hassas bilgileri çıkarılmış araç özetlerini gönderir. Maliyet ve veri işleme gereksinimlerinize uygun bir sağlayıcı seçin.
- `imageModel`: bir dizeyi (`"provider/model"`) veya bir nesneyi (`{ primary, fallbacks }`) kabul eder.
  - Etkin model görüntüleri kabul edemediğinde, `image` araç yolu tarafından görüntü modeli yapılandırması olarak kullanılır. Yerel görüntü destekli modeller bunun yerine yüklenen görüntü baytlarını doğrudan alır.
  - Seçilen/varsayılan model görüntü girdisini kabul edemediğinde geri dönüş yönlendirmesi olarak da kullanılır.
  - Açık `provider/model` başvurularını tercih edin. Yalın kimlikler uyumluluk amacıyla kabul edilir; yalın bir kimlik `models.providers.*.models` içindeki yapılandırılmış, görüntü destekli bir girdiyle benzersiz şekilde eşleşirse OpenClaw bunu ilgili sağlayıcıyla niteler. Birden fazla yapılandırılmış eşleşme varsa açık bir sağlayıcı öneki gerekir.
- `mediaModels.image`: bir dizeyi (`"provider/model"`) veya bir nesneyi (`{ primary, fallbacks }`) kabul eder.
  - Paylaşılan görüntü oluşturma yeteneği ve görüntü oluşturan gelecekteki araç/Plugin yüzeyleri tarafından kullanılır.
  - Tipik değerler: yerel Gemini görüntü oluşturma için `google/gemini-3.1-flash-image`, fal için `fal/fal-ai/flux/dev`, OpenAI Images için `openai/gpt-image-2` veya şeffaf arka planlı OpenAI PNG/WebP çıktısı için `openai/gpt-image-1.5`.
  - Bir sağlayıcı/modeli doğrudan seçerseniz eşleşen sağlayıcı kimlik doğrulamasını da yapılandırın (örneğin `google/*` için `GEMINI_API_KEY` veya `GOOGLE_API_KEY`, `openai/gpt-image-2` / `openai/gpt-image-1.5` için `OPENAI_API_KEY` veya OpenAI Codex OAuth, `fal/*` için `FAL_KEY`).
  - Atlanırsa `image_generate` yine de kimlik doğrulama destekli bir sağlayıcı varsayılanını çıkarabilir. Önce geçerli varsayılan sağlayıcıyı, ardından kalan kayıtlı görüntü oluşturma sağlayıcılarını sağlayıcı kimliği sırasına göre dener.
- `mediaModels.music`: bir dizeyi (`"provider/model"`) veya bir nesneyi (`{ primary, fallbacks }`) kabul eder.
  - Paylaşılan müzik oluşturma yeteneği ve yerleşik `music_generate` aracı tarafından kullanılır.
  - Tipik değerler: `google/lyria-3-clip-preview`, `google/lyria-3-pro-preview` veya `minimax/music-2.6`.
  - Atlanırsa `music_generate` yine de kimlik doğrulama destekli bir sağlayıcı varsayılanını çıkarabilir. Önce geçerli varsayılan sağlayıcıyı, ardından kalan kayıtlı müzik oluşturma sağlayıcılarını sağlayıcı kimliği sırasına göre dener.
  - Bir sağlayıcı/modeli doğrudan seçerseniz eşleşen sağlayıcı kimlik doğrulamasını/API anahtarını da yapılandırın.
- `mediaModels.video`: bir dizeyi (`"provider/model"`) veya bir nesneyi (`{ primary, fallbacks }`) kabul eder.
  - Paylaşılan video oluşturma yeteneği ve yerleşik `video_generate` aracı tarafından kullanılır.
  - Tipik değerler: `qwen/wan2.6-t2v`, `qwen/wan2.6-i2v`, `qwen/wan2.6-r2v`, `qwen/wan2.6-r2v-flash` veya `qwen/wan2.7-r2v`.
  - Atlanırsa `video_generate` yine de kimlik doğrulama destekli bir sağlayıcı varsayılanını çıkarabilir. Önce geçerli varsayılan sağlayıcıyı, ardından kalan kayıtlı video oluşturma sağlayıcılarını sağlayıcı kimliği sırasına göre dener.
  - Bir sağlayıcı/modeli doğrudan seçerseniz eşleşen sağlayıcı kimlik doğrulamasını/API anahtarını da yapılandırın.
  - Resmî Qwen video oluşturma Plugin'i en fazla 1 çıkış videosunu, 1 giriş görüntüsünü, 4 giriş videosunu, 10 saniyelik süreyi ve sağlayıcı düzeyindeki `size`, `aspectRatio`, `resolution`, `audio` ve `watermark` seçeneklerini destekler.
- `pdfModel`: bir dizeyi (`"provider/model"`) veya bir nesneyi (`{ primary, fallbacks }`) kabul eder.
  - Model yönlendirmesi için `pdf` aracı tarafından kullanılır.
  - Atlanırsa PDF aracı önce `imageModel` değerine, ardından çözümlenen oturum/varsayılan modele geri döner.
- `pdfMaxMb`: çağrı sırasında `maxBytesMb` iletilmediğinde `pdf` aracı için varsayılan PDF boyutu sınırı.
- `pdfMaxPages`: `pdf` aracındaki ayıklama geri dönüş modunun dikkate aldığı varsayılan azami sayfa sayısı.
- `verboseDefault`: aracılar için varsayılan ayrıntı düzeyi. Değerler: `"off"`, `"on"`, `"full"`. Varsayılan: `"off"`.
- `toolProgressDetail`: `/verbose` araç özetleri ve ilerleme taslağı araç satırları için ayrıntı modu. Değerler: `"explain"` (varsayılan, kısa ve insan tarafından okunabilir etiketler) veya `"raw"` (varsa ham komutu/ayrıntıyı ekler). Aracı başına `agents.entries.*.toolProgressDetail` bu varsayılanı geçersiz kılar.
- `reasoningDefault`: aracılar için varsayılan akıl yürütme görünürlüğü. Değerler: `"off"`, `"on"`, `"stream"`. Aracı başına `agents.entries.*.reasoningDefault` bu varsayılanı geçersiz kılar. Yapılandırılmış akıl yürütme varsayılanları, yalnızca mesaj veya oturum başına bir akıl yürütme geçersiz kılması ayarlanmamışsa sahipler, yetkili göndericiler ya da operatör-yönetici Gateway bağlamları için uygulanır.
- `elevatedDefault`: aracılar için varsayılan yükseltilmiş çıktı düzeyi. Değerler: `"off"`, `"on"`, `"ask"`, `"full"`. Varsayılan: `"on"`.
- `model.primary`: `provider/model` biçimi (ör. Codex OAuth erişimi için `openai/gpt-5.6-sol`). Sağlayıcıyı atlarsanız OpenClaw önce bir diğer adı, ardından tam olarak bu model kimliği için yapılandırılmış sağlayıcılar arasındaki benzersiz eşleşmeyi dener ve ancak bundan sonra yapılandırılmış varsayılan sağlayıcıya geri döner (kullanımdan kaldırılmış uyumluluk davranışı; bu nedenle açık `provider/model` tercih edin). Bu sağlayıcı artık yapılandırılmış varsayılan modeli sunmuyorsa OpenClaw, kaldırılmış sağlayıcıya ait geçersiz bir varsayılanı göstermek yerine yapılandırılmış ilk sağlayıcıya/modele geri döner.
- `contextTokens`: isteğe bağlı, aracı genelinde geçerli üst sınır. Daha büyük bir modelin etkin bütçesini azaltabilir ancak bir modeli yapılandırılmış veya keşfedilmiş `contextTokens` değerinin üzerine çıkaramaz. Tek bir doğrudan OpenAI modelini daha büyük yerel penceresine dahil etmek için bu modelde `models.providers.openai.models[].contextWindow` ve `contextTokens` değerlerini ayarlayın; bkz. [OpenAI bağlam penceresi varsayılanları](/tr/providers/openai#context-window-defaults-and-long-context-opt-in).
- `models`: yapılandırılmış diğer adlar ve model başına ayarlar. Her girdi `alias` (kısayol) ve `params` (sağlayıcıya özgü; örneğin `temperature`, `maxTokens`, `cacheRetention`, `context1m`, `responsesServerCompaction`, `responsesCompactThreshold`, OpenRouter `provider` yönlendirmesi, `chat_template_kwargs`, `extra_body`/`extraBody`) içerebilir. Girdi eklemek model geçersiz kılmalarını kısıtlamaz.
  - Her model kimliğini elle listelemeden seçilen sağlayıcılara ait keşfedilmiş tüm modelleri göstermek için `"openai/*": {}` veya `"vllm/*": {}` gibi `provider/*` girdilerini kullanın.
  - Bu sağlayıcı için dinamik olarak keşfedilen her modelin aynı çalışma zamanını kullanması gerektiğinde bir `provider/*` girdisine `agentRuntime` ekleyin. Tam `provider/model` çalışma zamanı ilkesi yine joker karaktere üstün gelir.
  - Güvenli meta veri düzenlemeleri: girdi eklemek için `openclaw config set agents.defaults.models '<json>' --strict-json --merge` kullanın. `--replace` iletmediğiniz sürece `config set`, mevcut girdileri kaldıracak değiştirmeleri reddeder.
- `modelPolicy.allow`: açık geçersiz kılma izin listesi. Diğer adları, tam `provider/model` başvurularını ve `openai/*` veya `clawrouter/anthropic/*` gibi sondaki önek joker karakterlerini kabul eder. Herhangi bir modele izin vermek için bunu atlayın veya `[]` kullanın. `agents.entries.*.modelPolicy.allow`, ilgili aracının varsayılan ilkesinin yerini alır; açıkça belirtilen boş liste, bu aracı için tümüne izin verilmesini etkinleştirir.
  - Sağlayıcı kapsamlı yapılandırma/ilk katılım akışları, seçilen sağlayıcı modellerini bu haritayla birleştirir ve önceden yapılandırılmış ilgisiz sağlayıcıları korur.
  - Doğrudan OpenAI Responses modellerinde sunucu tarafı Compaction otomatik olarak etkinleştirilir. `context_management` eklenmesini durdurmak için `params.responsesServerCompaction: false`, eşiği geçersiz kılmak için `params.responsesCompactThreshold` kullanın. Bkz. [OpenAI sunucu tarafı Compaction](/tr/providers/openai#advanced-configuration).
- `params`: tüm modellere uygulanan genel varsayılan sağlayıcı parametreleri. `agents.defaults.params` konumunda ayarlayın (ör. `{ cacheRetention: "long" }`).
- `params` birleştirme önceliği (yapılandırma): `agents.defaults.params` (genel temel) `agents.defaults.models["provider/model"].params` (model başına) tarafından geçersiz kılınır, ardından `agents.entries.*.params` (eşleşen aracı kimliği) anahtar bazında geçersiz kılar. Ayrıntılar için [İstem Önbelleğe Alma](/tr/reference/prompt-caching) bölümüne bakın.
- `models.providers.openrouter.params.provider`: OpenRouter genelindeki varsayılan sağlayıcı yönlendirme ilkesi. OpenClaw bunu OpenRouter'ın istek `provider` nesnesine iletir; model başına `agents.defaults.models["openrouter/<model>"].params.provider` ve aracı parametreleri anahtar bazında geçersiz kılar. Bkz. [OpenRouter sağlayıcı yönlendirmesi](/tr/providers/openrouter#advanced-configuration).
- `params.extra_body`/`params.extraBody`: OpenAI uyumlu vekiller için `api: "openai-completions"` istek gövdeleriyle birleştirilen gelişmiş doğrudan geçişli JSON. Oluşturulan istek anahtarlarıyla çakışırsa ek gövde üstün gelir; yerel olmayan tamamlama rotaları daha sonra yine yalnızca OpenAI'a özgü `store` değerini çıkarır.
- `params.chat_template_kwargs`: üst düzey `api: "openai-completions"` istek gövdeleriyle birleştirilen vLLM/OpenAI uyumlu sohbet şablonu bağımsız değişkenleri. Düşünme kapalıyken `vllm/nemotron-3-*` için paketlenmiş vLLM Plugin'i otomatik olarak `enable_thinking: false` ve `force_nonempty_content: true` gönderir; açık `chat_template_kwargs` oluşturulan varsayılanları geçersiz kılar ve `extra_body.chat_template_kwargs` yine son önceliğe sahiptir. Yapılandırılmış vLLM Qwen ve Nemotron düşünme modelleri, çok düzeyli çaba kademesi yerine ikili `/think` seçenekleri (`off`, `on`) sunar.
- `compat.thinkingFormat`: OpenAI uyumlu düşünme yükü biçemi. Together tarzı `reasoning.enabled` için `"together"`, Qwen tarzı üst düzey `enable_thinking` için `"qwen"` veya vLLM gibi istek düzeyinde sohbet şablonu anahtar sözcük bağımsız değişkenlerini destekleyen Qwen ailesi arka uçlarında `chat_template_kwargs.enable_thinking` için `"qwen-chat-template"` kullanın. OpenClaw, devre dışı düşünmeyi `false` değerine ve etkin düşünmeyi `true` değerine eşler; yapılandırılmış vLLM Qwen modelleri bu biçimler için ikili `/think` seçenekleri sunar.
- `compat.supportedReasoningEfforts`: model başına OpenAI uyumlu akıl yürütme eforu listesi. Bunu gerçekten kabul eden özel uç noktalar için `"xhigh"` öğesini ekleyin; OpenClaw ardından yapılandırılan sağlayıcı/model için komut menülerinde, Gateway oturum satırlarında, oturum yaması doğrulamasında, ajan CLI doğrulamasında ve `llm-task` doğrulamasında `/think xhigh` öğesini sunar. Arka uç, standart bir düzey için sağlayıcıya özgü bir değer istediğinde `compat.reasoningEffortMap` kullanın.
- `params.preserveThinking`: korunan düşünme için yalnızca Z.AI'a özgü isteğe bağlı etkinleştirme. Etkinleştirildiğinde ve düşünme açıkken OpenClaw, `thinking.clear_thinking: false` gönderir ve önceki `reasoning_content` öğelerini yeniden oynatır; bkz. [Z.AI düşünme ve korunan düşünme](/tr/providers/zai#advanced-configuration).
- `localService`: yerel/kendi barındırılan model sunucuları için isteğe bağlı, sağlayıcı düzeyinde süreç yöneticisi. Seçilen model bu sağlayıcıya ait olduğunda OpenClaw, `healthUrl` (veya `baseUrl + "/models"`) yoklaması yapar; uç nokta çalışmıyorsa `command` öğesini `args` ile başlatır, `readyTimeoutMs` süresine kadar bekler ve ardından model isteğini gönderir. `command` mutlak bir yol olmalıdır. `idleStopMs: 0`, OpenClaw kapanana kadar süreci çalışır durumda tutar; pozitif bir değer, OpenClaw tarafından başlatılan süreci belirtilen sayıda milisaniye boşta kaldıktan sonra durdurur. Bkz. [Yerel model hizmetleri](/tr/gateway/local-model-services).
- Çalışma zamanı politikası `agents.defaults` üzerinde değil, sağlayıcılar veya modeller üzerinde tanımlanmalıdır. Sağlayıcı genelindeki kurallar için `models.providers.<provider>.agentRuntime`, modele özgü kurallar için `agents.defaults.models["provider/model"].agentRuntime` / `agents.entries.*.models["provider/model"].agentRuntime` kullanın. Sağlayıcı/model öneki tek başına hiçbir zaman bir çalıştırma düzeneği seçmez. Çalışma zamanı ayarlanmamışsa veya `auto` ise OpenAI, yalnızca tam olarak resmî bir HTTPS Platform Responses veya ChatGPT Responses rotası kullanıldığında ve istekte kullanıcı tarafından belirtilmiş bir geçersiz kılma olmadığında Codex'i örtük olarak seçebilir. Bkz. [OpenAI örtük ajan çalışma zamanı](/tr/providers/openai#implicit-agent-runtime).
- Bu alanları değiştiren yapılandırma yazıcıları (örneğin `/models set`, `/models set-image` ve geri dönüş ekleme/kaldırma komutları), standart nesne biçiminde kaydeder ve mümkün olduğunda mevcut geri dönüş listelerini korur.
- `maxConcurrent`: oturumlar genelinde aynı anda çalışabilen en fazla ajan çalıştırması sayısı (her oturum yine sıralı olarak çalıştırılır). Varsayılan: `4`.

### Çalışma zamanı politikası

```json5
{
  models: {
    providers: {
      openai: {
        agentRuntime: { id: "codex" },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
        "vllm/*": {
          agentRuntime: { id: "openclaw" },
        },
      },
    },
  },
}
```

- `id`: `"auto"`, `"openclaw"`, kayıtlı bir plugin çalışma düzeneği kimliği veya desteklenen bir CLI arka uç diğer adı. Birlikte gelen Codex plugin'i `codex` kaydını yapar; birlikte gelen Anthropic plugin'i `claude-cli` CLI arka ucunu sağlar.
- `id: "auto"`, kayıtlı plugin çalışma düzeneklerinin destek sözleşmelerini bildiren veya başka şekilde karşılayan etkin rotaları üstlenmesine olanak tanır ve hiçbir çalışma düzeneği eşleşmediğinde OpenClaw'ı kullanır. `id: "codex"` gibi açık bir plugin çalışma zamanı, söz konusu çalışma düzeneğini ve uyumlu bir etkin rotayı gerektirir; bunlardan biri kullanılamıyorsa veya yürütme başarısız olursa güvenli biçimde başarısız olur.
- `id: "pi"`, v2026.5.22 ve önceki sürümlerden dağıtılmış yapılandırmaları korumak için yalnızca `openclaw` öğesinin kullanımdan kaldırılmış diğer adı olarak kabul edilir. Yeni yapılandırma `openclaw` kullanmalıdır.
- Çalışma zamanı önceliğinde önce tam model politikası (`agents.entries.*.models["provider/model"]`, `agents.defaults.models["provider/model"]` veya `models.providers.<provider>.models[]`), ardından `agents.entries.*` / `agents.defaults.models["provider/*"]`, son olarak `models.providers.<provider>.agentRuntime` konumundaki sağlayıcı genelindeki politika uygulanır.
- Tüm ajana yönelik çalışma zamanı anahtarları eskidir. `agents.defaults.agentRuntime`, `agents.entries.*.agentRuntime`, oturum çalışma zamanı sabitlemeleri ve `OPENCLAW_AGENT_RUNTIME` çalışma zamanı seçiminde yok sayılır. Eski değerleri kaldırmak için `openclaw doctor --fix` komutunu çalıştırın.
- Yazılmış bir istek geçersiz kılması bulunmayan uygun ve tam resmî HTTPS OpenAI Responses/ChatGPT rotaları, Codex çalışma düzeneğini örtük olarak kullanabilir. Sağlayıcı/model `agentRuntime.id: "codex"`, Codex'i güvenli biçimde başarısız olan bir gereksinim hâline getirir ancak uyumsuz bir rotayı uyumlu hâle getirmez.
- Claude CLI dağıtımları için `model: "anthropic/claude-opus-5"` ile model kapsamlı `agentRuntime.id: "claude-cli"` kullanımını tercih edin. Eski `claude-cli/<model>` başvuruları uyumluluk amacıyla çalışmaya devam eder ancak yeni yapılandırma, sağlayıcı/model seçimini standart biçimde tutmalı ve yürütme arka ucunu sağlayıcı/model çalışma zamanı politikasına yerleştirmelidir.
- Bu yalnızca metin ajan turu yürütmesini denetler. Medya oluşturma, görüntü, PDF, müzik, video ve TTS yine kendi sağlayıcı/model ayarlarını kullanır.

**Yerleşik diğer ad kısaltmaları** (yalnızca model `agents.defaults.models` içinde olduğunda uygulanır):

| Diğer ad            | Model                           |
| ------------------- | ------------------------------- |
| `opus`              | `anthropic/claude-opus-5`       |
| `sonnet`            | `anthropic/claude-sonnet-5`     |
| `gpt`               | `openai/gpt-5.4`                |
| `gpt-mini`          | `openai/gpt-5.4-mini`           |
| `gpt-nano`          | `openai/gpt-5.4-nano`           |
| `gemini`            | `google/gemini-3.1-pro-preview` |
| `gemini-flash`      | `google/gemini-3-flash-preview` |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite`  |

Yapılandırdığınız diğer adlar her zaman varsayılanlardan önceliklidir.

Z.AI GLM-4.x modelleri, `--thinking off` ayarlamadığınız veya `agents.defaults.models["zai/<model>"].params.thinking` öğesini kendiniz tanımlamadığınız sürece düşünme modunu otomatik olarak etkinleştirir.
Z.AI modelleri, araç çağrısı akışı için varsayılan olarak `tool_stream` özelliğini etkinleştirir. Devre dışı bırakmak için `agents.defaults.models["zai/<model>"].params.tool_stream` değerini `false` olarak ayarlayın.
Anthropic Claude Opus 4.8, OpenClaw'da düşünmeyi varsayılan olarak kapalı tutar; uyarlanabilir düşünme açıkça etkinleştirildiğinde Anthropic'in sağlayıcıya ait varsayılan çaba düzeyi `high` olur. Açık bir düşünme düzeyi ayarlanmadığında Claude 4.6 modelleri varsayılan olarak `adaptive` kullanır.

### CLI arka ucu seçimi

CLI bağdaştırıcı mekanizmaları, ajan varsayılanları altında yapılandırılmak yerine plugin'ler tarafından kaydedilir. Yukarıda gösterildiği gibi, model kapsamlı `agentRuntime.id` ile kayıtlı bir CLI arka ucu seçin. İşlemler için [CLI arka uçları](/tr/gateway/cli-backends), komut, oturum, görüntü ve ayrıştırıcı kaydı için [CLI arka uç plugin'leri oluşturma](/tr/plugins/cli-backend-plugins) bölümüne bakın.

### `agents.defaults.promptOverlays`

OpenClaw tarafından birleştirilen istem yüzeylerinde model ailesine göre uygulanan, sağlayıcıdan bağımsız istem katmanları. GPT-5 ailesi model kimlikleri, OpenClaw/sağlayıcı rotaları genelinde paylaşılan davranış sözleşmesini alır; `personality` yalnızca kullanıcı dostu etkileşim stili katmanını denetler. Yerel Codex uygulama sunucusu rotaları, bu OpenClaw GPT-5 katmanı yerine Codex'e ait temel/model talimatlarını korur ve OpenClaw, yerel ileti dizileri için Codex'in yerleşik kişiliğini devre dışı bırakır.

```json5
{
  agents: {
    defaults: {
      promptOverlays: {
        gpt5: {
          personality: "friendly", // friendly | on | off
        },
      },
    },
  },
}
```

- `"friendly"` (varsayılan) ve `"on"`, kullanıcı dostu etkileşim stili katmanını etkinleştirir.
- `"off"` yalnızca kullanıcı dostu katmanı devre dışı bırakır; etiketli GPT-5 davranış sözleşmesi etkin kalır.
- Bu paylaşılan ayar belirlenmediğinde eski `plugins.entries.openai.config.personality` değeri okunmaya devam eder.

### `agents.defaults.heartbeat`

Düzenli Heartbeat çalıştırmaları.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m disables
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        includeSystemPromptSection: true, // default: true; false omits the Heartbeat section from the system prompt
        lightContext: false, // default: false; true skips workspace bootstrap files for heartbeat runs
        isolatedSession: false, // default: false; true runs each heartbeat in a fresh session (no conversation history)
        skipWhenBusy: false, // default: false; true also waits for this agent's subagent/nested lanes
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (default) | block
        target: "none", // default: none | options: last | whatsapp | telegram | discord | ...
        prompt: "Follow the heartbeat monitor scratch context...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
        timeoutSeconds: 45,
      },
    },
  },
}
```

- `every`: süre dizesi (ms/s/m/h). Varsayılan: `30m` (API anahtarıyla kimlik doğrulama) veya `1h` (OAuth kimlik doğrulaması). Devre dışı bırakmak için `0m` olarak ayarlayın.
- Sıklık, sistemin sahip olduğu bir Cron izleyicisi satırına yazılır. Eksik veya eski bir satırı oluşturmak için `openclaw doctor --fix` komutunu çalıştırın. Cron devre dışıysa zamanlanmış Heartbeat'ler çalışmaz ve Gateway başlangıçta bir uyarı günlüğe kaydeder.
- `includeSystemPromptSection`: false olduğunda Heartbeat bölümünü sistem isteminden çıkarır. Varsayılan: `true`.
- `suppressToolErrorWarnings`: true olduğunda Heartbeat çalıştırmaları sırasında araç hatası uyarı yüklerini engeller.
- `timeoutSeconds`: bir Heartbeat ajan turunun iptal edilmeden önce çalışmasına izin verilen saniye cinsinden azami süre. Ayarlanmışsa `agents.defaults.timeoutSeconds` değerini, aksi takdirde 600 saniyeyle sınırlandırılmış Heartbeat sıklığını kullanmak için ayarlamadan bırakın.
- `directPolicy`: doğrudan/DM teslimat politikası. `allow` (varsayılan), doğrudan hedefe teslimata izin verir. `block`, doğrudan hedefe teslimatı engeller ve `reason=dm-blocked` üretir.
- `lightContext`: true olduğunda Heartbeat çalıştırmaları hafif başlangıç bağlamı kullanır ve çalışma alanı başlangıç dosyalarını atlar. İzleyici karalama alanı her iki durumda da Heartbeat çalıştırıcısı tarafından eklenir.
- `isolatedSession`: true olduğunda her Heartbeat, önceden konuşma geçmişi olmadan yeni bir oturumda çalışır. Cron `sessionTarget: "isolated"` ile aynı yalıtım örüntüsüdür. Heartbeat başına token maliyetini ~100K'dan ~2-5K token'a düşürür.
- `skipWhenBusy`: true olduğunda Heartbeat çalıştırmaları, ilgili ajanın ek meşgul kulvarlarında ertelenir: kendi oturum anahtarlı alt ajanı veya iç içe komut çalışması. Bu bayrak olmasa bile Cron kulvarları Heartbeat'leri her zaman erteler.
- Ajan başına: `agents.entries.*.heartbeat` ayarını yapın. Herhangi bir ajan `heartbeat` tanımladığında Heartbeat'leri **yalnızca bu ajanlar** çalıştırır.
- Heartbeat'ler tam ajan turları çalıştırır — daha kısa aralıklar daha fazla token tüketir.

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // default | safeguard
        provider: "my-provider", // id of a registered compaction provider plugin (optional)
        thinkingLevel: "low", // optional compaction-only thinking override
        timeoutSeconds: 180,
        keepRecentTokens: 50000,
        recentTurnsPreserve: 3,
        identifierPolicy: "strict", // strict | off
        qualityGuard: { enabled: true, maxRetries: 1 },
        midTurnPrecheck: { enabled: false }, // optional tool-loop pressure check
        postIndexSync: "async", // off | async | await
        postCompactionSections: ["Session Startup", "Red Lines"],
        model: "openrouter/anthropic/claude-sonnet-4-6", // optional compaction-only model override
        truncateAfterCompaction: true, // rotate to a smaller successor JSONL after compaction
        maxActiveTranscriptBytes: "20mb", // optional preflight local compaction trigger
        notifyUser: true, // notices when compaction starts/completes and on memory-flush degradation (default: false)
        memoryFlush: {
          enabled: true,
          model: "ollama/qwen3:8b", // optional memory-flush-only model override
          softThresholdTokens: 6000,
          forceFlushTranscriptBytes: "2mb",
        },
      },
    },
  },
}
```

- `mode`: `default` veya `safeguard` (uzun geçmişler için parçalı özetleme). Bkz. [Compaction](/tr/concepts/compaction).
- `provider`: kayıtlı bir Compaction sağlayıcı Plugin'inin kimliği. Ayarlandığında, yerleşik LLM özetlemesi yerine sağlayıcının `summarize()` işlevi çağrılır. Başarısızlık durumunda yerleşik işlev kullanılır. Bir sağlayıcı ayarlamak `mode: "safeguard"` kullanımını zorunlu kılar. Bkz. [Compaction](/tr/concepts/compaction).
- `thinkingLevel`: yalnızca gömülü OpenClaw Compaction özetleri için kullanılan isteğe bağlı düşünme düzeyi (`off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `adaptive`, `max` veya `ultra`). Oturumun geçerli düşünme düzeyini geçersiz kılar ve seçilen Compaction modeline/çalışma zamanına göre sınırlandırılır. Oturum düzeyini devralmak için ayarlamayın. Yerel Codex app-server Compaction işlemi bu ayarı yok sayar çünkü yerel compact isteğinde işlem başına düşünme geçersiz kılması yoktur; yapılandırıldığında OpenClaw bir uyarı kaydeder.
- `timeoutSeconds`: OpenClaw'ın tek bir Compaction işlemini iptal etmesinden önce izin verilen azami saniye sayısı. Varsayılan: `180`.
- `keepRecentTokens`: en son transkript kuyruğunu olduğu gibi tutmaya yönelik ajan kesme noktası bütçesi. Açıkça ayarlandığında manuel `/compact` buna uyar; aksi takdirde manuel Compaction kesin bir denetim noktasıdır.
- `recentTurnsPreserve`: koruma özetlemesinin dışında olduğu gibi tutulan en son kullanıcı/asistan dönüşlerinin sayısı. Varsayılan: `3`.
- `identifierPolicy`: `strict` (varsayılan) veya `off`. `strict`, Compaction özetlemesi sırasında yerleşik opak tanımlayıcı koruma yönergelerini başa ekler.
- `qualityGuard`: koruma özetleri için hatalı biçimlendirilmiş çıktıda yeniden deneme kontrolleri. Koruma modunda varsayılan olarak etkindir; denetimi atlamak için `enabled: false` olarak ayarlayın.
- `midTurnPrecheck`: isteğe bağlı araç döngüsü baskısı denetimi. `enabled: true` olduğunda OpenClaw, araç sonuçları eklendikten sonra ve bir sonraki model çağrısından önce bağlam baskısını denetler. Bağlam artık sığmıyorsa istemi göndermeden önce geçerli denemeyi iptal eder ve araç sonuçlarını kırpmak ya da Compaction gerçekleştirip yeniden denemek için mevcut ön denetim kurtarma yolunu yeniden kullanır. Hem `default` hem de `safeguard` Compaction modlarıyla çalışır. Varsayılan: devre dışı.
- `postIndexSync`: Compaction sonrası oturum belleğini yeniden indeksleme modu. Varsayılan: `"async"`. En yüksek güncellik için `"await"`, daha düşük Compaction gecikmesi için `"async"` veya yalnızca oturum belleği eşitlemesi başka bir yerde gerçekleştiriliyorsa `"off"` kullanın.
- `postCompactionSections`: Compaction sonrasında yeniden eklenecek isteğe bağlı AGENTS.md H2/H3 bölüm adları. Devre dışı bırakmak için ayarlamayın veya `[]` kullanın.
- `model`: yalnızca Compaction özetlemesi için isteğe bağlı `provider/model-id` veya `agents.defaults.models` içinden yalın takma ad. Yalın takma adlar gönderimden önce çözümlenir; yapılandırılmış değişmez model kimlikleri çakışmalarda önceliğini korur. Ana oturumun bir modeli kullanmaya devam etmesi, ancak Compaction özetlerinin başka bir modelde çalışması gerektiğinde bunu kullanın; ayarlanmadığında Compaction, oturumun birincil modelini kullanır.
- `truncateAfterCompaction`: Compaction sonrasında etkin oturum transkriptini döndürür; böylece önceki tam transkript arşivlenmiş olarak kalırken gelecekteki dönüşler yalnızca özeti ve özetlenmemiş kuyruğu yükler. Uzun süre çalışan oturumlarda etkin transkriptin sınırsız büyümesini önler. Varsayılan: `false`.
- `maxActiveTranscriptBytes`: transkript geçmişi eşiği aştığında bir çalıştırmadan önce normal yerel Compaction işlemini tetikleyen isteğe bağlı bayt eşiği (`number` veya `"20mb"` gibi dizeler). Başarılı Compaction işleminin daha küçük bir ardıl transkripte dönebilmesi için `truncateAfterCompaction` gerektirir. Ayarlanmadığında veya `0` olduğunda devre dışıdır.
- `notifyUser`: `true` olduğunda kullanıcıya kısa bağlam bakımı bildirimleri gönderir: Compaction başladığında ve tamamlandığında (örneğin, "Bağlam sıkıştırılıyor..." ve "Compaction tamamlandı") ve Compaction öncesi bellek boşaltma olanakları tükendiğinde, böylece yanıt kısıtlı bir durumda devam ettiğinde (örneğin, "Bellek bakımı geçici olarak başarısız oldu; yanıtınız sürdürülüyor."). Bu bildirimlerin sessiz kalması için varsayılan olarak devre dışıdır.
- `memoryFlush`: kalıcı anıları depolamak amacıyla otomatik Compaction öncesinde gerçekleştirilen sessiz ajansal dönüş. Bu bakım dönüşünün yerel bir modelde kalması gerektiğinde `model` değerini `ollama/qwen3:8b` gibi tam bir sağlayıcı/model olarak ayarlayın; geçersiz kılma etkin oturumun geri dönüş zincirini devralmaz. `forceFlushTranscriptBytes`, belirteç sayaçları güncel olmasa bile transkript boyutu eşiğe ulaştığında boşaltmayı zorunlu kılar. Çalışma alanı salt okunur olduğunda atlanır.

Özel Compaction yönergeleri kodun mülkiyetindedir. Özel özet oluşturma için
`summarize()` içeren bir Compaction sağlayıcı Plugin'i uygulayın ve Compaction
sonrası bağlamın sonraki model istemlerine eklenmesi gerektiğinde
`before_prompt_build` kullanın. Doctor, kullanımdan kaldırılmış yönerge alanlarını kaldırır ve bu
bağlantı noktalarına yönlendirir.

### `agents.defaults.contextPruning`

LLM'ye göndermeden önce bellek içi bağlamdan **eski araç sonuçlarını** budar. Diskteki oturum geçmişini **değiştirmez**. Varsayılan olarak devre dışıdır; etkinleştirmek için `mode: "cache-ttl"` olarak ayarlayın.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // kapalı (varsayılan) | cache-ttl
      },
    },
  },
}
```

<Accordion title="cache-ttl modu davranışı">

- `mode: "cache-ttl"` budama geçişlerini etkinleştirir.
- Budama, önce aşırı büyük araç sonuçlarını ölçülü biçimde kırpar, ardından gerekirse daha eski araç sonuçlarını tamamen temizler.

**Ölçülü kırpma**, başlangıcı ve sonu koruyup ortaya `...` ekler.

**Tam temizleme**, araç sonucunun tamamını yer tutucuyla değiştirir.

Notlar:

- Görüntü blokları hiçbir zaman kırpılmaz/temizlenmez.
- Oranlar kesin belirteç sayılarına değil, karakterlere dayalıdır (yaklaşıktır).
- En son asistan iletileri korunur.

</Accordion>

Davranış ayrıntıları için [Oturum Budama](/tr/concepts/session-pruning) bölümüne bakın.

### Blok akışı

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200, breakPreference: "paragraph" },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off (varsayılan) | natural | custom (minMs/maxMs kullanın)
    },
  },
}
```

- Telegram dışındaki kanallarda blok yanıtlarını etkinleştirmek için açıkça `*.streaming.block.enabled: true` gerekir. QQ Bot istisnadır: `streaming.block` anahtarları yoktur ve `channels.qqbot.streaming.mode`, `"off"` olmadığı sürece blok yanıtlarını akışla gönderir.
- Kanal geçersiz kılmaları: `channels.<channel>.streaming.block.coalesce` (ve hesap başına değişkenleri). Discord, Google Chat, Mattermost, MS Teams, Signal ve Slack varsayılan olarak `minChars: 1500` / `idleMs: 1000` kullanır.
- `blockStreamingChunk.breakPreference`: tercih edilen parça sınırı (`"paragraph" | "newline" | "sentence"`).
- `humanDelay`: blok yanıtları arasındaki rastgele bekleme. Varsayılan: `off`. `natural` = 800-2500ms. `custom`, `minMs`/`maxMs` kullanır (ayarlanmamış herhangi bir sınır için doğal aralığa geri döner). Ajan başına geçersiz kılma: `agents.entries.*.humanDelay`.

Davranış ve parçalama ayrıntıları için [Akış](/tr/concepts/streaming) bölümüne bakın.

### Yazma göstergeleri

```json5
{
  agents: {
    defaults: {
      typingMode: "instant", // never | instant | thinking | message
      typingIntervalSeconds: 6,
    },
  },
}
```

- Varsayılanlar: doğrudan sohbetler/bahsetmeler için `instant`, bahsetme içermeyen grup sohbetleri için `message`.
- `typingIntervalSeconds` varsayılanı: `6`.
- Ajan başına geçersiz kılma: `agents.entries.*.typingMode`.

Bkz. [Yazma Göstergeleri](/tr/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

Gömülü ajan için isteğe bağlı korumalı alan kullanımı. Tam kılavuz için [Korumalı Alan Kullanımı](/tr/gateway/sandboxing) bölümüne bakın.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off (varsayılan) | non-main | all
        backend: "docker", // docker (varsayılan) | ssh | openshell
        scope: "agent", // session | agent (varsayılan) | shared
        workspaceAccess: "none", // none (varsayılan) | ro | rw
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          containerPrefix: "openclaw-sbx-",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git curl jq",
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          gpus: "all",
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
          dns: ["1.1.1.1", "8.8.8.8"],
          extraHosts: ["internal.service:10.0.0.5"],
          binds: ["/home/user/source:/source:rw"],
        },
        ssh: {
          target: "user@gateway-host:22",
          command: "ssh",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // SecretRefs / satır içi içerikler de desteklenir:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
        browser: {
          enabled: false,
          image: "openclaw-sandbox-browser:bookworm-slim",
          network: "openclaw-sandbox-browser",
          cdpPort: 9222,
          cdpSourceRange: "172.21.0.1/32",
          vncPort: 5900,
          noVncPort: 6080,
          headless: false,
          enableNoVnc: true,
          allowHostControl: false,
          autoStart: true,
          autoStartTimeoutMs: 12000,
        },
        prune: {
          idleHours: 24,
          maxAgeDays: 7,
        },
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        allow: [
          "exec",
          "process",
          "read",
          "write",
          "edit",
          "apply_patch",
          "sessions_list",
          "sessions_history",
          "sessions_send",
          "sessions_spawn",
          "session_status",
        ],
        deny: ["browser", "canvas", "nodes", "cron", "discord", "gateway"],
      },
    },
  },
}
```

Yukarıda gösterilen varsayılanlar (`off`/`docker`/`agent`/`none`/`bookworm-slim` görüntüsü/`none` ağı/vb.) yalnızca örnek değerler değil, gerçek OpenClaw varsayılanlarıdır.

<Accordion title="Korumalı alan ayrıntıları">

**Arka uç:**

- `docker`: yerel Docker çalışma zamanı (varsayılan)
- `ssh`: genel SSH destekli uzak çalışma zamanı
- `openshell`: OpenShell çalışma zamanı

`backend: "openshell"` seçildiğinde çalışma zamanına özgü ayarlar
`plugins.entries.openshell.config` konumuna taşınır.

**SSH arka uç yapılandırması:**

- `target`: `user@host[:port]` biçiminde SSH hedefi
- `command`: SSH istemci komutu (varsayılan: `ssh`)
- `workspaceRoot`: kapsam başına çalışma alanları için kullanılan mutlak uzak kök (varsayılan: `/tmp/openclaw-sandboxes`)
- `identityFile` / `certificateFile` / `knownHostsFile`: OpenSSH'ye aktarılan mevcut yerel dosyalar
- `identityData` / `certificateData` / `knownHostsData`: OpenClaw'un çalışma zamanında geçici dosyalara dönüştürdüğü satır içi içerikler veya SecretRef'ler
- `strictHostKeyChecking` / `updateHostKeys`: OpenSSH ana makine anahtarı ilkesi ayarları (ikisinin de varsayılanı `true`)

**SSH kimlik doğrulama önceliği:**

- `identityData`, `identityFile` değerine göre önceliklidir
- `certificateData`, `certificateFile` değerine göre önceliklidir
- `knownHostsData`, `knownHostsFile` değerine göre önceliklidir
- SecretRef destekli `*Data` değerleri, korumalı alan oturumu başlamadan önce etkin gizli bilgiler çalışma zamanı anlık görüntüsünden çözümlenir

**SSH arka uç davranışı:**

- uzak çalışma alanını oluşturma veya yeniden oluşturma işleminden sonra bir kez başlangıç verileriyle doldurur
- ardından uzak SSH çalışma alanını kurallı kaynak olarak tutar
- `exec`, dosya araçları ve medya yollarını SSH üzerinden yönlendirir
- uzak değişiklikleri ana makineye otomatik olarak geri eşitlemez
- korumalı alan tarayıcı kapsayıcılarını desteklemez

**Çalışma alanı erişimi:**

- `none`: `~/.openclaw/sandboxes` altındaki kapsam başına korumalı alan çalışma alanı (varsayılan)
- `ro`: `/workspace` konumundaki korumalı alan çalışma alanı; aracı çalışma alanı `/agent` konumuna salt okunur olarak bağlanır
- `rw`: aracı çalışma alanı `/workspace` konumuna okuma/yazma erişimiyle bağlanır

**Kapsam:**

- `session`: oturum başına kapsayıcı + çalışma alanı
- `agent`: aracı başına bir kapsayıcı + çalışma alanı (varsayılan)
- `shared`: paylaşılan kapsayıcı ve çalışma alanı (oturumlar arası yalıtım yoktur)

**OpenShell Plugin yapılandırması:**

```json5
{
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          mode: "mirror", // yansıtma (varsayılan) | uzak
          command: "openshell",
          from: "openclaw",
          remoteWorkspaceDir: "/sandbox",
          remoteAgentWorkspaceDir: "/agent",
          gateway: "lab", // isteğe bağlı
          gatewayEndpoint: "https://lab.example", // isteğe bağlı
          policy: "strict", // isteğe bağlı OpenShell ilkesi kimliği
          providers: ["openai"], // isteğe bağlı
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**OpenShell modu:**

- `mirror`: yürütmeden önce uzağı yerelden başlangıç verileriyle doldurur, yürütmeden sonra geri eşitler; yerel çalışma alanı kurallı kaynak olarak kalır
- `remote`: korumalı alan oluşturulduğunda uzağı bir kez başlangıç verileriyle doldurur, ardından uzak çalışma alanını kurallı kaynak olarak tutar

`remote` modunda, başlangıç verileriyle doldurma adımından sonra OpenClaw dışında yapılan ana makine yerelindeki düzenlemeler korumalı alana otomatik olarak eşitlenmez.
Aktarım, OpenShell korumalı alanına SSH üzerinden yapılır; ancak korumalı alan yaşam döngüsünü ve isteğe bağlı yansıtma eşitlemesini Plugin yönetir.

**`setupCommand`**, kapsayıcı oluşturulduktan sonra (`sh -lc` aracılığıyla) bir kez çalışır. Ağ çıkışı, yazılabilir kök dizin ve kök kullanıcı gerektirir.

**Kapsayıcıların varsayılanı `network: "none"` değeridir** — aracının dışarıya erişmesi gerekiyorsa `"bridge"` (veya özel bir köprü ağı) olarak ayarlayın.
`"host"` engellenir. `sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` değerini açıkça ayarlamadığınız sürece `"container:<id>"` varsayılan olarak engellenir
(acil durum erişimi).
Etkin bir OpenClaw korumalı alanındaki Codex uygulama sunucusu turları, yerel kod modu ağ erişimleri için aynı çıkış ayarını kullanır.

**Gelen ekler**, etkin çalışma alanındaki `media/inbound/*` konumuna hazırlanır.

**`docker.binds`**, ek ana makine dizinlerini bağlar; genel ve aracı başına bağlamalar birleştirilir.

**Korumalı alan tarayıcısı** (`sandbox.browser.enabled`, varsayılan `false`): Bir kapsayıcı içinde Chromium + CDP. noVNC URL'si sistem istemine eklenir. `openclaw.json` içinde `browser.enabled` gerektirmez.
noVNC gözlemci erişimi varsayılan olarak VNC kimlik doğrulamasını kullanır ve OpenClaw, parolayı paylaşılan URL'de göstermek yerine kısa ömürlü bir belirteç URL'si oluşturur.

- `allowHostControl: false` (varsayılan), korumalı alan oturumlarının ana makine tarayıcısını hedeflemesini engeller.
- `network` varsayılan olarak `openclaw-sandbox-browser` değerini kullanır (ayrılmış köprü ağı). Yalnızca genel köprü bağlantısını açıkça istediğinizde `bridge` olarak ayarlayın. `"host"` burada da engellenir.
- `cdpSourceRange`, isteğe bağlı olarak kapsayıcı sınırındaki CDP girişini bir CIDR aralığıyla sınırlar (örneğin `172.21.0.1/32`).
- `sandbox.browser.binds`, ek ana makine dizinlerini yalnızca korumalı alan tarayıcı kapsayıcısına bağlar. Ayarlandığında (`[]` dâhil), tarayıcı kapsayıcısı için `docker.binds` değerinin yerini alır.
- Korumalı alan tarayıcı kapsayıcısındaki Chromium her zaman `--no-sandbox --disable-setuid-sandbox` ile başlatılır (kapsayıcılar, Chrome'un kendi korumalı alanının ihtiyaç duyduğu çekirdek temel işlevlerine sahip değildir); bunun için bir yapılandırma anahtarı yoktur.
- Başlatma varsayılanları `scripts/sandbox-browser-entrypoint.sh` içinde tanımlanır ve kapsayıcı ana makineleri için ayarlanmıştır:
  - `--remote-debugging-address=127.0.0.1`
  - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
  - `--user-data-dir=${HOME}/.chrome`
  - `--no-first-run`
  - `--no-default-browser-check`
  - `--disable-dev-shm-usage`
  - `--disable-background-networking`
  - `--disable-breakpad`
  - `--disable-crash-reporter`
  - `--no-zygote`
  - `--metrics-recording-only`
  - `--password-store=basic`
  - `--use-mock-keychain`
  - `--disable-3d-apis`, `--disable-gpu` ve `--disable-software-rasterizer`
    varsayılan olarak etkindir ve WebGL/3D kullanımı gerektiriyorsa
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` ile devre dışı bırakılabilir.
  - `--disable-extensions` (varsayılan olarak etkin); iş akışınız bunlara bağlıysa `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0`
    uzantıları yeniden etkinleştirir.
  - varsayılan olarak `--renderer-process-limit=2`; `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>` ile değiştirin,
    Chromium'un varsayılan işlem sınırını kullanmak için `0`
    olarak ayarlayın.
  - yalnızca `headless` etkinleştirildiğinde `--headless=new`.
  - Varsayılanlar kapsayıcı görüntüsünün temel değerleridir; kapsayıcı varsayılanlarını değiştirmek için özel
    bir giriş noktasına sahip özel tarayıcı görüntüsü kullanın.

</Accordion>

Tarayıcı korumalı alanı ve `sandbox.docker.binds` yalnızca Docker'da kullanılabilir.

Görüntüleri oluşturun (bir kaynak kod kullanıma alma kopyasından):

```bash
scripts/sandbox-setup.sh           # ana korumalı alan görüntüsü
scripts/sandbox-browser-setup.sh   # isteğe bağlı tarayıcı görüntüsü
```

Kaynak kod kullanıma alma kopyası olmadan yapılan npm kurulumları için satır içi `docker build` komutları hakkında [Korumalı alan § Görüntüler ve kurulum](/tr/gateway/sandboxing#images-and-setup) bölümüne bakın.

### `agents.entries` (aracı başına geçersiz kılmalar)

Bir aracıya kendi TTS sağlayıcısını, sesini, modelini, stilini veya otomatik TTS modunu vermek için `agents.entries.*.tts` kullanın. Aracı bloğu, genel
`tts` üzerine derinlemesine birleştirilir; böylece paylaşılan kimlik bilgileri tek bir yerde kalırken ayrı ayrı
aracılar yalnızca ihtiyaç duydukları ses veya sağlayıcı alanlarını geçersiz kılabilir. Etkin aracının
geçersiz kılması otomatik sesli yanıtlara, `/tts audio`, `/tts status` ve
`tts` aracı aracına uygulanır. Sağlayıcı örnekleri ve öncelik sırası için
[Metinden konuşmaya](/tr/tools/tts#per-agent-voice-overrides) bölümüne bakın.

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        name: "Ana Aracı",
        workspace: "~/.openclaw/workspace",
        agentDir: "~/.openclaw/agents/main/agent",
        model: "anthropic/claude-opus-4-6", // veya { primary, fallbacks }
        utilityModel: "openai/gpt-5.4-mini",
        thinkingDefault: "high", // aracı başına düşünme düzeyi geçersiz kılması
        reasoningDefault: "on", // aracı başına akıl yürütme görünürlüğü geçersiz kılması
        fastModeDefault: false, // aracı başına hızlı mod geçersiz kılması
        params: { cacheRetention: "none" }, // eşleşen defaults.models parametrelerini anahtara göre geçersiz kılar
        tts: {
          providers: {
            elevenlabs: { speakerVoiceId: "EXAVITQu4vr4xnSDxMaL" },
          },
        },
        skills: ["docs-search"], // ayarlandığında agents.defaults.skills değerinin yerini alır
        identity: {
          name: "Samantha",
          theme: "yardımsever tembel hayvan",
          emoji: "🦥",
          avatar: "avatars/samantha.png",
        },
        groupChat: { mentionPatterns: ["@openclaw"] },
        sandbox: { mode: "off" },
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent", // kalıcı | tek seferlik
            cwd: "/workspace/openclaw",
          },
        },
        subagents: { allowAgents: ["*"] },
        tools: {
          profile: "coding",
          allow: ["browser"],
          deny: ["canvas"],
          elevated: { enabled: true },
        },
      },
    ],
  },
}
```

- `id`: sabit aracı kimliği (zorunlu).
- `default`: birden fazla ayarlandığında ilki geçerli olur (uyarı günlüğe kaydedilir). Hiçbiri ayarlanmamışsa ilk liste girdisi varsayılandır.
- `model`: dize biçimi, model geri dönüşü olmadan aracı başına katı birincil model ayarlar; nesne biçimi `{ primary }` de `fallbacks` eklemediğiniz sürece katıdır. Bu aracı için geri dönüşü etkinleştirmek üzere `{ primary, fallbacks: [...] }`, katı davranışı açıkça belirtmek için `{ primary, fallbacks: [] }` kullanın. Yalnızca `primary` değerini geçersiz kılan Cron işleri, `fallbacks: []` ayarlanmadıkça varsayılan geri dönüşleri devralmaya devam eder.
- `utilityModel`: oluşturulan oturum ve iş parçacığı başlıkları gibi kısa dahili görevler için isteğe bağlı aracı başına geçersiz kılma. Önce `agents.defaults.utilityModel`, ardından etkin oturum sağlayıcısının bildirdiği varsayılan küçük model kullanılır. Pano başlıkları, etkin normal oturum modeliyle bir kez daha denenir. Boş bir dize, pano başlığı oluşturmayı devre dışı bırakmadan bu aracı için alternatif yardımcı program yolunu atlar.
- `params`: `agents.defaults.models` içindeki seçili model girdisinin üzerine birleştirilen aracı başına akış parametreleri. Model kataloğunun tamamını çoğaltmadan `cacheRetention`, `temperature` veya `maxTokens` gibi aracıya özgü geçersiz kılmalar için bunu kullanın.
- `tts`: isteğe bağlı aracı başına metinden konuşmaya geçersiz kılmaları. Blok, `tts` üzerine derinlemesine birleştirilir; bu nedenle paylaşılan sağlayıcı kimlik bilgilerini ve geri dönüş politikasını `tts` içinde tutun ve burada yalnızca sağlayıcı, ses, model, stil veya otomatik mod gibi kişiliğe özgü değerleri ayarlayın.
- `skills`: isteğe bağlı aracı başına skill izin listesi. Belirtilmezse aracı, ayarlanmış olduğunda `agents.defaults.skills` değerini devralır; açık bir liste birleştirilmek yerine varsayılanların yerini alır ve `[]` hiçbir skill olmadığı anlamına gelir.
- `thinkingDefault`: isteğe bağlı aracı başına varsayılan düşünme düzeyi (`off | minimal | low | medium | high | xhigh | adaptive | max`). İleti veya oturum başına geçersiz kılma ayarlanmadığında bu aracı için `agents.defaults.thinkingDefault` değerini geçersiz kılar. Hangi değerlerin geçerli olduğunu seçili sağlayıcı/model profili belirler; Google Gemini için `adaptive`, sağlayıcının yönettiği dinamik düşünmeyi korur (Gemini 3/3.1 üzerinde `thinkingLevel` belirtilmez, Gemini 2.5 üzerinde `thinkingBudget: -1`).
- `reasoningDefault`: isteğe bağlı aracı başına varsayılan akıl yürütme görünürlüğü (`on | off | stream`). İleti veya oturum başına akıl yürütme geçersiz kılması ayarlanmadığında bu aracı için `agents.defaults.reasoningDefault` değerini geçersiz kılar.
- `fastModeDefault`: hızlı mod için isteğe bağlı aracı başına varsayılan değer (`"auto" | true | false`). İleti veya oturum başına hızlı mod geçersiz kılması ayarlanmadığında uygulanır.
- `models`: tam `provider/model` kimlikleriyle anahtarlanan isteğe bağlı aracı başına model kataloğu/çalışma zamanı geçersiz kılmaları. Aracı başına çalışma zamanı istisnaları için `models["provider/model"].agentRuntime` kullanın.
- `runtime`: isteğe bağlı aracı başına çalışma zamanı tanımlayıcısı. Aracının varsayılan olarak ACP koşum takımı oturumlarını kullanması gerektiğinde `runtime.acp` varsayılanlarıyla (`agent`, `backend`, `mode`, `cwd`) `type: "acp"` kullanın.
- `identity.avatar`: çalışma alanına göreli yol, `http(s)` URL'si veya `data:` URI'si.
- Çalışma alanına göreli yerel `identity.avatar` görüntü dosyaları 2 MB ile sınırlıdır. `http(s)` URL'leri ve `data:` URI'leri yerel dosya boyutu sınırına göre denetlenmez.
- `identity` varsayılanları türetir: `emoji` değerinden `ackReaction`, `name`/`emoji` değerlerinden `mentionPatterns`.
- `subagents.allowAgents`: açık `sessions_spawn.agentId` hedefleri için yapılandırılmış aracı kimliklerinin izin listesi (`["*"]` = yapılandırılmış herhangi bir hedef; varsayılan: yalnızca aynı aracı). Kendi kendini hedefleyen `agentId` çağrılarına izin verilmesi gerekiyorsa istekte bulunanın kimliğini ekleyin. Aracı yapılandırması silinmiş eski girdiler `sessions_spawn` tarafından reddedilir ve `agents_list` içinden çıkarılır; bunları temizlemek için `openclaw doctor --fix` çalıştırın veya bu hedefin varsayılanları devralırken oluşturulabilir kalması gerekiyorsa asgari bir `agents.entries.*` girdisi ekleyin.
- Korumalı alan devralma denetimi: istekte bulunan oturum korumalı alandaysa `sessions_spawn`, korumalı alan dışında çalışacak hedefleri reddeder.
- `subagents.requireAgentId`: true olduğunda `agentId` belirtmeyen `sessions_spawn` çağrılarını engeller (açık profil seçimini zorunlu kılar; varsayılan: false).
- `subagents.maxConcurrent`: alt aracı yürütmesindeki eşzamanlı alt aracı çalıştırmalarının azami sayısı. Varsayılan: `8`.
- `subagents.maxChildrenPerAgent`: tek bir aracı oturumunun oluşturabileceği etkin alt öğelerin azami sayısı. Varsayılan: `5`.
- `subagents.maxSpawnDepth`: alt aracı oluşturma için azami iç içe geçme derinliği (`1`-`5`). Varsayılan: `1` (iç içe geçme yok).
- `subagents.archiveAfterMinutes`: tamamlanan alt aracı durumunun arşivlenmesinden önceki süre. Varsayılan: `60`.

---

## Çok aracılı yönlendirme

Tek bir Gateway içinde birden fazla yalıtılmış aracı çalıştırın. Bkz. [Çok Aracılı](/tr/concepts/multi-agent).

```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
    ],
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
  ],
}
```

### Bağlama eşleştirme alanları

- `type` (isteğe bağlı): normal yönlendirme için `route` (türün belirtilmemesi varsayılan olarak route olur), kalıcı ACP konuşma bağlamaları için `acp`.
- `match.channel` (zorunlu)
- `match.accountId` (isteğe bağlı; `*` = herhangi bir hesap; belirtilmemesi = varsayılan hesap)
- `match.peer` (isteğe bağlı; `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (isteğe bağlı; kanala özgü)
- `acp` (isteğe bağlı; yalnızca `type: "acp"` için): `{ mode, label, cwd, backend }`

**Belirlenimci eşleştirme sırası:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (tam, eş/sunucu/ekip yok)
5. `match.accountId: "*"` (kanal genelinde)
6. Varsayılan aracı

Her katman içinde eşleşen ilk `bindings` girdisi geçerli olur.

`type: "acp"` girdileri için OpenClaw, tam konuşma kimliğine (`match.channel` + hesap + `match.peer.id`) göre çözümleme yapar ve yukarıdaki yönlendirme bağlaması katman sırasını kullanmaz.

### Aracı başına erişim profilleri

<Accordion title="Tam erişim (korumalı alan yok)">

```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="Salt okunur araçlar + çalışma alanı">

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "ro" },
        tools: {
          allow: [
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="Dosya sistemi erişimi yok (yalnızca mesajlaşma)">

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "none" },
        tools: {
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "whatsapp",
            "telegram",
            "slack",
            "discord",
            "gateway",
          ],
          deny: [
            "read",
            "write",
            "edit",
            "apply_patch",
            "exec",
            "process",
            "browser",
            "canvas",
            "nodes",
            "cron",
            "gateway",
            "image",
          ],
        },
      },
    ],
  },
}
```

</Accordion>

Öncelik ayrıntıları için [Çok Aracılı Korumalı Alan ve Araçlar](/tr/tools/multi-agent-sandbox-tools) bölümüne bakın.

---

## Oturum

```json5
{
  session: {
    scope: "per-sender",
    dmScope: "main", // main | per-peer | per-channel-peer | per-account-channel-peer
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"],
    },
    reset: {
      mode: "daily", // daily | idle
      atHour: 4,
      idleMinutes: 60,
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      direct: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 },
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 30 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    maintenance: {
      mode: "enforce", // enforce (default) | warn
      pruneAfter: "30d",
      maxEntries: 500,
      resetArchiveRetention: "30d", // duration or false
      maxDiskBytes: "500mb", // optional hard budget
      highWaterBytes: "400mb", // optional cleanup target
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // default inactivity auto-unfocus in hours (`0` disables)
      maxAgeHours: 0, // default hard max age in hours (`0` disables)
    },
    sharing: {
      readOnly: true,
      suggest: true,
      drafts: true,
    },
    mainKey: "main", // legacy (runtime always uses "main")
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="Oturum alanı ayrıntıları">

- **`scope`**: grup sohbeti bağlamları için temel oturum gruplandırma stratejisi.
  - `per-sender` (varsayılan): her gönderici, bir kanal bağlamı içinde yalıtılmış bir oturuma sahip olur.
  - `global`: bir kanal bağlamındaki tüm katılımcılar tek bir oturumu paylaşır (yalnızca paylaşılan bağlam amaçlandığında kullanın).
- **`dmScope`**: DM'lerin nasıl gruplandırıldığı.
  - `main`: tüm DM'ler ana oturumu paylaşır.
  - `per-peer`: kanallar genelinde gönderici kimliğine göre yalıtır.
  - `per-channel-peer`: kanal + gönderici başına yalıtır (çok kullanıcılı gelen kutuları için önerilir).
  - `per-account-channel-peer`: hesap + kanal + gönderici başına yalıtır (çoklu hesap kullanımı için önerilir).
- **`identityLinks`**: kanallar arası oturum paylaşımı için kanonik kimlikleri sağlayıcı önekli eşlere eşler. `/dock_discord` gibi sabitleme komutları, etkin oturumun yanıt rotasını bağlı başka bir kanal eşine geçirmek için aynı eşlemeyi kullanır; bkz. [Kanal sabitleme](/tr/concepts/channel-docking).
- **`reset`**: birincil sıfırlama politikası. `none` otomatik sıfırlamayı devre dışı bırakır ve varsayılandır; bunun yerine Compaction etkin bağlamı sınırlar. `daily`, yerel saatle `atHour` zamanında sıfırlar; `idle`, `idleMinutes` sonrasında sıfırlar. Her ikisi de yapılandırıldığında, süresi önce dolan geçerli olur. `/new` ve `/reset` her modda kullanılabilir durumda kalır. Günlük sıfırlama güncelliği, oturum satırının `sessionStartedAt` değerini; boşta kalma sıfırlaması güncelliği ise `lastInteractionAt` değerini kullanır. Heartbeat, Cron uyandırmaları, exec bildirimleri ve Gateway kayıt tutma işlemleri gibi arka plan/sistem olayı yazımları `updatedAt` değerini güncelleyebilir, ancak günlük/boşta kalan oturumları güncel tutmaz.
  - **`resetByType`**: türe göre geçersiz kılmalar (`direct`, `group`, `thread`). Doctor, eski `dm` girdilerini `direct` biçimine taşır; şema `dm` değerini reddeder.
- **`resetByChannel`**: sağlayıcı/kanal kimliğine göre anahtarlanan kanal başına sıfırlama geçersiz kılmaları. Oturumun kanalıyla eşleşen bir girdi olduğunda, söz konusu oturum için `resetByType`/`reset` üzerinde doğrudan öncelik kazanır. Yalnızca bir kanal, tür düzeyindeki politikadan farklı sıfırlama davranışı gerektirdiğinde kullanın.
- **`mainKey`**: eski alan. Çalışma zamanı, ana doğrudan sohbet bölümü için her zaman `"main"` kullanır.
- **`sendPolicy`**: `channel`, `chatType` (`direct|group|channel`, eski `dm` diğer adıyla), `keyPrefix` veya `rawKeyPrefix` ölçütüne göre eşleştirir. İlk reddetme geçerli olur.
- **`maintenance`**: oturum deposu temizleme + saklama denetimleri.
  - `mode`: `enforce` temizleme işlemini uygular ve varsayılandır; `warn` yalnızca uyarı verir.
  - `pruneAfter`: eski girdiler için yaş sınırı (varsayılan `30d`).
  - `maxEntries`: azami SQLite oturum girdisi sayısı (varsayılan `500`). Çalışma zamanı yazımları, üretim ölçeğindeki sınırlar için küçük bir yüksek su seviyesi tamponuyla toplu temizleme yapar; `openclaw sessions cleanup --enforce` sınırı hemen uygular.
  - Kısa ömürlü Gateway model çalıştırma yoklama oturumları sabit `24h` saklama süresini kullanır, ancak temizleme baskıya bağlıdır: yalnızca oturum girdisi bakımına/sınır baskısına ulaşıldığında eski ve kesin model çalıştırma yoklama satırlarını kaldırır. Yalnızca `agent:*:explicit:model-run-<uuid>` ile eşleşen kesin ve açık yoklama anahtarları uygundur; normal doğrudan, grup, ileti dizisi, Cron, hook, Heartbeat, ACP ve alt ajan oturumları bu 24h saklama süresini devralmaz. Model çalıştırma temizliği yürütüldüğünde, daha geniş kapsamlı `pruneAfter` eski girdi temizliğinden ve `maxEntries` sınırından önce yürütülür.
  - Eski `rotateBytes` mevcut şema tarafından reddedilir; `openclaw doctor --fix` bunu eski yapılandırmalardan kaldırır.
  - `resetArchiveRetention`: sıfırlanmış/silinmiş transkript arşivleri için yaşa dayalı saklama. Varsayılan olarak arşivler, disk bütçesi nedeniyle çıkarılana kadar kalır; gerçek zamana dayalı silmeyi etkinleştirmek için bir süre, açıkça devre dışı bırakmak içinse `false` ayarlayın.
  - `maxDiskBytes`: isteğe bağlı oturum dizini disk bütçesi. `warn` modunda uyarıları günlüğe kaydeder; `enforce` modunda önce en eski yapıtları/oturumları kaldırır.
  - `highWaterBytes`: bütçe temizliğinden sonraki isteğe bağlı hedef. Varsayılan olarak `maxDiskBytes` değerinin `80%` kadarıdır.
- **`threadBindings`**: ileti dizisine bağlı oturum özellikleri için genel varsayılanlar.
  - `enabled`: desteklenen kanal ileti dizisi bağlamaları için ana anahtar
  - `idleHours`: saat cinsinden varsayılan hareketsizlik sonrası otomatik odaktan çıkarma (`0` devre dışı bırakır; sağlayıcılar geçersiz kılabilir)
  - `maxAgeHours`: saat cinsinden varsayılan kesin azami yaş (`0` devre dışı bırakır; sağlayıcılar geçersiz kılabilir)
  - `spawnSessions`: `sessions_spawn` ve ACP ileti dizisi başlatmalarından ileti dizisine bağlı çalışma oturumları oluşturmak için varsayılan geçit. İleti dizisi bağlamaları etkinleştirildiğinde varsayılan olarak `true` olur; sağlayıcılar/hesaplar geçersiz kılabilir.
  - `defaultSpawnContext`: ileti dizisine bağlı başlatmalar için varsayılan yerel alt ajan bağlamı (`"fork"` veya `"isolated"`). Varsayılan olarak `"fork"` olur.
- **`sharing`**: sahiplerin ve `operator.admin` bağlantılarının hangi oturum başına işbirliği modlarını seçebileceğini denetler. Her bayrak varsayılan olarak `true` değerindedir; birini `false` olarak ayarlamak, bu seçeneği Control UI'dan kaldırır ve oluşturma sırasındaki görünürlüğün veya `session.visibility.set` değerinin bunu reddetmesini sağlar. Control UI bir oturumu taslak olarak başlatmadığı sürece yeni oturumlar `shared` olarak başlar.
  - `readOnly`: üye olmayanların izleyebildiği ancak gönderemediği, yönlendiremediği, iptal edemediği, onaylayamadığı veya oturum durumunu değiştiremediği `read-only` moduna izin verir.
  - `suggest`: `suggest` moduna izin verir. Bu aşamada `read-only` ile aynı kabul davranışını uygular; öneri kuyruğu daha sonraki bir özelliktir.
  - `drafts`: oturumu yönetici olmayanların ve sahip olmayanların oturum listelerinden ve olay yayınlarından gizleyen `draft` moduna izin verir.

Üyelik ve görünürlük değişiklikleri, sistem notları olarak oturum transkriptine yazılır. Bu denetimler, tek bir ajanı paylaşan operatörler arasında koordinasyon sağlar; kiracılar arasında bir güvenlik sınırı değildir. Çalışmanın yalıtım gerektirdiği durumlarda ayrı Gateway'ler veya ajanlar kullanın.

</Accordion>

---

## Mesajlar

```json5
{
  messages: {
    responsePrefix: "🦞", // veya "auto"
    ackReaction: "👀",
    ackReactionScope: "group-mentions", // group-mentions | group-all | direct | all | off | none
    queue: {
      mode: "steer", // steer (varsayılan) | followup | collect | interrupt
      debounceMs: 500,
      cap: 20,
      drop: "summarize", // old | new | summarize (varsayılan)
      byChannel: {
        whatsapp: "followup",
        telegram: "followup",
      },
    },
    inbound: {
      debounceMs: 2000, // 0 devre dışı bırakır
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

### Yanıt öneki

Kanal/hesap başına geçersiz kılmalar: `channels.<channel>.responsePrefix`, `channels.<channel>.accounts.<id>.responsePrefix`.

Çözümleme (en belirgin olan geçerli olur): hesap → kanal → genel. `""` devre dışı bırakır ve zincirlemeyi durdurur. `"auto"`, `[{identity.name}]` değerini türetir.

**Şablon değişkenleri:**

| Değişken          | Açıklama            | Örnek                     |
| ----------------- | ---------------------- | --------------------------- |
| `{model}`         | Kısa model adı       | `claude-opus-4-6`           |
| `{modelFull}`     | Tam model tanımlayıcısı  | `anthropic/claude-opus-4-6` |
| `{provider}`      | Sağlayıcı adı          | `anthropic`                 |
| `{thinkingLevel}` | Geçerli düşünme düzeyi | `high`, `low`, `off`        |
| `{identity.name}` | Ajan kimliği adı    | (`"auto"` ile aynı)          |

Değişkenler büyük/küçük harfe duyarsızdır. `{think}`, `{thinkingLevel}` için bir diğer addır.

### Onay tepkisi

- Varsayılan olarak etkin ajanın `identity.emoji` değerini, aksi durumda `"👀"` değerini kullanır. Devre dışı bırakmak için `""` ayarlayın.
- Kanal başına geçersiz kılmalar: `channels.<channel>.ackReaction`, `channels.<channel>.accounts.<id>.ackReaction`.
- Çözümleme sırası: hesap → kanal → `messages.ackReaction` → kimlik geri dönüşü.
- Kapsam: `group-mentions` (varsayılan), `group-all`, `direct`, `all` veya `off`/`none` (onay tepkilerini tamamen devre dışı bırakır).
- `messages.statusReactions.enabled`: Slack, Discord, Signal, Telegram ve WhatsApp'ta yaşam döngüsü durum tepkilerini etkinleştirir.
  Discord'da ayarlanmamış olması, onay tepkileri etkin olduğunda durum tepkilerini etkin tutar.
  Slack, Signal, Telegram ve WhatsApp'ta yaşam döngüsü durum tepkilerini etkinleştirmek için bunu açıkça `true` olarak ayarlayın.
  Slack, yapılandırılmış onay tepkisini sabit tutarken ilerleme durumu için varsayılan olarak yerel asistan ileti dizisi durumunu ve dönüşümlü yükleme mesajlarını kullanır.

### Kuyruk

- `mode`: bir oturum çalıştırması etkinken gelen mesajlar için kuyruk stratejisi. Varsayılan: `"steer"`.
  - `steer`: yeni istemi etkin çalıştırmaya ekler.
  - `followup`: yeni istemi etkin çalıştırma tamamlandıktan sonra çalıştırır.
  - `collect`: uyumlu mesajları toplu işler ve daha sonra birlikte çalıştırır.
  - `interrupt`: en yeni istemi başlatmadan önce etkin çalıştırmayı iptal eder.
- `debounceMs`: kuyruğa alınmış/yönlendirilmiş bir mesajı göndermeden önceki gecikme. Varsayılan: `500`.
- `cap`: çıkarma politikası uygulanmadan önceki azami kuyruk mesajı sayısı. Varsayılan: `20`.
- `drop`: sınır aşıldığındaki strateji. `"summarize"` (varsayılan), kısa özetleri koruyarak en eski girdileri çıkarır; `"old"` en eski girdileri özetler olmadan çıkarır; `"new"` en yeni öğeyi reddeder.
- `byChannel`: sağlayıcı kimliğine göre anahtarlanan kanal başına `mode` geçersiz kılmaları.
- `debounceMsByChannel`: sağlayıcı kimliğine göre anahtarlanan kanal başına `debounceMs` geçersiz kılmaları.

### Gelen ileti bekletmesi

Aynı göndericiden art arda hızla gelen yalnızca metin içeren mesajları tek bir ajan turunda toplar. Medya/ekler hemen gönderimi tetikler. Denetim komutları bekletmeyi atlar. Varsayılan `debounceMs`: `2000`.

### Diğer mesaj anahtarları

- `channels.whatsapp.responsePrefix`: giden WhatsApp yanıt öneki. Doctor, kullanımdan kaldırılmış gelen `messagePrefix` değerini yalnızca bu kanonik değer ayarlanmamışsa buraya taşır.
- `messages.visibleReplies`: doğrudan, grup ve kanal konuşmalarındaki görünür kaynak yanıtlarını denetler (`"message_tool"` görünür çıktı için `message(action=send)` gerektirir; `"automatic"` normal yanıtları önceki gibi gönderir).
- `messages.usageTemplate` / `messages.responseUsage`: özel `/usage` altbilgi şablonu ve yanıt başına varsayılan kullanım modu (`off | tokens | full`, ayrıca `tokens` için eski `on` diğer adı).
- `messages.groupChat.mentionPatterns` / `historyLimit`: grup mesajı bahsetme tetikleyicileri ve geçmiş penceresi boyutlandırması.
- `messages.suppressToolErrors`: `true` olduğunda, kullanıcıya gösterilen `⚠️` araç hatası uyarılarını bastırır (ajan hataları bağlamda görmeye devam eder ve yeniden deneyebilir). Varsayılan: `false`.

### TTS (metinden sese)

```json5
{
  tts: {
    auto: "off", // off (varsayılan) | always | inbound | tagged
    mode: "final", // final | all
    provider: "elevenlabs",
    summaryModel: "openai/gpt-5.4-mini",
    modelOverrides: { enabled: true },
    maxTextLength: 4000,
    timeoutMs: 30000,
    providers: {
      elevenlabs: {
        apiKey: "example-elevenlabs-api-key",
        baseUrl: "https://api.elevenlabs.io",
        speakerVoiceId: "voice_id",
        modelId: "eleven_multilingual_v2",
        seed: 42,
        applyTextNormalization: "auto",
        languageCode: "en",
        voiceSettings: {
          stability: 0.5,
          similarityBoost: 0.75,
          style: 0.0,
          useSpeakerBoost: true,
          speed: 1.0,
        },
      },
      microsoft: {
        speakerVoice: "en-US-MichelleNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
      },
      openai: {
        apiKey: "example-openai-api-key",
        baseUrl: "https://api.openai.com/v1",
        model: "gpt-4o-mini-tts",
        speakerVoice: "coral",
      },
    },
  },
}
```

Genel tercihler yolu makine durumudur (varsayılan
`~/.openclaw/settings/tts.json`; `OPENCLAW_TTS_PREFS` ile geçersiz kılın). Gelişmiş
çok aracılı kurulumlar, aracı başına ayrı tercih depoları için
`agents.entries.<id>.tts.prefsPath` ayarlayabilir.

- `auto` varsayılan otomatik TTS modunu denetler: `off`, `always`, `inbound` veya `tagged`. `/tts on|off` yerel tercihleri geçersiz kılabilir ve `/tts status` geçerli durumu gösterir.
- `summaryModel`, otomatik özetleme için `agents.defaults.model.primary` değerini geçersiz kılar.
- `modelOverrides` varsayılan olarak etkindir (`enabled !== false`); `modelOverrides.allowProvider` isteğe bağlıdır.
- API anahtarları, `ELEVENLABS_API_KEY`/`XI_API_KEY` ve `OPENAI_API_KEY` değerlerine geri döner.
- Paketle gelen konuşma sağlayıcılarının sahipliği pluginlere aittir. `plugins.allow` ayarlanmışsa kullanmak istediğiniz her TTS sağlayıcı pluginini ekleyin; örneğin Edge TTS için `microsoft`. Eski `edge` sağlayıcı kimliği, `microsoft` için bir diğer ad olarak kabul edilir.
- `providers.openai.baseUrl`, OpenAI TTS uç noktasını geçersiz kılar. Çözümleme sırası: yapılandırma, ardından `OPENAI_TTS_BASE_URL`, ardından `https://api.openai.com/v1`.
- `providers.openai.baseUrl`, OpenAI dışı bir uç noktaya işaret ettiğinde OpenClaw bunu OpenAI uyumlu bir TTS sunucusu olarak değerlendirir ve model/ses doğrulamasını esnetir.

---

## Konuşma

Konuşma modu için varsayılanlar (macOS/iOS/Android ve tarayıcı Denetim Kullanıcı Arayüzü).

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        speakerVoiceId: "elevenlabs_voice_id",
        voiceAliases: {
          Clawd: "EXAVITQu4vr4xnSDxMaL",
          Roger: "CwhRBWXzGAHq8TQ4Fs17",
        },
        modelId: "eleven_multilingual_v2",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
      mlx: {
        modelId: "mlx-community/Soprano-80M-bf16",
      },
      system: {},
    },
    consultThinkingLevel: "low",
    consultFastMode: true,
    speechLocale: "ru-RU",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
    realtime: {
      provider: "openai",
      providers: {
        openai: {
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
      instructions: "Sıcak bir üslupla konuşun ve yanıtları kısa tutun.",
      mode: "realtime", // realtime | stt-tts | transcription
      transport: "webrtc", // webrtc | provider-websocket | gateway-relay | managed-room
      vadThreshold: 0.5,
      silenceDurationMs: 500,
      prefixPaddingMs: 300,
      reasoningEffort: "medium",
      brain: "agent-consult", // agent-consult | direct-tools | none
    },
  },
}
```

- Birden fazla Konuşma sağlayıcısı yapılandırıldığında `talk.provider`, `talk.providers` içindeki bir anahtarla eşleşmelidir.
- Eski düz Konuşma anahtarları (`talk.voiceId`, `talk.voiceAliases`, `talk.modelId`, `talk.outputFormat`, `talk.apiKey`) yalnızca uyumluluk içindir. Kalıcı yapılandırmayı `talk.providers.<provider>` biçiminde yeniden yazmak için `openclaw doctor --fix` komutunu çalıştırın.
- Ses kimlikleri, `ELEVENLABS_VOICE_ID` veya `SAG_VOICE_ID` değerine geri döner (macOS Konuşma istemcisi davranışı).
- `providers.*.apiKey`, düz metin dizelerini veya SecretRef nesnelerini kabul eder.
- `ELEVENLABS_API_KEY` geri dönüşü yalnızca hiçbir Konuşma API anahtarı yapılandırılmadığında uygulanır.
- `providers.*.voiceAliases`, Konuşma yönergelerinin kolay anlaşılır adlar kullanmasına olanak tanır.
- `providers.mlx.modelId`, macOS yerel MLX yardımcısının kullandığı Hugging Face deposunu seçer. Belirtilmezse macOS, `mlx-community/Soprano-80M-bf16` kullanır.
- macOS MLX oynatma, mevcut olduğunda paketle gelen `openclaw-mlx-tts` yardımcısı veya `PATH` üzerindeki bir çalıştırılabilir dosya üzerinden yürütülür; `OPENCLAW_MLX_TTS_BIN`, geliştirme amacıyla yardımcı yolunu geçersiz kılar.
- `consultThinkingLevel`, Denetim Kullanıcı Arayüzü Konuşma gerçek zamanlı `openclaw_agent_consult` çağrılarının arkasındaki tam OpenClaw aracı çalıştırmasının düşünme düzeyini denetler. Normal oturum/model davranışını korumak için ayarlamayın.
- `consultFastMode`, oturumun normal hızlı mod ayarını değiştirmeden Denetim Kullanıcı Arayüzü Konuşma gerçek zamanlı danışmaları için tek seferlik bir hızlı mod geçersiz kılması ayarlar.
- `speechLocale`, Android, iOS ve macOS Konuşma konuşma tanımanın kullandığı BCP 47 yerel ayar kimliğini belirler. Android ayrıca gerçek zamanlı giriş transkripsiyonuna yön vermek için bunun dil bileşenini kullanır. Cihazın varsayılanını kullanmak için ayarlamayın.
- `silenceTimeoutMs`, Konuşma modunun kullanıcı sessizliğinden sonra transkripti göndermeden önce ne kadar bekleyeceğini denetler. Ayarlanmaması, platformun varsayılan duraklama aralığını korur (`700 ms on macOS and Android, 900 ms on iOS`).
- `realtime.instructions`, sağlayıcıya yönelik sistem talimatlarını OpenClaw'ın yerleşik gerçek zamanlı istemine ekler; böylece varsayılan `openclaw_agent_consult` yönlendirmesi kaybedilmeden ses stili yapılandırılabilir.
- `realtime.vadThreshold`, sağlayıcının ses etkinliği eşiğini `0` (en hassas) ile `1` (en az hassas) arasında ayarlar. Ayarlanmaması, sağlayıcının varsayılanını korur.
- `realtime.silenceDurationMs`, sağlayıcı gerçek zamanlı bir kullanıcı sırasını kesinleştirmeden önceki pozitif tam sayı sessizlik aralığını ayarlar. Ayarlanmaması, sağlayıcının varsayılanını korur.
- `realtime.prefixPaddingMs`, algılanan konuşma başlamadan önce tutulan negatif olmayan tam sayı ses miktarını ayarlar. Ayarlanmaması, sağlayıcının varsayılanını korur.
- `realtime.reasoningEffort`, gerçek zamanlı oturumlar için sağlayıcıya özgü akıl yürütme düzeyini ayarlar. Ayarlanmaması, sağlayıcının varsayılanını korur.
- `realtime.consultRouting`: `"provider-direct"` (varsayılan), gerçek zamanlı sağlayıcı `openclaw_agent_consult` olmadan nihai bir kullanıcı transkripti ürettiğinde doğrudan sağlayıcı yanıtlarını korur. Bunun yerine `"force-agent-consult"`, kesinleştirilmiş isteği OpenClaw üzerinden yönlendirir.

---

## İlgili

- [Yapılandırma referansı](/tr/gateway/configuration-reference) — diğer tüm yapılandırma anahtarları
- [Yapılandırma](/tr/gateway/configuration) — yaygın görevler ve hızlı kurulum
- [Yapılandırma örnekleri](/tr/gateway/configuration-examples)
