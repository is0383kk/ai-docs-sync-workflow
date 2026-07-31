---
read_when:
    - Resmî Codex app-server test altyapısını kullanmak istiyorsunuz
    - Codex harness yapılandırma örneklerine ihtiyacınız var
    - Codex'e özel dağıtımların OpenClaw'a geri dönmek yerine başarısız olmasını istiyorsunuz
summary: Resmî Codex app-server test düzeneği üzerinden OpenClaw yerleşik aracı turlarını çalıştırın
title: Codex çalıştırma altyapısı
x-i18n:
    generated_at: "2026-07-26T22:52:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e016a1689af65c5520d529ce22a87bd25ee29369f7aedca77b27f943a7f21b0f
    source_path: plugins/codex-harness.md
    workflow: 16
---

Resmî `codex` plugin'i, yerleşik OpenClaw yürütme düzeneği yerine Codex
app-server üzerinden gömülü OpenAI ajan turlarını çalıştırır. Alt düzey ajan
oturumunun sahibi Codex'tir: yerel iş parçacığını sürdürme, yerel araç devamı,
yerel Compaction ve app-server yürütmesi. OpenClaw ise sohbet kanallarının,
oturum dosyalarının, model seçiminin, OpenClaw dinamik araçlarının, onayların,
medya tesliminin ve görünür transkript yansısının sahibi olmaya devam eder.

`openai/gpt-5.6-sol` gibi standart OpenAI model referanslarını kullanın. Eski
Codex GPT referanslarını yapılandırmayın; OpenAI ajan kimlik doğrulama sırasını
`auth.order.openai` altına yerleştirin. Eski Codex kimlik doğrulama profili
kimlikleri ve eski Codex kimlik doğrulama sırası girdileri
`openclaw doctor --fix` tarafından onarılır.

Sağlayıcı/model çalışma zamanı ilkesi ayarlanmamışken veya `auto` iken, yalnızca `openai/*` ön eki
bu yürütme düzeneğini hiçbir zaman seçmez. OpenAI, yalnızca istek üzerinde
kullanıcı tarafından belirlenmiş bir geçersiz kılma bulunmayan, tam olarak resmî
bir HTTPS Platform Responses veya ChatGPT Responses rotası için Codex'i örtük
olarak seçebilir. Bkz.
[OpenAI örtük ajan çalışma zamanı](/tr/providers/openai#implicit-agent-runtime).
Platform ve ChatGPT yönlendirmesi bilinmeden önce kimlik doğrulamanın sahibi Codex
olursa OpenClaw yine de her aday rotanın Codex uyumluluğunu bildirmesini
gerektirir. Yalnızca yerel kimlik doğrulama sahipliği bu rota denetimini hiçbir
zaman atlamaz.

Etkin bir OpenClaw korumalı alanı olmadığında OpenClaw, Codex app-server iş
parçacıklarını Codex yerel kod modu etkin olarak başlatır (yalnızca kod modu
varsayılan olarak kapalı kalır); böylece yerel çalışma alanı/kod yetenekleri,
app-server `item/tool/call` köprüsü üzerinden yönlendirilen OpenClaw dinamik
araçlarının yanında kullanılabilir durumda kalır. Etkin bir OpenClaw korumalı
alanı veya kısıtlı araç ilkesi, deneysel korumalı alan exec-server yolunu açıkça
etkinleştirmediğiniz sürece yerel kod modunu tamamen devre dışı bırakır.

Varsayılan `tools.exec.host: "auto"` ile ve etkin bir OpenClaw korumalı alanı yokken
Codex, eşleştirilmiş Node'larda komut çalıştırmak için `node_exec` ve `node_process` araçlarını da
alır. Yerel kabuk, Codex app-server ana makinesinde ve çalışma alanında kalır
(varsayılan stdio dağıtımı için Gateway yerelindedir); `node_exec` ada
veya kimliğe göre bir Node seçer ve OpenClaw'ın Node onay ilkesini yürürlükte
tutar. Sonlu bir çalışma zamanı izin listesi yerel Kod Modu'nu devre dışı bırakır
ve turu yürütme ortamı olmadan bırakırsa OpenClaw bunun yerine doğrudan,
korumalı alansız yürütme için ilke tarafından filtrelenmiş `exec` ve `process`
araçlarını kullanılabilir tutar.

Codex'e özgü bu özellik, farklı bir `exec` girdi biçimine sahip,
genel OpenClaw çalıştırmaları için isteğe bağlı bir QuickJS-WASI çalışma zamanı
olan [OpenClaw Kod Modu](/tr/tools/code-mode) özelliğinden ayrıdır. Daha geniş
model/sağlayıcı/çalışma zamanı ayrımı için
[Ajan çalışma zamanları](/tr/concepts/agent-runtimes) ile başlayın: `openai/gpt-5.6-sol`
model referansı, `codex` çalışma zamanı; Telegram, Discord, Slack veya
başka bir kanal ise iletişim yüzeyidir.

## Gereksinimler

- Resmî `@openclaw/codex` plugin'i kurulmuş olmalıdır. Yapılandırmanız bir izin
  listesi kullanıyorsa `codex` değerini `plugins.allow` içine ekleyin.
- `0.143.0` ile `0.145.0` arasındaki kararlı bir Codex app-server. Plugin varsayılan olarak uyumlu
  bir ikili dosyayı yönetir; bu nedenle `PATH` üzerindeki bir `codex` komutu normal
  başlatmayı etkilemez.
- `openclaw models auth login --provider openai` üzerinden Codex kimlik doğrulaması, ajanın Codex ana
  dizininde zaten bulunan bir app-server hesabı veya açıkça belirtilmiş bir Codex
  API anahtarı kimlik doğrulama profili.

Kimlik doğrulama önceliği, ortam yalıtımı, özel app-server komutları, model
keşfi ve yapılandırma alanlarının tam listesi için
[Codex yürütme düzeneği referansı](/tr/plugins/codex-harness-reference) bölümüne bakın.

## Hızlı başlangıç

Resmî plugin'i kurun, ardından Codex OAuth ile oturum açın:

```bash
openclaw plugins install @openclaw/codex
openclaw models auth login --provider openai
```

`codex` plugin'ini etkinleştirin ve bir OpenAI ajan modeli seçin:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

Yapılandırmanız `plugins.allow` kullanıyorsa `codex` değerini de
buraya ekleyin:

```json5
{
  plugins: {
    allow: ["codex"],
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Plugin yapılandırmasını değiştirdikten sonra Gateway'i yeniden başlatın. Bir
sohbetin zaten oturumu varsa bir sonraki turun yürütme düzeneğini geçerli
yapılandırmadan çözümlemesi için önce `/new` veya `/reset` çalıştırın.

## İş parçacıklarını Codex Desktop ve CLI ile paylaşma

Varsayılan `appServer.homeScope: "agent"`, her OpenClaw ajanını operatörün yerel Codex
durumundan yalıtır. Bir sahibin Codex Desktop ve Codex CLI tarafından gösterilen
aynı yerel iş parçacıklarını inceleyip yönetebilmesi için kullanıcı Codex ana
dizinini etkinleştirin:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            homeScope: "user",
          },
        },
      },
    },
  },
}
```

Kullanıcı ana dizini modu, yerel olarak yönetilen bir stdio işlemini veya
paylaşılan Unix soketi aktarımını destekler. Ayarlandığında `$CODEX_HOME`,
aksi durumda `~/.codex` kullanır; buna söz konusu ana dizinin yerel Codex
kimlik doğrulaması, yapılandırması, plugin'leri ve iş parçacığı deposu dahildir.
OpenClaw bu app-server'a bir OpenClaw kimlik doğrulama profili eklemez.

Sahip turları `codex_threads` aracını edinir: yerel iş parçacıklarını
listeleme, arama, okuma, çatallama, yeniden adlandırma, arşivleme ve geri yükleme.
Bir iş parçacığını OpenClaw'da sürdürmek için çatallayın; çatal geçerli OpenClaw
oturumuna bağlanır ve diğer yerel Codex istemcileri tarafından görünür kalır.
Arşivleme, iş parçacığının başka bir yerde kapalı olduğuna ilişkin açık onay
gerektirir. Denetim de etkinleştirildiğinde transkript alanları ve değişiklikler
eşleşen `supervision.allowRawTranscripts` veya `supervision.allowWriteControls` etkinleştirmesini gerektirir.

Aynı iş parçacığını bağımsız yönetilen stdio App Server'lar üzerinden eşzamanlı
olarak sürdürmeyin veya yazmayın. Codex, canlı yazıcıları ayrı işlemler arasında
değil, tek bir App Server içinde koordine eder. Çatallama, sıradan kullanıcı ana
dizini stdio oturumları için güvenli bir arada çalışma yoludur.

`appServer.homeScope: "user"` tek başına filo kataloğunu denetlemez. Plugin etkinken yerel
oturum keşfi etkinleştirilir; Codex'i devre dışı bırakmadan OpenClaw kenar
çubuğundan kaldırmak için `sessionCatalog.enabled: false` ayarlayın. Katalog ayrı bir denetim
bağlantısı kullanır; açık `appServer` bağlantı ayarları olmadan bu bağlantı
varsayılan olarak yönetilen kullanıcı ana dizini stdio'yu kullanırken sıradan
yürütme düzeneği ajan kapsamlı kalır. Açık `appServer` ayarlarına her iki
yol da uyar. Sıradan yürütme düzeneğinin de yerel durumu paylaşması gerektiğinde
yukarıdaki gibi `homeScope: "user"` değerini açıkça ayarlayın.

## Codex oturumlarını denetleme

Aynı `codex` plugin'i, Gateway bilgisayarındaki ve açıkça
etkinleştirilmiş eşleştirilmiş Node'lardaki arşivlenmemiş Codex oturumlarını
listeleyebilir. Depolanmış veya boşta olan Gateway yerelindeki bir oturum, kalıcı
kılınmış ve sınırlandırılmış kullanıcı ve asistan geçmişini yansıtan, modele
kilitli bir Sohbet oluşturabilir. Özel bağlaması yerel anlık görüntü, standart
dal ve sonraki turlar için denetim bağlantısını kullanırken sıradan Codex
oturumları ajan kapsamlı kalır. İlk standart başlatma, Codex'in anlık görüntü
çatalı için döndürdüğü model ve sağlayıcıyı tam olarak kullanır. Sonraki
sürdürmeler seçimi Codex'in yerel yapılandırmasına bırakır; dış OpenClaw modeli
ve geri dönüş zinciri bunu hiçbir zaman değiştirmez. Depolanmış ve boşta olan
satırlar, başka bir çalıştırıcı olmadığına ilişkin açık onayın ardından
arşivlenebilir. Etkin kaynaklardan dal oluşturulamaz ve bunlar arşivlenemez;
mevcut bir denetimli Sohbet yine de açılabilir. Eşleştirilmiş Node oturumları
yalnızca meta veri olarak kalır.

Kurulum, dallanma kuralları, eşleştirilmiş Node sınırları, meta veri gösterimi
ve sorun giderme için [Codex oturumlarını denetleme](/plugins/codex-supervision)
bölümüne bakın.

## Yapılandırma

| Gereksinim                                          | Ayar                                                                                             | Konum                              |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------ | ---------------------------------- |
| Yürütme düzeneğini etkinleştirme                    | `plugins.entries.codex.enabled: true`                                                            | OpenClaw yapılandırması            |
| Yerel Codex oturum keşfini gizleme                  | `plugins.entries.codex.config.sessionCatalog.enabled: false`                                     | Codex plugin yapılandırması        |
| İzin listesindeki plugin kurulumunu koruma          | `codex` değerini `plugins.allow` içine ekleyin                                              | OpenClaw yapılandırması            |
| Uygun OpenAI turlarının Codex'i örtük kullanmasına izin verme | Tam resmî HTTPS Responses/ChatGPT rotası, istek üzerinde kullanıcı tarafından belirlenmiş geçersiz kılma yok, çalışma zamanı ayarlanmamış/`auto` | OpenAI sağlayıcı/model yapılandırması |
| ChatGPT/Codex OAuth ile oturum açma                 | `openclaw models auth login --provider openai`                                                   | CLI kimlik doğrulama profili       |
| Codex çalıştırmaları için API anahtarı yedeği ekleme | `auth.order.openai` içinde abonelik kimlik doğrulamasından sonra listelenen `openai:*` API anahtarı profili | CLI kimlik doğrulama profili + OpenClaw yapılandırması |
| Codex kullanılamadığında kapalı durumda başarısız olma | Sağlayıcı veya model `agentRuntime.id: "codex"`                                                     | OpenClaw model/sağlayıcı yapılandırması |
| Doğrudan OpenAI API trafiği kullanma                | Normal OpenAI kimlik doğrulamasıyla sağlayıcı veya model `agentRuntime.id: "openclaw"`                  | OpenClaw model/sağlayıcı yapılandırması |
| App-server davranışını ayarlama                     | `plugins.entries.codex.config.appServer.*`                                                       | Codex plugin yapılandırması        |
| Yerel Codex plugin uygulamalarını etkinleştirme     | `plugins.entries.codex.config.codexPlugins.*`                                                    | Codex plugin yapılandırması        |
| Codex Bilgisayar Kullanımı'nı etkinleştirme         | `plugins.entries.codex.config.computerUse.*`                                                     | Codex plugin yapılandırması        |

Abonelik öncelikli/API anahtarı yedekli sıralama için `auth.order.openai`
tercih edin. Mevcut eski Codex kimlik doğrulama profili kimlikleri ve eski Codex
kimlik doğrulama sırası yalnızca doctor tarafından işlenen eski durumdur; yeni
eski Codex GPT referansları yazmayın.

```json5
{
  auth: {
    order: {
      openai: ["openai:user@example.com", "openai:api-key-backup"],
    },
  },
}
```

Codex ile uyumlu etkili bir rota için yukarıdaki iki profil de aynı Codex
çalıştırmasının adayları olarak kalır. Profil sırası çalışma zamanını değil,
kimlik bilgilerini seçer. Kimlik doğrulama sırasını değiştirmek özel,
Completions, HTTP veya istek tarafından geçersiz kılınmış bir rotayı Codex ile
uyumlu hâle getirmez.

### Compaction

Codex destekli ajanlarda `compaction.model` veya `compaction.provider`
ayarlamayın. Codex, yerel app-server iş parçacığı durumu aracılığıyla Compaction
gerçekleştirir; bu nedenle OpenClaw çalışma zamanında bu yerel özetleyici geçersiz
kılmalarını yok sayar ve ajan Codex kullandığında `openclaw doctor --fix` bunları
kaldırır.

Lossless, Codex turlarının çevresindeki birleştirme, içe alma ve bakım için bir
bağlam motoru olarak desteklenmeye devam eder; `agents.defaults.compaction.provider` üzerinden
değil, `plugins.slots.contextEngine: "lossless-claw"` ve `plugins.entries.lossless-claw.config.summaryModel` üzerinden yapılandırılır.
Codex etkin çalışma zamanı olduğunda `openclaw doctor --fix`, eski
`compaction.provider: "lossless-claw"` biçimini Lossless bağlam motoru yuvasına taşır; ancak
Compaction'ın sahibi yine yerel Codex'tir. Yerel app-server yürütme düzeneği,
istem öncesi birleştirme gerektiren bağlam motorlarını destekler; `codex-cli`
dâhil genel CLI arka uçları bu ana makine yeteneğini sağlamaz.

Codex destekli ajanlar için `/compact`, bağlı iş parçacığında yerel Codex
app-server Compaction işlemini başlatır ve nihai sonucunu bekler. Paylaşılan
`agents.defaults.compaction.timeoutSeconds` bütçesi uygulanır; zaman aşımında OpenClaw, Codex'ten yerel
turu kesmesini ister ve sonlandırma onaylanana kadar iş parçacığı başına korumayı
sürdürür. Hiçbir zaman bir bağlam motoruna veya herkese açık OpenAI özetleyicisine
geri dönmez. Yerel Codex iş parçacığı bağlaması eksik veya eskiyse komut,
Compaction arka uçlarını sessizce değiştirmek yerine kapalı durumda başarısız
olur.

### Doğrudan API uzun bağlam

Codex aboneliği ve doğrudan OpenAI API trafiği ayrı sözleşmelerdir. Canlı
ChatGPT/Codex kataloğu genellikle `272000` tokenlık bir model penceresi sunarken
OpenAI, GPT-5.5 ve GPT-5.6 için `1050000` tokenlık Platform API penceresi ve
`128000` maksimum çıktı belgelemektedir. Tam çıktı payının ayrılması,
türetilmiş `922000` tokenlık bir girdi bütçesi bırakır. `272000` girdi
tokenını aşan isteklerde OpenAI'ın daha yüksek uzun bağlam fiyatlandırması uygulanır.

Yüklü Codex sürümüyle uyumlu, eksiksiz bir Codex model kataloğuyla başlayın.
Uzun bağlam kullanması gereken her doğrudan GPT-5.5 veya GPT-5.6 girdisi için
tanımlayıcının geri kalanını koruyup şunları ayarlayın:

```json
{
  "context_window": 922000,
  "max_context_window": 922000,
  "auto_compact_token_limit": 700000
}
```

Codex, `922000` katalog değerine normal %95 etkin pencere rezervini
uygular; bu nedenle yaklaşık `875900` kullanılabilir token bildirir. `700000`
değerinde sıkıştırma, bu etkin korumadan önce `175900` token ve sağlayıcı
açısından güvenli girdi payından önce `222000` token bırakır. Bu daha geniş
marj kasıtlıdır: Codex, sonraki kullanıcı mesajını ve bağlam güncellemelerini
eklemeden önce zaten kaydedilmiş bağlamı denetler; dolayısıyla eşik, araçların,
talimatların, serileştirmenin ve sıkıştırma turunun yanı sıra büyük bir gelen
turu da kapsamalıdır.

Bağımsız Codex CLI veya Desktop kullanımı için komut kimlik doğrulamalı özel
sağlayıcı, normal ChatGPT oturumu bağlayıcılar için kullanılabilir kalırken API
anahtarını sistem anahtar zincirinden veya gizli bilgi yöneticisinden okuyabilir:

```toml
model = "gpt-5.6-terra"
model_provider = "openai_api_direct"
model_context_window = 922000
model_auto_compact_token_limit = 700000
model_auto_compact_token_limit_scope = "total"
model_catalog_json = "/absolute/path/to/models-api-1m.json"

[model_providers.openai_api_direct]
name = "OpenAI API direct"
base_url = "https://api.openai.com/v1"
wire_api = "responses"
requires_openai_auth = false

[model_providers.openai_api_direct.auth]
command = "/absolute/path/to/read-openai-inference-key"
timeout_ms = 5000
refresh_interval_ms = 300000
```

Kimlik doğrulama yardımcısı stdout'a yalnızca anahtarı yazdırmalıdır. Anahtarı
TOML içine koymayın.

OpenClaw Codex app-server koşum takımı için varsayılan, aracı kapsamındaki Codex
ana dizinini koruyun ve OpenClaw'ın bir `openai` API anahtarı profili eklemesine
izin verin. Kataloğu ve bağlam sınırlarını yerel Codex app-server bağımsız
değişkenleri olarak geçirin:

```json5
{
  auth: {
    order: {
      openai: ["openai:api-key"],
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            args: [
              "app-server",
              "--listen",
              "stdio://",
              "-c",
              'model_catalog_json="/absolute/path/to/models-api-1m.json"',
              "-c",
              "model_context_window=922000",
              "-c",
              "model_auto_compact_token_limit=700000",
              "-c",
              "model_auto_compact_token_limit_scope=total",
            ],
          },
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-terra",
      models: {
        "openai/gpt-5.6-terra": { agentRuntime: { id: "codex" } },
      },
    },
  },
}
```

Gerekirse `openai:api-key` değerini gerçek API anahtarı profil kimliğiyle değiştirin.
Aracı kapsamındaki app-server yalnızca hazırlanan bu anahtarı alır; operatörün
yerel `~/.codex` ChatGPT oturumu, pluginleri, bağlayıcıları ve ileti dizisi
deposu değişmeden kalır. Codex app-server `0.144.6`, app-server turlarında
komut kimlik doğrulamalı özel sağlayıcının bearer değerini eklemez; bu nedenle
bu rota için `homeScope: "user"` yerine yukarıdaki eklenmiş API anahtarı yolunu kullanın.

Kataloğu veya app-server bağımsız değişkenlerini değiştirdikten sonra Gateway'i
yeniden başlatıp yeni bir sohbet başlatın. Mevcut yerel ileti dizileri kaydedilmiş
sağlayıcı ve model ayarlarını korur. Çalışma zamanını `/status` ve
`/codex status` ile doğrulayın, ardından uzun bir oturuma başlamadan önce zararsız
bir doğrudan API turu gönderin.

<Warning>
Uzun bağlam kasıtlı olarak isteğe bağlıdır. Girdi `272000` tokenı aştığında
OpenAI, isteğin tamamını 2× girdi ve 1.5× çıktı oranlarıyla ücretlendirir. Erişim,
gerçek sınırlar ve faturalandırma konusunda API belirleyicidir. Bkz.
[OpenAI model sınırları](https://developers.openai.com/api/docs/models/compare) ve
[API fiyatlandırması](https://developers.openai.com/api/docs/pricing).
</Warning>

Bu sayfanın geri kalanında dağıtım biçimi, kapalı hata yönlendirmesi, koruyucu
onay politikası, yerel Codex pluginleri ve Bilgisayar Kullanımı ele alınmaktadır.
Eksiksiz seçenek listeleri, varsayılanlar, enumlar, keşif, ortam yalıtımı, zaman
aşımları ve app-server aktarım alanları için
[Codex koşum takımı referansına](/tr/plugins/codex-harness-reference) bakın.

## Codex çalışma zamanını doğrulama

Codex beklediğiniz sohbette `/status` kullanın. Codex destekli bir OpenAI
aracı turu şunu gösterir:

```text
Çalışma zamanı: OpenAI Codex
```

Ardından Codex app-server durumunu denetleyin:

```text
/codex status
/codex models
/codex binding
```

`/codex binding`, bağlı yerel ileti dizisini ve geçerli model ayarlarını bildirir.
`/codex status`, app-server bağlantısını, hesabı, hız sınırlarını, MCP
sunucularını ve becerileri bildirir. `/codex models`, koşum takımı ve hesap için
canlı Codex app-server kataloğunu listeler. `/status` beklenmedik görünüyorsa
[Sorun giderme](#troubleshooting) bölümüne bakın.

## Yönlendirme ve model seçimi

Sağlayıcı referanslarını ve çalışma zamanı politikasını ayrı tutun:

- Standart OpenAI model seçimi için `openai/gpt-*` kullanın. Önek tek başına
  hiçbir zaman Codex'i seçmez.
- Çalışma zamanı ayarlanmamışken veya `auto` olduğunda yalnızca yazılmış
  istek geçersiz kılması bulunmayan, tam olarak resmî HTTPS Platform Responses
  ya da ChatGPT Responses rotası Codex'i örtük olarak seçebilir.
- Yapılandırmada eski Codex GPT referanslarını kullanmayın; eski referansları ve
  güncelliğini yitirmiş oturum rota sabitlemelerini onarmak için `openclaw doctor --fix` çalıştırın.
- `agentRuntime.id: "codex"`, Codex'i uyumlu bir rota için kapalı hata gereksinimi hâline
  getirir. Uyumsuz bir etkin rotayı uyumlu hâle getirmez.
- `agentRuntime.id: "openclaw"`, kasıtlı olduğunda bir sağlayıcıyı veya modeli gömülü
  OpenClaw çalışma zamanına dahil eder.
- `/codex ...`, yerel Codex app-server konuşmalarını sohbetten denetler.
- ACP/acpx ayrı bir haricî koşum takımı yoludur. Yalnızca kullanıcı ACP/acpx
  veya haricî bir koşum takımı bağdaştırıcısı istediğinde kullanın.

| Kullanıcı amacı                                             | Kullanılacak                                                                                          |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Geçerli sohbeti bağlama                                    | `/codex bind [thread-id] [--cwd <path>] [--model <model>] [--provider <provider>]`                    |
| Mevcut bir Codex ileti dizisini sürdürme                    | `/codex resume <thread-id>`                                                                           |
| Codex ileti dizilerini listeleme veya filtreleme            | `/codex threads [filter]`                                                                             |
| Bağlı ileti dizisinin yerel hedefini okuma veya güncelleme  | `/codex goal [status\|set <objective>\|pause\|resume\|block\|complete\|clear]`                        |
| Yerel Codex pluginlerini listeleme                          | `/codex plugins list`                                                                                 |
| Yapılandırılmış yerel bir Codex pluginini etkinleştirme veya devre dışı bırakma | `/codex plugins enable <name>`, `/codex plugins disable <name>`                                       |
| Depolanmış bir Codex CLI oturumunu eşleştirilmiş Node turu olarak sürdürme | `/codex sessions --host <node> [filter]`, ardından `/codex resume <session-id> --host <node> --bind here` |
| Bilgisayarlar arasındaki arşivlenmemiş Codex oturumlarını görüntüleme | Codex gözetimini etkinleştirip **Codex Oturumları** bölümünü açın                                    |
| Bağlı ileti dizisinin modelini, hızlı modunu veya izinlerini değiştirme | `/codex model <model>`, `/codex fast [on\|off\|status]`, `/codex permissions [default\|yolo\|status]` |
| Etkin turu durdurma veya yönlendirme                        | `/codex stop`, `/codex steer <text>`                                                                  |
| Geçerli bağlantıyı ayırma                                  | `/codex detach` (`/codex unbind` diğer adıyla)                                                       |
| Yalnızca Codex geri bildirimi gönderme                      | `/codex diagnostics [note]`                                                                           |
| Bir ACP/acpx görevi başlatma                                | `/codex` değil, ACP/acpx oturum komutları                                                           |

| Kullanım örneği                                  | Yapılandırma                                                                                                 | Doğrulama                               | Notlar                                     |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | --------------------------------------- | ------------------------------------------ |
| Yerel Codex çalışma zamanına sahip uygun OpenAI rotası | Yazılmış istek geçersiz kılması bulunmayan, tam olarak resmî HTTPS Responses/ChatGPT rotası ve etkin `codex` plugini | `/status`, `Runtime: OpenAI Codex` gösterir | Çalışma zamanı ayarlanmamışken/`auto` olduğunda örtük yol |
| Codex kullanılamıyorsa kapalı hata               | Sağlayıcı veya model `agentRuntime.id: "codex"`                                                                | Tur, gömülü geri dönüş yerine başarısız olur | Yalnızca Codex dağıtımları için kullanın   |
| OpenClaw üzerinden doğrudan OpenAI API anahtarı trafiği | Sağlayıcı veya model `agentRuntime.id: "openclaw"` ve normal OpenAI kimlik doğrulaması                                      | `/status`, OpenClaw çalışma zamanını gösterir | Yalnızca OpenClaw kasıtlıysa kullanın      |
| Eski yapılandırma                                | eski Codex GPT referansları                                                                                  | `openclaw doctor --fix` bunu yeniden yazar     | Yeni yapılandırmayı bu şekilde yazmayın    |
| ACP/acpx Codex bağdaştırıcısı                    | ACP `sessions_spawn({ runtime: "acp" })`                                                                    | ACP görev/oturum durumu                 | Yerel Codex koşum takımından ayrıdır       |

`agents.defaults.imageModel` aynı önek ayrımını izler. Normal OpenAI rotası için
`openai/gpt-*`, yalnızca görüntü anlama işleminin sınırlı bir Codex app-server turu
üzerinden çalışması gerektiğinde ise `codex/gpt-*` kullanın. Doctor, eski Codex
GPT referanslarını `openai/gpt-*` olarak yeniden yazar.

## Dağıtım kalıpları

### Temel Codex dağıtımı

Etkin resmî HTTPS rotası Codex'i örtük olarak seçmeye uygun olan bir OpenAI
modeli için hızlı başlangıç yapılandırmasını kullanın:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

### Karma sağlayıcılı dağıtım

Claude'u varsayılan aracı olarak koruyup adlandırılmış bir Codex aracısı ekleyin:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "anthropic/claude-opus-4-6",
    },
    list: [
      {
        id: "main",
        default: true,
        model: "anthropic/claude-opus-4-6",
      },
      {
        id: "codex",
        name: "Codex",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

`main` aracısı normal sağlayıcı yolunu kullanır. `codex` aracısı, etkin
OpenAI rotası uyumlu kaldığında Codex app-server kullanır; bunun kapalı hata
gereksinimi olması gerekiyorsa açıkça model kapsamlı `agentRuntime.id: "codex"` ekleyin.

### Kapalı hata Codex dağıtımı

Uygun, tam olarak resmî bir HTTPS OpenAI rotası, paketlenmiş plugin kullanılabilir
olduğunda Codex'e çözümlenebilir. Yazılı bir kapalı hata kuralı için açık çalışma
zamanı politikası ekleyin:

```json5
{
  models: {
    providers: {
      openai: {
        agentRuntime: {
          id: "codex",
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Codex zorunlu kılındığında, etkin rota Codex uyumlu olarak bildirilmemişse, plugin devre dışıysa, app-server çok eskiyse veya app-server başlatılamıyorsa OpenClaw işlemi erkenden sonlandırır.

## App-server politikası

Plugin, varsayılan olarak OpenClaw tarafından yönetilen Codex ikili dosyasını stdio aktarımıyla yerel olarak başlatır. Yalnızca kasıtlı olarak farklı bir yürütülebilir dosya çalıştırmak için `appServer.command` ayarını belirleyin. Codex, WebSocket aktarımını deneysel ve desteklenmeyen olarak sınıflandırır; bunu yalnızca başka bir yerde zaten çalışan bir app-server ile üretim dışı testlerde kullanın:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
          },
        },
      },
    },
  },
}
```

Yerel stdio app-server oturumları varsayılan olarak güvenilir yerel operatör duruşunu kullanır: `approvalPolicy: "never"`, `approvalsReviewer: "user"` ve `sandbox: "danger-full-access"`. Yerel Codex gereksinimleri bu örtük YOLO duruşuna izin vermiyorsa OpenClaw bunun yerine izin verilen koruyucu izinlerini seçer. Oturum için bir OpenClaw korumalı alanı etkinken OpenClaw, Codex ana makine tarafı korumalı alanına güvenmek yerine o tur için Codex yerel Code Mode'u, kullanıcı MCP sunucularını ve uygulama destekli plugin yürütmesini devre dışı bırakır. Bunun yerine kabuk erişimi, normal exec/process araçları kullanılabildiğinde `sandbox_exec` ve `sandbox_process` gibi OpenClaw korumalı alanı destekli dinamik araçlar üzerinden sağlanır.

Korumalı alandan kaçışlardan veya ek izinlerden önce Codex yerel otomatik incelemesi için normalleştirilmiş OpenClaw exec modunu kullanın:

```json5
{
  tools: {
    exec: {
      mode: "auto",
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Codex app-server oturumlarında `tools.exec.mode: "auto"`, Codex Guardian tarafından incelenen onaylarla eşleştirilir: yerel gereksinimler bu değerlere izin verdiğinde genellikle `approvalPolicy: "on-request"`, `approvalsReviewer: "auto_review"` ve `sandbox: "workspace-write"`. `tools.exec.mode: "auto"` içinde OpenClaw, eski ve güvenli olmayan Codex `approvalPolicy: "never"` veya `sandbox: "danger-full-access"` geçersiz kılmalarını korumaz; kasıtlı olarak onaysız bir Codex duruşu için `tools.exec.mode: "full"` kullanın. Eski `plugins.entries.codex.config.appServer.mode: "guardian"` ön ayarı çalışmaya devam eder, ancak normalleştirilmiş OpenClaw yüzeyi `tools.exec.mode: "auto"` değeridir.

Ana makine exec onayları ve ACPX izinleriyle mod düzeyindeki karşılaştırma için [İzin modları](/tr/tools/permission-modes) bölümüne bakın. Tüm app-server alanları, kimlik doğrulama sırası, ortam yalıtımı ve zaman aşımı davranışı için [Codex donanımı referansı](/tr/plugins/codex-harness-reference) bölümüne bakın.

## Komutlar ve tanılama

`codex` plugin'i, OpenClaw metin komutlarını destekleyen tüm kanallarda `/codex` komutunu eğik çizgi komutu olarak kaydeder.

Yerel yürütme ve denetim için bir sahip veya `operator.admin` Gateway istemcisi gerekir: iş parçacıklarını bağlama veya sürdürme, turları gönderme veya durdurma, model, hızlı mod ya da izin durumunu değiştirme, sıkıştırma veya inceleme ve bir bağlantıyı ayırma. Diğer yetkili gönderenler; salt okunur durum, yardım, hesap, model, iş parçacığı, yerel hedef, MCP sunucusu, beceri ve bağlantı inceleme komutlarını kullanmaya devam eder.

Yaygın biçimler:

- `/codex status` app-server bağlantısını, modelleri, hesabı, hız sınırlarını, MCP sunucularını ve becerileri denetler.
- `/codex models` canlı Codex app-server modellerini listeler.
- `/codex threads [filter]` son Codex app-server iş parçacıklarını listeler.
- `/codex goal` bağlı iş parçacığının yerel Codex hedefini okur veya günceller. Codex otomatik hedef sürdürme özelliği devre dışı kalır; OpenClaw henüz özerk takip turlarını yönetmez.
- `/codex resume <thread-id>` geçerli OpenClaw oturumunu mevcut bir Codex iş parçacığına bağlar.
- `/codex bind [thread-id] [--cwd <path>] [--model <model>] [--provider <provider>]` geçerli sohbeti bağlar.
- `/codex detach` (veya `/codex unbind`) geçerli bağlantıyı ayırır.
- `/codex binding` geçerli bağlantıyı açıklar.
- `/codex stop` etkin turu durdurur; `/codex steer <text>` onu yönlendirir.
- `/codex model <model>`, `/codex fast [on|off|status]` ve `/codex permissions [default|yolo|status]` konuşma başına durumu değiştirir.
- `/codex compact`, Codex app-server'dan bağlı iş parçacığını sıkıştırmasını ister.
- `/codex review`, bağlı iş parçacığı için Codex yerel incelemesini başlatır.
- `/codex diagnostics [note]`, bağlı iş parçacığı için Codex geri bildirimi göndermeden önce onay ister.
- `/codex account` hesap ve hız sınırı durumunu gösterir.
- `/codex mcp` Codex app-server MCP sunucusu durumunu listeler.
- `/codex skills` Codex app-server becerilerini listeler.
- `/codex plugins list`, `/codex plugins enable <name>` ve `/codex plugins disable <name>` yapılandırılmış yerel Codex plugin'lerini yönetir.
- `/codex computer-use [status|install]` Codex Computer Use özelliğini yönetir.
- `/codex help` tam komut ağacını listeler.

Çoğu destek bildirimi için hatanın oluştuğu konuşmada `/diagnostics [note]` ile başlayın. Bu komut bir Gateway tanılama raporu oluşturur ve Codex donanımı oturumlarında ilgili Codex geri bildirim paketini göndermek için onay ister. Gizlilik modeli ve grup sohbeti davranışı için [Tanılama dışa aktarma](/tr/gateway/diagnostics) bölümüne bakın. Tam Gateway tanılama paketi olmadan yalnızca o anda bağlı olan iş parçacığına ait Codex geri bildirimini yüklemek istediğinizde `/codex diagnostics [note]` kullanın.

### Codex iş parçacıklarını yerel olarak inceleme

Sorunlu bir Codex çalışmasını incelemenin en hızlı yolu genellikle yerel Codex iş parçacığını doğrudan açmaktır:

```bash
codex resume <thread-id>
```

İş parçacığı kimliğini tamamlanan `/diagnostics` yanıtından, `/codex binding` veya `/codex threads [filter]` üzerinden alın.

Yükleme mekanizması ve çalışma zamanı düzeyindeki tanılama sınırları için [Codex donanımı çalışma zamanı](/tr/plugins/codex-harness-runtime#codex-feedback-upload) bölümüne bakın.

### Kimlik doğrulama sırası

Varsayılan aracı başına ana dizinde kimlik doğrulama şu sırayla seçilir:

1. Aracı için tercihen `auth.order.openai` altında bulunan sıralı OpenAI kimlik doğrulama profilleri. Eski Codex kimlik doğrulama profili kimliklerini ve eski Codex kimlik doğrulama sırasını taşımak için `openclaw doctor --fix` komutunu çalıştırın.
2. App-server'ın söz konusu aracının Codex ana dizinindeki mevcut hesabı.
3. Yalnızca yerel stdio app-server başlatmalarında, app-server hesabı yoksa ve OpenAI kimlik doğrulaması hâlâ gerekiyorsa önce `CODEX_API_KEY`, ardından `OPENAI_API_KEY`.

OpenClaw, ChatGPT aboneliği türünde bir Codex kimlik doğrulama profili gördüğünde başlatılan Codex alt işleminden `CODEX_API_KEY` ve `OPENAI_API_KEY` değerlerini kaldırır. Bu işlem, yerel Codex app-server turlarının yanlışlıkla API üzerinden ücretlendirilmesine yol açmadan Gateway düzeyindeki API anahtarlarını gömmeler veya doğrudan OpenAI modelleri için kullanılabilir tutar. Açık Codex API anahtarı profilleri ve yerel stdio ortam anahtarı geri dönüşü, devralınan alt işlem ortamı yerine app-server oturum açma yöntemini kullanır. WebSocket app-server bağlantıları Gateway ortamı API anahtarı geri dönüşünü almaz; açık bir kimlik doğrulama profili veya uzak app-server'ın kendi hesabını kullanın.

Bir abonelik profili Codex kullanım sınırına ulaşırsa OpenClaw, Codex tarafından bildirildiğinde sıfırlama zamanını kaydeder ve aynı Codex çalışması için sıradaki kimlik doğrulama profilini dener. Sıfırlama zamanı geçtikten sonra abonelik profili, seçili `openai/gpt-*` modeli veya Codex çalışma zamanı değiştirilmeden yeniden kullanılabilir hâle gelir.

Yerel Codex plugin'leri yapılandırıldığında OpenClaw, plugin'lerin sahip olduğu uygulamaları Codex iş parçacığına sunmadan önce bu plugin'leri bağlı app-server üzerinden kurar veya yeniler. `app/list`, uygulama kimlikleri, erişilebilirlik ve meta veriler için doğruluk kaynağı olmaya devam eder; ancak iş parçacığı başına etkinleştirme kararını OpenClaw yönetir: politika listelenen erişilebilir bir uygulamaya izin veriyorsa `app/list` şu anda uygulamanın devre dışı olduğunu bildirse bile OpenClaw `thread/start.config.apps[appId].enabled = true` gönderir. Bu yol, bilinmeyen kimlikler için uygulama kurulumu oluşturmaz; OpenClaw yalnızca `plugin/install` içeren pazar yeri plugin'lerini etkinleştirir ve ardından envanteri yeniler.

### Ortam yalıtımı

Yerel stdio app-server başlatmalarında OpenClaw, Codex yapılandırmasının, kimlik doğrulama/hesap dosyalarının, plugin önbelleğinin/verilerinin ve yerel iş parçacığı durumunun varsayılan olarak operatörün kişisel `~/.codex` dizinini okumaması veya bu dizine yazmaması için `CODEX_HOME` değerini aracı başına bir dizine ayarlar. OpenClaw normal işlem `HOME` değerini korur; Codex tarafından çalıştırılan alt işlemler kullanıcı ana dizini yapılandırmasını ve belirteçlerini bulmaya devam edebilir, ayrıca Codex paylaşılan `$HOME/.agents/skills` ve `$HOME/.agents/plugins/marketplace.json` girdilerini keşfedebilir. `appServer.homeScope: "user"` kullanıldığında OpenClaw bunun yerine yerel kullanıcının Codex ana dizinini ve mevcut hesabını kullanır ve bir OpenClaw kimlik doğrulama profili eklemez.

Bir dağıtım ek ortam yalıtımı gerektiriyorsa bu değişkenleri `appServer.clearEnv` içine ekleyin:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            clearEnv: ["CODEX_API_KEY", "OPENAI_API_KEY"],
          },
        },
      },
    },
  },
}
```

`appServer.clearEnv` yalnızca başlatılan Codex app-server alt işlemini etkiler. OpenClaw, yerel başlatma normalleştirmesi sırasında `CODEX_HOME` ve `HOME` değerlerini bu listeden kaldırır: `CODEX_HOME` seçilen aracı veya kullanıcı kapsamına yönelmeye devam eder ve alt işlemlerin normal kullanıcı ana dizini durumunu kullanabilmesi için `HOME` devralınmaya devam eder.

### Dinamik araçlar ve web araması

Codex dinamik araçları varsayılan olarak `searchable` yüklemesini kullanır. OpenClaw normalde Codex'in yerel çalışma alanı işlemlerini yineleyen dinamik araçları sunmaz: `read`, `write`, `edit`, `apply_patch`, `exec`, `process`, `update_plan`, `get_goal`, `create_goal`, `update_goal`, `tool_call`, `tool_describe`, `tool_search` ve `tool_search_code`. Hedef işlemleri Codex'e özgü kalır; bu nedenle OpenClaw, Codex turlarına ikinci bir hedef deposu yansıtmaz. Mesajlaşma, medya, cron, tarayıcı, node'lar, gateway ve `heartbeat_respond` gibi kalan OpenClaw entegrasyon araçlarının çoğu, ilk model bağlamını daha küçük tutmak için `openclaw` ad alanı altında Codex araç araması aracılığıyla kullanılabilir. Kısıtlı tur kabuk geri dönüşü, sonlu bir izin listesi yerel Code Mode'u devre dışı bıraktığında `exec` ve `process` için istisnadır; çalışma zamanı izin listeleri ve `codexDynamicToolsExclude` yine uygulanır.

OpenClaw `computer` aracı dâhil olmak üzere `catalogMode: "direct-only"` olarak işaretlenen araçlar bunun yerine `openclaw_direct` ad alanını kullanır. Codex bu ad alanını `DirectModelOnly` olarak değerlendirir; böylece bu araçlar iç içe Code Mode `tools.*` çağrılarından geçmek yerine normal ve yalnızca kod modu iş parçacıklarında doğrudan modele görünür kalır.

Arama etkinleştirildiğinde ve yönetilen bir sağlayıcı seçilmediğinde web araması varsayılan olarak Codex'in barındırılan `web_search` aracını kullanır. Yönetilen aramanın yerel etki alanı kısıtlamalarını aşamaması için yerel barındırılan arama ile OpenClaw'ın yönetilen `web_search` dinamik aracı birbirini dışlar. Barındırılan arama kullanılamadığında, açıkça devre dışı bırakıldığında veya seçili bir yönetilen sağlayıcıyla değiştirildiğinde OpenClaw yönetilen aracı kullanır. Üretim app-server trafiği kullanıcı tanımlı `web` ad alanını reddettiği için OpenClaw, Codex'in bağımsız `web.run` uzantısını devre dışı tutar. Yalnızca LLM kullanılan ve araçların devre dışı olduğu çalışmalar gibi `tools.web.search.enabled: false` de her iki yolu devre dışı bırakır. Codex, `"cached"` değerini bir tercih olarak değerlendirir ve kısıtlanmamış app-server turları için canlı dış erişime çözümler. Yerel `allowedDomains` ayarlandığında otomatik yönetilen geri dönüş güvenli biçimde başarısız olur; böylece izin listesi aşılamaz. Kalıcı etkin arama politikası değişiklikleri bir sonraki turdan önce bağlı Codex iş parçacığını döndürür; tur başına geçici kısıtlamalar, geçici bir kısıtlı iş parçacığı kullanır ve daha sonra sürdürmek üzere mevcut bağlantıyı korur.

`sessions_yield`, `sessions_spawn` ve yalnızca mesaj aracıyla kullanılan kaynak yanıtları,
sıra denetimi veya yetkilendirme sözleşmeleri oldukları için doğrudan kalır. Yönergeler,
birincil Codex alt ajan yüzeyi olarak hâlâ Codex'in yerel `spawn_agent` özelliğini
tercih ederken, açık OpenClaw veya ACP yetkilendirmesi
`sessions_spawn` üzerinden doğrudan çağrılabilir olmaya devam eder. Codex Code Mode'da genel OpenClaw
dinamik araç sonuçları JavaScript nesneleri yerine JSON metnidir; bu nedenle alanları
okumadan önce JSON'a benzeyen sonuçları ayrıştırın. Codex ayrıca iç içe dinamik
çağrıları seri hâle getirir; `Promise.all` öğesinin bunları eşzamanlı başlatmasını beklemek
yerine sınırlı bir döngüde birkaç `sessions_spawn` çağrısı gönderin. Daha önce kabul edilmiş
alt ajanlar, sonraki çağrılar gönderilirken yine de çakışabilir. Eksiksiz bir kalıp için
[Swarm](/tools/swarm#use-swarm-from-other-harnesses) bölümüne bakın.
Heartbeat iş birliği yönergeleri,
araç henüz yüklenmemişse bir heartbeat sırasını sonlandırmadan önce Codex'e
`heartbeat_respond` öğesini aramasını söyler.

`codexDynamicToolsLoading: "direct"` öğesini yalnızca ertelenmiş dinamik araçları arayamayan özel bir
Codex uygulama sunucusuna bağlanırken veya tam araç yükünde
hata ayıklarken ayarlayın.

### Yapılandırma alanları

Desteklenen üst düzey Codex plugin alanları:

| Alan                       | Varsayılan     | Anlamı                                                                                   |
| -------------------------- | -------------- | ---------------------------------------------------------------------------------------- |
| `codexDynamicToolsLoading` | `"searchable"` | OpenClaw dinamik araçlarını doğrudan ilk Codex araç bağlamına yerleştirmek için `"direct"` kullanın. |
| `codexDynamicToolsExclude` | `[]`           | Codex uygulama sunucusu sıralarında hariç tutulacak ek OpenClaw dinamik araç adları.      |
| `codexPlugins`             | devre dışı     | Taşınmış, kaynaktan yüklenen seçilmiş pluginler için yerel Codex plugin/uygulama desteği. |
| `sessionCatalog`           | etkin          | Bu Gateway'deki yerel Codex oturumları ve uygun eşleştirilmiş nodelar için kenar çubuğu keşfi. |
| `supervision`              | devre dışı     | Ajana yönelik yerel oturum transkripti ve yazma denetimi ilkesi.                          |

Desteklenen `appServer` alanları:

| Alan                                          | Varsayılan                                             | Anlamı                                                                                                                                                                                                                                                                                                                                                                                          |
| --------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                            | `"stdio"`                                     | `"stdio"`, Codex'i başlatır; açıkça belirtilen `"unix"` yerel denetim soketine bağlanır; `"websocket"`, `url` hedefine bağlanır.                                                                                                                                                                                                                              |
| `homeScope`                            | `"agent"`                                     | `"agent"`, sıradan çalıştırma düzeneği durumunu OpenClaw agent'ına göre yalıtır. `"user"`, yerel `$CODEX_HOME` veya `~/.codex` öğesini paylaşan, yerel kimlik doğrulamayı kullanan ve yalnızca sahip tarafından gerçekleştirilebilen iş parçacığı yönetimini etkinleştiren açık bir katılım seçeneğidir. Kullanıcı kapsamı, yerel stdio veya Unix aktarımını destekler. Ayrı gözetim bağlantısı için ayarlanmamış bir değer, stdio veya Unix için `"user"`, WebSocket için ise `"agent"` olarak çözümlenir. |
| `command`                            | yönetilen Codex ikili dosyası                          | Stdio aktarımına yönelik yürütülebilir dosya. Yönetilen ikili dosyayı kullanmak için ayarlamadan bırakın; yalnızca açıkça geçersiz kılmak için ayarlayın.                                                                                                                                                                                                                                          |
| `args`                            | `["app-server", "--listen", "stdio://"]`                                     | Stdio aktarımına yönelik bağımsız değişkenler.                                                                                                                                                                                                                                                                                                                                                  |
| `url`                            | ayarlanmamış                                           | WebSocket App Server URL'si veya `unix://` URL'si. Açıkça belirtilen boş bir Unix yolu, standart kullanıcı ana dizini denetim soketini seçer.                                                                                                                                                                                                                                           |
| `authToken`                            | ayarlanmamış                                           | WebSocket aktarımına yönelik Bearer belirteci. Değişmez bir dizeyi veya `${CODEX_APP_SERVER_TOKEN}` gibi bir SecretInput değerini kabul eder.                                                                                                                                                                                                                                                            |
| `headers`                            | `{}`                                     | Ek WebSocket üstbilgileri. Üstbilgi değerleri, değişmez dizeleri veya örneğin `x-codex-client-session-token: "${CODEX_CLIENT_SESSION_TOKEN}"` gibi SecretInput değerlerini kabul eder.                                                                                                                                                                                                                                                        |
| `clearEnv`                            | `[]`                                     | OpenClaw devralınan ortamını oluşturduktan sonra, başlatılan stdio app-server işleminden kaldırılan ek ortam değişkeni adları. OpenClaw, yerel başlatmalar için seçili `CODEX_HOME` değerini ve devralınan `HOME` değerini korur.                                                                                                                                                    |
| `codeModeOnly`                            | `false`                                     | Codex'in yalnızca kod modu araç yüzeyini etkinleştirin. Sıradan OpenClaw dinamik araçları, iç içe `tools.*` çağrıları aracılığıyla kullanılabilir olmaya devam eder; `openclaw_direct` araçları doğrudan modele görünür kalır.                                                                                                                                                              |
| `remoteWorkspaceRoot`                            | ayarlanmamış                                           | Uzak Codex app-server çalışma alanı kökü. Ayarlandığında OpenClaw, çözümlenen OpenClaw çalışma alanından yerel çalışma alanı kökünü çıkarır, geçerli cwd son ekini bu uzak kök altında korur ve Codex'e yalnızca nihai app-server cwd değerini gönderir. cwd, çözümlenen OpenClaw çalışma alanı kökünün dışındaysa OpenClaw, uzak app-server'a Gateway'e ait yerel bir yol göndermek yerine güvenli biçimde başarısız olur. |
| `requestTimeoutMs`                            | `60000`                                     | App-server denetim düzlemi çağrılarının zaman aşımı.                                                                                                                                                                                                                                                                                                                                           |
| `turnCompletionIdleTimeoutMs`                            | `60000`                                     | Codex bir turu kabul ettikten veya tur kapsamlı bir app-server isteğinden sonra OpenClaw `turn/completed` için beklerken uygulanan sessiz süre.                                                                                                                                                                                                                                                |
| `turnAssistantCompletionIdleTimeoutMs`                            | `10000`                                     | Nihai/yorum dışı bir assistant öğesi veya araç öncesi ham assistant tamamlaması, OpenClaw hâlâ `turn/completed` için beklerken assistant çıktısının serbest bırakılmasını etkinleştirdikten sonraki sessiz süre. Bu değeri artırmak, OpenClaw oturumu kesip oturum şeridini serbest bırakmadan önce Codex'e `turn/completed` yayması için daha fazla süre tanır.                                      |
| `postToolRawAssistantCompletionIdleTimeoutMs`                            | `300000`                                     | OpenClaw `turn/completed` için beklerken araç devrinden, yerel araç tamamlanmasından, araç sonrası ham assistant ilerlemesinden, ham akıl yürütme tamamlanmasından veya akıl yürütme ilerlemesinden sonra kullanılan tamamlanma-boşta kalma ve ilerleme koruması. Araç sonrası sentezin nihai assistant serbest bırakma bütçesinden meşru biçimde daha uzun süre sessiz kalabileceği güvenilir veya ağır iş yükleri için bunu kullanın. |
| `mode`                            | yerel Codex gereksinimleri YOLO'ya izin vermediği sürece `"yolo"` | YOLO veya koruyucu tarafından incelenen yürütmeye yönelik ön ayar. `danger-full-access`, `never` onayı veya `user` inceleyicisini belirtmeyen yerel stdio gereksinimleri, örtük varsayılanı koruyucu yapar.                                                                                                                                                                     |
| `approvalPolicy`                            | `"never"` veya izin verilen bir koruyucu onay ilkesi | İş parçacığı başlatma/sürdürme/tur işlemine gönderilen yerel Codex onay ilkesi. Koruyucu varsayılanları, izin verildiğinde `"on-request"` değerini tercih eder.                                                                                                                                                                                                                                |
| `sandbox`                            | `"danger-full-access"` veya izin verilen bir koruyucu sandbox'ı | İş parçacığı başlatma/sürdürme işlemine gönderilen yerel Codex sandbox modu. Koruyucu varsayılanları, izin verildiğinde `"workspace-write"` değerini; aksi takdirde `"read-only"` değerini tercih eder. Bir OpenClaw sandbox'ı etkinken `danger-full-access` turları, OpenClaw sandbox çıkış ayarından türetilen ağ erişimiyle Codex `workspace-write` kullanır.                                        |
| `approvalsReviewer`                            | `"user"` veya izin verilen bir koruyucu inceleyici | İzin verildiğinde Codex'in yerel onay istemlerini incelemesini sağlamak için `"auto_review"`; aksi takdirde `guardian_subagent` veya `user` kullanın. `guardian_subagent`, eski bir diğer ad olarak kalır.                                                                                                                                                                              |
| `serviceTier`                            | ayarlanmamış                                           | İsteğe bağlı Codex app-server hizmet katmanı. `"priority"` hızlı mod yönlendirmesini etkinleştirir, `"flex"` esnek işleme talep eder, `null` geçersiz kılmayı temizler ve eski `"fast"`, `"priority"` olarak kabul edilir.                                                                                                                                 |
| `networkProxy`                            | devre dışı                                             | App-server komutları için Codex izin profili ağını etkinleştirin. OpenClaw, seçili `permissions.<profile>.network` yapılandırmasını tanımlar ve `sandbox` göndermek yerine bunu `default_permissions` ile seçer.                                                                                                                                                                                           |
| `experimental.sandboxExecServer`                            | `false`                                     | Yerel Codex yürütmesinin etkin OpenClaw sandbox'ı içinde çalışabilmesi için desteklenen Codex app-server'a OpenClaw sandbox destekli bir ortam kaydeden önizleme katılım seçeneği.                                                                                                                                                                                                                |

`appServer.networkProxy`, Codex korumalı alan sözleşmesini değiştirdiği için açıkça belirtilir. Etkinleştirildiğinde OpenClaw, oluşturulan izin profilinin Codex tarafından yönetilen ağı başlatabilmesi için Codex iş parçacığı yapılandırmasında `features.network_proxy.enabled`
ve `default_permissions` değerlerini de ayarlar. OpenClaw varsayılan olarak profil gövdesinden çakışmaya dayanıklı bir `openclaw-network-<fingerprint>` profil
adı oluşturur; `profileName` seçeneğini yalnızca kararlı bir yerel ad
gerektiğinde kullanın.

```json5
{
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            sandbox: "workspace-write",
            networkProxy: {
              enabled: true,
              domains: {
                "api.openai.com": "allow",
                "blocked.example.com": "deny",
              },
              unixSockets: {
                "/tmp/proxy.sock": "allow",
                "/tmp/blocked.sock": "none",
              },
              allowUpstreamProxy: true,
              proxyUrl: "http://127.0.0.1:3128",
            },
          },
        },
      },
    },
  },
}
```

Normal uygulama sunucusu çalışma zamanı `danger-full-access` olacaksa,
`networkProxy` seçeneğinin etkinleştirilmesi oluşturulan izin profili için
çalışma alanı tarzı dosya sistemi erişimi kullanır: Codex tarafından yönetilen ağ uygulaması korumalı alanlı ağ olduğundan, tam erişimli bir profil giden trafiği korumaz.
Etki alanı girdileri `allow` veya `deny` kullanır; Unix soketi girdileri Codex'in
`allow` veya `none` değerlerini kullanır.

### Dinamik araç çağrısı zaman aşımları

OpenClaw'a ait dinamik araç çağrıları
`appServer.requestTimeoutMs` değerinden bağımsız olarak sınırlandırılır: Codex `item/tool/call` istekleri varsayılan olarak 90
saniyelik bir OpenClaw gözetim zamanlayıcısı kullanır. Çağrı başına pozitif bir `timeoutMs`
bağımsız değişkeni, 600000 ms ile sınırlı olmak üzere söz konusu aracın bütçesini uzatır veya kısaltır.
`image_generate` aracı, araç çağrısı kendi zaman aşımını sağlamadığında `agents.defaults.mediaModels.image.timeoutMs`
değerini; aksi durumda ise 120 saniyelik görüntü oluşturma varsayılanını kullanır. Medya anlama `image` aracı,
seçilen görüntü destekli `tools.media.models[]` girdisinin `timeoutSeconds` değerini veya 60 saniyelik medya varsayılanını kullanır; görüntü anlama için
bu zaman aşımı isteğin kendisine uygulanır ve daha önceki hazırlık çalışmaları nedeniyle azaltılmaz. Zaman aşımında OpenClaw, desteklendiği yerlerde araç
sinyalini iptal eder ve Codex'e başarısız bir dinamik araç yanıtı döndürür;
böylece oturumu `processing` durumunda bırakmak yerine dönüş devam edebilir.
Bu gözetim zamanlayıcısı dış dinamik `item/tool/call` bütçesidir; sağlayıcıya özgü
istek zaman aşımları bu çağrının içinde çalışır ve kendi zaman aşımı anlamlarını korur.

Codex bir dönüşü kabul ettikten ve OpenClaw dönüş kapsamlı bir
uygulama sunucusu isteğine yanıt verdikten sonra, düzenek Codex'in geçerli dönüşte ilerleme kaydetmesini
ve sonunda yerel dönüşü `turn/completed` ile tamamlamasını bekler. Uygulama sunucusu
`appServer.turnCompletionIdleTimeoutMs` boyunca sessiz kalırsa OpenClaw, Codex dönüşünü
mümkün olan en iyi şekilde kesintiye uğratır, tanılama amaçlı bir zaman aşımı kaydeder ve
takip eden sohbet iletilerinin eski bir yerel dönüşün arkasında
kuyruğa alınmaması için OpenClaw oturum hattını serbest bırakır. Aynı dönüşe ait çoğu
sonlandırıcı olmayan bildirim, Codex dönüşün hâlâ etkin olduğunu kanıtladığından bu kısa gözetim zamanlayıcısını devre dışı bırakır.

Araç devirleri daha uzun bir araç sonrası boşta kalma bütçesi kullanır: OpenClaw bir
`item/tool/call` yanıtı döndürdükten, `commandExecution` gibi yerel araç öğeleri
tamamlandıktan, ham `custom_tool_call_output` tamamlamalarından ve araç sonrası ham asistan ilerlemesi, ham akıl yürütme
tamamlamaları veya akıl yürütme ilerlemesinden sonra. Koruma, yapılandırıldığında
`appServer.postToolRawAssistantCompletionIdleTimeoutMs` değerini kullanır ve aksi durumda
varsayılan olarak beş dakikadır; aynı bütçe, Codex'in bir sonraki geçerli dönüş olayını yayımlamasından önceki
sessiz sentez penceresi için ilerleme gözetim zamanlayıcısını da uzatır. Hız sınırı güncellemeleri gibi
genel uygulama sunucusu bildirimleri, dönüş boşta kalma ilerlemesini sıfırlamaz. Akıl yürütme tamamlamalarını,
yorum `agentMessage` tamamlamalarını ve araç öncesi ham akıl yürütme veya
asistan ilerlemesini otomatik bir nihai yanıt izleyebileceğinden, bunlar oturum hattını
hemen serbest bırakmak yerine ilerleme sonrası yanıt korumasını kullanır.

Yalnızca nihai/yorum olmayan tamamlanmış `agentMessage` öğeleri ve araç öncesi ham
asistan tamamlamaları, asistan çıktısı serbest bırakmasını devreye alır: Codex daha sonra
`turn/completed` olmadan sessiz kalırsa OpenClaw yerel dönüşü mümkün olan en iyi şekilde kesintiye uğratır
ve oturum hattını serbest bırakır. Başka bir dönüş izleyicisi bu serbest bırakma
yarışını kazanırsa OpenClaw; hiçbir yerel istek, öğe veya dinamik araç tamamlaması etkin kalmadığında,
asistan çıktısı serbest bırakması hâlâ en son tamamlanan öğeye ait olduğunda
ve daha sonra tamamlanmış bir öğe bulunmadığında tamamlanmış nihai asistan öğesini yine de kabul eder.
Bu, dönüşü yeniden oynatmadan tamamlanmış araç çalışmasından sonraki nihai yanıtı koruyabilir.
Kısmi asistan deltaları, önceki eski yanıtlar ve daha sonraki boş tamamlamalar uygun değildir.

Asistan, araç, etkin öğe veya yan etki kanıtı bulunmayan dönüş tamamlama boşta kalma
zaman aşımları dâhil olmak üzere yeniden oynatması güvenli stdio uygulama sunucusu hataları,
yeni bir uygulama sunucusu denemesinde bir kez yeniden denenir. Güvenli olmayan zaman aşımları yine de takılı kalan
uygulama sunucusu istemcisini kullanımdan kaldırır ve OpenClaw oturum hattını serbest bırakır; ayrıca
otomatik olarak yeniden oynatılmak yerine eski yerel iş parçacığı bağlamasını temizler.
Tamamlama izleme zaman aşımları Codex'e özgü zaman aşımı metni gösterir:
yeniden oynatması güvenli durumlarda yanıtın eksik olabileceği belirtilirken, güvenli olmayan
durumlarda kullanıcıya yeniden denemeden önce geçerli durumu doğrulaması söylenir. Genel zaman aşımı
tanılamaları; son uygulama sunucusu bildirim yöntemi, ham asistan yanıt öğesi kimliği/türü/rolü,
etkin istek/öğe sayıları ve devredeki izleme durumu gibi yapısal alanları içerir; son bildirim bir
ham asistan yanıt öğesiyse sınırlı bir asistan metni önizlemesi de içerir.
Ham istem veya araç içeriğini içermezler.

### Yerel test ortamı geçersiz kılmaları

- `OPENCLAW_CODEX_APP_SERVER_BIN`, `appServer.command` ayarlanmamışsa yönetilen ikili dosyayı atlar.
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_MODE=yolo|guardian`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`

`OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1` kaldırıldı. Bunun yerine
`plugins.entries.codex.config.appServer.mode: "guardian"` veya tek seferlik yerel test için
`OPENCLAW_CODEX_APP_SERVER_MODE=guardian` kullanın. Yinelenebilir dağıtımlar için yapılandırma
tercih edilir; çünkü Plugin davranışını Codex düzeneğinin geri kalanıyla aynı incelenmiş dosyada tutar.

## Yerel Codex Plugin'leri

Yerel Codex Plugin desteği, OpenClaw düzenek dönüşüyle aynı Codex iş parçacığında Codex uygulama sunucusunun kendi uygulama ve Plugin
yeteneklerini kullanır. OpenClaw, Codex Plugin'lerini yapay `codex_plugin_*` OpenClaw
dinamik araçlarına dönüştürmez.

`codexPlugins` yalnızca yerel Codex düzeneğini seçen oturumları etkiler.
Yerleşik düzenek çalıştırmaları, normal OpenAI sağlayıcı çalıştırmaları, ACP
konuşma bağlamaları veya diğer düzenekler üzerinde etkisi yoktur.

Taşınmış asgari yapılandırma:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_destructive_actions: true,
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
              },
            },
          },
        },
      },
    },
  },
}
```

İş parçacığı uygulama yapılandırması, OpenClaw bir Codex düzenek oturumu oluşturduğunda
veya eski bir Codex iş parçacığı bağlamasını değiştirdiğinde hesaplanır; her dönüşte yeniden hesaplanmaz.
`codexPlugins` değiştirildikten sonra gelecekteki Codex düzenek oturumlarının güncellenmiş uygulama
kümesiyle başlaması için `/new`, `/reset` kullanın veya
Gateway'i yeniden başlatın.

Taşıma uygunluğu, uygulama envanteri, yıkıcı eylem politikası,
bilgi istemleri ve yerel Plugin tanılamaları için
[Yerel Codex Plugin'leri](/tr/plugins/codex-native-plugins) bölümüne bakın.

OpenAI tarafındaki uygulama ve Plugin erişimi, oturum açılmış Codex
hesabı ve Business ile Enterprise/Edu çalışma alanlarında çalışma alanı uygulama
denetimleri tarafından kontrol edilir. OpenAI'ın hesap ve çalışma alanı denetimlerine genel bakış için
[Codex'i ChatGPT planınızla kullanma](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan)
bölümüne bakın.

## Bilgisayar Kullanımı

Bilgisayar Kullanımı için ayrı bir kurulum kılavuzu vardır:
[Codex Bilgisayar Kullanımı](/tr/plugins/codex-computer-use).

Kısaca: OpenClaw masaüstü denetim uygulamasını paketlemez veya masaüstü
eylemlerini kendisi yürütmez. Codex uygulama sunucusunu hazırlar, `computer-use` MCP sunucusunun kullanılabilir olduğunu doğrular ve ardından Codex modundaki dönüşler sırasında yerel
MCP araç çağrılarının yönetimini Codex'e bırakır.

## Çalışma zamanı sınırları

Codex düzeneği yalnızca düşük seviyeli gömülü aracı yürütücüsünü değiştirir.

- OpenClaw dinamik araçları desteklenir. Codex, OpenClaw'dan
  bu araçları yürütmesini ister; dolayısıyla OpenClaw yürütme yolunda kalır.
- Codex'e özgü kabuk, yama, MCP ve yerel uygulama araçlarının sahibi Codex'tir.
  OpenClaw desteklenen aktarım aracılığıyla seçili yerel olayları gözlemleyebilir veya engelleyebilir,
  ancak yerel araç bağımsız değişkenlerini yeniden yazmaz.
- Yerel Compaction yönetimi Codex'e aittir. OpenClaw; kanal geçmişi, arama, `/new`, `/reset`
  ve gelecekteki model veya düzenek geçişleri için bir döküm yansısı tutar, ancak Codex Compaction'ını bir OpenClaw veya
  bağlam motoru özetleyicisiyle değiştirmez.
- Medya oluşturma, medya anlama, TTS, onaylar ve mesajlaşma aracı
  çıktıları eşleşen OpenClaw sağlayıcı/model ayarları üzerinden devam eder.
- `tool_result_persist`, Codex'e özgü araç sonuç kayıtlarına değil,
  OpenClaw'a ait döküm araç sonuçlarına uygulanır.

Kanca katmanları, desteklenen V1 yüzeyleri, yerel izin işleme, kuyruk
yönlendirme, Codex geri bildirimi yükleme mekanikleri ve Compaction ayrıntıları için
[Codex düzenek çalışma zamanı](/tr/plugins/codex-harness-runtime) bölümüne bakın.

## Sorun giderme

**Codex normal bir `/model` sağlayıcısı olarak görünmüyor:** yeni
yapılandırmalarda bu beklenir. Bir `openai/gpt-*` modeli seçin,
`plugins.entries.codex.enabled` seçeneğini etkinleştirin ve `plugins.allow` değerinin
`codex` değerini dışlayıp dışlamadığını denetleyin.

**OpenClaw Codex yerine yerleşik düzeneği kullanıyor:** etkin rotanın tam bir resmî HTTPS Platform Responses veya ChatGPT Responses rotası olduğunu,
yazılmış bir istek geçersiz kılması bulunmadığını ve Codex Plugin'inin kurulu ve etkin olduğunu doğrulayın.
Yalnızca `openai/gpt-*` ön eki yeterli değildir. Test sırasında kesin kanıt için
sağlayıcı veya model `agentRuntime.id: "codex"` değerini ayarlayın; zorlanan Codex, rota veya düzenek uyumsuz olduğunda
geri dönüş yapmak yerine başarısız olur.

**OpenAI Codex çalışma zamanı API anahtarı yoluna geri dönüyor:** modeli, çalışma zamanını, seçilen sağlayıcıyı ve
hatayı gösteren, hassas bilgileri çıkarılmış bir Gateway alıntısı toplayın.
Etkilenen çalışma arkadaşlarından bu salt okunur komutu OpenClaw ana makinelerinde çalıştırmalarını isteyin:

```bash
(
  pattern='openai/gpt-5\.[45]|openai[-]codex|agentRuntime(\.id)?|harnessRuntime|Runtime: OpenAI Codex|legacy OpenAI Codex prefix|resolveSelectedOpenAIRuntimeProvider|candidateProvider[": ]+openai|status[": ]+401|Incorrect API key|No API key|api-key path|API-key path|OAuth'

  if ls /tmp/openclaw/openclaw-*.log >/dev/null 2>&1; then
    grep -E -i -n "$pattern" /tmp/openclaw/openclaw-*.log 2>/dev/null || true
  else
    journalctl --user -u openclaw-gateway --since today --no-pager 2>/dev/null \
      | grep -E -i "$pattern" || true
  fi
) | sed -E \
    -e 's/(Authorization: Bearer )[A-Za-z0-9._~+\/-]+/\1[REDACTED]/Ig' \
    -e 's/(Bearer )[A-Za-z0-9._~+\/-]+/\1[REDACTED]/Ig' \
    -e 's/(api[_ -]?key[=: ]+)[^ ,}"]+/\1[REDACTED]/Ig' \
    -e 's/(OPENAI_API_KEY[=: ]+)[^ ,}"]+/\1[REDACTED]/Ig' \
    -e 's/sk-[A-Za-z0-9_-]{12,}/sk-[REDACTED]/g' \
    -e 's/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/[EMAIL-REDACTED]/g' \
  | tail -200
```

Yararlı alıntılar genellikle `openai/gpt-5.6-sol` veya `openai/gpt-5.6-luna`,
`Runtime: OpenAI Codex`, `agentRuntime.id` veya `harnessRuntime`,
`candidateProvider: "openai"` ve bir `401`, `Incorrect API key` veya
`No API key` sonucu içerir. Düzeltilmiş bir çalıştırmada düz bir OpenAI API anahtarı hatası
yerine OpenAI OAuth yolu görünmelidir.

**Eski Codex model başvuruları yapılandırması korunuyor:** `openclaw doctor --fix` komutunu çalıştırın.
Doctor, eski model başvurularını `openai/*` olarak yeniden yazar, güncelliğini yitirmiş oturum ve
tüm ajan çalışma zamanı sabitlemelerini kaldırır ve mevcut kimlik doğrulama profili geçersiz kılmalarını korur.

**Uygulama sunucusu reddediliyor:** paketle birlikte gelen `0.145.0` aracılığıyla
`0.143.0` kaynağından kararlı bir Codex uygulama sunucusu kullanın. Ön sürümler, derleme son eki
içeren sürümler ve daha yeni doğrulanmamış sürümler reddedilir; çünkü OpenClaw, oluşturulan şemaları
paketle birlikte gelen uygulama sunucusu sürümüne göre doğrular.

**`/codex status` bağlanamıyor:** `codex` Plugin'inin
etkin olduğunu, bir izin listesi yapılandırıldığında `plugins.allow` öğesinin bunu
içerdiğini ve tüm özel `appServer.command`, `url`, `authToken` veya
üstbilgilerin geçerli olduğunu denetleyin.

**Codex uygulama sunucusu çok fazla bellek kullanıyor:** önce iki işlemi
birbirinden ayırın. OpenClaw, yerel Codex uygulama sunucusunu ayrı bir Rust alt işlemi olarak çalıştırır.
`NODE_OPTIONS=--max-old-space-size=...` yalnızca Gateway'in Node.js V8
yığınını değiştirir; Codex'i sınırlamaz veya büyütmez. Yönetilen Gateway kurulumları zaten
uyarlanabilir bir V8 yığını seçer ve bunu artırmak Codex için daha az ana makine belleği bırakabilir. Gateway
baskısı için [Gateway bellek sorunlarını giderme](/tr/gateway/troubleshooting#gateway-exits-during-high-memory-use)
sayfasını kullanın; Codex alt işlemi için ana makine veya konteyner belleğini inceleyin.

Paketle birlikte gelen Codex'in yığın veya RSS sınırı ve yapılandırılabilir bir boşta yükten çıkarma
gecikmesi yoktur. Son istemcinin abonelikten çıkmasının ardından etkin olmayan bir iş parçacığı
30 dakikaya kadar yüklü kalabilir. Kaynakları kısıtlı ana makinelerde, Gateway yığınını artırmadan
önce yerel Codex alt ajanlarının paralel çalışma sayısını azaltın:

```json5
{
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            args: ["-c", "agents.max_threads=3", "app-server", "--listen", "stdio://"],
          },
        },
      },
    },
  },
}
```

Bu ayar, paketle birlikte gelen varsayılan Codex
çok ajanlı arka ucu için yerel alt iş parçacıklarını sınırlar. Codex çok ajanlı v2'yi açıkça etkinleştirirseniz
bunun yerine `features.multi_agent_v2.max_concurrent_threads_per_session=3` kullanın; v2
sınırı kök iş parçacığını içerir ve `agents.max_threads` ile birleştirilemez.
Codex'e daha fazla çalışma alanı sağlamak için ana makine, konteyner veya cgroup bellek
tahsisatını artırın. Bir işletim sistemi katı sınırı, geri basınç uygulamak yerine Codex'i sonlandırabilir.

**Model keşfi yavaş:** `plugins.entries.codex.config.discovery.timeoutMs` değerini
düşürün veya keşfi devre dışı bırakın.
Bkz. [Codex harness başvurusu](/tr/plugins/codex-harness-reference#model-discovery).

**WebSocket aktarımı hemen başarısız oluyor:** `appServer.url`,
`authToken`, üstbilgileri ve uzak uygulama sunucusunun aynı Codex
uygulama sunucusu protokol sürümünü kullandığını denetleyin. Codex WebSocket aktarımı deneysel
ve desteklenmeyen bir özellik olmaya devam etmektedir; yönetilen stdio veya yerel Unix denetim soketini tercih edin.

**Yerel kabuk veya yama araçları `Native hook relay
unavailable` ile engelleniyor:** Codex iş parçacığı hâlâ OpenClaw'un artık
kaydettirmediği yerel bir hook aktarma kimliğini kullanmaya çalışıyor. Bu, ACP arka ucu, sağlayıcı,
GitHub veya kabuk komutu hatası değil, yerel bir Codex hook aktarımı
sorunudur. Etkilenen sohbette `/new` veya `/reset` ile yeni bir oturum
başlatın, ardından zararsız bir komutu yeniden deneyin. Bu bir kez çalışır ancak sonraki yerel araç
çağrısı yeniden başarısız olursa `/new` öğesini yalnızca geçici bir çözüm olarak değerlendirin: eski
iş parçacıklarının bırakılması ve yerel hook kayıtlarının yeniden oluşturulması için Codex uygulama sunucusunu
veya OpenClaw Gateway'i yeniden başlattıktan sonra istemi yeni bir oturuma kopyalayın.

**Codex araç çağrıları çok fazla kısa ömürlü hook işlemi oluşturuyor:**
`plugins.entries.codex.config.appServer.loopDetectionPreToolUseRelay: false` ayarını yapın
ve Gateway'i yeniden başlatın. Bu, yalnızca OpenClaw döngü algılaması ve onun ilkesizlik işareti için kullanılan
Codex `PreToolUse` alt işlemini devre dışı bırakır. Gerekli
`before_tool_call` ve güvenilen araç ilkesi aktarımları etkin kalır.

**Codex dışı bir model yerleşik harness'ı kullanıyor:** sağlayıcı
veya model çalışma zamanı ilkesi bunu başka bir harness'a yönlendirmediği sürece bu beklenen davranıştır. Düz OpenAI dışı
sağlayıcı başvuruları, `auto` modunda normal sağlayıcı yollarında kalır.

**Computer Use kurulu ancak araçlar çalışmıyor:** yeni bir oturumdan
`/codex computer-use status` öğesini denetleyin. Bir araç
`Native hook relay unavailable` bildirirse yukarıdaki yerel hook aktarımı kurtarma yöntemini kullanın.
Bkz. [Codex Computer Use](/tr/plugins/codex-computer-use#troubleshooting).

## İlgili

- [Codex harness başvurusu](/tr/plugins/codex-harness-reference)
- [Codex harness çalışma zamanı](/tr/plugins/codex-harness-runtime)
- [Codex denetimi](/plugins/codex-supervision)
- [Yerel Codex Plugin'leri](/tr/plugins/codex-native-plugins)
- [Codex Computer Use](/tr/plugins/codex-computer-use)
- [Ajan çalışma zamanları](/tr/concepts/agent-runtimes)
- [Model sağlayıcıları](/tr/concepts/model-providers)
- [OpenAI sağlayıcısı](/tr/providers/openai)
- [OpenAI Codex yardımı](https://help.openai.com/en/collections/14937394-codex)
- [Ajan harness Plugin'leri](/tr/plugins/sdk-agent-harness)
- [Plugin hook'ları](/tr/plugins/hooks)
- [Tanılama dışa aktarımı](/tr/gateway/diagnostics)
- [Durum](/tr/cli/status)
- [Test](/tr/help/testing-live#live-codex-app-server-harness-smoke)
