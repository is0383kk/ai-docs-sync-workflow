---
read_when:
    - Quieres usar modelos de StepFun en OpenClaw
    - Necesita orientación para configurar StepFun
summary: Usa modelos de StepFun con OpenClaw
title: StepFun
x-i18n:
    generated_at: "2026-07-26T04:49:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 462a2588f15e8d6188914e238a3e472052d0da1da151751adecdb63cf009fc64
    source_path: providers/stepfun.md
    workflow: 16
---

StepFun se distribuye como un Plugin oficial externo (`@openclaw/stepfun-provider`) con dos identificadores de proveedor:

- `stepfun` para el endpoint estándar
- `stepfun-plan` para el endpoint de Step Plan

<Warning>
Standard y Step Plan son **proveedores independientes** con endpoints y prefijos de referencia de modelo diferentes (`stepfun/...` frente a `stepfun-plan/...`). Use una clave de China con los endpoints `.com` y una clave global con los endpoints `.ai`.
</Warning>

## Instalar el Plugin

```bash
openclaw plugins install @openclaw/stepfun-provider
openclaw gateway restart
```

## Resumen de regiones y endpoints

| Endpoint  | China (`.com`)                         | Global (`.ai`)                        |
| --------- | -------------------------------------- | ------------------------------------- |
| Estándar  | `https://api.stepfun.com/v1`           | `https://api.stepfun.ai/v1`           |
| Step Plan | `https://api.stepfun.com/step_plan/v1` | `https://api.stepfun.ai/step_plan/v1` |

Variable de entorno de autenticación: `STEPFUN_API_KEY`

## Catálogo integrado

Estándar (`stepfun`):

| Referencia de modelo     | Contexto | Salida máxima | Notas                          |
| ------------------------ | -------- | ------------- | ------------------------------ |
| `stepfun/step-3.5-flash` | 262,144 | 65,536     | Modelo estándar predeterminado |
| `stepfun/step-3.7-flash` | 262,144 | 262,144    | Admite entrada de imágenes multimodal |

Step Plan (`stepfun-plan`):

| Referencia de modelo               | Contexto | Salida máxima | Notas                          |
| ---------------------------------- | -------- | ------------- | ------------------------------ |
| `stepfun-plan/step-3.5-flash`      | 262,144 | 65,536     | Modelo de Step Plan predeterminado |
| `stepfun-plan/step-3.7-flash`      | 262,144 | 262,144    | Admite entrada de imágenes multimodal |
| `stepfun-plan/step-3.5-flash-2603` | 262,144 | 65,536     | Modelo adicional de Step Plan  |

## Primeros pasos

<Tabs>
  <Tab title="Estándar">
    La mejor opción para uso general mediante el endpoint estándar de StepFun.

    <Steps>
      <Step title="Elija la región del endpoint">
        | Opción de autenticación          | Endpoint                     | Región        |
        | -------------------------------- | ---------------------------- | ------------- |
        | `stepfun-standard-api-key-intl` | `https://api.stepfun.ai/v1`  | Internacional |
        | `stepfun-standard-api-key-cn`   | `https://api.stepfun.com/v1` | China         |
      </Step>
      <Step title="Ejecute la incorporación">
        ```bash
        openclaw onboard --auth-choice stepfun-standard-api-key-intl
        ```

        Endpoint de China:

        ```bash
        openclaw onboard --auth-choice stepfun-standard-api-key-cn
        ```
      </Step>
      <Step title="Alternativa no interactiva">
        ```bash
        openclaw onboard --auth-choice stepfun-standard-api-key-intl \
          --stepfun-api-key "$STEPFUN_API_KEY"
        ```
      </Step>
      <Step title="Compruebe que los modelos estén disponibles">
        ```bash
        openclaw models list --provider stepfun
        ```
      </Step>
    </Steps>

    Modelo predeterminado: `stepfun/step-3.5-flash`
    Modelo alternativo: `stepfun/step-3.7-flash`

  </Tab>

  <Tab title="Step Plan">
    La mejor opción para el endpoint de razonamiento de Step Plan.

    <Steps>
      <Step title="Elija la región del endpoint">
        | Opción de autenticación       | Endpoint                                | Región        |
        | ----------------------------- | --------------------------------------- | ------------- |
        | `stepfun-plan-api-key-intl` | `https://api.stepfun.ai/step_plan/v1`  | Internacional |
        | `stepfun-plan-api-key-cn`   | `https://api.stepfun.com/step_plan/v1` | China         |
      </Step>
      <Step title="Ejecute la incorporación">
        ```bash
        openclaw onboard --auth-choice stepfun-plan-api-key-intl
        ```

        Endpoint de China:

        ```bash
        openclaw onboard --auth-choice stepfun-plan-api-key-cn
        ```
      </Step>
      <Step title="Alternativa no interactiva">
        ```bash
        openclaw onboard --auth-choice stepfun-plan-api-key-intl \
          --stepfun-api-key "$STEPFUN_API_KEY"
        ```
      </Step>
      <Step title="Compruebe que los modelos estén disponibles">
        ```bash
        openclaw models list --provider stepfun-plan
        ```
      </Step>
    </Steps>

    Modelo predeterminado: `stepfun-plan/step-3.5-flash`
    Modelos alternativos: `stepfun-plan/step-3.7-flash`, `stepfun-plan/step-3.5-flash-2603`

  </Tab>
</Tabs>

Un único flujo de autenticación escribe perfiles correspondientes a la región tanto para `stepfun` como para `stepfun-plan`, por lo que ambas superficies se detectan conjuntamente después de una sola ejecución de incorporación.

## Configuración avanzada

<AccordionGroup>
  <Accordion title="Configuración completa: proveedor estándar">
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

  <Accordion title="Configuración completa: proveedor de Step Plan">
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

  <Accordion title="Notas">
    - `step-3.7-flash` acepta entradas de texto e imágenes mediante OpenClaw. La API de StepFun también admite vídeo, que aún no es una modalidad de entrada de modelos en OpenClaw.
    - Step 3.7 admite los niveles de esfuerzo de razonamiento `low`, `medium` y `high`. Dado que el modelo no tiene un modo sin razonamiento, `/think off` se asigna a `low`.
    - `step-3.5-flash-2603` actualmente solo está disponible en `stepfun-plan`.
    - Use `openclaw models list` y `openclaw models set <provider/model>` para consultar o cambiar los modelos.

  </Accordion>
</AccordionGroup>

## Recursos relacionados

<CardGroup cols={2}>
  <Card title="Proveedores de modelos" href="/es/concepts/model-providers" icon="layers">
    Resumen de todos los proveedores, las referencias de modelos y el comportamiento de conmutación por error.
  </Card>
  <Card title="Referencia de configuración" href="/es/gateway/configuration-reference" icon="gear">
    Esquema completo de configuración para proveedores, modelos y plugins.
  </Card>
  <Card title="CLI de modelos" href="/es/concepts/models" icon="brain">
    Cómo elegir y configurar modelos.
  </Card>
  <Card title="Plataforma StepFun" href="https://platform.stepfun.com" icon="globe">
    Gestión de claves de API y documentación de StepFun.
  </Card>
</CardGroup>
