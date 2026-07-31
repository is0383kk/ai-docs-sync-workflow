---
read_when:
    - Bağlantı/kimlik doğrulama sorunlarınız var ve yönlendirmeli çözümler istiyorsunuz
    - Güncelleme yaptınız ve hızlı bir doğruluk kontrolü istiyorsunuz
summary: '`openclaw doctor` için CLI başvurusu (durum denetimleri + yönlendirmeli onarımlar)'
title: Doktor
x-i18n:
    generated_at: "2026-07-26T22:39:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e2b0aa9b51d7bccd4357d3ec747be514a0245b44a90e6e6c7ea789ab68420465
    source_path: cli/doctor.md
    workflow: 16
---

# `openclaw doctor`

Gateway, kanallar, Pluginler, Skills, model yönlendirme, yerel durum ve yapılandırma geçişleri için sistem durumu denetimleri ve hızlı düzeltmeler. Bir şey beklendiği gibi çalışmadığında ve sorunun ne olduğunu tek bir komutla açıklamak istediğinizde bunu kullanın.

Gateway durumu, performansı düşmüş SecretRef sahipleri bildirdiğinde doctor, her soğuk veya eski sahip, etkilenen yapılandırma yolu, düzeltilmiş neden ve `openclaw secrets reload` yeniden deneme komutuyla birlikte bir **Gizli anahtar çalışma zamanı performans düşüşü** uyarısı yazdırır.

Kanal giriş olayları teslim edilemeyenler kuyruğuna alındığında doctor, etkilenen her kanal hesabını adlandırır ve inceleme ile kurtarma için [`openclaw channels dead-letters list`](/tr/cli/channels#inbound-dead-letters) bölümüne yönlendirir.

İlgili:

- Sorun giderme: [Sorun giderme](/tr/gateway/troubleshooting)
- Güvenlik denetimi: [Güvenlik](/tr/gateway/security)

## Duruşlar

Doctor beş duruşa sahiptir:

| Duruş                    | Komut                                     | Davranış                                                                                 |
| ------------------------ | ----------------------------------------- | ---------------------------------------------------------------------------------------- |
| İnceleme                 | `openclaw doctor`                         | İnsan odaklı denetimler ve yönlendirmeli istemler.                                        |
| Onarım                   | `openclaw doctor --fix`                   | Etkileşimsiz onarım güvenli olmadığı sürece istemleri kullanarak desteklenen onarımları uygular. |
| Lint                     | `openclaw doctor --lint`                  | CI, ön kontrol ve inceleme kapıları için salt okunur yapılandırılmış bulgular.            |
| Paylaşılan SQLite bakımı | `openclaw doctor --state-sqlite compact`  | Standart paylaşılan durum veritabanına açıkça denetim noktası uygular, onu sıkıştırır ve doğrular. |
| Oturum SQLite geçişi     | `openclaw doctor --session-sqlite <mode>` | Oturum durumunu inceler, içe aktarır, doğrular, sıkıştırır, kurtarır veya geri yükler.    |

Otomasyon kararlı bir sonuca ihtiyaç duyduğunda `--lint` seçeneğini tercih edin. Bir insan operatör doctor'ın yapılandırmayı veya durumu düzenlemesini istediğinde `--fix` seçeneğini tercih edin.

## Örnekler

```bash
openclaw doctor
openclaw doctor --lint
openclaw doctor --lint --json
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --all
openclaw doctor --lint --allow-exec
openclaw doctor --deep
openclaw doctor --fix
openclaw doctor --fix --non-interactive
openclaw doctor --generate-gateway-token
openclaw doctor --post-upgrade
openclaw doctor --post-upgrade --json
openclaw doctor --state-sqlite compact
openclaw doctor --state-sqlite compact --json
openclaw doctor --session-sqlite inspect --session-sqlite-all-agents
openclaw doctor --session-sqlite dry-run --session-sqlite-agent main --json
openclaw doctor --session-sqlite import --session-sqlite-all-agents
openclaw doctor --session-sqlite validate --session-sqlite-all-agents --json
openclaw doctor --session-sqlite compact --session-sqlite-all-agents
openclaw doctor --session-sqlite recover --github-issue
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

Kanala özgü izinler için `doctor` yerine kanal yoklamalarını kullanın:

```bash
openclaw channels capabilities --channel discord --target channel:<channel-id>
openclaw channels status --probe
```

`channels capabilities`, belirli bir kanal hedefi için botun geçerli izinlerini bildirir. `channels status --probe`, yapılandırılmış tüm kanalları ve sese otomatik katılma hedeflerini denetler.

## Seçenekler

| Seçenek                         | Etki                                                                                                                                                                                  |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--no-workspace-suggestions`    | Çalışma alanı belleği/arama önerilerini devre dışı bırakır.                                                                                                                            |
| `--yes`                         | İstemde bulunmadan varsayılanları kabul eder.                                                                                                                                          |
| `--repair` / `--fix`            | İstemde bulunmadan önerilen hizmet dışı onarımları uygular (`--fix` bir diğer addır). Gateway hizmeti kurulumları/yeniden yazımları hâlâ etkileşimli onay veya açık `gateway` komutları gerektirir. |
| `--force`                       | Özel hizmet yapılandırmasının üzerine yazmak da dâhil olmak üzere kapsamlı onarımları uygular.                                                                                         |
| `--non-interactive`             | İstemler olmadan çalışır; yalnızca güvenli geçişler ve hizmet dışı onarımlar.                                                                                                          |
| `--generate-gateway-token`      | Bir Gateway belirteci oluşturur ve yapılandırır.                                                                                                                                       |
| `--allow-exec`                  | Gizli anahtarları doğrularken doctor'ın yapılandırılmış `exec` SecretRef'lerini yürütmesine izin verir.                                                                     |
| `--deep`                        | Ek Gateway kurulumları için sistem hizmetlerini tarar; yakın zamandaki Gateway gözetmen yeniden başlatma devirlerini bildirir.                                                         |
| `--lint`                        | Modernleştirilmiş sistem durumu denetimlerini salt okunur modda çalıştırır ve tanılama bulguları üretir.                                                                                |
| `--post-upgrade`                | Yükseltme sonrası Plugin uyumluluk yoklamalarını çalıştırır; bulgular standart çıktıya gider; herhangi bir hata düzeyi bulgusu varsa çıkış kodu 1 olur.                                |
| `--state-sqlite <mode>`         | Açık paylaşılan durum SQLite bakımını çalıştırır. Tek mod `compact` modudur.                                                                                                   |
| `--session-sqlite <mode>`       | Hedeflenen oturum SQLite geçiş modunu çalıştırır: `inspect`, `dry-run`, `import`, `validate`, `compact`, `recover` veya `restore`. |
| `--session-sqlite-store <path>` | `--session-sqlite` ile: eski bir `sessions.json` depo yolu seçer.                                                                                                                   |
| `--session-sqlite-agent <id>`   | `--session-sqlite` ile: yapılandırılmış bir ajan seçer.                                                                                                                               |
| `--session-sqlite-all-agents`   | `--session-sqlite` ile: yapılandırılmış ve keşfedilmiş ajan depolarını seçer.                                                                                                          |
| `--github-issue`                | `--session-sqlite recover` ile: temizlenmiş bir openclaw/openclaw sorun raporu hazırlar; doctor, `--yes` veya etkileşimli onay sonrasında bunu `gh` ile oluşturur.       |
| `--json`                        | `--lint` ile: JSON bulguları. `--post-upgrade` ile: `{ probesRun, findings }`. `--state-sqlite` veya `--session-sqlite` ile: bakım raporu JSON biçiminde.                         |
| `--severity-min <level>`        | `--lint` ile: `info`, `warning` veya `error` düzeyinin altındaki bulguları çıkarır.                                                          |
| `--all`                         | `--lint` ile: varsayılan kümeden hariç tutulan isteğe bağlı denetimler dâhil olmak üzere kayıtlı tüm denetimleri çalıştırır.                                                |
| `--skip <id>`                   | `--lint` ile: bir denetim kimliğini atlar. Tekrarlanabilir.                                                                                                                  |
| `--only <id>`                   | `--lint` ile: yalnızca belirtilen denetim kimliklerini çalıştırır. Tekrarlanabilir.                                                                                          |

`--severity-min`, `--all`, `--only` ve `--skip` yalnızca `--lint` ile birlikte kabul edilir; `--json`, `--lint`, `--post-upgrade`, `--state-sqlite` ve `--session-sqlite` ile kabul edilir.

## Lint modu

`openclaw doctor --lint` salt okunurdur: istem, onarım veya yapılandırma/durum yeniden yazımı yoktur.

```bash
openclaw doctor --lint
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --json
openclaw doctor --lint --all
openclaw doctor --lint --allow-exec
openclaw doctor --lint --only core/doctor/gateway-config --json
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

İnsanlara yönelik çıktı kısadır:

```text
doctor --lint: 6 denetim çalıştırıldı, 1 bulgu bulundu
  [warning] core/doctor/gateway-config gateway.mode - gateway.mode ayarlanmamış; Gateway başlatma işlemi engellenecek.
    düzeltme: `openclaw configure` komutunu çalıştırıp Gateway modunu (local/remote) ayarlayın veya `openclaw config set gateway.mode local` komutunu çalıştırın.
```

JSON çıktısı, betik oluşturma yüzeyidir:

```json
{
  "ok": false,
  "checksRun": 5,
  "checksSkipped": 0,
  "findings": [
    {
      "checkId": "core/doctor/gateway-config",
      "severity": "warning",
      "message": "gateway.mode ayarlanmamış; Gateway başlatma işlemi engellenecek.",
      "path": "gateway.mode",
      "fixHint": "`openclaw configure` komutunu çalıştırıp Gateway modunu (local/remote) ayarlayın veya `openclaw config set gateway.mode local` komutunu çalıştırın."
    }
  ]
}
```

Çıkış kodları:

| Kod | Anlam                                                         |
| --- | ------------------------------------------------------------- |
| `0`  | Seçilen önem eşiğinde veya üzerinde bulgu yok.                 |
| `1`  | En az bir bulgu seçilen eşiği karşılıyor.                      |
| `2`  | Lint bulguları üretilemeden önce komut/çalışma zamanı hatası.   |

`--severity-min`, hem hangi bulguların yazdırılacağını hem de çıkış eşiğini denetler: daha düşük önem dereceli `info`/`warning` bulguları mevcut olsa bile `openclaw doctor --lint --severity-min error` hiçbir şey yazdırmadan `0` koduyla çıkabilir.

`--all`, önem derecesi filtrelemesinden önce hangi denetimlerin seçileceğini belirler. Varsayılan lint çalıştırması derin, geçmişe yönelik veya onarılabilir eski kalıntıları ortaya çıkarma olasılığı daha yüksek olan denetimleri hariç tutar; eksiksiz envanter için `--all` kullanın. `--only <id>` en hassas seçicidir ve kayıtlı herhangi bir denetimi kimliğine göre çalıştırabilir.

`core/doctor/local-audio-acceleration`, bir konuşma modeli yüklemeden otomatik seçilen yerel STT komutunu, ayrı yeterli/istenen/gözlemlenen arka uç kanıtlarını ve geri dönüş sırasını bildirir. Bilgilendirici bir bulgu üretir; bu nedenle görüntülemek için `--severity-min info` seçeneğini ekleyin.

## Yapılandırılmış sistem durumu denetimleri

Modern doctor denetimleri küçük, bölünmüş bir sözleşme kullanır:

```ts
detect(ctx, scope?) -> HealthFinding[]
repair?(ctx, findings) -> HealthRepairResult
```

`detect()`, `doctor --lint` öğesine güç sağlar. `repair()` isteğe bağlıdır ve yalnızca `doctor --fix` / `doctor --repair` altında çalışır. Henüz bu yapıya geçirilmemiş denetimler eski doctor katkı akışını kullanmaya devam eder.

Onarım bağlamları `dryRun`/`diff` isteklerini taşıyabilir; onarım sonuçları yapılandırılmış `diffs` (yapılandırma/dosya düzenlemeleri) ve `effects` (hizmet, süreç, paket, durum veya diğer yan etkiler) döndürebilir; böylece dönüştürülen denetimler, değişiklik planlamasını `detect()` içine taşımadan `doctor --fix --dry-run` yönünde gelişebilir.

`repair()`, `status: "repaired" | "skipped" | "failed"` bildirir (durumun belirtilmemesi `repaired` anlamına gelir). Onarım `skipped` veya `failed` döndürdüğünde Doctor nedeni bildirir ve o denetimin doğrulamasını atlar. Başarılı bir onarımın ardından Doctor, onarılan bulgularla sınırlı olarak `detect()` işlemini yeniden çalıştırır; bulgu hâlâ mevcutsa Doctor değişikliği tamamlanmış saymak yerine bir onarım uyarısı bildirir.

Bir bulgu şunları içerir:

| Alan              | Amaç                                                   |
| ----------------- | ------------------------------------------------------ |
| `checkId`         | Atlama/yalnızca filtreleri ve CI izin listeleri için kararlı kimlik. |
| `severity`        | `info`, `warning` veya `error`.                         |
| `message`         | İnsan tarafından okunabilir sorun açıklaması.          |
| `path`            | Mevcut olduğunda yapılandırma, dosya veya mantıksal yol. |
| `line` / `column` | Mevcut olduğunda kaynak konumu.                        |
| `ocPath`          | Bir denetim gösterebiliyorsa kesin `oc://` adresi. |
| `fixHint`         | Önerilen operatör eylemi veya onarım özeti.             |

Modernleştirilmiş çekirdek Doctor denetimleri, insana yönelik `doctor` / `doctor --fix` davranışlarının sahibi olan sıralı Doctor katkısına bağlı kalır. Paylaşılan yapılandırılmış sistem durumu kayıt defteri genişletme noktasıdır: paketle gelen ve plugin destekli denetimler, sahip paketleri bunları etkin komut yoluna kaydettikten sonra çekirdek Doctor denetimlerinin ardından çalışır. `openclaw/plugin-sdk/health`, plugin yazarlarına aynı sözleşmeyi sunar.

## Denetim seçimi

```bash
openclaw doctor --lint --only core/doctor/gateway-config --json
openclaw doctor --lint --skip core/doctor/skills-readiness
openclaw doctor --lint --all --skip core/doctor/session-locks
```

`--only` ve `--skip` tam denetim kimliklerini kabul eder ve yinelenebilir. Bir `--only` kimliği kayıtlı değilse bu kimlik için hiçbir denetim çalışmaz; odaklanmış bir geçidin beklediğiniz denetimleri seçtiğini doğrulamak için çıktıda `checksRun`/`checksSkipped` kullanın.

## Yükseltme sonrası modu

`openclaw doctor --post-upgrade`, bir derleme veya yükseltmenin ardından zincirlemek üzere plugin uyumluluk yoklamalarını çalıştırır. Bulgular stdout'a gönderilir; herhangi bir bulguda `level: "error"` varsa çıkış kodu 1 olur. CI, topluluk `fork-upgrade` skill'i ve diğer yükseltme sonrası duman testi araçları için uygun, makine tarafından okunabilir bir zarf (`{ probesRun, findings }`) elde etmek üzere `--json` ekleyin. Yüklü plugin dizini eksik veya bozuksa JSON modu yine de `plugin.index_unavailable` hata bulgusunu içeren zarfı üretir.

Konteyner imajı başlangıcı, olağan "güncellemeden sonra Doctor'ı çalıştır"
akışının istisnasıdır. `openclaw gateway run` yeni bir OpenClaw sürümünde başladığında,
hazır olduğunu bildirmeden önce güvenli durum ve plugin onarımlarını
çalıştırır. Onarım güvenli biçimde tamamlanamazsa başlangıç sonlanır ve
konteyneri normal şekilde yeniden başlatmadan önce aynı bağlı durum/yapılandırmaya
karşı aynı imajı `openclaw doctor --fix` ile bir kez çalıştırmanızı söyler.

## Eski durum geçişi

`openclaw doctor --fix`, kalıcı dosyadan SQLite'a geçişlerin tek sahibidir. Tanınan her kaynağı doğrular ve sahiplenir, kanonik satırları yazar ve doğrular, bir geçiş makbuzu kaydeder, ardından kullanımdan kaldırılmış kaynağı siler. Çalışma zamanı kodu gecikmeli içe aktarma veya yedek okuma gerçekleştirmez.

Buna `<state-dir>/mcp-oauth/*.json` altındaki kullanımdan kaldırılmış MCP OAuth dosyaları dahildir. Onarımdan önce Gateway'i durdurun. Doctor geçerli kimlik bilgilerini `<state-dir>/state/openclaw.sqlite` içine aktarır, her iki depo da mevcut olduğunda var olan kanonik SQLite oturumunu korur, eski kalıcı OAuth `state` değerini kaldırır ve yeniden oluşturulmuş eski bir dosyanın oturumu kapatılmış kimlik bilgilerini yeniden etkinleştirmesini önlemek için makbuzunu kullanır. Kullanımdan kaldırılmış `.lock` yan dosyaları güvenli biçimde başarısız olur: Doctor eski bir sahip bildirirse daha eski bir OpenClaw sürecinin çalışmadığını doğrulayın, bu yan dosyayı kaldırın ve Doctor'ı yeniden çalıştırın.

## Paylaşılan durum SQLite Compaction işlemi

Şema sürümleme, bütünlük denetimleri ve sürüm düşürme kurtarması için [Veritabanı şemaları](/tr/reference/database-schemas) bölümüne bakın.

`openclaw doctor --state-sqlite compact`,
`<state-dir>/state/openclaw.sqlite` konumundaki kanonik paylaşılan durum veritabanı için
açık çevrimdışı bakımdır. Rastgele bir veritabanı yolunu kabul etmez,
normal Gateway çalışması tarafından hiçbir zaman çağrılmaz ve
`openclaw doctor --fix` parçası değildir. Komut, Gateway başlangıcıyla aynı durum
sahipliği kilidini alır ve doğrulama, denetim noktası oluşturma, `VACUUM` ve
son bütünlük denetimleri boyunca bu kilidi tutar. Bir Gateway veya başka bir
SQLite bakım komutu bu kilide sahipken çalışmayı reddeder. `OPENCLAW_ALLOW_MULTI_GATEWAY=1`,
yapılandırma başına Gateway tekil örneğini atlasa bile durum kilidi etkin kalır;
dolayısıyla bir operatör kabuğunun bakım sırasında Gateway hizmetini algılayabilmesi
için hizmetin ortamını devralması gerekmez.

Önce Gateway'i durdurun ve doğrulanmış bir yedek oluşturun:

```bash
openclaw gateway stop
openclaw backup create --verify
openclaw doctor --state-sqlite compact --json
openclaw gateway start
```

Komut:

1. Kanonik paylaşılan durum yolunda normal bir dosya gerektirir. Eksik
   veritabanı `skipped` olarak bildirilir ve başarıyla sonlanır.
2. Denetim noktası oluşturmadan veya dosyayı değiştirmeden önce desteklenen
   geçerli şema sürümünü ve `schema_meta.role = "global"` öğesini doğrular.
3. Meşgul olmayan bir `wal_checkpoint(TRUNCATE)` gerektirir. Denetim noktası meşgulse
   kalan tüm OpenClaw süreçlerini durdurun ve yeniden deneyin.
4. `auto_vacuum` değerini `INCREMENTAL` olarak ayarlar, tam bir `VACUUM` çalıştırır ve
   yeniden denetim noktası oluşturur.
5. `quick_check`, `integrity_check` ve `foreign_key_check` işlemlerini çalıştırır, ardından
   veritabanına ve SQLite yan dosyalarına yalnızca sahip izinlerini yeniden uygular.

JSON çıktısı; veritabanı ve WAL boyutlarını, serbest liste sayfalarını, sayfa boyutunu ve
Compaction öncesi ve sonrasındaki `auto_vacuum` değerini, ayrıca geri kazanılan baytları ve
`quick_check` ile `integrity_check` sonuçlarını bildirir. `foreign_key_check` güvenli biçimde
başarısız olacak şekilde uygulanır ve ayrı bir başarı alanı yoktur. SQLite, `auto_vacuum` değerini
hiçbiri için `0`, tam için `1` ve artımlı için `2` olarak bildirir.

Şema eskiyse, çalışan OpenClaw derlemesinden daha yeniyse veya bir agent
veritabanına aitse Compaction değişiklik yapmadan başarısız olur. Eski bir
paylaşılan durum şeması için önce `openclaw doctor --fix` çalıştırın. Daha yeni bir
şema için uyumlu bir yedeği geri yükleyin veya OpenClaw'ı yükseltin.

## Oturum SQLite geçişi

OpenClaw, eski oturum satırlarını ve transkript geçmişini Gateway başlangıcı sırasında
ve `openclaw doctor --fix` sırasında otomatik olarak her agent'ın
SQLite veritabanına aktarır. `openclaw doctor --session-sqlite <mode>`, bu geçiş için
hedefli inceleme ve doğrulama aracıdır. Geçerli çalışma zamanı
oturum satırları
`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` içinde bulunur. Eski
`sessions.json` dosyaları geçiş kaynaklarıdır. Etkin transkript JSONL dosyaları
içe aktarılır ve başarılı içe aktarmadan sonra etkin oturumlar dizininin dışına
arşivlenir; arşiv katmanındaki JSONL dosyaları çalışma zamanı
yedekleri değil, destek yapıtları olarak kalır.

Modlar:

| Mod        | Davranış                                                                                                               |
| ---------- | ---------------------------------------------------------------------------------------------------------------------- |
| `inspect`  | İçe aktarmadan eski kayıtların ve SQLite'ın sayılarını, ayrıca başvurulmayan JSONL dosyalarını okur.                    |
| `dry-run`  | Eski girdileri ve transkript JSONL dosyalarını ayrıştırır, içe aktarılabilir satırları sayar ve SQLite satırlarını yazmadan sorunları bildirir. |
| `import`   | Seçilen hedefler için eski girdileri ve transkript olaylarını SQLite'a aktarır.                                        |
| `validate` | Seçilen eski kaynakları SQLite satırları ve transkript olay sayılarıyla karşılaştırır.                                  |
| `compact`  | Büyük silme veya arşiv temizliği sonrasında boş sayfaları geri kazanmak için seçilen agent SQLite veritabanlarında denetim noktası oluşturur ve VACUUM çalıştırır. |
| `recover`  | En son başarısız geçiş çalıştırmasını geri yükler, hedeflerini doğrular ve temizlenmiş bir GitHub sorun raporu hazırlar. |
| `restore`  | SQLite verilerini silmeden, kaydedilmiş geçiş bildirimlerinden arşivlenmiş transkript yapıtlarını geri yükler.          |

Seçiciler:

- Varsayılan: bu eski depo dosyası mevcut olduğunda yapılandırılmış varsayılan agent deposu.
- `--session-sqlite-agent <id>`: yapılandırılmış tek bir agent.
- `--session-sqlite-all-agents`: yapılandırılmış agent depoları ve keşfedilen agent depoları.
- `--session-sqlite-store <path>`: tek bir açık eski `sessions.json` yolu.

Elle inceleme sırası:

```bash
openclaw doctor --session-sqlite inspect --session-sqlite-all-agents
openclaw doctor --session-sqlite dry-run --session-sqlite-all-agents --json
openclaw doctor --session-sqlite import --session-sqlite-all-agents
openclaw doctor --session-sqlite validate --session-sqlite-all-agents --json
openclaw doctor --session-sqlite compact --session-sqlite-all-agents
openclaw doctor --session-sqlite recover --github-issue
```

Önemli geçmişi olan bir kurulumda `import` çalıştırmadan önce OpenClaw durum dizinini
yedekleyin. Seçilen bir eski girdi SQLite'ta yoksa, oturum kimliği farklıysa veya
transkript olay sayısı farklıysa `validate` sıfır dışında bir kodla sonlanır.
`--session-sqlite-store <path>` kullanırken raporun beklenen hedef sayısını içerdiğini
kontrol edin; mevcut olmayan açık bir depo yolu hiçbir hedef seçmez.

SQLite silme işlemleri önce veritabanının içindeki sayfaları geri kazanır; veritabanı
dosyasını hemen küçültmeleri gerekmez. Büyük transkriptleri sildikten veya
arşivledikten sonra WAL dosyalarında denetim noktası oluşturmak, `VACUUM`
çalıştırmak ve önceki/sonraki veritabanı ile WAL boyutlarını bildirmek için
`openclaw doctor --session-sqlite compact --session-sqlite-all-agents` çalıştırın. Compaction; geçerli agent şemasına sahip normal bir dosya,
seçilen agent'ın kalıcı sahip meta verileri ve Doctor sürecinde açık tanıtıcı
bulunmamasını gerektirir. Yıkıcı `import`, `compact`, `recover` ve `restore` modları,
tüm işlemleri boyunca Gateway başlangıcıyla aynı durum sahipliği kilidini tutar;
`inspect`, `dry-run` ve `validate` salt okunur kalır ve bu kilidi almaz. Önce
Gateway'i durdurun. Yıkıcı modlar, canlı yazmalarla veya başka bir bakım komutuyla
yarışmak yerine başarısız olur. Yıkıcı bir `--session-sqlite-store`
hedefi etkin durum dizininin içinde olmalıdır; başka bir kuruluma bakım uygulamadan önce
`OPENCLAW_STATE_DIR` değerini deponun sahibi olan durum dizinine ayarlayın.
Mevcut sabit bağlantılı hedefler reddedilir; çünkü başka bir yol kilitli durum
dizininin dışında aynı veritabanı inode'unu paylaşabilir. Aynı sahiplik
denetimleri SQLite WAL, paylaşılan bellek ve geri alma günlüğü yan dosyalarını kapsar.

Her içe aktarma, transkript yapıtlarını arşive taşımadan önce
`~/.openclaw/session-sqlite-migration-runs/` altında bir bildirim yazar.
Yapıtlar taşındıktan sonra başlangıç başarısız bir oturum SQLite geçişi bildirirse
kurtarmayı çalıştırın:

```bash
openclaw doctor --session-sqlite recover --github-issue
```

Kurtarma, en son başarısız geçiş manifestini seçer, yalnızca manifestin
arşivlenmiş yapıtlarını geri yükler, etkilenen hedefleri doğrular, temizlenmiş
`.failure.md` ve `.failure.json` raporlarını yeniler ve döküm içeriklerini,
ham ortamı, gizli bilgileri ve sınırsız yapılandırmayı içermeyen bir GitHub sorunu
gövdesi hazırlar. Başarısız bir geçiş manifesti bulunmadığında ancak seçili bir
aracı SQLite veritabanı bozuk olduğunda, veritabanı olmadığında veya ana
veritabanı olmadan günlük yan dosyalarına sahip olduğunda kurtarma, eksiksiz
dosya kümesini geçici bir inceleme dizinine kopyalar. SQLite, özgün adli inceleme
dosyaları değiştirilmeden kalırken `quick_check`, `integrity_check` ve
`foreign_key_check` çalışmadan önce bu tek kullanımlık kopyadaki geçerli bir etkin
günlüğü geri alabilir. Başarısız bütünlük denetimleri veya sahipsiz yan dosyalar,
keşfedilen kümenin tamamını tek bir `.corrupt-<timestamp>` son ekiyle yeniden
adlandırarak DB, WAL, SHM ve geri alma günlüğü dosyalarını korur. Yakalanan bir
yeniden adlandırma hatası, hata bildirilmeden önce taşınmış dosyaları geri
taşır; böylece kurtarılabilir bir dosya kümesi sessizce bölünmez. Kurtarma
işleminden önce Gateway'i durdurun; etkin biçimde değişen bir SQLite dosya
kümesini kopyalamak veya yeniden adlandırmak güvenli değildir ve işletim
sistemleri arasında farklı davranır. `--github-issue --yes` ile doctor, sorunu
`openclaw/openclaw` içinde oluşturmak için GitHub CLI'ı kullanır; onay olmadan
yerel destek raporunu yazar ve önceden doldurulmuş bir sorun URL'si yazdırır.

`restore`, daha düşük düzeyli geri alma işlemi olarak kalır. Manifest
`sourcePath -> archivePath` kayıtlarını kullanır, arşivlenmiş yapıtları yalnızca özgün
yol eksik olduğunda geri taşır, her iki yol da mevcut olduğunda çakışmaları
bildirir ve SQLite veritabanını yerinde bırakır.

### Oturum SQLite Geçişinden Sonra Eski Sürüme Dönme

Dosya destekli eski bir OpenClaw sürümünü başlatmadan önce arşivlenmiş eski
döküm yapıtlarını geri yükleyin:

```bash
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

Eski sürümler, `sessions.json` girdilerini ve bu girdilerde kaydedilen
`sessionFile` yollarını okur. SQLite geçişinden sonra başarılı içe aktarma
işlemleri, etkin JSONL dökümlerini `session-sqlite-import-archive/` konumuna taşır; bu nedenle
geri yükleme, manifestte kayıtlı bu yapıtları özgün yollarına geri taşıyana
kadar eski çalışma zamanı bu geçmişi göremez.

Geri yükleme, SQLite verilerini silmez. SQLite'a geçişten sonra oluşturulan
oturumlar yalnızca SQLite'ta bulunur ve eski çalışma zamanında görünmez.
Daha sonra yeniden yükseltirseniz OpenClaw'ın içe aktarmadan önce geri yüklenen
eski yapıtları SQLite satırlarıyla karşılaştırabilmesi için yukarıdaki normal
geçiş doğrulama sırasını çalıştırın.

## Notlar

- Nix modunda (`OPENCLAW_NIX_MODE=1`), salt okunur doctor denetimleri çalışmaya devam eder ancak `openclaw.json` değiştirilemez olduğundan `doctor --fix`, `doctor --repair`, `doctor --yes` ve `doctor --generate-gateway-token` devre dışıdır. Bunun yerine bu kurulumun Nix kaynağını düzenleyin; nix-openclaw için önce agent yaklaşımını kullanan [Hızlı Başlangıç](https://github.com/openclaw/nix-openclaw#quick-start) bölümüne bakın.
- Etkileşimli istemler (anahtar zinciri/OAuth düzeltmeleri vb.) yalnızca stdin bir TTY olduğunda ve `--non-interactive` **ayarlanmamış** olduğunda çalışır. Başsız çalıştırmalar (cron, Telegram, terminal yok) istemleri atlar.
- Etkileşimsiz `doctor` çalıştırmaları, başsız sistem durumu denetimlerinin hızlı kalması için önceden Plugin yüklemeyi atlar. Etkileşimli oturumlar, eski sistem durumu/düzeltme akışının gerektirdiği Plugin yüzeylerini yüklemeye devam eder.
- `--lint`, `--non-interactive` seçeneğinden daha katıdır: her zaman salt okunurdur, hiçbir zaman istem göstermez ve güvenli geçişleri hiçbir zaman uygulamaz. Doctor'ın değişiklik yapmasını istediğinizde `doctor --fix` veya `doctor --repair` kullanın.
- Doctor, sırları denetlerken varsayılan olarak `exec` SecretRef'lerini çalıştırmaz. Yalnızca doctor'ın yapılandırılmış bu sır çözümleyicilerini çalıştırmasını bilinçli olarak istediğinizde `--allow-exec` seçeneğini (`--lint` ile veya onsuz) kullanın.
- Herhangi bir yapılandırma yazımı (`--fix` düzeltmesi dâhil), yedeği `~/.openclaw/openclaw.json.bak` konumuna döndürür (numaralandırılmış bir `.bak.1`..`.bak.4` halkasıyla). `--fix` ayrıca şema doğrulamasının bildirdiği bilinmeyen yapılandırma anahtarlarını kaldırır ve her kaldırma işlemini listeler; kısmen yazılmış yükseltme durumu, geçişi tamamlanmadan kaldırılmasın diye güncelleme sürerken bunu atlar.
- `openclaw.json` ayrıştırılamazsa ve bilinen son sağlam yapılandırma kurtarılamazsa `doctor --fix`, özgün dosyayı `openclaw.json.clobbered.<timestamp>` olarak korur, geçerli dosyayı değiştirmeden bırakır ve kısmi bir değiştirme dosyası yazmak yerine hatayla çıkar.
- Gateway yaşam döngüsünü başka bir gözetmen yönetiyorsa `OPENCLAW_SERVICE_REPAIR_POLICY=external` ayarını yapın. Doctor, Gateway/hizmet durumunu bildirmeye ve hizmet dışı düzeltmeleri uygulamaya devam eder ancak hizmet kurulumunu/başlatılmasını/yeniden başlatılmasını/önyüklemesini ve eski hizmet temizliğini atlar.
- Doctor, yönetilen Gateway'e uygulanan yığın sınırını ve geçerli ana makine ya da kapsayıcı bellek sınırı için kullanılan uyarlanabilir türetimi bildirir. Bir düzeltme geçişinin dışında aynı raporu almak için `openclaw gateway status` kullanın.
- Linux'ta doctor, etkin olmayan ek Gateway benzeri systemd birimlerini yok sayar ve düzeltme sırasında çalışan bir systemd Gateway hizmetinin komut/giriş noktası meta verilerini yeniden yazmaz. Önce hizmeti durdurun veya etkin başlatıcıyı değiştirmek için `openclaw gateway install --force` kullanın.
- `doctor --fix --non-interactive`, eksik veya güncelliğini yitirmiş Gateway hizmet tanımlarını bildirir ancak güncelleme düzeltme modu dışında bunları kurmaz ya da yeniden yazmaz. Eksik bir hizmet için `openclaw gateway install`, başlatıcıyı değiştirmek için ise `openclaw gateway install --force` çalıştırın.
- Durum bütünlüğü denetimleri, oturumlar dizinindeki sahipsiz transkript dosyalarını algılar. Bunları `.deleted.<timestamp>` olarak arşivlemek etkileşimli onay gerektirir; `--fix`, `--yes` ve başsız çalıştırmalar dosyaları yerinde bırakır.
- Doctor, eski cron işi biçimleri için `~/.openclaw/cron/jobs.json` (veya `cron.store`) dosyasını tarar ve standart satırları SQLite'a aktarmadan önce bunları yeniden yazar.
- Doctor, açık bir `payload.model` geçersiz kılmasına sahip cron işlerini; sağlayıcı ad alanı sayıları ve `agents.defaults.model` ile uyuşmazlıkları dâhil olmak üzere bildirir. Böylece varsayılan modeli devralmayan zamanlanmış işler, kimlik doğrulama veya faturalandırma incelemeleri sırasında görünür olur.
- Doctor, hâlâ yürütülüyor olarak işaretlenmiş (`state.runningAtMs`) cron işlerini bildirir; bu durum `openclaw cron list` tarafından `running` olarak gösterilmelerine neden olabilir. Bu denetim salt okunurdur: işaretlenmiş bir işi o anda hiçbir Gateway yürütmüyorsa sonraki cron hizmeti başlangıcı, kesintiye uğrayan çalıştırmayı kaydeder ve işareti temizler.
- Linux'ta doctor, kullanıcının crontab'ı hâlâ bakımı yapılmayan eski `~/.openclaw/bin/ensure-whatsapp.sh` aracını çalıştırıyorsa uyarır; cron, systemd kullanıcı veri yolu ortamına sahip olmadığında bu araç `Gateway inactive` değerini yanlış bildirebilir.
- WhatsApp etkinleştirildiğinde doctor, yerel `openclaw-tui` istemcileri hâlâ çalışırken performansı düşmüş bir Gateway olay döngüsü olup olmadığını denetler. `doctor --fix`, WhatsApp yanıtlarının eski TUI yenileme döngülerinin arkasında kuyruğa alınmaması için yalnızca doğrulanmış yerel TUI istemcilerini durdurur.
- HTTP(S) proxy ortam değişkenleri mevcutken `tools.web.fetch.useTrustedEnvProxy` devre dışıysa doctor, `web_fetch` tarafından hâlâ doğrudan yönlendirme kullanıldığını açıklar, kısa bir doğrudan TLS bağlantı denemesi gerçekleştirir ve açık katılım seçeneğini belirtir. Proxy güvenini hiçbir zaman otomatik olarak etkinleştirmez.
- Doctor; birincil modeller, yedekler, model izin listeleri, görüntü/video oluşturma modelleri, Heartbeat/alt agent/Compaction geçersiz kılmaları, kancalar, kanal modeli geçersiz kılmaları, cron yükleri ve eski oturum/transkript rota sabitlemeleri genelindeki eski `codex/*` ve `openai-codex/*` model başvurularını standart `openai/*` başvurularına dönüştürür. `--fix` ayrıca güvenli olduğunda eski `models.providers.codex` ve `models.providers.openai-codex` yapılandırmalarını birleştirir, eski `openai-codex:*` kimlik doğrulama profillerini ve `auth.order.openai-codex` girdilerini `openai:*` konumuna taşır, Codex niyetini sağlayıcı/model kapsamlı `agentRuntime.id: "codex"` girdilerine aktarır, eski tüm-agent/oturum çalışma zamanı sabitlemelerini kaldırır ve düzeltilen OpenAI agent başvurularını doğrudan OpenAI API anahtarıyla kimlik doğrulama yerine Codex kimlik doğrulama yönlendirmesinde tutar.
- Doctor, uyumlu saklanmış kimlik bilgileri bulunmasına rağmen başvurdukları profillerin tümü kaybolmuş olan boş olmayan `auth.order.<provider>` listelerini bildirir. `doctor --fix`, yalnızca bu eski geçersiz kılmaları silerek agent başına otomatik kimlik bilgisi seçimini geri getirir; açıkça boş sıralamalar, kısmen geçerli listeler ve uyumlu saklanmış kimlik bilgisi bulunmayan sıralamalar değişmeden kalır. Etkin bir SQLite kimlik doğrulama deposu okunamıyorsa veya hatalı biçimlendirilmişse doctor, bu düzeltmeyi neden atladığını açıklar. Yapılandırma yeniden yükleme modu yazımı otomatik olarak uygulamıyorsa kimlik doğrulama durumunu yeniden denetlemeden önce çalışan Gateway'i yeniden başlatın.
- Doctor, eski OpenClaw sürümlerinden kalan eski Plugin bağımlılığı hazırlama durumunu temizler ve eş bağımlılık olarak bildiren yönetilen npm Pluginleri için ana makinenin `openclaw` paketini yeniden bağlar. Ayrıca yapılandırmanın başvurduğu eksik indirilebilir Pluginleri (`plugins.entries`, yapılandırılmış kanallar, yapılandırılmış sağlayıcı/arama ayarları, yapılandırılmış agent çalışma zamanları) düzeltir. Paket güncellemeleri sırasında doctor, paket değişimi tamamlanana kadar paket yöneticisi Plugin düzeltmesini atlar; yapılandırılmış bir Pluginin hâlâ kurtarılması gerekiyorsa sonrasında `openclaw doctor --fix` komutunu yeniden çalıştırın. İndirme başarısız olursa doctor kurulum hatasını bildirir ve sonraki düzeltme girişimi için yapılandırılmış Plugin girdisini korur.
- Doctor, Plugin keşfi sağlıklı olduğunda eksik Plugin kimliklerini `plugins.allow`/`plugins.deny`/`plugins.entries` içinden ve bunlarla eşleşen sahipsiz kanal yapılandırmasını, Heartbeat hedeflerini ve kanal modeli geçersiz kılmalarını kaldırarak eski Plugin yapılandırmasını düzeltir.
- Doctor, etkilenen `plugins.entries.<id>` girdisini devre dışı bırakıp geçersiz `config` yükünü kaldırarak geçersiz Plugin yapılandırmasını karantinaya alır. Gateway başlangıcı zaten yalnızca bu hatalı Plugini atladığından diğer Pluginler ve kanallar çalışmaya devam eder.
- Doctor, kullanımdan kaldırılmış `plugins.entries.codex.config.codexDynamicToolsProfile` öğesini kaldırır; Codex uygulama sunucusu, Codex'e özgü çalışma alanı araçlarını her zaman yerel tutar.
- Doctor, eski düz Talk yapılandırmasını (`talk.voiceId`, `talk.modelId` ve benzerleri) otomatik olarak `talk.provider` + `talk.providers.<provider>` biçimine geçirir. Tek fark nesne anahtarlarının sırası olduğunda tekrarlanan `doctor --fix` çalıştırmaları artık Talk normalleştirmesini bildirmez/uygulamaz.
- Doctor, bir bellek araması hazırlık denetimi içerir ve gömme kimlik bilgileri eksik olduğunda `openclaw configure --section model` önerebilir.
- Doctor, hiçbir komut sahibi yapılandırılmadığında uyarır. Komut sahibi, yalnızca sahibin kullanabildiği komutları çalıştırmasına ve tehlikeli eylemleri onaylamasına izin verilen insan operatör hesabıdır. DM eşleştirmesi yalnızca birinin botla konuşmasına izin verir; ilk sahip önyüklemesi mevcut olmadan önce bir göndericiyi onayladıysanız `commands.ownerAllowFrom` ayarını açıkça yapın.
- Doctor, Codex modundaki agent'lar yapılandırıldığında ve operatörün Codex ana dizininde kişisel Codex CLI varlıkları bulunduğunda bir bilgi notu bildirir. Yerel Codex uygulama sunucusu başlatmaları, agent başına yalıtılmış ana dizinler kullanır; gerekirse önce Codex Pluginini kurun, ardından bilinçli olarak aktarılması gereken varlıkların envanterini çıkarmak için `openclaw migrate plan codex` kullanın.
- Doctor, varsayılan agent için izin verilen Skills geçerli çalışma zamanı ortamında kullanılamadığında (eksik ikili dosyalar, ortam değişkenleri, yapılandırma veya işletim sistemi gereksinimleri) uyarır. `doctor --fix`, kullanılamayan bu Skills'i `skills.entries.<skill>.enabled=false` ile devre dışı bırakabilir; Skill'in etkin kalmasını istiyorsanız bunun yerine eksik gereksinimi kurun/yapılandırın.
- Korumalı alan modu etkinleştirilmiş ancak Docker kullanılamıyorsa doctor, düzeltme adımlarını (`install Docker` veya `openclaw config set agents.defaults.sandbox.mode off`) içeren belirgin bir uyarı bildirir.
- Eski korumalı alan kayıt dosyaları veya parça dizinleri (`~/.openclaw/sandbox/containers.json`, `~/.openclaw/sandbox/browsers.json`, `~/.openclaw/sandbox/containers/` veya `~/.openclaw/sandbox/browsers/`) mevcutsa doctor bunları bildirir; `--fix`, geçerli girdileri SQLite'a taşır ve geçersiz eski dosyaları karantinaya alır.
- `gateway.auth.token`/`gateway.auth.password` SecretRef tarafından yönetiliyorsa ve geçerli komut yolunda kullanılamıyorsa doctor salt okunur bir uyarı bildirir ve düz metin yedek kimlik bilgileri yazmaz. Çalıştırma tabanlı SecretRef'lerde doctor, `--allow-exec` mevcut olmadığı sürece çalıştırmayı atlar.
- Bir düzeltme yolunda kanal SecretRef incelemesi başarısız olursa doctor erken çıkmak yerine devam eder ve bir uyarı bildirir.
- Durum dizini geçişlerinden sonra doctor, etkin varsayılan Telegram veya Discord hesapları ortam yedeğine bağımlıysa ve `TELEGRAM_BOT_TOKEN` ya da `DISCORD_BOT_TOKEN` doctor işlemi tarafından kullanılamıyorsa uyarır.
- Telegram `allowFrom` kullanıcı adının otomatik çözümlenmesi (`doctor --fix`), geçerli komut yolunda çözümlenebilir bir Telegram belirteci gerektirir. Belirteç incelemesi kullanılamıyorsa doctor bir uyarı bildirir ve bu geçişte otomatik çözümlemeyi atlar.

## macOS: `launchctl` ortam geçersiz kılmaları

Daha önce `launchctl setenv OPENCLAW_GATEWAY_TOKEN ...` (veya `...PASSWORD`) çalıştırdıysanız bu değer yapılandırma dosyanızı geçersiz kılar ve kalıcı "yetkisiz" hatalarına neden olabilir.

```bash
launchctl getenv OPENCLAW_GATEWAY_TOKEN
launchctl getenv OPENCLAW_GATEWAY_PASSWORD

launchctl unsetenv OPENCLAW_GATEWAY_TOKEN
launchctl unsetenv OPENCLAW_GATEWAY_PASSWORD
```

## İlgili

- [CLI başvurusu](/tr/cli)
- [Gateway doctor](/tr/gateway/doctor)
