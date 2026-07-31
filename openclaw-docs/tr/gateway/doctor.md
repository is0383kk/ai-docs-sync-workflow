---
read_when:
    - Doctor geçişleri ekleme veya değiştirme
    - Geriye dönük uyumluluğu bozan yapılandırma değişikliklerinin kullanıma sunulması
sidebarTitle: Doctor
summary: 'Doctor komutu: sistem durumu kontrolleri, yapılandırma geçişleri ve onarım adımları'
title: Doktor
x-i18n:
    generated_at: "2026-07-26T23:59:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f599553a2455759cd0fe56bafbc16948f7ab4d381d344b08a496bf19c9dc636
    source_path: gateway/doctor.md
    workflow: 16
---

`openclaw doctor`, OpenClaw için onarım ve geçiş aracıdır. Eski yapılandırmayı/durumu düzeltir, sistem sağlığını denetler ve uygulanabilir onarım adımları sunar.

## Hızlı başlangıç

```bash
openclaw doctor
```

### Başsız ve otomasyon modları

<Tabs>
  <Tab title="--yes">
    ```bash
    openclaw doctor --yes
    ```

    Sormadan varsayılanları kabul eder (uygun olduğunda yeniden başlatma/hizmet/sandbox onarım adımları dahil).

  </Tab>
  <Tab title="--fix">
    ```bash
    openclaw doctor --fix
    ```

    Önerilen onarımları sormadan uygular (`--repair` bir diğer addır).

  </Tab>
  <Tab title="--lint">
    ```bash
    openclaw doctor --lint
    openclaw doctor --lint --json
    ```

    CI veya ön kontrol otomasyonu için yapılandırılmış sistem sağlığı denetimleri çalıştırır. Salt okunurdur:
    istem, onarım, geçiş, yeniden başlatma veya durum yazma işlemi yapmaz.

  </Tab>
  <Tab title="--fix --force">
    ```bash
    openclaw doctor --fix --force
    ```

    Agresif onarımları da uygular (özel denetleyici yapılandırmalarının üzerine yazar).

  </Tab>
  <Tab title="--non-interactive">
    ```bash
    openclaw doctor --non-interactive
    ```

    Sormadan çalışır ve yalnızca güvenli geçişleri uygular (yapılandırma normalleştirmesi +
    disk üzerindeki durum taşıma işlemleri). İnsan onayı gerektiren yeniden başlatma/hizmet/sandbox
    eylemlerini atlar. Eski durum geçişleri algılandığında yine otomatik olarak çalışır.

  </Tab>
  <Tab title="--deep">
    ```bash
    openclaw doctor --deep
    ```

    Ek Gateway kurulumları için sistem hizmetlerini tarar (launchd/systemd/schtasks).

  </Tab>
</Tabs>

Yazmadan önce değişiklikleri incelemek için önce yapılandırma dosyasını açın:

```bash
cat ~/.openclaw/openclaw.json
```

## Salt okunur lint modu

`openclaw doctor --lint`, `openclaw doctor --fix` aracının otomasyona uygun kardeşidir.
Aynı Doctor kural kayıt defterini paylaşırlar ancak kuralları aynı şekilde
seçmez veya uygulamazlar:

| Mod                      | İstemler       | Yapılandırma/durum yazma | Çıktı                      | Kullanım amacı                         |
| ------------------------ | -------------- | ------------------------ | -------------------------- | -------------------------------------- |
| `openclaw doctor`       | evet           | hayır                    | anlaşılır sistem sağlığı raporu | bir kişinin durumu denetlemesi     |
| `openclaw doctor --fix`       | bazen          | evet, onarım ilkesiyle   | anlaşılır onarım günlüğü   | onaylanmış onarımları uygulama          |
| `openclaw doctor --lint`       | hayır          | hayır                    | yapılandırılmış bulgular   | CI, ön kontrol ve inceleme kapıları     |

Varsayılan `doctor --lint`, geniş ve güvenli otomasyon profilini çalıştırır:
statik, yerel ve CI ya da ön kontrol çıktısında yararlı olan denetimler. İsteğe
bağlı danışmanlık denetimlerini, ortama duyarlı denetimleri, canlı hizmete bağlı
denetimleri, hesap/çalışma alanı envanterini ve geçmiş temizliğini atlar. Bu isteğe
bağlı denetimler dahil kayıtlı lint denetiminin tamamını istediğinizde
`doctor --lint --all`, hedefli bir denetim içinse `--only <id>` kullanın.

`doctor --fix`, lint varsayılan profilini kullanmaz ve
`--all` kabul etmez. Doctor'ın sıralı onarım yolunu çalıştırır: modern
sistem sağlığı denetimleri isteğe bağlı bir `repair()` uygulaması
sağlayabilirken eski alanlar hâlâ eski Doctor onarım akışını kullanır. Bazı lint
bulguları kasıtlı olarak yalnızca tanılama amaçlıdır; dolayısıyla bir denetimin
`--lint --all` içinde görünmesi, `--fix` aracının o alanı
değiştireceği anlamına gelmez. Sözleşme, `detect()` (bulguları bildirir)
ile `repair()` (değişiklikleri/farkları/yan etkileri bildirir) öğelerini
birbirinden ayırır. Böylece lint denetimlerini değişiklik planlayıcılarına
dönüştürmeden gelecekteki bir `doctor --fix --dry-run` için yol açık tutulur.

Bazı yerleşik denetimler, varsayılan `doctor --lint` otomasyon profilinin
parçası hâline gelmeden `--all`, `--only` ve Doctor onarım
akışlarında kullanılabilmeleri için dahili olarak varsayılan biçimde devre dışıdır.
Bulgu önem derecesi yine her bulgu için yayınlanır (`info`,
`warning` veya `error`); varsayılan seçim bir önem derecesi
değildir.

```bash
openclaw doctor --lint
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --json
openclaw doctor --lint --all
openclaw doctor --lint --only core/doctor/gateway-config --json
```

JSON çıktı alanları:

- `ok`: herhangi bir bulgunun seçilen önem derecesi eşiğini karşılayıp karşılamadığı
- `checksRun` / `checksSkipped`: sayılar (profil, `--only` veya `--skip` nedeniyle atlananlar)
- `findings`: `checkId`, `severity`, `message` ile isteğe bağlı `path`, `line`, `column`, `ocPath`, `source`, `target`, `requirement`, `fixHint` içeren yapılandırılmış tanılamalar

Çıkış kodları:

| Kod | Anlamı                                                          |
| --- | --------------------------------------------------------------- |
| `0` | seçilen eşikte veya üzerinde bulgu yok                           |
| `1` | bir veya daha fazla bulgu seçilen eşiği karşıladı                |
| `2` | bulgular yayınlanamadan önce komut/çalışma zamanı hatası oluştu   |

Bayraklar:

- `--severity-min info|warning|error` (varsayılan `warning`): hem neyin yazdırılacağını hem de neyin sıfır dışı çıkışa neden olacağını belirler.
- `--all`: varsayılan otomasyon kümesinin dışında tutulan isteğe bağlı denetimler dahil kayıtlı tüm lint denetimlerini çalıştırır.
- `--only <id>` (tekrarlanabilir): yalnızca belirtilen denetim kimliklerini çalıştırır; bilinmeyen bir kimlik hata bulgusu olarak bildirilir.
- `--skip <id>` (tekrarlanabilir): çalışmanın geri kalanını etkin tutarken bir denetimi hariç tutar.
- `--json`, `--severity-min`, `--all`, `--only` ve `--skip`, `--lint` gerektirir; düz `openclaw doctor` ve `--fix` çalıştırmaları bunları reddeder.

## Yaptıkları (özet)

<AccordionGroup>
  <Accordion title="Sistem sağlığı, kullanıcı arayüzü ve güncellemeler">
    - Git kurulumları için isteğe bağlı ön kontrol güncellemesi (yalnızca etkileşimli).
    - Kullanıcı arayüzü protokolünün güncellik denetimi (protokol şeması daha yeniyse Control UI yeniden oluşturulur).
    - Sistem sağlığı denetimi + yeniden başlatma istemi.
    - Yalnızca sorunlu Skills ve Plugin notları; sağlıklı envanter `openclaw skills check` ve `openclaw plugins list` içinde kalır.

  </Accordion>
  <Accordion title="Yapılandırma ve geçişler">
    - Eski değer biçimleri için yapılandırma normalleştirmesi.
    - Eski düz `talk.*` alanlarından `talk.provider` + `talk.providers.<provider>` biçimine Talk yapılandırması geçişi.
    - Eski Chrome uzantısı yapılandırmaları ve Chrome MCP hazırlığı için tarayıcı geçiş denetimleri.
    - OpenCode sağlayıcı geçersiz kılma uyarıları (`models.providers.opencode` / `opencode-zen` / `opencode-go`).
    - Eski OpenAI Codex sağlayıcı/profil geçişi (`openai-codex` → `openai`) ve eski `models.providers.openai-codex` için gölgeleme uyarıları.
    - OpenAI Codex OAuth profilleri için OAuth TLS ön koşul denetimi.
    - `plugins.allow` kısıtlayıcıyken araç ilkesi hâlâ joker karakter veya Plugin'e ait araçlar istediğinde Plugin/araç izin listesi uyarıları.
    - Eski disk üzerindeki durum geçişi (oturumlar/agent dizini/WhatsApp kimlik doğrulaması).
    - Eski Plugin manifest sözleşme anahtarı geçişi (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
    - Eski Cron deposu geçişi (`jobId`, `schedule.cron`, üst düzey teslim/yük alanları, yük `provider`, `notify: true` Webhook geri dönüş işleri).
    - `agents.defaults`, `agents.entries.*` ve `models.providers.*` genelinde (model başına girdiler dahil) Codex CLI çalışma zamanı sabitleme onarımı (`agentRuntime.id: "codex-cli"` → `"codex"`).
    - Plugin'ler etkinleştirildiğinde eski Plugin yapılandırması temizliği; `plugins.enabled=false` olduğunda eski Plugin başvuruları etkisiz çevreleme yapılandırması olarak korunur.

  </Accordion>
  <Accordion title="Durum ve bütünlük">
    - Oturum kilit dosyası incelemesi ve eski kilitlerin temizlenmesi.
    - Etkilenen 2026.4.24 derlemelerinin oluşturduğu yinelenen istem yeniden yazma dalları için oturum transkripti onarımı.
    - Kilitlenmiş ana oturum ve alt agent yeniden başlatma-kurtarma mezar taşı algılaması. Doctor, engellenen oturumları bildirir ve yalnızca mevcut bir mezar taşıyla çakışan eski iptal işaretlerini onarır; otomatik kurtarmayı yeniden etkinleştirmez.
    - Durum bütünlüğü ve izin denetimleri (oturumlar, transkriptler, durum dizini).
    - Yerel olarak çalışırken yapılandırma dosyası izin denetimleri (chmod 600).
    - Model kimlik doğrulaması sağlığı: OAuth süresinin dolmasını denetler, süresi dolmak üzere olan token'ları yenileyebilir ve kimlik doğrulama profili bekleme süresi/devre dışı durumlarını bildirir.

  </Accordion>
  <Accordion title="Gateway, hizmetler ve denetleyiciler">
    - Sandbox etkinleştirildiğinde sandbox imajı onarımı.
    - Eski hizmet geçişi ve ek Gateway algılama.
    - Matrix kanalının eski durum geçişi (`--fix` / `--repair` modunda).
    - Gateway çalışma zamanı denetimleri (hizmet kurulu ancak çalışmıyor; önbelleğe alınmış launchd etiketi).
    - Kanal durumu uyarıları (çalışan Gateway üzerinden yoklanır).
    - Kanala özgü izin denetimleri `openclaw channels capabilities` altında bulunur; örneğin Discord ses kanalı izinleri `openclaw channels capabilities --channel discord --target channel:<channel-id>` ile denetlenir.
    - Yerel TUI istemcileri hâlâ çalışırken bozulmuş Gateway olay döngüsü sağlığı için WhatsApp yanıt verebilirlik denetimleri; `--fix` yalnızca doğrulanmış yerel TUI istemcilerini durdurur.
    - Birincil modeller, geri dönüşler, görüntü/video oluşturma modelleri, Heartbeat/alt agent/Compaction geçersiz kılmaları, hook'lar, kanal modeli geçersiz kılmaları ve oturum rota sabitlemelerindeki eski `openai-codex/*` model başvuruları için Codex rota onarımı; `--fix` bunları `openai/*` olarak yeniden yazar, `openai-codex:*` kimlik doğrulama profillerini/sırasını `openai:*` biçimine geçirir, eski oturum/tüm agent çalışma zamanı sabitlemelerini kaldırır ve onarılan etkin rotanın Codex'in uyumlu olup olmadığını belirlemesini sağlar.
    - İsteğe bağlı onarımla denetleyici yapılandırması denetimi (launchd/systemd/schtasks).
    - Kurulum veya güncelleme sırasında kabuk `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` değerlerini yakalayan Gateway hizmetleri için gömülü proxy ortamı temizliği.
    - Gateway çalışma zamanı denetimleri (desteklenmeyen eski Bun hizmetleri, sürüm yöneticisi yolları).
    - Gateway bağlantı noktası çakışması tanılamaları (varsayılan `18789`).

  </Accordion>
  <Accordion title="Kimlik doğrulama, güvenlik ve eşleştirme">
    - Açık DM ilkeleri için güvenlik uyarıları.
    - Yerel token modu için Gateway kimlik doğrulama denetimleri (token kaynağı yoksa token oluşturmayı önerir; token SecretRef yapılandırmalarının üzerine yazmaz).
    - Cihaz eşleştirme sorunu algılama (bekleyen ilk eşleştirme istekleri, bekleyen rol/kapsam yükseltmeleri, eski yerel cihaz token'ı önbelleği sapması ve eşleştirilmiş kayıt kimlik doğrulama sapması).

  </Accordion>
  <Accordion title="Çalışma alanı ve kabuk">
    - Linux'ta systemd linger denetimi.
    - Çalışma alanı önyükleme dosyası boyutu denetimi (bağlam dosyaları için kesilme/sınıra yaklaşma uyarıları).
    - Varsayılan agent için Skills hazırlık denetimi; eksik ikili dosya, ortam, yapılandırma veya işletim sistemi gereksinimleri bulunan izin verilmiş Skills öğelerini bildirir ve `--fix`, `skills.entries` içindeki kullanılamayan Skills öğelerini devre dışı bırakabilir.
    - Kabuk tamamlama durumu denetimi ve otomatik kurulum/yükseltme.
    - Bellek araması gömme sağlayıcısı hazırlık denetimi (yerel model, uzak API anahtarı veya QMD ikili dosyası).
    - Kaynak kurulum denetimleri (pnpm çalışma alanı uyuşmazlığı, eksik kullanıcı arayüzü varlıkları, eksik tsx ikili dosyası).
    - Güncellenmiş yapılandırmayı + sihirbaz meta verilerini yazar.

  </Accordion>
</AccordionGroup>

## Dreams kullanıcı arayüzü geriye dönük doldurma ve sıfırlama

  Control UI Dreams sahnesi, grounded dreaming iş akışı için **Geri Doldur**, **Sıfırla** ve **Grounded Verileri Temizle** eylemlerini içerir. Bunlar Gateway doctor tarzı RPC yöntemlerini kullanır ancak `openclaw doctor` CLI onarımının/geçişinin parçası **değildir**.

  | Eylem                    | Yaptığı işlem                                                                                                                                                                                    |
  | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | Geri Doldur              | Etkin çalışma alanındaki geçmiş `memory/YYYY-MM-DD.md` dosyalarını tarar, grounded REM günlük geçişini çalıştırır ve geri alınabilir geri doldurma girdilerini `DREAMS.md` içine yazar.          |
  | Sıfırla                  | Yalnızca işaretlenmiş geri doldurma günlük girdilerini `DREAMS.md` içinden kaldırır.                                                                                                      |
  | Grounded Verileri Temizle | Yalnızca geçmiş yeniden oynatmadan gelen, henüz canlı hatırlama veya günlük destek biriktirmemiş, aşamalandırılmış ve yalnızca grounded olan kısa süreli girdileri kaldırır.                       |

  Bunların hiçbiri `MEMORY.md` üzerinde değişiklik yapmaz, tam doctor geçişlerini çalıştırmaz veya grounded adaylarını kendiliğinden canlı kısa süreli yükseltme deposunda aşamalandırmaz. Grounded geçmiş yeniden oynatmayı normal derin yükseltme hattına beslemek için bunun yerine CLI akışını kullanın:

  ```bash
  openclaw memory rem-backfill --path ./memory --stage-short-term
  ```

  Bu işlem, grounded kalıcı adayları kısa süreli dreaming deposunda aşamalandırırken `DREAMS.md` inceleme yüzeyi olarak kalır.

  ## Ayrıntılı davranış ve gerekçe

  <AccordionGroup>
  <Accordion title="0. İsteğe bağlı güncelleme (git kurulumları)">
    Bu bir git çalışma kopyasıysa ve doctor etkileşimli olarak çalışıyorsa doctor çalıştırılmadan önce güncelleme (fetch/rebase/build) önerir.
  </Accordion>
  <Accordion title="1. Yapılandırma normalleştirmesi">
    Doctor, eski değer biçimlerini geçerli şemaya normalleştirir. Geçerli Talk konuşma yapılandırması `talk.provider` + `talk.providers.<provider>` biçimindedir ve gerçek zamanlı ses yapılandırması `talk.realtime.*` altındadır. Doctor, eski `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` / `talk.apiKey` biçimlerini sağlayıcı eşlemesine dönüştürür ve eski üst düzey gerçek zamanlı seçicileri (`talk.mode`, `talk.transport`, `talk.brain`, `talk.model`, `talk.voice`) `talk.realtime` içine yeniden yazar.

    Doctor ayrıca `plugins.allow` boş değilse ve araç politikası joker karakter veya plugin'e ait araç girdileri kullanıyorsa uyarır. `tools.allow: ["*"]` yalnızca gerçekten yüklenen plugin'lerin araçlarıyla eşleşir; özel plugin izin listesini atlamaz.

  </Accordion>
  <Accordion title="2. Eski yapılandırma anahtarı geçişleri">
    Yapılandırma, etkin bir geçişe sahip kullanımdan kaldırılmış bir anahtar içerdiğinde diğer komutlar çalışmayı reddeder ve `openclaw doctor` komutunu çalıştırmanızı ister. Doctor, hangi eski anahtarların bulunduğunu açıklar, uyguladığı geçişi gösterir ve `~/.openclaw/openclaw.json` dosyasını güncellenmiş şemayla yeniden yazar. Gateway başlatma işlemi eski yapılandırma biçimlerini reddeder ve `openclaw doctor --fix` komutunu çalıştırmanızı ister; başlatma sırasında `openclaw.json` dosyasını yeniden yazmaz. Cron işi deposu geçişleri de `openclaw doctor --fix` tarafından işlenir.

    <Note>
      Doctor, bir anahtar kullanımdan kaldırıldıktan sonra yalnızca yaklaşık iki ay
      boyunca otomatik geçişleri sağlar. Daha eski anahtarların (örneğin özgün
      `routing.queue`, `routing.bindings`, `routing.agents`/`defaultAgentId`,
      `routing.transcribeAudio`, üst düzey `agent.*` veya çoklu ajan öncesi
      yapılandırma biçimindeki üst düzey `identity`) artık bir geçiş yolu
      yoktur; bunları kullanan yapılandırma artık yeniden yazılmak yerine
      doğrulamada başarısız olur. Doctor'ın devam edebilmesi için bu anahtarları
      geçerli yapılandırma referansına göre elle düzeltin.
    </Note>

    Etkin geçişler:

    | Eski anahtar                                                                                    | Geçerli anahtar                                                                 |
    | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
    | `routing.allowFrom`                                                                              | `channels.whatsapp.allowFrom`                                                |
    | `routing.groupChat.requireMention`                                                               | `channels.whatsapp/telegram/imessage.groups."*".requireMention`             |
    | `routing.groupChat.historyLimit`                                                                 | `messages.groupChat.historyLimit`                                            |
    | `routing.groupChat.mentionPatterns`                                                              | `messages.groupChat.mentionPatterns`                                         |
    | `channels.telegram.requireMention`                                                               | `channels.telegram.groups."*".requireMention`                               |
    | `channels.webchat`, `gateway.webchat`                                                            | kaldırıldı (WebChat kullanımdan kaldırıldı)                                                 |
    | `channels.feishu.accounts.<accountId>.botName`                                                   | `channels.feishu.accounts.<accountId>.name`                                 |
    | `session.threadBindings.ttlHours`, `channels.<id>.threadBindings.ttlHours` (ve hesap başına)      | `...threadBindings.idleHours`                                               |
    | eski `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey`        | `talk.provider` + `talk.providers.<provider>`                               |
    | eski üst düzey gerçek zamanlı Talk seçicileri (`talk.mode`/`talk.transport`/`talk.brain`/`talk.model`/`talk.voice`) | `talk.realtime`                                                              |
    | `messages.tts`                                                                                  | üst düzey `tts`                                                              |
    | `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`)                             | `tts.providers.<provider>`                                                   |
    | `messages.tts.provider: "edge"` / `messages.tts.providers.edge`                                  | `tts.provider: "microsoft"` / `tts.providers.microsoft`                    |
    | `tools.exec.security` + `tools.exec.ask`                                                         | `tools.exec.mode`                                                            |
    | `session.idleMinutes`                                                                            | `session.reset.idleMinutes`                                                  |
    | açık kanal bloklarına sahip `messages.responsePrefix`                                           | yapılandırılmış kanal/hesabın `responsePrefix` alanına kopyalandı; örtük/özel kanallar için genel geri dönüş korundu |
    | `web.enabled`                                                                                    | `channels.whatsapp.enabled`                                                  |
    | `meta.lastTouchedAt`, kanca kurulumları, cron deposu, paketle birlikte gelen keşif, genel TTS tercihleri yolu            | paylaşılan SQLite durumu                                                       |
    | TTS konuşmacı alanları `voice`/`voiceName`/`voiceId`                                                 | `speakerVoice`/`speakerVoiceId`                                              |
    | `channels.<id>.tts.<provider>` / `channels.<id>.accounts.<accountId>.tts.<provider>` (Discord dışındaki tüm kanallar)                                          | `...tts.providers.<provider>`                                                |
    | `channels.<id>.voice.tts.<provider>` / `channels.<id>.accounts.<accountId>.voice.tts.<provider>` (Discord dahil tüm kanallar)                          | `...voice.tts.providers.<provider>`                                          |
    | `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`)     | `plugins.entries.voice-call.config.tts.providers.<provider>`                |
    | `plugins.entries.voice-call.config.tts.provider: "edge"` / `...tts.providers.edge`                | `provider: "microsoft"` / `...tts.providers.microsoft`                      |
    | `plugins.entries.voice-call.config.provider: "log"`                                              | `"mock"`                                                                      |
    | `plugins.entries.voice-call.config.twilio.from`                                                  | `plugins.entries.voice-call.config.fromNumber`                              |
    | `plugins.entries.voice-call.config.streaming.sttProvider`                                        | `plugins.entries.voice-call.config.streaming.provider`                      |
    | `plugins.entries.voice-call.config.streaming.openaiApiKey`/`sttModel`/`silenceDurationMs`/`vadThreshold` | `plugins.entries.voice-call.config.streaming.providers.openai.*`             |
    | `models.providers.*.api: "openai"`                                                               | `"openai-completions"` (Gateway başlatılırken ayrıca `api` değeri gelecekteki/bilinmeyen bir enum değeri olan sağlayıcılar, kapalı biçimde başarısız olmak yerine atlanır) |
    | `browser.ssrfPolicy.allowPrivateNetwork`                                                         | `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`                          |
    | `browser.profiles.*.driver: "extension"`                                                         | `"existing-session"`                                                          |
    | `browser.relayBindHost`                                                                          | kaldırıldı (eski Chrome uzantısı aktarma ayarı)                             |
    | `mcp.servers.*.type` (CLI'ya özgü takma adlar)                                                        | `mcp.servers.*.transport`                                                    |
    | `mcp.servers.*.disabled`                                                                         | tersi `mcp.servers.*.enabled`                                              |
    | MCP zaman aşımı takma adları `connectTimeout`/`connect_timeout`/`timeout`                                 | `connectionTimeoutMs`/`requestTimeoutMs`                                    |
    | alt çizgili MCP sunucu alanları                                                                     | camelCase MCP sunucu alanları                                                   |
    | `tools.media.image/audio/video.models`                                                           | yetenek etiketli `tools.media.models`                                        |
    | `tools.media.asyncCompletion`                                                                    | kaldırıldı                                                                       |
    | `tools.message.allowCrossContextSend`                                                            | `tools.message.crossContext`                                                  |
    | medya modeli `deepgram` seçenekleri                                                                   | `providerOptions.deepgram`                                                    |
    | `talk.realtime.voice`, Discord gerçek zamanlı `voice`                                                 | `speakerVoice`                                                                |
    | `agents.defaults.pdfMaxBytesMb`                                                                  | `agents.defaults.pdfMaxMb`                                                    |
    | `tools.exec.timeoutSec`                                                                          | `tools.exec.timeoutSeconds`                                                   |
    | `browser.ssrfPolicy.hostnameAllowlist`                                                           | joker karakterlerini algılayan `browser.ssrfPolicy.allowedHostnames`                          |
    | korumalı alan tarayıcısı `enableNoVnc`                                                                    | `noVncEnabled`                                                                |
    | kök `media`                                                                                     | `attachments`                                                                |
    | kanal/hesap `heartbeat` görünürlük blokları                                                   | `heartbeatVisibility`                                                         |
    | `channels.slack.identity`                                                                        | `channels.slack.postAs`                                                       |
    | kök `audit`                                                                                     | `logging.audit`                                                               |
    | `gateway.nodes.skills.enabled`                                                                   | `gateway.nodes.allowSkills`                                                   |
    | `gateway.nodes.allowCommands`/`denyCommands`                                                    | `gateway.nodes.commands.allow`/`deny`                                         |
    | oluşturma modeli varsayılanları                                                                       | `agents.defaults.mediaModels.{image,video,music}`                              |
    | kullanımdan kaldırılmış nihai düzen ayarlama düğmeleri                                                               | yerleşik varsayılan davranış                                                     |
    | `channels.whatsapp.messagePrefix` ve eski `messages.messagePrefix`                            | `channels.whatsapp.responsePrefix`                                            |
    | `channels.whatsapp.ackReaction`                                                                  | genel `messages.ackReaction` ve çevrilebilir olduğu yerlerde `ackReactionScope`        |
    | `cron.failureDestination`                                                                        | `cron.failureAlert` üzerindeki hedef alanları                                     |
    | `gateway.controlUi.chatMessageMaxWidth`, yalnızca sunuma yönelik `ui.prefs` anahtarları                       | kaldırıldı (metin ölçeği, sohbet genişliği ve canlı kenar çubuğu etkinliği tarayıcıya özeldir) |
    | `agents.list`                                                                                    | anahtarlı `agents.entries`                                                        |
    | üst düzey `defaultModel`                                                                         | `agents.defaults.model`                                                      |
    | `messages.messagePrefix`                                                                         | `channels.whatsapp.responsePrefix`                                            |
    | `session.maintenance.pruneDays`, `session.resetByType.dm`                                        | `session.maintenance.pruneAfter`, `session.resetByType.direct`               |
    | üst düzey `tui`                                                                                  | kaldırıldı (TUI alt bilgisi kompakt varsayılanı kullanır)                            |
    | `plugins.entries.codex.config.codexDynamicToolsProfile`                                          | kaldırıldı (Codex uygulama sunucusu, Codex'e özgü çalışma alanı araçlarını her zaman yerel tutar) |
    | `commands.modelsWrite`                                                                           | kaldırıldı (`/models add` kullanımdan kaldırılmıştır)                                       |
    | `agents.defaults/list[].silentReplyRewrite`, `surfaces.*.silentReplyRewrite`                     | kaldırıldı (tam `NO_REPLY` artık görünür geri dönüş metni olarak yeniden yazılmaz)  |
    | `agents.defaults/list[].systemPromptOverride`                                                    | kaldırıldı (oluşturulan sistem isteminin sahibi OpenClaw'dur)                        |
    | `agents.defaults/list[].embeddedPi`                                                              | `embeddedAgent`                                                              |
    | `agents.defaults/list[].sandbox.perSession`                                                      | `sandbox.scope`                                                              |
    | `agents.defaults.llm`                                                                             | kaldırıldı (yavaş model/sağlayıcı zaman aşımları için `models.providers.<id>.timeoutSeconds` kullanın; aracı/çalıştırma zaman aşımı tavanının altında tutulur) |
    | üst düzey `memorySearch`, `agents.defaults.memorySearch`                                         | `memory.search`                                                             |
    | `agents.entries.*.memorySearch`                                                                     | `agents.entries.*.memory.search`                                               |
    | `memorySearch.provider: "auto"`                                                                  | `"openai"`                                                                    |
    | `memorySearch.store.path` (herhangi bir düzey)                                                            | kaldırıldı (bellek dizinleri her bir aracı veritabanında bulunur)                       |
    | üst düzey `heartbeat`                                                                            | `agents.defaults.heartbeat` / `channels.defaults.heartbeat`                 |
    | `plugins.openai-codex` politika kimlikleri                                                                | `plugins.openai`                                                             |
    | `tools.web.x_search.apiKey`                                                                      | `plugins.entries.xai.config.webSearch.apiKey`                               |
    | `session.maintenance.rotateBytes`, `session.parentForkMaxTokens`                                 | kaldırıldı (kullanımdan kaldırılmıştı)                                                        |
    | 2026.7 sürümünde kullanımdan kaldırılan çalışma zamanı ve kanal ayarları                                               | kaldırıldı (yerleşik üretim varsayılanları geçerlidir)                               |

    <Note>
      Yukarıdaki `plugins.entries.voice-call.config.*` satırları, her yapılandırma yüklemesinde
      `openclaw
      doctor` tarafından değil, Voice Call plugininin kendisi tarafından normalleştirilir. Plugin ayrıca `openclaw
      doctor --fix` konumunu işaret eden bir başlangıç uyarısı kaydeder, ancak doctor şu anda bu anahtarlar için
      `openclaw.json` öğesini yeniden yazmaz; değişikliği çalışma zamanında uygulayan,
      pluginin kendi normalleştirmesidir.
    </Note>

    Birden çok hesaba sahip kanallar için hesap varsayılanı kılavuzu:

    - İki veya daha fazla `channels.<channel>.accounts` girdisi, `channels.<channel>.defaultAccount` ya da `accounts.default` olmadan yapılandırılırsa doctor, yedek yönlendirmenin beklenmeyen bir hesabı seçebileceği konusunda uyarır.
    - `channels.<channel>.defaultAccount` bilinmeyen bir hesap kimliğine ayarlanmışsa doctor uyarır ve yapılandırılmış hesap kimliklerini listeler.

  </Accordion>
  <Accordion title="2b. OpenCode sağlayıcı geçersiz kılmaları">
    `models.providers.opencode`, `opencode-zen` veya `opencode-go` öğesini elle eklediyseniz bu, `openclaw/plugin-sdk/llm` içindeki yerleşik OpenCode kataloğunu geçersiz kılar. Bu durum, modelleri yanlış API'ye yönelmeye zorlayabilir veya maliyetleri sıfırlayabilir. Doctor, geçersiz kılmayı kaldırıp model başına API yönlendirmesini ve maliyetleri geri yükleyebilmeniz için uyarır.
  </Accordion>
  <Accordion title="2c. Tarayıcı geçişi ve Chrome MCP hazırlığı">
    Tarayıcı yapılandırmanız hâlâ kaldırılmış Chrome uzantısı yolunu gösteriyorsa doctor, bunu güncel ana makine yerelindeki Chrome MCP bağlanma modeline normalleştirir (`browser.profiles.*.driver: "extension"` → `"existing-session"`; `browser.relayBindHost` kaldırılır).

    Doctor ayrıca `defaultProfile: "user"` veya yapılandırılmış bir `existing-session` profili kullandığınızda ana makine yerelindeki Chrome MCP yolunu denetler:

    - varsayılan otomatik bağlantı profilleri için Google Chrome'un aynı ana makinede yüklü olup olmadığını denetler
    - algılanan Chrome sürümünü denetler ve Chrome 144'ten düşük olduğunda uyarır
    - tarayıcı inceleme sayfasında uzaktan hata ayıklamayı etkinleştirmenizi hatırlatır (örneğin `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging` veya `edge://inspect/#remote-debugging`)

    Doctor, Chrome tarafındaki ayarı sizin yerinize etkinleştiremez. Ana makine yerelindeki Chrome MCP hâlâ gateway/node ana makinesinde yerel olarak çalışan, uzaktan hata ayıklaması etkinleştirilmiş ve ilk bağlanma onayı istemi tarayıcıda kabul edilmiş Chromium tabanlı 144+ bir tarayıcı gerektirir.

    Buradaki hazırlık yalnızca yerel bağlanma ön koşullarını kapsar. Mevcut oturum, geçerli Chrome MCP rota sınırlarını korur; `responsebody`, PDF dışa aktarma, indirme yakalama ve toplu eylemler gibi gelişmiş rotalar hâlâ yönetilen bir tarayıcı veya ham CDP profili gerektirir. Bu denetim, ham CDP kullanmaya devam eden Docker, sandbox, uzak tarayıcı veya diğer başsız akışlar için geçerli değildir.

  </Accordion>
  <Accordion title="2d. OAuth TLS ön koşulları">
    Bir OpenAI Codex OAuth profili yapılandırıldığında doctor, yerel Node/OpenSSL TLS yığınının sertifika zincirini doğrulayabildiğini kontrol etmek için OpenAI yetkilendirme uç noktasını yoklar. Yoklama bir sertifika hatasıyla başarısız olursa (örneğin `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, süresi dolmuş sertifika veya kendinden imzalı sertifika), doctor platforma özgü düzeltme kılavuzu yazdırır. Homebrew Node kullanılan macOS'ta düzeltme genellikle `brew postinstall ca-certificates` şeklindedir. `--deep` ile Gateway sağlıklı olsa bile yoklama çalışır.
  </Accordion>
  <Accordion title="2e. Codex OAuth sağlayıcı geçersiz kılmaları">
    Daha önce `models.providers.openai-codex` altına eski OpenAI taşıma ayarları eklediyseniz bunlar, yerleşik Codex OAuth sağlayıcı yolunu gölgeleyebilir. Doctor, eski taşıma geçersiz kılmasını kaldırıp veya yeniden yazıp güncel yönlendirme davranışını geri yükleyebilmeniz için Codex OAuth ile birlikte bu eski taşıma ayarlarını gördüğünde uyarır. Özel proxy'ler ve yalnızca üstbilgi geçersiz kılmaları desteklenmeye devam eder ve bu uyarıyı tetiklemez, ancak kullanıcı tarafından oluşturulan bu istek rotaları örtük Codex seçimine uygun değildir.
  </Accordion>
  <Accordion title="2f. Codex rota onarımı">
    Doctor, eski `openai-codex/*` model başvurularını denetler. Yerel Codex çalıştırma sistemi yönlendirmesi, standart `openai/*` model başvurularını kullanır ancak yalnızca önek Codex'i hiçbir zaman seçmez. Çalışma zamanı ilkesi ayarlanmamışken veya `auto` iken, yalnızca kullanıcı tarafından oluşturulmuş istek geçersiz kılması bulunmayan, tam olarak eşleşen resmî HTTPS Platform Responses veya ChatGPT Responses rotası uygundur. Bkz. [OpenAI örtük ajan çalışma zamanı](/tr/providers/openai#implicit-agent-runtime).

    `--fix` / `--repair` modunda doctor; birincil modeller, yedekler, görüntü/video oluşturma modelleri, heartbeat/alt ajan/compaction geçersiz kılmaları, hook'lar, kanal modeli geçersiz kılmaları ve kalıcı hâle getirilmiş eski oturum rota durumu dâhil olmak üzere etkilenen varsayılan ajan ve ajan başına başvuruları yeniden yazar:

    - `openai-codex/gpt-*`, `openai/gpt-*` olur.
    - Codex amacı, onarılan ajan modeli başvuruları için sağlayıcı/model kapsamlı `agentRuntime.id: "codex"` girdilerine taşınır.
    - Çalışma zamanı seçimi sağlayıcı/model kapsamında olduğundan eski tüm ajan çalışma zamanı yapılandırması ve kalıcı hâle getirilmiş oturum çalışma zamanı sabitlemeleri kaldırılır.
    - Onarılan eski model başvurusu, eski kimlik doğrulama yolunu korumak için Codex yönlendirmesine ihtiyaç duymadığı sürece mevcut sağlayıcı/model çalışma zamanı ilkesi korunur.
    - Mevcut model yedek listeleri, eski girdileri yeniden yazılarak korunur; kopyalanan model başına ayarlar eski anahtardan standart `openai/*` anahtarına taşınır.
    - Kalıcı hâle getirilmiş oturum `modelProvider`/`providerOverride`, `model`/`modelOverride`, yedek bildirimleri ve kimlik doğrulama profili sabitlemeleri, keşfedilen tüm ajan oturumu depolarında onarılır.
    - Doctor, eski `agentRuntime.id: "codex-cli"` sabitlemelerini (ayrı bir eski çalışma zamanı kimliği) `agents.defaults`, `agents.entries.*` ve `models.providers.*` model girdilerinin tamamında ayrıca `"codex"` olarak onarır.
    - `/codex ...`, "sohbetten yerel bir Codex konuşmasını denetle veya bağla" anlamına gelir.
    - `/acp ...` veya `runtime: "acp"`, "harici ACP/acpx bağdaştırıcısını kullan" anlamına gelir.

  </Accordion>
  <Accordion title="2g. Oturum rotası temizliği">
    Doctor ayrıca yapılandırılmış modelleri veya çalışma zamanını Codex gibi bir pluginin sahip olduğu rotadan taşımanızın ardından kalan ve otomatik oluşturulmuş eski rota durumu için keşfedilen ajan oturumu depolarını tarar.

    `openclaw doctor --fix`; sahip olan rota artık yapılandırılmadığında `modelOverrideSource: "auto"` model sabitlemeleri, çalışma zamanı modeli meta verileri, sabitlenmiş çalıştırma sistemi kimlikleri, CLI oturum bağlamaları ve otomatik kimlik doğrulama profili geçersiz kılmaları gibi otomatik oluşturulmuş eski durumu temizleyebilir. Açık kullanıcı veya eski oturum modeli seçimleri, elle incelenmek üzere bildirilir ve değiştirilmeden bırakılır; bu rota artık istenmiyorsa bunları `/model ...`, `/new` ile değiştirin veya oturumu sıfırlayın.

  </Accordion>
  <Accordion title="3. Eski durum geçişleri (disk düzeni)">
    Doctor, eski disk düzenlerini güncel yapıya geçirebilir:

    - Oturum deposu + dökümler: `~/.openclaw/sessions/` konumundan `~/.openclaw/agents/<agentId>/sessions/` konumuna
    - Ajan dizini: `~/.openclaw/agent/` konumundan `~/.openclaw/agents/<agentId>/agent/` konumuna
    - WhatsApp kimlik doğrulama durumu (Baileys): eski `~/.openclaw/credentials/*.json` konumundan (`oauth.json` hariç) `~/.openclaw/credentials/whatsapp/<accountId>/...` konumuna (varsayılan hesap kimliği: `default`)
    - İmzalı cihaz kimliği: `~/.openclaw/identity/device.json` konumundan `state/openclaw.sqlite` içindeki `primary` `device_identities` satırına; ayrı cihaz kimlik doğrulama dosyasına dokunulmaz

    Bu geçişler mümkün olan en iyi çabayla ve bir kez uygulanmış olsa bile aynı sonucu verecek şekilde çalışır; doctor, herhangi bir eski klasörü yedek olarak yerinde bıraktığında uyarı verir. Gateway/CLI ayrıca başlangıçta eski oturumları ve ajan dizinini otomatik olarak geçirir; böylece geçmiş/kimlik doğrulama/modeller, elle doctor çalıştırılmadan ajan başına yola taşınır. WhatsApp kimlik doğrulaması kasıtlı olarak yalnızca `openclaw doctor` aracılığıyla geçirilir. Talk sağlayıcısı/sağlayıcı eşlemesi normalleştirmesi yapısal eşitliğe göre karşılaştırma yaptığından, yalnızca anahtar sırasından kaynaklanan farklar artık tekrarlanan ve etkisiz `doctor --fix` değişikliklerini tetiklemez.

  </Accordion>
  <Accordion title="3a. Eski plugin manifest geçişleri">
    Doctor, kullanımdan kaldırılmış üst düzey yetenek anahtarları (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders`) için yüklü tüm plugin manifestlerini tarar. Bulunduğunda bunları `contracts` nesnesine taşımayı ve manifest dosyasını yerinde yeniden yazmayı teklif eder. Bu geçiş bir kez uygulanmış olsa bile aynı sonucu verir; `contracts` zaten aynı değerlere sahipse eski anahtar, veriler yinelenmeden kaldırılır.
  </Accordion>
  <Accordion title="3b. Eski cron deposu geçişleri">
    Doctor ayrıca standart satırları SQLite'a aktarmadan önce eski cron işi deposunda (`~/.openclaw/cron/jobs.json`) eski iş şekillerini denetler.

    Güncel cron temizlemeleri şunları içerir:

    - `jobId` → `id`
    - `schedule.cron` → `schedule.expr`
    - üst düzey yük alanları (`message`, `model`, `thinking`, ...) → `payload`
    - üst düzey teslim alanları (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
    - yük `provider` teslim takma adları → açık `delivery.channel`
    - eski `notify: true` webhook yedek işleri → geçerliyse kullanımdan kaldırılan ham `cron.webhook` değerinden açık webhook teslimi; duyuru işleri sohbet teslimlerini korur ve `delivery.completionDestination` değerini alır. Ardından doctor eski yapılandırma anahtarını kaldırır. Kullanılabilir bir eski webhook yoksa çalışma zamanı teslimi bunu hiçbir zaman okumadığından, etkisiz üst düzey `notify` işareti hedefi olmayan işlerden kaldırılır (duyuru dâhil mevcut teslim korunur).

    Gateway ayrıca yükleme sırasında hatalı biçimlendirilmiş cron satırlarını temizler; böylece geçerli işler çalışmaya devam eder. Ham hatalı satırlar `jobs.json` içinden kaldırılmadan önce etkin deponun yanındaki `jobs-quarantine.json` konumuna kopyalanır; doctor, elle inceleyebilmeniz veya onarabilmeniz için karantinaya alınan satırları bildirir.

    Gateway başlangıcı, çalışma zamanı izdüşümünü normalleştirir ve üst düzey `notify` işaretini yok sayar ancak kalıcı cron durumunu doctor onarımı için bırakır. Doctor, geçiş hedefi olmayan işler için etkisiz işaretleri kaldırır (`delivery.mode` yok/eksik, kullanılamayan eski webhook hedefi veya mevcut duyuru/sohbet teslimi) ve mevcut teslime dokunmaz; böylece tekrarlanan `doctor --fix` çalıştırmaları artık aynı iş için yeniden uyarmaz.

    Linux'ta doctor, kullanıcının crontab'ı hâlâ eski `~/.openclaw/bin/ensure-whatsapp.sh` öğesini çağırdığında da uyarır. Ana makineye özgü bu yerel betik, güncel OpenClaw tarafından sürdürülmez ve cron, systemd kullanıcı veri yoluna erişemediğinde `~/.openclaw/logs/whatsapp-health.log` konumuna hatalı `Gateway inactive` iletileri yazabilir. Eski crontab girdisini `crontab -e` ile kaldırın; güncel sağlık denetimleri için `openclaw channels status --probe`, `openclaw doctor` ve `openclaw gateway status` kullanın.

  </Accordion>
  <Accordion title="3c. Oturum kilidi temizleme">
    Doctor, bir oturumun olağan dışı biçimde sonlanmasıyla geride kalan eski yazma kilidi dosyalarını bulmak için her ajan oturumu dizinini tarar. Bulunan her kilit dosyası için şunları bildirir: yol, PID, PID'nin hâlâ çalışıp çalışmadığı, kilidin yaşı ve eski sayılıp sayılmadığı (ölü PID, hatalı biçimlendirilmiş sahip meta verileri, 30 dakikadan eski olma veya OpenClaw dışı bir işleme ait olduğu kanıtlanmış çalışan bir PID). `--fix` / `--repair` modunda ölü, sahipsiz, yeniden kullanılmış, hatalı biçimlendirilmiş ve eski ya da OpenClaw dışı sahipleri olan kilitleri otomatik olarak kaldırır. Hâlâ çalışan bir OpenClaw işleminin sahip olduğu eski kilitler bildirilir ancak Doctor'ın etkin bir transkript yazıcısını kesintiye uğratmaması için yerinde bırakılır.
  </Accordion>
  <Accordion title="3d. Oturum transkripti dalı onarımı">
    Doctor, 2026.4.24 istem transkripti yeniden yazma hatasının oluşturduğu yinelenen dal yapısını bulmak için ajan oturumu JSONL dosyalarını tarar: OpenClaw iç çalışma zamanı bağlamını içeren terk edilmiş bir kullanıcı sırası ve aynı görünür kullanıcı istemini içeren etkin bir kardeş dal. `--fix` / `--repair` modunda Doctor, etkilenen her dosyayı özgün dosyanın yanında yedekler ve Gateway geçmişi ile bellek okuyucularının artık yinelenen sıralar görmemesi için transkripti etkin dala göre yeniden yazar.
  </Accordion>
  <Accordion title="4. Durum bütünlüğü denetimleri (oturum kalıcılığı, yönlendirme ve güvenlik)">
    Durum dizini, operasyonel beyin sapıdır. Kaybolursa başka bir yerde yedekleriniz olmadığı sürece oturumları, kimlik bilgilerini, günlükleri ve yapılandırmayı kaybedersiniz.

    Doctor şunları denetler:

    - **Durum dizini eksik**: yıkıcı durum kaybı hakkında uyarır, dizini yeniden oluşturmanızı ister ve eksik verileri kurtaramayacağını hatırlatır.
    - **Durum dizini izinleri**: yazılabilirliği doğrular; izinleri onarmayı önerir (ve sahip/grup uyuşmazlığı algılandığında bir `chown` ipucu verir).
    - **macOS bulutla eşitlenen durum dizini**: durum iCloud Drive (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) veya `~/Library/CloudStorage/...` altında çözümlendiğinde uyarır; çünkü eşitleme destekli yollar daha yavaş G/Ç'ye ve kilit/eşitleme yarışlarına neden olabilir.
    - **Linux SD veya eMMC durum dizini**: durum bir `mmcblk*` bağlama kaynağına çözümlendiğinde uyarır; çünkü SD/eMMC destekli rastgele G/Ç daha yavaş olabilir ve oturum ile kimlik bilgisi yazımları altında daha hızlı yıpranabilir.
    - **Linux geçici durum dizini**: durum `tmpfs` veya `ramfs` olarak çözümlendiğinde uyarır; çünkü oturumlar, kimlik bilgileri, yapılandırma ve SQLite durumu (WAL/günlük yan dosyalarıyla birlikte) yeniden başlatmada kaybolur. Docker `overlay` bağlamaları, yazılabilir katmanları kapsayıcı varlığını sürdürdüğü sürece ana makine yeniden başlatmaları boyunca kalıcı olduğundan bilerek işaretlenmez.
    - **Oturum dizinleri eksik**: geçmişi kalıcılaştırmak ve `ENOENT` çökmelerini önlemek için `sessions/` ile oturum deposu dizini gereklidir.
    - **Transkript uyuşmazlığı**: son oturum girdilerinin transkript dosyaları eksik olduğunda uyarır.
    - **Ana oturum "1 satırlı JSONL"**: ana transkript yalnızca bir satır içerdiğinde işaretler (geçmiş birikmiyordur).
    - **Birden çok durum dizini**: ana dizinler arasında birden çok `~/.openclaw` klasörü bulunduğunda veya `OPENCLAW_STATE_DIR` başka bir yeri gösterdiğinde uyarır (geçmiş kurulumlar arasında bölünebilir).
    - **Uzak mod hatırlatıcısı**: `gateway.mode=remote` ise Doctor, kendisini uzak ana makinede çalıştırmanızı hatırlatır (durum orada bulunur).
    - **Yapılandırma dosyası izinleri**: `~/.openclaw/openclaw.json` grup/dünya tarafından okunabiliyorsa uyarır ve izinleri `600` olarak sıkılaştırmayı önerir.

  </Accordion>
  <Accordion title="5. Model kimlik doğrulama sağlığı (OAuth süre sonu)">
    Doctor, kimlik doğrulama deposundaki OAuth profillerini inceler, tokenların süresi dolmak üzere olduğunda veya dolduğunda uyarır ve güvenli olduğunda bunları yenileyebilir. Anthropic OAuth/token profili eskiyse bir Anthropic API anahtarı veya Anthropic kurulum tokenı yolunu önerir. Yenileme istemleri yalnızca etkileşimli (TTY) çalıştırmada görünür; `--non-interactive` yenileme girişimlerini atlar.

    Bir OAuth yenilemesi kalıcı olarak başarısız olduğunda (örneğin `refresh_token_reused`, `invalid_grant` veya bir sağlayıcının yeniden oturum açmanızı istemesi), Doctor yeniden kimlik doğrulaması gerektiğini bildirir ve çalıştırılacak tam `openclaw models auth login --provider ...` komutunu yazdırır.

    Doctor ayrıca kısa bekleme süreleri (hız sınırları/zaman aşımları/kimlik doğrulama hataları) veya daha uzun süreli devre dışı bırakmalar (faturalandırma/kredi hataları) nedeniyle geçici olarak kullanılamayan kimlik doğrulama profillerini bildirir.

    Tokenları macOS Keychain'de bulunan eski Codex OAuth profilleri (dosya tabanlı yan dosya düzeninden önceki eski ilk katılım) yalnızca Doctor tarafından onarılır. Keychain destekli eski tokenları satır içi olarak `auth-profiles.json` içine taşımak için etkileşimli bir terminalden `openclaw doctor --fix` komutunu bir kez çalıştırın; bundan sonra gömülü sıralar (Telegram, cron, alt ajan gönderimi) bunları standart OpenAI OAuth profilleri olarak çözümler.

  </Accordion>
  <Accordion title="6. Hook model doğrulaması">
    `hooks.gmail.model` ayarlanmışsa Doctor, model referansını kataloğa ve izin listesine göre doğrular; çözümlenmeyecek veya izin verilmeyen durumlarda uyarır.
  </Accordion>
  <Accordion title="7. Korumalı alan görüntüsü onarımı">
    Korumalı alan etkinleştirildiğinde Doctor, Docker görüntülerini denetler ve geçerli görüntü eksikse oluşturmayı veya eski adlara geçmeyi önerir.
  </Accordion>
  <Accordion title="7b. Plugin kurulum temizliği">
    Doctor, `openclaw doctor --fix` / `openclaw doctor --repair` modunda OpenClaw tarafından oluşturulmuş eski Plugin bağımlılığı hazırlama durumunu kaldırır: eski oluşturulmuş bağımlılık kökleri, eski kurulum aşaması dizinleri, önceki paketlenmiş Plugin bağımlılığı onarım kodundan kalan paket yerelindeki kalıntılar ve geçerli paketlenmiş manifesti gölgeleyebilen paketlenmiş `@openclaw/*` Pluginlerinin sahipsiz veya kurtarılmış yönetilen npm kopyaları. Doctor ayrıca ana makine `openclaw` paketini, `peerDependencies.openclaw` bildiren yönetilen npm Pluginlerine yeniden bağlar; böylece `openclaw/plugin-sdk/*` gibi paket yerelindeki çalışma zamanı içe aktarımları güncellemelerden veya npm onarımlarından sonra çözümlenmeye devam eder.

    Doctor ayrıca yapılandırma bunlara başvurduğunda ancak yerel Plugin kayıt defteri bunları bulamadığında eksik indirilebilir Pluginleri yeniden kurabilir (önemli `plugins.entries`, yapılandırılmış kanal/sağlayıcı/arama ayarları, yapılandırılmış ajan çalışma zamanları). Paket güncellemeleri sırasında Doctor, çekirdek paket değiştirilirken Plugin paketlerini yeniden kurmaktan kaçınır; yapılandırılmış bir Plugin hâlâ kurtarma gerektiriyorsa güncellemeden sonra `openclaw doctor --fix` komutunu yeniden çalıştırın. Aşağıdaki kapsayıcı görüntüsü başlatma istisnası dışında Gateway başlatma ve yapılandırma yeniden yükleme paket onarımı çalıştırmaz; Plugin kurulumları açık Doctor/kurulum/güncelleme çalışmaları olarak kalır.

    Kapsayıcılaştırılmış Gateway başlatmanın dar kapsamlı bir yükseltme istisnası vardır: `openclaw gateway run` yeni bir OpenClaw sürümünde başladığında hazır olmadan önce güvenli durum taşımalarını ve mevcut çekirdek sonrası Plugin yakınsamasını çalıştırır, ardından sürüm başına bir denetim noktası kaydeder. Bu başlatma geçişi eski paketlenmiş Plugin kayıtlarını temizleyebilir, yerel Plugin bağlantılarını onarabilir, yakınsama yolu gerektirdiğinde yapılandırılmış Plugin paketlerini yeniden kurabilir ve etkin Plugin yüklerini denetleyebilir. Başlatma güvenli biçimde onaramazsa kapsayıcıyı normal şekilde yeniden başlatmadan önce aynı görüntüyü aynı bağlı durum/yapılandırmaya karşı `openclaw doctor --fix` ile bir kez çalıştırın.

  </Accordion>
  <Accordion title="8. Gateway hizmeti taşımaları ve temizlik ipuçları">
    Doctor, eski Gateway hizmetlerini (launchd/systemd/schtasks) algılar ve bunları kaldırıp geçerli Gateway bağlantı noktasını kullanarak OpenClaw hizmetini kurmayı önerir. Ayrıca ek Gateway benzeri hizmetleri tarayabilir ve temizlik ipuçları yazdırabilir. Profil adını taşıyan OpenClaw Gateway hizmetleri birinci sınıf kabul edilir ve "ekstra" olarak işaretlenmez.

    Linux'ta kullanıcı düzeyindeki Gateway hizmeti eksik ancak sistem düzeyinde bir OpenClaw Gateway hizmeti mevcutsa Doctor, ikinci bir kullanıcı düzeyi hizmeti otomatik olarak kurmaz. `openclaw gateway status --deep` veya `openclaw doctor --deep` ile inceleyin, ardından yinelenen hizmeti kaldırın ya da bir sistem yöneticisi Gateway yaşam döngüsünün sahibiyse `OPENCLAW_SERVICE_REPAIR_POLICY=external` değerini ayarlayın.

  </Accordion>
  <Accordion title="8b. Başlangıç Matrix taşıması">
    Bir Matrix kanal hesabında bekleyen veya uygulanabilir bir eski durum taşıması bulunduğunda Doctor (`--fix` / `--repair` modunda) taşıma öncesi bir anlık görüntü oluşturur ve ardından en iyi çaba esaslı taşıma adımlarını çalıştırır: eski Matrix durum taşıması ve eski şifreli durum hazırlığı. Her iki adım da ölümcül değildir; hatalar günlüğe kaydedilir ve başlatma devam eder. Salt okunur modda (`openclaw doctor`, `--fix` olmadan) bu denetim tamamen atlanır.
  </Accordion>
  <Accordion title="8c. Cihaz eşleştirme ve kimlik doğrulama sapması">
    Doctor, normal sağlık geçişinin bir parçası olarak cihaz eşleştirme durumunu inceler ve şunları bildirir:

    - bekleyen ilk eşleştirme istekleri
    - zaten eşleştirilmiş cihazlar için bekleyen rol veya kapsam yükseltmeleri
    - cihaz kimliği hâlâ eşleşirken cihazın tanımlayıcı kimliğinin artık onaylanmış kayıtla eşleşmediği açık anahtar uyuşmazlığı onarımları
    - onaylanmış bir rol için etkin tokenı eksik olan eşleştirilmiş kayıtlar
    - kapsamları onaylanmış eşleştirme temel çizgisinin dışına sapan eşleştirilmiş tokenlar
    - geçerli makine için Gateway tarafındaki bir token döndürmesinden önce oluşturulmuş veya eski kapsam meta verileri taşıyan yerel önbelleğe alınmış cihaz tokenı girdileri

    Doctor, eşleştirme isteklerini otomatik olarak onaylamaz veya cihaz tokenlarını otomatik olarak döndürmez. Tam sonraki adımları yazdırır:

    - bekleyen istekleri `openclaw devices list` ile inceleyin
    - tam isteği `openclaw devices approve <requestId>` ile onaylayın
    - `openclaw devices rotate --device <deviceId> --role <role>` ile yeni bir token döndürün
    - eski bir kaydı `openclaw devices remove <deviceId>` ile kaldırıp yeniden onaylayın

    Bu, ilk eşleştirmeyi bekleyen rol/kapsam yükseltmelerinden ve eski token/cihaz tanımlayıcı kimliği sapmasından ayırarak yaygın "zaten eşleştirildi ancak hâlâ eşleştirme gerekli hatası alınıyor" açığını kapatır.

  </Accordion>
  <Accordion title="9. Güvenlik uyarıları">
    Doctor yalnızca izin listesi olmadan doğrudan mesajlara açık bir sağlayıcı veya tehlikeli biçimde yapılandırılmış bir politika gibi bir uyarı bulduğunda Güvenlik notu verir. Tam güvenlik envanteri için `openclaw security audit` kullanın.
  </Accordion>
  <Accordion title="10. systemd kalıcılığı (Linux)">
    Bir systemd kullanıcı hizmeti olarak çalışıyorsa Doctor, Gateway'in oturum kapatıldıktan sonra çalışmayı sürdürmesi için kalıcılığın etkinleştirilmesini sağlar.
  </Accordion>
  <Accordion title="11. Çalışma alanı durumu (Skills, Pluginler ve TaskFlow'lar)">
    Doctor, sağlıklı durum envanterini değil, varsayılan ajan için sorunları ve eylemleri yazdırır:

    - **Skills**: izin verilen ancak kullanılamayan Skill adlarını listeler; gereksinim ayrıntıları ve tam sayımlar için `openclaw skills check` kullanın.
    - **Pluginler**: yalnızca hata veren Plugin kimliklerini bildirir; yüklenen, içe aktarılan, devre dışı bırakılan ve paket Plugin envanteri için `openclaw plugins list` kullanın.
    - **Plugin uyumluluk uyarıları**: geçerli çalışma zamanıyla uyumluluk sorunları olan Pluginleri işaretler.
    - **Plugin tanılamaları**: Plugin kayıt defterinin yükleme sırasında verdiği tüm uyarıları veya hataları gösterir.
    - **TaskFlow kurtarma**: elle incelenmesi veya iptal edilmesi gereken şüpheli yönetilen TaskFlow'ları gösterir.
    - **Claude CLI**: yalnızca ikili dosya, kimlik doğrulama, profil, çalışma alanı veya proje dizini sorunlarını bildirir; sağlıklı yoklama ayrıntıları gösterilmez.

  </Accordion>
  <Accordion title="11b. Önyükleme dosyası boyutu">
    Doctor, çalışma alanı önyükleme dosyalarının (örneğin `AGENTS.md`, `CLAUDE.md` veya eklenen diğer bağlam dosyaları) yapılandırılmış karakter bütçesine yakın veya bu bütçenin üzerinde olup olmadığını denetler. Dosya başına ham ve eklenen karakter sayılarını, kırpma yüzdesini, kırpma nedenini (`max/file` veya `max/total`) ve toplam eklenen karakterlerin toplam bütçeye oranını bildirir. Dosyalar kırpıldığında veya sınıra yaklaştığında Doctor, `agents.defaults.bootstrapMaxChars` ve `agents.defaults.bootstrapTotalMaxChars` ayarlarını düzenlemeye yönelik ipuçları yazdırır.
  </Accordion>
  <Accordion title="11c. Kabuk tamamlama">
    Doctor, geçerli kabuk (zsh, bash, fish veya PowerShell) için sekmeyle tamamlamanın kurulu olup olmadığını denetler:

    - Kabuk profili yavaş bir dinamik tamamlama kalıbı kullanıyorsa (`source <(openclaw completion ...)`), doctor bunu daha hızlı olan önbelleğe alınmış dosya varyantına yükseltir.
    - Tamamlama profilde yapılandırılmış ancak önbellek dosyası eksikse doctor önbelleği otomatik olarak yeniden oluşturur.
    - Hiçbir tamamlama yapılandırılmamışsa doctor bunu yüklemeyi önerir (yalnızca etkileşimli modda; `--non-interactive` ile atlanır).

    Önbelleği elle yeniden oluşturmak için `openclaw completion --write-state` komutunu çalıştırın.

  </Accordion>
  <Accordion title="11d. Eski kanal Plugin temizliği">
    `openclaw doctor --fix` eksik bir kanal Plugin'ini kaldırdığında, bu Plugin'e başvuran bağlantısız kanal kapsamlı yapılandırmayı da kaldırır: `channels.<id>` girdileri, kanalı adlandıran Heartbeat hedefleri ve `agents.*.models["<channel>/*"]` geçersiz kılmaları. Bu, kanal çalışma zamanı artık mevcut olmadığı hâlde yapılandırmanın Gateway'den ona bağlanmasını istemeye devam ettiği Gateway önyükleme döngülerini önler.
  </Accordion>
  <Accordion title="12. Gateway kimlik doğrulama kontrolleri (yerel belirteç)">
    Doctor, yerel Gateway belirteciyle kimlik doğrulamanın hazır olup olmadığını kontrol eder.

    - Belirteç modu bir belirteç gerektiriyorsa ve hiçbir belirteç kaynağı yoksa doctor bir tane oluşturmayı önerir.
    - `gateway.auth.token` SecretRef tarafından yönetiliyorsa ancak kullanılamıyorsa doctor uyarır ve bunu düz metinle değiştirmez.
    - `openclaw doctor --generate-gateway-token`, yalnızca hiçbir belirteç SecretRef'i yapılandırılmamışsa oluşturmayı zorunlu kılar.

  </Accordion>
  <Accordion title="12b. Salt okunur, SecretRef uyumlu onarımlar">
    Bazı onarım akışlarının, çalışma zamanının hızlı başarısız olma davranışını zayıflatmadan yapılandırılmış kimlik bilgilerini incelemesi gerekir.

    - `openclaw doctor --fix`, hedefli yapılandırma onarımları için durum ailesindeki komutlarla aynı salt okunur SecretRef özet modelini kullanır.
    - Örnek: Telegram `allowFrom` / `groupAllowFrom` `@username` onarımı, kullanılabilir olduğunda yapılandırılmış bot kimlik bilgilerini kullanmaya çalışır.
    - Telegram bot belirteci SecretRef aracılığıyla yapılandırılmış ancak geçerli komut yolunda kullanılamıyorsa doctor, kimlik bilgisinin yapılandırılmış-ancak-kullanılamıyor olduğunu bildirir ve çökmek ya da belirteci yanlışlıkla eksik olarak bildirmek yerine otomatik çözümlemeyi atlar.

  </Accordion>
  <Accordion title="13. Gateway sağlık kontrolü + yeniden başlatma">
    Doctor bir sağlık kontrolü çalıştırır ve Gateway sağlıksız görünüyorsa yeniden başlatmayı önerir.
  </Accordion>
  <Accordion title="13b. Bellek araması hazırlığı">
    Doctor, yapılandırılmış bellek araması gömme sağlayıcısının varsayılan aracı için hazır olup olmadığını kontrol eder. Davranış, yapılandırılmış arka uca ve sağlayıcıya bağlıdır:

    - **QMD arka ucu**: `qmd` ikili dosyasının kullanılabilir ve başlatılabilir olup olmadığını yoklar. Değilse `npm install -g @tobilu/qmd` (veya Bun eşdeğeri) ve elle ikili dosya yolu seçeneği dâhil olmak üzere düzeltme yönergeleri yazdırır.
    - **Açık yerel sağlayıcı**: Yerel bir model dosyasını veya tanınan bir uzak/indirilebilir model URL'sini kontrol eder. Eksikse uzak bir sağlayıcıya geçilmesini önerir.
    - **Açık uzak sağlayıcı** (`openai`, `voyage` vb.): Ortamda veya kimlik doğrulama deposunda bir API anahtarının bulunduğunu doğrular. Eksikse uygulanabilir düzeltme ipuçları yazdırır.
    - **Eski otomatik sağlayıcı**: `memorySearch.provider: "auto"` değerini OpenAI olarak değerlendirir, OpenAI hazırlığını kontrol eder ve `doctor --fix` bunu `provider: "openai"` olarak yeniden yazar.

    Önbelleğe alınmış bir Gateway yoklama sonucu mevcut olduğunda (kontrol sırasında Gateway sağlıklıysa) doctor, sonucu CLI tarafından görülebilen yapılandırmayla çapraz karşılaştırır ve varsa tutarsızlıkları belirtir. Doctor varsayılan yolda yeni bir gömme ping'i başlatmaz; canlı bir sağlayıcı kontrolü istediğinizde ayrıntılı bellek durumu komutunu kullanın.

    Çalışma zamanında gömme hazırlığını doğrulamak için `openclaw memory status --deep` komutunu kullanın.

  </Accordion>
  <Accordion title="14. Kanal durumu uyarıları">
    Gateway sağlıklıysa doctor bir kanal durumu yoklaması çalıştırır ve önerilen düzeltmelerle birlikte uyarıları bildirir.
  </Accordion>
  <Accordion title="15. Gözetmen yapılandırması denetimi + onarım">
    Doctor, yüklü gözetmen yapılandırmasını (launchd/systemd/schtasks) eksik veya güncel olmayan varsayılanlar (örneğin systemd network-online bağımlılıkları ve yeniden başlatma gecikmesi) açısından kontrol eder. Bir uyumsuzluk bulduğunda güncelleme önerir ve hizmet dosyasını/görevi geçerli varsayılanlarla yeniden yazabilir.

    Notlar:

    - `openclaw doctor`, gözetmen yapılandırmasını yeniden yazmadan önce onay ister.
    - `openclaw doctor --yes`, varsayılan onarım istemlerini kabul eder.
    - `openclaw doctor --fix`, önerilen düzeltmeleri istem göstermeden uygular (`--repair` bir diğer addır).
    - `openclaw doctor --fix --force`, özel gözetmen yapılandırmalarının üzerine yazar.
    - `OPENCLAW_SERVICE_REPAIR_POLICY=external`, Gateway hizmeti yaşam döngüsü için doctor'ı salt okunur durumda tutar. Hizmet durumunu bildirmeye ve hizmet dışı onarımları çalıştırmaya devam eder ancak bu yaşam döngüsünün sahibi harici bir gözetmen olduğundan hizmet yükleme/başlatma/yeniden başlatma/önyükleme, gözetmen yapılandırması yeniden yazma ve eski hizmet temizleme işlemlerini atlar.
    - Linux'ta doctor, eşleşen systemd Gateway birimi etkinken komut/giriş noktası meta verilerini yeniden yazmaz. Ayrıca yinelenen hizmet taraması sırasında etkin olmayan, eski olmayan ek Gateway benzeri birimleri yok sayar; böylece yardımcı hizmet dosyaları temizleme gürültüsü oluşturmaz.
    - Belirteç kimlik doğrulaması bir belirteç gerektiriyorsa ve `gateway.auth.token` SecretRef tarafından yönetiliyorsa doctor hizmet yükleme/onarımı SecretRef'i doğrular ancak çözümlenmiş düz metin belirteç değerlerini gözetmen hizmet ortamı meta verilerine kalıcı olarak kaydetmez.
    - Doctor, eski LaunchAgent, systemd veya Windows Zamanlanmış Görev kurulumlarının satır içine gömdüğü, yönetilen `.env`/SecretRef destekli hizmet ortamı değerlerini algılar ve bu değerlerin gözetmen tanımı yerine çalışma zamanı kaynağından yüklenmesi için hizmet meta verilerini yeniden yazar.
    - Doctor, `gateway.port` değiştikten sonra hizmet komutunun hâlâ eski bir `--port` değerini sabitlediğini algılar ve hizmet meta verilerini geçerli bağlantı noktasına yeniden yazar.
    - Belirteç kimlik doğrulaması bir belirteç gerektiriyorsa ve yapılandırılmış belirteç SecretRef'i çözümlenemiyorsa doctor, uygulanabilir yönergelerle yükleme/onarım yolunu engeller.
    - Hem `gateway.auth.token` hem de `gateway.auth.password` yapılandırılmışsa ve `gateway.auth.mode` ayarlanmamışsa doctor, mod açıkça ayarlanana kadar yükleme/onarımı engeller.
    - Linux kullanıcı systemd birimleri için doctor'ın belirteç sapması kontrolleri, hizmet kimlik doğrulama meta verilerini karşılaştırırken hem `Environment=` hem de `EnvironmentFile=` kaynaklarını içerir.
    - Doctor hizmet onarımları, yapılandırma en son daha yeni bir sürüm tarafından yazılmışsa eski bir OpenClaw ikili dosyasından gelen Gateway hizmetini yeniden yazmayı, durdurmayı veya yeniden başlatmayı reddeder. Bkz. [Gateway sorun giderme](/tr/gateway/troubleshooting#split-brain-installs-and-newer-config-guard).
    - `openclaw gateway install --force` aracılığıyla her zaman tam yeniden yazmayı zorlayabilirsiniz.

  </Accordion>
  <Accordion title="16. Gateway çalışma zamanı + bağlantı noktası tanılaması">
    Doctor hizmet çalışma zamanını (PID, son çıkış durumu) inceler ve hizmet yüklü olmasına rağmen gerçekte çalışmıyorsa uyarır. Ayrıca Gateway bağlantı noktasında (varsayılan `18789`) bağlantı noktası çakışmalarını kontrol eder ve olası nedenleri (Gateway zaten çalışıyor, SSH tüneli) bildirir.
  </Accordion>
  <Accordion title="17. Gateway çalışma zamanı için en iyi uygulamalar">
    Doctor, Gateway hizmeti Bun veya sürüm yöneticisi tarafından yönetilen bir Node yolunda (`nvm`, `fnm`, `volta`, `asdf` vb.) çalıştığında uyarır. Bun, OpenClaw'ın `node:sqlite` durum deposunu açamaz; bu nedenle onarımlar eski Bun hizmetlerini Node'a geçirir. Sürüm yöneticisi yolları, hizmet kabuk başlatma yapılandırmanızı yüklemediğinden yükseltmelerden sonra bozulabilir. Doctor, mevcut olduğunda sistem Node kurulumuna (Homebrew/apt/choco) geçiş yapmayı önerir.

    Yeni yüklenen veya onarılan macOS LaunchAgent'ları, etkileşimli kabuk PATH'ini kopyalamak yerine standart bir sistem PATH'i (`/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin`) kullanır; böylece Homebrew tarafından yönetilen sistem ikili dosyaları kullanılabilir kalırken Volta, asdf, fnm, pnpm ve diğer sürüm yöneticisi dizinleri, Node alt süreçlerinin hangisini çözümlediğini değiştirmez. Linux hizmetleri açık ortam köklerini (`NVM_DIR`, `FNM_DIR`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `BUN_INSTALL`, `PNPM_HOME`) ve kararlı kullanıcı ikili dosya dizinlerini korumaya devam eder ancak tahmin edilen sürüm yöneticisi yedek dizinleri yalnızca bu dizinler diskte mevcutsa hizmet PATH'ine yazılır.

  </Accordion>
  <Accordion title="18. Yapılandırma yazma + sihirbaz meta verileri">
    Doctor tüm yapılandırma değişikliklerini kalıcı olarak kaydeder ve doctor çalıştırmasını kaydetmek için sihirbaz meta verilerini damgalar.
  </Accordion>
  <Accordion title="19. Çalışma alanı ipuçları (yedekleme + bellek sistemi)">
    Doctor, eksik olduğunda bir çalışma alanı bellek sistemi önerir ve çalışma alanı zaten git denetimi altında değilse bir yedekleme ipucu yazdırır.

    Çalışma alanı yapısı ve git yedeklemesi (önerilen: özel GitHub veya GitLab) hakkında eksiksiz bir kılavuz için [/concepts/agent-workspace](/tr/concepts/agent-workspace) sayfasına bakın.

  </Accordion>
</AccordionGroup>

## İlgili

- [Gateway işletim kılavuzu](/tr/gateway)
- [Gateway sorun giderme](/tr/gateway/troubleshooting)
