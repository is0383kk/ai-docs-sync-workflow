---
read_when:
    - Control UI'da Kanban tarzı bir çalışma panosu istiyorsunuz
    - Paketle birlikte gelen Workboard Plugin'ini etkinleştiriyor veya devre dışı bırakıyorsunuz
    - Planlanan ajan çalışmalarını harici bir proje yöneticisi olmadan takip etmek istiyorsunuz
summary: Ajanların sahip olduğu kartlar ve oturum devri için isteğe bağlı pano çalışma tahtası
title: Workboard Plugin'i
x-i18n:
    generated_at: "2026-07-27T00:09:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ec05c990c3559015780d9cb80f3ceedd7cc79db89ccf1afd65906c8c7630331
    source_path: plugins/workboard.md
    workflow: 16
---

Workboard Plugin'i,
[Control UI](/tr/web/control-ui)'ye isteğe bağlı Kanban tarzı bir pano ekler: ajan boyutunda iş kartları, ajanlara atama
ve kartın görevine, çalıştırmasına ve pano oturumuna geri bağlantı.

Workboard bilinçli olarak küçük tutulmuştur: tek bir OpenClaw Gateway için yerel işletim işlerini
izler. GitHub Issues, Linear, Jira veya diğer ekip proje yönetimi
sistemlerinin yerine geçmez.

## Etkinleştirme

Workboard paketle birlikte gelir ancak varsayılan olarak devre dışıdır:

1. Control UI'de **Plugins** bölümünü açın veya yapılandırılmış Control UI temel yoluna göre `/settings/plugins` kullanın.
   Örneğin, `/openclaw` temel yolu
   `/openclaw/settings/plugins` kullanır.
2. **Workboard** öğesini bulun ve **Enable** seçeneğini belirleyin. Workboard, OpenClaw ile birlikte
   geldiğinden **Install** işlemi gerektirmez.
3. UI yeniden başlatma gerektiğini bildirirse Gateway'i yeniden başlatın.

Plugin çalışma zamanı yüklendikten sonra Workboard sekmesi pano gezintisinde görünür.
Devre dışıyken sekme gezintide gizli kalır. Plugin devre dışıyken veya
`plugins.allow`/`plugins.deny` tarafından engellenmişken `/workboard`
rotasını doğrudan açmak, kart verileri yerine Plugin kullanılamıyor durumunu
gösterir.

Eşdeğer CLI iş akışı şöyledir:

```bash
openclaw plugins enable workboard
openclaw gateway restart
openclaw dashboard
```

## Yapılandırma

Workboard'a özgü bir yapılandırma yoktur. Standart Plugin girdisiyle
etkinleştirin/devre dışı bırakın:

```json5
{
  plugins: {
    entries: {
      workboard: {
        enabled: true,
        config: {},
      },
    },
  },
}
```

```bash
openclaw plugins disable workboard
openclaw gateway restart
```

## Kart alanları

| Alan        | Değerler                                                                                                      |
| ----------- | ------------------------------------------------------------------------------------------------------------- |
| `status`    | `triage`, `backlog`, `todo`, `scheduled`, `ready`, `running`, `review`, `blocked`, `done`                     |
| `priority`  | `low`, `normal`, `high`, `urgent`                                                                             |
| `labels`    | serbest biçimli dizeler                                                                                       |
| `agentId`   | isteğe bağlı atanmış ajan                                                                                     |
| bağlantılı referanslar | isteğe bağlı görev, çalıştırma, oturum veya kaynak URL'si                                                    |
| `execution` | karttan başlatılan bir Codex/Claude çalıştırması için isteğe bağlı meta veriler (motor, mod, model, oturum, çalıştırma kimliği, durum) |

Kartlar ayrıca denemeler, yorumlar, bağlantılar, kanıtlar,
yapıtlar, otomasyon ayarları, ekler, çalışan günlükleri, çalışan protokolü
durumu, talepler, tanılamalar, bildirimler, şablon kimliği, arşiv durumu ve
bayat oturum algılamasının yanı sıra yakın zamandaki olayların listesini (`created`, `edited`,
`moved`, `linked`, `specified`, `decomposed`, `claimed`, `heartbeat`,
`execution_updated`, `attempt_started`, `attempt_updated`, `comment_added`,
`link_added`, `proof_added`, `artifact_added`, `attachment_added`,
`diagnostic`, `notification`, `dispatch`, `orchestration`,
`protocol_violation`, `archived`, `unarchived`, `stale`) içeren kompakt meta veriler taşır. Bu meta veriler,
bir operatörün bağlantılı oturumu açmadan kartın panoda nasıl ilerlediğini
görmesini sağlar; oturum dökümlerinin veya GitHub sorun geçmişinin yerine
geçen bir şey değil, yerel işletim bağlamıdır.

Plugin ve Control UI tek bir Workboard kart sözleşmesi kullanır. Bu nedenle pano yenilemeleri,
kartın daha küçük ve yalnızca UI'ye özel bir kopyasını yansıtmak yerine çalışma alanı kökenini ve yetkisini,
talep durumunu, tanılama eylemlerini ve bildirim sıra numaralarını korur.
Bilinmeyen tanılama türleri, tanılama önem dereceleri ve bildirim türleri, her iki yüzey de
bunları destekleyene kadar yok sayılır; hiçbir zaman başka bir geçerli duruma
yeniden yazılmaz.

Açık pano, `plugin.workboard.changed` geçersiz kılmalarından güncellenir. Her
olay yalnızca bir depo dönemi ve revizyon içerir; ardından UI, normal
`operator.read` RPC aracılığıyla kanonik kartları yeniden okur. Birden fazla revizyon,
tek bir takip okumasında birleştirilir. Workboard, bir kart sürüklenirken,
düzenlenirken veya yazılırken bu okumayı erteler, ardından yerel etkileşim tamamlandıktan
sonra devam ettirir. Yeniden bağlantı her zaman kanonik yeniden yükleme gerçekleştirir. Rutin bir tam kart
yoklaması yoktur ve **Refresh** elle kurtarma seçeneği olarak kullanılabilir kalır.

Birden fazla pano bulunduğunda araç çubuğu, yalnızca o anda görünür kartlar yerine
kalıcı pano meta verileriyle desteklenen bir **Board** filtresi içerir. Bu nedenle boş
ve arşivlenmiş panolar seçilebilir kalır. Açıkça belirtilmiş bir
pano kimliği olmayan kartlar kanonik `default` panosuna aittir. Her panonun yer imlerine eklenebilen,
paylaşılabilen veya kenar çubuğuna sabitlenebilen kanonik bir
`/workboard/<boardId>` sayfası vardır. Daha önce yayımlanan `/workboard?board=<boardId>` biçimi,
diğer sorgu parametrelerini koruyarak bu sayfaya yönlendiren bir
uyumluluk diğer adı olarak kalır. **All boards** seçildiğinde `/workboard` konumuna dönülür.

Kartlar Plugin'in kendi Gateway durumunda depolanır ve söz konusu Gateway'in
OpenClaw durumunun geri kalanıyla birlikte taşınır (bkz. [Depolama](#storage)).

## Karttan iş başlatma

Bağlantısız kartlar doğrudan iş başlatabilir:

- **Run Codex** / **Run Claude**, açıkça belirtilmiş bir motorla görev tarafından izlenen bir ajan çalıştırması başlatır,
  kart istemini gönderir ve kartı `running` olarak işaretler. Codex
  çalıştırmaları `openai/gpt-5.6-sol`; Claude çalıştırmaları `anthropic/claude-sonnet-4-6` kullanır.
- **Open Codex** / **Open Claude**, kart istemini göndermeden veya
  kartı taşımadan bağlantılı bir pano oturumu oluşturur; böylece elle yapılan çalışma
  panoya bağlı kalır.

Otonom başlatmalar Gateway'in görev tarafından izlenen ajan çalıştırma yolunu kullanır
(Codex/Claude açıkça seçilmediği sürece varsayılan ajan ve model); ardından Workboard,
elde edilen görevi, çalıştırma kimliğini ve oturum anahtarını karta bağlar. Bağlantılı her
yürütme ayrıca bir deneme özeti (motor, mod, model, çalıştırma kimliği,
zaman damgaları, durum, değişken hata sayısı) kaydeder; böylece yinelenen hatalar görünür kalır.

Pano, görevleri görev kimliğine, çalıştırma kimliğine veya bağlantılı oturum anahtarına göre
kartlarla eşleştirerek Gateway görev defterinden görev durumunu yeniler. Kuyruğa alınmış/çalışan
bir görev kartın yaşam döngüsünü etkin tutar; tamamlanmış, başarısız olmuş, zaman aşımına uğramış veya
iptal edilmiş bir görev, bağlantılı oturumlarla aynı eşitleme kuralını kullanarak kartı
`review` veya `blocked` yönüne taşır (bkz. [Oturum yaşam döngüsü eşitlemesi](#session-lifecycle-sync)).

## Ajan araçları

| Araç                                                                                                                                             | Amaç                                                                                                                                                                                   |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `workboard_list`                                                                                                                                 | Talep/tanı durumuyla birlikte kompakt kartları listeler; isteğe bağlı pano filtresi.                                                                                                                    |
| `workboard_read`                                                                                                                                 | Tek bir kartı ve sınırlandırılmış çalışan bağlamını (notlar, denemeler, yorumlar, bağlantılar, kanıtlar, yapıtlar, üst kart sonuçları, atanan kişinin son çalışmaları, etkin tanılar) döndürür.                               |
| `workboard_create`                                                                                                                               | İsteğe bağlı üst kartlar, kiracı, Skills, pano, çalışma alanı meta verileri, eşgüçlülük anahtarı, çalışma süresi sınırı ve yeniden deneme bütçesiyle bir kart oluşturur.                                                             |
| `workboard_link`                                                                                                                                 | Bir üst kartı bir alt karta bağlar. Her üst kart `done` durumuna ulaşana kadar alt kartlar `todo` durumunda kalır; ardından gönderim yükseltmesi onları `ready` durumuna taşır.                                                     |
| `workboard_claim`                                                                                                                                | Çağrıyı yapan ajan adına bir kartı talep eder; `backlog`/`todo`/`ready` durumlarını `running` durumuna taşır.                                                                                                        |
| `workboard_heartbeat`                                                                                                                            | Daha uzun bir çalıştırma sırasında talep Heartbeat'ini yeniler.                                                                                                                                          |
| `workboard_release`                                                                                                                              | Tamamlama, duraklatma veya devretme sonrasında talebi serbest bırakır; kartı sonraki bir duruma taşıyabilir.                                                                                                |
| `workboard_complete` / `workboard_block`                                                                                                         | Son özetler, kanıtlar, yapıtlar ve oluşturulan kart bildirimleri (tamamlanan karta geri bağlanan kartlara başvurmalıdır) ya da engelleyici nedenleri için yapılandırılmış yaşam döngüsü araçları.                 |
| `workboard_attachment_add` / `workboard_attachment_read` / `workboard_attachment_delete`                                                         | Küçük kart eklerini Plugin SQLite durumunda depolar, kartta dizine ekler ve çalışan bağlamında sunar.                                                                                         |
| `workboard_worker_log` / `workboard_protocol_violation`                                                                                          | Çalışan günlük satırlarını kaydeder ve otomatik bir çalışan `workboard_complete`/`workboard_block` çağrısı yapmadan durduğunda kartı engeller.                                                           |
| `workboard_board_create` / `workboard_board_archive` / `workboard_board_delete`                                                                  | Kalıcı pano meta verilerini (görünen ad, açıklama, arşiv durumu, varsayılan çalışma alanı) yönetir.                                                                                            |
| `workboard_runs`                                                                                                                                 | Bir kartın kalıcı çalıştırma denemesi geçmişini döndürür.                                                                                                                                      |
| `workboard_specify`                                                                                                                              | Taslak bir triyaj/birikim kartını netleştirilmiş bir `todo` kartına dönüştürür; belirtim özetini karta kaydeder.                                                                                      |
| `workboard_decompose`                                                                                                                            | Bir üst düzenleme kartını, pano/kiracı meta verilerini devralan bağlantılı alt kartlara ayırır; üst kartı oluşturulan kart bildirimiyle tamamlayabilir.                                             |
| `workboard_notify_subscribe` / `workboard_notify_list` / `workboard_notify_events` / `workboard_notify_advance` / `workboard_notify_unsubscribe` | Bildirim aboneliklerini yönetir. Olay okumaları yeniden oynatmaya karşı güvenlidir; `advance`, kalıcı imleci taşıyarak çağıranların tamamlanmış/başarısız/bayat kart olaylarını kaybetmeden veya iki kez okumadan devam etmesini sağlar. |
| `workboard_boards` / `workboard_stats`                                                                                                           | Pano ad alanlarını ve kuyruk istatistiklerini inceler.                                                                                                                                                 |
| `workboard_promote` / `workboard_reassign` / `workboard_reclaim`                                                                                 | Takılı kalan işi kurtarır veya devreder.                                                                                                                                                           |
| `workboard_comment` / `workboard_proof`                                                                                                          | Devretme notları ekler veya kanıt/yapıt başvuruları iliştirir.                                                                                                                                    |
| `workboard_unblock`                                                                                                                              | Engellenmiş işi yeniden `todo` durumuna taşır.                                                                                                                                                         |
| `workboard_move`                                                                                                                                 | Bir kartı başka bir duruma taşır; talep edilmiş kartlar, çağıranın ajan talebi kapsamını gerektirir.                                                                                                      |
| `workboard_dispatch`                                                                                                                             | Çalışanları başlatmadan bağımlılık yükseltmesini veya bayat talep temizliğini tetikler; çalışan başlatma, Gateway veya eğik çizgi komutu gönderimini kullanır.                                                        |

Kanıt durumları, bağımsız doğrulama değil, çalışanlar tarafından bildirilen sonuçlardır. Bir `passed`
girdisi, çalışanın komutunun veya denetiminin başarılı olduğunu bildirdiği anlamına gelir; bağımsız
bir kalite kapısına ihtiyaç duyan tüketiciler, ekli komutu, URL'yi veya yapıtı incelemeli ve
kendi doğrulayıcılarını çalıştırmalıdır. `workboard_proof`, yeni kaydın `proofId` değerini döndürür.
`workboard_complete` aynı kanıtın son durumunu bildirdiğinde, bekleyen kaydın kimliğini veya zaman damgasını
kaybetmeden yerinde çözümlenmesi için `proofId` değerini iletin. Zaten aynı son duruma
sahip bir kanıt değiştirilmeden yeniden kullanılır. `proofId` olmadan tamamlama kanıtı
yalnızca eklemeli kalır; böylece sonraki bir yeniden deneme, yalnızca komutu veya notu aynı olduğu
için eski geçmişi yeniden yazamaz.

Talep edilmiş kartlar, çağıran `workboard_claim` tarafından döndürülen talep belirtecine
sahip olmadığı sürece diğer ajanların ajan aracı değişikliklerini reddeder. Bir ajan aracı
veya Gateway RPC çağrısı tarafından döndürülen her kart, `metadata.claim.token` değerini
`[redacted]` olarak gizler (belirtecin kendisi yalnızca `workboard_claim` tarafından
bir kez ve en üst düzeyde döndürülür); böylece gösterge paneli operatörleri ve diğer ajanlar,
kullanılabilir bir belirteci hiç görmeden talep durumunu inceleyebilir. Kurtarma,
belirteç gerektirmeyen `workboard_promote`/`workboard_reassign`/`workboard_reclaim`
üzerinden gerçekleştirilir.

## Gönderim

Gönderim Gateway'e özeldir: rastgele işletim sistemi süreçleri oluşturmaz. Normal
OpenClaw alt ajan oturumları yürütmenin sahibi olmaya devam eder. Bir gönderim geçişi:

1. Bağımlılıkları hazır kartları yükseltir.
2. Hazır kartlara gönderim meta verilerini kaydeder.
3. Süresi dolmuş talepleri veya zaman aşımına uğramış çalıştırmaları engeller.
4. Pano tarafından yapılandırılmış triyaj kartlarını düzenleme adayı olarak işaretler.
5. Hazır kartlardan küçük bir grubu talep eder ve Gateway alt ajan çalışma zamanı
   üzerinden çalışan çalıştırmalarını başlatır.

Çalışanlar, sınırlandırılmış kart bağlamının yanı sıra Workboard araçları üzerinden
Heartbeat göndermek, kartı tamamlamak veya engellemek için gereken talep belirtecini alır.

Çalışma alanı yolları, çağıranın mevcut dosya sistemi yetkisine tabidir. `operator.write`
özelliğine sahip Gateway istemcileri yapılandırılmış ajan çalışma alanlarını;
`operator.admin` istemcileri ise diğer ana makine teslim alma kopyalarını kullanabilir.
Korumalı alandaki ajan araçları kendi korumalı alan çalışma alanı erişimlerini kullanırken,
korumalı alan dışında yalnızca çalışma alanına erişen araçlar yapılandırılmış çalışma alanı
kökünü kullanır. Workboard, bir çalışma alanı atandığında bu yetkiyi kaydeder ve gönderim
sırasında yeniden mevcut çağıranın yetkisiyle kesiştirir; böylece kalıcı bir kart daha sonraki
bir çağıranın erişimini genişletemez. Açık bir ana makine çalışma alanına sahip ancak kayıtlı
yetkisi olmayan eski kartların, tam ana makine gönderiminden önce çalışma alanı yeniden
kaydedilmelidir; ana makine yolu olmayan kartlar ilk kez gönderildiğinde mevcut çağıranın
yetkisini benimser.

Çalışma alanına bağlı gönderim, yalnızca depo kökü hedef ajan çalışma alanıyla tam olarak
eşleştiğinde bir dizini veya Git teslim alma kopyasını kabul eder. Bir worktree isteği bu
dizinle sınırlandırılır ve dizin çalışma alanı olarak kalıcılaştırılır; böylece ana makine
teslim alma kopyasını oluşturmaz veya depo kurulum kodunu çalıştırmaz. Hedef çalışan, tam
olarak bu çalışma alanı için yazılabilir ve paylaşılmayan bir Docker korumalı alanını;
yükseltilmiş yürütme, kalıcı ana makine/Node exec geçersiz kılmaları ya da sınıflandırılmamış
Plugin ve MCP araçları olmadan kullanmalıdır. Workboard, bir `workboard_*` önekine
güvenmek yerine kayıtlı araçlarını sıralar ve gönderim, canlı bağlama/yapılandırma karması
bayat olan etkin bir Docker kapsayıcısını reddeder. Gönderim, daha az kısıtlanmış bir çalışan
başlatmak yerine uyumsuz hedef politikasını bildirir. Tam ana makine gönderimi diğer yerel
teslim alma kopyalarını hedefleyebilir ve normal yönetilen worktree kurulumunu korur.

Çalışma alanı yetkisi ikinci bir kart yaşam döngüsü izin modeli oluşturmaz.
Workboard kartlarını değiştirebilen çağıranlar, bunları her yüzeyde aynı durumlar
arasında elle taşıyabilir; salt okunur çalışma alanı erişimi yalnızca yazma gerektiren
çalışan gönderimini engeller.

### Çalışan seçimi

Her geçiş varsayılan olarak **en fazla 3 worker başlatır**. Hazır kartlar
önceliğe, ardından konuma, sonra da oluşturulma zamanına göre sıralanır. Bir geçiş,
owner/agent başına yalnızca bir kart başlatır ve panoda hâlihazırda çalışan veya
inceleme aşamasında işi bulunan owner'ları atlar. Arşivlenmiş kartlar, etkin claim'i
bulunan kartlar ve `ready` durumunda olmayan kartlar worker başlatmak için
asla seçilmez (dispatch'in veri tarafı bunları yine de etkileyebilir: geçerliliğini
yitirmiş claim'leri temizleme, bağımlılıkları ilerletme, zaman aşımı temizliği).

Oturum anahtarları pano/kart başına deterministiktir; bu nedenle tekrarlanan
dispatch'ler ilgisiz oturumlar oluşturmak yerine aynı worker hattına geri yönlendirilir:

- Atanmış kartlar: `agent:<agentId>:subagent:workboard-<boardId>-<cardId>`
- Atanmamış kartlar: `subagent:workboard-<boardId>-<cardId>` (Gateway, yapılandırılmış
  varsayılan agent'ı çözümler)

Bir kart claim edildikten sonra worker başlatılamazsa Workboard kartı bloklar,
claim'i temizler, çalıştırma başlatma hatasını kaydeder ve dashboard, CLI JSON,
agent araçları ve kart tanılamalarında görülebilen bir worker günlük satırı
ekler.

### Giriş noktaları

- Dashboard dispatch eylemi
- `openclaw workboard dispatch`
- Komut destekleyen bir kanalda `/workboard dispatch`

Gateway kullanılabilir olduğunda üçünün tümü Gateway alt agent çalışma zamanını
kullanır. CLI'da tek bir operatör geri dönüşü vardır: Gateway çağrısı bir
bağlantı/kullanılamıyor hatasıyla (veya eski Gateway'lerde bir
`unknown method` hatasıyla) başarısız olursa, açık bir
`--url`/`--token` hedefi verilmemişse ve yapılandırılmış bir uzak
Gateway (`OPENCLAW_GATEWAY_URL` veya `gateway.mode: remote`) geçerli değilse CLI, yerel SQLite
durumunda yalnızca veri dispatch'i çalıştırır; bağımlılıkları ilerletebilir,
geçerliliğini yitirmiş claim'leri temizleyebilir ve zaman aşımına uğramış
çalıştırmaları bloklayabilir, ancak worker başlatamaz. Erişilebilir bir Gateway'den
gelen kimlik doğrulama, izin ve doğrulama hataları kullanılamıyor olarak
değerlendirilmez; bunlar komut hatası olarak gösterilir. Açık bir
`--url`/`--token` hedefi verildiğinde oluşan tüm Gateway hataları da
aynı şekilde gösterilir.

Pano meta verileri `autoDecompose`, `autoDecomposePerDispatch`,
`defaultAssignee` ve `orchestratorProfile` değerlerini ayarlayabilir. OpenClaw bu amacı
kaydeder ve worker bağlamında kullanıma sunar; asıl belirtim/ayrıştırma işlemi
yine normal Workboard araçları üzerinden yürütülür.

## CLI ve eğik çizgi komutu

```bash
openclaw workboard list [--board <id>] [--status <status>] [--include-archived] [--json]
openclaw workboard create "Geçerliliğini yitirmiş kart yaşam döngüsünü düzelt" --priority high --labels bug,workboard
openclaw workboard show <card-id> [--json]
openclaw workboard move <card-id> --status <status> [--json]
openclaw workboard dispatch [--board <id>] [--json]
```

`list` metin çıktısı varsayılan olarak arşivlenmiş kartları gizler
(`--include-archived` bunu geçersiz kılar); `--json`, mevcut betiklerin kullandığı
tam kart sözleşmesiyle uyumlu olarak arşivlenmiş kartları her zaman içerir.
`show` ve `move`, belirsizlik taşımayan bir kimlik önekini kabul
eder. `list`, `create`, `show` ve `move`
yerel Plugin durumunu her zaman doğrudan okur/yazar. Yalnızca
`dispatch`, yukarıda açıklanan geri dönüşle birlikte çalışan Gateway'i çağırır.

Tüm bayraklar, JSON çıktısı, Gateway geri dönüş davranışı, kimlik öneki işleme,
dispatch seçim kuralları ve sorun giderme için [Workboard CLI](/tr/cli/workboard)
sayfasına bakın.

`/workboard list`, `/workboard show <card-id>`, `/workboard create <title>`,
`/workboard move <card-id> --status <status>` ve `/workboard dispatch`
CLI'yı yansıtır. Listeleme ve gösterme, yetkili tüm komut göndericileri için
okuma işlemleridir. Oluşturma, taşıma ve dispatch işlemleri sohbet yüzeylerinde
owner durumu veya `operator.write`/`operator.admin` kapsamına sahip bir Gateway
istemcisi gerektirir. Manuel operatör taşımaları, dashboard'daki sürükle ve
bırak işlemiyle aynı claim geçersiz kılma davranışını kullanır. Bunların worktree
erişimi de yukarıda açıklanan aynı çalışma alanı sınırını izler.

## Oturum yaşam döngüsü eşitlemesi

Kartlar mevcut bir dashboard oturumuna veya karttan çalışmayı başlattığınızda
oluşturulan bir oturuma bağlanabilir. Bağlı kartlar oturum yaşam döngüsünü satır
içinde gösterir: çalışıyor, geçerliliğini yitirmiş, bağlı ve boşta, tamamlandı,
başarısız veya eksik. Ayrıca Sessions sekmesinden mevcut bir oturumu
**Add to Workboard** ile alabilirsiniz; kart bu oturuma bağlanır, başlık olarak
oturum etiketini veya son kullanıcı istemini kullanır ve kullanılabildiğinde
notları son kullanıcı istemiyle en son asistan yanıtından oluşturur.

Bağlı oturum kaybolursa kart bağlam amacıyla bağlı kalır ve yeni bir oturumda
yeniden başlatmaya yönelik başlatma denetimlerini sunmaya devam eder. Etkin bir
bağlı oturum yakın zamandaki etkinliği bildirmeyi bırakırsa Workboard kartı
`stale` olarak işaretler ve yaşam döngüsü temizleyene kadar bunu meta
veri olarak saklar.

Kart etkin bir çalışma durumundayken Workboard bağlı oturumu izler:

| Bağlı oturum durumu                    | Kart durumu |
| -------------------------------------- | ----------- |
| etkin                                  | `running`   |
| tamamlandı                             | `review`    |
| başarısız, sonlandırıldı, zaman aşımına uğradı veya iptal edildi | `blocked`   |

**Manuel inceleme durumları önceliklidir.** Bir kartı `review`,
`blocked` veya `done` durumuna taşımak, kartı yeniden
`todo` veya `running` durumuna taşıyana kadar o kartın otomatik
eşitlemesini durdurur.

Bir kartı başlatmak normal Gateway oturumlarını kullanır; Workboard yalnızca
kart meta verilerini ve bağlantıları saklar. Konuşma dökümü, model seçimi ve
çalıştırma yaşam döngüsünün sahipliği normal oturum sisteminde kalır. Etkin
bağlı çalıştırmayı iptal etmek için canlı bir bağlı kartta **Stop** seçeneğini
kullanın; Workboard bu kartı `blocked` olarak işaretleyerek takip için
görünür kalmasını sağlar.

Yeni kartlar Workboard şablonlarından (`bugfix`, `docs`,
`release`, `pr_review`, `plugin`) başlatılabilir. Şablonlar
başlık, notlar, etiketler ve önceliği önceden doldurur; şablon kimliği kart meta
verisi olarak saklanır.

## Dashboard iş akışı

1. Control UI'da Workboard sekmesini açın.
2. Başlık, notlar, öncelik, etiketler, isteğe bağlı agent ve isteğe bağlı
   bağlı oturum içeren bir kart oluşturun veya Sessions'ı açıp mevcut bir oturum
   için **Add to Workboard** seçeneğini belirleyin.
3. Kartı sütunlar arasında sürükleyin veya kompakt durum denetimine odaklanıp
   menüyü ya da ArrowLeft/ArrowRight tuşlarını kullanın. Sürükleme sırasında
   kaynak kart soluklaşır ve kullanılabilir bırakma sütunları çerçevelenir.
4. Bir dashboard oturumu oluşturmak veya yeniden kullanmak için karttan çalışmayı başlatın.
5. Agent çalışırken bağlı oturumu karttan açın.
6. Yaşam döngüsü eşitlemesinin çalışan işi `review`/`blocked`
   durumuna taşımasına izin verin, ardından kabul edildiğinde kartı manuel olarak
   `done` durumuna taşıyın.

### Oturum panosu widget'ları

Workboard, oturum dashboard'ları için iki yerel widget sunar (bkz.
[Dashboard'lar](/web/dashboards)). Agent bunları `dashboard` aracıyla
`content: { kind: "plugin", pluginKind, props }` kullanarak sabitler ve bunlar canlı verilerle
birinci taraf UI olarak işlenir; sandbox çerçevesi veya yetenek izni gerekmez:

- `props: { cardId }` ile `workboard:card`, durum denetimi, önceliği
  ve atanmış agent'ıyla tek bir kart gösterir.
- İsteğe bağlı `props: { boardId, limit }` ile `workboard:mini`, duruma göre
  sayıları ve en üstteki hazır/çalışan kartları gösterir ve tam pano sayfasına
  bağlantı verir. `boardId` olmadan tüm panoları birleştirir;
  `boardId` ile kapsamı bu panoyla sınırlar (açık bir pano kimliği olmadan
  oluşturulan kartlar `default` üzerinde bulunur).

## Tanılama

Tanılamalar yerel kart meta verilerinden hesaplanır. Yerleşik denetimler
şunları işaretler:

| Tür                         | Koşul                                                                          |
| --------------------------- | ------------------------------------------------------------------------------ |
| `stranded_ready`            | Atanmış `todo`/`backlog`/`ready` kartı 1 saatten uzun süredir güncellenmemiş.             |
| `running_without_heartbeat` | `running` kartında 20 dakikadan uzun süredir claim Heartbeat'i veya yürütme güncellemesi yok. |
| `blocked_too_long`          | `blocked` kartı 24 saatten uzun süredir güncellenmemiş.                                   |
| `repeated_failures`         | Kartın izlenen hata sayısı 2 veya daha fazlasına ulaşıyor.                                |
| `missing_proof`             | `done` kartında kanıt, yapıt veya ek yok.                          |
| `orphaned_session`          | `running` kartında `sessionKey` var ancak `execution` meta verisi yok.                |

## İzinler

Gateway RPC yöntemleri `workboard.*` altında bulunur:

| Kapsam           | Yöntemler                                                                                                                                                                                                                                                                                                                                                                           |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operator.read`  | `cards.list`, `cards.export`, `cards.diagnostics`, ek listeleme/alma, bildirim olayı okumaları, `boards.list`, `cards.stats`, `cards.runs`                                                                                                                                                                                                                                       |
| `operator.write` | `cards.diagnostics.refresh`, oluşturma/güncelleme/taşıma/silme/yorumlama/bağlama/bağımlılık bağlama/kanıt/yapıt, ek ekleme/silme, worker günlüğü, protokol ihlali, claim/Heartbeat/serbest bırakma/ilerletme/yeniden atama/yeniden claim etme/tamamlama/bloklama/bloku kaldırma, `cards.dispatch`, `cards.bulk`, arşivleme, `boards.upsert`/`archive`/`delete`, `cards.specify`/`decompose`, bildirime abone olma/silme/ilerletme |

Hiçbir RPC yöntemi `operator.admin` gerektirmez. Salt okunur operatör
erişimiyle bağlanan tarayıcılar panoyu inceleyebilir ancak kartları değiştiremez.
Yönetici kapsamı kabul edilen Workboard ana makine yollarını genişletir;
kullanılabilir yöntemleri değiştirmez.

## Depolama

Workboard kalıcı verileri OpenClaw durum dizini altında Plugin'e ait ilişkisel
bir SQLite veritabanında saklar: panolar, kartlar, etiketler, yaşam döngüsü
olayları, çalıştırma denemeleri, yorumlar, bağımlılık bağlantıları, kanıt,
yapıt referansları, ek meta verileri ve blob'ları, tanılamalar, bildirimler,
worker günlükleri, protokol durumu ve aboneliklerin tümü Workboard tablolarında
bulunur (Plugin anahtar-değer girdilerinde değil). Kart dışa aktarımı, ek blob
içeriklerini satır içine almadan pano anlatısını korur.

Workboard'u `.28` sürümünde kullanan kurulumlar, yayımlanmış eski
Plugin durumu ad alanlarını (`workboard.cards`, `workboard.boards`,
`workboard.notify` ve varsa `workboard.attachments`) ilişkisel veritabanına taşımak için
`openclaw doctor --fix` komutunu çalıştırabilir.

## Sorun giderme

**Sekmede Workboard'un kullanılamadığı belirtiliyor**

```bash
openclaw plugins inspect workboard --runtime --json
```

`plugins.allow` yapılandırılmışsa buna `workboard` ekleyin.
`plugins.deny`, `workboard` içeriyorsa Plugin'i etkinleştirmeden önce bunu
kaldırın.

**Kartlar kaydedilmiyor**

Tarayıcı bağlantısının `operator.write` erişimine sahip olduğunu doğrulayın.
Salt okunur operatör oturumları kartları listeleyebilir ancak oluşturamaz,
düzenleyemez, taşıyamaz veya silemez.

**Bir kartı başlatmak beklenen oturumu açmıyor**

Kartın agent kimliğini ve bağlı oturumunu kontrol edin, ardından gerçek
çalıştırma durumunu incelemek için Sessions veya Chat'i açın.

**Dispatch bir worker başlatmıyor**

Etkin claim'i olmayan en az bir `ready` kartı bulunduğunu doğrulayın:

```bash
openclaw workboard list --status ready
```

CLI yalnızca veri gönderimi bildirdiyse Gateway'i başlatın veya yeniden başlatın ve
tekrar deneyin; yalnızca veri gönderimi yerel pano durumunu günceller ancak
alt ajan çalışanı yürütmelerini başlatamaz. Aynı sahip veya ajan için başka bir kart
zaten çalışıyorsa ya da inceleme bekliyorsa kartlar da atlanabilir; aynı
sahip için daha fazlasını göndermeden önce bu etkin çalışmayı tamamlayın,
engelleyin veya serbest bırakın.

## İlgili

- [Kontrol Arayüzü](/tr/web/control-ui)
- [Çalışma Panosu CLI'si](/tr/cli/workboard)
- [Pluginler](/tr/tools/plugin)
- [Pluginleri yönetin](/tr/plugins/manage-plugins)
- [Oturumlar](/tr/concepts/session)
