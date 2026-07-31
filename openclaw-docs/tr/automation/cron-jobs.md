---
read_when:
    - Arka plan işlerini veya uyandırmaları zamanlama
    - Harici tetikleyicileri (webhook'lar, Gmail) OpenClaw'a bağlama
    - Zamanlanmış görevler için Heartbeat ile Cron arasında karar verme
sidebarTitle: Scheduled tasks
summary: Gateway zamanlayıcısı için zamanlanmış işler, webhook'lar ve Gmail PubSub tetikleyicileri
title: Zamanlanmış görevler
x-i18n:
    generated_at: "2026-07-26T23:49:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dd889cf8e45196eda3ec7c2af930abcb2cc2bae8bad2dbdcaf3cd521a9e884b2
    source_path: automation/cron-jobs.md
    workflow: 16
---

Cron, Gateway'in yerleşik zamanlayıcısıdır. İşleri kalıcı olarak saklar, aracıyı doğru zamanda uyandırır ve çıktıyı bir sohbet kanalına, bir webhook'a teslim edebilir veya hiçbir yere teslim etmeyebilir.

## Hızlı başlangıç

<Steps>
  <Step title="Tek seferlik bir hatırlatıcı ekleyin">
    ```bash
    openclaw cron create "2027-02-01T16:00:00Z" \
      --name "Reminder" \
      --session main \
      --system-event "Reminder: cron dokümanları taslağını kontrol edin" \
      --wake now \
      --delete-after-run
    ```
  </Step>
  <Step title="İşlerinizi kontrol edin">
    ```bash
    openclaw cron list
    openclaw cron get <job-id>
    openclaw cron show <job-id>
    ```
  </Step>
  <Step title="Çalıştırma geçmişini görüntüleyin">
    ```bash
    openclaw cron runs --id <job-id>
    ```
  </Step>
</Steps>

## Cron nasıl çalışır?

- Cron, modelin içinde değil, **Gateway işleminin içinde** çalışır. Zamanlamaların tetiklenmesi için Gateway çalışıyor olmalıdır.
- İş tanımları, çalışma zamanı durumu ve çalıştırma geçmişi OpenClaw'ın paylaşılan SQLite durum veritabanında kalıcı olarak saklanır; bu nedenle yeniden başlatmalar zamanlamaların kaybolmasına yol açmaz.
- Her cron yürütmesi bir [arka plan görevi](/tr/automation/tasks) kaydı oluşturur.
- Tek seferlik işler (`--at`) varsayılan olarak başarıdan sonra otomatik silinir; bunları tutmak için `--keep-after-run` iletin.
- Çalıştırma başına duvar saati bütçesi: ayarlandığında `--timeout-seconds`. Aksi takdirde, yalıtılmış/ayrılmış aracı dönüşü işleri, temel aracı dönüşü zaman aşımı (`agents.defaults.timeoutSeconds`, varsayılan 48 saat) uygulanmadan önce cron'un kendi 60 dakikalık gözetmeniyle sınırlandırılır; komut işleri varsayılan olarak 10 dakika, betik yükleri ise varsayılan olarak 5 dakika ile sınırlandırılır.
- Gateway başlatılırken, süresi geçmiş yalıtılmış aracı dönüşü işleri hemen yeniden oynatılmak yerine yeniden zamanlanır; böylece model/araç önyükleme çalışması kanal bağlantısı penceresinin dışında tutulur.
- `openclaw agent` öğesini sistem cron'undan veya başka bir harici zamanlayıcıdan çalıştırıyorsanız, CLI zaten `SIGTERM`/`SIGINT` sinyallerini işlese de bunu zorla sonlandırma yükseltmesiyle sarmalayın. Gateway destekli çalıştırmalar, Gateway'den kabul edilen çalıştırmaları iptal etmesini ister; `--local` çalıştırmaları da aynı iptal sinyalini alır. GNU `timeout` için düz `timeout 600 ...` yerine `timeout -k 60 600 openclaw agent ...` tercih edin — işlem zamanında boşaltılamazsa `-k` değeri son güvenceyi sağlar. systemd birimleri için, nihai sonlandırmadan önce ek süre penceresine (`TimeoutStopSec`) sahip bir `SIGTERM` durdurma sinyali kullanın. Özgün Gateway çalıştırması hâlâ etkinken bir `--run-id` yeniden kullanılırsa, ikinci bir çalıştırma başlatmak yerine yinelenen çalıştırmanın devam ettiği bildirilir.

<AccordionGroup>
  <Accordion title="Yalıtılmış çalıştırmaları sağlamlaştırma">
    - Yalıtılmış çalıştırmalar, tamamlandığında `cron:<jobId>` oturumlarına ait izlenen tarayıcı sekmelerini/işlemlerini kapatmak için mümkün olan en iyi çabayı gösterir ve iş için oluşturulan tüm paketlenmiş MCP çalışma zamanı örneklerini ana oturum ve özel oturum çalıştırmalarında kullanılan aynı paylaşılan kapatma yolu üzerinden imha eder. Temizleme hataları yok sayılır; böylece cron sonucu geçerliliğini korur.
    - Dar kapsamlı cron kendi kendini temizleme iznine sahip yalıtılmış çalıştırmalar, zamanlayıcı durumunu, yalnızca kendi işlerini içeren kendilerine göre filtrelenmiş bir listeyi ve bu işin çalıştırma geçmişini okuyabilir; yalnızca kendi işlerini kaldırabilir.
    - Yalıtılmış çalıştırmalar eski onay yanıtlarına karşı koruma sağlar: ilk sonuç yalnızca bir ara durum güncellemesiyse (`on it`, `pulling everything together` ve benzer ipuçları) ve hiçbir alt aracı nihai yanıttan hâlâ sorumlu değilse, OpenClaw teslimattan önce gerçek sonucu almak için bir kez daha istem gönderir.
    - Yapılandırılmış yürütme reddi meta verileri (iç içe hatası `SYSTEM_RUN_DENIED` veya `INVALID_REQUEST` ile başlayan node-host `UNAVAILABLE` sarmalayıcıları dâhil) tanınır; böylece engellenmiş bir komut başarılı çalıştırma olarak bildirilmez ve sıradan asistan metni yanlışlıkla ret olarak algılanmaz.
    - Yanıt yükü olmasa bile çalıştırma düzeyindeki aracı hataları iş hatası olarak sayılır; böylece model/sağlayıcı hataları, işi başarılı sayıp temizlemek yerine hata sayaçlarını artırır ve başarısızlık bildirimlerini tetikler.
    - Bir iş `timeoutSeconds` sınırına ulaştığında cron çalıştırmayı iptal eder ve kısa bir temizleme penceresi tanır. Bu sürede boşaltılmazsa, cron zaman aşımını kaydetmeden önce Gateway'in yönettiği temizleme söz konusu çalıştırmanın oturum sahipliğini zorla temizler; böylece kuyruğa alınmış sohbet çalışması eski bir işleme oturumunun arkasında takılı kalmaz.
    - Kurulum/başlatma takılmalarına aşamaya özgü bir zaman aşımı uygulanır (örneğin `cron: isolated agent setup timed out before runner start` veya `cron: isolated agent run stalled before execution start (last phase: context-engine)`). Bu gözetmenler, harici CLI işlemleri başlamadan önce bile gömülü ve CLI destekli sağlayıcıları kapsar ve uzun `timeoutSeconds` değerlerinden bağımsız olarak sınırlandırılır; böylece soğuk başlatma/kimlik doğrulama/bağlam hataları hızlıca görünür hâle gelir.

  </Accordion>
  <Accordion title="Görev uzlaştırması">
    Cron görev uzlaştırması önce çalışma zamanının sahipliğine, ikinci olarak kalıcı geçmişe dayanır: eski bir alt oturum satırı hâlâ mevcut olsa bile cron çalışma zamanı ilgili işi çalışıyor olarak izlemeye devam ettiği sürece etkin cron görevi canlı kalır. Çalışma zamanı işin sahipliğini bıraktıktan ve 5 dakikalık ek süre penceresi dolduktan sonra bakım kontrolleri, eşleşen `cron:<jobId>:<startedAt>` çalıştırmasına ait kalıcı çalıştırma günlüklerini ve iş durumunu inceler. Buradaki nihai sonuç görev defterini sonlandırır; aksi takdirde Gateway tarafından yönetilen bakım görevi `lost` olarak işaretleyebilir. Çevrimdışı CLI denetimi kalıcı geçmişten kurtarma yapabilir, ancak kendi boş işlem içi etkin iş kümesi, Gateway'in yönettiği bir çalıştırmanın sona erdiğinin kanıtı değildir.
  </Accordion>
</AccordionGroup>

## Zamanlama türleri

| Tür       | CLI bayrağı         | Açıklama                                                                                                 |
| --------- | ------------------ | -------------------------------------------------------------------------------------------------------- |
| `at`      | `--at`             | Tek seferlik zaman damgası (ISO 8601 veya `20m` gibi göreli)                                            |
| `every`   | `--every`          | Sabit aralık (`10m`, `1h`, `1d`)                                                                      |
| `cron`    | `--cron`           | İsteğe bağlı `--tz` içeren 5 alanlı veya 6 alanlı cron ifadesi                                        |
| `on-exit` | `--on-exit`        | İzlenen bir komut sonlandığında bir kez tetikle (olay tetikleyicisi; dönüş kapatmasından sonra yaşamayı sürdürür; isteğe bağlı `--on-exit-cwd`) |
| `stream`  | `--stream-command` | Denetlenen, uzun ömürlü bir komutun ürettiği toplu satırlardan tetikle                                    |

Saat dilimi içermeyen zaman damgaları UTC olarak değerlendirilir. Saat dilimi farkı içermeyen bir `--at` tarih-saatini yorumlamak veya bir cron ifadesini belirtilen IANA saat diliminde değerlendirmek için `--tz America/New_York` ekleyin. `--tz` içermeyen cron ifadeleri Gateway ana makinesinin saat dilimini kullanır. `--tz`, `--every` veya `--on-exit` ile geçerli değildir.

Tekrarlanan saat başı ifadeleri (dakika alanı `0`, saat alanı joker karakterli) yük artışlarını azaltmak için otomatik olarak en fazla 5 dakika dağıtılır. Kesin zamanlamayı zorlamak için `--exact`, açık bir pencere belirtmek için `--stagger 30s` kullanın (yalnızca cron zamanlamaları).

### Heartbeat görev geçişi

Eski heartbeat geçici alanı, yapılandırılmış bir `tasks:` bloğunu destekliyordu. Her girdiyi düzenlenebilir, sıradan bir ana oturum cron işine dönüştürmek için yükseltmeden sonra `openclaw doctor --fix` çalıştırın. Doctor, aralığı ve önceki son çalıştırma zamanlamasını korur, bloğu kaldırmadan önce işleri oluşturur ve yeniden çalıştırıldığında aynı bildirim anahtarlarını güvenle yakınsar.

Geçişi yapılan bu işler genel `systemEvent` yüklerini taşır; dolayısıyla `openclaw cron list`, `get`, `edit` ve `remove` ile cron aracı bunları diğer işler gibi yönetir. Yürütmeleri korumalı heartbeat görev uyandırmasını kullanır: etkin saatler, asgari aralık, taşma denetimi ve meşgul yeniden denemeleri uygulanmaya devam ederken cron her görevin bağımsız sıklığını yönetir. Aynı birleştirme penceresinde zamanı gelen işler tek bir heartbeat dönüşünü paylaşabilir. Heartbeat etkin saatlerinin dışında kalan zamanlanmış bir oluşum atlanır ve işin bir sonraki oluşumunda yeniden denenir.

Heartbeat geçici alanı artık yalnızca izleme metnidir. Çalışma zamanı heartbeat'leri `tasks:` metnini zamanlama olarak ayrıştırmaz; yeni tekrarlanan işleri cron ile oluşturun.

### Akış kaynakları

Bir akış zamanlaması, operatör tarafından yazılmış bir argv komutunu Gateway altında çalışır durumda tutar ve işi komutun stdout ve stderr satırlarından tetikler. Akış zamanlamaları olay güdümlüdür, hiçbir zaman zamana bağlı olarak vadesi gelmez ve uzun ömürlü komut tetikleyici betiklerle aynı gözetimsiz güven sınıfına sahip olduğundan `cron.triggers.enabled: true` gerektirir. İşin devre dışı bırakılması veya kaldırılması işlemi durdurur; Gateway kapanışı işlem ağacının kapatılmasını bekler. Hızlı hatalar cron'un yerleşik hata geri çekilmesiyle yeniden başlatılır. Art arda 60 saniyeden kısa süren beş çalıştırma, işi hata durumunda bırakır ve normal başarısızlık uyarısı yolunu kullanır; yeniden başlatma sınırını temizlemek için işi elle yeniden etkinleştirin.

```bash
openclaw cron add \
  --name "Build event stream" \
  --stream-command '["node","scripts/build-events.mjs"]' \
  --stream-mode match \
  --stream-match '^(failed|recovered):' \
  --stream-batch-ms 250 \
  --session isolated \
  --message "Bu derleme olaylarını araştırın."
```

`mode: "line"` (varsayılan) her satırı kabul eder. `mode: "match"` yalnızca derlenmiş `match` düzenli ifadesiyle eşleşen satırları kabul eder. Bir toplu iş, `batchMs` boyunca sessizlikten sonra (varsayılan 250 ms, 50–5000 aralığına sabitlenir) veya `maxBatchBytes` değerinde (varsayılan 16384, 1024–65536 aralığına sabitlenir) kapanır. Bayt sınırında toplu iş `[truncated]` ile sona erer. Eşleşme modu, `maxBatchBytes` sınırını aşsa bile tam satırları her zaman tam metinleriyle karşılaştırır (yalnızca teslim edilen toplu iş kırpılır); sınırlandırılmış ham alım sınırında kesilen bir satır yalnızca bir önektir, bu nedenle sona sabitlenmiş bir örüntünün kesik kısımda tetiklenmesine izin vermek yerine eşleşmemiş sayılır. Toplu iş, sistem olayı metnine veya aracı dönüşü iletisine eklenir. Kaynak komut ile yük komutu arasında belirsiz işlem sahipliği oluşacağından, komut yükleri akış zamanlamaları için reddedilir.

İş başına yalnızca bir yük tetiklemesi ve sınırlandırılmış bir bekleyen toplu iş tutulur. Bir yük çalışırken veya yerleşik 30 saniyelik tetikleyici aralığı dolmadan önce gelen satırlar, sınırsız bir kuyruk oluşturmak yerine bekleyen toplu işte birleştirilir. Tek bir serileştirilmiş sahip, geçit bırakmalarını, yük hatalarını ve çalışmayan gönderimleri `streamDroppedBatches` içinde kaydeder; sınırlandırılmış birleştirmeler `streamCoalescedBatches` değerini artırır. Başarısız yükler, eşgüçlü olmayabileceklerinden yeniden denenmez. Mantıksal kaynak kimliği denetlenen alt süreç yeniden başlatmaları boyunca kararlı kalır, ancak kaynak devre dışı bırakıldığında, kaldırıldığında veya değiştirildiğinde yenilenir; böylece kullanımdan kaldırılmış kaynaktan kuyruğa alınan toplu işler A'dan B'ye ve yeniden A'ya düzenlemesinden sonra bile tetiklenemez. Durdurma tamamlandıktan sonra eski bir alt süreçten gelen geç geri çağırmalar etkisizdir. V1 yerel bir WebSocket kaynağı içermez; `websocat wss://example.invalid/events` gibi bir argv komutuyla köprüleyin.

Bir akış işi ayrıca `trigger.script` içerdiğinde geçit, kapatılan her toplu iş için bir kez çalışır. Geçerli toplu iş, `trigger.state` ile birlikte derin dondurulmuş `trigger.streamBatch` dizesi olarak kullanılabilir. `fire: false`, geçit durumunu kalıcı olarak sakladıktan sonra bu toplu işi bırakır. `fire: true` mevcut tetikleyici ileti semantiğini korur, ardından toplu işi oluşan yüke ekler. Bir akış işi bunun yerine koşul geçidi olmadan bir betik yükü kullanabilir; bu betik toplu işi aynı `trigger.streamBatch` değeri üzerinden alır. Her ikisi de kalıcı `trigger.state` yuvasının sahibi olacağından, bir betik yükünün koşul geçidiyle birleştirilmesi reddedilir.

### Dinamik sıklık (hız ayarlama)

Tekrarlanan işler `pacing.min` ve/veya `pacing.max` değerlerini `15m` ya da `4h` gibi süre dizelerine ayarlayabilir; en az bir sınır gereklidir. `cron add|edit` ile `--pacing-min` ve `--pacing-max` kullanın (`--clear-pacing` her iki sınırı da kaldırır).

Yalıtılmış bir çalıştırma sırasında, hız ayarlı bir iş `cron` aracını `action: "next_check"` ve `in: "30m"` ile çağırabilir. Öneri yalnızca o anda çalışmakta olan işe uygulanır ve başarılı çalıştırmanın tamamlanmasından itibaren ölçülür. OpenClaw bunu yapılandırılmış sınırlar içinde sessizce sınırlar.

Öneri olmadan hız ayarlama, normal zamanlamayı değiştirmez. Başarısız olan, zaman aşımına uğrayan ve atlanan çalıştırmalar öneriyi iptal eder; böylece mevcut yeniden deneme ve hata geri çekilme davranışı öncelikli olur. Yinelenen bir işi manuel olarak zorlamak bant dışıdır ve bekleyen doğal ya da hız ayarlı zaman dilimini korur. Koşulla tetiklenen işlerde, bir öneri daha erken denetim istese bile yerleşik asgari aralık alt sınır olmaya devam eder.

### Ayın günü ve haftanın günü VEYA mantığını kullanır

Cron ifadeleri [croner](https://github.com/Hexagon/croner) tarafından ayrıştırılır. Hem ayın günü hem de haftanın günü alanları joker karakter olmadığında croner, her iki alan da değil, alanlardan **herhangi biri** eşleştiğinde eşleşme sağlar. Bu, standart Vixie cron davranışıdır.

```bash
# Amaçlanan: "Yalnızca pazartesiye denk geliyorsa ayın 15'inde saat 09.00"
# Gerçekte:  "Her ayın 15'inde saat 09.00 VE her pazartesi saat 09.00"
0 9 15 * 1
```

Bu, ayda 0-1 kez yerine yaklaşık 5-6 kez tetiklenir. Her iki koşulu da zorunlu kılmak için croner'ın `+` haftanın günü değiştiricisini (`0 9 15 * +1`) kullanın ya da alanlardan birine göre zamanlayıp diğerini işinizin isteminde veya komutunda denetleyin.

## Olay tetikleyicileri (koşul izleyicileri)

Bir olay tetikleyicisi, `every`, `cron` veya `stream` zamanlamasına başsız bir koşul betiği ekler. Zaman zamanlamaları zamanı geldiğinde, akış zamanlamaları ise kapanan her toplu işlem için bunu değerlendirir. Cron, normal yükü yalnızca betik `fire: true` döndürdüğünde çalıştırır:

```json5
{
  schedule: { kind: "every", everyMs: 30000 },
  trigger: {
    // Yalnızca gözlemlenen durum son değerlendirmeden farklı olduğunda tetiklenir.
    script: "const res = await tools.call('exec', { command: 'gh pr checks 123 --json state -q \\'.[].state\\' | sort -u' }); const status = String(res?.result?.details?.aggregated ?? '').trim(); json({ fire: status !== trigger.state?.status, message: `PR 123 CI: ${trigger.state?.status ?? 'unknown'} -> ${status}`, state: { status } });",
    once: false,
  },
  payload: { kind: "agentTurn", message: "CI durum değişikliğini araştırın." },
}
```

Betik `{ fire, message?, state? }` döndürmelidir. Önceki JSON durumu, derinlemesine dondurulmuş `trigger.state` olarak kullanılabilir; akış kapıları ayrıca geçerli toplu işlemi `trigger.streamBatch` olarak alır. Kalıcılaştırmak için yeni bir `state` değeri döndürün. Durum 16 KB ile sınırlıdır. Tetiklenen bir sonuç `message` içerdiğinde cron, çalıştırmadan önce bunu sistem olayı metnine veya ajan turu iletisine ekler. `once: true`, ilk başarılı tetiklenmiş yükünden sonra işi devre dışı bırakır.

`fire: false`, değerlendirme durumunu ve sayaçları kalıcılaştırır, ardından çalıştırma geçmişi oluşturmadan yeniden zamanlar. Tetiklenmiş bir yük çalıştırması başarısız olursa döndürülen `state` **kalıcılaştırılmaz**; sonraki değerlendirme önceki durumu görür ve yeniden tetiklenebilir. Bu nedenle betikleri salt okunur denetimler olarak yazın ve eylemleri yükte tutun. Tetikleyici zamanlamalarının yerleşik asgari aralığı 30 saniyedir. Her değerlendirmenin 30 saniyelik gerçek zaman bütçesi ve en fazla 5 araç çağrısı vardır.

İzleyicileri yalnızca başarı etrafında değil, **eyleme dönüştürülebilir durum** etrafında oluşturun: denetimi başarısız olduğunda veya zaman aşımına uğradığında sessizleşen bir izleyici, bozukken sağlıklı görünür. Gözlemi `trigger.state` ile karşılaştırın ve yinelenenleri ayıklamak için güncel durum döndürün; model veya işlem belleğine güvenmeyin. Tetikleme sırasında `message` değerini kendi kendine yeterli hâle getirin çünkü bu değer, tetiklenen çalıştırmanın eksiksiz olay bağlamı olur.

<Warning>
`cron.triggers.enabled` seçeneğini etkinleştirmek, hem koşul tetikleyici betiklerinin hem de `script` yüklerinin, sahip ajanın **`exec` dâhil tam araç politikasıyla** başsız olarak çalışmasına izin verir. Bunu, söz konusu ajanın izinleriyle gözetimsiz kod yürütme olarak değerlendirin; cron işleri oluşturmasına izin verilen her ajana bu doğrultuda güvenilmiyorsa devre dışı bırakın.
</Warning>

Yerel bir betik dosyasından izleyici oluşturun (`-` betiği standart girdiden okur):

```bash
openclaw cron add \
  --name "PR CI watcher" \
  --every 30s \
  --trigger-script ./watch-pr-ci.js \
  --message "Respond to the CI status change" \
  --session isolated
```

## Yükler

Her iş, bayrakla seçilen tam olarak bir yük türü taşır:

| Yük           | Bayrak                                         | Çalıştırdığı                                               |
| ------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Sistem olayı  | `--system-event <text>`                        | Ana oturumda kuyruğa alınır, tek başına model çağrısı yapmaz |
| Ajan iletisi  | `--message <text>`                             | Model destekli bir ajan turu                               |
| Komut         | `--command <shell>` veya `--command-argv <json>` | Gateway ana makinesinde bir kabuk/işlem, model çağrısı yok |
| Betik         | `--script <file\|->`                           | Sahip ajanın araçlarını kullanan başsız kod modu betiği    |

Ek bir yük türü olan `heartbeat` sistem tarafından yönetilir: gateway, Heartbeat etkin her ajan için tek bir Heartbeat izleme işini yakınsar (bkz. [Heartbeat](/tr/gateway/heartbeat)). Bu, `cron list --all` içinde görünür ancak CLI veya API aracılığıyla oluşturulamaz ya da düzenlenemez. Heartbeat yapılandırması başlangıçta, yapılandırma yeniden yüklendiğinde veya `openclaw doctor --fix` tarafından kalıcı izleme zamanlamasına aktarılır. Cron devre dışı bırakıldığında izleyici işlemez ve yedek bir Heartbeat zamanlayıcısı çalışmaz.

### Ajan turu seçenekleri

<ParamField path="--message" type="string" required>
  İstem metni (yalıtılmış/geçerli/özel oturum işleri için zorunludur).
</ParamField>
<ParamField path="--model" type="string">
  Model geçersiz kılma; izin verilen bir modele çözümlenmelidir, aksi takdirde çalıştırma doğrulama hatasıyla başarısız olur.
</ParamField>
<ParamField path="--fallbacks" type="string">
  İş başına yedek model listesi; örneğin `--fallbacks openai/gpt-5.6-sol,openrouter/meta-llama/llama-3.3-70b-instruct:free`. Yedeksiz katı bir çalıştırma için `--fallbacks ""` geçirin.
</ParamField>
<ParamField path="--clear-fallbacks" type="boolean">
  `cron edit` üzerinde iş başına yedek geçersiz kılmayı kaldırarak işin yapılandırılmış yedek önceliğini izlemesini sağlar. `--fallbacks` ile birleştirilemez.
</ParamField>
<ParamField path="--clear-model" type="boolean">
  `cron edit` üzerinde iş başına model geçersiz kılmayı kaldırarak işin normal cron model önceliğini (saklanan cron oturumu geçersiz kılması, yoksa ajan/varsayılan model) izlemesini sağlar. `--model` ile birleştirilemez.
</ParamField>
<ParamField path="--thinking" type="string">
  Düşünme düzeyi geçersiz kılma (`off|minimal|low|medium|high|xhigh|adaptive|max|ultra`). Kullanılabilir düzeyler yine de seçilen modele ve ajan çalışma zamanına bağlıdır.
</ParamField>
<ParamField path="--clear-thinking" type="boolean">
  `cron edit` üzerinde iş başına düşünme geçersiz kılmayı kaldırır. `--thinking` ile birleştirilemez.
</ParamField>
<ParamField path="--light-context" type="boolean">
  Çalışma alanı önyükleme dosyası eklemeyi atlar.
</ParamField>
<ParamField path="--tools" type="string">
  İşin kullanabileceği araçları kısıtlar; örneğin `--tools exec,read`.
</ParamField>

Araç çalıştırabilen yeni işler her zaman açık bir araç politikası saklar. Bir ajan tarafından oluşturulan işler,
oluşturucu turun kullanabildiği araçlarla sınırlanır ve ajan
saklanan listeyi genişletemez. `--tools` olmadan kimliği doğrulanmış bir operatör tarafından oluşturulan işler,
kısıtlanmamış bir `*` politikası saklar; `cron edit --clear-tools` bu açık kısıtlanmamış
politikayı geri yükler. Açık bir araç politikasından önce oluşturulmuş mevcut işler, araç politikaları açıkça
düzenlenene veya iş yeniden oluşturulana kadar mevcut davranışlarını korur.

`--model`, işin birincil modelini ayarlar; oturumdaki `/model` geçersiz kılmasını değiştirmez, dolayısıyla yapılandırılmış yedek zincirleri bunun üzerinde uygulanmaya devam eder. Çözümlenemeyen veya izin verilmeyen bir model, sessizce varsayılana dönmek yerine çalıştırmayı açık bir doğrulama hatasıyla başarısız kılar. Bir işte `--model` varsa ancak açık veya yapılandırılmış bir yedek listesi yoksa OpenClaw, ajan birincil modelini gizli yeniden deneme hedefi olarak sessizce eklemek yerine boş bir yedek geçersiz kılması geçirir.

Yalıtılmış işler için model seçimi önceliği, en yüksekten başlayarak:

1. İş başına yük `model` (açık yapılandırma; izin verilmeyen bir model çalıştırmayı başarısız kılar)
2. Gmail kancası model geçersiz kılması (yalnızca çalıştırma Gmail'den geldiyse ve bu geçersiz kılmaya izin veriliyorsa)
3. Kullanıcı tarafından seçilen, saklanmış cron oturumu model geçersiz kılması
4. Ajan/varsayılan model seçimi

Hızlı mod çözümlenen canlı seçimi izler. Seçilen model yapılandırmasında `params.fastMode` varsa yalıtılmış cron bunu varsayılan olarak kullanır; saklanmış oturum `fastMode` geçersiz kılması (ardından ajan `fastModeDefault`) her iki yönde de model yapılandırmasına üstün gelmeye devam eder. Otomatik mod, varsayılan olarak 60 saniye olan modelin `params.fastAutoOnSeconds` kesme değerini kullanır.

Bir çalıştırma canlı model değiştirme devrine ulaşırsa cron, değiştirilen sağlayıcı/model ile yeniden dener ve bu seçimi (ve varsa yeni kimlik doğrulama profilini) etkin çalıştırma için kalıcılaştırır. Yeniden denemeler sınırlıdır: ilk deneme ve 2 değiştirme yeniden denemesinden sonra cron döngüye girmek yerine işlemi iptal eder.

Yalıtılmış bir çalıştırma başlamadan önce OpenClaw, `baseUrl` değeri geri döngü, özel ağ veya `.local` olan yapılandırılmış `api: "ollama"` ve `api: "openai-completions"` sağlayıcılarının erişilebilir yerel uç noktalarını denetler. Bu ön denetim, işin yapılandırılmış yedek zincirini tarar ve çalıştırmayı yalnızca tüm adaylara erişilemiyorsa `skipped` olarak işaretler; `--fallbacks ""` bu taramayı yalnızca birincil modelle sınırlı tutar. Çalışmayan bir uç nokta, model çağrısını başlatmak yerine çalıştırmayı açık bir hatayla `skipped` olarak kaydeder. Sonuç, uç nokta başına 5 dakika boyunca önbelleğe alınır (iş veya model başına değil); böylece aynı çalışmayan yerel Ollama/vLLM/SGLang/LM Studio sunucusunu paylaşan ve zamanı gelen çok sayıda iş, istek fırtınası yerine tek bir yoklama maliyetine neden olur. Ön denetimde atlanan çalıştırmalar yürütme hatası geri çekilmesini artırmaz; yinelenen atlama uyarılarını etkinleştirmek için `failureAlert.includeSkipped` değerini ayarlayın.

### Komut yükleri

Komut yükleri, model destekli bir tur başlatmadan Gateway zamanlayıcısı içinde deterministik betikler çalıştırır. Gateway ana makinesinde yürütülür, stdout/stderr çıktısını yakalar, çalıştırmayı cron geçmişine kaydeder ve ajan turu işleriyle aynı `announce`, `webhook` ve `none` teslim modlarını yeniden kullanır.

<Note>
Komut cron'u bir operatör-yönetici Gateway otomasyon yüzeyidir; bir ajan `tools.exec` çağrısı değildir. Cron işlerini oluşturmak, güncellemek, kaldırmak veya manuel olarak çalıştırmak `operator.admin` gerektirir; zamanlanmış komut çalıştırmaları daha sonra Gateway işlemi içinde yönetici tarafından oluşturulmuş bu otomasyon olarak yürütülür. Ajan yürütme politikası (`tools.exec.mode`, onay istemleri, ajan başına araç izin listeleri), komut cron yüklerini değil, modele görünür yürütme araçlarını yönetir.
</Note>

```bash
openclaw cron create "*/15 * * * *" \
  --name "Queue depth probe" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>`, `argv: ["sh", "-lc", <shell>]` değerini saklar. Kabuk ayrıştırması olmadan tam argv yürütmesi için `--command-argv '["node","scripts/report.mjs"]'` kullanın. İsteğe bağlı `--command-env KEY=VALUE` (yinelenebilir), `--command-input`, `--timeout-seconds` (varsayılan 10 dakika), `--no-output-timeout-seconds` ve `--output-max-bytes`; işlem ortamını, standart girdiyi ve çıktı sınırlarını denetler.

Teslim edilen metin işlem çıktısından türetilir: boş olmayan stdout önceliklidir; stdout boş ve stderr boş değilse stderr teslim edilir; her ikisi de varsa cron küçük bir `stdout:` / `stderr:` bloğu gönderir. `0` çıkış kodu çalıştırmayı `ok` olarak kaydeder; sıfır olmayan çıkış, sinyal, zaman aşımı veya çıktı yok zaman aşımı `error` olarak kaydedilir ve başarısızlık uyarılarını tetikleyebilir. Yalnızca `NO_REPLY` yazdıran bir komut, normal cron sessiz belirteç bastırmasını kullanır ve sohbete hiçbir şey göndermez.

### Betik yükleri

Komut dosyası yükleri, konuşmaya dayalı bir aracı turu başlatmadan tetikleyici komut dosyalarıyla aynı kod modu yürütücüsünde gözetimsiz çalışır. Bunları oluşturmadan veya çalıştırmadan önce `cron.triggers.enabled` özelliğini etkinleştirin; bu tehlikeli otomasyon geçidi hem tetikleyici komut dosyalarını hem de komut dosyası yüklerini kapsar. Komut dosyası işleri yalnızca `main` ve `isolated` oturum hedeflerini destekler.

```bash
openclaw cron create "0 * * * *" \
  --name "Hourly queue check" \
  --script ./automation/check-queue.js \
  --script-timeout-seconds 300 \
  --script-tool-budget 50 \
  --session isolated \
  --announce
```

JavaScript'i bir dosyadan veya standart girdiden okumak için `--script <file|->` kullanın. Zaman aşımı varsayılan olarak 300 saniyedir ve en fazla 900 olabilir; araç bütçesi varsayılan olarak 50 çağrıdır ve en fazla 200 olabilir. Bu yük bütçeleri, daha küçük tetikleyici geçidi değerlendirme bütçelerinden ayrıdır.

Komut dosyası şu isteğe bağlı alanları içeren bir nesne döndürebilir:

- `notify`: İşin `announce`, `webhook` veya `none` teslim modu aracılığıyla teslim edilen metin. Atlanırsa hiçbir şey teslim edilmez. Bir `main` işi için metin, sistem olayına dönüşür.
- `wake`: `"now"`, `notify` (veya kısa bir tamamlanma olayı) kuyruğa eklendikten hemen sonra bir Heartbeat ister; `"next-heartbeat"`, olayı sonraki Heartbeat için kuyruğa ekler.
- `state`: 16 KB ile sınırlı ve yalnızca başarılı bir çalıştırmadan sonra kalıcı hâle getirilen JSON durumu. Sonraki çalıştırma, tetikleyici komut dosyalarında olduğu gibi `trigger.state` olarak sabitlenmiş bir kopya alır. Bu ad alanının tek bir kalıcı sahibi olduğundan, bir komut dosyası yükü aynı işte bir koşul tetikleyicisiyle birleştirilemez.
- `nextCheck`: `"15m"` gibi bir süre. Yalnızca ilerleme hızı etkinleştirilmiş işler için geçerlidir ve aracı turu önerileriyle aynı ilerleme hızı sınırlandırmasını kullanır.

İstisnalar, zaman aşımları, tükenmiş araç bütçeleri, geçersiz sonuçlar ve ilerleme hızı olmadan `nextCheck`, normal Cron çalıştırma hatalarıdır: döndürülen durumu kalıcı hâle getirmeden çalıştırma geçmişine, geri çekilme ve hata uyarısı işleme süreçlerine girerler.

## Yürütme biçimleri

| Biçim           | `--session` değeri   | Çalıştığı yer                  | En uygun olduğu durumlar                        |
| --------------- | ------------------- | ------------------------ | ------------------------------- |
| Ana oturum    | `main`              | Ayrılmış Cron uyandırma hattı | Hatırlatıcılar, sistem olayları        |
| Yalıtılmış        | `isolated`          | Ayrılmış `cron:<jobId>` | Raporlar, arka plan işleri      |
| Geçerli oturum | `current`           | Oluşturma sırasında bağlanır   | Bağlamı dikkate alan yinelenen işler    |
| Özel oturum  | `session:custom-id` | Kalıcı adlandırılmış oturum | Geçmişi temel alan iş akışları |

<AccordionGroup>
  <Accordion title="Ana oturum, yalıtılmış oturum ve özel oturum karşılaştırması">
    **Ana oturum** işleri, Cron'a ait bir çalıştırma hattına bir sistem olayı ekler ve isteğe bağlı olarak Heartbeat'i uyandırır (`--wake now` veya `--wake next-heartbeat`). Yanıtlar için hedef ana oturumun son teslim bağlamını kullanabilirler ancak rutin Cron turlarını insan sohbeti hattına eklemez ve hedef oturumun günlük/boşta sıfırlama güncelliğini uzatmazlar. **Yalıtılmış** işler, yeni bir oturumla ayrılmış bir aracı turu çalıştırır. **Özel oturumlar** (`session:xxx`), bağlamı çalıştırmalar arasında koruyarak önceki özetleri temel alan günlük durum toplantıları gibi iş akışlarını mümkün kılar.

    Ana oturum Cron olayları, kendi kendine yeterli sistem olayı hatırlatıcılarıdır. Varsayılan Heartbeat istemini veya Heartbeat izleyici taslağını otomatik olarak içermezler; bir hatırlatıcının bu bağlama başvurması gerekiyorsa bunu Cron olayı metninde açıkça belirtin.

  </Accordion>
  <Accordion title="Yalıtılmış işler için 'yeni oturum' ne anlama gelir?">
    Her çalıştırmada yeni bir döküm/oturum kimliği oluşturulur. OpenClaw güvenli tercihleri (düşünme/hızlı/ayrıntılı ayarları, etiketler, kullanıcı tarafından açıkça seçilen model/kimlik doğrulama geçersiz kılmaları) taşır ancak eski bir Cron satırından ortamdaki konuşma bağlamını devralmaz: kanal/grup yönlendirmesi, gönderme veya kuyruk ilkesi, yükseltme, kaynak ya da ACP çalışma zamanı bağlaması. Yinelenen bir işin bilinçli olarak aynı konuşma bağlamını temel alması gerektiğinde `current` veya `session:<id>` kullanın.
  </Accordion>
  <Accordion title="Gözetimsiz çalıştırma sözleşmesi">
    Yalıtılmış Cron ve kanca aracı turları açıkça gözetimsizdir: açıklama yapacak veya onay verecek kimse yoktur. Son yanıt; plan, onay mesajı veya girdi isteği yerine teslim edilecek çıktı olmalıdır. Yapılacak bir şey olmadığında aracı `HEARTBEAT_OK` döndürür ve hataları açıkça belirtir; yeniden deneme ve hata uyarısı ilkesi Cron'a aittir.

    Güvenilir zamanlanmış işlerde, bilinçli olarak bir soru veya plan istendiğinde işin kendi talimatları önceliklidir ve aracı artık gerekmeyen bir işi kaldırabilir. Harici kanca turları yalnızca ortak gözetimsiz sözleşmeyi alır; harici içerik sınırı boyunca bu geçersiz kılmayı veya kendi kendini kaldırma yönergelerini almazlar.

  </Accordion>
  <Accordion title="Alt aracı ve Discord teslimi">
    Yalıtılmış Cron çalıştırmaları alt aracıları yönettiğinde teslimat, güncelliğini yitirmiş üst aracı ara metni yerine son alt aracı çıktısını tercih eder. Alt aracılar hâlâ çalışıyorsa OpenClaw bu kısmi üst aracı güncellemesini duyurmak yerine bastırır.

    Yalnızca metin destekleyen Discord duyuru hedeflerinde OpenClaw, hem akış hâlindeki/ara metni hem de son yanıtı yeniden oynatmak yerine standart son asistan metnini bir kez gönderir. Eklerin ve bileşenlerin kaybolmaması için medya ve yapılandırılmış Discord yükleri yine ayrı olarak teslim edilir.

  </Accordion>
</AccordionGroup>

## Teslimat ve çıktı

| Mod       | Gerçekleşen işlem                                                        |
| ---------- | ------------------------------------------------------------------- |
| `announce` | Aracı göndermediyse son metni yedek olarak hedefe teslim eder |
| `webhook`  | Tamamlanan olay yükünü bir URL'ye POST eder                                |
| `none`     | Çalıştırıcı tarafından yedek teslimat yapılmaz                                         |

Kanal teslimatı için `--announce --channel telegram --to "-1001234567890"` kullanın. Telegram forum konuları için `-1001234567890:topic:123` kullanın; OpenClaw, Telegram'a ait `-1001234567890:123` kısa biçimini de kabul eder. Doğrudan RPC/yapılandırma çağıranları `delivery.threadId` değerini dize veya sayı olarak iletebilir. Slack/Discord/Mattermost hedefleri açık ön ekler kullanır (`channel:<id>`, `user:<id>`). Matrix oda kimlikleri büyük/küçük harfe duyarlıdır; tam oda kimliğini veya Matrix'teki `room:!room:server` biçimini kullanın.

Duyuru teslimatı `channel: "last"` kullandığında veya `channel` atlandığında, `telegram:123` gibi sağlayıcı ön ekli bir hedef, Cron oturum geçmişine veya yapılandırılmış tek bir kanala geri dönmeden önce kanalı seçebilir. Yalnızca yüklenen Plugin tarafından duyurulan ön ekler sağlayıcı seçicisidir. `delivery.channel` açıkça belirtilirse hedef ön eki aynı sağlayıcıyı adlandırmalıdır; WhatsApp'ın Telegram kimliğini telefon numarası olarak yorumlamasına izin vermek yerine `channel: "whatsapp"` ile `to: "telegram:123"` reddedilir. Hedef türü ve hizmet ön ekleri (`channel:<id>`, `user:<id>`, `imessage:<handle>`, `sms:<number>`), sağlayıcı seçicileri değil, kanala ait hedef söz dizimi olarak kalır.

Yalıtılmış işler için sohbet teslimatı ortaktır: bir sohbet rotası varsa aracı, `--no-deliver` ile bile `message` aracını kullanabilir. Aracı yapılandırılmış/geçerli hedefe gönderim yaparsa OpenClaw yedek duyuruyu atlar. Aksi takdirde `announce`, `webhook` ve `none` yalnızca aracı turundan sonra çalıştırıcının son yanıtla ne yapacağını denetler.

Bir aracı etkin bir sohbetten yalıtılmış bir hatırlatıcı oluşturduğunda OpenClaw, korunmuş canlı teslimat hedefini yedek duyuru rotası için saklar. Dahili oturum anahtarları küçük harfli olabilir; geçerli sohbet bağlamı kullanılabildiğinde sağlayıcı teslimat hedefleri bu anahtarlardan yeniden oluşturulmaz.

Örtük duyuru teslimatı, eski hedefleri doğrulamak ve yeniden yönlendirmek için yapılandırılmış kanal izin listelerini kullanır. DM eşleştirme deposu onayları yedek otomasyon alıcıları değildir; zamanlanmış bir işin bir DM'ye proaktif olarak gönderim yapması gerektiğinde `delivery.to` ayarlayın veya kanal `allowFrom` girdisini yapılandırın.

### Hata bildirimleri

Hata bildirimleri ayrı bir hedef yolu izler:

- `cron.failureDestination`, hata bildirimleri için genel bir varsayılan ayarlar.
- `job.delivery.failureDestination`, bunu iş bazında geçersiz kılar.
- İkisi de ayarlanmamışsa ve iş zaten `announce` aracılığıyla teslimat yapıyorsa hata bildirimleri bu birincil duyuru hedefine geri döner.
- `delivery.failureDestination`, birincil teslimat modu `webhook` olmadığı sürece yalnızca `sessionTarget="isolated"` işlerinde desteklenir.
- `failureAlert.includeSkipped: true`, bir işi veya genel Cron uyarı ilkesini yinelenen atlanmış çalıştırma uyarılarına dahil eder. Atlanan çalıştırmalar ayrı bir ardışık atlama sayacını koruduğundan yürütme hatası geri çekilmesini etkilemez.
- `openclaw cron edit`, iş bazında uyarı ayarlamasını kullanıma sunar: `--failure-alert`/`--no-failure-alert`, `--failure-alert-after <n>`, `--failure-alert-channel`, `--failure-alert-to`, `--failure-alert-cooldown`, `--failure-alert-include-skipped`/`--failure-alert-exclude-skipped`, `--failure-alert-mode` ve `--failure-alert-account-id`.

### Çıktı dili

Cron işleri kanal, yerel ayar veya önceki mesajlardan bir yanıt dili çıkarmaz. Dil kuralını zamanlanmış mesaja veya şablona ekleyin:

```bash
openclaw cron edit <jobId> \
  --message "Güncellemeleri özetle. Çince yanıt ver; URL'leri, kodu ve ürün adlarını değiştirmeden koru."
```

Şablon dosyalarında dil talimatını oluşturulan istemde tutun ve iş çalışmadan önce `{{language}}` gibi yer tutucuların doldurulduğunu doğrulayın. Çıktı dilleri karıştırıyorsa kuralı açıkça belirtin; örneğin: "Anlatım metninde Çince kullan ve teknik terimleri İngilizce olarak koru."

## CLI örnekleri

<Tabs>
  <Tab title="Tek seferlik hatırlatıcı">
    ```bash
    openclaw cron add \
      --name "Calendar check" \
      --at "20m" \
      --session main \
      --system-event "Next heartbeat: check calendar." \
      --wake now
    ```
  </Tab>
  <Tab title="Yinelenen yalıtılmış iş">
    ```bash
    openclaw cron create "0 7 * * *" \
      "Summarize overnight updates." \
      --name "Morning brief" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --announce \
      --channel slack \
      --to "channel:C1234567890"
    ```
  </Tab>
  <Tab title="Model ve düşünme geçersiz kılması">
    ```bash
    openclaw cron add \
      --name "Deep analysis" \
      --cron "0 6 * * 1" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --message "Weekly deep analysis of project progress." \
      --model "opus" \
      --thinking high \
      --announce
    ```
  </Tab>
  <Tab title="Webhook çıktısı">
    ```bash
    openclaw cron create "0 18 * * 1-5" \
      "Summarize today's deploys as JSON." \
      --name "Deploy digest" \
      --webhook "https://example.invalid/openclaw/cron"
    ```
  </Tab>
  <Tab title="Komut çıktısı">
    ```bash
    openclaw cron create "*/15 * * * *" \
      --name "Queue depth probe" \
      --command "scripts/check-queue.sh" \
      --command-cwd "/srv/app" \
      --announce \
      --channel telegram \
      --to "-1001234567890"
    ```
  </Tab>
</Tabs>

## İşleri yönetme

```bash
# Etkin işleri listele
openclaw cron list

# Devre dışı işleri dahil et
openclaw cron list --all

# Saklanan bir işi JSON olarak al
openclaw cron get <jobId>

# Çözümlenmiş teslimat rotası dahil bir işi göster
openclaw cron show <jobId>

# Silmeden etkinleştir/devre dışı bırak
openclaw cron enable <jobId>
openclaw cron disable <jobId>

# Bir işi düzenle
openclaw cron edit <jobId> --message "Güncellenmiş istem" --model "opus"

# Bir işi şimdi zorla çalıştır
openclaw cron run <jobId>

# Bir işi şimdi zorla çalıştır ve son durumunu bekle
openclaw cron run <jobId> --wait --wait-timeout 10m --poll-interval 2s

# Yalnızca zamanı geldiyse çalıştır
openclaw cron run <jobId> --due

# Çalıştırma geçmişini görüntüle
openclaw cron runs --id <jobId> --limit 50

# Belirli bir çalıştırmayı görüntüle
openclaw cron runs --id <jobId> --run-id <runId>

# Bir işi sil
openclaw cron remove <jobId>

# Ajan seçimi (çok ajanlı kurulumlar)
openclaw cron create "0 6 * * *" "Operasyon kuyruğunu kontrol et" --name "Operasyon taraması" --session isolated --agent ops
openclaw cron edit <jobId> --clear-agent
```

Bir oturumun arşivlenmesi (Control UI veya operatör-yönetici çağırıcısından `sessions.patch { archived: true }`), o oturuma bağlı tüm etkin cron işlerini devre dışı bırakır: yalıtılmış `cron:<jobId>` oturumu, bir `session:<key>` hedefi veya bir teslimat/uyandırma `sessionKey` hattı. Oturumun geri yüklenmesi bu işleri yeniden etkinleştirmez; `openclaw cron enable <jobId>` kullanın. Etkin bir bağlı işi bulunan oturumlar, Control UI kenar çubuğunda saat rozeti gösterir.

`openclaw cron run <jobId>`, manuel çalıştırmayı kuyruğa aldıktan sonra döner. Kuyruğa alınan çalıştırma tamamlanana kadar engellemesi gereken kapatma kancaları, bakım betikleri veya diğer otomasyonlar için `--wait` kullanın; döndürülen `runId` değerini yoklar (varsayılan zaman aşımı `10m`, yoklama aralığı `2s`) ve `ok` durumu için `0` koduyla, `error`, `skipped` veya bekleme zaman aşımı için sıfırdan farklı bir kodla çıkar.

Ajanın `cron` aracı, `cron(action: "list")` üzerinden kompakt iş özetleri (`id`, `name`, `enabled`, `nextRunAtMs`, `scheduleKind`, `lastRunStatus`) döndürür; eksiksiz bir iş tanımı için `cron(action: "get", jobId: "...")` kullanın. Doğrudan Gateway çağırıcıları, `cron.list` öğesine `compact: true` iletebilir; bunun atlanması, teslimat önizlemeleriyle birlikte tam yanıtı korur.

`openclaw cron create`, `openclaw cron add` için bir diğer addır. Yeni işler, konumsal bir zamanlamanın (`"0 9 * * 1"`, `"every 1h"`, `"20m"` veya bir ISO zaman damgası) ardından konumsal bir ajan istemi kullanabilir. Tamamlanan çalıştırma yükünü bir HTTP uç noktasına POST etmek için `cron add|create` veya `cron edit` üzerinde `--webhook <url>` kullanın; Webhook teslimatı, sohbet teslimatı bayraklarıyla (`--announce`, `--channel`, `--to`, `--thread-id`, `--account`) birleştirilemez. `cron edit`, `--clear-channel`, `--clear-to`, `--clear-thread-id` ve `--clear-account` üzerinde bu yönlendirme alanlarını ayrı ayrı kaldırın (her biri eşleşen ayarlama bayrağıyla birlikte kullanıldığında reddedilir) — bu, yalnızca çalıştırıcı geri dönüş teslimatını devre dışı bırakan `--no-deliver` seçeneğinden farklıdır.

<Note>
Model geçersiz kılma notu:

- `openclaw cron add|edit --model ...`, işin seçili modelini değiştirir.
- Modele izin veriliyorsa tam olarak bu sağlayıcı/model, yalıtılmış ajan çalıştırmasına ulaşır.
- Modele izin verilmiyorsa veya model çözümlenemiyorsa cron, açık bir doğrulama hatasıyla çalıştırmayı başarısız kılar.
- API `cron.update` yük yamaları, saklanan iş modeli geçersiz kılmasını temizlemek için `model: null` değerini ayarlayabilir.
- `openclaw cron edit <job-id> --clear-model`, bu geçersiz kılmayı CLI üzerinden temizler (`model: null` yamasıyla aynı etki) ve `--model` ile birleştirilemez.
- Yapılandırılmış geri dönüş zincirleri uygulanmaya devam eder; çünkü cron `--model` bir iş birincilidir, oturum `/model` geçersiz kılması değildir.
- `openclaw cron add|edit --fallbacks ...`, yük `fallbacks` değerini ayarlayarak bu iş için yapılandırılmış geri dönüşleri değiştirir; `--fallbacks ""` geri dönüşü devre dışı bırakır ve çalıştırmayı katı hâle getirir. `openclaw cron edit <job-id> --clear-fallbacks`, iş başına geçersiz kılmayı temizler.
- Açık veya yapılandırılmış bir geri dönüş listesi bulunmayan düz bir `--model`, sessiz bir ek yeniden deneme hedefi olarak ajan birinciline geçmez.

</Note>

## Webhook'lar

Gateway, dış tetikleyiciler için HTTP Webhook uç noktalarını kullanıma açabilir. Yapılandırmada etkinleştirin:

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
  },
}
```

### Kimlik doğrulama

Her istek, kanca belirtecini başlık üzerinden içermelidir:

- `Authorization: Bearer <token>` (önerilir)
- `x-openclaw-token: <token>`

Sorgu dizesi belirteçleri reddedilir.

<AccordionGroup>
  <Accordion title="POST /hooks/wake">
    Ana oturum için bir sistem olayını kuyruğa alın:

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/wake \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"text":"Yeni e-posta alındı","mode":"now"}'
    ```

    <ParamField path="text" type="string" required>
      Olay açıklaması.
    </ParamField>
    <ParamField path="mode" type="string" default="now">
      `now` veya `next-heartbeat`.
    </ParamField>

  </Accordion>
  <Accordion title="POST /hooks/agent">
    Yalıtılmış bir ajan dönüşü çalıştırın:

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/agent \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"message":"Gelen kutusunu özetle","name":"E-posta","model":"openai/gpt-5.6-sol"}'
    ```

    Alanlar: `message` (zorunlu), `name`, `agentId`, `sessionKey` (`hooks.allowRequestSessionKey=true` gerektirir), `idempotencyKey`, `wakeMode`, `deliver`, `channel`, `to`, `model`, `thinking`, `timeoutSeconds`.

  </Accordion>
  <Accordion title="Eşlenmiş kancalar (POST /hooks/<name>)">
    Özel kanca adları, yapılandırmadaki `hooks.mappings` üzerinden çözümlenir. Eşlemeler, rastgele yükleri şablonlar veya kod dönüşümleriyle `wake` ya da `agent` eylemlerine dönüştürebilir.
  </Accordion>
</AccordionGroup>

<Warning>
Kanca uç noktalarını geri döngü, tailnet veya güvenilir bir ters proxy arkasında tutun.

- Özel bir kanca belirteci kullanın; Gateway kimlik doğrulama belirteçlerini yeniden kullanmayın.
- `hooks.path` öğesini özel bir alt yolda tutun; `/` reddedilir.
- Bir kancanın hedefleyebileceği etkin ajanı sınırlamak için `hooks.allowedAgentIds` değerini ayarlayın; buna `agentId` atlandığında varsayılan ajan da dahildir.
- Çağırıcının seçtiği oturumlara ihtiyacınız yoksa `hooks.allowRequestSessionKey=false` değerini koruyun.
- `hooks.allowRequestSessionKey` özelliğini etkinleştirirseniz izin verilen oturum anahtarı biçimlerini sınırlamak için `hooks.allowedSessionKeyPrefixes` değerini de ayarlayın.
- Kanca yükleri varsayılan olarak güvenlik sınırlarıyla sarmalanır.

</Warning>

## Gmail PubSub entegrasyonu

Gmail gelen kutusu tetikleyicilerini Google PubSub aracılığıyla OpenClaw'a bağlayın.

<Note>
**Ön koşullar:** `gcloud` CLI, `gog` (gogcli), OpenClaw kancalarının etkinleştirilmiş olması, herkese açık HTTPS uç noktası için Tailscale.
</Note>

### Sihirbazla kurulum (önerilir)

```bash
openclaw webhooks gmail setup --account openclaw@gmail.com
```

Bu işlem `hooks.gmail` yapılandırmasını yazar, Gmail ön ayarını etkinleştirir ve gönderim uç noktası (`--tailscale funnel|serve|off`) için varsayılan olarak Tailscale Funnel'ı kullanır.

<Warning>
Gmail ön ayarının ileti başına oturumu konuşma bağlamını ayırır; hedef ajanın araçlarını veya çalışma alanını kısıtlamaz. `agentId` değerini ayarlayan özel bir eşleme olmadan Gmail kancaları varsayılan ajan olarak çalışır.

Güvenilmeyen gelen kutuları için kancayı özel bir okuyucu ajana yönlendirin, bu ajana salt okunur çalışma alanı erişimi verin veya hiç çalışma alanı erişimi vermeyin ve dosya sistemi yazma, kabuk, tarayıcı ve diğer gereksiz araçları engelleyin. Ana ajanı bilgilendirmesi gerekiyorsa yalnızca gerekli ajandan ajana devretmeye izin verin. Bkz. [İstem enjeksiyonu](/tr/gateway/security#prompt-injection), [Çok ajanlı korumalı alan ve araçlar](/tr/tools/multi-agent-sandbox-tools) ve [`tools.agentToAgent`](/tr/gateway/config-tools#toolsagenttoagent).
</Warning>

### Gateway otomatik başlatma

`hooks.enabled=true` ve `hooks.gmail.account` ayarlandığında Gateway, açılışta `gog gmail watch serve` öğesini başlatır ve izlemeyi otomatik olarak yeniler. Devre dışı bırakmak için `OPENCLAW_SKIP_GMAIL_WATCHER=1` değerini ayarlayın.

### Tek seferlik manuel kurulum

<Steps>
  <Step title="GCP projesini seçin">
    `gog` tarafından kullanılan OAuth istemcisinin sahibi olan GCP projesini seçin:

    ```bash
    gcloud auth login
    gcloud config set project <project-id>
    gcloud services enable gmail.googleapis.com pubsub.googleapis.com
    ```

  </Step>
  <Step title="Konu oluşturun ve Gmail gönderim erişimi verin">
    ```bash
    gcloud pubsub topics create gog-gmail-watch
    gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
      --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
      --role=roles/pubsub.publisher
    ```
  </Step>
  <Step title="İzlemeyi başlatın">
    ```bash
    gog gmail watch start \
      --account openclaw@gmail.com \
      --label INBOX \
      --topic projects/<project-id>/topics/gog-gmail-watch
    ```
  </Step>
</Steps>

### Gmail model geçersiz kılması

```json5
{
  hooks: {
    gmail: {
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

Güvenilmeyen gelen kutuları için sağlayıcınızda bulunan en yeni nesil ve en üst düzey modeli kullanın. Yukarıdaki değer bir örnektir; model, yapılandırılmış kataloğunuzda ve izin listenizde bulunmalıdır.

## Yapılandırma

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    triggers: {
      enabled: false,
    },
    webhookToken: "replace-with-dedicated-webhook-token",
    sessionRetention: "24h",
  },
}
```

`webhookToken`, cron Webhook POST isteklerinde `Authorization: Bearer <token>` olarak gönderilir.

`cron.store`, mantıksal bir depo anahtarı ve doctor geçiş yoludur; elle düzenlenecek etkin bir JSON dosyası değildir. İş verileri SQLite'ta bulunur; değişiklikler için CLI veya Gateway API'sini kullanın.

Cron'u devre dışı bırakın: `cron.enabled: false` veya `OPENCLAW_SKIP_CRON=1`.

<AccordionGroup>
  <Accordion title="Yeniden deneme davranışı">
    **Tek seferlik yeniden deneme**: geçici hatalar (hız sınırı, aşırı yük, ağ, zaman aşımı, sunucu hatası) yerleşik bir yeniden deneme zamanlaması kullanır. Kalıcı hatalar işi hemen devre dışı bırakır.

    **Yinelenen yeniden deneme**: ardışık yürütme hataları, genişletilmiş bir zamanlamayla geri çekilir (30s, 60s, 5m, 15m, 60m). Geri çekilme, sonraki başarılı çalıştırmadan sonra sıfırlanır.

  </Accordion>
  <Accordion title="Bakım">
    `cron.sessionRetention` (varsayılan `24h`, `false` devre dışı bırakır) yalıtılmış çalıştırma oturumu girdilerini temizler. Çalıştırma geçmişi, iş başına en yeni 2000 sonlandırılmış satırı tutar; kayıp satırlar 24 saatlik temizleme aralığını korur.
  </Accordion>
  <Accordion title="Eski depo geçişi">
    Yükseltme sırasında eski `~/.openclaw/cron/jobs.json`, `jobs-state.json` ve `runs/*.jsonl` dosyalarını SQLite'a aktarmak ve bunları `.migrated` son ekiyle yeniden adlandırmak için `openclaw doctor --fix` komutunu çalıştırın. Hatalı biçimlendirilmiş iş satırları çalışma zamanında atlanır ve daha sonra onarılmak veya incelenmek üzere `jobs-quarantine.json` konumuna kopyalanır.
  </Accordion>
</AccordionGroup>

## Sorun giderme

### Komut sıralaması

```bash
openclaw status
openclaw gateway status
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
openclaw doctor
```

<AccordionGroup>
  <Accordion title="Cron tetiklenmiyor">
    - `cron.enabled` ve `OPENCLAW_SKIP_CRON` ortam değişkenini kontrol edin.
    - Gateway'in kesintisiz çalıştığını doğrulayın.
    - `cron` zamanlamaları için saat dilimini (`--tz`) ana makinenin saat dilimiyle karşılaştırarak doğrulayın.
    - Çalıştırma çıktısındaki `reason: not-due`, manuel çalıştırmanın `openclaw cron run <jobId> --due` ile denetlendiği ve işin zamanının henüz gelmediği anlamına gelir.

  </Accordion>
  <Accordion title="Cron tetiklendi ancak teslimat yapılmadı">
    - Teslimat modu `none`, çalıştırıcının yedek gönderim yapmasının beklenmediği anlamına gelir. Bir sohbet rotası mevcut olduğunda aracı, `message` aracını kullanarak yine de doğrudan gönderim yapabilir.
    - Teslimat hedefinin eksik/geçersiz olması (`channel`/`to`), giden iletinin atlandığı anlamına gelir.
    - Matrix için, küçük harfe dönüştürülmüş `delivery.to` oda kimliklerine sahip kopyalanmış veya eski işler başarısız olabilir; çünkü Matrix oda kimlikleri büyük/küçük harfe duyarlıdır. İşi, Matrix'teki tam `!room:server` veya `room:!room:server` değerine göre düzenleyin.
    - Kanal kimlik doğrulama hataları (`unauthorized`, `Forbidden`), teslimatın kimlik bilgileri tarafından engellendiği anlamına gelir.
    - Yalıtılmış çalıştırma yalnızca sessiz belirteci (`NO_REPLY` / `no_reply`) döndürürse OpenClaw, doğrudan giden teslimatı ve yedek sıraya alınmış özet yolunu engeller; dolayısıyla sohbete hiçbir şey gönderilmez.
    - Aracının kullanıcıya kendisinin mesaj göndermesi gerekiyorsa işin kullanılabilir bir rotası olduğunu doğrulayın (önceki bir sohbetle `channel: "last"` veya açıkça belirtilmiş bir kanal/hedef).

  </Accordion>
  <Accordion title="Cron veya Heartbeat, /new tarzı geçişi engelliyor gibi görünüyor">
    - Günlük ve boşta kalma sıfırlamasının güncelliği `updatedAt` temel alınarak belirlenmez; bkz. [Oturum yönetimi](/tr/concepts/session#session-lifecycle).
    - Cron uyandırmaları, Heartbeat çalıştırmaları, exec bildirimleri ve Gateway kayıt işlemleri, yönlendirme/durum için oturum satırını güncelleyebilir ancak `sessionStartedAt` veya `lastInteractionAt` sürelerini uzatmaz.
    - Bu alanlar mevcut olmadan önce oluşturulan eski satırlarda OpenClaw, dosya hâlâ kullanılabiliyorsa transkript JSONL oturum başlığından `sessionStartedAt` değerini kurtarabilir. `lastInteractionAt` içermeyen eski boşta kalma satırları, kurtarılan bu başlangıç zamanını boşta kalma temel değeri olarak kullanır.

  </Accordion>
  <Accordion title="Saat dilimiyle ilgili dikkat edilmesi gerekenler">
    - `--tz` olmadan Cron, Gateway ana makinesinin saat dilimini kullanır.
    - Saat dilimi belirtilmeyen `at` zamanlamaları UTC olarak değerlendirilir.
    - Heartbeat `activeHours`, yapılandırılmış saat dilimi çözümlemesini kullanır.

  </Accordion>
</AccordionGroup>

## İlgili

- [Otomasyon](/tr/automation) — tüm otomasyon mekanizmalarına genel bakış
- [Arka Plan Görevleri](/tr/automation/tasks) — Cron yürütmelerinin görev defteri
- [Heartbeat](/tr/gateway/heartbeat) — periyodik ana oturum dönüşleri
- [Saat Dilimi](/tr/concepts/timezone) — saat dilimi yapılandırması
