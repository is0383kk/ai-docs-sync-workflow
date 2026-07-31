---
read_when:
    - Programación de trabajos en segundo plano o activaciones
    - Conexión de activadores externos (webhooks, Gmail) con OpenClaw
    - Decidir entre Heartbeat y Cron para tareas programadas
sidebarTitle: Scheduled tasks
summary: Trabajos programados, webhooks y activadores de Gmail PubSub para el planificador del Gateway
title: Tareas programadas
x-i18n:
    generated_at: "2026-07-26T05:30:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dd889cf8e45196eda3ec7c2af930abcb2cc2bae8bad2dbdcaf3cd521a9e884b2
    source_path: automation/cron-jobs.md
    workflow: 16
---

Cron es el programador integrado del Gateway. Conserva los trabajos, activa al agente en el momento adecuado y puede entregar la salida a un canal de chat, un Webhook o a ningún destino.

## Inicio rápido

<Steps>
  <Step title="Añadir un recordatorio único">
    ```bash
    openclaw cron create "2027-02-01T16:00:00Z" \
      --name "Recordatorio" \
      --session main \
      --system-event "Recordatorio: revisar el borrador de la documentación de cron" \
      --wake now \
      --delete-after-run
    ```
  </Step>
  <Step title="Comprobar los trabajos">
    ```bash
    openclaw cron list
    openclaw cron get <job-id>
    openclaw cron show <job-id>
    ```
  </Step>
  <Step title="Consultar el historial de ejecuciones">
    ```bash
    openclaw cron runs --id <job-id>
    ```
  </Step>
</Steps>

## Cómo funciona cron

- Cron se ejecuta **dentro del proceso del Gateway**, no dentro del modelo. El Gateway debe estar en ejecución para que se activen las programaciones.
- Las definiciones de trabajos, el estado de ejecución y el historial de ejecuciones se conservan en la base de datos SQLite de estado compartido de OpenClaw, por lo que los reinicios no hacen que se pierdan las programaciones.
- Cada ejecución de cron crea un registro de [tarea en segundo plano](/es/automation/tasks).
- Los trabajos únicos (`--at`) se eliminan automáticamente después de completarse correctamente de forma predeterminada; pase `--keep-after-run` para conservarlos.
- Presupuesto de tiempo real por ejecución: `--timeout-seconds` cuando se establece. De lo contrario, los trabajos de turno del agente aislados o desacoplados están limitados por el supervisor propio de cron de 60 minutos antes de que pudiera aplicarse el tiempo de espera del turno del agente subyacente (`agents.defaults.timeoutSeconds`, 48 horas de forma predeterminada); los trabajos de comandos tienen un valor predeterminado de 10 minutos y las cargas útiles de scripts, de 5 minutos.
- Al iniciarse el Gateway, los trabajos de turno del agente aislados y vencidos se reprograman en lugar de reproducirse inmediatamente, lo que mantiene el trabajo de arranque del modelo y las herramientas fuera del intervalo de conexión del canal.
- Si ejecuta `openclaw agent` desde el cron del sistema u otro programador externo, envuélvalo con un mecanismo de escalamiento que fuerce la terminación, aunque la CLI ya gestione `SIGTERM`/`SIGINT`. Las ejecuciones respaldadas por el Gateway solicitan al Gateway que cancele las ejecuciones aceptadas; las ejecuciones `--local` reciben la misma señal de cancelación. Para `timeout` de GNU, se recomienda `timeout -k 60 600 openclaw agent ...` en lugar de `timeout 600 ...` sin más: el valor `-k` actúa como último recurso si el proceso no puede finalizar a tiempo. Para unidades systemd, use una señal de detención `SIGTERM` con un periodo de gracia (`TimeoutStopSec`) antes de la terminación definitiva. Si se reutiliza un `--run-id` mientras la ejecución original del Gateway sigue activa, el duplicado se informa como en curso en lugar de iniciar una segunda ejecución.

<AccordionGroup>
  <Accordion title="Refuerzo de ejecuciones aisladas">
    - Al completarse, las ejecuciones aisladas intentan cerrar las pestañas y los procesos del navegador supervisados de su sesión `cron:<jobId>`, y descartan todas las instancias empaquetadas del entorno de ejecución de MCP creadas para el trabajo mediante la misma ruta compartida de desmontaje que utilizan las ejecuciones de la sesión principal y las sesiones personalizadas. Los fallos de limpieza se ignoran para que siga prevaleciendo el resultado de cron.
    - Las ejecuciones aisladas con el permiso limitado de autolimpieza de cron pueden consultar el estado del programador, una lista filtrada que contiene únicamente su propio trabajo y el historial de ejecuciones de ese trabajo, y solo pueden eliminar su propio trabajo.
    - Las ejecuciones aisladas se protegen frente a respuestas de confirmación obsoletas: si el primer resultado es únicamente una actualización provisional del estado (`on it`, `pulling everything together` e indicios similares) y ningún subagente descendiente sigue siendo responsable de la respuesta final, OpenClaw vuelve a solicitar una vez el resultado real antes de entregarlo.
    - Se reconocen los metadatos estructurados de denegación de ejecución (incluidos los envoltorios `UNAVAILABLE` del host Node cuyo error anidado comienza por `SYSTEM_RUN_DENIED` o `INVALID_REQUEST`) para que un comando bloqueado no se informe como una ejecución correcta, mientras que el texto normal del asistente no se confunde con una denegación.
    - Los fallos del agente en el nivel de ejecución cuentan como errores del trabajo incluso si no hay una carga útil de respuesta, por lo que los fallos del modelo o proveedor incrementan los contadores de errores y activan notificaciones de fallo en lugar de marcar el trabajo como completado correctamente.
    - Cuando un trabajo alcanza `timeoutSeconds`, cron cancela la ejecución y le concede un breve periodo de limpieza. Si no finaliza, la limpieza gestionada por el Gateway libera por la fuerza la propiedad de la sesión de esa ejecución antes de que cron registre el tiempo de espera agotado, para que el trabajo de chat en cola no quede bloqueado detrás de una sesión de procesamiento obsoleta.
    - Los bloqueos durante la configuración o el inicio reciben un tiempo de espera específico de la fase (por ejemplo, `cron: isolated agent setup timed out before runner start` o `cron: isolated agent run stalled before execution start (last phase: context-engine)`). Estos supervisores abarcan los proveedores integrados y los respaldados por la CLI incluso antes de que se inicie su proceso de CLI externo, y tienen un límite independiente de los valores prolongados de `timeoutSeconds`, para que los fallos de arranque en frío, autenticación o contexto se manifiesten rápidamente.

  </Accordion>
  <Accordion title="Conciliación de tareas">
    La conciliación de tareas de cron depende primero del entorno de ejecución y, en segundo lugar, del historial persistente: una tarea de cron activa permanece activa mientras el entorno de ejecución de cron siga registrando ese trabajo como en ejecución, aunque todavía exista una fila antigua de una sesión secundaria. Cuando el entorno de ejecución deja de ser propietario del trabajo y vence un periodo de gracia de 5 minutos, las comprobaciones de mantenimiento consultan los registros de ejecución persistentes y el estado del trabajo para la ejecución `cron:<jobId>:<startedAt>` correspondiente. Un resultado terminal allí finaliza el libro de tareas; de lo contrario, el mantenimiento gestionado por el Gateway puede marcar la tarea como `lost`. La auditoría sin conexión de la CLI puede recuperar información del historial persistente, pero que su propio conjunto de trabajos activos en el proceso esté vacío no demuestra que haya desaparecido una ejecución gestionada por el Gateway.
  </Accordion>
</AccordionGroup>

## Tipos de programación

| Tipo      | Indicador de CLI   | Descripción                                                                                              |
| --------- | ------------------ | -------------------------------------------------------------------------------------------------------- |
| `at`      | `--at`             | Marca de tiempo única (ISO 8601 o relativa, como `20m`)                                                     |
| `every`   | `--every`          | Intervalo fijo (`10m`, `1h`, `1d`)                                                                       |
| `cron`    | `--cron`           | Expresión cron de 5 o 6 campos con `--tz` opcional                                                  |
| `on-exit` | `--on-exit`        | Se activa una vez cuando finaliza un comando supervisado (desencadenador de eventos; sobrevive al desmontaje del turno; `--on-exit-cwd` opcional) |
| `stream`  | `--stream-command` | Se activa a partir de lotes de líneas producidos por un comando supervisado de larga duración                                      |

Las marcas de tiempo sin zona horaria se consideran UTC. Añada `--tz America/New_York` para interpretar una fecha y hora `--at` sin desplazamiento, o para evaluar una expresión cron, en esa zona horaria de IANA. Las expresiones cron sin `--tz` usan la zona horaria del host del Gateway. `--tz` no es válido con `--every` ni `--on-exit`.

Las expresiones recurrentes al inicio de la hora (minuto `0` con un campo de hora comodín) se escalonan automáticamente hasta 5 minutos para reducir los picos de carga. Use `--exact` para forzar una temporización precisa o `--stagger 30s` para establecer un intervalo explícito (solo programaciones cron).

### Migración de tareas de Heartbeat

La zona de trabajo temporal anterior de Heartbeat admitía un bloque estructurado `tasks:`. Ejecute `openclaw doctor --fix` después de actualizar para convertir cada entrada en un trabajo cron normal y editable de la sesión principal. Doctor conserva el intervalo y la hora de la ejecución anterior, crea los trabajos antes de eliminar el bloque y, al volver a ejecutarse, hace converger de forma segura las mismas claves de declaración.

Estos trabajos migrados contienen cargas útiles públicas `systemEvent`, por lo que `openclaw cron list`, `get`, `edit` y `remove`, junto con la herramienta cron, los gestionan como los demás trabajos. Su ejecución usa la activación protegida de tareas de Heartbeat: siguen aplicándose las horas activas, el espaciado mínimo, el control de saturación y los reintentos por ocupación, mientras que cron controla la cadencia independiente de cada tarea. Los trabajos que vencen en el mismo intervalo de agrupación pueden compartir un turno de Heartbeat. Una ejecución programada fuera de las horas activas de Heartbeat se omite y se vuelve a intentar en la siguiente ejecución del trabajo.

La zona de trabajo temporal de Heartbeat ahora solo contiene texto de supervisión. Los Heartbeat del entorno de ejecución no interpretan el texto `tasks:` como programaciones; cree el nuevo trabajo recurrente con cron.

### Orígenes de flujo

Una programación de flujo mantiene en ejecución un comando argv creado por el operador bajo el Gateway y activa el trabajo a partir de sus líneas de stdout y stderr. Las programaciones de flujo se controlan mediante eventos, nunca vencen por tiempo y requieren `cron.triggers.enabled: true`, porque el comando de larga duración pertenece a la misma clase de confianza sin supervisión que los scripts desencadenadores. Al deshabilitar o eliminar el trabajo, se detiene el proceso; al apagarse, el Gateway espera a que se desmonte el árbol de procesos. Los fallos rápidos se reinician con el retroceso de errores integrado de cron. Cinco ejecuciones consecutivas de menos de 60 segundos dejan el trabajo en estado de error y utilizan la ruta normal de alertas de fallo; vuelva a habilitar manualmente el trabajo para eliminar el límite de reinicios.

```bash
openclaw cron add \
  --name "Flujo de eventos de compilación" \
  --stream-command '["node","scripts/build-events.mjs"]' \
  --stream-mode match \
  --stream-match '^(failed|recovered):' \
  --stream-batch-ms 250 \
  --session isolated \
  --message "Investiga estos eventos de compilación."
```

`mode: "line"` (el valor predeterminado) acepta todas las líneas. `mode: "match"` acepta únicamente las líneas que coinciden con la expresión regular `match` compilada. Un lote se cierra después de `batchMs` de inactividad (250 ms de forma predeterminada, limitado a 50–5000) o al alcanzar `maxBatchBytes` (16384 de forma predeterminada, limitado a 1024–65536). Al alcanzar el límite de bytes, el lote termina con `[truncated]`. El modo de coincidencia siempre evalúa líneas completas respecto a todo su texto, incluso después de `maxBatchBytes` (solo se trunca el lote entregado); una línea cortada en el límite acotado de entrada sin procesar es únicamente un prefijo, por lo que se considera que no coincide en lugar de permitir que un patrón anclado al final se active sobre el texto cortado. El lote se añade al texto del evento del sistema o al mensaje del turno del agente. Las cargas útiles de comandos se rechazan en las programaciones de flujo porque el comando de origen y el comando de la carga útil tendrían una propiedad de proceso ambigua.

Solo se conserva por trabajo una activación de carga útil y un lote pendiente acotado. Las líneas que llegan mientras se ejecuta una carga útil, o antes de que transcurra el intervalo integrado de 30 segundos entre desencadenadores, se agrupan en ese lote pendiente en lugar de formar una cola sin límites. Un único propietario serializado registra las omisiones de la condición, los errores de la carga útil y los envíos mientras no está en ejecución en `streamDroppedBatches`; las combinaciones acotadas incrementan `streamCoalescedBatches`. Las cargas útiles fallidas no se vuelven a intentar porque podrían no ser idempotentes. La identidad lógica del origen permanece estable entre reinicios supervisados del proceso secundario, pero cambia cuando el origen se deshabilita, elimina o sustituye, de modo que los lotes en cola del origen retirado no puedan activarse ni siquiera después de una edición de A a B y de nuevo a A. Cuando finaliza una detención, las llamadas tardías de un proceso secundario anterior quedan inertes. V1 no incluye un origen WebSocket nativo; conéctelo mediante un comando argv como `websocat wss://example.invalid/events`.

Cuando un trabajo de flujo también tiene `trigger.script`, la condición se ejecuta una vez por cada lote cerrado. El lote actual está disponible como la cadena profundamente inmutable `trigger.streamBatch` junto con `trigger.state`. `fire: false` descarta ese lote después de conservar el estado de la condición. `fire: true` mantiene la semántica existente del mensaje desencadenador y, a continuación, añade el lote a la carga útil resultante. Un trabajo de flujo puede usar en su lugar una carga útil de script sin una condición; ese script recibe el lote mediante el mismo valor `trigger.streamBatch`. Se rechaza la combinación de una carga útil de script con una condición porque ambas serían propietarias de la ranura persistente `trigger.state`.

### Cadencia dinámica (ritmo)

Los trabajos recurrentes pueden establecer `pacing.min` o `pacing.max`, o ambos, en cadenas de duración como `15m` o `4h`; se requiere al menos un límite. Use `--pacing-min` y `--pacing-max` con `cron add|edit` (`--clear-pacing` elimina ambos límites).

Durante una ejecución aislada, un trabajo con ritmo controlado puede llamar a la herramienta `cron` con `action: "next_check"` y `in: "30m"`. La propuesta se aplica únicamente al trabajo que se está ejecutando en ese momento y se mide desde la finalización correcta de la ejecución. OpenClaw la ajusta silenciosamente a los límites configurados.

El ritmo controlado sin una propuesta deja sin cambios la programación normal. Las ejecuciones fallidas, agotadas por tiempo o omitidas descartan la propuesta, por lo que tienen prioridad el comportamiento existente de reintentos y la espera incremental por errores. Forzar manualmente un trabajo recurrente queda fuera de la secuencia y conserva su franja natural o con ritmo controlado pendiente. En los trabajos activados por condiciones, el intervalo mínimo integrado sigue siendo un límite inferior, incluso cuando una propuesta solicita una comprobación anterior.

### El día del mes y el día de la semana usan lógica OR

Las expresiones Cron se analizan mediante [croner](https://github.com/Hexagon/croner). Cuando tanto el campo de día del mes como el de día de la semana no son comodines, croner encuentra una coincidencia cuando coincide **cualquiera** de los campos, no ambos. Este es el comportamiento estándar de cron de Vixie.

```bash
# Previsto: "A las 9 a. m. del día 15, solo si es lunes"
# Real:     "A las 9 a. m. de cada día 15 Y a las 9 a. m. de cada lunes"
0 9 15 * 1
```

Esto se activa aproximadamente entre 5 y 6 veces al mes en lugar de entre 0 y 1 veces al mes. Para exigir ambas condiciones, use el modificador de día de la semana `+` de croner (`0 9 15 * +1`), o programe según un campo y compruebe el otro en el prompt o comando del trabajo.

## Activadores de eventos (supervisores de condiciones)

Un activador de eventos añade un script de condición sin interfaz a una programación `every`, `cron` o `stream`. Las programaciones temporales lo evalúan cuando corresponde; las programaciones de flujo lo evalúan para cada lote cerrado. Cron ejecuta la carga útil normal únicamente cuando el script devuelve `fire: true`:

```json5
{
  schedule: { kind: "every", everyMs: 30000 },
  trigger: {
    // Se activa únicamente cuando el estado observado difiere del de la última evaluación.
    script: "const res = await tools.call('exec', { command: 'gh pr checks 123 --json state -q \\'.[].state\\' | sort -u' }); const status = String(res?.result?.details?.aggregated ?? '').trim(); json({ fire: status !== trigger.state?.status, message: `CI del PR 123: ${trigger.state?.status ?? 'desconocido'} -> ${status}`, state: { status } });",
    once: false,
  },
  payload: { kind: "agentTurn", message: "Investiga el cambio de estado de la CI." },
}
```

El script debe devolver `{ fire, message?, state? }`. El estado JSON anterior está disponible como `trigger.state`, profundamente congelado; las compuertas de flujo también reciben el lote actual como `trigger.streamBatch`. Devuelva un nuevo valor `state` para conservarlo. El estado está limitado a 16 KB. Cuando un resultado de activación incluye `message`, Cron lo añade al texto del evento del sistema o al mensaje del turno del agente antes de la ejecución. `once: true` desactiva el trabajo después de su primera carga útil activada correctamente.

`fire: false` conserva el estado y los contadores de la evaluación y, a continuación, vuelve a programar sin crear un historial de ejecuciones. Si falla una ejecución de carga útil activada, el valor `state` devuelto **no** se conserva: la siguiente evaluación ve el estado anterior y puede volver a activarse, por lo que los scripts deben ser comprobaciones de solo lectura y las acciones deben permanecer en la carga útil. Las programaciones de activadores tienen un intervalo mínimo integrado de 30 segundos. Cada evaluación dispone de un límite de tiempo de reloj de 30 segundos y hasta 5 llamadas a herramientas.

Diseñe los supervisores en torno a un **estado que permita actuar**, no solo al éxito: un supervisor que deja de emitir resultados cuando su comprobación falla o agota el tiempo parece estar en buen estado aunque esté averiado. Compare la observación con `trigger.state` y devuelva un estado nuevo para eliminar duplicados; no dependa de la memoria del modelo ni del proceso. Al activarse, haga que `message` sea autosuficiente, ya que se convierte en el contexto completo del evento de la ejecución activada.

<Warning>
Habilitar `cron.triggers.enabled` permite que tanto los scripts activados por condiciones como las cargas útiles `script` se ejecuten sin interfaz con la **política de herramientas completa del agente propietario, incluido `exec`**. Trátelo como una ejecución de código desatendida con los permisos de ese agente; déjelo deshabilitado a menos que todos los agentes autorizados para crear trabajos Cron sean de confianza para ello.
</Warning>

Cree un supervisor a partir de un archivo de script local (`-` lee el script desde la entrada estándar):

```bash
openclaw cron add \
  --name "Supervisor de CI del PR" \
  --every 30s \
  --trigger-script ./watch-pr-ci.js \
  --message "Responde al cambio de estado de la CI" \
  --session isolated
```

## Cargas útiles

Cada trabajo contiene exactamente un tipo de carga útil, elegido mediante una opción:

| Carga útil      | Indicador                                      | Ejecución                                                  |
| --------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Evento del sistema | `--system-event <text>`                        | Se pone en cola en la sesión principal, sin llamada al modelo por sí solo |
| Mensaje del agente | `--message <text>`                         | Un turno de agente respaldado por un modelo                 |
| Comando         | `--command <shell>` o `--command-argv <json>` | Un shell/proceso en el host del Gateway, sin llamada al modelo |
| Script          | `--script <file\|->`                           | Un script sin interfaz en modo de código que usa las herramientas del agente propietario |

Un tipo adicional de carga útil, `heartbeat`, es propiedad del sistema: el Gateway converge una tarea de supervisión de Heartbeat por cada agente con Heartbeat habilitado (consulte [Heartbeat](/es/gateway/heartbeat)). Aparece en `cron list --all`, pero no se puede crear ni editar mediante la CLI ni la API. La configuración de Heartbeat se escribe en la programación persistente del supervisor al iniciar, al recargar la configuración o mediante `openclaw doctor --fix`. Cuando Cron está deshabilitado, el supervisor no se activa y no se ejecuta ningún temporizador alternativo de Heartbeat.

### Opciones de turno del agente

<ParamField path="--message" type="string" required>
  Texto de la instrucción (obligatorio para tareas de sesión aislada, actual o personalizada).
</ParamField>
<ParamField path="--model" type="string">
  Sustitución del modelo; debe resolverse en un modelo permitido o la ejecución falla con un error de validación.
</ParamField>
<ParamField path="--fallbacks" type="string">
  Lista de modelos alternativos por tarea, por ejemplo, `--fallbacks openai/gpt-5.6-sol,openrouter/meta-llama/llama-3.3-70b-instruct:free`. Pase `--fallbacks ""` para una ejecución estricta sin modelos alternativos.
</ParamField>
<ParamField path="--clear-fallbacks" type="boolean">
  En `cron edit`, elimina la sustitución de modelos alternativos por tarea para que esta siga la precedencia configurada de modelos alternativos. No se puede combinar con `--fallbacks`.
</ParamField>
<ParamField path="--clear-model" type="boolean">
  En `cron edit`, elimina la sustitución del modelo por tarea para que esta siga la precedencia normal de modelos de Cron (sustitución almacenada de la sesión de Cron o, en su defecto, modelo del agente/predeterminado). No se puede combinar con `--model`.
</ParamField>
<ParamField path="--thinking" type="string">
  Sustitución del nivel de razonamiento (`off|minimal|low|medium|high|xhigh|adaptive|max|ultra`). Los niveles disponibles siguen dependiendo del modelo y del entorno de ejecución del agente seleccionados.
</ParamField>
<ParamField path="--clear-thinking" type="boolean">
  En `cron edit`, elimina la sustitución del nivel de razonamiento por tarea. No se puede combinar con `--thinking`.
</ParamField>
<ParamField path="--light-context" type="boolean">
  Omite la inyección del archivo de arranque del espacio de trabajo.
</ParamField>
<ParamField path="--tools" type="string">
  Restringe las herramientas que puede usar la tarea, por ejemplo, `--tools exec,read`.
</ParamField>

Las tareas nuevas que pueden ejecutar herramientas siempre almacenan una política de herramientas explícita. Las tareas creadas por un agente
se limitan a las herramientas disponibles para el turno que las crea, y el agente no puede ampliar la
lista almacenada. Las tareas creadas por un operador autenticado sin `--tools` almacenan una
política `*` sin restricciones; `cron edit --clear-tools` restaura esa política explícita sin
restricciones. Las tareas existentes anteriores a una política de herramientas explícita conservan su comportamiento actual
hasta que su política de herramientas se edite explícitamente o se vuelva a crear la tarea.

`--model` establece el modelo principal de la tarea; no reemplaza una sustitución de sesión `/model`, por lo que las cadenas configuradas de modelos alternativos siguen aplicándose sobre este. Un modelo no resuelto o no permitido hace que la ejecución falle con un error de validación explícito en lugar de recurrir silenciosamente al predeterminado. Si una tarea tiene `--model`, pero ninguna lista explícita o configurada de modelos alternativos, OpenClaw pasa una sustitución vacía de modelos alternativos en lugar de añadir silenciosamente el modelo principal del agente como destino oculto de reintento.

Precedencia de selección de modelos para tareas aisladas, de mayor a menor:

1. Carga útil por tarea `model` (configuración explícita; un modelo no permitido hace que la ejecución falle)
2. Sustitución del modelo del hook de Gmail (solo cuando la ejecución procede de Gmail y esa sustitución está permitida)
3. Sustitución del modelo almacenada para la sesión de Cron seleccionada por el usuario
4. Selección del modelo del agente/predeterminado

El modo rápido sigue la selección activa resuelta. Si la configuración del modelo seleccionado tiene `params.fastMode`, Cron aislado lo usa de forma predeterminada; una sustitución de sesión almacenada `fastMode` (y después una `fastModeDefault` del agente) sigue teniendo prioridad sobre la configuración del modelo en cualquier dirección. El modo automático usa el límite `params.fastAutoOnSeconds` del modelo, con un valor predeterminado de 60 segundos.

Si una ejecución alcanza una transferencia activa por cambio de modelo, Cron vuelve a intentarlo con el proveedor/modelo al que se cambió y conserva esa selección (y cualquier perfil de autenticación nuevo) para la ejecución activa. Los reintentos están limitados: después del intento inicial y 2 reintentos por cambio, Cron se interrumpe en lugar de entrar en un bucle.

Antes de iniciar una ejecución aislada, OpenClaw comprueba los endpoints locales accesibles de los proveedores `api: "ollama"` y `api: "openai-completions"` configurados cuyo `baseUrl` sea de bucle invertido, de red privada o `.local`. Esta comprobación preliminar recorre la cadena configurada de modelos alternativos de la tarea y solo marca la ejecución como `skipped` cuando todos los candidatos son inaccesibles; `--fallbacks ""` limita estrictamente ese recorrido al modelo principal. Un endpoint inactivo registra la ejecución como `skipped` con un error claro en lugar de iniciar una llamada al modelo. El resultado se almacena en caché durante 5 minutos por endpoint (no por tarea ni modelo), por lo que muchas tareas programadas que comparten un servidor local Ollama/vLLM/SGLang/LM Studio inactivo generan una sola comprobación en lugar de una avalancha de solicitudes. Las ejecuciones omitidas en la comprobación preliminar no incrementan el retardo por errores de ejecución; establezca `failureAlert.includeSkipped` para habilitar alertas repetidas de omisión.

### Cargas útiles de comandos

Las cargas útiles de comandos ejecutan scripts deterministas dentro del planificador del Gateway sin iniciar un turno respaldado por un modelo. Se ejecutan en el host del Gateway, capturan stdout/stderr, registran la ejecución en el historial de Cron y reutilizan los mismos modos de entrega `announce`, `webhook` y `none` que las tareas de turno del agente.

<Note>
Cron de comandos es una superficie de automatización del Gateway para administradores y operadores, no una llamada `tools.exec` de un agente. Crear, actualizar, eliminar o ejecutar manualmente tareas de Cron requiere `operator.admin`; las ejecuciones programadas de comandos se ejecutan posteriormente dentro del proceso del Gateway como esa automatización creada por el administrador. La política de ejecución del agente (`tools.exec.mode`, solicitudes de aprobación y listas de herramientas permitidas por agente) rige las herramientas de ejecución visibles para el modelo, no las cargas útiles de Cron de comandos.
</Note>

```bash
openclaw cron create "*/15 * * * *" \
  --name "Sondeo de profundidad de la cola" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>` almacena `argv: ["sh", "-lc", <shell>]`. Use `--command-argv '["node","scripts/report.mjs"]'` para ejecutar argv de forma exacta sin análisis del shell. Los parámetros opcionales `--command-env KEY=VALUE` (repetible), `--command-input`, `--timeout-seconds` (valor predeterminado de 10 minutos), `--no-output-timeout-seconds` y `--output-max-bytes` controlan el entorno del proceso, stdin y los límites de salida.

El texto entregado se deriva de la salida del proceso: stdout no vacío tiene prioridad; si stdout está vacío y stderr no lo está, se entrega stderr; si ambos están presentes, Cron envía un pequeño bloque `stdout:` / `stderr:`. El código de salida `0` registra la ejecución como `ok`; una salida distinta de cero, una señal, un tiempo de espera agotado o un tiempo de espera agotado sin salida registra `error` y puede activar alertas de fallo. Un comando que solo imprime `NO_REPLY` usa la supresión normal de tokens silenciosos de Cron y no publica nada en el chat.

### Cargas útiles de scripts

Las cargas útiles de script se ejecutan sin interfaz en el mismo ejecutor de modo de código que los scripts de activación, sin iniciar un turno de agente conversacional. Habilite `cron.triggers.enabled` antes de crearlas o ejecutarlas; esta barrera para automatizaciones peligrosas abarca tanto los scripts de activación como las cargas útiles de script. Los trabajos de script solo admiten los destinos de sesión `main` y `isolated`.

```bash
openclaw cron create "0 * * * *" \
  --name "Hourly queue check" \
  --script ./automation/check-queue.js \
  --script-timeout-seconds 300 \
  --script-tool-budget 50 \
  --session isolated \
  --announce
```

Use `--script <file|->` para leer JavaScript desde un archivo o desde la entrada estándar. El tiempo de espera predeterminado es de 300 segundos, con un máximo de 900; el presupuesto de herramientas predeterminado es de 50 llamadas, con un máximo de 200. Estos presupuestos de carga útil son independientes de los presupuestos de evaluación más pequeños de la barrera de activación.

El script puede devolver un objeto con estos campos opcionales:

- `notify`: Texto entregado mediante el modo de entrega `announce`, `webhook` o `none` del trabajo. Si se omite, no se entrega nada. Para un trabajo `main`, el texto se convierte en un evento del sistema.
- `wake`: `"now"` solicita un heartbeat inmediato después de poner en cola `notify` (o un evento de finalización compacto); `"next-heartbeat"` pone el evento en cola para el siguiente heartbeat.
- `state`: Estado JSON, limitado a 16 KB y persistido únicamente después de una ejecución correcta. La siguiente ejecución recibe una copia inmutable como `trigger.state`, igual que los scripts de activación. Como ese espacio de nombres tiene un único propietario persistente, una carga útil de script no puede combinarse con un activador condicional en el mismo trabajo.
- `nextCheck`: Una duración, como `"15m"`. Solo es válida para trabajos con control de ritmo habilitado y utiliza el mismo límite de ritmo que las propuestas de turnos de agente.

Las excepciones, los tiempos de espera agotados, los presupuestos de herramientas agotados, los resultados no válidos y `nextCheck` sin control de ritmo son errores normales de ejecución de cron: se incorporan al historial de ejecuciones, al aplazamiento progresivo y a la gestión de alertas de fallo sin persistir el estado devuelto.

## Estilos de ejecución

| Estilo           | Valor de `--session`   | Se ejecuta en                  | Ideal para                        |
| --------------- | ------------------- | ------------------------ | ------------------------------- |
| Sesión principal    | `main`              | Carril dedicado de activación de cron | Recordatorios, eventos del sistema        |
| Aislado        | `isolated`          | `cron:<jobId>` dedicado | Informes, tareas rutinarias en segundo plano      |
| Sesión actual | `current`           | Vinculada al crearla   | Trabajo recurrente consciente del contexto    |
| Sesión personalizada  | `session:custom-id` | Sesión con nombre persistente | Flujos de trabajo que se basan en el historial |

<AccordionGroup>
  <Accordion title="Sesión principal frente a aislada y personalizada">
    Los trabajos de **sesión principal** ponen en cola un evento del sistema en un carril de ejecución propiedad de cron y, opcionalmente, activan el heartbeat (`--wake now` o `--wake next-heartbeat`). Pueden utilizar el último contexto de entrega de la sesión principal de destino para las respuestas, pero no añaden los turnos rutinarios de cron al carril de conversación humana ni prolongan la vigencia del restablecimiento diario o por inactividad de la sesión de destino. Los trabajos **aislados** ejecutan un turno de agente dedicado con una sesión nueva. Las **sesiones personalizadas** (`session:xxx`) conservan el contexto entre ejecuciones, lo que permite flujos de trabajo como reuniones diarias de seguimiento que se basan en resúmenes anteriores.

    Los eventos de cron de la sesión principal son recordatorios autocontenidos de eventos del sistema. No incluyen automáticamente el prompt predeterminado del heartbeat ni el borrador del monitor de heartbeat; indíquelo explícitamente en el texto del evento de cron si un recordatorio debe consultar ese contexto.

  </Accordion>
  <Accordion title="Qué significa «sesión nueva» para los trabajos aislados">
    Un nuevo identificador de transcripción/sesión por ejecución. OpenClaw conserva las preferencias seguras (configuración de razonamiento, rapidez y detalle; etiquetas; y anulaciones de modelo o autenticación seleccionadas explícitamente por el usuario), pero no hereda el contexto de conversación del entorno de una fila de cron anterior: enrutamiento de canal/grupo, política de envío o puesta en cola, elevación, origen ni vinculación del entorno de ejecución de ACP. Use `current` o `session:<id>` cuando un trabajo recurrente deba basarse deliberadamente en el mismo contexto de conversación.
  </Accordion>
  <Accordion title="Contrato de ejecución sin supervisión">
    Los turnos de agente de cron aislados y de hooks son explícitamente sin supervisión: no hay nadie presente para aclarar ni aprobar. La respuesta final debe ser el entregable, no un plan, acuse de recibo o solicitud de información. El agente devuelve `HEARTBEAT_OK` cuando no hay nada que hacer e indica los fallos con claridad; cron gestiona la política de reintentos y alertas de fallo.

    En los trabajos programados de confianza, las propias instrucciones del trabajo prevalecen cuando solicitan intencionadamente una pregunta o un plan, y el agente puede eliminar un trabajo que ya no sea necesario. Los turnos de hooks externos reciben únicamente el contrato común de ejecución sin supervisión; no reciben esa anulación ni las indicaciones de autoeliminación a través del límite de contenido externo.

  </Accordion>
  <Accordion title="Entrega de subagentes y Discord">
    Cuando las ejecuciones aisladas de cron coordinan subagentes, la entrega da preferencia a la salida final del último descendiente en lugar de al texto provisional obsoleto del agente principal. Si aún hay descendientes en ejecución, OpenClaw suprime esa actualización parcial del agente principal en vez de anunciarla.

    Para los destinos de anuncio de Discord que solo admiten texto, OpenClaw envía una vez el texto final canónico del asistente en lugar de reproducir tanto el texto transmitido o intermedio como la respuesta final. Los elementos multimedia y las cargas útiles estructuradas de Discord se entregan por separado para no omitir archivos adjuntos ni componentes.

  </Accordion>
</AccordionGroup>

## Entrega y salida

| Modo       | Qué sucede                                                        |
| ---------- | ------------------------------------------------------------------- |
| `announce` | Entrega como alternativa el texto final al destino si el agente no lo envió |
| `webhook`  | Envía mediante POST la carga útil del evento finalizado a una URL                                |
| `none`     | Sin entrega alternativa del ejecutor                                         |

Use `--announce --channel telegram --to "-1001234567890"` para la entrega por canal. Para los temas de foros de Telegram, use `-1001234567890:topic:123`; OpenClaw también acepta la forma abreviada `-1001234567890:123`, propiedad de Telegram. Los invocadores directos de RPC/configuración pueden proporcionar `delivery.threadId` como cadena o número. Los destinos de Slack/Discord/Mattermost utilizan prefijos explícitos (`channel:<id>`, `user:<id>`). Los identificadores de sala de Matrix distinguen entre mayúsculas y minúsculas; use el identificador de sala exacto o la forma `room:!room:server` de Matrix.

Cuando la entrega de anuncios utiliza `channel: "last"` u omite `channel`, un destino con prefijo de proveedor, como `telegram:123`, puede seleccionar el canal antes de que cron recurra al historial de la sesión o a un único canal configurado. Solo los prefijos anunciados por el Plugin cargado actúan como selectores de proveedor. Si `delivery.channel` es explícito, el prefijo del destino debe indicar el mismo proveedor; `channel: "whatsapp"` con `to: "telegram:123"` se rechaza en lugar de permitir que WhatsApp interprete el identificador de Telegram como un número de teléfono. Los prefijos de tipo de destino y servicio (`channel:<id>`, `user:<id>`, `imessage:<handle>`, `sms:<number>`) siguen siendo sintaxis de destino propiedad del canal, no selectores de proveedor.

En los trabajos aislados, la entrega por chat es compartida: si hay disponible una ruta de chat, el agente puede utilizar la herramienta `message` incluso con `--no-deliver`. Si el agente envía al destino configurado/actual, OpenClaw omite el anuncio alternativo. De lo contrario, `announce`, `webhook` y `none` solo controlan lo que hace el ejecutor con la respuesta final después del turno del agente.

Cuando un agente crea un recordatorio aislado desde un chat activo, OpenClaw almacena el destino de entrega activo conservado para la ruta de anuncio alternativa. Las claves internas de sesión pueden estar en minúsculas; los destinos de entrega del proveedor no se reconstruyen a partir de esas claves cuando está disponible el contexto de chat actual.

La entrega implícita de anuncios utiliza listas de canales permitidos configuradas para validar y redirigir destinos obsoletos. Las aprobaciones del almacén de vinculaciones de mensajes directos no son destinatarios de automatización alternativos; establezca `delivery.to` o configure la entrada `allowFrom` del canal cuando un trabajo programado deba enviar mensajes de forma proactiva a un mensaje directo.

### Notificaciones de fallo

Las notificaciones de fallo siguen una ruta de destino independiente:

- `cron.failureDestination` establece un valor global predeterminado para las notificaciones de fallo.
- `job.delivery.failureDestination` lo anula para cada trabajo.
- Si ninguno está establecido y el trabajo ya realiza entregas mediante `announce`, las notificaciones de fallo recurren a ese destino principal de anuncios.
- `delivery.failureDestination` solo es compatible con trabajos `sessionTarget="isolated"`, salvo que el modo de entrega principal sea `webhook`.
- `failureAlert.includeSkipped: true` permite que un trabajo o una política global de alertas de cron reciba alertas repetidas de ejecuciones omitidas. Las ejecuciones omitidas mantienen un contador consecutivo independiente, por lo que no afectan al aplazamiento progresivo por errores de ejecución.
- `openclaw cron edit` expone el ajuste de alertas por trabajo: `--failure-alert`/`--no-failure-alert`, `--failure-alert-after <n>`, `--failure-alert-channel`, `--failure-alert-to`, `--failure-alert-cooldown`, `--failure-alert-include-skipped`/`--failure-alert-exclude-skipped`, `--failure-alert-mode` y `--failure-alert-account-id`.

### Idioma de salida

Los trabajos de cron no deducen el idioma de respuesta a partir del canal, la configuración regional ni los mensajes anteriores. Incluya la regla de idioma en el mensaje o la plantilla programados:

```bash
openclaw cron edit <jobId> \
  --message "Resuma las actualizaciones. Responda en chino; mantenga sin cambios las URL, el código y los nombres de productos."
```

En los archivos de plantilla, mantenga la instrucción de idioma en el prompt renderizado y compruebe que los marcadores de posición como `{{language}}` estén rellenados antes de ejecutar el trabajo. Si la salida mezcla idiomas, haga explícita la regla, por ejemplo: «Use chino para el texto narrativo y mantenga los términos técnicos en inglés».

## Ejemplos de CLI

<Tabs>
  <Tab title="Recordatorio de una sola ejecución">
    ```bash
    openclaw cron add \
      --name "Calendar check" \
      --at "20m" \
      --session main \
      --system-event "Next heartbeat: check calendar." \
      --wake now
    ```
  </Tab>
  <Tab title="Trabajo aislado recurrente">
    ```bash
    openclaw cron create "0 7 * * *" \
      "Summarize overnight updates." \
      --name "Morning brief" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --announce \
      --channel slack \
      --to "channel:C1234567890"
    ```
  </Tab>
  <Tab title="Anulación de modelo y razonamiento">
    ```bash
    openclaw cron add \
      --name "Deep analysis" \
      --cron "0 6 * * 1" \
      --tz "America/Los_Angeles" \
      --session isolated \
      --message "Weekly deep analysis of project progress." \
      --model "opus" \
      --thinking high \
      --announce
    ```
  </Tab>
  <Tab title="Salida de Webhook">
    ```bash
    openclaw cron create "0 18 * * 1-5" \
      "Summarize today's deploys as JSON." \
      --name "Deploy digest" \
      --webhook "https://example.invalid/openclaw/cron"
    ```
  </Tab>
  <Tab title="Salida de comandos">
    ```bash
    openclaw cron create "*/15 * * * *" \
      --name "Queue depth probe" \
      --command "scripts/check-queue.sh" \
      --command-cwd "/srv/app" \
      --announce \
      --channel telegram \
      --to "-1001234567890"
    ```
  </Tab>
</Tabs>

## Gestión de trabajos

```bash
# Enumerar trabajos habilitados
openclaw cron list

# Incluir trabajos deshabilitados
openclaw cron list --all

# Obtener un trabajo almacenado como JSON
openclaw cron get <jobId>

# Mostrar un trabajo, incluida la ruta de entrega resuelta
openclaw cron show <jobId>

# Habilitar/deshabilitar sin eliminar
openclaw cron enable <jobId>
openclaw cron disable <jobId>

# Editar un trabajo
openclaw cron edit <jobId> --message "Prompt actualizado" --model "opus"

# Forzar la ejecución inmediata de un trabajo
openclaw cron run <jobId>

# Forzar la ejecución inmediata de un trabajo y esperar su estado terminal
openclaw cron run <jobId> --wait --wait-timeout 10m --poll-interval 2s

# Ejecutar solo si corresponde
openclaw cron run <jobId> --due

# Ver el historial de ejecuciones
openclaw cron runs --id <jobId> --limit 50

# Ver una ejecución exacta
openclaw cron runs --id <jobId> --run-id <runId>

# Eliminar un trabajo
openclaw cron remove <jobId>

# Selección de agente (configuraciones multiagente)
openclaw cron create "0 6 * * *" "Revisar la cola de operaciones" --name "Revisión de operaciones" --session isolated --agent ops
openclaw cron edit <jobId> --clear-agent
```

Archivar una sesión (desde la interfaz de control o mediante `sessions.patch { archived: true }` desde un llamador administrador-operador) deshabilita todos los trabajos de Cron habilitados vinculados a esa sesión: su sesión aislada `cron:<jobId>`, un destino `session:<key>` o un canal de entrega/activación `sessionKey`. Restaurar la sesión no vuelve a habilitar esos trabajos; utilice `openclaw cron enable <jobId>`. Las sesiones con un trabajo vinculado habilitado muestran una insignia de reloj en la barra lateral de la interfaz de control.

`openclaw cron run <jobId>` retorna después de poner en cola la ejecución manual. Utilice `--wait` para enlaces de apagado, scripts de mantenimiento u otras automatizaciones que deban bloquearse hasta que finalice la ejecución en cola; consulta periódicamente el `runId` retornado (tiempo de espera predeterminado `10m`, intervalo de consulta `2s`) y sale con `0` para el estado `ok`, y con un valor distinto de cero para `error`, `skipped` o un tiempo de espera agotado.

La herramienta `cron` del agente retorna resúmenes compactos de trabajos (`id`, `name`, `enabled`, `nextRunAtMs`, `scheduleKind`, `lastRunStatus`) desde `cron(action: "list")`; utilice `cron(action: "get", jobId: "...")` para obtener la definición completa de un trabajo. Los llamadores directos del Gateway pueden pasar `compact: true` a `cron.list`; omitirlo conserva la respuesta completa con vistas previas de entrega.

`openclaw cron create` es un alias de `openclaw cron add`. Los trabajos nuevos pueden utilizar una programación posicional (`"0 9 * * 1"`, `"every 1h"`, `"20m"` o una marca de tiempo ISO) seguida de un prompt posicional para el agente. Utilice `--webhook <url>` en `cron add|create` o `cron edit` para enviar mediante POST la carga útil de la ejecución finalizada a un endpoint HTTP; la entrega mediante Webhook no puede combinarse con indicadores de entrega por chat (`--announce`, `--channel`, `--to`, `--thread-id`, `--account`). En `cron edit`, `--clear-channel`, `--clear-to`, `--clear-thread-id` y `--clear-account`, desconfigure esos campos de enrutamiento individualmente (cada uno se rechaza junto con su indicador de configuración correspondiente), a diferencia de `--no-deliver`, que solo deshabilita la entrega alternativa del ejecutor.

<Note>
Nota sobre la sustitución del modelo:

- `openclaw cron add|edit --model ...` cambia el modelo seleccionado del trabajo.
- Si el modelo está permitido, ese proveedor/modelo exacto llega a la ejecución aislada del agente.
- Si no está permitido o no puede resolverse, Cron marca la ejecución como fallida con un error explícito de validación.
- Las actualizaciones parciales de la carga útil de la API `cron.update` pueden establecer `model: null` para borrar la sustitución de modelo almacenada de un trabajo.
- `openclaw cron edit <job-id> --clear-model` borra esa sustitución desde la CLI (con el mismo efecto que la actualización parcial `model: null`) y no puede combinarse con `--model`.
- Las cadenas alternativas configuradas siguen aplicándose porque `--model` de Cron es el modelo principal de un trabajo, no una sustitución `/model` de sesión.
- `openclaw cron add|edit --fallbacks ...` establece `fallbacks` en la carga útil y reemplaza las alternativas configuradas para ese trabajo; `--fallbacks ""` deshabilita las alternativas y hace que la ejecución sea estricta. `openclaw cron edit <job-id> --clear-fallbacks` borra la sustitución específica del trabajo.
- Un `--model` simple sin una lista de alternativas explícita o configurada no recurre al modelo principal del agente como destino adicional de reintento silencioso.

</Note>

## Webhooks

El Gateway puede exponer endpoints de Webhook HTTP para activadores externos. Habilítelos en la configuración:

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
  },
}
```

### Autenticación

Cada solicitud debe incluir el token del enlace mediante un encabezado:

- `Authorization: Bearer <token>` (recomendado)
- `x-openclaw-token: <token>`

Los tokens en cadenas de consulta se rechazan.

<AccordionGroup>
  <Accordion title="POST /hooks/wake">
    Pone en cola un evento del sistema para la sesión principal:

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/wake \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"text":"Nuevo correo electrónico recibido","mode":"now"}'
    ```

    <ParamField path="text" type="string" required>
      Descripción del evento.
    </ParamField>
    <ParamField path="mode" type="string" default="now">
      `now` o `next-heartbeat`.
    </ParamField>

  </Accordion>
  <Accordion title="POST /hooks/agent">
    Ejecuta un turno aislado del agente:

    ```bash
    curl -X POST http://127.0.0.1:18789/hooks/agent \
      -H 'Authorization: Bearer SECRET' \
      -H 'Content-Type: application/json' \
      -d '{"message":"Resumir la bandeja de entrada","name":"Correo electrónico","model":"openai/gpt-5.6-sol"}'
    ```

    Campos: `message` (obligatorio), `name`, `agentId`, `sessionKey` (requiere `hooks.allowRequestSessionKey=true`), `idempotencyKey`, `wakeMode`, `deliver`, `channel`, `to`, `model`, `thinking`, `timeoutSeconds`.

  </Accordion>
  <Accordion title="Enlaces asignados (POST /hooks/<name>)">
    Los nombres personalizados de enlaces se resuelven mediante `hooks.mappings` en la configuración. Las asignaciones pueden transformar cargas útiles arbitrarias en acciones `wake` o `agent` mediante plantillas o transformaciones de código.
  </Accordion>
</AccordionGroup>

<Warning>
Mantenga los endpoints de enlaces detrás de la interfaz de bucle invertido, la tailnet o un proxy inverso de confianza.

- Utilice un token específico para enlaces; no reutilice tokens de autenticación del Gateway.
- Mantenga `hooks.path` en una subruta específica; `/` se rechaza.
- Establezca `hooks.allowedAgentIds` para limitar el agente efectivo al que puede dirigirse un enlace, incluido el agente predeterminado cuando se omite `agentId`.
- Mantenga `hooks.allowRequestSessionKey=false`, salvo que necesite sesiones seleccionadas por el llamador.
- Si habilita `hooks.allowRequestSessionKey`, establezca también `hooks.allowedSessionKeyPrefixes` para restringir las formas permitidas de las claves de sesión.
- De forma predeterminada, las cargas útiles de los enlaces se encapsulan con límites de seguridad.

</Warning>

## Integración con Gmail PubSub

Conecte los activadores de la bandeja de entrada de Gmail con OpenClaw mediante Google PubSub.

<Note>
**Requisitos previos:** CLI `gcloud`, `gog` (gogcli), enlaces de OpenClaw habilitados y Tailscale para el endpoint HTTPS público.
</Note>

### Configuración mediante el asistente (recomendada)

```bash
openclaw webhooks gmail setup --account openclaw@gmail.com
```

Esto escribe la configuración `hooks.gmail`, habilita el preajuste de Gmail y utiliza Tailscale Funnel de forma predeterminada para el endpoint de envío (`--tailscale funnel|serve|off`).

<Warning>
La sesión por mensaje del preajuste de Gmail separa el contexto de la conversación; no restringe las herramientas ni el espacio de trabajo del agente de destino. Sin una asignación personalizada que establezca `agentId`, los enlaces de Gmail se ejecutan como el agente predeterminado.

Para bandejas de entrada no confiables, dirija el enlace a un agente lector específico, proporcione a ese agente acceso de solo lectura o ningún acceso al espacio de trabajo y deniegue la escritura en el sistema de archivos, el shell, el navegador y otras herramientas innecesarias. Si necesita notificar al agente principal, permita únicamente la transferencia necesaria entre agentes. Consulte [Inyección de prompts](/es/gateway/security#prompt-injection), [Entorno aislado y herramientas multiagente](/es/tools/multi-agent-sandbox-tools) y [`tools.agentToAgent`](/es/gateway/config-tools#toolsagenttoagent).
</Warning>

### Inicio automático del Gateway

Cuando `hooks.enabled=true` y `hooks.gmail.account` están establecidos, el Gateway inicia `gog gmail watch serve` durante el arranque y renueva automáticamente la supervisión. Establezca `OPENCLAW_SKIP_GMAIL_WATCHER=1` para deshabilitarlo.

### Configuración manual única

<Steps>
  <Step title="Seleccionar el proyecto de GCP">
    Seleccione el proyecto de GCP propietario del cliente OAuth utilizado por `gog`:

    ```bash
    gcloud auth login
    gcloud config set project <project-id>
    gcloud services enable gmail.googleapis.com pubsub.googleapis.com
    ```

  </Step>
  <Step title="Crear el tema y conceder acceso de envío a Gmail">
    ```bash
    gcloud pubsub topics create gog-gmail-watch
    gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
      --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
      --role=roles/pubsub.publisher
    ```
  </Step>
  <Step title="Iniciar la supervisión">
    ```bash
    gog gmail watch start \
      --account openclaw@gmail.com \
      --label INBOX \
      --topic projects/<project-id>/topics/gog-gmail-watch
    ```
  </Step>
</Steps>

### Sustitución del modelo de Gmail

```json5
{
  hooks: {
    gmail: {
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

Utilice el modelo de última generación y del mejor nivel disponible de su proveedor para las bandejas de entrada no confiables. El valor anterior es un ejemplo; el modelo debe existir en el catálogo y la lista de permitidos configurados.

## Configuración

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    triggers: {
      enabled: false,
    },
    webhookToken: "replace-with-dedicated-webhook-token",
    sessionRetention: "24h",
  },
}
```

`webhookToken` se envía como `Authorization: Bearer <token>` en los POST de Webhook de Cron.

`cron.store` es una clave lógica de almacenamiento y una ruta de migración de doctor, no un archivo JSON activo que deba editarse manualmente. Los datos de los trabajos se encuentran en SQLite; utilice la CLI o la API del Gateway para realizar cambios.

Deshabilite Cron mediante `cron.enabled: false` o `OPENCLAW_SKIP_CRON=1`.

<AccordionGroup>
  <Accordion title="Comportamiento de los reintentos">
    **Reintento único**: los errores transitorios (límite de velocidad, sobrecarga, red, tiempo de espera agotado o error del servidor) utilizan una programación de reintentos integrada. Los errores permanentes deshabilitan el trabajo inmediatamente.

    **Reintento recurrente**: los errores consecutivos de ejecución aplican una espera progresiva según una programación ampliada (30s, 60s, 5m, 15m, 60m). La espera progresiva se restablece después de la siguiente ejecución correcta.

  </Accordion>
  <Accordion title="Mantenimiento">
    `cron.sessionRetention` (valor predeterminado `24h`; `false` lo deshabilita) depura las entradas de sesiones de ejecución aisladas. El historial de ejecuciones conserva las 2000 filas terminales más recientes por trabajo; las filas perdidas mantienen su periodo de limpieza de 24 horas.
  </Accordion>
  <Accordion title="Migración del almacén heredado">
    Después de actualizar, ejecute `openclaw doctor --fix` para importar los archivos heredados `~/.openclaw/cron/jobs.json`, `jobs-state.json` y `runs/*.jsonl` en SQLite y cambiarles el nombre con el sufijo `.migrated`. Las filas de trabajos con formato incorrecto se omiten durante la ejecución y se copian en `jobs-quarantine.json` para repararlas o revisarlas posteriormente.
  </Accordion>
</AccordionGroup>

## Solución de problemas

### Secuencia de comandos

```bash
openclaw status
openclaw gateway status
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
openclaw doctor
```

<AccordionGroup>
  <Accordion title="Cron no se ejecuta">
    - Compruebe `cron.enabled` y la variable de entorno `OPENCLAW_SKIP_CRON`.
    - Confirme que el Gateway se ejecuta continuamente.
    - Para las programaciones `cron`, verifique la zona horaria (`--tz`) frente a la zona horaria del host.
    - `reason: not-due` en la salida de la ejecución significa que la ejecución manual se comprobó con `openclaw cron run <jobId> --due` y que aún no correspondía ejecutar el trabajo.

  </Accordion>
  <Accordion title="Cron se activó, pero no hubo entrega">
    - El modo de entrega `none` significa que no se espera ningún envío de respaldo del ejecutor. El agente aún puede enviar directamente con la herramienta `message` cuando haya disponible una ruta de chat.
    - Si falta el destino de entrega o no es válido (`channel`/`to`), se omite el envío saliente.
    - En Matrix, los trabajos copiados o heredados con los ID de sala `delivery.to` en minúsculas pueden fallar porque los ID de sala de Matrix distinguen entre mayúsculas y minúsculas. Edite el trabajo para que use el valor exacto `!room:server` o `room:!room:server` de Matrix.
    - Los errores de autenticación del canal (`unauthorized`, `Forbidden`) significan que las credenciales bloquearon la entrega.
    - Si la ejecución aislada devuelve únicamente el token silencioso (`NO_REPLY` / `no_reply`), OpenClaw suprime la entrega saliente directa y la ruta de respaldo del resumen en cola, por lo que no se publica nada en el chat.
    - Si el propio agente debe enviar un mensaje al usuario, compruebe que el trabajo tenga una ruta utilizable (`channel: "last"` con un chat anterior o un canal/destino explícito).

  </Accordion>
  <Accordion title="Cron o Heartbeat parece impedir la renovación de estilo /new">
    - La vigencia del restablecimiento diario y por inactividad no se basa en `updatedAt`; consulte [Administración de sesiones](/es/concepts/session#session-lifecycle).
    - Las activaciones de Cron, las ejecuciones de Heartbeat, las notificaciones de exec y las operaciones de registro del Gateway pueden actualizar la fila de la sesión para el enrutamiento o el estado, pero no amplían `sessionStartedAt` ni `lastInteractionAt`.
    - En las filas heredadas creadas antes de que existieran esos campos, OpenClaw puede recuperar `sessionStartedAt` del encabezado de sesión del archivo de transcripción JSONL si el archivo sigue disponible. Las filas heredadas inactivas sin `lastInteractionAt` usan esa hora de inicio recuperada como referencia de inactividad.

  </Accordion>
  <Accordion title="Problemas habituales con las zonas horarias">
    - Cron sin `--tz` usa la zona horaria del host del Gateway.
    - Las programaciones `at` sin zona horaria se tratan como UTC.
    - El `activeHours` de Heartbeat usa la resolución de zona horaria configurada.

  </Accordion>
</AccordionGroup>

## Contenido relacionado

- [Automatización](/es/automation) — todos los mecanismos de automatización de un vistazo
- [Tareas en segundo plano](/es/automation/tasks) — registro de tareas para las ejecuciones de Cron
- [Heartbeat](/es/gateway/heartbeat) — turnos periódicos de la sesión principal
- [Zona horaria](/es/concepts/timezone) — configuración de la zona horaria
