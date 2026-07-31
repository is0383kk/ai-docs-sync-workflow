---
read_when:
    - OpenClaw'u antirez/ds4 ile çalıştırmak istiyorsunuz
    - Araç çağrılarını destekleyen yerel bir DeepSeek V4 Flash arka ucu istiyorsunuz
    - ds4-server için OpenClaw yapılandırmasına ihtiyacınız var
summary: OpenClaw'ı yerel ve OpenAI uyumlu bir DeepSeek V4 Flash sunucusu olan ds4 üzerinden çalıştırın
title: ds4
x-i18n:
    generated_at: "2026-07-27T00:13:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: be449813295648694625ef8003b3f4b12903535b74816916ca5af0695174fbf4
    source_path: providers/ds4.md
    workflow: 16
---

[ds4](https://github.com/antirez/ds4), yerel bir Metal arka ucundan OpenAI uyumlu bir `/v1` API ile DeepSeek V4 Flash sunar. OpenClaw, genel `openai-completions` sağlayıcı ailesi üzerinden ds4'e bağlanır.

ds4, OpenClaw ile birlikte gelen bir sağlayıcı plugini değildir. Onu
`models.providers.ds4` altında yapılandırın, ardından `ds4/deepseek-v4-flash` seçeneğini belirleyin.

| Özellik       | Değer                                                     |
| ------------- | --------------------------------------------------------- |
| Sağlayıcı kimliği | `ds4`                                                     |
| Plugin        | yok (yalnızca yapılandırma)                               |
| API           | OpenAI uyumlu Chat Completions (`openai-completions`) |
| Temel URL     | `http://127.0.0.1:18000/v1` (önerilen)                   |
| Model kimliği | `deepseek-v4-flash`                                       |
| Araç çağrıları | OpenAI tarzı `tools` / `tool_calls`                       |
| Akıl yürütme  | DeepSeek tarzı `thinking` ve `reasoning_effort`          |

## Gereksinimler

- Metal desteğine sahip macOS.
- `ds4-server` ve DeepSeek V4 Flash GGUF dosyasını içeren çalışan bir ds4 çalışma kopyası.
- Seçtiğiniz bağlam için yeterli bellek; daha büyük `--ctx` değerleri, sunucu başlatılırken daha fazla
  KV belleği ayırır.

<Warning>
OpenClaw agent turları, araç şemalarını ve çalışma alanı bağlamını içerir. `--ctx 4096`
gibi küçük bir bağlam doğrudan curl testlerini geçebilir ancak tam agent çalıştırmaları
`500 prompt exceeds context` ile başarısız olabilir. Agent ve araç duman testleri için en az
`--ctx 32768` kullanın. `--ctx 393216` değerini yalnızca yeterli bellekle ve ds4
Think Max'i etkinleştirmek için kullanın.
</Warning>

## Hızlı başlangıç

<Steps>
  <Step title="ds4-server'ı başlatın">
    `<DS4_DIR>` değerini ds4 çalışma kopyanızın yoluyla değiştirin.

    ```bash
    <DS4_DIR>/ds4-server \
      --model <DS4_DIR>/ds4flash.gguf \
      --host 127.0.0.1 \
      --port 18000 \
      --ctx 32768 \
      --tokens 128
    ```

  </Step>
  <Step title="OpenAI uyumlu uç noktayı doğrulayın">
    ```bash
    curl http://127.0.0.1:18000/v1/models
    ```

    Yanıt `deepseek-v4-flash` değerini içermelidir.

  </Step>
  <Step title="OpenClaw sağlayıcı yapılandırmasını ekleyin">
    [Tam yapılandırma](#full-config) bölümündeki yapılandırmayı ekleyin, ardından tek seferlik bir model
    denetimi çalıştırın:

    ```bash
    openclaw infer model run \
      --local \
      --model ds4/deepseek-v4-flash \
      --thinking off \
      --prompt "Reply with exactly: openclaw-ds4-ok" \
      --json
    ```

  </Step>
</Steps>

## Tam yapılandırma

ds4 zaten `127.0.0.1:18000` üzerinde çalışıyorsa bu yapılandırmayı kullanın.

```json5
{
  agents: {
    defaults: {
      model: { primary: "ds4/deepseek-v4-flash" },
      models: {
        "ds4/deepseek-v4-flash": {
          alias: "DS4 local",
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      ds4: {
        baseUrl: "http://127.0.0.1:18000/v1",
        apiKey: "ds4-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        models: [
          {
            id: "deepseek-v4-flash",
            name: "DeepSeek V4 Flash (ds4)",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32768,
            maxTokens: 128,
            compat: {
              supportsUsageInStreaming: true,
              supportsReasoningEffort: true,
              maxTokensField: "max_tokens",
              supportsStrictMode: false,
              thinkingFormat: "deepseek",
              supportedReasoningEfforts: ["low", "medium", "high", "xhigh"],
            },
          },
        ],
      },
    },
  },
}
```

`contextWindow` değerini `ds4-server --ctx` ile uyumlu tutun. OpenClaw'un sunucu varsayılanından
daha az çıktı istemesini bilinçli olarak tercih etmiyorsanız `maxTokens` değerini
`--tokens` ile uyumlu tutun.

## İstek üzerine başlatma

OpenClaw, ds4'ü yalnızca bir `ds4/...` modeli seçildiğinde başlatabilir. Aynı sağlayıcı
girdisine `localService` ekleyin:

```json5
{
  models: {
    providers: {
      ds4: {
        baseUrl: "http://127.0.0.1:18000/v1",
        apiKey: "ds4-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "<DS4_DIR>/ds4-server",
          args: [
            "--model",
            "<DS4_DIR>/ds4flash.gguf",
            "--host",
            "127.0.0.1",
            "--port",
            "18000",
            "--ctx",
            "32768",
            "--tokens",
            "128",
          ],
          cwd: "<DS4_DIR>",
          healthUrl: "http://127.0.0.1:18000/v1/models",
          readyTimeoutMs: 300000,
          idleStopMs: 0,
        },
        models: [
          {
            id: "deepseek-v4-flash",
            name: "DeepSeek V4 Flash (ds4)",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32768,
            maxTokens: 128,
            compat: {
              supportsUsageInStreaming: true,
              supportsReasoningEffort: true,
              maxTokensField: "max_tokens",
              supportsStrictMode: false,
              thinkingFormat: "deepseek",
              supportedReasoningEfforts: ["low", "medium", "high", "xhigh"],
            },
          },
        ],
      },
    },
  },
}
```

`command` mutlak bir yürütülebilir dosya yolu olmalıdır. Kabuk araması ve `~` genişletmesi
kullanılmaz. Tüm `localService` alanları için [Yerel model hizmetleri](/tr/gateway/local-model-services)
bölümüne bakın.

## Think Max

ds4, Think Max'i yalnızca aşağıdaki iki koşul da karşılandığında uygular:

- `ds4-server`, `--ctx 393216` veya daha yüksek bir değerle başlar.
- İstek `reasoning_effort: "max"` değerini (veya eşdeğer ds4 efor alanını) kullanır.

Bu büyük bağlamı çalıştırırsanız hem sunucu bayraklarını hem de OpenClaw model
meta verilerini güncelleyin:

```json5
{
  contextWindow: 393216,
  maxTokens: 384000,
  compat: {
    supportsUsageInStreaming: true,
    supportsReasoningEffort: true,
    maxTokensField: "max_tokens",
    supportsStrictMode: false,
    thinkingFormat: "deepseek",
    supportedReasoningEfforts: ["low", "medium", "high", "xhigh", "max"],
  },
}
```

## Test

OpenClaw'u atlayan doğrudan HTTP denetimi:

```bash
curl http://127.0.0.1:18000/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"Reply with exactly: ds4-ok"}],"max_tokens":16,"stream":false,"thinking":{"type":"disabled"}}'
```

OpenClaw model yönlendirmesi (Hızlı başlangıç denetimiyle aynı):

```bash
openclaw infer model run \
  --local \
  --model ds4/deepseek-v4-flash \
  --thinking off \
  --prompt "Reply with exactly: openclaw-ds4-ok" \
  --json
```

En az 32768 bağlamla tam agent ve araç çağrısı duman testi:

```bash
openclaw agent \
  --local \
  --session-id ds4-tool-smoke \
  --model ds4/deepseek-v4-flash \
  --thinking off \
  --message "Use the shell command pwd once, then reply exactly: tool-ok <output>" \
  --json \
  --timeout 240
```

Beklenen sonuç:

- `executionTrace.winnerProvider`, `ds4` değeridir
- `executionTrace.winnerModel`, `deepseek-v4-flash` değeridir
- `toolSummary.calls`, en az `1` değeridir
- `finalAssistantVisibleText`, `tool-ok` ile başlar

## Sorun giderme

<AccordionGroup>
  <Accordion title="curl /v1/models bağlanamıyor">
    ds4 çalışmıyor veya `baseUrl` içindeki ana bilgisayara/bağlantı noktasına bağlı değil.
    `ds4-server` öğesini başlatın, ardından yeniden deneyin:

    ```bash
    curl http://127.0.0.1:18000/v1/models
    ```

  </Accordion>

  <Accordion title="500 istemi bağlamı aşıyor">
    Yapılandırılmış `--ctx`, OpenClaw turu için çok küçük. `ds4-server --ctx`
    değerini artırın, ardından `models.providers.ds4.models[].contextWindow` değerini eşleşecek şekilde
    güncelleyin. Araçları içeren tam agent turları, doğrudan tek mesajlı bir curl
    isteğinden önemli ölçüde daha fazla bağlam gerektirir.
  </Accordion>

  <Accordion title="Think Max etkinleşmiyor">
    ds4, Think Max'i yalnızca `--ctx` en az `393216` olduğunda ve istek
    `reasoning_effort: "max"` istediğinde kullanır. Daha küçük bağlamlar yüksek
    akıl yürütmeye geri döner.
  </Accordion>

  <Accordion title="İlk istek yavaş">
    ds4'ün soğuk Metal yerleşimi ve model ısınma aşaması vardır. OpenClaw sunucuyu istek
    üzerine başlattığında `localService.readyTimeoutMs: 300000` değerini ayarlayın.
  </Accordion>
</AccordionGroup>

## İlgili

<CardGroup cols={2}>
  <Card title="Yerel model hizmetleri" href="/tr/gateway/local-model-services" icon="play">
    Model isteklerinden önce yerel model sunucularını istek üzerine başlatın.
  </Card>
  <Card title="Yerel modeller" href="/tr/gateway/local-models" icon="server">
    Yerel model arka uçlarını seçin ve çalıştırın.
  </Card>
  <Card title="Model sağlayıcıları" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcı referanslarını, kimlik doğrulamayı ve yük devretmeyi yapılandırın.
  </Card>
  <Card title="DeepSeek" href="/tr/providers/deepseek" icon="brain">
    Yerel DeepSeek sağlayıcı davranışı ve düşünme denetimleri.
  </Card>
</CardGroup>
