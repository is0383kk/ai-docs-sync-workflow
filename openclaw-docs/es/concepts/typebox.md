---
read_when:
    - Actualización de esquemas de protocolo o generación de código
summary: Esquemas de TypeBox como única fuente de verdad para el protocolo del Gateway
title: TypeBox
x-i18n:
    generated_at: "2026-07-26T05:06:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 24490edf0d73e918f834e9dd53d09ba0e5183b2bc126ee981a94f8099e76283b
    source_path: concepts/typebox.md
    workflow: 16
---

TypeBox es una biblioteca de esquemas diseñada principalmente para TypeScript. OpenClaw la utiliza para definir el **protocolo WebSocket del Gateway** (negociación inicial, solicitud/respuesta y eventos del servidor). Esos esquemas controlan la **validación en tiempo de ejecución** (AJV), la **exportación de JSON Schema** y la **generación de código Swift** para la aplicación de macOS. Una única fuente de verdad; todo lo demás se genera.

Para conocer el contexto de más alto nivel del protocolo, comience por la [arquitectura del Gateway](/es/concepts/architecture).

## Modelo mental (30 segundos)

Cada mensaje WS del Gateway es uno de estos tres tipos de trama:

- **Solicitud**: `{ type: "req", id, method, params }`
- **Respuesta**: `{ type: "res", id, ok, payload | error }`
- **Evento**: `{ type: "event", event, payload, seq?, stateVersion? }`

La primera trama **debe** ser una solicitud `connect`. Después, los clientes llaman a métodos (por ejemplo, `health`, `send`, `chat.send`) y se suscriben a eventos (por ejemplo, `presence`, `tick`, `agent`).

Flujo de conexión (mínimo):

```text
Cliente                   Gateway
  |---- sol.:connect ------->|
  |<---- resp.:hello-ok ------|
  |<---- evento:tick ---------|
  |---- sol.:health --------->|
  |<---- resp.:health --------|
```

Métodos y eventos habituales:

| Categoría   | Ejemplos                                                   | Notas                                        |
| ---------- | ---------------------------------------------------------- | -------------------------------------------- |
| Núcleo       | `connect`, `health`, `status`                              | `connect` debe ser el primero                      |
| Mensajería  | `send`, `agent`, `agent.wait`, `system-event`, `logs.tail` | los métodos con efectos secundarios necesitan `idempotencyKey` |
| Chat       | `chat.history`, `chat.send`, `chat.abort`                  | WebChat utiliza estos                           |
| Sesiones   | `sessions.list`, `sessions.patch`, `sessions.delete`       | administración de sesiones                                |
| Automatización | `wake`, `cron.list`, `cron.run`, `cron.runs`               | control de activación y cron                        |
| Nodos      | `node.list`, `node.invoke`, `node.pair.*`                  | WS del Gateway y acciones de nodo                 |
| Eventos     | `tick`, `presence`, `agent`, `chat`, `health`, `shutdown`  | envío desde el servidor                                  |

El inventario autoritativo de **detección** anunciado se encuentra en `src/gateway/server-methods-list.ts` (`listGatewayMethods`, `GATEWAY_EVENTS`).

## Ubicación de los esquemas

- Módulo de exportación de origen: `packages/gateway-protocol/src/schema.ts` vuelve a exportar los módulos de dominio de `packages/gateway-protocol/src/schema/*.ts` (`frames.ts` para los envoltorios de nivel superior y la negociación inicial, y `agent.ts`, `sessions.ts`, `cron.ts`, etc., para cada área funcional). `protocol-schemas.ts` es el registro central `ProtocolSchemas` que asigna los nombres de esquema a sus definiciones de TypeBox.
- Validadores en tiempo de ejecución (AJV): `packages/gateway-protocol/src/index.ts`
- Registro anunciado de funciones y detección: `src/gateway/server-methods-list.ts`
- Negociación inicial del servidor y despacho de métodos: `src/gateway/server.impl.ts`
- Cliente de nodo: `src/gateway/client.ts`
- JSON Schema generado: `dist/protocol.schema.json` (salida de compilación, no incluida en los commits)
- Modelos Swift generados: `apps/shared/OpenClawKit/Sources/OpenClawProtocol/GatewayModels.swift`

## Pipeline actual

- `pnpm protocol:gen` escribe JSON Schema (borrador 07) en `dist/protocol.schema.json`.
- `pnpm protocol:gen:swift` genera los modelos Swift del Gateway.
- `pnpm protocol:check` ejecuta ambos generadores y verifica que la salida de Swift esté incluida en los commits (la salida de JSON Schema es un artefacto de compilación ignorado por Git).

## Uso de los esquemas en tiempo de ejecución

- **En el servidor**: cada trama entrante se valida con AJV. La negociación inicial solo acepta una solicitud `connect` cuyos parámetros coincidan con `ConnectParams`.
- **En el cliente**: el cliente JS valida las tramas de eventos y respuestas antes de utilizarlas.
- **Detección de funciones**: el Gateway envía listas conservadoras `features.methods` y `features.events` en `hello-ok`, provenientes de `listGatewayMethods()` y `GATEWAY_EVENTS`.
- Esa lista de detección no es un volcado generado de todas las funciones auxiliares invocables de `coreGatewayHandlers`; algunos RPC auxiliares están implementados en `src/gateway/server-methods/*.ts` sin estar enumerados en la lista de funciones anunciadas.

## Tramas de ejemplo

Conexión (primer mensaje):

```json
{
  "type": "req",
  "id": "c1",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 4,
    "client": {
      "id": "openclaw-macos",
      "displayName": "macos",
      "version": "1.0.0",
      "platform": "macos 15.1",
      "mode": "ui",
      "instanceId": "A1B2"
    }
  }
}
```

Respuesta hello-ok:

```json
{
  "type": "res",
  "id": "c1",
  "ok": true,
  "payload": {
    "type": "hello-ok",
    "protocol": 4,
    "server": { "version": "dev", "connId": "ws-1" },
    "features": { "methods": ["health"], "events": ["tick"] },
    "snapshot": {
      "presence": [],
      "health": {},
      "stateVersion": { "presence": 0, "health": 0 },
      "uptimeMs": 0
    },
    "auth": { "role": "operator", "scopes": ["operator.read"] },
    "policy": { "maxPayload": 1048576, "maxBufferedBytes": 1048576, "tickIntervalMs": 30000 }
  }
}
```

Solicitud y respuesta:

```json
{ "type": "req", "id": "r1", "method": "health" }
```

```json
{ "type": "res", "id": "r1", "ok": true, "payload": { "ok": true } }
```

Evento:

```json
{ "type": "event", "event": "tick", "payload": { "ts": 1730000000 }, "seq": 12 }
```

## Cliente mínimo (Node.js)

Flujo útil más sencillo: conexión + estado.

```ts
import { WebSocket } from "ws";

const ws = new WebSocket("ws://127.0.0.1:18789");

ws.on("open", () => {
  ws.send(
    JSON.stringify({
      type: "req",
      id: "c1",
      method: "connect",
      params: {
        minProtocol: 4,
        maxProtocol: 4,
        client: {
          id: "cli",
          displayName: "example",
          version: "dev",
          platform: "node",
          mode: "cli",
        },
      },
    }),
  );
});

ws.on("message", (data) => {
  const msg = JSON.parse(String(data));
  if (msg.type === "res" && msg.id === "c1" && msg.ok) {
    ws.send(JSON.stringify({ type: "req", id: "h1", method: "health" }));
  }
  if (msg.type === "res" && msg.id === "h1") {
    console.log("health:", msg.payload);
    ws.close();
  }
});
```

## Ejemplo práctico: añadir un método de extremo a extremo

Ejemplo: añadir una nueva solicitud `system.echo` que devuelva `{ ok: true, text }`.

1. **Esquema (fuente de verdad)**

Añádalo a `packages/gateway-protocol/src/schema/system.ts` (o al módulo funcional que mejor corresponda):

```ts
export const SystemEchoParamsSchema = Type.Object(
  { text: NonEmptyString },
  { additionalProperties: false },
);

export const SystemEchoResultSchema = Type.Object(
  { ok: Type.Boolean(), text: NonEmptyString },
  { additionalProperties: false },
);
```

Importe ambos en `packages/gateway-protocol/src/schema/protocol-schemas.ts`, añádalos al registro `ProtocolSchemas` y exporte los tipos derivados:

```ts
  SystemEchoParams: SystemEchoParamsSchema,
  SystemEchoResult: SystemEchoResultSchema,
```

```ts
export type SystemEchoParams = Static<typeof SystemEchoParamsSchema>;
export type SystemEchoResult = Static<typeof SystemEchoResultSchema>;
```

2. **Validación**

En `packages/gateway-protocol/src/index.ts`, exporte un validador AJV:

```ts
export const validateSystemEchoParams = ajv.compile<SystemEchoParams>(SystemEchoParamsSchema);
```

3. **Comportamiento del servidor**

Añada un controlador en `src/gateway/server-methods/system.ts`:

```ts
export const systemHandlers: GatewayRequestHandlers = {
  "system.echo": ({ params, respond }) => {
    const text = String(params.text ?? "");
    respond(true, { ok: true, text });
  },
};
```

Regístrelo en `src/gateway/server-methods.ts` (que ya combina `systemHandlers`) y después añada `"system.echo"` a la entrada `listGatewayMethods` de `src/gateway/server-methods-list.ts`.

Si el método puede ser invocado por clientes operadores o nodos, clasifíquelo también en `src/gateway/method-scopes.ts` para que la aplicación de ámbitos y el anuncio de funciones `hello-ok` permanezcan alineados.

4. **Regeneración**

```bash
pnpm protocol:check
```

5. **Pruebas y documentación**

Añada una prueba del servidor en `src/gateway/server.*.test.ts` y mencione el método en la documentación.

## Comportamiento de la generación de código Swift

El generador de Swift emite:

- una enumeración `GatewayFrame` con los casos `req`, `res`, `event` y `unknown`
- estructuras y enumeraciones de carga útil con tipado fuerte
- valores `ErrorCode`, `GATEWAY_PROTOCOL_VERSION` y `GATEWAY_MIN_PROTOCOL_VERSION`

Los tipos de trama desconocidos se conservan como cargas útiles sin procesar para garantizar la compatibilidad futura.

## Control de versiones y compatibilidad

- `PROTOCOL_VERSION` se encuentra en `packages/gateway-protocol/src/version.ts` (valor actual: `4`).
- Los clientes envían `minProtocol` y `maxProtocol`; el servidor rechaza los intervalos que no incluyen su protocolo actual.
- Los modelos Swift conservan los tipos de trama desconocidos para evitar que los clientes antiguos dejen de funcionar.

## Patrones y convenciones de los esquemas

- La mayoría de los objetos utilizan `additionalProperties: false` para cargas útiles estrictas.
- `NonEmptyString` (`Type.String({ minLength: 1 })`) es el valor predeterminado para los identificadores y los nombres de métodos y eventos.
- El `GatewayFrame` de nivel superior utiliza un **discriminador** en `type`.
- Los métodos con efectos secundarios suelen requerir un `idempotencyKey` en sus parámetros (ejemplo: `send`, `poll`, `agent`, `chat.send`).
- `agent` acepta el parámetro opcional `internalEvents` para el contexto de orquestación generado en tiempo de ejecución (por ejemplo, la entrega al finalizar una tarea de subagente o cron); debe tratarse como una superficie de API interna.

## JSON del esquema en directo

El JSON Schema generado es un artefacto de compilación y no se incluye en los commits del repositorio. El archivo sin procesar publicado suele estar disponible en:

- [https://raw.githubusercontent.com/openclaw/openclaw/main/dist/protocol.schema.json](https://raw.githubusercontent.com/openclaw/openclaw/main/dist/protocol.schema.json)

## Al modificar los esquemas

1. Actualice los esquemas de TypeBox en el módulo `packages/gateway-protocol/src/schema/*.ts` propietario y regístrelos en `protocol-schemas.ts`.
2. Registre el método o evento en `src/gateway/server-methods-list.ts`.
3. Actualice `src/gateway/method-scopes.ts` cuando el nuevo RPC necesite una clasificación de ámbito de operador o nodo.
4. Ejecute `pnpm protocol:check`.
5. Incluya los modelos Swift regenerados en el commit.

## Contenido relacionado

- [Protocolo de salida enriquecida](/es/reference/rich-output-protocol)
- [Adaptadores RPC](/es/reference/rpc)
