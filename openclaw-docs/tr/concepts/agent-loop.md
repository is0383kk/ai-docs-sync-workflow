---
read_when:
    - Ajan döngüsünün veya yaşam döngüsü olaylarının ayrıntılı bir açıklamasına ihtiyacınız var
    - Oturum kuyruğa alma, transkript yazma veya oturum yazma kilidi davranışını değiştiriyorsunuz
summary: Ajan döngüsü yaşam döngüsü, akışları ve bekleme semantiği
title: Ajan döngüsü
x-i18n:
    generated_at: "2026-07-26T23:17:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1d0102ffb6ebf572ea0201470db138775be33b0f0b655d9d08742177be5f3f31
    source_path: concepts/agent-loop.md
    workflow: 16
---

Aracı döngüsü, bir mesajı eylemlere ve bir yanıta dönüştüren, oturum başına serileştirilmiş çalıştırmadır: alım, bağlam oluşturma, model çıkarımı, araç yürütme, akış ve kalıcılık.

## Giriş noktaları

- Gateway RPC: `agent` ve `agent.wait`.
- CLI: `openclaw agent`.

## Çalıştırma sırası

1. `agent` RPC parametreleri doğrular, oturumu çözümler (`sessionKey`/`sessionId`), oturum meta verilerini kalıcı hâle getirir ve hemen `{ runId, acceptedAt }` döndürür.
2. `agentCommand` turu çalıştırır: model + düşünme/ayrıntı/iz varsayılanlarını çözümler, Skills anlık görüntüsünü yükler, `runEmbeddedAgent` çağrısını yapar ve gömülü döngü henüz bir tane yayınlamadıysa yedek bir **yaşam döngüsü sonu/hatası** yayınlar.
3. `runEmbeddedAgent`: çalıştırmaları oturum başına ve küresel kuyruklar aracılığıyla serileştirir, model + kimlik doğrulama profilini çözümler, OpenClaw oturumunu oluşturur, çalışma zamanı olaylarına abone olur, yardımcı/araç deltalarını akışla iletir, çalıştırma zaman aşımını uygular (süre dolduğunda iptal eder) ve yüklerle birlikte kullanım meta verilerini döndürür. Codex uygulama sunucusu turlarında ayrıca, kabul edilmiş bir tur terminal olayından önce uygulama sunucusu ilerlemesi üretmeyi durdurursa turu iptal eder.
4. `subscribeEmbeddedAgentSession`, çalışma zamanı olaylarını `agent` akışına köprüler: araç olaylarını `stream: "tool"`'ye, yardımcı deltalarını `stream: "assistant"`'e, yaşam döngüsü olaylarını `stream: "lifecycle"`'e (`phase: "start" | "end" | "error"`).
5. `agent.wait` (`waitForAgentRun`), bir `runId` üzerinde **yaşam döngüsü sonu/hatasını** bekler ve `{ status: ok|error|timeout, startedAt, endedAt, error? }` döndürür.

## Kuyruklama ve eşzamanlılık

Çalıştırmalar, oturum anahtarı (oturum hattı) başına ve isteğe bağlı olarak küresel bir hat üzerinden serileştirilerek araç/oturum yarışları önlenir. Mesajlaşma kanalları, bu hat sistemini besleyen bir kuyruk modu (yönlendirme/takip/toplama/kesme) seçer; bkz. [Komut Kuyruğu](/tr/concepts/queue).

Transkript yazımları ayrıca oturum dosyasındaki bir oturum yazma kilidiyle korunur. Kilit süreç duyarlı ve dosya tabanlıdır; böylece süreç içi kuyruğu atlayan veya başka bir süreçten gelen yazıcıları yakalar. Yazıcılar, oturumu meşgul olarak bildirmeden önce varsayılan olarak 60 saniyeye kadar bekler (ortam geçersiz kılma değişkeni: `OPENCLAW_SESSION_WRITE_LOCK_ACQUIRE_TIMEOUT_MS`).

Oturum yazma kilitleri varsayılan olarak yeniden girişli değildir. Tek bir mantıksal yazıcıyı korurken aynı kilidin alınmasını kasıtlı olarak iç içe yerleştiren bir yardımcı, `allowReentrant: true` ile açıkça etkinleştirilmelidir.

## Oturum ve çalışma alanı hazırlığı

- Çalışma alanı çözümlenir ve oluşturulur; korumalı alan çalıştırmaları, bir korumalı alan çalışma alanı köküne yönlendirilebilir.
- Skills yüklenir (veya bir anlık görüntüden yeniden kullanılır) ve ortama ve isteme eklenir.
- Önyükleme/bağlam dosyaları çözümlenir ve sistem istemine eklenir.
- Akış başlamadan önce bir oturum yazma kilidi alınır ve oturum transkript hedefi hazırlanır. Daha sonraki tüm transkript yeniden yazma, Compaction veya kesme yolları, SQLite transkript satırlarını değiştirmeden önce aynı kilidi almalıdır.

## İstem oluşturma

Sistem istemi; OpenClaw'ın temel isteminden, Skills isteminden, önyükleme bağlamından ve çalıştırma başına geçersiz kılmalardan oluşturulur. Modele özgü sınırlar ve Compaction için ayrılmış belirteçler uygulanır. Modelin gördükleri için bkz. [Sistem istemi](/tr/concepts/system-prompt).

## Kancalar

OpenClaw'ın iki kanca sistemi vardır:

- **Dahili kancalar** (Gateway kancaları): komutlar ve yaşam döngüsü olayları için olay güdümlü betikler.
- **Plugin kancaları**: aracı/araç yaşam döngüsü ve Gateway işlem hattı içindeki genişletme noktaları.

### Dahili kancalar (Gateway kancaları)

- **`agent:bootstrap`**: sistem istemi kesinleşmeden önce önyükleme dosyaları oluşturulurken çalışır. Önyükleme bağlam dosyaları eklemek veya kaldırmak için kullanılır.
- **Komut kancaları**: `/new`, `/reset`, `/stop` ve diğer komut olayları (Kancalar belgesine bakın).

Kurulum ve örnekler için bkz. [Kancalar](/tr/automation/hooks).

### Plugin kancaları

Bunlar aracı döngüsü veya Gateway işlem hattı içinde çalışır:

| Kanca                                                    | Çalışma zamanı                                                                                                                                                                                                                                                                                        |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `before_model_resolve`                                  | Sağlayıcıyı/modeli çözümlemeden önce belirlenimsel olarak geçersiz kılmak için oturum öncesinde (`messages` olmadan).                                                                                                                                                                                                |
| `before_prompt_build`                                   | Oturum yüklendikten sonra (`messages` ile), gönderimden önce `prependContext`, `systemPrompt`, `prependSystemContext` veya `appendSystemContext` eklemek için. Tur başına dinamik metin için `prependContext`, sistem istemi alanına ait kalıcı yönlendirme için sistem bağlamı alanlarını kullanın. |
| `before_agent_reply`                                    | Satır içi eylemlerden sonra, LLM çağrısından önce. Bir Plugin'in turu üstlenip yapay bir yanıt döndürmesine veya turu tamamen sessize almasına olanak tanır.                                                                                                                                                                |
| `agent_end`                                             | Tamamlandıktan sonra, nihai mesaj listesi ve çalıştırma meta verileriyle.                                                                                                                                                                                                                             |
| `before_compaction` / `after_compaction`                | Compaction döngülerini gözlemler veya bunlara açıklama ekler.                                                                                                                                                                                                                                                      |
| `before_tool_call` / `after_tool_call`                  | Araç parametrelerini/sonuçlarını keser.                                                                                                                                                                                                                                                              |
| `before_install`                                        | Operatör kurulum ilkesi çalıştıktan sonra, aşamalandırılmış Skills/Plugin kurulum materyali üzerinde, Plugin kancaları geçerli süreçte yüklüyken.                                                                                                                                                           |
| `tool_result_persist`                                   | Araç sonuçlarını OpenClaw'a ait bir oturum transkriptine yazılmadan önce eşzamanlı olarak dönüştürür.                                                                                                                                                                                      |
| `message_received` / `message_sending` / `message_sent` | Gelen ve giden mesaj kancaları.                                                                                                                                                                                                                                                         |
| `session_start` / `session_end`                         | Oturum yaşam döngüsü sınırları.                                                                                                                                                                                                                                                               |
| `gateway_start` / `gateway_stop`                        | Gateway yaşam döngüsü olayları.                                                                                                                                                                                                                                                                   |

Giden/araç korumaları için kanca karar kuralları:

- `before_tool_call`: `{ block: true }` terminaldir ve daha düşük öncelikli işleyicileri durdurur. `{ block: false }` işlem yapmaz ve önceki bir engeli temizlemez.
- `before_install`: yukarıdakiyle aynı terminal/işlemsiz semantiğine sahiptir. CLI kurulum ve güncelleme yollarını kapsaması gereken operatör sahipliğindeki kurulum izin verme/engelleme kararları için `before_install` değil, `security.installPolicy` kullanın.
- `message_sending`: `{ cancel: true }` terminaldir ve daha düşük öncelikli işleyicileri durdurur. `{ cancel: false }` işlem yapmaz ve önceki bir iptali temizlemez.

Kanca API'si ve kayıt ayrıntıları için bkz. [Plugin kancaları](/tr/plugins/hooks).

Donanımlar bu kancaları uyarlayabilir. Codex uygulama sunucusu donanımı, belgelenmiş yansıtılmış yüzeyler için uyumluluk sözleşmesi olarak OpenClaw Plugin kancalarını korur; Codex yerel kancaları ayrı, daha düşük seviyeli bir Codex mekanizmasıdır.

## Akış

- Yardımcı deltaları, aracı çalışma zamanından `assistant` olayları olarak akışla iletilir.
- Blok akışı, `text_end` veya `message_end` üzerinde kısmi yanıtlar yayınlayabilir.
- Akıl yürütme akışı ayrı bir akış olabilir veya yanıtları engelleyebilir.
- Parçalama ve blok yanıt davranışı için bkz. [Akış](/tr/concepts/streaming).

## Araç yürütme

- Araç başlatma/güncelleme/sonlandırma olayları `tool` akışında yayınlanır.
- Araç sonuçları, günlüğe kaydedilmeden/yayınlanmadan önce boyut ve görüntü yükleri bakımından temizlenir.
- Mesajlaşma aracı gönderimleri, yinelenen yardımcı onaylarını engellemek için izlenir.

## Yanıt şekillendirme

Nihai yükler; yardımcı metninden (isteğe bağlı akıl yürütmeyle birlikte), satır içi araç özetlerinden (ayrıntılı moddayken ve izin verildiğinde) ve model hata verdiğinde yardımcı hata metninden oluşturulur.

- Tam sessiz belirteç `NO_REPLY`, giden yüklerden filtrelenir.
- Mesajlaşma aracı yinelemeleri nihai yük listesinden kaldırılır.
- İşlenebilir yük kalmazsa ve bir araç hata verdiyse, bir mesajlaşma aracı kullanıcıya görünür bir yanıt göndermiş olmadığı sürece yedek bir araç hatası yanıtı yayınlanır.

## Compaction ve yeniden denemeler

Otomatik Compaction, `compaction` akış olayları yayınlar ve yeniden denemeyi tetikleyebilir. Yeniden denemede, yinelenen çıktıyı önlemek için bellek içi arabellekler ve araç özetleri sıfırlanır. Bkz. [Compaction](/tr/concepts/compaction).

## Olay akışları

- `lifecycle`: `subscribeEmbeddedAgentSession` tarafından (ve yedek olarak `agentCommand` tarafından) yayınlanır.
- `assistant`: aracı çalışma zamanından akışla iletilen deltalar.
- `tool`: aracı çalışma zamanından akışla iletilen araç olayları.

Gateway, yaşam döngüsü ve araç başlatma/terminal olaylarını sınırlı,
yalnızca meta veri içeren [denetim defterine](/tr/cli/audit) yansıtır. Bu yansıtma, istemleri, mesajları, araç bağımsız değişkenlerini, araç sonuçlarını
veya ham hataları transkript/çalışma zamanı yolundan dışarı kopyalamadan kaynak bilgisini ve
sonuç kodlarını kaydeder.

## Sohbet kanalı işleme

Yardımcı deltaları, sohbet `delta` mesajlarında arabelleğe alınır. **Yaşam döngüsü sonu/hatasında** bir sohbet `final` yayınlanır.

## Zaman aşımları

| Zaman aşımı                                      | Varsayılan                             | Notlar                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ------------------------------------------------ | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agent.wait`                                     | 30s                                    | Yalnızca bekleme içindir; `timeoutMs` parametresi bunu geçersiz kılar. Altta yatan çalışmayı durdurmaz.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Agent çalışma zamanı (`agents.defaults.timeoutSeconds`) | 172800s (48h)                          | `runEmbeddedAgent` öğesinin iptal zamanlayıcısı tarafından uygulanır. Sınırsız çalışma bütçesi için `0` ayarlayın; model akışı canlılık gözcüleri yine de geçerlidir.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| CLI arka ucunun çıktı yokluğu gözcüsü                   | her yeni/devam ettirilen CLI çalışması için hesaplanır     | Agent çalışma zamanından ayrıdır ve kayıtlı arka uç plugin'i tarafından yönetilir. CLI içindeki bir arka plan görevi üst alt süreçle ortaktır ve genel agent zaman aşımından daha uzun süre çalışmaz.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Cron yalıtılmış agent turu                         | cron tarafından yönetilir                          | Zamanlayıcı, yürütme başladığında kendi zamanlayıcısını başlatır, yapılandırılmış son tarihte çalışmayı iptal eder, ardından zaman aşımını kaydetmeden önce sınırlı temizlik gerçekleştirir; böylece eski bir alt oturum hattı takılı bırakamaz.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Model boşta kalma zaman aşımı                               | Bulut 120s; kendi kendine barındırılan 300s           | OpenClaw, boşta kalma aralığı dolmadan yanıt parçaları gelmezse model isteğini iptal eder. `models.providers.<id>.timeoutSeconds`, yavaş yerel/kendi kendine barındırılan sağlayıcılar için bu boşta kalma gözcüsünü uzatır; ancak bunlar tüm agent çalışmasını yönettiğinden daha düşük herhangi bir sonlu `agents.defaults.timeoutSeconds` veya çalışmaya özgü zaman aşımıyla sınırlı kalır. Sınırsız çalışma bütçelerinde bile sağlayıcı sınıfına ait boşta kalma gözcüsü korunur. Açık bir model/agent zaman aşımı olmadan Cron tarafından tetiklenen bulut modeli çalışmaları aynı varsayılanı kullanır; açık bir cron çalışma zaman aşımı varsa yapılandırılmış model geri dönüşlerinin dış cron son tarihinden önce çalışabilmesi için bulut modeli akışındaki duraklamalar 60s ile sınırlandırılır. Gerçek anlamda yerel uç noktalarda (geri döngü/özel baseUrl) Cron tarafından tetiklenen çalışmalar yerel boşta kalma devre dışı bırakma seçeneğini korur; ağ baseUrl'lerindeki kendi kendine barındırılan sağlayıcılara örtük 300s gözcü uygulanır. Açık bir cron çalışma zaman aşımı varsa yerel/kendi kendine barındırılan duraklamalar bu zaman aşımıyla sınırlanır. Yavaş yerel sağlayıcılar için `models.providers.<id>.timeoutSeconds` ayarlayın. |
| Sağlayıcı HTTP isteği zaman aşımı                    | `models.providers.<id>.timeoutSeconds` | Bağlantıyı, üst bilgileri, gövdeyi, SDK isteği zaman aşımını, korumalı fetch iptal işlemeyi ve söz konusu sağlayıcının model akışı boşta kalma gözcüsünü kapsar. Tüm agent çalışma zamanı aşımını artırmadan önce yavaş yerel/kendi kendine barındırılan sağlayıcılar (örneğin Ollama) için kullanın; model isteğinin daha uzun çalışması gerekiyorsa agent/çalışma zamanı aşımını en az onun kadar yüksek tutun.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |

### Takılı oturum tanılaması

Tanılama etkinleştirildiğinde, yerleşik iki dakikalık eşik; gözlemlenen yanıt, araç, durum, engel veya ACP ilerlemesi bulunmayan uzun `processing` oturumlarını sınıflandırır:

- Etkin gömülü çalışmalar, model çağrıları ve araç çağrıları `session.long_running` olarak bildirilir. Sahibi belirli sessiz model çağrıları, yavaş veya akışsız sağlayıcıların çok erken takılmış olarak işaretlenmemesi için iptal eşiğine kadar `session.long_running` olarak kalır.
- Yakın zamanda ilerleme göstermeyen etkin çalışma `session.stalled` olarak bildirilir. Sahibi belirli model çağrıları, iptal eşiğinde veya sonrasında `session.stalled` durumuna geçer; sahibi olmayan eski model/araç etkinliği uzun süreli olarak gizlenmez.
- `session.stuck`, sahibi olmayan eski model/araç etkinliğine sahip boşta bekleyen kuyruk oturumları dâhil olmak üzere kurtarılabilir eski oturum kayıtları için ayrılmıştır.

İptal eşiği en az 5 dakika ve uyarı eşiğinin 3 katıdır. Eski oturum kayıtları, kurtarma geçitleri geçildikten hemen sonra etkilenen oturum hattını serbest bırakır; takılmış gömülü çalışmalar yalnızca iptal eşiğinden sonra iptal edilip boşaltılır, böylece yalnızca yavaş çalışan işlemler kesilmeden kuyruktaki çalışma devam eder. Kurtarma, yapılandırılmış istenen/tamamlanan sonuçlar üretir; tanılama durumu yalnızca aynı işleme nesli hâlâ geçerliyse boşta olarak işaretlenir ve oturum değişmeden kaldığı sürece tekrarlanan `session.stuck` tanılamaları artan aralıklarla seyrekleştirilir.

## İşlemlerin erken sonlanabileceği durumlar

- Agent zaman aşımı (iptal)
- AbortSignal (iptal)
- Gateway bağlantısının kesilmesi veya RPC zaman aşımı
- `agent.wait` zaman aşımı (yalnızca bekleme içindir, agent'ı durdurmaz)

## İlgili

- [Araçlar](/tr/tools) - kullanılabilir agent araçları
- [Kancalar](/tr/automation/hooks) - agent yaşam döngüsü olayları tarafından tetiklenen olay odaklı betikler
- [Compaction](/tr/concepts/compaction) - uzun konuşmaların nasıl özetlendiği
- [Yürütme Onayları](/tr/tools/exec-approvals) - kabuk komutları için onay geçitleri
- [Düşünme](/tr/tools/thinking) - düşünme/akıl yürütme düzeyi yapılandırması
