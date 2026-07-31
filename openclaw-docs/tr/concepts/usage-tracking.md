---
read_when:
    - Sağlayıcı kullanım/kota yüzeylerini bağlıyorsunuz
    - Kullanım takibi davranışını veya kimlik doğrulama gereksinimlerini açıklamanız gerekir
summary: Kullanım takibi yüzeyleri ve kimlik bilgisi gereksinimleri
title: Kullanım takibi
x-i18n:
    generated_at: "2026-07-26T23:55:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5a1bc9aeb95cd80a48ab57a18fcd24894fdd6fb71e10e8bea8bae67a8688b78e
    source_path: concepts/usage-tracking.md
    workflow: 16
---

## Nedir

- Sağlayıcı kullanımını/kotasını doğrudan her sağlayıcının kullanım uç noktasından alır. Tahmini sağlayıcı faturalandırması yoktur; yalnızca sağlayıcının bildirdiği plan adları, kota pencereleri, bakiyeler, harcamalar, bütçeler, günlük maliyet geçmişi, token/model ilişkilendirmesi veya hesap durumu özetleri gösterilir.
- İnsan tarafından okunabilir kota penceresi çıktısı, sağlayıcı tüketilen kotayı, kalan kotayı veya yalnızca ham sayıları bildirse bile `X% left` biçimine normalleştirilir. Sıfırlanabilir kota pencereleri olmayan sağlayıcılarda bunun yerine sağlayıcı özet metni (örneğin bakiye) gösterilir.
- Oturum düzeyindeki `/status` ve `session_status` aracı, canlı oturum anlık görüntüsünde token/model verileri eksik olduğunda oturumun transkript günlüğüne geri döner. Bu geri dönüş, eksik token/önbellek sayaçlarını doldurur, etkin çalışma zamanı modeli etiketini kurtarabilir ve oturum meta verileri eksik veya daha küçük olduğunda (`totalTokensFresh !== true`, sıfır ya da transkriptten türetilen değerin altında) istem odaklı daha büyük toplamı tercih eder. Sıfır olmayan canlı değerler her zaman geri dönüş değerlerine üstün gelir.

## Göründüğü yerler

- Sohbetlerde `/status`: oturum token'larını ve tahmini maliyeti içeren durum kartı (yalnızca API anahtarı modelleri). Sağlayıcı kullanımı, kullanılabilir olduğunda **geçerli model sağlayıcısı** için normalleştirilmiş bir `X% left` penceresi veya sağlayıcı özet metni olarak gösterilir.
- Sohbetlerde `/usage off|tokens|full`: yanıt başına kullanım altbilgisi.
- Sohbetlerde `/usage cost`: OpenClaw oturum günlüklerinden birleştirilen yerel maliyet özeti.
- CLI: `openclaw status --usage`, sağlayıcı başına tam kullanım/kota dökümünü yazdırır.
- CLI: `openclaw models status`, OAuth/token kimlik doğrulama profillerini listeler ve kullanım penceresi olan her sağlayıcının yanında bunun özetini gösterir.
- Denetim Arayüzü: **Kullanım**, OpenClaw'ın oturumdan türetilen token ve tahmini maliyet analizinin üzerinde sağlayıcı planı ve faturalandırma kartlarını gösterir. Anthropic ve OpenAI Admin API kimlik bilgileri; sağlayıcının bildirdiği bugünkü, 7 günlük ve 30 günlük harcamayı, günlük eğilimleri, token toplamlarını, en çok kullanılan modelleri ve maliyet kategorilerini ekler.
- Denetim Arayüzü: sohbet oluşturucunun bağlam halkası açılır penceresi, abonelik sağlayıcıları için **plan kullanımını** gösterir — sıfırlanma zamanlarıyla pencere başına çubuklar (5 saatlik, haftalık, model kapsamlı), biliniyorsa sağlayıcı planı (örneğin `Max (20x)`) ve ek kullanım kredileri. Bir plan üzerinden faturalandırılan oturumlar token başına dolar tahminlerini gizler; API üzerinden faturalandırılan oturumlarda `Est. cost` ve türe göre maliyet dökümü korunur. Claude Code CLI (`claude-cli`) kurulumları aynı Anthropic abonelik kullanımını yeniden kullanır.
- macOS menü çubuğu: sağlayıcı kullanım anlık görüntüleri mevcut olduğunda Bağlam'ın altında kök düzeyinde bir "Kullanım" bölümü görünür. Bkz. [Menü çubuğu](/tr/platforms/mac/menu-bar).

`openclaw channels list` artık sağlayıcı kullanımını yazdırmaz; bunun yerine kullanıcıları `openclaw status` veya `openclaw models list` seçeneğine yönlendirir.

## Anthropic ve OpenAI maliyet geçmişi

Abonelik kotası ve API faturalandırması farklı sağlayıcı yüzeyleridir:

- Anthropic abonelik/kurulum kimlik bilgileri, Claude kota pencerelerini ve isteğe bağlı ek kullanım bütçelerini göstermeye devam eder. Bunun yerine kuruluş Usage and Cost API geçmişini göstermek için `ANTHROPIC_ADMIN_KEY` veya `ANTHROPIC_ADMIN_API_KEY` ayarlayın. `sk-ant-admin` ile başlayan bir Anthropic sağlayıcı kimlik bilgisi otomatik olarak algılanır.
- OpenAI ChatGPT/Codex OAuth; planı, kota pencerelerini ve kredi bakiyesini göstermeye devam eder. Bunun yerine kuruluş maliyetini ve tamamlama kullanım geçmişini göstermek için `OPENAI_ADMIN_KEY` ayarlayın; isteğe bağlı olarak kapsamı tek bir projeyle sınırlamak için `OPENAI_PROJECT_ID` ayarlayın. OpenClaw, `OPENAI_API_KEY`, sağlayıcı yapılandırması veya kimlik doğrulama profillerindeki çıkarım kimlik bilgilerini kuruluş API'lerine hiçbir zaman göndermez; çünkü bu anahtarlar özel uç noktalara ait olabilir.

Admin kimlik bilgileri, gerçek kuruluş faturalandırmasını sağladıkları için önceliklidir. OpenClaw, sağlayıcı tarafından bildirilen bu toplamları yerel oturum tahminleriyle birleştirmez; iki bölüm kasıtlı olarak farklı soruları yanıtlar.

## Varsayılan kullanım altbilgisi modu

`/usage off|tokens|full`, bir oturumun altbilgisini ayarlar ve bu seçim söz konusu
oturum için hatırlanır. `messages.responseUsage`, henüz bir mod
seçmemiş oturumlarda bu modu başlangıç değeri olarak kullanır; böylece her seferinde `/usage` yazmadan altbilgi varsayılan olarak açık olabilir.

Her kanal için tek bir mod veya `default` geri dönüşü içeren kanal başına bir eşleme ayarlayın:

```jsonc
{
  "messages": {
    "responseUsage": "tokens",
    // veya: { "default": "off", "discord": "full" }
  },
}
```

Kabul edilen değerler: `"off"`, `"tokens"`, `"full"` ve eski diğer ad `"on"` (`"tokens"` olarak değerlendirilir).

### Üç farklı oturum durumu

Bir oturumun `responseUsage` alanı, her biri farklı
anlamlara sahip üç temsil edilebilir duruma sahiptir:

| Durum                 | Saklanan değer                         | Etkin mod                                                                     |
| --------------------- | -------------------------------------- | ----------------------------------------------------------------------------- |
| **Ayarlanmamış / devral** | `undefined` (yok)           | `messages.responseUsage` yapılandırma varsayılanına, ardından `off` değerine geçer. |
| **Açıkça kapalı**     | `"off"` (saklanır)          | Her zaman kapalıdır; kapalı olmayan bir yapılandırma varsayılanı altbilgiyi yeniden etkinleştiremez. |
| **Açıkça açık**       | `"tokens"` veya `"full"` (saklanır) | Yapılandırma varsayılanından bağımsız olarak belirtilen mod.                  |

### Öncelik

Etkin mod = oturum geçersiz kılması → kanal yapılandırma girdisi → `default` → `off`.

Açıkça belirtilen bir `/usage off`, oturumda değişmez `"off"` değeri olarak **kalıcılaştırılır**;
bu, "ayarlanmamış" olmakla aynı değildir. Kullanıcı altbilgiyi açıkça devre dışı bıraktıktan sonra kapalı olmayan bir `messages.responseUsage`
varsayılanı altbilgiyi yeniden açamaz.

### Sıfırlama ile kapatma arasındaki fark

- `/usage off`, altbilgiyi zorunlu olarak kapatır ve bu seçimi kalıcılaştırır. Yapılandırılmış
  kapalı olmayan bir varsayılan bunu geçersiz kılamaz.
- `/usage reset` (diğer adlar: `default`, `inherit`, `inherited`, `clear`, `unpin`) oturum
  geçersiz kılmasını temizler. Ardından oturum, etkin yapılandırma varsayılanını
  (`messages.responseUsage`) **devralır**. Hiçbir varsayılan yapılandırılmamışsa altbilgi kapalı kalır.
- Tam oturum sıfırlaması (`/reset` veya `/new`) ya da oturum devri, açık kullanım modu
  tercihini **korur**; böylece kullanıcının görüntüleme seçimi oturum devirlerinden
  sonra da geçerli kalır. Geçersiz kılmayı yalnızca `/usage reset` (ve diğer adları) temizler.

### Geçiş davranışı

Bağımsız değişken olmadan `/usage` şu döngüyü izler: kapalı → token'lar → tam → kapalı. Döngünün
başlangıç noktası, **etkin** geçerli moddur (ayarlanmamış olduğunda oturum geçersiz kılması
yapılandırma varsayılanına geçer); böylece döngü her zaman kullanıcının
altbilgide o anda gördüğüyle eşleşir.

### Yapılandırma

Yapılandırma olmadığında önceki davranış geçerliliğini korur (altbilgi `/usage` yapılana kadar kapalıdır). Bir
oturum geçersiz kılmasını temizlemek ve yapılandırılmış varsayılanı yeniden devralmak için `/usage reset` kullanın.

## Özel `/usage full` altbilgisi

`/usage tokens` her zaman düz bir `Usage: X in / Y out` satırı oluşturur (varsa önbellek ve
tahmini maliyet son ekleriyle birlikte). Aşağıda açıklanan daha zengin
altbilgiyi yalnızca `/usage full` oluşturur.

`/usage full`; model, akıl yürütme, hızlı/yavaş, bağlam
penceresi ve mevcut olduğunda maliyeti içeren yerleşik, kompakt bir altbilgi gösterir. Yerleşik altbilgi için
şablon dosyası gerekmez.

`messages.usageTemplate` yalnızca gelişmiş özel düzenler içindir. Değer,
bir JSON dosyası yolu (`~` desteklenir) veya satır içi nesnedir ve geçerliyse yerleşik
altbilginin yerini alır. Dosya yolu izlenir ve değişiklik olduğunda canlı olarak yeniden yüklenir.

```json
{
  "messages": {
    "usageTemplate": "~/.openclaw/usage-footer.json"
  }
}
```

Eksik veya boş şablonlar sessizce yerleşik altbilgiye geri döner. Okunamayan
veya geçersiz yapılandırılmış şablonlar (bozuk JSON ya da oluşturulabilir çıktı
parçası içermeyen bir şekil) da yerleşik altbilgiye geri döner ve operatör uyarısı yayınlar.

Özel şablonlara yerleşik şekilden başlayın, ardından değiştirmek istediğiniz
bölümleri düzenleyin:

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": {
    "braille": "⠐⡀⡄⡆⡇⣇⣧⣷⣿",
    "block": "░▏▎▍▌▋▊▉█",
    "shade": "░▒▓█",
    "moon": "🌑🌘🌗🌖🌕",
    "level": "▁▂▃▄▅▆▇█",
    "weather": ["🥶", "☁️", "🌥", "⛅️", "🌤", "☀️"],
    "plants": ["🪾", "🍂", "🌱", "☘️", "🍀", "🌿"],
    "moons6": ["🌑", "🌚", "🌘", "🌗", "🌖", "🌝"],
  },
  "aliases": {
    "models": {
      "claude-opus-4-6": "opus46",
      "claude-opus-4-8": "opus48",
      "claude-sonnet-4-6": "sonnet46",
      "claude-haiku-4-5": "haiku45",
      "gpt-5.5": "gpt5.5",
    },
    "reasoning": {
      "off": "🌑",
      "minimal": "🌚",
      "low": "🌘",
      "medium": "🌗",
      "high": "🌕",
      "xhigh": "🌝",
    },
  },
  "output": {
    "sep": "",
    "default": [
      { "text": "{model.provider}{identity.emoji|🤖}{model.display_name|alias:models}" },
      { "map": "model.is_fallback", "cases": { "true": "🔄" } },
      { "map": "model.is_override", "cases": { "true": "📌" } },
      { "when": "model.reasoning", "text": "{model.reasoning|alias:reasoning}" },
      { "map": "state.fast_mode", "cases": { "true": "⚡️", "false": "🐌" } },
      {
        "when": "context.max_tokens",
        "text": " | 📚[{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
      },
      { "when": "cost.turn_usd", "text": " 💰{cost.turn_usd|fixed:4}" },
    ],
    "surfaces": {
      "discord": [
        { "text": "-# -\n" },
        { "text": "-# {model.provider}{identity.emoji|🤖}{model.display_name|alias:models}" },
        { "map": "model.is_fallback", "cases": { "true": "🔄" } },
        { "map": "model.is_override", "cases": { "true": "📌" } },
        { "when": "model.reasoning", "text": "{model.reasoning|alias:reasoning}" },
        { "map": "state.fast_mode", "cases": { "true": "⚡️", "false": "🐌" } },
        {
          "when": "context.max_tokens",
          "text": " | 📚[{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
        },
        { "when": "cost.turn_usd", "text": " 💰{cost.turn_usd|fixed:4}" },
      ],
    },
  },
}
```

### Şekil

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": { "<name>": "düşükten yükseğe glifler" }, // dize (karakter başına 1 glif) veya dizi
  "aliases": { "<table>": { "<value>": "<label>" } },
  "output": {
    "sep": "", // kalan parçaları birleştirir
    "default": [/* pieces */], // herhangi bir yüzey için geri dönüş
    "surfaces": {
      "discord": [/* pieces */],
      "telegram": [/* pieces */],
    },
  },
}
```

Her yüzey sıralı bir **parçalar** listesidir; motor her birini oluşturur, boş
olanları çıkarır ve kalanları `sep` ile birleştirir. Girdisi olmayan bir yüzey
`output.default` kullanır.

### Sözleşme Yolları

Bir parça, dönüş başına sözleşmedeki değerleri noktalı yol üzerinden okur. Mevcut olmayan değerler
boştur (böylece bir `when` koruması veya bir `|fallback` parçayı temiz tutar).

| Yol                                                                                | Anlamı                                                                                              |
| ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `surface`                                                                           | kanal kimliği (`discord`/`telegram`/vb.)                                                               |
| `agentId` / `chat_type`                                                             | sahip ajan kimliği / sohbet yüzeyi türü                                                                  |
| `model.id` / `model.display_name` / `model.provider`                                | model kimliği / görünen ad / sağlayıcı kimliği                                                                |
| `model.actual`, `model.resolved_ref`                                                | tur için gerçekten kullanılan sağlayıcı/model referansı                                                        |
| `model.requested`                                                                   | istenen sağlayıcı/model referansı (geri dönüşten önce)                                                       |
| `model.reasoning`                                                                   | çaba (`off` ile `xhigh` arası)                                                                       |
| `model.is_fallback` / `model.is_override`                                           | bool: geri dönüş kullanıldı / model sabitlendi                                                                   |
| `model.override_source` / `model.auth_mode`                                         | geçersiz kılma kaynağı etiketi / kimlik bilgisi modu (`oauth`, `api-key`, `token`, `mixed`, `aws-sdk`, `unknown`) |
| `state.fast_mode`                                                                   | bool: hızlı veya yavaş                                                                                   |
| `state.compactions`                                                                 | oturumun Compaction sayısı                                                                     |
| `context.max_tokens` / `context.used_tokens` / `context.pct_used`                   | pencere bütçesi / kullanılan token'lar / kullanılan 0-100                                                         |
| `usage.input_tokens` / `usage.output_tokens` / `usage.total_tokens`                 | tur toplamı                                                                                       |
| `usage.cache_read_tokens` / `usage.cache_write_tokens`                              | turun önbellek okuma ve önbellek yazma token'ları                                                       |
| `usage.has_tokens` / `usage.has_split_tokens` / `usage.has_total_only_tokens`       | token gösterim korumaları                                                                                 |
| `usage.cache_hit_pct`                                                               | toplam istem token'ları içinde önbellekten okunanların payı                                                              |
| `usage.last.input_tokens` / `usage.last.output_tokens` / `usage.last.cache_hit_pct` | yalnızca son model çağrısı (ayrıca `cache_read_tokens`, `cache_write_tokens`, `total_tokens` içerir)           |
| `cost.turn_usd` / `cost.available`                                                  | tahmini tur maliyeti / bir maliyet tablosunun çözümlenip çözümlenmediği                                                  |
| `timing.duration_ms`                                                                | duvar saatiyle tur süresi                                                                             |
| `identity.name` / `identity.emoji` / `identity.avatar`                              | ajan kimliği adı / emoji / avatar                                                                 |
| `session.id`                                                                        | oturum kimliği                                                                                           |

(Sağlayıcı hız sınırı pencereleri bu sözleşmede **yoktur**; bugün dizi değerli bir yol bulunmadığından bir `each` parçasının yineleyeceği hiçbir şey yoktur.)

### Fiiller

Bir değeri fiillerden soldan sağa geçirin; fiil olmayan bir bölüm geri dönüş değeridir.

| Fiil            | Etki                                | Örnek                           |
| --------------- | ------------------------------------- | --------------------------------- |
| `num`           | kısa gösterimli sayı                         | `272000 -> 272k`                  |
| `fixed:N`       | N ondalık basamak (`0..100`, varsayılan 2)      | `0.0377`                          |
| `dur`           | saniyeyi süreye dönüştürür                   | `14820 -> 4h07m`                  |
| `pct`           | sonuna `%` ekler                            | `96 -> 96%`                       |
| `inv`           | `100 - x`                             | kullanılandan kalana dönüştürmek için             |
| `alias:TABLE`   | `aliases` içinde arar, listede yoksa aynen döndürür | `medium -> 🌗`                    |
| `meter:W:SCALE` | 0-100 değeri üzerinde W hücreli glif çubuğu   | `[⣿⣿⠐⠐⠐]` (`meter:1` = bir glif) |

`fixed:N` yalnızca 0 ile 100 arasında eksiksiz bir ondalık tam sayı kabul eder. Geçersiz
hassasiyet bağımsız değişkenleri bu interpolasyonu boş bırakır.

`meter:W:SCALE` yalnızca 1 ile 100 arasında eksiksiz bir ondalık tam sayı genişliği kabul eder. Varsayılan 5 değerini (`meter::braille`) kullanmak için genişliği boş bırakın; geçersiz
genişlikler bu interpolasyonu boş bırakır.

### Parça biçimleri

- `{ "text": "📚 {context.max_tokens|num}" }`: sabit metin + interpolasyon.
- `{ "when": "<path>", "text": "..." }`: yalnızca yol doğruluk değeri taşıyorsa oluştur.
- `{ "map": "<path>", "cases": { "true": "⚡", "false": "🐌" } }`: değeri glife dönüştürür (bir `_default` durumu eşleşmeyen değerleri kapsar).
- `{ "each": "<array-path>", "item": "{label}" }`: dizi değerli bir yolu yineler (mevcut hiçbir sözleşme yolu dizi değildir).

### Örnek

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": { "braille": "⠐⡀⡄⡆⡇⣇⣧⣷⣿" },
  "aliases": { "reasoning": { "medium": "🌗", "high": "🌕" } },
  "output": {
    "surfaces": {
      "discord": [
        { "text": "{model.display_name}" },
        { "when": "model.reasoning", "text": " {model.reasoning|alias:reasoning}" },
        { "map": "state.fast_mode", "cases": { "true": " ⚡", "false": " 🐌" } },
        {
          "when": "context.max_tokens",
          "text": " | 📚 [{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
        },
      ],
    },
  },
}
```

örneğin `claude-sonnet-4-6 🌗 🐌 | 📚 [⣿⣿⣿⣿⣧]272k` olarak oluşturulur.

## Sağlayıcılar + kimlik bilgileri

Kullanılabilir sağlayıcı kullanım yetkilendirmesi çözümlenemediğinde kullanım gizlenir. OpenClaw,
`contracts.usageProviders` bildiren ve hem `resolveUsageAuth` hem de
`fetchUsageSnapshot` uygulayan etkin sağlayıcı Plugin'lerini otomatik olarak keşfeder;
ayrı bir çekirdek sağlayıcı izin listesi yoktur. Statik sözleşme,
her sağlayıcı Plugin'ini içe aktarmadan keşfin kapsamını korur. Her
Plugin, kendi yukarı akış uç noktasına ve yanıt eşlemesine sahiptir.
Paylaşılan anlık görüntü, plan adlarını, kota pencerelerini, bakiyeleri, harcamaları ve bütçeleri
CLI, uygulama ve Control UI tüketicileri için sağlayıcıdan bağımsız tutar.

- **Anthropic (Claude)**: Kimlik doğrulama profillerindeki OAuth token'ları. OAuth token'ında
  `user:profile` kapsamı yoksa, ayarlandığında bir `claude.ai` web oturumuna (`CLAUDE_AI_SESSION_KEY`,
  `CLAUDE_WEB_SESSION_KEY` veya `CLAUDE_WEB_COOKIE` içindeki bir `sessionKey=` çerezi) geri döner.
  Anthropic bunları bildirdiğinde model kapsamlı sınırlar ve etkinleştirilmiş ek kullanımın aylık harcamaları/bütçeleri
  dahil edilir. Açıkça belirtilen bir Anthropic Admin API anahtarı veya otomatik algılanan
  bir `sk-ant-admin...` sağlayıcı profili ise bunun yerine 30 günlük
  kuruluş maliyetini ve Messages API geçmişini gösterir.
- **ClawRouter**: API anahtarı (`CLAWROUTER_API_KEY`). Yapılandırıldığında aylık bir bütçe penceresi
  ve türü belirtilmiş USD bütçesi gösterir; aksi takdirde toplam harcamayı ve
  istek/token/maliyet özetini gösterir.
- **DeepSeek**: Ortam/yapılandırma/kimlik doğrulama deposu üzerinden API anahtarı (`DEEPSEEK_API_KEY`).
  Sağlayıcı tarafından bildirilen her para birimi bakiyesini gösterir.
- **GitHub Copilot**: Kimlik doğrulama profillerindeki OAuth token'ları.
- **Gemini CLI**: Kimlik doğrulama profillerindeki OAuth token'ları.
- **MiniMax**: API anahtarı veya MiniMax OAuth kimlik doğrulama profili. OpenClaw,
  `minimax`, `minimax-cn` ve `minimax-portal` değerlerini aynı MiniMax kota
  yüzeyi olarak değerlendirir, varsa depolanan MiniMax OAuth'ı tercih eder ve aksi takdirde
  `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY` veya `MINIMAX_API_KEY` değerine geri döner.
  Kullanım yoklaması, yapılandırıldığında Coding Plan ana makinesini `models.providers.minimax-portal.baseUrl`
  veya `models.providers.minimax.baseUrl` değerinden türetir; aksi takdirde
  MiniMax CN ana makinesini kullanır.
  MiniMax'in ham `usage_percent` / `usagePercent` alanları **kalan**
  kotayı ifade eder; bu nedenle OpenClaw bunları göstermeden önce tersine çevirir; mevcut
  olduğunda sayı tabanlı alanlar önceliklidir.
  - Pencere etiketleri, mevcut olduğunda sağlayıcının saat/dakika alanlarından gelir; ardından
    `start_time` / `end_time` aralığına geri döner.
  - Kodlama planı uç noktası `model_remains` döndürürse OpenClaw,
    sohbet modeli girdisini tercih eder, açık `window_hours` / `window_minutes`
    alanları yoksa pencere etiketini zaman damgalarından türetir ve model
    adını plan etiketine dahil eder.
- **OpenAI (Codex/ChatGPT planı)**: Kimlik doğrulama profillerindeki OAuth token'ları (bir hesap kimliği
  mevcut olduğunda `ChatGPT-Account-Id` üstbilgisi gönderilir). ChatGPT planını, sıfırlanabilir
  Codex pencerelerini ve bildirildiğinde kredi bakiyesini gösterir. Krediler sağlayıcı
  kredileri olarak kalır; OpenClaw bunları dolar olarak etiketlemez. `OPENAI_ADMIN_KEY`,
  anahtar Usage Dashboard erişimine sahip olduğunda 30 günlük kuruluş maliyetini ve tamamlama kullanım
  geçmişini ekler. Çıkarım kimlik bilgileri kuruluş API'lerine hiçbir zaman iletilmez.
- **OpenRouter**: API anahtarı veya OAuth destekli API anahtarı (`OPENROUTER_API_KEY` ya da bir kimlik doğrulama
  profili). Hesap kredileri uç noktasını anahtar kotası uç noktasıyla birleştirir;
  böylece kimlik bilgisi erişebildiğinde hesap bakiyesi/harcaması, anahtar bütçesi ve günlük/haftalık/aylık kullanım
  görünür. Her iki uç nokta da anlık görüntüyü birbirinden bağımsız olarak
  zenginleştirebilir.
- **Venice**: Ortam/yapılandırma/kimlik doğrulama deposu üzerinden API anahtarı (`VENICE_API_KEY`). Bildirildiğinde USD ve
  DIEM bakiyelerinin yanı sıra DIEM dönem tahsisi kullanımını gösterir.
- **Xiaomi MiMo**: İki ayrı kullanım yüzeyi. Kullandıkça öde, bir API anahtarı
  (`XIAOMI_API_KEY`) kullanır; Token Plan ayrı bir anahtar (`XIAOMI_TOKEN_PLAN_API_KEY`) kullanır.
  Şu anda ikisi de kota pencerelerini bildirmez.
- **z.ai**: Ortam/yapılandırma/kimlik doğrulama deposu üzerinden API anahtarı (`ZAI_API_KEY` veya `Z_AI_API_KEY`).

## İlgili

- [Token kullanımı ve maliyetleri](/tr/reference/token-use)
- [API kullanımı ve maliyetleri](/tr/reference/api-usage-costs)
- [İstem önbelleğe alma](/tr/reference/prompt-caching)
- [Menü çubuğu](/tr/platforms/mac/menu-bar)
