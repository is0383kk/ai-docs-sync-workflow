---
read_when:
    - Gateway'i CLI'dan çalıştırma (geliştirme veya sunucular)
    - Gateway kimlik doğrulaması, bağlama modları ve bağlantı sorunlarını ayıklama
    - Bonjour aracılığıyla Gateway'leri keşfetme (yerel + geniş alan DNS-SD)
    - Harici bir Gateway süreç denetleyicisini entegre etme
sidebarTitle: Gateway
summary: OpenClaw Gateway CLI (`openclaw gateway`) — Gateway'leri çalıştırma, sorgulama ve keşfetme
title: Gateway
x-i18n:
    generated_at: "2026-07-26T23:15:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0188d7c79571ebf8f350295775625533a83cb2eb909bcc8763e8ce81806d2214
    source_path: cli/gateway.md
    workflow: 16
---

Gateway, OpenClaw'ın WebSocket sunucusudur (kanallar, Node'lar, oturumlar, hook'lar). Aşağıdaki tüm alt komutlar `openclaw gateway ...` altında yer alır.

<CardGroup cols={3}>
  <Card title="Bonjour keşfi" href="/tr/gateway/bonjour">
    Yerel mDNS + geniş alan DNS-SD kurulumu.
  </Card>
  <Card title="Keşfe genel bakış" href="/tr/gateway/discovery">
    OpenClaw'ın Gateway'leri nasıl duyurduğu ve bulduğu.
  </Card>
  <Card title="Yapılandırma" href="/tr/gateway/configuration">
    Üst düzey Gateway yapılandırma anahtarları.
  </Card>
</CardGroup>

## Gateway'i çalıştırma

```bash
openclaw gateway
openclaw gateway run   # eşdeğer, açık biçim
```

<AccordionGroup>
  <Accordion title="Başlangıç davranışı">
    - `~/.openclaw/openclaw.json` içinde `gateway.mode=local` ayarlanmadıkça başlatmayı reddeder. Geçici/geliştirme çalıştırmaları için `--allow-unconfigured` kullanın; bu seçenek, yapılandırmayı yazmadan veya onarmadan korumayı atlar.
    - Başlangıçta onarılabilir geçersiz bir yapılandırma bulunursa etkileşimli terminal, `openclaw doctor --fix` komutunu çalıştırmayı önerir ve onaydan sonra başlatmayı bir kez yeniden dener. Etkileşimsiz çalıştırmalar hiçbir zaman otomatik onarım yapmaz; bunun yerine komutu yazdırır. Onarılan yapılandırma hâlâ geçersizse başlatma durdurulmuş olarak kalır.
    - `openclaw onboard --mode local` ve `openclaw setup`, `gateway.mode=local` değerini yazar. Yapılandırma dosyası mevcut ancak `gateway.mode` eksikse bu durum hasarlı/üzerine yazılmış yapılandırma olarak değerlendirilir ve Gateway sizin için `local` değerini tahmin etmeyi reddeder — ilk kurulumu yeniden çalıştırın, anahtarı elle ayarlayın veya `--allow-unconfigured` geçirin.
    - Kimlik doğrulama olmadan loopback dışına bağlanma engellenir.
    - `--bind` değerleri `lan`, `tailnet` ve `custom` günümüzde yalnızca IPv4 yolları üzerinden çözümlenir; yalnızca IPv6 kullanan kendi sunucunuzu getirin kurulumlarında Gateway'in önünde bir IPv4 sidecar'ı veya proxy gerekir.
    - `SIGUSR1`, yetkilendirildiğinde süreç içi yeniden başlatmayı tetikler. `commands.restart` (varsayılan: etkin), dışarıdan gönderilen `SIGUSR1` işlemlerini denetler; elle işletim sistemi sinyaliyle yeniden başlatmaları engellemek için bunu `false` olarak ayarlayın. Agent'a yönelik `gateway` aracı salt okunurdur; agent'lar yeniden başlatmayı insan onaylı `openclaw` yetkilendirme aracı üzerinden talep eder.
    - `SIGINT`/`SIGTERM` süreci durdurur ancak özel terminal durumunu geri yüklemez — CLI'yi bir TUI veya ham mod girişiyle sarmalıyorsanız çıkıştan önce terminali kendiniz geri yükleyin.

  </Accordion>
</AccordionGroup>

### Seçenekler

<ParamField path="--port <port>" type="number">
  WebSocket bağlantı noktası (yapılandırma/ortamdan varsayılan; genellikle `18789`).
</ParamField>
<ParamField path="--bind <mode>" type="string">
  Bağlama modu: `loopback` (varsayılan), `lan`, `tailnet`, `auto`, `custom`.
</ParamField>
<ParamField path="--token <token>" type="string">
  `connect.params.auth.token` için paylaşılan token. Ayarlandığında varsayılan olarak `OPENCLAW_GATEWAY_TOKEN` kullanılır.
</ParamField>
<ParamField path="--auth <mode>" type="string">
  Kimlik doğrulama modu: `none`, `token`, `password`, `trusted-proxy`.
</ParamField>
<ParamField path="--password <password>" type="string">
  `--auth password` için parola.
</ParamField>
<ParamField path="--password-file <path>" type="string">
  Gateway parolasını bir dosyadan okuyun.
</ParamField>
<ParamField path="--tailscale <mode>" type="string">
  Tailscale erişimi: `off`, `serve`, `funnel`.
</ParamField>
<ParamField path="--tailscale-reset-on-exit" type="boolean">
  Kapatma sırasında Tailscale serve/funnel yapılandırmasını sıfırlayın.
</ParamField>
<ParamField path="--allow-unconfigured" type="boolean">
  `gateway.mode=local` zorunluluğunu uygulamadan başlatın. Yalnızca geçici/geliştirme önyüklemesi içindir; yapılandırmayı kalıcılaştırmaz veya onarmaz.
</ParamField>
<ParamField path="--dev" type="boolean">
  Eksikse bir geliştirme yapılandırması + çalışma alanı oluşturun (`BOOTSTRAP.md` atlanır).
</ParamField>
<ParamField path="--dev-ambient-channels" type="boolean">
  Bir geliştirme Gateway'inin ortam değişkenlerinden kanalları otomatik yapılandırmasına izin verin. `--dev` gerektirir.
</ParamField>
<ParamField path="--reset" type="boolean">
  Geliştirme yapılandırmasını, kimlik bilgilerini, oturumları ve çalışma alanını sıfırlayın. `--dev` gerektirir.
</ParamField>
<ParamField path="--force" type="boolean">
  Başlatmadan önce hedef bağlantı noktasındaki mevcut tüm dinleyicileri sonlandırın. Etkileşimsiz bir kabukta bu seçenek, doğrulanmış bir Gateway dinleyicisini sonlandırmayı reddeder; bunun yerine `--dev` veya boş bir bağlantı noktasına sahip yalıtılmış bir `--profile` kullanın.
</ParamField>
<ParamField path="--verbose" type="boolean">
  stdout/stderr'e ayrıntılı günlük kaydı.
</ParamField>
<ParamField path="--cli-backend-logs" type="boolean">
  Konsolda yalnızca CLI arka uç günlüklerini gösterin (stdout/stderr'i de etkinleştirir).
</ParamField>
<ParamField path="--ws-log <style>" type="string" default="auto">
  WebSocket günlük stili: `auto`, `full`, `compact`.
</ParamField>
<ParamField path="--compact" type="boolean">
  `--ws-log compact` için takma ad.
</ParamField>
<ParamField path="--raw-stream" type="boolean">
  Ham model akışı olaylarını JSONL'ye kaydedin.
</ParamField>
<ParamField path="--raw-stream-path <path>" type="string">
  Ham akış JSONL yolu.
</ParamField>

`--claude-cli-logs`, `--cli-backend-logs` için kullanımdan kaldırılmış bir takma addır.

`--bind custom` için `gateway.customBindHost` değerini bir IPv4 adresine ayarlayın. `127.0.0.1` veya `0.0.0.0` dışındaki tüm adresler, aynı ana makinedeki istemciler için aynı bağlantı noktasında `127.0.0.1` değerini de gerektirir; dinleyicilerden herhangi biri bağlanamazsa başlangıç başarısız olur. Joker karakterli `0.0.0.0`, ayrıca zorunlu bir takma ad eklemez. Yalnızca IPv6 kullanan kendi sunucunuzu getirin kurulumlarında Gateway'in önünde bir IPv4 sidecar'ı veya proxy gerekir.

## Gateway'i yeniden başlatma

```bash
openclaw gateway restart
openclaw gateway restart --safe
openclaw gateway restart --safe --skip-deferral
openclaw gateway restart --force
openclaw gateway restart --wait 30s
```

`--safe`, çalışan Gateway'den etkin işler için ön kontrol yapmasını ve bu işler tamamlandıktan sonra birleştirilmiş tek bir yeniden başlatma planlamasını ister. Bekleme süresi 5 dakika ile sınırlıdır; süre dolduğunda yeniden başlatma zorlanır. `--safe`, `--force` veya `--wait` ile birlikte kullanılamaz.

`--skip-deferral`, güvenli yeniden başlatmada etkin iş erteleme denetimini atlar; böylece bildirilen engelleyiciler olsa bile Gateway hemen yeniden başlatılır. `--safe` gerektirir — erteleme kontrolden çıkmış bir görevde takılı kaldığında kullanın.

`--wait <duration>`, normal (güvenli olmayan) yeniden başlatma için tamamlanmayı bekleme süresini geçersiz kılar. Birimsiz milisaniyeleri veya `ms`, `s`, `m`, `h`, `d` birim soneklerini kabul eder (ör. `30s`, `5m`, `1h30m`); `--wait 0` süresiz bekler. `--force` veya `--safe` ile uyumlu değildir.

`--force`, etkin işlerin tamamlanmasını beklemeyi atlar ve hemen yeniden başlatır. Normal `restart` (bayraksız), mevcut hizmet yöneticisi yeniden başlatma davranışını korur.

<Warning>
Satır içi `--password`, yerel süreç listelerinde açığa çıkabilir. `--password-file`, ortam değişkeni veya SecretRef destekli `gateway.auth.password` tercih edin.
</Warning>

### Harici gözetmenler

`OPENCLAW_SUPERVISOR_MODE=external` değerini yalnızca Gateway yaşam döngüsünün başka bir süreç yöneticisi tarafından yönetildiği durumlarda ayarlayın. Bu modda:

- `openclaw gateway restart`, launchd, systemd veya Task Scheduler yerine doğrulanmış çalışan Gateway'i hedeflerken mevcut güvenli, zorlamalı ve sınırlı bekleme davranışını korur.
- Yerel hizmet yükleme, başlatma, durdurma ve kaldırma işlemleri reddedilir ve harici gözetmenin kullanılması yönünde rehberlik sağlanır.
- Gözetmenin Gateway'i durdurabilmesi, çalışma zamanını değiştirip tamamlayabilmesi ve güvenli şekilde yeniden başlatabilmesi için OpenClaw'ın kendini güncellemesi reddedilir.
- Yeni süreçle yeniden başlatma, temiz çıkıştan önce sınırlı bir SQLite devri yazar. Kalıcılaştırma başarısız olursa Gateway, tüketilebilir bir devir olmadan çıkmak yerine süreç içi yeniden başlatmaya geri döner.

`OPENCLAW_SERVICE_REPAIR_POLICY=external`, ayrı bir Doctor onarım politikası olarak kalır. Çalışma zamanı sahipliğini bildirmez; her iki davranışa da ihtiyaç duyan gözetmenler iki değişkeni de ayarlamalıdır.

Harici gözetmenler, gizli makine sözleşmesi üzerinden yeniden başlatma devirlerini uzlaştırabilir ve tüketebilir:

```bash
openclaw gateway restart-handoff capabilities --json
openclaw gateway restart-handoff consume --expected-pid <pid> --json
```

`1` protokol sürümü, `consume` işlemini destekler. Tüketim, beklenen PID'yi ve sınırlı devir alanlarını tek bir anlık SQLite işlemi içinde doğrular. Kabul edilen devir, başarı döndürülmeden önce silinir; böylece eşzamanlı veya yeniden oynatılan tüketicilerin ikisi birden bunu kabul edemez. PID uyuşmazlığı eşleşen sahip için korunur; eksik, süresi dolmuş ve geçersiz satırlar yeniden başlatmayı yetkilendirmez.

Geçerli makine istekleri, yeniden başlatma dışı sonuçlar dâhil olmak üzere `0` çıkış koduyla JSON döndürür. Geçersiz bağımsız değişkenler, `2` çıkış koduyla `reason: "invalid-expected-pid"` döndürür; durum deposu hataları, `1` çıkış koduyla `reason: "store-unavailable"` döndürür. Gözetmenler, desteği bir OpenClaw sürüm dizesinden çıkarmak veya özel SQLite şemasını doğrudan okumak yerine kullanacakları tam çalışma zamanı ya da başlatıcı üzerinde `capabilities` yoklaması yapmalıdır.

### Gateway profilleme

- `OPENCLAW_GATEWAY_STARTUP_TRACE=1`, başlangıç sırasında aşama zamanlamalarını günlüğe kaydeder; bunlara aşama başına `eventLoopMax` gecikmesi ve Plugin arama tablosu zamanlamaları (yüklü dizini, manifest kayıt defteri, başlangıç planlaması, sahip eşlemesi çalışması) dâhildir.
- `OPENCLAW_GATEWAY_RESTART_TRACE=1`, yeniden başlatma kapsamlı `restart trace:` satırlarını günlüğe kaydeder: sinyal işleme, etkin işlerin tamamlanmasını bekleme, kapatma aşamaları, sonraki başlangıç, hazır olma zamanlaması ve bellek metrikleri.
- `OPENCLAW_DIAGNOSTICS=timeline` ile `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=<path>`, harici QA düzenekleri için en iyi çabayla bir JSONL başlangıç tanılama zaman çizelgesi yazar (yapılandırmadaki `diagnostics.flags: ["timeline"]` ile eşdeğerdir; yol yine yalnızca ortam değişkeniyle ayarlanabilir). Olay döngüsü örneklerini eklemek için `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` ekleyin.
- `pnpm build`, ardından `pnpm test:startup:gateway -- --runs 5 --warmup 1`, Gateway başlangıcını derlenmiş CLI girişine göre karşılaştırmalı olarak ölçer: ilk süreç çıktısı, `/healthz`, `/readyz`, başlangıç izleme zamanlamaları, olay döngüsü gecikmesi ve Plugin arama tablosu zamanlaması.
- `pnpm build`, ardından `pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5`, macOS veya Linux'ta süreç içi yeniden başlatmayı karşılaştırmalı olarak ölçer (Windows'ta desteklenmez; yeniden başlatma `SIGUSR1` gerektirir). `SIGUSR1` kullanır, alt süreçte her iki izlemeyi etkinleştirir ve sonraki `/healthz`, sonraki `/readyz`, kesinti süresi, hazır olma zamanlaması, CPU, RSS ve yeniden başlatma izleme metriklerini kaydeder.
- `/healthz` çalışırlık göstergesidir; `/readyz` kullanılabilirlik hazırlığıdır. İzleme satırlarını ve karşılaştırmalı ölçüm çıktısını, tek bir zaman aralığı veya örnekten çıkarılmış eksiksiz bir performans sonucu olarak değil, sahip ilişkilendirme sinyali olarak değerlendirin.

## Çalışan bir Gateway'i sorgulama

Tüm sorgu komutları WebSocket RPC kullanır.

<Tabs>
  <Tab title="Çıktı modları">
    - Varsayılan: insanlar tarafından okunabilir (TTY'de renkli).
    - `--json`: makine tarafından okunabilir JSON (biçimlendirme/döndürücü yok).
    - `--no-color` (veya `NO_COLOR=1`): insan odaklı düzeni korurken ANSI'yi devre dışı bırakır.

  </Tab>
  <Tab title="Paylaşılan seçenekler">
    - `--url <url>`: Gateway WebSocket URL'si.
    - `--token <token>`: Gateway token'ı.
    - `--password <password>`: Gateway parolası.
    - `--timeout <ms>`: zaman aşımı/süre sınırı (varsayılan komuta göre değişir; aşağıdaki her komuta bakın).
    - `--expect-final`: "nihai" yanıtı bekler (agent çağrıları).

  </Tab>
</Tabs>

<Note>
`--url` ayarlandığında CLI, yapılandırma veya ortam kimlik bilgilerine geri dönmez. `--token` ya da `--password` değerini açıkça geçirin. Açık kimlik bilgilerinin eksik olması hatadır.
</Note>

### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
openclaw gateway health --port 18789
```

`/healthz` bir canlılık yoklamasıdır: sunucu HTTP'ye yanıt verebilir duruma gelir gelmez sonuç döndürür. `/readyz` daha katıdır ve başlangıç Plugin sidecar'ları, kanallar veya yapılandırılmış hook'lar hâlâ kararlı duruma geçerken kırmızı kalır. Yerel veya kimliği doğrulanmış ayrıntılı `/readyz` yanıtları bir `eventLoop` tanılama bloğu (gecikme, kullanım, CPU çekirdeği oranı, `degraded` bayrağı) içerir.

<ParamField path="--port <port>" type="number">
  Bu porttaki yerel geri döngü Gateway'ini hedefler. Bu çağrı için `OPENCLAW_GATEWAY_URL` ve `OPENCLAW_GATEWAY_PORT` değerlerini geçersiz kılar.
</ParamField>

### `gateway usage-cost`

Oturum günlüklerinden kullanım maliyeti özetlerini getirir.

```bash
openclaw gateway usage-cost
openclaw gateway usage-cost --days 7
openclaw gateway usage-cost --agent work --json
openclaw gateway usage-cost --all-agents
openclaw gateway usage-cost --json
```

<ParamField path="--days <days>" type="number" default="30">
  Dahil edilecek gün sayısı.
</ParamField>
<ParamField path="--agent <id>" type="string">
  Özeti yapılandırılmış tek bir aracı kimliğiyle sınırlar.
</ParamField>
<ParamField path="--all-agents" type="boolean">
  Yapılandırılmış tüm aracılar genelinde toplar. `--agent` ile birlikte kullanılamaz.
</ParamField>

### `gateway stability`

Çalışan bir Gateway'den yakın tarihli tanılama kararlılığı kaydedicisini getirir.

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --bundle latest
openclaw gateway stability --bundle latest --export
openclaw gateway stability --json
```

<ParamField path="--limit <limit>" type="number" default="25">
  Dahil edilecek en fazla yakın tarihli olay sayısı (en fazla `1000`).
</ParamField>
<ParamField path="--type <type>" type="string">
  Tanılama olayı türüne göre filtreler; ör. `payload.large` veya `diagnostic.memory.pressure`.
</ParamField>
<ParamField path="--since-seq <seq>" type="number">
  Yalnızca bir tanılama sıra numarasından sonraki olayları dahil eder.
</ParamField>
<ParamField path="--bundle [path]" type="string">
  Çalışan Gateway'i çağırmak yerine kalıcı bir kararlılık paketini okur. `--bundle latest` (veya tek başına `--bundle`) durum dizini altındaki en yeni paketi seçer; doğrudan bir paket JSON yolu da iletebilirsiniz.
</ParamField>
<ParamField path="--export" type="boolean">
  Kararlılık ayrıntılarını yazdırmak yerine paylaşılabilir bir destek tanılama zip'i yazar.
</ParamField>
<ParamField path="--output <path>" type="string">
  `--export` için çıktı yolu.
</ParamField>

<AccordionGroup>
  <Accordion title="Gizlilik ve paket davranışı">
    - Kayıtlar operasyonel meta verileri tutar: olay adları, sayılar, bayt boyutları, bellek ölçümleri, kuyruk/oturum durumu, onay kimlikleri, kanal/Plugin adları ve sansürlenmiş oturum özetleri. Sohbet metnini, Webhook gövdelerini, araç çıktılarını, ham istek/yanıt gövdelerini, token'ları, çerezleri, gizli değerleri, ana makine adlarını ve ham oturum kimliklerini hariç tutarlar. Kaydediciyi tamamen devre dışı bırakmak için `diagnostics.enabled: false` ayarını kullanın.
    - Ölümcül Gateway çıkışları, kapatma zaman aşımları ve yeniden başlatma başlangıç hataları, kaydedicide olaylar bulunduğunda aynı tanılama anlık görüntüsünü `~/.openclaw/logs/stability/openclaw-stability-*.json` konumuna yazar. En yeni paketi `openclaw gateway stability --bundle latest` ile inceleyin; `--limit`, `--type` ve `--since-seq` paket çıktısına da uygulanır.

  </Accordion>
</AccordionGroup>

### `gateway diagnostics export`

Hata raporları için tasarlanmış yerel bir tanılama zip'i yazar. Gizlilik modeli ve paket içeriği için [Tanılama Dışa Aktarımı](/tr/gateway/diagnostics) bölümüne bakın.

```bash
openclaw gateway diagnostics export
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
openclaw gateway diagnostics export --json
```

<ParamField path="--output <path>" type="string">
  Çıktı zip yolu. Varsayılan olarak durum dizini altında bir destek dışa aktarımı kullanılır.
</ParamField>
<ParamField path="--log-lines <count>" type="number" default="5000">
  Dahil edilecek en fazla temizlenmiş günlük satırı sayısı.
</ParamField>
<ParamField path="--log-bytes <bytes>" type="number" default="1000000">
  İncelenecek en fazla günlük baytı.
</ParamField>
<ParamField path="--url <url>" type="string">
  Sağlık anlık görüntüsü için Gateway WebSocket URL'si.
</ParamField>
<ParamField path="--token <token>" type="string">
  Sağlık anlık görüntüsü için Gateway token'ı.
</ParamField>
<ParamField path="--password <password>" type="string">
  Sağlık anlık görüntüsü için Gateway parolası.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="3000">
  Durum/sağlık anlık görüntüsü zaman aşımı.
</ParamField>
<ParamField path="--no-stability-bundle" type="boolean">
  Kalıcı kararlılık paketi aramasını atlar.
</ParamField>
<ParamField path="--json" type="boolean">
  Yazılan yolu, boyutu ve manifesti JSON olarak yazdırır.
</ParamField>

Dışa aktarım şu öğeleri paketler: `manifest.json` (dosya envanteri), `summary.md` (Markdown özeti), `diagnostics.json` (üst düzey yapılandırma/günlükler/keşif/kararlılık/durum/sağlık özeti), `config/sanitized.json`, `status/gateway-status.json`, `health/gateway-health.json`, `logs/openclaw-sanitized.jsonl` ve bir paket mevcut olduğunda `stability/latest.json`.

Paylaşılmak üzere tasarlanmıştır. Hata ayıklama için yararlı operasyonel ayrıntıları — güvenli günlük alanları, alt sistem adları, durum kodları, süreler, yapılandırılmış modlar, portlar, Plugin/sağlayıcı kimlikleri, gizli olmayan özellik ayarları ve sansürlenmiş operasyonel günlük iletileri — korur; sohbet metnini, Webhook gövdelerini, araç çıktılarını, kimlik bilgilerini, çerezleri, hesap/ileti tanımlayıcılarını, istem/talimat metnini, ana makine adlarını ve gizli değerleri hariç tutar veya sansürler. Bir günlük iletisi kullanıcı/sohbet/araç yükü metnine benziyorsa (ör. "kullanıcı söyledi", "sohbet metni", "araç çıktısı", "Webhook gövdesi"), dışa aktarım yalnızca bir iletinin çıkarıldığı bilgisini ve bayt sayısını korur.

### `gateway status`

Gateway hizmetini (launchd/systemd/schtasks) ve isteğe bağlı bağlantı/kimlik doğrulama yoklamasını gösterir.

```bash
openclaw gateway status
openclaw gateway status --json
openclaw gateway status --require-rpc
```

<ParamField path="--url <url>" type="string">
  Açık bir yoklama hedefi ekler. Yapılandırılmış uzak hedef ve localhost yine yoklanır.
</ParamField>
<ParamField path="--token <token>" type="string">
  Yoklama için token kimlik doğrulaması.
</ParamField>
<ParamField path="--password <password>" type="string">
  Yoklama için parola kimlik doğrulaması.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  Yoklama zaman aşımı.
</ParamField>
<ParamField path="--no-probe" type="boolean">
  Bağlantı yoklamasını atlar (yalnızca hizmet görünümü).
</ParamField>
<ParamField path="--deep" type="boolean">
  Sistem düzeyindeki hizmetleri de tarar.
</ParamField>
<ParamField path="--require-rpc" type="boolean">
  Bağlantı yoklamasını bir okuma yoklamasına yükseltir ve başarısız olursa sıfırdan farklı bir kodla çıkar. `--no-probe` ile birlikte kullanılamaz.
</ParamField>

<AccordionGroup>
  <Accordion title="Durum semantiği">
    - Yerel CLI yapılandırması eksik veya geçersiz olduğunda bile tanılama için kullanılabilir kalır.
    - Varsayılan çıktı; okuma/yazma/yönetici işlemlerini değil, hizmet durumunu, WebSocket bağlantısını ve el sıkışma sırasında görülebilen kimlik doğrulama yeteneğini doğrular.
    - İlk cihaz kimlik doğrulamasında yoklamalar değişiklik yapmaz: mevcutsa önbelleğe alınmış cihaz token'ını yeniden kullanırlar ancak yalnızca durumu kontrol etmek için asla yeni bir CLI cihaz kimliği veya salt okunur eşleştirme kaydı oluşturmazlar.
    - Mümkün olduğunda, yoklama kimlik doğrulaması için yapılandırılmış kimlik doğrulama SecretRef'lerini çözümler. Gerekli bir SecretRef çözümlenmemişse, yoklama bağlantısı/kimlik doğrulaması başarısız olduğunda `--json`, `rpc.authWarning` değerini bildirir; `--token`/`--password` değerlerini açıkça iletin veya gizli değer kaynağını düzeltin. Yoklama başarılı olduğunda çözümlenmemiş kimlik doğrulama uyarıları bastırılır.
    - Çalışan Gateway bildirdiğinde JSON çıktısı `gateway.version` değerini içerir; el sıkışma yoklaması sürüm meta verilerini sağlayamıyorsa `--require-rpc`, `status.runtimeVersion` RPC yüküne geri dönebilir.
    - Dinleyen bir hizmet yeterli olmadığında ve okuma kapsamlı RPC'nin de sağlıklı olması gerektiğinde betiklerde/otomasyonda `--require-rpc` kullanın.
    - `--deep`, ek launchd/systemd/schtasks kurulumlarını tarar; Gateway benzeri birden fazla hizmet bulunduğunda insan tarafından okunabilir çıktı temizleme ipuçlarını yazdırır (genellikle makine başına bir Gateway çalıştırın) ve ilgili olduğunda yakın tarihli bir yönetici yeniden başlatma devrini bildirir.
    - `--deep`, yapılandırma doğrulamasını Plugin'e duyarlı modda (`pluginValidation: "full"`) da çalıştırır ve Plugin manifest uyarılarını (ör. eksik kanal yapılandırması meta verileri) gösterir. Varsayılan `gateway status`, Plugin doğrulamasını atlayan hızlı salt okunur yolu korur.
    - İnsan tarafından okunabilir çıktı, profil veya durum dizini sapmasını tanılamaya yardımcı olmak için çözümlenmiş dosya günlük yolunun yanı sıra CLI ile hizmetin yapılandırma yollarını/geçerliliğini içerir.
    - İnsan tarafından okunabilir çıktı, uygulanan sınır ve uyarlamalı türetimiyle birlikte `Gateway heap:` değerini içerir. JSON çıktısı aynı raporu `service.gatewayHeap` olarak sunar.

  </Accordion>
  <Accordion title="Linux systemd kimlik doğrulama sapması kontrolleri">
    - Hizmet kimlik doğrulama sapması kontrolleri, birimden hem `Environment=` hem de `EnvironmentFile=` değerlerini okur (`%h`, tırnak içine alınmış yollar, birden fazla dosya ve isteğe bağlı `-` dosyaları dahil).
    - `gateway.auth.token` SecretRef'lerini birleştirilmiş çalışma zamanı ortamını kullanarak çözümler (önce hizmet komutu ortamı, ardından işlem ortamı geri dönüşü).
    - Token sapması kontrolleri, token kimlik doğrulaması etkin olarak kullanılmadığında yapılandırma token'ının çözümlenmesini atlar (`gateway.auth.mode` açıkça `password`/`none`/`trusted-proxy` olduğunda veya parolanın öncelik kazanabildiği ve hiçbir token adayının öncelik kazanamadığı ayarlanmamış modda).

  </Accordion>
</AccordionGroup>

### `gateway probe`

"Her şeyde hata ayıkla" komutu. Her zaman şunları yoklar:

- yapılandırılmış uzak Gateway'inizi (ayarlanmışsa) ve
- uzak hedef yapılandırılmış olsa bile localhost'u (geri döngü), **uzak hedef yapılandırılmış olsa bile**.

`--url` iletildiğinde bu açık hedef, her ikisinin önüne eklenir. İnsan tarafından okunabilir çıktı hedefleri `URL (explicit)`, `Remote (configured)` / `Remote (configured, inactive)` ve `Local loopback` olarak etiketler.

<Note>
Birden fazla yoklama hedefine erişilebiliyorsa tümü yazdırılır. SSH tüneli, TLS/proxy URL'si ve yapılandırılmış uzak URL, taşıma portları farklı olsa bile aynı Gateway'i gösterebilir; `multiple_gateways`, birbirinden farklı veya kimliği belirsiz erişilebilir Gateway'ler için ayrılmıştır. Birden fazla Gateway çalıştırmak, yalıtılmış profiller (ör. kurtarma botu) için desteklenir ancak çoğu kurulum tek bir Gateway çalıştırır.
</Note>

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --port 18789
```

<ParamField path="--port <port>" type="number">
  Yerel geri döngü yoklama hedefi ve SSH tüneli uzak portu için bu portu kullanır. `--url` olmadan bu seçenek; yapılandırılmış Gateway ortam URL'si, ortam portu veya uzak hedefler yerine yalnızca yerel geri döngü hedefini seçer.
</ParamField>

<AccordionGroup>
  <Accordion title="Yorumlama">
    - `Reachable: yes`, en az bir hedefin WebSocket bağlantısını kabul ettiği anlamına gelir.
    - `Capability: read-only|write-capable|admin-capable|pairing-pending|connect-only`, erişilebilirlikten ayrı olarak yoklamanın kimlik doğrulama hakkında neyi doğrulayabildiğini bildirir.
    - `Read probe: ok`, okuma kapsamlı ayrıntı RPC çağrılarının (`health`/`status`/`system-presence`/`config.get`) da başarılı olduğu anlamına gelir.
    - `Read probe: limited - missing scope: operator.read`, bağlantının başarılı olduğu ancak okuma kapsamlı RPC'nin sınırlı olduğu anlamına gelir. Tam başarısızlık olarak değil, **düşürülmüş** erişilebilirlik olarak bildirilir.
    - `Connect: ok` sonrasındaki `Read probe: failed`, WebSocket'in bağlandığı ancak takip eden okuma tanılamalarının zaman aşımına uğradığı veya başarısız olduğu anlamına gelir; bu da erişilemezlik değil, **düşürülmüş** durumdur.
    - `gateway status` gibi, yoklama da mevcut önbelleğe alınmış cihaz kimlik doğrulamasını yeniden kullanır ancak ilk cihaz kimliğini veya eşleştirme durumunu oluşturmaz.
    - Çıkış kodu yalnızca yoklanan hedeflerin hiçbirine erişilemediğinde sıfırdan farklıdır.

  </Accordion>
  <Accordion title="JSON çıktısı">
    Üst düzey:

    - `ok`: en az bir hedefe erişilebilir.
    - `degraded`: en az bir hedef bağlantıyı kabul etti ancak tam ayrıntılı RPC tanılamasını tamamlamadı.
    - `capability`: erişilebilir hedeflerde görülen en iyi yetenek (`read_only`, `write_capable`, `admin_capable`, `pairing_pending`, `connected_no_operator_scope` veya `unknown`).
    - `primaryTargetId`: etkin kazanan olarak değerlendirilecek en iyi hedef; sırasıyla: açık URL, SSH tüneli, yapılandırılmış uzak hedef, yerel geri döngü.
    - `warnings[]`: `code`, `message` ve isteğe bağlı `targetIds` içeren, mümkün olan en iyi şekilde oluşturulmuş uyarı kayıtları.
    - `network`: mevcut yapılandırmadan ve ana makine ağından türetilen yerel geri döngü/tailnet URL ipuçları.
    - `discovery.timeoutMs` / `discovery.count`: bu yoklama geçişinde kullanılan gerçek keşif bütçesi/sonuç sayısı.

    Hedef başına (`targets[].connect`): `ok` (erişilebilirlik + kısıtlı sınıflandırması), `rpcOk` (tam ayrıntılı RPC başarısı), `scopeLimited` (eksik operatör kapsamı nedeniyle ayrıntılı RPC başarısız oldu).

    Hedef başına (`targets[].auth`): kullanılabilir olduğunda `hello-ok` cinsinden bildirilen `role` ve `scopes` ile gösterilen `capability` sınıflandırması.

  </Accordion>
  <Accordion title="Yaygın uyarı kodları">
    - `ssh_tunnel_failed`: SSH tüneli kurulumu başarısız oldu; komut doğrudan yoklamalara geri döndü.
    - `multiple_gateways`: farklı gateway kimliklerine erişilebildi veya OpenClaw erişilebilir hedeflerin aynı gateway olduğunu kanıtlayamadı. Aynı gateway'e yönelik bir SSH tüneli, proxy URL'si veya yapılandırılmış uzak URL bunu tetiklemez.
    - `auth_secretref_unresolved`: yapılandırılmış bir kimlik doğrulama SecretRef'i, başarısız bir hedef için çözümlenemedi.
    - `probe_scope_limited`: WebSocket bağlantısı başarılı oldu ancak okuma yoklaması eksik `operator.read` nedeniyle kısıtlandı.
    - `local_tls_runtime_unavailable`: yerel Gateway TLS etkin ancak OpenClaw yerel sertifika parmak izini yükleyemedi.

  </Accordion>
</AccordionGroup>

#### SSH üzerinden uzak bağlantı (Mac uygulamasıyla eş değer)

macOS uygulamasındaki "Remote over SSH" modu, yalnızca geri döngü üzerinden erişilebilen uzak bir gateway'i `ws://127.0.0.1:<port>` adresinde erişilebilir kılmak için yerel port yönlendirme kullanır.

CLI eş değeri:

```bash
openclaw gateway probe --ssh user@gateway-host
```

<ParamField path="--ssh <target>" type="string">
  `user@host` veya `user@host:port` (port varsayılan olarak `22`).
</ParamField>
<ParamField path="--ssh-identity <path>" type="string">
  Kimlik dosyası.
</ParamField>
<ParamField path="--ssh-auto" type="boolean">
  Çözümlenen keşif uç noktasından (`local.` ve varsa yapılandırılmış geniş alan etki alanı) keşfedilen ilk gateway ana makinesini SSH hedefi olarak seçer. Yalnızca TXT içeren ipuçları yok sayılır.
</ParamField>

Yapılandırma varsayılanları (isteğe bağlı): `gateway.remote.sshTarget`, `gateway.remote.sshIdentity`.

### `gateway call <method>`

Düşük seviyeli RPC yardımcısı.

```bash
openclaw gateway call status
openclaw gateway call logs.tail --params '{"limit": 200}'
```

<ParamField path="--params <json>" type="string" default="{}">
  Parametreler için JSON nesnesi dizesi.
</ParamField>
<ParamField path="--url <url>" type="string">
  Gateway WebSocket URL'si.
</ParamField>
<ParamField path="--token <token>" type="string">
  Gateway belirteci.
</ParamField>
<ParamField path="--password <password>" type="string">
  Gateway parolası.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  Zaman aşımı bütçesi.
</ParamField>
<ParamField path="--expect-final" type="boolean">
  Temel olarak son yükten önce ara olayları akışla ileten aracı tarzı RPC'ler içindir.
</ParamField>
<ParamField path="--json" type="boolean">
  Makine tarafından okunabilir JSON çıktısı.
</ParamField>

<Note>
`--params` geçerli JSON olmalıdır ve her yöntem kendi parametre yapısını doğrular (fazladan/yanlış adlandırılmış alanlar reddedilir).
</Note>

## Gateway hizmetini yönetin

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway uninstall
```

### Bir sarmalayıcıyla yükleme

Yönetilen hizmetin başka bir yürütülebilir dosya üzerinden başlatılması gerektiğinde, örneğin bir gizli bilgi yöneticisi uyumluluk katmanı veya farklı kullanıcı olarak çalıştırma yardımcısı için `--wrapper` kullanın. Sarmalayıcı, normal Gateway bağımsız değişkenlerini alır ve sonunda bu bağımsız değişkenlerle `openclaw` veya Node'u exec ile çalıştırmaktan sorumludur.

```bash
cat > ~/.local/bin/openclaw-doppler <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
exec doppler run --project my-project --config production -- openclaw "$@"
EOF
chmod +x ~/.local/bin/openclaw-doppler

openclaw gateway install --wrapper ~/.local/bin/openclaw-doppler --force
openclaw gateway restart
```

Sarmalayıcıyı ortam üzerinden de ayarlayabilirsiniz. `gateway install`, yolun yürütülebilir bir dosya olduğunu doğrular, sarmalayıcıyı hizmet `ProgramArguments` içine yazar ve daha sonraki zorunlu yeniden yüklemeler, güncellemeler ve doctor onarımları için hizmet ortamında `OPENCLAW_WRAPPER` değerini kalıcı hâle getirir.

```bash
OPENCLAW_WRAPPER="$HOME/.local/bin/openclaw-doppler" openclaw gateway install --force
openclaw doctor
```

Kalıcı bir sarmalayıcıyı kaldırmak için yeniden yükleme sırasında `OPENCLAW_WRAPPER` değerini temizleyin:

```bash
OPENCLAW_WRAPPER= openclaw gateway install --force
openclaw gateway restart
```

<AccordionGroup>
  <Accordion title="Komut seçenekleri">
    - `gateway status`: `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json`
    - `gateway install`: `--port`, `--runtime <node>` (varsayılan: `node`), `--token`, `--wrapper <path>`, `--force`, `--json`
    - `gateway restart`: `--safe`, `--skip-deferral`, `--force`, `--wait <duration>`, `--json`
    - `gateway uninstall|start`: `--json`
    - `gateway stop`: `--disable`, `--force`, `--json`

  </Accordion>
  <Accordion title="Yaşam döngüsü davranışı">
    - `gateway start` birden çok kez güvenle çalıştırılabilir: yönetilen hizmet zaten çalışıyorsa çalışan işlemi bildirir ve ona dokunmaz. Yüklenmiş ancak durdurulmuş bir hizmet önceden olduğu gibi başlatılır.
    - Yönetilen bir hizmeti yeniden başlatmak için `gateway restart` kullanın. Yeniden başlatmanın yerine `gateway stop` ve `gateway start` komutlarını zincirlemeyin.
    - Etkileşimsiz bir kabukta `gateway stop`, `--force` gerektirir. Etkileşimli terminaller mevcut istemsiz davranışı korur. Otomasyon ve testler için `gateway run --dev` veya boş bir porta sahip yalıtılmış bir `--profile` tercih edin.
    - macOS'ta `gateway stop`, varsayılan olarak `launchctl bootout` kullanır; bu, kalıcı bir devre dışı bırakma oluşturmadan LaunchAgent'ı mevcut önyükleme oturumundan kaldırır. Böylece KeepAlive otomatik kurtarması gelecekteki çökmeler için etkin kalır ve `gateway start`, elle `launchctl enable` çalıştırmadan sorunsuz biçimde yeniden etkinleştirir. Gateway'in bir sonraki açık `gateway start` işlemine kadar yeniden başlatılmaması için KeepAlive ve RunAtLoad'u kalıcı olarak bastırmak üzere `--disable` iletin; elle durdurmanın yeniden başlatmalardan sonra da korunması gerektiğinde bunu kullanın.
    - Gateway yaşam döngüsü değişiklikleri; CLI başlatma, durdurma ve yeniden başlatma işlemleri, güvenli yeniden başlatma istekleri, gözetmen yeniden başlatmaları ve ayrılmış devirler dâhil olmak üzere, mümkün olan en iyi şekilde anahtar-değer denetim kayıtlarını `<state-dir>/logs/gateway-restart.log` dosyasına ekler.
    - Yaşam döngüsü komutları, betiklerde kullanım için `--json` kabul eder.

  </Accordion>
  <Accordion title="Yönetilen Gateway yığın boyutlandırması">
    - `gateway install`, yönetilen Gateway hizmeti için yalnızca yığına yönelik bir `NODE_OPTIONS` değeri yazar. Node bir kapsayıcı veya hizmet sınırı bildirdiğinde kısıtlı belleğin %50'sini, aksi takdirde fiziksel belleğin %50'sini hedefler.
    - Nominal hedef aralık 2048–8192 MiB'dir ve buna ek olarak %75 yerel ek kapasite üst sınırı uygulanır. Küçük ana makinelerde bu ek kapasite üst sınırı, uygulanan sınırı nominal 2048 MiB alt sınırının altına düşürebilir.
    - Yüklü hizmette zaten saklanan geçerli ve açık bir `--max-old-space-size`, zorunlu yeniden yüklemeler ve doctor onarımları boyunca korunur. Diğer `NODE_OPTIONS` bayrakları yönetilen hizmete aktarılmaz.
    - Kabuktaki `NODE_OPTIONS` ortam değeri bu politikayı geçersiz kılmaz. Yüklü değeri incelemek için `gateway status` veya `doctor` kullanın; yönetilen yığın ayarı bulunmayan eski hizmet meta verilerini yeniden oluşturmak için `openclaw gateway install --force` çalıştırın.
    - Politika yalnızca yönetilen Gateway hizmeti için geçerlidir. Ön plandaki `gateway run`, node hizmetleri ve elle yazılmış gözetmen birimleri kendi çalışma zamanı yapılandırmalarını korur.

  </Accordion>
  <Accordion title="Yükleme sırasında kimlik doğrulama ve SecretRef'ler">
    - Belirteç kimlik doğrulaması bir belirteç gerektirdiğinde ve `gateway.auth.token` SecretRef tarafından yönetildiğinde, `gateway install` SecretRef'in çözümlenebilir olduğunu doğrular ancak çözümlenen belirteci hizmet ortamı meta verilerinde kalıcı hâle getirmez.
    - Belirteç kimlik doğrulaması bir belirteç gerektiriyorsa ve yapılandırılmış belirteç SecretRef'i çözümlenemiyorsa yükleme, yedek düz metni kalıcı hâle getirmek yerine güvenli biçimde başarısız olur.
    - `gateway run` üzerindeki parola kimlik doğrulaması için satır içi `--password` yerine `OPENCLAW_GATEWAY_PASSWORD`, `--password-file` veya SecretRef destekli `gateway.auth.password` tercih edin.
    - Çıkarımlı kimlik doğrulama modunda yalnızca kabukta bulunan `OPENCLAW_GATEWAY_PASSWORD`, yükleme belirteci gereksinimlerini gevşetmez; yönetilen bir hizmet yüklerken kalıcı yapılandırma (`gateway.auth.password` veya yapılandırma `env`) kullanın.
    - Hem `gateway.auth.token` hem de `gateway.auth.password` yapılandırılmışsa ve `gateway.auth.mode` ayarlanmamışsa mod açıkça ayarlanana kadar yükleme engellenir.

  </Accordion>
</AccordionGroup>

## Gateway'leri keşfedin (Bonjour)

`gateway discover`, Gateway işaretçilerini (`_openclaw-gw._tcp`) tarar.

- Çok noktaya yayın DNS-SD: `local.`
- Tek noktaya yayın DNS-SD (geniş alan Bonjour): bir etki alanı seçin (örnek: `openclaw.internal.`) ve bölünmüş DNS ile bir DNS sunucusu kurun; [Bonjour](/tr/gateway/bonjour) bölümüne bakın.

Yalnızca Bonjour keşfi etkinleştirilmiş (varsayılan) gateway'ler işaretçiyi yayımlar.

Her işaretçideki TXT ipuçları: `role` (gateway rolü ipucu), `transport` (aktarım ipucu, ör. `gateway`), `gatewayPort` (WebSocket portu, genellikle `18789`), `tailnetDns` (kullanılabilir olduğunda MagicDNS ana makine adı), `gatewayTls` / `gatewayTlsSha256` (TLS etkin + sertifika parmak izi). `sshPort` ve `cliPath` yalnızca tam keşif modunda yayımlanır (`discovery.mdns.mode: "full"`; varsayılan `"minimal"` bunları hariç tutar — istemciler daha sonra SSH hedefleri için varsayılan olarak `22` portunu kullanır).

### `gateway discover`

```bash
openclaw gateway discover
```

<ParamField path="--timeout <ms>" type="number" default="2000">
  Komut başına zaman aşımı (göz atma/çözümleme).
</ParamField>
<ParamField path="--json" type="boolean">
  Makine tarafından okunabilir çıktı (biçimlendirmeyi/döndürücüyü de devre dışı bırakır).
</ParamField>

Örnekler:

```bash
openclaw gateway discover --timeout 4000
openclaw gateway discover --json | jq '.beacons[].wsUrl'
```

<Note>
- Etkinleştirilmişse yapılandırılmış geniş alan etki alanıyla birlikte `local.` taranır.
- JSON çıktısındaki `wsUrl`, `lanHost` veya `tailnetDns` gibi yalnızca TXT içeren ipuçlarından değil, çözümlenen hizmet uç noktasından türetilir.
- `discovery.mdns.mode`, hem `local.` mDNS hem de geniş alan DNS-SD'de `sshPort`/`cliPath` yayımlanmasını denetler (yukarıya bakın).

</Note>

## İlgili içerikler

- [CLI referansı](/tr/cli)
- [Gateway operasyon kılavuzu](/tr/gateway)
