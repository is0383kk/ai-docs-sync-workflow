---
read_when:
    - Se desea ejecutar OpenClaw con un servidor Inferrs local
    - Estás sirviendo Gemma u otro modelo mediante Inferrs
    - Necesitas las opciones de compatibilidad exactas de OpenClaw para Inferrs
summary: Ejecutar OpenClaw mediante Inferrs (servidor local compatible con OpenAI)
title: Inferrs
x-i18n:
    generated_at: "2026-07-26T05:18:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8b9b6fe337a2ec6536332dd62840052fd802fad0a5f3d885ce137523266ff3c9
    source_path: providers/inferrs.md
    workflow: 16
---

[inferrs](https://github.com/ericcurtin/inferrs) sirve modelos locales mediante una API `/v1` compatible con OpenAI. OpenClaw se comunica con él a través del adaptador genérico `openai-completions`.

| Propiedad          | Valor                                                                |
| ------------------ | -------------------------------------------------------------------- |
| Id. del proveedor  | `inferrs` (personalizado; se configura en `models.providers.inferrs`) |
| Plugin             | ninguno — no es un plugin de proveedor incluido con OpenClaw         |
| Variable de entorno de autenticación | no es necesaria; cualquier valor funciona si el servidor de inferrs no tiene autenticación |
| API                | Compatible con OpenAI (`openai-completions`)                            |
| URL base sugerida  | `http://127.0.0.1:8080/v1` (o donde escuche el servidor de inferrs)          |

<Note>
  `inferrs` es un backend personalizado, autoalojado y compatible con OpenAI, no un plugin de proveedor dedicado de OpenClaw: se configura en `models.providers.inferrs` en lugar de elegir una opción de autenticación durante la incorporación. Para usar un plugin incluido con detección automática, consulte [SGLang](/es/providers/sglang) o [vLLM](/es/providers/vllm).
</Note>

## Primeros pasos

<Steps>
  <Step title="Iniciar inferrs con un modelo">
    ```bash
    inferrs serve google/gemma-4-E2B-it \
      --host 127.0.0.1 \
      --port 8080 \
      --device metal
    ```
  </Step>
  <Step title="Verificar que se pueda acceder al servidor">
    ```bash
    curl http://127.0.0.1:8080/health
    curl http://127.0.0.1:8080/v1/models
    ```
  </Step>
  <Step title="Añadir una entrada de proveedor de OpenClaw">
    Añada una entrada de proveedor explícita y dirija el modelo predeterminado a ella. Consulte el ejemplo de configuración siguiente.
  </Step>
</Steps>

## Ejemplo de configuración completa

Gemma 4 en un servidor `inferrs` local:

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

## Inicio bajo demanda

OpenClaw puede iniciar `inferrs` por sí mismo únicamente cuando se selecciona un modelo `inferrs/...`. Añada `localService` a la misma entrada de proveedor:

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

`command` debe ser una ruta absoluta. Ejecute `which inferrs` en el host del Gateway y utilice esa ruta. Referencia completa de los campos: [Servicios de modelos locales](/es/gateway/local-model-services).

## Configuración avanzada

<AccordionGroup>
  <Accordion title="Por qué requiresStringContent es importante">
    Algunas rutas de Chat Completions de `inferrs` solo aceptan `messages[].content` de tipo cadena, no matrices estructuradas de partes de contenido.

    <Warning>
    Si las ejecuciones de OpenClaw fallan con:

    ```text
    messages[1].content: tipo no válido: secuencia; se esperaba una cadena
    ```

    establezca `compat.requiresStringContent: true` en la entrada del modelo. OpenClaw convierte entonces las partes que contienen únicamente texto en cadenas simples antes de enviar la solicitud.
    </Warning>

  </Accordion>

  <Accordion title="Advertencia sobre Gemma y el esquema de herramientas">
    Algunas combinaciones de `inferrs` y Gemma aceptan solicitudes directas pequeñas de `/v1/chat/completions`, pero fallan en turnos completos del entorno de ejecución de agentes de OpenClaw. Pruebe primero a desactivar la superficie del esquema de herramientas:

    ```json5
    compat: {
      requiresStringContent: true,
      supportsTools: false
    }
    ```

    Esto reduce la presión del prompt sobre los backends locales más estrictos. Si las solicitudes directas pequeñas siguen funcionando, pero los turnos normales de agentes de OpenClaw continúan bloqueándose dentro de `inferrs`, considérelo una limitación del modelo o servidor ascendente, no un problema del transporte de OpenClaw.

  </Accordion>

  <Accordion title="Prueba rápida manual">
    Pruebe ambas capas después de configurarlas:

    ```bash
    curl http://127.0.0.1:8080/v1/chat/completions \
      -H 'content-type: application/json' \
      -d '{"model":"google/gemma-4-E2B-it","messages":[{"role":"user","content":"¿Cuánto es 2 + 2?"}],"stream":false}'
    ```

    ```bash
    openclaw infer model run \
      --model inferrs/google/gemma-4-E2B-it \
      --prompt "¿Cuánto es 2 + 2? Responde con una frase breve." \
      --json
    ```

    Si el primer comando funciona, pero el segundo falla, consulte Solución de problemas más adelante.

  </Accordion>

  <Accordion title="Comportamiento de tipo proxy">
    Debido a que `inferrs` utiliza el adaptador genérico `openai-completions` (no `openai-responses`), nunca se aplica la conformación de solicitudes exclusiva de OpenAI nativo: no se envían `service_tier`, `store` de Responses, indicaciones de caché de prompts ni conformación de cargas útiles de compatibilidad de razonamiento de OpenAI.
  </Accordion>
</AccordionGroup>

## Solución de problemas

<AccordionGroup>
  <Accordion title="curl /v1/models falla">
    `inferrs` no se está ejecutando, no es accesible o no está vinculado al host o puerto configurado. Confirme que el servidor esté iniciado y escuchando en esa dirección.
  </Accordion>

  <Accordion title="messages[].content esperaba una cadena">
    Establezca `compat.requiresStringContent: true` en la entrada del modelo (consulte la sección anterior).
  </Accordion>

  <Accordion title="Las llamadas directas a /v1/chat/completions funcionan, pero openclaw infer model run falla">
    Establezca `compat.supportsTools: false` para desactivar la superficie del esquema de herramientas (consulte la advertencia sobre Gemma anterior).
  </Accordion>

  <Accordion title="inferrs sigue bloqueándose en turnos de agente más grandes">
    Si los errores de esquema han desaparecido, pero `inferrs` sigue bloqueándose en turnos de agente más grandes, considérelo una limitación de `inferrs` ascendente o del modelo. Reduzca la presión del prompt o cambie de backend o modelo.
  </Accordion>
</AccordionGroup>

<Tip>
Para obtener ayuda general, consulte [Solución de problemas](/es/help/troubleshooting) y [Preguntas frecuentes](/es/help/faq).
</Tip>

## Relacionado

<CardGroup cols={2}>
  <Card title="Modelos locales" href="/es/gateway/local-models" icon="server">
    Ejecución de OpenClaw con servidores de modelos locales.
  </Card>
  <Card title="Servicios de modelos locales" href="/es/gateway/local-model-services" icon="play">
    Inicio bajo demanda de servidores de modelos locales para los proveedores configurados.
  </Card>
  <Card title="Solución de problemas del Gateway" href="/es/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail" icon="wrench">
    Depuración de backends locales compatibles con OpenAI que superan las pruebas, pero fallan en las ejecuciones de agentes.
  </Card>
  <Card title="Selección de modelos" href="/es/concepts/model-providers" icon="layers">
    Descripción general de todos los proveedores, las referencias de modelos y el comportamiento de conmutación por error.
  </Card>
</CardGroup>
