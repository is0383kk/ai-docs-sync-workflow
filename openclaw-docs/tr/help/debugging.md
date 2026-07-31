---
read_when:
    - Ham model çıktısını akıl yürütme sızıntısı açısından incelemeniz gerekir
    - Yineleme yaparken Gateway'i izleme modunda çalıştırmak istiyorsunuz
    - Tekrarlanabilir bir hata ayıklama iş akışına ihtiyacınız var
summary: 'Hata ayıklama araçları: izleme modu, ham model akışları ve akıl yürütme sızıntısını izleme'
title: Hata Ayıklama
x-i18n:
    generated_at: "2026-07-26T23:22:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 45a1196c03e4deede3ce47553e1b2b3e1903ee04fe6855d929e0c32bf4e5e686
    source_path: help/debugging.md
    workflow: 16
---

Akış çıktısı, Gateway yinelemesi ve başlangıç profillemesi için hata ayıklama yardımcıları.

## Çalışma zamanı hata ayıklama geçersiz kılmaları

`/debug`, **yalnızca çalışma zamanına özgü** yapılandırma geçersiz kılmaları (diskte değil, bellekte) ayarlar. Varsayılan olarak devre dışıdır; `commands.debug: true` ile etkinleştirin.

```text
/debug show
/debug set channels.whatsapp.responsePrefix="[openclaw]"
/debug unset channels.whatsapp.responsePrefix
/debug reset
```

`/debug reset`, tüm geçersiz kılmaları temizler ve diskteki yapılandırmaya geri döner.

## Oturum izleme çıktısı

`/trace`, tam ayrıntılı modu etkinleştirmeden tek bir oturum için plugin'e ait izleme/hata ayıklama satırlarını gösterir. Active Memory hata ayıklama özetleri gibi plugin tanılamaları için bunu; normal durum/araç çıktısı için `/verbose` kullanın.

```text
/trace
/trace on
/trace off
```

## Plugin yaşam döngüsü izlemesi

Plugin meta verileri, keşif, kayıt defteri, çalışma zamanı aynası, yapılandırma değişikliği ve yenileme çalışmalarının aşama aşama dökümü için `OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1` ayarlayın. stderr'e yazar; böylece JSON komut çıktısı ayrıştırılabilir kalır.
Bu izleme etkin olduğunda plugin yükleme hataları yığın izlerini içerir.

```bash
OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1 openclaw plugins install tokenjuice --force
```

```text
[plugins:lifecycle] phase="config read" ms=6.83 status=ok command="install"
[plugins:lifecycle] phase="slot selection" ms=94.31 status=ok command="install" pluginId="tokenjuice"
[plugins:lifecycle] phase="registry refresh" ms=51.56 status=ok command="install" reason="source-changed"
```

Bir CPU profilleyiciye başvurmadan önce bunu kullanın. Kaynak kod deposundan, `pnpm build` sonrasında `node dist/entry.js ...` ile derlenmiş çalışma zamanını ölçün; `pnpm openclaw ...` ayrıca kaynak çalıştırıcı ek yükünü de ölçer.

Eşzamanlı modül yükleme süreleri için plugin'e özgü ayrı bir ortam anahtarı yerine paylaşılan tanılama yüzeyini kullanın:

```bash
OPENCLAW_DIAGNOSTICS=plugin.load-profile openclaw plugins list
```

## CLI başlangıç ve komut profillemesi

Depoya kaydedilmiş başlangıç karşılaştırmalı değerlendirmeleri:

```bash
pnpm test:startup:bench:smoke
pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --runs 3
pnpm tsx scripts/bench-cli-startup.ts --preset real --cpu-prof-dir .artifacts/cli-cpu
```

Normal kaynak çalıştırıcı üzerinden tek seferlik profilleme için `OPENCLAW_RUN_NODE_CPU_PROF_DIR` ayarlayın:

```bash
OPENCLAW_RUN_NODE_CPU_PROF_DIR=.artifacts/cli-cpu pnpm openclaw status
```

Kaynak çalıştırıcı, Node CPU profili bayraklarını ekler ve komut için bir `.cpuprofile` yazar. Komut koduna geçici enstrümantasyon eklemeden önce bunu kullanın.

Eşzamanlı dosya sistemi veya modül yükleyici çalışması gibi görünen başlangıç takılmalarında, kaynak çalıştırıcı üzerinden Node'un eşzamanlı G/Ç izleme bayrağını ekleyin:

```bash
OPENCLAW_TRACE_SYNC_IO=1 pnpm openclaw gateway --force
```

`pnpm gateway:watch`, izlenen Gateway alt süreci için bu bayrağı varsayılan olarak devre dışı bırakır; izleme modunda da eşzamanlı G/Ç izleme çıktısı istediğinizde `OPENCLAW_TRACE_SYNC_IO=1` ayarlayın.

## Gateway izleme modu

```bash
pnpm gateway:watch
```

Bu, varsayılan olarak `openclaw-gateway-watch-<profile>` adlı bir tmux oturumunu başlatır veya yeniden başlatır (örneğin `openclaw-gateway-watch-main`). `OPENCLAW_GATEWAY_PORT`, varsayılan `18789` portundan farklı olduğunda `openclaw-gateway-watch-dev-19001` gibi bir port son eki eklenir. Etkileşimli terminallerden otomatik olarak bağlanır; etkileşimsiz kabuklar, CI ve ajan yürütme çağrıları bağlantısız kalır ve bunun yerine bağlanma talimatlarını yazdırır:

```bash
tmux attach -t openclaw-gateway-watch-main
# Bağlanmadan son çıktıyı okuyun
tmux capture-pane -ep -t openclaw-gateway-watch-main -S -200
```

Bölme, tmux `remain-on-exit` kullanır; böylece başlangıç hataları oturumu silmek yerine bağlanma veya yakalama için kullanılabilir kalır. `pnpm gateway:watch` komutunun yeniden çalıştırılması bu bölmeyi yeniden oluşturur.

tmux bölmesi ham izleyiciyi çalıştırır:

```bash
node scripts/watch-node.mjs gateway --force
```

Yapılandırılmış/varsayılan portu izlemeden önce tmux sarmalayıcı, etkin profilin kurulu Gateway hizmetini durdurur. Bu, launchd, systemd veya Scheduled Task hizmeti yeniden başlatıp yerine geçmeden portu kaynak izleyiciye devreder. Hizmet kurulu kalır; izleme oturumundan sonra şu komutla geri yükleyin:

```bash
pnpm openclaw gateway start
```

Açıkça belirtilen `--port` veya `OPENCLAW_GATEWAY_PORT`, kurulu hizmetin etkin portundan farklı olduğunda sarmalayıcı hizmeti çalışır durumda bırakır; böylece her iki Gateway yan yana çalışabilir.

tmux olmadan ön plan modu:

```bash
pnpm gateway:watch:raw
# veya
OPENCLAW_GATEWAY_WATCH_TMUX=0 pnpm gateway:watch
```

Ham mod, kurulu hizmeti yönetmez. Aynı portu kullanıyorsa önce `pnpm openclaw gateway stop` çalıştırın.

tmux yönetimini koruyup otomatik bağlanmayı devre dışı bırakın:

```bash
OPENCLAW_GATEWAY_WATCH_ATTACH=0 pnpm gateway:watch
```

Başlangıç/çalışma zamanı performans noktalarında hata ayıklarken izlenen Gateway CPU süresini profilleyin:

```bash
pnpm gateway:watch --benchmark
```

İzleme sarmalayıcı, Gateway'i çağırmadan önce `--benchmark` değerini tüketir ve `.artifacts/gateway-watch-profiles/` altında her Gateway alt süreç çıkışı için bir V8 `.cpuprofile` dosyası yazar. Geçerli profili diske yazmak için izlenen Gateway'i durdurun veya yeniden başlatın, ardından Chrome DevTools ya da Speedscope ile açın:

```bash
npx speedscope .artifacts/gateway-watch-profiles/*.cpuprofile
```

- `--benchmark-dir <path>`: profilleri başka bir yere yazın.
- `--benchmark-no-force`: varsayılan `--force` port temizliğini atlayın ve Gateway portu zaten kullanımdaysa hemen başarısız olun.

Karşılaştırmalı değerlendirme modu, eşzamanlı G/Ç izleme kalabalığını varsayılan olarak bastırır. Hem CPU profilleri hem de eşzamanlı G/Ç yığın izleri almak için `--benchmark` ile `OPENCLAW_TRACE_SYNC_IO=1` ayarlayın; karşılaştırmalı değerlendirme modunda bu izleme blokları karşılaştırmalı değerlendirme dizini altındaki `gateway-watch-output.log` konumuna gider (terminal bölmesinden filtrelenir), normal Gateway günlükleri ise görünür kalır.

tmux sarmalayıcı, `OPENCLAW_PROFILE`, `OPENCLAW_CONFIG_PATH`, `OPENCLAW_STATE_DIR`, `OPENCLAW_GATEWAY_PORT` ve `OPENCLAW_SKIP_CHANNELS` dahil olmak üzere yaygın ve gizli olmayan çalışma zamanı seçicilerini bölmeye taşır. Sağlayıcı kimlik bilgilerini normal profilinize/yapılandırmanıza koyun veya tek seferlik geçici gizli değerler için ham ön plan modunu kullanın.

İzlenen Gateway başlangıç sırasında çıkarsa izleyici `openclaw doctor --fix --non-interactive` komutunu bir kez çalıştırır ve Gateway alt sürecini yeniden başlatır. Geliştirmeye özgü onarım geçişi olmadan özgün başlangıç hatasını görmek için `OPENCLAW_GATEWAY_WATCH_AUTO_DOCTOR=0` ayarlayın.

Yönetilen tmux bölmesi varsayılan olarak renkli Gateway günlükleri kullanır; ANSI çıktısını devre dışı bırakmak için `pnpm gateway:watch` başlatılırken `FORCE_COLOR=0` ayarlayın.

İzleyici; `src/` altındaki derlemeyle ilgili dosyalarda, uzantı kaynak dosyalarında, uzantı `package.json` ve `openclaw.plugin.json` meta verilerinde, `tsconfig.json`, `package.json` ve `tsdown.config.ts` dosyalarında değişiklik olduğunda yeniden başlatılır. Uzantı meta verisi değişiklikleri yeniden derlemeyi zorlamadan Gateway'i yeniden başlatır; kaynak ve yapılandırma değişiklikleri yine de önce `dist` öğesini yeniden derler.

Gateway CLI bayraklarını `gateway:watch` sonrasına eklediğinizde her yeniden başlatmada aktarılırlar. Aynı izleme komutunun yeniden çalıştırılması adlandırılmış tmux bölmesini yeniden oluşturur; ham izleyici tek izleyici kilidi kullanır, böylece yinelenen izleyici üst süreçleri birikmek yerine değiştirilir.

## Geliştirme profili + geliştirme Gateway'i (--dev)

İki **ayrı** `--dev` bayrağı:

- **Genel `--dev` (profil):** durumu `~/.openclaw-dev` altında yalıtır ve Gateway portunu varsayılan olarak `19001` değerine ayarlar (türetilmiş portlar da buna göre kayar).
- **`gateway --dev`:** Gateway'e, eksik olduğunda varsayılan yapılandırmayı ve çalışma alanını otomatik oluşturmasını (ve önyüklemeyi atlamasını) söyler.

Önerilen akış (geliştirme profili + geliştirme önyüklemesi):

```bash
pnpm gateway:dev
OPENCLAW_PROFILE=dev openclaw tui
```

Genel kurulum olmadan CLI'yi `pnpm openclaw ...` üzerinden çalıştırın.

Yaptıkları:

1. **Profil yalıtımı** (genel `--dev`)
   - `OPENCLAW_PROFILE=dev`
   - `OPENCLAW_STATE_DIR=~/.openclaw-dev`
   - `OPENCLAW_CONFIG_PATH=~/.openclaw-dev/openclaw.json`
   - `OPENCLAW_GATEWAY_PORT=19001` (tarayıcı/canvas portları buna göre kayar)

2. **Geliştirme önyüklemesi** (`gateway --dev`)
   - Eksikse en küçük yapılandırmayı yazar (`gateway.mode=local`, geri döngüye bağlanır).
   - `agents.defaults.workspace` değerini geliştirme çalışma alanına, `agents.defaults.skipBootstrap=true` değerine ayarlar.
   - Eksikse çalışma alanı dosyalarını başlangıç verileriyle oluşturur: `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`.
   - Varsayılan kimlik: **C3-PO** (protokol droidi).
   - `pnpm gateway:dev` ayrıca kanal sağlayıcılarını atlamak için `OPENCLAW_SKIP_CHANNELS=1` değerini ayarlar.

Geliştirme Gateway'leri varsayılan olarak ortamdaki kanal tetikleyicilerini yok sayar; böylece kabuğunuzdan devralınan kimlik bilgileri geliştirme örneğini gerçek kanal hizmetlerine bağlamaz. Açık `channels.<id>` yapılandırması çalışmaya devam eder. O çalıştırma için ortamdaki kanal otomatik yapılandırmasını geri yüklemek üzere `--dev` ile `--dev-ambient-channels` aktarın.

Sıfırlama akışı (temiz başlangıç):

```bash
pnpm gateway:dev:reset
```

<Note>
`--dev`, **genel** bir profil bayrağıdır ve bazı çalıştırıcılar tarafından tüketilir. Açıkça belirtmeniz gerekiyorsa ortam değişkeni biçimini kullanın:

```bash
OPENCLAW_PROFILE=dev openclaw gateway --dev --reset
```

</Note>

`--reset`; yapılandırmayı, kimlik bilgilerini, oturumları ve geliştirme çalışma alanını temizler (silmek yerine çöp kutusuna taşır), ardından varsayılan geliştirme kurulumunu yeniden oluşturur.

<Tip>
Geliştirme dışı bir Gateway zaten çalışıyorsa (launchd veya systemd), önce onu durdurun:

```bash
openclaw gateway stop
```

</Tip>

## Ham akış günlükleme

OpenClaw, herhangi bir filtreleme/biçimlendirme öncesinde **ham asistan akışını** günlüğe kaydedebilir. Bu, akıl yürütmenin düz metin deltaları olarak mı (yoksa ayrı düşünme blokları olarak mı) geldiğini görmenin en iyi yoludur.

CLI üzerinden etkinleştirin:

```bash
pnpm gateway:watch --raw-stream
```

İsteğe bağlı yol geçersiz kılması:

```bash
pnpm gateway:watch --raw-stream --raw-stream-path ~/.openclaw/logs/raw-stream.jsonl
```

Eşdeğer ortam değişkenleri:

```bash
OPENCLAW_RAW_STREAM=1
OPENCLAW_RAW_STREAM_PATH=~/.openclaw/logs/raw-stream.jsonl
```

Varsayılan dosya: `~/.openclaw/logs/raw-stream.jsonl`

## Güvenlik notları

- Ham akış günlükleri tam istemleri, araç çıktısını ve kullanıcı verilerini içerebilir.
- Günlükleri yerel olarak tutun ve hata ayıklamadan sonra silin.
- Günlükleri paylaşırsanız önce gizli değerleri ve kişisel olarak tanımlanabilir bilgileri temizleyin.

## VSCode'da hata ayıklama

Derleme, oluşturulan dosya adlarını özetlediği için kaynak haritaları gereklidir. Dahil edilen `launch.json`, Gateway hizmetini hedefler:

1. **Rebuild and Debug Gateway** - Gateway'i başlatmadan önce `/dist` öğesini siler ve hata ayıklama etkin olarak yeniden derler.
2. **Debug Gateway** - `/dist` öğesine dokunmadan mevcut bir derlemede hata ayıklar.

### Kurulum

1. **Run and Debug** seçeneğini açın (Activity Bar veya `Ctrl`+`Shift`+`D`).
2. **Rebuild and Debug Gateway** seçeneğini belirleyin ve **Start Debugging** düğmesine basın.

Bunun yerine derleme/hata ayıklama döngüsünü elle yönetmek için:

1. Bir terminalde kaynak haritalarını etkinleştirin:
   - **Linux/macOS**: `export OUTPUT_SOURCE_MAPS=1`
   - **Windows (PowerShell)**: `$env:OUTPUT_SOURCE_MAPS="1"`
   - **Windows (CMD)**: `set OUTPUT_SOURCE_MAPS=1`
2. Yeniden derleyin: `pnpm clean:dist && pnpm build`
3. **Debug Gateway** seçeneğini belirleyin ve **Start Debugging** düğmesine basın.

`src/` TypeScript dosyalarında kesme noktaları ayarlayın; hata ayıklayıcı bunları kaynak haritaları aracılığıyla derlenmiş JavaScript'e eşler.

### Notlar

- **Rebuild and Debug Gateway**, `/dist` öğesini siler ve her başlatmada kaynak haritalarıyla tam bir `pnpm build` çalıştırır.
- **Debug Gateway**, `/dist` öğesini etkilemeden başlatılıp durdurulabilir; ancak derleme döngüsünü ayrı bir terminalde yönetirsiniz.
- Diğer CLI alt komutlarında hata ayıklamak için `launch.json` içindeki `args` öğesini düzenleyin.
- Derlenmiş CLI'yi başka görevlerde kullanmak için (örneğin hata ayıklama oturumunuz yeni bir kimlik doğrulama belirteci oluşturuyorsa `dashboard --no-open`), başka bir terminalden çalıştırın: `node ./openclaw.mjs` veya `alias openclaw-build="node $(pwd)/openclaw.mjs"` gibi bir takma ad.

## İlgili

- [Sorun giderme](/tr/help/troubleshooting)
- [SSS](/tr/help/faq)
