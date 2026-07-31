---
read_when:
    - Otomatik yanıt yürütmesini veya eşzamanlılığı değiştirme
    - /queue modlarını veya mesaj yönlendirme davranışını açıklama
summary: Otomatik yanıt kuyruğu modları, varsayılanlar ve oturum bazında geçersiz kılmalar
title: Komut kuyruğu
x-i18n:
    generated_at: "2026-07-26T23:57:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 69b40f67146226b0315492b27fc9d2218cace8bbd1eaff6514f7efb33b69d763
    source_path: concepts/queue.md
    workflow: 16
---

OpenClaw, oturumlar arasında güvenli paralelliğe izin vermeye devam ederken birden fazla ajan çalıştırmasının çakışmasını önlemek için gelen otomatik yanıt çalıştırmalarını (tüm kanallar) küçük bir işlem içi kuyruk üzerinden serileştirir.

## Neden

- Otomatik yanıt çalıştırmaları maliyetli olabilir (LLM çağrıları) ve birden fazla gelen ileti birbirine yakın zamanlarda ulaştığında çakışabilir.
- Serileştirme, paylaşılan kaynaklar (oturum dosyaları, günlükler, CLI standart girdisi) için rekabeti önler ve yukarı akış hız sınırlarına takılma olasılığını azaltır.

## Nasıl çalışır

- Hat duyarlı bir FIFO kuyruğu, her hattı yapılandırılabilir bir eşzamanlılık sınırıyla boşaltır (yapılandırılmamış hatlar için varsayılan 1; `main` varsayılan olarak 4, `subagent` ise 8).
- `runEmbeddedAgent`, oturum başına yalnızca bir etkin çalıştırmayı garanti etmek için **oturum anahtarına** göre (hat `session:<key>`) kuyruğa ekler.
- Ardından her oturum çalıştırması bir **genel hatta** (varsayılan olarak `main`) kuyruğa alınır; böylece genel paralellik `agents.defaults.maxConcurrent` ile sınırlandırılır.
- Ayrıntılı günlük kaydı etkinleştirildiğinde, kuyruğa alınan çalıştırmalar başlamadan önce ~2s'den uzun beklediyse kısa bir bildirim oluşturur.
- Yazıyor göstergeleri kuyruğa ekleme sırasında hemen tetiklenmeye devam eder (kanal destekliyorsa); böylece çalıştırma sırasını beklerken kullanıcı deneyimi değişmez.

## Varsayılanlar

Ayarlanmadığında tüm gelen kanal yüzeyleri şunları kullanır:

- `mode: "steer"`
- `debounceMs: 500`
- `cap: 20`
- `drop: "summarize"`

Aynı turda yönlendirme varsayılandır. Çalıştırma sürerken gelen bir istem, çalıştırma yönlendirmeyi kabul edebiliyorsa etkin çalışma zamanına enjekte edilir; böylece ikinci bir oturum çalıştırması başlatılmaz. Etkin çalıştırma yönlendirmeyi kabul edemiyorsa OpenClaw, istemi başlatmadan önce etkin çalıştırmanın tamamlanmasını bekler.

## Kuyruk modları

`/queue`, bir oturumda zaten etkin bir çalıştırma varken normal gelen iletilerin ne yapacağını denetler:

- `steer`: iletileri etkin çalışma zamanına enjekte eder. OpenClaw, bekleyen tüm yönlendirme iletilerini **mevcut asistan turu araç çağrılarını yürütmeyi tamamladıktan sonra**, bir sonraki LLM çağrısından önce teslim eder; Codex app-server, toplu tek bir `turn/steer` alır. Çalıştırma etkin biçimde akış yapmıyorsa veya yönlendirme kullanılamıyorsa OpenClaw, istemi başlatmadan önce etkin çalıştırmanın sona ermesini bekler.
- `followup`: yönlendirme yapmaz. Her iletiyi mevcut çalıştırma sona erdikten sonraki bir ajan turu için kuyruğa ekler.
- `collect`: yönlendirme yapmaz. Kuyruğa alınmış iletileri sessizlik penceresinden sonra **tek** bir takip turunda birleştirir. İletiler farklı kanalları/iş parçacıklarını hedefliyorsa yönlendirmeyi korumak için ayrı ayrı boşaltılır.
- `interrupt`: söz konusu oturumun etkin çalıştırmasını iptal eder, ardından en yeni iletiyi çalıştırır.

Çalışma zamanına özgü zamanlama ve bağımlılık davranışı için [Yönlendirme kuyruğu](/tr/concepts/queue-steering) bölümüne bakın. Açık `/steer <message>` komutu için [Yönlendir](/tr/tools/steer) bölümüne bakın.

`messages.queue` üzerinden genel olarak veya kanal başına yapılandırın:

```json5
{
  messages: {
    queue: {
      mode: "steer",
      debounceMs: 500,
      cap: 20,
      drop: "summarize",
      byChannel: { discord: "collect" },
    },
  },
}
```

## Kuyruk seçenekleri

Seçenekler kuyruğa alınmış teslimata uygulanır. `debounceMs`, `steer` modunda Codex yönlendirme sessizlik penceresini de ayarlar:

- `debounceMs`: kuyruğa alınmış takipleri veya toplama gruplarını boşaltmadan önceki sessizlik penceresi; Codex `steer` modunda, toplu `turn/steer` gönderilmeden önceki sessizlik penceresi. Birimsiz sayılar milisaniyedir; `/queue` seçenekleri `ms`, `s`, `m`, `h` ve `d` birimlerini kabul eder.
- `cap`: oturum başına kuyruğa alınabilecek azami ileti sayısı. `1` altındaki değerler yok sayılır.
- `drop: "summarize"` (varsayılan): gerektiğinde kuyruğa en önce eklenen girdileri kaldırır, kısa özetleri korur ve bunları yapay bir takip istemi olarak enjekte eder.
- `drop: "old"`: özetleri korumadan, gerektiğinde kuyruğa en önce eklenen girdileri kaldırır.
- `drop: "new"`: kuyruk zaten doluyken en yeni iletiyi reddeder.

Varsayılanlar: `debounceMs: 500`, `cap: 20`, `drop: summarize`.

## Yönlendirme ve akış

Kanal akışı `partial` veya `block` olduğunda, etkin çalıştırma çalışma zamanı sınırlarına ulaşırken yönlendirme kısa ve görünür birkaç yanıt gibi görünebilir:

- `partial`: önizleme erken sonlandırılabilir, ardından yönlendirme kabul edildikten sonra yeni bir önizleme başlar.
- `block`: taslak boyutundaki bloklar aynı sıralı görünümü oluşturabilir.
- Akış olmadan, çalışma zamanı aynı turda yönlendirmeyi kabul edemediğinde yönlendirme etkin çalıştırmadan sonraki bir takibe geri döner.

`steer`, devam eden araçları iptal etmez. En yeni iletinin mevcut çalıştırmayı iptal etmesi gerekiyorsa `/queue interrupt` kullanın.

## Öncelik

OpenClaw, mod seçimi için şu sırayı uygular:

1. Satır içi veya kayıtlı oturum başına `/queue` geçersiz kılması.
2. `messages.queue.byChannel.<channel>`.
3. `messages.queue.mode`.
4. Varsayılan `steer`.

Seçeneklerde, satır içi veya kayıtlı `/queue` seçenekleri yapılandırmaya göre önceliklidir. Ardından sırasıyla kanala özgü gecikme (`messages.queue.debounceMsByChannel`), Plugin gecikme varsayılanları, genel `messages.queue` seçenekleri ve yerleşik varsayılanlar uygulanır. `cap` ve `drop`, kanal başına yapılandırma anahtarları değil, genel/oturum seçenekleridir.

## Oturum başına geçersiz kılmalar

- Mevcut oturumun kuyruk modunu kaydetmek için `/queue <steer|followup|collect|interrupt>` komutunu bağımsız bir komut olarak gönderin.
- Seçenekler birleştirilebilir: `/queue collect debounce:0.5s cap:25 drop:summarize`
- `/queue default` veya `/queue reset`, oturum geçersiz kılmasını temizler.

## Kuyruğa alınmış turu iptal etme

Bir istem takip/toplama kuyruğunda beklerken (örneğin başka bir tur etkinken gelen bir TUI veya
web sohbeti `chat.send`), Gateway, kuyruğa alınmış
içerik çalıştırılana veya kaldırılana kadar söz konusu istemci `runId` için
**Gateway'e ait bir iptal kimliği** tutar. Kimlik, taşma
özetinde birleştirilen içeriği izler.

- Belirli bir `runId` içeren `chat.abort`, istekte bulunan yetkiliyse (etkin çalıştırmalarla aynı sahiplik kuralları) bu turu hâlâ
  kuyruktayken iptal eder.
- `runId` içermeyen bir oturum için `chat.abort`, önce **yetkilendirilmiş kuyruğa alınmış turları
  iptal eder**, ardından yetkilendirilmiş etkin çalıştırmaları iptal eder. Bu sıra, kuyruğun boşaltılmasıyla
  işlerin yarı durdurulmuş bir oturuma yükseltilmesini önler.
- İstek sahibi başına denetim yapmadan tüm oturum kuyruğunu temizlemek, çok sahipli oturumlar için
  durdurma yolu değildir.
- Kuyruktaki beklemeler `sessions.list` için etkin ajan çalıştırmaları olarak yansıtılmaz ve
  etkin çalıştırma zaman aşımı anlamlarına sahip değildir; bunlar yalnızca etkin aşamaya aittir.

Gateway destekli istemciler (`openclaw tui` dâhil), çalıştırma ortasında gelen istemleri iletir ve
kuyruk modunu Gateway'in uygulamasına izin verir. Esc/`/stop`, kaybolan yerel tanıtıcıların hâlâ kuyruğa alınmış bir istemi çalışır durumda bırakamaması için oturum kapsamlı bir iptal
kullanır.

`openclaw chat` ve `openclaw tui --local`, gömülü çalışma zamanında aynı dört modu
uygular. Yerel `steer`, çalışma zamanı yönlendirmeyi kabul ettiğinde etkin bir gömülü çalıştırmaya enjekte edilir,
aksi hâlde bir takibe dönüşür; `followup` ve
`collect` yerel bekleyen işler olarak kalır; `interrupt`, en yeni iletiyi
başlatmadan önce etkin yerel çalıştırmayı iptal eder. Açık `/steer <message>` komutu
yerel mod komutu değildir.

## Kapsam ve garantiler

- Gateway yanıt işlem hattını kullanan tüm gelen kanallardaki otomatik yanıt ajan çalıştırmalarına uygulanır (WhatsApp web, Telegram, Slack, Discord, Signal, iMessage, web sohbeti vb.).
- Varsayılan hat (`main`), gelen iletiler + ana Heartbeat'ler için işlem genelindedir; birden fazla oturumun paralel çalışmasına izin vermek için `agents.defaults.maxConcurrent` ayarlayın.
- Arka plan işlerinin gelen yanıtları engellemeden paralel çalışabilmesi için ek hatlar (ör. `cron`, `cron-nested`, `nested`, `subagent`) bulunabilir. Yalıtılmış Cron ajan turları, iç ajan yürütmeleri `cron-nested` kullanırken bir `cron` yuvasını tutar. Paylaşılan Cron dışı `nested` akışları kendi hat davranışlarını korur. Bu ayrılmış çalıştırmalar [arka plan görevleri](/tr/automation/tasks) olarak izlenir.
- Oturum başına hatlar, belirli bir oturuma aynı anda yalnızca bir ajan çalıştırmasının erişmesini garanti eder.
- Harici bağımlılık veya arka plan çalışan iş parçacığı yoktur; yalnızca TypeScript + promise'lar kullanılır.

## Sorun giderme

- Komutlar takılmış görünüyorsa ayrıntılı günlükleri etkinleştirin ve kuyruğun boşaltıldığını doğrulamak için "queued for ...ms" satırlarını arayın.
- Bir turu kabul edip ardından ilerleme bildirmeyi durduran Codex app-server çalıştırmaları, etkin oturum hattının dış çalıştırma zaman aşımını beklemek yerine serbest bırakılabilmesi için Codex bağdaştırıcısı tarafından kesintiye uğratılır.
- Tanılama etkinleştirildiğinde yerleşik uyarı eşiğini geçecek kadar `processing` durumunda kalan ve gözlemlenmiş yanıt, araç, durum, blok veya ACP ilerlemesi bulunmayan oturumlar mevcut etkinliğe göre sınıflandırılır:
  - Yakın zamanda ilerleme kaydedilmiş etkin çalışma `session.long_running` olarak günlüğe kaydedilir. Sahibi olan sessiz model çağrıları da yavaş veya akışsız sağlayıcıların gereğinden erken takılmış olarak bildirilmemesi için yerleşik iptal eşiğine kadar `session.long_running` durumunda kalır.
  - Yakın zamanda ilerleme kaydı bulunmayan etkin çalışma `session.stalled` olarak günlüğe kaydedilir; sahibi olan model çağrıları, engellenmiş araç çağrıları ve takılmış gömülü çalıştırmalar iptal eşiğinde veya sonrasında `session.stalled` durumuna geçer. Sahipsiz ve eskimiş model/araç etkinliği uzun süredir devam ediyormuş gibi gizlenmez.
  - `session.stuck`, sahipsiz ve eskimiş model/araç etkinliğine sahip boşta kuyruğa alınmış oturumlar dâhil olmak üzere kurtarılabilir eskimiş oturum kayıtları için ayrılmıştır.
  - `session.stuck`, etkilenen oturum hattını serbest bırakabilen kurtarmayı her zaman tetikler. İptal eşiğini geçen bir `session.stalled` sınıflandırması (engellenmiş araç çağrısı, takılmış model çağrısı veya takılmış gömülü çalıştırma) da etkin iptal kurtarmasını tetikleyebilir; dolayısıyla yalnızca `session.stuck` değil, her iki sınıflandırma da kuyruğun takılmasını giderebilir.
  - Yinelenen `session.stuck` ve `session.long_running` uyarı günlüğü satırları, oturum değişmeden kaldığı sürece üstel olarak seyrekleşir; kurtarma girişimleri bu seyrekleşmeden bağımsız olarak her Heartbeat döngüsünde çalışmaya devam eder.

## İlgili konular

- [Oturum yönetimi](/tr/concepts/session)
- [Yönlendirme kuyruğu](/tr/concepts/queue-steering)
- [Yönlendir](/tr/tools/steer)
- [Yeniden deneme ilkesi](/tr/concepts/retry)
