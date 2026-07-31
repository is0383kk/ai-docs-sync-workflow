---
read_when:
    - Generación de música o audio mediante el agente
    - Configuración de proveedores y modelos de generación de música
    - Descripción de los parámetros de la herramienta music_generate
sidebarTitle: Music generation
summary: Genera música mediante music_generate en flujos de trabajo de ComfyUI, fal, Google Lyria, MiniMax y OpenRouter
title: Generación de música
x-i18n:
    generated_at: "2026-07-26T05:01:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f2a8a4a36e47839c7896046a556f7bf84f6c168492e2de46736635fe2a9358e
    source_path: tools/music-generation.md
    workflow: 16
---

La herramienta `music_generate` crea música o audio mediante la capacidad
compartida de generación de música, respaldada por ComfyUI, fal, Google, MiniMax y
OpenRouter.

<Note>
`music_generate` solo aparece cuando hay al menos un proveedor de generación de música
disponible: una configuración explícita de `agents.defaults.mediaModels.music` o un
proveedor con autenticación configurada (por ejemplo, una clave de API establecida).
</Note>

Para las ejecuciones de agentes respaldadas por sesiones, `music_generate` se inicia como una tarea en segundo plano,
registra el progreso en el registro de tareas y, cuando la pista está
lista, reactiva al agente para que pueda informar al usuario y adjuntar el audio terminado. El agente
de finalización sigue el contrato de respuesta visible de la sesión: respuesta final automática
cuando está configurada, o `message(action="send")` cuando la sesión requiere la
herramienta de mensajes. Si la sesión solicitante está inactiva o su reactivación falla y
el audio generado todavía no aparece en la respuesta, OpenClaw envía un
recurso alternativo directo e idempotente que contiene únicamente el audio faltante.

## Inicio rápido

<Tabs>
  <Tab title="Respaldado por un proveedor compartido">
    <Steps>
      <Step title="Configurar la autenticación">
        Establezca una clave de API para al menos un proveedor; por ejemplo,
        `GEMINI_API_KEY` o `MINIMAX_API_KEY`.
      </Step>
      <Step title="Elegir un modelo predeterminado (opcional)">
        ```json5
        {
          agents: {
            defaults: {
              musicGenerationModel: {
                primary: "google/lyria-3-clip-preview",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="Solicitarlo al agente">
        _"Genera una pista synthpop animada sobre un viaje nocturno en automóvil por una
        ciudad de neón."_

        El agente llama automáticamente a `music_generate`. No es necesario
        incluir la herramienta en una lista de permitidas.
      </Step>
    </Steps>

    Sin una ejecución de agente respaldada por una sesión (en contextos directos/locales), la herramienta
    se ejecuta en línea y devuelve la ruta final del contenido multimedia en el mismo resultado de la herramienta.

  </Tab>
  <Tab title="Flujo de trabajo de ComfyUI">
    <Steps>
      <Step title="Configurar el flujo de trabajo">
        Configure `plugins.entries.comfy.config.music` con un archivo JSON de flujo de trabajo
        y nodos de solicitud/salida.
      </Step>
      <Step title="Autenticación en la nube (opcional)">
        Para Comfy Cloud, establezca `COMFY_API_KEY` o `COMFY_CLOUD_API_KEY`.
      </Step>
      <Step title="Llamar a la herramienta">
        ```text
        /tool music_generate prompt="Bucle cálido de sintetizador ambiental con una suave textura de cinta"
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

Ejemplos de solicitudes:

```text
Genera una pista cinematográfica de piano con cuerdas suaves y sin voces.
```

```text
Genera un bucle chiptune enérgico sobre el lanzamiento de un cohete al amanecer.
```

Use `action: "list"` para consultar los proveedores/modelos disponibles y
`action: "status"` para consultar la tarea de música activa respaldada por la sesión:

```text
/tool music_generate action=list
/tool music_generate action=status
```

Ejemplo de generación directa:

```text
/tool music_generate prompt="Hip hop lo-fi de ensueño con textura de vinilo y lluvia suave" instrumental=true
```

## Proveedores compatibles

| Proveedor  | Modelo predeterminado          | Entradas de referencia | Controles compatibles                                  | Autenticación                          |
| ---------- | ------------------------------ | ---------------------- | ------------------------------------------------------ | -------------------------------------- |
| ComfyUI    | `workflow`                   | Hasta 1 imagen         | Música o audio definidos por el flujo de trabajo       | `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY` |
| fal        | `fal-ai/minimax-music/v2.6`  | Ninguna                | `lyrics`, `instrumental`, `durationSeconds`, `format` | `FAL_KEY` o `FAL_API_KEY`             |
| Google     | `lyria-3-clip-preview`       | Hasta 10 imágenes      | `lyrics`, `instrumental`, `format`                    | `GEMINI_API_KEY`, `GOOGLE_API_KEY`     |
| MiniMax    | `music-2.6`                  | Ninguna                | `lyrics`, `instrumental`, `format` (solo mp3)         | `MINIMAX_API_KEY` o OAuth de MiniMax  |
| OpenRouter | `google/lyria-3-pro-preview` | Hasta 1 imagen         | `lyrics`, `instrumental`, `durationSeconds`, `format` | `OPENROUTER_API_KEY`                   |

MiniMax registra dos identificadores de proveedor que comparten los mismos modelos: `minimax` para
la autenticación mediante clave de API y `minimax-portal` para OAuth. Las referencias de los modelos siguen la ruta de autenticación
(`minimax/music-2.6` frente a `minimax-portal/music-2.6`); consulte
[MiniMax](/es/providers/minimax#music-generation).

fal también ofrece `fal-ai/ace-step/prompt-to-audio` (wav, sin letras ni
selector instrumental) y `fal-ai/stable-audio-25/text-to-audio` (wav,
solo solicitud), además de su modelo predeterminado respaldado por MiniMax. El modelo predeterminado de Google,
`lyria-3-clip-preview`, solo genera mp3; `lyria-3-pro-preview` también admite
wav. MiniMax también ofrece `music-2.6-free`, `music-cover` y
`music-cover-free`. OpenRouter también ofrece `google/lyria-3-clip-preview`.

### Matriz de capacidades

El contrato de modo explícito utilizado por `music_generate`, las pruebas de contrato y el
barrido en vivo compartido:

| Proveedor  | `generate` | `edit` | Límite de edición | Carriles en vivo compartidos                                              |
| ---------- | :--------: | :----: | ----------------- | ------------------------------------------------------------------------- |
| ComfyUI    |     ✓      |   ✓    | 1 imagen          | No se incluye en el barrido compartido; lo cubre `extensions/comfy/comfy.live.test.ts` |
| fal        |     ✓      |   —    | Ninguno           | `generate`                                                                |
| Google     |     ✓      |   ✓    | 10 imágenes       | `generate`, `edit`                                                        |
| MiniMax    |     ✓      |   —    | Ninguno           | `generate`                                                                |
| OpenRouter |     ✓      |   ✓    | 1 imagen          | `generate`, `edit`                                                        |

## Parámetros de la herramienta

<ParamField path="prompt" type="string" required>
  Solicitud de generación de música. Obligatoria para `action: "generate"`.
</ParamField>
<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  `"status"` devuelve la tarea de la sesión actual; `"list"` consulta los proveedores.
</ParamField>
<ParamField path="model" type="string">
  Sustitución del proveedor/modelo (p. ej., `google/lyria-3-pro-preview`,
  `comfy/workflow`).
</ParamField>
<ParamField path="lyrics" type="string">
  Letras opcionales cuando el proveedor admite una entrada explícita de letras.
</ParamField>
<ParamField path="instrumental" type="boolean">
  Solicita una salida exclusivamente instrumental cuando el proveedor la admite.
</ParamField>
<ParamField path="image" type="string">
  Ruta o URL de una única imagen de referencia.
</ParamField>
<ParamField path="images" type="string[]">
  Varias imágenes de referencia (hasta 10 en los proveedores compatibles).
</ParamField>
<ParamField path="durationSeconds" type="number">
  Duración objetivo en segundos cuando el proveedor admite indicaciones de duración.
</ParamField>
<ParamField path="format" type='"mp3" | "wav"'>
  Indicación del formato de salida cuando el proveedor lo admite.
</ParamField>
<ParamField path="filename" type="string">Indicación del nombre del archivo de salida.</ParamField>

<Note>
No todos los proveedores admiten todos los parámetros. OpenClaw sigue validando los
límites estrictos, como el número de entradas, antes del envío. Cuando un proveedor admite
la duración, pero utiliza un máximo inferior al valor solicitado, OpenClaw
lo limita a la duración compatible más cercana. Las indicaciones opcionales que realmente no se admiten
se ignoran con una advertencia cuando el proveedor o modelo seleccionado no puede
respetarlas. Los resultados de la herramienta indican la configuración aplicada; `details.normalization`
registra cualquier correspondencia entre el valor solicitado y el aplicado.
</Note>

Los tiempos de espera de las solicitudes a proveedores son exclusivamente una configuración del operador. OpenClaw utiliza
`agents.defaults.mediaModels.music.timeoutMs` cuando está configurado, eleva
los valores inferiores a 120000ms hasta 120000ms y, en los demás casos, establece de forma predeterminada las solicitudes a proveedores
en 300000ms.

## Comportamiento asíncrono

La generación de música respaldada por una sesión se ejecuta como una tarea en segundo plano:

- **Tarea en segundo plano:** `music_generate` crea una tarea en segundo plano, devuelve
  inmediatamente una respuesta de inicio/tarea y publica posteriormente la pista terminada en
  un mensaje de seguimiento del agente.
- **Prevención de duplicados:** mientras una tarea está en `queued` o `running`, las llamadas posteriores a
  `music_generate` en la misma sesión devuelven el estado de la tarea en lugar de
  iniciar otra generación. Use `action: "status"` para comprobarlo explícitamente.
  Una solicitud coincidente completada recientemente también se desduplica durante 2 minutos.
- **Consulta de estado:** `openclaw tasks list` o `openclaw tasks show <taskId>`
  consulta los estados en cola, en ejecución y terminales.
- **Reactivación al finalizar:** OpenClaw vuelve a inyectar un evento interno de finalización
  en la misma sesión para que el propio modelo pueda redactar el seguimiento dirigido al usuario.
- **Indicación de la solicitud:** los turnos posteriores del usuario o manuales de la misma sesión reciben una pequeña
  indicación en tiempo de ejecución cuando ya hay una tarea de música en curso, para que el modelo
  no vuelva a llamar a `music_generate` a ciegas.
- **Alternativa sin sesión:** los contextos directos/locales sin una sesión real del agente
  se ejecutan en línea y devuelven el resultado final del audio en el mismo turno.

### Ciclo de vida de las tareas

La tarea de música presenta los mismos estados que el registro general de tareas (consulte
[Tareas en segundo plano](/es/automation/tasks#task-lifecycle) para ver la máquina de estados
completa, incluidos `timed_out`, `cancelled` y `lost`). La mayoría de las ejecuciones de música
pasan por:

| Estado      | Significado                                                                                    |
| ----------- | ---------------------------------------------------------------------------------------------- |
| `queued`    | Tarea creada, a la espera de que el proveedor la acepte.                                       |
| `running`   | El proveedor está procesando (normalmente entre 30 segundos y 3 minutos, según el proveedor y la duración). |
| `succeeded` | La pista está lista; el agente se reactiva y la publica en la conversación.                    |
| `failed`    | Error o tiempo de espera agotado del proveedor; el agente se reactiva con los detalles del error. |

Compruebe el estado desde la CLI:

```bash
openclaw tasks list
openclaw tasks show <taskId>
openclaw tasks cancel <taskId>
```

## Configuración

### Selección del modelo

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
        fallbacks: ["fal/fal-ai/minimax-music/v2.6", "minimax/music-2.6"],
      },
    },
  },
}
```

### Orden de selección de proveedores

OpenClaw prueba los proveedores en este orden:

1. Parámetro `model` de la llamada a la herramienta (si el agente especifica uno).
2. `musicGenerationModel.primary` de la configuración.
3. `musicGenerationModel.fallbacks` en orden.
4. Detección automática utilizando únicamente los valores predeterminados de proveedores respaldados por autenticación:
   - primero, el proveedor predeterminado actual del modelo de texto, si también ofrece
     generación de música;
   - los demás proveedores de generación de música registrados, ordenados alfabéticamente por
     identificador del proveedor.

Si un proveedor falla, se prueba automáticamente el siguiente candidato. Si todos
fallan, el error incluye detalles de cada intento.

La alternativa automática entre proveedores autenticados siempre está habilitada. Un valor
`model` por llamada sigue siendo determinante.

## Notas sobre los proveedores

<AccordionGroup>
  <Accordion title="ComfyUI">
    Se basa en flujos de trabajo y depende del grafo configurado, además de la asignación de nodos
    para los campos de entrada y salida. El Plugin `comfy` incluido se integra en la
    herramienta compartida `music_generate` mediante el registro de proveedores
    de generación de música.
  </Accordion>
  <Accordion title="fal">
    Usa los endpoints de modelos de fal mediante la ruta compartida de autenticación de proveedores. El
    proveedor incluido utiliza `fal-ai/minimax-music/v2.6` de forma predeterminada y también ofrece
    `fal-ai/ace-step/prompt-to-audio` y
    `fal-ai/stable-audio-25/text-to-audio` para solicitudes de generación de audio a partir de indicaciones.
    Las letras y el modo instrumental son exclusivos del modelo MiniMax; los otros dos
    modelos solo admiten indicaciones.
  </Accordion>
  <Accordion title="Google (Lyria 3)">
    Usa la generación por lotes de Lyria 3. El flujo incluido actual admite
    indicaciones, texto de letras opcional e imágenes de referencia opcionales. El
    modelo predeterminado `lyria-3-clip-preview` solo genera mp3; el
    modelo `lyria-3-pro-preview` también admite wav.
  </Accordion>
  <Accordion title="MiniMax">
    Usa el endpoint por lotes `music_generation`. Admite indicaciones, letras opcionales,
    modo instrumental y salida mp3 mediante autenticación con clave de API `minimax`
    o mediante OAuth de `minimax-portal`. También ofrece los modelos `music-2.6-free`,
    `music-cover` y `music-cover-free`.
  </Accordion>
  <Accordion title="OpenRouter">
    Usa la salida de audio de finalizaciones de chat de OpenRouter con la transmisión activada. El
    proveedor incluido utiliza `google/lyria-3-pro-preview` de forma predeterminada y también ofrece
    `openrouter/google/lyria-3-clip-preview`.
  </Accordion>
</AccordionGroup>

## Elección de la ruta adecuada

- **Con respaldo de proveedor compartido** cuando se requiera selección de modelos, conmutación
  por error entre proveedores y el flujo asíncrono integrado de tareas y estados.
- **Ruta del Plugin (ComfyUI)** cuando se necesite un grafo de flujo de trabajo personalizado o un
  proveedor que no forme parte de la capacidad compartida incluida de generación de música.

Para depurar comportamientos específicos de ComfyUI, consulte
[ComfyUI](/es/providers/comfy). Para depurar comportamientos de proveedores
compartidos, comience con [fal](/es/providers/fal), [Google (Gemini)](/es/providers/google),
[MiniMax](/es/providers/minimax) u [OpenRouter](/es/providers/openrouter).

## Modos de capacidad de proveedores

El contrato compartido de generación de música admite declaraciones explícitas de modo:

- `generate` para la generación basada únicamente en indicaciones.
- `edit` cuando la solicitud incluye una o más imágenes de referencia.

Las nuevas implementaciones de proveedores deben preferir bloques de modo explícitos:

```typescript
capabilities: {
  generate: {
    maxTracks: 1,
    supportsLyrics: true,
    supportsFormat: true,
  },
  edit: {
    enabled: true,
    maxTracks: 1,
    maxInputImages: 1,
    supportsFormat: true,
  },
}
```

Los campos planos heredados, como `maxInputImages`, `supportsLyrics` y
`supportsFormat`, **no** bastan para anunciar la compatibilidad con la edición. Los proveedores
deben declarar `generate` y `edit` explícitamente para que las pruebas en vivo, las pruebas de
contrato y la herramienta compartida `music_generate` puedan validar la compatibilidad con los modos
de manera determinista.

## Pruebas en vivo

Cobertura en vivo opcional para los proveedores compartidos incluidos (fal, Google, MiniMax,
OpenRouter):

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts
```

Contenedor equivalente del repositorio, que ejecuta el mismo archivo de prueba:

```bash
pnpm test:live:media:music
```

De forma predeterminada, este archivo de pruebas en vivo usa las variables de entorno de proveedores ya exportadas antes que los perfiles
de autenticación almacenados, y ejecuta tanto la cobertura de `generate` como la de `edit` declarada cuando
el proveedor activa el modo de edición. Cobertura actual:

- `google`: `generate` más `edit`
- `fal`: solo `generate`
- `minimax`: solo `generate`
- `openrouter`: `generate` más `edit`
- `comfy`: cobertura en vivo de Comfy independiente, no incluida en el barrido de proveedores compartidos

Cobertura en vivo opcional para la ruta incluida de música de ComfyUI:

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

El archivo de pruebas en vivo de Comfy también abarca los flujos de trabajo de imágenes y vídeos de Comfy cuando esas
secciones están configuradas.

## Temas relacionados

- [Tareas en segundo plano](/es/automation/tasks) — seguimiento de tareas para ejecuciones independientes de `music_generate`
- [ComfyUI](/es/providers/comfy)
- [Referencia de configuración](/es/gateway/config-agents#agent-defaults) — configuración de `musicGenerationModel`
- [Google (Gemini)](/es/providers/google)
- [MiniMax](/es/providers/minimax)
- [Modelos](/es/concepts/models) — configuración y conmutación por error de modelos
- [Descripción general de las herramientas](/es/tools)
