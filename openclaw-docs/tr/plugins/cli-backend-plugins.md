---
read_when:
    - Yerel bir yapay zekâ CLI arka uç Plugin’i oluşturuyorsunuz
    - acme-cli/model gibi model referansları için bir arka uç kaydetmek istiyorsunuz
    - Üçüncü taraf bir CLI'yi OpenClaw'ın metin yedek çalıştırıcısıyla eşlemeniz gerekir
sidebarTitle: CLI backend plugins
summary: Yerel bir yapay zekâ CLI arka ucu kaydeden bir plugin oluşturun
title: CLI arka uç Pluginleri oluşturma
x-i18n:
    generated_at: "2026-07-26T23:50:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1923b0829b46a309e4b5a6cbbbfd3dcb76a1e14fe4106310d7a9fb37bca41d70
    source_path: plugins/cli-backend-plugins.md
    workflow: 16
---

CLI arka uç pluginleri, OpenClaw'un yerel bir AI CLI'ını metin çıkarımı
arka ucu olarak çağırmasını sağlar. Arka uç, model referanslarında bir sağlayıcı ön eki olarak görünür:

```text
acme-cli/acme-large
```

Üst sistem entegrasyonu zaten yerel bir komut olarak sunulduğunda, yerel oturum açma
durumunu CLI yönettiğinde veya API sağlayıcıları kullanılamadığında yedek seçenek
olarak bir CLI arka ucu kullanın.

<Info>
  Üst sistem hizmeti normal bir HTTP model API'si sunuyorsa bunun yerine bir
  [sağlayıcı plugini](/tr/plugins/sdk-provider-plugins) yazın. Üst sistem
  çalışma zamanı eksiksiz ajan oturumlarını, araç olaylarını, Compaction'ı veya arka plan
  görev durumunu yönetiyorsa bir [ajan çalıştırma altyapısı](/tr/plugins/sdk-agent-harness) kullanın.
</Info>

## Pluginin yönettiği alanlar

Bir CLI arka uç plugininin üç sözleşmesi vardır:

| Sözleşme             | Dosya                   | Amaç                                                   |
| -------------------- | ---------------------- | --------------------------------------------------------- |
| Paket giriş noktası        | `package.json`         | OpenClaw'u plugin çalışma zamanı modülüne yönlendirir              |
| Manifest sahipliği   | `openclaw.plugin.json` | Çalışma zamanı yüklenmeden önce arka uç kimliğini bildirir              |
| Çalışma zamanı kaydı | `index.ts`             | Komut varsayılanlarıyla `api.registerCliBackend(...)` çağrısını yapar |

Manifest, keşif meta verisidir: CLI'ı çalıştırmaz veya çalışma zamanı
davranışını kaydetmez. Çalışma zamanı davranışı, plugin giriş noktası
`api.registerCliBackend(...)` çağrısını yaptığında başlar.

## Asgari arka uç plugini

<Steps>
  <Step title="Paket meta verilerini oluşturun">
    ```json package.json
    {
      "name": "@acme/openclaw-acme-cli",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      },
      "dependencies": {
        "openclaw": "^2026.3.24"
      },
      "devDependencies": {
        "typescript": "^5.9.0"
      }
    }
    ```

    Yayımlanan paketler, derlenmiş JavaScript çalışma zamanı dosyalarını içermelidir. Kaynak
    giriş noktanız `./src/index.ts` ise derlenmiş JavaScript eşini gösteren
    `openclaw.runtimeExtensions` öğesini ekleyin. Bkz. [Giriş noktaları](/tr/plugins/sdk-entrypoints).

  </Step>

  <Step title="Arka uç sahipliğini bildirin">
    ```json openclaw.plugin.json
    {
      "id": "acme-cli",
      "name": "Acme CLI",
      "description": "Run Acme's local AI CLI through OpenClaw",
      "cliBackends": ["acme-cli"],
      "setup": {
        "cliBackends": ["acme-cli"],
        "requiresRuntime": false
      },
      "activation": {
        "onStartup": false
      },
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```

    `cliBackends`, çalışma zamanı sahiplik listesidir; model seçimi veya
    `agentRuntime.id`, `acme-cli` öğesinden bahsettiğinde OpenClaw'un plugini otomatik yüklemesini sağlar.

    `setup.cliBackends`, önce tanımlayıcı yaklaşımını kullanan kurulum yüzeyidir. Model
    keşfinin, ilk katılımın veya durumun plugin çalışma zamanını yüklemeden arka ucu
    tanıması gerektiğinde bunu ekleyin. Yalnızca bu statik tanımlayıcılar kurulum için
    yeterliyse `requiresRuntime: false` kullanın.

  </Step>

  <Step title="Arka ucu kaydedin">
    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      CLI_FRESH_WATCHDOG_DEFAULTS,
      CLI_RESUME_WATCHDOG_DEFAULTS,
      type CliBackendPlugin,
    } from "openclaw/plugin-sdk/cli-backend";

    function buildAcmeCliBackend(): CliBackendPlugin {
      return {
        id: "acme-cli",
        liveTest: {
          defaultModelRef: "acme-cli/acme-large",
          defaultImageProbe: false,
          defaultMcpProbe: false,
          docker: {
            npmPackage: "@acme/acme-cli",
            binaryName: "acme",
          },
        },
        config: {
          command: "acme",
          args: ["chat", "--output-format", "stream-json", "--prompt", "{prompt}"],
          resumeArgs: [
            "chat",
            "--resume",
            "{sessionId}",
            "--output-format",
            "stream-json",
            "--prompt",
            "{prompt}",
          ],
          output: "jsonl",
          resumeOutput: "jsonl",
          jsonlDialect: "gemini-stream-json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            large: "acme-large-2026",
            fast: "acme-fast-2026",
          },
          sessionArgs: ["--session", "{sessionId}"],
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptFileArg: "--system-file",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          imagePathScope: "workspace",
          reliability: {
            watchdog: {
              fresh: { ...CLI_FRESH_WATCHDOG_DEFAULTS },
              resume: { ...CLI_RESUME_WATCHDOG_DEFAULTS },
            },
          },
          serialize: true,
        },
      };
    }

    export default definePluginEntry({
      id: "acme-cli",
      name: "Acme CLI",
      description: "Run Acme's local AI CLI through OpenClaw",
      register(api) {
        api.registerCliBackend(buildAcmeCliBackend());
      },
    });
    ```

    Arka uç kimliği, manifestteki `cliBackends` girdisiyle eşleşmelidir. Kaydedilen
    bağdaştırıcı, yetkili plugin kodudur; OpenClaw yapılandırması arka ucu seçer
    ancak komut sözleşmesini yeniden yazmaz.

  </Step>
</Steps>

## Yapılandırma biçimi

`CliBackendConfig`, OpenClaw'un CLI'ı nasıl başlatıp ayrıştırması gerektiğini açıklar.
Yukarıdaki ayrıntılı örnek, paketle gelen `google-gemini-cli` bağdaştırıcısıyla aynı komut,
sürdürme, JSONL, model takma adı, oturum, görüntü ve gözetleyici alanlarını özellikle kullanır:

| Alan                                                     | Kullanım                                                                               |
| --------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `command`                                                 | İkili dosya adı veya mutlak komut yolu                                              |
| `args`                                                    | Yeni çalıştırmalar için temel argv                                                          |
| `resumeArgs`                                              | Sürdürülen oturumlar için alternatif argv; `{sessionId}` destekler                       |
| `output` / `resumeOutput`                                 | Ayrıştırıcı: `json`, `jsonl` veya `text`                                                |
| `jsonlDialect`                                            | JSONL olay lehçesi: `claude-stream-json` veya `gemini-stream-json`                 |
| `liveSession`                                             | Uzun ömürlü CLI işlem modu (`claude-stdio`)                                      |
| `input`                                                   | İstem aktarımı: `arg` veya `stdin`                                                |
| `maxPromptArgChars`                                       | Standart girdiye geri dönmeden önce `arg` modu için azami istem uzunluğu                     |
| `env` / `clearEnv`                                        | Eklenecek ilave ortam değişkenleri veya başlatmadan önce kaldırılacak adlar                         |
| `modelArg`                                                | Model kimliğinden önce kullanılan bayrak                                                     |
| `modelAliases`                                            | OpenClaw model kimliklerini CLI'a özgü kimliklerle eşleştirir                                          |
| `sessionArgs`                                             | `{sessionId}` kullanarak oturum kimliğinin nasıl geçirileceği                                      |
| `sessionMode`                                             | `always`, `existing` veya `none`                                                   |
| `sessionIdFields`                                         | OpenClaw'un CLI çıktısından okuduğu JSON alanları                                        |
| `systemPromptArg` / `systemPromptFileArg`                 | Sistem istemi aktarımı                                                           |
| `systemPromptFileConfigArg` / `systemPromptFileConfigKey` | Sistem istemi dosyası için yapılandırma geçersiz kılma aktarımı (örneğin `-c`)             |
| `systemPromptMode`                                        | `append` veya `replace`                                                             |
| `systemPromptWhen`                                        | `first`, `always` veya `never`                                                     |
| `imageArg` / `imageMode`                                  | Görüntü yolu bayrağı ve birden fazla görüntünün nasıl geçirileceği (`repeat` veya `list`)              |
| `imagePathScope`                                          | Hazırlanan görüntü dosyalarının devredilmeden önce bulunduğu yer: `temp` veya `workspace`               |
| `serialize`                                               | Aynı arka uçtaki çalıştırmaları sıralı tutar                                                    |
| `reseedFromRawTranscriptWhenUncompacted`                  | Güvenli oturum sıfırlamaları için Compaction öncesinde sınırlı ham dökümün yeniden kaynak olarak kullanılmasını etkinleştirir |
| `reliability.watchdog`                                    | Yeni ve sürdürülen çalıştırmalar için ayrı ayrı çıktısız kalma zaman aşımı ayarı                      |

CLI ile eşleşen en küçük statik yapılandırmayı tercih edin. Plugin geri çağrılarını
yalnızca gerçekten arka uca ait davranışlar için ekleyin.

## Gelişmiş arka uç kancaları

`CliBackendPlugin` ayrıca şunları tanımlayabilir:

| Kanca                               | Kullanım                                                                         |
| ---------------------------------- | --------------------------------------------------------------------------- |
| `normalizeConfig(config, context)` | Kayıtlı statik bağdaştırıcıyı çalışma zamanı bağlamıyla normalleştirir                |
| `resolveExecutionArgs(ctx)`        | Düşünme çabası veya yan soru yalıtımı gibi istek kapsamlı bayraklar ekler |
| `prepareExecution(ctx)`            | Başlatmadan önce geçici kimlik doğrulama, yapılandırma veya ortam köprüleri oluşturur         |
| `transformSystemPrompt(ctx)`       | CLI'a özgü son bir sistem istemi dönüşümü uygular                          |
| `textTransforms`                   | Çift yönlü istem/çıktı değiştirmeleri                                    |
| `defaultAuthProfileId`             | Belirli bir OpenClaw kimlik doğrulama profilini tercih eder                                     |
| `authEpochMode`                    | Kimlik doğrulama değişikliklerinin saklanan CLI oturumlarını nasıl geçersiz kılacağına karar verir                      |
| `nativeToolMode`                   | Yerel araçların bulunmadığını, her zaman açık olduğunu veya ana makine tarafından seçilebildiğini bildirir      |
| `toolAvailabilityEnforcement`      | Kesin araç sınırlarının argv'de mi yoksa yürütme hazırlığında mı uygulandığını bildirir   |
| `sideQuestionToolMode`             | `/btw` yan soruları için devre dışı bırakılan yerel araçları bildirir                     |
| `bundleMcp` / `bundleMcpMode`      | OpenClaw'un geri döngü MCP araç köprüsünü etkinleştirir                                |
| `ownsNativeCompaction`             | Arka uç kendi Compaction işlemini yönetir; OpenClaw bunu arka uca bırakır                           |
| `subscriptionAuthDispatch`         | Abonelik kimlik bilgileriyle etkinleştirilmiş gömülü çalıştırmalar bu arka uç üzerinden yürütülür |
| `runtimeArtifact`                  | Bir betik başlatıcısını eksiksiz paketlenmiş paket ağacına sınırlar                |

Bu kancaları sağlayıcının yönetiminde tutun. Bir arka uç kancası davranışı ifade
edebiliyorsa çekirdeğe CLI'a özgü dallar eklemeyin.

`prepareExecution(ctx)`, çalıştırma için seçilen etkin token
sınırı olan `ctx.contextTokenBudget` değerini alır. Yerel Compaction işlevini yöneten arka uçlar bu
bütçeyi CLI'larına özgü başlatma sözleşmesine eşleyebilir.

`runtimeArtifact`, plugin tarafından yönetilir. Buna yalnızca
canlı bir çıkarım turu doğrulanmış kurulum yetkisi oluşturduğunda veya yeniden doğruladığında başvurulur;
normal CLI çalıştırmaları bunu gerektirmez. Bu bildirime sahip olmayan bir arka uç,
doğrulanmış CLI kurulum yetkisi oluşturamaz. Bir `bundled-package-tree` bildirimi,
tam `package.json` sahibini adlandırır ve paket giriş noktasının
komut olmasını gerektirir. OpenClaw, iç içe bağımlılıklar dahil olmak üzere
sınırlandırılmış eksiksiz kurulu paket ağacının karmasını oluşturur ve yönlendiren sembolik bağlantılar,
bildirilen paketin dışındaki başlatıcılar, gerekli harici bağımlılık
bildirimleri, aşırı büyük ağaçlar ve bilinmeyen betikler için kapalı biçimde başarısız olur. Bunu yalnızca söz konusu
ağaç eksiksiz çıkarım uygulamasını içerdiğinde bildirin; isteğe bağlı araç entegrasyonları,
harici bir uygulama grafiğini güvenli hâle getirmez.

Aynı arka uç ayrıca kendi kendine yeten yerel bir yürütülebilir dosya sağlıyorsa bunun
standart taban adlarını `nativeExecutableNames` içinde listeleyin. Diğer yerel komutlar
doğrulanmamış olarak kalır.

`ctx.executionMode`, normal turlar için `"agent"`, geçici
`/btw` çağrıları için ise `"side-question"` değeridir. CLI'ın,
BTW için yerel araçları, oturum kalıcılığını veya sürdürme davranışını devre dışı bırakmak gibi
farklı tek seferlik bayraklara ihtiyaç duyduğu durumlarda bunu kullanın. Bir arka uç normalde
`nativeToolMode: "always-on"` değerine sahipse ancak yan soru argv'si bu araçları güvenilir biçimde
devre dışı bırakıyorsa `sideQuestionToolMode: "disabled"` değerini de ayarlayın; aksi hâlde BTW
araçsız bir CLI çalıştırması gerektirdiğinde OpenClaw kapalı biçimde başarısız olur.

`nativeToolMode: "selectable"` değerini yalnızca arka uç, her bir çalıştırma için
arka uca özgü yerel araçların tamamını devre dışı bırakabiliyorsa ayarlayın. Kısıtlı çalıştırmalar standart bir
sözleşme alır: `ctx.toolAvailability.native`, arka uca özgü yerel araçların tam listesidir ve
`ctx.toolAvailability.openClaw`, OpenClaw araç adlarının tam listesidir.
Ana makine, oluşturulan MCP yapılandırmasını ve izni bağımsız olarak bu
OpenClaw listesiyle sınırlar; plugin'ler bunu çekirdekte dönüştürmemeli veya aktarım ön ekleri eklememelidir.

Arka ucun bu sözleşmeyi nasıl uyguladığını bildirin:

- `toolAvailabilityEnforcement: "execution-args"`,
  `resolveExecutionArgs` gerektirir. Kanca, çakışan araç bayraklarını değiştirmeli, seçilen araçların dışında
  çalıştırma yapabilen özelleştirme yüzeylerini devre dışı bırakmalı ve
  hem yeni hem de sürdürülen çalıştırmalar için uygulayıcı argv döndürmelidir.
- `toolAvailabilityEnforcement: "prepare-execution"`,
  `prepareExecution` gerektirir. Kanca, çalıştırma başına kesin bir politika hazırlamalı ve
  `toolAvailabilityEnforced: true` döndürmelidir; onayın eksik olması kapalı biçimde başarısızlığa yol açar ve
  OpenClaw, hazırlanan kaynakları başlatmadan önce temizler.

Cron `toolsAllow` gibi çalışma zamanı sınırları, bu sözleşme oluşturulmadan önce
OpenClaw tarafından normalleştirilir ve gruplarına genişletilir. Yerel araçlar devre dışı bırakılır ve
eksiksiz olarak bildirilmiş bir uygulama yolu bulunmayan arka uç, yürütmeden önce başarısız olur.

`v2026.7.2-beta.1` ile `v2026.7.2-beta.3` arasındaki sürümlere göre oluşturulan plugin'ler,
kullanımdan kaldırılmış `ctx.toolAvailability.mcp` aktarım adı yansımasını hâlâ okuyabilir ve
seçilebilir bir arka uç `resolveExecutionArgs` uyguladığında
`toolAvailabilityEnforcement` değerini atlayabilir. OpenClaw, sağlanan bu beta yolunu
plugin paketinin gerekli `openclaw.build.openclawVersion` meta verilerinden tanır ve
`2026.8.x` serisi boyunca korur. Yeni ve güncellenen plugin'ler standart
`ctx.toolAvailability.openClaw` adlarını kullanmalı ve
`toolAvailabilityEnforcement: "execution-args"` değerini açıkça bildirmelidir; beta
uyumluluk yolunun bu dönemden sonra kaldırılması planlanmaktadır.

### `ownsNativeCompaction`: OpenClaw Compaction işlevini devre dışı bırakma

Arka ucunuz **kendi** transkriptini sıkıştıran bir ajan çalıştırıyorsa
`ownsNativeCompaction: true` değerini ayarlayın; böylece OpenClaw'ın koruyucu özetleyicisi
bu arka ucun oturumlarında hiçbir zaman çalışmaz. CLI Compaction yaşam döngüsü işlem yapmadan döner ve
tur devam eder. `claude-cli` bunu bildirir çünkü Claude Code,
herhangi bir donanım uç noktası olmadan dahili olarak sıkıştırma yapar. Codex gibi yerel donanım oturumları
ise bunun yerine kendi donanım Compaction uç noktalarına yönlendirilmeye devam eder.

**Yalnızca aşağıdakilerin tümü geçerliyse bunu bildirin**; aksi hâlde ertelenmiş,
bütçeyi aşan bir oturum bütçeyi aşmaya devam edebilir veya bayatlayabilir (OpenClaw artık
onu kurtarmaz):

- arka uç, penceresine yaklaştıkça kendi transkriptini güvenilir biçimde sıkıştırır veya sınırlar;
- sıkıştırılmış durumun turlar arasında korunması için sürdürülebilir bir oturumu kalıcılaştırır
  (örneğin `--resume` / `--session-id`);
- yerel donanım Compaction oturumu değildir; eşleşen `agentHarnessId`
  oturumları bunun yerine donanım uç noktasına yönlendirilir.

## MCP araç köprüsü

CLI arka uçları varsayılan olarak OpenClaw araçlarını almaz. CLI bir
MCP yapılandırmasını kullanabiliyorsa açıkça etkinleştirin:

```typescript
return {
  id: "acme-cli",
  bundleMcp: true,
  bundleMcpMode: "codex-config-overrides",
  config: {
    command: "acme",
    args: ["chat", "--json"],
    output: "json",
  },
};
```

Desteklenen köprü modları:

| Mod                      | Kullanım                                                           |
| ------------------------ | ------------------------------------------------------------------ |
| `claude-config-file`     | MCP yapılandırma dosyasını kabul eden CLI'lar                      |
| `codex-config-overrides` | argv üzerinde yapılandırma geçersiz kılmalarını kabul eden CLI'lar |
| `gemini-system-settings` | MCP ayarlarını sistem ayarları dizininden okuyan CLI'lar            |

Köprüyü yalnızca CLI gerçekten kullanabiliyorsa etkinleştirin. CLI'ın
devre dışı bırakılamayan kendi yerleşik araç katmanı varsa, bir çağıran yerel araçların
olmamasını gerektirdiğinde OpenClaw'ın kapalı biçimde başarısız olabilmesi için `nativeToolMode:
"always-on"` değerini ayarlayın.
Her çalıştırmada tüm yerel araçları devre dışı bırakabiliyorsa yukarıdaki
`resolveExecutionArgs` sözleşmesiyle `"selectable"` değerini kullanın.

## Arka ucu seçme

Kullanıcılar bağımsız bir arka ucu model başvurusu ön eki üzerinden seçer. Standart bir
`modelProvider` bildiren arka uç, bunun yerine söz konusu
sağlayıcı modelinin `agentRuntime.id` değeri üzerinden seçilebilir. Bağdaştırıcı mekanikleri plugin'de kalır:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "openai/gpt-5.6-sol",
        fallbacks: ["acme-cli/large"],
      },
    },
  },
}
```

Kimlik bilgilerini OpenClaw kimlik doğrulama profillerine veya plugin tarafından yönetilen yapılandırmaya yerleştirin.
Kayıtlı komutun Gateway hizmetinin `PATH` değerinde bulunduğundan emin olun; farklı bir
yola veya argv'ye ihtiyaç duyan dağıtımlar, plugin kaydını değiştirmeli veya sarmalamalıdır.

## Doğrulama

Paketle birlikte gelen plugin'ler için oluşturucu ve kurulum kaydı etrafına odaklı bir test ekleyin,
ardından plugin'in hedeflenmiş test hattını çalıştırın:

```bash
pnpm test extensions/acme-cli
```

Yerel veya kurulu plugin'ler için keşfi ve bir gerçek model çalıştırmasını doğrulayın:

```bash
openclaw plugins inspect acme-cli --runtime --json
openclaw agent --message "tam olarak şöyle yanıt ver: arka uç tamam" --model acme-cli/acme-large
```

Arka uç görüntüleri veya MCP'yi destekliyorsa bu yolları gerçek CLI ile kanıtlayan bir
canlı duman testi ekleyin. İstem, görüntü, MCP veya oturum sürdürme davranışı için
statik incelemeye güvenmeyin.

## Kontrol listesi

<Check>`package.json`, yayımlanan paketler için `openclaw.extensions` ve oluşturulmuş çalışma zamanı girişlerine sahiptir</Check>
<Check>`openclaw.plugin.json`, `cliBackends` ve bilinçli olarak seçilmiş `activation.onStartup` değerini bildirir</Check>
<Check>Kurulum/model keşfinin arka ucu başlatılmadan görmesi gerekiyorsa `setup.cliBackends` mevcuttur</Check>
<Check>`api.registerCliBackend(...)`, manifest ile aynı arka uç kimliğini kullanır</Check>
<Check>Arka uç model ön eki veya model kapsamlı `agentRuntime.id`, kaydı seçer</Check>
<Check>Oturum, sistem istemi, görüntü ve çıktı ayrıştırıcı ayarları gerçek CLI sözleşmesiyle eşleşir</Check>
<Check>Hedeflenmiş testler ve en az bir canlı CLI duman testi, arka uç yolunu kanıtlar</Check>

## İlgili

- [CLI arka uçları](/tr/gateway/cli-backends) - çalışma zamanı seçimi ve davranışı
- [Plugin oluşturma](/tr/plugins/building-plugins) - paket ve manifest temelleri
- [Plugin SDK'ya genel bakış](/tr/plugins/sdk-overview) - kayıt API'si başvurusu
- [Plugin manifesti](/tr/plugins/manifest) - `cliBackends` ve kurulum tanımlayıcıları
- [Ajan donanımı](/tr/plugins/sdk-agent-harness) - eksiksiz harici ajan çalışma zamanları
