---
read_when:
    - '`tools.*` politikası, izin listeleri veya deneysel özellikler yapılandırılıyor'
    - Özel sağlayıcıları kaydetme veya temel URL'leri geçersiz kılma
    - OpenAI uyumlu, kendi sunucunuzda barındırılan uç noktaları ayarlama
sidebarTitle: Tools and custom providers
summary: Araç yapılandırması (politika, deneysel geçişler, sağlayıcı destekli araçlar) ve özel sağlayıcı/temel URL kurulumu
title: Yapılandırma — araçlar ve özel sağlayıcılar
x-i18n:
    generated_at: "2026-07-26T22:45:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2010a2e48e8f4c8d0049e5c707bb8286e291a92312baac94301a7b5a674583c1
    source_path: gateway/config-tools.md
    workflow: 16
---

`tools.*` yapılandırma anahtarları ve özel sağlayıcı / temel URL kurulumu. Aracılar, kanallar ve diğer üst düzey yapılandırma anahtarları için [Yapılandırma referansı](/tr/gateway/configuration-reference) bölümüne bakın.

## Araçlar

### Araç profilleri

`tools.profile`, `tools.allow`/`tools.deny` öncesinde temel bir izin verilenler listesi ayarlar:

<Note>
Yerel ilk katılım, ayarlanmamışsa yeni yerel yapılandırmalar için varsayılan olarak `tools.profile: "coding"` değerini kullanır (mevcut açık profiller korunur).
</Note>

| Profil     | İçerik                                                                                                                                                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | Yalnızca `session_status`                                                                                                                                                                                                                                   |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`, `image`, `image_generate`, `music_generate`, `video_generate`                |
| `messaging` | `group:messaging`, `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `ask_user` |
| `full`      | Kısıtlama yok (ayarlanmamış olmasıyla aynı)                                                                                                                                                                                                                          |

`coding` ve `messaging`, `bundle-mcp` öğesine de örtük olarak izin verir (yapılandırılmış MCP sunucuları).

### Araç grupları

| Grup              | Araçlar                                                                                                                                                                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash`, `exec` için takma ad olarak kabul edilir)                                                                                                                                                                        |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                                                                                                                                                 |
| `group:sessions`   | `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `spawn_task`, `dismiss_task` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                                                                                                                                                          |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                                                                                                                                                  |
| `group:ui`         | `browser`, `screen`, `terminal`, `canvas`, `show_widget`                                                                                                                                                                                               |
| `group:automation` | `heartbeat_respond`, `cron`, `gateway`                                                                                                                                                                                                                 |
| `group:messaging`  | `message`                                                                                                                                                                                                                                              |
| `group:nodes`      | `nodes`, `computer`                                                                                                                                                                                                                                    |
| `group:agents`     | `agents_list`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`                                                                                                                                                   |
| `group:media`      | `image`, `image_generate`, `music_generate`, `video_generate`, `tts`                                                                                                                                                                                   |
| `group:openclaw`   | `read`/`write`/`edit`/`apply_patch`/`exec`/`process`/`canvas` hariç yukarıdaki tüm yerleşik araçlar (Plugin araçlarını hariç tutar)                                                                                                                                  |
| `group:plugins`    | `bundle-mcp` aracılığıyla sunulan yapılandırılmış MCP sunucuları dahil, yüklenmiş Plugin'lerin sahip olduğu araçlar                                                                                                                                                           |

`spawn_task`, bir kodlama aracısının onaylanmış takip çalışmasını başlatmadan önermesini sağlar. Control UI, başlığı ve özeti eyleme geçirilebilir bir çip olarak gösterir; Gateway destekli bir TUI, eşdeğer bir etkileşimli istem gösterir. Bunlardan herhangi birinin kabul edilmesi yeni bir yönetilen çalışma ağacı oturumu oluşturur ve mevcut tur devam ederken tam istemi oraya gönderir. `dismiss_task`, `spawn_task` tarafından döndürülen geçici `task_id` aracılığıyla hâlâ beklemede olan bir öneriyi geri çeker.

Araçlar yalnızca başlatan operatör yüzeyi Gateway görev önerisi olaylarını alıp işleyebildiğinde sunulur. Kanal oturumları ile yerel/gömülü TUI oturumları bunları almaz; kanal aktarımlarının bu akışı güvenli biçimde sunabilmesi için taşınabilir ve türü belirlenmiş bir görev eylemine ihtiyacı vardır. Öneriler işleme özeldir ve Gateway yeniden başlatıldığında kaybolur. Her iki araç da `coding` profilinde ve `group:sessions` içinde kalır; böylece yüzey bunları desteklediğinde normal `tools.allow` ve `tools.deny` ilkesi bunları otomatik olarak yapılandırır.

### Korumalı alan araç ilkesi içindeki MCP ve Plugin araçları

Yapılandırılmış MCP sunucuları, `bundle-mcp` Plugin kimliği altında Plugin'e ait araçlar olarak sunulur. Normal araç profilleri bunlara izin verebilir, ancak `tools.sandbox.tools`, korumalı alan oturumları için ek bir geçittir. Korumalı alan modu `"all"` veya `"non-main"` ise MCP/Plugin araçlarının görünür olması gerektiğinde korumalı alan araç izin verilenler listesine şu girdilerden birini ekleyin:

- `mcp.servers` kaynağındaki OpenClaw tarafından yönetilen MCP sunucuları için `bundle-mcp`
- belirli bir yerel Plugin için Plugin kimliği
- yüklenmiş ve Plugin'e ait tüm araçlar için `group:plugins`
- yalnızca tek bir sunucu istediğinizde `outlook__send_mail` veya `outlook__*` gibi tam MCP sunucu araç adları ya da sunucu glob kalıpları

Sunucu glob kalıpları, ham `mcp.servers` anahtarı olmak zorunda olmayan, sağlayıcı açısından güvenli MCP sunucu ön ekini kullanır. `[A-Za-z0-9_-]` dışındaki karakterler `-` olur, bir harfle başlamayan adlara `mcp-` ön eki eklenir ve uzun ya da yinelenen ön ekler kısaltılabilir veya son ek alabilir; örneğin `mcp.servers["Outlook Graph"]`, `outlook-graph__*` benzeri bir glob kalıbı kullanır.

```json5
{
  agents: { defaults: { sandbox: { mode: "all" } } },
  mcp: {
    servers: {
      outlook: { command: "node", args: ["./outlook-mcp.js"] },
    },
  },
  tools: {
    sandbox: {
      tools: {
        alsoAllow: ["web_search", "web_fetch", "memory_search", "memory_get", "bundle-mcp"],
      },
    },
  },
}
```

Bu korumalı alan katmanı girdisi olmadan MCP sunucusu yine başarıyla yüklenebilir, ancak araçları sağlayıcı isteğinden önce filtrelenir. `mcp.servers` içindeki OpenClaw tarafından yönetilen sunucularda bu durumu yakalamak için `openclaw doctor` kullanın. Paketlenmiş Plugin manifestlerinden veya Claude `.mcp.json` üzerinden yüklenen MCP sunucuları aynı korumalı alan geçidini kullanır, ancak bu tanılama henüz bu kaynakları listelemez; araçları korumalı alan turlarında kaybolursa aynı izin verilenler listesi girdilerini kullanın.

### `tools.codeMode`

`tools.codeMode`, genel OpenClaw kod modu yüzeyini etkinleştirir. Araçların bulunduğu
bir çalıştırma için etkinleştirildiğinde normal OpenClaw araçları, korumalı alan içindeki `tools.*`
katalog köprüsünün arkasına taşınır ve MCP araçları oluşturulan `MCP`
ad alanı üzerinden kullanılabilir. Model normalde `exec` ve `wait` öğelerini görür; yapılandırılmış sonuçları
yalnızca JSON destekleyen köprüden geçemeyen `computer` gibi araçlar doğrudan kalır.

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

Kısa gösterim de kabul edilir:

```json5
{
  tools: { codeMode: true },
}
```

MCP bildirimleri, kod modundaki salt okunur sanal API dosyası yüzeyi üzerinden
sunulur. Konuk kod, `MCP.<server>.<tool>()` çağrısından önce TypeScript tarzı
imzaları incelemek için `API.list("mcp")` ve
`API.read("mcp/<server>.d.ts")` çağrılarını yapabilir. Çalışma zamanı sözleşmesi,
sınırlar ve hata ayıklama adımları için [Kod Modu](/tools/code-mode) bölümüne bakın.

### `tools.allow` / `tools.deny`

Genel araç izin verme/reddetme ilkesi (reddetme önceliklidir). Büyük/küçük harfe duyarsızdır ve `*` joker karakterlerini destekler. Docker korumalı alanı kapalıyken bile uygulanır.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

`write` ve `apply_patch` ayrı araç kimlikleridir. `allow: ["write"]`, uyumlu modeller için `apply_patch` öğesini de etkinleştirir, ancak `deny: ["write"]`, `apply_patch` öğesini reddetmez. Tüm dosya değişikliklerini engellemek için `group:fs` öğesini reddedin veya değişiklik yapan her aracı açıkça listeleyin:

```json5
{
  tools: { deny: ["write", "edit", "apply_patch"] },
}
```

<Note>
`allow` ve `alsoAllow` aynı kapsamda (`tools`, `tools.byProvider.<id>`, `agents.entries.*.tools`) birlikte ayarlanamaz; yapılandırma doğrulaması bunu reddeder. `alsoAllow` girdilerini `allow` içine birleştirin veya `allow` öğesini kaldırıp bunun yerine `profile` + `alsoAllow` kullanın.
</Note>

### `tools.byProvider`

Belirli sağlayıcılar veya modeller için araçları daha da kısıtlayın. Sıra: temel profil → sağlayıcı profili → izin verme/reddetme.

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.toolsBySender`

Araçları, geçerli turdaki isteğin kaynak isteyicisi için kısıtlar. Bu, kanal erişim denetiminin üzerine eklenen derinlemesine savunmadır; gönderen değerleri mesaj metninden değil, kanal bağdaştırıcısından gelmelidir. Model istemindeki diğer içeriklerin kimliğini doğrulamaz; bkz. [İsteyici kapsamlı denetimler ve istem bağlamı](/tr/gateway/security#requester-scoped-controls-and-prompt-context).

```json5
{
  tools: {
    toolsBySender: {
      "channel:discord:1234567890123": { alsoAllow: ["group:fs"] },
      "id:guest-user-id": { deny: ["group:runtime", "group:fs"] },
      "*": { deny: ["exec", "process", "write", "edit", "apply_patch"] },
    },
  },
}
```

Anahtarlar açık önekler kullanır: `channel:<channelId>:<senderId>`, `id:<senderId>`, `e164:<phone>`, `username:<handle>`, `name:<displayName>` veya `"*"`. Kanal kimlikleri standart OpenClaw kimlikleridir; `teams` gibi takma adlar `msteams` biçimine normalleştirilir. Eski, öneksiz anahtarlar yalnızca `id:` olarak kabul edilir. Eşleştirme sırası kanal+kimlik, kimlik, e164, kullanıcı adı, ad ve ardından joker karakterdir.

Ajan başına `agents.entries.*.tools.toolsBySender`, eşleştiğinde boş bir `{}` politikasıyla bile genel gönderen eşleşmesini geçersiz kılar.

### `tools.elevated`

Korumalı alan dışındaki yükseltilmiş exec erişimini denetler:

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123", "987654321098765432"],
      },
    },
  },
}
```

- Ajan başına geçersiz kılma (`agents.entries.*.tools.elevated`) yalnızca daha fazla kısıtlama getirebilir.
- `/elevated on|off|ask|full` durumu oturum başına saklar; satır içi yönergeler tek bir mesaja uygulanır.
- Yükseltilmiş `exec`, korumalı alanı atlar ve yapılandırılmış kaçış yolunu kullanır (varsayılan olarak `gateway`; exec hedefi `node` olduğunda ise `node`).

### `tools.exec`

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      approvalRunningNoticeMs: 10000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      commandHighlighting: false,
      applyPatch: {
        enabled: true,
        allowModels: ["gpt-5.6-sol"],
      },
    },
  },
}
```

Gösterilen değerler, `applyPatch.allowModels` dışında varsayılanlardır (varsayılan olarak boş/ayarlanmamış; bu, herhangi bir uyumlu modelin `apply_patch` kullanabileceği anlamına gelir). `approvalRunningNoticeMs`, onay destekli exec uzun sürdüğünde çalışıyor bildirimi yayınlar; `0` bunu devre dışı bırakır.

### `tools.loopDetection`

Araç döngüsü güvenlik denetimleri **varsayılan olarak devre dışıdır**. Algılamayı etkinleştirmek için `enabled: true` ayarlayın. Ayarlar genel olarak `tools.loopDetection` içinde tanımlanabilir ve ajan başına `agents.entries.*.tools.loopDetection` konumunda geçersiz kılınabilir.

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
    },
  },
}
```

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // veya BRAVE_API_KEY ortam değişkeni (Brave sağlayıcısı)
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // isteğe bağlı; otomatik algılama için atlayın
        maxChars: 20000,
        maxCharsCap: 20000,
        maxResponseBytes: 750000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true,
        userAgent: "custom-ua",
      },
    },
  },
}
```

Gösterilen değerler, `provider` ve `userAgent` dışında varsayılanlardır. `maxResponseBytes` değeri 32000–10000000 aralığıyla sınırlar; `maxChars` değeri `maxCharsCap` ile sınırlar (daha büyük yanıtlara izin vermek için `maxCharsCap` değerini artırın).

### `tools.media`

Gelen medya anlamlandırmasını (görüntü/ses/video) yapılandırır:

```json5
{
  tools: {
    media: {
      concurrency: 2,
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          capabilities: ["audio"],
        },
        { provider: "ollama", model: "gemma4:26b", capabilities: ["image"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["video"] },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-mini-transcribe" },
      image: { enabled: true, preferredModel: "ollama/gemma4:26b" },
      video: { enabled: true },
    },
  },
}
```

`tools.media.models`, yapılandırılmış tek model listesidir. Her girdi, işlediği yetenekleri bildirir. İsteğe bağlı `preferredModel` seçicisi; `provider/model`, bir model kimliği, sağlayıcı varsayılanı girdileri için `provider:<id>` veya `cli:command` kabul eder; eşleşen girdiler ilgili yeteneğin geri dönüş sırasının başına taşınır. Yetenek başına istemler, sınırlar, istek ayarları, kapsam, ek politikası ve ses dökümü yankısı; yapılandırılmış ve otomatik algılanan modeller için varsayılan olarak kalır. Bir model girdisi, modele özgü alanları geçersiz kılabilir.

<AccordionGroup>
  <Accordion title="Medya modeli girdisi alanları">
    **Sağlayıcı girdisi** (`type: "provider"` veya atlanmış):

    - `provider`: API sağlayıcı kimliği (`openai`, `anthropic`, `google`/`gemini`, `groq` vb.)
    - `model`: model kimliği geçersiz kılması
    - `profile` / `preferredProfile`: `auth-profiles.json` profil seçimi

    **CLI girdisi** (`type: "cli"`):

    - `command`: çalıştırılacak yürütülebilir dosya
    - `args`: şablonlu bağımsız değişkenler (`{{AttachmentPath}}`, `{{AttachmentUrl}}`, `{{AttachmentContentType}}`, `{{AttachmentDir}}`, `{{AttachmentIndex}}`, `{{Prompt}}`, `{{MaxChars}}` vb. destekler; `openclaw doctor --fix`, kullanım dışı bırakılmış `{input}` yer tutucularını `{{AttachmentPath}}` biçimine geçirir). Eski `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` ve `{{MediaDir}}` takma adları uyumluluk süreleri boyunca kullanılabilir durumda kalır ancak kullanımları önerilmez.

    **Ortak alanlar:**

    - `capabilities`: `image`, `audio` ve `video` değerlerinden birini veya daha fazlasını içeren liste.
    - `prompt`, `maxChars`, `maxBytes`, `timeoutSeconds`, `language`: girdi başına geçersiz kılmalar.
    - Eşleşen görüntü modeli `timeoutSeconds` girdileri, ajan açık `image` aracını çağırdığında da uygulanır. Görüntü anlamlandırmasında bu zaman aşımı isteğin kendisine uygulanır ve önceki hazırlık çalışmaları nedeniyle azaltılmaz.
    - Hatalarda sıradaki girdiye geri dönülür.

    Sağlayıcı kimlik doğrulaması standart sırayı izler: `auth-profiles.json` → ortam değişkenleri → `models.providers.*.apiKey`.

  </Accordion>
</AccordionGroup>

### `tools.agentToAgent`

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `tools.sessions`

Oturum araçlarının (`sessions_list`, `sessions_history`, `sessions_send`) hangi oturumları hedefleyebileceğini denetler.

Varsayılan: `tree` (geçerli oturum + alt ajanlar gibi onun başlattığı oturumlar ve aynı ajan için ortamdan
izlenen grup oturumları).

```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Görünürlük kapsamları">
    - `self`: yalnızca geçerli oturum anahtarı.
    - `tree`: geçerli oturum + geçerli oturumun başlattığı oturumlar (alt ajanlar). Okuma işlemleri için, geçerli oturumun ortamsal grup farkındalığı aracılığıyla izlediği aynı ajana ait grup oturumlarını da içerir.
    - `agent`: geçerli ajan kimliğine ait herhangi bir oturum (aynı ajan kimliği altında gönderen başına oturumlar çalıştırıyorsanız diğer kullanıcıları içerebilir).
    - `all`: herhangi bir oturum. Ajanlar arası hedefleme yine de `tools.agentToAgent` gerektirir.
    - Korumalı alan sınırlaması: geçerli oturum korumalı alandaysa ve `agents.defaults.sandbox.sessionToolsVisibility="spawned"` (varsayılan) ise, `tools.sessions.visibility="all"` olsa bile görünürlük `tree` olarak zorlanır.
    - `all` olmadığında, `sessions_list` etkili modu açıklayan ve bazı oturumların
      geçerli kapsam dışında bırakılabileceği konusunda uyaran kısa bir `visibility` alanı içerir.

  </Accordion>
</AccordionGroup>

Varsayılan `session.dmScope: "main"` ile bir gruptaki insan etkinliği, aynı ajana ait bu grup
oturumunu ajanın ana oturumuna ortamsal olarak görünür kılar. Çok kullanıcılı bir kurulumda `"main"` ayrıca
kullanıcılar arasında tek bir DM oturumu paylaşır; böylece buraya yönlendirilen her kullanıcı, oturum belleği
`memory_search` üzerinden de dahil olmak üzere ortamsal olarak izlenen gruplardan okuma yapabilir. DM yalıtımı için eş başına `dmScope` kullanın veya
ortamsal izlenen oturum okumalarını devre dışı bırakmak için `tools.sessions.visibility: "self"` ayarlayın.

### `tools.sessions_spawn`

`sessions_spawn` için satır içi ek desteğini denetler.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // katılım gerektirir: satır içi dosya eklerine izin vermek için true olarak ayarlayın
        maxTotalBytes: 5242880, // tüm dosyalarda toplam 5 MB
        maxFiles: 50,
        maxFileBytes: 1048576, // dosya başına 1 MB
        retainOnSessionKeep: false, // cleanup="keep" olduğunda ekleri koru
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Ek notları">
    - Ekler `enabled: true` gerektirir.
    - Alt ajan ekleri, alt çalışma alanında `.openclaw/attachments/<uuid>/` konumuna bir `.manifest.json` ile yerleştirilir.
    - ACP ekleri yalnızca görüntü olabilir ve aynı dosya sayısı, dosya başına bayt ve toplam bayt sınırları geçildikten sonra ACP çalışma zamanına satır içinde iletilir.
    - Ek içeriği, döküm kalıcılığından otomatik olarak sansürlenir.
    - Base64 girdileri, katı alfabe/doldurma denetimleri ve kod çözme öncesi boyut korumasıyla doğrulanır.
    - Alt ajan eklerinin dosya izinleri, dizinler için `0700` ve dosyalar için `0600` değeridir.
    - Alt ajan temizliği `cleanup` politikasını izler: `delete` ekleri her zaman kaldırır; `keep` bunları yalnızca `retainOnSessionKeep: true` olduğunda korur.

  </Accordion>
</AccordionGroup>

<a id="toolsexperimental"></a>

### `tools.experimental`

Deneysel yerleşik araç bayrakları. Katı ajanlı GPT-5 otomatik etkinleştirme kuralı uygulanmadıkça varsayılan olarak kapalıdır.

```json5
{
  tools: {
    experimental: {
      planTool: true, // deneysel update_plan aracını etkinleştir
    },
  },
}
```

- `planTool`: önemsiz olmayan çok adımlı çalışmaların izlenmesi için yapılandırılmış `update_plan` aracını etkinleştirir.
- Varsayılan: Bir GPT-5 ailesi model kimliğine karşı `openai` sağlayıcı çalıştırması için `agents.defaults.embeddedAgent.executionContract` (veya ajan başına geçersiz kılma) `"strict-agentic"` olarak ayarlanmadıkça `false` değeridir (Codex kimlik doğrulaması/model yönlendirmesi `openai` sağlayıcısı altında bulunduğundan bu, OpenAI Codex CLI çalıştırmalarını da kapsar). Aracı bu kapsamın dışında zorla açmak için `true`, katı ajanlı GPT-5 çalıştırmalarında bile kapalı tutmak için `false` ayarlayın.
- Etkinleştirildiğinde sistem istemi ayrıca kullanım rehberliği ekler; böylece model aracı yalnızca kapsamlı çalışmalar için kullanır ve en fazla bir adımı `in_progress` durumunda tutar.

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        allowAgents: ["research"],
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 8,
        runTimeoutSeconds: 900,
        announceTimeoutMs: 120000,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model`: oluşturulan alt ajanlar için varsayılan model. Belirtilmezse alt ajanlar çağıranın modelini devralır.
- `allowAgents`: istekte bulunan ajan kendi `subagents.allowAgents` değerini ayarlamadığında `sessions_spawn` için yapılandırılmış hedef ajan kimliklerinin varsayılan izin listesi (`["*"]` = yapılandırılmış herhangi bir hedef; varsayılan: yalnızca aynı ajan). Ajan yapılandırması silinmiş eski girdiler `sessions_spawn` tarafından reddedilir ve `agents_list` kapsamına alınmaz; bunları temizlemek için `openclaw doctor --fix` komutunu çalıştırın.
- `maxConcurrent`: eşzamanlı azami alt ajan çalıştırma sayısı. Varsayılan: `8`.
- `runTimeoutSeconds`: çağıran kendi geçersiz kılma değerini iletmediğinde `sessions_spawn` için zaman aşımı (saniye). Varsayılan: `0` (zaman aşımı yok); yukarıda gösterilen `900` yerleşik varsayılan değil, yaygın bir isteğe bağlı değerdir.
- `announceTimeoutMs`: Gateway `agent` duyuru teslimi denemeleri için çağrı başına zaman aşımı (milisaniye). Varsayılan: `120000`. Geçici yeniden denemeler, toplam duyuru bekleme süresini yapılandırılmış tek bir zaman aşımından daha uzun hâle getirebilir.
- `archiveAfterMinutes`: bir alt ajan oturumu tamamlandıktan sonra otomatik olarak arşivlenmesine kadar geçen dakika. Varsayılan: `60`; `0` otomatik arşivlemeyi devre dışı bırakır.
- Alt ajan başına araç politikası: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## Özel sağlayıcılar ve temel URL'ler

Sağlayıcı Plugin'leri kendi model kataloğu satırlarını yayımlar. Özel sağlayıcıları yapılandırmadaki `models.providers` veya `~/.openclaw/agents/<agentId>/agent/models.json` aracılığıyla ekleyin.

Özel/yerel bir sağlayıcının `baseUrl` değerini yapılandırmak, model HTTP istekleri için dar kapsamlı ağ güveni kararıdır: OpenClaw, ayrı bir yapılandırma seçeneği eklemeden veya diğer özel kaynaklara güvenmeden, korumalı getirme yolu üzerinden yalnızca tam olarak bu `scheme://host:port` kaynağına izin verir.

```json5
{
  models: {
    mode: "merge", // birleştir (varsayılan) | değiştir
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions", // openai-completions | openai-responses | anthropic-messages | google-generative-ai | vb.
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            contextTokens: 96000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Kimlik doğrulama ve birleştirme önceliği">
    - Özel kimlik doğrulama gereksinimleri için `authHeader: true` + `headers` kullanın.
    - Ajan yapılandırma kökünü `OPENCLAW_AGENT_DIR` ile geçersiz kılın.
    - Eşleşen sağlayıcı kimlikleri için birleştirme önceliği:
      - Boş olmayan ajan `models.json` `baseUrl` değerleri önceliklidir.
      - Boş olmayan ajan `apiKey` değerleri yalnızca ilgili sağlayıcı geçerli yapılandırma/kimlik doğrulama profili bağlamında SecretRef tarafından yönetilmiyorsa önceliklidir.
      - SecretRef tarafından yönetilen sağlayıcı `apiKey` değerleri, çözümlenmiş gizli bilgileri kalıcı hâle getirmek yerine kaynak işaretçilerinden (`ENV_VAR_NAME` ortam referansları, `secretref-managed` dosya/çalıştırma referansları için) yenilenir.
      - SecretRef tarafından yönetilen sağlayıcı üst bilgi değerleri kaynak işaretçilerinden (`secretref-env:ENV_VAR_NAME` ortam referansları, `secretref-managed` dosya/çalıştırma referansları için) yenilenir.
      - Boş veya eksik ajan `apiKey`/`baseUrl` değerleri, yapılandırmadaki `models.providers` değerine geri döner.
      - Eşleşen model `contextWindow`/`maxTokens`: mevcut ve geçerli olduğunda (pozitif sonlu bir sayı) açık yapılandırma değeri önceliklidir; aksi takdirde örtük/oluşturulmuş katalog değeri kullanılır.
      - Eşleşen model `contextTokens` aynı açık-değer-öncelikli-aksi-hâlde-örtük kuralını izler; yerel model meta verilerini değiştirmeden etkin bağlamı sınırlamak için bunu kullanın.
      - Sağlayıcı Plugin'i katalogları, ajanın Plugin durumu altında oluşturulmuş ve Plugin'in sahip olduğu katalog parçaları olarak saklanır.
      - Yapılandırmanın `models.json` değerini tamamen yeniden yazmasını ve Plugin'in sahip olduğu katalog parçalarının birleştirilmesini atlamasını istediğinizde `models.mode: "replace"` kullanın.
      - İşaretçi kalıcılığı kaynak yetkilidir: işaretçiler, çözümlenmiş çalışma zamanı gizli bilgi değerlerinden değil, etkin kaynak yapılandırma anlık görüntüsünden (çözümleme öncesi) yazılır.

  </Accordion>
</AccordionGroup>

### Sağlayıcı alanı ayrıntıları

<AccordionGroup>
  <Accordion title="Üst düzey katalog">
    - `models.mode`: sağlayıcı kataloğu davranışı (`merge` veya `replace`).
    - `models.providers`: sağlayıcı kimliğine göre anahtarlanmış özel sağlayıcı eşlemesi.
      - Güvenli düzenlemeler: eklemeli güncellemeler için `openclaw config set models.providers.<id> '<json>' --strict-json --merge` veya `openclaw config set models.providers.<id>.models '<json-array>' --strict-json --merge` kullanın. `config set`, `--replace` iletmediğiniz sürece yıkıcı değiştirmeleri reddeder.

  </Accordion>
  <Accordion title="Sağlayıcı bağlantısı ve kimlik doğrulama">
    - `models.providers.*.api`: istek bağdaştırıcısı (`openai-completions`, `openai-responses`, `openai-chatgpt-responses`, `anthropic-messages`, `google-generative-ai`, `google-vertex`, `github-copilot`, `bedrock-converse-stream`, `ollama`, `azure-openai-responses`). MLX, vLLM, SGLang ve çoğu OpenAI uyumlu yerel sunucu gibi kendi kendine barındırılan `/v1/chat/completions` arka uçları için `openai-completions` kullanın. `baseUrl` içeren ancak `api` içermeyen özel bir sağlayıcı varsayılan olarak `openai-completions` kullanır; `openai-responses` değerini yalnızca arka uç `/v1/responses` desteklediğinde ayarlayın.
    - `models.providers.*.apiKey`: sağlayıcı kimlik bilgisi (SecretRef/ortam değişkeni ikamesini tercih edin).
    - `models.providers.*.auth`: kimlik doğrulama stratejisi (`api-key`, `token`, `oauth`, `aws-sdk`).
    - `models.providers.*.contextWindow`: model girdisi `contextWindow` ayarlamadığında bu sağlayıcı altındaki modeller için varsayılan yerel bağlam penceresi.
    - `models.providers.*.contextTokens`: model girdisi `contextTokens` ayarlamadığında bu sağlayıcı altındaki modeller için varsayılan etkin çalışma zamanı bağlam sınırı.
    - `models.providers.*.maxTokens`: model girdisi `maxTokens` ayarlamadığında bu sağlayıcı altındaki modeller için varsayılan çıktı belirteci sınırı.
    - `models.providers.*.timeoutSeconds`: bağlantı, üst bilgiler, gövde ve toplam istek iptal işlemleri dâhil olmak üzere sağlayıcı başına isteğe bağlı model HTTP isteği zaman aşımı (saniye).
    - `models.providers.*.injectNumCtxForOpenAICompat`: Ollama + `openai-completions` için isteklere `options.num_ctx` ekler (varsayılan: `true`).
    - `models.providers.*.authHeader`: gerektiğinde kimlik bilgilerinin `Authorization` üst bilgisinde taşınmasını zorunlu kılar.
    - `models.providers.*.baseUrl`: yukarı akış API temel URL'si.
    - `models.providers.*.headers`: proxy/kiracı yönlendirmesi için ek statik üst bilgiler.

  </Accordion>
  <Accordion title="İstek aktarımı geçersiz kılmaları">
    `models.providers.*.request`: model sağlayıcısı HTTP istekleri için aktarım geçersiz kılmaları.

    - `request.headers`: ek üst bilgiler (sağlayıcı varsayılanlarıyla birleştirilir). Değerler SecretRef kabul eder.
    - `request.auth`: kimlik doğrulama stratejisi geçersiz kılması. Kipler: `"provider-default"` (sağlayıcının yerleşik kimlik doğrulamasını kullanır), `"authorization-bearer"` (`token` ile), `"header"` (`headerName`, `value` ve isteğe bağlı `prefix` ile).
    - `request.proxy`: HTTP proxy geçersiz kılması. Kipler: `"env-proxy"` (`HTTP_PROXY`/`HTTPS_PROXY` ortam değişkenlerini kullanır), `"explicit-proxy"` (`url` ile). Her iki kip de isteğe bağlı bir `tls` alt nesnesi kabul eder.
    - `request.tls`: doğrudan bağlantılar için TLS geçersiz kılması. Alanlar: `ca`, `cert`, `key`, `passphrase` (tümü SecretRef kabul eder), `serverName`, `insecureSkipVerify`.
    - `request.allowPrivateNetwork`: `true` olduğunda, model sağlayıcısı HTTP isteklerinin sağlayıcı HTTP getirme koruması üzerinden özel, CGNAT veya benzeri aralıklara erişmesine izin verir. Özel/yerel sağlayıcı temel URL'leri, açıkça kabul edilmedikçe engellenmeye devam eden meta veri/bağlantı-yerel kaynaklar hariç, zaten tam olarak yapılandırılmış kaynağa güvenir. Tam kaynak güvenini devre dışı bırakmak için bunu `false` olarak ayarlayın. WebSocket, üst bilgiler/TLS için aynı `request` değerini kullanır ancak bu getirme SSRF korumasını kullanmaz. Varsayılan `false`.

  </Accordion>
  <Accordion title="Model kataloğu girdileri">
    - `models.providers.*.models`: açık sağlayıcı model kataloğu girdileri.
    - `models.providers.*.models.*.input`: model giriş kipleri. Yalnızca metin modelleri için `["text"]`, yerel görüntü/görme modelleri için `["text", "image"]` kullanın. Görüntü ekleri, yalnızca seçilen model görüntü destekli olarak işaretlendiğinde ajan turlarına eklenir.
    - `models.providers.*.models.*.contextWindow`: yerel model bağlam penceresi meta verileri. Bu, ilgili model için sağlayıcı düzeyindeki `contextWindow` değerini geçersiz kılar.
    - `models.providers.*.models.*.contextTokens`: isteğe bağlı çalışma zamanı bağlam sınırı. Bu, sağlayıcı düzeyindeki `contextTokens` değerini geçersiz kılar; modelin yerel `contextWindow` değerinden daha küçük bir etkin bağlam bütçesi istediğinizde bunu kullanın; değerler farklı olduğunda `openclaw models list` her ikisini de gösterir.

    #### Özel sağlayıcı yetenek bildirimleri

    Sağlayıcı katalogları, paketlenmiş ve katalog tarafından bilinen model yolları için `compat` değerinin sahibidir. Bu bayrakları yapılandırmaya kopyalamayın: yapılandırılmış `api` ve `baseUrl` hâlâ bu yolu tanımladığında OpenClaw katalog satırını kullanır. `openclaw doctor --fix`, eşleşen eski geçersiz kılmaları kaldırır ve farklı değerleri inceleme için bildirir.

    Gerçek anlamda özel bir sağlayıcı, özel model veya farklı bir uç noktaya yönlendirilmiş katalog modeli için `compat` bloğu desteklenmeye devam eder. Yalnızca ilgili uç noktaya karşı doğrulanan yetenekleri ayarlayın:

    | Özel yol anahtarı | Çalışma zamanı sözleşmesi |
    | --- | --- |
    | `supportsStore` | OpenAI `store` istek alanını kabul eder. |
    | `supportsPromptCacheKey` | OpenAI istem önbelleği/oturum yakınlığı anahtarlarını kabul eder. |
    | `supportsDeveloperRole` | `system` gerektirmek yerine `developer` iletilerini kabul eder. |
    | `supportsReasoningEffort` | Bir akıl yürütme çabası denetimini kabul eder. |
    | `supportsTemperature` | Bu model ve bağdaştırıcı için `temperature` kabul eder. |
    | `supportsUsageInStreaming` | Akış yanıtlarında kullanım meta verilerini yayımlar. |
    | `supportsTools` | Yapılandırılmış araç/işlev çağrısını destekler. Araçları devre dışı bırakmak için `false` ayarlayın. |
    | `supportsStrictMode` | Katı araç şemalarını kabul eder. |
    | `requiresStringContent` | Düz dize biçiminde Chat Completions ileti içeriği gerektirir. |
    | `strictMessageKeys` | Giden iletilerin yalnızca kabul edilen anahtarları içermesini gerektirir. |
    | `visibleReasoningDetailTypes` | Dökümlerde gösterilmesi güvenli akıl yürütme ayrıntısı blok türlerini adlandırır. |
    | `supportedReasoningEfforts` | Uç noktanın kabul ettiği akıl yürütme etiketlerini listeler. |
    | `reasoningEffortMap` | OpenClaw düşünme etiketlerini uç noktaya özgü etiketlerle eşler. |
    | `maxTokensField` | `max_tokens` veya `max_completion_tokens` seçer. |
    | `thinkingFormat` | Uç noktanın akıl yürütme yükü lehçesini seçer. |
    | `requiresToolResultName` | Araç sonucu iletilerinde bir araç adı gerektirir. |
    | `requiresAssistantAfterToolResult` | Araç sonuçlarından sonra bir asistan iletisi gerektirir. |
    | `requiresThinkingAsText` | Akıl yürütmeyi yapılandırılmış içerik yerine metin olarak yeniden oynatır. |
    | `requiresReasoningContentOnAssistantMessages` | Yeniden oynatma sırasında DeepSeek tarzı `reasoning_content` değerini korur. |
    | `toolSchemaProfile` | Sağlayıcı tarafından tanımlanan bir araç şeması normalleştirme profili seçer. |
    | `unsupportedToolSchemaKeywords` | Uç noktanın reddettiği adlandırılmış JSON Schema anahtar sözcüklerini kaldırır. |
    | `toolCallArgumentsEncoding` | Uç noktanın araç çağrısı bağımsız değişkeni kodlamasını seçer. |
    | `requiresOpenAiAnthropicToolPayload` | OpenAI biçimli araç çağrılarını Anthropic ailesi yüklerine dönüştürür. |

  </Accordion>
  <Accordion title="Amazon Bedrock keşfi">
    - `plugins.entries.amazon-bedrock.config.discovery`: Bedrock otomatik keşif ayarlarının kökü.
    - `plugins.entries.amazon-bedrock.config.discovery.enabled`: örtük keşfi açar/kapatır.
    - `plugins.entries.amazon-bedrock.config.discovery.region`: keşif için AWS bölgesi.
    - `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: hedefli keşif için isteğe bağlı sağlayıcı kimliği filtresi.
    - `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: keşif yenilemesi için yoklama aralığı.
    - `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: keşfedilen modeller için yedek bağlam penceresi.
    - `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: keşfedilen modeller için yedek maksimum çıktı token sayısı.

  </Accordion>
</AccordionGroup>

Etkileşimli özel sağlayıcı ilk kurulumu; GPT-4o/GPT-4.1/GPT-5+, `o1`/`o3`/`o4` akıl yürütme aileleri, Claude, Gemini, `-vl` son ekine sahip tüm kimlikler (Qwen-VL ve benzerleri) ve LLaVA, Pixtral, InternVL, Mllama, MiniCPM-V ve GLM-4V gibi adlandırılmış aileler dahil olmak üzere bilinen görsel model kimliği kalıpları için görüntü girdisini çıkarır; yalnızca metin kullandığı bilinen ailelerde (Llama, DeepSeek, Mistral/Mixtral, Kimi/Moonshot, Codestral, Devstral, Phi, QwQ, CodeLlama ve vl/vision son eki bulunmayan yalın Qwen kimlikleri) ek soruyu atlar. Bilinmeyen model kimliklerinde görüntü desteği yine sorulur. Etkileşimsiz ilk kurulum da aynı çıkarımı kullanır; görüntü özellikli meta verileri zorunlu kılmak için `--custom-image-input`, yalnızca metin meta verilerini zorunlu kılmak için `--custom-text-input` iletin.

### Sağlayıcı örnekleri

<AccordionGroup>
  <Accordion title="Cerebras (GLM 4.7 / GPT OSS)">
    Resmî harici `cerebras` sağlayıcı plugini bunu `openclaw onboard --auth-choice cerebras-api-key` aracılığıyla yapılandırabilir. Açık sağlayıcı yapılandırmasını yalnızca varsayılanları geçersiz kılarken kullanın.

    ```json5
    {
      env: { CEREBRAS_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: {
            primary: "cerebras/zai-glm-4.7",
            fallbacks: ["cerebras/gpt-oss-120b"],
          },
          models: {
            "cerebras/zai-glm-4.7": { alias: "GLM 4.7 (Cerebras)" },
            "cerebras/gpt-oss-120b": { alias: "GPT OSS 120B (Cerebras)" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          cerebras: {
            baseUrl: "https://api.cerebras.ai/v1",
            apiKey: "${CEREBRAS_API_KEY}",
            api: "openai-completions",
            models: [
              { id: "zai-glm-4.7", name: "GLM 4.7 (Cerebras)" },
              { id: "gpt-oss-120b", name: "GPT OSS 120B (Cerebras)" },
            ],
          },
        },
      },
    }
    ```

    Cerebras için `cerebras/zai-glm-4.7`; doğrudan Z.AI için `zai/glm-4.7` kullanın.

  </Accordion>
  <Accordion title="Kimi Coding">
    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: { "kimi/kimi-for-coding": { alias: "Kimi Code" } },
        },
      },
    }
    ```

    Anthropic uyumlu, yerleşik sağlayıcı. Kısayol: `openclaw onboard --auth-choice kimi-code-api-key`.

  </Accordion>
  <Accordion title="Yerel modeller (LM Studio)">
    [Yerel Modeller](/tr/gateway/local-models) bölümüne bakın. Kısaca: güçlü donanımlarda LM Studio Responses API aracılığıyla büyük bir yerel model çalıştırın; yedek olarak barındırılan modelleri birleştirilmiş durumda tutun.
  </Accordion>
  <Accordion title="MiniMax M3 (doğrudan)">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "Minimax" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          minimax: {
            baseUrl: "https://api.minimax.io/anthropic",
            apiKey: "${MINIMAX_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.12, cacheWrite: 0 },
                contextWindow: 1000000,
                maxTokens: 131072,
              },
            ],
          },
        },
      },
    }
    ```

    `MINIMAX_API_KEY` ayarlayın. Kısayollar: `openclaw onboard --auth-choice minimax-global-api` veya `openclaw onboard --auth-choice minimax-cn-api`. Model kataloğu varsayılan olarak M3'ü kullanır ve M2.7 varyantlarını da içerir. Anthropic uyumlu akış yolunda OpenClaw, `thinking` değerini açıkça kendiniz ayarlamadığınız sürece MiniMax M2.x düşünmesini varsayılan olarak devre dışı bırakır; MiniMax-M3 (ve M3.x) varsayılan olarak sağlayıcının belirtilmemiş/uyarlanabilir düşünme yolunda kalır. `/fast on` veya `params.fastMode: true`, `MiniMax-M2.7` değerini `MiniMax-M2.7-highspeed` olarak yeniden yazar.

  </Accordion>
  <Accordion title="Moonshot AI (Kimi)">
    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: { "moonshot/kimi-k2.6": { alias: "Kimi K2.6" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
            ],
          },
        },
      },
    }
    ```

    Çin uç noktası için: `baseUrl: "https://api.moonshot.cn/v1"` veya `openclaw onboard --auth-choice moonshot-api-key-cn`.

    Yerel Moonshot uç noktaları, paylaşılan `openai-completions` aktarımında akış kullanım uyumluluğunu bildirir ve OpenClaw bunu yalnızca yerleşik sağlayıcı kimliğine göre değil, uç nokta yeteneklerine göre etkinleştirir.

  </Accordion>
  <Accordion title="OpenCode">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "opencode/claude-opus-4-6" },
          models: { "opencode/claude-opus-4-6": { alias: "Opus" } },
        },
      },
    }
    ```

    `OPENCODE_API_KEY` (veya `OPENCODE_ZEN_API_KEY`) ayarlayın. Zen kataloğu için `opencode/...` başvurularını, Go kataloğu için `opencode-go/...` başvurularını kullanın. Kısayol: `openclaw onboard --auth-choice opencode-zen` veya `openclaw onboard --auth-choice opencode-go`.

  </Accordion>
  <Accordion title="Synthetic (Anthropic uyumlu)">
    ```json5
    {
      env: { SYNTHETIC_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
          models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          synthetic: {
            baseUrl: "https://api.synthetic.new/anthropic",
            apiKey: "${SYNTHETIC_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "hf:MiniMaxAI/MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```

    Temel URL, `/v1` bölümünü içermemelidir (Anthropic istemcisi bunu ekler). Kısayol: `openclaw onboard --auth-choice synthetic-api-key`.

  </Accordion>
  <Accordion title="Z.AI (GLM-4.7)">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-4.7" },
          models: { "zai/glm-4.7": {} },
        },
      },
    }
    ```

    `ZAI_API_KEY` ayarlayın. Model başvuruları standart `zai/*` sağlayıcı kimliğini kullanır. Kısayol: `openclaw onboard --auth-choice zai-api-key`.

    - Genel uç nokta: `https://api.z.ai/api/paas/v4`
    - Kodlama uç noktası: `https://api.z.ai/api/coding/paas/v4`
    - Varsayılan `zai-api-key` kimlik doğrulama seçeneği anahtarınızı sınar ve hangi uç noktaya ait olduğunu otomatik olarak algılar (algılama kesin sonuç vermezse varsayılanı Global olan bir isteme geri döner). Açık seçim için özel CN ve Coding-Plan kimlik doğrulama seçenekleri de kullanılabilir.
    - Genel uç nokta için temel URL geçersiz kılmasına sahip özel bir sağlayıcı tanımlayın.

  </Accordion>
</AccordionGroup>

---

## İlgili

- [Yapılandırma — aracılar](/tr/gateway/config-agents)
- [Yapılandırma — kanallar](/tr/gateway/config-channels)
- [Yapılandırma başvurusu](/tr/gateway/configuration-reference) — diğer üst düzey anahtarlar
- [Araçlar ve pluginler](/tr/tools)
