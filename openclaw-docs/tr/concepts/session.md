---
read_when:
    - Oturum yönlendirmesini ve yalıtımını anlamak istiyorsunuz
    - Çok kullanıcılı kurulumlar için DM kapsamını yapılandırmak istiyorsunuz
    - Günlük veya boşta kalan oturum sıfırlamalarında hata ayıklıyorsunuz
summary: OpenClaw konuşma oturumlarını nasıl yönetir
title: Oturum yönetimi
x-i18n:
    generated_at: "2026-07-26T23:55:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de85fe5a623bdbc6d5564d822b39e9077a582b0816b62ab30d2f7245bd097000
    source_path: concepts/session.md
    workflow: 16
---

OpenClaw, gelen her mesajı geldiği yere göre bir **oturuma** yönlendirir:
DM'ler, grup sohbetleri, cron işleri vb. Tüm oturum durumunun sahibi
**gateway**'dir; kullanıcı arayüzü istemcileri oturum verilerini gateway'den sorgular.

Kişisel ajan varsayılanı — tüm DM kanallarınızın paylaştığı, grup etkinlikleri
ve arka plan çalışmalarının aktığı tek bir sürekli görüşme — hakkında bilgi için
[Ana oturum](/concepts/main-session) bölümüne bakın.

## Mesajlar nasıl yönlendirilir

| Kaynak          | Davranış                  |
| --------------- | ------------------------- |
| Doğrudan mesajlar | Varsayılan olarak paylaşılan oturum |
| Grup sohbetleri     | Grup başına yalıtılmış        |
| Odalar/kanallar  | Oda başına yalıtılmış         |
| Cron işleri       | Her çalıştırmada yeni oturum     |
| Webhook'lar        | Hook başına yalıtılmış         |

## DM yalıtımı

Varsayılan olarak tüm DM'ler süreklilik sağlamak amacıyla tek bir oturumu
paylaşır; bu, tek kullanıcılı kurulumlar için uygundur.

<Warning>
Ajanınıza birden fazla kişi mesaj gönderebiliyorsa DM yalıtımını etkinleştirin.
Aksi takdirde tüm kullanıcılar aynı görüşme bağlamını paylaşır; dolayısıyla
Alice'in özel mesajları Bob tarafından görülebilir.
</Warning>

```json5
{
  session: {
    dmScope: "per-channel-peer", // kanala + gönderene göre yalıt
  },
}
```

`session.dmScope` seçenekleri:

| Değer                      | Davranış                                                 |
| -------------------------- | -------------------------------------------------------- |
| `main` (varsayılan)           | Tüm DM'ler [ana oturumu](/concepts/main-session) paylaşır |
| `per-peer`                 | Kanallar genelinde gönderene göre yalıt                       |
| `per-channel-peer`         | Kanala + gönderene göre yalıt (önerilir)                |
| `per-account-channel-peer` | Hesaba + kanala + gönderene göre yalıt                    |

<Tip>
Aynı kişi sizinle birden fazla kanaldan iletişime geçiyorsa kimliklerini tek bir
standart eş kimliğine eşlemek için `session.identityLinks` kullanın; böylece
aynı oturumu paylaşırlar.
</Tip>

### Bağlı kanalları kenetleme

Kenetleme komutları, yeni bir oturum başlatmadan mevcut doğrudan sohbet
oturumunun yanıt rotasını başka bir bağlı kanala taşır. Örnekler, yapılandırma
ve sorun giderme için [Kanal kenetleme](/tr/concepts/channel-docking) bölümüne bakın.

Kurulumunuzu `openclaw security audit` ile doğrulayın.

## Gizli oturumlar

Gizli oturumlar yalnızca Control UI'nin **Yeni ileti dizisi** ekranından kullanılabilir. Oturum girdisini, transkripti ve Compaction durumunu disk yerine işlem belleğinde tutmak için ileti dizisini başlatmadan önce **Gizli** seçeneğini açın. İleti dizisi Gateway yeniden başlatıldığında kaybolur, OpenClaw'ın otomatik bellek boşaltmasını çalıştırmaz ve sıfırlandığında ya da silindiğinde transkript arşivi oluşturmaz. Codex destekli çalıştırmalar da çalıştırma düzeneği ileti dizisini geçici modda başlatır; böylece Codex hiçbir çalıştırma kaydı veya yerel oturum durumu dosyası yazmaz. Diğer model sağlayıcıları HTTP API'lerini kullanır ve OpenClaw'da yerel sağlayıcı transkripti tutmaz.

`incognito-` segmenti pano, alt ajan ve gizli dahili oturum anahtarları için ayrılmıştır; `openclaw doctor --fix`, çakışan eski kalıcı anahtarları yeniden adlandırır.

Gizli mod, ajanın normal araçlarını kısıtlamaz. Bilgilerin kaydedilmesine yönelik açık bir istek veya araç aracılığıyla gerçekleştirilen herhangi bir dosya yazma işlemi, verileri gizli oturum deposunun dışında kalıcı hâle getirebilir. Yapılandırılmış model sağlayıcınız gönderdiğiniz mesajları işlemeye devam eder, tanılama günlükleri değişmeden kalır ve OpenClaw, HMAC referansları gibi içerik içermeyen denetim meta verilerini kaydetmeye devam eder.

Çok kullanıcılı gateway'lerde gizli ileti dizileri yalnızca yönetici kapsamındaki bağlantılar tarafından görülebilir ve başka bir oturumun ajan oturumu araçlarında veya transkript aramasında hiçbir zaman görünmez. Bu, onları depolamadan ve gateway aracılığıyla erişen diğer kullanıcılardan korur; canlı oturumları her zaman gözlemleyebilen gateway sahibinden veya işlem operatöründen korumaz.

## Görüşmeler arasında hatırlama

Ayrı transkriptler, her görüşmenin yerel geçmişini denetler. Kişisel veya
tamamen güvenilen bir ajan için `memory.search.rememberAcrossConversations: true`,
bu ajanın diğer özel görüşmeleri genelinde isteğe bağlı bir erişim adımı
ekler; transkriptlerini birleştirmez.

Özel doğrudan görüşmeler ve açıkça kalıcı kullanıcı arayüzü görüşmeleri,
birbirlerine ilgili bağlamı sağlayabilir. Gruplar ve kanallar her iki yönde de
ayrı kalır: transkriptleri özel hatırlama kaynakları değildir ve bu
görüşmelerdeki yanıtlar özel transkript bağlamını almaz. Geçmişi zaten
yüklendiği için mevcut görüşme de kapsam dışındadır.

Bu ayar oturum anahtarlarını, DM kapsamını, yönlendirmeyi, teslimatı veya
`tools.sessions.visibility` davranışını değiştirmez. `MEMORY.md` ve
`memory/*.md` içindeki paylaşılan çalışma alanı belleği de mevcut
davranışını korur. Geçerli bellek sağlayıcısının korumalı özel transkript
hatırlamasını desteklemesi gerekir; Lossless Claw gibi bağlam motorları bağımsız
kalır ve bununla birlikte çalışabilir. Kurulum ve çalışma zamanı ayrıntıları
için [Active Memory](/tr/concepts/active-memory#remember-across-conversations)
bölümüne bakın.

## Oturum yaşam döngüsü

Oturumlar, elle sıfırlanana veya otomatik sıfırlama ilkesi etkinleştirilene kadar yeniden kullanılır:

- **Otomatik sıfırlama yok** (varsayılan `mode: "none"`) - oturumlar aynı
  `sessionId` değerini korur; görüşme büyüdükçe etkin bağlamı Compaction yönetir.
- **Günlük sıfırlama** (`mode: "daily"`) - gateway ana makinesinde yapılandırılmış yerel
  saatte (`session.reset.atHour`, varsayılan `4`, 0-23) yeni bir oturuma geçilmesini sağlar. Günlük
  güncellik, sonraki meta veri yazma işlemlerine değil, mevcut `sessionId` başlangıç zamanına dayanır.
- **Boşta kalma sıfırlaması** (`mode: "idle"`) - `session.reset.idleMinutes`
  işlem yapılmadıktan sonra yeni bir oturuma geçilmesini sağlar. Boşta kalma güncelliği, son gerçek kullanıcı/kanal
  etkileşimine dayanır; dolayısıyla Heartbeat, Cron ve exec sistem olayları
  oturumu canlı tutmaz.
- **Elle sıfırlama** - sohbette `/new` veya `/reset` yazın. `/new <model>`
  modeli de değiştirir.

Hem günlük hem de boşta kalma sıfırlamaları yapılandırıldığında, önce süresi
dolan uygulanır. Heartbeat, Cron, exec ve diğer sistem olayı turları oturum
meta verilerini yazabilir ancak bu yazma işlemleri günlük veya boşta kalma
sıfırlaması güncelliğini uzatmaz. Bir sıfırlama oturumu yenilediğinde eski
oturum için sıraya alınmış sistem olayı bildirimleri atılır; böylece eski arka
plan güncellemeleri yeni oturumdaki ilk istemin başına eklenmez.

Sağlayıcıya ait etkin bir CLI oturumu bulunan oturumlar da aynı otomatik
sıfırlama yok varsayılanını izler. Bu oturumların bir zamanlayıcıyla süresinin
dolması gerekiyorsa `/reset` kullanın veya `session.reset`
değerini açıkça yapılandırın.

Otomatik sıfırlamaları genel olarak etkinleştirip ardından sohbet türü veya kanal başına geçersiz kılın:

```json5
{
  session: {
    reset: { mode: "daily", atHour: 4 },
    resetByType: {
      group: { mode: "idle", idleMinutes: 120 },
      thread: { mode: "daily", atHour: 6 },
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 10080 },
    },
  },
}
```

`resetByType`; `direct`, `group` ve `thread` değerlerini destekler. Doctor, eski `dm` girdilerini `direct` biçimine ve `session.idleMinutes` değerini `session.reset.idleMinutes` biçimine taşır; şema, kullanımdan kaldırılmış iki biçimi de reddeder.

## Durumun bulunduğu yer

- **Çalışma zamanı oturum satırları:** `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- **Arşivlenmiş transkript dosyaları:** `~/.openclaw/agents/<agentId>/sessions/`
- **Eski satır taşıma kaynağı:** `~/.openclaw/agents/<agentId>/sessions/sessions.json`

Ajan başına SQLite veritabanındaki oturum satırları ayrı yaşam döngüsü
zaman damgalarını tutar:

- `sessionStartedAt`: mevcut `sessionId` başlangıç zamanı; günlük sıfırlama bunu kullanır.
- `lastInteractionAt`: boşta kalma ömrünü uzatan son kullanıcı/kanal etkileşimi.
- `updatedAt`: son depo satırı değişikliği; listeleme ve budama için kullanışlıdır ancak
  günlük/boşta kalma sıfırlaması güncelliği için belirleyici değildir.

Eski kurulumlardan geçiş sırasında gateway başlangıcı ve `openclaw doctor
--fix`,
eski `sessions.json` satırlarını ve etkin transkript JSONL geçmişini otomatik
olarak SQLite'a aktarır. `sessionStartedAt` içermeyen satırlar, mevcutsa eski
transkript JSONL oturum başlığından çözümlenir. Eski bir satırda
`lastInteractionAt` de yoksa boşta kalma güncelliği, sonraki kayıt tutma yazma
işlemlerine değil, o oturumun başlangıç zamanına geri döner. Açık inceleme veya
doğrulama kanıtı gerektiğinde `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` ve [Doctor taşıma
sırası](/tr/cli/doctor#session-sqlite-migration) bölümünü kullanın.

## Oturum bakımı

OpenClaw, aşağıda varsayılanları gösterilen `session.maintenance` aracılığıyla
oturum depolamasını zaman içinde sınırlar:

```json5
{
  session: {
    maintenance: {
      mode: "enforce", // "enforce" temizliği uygular; "warn" yalnızca bildirir
      pruneAfter: "30d",
      maxEntries: 500,
    },
  },
}
```

Üretim ölçeğindeki `maxEntries` sınırları için Gateway çalışma zamanı
yazma işlemleri küçük bir üst sınır tamponu kullanır ve toplu olarak
yapılandırılmış sınıra geri temizler. Oturum deposu okumaları Gateway
başlangıcında girdileri budamaz veya sınırlamaz; böylece başlangıç ve
yalıtılmış Cron oturumları tam depo temizliğinin maliyetini üstlenmez.
`openclaw sessions cleanup --enforce`, sınırı hemen uygular.

Gateway model çalıştırması yoklama oturumları varsayılan olarak kısa ömürlüdür.
`agent:*:explicit:model-run-<uuid>` ile eşleşen satırlar sabit `24h` saklama
süresini kullanır ancak temizlik baskıya bağlıdır: eski yoklama satırlarını
yalnızca oturum girdisi bakım/sınır baskısına ulaşıldığında kaldırır ve daha
genel eski girdi yaş sınırından ve girdi sınırından önce çalışır. Normal
doğrudan, grup, ileti dizisi, Cron, hook, Heartbeat, ACP ve alt ajan oturumları
bu 24h saklama süresini devralmaz.

Bakım, sentetik Cron, hook, Heartbeat, ACP ve alt ajan girdilerinin zamanla
süresinin dolmasına izin verirken grup oturumları ve ileti dizisi kapsamlı
sohbet oturumları dâhil kalıcı harici görüşme işaretçilerini korur.

Arşivlenmiş oturumlar kullanıcı tarafından rafa kaldırılmıştır ve yaşa göre
budama, girdi sınırları, model çalıştırması temizliği ve disk bütçesi tahliyesi
dâhil tüm otomatik bakım yollarından muaftır. Arşivden çıkarılana veya açıkça
silinene kadar arşivlenmiş olarak kalırlar.

Daha önce DM yalıtımını kullandıysanız ve sonradan `session.dmScope` değerini
`main` olarak geri ayarladıysanız eski eş anahtarlı DM satırlarını
`openclaw sessions cleanup --dry-run --fix-dm-scope` ile önizleyin. Aynı bayrağın uygulanması
bu eski doğrudan DM satırlarını kullanımdan kaldırır ve transkriptlerini
silinmiş arşivler olarak tutar.

Herhangi bir bakım çalıştırmasını `openclaw sessions cleanup --dry-run` ile önizleyin.

## Oturumları inceleme

| Komut                    | Gösterdikleri                                           |
| -------------------------- | ----------------------------------------------- |
| `openclaw status`          | Oturum deposu yolu ve son etkinlik          |
| `openclaw sessions --json` | Tüm oturumlar (`--active <minutes>` ile filtreleyin) |
| Sohbette `/status`          | Bağlam kullanımı, model ve açma/kapama seçenekleri               |
| `/context list`            | Sistem isteminde bulunanlar                    |

## Ek okumalar

- [Oturum araması](/tr/concepts/session-search) - geçmiş transkriptler genelinde tam metin hatırlama
- [Oturum Budama](/tr/concepts/session-pruning) - araç sonuçlarını kısaltma
- [Compaction](/tr/concepts/compaction) - uzun görüşmeleri özetleme
- [Oturum Araçları](/tr/concepts/session-tool) - oturumlar arası çalışma için ajan araçları
- [Oturum Yönetimine Derinlemesine Bakış](/tr/reference/session-management-compaction) -
  depo şeması, transkriptler, gönderme ilkesi, kaynak meta verileri ve gelişmiş yapılandırma
- [Çoklu Ajan](/tr/concepts/multi-agent) - ajanlar genelinde yönlendirme ve oturum yalıtımı
- [Arka Plan Görevleri](/tr/automation/tasks) - ayrılmış çalışmaların oturum referansları içeren görev kayıtlarını nasıl oluşturduğu
- [Kanal Yönlendirme](/tr/channels/channel-routing) - gelen mesajların oturumlara nasıl yönlendirildiği

## İlgili

- [Oturum budama](/tr/concepts/session-pruning)
- [Oturum araçları](/tr/concepts/session-tool)
- [Komut kuyruğu](/tr/concepts/queue)
