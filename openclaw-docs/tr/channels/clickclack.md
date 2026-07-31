---
read_when:
    - OpenClaw'u bir ClickClack çalışma alanına bağlama
    - ClickClack bot kimliklerini test etme
summary: ClickClack bot belirteci kanal kurulumu ve hedef söz dizimi
title: ClickClack
x-i18n:
    generated_at: "2026-07-26T23:09:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 761538cdd7a916415719131b9ff2f40bf3e3e0eab0f7bda450250886acde8a64
    source_path: channels/clickclack.md
    workflow: 16
---

ClickClack, OpenClaw'u birinci sınıf ClickClack bot tokenleri aracılığıyla kendi kendine barındırılan bir ClickClack çalışma alanına bağlar.

Bir OpenClaw aracısının ClickClack bot kullanıcısı olarak görünmesini istediğinizde bunu kullanın. ClickClack, bağımsız hizmet botlarını ve kullanıcıya ait botları destekler; kullanıcıya ait botlar bir `owner_user_id` tutar ve yalnızca verdiğiniz token kapsamlarını alır.

## Hızlı kurulum

ClickClack'te **Workspace settings → Integrations → OpenClaw** bölümünü açın, **Setup code (recommended)** kullanarak bir bot oluşturun ve oluşturulan komutu kopyalayın:

```bash
openclaw channels add clickclack --code 'https://clickclack.example.com/#XXXX-XXXX-XXXX'
```

Ayrı ön uç ve API kaynakları ya da bir yol altında bağlanmış API için ClickClack bunun yerine tam talep uç noktasını oluşturur:

```bash
openclaw channels add clickclack --code 'https://api.example.com/services/clickclack/api/bot-setup-codes/claim#XXXX-XXXX-XXXX'
```

Kurulum kodu tek kullanımlıktır ve 10 dakika sonra geçerliliğini yitirir. OpenClaw kodu talep eder, yeni oluşturulan bot tokenini ve çalışma alanı ayarlarını alır, hesabı kaydeder, bağlantıyı doğrular ve çalışmakta olan Gateway'in hesabı algılayıp algılamadığını bildirir. Sürümlendirilmiş tam uç noktalar için OpenClaw, ClickClack tarafından döndürülen standart API tabanını, tüm yol önekleriyle birlikte doğrular ve kaydeder. Kurulum kodunun kendisi OpenClaw yapılandırmasında saklanmaz.

Kurulum kodu talepleri genel sunucular için HTTPS kullanır. `localhost` ve `127.0.0.1` gibi geri döngü adreslerindeki yerel kurulumlar için düz HTTP de desteklenir.

OpenClaw zaten çalışıyorsa ClickClack otomatik olarak bağlanır ve ikinci bir komut gerekmez. Aksi takdirde şu komutla başlatın:

```bash
openclaw gateway
```

Kodu sunucu URL'sinden ayrı olarak da iletebilirsiniz:

```bash
openclaw channels add clickclack --code XXXX-XXXX-XXXX --base-url https://clickclack.example.com
```

Yönlendirmeli kurulum için şunu çalıştırın:

```bash
openclaw onboard
```

ClickClack'i seçin, ardından istendiğinde sunucu URL'sini, bot tokenini ve çalışma alanını girin. Yönlendirmeli kurulum, kaydettikten sonra sunucuyu, tokeni ve çalışma alanını denetler; başarısız bir denetim yapılandırmayı silmez.

### Alternatif: manuel token

OpenClaw dışındaki bir istemciyi yapılandırırken veya tokeni açıkça kendiniz yönetmeniz gerektiğinde ClickClack'te **Manual token** seçeneğini belirleyin:

```bash
openclaw channels add clickclack --base-url https://clickclack.example.com --token ccb_... --workspace default
```

`workspace`, çalışma alanı kimliğini (`wsp_...`), kısa adını veya görünen adını kabul eder.
`--code`; `--token`, `--token-file` veya `--use-env` ile birlikte kullanılamaz.

### Alternatif: ortam tabanlı token

Varsayılan hesap, yapılandırmada token saklamak yerine `CLICKCLACK_BOT_TOKEN` değerini okuyabilir:

```bash
export CLICKCLACK_BOT_TOKEN="ccb_..."
openclaw channels add clickclack --base-url https://clickclack.example.com --workspace default --use-env
openclaw gateway
```

Adlandırılmış hesaplar yapılandırılmış bir token veya token dosyası kullanmalıdır; paylaşılan ortam değişkeni kasıtlı olarak varsayılan hesapla sınırlandırılmıştır.

### JSON5 başvurusu

Eşdeğer yapılandırma biçimi şöyledir:

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      defaultTo: "channel:general",
    },
  },
}
```

Bir hesap yalnızca `baseUrl`, bir token kaynağı ve `workspace` değerlerinin tümü ayarlandığında yapılandırılmış sayılır. Varsayılan hesap için token kaynağı `token`, `tokenFile` veya `CLICKCLACK_BOT_TOKEN` olabilir. `workspace`, çalışma alanı kimliğini (`wsp_...`), kısa adını veya adını kabul eder; Gateway başlangıçta bunu kimliğe çözümler.

### Hesap yapılandırma anahtarları

| Anahtar                 | Varsayılan          | Notlar                                                                                  |
| ----------------------- | ------------------- | --------------------------------------------------------------------------------------- |
| `baseUrl`               | yok (zorunlu)       | Tarayıcıya yönelik bağlantılar için kullanılan genel ClickClack URL'si.                 |
| `apiBaseUrl`            | `baseUrl`           | REST ve gerçek zamanlı WebSocket trafiği için isteğe bağlı sunucudan sunucuya uç nokta. |
| `token`                 | yok                 | Düz metin veya gizli bilgi başvurusu (`source: "env" \| "file" \| "exec"`) olarak bot tokeni. |
| `tokenFile`             | yok                 | Bot tokeni dosyasının yolu; `token` değerinden önceliklidir.                 |
| `workspace`             | yok (zorunlu)       | Çalışma alanı kimliği, kısa adı veya adı.                                                |
| `replyMode`             | `"agent"`           | `"agent"` tam aracı işlem hattını çalıştırır; `"model"` kısa ve doğrudan model tamamlamaları gönderir. |
| `defaultTo`             | `"channel:general"` | Giden yol hedef belirtmediğinde kullanılan hedef.                                       |
| `allowFrom`             | `["*"]`             | Gelen doğrudan mesajlar ve kanal mesajları için kullanıcı kimliği izin listesi.         |
| `botUserId`             | otomatik algılanır  | Başlangıçta bot tokeni kimliğinden çözümlenir.                                           |
| `agentId`               | rota varsayılanı    | Bu hesabın gelen mesajlarını tek bir aracıya sabitler.                                   |
| `toolsAllow`            | yok                 | Bu hesaptan gelen aracı yanıtları için araç izin listesi.                               |
| `model`, `systemPrompt` | yok                 | `replyMode: "model"` tamamlamaları tarafından kullanılır.                                 |
| `commandMenu`           | `true`              | Yerel komutları ClickClack yazıcı otomatik tamamlamasında yayımlar.                     |
| `reconnectMs`           | `1500`              | Gerçek zamanlı yeniden bağlanma gecikmesi (100 ile 60000 arası).                        |
| `discussions`           | devre dışı          | Oturum başına yönetilen kanal ayarları; bkz. [Oturum tartışmaları](#session-discussions). |

### Kimlik doğrulama korumalı genel ana bilgisayar adını koruma

ClickClack ve OpenClaw Gateway aynı ana bilgisayarda çalışırken ancak genel ClickClack ana bilgisayar adı Cloudflare Access gibi bir kimlik doğrulama Gateway'i tarafından korunuyorsa `apiBaseUrl` kullanın:

```json5
{
  channels: {
    clickclack: {
      baseUrl: "https://clack.openclaw.ai",
      apiBaseUrl: "http://127.0.0.1:8484",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
    },
  },
}
```

Genel ana bilgisayar adı, tarayıcı kullanıcıları için tamamen kimlik doğrulama korumalı kalabilir. OpenClaw, REST istekleri, kurulum doğrulaması ve gerçek zamanlı WebSocket için geri döngü uç noktasını kullanırken tartışma `embedUrl` ve `openUrl` bağlantıları genel `baseUrl` değerini kullanmaya devam eder. `apiBaseUrl` belirtilmezse tüm trafik `baseUrl` değerini kullanarak mevcut davranışı korur.

`plugins.allow` boş olmayan, kısıtlayıcı bir listeyse kanal kurulumunda ClickClack'in açıkça seçilmesi veya `openclaw plugins enable clickclack` komutunun çalıştırılması `clickclack` değerini bu listeye ekler. İlk katılım kurulumu aynı açık seçim davranışını kullanır. Bu yollar `plugins.deny` değerini veya genel bir `plugins.enabled: false` ayarını geçersiz kılmaz. Doğrudan `openclaw plugins install @openclaw/clickclack`, normal Plugin yükleme politikasını izler ve ClickClack'i mevcut bir izin listesine de kaydeder.

## Birden çok bot

Her hesap kendi ClickClack gerçek zamanlı bağlantısını açar ve kendi bot tokenini kullanır.

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      defaultAccount: "service",
      accounts: {
        service: {
          token: { source: "env", provider: "default", id: "CLICKCLACK_SERVICE_BOT_TOKEN" },
          workspace: "default",
          defaultTo: "channel:general",
          agentId: "service-bot",
        },
        support: {
          token: { source: "env", provider: "default", id: "CLICKCLACK_SUPPORT_BOT_TOKEN" },
          workspace: "default",
          defaultTo: "dm:usr_...",
          agentId: "support-bot",
        },
      },
    },
  },
}
```

## Oturum tartışmaları

Her OpenClaw oturumuna özel bir ClickClack kanalı vermek için bir ClickClack hesabında tartışmaları etkinleştirin. Hesap tokeni `channels:write` kapsamını içermelidir (`bot:admin` paketi bunu içerir); normal `bot:write` kurulum tokeni kanal oluşturamaz veya eşitleyemez.

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      discussions: {
        enabled: true,
        workspace: "default",
        controlUrlBase: "https://team.openclaw.ai",
        section: "Sessions",
      },
    },
  },
}
```

`discussions.workspace`, hesap düzeyindeki `workspace` ile aynı çalışma alanı kimliğini, kısa adını veya görünen adını kabul eder ve varsayılan olarak bu değeri kullanır. `section`, ClickClack kenar çubuğu bölümünü denetler ve varsayılan olarak `Sessions` değerini kullanır. `controlUrlBase` ayarlandığında yönetilen kanal, gerçek Control UI oturum rotası olan `/chat?session=<encoded-session-key>` değerine geri bağlantı verir.

Tartışmaları tam olarak bir ClickClack hesabında etkinleştirin. Gateway sağlayıcısının hesap seçicisi yoktur; bu nedenle birden çok etkin tartışma hesabı, yapılandırma sırasına göre birini seçmek yerine reddedilir.

Bir tartışma açıldığında haricen yönetiliyor olarak işaretlenmiş genel bir ClickClack kanalı oluşturulur. Plugin; oturum etiketini, kategorisini ve arşiv durumunu eşitlenmiş tutar. Bir oturumun geri yüklenmesi kanalını geri yükler; oturum kategorisinin temizlenmesi kanalı yapılandırılmış varsayılan bölüme geri taşır. Bir OpenClaw oturumunun silinmesi, geçmişinin kullanılabilir kalması için ClickClack kanalını silmek yerine arşivler. Plugin, tartışma RPC'leri kullanıldığında ve herhangi bir bağlama varken yaklaşık dakikada bir bağlamaları uzlaştırır.

Yönetilen bir kanaldaki gelen mesajlar, bağlı ana oturumla aynı aracı kimliği altında belirlenimci bir yan oturum kullanır. Yan aracıya hangi ana oturumu gözlemleyeceği bildirilir ve `sessions_history` ile `session_status` araçlarını kullanabilir (`changesSince`, artımlı denetimler için kullanışlıdır). `sessions_send` aracını yalnızca tartışmadaki kişiler ana oturuma ileti aktarmasını veya yön vermesini istediğinde kullanır. Bağlama, yönetilen sahiplik başvurusu ve yan oturum eş kimliği; sabitlenmiş ClickClack sunucusu ve kanalıyla birlikte somut OpenClaw oturum kimliğini içerir. Yeniden kullanılabilir bir oturum anahtarının sıfırlanması veya hesabın yeniden hedeflenmesi eski kanalı yerel olarak iptal eder, eski kimlik bilgisi kullanılabilir durumda kaldığında kanalı arşivler ve yan dökümünün yeniden kullanılmasına izin vermez. Arşivlenmiş, sıfırlanmış, devre dışı bırakılmış veya yeniden hedeflenmiş bir bağlama üzerinden gelen mesajlar, hesabın normal kanal yönlendirmesine geri dönmek yerine bırakılır. Serbest bırakılan bağlamalar, gecikmiş gerçek zamanlı olayların kapalı kalmasını sağlamak için kalıcı bir iptal edilmiş kanal işareti bırakır. Uzak sahiplik ClickClack sunucusu ve kanal kimliğiyle anahtarlanır; dolayısıyla yerel hesabın yeniden adlandırılması yönetilen bir kanalı sıradan bir kanala dönüştüremez.

`tools.sessions.visibility` değerini daha güvenli varsayılanı olan `tree` olarak tutun. Plugin, yalnızca her yan oturum ile bağlı ana oturumu arasında ana bilgisayar kapsamlı bir yetki ve ayrıca oturum keşfini ve oturumlar arası hedefleri engelleyen bir araç politikası kancası kurar. `sessions_history`, `session_status` ve `sessions_send` araçlarına yalnızca bağlı ana oturum için izin verir ve durum çağrısının bu oturumun modelini değiştirmesini önler. Bu araçların yine de aracının etkin araç izin listesinde bulunması gerekir. Sistem istemi yalnızca yönlendirmedir; ana bilgisayar yetkisi ve kanca, yetkilendirme sınırıdır.

ClickClack sunucusu, kanal oluşturma ve güncellemelerde yönetilen kanal alanlarını (`external_managed`,
`external_ref`, `external_url` ve `sidebar_section`) desteklemeli ve
bunları kanal yanıtlarında döndürmelidir. OpenClaw, bir bağlamayı kalıcı hâle
getirmeden önce bu sözleşmeyi doğrular. Bir oluşturma yanıtı kaybolursa sonraki
açma işlemi, başka bir kanal oluşturmak yerine sunucunun zorunlu kıldığı
`external_ref` üzerinden kanalı benimser. Bu sonuç uzlaştırılana kadar bekleyen
rezervasyon, hedef çalışma alanındaki aksi hâlde bağlanmamış olayları karantinaya
alır. Kaba uzlaştırıcı, aynı oturum hâlâ etkinse kanalı benimser veya sıfırlama
sonrasında arşivler; uzak kanal oluşturulmamışsa rezervasyonu temizler.
Bu referans; OpenClaw kurulumu başına kalıcı bir ad alanının yanı sıra oturum
anahtarının, somut oturum kimliğinin, ClickClack hedefinin ve kalıcı bağlama
neslinin karmasını içerir. Ayrı gateway'ler birbirlerinin kanallarını
benimseyemez, sıfırlanan oturumlar eski kanal geçmişini devralamaz ve bir hesap
veya çalışma alanı gidiş dönüşü önceki bir kanalı yeniden benimseyemez.
Bağlamalar ayrıca yapılandırılmış ClickClack sunucu URL'sine sabitlenir ve hesap
başka bir hedefe yönlendirilirse geçersiz kılınır. `controlUrlBase` değerinin
değiştirilmesi veya kaldırılması, sonraki uzlaştırma geçişinde yönetilen kanal
bağlantısını günceller veya temizler. `discussions.workspace` değerinin değiştirilmesi,
eski çalışma alanı kimlik bilgisi yapılandırılmış durumda kaldığında yeni çalışma
alanında bir kanal açılabilmesinden önce eski bağlamayı arşivler ve serbest
bırakır. Token, eski çalışma alanına erişemeyen çalışma alanı kapsamlı bir kimlik
bilgisiyle değiştirilmişse OpenClaw, değiştirme token'ını denemeden eski kanalı
iptal edilmiş olarak kaydeder ve bağlamayı serbest bırakır; geride kalan bu kanalı
ClickClack üzerinden arşivleyin.

Ekli ana oturum ayrıca yalnızca çekme amaçlı bir `discussion` aracı alır. Bu
araç, en son mesajları ve yakın tarihli ileti dizisi yanıtlarını mesaj başına
kaçış karakterleri uygulanmış, kaynak belirtilmiş tek bir kayıt olarak okur ve
yazma veya yaşam döngüsü yan etkisi oluşturmaz. Kanal kökü ve ileti dizisi
aramalarının sabit istek bütçeleri vardır; sonuç, bu güvenlik sınırı nedeniyle
daha eski etkin bir ileti dizisinin atlanabileceğini açıkça bildirir.

## Yanıt modları

- `replyMode: "agent"` (varsayılan), gelen mesajları oturum kaydı ve araç politikası dâhil olmak üzere normal agent işlem hattı üzerinden işler.
- `replyMode: "model"`, agent işlem hattını atlar ve doğrudan bot yanıtları için Plugin çalışma zamanının `llm.complete` özelliğini kullanır; yanıtlar isteğe bağlı olarak `model` ve `systemPrompt` ile biçimlendirilebilir. Tamamlama bütçesini seçilen sağlayıcı ve model belirler.

Model modu, tamamlamaları çözümlenen bot agent kimliğine karşı çalıştırır; bunun
için açık `plugins.entries.clickclack.llm.allowAgentIdOverride: true` güven
biti gerekir:

```json5
{
  plugins: {
    entries: {
      clickclack: {
        llm: {
          allowAgentIdOverride: true,
        },
      },
    },
  },
}
```

Yalnızca varsayılan `agent` yanıt modunu kullanıyorsanız güven bitini
kapalı tutun; bu modda gerekli değildir.

## Komut menüsü

Gateway başlatıldığında yapılandırılmış her hesap, OpenClaw'ın yerel komutlarını
ClickClack'e yayımlar. Bunlar, düzenleyici otomatik tamamlamasında botun kullanıcı
adıyla etiketlenmiş olarak görünür. Yayımlanan küme, yerel komut kataloğu boş
olduğunda eski bir menünün temizlenmesi de dâhil olmak üzere her başlatmada
tümüyle değiştirilir.

Komut menüsü eşitlemesi varsayılan olarak etkindir. Devre dışı bırakmak için bir
hesapta `commandMenu: false` ayarlayın:

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      commandMenu: false,
    },
  },
}
```

Token için `commands:write` gerekir. Güncel ClickClack `bot:write` ve
`bot:admin` paketleri bu kapsamı içerir; kapsam ayrıca tek başına da
verilebilir. Komut menüleri kullanıma sunulmadan önce oluşturulan token'lara bu
kapsamın eklenmesi veya token'ın değiştirilmesi gerekebilir.

Eşitleme azami gayretle gerçekleştirilir ve her Gateway başlatmasında bir kez
çalışır. Eksik kapsam veya ağ hatası bir uyarı günlüğe kaydeder; uç noktayı
içermeyen eski bir ClickClack sunucusu hata ayıklama düzeyinde günlüğe kaydedilir.
Bu hataların hiçbiri gerçek zamanlı başlatmayı engellemez. Menüler, agent çevrim
dışıyken kullanılabilir kalır ve bot çalışma alanından ayrıldığında kaldırılır.

Bu sürüm yalnızca yerel komut belirtimlerini yayımlar. Takma adlar ve Skills,
Plugin veya özel komut katalogları menüye eklenmez. Bir ad aynı zamanda HTTP
eğik çizgi komutu olarak kaydedilmişse ClickClack önce bu kaydı işler; diğer menü
komutları normal mesaj teslimi üzerinden devam eder.

Hizmetler arası korelasyon kanıtı için `agent` modunu kullanın.
Kanonik `msg_<ulid>` biçimindeki yetkili bir ClickClack mesaj kimliği için
kanal, deterministik OpenClaw çalıştırma kimliği `clickclack:<message-id>` değerini
türetir. Ardından her model çağrısı tanılamalarda `clickclack:<message-id>:model:<n>` olarak
görünür; bu tur ClawRouter kullandığında aynı model çağrısı kimliği
`X-Request-ID` olarak gönderilir. `model` modu normal agent
çalıştırma/oturum tanılamalarını atlar ve bu nedenle bu kanıt yolu için uygun
değildir.

Gerçek zamanlı bir olay doğrulanmış bir `payload.correlation_id` içerdiğinde kanal,
bunu yetkili mesaj getirme ve sonuçta oluşan ClickClack yanıt isteklerinde
`X-Correlation-ID` olarak taşır. Değerler ClickClack'in güvenli 128 karakterlik
kümesini (`A-Z`, `a-z`, `0-9`, `.`, `_`, `:` ve `-`) kullanır; geçersiz değerler
atlanır. Bu birleştirmeler yalnızca tanımlayıcılar içerir; mesaj gövdelerini,
istemleri, tamamlamaları, kimlik bilgilerini veya araç çıktısını hiçbir zaman
içermez.

## Kalıcı medya teslimi

Medya içeren agent yanıtları zorunlu kalıcı teslimi kullanır. OpenClaw, ilk
ClickClack yazma işleminden önce parça başına kararlı mesaj ve yükleme nonce
değerleri atar; böylece yeniden deneme, depolama kotası tüketmek veya yinelenen
iletiler yayımlamak yerine aynı yüklemeyi ve mesajı yeniden kullanır. Yeniden
başlatma sonrasında bir yükleme zaten mevcutsa OpenClaw, özgün yerel yolu veya
uzak medya URL'sini yeniden okumaz.

Bu kurtarma sözleşmesi, aşağıdakileri destekleyen bir ClickClack sunucusu
gerektirir:

- `GET /api/uploads/by-nonce`; bulunan ve eksik sonuçlarda
  `X-ClickClack-Upload-Nonce: supported` ile.
- `GET /api/messages/by-nonce`; bulunan ve eksik sonuçlarda
  `X-ClickClack-Message-Nonce: supported` ile.
- Aynı sahip kapsamlı nonce ve yükleme için eşgüçlü mesaj oluşturma ve ek ilişkilendirme.

Eski bir sunucunun genel 404 yanıtı, bir gönderimin mevcut olmadığının kanıtı
olarak değerlendirilmez. OpenClaw, yinelenen gönderim riskine girmek yerine
teslimi çözümlenmemiş durumda bırakır; medya üreten agent yanıtlarını
etkinleştirmeden önce ClickClack'i güncelleyin.

## Agent etkinlik satırları

Varsayılan olarak bir agent turu çalışırken ClickClack kanalında hiçbir şey gösterilmez; yalnızca son yanıt gönderilir. Tur devam ederken kalıcı `agent_commentary` ve `agent_tool` mesaj satırları yayımlamak için bir hesapta `agentActivity: true` ayarlayın:

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      agentActivity: true,
    },
  },
}
```

Gereksinimler ve davranış:

- **Varsayılan olarak kapalıdır.** Standart kurulumlar ve eski ClickClack sunucuları etkilenmez.
- **`agent_activity:write` token kapsamını gerektirir.** Bu kapsam `bot:write` kapsamından ayrıdır ve ondan devralınmaz; seçeneği etkinleştirmeden önce bot token'ını `--scopes bot:write,agent_activity:write` ile oluşturun (veya kapsamı mevcut bir token'a verin).
- **Azami gayretle işlev kaybı.** Token'da `agent_activity:write` yoksa veya sunucu etkinlik yazmalarını reddederse hatalar günlüğe kaydedilir ve son yanıt yine normal şekilde teslim edilir; hiçbir etkinlik satırı görünmez.
- Satırlar tur başına (`turn_id`) gruplanır, her mantıksal adım tek bir satır olacak şekilde birleştirilir ve araç satırları Discord/Slack/Telegram ile aynı ilerleme biçimlendirmesini kullanır (araç adı ve komut ayrıntısı).
- **Atıf meta verileri.** Agent tarafından yazılan gönderiler (etkinlik satırları ve son yanıt), turda fiilen kullanılan modelden (geri dönüş sonrasında kullanılan model dâhil) çözümlenen `author_model` ve `author_thinking` alanlarını taşır. Bu sütunları tanımlamayan sunucular bilinmeyen JSON alanlarını yok sayar; bunları kalıcılaştıran sunucular mesaj başına "bu satırı hangi model, hangi düşünme düzeyinde söyledi" sorusunu yanıtlayabilir.

## Hedefler

- `channel:<name-or-id>`, bir çalışma alanı kanalına gönderir. Öneksiz hedeflerin varsayılanı `channel:` olur.
- `dm:<user_id>`, bu kullanıcıyla doğrudan bir konuşma oluşturur veya mevcut konuşmayı yeniden kullanır.
- `thread:<message_id>`, bu mesajın kökünü oluşturduğu ileti dizisinde yanıt verir.

Açık giden hedefler ayrıca `clickclack:` veya `cc:` sağlayıcı
önekini taşıyabilir.

Giden medya, ClickClack'in yükleme API'sini kullanır ve ardından kalıcı yüklemeyi
oluşturulan kanal mesajına, ileti dizisi yanıtına veya doğrudan mesaja ekler.
Yerel dosyalar ve desteklenen uzak medya URL'leri, dosya başına 64 MiB sınırıyla
OpenClaw'ın normal medya erişimi politikasını izler. Kalıcı kuyruğa alınmış
gönderimler, her yükleme ve mesaj parçası için ayrı sahip kapsamlı nonce
değerleri kullanır ve ardından aynı nesnelerle ek ilişkilendirmeyi yeniden dener.
Sunucu sözleşmesi ve kurtarma davranışı için [Kalıcı medya teslimi](#durable-media-delivery)
bölümüne bakın.

Örnekler:

```bash
openclaw message send --channel clickclack --target channel:general --message "hello"
openclaw message send --channel clickclack --target dm:usr_123 --message "hello"
openclaw message send --channel clickclack --target thread:msg_123 --message "following up"
```

## İzinler

ClickClack token kapsamları ClickClack API'si tarafından zorunlu kılınır.

- `bot:read`: çalışma alanı/kanal/mesaj/ileti dizisi/doğrudan mesaj/gerçek zamanlı/profil verilerini okur.
- `bot:write`: `bot:read` ile birlikte kanal mesajları, ileti dizisi yanıtları, doğrudan mesajlar, yüklemeler ve komut menüsü yayımlama.
- `bot:admin`: `bot:write` ile birlikte kanal oluşturma.
- `commands:write`: botun komut menüsünü yayımlar. Güncel `bot:write` ve `bot:admin` paketlerine dâhildir ve tek başına verilebilir.
- `agent_activity:write`: kalıcı agent etkinlik satırları (`agent_commentary` / `agent_tool`). `bot:write` veya `bot:admin` tarafından devralınmaz; yalnızca `agentActivity: true` ayarlandığında gereklidir.

OpenClaw, normal agent sohbeti ve komut menüsü eşitlemesi için yalnızca güncel
`bot:write` kapsamına ihtiyaç duyar. [Agent etkinlik satırları](#agent-activity-rows)
etkinleştirilirken `agent_activity:write` ekleyin.

## Sorun giderme

- `ClickClack is not configured for account "<id>"`: bu hesap için `baseUrl`, `token` (örneğin `CLICKCLACK_BOT_TOKEN` aracılığıyla) ve `workspace` ayarlayın.
- `ClickClack workspace not found: <value>`: `workspace` değerini ClickClack tarafından döndürülen çalışma alanı kimliği, kısa adı veya adı olarak ayarlayın.
- Gelen yanıt yok: token'ın gerçek zamanlı okuma erişimine sahip olduğunu doğrulayın ve botun kendi mesajlarını ve diğer botlardan gelen mesajları yok saydığını unutmayın.
- Kanal gönderimleri başarısız oluyor: botun çalışma alanının üyesi olduğunu ve `bot:write` kapsamına sahip olduğunu doğrulayın.
- Komut menüsü yok: `commandMenu` değerinin `false` olmadığını, ClickClack sunucusunun `PUT /api/bots/self/commands` özelliğini desteklediğini ve token'ın `commands:write` kapsamına sahip olduğunu doğrulayın.
