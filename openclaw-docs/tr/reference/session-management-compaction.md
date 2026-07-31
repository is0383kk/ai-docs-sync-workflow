---
read_when:
    - Oturum kimliklerinde, transkript olaylarında veya oturum satırı alanlarında hata ayıklamanız gerekiyor
    - Otomatik Compaction davranışını değiştiriyor veya "Compaction öncesi" bakım işlemleri ekliyorsunuz
    - Bellek boşaltmalarını veya sessiz sistem turlarını uygulamak istiyorsunuz
summary: 'Derinlemesine inceleme: oturum deposu ve transkriptler, yaşam döngüsü ve (otomatik) Compaction iç işleyişi'
title: Oturum yönetimine derinlemesine bakış
x-i18n:
    generated_at: "2026-07-27T00:16:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ae02d49245768831abd17e1c2e5adacfa1a36673cef2a8a7a06a5300392b104
    source_path: reference/session-management-compaction.md
    workflow: 16
---

Tek bir **Gateway işlemi**, oturum durumunu uçtan uca yönetir. Kullanıcı arayüzleri (macOS uygulaması, web Control UI, TUI), oturum listelerini ve token sayılarını Gateway'den sorgular. Uzak modda oturum dosyaları uzak ana makinede bulunur; bu nedenle yerel Mac'inizdeki dosyaları denetlemek, Gateway'in kullandıklarını yansıtmaz.

Önce genel bakış belgeleri: [Oturum yönetimi](/tr/concepts/session), [Compaction](/tr/concepts/compaction), [Belleğe genel bakış](/tr/concepts/memory), [Bellek araması](/tr/concepts/memory-search), [Oturum budama](/tr/concepts/session-pruning), [Transkript düzeni](/tr/reference/transcript-hygiene), tam yapılandırma başvurusu için [Aracı yapılandırması](/tr/gateway/config-agents).

## İki kalıcılık katmanı

1. **Oturum satırları (aracı başına SQLite)** - anahtar/değer eşlemesi `sessionKey -> SessionEntry`. Gateway'in yönettiği değiştirilebilir çalışma zamanı durumu. Meta verileri izler: geçerli oturum kimliği, son etkinlik, açma/kapama ayarları, token sayaçları.
2. **Transkript olayları (aracı başına SQLite)** - yalnızca eklemeli, ağaç yapılıdır (girdilerde `id` + `parentId` bulunur). Konuşmayı, araç çağrılarını ve Compaction özetlerini depolar; gelecekteki dönüşler için model bağlamını yeniden oluşturur. Compaction denetim noktaları, sıkıştırılmış ardıl transkript üzerindeki meta verilerdir; yeni bir Compaction, ikinci bir `.checkpoint.*.jsonl` kopyası yazmaz.

Eski kurulumlarda aracı `sessions/` dizini altında hâlâ `sessions.json` dosyaları bulunabilir. Bu dosyaları eski oturum satırı geçiş girdileri veya açık çevrimdışı bakım hedefleri olarak değerlendirin. Gateway başlangıcı ve `openclaw doctor --fix`, etkin eski satırları ve transkript geçmişini otomatik olarak aracı başına SQLite deposuna aktarır. Açık inceleme veya doğrulama kanıtına ihtiyaç duyduğunuzda `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` komutunu çalıştırın, ardından [Doctor geçiş sırasını](/tr/cli/doctor#session-sqlite-migration) izleyin. Eski transkript yapıtları arşivlendikten sonra geçiş başarısız olursa bu sıradaki Doctor kurtarma modunu kullanın. Kurtarma; geçiş bildirimlerini kullanır, yalnızca etkilenen arşivlenmiş destek yapıtlarını geri yükler, istendiğinde temizlenmiş bir GitHub sorun raporu hazırlar ve etkin çalışma zamanının JSONL dosyalarını yeniden okumasına yol açmaz.

Gateway geçmiş okuyucuları, yüzey keyfî geçmiş erişimi gerektirmedikçe transkriptin tamamını bellekte oluşturmaz. İlk sayfa geçmişi, gömülü sohbet geçmişi, yeniden başlatma kurtarması ve token/kullanım denetimleri SQLite'tan sınırlı kuyruk okumaları kullanır. Tam transkript taramaları eşzamansız transkript dizini üzerinden gerçekleştirilir ve eşzamanlı okuyucular arasında paylaşılır.

## Disk üzerindeki konumlar

Gateway ana makinesinde aracı başına (`src/config/sessions.ts` aracılığıyla çözümlenir):

- Çalışma zamanı oturum satırı deposu: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- Çalışma zamanı transkript satırları: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- Eski/arşiv transkript yapıtları: `~/.openclaw/agents/<agentId>/sessions/`
- Eski satır geçiş girdisi: `~/.openclaw/agents/<agentId>/sessions/sessions.json`

## Depo bakımı ve disk denetimleri

`session.maintenance`, SQLite oturum satırları, SQLite transkript satırları, arşiv yapıtları ve yörünge yan dosyaları için otomatik bakımı denetler:

| Anahtar                 | Varsayılan            | Notlar                                                                                      |
| ----------------------- | --------------------- | ------------------------------------------------------------------------------------------- |
| `mode`                  | `"enforce"`           | veya `"warn"` (yalnızca rapor, değişiklik yok)                                             |
| `pruneAfter`            | `"30d"`               | eski girdi yaşı eşiği                                                                       |
| `maxEntries`            | `500`                 | oturum girdisi üst sınırı                                                                   |
| `resetArchiveRetention` | tut (yaş eşiği yok)   | `*.reset.*`/`*.deleted.*` transkript arşivlerinin yaş eşiği; süre belirtilmesi silmeyi etkinleştirir |
| `maxDiskBytes`          | `10gb`                | aracı başına oturum disk bütçesi; `false` devre dışı bırakır                              |
| `highWaterBytes`        | `maxDiskBytes` değerinin %80'i | bütçe temizliğinden sonraki hedef                                                            |

Sıfırlama, etkin `sessionKey -> sessionId` eşlemesini ilerletir ancak önceki SQLite oturum, transkript, yörünge ve arama satırlarını korur. Bu geçmiş aynı oturum anahtarı altında aranabilir durumda kalır; normal girdi ve oturum listeleri yalnızca yeni etkin eşlemeyi gösterir. Korunan sıfırlama geçmişi, yalnızca arşiv yapıtlarını yaşlandıran `resetArchiveRetention` ile değil, disk bütçesiyle sınırlandırılır. Açık silme farklıdır: silinen oturumun satırlarını kaldırmadan önce sıkıştırılmış bir transkript arşivi yazar ve doğrular (zstd kullanılabiliyorsa `*.jsonl.deleted.<timestamp>.zst`).

`maxDiskBytes` uygulaması fiziksel baytları kullanır: aracı başına SQLite ana dosyası, bunun `-wal` dosyası ve aracı oturumları dizinindeki sayılan dosyalar. Satır JSON boyutlarını hiçbir zaman tahmin etmez veya mantıksal satır boyutlarını bu toplamdan çıkarmaz.

Gateway model çalıştırma yoklama oturumları (`agent:*:explicit:model-run-<uuid>` ile eşleşen anahtarlar) ayrı, sabit bir `24h` saklama süresine sahiptir. Bu budama baskı koşuluna bağlıdır: yalnızca oturum girdisi bakımına/üst sınırına ilişkin baskıya ulaşıldığında ve yalnızca genel eski girdi temizliği/üst sınır adımından önce çalışır. Diğer açık oturumlar bu saklama süresini kullanmaz.

Birleşik fiziksel kullanım `maxDiskBytes` değerini aştığında `mode: "enforce"`, önce denetim noktası oluşturulabilir veritabanı alanını geri kazanır, ardından korunan en eski sıfırlama/silme arşivlerini kaldırır. Kullanım hâlâ `highWaterBytes` değerinin üzerindeyse geçmiş SQLite oturumlarını `sessions.updated_at` değerine göre en eskiden başlayarak tarar. Geçmiş oturum, oturum kimliğine etkin bir oturum girdisi, rota hedefi veya kabul edilmiş/devam eden bir çalıştırma tarafından başvurulmaması anlamına gelir. Temizlik, her hedef için sıkıştırılmış arşivi yazar, fsync uygular ve geri okur; ardından bir yazma işlemi oturum satırını ve bunun transkript, yörünge, etkin, dizin ve FTS izdüşümlerini kaldırır. Buna yörünge olayları içerip transkript olayları içermeyen oturumlar da dahildir. Temizlik, silme sırasında rota, girdi ve kabul başvurularını yeniden denetler; her arşiv veya oturum hedefinden sonra fiziksel kullanımı yeniden ölçer ve `highWaterBytes` değerinde durur.

Kesinleşmiş yazma ve silme işlemleri önce WAL'a kaydedilir. Temizlik, WAL'ın hemen küçülebilmesi için denetim noktası oluşturur; ardından ana dosyanın geri alınmaya uygun boş kuyruk sayfalarını iade etmek üzere artımlı vakum kullanır. Henüz geri alınamayan sayfalar ana dosyada kalır ve bu nedenle sonraki fiziksel ölçümde sayılmaya devam eder. `mode: "warn"`, denetim noktası oluşturmadan, arşiv yazmadan veya satır silmeden mevcut fiziksel aşımı bildirir.

Bakımı isteğe bağlı olarak çalıştırın:

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

Bakım, grup oturumları ve iş parçacığı kapsamlı sohbet oturumları gibi kalıcı haricî konuşma işaretçilerini korur; ancak sentetik çalışma zamanı girdileri (cron, kancalar, Heartbeat, ACP, alt aracılar), yapılandırılmış yaşı, sayıyı veya disk bütçesini aştıklarında yine kaldırılabilir. Yalıtılmış cron çalıştırmaları, model çalıştırma yoklaması saklama süresinden bağımsız, ayrı bir `cron.sessionRetention` denetimi kullanır.

Normal Gateway yazmaları, aracı başına SQLite değişikliklerini çalışma zamanı yazıcı yolu üzerinden seri hâle getiren oturum erişimcisi üzerinden akar. Çalışma zamanı kodu `src/config/sessions/session-accessor.ts` içindeki erişimci yardımcılarını tercih etmelidir; eski `sessions.json` yardımcıları geçiş ve çevrimdışı bakım araçlarıdır. Bir Gateway'e erişilebildiğinde, kuru çalıştırma olmayan `openclaw sessions cleanup` ve `openclaw agents delete`, depo değişikliklerini Gateway'e devrederek temizliğin aynı yazıcı kuyruğuna katılmasını sağlar; `--store <path>`, seçilen eski depo için açık çevrimdışı onarım yoludur ve her zaman yerel kalır (`--dry-run` de öyle). `maxEntries` temizliği, üretim boyutundaki depolar için toplu yapılır; bu nedenle bir depo, sonraki yüksek su seviyesi temizliği onu yeniden üst sınırın altına indirene kadar yapılandırılmış üst sınırı kısa süreliğine aşabilir. Okumalar Gateway başlangıcı sırasında girdileri hiçbir zaman budamaz veya sınırlamaz; bunu yalnızca yazmalar ya da `openclaw sessions cleanup --enforce` yapar. İkincisi ayrıca üst sınırı hemen uygular ve disk bütçesi yapılandırılmamış olsa bile başvurulmayan eski transkript, denetim noktası ve yörünge yapıtlarını budar.

OpenClaw artık Gateway yazmaları sırasında otomatik `sessions.json.bak.*` döndürme yedekleri oluşturmaz. Geçerli şema eski `session.maintenance.rotateBytes` anahtarını reddeder ve `openclaw doctor --fix` bunu eski yapılandırmalardan kaldırır.

Transkript değişiklikleri, SQLite transkript hedefi için oturum yazma kuyruğunu kullanır:

Oturum yazma kilitleri sabit üretim varsayılanlarını kullanır. Karşılık gelen
`OPENCLAW_SESSION_WRITE_LOCK_*` ortam değişkenleri, işlem düzeyinde tanılama ve acil durum geçersiz kılmaları için kullanılabilir durumda kalır.

### SQLite Geçişinden Sonra Eski Sürüme Dönme

Dosya destekli eski bir OpenClaw sürümünü çalıştırmadan önce arşivlenmiş eski transkript yapıtlarını geri yükleyin:

```bash
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

Geçiş, destek ve geri alma için eski `sessions.json` dosyalarını yerinde bırakır; ancak SQLite'a aktarılmış etkin transkript JSONL dosyaları `session-sqlite-import-archive/` olarak yeniden adlandırılır. Dosya destekli eski çalışma zamanları `sessions.json` içindeki `sessionFile` yollarını izlediğinden, başlangıçtan önce bu yapıtların geri yüklenmesi gerekir. Geri yükleme; geçiş bildirimlerini kullanır, yalnızca özgün yolları eksik olan kayıtlı arşiv yapıtlarını taşır ve ileriye dönük kurtarma için SQLite veritabanını yerinde bırakır.

SQLite geçişinden sonra oluşturulan oturumlar yalnızca SQLite'ta bulunur ve dosya destekli eski çalışma zamanlarında görünmez. Eski sürüme döndükten sonra yeniden yükseltme yaparsanız OpenClaw'ın geri yüklenen eski yapıtları içe aktarmadan önce doğrulayabilmesi için Doctor inceleme ve doğrulama sırasını yeniden çalıştırın.

## Cron oturumları ve çalıştırma günlükleri

Yalıtılmış cron çalıştırmaları, özel saklama süresine sahip kendi oturum girdilerini/transkriptlerini oluşturur:

- `cron.sessionRetention` (varsayılan `"24h"`), eski yalıtılmış cron çalıştırma oturumlarını depodan budar; `false` devre dışı bırakır.
- Çalıştırma geçmişi, cron işi başına en yeni 2000 terminal satırını korur. Kaybolan satırlar 24 saatlik temizlik aralığını korur.

Cron zorla yeni bir yalıtılmış çalıştırma oturumu oluşturduğunda, yeni satırı yazmadan önce önceki `cron:<jobId>` oturum girdisini temizler: güvenli tercihleri (düşünme/hızlı/ayrıntılı/akıl yürütme ayarları, etiketler, görünen ad) ve kullanıcı tarafından açıkça seçilmiş model/kimlik doğrulama geçersiz kılmalarını taşır; ancak yeni bir yalıtılmış çalıştırmanın eski bir çalıştırmadan güncelliğini yitirmiş teslimat veya çalışma zamanı yetkisi devralmaması için ortam konuşma bağlamını (kanal/grup yönlendirmesi, gönderme/kuyruk ilkesi, yükseltme, kaynak, ACP çalışma zamanı bağlaması) bırakır.

## Oturum anahtarları (`sessionKey`)

Bir `sessionKey`, hangi konuşma bölümünde olduğunuzu belirler (yönlendirme + yalıtım). Kurallı ilkeler: [/concepts/session](/tr/concepts/session).

| Örüntü                      | Örnek                                                       |
| --------------------------- | ----------------------------------------------------------- |
| Ana/doğrudan sohbet (aracı başına) | `agent:<agentId>:<mainKey>` (varsayılan `main`)     |
| Grup                        | `agent:<agentId>:<channel>:group:<id>`                                          |
| Oda/kanal (Discord/Slack)   | `agent:<agentId>:<channel>:channel:<id>` veya `...:room:<id>`                  |
| Cron                        | `cron:<job.id>`                                          |
| Webhook                     | `hook:<uuid>` (geçersiz kılınmadıkça)                  |

## Oturum kimlikleri (`sessionId`)

Her `sessionKey`, geçerli bir `sessionId` kimliğine (konuşmayı sürdüren SQLite transkript kimliği) işaret eder. Karar mantığı, `src/auto-reply/reply/session.ts` içindeki `initSessionState()` konumunda bulunur.

- **Sıfırlama** (`/new`, `/reset`), söz konusu `sessionKey` için yeni bir `sessionId` oluşturur.
- **Otomatik sıfırlama yok** varsayılan ayardır. Compaction, etkin model bağlamını sınırlı tutarken mevcut `sessionId` devam eder.
- **Günlük sıfırlama** (`session.reset.mode: "daily"`), yapılandırılan yerel saat sınırından (`session.reset.atHour`, varsayılan `4`) sonraki ilk iletide yeni bir `sessionId` oluşturur.
- **Boşta kalma süresinin dolması** (`session.reset.idleMinutes` ile `session.reset.mode: "idle"` veya eski `session.idleMinutes`), boşta kalma aralığından sonra bir ileti geldiğinde yeni bir `sessionId` oluşturur. Hem günlük hem de boşta kalma sıfırlaması yapılandırılmışsa önce süresi dolan uygulanır.
- **Control UI yeniden bağlantısında sürdürme**, Gateway bir operatör UI istemcisinden eşleşen `sessionId` değerini aldığında, yeniden bağlantı sonrasında gönderilen tek bir ileti için o anda görünür olan oturumu korur. Bu tek kullanımlık bir sinyaldir; sıradan eski gönderimler yine yeni bir `sessionId` oluşturur.
- **Sistem olayları** (heartbeat, cron uyandırmaları, exec bildirimleri, gateway kayıt işlemleri) oturum satırını değiştirebilir ancak günlük/boşta kalma sıfırlamasının güncelliğini hiçbir zaman uzatmaz. Sıfırlama geçişi, yeni istem oluşturulmadan önce önceki oturum için kuyruğa alınmış sistem olayı bildirimlerini atar.
- **Üst dal çatallama ilkesi**, bir iş parçacığı veya alt ajan çatalı oluştururken OpenClaw'ın etkin dalını kullanır. Bu dal çok büyükse (sabit bir dahili sınırın üzerinde; şu anda 100K token), OpenClaw başarısız olmak veya kullanılamaz geçmişi devralmak yerine alt öğeyi yalıtılmış bağlamla başlatır. Boyutlandırma otomatiktir ve yapılandırılamaz; eski `session.parentForkMaxTokens` yapılandırması `openclaw doctor --fix` tarafından kaldırılır.
- **Operatör çatalları**: `sessions.create { parentSessionKey, fork: true }`, dökümü üst oturumun mevcut durumundan dallanan yeni bir oturum oluşturur (yukarıdaki boyut sınırı dâhil, alt ajan başlatmalarıyla aynı çatallama mekanizması). Üst oturumda etkin bir çalıştırma varken çatallama reddedilir; açıkça bir model iletilmedikçe üst oturumun model seçimi devralınır ve alt oturum `forkedFromParent` olarak, yeni token sayaçlarıyla işaretlenir.

## Oturum deposu şeması

Çalışma zamanı deposu, `SessionEntry` değerlerini ajan başına SQLite'ta tutar. Değer türü, `src/config/sessions.ts` içindeki `SessionEntry` türüdür. Temel alanlar (tümünü kapsamaz):

- `sessionId`: SQLite döküm satırlarını adreslemek için kullanılan mevcut döküm kimliği
- `sessionStartedAt`: mevcut `sessionId` için başlangıç zaman damgası; günlük sıfırlamanın güncelliği bunu kullanır. Eski satırlar bunu JSONL oturum üstbilgisinden türetebilir.
- `lastInteractionAt`: son gerçek kullanıcı/kanal etkileşiminin zaman damgası; boşta kalma sıfırlamasının güncelliği bunu kullanır, böylece heartbeat, cron ve exec olayları oturumları canlı tutmaz. Bu alanı içermeyen eski satırlar, kurtarılan oturum başlangıç zamanını kullanır.
- `updatedAt`: listeleme/budama/kayıt işlemleri için kullanılan son depo satırı değişiklik zaman damgası; günlük/boşta kalma güncelliği için yetkili kaynak değildir.
- `archivedAt`: isteğe bağlı arşiv zaman damgası. Arşivlenmiş oturumlar, dökümleri bozulmadan depoda kalır ve normal etkin listelere dâhil edilmez.
- `pinnedAt`: isteğe bağlı sabitleme zaman damgası. Etkin sabitlenmiş oturumlar, sabitlenmemiş oturumlardan önce sıralanır; bir oturumun arşivlenmesi sabitlemesini kaldırır.
- Codex iş parçacığı birlikte çalışabilirliği: her iki alan da Codex iş parçacığı yönetimi biçimini izler; aktarım sırasındaki `archived`/`pinned` boole değerleri her zaman zaman damgasından türetilir ve sunucu tarafında damgalanır; bu, Codex `threads.archived_at` semantiği ve camelCase serileştirmesiyle eşleşir. OpenClaw zaman damgaları epoch milisaniye, Codex ise epoch saniye kullandığından köprüler dönüşümü `codex` Plugin sınırında yapar. Codex henüz bir sabitleme API'sine sahip değildir (yalnızca `thread/archive`/`thread/unarchive`); böyle bir API sunulana kadar sabitlenmiş durum OpenClaw tarafında kalır. Sunulduğunda eşleşen biçim, bağlanmış oturumların sabitleme durumunu mekanik olarak gidiş dönüşlü aktarabilmesini sağlar.
- Codex gözetimi yalnızca arşivlenmemiş yerel iş parçacıklarını listeler. Gateway'e yerel bir `idle` veya etkinliği bilinmeyen `notLoaded` iş parçacığı, ancak operatör başka hiçbir Codex işleminin ona sahip olmadığını açıkça onayladıktan sonra yerel `thread/archive` aracılığıyla arşivlenebilir; Plugin önce işlem açısından yerel, güncel bir durum okuması gerçekleştirir ve ardından iş parçacığı katalogdan kaybolur. Bu okuma, başka bir App Server işleminin iş parçacığını kullanmadığını kanıtlayamaz. OpenClaw etkin ve hata durumundaki satırları arşivlemeyi reddeder; eşleştirilmiş Node arşivleme ise Node köprüsü akış hâlindeki iş parçacığının tüm yaşam döngüsünün sahipliğini üstlenene kadar kullanılamaz. Yerel bir Codex istemcisinde arşivden çıkarılan iş parçacığı yeniden görünmeye uygun hâle gelir.
- `lastReadAt` / `markedUnreadAt`: `sessions.patch { unread }` tarafından sunucu tarafında damgalanan okuma durumu zaman damgaları; `unread: false` bir okumayı kaydeder (`lastReadAt` değerini ayarlar, `markedUnreadAt` değerini temizler); `unread: true`, sonraki okumaya kadar oturumu okunmamış olarak işaretler. Oturum satırları türetilmiş bir `unread` boole değeri sunar: açıkça okunmamış olarak işaretlenmiş veya en son etkinlikten önce okunmuş. Hiçbir zaman okunmuş olarak işaretlenmeyen oturumlar `unread: false` olarak kalır; böylece mevcut kurulumlar yükseltme sonrasında uyarı göstermez.
- `lastActivityAt`: okunmamış sayılmaya değer etkinlik olarak kabul edilen, tamamlanmış son ajan çalıştırmasının zaman damgası (kullanıcı, kanal ve cron çalıştırmaları). Heartbeat ve dahili olay turları ile meta veri yamaları bunu güncellemez; `updatedAt` bir etkinlik sinyali değildir.
- `sessionFile`: geçiş/arşiv uyumluluğu için tutulan eski işaretçi; etkin çalışma zamanı SQLite kimliğini kullanır
- `chatType`: `direct | group | room`
- `provider`, `subject`, `room`, `space`, `displayName`: grup/kanal etiketleme meta verileri
- Açma/kapama seçenekleri: `thinkingLevel`, `verboseLevel`, `reasoningLevel`, `elevatedLevel`, `sendPolicy` (oturum başına geçersiz kılma)
- Model seçimi: `providerOverride`, `modelOverride`, `authProfileOverride`
- Token sayaçları (azami gayretle/sağlayıcıya bağlı): `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`
- `compactionCount`: bu oturum anahtarı için otomatik compaction işleminin kaç kez tamamlandığı
- `memoryFlushAt` / `memoryFlushCompactionCount`: compaction öncesindeki son bellek boşaltımının zaman damgası ve compaction sayısı

Yetkili kaynak Gateway'dir: oturumlar çalışırken girdileri yeniden yazabilir veya yeniden yükleyebilir. Eski dosya tabanlı kurulumlarda,
`sessions.json` dosyasını düzenleyip çalışma zamanının bu dosyayı okumaya devam etmesini beklemek yerine
`openclaw doctor --session-sqlite import --session-sqlite-all-agents` ile geçiş yapın.

## Döküm olayı yapısı

Dökümler OpenClaw oturum erişimcisi tarafından yönetilir ve kimlik tabanlı yardımcılar aracılığıyla çalışma zamanı koduna sunulur. Olay akışı yalnızca eklemelidir:

- İlk girdi: oturum üstbilgisi - `type: "session"`, `id`, `cwd`, `timestamp`, isteğe bağlı `parentSession`.
- Ardından: `id` + `parentId` içeren girdiler (ağaç yapısı).

Dikkate değer girdi türleri:

- `message`: kullanıcı/asistan/toolResult iletileri
- `custom_message`: model bağlamına _giren_, uzantı tarafından eklenmiş ileti (`display: true` olduğunda TUI'da işlenir, `display: false` olduğunda tamamen gizlenir)
- `custom`: model bağlamına _girmeyen_ uzantı durumu (uzantı durumunu yeniden yüklemeler arasında kalıcı kılmak için)
- `compaction`: `firstKeptEntryId` ve `tokensBefore` içeren kalıcı compaction özeti
- `branch_summary`: bir ağaç dalında gezinirken oluşturulan kalıcı özet

OpenClaw dökümleri bilinçli olarak "düzeltmez"; Gateway bunları okumak/yazmak için `SessionManager` kullanır.

## Bağlam pencereleri ve izlenen token'lar

İki farklı kavram:

1. **Model bağlam penceresi**: model başına kesin sınır (modelin görebildiği token'lar). Model kataloğundan gelir ve yapılandırma aracılığıyla geçersiz kılınabilir.
2. **Oturum deposu sayaçları**: oturum satırına yazılan hareketli istatistikler (`/status` ve panolar için kullanılır). `contextTokens` bir çalışma zamanı tahmin/raporlama değeridir; bunu kesin bir garanti olarak değerlendirmeyin.

Sınırlar hakkında daha fazla bilgi: [/reference/token-use](/tr/reference/token-use).

## Compaction: nedir?

Compaction, eski konuşmaları dökümde kalıcı bir `compaction` girdisi hâlinde özetler ve yakın tarihli iletileri olduğu gibi tutar. Compaction sonrasında gelecekteki turlar, compaction özetini ve `firstKeptEntryId` sonrasındaki iletileri görür. Compaction, oturum budamasının aksine **kalıcıdır**; bkz. [/concepts/session-pruning](/tr/concepts/session-pruning).

Gömülü OpenClaw compaction işlemi varsayılan olarak oturumun düşünme düzeyini devralır. Özet çağrılarında ayrı bir düzey kullanmak için `agents.defaults.compaction.thinkingLevel` ayarını yapın; çalışma zamanı bunu her somut compaction modeline veya geri dönüş modeline göre sınırlar. Yerel Codex app-server compaction işlemi kendi compact isteğinin sahibidir ve compaction başına düşünme düzeyi geçersiz kılmasını kabul edemez; bu nedenle OpenClaw uyarı verir ve bu ayarı Codex'e bırakır.

Compaction sonrasında AGENTS.md bölümünün yeniden eklenmesi, `agents.defaults.compaction.postCompactionSections` aracılığıyla isteğe bağlı olmaya devam eder. Plugin'ler `before_prompt_build` aracılığıyla başka istem bağlamları ekleyebilir.

### Parça sınırları ve araç eşleştirmesi

OpenClaw, uzun bir dökümü compaction parçalarına bölerken asistan araç çağrılarını eşleşen `toolResult` girdileriyle birlikte tutar:

- Token payı bölmesi bir araç çağrısı ile sonucu arasına denk gelecekse OpenClaw çifti ayırmak yerine sınırı asistanın araç çağrısı iletisine kaydırır.
- Sondaki bir araç sonucu bloğu parçayı aksi hâlde hedefin üzerine çıkaracaksa OpenClaw bekleyen araç bloğunu korur ve özetlenmemiş kuyruğu olduğu gibi tutar.
- İptal edilmiş/hatalı araç çağrısı blokları, bekleyen bir bölmeyi açık tutmaz.

## Otomatik compaction ne zaman gerçekleşir?

Gömülü OpenClaw ajanında iki tetikleyici vardır:

1. **Taşma kurtarması**: model bir bağlam taşması hatası döndürür (`request_too_large`, `context length exceeded`, `input exceeds the maximum number of tokens`, `input token count exceeds the maximum number of input tokens`, `input is too long for the model`, `ollama error: context length exceeded` ve sağlayıcıya özgü diğer varyantlar); compaction uygulanır, ardından yeniden denenir. Sağlayıcı denenen token sayısını bildirdiğinde OpenClaw gözlemlenen bu sayıyı taşma kurtarma compaction işlemine iletir; sağlayıcı taşmayı doğrular ancak ayrıştırılabilir bir sayı sunmazsa OpenClaw compaction motorlarına ve tanılamalara bütçeyi asgari düzeyde aşan sentetik bir sayı iletir. Taşma kurtarması yine başarısız olursa OpenClaw sessizce yeni bir oturum kimliğine geçmek yerine açık yönlendirme sunar ve mevcut oturum eşlemesini korur; iletiyi yeniden deneyin, `/compact` veya `/new` komutunu çalıştırın.
2. **Eşik bakımı**: başarılı bir turdan sonra, mevcut bağlam model penceresinden OpenClaw'ın istemler ve sonraki model çıktısı için yerleşik boşluk payı çıkarıldığında kalan değeri aştığında.

Bu iki tetikleyicinin dışında iki ek koruma çalışır:

- **Çalıştırma öncesi yerel Compaction**: etkin döküm bu boyuta ulaştığında bir sonraki çalıştırmayı açmadan önce yerel Compaction'ı tetiklemek için `agents.defaults.compaction.maxActiveTranscriptBytes` değerini (bayt veya `"20mb"` gibi bir dize) ayarlayın. Bu, ham arşivleme için değil, yerel yeniden açma maliyetine yönelik bir boyut korumasıdır; normal anlamsal Compaction çalışmaya devam eder ve sıkıştırılmış özetin yeni bir ardıl döküm hâline gelmesi için `truncateAfterCompaction` gerekir.
- **Tur ortası ön kontrolü**: araç döngüsü koruması eklemek için `agents.defaults.compaction.midTurnPrecheck.enabled: true` değerini (varsayılan `false`) ayarlayın. Bir araç sonucu eklendikten sonra ve sonraki model çağrısından önce OpenClaw, turun başında kullanılan aynı çalıştırma öncesi bütçe mantığıyla istem baskısını tahmin eder. Bağlam artık sığmıyorsa koruma satır içinde Compaction yapmaz; yapılandırılmış bir tur ortası ön kontrol sinyali oluşturur, geçerli istem gönderimini durdurur ve dış çalışma döngüsünün mevcut kurtarma yolunu kullanmasını sağlar (yeterli olduğunda aşırı büyük araç sonuçlarını kısaltır veya yapılandırılmış Compaction modunu tetikleyip yeniden dener). Sağlayıcı destekli koruma Compaction'ı dâhil olmak üzere hem `default` hem de `safeguard` Compaction modlarıyla çalışır. `maxActiveTranscriptBytes` öğesinden bağımsızdır: bayt boyutu koruması bir tur açılmadan önce çalışır, tur ortası ön kontrolü ise daha sonra, yeni araç sonuçları eklendikten sonra çalışır.

## Compaction ayarları

```json5
{
  agents: {
    defaults: {
      compaction: {
        enabled: true,
        keepRecentTokens: 20000,
      },
    },
  },
}
```

OpenClaw, gömülü çalışmalar için yerleşik bir rezerv uygular ve tüm istem bütçesini tüketememesi için bunu etkin model bağlam penceresine göre sınırlar. Bu, küçük bağlamlı yerel modellerin ilk tokenden itibaren Compaction'a girmesini önlerken bellek boşaltma gibi çok turlu bakım işlemleri için yeterli boşluk bırakır.

Manuel `/compact`, açıkça belirtilmiş bir `agents.defaults.compaction.keepRecentTokens` değerine uyar ve çalışma zamanının yakın geçmiş kuyruğu kesim noktasını korur. Açıkça belirtilmiş bir koruma bütçesi yoksa manuel Compaction kesin bir denetim noktasıdır ve yeniden oluşturulan bağlam yeni özetten başlar.

`truncateAfterCompaction` etkinleştirildiğinde OpenClaw, Compaction sonrasında etkin dökümü sıkıştırılmış bir ardıla döndürür. Dal/geri yükleme denetim noktası eylemleri bu sıkıştırılmış ardılı kullanır; eski Compaction öncesi denetim noktası dosyaları referans verildikleri sürece okunabilir kalır.

## Takılabilir Compaction sağlayıcıları

Plugin'ler, Plugin API'sindeki `registerCompactionProvider()` aracılığıyla bir Compaction sağlayıcısı kaydeder. `agents.defaults.compaction.provider` kayıtlı bir sağlayıcı kimliğine ayarlandığında koruma uzantısı, özetlemeyi yerleşik `summarizeInStages` işlem hattı yerine bu sağlayıcıya devreder.

- `provider`: kayıtlı bir Compaction sağlayıcısı Plugin'inin kimliği. Varsayılan LLM özetlemesi için ayarlamadan bırakın. Bir `provider` ayarlamak `mode: "safeguard"` kullanımını zorunlu kılar.
- Sağlayıcılar, yerleşik yolla aynı Compaction talimatlarını ve tanımlayıcı koruma politikasını alır; koruma ayrıca sağlayıcı çıktısından sonra yakın tur ve bölünmüş tur son ek bağlamını korur.
- Yerleşik koruma özetlemesi, önceki özetin tamamını olduğu gibi korumak yerine önceki özetleri yeni mesajlarla birlikte yeniden damıtır.
- Koruma modu, özet kalite denetimlerini varsayılan olarak etkinleştirir; hatalı biçimlendirilmiş çıktıda yeniden deneme davranışını atlamak için `qualityGuard.enabled: false` değerini ayarlayın.
- Sağlayıcı başarısız olursa veya boş bir sonuç döndürürse OpenClaw otomatik olarak yerleşik LLM özetlemesine geri döner. Çağıranın açıkça tetiklediği iptal/zaman aşımı sinyalleri yutulmak yerine yeniden oluşturulur; böylece iptale her zaman uyulur.

Kaynak: `src/plugins/compaction-provider.ts`, `src/agents/agent-hooks/compaction-safeguard.ts`.

## Kullanıcıya görünür yüzeyler

- Herhangi bir sohbet oturumunda `/status`
- `openclaw status` (CLI)
- `openclaw sessions` / `openclaw sessions --json`
- Gateway günlükleri (`pnpm gateway:watch` veya `openclaw logs --follow`): `embedded run auto-compaction start` + `complete`
- Ayrıntılı mod: `🧹 Auto-compaction complete` ve Compaction sayısı

## Sessiz bakım (`NO_REPLY`)

OpenClaw, kullanıcının ara çıktıları görmemesi gereken arka plan görevleri için "sessiz" turları destekler.

- Asistan, "kullanıcıya yanıt iletme" anlamına gelmesi için çıktısını tam sessiz belirteç `NO_REPLY` / `no_reply` ile başlatır. OpenClaw bunu teslimat katmanında kaldırır/baskılar.
- Tam sessiz belirteç baskılama büyük/küçük harfe duyarsızdır: tüm yük yalnızca sessiz belirteçten oluştuğunda hem `NO_REPLY` hem de `no_reply` geçerli sayılır.
- `2026.1.10` itibarıyla OpenClaw, kısmi bir parça `NO_REPLY` ile başladığında taslak/yazma akışını da baskılar; böylece sessiz işlemler turun ortasında kısmi çıktı sızdırmaz.
- Bu yalnızca gerçek arka plan/teslimatsız turlar içindir; sıradan ve eylem gerektiren kullanıcı istekleri için bir kestirme değildir.

## Compaction öncesi bellek boşaltma

Otomatik Compaction gerçekleşmeden önce OpenClaw, Compaction'ın kritik bağlamı silememesi için kalıcı durumu diske (örneğin aracının çalışma alanındaki `memory/YYYY-MM-DD.md`) yazan sessiz ve etmenli bir tur çalıştırabilir. Oturum bağlamı kullanımını izler ve kullanım Compaction eşiğinin altındaki bir yumuşak eşiği geçtiğinde, kullanıcının hiçbir şey görmemesi için tam sessiz belirteç `NO_REPLY` / `no_reply` kullanarak sessiz bir "belleği şimdi yaz" yönergesi gönderir.

Yapılandırma (`agents.defaults.compaction.memoryFlush`), tam başvuru: [/gateway/config-agents](/tr/gateway/config-agents#agentsdefaultscompaction)

| Anahtar                     | Varsayılan       | Notlar                                                                                                                                 |
| --------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                   | `true`           |                                                                                                                                        |
| `model`                     | ayarlanmamış     | yalnızca boşaltma turu için tam sağlayıcı/model geçersiz kılması, örneğin `ollama/qwen3:8b`                                             |
| `softThresholdTokens`       | `4000`           | Compaction eşiğinin altında boşaltmayı tetikleyen fark                                                                                  |
| `forceFlushTranscriptBytes` | ayarlanmamış (devre dışı) | token sayaçları güncel olmasa bile döküm dosyası bu bayt boyutuna ulaştığında (veya `"2mb"` gibi bir dizede) boşaltmayı zorunlu kılar; `0` devre dışı bırakır |

Notlar:

- Yerleşik istem ve sistem istemi, teslimatı baskılamak için bir `NO_REPLY` ipucu içerir.
- `model` ayarlandığında boşaltma turu, etkin oturumun geri dönüş zincirini devralmadan bu modeli kullanır; böylece yalnızca yerel bakım, başarısızlık durumunda sessizce ücretli bir konuşma modeline geri dönmez.
- Boşaltma, her Compaction döngüsünde bir kez çalışır (oturum satırında izlenir).
- Boşaltma yalnızca gömülü OpenClaw oturumlarında çalışır; CLI arka uçları ve Heartbeat turları bunu atlar.
- Oturum çalışma alanı salt okunur olduğunda (`workspaceAccess: "ro"` veya `"none"`) boşaltma atlanır.
- Çalışma alanı dosya düzeni ve yazma kalıpları için [Bellek](/tr/concepts/memory) bölümüne bakın.

OpenClaw, uzantı API'sinde bir `session_before_compact` kancası sunar; ancak yukarıdaki boşaltma mantığı bu kancada değil, Gateway tarafında (`src/auto-reply/reply/memory-flush.ts`, `src/auto-reply/reply/agent-runner-memory.ts`) bulunur.

## Sorun giderme kontrol listesi

- **Oturum anahtarı yanlış mı?** [/concepts/session](/tr/concepts/session) ile başlayın ve `/status` içindeki `sessionKey` değerini doğrulayın.
- **Depo ve döküm uyuşmazlığı mı var?** Gateway ana makinesini ve `openclaw status` içindeki depo yolunu doğrulayın.
- **Compaction istenmeyen şekilde tekrarlanıyor mu?** Modelin bağlam penceresini (çok küçük olması sık Compaction'ı zorunlu kılar) ve araç sonucu şişmesini (oturum budamasını ayarlayın) kontrol edin.
- **Küçük bir yerel modelde her istem taşıyor gibi mi görünüyor?** Sağlayıcının doğru model bağlam penceresini bildirdiğini doğrulayın. OpenClaw, etkili rezervi yalnızca bu pencere bilindiğinde sınırlayabilir.
- **Sessiz turlar sızıyor mu?** Yanıtın tam sessiz belirteç `NO_REPLY` ile başladığını (büyük/küçük harfe duyarsız) ve akış baskılama düzeltmesini içeren bir derlemeyi (`2026.1.10`+) kullandığınızı doğrulayın.

## İlgili konular

- [Oturum yönetimi](/tr/concepts/session)
- [Oturum budama](/tr/concepts/session-pruning)
- [Bağlam motoru](/tr/concepts/context-engine)
- [Aracı yapılandırma başvurusu](/tr/gateway/config-agents)
