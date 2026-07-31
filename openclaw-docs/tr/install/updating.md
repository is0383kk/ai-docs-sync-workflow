---
read_when:
    - OpenClaw'u Güncelleme
    - Bir güncellemeden sonra bir şey bozuluyor
summary: OpenClaw'u güvenli bir şekilde güncelleme (genel kurulum veya kaynak), ayrıca geri alma stratejisi
title: Güncelleniyor
x-i18n:
    generated_at: "2026-07-26T22:50:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83444d56e0aa34f47830610538b0c3012903abb812bfe0fffb8163a5db9ac2db
    source_path: install/updating.md
    workflow: 16
---

OpenClaw'u güncel tutun.

Docker, Podman ve Kubernetes imaj değiştirmeleri için
[Kapsayıcı imajlarını yükseltme](/tr/install/docker#upgrading-container-images) bölümüne bakın. Gateway,
hazır olma durumundan önce başlangıç açısından güvenli yükseltme çalışmalarını yürütür ve bağlanan
durumun elle onarılması gerekiyorsa kapanır.

## Önerilen: `openclaw update`

Kurulum türünü (npm, pnpm, Bun veya git) algılar, en son sürümü getirir, `openclaw doctor` komutunu çalıştırır ve Gateway'i yeniden başlatır.

```bash
openclaw update
```

Kanalları değiştirin veya belirli bir sürümü hedefleyin:

```bash
openclaw update --channel beta
openclaw update --channel extended-stable
openclaw update --channel dev
openclaw update --dry-run   # uygulamadan önizle
```

`openclaw update` bir `--verbose` bayrağına sahip değildir (yükleyicide vardır). Tanılama için
planlanan eylemleri önizlemek üzere `--dry-run`, yapılandırılmış sonuçlar için `--json` veya
kanal ve kullanılabilirlik durumunu incelemek üzere `openclaw update status --json` kullanın.

`--channel beta`, beta npm dist-tag'ini tercih eder ancak beta etiketi eksikse veya
sürümü en son kararlı sürümden eskiyse stable/latest'e geri döner. Bunun yerine
ham npm beta dist-tag'ine sabitlenmiş tek seferlik bir paket güncellemesi için
`--tag beta` kullanın.

`--channel extended-stable` yalnızca paket içindir ve kurulum
yalnızca ön planda kalır. OpenClaw, genel npm `extended-stable` seçicisini okur,
seçilen kesin paketi doğrular ve tam olarak bu sürümü kurar. Eksik
veya tutarsız kayıt defteri verilerinde güvenli biçimde başarısız olur; hiçbir zaman `latest` seçeneğine geri dönmez.
Seçilen sürüm kurulu sürümden eskiyse normal
sürüm düşürme onayı yine geçerlidir. CLI, başarılı bir
çekirdek güncellemesinden sonra kanalı kalıcı hâle getirir; doğrudan `npm install -g openclaw@extended-stable`
işlemi `update.channel` değerini güncellemez.
Çekirdek değişiminden sonra, salt/varsayılan veya
`latest` amacı taşıyan uygun resmî npm plugin'leri tam olarak bu çekirdek sürümüne yakınsar. Kesin sabitlemeler ve açık
`latest` dışı etiketler, üçüncü taraf plugin'leri ve npm dışı kaynaklar değişmeden kalır.
Güncel OpenClaw sürümleri tarafından oluşturulan katalog kurulumları bu varsayılan
amacı korur. Yalnızca kesin bir sürüm içeren eski kayıtlar sabitlenmiş olarak kalır; çünkü
OpenClaw eski bir otomatik sabitlemeyi kullanıcı sabitlemesinden güvenli biçimde ayırt edemez.
Bu plugin'i kesin çekirdek izlemeye yeniden dahil etmek için extended-stable kanalında
`openclaw plugins update @openclaw/name` komutunu bir kez çalıştırın.

`--channel dev`, kalıcı ve hareketli bir GitHub `main` çalışma kopyası sağlar. Tek seferlik
bir paket güncellemesi için `--tag main`, `github:openclaw/openclaw#main` paket
belirtimine eşlenir ve hedef paket yöneticisi (npm/pnpm/bun) aracılığıyla doğrudan kurulur.

Yönetilen plugin'lerde eksik bir beta sürümü hata değil, uyarıdır:
bir plugin kayıtlı varsayılan/en son sürümüne geri dönerken çekirdek güncellemesi yine de başarılı olabilir.

Kanal anlamları için [Sürüm kanalları](/tr/install/development-channels) bölümüne bakın.

## npm ve git kurulumları arasında geçiş yapma

Kurulum türünü değiştirmek için kanalları kullanın. Güncelleyici, `~/.openclaw` içindeki
durumunuzu, yapılandırmanızı, kimlik bilgilerinizi ve çalışma alanınızı korur; yalnızca CLI ve
Gateway'in kullandığı OpenClaw kod kurulumunu değiştirir.

```bash
# npm paketi kurulumu -> düzenlenebilir git çalışma kopyası
openclaw update --channel dev

# git çalışma kopyası -> npm paketi kurulumu
openclaw update --channel stable
```

Önce kurulum modu geçişini önizleyin:

```bash
openclaw update --channel dev --dry-run
openclaw update --channel stable --dry-run
```

`dev` bir git çalışma kopyasının bulunmasını sağlar, bunu derler ve genel CLI'yi bu
çalışma kopyasından kurar. `stable`, `extended-stable` ve `beta` kanalları paket
kurulumlarını kullanır. Extended-stable, bir git çalışma kopyasında onu değiştirmeden veya
dönüştürmeden reddedilir. Gateway zaten kuruluysa, `--no-restart` iletmediğiniz sürece
`openclaw update` hizmet meta verilerini yeniler ve hizmeti yeniden başlatır.

Yönetilen bir Gateway hizmetine sahip paket kurulumlarında `openclaw update`,
bu hizmet tarafından kullanılan paket kökünü hedefler. Kabuktaki `openclaw` komutu
farklı bir kurulumdan geliyorsa güncelleyici her iki kökü ve yönetilen
hizmetin Node yolunu yazdırır; paketi değiştirmeden önce bu Node sürümünü hedef sürümün
`engines.node` gereksinimiyle karşılaştırır.

## Kaynak çalışma kopyası sunucuları (referans betiği)

Bir sunucuda doğrudan git çalışma kopyasından Gateway çalıştıran ekipler, bu çalışma kopyasının
içinden `scripts/update-gateway.sh` ile güncelleme yapabilir. Bu betik, verimli bir
kaynak sunucu güncellemesi için referanstır: `pnpm build` tarafından yeniden yazılan izlenen derleme çıktılarını geri yükler,
diğer tüm yerel değişikliklerde güvenli biçimde başarısız olur, `main` dalını hızlı ileri alır
(veya yerel bir sunucu dalını `origin/main` üzerine yeniden temellendirir), bağımlılıkları
kurar, temiz bir derleme yapar ve Gateway'i yeniden başlatır.

```bash
ssh you@server 'cd /path/to/openclaw && scripts/update-gateway.sh'
```

Özel hizmet birimleri için yeniden başlatma işlemini geçersiz kılın veya tamamen atlayın:

```bash
OPENCLAW_UPDATE_RESTART_CMD='systemctl --user restart openclaw-gateway.service' scripts/update-gateway.sh
OPENCLAW_UPDATE_RESTART_CMD='' scripts/update-gateway.sh
```

Sade, tek kullanıcılı bir kaynak kurulumu için bunun yerine `openclaw update --channel dev`
tercih edin; çalışma kopyasını, derlemeyi ve Gateway'in yeniden başlatılmasını sizin için yönetir.

## Alternatif: yükleyiciyi yeniden çalıştırma

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

İlk katılımı atlamak için `--no-onboard` ekleyin. Belirli bir kurulum türünü zorunlu kılmak için
`--install-method git --no-onboard` veya `--install-method npm --no-onboard` iletin.

npm paketi kurulum aşamasından sonra `openclaw update` başarısız olursa
bunun yerine yükleyiciyi yeniden çalıştırın. Yükleyici güncelleyiciyi çağırmaz; genel paket
kurulumunu doğrudan yürütür ve kısmen güncellenmiş bir npm kurulumunu kurtarabilir.

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm
```

Kurtarmayı belirli bir sürüme veya dist-tag'e sabitlemek için `--version` kullanın:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm --version <version-or-dist-tag>
```

## Alternatif: elle npm, pnpm veya bun kullanma

```bash
npm i -g openclaw@latest
```

Gözetimli kurulumlar için `openclaw update` tercih edin: paket
değişimini çalışan Gateway hizmetiyle koordine edebilir. Gözetimli bir
kurulumu elle güncellerseniz önce yönetilen Gateway'i durdurun. Paket yöneticileri dosyaları
yerinde değiştirir; aksi hâlde çalışan bir Gateway, değişim sırasında çekirdek veya plugin
dosyalarını yüklemeye çalışabilir. Yeni kurulumu kullanması için paket yöneticisi
tamamlandıktan sonra Gateway'i yeniden başlatın.

Root mülkiyetindeki Linux sistem geneli kurulumda `openclaw update`,
`EACCES` hatasıyla başarısız olursa elle değiştirme sırasında Gateway'i durdurulmuş
tutarak sistem npm'iyle kurtarma yapın. Bu Gateway için normalde kullandığınız profil
bayraklarını/ortamı kullanın. `/usr/bin/npm` değerini, ana makinenizde root mülkiyetindeki
genel önekin sahibi olan sistem npm'iyle değiştirin:

```bash
openclaw gateway stop
sudo /usr/bin/npm i -g openclaw@latest
openclaw gateway install --force
openclaw gateway restart
```

Ardından doğrulayın:

```bash
openclaw --version
curl -fsS http://127.0.0.1:18789/readyz
openclaw plugins list --json
openclaw gateway status --deep --json
openclaw doctor --lint --json
```

`openclaw update` genel bir npm kurulumunu yönettiğinde hedefi önce
geçici bir npm önekine kurar. Aday paket, `preinstall` sırasında ana makinenin
Node sürümünü doğrular; OpenClaw ancak bundan sonra paketlenmiş `dist`
envanterini doğrular ve temiz paket ağacını gerçek genel öneke taşır. Paketlenmiş bir
tamamlama koruması beklenen envanterden çıkarılır ve yalnızca `preinstall`
başarılı olduktan sonra kaldırılır; böylece atlanan yaşam döngüsü betikleri de değişimden önce
başarısız olur. npm 12 ve daha yeni sürümlerde güncelleyici yalnızca aday OpenClaw
yaşam döngüsünü onaylar; geçişli bağımlılık betikleri engellenmiş kalır. Bu, npm'in
yeni bir paketi eskisinden kalan dosyaların üzerine bindirmesini önler. Kurulum
komutu başarısız olursa OpenClaw, yerel isteğe bağlı bağımlılıkların derlenemediği
ana makinelerde yardımcı olan `--omit=optional` ile bir kez yeniden dener.

OpenClaw tarafından yönetilen npm güncelleme ve plugin güncelleme komutları, alt npm
işlemi için npm'in `min-release-age` tedarik zinciri karantinasını (veya eski
`before` yapılandırma anahtarını) da temizler. Bu politika genel koruma için vardır ancak
açık bir OpenClaw güncellemesi "seçilen sürümü şimdi kur" anlamına gelir.

```bash
pnpm add -g openclaw@latest
```

pnpm 11, OpenClaw 2026.7.1 sürümünü kurduysa bu elle çalıştırılan komutu bir kez uygulayın. Bu
sürüm, pnpm 11'in yalıtılmış genel paket düzeninden öncedir; dolayısıyla güncelleyicisi
başka bir npm kurulumunu çalışan CLI sanabilir. Sonraki sürümler
pnpm sahipliğini korur ve güncellemeler sırasında değiştirilen paket kökünü izler. Ayrıca
sahip olan yöneticinin bildirdiği genel ikili dizini kullanır ve kullanılabilir
pnpm komutu başka bir genel kök ya da ana sürüm bildirdiğinde veya çağıran paket
sahipsiz olduğunda ya da oradaki tek etkin OpenClaw kurulumu olmadığında değişiklikten önce durur.

OpenClaw, bir pnpm 11 genel kurulum grubunu başka bir paketle paylaşıyorsa
otomatik güncelleyici grubu değiştirmeden önce durur. Kardeş paketlerin ve derleme
politikasının bozulmaması için özgün virgülle ayrılmış grubu elle güncelleyin.

```bash
bun add -g openclaw@latest
```

### İleri düzey npm kurulum konuları

<AccordionGroup>
  <Accordion title="Salt okunur paket ağacı">
    OpenClaw, genel paket dizini geçerli kullanıcı tarafından yazılabilir olsa bile paketlenmiş genel kurulumları çalışma zamanında salt okunur kabul eder. Plugin paketi kurulumları, kullanıcı yapılandırma dizini altındaki OpenClaw'a ait npm/git köklerinde bulunur ve Gateway başlangıcı OpenClaw paket ağacını değiştirmez.

    Bazı Linux npm kurulumları genel paketleri `/usr/lib/node_modules/openclaw` gibi root mülkiyetindeki dizinlere kurar. Plugin kurma/güncelleme komutları bu genel paket dizininin dışına yazdığı için OpenClaw bu düzeni destekler.

  </Accordion>
  <Accordion title="Güçlendirilmiş systemd birimleri">
    Açık plugin kurulumlarının, plugin güncellemelerinin ve doctor temizliğinin değişikliklerini kalıcı hâle getirebilmesi için OpenClaw'a yapılandırma/durum köklerine yazma erişimi verin:

    ```ini
    ReadWritePaths=/var/lib/openclaw /home/openclaw/.openclaw /tmp
    ```

  </Accordion>
  <Accordion title="Disk alanı ön denetimi">
    OpenClaw, paket güncellemelerinden ve açık plugin kurulumlarından önce hedef birim için mümkün olan en iyi disk alanı denetimini yapmaya çalışır. Düşük alan, denetlenen yolu içeren bir uyarı oluşturur ancak dosya sistemi kotaları, anlık görüntüler ve ağ birimleri denetimden sonra değişebileceği için güncellemeyi engellemez. Asıl paket yöneticisi kurulumu ve kurulum sonrası doğrulama belirleyici olmaya devam eder.
  </Accordion>
</AccordionGroup>

## Otomatik güncelleyici

Varsayılan olarak kapalıdır. `~/.openclaw/openclaw.json` içinde etkinleştirin:

```json5
{
  update: {
    channel: "stable",
    auto: {
      enabled: true,
    },
  },
}
```

| Kanal             | Davranış                                                                                                                           |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `stable`          | Dağıtımlı bir kullanıma sunma için deterministik sapma içeren yerleşik bir gecikmeden sonra uygular.                               |
| `extended-stable` | `checkOnStart` etkin olduğunda başlangıçta ve her 24 saatte bir salt okunur güncelleme ipucu olup olmadığını denetler. Asla otomatik olarak uygulamaz. |
| `beta`            | Yerleşik bir aralıkta denetler ve hemen uygular.                                                                                    |
| `dev`             | Otomatik uygulama yoktur. `openclaw update` komutunu elle kullanın.                                                               |

Gateway ayrıca başlangıçta bir güncelleme ipucu günlüğe kaydeder (`update.checkOnStart: false` ile devre dışı bırakın). Saklanan extended-stable seçimleri bu salt okunur ipucu yolunu ve mevcut 24 saatlik ipucu aralığını kullanır, ancak otomatik kurulumu, devretmeyi, yeniden başlatmayı, stable gecikmesini/rastgele sapmasını veya beta yoklamasını hiçbir zaman çağırmaz. Sürüm düşürme veya olay kurtarma için, `update.auto.enabled` yapılandırılmış olsa bile otomatik uygulamaları engellemek üzere Gateway ortamında `OPENCLAW_NO_AUTO_UPDATE=1` ayarlayın. `update.checkOnStart` de devre dışı bırakılmadığı sürece başlangıç güncelleme ipuçları çalışmaya devam edebilir.

Canlı Gateway kontrol düzlemi üzerinden istenen paket yöneticisi güncellemeleri
(`update.run`), çalışan Gateway işleminin içindeki paket ağacını değiştirmez.
Yönetilen hizmet kurulumlarında Gateway, ayrılmış bir devretme başlatır,
çıkar ve normal `openclaw update --yes --json` CLI yolunun hizmeti durdurmasına,
paketi değiştirmesine, hizmet meta verilerini yenilemesine, yeniden başlatmasına, Gateway
sürümünü ve erişilebilirliğini doğrulamasına ve mümkün olduğunda kurulmuş ancak yüklenmemiş bir macOS
LaunchAgent'ını kurtarmasına izin verir. Gateway bu devretmeyi güvenli bir şekilde gerçekleştiremezse,
`update.run` paket yöneticisini işlem içinde çalıştırmak yerine güvenli bir kabuk komutu
bildirir.

Control UI kenar çubuğundaki güncelleme kartı, bu `update.run` akışını
doğrudan başlatacağı zaman **Gateway'i Güncelle** seçeneğini gösterir. Bu, tarayıcıda barındırılan Control UI'ı, uzak
Gateway'leri ve elle yönetilen yerel Gateway'leri kapsar.

İmzalı macOS uygulamasında, uygulamanın sahip olduğu yerel bir Gateway bu kartı
**Mac uygulamasını + Gateway'i güncelle** olarak değiştirir. Sparkle önce uygulamayı günceller; yeniden başlatıldıktan sonra
uygulama `openclaw update --tag <app-version> --json` çalıştırır, Gateway'ini yeniden başlatır
ve kurulum tarzı bir ilerleme penceresinde sistem durumunu doğrular. Pencere yalnızca
bu yönetilen Gateway'in güncellenmesi, onarılması veya kurulması gerektiğinde görünür; yalnızca uygulama güncellemeleri
doğrudan uygulamaya yeniden başlatılır. Hata ayrıntıları Yeniden Dene, [Güncelleme kılavuzu](/tr/install/updating) ve
[Discord](https://discord.gg/clawd) eylemleriyle görünür kalır. Uygulama bu eşgüdümlü
yolu uzak veya haricen yönetilen bir Gateway için hiçbir zaman kullanmaz, daha yeni bir
Gateway'in sürümünü hiçbir zaman düşürmez ve bir `extended-stable` kanal sabitlemesini hiçbir zaman geçersiz kılmaz.

Güncelleme başarılı olduğunda uygulama, gerçek bir kullanıcı/kanal etkileşimi içeren
en son üst düzey doğrudan oturum için tek seferlik bir karşılama olayı kuyruğa alır. Cron çalıştırmaları,
Heartbeat'ler ve yalnızca arka plandaki oturum güncellemeleri bu seçimi değiştirmez. Uzak
modda uygulama yalnızca yerel Mac Node çalışma zamanını günceller ve olayı
yalnızca bağlı uzak Gateway en az uygulama kadar yeniyse gönderir.

## Güncellemeden sonra

<Steps>

### Doctor'ı çalıştırın

```bash
openclaw doctor
```

Yapılandırmayı taşır, DM politikalarını denetler ve Gateway sistem durumunu kontrol eder. Ayrıntılar: [Doctor](/tr/gateway/doctor)

### Gateway'i yeniden başlatın

```bash
openclaw gateway restart
```

### Doğrulayın

```bash
openclaw health
```

</Steps>

## Geri alma

Geri alma iki katmandan oluşur:

1. Geçerli durumu koruyarak eski OpenClaw kodunu yeniden kurun.
2. Yalnızca eski kod taşınmış bir yapılandırmayı veya veritabanını kullanamıyorsa güncelleme öncesi durumu geri yükleyin.

Yalnızca kodu geri alarak başlayın. Durumu geri yüklemek, yedeklemeden sonra yapılan
değişiklikleri siler.

### Güncellemeden önce: doğrulanmış bir yedek oluşturun

`openclaw update` güncelleme öncesi otomatik yapılandırma kopyasını korur, ancak
tam bir durum kurtarma noktası oluşturmaz. Önemli bir güncellemeden önce açıkça bir tane
oluşturun:

```bash
mkdir -p ~/Backups/openclaw
openclaw backup create --output ~/Backups/openclaw --verify
```

Arşiv manifesti, OpenClaw sürümünü ve yedeklemeye dahil edilen kaynak yolları
kaydeder. Arşiv kimlik bilgilerini, kimlik doğrulama profillerini ve kanal
durumunu içerebilir; bu nedenle arşivi yalnızca sahibin erişebildiği izinlerle ve
canlı durum diziniyle aynı koruma düzeyinde saklayın. Dahil edilen ve kasıtlı olarak
hariç tutulan dosyalar için [Yedekleme](/tr/cli/backup) bölümüne bakın.

Taşınabilir arşivin hariç tuttuğu geçici yapıtları içeren, bayt düzeyinde birebir bir
kurtarma noktası için Gateway'i durdurun ve platformunuzun sağladığı bir dosya sistemi,
birim veya VM anlık görüntüsünü kullanın.

### Bir paket kurulumunu geri alın

Yayımlanmış sürümleri listeleyin, ardından sorunsuz olduğu bilinen sürümü önizleyip kurun:

```bash
npm view openclaw versions --json
openclaw update --tag <known-good-version> --dry-run
openclaw update --tag <known-good-version>
```

Doğrudan paket yöneticisi kurulumuna kıyasla `openclaw update --tag` tercih edilir. Bu yol,
sürüm düşürmeyi algılar, onay ister, kurulu hedefe göre yönetilen Plugin yakınsamasını
ve uyumluluk denetimlerini çalıştırır, hizmet meta verilerini yeniler, Gateway'i
yeniden başlatır ve çalışan sürümü doğrular. Saklanan kanal `extended-stable` ise,
tek seferlik tam etiketler `extended-stable` seçicisiyle
birleştirilemediğinden `--channel stable --tag <known-good-version>` kullanın.

Paket güncellemeleri, adayı etkinleştirmeden önce hazırlar ve doğrular.
Dosya sistemi değiştirme işlemi veya komut aracısı değişimi başarısız olursa OpenClaw eski
paketi otomatik olarak geri yükler. Başarılı bir değişimden sonra Gateway sistem durumu
daha sonra başarısız olursa, paket yeniden otomatik olarak değiştirilmek yerine
önceki sürüm ve elle geri alma talimatları bildirilir.

CLI güncelleme yolu kullanılamıyorsa mevcut Gateway'in sahibi olan aynı paket yöneticisini
ve kurulum kapsamını kullanın:

```bash
openclaw gateway stop
npm i -g openclaw@<known-good-version>
openclaw gateway install --force
openclaw gateway restart
```

Kurulumun sahibi ilgili yönetici olduğunda `npm` yerine `pnpm` veya `bun` kullanın. Olay
kurtarma sırasında, etkin bir otomatik güncelleyicinin daha yeni bir sürümü hemen
uygulamasını önlemek için Gateway ortamında `OPENCLAW_NO_AUTO_UPDATE=1` ayarlayın.

### Bir kaynak kod teslim alma alanını geri alın

Temiz bir teslim alma alanı kullanın ve sorunsuz olduğu bilinen bir etiketi veya commit'i seçin:

```bash
git fetch --all --tags
git checkout --detach <known-good-tag-or-commit>
pnpm install && pnpm build
openclaw gateway restart
```

En son sürüme dönmek için: `git checkout main && git pull`.

Güncelleyici; bağımlılık kurulumu, derleme, UI derlemesi veya Doctor, bir git
güncellemesi başladıktan sonra başarısız olursa git teslim alma alanını otomatik olarak önceki dalına ve
SHA değerine döndürür. Daha eski bir commit'i kasıtlı olarak seçtiğinizde yine de
elle teslim alma gerekir.

### Oturum SQLite taşıması üzerinden sürüm düşürme

Dosya tabanlı eski bir OpenClaw sürümünü başlatmadan önce arşivlenmiş eski transkript
yapıtlarını geri yüklemek için geçerli CLI'ı kullanın:

```bash
openclaw gateway stop
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

Bu işlem SQLite verilerini silmez. SQLite taşımasından sonra oluşturulan oturumlar
yalnızca SQLite'ta bulunur ve eski çalışma zamanında görünmez. Bkz.
[Oturum SQLite taşımasından sonra sürüm düşürme](/tr/cli/doctor#downgrading-after-session-sqlite-migration).

### Durumu yalnızca gerektiğinde geri yükleyin

Eski kod daha yeni bir yapılandırmayı veya veritabanı şemasını okuyamıyorsa
Gateway'i durdurun ve doğrulanmış güncelleme öncesi dosya sistemi, birim veya VM anlık görüntüsünü geri yükleyin.
Geri yükleme, anlık görüntüden sonra yapılan değişiklikleri sileceğinden
önce geçerli durumu ayrı olarak koruyun.

Geniş kapsamlı `openclaw backup create` arşivleri oluşturmayı ve doğrulamayı destekler, ancak
tüm arşivin yerinde etkinleştirilmesini desteklemez. Geniş kapsamlı bir arşivi bir hazırlama
dizinine çıkarın ve çevrimdışı geri yükleme için arşivin `manifest.json` kaynaktan arşive
eşlemesini kullanın. Benzer şekilde `openclaw backup sqlite restore`, doğrulanmış bir veritabanını
yeni bir hedefe yazar; bu hedefi etkinleştirmek açık bir çevrimdışı operatör
adımı olarak kalır.

### Geri almayı doğrulayın

```bash
openclaw --version
openclaw health
openclaw plugins list --json
openclaw gateway status --deep --json
openclaw doctor --lint --json
```

## Takılırsanız

- `openclaw doctor` komutunu yeniden çalıştırın ve çıktıyı dikkatle okuyun.
- Kaynak kod teslim alma alanlarında `openclaw update --channel dev` için güncelleyici, gerektiğinde `pnpm` öğesini otomatik olarak önyükler. Bir pnpm/corepack önyükleme hatası görürseniz `pnpm` öğesini elle kurun (veya `corepack` öğesini yeniden etkinleştirin) ve güncellemeyi yeniden çalıştırın.
- Şuraya bakın: [Sorun giderme](/tr/gateway/troubleshooting)
- Discord'da sorun: [https://discord.gg/clawd](https://discord.gg/clawd)

## İlgili

- [Kuruluma genel bakış](/tr/install): tüm kurulum yöntemleri.
- [Doctor](/tr/gateway/doctor): güncellemelerden sonraki sistem durumu kontrolleri.
- [Taşıma](/tr/install/migrating): ana sürüm taşıma kılavuzları.
