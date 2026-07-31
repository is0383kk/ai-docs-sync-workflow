---
read_when:
    - Explicación de cómo los mensajes entrantes se convierten en respuestas
    - Aclaración de las sesiones, los modos de cola o el comportamiento de transmisión
    - Documentación de la visibilidad del razonamiento y sus implicaciones de uso
summary: Flujo de mensajes, sesiones, colas y visibilidad del razonamiento
title: Mensajes
x-i18n:
    generated_at: "2026-07-26T04:38:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e42bed834e9a57fb8a248c8654b75ea9977928582f68a83859cf6c16ed0b6bf5
    source_path: concepts/messages.md
    workflow: 16
---

Los mensajes entrantes pasan por el enrutamiento, la deduplicación/antirrebote, una ejecución del agente y la entrega saliente:

```text
Mensaje entrante
  -> enrutamiento/vinculaciones -> clave de sesión
  -> deduplicación + antirrebote
  -> cola (si ya hay una ejecución activa)
  -> ejecución del agente (transmisión + herramientas)
  -> respuestas salientes (límites del canal + fragmentación)
```

Principales superficies de configuración:

- `messages.*` para prefijos, puesta en cola, antirrebote de entrada y comportamiento de grupos.
- `agents.defaults.*` para transmisión por bloques, fragmentación y valores predeterminados de respuestas silenciosas.
- Anulaciones de canal (`channels.telegram.*`, `channels.whatsapp.*`, etc.) para límites y controles de transmisión específicos de cada canal.

Consulte [Configuración](/es/gateway/configuration) para ver el esquema completo.

## Deduplicación de entrada

Los canales pueden volver a entregar el mismo mensaje después de una reconexión. OpenClaw mantiene una caché en memoria cuya clave se compone del ámbito del agente, la ruta del canal (canal + interlocutor + cuenta + hilo) y el identificador del mensaje, de modo que un mensaje vuelto a entregar no activa una segunda ejecución del agente. La entrada de caché caduca después de 20 minutos o cuando se alcanzan 5000 entradas, lo que ocurra primero.

## Antirrebote de entrada

Los mensajes de texto consecutivos enviados rápidamente por el mismo remitente pueden agruparse en un solo turno del agente mediante `messages.inbound`. El antirrebote se aplica por canal + conversación y utiliza el mensaje más reciente para el hilo y los identificadores de respuesta.

```json5
{
  messages: {
    inbound: {
      debounceMs: 2000,
      byChannel: {
        discord: 1500,
        slack: 1500,
        whatsapp: 5000,
      },
    },
  },
}
```

- El antirrebote se aplica únicamente a mensajes de texto; los archivos multimedia/adjuntos se procesan de inmediato.
- Los comandos de control (detener/abortar/estado, etc.) omiten el antirrebote para enviarse de inmediato.
- Desactivado de forma predeterminada: `messages.inbound.debounceMs` no tiene ningún valor predeterminado incorporado, por lo que el antirrebote solo se activa cuando se configura (globalmente o por canal).
- iMessage sigue la misma política genérica de antirrebote. `imsg` 0.13.1 y versiones posteriores combina los envíos divididos por la vista previa de URL de Apple antes de que OpenClaw los reciba, por lo que no se necesita ninguna configuración de antirrebote específica para iMessage.

## Sesiones y dispositivos

Las sesiones pertenecen al Gateway, no a los clientes.

- Los chats directos se consolidan en la clave de sesión principal del agente.
- Los grupos/canales obtienen sus propias claves de sesión.
- El almacén de sesiones y las transcripciones residen en el host del Gateway.

Varios dispositivos/canales pueden asignarse a la misma sesión, pero el historial no se vuelve a sincronizar por completo con todos los clientes. Se recomienda usar un dispositivo principal para las conversaciones largas a fin de evitar contextos divergentes. La interfaz de control y la TUI siempre muestran la transcripción de sesión respaldada por el Gateway, por lo que constituyen la fuente de información fidedigna.

Detalles: [Gestión de sesiones](/es/concepts/session).

## Cuerpos de prompts y contexto del historial

Los plugins de canal rellenan varios campos de texto en el contexto entrante, del más al menos preferido:

| Campo             | Propósito                                                                                                     |
| ----------------- | ----------------------------------------------------------------------------------------------------------- |
| `BodyForAgent`    | Texto orientado al modelo para el turno actual. Si no está definido, recurre a `CommandBody` / `RawBody` / `Body`.        |
| `BodyForCommands` | Texto limpio utilizado para analizar directivas/comandos. Si no está definido, recurre a `CommandBody` / `RawBody` / `Body`. |
| `CommandBody`     | Cuerpo intermedio heredado; se recomienda `BodyForCommands`.                                                         |
| `RawBody`         | Alias obsoleto de `CommandBody`.                                                                         |
| `Body`            | Cuerpo de prompt heredado; puede incluir envoltorios del canal y del historial.                                     |

Cuando un canal proporciona historial, lo envuelve con:

- `[Chat messages since your last reply - for context]`
- `[Current message - respond to this]`

En los chats no directos (grupos/canales/salas), el cuerpo del mensaje actual lleva como prefijo la etiqueta del remitente, de acuerdo con el estilo utilizado para las entradas del historial. La eliminación de directivas solo se aplica a la sección del mensaje actual, por lo que el historial permanece intacto. Los canales que envuelvan el historial deben establecer `BodyForCommands` (o los campos heredados `CommandBody` / `RawBody`) con el texto original del mensaje y conservar `Body` como prompt combinado.

Los búferes del historial solo contienen elementos pendientes: incluyen mensajes de grupo que no activaron una ejecución (por ejemplo, mensajes condicionados a una mención) y excluyen los mensajes que ya están en la transcripción de la sesión. Durante la composición del prompt, el historial estructurado y los metadatos de respuestas, reenvíos y canales se representan como bloques de contexto no fiables con el rol de usuario.

Configure el tamaño del historial mediante `messages.groupChat.historyLimit` (valor predeterminado global) o anulaciones por canal, como `channels.slack.historyLimit` y `channels.telegram.accounts.<id>.historyLimit` (establezca `0` para desactivarlo).

## Metadatos de resultados de herramientas

El `content` del resultado de una herramienta es el resultado visible para el modelo; `details` contiene metadatos de ejecución para la representación en la interfaz, el diagnóstico, la entrega de archivos multimedia y los plugins.

- `toolResult.details` se elimina antes de la reproducción para el proveedor y antes de la entrada de Compaction.
- Las transcripciones de sesión persistentes solo conservan un `details` limitado; los metadatos demasiado grandes se sustituyen por un resumen compacto marcado con `persistedDetailsTruncated: true`.
- Los plugins y las herramientas deben colocar en `content` el texto que el modelo deba leer, no solo en `details`.

## Puesta en cola y seguimientos

Cuando ya hay una ejecución activa, los mensajes entrantes se incorporan a ella de forma predeterminada. `messages.queue` controla el modo:

| Modo              | Comportamiento                                            |
| ----------------- | --------------------------------------------------- |
| `steer` (predeterminado) | Inyecta el nuevo prompt en la ejecución activa.          |
| `followup`        | Ejecuta el mensaje después de que finalice la ejecución activa.      |
| `collect`         | Agrupa mensajes compatibles en un único turno posterior.      |
| `interrupt`       | Aborta la ejecución activa y, a continuación, inicia el prompt más reciente. |

La cola utiliza un antirrebote incorporado de 500ms para la incorporación, el seguimiento y la agrupación. El valor predeterminado de `messages.queue.cap` es 20 mensajes en cola y el de `messages.queue.drop` es `summarize` (también están disponibles `old` y `new`). Configure anulaciones por canal mediante `messages.queue.byChannel` y `messages.queue.debounceMsByChannel`.

Detalles: [Cola de comandos](/es/concepts/queue) y [Cola de incorporación](/es/concepts/queue-steering).

## Propiedad de la ejecución del canal

Los plugins de canal pueden conservar el orden, aplicar antirrebote a la entrada y ejercer contrapresión de transporte antes de que un mensaje entre en la cola de la sesión. No deben imponer un tiempo de espera independiente al propio turno del agente. Una vez que un mensaje se enruta a una sesión, los ciclos de vida de la sesión, las herramientas y el entorno de ejecución controlan el trabajo prolongado, de modo que todos los canales informen sobre los turnos lentos y se recuperen de ellos de forma coherente.

## Transmisión, fragmentación y agrupación

La transmisión por bloques envía respuestas parciales a medida que el modelo genera bloques de texto; la fragmentación respeta los límites de texto del canal y evita dividir bloques de código delimitados.

- `agents.defaults.blockStreamingDefault` (`on|off`, valor predeterminado `off`)
- `agents.defaults.blockStreamingBreak` (`text_end|message_end`)
- `agents.defaults.blockStreamingChunk` (`minChars|maxChars|breakPreference`)
- `agents.defaults.blockStreamingCoalesce` (agrupación basada en inactividad)
- `agents.defaults.humanDelay` (pausa similar a la humana entre respuestas por bloques)
- Anulaciones de canal: `*.streaming.block.enabled` y `*.streaming.block.coalesce` en los canales incluidos; `openclaw doctor --fix` migra las claves planas obsoletas. La transmisión por bloques está desactivada salvo que se habilite explícitamente, en todos los canales, incluido Telegram. QQ Bot es la excepción: no tiene claves `streaming.block` y transmite respuestas por bloques salvo que `channels.qqbot.streaming.mode` sea `"off"`.

Detalles: [Transmisión + fragmentación](/es/concepts/streaming).

## Visibilidad del razonamiento y tokens

- `/reasoning on|off|stream` controla la visibilidad.
- El contenido de razonamiento sigue contando para el uso de tokens cuando el modelo lo genera.
- Telegram admite la transmisión del razonamiento en una burbuja de borrador transitoria que se elimina tras la entrega final; utilice `/reasoning on` para obtener una salida de razonamiento persistente.

Detalles: [Directivas de pensamiento + razonamiento](/es/tools/thinking) y [Uso de tokens](/es/reference/token-use).

## Prefijos, hilos y respuestas

- Los prefijos de salida se encuentran en `channels.<channel>.responsePrefix` y `channels.<channel>.accounts.<id>.responsePrefix`. Los valores de cuenta tienen prioridad. Doctor copia el valor alternativo global en los bloques de canales configurados cuando esos campos canónicos no están definidos; `messages.responsePrefix` permanece como alternativa para canales implícitos y personalizados.
- Hilos de respuesta mediante `replyToMode` y valores predeterminados por canal.

Detalles: [Configuración](/es/gateway/config-agents#messages) y la documentación de los canales.

## Respuestas silenciosas

El token silencioso `NO_REPLY` (no distingue entre mayúsculas y minúsculas, por lo que `no_reply` también coincide) significa «no entregar una respuesta visible para el usuario». Cuando un turno también tiene archivos multimedia de herramientas pendientes, como audio TTS generado, OpenClaw elimina el texto silencioso, pero sigue entregando el archivo multimedia adjunto.

La política de silencio se determina según el tipo de conversación:

- Las conversaciones directas nunca reciben indicaciones de prompt de `NO_REPLY`. Si una ejecución directa devuelve por error un token silencioso aislado, OpenClaw lo suprime en lugar de reescribirlo o entregarlo.
- Los grupos/canales permiten el silencio de forma predeterminada. En el modo de respuesta visible `message_tool`, el silencio significa que el modelo no llama a `message(action=send)`.
- La orquestación interna permite el silencio de forma predeterminada.

Los valores predeterminados se encuentran en `agents.defaults.silentReply`; `surfaces.<id>.silentReply` puede anular la política de grupo/interna por superficie.

OpenClaw también utiliza respuestas silenciosas para fallos genéricos del ejecutor interno en chats no directos, de modo que los grupos/canales no vean el texto estándar de errores del Gateway. Los fallos clasificados que incluyen instrucciones de recuperación orientadas al usuario, como avisos de autenticación ausente, límite de frecuencia o sobrecarga, sí pueden entregarse. Los chats directos muestran de forma predeterminada un mensaje de fallo conciso; los detalles sin procesar del ejecutor solo se muestran cuando `/verbose full` está habilitado.

Las respuestas silenciosas aisladas se descartan en todas las superficies, de modo que las sesiones principales permanezcan en silencio en lugar de reescribir el texto centinela como conversación alternativa.

## Temas relacionados

- [Refactorización del ciclo de vida de los mensajes](/es/concepts/message-lifecycle-refactor) - diseño objetivo duradero para el envío y la recepción
- [Transmisión](/es/concepts/streaming) - entrega de mensajes en tiempo real
- [Reintentos](/es/concepts/retry) - comportamiento de los reintentos de entrega de mensajes
- [Cola](/es/concepts/queue) - cola de procesamiento de mensajes
- [Canales](/es/channels) - integraciones con plataformas de mensajería
