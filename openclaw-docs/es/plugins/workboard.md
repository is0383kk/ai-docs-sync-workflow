---
read_when:
    - Quieres un tablero de trabajo de estilo Kanban en la interfaz de control
    - Está habilitando o deshabilitando el plugin Workboard incluido.
    - Se desea realizar un seguimiento del trabajo planificado de los agentes sin un gestor de proyectos externo
summary: Tablero de trabajo opcional para tarjetas gestionadas por agentes y transferencia de sesiones
title: Plugin de tablero de trabajo
x-i18n:
    generated_at: "2026-07-26T05:23:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ec05c990c3559015780d9cb80f3ceedd7cc79db89ccf1afd65906c8c7630331
    source_path: plugins/workboard.md
    workflow: 16
---

El plugin Workboard añade un tablero opcional de estilo Kanban a la
[interfaz de control](/es/web/control-ui): tarjetas de trabajo de tamaño adecuado para agentes, asignación a agentes
y un enlace a la tarea, la ejecución y la sesión del panel asociadas a la tarjeta.

Workboard es intencionadamente pequeño: realiza el seguimiento del trabajo operativo local de un
Gateway de OpenClaw. No sustituye a GitHub Issues, Linear, Jira ni a
otros sistemas de gestión de proyectos en equipo.

## Habilitarlo

Workboard está incluido, pero deshabilitado de forma predeterminada:

1. Abra **Plugins** en la interfaz de control o use `/settings/plugins` en relación con
   la ruta base configurada de la interfaz de control. Por ejemplo, una ruta base de `/openclaw`
   usa `/openclaw/settings/plugins`.
2. Busque **Workboard** y seleccione **Enable**. Como Workboard está incluido con
   OpenClaw, no requiere la acción **Install**.
3. Si la interfaz indica que es necesario reiniciar, reinicie el Gateway.

La pestaña Workboard aparece en la navegación del panel una vez que se carga el entorno de ejecución
del plugin. Mientras está deshabilitado, la pestaña permanece oculta en la navegación. Si se abre
directamente la ruta `/workboard` mientras el plugin está deshabilitado o bloqueado por
`plugins.allow`/`plugins.deny`, se muestra un estado de plugin no disponible en lugar de los datos
de las tarjetas.

El flujo de trabajo equivalente mediante la CLI es:

```bash
openclaw plugins enable workboard
openclaw gateway restart
openclaw dashboard
```

## Configuración

Workboard no tiene configuración específica del plugin. Habilítelo o deshabilítelo con la entrada
estándar del plugin:

```json5
{
  plugins: {
    entries: {
      workboard: {
        enabled: true,
        config: {},
      },
    },
  },
}
```

```bash
openclaw plugins disable workboard
openclaw gateway restart
```

## Campos de las tarjetas

| Campo       | Valores                                                                                                       |
| ----------- | ------------------------------------------------------------------------------------------------------------- |
| `status`    | `triage`, `backlog`, `todo`, `scheduled`, `ready`, `running`, `review`, `blocked`, `done`                     |
| `priority`  | `low`, `normal`, `high`, `urgent`                                                                             |
| `labels`    | cadenas de formato libre                                                                                      |
| `agentId`   | agente asignado opcional                                                                                      |
| referencias vinculadas | tarea, ejecución, sesión o URL de origen opcionales                                                         |
| `execution` | metadatos opcionales de una ejecución de Codex/Claude iniciada desde la tarjeta (motor, modo, modelo, sesión, id. de ejecución, estado) |

Las tarjetas también contienen metadatos compactos sobre intentos, comentarios, enlaces, pruebas,
artefactos, ajustes de automatización, archivos adjuntos, registros de trabajadores, estado del protocolo
de trabajadores, reclamaciones, diagnósticos, notificaciones, id. de plantilla, estado de archivado y
detección de sesiones obsoletas, además de una lista de eventos recientes (`created`, `edited`,
`moved`, `linked`, `specified`, `decomposed`, `claimed`, `heartbeat`,
`execution_updated`, `attempt_started`, `attempt_updated`, `comment_added`,
`link_added`, `proof_added`, `artifact_added`, `attachment_added`,
`diagnostic`, `notification`, `dispatch`, `orchestration`,
`protocol_violation`, `archived`, `unarchived`, `stale`). Estos metadatos permiten que un
operador vea cómo se desplazó una tarjeta por el tablero sin abrir la sesión vinculada;
constituyen contexto operativo local, no un sustituto de las transcripciones de sesiones
ni del historial de incidencias de GitHub.

El plugin y la interfaz de control usan un único contrato de tarjetas de Workboard. Por tanto, las
actualizaciones del panel conservan la procedencia y la autoridad del espacio de trabajo, el estado de
reclamación, las acciones de diagnóstico y los números de secuencia de las notificaciones, en lugar de
proyectar una copia más reducida de la tarjeta destinada únicamente a la interfaz. Los tipos de diagnóstico,
las gravedades de diagnóstico y los tipos de notificación desconocidos se ignoran hasta que ambas
superficies los admitan; nunca se reescriben como otro estado válido.

El panel abierto se actualiza mediante invalidaciones de `plugin.workboard.changed`. Cada
evento contiene únicamente una época y una revisión del almacén; después, la interfaz vuelve a leer las
tarjetas canónicas mediante la RPC `operator.read` normal. Varias revisiones se agrupan en
una única lectura posterior. Workboard aplaza esa lectura mientras se arrastra, edita o escribe una
tarjeta y la reanuda cuando termina la interacción local. Una reconexión siempre realiza una recarga
canónica. No existe un sondeo completo rutinario de las tarjetas y **Refresh** sigue disponible como
mecanismo de recuperación manual.

Cuando existe más de un tablero, la barra de herramientas incluye un filtro **Board** respaldado
por metadatos persistentes de los tableros, en lugar de basarse únicamente en las tarjetas visibles en ese
momento. Por tanto, los tableros vacíos y archivados siguen siendo seleccionables. Las tarjetas sin un id.
de tablero explícito pertenecen al tablero canónico `default`. Cada tablero tiene una página
canónica `/workboard/<boardId>` que puede añadirse a marcadores, compartirse o fijarse en la
barra lateral. El formato `/workboard?board=<boardId>` distribuido anteriormente se mantiene como
alias de compatibilidad y redirige a esa página conservando los demás parámetros de consulta.
Al seleccionar **All boards**, se vuelve a `/workboard`.

Las tarjetas se almacenan en el estado propio del Gateway del plugin y se trasladan junto con el resto
del estado de OpenClaw de ese Gateway (consulte [Almacenamiento](#storage)).

## Iniciar trabajo desde una tarjeta

Las tarjetas no vinculadas pueden iniciar trabajo directamente:

- **Run Codex** / **Run Claude** inicia una ejecución de agente con seguimiento de tareas y un
  motor explícito, envía la instrucción de la tarjeta y marca la tarjeta como `running`. Las ejecuciones
  de Codex usan `openai/gpt-5.6-sol`; las de Claude usan `anthropic/claude-sonnet-4-6`.
- **Open Codex** / **Open Claude** crea una sesión vinculada en el panel sin
  enviar la instrucción de la tarjeta ni moverla, para realizar trabajo manual que permanece
  asociado al tablero.

Los inicios autónomos usan la ruta de ejecución de agentes con seguimiento de tareas del Gateway
(con el agente y el modelo predeterminados, salvo que se elija explícitamente Codex/Claude); después,
Workboard vincula a la tarjeta la tarea resultante, el id. de ejecución y la clave de sesión. Cada
ejecución vinculada también registra un resumen del intento (motor, modo, modelo, id. de ejecución,
marcas de tiempo, estado y recuento acumulado de errores) para que los errores repetidos permanezcan visibles.

El panel actualiza el estado de las tareas desde el registro de tareas del Gateway y relaciona
las tareas con las tarjetas mediante el id. de tarea, el id. de ejecución o la clave de sesión vinculada. Una
tarea en cola o en ejecución mantiene activo el ciclo de vida de la tarjeta; una tarea finalizada, fallida,
agotada por tiempo o cancelada desplaza la tarjeta hacia `review` o `blocked` mediante la misma
regla de sincronización que las sesiones vinculadas (consulte [Sincronización del ciclo de vida de las sesiones](#session-lifecycle-sync)).

## Herramientas de agentes

| Herramienta                                                                                                                                       | Propósito                                                                                                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `workboard_list`                                                                                                                               | Enumera tarjetas compactas con el estado de reclamación/diagnóstico; filtro de tablero opcional.                                                                                         |
| `workboard_read`                                                                                                                               | Devuelve una tarjeta junto con contexto acotado del trabajador (notas, intentos, comentarios, enlaces, pruebas, artefactos, resultados principales, trabajo reciente del asignado, diagnósticos activos). |
| `workboard_create`                                                                                                                               | Crea una tarjeta con elementos principales, inquilino, Skills, tablero, metadatos del espacio de trabajo, clave de idempotencia, límite de ejecución y presupuesto de reintentos opcionales. |
| `workboard_link`                                                                                                                               | Vincula una tarjeta principal con una secundaria. Las secundarias permanecen en `todo` hasta que todas las principales alcanzan `done`; entonces la promoción de despacho las mueve a `ready`. |
| `workboard_claim`                                                                                                                               | Reclama una tarjeta para el agente que realiza la llamada; mueve `backlog`/`todo`/`ready` a `running`.                                           |
| `workboard_heartbeat`                                                                                                                               | Actualiza el Heartbeat de la reclamación durante una ejecución más larga.                                                                                                                |
| `workboard_release`                                                                                                                               | Libera la reclamación tras completarla, pausarla o transferirla; puede mover la tarjeta a un estado posterior.                                                                           |
| `workboard_complete` / `workboard_block`                                                                                                         | Herramientas estructuradas del ciclo de vida para resúmenes finales, pruebas, artefactos y manifiestos de tarjetas creadas (deben hacer referencia a tarjetas vinculadas con la tarjeta completada) o motivos de bloqueo. |
| `workboard_attachment_add` / `workboard_attachment_read` / `workboard_attachment_delete`                                                                                   | Almacena pequeños archivos adjuntos de tarjetas en el estado SQLite del Plugin, los indexa en la tarjeta y los expone en el contexto del trabajador.                                    |
| `workboard_worker_log` / `workboard_protocol_violation`                                                                                                         | Registra líneas del registro del trabajador y bloquea una tarjeta cuando un trabajador automatizado se detiene sin llamar a `workboard_complete`/`workboard_block`.                     |
| `workboard_board_create` / `workboard_board_archive` / `workboard_board_delete`                                                                                   | Gestiona los metadatos persistentes del tablero (nombre para mostrar, descripción, estado de archivo y espacio de trabajo predeterminado).                                               |
| `workboard_runs`                                                                                                                               | Devuelve el historial persistente de intentos de ejecución de una tarjeta.                                                                                                               |
| `workboard_specify`                                                                                                                               | Convierte una tarjeta preliminar de triaje/tareas pendientes en una tarjeta `todo` aclarada; registra el resumen de la especificación en la tarjeta.                         |
| `workboard_decompose`                                                                                                                               | Distribuye una tarjeta principal de orquestación en tarjetas secundarias vinculadas que heredan los metadatos de tablero/inquilino; puede completar la principal con un manifiesto de tarjetas creadas. |
| `workboard_notify_subscribe` / `workboard_notify_list` / `workboard_notify_events` / `workboard_notify_advance` / `workboard_notify_unsubscribe`                                         | Gestiona las suscripciones a notificaciones. Las lecturas de eventos permiten una reproducción segura; `advance` mueve el cursor duradero para que los llamadores reanuden sin perder ni leer dos veces los eventos de tarjetas completadas, fallidas u obsoletas. |
| `workboard_boards` / `workboard_stats`                                                                                                         | Inspecciona los espacios de nombres del tablero y las estadísticas de la cola.                                                                                                           |
| `workboard_promote` / `workboard_reassign` / `workboard_reclaim`                                                                                   | Recupera o transfiere trabajo atascado.                                                                                                                                                   |
| `workboard_comment` / `workboard_proof`                                                                                                         | Añade notas de transferencia o adjunta referencias a pruebas/artefactos.                                                                                                                 |
| `workboard_unblock`                                                                                                                               | Devuelve el trabajo bloqueado a `todo`.                                                                                                                                      |
| `workboard_move`                                                                                                                               | Mueve una tarjeta a otro estado; las tarjetas reclamadas requieren el ámbito de reclamación del agente llamador.                                                                         |
| `workboard_dispatch`                                                                                                                               | Impulsa la promoción de dependencias o la limpieza de reclamaciones obsoletas sin iniciar trabajadores; el inicio de trabajadores utiliza el Gateway o el despacho mediante comandos con barra. |

Los estados de las pruebas son resultados comunicados por los trabajadores, no una verificación independiente. Una entrada `passed`
significa que el trabajador informa de que su comando o comprobación se realizó correctamente; los consumidores que necesiten
una puerta de calidad independiente deben inspeccionar el comando, la URL o el artefacto adjuntos y
ejecutar su propio verificador. `workboard_proof` devuelve el `proofId` del nuevo registro. Cuando
`workboard_complete` comunique el estado terminal de esa misma prueba, se debe proporcionar `proofId` para que el
registro pendiente se resuelva en el mismo lugar sin perder su identidad ni su marca temporal. Una prueba que
ya tenga el mismo estado terminal se reutiliza sin cambios. Las pruebas de finalización sin
`proofId` solo permiten anexiones, por lo que un reintento posterior no puede reescribir el historial anterior únicamente porque
su comando o nota sean idénticos.

Las tarjetas reclamadas rechazan las mutaciones realizadas mediante herramientas de agente por otros agentes, salvo que el llamador
posea el token de reclamación devuelto por `workboard_claim`. Cada tarjeta devuelta por una
herramienta de agente o una llamada RPC del Gateway censura `metadata.claim.token` como `[redacted]`
(el propio token solo se devuelve una vez, en el nivel superior y únicamente desde `workboard_claim`),
de modo que los operadores del panel y otros agentes puedan inspeccionar el estado de reclamación sin
ver nunca un token utilizable. La recuperación se realiza mediante
`workboard_promote`/`workboard_reassign`/`workboard_reclaim`, que no
requieren el token.

## Despacho

El despacho es local al Gateway: no genera procesos arbitrarios del sistema operativo. Las sesiones normales
de subagentes de OpenClaw siguen siendo responsables de la ejecución. Una pasada de despacho:

1. Promueve las tarjetas cuyas dependencias están listas.
2. Registra los metadatos de despacho en las tarjetas listas.
3. Bloquea las reclamaciones caducadas o las ejecuciones que han superado el tiempo límite.
4. Marca como candidatas a orquestación las tarjetas de triaje configuradas en el tablero.
5. Reclama un pequeño lote de tarjetas listas e inicia las ejecuciones de trabajadores mediante el
   entorno de ejecución de subagentes del Gateway.

Los trabajadores reciben un contexto acotado de la tarjeta junto con el token de reclamación necesario para enviar Heartbeats,
completar o bloquear la tarjeta mediante las herramientas de Workboard.

Las rutas del espacio de trabajo siguen la autoridad existente del llamador sobre el sistema de archivos. Los clientes del Gateway
con `operator.write` pueden utilizar espacios de trabajo configurados de agentes;
los clientes `operator.admin` pueden utilizar otros checkouts del host. Las herramientas de agente aisladas utilizan
el acceso al espacio de trabajo de su sandbox, mientras que las herramientas no aisladas y limitadas al espacio de trabajo utilizan su
raíz configurada del espacio de trabajo. Workboard registra esa autoridad cuando se asigna un espacio de trabajo
y vuelve a intersectarla con la autoridad del llamador actual durante el despacho,
por lo que una tarjeta persistente no puede ampliar el acceso de un llamador posterior. En las tarjetas antiguas con un
espacio de trabajo explícito del host, pero sin una autoridad registrada, ese espacio de trabajo
debe volver a guardarse antes de un despacho con acceso completo al host; las tarjetas sin una ruta del host adoptan la
autoridad del llamador actual cuando se despachan por primera vez.

El despacho vinculado a un espacio de trabajo solo acepta un directorio o checkout de Git cuando la
raíz de su repositorio coincide exactamente con el espacio de trabajo del agente de destino. Una solicitud de
worktree se limita a ese directorio y se conserva como un espacio de trabajo de directorio, por lo que el
host no materializa el checkout ni ejecuta código de configuración del repositorio. El
trabajador de destino debe utilizar un sandbox de Docker con capacidad de escritura y no compartido para ese
espacio de trabajo exacto, sin ejecución elevada, anulaciones persistentes de ejecución en el host/Node ni
herramientas de Plugin y MCP sin clasificar. Workboard enumera sus herramientas registradas
en lugar de confiar en un prefijo `workboard_*`, y el despacho rechaza un contenedor de Docker
activo cuyo hash de montaje/configuración en ejecución esté obsoleto. El despacho informa de la
política de destino incompatible en lugar de iniciar un trabajador con menos aislamiento.
El despacho con acceso completo al host puede dirigirse a otros checkouts locales y conserva la configuración normal de
worktrees gestionados.

La autoridad del espacio de trabajo no crea un segundo modelo de permisos para el ciclo de vida de las tarjetas.
Los llamadores que pueden modificar las tarjetas de Workboard pueden moverlas manualmente por los mismos
estados en todas las superficies; el acceso de solo lectura al espacio de trabajo únicamente impide el
despacho de trabajadores que necesiten realizar escrituras.

### Selección de trabajadores

Cada pasada inicia **como máximo 3 workers de forma predeterminada**. Las tarjetas listas se ordenan por
prioridad, luego por posición y después por hora de creación. Una pasada inicia solo una tarjeta por
propietario/agente y omite a los propietarios que ya tienen trabajo en ejecución o en revisión en el
tablero. Las tarjetas archivadas, las tarjetas con una asignación activa y las tarjetas que no están en el estado `ready`
nunca se seleccionan para iniciar workers (aun así, pueden verse afectadas por el
lado de datos del despacho: limpieza de asignaciones obsoletas, promoción de dependencias y limpieza
por tiempo de espera).

Las claves de sesión son deterministas por tablero/tarjeta, por lo que los despachos repetidos se dirigen
de nuevo al mismo carril de worker en lugar de crear sesiones no relacionadas:

- Tarjetas asignadas: `agent:<agentId>:subagent:workboard-<boardId>-<cardId>`
- Tarjetas sin asignar: `subagent:workboard-<boardId>-<cardId>` (el Gateway resuelve
  el agente predeterminado configurado)

Si no se puede iniciar un worker después de asignar una tarjeta, Workboard bloquea la
tarjeta, borra la asignación, registra el fallo de inicio de la ejecución y añade una línea al
registro del worker, visible en el panel, el JSON de la CLI, las herramientas del agente y los
diagnósticos de la tarjeta.

### Puntos de entrada

- Acción de despacho del panel
- `openclaw workboard dispatch`
- `/workboard dispatch` en un canal que admita comandos

Los tres utilizan el entorno de ejecución de subagentes del Gateway cuando este está disponible. La
CLI dispone de una alternativa para el operador: si la llamada al Gateway falla con un error de
conexión/no disponibilidad (o un error `unknown method` en Gateways anteriores),
y no se aplica ningún destino explícito `--url`/`--token` ni ningún Gateway remoto
configurado (`OPENCLAW_GATEWAY_URL` o `gateway.mode: remote`), la CLI ejecuta un
despacho exclusivamente de datos sobre el estado SQLite local: puede promover dependencias,
limpiar asignaciones obsoletas y bloquear ejecuciones que hayan superado el tiempo de espera, pero no puede iniciar workers. Los fallos de autenticación,
permisos y validación de un Gateway accesible no se consideran errores de disponibilidad;
se muestran como errores del comando, al igual que cualquier fallo del Gateway
cuando se ha proporcionado un destino explícito `--url`/`--token`.

Los metadatos del tablero pueden establecer `autoDecompose`, `autoDecomposePerDispatch`,
`defaultAssignee` y `orchestratorProfile`. OpenClaw registra esta intención y
la expone en el contexto del worker; la especificación/descomposición efectiva sigue ejecutándose
mediante las herramientas normales de Workboard.

## CLI y comando de barra

```bash
openclaw workboard list [--board <id>] [--status <status>] [--include-archived] [--json]
openclaw workboard create "Fix stale card lifecycle" --priority high --labels bug,workboard
openclaw workboard show <card-id> [--json]
openclaw workboard move <card-id> --status <status> [--json]
openclaw workboard dispatch [--board <id>] [--json]
```

La salida de texto de `list` oculta de forma predeterminada las tarjetas archivadas (`--include-archived`
lo anula); `--json` siempre incluye las tarjetas archivadas, de acuerdo con el contrato de tarjetas completas
utilizado por los scripts existentes. `show` y `move` aceptan un prefijo de id
inequívoco. `list`, `create`, `show` y `move` siempre leen/escriben directamente el estado
local del plugin. Solo `dispatch` llama al Gateway en ejecución, con la alternativa
descrita anteriormente.

Consulte [CLI de Workboard](/es/cli/workboard) para conocer todos los indicadores, la salida JSON, el comportamiento
alternativo del Gateway, la gestión de prefijos de id, las reglas de selección del despacho y la
resolución de problemas.

`/workboard list`, `/workboard show <card-id>`, `/workboard create <title>`,
`/workboard move <card-id> --status <status>` y `/workboard dispatch` reflejan
la CLI. Listar y mostrar son operaciones de lectura disponibles para cualquier remitente de comandos autorizado.
Crear, mover y despachar requieren el estado de propietario en las superficies de chat, o un cliente del Gateway
con `operator.write`/`operator.admin`. Los movimientos manuales del operador utilizan el
mismo comportamiento de anulación de asignaciones que la función de arrastrar y soltar del panel. El acceso al árbol de trabajo
sigue respetando el mismo límite del espacio de trabajo descrito anteriormente.

## Sincronización del ciclo de vida de las sesiones

Las tarjetas pueden enlazarse a una sesión existente del panel o a una creada cuando se
inicia el trabajo desde la tarjeta. Las tarjetas enlazadas muestran en línea el ciclo de vida de la sesión:
en ejecución, obsoleta, enlazada e inactiva, finalizada, fallida o ausente. También se puede capturar una
sesión existente desde la pestaña Sesiones mediante **Añadir a Workboard**; la tarjeta
se enlaza a esa sesión, utiliza como título la etiqueta de la sesión o el mensaje reciente del usuario
y rellena las notas con el mensaje reciente del usuario y la respuesta más reciente del asistente
cuando están disponibles.

Si la sesión enlazada desaparece, la tarjeta permanece enlazada para conservar el contexto y
sigue ofreciendo controles de inicio para reiniciar en una sesión nueva. Si una sesión
enlazada activa deja de informar de actividad reciente, Workboard marca la tarjeta como
`stale` y lo almacena como metadatos hasta que el ciclo de vida lo elimina.

Mientras una tarjeta se encuentra en un estado de trabajo activo, Workboard sigue la sesión enlazada:

| Estado de la sesión enlazada          | Estado de la tarjeta |
| ------------------------------------- | ----------- |
| activa                                | `running`   |
| completada                            | `review`    |
| fallida, terminada, agotada o cancelada | `blocked`   |

**Los estados de revisión manual tienen prioridad.** Mover una tarjeta a `review`, `blocked` o `done`
detiene la sincronización automática de esa tarjeta hasta que se vuelve a mover a `todo` o `running`.

Al iniciar una tarjeta se utilizan sesiones normales del Gateway; Workboard solo almacena los
metadatos y enlaces de la tarjeta. La transcripción de la conversación, la selección del modelo y el
ciclo de vida de la ejecución siguen siendo responsabilidad del sistema de sesiones normal. Utilice **Detener** en una tarjeta
enlazada activa para cancelar la ejecución activa; Workboard marca esa tarjeta como `blocked` para que
permanezca visible y pueda realizarse un seguimiento.

Las tarjetas nuevas pueden partir de plantillas de Workboard (`bugfix`, `docs`, `release`,
`pr_review`, `plugin`). Las plantillas rellenan previamente el título, las notas, las etiquetas y la prioridad;
el id de la plantilla se almacena como metadatos de la tarjeta.

## Flujo de trabajo del panel

1. Abra la pestaña Workboard en la interfaz de control.
2. Cree una tarjeta con título, notas, prioridad, etiquetas, un agente opcional y
   una sesión enlazada opcional, o abra Sesiones y elija **Añadir a Workboard**
   para una sesión existente.
3. Arrastre la tarjeta entre columnas, o enfoque su control compacto de estado y utilice
   el menú o ArrowLeft/ArrowRight. Durante el arrastre, la tarjeta de origen se atenúa y
   las columnas de destino disponibles muestran un contorno.
4. Inicie el trabajo desde la tarjeta para crear o reutilizar una sesión del panel.
5. Abra la sesión enlazada desde la tarjeta mientras trabaja el agente.
6. Permita que la sincronización del ciclo de vida mueva el trabajo en ejecución a `review`/`blocked` y, después,
   mueva manualmente la tarjeta a `done` cuando se acepte.

### Widgets del tablero de sesiones

Workboard incluye dos widgets nativos para los paneles de sesiones (consulte
[Paneles](/es/web/dashboards)). El agente los fija con su herramienta `dashboard`
mediante `content: { kind: "plugin", pluginKind, props }`, y se representan como
una interfaz propia con datos en tiempo real, sin marcos de entorno aislado ni concesiones de capacidades:

- `workboard:card` con `props: { cardId }` muestra una tarjeta con su control
  de estado, prioridad y agente asignado.
- `workboard:mini` con `props: { boardId, limit }` opcional muestra los recuentos por estado
  junto con las principales tarjetas listas/en ejecución, y enlaza con la página completa del tablero.
  Sin `boardId`, agrega todos los tableros; con `boardId`, limita el ámbito a ese
  tablero (las tarjetas creadas sin un id de tablero explícito se encuentran en `default`).

## Diagnósticos

Los diagnósticos se calculan a partir de los metadatos locales de las tarjetas. Las comprobaciones integradas detectan:

| Tipo                        | Condición                                                                      |
| --------------------------- | ------------------------------------------------------------------------------ |
| `stranded_ready`            | Tarjeta `todo`/`backlog`/`ready` asignada que no se ha actualizado durante más de 1 hora.             |
| `running_without_heartbeat` | Tarjeta `running` sin señal de actividad de la asignación ni actualización de ejecución durante más de 20 minutos. |
| `blocked_too_long`          | Tarjeta `blocked` que no se ha actualizado durante más de 24 horas.                                   |
| `repeated_failures`         | El recuento de fallos registrado de la tarjeta alcanza 2 o más.                                |
| `missing_proof`             | Tarjeta `done` sin pruebas, artefactos ni archivos adjuntos.                          |
| `orphaned_session`          | Tarjeta `running` con un `sessionKey` pero sin metadatos `execution`.                |

## Permisos

Los métodos RPC del Gateway se encuentran bajo `workboard.*`:

| Ámbito            | Métodos                                                                                                                                                                                                                                                                                                                                                                            |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operator.read`  | `cards.list`, `cards.export`, `cards.diagnostics`, listar/obtener archivos adjuntos, lecturas de eventos de notificación, `boards.list`, `cards.stats`, `cards.runs`                                                                                                                                                                                                                                       |
| `operator.write` | `cards.diagnostics.refresh`, crear/actualizar/mover/eliminar/comentar/enlazar/linkDependency/prueba/artefacto, añadir/eliminar archivos adjuntos, registro del worker, infracción del protocolo, asignar/señal de actividad/liberar/promover/reasignar/recuperar/completar/bloquear/desbloquear, `cards.dispatch`, `cards.bulk`, archivar, `boards.upsert`/`archive`/`delete`, `cards.specify`/`decompose`, suscribirse/eliminar/avanzar notificaciones |

Ningún método RPC requiere `operator.admin`. Los navegadores conectados con acceso de operador
de solo lectura pueden inspeccionar el tablero, pero no pueden modificar las tarjetas. Un ámbito de administrador
amplía las rutas de host de Workboard aceptadas; no cambia los métodos disponibles.

## Almacenamiento

Workboard almacena datos duraderos en una base de datos SQLite relacional propiedad del plugin,
dentro del directorio de estado de OpenClaw: los tableros, las tarjetas, las etiquetas, los eventos del ciclo de vida,
los intentos de ejecución, los comentarios, los enlaces de dependencias, las pruebas, las referencias de artefactos,
los metadatos y blobs de archivos adjuntos, los diagnósticos, las notificaciones, los registros de workers,
el estado del protocolo y las suscripciones se almacenan en tablas de Workboard (no en
entradas de clave-valor del plugin). La exportación de una tarjeta conserva la narrativa del tablero
sin insertar el contenido de los blobs de archivos adjuntos.

Las instalaciones que utilizaron Workboard en la versión `.28` pueden ejecutar
`openclaw doctor --fix` para migrar los espacios de nombres heredados del estado del plugin incluidos
(`workboard.cards`, `workboard.boards`, `workboard.notify` y, si existe,
`workboard.attachments`) a la base de datos relacional.

## Resolución de problemas

**La pestaña indica que Workboard no está disponible**

```bash
openclaw plugins inspect workboard --runtime --json
```

Si `plugins.allow` está configurado, añada `workboard`. Si `plugins.deny`
contiene `workboard`, elimínelo antes de activar el plugin.

**Las tarjetas no se guardan**

Confirme que la conexión del navegador tenga acceso `operator.write`. Las sesiones de operador
de solo lectura pueden listar tarjetas, pero no pueden crearlas, editarlas, moverlas ni eliminarlas.

**Al iniciar una tarjeta no se abre la sesión esperada**

Compruebe el id de agente y la sesión enlazada de la tarjeta y, después, abra Sesiones o Chat para
inspeccionar el estado real de la ejecución.

**El despacho no inicia un worker**

Confirme que haya al menos una tarjeta `ready` sin una asignación activa:

```bash
openclaw workboard list --status ready
```

Si la CLI informa de un despacho solo de datos, inicia o reinicia el Gateway y
vuelve a intentarlo: el despacho solo de datos actualiza el estado local del tablero, pero no puede iniciar
ejecuciones de trabajadores de subagentes. Las tarjetas también pueden omitirse cuando otra tarjeta del
mismo propietario o agente ya se está ejecutando o está esperando revisión; completa,
bloquea o libera ese trabajo activo antes de despachar más trabajo para el mismo
propietario.

## Relacionado

- [Interfaz de control](/es/web/control-ui)
- [CLI del tablero de trabajo](/es/cli/workboard)
- [Plugins](/es/tools/plugin)
- [Gestionar plugins](/es/plugins/manage-plugins)
- [Sesiones](/es/concepts/session)
