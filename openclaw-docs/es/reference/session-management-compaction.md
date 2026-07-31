---
read_when:
    - Necesita depurar los identificadores de sesión, los eventos de transcripción o los campos de las filas de sesión
    - Se está cambiando el comportamiento de la compactación automática o añadiendo tareas de mantenimiento «previas a la compactación»
    - Se quieren implementar vaciados de memoria o turnos silenciosos del sistema
summary: 'Análisis en profundidad: almacenamiento de sesiones y transcripciones, ciclo de vida y funcionamiento interno de la Compaction (automática)'
title: Análisis detallado de la gestión de sesiones
x-i18n:
    generated_at: "2026-07-26T05:55:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ae02d49245768831abd17e1c2e5adacfa1a36673cef2a8a7a06a5300392b104
    source_path: reference/session-management-compaction.md
    workflow: 16
---

Un único **proceso del Gateway** controla el estado de la sesión de principio a fin. Las interfaces de usuario (aplicación de macOS, interfaz web de Control, TUI) consultan al Gateway las listas de sesiones y los recuentos de tokens. En modo remoto, los archivos de sesión se encuentran en el host remoto, por lo que revisar los archivos del Mac local no reflejará lo que utiliza el Gateway.

Primero, la documentación general: [Gestión de sesiones](/es/concepts/session), [Compaction](/es/concepts/compaction), [Descripción general de la memoria](/es/concepts/memory), [Búsqueda en la memoria](/es/concepts/memory-search), [Depuración de sesiones](/es/concepts/session-pruning), [Higiene de transcripciones](/es/reference/transcript-hygiene), referencia completa de configuración en [Configuración del agente](/es/gateway/config-agents).

## Dos capas de persistencia

1. **Filas de sesión (SQLite por agente)** - mapa de clave/valor `sessionKey -> SessionEntry`. Estado mutable en tiempo de ejecución controlado por el Gateway. Registra metadatos: id. de la sesión actual, última actividad, opciones, contadores de tokens.
2. **Eventos de transcripción (SQLite por agente)** - de solo anexión, estructurados en árbol (las entradas tienen `id` + `parentId`). Almacena la conversación, las llamadas a herramientas y los resúmenes de Compaction; reconstruye el contexto del modelo para turnos futuros. Los puntos de control de Compaction son metadatos sobre la transcripción sucesora compactada; una nueva Compaction no escribe una segunda copia de `.checkpoint.*.jsonl`.

Las instalaciones antiguas aún pueden tener archivos `sessions.json` en el directorio `sessions/`
del agente. Estos archivos deben tratarse como entradas de migración de filas de sesión heredadas u objetivos explícitos
de mantenimiento sin conexión. El inicio del Gateway y `openclaw doctor --fix` importan
automáticamente las filas heredadas activas y el historial de transcripciones en el almacén SQLite por agente.
Ejecute `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` y, a continuación, siga la [secuencia de migración de
Doctor](/es/cli/doctor#session-sqlite-migration) cuando necesite pruebas explícitas
de inspección o validación. Si una migración falla después de archivar los artefactos
de transcripción heredados, utilice el modo de recuperación de Doctor de esa secuencia.
La recuperación utiliza manifiestos de migración, restaura únicamente los artefactos
de soporte archivados afectados, prepara un informe saneado de incidencia de GitHub cuando se solicita y no
hace que el entorno de ejecución activo vuelva a leer archivos JSONL.

Los lectores del historial del Gateway evitan materializar toda la transcripción, salvo que la superficie necesite acceso arbitrario al historial. El historial de la primera página, el historial de chat integrado, la recuperación tras un reinicio y las comprobaciones de tokens/uso emplean lecturas limitadas del final desde SQLite. Los análisis completos de transcripciones pasan por el índice asíncrono de transcripciones y se comparten entre lectores simultáneos.

## Ubicaciones en disco

Por agente, en el host del Gateway (resueltas mediante `src/config/sessions.ts`):

- Almacén de filas de sesión en tiempo de ejecución: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- Filas de transcripción en tiempo de ejecución: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- Artefactos de transcripción heredados/archivados: `~/.openclaw/agents/<agentId>/sessions/`
- Entrada de migración de filas heredadas: `~/.openclaw/agents/<agentId>/sessions/sessions.json`

## Mantenimiento del almacén y controles de disco

`session.maintenance` controla el mantenimiento automático de las filas de sesión de SQLite, las filas de transcripción de SQLite, los artefactos archivados y los archivos auxiliares de trayectorias:

| Clave                   | Valor predeterminado   | Notas                                                                                       |
| ----------------------- | ---------------------- | ------------------------------------------------------------------------------------------- |
| `mode`                  | `"enforce"`           | o `"warn"` (solo informa, no modifica)                                                      |
| `pruneAfter`            | `"30d"`               | umbral de antigüedad de las entradas obsoletas                                                                      |
| `maxEntries`            | `500`                 | límite de entradas de sesión                                                                      |
| `resetArchiveRetention` | conservar (sin umbral de antigüedad)  | umbral de antigüedad de los archivos de transcripción `*.reset.*`/`*.deleted.*`; una duración habilita la eliminación |
| `maxDiskBytes`          | `10gb`                | presupuesto de disco por agente para las sesiones; `false` lo desactiva                                            |
| `highWaterBytes`        | 80 % de `maxDiskBytes` | objetivo tras la limpieza por presupuesto                                                                 |

El restablecimiento hace avanzar la asignación activa de `sessionKey -> sessionId`, pero conserva las filas anteriores de sesión, transcripción, trayectoria y búsqueda de SQLite. Ese historial sigue siendo consultable con la misma clave de sesión; las listas normales de entradas y sesiones solo muestran la nueva asignación activa. El historial de restablecimientos conservado está limitado por el presupuesto de disco, no por `resetArchiveRetention`, que solo aplica antigüedad a los artefactos archivados. La eliminación explícita es diferente: escribe y verifica un archivo comprimido de la transcripción (`*.jsonl.deleted.<timestamp>.zst` cuando zstd está disponible) antes de eliminar las filas de la sesión eliminada.

La aplicación de `maxDiskBytes` utiliza bytes físicos: el archivo principal de SQLite por agente, su archivo `-wal` y los archivos contabilizados en el directorio de sesiones del agente. Nunca calcula tamaños estimados del JSON de las filas ni resta tamaños lógicos de filas de ese total.

Las sesiones de sondeo de ejecuciones de modelos del Gateway (claves que coinciden con `agent:*:explicit:model-run-<uuid>`) tienen una retención fija e independiente de `24h`. Esta depuración está condicionada por la presión: solo se ejecuta cuando se alcanza la presión del mantenimiento/límite de entradas de sesión y únicamente antes del paso global de limpieza/límite de entradas obsoletas. Las demás sesiones explícitas no utilizan esta retención.

Cuando el uso físico combinado supera `maxDiskBytes`, `mode: "enforce"` recupera primero el espacio de base de datos que puede liberarse mediante puntos de control y, después, elimina los archivos de restablecimiento/eliminación conservados más antiguos. Si el uso sigue por encima de `highWaterBytes`, recorre las sesiones históricas de SQLite por `sessions.updated_at`, comenzando por la más antigua. Una sesión es histórica si su id. no está referenciado por una entrada de sesión activa, un destino de ruta o una ejecución admitida/en curso. Para cada víctima, la limpieza escribe, sincroniza con fsync y vuelve a leer el archivo comprimido antes de que una transacción de escritura elimine la fila de sesión y sus proyecciones de transcripción, trayectoria, actividad, índice y FTS. Esto incluye sesiones que contienen eventos de trayectoria, pero no eventos de transcripción. La limpieza vuelve a comprobar las referencias de rutas, entradas y admisiones en el momento de la eliminación, vuelve a medir el uso físico después de cada archivo o sesión víctima y se detiene en `highWaterBytes`.

Las escrituras confirmadas y las eliminaciones llegan primero al WAL. La limpieza crea un punto de control para que el WAL pueda reducirse inmediatamente y, después, utiliza una limpieza incremental para devolver las páginas finales libres aptas del archivo principal; las páginas que aún no puedan recuperarse permanecen en el archivo principal y, por tanto, siguen contabilizándose en la siguiente medición física. `mode: "warn"` informa del exceso físico actual sin crear puntos de control, escribir un archivo ni eliminar filas.

Ejecute el mantenimiento bajo demanda:

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

El mantenimiento conserva punteros duraderos a conversaciones externas, como sesiones de grupo y sesiones de chat limitadas a un hilo, pero las entradas sintéticas del entorno de ejecución (Cron, enlaces, Heartbeat, ACP, subagentes) pueden eliminarse cuando superan la antigüedad, el recuento o el presupuesto de disco configurados. Las ejecuciones aisladas de Cron utilizan un control `cron.sessionRetention` independiente, separado de la retención de sondeos de ejecuciones de modelos.

Las escrituras normales del Gateway pasan por el descriptor de acceso a sesiones, que serializa las mutaciones de SQLite por agente mediante la ruta de escritura del entorno de ejecución. El código del entorno de ejecución debe utilizar preferentemente las funciones auxiliares del descriptor de acceso en `src/config/sessions/session-accessor.ts`; las funciones auxiliares heredadas de `sessions.json` son herramientas de migración y mantenimiento sin conexión. Cuando se puede acceder a un Gateway, `openclaw sessions cleanup` y `openclaw agents delete` sin simulación delegan las mutaciones del almacén al Gateway para que la limpieza se incorpore a la misma cola de escritura; `--store <path>` es la ruta explícita de reparación sin conexión para un almacén heredado seleccionado y siempre permanece local (al igual que `--dry-run`). La limpieza de `maxEntries` se realiza por lotes para almacenes de tamaño de producción, por lo que un almacén puede superar brevemente el límite configurado antes de que la siguiente limpieza de nivel máximo vuelva a reducirlo. Las lecturas nunca depuran ni limitan entradas durante el inicio del Gateway: solo lo hacen las escrituras o `openclaw sessions cleanup --enforce`; este último también aplica el límite inmediatamente y depura artefactos antiguos no referenciados de transcripciones, puntos de control y trayectorias heredados, incluso sin un presupuesto de disco configurado.

OpenClaw ya no crea copias de seguridad automáticas de rotación `sessions.json.bak.*` durante las escrituras del Gateway. El esquema actual rechaza la clave heredada `session.maintenance.rotateBytes`, y `openclaw doctor --fix` la elimina de las configuraciones antiguas.

Las mutaciones de transcripciones utilizan la cola de escritura de sesiones para el destino de transcripciones de SQLite:

Los bloqueos de escritura de sesiones utilizan valores predeterminados fijos de producción. Las variables de entorno
`OPENCLAW_SESSION_WRITE_LOCK_*` correspondientes siguen disponibles para
diagnósticos a nivel de proceso y anulaciones de emergencia.

### Cambio a una versión anterior tras la transición a SQLite

Restaure los artefactos archivados de transcripciones heredadas antes de ejecutar una versión anterior
de OpenClaw basada en archivos:

```bash
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

La migración conserva los archivos heredados `sessions.json` para soporte y
reversión, pero los archivos JSONL de transcripciones activas importados en SQLite se
renombran como `session-sqlite-import-archive/`. Los entornos de ejecución anteriores basados en archivos siguen
las rutas `sessionFile` de `sessions.json`, por lo que necesitan que esos artefactos se restauren
antes del inicio. La restauración utiliza manifiestos de migración, mueve únicamente los artefactos
archivados registrados cuyas rutas originales no existen y conserva la base de datos SQLite
para una recuperación posterior.

Las sesiones creadas después de la transición a SQLite solo existen en SQLite y no aparecerán en un
entorno de ejecución anterior basado en archivos. Si vuelve a actualizar después de cambiar a una versión anterior, ejecute de nuevo la secuencia de
inspección y validación de Doctor para que OpenClaw pueda verificar los artefactos heredados
restaurados antes de importarlos.

## Sesiones de Cron y registros de ejecución

Las ejecuciones aisladas de Cron crean sus propias entradas/transcripciones de sesión con una retención específica:

- `cron.sessionRetention` (valor predeterminado: `"24h"`) depura del almacén las sesiones antiguas de ejecuciones aisladas de Cron; `false` lo desactiva.
- El historial de ejecuciones conserva las 2000 filas terminales más recientes por tarea de Cron. Las filas perdidas mantienen su ventana de limpieza de 24 horas.

Cuando Cron fuerza la creación de una nueva sesión de ejecución aislada, sanea la entrada de sesión `cron:<jobId>` anterior antes de escribir la nueva fila: conserva las preferencias seguras (ajustes de pensamiento/rapidez/detalle/razonamiento, etiquetas y nombre para mostrar) y las anulaciones de modelo/autenticación seleccionadas explícitamente por el usuario, pero descarta el contexto ambiental de la conversación (enrutamiento de canales/grupos, política de envío/cola, elevación, origen y enlace con el entorno de ejecución de ACP), de modo que una nueva ejecución aislada no pueda heredar una autoridad obsoleta de entrega o del entorno de ejecución de una ejecución anterior.

## Claves de sesión (`sessionKey`)

Una `sessionKey` identifica el contenedor de conversación en el que se encuentra (enrutamiento + aislamiento). Reglas canónicas: [/concepts/session](/es/concepts/session).

| Patrón                       | Ejemplo                                                     |
| ---------------------------- | ----------------------------------------------------------- |
| Chat principal/directo (por agente) | `agent:<agentId>:<mainKey>` (valor predeterminado: `main`)                |
| Grupo                        | `agent:<agentId>:<channel>:group:<id>`                      |
| Sala/canal (Discord/Slack)   | `agent:<agentId>:<channel>:channel:<id>` o `...:room:<id>` |
| Cron                         | `cron:<job.id>`                                             |
| Webhook                      | `hook:<uuid>` (salvo que se anule)                           |

## Id. de sesión (`sessionId`)

Cada `sessionKey` apunta a un `sessionId` actual (la identidad de la transcripción de SQLite que continúa la conversación). La lógica de decisión se encuentra en `initSessionState()`, en `src/auto-reply/reply/session.ts`.

- **Restablecimiento** (`/new`, `/reset`) crea un nuevo `sessionId` para ese `sessionKey`.
- **Sin restablecimiento automático** es el valor predeterminado. El `sessionId` actual continúa mientras Compaction mantiene acotado el contexto activo del modelo.
- **Restablecimiento diario** (`session.reset.mode: "daily"`) crea un nuevo `sessionId` con el siguiente mensaje después del límite de hora local configurado (`session.reset.atHour`, valor predeterminado `4`).
- **Expiración por inactividad** (`session.reset.mode: "idle"` con `session.reset.idleMinutes`, o el `session.idleMinutes` heredado) crea un nuevo `sessionId` cuando llega un mensaje después del periodo de inactividad. Si se configuran tanto el restablecimiento diario como el de inactividad, prevalece el que expire primero.
- **Reanudación tras la reconexión de la interfaz de control** conserva la sesión visible en ese momento para un envío de reconexión cuando el Gateway recibe el `sessionId` correspondiente de un cliente de interfaz de operador. Esta es una señal de un solo uso; los envíos obsoletos ordinarios siguen creando un nuevo `sessionId`.
- **Eventos del sistema** (Heartbeat, activaciones de Cron, notificaciones de ejecución, mantenimiento interno del Gateway) pueden modificar la fila de la sesión, pero nunca prolongan la vigencia del restablecimiento diario o por inactividad. El cambio de sesión por restablecimiento descarta los avisos de eventos del sistema en cola de la sesión anterior antes de construir el prompt nuevo.
- **Política de bifurcación del elemento principal** usa la rama activa de OpenClaw al crear un hilo o una bifurcación de subagente. Si esa rama es demasiado grande (supera un límite interno fijo, actualmente de 100K tokens), OpenClaw inicia el elemento secundario con contexto aislado en lugar de fallar o heredar un historial inutilizable. El dimensionamiento es automático y no se puede configurar; la configuración heredada `session.parentForkMaxTokens` se elimina mediante `openclaw doctor --fix`.
- **Bifurcaciones del operador**: `sessions.create { parentSessionKey, fork: true }` crea una sesión nueva cuya transcripción se bifurca desde el estado actual del elemento principal (el mismo mecanismo de bifurcación que en la creación de subagentes, incluido el límite de tamaño anterior). La bifurcación se rechaza mientras el elemento principal tiene una ejecución activa, hereda la selección de modelo del elemento principal salvo que se proporcione una explícitamente y marca el `forkedFromParent` secundario con contadores de tokens nuevos.

## Esquema del almacén de sesiones

El almacén de ejecución conserva los valores de `SessionEntry` en una base SQLite por agente. El tipo de valor es `SessionEntry` en `src/config/sessions.ts`. Campos clave (lista no exhaustiva):

- `sessionId`: identificador de la transcripción actual utilizado para direccionar las filas de transcripción de SQLite
- `sessionStartedAt`: marca de tiempo de inicio del `sessionId` actual; se utiliza para determinar la vigencia del restablecimiento diario. Las filas heredadas pueden obtenerla del encabezado de sesión JSONL.
- `lastInteractionAt`: marca de tiempo de la última interacción real del usuario o canal; se utiliza para determinar la vigencia del restablecimiento por inactividad, de modo que los eventos de Heartbeat, Cron y ejecución no mantengan activas las sesiones. Las filas heredadas sin este campo recurren a la hora de inicio de sesión recuperada.
- `updatedAt`: marca de tiempo de la última modificación de la fila del almacén, utilizada para listados, depuración y mantenimiento interno; no es la referencia autorizada para la vigencia diaria o por inactividad.
- `archivedAt`: marca de tiempo de archivado opcional. Las sesiones archivadas permanecen en el almacén con su transcripción intacta y se excluyen de los listados activos normales.
- `pinnedAt`: marca de tiempo de fijación opcional. Las sesiones activas fijadas se ordenan antes que las no fijadas; al archivar una sesión se elimina su fijación.
- Interoperabilidad de hilos de Codex: ambos campos siguen la estructura de gestión de hilos de Codex; los valores booleanos `archived`/`pinned` en la comunicación siempre se derivan de la marca de tiempo y el servidor los asigna, de acuerdo con la semántica `threads.archived_at` de Codex y la serialización camelCase. Las marcas de tiempo de OpenClaw son milisegundos desde la época, mientras que Codex utiliza segundos desde la época, por lo que los puentes realizan la conversión en el punto de integración del Plugin `codex`. Codex aún no tiene una API de fijación (solo `thread/archive`/`thread/unarchive`); el estado fijado permanece del lado de OpenClaw hasta que exista una, momento en el que la estructura coincidente permitirá que las sesiones vinculadas transfieran mecánicamente el estado de fijación en ambos sentidos.
- La supervisión de Codex solo enumera los hilos nativos no archivados. Un hilo `idle` o `notLoaded` local del Gateway y con actividad desconocida solo puede archivarse mediante `thread/archive` nativo después de que el operador confirme explícitamente que ningún otro proceso de Codex es su propietario; primero, el Plugin realiza una lectura nueva del estado local del proceso y, después, el hilo desaparece del catálogo. Esa lectura no puede demostrar que otro proceso de App Server no esté usando el hilo. OpenClaw se niega a archivar filas activas y con errores, y el archivado de nodos emparejados no está disponible hasta que el puente de Node pueda controlar el ciclo de vida completo del hilo transmitido. Al desarchivarlo en un cliente nativo de Codex, el hilo puede volver a aparecer.
- `lastReadAt` / `markedUnreadAt`: marcas de tiempo del estado de lectura asignadas en el servidor mediante `sessions.patch { unread }`; `unread: false` registra una lectura (establece `lastReadAt` y borra `markedUnreadAt`); `unread: true` marca la sesión como no leída hasta la siguiente lectura. Las filas de sesión exponen un valor booleano derivado `unread`: marcado explícitamente como no leído o leído antes de la actividad más reciente. Las sesiones que nunca se hayan marcado como leídas permanecen `unread: false`, por lo que las instalaciones existentes no muestran indicadores nuevos al actualizarse.
- `lastActivityAt`: marca de tiempo de la última ejecución completada del agente que cuenta como actividad que merece marcarse como no leída (ejecuciones del usuario, del canal y de Cron). Los turnos de Heartbeat y eventos internos, así como las modificaciones de metadatos, no la actualizan; `updatedAt` no es una señal de actividad.
- `sessionFile`: marcador heredado que se conserva para mantener la compatibilidad de migración y archivado; la ejecución activa utiliza la identidad de SQLite
- `chatType`: `direct | group | room`
- `provider`, `subject`, `room`, `space`, `displayName`: metadatos de etiquetado de grupos y canales
- Opciones: `thinkingLevel`, `verboseLevel`, `reasoningLevel`, `elevatedLevel`, `sendPolicy` (anulación por sesión)
- Selección de modelo: `providerOverride`, `modelOverride`, `authProfileOverride`
- Contadores de tokens (aproximados y dependientes del proveedor): `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`
- `compactionCount`: número de veces que se ha completado la Compaction automática para esta clave de sesión
- `memoryFlushAt` / `memoryFlushCompactionCount`: marca de tiempo y número de Compaction del último vaciado de memoria previo a la Compaction

El Gateway es la autoridad: puede reescribir o rehidratar entradas a medida que se
ejecutan las sesiones. En instalaciones heredadas respaldadas por archivos, realice la migración con
`openclaw doctor --session-sqlite import --session-sqlite-all-agents` en lugar de
editar `sessions.json` y esperar que la ejecución siga leyendo ese archivo.

## Estructura de los eventos de transcripción

El descriptor de acceso a sesiones de OpenClaw gestiona las transcripciones y las expone al código de ejecución mediante asistentes basados en identidad. El flujo de eventos es de solo anexado:

- Primera entrada: encabezado de sesión: `type: "session"`, `id`, `cwd`, `timestamp` y `parentSession` opcional.
- Después: entradas con `id` + `parentId` (estructura de árbol).

Tipos de entrada destacados:

- `message`: mensajes de usuario/asistente/toolResult
- `custom_message`: mensaje inyectado por una extensión que _sí_ entra en el contexto del modelo (se representa en la TUI cuando `display: true` y se oculta por completo cuando `display: false`)
- `custom`: estado de la extensión que _no_ entra en el contexto del modelo (para conservar el estado de la extensión entre recargas)
- `compaction`: resumen persistente de Compaction con `firstKeptEntryId` y `tokensBefore`
- `branch_summary`: resumen persistente al navegar por una rama del árbol

OpenClaw no «corrige» intencionadamente las transcripciones; el Gateway utiliza `SessionManager` para leerlas y escribirlas.

## Ventanas de contexto frente a tokens registrados

Dos conceptos diferentes:

1. **Ventana de contexto del modelo**: límite estricto por modelo (tokens visibles para el modelo). Procede del catálogo de modelos y puede anularse mediante la configuración.
2. **Contadores del almacén de sesiones**: estadísticas acumulativas escritas en la fila de sesión (utilizadas para `/status` y paneles). `contextTokens` es un valor estimado o informado durante la ejecución; no debe tratarse como una garantía estricta.

Más información sobre los límites: [/reference/token-use](/es/reference/token-use).

## Compaction: qué es

Compaction resume la conversación anterior en una entrada persistente `compaction` de la transcripción y mantiene intactos los mensajes recientes. Después de Compaction, los turnos futuros ven el resumen de Compaction junto con los mensajes posteriores a `firstKeptEntryId`. Compaction es **persistente**, a diferencia de la depuración de sesiones; consulte [/concepts/session-pruning](/es/concepts/session-pruning).

De forma predeterminada, la Compaction integrada de OpenClaw hereda el nivel de razonamiento de la sesión. Establezca `agents.defaults.compaction.thinkingLevel` para usar un nivel distinto en las solicitudes de resumen; la ejecución lo limita en función de cada modelo concreto de Compaction o modelo alternativo. La Compaction nativa de App Server de Codex controla su propia solicitud de compactación y no admite una anulación del nivel de razonamiento por Compaction, por lo que OpenClaw muestra una advertencia y deja esa configuración en manos de Codex.

La reinyección de la sección AGENTS.md después de Compaction sigue siendo opcional mediante `agents.defaults.compaction.postCompactionSections`. Los Plugins pueden añadir otro contexto al prompt mediante `before_prompt_build`.

### Límites de fragmentos y emparejamiento de herramientas

Al dividir una transcripción larga en fragmentos para Compaction, OpenClaw mantiene emparejadas las llamadas a herramientas del asistente con sus entradas `toolResult` correspondientes:

- Si la división por proporción de tokens quedara entre una llamada a herramienta y su resultado, OpenClaw desplaza el límite al mensaje de llamada a herramienta del asistente en lugar de separar el par.
- Si, de otro modo, un bloque final de resultados de herramientas hiciera que el fragmento superara el objetivo, OpenClaw conserva ese bloque de herramientas pendiente y mantiene intacta la parte final sin resumir.
- Los bloques de llamadas a herramientas canceladas o con errores no mantienen abierta una división pendiente.

## Cuándo se produce la Compaction automática

Hay dos desencadenantes en el agente integrado de OpenClaw:

1. **Recuperación tras desbordamiento**: el modelo devuelve un error de desbordamiento de contexto (`request_too_large`, `context length exceeded`, `input exceeds the maximum number of tokens`, `input token count exceeds the maximum number of input tokens`, `input is too long for the model`, `ollama error: context length exceeded` y otras variantes con el formato del proveedor); se ejecuta Compaction y, después, se vuelve a intentar. Cuando el proveedor informa del número de tokens intentado, OpenClaw reenvía ese número observado a la Compaction de recuperación tras desbordamiento; si el proveedor confirma el desbordamiento, pero no expone un número analizable, OpenClaw pasa a los motores de Compaction y a los diagnósticos un número sintético mínimamente superior al presupuesto. Si la recuperación tras desbordamiento sigue fallando, OpenClaw muestra instrucciones explícitas y conserva la asignación de sesión actual en lugar de cambiar silenciosamente a un identificador de sesión nuevo: vuelva a intentar el mensaje, ejecute `/compact` o ejecute `/new`.
2. **Mantenimiento por umbral**: después de un turno correcto, cuando el contexto actual supera la ventana del modelo menos el margen incorporado de OpenClaw para los prompts y la siguiente salida del modelo.

Se ejecutan otras dos protecciones fuera de estos dos desencadenantes:

- **Compaction local previa**: establezca `agents.defaults.compaction.maxActiveTranscriptBytes` (bytes o una cadena como `"20mb"`) para activar la Compaction local antes de abrir la siguiente ejecución cuando la transcripción activa alcance ese tamaño. Esta es una protección de tamaño para el coste de reapertura local, no para el archivado sin procesar; la Compaction semántica normal se sigue ejecutando y requiere `truncateAfterCompaction` para que el resumen compactado se convierta en una nueva transcripción sucesora.
- **Comprobación previa a mitad del turno**: establezca `agents.defaults.compaction.midTurnPrecheck.enabled: true` (valor predeterminado: `false`) para añadir una protección al bucle de herramientas. Después de añadir el resultado de una herramienta y antes de la siguiente llamada al modelo, OpenClaw estima la presión sobre el prompt mediante la misma lógica de presupuesto previo utilizada al inicio del turno. Si el contexto ya no cabe, la protección no realiza la Compaction en línea: genera una señal estructurada de comprobación previa a mitad del turno, detiene el envío del prompt actual y permite que el bucle de ejecución externo use la ruta de recuperación existente (truncar los resultados de herramientas demasiado grandes cuando sea suficiente, o activar el modo de Compaction configurado y volver a intentarlo). Funciona con los modos de Compaction `default` y `safeguard`, incluida la Compaction de protección respaldada por el proveedor. Es independiente de `maxActiveTranscriptBytes`: la protección por tamaño en bytes se ejecuta antes de abrir un turno; la comprobación previa a mitad del turno se ejecuta después, una vez añadidos los nuevos resultados de herramientas.

## Configuración de Compaction

```json5
{
  agents: {
    defaults: {
      compaction: {
        enabled: true,
        keepRecentTokens: 20000,
      },
    },
  },
}
```

OpenClaw aplica una reserva integrada para las ejecuciones embebidas y la limita en función de la ventana de contexto del modelo activo para que no pueda consumir todo el presupuesto del prompt. Esto evita que los modelos locales con poco contexto entren en Compaction desde el primer token, al tiempo que deja margen suficiente para tareas de mantenimiento entre varios turnos, como el volcado de memoria.

La operación manual `/compact` respeta un valor explícito de `agents.defaults.compaction.keepRecentTokens` y conserva el punto de corte de la cola reciente del entorno de ejecución. Sin un presupuesto de conservación explícito, la Compaction manual constituye un punto de control estricto y el contexto reconstruido comienza a partir del nuevo resumen.

Cuando `truncateAfterCompaction` está habilitado, OpenClaw rota la transcripción activa a una sucesora compactada después de la Compaction. Las acciones de punto de control de ramificación/restauración utilizan esa sucesora compactada; los archivos de puntos de control heredados anteriores a la Compaction siguen siendo legibles mientras estén referenciados.

## Proveedores de Compaction conectables

Los plugins registran un proveedor de Compaction mediante `registerCompactionProvider()` en la API del plugin. Cuando `agents.defaults.compaction.provider` se establece en el id de un proveedor registrado, la extensión de protección delega el resumen en ese proveedor en lugar de usar el pipeline integrado `summarizeInStages`.

- `provider`: id de un plugin proveedor de Compaction registrado. Déjelo sin establecer para usar el resumen predeterminado del LLM. Establecer un `provider` fuerza `mode: "safeguard"`.
- Los proveedores reciben las mismas instrucciones de Compaction y la misma política de conservación de identificadores que la ruta integrada, y la protección sigue conservando el contexto del sufijo de los turnos recientes y los turnos divididos después de la salida del proveedor.
- El resumen de protección integrado vuelve a destilar los resúmenes anteriores junto con los mensajes nuevos, en lugar de conservar literalmente todo el resumen anterior.
- El modo de protección habilita de forma predeterminada las auditorías de calidad del resumen; establezca `qualityGuard.enabled: false` para omitir el comportamiento de reintento cuando la salida tenga un formato incorrecto.
- Si el proveedor falla o devuelve un resultado vacío, OpenClaw recurre automáticamente al resumen integrado del LLM. Las señales de cancelación o tiempo de espera activadas explícitamente por el invocador se vuelven a lanzar, en lugar de ignorarse, para que siempre se respete la cancelación.

Fuente: `src/plugins/compaction-provider.ts`, `src/agents/agent-hooks/compaction-safeguard.ts`.

## Superficies visibles para el usuario

- `/status` en cualquier sesión de chat
- `openclaw status` (CLI)
- `openclaw sessions` / `openclaw sessions --json`
- Registros del Gateway (`pnpm gateway:watch` o `openclaw logs --follow`): `embedded run auto-compaction start` + `complete`
- Modo detallado: `🧹 Auto-compaction complete` más el número de operaciones de Compaction

## Mantenimiento silencioso (`NO_REPLY`)

OpenClaw admite turnos «silenciosos» para tareas en segundo plano en las que el usuario no debe ver resultados intermedios.

- El asistente comienza su salida con el token silencioso exacto `NO_REPLY` / `no_reply` para indicar «no entregar una respuesta al usuario». OpenClaw lo elimina o suprime en la capa de entrega.
- La supresión del token silencioso exacto no distingue entre mayúsculas y minúsculas: `NO_REPLY` y `no_reply` cuentan por igual cuando toda la carga útil consiste únicamente en el token silencioso.
- A partir de `2026.1.10`, OpenClaw también suprime el streaming de borradores/indicaciones de escritura cuando un fragmento parcial comienza con `NO_REPLY`, para que las operaciones silenciosas no filtren resultados parciales a mitad del turno.
- Esto se aplica únicamente a turnos realmente en segundo plano o sin entrega; no es un atajo para solicitudes prácticas ordinarias del usuario.

## Volcado de memoria previo a la Compaction

Antes de que se produzca la Compaction automática, OpenClaw puede ejecutar un turno agéntico silencioso que escriba estado duradero en el disco (por ejemplo, `memory/YYYY-MM-DD.md` en el espacio de trabajo del agente) para que la Compaction no pueda borrar contexto crítico. Supervisa el uso del contexto de la sesión y, cuando supera un umbral flexible inferior al umbral de Compaction, envía una directiva silenciosa de «escribir la memoria ahora» mediante el token silencioso exacto `NO_REPLY` / `no_reply`, para que el usuario no vea nada.

Configuración (`agents.defaults.compaction.memoryFlush`), referencia completa en [/gateway/config-agents](/es/gateway/config-agents#agentsdefaultscompaction):

| Clave                       | Valor predeterminado | Notas                                                                                                                                  |
| --------------------------- | -------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                   | `true`           |                                                                                                                                        |
| `model`                     | sin establecer      | anulación exacta del proveedor/modelo solo para el turno de volcado, por ejemplo `ollama/qwen3:8b`                                      |
| `softThresholdTokens`       | `4000`           | margen por debajo del umbral de Compaction que activa un volcado                                                                        |
| `forceFlushTranscriptBytes` | sin establecer (deshabilitado) | fuerza un volcado cuando el archivo de transcripción alcanza este tamaño en bytes (o una cadena como `"2mb"`), aunque los contadores de tokens estén desactualizados; `0` lo deshabilita |

Notas:

- El prompt integrado y el prompt del sistema incluyen una indicación `NO_REPLY` para suprimir la entrega.
- Cuando se establece `model`, el turno de volcado utiliza ese modelo sin heredar la cadena de respaldo de la sesión activa, para que el mantenimiento exclusivamente local no recurra silenciosamente a un modelo de conversación de pago en caso de fallo.
- El volcado se ejecuta una vez por cada ciclo de Compaction (se registra en la fila de la sesión).
- El volcado solo se ejecuta para sesiones embebidas de OpenClaw; los backends de la CLI y los turnos de Heartbeat lo omiten.
- El volcado se omite cuando el espacio de trabajo de la sesión es de solo lectura (`workspaceAccess: "ro"` o `"none"`).
- Consulte [Memoria](/es/concepts/memory) para conocer la disposición de archivos del espacio de trabajo y los patrones de escritura.

OpenClaw expone un hook `session_before_compact` en la API de extensiones, pero la lógica de volcado descrita anteriormente reside en el Gateway (`src/auto-reply/reply/memory-flush.ts`, `src/auto-reply/reply/agent-runner-memory.ts`), no en ese hook.

## Lista de comprobación para la resolución de problemas

- **¿Clave de sesión incorrecta?** Comience por [/concepts/session](/es/concepts/session) y confirme el `sessionKey` en `/status`.
- **¿Discrepancia entre el almacén y la transcripción?** Confirme el host del Gateway y la ruta del almacén mediante `openclaw status`.
- **¿Compaction excesiva?** Compruebe la ventana de contexto del modelo (si es demasiado pequeña, fuerza una Compaction frecuente) y el aumento excesivo de los resultados de herramientas (ajuste la poda de la sesión).
- **¿Todos los prompts parecen desbordarse en un modelo local pequeño?** Confirme que el proveedor informa de la ventana de contexto correcta del modelo. OpenClaw solo puede limitar la reserva efectiva cuando se conoce esa ventana.
- **¿Se filtran los turnos silenciosos?** Confirme que la respuesta comienza con el token silencioso exacto `NO_REPLY` (sin distinguir entre mayúsculas y minúsculas) y que utiliza una compilación que incluye la corrección de supresión del streaming (`2026.1.10`+).

## Temas relacionados

- [Gestión de sesiones](/es/concepts/session)
- [Poda de sesiones](/es/concepts/session-pruning)
- [Motor de contexto](/es/concepts/context-engine)
- [Referencia de configuración del agente](/es/gateway/config-agents)
