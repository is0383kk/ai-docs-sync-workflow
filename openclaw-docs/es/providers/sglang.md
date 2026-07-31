---
read_when:
    - Se desea ejecutar OpenClaw con un servidor SGLang local
    - Quieres endpoints `/v1` compatibles con OpenAI con tus propios modelos
summary: Ejecutar OpenClaw con SGLang (servidor autoalojado compatible con OpenAI)
title: SGLang
x-i18n:
    generated_at: "2026-07-26T04:52:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 54a7805315a7d65fdd2c7c9b6836aa2faccc88db7802cce0ba8c2d4a1aac9d65
    source_path: providers/sglang.md
    workflow: 16
---

SGLang sirve modelos de pesos abiertos mediante una API HTTP compatible con OpenAI. OpenClaw se conecta a SGLang mediante la familia de proveedores `openai-completions`, con detección automática de los modelos disponibles.

| Propiedad                 | Valor                                                        |
| ------------------------- | ------------------------------------------------------------ |
| Id. del proveedor         | `sglang`                                                     |
| Plugin                    | incluido, `enabledByDefault: true`                            |
| Variable de entorno de autenticación | `SGLANG_API_KEY` (cualquier valor no vacío si el servidor no tiene autenticación) |
| Opción de incorporación   | `--auth-choice sglang`                                       |
| API                       | Compatible con OpenAI (`openai-completions`)                     |
| URL base predeterminada   | `http://127.0.0.1:30000/v1`                                  |
| Marcador de posición del modelo predeterminado | `sglang/Qwen/Qwen3-8B`                                       |
| Uso de streaming          | Sí (`supportsStreamingUsage: true`)                         |
| Precios                   | Marcado como externo y gratuito (`modelPricing.external: false`)        |

OpenClaw también **detecta automáticamente** los modelos disponibles en SGLang al habilitarlo con `SGLANG_API_KEY`. Use `sglang/*` en `agents.defaults.models` para mantener la detección dinámica cuando también configure una URL base personalizada de SGLang. Consulte [Detección de modelos (proveedor implícito)](#model-discovery-implicit-provider) más adelante.

## Primeros pasos

<Steps>
  <Step title="Iniciar SGLang">
    Inicie SGLang con un servidor compatible con OpenAI. La URL base debe exponer
    los endpoints `/v1` (por ejemplo, `/v1/models`, `/v1/chat/completions`). SGLang
    suele ejecutarse en:

    - `http://127.0.0.1:30000/v1`

  </Step>
  <Step title="Establecer una clave de API">
    Cualquier valor funciona si no hay autenticación configurada en el servidor:

    ```bash
    export SGLANG_API_KEY="sglang-local"
    ```

  </Step>
  <Step title="Ejecutar la incorporación o establecer un modelo directamente">
    ```bash
    openclaw onboard
    ```

    También puede configurar el modelo manualmente:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "sglang/your-model-id" },
        },
      },
    }
    ```

  </Step>
</Steps>

## Detección de modelos (proveedor implícito)

Cuando se establece `SGLANG_API_KEY` (o existe un perfil de autenticación) y **no se**
define `models.providers.sglang`, OpenClaw consulta:

- `GET http://127.0.0.1:30000/v1/models`

y convierte los identificadores devueltos en entradas de modelos.

<Note>
Si establece `models.providers.sglang` explícitamente, OpenClaw utiliza de forma predeterminada
los modelos declarados. Añada `"sglang/*": {}` a `agents.defaults.models` cuando
desee que OpenClaw consulte el endpoint `/models` de ese proveedor configurado e incluya
todos los modelos de SGLang anunciados.
</Note>

## Configuración explícita (modelos manuales)

Use una configuración explícita cuando:

- SGLang se ejecute en otro host o puerto.
- Desee fijar los valores de `contextWindow`/`maxTokens`.
- El servidor requiera una clave de API real (o desee controlar los encabezados).

```json5
{
  models: {
    providers: {
      sglang: {
        baseUrl: "http://127.0.0.1:30000/v1",
        apiKey: "${SGLANG_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "your-model-id",
            name: "Local SGLang Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

## Configuración avanzada

<AccordionGroup>
  <Accordion title="Comportamiento de tipo proxy">
    SGLang se trata como un backend `/v1` compatible con OpenAI y de tipo proxy, no como un
    endpoint nativo de OpenAI.

    | Comportamiento | SGLang |
    |----------|--------|
    | Adaptación de solicitudes exclusiva de OpenAI | No se aplica |
    | `service_tier`, Responses `store`, indicaciones de caché de prompts | No se envían |
    | Adaptación de la carga útil para compatibilidad con el razonamiento | No se aplica |
    | Encabezados de atribución ocultos (`originator`, `version`, `User-Agent`) | No se inyectan en URL base personalizadas de SGLang |

  </Accordion>

  <Accordion title="Solución de problemas">
    **No se puede acceder al servidor**

    Verifique que el servidor esté en ejecución y responda:

    ```bash
    curl http://127.0.0.1:30000/v1/models
    ```

    **Errores de autenticación**

    Si las solicitudes fallan con errores de autenticación, establezca un valor real para `SGLANG_API_KEY` que coincida con
    la configuración del servidor o configure el proveedor explícitamente en
    `models.providers.sglang`.

    <Tip>
    Si ejecuta SGLang sin autenticación, cualquier valor no vacío para
    `SGLANG_API_KEY` es suficiente para habilitar la detección de modelos.
    </Tip>

  </Accordion>
</AccordionGroup>

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Selección de modelos" href="/es/concepts/model-providers" icon="layers">
    Selección de proveedores, referencias de modelos y comportamiento de conmutación por error.
  </Card>
  <Card title="Referencia de configuración" href="/es/gateway/configuration-reference" icon="gear">
    Esquema de configuración completo, incluidas las entradas de proveedores.
  </Card>
</CardGroup>
