---
read_when:
    - Cambio de la ejecución o la concurrencia de las respuestas automáticas
    - Explicación de los modos de `/queue` o del comportamiento de direccionamiento de mensajes
summary: Modos de cola de respuesta automática, valores predeterminados y anulaciones por sesión
title: Cola de comandos
x-i18n:
    generated_at: "2026-07-26T05:38:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 69b40f67146226b0315492b27fc9d2218cace8bbd1eaff6514f7efb33b69d763
    source_path: concepts/queue.md
    workflow: 16
---

OpenClaw serializa las ejecuciones de respuesta automática entrantes (de todos los canales) mediante una pequeña cola en proceso para evitar que varias ejecuciones del agente colisionen, al tiempo que permite un paralelismo seguro entre sesiones.

## Motivo

- Las ejecuciones de respuesta automática pueden ser costosas (llamadas al LLM) y pueden colisionar cuando llegan varios mensajes entrantes casi al mismo tiempo.
- La serialización evita la competencia por recursos compartidos (archivos de sesión, registros, entrada estándar de la CLI) y reduce la probabilidad de alcanzar los límites de frecuencia de servicios ascendentes.

## Funcionamiento

- Una cola FIFO con reconocimiento de carriles procesa cada carril con un límite de concurrencia configurable (de forma predeterminada, 1 para los carriles no configurados; `main` tiene un valor predeterminado de 4 y `subagent` de 8).
- `runEmbeddedAgent` pone en cola por **clave de sesión** (carril `session:<key>`) para garantizar que solo haya una ejecución activa por sesión.
- Después, cada ejecución de sesión se añade a un **carril global** (`main` de forma predeterminada), de modo que el paralelismo total quede limitado por `agents.defaults.maxConcurrent`.
- Cuando el registro detallado está habilitado, las ejecuciones en cola emiten un breve aviso si esperan más de ~2s antes de comenzar.
- Los indicadores de escritura se activan inmediatamente al poner una ejecución en cola (cuando el canal los admite), por lo que la experiencia de usuario no cambia mientras la ejecución espera su turno.

## Valores predeterminados

Cuando no se configura ningún valor, todas las superficies de canales entrantes usan:

- `mode: "steer"`
- `debounceMs: 500`
- `cap: 20`
- `drop: "summarize"`

La dirección durante el mismo turno es el comportamiento predeterminado. Un prompt que llega durante una ejecución se inyecta en el entorno de ejecución activo cuando este puede aceptar dirección, por lo que no se inicia una segunda ejecución de sesión. Si la ejecución activa no puede aceptar dirección, OpenClaw espera a que termine antes de iniciar el prompt.

## Modos de cola

`/queue` controla lo que hacen los mensajes entrantes normales mientras una sesión ya tiene una ejecución activa:

- `steer`: inyecta mensajes en el entorno de ejecución activo. OpenClaw entrega todos los mensajes de dirección pendientes **después de que el turno actual del asistente termine de ejecutar sus llamadas a herramientas**, antes de la siguiente llamada al LLM; el servidor de aplicaciones de Codex recibe un único `turn/steer` por lotes. Si la ejecución no está transmitiendo activamente o la dirección no está disponible, OpenClaw espera hasta que termine la ejecución activa antes de iniciar el prompt.
- `followup`: no aplica dirección. Pone cada mensaje en cola para un turno posterior del agente cuando termine la ejecución actual.
- `collect`: no aplica dirección. Agrupa los mensajes en cola en un **único** turno de seguimiento después del intervalo de inactividad. Si los mensajes se dirigen a canales o hilos distintos, se procesan individualmente para conservar el enrutamiento.
- `interrupt`: cancela la ejecución activa de esa sesión y, a continuación, ejecuta el mensaje más reciente.

Para conocer el comportamiento de temporización y dependencias específico del entorno de ejecución, consulte [Cola de dirección](/es/concepts/queue-steering). Para el comando explícito `/steer <message>`, consulte [Dirigir](/es/tools/steer).

Configure el comportamiento globalmente o por canal mediante `messages.queue`:

```json5
{
  messages: {
    queue: {
      mode: "steer",
      debounceMs: 500,
      cap: 20,
      drop: "summarize",
      byChannel: { discord: "collect" },
    },
  },
}
```

## Opciones de la cola

Las opciones se aplican a la entrega en cola. `debounceMs` también establece el intervalo de inactividad de dirección de Codex en el modo `steer`:

- `debounceMs`: intervalo de inactividad antes de procesar los seguimientos en cola o los lotes agrupados; en el modo `steer` de Codex, intervalo de inactividad antes de enviar `turn/steer` por lotes. Los números sin unidad representan milisegundos; las opciones `/queue` aceptan las unidades `ms`, `s`, `m`, `h` y `d`.
- `cap`: número máximo de mensajes en cola por sesión. Los valores inferiores a `1` se ignoran.
- `drop: "summarize"` (valor predeterminado): descarta las entradas más antiguas de la cola según sea necesario, conserva resúmenes compactos y los inyecta como un prompt de seguimiento sintético.
- `drop: "old"`: descarta las entradas más antiguas de la cola según sea necesario, sin conservar resúmenes.
- `drop: "new"`: rechaza el mensaje más reciente cuando la cola ya está llena.

Valores predeterminados: `debounceMs: 500`, `cap: 20`, `drop: summarize`.

## Dirección y transmisión

Cuando la transmisión del canal es `partial` o `block`, la dirección puede parecer una serie de respuestas visibles breves mientras la ejecución activa alcanza los límites del entorno de ejecución:

- `partial`: la vista previa puede finalizar anticipadamente y, después, comienza una nueva vista previa cuando se acepta la dirección.
- `block`: los bloques del tamaño de un borrador pueden producir la misma apariencia secuencial.
- Sin transmisión, cuando el entorno de ejecución no puede aceptar dirección durante el mismo turno, la dirección pasa a ser un seguimiento posterior a la ejecución activa.

`steer` no cancela las herramientas en curso. Use `/queue interrupt` cuando el mensaje más reciente deba cancelar la ejecución actual.

## Precedencia

Para seleccionar el modo, OpenClaw resuelve:

1. La anulación `/queue` por sesión, integrada o almacenada.
2. `messages.queue.byChannel.<channel>`.
3. `messages.queue.mode`.
4. Valor predeterminado `steer`.

Para las opciones, las opciones `/queue` integradas o almacenadas tienen precedencia sobre la configuración. A continuación, se aplican, en este orden, el antirrebote específico del canal (`messages.queue.debounceMsByChannel`), los valores predeterminados de antirrebote del plugin, las opciones globales `messages.queue` y los valores predeterminados integrados. `cap` y `drop` son opciones globales o de sesión, no claves de configuración por canal.

## Anulaciones por sesión

- Envíe `/queue <steer|followup|collect|interrupt>` como comando independiente para almacenar el modo de cola de la sesión actual.
- Las opciones pueden combinarse: `/queue collect debounce:0.5s cap:25 drop:summarize`
- `/queue default` o `/queue reset` elimina la anulación de la sesión.

## Cancelación de turnos en cola

Mientras un prompt permanece en la cola de seguimiento o agrupación (por ejemplo, un `chat.send` de la TUI o del
chat web que llega mientras otro turno está activo), el Gateway conserva una
**identidad de cancelación propiedad del Gateway** para el `runId` de ese cliente hasta que el contenido
en cola se ejecute o se descarte. La identidad acompaña al contenido incorporado en un
resumen de desbordamiento.

- `chat.abort` con un `runId` específico cancela ese turno mientras todavía está
  en cola, si el solicitante está autorizado (se aplican las mismas reglas de propiedad que para las ejecuciones activas).
- `chat.abort` para una sesión sin `runId` cancela **primero los turnos en cola autorizados
  ** y, después, cancela las ejecuciones activas autorizadas. Este orden evita que el procesamiento de la cola
  promueva trabajo a una sesión detenida parcialmente.
- Vaciar toda la cola de la sesión sin comprobaciones por solicitante no es la
  vía de detención para sesiones con varios propietarios.
- Las esperas en cola no se representan como ejecuciones activas del agente para `sessions.list` y
  no poseen la semántica de tiempo de espera de una ejecución activa; solo la posee la fase activa.

Los clientes respaldados por el Gateway (incluido `openclaw tui`) reenvían los prompts recibidos durante una ejecución y
permiten que el Gateway aplique el modo de cola. Esc/`/stop` usa una cancelación limitada a la sesión
para que la pérdida de identificadores locales no permita que un prompt todavía en cola siga ejecutándose.

`openclaw chat` y `openclaw tui --local` aplican los mismos cuatro modos en el
entorno de ejecución integrado. El `steer` local se inyecta en una ejecución integrada activa cuando ese
entorno de ejecución acepta dirección y, en caso contrario, se convierte en un seguimiento; `followup` y
`collect` permanecen como trabajo local pendiente; `interrupt` cancela la ejecución local activa
antes de iniciar el mensaje más reciente. El comando explícito `/steer <message>`
no es un comando de modo local.

## Alcance y garantías

- Se aplica a las ejecuciones del agente de respuesta automática en todos los canales entrantes que usan el pipeline de respuestas del Gateway (web de WhatsApp, Telegram, Slack, Discord, Signal, iMessage, chat web, etc.).
- El carril predeterminado (`main`) se aplica a todo el proceso para las entradas y los heartbeats principales; establezca `agents.defaults.maxConcurrent` para permitir varias sesiones en paralelo.
- Pueden existir carriles adicionales (por ejemplo, `cron`, `cron-nested`, `nested`, `subagent`) para que los trabajos en segundo plano puedan ejecutarse en paralelo sin bloquear las respuestas entrantes. Los turnos aislados del agente de Cron ocupan una ranura `cron` mientras la ejecución interna del agente usa `cron-nested`. Los flujos `nested` compartidos que no son de Cron conservan su propio comportamiento de carril. Estas ejecuciones independientes se registran como [tareas en segundo plano](/es/automation/tasks).
- Los carriles por sesión garantizan que solo una ejecución del agente acceda a una sesión determinada a la vez.
- Sin dependencias externas ni hilos de trabajo en segundo plano; solo TypeScript y promesas.

## Solución de problemas

- Si los comandos parecen bloqueados, habilite los registros detallados y busque líneas como "queued for ...ms" para confirmar que la cola se está procesando.
- El adaptador de Codex interrumpe las ejecuciones del servidor de aplicaciones de Codex que aceptan un turno y después dejan de emitir progreso, de modo que el carril de la sesión activa pueda liberarse en lugar de esperar al tiempo de espera de la ejecución exterior.
- Cuando los diagnósticos están habilitados, las sesiones que permanecen en `processing` más allá del umbral de advertencia integrado sin que se observe progreso de respuestas, herramientas, estados, bloques o ACP se clasifican según su actividad actual:
  - El trabajo activo con progreso reciente se registra como `session.long_running`. Las llamadas silenciosas al modelo que tienen propietario también permanecen como `session.long_running` hasta el umbral de cancelación integrado, para evitar que los proveedores lentos o sin transmisión se notifiquen como bloqueados demasiado pronto.
  - El trabajo activo sin progreso reciente se registra como `session.stalled`; las llamadas al modelo con propietario, las llamadas a herramientas bloqueadas y las ejecuciones integradas bloqueadas cambian a `session.stalled` al alcanzar o superar el umbral de cancelación. La actividad obsoleta de modelos o herramientas sin propietario no se oculta como actividad de larga duración.
  - `session.stuck` se reserva para registros obsoletos recuperables de sesiones, incluidas las sesiones inactivas en cola con actividad obsoleta de modelos o herramientas sin propietario.
  - `session.stuck` siempre activa una recuperación que puede liberar el carril de la sesión afectada. Una clasificación `session.stalled` que supere el umbral de cancelación (llamada a herramienta bloqueada, llamada al modelo bloqueada o ejecución integrada bloqueada) también puede activar la recuperación mediante cancelación activa, por lo que ambas clasificaciones pueden desbloquear una cola, no solo `session.stuck`.
  - Las líneas de registro de advertencia `session.stuck` y `session.long_running` repetidas reducen su frecuencia exponencialmente mientras la sesión no cambia; los intentos de recuperación siguen ejecutándose en cada ciclo del heartbeat independientemente de esa reducción.

## Temas relacionados

- [Gestión de sesiones](/es/concepts/session)
- [Cola de dirección](/es/concepts/queue-steering)
- [Dirigir](/es/tools/steer)
- [Política de reintentos](/es/concepts/retry)
