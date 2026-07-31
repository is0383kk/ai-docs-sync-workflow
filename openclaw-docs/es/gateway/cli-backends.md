---
read_when:
    - Se busca una alternativa fiable cuando fallan los proveedores de API
    - Se están ejecutando CLI de IA locales y se desea reutilizarlas
    - Quiere comprender el puente de bucle invertido MCP para el acceso a herramientas del backend de la CLI
summary: 'Backends de CLI: alternativa local de CLI de IA con puente opcional de herramientas MCP'
title: Backends de CLI
x-i18n:
    generated_at: "2026-07-26T04:37:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ce0427c587bf2a1e0a2ff24b5e76952eecae059e6f900af777b897b2d8d4210
    source_path: gateway/cli-backends.md
    workflow: 16
---

OpenClaw puede ejecutar una CLI de IA local como alternativa de solo texto cuando los proveedores de API no están disponibles, tienen límites de velocidad o funcionan incorrectamente. Es deliberadamente conservadora:

- Las herramientas de OpenClaw no se inyectan directamente, pero un backend con `bundleMcp: true` puede recibir herramientas del Gateway mediante un puente MCP de bucle invertido.
- Transmisión JSONL para las CLI que la admiten.
- Se admiten sesiones, por lo que los turnos posteriores mantienen la coherencia.
- Las imágenes se transfieren si la CLI acepta rutas de imágenes.

Debe usarse como red de seguridad para respuestas de texto que «siempre funcionan», no como ruta principal. Para un entorno de ejecución completo con controles de sesión ACP, tareas en segundo plano, vinculación de hilos/conversaciones y sesiones externas persistentes de programación, use [agentes ACP](/es/tools/acp-agents); los backends de CLI no son ACP.

<Tip>
  ¿Está creando un nuevo plugin de backend? Consulte [Plugins de backend de CLI](/es/plugins/cli-backend-plugins). Esta página explica cómo configurar y utilizar un backend ya registrado.
</Tip>

## Inicio rápido

El plugin de Anthropic incluido registra un backend `claude-cli` predeterminado, por lo que funciona sin ninguna configuración adicional aparte de tener Claude Code instalado y con la sesión iniciada:

```bash
openclaw agent --agent main --message "hi" --model claude-cli/claude-sonnet-4-6
```

`main` es el identificador de agente predeterminado cuando no se configura ninguna lista explícita de agentes; de lo contrario, sustitúyalo por el identificador de su propio agente.

El servicio del Gateway debe tener la CLI en su `PATH`. Si un despliegue necesita una
ruta de ejecutable o argumentos no estándar, registre ese adaptador en un
[plugin de backend de CLI](/es/plugins/cli-backend-plugins) en lugar de incluir los mecanismos
de inicio en `openclaw.json`.

OpenClaw carga automáticamente un plugin incluido propietario cuando la selección del modelo o una
referencia `agentRuntime.id` específica del modelo hace referencia a su backend.

## Uso como alternativa

Añada el backend de CLI a la lista de alternativas para que solo se ejecute cuando fallen los modelos principales:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["claude-cli/claude-sonnet-4-6"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "claude-cli/claude-sonnet-4-6": {},
      },
    },
  },
}
```

Las alternativas configuradas siguen siendo aptas cuando falla el proveedor principal (autenticación, límites de velocidad, tiempos de espera), incluso cuando no están en `agents.defaults.modelPolicy.allow`. Añada un modelo de backend de CLI a esa política solo cuando los usuarios también deban poder seleccionarlo directamente mediante `/model`, una anulación de sesión o `--model`. `agents.defaults.models` solo gestiona los alias, parámetros y metadatos de cada modelo.

## Configuración

Los usuarios eligen un backend registrado mediante la política del modelo y del entorno de ejecución. Mantenga
la referencia del modelo en su forma canónica y seleccione el entorno de ejecución de la CLI por modelo:

```json5
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-5",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
      },
    },
  },
}
```

Las credenciales permanecen en los perfiles de autenticación de OpenClaw o en la configuración del plugin propietario.
El comando, argv, el entorno, el análisis, la sesión, las imágenes y los mecanismos de vigilancia son
código del plugin registrado con `api.registerCliBackend(...)`.

## Cómo funciona

1. Selecciona un backend mediante el prefijo del proveedor (`claude-cli/...`).
2. Crea un prompt del sistema mediante el mismo prompt y contexto del espacio de trabajo de OpenClaw.
3. Ejecuta la CLI con un identificador de sesión (si se admite) para mantener la coherencia del historial. El backend `claude-cli` incluido mantiene activo un proceso stdio de Claude por sesión de OpenClaw y envía los turnos posteriores mediante la entrada estándar stream-json.
4. Analiza la salida (JSON o texto sin formato) y devuelve el texto final.
5. Conserva los identificadores de sesión por backend para que los turnos posteriores reutilicen la misma sesión de CLI.

## Tiempos de espera y trabajo de larga duración

Los backends de CLI tienen dos límites independientes:

- `agents.defaults.timeoutSeconds` limita el turno completo del agente. Los turnos normales del Gateway heredan el valor predeterminado de 48 horas; `0` hace que el presupuesto del turno sea ilimitado. Una anulación almacenada, como `600`, sustituye ese valor predeterminado.
- El supervisor de ausencia de salida de la CLI detiene un subproceso que permanece en silencio. Cada plugin de backend posee perfiles independientes para sesiones nuevas y reanudadas, y el supervisor permanece activo incluso cuando el presupuesto total del turno es ilimitado.

Elimine una anulación breve del tiempo de espera total para volver al valor predeterminado de 48 horas, o establezca un presupuesto explícito, como 12 horas:

```bash
# Volver al valor predeterminado de 48 horas:
openclaw config unset agents.defaults.timeoutSeconds

# O elegir un límite explícito de 12 horas:
openclaw config set agents.defaults.timeoutSeconds 43200
```

El trabajo en segundo plano iniciado dentro de una CLI sigue formando parte de ese subproceso de la CLI. Si el turno principal alcanza su límite total, OpenClaw detiene conjuntamente el subproceso y sus tareas internas en segundo plano. Para trabajos duraderos y prolongados, use un [subagente](/es/tools/subagents) separado de OpenClaw o un [agente ACP](/es/tools/acp-agents); de forma predeterminada, los subagentes separados no tienen tiempo de espera de ejecución.

El comando `openclaw agent` también tiene su propio plazo de solicitud. Su valor alternativo predeterminado de 600 segundos se aplica a la invocación de ese comando, no a los turnos normales del Gateway; consulte [`openclaw agent`](/es/cli/agent).

### Particularidades de la CLI de Claude

El backend `claude-cli` incluido prefiere el solucionador nativo de Skills de Claude Code. Cuando la instantánea actual de Skills tiene al menos una Skill seleccionada con una ruta materializada, OpenClaw pasa un plugin temporal de Claude Code mediante `--plugin-dir` y omite del prompt del sistema añadido el catálogo duplicado de Skills de OpenClaw. Sin una Skill de plugin materializada, OpenClaw conserva el catálogo del prompt como alternativa. Las anulaciones de entorno y clave de API de las Skills siguen aplicándose al entorno del proceso secundario durante la ejecución.

La CLI de Claude tiene su propio modo de permisos no interactivo; OpenClaw lo asigna a la política de ejecución existente en lugar de añadir una configuración específica de Claude. Para las sesiones activas de Claude gestionadas por OpenClaw, la política de ejecución efectiva es la autoridad: YOLO (`tools.exec.mode: "full"`) normalmente inicia Claude con `--permission-mode bypassPermissions`, mientras que una política restrictiva lo inicia con `--permission-mode default`. Los gateways ejecutados como root también usan `default`, ya que Claude Code rechaza el modo de omisión para root. Los ajustes `agents.entries.*.tools.exec` de cada agente prevalecen sobre el valor global `tools.exec` para ese agente. El plugin de Anthropic normaliza los indicadores de permisos de Claude para que coincidan con la política efectiva y la restricción del host.

Con una política restrictiva, Claude solicita permiso a OpenClaw mediante stdio antes de usar una de sus herramientas nativas o de extensión (sus propias herramientas Bash, WebFetch o del navegador Claude in Chrome). Cuando el ajuste efectivo de consulta de ejecución es `on-miss` o `always`, OpenClaw transmite cada solicitud como una aprobación interactiva al canal de la sesión: **Permitir una vez** permite la llamada individual, **Permitir siempre** permite ese nombre de herramienta durante el resto de la sesión activa de Claude (solo en memoria, nunca se conserva), y **Denegar**, un tiempo de espera agotado o una ruta de aprobación inaccesible deniegan la llamada. Las políticas que nunca solicitan confirmación mantienen su comportamiento anterior: `security: "deny"` rechaza todas las solicitudes, y la consulta `off` con un nivel de seguridad inferior al completo (modo de ejecución `allowlist`) las deniega sin preguntar.

### Herramientas de navegador de Claude e inicio de sesión con 1Password

Claude Code puede controlar un navegador Chrome mediante la [extensión Claude in Chrome](https://code.claude.com/docs/en/chrome), incluido el rellenado automático de credenciales de [1Password para Claude](/es/gateway/1password#browser-sign-in-with-1password-for-claude). El backend incluido no la activa; registre un [plugin de backend de CLI](/es/plugins/cli-backend-plugins) que añada `--chrome` a los argumentos de inicio de un backend con dialecto `claude-stream-json`. OpenClaw conserva un `--chrome` configurado en las ejecuciones normales y siempre fuerza `--no-chrome` en ejecuciones con una política de herramientas restringida, como las preguntas secundarias. La ventana de Chrome, la extensión y cualquier solicitud de aprobación de 1Password se encuentran en el host del Gateway, por lo que alguien debe estar en ese equipo para aprobar el uso de las credenciales.

El backend también asigna los niveles `/think` de OpenClaw al indicador nativo `--effort` de Claude Code: `minimal`/`low` -> `low`, `medium` -> `medium`, y `high`/`xhigh`/`max` se transfieren directamente. Esto mantiene iguales los niveles de esfuerzo admitidos de Fable 5 para la CLI de Claude respaldada por suscripción y las rutas con clave de API. `adaptive` elimina los indicadores `--effort` configurados y no proporciona ningún sustituto, por lo que Claude Code determina el esfuerzo efectivo a partir de su propio entorno, ajustes y valores predeterminados del modelo. Otros backends de CLI necesitan que su plugin propietario declare un asignador de argv equivalente antes de que `/think` afecte a la CLI iniciada.

Antes de que OpenClaw pueda usar `claude-cli`, Claude Code debe tener iniciada la sesión en el mismo host:

```bash
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

Las instalaciones de Docker necesitan que Claude Code esté instalado y tenga la sesión iniciada dentro del directorio de inicio persistente del contenedor, no solo en el host; consulte [Backend de la CLI de Claude en Docker](/es/install/docker#claude-cli-backend-in-docker).

El servicio del Gateway debe resolver `claude` en `PATH`. Para una ruta no estándar,
registre un pequeño plugin de backend envoltorio.

## Sesiones

- Si la CLI admite sesiones, establezca `sessionArgs` con un marcador de posición `{sessionId}` (por ejemplo, `["--session-id", "{sessionId}"]`).
- Si la CLI usa un subcomando de reanudación con indicadores distintos, establezca `resumeArgs` (sustituye a `args` al reanudar) y, opcionalmente, `resumeOutput` para reanudaciones que no sean JSON.
- `sessionMode`:
  - `always`: envía siempre un identificador de sesión (un UUID nuevo si no hay ninguno almacenado).
  - `existing`: solo envía un identificador de sesión si ya había uno almacenado.
  - `none`: nunca envía un identificador de sesión.
- `claude-cli` tiene como valores predeterminados `liveSession: "claude-stdio"`, `output: "jsonl"` y `input: "stdin"`, por lo que los turnos posteriores reutilizan el proceso activo de Claude mientras permanezca en ejecución, incluidas las configuraciones personalizadas que omiten campos de transporte. Si el Gateway se reinicia o el proceso inactivo termina, OpenClaw reanuda desde el identificador de sesión almacenado de Claude. Antes de reanudar, los identificadores de sesión almacenados se verifican mediante una transcripción legible del proyecto; si falta la transcripción, se elimina la vinculación (se registra como `reason=transcript-missing`) en lugar de iniciar silenciosamente una sesión nueva con `--resume`.
- Las sesiones activas de Claude mantienen límites acotados para la salida JSONL: 8 MiB y 20,000 líneas JSONL sin procesar por turno.
- Las sesiones de CLI almacenadas constituyen una continuidad propiedad del proveedor. El restablecimiento automático está desactivado de forma predeterminada; `/reset` y las políticas explícitas `session.reset` diarias o por inactividad siguen interrumpiéndolas.
- Las sesiones nuevas de CLI normalmente se reinicializan solo a partir del resumen de Compaction de OpenClaw y la parte posterior a la Compaction. Para recuperar sesiones cortas invalidadas antes de la Compaction, un backend puede habilitar `reseedFromRawTranscriptWhenUncompacted: true`. La reinicialización mediante la transcripción sin procesar permanece acotada y limitada a invalidaciones seguras, como la ausencia de una transcripción de la CLI, una parte final huérfana de uso de herramientas, cambios en la política de mensajes, el prompt del sistema, cwd o MCP, o un reintento por sesión caducada; los cambios en el perfil de autenticación o en la época de las credenciales nunca reinicializan el historial de la transcripción sin procesar.

Serialización: `serialize: true` mantiene ordenadas las ejecuciones del mismo carril (la mayoría de las CLI se serializan en un solo carril de proveedor). OpenClaw también descarta la reutilización de la sesión de CLI almacenada cuando cambia la identidad de autenticación seleccionada, incluidos los cambios en el identificador del perfil de autenticación, la clave de API estática, el token estático o la identidad de la cuenta OAuth cuando la CLI expone una; la rotación de tokens de acceso o actualización de OAuth por sí sola no interrumpe la sesión. Si una CLI no dispone de un identificador estable de cuenta OAuth, OpenClaw permite que esa CLI aplique sus propios permisos de reanudación.

## Preámbulo alternativo de las sesiones de claude-cli

Cuando un intento de `claude-cli` conmuta por error a un candidato que no es de CLI en [`agents.defaults.model.fallbacks`](/es/concepts/model-failover), OpenClaw inicializa el siguiente intento con un preámbulo de contexto extraído de la transcripción JSONL local de Claude Code (en `~/.claude/projects/`, con una clave por espacio de trabajo). Sin esta inicialización, el proveedor alternativo comienza sin contexto, ya que la transcripción de sesión propia de OpenClaw está vacía para las ejecuciones de `claude-cli`.

- El preámbulo prioriza el resumen de `/compact` o el marcador de `compact_boundary` más reciente y, a continuación, agrega los turnos posteriores al límite más recientes hasta alcanzar un presupuesto de caracteres. Los turnos anteriores al límite se descartan porque el resumen ya los representa.
- Los bloques de herramientas se combinan en indicaciones compactas de `(tool call: name)` y `(tool result: …)` para respetar el presupuesto del prompt; un resumen demasiado grande se trunca y se etiqueta como `(truncated)`.
- Las alternativas del mismo proveedor de `claude-cli` a `claude-cli` dependen del propio `--resume` de Claude y omiten el preámbulo.
- La inicialización reutiliza la validación existente de la ruta del archivo de sesión de Claude, por lo que no se pueden leer rutas arbitrarias.

## Imágenes

Los autores de Plugins declaran la compatibilidad con rutas de imágenes mediante `imageArg`:

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw escribe las imágenes en base64 en archivos temporales. Si se establece `imageArg`, esas rutas se pasan como argumentos de la CLI; de lo contrario, OpenClaw agrega las rutas de los archivos al prompt (inyección de rutas), lo que funciona con las CLI que cargan automáticamente archivos locales a partir de rutas simples.

## Entradas y salidas

- `output: "text"` (valor predeterminado) trata stdout como la respuesta final.
- `output: "json"` intenta analizar JSON y extraer el texto junto con un identificador de sesión.
- `output: "jsonl"` analiza un flujo JSONL y extrae el mensaje final del agente junto con los identificadores de sesión cuando están presentes.
- Para la salida JSON de la CLI de Gemini, OpenClaw lee el texto de respuesta de `response` y el uso de `stats` cuando `usage` falta o está vacío. El adaptador integrado de la CLI de Gemini utiliza `stream-json`.

Modos de entrada:

- `input: "arg"` (valor predeterminado) pasa el prompt como último argumento de la CLI.
- `input: "stdin"` envía el prompt mediante stdin.
- Si el prompt es muy largo y se establece `maxPromptArgChars`, se utiliza stdin en su lugar.

## Valores predeterminados propiedad del Plugin

Los valores predeterminados del backend de CLI forman parte de la superficie del Plugin:

- Los Plugins los registran mediante `api.registerCliBackend(...)`.
- El `id` del backend se convierte en el prefijo del proveedor en las referencias de modelos.
- El comportamiento del comando, argv, entorno, analizador, sesión y mecanismo de vigilancia permanece en el código del Plugin.
- La normalización específica del backend sigue siendo propiedad del Plugin mediante el enlace opcional `normalizeConfig`.

Anthropic es propietario de `claude-cli` y Google es propietario de `google-gemini-cli`. Las ejecuciones del agente OpenAI Codex utilizan el entorno del servidor de aplicaciones de Codex mediante `openai/*`; OpenClaw ya no registra un backend `codex-cli` integrado.

El Plugin integrado de Anthropic se registra para `claude-cli`:

| Clave                   | Valor                                                                                                                                                                                                         |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`             | `claude`                                                                                                                                                                                                      |
| `args`                | `-p --output-format stream-json --include-partial-messages --verbose --setting-sources user --allowedTools mcp__openclaw__* --disallowedTools ScheduleWakeup,CronCreate,Bash(run_in_background:true),Monitor` |
| `output`              | `jsonl`                                                                                                                                                                                                       |
| `input`               | `stdin`                                                                                                                                                                                                       |
| `modelArg`            | `--model`                                                                                                                                                                                                     |
| `sessionArgs`         | `["--session-id", "{sessionId}"]`                                                                                                                                                                             |
| `sessionMode`         | `always`                                                                                                                                                                                                      |
| `imageArg`            | `@`                                                                                                                                                                                                           |
| `imagePathScope`      | `workspace`                                                                                                                                                                                                   |
| `systemPromptFileArg` | `--append-system-prompt-file`                                                                                                                                                                                 |
| `systemPromptMode`    | `append`                                                                                                                                                                                                      |

El Plugin integrado de Google se registra para `google-gemini-cli`:

| Clave                       | Valor                                                                                  |
| ------------------------- | -------------------------------------------------------------------------------------- |
| `command`                 | `gemini`                                                                               |
| `args`                    | `--skip-trust --approval-mode auto_edit --output-format stream-json --prompt {prompt}` |
| `resumeArgs`              | igual, con `--resume {sessionId}`                                                      |
| `output` / `resumeOutput` | `jsonl`                                                                                |
| `jsonlDialect`            | `gemini-stream-json`                                                                   |
| `imageArg`                | `@`                                                                                    |
| `imagePathScope`          | `workspace`                                                                            |
| `modelArg`                | `--model`                                                                              |
| `sessionMode`             | `existing`                                                                             |
| `sessionIdFields`         | `["session_id", "sessionId"]`                                                          |

Requisito previo: la CLI local de Gemini debe estar instalada y disponible en `PATH` como `gemini` (`brew install gemini-cli` o `npm install -g @google/gemini-cli`).

Notas sobre la salida de la CLI de Gemini:

- El analizador `stream-json` predeterminado lee los eventos `message` del asistente, los eventos de herramientas, el uso final de `result` y los eventos de error fatal de Gemini.
- El uso recurre a `stats` cuando `usage` no está presente o está vacío; `stats.cached` se normaliza como `cacheRead` de OpenClaw y, si falta `stats.input`, los tokens de entrada se derivan de `stats.input_tokens - stats.cached`.

## Superposiciones de transformación de texto

Los Plugins que necesitan pequeños adaptadores de compatibilidad de prompts o mensajes pueden declarar transformaciones de texto bidireccionales sin sustituir un proveedor ni un backend de CLI:

```typescript
api.registerTextTransforms({
  input: [{ from: /red basket/g, to: "blue basket" }],
  output: [{ from: /blue basket/g, to: "red basket" }],
});
```

`input` reescribe el prompt del sistema y el prompt del usuario que se pasan a la CLI. `output` reescribe el texto transmitido del asistente y el texto final analizado antes de que OpenClaw procese sus propios marcadores de control y la entrega al canal; en las llamadas de modelos respaldadas por proveedores, también restaura los valores de cadena dentro de los argumentos estructurados de llamadas a herramientas después de reparar el flujo y antes de ejecutar la herramienta. Los fragmentos JSON sin procesar del proveedor permanecen sin cambios; los consumidores deben utilizar la carga útil estructurada parcial, final o de resultado.

Para las CLI que emiten eventos JSONL específicos del proveedor, establezca `jsonlDialect` en la configuración de ese backend: `claude-stream-json` para flujos compatibles con Claude Code y `gemini-stream-json` para eventos `stream-json` de la CLI de Gemini.

## Propiedad de Compaction nativa

Algunos backends de CLI ejecutan un agente que compacta su propia transcripción, por lo que OpenClaw no debe ejecutar su resumidor de protección sobre ellos, ya que hacerlo interfiere con la propia Compaction del backend y puede provocar un fallo irreversible del turno.

`claude-cli` no tiene un punto de conexión del entorno (Claude Code realiza la Compaction internamente), por lo que declara `ownsNativeCompaction: true` y la ruta de Compaction de OpenClaw devuelve la entrada de sesión sin cambios. OpenClaw pasa el presupuesto de contexto efectivo de la ejecución mediante la variable documentada [`CLAUDE_CODE_AUTO_COMPACT_WINDOW`](https://code.claude.com/docs/en/env-vars) de Claude Code, lo que mantiene la Compaction automática nativa alineada con los límites configurados de `contextTokens` de Anthropic. En cambio, las sesiones con entorno nativo, como Codex, siguen dirigiéndose al punto de conexión de Compaction de su entorno.

```typescript
api.registerCliBackend({ id: "my-cli", ownsNativeCompaction: true /* ... */ });
```

Declare `ownsNativeCompaction` únicamente para un backend que sea realmente propietario de la Compaction: debe limitar de forma fiable su propia transcripción cerca de la ventana de contexto y conservar una sesión reanudable (por ejemplo, `--resume` / `--session-id`); de lo contrario, una sesión diferida puede permanecer por encima del presupuesto.

## Superposiciones de MCP del paquete

Los backends de CLI no reciben directamente las llamadas a herramientas de OpenClaw, pero un backend puede habilitar una superposición de configuración de MCP generada mediante `bundleMcp: true`. Comportamiento integrado actual:

- `claude-cli`: archivo de configuración estricta de MCP generado.
- `google-gemini-cli`: archivo generado de configuración del sistema de Gemini.

Cuando se habilita el MCP del paquete, OpenClaw:

- inicia un servidor MCP HTTP de bucle local que expone las herramientas del Gateway al proceso de la CLI, autenticado mediante una concesión de contexto por ejecución (`OPENCLAW_MCP_TOKEN`) activa únicamente durante el intento de ejecución actual;
- vincula el acceso a las herramientas con el contexto de sesión, cuenta y canal seleccionado por el Gateway, en lugar de confiar en los encabezados del proceso secundario;
- carga los servidores MCP del paquete habilitados para el espacio de trabajo actual y los combina con cualquier estructura existente de configuración o ajustes de MCP del backend;
- reescribe la configuración de inicio mediante el modo de integración propiedad del backend del Plugin propietario.

Las ejecuciones restringidas, como los trabajos de Cron con `toolsAllow`, requieren una traducción exacta
propiedad del backend. El backend `claude-cli` incluido deshabilita las herramientas
nativas de Claude y las personalizaciones del usuario, del proyecto y locales, incluidos hooks,
plugins, agentes, Skills y `CLAUDE.md`. A continuación, expone todas las herramientas
permitidas de OpenClaw mediante el servidor MCP con ámbito de concesión. Esto mantiene las políticas
del sistema de archivos, procesos, ejecución, aprobación y sandbox dentro de OpenClaw, en lugar de ampliar
la autoridad a las herramientas nativas o los procesos de personalización de Claude. La misma lista de MCP
se aplica en la configuración generada de Claude y, de nuevo, mediante el Gateway al enumerar
y ejecutar herramientas. Antes de emitir la concesión, el núcleo rechaza las
traducciones del backend que incluyan cualquier permiso de MCP fuera de la lista de permitidos original.
Los backends sin una traducción exacta siguen denegando el acceso de forma predeterminada.

Si no hay servidores MCP habilitados, OpenClaw sigue inyectando una configuración estricta cuando un backend opta por el MCP incluido, de modo que las ejecuciones en segundo plano permanezcan aisladas.

Los entornos de ejecución de MCP incluidos y con ámbito de sesión se almacenan en caché para reutilizarlos durante una sesión y se eliminan después de 10 minutos de inactividad. Las ejecuciones integradas de un solo uso, como las pruebas de autenticación, la generación de slugs y la recuperación de Active Memory, solicitan la limpieza al finalizar la ejecución para que los procesos secundarios de stdio y los flujos HTTP/SSE transmisibles no sobrevivan a la ejecución.

Para `claude-cli`, se reenvía a ese proceso secundario de Claude un perfil
de OAuth/token de OpenClaw seleccionado u ordenado que sea compatible. Esto hace que los perfiles por agente sean la fuente autoritativa
para el turno, a la vez que conserva el inicio de sesión nativo de Claude en el host cuando no existe ningún perfil
compatible.

## Límite del historial de reinicialización

Cuando se inicializa una sesión nueva de la CLI a partir de una transcripción anterior de OpenClaw (por ejemplo, después de un reintento de `session_expired`), el bloque `<conversation_history>` renderizado se limita para evitar que las indicaciones de reinicialización crezcan desmesuradamente. El valor predeterminado es de 12,288 caracteres (unos 3,000 tokens).

En su lugar, los backends de la CLI de Claude ajustan este límite según la ventana de contexto resuelta de Claude: las ventanas de contexto más grandes reciben una porción mayor del historial anterior, hasta un límite máximo fijo; los demás backends de la CLI mantienen el valor predeterminado conservador. Este límite solo controla el bloque de historial anterior de la indicación de reinicialización.

## Limitaciones

- OpenClaw no inyecta llamadas a herramientas en el protocolo del backend de la CLI. Los backends solo ven las herramientas del Gateway cuando habilitan `bundleMcp: true`.
- La transmisión depende del backend: algunos backends transmiten JSONL y otros almacenan en búfer hasta que finaliza el proceso.
- Las salidas estructuradas dependen del formato JSON propio de la CLI.

## Solución de problemas

| Síntoma                       | Solución                                                                                                                   |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| No se encuentra la CLI        | Añada la CLI al `PATH` del servicio del Gateway o actualice el comando registrado del plugin propietario.      |
| Nombre de modelo incorrecto   | Actualice la asignación `modelAliases` del plugin.                                                                      |
| Sin continuidad de sesión     | Compruebe los valores `sessionArgs` y `sessionMode` del plugin.                                                   |
| Se ignoran las imágenes       | Compruebe `imageArg` del plugin y la compatibilidad de la CLI con rutas de archivos.                                |

## Temas relacionados

- [Manual operativo del Gateway](/es/gateway)
- [Modelos locales](/es/gateway/local-models)
