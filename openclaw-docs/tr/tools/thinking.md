---
read_when:
    - Düşünme, hızlı mod veya ayrıntılı çıktı yönergelerinin ayrıştırılmasını ya da varsayılanlarını ayarlama
summary: /think, /fast, /verbose, /trace ve akıl yürütme görünürlüğü için direktif söz dizimi
title: Düşünme düzeyleri
x-i18n:
    generated_at: "2026-07-26T23:08:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80968ce58f642090ba0f807874e43eea1206cd31d919414c690b7537dc523658
    source_path: tools/thinking.md
    workflow: 16
---

## Ne yapar

- Gelen herhangi bir gövdedeki satır içi direktif: `/t <level>`, `/think:<level>` veya `/thinking <level>`.
- Düzeyler (takma adlar): `off | minimal | low | medium | high | xhigh | adaptive | max | ultra`; kabaca Anthropic'in klasik "düşün" < "çok düşün" < "daha çok düşün" < "ultra düşün" sihirli sözcük sıralamasını yansıtır:
  - minimal ~ "düşün"
  - low ~ "çok düşün"
  - medium ~ "daha çok düşün"
  - high ~ "ultra düşün" (maksimum bütçe)
  - xhigh ~ "ultra düşün+" (GPT-5.2+ ve Codex modellerinin yanı sıra Anthropic Claude Opus 4.7+ eforu)
  - adaptive → sağlayıcı tarafından yönetilen uyarlanabilir düşünme (Anthropic/Bedrock üzerindeki Claude 4.6, Anthropic Claude Opus 4.7+ ve Google Gemini dinamik düşünmesi için desteklenir)
  - max → sağlayıcının maksimum akıl yürütmesi (Anthropic Claude Opus 4.7+; Ollama bunu en yüksek yerel `think` eforuna eşler)
  - ultra → seçili model/çalışma zamanı desteklediğinde sağlayıcının maksimum akıl yürütmesine ek olarak proaktif alt ajan orkestrasyonu
  - `x-high`, `x_high`, `extra-high`, `extra high` ve `extra_high`, `xhigh` ile eşlenir.
  - `highest`, `high` ile eşlenir.
- Sağlayıcı notları:
  - Düşünme menüleri ve seçicileri sağlayıcı profiline göre belirlenir. Sağlayıcı Plugin'leri, ikili `on` gibi etiketler dâhil olmak üzere seçili modelin tam düzey kümesini bildirir.
  - `adaptive`, `xhigh`, `max` ve `ultra` yalnızca bunları destekleyen sağlayıcı/model/çalışma zamanı profilleri için sunulur. Desteklenmeyen düzeylere ait yazılı direktifler, ilgili modelin geçerli seçenekleri belirtilerek reddedilir.
  - Önceden depolanmış desteklenmeyen düzeyler, sağlayıcı profilindeki sıralamaya göre yeniden eşlenir. `adaptive`, uyarlanabilir olmayan modellerde `medium` düzeyine geri dönerken `xhigh` ve `max`, seçili modelin desteklediği kapalı dışındaki en yüksek düzeye geri döner.
  - Anthropic Claude 4.6 modelleri, açıkça bir düşünme düzeyi ayarlanmadığında varsayılan olarak `adaptive` kullanır.
  - Anthropic Claude Opus 4.8 ve Opus 4.7, açıkça bir düşünme düzeyi ayarlanmadığı sürece düşünmeyi kapalı tutar. Uyarlanabilir düşünme etkinleştirildikten sonra Opus 4.8'in sağlayıcıya ait varsayılan eforu `high` olur.
  - Anthropic Claude Opus 4.7+, `/think xhigh` düzeyini uyarlanabilir düşünme ve `output_config.effort: "xhigh"` ile eşler; çünkü `/think` bir düşünme direktifi, `xhigh` ise Opus efor ayarıdır.
  - Anthropic Claude Opus 4.7+ ayrıca `/think max` düzeyini sunar; bu düzey, sağlayıcıya ait aynı maksimum efor yoluyla eşlenir.
  - Doğrudan DeepSeek V4 modelleri `/think xhigh|max` düzeyini sunar; her ikisi de DeepSeek `reasoning_effort: "max"` ile eşlenirken kapalı dışındaki daha düşük düzeyler `high` ile eşlenir.
  - OpenRouter üzerinden yönlendirilen DeepSeek V4 modelleri `/think xhigh` düzeyini sunar ve DeepSeek'e özgü üst düzey `reasoning_effort` yerine OpenRouter tarafından desteklenen `reasoning.effort` değerlerini gönderir. Kapalı dışındaki daha düşük düzeyler `high` ile eşlenir ve depolanmış `max` geçersiz kılmaları `xhigh` düzeyine geri döner.
  - Düşünme özellikli Ollama modelleri `/think low|medium|high|max` düzeyini sunar; Ollama'nın yerel API'si `low`, `medium` ve `high` efor dizelerini kabul ettiğinden `max`, yerel `think: "high"` ile eşlenir.
  - OpenAI GPT modelleri, `/think` düzeyini modele özgü Responses API efor desteği aracılığıyla eşler. `/think off`, yalnızca hedef model desteklediğinde `reasoning.effort: "none"` gönderir; aksi takdirde OpenClaw, desteklenmeyen bir değer göndermek yerine devre dışı bırakılmış akıl yürütme yükünü atlar.
  - GPT-5.6 Sol ve Terra, Codex çalışma zamanı üzerinden yerel `/think ultra` düzeyini sunar. Codex kataloğu Ultra'yı sunmadığından GPT-5.6 Luna, düzeyleri `max` üzerinden sunar.
  - Gömülü OpenClaw çalışma zamanı, GPT-5.6 Sol, Terra ve Luna için mantıksal `/think ultra` düzeyini sunar. Sağlayıcının maksimum eforunu gönderir ve çalıştırma kapsamlı proaktif alt ajan orkestrasyonu yönlendirmesi ekler.
  - Özel OpenAI uyumlu katalog girdileri, `models.providers.<provider>.models[].compat.supportedReasoningEfforts` değerini `"xhigh"` içerecek şekilde ayarlayarak `/think xhigh` desteğini etkinleştirebilir. Bu işlem, giden OpenAI akıl yürütme eforu yüklerini eşleyen uyumluluk meta verilerini kullanır; böylece menüler, oturum doğrulaması, ajan CLI'si ve `llm-task` aktarım davranışıyla tutarlı olur.
  - Yapılandırılmış eski OpenRouter Hunter Alpha başvuruları, kullanımdan kaldırılmış bu rota nihai yanıt metnini akıl yürütme alanları üzerinden döndürebildiğinden proxy akıl yürütme eklemesini atlar.
  - Google Gemini, `/think adaptive` düzeyini Gemini'nin sağlayıcıya ait dinamik düşünmesiyle eşler. Gemini 3 istekleri sabit bir `thinkingLevel` değerini atlarken Gemini 2.5 istekleri `thinkingBudget: -1` gönderir; sabit düzeyler yine de ilgili model ailesi için en yakın Gemini `thinkingLevel` değerine veya bütçesine eşlenir.
  - Anthropic uyumlu akış yolundaki MiniMax M2.x (`minimax/MiniMax-M2*`), model parametrelerinde veya istek parametrelerinde açıkça düşünme ayarlanmadığı sürece varsayılan olarak `thinking: { type: "disabled" }` kullanır. Bu, M2.x'in yerel olmayan Anthropic akış biçiminden `reasoning_content` deltalarının sızmasını önler. MiniMax-M3 (ve M3.x) bundan muaftır: M3, doğru Anthropic düşünme blokları üretir ve düşünme devre dışı bırakıldığında boş içerik döndürür; bu nedenle OpenClaw, M3'ü sağlayıcının atlanmış/uyarlanabilir düşünme yolunda tutar.
  - Z.AI (`zai/*`), çoğu GLM modeli için ikilidir (`on`/`off`). GLM-5.2 istisnadır: `/think off|low|high|max` düzeyini sunar, `low` ve `high` düzeylerini Z.AI `reasoning_effort: "high"` ile, `max` düzeyini ise `reasoning_effort: "max"` ile eşler.
  - Moonshot API Kimi K3 (`moonshot/kimi-k3`) her zaman `max` düzeyinde düşünür, `reasoning_effort: "max"` gönderir, K2'nin `thinking` alanını ve sabit örnekleme geçersiz kılmalarını atlar ve K3 tarafından desteklenen araç seçimlerini korur. Kimi Code K3 (`kimi/k3` ve `kimi/k3[1m]`), `/think off|max` düzeyini sunar: kapalı durum `thinking.type: "disabled"` gönderirken maksimum durum maksimum eforlu uyarlanabilir düşünme gönderir. Güncel Kimi Code başvuruları ayrıca `kimi/kimi-for-coding` ve `kimi/kimi-for-coding-highspeed` değerlerini içerir. Kimi K2.7 Code (`moonshot/kimi-k2.7-code` ve `moonshot/kimi-k2.7-code-highspeed`) her zaman düşünür, yalnızca `on` düzeyini sunar ve hem giden `thinking` hem de `reasoning_effort` değerini atlar. Diğer `moonshot/*` modelleri, `/think off` düzeyini `thinking: { type: "disabled" }` ile ve `off` dışındaki tüm düzeyleri `thinking: { type: "enabled" }` ile eşler. K2 düşünmesi etkinleştirildiğinde Moonshot yalnızca `tool_choice` `auto|none` değerini kabul eder; OpenClaw uyumsuz değerleri `auto` olarak normalleştirir.

## Çözümleme sırası

1. Mesajdaki satır içi direktif (yalnızca o mesaj için geçerlidir).
2. Oturum geçersiz kılması (yalnızca direktif içeren bir mesaj gönderilerek ayarlanır).
3. Ajan başına varsayılan (`agents.entries.*.thinkingDefault` yapılandırmada).
4. Genel varsayılan (`agents.defaults.thinkingDefault` yapılandırmada).
5. Geri dönüş: mevcutsa sağlayıcının bildirdiği varsayılan; aksi takdirde akıl yürütme özellikli modeller `medium` düzeyine veya ilgili model için desteklenen, `off` dışındaki en yakın düzeye çözümlenir ve akıl yürütme özelliği olmayan modeller `off` olarak kalır.

## Oturum varsayılanını ayarlama

- **Yalnızca** direktiften oluşan bir mesaj gönderin (boşluklara izin verilir), ör. `/think:medium` veya `/t high`.
- Bu, geçerli oturum boyunca kalıcıdır (varsayılan olarak gönderen başına). Oturum geçersiz kılmasını temizlemek ve yapılandırılmış/sağlayıcı varsayılanını devralmak için `/think default` kullanın; diğer adlar arasında `inherit`, `clear`, `reset` ve `unpin` bulunur.
- `/think off`, açık bir kapalı geçersiz kılması saklar. Oturum geçersiz kılmasını değiştirene veya temizleyene kadar düşünmeyi devre dışı bırakır.
- Onay yanıtı gönderilir (`Thinking level set to high.` / `Thinking disabled.`). Düzey geçersizse (ör. `/thinking big`), komut bir ipucuyla reddedilir ve oturum durumu değiştirilmez.
- Geçerli düşünme düzeyini görmek için bağımsız değişken olmadan `/think` (veya `/think:`) gönderin.

## Agent tarafından uygulama

- **Gömülü OpenClaw**: çözümlenen düzey, işlem içi OpenClaw agent çalışma zamanına aktarılır.
- **Claude CLI arka ucu**: `claude-cli` kullanılırken kapalı dışındaki somut düzeyler Claude Code'a `--effort` olarak aktarılır; `adaptive`, yapılandırılmış efor bayraklarını kaldırır ve etkin eforu Claude Code'un ortamına, ayarlarına ve model varsayılanlarına devreder. Bkz. [CLI arka uçları](/tr/gateway/cli-backends).

## Hızlı mod (/fast)

- Düzeyler: `auto|on|off|default`.
- Yalnızca direktiften oluşan mesaj, oturum hızlı mod geçersiz kılmasını açıp kapatır ve `Fast mode set to auto.`, `Fast mode enabled.` veya `Fast mode disabled.` yanıtını verir. Oturum geçersiz kılmasını temizlemek ve yapılandırılmış varsayılanı devralmak için `/fast default` kullanın; diğer adlar arasında `inherit`, `clear`, `reset` ve `unpin` bulunur.
- Geçerli etkin hızlı mod durumunu görmek için mod belirtmeden `/fast` (veya `/fast status`) gönderin.
- OpenClaw hızlı modu şu sırayla çözümler:
  1. Satır içi/yalnızca direktiften oluşan `/fast auto|on|off` geçersiz kılması (`/fast default` bu katmanı temizler)
  2. Oturum geçersiz kılması
  3. Agent başına varsayılan (`agents.entries.*.fastModeDefault`)
  4. Model başına yapılandırma: `agents.defaults.models["<provider>/<model>"].params.fastMode`
  5. Geri dönüş: `off`
- `auto`, oturum/yapılandırma modunu otomatik olarak tutar ancak her yeni model çağrısını bağımsız olarak çözümler. Otomatik kesimden önce başlayan çağrılarda hızlı mod etkindir; daha sonraki yeniden deneme, geri dönüş, araç sonucu veya devam çağrıları hızlı mod devre dışı olarak başlar. Kesim varsayılan olarak 60 saniyedir; bunu değiştirmek için etkin modelde `agents.defaults.models["<provider>/<model>"].params.fastAutoOnSeconds` ayarlayın.
- `openai/*` için hızlı mod, desteklenen Responses isteklerinde `service_tier=priority` göndererek OpenAI öncelikli işlemeye eşlenir.
- Codex destekli `openai/*` / `openai-codex/*` modellerinde hızlı mod, Codex Responses üzerinde aynı `service_tier=priority` bayrağını gönderir. Yerel Codex uygulama sunucusu turları, katmanı yalnızca `turn/start` sırasında veya iş parçacığı başlatılırken/sürdürülürken alır; bu nedenle `auto`, zaten çalışmakta olan bir uygulama sunucusu turunun katmanını değiştiremez; OpenClaw'ın başlattığı sonraki model turuna uygulanır.
- OAuth kimlik doğrulamalı olarak `api.anthropic.com` hedefine gönderilen trafik dâhil olmak üzere doğrudan genel `anthropic/*` isteklerinde hızlı mod, Anthropic hizmet katmanlarına eşlenir: `/fast on`, `service_tier=auto` değerini; `/fast off` ise `service_tier=standard_only` değerini ayarlar.
- Anthropic uyumlu yoldaki `minimax/*` için `/fast on` (veya `params.fastMode: true`), `MiniMax-M2.7` değerini `MiniMax-M2.7-highspeed` olarak yeniden yazar.
- Açık Anthropic `serviceTier` / `service_tier` model parametreleri, ikisi de ayarlandığında hızlı mod varsayılanını geçersiz kılar. OpenClaw, Anthropic dışındaki proxy temel URL'leri için Anthropic hizmet katmanı eklemeyi yine de atlar.
- `/status`, hızlı mod etkinken `Fast`; yapılandırılmış mod otomatik olduğunda ise `Fast:auto` gösterir.

## Ayrıntılı direktifler (/verbose veya /v)

- Düzeyler: `on` (minimum) | `full` | `off` (varsayılan).
- Yalnızca yönerge içeren mesaj, oturumun ayrıntılı modunu değiştirir ve `Verbose logging enabled.` / `Verbose logging disabled.` yanıtını verir; geçersiz düzeyler, durumu değiştirmeden bir ipucu döndürür.
- `/verbose off`, açık bir oturum geçersiz kılma ayarı depolar; Sessions kullanıcı arayüzünde `inherit` seçeneğini belirleyerek bunu temizleyin.
- Yetkilendirilmiş harici kanal göndericileri, oturumun ayrıntılı mod geçersiz kılma ayarını kalıcı hâle getirebilir. Dahili gateway/webchat istemcilerinin bunu kalıcı hâle getirmek için `operator.admin` kullanması gerekir.
- Satır içi yönerge yalnızca ilgili mesajı etkiler; aksi hâlde oturum/genel varsayılanları uygulanır.
- Geçerli ayrıntı düzeyini görmek için bağımsız değişken olmadan `/verbose` (veya `/verbose:`) gönderin.
- Ayrıntılı mod açıkken yapılandırılmış araç sonuçları üreten aracılar, her araç çağrısını yalnızca meta veri içeren ayrı bir mesaj olarak geri gönderir ve kullanılabildiğinde başına `<emoji> <tool-name>: <arg>` ekler. Bu araç özetleri, her araç başlatılır başlatılmaz (ayrı baloncuklar hâlinde) gönderilir; akış deltaları olarak gönderilmez.
- Araç hatası özetleri normal modda görünür kalır, ancak ayrıntılı mod `full` olmadığı sürece ham hata ayrıntısı sonekleri gizlenir.
- Ayrıntılı mod `full` olduğunda araç çıktıları da tamamlandıktan sonra iletilir (ayrı bir baloncukta, güvenli bir uzunluğa kadar kesilerek). Bir çalıştırma devam ederken `/verbose on|full|off` ayarını değiştirirseniz sonraki araç baloncukları yeni ayara uyar.
- `agents.defaults.toolProgressDetail`, `/verbose` araç özetlerinin ve ilerleme taslağındaki araç satırlarının biçimini denetler. `🛠️ Exec: checking JS syntax` gibi kısa ve anlaşılır etiketler için `"explain"` (varsayılan) kullanın; hata ayıklama amacıyla ham komutun/ayrıntının da eklenmesini istiyorsanız `"raw"` kullanın. Aracı başına `agents.entries.*.toolProgressDetail`, varsayılanı geçersiz kılar.
  - `explain`: `🛠️ Exec: check JS syntax for /tmp/app.js`
  - `raw`: `🛠️ Exec: check JS syntax for /tmp/app.js, node --check /tmp/app.js`

## Plugin izleme yönergeleri (/trace)

- Düzeyler: `on` | `off` (varsayılan).
- Yalnızca yönerge içeren mesaj, oturumun Plugin izleme çıktısını açıp kapatır ve `Plugin trace enabled.` / `Plugin trace disabled.` yanıtını verir.
- Satır içi yönerge yalnızca ilgili mesajı etkiler; aksi hâlde oturum/genel varsayılanları uygulanır.
- Geçerli izleme düzeyini görmek için bağımsız değişken olmadan `/trace` (veya `/trace:`) gönderin.
- `/trace`, `/verbose` seçeneğinden daha dar kapsamlıdır: yalnızca Active Memory hata ayıklama özetleri gibi Plugin'e ait izleme/hata ayıklama satırlarını gösterir.
- İzleme satırları `/status` içinde ve normal asistan yanıtından sonra gönderilen bir tanılama mesajında görünebilir.

## Akıl yürütme görünürlüğü (/reasoning)

- Düzeyler: `on|off|stream`.
- Yalnızca yönerge içeren mesaj, düşünme bloklarının yanıtlarda gösterilip gösterilmeyeceğini değiştirir.
- Etkinleştirildiğinde akıl yürütme, başına `Thinking` eklenmiş **ayrı bir mesaj** olarak gönderilir.
- `stream`: etkin kanal akıl yürütme önizlemelerini desteklediğinde yanıt oluşturulurken akıl yürütmeyi akış hâlinde iletir, ardından nihai yanıtı akıl yürütme olmadan gönderir.
- Takma ad: `/reason`.
- Geçerli akıl yürütme düzeyini görmek için bağımsız değişken olmadan `/reasoning` (veya `/reasoning:`) gönderin.
- Çözümleme sırası: satır içi yönerge, ardından oturum geçersiz kılma ayarı, ardından aracı başına varsayılan (`agents.entries.*.reasoningDefault`), ardından genel varsayılan (`agents.defaults.reasoningDefault`), ardından geri dönüş değeri (`off`).

Hatalı yerel model akıl yürütme etiketleri ihtiyatlı biçimde işlenir. Kapatılmış `<think>...</think>` blokları normal yanıtlarda gizli kalır ve zaten görünür olan metinden sonraki kapatılmamış akıl yürütme de gizlenir. Bir yanıtın tamamı tek bir kapatılmamış açılış etiketiyle çevrelenmişse ve aksi takdirde boş metin olarak iletilecekse OpenClaw hatalı açılış etiketini kaldırır ve kalan metni iletir.

## İlgili

- Yükseltilmiş mod belgeleri [Yükseltilmiş mod](/tr/tools/elevated) bölümündedir.

## Heartbeat'ler

- Heartbeat yoklama gövdesi, yapılandırılmış Heartbeat istemidir (varsayılan: `Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`). Bir Heartbeat mesajındaki satır içi yönergeler her zamanki gibi uygulanır (ancak oturum varsayılanlarını Heartbeat'lerden değiştirmekten kaçının).
- Heartbeat iletimi varsayılan olarak yalnızca nihai yükü içerir. Ayrı `Thinking` mesajını da göndermek için (kullanılabildiğinde) `agents.defaults.heartbeat.includeReasoning: true` veya aracı başına `agents.entries.*.heartbeat.includeReasoning: true` ayarını belirleyin.

## Web sohbeti kullanıcı arayüzü

- Sayfa yüklendiğinde web sohbetindeki düşünme seçici, gelen oturum deposunda/yapılandırmasında kayıtlı oturum düzeyini yansıtır.
- Başka bir düzey seçildiğinde oturum geçersiz kılma ayarı `sessions.patch` aracılığıyla hemen yazılır; bir sonraki gönderimi beklemez ve tek seferlik bir `thinkingOnce` geçersiz kılma ayarı değildir.
- Model, akıl yürütme veya hız seçici değişiklikleri hâlâ uygulanırken yapılan gönderim, bekleyen tüm seçici yamalarının tamamlanmasını bekler; bir değişiklik başarısız olursa mesaj incelenmek üzere gönderilmeden kalır.
- İlk seçenek her zaman geçersiz kılma ayarını temizleme seçeneğidir. Devralınan düşünme devre dışı olduğunda `Inherited: Off` dâhil olmak üzere `Inherited: <resolved level>` gösterir.
- Açık seçici tercihleri, varsa sağlayıcı etiketlerini koruyarak doğrudan düzey etiketlerini kullanır (örneğin, sağlayıcı etiketli bir `max` seçeneği için `Maximum`).
- Seçici, eski bir etiket listesi olarak `thinkingOptions` korunurken Gateway oturum satırı/varsayılanları tarafından döndürülen `thinkingLevels` değerini kullanır. Tarayıcı kullanıcı arayüzü kendi sağlayıcı regex listesini tutmaz; modele özgü düzey kümelerinin sahibi Plugin'lerdir.
- `/think:<level>` çalışmaya devam eder ve aynı kayıtlı oturum düzeyini günceller; böylece sohbet yönergeleriyle seçici eşitlenmiş kalır.

## Sağlayıcı profilleri

- Sağlayıcı Plugin'leri, modelin desteklediği düzeyleri ve varsayılanını tanımlamak için `resolveThinkingProfile(ctx)` sunabilir.
- Claude modellerine proxy görevi yapan sağlayıcı Plugin'leri, doğrudan Anthropic ve proxy kataloglarının uyumlu kalması için `openclaw/plugin-sdk/provider-model-shared` içindeki `resolveClaudeThinkingProfile(modelId)` değerini yeniden kullanmalıdır.
- Her profil düzeyinin kayıtlı bir standart `id` değeri (`off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `adaptive`, `max` veya `ultra`) vardır ve bir görüntüleme `label` değeri içerebilir. İkili sağlayıcılar `{ id: "low", label: "on" }` kullanır.
- Profil kancaları, mevcut olduğunda `reasoning`, `compat.thinkingFormat` ve `compat.supportedReasoningEfforts` dâhil birleştirilmiş katalog bilgilerini alır. Bu bilgileri, yalnızca yapılandırılmış istek sözleşmesi eşleşen yükü desteklediğinde ikili veya özel profilleri sunmak için kullanın.
- Açık bir düşünme geçersiz kılma ayarını doğrulaması gereken araç Plugin'leri, `api.runtime.agent.resolveThinkingPolicy({ provider, model, agentRuntime })` ile `api.runtime.agent.normalizeThinkingLevel(...)` kullanmalıdır; kendi sağlayıcı/model düzeyi listelerini tutmamalıdır. Araç, her zaman gömülü bir çalıştırma gibi yürütme yolunun sahibiyse `agentRuntime` iletin.
- Yapılandırılmış özel model meta verilerine erişimi olan araç Plugin'leri, `compat.supportedReasoningEfforts` katılımlarının Plugin tarafındaki doğrulamaya yansıtılması için `catalog` değerini `resolveThinkingPolicy` içine iletebilir.
- Yayımlanmış eski kancalar (`supportsXHighThinking`, `isBinaryThinking` ve `resolveDefaultThinkingLevel`) uyumluluk bağdaştırıcıları olarak kalır, ancak yeni özel düzey kümeleri `resolveThinkingProfile` kullanmalıdır.
- Gateway satırları/varsayılanları; ACP/sohbet istemcilerinin, çalışma zamanı doğrulamasının kullandığı profil kimliklerini ve etiketleri aynı şekilde oluşturması için `thinkingLevels`, `thinkingOptions` ve `thinkingDefault` değerlerini sunar.
