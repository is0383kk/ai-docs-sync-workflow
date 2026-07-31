---
read_when:
    - Terminalden Workboard kartlarını incelemek veya oluşturmak istiyorsunuz
    - Workboard çalışanı çalıştırmalarını CLI üzerinden göndermek istiyorsunuz
    - Workboard CLI veya eğik çizgi komutu davranışında hata ayıklıyorsunuz
summary: '`openclaw workboard` kartları, gönderimi ve worker çalıştırmaları için CLI başvurusu'
title: Çalışma Panosu CLI'sı
x-i18n:
    generated_at: "2026-07-26T22:42:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 640260ea6f5959b3aee1cdce76f2501097bff79e9bf1741bdd9ff7a8b43e1a7f
    source_path: cli/workboard.md
    workflow: 16
---

`openclaw workboard`, paketle birlikte gelen [Workboard plugin](/tr/plugins/workboard) için terminal arayüzüdür. Bir operatörün kartları listelemesine, kart oluşturmasına, tek bir kartı incelemesine ve çalışan Gateway'den hazır işleri alt ajan işçi çalıştırmalarına dağıtmasını istemesine olanak tanır.

Komutu kullanmadan önce plugin'i etkinleştirin:

```bash
openclaw plugins enable workboard
openclaw gateway restart
```

## Kullanım

```bash
openclaw workboard list [--board <id>] [--status <status>] [--include-archived] [--json]
openclaw workboard create <title...> [--notes <text>] [--status <status>] [--priority <priority>] [--agent <id>] [--board <id>] [--labels <items>] [--json]
openclaw workboard show <id> [--json]
openclaw workboard move <id> --status <status> [--json]
openclaw workboard dispatch [--board <id>] [--max-starts <count>] [--admin] [--url <url>] [--token <token>] [--timeout <ms>] [--json]
```

Komut, pano ve Workboard ajan araçları tarafından kullanılan, plugin'e ait aynı SQLite veritabanını okur ve bu veritabanına yazar. Kart kimlikleri UUID'dir; kart kimliği kabul eden komutlar, belirsiz olmayan bir kimlik önekini de kabul eder (kompakt metin çıktısı ilk 8 karakteri gösterir).

Geçerli `status` değerleri: `triage`, `backlog`, `todo`, `scheduled`, `ready`, `running`, `review`, `blocked`, `done`. Geçerli `priority` değerleri: `low`, `normal`, `high`, `urgent`.

## `list`

```bash
openclaw workboard list
openclaw workboard list --board default --status ready
openclaw workboard list --json
```

Metin çıktısı kompakttır:

```text
7f4a2c10  ready     high    default agent-a  Eski işçi heartbeat'ini düzelt
```

Sütunlar sırasıyla kimlik öneki, durum, öncelik, pano kimliği, isteğe bağlı ajan kimliği ve başlıktır.

| Bayrak               | Amaç                                           |
| -------------------- | ---------------------------------------------- |
| `--board <id>`       | Sonuçları tek bir pano ad alanıyla sınırlar    |
| `--status <status>`  | Sonuçları tek bir Workboard durumuyla sınırlar |
| `--include-archived` | Arşivlenmiş kartları kompakt metin çıktısına dahil eder |
| `--json`             | Tam kart listesini makine tarafından okunabilir JSON olarak yazdırır |

CLI'nin `/workboard list` ile eşleşmesi için kompakt metin çıktısı varsayılan olarak arşivlenmiş kartları gizler. Bunları göstermek için `--include-archived` iletin. JSON çıktısı, mevcut otomasyonlar için arşivlenmiş kartlar dahil olmak üzere tam kart listesini her zaman korur.

## `create`

```bash
openclaw workboard create "Fix stale worker heartbeat" --priority high --labels bug,workboard
openclaw workboard create "Write Workboard docs" --status ready --agent docs-agent --board docs --notes "Cover CLI, slash command, dispatch, and SQLite state."
```

| Bayrak                  | Amaç                                      |
| ----------------------- | ----------------------------------------- |
| `--notes <text>`        | İlk kart notları                          |
| `--status <status>`     | İlk durum, varsayılan `todo`  |
| `--priority <priority>` | Öncelik, varsayılan `normal`    |
| `--agent <id>`          | Kartı bir ajana veya sahip kimliğine atar |
| `--board <id>`          | Kartı bir pano ad alanında depolar        |
| `--labels <items>`      | Virgülle ayrılmış etiketler                |
| `--json`                | Oluşturulan kartı makine tarafından okunabilir JSON olarak yazdırır |

`create`, doğrudan Workboard SQLite durumuna yazar. Kart, Control UI Workboard sekmesinde ve Workboard araçlarında hemen görünür.

## `show`

```bash
openclaw workboard show 7f4a2c10
openclaw workboard show 7f4a2c10 --json
```

Metin çıktısı, kompakt kart satırını ve notları yazdırır. JSON çıktısı; yürütme meta verileri, denemeler, yorumlar, bağlantılar, kanıtlar, yapıtlar, işçi günlükleri, protokol durumu, tanılamalar ve otomasyon meta verileri dahil olmak üzere tam kart kaydını döndürür.

JSON'daki kanıt durumları, işçi tarafından bildirilen sonuçlardır. `passed`, işçinin ekli komut veya kontrole ilişkin
öz değerlendirmesini kaydeder; bağımsız bir doğrulama
sonucu değildir.

## `move`

```bash
openclaw workboard move 7f4a2c10 --status review
openclaw workboard move 7f4a2c10 --status done --json
```

`move`, panoda bir kartı sürüklemekle aynı manuel operatör yolunu kullanarak kartın durumunu değiştirir. Tam kart kimliğini veya belirsiz olmayan bir öneki kabul eder. Etkin bağımlılık ve zamanlama bekletmeleri uygulanmaya devam eder. Operatörler, talep belirteci olmadan talep edilmiş bir kartı taşıyabilir; talep belirteçleri ajan aracı mutasyonlarıyla sınırlı kalır ve JSON çıktısından çıkarılır.

## `dispatch`

```bash
openclaw workboard dispatch
openclaw workboard dispatch --json
openclaw workboard dispatch --max-starts 10
openclaw workboard dispatch --admin
openclaw workboard dispatch --url http://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

`dispatch`, önce çalışan Gateway RPC yöntemi `workboard.cards.dispatch`'ü çağırır. Bu yöntem, pano dağıtım eylemiyle aynı alt ajan çalışma zamanını kullanır; böylece hazır kartlar, bağlantılı oturum anahtarlarına sahip, görevle izlenen işçi çalıştırmalarına dönüşür. `--max-starts`, eklemeli `workboard.cards.dispatchWithOptions` yöntemini kullanır; böylece eski bir Gateway, herhangi bir işçiyi başlatmadan önce seçeneği reddeder. Bayrağı kullanmadan önce yükseltmenin ardından Gateway'i yeniden başlatın. Atanmış bir ajanı olan kartlar, ajan kapsamlı alt ajan oturum anahtarlarını kullanır; atanmamış kartlar ise Gateway'in yapılandırılmış varsayılan ajanının korunması için kapsamsız bir alt ajan anahtarını korur.

Dağıtım döngüsü:

1. Bağımlılıkları hazır alt öğeleri `ready` durumuna yükseltir.
2. Süresi dolmuş talepleri veya zaman aşımına uğramış işçi çalıştırmalarını engeller.
3. Hazır kartlara dağıtım meta verilerini kaydeder.
4. Talep edilmemiş hazır kartlardan küçük bir grup seçer.
5. Seçilen her kartı dağıtıcı veya atanmış ajan adına talep eder.
6. Sınırlı kart bağlamı ve kart talep belirteciyle bir alt ajan işçi çalıştırması başlatır.
7. İşçi çalıştırma kimliğini, oturum anahtarını, Gateway görev defteri bildirdiğinde görev bağlantısını, yürütme durumunu ve işçi günlüğünü kartta depolar.

Seçim ihtiyatlıdır: bir dağıtım varsayılan olarak en fazla üç işçi başlatır, arşivlenmiş veya zaten talep edilmiş kartları atlar ve tek geçişte sahip ya da ajan başına yalnızca bir kart başlatır. Zaten etkin çalışan veya incelemedeki işlere ait kartlar daha sonraki bir dağıtıma bırakılır. Geçiş başına sınırı değiştirmek için pozitif bir tam sayı ile `--max-starts <count>` iletin; sahip başına tek kart kuralı uygulanmaya devam ettiğinden etkin başlatma sayısı daha düşük olabilir.

Bir kart talep edildikten sonra işçi başlatma işlemi başarısız olursa Workboard bu kartı engeller, talebi temizler ve hatayı kart yürütme ve işçi günlüğü meta verilerine kaydeder; böylece başarısız başlatmalar, kart sessizce kuyruğa döndürülmek yerine görünür kalır.

Açık bir Gateway hedefi verilmezse ve yerel Gateway kullanılamıyorsa veya Workboard dağıtım yöntemini henüz sunmuyorsa CLI, yerel Workboard durumu üzerinde yalnızca veri dağıtımına geri döner. Yalnızca veri dağıtımı yine de bağımlılıkları yükseltebilir, eski talepleri temizleyebilir ve zaman aşımına uğramış çalıştırmaları engelleyebilir; ancak işçi başlatmaz. Kimlik doğrulama, izin ve doğrulama hataları ile açık bir `--url` veya `--token` hedefi için oluşan hatalar, geri dönüşü tetiklemek yerine doğrudan bildirilir.

Metin çıktısı işçi başlatmalarını bildirir:

```text
dağıtım tamamlandı: başlatılan=2 hatalar=0
```

Geri dönüş çıktısı açıktır:

```text
gateway kullanılamıyor; yalnızca veri dağıtımı: yükseltilen=1 engellenen=0
```

JSON çıktısı dağıtım sonucunu içerir. Gateway destekli dağıtım `started` ve `startFailures` içerebilir; yalnızca veri geri dönüşü `gatewayUnavailable: true` içerir. Talep belirteçleri kart JSON çıktısından çıkarılır.

Panoda aynı dağıtım sonucu kısa bir özet olarak gösterilir; böylece operatör, kart ayrıntılarını açmadan kaç kartın başlatıldığını, yükseltildiğini, engellendiğini, yeniden talep edildiğini veya başarısız olduğunu görebilir.

## Eğik çizgi komutlarıyla eşdeğerlik

Komut destekli kanallar eşleşen eğik çizgi komutunu kullanabilir:

```text
/workboard list
/workboard show 7f4a2c10
/workboard create Fix stale worker heartbeat
/workboard move 7f4a2c10 --status review
/workboard dispatch
```

Eğik çizgi komutuyla dağıtım da Gateway alt ajan çalışma zamanını kullanır; bu nedenle pano ve CLI Gateway yoluyla aynı talep, işçi başlatma ve hata davranışını izler.

`/workboard list` ve `/workboard show`, yetkili komut gönderenler için okuma komutlarıdır. `/workboard create`, `/workboard move` ve `/workboard dispatch` pano durumunu değiştirir ve sohbet yüzeylerinde sahip durumu ya da `operator.write` veya `operator.admin` sahibi bir Gateway istemcisi gerektirir.

## İzinler

CLI dağıtım yolu normalde Gateway `operator.write` ve `operator.read` kapsamlarını ister. Çalışma alanına bağlı kartlar, tam olarak yapılandırılmış bir ajan çalışma alanında doğrudan çalışır; bir çalışma ağacı isteği, ana makinenin depo tarafından denetlenen kodu somutlaştırmasına izin vermek yerine bu dizinle sınırlandırılır. Seçilen işçi, tam olarak bu çalışma alanına yazılabilir ve paylaşılmayan Docker korumalı alan erişimine, istenen bağlamalar ve politikayla eşleşen canlı bir kapsayıcı karmasına sahip olmalı ve ana makineden kaçış yeteneğine sahip olmamalıdır. `operator.admin` kapsamını açıkça istemek, başka bir ana makine ödeme ağacına izin vermek ve normal yönetilen çalışma ağacı kurulumunu kullanmak için `--admin` iletin; bu kapsam istemci için onaylanmamışsa bağlantı başarısız olur. Salt okunur bir Gateway belirteci, okuma yöntemleri aracılığıyla Workboard verilerini inceleyebilir ancak kart oluşturamaz veya işçi dağıtamaz. Çalışma alanı sınırları, Workboard değiştirme iznine sahip çağıranlar için manuel kart taşımayı başka şekilde değiştirmez.

Yerel `list`, `create`, `show` ve `move` komutları, geçerli profil tarafından kullanılan yerel OpenClaw durum dizininde çalışır. Farklı bir durum kökü gerektiğinde üst düzey `openclaw` komutunda `--dev` veya `--profile <name>` kullanın.

## Sorun giderme

### Hiçbir kart görünmüyor

Plugin'in aynı profil ve durum kökü için etkinleştirildiğini doğrulayın:

```bash
openclaw plugins inspect workboard --runtime --json
```

Pano kartları gösteriyor ancak CLI göstermiyorsa her iki komutun da aynı `--dev` veya `--profile` ayarını kullandığını kontrol edin.

### Dağıtım yalnızca veri olduğunu bildiriyor

Gateway'i başlatın veya yeniden başlatın:

```bash
openclaw gateway restart
openclaw gateway status --deep
```

Ardından `openclaw workboard dispatch` işlemini yeniden deneyin. Yalnızca veri geri dönüşü, yerel durum temizliği için kullanışlıdır ancak işçi çalıştırmaları canlı bir Gateway gerektirir.

### Dağıtım hiçbir şey başlatmıyor

Etkin talebi olmayan en az bir `ready` kart bulunduğunu kontrol edin:

```bash
openclaw workboard list --status ready
```

Aynı sahip zaten çalışan veya incelemede olan bir işe sahipse kartlar da atlanabilir. Tamamlanan işi `done` durumuna taşıyın, eski talepleri Workboard araçları aracılığıyla serbest bırakın veya etkin işçi tamamlandıktan sonra dağıtımı tekrar çalıştırın.

## İlgili

- [Workboard plugin](/tr/plugins/workboard)
- [CLI başvurusu](/tr/cli)
- [Eğik çizgi komutları](/tr/tools/slash-commands)
- [Control UI](/tr/web/control-ui)
