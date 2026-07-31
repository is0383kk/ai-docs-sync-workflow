---
read_when:
    - Güvenli ikili dosyaları veya özel güvenli ikili dosya profillerini yapılandırma
    - Onayları Slack/Discord/Telegram veya diğer sohbet kanallarına yönlendirme
    - Bir kanal için yerel onay istemcisi uygulama
summary: 'Gelişmiş exec onayları: güvenli ikili dosyalar, yorumlayıcı bağlama, onay iletme, yerel teslimat'
title: Çalıştırma onayları — gelişmiş
x-i18n:
    generated_at: "2026-07-26T23:03:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac90d41f867a8ae4f14b6c9c13f3732d102a65707f456623932b858145a9bf46
    source_path: tools/exec-approvals-advanced.md
    workflow: 16
---

Gelişmiş exec onayı konuları: `safeBins` hızlı yolu, yorumlayıcı/çalışma zamanı
bağlama ve onayların sohbet kanallarına iletilmesi (yerel teslimat dahil).
Temel ilke ve onay akışı için [Exec onayları](/tr/tools/exec-approvals) bölümüne bakın.

## Güvenli ikili dosyalar (yalnızca stdin)

`tools.exec.safeBins`, izin verilenler listesi modunda açık izin verilenler listesi girdileri
**olmadan** çalışan **yalnızca stdin** ikili dosyalarını (örneğin `cut`) belirtir.
Güvenli ikili dosyalar konumsal dosya bağımsız değişkenlerini ve yol benzeri belirteçleri
reddettiğinden yalnızca gelen akış üzerinde çalışabilir. Bunu genel bir güven listesi olarak
değil, akış filtreleri için dar kapsamlı bir hızlı yol olarak değerlendirin.

<Warning>
Yorumlayıcı veya çalışma zamanı ikili dosyalarını (örneğin `python3`, `node`,
`ruby`, `bash`, `sh`, `zsh`) `safeBins` içine **eklemeyin**. Bir komut tasarımı
gereği kod değerlendirebiliyor, alt komutlar çalıştırabiliyor veya dosyaları okuyabiliyorsa
açık izin verilenler listesi girdilerini tercih edin ve onay istemlerini etkin tutun.
Özel güvenli ikili dosyalar `tools.exec.safeBinProfiles.<bin>` içinde açık bir profil tanımlamalıdır.
</Warning>

Varsayılan güvenli ikili dosyalar:

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`, `uniq`, `head`, `tail`, `tr`, `wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` ve `sort` varsayılan listede değildir. Bunları etkinleştirirseniz stdin dışındaki
iş akışları için açık izin verilenler listesi girdilerini koruyun. Güvenli ikili dosya modunda
`grep` için kalıbı `-e`/`--regexp` ile sağlayın; dosya işlenenlerinin belirsiz konumsal
değerler olarak gizlice geçirilememesi için konumsal kalıp biçimi reddedilir.

### Argv doğrulaması ve reddedilen bayraklar

Doğrulama yalnızca argv biçimine göre belirlenimlidir (ana bilgisayar dosya sisteminde
varlık denetimi yapılmaz); bu, izin verme/reddetme farklılıklarından kaynaklanan dosya
varlığı kâhini davranışını önler. Varsayılan güvenli ikili dosyalarda dosyaya yönelik
seçenekler reddedilir; uzun seçenekler kapalı başarısız olacak şekilde doğrulanır
(bilinmeyen bayraklar ve belirsiz kısaltmalar reddedilir). Varsayılan ikili dosyaların
tanınan salt okunur Boole bayrakları (örneğin `wc -l`, `tr -d`, `uniq -c`) kabul edilirken
tanınmayan kısa bayraklar kapalı başarısız olarak kalır ve manuel onaya yönlendirilir.

Güvenli ikili dosya profiline göre reddedilen bayraklar:

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`: `--dereference-recursive`, `--directories`, `--exclude-from`, `--file`, `--recursive`, `-R`, `-d`, `-f`, `-r`
- `jq`: `--argfile`, `--from-file`, `--library-path`, `--rawfile`, `--slurpfile`, `-L`, `-f`
- `sort`: `--compress-program`, `--files0-from`, `--output`, `--random-source`, `--temporary-directory`, `-T`, `-o`
- `tail`: `--follow`, `--retry`, `-F`, `-f`
- `wc`: `--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

Güvenli ikili dosyalar ayrıca yalnızca stdin bölümlerinde argv belirteçlerinin çalışma
zamanında **değişmez metin** olarak ele alınmasını zorunlu kılar (glob eşleştirmesi ve
`$VARS` genişletmesi yoktur); böylece `*` veya `$HOME/...` gibi kalıplar dosya okumalarını
gizlice geçirmek için kullanılamaz. Anlamları yalnızca stdin ile sınırlandırılacak şekilde
doğrulanamadığından `awk`, `sed` ve `jq` güvenli ikili dosya olarak her zaman reddedilir:
`jq` ortam verilerini okuyabilir ve modüllerden veya başlangıç dosyalarından jq kodu
yükleyebilir. Bu araçlar için `safeBins` yerine açık bir izin verilenler listesi girdisi
veya onay istemi kullanın.

### Güvenilir ikili dosya dizinleri

Güvenli ikili dosyalar, güvenilir ikili dosya dizinlerinden (sistem varsayılanları ve
isteğe bağlı `tools.exec.safeBinTrustedDirs`) çözümlenmelidir. `PATH` girdilerine hiçbir zaman otomatik olarak
güvenilmez. Varsayılan güvenilir dizinler kasıtlı olarak en aza indirilmiştir:
`/bin`, `/usr/bin`. Güvenli ikili dosyanız paket yöneticisi/kullanıcı yollarında (örneğin
`/opt/homebrew/bin`, `/usr/local/bin`, `/opt/local/bin`, `/snap/bin`) bulunuyorsa bunları
`tools.exec.safeBinTrustedDirs` içine açıkça ekleyin.

### Kabuk zincirleme, sarmalayıcılar ve çoklayıcılar

Her üst düzey bölüm izin verilenler listesini (güvenli ikili dosyalar veya Skills
otomatik izni dahil) karşılıyorsa kabuk zincirlemeye (`&&`, `||`, `;`) izin verilir.
Yönlendirmeler, izin verilenler listesi modunda desteklenmez. Komut ikamesi
(`$()` / ters tırnaklar), çift tırnakların içi dahil olmak üzere izin verilenler listesi
ayrıştırması sırasında reddedilir; değişmez `$()` metnine ihtiyacınız varsa tek tırnak
kullanın.

macOS yardımcı uygulama onaylarında, kabuk denetimi veya genişletme söz dizimi
(`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) içeren ham kabuk metni,
kabuk ikili dosyasının kendisi izin verilenler listesinde olmadığı sürece izin verilenler
listesi eşleşmemesi olarak değerlendirilir.

Kabuk sarmalayıcıları (`bash|sh|zsh ... -c/-lc`) için istek kapsamındaki ortam geçersiz kılmaları küçük
ve açık bir izin verilenler listesine (`TERM`, `LANG`, `LC_*`, `COLORTERM`,
`NO_COLOR`, `FORCE_COLOR`) indirgenir.

İzin verilenler listesi modundaki `allow-always` kararları için şeffaf dağıtım sarmalayıcıları
(örneğin `env`, `flock`, `nice`, `nohup`, `stdbuf`, `timeout`), sarmalayıcı yolu yerine
iç yürütülebilir dosya yolunu kalıcılaştırır. Kabuk çoklayıcıları (`busybox`, `toybox`) da
kabuk uygulamacıkları (`sh`, `ash` vb.) için aynı şekilde açılır. Bir sarmalayıcı
veya çoklayıcı güvenli biçimde açılamazsa hiçbir izin verilenler listesi girdisi otomatik
olarak kalıcılaştırılmaz.

`python3` veya `node` gibi yorumlayıcıları izin verilenler listesine eklerseniz satır içi
değerlendirmenin yine de açık onay gerektirmesi için `tools.exec.strictInlineEval=true` seçeneğini tercih edin.
Katı modda `allow-always`, zararsız yorumlayıcı/betik çağrılarını yine kalıcılaştırabilir;
ancak satır içi değerlendirme taşıyıcıları otomatik olarak kalıcılaştırılmaz.

### Güvenli ikili dosyalar ile izin verilenler listesinin karşılaştırması

| Konu             | `tools.exec.safeBins`                                      | İzin verilenler listesi (`exec-approvals.json`)                                              |
| ---------------- | ------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| Amaç             | Dar kapsamlı stdin filtrelerine otomatik izin vermek   | Belirli yürütülebilir dosyalara açıkça güvenmek                                    |
| Eşleşme türü     | Yürütülebilir dosya adı + güvenli ikili argv ilkesi    | Çözümlenmiş yürütülebilir dosya yolu globu veya PATH ile çağrılan komutlar için yalın komut adı globu |
| Bağımsız değişken kapsamı | Güvenli ikili dosya profili ve değişmez belirteç kurallarıyla sınırlandırılır | Varsayılan olarak yol eşleşmesi; isteğe bağlı `argPattern` ayrıştırılan argv'yi sınırlandırabilir |
| Tipik örnekler   | `head`, `tail`, `tr`, `wc`                             | `jq`, `python3`, `node`, `ffmpeg`, özel CLI'lar                                 |
| En iyi kullanım  | İşlem hatlarında düşük riskli metin dönüşümleri        | Daha geniş davranışa veya yan etkilere sahip herhangi bir araç                     |

Yapılandırma konumu:

- `safeBins`, yapılandırmadan (`tools.exec.safeBins` veya ajan başına `agents.entries.*.tools.exec.safeBins`) gelir.
- `safeBinTrustedDirs`, yapılandırmadan (`tools.exec.safeBinTrustedDirs` veya ajan başına `agents.entries.*.tools.exec.safeBinTrustedDirs`) gelir.
- `safeBinProfiles`, yapılandırmadan (`tools.exec.safeBinProfiles` veya ajan başına `agents.entries.*.tools.exec.safeBinProfiles`) gelir. Ajan başına profil anahtarları genel anahtarları geçersiz kılar.
- izin verilenler listesi girdileri, `agents.<id>.allowlist` altındaki ana bilgisayara yerel onay dosyasında (veya Control UI / `openclaw approvals allowlist ...` aracılığıyla) bulunur.
- `openclaw security audit`, yorumlayıcı/çalışma zamanı ikili dosyaları açık profiller olmadan `safeBins` içinde göründüğünde `tools.exec.safe_bins_interpreter_unprofiled` ile uyarır.
- `openclaw doctor --fix`, eksik özel `safeBinProfiles.<bin>` girdilerini `{}` olarak oluşturabilir (sonrasında inceleyip sıkılaştırın). Yorumlayıcı/çalışma zamanı ikili dosyaları otomatik olarak oluşturulmaz.

Özel profil örneği:

```json5
{
  tools: {
    exec: {
      safeBins: ["myfilter"],
      safeBinProfiles: {
        myfilter: {
          minPositional: 0,
          maxPositional: 0,
          allowedValueFlags: ["-n", "--limit"],
          deniedFlags: ["-f", "--file", "-c", "--command"],
        },
      },
    },
  },
}
```

## Yorumlayıcı/çalışma zamanı komutları

Onay destekli yorumlayıcı/çalışma zamanı çalıştırmaları kasıtlı olarak ihtiyatlıdır:

- Tam argv/cwd/env bağlamı her zaman bağlanır.
- Doğrudan kabuk betiği ve doğrudan çalışma zamanı dosyası biçimleri, mümkün olan en iyi çabayla tek bir somut yerel
  dosya anlık görüntüsüne bağlanır.
- Yine de tek bir doğrudan yerel dosyaya çözümlenen yaygın paket yöneticisi sarmalayıcı biçimleri (örneğin
  `pnpm exec`, `pnpm node`, `npm exec`, `npx`) bağlamadan önce açılır.
- OpenClaw bir yorumlayıcı/çalışma zamanı komutu için tam olarak bir somut yerel dosya belirleyemezse
  (örneğin paket betikleri, değerlendirme biçimleri, çalışma zamanına özgü yükleyici zincirleri veya belirsiz çok dosyalı
  biçimler), sahip olmadığı anlamsal kapsamı sağladığını iddia etmek yerine onay destekli yürütme
  reddedilir.
- Bu iş akışlarında korumalı alan kullanmayı, ayrı bir ana bilgisayar sınırını veya operatörün daha geniş çalışma
  zamanı anlamlarını kabul ettiği açıkça güvenilir bir izin verilenler listesi/tam iş akışını tercih edin.

Onay gerektiğinde exec aracı hemen bir onay kimliğiyle döner. Daha sonra oluşan onaylı çalıştırma
sistem olaylarını (`Exec finished` ve yapılandırıldığında `Exec running`) ilişkilendirmek için bu kimliği kullanın.
Zaman aşımından önce karar gelmezse istek, onay zaman aşımı olarak değerlendirilir ve
nihai bir ana bilgisayar komutu reddi olarak gösterilir. Kaynak oturumu bulunan ana ajan eşzamansız
onaylarında OpenClaw, komutun çalışmadığını ajanın daha sonra eksik bir sonucu düzeltmeye
çalışmak yerine gözlemlemesi için bu oturumu dahili bir takip ile sürdürür. Bekleyen exec onaylarının
süresi varsayılan olarak 30 dakika sonra dolar.

### Takip teslimatı davranışı

Onaylanmış bir eşzamansız exec tamamlandıktan sonra OpenClaw aynı oturuma bir takip `agent` dönüşü gönderir.
Reddedilen eşzamansız onaylar, ret durumu için aynı ana oturum takip yolunu kullanır; ancak
yükseltilmiş çalışma zamanı devirleri kaydetmez ve komutu çalıştırmaz. Sürdürülebilir bir ana
oturumu olmayan retler ya bastırılır ya da mevcutsa güvenli bir doğrudan yol üzerinden bildirilir.

- Geçerli bir harici teslimat hedefi varsa (teslim edilebilir kanal ve hedef `to`), takip teslimatı bu kanalı kullanır.
- Harici hedefi olmayan yalnızca web sohbeti veya dahili oturum akışlarında takip teslimatı yalnızca oturumda kalır (`deliver: false`).
- Bir çağıran, çözümlenebilir harici kanal olmadan açıkça katı harici teslimat isterse istek `INVALID_REQUEST` ile başarısız olur.
- `bestEffortDeliver` etkinse ve hiçbir harici kanal çözümlenemiyorsa teslimat başarısız olmak yerine yalnızca oturuma indirgenir.

## Üçüncü taraf istemciler için asgari kapsamlar

Gateway onay çözümlemesi, özel `operator.approvals` kapsamıyla korunur. Bu hem sahibe özgü `exec.approval.resolve` yöntemi hem de türden bağımsız `approval.resolve` yöntemi için geçerlidir; `operator.write` bunu kapsamaz. Panolar ve entegrasyonlar yalnızca kullandıkları yöntemlerin gerektirdiği kapsamları istemelidir. Onay çözümleme erişimini uzaktan yürütme düzeyinde bir yetki olarak değerlendirin ve istemci yalnızca küçük bir onay kullanıcı arayüzü sunsa bile `operator.approvals` yetkisini bilinçli olarak verin.

## Onayların sohbet kanallarına iletilmesi

Çalıştırma onayı istemlerini herhangi bir sohbet kanalına (plugin kanalları dahil) iletebilir ve
bunları `/approve` ile onaylayabilirsiniz. Bu işlem normal giden ileti teslim işlem hattını kullanır.

Yapılandırma:

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session", // "session" | "targets" | "both"
      agentFilter: ["main"],
      sessionFilter: ["discord"], // alt dize veya düzenli ifade
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" },
      ],
    },
  },
}
```

Sohbette yanıtlayın:

```
/approve <id> allow-once
/approve <id> allow-always
/approve <id> deny
```

`/approve` komutu hem çalıştırma onaylarını hem de plugin onaylarını işler. Kimlik bekleyen bir çalıştırma onayıyla eşleşmezse bunun yerine otomatik olarak plugin onaylarını denetler. Bu geri dönüş yalnızca "onay bulunamadı" hatalarıyla sınırlıdır; gerçek bir çalıştırma onayı reddi/hatası, plugin onayı olarak sessizce yeniden denenmez.

### Plugin onaylarını iletme

Plugin onaylarını iletme, çalıştırma onaylarıyla aynı teslim işlem hattını kullanır ancak
`approvals.plugin` altında kendi bağımsız yapılandırmasına sahiptir. Birini etkinleştirmek veya devre dışı bırakmak diğerini etkilemez.
Plugin geliştirme davranışı, istek alanları ve karar semantiği için
[Plugin izin istekleri](/plugins/plugin-permission-requests) bölümüne bakın.

```json5
{
  approvals: {
    plugin: {
      enabled: true,
      mode: "targets",
      agentFilter: ["main"],
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" },
      ],
    },
  },
}
```

Yapılandırma biçimi `approvals.exec` ile aynıdır: `enabled`, `mode`, `agentFilter`,
`sessionFilter` ve `targets` aynı şekilde çalışır.

Paylaşılan etkileşimli yanıtları destekleyen kanallar, hem çalıştırma hem de
plugin onayları için aynı onay düğmelerini görüntüler. Paylaşılan etkileşimli kullanıcı arayüzü bulunmayan kanallar, `/approve`
talimatlarını içeren düz metne geri döner. Plugin onay istekleri kullanılabilir kararları kısıtlayabilir: onay yüzeyleri
isteğin bildirdiği karar kümesini kullanır ve Gateway, sunulmamış bir kararı gönderme girişimlerini
reddeder.

### Herhangi bir kanalda aynı sohbetten onaylar

Bir çalıştırma veya plugin onay isteği teslim edilebilir bir sohbet yüzeyinden kaynaklandığında, aynı sohbet
varsayılan olarak bunu `/approve` ile onaylayabilir. Bu, mevcut Web kullanıcı arayüzü ve terminal kullanıcı arayüzü akışlarına ek olarak Slack, Matrix, Microsoft Teams ve
benzer teslim edilebilir sohbetler için geçerlidir ve söz konusu görüşmenin
normal kanal kimlik doğrulama modelini kullanır. Kaynak sohbet zaten komut gönderebiliyor
ve yanıt alabiliyorsa onay isteklerinin beklemede kalmak için artık ayrı bir yerel teslim bağdaştırıcısına
ihtiyacı yoktur.

Discord, Telegram ve QQ bot da aynı sohbetten `/approve` desteği sunar ancak bu kanallar, yerel onay teslimi devre dışı bırakıldığında bile
yetkilendirme için çözümlenmiş onaylayanlar listesini kullanmaya devam eder.

### Yerel onay teslimi

Bazı kanallar yerel onay istemcileri olarak da işlev görebilir: Discord, Slack, Telegram, Matrix ve QQ bot.
Yerel istemciler, paylaşılan aynı sohbetten `/approve` akışına ek olarak onaylayanlara doğrudan mesajları, kaynak sohbet dağıtımını ve kanala özgü etkileşimli onay kullanıcı deneyimini
sağlar.

Yerel onay kartları/düğmeleri kullanılabildiğinde, bu yerel kullanıcı arayüzü aracıya yönelik birincil yoldur.
Araç sonucu sohbet onaylarının kullanılamadığını veya geriye kalan tek yolun manuel onay olduğunu belirtmedikçe aracı, yinelenen bir düz sohbet `/approve` komutunu ayrıca
göstermemelidir.

Yerel onay istemcisi yapılandırılmış ancak kaynak
kanal için etkin bir yerel çalışma zamanı yoksa OpenClaw, yerel deterministik `/approve` istemini görünür tutar. Yerel çalışma zamanı
etkinse ve teslimi denediği hâlde hiçbir hedef kartı almazsa OpenClaw, isteğin yine de çözülebilmesi için tam `/approve <id> <decision>` komutunu içeren aynı sohbetten bir geri dönüş
bildirimi gönderir.

Genel model:

- ana makinenin çalıştırma ilkesi, çalıştırma onayının gerekli olup olmadığına karar vermeye devam eder
- `approvals.exec`, onay istemlerinin diğer sohbet hedeflerine iletilmesini denetler
- `channels.<channel>.execApprovals`, Discord, Slack, Telegram, QQ bot ve benzeri
  kanala özgü yerel istemcilerin etkin olup olmadığını denetler
- İstek Slack'ten geldiğinde ve Slack plugin onaylayanları çözümlendiğinde Slack plugin onayları, Slack'in yerel onay istemcisini kullanabilir;
  `approvals.plugin`, Slack çalıştırma onayları devre dışı bırakılmış olsa bile plugin onaylarını Slack
  oturumlarına veya hedeflerine yönlendirebilir
- Google Chat yerel onay kartları, kararlı `users/<id>` onaylayanları `dm.allowFrom` veya
  `defaultTo` üzerinden çözümlendiğinde Google Chat alanlarından veya ileti dizilerinden kaynaklanan çalıştırma ve plugin onaylarını
  işler; kararlar için tepki olaylarını kullanmazlar
- WhatsApp ve Signal tepkiyle onay teslimi, `approvals.exec` ve
  `approvals.plugin` tarafından denetlenir; `channels.<channel>.execApprovals` blokları yoktur

Yerel onay istemcileri, aşağıdakilerin tümü doğru olduğunda önce doğrudan mesaja teslimi otomatik olarak etkinleştirir:

- kanal yerel onay teslimini destekler
- onaylayanlar açık `execApprovals.approvers` veya `commands.ownerAllowFrom` gibi sahip
  kimliğinden çözümlenebilir
- `channels.<channel>.execApprovals.enabled` ayarlanmamıştır veya `"auto"`

Yerel onay istemcisini açıkça devre dışı bırakmak için `enabled: false` değerini ayarlayın. Onaylayanlar çözümlendiğinde istemciyi zorla
etkinleştirmek için `enabled: true` değerini ayarlayın. Herkese açık kaynak sohbet teslimi
`channels.<channel>.execApprovals.target` üzerinden açıkça etkinleştirilmeye devam eder. Yerel `target` kaynak sohbet teslimini etkinleştirdiğinde
onay istemleri komut metnini içerir.

SSS: [Sohbet onayları için neden iki çalıştırma onayı yapılandırması var?](/help/faq-first-run)

- Discord: `channels.discord.execApprovals.*`
- Slack: `channels.slack.execApprovals.*`
- Telegram: `channels.telegram.execApprovals.*`
- QQ bot: `channels.qqbot.execApprovals.*`
- Google Chat: kararlı onaylayanları `channels.googlechat.dm.allowFrom` veya
  `channels.googlechat.defaultTo` ile yapılandırın; `execApprovals` bloğu gerekli değildir
- WhatsApp: onay istemlerini WhatsApp'a yönlendirmek için `approvals.exec` ve `approvals.plugin` kullanın
- Signal: onay istemlerini Signal'e yönlendirmek için `approvals.exec` ve `approvals.plugin` kullanın

Yerel istemciye özgü yönlendirme:

- Telegram varsayılan olarak onaylayanlara doğrudan mesajları (`target: "dm"`) kullanır. Onay istemlerini kaynak Telegram sohbetinde/konusunda da göstermek için `channel` veya `both` seçeneğine geçin.
  Telegram forum konularında OpenClaw, onay istemi ve onay sonrası takip iletisi için
  konuyu korur.
- Discord ve Telegram onaylayanları açıkça belirtilebilir (`execApprovals.approvers`) veya
  `commands.ownerAllowFrom` üzerinden çıkarılabilir; yalnızca çözümlenmiş onaylayanlar onaylayabilir veya reddedebilir.
- Slack onaylayanları açıkça belirtilebilir (`execApprovals.approvers`) veya
  `commands.ownerAllowFrom` üzerinden çıkarılabilir. Slack plugin onayı doğrudan mesajları, Slack çalıştırma onaylayanlarını değil `allowFrom`
  içindeki Slack plugin onaylayanlarını ve hesap varsayılan yönlendirmesini kullanır. Slack yerel düğmeleri onay kimliği
  türünü korur; böylece `plugin:` kimlikleri ikinci bir Slack'e özgü geri dönüş katmanı olmadan plugin onaylarını çözebilir.
- Google Chat yerel kartları, ileti metnindeki manuel `/approve` geri dönüşünü korur ancak kart düğmesi
  geri çağrıları yalnızca opak eylem belirteçleri taşır; onay kimliği ve karar
  sunucu tarafındaki bekleyen durumdan alınır.
- WhatsApp emoji onayları, eşleşen üst düzey
  iletme ailesi WhatsApp'a yönlendirildiğinde hem çalıştırma hem de plugin istemlerini işler. Yerel kaynaklı istemler doğrudan bağlanır; paylaşılan hedef modundaki
  teslim, aynı türü belirlenmiş onay meta verilerini kabul edilen WhatsApp ileti makbuzuna bağlar.
- Signal tepki onayları, yalnızca eşleşen üst düzey
  iletme ailesi etkinleştirildiğinde ve Signal'e yönlendirildiğinde hem çalıştırma hem de plugin istemlerini işler. Doğrudan aynı sohbetten Signal çalıştırma onayları,
  açık onaylayanlar olmadan yerel `/approve` geri dönüşünü engelleyebilir; Signal tepki çözümlemesi
  yine de `channels.signal.allowFrom` veya `defaultTo` üzerinden açıkça belirtilmiş Signal onaylayanları gerektirir.
- Matrix yerel doğrudan mesaj/kanal yönlendirmesi ve tepki kısayolları hem çalıştırma hem de plugin onaylarını işler;
  plugin yetkilendirmesi yine `channels.matrix.dm.allowFrom` üzerinden gelir. Matrix yerel istemleri,
  ilk istem olayında `com.openclaw.approval` özel olay içeriğini barındırır; böylece OpenClaw uyumlu
  Matrix istemcileri yapılandırılmış onay durumunu okuyabilirken standart istemciler düz metin
  `/approve` geri dönüşünü korur.
- Yerel Discord ve Telegram onay düğmeleri, taşıma katmanına özel geri çağrı verilerinde açık bir çalıştırma veya plugin sahibi türü taşır ve
  yalnızca o sahibi çözümler. Tür içermeyen eski `/approve` denetimleri
  sınırlı bir uyumluluk yolu olarak kalır: yalnızca aktörün onaylayabileceği sahip türlerini dener,
  yalnızca onay bulunamadı sonucundan sonra devam eder ve hiçbir zaman onay kimliğinden sahipliği çıkarmaz.
- İstekte bulunan kişinin onaylayan olması gerekmez.
- Hiçbir operatör kullanıcı arayüzü veya yapılandırılmış onay istemcisi isteği kabul edemiyorsa istem,
  `askFallback` değerine geri döner.

`/diagnostics` ve `/export-trajectory` gibi yalnızca sahiplere açık hassas grup komutları,
onay istemleri ve nihai sonuçlar için özel sahip yönlendirmesini kullanır. OpenClaw önce, sahibin komutu çalıştırdığı
aynı yüzeyde özel bir yol dener. Bu yüzeyde özel sahip yolu yoksa
`commands.ownerAllowFrom` içindeki ilk kullanılabilir sahip yoluna geri döner; böylece Telegram yapılandırılmış
birincil özel arayüz olduğunda Discord grup komutu onayı ve sonucu yine de sahibin Telegram doğrudan mesajına
gönderebilir. Grup sohbeti yalnızca kısa bir alındı bildirimi alır.

Ayrıca bakınız:

- [Discord](/channels/discord)
- [Telegram](/channels/telegram)
- [QQ bot](/channels/qqbot)

### Resmî mobil operatör uygulamaları

Resmî iOS ve Android uygulamaları, bir `operator.admin` bağlantısı kullanıldığında veya eşleştirilmiş
`operator.approvals` cihazları istek tarafından açıkça hedeflendiğinde Gateway'in sahip olduğu bekleyen çalıştırma
onaylarını da inceleyebilir. Control UI tarafından kullanılan
aynı temizlenmiş kalıcı kaydı okur, türü dikkate alan bir karar gönderir ve Gateway'in standart
ilk yanıt sonucunu görüntülerler. Apple Watch, bir kez izin verme ve reddetme eylemleriyle
bu onay istemlerini eşleştirilmiş iPhone üzerinden yansıtır. Doğrudan Watch Gateway modu
onayları incelemez.

Kaybolan bir çözümleme alındı bildirimi, gönderilen seçimi yetkili hâle getirmez:
uygulama denetimleri devre dışı bırakır ve kaydı yeniden okur. Başka bir yüzey
kazandıysa uygulama bu kaydedilen kararı gösterir. Bekleyen istemler, onları
oluşturan Gateway'e bağlı kalır; dolayısıyla etkin Gateway'i değiştirmek eski bir
onay kimliğini yeniden yönlendiremez.

### macOS IPC akışı

```
Gateway -> Node Hizmeti (WS)
                 |  IPC (UDS + belirteç + HMAC + TTL)
                 v
             Mac Uygulaması (kullanıcı arayüzü + onaylar + system.run)
```

Güvenlik notları:

- Unix yuvası modu `0600`, belirteç `exec-approvals.json` içinde saklanır.
- Aynı UID'ye sahip eş denetimi.
- Sınama/yanıt (tek kullanımlık değer + HMAC belirteci + istek karması) + kısa TTL.

## SSS

### Bir onay hedefinde `accountId` ve `threadId` ne zaman kullanılır?

Kanalda birden fazla yapılandırılmış kimlik varsa ve onay isteminin belirli bir hesaptan
gönderilmesi gerekiyorsa `accountId` kullanın. Hedef konuları veya
ileti dizilerini destekliyorsa ve istemin üst düzey sohbet yerine ilgili ileti dizisi içinde kalması gerekiyorsa `threadId` kullanın.

Somut bir Telegram örneği, forum konularına ve iki Telegram bot
hesabına sahip bir operasyon süper grubudur. `to` değeri süper grubu adlandırır, `accountId` bot hesabını seçer ve `threadId`
forum konusunu seçer:

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "targets",
      targets: [
        {
          channel: "telegram",
          to: "-1001234567890",
          accountId: "ops-bot",
          threadId: "77",
        },
      ],
    },
  },
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "env:TELEGRAM_PRIMARY_BOT_TOKEN",
        },
        "ops-bot": {
          name: "Operations bot",
          botToken: "env:TELEGRAM_OPS_BOT_TOKEN",
        },
      },
    },
  },
}
```

Bu yapılandırmayla, yönlendirilen yürütme onayları `ops-bot` Telegram hesabı tarafından `-1001234567890` sohbetinin
`77` konusuna gönderilir. `accountId` içermeyen bir hedef, kanalın varsayılan hesabını kullanır ve
`threadId` içermeyen bir hedef, üst düzey hedefe gönderilir.

### Onaylar bir oturuma gönderildiğinde, o oturumdaki herkes bunları onaylayabilir mi?

Hayır. Oturuma teslim, yalnızca istemin nerede görüneceğini belirler. Tek başına o sohbetteki her
katılımcıya onaylama yetkisi vermez.

Genel aynı sohbet `/approve` için gönderenin, söz konusu kanal oturumunda komutlar için zaten yetkilendirilmiş olması
gerekir. Kanal açık onay yetkilileri sunuyorsa bu yetkililer, söz konusu oturumda başka türlü komut yetkisine sahip olmasalar
bile `/approve` eylemini yetkilendirebilir.

Bazı kanallar daha katıdır. Discord, Telegram, Matrix, Slack yerel onay DM'leri ve benzeri
yerel onay istemcileri, onay yetkilendirmesi için çözümlenmiş onaylayan listelerini kullanır. Örneğin,
bir Telegram forum konusu onay istemi konudaki herkes tarafından görülebilir, ancak yalnızca
`channels.telegram.execApprovals.approvers` veya `commands.ownerAllowFrom` üzerinden çözümlenen sayısal
Telegram kullanıcı kimlikleri bunu onaylayabilir veya reddedebilir.

## İlgili

- [Yürütme onayları](/tr/tools/exec-approvals) — temel politika ve onay akışı
- [Yürütme aracı](/tr/tools/exec)
- [Yükseltilmiş mod](/tr/tools/elevated)
- [Skills](/tr/tools/skills) — skill destekli otomatik izin verme davranışı
