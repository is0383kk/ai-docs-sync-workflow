---
read_when:
    - Diseño o refactorización de la comprensión de contenido multimedia
    - Ajuste del preprocesamiento de audio, vídeo e imágenes entrantes
sidebarTitle: Media understanding
summary: Comprensión de imágenes, audio y vídeo entrantes (opcional) con alternativas de proveedor y CLI
title: Comprensión de contenido multimedia
x-i18n:
    generated_at: "2026-07-26T05:44:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 38e9a0f89607bb9c4af85689ef0fbd3df9234b36e06d86c129e0d823d6e05143
    source_path: nodes/media-understanding.md
    workflow: 16
---

OpenClaw puede resumir los archivos multimedia entrantes (imagen/audio/video) antes de que se ejecute el pipeline de respuesta, de modo que el análisis de comandos y el enrutamiento trabajen con texto breve en lugar de bytes sin procesar. La comprensión detecta automáticamente las herramientas locales o las claves de proveedores, o se pueden configurar modelos explícitos. Los archivos multimedia originales siempre se entregan al modelo como de costumbre; cuando la comprensión falla o está deshabilitada, el flujo de respuesta continúa sin cambios.

Los plugins de proveedores registran metadatos de capacidades (qué proveedor admite cada tipo de archivo multimedia, modelo predeterminado y prioridad). El núcleo de OpenClaw controla la configuración compartida `tools.media`, el orden de respaldo y la integración con el pipeline de respuesta.

## Cómo funciona

<Steps>
  <Step title="Recopilar archivos adjuntos">
    Recopila en orden los datos de los archivos multimedia entrantes (`path`, `url`, `contentType` y `kind`).
  </Step>
  <Step title="Seleccionar por capacidad">
    Para cada capacidad habilitada (imagen/audio/video), selecciona los archivos adjuntos según la política `attachments` (valor predeterminado: solo el primer archivo adjunto).
  </Step>
  <Step title="Elegir un modelo">
    Elige la primera entrada de modelo apta (tamaño + capacidad + autenticación disponible).
  </Step>
  <Step title="Usar el respaldo en caso de error">
    Si un modelo genera un error, agota el tiempo de espera o el archivo multimedia supera `maxBytes`, prueba la siguiente entrada.
  </Step>
  <Step title="Aplicar cuando se completa correctamente">
    `Body` se convierte en un bloque `[Image]`, `[Audio]` o `[Video]`. El audio también establece `{{Transcript}}`; el análisis de comandos utiliza el texto del pie de contenido cuando está presente y, en caso contrario, la transcripción. Los pies de contenido se conservan como `User text:` dentro del bloque.
  </Step>
</Steps>

## Configuración

`tools.media` contiene una lista de modelos etiquetados por capacidad y pequeños controles para cada capacidad:

```json5
{
  tools: {
    media: {
      concurrency: 2, // máximo de ejecuciones de capacidades simultáneas (valor predeterminado)
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["image", "video"] },
      ],
      image: { preferredModel: "google/gemini-3-flash-preview" },
      audio: { enabled: true },
      video: { enabled: true },
    },
  },
}
```

Claves por capacidad (`image`/`audio`/`video`):

| Clave              | Tipo      | Valor predeterminado                                | Notas                                                                |
| ---------------- | --------- | -------------------------------------- | -------------------------------------------------------------------- |
| `enabled`        | `boolean` | automático (`false` lo deshabilita)                | Establece `false` para desactivar la detección automática de esta capacidad              |
| `preferredModel` | `string`  | primera entrada compatible                 | Da preferencia a `provider/model`, el id. de modelo, `provider:<id>` o `cli:command` |
| `prompt`         | `string`  | valor predeterminado de la capacidad                     | Prompt predeterminado cuando una entrada no lo reemplaza                    |
| `maxChars`       | `number`  | `500` para imagen/video, sin establecer para audio         | Límite de salida predeterminado                                                 |
| `maxBytes`       | `number`  | 10MB para imagen, 20MB para audio, 50MB para video     | Límite de entrada predeterminado                                                  |
| `timeoutSeconds` | `number`  | `60` para imagen/audio, `120` para video          | Tiempo de espera predeterminado de la solicitud                                              |
| `language`       | `string`  | sin establecer                                  | Indicación para la transcripción de audio                                             |
| `scope`          | objeto    | sin establecer                                  | Restringe por canal, tipo de chat o clave de origen                                 |
| `attachments`    | objeto    | `{ mode: "first", maxAttachments: 1 }` | Selecciona qué archivos adjuntos coincidentes se procesan                      |
| `echoTranscript` | `boolean` | `false`                                | Solo audio: muestra la transcripción antes del procesamiento del agente              |
| `echoFormat`     | `string`  | `'📝 "{transcript}"'`                  | Solo audio: formato de la transcripción mostrada                         |

Los prompts, límites, indicaciones de idioma, reemplazos de solicitudes y opciones de proveedores se pueden establecer como valores predeterminados de la capacidad o reemplazar en entradas `tools.media.models[]` individuales. Los valores predeterminados de la capacidad también se aplican a los proveedores detectados automáticamente cuando no se configura ningún modelo explícito.

### Entradas de modelos

Cada entrada `models[]` es una entrada de **proveedor** (valor predeterminado) o una entrada de **CLI**:

<Tabs>
  <Tab title="Entrada de proveedor">
    ```json5
    {
      type: "provider", // valor predeterminado si se omite
      provider: "openai",
      model: "gpt-5.6-sol",
      prompt: "Describe la imagen en <= 500 caracteres.",
      maxChars: 500,
      maxBytes: 10485760,
      timeoutSeconds: 60,
      capabilities: ["image"],
      profile: "vision-profile",
      preferredProfile: "vision-fallback",
    }
    ```
  </Tab>
  <Tab title="Entrada de CLI">
    ```json5
    {
      type: "cli",
      command: "gemini",
      args: [
        "-m",
        "gemini-3-flash",
        "--allowed-tools",
        "read_file",
        "Lee el archivo multimedia en {{AttachmentPath}} y descríbelo en <= {{MaxChars}} caracteres.",
      ],
      maxChars: 500,
      maxBytes: 52428800,
      timeoutSeconds: 120,
      capabilities: ["video", "image"],
    }
    ```

    Las plantillas de CLI también pueden utilizar `{{AttachmentUrl}}`, `{{AttachmentContentType}}`, `{{AttachmentDir}}`, `{{AttachmentIndex}}`, `{{OutputDir}}` (directorio temporal creado para esta ejecución) y `{{OutputBase}}` (ruta base del archivo temporal, sin extensión). Los nombres anteriores `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` y `{{MediaDir}}` siguen siendo alias de compatibilidad obsoletos.

  </Tab>
</Tabs>

### Credenciales del proveedor

La comprensión de archivos multimedia mediante proveedores utiliza la misma resolución de autenticación que las llamadas normales a modelos: perfiles de autenticación, variables de entorno y, a continuación, `models.providers.<providerId>.apiKey`. Las entradas `tools.media.models[]` no aceptan un campo `apiKey` en línea.

```json5
{
  models: {
    providers: {
      openai: { apiKey: "<OPENAI_API_KEY>" },
      moonshot: { apiKey: "<MOONSHOT_API_KEY>" },
    },
  },
}
```

Consulta [Herramientas y proveedores personalizados](/es/gateway/config-tools) para obtener información sobre los perfiles, las variables de entorno y las URL base personalizadas.

## Reglas y comportamiento

- Los archivos multimedia que superan `maxBytes` omiten ese modelo y prueban el siguiente.
- Los archivos de audio de menos de 1024 bytes se consideran vacíos o dañados y se omiten antes de la transcripción; en su lugar, el agente recibe una transcripción de marcador de posición determinista.
- Si el modelo de imagen principal activo ya admite visión de forma nativa, OpenClaw omite el bloque de resumen `[Image]` y pasa la imagen original directamente al modelo. MiniMax es una excepción: `minimax`, `minimax-cn`, `minimax-portal` y `minimax-portal-cn` siempre enrutan la comprensión de imágenes mediante el proveedor de archivos multimedia `MiniMax-VL-01` controlado por el plugin, aunque los metadatos heredados de chat de MiniMax M2.x indiquen que admite entrada de imágenes (solo `MiniMax-M3` y versiones posteriores se consideran compatibles con visión de forma nativa).
- Si el modelo principal de Gateway/WebChat solo admite texto, los archivos de imagen adjuntos se conservan como referencias `media://inbound/*` descargadas, de modo que las herramientas de imagen/PDF o un modelo de imagen configurado puedan inspeccionarlos sin perder el archivo adjunto.
- La configuración explícita de `openclaw infer image describe --file <path> --model <provider/model>` (alias: `openclaw capability image describe`) ejecuta directamente ese proveedor/modelo compatible con imágenes, incluidas referencias de Ollama como `ollama/qwen2.5vl:7b` cuando se configura un modelo compatible con imágenes coincidente en `models.providers.ollama.models[]`.
- Si `<capability>.enabled` no es `false`, pero no hay modelos configurados, OpenClaw prueba el modelo de respuesta activo cuando su proveedor admite la capacidad.

### Detección automática (valor predeterminado)

Cuando `tools.media.<capability>.enabled` no es `false` y no hay modelos configurados, OpenClaw prueba las siguientes opciones en orden y se detiene en la primera que funciona:

<Steps>
  <Step title="Modelo de imagen configurado (solo imagen)">
    Referencias principales/de respaldo de `agents.defaults.imageModel`, salvo que el modelo de respuesta activo ya admita visión de forma nativa. Se da preferencia a las referencias `provider/model`; las referencias simples solo se completan a partir de entradas de modelos de proveedores configurados compatibles con imágenes cuando la coincidencia es única.
  </Step>
  <Step title="Modelo de respuesta activo">
    El modelo de respuesta activo, cuando su proveedor admite la capacidad.
  </Step>
  <Step title="Autenticación del proveedor (solo audio, antes de las CLI locales)">
    Las entradas `models.providers.*` configuradas que admiten audio se prueban antes que las CLI locales. Orden de prioridad de proveedores incluidos (los empates se resuelven alfabéticamente por id. de proveedor): Groq/OpenAI &rarr; xAI &rarr; Deepgram &rarr; OpenRouter &rarr; Google/SenseAudio &rarr; Deepinfra/ElevenLabs &rarr; Mistral.
  </Step>
  <Step title="CLI locales (solo audio)">
    Los binarios locales disponibles forman una lista ordenada de respaldo:
    - `whisper-cli` primero solo después de que una invocación anterior de un modelo en el proceso actual haya detectado Metal o CUDA
    - `sherpa-onnx-offline` con CPU de forma predeterminada (requiere `SHERPA_ONNX_MODEL_DIR` con `tokens.txt`/`encoder.onnx`/`decoder.onnx`/`joiner.onnx`)
    - `whisper-cli` cuando la aceleración solo es compatible en la compilación o no se ha observado
    - `parakeet-mlx` en Apple Silicon (compatible con MLX, uso del dispositivo no observado)
    - `whisper` (CLI de Python; usa de forma predeterminada el modelo `turbo` y lo descarga automáticamente)

    La inspección de capacidades del backend se almacena en caché y no carga ningún modelo. La capacidad de compilación, las marcas de backend solicitadas y el backend observado en una invocación real se mantienen por separado. whisper.cpp detectado automáticamente mantiene habilitados los registros de ejecución del modelo para poder registrar la línea del backend seleccionado por el componente de origen. Las entradas de CLI explícitas conservan el orden, las marcas de backend y las marcas de salida configurados.

  </Step>
  <Step title="Autenticación del proveedor (imagen/video)">
    Las entradas `models.providers.*` configuradas que admiten la capacidad se prueban antes que el orden de respaldo incluido. Los proveedores configurados solo para imágenes con un modelo compatible con imágenes se registran automáticamente para la comprensión de archivos multimedia, aunque no sean un plugin de proveedor incluido.

    Orden de prioridad de proveedores incluidos (los empates se resuelven alfabéticamente por id. de proveedor):
    - Imagen: Anthropic/OpenAI &rarr; Google &rarr; MiniMax &rarr; Deepinfra &rarr; MiniMax Portal &rarr; Z.AI
    - Video: Google &rarr; Qwen &rarr; Moonshot

  </Step>
  <Step title="CLI de Antigravity (solo imagen/video)">
    El primer binario `agy` o `antigravity` instalado (se reemplaza con `OPENCLAW_ANTIGRAVITY_CLI`), aislado en el directorio del archivo multimedia.
  </Step>
</Steps>

Para deshabilitar la detección automática de una capacidad:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: false,
      },
    },
  },
}
```

<Note>
La detección de binarios se realiza con el máximo esfuerzo en macOS/Linux/Windows; asegúrate de que la CLI esté en `PATH` (`~` se expande) o establece una entrada explícita de modelo de CLI con la ruta completa del comando.
</Note>

### Compatibilidad con proxy (llamadas de proveedores de audio/video)

La comprensión de **audio** y **video** mediante proveedores respeta las variables de entorno estándar del proxy saliente, incluidas las reglas de omisión `NO_PROXY`/`no_proxy`: `HTTPS_PROXY`, `HTTP_PROXY`, `ALL_PROXY`, `https_proxy`, `http_proxy`, `all_proxy`. Las variables en minúsculas tienen prioridad sobre las que están en mayúsculas. Si no se establece ninguna, la comprensión de archivos multimedia utiliza una salida directa; si el valor del proxy tiene un formato incorrecto, OpenClaw registra una advertencia y vuelve a la recuperación directa. La comprensión de imágenes no utiliza esta ruta de proxy.

## Capacidades

Establece `capabilities` en una entrada `models[]` para restringirla a tipos específicos de archivos multimedia. En las listas compartidas, OpenClaw infiere los valores predeterminados para cada proveedor incluido:

| Proveedor                                                                | Capacidades           |
| ------------------------------------------------------------------------ | --------------------- |
| `openai`, `anthropic`, `minimax`                                         | imagen                |
| `minimax-portal`                                                         | imagen                |
| `moonshot`                                                               | imagen + vídeo        |
| `openrouter`                                                             | imagen + audio        |
| `google` (API de Gemini)                                                    | imagen + audio + vídeo |
| `qwen`                                                                   | imagen + vídeo        |
| `deepinfra`                                                              | imagen + audio        |
| `mistral`                                                                | audio                 |
| `zai`                                                                    | imagen                |
| `groq`, `xai`, `deepgram`, `senseaudio`                                  | audio                 |
| Cualquier catálogo de `models.providers.<id>.models[]` con un modelo compatible con imágenes | imagen                |

Para las entradas de la CLI, establezca `capabilities` explícitamente para evitar coincidencias inesperadas; si se omite, la entrada será apta para todas las listas de capacidades en las que aparezca.

## Matriz de compatibilidad de proveedores

| Capacidad | Proveedores                                                                                                                                               | Notas                                                                                                                                                                                   |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Imagen     | Anthropic, servidor de aplicaciones de Codex, Deepinfra, Google, MiniMax, MiniMax Portal, Moonshot, OpenAI, OAuth de OpenAI Codex, OpenRouter, Qwen, Z.AI, proveedores de configuración | Los plugins de los proveedores registran la compatibilidad con imágenes; `openai/*` puede usar enrutamiento mediante clave de API u OAuth de Codex; `codex/*` usa un turno acotado del servidor de aplicaciones de Codex; los proveedores de configuración compatibles con imágenes se registran automáticamente. |
| Audio      | Deepgram, Deepinfra, ElevenLabs, Google, Groq, Mistral, OpenAI, OpenRouter, SenseAudio, xAI                                                             | Transcripción del proveedor (Whisper/Groq/xAI/Deepgram/STT de OpenRouter/Gemini/SenseAudio/Scribe/Voxtral).                                                                                     |
| Vídeo      | Google, Moonshot, Qwen                                                                                                                                  | Comprensión de vídeo del proveedor mediante plugins del proveedor; la comprensión de vídeo de Qwen utiliza los endpoints estándar de DashScope.                                                                        |

<Note>
**Nota sobre MiniMax**: la comprensión de imágenes de `minimax`, `minimax-cn`, `minimax-portal` y `minimax-portal-cn` siempre procede del proveedor de medios `MiniMax-VL-01`, propiedad del plugin, aunque los metadatos heredados del chat de MiniMax M2.x indiquen que admite la entrada de imágenes.
</Note>

## Orientación para seleccionar modelos

- Cuando la calidad y la seguridad sean importantes, utilice el modelo de la generación actual más potente para cada capacidad multimedia.
- En agentes con herramientas que gestionen entradas no confiables, evite los modelos multimedia antiguos o menos potentes.
- Mantenga al menos una alternativa por capacidad para garantizar la disponibilidad (un modelo de calidad y otro más rápido o económico).
- Las alternativas de la CLI (`whisper-cli`, `whisper`, `gemini`) resultan útiles cuando las API de los proveedores no están disponibles.
- Los modos conocidos de salida a archivo son autoritativos: si el archivo de transcripción inferido está vacío o no existe, no se genera ninguna transcripción en lugar de recurrir a la salida de progreso de la CLI.
- `parakeet-mlx`: utilice `--output-format txt` (o `all`) con `--output-dir` y la plantilla de salida predeterminada `{filename}`. También se respetan las variables de entorno del proyecto de origen `PARAKEET_OUTPUT_FORMAT` y `PARAKEET_OUTPUT_TEMPLATE`. OpenClaw lee `<output-dir>/<media-basename>.txt`; el formato predeterminado `srt`, los demás formatos y las plantillas de salida personalizadas siguen utilizando stdout.

## Política de archivos adjuntos

La opción `attachments` de cada capacidad controla qué archivos adjuntos se procesan:

<ParamField path="mode" type='"first" | "all"' default="first">
  Procesa solo el primer archivo adjunto seleccionado o todos ellos.
</ParamField>
<ParamField path="maxAttachments" type="number" default="1">
  Limita el número de archivos procesados.
</ParamField>
<ParamField path="prefer" type='"first" | "last" | "path" | "url"'>
  Preferencia de selección entre los archivos adjuntos candidatos.
</ParamField>

Cuando `mode: "all"`, las salidas se etiquetan como `[Image 1/2]`, `[Audio 2/2]`, etc.

### Extracción de archivos adjuntos

- El texto extraído de los archivos se delimita como contenido externo no confiable antes de añadirse al prompt multimedia, mediante marcadores de límite como `<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>` y una línea de metadatos `Source: External`.
- Esta ruta omite intencionadamente el extenso aviso `SECURITY NOTICE:` para mantener breve el prompt multimedia; los marcadores de límite y los metadatos siguen aplicándose.
- Los archivos sin texto extraíble reciben `[No extractable text]`.
- Si un PDF recurre a imágenes renderizadas de sus páginas, OpenClaw reenvía esas imágenes a los modelos de respuesta con capacidad de visión y conserva el marcador de posición `[PDF content rendered to images]` en el bloque del archivo.

## Ejemplos de configuración

<Tabs>
  <Tab title="Modelos compartidos y anulaciones">
    ```json5
    {
      tools: {
        media: {
          models: [
            { provider: "openai", model: "gpt-5.6-sol", capabilities: ["image"] },
            {
              provider: "google",
              model: "gemini-3-flash-preview",
              capabilities: ["image", "audio", "video"],
            },
            {
              type: "cli",
              command: "gemini",
              args: [
                "-m",
                "gemini-3-flash",
                "--allowed-tools",
                "read_file",
                "Lee el contenido multimedia de {{AttachmentPath}} y descríbelo en <= {{MaxChars}} caracteres.",
              ],
              capabilities: ["image", "video"],
            },
          ],
          audio: {
            attachments: { mode: "all", maxAttachments: 2 },
          },
          video: {
            maxChars: 500,
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Solo audio y vídeo">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [
              { provider: "openai", model: "gpt-4o-mini-transcribe" },
              {
                type: "cli",
                command: "whisper",
                args: ["--model", "base", "{{AttachmentPath}}"],
              },
            ],
          },
          video: {
            enabled: true,
            maxChars: 500,
            models: [
              { provider: "google", model: "gemini-3-flash-preview" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "Lee el contenido multimedia de {{AttachmentPath}} y descríbelo en <= {{MaxChars}} caracteres.",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Solo imagen">
    ```json5
    {
      tools: {
        media: {
          image: {
            enabled: true,
            maxBytes: 10485760,
            maxChars: 500,
            models: [
              { provider: "openai", model: "gpt-5.6-sol" },
              { provider: "anthropic", model: "claude-opus-5" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "Lee el contenido multimedia de {{AttachmentPath}} y descríbelo en <= {{MaxChars}} caracteres.",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Entrada multimodal única">
    ```json5
    {
      tools: {
        media: {
          image: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          audio: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          video: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## Salida de estado

Cuando se ejecuta la comprensión multimedia, `/status` incluye una línea de resumen por capacidad:

```
📎 Medios: imagen correcta (openai/gpt-5.6-sol) · audio correcto (whisper-cli observado=metal)
```

Para realizar el inventario previo, ejecute `openclaw capability audio providers`. Las filas locales muestran por separado la alternativa local elegida, la selección global del proveedor, la disponibilidad y los campos independientes de backend capaz/solicitado/observado. La misma selección local está disponible como hallazgo informativo de doctor:

```bash
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

## Notas

- La comprensión funciona según el mejor esfuerzo posible. Los errores no bloquean las respuestas.
- Los archivos adjuntos se siguen enviando a los modelos aunque la comprensión esté desactivada.
- Utilice `scope` para limitar dónde se ejecuta la comprensión (por ejemplo, solo en mensajes directos).

## Contenido relacionado

- [Configuración](/es/gateway/configuration)
- [Compatibilidad con imágenes y contenido multimedia](/es/nodes/images)
