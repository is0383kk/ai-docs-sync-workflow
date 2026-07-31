---
read_when:
    - Yerel OpenClaw durumu için birinci sınıf bir yedekleme arşivi istiyorsunuz
    - Tek bir OpenClaw SQLite veritabanının kompakt, doğrulanmış bir anlık görüntüsüne ihtiyacınız var
    - Sıfırlama veya kaldırma işleminden önce hangi yolların dahil edileceğini önizlemek istiyorsunuz
summary: '`openclaw backup` için CLI referansı (arşivler ve SQLite anlık görüntüleri)'
title: Yedekleme
x-i18n:
    generated_at: "2026-07-26T23:34:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dfb5a118545589b181cede26dab72e9d029d98a1cac5cfccedd9d9cf2c56d3b5
    source_path: cli/backup.md
    workflow: 16
---

# `openclaw backup`

OpenClaw durumu, yapılandırması, kimlik doğrulama profilleri, kanal/sağlayıcı kimlik bilgileri, oturumları ve isteğe bağlı olarak çalışma alanları için yerel bir yedekleme arşivi oluşturun.

```bash
openclaw backup create
openclaw backup create --output ~/Backups
openclaw backup create --dry-run --json
openclaw backup create --verify
openclaw backup create --no-include-workspace
openclaw backup create --only-config
openclaw backup verify ./2026-03-09T08-00-00.000+08-00-openclaw-backup.tar.gz
openclaw backup sqlite create --global --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite create --agent main --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite list --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite verify ~/Backups/openclaw-sqlite/<snapshot-id>
openclaw backup sqlite verify ~/Backups/openclaw-sqlite/<snapshot-id> --scratch ~/Private/openclaw-scratch
openclaw backup sqlite restore ~/Backups/openclaw-sqlite/<snapshot-id> --target ./restored/openclaw.sqlite
```

## Notlar

- Arşiv, çözümlenmiş kaynak yollarını ve arşiv düzenini içeren bir `manifest.json` barındırır.
- Varsayılan çıktı, geçerli çalışma dizininde zaman damgalı bir `.tar.gz` arşividir. Zaman damgalı dosya adları makinenizin yerel saat dilimini kullanır ve UTC farkını içerir. Geçerli çalışma dizini, yedeklenen bir kaynak ağacının içindeyse OpenClaw varsayılan arşiv konumu olarak ana dizininizi kullanır.
- Mevcut arşiv dosyalarının üzerine hiçbir zaman yazılmaz. Kaynak durum/çalışma alanı ağaçlarının içindeki çıktı yolları, kendi kendini arşive dahil etmeyi önlemek için reddedilir.
- `openclaw backup verify <archive>`, arşivin tam olarak bir kök manifest içerdiğini denetler; dizin geçişi biçimindeki arşiv yollarını ve SQLite yan dosyalarını reddeder; manifestte bildirilen her yükün mevcut olduğunu doğrular; her SQLite anlık görüntüsünün dosya biçimini doğrular ve standart OpenClaw veritabanlarında tam bütünlük ve rol denetimleri çalıştırır. Özel Plugin şemaları, sahip tarafından tanımlanan SQLite yetenekleri gerektirebilecekleri için opak kalır. `openclaw backup create --verify`, arşivi yazdıktan hemen sonra bu doğrulamayı çalıştırır.
- `openclaw backup create --only-config`, yalnızca etkin JSON yapılandırma dosyasını yedekler.

## SQLite anlık görüntüleri

Geniş kapsamlı bir durum arşivi yerine OpenClaw'a ait tek bir SQLite veritabanı için taşınabilir bir yapıt gerektiğinde `openclaw backup sqlite` kullanın.

Anlık görüntü oluşturma işlemi, adlandırılmış tam olarak bir kaynak kabul eder:

| Komut                                                           | Veritabanı                      |
| --------------------------------------------------------------- | ------------------------------- |
| `openclaw backup sqlite create --global --repository <dir>`     | Paylaşılan OpenClaw durumu      |
| `openclaw backup sqlite create --agent <id> --repository <dir>` | Her aracı için bir veritabanı   |

Depo, kaydedilmiş her anlık görüntü için bir dizin içerir. Her anlık görüntü dizini tam olarak şunları içerir:

- `manifest.json`
- `database.sqlite`

Anlık görüntü oluşturma işlemi, canlı veritabanını okumadan önce doğrular; uzun süreli tek bir okuma işlemi tutmadan kaydedilmiş WAL durumunu yakalamak için SQLite'ın çevrimiçi yedekleme API'sini kullanır; canlı veritabanını kapatır; özel kopyayı `VACUUM` ile sıkıştırır; oluşturulan veritabanını yeniden doğrular ve tamamlanan dizini mevcut yolların üzerine yazmadan yayımlar. Genel anlık görüntüler, silinen kuyruk yüklerinin boş sayfalarda tutulmaması için sıkıştırmadan önce geçici teslimat kuyruğu satırlarını kaldırır.

Canlı `.sqlite`, `-wal`, `-shm` veya `-journal` dosyalarını taşınabilirlik yapıtı olarak kopyalamayın. Yalnızca tamamlanmış anlık görüntü dizinlerini kopyalayın.

SQLite anlık görüntüleri kimlik doğrulama profilleri, oturum durumu, Plugin durumu ve diğer hassas kayıtları içerebilir. Depoları, canlı OpenClaw durum diziniyle aynı izinler, şifreleme, saklama ilkesi ve hedef kısıtlamalarıyla koruyun.

### Doğrulama ve geri yükleme

```bash
openclaw backup sqlite verify <snapshot-directory>
openclaw backup sqlite restore <snapshot-directory> --target <new-database-path>
```

Doğrulama; katı manifest biçimini, yapıt boyutunu ve SHA-256 değerini, SQLite bütünlüğünü, yabancı anahtarları, şema sürümünü, veritabanı rolünü ve sahibini, ayrıca OpenClaw'a ait dizin tanımlarını denetler.

Doğrulama, yol adı yarışlarının SQLite'ın incelediği baytları değiştirememesi için içeriği sabitlenmiş özel bir kopyayı doğrular. Varsayılan olarak bu geçici kopya, anlık görüntü deposunun yanında oluşturulur ve komut dönmeden önce kaldırılır. Hazırlama kökü ve üst dizin zinciri, diğer kullanıcıların bunu değiştirmesini engellemelidir. POSIX kökleri geçerli kullanıcıya ait olmalı ve grup ya da herkes tarafından yazılabilir olmamalıdır; `/tmp` gibi yapışkan üst dizinler, kullanıcıya ait alt dizinler için kabul edilir. Hazırlama alanını açığa çıkaran veya değiştirilebilir hâle getiren macOS ACL izinleri reddedilir. Windows kökleri ve üst dizinleri geçerli kullanıcıya veya güvenilir bir işletim sistemi sorumlusuna ait olmalı ve ACL'ler güvenilmeyen hazırlama erişimini engellemelidir. Salt okunur bir bağlama noktası veya ağ paylaşımı için eşdeğer şifreleme ve hedef denetimlerine sahip depolama alanında `--scratch <existing-private-directory>` geçirin.

Anlık görüntü oluşturma işlemi, veritabanı baytlarını hazırlamadan veya yayımlamadan önce depoya aynı sahiplik, ACL, üst dizin ve yol kimliği denetimlerini uygular.

Geri yükleme, doğrulamayı tekrarlar ve yalnızca yeni bir hedefe yazar. Mevcut bir hedefi, `-wal`, `-shm` veya `-journal` yan dosyasını reddeder ve canlı bir OpenClaw veritabanını hiçbir zaman yerinde değiştirmez. Hedef üst dizin, doğrulama çalışma alanıyla aynı yol güvenliği gereksinimlerine sahiptir. Geri yüklenen bir veritabanını etkinleştirmek, açıkça çevrimdışı gerçekleştirilen bir operatör adımı olarak kalır.

Anlık görüntü depoları yerel dizinlerdir. Zamanlama, yükleme, saklama, artımlı WAL paketleri, yük devretme ve önyükleme sırasında geri yükleme davranışı bilinçli olarak bu komutun kapsamı dışındadır.

## Neler yedeklenir?

`openclaw backup create`, yerel OpenClaw kurulumunuzdaki kaynakları planlar:

- Durum dizini (genellikle `~/.openclaw`)
- Etkin yapılandırma dosyasının yolu
- Durum dizininin dışında mevcut olduğunda çözümlenmiş `credentials/` dizini
- `--no-include-workspace` geçirmediğiniz sürece geçerli yapılandırmadan keşfedilen çalışma alanı dizinleri

Kimlik doğrulama profilleri ve aracı başına diğer çalışma zamanı durumları, durum dizini altındaki SQLite'ta (`agents/<agentId>/agent/openclaw-agent.sqlite`) bulunduğundan durum yedekleme girdisi tarafından otomatik olarak kapsanır.

`--only-config`; durum, kimlik bilgileri dizini ve çalışma alanı keşfini atlar ve yalnızca etkin yapılandırma dosyası yolunu arşivler.

OpenClaw, arşivi oluşturmadan önce yolları standartlaştırır: yapılandırma, kimlik bilgileri dizini veya bir çalışma alanı zaten durum dizininin içindeyse ayrı üst düzey yedekleme kaynakları olarak çoğaltılmaz. Eksik yollar atlanır.

Arşiv oluşturma sırasında OpenClaw, `tar` bunları okumadan önce bilinen canlı değişiklik yollarını hariç tutar. Bu, bir dosyanın kaydedilen boyutuyla eşzamanlı yazma işlemleri arasındaki yarışları önler. Filtre, yedeklenen her durum dizini altında duruma göre şu kuralları uygular:

| Duruma göre kapsam                            | Atlanan dosya son ekleri       |
| --------------------------------------------- | ------------------------------ |
| `sessions/**`                                | `.jsonl`, `.log`              |
| `agents/<agentId>/sessions/**`               | `.jsonl`, `.log`              |
| `cron/runs/**`                               | `.jsonl`, `.log`              |
| `logs/**`                                    | `.jsonl`, `.log`              |
| `delivery-queue/**`                          | `.json`, `.delivered`, `.tmp` |
| `session-delivery-queue/**`                  | `.json`, `.delivered`, `.tmp` |
| Yedeklenen durum dizini altındaki herhangi bir yol | `.sock`, `.pid`, `.tmp`       |

Bu kurallar, durum dizini dışındaki çalışma alanı dosyalarını filtrelemez. Ayrıca tabloyla eşleşen tamamlanmış döküm ve günlük dosyalarını da atlar; bu nedenle gerektiğinde bu kayıtları ayrı olarak saklayın. JSON sonucundaki `skippedVolatileCount`, bilinçli olarak kaç dosyanın atlandığını bildirir.

Durum dizini altındaki SQLite veritabanları, silinen sayfa kalıntılarının arşive girmemesi için SQLite'ın çevrimiçi yedekleme API'siyle yakalanır ve `VACUUM` ile çevrimdışı olarak sıkıştırılır; canlı WAL/SHM dosyaları kopyalanmaz. Kullanılamayan, sahip tarafından tanımlanmış SQLite yetenekleri gerektiren Plugin'e ait bir veritabanı, doğrudan dosya kopyalamaya geri dönmek yerine güvenli biçimde başarısız olur. Çalışma alanı yedeklemeleri aracılığıyla dahil edilen SQLite dosyaları, çalışma alanı dosyaları olarak kopyalanır ve sıkıştırma garantisi kapsamında değildir.

Durum dizininin `extensions/` ağacı altındaki yüklü Plugin kaynak ve manifest dosyaları dahil edilir, ancak iç içe `node_modules/` bağımlılık ağaçları yeniden oluşturulabilir kurulum yapıtları olarak atlanır. Bir arşivi geri yükledikten sonra, geri yüklenen bir Plugin eksik bağımlılıklar bildirirse `openclaw plugins update <id>` kullanın veya `openclaw plugins install <spec> --force` ile yeniden yükleyin.

Durum dizini altındaki yükleyici tarafından yönetilen ve yeniden oluşturulabilir çalışma zamanı kökleri de atlanır: `dev/`, `git/`, `npm/`, eski `npm-runtime/` ve `tools/`. Bunlar yetkili kullanıcı durumu yerine yönetilen çalışma kopyaları, paket ağaçları ve indirilmiş çalışma zamanları içerir; geri yüklemeden sonra ilgili çalışma zamanını veya Plugin'i yeniden yükleyin ya da güncelleyin. Bu köklerden birinin içindeki açıkça yapılandırılmış bir yapılandırma dosyası, kimlik bilgileri dizini veya çalışma alanı dahil edilmeye devam eder.

## Geçersiz yapılandırma davranışı

`openclaw backup`, kurtarma sırasında da yardımcı olabilmek için normal yapılandırma ön denetimini atlar. Çalışma alanı keşfi geçerli bir yapılandırmaya bağlıdır; bu nedenle yapılandırma dosyası mevcut ancak geçersizse ve çalışma alanı yedeklemesi hâlâ etkinse `openclaw backup create` hızla başarısız olur.

Bu durumda kısmi bir yedekleme için `--no-include-workspace` ile yeniden çalıştırın: çalışma alanı keşfini tamamen atlarken durum, yapılandırma ve harici kimlik bilgileri dizinini kapsamda tutar.

`--only-config`, çalışma alanı keşfi için yapılandırmayı ayrıştırmadığından yapılandırma hatalı biçimlendirilmiş olduğunda da çalışır.

## Boyut ve performans

OpenClaw, yerleşik bir azami yedekleme boyutu veya dosya başına boyut sınırı uygulamaz. Beş dakika boyunca veri üretmeyen bir arşiv yazma işlemi, süresiz olarak takılmak yerine başarısız olur ve kısmi geçici dosyasını kaldırır. Bunun dışındaki pratik sınırlar şunlardan kaynaklanır:

- Geçici arşiv yazımı ve nihai arşiv için kullanılabilir alan
- Büyük çalışma alanı ağaçlarını taramak ve bunları bir `.tar.gz` içine sıkıştırmak için gereken süre
- Arşivi `--verify` veya `openclaw backup verify` ile yeniden taramak için gereken süre
- Hedef dosya sistemi davranışı: OpenClaw, nihai arşiv yolunun devam eden bir kopyayı hiçbir zaman açığa çıkarmaması için üzerine yazmayan sabit bağlantı yayımlaması gerektirir; desteklenmeyen dosya sistemleri uygulanabilir bir hatayla başarısız olur

Yayımlamadan sonra nihai dizin dayanıklılığı doğrulaması başarısız olursa komut, eşzamanlı bir değişikliği silme riskine girmek yerine başarısızlık bildirir ancak tamamlanmış nihai girdiyi korur.

Büyük çalışma alanları genellikle arşiv boyutunun ana belirleyicisidir. Daha küçük ve hızlı bir yedekleme için `--no-include-workspace`, en küçük arşiv içinse `--only-config` kullanın.

## İlgili

- [CLI başvurusu](/tr/cli)
