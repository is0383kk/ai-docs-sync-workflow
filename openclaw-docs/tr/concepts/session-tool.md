---
read_when:
    - Agentın hangi oturum araçlarına sahip olduğunu anlamak istiyorsunuz
    - Oturumlar arası erişimi veya alt aracıların başlatılmasını yapılandırmak istiyorsunuz
    - Başlatılan alt ajanların durumunu incelemek istiyorsunuz
summary: Oturumlar arası durum, hatırlama, mesajlaşma ve alt ajan orkestrasyonu için ajan araçları
title: Oturum araçları
x-i18n:
    generated_at: "2026-07-26T23:38:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ceaf48addc9fc57afe2f6428cda03ed8b19f4efce93b13b58b7ef493a41c62fe
    source_path: concepts/session-tool.md
    workflow: 16
---

OpenClaw, aracılara oturumlar arasında çalışmak, durumu incelemek ve alt aracıları orkestre etmek için araçlar sunar.

## Kullanılabilir araçlar

| Araç                 | İşlevi                                                                |
| -------------------- | --------------------------------------------------------------------------- |
| `sessions`           | Görünür oturum ayarlarını yamalar ve genel oturum grubu kataloğunu yönetir  |
| `sessions_list`      | Oturumları isteğe bağlı filtrelerle (tür, etiket, aracı, arşiv, önizleme) listeler  |
| `sessions_search`    | Görünür oturum dökümlerinde arama yapar ve eşleşen alıntıları döndürür             |
| `sessions_history`   | Belirli bir oturumun dökümünü okur                                   |
| `sessions_send`      | Aynı Gateway üzerinde başka bir oturum çalıştırır ve isteğe bağlı olarak bekler                 |
| `conversations_list` | Kararlı harici konuşma adreslerini listeler                                 |
| `conversations_send` | Yerel bir oturum çalıştırmadan tam olarak belirtilen bir harici konuşmaya gönderir     |
| `conversations_turn` | Tam olarak belirtilen bir harici konuşmaya gönderir ve ilişkili yanıtını bekler   |
| `sessions_spawn`     | Arka plan çalışması için yalıtılmış bir alt aracı oturumu başlatır                     |
| `sessions_yield`     | Geçerli turu sonlandırır ve sonraki alt aracı sonuçlarını bekler               |
| `subagents`          | Bu oturum ağacındaki arka plan çalışmalarını listeler veya iptal eder                         |
| `session_status`     | `/status` tarzında bir kart gösterir ve isteğe bağlı olarak oturuma özel bir model geçersiz kılması ayarlar |

Bu araçlar yine de etkin araç profiline ve izin/verme politikasına tabidir. `tools.profile: "coding"`, tam oturum orkestrasyonu kümesini içerir. `tools.profile: "messaging"`; oturum öz yönetimini, keşfi, hatırlamayı, oturumlar arası mesajlaşmayı, harici konuşma araçlarını ve eksiksiz başlatma yaşam döngüsünü (`sessions_spawn`, `sessions_yield` ve `subagents`) içerir. Yalnızca kullanıcı arayüzüne yönelik görev önerisi araçları `spawn_task` ve `dismiss_task`, kodlama profili araçları olarak kalır.

Grup, sağlayıcı, korumalı alan ve aracı başına politikalar, profil aşamasından sonra da bu araçları kaldırabilir. Etkili araç listesini incelemek için etkilenen oturumdan `/tools` kullanın.

## Oturumları listeleme ve okuma

`sessions_list`, odaklanmış keşif satırları döndürür: oturum anahtarı, aracı, tür, kanal, etiket/başlık/önizleme alanları, üst ve alt ilişkileri, son güncelleme, arşiv/sabitleme durumu, durum sürümü, model, bağlam/toplam token sayıları, çalıştırma durumu ve son çalıştırmanın yarıda kesilip kesilmediği. `kinds` (dizi; kabul edilen değerler: `main`, `group`, `cron`, `hook`, `node`, `other`), tam `label`, tam `agentId`, `search` metni veya güncellik (`activeMinutes`) ölçütlerine göre filtreleyin. Varsayılan olarak etkin oturumlar döndürülür; bunun yerine arşivlenmiş oturumları incelemek için `archived: true` iletin. Posta kutusu tarzı önceliklendirme gerektiğinde `includeDerivedTitles`, `includeLastMessage` veya `messageLimit` (en fazla 20) ayarlayın: görünürlük kapsamlı türetilmiş bir başlık, son mesajdan bir önizleme parçacığı veya her satırda sınırlı sayıda yakın tarihli mesaj. Teslimat yönlendirmesi, dahili oturum kimlikleri, çalıştırma başına zamanlamalar/ayarlar, maliyet tahminleri ve döküm yolları kasıtlı olarak dahil edilmez; bu sahip özelindeki ayrıntılar için `session_status`, konuşma araçları ve `sessions_history` kullanın. Türetilmiş başlıklar ve önizlemeler yalnızca çağıranın yapılandırılmış oturum aracı görünürlük politikası kapsamında zaten görebildiği oturumlar için üretilir; böylece ilgisiz oturumlar gizli kalır. Görünürlük kısıtlandığında `sessions_list`, etkili modu ve sonuçların kapsamla sınırlı olabileceğine dair bir uyarıyı gösteren isteğe bağlı `visibility` meta verilerini döndürür.

`sessions_history`, belirli bir oturumun konuşma dökümünü getirir. Varsayılan olarak araç sonuçları hariç tutulur; bunları görmek için `includeTools: true` iletin. Sınırlandırılmış en yeni kuyruk için `limit` kullanın. Sayfalama meta verilerine ihtiyaç duyduğunuzda `offset: 0` iletin; ardından ham döküm dosyalarını okumadan eski OpenClaw döküm pencerelerinde geriye doğru sayfalamak için döndürülen `nextOffset` değerlerini iletin. Açık ofsetli sayfalar harici CLI geri dönüş içe aktarımlarını birleştirmez; bu birleştirilmiş görüntüleme geçmişine ihtiyaç duyduğunuzda varsayılan en yeni kuyruk görünümünü (`offset` olmadan) kullanın.

Döndürülen görünüm kasıtlı olarak sınırlandırılmış ve güvenlik filtresinden geçirilmiştir:

- assistant metni hatırlamadan önce normalleştirilir:
  - thinking etiketleri kaldırılır
  - `<relevant-memories>` / `<relevant_memories>` iskele blokları kaldırılır
  - `<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>` ve `<function_calls>...</function_calls>` gibi düz metin araç çağrısı XML yük blokları, hiçbir zaman düzgün kapanmayan kesilmiş yükler dahil olmak üzere kaldırılır
  - `[Tool Call: ...]`, `[Tool Result ...]` ve `[Historical context ...]` gibi indirgenmiş araç çağrısı/sonuç iskeleleri kaldırılır
  - `<|assistant|>`, diğer ASCII `<|...|>` token'ları ve tam genişlikli `<｜...｜>` çeşitleri gibi sızmış model denetim token'ları kaldırılır
  - `<invoke ...>` / `</minimax:tool_call>` gibi hatalı biçimlendirilmiş MiniMax araç çağrısı XML'i kaldırılır
- kimlik bilgisi/token benzeri metin döndürülmeden önce sansürlenir
- uzun metin blokları kesilir
- çok büyük geçmişlerde eski satırlar atılabilir veya aşırı büyük bir satır `[sessions_history omitted: message too large]` ile değiştirilebilir
- araç; `truncated`, `droppedMessages`, `contentTruncated`, `contentRedacted`, `bytes` gibi özet işaretlerini ve sayfalama meta verilerini bildirir

Döndürülen **oturum anahtarını** (`"main"` gibi) `sessions_history`, `sessions_send` ve `session_status` ile kullanın. Bu hedef araçlar bilinen bir oturum kimliğini de çözümleyebilir, ancak `sessions_list` dahili kimlikleri açığa çıkarmaz.

Tam ham döküme ihtiyacınız varsa `sessions_history` öğesini filtresiz bir döküm olarak değerlendirmek yerine kapsamlı SQLite döküm satırlarını inceleyin.

Görünür kullanıcı ve assistant döküm metinleri genelinde tam metin hatırlaması için [`sessions_search`](/tr/concepts/session-search) kullanın. Sonuçları, sonraki bir `sessions_history` çağrısı için bir `sessionKey` içerir; görünürlük filtrelemesi, parçacık sansürleme ve çıktı sınırları geçmiş sınırıyla eşleşir.

## Oturum ayarlarını ve grupları yönetme

Sahip denetimli `sessions` aracı, iki sınırlandırılmış öz yönetim yüzeyi sunar:

- `action: "patch"` varsayılan olarak geçerli oturumu veya `sessionKey` tarafından seçilen başka bir görünür oturumu değiştirir. Etiketi, kenar çubuğu simgesini, sabitleme/arşiv durumunu, modeli ve düşünme düzeyini ayarlayabilir. Sıfırlama, silme veya compact eylemlerini kullanıma açmaz.
- `group_list`, `group_set`, `group_rename` ve `group_delete`, genel sıralı oturum grubu kataloğunu yönetir. `group_set`, tek bir girdiyi yamamak yerine sıralı ad listesini değiştirir.

Aracı tarafından seçilen bir model yaması, bu seçim başarılı bir çalıştırmayı tamamlayana kadar geri alınabilir durumda kalır. Seçilen model kimlik doğrulama, faturalandırma veya model bulunamadı hatası nedeniyle kesin olarak kullanılamıyorsa OpenClaw önceki modeli geri yükler ve görünür bir sistem notu yazar. Geçici hız sınırı, aşırı yük, zaman aşımı, ağ ve sunucu hataları seçimi geri almaz.

## Oturumlar ve konuşmalar

Bir **oturum**, yerel model bağlamıdır. Bir **konuşma**, bir eş, kanal veya ileti dizisi gibi tam bir harici adrestir. İkisi bağlantılıdır ancak birbirinin yerine kullanılamaz: doğrudan mesajlar ayrı konuşma adreslerini korurken tek bir `main` oturumunu paylaşabilir.

`conversations_list`, etkin aracı için opak `conversationRef` değerleri döndürür. Açık bir `channel` sağlandığında Gateway, onaylanmış Reef eşleri gibi adresleri o kanalın yerel dizininden de yeniler; geçerli sonuç sayfasının ötesinde belirli bir eşi bulmak için `query` kullanın. Keşif, model bağlamı oturumu oluşturmadan adresi kataloglar; destekleyen oturum yalnızca teslimat veya gelen bağlam buna ihtiyaç duyduğunda oluşturulur. Konuşma keşfi ve teslimatı, Gateway'in kanal kimlik bilgilerini kullandıkları için yalnızca sahibe açıktır. Gönder ve unut teslimatı için `conversations_send` kullanın. Uzak yanıt geçerli model turuna ait olduğunda `conversations_turn` kullanın: Gateway tek bir taşıma mesaj kimliği ayırır, taşıma G/Ç'sinden önce bir teslimat işlemini ve kuyruk niyetini kalıcılaştırır ve ikinci bir yerel aracı turu başlatmak yerine ilişkili yanıtı araçtan döndürür. Teslimat işlemleri model dökümlerinin dışında bulunur; yakalanan yanıt yalnızca araç sonucu model bağlamının sahibi olduğu sürece bir yan eser olarak tutulur. Gateway kuyruğa alma işleminden sonra yeniden başlatılırsa teslimat kurtarılabilir, ancak işlem yerelindeki bekleyici artık bulunmadığı için daha sonraki bir yanıt olağan gelen gönderim yolunu izler. İstenmeyen gelen mesajlar her zaman normal kanal gönderim yolu üzerinden devam eder.

Zaten açık bir ham kanal hedefiniz varsa veya kanala özgü bir eyleme ihtiyaç duyuyorsanız paylaşılan `message` aracını kullanın. Konuşma başvuruları etkin aracıyla sınırlıdır ve oturum anahtarlarından oluşturulmamalı, `conversations_list` üzerinden edinilmelidir.

Code Mode'da konuşma araçları tam Gateway çıktı sözleşmelerini yeniden kullanır. Tek bir `exec` hücresi adresleri listeleyebilir, döndürülen bir `conversationRef` seçebilir ve `conversations_send` veya `conversations_turn` çağrısı yapabilir; normal araç politikası ve onaylar iç içe çağrılar için de geçerlidir.

## Oturumlar arası mesaj gönderme

`sessions_send`, aynı Gateway üzerinde başka bir oturum çalıştırır ve isteğe bağlı olarak yanıtı bekler. Aracın `sessionKey`, `label` veya `agentId` değeri harici bir hedefi değil, yerel model bağlamını seçer. Ortaya çıkan yanıt yine de yerleşik istekte bulunan veya hedef teslimat bağlamı üzerinden duyurulabilir; bu mevcut davranış değişmemiştir. Tam harici teslimat için bir konuşma aracını veya açık bir kanal ve hedefle `message` kullanın.

- **Gönder ve unut:** Kuyruğa almak ve hemen dönmek için `timeoutSeconds: 0` ayarlayın.
- **Yanıtı bekle:** Bir zaman aşımı ayarlayın ve yanıtı satır içinde alın.

Anahtarları `:thread:<id>` ile bitenler gibi ileti dizisi kapsamlı sohbet oturumları, geçerli `sessions_send` hedefleri değildir. Araç üzerinden yönlendirilen mesajların etkin ve insanlara yönelik bir ileti dizisinde görünmemesi için aracı koordinasyonunda üst kanal oturumu anahtarını kullanın.

Mesajlar ve A2A takip yanıtları, alıcı isteminde (`[Inter-session message ... isUser=false]`) ve döküm kaynağında oturumlar arası veri olarak işaretlenir. Alıcı aracı bunları doğrudan son kullanıcı tarafından yazılmış talimatlar olarak değil, araç üzerinden yönlendirilmiş veriler olarak değerlendirmelidir.

Hedef yanıt verdikten sonra OpenClaw, aracıların yerleşik sınıra kadar sırayla mesaj gönderdiği bir **geri yanıtlama döngüsü** çalıştırabilir. Hedef aracı erken durmak için `REPLY_SKIP` yanıtını verebilir.

Göndereni hedefteki durum değişikliklerinin izleyicisi olarak da kaydetmek için `watch: true` iletin: başka bir aktör daha sonra hedefe doğrudan bir insan mesajı gönderdiğinde veya hedefin amacını değiştirdiğinde, gönderen `session_status` `changesSince` öğesine işaret eden bir sistem bildirimi alır. Kayıt başarılı gönderimden sonra gerçekleşir, mesajı gerçekten alan oturumu hedefler ve geçerli durum sürümünden başlar; dolayısıyla yalnızca sonraki değişiklikler bildirim üretir. Sonuç, kayıt başarılı olduğunda `watched: true` bildirir. Bkz. [Oturum durumu farkındalığı](/tr/concepts/session-state).

## Durum ve orkestrasyon yardımcıları

`session_status`, geçerli veya başka bir görünür oturum için hafif `/status` eşdeğeri araçtır. Kullanımı, zamanı, model/çalışma zamanı durumunu ve varsa bağlantılı arka plan görevi bağlamını bildirir. `/status` gibi, seyrek token/önbellek sayaçlarını en son döküm kullanım girdisinden tamamlayabilir ve `model=default` oturuma özel geçersiz kılmayı temizler. Çağıranın geçerli oturumu için `sessionKey="current"` kullanın; `openclaw-tui` gibi görünür istemci etiketleri oturum anahtarı değildir.

Rota meta verileri kullanılabilir olduğunda, `session_status` ayrıca görünür bir `Route context` JSON bloğu ve bununla eşleşen yapılandırılmış `details` alanlarını içerir. Bu alanlar, oturum anahtarını canlı çalıştırmayı şu anda işleyen rotadan ayırt eder:

- `origin`, oturumun oluşturulduğu yerdir veya eski durumda kayıtlı kaynak meta verileri bulunmadığında, teslim edilebilir bir oturum anahtarı ön ekinden çıkarılan sağlayıcıdır.
- `active`, mevcut canlı çalıştırma rotasıdır. Yalnızca şu anda işlenen canlı veya mevcut oturum için bildirilir.
- `deliveryContext`, oturumda saklanan kalıcı teslimat rotasıdır; OpenClaw, etkin yüzey farklı olduğunda bile daha sonraki teslimatlar için bunu yeniden kullanabilir.

## Oturum durumu değişiklikleri

OpenClaw, önemli oturum durumu değişikliklerinin (izlenen oturumlara gönderilen doğrudan insan mesajları, alt çalışma sonuçları, hedef değişiklikleri, Compaction) kalıcı bir sinyal günlüğünü tutar. `sessions_list` satırları ve `session_status`, oturumun `stateVersion` değerini sunar; `session_status` ise bu sürümden sonraki türü belirlenmiş olayları döndürmek için `changesSince: <version>` kabul eder ve istenen sürüm saklanan geçmişten daha eski olduğunda bunu tam olarak `historyGap` ile bildirir. İzleyiciler — oluşturma üst öğeleri otomatik olarak, `sessions_send watch: true` açıkça — başka bir aktör izlenen bir oturumu değiştirdiğinde birleştirilmiş tek bir eski durum bildirimi alır.

Durum değişikliği olayları, yinelenen oturum/ajan kimliklerini atlar ve yalnızca model için yararlı yük alanlarını (`outcome`, `channel` veya `turns`) sunar. Olay özeti ve aktör/çalışma tanımlayıcıları uzlaştırma için kullanılabilir kalır.

Tam model için [Oturum durumu farkındalığı](/tr/concepts/session-state) bölümüne bakın: olay türleri, izleyici kaydı, istenmeyen bildirimleri önleme protokolü, uzlaştırma akışı ve mevcut sınırlar.

`sessions_yield`, bir sonraki mesajın beklediğiniz takip olayı olabilmesi için mevcut turu kasıtlı olarak sonlandırır. Tamamlanma sonuçlarının yoklama döngüleri oluşturmak yerine bir sonraki mesaj olarak gelmesini istediğinizde, alt ajanları oluşturduktan sonra bunu kullanın.

`subagents`, yerel alt ajan çalışmaları ve paylaşılan arka plan görevi defteri üzerindeki oturum ağacı görünümüdür. `action: "list"`, etkin/yakın tarihli alt ajanların yanı sıra kapsamlandırılmış ACP, CLI/medya ve Cron görevlerini bildirir. `action: "cancel"`, döndürülen bir `taskId` değerini kabul eder ve yalnızca çağıranın denetimindeki oturum ağacında bulunan çalışmaları durdurabilir; yaprak alt ajanlar başka bir oturumun görevini iptal edemez.

## Alt ajanları oluşturma

`sessions_spawn`, varsayılan olarak bir arka plan görevi için yalıtılmış bir oturum oluşturur. Her zaman engellemesizdir; hemen bir `runId` ve `childSessionKey` döndürür. Yerel alt ajan çalışmaları, devredilen görevi alt oturumun ilk görünür `[Subagent Task]` mesajında alırken sistem istemi yalnızca alt ajan çalışma zamanı kurallarını ve yönlendirme bağlamını taşır.

Temel seçenekler:

- Yerel alt ajanlar için `runtime: "subagent"` (varsayılan) veya harici çalıştırma düzeni ajanları için `"acp"`.
- Alt oturum için `model` ve `thinking` geçersiz kılmaları.
- Oluşturmayı bir sohbet iş parçacığına (Discord, Slack vb.) bağlamak için `thread: true`.
- Alt oturumda korumalı alan kullanımını zorunlu kılmak için `sandbox: "require"`.
- Alt oturumun mevcut istekte bulunanın dökümüne ihtiyaç duyduğu yerel alt ajanlar için `context: "fork"`; temiz bir alt oturum için bunu atlayın veya `context: "isolated"` kullanın. `context: "fork"` yalnızca `runtime: "subagent"` ile geçerlidir. İş parçacığına bağlı yerel alt ajanlar, `threadBindings.defaultSpawnContext` aksini belirtmediği sürece varsayılan olarak `context: "fork"` kullanır.
- Gizli bir alt ajan oturumu yerine kalıcı bir pano oturumu oluşturmak için `visible: true`. Görünür oluşturmalar; açık bir modeli, çalışma dizinini, aynı ajana ait döküm çatallamasını ve isteğe bağlı bir [yönetilen çalışma ağacını](/tr/concepts/managed-worktrees) destekler; kesin uyumluluk sınırları için [Alt ajanlar](/tr/tools/subagents#tool-parameters) bölümüne bakın.

Varsayılan yaprak alt ajanlar oturum araçlarını almaz. `maxSpawnDepth >= 2` olduğunda, derinlik-1 düzenleyici alt ajanlar kendi alt öğelerini yönetebilmeleri için ek olarak `sessions_spawn`, `subagents`, `sessions_list` ve `sessions_history` alır. Yaprak çalışmaları yinelemeli düzenleme araçlarını yine de almaz.

Tamamlanmadan sonra bir duyuru adımı, sonucu istekte bulunanın kanalına gönderir. Tamamlanma teslimatı, mevcut olduğunda bağlı iş parçacığı/konu yönlendirmesini korur ve tamamlanma kaynağı yalnızca bir kanalı tanımlıyorsa OpenClaw doğrudan teslimat için istekte bulunanın oturumunda saklanan rotayı (`lastChannel` / `lastTo`) yine de yeniden kullanabilir.

ACP'ye özgü davranış için [ACP Ajanları](/tr/tools/acp-agents) bölümüne bakın.

## Görünürlük

Oturum araçları, ajanın görebileceklerini sınırlayacak şekilde kapsamlandırılır:

| Düzey   | Kapsam                                                      |
| ------- | ---------------------------------------------------------- |
| `self`  | Yalnızca mevcut oturum                                   |
| `tree`  | Mevcut + oluşturulanlar; okumalar izlenen aynı ajan gruplarını içerir |
| `agent` | Bu ajanın tüm oturumları                                |
| `all`   | Tüm oturumlar (yapılandırılmışsa ajanlar arası)                   |

Varsayılan değer `tree` şeklindedir. Korumalı alan oturumları, yapılandırmadan bağımsız olarak `tree` ile sınırlandırılır.
Varsayılan `session.dmScope: "main"` ile grup etkinliği, izlenen
aynı ajan grup oturumlarının ana oturumdan okunabilmesini sağlar.

## Ek okumalar

- [Oturum Yönetimi](/tr/concepts/session): yönlendirme, yaşam döngüsü, bakım
- [Alt ajanlar](/tr/tools/subagents): alt oturum yaşam döngüsü ve teslimatı
- [ACP Ajanları](/tr/tools/acp-agents): harici çalıştırma düzeni oluşturma
- [Çoklu ajan](/tr/concepts/multi-agent): çoklu ajan mimarisi
- [Gateway Yapılandırması](/tr/gateway/configuration): oturum aracı yapılandırma ayarları

## İlgili

- [Oturum yönetimi](/tr/concepts/session)
- [Oturum budama](/tr/concepts/session-pruning)
