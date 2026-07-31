---
read_when:
    - Refactorización del comportamiento de envío o recepción del canal
    - Cambio de las API de mensajes del SDK de plugins, la transmisión de vistas previas, la cola de salida, el envío de respuestas o la entrada de canales
    - Diseño de un nuevo plugin de canal que requiere envíos duraderos, confirmaciones de recepción, vistas previas, ediciones o reintentos
summary: 'Estado del ciclo de vida duradero de recepción y envío de mensajes: qué se publicó, qué cambió respecto al diseño original y qué sigue pendiente'
title: Refactorización del ciclo de vida de los mensajes
x-i18n:
    generated_at: "2026-07-26T05:37:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d21eda70b8be0de78677f4ff6d7547317112731d9e86a5bef58eac0268899818
    source_path: concepts/message-lifecycle-refactor.md
    workflow: 16
---

<Note>
Esta página se originó como una propuesta de diseño con visión de futuro. Desde entonces, el núcleo de ese
diseño se ha publicado en `src/channels/message/*` y las subrutas públicas
`openclaw/plugin-sdk/channel-outbound` / `channel-inbound`. Para consultar la
API actual, use [API de salida de canales](/es/plugins/sdk-channel-outbound) y
[API de entrada de canales](/es/plugins/sdk-channel-inbound). Esta página registra qué
se publicó, dónde se desvió la implementación del boceto original y qué
sigue pendiente.
</Note>

## Por qué se realizó esta refactorización

La pila de canales surgió de varias correcciones locales: asistentes de entrada separados por
nivel de madurez (`runtime.channel.inbound.run` para adaptadores sencillos,
`runtime.channel.inbound.runPreparedReply` para los avanzados), asistentes heredados de envío de respuestas
(`dispatchInboundReplyWithBase`, `recordInboundSessionAndDispatchReply`),
streaming de vistas previas específico de cada canal y durabilidad de la entrega final añadida
a las rutas existentes de cargas útiles de respuesta. Esa estructura generó demasiados conceptos públicos y
demasiados lugares donde la semántica de entrega podía divergir.

La brecha de fiabilidad que obligó al rediseño:

```text
Actualización de sondeo de Telegram confirmada
  -> existe el texto final del asistente
  -> el proceso se reinicia antes de que sendMessage se complete correctamente
  -> se pierde la respuesta final
```

Invariante objetivo: una vez que el núcleo decide que debe existir un mensaje saliente visible,
la intención de envío debe ser duradera antes de intentar la llamada a la plataforma, y el
recibo de la plataforma debe confirmarse después del éxito. Esto proporciona una recuperación de al menos una vez
de forma predeterminada. El comportamiento de exactamente una vez solo existe cuando un adaptador demuestra
idempotencia nativa o coteja un intento de estado desconocido tras el envío con
el estado de la plataforma antes de repetirlo.

## Lo que se publicó

El dominio interno se encuentra en `src/channels/message/*`:

| Archivo                        | Responsabilidad                                                                                                      |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------- |
| `types.ts`                  | Contratos de tipos de adaptador, contexto de envío, recibo e intención duradera                                      |
| `send.ts`                   | `withDurableMessageSendContext` / `sendDurableMessageBatch`: el contexto de envío duradero                           |
| `receive.ts`                | `createMessageReceiveContext`: máquina de estados de la política de confirmación de entrada                          |
| `live.ts`                   | Estado de vista previa en vivo y lógica para finalizar en el mismo lugar o recurrir a una alternativa               |
| `state.ts`                  | `classifyDurableSendRecoveryState`: clasificación de recuperación tras una interrupción                               |
| `receipt.ts`                | Normaliza los resultados de envío de la plataforma en `MessageReceipt`                                               |
| `capabilities.ts`           | Deriva de una carga útil las capacidades necesarias para una finalización duradera                                    |
| `contracts.ts`              | Verificación mediante prueba de contrato de las capacidades declaradas por el adaptador                               |
| `adapter.ts`                | `defineChannelMessageAdapter`                                                                                      |
| `outbound-bridge.ts`        | `createChannelMessageAdapterFromOutbound`: encapsula las funciones heredadas `sendText`/`sendMedia`/`sendPayload`/`sendPoll` |
| `ingress-queue.ts`          | `createChannelIngressQueue`: cola duradera de eventos de entrada                                                  |
| `durable-receive.ts`        | `createDurableInboundReceiveJournal`: registro de aceptación/pendiente/finalización/liberación para la desduplicación de entradas |
| `inbound-reply-dispatch.ts` | `dispatchChannelInboundReply` y encapsuladores con nombres heredados                                                |
| `reply-pipeline.ts`         | `createChannelReplyPipeline`, prefijo de respuesta y asistentes de devolución de llamada de escritura                         |

Superficie pública: `openclaw/plugin-sdk/channel-outbound` (asistentes de envío/recibo/durabilidad/en vivo/Pipeline de respuestas)
y `openclaw/plugin-sdk/channel-inbound` (contexto de entrada, `runChannelInboundEvent`,
`dispatchChannelInboundReply`). Consulte esas páginas para ver ejemplos de adaptadores, los nombres
de tipos actuales y las notas de migración: son la fuente de referencia de la forma de la API,
no los bocetos siguientes.

### Contexto de envío

`withDurableMessageSendContext` proporciona al código del canal los pasos `render`, `previewUpdate`,
`send`, `edit`, `delete`, `commit` y `fail` en torno a un mensaje
saliente. `sendDurableMessageBatch` es el encapsulador para el caso habitual: renderizar, enviar
y luego confirmar con `sent`/`suppressed` o marcar como fallido si se produce un error.

`sendDurableMessageBatch` devuelve un resultado discriminado:

| Estado           | Significado                                                                      |
| ---------------- | -------------------------------------------------------------------------------- |
| `sent`           | Se entregó al menos un mensaje visible de la plataforma                          |
| `suppressed`     | Ningún mensaje de la plataforma debe considerarse ausente (cancelado por un hook, simulación, etc.) |
| `partial_failed` | Se entregó al menos un mensaje antes de que fallara una carga útil o un efecto secundario posterior |
| `failed`         | No se produjo ningún recibo de la plataforma                                     |

La durabilidad es `required`, `best_effort` o `disabled`
(`MessageDurabilityPolicy` en `src/channels/message/types.ts`). `required`
aplica un cierre seguro cuando no se puede escribir la intención duradera; `best_effort` pasa
a un envío directo cuando la persistencia no está disponible; `disabled` conserva el
comportamiento de envío directo anterior a la refactorización. Los asistentes de compatibilidad heredados usan
`disabled` de forma predeterminada y no deducen `required` solo porque un canal tenga un adaptador
de salida genérico.

El límite que sigue siendo peligroso se encuentra después de que la llamada a la plataforma se complete correctamente y antes
de que se confirme el recibo. Si el proceso termina en ese punto, el núcleo no puede saber si el
mensaje de la plataforma existe, a menos que el adaptador declare `reconcileUnknownSend`.
Ese hook clasifica un envío interrumpido como `sent`, `not_sent` o
`unresolved`; solo `not_sent` permite repetirlo. Los canales sin cotejo
recurren al estado `unknown_after_send` (`src/channels/message/state.ts`,
`src/infra/outbound/delivery-queue-recovery.ts`) y pueden optar por repetir el envío
al menos una vez solo si los mensajes visibles duplicados constituyen una contrapartida aceptable y documentada
para ese canal.

### Contexto de recepción

`createMessageReceiveContext` realiza un seguimiento del estado de confirmación/rechazo de cada evento de entrada con una
operación `ack()` idempotente y una operación `nack(error)` explícita. La política de confirmación
(`ChannelMessageReceiveAckPolicy`) es una de las siguientes:

| Política                 | Cuándo confirma                                                                                |
| ------------------------ | ---------------------------------------------------------------------------------------------- |
| `after_receive_record` | El núcleo ha conservado suficientes metadatos de entrada para desduplicar o enrutar una nueva entrega |
| `after_agent_dispatch` | Se ha enviado la ejecución del agente                                                          |
| `after_durable_send`   | Se ha confirmado el envío saliente duradero de este turno                                      |
| `manual`               | El llamador controla explícitamente el momento de la confirmación (valor predeterminado para los adaptadores que no declaran una política) |

El sondeo de Telegram usa esto para conservar una marca de agua segura de actualizaciones completadas
(`safeCompletedUpdateId` en `extensions/telegram/src/bot-update-tracker.ts`):
grammY sigue observando cada actualización cuando entra en la cadena de middleware, pero
OpenClaw solo hace avanzar la marca de agua de reinicio conservada más allá de las actualizaciones que
terminaron su envío, por lo que las actualizaciones fallidas o aún pendientes se repiten tras un reinicio.
El desplazamiento `getUpdates` ascendente de Telegram sigue siendo responsabilidad de grammY; no se ha creado
una fuente de sondeo completamente duradera que controle las nuevas entregas a nivel de plataforma más allá de esta
marca de agua (consulte Preguntas pendientes).

### Vista previa en vivo

`src/channels/message/live.ts` modela la vista previa, la edición y la finalización como un único ciclo de vida:
`createLiveMessageState`, `markLiveMessagePreviewUpdated`,
`markLiveMessageFinalized`, `markLiveMessageCancelled` y
`deliverFinalizableLivePreviewAdapter` (crear una edición final a partir de un borrador, aplicarla
y recurrir a un envío normal cuando la edición no sea posible o falle).
`LiveMessageState.phase` es `idle | previewing | finalizing | finalized |
cancelled`; `canFinalizeInPlace` determina si una vista previa puede convertirse en el mensaje
final mediante una edición en lugar de un nuevo envío.

### Recibos duraderos

`MessageReceipt` (`src/channels/message/types.ts`) normaliza uno o varios
identificadores de mensajes de la plataforma procedentes de un único envío lógico en `platformMessageIds`, además de
`parts` por parte (tipo, índice, identificador de hilo, identificador de respuesta). Se conserva un identificador principal
para los hilos y las ediciones posteriores. Esto permite que las entregas de varias partes (texto
y contenido multimedia, texto fragmentado, alternativa de tarjeta) puedan repetirse y desduplicarse después
de un reinicio.

### Reducción del SDK público

La refactorización incorporó o dejó obsoletos: `reply-runtime`, `reply-dispatch-runtime`,
`reply-reference`, `reply-chunking`, los asistentes `reply-payload` expuestos como API
pública, `inbound-reply-dispatch`, `channel-reply-pipeline` y la mayoría de los usos públicos
de la antigua fachada de salida. `src/plugin-sdk/channel-message.ts` ahora es un
módulo de reexportación `@deprecated` que apunta a `channel-outbound` /
`channel-inbound`; se eliminaron los alias de tiempo de ejecución `channel.turn` y la antigua
página de documentación `/plugins/sdk-channel-turn` redirige a
[API de entrada de canales](/es/plugins/sdk-channel-inbound). El nuevo código de plugins debe
usar directamente `channel-outbound` y `channel-inbound`.

## En qué se desvió la implementación del diseño original

El boceto de diseño siguiente nunca se publicó literalmente como se describe. Se conserva como registro para
mantener la precisión histórica; no considere estos nombres de tipos como parte de la API actual.

- **No existen `MessageOrigin` ni `shouldDropOpenClawEcho`.** El plan original requería
  una etiqueta de origen `source: "openclaw"` en los mensajes de error del Gateway, además de un
  predicado compartido que descartara los ecos etiquetados generados por bots en salas compartidas
  antes de la autorización `allowBots`. Ese tipo y ese predicado no existen en
  el código base. `allowBots` sí es una clave de configuración real por canal (Slack,
  Discord, Google Chat y otros), pero nunca se creó el mecanismo de etiquetado de origen que debía
  protegerla. La supresión de ecos de errores del Gateway en
  salas con bots habilitados sigue siendo una carencia pendiente, no una garantía publicada.
- **No existe un espacio de nombres `core.messages.receive/send/live/state` unificado.** Las
  funciones publicadas se encuentran directamente en `src/channels/message/*`
  (`withDurableMessageSendContext`, `createMessageReceiveContext`,
  `createLiveMessageState`, `classifyDurableSendRecoveryState`) en lugar de estar
  detrás de una fachada `core.messages.*`.
- **No existe un tipo de mensaje normalizado genérico `ChannelMessage` / `MessageTarget` / `MessageRelation`.**
  El núcleo sigue pasando cargas útiles de respuesta concretas
  (`ReplyPayload`) y contextos específicos del canal a través de los adaptadores de envío,
  en lugar de una única forma de mensaje independiente de la plataforma con una relación `kind: "reply" |
"followup" | "broadcast" | "system"`.
- **Los nombres de las políticas de confirmación difieren de los del boceto.** Publicado:
  `after_receive_record | after_agent_dispatch | after_durable_send | manual`.
  El boceto original usaba `immediate | after-record | after-durable-send |
manual` con un campo de motivo del tiempo de espera del Webhook; esa forma no se creó.
- **Las claves de capacidad `DurableFinalDeliveryRequirementMap` sustituyeron al objeto
  `MessageCapabilities` del boceto.** Las capacidades son indicadores booleanos planos (`text`,
  `media`, `poll`, `payload`, `silent`, `replyTo`, `thread`, `nativeQuote`,
  `messageSendingHooks`, `batch`, `reconcileUnknownSend`, `afterSendSuccess`,
  `afterCommit`) verificados mediante `verifyDurableFinalCapabilityProofs`, en lugar
  de una estructura anidada al estilo de `text.chunking` / `attachments.voice`.

## Riesgos concretos de la migración (aún pertinentes)

Estos efectos secundarios específicos de cada canal son anteriores a la refactorización y deben seguir
funcionando mediante las nuevas rutas de envío. No son hipotéticos: todos están
implementados y son esenciales actualmente.

- **iMessage** (`extensions/imessage/src/monitor/echo-cache.ts`,
  `persisted-echo-cache.ts`): el monitor registra los mensajes enviados en una caché
  de eco después de un envío correcto. Los envíos finales duraderos deben seguir
  llenando esa caché; de lo contrario, OpenClaw puede volver a ingerir sus propias
  respuestas como mensajes entrantes del usuario.
- **Tlon** (`extensions/tlon/src/monitor/index.ts`): añade una firma opcional del modelo
  y registra los hilos en los que se ha participado después de las respuestas de grupo.
  La entrega duradera no debe omitir esos efectos.
- **Discord y otros despachadores preparados** ya gestionan la entrega directa y
  el comportamiento de las vistas previas. Un canal no es duradero de extremo a extremo
  hasta que su despachador preparado enruta explícitamente los mensajes finales mediante
  el contexto de envío; no se debe asumir que el adaptador genérico por sí solo ofrece cobertura.
- La **entrega alternativa silenciosa de Telegram** debe entregar todo el arreglo
  de cargas útiles proyectadas, no solo la primera carga útil, después de la
  fragmentación o proyección alternativa.
- **LINE, Zalo, Nostr** y rutas auxiliares similares pueden incluir la gestión
  de tokens de respuesta, el uso de proxies para contenido multimedia, cachés de mensajes
  enviados o destinos disponibles únicamente mediante callbacks. Permanecen bajo la entrega
  gestionada por el canal hasta que el adaptador de envío represente esas semánticas y estén
  cubiertas por pruebas.
- Los **auxiliares de mensajes directos** pueden tener un callback de respuesta
  que sea el único destino de transporte correcto. La salida genérica no debe inferir un
  destino a partir de campos sin procesar de la plataforma y omitir ese callback.

## Clasificación de fallos

Los adaptadores clasifican los fallos de transporte en categorías cerradas
del tipo `DeliveryFailureKind` (transitorio, límite de frecuencia, autenticación,
permiso, no encontrado, carga útil no válida, conflicto, cancelado, desconocido).
Política del núcleo:

- Reintentar los fallos transitorios y por límite de frecuencia.
- No reintentar los fallos por carga útil no válida, salvo que exista una alternativa de renderizado.
- No reintentar los fallos de autenticación o permisos hasta que cambie la configuración.
- Cuando no se encuentre el elemento, permitir que la finalización en vivo pase
  de la edición a un nuevo envío cuando el canal declare que es seguro.
- En caso de conflicto, utilizar el estado del recibo o de idempotencia para
  determinar si el mensaje ya existe.
- Cualquier error que se produzca después de que la llamada a la plataforma
  pueda haber tenido éxito, pero antes de confirmar el recibo, se convierte en
  `unknown_after_send`, salvo que el adaptador demuestre que la operación de la
  plataforma no se realizó.

## Preguntas abiertas

- Si Telegram debería sustituir en algún momento el ejecutor de sondeo de grammY
  (`1.43.0`) por una fuente de sondeo completamente duradera que controle
  la reentrega a nivel de plataforma, y no solo la marca de agua de reinicio persistente
  de OpenClaw (`safeCompletedUpdateId`).
- Si el estado de la vista previa en vivo debería residir en el mismo registro
  que la intención del envío final o en un almacén hermano de estado en vivo.
- Si la supresión de eco por fallos del Gateway en salas compartidas con bots
  habilitados necesita el mecanismo de etiquetado de origen previsto inicialmente,
  un contrato más sencillo por canal o queda fuera del alcance.
- Qué canales ofrecen compatibilidad nativa con el origen o los metadatos
  para suprimir el eco entre bots y cuáles necesitan un registro persistente de
  mensajes salientes.

## Contenido relacionado

- [Mensajes](/es/concepts/messages)
- [Streaming y fragmentación](/es/concepts/streaming)
- [Borradores de progreso](/es/concepts/progress-drafts)
- [Política de reintentos](/es/concepts/retry)
- [API de salida de canales](/es/plugins/sdk-channel-outbound)
- [API de entrada de canales](/es/plugins/sdk-channel-inbound)
