---
read_when:
    - Se desea una automatización basada en eventos para /new, /reset, /stop y los eventos del ciclo de vida del agente
    - Quieres crear, instalar o depurar hooks
summary: 'Hooks: automatización basada en eventos para comandos y eventos del ciclo de vida'
title: Ganchos
x-i18n:
    generated_at: "2026-07-26T04:30:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 039a55cca60e0005d7b9c4d950a86aceb6e7c29d5768108b34011bfc21c85be6
    source_path: automation/hooks.md
    workflow: 16
---

Los hooks son pequeños scripts que se ejecutan dentro del Gateway cuando se producen eventos del agente: comandos como `/new`, `/reset`, `/stop`, Compaction de sesión, ciclo de vida del Gateway y flujo de mensajes. Se detectan en directorios y se gestionan con `openclaw hooks`. El Gateway carga los hooks internos únicamente después de habilitar los hooks o configurar al menos una entrada de hook, un paquete de hooks, un controlador heredado o un directorio adicional de hooks.

Hay dos tipos de hooks en OpenClaw:

- **Hooks internos** (esta página): se ejecutan dentro del Gateway cuando se producen eventos del agente.
- **Webhooks**: endpoints HTTP externos que permiten que otros sistemas activen tareas en OpenClaw. Consulte [Webhooks](/es/automation/cron-jobs#webhooks).

Los hooks también pueden incluirse dentro de plugins. `openclaw hooks list` muestra tanto los hooks independientes como los gestionados por plugins (que aparecen como `plugin:<id>`).

## Elegir la superficie adecuada

OpenClaw tiene varias superficies de extensión que parecen similares, pero resuelven problemas distintos:

| Si se desea...                                                                                                     | Usar...                                | Motivo                                                                                           |
| --------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------- |
| Guardar una instantánea en `/new`, registrar `/reset`, llamar a una API externa después de `message:sent` o añadir automatización general para operadores | Hooks internos (`HOOK.md`, esta página) | Los hooks basados en archivos están diseñados para efectos secundarios gestionados por el operador y automatización de comandos y del ciclo de vida |
| Reescribir prompts, bloquear herramientas, cancelar mensajes salientes o añadir middleware o políticas ordenados                              | Hooks de plugin tipados mediante `api.on(...)`  | Los hooks tipados tienen contratos, prioridades, reglas de combinación y semántica de bloqueo y cancelación explícitos      |
| Añadir exportación exclusiva de telemetría u observabilidad                                                                            | Eventos de diagnóstico                     | La observabilidad es un bus de eventos independiente, no una superficie de hooks de políticas                              |

Utilice hooks internos cuando necesite una automatización que se comporte como una pequeña integración instalada. Utilice hooks de plugin tipados cuando necesite controlar el ciclo de vida del entorno de ejecución.

## Inicio rápido

```bash
# Enumerar los hooks disponibles
openclaw hooks list

# Habilitar un hook
openclaw hooks enable session-memory

# Comprobar el estado de los hooks
openclaw hooks check

# Obtener información detallada
openclaw hooks info session-memory
```

## Tipos de eventos

Los hooks se suscriben a una clave específica de esta tabla o a un nombre de familia sin calificar
(`command`, `session`, `agent`, `gateway`, `message`) para recibir todas las acciones
de esa familia. El núcleo de OpenClaw no emite ningún otro evento, por lo que cualquier otro nombre es casi
siempre un error tipográfico que deja el hook inactivo de forma silenciosa (solo podría activarlo un plugin que emitiera un
evento personalizado). El cargador de hooks registra una advertencia para esos nombres
(por ejemplo, `command:nwe`) y `openclaw hooks info <name>` los marca, por lo que es posible
diagnosticar un hook que nunca se ejecuta.

| Evento                    | Cuándo se activa                                              |
| ------------------------ | ---------------------------------------------------------- |
| `command:new`            | Se emite el comando `/new`                                      |
| `command:reset`          | Se emite el comando `/reset`                                    |
| `command:stop`           | Se emite el comando `/stop`                                     |
| `command`                | Cualquier evento de comando (escucha general)                       |
| `session:compact:before` | Antes de que Compaction resuma el historial                       |
| `session:compact:after`  | Después de que finalice Compaction                                 |
| `session:patch`          | Cuando se modifican las propiedades de la sesión                       |
| `agent:bootstrap`        | Antes de inyectar los archivos de arranque del espacio de trabajo              |
| `gateway:startup`        | Después de que se inicien los canales y se carguen los hooks                  |
| `gateway:shutdown`       | Cuando comienza el cierre del Gateway                               |
| `gateway:pre-restart`    | Antes de un reinicio previsto del Gateway                         |
| `message:received`       | Mensaje entrante de cualquier canal                           |
| `message:transcribed`    | Después de que finalice la transcripción de audio                        |
| `message:preprocessed`   | Después de que finalice o se omita el preprocesamiento de contenidos multimedia y enlaces |
| `message:sent`           | Se intentó realizar un envío saliente (`context.success` contiene el resultado) |

## Escribir hooks

### Estructura de un hook

Cada hook es un directorio que contiene dos archivos:

```text
my-hook/
├── HOOK.md          # Metadatos y documentación
└── handler.ts       # Implementación del controlador
```

El archivo del controlador puede ser `handler.ts`, `handler.js`, `index.ts` o `index.js`.

### Formato de HOOK.md

```markdown
---
name: my-hook
description: "Descripción breve de lo que hace este hook"
metadata:
  { "openclaw": { "emoji": "🔗", "events": ["command:new"], "requires": { "bins": ["node"] } } }
---

# Mi hook

Aquí se incluye la documentación detallada.
```

**Campos de metadatos** (`metadata.openclaw`):

| Campo      | Descripción                                          |
| ---------- | ---------------------------------------------------- |
| `emoji`    | Emoji mostrado en la CLI                                |
| `events`   | Matriz de eventos que se deben escuchar                        |
| `export`   | Exportación con nombre que se debe utilizar (el valor predeterminado es `"default"`)        |
| `os`       | Plataformas requeridas (por ejemplo, `["darwin", "linux"]`)     |
| `requires` | Rutas de `bins`, `anyBins`, `env` o `config` requeridas |
| `always`   | Omitir las comprobaciones de idoneidad (booleano)                  |
| `hookKey`  | Sustitución de la clave de configuración (el valor predeterminado es el nombre del hook)      |
| `homepage` | URL de documentación que muestra `openclaw hooks info`              |
| `install`  | Métodos de instalación                                 |

### Implementación del controlador

```typescript
const handler = async (event) => {
  if (event.type !== "command" || event.action !== "new") {
    return;
  }

  console.log(`[my-hook] Se ha activado un nuevo comando`);
  // Su lógica aquí

  // Enviar opcionalmente una respuesta en superficies que admitan respuestas
  event.messages.push("¡Hook ejecutado!");
};

export default handler;
```

Cada evento incluye: `type`, `action`, `sessionKey`, `timestamp`, `messages` y `context` (datos específicos del evento). Los contextos de hooks de plugin tipados para hooks de agentes y herramientas también pueden incluir `trace`, un contexto de seguimiento de diagnóstico de solo lectura compatible con W3C que los plugins pueden pasar a registros estructurados para la correlación con OTEL.

Las cadenas añadidas a `event.messages` se devuelven al chat únicamente para
`command:new` y `command:reset` (enrutadas como respuesta a la conversación
de origen) y para `session:compact:before` / `session:compact:after`
(enviadas como avisos del estado de Compaction). Todos los demás eventos, incluidos
`command:stop`, `message:*`, `agent:bootstrap`, `session:patch` y
`gateway:*`, ignoran los mensajes añadidos.

### Aspectos destacados del contexto de los eventos

**Eventos de comando** (`command:new`, `command:reset`): `context.sessionEntry`, `context.previousSessionEntry`, `context.commandSource`, `context.senderId`, `context.workspaceDir`, `context.cfg`.

**Eventos de comando** (`command:stop`): `context.sessionEntry`, `context.sessionId`, `context.commandSource`, `context.senderId`.

**Eventos de mensaje** (`message:received`): `context.from`, `context.content`, `context.channelId`, `context.media` (datos de archivos adjuntos preparados y ordenados), `context.originalMedia` junto con `context.mediaStagingPending` cuando el contenido multimedia remoto aún no se ha preparado localmente, y `context.metadata` (datos específicos del proveedor, incluidos `senderId`, `senderName`, `guildId`). `context.content` da preferencia a un cuerpo de comando que no esté en blanco para los mensajes similares a comandos y, a continuación, recurre al cuerpo entrante sin procesar y al cuerpo genérico; no incluye enriquecimiento exclusivo del agente, como el historial del hilo o los resúmenes de enlaces. Los alias de contenido multimedia heredados dentro de `metadata` están obsoletos.

**Eventos de mensaje** (`message:sent`): `context.to`, `context.content`, `context.success`, `context.channelId`, además de `context.error` cuando se produce un error en el envío.

**Eventos de mensaje** (`message:transcribed`): `context.transcript`, `context.from`, `context.channelId` y `context.media`. `context.mediaPath` y `context.mediaType` siguen siendo alias obsoletos del primer dato.

**Eventos de mensaje** (`message:preprocessed`): `context.bodyForAgent` (cuerpo final enriquecido), `context.from`, `context.channelId`.

**Eventos de arranque** (`agent:bootstrap`): `context.bootstrapFiles` (matriz mutable), `context.agentId`.

**Eventos de modificación de sesión** (`session:patch`): `context.sessionEntry`, `context.patch` (solo los campos modificados), `context.cfg`. Solo los clientes con privilegios pueden activar eventos de modificación; el contexto es un clon, por lo que los controladores no pueden modificar la entrada de sesión activa.

**Eventos de Compaction**: `session:compact:before` incluye `messageCount`, `tokenCount`. `session:compact:after` añade `compactedCount`, `summaryLength`, `tokensBefore`, `tokensAfter`.

`command:stop` observa que el usuario emite `/stop`; pertenece al ciclo de vida de cancelación/comando,
no es una barrera de finalización del agente. Los plugins que necesiten inspeccionar una
respuesta final natural y pedir al agente una iteración más deben utilizar en su lugar el hook
de plugin tipado `before_agent_finalize`. Consulte [Hooks de plugins](/es/plugins/hooks).

**Eventos del ciclo de vida del Gateway**: `gateway:shutdown` incluye `reason` y `restartExpectedMs`, y se activa cuando comienza el cierre del Gateway. `gateway:pre-restart` incluye el mismo contexto, pero solo se activa cuando el cierre forma parte de un reinicio previsto y se proporciona un valor finito de `restartExpectedMs`. Durante el cierre, la espera de cada hook del ciclo de vida se realiza según el mejor esfuerzo y está limitada, de modo que el cierre continúa si un controlador se bloquea. El tiempo de espera predeterminado es de 5 segundos para `gateway:shutdown` y de 10 segundos para `gateway:pre-restart`.

Utilice `gateway:pre-restart` para enviar avisos breves de reinicio mientras los canales sigan disponibles:

```typescript
import { execFile } from "node:child_process";
import { promisify } from "node:util";

const execFileAsync = promisify(execFile);

export default async function handler(event) {
  if (event.type !== "gateway" || event.action !== "pre-restart") {
    return;
  }

  const restartInSeconds = Math.ceil(event.context.restartExpectedMs / 1000);
  await execFileAsync("openclaw", [
    "system",
    "event",
    "--mode",
    "now",
    "--text",
    `El Gateway se reiniciará en ~${restartInSeconds}s (${event.context.reason}). Cree ahora un punto de control.`,
  ]);
}
```

Entre el evento `gateway:shutdown` (o `gateway:pre-restart`) y el resto de la secuencia de cierre, el Gateway también activa un hook de plugin tipado `session_end` para cada sesión que seguía activa cuando se detuvo el proceso. El valor de `reason` del evento es `shutdown` para una detención normal mediante SIGTERM/SIGINT y `restart` cuando el cierre se programó como parte de un reinicio previsto. Este vaciado está limitado para que un controlador `session_end` lento no pueda bloquear la salida del proceso, y se omiten las sesiones que ya se hayan finalizado mediante sustitución / restablecimiento / eliminación / Compaction para evitar que se active dos veces.

## Detección de hooks

Los hooks se detectan en cuatro fuentes:

1. **Hooks incluidos**: se distribuyen con OpenClaw
2. **Hooks de plugins**: incluidos dentro de los plugins instalados; pueden reemplazar los hooks incluidos que tengan el mismo nombre
3. **Hooks gestionados**: `~/.openclaw/hooks/` (instalados por el usuario y compartidos entre espacios de trabajo); pueden reemplazar los hooks incluidos y los de plugins. Los directorios adicionales de `hooks.internal.load.extraDirs` comparten esta precedencia.
4. **Hooks del espacio de trabajo**: `<workspace>/hooks/` (por agente, deshabilitados de forma predeterminada hasta que se habiliten explícitamente)

Los hooks del espacio de trabajo pueden añadir nombres de hooks nuevos, pero no pueden reemplazar hooks incluidos, gestionados o proporcionados por plugins que tengan el mismo nombre.

El Gateway omite la detección de hooks internos durante el inicio hasta que estos se configuren. Habilite un hook incluido o gestionado con `openclaw hooks enable <name>`, instale un paquete de hooks o establezca `hooks.internal.enabled=true` para activarla. Cuando se habilita un hook por su nombre, el Gateway carga únicamente el controlador de ese hook; `hooks.internal.enabled=true`, los directorios de hooks adicionales y los controladores heredados activan la detección amplia.

### Paquetes de hooks

Los paquetes de hooks son paquetes npm que exportan hooks mediante `openclaw.hooks` en `package.json`. Instálelos con:

```bash
openclaw plugins install <path-or-spec>
```

Las especificaciones de npm se limitan al registro (nombre del paquete + versión exacta opcional o etiqueta de distribución). Se rechazan las especificaciones de Git/URL/archivo y los rangos semver. Los comandos antiguos `openclaw hooks install` y `openclaw hooks update` son alias obsoletos de `openclaw plugins install` / `openclaw plugins update`.

## Hooks incluidos

| Hook                  | Eventos                                           | Qué hace                                                        |
| --------------------- | ------------------------------------------------- | --------------------------------------------------------------- |
| session-memory        | `command:new`, `command:reset`                    | Guarda el contexto de la sesión en `<workspace>/memory/`        |
| bootstrap-extra-files | `agent:bootstrap`                                 | Inyecta archivos de arranque adicionales a partir de patrones glob |
| command-logger        | `command`                                         | Registra todos los comandos en `~/.openclaw/logs/commands.log`  |
| compaction-notifier   | `session:compact:before`, `session:compact:after` | Envía avisos visibles al chat cuando comienza o termina la compactación de la sesión |
| boot-md               | `gateway:startup`                                 | Ejecuta `BOOT.md` cuando se inicia el Gateway                  |

Habilite cualquier hook incluido:

```bash
openclaw hooks enable <hook-name>
```

<a id="session-memory"></a>

### Detalles de session-memory

Extrae los últimos mensajes del usuario y del asistente (15 de forma predeterminada, configurable con `hooks.internal.entries.session-memory.messages`) y los guarda en `<workspace>/memory/YYYY-MM-DD-HHMM.md` utilizando la fecha local del host. La captura de memoria se ejecuta en segundo plano para que las confirmaciones de `/new` y `/reset` no se retrasen por la lectura de la transcripción ni por la generación opcional del slug. Establezca `hooks.internal.entries.session-memory.llmSlug: true` para generar slugs descriptivos para los nombres de archivo y, opcionalmente, establezca `hooks.internal.entries.session-memory.model` en un alias configurado, como `sonnet`, un ID de modelo sin calificar del proveedor predeterminado del agente o una referencia `provider/model`. La generación del slug usa el modelo predeterminado del agente cuando se omite `model` y recurre a slugs de marca de tiempo cuando no está disponible. Requiere que `workspace.dir` esté configurado.

<a id="bootstrap-extra-files"></a>

### Configuración de bootstrap-extra-files

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "bootstrap-extra-files": {
          "enabled": true,
          "paths": ["packages/*/AGENTS.md", "packages/*/TOOLS.md"]
        }
      }
    }
  }
}
```

`patterns` y `files` se aceptan como alias de `paths`. Las rutas se resuelven con respecto al espacio de trabajo y deben permanecer dentro de él. Solo se cargan los nombres base de arranque reconocidos (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`, `MEMORY.md`).

<a id="command-logger"></a>

### Detalles de command-logger

Registra cada comando con barra como una línea JSON (marca de tiempo, acción, clave de sesión, ID del remitente, origen) en `~/.openclaw/logs/commands.log`.

<a id="compaction-notifier"></a>

### Detalles de compaction-notifier

Envía mensajes de estado breves a la conversación actual cuando OpenClaw comienza y termina de compactar la transcripción de la sesión. Esto hace que los turnos largos sean menos confusos en las interfaces de chat, porque el usuario puede ver que el asistente está resumiendo el contexto y continuará después de la compactación.

<a id="boot-md"></a>

### Detalles de boot-md

Ejecuta `BOOT.md` al iniciar el Gateway para cada ámbito de agente configurado, si el archivo existe en el espacio de trabajo resuelto de ese agente.

## Hooks de plugins

Los plugins pueden registrar hooks tipados mediante el SDK de Plugin para una integración más profunda:
interceptar llamadas a herramientas, modificar prompts, controlar el flujo de mensajes y mucho más.
Utilice hooks de plugins cuando necesite `before_tool_call`, `before_agent_reply`,
`before_install` u otros hooks de ciclo de vida en proceso.

Los hooks internos gestionados por plugins son diferentes: participan en el sistema general de eventos de comandos y del ciclo de vida de esta página y aparecen en `openclaw hooks list` como
`plugin:<id>`. Utilícelos para efectos secundarios y compatibilidad con paquetes de hooks, no
como middleware ordenado ni como barreras de políticas.

Para consultar la referencia completa de hooks de plugins, véase [Hooks de plugins](/es/plugins/hooks).

## Configuración

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "session-memory": { "enabled": true },
        "command-logger": { "enabled": false }
      }
    }
  }
}
```

Los valores de entorno por hook satisfacen las comprobaciones de elegibilidad `requires.env` de un hook (junto con el entorno del proceso), y los controladores pueden leerlos desde la entrada de configuración del hook:

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "my-hook": {
          "enabled": true,
          "env": { "MY_CUSTOM_VAR": "value" }
        }
      }
    }
  }
}
```

Directorios de hooks adicionales:

```json
{
  "hooks": {
    "internal": {
      "load": {
        "extraDirs": ["/path/to/more/hooks"]
      }
    }
  }
}
```

<Note>
El formato de configuración de matriz heredado `hooks.internal.handlers` sigue siendo compatible por retrocompatibilidad, pero los hooks nuevos deben utilizar el sistema basado en detección.
</Note>

## Referencia de la CLI

```bash
# Enumerar todos los hooks (añada --eligible, --verbose o --json)
openclaw hooks list

# Mostrar información detallada sobre un hook
openclaw hooks info <hook-name>

# Mostrar el resumen de elegibilidad
openclaw hooks check

# Habilitar/deshabilitar
openclaw hooks enable <hook-name>
openclaw hooks disable <hook-name>
```

## Prácticas recomendadas

- **Mantenga los controladores rápidos.** Los hooks se ejecutan durante el procesamiento de comandos. Ejecute el trabajo pesado sin esperar su resultado mediante `void processInBackground(event)`.
- **Gestione los errores correctamente.** Envuelva las operaciones de riesgo en try/catch; no lance excepciones, para que puedan ejecutarse los demás controladores.
- **Filtre los eventos cuanto antes.** Finalice inmediatamente si el tipo o la acción del evento no son pertinentes.
- **Utilice claves de evento específicas.** Prefiera `"events": ["command:new"]` a `"events": ["command"]` para reducir la sobrecarga.

## Solución de problemas

### No se detecta el hook

```bash
# Verificar la estructura del directorio
ls -la ~/.openclaw/hooks/my-hook/
# Debe mostrar: HOOK.md, handler.ts

# Enumerar todos los hooks detectados
openclaw hooks list
```

### El hook no es apto

```bash
openclaw hooks info my-hook
```

Compruebe si faltan archivos binarios (PATH), variables de entorno o valores de configuración, o si hay problemas de compatibilidad con el sistema operativo.

### El hook no se ejecuta

1. Compruebe que el hook esté habilitado: `openclaw hooks list`
2. Reinicie el proceso del Gateway para que los hooks se vuelvan a cargar.
3. Compruebe los registros del Gateway: `openclaw logs --follow | grep -i hook`

## Contenido relacionado

- [Referencia de la CLI: hooks](/es/cli/hooks)
- [Webhooks](/es/automation/cron-jobs#webhooks)
- [Hooks de plugins](/es/plugins/hooks) — hooks de ciclo de vida de plugins en proceso
- [Configuración](/es/gateway/configuration-reference#hooks)
