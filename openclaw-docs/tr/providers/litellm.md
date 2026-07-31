---
read_when:
    - OpenClaw'u bir LiteLLM proxy'si üzerinden yönlendirmek istiyorsunuz
    - LiteLLM üzerinden maliyet takibine, günlük kaydına veya model yönlendirmesine ihtiyacınız var
summary: Birleşik model erişimi ve maliyet takibi için OpenClaw'ı LiteLLM Proxy üzerinden çalıştırın
title: LiteLLM
x-i18n:
    generated_at: "2026-07-27T00:15:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 22451f0eefcf991a602409701fc752f97600a67752c67304137c7f17f3dd1a16
    source_path: providers/litellm.md
    workflow: 16
---

[LiteLLM](https://litellm.ai), 100'den fazla model sağlayıcısı için birleşik API sunan açık kaynaklı bir LLM gateway'idir.
Merkezi maliyet takibi, günlük kaydı, harcama limitli sanal anahtarlar ve OpenClaw yapılandırmasını
değiştirmeden arka uç yük devretme olanağı için OpenClaw'ı LiteLLM üzerinden yönlendirin.

## Hızlı başlangıç

<Tabs>
  <Tab title="İlk kurulum (önerilen)">
    ```bash
    openclaw onboard --auth-choice litellm-api-key
    ```

    Uzak bir proxy ile etkileşimsiz kurulum için proxy URL'sini açıkça iletin:

    ```bash
    openclaw onboard --non-interactive --accept-risk --auth-choice litellm-api-key \
      --litellm-api-key "$LITELLM_API_KEY" --custom-base-url "https://litellm.example/v1"
    ```

  </Tab>

  <Tab title="Manuel kurulum">
    <Steps>
      <Step title="LiteLLM Proxy'yi başlatın">
        ```bash
        pip install 'litellm[proxy]'
        litellm --model claude-opus-4-6
        ```
      </Step>
      <Step title="OpenClaw'ı LiteLLM'e yönlendirin">
        ```bash
        export LITELLM_API_KEY="your-litellm-key"
        openclaw
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

## Yapılandırma

```json5
{
  models: {
    providers: {
      litellm: {
        baseUrl: "http://localhost:4000",
        apiKey: "${LITELLM_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "claude-opus-4-6",
            name: "Claude Opus 4.6",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 200000,
            maxTokens: 64000,
          },
          {
            id: "gpt-4o",
            name: "GPT-4o",
            reasoning: false,
            input: ["text", "image"],
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "litellm/claude-opus-4-6" },
    },
  },
}
```

İlk kurulumun yazdığı varsayılan model `litellm/claude-opus-4-6` değeridir.

## Görsel oluşturma

LiteLLM, OpenAI uyumlu `/images/generations` ve `/images/edits` rotaları üzerinden
`image_generate` aracını destekleyebilir. Varsayılan görsel modeli `gpt-image-2` değeridir; farklı bir modeli
`agents.defaults.mediaModels.image` altında yapılandırın:

```json5
{
  models: {
    providers: {
      litellm: {
        baseUrl: "http://localhost:4000",
        apiKey: "${LITELLM_API_KEY}",
      },
    },
  },
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "litellm/gpt-image-2",
        timeoutMs: 180_000,
      },
    },
  },
}
```

Geri döngü LiteLLM URL'leri (`http://localhost:4000`, `127.0.0.1`, `::1`, `host.docker.internal`) genel bir
özel ağ geçersiz kılması olmadan çalışır. LAN üzerinde barındırılan bir proxy için API anahtarı bu ana makineye
gönderildiğinden `models.providers.litellm.request.allowPrivateNetwork: true` ayarını yapın.

## Gelişmiş

<AccordionGroup>
  <Accordion title="Sanal anahtarlar">
    OpenClaw için harcama limitleri olan özel bir anahtar oluşturun:

    ```bash
    curl -X POST "http://localhost:4000/key/generate" \
      -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
      -H "Content-Type: application/json" \
      -d '{
        "key_alias": "openclaw",
        "max_budget": 50.00,
        "budget_duration": "monthly"
      }'
    ```

    Oluşturulan anahtarı `LITELLM_API_KEY` olarak kullanın.

  </Accordion>

  <Accordion title="Model yönlendirme">
    LiteLLM, model isteklerini farklı arka uçlara yönlendirebilir. LiteLLM `config.yaml` dosyanızda yapılandırın:

    ```yaml
    model_list:
      - model_name: claude-opus-4-6
        litellm_params:
          model: claude-opus-4-6
          api_key: os.environ/ANTHROPIC_API_KEY

      - model_name: gpt-4o
        litellm_params:
          model: gpt-4o
          api_key: os.environ/OPENAI_API_KEY
    ```

    OpenClaw, `claude-opus-4-6` istemeye devam eder; yönlendirmeyi LiteLLM gerçekleştirir.

  </Accordion>

  <Accordion title="Kullanımı görüntüleme">
    ```bash
    # Anahtar bilgileri
    curl "http://localhost:4000/key/info" \
      -H "Authorization: Bearer sk-litellm-key"

    # Harcama günlükleri
    curl "http://localhost:4000/spend/logs" \
      -H "Authorization: Bearer $LITELLM_MASTER_KEY"
    ```

  </Accordion>

  <Accordion title="Proxy davranışı notları">
    - LiteLLM varsayılan olarak `http://localhost:4000` üzerinde çalışır.
    - OpenClaw, LiteLLM'in proxy tarzı OpenAI uyumlu `/v1` uç noktası üzerinden bağlanır.
    - Yalnızca yerel OpenAI için kullanılan istek şekillendirme, yapılandırılmış bir LiteLLM temel URL'si üzerinden uygulanmaz:
      `service_tier` yoktur, Responses `store` yoktur, istem önbelleği ipuçları yoktur ve OpenAI akıl yürütme çabası
      yükü şekillendirmesi yoktur.
    - Gizli OpenClaw ilişkilendirme başlıkları (`originator`, `version`, `User-Agent`) yalnızca
      doğrulanmış yerel OpenAI uç noktalarına gönderilir; bu nedenle özel bir LiteLLM temel URL'sine eklenmez.
  </Accordion>
</AccordionGroup>

<Note>
Genel sağlayıcı yapılandırması ve yük devretme davranışı için [Model Sağlayıcıları](/tr/concepts/model-providers) bölümüne bakın.
</Note>

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="LiteLLM Belgeleri" href="https://docs.litellm.ai" icon="book">
    Resmî LiteLLM belgeleri ve API referansı.
  </Card>
  <Card title="Model seçimi" href="/tr/concepts/model-providers" icon="layers">
    Tüm sağlayıcılara, model referanslarına ve yük devretme davranışına genel bakış.
  </Card>
  <Card title="Yapılandırma" href="/tr/gateway/configuration" icon="gear">
    Eksiksiz yapılandırma referansı.
  </Card>
  <Card title="Modeller" href="/tr/concepts/models" icon="brain">
    Modellerin nasıl seçileceği ve yapılandırılacağı.
  </Card>
</CardGroup>
