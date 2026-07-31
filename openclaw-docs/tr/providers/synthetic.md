---
read_when:
    - Synthetic'i bir model sağlayıcısı olarak kullanmak istiyorsunuz
    - Synthetic API anahtarı veya temel URL yapılandırması gerekir
summary: OpenClaw'da Synthetic'in Anthropic uyumlu API'sini kullanın
title: Synthetic
x-i18n:
    generated_at: "2026-07-26T23:37:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3f6cc89a7b837f57555d176ce78e62a39095d4ef0765c96b6b7b93ffebd7388
    source_path: providers/synthetic.md
    workflow: 16
---

[Synthetic](https://synthetic.new), Anthropic uyumlu uç noktalar sunar.
OpenClaw bunu `synthetic` sağlayıcısı olarak paketler ve Anthropic
Messages API'sini kullanır.

| Özellik   | Değer                                 |
| --------- | ------------------------------------- |
| Sağlayıcı | `synthetic`                    |
| Kimlik doğrulama | `SYNTHETIC_API_KEY`           |
| API       | Anthropic Messages                    |
| Temel URL | `https://api.synthetic.new/anthropic`                    |

## Başlarken

<Steps>
  <Step title="Bir API anahtarı edinin">
    Synthetic hesabınızdan bir `SYNTHETIC_API_KEY` edinin veya ilk kurulumun
    sizden bir tane istemesine izin verin.
  </Step>
  <Step title="İlk kurulumu çalıştırın">
    ```bash
    openclaw onboard --auth-choice synthetic-api-key
    ```
  </Step>
  <Step title="Varsayılan modeli doğrulayın">
    İlk kurulum, varsayılan modeli şu şekilde ayarlar:
    ```text
    synthetic/hf:MiniMaxAI/MiniMax-M3
    ```
  </Step>
</Steps>

<Warning>
OpenClaw'ın Anthropic istemcisi, temel URL'ye otomatik olarak `/v1` ekler; bu nedenle
`https://api.synthetic.new/anthropic` kullanın (`/anthropic/v1` değil). Synthetic
temel URL'sini değiştirirse `models.providers.synthetic.baseUrl` değerini geçersiz kılın.
</Warning>

## Yapılandırma örneği

```json5
{
  env: { SYNTHETIC_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
      models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "hf:MiniMaxAI/MiniMax-M3",
            name: "MiniMax M3",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

## Yerleşik katalog

Tüm Synthetic modelleri `0` maliyetini kullanır (girdi/çıktı/önbellek). Hizmet kullanılabilirliği için Synthetic'in
[güncel model listesine](https://dev.synthetic.new/docs/api/models) bakın.

| Model kimliği                                       | Bağlam penceresi | Maksimum belirteç | Akıl yürütme | Girdi        |
| --------------------------------------------------- | ---------------- | ----------------- | ------------ | ------------ |
| `hf:MiniMaxAI/MiniMax-M3`                                  | 262,144          | 65,536            | evet         | metin + görsel |
| `hf:moonshotai/Kimi-K2.7-Code`                                  | 262,144          | 8,192             | evet         | metin + görsel |
| `hf:nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4`                                  | 262,144          | 8,192             | evet         | metin        |
| `hf:openai/gpt-oss-120b`                                  | 131,072          | 8,192             | evet         | metin        |
| `hf:Qwen/Qwen3.6-27B`                                  | 262,144          | 81,920            | evet         | metin + görsel |
| `hf:zai-org/GLM-4.7-Flash`                                  | 196,608          | 131,072           | evet         | metin        |
| `hf:zai-org/GLM-5.2`                                  | 524,288          | 131,072           | evet         | metin        |

<Tip>
Model referansları `synthetic/<modelId>` biçimini kullanır. Hesabınızda
kullanılabilen tüm modelleri görmek için `openclaw models list --provider synthetic`
kullanın.
</Tip>

<AccordionGroup>
  <Accordion title="Model izin listesi">
    Bir model izin listesini (`agents.defaults.modelPolicy.allow`) etkinleştirirseniz kullanmayı
    planladığınız her Synthetic modelini ekleyin. İzin listesinde bulunmayan modeller
    ajandan gizlenir.
  </Accordion>

  <Accordion title="Temel URL'yi geçersiz kılma">
    Synthetic API uç noktasını değiştirirse temel URL'yi geçersiz kılın:

    ```json5
    {
      models: {
        providers: {
          synthetic: {
            baseUrl: "https://new-api.synthetic.new/anthropic",
          },
        },
      },
    }
    ```

    OpenClaw yine de `/v1` değerini otomatik olarak ekler.

  </Accordion>
</AccordionGroup>

## İlgili

<CardGroup cols={2}>
  <Card title="Model sağlayıcıları" href="/tr/concepts/model-providers" icon="layers">
    Sağlayıcı kuralları, model referansları ve yük devretme davranışı.
  </Card>
  <Card title="Yapılandırma başvurusu" href="/tr/gateway/configuration-reference" icon="gear">
    Sağlayıcı ayarlarını içeren eksiksiz yapılandırma şeması.
  </Card>
  <Card title="Synthetic" href="https://synthetic.new" icon="arrow-up-right-from-square">
    Synthetic panosu ve API belgeleri.
  </Card>
</CardGroup>
