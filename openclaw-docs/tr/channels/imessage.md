---
read_when:
    - iMessage desteğini ayarlama
    - iMessage gönderme/alma sorunlarını giderme
summary: Yanıtlar, tapback'ler, efektler, anketler, ekler ve grup yönetimine yönelik özel API eylemleriyle imsg üzerinden yerel iMessage desteği (stdio üzerinden JSON-RPC). Ana makine gereksinimleri uygunsa yeni OpenClaw iMessage kurulumları için tercih edilir.
title: iMessage
x-i18n:
    generated_at: "2026-07-26T23:12:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f3e8b1a65c76b25d03615c06a976f86a8af555cd96d5bfdb10cef9c955893ddc
    source_path: channels/imessage.md
    workflow: 16
---

<Note>
Standart OpenClaw iMessage dağıtımı için Gateway'i ve `imsg` öğesini aynı oturum açılmış macOS Messages ana bilgisayarında çalıştırın. Gateway'iniz başka bir yerde çalışıyorsa `channels.imessage.cliPath` öğesini, Mac'te `imsg` çalıştıran şeffaf bir SSH sarmalayıcısına yönlendirin.

**Gelen ileti kurtarma otomatiktir.** Bir köprü veya gateway yeniden başlatıldıktan sonra iMessage, devre dışı kaldığı sırada kaçırılan iletileri yeniden oynatır ve Apple'ın Push kurtarmasından sonra boşaltabileceği eski "birikmiş ileti bombasını" engeller; hiçbir şeyin iki kez gönderilmemesi için yinelenenleri ayıklar. Etkinleştirilecek bir yapılandırma yoktur — bkz. [Bir köprü veya gateway yeniden başlatıldıktan sonra gelen ileti kurtarma](#inbound-recovery-after-a-bridge-or-gateway-restart).
</Note>

<Warning>
BlueBubbles desteği kaldırıldı. `channels.bluebubbles` yapılandırmalarını `channels.imessage` biçimine taşıyın; OpenClaw, iMessage'ı yalnızca `imsg` üzerinden destekler. Kısa duyuru için [BlueBubbles'ın kaldırılması ve imsg iMessage yolu](/tr/announcements/bluebubbles-imessage) ile, tam geçiş tablosu içinse [BlueBubbles'tan geçiş](/tr/channels/imessage-from-bluebubbles) ile başlayın.
</Warning>

Durum: yerel harici CLI entegrasyonu. Gateway, `imsg rpc` işlemini başlatır ve stdio üzerinden JSON-RPC iletişimi kurar; ayrı bir daemon veya port yoktur. Eksiksiz bir iMessage kanalı için özel API modu önemle önerilir; yanıtlar, tapback'ler, efektler, anketler, eklere verilen yanıtlar ve grup eylemleri için `imsg launch` ve başarılı bir özel API yoklaması gerekir.

Yaygın yerel kurulumda OpenClaw kurulumu, oturum açılmış Messages Mac'inde `imsg` için kullanıcı tarafından onaylanan bir Homebrew kurulumu veya güncellemesi sunabilir. Manuel kurulum ve SSH sarmalayıcılı topolojiler operatör tarafından yönetilmeye devam eder: `imsg` öğesini Gateway'i veya sarmalayıcıyı çalıştıracak aynı kullanıcı bağlamında kurun ya da güncelleyin.

<CardGroup cols={3}>
  <Card title="Özel API eylemleri" icon="wand-sparkles" href="#private-api-actions">
    Yanıtlar, tapback'ler, efektler, anketler, ekler ve grup yönetimi.
  </Card>
  <Card title="Eşleştirme" icon="link" href="/tr/channels/pairing">
    iMessage doğrudan iletileri varsayılan olarak eşleştirme modunu kullanır.
  </Card>
  <Card title="Uzak Mac" icon="terminal" href="#remote-mac-over-ssh">
    Gateway, Messages Mac'inde çalışmıyorsa bir SSH sarmalayıcısı kullanın.
  </Card>
  <Card title="Yapılandırma başvurusu" icon="settings" href="/tr/gateway/config-channels#imessage">
    iMessage alanlarının tam başvurusu.
  </Card>
</CardGroup>

## Hızlı kurulum

<Tabs>
  <Tab title="Yerel Mac (hızlı yol)">
    <Steps>
      <Step title="imsg'yi kurun ve doğrulayın">

```bash
brew install steipete/tap/imsg
brew update && brew upgrade imsg
imsg rpc --help
imsg launch
openclaw channels status --probe
```

        Yerel kurulum sihirbazı eksik bir varsayılan `imsg` komutu algıladığında, `steipete/tap/imsg` öğesini Homebrew aracılığıyla kurmayı önerebilir. Homebrew tarafından yönetilen bir `imsg` algılarsa yeniden kurmayı veya güncellemeyi önerebilir. Özel `cliPath` sarmalayıcıları değiştirilmez.

      </Step>

      <Step title="OpenClaw'ı yapılandırın">

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/user/Library/Messages/chat.db",
    },
  },
}
```

      </Step>

      <Step title="Gateway'i başlatın">

```bash
openclaw gateway
```

      </Step>

      <Step title="İlk doğrudan ileti eşleştirmesini onaylayın (varsayılan dmPolicy)">

```bash
openclaw pairing list imessage
openclaw pairing approve imessage <CODE>
```

        Eşleştirme isteklerinin süresi 1 saat sonra dolar.
      </Step>
    </Steps>

  </Tab>

  <Tab title="SSH üzerinden uzak Mac">
    Çoğu kurulum SSH gerektirmez. Bu topolojiyi yalnızca Gateway oturum açılmış Messages Mac'inde çalışamıyorsa kullanın. OpenClaw yalnızca stdio uyumlu bir `cliPath` gerektirir; bu nedenle `cliPath` öğesini, uzak bir Mac'e SSH bağlantısı kurup `imsg` çalıştıran bir sarmalayıcı betiğine yönlendirebilirsiniz.
    `imsg` öğesini Gateway ana bilgisayarına değil, bu uzak Mac'e kurun ve orada güncelleyin:

```bash
ssh messages-mac 'brew install steipete/tap/imsg && brew update && brew upgrade imsg'
```

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    Ekler etkinleştirildiğinde önerilen yapılandırma:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "user@gateway-host", // SCP ek getirmeleri için kullanılır
      includeAttachments: true,
      // İsteğe bağlı: izin verilen ek kökleri (varsayılan
      // /Users/*/Library/Messages/Attachments ile birleştirilir).
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
    },
  },
}
```

    `remoteHost` ayarlanmamışsa OpenClaw, SSH sarmalayıcı betiğini ayrıştırarak bunu otomatik algılamaya çalışır.
    `remoteHost`, `host` veya `user@host` olmalıdır (boşluk veya SSH seçeneği içeremez); güvenli olmayan değerler yok sayılır.
    OpenClaw, SCP için katı ana bilgisayar anahtarı denetimi kullanır; bu nedenle aktarma ana bilgisayarının anahtarı `~/.ssh/known_hosts` içinde zaten bulunmalıdır.
    Ek yolları izin verilen köklere (`attachmentRoots` / `remoteAttachmentRoots`) göre doğrulanır.

<Warning>
`imsg` önüne yerleştirdiğiniz herhangi bir `cliPath` sarmalayıcısı veya SSH proxy'si, uzun ömürlü JSON-RPC için şeffaf bir stdio borusu gibi davranmalıdır. OpenClaw, kanalın ömrü boyunca sarmalayıcının stdin/stdout'u üzerinden yeni satırlarla çerçevelenmiş küçük JSON-RPC iletileri alışverişi yapar:

- Her stdin parçasını/satırını **baytlar kullanılabilir olur olmaz** iletin; EOF'yi beklemeyin.
- Her stdout parçasını/satırını ters yönde hemen iletin.
- Yeni satırları koruyun.
- Küçük çerçeveleri bekletebilecek sabit boyutlu engelleyici okumalardan (`read(4096)`, `cat | buffer`, varsayılan kabuk `read`) kaçının.
- stderr'i JSON-RPC stdout akışından ayrı tutun.

Büyük bir blok dolana kadar stdin'i arabelleğe alan bir sarmalayıcı, `imsg rpc` sağlıklı olsa bile iMessage kesintisine benzeyen belirtilere — `imsg rpc timeout (chats.list)` veya tekrarlanan kanal yeniden başlatmaları — yol açar. `ssh -T host imsg "$@"` (yukarıdaki) güvenlidir; çünkü OpenClaw'ın `rpc` ve `--db` gibi `cliPath` bağımsız değişkenlerini iletir. `ssh host imsg | grep -v '^DEBUG'` gibi işlem hatları güvenli DEĞİLDİR; satır arabellekli araçlar yine de çerçeveleri bekletebilir. Filtreleme yapmanız gerekiyorsa her aşamada `stdbuf -oL -eL` kullanın.
</Warning>

  </Tab>
</Tabs>

## Gereksinimler ve izinler (macOS)

- Messages uygulamasında, `imsg` çalıştıran Mac'te oturum açılmış olmalıdır.
- OpenClaw/`imsg` çalıştıran işlem bağlamı için Tam Disk Erişimi gerekir (Messages veritabanı erişimi).
- Messages.app üzerinden ileti göndermek için Otomasyon izni gerekir.
- Gelişmiş eylemler (tepki / düzenleme / göndermeyi geri alma / ileti dizili yanıt / efektler / anketler / grup işlemleri) için Sistem Bütünlüğü Koruması devre dışı bırakılmalıdır — bkz. [imsg özel API'sini etkinleştirme](#enabling-the-imsg-private-api). Temel metin ve medya gönderme/alma işlemleri onsuz çalışır.

<Tip>
İzinler işlem bağlamı başına verilir. Gateway başsız olarak (LaunchAgent/SSH) çalışıyorsa istemleri tetiklemek için aynı bağlamda tek seferlik etkileşimli bir komut çalıştırın:

```bash
imsg chats --limit 1
# veya
imsg send <handle> "test"
```

</Tip>

<Accordion title="SSH sarmalayıcısıyla göndermeler AppleEvents -1743 hatasıyla başarısız oluyor">
  Uzak SSH kurulumu sohbetleri okuyabilir, `channels status --probe` işlemini geçebilir ve gelen iletileri işleyebilirken giden gönderimler yine de bir AppleEvents yetkilendirme hatasıyla başarısız olabilir:

```text
Messages'a Apple olayları göndermek için yetkiniz yok. (-1743)
```

Oturum açmış Mac kullanıcısının TCC veritabanını veya System Settings > Privacy & Security > Automation yolunu denetleyin. Otomasyon girdisi `imsg` veya yerel kabuk işlemi yerine `/usr/libexec/sshd-keygen-wrapper` için kaydedilmişse macOS, bu SSH sunucusu tarafındaki istemci için kullanılabilir bir Messages anahtarı sunmayabilir:

```text
kTCCServiceAppleEvents | /usr/libexec/sshd-keygen-wrapper | auth_value=0 | com.apple.MobileSMS
```

Bu durumda `tccutil reset AppleEvents` işlemini tekrarlamak veya `imsg send` öğesini aynı SSH sarmalayıcısı üzerinden yeniden çalıştırmak başarısız olmaya devam edebilir; çünkü Messages Otomasyonu'na ihtiyaç duyan işlem bağlamı, arayüzün izin verebileceği bir uygulama değil SSH sarmalayıcısıdır.

Bunun yerine desteklenen `imsg` işlem bağlamlarından birini kullanın:

- Gateway'i veya en azından `imsg` köprüsünü, Messages kullanıcısının oturum açmış yerel oturumunda çalıştırın.
- Aynı oturumdan Tam Disk Erişimi ve Otomasyon izni verdikten sonra Gateway'i bu kullanıcı için bir LaunchAgent ile başlatın.
- İki kullanıcılı SSH topolojisini koruyorsanız kanalı etkinleştirmeden önce gerçek bir giden `imsg send` işleminin tam olarak kullanılan sarmalayıcı üzerinden başarılı olduğunu doğrulayın. Otomasyon izni verilemiyorsa gönderimler için SSH sarmalayıcısına güvenmek yerine tek kullanıcılı bir `imsg` kurulumuna geçin.

</Accordion>

## imsg özel API'sini etkinleştirme

`imsg` iki çalışma moduyla sunulur. OpenClaw için önerilen kurulum Özel API modudur; çünkü bu mod kanala kullanıcıların beklediği yerel iMessage eylemlerini sağlar. Temel mod; düşük riskli kurulumlar, ilk doğrulama veya SIP'in devre dışı bırakılamadığı ana bilgisayarlar için kullanışlı olmaya devam eder.

- **Temel mod** (varsayılan, SIP değişikliği gerekmez): `send` üzerinden giden metin ve medya, gelen ileti izleme/geçmişi ve sohbet listesi. Yeni bir `brew install steipete/tap/imsg` kurulumu ve yukarıdaki standart macOS izinleriyle kullanıma hazır olarak elde edilen budur.
- **Özel API modu**: `imsg`, dahili `IMCore` işlevlerini çağırmak için `Messages.app` içine yardımcı bir dylib enjekte eder. Bu; `react`, `edit`, `unsend`, `reply` (ileti dizili), `sendWithEffect`, `poll` ve `poll-vote` (yerel Messages anketleri), `renameGroup`, `setGroupIcon`, `addParticipant`, `removeParticipant`, `leaveGroup` özelliklerinin yanı sıra yazma göstergelerini ve okundu bilgilerini etkinleştirir.

Bu sayfada önerilen eylem yüzeyi Özel API modunu gerektirir. `imsg` README dosyası bu gereksinimi açıkça belirtir:

> `read`, `typing`, `launch`, köprü destekli zengin gönderim, ileti değiştirme ve sohbet yönetimi gibi gelişmiş özellikler isteğe bağlıdır. Bunlar, SIP'in devre dışı bırakılmasını ve `Messages.app` içine yardımcı bir dylib enjekte edilmesini gerektirir. SIP etkinse `imsg launch` enjeksiyon yapmayı reddeder.

Yardımcı enjeksiyon tekniği, Messages özel API'lerine erişmek için `imsg` öğesinin kendi dylib'ini kullanır. OpenClaw iMessage yolunda üçüncü taraf bir sunucu veya BlueBubbles çalışma zamanı yoktur.

<Warning>
**SIP'i devre dışı bırakmak gerçek bir güvenlik ödünleşimidir.** SIP, değiştirilmiş sistem kodunun çalıştırılmasına karşı macOS'in temel korumalarından biridir; sistem genelinde kapatılması ek saldırı yüzeyi ve yan etkiler oluşturur. Özellikle **Apple Silicon Mac'lerde SIP'in devre dışı bırakılması, iOS uygulamalarını Mac'inize kurma ve çalıştırma olanağını da devre dışı bırakır**.

Özellikle birincil kişisel Mac'te bunu bilinçli bir operasyonel seçim olarak ele alın. Üretim kalitesinde OpenClaw iMessage için köprüyü etkinleştirme konusunda rahat olduğunuz ayrılmış bir Mac'i veya bot macOS kullanıcısını tercih edin. Tehdit modeliniz SIP'in herhangi bir yerde kapalı olmasını kabul edemiyorsa paketlenmiş iMessage temel modla sınırlıdır: yalnızca metin ve medya gönderme/alma; tepki / düzenleme / göndermeyi geri alma / efektler / grup işlemleri yoktur.
</Warning>

### Kurulum

1. Messages.app çalıştıran Mac'te **`imsg` öğesini kurun (veya yükseltin)**:

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg status --json
   ```

   `imsg status --json` çıktısı `bridge_version`, `rpc_methods` ve yöntem başına `selectors` bilgilerini bildirir; böylece başlamadan önce mevcut derlemenin neleri desteklediğini görebilirsiniz.

2. **Sistem Bütünlüğü Koruması'nı ve (modern macOS'te) Kitaplık Doğrulaması'nı devre dışı bırakın.** Apple imzalı `Messages.app` içine Apple'a ait olmayan bir yardımcı dylib enjekte etmek için SIP'in kapalı **ve** kitaplık doğrulamasının gevşetilmiş olması gerekir. Kurtarma modundaki SIP adımı macOS sürümüne özeldir:
   - **macOS 10.13-10.15 (Sierra-Catalina):** Terminal aracılığıyla Kitaplık Doğrulaması'nı devre dışı bırakın, Kurtarma Modu'nda yeniden başlatın, `csrutil disable` komutunu çalıştırın ve yeniden başlatın.
   - **macOS 11+ (Big Sur ve sonrası), Intel:** Kurtarma Modu'na (veya İnternet Kurtarma'ya) girin, `csrutil disable` komutunu çalıştırın ve yeniden başlatın.
   - **macOS 11+, Apple Silicon:** Kurtarma'ya girmek için güç düğmesiyle başlatma sırasını uygulayın; güncel macOS sürümlerinde Continue seçeneğine tıklarken **Left Shift** tuşunu basılı tutun, ardından `csrutil disable` komutunu çalıştırın. Sanal makine kurulumları ayrı bir akış izlediğinden önce bir VM anlık görüntüsü alın.

   **macOS 11 ve sonrasında yalnızca `csrutil disable` genellikle yeterli değildir.** Apple, bir platform ikili dosyası olan `Messages.app` için kitaplık doğrulamasını uygulamaya devam eder; bu nedenle geçici imzalanmış bir yardımcı, SIP kapalı olsa bile reddedilir (`Library Validation failed: ... platform binary, but mapped file is not`). SIP'i devre dışı bıraktıktan sonra kitaplık doğrulamasını da devre dışı bırakıp yeniden başlatın:

   ```bash
   sudo defaults write /Library/Preferences/com.apple.security.libraryvalidation.plist DisableLibraryValidation -bool true
   ```

   **macOS 26 (Tahoe), 26.5.1 üzerinde doğrulandı:** SIP'in kapalı olmasıyla birlikte yukarıdaki `DisableLibraryValidation` komutu, yardımcıyı 26.0 ile 26.5.x arasındaki sürümlerin tamamında enjekte etmek için yeterlidir. **Hiçbir boot-arg gerekmez.** Plist belirleyici etkendir ve Tahoe'da enjeksiyon başarısız olduğunda en sık eksik olan adımdır:
   - **Plist varken:** `imsg launch` enjeksiyonu gerçekleştirir ve `imsg status`, `advanced_features: true` bildirir.
   - **Plist olmadan (SIP kapalı olsa bile):** `imsg launch`, `Failed to launch: Timeout waiting for Messages.app to initialize` hatasıyla başarısız olur. AMFI, geçici imzalanmış yardımcıyı yükleme sırasında reddettiğinden köprü hiçbir zaman hazır duruma gelmez ve başlatma zaman aşımına uğrar. Bu zaman aşımı, Tahoe'da çoğu kişinin karşılaştığı belirtidir; çözüm daha köklü bir işlem değil, yukarıdaki plist'tir.

   Bir macOS yükseltmesinden sonra `imsg launch` enjeksiyonu veya belirli `selectors` değerleri false döndürmeye başlarsa olağan neden bu geçittir. SIP adımının başarısız olduğunu varsaymadan önce SIP ve kitaplık doğrulaması durumunuzu kontrol edin. Bu ayarlar doğru olduğu hâlde köprü hâlâ enjeksiyon yapamıyorsa `imsg status --json` ile `imsg launch` çıktısını toplayın ve sistem genelindeki ek güvenlik denetimlerini zayıflatmak yerine durumu `imsg` projesine bildirin.

3. **Yardımcıyı enjekte edin.** SIP devre dışıyken ve Messages.app oturumu açıkken:

   ```bash
   imsg launch
   ```

   `imsg launch`, SIP hâlâ etkinken enjeksiyonu reddeder; dolayısıyla bu işlem 2. adımın uygulandığını da doğrular.

4. **Köprüyü OpenClaw üzerinden doğrulayın:**

   ```bash
   openclaw channels status --probe
   ```

   iMessage girdisi `works` bildirmeli ve `imsg status --json | jq '{rpc_methods, selectors}'`, macOS derlemenizin sunduğu yetenekleri göstermelidir. Anket oluşturmak için `selectors.pollPayloadMessage`; oy vermek için hem `selectors.pollVoteMessage` hem de `poll.vote` RPC yöntemi gerekir. OpenClaw plugin'i yalnızca önbelleğe alınmış yoklamanın desteklediği eylemleri duyururken boş bir önbellek iyimser kalır ve ilk gönderimde yoklama yapar.

`openclaw channels status --probe`, kanalı `works` olarak bildiriyor ancak belirli eylemler gönderim sırasında "iMessage `<action>` requires the imsg private API bridge" hatası veriyorsa `imsg launch` komutunu yeniden çalıştırın — yardımcı devreden çıkabilir (Messages.app'in yeniden başlatılması, işletim sistemi güncellemesi vb.) ve önbelleğe alınmış `available: true` durumu, bir sonraki yoklama yenilenene kadar eylemleri duyurmaya devam eder.

### SIP etkin kaldığında

SIP'i devre dışı bırakmak tehdit modeliniz açısından kabul edilebilir değilse:

- `imsg` temel moda geri döner — yalnızca metin + medya + alma.
- OpenClaw plugin'i metin/medya gönderimini ve gelen ileti izlemeyi duyurmaya devam eder; `react`, `edit`, `unsend`, `reply`, `sendWithEffect` ve grup işlemlerini eylem yüzeyinden gizler (yöntem başına yetenek geçidine göre).
- Birincil cihazlarınızda SIP'i etkin tutarken iMessage iş yükü için SIP'i kapalı ayrı bir Apple Silicon olmayan Mac (veya özel bir bot Mac'i) çalıştırabilirsiniz. Aşağıdaki [Özel bot macOS kullanıcısı (ayrı iMessage kimliği)](#deployment-patterns) bölümüne bakın.

## Erişim denetimi ve yönlendirme

<Tabs>
  <Tab title="DM ilkesi">
    `channels.imessage.dmPolicy` doğrudan mesajları denetler:

    - `pairing` (varsayılan)
    - `allowlist` (en az bir `allowFrom` girdisi gerektirir)
    - `open` (`allowFrom` öğesinin `"*"` içermesini gerektirir)
    - `disabled`

    İzin listesi alanı: `channels.imessage.allowFrom`.

    İzin listesi girdileri gönderenleri tanımlamalıdır: tanıtıcılar veya statik gönderen erişim grupları (`accessGroup:<name>`). `chat_id:*`, `chat_guid:*` veya `chat_identifier:*` gibi sohbet hedefleri için `channels.imessage.groupAllowFrom`; sayısal `chat_id` kayıt defteri anahtarları için `channels.imessage.groups` kullanın.

  </Tab>

  <Tab title="Grup ilkesi + bahsetmeler">
    `channels.imessage.groupPolicy` grup işlemeyi denetler:

    - `allowlist` (varsayılan)
    - `open`
    - `disabled`

    Grup gönderen izin listesi: `channels.imessage.groupAllowFrom`.

    `groupAllowFrom` girdileri statik gönderen erişim gruplarına da başvurabilir (`accessGroup:<name>`).

    Çalışma zamanı geri dönüşü: `groupAllowFrom` ayarlanmamışsa iMessage grup göndereni denetimleri `allowFrom` kullanır; DM ve grup kabulü farklı olmalıysa `groupAllowFrom` ayarını yapın. Açıkça boş bir `groupAllowFrom: []` geri dönüş yapmaz — `allowlist` kapsamında tüm grup gönderenlerini engeller.
    Çalışma zamanı notu: `channels.imessage` tamamen eksikse çalışma zamanı `groupPolicy="allowlist"` değerine geri döner ve bir uyarı günlüğe kaydeder (`channels.defaults.groupPolicy` ayarlanmış olsa bile).

    <Warning>
    `groupPolicy: "allowlist"` kapsamındaki grup yönlendirmesi art arda **iki** geçit çalıştırır:

    1. **Gönderen izin listesi** (`channels.imessage.groupAllowFrom`) — tanıtıcı, `accessGroup:<name>`, `chat_guid`, `chat_identifier` veya `chat_id`. Boş bir etkin liste (`groupAllowFrom` ve `allowFrom` geri dönüşü yoksa) tüm grup gönderenlerini engeller.
    2. **Grup kayıt defteri** (`channels.imessage.groups`) — eşlemede girdiler olduğunda uygulanır: sohbet, açık bir `chat_id` başına girdiyle veya `groups: { "*": { ... } }` joker karakteriyle eşleşmelidir. `groups` boş veya eksik olduğunda kabulü yalnızca gönderen izin listesi belirler.

    Etkin bir grup gönderen izin listesi yapılandırılmamışsa her grup mesajı kayıt defteri geçidinden önce bırakılır. Her geçit, varsayılan günlük düzeyinde kendi `warn` düzeyi sinyaline sahiptir ve her biri farklı bir çözümü belirtir:

    - etkin grup gönderen izin listesi boş olduğunda başlangıçta hesap başına bir kez: `imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...` — `channels.imessage.groupAllowFrom` (veya `allowFrom`) ayarını yaparak düzeltin; yalnızca `groups` girdileri eklemek, 1. geçidin tüm gönderenleri engellemeye devam etmesine yol açar.
    - bir gönderen 1. geçidi geçtiği ancak sohbet doldurulmuş bir `groups` kayıt defterinde bulunmadığı zaman çalışma sırasında `chat_id` başına bir kez: `imessage: dropping group message from chat_id=<id> ...` — ilgili `chat_id` (veya `"*"`) değerini `channels.imessage.groups` altına ekleyerek düzeltin.

    DM'ler etkilenmez — farklı bir kod yolunu kullanırlar.

    `groupPolicy: "allowlist"` kapsamındaki grup akışı için önerilen yapılandırma:

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: { "*": { "requireMention": true } },
        },
      },
    }
    ```

    Yalnızca `groupAllowFrom`, bu gönderenleri herhangi bir grupta kabul eder; izin verilen sohbetleri sınırlandırmak (ve `requireMention` gibi sohbet başına seçenekleri ayarlamak) için `groups` bloğunu ekleyin.
    </Warning>

    Gruplar için bahsetme geçidi:

    - iMessage'da yerel bahsetme meta verisi yoktur
    - bahsetme algılama, regex kalıplarını kullanır (`agents.entries.*.groupChat.mentionPatterns`, geri dönüş `messages.groupChat.mentionPatterns`)
    - yapılandırılmış kalıp yoksa bahsetme geçidi uygulanamaz
    - yetkili gönderenlerin denetim komutları bahsetme geçidini atlar

    Grup başına `systemPrompt`:

    `channels.imessage.groups.*` altındaki her girdi, söz konusu gruptaki bir mesajı işleyen her turda aracının sistem istemine eklenen isteğe bağlı bir `systemPrompt` dizesini kabul eder. Çözümleme, `channels.whatsapp.groups` davranışını yansıtır:

    1. **Gruba özgü sistem istemi** (`groups["<chat_id>"].systemPrompt`): belirli grup girdisi eşlemede mevcut **ve** `systemPrompt` anahtarı tanımlı olduğunda kullanılır. `systemPrompt` boş bir dizeyse (`""`) joker karakter bastırılır ve bu gruba hiçbir sistem istemi uygulanmaz.
    2. **Grup joker karakteri sistem istemi** (`groups["*"].systemPrompt`): belirli grup girdisi eşlemede hiç bulunmadığında veya bulunduğu hâlde `systemPrompt` anahtarı tanımlamadığında kullanılır.

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: {
            "*": { systemPrompt: "Britanya yazımını kullanın." },
            "8421": {
              requireMention: true,
              systemPrompt: "Bu, nöbet rotasyonu sohbetidir. Yanıtları 3 cümlenin altında tutun.",
            },
            "9907": {
              // açık bastırma: "Britanya yazımını kullanın." joker karakteri burada uygulanmaz
              systemPrompt: "",
            },
          },
        },
      },
    }
    ```

    Grup başına istemler yalnızca grup mesajlarına uygulanır — doğrudan mesajlar etkilenmez.

  </Tab>

  <Tab title="Oturumlar ve belirlenimci yanıtlar">
    - DM'ler doğrudan yönlendirmeyi, gruplar grup yönlendirmesini kullanır.
    - Varsayılan `session.dmScope=main` ile iMessage DM'leri aracının ana oturumunda birleştirilir.
    - Grup oturumları yalıtılmıştır (`agent:<agentId>:imessage:group:<chat_id>`).
    - Yanıtlar, kaynak kanal/hedef meta verileri kullanılarak iMessage'a geri yönlendirilir.

    Grup benzeri ileti dizisi davranışı:

    Birden fazla katılımcının bulunduğu bazı iMessage ileti dizileri `is_group=false` ile gelebilir.
    Bu `chat_id`, `channels.imessage.groups` altında açıkça yapılandırılmışsa OpenClaw bunu grup trafiği olarak işler (grup geçidi + grup oturumu yalıtımı).

  </Tab>
</Tabs>

## ACP konuşma bağlamaları

iMessage sohbetleri ACP oturumlarına bağlanabilir.

Hızlı operatör akışı:

- DM veya izin verilen grup sohbeti içinde `/acp spawn codex --bind here` komutunu çalıştırın.
- Aynı iMessage konuşmasındaki sonraki mesajlar, oluşturulan ACP oturumuna yönlendirilir.
- `/new` ve `/reset`, aynı bağlı ACP oturumunu yerinde sıfırlar.
- `/acp close`, ACP oturumunu kapatır ve bağlamayı kaldırır.

Yapılandırılmış kalıcı bağlamalar, `type: "acp"` ve `match.channel: "imessage"` içeren üst düzey `bindings[]` girdilerini kullanır.

`match.peer.id` şunları kullanabilir:

- `+15555550123` veya `user@example.com` gibi normalleştirilmiş DM tanıtıcısı
- `chat_id:<id>` (kararlı grup bağlamaları için önerilir)
- `chat_guid:<guid>`
- `chat_identifier:<identifier>`

Örnek:

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: { agent: "codex", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "imessage",
        accountId: "default",
        peer: { kind: "group", id: "chat_id:123" },
      },
      acp: { label: "codex-group" },
    },
  ],
}
```

Paylaşılan ACP bağlama davranışı için [ACP Aracıları](/tr/tools/acp-agents) bölümüne bakın.

## Dağıtım kalıpları

<AccordionGroup>
  <Accordion title="Özel bot macOS kullanıcısı (ayrı iMessage kimliği)">
    Bot trafiğini kişisel Messages profilinizden yalıtmak için özel bir Apple ID ve macOS kullanıcısı kullanın.

    Tipik akış:

    1. Özel bir macOS kullanıcısı oluşturun/bu kullanıcıyla oturum açın.
    2. Bu kullanıcıda bot Apple ID'siyle Messages'a giriş yapın.
    3. Bu kullanıcıda `imsg` öğesini yükleyin.
    4. OpenClaw'ın `imsg` öğesini bu kullanıcı bağlamında çalıştırabilmesi için bir SSH sarmalayıcısı oluşturun.
    5. `channels.imessage.accounts.<id>.cliPath` ve `.dbPath` öğelerini bu kullanıcı profiline yönlendirin.

    İlk çalıştırmada bu bot kullanıcısının oturumunda GUI onayları (Automation + Full Disk Access) gerekebilir.

  </Accordion>

  <Accordion title="Tailscale üzerinden uzak Mac (örnek)">
    Yaygın topoloji:

    - Gateway Linux/VM üzerinde çalışır
    - iMessage + `imsg` tailnet'inizdeki bir Mac üzerinde çalışır
    - `cliPath` sarmalayıcısı, `imsg` öğesini çalıştırmak için SSH kullanır
    - `remoteHost`, SCP ile eklerin alınmasını etkinleştirir

    Örnek:

    ```json5
    {
      channels: {
        imessage: {
          enabled: true,
          cliPath: "~/.openclaw/scripts/imsg-ssh",
          remoteHost: "bot@mac-mini.tailnet-1234.ts.net",
          includeAttachments: true,
          dbPath: "/Users/bot/Library/Messages/chat.db",
        },
      },
    }
    ```

    ```bash
    #!/usr/bin/env bash
    exec ssh -T bot@mac-mini.tailnet-1234.ts.net imsg "$@"
    ```

    Hem SSH'nin hem de SCP'nin etkileşimsiz olması için SSH anahtarları kullanın.
    `known_hosts` öğesinin doldurulması için önce ana makine anahtarının güvenilir olduğundan emin olun (örneğin `ssh bot@mac-mini.tailnet-1234.ts.net`).

  </Accordion>

  <Accordion title="Çoklu hesap kalıbı">
    iMessage, `channels.imessage.accounts` altında hesap başına yapılandırmayı destekler.

    Her hesap; `cliPath`, `dbPath`, `allowFrom`, `groupPolicy`, `mediaMaxMb`, geçmiş ayarları ve ek kök izin listeleri gibi alanları geçersiz kılabilir.

  </Accordion>

  <Accordion title="Doğrudan mesaj geçmişi">
    Yeni doğrudan mesaj oturumlarını ilgili konuşmanın yakın zamanda kodu çözülmüş `imsg` geçmişiyle başlatmak için `channels.imessage.dmHistoryLimit` değerini ayarlayın. Bir gönderen için geçmişi devre dışı bırakan `0` dahil olmak üzere gönderen başına geçersiz kılmalar için `channels.imessage.dms["<sender>"].historyLimit` kullanın.

    iMessage DM geçmişi, gerektiğinde `imsg` kaynağından alınır. `dmHistoryLimit` değerinin ayarlanmaması, genel DM geçmişiyle başlangıç verisi sağlamayı devre dışı bırakır; ancak gönderen başına pozitif bir `channels.imessage.dms["<sender>"].historyLimit` değeri, ilgili gönderen için başlangıç verisi sağlamayı yine de etkinleştirir.

  </Accordion>
</AccordionGroup>

## Medya, parçalara ayırma ve teslim hedefleri

<AccordionGroup>
  <Accordion title="Ekler ve medya">
    - gelen eklerin içe alınması **varsayılan olarak kapalıdır** — fotoğrafları, sesli notları, videoları ve diğer ekleri ajana iletmek için `channels.imessage.includeAttachments: true` değerini ayarlayın. Bu özellik devre dışıyken yalnızca ek içeren iMessage'lar ajana ulaşmadan bırakılır ve hiçbir `Inbound message` günlük satırı oluşturulmayabilir.
    - `remoteHost` ayarlandığında uzak ek yolları SCP üzerinden alınabilir
    - ek yolları izin verilen köklerle eşleşmelidir:
      - `channels.imessage.attachmentRoots` (yerel)
      - `channels.imessage.remoteAttachmentRoots` (uzak SCP modu)
      - yapılandırılmış kökler varsayılan `/Users/*/Library/Messages/Attachments` kök kalıbını genişletir (değiştirilmez, birleştirilir)
    - SCP, katı ana makine anahtarı denetimi kullanır (`StrictHostKeyChecking=yes`)
    - giden medya boyutu `channels.imessage.mediaMaxMb` değerini kullanır (varsayılan 16 MB)

  </Accordion>

  <Accordion title="Giden metin ve parçalara ayırma">
    - metin parçası sınırı: `channels.imessage.textChunkLimit` (varsayılan 4000)
    - parçalama modu: `channels.imessage.streaming.chunkMode`
      - `length` (varsayılan)
      - `newline` (önce paragrafa göre bölme)
    - giden Markdown kalın/italik/altı çizili/üstü çizili biçimlendirmesi yerel biçimlendirilmiş metne dönüştürülür (macOS 15+ alıcıları biçimlendirmeyi görüntüler; daha eski alıcılar işaretçiler olmadan düz metin görür); Markdown tabloları kanalın Markdown tablo moduna göre dönüştürülür
    - `channels.imessage.sendTransport` (`auto` varsayılan, `bridge`, `applescript`), `imsg` öğesinin gönderimleri nasıl teslim edeceğini seçer

  </Accordion>

  <Accordion title="Adresleme biçimleri">
    Tercih edilen açık hedefler:

    - `chat_id:123` (kararlı yönlendirme için önerilir)
    - `chat_guid:...`
    - `chat_identifier:...`

    Tanıtıcı hedefleri de desteklenir:

    - `imessage:+1555...`
    - `sms:+1555...`
    - `user@example.com`

    ```bash
    imsg chats --limit 20
    ```

  </Accordion>
</AccordionGroup>

## Özel API eylemleri

`imsg launch` çalışırken ve `openclaw channels status --probe`, `privateApi.available: true` bildirdiğinde mesaj aracı, normal metin gönderimlerine ek olarak iMessage'a özgü eylemleri kullanabilir.

Tüm eylemler varsayılan olarak etkindir; ayrı ayrı eylemleri kapatmak için `channels.imessage.actions` kullanın:

```json5
{
  channels: {
    imessage: {
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
        renameGroup: true,
        setGroupIcon: true,
        addParticipant: true,
        removeParticipant: true,
        leaveGroup: true,
        polls: true,
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Kullanılabilir eylemler">
    - **tepki verme**: iMessage tapback'leri ekleyin/kaldırın (`messageId`, `emoji`, `remove`). Desteklenen tapback'ler sevgi, beğenme, beğenmeme, gülme, vurgulama ve soru tepkilerine eşlenir. Emoji belirtmeden kaldırma işlemi, ayarlanmış olan tapback'i temizler.
    - **yanıtlama**: Mevcut bir mesaja ileti dizili yanıt gönderin (`messageId`, `text` veya `message` ve ayrıca `chatGuid`, `chatId`, `chatIdentifier` veya `to`). Ek içeren yanıt için ayrıca `--file` desteğine sahip bir `send-rich` içeren `imsg` derlemesi gerekir.
    - **efektle gönderme**: iMessage efektiyle metin gönderin (`text` veya `message`, `effect` veya `effectId`). Kısa adlar: slam, loud, gentle, invisibleink, confetti, lasers, fireworks, balloon, heart, echo, happybirthday, shootingstar, sparkles, spotlight.
    - **düzenleme**: Desteklenen macOS/özel API sürümlerinde gönderilmiş bir mesajı düzenleyin (`messageId`, `text` veya `newText`). Yalnızca Gateway'in kendisinin gönderdiği mesajlar düzenlenebilir.
    - **gönderimi geri alma**: Desteklenen macOS/özel API sürümlerinde gönderilmiş bir mesajı geri çekin (`messageId`). Yalnızca Gateway'in kendisinin gönderdiği mesajların gönderimi geri alınabilir.
    - **dosya yükleme**: Medya/dosya gönderin (base64 olarak `buffer` veya verileri yüklenmiş bir `media`/`path`/`filePath`, `filename`, isteğe bağlı `asVoice`). Eski diğer ad: `sendAttachment`.
    - **grubu yeniden adlandırma**, **grup simgesini ayarlama**, **katılımcı ekleme**, **katılımcı kaldırma**, **gruptan ayrılma**: Geçerli hedef bir grup konuşması olduğunda grup sohbetlerini yönetin. Bunlar ana makinenin Messages kimliğini değiştirir; bu nedenle sahip gönderen veya bir `operator.admin` Gateway istemcisi gerektirir.
    - **anket**: Yerel bir Apple Messages anketi oluşturun (`pollQuestion`, 2 ila 12 kez yinelenen `pollOption` ve ayrıca `chatGuid`, `chatId`, `chatIdentifier` veya `to`). iOS/iPadOS/macOS 26+ kullanan alıcılar anketi yerel olarak görüp oy verebilir; daha eski işletim sistemi sürümleri yedek olarak "Anket gönderildi" metnini alır. `selectors.pollPayloadMessage` gerektirir.
    - **ankette oy verme**: Mevcut bir ankette oy verin (`pollId` veya `messageId` ve ayrıca `pollOptionIndex`, `pollOptionId` veya `pollOptionText` seçeneklerinden tam olarak biri). `selectors.pollVoteMessage` ve `poll.vote` RPC yöntemini gerektirir.

    Kabul edilen gelen anketler; soru, numaralandırılmış seçenek etiketleri, oy sayıları ve `poll-vote` için gereken anket mesajı kimliğiyle birlikte ajan için görüntülenir.

  </Accordion>

  <Accordion title="Mesaj kimlikleri">
    Gelen iMessage bağlamı, kullanılabilir olduğunda hem kısa `MessageSid` değerlerini hem de tam mesaj GUID'lerini (`MessageSidFull`) içerir. Kısa kimliklerin kapsamı, SQLite destekli yakın tarihli yanıt önbelleğiyle sınırlıdır ve kullanılmadan önce geçerli sohbetle eşleşip eşleşmedikleri denetlenir. Kısa bir kimliğin süresi dolarsa kimliği sağlayan konuşmayı hedefleyerek ilgili `MessageSidFull` ile yeniden deneyin. Tam kimlikler konuşma veya hesap bağlamasını atlamaz; bu nedenle başka bir sohbetten alınan kimliği geçerli hedefteki bir kimlikle değiştirin. Uzaktan devredilen çağrılar, geçerli konuşmaya ilişkin kanıt bulunmadığında eski tam kimlikleri reddedebilir.

  </Accordion>

  <Accordion title="Yetenek algılama">
    OpenClaw, özel API eylemlerini yalnızca önbelleğe alınmış yoklama durumu köprünün kullanılamadığını belirttiğinde gizler. Durum bilinmiyorsa eylemler görünür kalır ve yoklamalar gönderim sırasında gecikmeli olarak çalıştırılır; böylece ilk eylem, ayrı bir manuel durum yenilemesi olmadan `imsg launch` sonrasında başarılı olabilir.

  </Accordion>

  <Accordion title="Okundu bilgileri ve yazma göstergesi">
    Özel API köprüsü çalışırken kabul edilen gelen sohbetler okundu olarak işaretlenir ve doğrudan sohbetlerde, sıra kabul edilir edilmez ajan bağlamı hazırlayıp yanıt üretirken bir yazma balonu gösterilir. Okundu olarak işaretlemeyi şu yapılandırmayla devre dışı bırakın:

    ```json5
    {
      channels: {
        imessage: {
          sendReadReceipts: false,
        },
      },
    }
    ```

    Yöntem başına yetenek listesi denetiminden önceki eski `imsg` derlemeleri, yazma/okundu özelliklerini sessizce devre dışı bırakır; OpenClaw, eksik okundu bilgisinin nedeninin anlaşılabilmesi için her yeniden başlatmada bir kez uyarı günlüğü oluşturur.

  </Accordion>

  <Accordion title="Gelen tapback'ler">
    OpenClaw, iMessage tapback'lerine abone olur ve kabul edilen tepkileri normal mesaj metni yerine sistem olayları olarak yönlendirir; böylece bir kullanıcı tapback'i sıradan bir yanıt döngüsünü tetiklemez.

    Bildirim modu `channels.imessage.reactionNotifications` tarafından denetlenir:

    - `"own"` (varsayılan): yalnızca kullanıcılar bot tarafından yazılmış mesajlara tepki verdiğinde bildirim gönderilir.
    - `"all"`: yetkili gönderenlerden gelen tüm tapback'ler için bildirim gönderilir.
    - `"off"`: gelen tapback'ler yok sayılır.

    Hesap başına geçersiz kılmalar `channels.imessage.accounts.<id>.reactionNotifications` kullanır.

  </Accordion>

  <Accordion title="Onay tepkileri (👍 / 👎)">
    `approvals.exec.enabled` veya `approvals.plugin.enabled` doğru olduğunda ve istek iMessage'a yönlendirildiğinde Gateway, bir onay istemini yerel olarak teslim eder ve istemi sonuçlandırmak için bir tapback'i kabul eder:

    - `👍` (Beğen tapback'i) → `allow-once`
    - `👎` (Beğenme tapback'i) → `deny`
    - `allow-always` manuel bir yedek olarak kalır: `/approve <id> allow-always` öğesini normal yanıt olarak gönderin.

    Tepki işleme, tepki veren kullanıcının tanıtıcısının açıkça bir onaylayıcı olmasını gerektirir. Onaylayıcı listesi `channels.imessage.allowFrom` (veya `channels.imessage.accounts.<id>.allowFrom`) kaynağından okunur; kullanıcının telefon numarasını E.164 biçiminde veya Apple ID e-posta adresini ekleyin (`chat_id:*` gibi sohbet hedefleri geçerli onaylayıcı girdileri değildir). `"*"` joker girdisi dikkate alınır ancak herhangi bir gönderenin onay vermesine izin verir; boş bir onaylayıcı listesi tepki kısayolunu tamamen devre dışı bırakır. Tepki kısayolu, onayın sonuçlandırılmasında önemli olan tek denetim açık onaylayıcı izin listesi olduğundan `reactionNotifications`, `dmPolicy` ve `groupAllowFrom` öğelerini kasıtlı olarak atlar.

    `/approve` metin komutu yetkilendirmesi aynı listeyi izler: `channels.imessage.allowFrom` boş olmadığında `/approve <id> <decision>`, daha geniş DM izin listesine göre değil, bu onaylayıcı listesine göre yetkilendirilir ve DM izin listesinde izin verilen ancak `allowFrom` içinde bulunmayan gönderenler açıkça reddedilir. `allowFrom` boş olduğunda aynı sohbet yedeği geçerliliğini korur ve `/approve`, DM izin listesinin izin verdiği herkesi yetkilendirir. Onay vermesi gereken her operatörü — `/approve` veya tepkiler yoluyla — `allowFrom` listesine ekleyin.

    Operatör notları:
    - Tepki bağlaması hem bellekte hem de Gateway'in kalıcı anahtarlı deposunda saklanır (TTL, onayın sona erme süresiyle eşleştirilir); ayrıca Gateway, bekleyen istemleri tapback'ler için yoklar. Böylece Gateway yeniden başlatıldıktan kısa süre sonra gelen bir tapback yine de onayı sonuçlandırır.
    - Operatörün kendi `is_from_me=true` tapback'i (örneğin eşleştirilmiş bir Apple cihazından) bu tanıtıcı açıkça onaylayıcı olarak tanımlandığında onayı sonuçlandırır.
    - Onay istemleri yalnızca açık onaylayıcılar yapılandırıldığında bir grup konuşmasına yönlendirilir; aksi takdirde herhangi bir grup üyesi onay verebilir.
    - Eski metin tarzı tapback'ler (çok eski Apple istemcilerinden gelen `Liked "…"` düz metin) ileti GUID'si taşımadıkları için onayları sonuçlandıramaz; tepkinin sonuçlandırılması, güncel macOS / iOS istemcilerinin yaydığı yapılandırılmış tapback meta verilerini gerektirir.

  </Accordion>

  <Accordion title="Soru tepkileri (1️⃣ / 2️⃣ / 3️⃣ / 4️⃣)">
    Gizli olmayan, tek seçimli bir soru ve bir ila dört seçenek içeren bir `ask_user` istemi için OpenClaw numaralı emoji seçenekleri ekler. Yanıtlamak için teslim edilen isteme eşleşen numarayla tepki verin. Tepki, bot tarafından oluşturulan iletinin kararlı GUID'sini taşımalıdır; ardından OpenClaw, numarayı Gateway üzerinden standart seçeneğe eşler. Eski veya yinelenen dokunuşlar yok sayılır.

    Çok sorulu, çok seçimli ve serbest metin istemleri yalnızca metinle yanıtlanabilir. Soru tepkileri normal iMessage DM/grup kabul kurallarına uyar. Genel `reactionNotifications`, `"off"` olduğunda bile ilgisiz tepkileri aracı olaylarına dönüştürmeden tanınırlar.

  </Accordion>
</AccordionGroup>

## Yapılandırma yazmaları

iMessage, kanal tarafından başlatılan yapılandırma yazmalarına varsayılan olarak izin verir (`commands.config: true` olduğunda `/config set|unset` için).

Devre dışı bırakmak için:

```json5
{
  channels: {
    imessage: {
      configWrites: false,
    },
  },
}
```

<a id="coalescing-split-send-dms-command--url-in-one-composition"></a>

## Bölünmüş gönderimli DM'leri birleştirme (tek bir oluşturmada komut + URL)

Apple, bir komutu ve URL önizlemesini ayrı fiziksel `chat.db` satırları olarak saklayabilir. `imsg` 0.13.1 ve daha yeni sürümler, izleme, geçmiş veya arama iletiyi döndürmeden önce bu satırları birleştirir; böylece OpenClaw, kanala özgü DM gecikmesi eklemeden tek bir mantıksal gelen ileti alır.

Herhangi bir iMessage birleştirme ayarı gerekmez. Kullanımdan kaldırılan `channels.imessage.coalesceSameSenderDms` anahtarı, `openclaw doctor --fix` tarafından kaldırılır. Bir kanaldaki hızlı metin iletilerini bilinçli olarak toplu işlemek istediğinizde genel `messages.inbound` gecikmeli birleştirmesi kullanılabilir.

Komut ve URL içeren gönderimler ayrı aracı dönüşleri olarak geliyorsa Messages Mac'te `imsg` öğesini güncelleyin:

```bash
brew update && brew upgrade imsg
```

## Köprü veya Gateway yeniden başlatıldıktan sonra gelen iletileri kurtarma

iMessage, Gateway kapalıyken kaçırılan iletileri kurtarır ve aynı zamanda Apple'ın Push kurtarmasından sonra boşaltabileceği eski "birikmiş ileti bombası"nı bastırır. Varsayılan davranış her zaman etkindir ve kalıcı giriş ile yaş sınırına dayanır.

- **Kalıcı yeniden oynatma koruması.** OpenClaw, kurtarma imlecini ilerletmeden önce her ham satırı, olay kimliği olarak Apple GUID'siyle paylaşılan SQLite giriş kuyruğuna kaydeder. Tamamlanan bir satır yaklaşık 4 saat boyunca ve en fazla 10.000 girdi sınırıyla bir mezar taşı bırakır; böylece aynı GUID'ye sahip yeniden oynatma, yeniden başlatmadan sonra bile bırakılır. Bekleyen bir satır, dağıtım onu devralana kadar kurtarılabilir durumda kalır.
- **Kesinti süresinden kurtarma.** Başlangıçta izleyici, kalıcı olarak kabul edilen son `chat.db` rowid değerini (hesap başına kalıcı bir imleç) hatırlar ve bunu `imsg watch.subscribe` öğesine `since_rowid` olarak geçirir; böylece imsg, henüz günlüğe kaydedilmemiş satırları yeniden oynatır ve ardından canlı akışı izler. Çökmeden önce günlüğe kaydedilen satırlar SQLite'tan devam eder. Yeniden oynatma, en son 500 satırla ve en fazla ~2 saatlik iletilerle sınırlıdır; GUID mezar taşları ise daha önce işlenmiş her şeyi bırakır.
- **Eski birikmiş iletiler için yaş sınırı.** Başlangıç sınırının üzerindeki satırlar gerçekten canlıdır; gönderim tarihi varışından ~15 dakikadan daha eski olanlar Push boşaltma birikimidir ve bastırılır. Yeniden oynatılan satırlar (sınırda veya sınırın altında) bunun yerine daha geniş kurtarma penceresini kullanır; böylece yakın zamanda kaçırılan bir ileti teslim edilirken çok eski geçmiş teslim edilmez.

Kurtarma hem yerel hem de uzak `cliPath` kurulumlarında çalışır; çünkü `since_rowid` yeniden oynatması aynı `imsg` RPC bağlantısı üzerinden çalışır. Fark, penceredir: Gateway `chat.db` öğesini okuyabildiğinde (yerel), başlangıç rowid sınırını sabitler, yeniden oynatma aralığını sınırlar ve birkaç saat öncesine kadar kaçırılan iletileri teslim eder. Uzak bir SSH `cliPath` üzerinden veritabanını okuyamaz; bu nedenle yeniden oynatma sınırsızdır ve her satır canlı yaş sınırını kullanır — yakın zamanda kaçırılan iletileri yine kurtarır ve eski birikimi yine bastırır, ancak daha dar canlı pencereyle. Daha geniş kurtarma penceresi için Gateway'i Messages Mac'te çalıştırın.

### Operatör tarafından görülebilen sinyal

Bastırılan birikim varsayılan düzeyde günlüğe kaydedilir, hiçbir zaman sessizce bırakılmaz (`recovery` bayrağı hangi pencerenin uygulandığını gösterir):

```text
imessage: suppressed stale inbound backlog account=<id> sent=<iso> recovery=<bool> (<N> başlangıçtan bu yana bastırıldı)
```

### Geçiş

`channels.imessage.catchup.*` kullanımdan kaldırılmıştır — kesinti süresinden kurtarma otomatiktir ve yeni kurulumlarda yapılandırma gerektirmez. `catchup.enabled: true` içeren mevcut yapılandırmalar, kurtarma yeniden oynatma penceresi için bir uyumluluk profili olarak desteklenmeye devam eder. Devre dışı bırakılmış yakalama blokları (`enabled: false` veya `enabled: true` olmaması) kullanımdan kaldırılmıştır; `openclaw doctor --fix` bunları kaldırır.

## Sorun giderme

<AccordionGroup>
  <Accordion title="imsg bulunamadı veya RPC desteklenmiyor">
    İkili dosyayı ve RPC desteğini doğrulayın:

    ```bash
    imsg rpc --help
    imsg status --json
    openclaw channels status --probe
    ```

    Yoklama RPC'nin desteklenmediğini bildirirse `imsg` öğesini güncelleyin. Özel API eylemleri kullanılamıyorsa oturum açmış macOS kullanıcı oturumunda `imsg launch` öğesini çalıştırın ve yeniden yoklayın. Gateway macOS'te çalışmıyorsa varsayılan yerel `imsg` yolu yerine yukarıdaki SSH üzerinden Uzak Mac kurulumunu kullanın.

  </Accordion>

  <Accordion title="İletiler gönderiliyor ancak gelen iMessage'lar ulaşmıyor">
    Önce iletinin yerel Mac'e ulaşıp ulaşmadığını doğrulayın. `chat.db` değişmiyorsa `imsg status --json` sağlıklı bir köprü bildirse bile OpenClaw iletiyi alamaz.

```bash
imsg chats --limit 10 --json
imsg watch --chat-id <chat-id> --json
sqlite3 ~/Library/Messages/chat.db \
  "select datetime(max(date)/1000000000 + 978307200, 'unixepoch', 'localtime'), max(ROWID) from message;"
```

    Telefondan gönderilen iletiler yeni satırlar oluşturmuyorsa OpenClaw yapılandırmasını değiştirmeden önce macOS Messages ve Apple Push katmanını onarın. Tek seferlik bir hizmet yenilemesi çoğu zaman yeterlidir:

```bash
launchctl kickstart -k system/com.apple.apsd
launchctl kickstart -k gui/$(id -u)/com.apple.CommCenter
launchctl kickstart -k gui/$(id -u)/com.apple.identityservicesd
launchctl kickstart -k gui/$(id -u)/com.apple.imagent
imsg launch
openclaw gateway restart
```

    Telefondan yeni bir iMessage gönderin ve OpenClaw oturumlarında hata ayıklamadan önce yeni bir `chat.db` satırını veya `imsg watch` olayını doğrulayın. Bunu düzenli aralıklarla çalışan bir köprü yeniden başlatma döngüsü olarak kullanmayın; etkin çalışma sırasında yinelenen `imsg launch` işlemleri ve Gateway yeniden başlatmaları teslimatları kesintiye uğratabilir ve devam eden kanal çalıştırmalarını yarıda bırakabilir.

  </Accordion>

  <Accordion title="Gateway macOS'te çalışmıyor">
    Varsayılan `cliPath: "imsg"`, Messages oturumu açık olan Mac'te çalışmalıdır. Linux veya Windows'ta `channels.imessage.cliPath` öğesini, bu Mac'e SSH ile bağlanıp `imsg "$@"` çalıştıran bir sarmalayıcı betiğe ayarlayın.

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    Ardından şunu çalıştırın:

```bash
openclaw channels status --probe --channel imessage
```

  </Accordion>

  <Accordion title="DM'ler yok sayılıyor">
    Şunları kontrol edin:

    - `channels.imessage.dmPolicy`
    - `channels.imessage.allowFrom`
    - eşleştirme onayları (`openclaw pairing list imessage`)

  </Accordion>

  <Accordion title="Grup iletileri yok sayılıyor">
    Şunları kontrol edin:

    - `channels.imessage.groupPolicy`
    - `channels.imessage.groupAllowFrom`
    - `channels.imessage.groups` izin verilenler listesi davranışı
    - bahsetme kalıbı yapılandırması (`agents.entries.*.groupChat.mentionPatterns`)

  </Accordion>

  <Accordion title="Uzak ekler başarısız oluyor">
    Şunları kontrol edin:

    - `channels.imessage.remoteHost`
    - `channels.imessage.remoteAttachmentRoots`
    - Gateway ana makinesinden SSH/SCP anahtar kimlik doğrulaması
    - ana makine anahtarının Gateway ana makinesindeki `~/.ssh/known_hosts` içinde bulunması
    - Messages çalıştıran Mac'teki uzak yolun okunabilirliği

  </Accordion>

  <Accordion title="macOS izin istemleri kaçırıldı">
    Aynı kullanıcı/oturum bağlamındaki etkileşimli bir GUI terminalinde yeniden çalıştırın ve istemleri onaylayın:

    ```bash
    imsg chats --limit 1
    imsg send <handle> "test"
    ```

    OpenClaw/`imsg` çalıştıran işlem bağlamına Tam Disk Erişimi + Otomasyon izinlerinin verildiğini doğrulayın.

  </Accordion>
</AccordionGroup>

## Yapılandırma referansı bağlantıları

- [Yapılandırma referansı - iMessage](/tr/gateway/config-channels#imessage)
- [Gateway yapılandırması](/tr/gateway/configuration)
- [Eşleştirme](/tr/channels/pairing)

## İlgili

- [Kanallara Genel Bakış](/tr/channels) — desteklenen tüm kanallar
- [BlueBubbles'ın kaldırılması ve imsg iMessage yolu](/tr/announcements/bluebubbles-imessage) — duyuru ve geçiş özeti
- [BlueBubbles'tan geçiş](/tr/channels/imessage-from-bluebubbles) — yapılandırma çeviri tablosu ve adım adım geçiş
- [Eşleştirme](/tr/channels/pairing) — DM kimlik doğrulaması ve eşleştirme akışı
- [Gruplar](/tr/channels/groups) — grup sohbeti davranışı ve bahsetme denetimi
- [Kanal Yönlendirme](/tr/channels/channel-routing) — iletiler için oturum yönlendirmesi
- [Güvenlik](/tr/gateway/security) — erişim modeli ve sağlamlaştırma
