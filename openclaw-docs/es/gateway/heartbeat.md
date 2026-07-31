---
read_when:
    - Ajuste de la frecuencia o los mensajes de Heartbeat
    - Decidir entre Heartbeat y Cron para tareas programadas
sidebarTitle: Heartbeat
summary: Mensajes de sondeo de Heartbeat y reglas de notificación
title: Heartbeat
x-i18n:
    generated_at: "2026-07-26T05:11:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 44c78e797987d8dccab910cd82fc1f482df86afce40677846d8f26522d33f6fa
    source_path: gateway/heartbeat.md
    workflow: 16
---

<Note>
**¿Heartbeat o cron?** Consulta [Automatización](/es/automation) para obtener orientación sobre cuándo usar cada uno.
</Note>

Heartbeat ejecuta **turnos periódicos del agente** en la sesión principal para que el modelo pueda señalar cualquier asunto que requiera atención sin enviar mensajes innecesarios.

Heartbeat es un turno programado de la sesión principal; **no** crea registros de [tareas en segundo plano](/es/automation/tasks). Los registros de tareas se usan para trabajos independientes (ejecuciones de ACP, subagentes y trabajos de cron aislados).

Internamente, la cadencia de Heartbeat pertenece al programador de cron: el Gateway mantiene un trabajo de cron propiedad del sistema por cada agente con Heartbeat habilitado (visible en `openclaw cron list --all` como `Heartbeat (agent-id)`). La configuración de Heartbeat sigue siendo la entrada del estado deseado, mientras que la programación persistente del monitor controla el tick real y el posterior tiempo de espera del ejecutor. El Gateway aplica los cambios de configuración al iniciarse y al recargar la configuración; `openclaw doctor --fix` puede materializar las filas del monitor que falten o estén obsoletas antes del siguiente inicio del Gateway. Edita `agents.*.heartbeat`, no el trabajo de cron.

Los Heartbeats programados requieren cron. Cuando `cron.enabled` es `false` o `OPENCLAW_SKIP_CRON=1`, el Gateway registra una advertencia al iniciarse y no ejecuta Heartbeats programados; las activaciones manuales y basadas en eventos de Heartbeat siguen disponibles. No existe un temporizador de reserva independiente para Heartbeat.

Solución de problemas: [Tareas programadas](/es/automation/cron-jobs#troubleshooting)

## Inicio rápido (principiantes)

<Steps>
  <Step title="Elegir una cadencia">
    Deja habilitados los Heartbeats (el valor predeterminado es `30m`, o `1h` cuando se configura la autenticación mediante OAuth/token de Anthropic, incluida la reutilización de Claude CLI) o establece tu propia cadencia.
  </Step>
  <Step title="Añadir notas temporales del monitor (opcional)">
    Guarda una pequeña lista de comprobación en las notas temporales del monitor de Heartbeat con `openclaw cron scratch <jobId> --set "..."`.
  </Step>
  <Step title="Decidir adónde deben enviarse los mensajes de Heartbeat">
    `target: "none"` es el valor predeterminado; establece `target: "last"` para dirigirlos al último contacto.
  </Step>
  <Step title="Ajustes opcionales">
    - Usa un contexto de arranque ligero si las ejecuciones de Heartbeat solo necesitan las notas temporales del monitor.
    - Habilita sesiones aisladas para evitar enviar el historial completo de la conversación en cada Heartbeat.
    - Restringe los Heartbeats al horario activo (hora local).

  </Step>
</Steps>

Ejemplo de configuración:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // entrega explícita al último contacto (el valor predeterminado es "none")
        directPolicy: "allow", // valor predeterminado: permitir destinos directos/DM; establece "block" para suprimirlos
        lightContext: true, // opcional: omitir los archivos de arranque del espacio de trabajo en las ejecuciones de Heartbeat
        isolatedSession: true, // opcional: sesión nueva en cada ejecución (sin historial de conversación)
        // activeHours: { start: "08:00", end: "24:00" },
      },
    },
  },
}
```

## Valores predeterminados

- Intervalo: `30m`. Al aplicar los valores predeterminados del proveedor Anthropic, aumenta a `1h` cuando el modo de autenticación resuelto es OAuth/token (incluida la reutilización de Claude CLI), pero solo mientras `heartbeat.every` no esté definido. Establece `agents.defaults.heartbeat.every` o `agents.entries.*.heartbeat.every` por agente; usa `0m` para deshabilitarlo.
- Cuerpo del prompt (configurable mediante `agents.defaults.heartbeat.prompt`): `Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- Tiempo de espera: los turnos de Heartbeat sin un valor definido usan `agents.defaults.timeoutSeconds` cuando está configurado. De lo contrario, usan la cadencia de Heartbeat con un límite de 600 segundos. Establece `agents.defaults.heartbeat.timeoutSeconds` o `agents.entries.*.heartbeat.timeoutSeconds` por agente para trabajos de Heartbeat más largos.
- El prompt de Heartbeat se envía **textualmente** como mensaje del usuario. El prompt del sistema incluye una sección «Heartbeats» cuando están habilitados para el agente predeterminado, y la ejecución se marca internamente.
- Cuando los Heartbeats se deshabilitan con `0m`, el trabajo de cron del monitor permanece, pero se deshabilita, y sus notas temporales se conservan para cuando vuelvas a habilitar la cadencia.
- Cuando cron está deshabilitado, los Heartbeats programados no se ejecutan aunque la cadencia de Heartbeat siga habilitada.
- El horario activo (`heartbeat.activeHours`) se comprueba en la zona horaria configurada. Fuera de ese intervalo, los Heartbeats se omiten hasta el siguiente tick dentro del intervalo.
- Los Heartbeats se aplazan automáticamente mientras haya trabajo de cron activo o en cola, o mientras estén ocupados los subagentes vinculados a la clave de sesión de ese agente o los canales de comandos anidados. Los agentes relacionados no se pausan entre sí.

## Para qué sirve el prompt de Heartbeat

El prompt predeterminado es intencionadamente amplio:

- **Tareas en segundo plano**: «Considera las tareas pendientes» anima al agente a revisar los seguimientos (bandeja de entrada, calendario, recordatorios y trabajo en cola) y señalar cualquier asunto urgente.
- **Comprobación con la persona**: «Comprueba ocasionalmente cómo está la persona durante el día» invita a enviar de vez en cuando un breve mensaje del tipo «¿Necesitas algo?», pero evita los mensajes nocturnos mediante la zona horaria local configurada (consulta [Zona horaria](/es/concepts/timezone)).

Heartbeat puede reaccionar a [tareas en segundo plano](/es/automation/tasks) completadas, pero una ejecución de Heartbeat no crea por sí misma un registro de tarea.

Si quieres que un Heartbeat haga algo muy específico (por ejemplo, «comprobar las estadísticas de Gmail PubSub» o «verificar el estado del Gateway»), establece `agents.defaults.heartbeat.prompt` (o `agents.entries.*.heartbeat.prompt`) con un cuerpo personalizado (se envía textualmente).

## Contrato de respuesta

- Si nada requiere atención, responde con **`HEARTBEAT_OK`**.
- En su lugar, las ejecuciones de Heartbeat pueden llamar a `heartbeat_respond` con `notify: false` para no mostrar ninguna actualización visible, o a `notify: true` junto con `notificationText` para emitir una alerta. Cuando existe, la respuesta estructurada de la herramienta tiene prioridad sobre la alternativa de texto.
- Un resultado significativo de `heartbeat_respond` con `notify: false` permanece silencioso, pero se recuerda como contexto interno acotado para el siguiente turno del usuario en esa sesión. Las confirmaciones de `no_change` y las notificaciones visibles no se almacenan de esta forma.
- Durante las ejecuciones de Heartbeat, OpenClaw considera `HEARTBEAT_OK` una confirmación cuando aparece al **principio o al final** de la respuesta. El token se elimina y la respuesta se descarta si el contenido restante tiene como máximo 300 caracteres.
- Si `HEARTBEAT_OK` aparece en el **medio** de una respuesta, no recibe ningún tratamiento especial.
- Para las alertas, **no** incluyas `HEARTBEAT_OK`; devuelve únicamente el texto de la alerta.

Fuera de los Heartbeats, cualquier `HEARTBEAT_OK` aislado al principio o al final de un mensaje se elimina y se registra; los mensajes que solo contienen `HEARTBEAT_OK` se descartan.

## Configuración

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // valor predeterminado: 30m (0m lo deshabilita)
        model: "anthropic/claude-opus-4-6",
        lightContext: false, // valor predeterminado: false; true omite los archivos de arranque del espacio de trabajo en las ejecuciones de Heartbeat
        isolatedSession: false, // valor predeterminado: false; true ejecuta cada Heartbeat en una sesión nueva (sin historial de conversación)
        target: "last", // valor predeterminado: none | opciones: last | none | <channel id> (núcleo o plugin, p. ej., "imessage")
        to: "+15551234567", // anulación opcional específica del canal
        accountId: "ops-bot", // id. opcional del canal con varias cuentas
        prompt: "Sigue el contexto de las notas temporales del monitor de Heartbeat cuando se proporcione. Las tareas recurrentes son trabajos de cron; crea o modifica sus programaciones con las herramientas de cron o la CLI de cron de openclaw, no con las notas temporales de Heartbeat. No deduzcas ni repitas tareas antiguas de chats anteriores. Si nada requiere atención, responde HEARTBEAT_OK.",
      },
    },
  },
}
```

### Alcance y precedencia

- `agents.defaults.heartbeat` establece el comportamiento global de Heartbeat.
- `agents.entries.*.heartbeat` se combina por encima; si algún agente tiene un bloque `heartbeat`, **solo esos agentes** ejecutan Heartbeats.
- `channels.defaults.heartbeatVisibility` establece los valores predeterminados de visibilidad para todos los canales.
- `channels.<channel>.heartbeatVisibility` anula los valores predeterminados del canal.
- `channels.<channel>.accounts.<id>.heartbeatVisibility` (canales con varias cuentas) anula la configuración de cada canal.

### Heartbeats por agente

Si alguna entrada de `agents.entries.*` incluye un bloque `heartbeat`, **solo esos agentes** ejecutan Heartbeats. El bloque por agente se combina por encima de `agents.defaults.heartbeat` (por lo que se pueden establecer una vez los valores predeterminados compartidos y anularlos para cada agente).

Ejemplo: dos agentes; solo el segundo ejecuta Heartbeats.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // entrega explícita al último contacto (el valor predeterminado es "none")
      },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567",
          timeoutSeconds: 45,
          prompt: "Sigue el contexto de las notas temporales del monitor de Heartbeat cuando se proporcione. Las tareas recurrentes son trabajos de cron; crea o modifica sus programaciones con las herramientas de cron o la CLI de cron de openclaw, no con las notas temporales de Heartbeat. No deduzcas ni repitas tareas antiguas de chats anteriores. Si nada requiere atención, responde HEARTBEAT_OK.",
        },
      },
    ],
  },
}
```

### Ejemplo de horario activo

Restringe los Heartbeats al horario laboral de una zona horaria específica:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // entrega explícita al último contacto (el valor predeterminado es "none")
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // opcional; usa userTimezone si está definido; de lo contrario, la zona horaria del host
        },
      },
    },
  },
}
```

Fuera de este intervalo (antes de las 9 a. m. o después de las 10 p. m., hora del Este), los Heartbeats se omiten. El siguiente tick programado dentro del intervalo se ejecutará con normalidad.

### Configuración de 24/7

Si quieres que los Heartbeats se ejecuten durante todo el día, usa uno de estos patrones:

- Omite por completo `activeHours` (sin restricción de intervalo horario; este es el comportamiento predeterminado).
- Establece un intervalo de día completo: `activeHours: { start: "00:00", end: "24:00" }`.

<Warning>
No establezcas la misma hora para `start` y `end` (por ejemplo, de `08:00` a `08:00`). Se interpreta como un intervalo de amplitud cero, por lo que los Heartbeats siempre se omiten.
</Warning>

### Ejemplo con varias cuentas

Usa `accountId` para seleccionar una cuenta específica en canales con varias cuentas, como Telegram:

```json5
{
  agents: {
    list: [
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678:topic:42", // opcional: dirigir a un tema/hilo específico
          accountId: "ops-bot",
        },
      },
    ],
  },
  channels: {
    telegram: {
      accounts: {
        "ops-bot": { botToken: "YOUR_TELEGRAM_BOT_TOKEN" },
      },
    },
  },
}
```

### Notas sobre los campos

<ParamField path="every" type="string">
  Intervalo de Heartbeat (cadena de duración; unidad predeterminada = minutos).
</ParamField>
<ParamField path="model" type="string">
  Anulación opcional del modelo para las ejecuciones de Heartbeat (`provider/model`).
</ParamField>
<ParamField path="lightContext" type="boolean" default="false">
  Cuando es true, las ejecuciones de Heartbeat usan un contexto de arranque ligero y omiten los archivos de arranque del espacio de trabajo. Las notas temporales del monitor son insertadas por el ejecutor de Heartbeat en cualquier caso.
</ParamField>
<ParamField path="isolatedSession" type="boolean" default="false">
  Cuando es true, cada Heartbeat se ejecuta en una sesión nueva sin historial de conversación anterior. Usa el mismo patrón de aislamiento que `sessionTarget: "isolated"` de cron. Reduce considerablemente el coste de tokens por Heartbeat. Combínalo con `lightContext: true` para obtener el máximo ahorro. El enrutamiento de la entrega sigue usando el contexto de la sesión principal.
</ParamField>
<ParamField path="session" type="string">
  Clave de sesión opcional para las ejecuciones de Heartbeat.

- `main` (valor predeterminado): sesión principal del agente.
- Clave de sesión explícita (cópiala de `openclaw sessions --json` o de la [CLI de sesiones](/es/cli/sessions)).
- Formatos de las claves de sesión: consulta [Sesiones](/es/concepts/session) y [Grupos](/es/channels/groups).

</ParamField>
<ParamField path="target" type="string">
- `last`: entregar en el último canal externo utilizado.
- canal explícito: cualquier canal configurado o id de Plugin, por ejemplo `discord`, `matrix`, `telegram` o `whatsapp`.
- `none` (valor predeterminado): ejecutar el Heartbeat, pero **no realizar ninguna entrega** externa.

</ParamField>
<ParamField path="directPolicy" type='"allow" | "block"' default="allow">
  Controla el comportamiento de entrega directa/por MD. `allow`: permitir la entrega directa/por MD del Heartbeat. `block`: suprimir la entrega directa/por MD (`reason=dm-blocked`).

</ParamField>
<ParamField path="to" type="string">
  Reemplazo opcional del destinatario (id específico del canal, por ejemplo, E.164 para WhatsApp o un id de chat de Telegram). Para temas/hilos de Telegram, usar `<chatId>:topic:<messageThreadId>`.

</ParamField>
<ParamField path="accountId" type="string">
  Id de cuenta opcional para canales con varias cuentas. Cuando se usa `target: "last"`, el id de cuenta se aplica al último canal resuelto si este admite cuentas; de lo contrario, se ignora. Si el id de cuenta no coincide con una cuenta configurada para el canal resuelto, se omite la entrega.

</ParamField>
<ParamField path="prompt" type="string">
  Reemplaza el cuerpo predeterminado del prompt (no se combina).

</ParamField>
<ParamField path="timeoutSeconds" type="number" default="global timeout or min(every, 600)">
  Cantidad máxima de segundos permitida para un turno del agente de Heartbeat antes de que se cancele. Dejar sin establecer para usar `agents.defaults.timeoutSeconds` cuando esté configurado; de lo contrario, se usa la cadencia del Heartbeat, limitada a 600 segundos.

</ParamField>
<ParamField path="activeHours" type="object">
  Restringe las ejecuciones del Heartbeat a una franja horaria. Objeto con `start` (HH:MM, inclusivo; usar `00:00` para el inicio del día), `end` (HH:MM, exclusivo; se permite `24:00` para el final del día) y `timezone` opcional.

- Si se omite o es `"user"`: usa `agents.defaults.userTimezone` si está configurado; de lo contrario, recurre a la zona horaria del sistema anfitrión.
- `"local"`: siempre usa la zona horaria del sistema anfitrión.
- Cualquier identificador de IANA (por ejemplo, `America/New_York`): se usa directamente; si no es válido, se recurre al comportamiento de `"user"` descrito anteriormente.
- `start` y `end` no deben ser iguales para una ventana activa; los valores iguales se consideran de amplitud cero (siempre fuera de la ventana).
- Fuera de la ventana activa, los Heartbeats se omiten hasta el siguiente ciclo que esté dentro de la ventana.

</ParamField>

## Comportamiento de entrega

<AccordionGroup>
  <Accordion title="Enrutamiento de sesiones y destinos">
    - De forma predeterminada, los Heartbeats se ejecutan en la sesión principal del agente (`agent:<id>:<mainKey>`), o en `global` cuando se usa `session.scope = "global"`. Establecer `session` para reemplazarla por una sesión de canal específica (Discord/WhatsApp/etc.).
    - `session` solo afecta al contexto de ejecución; la entrega está controlada por `target` y `to`.
    - Para entregar a un canal/destinatario específico, establecer `target` + `to`. Con `target: "last"`, la entrega usa el último canal externo de esa sesión.
    - De forma predeterminada, las entregas de Heartbeat permiten destinos directos/por MD. Establecer `directPolicy: "block"` para suprimir los envíos a destinos directos sin dejar de ejecutar el turno del Heartbeat.
    - Si la cola principal, la vía de la sesión de destino, la vía de Cron o un trabajo de Cron activo están ocupados, el Heartbeat se omite y vuelve a intentarse más tarde.
    - Si `target` no se resuelve en ningún destino externo, la ejecución sigue produciéndose, pero no se envía ningún mensaje saliente.

  </Accordion>
  <Accordion title="Visibilidad y comportamiento de omisión">
    - Si `showOk`, `showAlerts` y `useIndicator` están deshabilitados, la ejecución se omite desde el principio como `reason=alerts-disabled`.
    - Si solo está deshabilitada la entrega de alertas, OpenClaw aún puede ejecutar el Heartbeat, actualizar las marcas de tiempo de las tareas vencidas, restaurar la marca de tiempo de inactividad de la sesión y suprimir la carga útil de la alerta externa.
    - Si el destino resuelto del Heartbeat admite indicadores de escritura, OpenClaw muestra que se está escribiendo mientras la ejecución del Heartbeat está activa. Para ello se usa el mismo destino al que el Heartbeat enviaría la salida del chat, y `typingMode: "never"` lo deshabilita.

  </Accordion>
  <Accordion title="Ciclo de vida y auditoría de la sesión">
    - Las respuestas exclusivas del Heartbeat **no** mantienen activa la sesión. Los metadatos del Heartbeat pueden actualizar la fila de la sesión, pero el vencimiento por inactividad usa `lastInteractionAt` del último mensaje real del usuario/canal, y el vencimiento diario usa `sessionStartedAt`.
    - El historial de la interfaz de control y WebChat oculta los prompts del Heartbeat y las confirmaciones que solo contienen OK. La transcripción subyacente de la sesión aún puede contener esos turnos para fines de auditoría/reproducción.
    - Las [tareas en segundo plano](/es/automation/tasks) independientes pueden poner en cola un evento del sistema y activar el Heartbeat cuando la sesión principal deba advertir algo rápidamente. Esa activación no hace que la ejecución del Heartbeat sea una tarea en segundo plano.

  </Accordion>
</AccordionGroup>

## Controles de visibilidad

De forma predeterminada, las confirmaciones `HEARTBEAT_OK` se suprimen mientras se entrega el contenido de las alertas. Esto puede ajustarse por canal o por cuenta:

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false # Ocultar HEARTBEAT_OK (valor predeterminado)
      showAlerts: true # Mostrar mensajes de alerta (valor predeterminado)
      useIndicator: true # Emitir eventos de indicador (valor predeterminado)
  telegram:
    heartbeat:
      showOk: true # Mostrar confirmaciones OK en Telegram
  whatsapp:
    accounts:
      work:
        heartbeat:
          showAlerts: false # Suprimir la entrega de alertas para esta cuenta
```

Precedencia: por cuenta → por canal → valores predeterminados del canal → valores predeterminados integrados.

### Función de cada indicador

- `showOk`: envía una confirmación `HEARTBEAT_OK` cuando el modelo devuelve una respuesta que solo contiene OK.
- `showAlerts`: envía el contenido de la alerta cuando el modelo devuelve una respuesta que no contiene solo OK.
- `useIndicator`: emite eventos de indicador para las superficies de estado de la interfaz de usuario.

Si **los tres** son falsos, OpenClaw omite por completo la ejecución del Heartbeat (sin llamada al modelo).

### Ejemplos por canal y por cuenta

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true
  slack:
    heartbeat:
      showOk: true # todas las cuentas de Slack
    accounts:
      ops:
        heartbeat:
          showAlerts: false # suprimir alertas solo para la cuenta de operaciones
  telegram:
    heartbeat:
      showOk: true
```

### Patrones habituales

| Objetivo                                     | Configuración                                                                                   |
| ---------------------------------------- | ---------------------------------------------------------------------------------------- |
| Comportamiento predeterminado (OK silenciosos, alertas activadas) | _(no se necesita configuración)_                                                                     |
| Totalmente silencioso (sin mensajes ni indicador) | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: false }` |
| Solo indicador (sin mensajes)             | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: true }`  |
| OK solo en un canal                  | `channels.telegram.heartbeat: { showOk: true }`                                          |

## Documento auxiliar del monitor (opcional)

Cada trabajo de Cron del monitor de Heartbeat posee un documento auxiliar privado almacenado en la base de datos de estado compartida. Puede considerarse una «lista de comprobación del Heartbeat»: pequeña, estable y segura para consultarla cada 30 minutos. Cuando existe el documento auxiliar, su contenido se añade al prompt del Heartbeat.

Se administra con la CLI de Cron (el id del trabajo procede de `openclaw cron list --all`):

```bash
openclaw cron scratch <jobId>                 # mostrar el documento auxiliar actual
openclaw cron scratch <jobId> --set "..."     # reemplazarlo con el texto exacto
openclaw cron scratch <jobId> --file notes.md # reemplazarlo desde un archivo (- para stdin)
openclaw cron scratch <jobId> --unset         # eliminarlo
```

Las escrituras están protegidas mediante comparación e intercambio: pasar `--expected-revision <n>` para que se produzca un error en lugar de sobrescribir una edición simultánea. El documento auxiliar está limitado a 256 KiB y nunca aparece en la salida de `cron list`/`cron runs`.

El agente también puede actualizar su propio documento auxiliar: durante un turno del Heartbeat, `heartbeat_respond` acepta una cadena `scratch` opcional que reemplaza por completo el documento auxiliar del monitor para futuros Heartbeats.

<Note>
**¿Se está migrando desde HEARTBEAT.md o desde una cadencia definida únicamente en la configuración?** Ejecutar `openclaw doctor --fix`. Doctor primero crea o actualiza las filas del monitor propiedad del sistema a partir de `agents.*.heartbeat`; después importa el archivo `HEARTBEAT.md` del espacio de trabajo de cada agente en el documento auxiliar del monitor, convierte todas las entradas heredadas válidas de `tasks:` en trabajos de Cron, archiva el original en el directorio de estado (`backups/heartbeat-migration/`) y elimina el archivo. Las instrucciones del Heartbeat en tiempo de ejecución proceden únicamente del documento auxiliar de la base de datos; el entorno de ejecución nunca lee `HEARTBEAT.md`.
</Note>

Si el documento auxiliar existe, pero está prácticamente vacío (solo contiene líneas en blanco, comentarios de Markdown/HTML, encabezados de Markdown como `# Heading`, marcadores de bloques delimitados o elementos vacíos de una lista de comprobación), OpenClaw omite la ejecución del Heartbeat para ahorrar llamadas a la API. Esa omisión se registra como `reason=empty-heartbeat-file`. Si no existe ningún documento auxiliar, el Heartbeat sigue ejecutándose y el modelo decide qué hacer.

Debe mantenerse reducido (una lista de comprobación breve o recordatorios) para evitar que el prompt crezca innecesariamente.

Ejemplo de documento auxiliar:

```md
# Lista de comprobación del Heartbeat

- Revisión rápida: ¿hay algo urgente en las bandejas de entrada?
- Si es de día y no hay nada más pendiente, realizar una comprobación ligera.
- Si una tarea está bloqueada, anotar _qué falta_ y preguntar a Peter la próxima vez.
```

### Programación de comprobaciones recurrentes con Cron

El documento auxiliar del Heartbeat es contexto del prompt, no un programador. Crear cada comprobación recurrente como un [trabajo de Cron](/es/automation/cron-jobs) para que tenga su propia cadencia, estado de activación/desactivación e historial de ejecuciones. Los trabajos de Cron pueden seguir apuntando a la sesión principal cuando la comprobación deba usar el contexto normal de la conversación.

Los documentos auxiliares antiguos pueden contener un bloque estructurado `tasks:`. Ejecutar `openclaw doctor --fix` una vez después de actualizar: Doctor convierte cada entrada válida en un trabajo de Cron programado de forma independiente, conserva su intervalo y el momento de la última ejecución anterior, y elimina el bloque retirado sin alterar el texto circundante del documento auxiliar. Los turnos del Heartbeat en tiempo de ejecución no interpretan el texto `tasks:` como una programación.

Los trabajos de tareas del Heartbeat creados por Doctor conservan las horas activas, el tiempo de espera, la protección contra inundaciones y las protecciones ante ocupación del Heartbeat. Los trabajos que vencen al mismo tiempo pueden agruparse en un solo turno del Heartbeat. Las ocurrencias fuera del horario activo se omiten y vuelven a intentarse en su siguiente ocurrencia de Cron.

### ¿Puede el agente actualizar su documento auxiliar?

Sí. Durante un turno del Heartbeat, el agente puede pasar un valor `scratch` a `heartbeat_respond` para reemplazar por completo el texto del monitor para futuros Heartbeats. También se le puede pedir en un chat normal que ejecute `openclaw cron scratch <jobId> --set ...`, o editar el documento auxiliar con el mismo comando. Las programaciones recurrentes deben administrarse con Cron en lugar de escribir sintaxis de programación en el documento auxiliar.

<Warning>
No se deben incluir secretos (claves de API, números de teléfono ni tokens privados) en el documento auxiliar del monitor: pasan a formar parte del contexto del prompt.
</Warning>

## Activación manual (bajo demanda)

Usar `openclaw system event` para poner en cola un evento del sistema y, de forma opcional, activar inmediatamente un Heartbeat:

```bash
openclaw system event --text "Comprobar si hay seguimientos urgentes" --mode now
```

| Indicador                    | Descripción                                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------------------ |
| `--text <text>`              | Texto del evento del sistema (obligatorio).                                                      |
| `--mode <mode>`              | `now` ejecuta un Heartbeat inmediato; `next-heartbeat` (valor predeterminado) espera al siguiente ciclo programado. |
| `--session-key <sessionKey>` | Dirige el evento a una sesión específica; de forma predeterminada, se usa la sesión principal del agente. |
| `--json`                     | Genera la salida en formato JSON.                                                                |

Si no se proporciona `--session-key` y varios agentes tienen `heartbeat` configurado, `--mode now` ejecuta inmediatamente el Heartbeat de cada uno de esos agentes.

Controles de Heartbeat relacionados en el mismo grupo de la CLI:

```bash
openclaw system heartbeat last     # muestra el último evento de Heartbeat
openclaw system heartbeat enable   # habilita los Heartbeats
openclaw system heartbeat disable  # deshabilita los Heartbeats
```

## Consideraciones sobre los costes

Los Heartbeats ejecutan turnos completos del agente. Los intervalos más cortos consumen más tokens. Para reducir los costes:

- Utilice `isolatedSession: true` para evitar enviar el historial completo de la conversación (de ~100K tokens a ~2-5K por ejecución).
- Utilice `lightContext: true` para omitir los archivos de inicialización del espacio de trabajo en las ejecuciones de Heartbeat.
- Configure un `model` más económico (p. ej., `ollama/llama3.2:1b`).
- Mantenga pequeño el espacio temporal del monitor.
- Utilice `target: "none"` si solo desea actualizar el estado interno.

## Desbordamiento de contexto después de un Heartbeat

Los Heartbeats conservan el modelo de ejecución existente de la sesión compartida una vez finalizada la ejecución, por lo que un Heartbeat que haya cambiado una sesión a un modelo local más pequeño (por ejemplo, un modelo de Ollama con una ventana de 32k) puede dejar ese modelo activo para el siguiente turno de la sesión principal. Si ese turno informa de un desbordamiento de contexto y el último modelo de ejecución de la sesión coincide con el `heartbeat.model` configurado, el mensaje de recuperación de OpenClaw señala como causa probable la propagación del modelo del Heartbeat y sugiere una solución.

Para evitarlo: utilice `isolatedSession: true` para ejecutar los Heartbeats en una sesión nueva (opcionalmente combinado con `lightContext: true` para obtener el prompt más pequeño), o elija un modelo de Heartbeat con una ventana de contexto lo bastante grande para la sesión compartida.

## Contenido relacionado

- [Automatización](/es/automation) - todos los mecanismos de automatización de un vistazo
- [Tareas en segundo plano](/es/automation/tasks) - cómo se realiza el seguimiento del trabajo separado
- [Zona horaria](/es/concepts/timezone) - cómo afecta la zona horaria a la programación de Heartbeat
- [Solución de problemas](/es/automation/cron-jobs#troubleshooting) - depuración de problemas de automatización
