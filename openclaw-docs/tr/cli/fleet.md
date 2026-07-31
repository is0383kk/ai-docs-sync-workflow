---
read_when:
    - Tek bir makinede birden fazla kiracı güven etki alanı barındırıyorsunuz
    - Filo hücreleri oluşturmanız, incelemeniz, yükseltmeniz veya kaldırmanız gerekiyor
summary: Kiracı başına yalıtılmış OpenClaw hücrelerini sağlama ve yönetmeye yönelik CLI başvurusu
title: Filo
x-i18n:
    generated_at: "2026-07-26T22:41:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: be589500e4715541f175caf0d5135a96baee4874e64c60c8b6f188ff1f70bc9f
    source_path: cli/fleet.md
    workflow: 16
---

# `openclaw fleet`

`openclaw fleet`, **cells** olarak adlandırılan eksiksiz OpenClaw örneklerini yönetir. Her cell'ın kendi Gateway'i, durumu, kimlik bilgileri, kanal hesapları, konteyneri ve yalnızca geri döngüye açık ana makine bağlantı noktası vardır. Her kiracı güven sınırı için bir cell kullanın; güvenilmeyen çok kiracılı bir sınır olarak tek bir paylaşılan Gateway kullanmayın.

Fleet **deneyseldir**. Komut adları, bayraklar, çıktı biçimleri ve konteyner profili, kullanımdan kaldırma geçiş süresi olmadan sürümler arasında değişebilir.

Fleet, Docker ve Podman'ı destekler. Varsayılan imaj `ghcr.io/openclaw/openclaw:latest` değeridir.

Fleet, Linux ve macOS ana makinelerinde test edilmiştir. Windows ana makineleri şu anda test edilmemiştir.

## Hızlı başlangıç

```bash
openclaw fleet create acme
openclaw fleet status acme
openclaw fleet list
```

`fleet create`, oluşturulan Gateway belirtecini cell URL'siyle birlikte bir kez yazdırır. Belirteci hemen saklayın, ardından her kiracının kanal hesaplarını o kiracının cell'ı içinde yapılandırın.

## Kiracı kimlikleri

Kiracı kimlikleri şununla eşleşmelidir:

```text
^[a-z0-9](?:[a-z0-9-]{0,38}[a-z0-9])?$
```

Bu, 1 ile 40 arasında küçük harfe, rakama ve iç tirelere izin verir. Bir kimlik harf veya rakamla başlayıp bitmelidir. Büyük harfler, alt çizgiler, eğik çizgiler, noktalar, boşluklar ve `../acme` gibi dizin geçişi dizeleri reddedilir.

Kimlik, konteyner adının bir parçası olur: `openclaw-cell-<tenant>`.

## `fleet create`

Bir cell oluşturup başlatın:

```bash
openclaw fleet create acme
```

Sabit bir bağlantı noktasında, başlatmadan bir Podman cell'ı oluşturun:

```bash
openclaw fleet create acme \
  --runtime podman \
  --port 19125 \
  --no-start
```

`--env` seçeneğini yineleyerek kiracıya özgü ortam değişkenlerini aktarın:

```bash
openclaw fleet create acme \
  --env TZ=America/Los_Angeles \
  --env OPENCLAW_DISABLE_BONJOUR=1
```

Ortam anahtarları harf, rakam ve alt çizgi kullanır ve rakamla başlayamaz. Fleet bunları korumalı bir çalışma zamanı ortam dosyası üzerinden aktardığı için değerler tek satırlı olmalıdır. Fleet, [Depolama ve konteyner düzeni](#storage-and-container-layout) altında listelenen, yönetilen konteyner yolu ve Gateway belirteci değişkenlerini geçersiz kılma girişimlerini reddeder.

### Oluşturma seçenekleri

| Seçenek                    | Varsayılan                               | Açıklama                                                                                    |
| ------------------------- | ------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `--image <ref>`           | `ghcr.io/openclaw/openclaw:latest`    | Cell için konteyner imajı.                                                                  |
| `--runtime <runtime>`     | `docker`                              | Konteyner CLI'si: `docker` veya `podman`.                                                           |
| `--port <number>`         | `19100` değerinden otomatik olarak ayrılır  | Geri döngü ana makine bağlantı noktası. Açıkça seçilen bir bağlantı noktası başka bir kayıtlı cell'a ait olmamalıdır.    |
| `--memory <value>`        | `2g`                                  | Docker/Podman söz diziminde konteyner bellek sınırı.                                                |
| `--cpus <value>`          | `2`                                   | Konteyner CPU sınırı.                                                                           |
| `--disk <size>`           | Yok                                  | Depolama arka ucu kotaları desteklediğinde konteynerin yazılabilir katmanını sınırlar.                     |
| `--network <mode>`        | `bridge`                              | Giden ağ modu: `bridge` veya `internal`.                                                 |
| `--pids-limit <number>`   | `512`                                 | Konteynerdeki azami işlem sayısı.                                                  |
| `--env <KEY=VALUE>`       | Yok                                  | Cell'a bir ortam değişkeni aktarır. Birden çok değer için yineleyin.                          |
| `--gateway-token <value>` | Rastgele 32 karakterli onaltılık belirteç | Bir belirteç oluşturmak yerine sağlanan Gateway belirtecini kullanır. Bkz. [Belirteç işleme](#token-handling). |
| `--no-start`              | Cell başlatılır                           | Konteyneri başlatmadan oluşturur.                                                      |
| `--json`                  | İnsan tarafından okunabilir çıktı                 | Makine tarafından okunabilir çıktı yazdırır.                                                                 |

Otomatik ayırma, `19100` veya üzerindeki ilk kullanılmayan kayıt defteri bağlantı noktasını seçer. Fleet, yinelenen kiracı kimliklerini ve başka bir cell'a zaten atanmış açık bağlantı noktalarını reddeder.

İmaj başvuruları, tek bir konteyner çalışma zamanı bağımsız değişkeni olarak aktarılır. Bir imajın Docker veya Podman seçeneği olarak yorumlanamaması için boş başvurular ve `-` ile başlayan değerler reddedilir.

Seçilen Docker veya Podman uç noktası yerel olmalıdır. Fleet, bir bağlantı noktası ayırmadan veya yerel durum oluşturmadan önce uzak Docker bağlamlarını, `DOCKER_HOST` uç noktalarını ve uzak Podman hizmetlerini reddeder. Uzak cell ana makineleri desteklenmez.

Fleet yeni bir cell başlattığında, oluşturma işlemi Gateway'in `/healthz` isteğine yanıt vermesi için yaklaşık bir dakika bekler. Cell sağlıklı duruma gelmezse Fleet, `fleet status`, `fleet logs` veya açıkça kaldırma işlemi için konteynerini ve kayıt defteri satırını olduğu gibi bırakır. `--no-start` bu sağlık denetimini atlar. Sağlıksız yeni bir cell'ın oluşturulan Gateway belirteci kaybolmaz; konteyner ortamında (`docker|podman inspect`) kalır ve cell henüz hiç trafik sunmadığından, `fleet rm --force` sonrasında yeniden oluşturma her zaman güvenli bir alternatiftir.

### Özetle sabitleme

Oluşturma ve yükseltme, `--image ghcr.io/openclaw/openclaw@sha256:<digest>` gibi özetle sabitlenmiş imaj başvurularını kabul eder. Fleet, imaj başvurusunu olduğu gibi Docker veya Podman'a aktarır; bu da bir operatörün cell'ı değişken bir etiket yerine değişmez imaj baytlarında tutmasını sağlar.

Oluşturma sonucu; kiracı kimliğini, konteyner adını, ana makine bağlantı noktasını, Gateway belirtecini ve yerel URL'yi içerir. Sonuç, JSON çıktısında bile belirteci içerdiğinden gizli bilgi taşıyan bir değer olarak değerlendirilmelidir.

### Disk sınırları

`--disk` yalnızca konteynerin yazılabilir katmanını sınırlar. Bağlama yoluyla monte edilen kiracı başına durum ve kimlik doğrulama dizinleri ana makine depolaması olarak kalır; bu dizinler için de kesin bir sınır gerektiğinde ana makine dosya sistemi proje kotalarını kullanın.

| Çalışma zamanı/depolama arka ucu | `--disk` desteği                                                             |
| ----------------------- | ---------------------------------------------------------------------------- |
| XFS üzerinde Docker overlay2  | XFS `pquota` bağlama seçeneğini gerektirir.                                      |
| Docker btrfs veya zfs     | Depolama sürücüsü tarafından desteklenir.                                             |
| Podman overlay          | XFS destekli depolama gerektirir.                                                |
| Diğer arka uçlar          | Konteyner oluşturma işlemi, arka plan programı hatası ve Fleet'in arka uç yönlendirmesiyle başarısız olur. |

### Çıkış politikası

| Mod       | Docker                                                                                                | Podman                                                                              |
| ---------- | ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `bridge`   | Desteklenir; giden trafik varsayılan olarak kısıtlanmaz.                                                | Desteklenir; giden trafik varsayılan olarak kısıtlanmaz.                              |
| `internal` | Docker, dahili bir ağda yayımlanan geri döngü Gateway bağlantı noktasını korumadığı için reddedilir. | Desteklenir; giden trafik engellenirken geri döngü Gateway'i yayımlanmış olarak kalır. |

Docker için köprü modunu koruyun ve giden trafik politikasını `DOCKER-USER` zinciri gibi ana makine güvenlik duvarı kurallarıyla uygulayın.

## `fleet list`

Cell'ları kiracı kimliği sırasıyla listeleyin:

```bash
openclaw fleet list
openclaw fleet ls
openclaw fleet list --json
```

Tablo şunları içerir:

| Sütun    | Anlam                                                                                                                                                                                                                                                                               |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tenant`  | Kiracı kimliği.                                                                                                                                                                                                                                                                            |
| `state`   | Docker veya Podman incelemesinden alınan canlı konteyner durumu. `unknown`, çalışma zamanının kullanılamadığı veya cell'ın adına sahip bir konteyner bulunduğu hâlde Fleet sahiplik etiketlerinin kayıt defteri kaydıyla eşleşmediği anlamına gelir (çakışma veya kurcalama işareti — işlem yapmadan önce elle inceleyin). |
| `port`    | Cell Gateway'ine eşlenen geri döngü ana makine bağlantı noktası.                                                                                                                                                                                                                                        |
| `image`   | Kaydedilen konteyner imajı.                                                                                                                                                                                                                                                             |
| `created` | Cell oluşturma zamanı.                                                                                                                                                                                                                                                                   |

Docker veya Podman kullanılamadığında kayıt defteri satırları görünür kalır; yalnızca canlı durum `unknown` olur.

## `fleet status`

Bir cell'ı inceleyin:

```bash
openclaw fleet status acme
openclaw fleet status acme --json
```

Durum; Fleet kayıt defteri satırını, canlı konteyner incelemesini ve aşağıdaki adrese yönelik kısa, en iyi çaba esaslı bir isteği birleştirir:

```text
http://127.0.0.1:<host-port>/healthz
```

Sağlık sonucu `ok`, `failed` veya `skipped` olur. `/healthz`, Gateway'in çalıştığını kanıtlar; yapılandırılmış her kanalın veya Plugin'in tam olarak hazır olduğunu kanıtlamaz. Denetlenecek kullanılabilir bir yerel uç nokta olmadığında yoklama atlanır.

## `fleet logs`

Bir cell'ın konteyner günlüklerini doğrudan terminale aktarın:

```bash
openclaw fleet logs acme
openclaw fleet logs acme --follow
openclaw fleet logs acme --tail 200
openclaw fleet logs acme --since 10m
```

Fleet, herhangi bir günlüğü okumadan önce kayıtlı konteynerin sahiplik etiketlerini doğrular; bu nedenle beklenen cell adını kullanan yabancı bir konteyneri reddeder. Akış, incelenen konteyner kimliğine sabitlenir; böylece eş zamanlı bir değiştirme işlemi akışı daha yeni bir nesle yönlendiremez. Operatörün durdurma işlemini komut hatası olarak değerlendirmeden `--follow` akışını sonlandırmak için Ctrl-C tuşlarına basın. Günlük çıktısı, terminale herhangi bir şey ulaşmadan önce cell'ın geçerli Gateway belirtecini `<redacted>` ile değiştiren bir gizleme filtresinden geçirilir.

Konteyner günlükleri ham bir stdout/stderr akışı olduğundan `fleet logs` için `--json` modu yoktur. Betiklerde çıktıyı `--tail` ile sınırlayın ve standart kabuk yönlendirmesi veya işlem hatlarını kullanın.

## `fleet start`, `fleet stop` ve `fleet restart`

Mevcut bir hücreyi kayıtlı çalışma zamanı ile denetleyin:

```bash
openclaw fleet start acme
openclaw fleet stop acme
openclaw fleet restart acme
```

Bu komutlar kayıtlı kapsayıcı adı üzerinde işlem yapar. Kiracı bilinmiyorsa veya kayıtlı çalışma zamanı işlemi gerçekleştiremiyorsa başarısız olurlar.

## `fleet upgrade`

Kayıtlı imajı yeniden çekin ve hücre kapsayıcısını değiştirin:

```bash
openclaw fleet upgrade acme
```

Hücreyi başka bir imaja taşıyın:

```bash
openclaw fleet upgrade acme --image ghcr.io/openclaw/openclaw:<version>
```

Yükseltme, hedef imajı çeker; mevcut kapsayıcıyı ve hücre başına ağı inceler; kapsayıcıyı durdurup kaldırır; ardından yeniden oluşturup başlatır. Yeni kapsayıcı aynı ana makine bağlantı noktasını, veri dizinlerini, hücre başına köprü ağını, çalışma zamanı profilini, kaynak sınırlarını, yeniden başlatma politikasını, Fleet tarafından yönetilen ortamı ve başlangıçta `--env` ile sağlanan değerleri korur. Bağlanan durum, kapsayıcı değişiminden etkilenmez; imajın varsayılan ortamı hedef imajla birlikte değişebilir.

Değişiklik yalnızca Gateway, hücrenin geri döngü bağlantı noktasında `/healthz` isteğine yanıt verdikten sonra kesinleştirilir; bu, resmî compose dosyasının kullandığı sistem durumu sözleşmesiyle aynıdır. Çıkan, sürekli çöken veya yaklaşık bir dakika içinde sağlıklı duruma geçemeyen yeni kapsayıcı kaldırılır ve önceki kapsayıcı geri yüklenir; böylece bozuk bir imaj çalışan bir hücreyi devre dışı bırakmaz.

Gateway belirteci kasıtlı olarak fleet kayıt defterinde saklanmaz. Fleet, eski kapsayıcıyı kaldırmadan önce kapsayıcının ortamını okur ve `OPENCLAW_GATEWAY_TOKEN` değerini yeni kapsayıcıya aktarır. Belirteç denetiminizdeki başka hiçbir yerde bulunmuyorsa yükseltmeden önce eski kapsayıcıyı elle kaldırmayın.

## `fleet backup` ve `fleet restore`

Durdurulmuş bir hücreyi yedekleyin:

```bash
openclaw fleet stop acme
openclaw fleet backup acme --out ./acme.tgz
```

Bu arşivi kayıtlı hücreye geri yükleyin:

```bash
openclaw fleet restore acme --from ./acme.tgz
```

Bunlar ana makine operatörü ayrıcalıkları gerektiren komutlardır. Arşivler kiracı durumunu ve kimlik doğrulama gizli bilgilerini içerir, `0600` moduyla oluşturulur ve kimlik bilgileri gibi saklanmalıdır. Yedekleme, SQLite durumunun tutarlı biçimde yakalanabilmesi için çalışan bir hücreyi reddeder. Geri yükleme, `--force` sağlanmadıkça çalışan bir hücreyi reddeder; yalnızca ilgili kiracının durumunu değiştirir, Gateway belirtecini yeniler ve yeni belirteci bir kez yazdırır. Fleet tek seferde bir kiracıyı yedekler; tüm kiracıları yedeklemek ayrı bir operatör işlemidir.

Geri yükleme için mevcut ve durdurulmuş bir kapsayıcı gerekir; çünkü incelenen çalışma zamanı profili yeni kapsayıcının sınırlarını, kullanıcı eşlemesini, ortam kaynağını ve imajını sağlar. Kayıtlı kapsayıcı harici bir yöntemle kaldırıldıysa önce `--purge-data` olmadan `fleet rm <tenant> --force` komutunu çalıştırın, hücreyi amaçlanan imaj ve `--no-start` ile yeniden oluşturun, ardından geri yüklemeyi tekrar deneyin. İlk kaldırma işlemi her iki kiracı veri dizinini de korur.

Her iki komut da arşivlenen veya çıkarılan dosya verilerini sınırlamak için `--max-bytes <bytes>` kabul eder ve meta veri içeren arşiv bombalarının ana makine inode'larını tüketmesini önlemek ve kabul edilen her yedeğin geri yüklenebilir kalmasını sağlamak için arşiv yolu segmentlerine aynı sabit bir milyonluk bütçeyi uygular. Yedekleme `--out <path>` kabul eder ve her iki komut da `--json` destekler.

Arşivler yalnızca normal dosyaları ve dizinleri içerir. Yedekleme sembolik bağlantıları, sabit bağlantıları, yuvaları veya aygıt düğümlerini asla izlemez ya da saklamaz; atlanan öğelerin sayısı sonuçta bildirilir. Geri yükleme, başka türde giriş içeren arşivleri reddeder. Çalışma alanı `node_modules` gibi yeniden oluşturulabilir sembolik bağlantı ağaçları, geri yüklemeden sonra hücre içinde yeniden kurulmalıdır.

## `fleet doctor`

Çalışma zamanı veya dosya sistemi durumunu değiştirmeden her hücreyi ya da tek bir kiracıyı denetleyin:

```bash
openclaw fleet doctor
openclaw fleet doctor acme --json
```

Doctor; çalışma zamanının yerelliğini, sahiplik etiketlerini, sistem durumunu, sıkılaştırmayı, kaynak sınırlarını, geri döngü bağlantı noktası bağlamasını, belirteç varlığını, ağ sahipliğini ve çıkış modunu ve özel durum dizini izinlerini denetler. Uyarılar durdurulmuş hücreleri veya sahiplik farklılıklarını açıklar; başarısız olan herhangi bir bulgu, işlem çıkış kodunu sıfırdan farklı bir değere ayarlar.

## `fleet rm`

Kiracı verilerini koruyarak durdurulmuş bir hücreyi çalışma zamanından ve kayıt defterinden kaldırın:

```bash
openclaw fleet rm acme
```

Çalışan bir kapsayıcı için `--force` gerekir:

```bash
openclaw fleet rm acme --force
```

Hücre verilerini de kalıcı olarak kaldırın:

```bash
openclaw fleet rm acme --purge-data --force
```

Fleet, ayrılmış köprü ağını kaldırmadan önce hücre kapsayıcısını kaldırır. `--purge-data`, `--force` gerektirir. Fleet, özyinelemeli silmeden önce hem Fleet'e ait kökleri hem de kiracı başına iki dizini çözümler. Her hedef tam olarak beklenen kiracı yaprağı olmalı, kesin biçimde kendi kökünün içinde bulunmalı ve sembolik bağlantı olmamalıdır. Bu kapsama denetimleri, bozuk bir kayıt defteri yolunun veya kiracılar arası sembolik bağlantının silme işlemini başka bir konuma yönlendirmesini önler.

Tam temizleme, tam olarak beklenen bir kiracı dizini zaten yoksa yeniden denenebilir. Bu, daha sonraki bir çağrının hâlâ mevcut dizinlere ilişkin yol denetimlerini gevşetmeden, kısmi bir dosya sistemi hatasından sonra temizliği tamamlamasına olanak tanır.

## Depolama ve kapsayıcı düzeni

Hücre durumu ve kimlik doğrulama profili şifreleme anahtarları, etkin OpenClaw durum dizini altında kiracı başına ayrı ana makine yolları kullanır:

```text
<state-dir>/fleet/cells/<tenant>/
<state-dir>/fleet/auth-profile-secrets/<tenant>/
```

İlk dizin `/home/node/.openclaw` konumuna bağlanır. İkinci dizin, resmî Docker kurulumunun şifreleme anahtarı bağlamasıyla eşleşecek şekilde `/home/node/.config/openclaw` konumuna bağlanır. Bu nedenle şifreleme anahtarı normal durum bağlamasının altında açığa çıkmaz ve yalnızca hücre durumu dizini yedeklendiğinde veya paylaşıldığında buna dâhil edilmez. Her iki dizin de normal kaldırma ve yükseltme işlemlerinden etkilenmez; `fleet rm --purge-data --force`, ayrı kapsama denetimlerinden sonra ikisini de siler.

Fleet ilk başlatmadan önce hücre yapılandırmasını `gateway.mode=local`, belirteç kimlik doğrulaması, LAN kapsayıcı bağlaması ve ayrılan ana makine bağlantı noktası için Control UI kaynaklarıyla başlatır. Belirteç değeri bu yapılandırmaya yazılmaz; kapsayıcı ortamında kalır.

Fleet, resmî imajın kapsayıcı yollarını şu ortam değerleriyle sabitler:

| Değişken                 | Kapsayıcı değeri                      |
| ------------------------ | ------------------------------------ |
| `HOME`                   | `/home/node`                         |
| `OPENCLAW_HOME`          | `/home/node`                         |
| `OPENCLAW_STATE_DIR`     | `/home/node/.openclaw`               |
| `OPENCLAW_CONFIG_PATH`   | `/home/node/.openclaw/openclaw.json` |
| `OPENCLAW_WORKSPACE_DIR` | `/home/node/.openclaw/workspace`     |
| `OPENCLAW_GATEWAY_TOKEN` | Oluşturulan veya sağlanan hücre belirteci     |

Resmî imaj varsayılan olarak UID 1000'e sahip root olmayan `node` kullanıcısını kullanır. Fleet, özel `0700` bağlama bağlantılarını herkesçe erişilebilir hâle getirmeden yazılabilir tutar. Root yetkili Docker, hücreyi çağrıyı yapan root olmayan UID ve GID ile çalıştırır; root olmayan Docker ise kapsayıcı UID 0'ı kullanır ve bu değer daemon'ın kullanıcı ad alanında çağrıyı yapan ayrıcalıksız ana makine kullanıcısına eşlenir. Podman, çağrıyı yapan UID ve GID ile `keep-id` kullanır. Fleet'in kendisi root yetkili bir çalışma zamanına karşı root olarak çalıştığında imaj kullanıcısını korur ve ilk bağlama dosyalarını UID/GID 1000'e atar.

SELinux ana makinelerinde Docker ve Podman bağlamaları özel bir `:Z` yeniden etiketlemesi alır. Hücre verilerini geri yükler veya başka bir konuma taşırsanız bağlanan yolları etkin kapsayıcı kullanıcısının yazabileceği durumda tutun. Profil root olmayan kullanıma uygundur ancak Docker veya Podman ana makinede root olmayan çalışma için önceden yapılandırılmış olmalıdır; Fleet, root yetkili bir daemon'ı root olmayan bir daemon'a dönüştürmez.

## Güvenlik profili

Fleet her hücreye aşağıdaki profili uygular:

| Denetim              | Uygulanan profil                                      | Neden                                                                                    |
| -------------------- | ---------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Linux yetenekleri   | `--cap-drop=ALL`                                     | Gateway bir Node.js işlemidir ve ek Linux yeteneklerine ihtiyaç duymaz.                |
| Ayrıcalık yükseltme | `--security-opt no-new-privileges`                   | İşlemlerin setuid veya setgid ikili dosyaları aracılığıyla ayrıcalık kazanmasını önler.          |
| Başlatma işlemi         | `--init`                                             | Alt işlemleri sonlandırır ve kapsayıcı yaşam döngüsü sinyallerini iletir.                   |
| İşlem sınırı        | Varsayılan olarak `--pids-limit 512`                        | Çatallanma ve işlem tükenmesini sınırlar.                                                    |
| Bellek sınırı         | Varsayılan olarak `--memory 2g`                             | Hücrenin bellek kullanımını sınırlar.                                                                |
| CPU sınırı            | Varsayılan olarak `--cpus 2`                                | Hücrenin CPU kullanımını sınırlar.                                                                   |
| Yazılabilir katman diski  | İsteğe bağlı `--disk`                                    | Çalışma zamanı depolama arka ucu kotaları desteklediğinde kapsayıcı katmanını sınırlar.           |
| Yeniden başlatma politikası       | `--restart unless-stopped`                           | Kasıtlı bir durdurmayı geçersiz kılmadan başarısız bir hücreyi yeniden başlatır.                         |
| Ana makinede yayımlama      | Yalnızca `127.0.0.1:<host-port>:18789`                   | Gateway'i joker karakterli ana makine arayüzlerinden uzak tutar.                                        |
| Hücre ağı         | Hücre başına bir köprü veya Podman iç ağı       | Kapsayıcı IP trafiğini ayırır ve isteğe bağlı olarak Podman dış çıkışını engeller.           |
| Kapsayıcı kimliği   | Ana makineyle eşleşen kullanıcı eşlemesi                            | Özel bağlamaları herkese erişim vermeden yazılabilir tutar.                      |
| Kalıcı durum     | Hücre başına bağlamalar; paylaşılan durum bağlaması yok               | Kiracı yapılandırmasını, kimlik bilgilerini, oturumlarını ve çalışma alanlarını ilgili kiracının veri ağacında tutar. |
| Kapsayıcı komutu    | `node dist/index.js gateway --bind lan --port 18789` | Yalnızca geri döngüye açık ana makine bağlantı noktası eşlemesinin erişebilmesi için kapsayıcı ağını dinler.  |

Fleet hiçbir zaman `/var/run/docker.sock` bağlamaz, `--privileged` veya ana makine ağını kullanmaz ya da yetenek eklemez. Hücre başına köprü, hücreler arası bir ayırma sınırıdır; dışa giden trafik için güvenlik duvarı değildir: hücreler sağlayıcılar ve kanallar için gereken ağ çıkışını korur. Geri döngü bağlantı noktasının önüne dağıtımınıza uygun bir proxy, SSH tüneli veya tailnet yapılandırması yerleştirin. `http://127.0.0.1:<port>` doğrudan yalnızca Fleet ana makinesinden erişilebilir.

Bu profil kiracı kapsayıcılarını birbirinden ayırır ancak kiracıları Fleet operatöründen, kapsayıcı çalışma zamanı yöneticisinden veya güvenliği ihlal edilmiş bir ana makineden korumaz. Eksiksiz güven modeli ve daha güçlü yalıtım seçenekleri için [Çok kiracılı barındırma](/tr/gateway/multi-tenant-hosting) bölümüne bakın.

## Belirteç işleme

Varsayılan olarak `fleet create`, kriptografik olarak rastgele 32 karakterli onaltılık bir Gateway belirteci oluşturur ve bunu oluşturma sonucunda bir kez yazdırır. Belirteci onaylı gizli bilgi yöneticinizde saklayın ve oluşturma çıktısının günlüklere kaydedilmesini önleyin.

`--gateway-token`, özel bir belirteci yerel işlem bağımsız değişkenlerine yerleştirir; bu belirteç kabuk geçmişinde tutulabilir veya işlem listelerinde görülebilir. Mevcut bir gizli bilgi yönetimi iş akışı sağlanan bir değer gerektirmediği sürece oluşturulan belirteci tercih edin.

Belirteç ve `--env` ile aktarılan her değer kapsayıcı ortamında bulunur. Fleet bunları kısa ömürlü, `0600` modundaki bir ortam dosyasına yazar, Docker veya Podman'a yalnızca bu dosyanın yolunu aktarır ve çalışma zamanı komutu tamamlandıktan sonra dosyayı kaldırır. `openclaw fleet create --gateway-token ...` veya `--env KEY=VALUE` içinde açıkça yazılan değerler, dış `openclaw` işleminin bağımsız değişkenlerinde ve kabuk geçmişinde yine de görülebilir.

Konteyner ortam değerleri güvenilen ana makine operatöründen gizlenmez: Docker veya Podman yöneticileri bunları konteyner incelemesiyle okuyabilir. Fleet'in "bir kez gösterilir" notu, ana makine yöneticisine karşı korumayı değil, normal CLI çıktısını açıklar.

## İlgili

- [Çok kiracılı barındırma](/tr/gateway/multi-tenant-hosting)
- [Docker](/tr/install/docker)
- [Podman](/tr/install/podman)
- [Gateway güvenliği](/tr/gateway/security)
