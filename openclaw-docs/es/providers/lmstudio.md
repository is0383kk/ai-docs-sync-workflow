---
read_when:
    - Quieres ejecutar OpenClaw con modelos de código abierto mediante LM Studio
    - Se desea instalar y configurar LM Studio
summary: Ejecutar OpenClaw con LM Studio
title: LM Studio
x-i18n:
    generated_at: "2026-07-26T04:51:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f43b4d04aad6e5edfdf224747083834ebd441aa7f91ccbf2d61de990443fc414
    source_path: providers/lmstudio.md
    workflow: 16
---

LM Studio ejecuta modelos llama.cpp (GGUF) o MLX localmente, como aplicación con interfaz gráfica o mediante el daemon sin interfaz `llmster`.
Para consultar la instalación y la documentación del producto, véase [lmstudio.ai](https://lmstudio.ai/).

## Inicio rápido

<Steps>
  <Step title="Instalar e iniciar el servidor">
    Instale LM Studio (escritorio) o `llmster` (sin interfaz) y, a continuación, inicie el servidor:

    ```bash
    lms server start --port 1234
    ```

    También puede ejecutar el daemon sin interfaz:

    ```bash
    lms daemon up
    ```

    Si utiliza la aplicación de escritorio, habilite JIT para que la carga de modelos sea fluida; consulte la
    [guía de JIT y TTL de LM Studio](https://lmstudio.ai/docs/developer/core/ttl-and-auto-evict).

  </Step>
  <Step title="Establecer una clave de API si la autenticación está habilitada">
    ```bash
    export LM_API_TOKEN="your-lm-studio-api-token"
    ```

    Si la autenticación de LM Studio está deshabilitada, deje en blanco la clave de API durante la configuración. Consulte
    [Autenticación de LM Studio](https://lmstudio.ai/docs/developer/core/authentication).

  </Step>
  <Step title="Ejecutar la incorporación">
    ```bash
    openclaw onboard
    ```

    Elija `LM Studio` y, a continuación, seleccione un modelo cuando aparezca la solicitud `Default model`.

    En una nueva configuración guiada, OpenClaw consulta primero `/api/v1/models` en el
    host predeterminado o configurado de LM Studio. Un LLM existente se ofrece automáticamente
    solo cuando LM Studio indica que está entrenado para usar herramientas y que tiene al menos 16K de
    contexto efectivo. Para los modelos cargados, el contexto de la instancia cargada tiene prioridad
    sobre el máximo anunciado, aunque sea mayor. La misma secuencia de configuración de CLI/macOS verifica la
    ruta con una finalización real antes de guardarla. La comprobación automática nunca
    descarga un modelo e ignora las entradas del catálogo que solo admiten embeddings.

  </Step>
</Steps>

Para cambiar posteriormente el modelo predeterminado:

```bash
openclaw models set lmstudio/qwen/qwen3.5-9b
```

Las claves de modelos de LM Studio utilizan el formato `author/model-name` (por ejemplo, `qwen/qwen3.5-9b`); las referencias de modelos de OpenClaw
anteponen el proveedor: `lmstudio/qwen/qwen3.5-9b`. Para encontrar la clave exacta de un modelo, ejecute el
siguiente comando y consulte el campo `key`:

```bash
curl http://localhost:1234/api/v1/models
```

## Incorporación no interactiva

```bash
openclaw onboard --non-interactive --accept-risk --auth-choice lmstudio
```

También puede especificar explícitamente la URL base, el modelo y la clave de API:

```bash
openclaw onboard \
  --non-interactive \
  --accept-risk \
  --auth-choice lmstudio \
  --custom-base-url http://localhost:1234/v1 \
  --lmstudio-api-key "$LM_API_TOKEN" \
  --custom-model-id qwen/qwen3.5-9b
```

`--custom-model-id` recibe la clave del modelo tal como la devuelve LM Studio (por ejemplo, `qwen/qwen3.5-9b`), sin
el prefijo de proveedor `lmstudio/`. Pase `--lmstudio-api-key` (o establezca `LM_API_TOKEN`) para los servidores con
autenticación; omítalo para los servidores sin autenticación y OpenClaw almacenará en su lugar un marcador local no secreto.
`--custom-api-key` sigue siendo compatible, pero se prefiere `--lmstudio-api-key`.

Esto escribe `models.providers.lmstudio` y establece `lmstudio/<custom-model-id>` como modelo predeterminado.
Al proporcionar una clave de API, también se escribe el perfil de autenticación `lmstudio:default`.

La configuración interactiva también puede solicitar una longitud de contexto de carga preferida y aplicarla a todos
los modelos detectados que guarda en la configuración.

## Configuración

### Compatibilidad del uso con la transmisión

LM Studio no siempre emite un objeto `usage` con el formato de OpenAI en las respuestas transmitidas. OpenClaw
recupera en su lugar los recuentos de tokens de los metadatos de estilo llama.cpp `timings.prompt_n` / `timings.predicted_n`.
Cualquier endpoint compatible con OpenAI que se resuelva como endpoint local (host de bucle invertido) obtiene el mismo
mecanismo alternativo, lo que abarca otros backends locales como vLLM, SGLang, llama.cpp, LocalAI, Jan, TabbyAPI
y text-generation-webui.

### Compatibilidad del razonamiento

Cuando la detección `/api/v1/models` de LM Studio informa de opciones de razonamiento específicas del modelo, OpenClaw
expone los valores `reasoning_effort` correspondientes (`none`, `minimal`, `low`, `medium`, `high`, `xhigh`) en
los metadatos de compatibilidad del modelo. Algunas versiones de LM Studio anuncian una opción binaria en la interfaz de usuario (`allowed_options: ["off",
"on"]`), pero rechazan esos valores literales en `/v1/chat/completions`; OpenClaw normaliza esa
estructura binaria a la escala de seis niveles antes de enviar las solicitudes, también para configuraciones guardadas anteriormente que
aún contienen mapas de razonamiento `off`/`on`.

### Configuración explícita

```json5
{
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "${LM_API_TOKEN}",
        api: "openai-completions",
        models: [
          {
            id: "qwen/qwen3-coder-next",
            name: "Qwen 3 Coder Next",
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

### Deshabilitar la precarga

LM Studio admite la carga de modelos justo a tiempo (JIT), que carga los modelos con la primera solicitud. De forma predeterminada, OpenClaw
precarga los modelos mediante el endpoint de carga nativo de LM Studio, lo que resulta útil cuando JIT está
deshabilitado. Para que JIT, el TTL de inactividad y la expulsión automática de LM Studio gestionen el ciclo de vida del modelo,
deshabilite el paso de precarga de OpenClaw:

```json5
{
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        api: "openai-completions",
        params: { preload: false },
        models: [{ id: "qwen/qwen3.5-9b" }],
      },
    },
  },
}
```

### Host de LAN o tailnet

Utilice la dirección accesible del host de LM Studio, mantenga `/v1` y asegúrese de que LM Studio esté vinculado más allá
de la interfaz de bucle invertido en ese equipo:

```json5
{
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://gpu-box.local:1234/v1",
        apiKey: "lmstudio",
        api: "openai-completions",
        models: [{ id: "qwen/qwen3.5-9b" }],
      },
    },
  },
}
```

`lmstudio` confía automáticamente en su endpoint configurado para las solicitudes de modelos, incluidos los hosts de bucle invertido,
LAN y tailnet (excepto los orígenes de metadatos o de vínculo local). Cualquier entrada de proveedor personalizada/local compatible con OpenAI
obtiene la misma confianza en el origen exacto. Las solicitudes a otro host privado o puerto siguen
requiriendo `models.providers.<id>.request.allowPrivateNetwork: true`; establézcalo en `false` para no utilizar
la confianza predeterminada.

## Solución de problemas

### No se detecta LM Studio

Asegúrese de que LM Studio esté en ejecución:

```bash
lms server start --port 1234
```

Si la autenticación está habilitada, establezca también `LM_API_TOKEN`. Compruebe que la API sea accesible:

```bash
curl http://localhost:1234/api/v1/models
```

### Errores de autenticación (HTTP 401)

- Compruebe que `LM_API_TOKEN` coincida con la clave configurada en LM Studio.
- Consulte [Autenticación de LM Studio](https://lmstudio.ai/docs/developer/core/authentication).
- Si el servidor no requiere autenticación, deje la clave en blanco durante la configuración.

## Contenido relacionado

- [Selección de modelos](/es/concepts/model-providers)
- [Ollama](/es/providers/ollama)
- [Modelos locales](/es/gateway/local-models)
