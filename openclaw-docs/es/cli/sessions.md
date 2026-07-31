---
read_when:
    - Quieres enumerar las sesiones almacenadas y ver la actividad reciente
summary: Referencia de la CLI para `openclaw sessions` (lista de sesiones almacenadas + uso)
title: Sesiones
x-i18n:
    generated_at: "2026-07-26T04:34:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e00d846229dfad1ada1a8c9a548e26f26247d3f7e5a35106903f6cd4818878b5
    source_path: cli/sessions.md
    workflow: 16
---

# `openclaw sessions`

Lista las sesiones de conversación almacenadas.

Las listas de sesiones no comprueban la disponibilidad de canales o proveedores. Muestran filas de conversaciones persistentes de los almacenes de sesiones. Un canal de Discord, Slack, Telegram u otro canal sin actividad puede volver a conectarse correctamente sin crear una nueva fila de sesión hasta que se procese un mensaje. Usa `openclaw channels status --probe`, `openclaw status --deep` o `openclaw health --verbose` cuando necesites conectividad del canal en tiempo real.

```bash
openclaw sessions
openclaw sessions --agent work
openclaw sessions --all-agents
openclaw sessions --active 120
openclaw sessions --limit 25
openclaw sessions --store ./tmp/sessions.json
openclaw sessions --json
```

Opciones:

| Opción                 | Descripción                                                            |
| -------------------- | ---------------------------------------------------------------------- |
| `--agent <id>`       | Un almacén de agente configurado (valor predeterminado: el agente predeterminado configurado).        |
| `--all-agents`       | Agrega todos los almacenes de agentes configurados.                                 |
| `--store <path>`     | Ruta explícita del almacén (no se puede combinar con `--agent` ni `--all-agents`). |
| `--active <minutes>` | Muestra solo las sesiones actualizadas durante los últimos N minutos.                  |
| `--limit <n\|all>`   | Número máximo de filas que se mostrarán (valor predeterminado: `100`; `all` restaura la salida completa).        |
| `--json`             | Salida legible por máquina.                                               |
| `--verbose`          | Registro detallado.                                                       |

`openclaw sessions` y la RPC `sessions.list` del Gateway están limitadas de forma predeterminada para que los almacenes grandes y de larga duración no puedan monopolizar el proceso de la CLI ni el bucle de eventos del Gateway. La CLI devuelve de forma predeterminada las 100 sesiones más recientes; especifica `--limit <n>` para obtener una ventana más pequeña o más grande, o `--limit all` cuando necesites intencionadamente el almacén completo. Las respuestas JSON incluyen `totalCount`, `limitApplied` y `hasMore` cuando los clientes necesitan mostrar que existen más filas.

Los clientes RPC pueden especificar `configuredAgentsOnly: true` para conservar la fuente de detección combinada amplia, pero devolver solo las filas de los agentes presentes actualmente en la configuración. Control UI usa este modo de forma predeterminada para que los almacenes de agentes eliminados o presentes únicamente en disco no vuelvan a aparecer en la vista Sesiones.

`--all-agents` lee los almacenes de agentes configurados. La detección de sesiones del Gateway y ACP es más amplia: también incluye los almacenes SQLite resueltos desde las raíces de agentes configuradas o desde una raíz `session.store` basada en plantilla. Las rutas de selectores heredadas deben resolverse dentro de la raíz del agente; se omiten los enlaces simbólicos y las rutas externas a la raíz.

`openclaw sessions --all-agents --json`:

```json
{
  "path": null,
  "stores": [
    { "agentId": "main", "path": "/home/user/.openclaw/agents/main/sessions/sessions.json" },
    { "agentId": "work", "path": "/home/user/.openclaw/agents/work/sessions/sessions.json" }
  ],
  "allAgents": true,
  "count": 2,
  "totalCount": 2,
  "limitApplied": 100,
  "hasMore": false,
  "activeMinutes": null,
  "sessions": [
    { "agentId": "main", "key": "agent:main:main", "model": "openai/gpt-5.6-sol" },
    { "agentId": "work", "key": "agent:work:main", "model": "anthropic/claude-sonnet-4-6" }
  ]
}
```

## Seguir el progreso de la trayectoria

```bash
openclaw sessions tail
openclaw sessions tail --follow
openclaw sessions tail --session-key "agent:main:telegram:direct:123" --tail 25
openclaw sessions --agent work tail --follow
openclaw sessions --all-agents tail --follow
```

`openclaw sessions tail` representa los eventos recientes de la trayectoria de ejecución como líneas de progreso compactas. Sin `--session-key`, primero sigue las sesiones en ejecución y después la última sesión almacenada. `--tail <count>` controla cuántos eventos existentes se imprimen antes del modo de seguimiento; el valor predeterminado es `80`, y `0` comienza en el final actual. `--follow` continúa observando la sesión seleccionada respaldada por SQLite o un archivo de trayectoria heredado explícito.

La vista de progreso es deliberadamente conservadora: no se imprimen el texto del prompt, los argumentos de las herramientas ni los cuerpos de sus resultados. Las llamadas a herramientas muestran el nombre de la herramienta con `{...redacted...}`; los resultados de las herramientas muestran estados como `ok`, `error` o `done`; las líneas de finalización del modelo muestran el proveedor/modelo y el estado final.

## Exportar un paquete de trayectoria

```bash
openclaw sessions export-trajectory --session-key "agent:main:telegram:direct:123" --workspace .
openclaw sessions export-trajectory --session-key "agent:main:telegram:direct:123" --output bug-123 --json
```

Esta es la ruta de comando que utiliza el comando de barra `/export-trajectory` después de que el propietario apruebe la solicitud de ejecución. El directorio de salida siempre se resuelve dentro de `.openclaw/trajectory-exports/`, bajo el espacio de trabajo seleccionado.

## Mantenimiento de limpieza

Ejecuta el mantenimiento ahora en lugar de esperar al siguiente ciclo de escritura:

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --agent work --dry-run
openclaw sessions cleanup --all-agents --dry-run
openclaw sessions cleanup --enforce
openclaw sessions cleanup --enforce --active-key "agent:main:telegram:direct:123"
openclaw sessions cleanup --dry-run --fix-dm-scope
openclaw sessions cleanup --json
```

`openclaw sessions cleanup` utiliza la configuración de `session.maintenance` definida en la configuración ([Referencia de configuración](/es/gateway/config-agents#session)):

- Nota sobre el alcance: `openclaw sessions cleanup` mantiene los almacenes de sesiones, las transcripciones, las filas de trayectoria y los archivos auxiliares de trayectoria heredados. No elimina el historial de ejecuciones de Cron, que conserva automáticamente las 2000 filas más recientes por trabajo ([Configuración de Cron](/es/automation/cron-jobs#configuration)).
- La limpieza también elimina los artefactos de transcripción heredados o archivados sin referencias, los puntos de control de Compaction y los archivos auxiliares de trayectoria con una antigüedad superior a `session.maintenance.pruneAfter`; se conservan los artefactos que todavía estén referenciados por filas de sesión de SQLite.
- La limpieza informa por separado de la eliminación de sondeos de ejecución de modelos de corta duración del Gateway como `modelRunPruned`. Esto solo coincide con claves explícitas estrictas con una estructura como `agent:*:explicit:model-run-<uuid>`. La retención es de `24h` fijos y está condicionada por la presión: solo elimina las filas de sondeo obsoletas cuando se alcanza la presión de mantenimiento o del límite de entradas de sesión. Cuando se ejecuta, la limpieza de ejecuciones de modelos ocurre antes de la limpieza global de elementos obsoletos y de la aplicación de límites.

Opciones:

| Opción                 | Descripción                                                                                                                                                                                                                                                                                                |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--dry-run`          | Muestra una vista previa de cuántas entradas se eliminarían o limitarían sin escribir. En modo de texto, imprime una tabla de acciones por sesión (`Action`, `Key`, `Age`, `Model`, `Flags`) y un resumen agrupado por etiqueta de sesión.                                                                                                       |
| `--enforce`          | Aplica el mantenimiento incluso cuando `session.maintenance.mode` es `warn`.                                                                                                                                                                                                                                          |
| `--fix-missing`      | Elimina las entradas heredadas cuyos artefactos de transcripción archivados falten o solo contengan el encabezado o estén vacíos, aunque normalmente todavía no fueran a excluirse por antigüedad o cantidad.                                                                                                                                                             |
| `--fix-dm-scope`     | Cuando `session.dmScope` es `main`, retira las filas obsoletas de mensajes directos con claves de interlocutor que dejaron rutas anteriores de `per-peer`, `per-channel-peer` o `per-account-channel-peer`. Usa primero `--dry-run`; al aplicar la operación, se eliminan esas filas de SQLite y se conservan sus artefactos de transcripción heredados como archivos eliminados. |
| `--active-key <key>` | Protege una clave activa específica frente a la expulsión por presupuesto de disco. Los punteros persistentes de conversaciones externas, como las sesiones de grupo y las sesiones de chat asociadas a hilos, también se conservan durante el mantenimiento por antigüedad, cantidad y presupuesto de disco.                                                                                               |
| `--agent <id>`       | Ejecuta la limpieza de un almacén de agente configurado.                                                                                                                                                                                                                                                                |
| `--all-agents`       | Ejecuta la limpieza de todos los almacenes de agentes configurados.                                                                                                                                                                                                                                                               |
| `--store <path>`     | Ejecuta la operación en una ruta específica de selector de almacén heredado.                                                                                                                                                                                                                                                         |
| `--json`             | Imprime un resumen JSON. Con `--all-agents`, la salida incluye un resumen por almacén.                                                                                                                                                                                                                          |

Cuando se puede acceder a un Gateway, la limpieza sin simulación de los almacenes de agentes configurados se envía a través del Gateway para que comparta el mismo proceso de escritura del almacén de sesiones que el tráfico de ejecución. Usa `--store <path>` para reparar explícitamente sin conexión un selector de almacén heredado.

`openclaw sessions cleanup --all-agents --dry-run --json`:

```json
{
  "allAgents": true,
  "mode": "warn",
  "dryRun": true,
  "stores": [
    {
      "agentId": "main",
      "storePath": "/home/user/.openclaw/agents/main/sessions/sessions.json",
      "beforeCount": 120,
      "afterCount": 80,
      "missing": 0,
      "dmScopeRetired": 0,
      "pruned": 40,
      "capped": 0
    },
    {
      "agentId": "work",
      "storePath": "/home/user/.openclaw/agents/work/sessions/sessions.json",
      "beforeCount": 18,
      "afterCount": 18,
      "missing": 0,
      "dmScopeRetired": 0,
      "pruned": 0,
      "capped": 0
    }
  ]
}
```

## Compactar una sesión

Recupera presupuesto de contexto para una sesión bloqueada o sobredimensionada. `openclaw sessions
compact <key>` es el contenedor de primera clase de la RPC `sessions.compact` del Gateway y requiere que haya un Gateway en ejecución.

```bash
openclaw sessions compact "agent:main:main"
openclaw sessions compact "agent:main:main" --max-lines 200
openclaw sessions compact "agent:work:main" --agent work --json
```

- Sin `--max-lines`, el Gateway resume la transcripción mediante un LLM. La CLI no impone de forma predeterminada un plazo límite del cliente; el Gateway controla el ciclo de vida configurado de Compaction.
- Con `--max-lines <n>`, trunca la transcripción a sus últimas `n` líneas y archiva la transcripción anterior como un archivo auxiliar `.bak`.
- `--agent <id>`: agente propietario de la sesión; obligatorio para las claves `global`.
- `--url` / `--token` / `--password`: anulaciones de la conexión del Gateway.
- `--timeout <ms>`: tiempo de espera opcional de la RPC en el cliente, en milisegundos.
- `--json`: imprime la carga útil RPC sin procesar.

El comando finaliza con un código distinto de cero cuando el Gateway informa de una Compaction fallida o está
inaccesible, de modo que los crons y scripts nunca confundan una operación nula silenciosa con un resultado satisfactorio.

<Note>
`openclaw agent --message '/compact ...'` **no** es una ruta de Compaction. Los comandos con barra
de la CLI son rechazados por la comprobación del remitente autorizado; esa
invocación finaliza con un código distinto de cero y proporciona indicaciones que remiten aquí, en lugar de
no realizar silenciosamente ninguna operación.
</Note>

### RPC sessions.compact

`openclaw gateway call sessions.compact --params '<json>'` acepta:

| Campo      | Tipo        | Obligatorio | Descripción                                                |
| ---------- | ----------- | ----------- | ---------------------------------------------------------- |
| `key`      | string      | sí          | Clave de sesión que se va a compactar (por ejemplo, `agent:main:main`).    |
| `agentId`  | string      | no          | Id. del agente propietario de la sesión (para claves `global`).        |
| `maxLines` | integer ≥ 1 | no          | Trunca a las últimas N líneas en lugar de usar el resumen mediante LLM. |

Ejemplo de respuesta de resumen mediante LLM:

```json
{
  "ok": true,
  "key": "agent:main:main",
  "compacted": true,
  "result": { "tokensBefore": 243868, "tokensAfter": 34941 }
}
```

Ejemplo de respuesta de truncamiento (`--max-lines 200`):

```json
{
  "ok": true,
  "key": "agent:main:main",
  "compacted": true,
  "archived": "/home/user/.openclaw/agents/main/sessions/transcripts/<id>.jsonl.bak",
  "kept": 200
}
```

## Relacionado

- [Configuración de sesiones](/es/gateway/config-agents#session)
- [Gestión de sesiones](/es/concepts/session)
- [Compaction](/es/concepts/compaction)
- [Referencia de la CLI](/es/cli)
