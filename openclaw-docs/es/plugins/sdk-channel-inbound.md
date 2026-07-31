---
read_when:
    - Está creando o refactorizando la ruta de recepción de un plugin de canal de mensajería
    - Se necesita una construcción compartida del contexto entrante, el registro de sesiones o el envío de respuestas preparadas
    - Está migrando los antiguos auxiliares de turnos de canal a las API de entrada/mensajes
summary: 'Utilidades de eventos entrantes para plugins de canales: creación de contexto, orquestación del ejecutor compartido, registro de sesión y envío de respuestas preparadas'
title: API de entrada del canal
x-i18n:
    generated_at: "2026-07-26T04:49:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 854408ca42cfe1e1b48e4fd223b176f438e1db28deb9a5aa33eea8238127d9df
    source_path: plugins/sdk-channel-inbound.md
    workflow: 16
---

Las rutas de recepción de los canales siguen un único flujo:

```text
evento de la plataforma -> hechos/contexto entrantes -> respuesta del agente -> entrega del mensaje
```

Use `openclaw/plugin-sdk/channel-inbound` para la normalización, el formato, las raíces y la orquestación de eventos entrantes. Use
`openclaw/plugin-sdk/channel-outbound` para el envío nativo, el acuse de recibo, la entrega
duradera y el comportamiento de la vista previa en directo.

## Ayudantes principales

```ts
import {
  buildChannelInboundEventContext,
  runChannelInboundEvent,
  dispatchChannelInboundReply,
} from "openclaw/plugin-sdk/channel-inbound";
```

- `buildChannelInboundEventContext(...)`: proyecta los hechos normalizados del canal
  en el contexto del prompt y de la sesión. Pase los metadatos del remitente/chat
  que pertenecen al canal mediante `channelContext`, que los hooks del plugin ven como `ctx.channelContext`.
  Amplíe `PluginHookChannelSenderContext` o `PluginHookChannelChatContext`
  desde esta subruta para los campos específicos del canal.
- `runChannelInboundEvent(...)`: ejecuta la ingesta, clasificación, comprobación previa, resolución,
  registro, despacho y finalización de un evento entrante de la plataforma.
- `dispatchChannelInboundReply(...)`: registra y despacha una respuesta entrante
  ya ensamblada mediante un adaptador de entrega.

Para los eventos entrantes que solo contienen contenido multimedia, mantenga vacíos el cuerpo del mensaje y el texto del comando y
pase un hecho `ChannelInboundMediaInput` por cada archivo adjunto nativo. Cuando una línea del
historial ambiental u otro portador exclusivamente textual deba describir esos hechos, use
`formatMediaPlaceholderText(media)`. Clasifica cada hecho a partir de `kind`, el tipo
MIME y, después, la extensión de la ruta o URL; los archivos adjuntos nativos no descargados deben seguir
aportando un hecho que contenga únicamente el tipo. No use el formateador para sintetizar el
cuerpo entrante principal.

Normalice los registros de archivos adjuntos que pertenecen al plugin con `toInboundMediaFacts(...)` y, después,
pase la matriz ordenada resultante mediante el campo `media` del contexto:

```ts
const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

La posición en la matriz constituye la identidad del archivo adjunto. Los valores `transcribed`, `messageId` y
`workspaceDir` de cada hecho sustituyen a los campos paralelos heredados de índice/espacio de trabajo. Los
campos de contexto `MediaPath`, `MediaPaths`, `MediaUrl`, `MediaUrls`, `MediaType`, `MediaTypes`,
`MediaTranscribedIndexes`, `MediaWorkspaceDir` y `MediaStaged`,
además de `buildChannelInboundMediaPayload(...)`, siguen disponibles únicamente por motivos de compatibilidad
obsoleta. Los plugins nuevos no deben crearlos ni leerlos.

Los canales incluidos/nativos que ya reciben el objeto inyectado del entorno de ejecución del plugin
pueden llamar a los mismos ayudantes en `runtime.channel.inbound.*`, en lugar de
importar directamente esta subruta:

```ts
await runtime.channel.inbound.run({
  channel: "demo",
  accountId,
  raw: platformEvent,
  adapter: {
    ingest: normalizePlatformEvent,
    resolveTurn: resolveInboundReply,
  },
});
```

Ensamble las entradas `dispatchChannelInboundReply(...)` para los despachadores
de compatibilidad que mantienen la entrega de la plataforma en el adaptador de entrega. Las nuevas rutas
de envío deben usar adaptadores de mensajes y ayudantes de mensajes duraderos de
`channel-outbound`.

## Contrato de resolución de entrega

`ChannelInboundTurnPlan.delivery` se encarga del envío nativo de cada carga útil de respuesta
lógica. El núcleo controla el orden de los hooks salientes y, cuando el adaptador lo habilita,
la observación terminal `message_sent`. Mantenga separadas esas responsabilidades para que
una carga útil no pueda generar eventos terminales duplicados.

Los campos del resultado de entrega tienen estos significados:

| Campo                    | Contrato                                                                                                                                                                                                                     |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `content`                | Texto visible aceptado por el proveedor para la carga útil lógica después del formato o la finalización nativos. Omítalo para usar el texto preparado de la carga útil en la observación terminal. Los envíos que solo contienen contenido multimedia pueden omitirlo.                             |
| `messageIds` / `receipt` | Identidades reales del proveedor correspondientes al envío visible. Prefiera un `MessageReceipt`; el núcleo usa su identificador principal del proveedor para `message_sent`.                                                                                            |
| `visibleReplySent`       | Establézcalo en `false` únicamente cuando el proveedor no haya generado ninguna vista previa ni mensaje final visibles. El núcleo no emite un `message_sent` correcto para ese resultado.                                                                          |
| `finalization`           | Una promesa para la resolución nativa diferida de la misma carga útil lógica, como cerrar o editar una tarjeta de transmisión en el mismo lugar. Sus campos resueltos sustituyen al resultado inmediato antes de la observación terminal y de `onDelivered`. |

Establezca la opción `observeMessageSent` del adaptador de entrega en `true` cuando el núcleo
deba emitir los eventos canónicos internos y del plugin `message_sent` para los
envíos no duraderos de este adaptador. No devuelva esta opción desde `deliver` ni
emita también esos eventos en el plugin. Los envíos duraderos ya se emiten mediante
el responsable saliente compartido y no se duplican.

Devuelva un resultado por carga útil lógica. `finalization` no es un segundo envío y
no debe volver a ejecutar `reply_payload_sending` ni `message_sending`. En cuanto
`deliver` devuelve el control, el núcleo observa el rechazo de la promesa de finalización para que
no quede sin gestionar; el núcleo sigue esperando la promesa original después de que finalice el
despacho de la respuesta. Después emite como máximo una observación terminal por carga útil,
con el contenido finalizado y el identificador del proveedor. Cuando está presente,
`onDelivered` recibe el resultado resuelto después de esa observación.

Rechace `deliver` o `finalization` cuando falle la entrega nativa. Si no se
intentó ningún envío del proveedor, lance `PlatformMessageNotDispatchedError` desde
`openclaw/plugin-sdk/error-runtime`; el núcleo suprime un evento
`message_sent` falso. Si un envío nativo se hizo visible antes de que fallara
una operación posterior, conserve el subconjunto visible en el error:

```ts
import { createChannelPartialDeliveryError } from "openclaw/plugin-sdk/channel-inbound";

throw createChannelPartialDeliveryError(cause, {
  visibleReplySent: true,
  content: finalizedVisibleText,
  receipt,
});
```

El núcleo emite una observación terminal fallida con ese contenido visible para el proveedor y su
identidad; después mantiene la entrega como fallida para que los llamadores no confundan un éxito
parcial con un envío correcto. No informe de `visibleReplySent: false` después de que cualquier
vista previa, borrador, archivo adjunto o mensaje final se haya hecho visible.

Cuando `reply_payload_sending` o `message_sending` estén registrados, esos hooks
deben resolverse antes de crear cualquier elemento visible para el proveedor, porque cualquiera de ellos
puede reescribir o cancelar la carga útil lógica. Una vista previa nativa anticipada filtraría
contenido anterior a la reescritura o dejaría un borrador cancelado. Almacene en búfer el contenido de la vista previa
hasta que la carga útil aceptada llegue a `deliver`; los despachadores de compatibilidad que
inicien las vistas previas antes deben suprimir esa vista previa anticipada mientras cualquiera de los hooks esté
registrado. Use los ayudantes de vista previa en directo finalizable de la
[API saliente de canales](/es/plugins/sdk-channel-outbound) para las nuevas rutas de vista previa.

## Migración

Se eliminaron los alias del entorno de ejecución `runtime.channel.turn.*`. Use:

- `runtime.channel.inbound.run(...)` para eventos entrantes sin procesar.
- `runtime.channel.inbound.dispatchReply(...)` para contextos de respuesta ensamblados.
- `runtime.channel.inbound.buildContext(...)` para cargas útiles de contexto entrante.
- `runtime.channel.inbound.runPreparedReply(...)`, obsoleto, únicamente para
  rutas de despacho preparadas que pertenecen al canal y que ya ensamblan su propio
  cierre de despacho.

El código de plugins nuevo no debe introducir API de canales denominadas `turn`. Mantenga el vocabulario de turnos
del modelo o del agente dentro del código del agente/proveedor; los plugins de canales usan términos de entrada,
mensaje, entrega y respuesta.
