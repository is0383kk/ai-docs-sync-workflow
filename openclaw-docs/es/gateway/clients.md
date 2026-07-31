---
read_when:
    - Creación de un cliente de operador, panel de control o WebChat fuera del repositorio de OpenClaw
    - Implementación de la reconexión del Gateway, el historial, las aprobaciones o el emparejamiento de dispositivos
    - Actualización de un cliente de terceros para una nueva versión del protocolo de comunicación del Gateway
summary: Crea un cliente operador de terceros o WebChat para el protocolo WebSocket del Gateway
title: Creación de un cliente de Gateway
x-i18n:
    generated_at: "2026-07-26T05:07:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fa24b196ff1fa28fb3b64d49ac25597f22cf1945aea56029e78e4375f1bdddb7
    source_path: gateway/clients.md
    workflow: 16
---

Usa los paquetes publicados de Gateway para crear paneles de operadores, clientes de WebChat
y otras aplicaciones de terceros. Esta guía abarca el ciclo de vida del cliente en torno
al contrato de comunicación: autenticación, capacidades, recuperación tras reconexiones, historial,
suscripciones y actualizaciones de versión.

Para conocer las estructuras de las tramas, el protocolo de enlace, los errores y la superficie completa de métodos, consulta la
[especificación del protocolo de Gateway](https://docs.openclaw.ai/gateway/protocol).

## Instalar los paquetes

```bash
npm install @openclaw/gateway-client @openclaw/gateway-protocol
```

<Note>
Estos paquetes se distribuyen con los ciclos de versiones de OpenClaw. Durante el despliegue inicial, npm
puede devolver `E404` hasta que se publique la primera versión de OpenClaw que incluya los paquetes;
instálalos únicamente después de que las páginas del registro indicadas a continuación estén disponibles.
</Note>

- [`@openclaw/gateway-protocol`](https://www.npmjs.com/package/@openclaw/gateway-protocol)
  proporciona esquemas, validadores en tiempo de ejecución, tipos de TypeScript, registros de identidad y
  capacidades del cliente, lectores de errores estructurados y constantes de versión del protocolo.
  Su archivo tar de npm también incluye el contrato generado
  [`protocol.schema.json`](https://unpkg.com/@openclaw/gateway-protocol/protocol.schema.json)
  legible por máquinas.
- [`@openclaw/gateway-client`](https://www.npmjs.com/package/@openclaw/gateway-client)
  es la implementación de referencia de la conexión. Importa la raíz del paquete para el cliente de Node
  y `@openclaw/gateway-client/browser` para los ayudantes compatibles con navegadores relativos al protocolo,
  la autenticación de dispositivos y la reconexión.

El punto de entrada de Node administra su transporte WebSocket. Un host de navegador proporciona un adaptador
WebSocket, además de almacenamiento persistente y funciones de devolución de llamada de firma para la identidad
y el token del dispositivo.

## Elegir ámbitos y vincular el dispositivo

Un cliente de chat interactivo completo que también muestre solicitudes de aprobación debe solicitar
`role: "operator"` con estos ámbitos:

| Ámbito               | Uso                                                                                       |
| -------------------- | ----------------------------------------------------------------------------------------- |
| `operator.read`      | `chat.history`, `sessions.list`, `sessions.subscribe`, estado del modelo y eventos de solo lectura |
| `operator.write`     | `chat.send` y modificaciones normales de sesiones                                         |
| `operator.approvals` | Enumerar, mostrar y resolver aprobaciones de ejecución o de plugins                        |

Añade `operator.questions` únicamente si el cliente gestiona preguntas interactivas,
`operator.pairing` únicamente si administra dispositivos o nodos vinculados y
`operator.admin` únicamente para operaciones administrativas como `config.patch`.
La [referencia de ámbitos de operador](https://docs.openclaw.ai/gateway/operator-scopes)
define las reglas completas para los métodos y el momento de aprobación.

No crees manualmente un token de portador para cada cliente editando `openclaw.json`. Configura
la autenticación de arranque compartida de Gateway con `openclaw configure --section
gateway` o las opciones de `openclaw onboard --gateway-auth ...` y, después, permite que la
vinculación del dispositivo genere el token del cliente:

1. Conserva una identidad de dispositivo Ed25519 en el cliente.
2. Espera a `connect.challenge`, firma la carga útil del dispositivo vinculada al desafío y envía
   `connect` con el rol de operador y los ámbitos solicitados, además del token
   o la contraseña compartidos de Gateway para la autenticación de arranque.
3. Si Gateway devuelve detalles estructurados de `PAIRING_REQUIRED`, muestra el ID
   de la solicitud y pausa o reintenta según `error.details.recommendedNextStep`.
4. En el host de Gateway, revisa la solicitud con `openclaw devices list` y, después,
   aprueba exactamente esa solicitud vigente con `openclaw devices approve <requestId>`.
5. Vuelve a conectarte y conserva `hello-ok.auth.deviceToken` con el rol y los
   ámbitos negociados. Usa ese token de dispositivo para las conexiones posteriores.

Las ampliaciones de ámbitos o roles crean una nueva solicitud de vinculación pendiente. La rotación de tokens no puede
ampliar el contrato de vinculación aprobado. Consulta la
[CLI de dispositivos](https://docs.openclaw.ai/cli/devices) para conocer los comandos de aprobación, rotación y
revocación.

## Anunciar las capacidades del cliente

`connect.params.caps` describe el comportamiento opcional que el cliente puede utilizar. No
concede autorización. Importa los nombres desde `GATEWAY_CLIENT_CAPS` en lugar de
duplicar literales de cadena:

```ts
import { GATEWAY_CLIENT_CAPS } from "@openclaw/gateway-protocol/client-info";

const caps = [GATEWAY_CLIENT_CAPS.TOOL_EVENTS];
```

El registro actual contiene `approvals`, `exec-approvals`, `inline-widgets`,
`run-tool-bindings`, `session-scoped-events`, `plugin-approvals`,
`task-suggestions`, `terminal-offset-seq`, `tool-events` y `ui-commands`.
Anuncia únicamente las capacidades que el cliente implemente realmente.

<Warning>
`tool-events` controla la transmisión en directo de la ejecución de herramientas. Gateway registra únicamente
como destinatarias de los eventos estructurados de herramientas de una ejecución las conexiones que anuncian
esta capacidad. Sin ella, la conexión no recibe eventos de herramientas en directo y el
protocolo de enlace no informa de ningún error.
</Warning>

Las herramientas del agente restringidas por capacidades constituyen un uso independiente de la misma declaración. Si una
herramienta del agente requiere una capacidad del cliente, Gateway omite esa herramienta salvo que el
cliente de origen haya anunciado todas las capacidades requeridas.

## Recuperar el estado tras una reconexión

Trata cada reconexión correcta como una nueva proyección sobre el historial persistente y
el estado actual de las ejecuciones en memoria:

1. Restablece `sessions.subscribe` y la suscripción
   `sessions.messages.subscribe` de la sesión seleccionada.
2. Llama a `chat.history` para el `sessionKey` seleccionado y sustituye las filas persistentes
   locales por la proyección `messages` devuelta.
3. Si `inFlightRun` está presente, adopta su `runId`, el `text` almacenado en búfer y el
   `plan` opcional. Adopta la ejecución incluso cuando `text` esté vacío.
4. Lee `sessionInfo.hasActiveRun` y `sessionInfo.activeRunIds`. Al determinar si una ejecución conservada sigue controlando
   la interfaz de transmisión, da preferencia a la pertenencia exacta a `activeRunIds`.
   Un `hasActiveRun` verdadero sin ningún ID enumerado puede representar otra
   proyección de ejecución activa.
5. Concilia los eventos `agent` posteriores mediante `payload.runId` y `payload.seq`.
   Mantén de forma independiente la secuencia aceptada más alta para cada ejecución, ignora una
   secuencia ya vista o inferior y considera un salto hacia delante como motivo para volver a cargar
   el historial autorizado.

La trama externa del evento también tiene un `seq` opcional, que ordena los eventos en la
conexión WebSocket actual. Se reinicia con cada conexión nueva. El `seq` dentro de
la carga útil de un evento `agent` se asigna por ejecución y ordena el ciclo de vida,
el asistente, el plan, las herramientas y los demás eventos transmitidos de esa ejecución.

## Usar metadatos del historial y anclajes estables

Las filas devueltas por `chat.history` pueden incluir un contenedor de metadatos `__openclaw`:

- `id` es la identidad de la entrada de la transcripción. Úsala para solicitudes de historial ancladas,
  pero no como clave única de una fila mostrada.
- `seq` es la secuencia positiva del registro de la transcripción. Un registro almacenado puede proyectarse
  en más de una fila mostrada, por lo que deben mantenerse juntas las filas relacionadas con el mismo `id` y la misma secuencia.
- `kind` identifica las filas sintéticas. Un límite de Compaction usa
  `kind: "compaction"` y puede incluir `tokensBefore` y `tokensAfter` cuando un
  punto de control correspondiente haya registrado esas métricas.

Retrocede por páginas mediante los valores `hasMore` y `nextOffset` de la respuesta. Los desplazamientos
numéricos describen la proyección actual de la transcripción, por lo que no deben conservarse como
marcadores de larga duración entre reinicios o procesos de Compaction. Conserva `__openclaw.id` en su lugar.
Para restaurar el contexto en torno a una fila conocida, llama a `chat.history` con `messageId` y el
`sessionId` que lo devolvió. Gateway puede resolver ese anclaje a partir del historial
archivado tras el reinicio; las respuestas ancladas omiten intencionadamente los metadatos numéricos de paginación.

## Suscribirse en lugar de consultar periódicamente el uso

Carga el catálogo inicial con `sessions.list` y, después, llama a `sessions.subscribe` una vez
por conexión. Combina los eventos `sessions.changed` mediante `sessionKey`. Las cargas útiles de cambios
de sesión pueden incluir datos en directo de `inputTokens`, `outputTokens`, `totalTokens`,
`totalTokensFresh`, `contextTokens`, `estimatedCostUsd`, ajustes de uso de respuestas
y el estado de las ejecuciones activas.

Algunas notificaciones de cambios solo son señales de invalidación. Si un evento omite los
campos de fila que necesita la vista, actualiza `sessions.list`. No consultes periódicamente `usage.cost` ni
`sessions.usage` para mantener actualizada una lista de sesiones en directo; reserva esos métodos para
informes agregados o detallados bajo demanda.

## Recuperar aprobaciones de ejecución anteriores

Un cliente con `operator.approvals` debe instalar su receptor de eventos en cuanto
termine `hello-ok` y, después, llamar a `exec.approval.list` para recuperar las solicitudes
anteriores a la conexión. Concilia la lista y los eventos en directo
`exec.approval.requested` / `exec.approval.resolved` mediante el ID de aprobación para que una
transición simultánea a la solicitud de la lista no se pierda ni vuelva a aparecer.

## Controlar las versiones del protocolo

La versión actual del protocolo de comunicación es `4`. Los clientes generales de operador y WebChat deben
negociar la versión actual exacta con `minProtocol: 4` y `maxProtocol: 4`.
Solo los clientes Node autenticados y las sondas ligeras disponen del intervalo de aceptación
N-1, que actualmente abarca desde el protocolo `3` hasta `4`.

Los cambios del protocolo son primero aditivos. `protocol.schema.json` incluye metadatos de antigüedad
de la versión `since` y metadatos de los ámbitos requeridos para los métodos principales, pero un
incremento de la versión del protocolo de comunicación sigue siendo un cambio incompatible explícito para los clientes de terceros. Fija las
versiones de los paquetes que pruebes, actualiza el cliente y Gateway conjuntamente cuando cambie la versión
del protocolo de comunicación y revisa el
[registro de cambios de OpenClaw](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)
antes de cada actualización.

## Contenido relacionado

- [Protocolo de Gateway](https://docs.openclaw.ai/gateway/protocol)
- [Incorporación de OpenClaw](https://docs.openclaw.ai/gateway/embedding)
- [Referencia de RPC de Gateway](https://docs.openclaw.ai/reference/rpc)
- [Integraciones de Gateway para aplicaciones externas](https://docs.openclaw.ai/gateway/external-apps)
