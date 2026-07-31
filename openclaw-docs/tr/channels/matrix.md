---
read_when:
    - OpenClaw'da Matrix kurulumu
    - Matrix E2EE ve doğrulamayı yapılandırma
summary: Matrix destek durumu, kurulumu ve yapılandırma örnekleri
title: Matrix
x-i18n:
    generated_at: "2026-07-26T23:29:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa84c7d9d9019040a3fec3cfaabb78590006a4a2dd4bb95836f2cf37072777c5
    source_path: channels/matrix.md
    workflow: 16
---

Matrix, resmi `matrix-js-sdk` üzerine kurulu, indirilebilir bir kanal pluginidir (`@openclaw/matrix`). DM'leri, odaları, ileti dizilerini, medyayı, tepkileri, anketleri, konumu ve E2EE'yi destekler.

## Kurulum

```bash
openclaw plugins install @openclaw/matrix
```

Yalın plugin belirtimleri önce ClawHub'ı, ardından npm geri dönüşünü dener. `openclaw plugins install clawhub:@openclaw/matrix` veya `npm:@openclaw/matrix` ile bir kaynağı zorunlu kılın. Yerel bir çalışma kopyasından: `openclaw plugins install ./path/to/local/matrix-plugin`.

`plugins install`, plugini kaydeder ve etkinleştirir; ayrı bir `enable` adımı gerekmez. Kanal, aşağıda yapılandırılana kadar yine de hiçbir şey yapmaz. Genel kurulum kuralları için [Pluginler](/tr/tools/plugin) bölümüne bakın.

## Kurulum

1. Ana sunucunuzda bir Matrix hesabı oluşturun.
2. `channels.matrix` öğesini `homeserver` + `accessToken` veya `homeserver` + `userId` + `password` ile yapılandırın.
3. Gateway'i yeniden başlatın.
4. Botla bir DM başlatın veya botu bir odaya davet edin. Yeni davetler yalnızca [`autoJoin`](#auto-join) izin verdiğinde ulaşır.

### Etkileşimli kurulum

```bash
openclaw channels add
openclaw configure --section channels
```

Sihirbaz; ana sunucu URL'sini, kimlik doğrulama yöntemini (belirteç veya parola), kullanıcı kimliğini (yalnızca parola kimlik doğrulaması), isteğe bağlı cihaz adını, E2EE'nin etkinleştirilip etkinleştirilmeyeceğini ve oda erişimi/otomatik katılım ayarlarını sorar. Eşleşen `MATRIX_*` ortam değişkenleri zaten mevcutsa ve hesapta kaydedilmiş kimlik doğrulama bilgisi yoksa sihirbaz bir ortam değişkeni kısayolu sunar. `openclaw channels resolve --channel matrix "Project Room"` ile bir izin verilenler listesini kaydetmeden önce oda adlarını çözümleyin. Sihirbazda E2EE'yi etkinleştirmek, [`openclaw matrix encryption setup`](#encryption-and-verification) ile aynı önyüklemeyi çalıştırır.

### Asgari yapılandırma

Belirteç tabanlı:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

Parola tabanlı (belirteç, ilk oturum açmanın ardından önbelleğe alınır):

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: izin verilenler listesi gizli bilgisi
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

### Otomatik katılım

`channels.matrix.autoJoin` varsayılan olarak `"off"` değerindedir: siz elle katılana kadar bot, yeni davetlerden gelen yeni odalarda veya DM'lerde görünmez. OpenClaw, davet anında bir davetin DM mi yoksa grup mu olduğunu belirleyemez; bu nedenle her davet önce `autoJoin` üzerinden geçer. `dm.policy` yalnızca daha sonra, bot katıldıktan ve oda sınıflandırıldıktan sonra uygulanır.

<Warning>
Kabul edilen davetleri kısıtlamak için `autoJoin: "allowlist"` ile birlikte `autoJoinAllowlist` değerini veya her daveti kabul etmek için `autoJoin: "always"` değerini ayarlayın.

`autoJoinAllowlist` yalnızca `!roomId:server`, `#alias:server` veya `*` kabul eder. Düz oda adları reddedilir; diğer adlar, davet edilen odanın iddia ettiği duruma göre değil, ana sunucuya göre çözümlenir.
</Warning>

```json5
{
  channels: {
    matrix: {
      autoJoin: "allowlist",
      autoJoinAllowlist: ["!ops:example.org", "#support:example.org"],
      groups: {
        "!ops:example.org": { requireMention: true },
      },
    },
  },
}
```

### İzin verilenler listesi hedef biçimleri

- DM'ler (`dm.allowFrom`, `groupAllowFrom`, `groups.<room>.users`): `@user:server` kullanın. Görünen adlar varsayılan olarak yok sayılır (değiştirilebilir); yalnızca açık görünen ad uyumluluğu için `dangerouslyAllowNameMatching: true` ayarlayın.
- Oda izin verilenler listesi anahtarları (`groups`, eski diğer ad `rooms`): `!room:server` veya `#alias:server` kullanın. `dangerouslyAllowNameMatching: true` olmadığı sürece düz adlar yok sayılır.
- Davet izin verilenler listeleri (`autoJoinAllowlist`): `!room:server`, `#alias:server` veya `*` kullanın. Düz adlar her zaman reddedilir.

### Hesap kimliği normalleştirmesi

Sihirbaz, kolay anlaşılır bir adı normalleştirilmiş hesap kimliğine dönüştürür (`Ops Bot` -> `ops-bot`). Hesapların çakışmaması için kapsamlı ortam değişkeni adlarındaki noktalama işaretleri onaltılık biçimde kaçışlanır: `-` (0x2D), `_X2D_` olur; böylece `ops-prod`, `MATRIX_OPS_X2D_PROD_` ortam önekiyle eşleşir.

### Önbelleğe alınmış kimlik bilgileri

Matrix, hesap kimlik bilgilerini paylaşılan `state/openclaw.sqlite` plugin durumunda önbelleğe alır. Önbelleğe alınmış kimlik bilgileri mevcut olduğunda OpenClaw, yapılandırma dosyasında `accessToken` olmasa bile Matrix'i yapılandırılmış kabul eder; bu durum kurulumu, `openclaw doctor` ve kanal durumu yoklamalarını kapsar. Yükseltmeler, kullanımdan kaldırılmış `~/.openclaw/credentials/matrix/credentials*.json` dosyalarını `openclaw doctor --fix` aracılığıyla içe aktarır, SQLite satırlarını doğrular ve ardından dosyaları arşivler.

### Ortam değişkenleri

Eşdeğer yapılandırma anahtarı ayarlanmamışken kullanılan, yapılandırma anahtarı destekli ortam değişkenleri. Varsayılan hesap öneksiz adları kullanır; adlandırılmış hesaplar, hesap belirtecini son ekten önce ekler ([normalleştirme](#account-id-normalization) bölümüne bakın).

| Varsayılan hesap       | Adlandırılmış hesap (`<ID>` = hesap belirteci) |
| --------------------- | -------------------------------------- |
| `MATRIX_HOMESERVER`   | `MATRIX_<ID>_HOMESERVER`               |
| `MATRIX_ACCESS_TOKEN` | `MATRIX_<ID>_ACCESS_TOKEN`             |
| `MATRIX_USER_ID`      | `MATRIX_<ID>_USER_ID`                  |
| `MATRIX_PASSWORD`     | `MATRIX_<ID>_PASSWORD`                 |
| `MATRIX_DEVICE_ID`    | `MATRIX_<ID>_DEVICE_ID`                |
| `MATRIX_DEVICE_NAME`  | `MATRIX_<ID>_DEVICE_NAME`              |

`ops` hesabı için adlar `MATRIX_OPS_HOMESERVER`, `MATRIX_OPS_ACCESS_TOKEN` vb. olur. `MATRIX_HOMESERVER` (ve `*_HOMESERVER` kapsamlı herhangi bir değişken), çalışma alanındaki bir `.env` üzerinden ayarlanamaz; [Çalışma alanı `.env` dosyaları](/tr/gateway/security) bölümüne bakın.

<Note>
Kurtarma anahtarı, yapılandırma destekli bir ortam değişkeni değildir: OpenClaw bunu hiçbir zaman doğrudan ortamdan okumaz. CLI yönlendirme metni, varsayılan hesap için `MATRIX_RECOVERY_KEY` adlı bir kabuk değişkeni veya adlandırılmış hesap için `MATRIX_RECOVERY_KEY_<ID>` (düz büyük harfli hesap kimliği, onaltılık kaçış yok) üzerinden aktarılmasını önerir; [Bu cihazı kurtarma anahtarıyla doğrulama](#verify-this-device-with-a-recovery-key) bölümüne bakın.
</Note>

## Yapılandırma örneği

DM eşleştirme, oda izin verilenler listesi ve E2EE içeren pratik bir temel yapılandırma:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": { requireMention: true },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: { mode: "partial" },
    },
  },
}
```

## Akış önizlemeleri

Matrix yanıt akışı isteğe bağlıdır. `streaming.mode`, OpenClaw'ın devam eden asistan yanıtını nasıl ilettiğini; `streaming.block.enabled` ise tamamlanan her bloğun ayrı bir Matrix mesajı olarak tutulup tutulmayacağını denetler.

```json5
{
  channels: {
    matrix: {
      streaming: { mode: "partial" },
    },
  },
}
```

Canlı yanıt önizlemelerini koruyup ara araç/ilerleme satırlarını gizlemek için:

```json5
{
  channels: {
    matrix: {
      streaming: {
        mode: "partial",
        preview: {
          toolProgress: false,
        },
      },
    },
  },
}
```

Tam yapılandırma `{ mode, chunkMode, block, preview, progress }` kabul eder:

```json5
{
  channels: {
    matrix: {
      streaming: {
        mode: "progress",
        progress: {
          label: "auto", // yapılandırılmış veya yerleşik etiketlerden seç (gizlemek için false)
          labels: ["Thinking", "Writing", "Searching"], // label: "auto" için adaylar
          maxLines: 8, // azami kayan ilerleme satırı sayısı (varsayılan: 8)
          maxLineChars: 120, // kesmeden önce satır başına azami karakter sayısı (varsayılan: 120)
          toolProgress: true, // araç/ilerleme etkinliğini göster (varsayılan: true)
        },
      },
    },
  },
}
```

- `progress.label`: özel etiket; yapılandırılmış veya yerleşik bir etiket seçmek için `"auto"`/ayarlanmamış ya da gizlemek için `false`.
- `progress.labels`: yalnızca `label`, `"auto"` olduğunda veya ayarlanmadığında kullanılan adaylar.
- `progress.maxLines`: taslakta tutulan azami kayan ilerleme satırı sayısı; bu sınır aşıldığında eski satırlar kırpılır.
- `progress.maxLineChars`: kesmeden önce kompakt ilerleme satırı başına azami karakter sayısı.
- `progress.toolProgress`: `true` olduğunda (varsayılan), canlı araç/ilerleme etkinliği taslakta görünür.

| `streaming.mode`  | Davranış                                                                                                                                                 |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `"off"` (varsayılan) | Tam yanıtı bekler ve bir kez gönderir.                                                                                                                      |
| `"partial"`       | Model geçerli bloğu yazarken normal bir metin mesajını yerinde düzenler. Standart istemciler son düzenlemede değil, ilk önizlemede bildirim gönderebilir.          |
| `"quiet"`         | `"partial"` ile aynıdır ancak mesaj, bildirim göndermeyen bir duyurudur. Alıcılara, kullanıcı başına bir anlık bildirim kuralı tamamlanmış düzenlemeyle eşleştiğinde bir kez bildirim gönderilir (aşağıya bakın). |
| `"progress"`      | Bir ilerleme taslağı kullanarak ayrı kompakt ilerleme satırları gönderir.                                                                                          |

`streaming.block.enabled` (varsayılan `false`), `streaming.mode` değerinden bağımsızdır:

| `streaming.mode`        | `block.enabled: true`                                               | `block.enabled: false` (varsayılan)                     |
| ----------------------- | ------------------------------------------------------------------- | ---------------------------------------------------- |
| `"partial"` / `"quiet"` | Geçerli blok için canlı taslak, tamamlanan bloklar mesaj olarak tutulur | Geçerli blok için canlı taslak, yerinde sonlandırılır |
| `"off"`                 | Tamamlanan her blok için bildirim gönderen bir Matrix mesajı                     | Tam yanıt için bildirim gönderen tek bir Matrix mesajı      |

Notlar:

- Bir önizleme Matrix'in olay başına boyut sınırını aşarsa OpenClaw önizleme akışını durdurur ve yalnızca son iletime geri döner.
- Medya yanıtları ekleri her zaman normal biçimde gönderir; eski bir önizleme güvenle yeniden kullanılamıyorsa OpenClaw son medya yanıtını göndermeden önce onu sansürler.
- Önizleme akışı etkinken araç ilerleme önizlemesi güncellemeleri varsayılan olarak açıktır. Yanıt metni için önizleme düzenlemelerini koruyup araç ilerlemesini normal iletim yolunda bırakmak üzere `streaming.preview.toolProgress: false` ayarlayın.
- Önizleme düzenlemeleri ek Matrix API çağrılarına mal olur. En ölçülü hız sınırı profili için `streaming.mode: "off"` değerini koruyun.
- Eski skaler/boole `streaming` değerleri ile düz `blockStreaming` / `chunkMode` anahtarları, `openclaw doctor --fix` tarafından bu iç içe şekle yeniden yazılır.

## Sesli mesajlar

Gelen Matrix sesli notları oda bahsetme kapısından önce yazıya dökülür; böylece botun adını söyleyen bir sesli not, `requireMention: true` odasında agent'ı tetikleyebilir ve agent yalnızca bir ses eki yer tutucusu yerine dökümü alır.

Matrix, `tools.media.audio` altındaki paylaşılan ses medyası sağlayıcısını kullanır; örneğin OpenAI `gpt-4o-mini-transcribe`. Sağlayıcı kurulumu ve sınırlar için [Medya araçlarına genel bakış](/tr/tools/media-overview) bölümüne bakın.

- `m.audio` olayları ve `audio/*` MIME türüne sahip `m.file` olayları uygundur.
- Şifreli odalarda OpenClaw, transkripsiyondan önce eki mevcut Matrix medya yolu üzerinden çözer.
- Transkript, aracı isteminde makine tarafından oluşturulmuş ve güvenilmeyen olarak işaretlenir.
- Ek, aşağı akıştaki medya araçlarının onu yeniden yazıya dökmemesi için zaten yazıya dökülmüş olarak işaretlenir.
- Ses transkripsiyonunu genel olarak devre dışı bırakmak için `tools.media.audio.enabled: false` değerini ayarlayın.

## Onay meta verileri

Matrix yerel onay istemleri, `com.openclaw.approval` anahtarı altında OpenClaw'a özgü içerik barındıran normal `m.room.message` olaylarıdır. Standart istemciler metin gövdesini yine görüntüler; OpenClaw uyumlu istemciler yapılandırılmış onay kimliğini, türünü, durumunu, kararlarını ve exec/plugin ayrıntılarını okuyabilir.

Bir istem tek bir Matrix olayı için çok uzun olduğunda OpenClaw, görünür metni parçalara böler ve `com.openclaw.approval` öğesini yalnızca ilk parçaya ekler. İzin verme/reddetme tepkileri bu ilk olaya bağlanır; böylece uzun istemler, tek olaylı istemlerle aynı onay hedefini korur.

### Sessiz sonlandırılmış önizlemeler için kendi sunucunuzdaki anlık bildirim kuralları

`streaming.mode: "quiet"`, alıcıları yalnızca bir blok veya dönüş sonlandırıldığında bilgilendirir; kullanıcı başına bir anlık bildirim kuralının sonlandırılmış önizleme işaretiyle eşleşmesi gerekir. Tarifin tamamı için [sessiz önizlemelere yönelik Matrix anlık bildirim kuralları](/tr/channels/matrix-push-rules) bölümüne bakın.

## Botlar arası odalar

Varsayılan olarak, yapılandırılmış diğer OpenClaw Matrix hesaplarından gelen Matrix mesajları yok sayılır. Aracılar arası trafiğe bilinçli olarak izin vermek için `allowBots` kullanın:

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- `allowBots: true`, izin verilen odalarda ve DM'lerde yapılandırılmış diğer Matrix bot hesaplarından gelen mesajları kabul eder.
- `allowBots: "mentions"`, bu mesajları odalarda yalnızca bu bottan görünür şekilde bahsettiklerinde kabul eder; DM'lere bundan bağımsız olarak izin verilir.
- `groups.<room>.allowBots`, tek bir oda için hesap düzeyindeki ayarı geçersiz kılar.
- Kabul edilen yapılandırılmış bot mesajları, paylaşılan [bot döngüsü korumasını](/tr/channels/bot-loop-protection) kullanır. `channels.defaults.botLoopProtection` öğesini yapılandırın, ardından hesap başına `channels.matrix.botLoopProtection` veya oda başına `channels.matrix.groups.<room>.botLoopProtection` ile geçersiz kılın.
- OpenClaw, kendi kendine yanıt döngülerini önlemek için aynı Matrix kullanıcı kimliğinden gelen mesajları yine yok sayar.
- Matrix'in yerel bir bot bayrağı yoktur; OpenClaw, "bot tarafından yazılmış" ifadesini "bu OpenClaw Gateway'inde yapılandırılmış başka bir Matrix hesabı tarafından gönderilmiş" olarak kabul eder.

Paylaşılan odalarda botlar arası trafiği etkinleştirirken katı oda izin listeleri ve bahsetme gereksinimleri kullanın.

## Şifreleme ve doğrulama

Şifreli (E2EE) odalarda giden görsel olayları, görsel önizlemelerinin tam ekle birlikte şifrelenmesi için `thumbnail_file` kullanır; şifrelenmemiş odalar düz `thumbnail_url` kullanır. Yapılandırma gerekmez; plugin, E2EE durumunu otomatik olarak algılar.

Tüm `openclaw matrix` komutları `--verbose` (tam tanılama), `--json` (makine tarafından okunabilir çıktı) ve `--account <id>` (çok hesaplı kurulumlar) seçeneklerini kabul eder. Çıktı varsayılan olarak özlüdür.

### Şifrelemeyi etkinleştirme

```bash
openclaw matrix encryption setup
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix encryption setup --recovery-key-stdin
```

Gizli veri depolamayı ve çapraz imzalamayı başlatır, gerekirse oda anahtarı yedeği oluşturur, ardından durumu ve sonraki adımları yazdırır. Yararlı bayraklar:

- `--recovery-key-stdin`, kurtarma anahtarını işlem bağımsız değişkenlerinde açığa çıkarmadan stdin'den okur; `--recovery-key <key>` uyumluluk için kullanılabilir durumda kalır
- `--force-reset-cross-signing`, mevcut çapraz imzalama kimliğini atar ve yeni bir kimlik oluşturur (yalnızca bilinçli kullanım için)

Yeni bir hesapta E2EE'yi hesap oluşturulurken etkinleştirin:

```bash
openclaw matrix account add \
  --homeserver https://matrix.example.org \
  --access-token syt_xxx \
  --enable-e2ee
```

`--encryption`, `--enable-e2ee` için bir diğer addır. Eşdeğer manuel yapılandırma:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

### Durum ve güven sinyalleri

```bash
openclaw matrix verify status
openclaw matrix verify status --include-recovery-key --json
```

`verify status`, üç bağımsız güven sinyali bildirir (`--verbose` bunların tümünü gösterir):

- `Locally trusted`: yalnızca bu istemci tarafından güveniliyor
- `Cross-signing verified`: SDK, çapraz imzalama üzerinden doğrulama bildiriyor
- `Signed by owner`: kendi öz imzalama anahtarınızla imzalanmış (yalnızca tanılama)

`Verified by owner`, yalnızca `Cross-signing verified` değeri `yes` olduğunda `yes` olur; yalnızca yerel güven veya sahip imzası yeterli değildir.

`--allow-degraded-local-state`, önce Matrix hesabını hazırlamadan elden geldiğince tanılama döndürür; çevrimdışı veya kısmen yapılandırılmış incelemeler için kullanışlıdır.

### Bu cihazı kurtarma anahtarıyla doğrulama

Kurtarma anahtarını komut satırında geçirmek yerine stdin üzerinden yönlendirin:

```bash
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin
```

Komut üç durum bildirir:

- `Recovery key accepted`: Matrix, anahtarı gizli veri depolama veya cihaz güveni için kabul etti.
- `Backup usable`: oda anahtarı yedeği, güvenilir kurtarma materyaliyle yüklenebilir.
- `Device verified by owner`: bu cihaz, tam Matrix çapraz imzalama kimliği güvenine sahip.

Kurtarma anahtarı yedek materyalinin kilidini açmış olsa bile tam kimlik güveni eksik olduğunda sıfırdan farklı bir kodla çıkar. Bu durumda öz doğrulamayı başka bir Matrix istemcisinden tamamlayın:

```bash
openclaw matrix verify self
```

`verify self`, başarıyla çıkmadan önce `Cross-signing verified: yes` için bekler. Bekleme süresini ayarlamak için `--timeout-ms <ms>` kullanın.

Anahtarın doğrudan belirtildiği `openclaw matrix verify device "<recovery-key>"` biçimi de çalışır ancak anahtar kabuk geçmişine kaydedilir.

### Çapraz imzalamayı başlatma veya onarma

```bash
openclaw matrix verify bootstrap
```

Şifreli hesaplara yönelik onarım/kurulum komutudur. Sırasıyla şunları yapar:

- mümkün olduğunda mevcut kurtarma anahtarını yeniden kullanarak gizli veri depolamayı başlatır
- çapraz imzalamayı başlatır ve eksik açık anahtarları yükler
- mevcut cihazı işaretler ve çapraz imzalar
- henüz yoksa sunucu tarafında bir oda anahtarı yedeği oluşturur

Homeserver, çapraz imzalama anahtarlarını yüklemek için UIA gerektiriyorsa OpenClaw önce kimlik doğrulamasız yöntemi, ardından `m.login.dummy` ve sonra `m.login.password` yöntemini dener (`channels.matrix.password` gerektirir).

Yararlı bayraklar:

- `--recovery-key-stdin` (`printf '%s\n' "$MATRIX_RECOVERY_KEY" | ...` ile birlikte kullanın) veya `--recovery-key <key>`
- mevcut çapraz imzalama kimliğini atmak için `--force-reset-cross-signing` (yalnızca bilinçli kullanım için; etkin kurtarma anahtarının depolanmış veya `--recovery-key-stdin` ile sağlanmış olmasını gerektirir)

### Oda anahtarı yedeği

```bash
openclaw matrix verify backup status
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin
```

`backup status`, sunucu tarafında bir yedeğin bulunup bulunmadığını ve bu cihazın yedeğin şifresini çözüp çözemediğini gösterir. `backup restore`, yedeklenmiş oda anahtarlarını yerel kripto deposuna aktarır; kurtarma anahtarı zaten diskteyse `--recovery-key-stdin` öğesini kullanmayın.

Bozuk bir yedeği yeni bir temel durumla değiştirmek için (kurtarılamayan eski geçmişin kaybedilmesini kabul eder; mevcut yedek sırrı yüklenemiyorsa gizli veri depolamayı da yeniden oluşturabilir):

```bash
openclaw matrix verify backup reset --yes
```

Yalnızca önceki kurtarma anahtarının yeni yedek temel durumunun kilidini bilinçli olarak artık açmaması gerektiğinde `--rotate-recovery-key` ekleyin.

### Doğrulamaları listeleme, isteme ve yanıtlama

```bash
openclaw matrix verify list
```

Seçili hesap için bekleyen doğrulama isteklerini listeler.

```bash
openclaw matrix verify request --own-user
openclaw matrix verify request --user-id @ops:example.org --device-id ABCDEF
```

Bu hesaptan bir doğrulama isteği gönderir. `--own-user`, öz doğrulama ister (istemi aynı kullanıcının başka bir Matrix istemcisinde kabul edin); `--user-id`/`--device-id`/`--room-id` başka birini hedefler. `--own-user`, diğer hedefleme bayraklarıyla birlikte kullanılamaz.

Daha düşük düzeyli yaşam döngüsü işleme için — genellikle başka bir istemciden gelen istekleri izlerken — bu komutlar belirli bir `<id>` isteği üzerinde işlem yapar (`verify list` ve `verify request` tarafından yazdırılır):

| Komut                                      | Amaç                                                                |
| ------------------------------------------ | ------------------------------------------------------------------- |
| `openclaw matrix verify accept <id>`       | Gelen bir isteği kabul et                                           |
| `openclaw matrix verify start <id>`        | SAS akışını başlat                                                  |
| `openclaw matrix verify sas <id>`          | SAS emojisini veya ondalık sayılarını yazdır                        |
| `openclaw matrix verify confirm-sas <id>`  | SAS'ın diğer istemcinin gösterdiğiyle eşleştiğini onayla            |
| `openclaw matrix verify mismatch-sas <id>` | Emoji veya ondalık sayıları eşleşmediğinde SAS'ı reddet             |
| `openclaw matrix verify cancel <id>`       | İptal et; isteğe bağlı `--reason <text>` ve `--code <matrix-code>` alır |

`accept`, `start`, `sas`, `confirm-sas`, `mismatch-sas` ve `cancel`, doğrulama belirli bir doğrudan mesaj odasına bağlı olduğunda DM takibi ipuçları olarak `--user-id` ve `--room-id` değerlerini kabul eder.

### Çok hesaplı kullanıma ilişkin notlar

`--account <id>` olmadan Matrix CLI komutları örtük varsayılan hesabı kullanır. Birden fazla adlandırılmış hesap varken `channels.matrix.defaultAccount` sağlanmazsa komutlar tahminde bulunmayı reddeder ve seçim yapmanızı ister. E2EE, adlandırılmış bir hesap için devre dışıysa veya kullanılamıyorsa hatalar ilgili hesabın yapılandırma anahtarını gösterir; örneğin `channels.matrix.accounts.assistant.encryption`.

<AccordionGroup>
  <Accordion title="Başlangıç davranışı">
    `encryption: true` ile `startupVerification` varsayılan olarak `"if-unverified"` olur. Başlangıçta doğrulanmamış bir cihaz, başka bir Matrix istemcisinde öz doğrulama ister; yinelenen istekleri atlar ve bir bekleme süresi uygular (varsayılan olarak 24 saat). `startupVerificationCooldownHours` ile ayarlayın veya `startupVerification: "off"` ile devre dışı bırakın.

    Başlangıç ayrıca mevcut gizli veri depolamayı ve çapraz imzalama kimliğini yeniden kullanan temkinli bir kripto başlatma geçişi çalıştırır. Başlatma durumu bozuksa OpenClaw, `channels.matrix.password` olmadan bile korumalı bir onarım dener; homeserver parola UIA'sı gerektiriyorsa başlangıç bir uyarı kaydeder ve ölümcül olmayan biçimde devam eder. Zaten sahip tarafından imzalanmış cihazlar korunur.

    Tam yükseltme akışı için [Matrix geçişi](/tr/channels/matrix-migration) bölümüne bakın.

  </Accordion>

  <Accordion title="Doğrulama bildirimleri">
    Matrix, doğrulama yaşam döngüsü bildirimlerini katı DM doğrulama odasına `m.notice` mesajları olarak gönderir: istek, hazır ("Emojiyle doğrulayın" yönlendirmesiyle), başlangıç/tamamlama ve kullanılabilir olduğunda SAS (emoji/ondalık) ayrıntıları.

    Başka bir Matrix istemcisinden gelen istekler izlenir ve otomatik olarak kabul edilir. Öz doğrulama için OpenClaw, SAS akışını otomatik olarak başlatır ve emoji doğrulaması kullanılabilir olduğunda kendi tarafını onaylar; yine de Matrix istemcinizde emojileri karşılaştırıp "They match" seçeneğini onaylamanız gerekir.

    Doğrulama sistemi bildirimleri aracı sohbet işlem hattına iletilmez.

  </Accordion>

  <Accordion title="Silinmiş veya geçersiz Matrix cihazı">
    `verify status`, mevcut cihazın artık homeserver'da listelenmediğini söylüyorsa yeni bir OpenClaw Matrix cihazı oluşturun. Parolayla oturum açmak için:

```bash
openclaw matrix account add \
  --account assistant \
  --homeserver https://matrix.example.org \
  --user-id '@assistant:example.org' \
  --password '<password>' \
  --device-name OpenClaw-Gateway
```

    Token kimlik doğrulaması için Matrix istemcinizde veya yönetici arayüzünde yeni bir erişim token'ı oluşturun, ardından OpenClaw'ı güncelleyin:

```bash
openclaw matrix account add \
  --account assistant \
  --homeserver https://matrix.example.org \
  --access-token '<token>'
```

    `assistant` yerine başarısız komuttaki hesap kimliğini yazın veya varsayılan hesap için `--account` seçeneğini atlayın.

  </Accordion>

  <Accordion title="Cihaz temizliği">
    OpenClaw tarafından yönetilen eski cihazlar birikebilir. Listeleyin ve eski olanları temizleyin:

```bash
openclaw matrix devices list
openclaw matrix devices prune-stale
```

  </Accordion>

  <Accordion title="Kripto deposu">
    Matrix E2EE, IndexedDB uyumluluk katmanı olarak `fake-indexeddb` ile resmi `matrix-js-sdk` Rust kripto yolunu kullanır. Kripto durumu `crypto-idb-snapshot.json` konumunda kalıcı olarak saklanır (kısıtlayıcı dosya izinleriyle).

    Şifrelenmiş çalışma zamanı durumu `~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/` altında bulunur ve eşitleme deposunu, kripto deposunu, kurtarma anahtarını, IDB anlık görüntüsünü, ileti dizisi bağlamalarını ve başlangıç doğrulama durumunu içerir. Token değiştiğinde ancak hesap kimliği aynı kaldığında OpenClaw, önceki durumun görünür kalması için mevcut en uygun kökü yeniden kullanır.

    Eski bir token karmasına ait tek bir kök, normal bir token döndürme sürekliliği yolu olabilir. OpenClaw `matrix: multiple populated token-hash storage roots detected` kaydını oluşturursa hesap dizinini inceleyin ve eski eş kökleri yalnızca seçilen etkin kökün sağlıklı olduğunu doğruladıktan sonra arşivleyin. Eski kökleri hemen silmek yerine bir `_archive/` dizinine taşımayı tercih edin.

  </Accordion>
</AccordionGroup>

## Profil yönetimi

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

Her iki seçeneği de tek bir çağrıda iletin. Matrix, `mxc://` avatar URL'lerini doğrudan kabul eder; `http://`/`https://` iletildiğinde önce dosya yüklenir ve çözümlenen `mxc://` URL'si `channels.matrix.avatarUrl` içine (veya hesaba özel geçersiz kılma ayarına) kaydedilir.

## İleti dizileri

Matrix, hem otomatik yanıtlar hem de mesaj aracı gönderimleri için yerel ileti dizilerini destekler. Davranışı iki bağımsız ayar denetler:

### Oturum yönlendirmesi (`sessionScope`)

`dm.sessionScope`, Matrix DM odalarının OpenClaw oturumlarıyla nasıl eşleneceğini belirler:

- `"per-user"` (varsayılan): aynı yönlendirilmiş eşe sahip tüm DM odaları tek bir oturumu paylaşır.
- `"per-room"`: aynı eş için bile her Matrix DM odası kendi oturum anahtarını alır.

Açık konuşma bağlamaları her zaman `sessionScope` ayarından önceliklidir; bağlanmış odalar ve ileti dizileri seçtikleri hedef oturumu korur.

### Yanıtların ileti dizisine eklenmesi (`threadReplies`)

`threadReplies`, botun yanıtını nereye göndereceğini belirler:

- `"off"`: yanıtlar üst düzeyde gönderilir. İleti dizisinden gelen mesajlar üst oturumda kalır.
- `"inbound"`: yalnızca gelen mesaj zaten bir ileti dizisindeyse yanıtı o ileti dizisinin içinde gönderir.
- `"always"`: yanıtı tetikleyici mesajı kök alan bir ileti dizisinin içinde gönderir; bu konuşma, ilk tetikleyiciden itibaren eşleşen ileti dizisi kapsamlı bir oturum üzerinden yönlendirilir.

`dm.threadReplies` bunu yalnızca DM'ler için geçersiz kılar; örneğin DM'leri düz tutarken oda ileti dizilerini yalıtılmış tutar.

### İleti dizisi devralma ve eğik çizgi komutları

- İleti dizisinden gelen mesajlar, ek ajan bağlamı olarak ileti dizisinin kök mesajını içerir.
- Açık bir `threadId` sağlanmadığı sürece mesaj aracı gönderimleri, aynı odayı (veya aynı DM kullanıcı hedefini) hedeflerken geçerli Matrix ileti dizisini otomatik olarak devralır.
- DM kullanıcı hedefinin yeniden kullanılması yalnızca geçerli oturum meta verileri aynı Matrix hesabındaki aynı DM eşini kanıtladığında devreye girer; aksi takdirde OpenClaw normal kullanıcı kapsamlı yönlendirmeye geri döner.
- `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` ve ileti dizisine bağlı `/acp spawn`, Matrix odalarında ve DM'lerde çalışır.
- Üst düzey `/focus`, `threadBindings.spawnSessions` etkinleştirildiğinde yeni bir Matrix ileti dizisi oluşturur ve bunu hedef oturuma bağlar.
- Mevcut bir Matrix ileti dizisi içinde `/focus` veya `/acp spawn --thread here` çalıştırmak, bu ileti dizisini bulunduğu yerde bağlar.

OpenClaw, aynı paylaşılan oturumda başka bir DM odasıyla çakışan bir Matrix DM odası algıladığında, `/focus` kaçış yoluna işaret eden ve bir `dm.sessionScope` değişikliği öneren tek seferlik bir `m.notice` gönderir. Bildirim yalnızca ileti dizisi bağlamaları etkinleştirildiğinde görünür.

## ACP konuşma bağlamaları

Matrix odaları, DM'ler ve mevcut Matrix ileti dizileri, sohbet yüzeyi değiştirilmeden kalıcı ACP çalışma alanlarına dönüşebilir.

Hızlı operatör akışı:

- Kullanmaya devam etmek için Matrix DM'si, odası veya mevcut ileti dizisi içinde `/acp spawn codex --bind here` çalıştırın.
- Üst düzey bir DM veya odada, geçerli DM/oda sohbet yüzeyi olarak kalır ve gelecekteki mesajlar oluşturulan ACP oturumuna yönlendirilir.
- Mevcut bir ileti dizisi içinde `--bind here`, geçerli ileti dizisini bulunduğu yerde bağlar.
- `/new` ve `/reset`, aynı bağlı ACP oturumunu bulunduğu yerde sıfırlar.
- `/acp close`, ACP oturumunu kapatır ve bağlamayı kaldırır.

`--bind here` bir alt Matrix ileti dizisi oluşturmaz. `threadBindings.spawnSessions`, OpenClaw'ın bir alt ileti dizisi oluşturması veya bağlaması gereken `/acp spawn --thread auto|here` işlemini denetler.

### İleti dizisi bağlama yapılandırması

Matrix, `session.threadBindings` içindeki genel varsayılanları devralır ve kanal bazında geçersiz kılmaları destekler:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSessions`: hem alt ajan hem de ACP ileti dizisi oluşturmalarını denetler.
- Kullanımdan kaldırılan `threadBindings.spawnSubagentSessions` / `threadBindings.spawnAcpSessions` anahtarları, `openclaw doctor --fix` tarafından `spawnSessions` biçimine geçirilir.
- `threadBindings.defaultSpawnContext`

Matrix ileti dizisine bağlı oturum oluşturmaları varsayılan olarak etkindir. Üst düzey `/focus` ve `/acp spawn --thread auto|here` işlemlerinin Matrix ileti dizileri oluşturmasını/bağlamasını engellemek için `threadBindings.spawnSessions: false` ayarını yapın. Yerel alt ajan ileti dizisi oluşturmalarının üst transkripti çatallamaması gerektiğinde `threadBindings.defaultSpawnContext: "isolated"` ayarını yapın.

## Tepkiler

Matrix, giden tepkileri, gelen tepki bildirimlerini ve alındı tepkilerini destekler.

Giden tepki araçları `channels.matrix.actions.reactions` tarafından denetlenir:

- `react`, bir Matrix olayına tepki ekler.
- `reactions`, bir Matrix olayının geçerli tepki özetini listeler.
- `emoji=""`, botun bu olaydaki kendi tepkilerini kaldırır.
- `remove: true`, bottan yalnızca belirtilen emoji tepkisini kaldırır.

**Çözümleme sırası** (tanımlanan ilk değer kazanır):

| Ayar                 | Sıra                                                                               |
| ----------------------- | ----------------------------------------------------------------------------------- |
| `ackReaction`           | hesap bazında -> kanal -> `messages.ackReaction` -> ajan kimliği emojisine geri dönüş   |
| `ackReactionScope`      | hesap bazında -> kanal -> `messages.ackReactionScope` -> varsayılan `"group-mentions"` |
| `reactionNotifications` | hesap bazında -> kanal -> varsayılan `"own"`                                           |

`reactionNotifications: "own"`, bot tarafından yazılmış Matrix mesajlarını hedefleyen eklenmiş `m.reaction` olaylarını iletir; `"off"` tepki sistemi olaylarını devre dışı bırakır. Tepki kaldırmaları sistem olaylarına dönüştürülmez; Matrix bunları bağımsız `m.reaction` kaldırmaları olarak değil, redaksiyonlar olarak sunar.

## Geçmiş bağlamı

- `channels.matrix.historyLimit`, bir oda mesajı ajanı tetiklediğinde kaç adet son oda mesajının `InboundHistory` olarak ekleneceğini denetler. `messages.groupChat.historyLimit` ayarına geri döner; ikisi de ayarlanmamışsa geçerli varsayılan `0` değeridir (devre dışı).
- Matrix oda geçmişi yalnızca odaya özgüdür; DM'ler normal oturum geçmişini kullanmaya devam eder.
- Oda geçmişi yalnızca bekleyen mesajları içerir: OpenClaw, henüz bir yanıtı tetiklememiş oda mesajlarını arabelleğe alır ve bir bahsetme veya başka bir tetikleyici geldiğinde bu pencerenin anlık görüntüsünü oluşturur.
- Geçerli tetikleyici mesaj `InboundHistory` içine eklenmez; bu tur için ana gelen gövdede kalır.
- Aynı Matrix olayının yeniden denemeleri, daha yeni oda mesajlarına doğru ilerlemek yerine özgün geçmiş anlık görüntüsünü yeniden kullanır.

## Bağlam görünürlüğü

Matrix, getirilen yanıt metni, ileti dizisi kökleri ve bekleyen geçmiş gibi ek oda bağlamı için paylaşılan `contextVisibility` denetimini destekler.

- `contextVisibility: "all"` varsayılandır. Ek bağlam alındığı biçimde korunur.
- `contextVisibility: "allowlist"`, ek bağlamı etkin oda/kullanıcı izin listesi denetimlerinin izin verdiği göndericilerle sınırlar.
- `contextVisibility: "allowlist_quote"`, `allowlist` gibi davranır ancak açıkça alıntılanmış bir yanıtı yine de korur.

Bu, gelen mesajın kendisinin bir yanıtı tetikleyip tetikleyemeyeceğini değil, yalnızca ek bağlamın görünürlüğünü etkiler. Tetikleyici yetkilendirmesi hâlâ `groupPolicy`, `groups`, `groupAllowFrom` ve DM ilkesi ayarlarından gelir.

## DM ve oda ilkesi

```json5
{
  channels: {
    matrix: {
      dm: {
        policy: "allowlist",
        allowFrom: ["@admin:example.org"],
        threadReplies: "off",
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": { requireMention: true },
      },
    },
  },
}
```

Odaları çalışır durumda tutarken DM'leri tamamen sessize almak için `dm.enabled: false` ayarını yapın:

```json5
{
  channels: {
    matrix: {
      dm: { enabled: false },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
    },
  },
}
```

Bahsetmeyle tetikleme ve izin listesi davranışı için [Gruplar](/tr/channels/groups) bölümüne bakın.

Matrix DM'leri için eşleştirme örneği:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

Onaylanmamış bir Matrix kullanıcısı onay verilmeden önce mesaj göndermeyi sürdürürse OpenClaw aynı bekleyen eşleştirme kodunu yeniden kullanır ve yeni bir kod oluşturmak yerine kısa bir bekleme süresinden sonra hatırlatma yanıtı gönderebilir.

Paylaşılan DM eşleştirme akışı ve depolama düzeni için [Eşleştirme](/tr/channels/pairing) bölümüne bakın.

## Doğrudan oda onarımı

Doğrudan mesaj durumu saparsa OpenClaw, canlı DM yerine eski tek kişilik odalara işaret eden eski `m.direct` eşlemeleriyle kalabilir. Bir eş için geçerli eşlemeyi inceleyin:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

Onarın:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

Her iki komut da çok hesaplı kurulumlar için `--account <id>` kabul eder. Onarım akışı:

- `m.direct` içinde zaten eşlenmiş olan katı bir 1:1 DM'yi tercih eder
- bu kullanıcıyla şu anda katılınmış herhangi bir katı 1:1 DM'ye geri döner
- sağlıklı bir DM yoksa yeni bir doğrudan oda oluşturur ve `m.direct` değerini yeniden yazar

Eski odaları otomatik olarak silmez. Sağlıklı DM'yi seçer ve gelecekteki Matrix gönderimlerinin, doğrulama bildirimlerinin ve diğer doğrudan mesaj akışlarının doğru odayı hedeflemesi için eşlemeyi günceller.

## Çalıştırma onayları

Matrix, yerel bir onay istemcisi olarak çalışabilir. `channels.matrix.execApprovals` altında (veya hesap bazında geçersiz kılma için `channels.matrix.accounts.<account>.execApprovals` altında) yapılandırın:

- `enabled`: onayları Matrix'e özgü istemler aracılığıyla iletir. Ayarlanmamış olması veya `"auto"`, en az bir onaylayan çözümlenebildiğinde otomatik olarak etkinleştirir; açıkça devre dışı bırakmak için `false` ayarını yapın.
- `approvers`: çalıştırma isteklerini onaylamasına izin verilen Matrix kullanıcı kimlikleri (`@owner:example.org`). `channels.matrix.dm.allowFrom` ayarına geri döner.
- `target`: istemlerin nereye gönderileceğini belirler. `"dm"` (varsayılan) onaylayanların DM'lerine gönderir; `"channel"` kaynak odaya veya DM'ye gönderir; `"both"` her ikisine de gönderir.
- `agentFilter` / `sessionFilter`: hangi ajanların/oturumların Matrix üzerinden iletimi tetikleyeceğine yönelik isteğe bağlı izin listeleri.

Yetkilendirme, onay türleri arasında küçük farklılıklar gösterir:

- **Exec onayları**, `execApprovals.approvers` kullanır ve kullanılamazsa `dm.allowFrom` seçeneğine geri döner.
- **Plugin onayları** yalnızca `dm.allowFrom` üzerinden yetkilendirilir.

Her iki tür de Matrix tepki kısayollarını ve mesaj güncellemelerini paylaşır. Onaylayıcılar, birincil onay mesajında tepki kısayollarını görür:

- ✅ bir kez izin ver
- ❌ reddet
- ♾️ her zaman izin ver (etkin exec politikası buna izin verdiğinde)

Yedek eğik çizgi komutları: `/approve <id> allow-once`, `/approve <id> allow-always`, `/approve <id> deny`.

Yalnızca çözümlenmiş onaylayıcılar onaylayabilir veya reddedebilir. Exec onaylarının kanal üzerinden teslimi komut metnini içerir; `channel` veya `both` yalnızca güvenilir odalarda etkinleştirilmelidir.

İlgili: [Exec onayları](/tr/tools/exec-approvals).

## Eğik çizgi komutları

Eğik çizgi komutları (`/new`, `/reset`, `/model`, `/focus`, `/unfocus`, `/agents`, `/session`, `/acp`, `/approve` vb.) doğrudan DM'lerde çalışır. OpenClaw, odalarda botun kendi Matrix bahsiyle öneklenmiş komutları da tanır; böylece `@bot:server /new`, özel bir bahis regex'i olmadan komut yolunu tetikler. Bu, kullanıcı komutu yazmadan önce botu sekmeyle tamamladığında Element ve benzeri istemcilerin gönderdiği oda tarzı `@mention /command` iletilerine botun yanıt vermeye devam etmesini sağlar.

Yetkilendirme kuralları geçerliliğini korur: komut gönderenler, düz mesajlarla aynı DM veya oda izin listesi/sahip politikalarını karşılamalıdır.

## Çoklu hesap

```json5
{
  channels: {
    matrix: {
      enabled: true,
      defaultAccount: "assistant",
      dm: { policy: "pairing" },
      accounts: {
        assistant: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_assistant_xxx",
          encryption: true,
        },
        alerts: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_alerts_xxx",
          dm: {
            policy: "allowlist",
            allowFrom: ["@ops:example.org"],
            threadReplies: "off",
          },
        },
      },
    },
  },
}
```

**Devralma:**

- Üst düzey `channels.matrix` değerleri, bir hesap bunları geçersiz kılmadığı sürece adlandırılmış hesaplar için varsayılan görevi görür.
- Devralınan bir oda girdisini `groups.<room>.account` ile belirli bir hesapla sınırlandırın. `account` içermeyen girdiler hesaplar arasında paylaşılır; varsayılan hesap üst düzeyde yapılandırıldığında `account: "default"` çalışmaya devam eder.

**Varsayılan hesap seçimi:**

- Örtük yönlendirme, yoklama ve CLI komutlarının tercih edeceği adlandırılmış hesabı seçmek için `defaultAccount` ayarlayın.
- Birden çok hesabınız varsa ve bunlardan birinin adı tam olarak `default` ise OpenClaw, `defaultAccount` ayarlanmamış olsa bile onu örtük olarak kullanır.
- Birden çok adlandırılmış hesap bulunurken varsayılan hesap seçilmemişse CLI komutları tahminde bulunmayı reddeder; `defaultAccount` ayarlayın veya `--account <id>` iletin.
- Üst düzey `channels.matrix.*` bloğu, yalnızca kimlik doğrulaması tamamlandığında (`homeserver` + `accessToken` veya `homeserver` + `userId` + `password`) örtük `default` hesabı olarak değerlendirilir. Önbelleğe alınmış kimlik bilgileri kimlik doğrulamasını karşıladığında adlandırılmış hesaplar `homeserver` + `userId` üzerinden keşfedilebilir olmaya devam eder.

**Yükseltme:**

- OpenClaw, onarım veya kurulum sırasında tek hesaplı bir yapılandırmayı çoklu hesaba yükselttiğinde, mevcut bir adlandırılmış hesap varsa ya da `defaultAccount` zaten bir hesabı gösteriyorsa bu hesabı korur. Yalnızca Matrix kimlik doğrulama/önyükleme anahtarları yükseltilen hesaba taşınır; paylaşılan teslimat politikası anahtarları üst düzeyde kalır.

Paylaşılan çoklu hesap kalıbı için [Yapılandırma başvurusuna](/tr/gateway/config-channels#multi-account-all-channels) bakın.

## Özel/LAN homeserver'ları

OpenClaw, siz hesap bazında izin vermediğiniz sürece SSRF koruması amacıyla özel/dahili Matrix homeserver'larını varsayılan olarak engeller.

Homeserver'ınız localhost, bir LAN/Tailscale IP'si veya dahili bir ana bilgisayar adında çalışıyorsa söz konusu hesap için `network.dangerouslyAllowPrivateNetwork` seçeneğini etkinleştirin:

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
      accessToken: "syt_internal_xxx",
    },
  },
}
```

CLI kurulum örneği:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

Bu isteğe bağlı etkinleştirme yalnızca güvenilir özel/dahili hedeflere izin verir. `http://matrix.example.org:8008` gibi herkese açık şifresiz homeserver'lar engellenmeye devam eder. Mümkün olduğunda `https://` tercih edin.

## Matrix trafiğine proxy uygulama

Matrix dağıtımınız açıkça belirtilmiş bir giden HTTP(S) proxy'si gerektiriyorsa `channels.matrix.proxy` ayarlayın:

```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
    },
  },
}
```

Adlandırılmış hesaplar, üst düzey varsayılanı `channels.matrix.accounts.<id>.proxy` ile geçersiz kılabilir. OpenClaw, çalışma zamanı Matrix trafiği ve hesap durumu yoklamaları için aynı proxy ayarını kullanır.

## Hedef çözümleme

Matrix, OpenClaw'ın oda veya kullanıcı hedefi istediği her yerde şu hedef biçimlerini kabul eder:

- Kullanıcılar: `@user:server`, `user:@user:server` veya `matrix:user:@user:server`
- Odalar: `!room:server`, `room:!room:server` veya `matrix:room:!room:server`
- Takma adlar: `#alias:server`, `channel:#alias:server` veya `matrix:channel:#alias:server`

Matrix oda kimlikleri büyük/küçük harfe duyarlıdır. Açık teslimat hedeflerini, cron işlerini, bağlamaları veya izin listelerini yapılandırırken Matrix'teki oda kimliğinin büyük/küçük harf düzenini aynen kullanın. OpenClaw, depolama amacıyla dahili oturum anahtarlarını standart biçimde tutar; bu nedenle küçük harfli anahtarlar Matrix teslimat kimlikleri için güvenilir bir kaynak değildir.

Canlı dizin araması, oturum açmış Matrix hesabını kullanır:

- Kullanıcı aramaları, ilgili homeserver'daki Matrix kullanıcı dizinini sorgular.
- Oda aramaları, açık oda kimliklerini ve takma adları doğrudan kabul eder. Katılınmış oda adlarını arama en iyi çaba esasına dayanır ve yalnızca `dangerouslyAllowNameMatching: true` ayarlandığında çalışma zamanı oda izin listelerine uygulanır.
- Bir oda adı kimliğe veya takma ada çözümlenemiyorsa çalışma zamanı izin listesi çözümlemesinde yok sayılır.

## Yapılandırma başvurusu

İzin listesi tarzındaki kullanıcı alanları (`groupAllowFrom`, `dm.allowFrom`, `groups.<room>.users`) tam Matrix kullanıcı kimliklerini kabul eder (en güvenlisi). Kimlik olmayan girdiler varsayılan olarak yok sayılır. `dangerouslyAllowNameMatching: true` ayarlanmışsa tam Matrix dizini görünen ad eşleşmeleri başlangıçta ve izleyici çalışırken izin listesi her değiştiğinde çözümlenir; çözümlenemeyen girdiler çalışma zamanında yok sayılır.

Oda izin listesi anahtarları (`groups`, eski `rooms`) oda kimlikleri veya takma adlar olmalıdır. Düz oda adı anahtarları varsayılan olarak yok sayılır; `dangerouslyAllowNameMatching: true`, katılınmış oda adlarına yönelik en iyi çaba esaslı aramayı geri getirir.

### Hesap ve bağlantı

- `enabled`: kanalı etkinleştirir veya devre dışı bırakır.
- `name`: hesabın isteğe bağlı görünen etiketi.
- `defaultAccount`: birden çok Matrix hesabı yapılandırıldığında tercih edilen hesap kimliği.
- `accounts`: hesap bazında adlandırılmış geçersiz kılmalar. Üst düzey `channels.matrix` değerleri varsayılan olarak devralınır.
- `homeserver`: homeserver URL'si; örneğin `https://matrix.example.org`.
- `network.dangerouslyAllowPrivateNetwork`: bu hesabın `localhost`, LAN/Tailscale IP'leri veya dahili ana bilgisayar adlarına bağlanmasına izin verir.
- `proxy`: Matrix trafiği için isteğe bağlı HTTP(S) proxy URL'si. Hesap bazında geçersiz kılma desteklenir.
- `userId`: tam Matrix kullanıcı kimliği (`@bot:example.org`).
- `accessToken`: belirteç tabanlı kimlik doğrulaması için erişim belirteci. env/file/exec sağlayıcılarında düz metin ve SecretRef değerleri desteklenir ([Gizli Bilgi Yönetimi](/tr/gateway/secrets)).
- `password`: parola tabanlı oturum açma parolası. Düz metin ve SecretRef değerleri desteklenir.
- `deviceId`: açık Matrix cihaz kimliği.
- `deviceName`: parolayla oturum açma sırasında kullanılan cihaz görünen adı.
- `avatarUrl`: profil eşitlemesi ve `profile set` güncellemeleri için depolanan öz avatar URL'si.
- `initialSyncLimit`: başlangıç eşitlemesi sırasında alınan en fazla olay sayısı.

### Şifreleme

- `encryption`: E2EE'yi etkinleştirir. Varsayılan: `false`.
- `startupVerification`: `"if-unverified"` (E2EE açıkken varsayılan) veya `"off"`. Bu cihaz doğrulanmamışsa başlangıçta otomatik olarak öz doğrulama ister.
- `startupVerificationCooldownHours`: sonraki otomatik başlangıç isteğinden önceki bekleme süresi. Varsayılan: `24`.

### Erişim ve politika

- `groupPolicy`: `"open"`, `"allowlist"` veya `"disabled"`. Varsayılan: `"allowlist"`.
- `groupAllowFrom`: oda trafiği için kullanıcı kimlikleri izin listesi.
- `mentionPatterns`: oda bahisleri için kapsamlı regex kalıpları. `{ mode: "allow"|"deny", allowIn: [roomId, ...], denyIn: [roomId, ...] }` içeren nesne. Yapılandırılmış `agents.entries.*.groupChat.mentionPatterns` değerlerinin oda bazında uygulanıp uygulanmayacağını denetler.
- `dm.enabled`: `false` olduğunda tüm DM'leri yok sayar. Varsayılan: `true`.
- `dm.policy`: `"pairing"` (varsayılan), `"allowlist"`, `"open"` veya `"disabled"`. Bot odaya katılıp odayı DM olarak sınıflandırdıktan sonra uygulanır; davet işlemeyi etkilemez.
- `dm.allowFrom`: DM trafiği için kullanıcı kimlikleri izin listesi.
- `dm.sessionScope`: `"per-user"` (varsayılan) veya `"per-room"`.
- `dm.threadReplies`: yanıt iş parçacığı oluşturma için yalnızca DM'ye özgü geçersiz kılma (`"off"`, `"inbound"`, `"always"`).
- `allowBots`: yapılandırılmış diğer Matrix bot hesaplarından gelen mesajları kabul eder (`true` veya `"mentions"`).
- `allowlistOnly`: `true` olduğunda tüm etkin DM politikalarını (`"disabled"` hariç) ve `"open"` grup politikalarını `"allowlist"` olmaya zorlar. `"disabled"` politikalarını değiştirmez.
- `dangerouslyAllowNameMatching`: `true` olduğunda kullanıcı izin listesi girdileri için Matrix görünen ad dizini aramasına ve oda izin listesi anahtarları için katılınmış oda adı aramasına izin verir. Tam `@user:server` kimliklerini ve oda kimliklerini veya takma adları tercih edin.
- `autoJoin`: `"always"`, `"allowlist"` veya `"off"`. Varsayılan: `"off"`. DM tarzı davetler dâhil her Matrix davetine uygulanır.
- `autoJoinAllowlist`: `autoJoin`, `"allowlist"` olduğunda izin verilen odalar/takma adlar. Takma ad girdileri, davet edilen odanın bildirdiği duruma göre değil homeserver'a göre çözümlenir.
- `contextVisibility`: ek bağlam görünürlüğü (`"all"` varsayılan, `"allowlist"`, `"allowlist_quote"`).

### Yanıt davranışı

- `replyToMode`: `"off"` (varsayılan), `"first"`, `"all"` veya `"batched"`.
- `threadReplies`: `"off"` (açıkça ayarlanmadığı sürece üst düzey varsayılan `"inbound"` olarak çözümlenir), `"inbound"` veya `"always"`.
- `threadBindings`: iş parçacığına bağlı oturum yönlendirmesi ve yaşam döngüsü için kanal bazında geçersiz kılmalar.
- `streaming`: iç içe nesne `{ mode, chunkMode, block: { enabled, coalesce }, preview: { toolProgress }, progress: { label, labels, maxLines, maxLineChars, toolProgress } }`. `mode`; `"off"` (varsayılan), `"partial"`, `"quiet"` veya `"progress"` değerlerinden biridir. Eski skaler/boole yazımları `openclaw doctor --fix` aracılığıyla taşınır.
- `streaming.block.enabled`: `true` olduğunda tamamlanan asistan blokları ayrı ilerleme mesajları olarak tutulur. Varsayılan: `false`.
- `markdown`: giden metin için isteğe bağlı Markdown işleme yapılandırması.
- `responsePrefix`: giden yanıtların başına eklenen isteğe bağlı dize.
- `textChunkLimit`: `streaming.chunkMode: "length"` olduğunda giden parçaların karakter cinsinden boyutu. Varsayılan: `4000`.
- `streaming.chunkMode`: `"length"` (varsayılan, karakter sayısına göre böler) veya `"newline"` (satır sınırlarında böler).
- `historyLimit`: bir oda mesajı ajanı tetiklediğinde `InboundHistory` olarak dahil edilen son oda mesajlarının sayısı. `messages.groupChat.historyLimit` değerine geri döner; etkin varsayılan `0` (devre dışı).
- `mediaMaxMb`: giden gönderimler ve gelen içeriklerin işlenmesi için MB cinsinden medya boyutu üst sınırı. Varsayılan: `20`.

### Tepki ayarları

- `ackReaction`: bu kanal/hesap için alındı tepkisi geçersiz kılması.
- `ackReactionScope`: kapsam geçersiz kılması (`"group-mentions"` varsayılan, `"group-all"`, `"direct"`, `"all"`, `"none"`, `"off"`).
- `reactionNotifications`: gelen tepki bildirim modu (`"own"` varsayılan, `"off"`).

### Araçlar ve oda bazında geçersiz kılmalar

- `actions`: eylem bazında araç kısıtlaması (`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).
- `groups`: oda bazında politika eşlemesi. Oturum kimliği, çözümlemeden sonra kararlı oda kimliğini kullanır. (`rooms` eski bir diğer addır.)
  - `groups.<room>.account`: devralınan bir oda girdisini belirli bir hesapla sınırlar.
  - `groups.<room>.enabled`: oda bazında açma/kapatma ayarı. `false` olduğunda oda, eşlemede yokmuş gibi yok sayılır.
  - `groups.<room>.requireMention`: kanal düzeyindeki bahsetme gereksiniminin oda bazında geçersiz kılınması.
  - `groups.<room>.allowBots`: kanal düzeyindeki ayarın oda bazında geçersiz kılınması (`true` veya `"mentions"`).
  - `groups.<room>.botLoopProtection`: bottan bota döngü koruması bütçesinin oda bazında geçersiz kılınması.
  - `groups.<room>.users`: oda bazında gönderen izin listesi.
  - `groups.<room>.tools`: oda bazında araç izin/ret geçersiz kılmaları.
  - `groups.<room>.autoReply`: oda bazında bahsetme kısıtlaması geçersiz kılması. `true`, bu oda için bahsetme gereksinimlerini devre dışı bırakır; `false` bunları yeniden zorunlu kılar.
  - `groups.<room>.skills`: oda bazında Skills filtresi.
  - `groups.<room>.systemPrompt`: oda bazında sistem istemi parçası.

### Exec onayı ayarları

- `execApprovals.enabled`: exec onaylarını Matrix'e özgü istemler aracılığıyla iletir.
- `execApprovals.approvers`: onay vermesine izin verilen Matrix kullanıcı kimlikleri. `dm.allowFrom` değerine geri döner.
- `execApprovals.target`: `"dm"` (varsayılan), `"channel"` veya `"both"`.
- `execApprovals.agentFilter` / `execApprovals.sessionFilter`: iletim için isteğe bağlı ajan/oturum izin listeleri.

## İlgili

- [Kanallara Genel Bakış](/tr/channels) - desteklenen tüm kanallar
- [Eşleştirme](/tr/channels/pairing) - DM kimlik doğrulaması ve eşleştirme akışı
- [Gruplar](/tr/channels/groups) - grup sohbeti davranışı ve bahsetme kısıtlaması
- [Kanal Yönlendirmesi](/tr/channels/channel-routing) - mesajlar için oturum yönlendirmesi
- [Güvenlik](/tr/gateway/security) - erişim modeli ve sağlamlaştırma
