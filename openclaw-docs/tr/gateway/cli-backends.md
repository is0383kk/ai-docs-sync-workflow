---
read_when:
    - API sağlayıcıları başarısız olduğunda güvenilir bir yedek çözüm istiyorsunuz
    - Yerel AI CLI'larını çalıştırıyorsunuz ve bunları yeniden kullanmak istiyorsunuz
    - CLI arka ucunun araç erişimi için MCP geri döngü köprüsünü anlamak istiyorsunuz
summary: 'CLI arka uçları: isteğe bağlı MCP araç köprüsüyle yerel AI CLI geri dönüşü'
title: CLI arka uçları
x-i18n:
    generated_at: "2026-07-26T23:17:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ce0427c587bf2a1e0a2ff24b5e76952eecae059e6f900af777b897b2d8d4210
    source_path: gateway/cli-backends.md
    workflow: 16
---

OpenClaw, API sağlayıcıları çalışmadığında, hız sınırına takıldığında veya hatalı davrandığında yalnızca metin tabanlı bir geri dönüş olarak yerel bir AI CLI çalıştırabilir. Bu özellik kasıtlı olarak ölçülü tasarlanmıştır:

- OpenClaw araçları doğrudan enjekte edilmez, ancak `bundleMcp: true` içeren bir arka uç, geri döngü MCP köprüsü üzerinden gateway araçlarını alabilir.
- Destekleyen CLI'lar için JSONL akışı.
- Oturumlar desteklenir; böylece takip eden etkileşimler tutarlı kalır.
- CLI görüntü yollarını kabul ediyorsa görüntüler iletilir.

Bunu birincil yol olarak değil, "her zaman çalışan" metin yanıtları için bir güvenlik ağı olarak kullanın. ACP oturum denetimleri, arka plan görevleri, iş parçacığı/konuşma bağlama ve kalıcı harici kodlama oturumları içeren eksiksiz bir çalıştırma ortamı için bunun yerine [ACP Aracılarını](/tr/tools/acp-agents) kullanın; CLI arka uçları ACP değildir.

<Tip>
  Yeni bir arka uç Plugin'i mi oluşturuyorsunuz? [CLI arka uç Plugin'lerine](/tr/plugins/cli-backend-plugins) bakın. Bu sayfa, önceden kaydedilmiş bir arka ucun yapılandırılmasını ve çalıştırılmasını ele alır.
</Tip>

## Hızlı başlangıç

Paketle gelen Anthropic Plugin'i varsayılan bir `claude-cli` arka ucu kaydeder; bu nedenle Claude Code'un kurulu olması ve oturum açılmış olması dışında herhangi bir yapılandırma olmadan çalışır:

```bash
openclaw agent --agent main --message "hi" --model claude-cli/claude-sonnet-4-6
```

Açık bir aracı listesi yapılandırılmadığında varsayılan aracı kimliği `main`'dir; aksi durumda bunu kendi aracı kimliğinizle değiştirin.

Gateway hizmetinin `PATH` üzerinde CLI'ı bulabilmesi gerekir. Bir dağıtım standart dışı bir yürütülebilir dosya yolu veya bağımsız değişkenler gerektiriyorsa başlatma mekaniklerini `openclaw.json` içine koymak yerine bu bağdaştırıcıyı bir [CLI arka uç Plugin'inde](/tr/plugins/cli-backend-plugins) kaydedin.

Model seçimi veya model kapsamlı bir `agentRuntime.id` kendi arka ucuna başvurduğunda OpenClaw, sahip olan paketlenmiş Plugin'i otomatik olarak yükler.

## Geri dönüş olarak kullanma

CLI arka ucunu geri dönüş listenize ekleyin; böylece yalnızca birincil modeller başarısız olduğunda çalışır:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["claude-cli/claude-sonnet-4-6"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "claude-cli/claude-sonnet-4-6": {},
      },
    },
  },
}
```

Yapılandırılmış geri dönüşler, `agents.defaults.modelPolicy.allow` içinde olmasalar bile birincil sağlayıcı başarısız olduğunda (kimlik doğrulama, hız sınırları, zaman aşımı) kullanılabilir durumda kalır. Kullanıcıların bir CLI arka uç modelini ayrıca `/model`, oturum geçersiz kılması veya `--model` üzerinden doğrudan seçebilmesi gerekiyorsa modeli yalnızca o zaman bu politikaya ekleyin. `agents.defaults.models` yalnızca model başına takma adları, parametreleri ve meta verileri yönetir.

## Yapılandırma

Kullanıcılar model ve çalışma zamanı politikası üzerinden kayıtlı bir arka uç seçer. Model başvurusunu standart biçimde tutun ve her model için CLI çalışma zamanını seçin:

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

Kimlik bilgileri OpenClaw kimlik doğrulama profillerinde veya sahibi olan Plugin'in yapılandırmasında kalır. Komut, argv, ortam, ayrıştırma, oturum, görüntü ve gözetim mekanikleri, `api.registerCliBackend(...)` ile kaydedilen Plugin kodudur.

## Nasıl çalışır?

1. Sağlayıcı ön ekine göre bir arka uç seçer (`claude-cli/...`).
2. Aynı OpenClaw istemini ve çalışma alanı bağlamını kullanarak bir sistem istemi oluşturur.
3. Geçmişin tutarlı kalması için CLI'ı bir oturum kimliğiyle (destekleniyorsa) yürütür. Paketle gelen `claude-cli` arka ucu, her OpenClaw oturumu için bir Claude stdio sürecini canlı tutar ve takip eden etkileşimleri stream-json stdin üzerinden gönderir.
4. Çıktıyı (JSON veya düz metin) ayrıştırır ve son metni döndürür.
5. Takip eden etkileşimlerin aynı CLI oturumunu yeniden kullanması için oturum kimliklerini arka uç başına kalıcı olarak saklar.

## Zaman aşımları ve uzun süreli işler

CLI arka uçlarının birbirinden bağımsız iki sınırı vardır:

- `agents.defaults.timeoutSeconds` aracı etkileşiminin tamamını sınırlar. Normal Gateway etkileşimleri varsayılan 48 saatlik süreyi devralır; `0` etkileşim bütçesini sınırsız yapar. `600` gibi saklanan bir geçersiz kılma bu varsayılanın yerini alır.
- CLI çıktısızlık gözetimi, sessiz kalan bir alt süreci durdurur. Her arka uç Plugin'i ayrı yeni/devam profillerini yönetir ve genel etkileşim bütçesi sınırsız olduğunda bile gözetim etkin kalır.

48 saatlik varsayılana dönmek için kısa genel zaman aşımı geçersiz kılmasını kaldırın veya 12 saat gibi açık bir bütçe belirleyin:

```bash
# 48 saatlik varsayılana dönün:
openclaw config unset agents.defaults.timeoutSeconds

# Ya da açık bir 12 saatlik sınır seçin:
openclaw config set agents.defaults.timeoutSeconds 43200
```

Bir CLI içinde başlatılan arka plan işi yine de o CLI alt sürecinin parçasıdır. Üst etkileşim genel sınırına ulaşırsa OpenClaw, alt süreci ve CLI içindeki arka plan görevlerini birlikte durdurur. Kalıcı uzun işler için ayrılmış bir OpenClaw [alt aracısı](/tr/tools/subagents) veya [ACP aracısı](/tr/tools/acp-agents) kullanın; ayrılmış alt aracıların varsayılan olarak çalışma zaman aşımı yoktur.

`openclaw agent` komutunun ayrıca kendi istek son tarihi vardır. 600 saniyelik geri dönüş varsayılanı normal Gateway etkileşimlerine değil, bu komut çağrısına uygulanır; [`openclaw agent`](/tr/cli/agent) sayfasına bakın.

### Claude CLI'a özgü ayrıntılar

Paketle gelen `claude-cli` arka ucu, Claude Code'un yerel beceri çözümleyicisini tercih eder. Geçerli beceri anlık görüntüsünde somutlaştırılmış yola sahip en az bir seçili beceri bulunduğunda OpenClaw, `--plugin-dir` üzerinden geçici bir Claude Code Plugin'i iletir ve yinelenen OpenClaw beceri kataloğunu eklenen sistem isteminden çıkarır. Somutlaştırılmış bir Plugin becerisi yoksa OpenClaw, istem kataloğunu geri dönüş olarak tutar. Beceri ortamı/API anahtarı geçersiz kılmaları, çalışma sırasında alt süreç ortamına uygulanmaya devam eder.

Claude CLI'ın kendi etkileşimsiz izin modu vardır; OpenClaw, Claude'a özgü yapılandırma eklemek yerine bunu mevcut yürütme politikasıyla eşler. OpenClaw tarafından yönetilen canlı Claude oturumlarında etkin yürütme politikası belirleyicidir: YOLO (`tools.exec.mode: "full"`) normalde Claude'u `--permission-mode bypassPermissions` ile başlatırken kısıtlayıcı bir politika `--permission-mode default` ile başlatır. Kök kullanıcı tarafından çalıştırılan gateway'ler de Claude Code kök kullanıcı için atlama modunu reddettiğinden `default` kullanır. Aracı başına `agents.entries.*.tools.exec` ayarları, ilgili aracı için genel `tools.exec` ayarlarını geçersiz kılar. Anthropic Plugin'i, Claude'un izin bayraklarını etkin politikayla ve ana makine kısıtlamasıyla eşleşecek biçimde normalleştirir.

Kısıtlayıcı bir politika altında Claude, yerel veya uzantı araçlarından birini (kendi Bash, WebFetch veya Claude in Chrome tarayıcı araçları) kullanmadan önce stdio üzerinden OpenClaw'a sorar. Etkin yürütme sorma ayarı `on-miss` veya `always` olduğunda OpenClaw her isteği etkileşimli onay olarak oturumun kanalına aktarır: **Allow once** tek çağrıya izin verir, **Allow always** canlı Claude oturumunun geri kalanı boyunca ilgili araç adına izin verir (yalnızca bellekte, hiçbir zaman kalıcı olarak saklanmaz) ve **Deny**, zaman aşımı veya erişilemeyen onay rotası çağrıyı reddeder. Hiçbir zaman istem göstermeyen politikalar eski davranışlarını korur: `security: "deny"` her isteği reddeder ve tam güvenlikten daha düşük bir düzeydeki (`allowlist` yürütme modu) `off` sorma ayarı sormadan reddeder.

### Claude tarayıcı araçları ve 1Password ile oturum açma

Claude Code, kimlik bilgilerini otomatik doldurmak için [1Password for Claude](/tr/gateway/1password#browser-sign-in-with-1password-for-claude) dahil olmak üzere [Claude in Chrome extension](https://code.claude.com/docs/en/chrome) aracılığıyla bir Chrome tarayıcısını yönetebilir. Paketle gelen arka uç bunu etkinleştirmez; `claude-stream-json` lehçeli bir arka ucun başlatma bağımsız değişkenlerine `--chrome` ekleyen bir [CLI arka uç Plugin'i](/tr/plugins/cli-backend-plugins) kaydedin. OpenClaw, normal çalışmalarda yapılandırılmış bir `--chrome` değerini korur ve yan sorular gibi kısıtlanmış araç politikasına sahip çalışmalarda her zaman `--no-chrome` değerini zorunlu kılar. Chrome penceresi, uzantı ve tüm 1Password onay istemleri gateway ana makinesinde bulunduğundan kimlik bilgisi kullanımını onaylamak için birinin bu makinenin başında olması gerekir.

Arka uç ayrıca OpenClaw `/think` düzeylerini Claude Code'un yerel `--effort` bayrağıyla eşler: `minimal`/`low` -> `low`, `medium` -> `medium` ve `high`/`xhigh`/`max` doğrudan geçirilir. Böylece desteklenen Fable 5 çaba düzeyleri, abonelik destekli Claude CLI ve API anahtarı rotalarında aynı kalır. `adaptive`, yapılandırılmış `--effort` bayraklarını kaldırır ve yerine bir değer sağlamaz; böylece Claude Code etkin çabayı kendi ortamından, ayarlarından ve model varsayılanlarından çözümler. `/think` oluşturulan CLI'ı etkilemeden önce diğer CLI arka uçlarının sahibi olan Plugin'in eşdeğer bir argv eşleyicisi bildirmesi gerekir.

OpenClaw'ın `claude-cli` kullanabilmesi için öncelikle Claude Code'un aynı ana makinede oturum açmış olması gerekir:

```bash
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

Docker kurulumlarında Claude Code'un yalnızca ana makinede değil, kalıcı kapsayıcı ana dizininin içinde de kurulmuş ve oturum açılmış olması gerekir; [Docker'da Claude CLI arka ucu](/tr/install/docker#claude-cli-backend-in-docker) sayfasına bakın.

Gateway hizmetinin `PATH` üzerinde `claude` değerini çözümleyebilmesi gerekir. Standart dışı bir yol için küçük bir sarmalayıcı arka uç Plugin'i kaydedin.

## Oturumlar

- CLI oturumları destekliyorsa bir `{sessionId}` yer tutucusuyla `sessionArgs` değerini ayarlayın (örneğin `["--session-id", "{sessionId}"]`).
- CLI farklı bayraklara sahip bir devam alt komutu kullanıyorsa `resumeArgs` değerini (devam ederken `args` değerinin yerini alır) ve JSON olmayan devamlar için isteğe bağlı olarak `resumeOutput` değerini ayarlayın.
- `sessionMode`:
  - `always`: her zaman bir oturum kimliği gönderir (saklanmış bir kimlik yoksa yeni UUID).
  - `existing`: yalnızca önceden saklanmış bir oturum kimliği varsa gönderir.
  - `none`: hiçbir zaman oturum kimliği göndermez.
- `claude-cli` varsayılan olarak `liveSession: "claude-stdio"`, `output: "jsonl"` ve `input: "stdin"` değerlerini kullanır; böylece takip eden etkileşimler, aktarım alanlarını belirtmeyen özel yapılandırmalarda bile canlı Claude süreci etkin olduğu sürece onu yeniden kullanır. Gateway yeniden başlarsa veya boşta olan süreç sonlanırsa OpenClaw, saklanan Claude oturum kimliğinden devam eder. Saklanan oturum kimlikleri, devam ettirmeden önce okunabilir bir proje dökümüne göre doğrulanır; eksik döküm, `--resume` altında sessizce yeni bir oturum başlatmak yerine bağlamayı temizler (`reason=transcript-missing` olarak günlüğe kaydedilir).
- Canlı Claude oturumları sınırlı JSONL çıktı korumalarını sürdürür: etkileşim başına 8 MiB ve 20,000 ham JSONL satırı.
- Saklanan CLI oturumları, sağlayıcının yönettiği sürekliliktir. Otomatik sıfırlama varsayılan olarak devre dışıdır; `/reset` ve açık günlük ya da boşta kalma `session.reset` politikaları yine de bunları keser.
- Yeni CLI oturumları normalde yalnızca OpenClaw'ın Compaction özetinden ve Compaction sonrası kuyruktan yeniden beslenir. Compaction öncesinde geçersiz kılınmış kısa oturumları kurtarmak için bir arka uç `reseedFromRawTranscriptWhenUncompacted: true` ile bunu etkinleştirebilir. Ham dökümü yeniden besleme işlemi sınırlı kalır ve eksik CLI dökümü, sahipsiz araç kullanımı kuyruğu, ileti politikası/sistem istemi/cwd/MCP değişiklikleri veya süresi dolan oturumun yeniden denenmesi gibi güvenli geçersiz kılmalarla sınırlandırılır; kimlik doğrulama profili veya kimlik bilgisi dönemi değişiklikleri hiçbir zaman ham döküm geçmişini yeniden beslemez.

Serileştirme: `serialize: true`, aynı şeritteki çalışmaları sıralı tutar (çoğu CLI tek bir sağlayıcı şeridinde serileştirilir). OpenClaw ayrıca seçilen kimlik doğrulama kimliği değiştiğinde saklanan CLI oturumunun yeniden kullanımını bırakır; buna değişen kimlik doğrulama profili kimliği, statik API anahtarı, statik belirteç veya CLI'ın sunduğu OAuth hesap kimliği dahildir. Yalnızca OAuth erişim/yenileme belirtecinin döndürülmesi oturumu kesmez. Bir CLI'ın kararlı bir OAuth hesap kimliği yoksa OpenClaw, ilgili CLI'ın kendi devam izinlerini uygulamasına olanak tanır.

## claude-cli oturumlarından geri dönüş ön bilgisi

Bir `claude-cli` denemesi [`agents.defaults.model.fallbacks`](/tr/concepts/model-failover) içindeki CLI olmayan bir adaya devredildiğinde OpenClaw, sonraki denemeyi Claude Code'un yerel JSONL transkriptinden (çalışma alanı başına anahtarlanmış olarak `~/.claude/projects/` altında) alınan bir bağlam önsözüyle başlatır. Bu başlangıç verisi olmadan yedek sağlayıcı boş bağlamla başlar; çünkü OpenClaw'ın kendi oturum transkripti `claude-cli` çalıştırmaları için boştur.

- Önsöz, en son `/compact` özetini veya `compact_boundary` işaretçisini tercih eder ve ardından karakter bütçesine kadar sınır sonrasındaki en son konuşma sıralarını ekler. Sınır öncesindeki konuşma sıraları, özet bunları zaten temsil ettiği için çıkarılır.
- İstem bütçesini doğru tutmak amacıyla araç blokları, kompakt `(tool call: name)` ve `(tool result: …)` ipuçları hâlinde birleştirilir; fazla büyük bir özet kısaltılır ve `(truncated)` olarak etiketlenir.
- Aynı sağlayıcı içindeki `claude-cli` ile `claude-cli` arasındaki yedek geçişler Claude'un kendi `--resume` özelliğine dayanır ve önsözü atlar.
- Başlangıç verisi, mevcut Claude oturum dosyası yolu doğrulamasını yeniden kullanır; böylece rastgele yollar okunamaz.

## Görüntüler

Plugin yazarları, görüntü yolu desteğini `imageArg` ile bildirir:

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw, base64 görüntülerini geçici dosyalara yazar. `imageArg` ayarlanmışsa bu yollar CLI bağımsız değişkenleri olarak iletilir; ayarlanmamışsa OpenClaw dosya yollarını isteme ekler (yol ekleme). Bu yöntem, yerel dosyaları düz yollardan otomatik olarak yükleyen CLI'larda çalışır.

## Girdiler ve çıktılar

- `output: "text"` (varsayılan), stdout'u nihai yanıt olarak değerlendirir.
- `output: "json"`, JSON'u ayrıştırmayı ve metin ile bir oturum kimliğini çıkarmayı dener.
- `output: "jsonl"`, bir JSONL akışını ayrıştırır ve mevcut olduklarında oturum tanımlayıcılarıyla birlikte son ajan iletisini çıkarır.
- Gemini CLI JSON çıktısında OpenClaw, `usage` eksik veya boş olduğunda yanıt metnini `response` alanından, kullanım verilerini ise `stats` alanından okur. Birlikte gelen Gemini CLI bağdaştırıcısı `stream-json` kullanır.

Girdi modları:

- `input: "arg"` (varsayılan), istemi son CLI bağımsız değişkeni olarak iletir.
- `input: "stdin"`, istemi stdin üzerinden gönderir.
- İstem çok uzunsa ve `maxPromptArgChars` ayarlanmışsa stdin kullanılır.

## Plugin'e ait varsayılanlar

CLI arka ucu varsayılanları, Plugin yüzeyinin bir parçasıdır:

- Plugin'ler bunları `api.registerCliBackend(...)` ile kaydeder.
- Arka ucun `id` değeri, model başvurularındaki sağlayıcı öneki olur.
- Komut, argv, ortam, ayrıştırıcı, oturum ve gözetim davranışı Plugin kodunda kalır.
- Arka uca özgü normalleştirme, isteğe bağlı `normalizeConfig` kancası aracılığıyla Plugin'e ait kalır.

Anthropic, `claude-cli` arka ucunun; Google ise `google-gemini-cli` arka ucunun sahibidir. OpenAI Codex ajan çalıştırmaları, `openai/*` aracılığıyla Codex uygulama sunucusu çalıştırma altyapısını kullanır; OpenClaw artık birlikte gelen bir `codex-cli` arka ucu kaydetmez.

Birlikte gelen Anthropic Plugin'i, `claude-cli` için şunları kaydeder:

| Anahtar                   | Değer                                                                                                                                                                                                         |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`             | `claude`                                                                                                                                                                                                      |
| `args`                | `-p --output-format stream-json --include-partial-messages --verbose --setting-sources user --allowedTools mcp__openclaw__* --disallowedTools ScheduleWakeup,CronCreate,Bash(run_in_background:true),Monitor` |
| `output`              | `jsonl`                                                                                                                                                                                                       |
| `input`               | `stdin`                                                                                                                                                                                                       |
| `modelArg`            | `--model`                                                                                                                                                                                                     |
| `sessionArgs`         | `["--session-id", "{sessionId}"]`                                                                                                                                                                             |
| `sessionMode`         | `always`                                                                                                                                                                                                      |
| `imageArg`            | `@`                                                                                                                                                                                                           |
| `imagePathScope`      | `workspace`                                                                                                                                                                                                   |
| `systemPromptFileArg` | `--append-system-prompt-file`                                                                                                                                                                                 |
| `systemPromptMode`    | `append`                                                                                                                                                                                                      |

Birlikte gelen Google Plugin'i, `google-gemini-cli` için şunları kaydeder:

| Anahtar                       | Değer                                                                                  |
| ------------------------- | -------------------------------------------------------------------------------------- |
| `command`                 | `gemini`                                                                               |
| `args`                    | `--skip-trust --approval-mode auto_edit --output-format stream-json --prompt {prompt}` |
| `resumeArgs`              | aynı, `--resume {sessionId}` ile                                                      |
| `output` / `resumeOutput` | `jsonl`                                                                                |
| `jsonlDialect`            | `gemini-stream-json`                                                                   |
| `imageArg`                | `@`                                                                                    |
| `imagePathScope`          | `workspace`                                                                            |
| `modelArg`                | `--model`                                                                              |
| `sessionMode`             | `existing`                                                                             |
| `sessionIdFields`         | `["session_id", "sessionId"]`                                                          |

Ön koşul: Yerel Gemini CLI kurulmuş ve `PATH` üzerinde `gemini` olarak bulunabilir olmalıdır (`brew install gemini-cli` veya `npm install -g @google/gemini-cli`).

Gemini CLI çıktı notları:

- Varsayılan `stream-json` ayrıştırıcısı; asistan `message` olaylarını, araç olaylarını, nihai `result` kullanımını ve önemli Gemini hata olaylarını okur.
- `usage` yoksa veya boşsa kullanım verileri `stats` alanına geri döner; `stats.cached`, OpenClaw `cacheRead` biçimine normalleştirilir ve `stats.input` eksikse girdi token'ları `stats.input_tokens - stats.cached` alanından türetilir.

## Metin dönüştürme katmanları

Küçük istem/ileti uyumluluk katmanlarına ihtiyaç duyan Plugin'ler, bir sağlayıcıyı veya CLI arka ucunu değiştirmeden çift yönlü metin dönüşümleri bildirebilir:

```typescript
api.registerTextTransforms({
  input: [{ from: /red basket/g, to: "blue basket" }],
  output: [{ from: /blue basket/g, to: "red basket" }],
});
```

`input`, CLI'a iletilen sistem istemini ve kullanıcı istemini yeniden yazar. `output`, OpenClaw kendi denetim işaretçilerini ve kanal teslimini işlemeden önce akış hâlindeki asistan metnini ve ayrıştırılmış nihai metni yeniden yazar; sağlayıcı destekli model çağrılarında ayrıca akış onarımından sonra ve araç yürütülmeden önce yapılandırılmış araç çağrısı bağımsız değişkenlerindeki dize değerlerini geri yükler. Ham sağlayıcı JSON parçaları değiştirilmeden bırakılır; tüketiciler yapılandırılmış kısmi, bitiş veya sonuç yükünü kullanmalıdır.

Sağlayıcıya özgü JSONL olayları yayan CLI'lar için ilgili arka ucun yapılandırmasında `jsonlDialect` değerini ayarlayın: Claude Code uyumlu akışlar için `claude-stream-json`, Gemini CLI `stream-json` olayları için `gemini-stream-json`.

## Yerel Compaction sahipliği

Bazı CLI arka uçları kendi transkriptini Compaction işlemine tabi tutan bir ajan çalıştırır; bu nedenle OpenClaw, koruyucu özetleyicisini bunlar üzerinde çalıştırmamalıdır. Bunu yapmak arka ucun kendi Compaction işlemiyle çakışır ve konuşma sırasının kesin olarak başarısız olmasına neden olabilir.

`claude-cli` bir çalıştırma altyapısı uç noktasına sahip değildir (Claude Code, Compaction işlemini dahili olarak gerçekleştirir); bu nedenle `ownsNativeCompaction: true` bildirir ve OpenClaw'ın Compaction yolu oturum girdisini değiştirmeden döndürür. OpenClaw, çalıştırmanın etkin bağlam bütçesini Claude Code'un belgelenmiş [`CLAUDE_CODE_AUTO_COMPACT_WINDOW`](https://code.claude.com/docs/en/env-vars) değişkeni üzerinden ileterek yerel otomatik Compaction işlemini yapılandırılmış Anthropic `contextTokens` sınırlarıyla uyumlu tutar. Codex gibi yerel çalıştırma altyapısı oturumları ise kendi çalıştırma altyapılarının Compaction uç noktasına yönlendirilmeye devam eder.

```typescript
api.registerCliBackend({ id: "my-cli", ownsNativeCompaction: true /* ... */ });
```

`ownsNativeCompaction` yalnızca Compaction işleminin gerçekten sahibi olan bir arka uç için bildirilmelidir: arka uç, kendi transkriptini bağlam penceresi yakınında güvenilir biçimde sınırlamalı ve sürdürülebilir bir oturumu (ör. `--resume` / `--session-id`) kalıcı hâle getirmelidir; aksi takdirde ertelenmiş bir oturum bütçeyi aşmış durumda kalabilir.

## Paket MCP katmanları

CLI arka uçları OpenClaw araç çağrılarını doğrudan almaz; ancak bir arka uç `bundleMcp: true` ile oluşturulan bir MCP yapılandırma katmanını etkinleştirebilir. Birlikte gelen mevcut davranış:

- `claude-cli`: oluşturulan katı MCP yapılandırma dosyası.
- `google-gemini-cli`: oluşturulan Gemini sistem ayarları dosyası.

Paket MCP etkinleştirildiğinde OpenClaw:

- Gateway araçlarını CLI işlemine sunan ve yalnızca geçerli yürütme denemesinde etkin olan çalıştırma başına bağlam izniyle (`OPENCLAW_MCP_TOKEN`) kimlik doğrulaması yapılan bir geri döngü HTTP MCP sunucusu başlatır;
- alt işlem başlıklarına güvenmek yerine araç erişimini Gateway tarafından seçilen oturum, hesap ve kanal bağlamına bağlar;
- geçerli çalışma alanı için etkinleştirilmiş paket MCP sunucularını yükler ve bunları mevcut arka uç MCP yapılandırma/ayar biçimiyle birleştirir;
- başlatma yapılandırmasını, sahibi olan Plugin'in arka uca ait entegrasyon modunu kullanarak yeniden yazar.

`toolsAllow` içeren cron işleri gibi kısıtlı çalıştırmalar, tam olarak
arka uç tarafından sağlanan bir çeviri gerektirir. Paketlenmiş `claude-cli` arka ucu; Claude'un
yerel araçlarını ve kancalar, Plugin'ler, aracılar, Skills ve `CLAUDE.md` dâhil
kullanıcı, proje ve yerel özelleştirmelerini devre dışı bırakır. Ardından izin verilen tüm
OpenClaw araçlarını, yetki kapsamlı MCP sunucusu üzerinden kullanıma sunar. Bu yaklaşım; dosya sistemi,
işlem, exec, onay ve korumalı alan politikasını, yetkiyi Claude'un yerel araçlarına veya
özelleştirme işlemlerine genişletmek yerine OpenClaw içinde tutar. Aynı MCP
listesi, Claude'un oluşturulan yapılandırmasında ve araçların listelenmesi ile
çalıştırılması sırasında Gateway tarafından yeniden uygulanır. Çekirdek, yetkiyi oluşturmadan önce
özgün izin verilenler listesinin dışında herhangi bir MCP iznini adlandıran arka uç
çevirilerini reddeder. Tam çevirisi olmayan arka uçlar yine güvenli biçimde başarısız olur.

Hiçbir MCP sunucusu etkin değilse OpenClaw, bir arka uç paket MCP'yi kullanmayı seçtiğinde yine de katı bir yapılandırma ekler; böylece arka plan çalıştırmaları yalıtılmış kalır.

Oturum kapsamlı paketlenmiş MCP çalışma zamanları, oturum içinde yeniden kullanılmak üzere önbelleğe alınır ve ardından 10 dakikalık boşta kalma süresinden sonra sonlandırılır. Kimlik doğrulama yoklamaları, kısa ad oluşturma ve Active Memory geri çağırma gibi tek seferlik gömülü çalıştırmalar, stdio alt süreçlerinin ve Akış Sağlayabilen HTTP/SSE akışlarının çalıştırmadan daha uzun süre yaşamaması için çalıştırma sonunda temizleme talep eder.

`claude-cli` için uyumlu, seçilmiş veya sıralanmış bir OpenClaw OAuth/token profili
ilgili Claude alt sürecine iletilir. Bu, uyumlu bir profil bulunmadığında Claude'un
yerel ana makine oturumunu korurken dönüş için aracı başına profilleri yetkili kılar.

## Geçmişi yeniden tohumlama sınırı

Yeni bir CLI oturumu, önceki bir OpenClaw dökümünden tohumlandığında (örneğin bir `session_expired` yeniden denemesinden sonra), yeniden tohumlama istemlerinin aşırı büyümesini önlemek için oluşturulan `<conversation_history>` bloğu sınırlandırılır. Varsayılan değer 12,288 karakterdir (yaklaşık 3,000 token).

Claude CLI arka uçları bunun yerine bu sınırı çözümlenen Claude bağlam penceresine göre ölçeklendirir: daha büyük bağlam pencereleri, sabit bir üst sınıra kadar önceki geçmişten daha büyük bir kesit alır; diğer CLI arka uçları ise ihtiyatlı varsayılan değeri korur. Bu sınır yalnızca yeniden tohumlama isteminin önceki geçmiş bloğunu yönetir.

## Sınırlamalar

- OpenClaw, CLI arka uç protokolüne araç çağrıları eklemez. Arka uçlar Gateway araçlarını yalnızca `bundleMcp: true` kullanmayı seçtiklerinde görür.
- Akış arka uca özgüdür: bazı arka uçlar JSONL akışı sağlar, diğerleri çıkışa kadar arabelleğe alır.
- Yapılandırılmış çıktılar CLI'ın kendi JSON biçimine bağlıdır.

## Sorun giderme

| Belirti               | Düzeltme                                                                                            |
| --------------------- | ---------------------------------------------------------------------------------------------- |
| CLI bulunamadı         | CLI'ı Gateway hizmetinin `PATH` konumuna ekleyin veya sahibi olan Plugin'in kayıtlı komutunu güncelleyin. |
| Yanlış model adı      | Plugin'in `modelAliases` eşlemesini güncelleyin.                                                    |
| Oturum sürekliliği yok | Plugin'in `sessionArgs` ve `sessionMode` değerlerini denetleyin.                                            |
| Görseller yok sayılıyor        | Plugin'in `imageArg` değerini ve CLI'ın dosya yolu desteğini denetleyin.                                 |

## İlgili

- [Gateway çalıştırma kılavuzu](/tr/gateway)
- [Yerel modeller](/tr/gateway/local-models)
