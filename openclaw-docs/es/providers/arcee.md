---
read_when:
    - Quieres usar Arcee AI con OpenClaw
    - Se necesita la variable de entorno de la clave de API o la opción de autenticación de la CLI
summary: Configuración de Arcee AI (autenticación + selección de modelo)
title: Arcee AI
x-i18n:
    generated_at: "2026-07-26T05:24:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a4c2fc7b8d86dd0d2a300dfc48951657cbcfcd9250016f52c1804777b2966e11
    source_path: providers/arcee.md
    workflow: 16
---

[Arcee AI](https://arcee.ai) ofrece la familia Trinity de modelos de mezcla de expertos mediante una API compatible con OpenAI. Todos los modelos Trinity tienen licencia Apache 2.0. Arcee es un plugin oficial de OpenClaw que no viene incluido con el núcleo, por lo que debe instalarse antes de la incorporación.

Acceda a los modelos de Arcee directamente mediante la plataforma Arcee o a través de [OpenRouter](/es/providers/openrouter).

| Propiedad | Valor                                                                                 |
| -------- | ------------------------------------------------------------------------------------- |
| Proveedor | `arcee`                                                                               |
| Autenticación     | `ARCEEAI_API_KEY` (directa) o `OPENROUTER_API_KEY` (mediante OpenRouter)                   |
| API      | Compatible con OpenAI                                                                     |
| URL base | `https://api.arcee.ai/api/v1` (directa) o `https://openrouter.ai/api/v1` (OpenRouter) |

## Instalar el plugin

```bash
openclaw plugins install @openclaw/arcee-provider
openclaw gateway restart
```

## Primeros pasos

<Tabs>
  <Tab title="Directamente (plataforma Arcee)">
    <Steps>
      <Step title="Obtener una clave de API">
        Cree una clave de API en [Arcee AI](https://chat.arcee.ai/).
      </Step>
      <Step title="Ejecutar la incorporación">
        ```bash
        openclaw onboard --auth-choice arceeai-api-key
        ```
      </Step>
      <Step title="Establecer un modelo predeterminado">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```
      </Step>
    </Steps>
  </Tab>

  <Tab title="Mediante OpenRouter">
    <Steps>
      <Step title="Obtener una clave de API">
        Cree una clave de API en [OpenRouter](https://openrouter.ai/keys).
      </Step>
      <Step title="Ejecutar la incorporación">
        ```bash
        openclaw onboard --auth-choice arceeai-openrouter
        ```
      </Step>
      <Step title="Establecer un modelo predeterminado">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```

        Las mismas referencias de modelo funcionan tanto para la configuración directa como para la de OpenRouter.
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Configuración no interactiva

<Tabs>
  <Tab title="Directamente (plataforma Arcee)">
    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-api-key \
      --arceeai-api-key "$ARCEEAI_API_KEY"
    ```
  </Tab>

  <Tab title="Mediante OpenRouter">
    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-openrouter \
      --openrouter-api-key "$OPENROUTER_API_KEY"
    ```
  </Tab>
</Tabs>

## Catálogo directo de Arcee

| Referencia del modelo                      | Nombre                   | Entrada | Contexto | Salida máxima | Coste (entrada/salida por 1 M) | Herramientas | Notas                                     |
| ------------------------------ | ---------------------- | ----- | ------- | ---------- | -------------------- | ----- | ----------------------------------------- |
| `arcee/trinity-large-thinking` | Trinity Large Thinking | texto  | 256K    | 80K        | $0.25 / $0.90        | No    | Modelo predeterminado; razonamiento ampliado          |
| `arcee/trinity-large-preview`  | Trinity Large Preview  | texto  | 128K    | 16K        | $0.25 / $1.00        | Sí   | Uso general; 400B parámetros, 13B activos  |
| `arcee/trinity-mini`           | Trinity Mini 26B       | texto  | 128K    | 80K        | $0.045 / $0.15       | Sí   | Rápido y rentable; llamada a funciones |

<Tip>
El ajuste preestablecido de incorporación establece `arcee/trinity-large-thinking` como modelo predeterminado.
</Tip>

## Catálogo de OpenRouter

La incorporación de OpenRouter expone `arcee/trinity-large-preview` y `arcee/trinity-large-thinking`. OpenClaw conserva esas referencias de modelo calificadas por el proveedor en la configuración y envía los identificadores de ejecución canónicos `arcee-ai/*` de OpenRouter. Trinity Mini ya no está disponible mediante OpenRouter; utilice la API directa de Arcee para ese modelo.

## Funciones compatibles

| Función                                       | Compatibilidad                                    |
| --------------------------------------------- | -------------------------------------------- |
| Transmisión                                     | Sí                                          |
| Uso de herramientas / llamada a funciones                   | Sí (Trinity Mini, Trinity Large Preview)    |
| Salida estructurada (modo JSON y esquema JSON) | Sí                                          |
| Razonamiento ampliado                             | Sí (Trinity Large Thinking; herramientas desactivadas) |

<AccordionGroup>
  <Accordion title="Nota sobre el entorno">
    Si el Gateway se ejecuta como demonio (launchd/systemd), asegúrese de que `ARCEEAI_API_KEY`
    (o `OPENROUTER_API_KEY`) esté disponible para ese proceso, por ejemplo, en
    `~/.openclaw/.env` o mediante `env.shellEnv`.
  </Accordion>

  <Accordion title="Enrutamiento de OpenRouter">
    OpenRouter utiliza la misma referencia de modelo `arcee/trinity-large-thinking` de OpenClaw.
    OpenClaw la enruta con el identificador de ejecución canónico `arcee-ai/trinity-large-thinking`
    de OpenRouter. Consulte la
    [documentación del proveedor OpenRouter](/es/providers/openrouter) para obtener detalles
    de configuración específicos de OpenRouter.
  </Accordion>
</AccordionGroup>

## Relacionado

<CardGroup cols={2}>
  <Card title="OpenRouter" href="/es/providers/openrouter" icon="shuffle">
    Acceda a los modelos de Arcee y a muchos otros con una única clave de API.
  </Card>
  <Card title="Selección de modelos" href="/es/concepts/model-providers" icon="layers">
    Elección de proveedores, referencias de modelos y comportamiento de conmutación por error.
  </Card>
</CardGroup>
