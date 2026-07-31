---
read_when:
    - Yapılandırmayı etkileşimsiz olarak okumak veya düzenlemek istiyorsunuz
sidebarTitle: Config
summary: '`openclaw config` için CLI referansı (get/set/patch/unset/file/schema/validate)'
title: Yapılandırma
x-i18n:
    generated_at: "2026-07-26T23:35:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4c4f8edb19737070e421c9107f7da8886e5617d9a043d8647666505c7ac9638d
    source_path: cli/config.md
    workflow: 16
---

`openclaw.json` için etkileşimsiz yardımcılar: yola göre bir değeri getirin/ayarlayın/yamalayın/kaldırın, şemayı yazdırın, doğrulayın veya etkin dosya yolunu yazdırın. `openclaw configure` ile aynı yönlendirmeli sihirbazı açmak için `openclaw config` komutunu alt komut olmadan çalıştırın.

<Note>
`OPENCLAW_NIX_MODE=1` olduğunda OpenClaw, `openclaw.json` öğesini değişmez olarak kabul eder. Salt okunur komutlar (`config get`, `config file`, `config schema`, `config validate`) çalışmaya devam eder; yapılandırma yazıcıları işlemi reddeder. Bunun yerine kurulumun Nix kaynağını düzenleyin; birinci taraf nix-openclaw dağıtımı için [nix-openclaw Hızlı Başlangıç](https://github.com/openclaw/nix-openclaw#quick-start) kılavuzunu kullanın ve değerleri `programs.openclaw.config` veya `instances.<name>.config` altında ayarlayın.
</Note>

## Kök seçenekleri

<ParamField path="--section <section>" type="string">
  `openclaw config` komutunu alt komut olmadan çalıştırdığınızda kullanılabilen, yinelenebilir yönlendirmeli kurulum bölümü filtresi.
</ParamField>

Yönlendirmeli bölümler: `workspace`, `model`, `web`, `gateway`, `daemon`, `channels`, `plugins`, `skills`, `health`.

## Örnekler

```bash
openclaw config file
openclaw config --section model
openclaw config --section gateway --section daemon
openclaw config schema
openclaw config get browser.executablePath
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set browser.profiles.work.executablePath "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
openclaw config set agents.defaults.heartbeat.every "2h"
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
openclaw config set agents.defaults.models '{"openai/gpt-5.4":{}}' --strict-json --merge
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set secrets.providers.vaultfile --provider-source file --provider-path /etc/openclaw/secrets.json --provider-mode json
openclaw config patch --file ./openclaw.patch.json5 --dry-run
openclaw config unset plugins.entries.brave.config.webSearch.apiKey
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config validate
openclaw config validate --json
```

### Yollar

Nokta veya köşeli parantez gösterimi. zsh'nin `[0]` ifadesini glob ile genişletmemesi için kabuk örneklerinde köşeli parantezli yolları tırnak içine alın:

```bash
openclaw config get agents.defaults.workspace
openclaw config get agents.entries.main
openclaw config get agents.entries
openclaw config set 'agents.entries.work.tools.exec.node' "node-id-or-name"
```

### `config get`

Düzenlenmiş yapılandırma anlık görüntüsünden bir değer okur (gizli bilgiler hiçbir zaman yazdırılmaz). `--json` ham değeri JSON olarak yazdırır; aksi takdirde dizeler/sayılar/boole değerleri yalın, nesneler/diziler ise biçimlendirilmiş JSON olarak yazdırılır.

Yol eksik olduğunda `--json`, stdout'a `{ "error": "Config path not found: <path>" }` yazar ve 1 durum koduyla çıkar. `--json` olmadan tanılama iletisi stderr'de kalır.

```bash
openclaw config get browser.executablePath
openclaw config get agents.defaults.model --json
```

### `config file`

`OPENCLAW_CONFIG_PATH` veya varsayılan konumdan çözümlenen etkin yapılandırma dosyası yolunu yazdırır. Yol bir sembolik bağlantıyı değil, normal bir dosyayı belirtir; bkz. [Yazma güvenliği](#write-safety).

### `config schema`

`openclaw.json` için oluşturulan JSON şemasını stdout'a yazdırır.

<AccordionGroup>
  <Accordion title="İçerdikleri">
    - Geçerli kök yapılandırma şeması ve düzenleyici araçları için bir kök `$schema` dize alanı.
    - Control UI tarafından kullanılan `title` / `description` alan dokümantasyonu meta verileri.
    - İç içe nesne, joker karakter (`*`) ve dizi öğesi (`[]`) düğümleri, eşleşen alan dokümantasyonu bulunduğunda aynı `title` / `description` meta verilerini devralır.
    - `anyOf` / `oneOf` / `allOf` dalları da aynı dokümantasyon meta verilerini devralır.
    - Çalışma zamanı manifestleri yüklenebildiğinde en iyi çabayla sağlanan canlı plugin + kanal şeması meta verileri.
    - Geçerli yapılandırma geçersiz olduğunda bile temiz bir geri dönüş şeması.

  </Accordion>
  <Accordion title="İlgili çalışma zamanı RPC'si">
    `config.schema.lookup`; yüzeysel bir şema düğümü (`title`, `description`, `type`, `enum`, `const`, ortak sınırlar), eşleşen kullanıcı arayüzü ipucu meta verileri ve doğrudan alt öğe özetleriyle birlikte normalleştirilmiş tek bir yapılandırma yolu döndürür. Control UI veya özel istemcilerde yol kapsamlı ayrıntılı inceleme için kullanın.
  </Accordion>
</AccordionGroup>

```bash
openclaw config schema
openclaw config schema > openclaw.schema.json
```

### `config validate`

Gateway'i başlatmadan geçerli yapılandırmayı etkin şemaya göre doğrular.

```bash
openclaw config validate
openclaw config validate --json
```

<Note>
Doğrulama zaten başarısız oluyorsa `openclaw configure` veya `openclaw doctor --fix` ile başlayın. `openclaw chat`, geçersiz yapılandırma korumasını atlamaz.
</Note>

## Değerler

Değerler mümkün olduğunda JSON5 olarak ayrıştırılır; aksi takdirde ham dizeler olarak değerlendirilir. Dizeye geri dönüş olmadan standart JSON gerektirmek için `--strict-json` kullanın (bu durumda yorumlar, sondaki virgüller veya tırnaksız anahtarlar gibi yalnızca JSON5'e özgü söz dizimi reddedilir). `--json`, `config set` üzerindeki `--strict-json` için eski bir diğer addır.

```bash
openclaw config set agents.defaults.heartbeat.every "0m"
openclaw config set gateway.port 19001 --strict-json
openclaw config set channels.whatsapp.groups '["*"]' --strict-json
```

`config get <path> --json`, terminal için biçimlendirilmiş metin yerine ham değeri JSON olarak yazdırır.

Bir yazma işlemi `agents.defaults.model` veya ajan başına bir `agents.entries.*.model` değerini değiştirdiğinde OpenClaw, yazmadan önce değişen her birincil veya geri dönüş modelini yapılandırılmış sağlayıcı katalogları üzerinden çözümler. Bilinmeyen model başvuruları etkin yapılandırma değiştirilmeden reddedilir; kullanılabilir modelleri görmek için `openclaw models list` komutunu çalıştırın.

<Note>
Nesne ataması varsayılan olarak hedef yolun yerini alır. Genellikle kullanıcı tarafından eklenen girdileri barındıran korumalı yollar, `--replace` iletmediğiniz sürece mevcut girdileri kaldıracak değiştirme işlemlerini reddeder: `agents.defaults.models`, `agents.entries`, `models.providers`, `models.providers.<id>`, `models.providers.<id>.models`, `plugins.entries` ve `auth.profiles`.
</Note>

Bu eşlemelere girdiler eklerken `--merge` kullanın:

```bash
openclaw config set agents.defaults.models '{"openai/gpt-5.4":{}}' --strict-json --merge
openclaw config set models.providers.ollama.models '[{"id":"llama3.2","name":"Llama 3.2"}]' --strict-json --merge
```

Yalnızca sağlanan değerin kasıtlı olarak eksiksiz hedef değer hâline gelmesi gerektiğinde `--replace` kullanın.

## `config set` modları

<Tabs>
  <Tab title="Değer modu">
    ```bash
    openclaw config set <path> <value>
    ```
  </Tab>
  <Tab title="SecretRef oluşturucu modu">
    ```bash
    openclaw config set channels.discord.token \
      --ref-provider default \
      --ref-source env \
      --ref-id DISCORD_BOT_TOKEN
    ```
  </Tab>
  <Tab title="Sağlayıcı oluşturucu modu">
    Yalnızca `secrets.providers.<alias>` yollarını hedefler:

    ```bash
    openclaw config set secrets.providers.vault \
      --provider-source exec \
      --provider-command /usr/local/bin/openclaw-vault \
      --provider-arg read \
      --provider-arg openai/api-key \
      --provider-timeout-ms 5000
    ```

  </Tab>
  <Tab title="Toplu iş modu">
    ```bash
    openclaw config set --batch-json '[
      {
        "path": "secrets.providers.default",
        "provider": { "source": "env" }
      },
      {
        "path": "channels.discord.token",
        "ref": { "source": "env", "provider": "default", "id": "DISCORD_BOT_TOKEN" }
      }
    ]'
    ```

    ```bash
    openclaw config set --batch-file ./config-set.batch.json --dry-run
    ```

    Toplu iş dosyaları 8 MiB ile sınırlıdır.

  </Tab>
</Tabs>

<Warning>
SecretRef atamaları, desteklenmeyen ve çalışma zamanında değiştirilebilen yüzeylerde (örneğin `hooks.token`, `commands.ownerDisplaySecret`, Discord iş parçacığı bağlama webhook token'ları ve WhatsApp kimlik bilgileri JSON'u) reddedilir. Bkz. [SecretRef Kimlik Bilgisi Yüzeyi](/tr/reference/secretref-credential-surface).
</Warning>

Toplu iş ayrıştırması her zaman doğruluk kaynağı olarak toplu iş yükünü (`--batch-json`/`--batch-file`) kullanır; `--strict-json` / `--json` toplu iş ayrıştırma davranışını değiştirmez.

JSON yol/değer modu, doğrudan SecretRef'ler ve sağlayıcılar için de çalışır:

```bash
openclaw config set channels.discord.token \
  '{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}' \
  --strict-json

openclaw config set secrets.providers.vaultfile \
  '{"source":"file","path":"/etc/openclaw/secrets.json","mode":"json"}' \
  --strict-json
```

### Sağlayıcı oluşturucu bayrakları

Sağlayıcı oluşturucu hedefleri yol olarak `secrets.providers.<alias>` kullanmalıdır.

<AccordionGroup>
  <Accordion title="Ortak bayraklar">
    - `--provider-source <env|file|exec>`
    - `--provider-timeout-ms <ms>` (`file`, `exec`)

  </Accordion>
  <Accordion title="Ortam sağlayıcısı (--provider-source env)">
    - `--provider-allowlist <ENV_VAR>` (yinelenebilir)

  </Accordion>
  <Accordion title="Dosya sağlayıcısı (--provider-source file)">
    - `--provider-path <path>` (gerekli)
    - `--provider-mode <singleValue|json>`
    - `--provider-max-bytes <bytes>`
    - `--provider-allow-insecure-path`

  </Accordion>
  <Accordion title="Çalıştırma sağlayıcısı (--provider-source exec)">
    - `--provider-command <path>` (gerekli)
    - `--provider-arg <arg>` (yinelenebilir)
    - `--provider-no-output-timeout-ms <ms>`
    - `--provider-max-output-bytes <bytes>`
    - `--provider-json-only`
    - `--provider-env <KEY=VALUE>` (yinelenebilir)
    - `--provider-pass-env <ENV_VAR>` (yinelenebilir)
    - `--provider-trusted-dir <path>` (yinelenebilir)
    - `--provider-allow-insecure-path`
    - `--provider-allow-symlink-command`

  </Accordion>
</AccordionGroup>

Güçlendirilmiş çalıştırma sağlayıcısı örneği:

```bash
openclaw config set secrets.providers.vault \
  --provider-source exec \
  --provider-command /usr/local/bin/openclaw-vault \
  --provider-arg read \
  --provider-arg openai/api-key \
  --provider-json-only \
  --provider-pass-env VAULT_TOKEN \
  --provider-trusted-dir /usr/local/bin \
  --provider-timeout-ms 5000
```

## `config patch`

Yol tabanlı çok sayıda `config set` komutu çalıştırmak yerine yapılandırma biçiminde bir JSON5 yaması yapıştırın veya boru ile aktarın. Nesneler özyinelemeli olarak birleştirilir; diziler ve skaler değerler hedefin yerini alır; `null` hedef yolu siler.

```bash
openclaw config patch --file ./openclaw.patch.json5 --dry-run
openclaw config patch --file ./openclaw.patch.json5
```

Yama dosyaları 8 MiB ile sınırlıdır. Boru ile aktarılan `--stdin` yamaları 1 MiB ile sınırlıdır.

Uzak kurulum betikleri için stdin üzerinden bir yama aktarın:

```bash
ssh user@gateway-host 'openclaw config patch --stdin --dry-run' < ./openclaw.patch.json5
ssh user@gateway-host 'openclaw config patch --stdin' < ./openclaw.patch.json5
```

Örnek yama:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      groupPolicy: "open",
      requireMention: false,
    },
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
      dmPolicy: "disabled",
      dm: { enabled: false },
      groupPolicy: "allowlist",
    },
  },
  agents: {
    defaults: {
      model: { primary: "openai/gpt-5.6-sol" },
      models: {
        "openai/gpt-5.6-sol": { params: { fastMode: true } },
      },
    },
  },
}
```

Bir nesne veya dizinin özyinelemeli olarak yamalanmak yerine tam olarak sağlanan değer hâline gelmesi gerektiğinde `--replace-path <path>` kullanın:

```bash
openclaw config patch --file ./discord.patch.json5 --replace-path 'channels.discord.guilds["123"].channels'
```

`--dry-run`, yazma işlemi yapmadan şema ve SecretRef çözümlenebilirlik denetimlerini çalıştırır. Exec destekli SecretRef'ler deneme çalıştırması sırasında varsayılan olarak atlanır; deneme çalıştırmasının sağlayıcı komutlarını yürütmesini özellikle istediğinizde `--allow-exec` ekleyin.

## Deneme çalıştırması

`--dry-run`, `openclaw.json` dosyasına yazmadan değişiklikleri doğrular. `config set`, `config patch` ve `config unset` üzerinde kullanılabilir.

```bash
openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN \
  --dry-run \
  --json

openclaw config set channels.discord.token \
  --ref-provider vault \
  --ref-source exec \
  --ref-id discord/token \
  --dry-run \
  --allow-exec
```

<AccordionGroup>
  <Accordion title="Deneme çalıştırması davranışı">
    - Oluşturucu modu: değiştirilen referanslar/sağlayıcılar için SecretRef çözümlenebilirlik denetimlerini çalıştırır.
    - JSON modu (`--strict-json`, `--json` veya toplu mod): şema doğrulamasının yanı sıra SecretRef çözümlenebilirlik denetimlerini çalıştırır.
    - İlke doğrulaması, değişiklik sonrası yapılandırmanın tamamına uygulanır; dolayısıyla üst nesneye yapılan yazma işlemleri (örneğin `hooks` değerini nesne olarak ayarlamak) desteklenmeyen yüzey doğrulamasını atlayamaz.
    - Komut yan etkilerini önlemek için Exec SecretRef denetimleri varsayılan olarak atlanır; etkinleştirmek için `--allow-exec` iletin (bu, sağlayıcı komutlarını yürütebilir). `--allow-exec` yalnızca deneme çalıştırmasına özeldir ve `--dry-run` olmadan hata verir.

  </Accordion>
  <Accordion title="--dry-run --json alanları">
    - `ok`: deneme çalıştırmasının başarılı olup olmadığı
    - `operations`: değerlendirilen atama sayısı
    - `checks`: şema/çözümlenebilirlik denetimlerinin çalıştırılıp çalıştırılmadığı
    - `checks.resolvabilityComplete`: çözümlenebilirlik denetimlerinin tamamlanıp tamamlanmadığı (exec referansları atlandığında false)
    - `refsChecked`: deneme çalıştırması sırasında gerçekten çözümlenen referans sayısı
    - `skippedExecRefs`: `--allow-exec` ayarlanmadığı için atlanan exec referanslarının sayısı
    - `errors`: `ok=false` olduğunda yapılandırılmış eksik yol, şema veya çözümlenebilirlik hataları

  </Accordion>
</AccordionGroup>

### JSON çıktı biçimi

```json5
{
  ok: boolean,
  operations: number,
  configPath: string,
  inputModes: ["value" | "json" | "builder" | "unset", ...],
  checks: {
    schema: boolean,
    resolvability: boolean,
    resolvabilityComplete: boolean,
  },
  refsChecked: number,
  skippedExecRefs: number,
  errors?: [
    {
      kind: "missing-path" | "schema" | "resolvability" | "model",
      message: string,
      ref?: string, // çözümlenebilirlik hatalarında bulunur
    },
  ],
}
```

<Tabs>
  <Tab title="Başarı örneği">
    ```json
    {
      "ok": true,
      "operations": 1,
      "configPath": "~/.openclaw/openclaw.json",
      "inputModes": ["builder"],
      "checks": {
        "schema": false,
        "resolvability": true,
        "resolvabilityComplete": true
      },
      "refsChecked": 1,
      "skippedExecRefs": 0
    }
    ```
  </Tab>
  <Tab title="Hata örneği">
    ```json
    {
      "ok": false,
      "operations": 1,
      "configPath": "~/.openclaw/openclaw.json",
      "inputModes": ["builder"],
      "checks": {
        "schema": false,
        "resolvability": true,
        "resolvabilityComplete": true
      },
      "refsChecked": 1,
      "skippedExecRefs": 0,
      "errors": [
        {
          "kind": "resolvability",
          "message": "Hata: \"MISSING_TEST_SECRET\" ortam değişkeni ayarlanmamış.",
          "ref": "env:default:MISSING_TEST_SECRET"
        }
      ]
    }
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="Deneme çalıştırması başarısız olursa">
    - `config schema validation failed`: değişiklik sonrası yapılandırma biçiminiz geçersizdir; yol/değer veya sağlayıcı/referans nesnesi biçimini düzeltin.
    - `Config policy validation failed: unsupported SecretRef usage`: bu kimlik bilgisini yeniden düz metin/dize girdisine taşıyın; SecretRef'leri yalnızca desteklenen yüzeylerde tutun.
    - `SecretRef assignment(s) could not be resolved`: başvurulan sağlayıcı/referans şu anda çözümlenemiyor (eksik ortam değişkeni, geçersiz dosya işaretçisi, exec sağlayıcısı hatası veya sağlayıcı/kaynak uyuşmazlığı).
    - `model reference validation failed`: değiştirilen bir metin modeli birincil veya yedek modeli bilinmiyor; `openclaw models list` komutunu çalıştırıp kullanılabilir bir model seçin.
    - `Dry run note: skipped <n> exec SecretRef resolvability check(s)`: exec çözümlenebilirlik doğrulamasına ihtiyacınız varsa `--allow-exec` ile yeniden çalıştırın.
    - Toplu modda, yazma işleminden önce başarısız girdileri düzeltip `--dry-run` komutunu yeniden çalıştırın.

  </Accordion>
</AccordionGroup>

## Değişiklikleri uygulama

Başarıyla tamamlanan her `config set` / `config patch` / `config unset` işleminden sonra CLI, Gateway'in yeniden başlatılmasının gerekip gerekmediğini anlayabilmeniz için üç ipucundan birini yazdırır:

| İpucu                                              | Anlamı                                                  |
| --------------------------------------------------- | -------------------------------------- |
| `Restart the gateway to apply.`                     | Değiştirilen yolun tamamen yeniden başlatılması gerekir. |
| `Change will apply without restarting the gateway.` | Çalışırken yeniden yükleme bunu otomatik olarak algılar. |
| `No gateway restart needed.`                        | Çalışma zamanıyla ilgili hiçbir şey değişmedi.           |

CLI her plugin'in yeniden yükleme meta verilerinin yüklendiğini doğrulayamadığından, `plugins.entries` öğesine (veya herhangi bir alt yoluna) yapılan yazma işlemleri her zaman yeniden başlatma gerektirir.

## Yazma güvenliği

`openclaw config set` ve OpenClaw'a ait diğer yapılandırma yazıcıları, diske kaydetmeden önce değişiklik sonrası yapılandırmanın tamamını doğrular. Yeni yük şema doğrulamasında başarısız olursa veya yıkıcı bir üzerine yazma işlemi gibi görünürse etkin yapılandırmaya dokunulmaz ve reddedilen yük, `openclaw.json.rejected.*` olarak yanına kaydedilir.

OpenClaw'a ait yazma işlemleri JSON5'i standart JSON olarak yeniden serileştirir. Kaynak yorum içerdiğinde yazıcı, bunları kaldırmadan hemen önce uyarır; yorumları korumak önemliyse doğrudan bir düzenleyici kullanın.

<Warning>
Etkin yapılandırma yolu normal bir dosya olmalıdır. Sembolik bağlantılı `openclaw.json` düzenleri yazma işlemleri için desteklenmez; bunun yerine doğrudan gerçek dosyayı göstermek için `OPENCLAW_CONFIG_PATH` kullanın.
</Warning>

Küçük düzenlemeler için CLI yazma işlemlerini tercih edin:

```bash
openclaw config set gateway.reload.mode hybrid --dry-run
openclaw config set gateway.reload.mode hybrid
openclaw config validate
```

Bir yazma işlemi reddedilirse kaydedilen yükü inceleyin ve yapılandırma biçiminin tamamını düzeltin:

```bash
CONFIG="$(openclaw config file)"
ls -lt "$CONFIG".rejected.* 2>/dev/null | head
openclaw config validate
```

Doğrudan düzenleyiciyle yazmaya hâlâ izin verilir ancak çalışan Gateway, doğrulanana kadar bunları güvenilmeyen olarak değerlendirir. Geçersiz doğrudan düzenlemeler başlatma sırasında başarısız olur veya çalışırken yeniden yükleme tarafından atlanır; Gateway, `openclaw.json` dosyasını yeniden yazmaz. Önek eklenmiş/üzerine yazılmış yapılandırmayı onarmak veya bilinen son sağlam kopyayı geri yüklemek için `openclaw doctor --fix` komutunu çalıştırın. Bkz. [Gateway sorun giderme](/tr/gateway/troubleshooting#gateway-rejected-invalid-config).

Tüm dosyayı kurtarma yalnızca doctor onarımına ayrılmıştır. Plugin şeması değişiklikleri veya `minHostVersion` uyumsuzluğu; modeller, sağlayıcılar, kimlik doğrulama profilleri, kanallar, Gateway erişimi, araçlar, bellek, tarayıcı ya da cron yapılandırması gibi ilgisiz kullanıcı ayarlarını geri almak yerine açıkça hata vermeye devam eder.

## Onarım döngüsü

`openclaw config validate` başarılı olduktan sonra, her değişikliği aynı terminalden doğrularken yerleşik bir ajanın etkin yapılandırmayı belgelerle karşılaştırmasını sağlamak için yerel TUI'yi kullanın:

```bash
openclaw chat
```

TUI içinde, baştaki `!` gerçek bir yerel kabuk komutunu çalıştırır (oturum başına bir kez gösterilen onay isteminden sonra):

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

<Steps>
  <Step title="Belgelerle karşılaştırın">
    Ajandan mevcut yapılandırmanızı ilgili belge sayfasıyla karşılaştırmasını ve en küçük düzeltmeyi önermesini isteyin.
  </Step>
  <Step title="Hedefli düzenlemeleri uygulayın">
    `openclaw config set` veya `openclaw configure` ile hedefli düzenlemeler uygulayın.
  </Step>
  <Step title="Yeniden doğrulayın">
    Her değişiklikten sonra `openclaw config validate` komutunu yeniden çalıştırın.
  </Step>
  <Step title="Çalışma zamanı sorunları için Doctor">
    Doğrulama başarılı olduğu hâlde çalışma zamanı hâlâ sağlıksızsa geçiş ve onarım yardımı için `openclaw doctor` veya `openclaw doctor --fix` komutunu çalıştırın.
  </Step>
</Steps>

## İlgili

- [CLI başvurusu](/tr/cli)
- [Yapılandırma](/tr/gateway/configuration)
