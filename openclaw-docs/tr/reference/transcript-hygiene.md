---
read_when:
    - Transkript yapısına bağlı sağlayıcı istek reddetmelerinde hata ayıklıyorsunuz
    - Transkript temizleme veya araç çağrısı onarma mantığını değiştiriyorsunuz
    - Sağlayıcılar arasındaki araç çağrısı kimliği uyuşmazlıklarını araştırıyorsunuz
summary: 'Referans: sağlayıcıya özgü transkript temizleme ve onarma kuralları'
title: Transkript düzeni
x-i18n:
    generated_at: "2026-07-26T23:01:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 33d978772062cb2a81eb358bb5c62bd1261b433ffdc8acdbaa6679b121fbbf62
    source_path: reference/transcript-hygiene.md
    workflow: 16
---

OpenClaw, bir çalıştırmadan önce (model bağlamını oluştururken) transkriptlere
**sağlayıcıya özgü düzeltmeler** uygular. Bunların çoğu, katı sağlayıcı
gereksinimlerini karşılamak için kullanılan **bellek içi** ayarlamalardır.
Ayrı bir oturum dosyası onarım geçişi de oturum yüklenmeden önce depolanan
JSONL'yi yeniden yazabilir, ancak yalnızca hatalı biçimlendirilmiş satırlar
veya geçerli kalıcı kayıtlar olmayan kalıcılaştırılmış turlar için.
İletilen asistan yanıtları diskte korunur; sağlayıcıya özgü asistan ön dolgu
kaldırma işlemi yalnızca giden yükler oluşturulurken gerçekleşir.

Bir onarım gerçekleştiğinde, atomik değiştirme işleminden önce özgün dosya
geçici bir `*.bak-<pid>-<ts>` eş dosyasına yazılır ve değiştirme başarıyla
tamamlandıktan sonra kaldırılır. Yedek yalnızca temizleme işleminin kendisi
başarısız olursa tutulur; bu durumda yol geri bildirilir.

Kapsam şunları içerir:

- Yalnızca çalışma zamanına ait istem bağlamının kullanıcıya görünür transkript turlarının dışında tutulması
- Araç çağrısı kimliği temizleme
- Araç çağrısı girdisi doğrulama
- Araç sonucu eşleştirme onarımı
- Tur doğrulama / sıralama
- Düşünce imzası temizleme
- Düşünme imzası temizleme
- Görüntü yükü temizleme
- Sağlayıcı yeniden oynatmasından önce boş metin bloğu temizleme
- Sağlayıcı yeniden oynatmasından önce tamamlanmamış, yalnızca akıl yürütme içeren uzunluk turu temizleme
- Kullanıcı girdisi kaynağını etiketleme (oturumlar arası yönlendirilmiş istemler için)
- Bedrock Converse yeniden oynatması için boş asistan hata turu onarımı

Transkript depolama ayrıntıları için
[Oturum yönetimine derinlemesine bakış](/tr/reference/session-management-compaction) bölümüne bakın.

---

## Genel kural: çalışma zamanı bağlamı kullanıcı transkripti değildir

Çalışma zamanı/sistem bağlamı, bir tur için model istemine eklenebilir ancak
son kullanıcı tarafından yazılmış içerik değildir. OpenClaw; Gateway
yanıtları, kuyruğa alınmış takipler, ACP, CLI ve gömülü OpenClaw
çalıştırmaları için transkripte yönelik ayrı bir istem gövdesi tutar.
Depolanan görünür kullanıcı turları, çalışma zamanı bağlamıyla zenginleştirilmiş
istem yerine bu transkript gövdesini kullanır.

Çalışma zamanı sarmalayıcılarını zaten kalıcılaştırmış eski oturumlarda,
Gateway geçmiş yüzeyleri iletileri WebChat, TUI, REST veya SSE istemcilerine
döndürmeden önce bir görüntüleme izdüşümü uygular.

---

## Bunun çalıştığı yer

Tüm transkript hijyeni gömülü çalıştırıcıda merkezileştirilmiştir:

- İlke seçimi: `src/agents/transcript-policy.ts`
  (`resolveTranscriptPolicy`; `provider`, `modelApi` ve `modelId` anahtarlarına göre)
- Temizleme/onarım uygulaması: `src/agents/embedded-agent-runner/replay-history.ts` içindeki
  `sanitizeSessionHistory`

Transkript hijyeninden ayrı olarak, oturum dosyaları yüklenmeden önce
(gerekirse) onarılır:

- `src/agents/session-file-repair.ts` içindeki `repairSessionFileIfNeeded`
- `src/agents/embedded-agent-runner/run/attempt.ts` ve
  `src/agents/embedded-agent-runner/compact.ts` tarafından çağrılır

---

## Genel kural: görüntü temizleme

Görüntü yükleri, boyut sınırları nedeniyle sağlayıcı tarafında reddedilmeyi
önlemek için her zaman temizlenir (aşırı büyük base64 görüntüler küçültülür/
yeniden sıkıştırılır). Bu işlem, görüntü özellikli modellerde görüntü kaynaklı
token baskısını denetlemeye de yardımcı olur: daha düşük azami boyutlar token
kullanımını azaltırken daha yüksek boyutlar ayrıntıları korur.

Uygulama:

- `src/agents/embedded-agent-helpers/images.ts` içindeki
  `sanitizeSessionMessagesImages`
- `src/agents/tool-images.ts` içindeki `sanitizeContentBlocksImages`
- Azami görüntü kenarı `agents.defaults.imageMaxDimensionPx` aracılığıyla yapılandırılabilir
  (varsayılan: `1200`)
- Bu geçiş yeniden oynatma içeriğini işlerken boş metin blokları kaldırılır.
  Boş hâle gelen asistan turları yeniden oynatma kopyasından çıkarılır;
  boş hâle gelen kullanıcı ve araç sonucu turlarına boş olmayan bir
  atlanan içerik yer tutucusu eklenir.

---

## Genel kural: hatalı biçimlendirilmiş araç çağrıları

Hem `input` hem de `arguments` eksik olan asistan araç çağrısı blokları,
model bağlamı oluşturulmadan önce kaldırılır. Bu, kısmen kalıcılaştırılmış
araç çağrılarının (örneğin bir hız sınırı hatasından sonra) sağlayıcı
tarafından reddedilmesini önler.

Uygulama:

- `src/agents/session-transcript-repair.ts` içindeki `sanitizeToolCallInputs`
- `sanitizeSessionHistory` içinde uygulanır
  (`src/agents/embedded-agent-runner/replay-history.ts`)

---

## Genel kural: araç sonucu eşleştirme

Araç sonuçları, sağlayıcıya özgü çağrı kimlikleri yeniden yazılmadan önce her asistan
turundaki araç çağrısı oluşumlarıyla eşleştirilir. Sağlayıcı tarafından oluşturulan
kimlikler sonraki turlarda tekrarlanabilir; bu nedenle tekrarlanan bir çağrıya bitişik
olan sonuç o oluşumla birlikte kalır. Yerinden çıkmış bir sonuç yalnızca tam olarak bir
çözümlenmemiş oluşum ona sahip olabiliyorsa taşınır; belirsiz fazlalıklar kaldırılır ve
eksik oluşumlara sentetik hata sonuçları eklenir.

Uygulama: `src/agents/session-transcript-repair.ts` içindeki
`sanitizeToolUseResultPairing`

---

## Genel kural: tamamlanmamış veya sessiz, yalnızca akıl yürütme içeren turlar

Asistan turları, aşağıdaki olaylardan herhangi birinden sonra yalnızca
düşünme veya sansürlenmiş düşünme içeriği barındırıyorsa bellek içi yeniden
oynatma kopyasından çıkarılır:

- Sağlayıcı çıktı sınırı, turu tamamlanmamış akıl yürütme durumuyla sonlandırır.
- Sessiz yanıt temizleme, turun tek görünür `NO_REPLY` metnini kaldırır.

Sessiz yanıt temizleme, katı sağlayıcılar konuşmayı yeniden oluşturduğunda
gizli akıl yürütmenin daha sonraki bir asistan araç kullanımı turuyla
birleşmesini önler.

Görünür metin, araç çağrıları veya bilinmeyen içerik blokları içeren uzunluk
turları gibi boş uzunluk turları da değiştirilmeden kalır. Araç çağrıları
veya bilinmeyen içerik blokları içeren sessiz yanıt turları da değiştirilmeden
kalır. Depolanan transkriptler yeniden yazılmaz.

Uygulama: `src/agents/embedded-agent-runner/replay-history.ts` içindeki
`normalizeAssistantReplayContent`

---

## Genel kural: oturumlar arası girdi kaynağı

Bir aracı, `sessions_send` aracılığıyla başka bir oturuma istem
gönderdiğinde (aracıdan aracıya yanıt/duyuru adımları dâhil), OpenClaw
oluşturulan kullanıcı turunu `message.provenance.kind = "inter_session"` ile kalıcılaştırır.

OpenClaw ayrıca, etkin model çağrısının yabancı oturum çıktısını harici son
kullanıcı talimatlarından ayırt edebilmesi için yönlendirilmiş istem
metninin önüne aynı turda bir `[Inter-session message] ... isUser=false`
işareti ekler. Bu işaret, mevcut olduğunda kaynak oturumu, kanalı ve aracı
içerir. Transkript, sağlayıcı uyumluluğu için yine `role: "user"`
kullanır ancak hem görünür metin hem de kaynak meta verileri turu
oturumlar arası veri olarak işaretler.

Bağlam yeniden oluşturulurken OpenClaw, aynı işareti yalnızca kaynak meta
verilerine sahip eski kalıcılaştırılmış oturumlar arası kullanıcı turlarına
da uygular.

---

## Sağlayıcı matrisi (mevcut davranış)

**OpenAI / OpenAI Codex**

- Yalnızca görüntü temizleme.
- OpenAI Responses/Codex transkriptleri için öksüz akıl yürütme imzalarını
  (ardından içerik bloğu gelmeyen bağımsız akıl yürütme öğeleri) kaldırır ve
  model rotası değişikliğinden sonra yeniden oynatılabilir OpenAI akıl yürütmesini
  kaldırır.
- Şifrelenmiş boş özetli öğeler dâhil olmak üzere yeniden oynatılabilir
  OpenAI Responses akıl yürütme öğesi yüklerini korur; böylece manuel/WebSocket
  yeniden oynatması gerekli `rs_*` durumunu asistan çıktı öğeleriyle
  eşleştirilmiş tutar.
- Yerel ChatGPT Codex Responses, oturum `prompt_cache_key` değerini korurken
  önceki öğe kimlikleri olmadan önceki Responses akıl yürütme/ileti/işlev
  yüklerini yeniden oynatarak Codex kablo protokolü eşliğini izler.
- OpenAI Responses ailesi yeniden oynatması, aynı modeldeki kurallı
  `call_*|fc_*` akıl yürütme çiftlerini korur ancak pi-ai yükü dönüşümünden önce
  hatalı biçimlendirilmiş veya aşırı uzun `call_id`/işlev çağrısı öğe kimliklerini
  belirlenimsel olarak normalleştirir.
- Araç sonucu eşleştirme onarımı, gerçek eşleşmiş çıktıları taşıyabilir ve
  eksik araç çağrıları için Codex tarzı `aborted` çıktıları sentezleyebilir.
- Tur doğrulama veya yeniden sıralama yoktur; düşünce imzası kaldırılmaz.

**OpenAI uyumlu Chat Completions**

- Yerel ve proxy tarzı OpenAI uyumlu sunucuların `reasoning` veya
  `reasoning_content` gibi önceki tura ait akıl yürütme alanlarını almaması için geçmiş
  asistan düşünme/akıl yürütme blokları yeniden oynatmadan önce kaldırılır.
- Mevcut aynı tur araç çağrısı devamları, araç sonucu yeniden oynatılana
  kadar asistan akıl yürütme bloğunu araç çağrısına bağlı tutar.
- `reasoning: true` değerine sahip özel/kendi barındırılan model girdileri,
  yeniden oynatılan akıl yürütme meta verilerini korur.
- Sağlayıcının sahip olduğu istisnalar, kablo protokolleri yeniden oynatılan
  akıl yürütme meta verilerini gerektiriyorsa kapsam dışında kalmayı seçebilir.

**Google (Generative AI / Gemini CLI / Antigravity)**

- Araç çağrısı kimliği temizleme: katı alfasayısal.
- Araç sonucu eşleştirme onarımı ve sentetik araç sonuçları.
- Tur doğrulama (Gemini tarzı tur dönüşümü).
- Google tur sıralaması düzeltmesi (geçmiş asistanla başlıyorsa başa küçük
  bir kullanıcı başlangıcı ekler).
- Antigravity Claude: düşünme imzalarını normalleştirir; imzasız düşünme
  bloklarını kaldırır.

**Anthropic / Minimax (Anthropic uyumlu)**

- Araç sonucu eşleştirme onarımı ve sentetik araç sonuçları.
- Tur doğrulama (katı dönüşümü karşılamak için art arda gelen kullanıcı
  turlarını birleştirir).
- Cloudflare AI Gateway rotaları dâhil olmak üzere düşünme etkinleştirildiğinde,
  sondaki asistan ön dolgu turları giden Anthropic Messages yüklerinden kaldırılır.
- Bir oturum sıkıştırıldığında, sıkıştırma öncesi asistan düşünme imzaları
  sağlayıcı yeniden oynatmasından önce kaldırılır. Düşünme imzaları, oluşturma
  sırasında konuşma önekine kriptografik olarak bağlıdır; sıkıştırmadan sonra
  önek değişir (özetlenmiş içerik özgün içeriğin yerini alır), bu nedenle özgün
  imzaların yeniden oynatılması Anthropic'in isteği "Invalid signature in
  thinking block" hatasıyla reddetmesine neden olur. Düşünme metni imzasız bir
  blok olarak korunur ve ardından aşağıdaki kural tarafından işlenir.
- Eksik, boş veya yalnızca boşluk içeren yeniden oynatma imzalarına sahip
  düşünme blokları sağlayıcı dönüşümünden önce kaldırılır. Bu işlem bir asistan
  turunu boşaltırsa OpenClaw, boş olmayan atlanmış akıl yürütme metniyle tur
  şeklini korur.
- Kaldırılması gereken eski, yalnızca düşünme içeren asistan turları, sağlayıcı
  bağdaştırıcılarının yeniden oynatma turunu kaldırmaması için boş olmayan
  atlanmış akıl yürütme metniyle değiştirilir.

**Amazon Bedrock (Converse API)**

- Boş asistan akış hatası turları, yeniden oynatmadan önce boş olmayan bir
  yedek metin bloğuna dönüştürülerek onarılır. Bedrock Converse, `content: []`
  içeren asistan iletilerini reddeder; bu nedenle `stopReason:
"error"` ve boş içerikle
  kalıcılaştırılmış asistan turları da yüklenmeden önce diskte onarılır.
- Yalnızca boş metin blokları içeren asistan akış hatası turları, geçersiz bir
  boş bloğu yeniden oynatmak yerine bellek içi yeniden oynatma kopyasından
  kaldırılır.
- Bir oturum sıkıştırıldığında, yukarıdaki Anthropic ile aynı nedenle
  sıkıştırma öncesi asistan düşünme imzaları Converse yeniden oynatmasından
  önce kaldırılır.
- Eksik, boş veya yalnızca boşluk içeren yeniden oynatma imzalarına sahip
  Claude düşünme blokları Converse yeniden oynatmasından önce kaldırılır. Bu
  işlem bir asistan turunu boşaltırsa OpenClaw, boş olmayan atlanmış akıl
  yürütme metniyle tur şeklini korur.
- Kaldırılması gereken eski, yalnızca düşünme içeren asistan turları, Converse
  yeniden oynatmasının katı tur şeklini koruması için boş olmayan atlanmış
  akıl yürütme metniyle değiştirilir.
- Yeniden oynatma, OpenClaw teslimat aynası ve Gateway tarafından eklenen
  asistan turlarını filtreler.
- Görüntü temizleme, genel kural aracılığıyla uygulanır.

**Mistral (model kimliğine dayalı algılama dâhil)**

- Araç çağrısı kimliği temizleme: strict9 (alfasayısal, uzunluk 9).

**OpenRouter Gemini**

- Düşünce imzası temizleme: base64 olmayan `thought_signature` değerlerini
  kaldırır (base64 değerlerini korur).

**OpenRouter Anthropic**

- Akıl yürütme etkinleştirildiğinde, doğrulanmış OpenRouter OpenAI uyumlu
  Anthropic model yüklerinin sonundaki asistan ön dolgu turları kaldırılır;
  bu, doğrudan Anthropic ve Cloudflare Anthropic yeniden oynatma davranışıyla
  eşleşir.

**Diğer her şey**

- Yalnızca görüntü temizleme.

---

## Geçmiş davranış (2026.1.22 öncesi)

OpenClaw, 2026.1.22 sürümünden önce birden çok transkript hijyeni katmanı
uyguluyordu:

- Her bağlam oluşturma işleminde bir **transcript-sanitize uzantısı** çalışıyordu ve şunları yapabiliyordu:
  - Araç kullanımı/sonucu eşleştirmesini onarmak.
  - Araç çağrısı kimliklerini temizlemek (`_`/`-` öğelerini koruyan
    katı olmayan bir mod dâhil).
- Çalıştırıcı ayrıca sağlayıcıya özgü temizleme gerçekleştiriyordu; bu da
  işlerin yinelenmesine yol açıyordu.
- Sağlayıcı politikasının dışında, kalıcı depolamadan önce asistan metninden
  `<final>` etiketlerinin kaldırılması, boş asistan hata iletilerinin atılması ve araç
  çağrılarından sonra asistan içeriğinin kırpılması dâhil olmak üzere ek
  değişiklikler yapılıyordu.

Bu karmaşıklık, sağlayıcılar arasında gerilemelere (özellikle
`openai-responses` `call_id|fc_id` eşleştirmesinde) neden oldu. 2026.1.22 temizliği
uzantıyı kaldırdı, mantığı çalıştırıcıda merkezileştirdi ve OpenAI'ı görüntü temizleme
dışında **değişiklik yapılmaz** hâle getirdi.

## İlgili

- [Oturum yönetimi](/tr/concepts/session)
- [Oturum budama](/tr/concepts/session-pruning)
