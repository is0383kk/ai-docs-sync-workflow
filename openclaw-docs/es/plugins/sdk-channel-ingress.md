---
read_when:
    - Creación o migración de un plugin de canal de mensajería
    - Cambiar las listas de permitidos de mensajes directos o grupos, las puertas de enrutamiento, la autenticación de comandos, la autenticación de eventos o la activación mediante menciones
    - Revisión de los límites de compatibilidad del SDK o de la censura de entrada del canal
sidebarTitle: Channel Ingress
summary: API experimental de entrada de canales para la autorización de mensajes entrantes
title: API de entrada de canales
x-i18n:
    generated_at: "2026-07-26T04:47:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 60feecb7bcf203cf37d2543a7855e89b5bfb2eb9d8263d804219e140facb8fc6
    source_path: plugins/sdk-channel-ingress.md
    workflow: 16
---

La entrada de canales es el límite experimental de control de acceso para los eventos de canal entrantes. Los plugins controlan los datos de la plataforma y los efectos secundarios; el núcleo controla la política genérica: listas de permitidos de MD/grupos, entradas de MD del almacén de emparejamiento, puertas de rutas, puertas de comandos, autorización de eventos, activación mediante menciones, diagnósticos censurados y admisión.

Use `openclaw/plugin-sdk/channel-ingress-runtime` para las rutas de recepción.

## Resolutor de tiempo de ejecución

```ts
import {
  defineStableChannelIngressIdentity,
  resolveChannelMessageIngress,
} from "openclaw/plugin-sdk/channel-ingress-runtime";

const identity = defineStableChannelIngressIdentity({
  key: "platform-user-id",
  normalize: normalizePlatformUserId,
  sensitivity: "pii",
});

const result = await resolveChannelMessageIngress({
  channelId: "my-channel",
  accountId,
  identity,
  subject: { stableId: platformUserId },
  conversation: { kind: isGroup ? "group" : "direct", id: conversationId },
  event: { kind: "message", authMode: "inbound", mayPair: !isGroup },
  policy: {
    dmPolicy: config.dmPolicy,
    groupPolicy: config.groupPolicy,
    groupAllowFromFallbackToAllowFrom: true,
  },
  allowFrom: config.allowFrom,
  groupAllowFrom: config.groupAllowFrom,
  accessGroups: cfg.accessGroups,
  route,
  readStoreAllowFrom,
  command: hasControlCommand ? { allowTextCommands: true, hasControlCommand } : undefined,
});
```

No precalcule las listas de permitidos efectivas, los propietarios de comandos ni los grupos de comandos. El resolutor los deriva de las listas de permitidos sin procesar, las devoluciones de llamada del almacén, los descriptores de rutas, los grupos de acceso, la política y el tipo de conversación.

## Resultado

Los plugins incluidos deben consumir directamente las proyecciones modernas:

| Campo              | Significado                                                            |
| ------------------ | ------------------------------------------------------------------ |
| `ingress`          | decisión ordenada de las puertas y admisión                                |
| `senderAccess`     | solo autorización del remitente y la conversación                             |
| `routeAccess`      | proyección de la ruta y del remitente de la ruta                                  |
| `commandAccess`    | autorización de comandos; `requested: false` cuando no se ejecutó ninguna puerta de comandos |
| `activationAccess` | resultado de mención/activación                                          |

La autorización de eventos sigue disponible en el elemento ordenado `ingress.graph` y en el elemento decisivo `ingress.reasonCode`; no se emite ninguna proyección de eventos independiente.

Los asistentes obsoletos del SDK de terceros pueden reconstruir internamente las estructuras anteriores. Las nuevas rutas de recepción incluidas no deben volver a convertir los resultados modernos en DTO locales.

## Grupos de acceso

Las entradas `accessGroup:<name>` permanecen censuradas. El núcleo resuelve por sí mismo los grupos estáticos `message.senders` y llama a `resolveAccessGroupMembership` solo para los grupos dinámicos que requieren una consulta a la plataforma. Los grupos ausentes, no compatibles y fallidos se cierran de forma segura.

## Modos de eventos

| `authMode`       | Significado                                          |
| ---------------- | ------------------------------------------------ |
| `inbound`        | puertas normales del remitente entrante                      |
| `command`        | puertas de comandos para devoluciones de llamada o botones con ámbito    |
| `origin-subject` | el actor debe coincidir con el sujeto del mensaje original    |
| `route-only`     | solo puertas de rutas para eventos de confianza con ámbito de ruta |
| `none`           | los eventos internos controlados por el plugin omiten la autorización compartida  |

Use `mayPair: false` para reacciones, botones, devoluciones de llamada y comandos nativos.

## Rutas y activación

Use descriptores de rutas para las políticas de sala, tema, servidor, hilo o ruta anidada:

```ts
route: {
  id: "room",
  allowed: roomAllowed,
  enabled: roomEnabled,
  senderPolicy: "replace",
  senderAllowFrom: roomAllowFrom,
  blockReason: "room_sender_not_allowlisted",
}
```

Use `channelIngressRoutes(...)` cuando un plugin tenga varios descriptores de rutas opcionales; filtra las ramas deshabilitadas y mantiene los datos de las rutas genéricos y ordenados según el valor `precedence` de cada descriptor.

La puerta de menciones es una puerta de activación. Una mención no detectada devuelve `admission: "skip"` para que el núcleo de turnos no procese un turno de solo observación. La mayoría de los canales deben dejar la activación después de las puertas de remitentes y comandos. Las superficies de chat públicas que deban silenciar el tráfico sin menciones antes del ruido de las listas de permitidos de remitentes pueden optar por `activation.order: "before-sender"` cuando la omisión mediante comandos de texto esté deshabilitada. Los canales con activación implícita, como las respuestas en hilos de bots, resuelven `channels.defaults.implicitMentions` junto con las anulaciones del canal y de la cuenta mediante `resolveChannelImplicitMentions(...)` y, a continuación, pasan el resultado como `activation.implicitMentions`. El elemento proyectado `activationAccess.shouldBypassMention` indica cuándo un comando o una activación implícita omitió una mención explícita.

## Censura

Los valores sin procesar de los remitentes y las entradas sin procesar de las listas de permitidos son únicamente datos de entrada del resolutor. No deben aparecer en el estado resuelto, las decisiones, los diagnósticos, las instantáneas ni los datos de compatibilidad. Use identificadores opacos de sujetos, entradas, rutas y diagnósticos.

## Verificación

```bash
pnpm test src/channels/message-access/message-access.test.ts src/plugin-sdk/channel-ingress-runtime.test.ts
pnpm plugin-sdk:api:check
```
