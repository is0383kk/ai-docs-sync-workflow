---
read_when:
    - OpenClaw'u yerel bir inferrs sunucusuna karşı çalıştırmak istiyorsunuz
    - Gemma veya başka bir modeli Inferrs üzerinden sunuyorsunuz
    - inferrs için tam OpenClaw uyumluluk bayraklarına ihtiyacınız var
summary: OpenClaw'ı inferrs (OpenAI uyumlu yerel sunucu) üzerinden çalıştırın
title: Inferrs
x-i18n:
    generated_at: "2026-07-26T23:32:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8b9b6fe337a2ec6536332dd62840052fd802fad0a5f3d885ce137523266ff3c9
    source_path: providers/inferrs.md
    workflow: 16
---

[inferrs](https://github.com/ericcurtin/inferrs), yerel modelleri OpenAI uyumlu bir `/v1` API'sinin arkasında sunar. OpenClaw, genel `openai-completions` bağdaştırıcısı üzerinden onunla iletişim kurar.

| Özellik            | Değer                                                                |
| ------------------ | -------------------------------------------------------------------- |
| Sağlayıcı kimliği  | `inferrs` (özel; `models.providers.inferrs` altında yapılandırın)   |
| Plugin             | yok — paketle birlikte gelen bir OpenClaw sağlayıcı Plugin'i değildir |
| Kimlik doğrulama ortam değişkeni | gerekli değil; inferrs sunucunuzda kimlik doğrulama yoksa herhangi bir değer çalışır |
| API                | OpenAI uyumlu (`openai-completions`)                                   |
| Önerilen temel URL | `http://127.0.0.1:8080/v1` (veya inferrs sunucunuzun dinlediği konum)         |

<Note>
  `inferrs`, özel ve kendi ortamınızda barındırılan OpenAI uyumlu bir arka uçtur; özel bir OpenClaw sağlayıcı Plugin'i değildir: bir ilk katılım kimlik doğrulama seçeneğini belirlemek yerine onu `models.providers.inferrs` altında yapılandırırsınız. Otomatik keşif özelliğine sahip, paketle birlikte gelen bir Plugin için [SGLang](/tr/providers/sglang) veya [vLLM](/tr/providers/vllm) bölümüne bakın.
</Note>

## Başlarken

<Steps>
  <Step title="inferrs'ı bir modelle başlatın">
    ```bash
    inferrs serve google/gemma-4-E2B-it \
      --host 127.0.0.1 \
      --port 8080 \
      --device metal
    ```
  </Step>
  <Step title="Sunucuya erişilebildiğini doğrulayın">
    ```bash
    curl http://127.0.0.1:8080/health
    curl http://127.0.0.1:8080/v1/models
    ```
  </Step>
  <Step title="Bir OpenClaw sağlayıcı girdisi ekleyin">
    Açık bir sağlayıcı girdisi ekleyin ve varsayılan modelinizi ona yönlendirin. Aşağıdaki yapılandırma örneğine bakın.
  </Step>
</Steps>

## Tam yapılandırma örneği

Yerel bir `inferrs` sunucusunda Gemma 4:

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/google/gemma-4-E2B-it" },
      models: {
        "inferrs/google/gemma-4-E2B-it": {
          alias: "Gemma 4 (inferrs)",
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

## İsteğe bağlı başlatma

OpenClaw, yalnızca bir `inferrs/...` modeli seçildiğinde `inferrs` öğesini kendisi başlatabilir. Aynı sağlayıcı girdisine `localService` ekleyin:

```json5
{
  models: {
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "/opt/homebrew/bin/inferrs",
          args: [
            "serve",
            "google/gemma-4-E2B-it",
            "--host",
            "127.0.0.1",
            "--port",
            "8080",
            "--device",
            "metal",
          ],
          healthUrl: "http://127.0.0.1:8080/v1/models",
          readyTimeoutMs: 180000,
          idleStopMs: 0,
        },
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

`command` mutlak bir yol olmalıdır. Gateway ana makinesinde `which inferrs` komutunu çalıştırın ve bu yolu kullanın. Tam alan başvurusu: [Yerel model hizmetleri](/tr/gateway/local-model-services).

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="requiresStringContent neden önemlidir">
    Bazı `inferrs` Chat Completions yolları, yapılandırılmış içerik parçası dizileri yerine yalnızca dize türündeki `messages[].content` değerlerini kabul eder.

    <Warning>
    OpenClaw çalıştırmaları şu hatayla başarısız olursa:

    ```text
    messages[1].content: geçersiz tür: dizi, bir dize bekleniyordu
    ```

    model girdisinde `compat.requiresStringContent: true` ayarını yapın. Ardından OpenClaw, isteği göndermeden önce yalnızca metin içeren içerik parçalarını düz dizelere dönüştürür.
    </Warning>

  </Accordion>

  <Accordion title="Gemma ve araç şemasıyla ilgili uyarı">
    Bazı `inferrs` + Gemma birleşimleri, küçük ve doğrudan `/v1/chat/completions` isteklerini kabul ederken tam OpenClaw aracı çalışma zamanı turlarında başarısız olur. Önce araç şeması yüzeyini devre dışı bırakmayı deneyin:

    ```json5
    compat: {
      requiresStringContent: true,
      supportsTools: false
    }
    ```

    Bu, daha katı yerel arka uçlardaki istem yükünü azaltır. Küçük doğrudan istekler çalışmaya devam ederken normal OpenClaw aracı turları `inferrs` içinde çökmeyi sürdürüyorsa bunu bir OpenClaw aktarım sorunu yerine üst kaynaklı bir model/sunucu sınırlaması olarak değerlendirin.

  </Accordion>

  <Accordion title="Elle hızlı doğrulama testi">
    Yapılandırmanın ardından her iki katmanı da bir kez test edin:

    ```bash
    curl http://127.0.0.1:8080/v1/chat/completions \
      -H 'content-type: application/json' \
      -d '{"model":"google/gemma-4-E2B-it","messages":[{"role":"user","content":"2 + 2 kaç eder?"}],"stream":false}'
    ```

    ```bash
    openclaw infer model run \
      --model inferrs/google/gemma-4-E2B-it \
      --prompt "2 + 2 kaç eder? Tek bir kısa cümleyle yanıt ver." \
      --json
    ```

    İlk komut çalışıyor ancak ikincisi başarısız oluyorsa aşağıdaki Sorun giderme bölümüne bakın.

  </Accordion>

  <Accordion title="Proxy tarzı davranış">
    `inferrs`, `openai-responses` yerine genel `openai-completions` bağdaştırıcısını kullandığından yalnızca yerel OpenAI'ye özgü istek biçimlendirmesi hiçbir zaman uygulanmaz: `service_tier`, Responses `store`, istem önbelleği ipuçları veya OpenAI akıl yürütme uyumluluğu yük biçimlendirmesi gönderilmez.
  </Accordion>
</AccordionGroup>

## Sorun giderme

<AccordionGroup>
  <Accordion title="curl /v1/models başarısız oluyor">
    `inferrs` çalışmıyor, erişilebilir değil veya yapılandırdığınız ana makineye/bağlantı noktasına bağlı değil. Sunucunun başlatıldığını ve bu adreste dinlediğini doğrulayın.
  </Accordion>

  <Accordion title="messages[].content bir dize bekliyordu">
    Model girdisinde `compat.requiresStringContent: true` ayarını yapın (yukarıya bakın).
  </Accordion>

  <Accordion title="Doğrudan /v1/chat/completions çağrıları başarılı ancak openclaw infer model run başarısız oluyor">
    Araç şeması yüzeyini devre dışı bırakmak için `compat.supportsTools: false` ayarını yapın (yukarıdaki Gemma uyarısına bakın).
  </Accordion>

  <Accordion title="inferrs daha büyük aracı turlarında hâlâ çöküyor">
    Şema hataları giderilmiş olmasına rağmen `inferrs` daha büyük aracı turlarında hâlâ çöküyorsa bunu üst kaynaklı bir `inferrs` veya model sınırlaması olarak değerlendirin. İstem yükünü azaltın ya da arka ucu/modeli değiştirin.
  </Accordion>
</AccordionGroup>

<Tip>
Genel yardım için [Sorun giderme](/tr/help/troubleshooting) ve [SSS](/tr/help/faq) bölümlerine bakın.
</Tip>

## İlgili

<CardGroup cols={2}>
  <Card title="Yerel modeller" href="/tr/gateway/local-models" icon="server">
    OpenClaw'ı yerel model sunucularıyla çalıştırma.
  </Card>
  <Card title="Yerel model hizmetleri" href="/tr/gateway/local-model-services" icon="play">
    Yapılandırılmış sağlayıcılar için yerel model sunucularını isteğe bağlı olarak başlatma.
  </Card>
  <Card title="Gateway sorunlarını giderme" href="/tr/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail" icon="wrench">
    Yoklamaları geçen ancak aracı çalıştırmalarında başarısız olan yerel OpenAI uyumlu arka uçlarda hata ayıklama.
  </Card>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Tüm sağlayıcılara, model referanslarına ve yük devretme davranışına genel bakış.
  </Card>
</CardGroup>
