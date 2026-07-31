---
read_when:
    - Está creando o refactorizando la ruta de envío de un plugin de canal de mensajería
    - Se necesita una política de entrega duradera de respuestas finales, confirmaciones de entrega, finalización de la vista previa en tiempo real o acuse de recibo.
    - Está migrando desde los ayudantes de envío de mensajes de canal o de respuestas heredadas
summary: 'API del ciclo de vida de los mensajes salientes para plugins de canal: adaptadores, confirmaciones, envíos duraderos, vista previa en tiempo real y auxiliares del pipeline de respuestas'
title: API de salida del canal
x-i18n:
    generated_at: "2026-07-26T05:15:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8edeca81d2e9261f33be1d538153caaea87caedb90dfccac33dd227c924501f1
    source_path: plugins/sdk-channel-outbound.md
    workflow: 16
---

Los plugins de canal exponen el comportamiento de los mensajes salientes desde
`openclaw/plugin-sdk/channel-outbound`. Use
`openclaw/plugin-sdk/channel-inbound` para la orquestación de recepción, contexto y despacho.

El núcleo se encarga de las colas, la durabilidad, el **monitor y drenaje de entrada**
duradero (`createChannelIngressMonitor`, `createChannelIngressDrain` y
`openChannelIngressDrain`), la política genérica de reintentos, el ciclo de vida
de adopción del turno (`turnAdoptionLifecycle` / `bindIngressLifecycleToReplyOptions`), los hooks,
los recibos y la herramienta compartida `message`. El plugin se encarga de
las llamadas nativas de envío, edición y eliminación, la normalización del destino,
los hilos de la plataforma, las citas seleccionadas, los indicadores de notificación,
el estado de la cuenta, la inspección de entrada y la codificación de la carga útil,
las claves de carril, los predicados que impiden reintentos, la autorización opcional
de sustitución y los efectos secundarios específicos de la plataforma.

## Monitores de entrada duraderos

Use `createChannelIngressMonitor(...)` cuando un canal deba conservar los eventos
de transporte aceptados antes de despacharlos. Combina una cola y un drenaje de entrada
del canal con el ciclo de vida compartido de admisión, sondeo, depuración, entrega y
cierre. Use el nivel inferior `createChannelIngressDrain(...)` solo cuando el transporte
se encargue de un contrato de admisión o bombeo sustancialmente diferente.

Las opciones obligatorias son:

| Opción                           | Contrato                                                                                                                                                                                                                                                                                                         |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `queue`                          | Un `ChannelIngressQueue` o una fábrica diferida que abre la cola del ámbito de la cuenta.                                                                                                                                                                                                                                  |
| `inspect(raw, context)`          | Devuelve el `eventId` estable y el `laneKey` serializado, o `null` para un evento ignorado. Los datos del momento de la reclamación deben coincidir con el identificador y el carril persistentes.                                                                                                                                                                    |
| `payload`                        | Proporciona la versión de la carga útil y la serialización/deserialización del cuerpo. Use `storage: "raw-event"` para el sobre de cadena estándar `{ version, rawEvent }`, o proporcione callbacks personalizados de codificación/decodificación para una forma existente específica del canal. `createClaimError` clasifica las versiones no válidas o los cambios de identidad. |
| `deliver(raw, lifecycle, claim)` | Despacha un evento decodificado y recibe el ciclo de vida completo de adopción. Puede devolver `completed`, `deferred`, `failed-retryable` o nada.                                                                                                                                                                |
| `pollIntervalMs`                 | Programa sondeos de recuperación/drenaje mientras el monitor está en ejecución.                                                                                                                                                                                                                                                     |
| `retention`                      | Proporciona la cadencia de depuración y el TTL y los límites de entradas completadas/fallidas.                                                                                                                                                                                                                                              |

El monitor serializa las admisiones para que el aplazamiento de anexión no pueda
invertir un carril. Los retrasos acotados predeterminados para anexar son
`0`, `100` y `300` ms; si se agotan,
se rechaza el callback del transporte en lugar de despachar un evento que no se hizo
duradero. En el momento de la reclamación, decodifica la carga útil versionada, vuelve
a ejecutar `inspect` y rechaza cualquier discrepancia del identificador o del
carril antes de la entrega.

`deliver` recibe `onAdopted`, `onDeferred`, `onAdoptionFinalizing`,
`onAbandoned` y `abortSignal`. Devolver sin una transferencia explícita marca
como adoptado un evento terminal sin despacho. `admission` siempre es
`exclusive`. Una transferencia diferida mantiene retenida la reclamación, mientras
que el cierre o la cancelación permiten volver a intentar el trabajo no adoptado. El
monitor realiza el seguimiento de la entrega independientemente de la resolución de la
reclamación porque la adopción puede crear una lápida para una fila antes de que se
resuelva la promesa de entrega del canal.

La configuración opcional incluye retrasos personalizados de anexión, un bloque de
opciones `drain` para políticas avanzadas de ordenación, concurrencia y
reintentos del drenaje, un `abortSignal` externo, un reloj, informes de errores
del bombeo, una fábrica de errores de detención y una política de admisión.
El monitor devuelto expone `admit`, `start`, `pause`,
`stop`, `waitForIdle`, `isRunning` y `isStopped`.
`stop` primero resuelve las admisiones aceptadas; después cancela y
descarta el drenaje, espera al bombeo y a las entregas activas, y vuelve a descartarlo
para cerrar la condición de carrera de la creación diferida.

Mantenga en el plugin la ocultación específica del transporte, la validación del
sobre sin procesar, la clasificación de los errores que no deben reintentarse y la
forma de la carga útil persistente. Los transportes Webhook solo deben confirmar
después de que se resuelva `admit`; los transportes sin repetición deben
señalar el agotamiento de la anexión duradera en lugar de despachar silenciosamente.

## Adaptador

La mayoría de los plugins definen un adaptador `message`:

```ts
import {
  defineChannelMessageAdapter,
  createMessageReceiptFromOutboundResults,
} from "openclaw/plugin-sdk/channel-outbound";

export const demoMessageAdapter = defineChannelMessageAdapter({
  id: "demo",
  durableFinal: {
    capabilities: {
      text: true,
      replyTo: true,
      thread: true,
      messageSendingHooks: true,
    },
  },
  send: {
    text: async ({ cfg, to, text, accountId, replyToId, threadId, signal }) => {
      const sent = await sendDemoMessage({
        cfg,
        to,
        text,
        accountId: accountId ?? undefined,
        replyToId: replyToId ?? undefined,
        threadId: threadId == null ? undefined : String(threadId),
        signal,
      });

      return {
        receipt: createMessageReceiptFromOutboundResults({
          results: [{ channel: "demo", messageId: sent.id, conversationId: to }],
          kind: "text",
          threadId: threadId == null ? undefined : String(threadId),
          replyToId: replyToId ?? undefined,
        }),
      };
    },
  },
});
```

Declare únicamente las capacidades que el transporte nativo realmente conserva.
Cubra cada capacidad declarada de envío, recibo, vista previa en directo y
confirmación de recepción con los auxiliares de contrato exportados desde esta
subruta.

## Supresión del eco saliente

Cuando una plataforma pueda volver a entregar como entrante el propio mensaje
saliente del plugin, llame a `recordOutboundMessageIdentity(...)` con el canal, la cuenta, la
conversación y una identidad estable del mensaje o del origen de la plataforma. La
ruta compartida de turnos entrantes descarta las identidades coincidentes durante
una ventana acotada de 30 segundos antes de registrar la sesión o despachar al
agente; una identidad de origen puede reservarse antes del envío o actualizarse
cuando se elimina una ruta de canal para cerrar las condiciones de carrera de
entrega. `isRecentOutboundMessageIdentity(...)` expone la misma consulta para los diagnósticos y las
pruebas del canal. No mantenga una caché TTL local del canal paralela para la misma
identidad estable.

## Saneamiento de texto sin formato

Use `sanitizeForPlainText(...)` cuando un adaptador saliente necesite convertir las
etiquetas de formato HTML compatibles en marcado de texto ligero. De forma
predeterminada, se conservan los marcadores existentes de negrita y tachado del
estilo de chat. Pase `{ style: "markdown" }` solo cuando el canal vuelva a analizar el
resultado como Markdown:

```ts
import { sanitizeForPlainText } from "openclaw/plugin-sdk/channel-outbound";

const chatText = sanitizeForPlainText(text);
const markdownText = sanitizeForPlainText(text, { style: "markdown" });
```

El estilo Markdown usa `**bold**` y `~~strikethrough~~`; la cursiva y el
código en línea conservan `_italic_` y los marcadores de acento grave en
ambos estilos. Seleccione el estilo en el límite del canal en lugar de reescribir
el texto de los marcadores después del saneamiento.

## Evidencia de entrega

Un `MessageReceipt` registra el resultado devuelto por un adaptador de canal. Los
identificadores concretos de mensajes de la plataforma demuestran que la ruta de
envío de la plataforma aceptó el mensaje; no demuestran que el dispositivo de un
destinatario lo mostrara o leyera. Los recibos sin identificadores de mensajes de
la plataforma solo son metadatos locales del recibo. Los canales con confirmaciones
de lectura o estados de entrega al dispositivo deben realizar el seguimiento de
esos datos mediante una ruta independiente y específica del canal.

Si un adaptador de canal puede demostrar que reintentar un fallo no puede duplicar
un envío visible para el destinatario y que no comenzó ninguna llamada capaz de
finalizarlo, lance `new PlatformMessageNotDispatchedError("...", { cause: error })` desde
`openclaw/plugin-sdk/error-runtime`. De este modo, el núcleo puede borrar la evidencia obsoleta del
intento de envío y reintentar de forma segura la intención en cola. Solo el
adaptador que controla el límite del despacho final puede realizar esta afirmación.
Nunca use el marcador después de que comience una llamada de finalización/envío ni
cuando esta devuelva un resultado ambiguo; un marcado incorrecto puede duplicar
mensajes.

## Adaptadores salientes existentes

Si el canal ya tiene un adaptador `outbound` compatible, derive el adaptador
de mensajes en lugar de duplicar el código de envío:

```ts
import { createChannelMessageAdapterFromOutbound } from "openclaw/plugin-sdk/channel-outbound";

export const messageAdapter = createChannelMessageAdapterFromOutbound({
  id: "demo",
  outbound,
  durableFinal: {
    capabilities: {
      text: true,
      media: true,
    },
  },
});
```

## Envíos duraderos

Los auxiliares de envío en tiempo de ejecución también se encuentran en
`channel-outbound`:

- `sendDurableMessageBatch(...)`
- `withDurableMessageSendContext(...)`
- `deliverInboundReplyWithMessageSendContext(...)`
- auxiliares de streaming/progreso de borradores, como `resolveChannelDraftStreamingChunking(...)`

`sendDurableMessageBatch(...)` devuelve un resultado explícito:

| Resultado          | Significado                                                                                 |
| ---------------- | --------------------------------------------------------------------------------------- |
| `sent`           | la ruta de envío de la plataforma aceptó al menos un mensaje visible de la plataforma            |
| `suppressed`     | ningún mensaje de la plataforma debe considerarse ausente                                        |
| `partial_failed` | se aceptó al menos un mensaje de la plataforma antes de que fallara una carga útil o un efecto secundario posterior |
| `failed`         | no se generó ningún recibo de la plataforma                                                        |

Use `payloadOutcomes` cuando un lote combine cargas útiles enviadas, suprimidas y
fallidas. No deduzca la cancelación de un hook a partir de un resultado vacío de
entrega directa heredada.

## Admisión de entregas diferidas

Use `message.durableFinal.admitDeferredDelivery(...)` cuando una cuenta resuelta no pueda aceptar de forma segura
entregas salientes o diferidas administradas por el núcleo. El núcleo llama a este
hook de forma síncrona antes del trabajo saliente en directo, incluidas las rutas
que omiten la persistencia en cola, y de nuevo antes de reproducir una intención
recuperada. El contexto incluye `cfg`, `channel`,
`to`, `accountId` y un `phase` de
`live` o `recovery`.

Devuelva `{ status: "allowed" }` para continuar. Devuelva
`{ status: "permanent_rejection", reason }` cuando la entrega no deba persistirse, enviarse directamente ni
reproducirse. Un rechazo en directo falla antes de la creación de la cola, los
hooks de mensajes o el trabajo de la plataforma. Un rechazo durante la recuperación
marca el registro en cola como fallido y omite la conciliación y la reproducción.
Omitir el hook significa que está permitido.

El hook es una decisión de admisión síncrona, no una ruta de envío. Lea únicamente
la configuración o el estado de ejecución ya cargados; no realice operaciones de E/S
de red, del sistema de archivos ni otras operaciones asíncronas. Las pruebas de contrato deben ejercitar ambas fases y ambas
variantes de resultado mediante `ChannelMessageDurableFinalAdapter` desde
`openclaw/plugin-sdk/channel-outbound`.

## Despacho de compatibilidad

Ensamble el despacho de respuestas entrantes mediante `dispatchChannelInboundReply(...)`
desde `channel-inbound`. Mantenga la entrega de la plataforma en el adaptador de entrega; use
`channel-outbound` para adaptadores de mensajes, envíos duraderos, confirmaciones, vista previa
en vivo y opciones del pipeline de respuestas.
