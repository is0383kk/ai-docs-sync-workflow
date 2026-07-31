---
read_when:
    - Quieres ejecutar OpenClaw con antirez/ds4
    - Quieres un backend local de DeepSeek V4 Flash con llamadas a herramientas
    - Necesitas la configuración de OpenClaw para ds4-server
summary: Ejecuta OpenClaw mediante ds4, un servidor local compatible con OpenAI para DeepSeek V4 Flash
title: ds4
x-i18n:
    generated_at: "2026-07-26T05:26:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: be449813295648694625ef8003b3f4b12903535b74816916ca5af0695174fbf4
    source_path: providers/ds4.md
    workflow: 16
---

[ds4](https://github.com/antirez/ds4) sirve DeepSeek V4 Flash desde un backend
Metal local con una API `/v1` compatible con OpenAI. OpenClaw se conecta a ds4
mediante la familia de proveedores genérica `openai-completions`.

ds4 no es un plugin de proveedor incluido con OpenClaw. Configúrelo en
`models.providers.ds4` y, a continuación, seleccione `ds4/deepseek-v4-flash`.

| Propiedad        | Valor                                                     |
| ---------------- | --------------------------------------------------------- |
| Id. del proveedor | `ds4`                                                     |
| Plugin           | ninguno (solo configuración)                              |
| API              | Chat Completions compatible con OpenAI (`openai-completions`) |
| URL base         | `http://127.0.0.1:18000/v1` (sugerida)                   |
| Id. del modelo   | `deepseek-v4-flash`                                       |
| Llamadas a herramientas | `tools` / `tool_calls` al estilo de OpenAI                       |
| Razonamiento     | `thinking` y `reasoning_effort` al estilo de DeepSeek          |

## Requisitos

- macOS con compatibilidad con Metal.
- Una copia de trabajo de ds4 con `ds4-server` y el archivo GGUF de DeepSeek V4 Flash.
- Memoria suficiente para el contexto elegido; los valores de `--ctx` más grandes asignan más
  memoria KV al iniciar el servidor.

<Warning>
Los turnos del agente de OpenClaw incluyen esquemas de herramientas y contexto del espacio de trabajo. Un contexto
pequeño como `--ctx 4096` puede superar las pruebas directas con curl, pero fallar en ejecuciones completas del agente con
`500 prompt exceeds context`. Use al menos `--ctx 32768` para las
pruebas de humo del agente y las herramientas. Use `--ctx 393216` solo si dispone de memoria suficiente y para habilitar
Think Max de ds4.
</Warning>

## Inicio rápido

<Steps>
  <Step title="Iniciar ds4-server">
    Sustituya `<DS4_DIR>` por la ruta de la copia de trabajo de ds4.

    ```bash
    <DS4_DIR>/ds4-server \
      --model <DS4_DIR>/ds4flash.gguf \
      --host 127.0.0.1 \
      --port 18000 \
      --ctx 32768 \
      --tokens 128
    ```

  </Step>
  <Step title="Verificar el endpoint compatible con OpenAI">
    ```bash
    curl http://127.0.0.1:18000/v1/models
    ```

    La respuesta debe incluir `deepseek-v4-flash`.

  </Step>
  <Step title="Añadir la configuración del proveedor de OpenClaw">
    Añada la configuración de [Configuración completa](#full-config) y, a continuación, ejecute una comprobación
    puntual del modelo:

    ```bash
    openclaw infer model run \
      --local \
      --model ds4/deepseek-v4-flash \
      --thinking off \
      --prompt "Responde exactamente con: openclaw-ds4-ok" \
      --json
    ```

  </Step>
</Steps>

## Configuración completa

Use esta configuración cuando ds4 ya se esté ejecutando en `127.0.0.1:18000`.

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

Mantenga `contextWindow` alineado con `ds4-server --ctx`. Mantenga `maxTokens` alineado
con `--tokens`, salvo que se quiera que OpenClaw solicite intencionadamente menos salida
que el valor predeterminado del servidor.

## Inicio bajo demanda

OpenClaw puede iniciar ds4 solo cuando se selecciona un modelo `ds4/...`. Añada
`localService` a la misma entrada del proveedor:

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

`command` debe ser una ruta absoluta a un ejecutable. No se utilizan la búsqueda del shell ni la expansión de
`~`. Consulte [Servicios de modelos locales](/es/gateway/local-model-services) para conocer
todos los campos de `localService`.

## Think Max

ds4 aplica Think Max solo cuando se cumplen ambas condiciones:

- `ds4-server` comienza con `--ctx 393216` o un valor superior.
- La solicitud usa `reasoning_effort: "max"` (o el campo de esfuerzo equivalente de ds4).

Si se ejecuta ese contexto de gran tamaño, actualice tanto las opciones del servidor como los metadatos del modelo
de OpenClaw:

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

## Prueba

Comprobación HTTP directa, sin pasar por OpenClaw:

```bash
curl http://127.0.0.1:18000/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"Responde exactamente con: ds4-ok"}],"max_tokens":16,"stream":false,"thinking":{"type":"disabled"}}'
```

Enrutamiento del modelo de OpenClaw (igual que la comprobación del inicio rápido):

```bash
openclaw infer model run \
  --local \
  --model ds4/deepseek-v4-flash \
  --thinking off \
  --prompt "Responde exactamente con: openclaw-ds4-ok" \
  --json
```

Prueba de humo completa del agente y de las llamadas a herramientas, con un contexto de al menos 32768:

```bash
openclaw agent \
  --local \
  --session-id ds4-tool-smoke \
  --model ds4/deepseek-v4-flash \
  --thinking off \
  --message "Usa una vez el comando de shell pwd y luego responde exactamente con: tool-ok <output>" \
  --json \
  --timeout 240
```

Resultado esperado:

- `executionTrace.winnerProvider` es `ds4`
- `executionTrace.winnerModel` es `deepseek-v4-flash`
- `toolSummary.calls` es al menos `1`
- `finalAssistantVisibleText` comienza con `tool-ok`

## Solución de problemas

<AccordionGroup>
  <Accordion title="curl /v1/models no puede conectarse">
    ds4 no se está ejecutando o no está vinculado al host/puerto de `baseUrl`. Inicie
    `ds4-server` y vuelva a intentarlo:

    ```bash
    curl http://127.0.0.1:18000/v1/models
    ```

  </Accordion>

  <Accordion title="500: el prompt supera el contexto">
    El valor configurado de `--ctx` es demasiado pequeño para el turno de OpenClaw. Aumente
    `ds4-server --ctx` y, a continuación, actualice `models.providers.ds4.models[].contextWindow`
    para que coincida. Los turnos completos del agente con herramientas necesitan bastante más contexto que una
    solicitud directa de curl con un solo mensaje.
  </Accordion>

  <Accordion title="Think Max no se activa">
    ds4 solo usa Think Max cuando `--ctx` es al menos `393216` y la solicitud
    pide `reasoning_effort: "max"`. Los contextos más pequeños recurren al razonamiento
    alto.
  </Accordion>

  <Accordion title="La primera solicitud es lenta">
    ds4 tiene una fase inicial de carga en Metal y calentamiento del modelo. Configure
    `localService.readyTimeoutMs: 300000` cuando OpenClaw inicie el servidor
    bajo demanda.
  </Accordion>
</AccordionGroup>

## Relacionado

<CardGroup cols={2}>
  <Card title="Servicios de modelos locales" href="/es/gateway/local-model-services" icon="play">
    Inicie servidores de modelos locales bajo demanda antes de las solicitudes al modelo.
  </Card>
  <Card title="Modelos locales" href="/es/gateway/local-models" icon="server">
    Elija y opere backends de modelos locales.
  </Card>
  <Card title="Proveedores de modelos" href="/es/concepts/model-providers" icon="layers">
    Configure referencias de proveedores, autenticación y conmutación por error.
  </Card>
  <Card title="DeepSeek" href="/es/providers/deepseek" icon="brain">
    Comportamiento nativo del proveedor DeepSeek y controles de pensamiento.
  </Card>
</CardGroup>
