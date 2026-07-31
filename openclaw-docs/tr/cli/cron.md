---
read_when:
    - Zamanlanmış işler ve uyandırmalar istiyorsunuz
    - Cron yürütmesi ve günlüklerinde hata ayıklıyorsunuz
summary: '`openclaw cron` için CLI başvurusu (arka plan işlerini zamanlama ve çalıştırma)'
title: Cron
x-i18n:
    generated_at: "2026-07-26T22:38:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5989a7558f4ae2f046480b6a52e3fa296c95d47b14b11c5bad709fea4af6af3e
    source_path: cli/cron.md
    workflow: 16
---

# `openclaw cron`

Gateway zamanlayıcısı için Cron işlerini yönetin.

<Tip>
Tam komut yüzeyi için `openclaw cron --help` komutunu çalıştırın. Kavramsal kılavuz için [Cron işleri](/tr/automation/cron-jobs) sayfasına bakın.
</Tip>

<Note>
Tüm Cron değişiklikleri (`add`/`create`, `update`/`edit`, `remove`, `run`) `operator.admin` gerektirir. Komut yükü çalıştırmaları, bir ajan `tools.exec` araç çağrısı olarak değil, doğrudan Gateway işlemi içinde yürütülür; `tools.exec.*` ve yürütme onayları, modelin görebildiği yürütme araçlarını yine yönetir.
</Note>

## Hızlıca işler oluşturma

`openclaw cron create`, `openclaw cron add` için bir diğer addır. Yeni işlerde önce zamanlamayı, ardından istemi yazın:

```bash
openclaw cron create "0 7 * * *" \
  "Gece boyunca gerçekleşen güncellemeleri özetle." \
  --name "Sabah özeti" \
  --agent ops
```

İşin tamamlanan yükü bir sohbet hedefine teslim etmek yerine POST ile göndermesi gerektiğinde `--webhook <url>` kullanın:

```bash
openclaw cron create "0 18 * * 1-5" \
  "Bugünkü dağıtımları JSON olarak özetle." \
  --name "Dağıtım özeti" \
  --webhook "https://example.invalid/openclaw/cron"
```

Yalıtılmış bir ajan/model çalıştırması başlatmadan OpenClaw Cron içinde çalışan, deterministik kabuk tarzı işler için `--command` kullanın:

```bash
openclaw cron create "*/15 * * * *" \
  --name "Kuyruk derinliği yoklaması" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>`, `argv: ["sh", "-lc", <shell>]` depolar. Tam argv yürütmesi için `--command-argv '["node","scripts/report.mjs"]'` kullanın. Komut işleri stdout/stderr çıktısını yakalar, normal Cron geçmişini kaydeder ve çıktıyı yalıtılmış işlerle aynı `announce`, `webhook` veya `none` teslimat modları üzerinden yönlendirir. Yalnızca `NO_REPLY` yazdıran bir komutun çıktısı gösterilmez.

## Oturumlar

`--session`; `main`, `isolated`, `current` veya `session:<id>` kabul eder.

<AccordionGroup>
  <Accordion title="Oturum anahtarları">
    - `main`, ajanın ana oturumuna bağlanır.
    - `isolated`, her çalıştırma için yeni bir transkript ve oturum kimliği oluşturur.
    - `current`, oluşturulma anındaki etkin oturuma bağlanır.
    - `session:<id>`, açıkça belirtilmiş kalıcı bir oturum anahtarına sabitlenir.

  </Accordion>
  <Accordion title="Yalıtılmış oturum semantiği">
    Yalıtılmış çalıştırmalar, ortam konuşma bağlamını sıfırlar. Kanal ve grup yönlendirmesi, gönderme/kuyruğa alma politikası, yetki yükseltme, kaynak ve ACP çalışma zamanı bağlantısı yeni çalıştırma için sıfırlanır. Güvenli tercihler ile kullanıcı tarafından açıkça seçilen model veya kimlik doğrulama geçersiz kılmaları çalıştırmalar arasında taşınabilir.
  </Accordion>
</AccordionGroup>

## Teslimat

`openclaw cron list` ve `openclaw cron show <job-id>`, çözümlenen teslimat rotasının önizlemesini gösterir. `channel: "last"` için önizleme, rotanın ana veya geçerli oturumdan çözümlenip çözümlenmediğini ya da güvenli biçimde başarısız olup olmayacağını gösterir.

Sağlayıcı önekli hedefler, çözümlenmemiş duyuru kanallarındaki belirsizliği giderebilir. Örneğin `to: "telegram:123"`, `delivery.channel` belirtilmediğinde veya `last` olduğunda Telegram'ı seçer. Yalnızca yüklenen Plugin tarafından bildirilen önekler sağlayıcı seçicisidir. `delivery.channel` açıkça belirtilmişse önek bu kanalla eşleşmelidir; `to: "telegram:123"` ile `channel: "whatsapp"` reddedilir. `imessage:` ve `sms:` gibi hizmet önekleri, kanala ait hedef söz dizimi olarak kalır.

<Note>
Yalıtılmış `cron add` işleri varsayılan olarak `--announce` teslimatını kullanır. Çıktıyı dahili tutmak için `--no-deliver` kullanın. `--deliver`, `--announce` için kullanımdan kaldırılmış bir diğer ad olarak kalır.
</Note>

### Teslimat sahipliği

Yalıtılmış Cron sohbet teslimatı, ajan ve çalıştırıcı arasında paylaşılır:

- Bir sohbet rotası kullanılabilir olduğunda ajan, `message` aracını kullanarak doğrudan gönderebilir.
- `announce`, yalnızca ajan çözümlenen hedefe doğrudan göndermediyse son yanıtı yedek yöntemle teslim eder.
- `webhook`, tamamlanan yükü bir URL'ye gönderir.
- `none`, çalıştırıcının yedek teslimatını devre dışı bırakır.

Webhook teslimatını ayarlamak için `cron add|create --webhook <url>` veya `cron edit <job-id> --webhook <url>` kullanın. `--webhook` ile `--announce`, `--no-deliver`, `--channel`, `--to`, `--thread-id` veya `--account` gibi sohbet teslimatı bayraklarını birlikte kullanmayın.

`cron edit <job-id>`, `--clear-channel`, `--clear-to`, `--clear-thread-id` ve `--clear-account` ile ayrı teslimat yönlendirme alanlarının ayarını kaldırabilir (her biri, karşılık gelen ayarlama bayrağıyla birlikte kullanıldığında reddedilir). Yalnızca çalıştırıcının yedek teslimatını devre dışı bırakan `--no-deliver` seçeneğinin aksine bunlar, depolanan alanı kaldırarak işin rotasının ilgili bölümünü yeniden varsayılanlardan çözümlemesini sağlar.

`--announce`, son yanıt için çalıştırıcının yedek teslimatıdır. `--no-deliver` bu yedek yöntemi devre dışı bırakır ancak bir sohbet rotası kullanılabilir olduğunda ajanın `message` aracını kaldırmaz.

Etkin bir sohbetten oluşturulan hatırlatıcılar, yedek duyuru teslimatı için canlı sohbet teslimat hedefini korur. Dahili oturum anahtarları küçük harfli olabilir; bunları Matrix oda kimlikleri gibi büyük/küçük harfe duyarlı sağlayıcı kimlikleri için doğruluk kaynağı olarak kullanmayın.

### Hata teslimatı

Hata bildirimleri şu sırayla çözümlenir:

1. İşteki `delivery.failureDestination`.
2. Genel `cron.failureDestination`.
3. İşin birincil duyuru hedefi (yukarıdakilerin hiçbiri somut bir hedefe çözümlenmediğinde).

<Note>
Ana oturum işleri, yalnızca birincil teslimat modu `webhook` olduğunda `delivery.failureDestination` kullanabilir. Yalıtılmış işler bunu tüm modlarda kabul eder.
</Note>

Yalıtılmış Cron çalıştırmaları, hiçbir yanıt yükü üretilmediğinde bile çalıştırma düzeyindeki ajan hatalarını iş hataları olarak değerlendirir; böylece model/sağlayıcı hataları yine hata sayaçlarını artırır ve hata bildirimlerini tetikler.

Komut Cron işleri yalıtılmış bir ajan turu başlatmaz. Sıfır çıkış kodu `ok` kaydeder; sıfır olmayan çıkış, sinyal, zaman aşımı veya çıktı vermeme zaman aşımı `error` kaydeder ve aynı hata bildirimi yolunu tetikleyebilir.

Yalıtılmış bir çalıştırma ilk model isteğinden önce zaman aşımına uğrarsa `openclaw cron show` ve `openclaw cron runs`, `setup timed out before runner start` gibi aşamaya özgü bir hata veya bilinen son başlatma aşamasını belirten bir takılma iletisi (örneğin `context-engine`) içerir. CLI tabanlı sağlayıcılarda model öncesi gözetleyici, harici CLI turu başlayana kadar etkin kalır; böylece oturum arama, kanca, kimlik doğrulama, istem ve CLI kurulumu takılmaları model öncesi Cron hataları olarak bildirilir.

## Zamanlama

### Tek seferlik işler

`--at <datetime>`, tek seferlik bir çalıştırma zamanlar. UTC farkı içermeyen tarih-saatler, duvar saati zamanını belirtilen saat diliminde yorumlayan `--tz <iana>` seçeneğini de iletmediğiniz sürece UTC olarak değerlendirilir.

<Note>
Tek seferlik işler, başarılı olduktan sonra varsayılan olarak silinir. Bunları korumak için `--keep-after-run` kullanın.
</Note>

### Yinelenen işler

Yinelenen işler, art arda oluşan hataların ardından üstel yeniden deneme geri çekilmesi kullanır: 30s, 1m, 5m, 15m, 60m. Bir sonraki başarılı çalıştırmanın ardından zamanlama normale döner.

Atlanan çalıştırmalar, yürütme hatalarından ayrı izlenir. Yeniden deneme geri çekilmesini etkilemezler ancak `openclaw cron edit <job-id> --failure-alert-include-skipped`, hata uyarılarının yinelenen atlanmış çalıştırma bildirimlerini de kapsamasını sağlayabilir.

Yerel olarak yapılandırılmış bir model sağlayıcısını (geri döngüde, özel bir ağda veya `.local` üzerinde temel URL) hedefleyen yalıtılmış işlerde Cron, ajan turunu başlatmadan önce hafif bir sağlayıcı ön denetimi çalıştırır: `api: "ollama"` sağlayıcıları `/api/tags` adresinde; diğer yerel OpenAI uyumlu sağlayıcılar (`api: "openai-completions"`, ör. vLLM, SGLang, LM Studio) ise `/models` adresinde yoklanır. Uç noktaya erişilemiyorsa çalıştırma `skipped` olarak kaydedilir ve sonraki bir zamanlamada yeniden denenir; aynı yerel sunucuyu kullanan çok sayıda işin tekrarlanan yoklamalarla sunucuya yüklenmemesi için erişilebilirlik sonucu uç nokta başına 5 dakika önbelleğe alınır.

Cron işleri, bekleyen çalışma zamanı durumu ve çalıştırma geçmişi, paylaşılan SQLite durum veritabanında bulunur. Eski `jobs.json`, `<name>-state.json` ve `runs/*.jsonl` dosyaları bir kez içe aktarılır ve `.migrated` son ekiyle yeniden adlandırılır. İçe aktarmanın ardından zamanlamaları JSON dosyalarını düzenlemek yerine `openclaw cron add|edit|remove` ile düzenleyin.

### Manuel çalıştırmalar

`openclaw cron run <job-id>`, varsayılan olarak zorla çalıştırır ve manuel çalıştırma kuyruğa alınır alınmaz döner. Başarılı yanıtlar `{ ok: true, enqueued: true, runId }` içerir. Sonraki sonucu incelemek için döndürülen `runId` değerini kullanın:

```bash
openclaw cron run <job-id>
openclaw cron runs --id <job-id> --run-id <run-id>
```

Bir betiğin, tam olarak o kuyruğa alınmış çalıştırma sonlandırıcı bir durum kaydedene kadar beklemesi gerektiğinde `--wait` ekleyin:

```bash
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
```

`--wait` kullanıldığında CLI yine önce `cron.run` çağrısını yapar, ardından döndürülen `runId` için `cron.runs` yoklaması yapar. Komut yalnızca çalıştırma `ok` durumuyla tamamlandığında `0` koduyla çıkar. Çalıştırma `error` veya `skipped` ile tamamlandığında, Gateway yanıtı `runId` içermediğinde ya da `--wait-timeout` süresi dolduğunda (varsayılan `10m`, varsayılan olarak her `2s` aralığında yoklanır) sıfır olmayan kodla çıkar. `--poll-interval` sıfırdan büyük olmalıdır.

<Note>
Manuel komutun yalnızca işin zamanı gelmişse çalışmasını istediğinizde `--due` kullanın. `--due --wait` bir çalıştırmayı kuyruğa almazsa komut yoklama yapmak yerine normal çalıştırmama yanıtını döndürür.
</Note>

## Modeller

`cron add|edit --model <ref>`, iş için izin verilen bir model seçer. `cron add|edit --fallbacks <list>`, örneğin `--fallbacks openrouter/gpt-4.1-mini,openai/gpt-5`, iş başına yedek modelleri ayarlar; yedeksiz katı bir çalıştırma için `--fallbacks ""` iletin. `cron edit <job-id> --clear-fallbacks`, iş başına yedek model geçersiz kılmasını kaldırır. `cron edit <job-id> --clear-model`, iş başına model geçersiz kılmasını kaldırarak işin normal Cron model seçimi önceliğini izlemesini sağlar (varsa depolanmış bir Cron oturumu geçersiz kılması, aksi hâlde ajan/varsayılan model); `--model` ile birlikte kullanılamaz. `cron add|edit --thinking <level>`, iş başına düşünme geçersiz kılması ayarlar; `cron edit <job-id> --clear-thinking` bunu kaldırarak işin normal Cron düşünme önceliğini izlemesini sağlar ve `--thinking` ile birlikte kullanılamaz.

<Warning>
Model izinli değilse veya çözümlenemiyorsa Cron, işin ajanına ya da varsayılan model seçimine geri dönmek yerine çalıştırmayı açık bir doğrulama hatasıyla başarısız kılar.
</Warning>

Cron `--model`, sohbet oturumu `/model` geçersiz kılması değil, **işin birincil modelidir**. Bunun anlamı şudur:

- Seçilen iş modeli başarısız olduğunda yapılandırılmış model yedekleri yine uygulanır.
- İş başına `fallbacks` yükü, mevcut olduğunda yapılandırılmış yedek listesinin yerini alır.
- İş yükünde/API'de boş bir iş başına yedek listesi (`--fallbacks ""` veya `fallbacks: []`), Cron çalıştırmasını katı hâle getirir.
- Bir işte `--model` bulunduğunda ancak hiçbir yedek listesi yapılandırılmadığında OpenClaw, ajanın birincil modelinin gizli bir yeniden deneme hedefi olarak eklenmemesi için açıkça boş bir yedek geçersiz kılması iletir.
- Yerel sağlayıcı ön denetimleri, bir Cron çalıştırmasını `skipped` olarak işaretlemeden önce yapılandırılmış yedekleri sırayla denetler.

`openclaw doctor`, sağlayıcı ad alanı sayıları ve `agents.defaults.model` ile uyuşmazlıklar dâhil olmak üzere `payload.model` değeri önceden ayarlanmış işleri bildirir. Kimlik doğrulama, sağlayıcı veya faturalandırma davranışı canlı sohbet ile zamanlanmış işler arasında farklı göründüğünde bu denetimi kullanın.

### Yalıtılmış Cron model önceliği

Yalıtılmış Cron, etkin modeli şu sırayla çözümler:

1. Gmail kancası geçersiz kılması.
2. İş başına `--model`.
3. Depolanmış Cron oturumu model geçersiz kılması (kullanıcı bir model seçtiğinde).
4. Ajan veya varsayılan model seçimi.

### Hızlı mod

Yalıtılmış Cron hızlı modu, çözümlenen canlı model seçimini izler. Model yapılandırması `params.fastMode` varsayılan olarak uygulanır, ancak depolanmış bir oturumun `fastMode` geçersiz kılması yine de yapılandırmaya üstün gelir. Çözümlenen mod `auto` olduğunda kesme süresi, seçilen modelin `params.fastAutoOnSeconds` değerini kullanır ve varsayılan olarak 60 saniyedir.

### Canlı model değiştirme yeniden denemeleri

Yalıtılmış bir çalıştırma `LiveSessionModelSwitchError` hatası oluşturursa Cron, yeniden denemeden önce etkin çalıştırma için geçiş yapılan sağlayıcıyı ve modeli (varsa geçiş yapılan kimlik doğrulama profili geçersiz kılmasını da) kalıcı olarak kaydeder. Dış yeniden deneme döngüsü, ilk denemeden sonra iki geçiş yeniden denemesiyle sınırlıdır; ardından sonsuza kadar döngüye girmek yerine işlemi iptal eder.

## Çalıştırma çıktısı ve retler

### Eski alındı yanıtlarını bastırma

Yalıtılmış Cron dönüşleri, yalnızca eski alındı bildirimlerinden oluşan yanıtları bastırır. İlk sonuç yalnızca geçici bir durum güncellemesiyse ve nihai yanıttan hiçbir alt alt-agent çalıştırması sorumlu değilse Cron, teslimattan önce gerçek sonucu almak için bir kez daha istem gönderir.

### Sessiz token bastırma

Yalıtılmış bir Cron çalıştırması yalnızca sessiz tokenı (`NO_REPLY` veya `no_reply`) döndürürse Cron hem doğrudan giden teslimatı hem de yedek kuyruklanmış özet yolunu bastırır; böylece sohbete hiçbir şey gönderilmez.

### Yapılandırılmış retler

Yalıtılmış Cron çalıştırmaları, yerleşik çalıştırmadan gelen yapılandırılmış yürütme reddi meta verilerini (`SYSTEM_RUN_DENIED` veya `INVALID_REQUEST` kodlu kritik yürütme aracı hataları) yetkili ret sinyali olarak kullanır. Ayrıca bu kodlardan birini taşıyan iç içe yapılandırılmış bir hatayı sarmalayan Node ana makinesi `UNAVAILABLE` sarmalayıcılarını da dikkate alırlar.

Yerleşik çalıştırma ayrıca yapılandırılmış ret meta verileri sağlamadığı sürece Cron, nihai çıktı metnini veya onay görünümündeki ret ifadelerini ret olarak sınıflandırmaz; dolayısıyla sıradan asistan metni engellenmiş bir komut olarak değerlendirilmez.

`cron list` ve çalıştırma geçmişi, engellenmiş bir komutu `ok` olarak bildirmek yerine ret nedenini gösterir.

## Saklama

Saklama davranışı:

- `cron.sessionRetention` (varsayılan `24h`; devre dışı bırakmak için `false`) tamamlanmış yalıtılmış çalıştırma oturumlarını temizler.
- Çalıştırma geçmişi, Cron işi başına en yeni 2000 terminal satırını tutar. Kayıp satırlar, standart 24 saatlik kayıp görev temizleme aralığını korur.

## Eski işleri taşıma

<Note>
Geçerli teslimat ve depolama biçiminden önce oluşturulmuş Cron işleriniz varsa `openclaw doctor --fix` komutunu çalıştırın. Doctor, eski Cron alanlarını (`jobId`, `schedule.cron`, eski `threadId` dâhil üst düzey teslimat alanları, yük `provider` teslimat takma adları) normalleştirir ve bu yapılandırma anahtarını kaldırmadan önce `notify: true` Webhook yedek işlerini kullanımdan kaldırılmış ham `cron.webhook` değerinden açık Webhook teslimatına taşır. Zaten bir sohbete duyuru yapan işler bu teslimatı korur ve bir tamamlanma Webhook hedefi edinir. Eski bir Webhook yoksa taşıma hedefi bulunmayan işlerden işlevsiz üst düzey `notify` işareti kaldırılır (mevcut teslimat değiştirilmeden korunur); böylece `doctor --fix` artık bunlar hakkında tekrar tekrar uyarı vermez.
</Note>

## Yaygın düzenlemeler

Mesajı değiştirmeden teslimat ayarlarını güncelleyin:

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "123456789"
```

Yalıtılmış bir iş için teslimatı devre dışı bırakın:

```bash
openclaw cron edit <job-id> --no-deliver
```

Yalıtılmış bir iş için hafif başlangıç bağlamını etkinleştirin:

```bash
openclaw cron edit <job-id> --light-context
```

Belirli bir kanala duyuru yapın:

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
```

Bir Telegram forum konusuna duyuru yapın:

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "-1001234567890" --thread-id 42
```

Hafif başlangıç bağlamına sahip yalıtılmış bir iş oluşturun:

```bash
openclaw cron create "0 7 * * *" \
  "Gece boyunca gerçekleşen güncellemeleri özetle." \
  --name "Hafif sabah özeti" \
  --session isolated \
  --light-context \
  --no-deliver
```

`--light-context` yalnızca yalıtılmış agent dönüşü işlerine uygulanır. Cron çalıştırmalarında hafif mod, tam çalışma alanı başlangıç kümesini eklemek yerine başlangıç bağlamını boş tutar.

Tam argv, cwd, env, stdin ve çıktı sınırlarına sahip bir komut işi oluşturun:

```bash
openclaw cron create "*/30 * * * *" \
  --name "Pozisyon dışa aktarımı" \
  --command-argv '["node","scripts/export-position.mjs"]' \
  --command-cwd "/srv/app" \
  --command-env "NODE_ENV=production" \
  --command-input '{"mode":"summary"}' \
  --timeout-seconds 120 \
  --no-output-timeout-seconds 30 \
  --output-max-bytes 65536 \
  --webhook "https://example.invalid/openclaw/cron"
```

## Yaygın yönetim komutları

Elle çalıştırma ve inceleme:

```bash
openclaw cron list
openclaw cron list --agent ops
openclaw cron get <job-id>
openclaw cron show <job-id>
openclaw cron run <job-id>
openclaw cron run <job-id> --due
openclaw cron run <job-id> --wait --wait-timeout 10m
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
openclaw cron runs --id <job-id> --limit 50
openclaw cron runs --id <job-id> --run-id <run-id>
```

`openclaw cron list` varsayılan olarak etkin işleri gösterir. Devre dışı bırakılmış işleri dâhil etmek için `--all`, yalnızca etkin normalleştirilmiş agent kimliği eşleşen işleri göstermek için `--agent <id>` iletin; depolanmış bir agent kimliği bulunmayan işler, yapılandırılmış varsayılan agent'a ait sayılır.

`openclaw cron get <job-id>` depolanmış iş JSON'unu doğrudan döndürür. Teslimat rotası önizlemesini içeren, insanlar tarafından okunabilir görünümü istediğinizde `cron show <job-id>` kullanın.

`cron list --json` ve `cron show <job-id> --json`, her işte `enabled`, `state.runningAtMs` ve `state.lastRunStatus` değerlerinden hesaplanan üst düzey bir `status` alanı içerir. Değerler: `disabled`, `running`, `ok`, `error`, `skipped` veya `idle`. Harici araçların iş durumunu yeniden türetmeden okuyabilmesi için JSON durumu kurallı ve süslemesiz kalır; insanlar tarafından okunabilir çıktı, yinelenen `error` durumlarını hata sayısıyla süsleyebilir.

`cron runs` girdileri; amaçlanan Cron hedefi, çözümlenen hedef, mesaj aracı gönderimleri, yedek kullanım ve teslim durumu hakkındaki teslimat tanılamalarını içerir.

İş başına özel karalama alanı (Heartbeat kontrol listeleri ve benzer izleme bağlamı):

```bash
openclaw cron scratch <job-id>                  # geçerli karalama alanı içeriğini yazdır
openclaw cron scratch <job-id> --json           # karalama alanı ve revizyon meta verileri
openclaw cron scratch <job-id> --set "text"     # karalama alanını tam metinle değiştir
openclaw cron scratch <job-id> --file notes.md  # karalama alanını bir dosyadan değiştir (stdin için -)
openclaw cron scratch <job-id> --unset          # karalama alanı satırını kaldır
```

Karalama alanı paylaşılan durum veritabanında depolanır, 256 KiB ile sınırlıdır ve hiçbir zaman `cron list`/`cron get`/`cron runs` çıktısına dâhil edilmez. Yazmalar, komut başlangıcında okunan revizyona karşı karşılaştır ve değiştir yöntemiyle korunur; bunun yerine açık bir revizyonu sabitlemek için `--expected-revision <n>` iletin. Heartbeat izleyicilerinin karalama alanını nasıl kullandığı için [Heartbeat](/tr/gateway/heartbeat#monitor-scratch-optional) bölümüne bakın.

Agent ve oturum hedefini değiştirme:

```bash
openclaw cron edit <job-id> --agent ops
openclaw cron edit <job-id> --clear-agent
openclaw cron edit <job-id> --session current
openclaw cron edit <job-id> --session "session:daily-brief"
```

Agent dönüşü işlerinde `--agent` atlandığında `openclaw cron add` uyarı verir ve varsayılan agent'a (`main`) geri döner. Belirli bir agent'ı sabitlemek için oluşturma sırasında `--agent <id>` iletin.

Teslimat ayarlamaları:

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
openclaw cron edit <job-id> --webhook "https://example.invalid/openclaw/cron"
openclaw cron edit <job-id> --best-effort-deliver
openclaw cron edit <job-id> --no-best-effort-deliver
openclaw cron edit <job-id> --no-deliver
```

## İlgili

- [CLI referansı](/tr/cli)
- [Zamanlanmış görevler](/tr/automation/cron-jobs)
