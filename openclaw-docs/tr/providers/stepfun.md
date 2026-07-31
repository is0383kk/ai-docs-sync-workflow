---
read_when:
    - OpenClaw'da StepFun modellerini kullanmak istiyorsunuz
    - StepFun kurulum rehberliğine ihtiyacınız var
summary: OpenClaw ile StepFun modellerini kullanma
title: StepFun
x-i18n:
    generated_at: "2026-07-26T22:59:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 462a2588f15e8d6188914e238a3e472052d0da1da151751adecdb63cf009fc64
    source_path: providers/stepfun.md
    workflow: 16
---

StepFun, iki sağlayıcı kimliğine sahip harici bir resmî plugin (`@openclaw/stepfun-provider`) olarak sunulur:

Standart uç nokta için

- `stepfun`
Step Plan uç noktası için
- `stepfun-plan`

<Warning>
Standard ve Step Plan, farklı uç noktalara ve model başvurusu ön eklerine (`stepfun/...` ile `stepfun-plan/...`) sahip **ayrı sağlayıcılardır**. `.com` uç noktalarıyla Çin anahtarı, `.ai` uç noktalarıyla global anahtar kullanın.
</Warning>

## Plugini yükleme

```bash
openclaw plugins install @openclaw/stepfun-provider
openclaw gateway restart
```

## Bölge ve uç noktalara genel bakış

| Uç nokta  | Çin (`.com`)                         | Global (`.ai`)                        |
| --------- | -------------------------------------- | ------------------------------------- |
| Standard  | `https://api.stepfun.com/v1`           | `https://api.stepfun.ai/v1`           |
| Step Plan | `https://api.stepfun.com/step_plan/v1` | `https://api.stepfun.ai/step_plan/v1` |

Kimlik doğrulama ortam değişkeni: `STEPFUN_API_KEY`

## Yerleşik katalog

Standard (`stepfun`):

| Model başvurusu          | Bağlam  | Maks. çıktı | Notlar                         |
| ------------------------ | ------- | ----------- | ------------------------------ |
| `stepfun/step-3.5-flash` | 262,144 | 65,536      | Varsayılan standart model      |
| `stepfun/step-3.7-flash` | 262,144 | 262,144     | Çok modlu görüntü girişi desteği |

Step Plan (`stepfun-plan`):

| Model başvurusu                    | Bağlam  | Maks. çıktı | Notlar                         |
| ---------------------------------- | ------- | ----------- | ------------------------------ |
| `stepfun-plan/step-3.5-flash`      | 262,144 | 65,536      | Varsayılan Step Plan modeli    |
| `stepfun-plan/step-3.7-flash`      | 262,144 | 262,144     | Çok modlu görüntü girişi desteği |
| `stepfun-plan/step-3.5-flash-2603` | 262,144 | 65,536      | Ek Step Plan modeli            |

## Başlarken

<Tabs>
  <Tab title="Standard">
    Standart StepFun uç noktası üzerinden genel amaçlı kullanım için en uygunudur.

    <Steps>
      <Step title="Uç nokta bölgenizi seçin">
        | Kimlik doğrulama seçimi        | Uç nokta                      | Bölge          |
        | -------------------------------- | ----------------------------- | -------------- |
        | `stepfun-standard-api-key-intl` | `https://api.stepfun.ai/v1`  | Uluslararası   |
        | `stepfun-standard-api-key-cn`   | `https://api.stepfun.com/v1` | Çin            |
      </Step>
      <Step title="İlk kurulumu çalıştırın">
        ```bash
        openclaw onboard --auth-choice stepfun-standard-api-key-intl
        ```

        Çin uç noktası:

        ```bash
        openclaw onboard --auth-choice stepfun-standard-api-key-cn
        ```
      </Step>
      <Step title="Etkileşimsiz alternatif">
        ```bash
        openclaw onboard --auth-choice stepfun-standard-api-key-intl \
          --stepfun-api-key "$STEPFUN_API_KEY"
        ```
      </Step>
      <Step title="Modellerin kullanılabilir olduğunu doğrulayın">
        ```bash
        openclaw models list --provider stepfun
        ```
      </Step>
    </Steps>

    Varsayılan model: `stepfun/step-3.5-flash`
    Alternatif model: `stepfun/step-3.7-flash`

  </Tab>

  <Tab title="Step Plan">
    Step Plan akıl yürütme uç noktası için en uygunudur.

    <Steps>
      <Step title="Uç nokta bölgenizi seçin">
        | Kimlik doğrulama seçimi     | Uç nokta                                 | Bölge          |
        | ------------------------------ | ------------------------------------------ | -------------- |
        | `stepfun-plan-api-key-intl` | `https://api.stepfun.ai/step_plan/v1`  | Uluslararası   |
        | `stepfun-plan-api-key-cn`   | `https://api.stepfun.com/step_plan/v1` | Çin            |
      </Step>
      <Step title="İlk kurulumu çalıştırın">
        ```bash
        openclaw onboard --auth-choice stepfun-plan-api-key-intl
        ```

        Çin uç noktası:

        ```bash
        openclaw onboard --auth-choice stepfun-plan-api-key-cn
        ```
      </Step>
      <Step title="Etkileşimsiz alternatif">
        ```bash
        openclaw onboard --auth-choice stepfun-plan-api-key-intl \
          --stepfun-api-key "$STEPFUN_API_KEY"
        ```
      </Step>
      <Step title="Modellerin kullanılabilir olduğunu doğrulayın">
        ```bash
        openclaw models list --provider stepfun-plan
        ```
      </Step>
    </Steps>

    Varsayılan model: `stepfun-plan/step-3.5-flash`
    Alternatif modeller: `stepfun-plan/step-3.7-flash`, `stepfun-plan/step-3.5-flash-2603`

  </Tab>
</Tabs>

Tek bir kimlik doğrulama akışı hem `stepfun` hem de `stepfun-plan` için bölgeyle eşleşen profiller yazar; böylece tek bir ilk kurulum çalıştırmasından sonra her iki yüzey de birlikte keşfedilir.

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Tam yapılandırma: Standard sağlayıcısı">
    ```json5
    {
      env: { STEPFUN_API_KEY: "your-key" },
      agents: { defaults: { model: { primary: "stepfun/step-3.5-flash" } } },
      models: {
        mode: "merge",
        providers: {
          stepfun: {
            baseUrl: "https://api.stepfun.ai/v1",
            api: "openai-completions",
            apiKey: "${STEPFUN_API_KEY}",
            models: [
              {
                id: "step-3.7-flash",
                name: "Step 3.7 Flash",
                reasoning: true,
                input: ["text", "image"],
                thinkingLevelMap: { off: "low", minimal: "low", xhigh: "high", max: "high" },
                cost: { input: 0.2, output: 1.15, cacheRead: 0.04, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "step-3.5-flash",
                name: "Step 3.5 Flash",
                reasoning: true,
                input: ["text"],
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
  </Accordion>

  <Accordion title="Tam yapılandırma: Step Plan sağlayıcısı">
    ```json5
    {
      env: { STEPFUN_API_KEY: "your-key" },
      agents: { defaults: { model: { primary: "stepfun-plan/step-3.5-flash" } } },
      models: {
        mode: "merge",
        providers: {
          "stepfun-plan": {
            baseUrl: "https://api.stepfun.ai/step_plan/v1",
            api: "openai-completions",
            apiKey: "${STEPFUN_API_KEY}",
            models: [
              {
                id: "step-3.7-flash",
                name: "Step 3.7 Flash",
                reasoning: true,
                input: ["text", "image"],
                thinkingLevelMap: { off: "low", minimal: "low", xhigh: "high", max: "high" },
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "step-3.5-flash",
                name: "Step 3.5 Flash",
                reasoning: true,
                input: ["text"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 65536,
              },
              {
                id: "step-3.5-flash-2603",
                name: "Step 3.5 Flash 2603",
                reasoning: true,
                input: ["text"],
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
  </Accordion>

  <Accordion title="Notlar">
    - `step-3.7-flash`, OpenClaw üzerinden metin ve görüntü girişini kabul eder. StepFun API'si videoyu da destekler ancak video henüz OpenClaw'da bir model giriş modalitesi değildir.
    - Step 3.7; `low`, `medium` ve `high` akıl yürütme çabasını destekler. Modelin akıl yürütmesiz bir modu olmadığından `/think off`, `low` ile eşlenir.
    - `step-3.5-flash-2603` şu anda yalnızca `stepfun-plan` üzerinde sunulur.
    - Modelleri incelemek veya değiştirmek için `openclaw models list` ve `openclaw models set <provider/model>` kullanın.

  </Accordion>
</AccordionGroup>

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="Model sağlayıcıları" href="/tr/concepts/model-providers" icon="layers">
    Tüm sağlayıcılara, model başvurularına ve yük devretme davranışına genel bakış.
  </Card>
  <Card title="Yapılandırma başvurusu" href="/tr/gateway/configuration-reference" icon="gear">
    Sağlayıcılar, modeller ve pluginler için tam yapılandırma şeması.
  </Card>
  <Card title="Modeller CLI'sı" href="/tr/concepts/models" icon="brain">
    Modellerin nasıl seçileceği ve yapılandırılacağı.
  </Card>
  <Card title="StepFun Platformu" href="https://platform.stepfun.com" icon="globe">
    StepFun API anahtarı yönetimi ve belgeleri.
  </Card>
</CardGroup>
