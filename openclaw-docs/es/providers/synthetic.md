---
read_when:
    - Quiere usar Synthetic como proveedor de modelos
    - Necesita configurar una clave de API o una URL base de Synthetic.
summary: Usar la API compatible con Anthropic de Synthetic en OpenClaw
title: Synthetic
x-i18n:
    generated_at: "2026-07-26T04:56:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3f6cc89a7b837f57555d176ce78e62a39095d4ef0765c96b6b7b93ffebd7388
    source_path: providers/synthetic.md
    workflow: 16
---

[Synthetic](https://synthetic.new) expone endpoints compatibles con Anthropic.
OpenClaw lo incluye como el proveedor `synthetic` y utiliza la API
Messages de Anthropic.

| Propiedad | Valor                                 |
| -------- | ------------------------------------- |
| Proveedor | `synthetic`                           |
| Autenticación     | `SYNTHETIC_API_KEY`                   |
| API      | Anthropic Messages                    |
| URL base | `https://api.synthetic.new/anthropic` |

## Primeros pasos

<Steps>
  <Step title="Obtener una clave de API">
    Obtenga una `SYNTHETIC_API_KEY` de su cuenta de Synthetic o permita que el proceso de incorporación
    se la solicite.
  </Step>
  <Step title="Ejecutar el proceso de incorporación">
    ```bash
    openclaw onboard --auth-choice synthetic-api-key
    ```
  </Step>
  <Step title="Verificar el modelo predeterminado">
    El proceso de incorporación establece el siguiente modelo predeterminado:
    ```text
    synthetic/hf:MiniMaxAI/MiniMax-M3
    ```
  </Step>
</Steps>

<Warning>
El cliente Anthropic de OpenClaw añade `/v1` automáticamente a la URL base, por lo que debe utilizar
`https://api.synthetic.new/anthropic` (no `/anthropic/v1`). Si Synthetic
cambia su URL base, sustituya `models.providers.synthetic.baseUrl`.
</Warning>

## Ejemplo de configuración

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

## Catálogo integrado

Todos los modelos de Synthetic tienen un coste `0` (entrada/salida/caché). Consulte la
[lista de modelos actual](https://dev.synthetic.new/docs/api/models) de Synthetic para conocer la disponibilidad del servicio.

| ID del modelo                                            | Ventana de contexto | Máximo de tokens | Razonamiento | Entrada        |
| --------------------------------------------------- | -------------- | ---------- | --------- | ------------ |
| `hf:MiniMaxAI/MiniMax-M3`                           | 262,144        | 65,536     | sí       | texto + imagen |
| `hf:moonshotai/Kimi-K2.7-Code`                      | 262,144        | 8,192      | sí       | texto + imagen |
| `hf:nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4` | 262,144        | 8,192      | sí       | texto         |
| `hf:openai/gpt-oss-120b`                            | 131,072        | 8,192      | sí       | texto         |
| `hf:Qwen/Qwen3.6-27B`                               | 262,144        | 81,920     | sí       | texto + imagen |
| `hf:zai-org/GLM-4.7-Flash`                          | 196,608        | 131,072    | sí       | texto         |
| `hf:zai-org/GLM-5.2`                                | 524,288        | 131,072    | sí       | texto         |

<Tip>
Las referencias de modelos utilizan el formato `synthetic/<modelId>`. Utilice
`openclaw models list --provider synthetic` para ver todos los modelos disponibles en su
cuenta.
</Tip>

<AccordionGroup>
  <Accordion title="Lista de modelos permitidos">
    Si habilita una lista de modelos permitidos (`agents.defaults.modelPolicy.allow`), añada todos los
    modelos de Synthetic que tenga previsto utilizar. Los modelos que no estén en la lista se ocultan
    al agente.
  </Accordion>

  <Accordion title="Sustitución de la URL base">
    Si Synthetic cambia su endpoint de API, sustituya la URL base:

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

    OpenClaw sigue añadiendo `/v1` automáticamente.

  </Accordion>
</AccordionGroup>

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Proveedores de modelos" href="/es/concepts/model-providers" icon="layers">
    Reglas de proveedores, referencias de modelos y comportamiento de conmutación por error.
  </Card>
  <Card title="Referencia de configuración" href="/es/gateway/configuration-reference" icon="gear">
    Esquema de configuración completo, incluidos los ajustes de proveedores.
  </Card>
  <Card title="Synthetic" href="https://synthetic.new" icon="arrow-up-right-from-square">
    Panel de Synthetic y documentación de la API.
  </Card>
</CardGroup>
