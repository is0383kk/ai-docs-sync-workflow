---
read_when:
    - Token kullanımını, maliyetleri veya bağlam pencerelerini açıklama
    - Bağlam büyümesi veya Compaction davranışında hata ayıklama
summary: OpenClaw istem bağlamını nasıl oluşturur ve token kullanımını + maliyetleri nasıl raporlar
title: Token kullanımı ve maliyetler
x-i18n:
    generated_at: "2026-07-26T23:01:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6624bceb0bcbca769c9d569389b73b82f1ea73133e09f0ae9859833196d85911
    source_path: reference/token-use.md
    workflow: 16
---

OpenClaw karakterleri değil, **tokenları** izler. Tokenlar modele özgüdür, ancak çoğu
OpenAI tarzı modelde İngilizce metin için token başına ortalama ~4 karakter düşer.

## Sistem istemi nasıl oluşturulur

OpenClaw her çalıştırmada kendi sistem istemini oluşturur. Şunları içerir:

- Araç listesi + kısa açıklamalar
- Skills listesi (yalnızca meta veriler; talimatlar gerektiğinde `read` ile yüklenir). Yerel
  Codex turları, kompakt Skills bloğunu tur kapsamlı iş birliği
  geliştirici talimatları olarak alır; diğer çalıştırma ortamları bunu normal istem yüzeyinde alır.
  `skills.limits.maxSkillsPromptChars` ile sınırlandırılır ve `agents.entries.*.skillsLimits.maxSkillsPromptChars` üzerinden isteğe bağlı olarak agent başına
  geçersiz kılınabilir.
- Kendi kendini güncelleme talimatları
- Çalışma alanı + önyükleme dosyaları (yeni olduğunda `AGENTS.md`, `SOUL.md`, `TOOLS.md`,
  `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`; ayrıca mevcut olduğunda
  `MEMORY.md`). Eklenen büyük dosyalar
  `agents.defaults.bootstrapMaxChars` tarafından kırpılır (varsayılan: `20000`); toplam önyükleme
  eklemesi `agents.defaults.bootstrapTotalMaxChars` ile sınırlandırılır (varsayılan:
  `60000`).
  - Yerel Codex turları, ilgili çalışma alanında bellek araçları
    kullanılabiliyorsa ham `MEMORY.md` içeriğini yapıştırmaz; bunun yerine tur kapsamlı
    iş birliği geliştirici talimatlarında küçük bir bellek işaretçisi alır ve gerektiğinde
    bellek araçlarını kullanır. Araçlar devre dışıysa, bellek araması kullanılamıyorsa veya
    etkin çalışma alanı agent bellek çalışma alanından farklıysa `MEMORY.md`,
    normal sınırlandırılmış tur bağlamı yoluna geri döner.
  - Küçük harfli kök `memory.md` hiçbir zaman eklenmez. Bu, onu
    `MEMORY.md` içine taşıyan `openclaw doctor --fix` için eski onarım girdisidir.
  - `memory/*.md` günlük dosyaları normal önyükleme isteminin parçası değildir;
    sıradan turlarda bellek araçları aracılığıyla gerektiğinde kullanılmaya devam eder. Sıfırlama/başlangıç
    model çalıştırmaları, `agents.defaults.startupContext` tarafından denetlenen ve ilk tur için
    yakın tarihli günlük belleği içeren tek seferlik bir başlangıç bağlamı bloğunu başa ekleyebilir.
    Yalın sohbet `/new` ve `/reset` komutları model çağrılmadan
    onaylanır.
  - Compaction sonrası `AGENTS.md` alıntıları açıkça
    `agents.defaults.compaction.postCompactionSections` etkinleştirmesi gerektirir; pluginler
    `before_prompt_build` aracılığıyla başka bağlamlar ekleyebilir.
- Saat (UTC + kullanıcı saat dilimi)
- Yanıt etiketleri + heartbeat davranışı
- Çalışma zamanı meta verileri (ana makine/işletim sistemi/model/düşünme)

Ayrıntılı döküm için [Sistem İstemi](/tr/concepts/system-prompt) bölümüne bakın.

Kimlik bilgilerini veya kimlik doğrulama parçacıklarını belgelendirirken, yalnızca doküman
değişikliklerinde gizli bilgi tarayıcısının yanlış pozitiflerini önlemek için
[Gizli Bilgi Yer Tutucu Kuralları](/tr/reference/secret-placeholder-conventions) bölümünü kullanın.

## Bağlam penceresine neler dâhildir

Modelin aldığı her şey bağlam sınırına dâhildir:

- Sistem istemi (yukarıdaki tüm bölümler)
- Konuşma geçmişi (kullanıcı + asistan mesajları)
- Araç çağrıları ve araç sonuçları
- Ekler/transkriptler (görüntüler, sesler, dosyalar)
- Compaction özetleri ve budama yapıtları
- Sağlayıcı sarmalayıcıları veya güvenlik başlıkları (görünmez, ancak yine de sayılır)

Çalışma zamanı açısından yoğun yüzeylerin
`agents.defaults.contextLimits` altında kendi açık sınırları vardır (agent başına geçersiz kılmalar
`agents.entries.*.contextLimits` altındadır):

| Anahtar                  | Amaç                                                                     |
| ------------------------ | ------------------------------------------------------------------------ |
| `memoryGetMaxChars`      | `memory_get` kırpılmadan önce en fazla kaç karakter döndürebilir.        |
| `postCompactionMaxChars` | Compaction sonrası yenileme sırasında `AGENTS.md` içinden tutulan en fazla karakter sayısı. |

Bunlar; önyükleme sınırlarından, başlangıç bağlamı sınırlarından ve Skills istemi
sınırlarından ayrı, sınırlandırılmış çalışma zamanı alıntıları ve çalışma zamanının sahip olduğu eklenmiş bloklardır.

OpenClaw, canlı araç sonucu sınırını etkin model bağlam
penceresinden türetir: 100K tokenın altında `16000` karakter,
100K+ tokenda `32000` karakter, 200K+ tokenda `64000` karakter.
Çalışma zamanı bağlam payı koruması ayrıca tek bir araç sonucunu bağlam
penceresinin %30'u ile sınırlar.

Büyük sağlayıcı pencereleri, maliyeti veya gecikmeyi önemli ölçüde
değiştirdiğinde otomatik olarak etkinleştirilmez. Örneğin doğrudan OpenAI GPT-5.5 ve GPT-5.6 modelleri
toplam `1050000` tokenlık bir pencere yayımlar, ancak OpenClaw etkin
çalışma zamanı bütçelerini varsayılan olarak `272000` tokenla sınırlar. İsteğe bağlı `922000` girdi bütçesi,
`128000` çıktı payının tamamını ayırır ve girdi `272000` tokenı aştığında OpenAI,
isteğin tamamına daha yüksek uzun bağlam fiyatlandırması uygular. Bkz.
[OpenAI bağlam penceresi varsayılanları](/tr/providers/openai#context-window-defaults-and-long-context-opt-in).

OpenClaw, görüntüler için sağlayıcı çağrılarından önce transkript/araç görüntüsü yüklerini
küçültür. `agents.defaults.imageMaxDimensionPx` ile ayarlayın (varsayılan:
`1200`):

- Daha düşük değerler, görsel token kullanımını ve yük boyutunu azaltır.
- Daha yüksek değerler, OCR/UI ağırlıklı ekran görüntülerinde daha fazla görsel ayrıntıyı korur.

Pratik bir döküm (eklenen dosya, araçlar, Skills ve sistem
istemi boyutu başına) için `/context list` veya `/context detail` kullanın. Bkz.
[Bağlam](/tr/concepts/context).

## Geçerli token kullanımı nasıl görüntülenir

Sohbette:

- `/status` -> oturum modeli, bağlam kullanımı,
  son yanıtın girdi/çıktı tokenları ve etkin model için yerel fiyatlandırma
  yapılandırılmışsa tahmini maliyet bilgilerini içeren emoji bakımından zengin durum kartı.
- `/usage off|tokens|full` -> her yanıta yanıt başına kullanım altbilgisi ekler.
  Oturum başına kalıcıdır (`responseUsage` olarak saklanır).
  - `/usage reset` (takma adlar: `inherit`, `clear`, `default`),
    yapılandırılmış varsayılanı yeniden devralması için oturum geçersiz kılmasını temizler.
  - `/usage tokens`, tur token/önbellek ayrıntılarını gösterir.
  - `/usage full`, kısa model/bağlam/maliyet ayrıntılarını gösterir; tahmini maliyet
    yalnızca OpenClaw etkin model için kullanım meta verilerine ve yerel fiyatlandırmaya
    sahip olduğunda görünür. Özel `messages.usageTemplate` düzenleri
    token/önbellek alanlarını içerebilir.
- `/usage cost` -> OpenClaw oturum günlüklerinden yerel maliyet özeti.

Diğer yüzeyler:

- **TUI/Web TUI:** `/status` ve `/usage` desteklenir.
- **CLI:** `openclaw status --usage` ve `openclaw channels list`,
  normalleştirilmiş sağlayıcı kota pencerelerini gösterir (`X% left`, yanıt başına maliyetleri değil).
  Geçerli kullanım penceresi sağlayıcıları: Claude (Anthropic), ClawRouter, Copilot
  (GitHub), DeepSeek, Gemini (Google Gemini CLI), MiniMax, OpenAI, Xiaomi,
  Xiaomi Token Plan ve z.ai.

Kullanım yüzeyleri, görüntülemeden önce yaygın sağlayıcıya özgü alan takma adlarını
normalleştirir. OpenAI ailesi Responses trafiği için buna hem
`input_tokens`/`output_tokens` hem de `prompt_tokens`/`completion_tokens` dâhildir; böylece
taşımaya özgü alan adları `/status`, `/usage` veya oturum
özetlerini değiştirmez. Gemini CLI kullanımı da normalleştirilir: varsayılan `stream-json`
ayrıştırıcısı asistan `message` olaylarını okur ve `stats.cached`,
`cacheRead` değerine eşlenir; CLI açık bir `stats.input` alanını
atladığında `stats.input_tokens - stats.cached` kullanılır. Eski JSON geçersiz kılmaları yanıt metnini
hâlâ `response` içinden okur.

Yerel OpenAI ailesi Responses trafiğinde WebSocket/SSE kullanım takma adları
aynı şekilde normalleştirilir ve `total_tokens` eksik veya `0` olduğunda
toplamlar normalleştirilmiş girdi + çıktıya geri döner.

Geçerli oturum anlık görüntüsü seyrek olduğunda `/status` ve `session_status`,
en son transkript kullanım günlüğünden token/önbellek sayaçlarını ve etkin çalışma zamanı
model etiketini kurtarabilir. Sıfır olmayan mevcut canlı değerler, transkript geri dönüş
değerlerine göre önceliğini korur; saklanan toplamlar eksik veya daha küçük olduğunda
istem odaklı daha büyük transkript toplamları geçerli olabilir.

Sağlayıcı kota pencerelerinin kullanım kimlik doğrulaması önce sağlayıcıya özgü kancalardan
gelir; bir sağlayıcının kancası yoksa (veya kanca bir token çözümleyemezse)
OpenClaw, kimlik doğrulama profilleri, ortam veya yapılandırmadaki eşleşen OAuth/API anahtarı
kimlik bilgilerine geri döner.

Asistan transkript girdileri, etkin modelde fiyatlandırma yapılandırılmışsa ve
sağlayıcı kullanım meta verileri döndürüyorsa `usage.cost` dâhil olmak üzere aynı
normalleştirilmiş kullanım biçimini kalıcı olarak saklar. Bu, canlı çalışma zamanı
durumu ortadan kalktıktan sonra bile `/usage cost` ve transkript destekli oturum durumu
için kararlı bir kaynak sağlar.

OpenClaw, sağlayıcı kullanım hesabını geçerli bağlam anlık görüntüsünden ayrı tutar.
Sağlayıcı `usage.total`; önbelleğe alınmış girdiyi, çıktıyı ve birden fazla
araç döngüsü model çağrısını içerebilir; bu nedenle maliyet ve telemetri için kullanışlıdır,
ancak canlı bağlam penceresini olduğundan büyük gösterebilir. Bağlam ekranları ve tanılamalar,
`context.used` için en son istem anlık görüntüsünü (`promptTokens`; istem anlık görüntüsü
yoksa son model çağrısı) kullanır.

## Maliyet tahmini (gösterildiğinde)

Maliyetler, model fiyatlandırma yapılandırmanızdan tahmin edilir:

```text
models.providers.<provider>.models[].cost
```

Bunlar `input`, `output`, `cacheRead` ve
`cacheWrite` için **1M token başına USD** değerleridir. Fiyatlandırma eksikse
`/usage full` maliyeti atlar; her yanıtta token/önbellek ayrıntılarına
ihtiyaç duyduğunuzda `/usage tokens` veya özel bir `messages.usageTemplate` kullanın.
Maliyet gösterimi API anahtarıyla kimlik doğrulamayla sınırlı değildir:
`aws-sdk` gibi API anahtarı kullanmayan sağlayıcılar, yapılandırılmış model girdileri
yerel fiyatlandırma içerdiğinde ve sağlayıcı kullanım meta verileri döndürdüğünde
tahmini maliyeti gösterebilir.

Yardımcı işlemler ve kanallar Gateway hazır yoluna ulaştıktan sonra OpenClaw,
henüz yerel fiyatlandırması olmayan yapılandırılmış model referansları için isteğe bağlı
bir arka plan fiyatlandırma önyüklemesi başlatır. Bu önyükleme, uzak OpenRouter ve
LiteLLM fiyatlandırma kataloglarını getirir. Çevrimdışı veya kısıtlı ağlarda bu
katalog getirmelerini atlamak için `models.pricing.enabled: false` değerini ayarlayın; açık
`models.providers.*.models[].cost` girdileri yerel maliyet tahminlerini yönlendirmeye devam eder.

## Önbellek TTL'si ve budama etkisi

Sağlayıcı istem önbelleğe alması yalnızca önbellek TTL penceresi içinde geçerlidir. OpenClaw,
isteğe bağlı olarak **önbellek TTL budaması** çalıştırabilir: önbellek TTL'sinin süresi dolduğunda
oturumu budar, ardından önbellek penceresini sıfırlar; böylece sonraki istekler tüm geçmişi
yeniden önbelleğe almak yerine yeni önbelleğe alınmış bağlamı yeniden kullanır.
Bu, bir oturum TTL süresinden daha uzun süre boşta kaldığında önbellek yazma maliyetlerini düşürür.

Bunu [Gateway yapılandırması](/tr/gateway/configuration) bölümünde yapılandırın ve davranış
ayrıntıları için [Oturum budama](/tr/concepts/session-pruning) bölümüne bakın.

Heartbeat, boşta kalma aralıklarında önbelleği **sıcak** tutabilir. Model önbelleği
TTL'niz `1h` ise heartbeat aralığını bunun hemen altına (örneğin `55m`) ayarlamak,
tüm istemin yeniden önbelleğe alınmasını önleyerek önbellek yazma maliyetlerini azaltabilir.

Çok agentlı kurulumlarda, tek bir paylaşılan model yapılandırmasını koruyabilir ve
önbellek davranışını `agents.entries.*.params.cacheRetention` ile agent başına ayarlayabilirsiniz.

Tüm ayarları tek tek açıklayan bir kılavuz için [İstem Önbelleğe Alma](/tr/reference/prompt-caching) bölümüne bakın.

Anthropic API fiyatlandırmasında önbellek okumaları, girdi tokenlarından önemli ölçüde
daha ucuzken önbellek yazmaları daha yüksek bir çarpanla ücretlendirilir. En güncel ücretler ve
TTL çarpanları için Anthropic'in istem önbelleğe alma fiyatlandırmasına bakın:
[https://docs.anthropic.com/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/docs/build-with-claude/prompt-caching)

### Örnek: 1h önbelleği heartbeat ile sıcak tutma

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
    heartbeat:
      every: "55m"
```

### Örnek: agent başına önbellek stratejisiyle karma trafik

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long" # çoğu agent için varsayılan temel değer
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m" # derin oturumlar için uzun önbelleği sıcak tut
    - id: "alerts"
      params:
        cacheRetention: "none" # yoğun bildirimler için önbelleğe yazmayı önle
```

`agents.entries.*.params`, seçilen modelin `params` değerinin üzerine birleştirilir; böylece
yalnızca `cacheRetention` değerini geçersiz kılabilir ve diğer model varsayılanlarını
değişmeden devralabilirsiniz.

### Anthropic 1M bağlamı

OpenClaw; Opus 4.8, Opus 4.7, Opus 4.6 ve Sonnet 4.6 gibi genel kullanıma
sunulmuş Claude 4.x modellerini Anthropic'in 1M bağlam penceresiyle boyutlandırır.
Bu modeller için `params.context1m: true` gerekmez.

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        alias: opus
```

Eski yapılandırmalar `context1m: true` değerini koruyabilir, ancak OpenClaw artık
bu ayar için Anthropic'in kullanımdan kaldırılmış `context-1m-2025-08-07` beta başlığını göndermez
ve desteklenmeyen eski Claude modellerini 1M bağlama genişletmez.

Gereksinim: kimlik bilgisi uzun bağlam kullanımına uygun olmalıdır. Uygun değilse
Anthropic, bu istek için sağlayıcı taraflı bir hız sınırı hatası döndürür.

Anthropic kimlik doğrulamasını OAuth/abonelik token'larıyla
(`sk-ant-oat-*`) yaparsanız OpenClaw, eski yapılandırmada kalmışsa kullanımdan
kaldırılmış `context-1m-*` beta başlığını çıkarırken OAuth için gerekli Anthropic
beta başlıklarını korur.

## Token baskısını azaltmaya yönelik ipuçları

- Uzun oturumları özetlemek için `/compact` kullanın.
- İş akışlarınızdaki büyük araç çıktılarını kısaltın.
- Ekran görüntüsü ağırlıklı oturumlar için `agents.defaults.imageMaxDimensionPx` değerini düşürün.
- Skill açıklamalarını kısa tutun (Skill listesi isteme eklenir).
- Ayrıntılı, keşif amaçlı çalışmalar için daha küçük modelleri tercih edin.

Skill listesinin ek yükünü hesaplayan kesin formül için [Skills](/tr/tools/skills) bölümüne bakın.

## İlgili

- [API kullanımı ve maliyetleri](/tr/reference/api-usage-costs)
- [İstem önbelleğe alma](/tr/reference/prompt-caching)
- [Kullanım takibi](/tr/concepts/usage-tracking)
