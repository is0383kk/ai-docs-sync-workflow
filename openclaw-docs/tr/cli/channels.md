---
read_when:
    - Kanal hesapları (Discord, Google Chat, iMessage, Matrix, Signal, Slack, Telegram, WhatsApp ve daha fazlası) eklemek veya kaldırmak istiyorsunuz
    - Kanal durumunu kontrol etmek veya kanal günlüklerini canlı olarak izlemek istiyorsunuz
    - Başarısız bir gelen kanal olayını incelemeniz veya yeniden göndermeniz gerekiyor
summary: '`openclaw channels` için CLI referansı (hesaplar, durum, teslim edilemeyen iletiler, yetenekler, çözümleme, günlükler, oturum açma/oturumu kapatma)'
title: Kanallar
x-i18n:
    generated_at: "2026-07-26T23:53:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5b7d674264af51d6fec34c8c95256129d66918b7c4515ac0f2c2bd311f2c3b
    source_path: cli/channels.md
    workflow: 16
---

# `openclaw channels`

Gateway üzerindeki sohbet kanalı hesaplarını ve bunların çalışma zamanı durumunu yönetin.

İlgili belgeler:

- Kanal kılavuzları: [Kanallar](/tr/channels)
- Gateway yapılandırması: [Yapılandırma](/tr/gateway/configuration)

## Yaygın komutlar

```bash
openclaw channels list
openclaw channels list --all
openclaw channels status
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels logs --channel all
openclaw channels dead-letters list --channel telegram --account default
```

`channels list` yalnızca sohbet kanallarını gösterir: varsayılan olarak yapılandırılmış hesaplar ve hesap başına `installed`, `configured` ve `enabled` durum etiketleri (`--json` makine çıktısı içindir). Henüz yapılandırılmış hesabı olmayan paketlenmiş kanalları ve henüz diskte bulunmayan, katalogdan yüklenebilir kanalları da göstermek için `--all` seçeneğini iletin. Sağlayıcı kimlik doğrulaması ve model kullanımı başka yerde bulunur: sağlayıcı kimlik doğrulama profilleri için `openclaw models auth list`, kullanım/kota için `openclaw status` veya `openclaw models list`.

## Durum / yetenekler / çözümleme / günlükler

- `channels status`: `--channel <name>`, `--probe`, `--timeout <ms>` (varsayılan `10000`), `--json`
- `channels capabilities`: `--channel <name>`, `--account <id>` (`--channel` gerektirir), `--target <dest>` (`--channel` gerektirir), `--timeout <ms>` (varsayılan `10000`, üst sınır `30000`), `--json`
- `channels resolve <entries...>`: `--channel <name>`, `--account <id>`, `--kind <auto|user|group>` (varsayılan `auto`), `--json`
- `channels logs`: `--channel <name|all>` (varsayılan `all`), `--lines <n>` (varsayılan `200`), `--json`

`channels status --probe` canlı yoldur: erişilebilir bir gateway üzerinde hesap başına
`probeAccount` ve isteğe bağlı `auditAccount` denetimlerini çalıştırır; böylece çıktı, aktarım
durumunun yanı sıra `works`, `probe failed`, `audit ok` veya `audit failed` gibi yoklama sonuçlarını içerebilir.
Gateway erişilemez durumdaysa `channels status`, canlı yoklama çıktısı
yerine yalnızca yapılandırmaya dayalı özetlere geri döner.

## Gelen başarısız iletiler

Yeniden deneme politikasını tüketen gelen olaylar, kuyruğun mevcut başarısız girdi saklama süresi boyunca paylaşılan durum veritabanında kalır. Bir kanal hesabını şu komutla inceleyin:

```bash
openclaw channels dead-letters list --channel telegram --account default
openclaw channels dead-letters list --channel telegram --account default --json
```

Metin görünümü olay kimliklerini, başarısızlık nedenlerini, deneme sayılarını ve başarısızlıkların üzerinden geçen süreleri gösterir. JSON çıktısı ayrıca tanılama için saklanan yükü, meta verileri, hattı ve deneme zaman damgalarını içerir.

Temel sorunu düzelttikten sonra bir olayı özgün olay kimliğiyle yeniden kuyruğa alın:

```bash
openclaw channels dead-letters resubmit <event-id> --channel telegram --account default
```

Bu komutları, kanal çalışma zamanıyla aynı paylaşılan durum veritabanına erişmeleri için Gateway ana makinesinde çalıştırın. Yeniden gönderim yükü, meta verileri ve hattı korur ancak deneme sayacını ve kuyruk yaşını sıfırlar. İlgili olayın başarısızlık işaretini atomik olarak değiştirir; bu nedenle olay beklemedeyken veya sahiplenilmişken komutu yinelemek, ikinci bir gönderim oluşturmak yerine işlemi reddeder. Çalışan kanal, bir sonraki giriş boşaltımında olayı alır. Tamamlanmış olaylar sonlandırılmış durumda kalır ve yeniden gönderilemez. Yük saklama özelliği eklenmeden önce oluşturulmuş başarısız satırlar listede görünmeye devam edebilir, ancak yükleri kullanılamadığından yeniden gönderimleri reddedilir.

`openclaw health`, kanal hesabı başına başarısız ileti sayılarını ve en eski başarısızlığın üzerinden geçen süreyi bildirir. `openclaw doctor`, etkilenen hesapları adlandırır ve inceleme komutuna yönlendirir.

Kanal soketi sağlığı sinyali olarak `openclaw sessions`, Gateway `sessions.list` veya ajan
`sessions_list` aracını kullanmayın. Bu yüzeyler sağlayıcı çalışma zamanı durumunu değil,
saklanan konuşma satırlarını bildirir. Discord sağlayıcısı yeniden başlatıldıktan sonra,
bağlı ancak sessiz bir hesap sağlıklı olabilir; buna karşın bir sonraki gelen veya giden konuşma
olayına kadar hiçbir Discord oturum satırı görünmeyebilir.

## Hesap ekleme / kaldırma

```bash
openclaw channels add --channel telegram --token <bot-token>
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels remove --channel telegram --delete
```

<Tip>
`openclaw channels add telegram --help` veya `openclaw channels add --channel telegram --help` yalnızca Telegram'ın kurulum bayraklarını gösterir. `openclaw channels add --help` yalnızca paylaşılan komut zarfını gösterir.
</Tip>

`channels remove` yalnızca yüklenmiş/yapılandırılmış kanal plugin'leri üzerinde çalışır. Katalogdan yüklenebilir kanallar için önce `channels add` kullanın. `--delete` olmadan hesabı devre dışı bırakmayı sorar ve yapılandırmasını korur; `--delete` yapılandırma girdilerini sormadan kaldırır.
Çalışma zamanı destekli kanal plugin'lerinde `channels remove`, yapılandırmayı güncellemeden önce çalışan Gateway'den seçilen hesabı durdurmasını da ister; böylece bir hesabın devre dışı bırakılması veya silinmesi eski dinleyiciyi yeniden başlatmaya kadar etkin bırakmaz.

Paylaşılan denetim zarfı yalnızca `--channel`, `--account` ve isteğe bağlı hesap görünen adı `--name` alanlarını içerir. Her modern kanal plugin'i kendi kimlik bilgilerini, aktarımını ve sağlayıcıya özgü anlamlarını yönetir. Bir kanal konumsal kimlikle veya `--channel <id>` aracılığıyla seçildikten sonra CLI, kanal çalışma zamanı kodunu yüklemeden yalnızca o kanalın seçeneklerini paketlenmiş ya da yüklenmiş plugin paketi meta verilerinden oluşturur.

`--token`, `--url` veya `--use-env` gibi ortak görünen bayraklar da modern bir sözleşme bunları işlediğinde kanal tarafından yönetilir. Seçilen bir üçüncü taraf plugin hâlâ eski paylaşılan kurulum bağdaştırıcısını kullanıyorsa çekirdek, ilgili kanal için yayımlanmış uyumluluk bayrağı kümesini yalnızca o kanalın eski `cliAddOptions` alanıyla birlikte kaydeder. İlgisiz eski alanlar diğer kanallara sızmaz ve seçilen modern kanal, bildirmediği uyumluluk bayraklarını reddeder.

Kanal tarafından yönetilen bayrak örnekleri:

| Kanal       | Bayraklar                                                                                            |
| ----------- | ---------------------------------------------------------------------------------------------------- |
| Google Chat | `--webhook-path`, `--webhook-url`, `--audience-type`, `--audience`                                   |
| iMessage    | `--cli-path`, `--db-path`, `--service`, `--region`                                                   |
| Matrix      | `--homeserver`, `--user-id`, `--access-token`, `--password`, `--device-name`, `--initial-sync-limit` |
| Nostr       | `--private-key`, `--relay-urls`                                                                      |
| Signal      | `--signal-number`, `--signal-transport`, `--cli-path`, `--http-url`, `--http-host`, `--http-port`    |
| Tlon        | `--ship`, `--url`, `--code`, `--group-channels`, `--dm-allowlist`, `--auto-discover-channels`        |
| WhatsApp    | `--auth-dir`                                                                                         |

Bayrakla yönlendirilen bir ekleme komutu sırasında kanal plugin'inin yüklenmesi gerekirse OpenClaw, etkileşimli plugin yükleme istemini açmadan kanalın varsayılan yükleme kaynağını kullanır.

Hem kılavuzlu kurulum hem de bayrakla yönlendirilen kurulum; seçilen kanalın ayrıştırıcısından, doğrulamasından, hesap çözümlemesinden, yapılandırma yazıcısından ve yazma sonrası kancalarından geçer. Desteklenmeyen bayraklar, genel bir girdi torbası üzerinden kabul edilmek yerine sahibi olan kanalın kurulum hatasıyla başarısız olur.

Doğrudan hesap, kimlik bilgisi veya kanal yapılandırma bayrağı olmadan `openclaw channels add` çalıştırıldığında etkileşimli sihirbaz istemde bulunabilir. Hem konumsal kanal kimliği hem de `--channel <id>`, yönlendirmeyi atlamadan ilgili kanalı önceden seçer:

```bash
openclaw channels add telegram
openclaw channels add --channel telegram
```

Sihirbaz şunları isteyebilir:

- seçilen kanal başına hesap kimlikleri
- bu hesaplar için isteğe bağlı görünen adlar
- `Route these channel accounts to agents now?`

Şimdi bağlamayı onaylarsanız sihirbaz, yapılandırılmış her kanal hesabının hangi ajan tarafından yönetileceğini sorar ve hesap kapsamlı yönlendirme bağlamalarını yazar.

Aynı yönlendirme kurallarını daha sonra `openclaw agents bindings`, `openclaw agents bind` ve `openclaw agents unbind` ile de yönetebilirsiniz (bkz. [ajanlar](/tr/cli/agents)).

Hâlâ tek hesaplı üst düzey ayarları kullanan bir kanala varsayılan olmayan bir hesap eklediğinizde OpenClaw, yeni hesabı yazmadan önce bu üst düzey değerleri kanalın hesap eşlemesine yükseltir. Kanalın tam olarak bir hesabı varsa veya `defaultAccount` bir hesabı gösteriyorsa yükseltme mevcut adlandırılmış hesabı yeniden kullanır; aksi takdirde değerler `channels.<channel>.accounts.default` içine yerleştirilir.

Yönlendirme davranışı tutarlı kalır:

- Mevcut yalnızca kanal kapsamlı bağlamalar (`accountId` yoksa) varsayılan hesapla eşleşmeye devam eder.
- `channels add`, etkileşimsiz modda bağlamaları otomatik olarak oluşturmaz veya yeniden yazmaz.
- Etkileşimli kurulum isteğe bağlı olarak hesap kapsamlı bağlamalar ekleyebilir.

Yapılandırmanız zaten karma durumdaysa (adlandırılmış hesaplar varken üst düzey tek hesap değerleri hâlâ ayarlıysa), hesap kapsamlı değerleri ilgili kanal için seçilen yükseltilmiş hesaba taşımak üzere `openclaw doctor --fix` çalıştırın.

## Oturum açma ve kapatma (etkileşimli)

```bash
openclaw channels login --channel whatsapp
openclaw channels logout --channel whatsapp
```

- `channels login`, `--account <id>` ve `--verbose` seçeneklerini; `channels logout` ise `--account <id>` seçeneğini destekler.
- Yalnızca bir yapılandırılmış kanal ilgili eylemi destekliyorsa `channels login` ve `logout` kanalı çıkarabilir; birden fazla kanal varsa `--channel` iletin.
- `channels logout`, erişilebilir olduğunda canlı Gateway yolunu tercih eder; böylece oturum kapatma işlemi kanal kimlik doğrulama durumunu temizlemeden önce etkin dinleyicileri durdurur. Yerel Gateway erişilebilir değilse yerel kimlik doğrulama temizliğine geri döner; `gateway.mode: "remote"` kullanıldığında ise gateway hatası komutun başarısız olmasına neden olur.
- Başarılı bir oturum açma işleminden sonra CLI, erişilebilir yerel Gateway'den hesabı başlatmasını ister; uzak modda kimlik doğrulamasını yerel olarak kaydeder ve uzak çalışma zamanının yeniden başlatılmadığını belirtir.
- `channels login` komutunu gateway ana makinesindeki bir terminalden çalıştırın. Ajan `exec` bu etkileşimli oturum açma akışını engeller; mevcut olduğunda sohbetten `whatsapp_login` gibi kanala özgü ajan oturum açma araçları kullanılmalıdır.

## Sorun giderme

- Geniş kapsamlı bir yoklama için `openclaw status --deep` çalıştırın.
- Kılavuzlu düzeltmeler için `openclaw doctor` kullanın.
- `openclaw channels status`, gateway erişilemez olduğunda yalnızca yapılandırmaya dayalı özetlere geri döner. Desteklenen bir kanal kimlik bilgisi SecretRef aracılığıyla yapılandırılmış ancak mevcut komut yolunda kullanılamıyorsa hesabı yapılandırılmamış olarak göstermek yerine, düşük işlevsellik notlarıyla yapılandırılmış olarak bildirir.

## Yetenek yoklaması

Sağlayıcı yetenek ipuçlarını (mevcut olduğunda intent'ler/kapsamlar) ve statik özellik desteğini getirin:

```bash
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
```

Notlar:

- `--channel` isteğe bağlıdır; tüm kanalları (plugin tarafından sağlanan kanallar dahil) listelemek için bunu atlayın.
- `--account` yalnızca `--channel` ile geçerlidir.
- `--target`, `channel:<id>` veya ham bir sayısal kanal kimliğini kabul eder ve yalnızca Discord için geçerlidir. Discord ses kanallarında izin denetimi; eksik `ViewChannel`, `Connect`, `Speak`, `SendMessages` ve `ReadMessageHistory` izinlerini işaretler.
- Yoklamalar sağlayıcıya özeldir: Discord bot kimliği + intent'ler ve isteğe bağlı kanal izinleri; Slack bot + kullanıcı kapsamları; Telegram bot bayrakları + webhook; Signal daemon sürümü; Microsoft Teams uygulama token'ı + Graph rolleri/kapsamları (bilindiği durumlarda açıklama eklenir). Yoklaması olmayan kanallar `Probe: unavailable` bildirir.

## Adları kimliklere çözümleme

Sağlayıcı dizinini kullanarak kanal/kullanıcı adlarını kimliklere çözümleyin:

```bash
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels resolve --channel discord "My Server/#support" "@someone"
openclaw channels resolve --channel matrix "Project Room"
```

Notlar:

- Hedef türünü zorlamak için `--kind user|group|auto` kullanın.
- Birden fazla giriş aynı adı paylaştığında çözümleme etkin eşleşmeleri tercih eder.
- `channels resolve` salt okunurdur. Seçilen bir hesap SecretRef üzerinden yapılandırılmışsa ancak bu kimlik bilgisi mevcut komut yolunda kullanılamıyorsa komut, tüm çalıştırmayı sonlandırmak yerine notlarla birlikte kısıtlı, çözümlenmemiş sonuçlar döndürür.
- `channels resolve` kanal pluginlerini yüklemez. Yüklenebilir bir katalog kanalının adlarını çözümlemeden önce `channels add --channel <name>` kullanın.

## İlgili

- [CLI referansı](/tr/cli)
- [Kanallara genel bakış](/tr/channels)
