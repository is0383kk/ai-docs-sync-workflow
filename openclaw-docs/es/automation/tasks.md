---
read_when:
    - Inspección del trabajo en segundo plano en curso o completado recientemente
    - Depuración de fallos de entrega en ejecuciones de agentes desacopladas
    - Comprender cómo se relacionan las ejecuciones en segundo plano con las sesiones, Cron y Heartbeat
sidebarTitle: Background tasks
summary: Seguimiento de tareas en segundo plano para ejecuciones de ACP, subagentes, ejecuciones de Cron y operaciones de la CLI
title: Tareas en segundo plano
x-i18n:
    generated_at: "2026-07-26T04:30:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dbdc5ced133764fec0c8b9ae7b1957e24272dc9c1c86099de81f6923955d6b5a
    source_path: automation/tasks.md
    workflow: 16
---

<Note>
¿Busca opciones de programación? Consulte [Automatización](/es/automation) para elegir el mecanismo adecuado. Esta página es el registro de actividad del trabajo en segundo plano, no el programador.
</Note>

Las tareas en segundo plano registran el trabajo que se ejecuta **fuera de la sesión de conversación principal**: ejecuciones de ACP, creación de subagentes, ejecuciones de trabajos cron y operaciones iniciadas mediante la CLI.

Las tareas **no** sustituyen a las sesiones, los trabajos cron ni los heartbeats: son el **registro de actividad** que deja constancia de qué trabajo desacoplado se realizó, cuándo y si se completó correctamente.

<Note>
No todas las ejecuciones del agente crean una tarea. Los turnos de heartbeat y el chat interactivo normal no lo hacen. Sí lo hacen todas las ejecuciones de cron, la creación de ACP, la creación de subagentes, los comandos de agente de la CLI enviados por el Gateway y los comandos en segundo plano `exec` iniciados por el agente.
</Note>

## Resumen

- Las tareas son **registros**, no programadores: cron y heartbeat deciden _cuándo_ se ejecuta el trabajo; las tareas registran _qué ocurrió_.
- ACP, los subagentes, todos los trabajos cron y las operaciones de la CLI crean tareas. Los turnos de heartbeat no.
- Cada tarea pasa por `queued → running → terminal` (succeeded, failed, timed_out, cancelled o lost).
- Las tareas cron permanecen activas mientras el entorno de ejecución de cron siga siendo responsable del trabajo; si el estado del entorno de ejecución en memoria desaparece, el mantenimiento de tareas comprueba primero el historial persistente de ejecuciones de cron antes de marcar una tarea como perdida.
- La finalización se basa en notificaciones push: el trabajo desacoplado puede notificar directamente o activar la sesión o el heartbeat del solicitante cuando termina, por lo que los bucles de sondeo de estado suelen ser un enfoque inadecuado.
- Las ejecuciones de cron aisladas y las finalizaciones de subagentes intentan, sin garantía, limpiar las pestañas y los procesos del navegador registrados para su sesión secundaria antes de realizar el registro final de limpieza.
- La entrega de cron aislada suprime las respuestas provisionales obsoletas del agente principal mientras el trabajo de los subagentes descendientes sigue terminando y da preferencia a la salida final de los descendientes si llega antes de la entrega.
- Las notificaciones de finalización se entregan directamente a un canal o se ponen en cola para el siguiente heartbeat.
- `openclaw tasks list` muestra todas las tareas; `openclaw tasks audit` presenta los problemas.
- Los registros terminales se conservan durante 7 días (los registros `lost`, durante 24 horas) y después se eliminan automáticamente.

## Inicio rápido

<Tabs>
  <Tab title="Mostrar y filtrar">
    ```bash
    # Mostrar todas las tareas (primero las más recientes)
    openclaw tasks list

    # Filtrar por entorno de ejecución o estado
    openclaw tasks list --runtime acp
    openclaw tasks list --status running
    ```

  </Tab>
  <Tab title="Inspeccionar">
    ```bash
    # Mostrar los detalles de una tarea específica (por ID de tarea, ID de ejecución o clave de sesión)
    openclaw tasks show <lookup>
    ```
  </Tab>
  <Tab title="Cancelar y notificar">
    ```bash
    # Cancelar una tarea en ejecución (termina la sesión secundaria)
    openclaw tasks cancel <lookup>

    # Cambiar la política de notificaciones de una tarea
    openclaw tasks notify <lookup> state_changes
    ```

  </Tab>
  <Tab title="Auditoría y mantenimiento">
    ```bash
    # Ejecutar una auditoría de estado
    openclaw tasks audit

    # Previsualizar o aplicar el mantenimiento
    openclaw tasks maintenance
    openclaw tasks maintenance --apply
    ```

  </Tab>
  <Tab title="Flujo de tareas">
    ```bash
    # Inspeccionar el estado de TaskFlow
    openclaw tasks flow list
    openclaw tasks flow show <lookup>
    openclaw tasks flow cancel <lookup>
    ```
  </Tab>
</Tabs>

## Qué crea una tarea

| Origen                 | Tipo de entorno de ejecución | Cuándo se crea un registro de tarea                                          | Política de notificaciones predeterminada |
| ---------------------- | ------------ | ---------------------------------------------------------------------- | --------------------- |
| Ejecuciones de ACP en segundo plano    | `acp`        | Al crear una sesión secundaria de ACP                                           | `done_only`           |
| Orquestación de subagentes | `subagent`   | Al crear un subagente mediante `sessions_spawn`                               | `done_only`           |
| Trabajos cron (de cualquier tipo)  | `cron`       | En cada ejecución de cron (en la sesión principal o aislada)                       | `silent`              |
| Operaciones de la CLI         | `cli`        | Comandos `openclaw agent` que se ejecutan mediante el Gateway                 | `silent`              |
| Trabajos multimedia del agente       | `cli`        | Ejecuciones `image_generate`/`music_generate`/`video_generate` respaldadas por una sesión | `silent`              |

<AccordionGroup>
  <Accordion title="Valores predeterminados de notificación para cron y contenido multimedia">
    Las tareas cron (tanto de la sesión principal como aisladas) usan la política de notificaciones `silent`: crean registros para su seguimiento, pero no generan notificaciones de tarea propias; cron controla su ruta de entrega.

    Las ejecuciones `image_generate`, `music_generate` y `video_generate` respaldadas por una sesión también usan la política de notificaciones `silent`. Siguen creando registros de tareas, pero la finalización se devuelve a la sesión original del agente como una activación interna para que el agente pueda escribir el mensaje de seguimiento y adjuntar por sí mismo el contenido multimedia terminado. El agente solicitante sigue su contrato normal de respuesta visible: una respuesta final automática cuando está configurada, o `message(action="send")` junto con `NO_REPLY` cuando la sesión requiere respuestas mediante la herramienta de mensajes. Si la sesión solicitante ya no está activa o falla su activación, y el agente de finalización omite parte o la totalidad del contenido multimedia generado, OpenClaw envía directamente al destino del canal original una alternativa idempotente que contiene únicamente el contenido multimedia faltante.

  </Accordion>
  <Accordion title="Protección contra la generación simultánea de contenido multimedia">
    Mientras una tarea de generación de contenido multimedia respaldada por una sesión siga activa, `image_generate`, `music_generate` y `video_generate` protegen contra reintentos accidentales: repetir la llamada con la misma solicitud o instrucción devuelve el estado de la tarea activa coincidente en lugar de iniciar un duplicado, mientras que una instrucción distinta puede iniciar su propia tarea. Use `action: "status"` cuando desee consultar explícitamente el progreso o el estado desde el agente.
  </Accordion>
  <Accordion title="Qué no crea tareas">
    - Turnos de heartbeat en la sesión principal; consulte [Heartbeat](/es/gateway/heartbeat)
    - Turnos normales de chat interactivo
    - Respuestas directas de `/command`

  </Accordion>
</AccordionGroup>

## Ciclo de vida de las tareas

```mermaid
stateDiagram-v2
    [*] --> queued
    queued --> running : el agente se inicia
    running --> succeeded : se completa correctamente
    running --> failed : error
    running --> timed_out : se supera el tiempo de espera
    queued --> cancelled : el operador cancela
    running --> cancelled : el operador cancela
    queued --> lost : el estado subyacente desaparece durante más de 5 min
    running --> lost : el estado subyacente desaparece durante más de 5 min
```

| Estado      | Significado                                                               |
| ----------- | --------------------------------------------------------------------------- |
| `queued`    | Creada, a la espera de que se inicie el agente                                     |
| `running`   | El turno del agente se está ejecutando activamente                                            |
| `succeeded` | Completada correctamente                                                      |
| `failed`    | Completada con un error                                                     |
| `timed_out` | Se superó el tiempo de espera configurado                                             |
| `cancelled` | Detenida por el operador mediante `openclaw tasks cancel`, o se anuló la ejecución |
| `lost`      | El entorno de ejecución perdió el estado subyacente autoritativo tras un periodo de gracia de 5 minutos  |

Las transiciones se producen automáticamente: los eventos del ciclo de vida de la ejecución del agente (inicio, finalización y error) actualizan el estado de la tarea; no se administra manualmente.

La finalización de la ejecución del agente es autoritativa para los registros de tareas activas. Una ejecución desacoplada correcta termina como `succeeded`, los errores normales de ejecución terminan como `failed`, los tiempos de espera terminan como `timed_out` y los resultados de cancelación o anulación terminan como `cancelled`. Una vez que una tarea alcanza un estado terminal, las señales posteriores del ciclo de vida no reducen su estado: una tarea cancelada por el operador o que ya tenga el estado `failed`/`timed_out`/`lost` permanece así aunque después llegue una señal de finalización correcta.

`lost` tiene en cuenta el entorno de ejecución:

- Tareas de ACP: solo un turno de ACP activo dentro del proceso del Gateway demuestra que la ejecución sigue activa; los metadatos persistentes de la sesión por sí solos no lo demuestran. La auditoría de la CLI sin conexión es conservadora y nunca recupera tareas de ACP.
- Tareas de subagentes: la sesión secundaria subyacente desapareció del almacén del agente de destino (o contiene una marca de recuperación tras reinicio).
- Tareas cron: el entorno de ejecución de cron ya no registra el trabajo como activo y el historial persistente de ejecuciones de cron no muestra un resultado terminal para esa ejecución. La auditoría de la CLI sin conexión no considera autoritativo el estado vacío de su propio entorno de ejecución de cron dentro del proceso.
- Tareas de la CLI: las tareas con un ID de ejecución o de origen usan el contexto de ejecución activo, por lo que las filas persistentes de sesiones secundarias o de chat no las mantienen activas después de que desaparece la ejecución controlada por el Gateway. Las tareas heredadas de la CLI sin identidad de ejecución siguen recurriendo a la sesión secundaria. Las ejecuciones `openclaw agent` respaldadas por el Gateway también terminan según el resultado de su ejecución, por lo que las ejecuciones completadas no permanecen activas hasta que el proceso de limpieza las marca como `lost`.

## Entrega y notificaciones

Cuando una tarea alcanza un estado terminal, OpenClaw envía una notificación. Hay dos rutas de entrega:

**Entrega directa**: si la tarea tiene un canal de destino (el `requesterOrigin`), el mensaje de finalización se envía directamente a ese canal (Discord, Slack, Telegram, etc.). En cambio, las finalizaciones de tareas de grupos y canales se encaminan mediante la sesión solicitante para que el agente principal pueda escribir la respuesta visible. Para las finalizaciones de subagentes, OpenClaw también conserva el enrutamiento vinculado a hilos o temas cuando está disponible y puede completar un `to` o una cuenta faltantes a partir de la ruta almacenada de la sesión solicitante (`lastChannel` / `lastTo` / `lastAccountId`) antes de renunciar a la entrega directa.

**Entrega en cola de la sesión**: si falla la entrega directa o no se establece un origen, la actualización se pone en cola como un evento del sistema en la sesión solicitante y aparece en el siguiente heartbeat.

<Tip>
Las finalizaciones de tareas puestas en cola en la sesión activan inmediatamente un heartbeat, por lo que el resultado aparece rápidamente: no es necesario esperar al siguiente ciclo programado del heartbeat.
</Tip>

Esto significa que el flujo de trabajo habitual se basa en notificaciones push: se inicia una vez el trabajo desacoplado y después se deja que el entorno de ejecución active una sesión o envíe una notificación al finalizar. Consulte el estado de las tareas únicamente cuando necesite depurar, intervenir o realizar una auditoría explícita.

### Políticas de notificaciones

Controle cuánta información se recibe sobre cada tarea:

| Política                | Qué se entrega                                       |
| --------------------- | ------------------------------------------------------- |
| `done_only` (predeterminada) | Solo el estado terminal (succeeded, failed, etc.)           |
| `state_changes`       | Todas las transiciones de estado y actualizaciones de progreso              |
| `silent`              | Nada en absoluto (valor predeterminado para las tareas cron, de la CLI y multimedia) |

Cambie la política mientras se ejecuta una tarea:

```bash
openclaw tasks notify <lookup> state_changes
```

## Referencia de la CLI

<AccordionGroup>
  <Accordion title="tasks list">
    ```bash
    openclaw tasks list [--runtime <acp|subagent|cron|cli>] [--status <status>] [--json]
    ```

    Columnas de salida: Tarea, Tipo, Estado, Entrega, Ejecución, Sesión secundaria, Resumen. `openclaw tasks` sin argumentos se comporta como `openclaw tasks list`.

  </Accordion>
  <Accordion title="tasks show">
    ```bash
    openclaw tasks show <lookup> [--json]
    ```

    El token de búsqueda acepta un ID de tarea, un ID de ejecución o una clave de sesión. Muestra el registro completo, incluidos los tiempos, el estado de entrega, el error y el resumen terminal.

  </Accordion>
  <Accordion title="tasks cancel">
    ```bash
    openclaw tasks cancel <lookup>
    ```

    Para las tareas ACP y de subagentes, esto finaliza la sesión secundaria; las cancelaciones de ACP y Cron se enrutan a través del Gateway en ejecución (`tasks.cancel`). Para las tareas rastreadas por la CLI, la cancelación se registra en el registro de tareas (no hay un identificador independiente del entorno de ejecución secundario). El estado cambia a `cancelled` y, cuando corresponde, se envía una notificación de entrega.

  </Accordion>
  <Accordion title="notificación de tareas">
    ```bash
    openclaw tasks notify <lookup> <done_only|state_changes|silent>
    ```
  </Accordion>
  <Accordion title="auditoría de tareas">
    ```bash
    openclaw tasks audit [--severity <warn|error>] [--code <name>] [--limit <n>] [--json]
    ```

    Muestra en un único informe los problemas operativos de las tareas **y** los TaskFlows. Los hallazgos también aparecen en `openclaw status` cuando se detectan problemas.

    Hallazgos de tareas:

    | Hallazgo                  | Gravedad   | Desencadenante                                                                                               |
    | ------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------ |
    | `stale_queued`            | advertencia | En cola durante más de 10 minutos                                                                           |
    | `stale_running`           | error      | En ejecución durante más de 30 minutos                                                                      |
    | `lost`                    | advertencia/error | La propiedad de la tarea respaldada por el entorno de ejecución desapareció; las tareas perdidas retenidas generan advertencias hasta `cleanupAfter` y después se convierten en errores |
    | `delivery_failed`         | advertencia | La entrega falló y la política de notificación no es `silent`                                     |
    | `missing_cleanup`         | advertencia | Tarea terminal sin marca de tiempo de limpieza                                                              |
    | `inconsistent_timestamps` | advertencia | Infracción de la cronología (por ejemplo, terminó antes de comenzar)                                        |

    Hallazgos de TaskFlow:

    | Hallazgo               | Gravedad   | Desencadenante                                                                 |
    | ---------------------- | ---------- | ----------------------------------------------------------------------------- |
    | `restore_failed`       | error      | Falló la restauración del registro de flujos desde SQLite                     |
    | `stale_running`        | error      | El flujo en ejecución no ha avanzado durante más de 30 minutos                |
    | `stale_waiting`        | advertencia | El flujo en espera no ha avanzado durante más de 30 minutos                   |
    | `stale_blocked`        | advertencia | El flujo bloqueado no ha avanzado durante más de 30 minutos                   |
    | `cancel_stuck`         | advertencia | La cancelación se solicitó hace más de 5 minutos, no hay tareas secundarias activas y el flujo sigue sin ser terminal |
    | `missing_linked_tasks` | advertencia/error | Flujo administrado obsoleto sin tareas vinculadas ni estado de espera         |
    | `blocked_task_missing` | advertencia | El flujo bloqueado apunta a un identificador de tarea que ya no existe        |

  </Accordion>
  <Accordion title="mantenimiento de tareas">
    ```bash
    openclaw tasks maintenance [--json]
    openclaw tasks maintenance --apply [--json]
    ```

    Se utiliza para obtener una vista previa o aplicar la conciliación, el marcado temporal de limpieza y la depuración de tareas, del estado de TaskFlow y de las filas obsoletas del registro de sesiones de ejecuciones de Cron.

    La conciliación tiene en cuenta el entorno de ejecución:

    - Las tareas ACP requieren un turno activo en proceso dentro del Gateway; las tareas de subagentes comprueban su sesión secundaria subyacente.
    - Las tareas de subagentes cuya sesión secundaria tiene una marca de recuperación tras reinicio se marcan como perdidas en lugar de tratarse como sesiones subyacentes recuperables.
    - Las tareas de Cron comprueban si el entorno de ejecución de Cron aún posee el trabajo y después recuperan el estado terminal de los registros persistentes de ejecuciones de Cron o del estado del trabajo antes de recurrir a `lost`. Solo el proceso del Gateway es la fuente autoritativa del conjunto en memoria de trabajos de Cron activos; la auditoría sin conexión mediante la CLI utiliza el historial persistente, pero no marca una tarea de Cron como perdida únicamente porque ese conjunto local esté vacío.
    - Las tareas de la CLI con identidad de ejecución comprueban el contexto activo de la ejecución propietaria, no solo las filas de la sesión secundaria o de chat.

    La limpieza al finalizar también tiene en cuenta el entorno de ejecución:

    - Al finalizar un subagente, se intenta cerrar las pestañas y los procesos del navegador rastreados para la sesión secundaria antes de continuar con la limpieza del anuncio.
    - Al finalizar una ejecución aislada de Cron, se intenta cerrar las pestañas y los procesos del navegador rastreados para la sesión de Cron antes de que la ejecución termine por completo.
    - La entrega de una ejecución aislada de Cron espera a que finalice el seguimiento de los subagentes descendientes cuando es necesario y suprime el texto obsoleto de confirmación del proceso principal en lugar de anunciarlo.
    - La entrega al finalizar un subagente utiliza únicamente el texto visible más reciente del asistente secundario. La salida de tool/toolResult no se convierte en texto del resultado secundario. Las ejecuciones terminales fallidas anuncian el estado de error sin reproducir el texto de respuesta capturado.
    - Los fallos de limpieza no ocultan el resultado real de la tarea.

    Al aplicar el mantenimiento, OpenClaw también elimina las filas obsoletas `cron:<jobId>:run:<runId>` del registro de sesiones con más de 7 días de antigüedad, conserva las filas de los trabajos de Cron actualmente en ejecución y no modifica las filas de sesiones ajenas a Cron.

  </Accordion>
  <Accordion title="listar | mostrar | cancelar flujos de tareas">
    ```bash
    openclaw tasks flow list [--status <status>] [--json]
    openclaw tasks flow show <lookup> [--json]
    openclaw tasks flow cancel <lookup>
    ```

    El token de búsqueda del flujo acepta un identificador de flujo o una clave de propietario. Se utilizan cuando lo importante es el [flujo de tareas](/es/automation/taskflow) que realiza la orquestación, en lugar de un registro individual de una tarea en segundo plano.

  </Accordion>
</AccordionGroup>

## Panel de tareas del chat (`/tasks`)

Utilice `/tasks` en cualquier sesión de chat para ver las tareas en segundo plano vinculadas a esa sesión. El panel muestra hasta cinco tareas activas y completadas recientemente, con detalles del entorno de ejecución, el estado, los tiempos y el progreso o los errores.

Cuando la sesión actual no tiene tareas vinculadas visibles, `/tasks` recurre a los recuentos de tareas locales del agente, por lo que se sigue obteniendo una vista general sin revelar detalles de otras sesiones.

Para consultar el registro completo del operador, utilice la CLI: `openclaw tasks list`.

### Interfaz de control

La interfaz de control web tiene una página **Tareas** en la barra lateral con las tareas en segundo plano activas y recientes en tiempo real. Se utiliza para inspeccionar el progreso, abrir sesiones vinculadas, actualizar el registro o cancelar tareas en cola y en ejecución.

Los paneles de chat también tienen una barra contraíble de **Tareas en segundo plano** limitada al agente del panel: tareas y subagentes en ejecución con un control para detenerlos, una sección de tareas finalizadas y enlaces para Ver la transcripción de la sesión secundaria de cada tarea. Ábrala desde el selector de actividad del encabezado del panel (o desde el botón flotante de actividad en el chat de un solo panel).

Seleccione una tarea en la barra para inspeccionar su instrucción de entrada delimitada y el resumen más reciente de la salida o del error. El trabajo en ejecución permanece separado del finalizado, y las filas finalizadas indican si la tarea se completó o falló. En iOS, abra **Acciones del chat → Tareas en segundo plano**; en Android, abra el menú adicional del chat y seleccione **Tareas en segundo plano**. Ambas vistas móviles utilizan la misma agrupación de En ejecución y Finalizadas, y abren los detalles de la tarea al seleccionarla.

## Integración del estado (carga de tareas)

`openclaw status` incluye una línea de tareas que permite consultar el estado de un vistazo:

```
Tareas    2 activas · 1 en cola · 1 en ejecución · 1 problema · auditoría correcta · 6 rastreadas
```

El resumen cuenta el trabajo activo (`queued` + `running`), los fallos (`failed` + `timed_out` + `lost`), los hallazgos de auditoría y el total de registros rastreados; la carga JSON también desglosa los recuentos por entorno de ejecución (`acp`, `subagent`, `cron`, `cli`).

Tanto `/status` como la herramienta `session_status` utilizan una instantánea de tareas que tiene en cuenta la limpieza: se da prioridad a las tareas activas, se ocultan las filas caducadas y las tareas terminales solo aparecen durante un breve periodo reciente (5 minutos); cuando no queda trabajo activo, se destacan los fallos. De este modo, la tarjeta de estado se centra en lo importante en ese momento.

## Almacenamiento y mantenimiento

### Ubicación de las tareas

Los registros de tareas y el estado de entrega persisten en la base de datos de estado SQLite compartida de OpenClaw:

```
~/.openclaw/state/openclaw.sqlite   (tablas: task_runs, task_delivery_state, flow_runs)
```

Establezca `OPENCLAW_STATE_DIR` para trasladar toda la raíz de estado (de forma predeterminada, `~/.openclaw`) a otra ubicación; la ruta de la base de datos compartida se traslada con ella.

El registro se carga en memoria la primera vez que se utiliza y cada escritura se vuelve a guardar en SQLite, por lo que los registros sobreviven a los reinicios del Gateway. El crecimiento del WAL se mantiene limitado mediante el umbral de puntos de control automáticos predeterminado de SQLite y los puntos de control periódicos `PASSIVE`; los puntos de control del apagado y del mantenimiento explícito utilizan `TRUNCATE`, por lo que los cierres normales recuperan el espacio del WAL sin hacer que el proceso de limpieza en segundo plano espere a los lectores activos.

Los almacenes auxiliares heredados de instalaciones anteriores (`tasks/runs.sqlite`, `flows/registry.sqlite`) se importan en la base de datos compartida mediante `openclaw doctor`.

### Mantenimiento automático

Un proceso de limpieza se ejecuta cada **60 segundos** (la primera pasada ocurre unos 5 segundos después de iniciarse el Gateway) y se ocupa de cuatro tareas:

<Steps>
  <Step title="Conciliación">
    Comprueba si las tareas activas siguen teniendo un respaldo autoritativo del entorno de ejecución. Las tareas ACP requieren un turno activo en proceso, las tareas de subagentes utilizan el estado de la sesión secundaria, las tareas de Cron utilizan la propiedad del trabajo activo junto con el historial persistente de ejecuciones y las tareas de la CLI con identidad de ejecución utilizan el contexto de la ejecución propietaria. Si el estado subyacente ha desaparecido durante más de 5 minutos (30 minutos para las tareas nativas de subagentes sin sesión secundaria), la tarea se marca como `lost`.
  </Step>
  <Step title="Reparación de sesiones ACP">
    Cierra las sesiones ACP terminales o huérfanas de un solo uso propiedad del proceso principal, y cierra las sesiones ACP persistentes obsoletas que sean terminales o huérfanas únicamente cuando no quede ninguna vinculación activa de conversación.
  </Step>
  <Step title="Marcado temporal de limpieza">
    Establece una marca de tiempo `cleanupAfter` en las tareas terminales (hora de finalización + periodo de retención). Durante la retención, las tareas perdidas siguen apareciendo en la auditoría como advertencias; después de que caduque `cleanupAfter` o cuando falten los metadatos de limpieza, se convierten en errores.
  </Step>
  <Step title="Depuración">
    Elimina los registros posteriores a su fecha `cleanupAfter`.
  </Step>
</Steps>

<Note>
**Retención:** los registros de tareas terminales se conservan durante **7 días** (los registros `lost` durante **24 horas**) y después se depuran automáticamente. No se requiere ninguna configuración.
</Note>

## Relación entre las tareas y otros sistemas

<AccordionGroup>
  <Accordion title="Tareas y flujo de tareas">
    El [flujo de tareas](/es/automation/taskflow) es la capa de orquestación de flujos situada por encima de las tareas en segundo plano. Un único flujo puede coordinar varias tareas a lo largo de su ciclo de vida mediante modos de sincronización administrados o reflejados. Utilice `openclaw tasks` para inspeccionar registros individuales de tareas y `openclaw tasks flow` para inspeccionar el flujo que las orquesta.

  </Accordion>
  <Accordion title="Tareas y Cron">
    Las definiciones de trabajos de Cron, el estado de ejecución y el historial de ejecuciones se almacenan en la base de datos de estado SQLite compartida de OpenClaw. **Cada** ejecución de Cron crea un registro de tarea, tanto para la sesión principal como para una sesión aislada, con la política de notificación `silent`, por lo que las ejecuciones de Cron se rastrean sin generar notificaciones de tareas propias.

    Consulte [Trabajos de Cron](/es/automation/cron-jobs).

  </Accordion>
  <Accordion title="Tareas y Heartbeat">
    Las ejecuciones de Heartbeat son turnos de la sesión principal; no crean registros de tareas. Cuando se completa una tarea, puede activar un despertar de Heartbeat para que el resultado aparezca de inmediato.

    Consulte [Heartbeat](/es/gateway/heartbeat).

  </Accordion>
  <Accordion title="Tareas y sesiones">
    Una tarea puede hacer referencia a una `childSessionKey` (donde se ejecuta el trabajo) y a un `requesterSessionKey` (quien la inició). Su `agentId` identifica al agente que ejecuta el trabajo, mientras que los campos de solicitante y propietario conservan el contexto de inicio y control. Las sesiones constituyen el contexto de la conversación; las tareas permiten hacer un seguimiento de la actividad sobre ese contexto.
  </Accordion>
  <Accordion title="Tareas y ejecuciones de agentes">
    El `runId` de una tarea se vincula con la ejecución del agente que realiza el trabajo. Los eventos del ciclo de vida del agente (inicio, finalización, error) actualizan automáticamente el estado de la tarea; no es necesario gestionar el ciclo de vida manualmente.
  </Accordion>
</AccordionGroup>

## Contenido relacionado

- [Automatización](/es/automation) - todos los mecanismos de automatización de un vistazo
- [CLI: Tareas](/es/cli/tasks) - referencia de comandos de la CLI
- [Heartbeat](/es/gateway/heartbeat) - turnos periódicos de la sesión principal
- [Tareas programadas](/es/automation/cron-jobs) - programación de trabajo en segundo plano
- [TaskFlow](/es/automation/taskflow) - orquestación de flujos por encima de las tareas
