---
read_when:
    - Quieres tareas programadas y activaciones.
    - Está depurando la ejecución y los registros de Cron
summary: Referencia de la CLI para `openclaw cron` (programar y ejecutar trabajos en segundo plano)
title: Cron
x-i18n:
    generated_at: "2026-07-26T04:33:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5989a7558f4ae2f046480b6a52e3fa296c95d47b14b11c5bad709fea4af6af3e
    source_path: cli/cron.md
    workflow: 16
---

# `openclaw cron`

Gestiona los trabajos Cron del programador del Gateway.

<Tip>
Ejecuta `openclaw cron --help` para ver todos los comandos disponibles. Consulta [Trabajos Cron](/es/automation/cron-jobs) para acceder a la guía conceptual.
</Tip>

<Note>
Todas las modificaciones de Cron (`add`/`create`, `update`/`edit`, `remove`, `run`) requieren `operator.admin`. Las ejecuciones con carga útil de comando se realizan directamente en el proceso del Gateway, no como una llamada de herramienta `tools.exec` del agente; `tools.exec.*` y las aprobaciones de ejecución siguen rigiendo las herramientas de ejecución visibles para el modelo.
</Note>

## Crear trabajos rápidamente

`openclaw cron create` es un alias de `openclaw cron add`. Para trabajos nuevos, coloca primero la programación y después el prompt:

```bash
openclaw cron create "0 7 * * *" \
  "Resume las actualizaciones de la noche." \
  --name "Resumen matutino" \
  --agent ops
```

Usa `--webhook <url>` cuando el trabajo deba enviar mediante POST la carga útil finalizada, en lugar de entregarla a un destino de chat:

```bash
openclaw cron create "0 18 * * 1-5" \
  "Resume los despliegues de hoy como JSON." \
  --name "Resumen de despliegues" \
  --webhook "https://example.invalid/openclaw/cron"
```

Usa `--command` para trabajos deterministas de estilo shell que se ejecuten dentro de Cron de OpenClaw sin iniciar una ejecución aislada de agente/modelo:

```bash
openclaw cron create "*/15 * * * *" \
  --name "Sondeo de profundidad de la cola" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>` almacena `argv: ["sh", "-lc", <shell>]`. Usa `--command-argv '["node","scripts/report.mjs"]'` para una ejecución exacta de argv. Los trabajos de comando capturan stdout/stderr, registran el historial normal de Cron y enrutan la salida mediante los mismos modos de entrega `announce`, `webhook` o `none` que los trabajos aislados. Se suprime un comando que solo muestra `NO_REPLY`.

## Sesiones

`--session` acepta `main`, `isolated`, `current` o `session:<id>`.

<AccordionGroup>
  <Accordion title="Claves de sesión">
    - `main` se vincula a la sesión principal del agente.
    - `isolated` crea una transcripción y un id. de sesión nuevos para cada ejecución.
    - `current` se vincula a la sesión activa en el momento de la creación.
    - `session:<id>` se fija a una clave de sesión persistente explícita.

  </Accordion>
  <Accordion title="Semántica de las sesiones aisladas">
    Las ejecuciones aisladas restablecen el contexto de conversación del entorno. Para la nueva ejecución, se restablecen el enrutamiento de canales y grupos, la política de envío/puesta en cola, la elevación, el origen y la vinculación del entorno de ejecución de ACP. Las preferencias seguras y las anulaciones explícitas de modelo o autenticación seleccionadas por el usuario pueden conservarse entre ejecuciones.
  </Accordion>
</AccordionGroup>

## Entrega

`openclaw cron list` y `openclaw cron show <job-id>` muestran una vista previa de la ruta de entrega resuelta. Para `channel: "last"`, la vista previa indica si la ruta se resolvió a partir de la sesión principal o actual, o si fallará de forma cerrada.

Los destinos con prefijo de proveedor pueden desambiguar los canales de anuncio no resueltos. Por ejemplo, `to: "telegram:123"` selecciona Telegram cuando se omite `delivery.channel` o se usa `last`. Solo los prefijos anunciados por el Plugin cargado actúan como selectores de proveedor. Si `delivery.channel` es explícito, el prefijo debe coincidir con ese canal; se rechaza `channel: "whatsapp"` con `to: "telegram:123"`. Los prefijos de servicio como `imessage:` y `sms:` siguen siendo sintaxis de destino propiedad del canal.

<Note>
Los trabajos aislados `cron add` usan de forma predeterminada la entrega `--announce`. Usa `--no-deliver` para mantener la salida interna. `--deliver` se conserva como alias obsoleto de `--announce`.
</Note>

### Propiedad de la entrega

La entrega de chat de Cron aislado se comparte entre el agente y el ejecutor:

- El agente puede enviar directamente mediante la herramienta `message` cuando haya una ruta de chat disponible.
- `announce` entrega como alternativa la respuesta final solo cuando el agente no la envía directamente al destino resuelto.
- `webhook` envía la carga útil finalizada a una URL.
- `none` desactiva la entrega alternativa del ejecutor.

Usa `cron add|create --webhook <url>` o `cron edit <job-id> --webhook <url>` para establecer la entrega mediante Webhook. No combines `--webhook` con opciones de entrega por chat como `--announce`, `--no-deliver`, `--channel`, `--to`, `--thread-id` o `--account`.

`cron edit <job-id>` puede eliminar campos individuales de enrutamiento de entrega con `--clear-channel`, `--clear-to`, `--clear-thread-id` y `--clear-account` (cada uno se rechaza cuando se combina con su opción de establecimiento correspondiente). A diferencia de `--no-deliver`, que solo desactiva la entrega alternativa del ejecutor, estos eliminan el campo almacenado para que el trabajo vuelva a resolver esa parte de su ruta a partir de los valores predeterminados.

`--announce` es la entrega alternativa del ejecutor para la respuesta final. `--no-deliver` desactiva esa alternativa, pero no elimina la herramienta `message` del agente cuando hay una ruta de chat disponible.

Los recordatorios creados desde un chat activo conservan el destino de entrega del chat en curso para la entrega alternativa de anuncios. Las claves de sesión internas pueden estar en minúsculas; no las uses como fuente fiable para los identificadores de proveedor que distinguen entre mayúsculas y minúsculas, como los identificadores de sala de Matrix.

### Entrega de fallos

Las notificaciones de fallos se resuelven en este orden:

1. `delivery.failureDestination` en el trabajo.
2. `cron.failureDestination` global.
3. El destino principal de anuncio del trabajo (cuando ninguna de las opciones anteriores se resuelve en un destino concreto).

<Note>
Los trabajos de la sesión principal solo pueden usar `delivery.failureDestination` cuando el modo de entrega principal es `webhook`. Los trabajos aislados lo aceptan en todos los modos.
</Note>

Las ejecuciones aisladas de Cron tratan los fallos del agente en el nivel de ejecución como errores del trabajo, incluso cuando no se produce ninguna carga útil de respuesta, por lo que los fallos del modelo/proveedor siguen incrementando los contadores de errores y activando las notificaciones de fallos.

Los trabajos de comando de Cron no inician un turno aislado del agente. Un código de salida cero registra `ok`; una salida distinta de cero, una señal, un tiempo de espera agotado o un tiempo de espera sin salida agotado registra `error` y puede activar la misma ruta de notificación de fallos.

Si una ejecución aislada agota el tiempo de espera antes de la primera solicitud al modelo, `openclaw cron show` y `openclaw cron runs` incluyen un error específico de la fase, como `setup timed out before runner start`, o un mensaje de bloqueo que indica la última fase de inicio conocida (por ejemplo, `context-engine`). Para los proveedores basados en CLI, el supervisor previo al modelo permanece activo hasta que comienza el turno de la CLI externa, de modo que los bloqueos en la búsqueda de sesiones, hooks, autenticación, prompts y configuración de la CLI se notifican como fallos de Cron previos al modelo.

## Programación

### Trabajos de una sola ejecución

`--at <datetime>` programa una ejecución única. Las fechas y horas sin desplazamiento se tratan como UTC, salvo que también se proporcione `--tz <iana>`, que interpreta la hora local en la zona horaria indicada.

<Note>
Los trabajos de una sola ejecución se eliminan de forma predeterminada tras finalizar correctamente. Usa `--keep-after-run` para conservarlos.
</Note>

### Trabajos recurrentes

Los trabajos recurrentes usan un retroceso exponencial de reintentos tras errores consecutivos: 30s, 1m, 5m, 15m, 60m. La programación vuelve a la normalidad tras la siguiente ejecución correcta.

Las ejecuciones omitidas se registran por separado de los errores de ejecución. No afectan al retroceso de reintentos, pero `openclaw cron edit <job-id> --failure-alert-include-skipped` puede hacer que las alertas de fallos incluyan notificaciones de ejecuciones omitidas repetidas.

Para los trabajos aislados dirigidos a un proveedor de modelos local configurado (URL base en loopback, una red privada o `.local`), Cron realiza una comprobación preliminar ligera del proveedor antes de iniciar el turno del agente: los proveedores `api: "ollama"` se comprueban en `/api/tags`; los demás proveedores locales compatibles con OpenAI (`api: "openai-completions"`, por ejemplo, vLLM, SGLang, LM Studio) se comprueban en `/models`. Si no se puede acceder al endpoint, la ejecución se registra como `skipped` y se reintenta en una programación posterior; el resultado de accesibilidad se almacena en caché por endpoint durante 5 minutos para que muchos trabajos dirigidos al mismo servidor local no lo saturen con comprobaciones repetidas.

Los trabajos de Cron, el estado pendiente del entorno de ejecución y el historial de ejecuciones residen en la base de datos de estado SQLite compartida. Los archivos antiguos `jobs.json`, `<name>-state.json` y `runs/*.jsonl` se importan una vez y se renombran con el sufijo `.migrated`. Tras la importación, edita las programaciones con `openclaw cron add|edit|remove` en lugar de editar archivos JSON.

### Ejecuciones manuales

`openclaw cron run <job-id>` fuerza la ejecución de forma predeterminada y devuelve el resultado en cuanto la ejecución manual queda en cola. Las respuestas correctas incluyen `{ ok: true, enqueued: true, runId }`. Usa el `runId` devuelto para consultar el resultado posteriormente:

```bash
openclaw cron run <job-id>
openclaw cron runs --id <job-id> --run-id <run-id>
```

Añade `--wait` cuando un script deba bloquearse hasta que esa ejecución exacta en cola registre un estado terminal:

```bash
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
```

Con `--wait`, la CLI sigue llamando primero a `cron.run` y, después, consulta periódicamente `cron.runs` para el `runId` devuelto. El comando solo finaliza con `0` cuando la ejecución termina con el estado `ok`. Finaliza con un valor distinto de cero cuando la ejecución termina con `error` o `skipped`, cuando la respuesta del Gateway no incluye un `runId` o cuando vence `--wait-timeout` (valor predeterminado: `10m`, con consultas cada `2s` de forma predeterminada). `--poll-interval` debe ser mayor que cero.

<Note>
Usa `--due` cuando se quiera que el comando manual solo se ejecute si el trabajo debe ejecutarse en ese momento. Si `--due --wait` no pone una ejecución en cola, el comando devuelve la respuesta normal de no ejecución en lugar de iniciar las consultas.
</Note>

## Modelos

`cron add|edit --model <ref>` selecciona un modelo permitido para el trabajo. `cron add|edit --fallbacks <list>` establece modelos alternativos por trabajo, por ejemplo, `--fallbacks openrouter/gpt-4.1-mini,openai/gpt-5`; proporciona `--fallbacks ""` para una ejecución estricta sin alternativas. `cron edit <job-id> --clear-fallbacks` elimina la anulación de alternativas por trabajo. `cron edit <job-id> --clear-model` elimina la anulación del modelo por trabajo para que este siga la precedencia normal de selección de modelos de Cron (una anulación almacenada de la sesión de Cron, si existe; de lo contrario, el modelo del agente/predeterminado); no se puede combinar con `--model`. `cron add|edit --thinking <level>` establece una anulación de razonamiento por trabajo; `cron edit <job-id> --clear-thinking` la elimina para que el trabajo siga la precedencia normal de razonamiento de Cron y no se puede combinar con `--thinking`.

<Warning>
Si el modelo no está permitido o no puede resolverse, Cron hace que la ejecución falle con un error de validación explícito en lugar de recurrir a la selección del agente del trabajo o del modelo predeterminado.
</Warning>

El `--model` de Cron es el **principal del trabajo**, no una anulación `/model` de la sesión de chat. Esto significa lo siguiente:

- Las alternativas de modelo configuradas siguen aplicándose cuando falla el modelo seleccionado para el trabajo.
- La carga útil `fallbacks` por trabajo sustituye la lista de alternativas configurada cuando está presente.
- Una lista vacía de alternativas por trabajo (`--fallbacks ""` o `fallbacks: []` en la carga útil/API del trabajo) hace que la ejecución de Cron sea estricta.
- Cuando un trabajo tiene `--model`, pero no hay ninguna lista de alternativas configurada, OpenClaw proporciona una anulación explícita de alternativas vacía para que el modelo principal del agente no se añada como destino de reintento oculto.
- Las comprobaciones preliminares del proveedor local recorren las alternativas configuradas antes de marcar una ejecución de Cron como `skipped`.

`openclaw doctor` informa de los trabajos que ya tienen establecido `payload.model`, incluidos los recuentos por espacio de nombres del proveedor y las discrepancias con `agents.defaults.model`. Usa esa comprobación cuando el comportamiento de autenticación, proveedor o facturación sea distinto entre el chat en vivo y los trabajos programados.

### Precedencia de modelos de Cron aislado

Cron aislado resuelve el modelo activo en este orden:

1. Anulación del hook de Gmail.
2. `--model` por trabajo.
3. Anulación almacenada del modelo de la sesión de Cron (cuando el usuario ha seleccionado una).
4. Selección del modelo del agente o predeterminado.

### Modo rápido

El modo rápido de Cron aislado sigue la selección de modelo activa resuelta. La configuración del modelo `params.fastMode` se aplica de forma predeterminada, pero una anulación almacenada en la sesión `fastMode` sigue teniendo prioridad sobre la configuración. Cuando el modo resuelto es `auto`, el límite usa el valor `params.fastAutoOnSeconds` del modelo seleccionado, con 60 segundos como valor predeterminado.

### Reintentos al cambiar el modelo activo

Si una ejecución aislada genera `LiveSessionModelSwitchError`, Cron conserva el proveedor y el modelo seleccionados tras el cambio (y la anulación del perfil de autenticación seleccionado tras el cambio, si existe) para la ejecución activa antes de volver a intentarlo. El bucle de reintentos externo se limita a dos reintentos de cambio después del intento inicial y luego se interrumpe en lugar de continuar indefinidamente.

## Resultado de la ejecución y denegaciones

### Supresión de confirmaciones obsoletas

Los turnos de Cron aislados suprimen las respuestas obsoletas que solo contienen una confirmación. Si el primer resultado es solo una actualización de estado provisional y ninguna ejecución de un subagente descendiente es responsable de la respuesta final, Cron vuelve a solicitar una vez el resultado real antes de entregarlo.

### Supresión del token de silencio

Si una ejecución de Cron aislada devuelve únicamente el token de silencio (`NO_REPLY` o `no_reply`), Cron suprime tanto la entrega saliente directa como la ruta alternativa de resumen en cola, por lo que no se publica nada en el chat.

### Denegaciones estructuradas

Las ejecuciones de Cron aisladas usan los metadatos estructurados de denegación de ejecución de la ejecución integrada (errores fatales de la herramienta de ejecución con código `SYSTEM_RUN_DENIED` o `INVALID_REQUEST`) como señal de denegación autoritativa. También admiten contenedores `UNAVAILABLE` del host del Node en torno a un error estructurado anidado que contenga uno de esos códigos.

Cron no clasifica como denegaciones el texto del resultado final ni las frases de rechazo que parezcan solicitar aprobación, salvo que la ejecución integrada también proporcione metadatos estructurados de denegación, por lo que el texto normal del asistente no se trata como un comando bloqueado.

`cron list` y el historial de ejecuciones muestran el motivo de la denegación en lugar de informar de un comando bloqueado como `ok`.

## Retención

Comportamiento de retención:

- `cron.sessionRetention` (valor predeterminado: `24h`; use `false` para desactivarlo) elimina las sesiones de ejecuciones aisladas completadas.
- El historial de ejecuciones conserva las 2000 filas terminales más recientes por tarea de Cron. Las filas perdidas mantienen el periodo estándar de limpieza de tareas perdidas de 24 horas.

## Migración de tareas antiguas

<Note>
Si hay tareas de Cron anteriores al formato actual de entrega y almacenamiento, ejecute `openclaw doctor --fix`. Doctor normaliza los campos de Cron antiguos (`jobId`, `schedule.cron`, los campos de entrega de nivel superior, incluido el antiguo `threadId`, y los alias de entrega `provider` de la carga útil) y migra las tareas alternativas de Webhook `notify: true` del valor sin procesar retirado `cron.webhook` a una entrega explícita mediante Webhook antes de eliminar esa clave de configuración. Las tareas que ya envían anuncios a un chat conservan esa entrega y reciben un destino de Webhook de finalización. Sin un Webhook antiguo, el marcador inerte de nivel superior `notify` se elimina de las tareas sin destino de migración (la entrega existente se conserva sin cambios), por lo que `doctor --fix` deja de emitir advertencias repetidas sobre ellas.
</Note>

## Ediciones habituales

Actualice la configuración de entrega sin cambiar el mensaje:

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "123456789"
```

Desactive la entrega de una tarea aislada:

```bash
openclaw cron edit <job-id> --no-deliver
```

Active el contexto ligero de arranque para una tarea aislada:

```bash
openclaw cron edit <job-id> --light-context
```

Envíe un anuncio a un canal específico:

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
```

Envíe un anuncio a un tema de foro de Telegram:

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "-1001234567890" --thread-id 42
```

Cree una tarea aislada con contexto ligero de arranque:

```bash
openclaw cron create "0 7 * * *" \
  "Resuma las actualizaciones de la noche." \
  --name "Resumen matutino ligero" \
  --session isolated \
  --light-context \
  --no-deliver
```

`--light-context` solo se aplica a las tareas de turno de agente aisladas. En las ejecuciones de Cron, el modo ligero mantiene vacío el contexto de arranque en lugar de insertar el conjunto completo de arranque del espacio de trabajo.

Cree una tarea de comando con argv, cwd, env y stdin exactos, y límites de salida:

```bash
openclaw cron create "*/30 * * * *" \
  --name "Exportación de posición" \
  --command-argv '["node","scripts/export-position.mjs"]' \
  --command-cwd "/srv/app" \
  --command-env "NODE_ENV=production" \
  --command-input '{"mode":"summary"}' \
  --timeout-seconds 120 \
  --no-output-timeout-seconds 30 \
  --output-max-bytes 65536 \
  --webhook "https://example.invalid/openclaw/cron"
```

## Comandos habituales de administración

Ejecución manual e inspección:

```bash
openclaw cron list
openclaw cron list --agent ops
openclaw cron get <job-id>
openclaw cron show <job-id>
openclaw cron run <job-id>
openclaw cron run <job-id> --due
openclaw cron run <job-id> --wait --wait-timeout 10m
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
openclaw cron runs --id <job-id> --limit 50
openclaw cron runs --id <job-id> --run-id <run-id>
```

`openclaw cron list` muestra de forma predeterminada las tareas activadas. Pase `--all` para incluir las tareas desactivadas o `--agent <id>` para mostrar solo las tareas cuyo identificador normalizado efectivo de agente coincida; las tareas sin un identificador de agente almacenado se consideran asignadas al agente predeterminado configurado.

`openclaw cron get <job-id>` devuelve directamente el JSON almacenado de la tarea. Use `cron show <job-id>` cuando se necesite la vista legible para personas con una vista previa de la ruta de entrega.

`cron list --json` y `cron show <job-id> --json` incluyen un campo de nivel superior `status` en cada tarea, calculado a partir de `enabled`, `state.runningAtMs` y `state.lastRunStatus`. Valores: `disabled`, `running`, `ok`, `error`, `skipped` o `idle`. El estado JSON se mantiene canónico y sin elementos decorativos para que las herramientas externas puedan leer el estado de la tarea sin volver a derivarlo; el resultado legible para personas puede acompañar los estados `error` repetidos con un recuento de fallos.

Las entradas `cron runs` incluyen diagnósticos de entrega con el destino de Cron previsto, el destino resuelto, los envíos de la herramienta de mensajes, el uso de la alternativa y el estado de entrega.

Datos temporales privados por tarea (listas de comprobación de Heartbeat y contexto similar del monitor):

```bash
openclaw cron scratch <job-id>                  # mostrar el contenido temporal actual
openclaw cron scratch <job-id> --json           # datos temporales y metadatos de revisión
openclaw cron scratch <job-id> --set "text"     # sustituir los datos temporales por el texto exacto
openclaw cron scratch <job-id> --file notes.md  # sustituir los datos temporales desde un archivo (- para stdin)
openclaw cron scratch <job-id> --unset          # eliminar la fila de datos temporales
```

Los datos temporales se almacenan en la base de datos de estado compartida, tienen un límite de 256 KiB y nunca se incluyen en la salida de `cron list`/`cron get`/`cron runs`. Las escrituras están protegidas mediante comparación e intercambio con respecto a la revisión leída al iniciar el comando; pase `--expected-revision <n>` para fijar en su lugar una revisión explícita. Consulte [Heartbeat](/es/gateway/heartbeat#monitor-scratch-optional) para saber cómo usan estos datos temporales los monitores de Heartbeat.

Cambio de destino del agente y de la sesión:

```bash
openclaw cron edit <job-id> --agent ops
openclaw cron edit <job-id> --clear-agent
openclaw cron edit <job-id> --session current
openclaw cron edit <job-id> --session "session:daily-brief"
```

`openclaw cron add` emite una advertencia cuando se omite `--agent` en las tareas de turno de agente y recurre al agente predeterminado (`main`). Pase `--agent <id>` en el momento de la creación para fijar un agente específico.

Ajustes de entrega:

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
openclaw cron edit <job-id> --webhook "https://example.invalid/openclaw/cron"
openclaw cron edit <job-id> --best-effort-deliver
openclaw cron edit <job-id> --no-best-effort-deliver
openclaw cron edit <job-id> --no-deliver
```

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
- [Tareas programadas](/es/automation/cron-jobs)
