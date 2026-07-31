---
read_when:
    - OpenClaw, Codex, ACP veya başka bir yerel aracı çalışma zamanı arasında seçim yapıyorsunuz
    - Durum veya yapılandırmadaki sağlayıcı/model/çalışma zamanı etiketleri kafa karıştırıyor
    - Yerel bir harness için destek eşdeğerliğini belgeliyorsunuz
summary: OpenClaw model sağlayıcılarını, modelleri, kanalları ve aracı çalışma zamanlarını nasıl ayırır
title: Ajan çalışma zamanları
x-i18n:
    generated_at: "2026-07-26T23:37:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 980d112946535df1566f2df4e3e71abacc2b073b51717c1e85fbb678691d39cb
    source_path: concepts/agent-runtimes.md
    workflow: 16
---

Bir **ajan çalışma zamanı**, hazırlanmış tek bir model döngüsünün sahibidir: istemi alır,
model çıktısını yönlendirir, yerel araç çağrılarını işler ve tamamlanan etkileşimi
OpenClaw'a döndürür.

Her ikisi de model yapılandırmasının yakınında göründüğünden çalışma zamanlarıyla
sağlayıcıları karıştırmak kolaydır. Bunlar farklı katmanlardır:

| Katman        | Örnekler                                     | Anlamı                                                              |
| ------------- | -------------------------------------------- | ------------------------------------------------------------------- |
| Sağlayıcı     | `anthropic`, `github-copilot`, `openai`      | OpenClaw'ın kimlik doğrulamasını nasıl yaptığı, modelleri nasıl keşfettiği ve model referanslarını nasıl adlandırdığı. |
| Model         | `claude-opus-4-6`, `gpt-5.6-sol`             | Ajan etkileşimi için seçilen model.                                 |
| Ajan çalışma zamanı | `claude-cli`, `codex`, `copilot`, `openclaw` | Hazırlanmış etkileşimi yürüten düşük seviyeli döngü veya arka uç.    |
| Kanal         | Discord, Slack, Telegram, WhatsApp           | Mesajların OpenClaw'a girip çıktığı yer.                            |

Bir **harness**, ajan çalışma zamanını sağlayan uygulamadır (kod
terimi). Örneğin, paketle gelen Codex harness'ı `codex` çalışma zamanını uygular.
Herkese açık yapılandırma, sağlayıcı veya model girdilerinde `agentRuntime.id` kullanır; ajan genelindeki
çalışma zamanı anahtarları eskidir ve yok sayılır. `openclaw doctor --fix`, eski
ajan genelindeki çalışma zamanı sabitlemelerini kaldırır ve eski çalışma zamanı model referanslarını, gerektiğinde
model kapsamlı çalışma zamanı ilkesiyle birlikte standart sağlayıcı/model referanslarına
yeniden yazar.

İki çalışma zamanı ailesi:

- **Gömülü harness'lar**, OpenClaw'ın hazırlanmış ajan döngüsü içinde çalışır: yerleşik
  `openclaw` çalışma zamanı ve `codex` ile
  `copilot` gibi kayıtlı Plugin harness'ları.
- **CLI arka uçları**, model referansını standart biçimde tutarken yerel bir CLI işlemi
  çalıştırır. Örneğin, model kapsamlı
  `agentRuntime.id: "claude-cli"` ile `anthropic/claude-opus-5`, "Anthropic modelini seç,
  Claude CLI aracılığıyla çalıştır" anlamına gelir. `claude-cli` gömülü bir harness kimliği değildir ve
  AgentHarness seçimine aktarılmamalıdır.

`copilot` harness'ı, GitHub Copilot CLI için ayrı ve isteğe bağlı bir harici Plugin harness'ıdır;
PI, Codex ve GitHub Copilot ajan çalışma zamanı arasındaki
kullanıcıya yönelik karar için [GitHub Copilot ajan çalışma zamanı](/tr/plugins/copilot) bölümüne bakın.

## Codex yüzeyleri

Birden çok yüzey Codex adını paylaşır:

| Yüzey                                           | OpenClaw adı/yapılandırması            | Yaptığı iş                                                                                                     |
| ------------------------------------------------ | ------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| Yerel Codex app-server çalışma zamanı            | `openai/*` model referansları         | OpenAI gömülü ajan etkileşimlerini Codex app-server üzerinden çalıştırır. Bu, olağan ChatGPT/Codex abonelik kurulumudur. |
| Codex OAuth kimlik doğrulama profilleri          | `openai` OAuth profilleri           | Codex app-server harness'ının kullandığı ChatGPT/Codex abonelik kimlik doğrulamasını saklar.                    |
| Codex ACP bağdaştırıcısı                         | `runtime: "acp"`, `agentId: "codex"` | Codex'i harici ACP/acpx kontrol düzlemi üzerinden çalıştırır. Yalnızca ACP/acpx açıkça istendiğinde kullanın.   |
| Yerel Codex sohbet denetimi komut kümesi         | `/codex ...`                         | Sohbetten Codex app-server iş parçacıklarını bağlar, sürdürür, yönlendirir, durdurur ve inceler.                |
| Ajan dışı yüzeyler için OpenAI Platform API rotası | `openai/*` ve API anahtarıyla kimlik doğrulama | Görüntüler, gömmeler, konuşma ve gerçek zamanlı özellikler gibi doğrudan OpenAI API'leri.                       |

Bu yüzeyler kasıtlı olarak birbirinden bağımsızdır. `codex` Plugin'ini
etkinleştirmek yerel app-server özelliklerini kullanılabilir hâle getirir; `openclaw doctor --fix`,
eski Codex rota onarımının ve geçersiz oturum sabitlemelerinin temizlenmesinin sahibidir. Bir ajan modeli için `openai/*`
seçmek artık, ajan dışı bir OpenAI API yüzeyi kullanılmadığı sürece,
"bunu Codex üzerinden çalıştır" anlamına gelir.

Yaygın ChatGPT/Codex abonelik kurulumu, kimlik doğrulama için Codex OAuth kullanır ancak
model referansını `openai/*` olarak tutar ve `codex` çalışma zamanını seçer:

```json5
{
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

Bu, OpenClaw'ın bir OpenAI model referansı seçtiği ve ardından Codex
app-server çalışma zamanından gömülü ajan etkileşimini çalıştırmasını istediği anlamına gelir. "API
faturalandırmasını kullan" anlamına gelmez; kanalın, model sağlayıcı kataloğunun veya
OpenClaw oturum deposunun Codex hâline geldiği anlamına da gelmez.

Paketle gelen `codex` Plugin'i etkinleştirildiğinde doğal dilde Codex denetimi için ACP yerine
yerel `/codex` komut yüzeyini (`/codex bind`, `/codex threads`, `/codex resume`, `/codex steer`,
`/codex stop`) kullanın. Codex için ACP'yi yalnızca kullanıcı açıkça ACP/acpx istediğinde veya ACP
bağdaştırıcı yolunu test ederken kullanın. Claude Code, Gemini CLI, OpenCode, Cursor ve benzeri harici
harness'lar ACP kullanmaya devam eder.

Karar ağacı:

1. **Codex bağlama/denetim/iş parçacığı/sürdürme/yönlendirme/durdurma** -> paketle gelen `codex` Plugin'i etkinleştirildiğinde yerel `/codex` komut yüzeyi.
2. **Gömülü çalışma zamanı olarak Codex** veya abonelik destekli olağan Codex ajan deneyimi -> `openai/<model>`.
3. **Bir OpenAI modeli için OpenClaw açıkça seçildiğinde** -> model referansını `openai/<model>` olarak tutun ve sağlayıcı/model çalışma zamanı ilkesini `agentRuntime.id: "openclaw"` olarak ayarlayın. Seçili bir `openai` OAuth profili, OpenClaw'ın Codex kimlik doğrulama aktarımı üzerinden dahili olarak yönlendirilir.
4. **Yapılandırmadaki eski Codex model referansları** -> `openclaw doctor --fix` ile `openai/<model>` biçimine onarın; eski model referansının bunu gerektirdiği yerlerde doctor, sağlayıcı/model kapsamlı `agentRuntime.id: "codex"` ekleyerek Codex kimlik doğrulama rotasını korur. Eski **`codex-cli/*`** model referansları aynı `openai/<model>` Codex app-server rotasına onarılır; OpenClaw artık paketle gelen bir Codex CLI arka ucunu korumaz.
5. **ACP, acpx veya Codex ACP bağdaştırıcısı açıkça istendiğinde** -> `runtime: "acp"` ve `agentId: "codex"`.
6. **Claude Code, Gemini CLI, OpenCode, Cursor, Droid veya başka bir harici harness** -> yerel alt ajan çalışma zamanı değil, ACP/acpx.

| Kastettiğiniz...                        | Kullanın...                                  |
| --------------------------------------- | -------------------------------------------- |
| Codex app-server sohbet/iş parçacığı denetimi | Paketle gelen `codex` Plugin'indeki `/codex ...` |
| Codex app-server gömülü ajan çalışma zamanı | `openai/*` ajan modeli referansları          |
| OpenAI Codex OAuth                      | `openai` OAuth profilleri                   |
| Claude Code veya başka bir harici harness | ACP/acpx                                   |

OpenAI ailesindeki ön ek ayrımı için [OpenAI](/tr/providers/openai) ve
[Model sağlayıcıları](/tr/concepts/model-providers) bölümlerine bakın. Codex çalışma zamanı destek
sözleşmesi için [Codex harness çalışma zamanı](/tr/plugins/codex-harness-runtime#v1-support-contract) bölümüne bakın.

## Çalışma zamanı sahipliği

Farklı çalışma zamanları, döngünün farklı bölümlerinin sahibidir:

| Yüzey                       | OpenClaw gömülü                                 | Codex app-server                                                            |
| --------------------------- | ---------------------------------------------- | --------------------------------------------------------------------------- |
| Model döngüsünün sahibi     | OpenClaw gömülü çalıştırıcısı aracılığıyla OpenClaw | Codex app-server                                                        |
| Standart iş parçacığı durumu | OpenClaw transkripti                           | Codex iş parçacığı ve OpenClaw transkript yansısı                            |
| OpenClaw dinamik araçları   | Yerel OpenClaw araç döngüsü                    | Codex bağdaştırıcısı üzerinden köprülenir                                    |
| Yerel kabuk ve dosya araçları | OpenClaw yolu                                | Desteklendiği yerlerde yerel kancalar üzerinden köprülenen Codex yerel araçları |
| Bağlam motoru               | Yerel OpenClaw bağlam derlemesi                | OpenClaw, derlenmiş bağlamı Codex etkileşimine aktarır                       |
| Compaction                  | OpenClaw veya seçili bağlam motoru             | OpenClaw bildirimleri ve yansı bakımıyla Codex yerel Compaction              |
| Kanal teslimi               | OpenClaw                                       | OpenClaw                                                                    |

Tasarım kuralı: Yüzeyin sahibi OpenClaw ise normal Plugin kancası
davranışı sağlayabilir. Yüzeyin sahibi yerel çalışma zamanı ise OpenClaw'ın çalışma zamanı
olaylarına veya yerel kancalara ihtiyacı vardır. Standart iş parçacığı durumunun sahibi
yerel çalışma zamanı ise OpenClaw, desteklenmeyen iç bileşenleri yeniden yazmak yerine
bağlamı yansıtır ve yansıtır.

## Çalışma zamanı seçimi

OpenClaw, sağlayıcı ve model çözümlemesinden sonra gömülü çalışma zamanını şu
sırayla çözümler:

1. **Model kapsamlı çalışma zamanı ilkesi** önceliklidir. Bu, yapılandırılmış bir sağlayıcı
   model girdisinde veya `agents.defaults.models["provider/model"].agentRuntime`
   / `agents.entries.*.models["provider/model"].agentRuntime` içinde bulunur. `agents.defaults.models["vllm/*"].agentRuntime` gibi bir sağlayıcı
   joker karakteri, tam model ilkesinden sonra uygulanır; böylece dinamik olarak keşfedilen sağlayıcı modelleri,
   model başına kesin istisnaları geçersiz kılmadan tek bir çalışma zamanını
   paylaşabilir.
2. **Sağlayıcı kapsamlı çalışma zamanı ilkesi**: `models.providers.<provider>.agentRuntime`.
3. **`auto` modu**: Kayıtlı Plugin çalışma zamanları desteklenen sağlayıcı/model çiftlerini üstlenebilir.
4. `auto` modunda hiçbir şey etkileşimi üstlenmezse OpenClaw,
   uyumluluk çalışma zamanı olarak `openclaw` seçeneğine geri döner. Çalıştırmanın
   katı olması gerektiğinde açık bir çalışma zamanı kimliği kullanın.

Tüm oturum ve tüm ajan kapsamındaki çalışma zamanı sabitlemeleri yok sayılır: `OPENCLAW_AGENT_RUNTIME`,
oturum `agentHarnessId`/`agentRuntimeOverride` durumu, `agents.defaults.agentRuntime`
ve `agents.entries.*.agentRuntime`. Geçersiz tüm ajan kapsamındaki çalışma zamanı yapılandırmasını kaldırmak ve
amacın korunabildiği yerlerde eski çalışma zamanı model referanslarını dönüştürmek için `openclaw doctor --fix` çalıştırın.

Açık sağlayıcı/model Plugin çalışma zamanları kapalı durumda hata verir: Bir sağlayıcı veya modeldeki `agentRuntime.id: "codex"`,
Codex ya da açık bir seçim/çalışma zamanı hatası anlamına gelir; hiçbir zaman
sessizce OpenClaw'a geri yönlendirilmez. Yalnızca `auto`, eşleşmeyen bir
etkileşimi OpenClaw'a yönlendirebilir.

CLI arka uç diğer adları, gömülü harness kimliklerinden farklıdır. Tercih edilen Claude CLI biçimi:

```json5
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-5",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
      },
    },
  },
}
```

`claude-cli/claude-opus-4-7` gibi eski referanslar uyumluluk için desteklenmeye devam eder,
ancak yeni yapılandırma sağlayıcı/modeli standart biçimde tutmalı ve
yürütme arka ucunu sağlayıcı/model çalışma zamanı ilkesine yerleştirmelidir.

Eski `codex-cli/*` referansları farklıdır: doctor, Codex CLI arka ucunu
korumak yerine Codex app-server harness'ı üzerinden çalışmaları için bunları `openai/*` biçimine taşır.

`auto` modu çoğu sağlayıcı için kasıtlı olarak tutucudur. OpenAI ajan
modelleri istisnadır: Ayarlanmamış çalışma zamanı ve `auto`, Codex
harness'ına çözümlenir. Açık OpenClaw çalışma zamanı yapılandırması, `openai/*` ajan
etkileşimleri için isteğe bağlı bir uyumluluk rotası olmaya devam eder; seçili bir `openai` OAuth
profiliyle eşleştirildiğinde OpenClaw, herkese açık model referansını `openai/*` olarak
tutarken bu yolu Codex kimlik doğrulama aktarımı üzerinden dahili olarak yönlendirir. Geçersiz OpenAI
çalışma zamanı oturum sabitlemeleri, çalışma zamanı seçimi tarafından yok sayılır ve
`openclaw doctor --fix` ile temizlenebilir.

If `openclaw doctor`, eski Codex model referansları yapılandırmada kalırken `codex` plugininin etkin olduğu konusunda uyarırsa bunu eski rota durumu olarak değerlendirin ve Codex çalışma zamanı ile `openai/*` biçimine yeniden yazmak için
`openclaw doctor --fix` komutunu çalıştırın.

## GitHub Copilot ajan çalışma zamanı

Harici `@openclaw/copilot` plugini, GitHub Copilot CLI (`@github/copilot-sdk`) tarafından desteklenen ve isteğe bağlı etkinleştirilen bir `copilot` çalışma zamanı kaydeder. Standart abonelik `github-copilot` sağlayıcısını üstlenir ve `auto` tarafından **asla** seçilmez. `agentRuntime.id` aracılığıyla model veya sağlayıcı bazında etkinleştirin:

```json5
{
  agents: {
    defaults: {
      model: "github-copilot/gpt-5.5",
      models: {
        "github-copilot/gpt-5.5": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
}
```

Harness; sağlayıcısını, çalışma zamanını, CLI oturum anahtarını ve kimlik doğrulama profili önekini `extensions/copilot/doctor-contract-api.ts` içinde üstlenir; `openclaw doctor` bunu otomatik olarak yükler. Yapılandırma, kimlik doğrulama, transkript yansıtma, Compaction, bildirimsel doctor sözleşmesi ve daha geniş kapsamlı PI, Codex ve Copilot SDK kararı için [GitHub Copilot ajan çalışma zamanı](/tr/plugins/copilot) bölümüne bakın.

## Uyumluluk sözleşmesi

Bir çalışma zamanı OpenClaw olmadığında, belgelerinde hangi OpenClaw yüzeylerini desteklediği belirtilmelidir:

| Soru                                   | Neden önemlidir                                                                                                   |
| -------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Model döngüsünün sahibi kimdir?        | Yeniden denemelerin, araç devamının ve nihai yanıt kararlarının nerede gerçekleştiğini belirler.                  |
| Standart iş parçacığı geçmişinin sahibi kimdir? | OpenClaw'ın geçmişi düzenleyip düzenleyemeyeceğini veya yalnızca yansıtıp yansıtamayacağını belirler.             |
| OpenClaw dinamik araçları çalışıyor mu? | Mesajlaşma, oturumlar, Cron ve OpenClaw'a ait araçlar buna dayanır.                                               |
| Dinamik araç kancaları çalışıyor mu?   | Pluginler, OpenClaw'a ait araçların çevresinde `before_tool_call`, `after_tool_call` ve ara yazılım bekler.       |
| Yerel araç kancaları çalışıyor mu?     | Kabuk, yama ve çalışma zamanına ait araçlar, politika ve gözlem için yerel kanca desteğine ihtiyaç duyar.          |
| Bağlam motoru yaşam döngüsü çalışıyor mu? | Bellek ve bağlam pluginleri; derleme, alım, tur sonrası ve Compaction yaşam döngüsüne bağlıdır.                    |
| Hangi Compaction verileri sunulur?     | Bazı pluginler yalnızca bildirimlere, diğerleri ise tutulan/atılan meta verilere ihtiyaç duyar.                   |
| Bilerek desteklenmeyenler nelerdir?    | Yerel çalışma zamanı daha fazla durumun sahibiyken kullanıcılar OpenClaw ile eşdeğerlik varsaymamalıdır.           |

Codex çalışma zamanı destek sözleşmesi
[Codex harness çalışma zamanı](/tr/plugins/codex-harness-runtime#v1-support-contract) bölümünde belgelenmiştir.

## Durum etiketleri

Durum çıktısı hem `Execution` hem de `Runtime` etiketlerini gösterebilir. Bunları sağlayıcı adları olarak değil, tanılama bilgileri olarak değerlendirin:

- `openai/gpt-5.6-sol` gibi bir model referansı, seçili sağlayıcı/modeldir.
- `codex` gibi bir çalışma zamanı kimliği, turu yürüten döngüdür.
- Telegram veya Discord gibi bir kanal etiketi, konuşmanın gerçekleştiği yerdir.

Bir çalıştırmada beklenmeyen bir çalışma zamanı gösteriliyorsa önce seçili sağlayıcı/model çalışma zamanı politikasını inceleyin. Eski oturum çalışma zamanı sabitlemeleri artık yönlendirmeyi belirlemez.

## İlgili

- [Codex harness](/tr/plugins/codex-harness)
- [Codex harness çalışma zamanı](/tr/plugins/codex-harness-runtime)
- [GitHub Copilot ajan çalışma zamanı](/tr/plugins/copilot)
- [OpenAI](/tr/providers/openai)
- [Ajan harness pluginleri](/tr/plugins/sdk-agent-harness)
- [Ajan döngüsü](/tr/concepts/agent-loop)
- [Modeller](/tr/concepts/models)
- [Durum](/tr/cli/status)
