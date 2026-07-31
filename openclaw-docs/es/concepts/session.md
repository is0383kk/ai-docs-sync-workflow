---
read_when:
    - Quieres comprender el enrutamiento y el aislamiento de las sesiones
    - Se desea configurar el ámbito de los mensajes directos para entornos multiusuario
    - Está depurando los restablecimientos diarios o por inactividad de las sesiones
summary: Cómo gestiona OpenClaw las sesiones de conversación
title: Gestión de sesiones
x-i18n:
    generated_at: "2026-07-26T05:11:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de85fe5a623bdbc6d5564d822b39e9077a582b0816b62ab30d2f7245bd097000
    source_path: concepts/session.md
    workflow: 16
---

OpenClaw dirige cada mensaje entrante a una **sesión** según su procedencia:
mensajes directos, chats grupales, trabajos de Cron, etc. Todo el estado de las sesiones
pertenece al **Gateway**; los clientes de interfaz consultan al Gateway para obtener los datos de las sesiones.

Para la configuración predeterminada del agente personal —una conversación continua compartida por todos los
canales de mensajes directos, a la que se incorporan la actividad de los grupos y el trabajo en segundo plano—, consulte
[La sesión principal](/es/concepts/main-session).

## Cómo se dirigen los mensajes

| Origen             | Comportamiento                         |
| ------------------ | -------------------------------------- |
| Mensajes directos  | Sesión compartida de forma predeterminada |
| Chats grupales     | Aislada por grupo                      |
| Salas/canales      | Aislada por sala                       |
| Trabajos de Cron   | Sesión nueva por ejecución             |
| Webhooks           | Aislada por Webhook                    |

## Aislamiento de mensajes directos

De forma predeterminada, todos los mensajes directos comparten una sesión para mantener la continuidad, lo cual es adecuado para
configuraciones de un solo usuario.

<Warning>
Si varias personas pueden enviar mensajes al agente, active el aislamiento de mensajes directos. Sin él, todos los
usuarios comparten el mismo contexto de conversación, por lo que los mensajes privados de Alice serían
visibles para Bob.
</Warning>

```json5
{
  session: {
    dmScope: "per-channel-peer", // aislar por canal + remitente
  },
}
```

Opciones de `session.dmScope`:

| Valor                      | Comportamiento                                                 |
| -------------------------- | -------------------------------------------------------------- |
| `main` (predeterminado)           | Todos los mensajes directos comparten la [sesión principal](/es/concepts/main-session) |
| `per-peer`                 | Aislar por remitente entre distintos canales                   |
| `per-channel-peer`         | Aislar por canal + remitente (recomendado)                      |
| `per-account-channel-peer` | Aislar por cuenta + canal + remitente                           |

<Tip>
Si la misma persona establece contacto desde varios canales, use
`session.identityLinks` para asignar sus identidades a un único identificador canónico de par, de modo que
compartan una sesión.
</Tip>

### Acoplar canales vinculados

Los comandos de acoplamiento trasladan la ruta de respuesta de la sesión actual del chat directo a otro
canal vinculado sin iniciar una sesión nueva. Consulte
[Acoplamiento de canales](/es/concepts/channel-docking) para ver ejemplos, configuración y
solución de problemas.

Verifique la configuración con `openclaw security audit`.

## Sesiones de incógnito

Las sesiones de incógnito solo están disponibles en la pantalla **Nuevo hilo** de la interfaz de control. Active **Incógnito** antes de iniciar el hilo para mantener su entrada de sesión, transcripción y estado de Compaction en la memoria del proceso en lugar de almacenarlos en disco. El hilo desaparece cuando se reinicia el Gateway, no ejecuta el vaciado automático de memoria de OpenClaw y no crea un archivo de transcripción al restablecerlo o eliminarlo. Las ejecuciones respaldadas por Codex también inician el hilo de su entorno en modo efímero, por lo que Codex no escribe archivos de despliegue ni de estado de sesión local; otros proveedores de modelos utilizan API HTTP y no mantienen ninguna transcripción local del proveedor en OpenClaw.

El segmento `incognito-` está reservado para las claves de sesión del panel, los subagentes y las claves internas ocultas; `openclaw doctor --fix` cambia el nombre de cualquier clave duradera heredada que entre en conflicto.

El modo de incógnito no restringe las herramientas normales del agente. Una solicitud explícita de guardar información o cualquier escritura de archivos mediante herramientas aún puede conservar datos fuera del almacén de la sesión de incógnito. El proveedor de modelos configurado sigue procesando los mensajes enviados, el registro de diagnóstico no cambia y OpenClaw continúa registrando metadatos de auditoría sin contenido, como referencias HMAC.

En los Gateways multiusuario, los hilos de incógnito solo son visibles para conexiones con ámbito de administrador y nunca aparecen mediante las herramientas de sesión del agente ni la búsqueda de transcripciones de otra sesión. Esto los protege del almacenamiento y de otros usuarios mediados por el Gateway, pero no del propietario del Gateway ni del operador del proceso, que siempre pueden observar las sesiones activas.

## Recordar entre conversaciones

Las transcripciones separadas controlan el historial local de cada conversación. Para un agente personal
o de plena confianza, `memory.search.rememberAcrossConversations: true`
añade un paso opcional de recuperación entre las demás conversaciones privadas
de ese agente; no combina sus transcripciones.

Las conversaciones directas privadas y las conversaciones explícitas persistentes de la interfaz pueden proporcionarse
contexto relevante entre sí. Los grupos y canales permanecen separados en ambas direcciones:
sus transcripciones no son fuentes privadas de recuperación y las respuestas de esas
conversaciones no reciben contexto de transcripciones privadas. La conversación actual
también se excluye porque su historial ya está cargado.

Esta opción no cambia las claves de sesión, el ámbito de los mensajes directos, el enrutamiento, la entrega ni
`tools.sessions.visibility`. La memoria compartida del espacio de trabajo en `MEMORY.md` y
`memory/*.md` también conserva su comportamiento actual. El proveedor de memoria actual
debe admitir la recuperación protegida de transcripciones privadas; los motores de contexto como
Lossless Claw siguen siendo independientes y pueden ejecutarse junto con ella. Consulte
[Active Memory](/es/concepts/active-memory#remember-across-conversations) para obtener información sobre la configuración
y el funcionamiento.

## Ciclo de vida de las sesiones

Las sesiones se reutilizan hasta que se restablecen manualmente o se activa una política de restablecimiento automático:

- **Sin restablecimiento automático** (valor predeterminado de `mode: "none"`) - las sesiones mantienen el mismo
  `sessionId`; Compaction gestiona el contexto activo a medida que crece la conversación.
- **Restablecimiento diario** (`mode: "daily"`) - permite iniciar una sesión nueva a una hora local
  configurada (`session.reset.atHour`, valor predeterminado `4`, 0-23) en el host del Gateway. La
  vigencia diaria se basa en el momento en que comenzó el `sessionId` actual, no en escrituras
  posteriores de metadatos.
- **Restablecimiento por inactividad** (`mode: "idle"`) - permite iniciar una sesión nueva después de `session.reset.idleMinutes`
  de inactividad. La vigencia por inactividad se basa en la última interacción real del usuario o canal,
  por lo que los eventos del sistema de Heartbeat, Cron y ejecución no mantienen
  activa la sesión.
- **Restablecimiento manual** - escriba `/new` o `/reset` en el chat. `/new <model>` también
  cambia el modelo.

Cuando se configuran tanto el restablecimiento diario como el restablecimiento por inactividad, se aplica el que venza primero.
Los turnos de Heartbeat, Cron, ejecución y otros eventos del sistema pueden escribir metadatos de sesión,
pero esas escrituras no prolongan la vigencia del restablecimiento diario ni por inactividad. Cuando un restablecimiento
renueva la sesión, se descartan los avisos de eventos del sistema en cola correspondientes a la sesión anterior
para que las actualizaciones obsoletas en segundo plano no se antepongan al primer prompt de
la sesión nueva.

Las sesiones con una sesión de CLI activa perteneciente al proveedor siguen el mismo comportamiento predeterminado
sin restablecimiento automático. Use `/reset` o configure `session.reset` explícitamente cuando esas sesiones
deban caducar mediante un temporizador.

Active los restablecimientos automáticos globalmente y luego sobrescríbalos por tipo de chat o canal:

```json5
{
  session: {
    reset: { mode: "daily", atHour: 4 },
    resetByType: {
      group: { mode: "idle", idleMinutes: 120 },
      thread: { mode: "daily", atHour: 6 },
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 10080 },
    },
  },
}
```

`resetByType` admite `direct`, `group` y `thread`. Doctor migra las entradas heredadas `dm` a `direct` y `session.idleMinutes` a `session.reset.idleMinutes`; el esquema rechaza ambas formas retiradas.

## Dónde reside el estado

- **Filas de sesiones en tiempo de ejecución:** `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- **Archivos de transcripciones archivadas:** `~/.openclaw/agents/<agentId>/sessions/`
- **Origen de migración de filas heredadas:** `~/.openclaw/agents/<agentId>/sessions/sessions.json`

Las filas de sesiones de la base de datos SQLite de cada agente mantienen marcas de tiempo
separadas para el ciclo de vida:

- `sessionStartedAt`: momento en que comenzó el `sessionId` actual; el restablecimiento diario usa este valor.
- `lastInteractionAt`: última interacción de usuario o canal que prolonga la duración por inactividad.
- `updatedAt`: última modificación de la fila del almacén; resulta útil para enumerar y depurar, pero no es
  la fuente autoritativa para determinar la vigencia de los restablecimientos diarios o por inactividad.

Durante la migración desde instalaciones anteriores, el inicio del Gateway y `openclaw doctor
--fix` importan automáticamente a SQLite las filas heredadas `sessions.json` y el historial activo de transcripciones JSONL. Las filas sin `sessionStartedAt` se resuelven a partir del encabezado de sesión de la transcripción JSONL heredada cuando está disponible. Si una fila anterior tampoco contiene `lastInteractionAt`, la vigencia por inactividad recurre a la hora de inicio de esa sesión, no a escrituras administrativas posteriores. Use `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` y la [secuencia de migración de Doctor](/es/cli/doctor#session-sqlite-migration) cuando se requieran pruebas explícitas de inspección o validación.

## Mantenimiento de sesiones

OpenClaw limita el almacenamiento de sesiones a lo largo del tiempo mediante `session.maintenance`; se muestran los valores
predeterminados:

```json5
{
  session: {
    maintenance: {
      mode: "enforce", // "enforce" aplica la limpieza; "warn" solo informa
      pruneAfter: "30d",
      maxEntries: 500,
    },
  },
}
```

Para límites `maxEntries` de tamaño de producción, las escrituras del entorno del Gateway utilizan un pequeño
búfer de nivel máximo y limpian por lotes hasta volver al límite configurado.
Las lecturas del almacén de sesiones no depuran ni limitan las entradas durante el inicio del Gateway, por lo que
el inicio y las sesiones aisladas de Cron no incurren en una limpieza completa del almacén.
`openclaw sessions cleanup --enforce` aplica el límite inmediatamente.

Las sesiones de sondeo de ejecuciones de modelos del Gateway tienen una duración corta de forma predeterminada. Las filas que coinciden con
`agent:*:explicit:model-run-<uuid>` utilizan una retención fija de `24h`, pero la limpieza
está condicionada por la presión: solo elimina las filas de sondeo obsoletas cuando se alcanza la presión
del mantenimiento o el límite de entradas de sesión, y se ejecuta antes del límite general por antigüedad
de entradas obsoletas y del límite de entradas. Las sesiones normales directas, grupales, de hilo, Cron, enlace, Heartbeat,
ACP y subagente no heredan esta retención de 24h.

El mantenimiento conserva los punteros duraderos a conversaciones externas, incluidas las sesiones
grupales y las sesiones de chat limitadas a un hilo, a la vez que permite que las entradas sintéticas de Cron,
enlace, Heartbeat, ACP y subagente caduquen con el tiempo.

Las sesiones archivadas son guardadas por el usuario y están exentas de todas las rutas de mantenimiento
automático, incluidas la depuración por antigüedad, los límites de entradas, la limpieza de ejecuciones de modelos y el desalojo
por presupuesto de disco. Permanecen archivadas hasta que se desarchivan o se eliminan
explícitamente.

Si anteriormente se utilizó el aislamiento de mensajes directos y después se devolvió `session.dmScope` a
`main`, previsualice las filas obsoletas de mensajes directos con claves por par mediante
`openclaw sessions cleanup --dry-run --fix-dm-scope`. Al aplicar la misma opción,
se retiran esas filas antiguas de mensajes directos y se conservan sus transcripciones como archivos
eliminados.

Previsualice cualquier ejecución de mantenimiento con `openclaw sessions cleanup --dry-run`.

## Inspección de sesiones

| Comando                    | Muestra                                           |
| -------------------------- | ------------------------------------------------- |
| `openclaw status`          | Ruta del almacén de sesiones y actividad reciente |
| `openclaw sessions --json` | Todas las sesiones (filtrar con `--active <minutes>`) |
| `/status` en el chat          | Uso del contexto, modelo y opciones               |
| `/context list`            | Contenido del prompt del sistema                   |

## Lecturas adicionales

- [Búsqueda de sesiones](/es/concepts/session-search) - recuperación de texto completo en transcripciones anteriores
- [Depuración de sesiones](/es/concepts/session-pruning) - recorte de resultados de herramientas
- [Compaction](/es/concepts/compaction) - resumen de conversaciones largas
- [Herramientas de sesión](/es/concepts/session-tool) - herramientas del agente para trabajar entre sesiones
- [Análisis detallado de la gestión de sesiones](/es/reference/session-management-compaction) -
  esquema del almacén, transcripciones, política de envío, metadatos de origen y configuración avanzada
- [Multiagente](/es/concepts/multi-agent) - enrutamiento y aislamiento de sesiones entre agentes
- [Tareas en segundo plano](/es/automation/tasks) - cómo el trabajo desacoplado crea registros de tareas con referencias de sesión
- [Enrutamiento de canales](/es/channels/channel-routing) - cómo se dirigen los mensajes entrantes a las sesiones

## Relacionado

- [Depuración de sesiones](/es/concepts/session-pruning)
- [Herramientas de sesión](/es/concepts/session-tool)
- [Cola de comandos](/es/concepts/queue)
