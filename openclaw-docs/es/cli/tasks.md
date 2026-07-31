---
read_when:
    - Se desea inspeccionar, auditar o cancelar registros de tareas en segundo plano
    - Está documentando los comandos de TaskFlow en `openclaw tasks flow`
summary: Referencia de la CLI para `openclaw tasks` (registro de tareas en segundo plano y estado de Task Flow)
title: '`openclaw tasks`'
x-i18n:
    generated_at: "2026-07-26T05:35:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b03a4aa9fab12b6e5773259a76a1e89fd6e6398c73e5b0533a31e5e3a3894f9c
    source_path: cli/tasks.md
    workflow: 16
---

Inspecciona las tareas duraderas en segundo plano y el estado de Task Flow. Sin un subcomando,
`openclaw tasks` equivale a `openclaw tasks list`.

Consulte [Tareas en segundo plano](/es/automation/tasks) para conocer el ciclo de vida y el modelo
de entrega, y su sección `tasks audit` para obtener descripciones completas de los hallazgos.

## Uso

```bash
openclaw tasks
openclaw tasks list
openclaw tasks list --runtime acp
openclaw tasks list --status running
openclaw tasks show <lookup>
openclaw tasks notify <lookup> state_changes
openclaw tasks cancel <lookup>
openclaw tasks audit
openclaw tasks maintenance
openclaw tasks maintenance --apply
openclaw tasks flow list
openclaw tasks flow show <lookup>
openclaw tasks flow cancel <lookup>
```

## Opciones raíz

| Indicador          | Descripción                                                                                        |
| ------------------ | -------------------------------------------------------------------------------------------------- |
| `--json`           | Genera JSON.                                                                                       |
| `--runtime <name>` | Filtra por tipo: `subagent`, `acp`, `cron` o `cli`.                                               |
| `--status <name>`  | Filtra por estado: `queued`, `running`, `succeeded`, `failed`, `timed_out`, `cancelled` o `lost`. |

## Subcomandos

### `list`

```bash
openclaw tasks list [--runtime <name>] [--status <name>] [--json]
```

Enumera las tareas en segundo plano registradas, comenzando por las más recientes.

### `show`

```bash
openclaw tasks show <lookup> [--json]
```

Muestra una tarea por su ID de tarea, ID de ejecución o clave de sesión.

### `notify`

```bash
openclaw tasks notify <lookup> <done_only|state_changes|silent>
```

Cambia la política de notificaciones de una tarea en ejecución.

### `cancel`

```bash
openclaw tasks cancel <lookup>
```

Cancela una tarea en segundo plano en ejecución.

### `audit`

```bash
openclaw tasks audit [--severity <warn|error>] [--code <name>] [--limit <n>] [--json]
```

Expone registros de tareas y Task Flow obsoletos, perdidos, con errores de entrega
o que presentan otras incoherencias. Las tareas perdidas que se conservan hasta `cleanupAfter` son advertencias;
las tareas perdidas caducadas o sin marca son errores.

`--code` acepta códigos de tareas (`stale_queued`, `stale_running`, `lost`,
`delivery_failed`, `missing_cleanup`, `inconsistent_timestamps`) y códigos de Task
Flow (`restore_failed`, `stale_waiting`, `stale_blocked`,
`cancel_stuck`, `missing_linked_tasks`, `blocked_task_missing`). Consulte
[Tareas en segundo plano](/es/automation/tasks) para obtener detalles sobre la gravedad y el desencadenante de cada
código.

### `maintenance`

```bash
openclaw tasks maintenance [--apply] [--json]
```

Previsualiza o aplica la conciliación de tareas y Task Flow, el marcado de limpieza,
la depuración y la limpieza del registro de sesiones de ejecuciones de Cron obsoletas.

Para las tareas de Cron, la conciliación utiliza los registros persistentes de ejecución y el estado del trabajo antes de
marcar una tarea activa antigua como `lost`, de modo que las ejecuciones de Cron completadas no se conviertan en
falsos errores de auditoría solo porque el estado del entorno de ejecución en memoria del Gateway haya desaparecido.
La auditoría sin conexión de la CLI no es una fuente autoritativa para el conjunto de trabajos de Cron activos
local al proceso del Gateway. Las tareas de la CLI con un ID de ejecución o un ID de origen se marcan como `lost` cuando
su contexto de ejecución activo del Gateway ha desaparecido, aunque aún quede una fila antigua de
sesión secundaria.

Cuando se aplica, el mantenimiento también depura las filas del registro de sesiones `cron:<jobId>:run:<uuid>`
con más de 7 días de antigüedad, al tiempo que conserva los trabajos de Cron en ejecución
y no modifica las filas de sesiones que no pertenecen a Cron.

### `flow`

```bash
openclaw tasks flow list [--status <name>] [--json]
openclaw tasks flow show <lookup> [--json]
openclaw tasks flow cancel <lookup>
```

Inspecciona o cancela el estado duradero de Task Flow en el libro mayor de tareas.
`flow list --status` acepta `queued`, `running`, `waiting`, `blocked`,
`succeeded`, `failed`, `cancelled` o `lost`.

## Relacionado

- [Referencia de la CLI](/es/cli)
- [Tareas en segundo plano](/es/automation/tasks)
