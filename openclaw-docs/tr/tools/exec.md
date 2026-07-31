---
read_when:
    - Exec aracını kullanma veya değiştirme
    - stdin veya TTY davranışında hata ayıklama
summary: Exec aracı kullanımı, stdin modları ve TTY desteği
title: Çalıştırma aracı
x-i18n:
    generated_at: "2026-07-26T23:42:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9c16b5122c527c069a4d1a0c1649726073339e95b9084100c1a0f45ebcae759d
    source_path: tools/exec.md
    workflow: 16
---

Çalışma alanında kabuk komutları çalıştırın. `exec`, değişiklik yapan bir kabuk yüzeyidir: komutlar, seçilen ana makine veya sandbox dosya sisteminin izin verdiği her yerde dosya oluşturabilir, düzenleyebilir veya silebilir. `write`, `edit` veya `apply_patch` gibi OpenClaw dosya sistemi araçlarının devre dışı bırakılması, `exec` aracını salt okunur hâle getirmez.

`process` aracılığıyla ön planda ve arka planda yürütmeyi destekler. `process` kullanımına izin verilmiyorsa `exec` eşzamanlı olarak çalışır ve `yieldMs`/`background` değerlerini yok sayar. Arka plan oturumları aracı başına kapsamlandırılır; `process` yalnızca aynı aracının oturumlarını görür.

## Parametreler

<ParamField path="command" type="string" required>
Çalıştırılacak kabuk komutu.
</ParamField>

<ParamField path="workdir" type="string" default="cwd">
Komutun çalışma dizini.
</ParamField>

<ParamField path="env" type="object">
Devralınan ortamın üzerine birleştirilen anahtar/değer ortam geçersiz kılmaları.
</ParamField>

<ParamField path="yieldMs" type="number" default="10000">
Bu gecikmeden (ms) sonra komutu otomatik olarak arka plana alın.
</ParamField>

<ParamField path="background" type="boolean" default="false">
`yieldMs` beklemek yerine komutu hemen arka plana alın.
</ParamField>

<ParamField path="timeout" type="number" default="tools.exec.timeoutSeconds">
Bu çağrı için yapılandırılmış exec zaman aşımını saniye cinsinden geçersiz kılın. Ön plan, arka plan, `yieldMs`, gateway, sandbox ve node `system.run` yürütmeleri için geçerlidir. `timeout: 0`, söz konusu çağrı için exec işlemi zaman aşımını devre dışı bırakır.
</ParamField>

<ParamField path="pty" type="boolean" default="false">
Mevcut olduğunda sözde terminalde çalıştırın. Yalnızca TTY ile çalışan CLI'lar, kodlama aracıları ve terminal kullanıcı arayüzleri için kullanın.
</ParamField>

<ParamField path="host" type="'auto' | 'sandbox' | 'gateway' | 'node'" default="auto">
Yürütmenin yapılacağı yer. Bir sandbox çalışma zamanı etkin olduğunda `auto`, `sandbox` olarak; aksi durumda `gateway` olarak çözümlenir.
</ParamField>

<ParamField path="security" type="'deny' | 'allowlist' | 'full'">
Normal araç çağrılarında yok sayılır. `gateway`/`node` güvenliği, `tools.exec.mode` ve ana makine onayları dosyasından türetilir; yükseltilmiş mod yalnızca operatör yükseltilmiş erişimi açıkça verdiğinde tam erişimi zorunlu kılabilir.
</ParamField>

<ParamField path="ask" type="'off' | 'on-miss' | 'always'">
Temel sorma modu, `tools.exec.mode` ve ana makine onaylarından türetilir. Kanal kaynaklı model çağrılarında, etkin ana makine sorma ayarı `off` olduğunda çağrı başına `ask` yok sayılır; aksi takdirde yalnızca daha katı bir moda sıkılaştırılabilir.
</ParamField>

<ParamField path="node" type="string">
`host=node` olduğunda Node kimliği/adı.
</ParamField>

<ParamField path="elevated" type="boolean" default="false">
Yükseltilmiş mod isteyin: sandbox'tan çıkarak yapılandırılmış ana makine yoluna geçin. Yalnızca yükseltilmiş ayarı `full` olarak çözümlendiğinde `security=full` zorunlu kılınır.
</ParamField>

Notlar:

- `host` yalnızca `auto`, `sandbox`, `gateway` veya `node` kabul eder. Bu bir ana makine adı seçicisi değildir; ana makine adına benzeyen değerler komut çalışmadan önce reddedilir.
- Çağrı başına `host=node` kullanımına `auto` üzerinden izin verilir; çağrı başına `host=gateway` kullanımına yalnızca etkin bir sandbox çalışma zamanı olmadığında izin verilir.
- Ek yapılandırma olmadan da `host=auto` "doğrudan çalışır": sandbox yoksa `gateway` olarak çözümlenir; etkin bir sandbox varsa sandbox içinde kalır.
- `elevated`, sandbox'tan çıkarak yapılandırılmış ana makine yoluna geçer: varsayılan olarak `gateway`; `tools.exec.host=node` olduğunda (veya oturum varsayılanı `host=node` olduğunda) `node`. Yalnızca geçerli oturum/sağlayıcı için yükseltilmiş erişim etkinleştirildiğinde kullanılabilir.
- `gateway`/`node` onayları, ana makine onayları dosyası tarafından denetlenir.
- `node`, eşleştirilmiş bir Node (yardımcı uygulama veya başsız Node ana makinesi) gerektirir. Birden fazla Node varsa birini seçmek için `exec.node` veya `tools.exec.node` ayarlayın.
- `exec host=node`, Node'lar için tek kabuk yürütme yoludur; eski `nodes.run` sarmalayıcısı kaldırılmıştır.
- Windows dışındaki ana makinelerde exec, ayarlandığında `SHELL` kullanır; `SHELL`, `fish` ise fish ile uyumsuz bash yapılarını önlemek için `PATH` içinden `bash` (veya `sh`) tercih edilir, ikisi de yoksa `SHELL` seçeneğine geri dönülür.
- Windows ana makinelerinde exec, önce PowerShell 7 (`pwsh`) keşfini (Program Files, ProgramW6432, ardından PATH) tercih eder; ardından Windows PowerShell 5.1'e geri döner.
- Windows dışındaki gateway ana makinelerinde bash ve zsh exec komutları bir başlangıç anlık görüntüsü kullanır. OpenClaw, kabuk başlangıç dosyalarındaki kaynak olarak yüklenebilir diğer adları/işlevleri ve küçük, güvenli bir ortam kümesini `$OPENCLAW_STATE_DIR/cache/shell-snapshots/` içine kaydeder; ardından her exec komutundan önce bu anlık görüntüyü kaynak olarak yükler. Gizli bilgi gibi görünen değişkenler hariç tutulur; sandbox ve Node exec bu anlık görüntüyü kullanmaz. Bu anlık görüntü yolunu devre dışı bırakmak için Gateway işlemi ortamında `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` ayarlayın.
- Ana makine yürütmesi (`gateway`/`node`), ikili dosya ele geçirmeyi veya kod eklemeyi önlemek için `env.PATH` ve yükleyici geçersiz kılmalarını (`LD_*`/`DYLD_*`) reddeder.
- OpenClaw, kabuk/profil kurallarının exec aracı bağlamını algılayabilmesi için başlatılan komut ortamında (PTY ve sandbox yürütmesi dâhil) `OPENCLAW_SHELL=exec` ayarlar.
- Kanal kaynaklı çalıştırmalarda OpenClaw, kanal bu kimlikleri sağladığında `OPENCLAW_CHANNEL_CONTEXT` içinde dar kapsamlı bir gönderen/sohbet kimliği JSON yükünü de kullanıma sunar.
- `exec`, `openclaw channels login` veya `/approve` kabuk komutlarını çalıştıramaz: `openclaw channels login` etkileşimli bir kanal kimlik doğrulama akışıdır ve `/approve` bir kabuk üzerinden değil, onay komutu işleyicisi üzerinden yürütülmelidir. Kanal oturum açma işlemini gateway ana makinesindeki bir terminalde çalıştırın veya mevcut olduğunda kanala özgü bir oturum açma aracı aracı kullanın (örneğin `whatsapp_login`).
- Önemli: sandbox kullanımı **varsayılan olarak kapalıdır**. Sandbox kullanımı kapalıysa örtük `host=auto`, `gateway` olarak çözümlenir. Açıkça belirtilen `host=sandbox`, gateway ana makinesinde sessizce çalışmak yerine yine güvenli biçimde başarısız olur. Sandbox kullanımını etkinleştirin veya onaylarla birlikte `host=gateway` kullanın.
- Betik ön kontrol denetimleri (yaygın Python/Node kabuk söz dizimi hataları için) yalnızca etkin `workdir` sınırı içindeki dosyaları inceler. Bir betik yolu `workdir` dışında çözümlenirse söz konusu dosya için ön kontrol atlanır. `host=gateway` olduğunda ve etkin politika `ask=off` ile birlikte `security=full` olduğunda da ön kontrol tamamen atlanır.
- Şimdi başlayan uzun süreli çalışmalar için işi bir kez başlatın ve özellik etkin olduğunda, komut çıktı ürettiğinde veya başarısız olduğunda gerçekleşen otomatik tamamlanma uyandırmasına güvenin. Günlükler, durum, girdi veya müdahale için `process` kullanın; zamanlamayı uyku döngüleri, zaman aşımı döngüleri veya tekrarlanan yoklamayla taklit etmeyin.
- Aracı tarafından başlatılan arka plan komutları tamamlanana kadar Web, iOS ve Android arka plan görevi görünümlerinde görünür. Görev defteri, tamamlanma Heartbeat'i aracıyı yeniden uyandırmadan önce sonlandırılır.
- Daha sonra veya bir programa göre yapılması gereken işler için `exec` uyku/gecikme kalıpları yerine Cron kullanın.

## Yapılandırma

| Anahtar                              | Varsayılan               | Notlar                                                                                                                                                  |
| ------------------------------------ | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tools.exec.timeoutSeconds`          | `1800`                   | Komut başına varsayılan exec zaman aşımı, saniye cinsinden. Çağrı başına `timeout` bunu geçersiz kılar; çağrı başına `timeout: 0` exec işlemi zaman aşımını devre dışı bırakır. |
| `tools.exec.host`                    | `auto`                   | Bir sandbox çalışma zamanı etkin olduğunda `sandbox`, aksi durumda `gateway` olarak çözümlenir.                                                   |
| `tools.exec.mode`                    | ana makineden türetilir  | Standart politika ayarı. Aşağıdaki [Modlar](#modes) bölümüne bakın.                                                                                      |
| `tools.exec.reviewer.model`          | yapılandırılmış birincil aracı | `mode=auto` incelemesi için isteğe bağlı sağlayıcı/model geçersiz kılması.                                                                        |
| `tools.exec.reviewer.timeoutMs`      | `30000`                  | İnsan incelemesine geri dönmeden önce inceleyici modelinin hazırlanması ve tamamlanması için aşama başına zaman aşımı.                                    |
| `tools.exec.node`                    | ayarlanmamış             |                                                                                                                                                         |
| `tools.exec.notifyOnExit`            | `true`                   | Doğru olduğunda, arka plana alınan exec oturumları bir sistem olayını kuyruğa ekler ve çıkışta bir Heartbeat ister.                                      |
| `tools.exec.approvalRunningNoticeMs` | `10000`                  | Onay geçidine tabi bir exec bundan daha uzun süre çalıştığında tek bir "çalışıyor" bildirimi yayınlar (`0` devre dışı bırakır).                      |
| `tools.exec.strictInlineEval`        | `false`                  | [Satır içi değerlendirme](#inline-eval-strictinlineeval) bölümüne bakın.                                                                                |
| `tools.exec.commandHighlighting`     | `false`                  | Doğru olduğunda, onay istemleri komut metninde ayrıştırıcıdan türetilen komut aralıklarını vurgulayabilir. Genel olarak veya aracı başına ayarlayın; onay politikasını değiştirmez. |
| `tools.exec.pathPrepend`             | ayarlanmamış             | Exec çalıştırmaları için `PATH` başına eklenecek dizinlerin listesi (yalnızca gateway + sandbox).                                                     |
| `tools.exec.safeBins`                | ayarlanmamış             | Açık izin listesi girdileri olmadan çalışabilen, yalnızca stdin kullanan güvenli ikili dosyalar. [Güvenli ikili dosyalar](/tr/tools/exec-approvals-advanced#safe-bins-stdin-only) bölümüne bakın. |
| `tools.exec.safeBinTrustedDirs`      | `/bin`, `/usr/bin`       | `safeBins` yol denetimleri için güvenilen ek açık dizinler. `PATH` girdilerine hiçbir zaman otomatik olarak güvenilmez.                              |
| `tools.exec.safeBinProfiles`         | ayarlanmamış             | Güvenli ikili dosya başına isteğe bağlı özel argv politikası (`minPositional`, `maxPositional`, `allowedValueFlags`, `deniedFlags`).                    |

Onaysız ana makine exec yürütmesi, gateway ve Node (`mode=full`) için varsayılandır; bu, `host=auto` ayarından değil ana makine politikası varsayılanlarından kaynaklanır. Onay/izin listesi davranışı istiyorsanız `tools.exec.mode` ayarlayın ve ana makine onayları dosyasını sıkılaştırın; [Exec onayları](/tr/tools/exec-approvals#yolo-mode-no-approval) bölümüne bakın. Sandbox durumundan bağımsız olarak gateway veya Node yönlendirmesini zorunlu kılmak için `tools.exec.host` ayarlayın ya da `/exec host=...` kullanın.

Örnek:

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
  },
}
```

### Modlar

`tools.exec.mode`, kalıcı standart politika ayarıdır. Çalışma zamanı güvenliği ve onay davranışı bundan türetilir.

| Mod         | security    | ask       | Davranış                                                                                                                       |
| ----------- | ----------- | --------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `deny`      | `deny`      | `off`     | Exec reddedilir.                                                                                                               |
| `allowlist` | `allowlist` | `off`     | Yalnızca izin listesindeki/güvenli ikili dosya komutları çalıştırılır; başka hiçbir şey için onay istenmez.                    |
| `ask`       | `allowlist` | `on-miss` | İzin listesiyle eşleşenler doğrudan çalıştırılır; diğer her şey için bir insandan onay istenir.                                |
| `auto`      | `allowlist` | `on-miss` | İzin listesiyle/güvenli ikili dosyayla eşleşenler doğrudan çalıştırılır; diğer her şey, bir insana sorulmadan önce OpenClaw'ın yerel otomatik inceleyicisine yönlendirilir. |
| `full`      | `full`      | `off`     | Onay kapısı yoktur.                                                                                                            |

Oturum başına `/exec ask=always`, kalıcı moda bakılmaksızın her seferinde yine bir insandan onay ister.

Otomatik inceleme onayı tek kullanımlıktır. Gateway üzerinde OpenClaw, çözümlenen yürütülebilir dosya yolunu inceleyiciye sağlar ve yürütmeyi aynı yola sabitler. Heredoc'lar, kabuk genişletmeleri veya desteklenmeyen sarmalayıcı tırnaklamaları gibi uygulanabilir tek bir yürütme planına indirgenemeyen komutlar, model bunlara normalde izin verecek olsa bile insan onayına geri döner.

Açık çalışma zamanı veya yerel politika tarafından zaten karara bağlanmamış Codex app-server komut onayları, insan onayı yolunu kullanır. Codex, inceleme kararını Codex'in çalıştırdığı komuta bağlayabilecek uygulanabilir ve çözümlenmiş bir yürütülebilir dosya sunmadığından OpenClaw, yapılandırılmış exec inceleyicisini bu istekler için çalıştırmaz.

### Satır içi değerlendirme (`strictInlineEval`)

`tools.exec.strictInlineEval`, `true` olduğunda satır içi yorumlayıcı değerlendirme biçimleri inceleyici veya açık onay gerektirir: `python -c`, `node -e`, `ruby -e`, `perl -e`, `php -r`, `lua -e`, `osascript -e` ve desteklenen diğer yorumlayıcılar ile komut taşıyıcılarındaki benzer biçimler (`awk`, `find -exec`, `make`, `sed`, `xargs` ve diğerleri). `mode=auto` içinde normal exec onay yolu, yerel otomatik inceleyicinin açıkça düşük riskli tek seferlik bir komuta izin vermesini sağlayabilir; doğrudan node ana bilgisayarındaki `system.run` çağrıları, komutu bir insan onayı yoluna aktaramadıkları için yine açık onay gerektirir. İnceleyici onay isterse istek bir insana gider. `allow-always`, zararsız yorumlayıcı/betik çağrılarını yine kalıcı hâle getirebilir ancak satır içi değerlendirme biçimleri kalıcı izin kurallarına dönüşmez.

### PATH işleme

- `host=gateway`: oturum açma kabuğunuzun `PATH` değerini exec ortamıyla birleştirir. `env.PATH` geçersiz kılmaları ana bilgisayarda yürütme için reddedilir. Arka plan programının kendisi yine asgari bir `PATH` ile çalışır:
  - macOS: `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`
  - Linux: `/usr/local/bin`, `/usr/bin`, `/bin`
  - Kullanıcı kabuk yapılandırmasının (`~/.zshenv` veya `/etc/zshenv` gibi) başlangıç sırasında öncelikli yolları geçersiz kılmasını önlemek için `tools.exec.pathPrepend` girdileri, yürütmeden hemen önce kabuk komutunun içindeki nihai `PATH` değerinin başına güvenli biçimde eklenir.
- `host=sandbox`: kapsayıcı içinde `sh -lc` (oturum açma kabuğu) çalıştırır; bu nedenle `/etc/profile`, `PATH` değerini sıfırlayabilir. OpenClaw, profil kaynakları yüklendikten sonra dahili bir ortam değişkeni aracılığıyla `env.PATH` değerini başa ekler (kabuk enterpolasyonu yoktur); `tools.exec.pathPrepend` burada da geçerlidir.
- `host=node`: yalnızca ilettiğiniz ve engellenmemiş ortam geçersiz kılmaları node'a gönderilir. `env.PATH` geçersiz kılmaları ana bilgisayarda yürütme için reddedilir ve node ana bilgisayarları tarafından yok sayılır. Bir node üzerinde ek PATH girdilerine ihtiyacınız varsa node ana bilgisayar hizmeti ortamını (systemd/launchd) yapılandırın veya araçları standart konumlara kurun.

Aracı başına node bağlama (yapılandırmada anahtarlı aracı kimliğini kullanın):

```bash
openclaw config get agents.entries
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
```

Control UI: **Cihazlar** sayfası, aynı ayarlar için küçük bir "Exec node bağlama" paneli içerir.

## Oturum geçersiz kılmaları (`/exec`)

`host`, `security`, `ask` ve `node` için **oturum başına** varsayılanları ayarlamak üzere `/exec` kullanın. Geçerli değerleri göstermek için bağımsız değişken olmadan `/exec` gönderin.

Örnek:

```text
/exec host=auto security=allowlist ask=on-miss node=mac-1
```

`/exec` yalnızca kanal izin listeleri/eşleştirme ve erişim grupları üzerinden **yetkili gönderenler** için dikkate alınır. Erişim grubu uygulaması her zaman etkindir. Yalnızca **oturum durumunu** günceller ve yapılandırmaya yazmaz. Yetkili harici kanal gönderenleri bu oturum varsayılanlarını ayarlayabilir. Dahili gateway/webchat istemcilerinin bunları kalıcı hâle getirmek için `operator.admin` kullanması gerekir.

Exec'i kesin olarak devre dışı bırakmak için araç politikası (`tools.deny: ["exec"]` veya aracı başına) aracılığıyla reddedin. `security=full` ve `ask=off` değerlerini açıkça ayarlamadığınız sürece ana bilgisayar onayları geçerliliğini korur.

## Exec onayları (eşlikçi uygulama / node ana bilgisayarı)

Korumalı alan aracıları, `exec` gateway veya node ana bilgisayarında çalışmadan önce istek başına onay gerektirebilir. Politika, izin listesi ve UI akışı için [Exec onayları](/tr/tools/exec-approvals) bölümüne bakın.

İnsan onayı gerektiğinde node ana bilgisayarı ve yerel olmayan gateway akışları, `status: "approval-pending"` ve bir onay kimliğiyle hemen döner. Yerel sohbet ve Web UI gateway akışları bunun yerine satır içinde bekleyebilir ve onaydan sonra nihai komut sonucunu döndürebilir. `approval-pending` sonucu, komutun başlamadığı anlamına gelir; bu nedenle ön plan geri dönüş uyarıları yalnızca onaylanan komut gerçekten satır içinde çalışırsa görünür. Onaylanan eşzamansız çalıştırmalar, komut ilerleme ve tamamlanma sistem olayları (`Exec running` / `Exec finished`) yayar; reddedilen veya zaman aşımına uğrayan onaylar terminal durumundadır ve aracı oturumunu bir ret sistem olayıyla uyandırmaz.

Yerel onay kartları/düğmeleri bulunan kanallarda aracı önce bu yerel UI'a güvenmeli ve yalnızca araç sonucu sohbet onaylarının kullanılamadığını veya manuel onayın tek yol olduğunu açıkça belirttiğinde manuel bir `/approve` komutu eklemelidir.

## İzin listesi + güvenli ikili dosyalar

Manuel izin listesi uygulaması, çözümlenen ikili dosya yolu glob'larıyla ve yalnız komut adı glob'larıyla eşleşir. Yalın adlar yalnızca PATH üzerinden çağrılan komutlarla eşleşir; dolayısıyla komut `rg` olduğunda `rg`, `/opt/homebrew/bin/rg` ile eşleşebilir ancak `./rg` veya `/tmp/rg` ile eşleşemez.

`security=allowlist` olduğunda kabuk komutlarına yalnızca her işlem hattı segmenti izin listesinde veya güvenli bir ikili dosya olduğunda otomatik olarak izin verilir. Zincirleme (`;`, `&&`, `||`) ve yönlendirmeler, her üst düzey segment izin listesini (güvenli ikili dosyalar dâhil) karşılamadığı sürece izin listesi modunda reddedilir. Yönlendirmeler desteklenmemeye devam eder. Kalıcı `allow-always` güveni bu kuralı atlamaz: zincirlenmiş bir komutta yine her üst düzey segmentin eşleşmesi gerekir.

`autoAllowSkills`, exec onaylarındaki ayrı bir kolaylık yoludur; manuel yol izin listesi girdileriyle aynı değildir. Kesin ve açık güven için `autoAllowSkills` devre dışı bırakılmış olarak tutulmalıdır.

İki denetimi farklı işler için kullanın:

- `tools.exec.safeBins`: küçük, yalnızca stdin kullanan akış filtreleri.
- `tools.exec.safeBinTrustedDirs`: güvenli ikili dosya yürütülebilir yolları için açıkça güvenilen ek dizinler.
- `tools.exec.safeBinProfiles`: özel güvenli ikili dosyalar için açık argv politikası.
- izin listesi: yürütülebilir dosya yolları için açık güven.

`safeBins` genel bir izin listesi olarak değerlendirilmemeli ve yorumlayıcı/çalışma zamanı ikili dosyaları (örneğin `python3`, `node`, `ruby`, `bash`) eklenmemelidir. Bunlara ihtiyacınız varsa açık izin listesi girdileri kullanın ve onay istemlerini etkin tutun.

`openclaw security audit`, yorumlayıcı/çalışma zamanı `safeBins` girdilerinde açık profiller eksik olduğunda uyarır ve `openclaw doctor --fix`, eksik özel `safeBinProfiles` girdilerini oluşturabilir. `openclaw security audit` ve `openclaw doctor`, `jq` gibi geniş davranışlı ikili dosyaları açıkça yeniden `safeBins` içine eklediğinizde de uyarır (`jq` ortam verilerini okuyabilir ve modüllerden veya başlangıç dosyalarından jq kodu yükleyebilir; bu nedenle bunun yerine açık izin listesi girdilerini veya onay kapılı çalıştırmaları tercih edin). `jq`, açıkça listelenmiş olsa bile güvenli bir ikili dosya olarak reddedilir. Yorumlayıcıları açıkça izin listesine alırsanız satır içi kod değerlendirme biçimlerinin yine inceleyici veya açık onay gerektirmesi için `tools.exec.strictInlineEval` özelliğini etkinleştirin.

Tüm politika ayrıntıları ve örnekleri için [Exec onayları](/tr/tools/exec-approvals-advanced#safe-bins-stdin-only) ve [Güvenli ikili dosyalar ile izin listesinin karşılaştırması](/tr/tools/exec-approvals-advanced#safe-bins-versus-allowlist) bölümlerine bakın.

## Örnekler

Ön plan:

```json
{ "tool": "exec", "command": "ls -la" }
```

Arka plan + yoklama:

```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

Yoklama, bekleme döngüleri için değil isteğe bağlı durum denetimi içindir. Otomatik tamamlanma uyandırması etkinse komut, çıktı yaydığında veya başarısız olduğunda oturumu uyandırabilir.

Tuş gönderme (tmux tarzı):

```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

Gönderme (yalnızca CR gönderir):

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

Yapıştırma (varsayılan olarak köşeli ayraçlı):

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## apply_patch

`apply_patch`, yapılandırılmış çok dosyalı düzenlemeler için `exec` aracının bir alt aracıdır. Varsayılan olarak etkindir ve tüm model sağlayıcıları tarafından kullanılabilir; `allowModels` bunu kısıtlayabilir. Yapılandırmayı yalnızca devre dışı bırakmak veya belirli modellerle sınırlandırmak istediğinizde kullanın:

```json5
{
  tools: {
    exec: {
      applyPatch: { workspaceOnly: true, allowModels: ["gpt-5.6-sol"] },
    },
  },
}
```

Notlar:

- Araç politikası yine geçerlidir; `allow: ["write"]`, `apply_patch` için örtük olarak izin verir.
- `deny: ["write"]`, `apply_patch` aracını reddetmez; `apply_patch` aracını açıkça reddedin veya yama yazma işlemlerinin de engellenmesi gerekiyorsa `deny: ["group:fs"]` kullanın.
- Yapılandırma `tools.exec.applyPatch` altında bulunur.
- `tools.exec.applyPatch.enabled` varsayılan olarak `true` değerindedir; aracı devre dışı bırakmak için `false` olarak ayarlayın.
- `tools.exec.applyPatch.workspaceOnly` varsayılan olarak `true` değerindedir (çalışma alanıyla sınırlı). Yalnızca `apply_patch` aracının çalışma alanı dizini dışına yazmasını/silmesini bilerek istiyorsanız `false` olarak ayarlayın.
- `tools.exec.applyPatch.allowModels`, model kimliklerinden oluşan isteğe bağlı bir izin listesidir (ham, `gpt-5.4` gibi veya tam, `openai/gpt-5.4` gibi). Ayarlandığında aracı yalnızca eşleşen modeller alır; ayarlanmadığında tüm modeller alır.

## İlgili

- [Exec Onayları](/tr/tools/exec-approvals) — kabuk komutları için onay kapıları
- [Korumalı Alan Kullanımı](/tr/gateway/sandboxing) — komutları korumalı ortamlarda çalıştırma
- [Arka Plan İşlemi](/tr/gateway/background-process) — uzun süre çalışan exec ve process aracı
- [Güvenlik](/tr/gateway/security) — araç politikası ve yükseltilmiş erişim
