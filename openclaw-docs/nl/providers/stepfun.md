---
read_when:
    - Je wilt StepFun-modellen in OpenClaw
    - Je hebt hulp nodig bij het instellen van StepFun
summary: Gebruik StepFun-modellen met OpenClaw
title: StepFun
x-i18n:
    generated_at: "2026-07-27T05:20:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 462a2588f15e8d6188914e238a3e472052d0da1da151751adecdb63cf009fc64
    source_path: providers/stepfun.md
    workflow: 16
---

StepFun wordt geleverd als een externe officiële plugin (`@openclaw/stepfun-provider`) met twee provider-id's:

- `stepfun` voor het standaardendpoint
- `stepfun-plan` voor het Step Plan-endpoint

<Warning>
Standard en Step Plan zijn **afzonderlijke providers** met verschillende endpoints en modelreferentievoorvoegsels (`stepfun/...` versus `stepfun-plan/...`). Gebruik een sleutel voor China met de `.com`-endpoints en een globale sleutel met de `.ai`-endpoints.
</Warning>

## Plugin installeren

```bash
openclaw plugins install @openclaw/stepfun-provider
openclaw gateway restart
```

## Overzicht van regio's en endpoints

| Endpoint  | China (`.com`)                         | Globaal (`.ai`)                        |
| --------- | -------------------------------------- | ------------------------------------- |
| Standard  | `https://api.stepfun.com/v1`           | `https://api.stepfun.ai/v1`           |
| Step Plan | `https://api.stepfun.com/step_plan/v1` | `https://api.stepfun.ai/step_plan/v1` |

Omgevingsvariabele voor authenticatie: `STEPFUN_API_KEY`

## Ingebouwde catalogus

Standard (`stepfun`):

| Modelreferentie          | Context | Maximale uitvoer | Opmerkingen                    |
| ------------------------ | ------- | ---------------- | ------------------------------ |
| `stepfun/step-3.5-flash` | 262,144 | 65,536     | Standaardmodel                 |
| `stepfun/step-3.7-flash` | 262,144 | 262,144    | Ondersteuning voor multimodale afbeeldingsinvoer |

Step Plan (`stepfun-plan`):

| Modelreferentie                    | Context | Maximale uitvoer | Opmerkingen                    |
| ---------------------------------- | ------- | ---------------- | ------------------------------ |
| `stepfun-plan/step-3.5-flash`      | 262,144 | 65,536     | Standaardmodel voor Step Plan  |
| `stepfun-plan/step-3.7-flash`      | 262,144 | 262,144    | Ondersteuning voor multimodale afbeeldingsinvoer |
| `stepfun-plan/step-3.5-flash-2603` | 262,144 | 65,536     | Aanvullend Step Plan-model     |

## Aan de slag

<Tabs>
  <Tab title="Standard">
    Het meest geschikt voor algemeen gebruik via het standaardendpoint van StepFun.

    <Steps>
      <Step title="Kies de regio van je endpoint">
        | Authenticatiekeuze             | Endpoint                     | Regio          |
        | -------------------------------- | ----------------------------- | -------------- |
        | `stepfun-standard-api-key-intl` | `https://api.stepfun.ai/v1`  | Internationaal |
        | `stepfun-standard-api-key-cn`   | `https://api.stepfun.com/v1` | China          |
      </Step>
      <Step title="Voer de onboarding uit">
        ```bash
        openclaw onboard --auth-choice stepfun-standard-api-key-intl
        ```

        Endpoint voor China:

        ```bash
        openclaw onboard --auth-choice stepfun-standard-api-key-cn
        ```
      </Step>
      <Step title="Niet-interactief alternatief">
        ```bash
        openclaw onboard --auth-choice stepfun-standard-api-key-intl \
          --stepfun-api-key "$STEPFUN_API_KEY"
        ```
      </Step>
      <Step title="Controleer of de modellen beschikbaar zijn">
        ```bash
        openclaw models list --provider stepfun
        ```
      </Step>
    </Steps>

    Standaardmodel: `stepfun/step-3.5-flash`
    Alternatief model: `stepfun/step-3.7-flash`

  </Tab>

  <Tab title="Step Plan">
    Het meest geschikt voor het redeneerendpoint van Step Plan.

    <Steps>
      <Step title="Kies de regio van je endpoint">
        | Authenticatiekeuze          | Endpoint                                | Regio          |
        | ------------------------------ | ------------------------------------------ | -------------- |
        | `stepfun-plan-api-key-intl` | `https://api.stepfun.ai/step_plan/v1`  | Internationaal |
        | `stepfun-plan-api-key-cn`   | `https://api.stepfun.com/step_plan/v1` | China          |
      </Step>
      <Step title="Voer de onboarding uit">
        ```bash
        openclaw onboard --auth-choice stepfun-plan-api-key-intl
        ```

        Endpoint voor China:

        ```bash
        openclaw onboard --auth-choice stepfun-plan-api-key-cn
        ```
      </Step>
      <Step title="Niet-interactief alternatief">
        ```bash
        openclaw onboard --auth-choice stepfun-plan-api-key-intl \
          --stepfun-api-key "$STEPFUN_API_KEY"
        ```
      </Step>
      <Step title="Controleer of de modellen beschikbaar zijn">
        ```bash
        openclaw models list --provider stepfun-plan
        ```
      </Step>
    </Steps>

    Standaardmodel: `stepfun-plan/step-3.5-flash`
    Alternatieve modellen: `stepfun-plan/step-3.7-flash`, `stepfun-plan/step-3.5-flash-2603`

  </Tab>
</Tabs>

Eén authenticatieflow schrijft profielen met de overeenkomende regio voor zowel `stepfun` als `stepfun-plan`, zodat beide oppervlakken samen worden gedetecteerd na één onboarding.

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Volledige configuratie: Standard-provider">
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

  <Accordion title="Volledige configuratie: Step Plan-provider">
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

  <Accordion title="Opmerkingen">
    - `step-3.7-flash` accepteert tekst- en afbeeldingsinvoer via OpenClaw. De API van StepFun ondersteunt ook video, wat nog geen modelinvoermodaliteit is in OpenClaw.
    - Step 3.7 ondersteunt `low`, `medium` en `high` als redeneerinspanning. Omdat het model geen modus zonder redeneren heeft, wordt `/think off` toegewezen aan `low`.
    - `step-3.5-flash-2603` is momenteel alleen beschikbaar op `stepfun-plan`.
    - Gebruik `openclaw models list` en `openclaw models set <provider/model>` om modellen te bekijken of te wisselen.

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelproviders" href="/nl/concepts/model-providers" icon="layers">
    Overzicht van alle providers, modelreferenties en failovergedrag.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/configuration-reference" icon="gear">
    Volledig configuratieschema voor providers, modellen en plugins.
  </Card>
  <Card title="CLI voor modellen" href="/nl/concepts/models" icon="brain">
    Modellen kiezen en configureren.
  </Card>
  <Card title="StepFun-platform" href="https://platform.stepfun.com" icon="globe">
    Beheer en documentatie van StepFun-API-sleutels.
  </Card>
</CardGroup>
