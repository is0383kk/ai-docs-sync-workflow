---
read_when:
    - Implementación o actualización de clientes WS del Gateway
    - Depuración de incompatibilidades de protocolo o fallos de conexión
    - Regeneración del esquema y los modelos del protocolo
summary: 'Protocolo WebSocket del Gateway: negociación, tramas y control de versiones'
title: Protocolo del Gateway
x-i18n:
    generated_at: "2026-07-26T05:13:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89d637a9070bc6512a182fea0fd890b56287e0080515ba4fba9b2591c6247e0d
    source_path: gateway/protocol.md
    workflow: 16
---

El protocolo WS del Gateway es el único plano de control y transporte de nodos de
OpenClaw. Los clientes de operador y nodo (CLI, interfaz web, aplicación para macOS, nodos iOS/Android,
nodos sin interfaz) se conectan mediante WebSocket y declaran un **rol** y un **ámbito** durante
el protocolo de enlace.

## Paquetes npm

Estos paquetes se distribuyen con los ciclos de lanzamiento de OpenClaw. Durante el despliegue inicial,
npm puede devolver `E404` hasta que se publique la primera versión que incluya paquetes.

- [`@openclaw/gateway-protocol`](https://www.npmjs.com/package/@openclaw/gateway-protocol)
  publica los esquemas, validadores, tipos de TypeScript, utilidades ligeras para tramas y errores,
  y constantes de versión. Su archivo tar incluye el contrato generado
  [`protocol.schema.json`](https://unpkg.com/@openclaw/gateway-protocol/protocol.schema.json)
  legible por máquina.
- [`@openclaw/gateway-client`](https://www.npmjs.com/package/@openclaw/gateway-client)
  publica el cliente Node de referencia y un punto de entrada compatible con navegadores en
  `@openclaw/gateway-client/browser`.

Para obtener orientación sobre el ciclo de vida de las aplicaciones, consulte
[Crear un cliente de Gateway](https://docs.openclaw.ai/gateway/clients). Para las aplicaciones
que supervisan el Gateway como proceso secundario, consulte
[Integrar OpenClaw](https://docs.openclaw.ai/gateway/embedding).

## Transporte y entramado

- WebSocket, tramas de texto, cargas útiles JSON.
- La primera trama **debe** ser una solicitud `connect`.
- Las tramas previas a la conexión tienen un límite de 64 KiB (`MAX_PREAUTH_PAYLOAD_BYTES`). Después
  del protocolo de enlace, se siguen `hello-ok.policy.maxPayload` y
  `hello-ok.policy.maxBufferedBytes`. Con los diagnósticos habilitados, las tramas
  entrantes sobredimensionadas y los búferes salientes lentos emiten eventos `payload.large` antes de que
  el Gateway cierre o descarte la trama. Estos eventos contienen `surface`, tamaños
  en bytes, límites y un código de motivo seguro, pero nunca cuerpos de mensajes, contenido
  de archivos adjuntos, bytes de trama sin procesar, tokens, cookies ni secretos.

Formas de las tramas:

- Solicitud: `{type:"req", id, method, params}`
- Respuesta: `{type:"res", id, ok, payload|error}`
- Evento: `{type:"event", event, payload, seq?, stateVersion?}`

Los errores de respuesta usan `{ code, message, details?, retryable?, retryAfterMs? }`.
Los clientes deben realizar la bifurcación según `code` y `details.code`; `message` sigue siendo legible
para las personas y puede cambiar, excepto cuando una nota de compatibilidad indique lo contrario. Los fallos de
autorización a nivel de método usan `code: "FORBIDDEN"` en el nivel superior, con
detalles estructurados de los ámbitos ausentes:

- Ámbito ausente: `{ code: "MISSING_SCOPE", missingScope, requiredScopes }`.
  `requiredScopes` es el conjunto completo de ámbitos conocidos para la operación solicitada.
  El mensaje heredado `missing scope: <scope>` se conserva para clientes antiguos.

Los clientes deben leer primero `details` y usar el mensaje heredado únicamente como alternativa
de compatibilidad. `readMissingScopeError` y `readMissingScopeErrorDetails` se exportan desde
`@openclaw/gateway-protocol/gateway-error-details`; el cliente de Gateway compatible con navegadores
los reexporta desde `@openclaw/gateway-client/browser`.

Los esquemas se exportan como `GatewayErrorDetailsSchema`,
`MissingScopeErrorDetailsSchema` desde `@openclaw/gateway-protocol/schema`.
Los fallos de ámbito HTTP reflejan el objeto `MISSING_SCOPE` bajo `error.details` y
usan el estado HTTP `403`.

Los métodos con efectos secundarios requieren claves de idempotencia (consulte el esquema).

## Protocolo de enlace

El Gateway envía un desafío previo a la conexión:

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

El cliente responde con `connect`:

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 4,
    "maxProtocol": 4,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

El Gateway responde con `hello-ok`:

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": {
    "type": "hello-ok",
    "protocol": 4,
    "server": { "version": "…", "connId": "…" },
    "features": { "methods": ["…"], "events": ["…"] },
    "snapshot": { "…": "…" },
    "auth": {
      "role": "operator",
      "scopes": ["operator.read", "operator.write"]
    },
    "policy": {
      "maxPayload": 26214400,
      "maxBufferedBytes": 52428800,
      "tickIntervalMs": 15000
    }
  }
}
```

`server`, `features`, `snapshot`, `policy` y `auth` son obligatorios para
`HelloOkSchema` (`packages/gateway-protocol/src/schema/frames.ts`). `auth`
informa del rol y los ámbitos negociados incluso cuando no se emite ningún token de dispositivo (forma
anterior). `pluginSurfaceUrls` es opcional y asigna nombres de superficies de Plugin (por ejemplo,
`canvas`) a URL alojadas con ámbito; puede caducar, por lo que los nodos llaman a
`node.pluginSurface.refresh` con `{ "surface": "canvas" }` para obtener una entrada nueva.
La ruta obsoleta `canvasHostUrl` / `canvasCapability` / `node.canvas.capability.refresh`
no es compatible; use superficies de Plugin.
El campo opcional `appliedConfigHash` de la instantánea es la revisión resuelta de la configuración de origen
aceptada por el entorno de ejecución activo del Gateway. Los clientes pueden compararla con
`config.get.configRevisionHash` para determinar si una configuración guardada más reciente aún
requiere un reinicio. `config.get.hash` sigue siendo la revisión sin procesar del archivo raíz usada por
las protecciones contra conflictos de escritura de configuración.

Mientras el Gateway aún termina de iniciar los procesos auxiliares, `connect` puede devolver un
error reintentable `UNAVAILABLE` con `details.reason: "startup-sidecars"` y
`retryAfterMs`. Vuelva a intentarlo dentro del presupuesto de conexión en lugar de tratarlo como
un fallo terminal del protocolo de enlace.

Cuando se emite un token de dispositivo, `hello-ok.auth` lo añade:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

La inicialización integrada mediante QR/código de configuración es una ruta de transferencia móvil. Una
conexión correcta con el código de configuración de referencia devuelve un token de nodo principal y un
token de operador limitado:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

Esta transferencia al operador se limita deliberadamente: es suficiente para iniciar el bucle del
operador móvil y la configuración nativa, incluido `operator.talk.secrets` para las lecturas
de configuración de Talk, pero sin ámbitos para modificar el emparejamiento ni `operator.admin`. Un acceso más amplio
de emparejamiento o administración requiere un flujo separado de emparejamiento aprobado o de tokens. Conserve
`hello-ok.auth.deviceTokens` únicamente cuando la autenticación de inicialización se haya ejecutado mediante un transporte
de confianza (`wss://` o emparejamiento de bucle invertido/local).

Los clientes de backend de confianza dentro del mismo proceso (`client.id: "gateway-client"`,
`client.mode: "backend"`) pueden omitir `device` en conexiones directas de bucle invertido cuando
se autentican con el token o la contraseña compartidos del Gateway. Esta ruta está reservada
para RPC internas del plano de control (por ejemplo, actualizaciones de sesiones de subagentes) y evita
que las referencias de emparejamiento obsoletas de la CLI o del dispositivo bloqueen el trabajo local del backend. Los clientes remotos,
con origen en navegador, de nodo y los clientes explícitos de token o identidad de dispositivo siguen
los controles normales de emparejamiento y ampliación de ámbitos.

### Rol de trabajador y protocolo cerrado

Los trabajadores en la nube usan una entrada de bucle invertido dedicada a través del túnel SSH propiedad del Gateway,
con la clave del host fijada. Solo acepta la identidad del trabajador y nunca distribuye
autenticación general, eventos de nodo, RPC de operador ni métodos de Plugin. Un `connect` estricto
verifica una credencial de corta duración almacenada como hash y vinculada al entorno, al hash
del paquete, a la época del propietario, a la versión del conjunto de RPC, a la caducidad y a una sesión anulable; también
comprueba por separado la versión y el conjunto de características actuales. El éxito devuelve un
`worker-hello-ok` mínimo; la negociación de características es independiente de la versión general
del protocolo. Las tramas se mantienen por debajo de 64 KiB, excepto una trama `worker.inference.start`
negociada, que puede alcanzar 25 MiB. La lista cerrada de permitidos contiene `worker.heartbeat`,
`worker.transcript.commit`, `worker.live-event`, `worker.inference.start` y
`worker.inference.cancel`.

Las confirmaciones de transcripciones usan delimitación por época del propietario, una vinculación de sesión
propiedad del Gateway, comparación e intercambio de la hoja base y repetición duradera de secuencias; el Gateway genera
los identificadores de las entradas de transcripción y de sus elementos superiores mediante el escritor normal de sesiones. La propiedad y
la caducidad se vuelven a comprobar en cada RPC.

### Capacidades del cliente

Los clientes de operador pueden anunciar capacidades opcionales en `connect.params.caps`:

- `tool-events`: acepta eventos estructurados del ciclo de vida de las herramientas.
- `inline-widgets`: puede representar resultados de herramientas de widgets integrados alojados.

Las capacidades del cliente describen al cliente conectado, no la autorización. Las herramientas del agente pueden declarar capacidades obligatorias; el Gateway omite esas herramientas a menos que todos los requisitos aparezcan en `caps` del cliente de origen. Las ejecuciones originadas en canales no tienen capacidades de cliente del Gateway, por lo que las herramientas restringidas por capacidades no están disponibles, incluso cuando la política de herramientas las permite explícitamente.

### Ejemplo de conexión de nodo

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 4,
    "maxProtocol": 4,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Los nodos declaran capacidades durante la conexión:

- `caps`: categorías de alto nivel como `camera`, `canvas`, `screen`,
  `location`, `voice`, `talk`.
- `commands`: lista de comandos permitidos para la invocación.
- `permissions`: controles granulares (por ejemplo, `screen.record`, `camera.capture`).

El Gateway los trata como declaraciones y aplica listas de permitidos en el servidor.

## Roles y ámbitos

Para consultar el modelo completo de ámbitos del operador, las comprobaciones durante la aprobación y la semántica
de los secretos compartidos, consulte [Ámbitos del operador](/es/gateway/operator-scopes).

Roles:

- `operator`: cliente del plano de control (CLI/interfaz de usuario/automatización).
- `node`: host de capacidades (cámara/pantalla/lienzo/system.run).
- `worker`: host de ejecución en la nube mediante el protocolo de trabajador dedicado y cerrado.

Ámbitos del operador (`src/gateway/operator-scopes.ts`), el conjunto cerrado completo:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`talk.config` con `includeSecrets: true` requiere `operator.talk.secrets` (o
`operator.admin`). Cuando se incluyen secretos, lea la credencial activa del proveedor de Talk
desde `talk.resolved.config.apiKey`; `talk.providers.<id>.apiKey`
conserva la forma de origen y puede ser un objeto SecretRef o una cadena censurada.

Los métodos RPC del Gateway registrados por plugins pueden solicitar su propio ámbito de operador,
pero estos prefijos reservados del núcleo siempre se resuelven como `operator.admin`
(`src/shared/gateway-method-policy.ts`): `config.*`, `exec.approvals.*`,
`wizard.*`, `update.*`.

El ámbito del método es solo la primera barrera. Algunos comandos con barra diagonal a los que se accede mediante
`chat.send` aplican comprobaciones más estrictas a nivel de comando: las escrituras persistentes `/config set` y
`/config unset` requieren `operator.admin` incluso para clientes del Gateway que
ya poseen un ámbito de operador inferior.

`node.pair.approve` tiene una comprobación de ámbito adicional durante la aprobación, además del
ámbito base del método (`operator.pairing`), basada en el `commands` declarado
por la solicitud pendiente (`src/infra/node-pairing-authz.ts`):

| Comandos declarados                                                                                                           | Ámbitos requeridos                    |
| ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| ninguno                                                                                                                       | `operator.pairing`                    |
| comandos ordinarios                                                                                                           | `operator.pairing` + `operator.write` |
| incluye `system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `fs.listDir` o `system.execApprovals.get/set` | `operator.pairing` + `operator.admin` |

### Capacidades/comandos/permisos (nodo)

Los nodos declaran afirmaciones de capacidades al conectarse:

- `caps`: categorías de capacidades de alto nivel, como `camera`, `canvas`, `screen`,
  `location`, `voice` y `talk`.
- `commands`: lista de comandos permitidos para la invocación.
- `permissions`: opciones granulares (p. ej., `screen.record`, `camera.capture`).

El Gateway las trata como **afirmaciones** y aplica listas de elementos permitidos en el servidor.
Los nodos conectados pueden publicar descriptores opcionales de herramientas de plugins o MCP visibles para el agente
mediante `node.pluginTools.update` después de conectarse o
reconectarse correctamente. Los hosts de nodos sin interfaz reinician para aplicar los cambios declarativos
del inventario MCP. Este método de actualización es la única vía de publicación; no se aceptan descriptores de herramientas de plugins en los
parámetros de `connect`. Cada descriptor debe usar un `name` de herramienta seguro para el proveedor y nombrar
un `command` incluido en la lista actual de comandos permitidos del nodo. El Gateway confía en los metadatos
de los descriptores procedentes del nodo emparejado, filtra los descriptores que están fuera de la superficie de comandos
aprobada, los elimina cuando el nodo se desconecta y rechaza los intentos del operador
de modificar el catálogo de otro nodo. Establezca `gateway.nodes.pluginTools.enabled: false`
para ignorar los descriptores publicados por los nodos.

Los hosts de nodos conectados publican su catálogo completo de reemplazo de Skills mediante
`node.skills.update`. Este método del rol de nodo es la única vía de publicación
de Skills del nodo; no se aceptan Skills en los parámetros de `connect`. Cada descriptor contiene un
nombre seguro, una descripción y contenido limitado de `SKILL.md`. El Gateway analiza ese
contenido con el cargador normal de Skills, lo incluye en las instantáneas de Skills del agente
mientras el nodo está conectado y lo elimina al desconectarse. Establezca
`gateway.nodes.allowSkills: false` para ignorar las Skills publicadas por los nodos.

## Presencia

- `system-presence` devuelve entradas indexadas por la identidad del dispositivo, que incluyen
  `deviceId`, `roles` y `scopes`, para que las interfaces puedan mostrar una fila por dispositivo incluso
  cuando se conecta como operador y como nodo.
- `node.list` incluye los valores opcionales `lastSeenAtMs` y `lastSeenReason`. Los nodos
  conectados informan de la hora de conexión actual con el motivo `connect`; los nodos emparejados también pueden
  informar de una presencia persistente en segundo plano mediante un evento de nodo de confianza.

Los nodos nativos de macOS también pueden enviar eventos autenticados `node.presence.activity`
con un tiempo de inactividad de entrada limitado. El Gateway deriva las marcas de tiempo de actividad con su
propio reloj, expone el Mac conectado más reciente mediante `node.list` y
`node.describe`, y difunde actualizaciones de `node.presence` a los clientes con ámbito de lectura.
La aplicación envía `{ "action": "clear" }` cuando se desactiva el uso compartido de actividad;
el Gateway borra las marcas de tiempo solo para esa conexión exacta del nodo autenticado.
Los Gateways anteriores a esta acción confirmada la devuelven como no gestionada, por lo que el nodo
Mac se reconecta una vez y permite que la limpieza de la desconexión elimine el estado de la conexión anterior.
Consulte [Presencia del ordenador activo](/es/nodes/presence) para conocer el comportamiento de selección, privacidad, contexto
del modelo y enrutamiento de notificaciones.

### Evento de actividad en segundo plano del nodo

Los nodos llaman a `node.event` con `event: "node.presence.alive"` para registrar que un
nodo emparejado estuvo activo durante una reactivación en segundo plano, sin marcarlo como conectado:

```json
{
  "event": "node.presence.alive",
  "payloadJSON": "{\"trigger\":\"silent_push\",\"sentAtMs\":1737264000000,\"displayName\":\"iPhone de Peter\",\"version\":\"2026.4.28\",\"platform\":\"iOS 18.4.0\",\"deviceFamily\":\"iPhone\",\"modelIdentifier\":\"iPhone17,1\",\"pushTransport\":\"relay\"}"
}
```

`trigger` es una enumeración cerrada: `background`, `silent_push`, `bg_app_refresh`,
`significant_location`, `manual`, `connect`. Los valores desconocidos se normalizan como
`background` (`src/shared/node-presence.ts`). El evento solo se conserva para
sesiones autenticadas de dispositivos de nodo; las sesiones sin dispositivo o sin emparejar devuelven
`handled: false`.

Los Gateways compatibles devuelven un resultado estructurado:

```json
{
  "ok": true,
  "event": "node.presence.alive",
  "handled": true,
  "reason": "persisted"
}
```

Los Gateways más antiguos pueden devolver únicamente `{ "ok": true }` para `node.event`; debe tratarse
como una RPC confirmada, no como persistencia duradera de la presencia.

## Ámbito de los eventos de difusión

Los eventos de difusión enviados por el servidor se restringen por ámbito para que las sesiones
limitadas al emparejamiento o exclusivas de nodos no reciban pasivamente contenido de sesión
(`src/gateway/server-broadcast.ts`):

- Los fotogramas de chat, agente y resultados de herramientas (eventos `agent` transmitidos, eventos
  de resultados de herramientas) requieren al menos `operator.read`. Las sesiones que no lo tengan omiten estos
  fotogramas por completo.
- Las difusiones `plugin.*` definidas por plugins se restringen de forma predeterminada a `operator.write` o
  `operator.admin`; las entradas explícitas como
  `plugin.approval.requested` / `plugin.approval.resolved` usan
  `operator.approvals` en su lugar.
- Los eventos de estado/transporte (`heartbeat`, `presence`, `tick` y el ciclo de vida
  de conexión/desconexión) permanecen sin restricciones para que todas las sesiones
  autenticadas puedan observar el estado del transporte.
- Las familias desconocidas de eventos de difusión se restringen por ámbito de forma predeterminada (se deniega
  en caso de duda), salvo que un gestor registrado las flexibilice explícitamente.

Cada conexión de cliente mantiene su propio número de secuencia por cliente, por lo que las difusiones
conservan un orden monótono en ese socket incluso cuando distintos clientes ven
subconjuntos diferentes del flujo de eventos filtrados por ámbito.

## Familias de métodos RPC

`hello-ok.features.methods` es una lista conservadora de descubrimiento creada a partir de
`src/gateway/server-methods-list.ts` y de las exportaciones de métodos de plugins/canales
cargados; no es un volcado generado de todos los métodos, y algunos métodos (por
ejemplo, `push.test`, `web.login.start`, `web.login.wait`, `sessions.usage`)
se excluyen intencionadamente del descubrimiento aunque sean métodos reales que
se pueden invocar. Debe tratarse como descubrimiento de funciones, no como una enumeración completa de
`src/gateway/server-methods/*.ts`.

<AccordionGroup>
  <Accordion title="Sistema e identidad">
    - `health` devuelve la instantánea almacenada en caché o recién sondeada del estado del Gateway.
    - `diagnostics.stability` devuelve el registro reciente y limitado de estabilidad diagnóstica: nombres de eventos, recuentos, tamaños en bytes, lecturas de memoria, estado de colas/sesiones, nombres de canales/plugins e identificadores de sesión. No incluye texto de chats, cuerpos de Webhooks, salidas de herramientas, cuerpos sin procesar de solicitudes/respuestas, tokens, cookies ni secretos. Requiere `operator.read`.
    - `status` devuelve el resumen del Gateway con el formato de `/status`; los campos confidenciales solo se incluyen para clientes operadores con ámbito de administración.
    - `gateway.identity.get` devuelve la identidad del dispositivo del Gateway utilizada por los flujos de retransmisión y emparejamiento.
    - `system-presence` devuelve la instantánea de presencia actual de los dispositivos operadores/nodos conectados.
    - `system-event` añade un evento del sistema y puede actualizar/difundir el contexto de presencia.
    - `last-heartbeat` devuelve el último evento de Heartbeat conservado.
    - `set-heartbeats` activa o desactiva el procesamiento de Heartbeat en el Gateway.
    - `gateway.suspend.prepare` crea un arrendamiento breve de suspensión cooperativa solo cuando el trabajo del Gateway que se supervisa está inactivo. `gateway.suspend.status` comprueba ese arrendamiento y `gateway.suspend.resume` lo libera después de la reanudación o de una operación del host cancelada.

  </Accordion>

  <Accordion title="Modelos y uso">
    - `models.list` devuelve el catálogo de modelos permitidos durante la ejecución. Consulte las «vistas de `models.list`» más adelante.
    - `usage.status` devuelve resúmenes de las ventanas de uso y la cuota restante del proveedor.
    - `usage.cost` devuelve resúmenes agregados del uso de costes para un intervalo de fechas. Pase `agentId` para un agente o `agentScope: "all"` para agregar los agentes configurados.
    - `doctor.memory.status` devuelve la disponibilidad de la memoria vectorial y las incrustaciones almacenadas en caché para el espacio de trabajo del agente predeterminado activo. Pase `{ "probe": true }` o `{ "deep": true }` únicamente para realizar un sondeo explícito en vivo del proveedor de incrustaciones. Pase `{ "agentId": "agent-id" }` para limitar las estadísticas del almacén de Dreaming al espacio de trabajo de un agente; si se omite, se agregan los espacios de trabajo de Dreaming configurados.
    - `doctor.memory.dreamDiary`, `doctor.memory.backfillDreamDiary`, `doctor.memory.resetDreamDiary`, `doctor.memory.resetGroundedShortTerm`, `doctor.memory.repairDreamingArtifacts` y `doctor.memory.dedupeDreamDiary` aceptan el parámetro opcional `{ "agentId": "agent-id" }`; si se omite, operan en el espacio de trabajo del agente predeterminado configurado.
    - `doctor.memory.remHarness` devuelve una vista previa limitada y de solo lectura del entorno de pruebas REM para clientes remotos del plano de control, incluidas rutas del espacio de trabajo, fragmentos de memoria, Markdown fundamentado renderizado y candidatos para promoción profunda. Requiere `operator.read`.
    - `sessions.usage` devuelve resúmenes de uso por sesión. Pase `agentId` para un agente o `agentScope: "all"` para enumerar conjuntamente los agentes configurados.
      Ambos métodos de uso aceptan `mode: "specific"` con un `timeZone` de IANA para establecer límites y agrupaciones de días naturales que tengan en cuenta el horario de verano. `utcOffset` sigue siendo compatible con clientes antiguos y se usa como alternativa cuando el entorno de ejecución del Gateway no reconoce la zona solicitada.
    - `sessions.usage.timeseries` devuelve el uso en forma de serie temporal para una sesión.
    - `sessions.usage.logs` devuelve las entradas del registro de uso de una sesión.

  </Accordion>

  <Accordion title="Canales y asistentes de inicio de sesión">
    - `channels.status` devuelve resúmenes de estado de los canales/plugins integrados y agrupados.
    - `channels.logout` cierra la sesión de un canal/cuenta específicos cuando el canal lo admite.
    - `web.login.start` inicia un flujo de inicio de sesión mediante QR/web para el proveedor actual de canales web compatible con QR.
    - `web.login.wait` espera a que termine ese flujo e inicia el canal si se completa correctamente.
    - `push.test` envía una notificación push de prueba mediante APNs a un nodo iOS registrado.
    - `voicewake.get` devuelve los activadores de palabras de reactivación almacenados.
    - `voicewake.set` actualiza los activadores de palabras de reactivación y difunde el cambio.

  </Accordion>

  <Accordion title="Gestión de plugins">
    - `plugins.list` (`operator.read`) devuelve el inventario de plugins instalados, además de una selección oficial curada localmente, diagnósticos y si el modo de instalación actual permite modificaciones.
    - `plugins.search` (`operator.read`) busca familias instalables de plugins de código y plugins de paquete de ClawHub. Proporcione un valor no vacío para `query` y un valor opcional para `limit` de 1 a 100.
    - `plugins.install` (`operator.admin`) instala una entrada del catálogo oficial con `{ source: "official", pluginId }` o un paquete de ClawHub con `{ source: "clawhub", packageName, version?, acknowledgeClawHubRisk? }`. Las instalaciones de ClawHub conservan las comprobaciones de confianza, integridad y política de instalación del Gateway. Las instalaciones correctas requieren reiniciar el Gateway.
    - `plugins.setEnabled` (`operator.admin`) cambia la política de activación de un plugin instalado mediante `{ pluginId, enabled }`. La respuesta incluye la entrada de catálogo actualizada, los metadatos de reinicio y cualquier advertencia sobre la selección de ranuras.
    - `plugins.uninstall` (`operator.admin`) elimina un plugin instalado externamente mediante `{ pluginId }`: las referencias de configuración, el registro de instalación y los archivos administrados. Los plugins incluidos no se pueden desinstalar, solo desactivar. La respuesta enumera las acciones de eliminación y siempre requiere reiniciar el Gateway.

  </Accordion>

  <Accordion title="Mensajería y registros">
    - `send` es la RPC de entrega saliente directa para envíos dirigidos a canales, cuentas o hilos fuera del ejecutor de chat.
    - `logs.tail` devuelve la cola configurada del registro de archivos del Gateway con controles de cursor, límite y máximo de bytes.

  </Accordion>

  <Accordion title="Terminal del operador">
    - `terminal.open` inicia una PTY del host para un `agentId` explícito o para el agente predeterminado, y devuelve el agente resuelto, el directorio de trabajo, el shell y el estado de confinamiento.
    - `terminal.input`, `terminal.resize` y `terminal.close` operan únicamente sobre sesiones propiedad de la conexión que realiza la llamada.
    - `terminal.upload` acepta un archivo en base64 de hasta 16 MiB, lo almacena provisionalmente en un directorio temporal privado durante 24 horas en el Gateway de la sesión o en el host del nodo emparejado, y devuelve la ruta absoluta. Quien realiza la llamada debe pegar o utilizar de otro modo esa ruta; la RPC nunca escribe en la entrada del terminal ni ejecuta comandos.
    - Los eventos `terminal.data` y `terminal.exit` se transmiten únicamente a la conexión propietaria de la sesión.
    - Las sesiones cuya conexión se interrumpe se desvinculan, pero no se terminan: pueden volver a vincularse durante `gateway.terminal.detachedSessionTimeoutSeconds` (valor predeterminado: 300; `0` restablece la terminación al desconectarse), mientras la salida reciente se acumula en un búfer acotado del servidor.
    - `terminal.list` devuelve las sesiones que pueden vincularse; `terminal.attach` vuelve a vincular una sesión activa o desvinculada con la conexión que realiza la llamada y devuelve el búfer de reproducción (toma de control al estilo de tmux: el anterior propietario activo recibe `terminal.exit` con el motivo `detached`); `terminal.text` lee el búfer como texto sin formato sin vincularse.
    - Todos los métodos de terminal requieren `operator.admin`; `gateway.terminal.enabled` debe ser explícitamente verdadero. Se rechazan los agentes completamente aislados, y cualquier cambio en la política de un agente cierra las PTY existentes y en curso, incluidas las desvinculadas.

  </Accordion>

  <Accordion title="Conversación y TTS">
    - `talk.catalog` devuelve el catálogo de solo lectura de proveedores de conversación para voz, transcripción en streaming y voz en tiempo real: identificadores canónicos de proveedores, alias del registro, etiquetas, estado de configuración, un resultado opcional de `ready` a nivel de grupo, identificadores de modelos y voces expuestos, modos canónicos, transportes, estrategias del cerebro e indicadores de audio y capacidades en tiempo real, sin devolver secretos de proveedores ni modificar la configuración global. Los gateways actuales establecen `ready` tras aplicar la selección de proveedor en tiempo de ejecución; en gateways anteriores, su ausencia debe considerarse como no verificada.
    - `talk.config` devuelve la carga útil efectiva de configuración de conversación; `includeSecrets` requiere `operator.talk.secrets` (o `operator.admin`).
    - `talk.session.create` crea una sesión de conversación propiedad del gateway para `realtime/gateway-relay`, `transcription/gateway-relay` o `stt-tts/managed-room`. Para `stt-tts/managed-room`, quienes llaman a `operator.write` y proporcionan `sessionKey` también deben proporcionar `spawnedBy` para que la clave de sesión tenga visibilidad limitada al ámbito correspondiente; la creación de `sessionKey` sin ámbito y `brain: "direct-tools"` requieren `operator.admin`.
    - `talk.session.join` valida un token de sesión de sala administrada, emite `session.ready` o `session.replaced` según sea necesario y devuelve los metadatos de la sala y la sesión junto con eventos recientes de conversación, pero nunca el token en texto sin formato ni su hash.
    - `talk.session.appendAudio` añade audio de entrada PCM en base64 a las sesiones de retransmisión en tiempo real y transcripción propiedad del gateway.
    - `talk.session.startTurn`, `talk.session.endTurn` y `talk.session.cancelTurn` controlan el ciclo de vida de los turnos de las salas administradas y rechazan los turnos obsoletos antes de borrar el estado.
    - `talk.session.cancelOutput` detiene la salida de audio del asistente, principalmente para permitir interrupciones controladas por VAD en sesiones de retransmisión del gateway.
    - `talk.session.submitToolResult` completa una llamada a una herramienta del proveedor emitida por una sesión de retransmisión en tiempo real propiedad del gateway. La solicitud espera cualquier señal de finalización asíncrona expuesta por el puente del proveedor; los envíos fallidos mantienen activa la ejecución vinculada y no emiten un evento de resultado de herramienta correcto. Proporcione `options: { willContinue: true }` para la salida provisional de la herramienta o `options: { suppressResponse: true }` cuando el puente del proveedor anuncie compatibilidad con la supresión y el resultado no deba iniciar otra respuesta.
    - `talk.session.steer` envía el control por voz de la ejecución activa a una sesión de conversación respaldada por un agente y propiedad del gateway: `{ sessionId, text, mode? }`, donde `mode` es `status`, `steer`, `cancel` o `followup`; si se omite el modo, se clasifica a partir del texto hablado.
    - `talk.session.close` cierra una sesión de retransmisión, transcripción o sala administrada propiedad del gateway y emite eventos terminales de conversación.
    - `talk.mode` establece o difunde el estado actual del modo de conversación para los clientes de WebChat y la interfaz de control.
    - `talk.client.create` crea o reanuda una sesión de proveedor en tiempo real propiedad del cliente mediante `webrtc` o `provider-websocket`, mientras el gateway administra las credenciales, las instrucciones, la política de herramientas y el valor `voiceSessionId` devuelto. Los clientes proporcionan `sessionKey` y reutilizan `voiceSessionId` al sustituir el transporte del proveedor durante una llamada.
    - `talk.client.transcript` añade un elemento `{ role, text }` finalizado a la sesión normal del agente. El valor obligatorio `entryId` es idempotente dentro de `voiceSessionId`; los reintentos no duplican los mensajes de la transcripción.
    - `talk.client.close` cierra la sesión de voz lógica después de las escrituras pendientes en la transcripción. El cierre es idempotente y puede entregar un resumen de la llamada que solo contiene modificaciones al último canal de la sesión que no sea WebChat.
    - `talk.client.toolCall` permite que los transportes en tiempo real propiedad del cliente reenvíen las llamadas a herramientas del proveedor a la política del gateway. La primera herramienta compatible es `openclaw_agent_consult`; los clientes reciben un identificador de ejecución y esperan los eventos normales del ciclo de vida del chat antes de enviar el resultado de herramienta específico del proveedor. Las acciones de alto impacto vinculadas a la voz devuelven `VOICE_CONFIRMATION_REQUIRED:<id>` hasta que una intervención posterior y finalizada del usuario confirme explícitamente esa acción exacta y la siguiente consulta proporcione `confirmationId`.
    - `talk.client.steer` envía el control por voz de la ejecución activa para transportes en tiempo real propiedad del cliente. El gateway resuelve la ejecución integrada activa a partir de `sessionKey` y devuelve un resultado estructurado de aceptación o rechazo, en lugar de descartar silenciosamente las instrucciones.
    - `talk.event` es el canal único de eventos de conversación para adaptadores de tiempo real, transcripción, STT/TTS, salas administradas, telefonía y reuniones.
    - `talk.speak` sintetiza voz mediante el proveedor de voz de conversación activo.
    - `tts.status` devuelve el estado de activación de TTS, el proveedor activo, los proveedores alternativos y el estado de configuración del proveedor.
    - `tts.providers` devuelve el inventario visible de proveedores de TTS.
    - `tts.enable` y `tts.disable` alternan el estado de las preferencias de TTS.
    - `tts.setProvider` actualiza el proveedor de TTS preferido.
    - `tts.convert` ejecuta una conversión puntual de texto a voz.
    - `tts.speak` (`operator.write`) procesa un valor no vacío de `text` con la cadena configurada de proveedores generales de TTS y devuelve un clip completo en línea como `audioBase64`, además de `provider` y los metadatos opcionales `outputFormat`, `mimeType` y `fileExtension`. A diferencia de `tts.convert`, no devuelve una ruta local del Gateway; a diferencia de `talk.speak`, no requiere un proveedor de conversación. El texto que supera `tts.maxTextLength` devuelve `INVALID_REQUEST`; los errores de síntesis devuelven `UNAVAILABLE`.

  </Accordion>

  <Accordion title="Secretos, configuración, actualización y asistente">
    - `secrets.reload` vuelve a resolver las SecretRefs activas y publica atómicamente un estado de tiempo de ejecución que tiene en cuenta al propietario. Los fallos de propietarios aptos pueden publicarse como degradación en frío u obsoleta con `warningCount`; los fallos estrictos o sin asignar rechazan la recarga y conservan la instantánea activa.
    - `secrets.resolve` resuelve las asignaciones de secretos de destinos de comandos para un conjunto específico de comandos y destinos.
    - `config.get` devuelve la instantánea actual de la configuración en disco, el `hash` sin procesar del archivo raíz, el `configRevisionHash` resuelto y el `appliedConfigHash` opcional para la revisión resuelta aceptada por el tiempo de ejecución activo del Gateway.
    - `config.set` escribe una carga útil de configuración validada.
    - `config.patch` combina una actualización parcial de la configuración. El reemplazo destructivo de matrices requiere la ruta afectada en `replacePaths`; las matrices anidadas bajo entradas de matrices utilizan rutas `[]`, como `agents.entries.*.skills`.
    - `config.apply` valida y reemplaza la carga útil de configuración completa.
    - `config.schema` devuelve la carga útil del esquema de configuración en vivo utilizada por la interfaz de control y las herramientas de la CLI: esquema, `uiHints`, versión, metadatos de generación y metadatos de esquemas de plugins y canales cuando se pueden cargar. Incluye metadatos `title` / `description` procedentes de las mismas etiquetas y textos de ayuda que la interfaz, incluidas las ramas de composición de objetos anidados, comodines, elementos de matriz y `anyOf` / `oneOf` / `allOf` cuando existe documentación de campos coincidente.
    - `config.schema.lookup` devuelve una carga útil de consulta limitada a una ruta para una ruta de configuración: ruta normalizada, un nodo de esquema superficial, indicación coincidente y `hintPath`, `reloadKind` opcional y resúmenes de los elementos secundarios inmediatos para la exploración detallada en la interfaz o la CLI. `reloadKind` es uno de `restart`, `hot` o `none` (`src/config/schema.ts`) y refleja el planificador de recarga de la configuración del Gateway para la ruta solicitada. Los nodos del esquema de consulta conservan la documentación orientada al usuario y los campos de validación comunes (`title`, `description`, `type`, `enum`, `const`, `format`, `pattern`, límites numéricos, de cadenas, matrices y objetos, `additionalProperties`, `deprecated`, `readOnly`, `writeOnly`). Los resúmenes de elementos secundarios exponen `key`, el `path` normalizado, `type`, `required`, `hasChildren`, el `reloadKind` opcional, además de los `hint` / `hintPath` coincidentes.
    - `update.run` ejecuta el flujo de actualización del Gateway y programa un reinicio solo si la actualización se realizó correctamente; los llamadores con una sesión pueden incluir `continuationMessage` para que el arranque reanude un turno de seguimiento del agente mediante la cola de continuación del reinicio. Las actualizaciones del gestor de paquetes y las actualizaciones supervisadas de copias de trabajo de Git desde el plano de control utilizan una transferencia a un servicio administrado independiente en lugar de reemplazar el árbol de paquetes o modificar la copia de trabajo o la salida de compilación dentro del Gateway activo. Una transferencia iniciada devuelve `ok: true` con `result.reason: "managed-service-handoff-started"` y `handoff.status: "started"`. Un segundo `update.run` simultáneo gestionado por el mismo proceso del Gateway devuelve `ok: false` con `result.reason: "managed-service-handoff-already-running"` y `handoff.status: "already-running"`; su continuación no se acepta, por lo que el llamador puede volver a intentarlo cuando finalice la actualización activa. Los actualizadores independientes de la CLI y los procesos de reemplazo del Gateway quedan fuera de esta protección local del proceso. Las transferencias no disponibles o fallidas devuelven `ok: false` con `managed-service-handoff-unavailable` o `managed-service-handoff-failed`, además de `handoff.command` cuando se requiere una actualización manual mediante el shell. «No disponible» significa que OpenClaw carece de un límite de supervisor seguro o de una identidad de servicio persistente, como `OPENCLAW_SYSTEMD_UNIT` para systemd. Durante una transferencia iniciada, el marcador de reinicio puede informar brevemente de `stats.reason: "restart-health-pending"`; la continuación se retrasa hasta que la CLI verifica el Gateway reiniciado y escribe el marcador `ok` final.
    - `update.status` actualiza y devuelve el marcador de reinicio de actualización más reciente, incluida la versión en ejecución posterior al reinicio cuando está disponible.
    - `wizard.start`, `wizard.next`, `wizard.status` y `wizard.cancel` exponen el asistente de incorporación mediante RPC de WS.

  </Accordion>

  <Accordion title="Ayudantes del agente y del espacio de trabajo">
    - `agents.list` devuelve las entradas de agentes visibles para el Gateway, incluidos los metadatos efectivos del modelo y el tiempo de ejecución y el `kind` semántico opcional (`agent` o `system`). Los clientes anuncian la capacidad de negociación `agent-kind` para recibir la lista completa con tipos; los clientes que no la tienen conservan la lista heredada, segura para selectores y sin filas del sistema. Los clientes que reconocen el tipo excluyen las filas `system` de los selectores ordinarios, pero las conservan en las vistas de diagnóstico. Los Gateways v4 más antiguos pueden devolver filas sin `kind`.
    - `agents.create`, `agents.update` y `agents.delete` administran los registros de agentes y la vinculación del espacio de trabajo.
    - `agents.files.list`, `agents.files.get` y `agents.files.set` administran los archivos de arranque del espacio de trabajo expuestos para un agente.
    - `audit.activity.list` devuelve el registro de actividad versionado que contiene solo metadatos; `audit.list` sigue siendo el RPC de ejecuciones y herramientas seguro para la compatibilidad.
    - `agents.workspace.list` y `agents.workspace.get` (`operator.read`) exponen la exploración paginada y de solo lectura del directorio del espacio de trabajo de un agente para los clientes del dominio de operadores de confianza descrito en [Ámbitos de operador](/es/gateway/operator-scopes). Las solicitudes solo aceptan rutas relativas al espacio de trabajo; las lecturas permanecen limitadas a la raíz del espacio de trabajo cuya ruta real se ha resuelto (se rechazan los escapes mediante enlaces simbólicos y enlaces físicos), tienen un límite de tamaño y se restringen a texto UTF-8 y tipos de imagen comunes (base64). Las respuestas no exponen la ruta del espacio de trabajo del host. No hay operaciones de escritura en este espacio de nombres.
    - `tasks.list`, `tasks.get` y `tasks.cancel` exponen el registro de tareas del Gateway a los clientes del SDK y a los operadores. Consulte [RPC del registro de tareas](#task-ledger-rpcs) más adelante.
    - `artifacts.list`, `artifacts.get` y `artifacts.download` exponen resúmenes y descargas de artefactos derivados de transcripciones para un ámbito explícito `sessionKey`, `runId` o `taskId`. Las consultas de ejecuciones y tareas resuelven la sesión propietaria en el servidor y solo devuelven contenido multimedia de la transcripción con procedencia coincidente; las fuentes URL no seguras o locales devuelven descargas no compatibles en lugar de obtenerlas desde el servidor.
    - `environments.list` y `environments.status` conservan el descubrimiento de entornos locales del Gateway y de Node. Los trabajadores en la nube configurados y los registros persistentes dejados por perfiles anteriores añaden metadatos `worker` con `providerId`, `leaseId` opcional, `state`, `ageMs`, `idleMs` opcional y `attachedSessionIds`. Los estados del ciclo de vida de los trabajadores son `requested`, `provisioning`, `bootstrapping`, `ready`, `attached`, `idle`, `draining`, `destroying`, `destroyed`, `failed` y `orphaned`.
    - `environments.create` (`{ profileId, idempotencyKey }`) aprovisiona un trabajador desde un perfil configurado del proveedor del plugin; los reintentos con la misma clave reutilizan la operación persistente. `environments.destroy` (`{ environmentId }`) solicita el desmantelamiento idempotente de un entorno persistente de trabajador. Ambos requieren `operator.admin`, son escrituras del plano de control y devuelven la misma forma de resumen del entorno que utilizan las respuestas de estado.
    - `agent.identity.get` devuelve la identidad efectiva del asistente para un agente o una sesión.
    - `agent.wait` espera a que finalice una ejecución y devuelve la instantánea terminal cuando está disponible.

  </Accordion>

  <Accordion title="Control de sesiones">
    - `sessions.list` devuelve el índice de sesiones actual, incluidos los metadatos `agentRuntime` de cada fila cuando se configura un backend de entorno de ejecución de agente. Cuando está habilitada la asignación a trabajadores en la nube o existe un estado de recuperación duradero, las filas de sesión también incluyen un estado `placement` cerrado (`local`, `requested`, `provisioning`, `syncing`, `starting`, `active`, `draining`, `reconciling`, `reclaimed` o `failed`), además de campos de entorno, época del propietario, espacio de trabajo, paquete, cursor ACK o recuperación específicos del estado.
    - `sessions.subscribe` y `sessions.unsubscribe` activan o desactivan las suscripciones a eventos de cambios de sesión para el cliente WS actual.
    - `sessions.messages.subscribe` y `sessions.messages.unsubscribe` activan o desactivan las suscripciones a eventos de transcripción/mensajes para una sesión. Pase `includeApprovals: true` para recibir también eventos de ciclo de vida `session.approval` saneados para aprobaciones cuya audiencia persistida incluya esa sesión exacta y cuya vinculación de revisor autorice al cliente suscriptor. La respuesta de suscripción incluye entonces un conjunto pendiente acotado `approvalReplay`; es autoritativo cuando `truncated` es falso. La habilitación es específica de cada llamada de suscripción, no persistente: volver a suscribirse a la misma sesión sin `includeApprovals: true` elimina una suscripción de aprobación existente. Además de la autoridad normal de lectura de sesiones, esta habilitación requiere `operator.admin` o `operator.approvals` en un dispositivo emparejado.
    - `sessions.preview` devuelve vistas previas acotadas de transcripciones para claves de sesión específicas.
    - `sessions.describe` devuelve una fila de sesión del Gateway para una clave de sesión exacta.
    - `sessions.resolve` resuelve o canonicaliza un destino de sesión.
    - `sessions.create` crea una nueva entrada de sesión. Los valores opcionales `model` y `thinkingLevel` persisten atómicamente las anulaciones iniciales del modelo y del razonamiento. `worktree: true` aprovisiona un árbol de trabajo administrado; los valores opcionales `worktreeBaseRef`/`worktreeName` seleccionan la referencia base y el nombre de la rama, y `execNode` (`operator.admin`) vincula la ejecución de la sesión a un host de Node. El árbol de trabajo creado se reproduce en el resultado y se persiste en la fila de sesión (`worktree: { id, branch, repoRoot }`). Cuando se crea la entrada, pero se rechaza su `chat.send` inicial anidado, el resultado satisfactorio incluye `runStarted: false` y `runError`; los clientes pueden conservar el prompt y volver a intentarlo con la clave de sesión devuelta. Un llamador que pase `parentSessionKey` con `emitCommandHooks: true` también debe declarar la disposición del ciclo de vida de un elemento secundario distinto: `succeedsParent: true` finaliza el elemento primario con `session_end`, mientras que `false` mantiene activo el elemento primario y emite únicamente el `session_start` del secundario. Omitir `succeedsParent` conserva el comportamiento heredado de sustitución del elemento primario para los clientes existentes. La disposición requiere tanto la vinculación con el elemento primario como los hooks de comandos; una bifurcación no puede completar correctamente su elemento primario. El comportamiento de restablecimiento in situ de la sesión principal no cambia porque no se crea ningún elemento secundario distinto. Las filas nuevas se marcan con procedencia de creación de escritura única (`createdVia`, `createdActor`, `createdAt`) desde el punto de creación de confianza; adoptar una clave existente nunca vuelve a marcarla. Para los actores de perfiles humanos, `createdActor.label` se resuelve a partir del perfil de usuario actual cuando se proyecta la fila y nunca se almacena en la entrada de sesión, por lo que los cambios de nombre del perfil no producen divergencias. Las filas de sesión también contienen `parentSessionKey` (elemento primario de navegación, persistido), `controlOwnerSessionKey` (controlador del entorno de ejecución cuando está activo), `forkSource` (clave de origen exacta + generación de transcripción para bifurcaciones) y `previousSessionId` (generación de transcripción anterior con la misma clave).
    - `sessions.dispatch` (`operator.admin`) mueve una sesión local existente de OpenClaw con un árbol de trabajo administrado propiedad de la sesión a un perfil configurado de trabajador en la nube. Pase `{ key, profileId, agentId? }`. El método no está disponible cuando no se ha configurado ningún perfil de trabajador, cierra la admisión de turnos locales antes de agotar el trabajo activo y solo devuelve el control después de que la asignación alcance la propiedad del trabajador `active`. El envío es unidireccional; la recuperación del trabajador al entorno local no forma parte de este RPC.
    - `sessions.groups.list`, `sessions.groups.put`, `sessions.groups.rename` y `sessions.groups.delete` administran el catálogo de grupos de sesiones personalizados propiedad del Gateway (nombres + orden de visualización). La pertenencia permanece en el campo `category` de cada sesión; las operaciones de cambio de nombre y eliminación actualizan las sesiones miembro en el servidor.
    - `sessions.send` envía un mensaje a una sesión existente.
    - `sessions.steer` es la variante de interrupción y redirección para una sesión activa.
    - `sessions.abort` cancela el trabajo activo de una sesión. Pase `key` junto con el valor opcional `runId`, o únicamente `runId` para ejecuciones activas que el Gateway pueda resolver como una sesión. Proporcionar `runId` limita la cancelación a esa ejecución. Establezca `clearQueued: true` en una solicitud no global basada únicamente en una clave para descartar también las colas de seguimiento y de carril propiedad de esa sesión. Los llamadores existentes que omitan `clearQueued` conservan esas colas. La clave literal `global` mantiene las reglas existentes de propiedad `chat.abort` calificadas por agente y no realiza una limpieza no global de las colas de seguimiento ni de carril.
    - `sessions.patch` actualiza los metadatos/anulaciones de la sesión e informa del modelo canónico resuelto junto con el `agentRuntime` efectivo. El linaje de generación (`spawnedBy`, `spawnedWorkspaceDir`, `spawnedCwd`, `spawnDepth`, `subagentRole`, `subagentControlScope`) ya no puede modificarse públicamente; estos datos se escriben una sola vez mediante rutas de creación de confianza, y se rechazan las solicitudes que aún los envíen.
    - `sessions.reset`, `sessions.delete` y `sessions.compact` realizan el mantenimiento de sesiones.
    - `sessions.get` devuelve la fila de sesión almacenada completa.
    - La ejecución del chat sigue utilizando `chat.history`, `chat.send`, `chat.abort` y `chat.inject`. `chat.history` se normaliza para su visualización en clientes de interfaz de usuario: las etiquetas de directivas insertadas se eliminan del texto visible; se eliminan las cargas XML de llamadas a herramientas en texto sin formato (`<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` y los bloques truncados de llamadas a herramientas) y los tokens de control del modelo ASCII/de ancho completo filtrados; se omiten las filas del asistente que solo contienen tokens silenciosos (`NO_REPLY` / `no_reply` exactos), y las filas demasiado grandes pueden sustituirse por marcadores de posición.
    - `chat.message.get` es el lector completo, acotado y aditivo de mensajes para una única entrada visible de la transcripción. Pase `sessionKey`, el valor opcional `agentId` cuando la selección de sesión esté limitada al agente y un `messageId` de transcripción expuesto anteriormente mediante `chat.history`; el Gateway devuelve la misma proyección normalizada para visualización sin el límite ligero de truncamiento del historial cuando la entrada almacenada sigue disponible y no es demasiado grande.
    - `chat.toolTitles` devuelve títulos breves de propósito para las llamadas a herramientas representadas en la interfaz de control (por lotes, con un máximo de 24 elementos y entradas acotadas). La función se habilita opcionalmente mediante `gateway.controlUi.toolTitles` (desactivada de forma predeterminada); los Gateway deshabilitados responden `{ titles: {}, disabled: true }` sin ninguna llamada al modelo para que los clientes dejen de solicitarla. Cuando está habilitada, los títulos utilizan el enrutamiento estándar de modelos auxiliares: un `utilityModel` configurado explícitamente (una decisión del operador que, al igual que todas las tareas auxiliares, puede enviar contenido acotado de la tarea al proveedor elegido) o, en su defecto, el modelo pequeño predeterminado declarado por el proveedor de la sesión, para que no aparezca implícitamente ningún destino de salida nuevo; un `utilityModel` vacío los deshabilita por completo. Los títulos nunca recurren al modelo principal. Los resultados se almacenan en caché en la base de datos de estado de cada agente mediante una clave compuesta por el nombre de la herramienta + la entrada, por lo que las visualizaciones repetidas nunca vuelven a facturar las mismas llamadas.
    - `chat.send` acepta un `fastMode: "auto"` de un solo turno para utilizar el modo rápido en las llamadas al modelo iniciadas antes del límite automático y, después, iniciar llamadas posteriores de reintento, alternativa, resultado de herramienta o continuación sin el modo rápido. El límite predeterminado es de 60 segundos (`DEFAULT_FAST_MODE_AUTO_ON_SECONDS`) y puede configurarse por modelo con `agents.defaults.models["<provider>/<model>"].params.fastAutoOnSeconds`. Un llamador `chat.send` puede pasar un `fastAutoOnSeconds` de un solo turno para anular el límite de esa solicitud. Pase `queueMode` (`steer`, `followup`, `collect` o `interrupt`) para anular el modo de cola almacenado únicamente para esta solicitud; las acciones explícitas de redirección de la interfaz de control utilizan `queueMode: "steer"`. Los clientes interactivos pueden pasar `expectedLeafEntryId` con la hoja activa de la rama de transcripción que muestran, o `null` para indicar de forma autoritativa una transcripción vacía; el Gateway rechaza el envío con `details.reason: "active-leaf-changed"` si otro cliente cambió de rama primero.

  </Accordion>

  <Accordion title="Emparejamiento de dispositivos y tokens de dispositivo">
    - `device.pair.list` devuelve los dispositivos emparejados pendientes y aprobados.
    - `device.pair.setupCode` crea un código de configuración móvil y, de forma predeterminada, una URL de datos de un código QR PNG. Requiere `operator.admin` y se omite intencionadamente del descubrimiento anunciado. El resultado incluye `setupCode`, el valor opcional `qrDataUrl`, `gatewayUrl`, la etiqueta no secreta `auth` y `urlSource`.
    - `device.pair.approve`, `device.pair.reject` y `device.pair.remove` administran los registros de emparejamiento de dispositivos.
    - `device.pair.rename` asigna una etiqueta del operador (`{ deviceId, label }`) que tiene prioridad sobre el nombre para mostrar informado por el cliente y se conserva tras reparar o volver a aprobar el dispositivo.
    - `device.token.rotate` rota un token de dispositivo emparejado dentro de los límites de su rol aprobado y del ámbito del llamador.
    - `device.token.revoke` revoca un token de dispositivo emparejado dentro de los límites de su rol aprobado y del ámbito del llamador.

    El código de configuración incorpora una credencial de arranque de corta duración. Los clientes no deben
    registrarla ni conservarla después del flujo de emparejamiento.

  </Accordion>

  <Accordion title="Emparejamiento de Node, invocación y trabajo pendiente">
    - `node.pair.list`, `node.pair.approve`, `node.pair.reject` y `node.pair.remove` abarcan las aprobaciones de capacidades de Node. `node.pair.request` y `node.pair.verify` se eliminaron en 2026.7 junto con el almacén independiente de emparejamiento de Node; el Gateway crea las solicitudes pendientes durante las conexiones de Node.
    - `node.list` y `node.describe` devuelven el estado conocido/conectado de Node.
    - `node.rename` actualiza la etiqueta de un Node emparejado.
    - `node.invoke` reenvía un comando a un Node conectado.
    - `node.invoke.result` devuelve el resultado de una solicitud de invocación.
    - `mcp.tools.call.v1` es el comando del host de Node sin interfaz gráfica para llamar a una herramienta MCP local de Node configurada. Se transporta mediante `node.invoke`, requiere que el Node declare el comando y sigue sujeto a la aprobación de emparejamiento y a `gateway.nodes.commands.deny`.
    - `node.event` transporta los eventos originados en Node de vuelta al Gateway.
    - `node.pluginTools.update` es la única ruta de publicación para reemplazar los descriptores de herramientas de Plugin/MCP visibles para el agente del Node conectado; los parámetros `connect` no los transportan.
    - `node.pending.pull` y `node.pending.ack` son las API de cola del Node conectado.
    - `node.pending.enqueue` y `node.pending.drain` administran el trabajo pendiente duradero de los Node sin conexión/desconectados.

  </Accordion>

  <Accordion title="Familias de aprobaciones">
    - `approval.history` devuelve, comenzando por las más recientes, las aprobaciones terminales conservadas durante 30 días para solicitudes de ejecución, plugins y agentes del sistema (ámbito `operator.approvals`). Admite paginación mediante cursor y un filtro opcional por tipo; las aprobaciones pendientes no son filas del historial.
    - `approval.get` y `approval.resolve` son los métodos duraderos de aprobación independientes del tipo (ámbito `operator.approvals`). `approval.get` devuelve una proyección depurada, pendiente o terminal conservada, con un `urlPath` estable; `approval.resolve` acepta el identificador canónico de aprobación, un `kind` explícito y una decisión, aplica una resolución en la que prevalece la primera respuesta y siempre devuelve el resultado canónico registrado.
    - `exec.approval.request`, `exec.approval.get`, `exec.approval.list` y `exec.approval.resolve` abarcan las solicitudes de aprobación de ejecución de un solo uso, además de la consulta y repetición de aprobaciones pendientes. Son adaptadores de límite de protocolo sobre el mismo registro duradero de aprobaciones.
    - `exec.approval.waitDecision` espera una aprobación de ejecución pendiente y devuelve la decisión final (o `null` al agotarse el tiempo de espera).
    - `exec.approvals.get` y `exec.approvals.set` administran instantáneas de la política de aprobación de ejecución del Gateway.
    - `exec.approvals.node.get` y `exec.approvals.node.set` administran la política local del Node para la aprobación de ejecución mediante comandos de retransmisión del Node.
    - `plugin.approval.request`, `plugin.approval.list`, `plugin.approval.waitDecision` y `plugin.approval.resolve` abarcan los flujos de aprobación definidos por plugins.

  </Accordion>

  <Accordion title="Comandos de la interfaz de control">
    - `ui.command` permite que un emisor `operator.write` envíe comandos tipados de disposición y navegación a los clientes conectados de la interfaz de control que anuncien la capacidad `ui-commands`.
    - Los comandos abarcan la división, el cierre y el enfoque de paneles; la visibilidad de la barra lateral; la visibilidad y el acoplamiento de los paneles de terminal y navegador; y la navegación entre sesiones.
    - El protocolo v1 distribuye intencionadamente los comandos a todas las interfaces de control conectadas que sean compatibles. Si no hay ninguna conectada, la solicitud falla con `UNAVAILABLE` en lugar de simular que la disposición cambió.

  </Accordion>

  <Accordion title="Automatización, Skills y herramientas">
    - Automatización: `wake` programa la inserción inmediata o en el próximo Heartbeat de un texto de activación; `cron.get`, `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`, `cron.run` y `cron.runs` administran el trabajo programado.
    - `cron.run` sigue siendo un RPC basado en encolado para ejecuciones manuales. Los clientes que necesiten semántica de finalización deben leer el `runId` devuelto y consultar periódicamente `cron.runs`.
    - `cron.runs` acepta un filtro `runId` opcional y no vacío para que los clientes puedan seguir una ejecución manual encolada sin competir con otras entradas del historial correspondientes al mismo trabajo.
    - Skills y herramientas: `commands.list`, `skills.*`, `tools.catalog`, `tools.effective`, `tools.invoke`. Consulte [Métodos auxiliares del operador](#operator-helper-methods) más adelante.

  </Accordion>
</AccordionGroup>

### Familias de eventos comunes

- `chat`: actualizaciones del chat de la interfaz, como `chat.inject` y otros eventos de chat
  exclusivos de la transcripción. En el protocolo v4, las cargas útiles incrementales contienen `deltaText`; `message` sigue siendo
  la instantánea acumulativa del asistente. Los reemplazos que no sean prefijos establecen
  `replace=true` y utilizan `deltaText` como texto de reemplazo.
- `session.message`, `session.operation`, `session.tool`: actualizaciones de la transcripción, de las operaciones de sesión
  en curso y del flujo de eventos para una sesión suscrita.
- `session.approval`: estado depurado de las aprobaciones pendientes y terminales para un
  suscriptor de sesión exacta que haya aceptado explícitamente recibirlo. Las aprobaciones secundarias utilizan la
  audiencia persistente del antecesor; los eventos nunca modifican las transcripciones ni activan agentes.
- `sessions.changed`: el índice o los metadatos de la sesión cambiaron.
- `presence`: actualizaciones de la instantánea de presencia del sistema.
- `tick`: evento periódico de mantenimiento de conexión y actividad.
- `health`: actualización de la instantánea del estado del Gateway.
- `heartbeat`: actualización del flujo de eventos de Heartbeat.
- `cron`: evento de cambio de una ejecución o un trabajo de Cron.
- `shutdown`: notificación de apagado del Gateway.
- `node.pair.requested` / `node.pair.resolved`: ciclo de vida del emparejamiento de Nodes.
- `node.invoke.request`: difusión de una solicitud de invocación de Node.
- `device.pair.requested` / `device.pair.resolved`: ciclo de vida de los dispositivos emparejados.
- `voicewake.changed`: cambió la configuración del activador por palabra de activación.
- `config.changed`: se ha persistido una escritura de configuración (la carga útil contiene la ruta de configuración,
  el hash de la nueva instantánea y una marca de tiempo, pero nunca el contenido de la configuración). Limitado
  a la lectura por operadores; los clientes actualizan mediante `config.get`.
- `exec.approval.requested` / `exec.approval.resolved`: ciclo de vida
  de las aprobaciones de ejecución.
- `plugin.approval.requested` / `plugin.approval.resolved`: ciclo de vida
  de las aprobaciones de plugins.

### Métodos auxiliares de Node

Los Nodes pueden llamar a `skills.bins` para obtener la lista actual de ejecutables de Skills
para las comprobaciones de autorización automática.

## RPC del libro mayor de auditoría

`audit.activity.list` proporciona a los clientes operadores una vista estable, comenzando por los más recientes, de los metadatos del ciclo de vida
de las ejecuciones de agentes, las acciones de herramientas y los mensajes que requieren aceptación explícita. Requiere
`operator.read`. Las consultas excluyen los registros con más de 30 días de antigüedad y el libro mayor
SQLite compartido está limitado a 100,000 registros. Las filas caducadas se eliminan durante
el inicio del Gateway, el mantenimiento por hora y las escrituras posteriores. Consulte
[Historial de auditoría](/es/gateway/audit) para conocer el modelo de datos y la semántica de privacidad.

- Parámetros: `agentId`, `sessionKey` o `runId` exactos y opcionales; `kind` opcional
  (`"agent_run"`, `"tool_action"` o `"message"`); `status` opcional
  (`"started"`, `"succeeded"`, `"failed"`, `"cancelled"`, `"timed_out"`,
  `"blocked"` o `"unknown"`); `direction` de mensaje opcional (`"inbound"` o
  `"outbound"`) y `channel` exacto; límites inclusivos opcionales `after` / `before`
  en milisegundos Unix; `limit` opcional de `1` a `500`; y una cadena
  `cursor` opcional de la página anterior.
- Resultado: `{ "events": AuditActivityEventV1[], "nextCursor"?: string }`.

La unión de resultados V1 con nombre contiene esquemas separados para ejecuciones de agentes, acciones de herramientas, mensajes entrantes
y mensajes salientes. El discriminador `eventType` es, respectivamente,
`agent_run`, `tool_action`, `inbound_message` o `outbound_message`; `kind` y
el `direction` de mensaje siguen disponibles para el filtrado y la visualización. Cada evento tiene
un `schemaVersion: 1` entero. Las referencias de identidad de mensajes utilizan el formato
`hmac-sha256:v1:<32 hex key id>:<64 hex digest>` exacto; el identificador del actor que envía por un canal
utiliza el mismo formato.

Todas las variantes requieren `eventType`, `schemaVersion`, `eventId`, `sequence`,
`sourceSequence`, `occurredAt`, `kind`, `action`, `status`, `actor` y
`redaction`. Los campos de las variantes son:

| `eventType`        | Campos obligatorios                                               | Campos opcionales                                                                                                                |
| ------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `agent_run`        | `agentId`, `runId`; `kind: "agent_run"`                           | `sessionKey`, `sessionId`, `errorCode`                                                                                          |
| `tool_action`      | `agentId`, `runId`; `kind: "tool_action"`                         | `sessionKey`, `sessionId`, `toolCallId`, `toolName`, `errorCode`                                                                |
| `inbound_message`  | `direction: "inbound"`, `channel`, `conversationKind`, `outcome`  | `agentId`, `runId`, `durationMs`, `resultCount`, referencias de identidad, `reasonCode`, `errorCode`                                 |
| `outbound_message` | `direction: "outbound"`, `channel`, `conversationKind`, `outcome` | `agentId`, `runId`, `durationMs`, `resultCount`, referencias de identidad, `reasonCode`, `deliveryKind`, `failureStage`, `errorCode` |

Las enumeraciones cerradas de mensajes son:

- `conversationKind`: `direct`, `group`, `channel` o `unknown`.
- `outcome` entrante: `completed`, `skipped` o `failed`; `reasonCode` opcional:
  `duplicate`, `reply_operation_active`,
  `reply_operation_aborted`, `fast_abort`, `plugin_bound_handled`,
  `plugin_bound_unavailable`, `plugin_bound_declined`, `plugin_bound_error`,
  `before_dispatch_handled`, `acp_dispatch_completed`, `acp_dispatch_failed`,
  `acp_dispatch_empty` o `acp_dispatch_aborted`.
- `outcome` saliente: `sent`, `suppressed`, `failed` o `unknown`; `reasonCode` opcional:
  `cancelled_by_message_sending_hook`,
  `cancelled_by_reply_payload_sending_hook`,
  `empty_after_message_sending_hook`, `empty_after_reply_payload_sending_hook`
  o `no_visible_payload`. Un adaptador que no devuelve ninguna identidad de plataforma es
  `unknown`, porque no se puede descartar el efecto secundario externo.
- `deliveryKind`: `text`, `media` o `other`; `failureStage`:
  `platform_send`, `queue` o `unknown`.

Los campos terminales están correlacionados, no son opcionales de forma independiente:

| Variante         | Correspondencia terminal                                                                                                                                           |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Ejecución de agente | `started` no tiene `errorCode`; cada estado finalizado que no sea correcto requiere su código `run_*` correspondiente.                                                                 |
| Acción de herramienta | `started` y el estado correcto no tienen `errorCode`; cada otro estado finalizado requiere su código `tool_*` correspondiente.                                                       |
| Mensaje entrante | correcto = `completed`; bloqueado = `skipped`; fallido = `failed` más `message_processing_failed`. `reasonCode`, cuando está presente, debe pertenecer a esa familia terminal. |
| Mensaje saliente | correcto = `sent`; bloqueado = `suppressed` más `reasonCode`; fallido = `failed` más `errorCode` y `failureStage`; desconocido = `unknown` más `failureStage`.      |

Cada evento de actividad incluye un identificador de evento estable, una secuencia monotónica del libro mayor,
una secuencia del evento de origen, una marca de tiempo, un actor, una acción, un estado, un
`schemaVersion: 1` entero y `redaction: "metadata_only"`. Los registros de ejecuciones y herramientas
requieren la procedencia del agente y de la ejecución, y pueden incluir la procedencia de la sesión. Los registros de
mensajes pueden incluir identificadores de agentes y ejecuciones, pero intencionadamente nunca incluyen
`sessionKey` ni `sessionId`; por tanto, el filtro de consulta `sessionKey` se aplica
solo a las filas de ejecuciones y herramientas. Los eventos de herramientas pueden incluir el identificador de llamada y el nombre de la herramienta.

Los registros de mensajes usan `message.inbound.processed` o
`message.outbound.finished` y añaden la dirección, el canal, el tipo de conversación,
el resultado normalizado y, opcionalmente, el tipo de entrega, la etapa del fallo, la duración,
el recuento de resultados, el código de motivo y seudónimos con clave local de la instalación
para la cuenta, la conversación, el mensaje y el destino. Estos seudónimos facilitan
la correlación, pero no constituyen anonimización: la base de datos de estado contiene su clave,
mientras que las exportaciones de RPC y CLI no. El registro no almacena prompts, cuerpos de
mensajes, argumentos de herramientas, resultados de herramientas, salida de comandos ni texto de error sin procesar.
Los valores `sessionKey` de ejecución/herramienta siguen siendo metadatos de correlación sin procesar y pueden incluir
identificadores de cuentas o pares de la plataforma; los registros de mensajes omiten las claves de sesión.

Para las filas entrantes, `durationMs` mide el despacho del núcleo hasta su finalización y
`resultCount` cuenta las cargas útiles finalizadas en cola de herramientas, bloques y respuestas. Para
las filas salientes, `durationMs` abarca la propiedad de la entrega hasta la confirmación,
la cola de mensajes fallidos o la conciliación (incluido el tiempo de espera en cola), y `resultCount`
cuenta los envíos físicos identificados a la plataforma. `deliveryKind`, cuando está presente,
describe la carga útil efectiva después de los hooks y la renderización; las filas suprimidas o
con ambigüedad por fallos omiten este valor.

La cobertura actual de mensajes incluye los mensajes entrantes aceptados que llegan al
despacho del núcleo, incluidos los resultados de duplicación/finalización del núcleo. La cobertura saliente escribe
una fila final por cada carga útil de respuesta lógica original que llega a la entrega
duradera compartida; la fragmentación y la distribución en abanico del adaptador se agregan en `resultCount`. Los envíos
reintentables o ambiguos en cola solo se registran después de la confirmación, la cola de
mensajes fallidos o la conciliación. Las rutas locales de los plugins y de envío directo que omiten esos
límites compartidos aún no están cubiertas. La cola de trabajadores acotada funciona según el mejor esfuerzo
y puede descartar registros en caso de fallo o saturación, por lo que esta superficie no es un
archivo de cumplimiento sin pérdidas.

El registro está activado de forma predeterminada y se controla mediante
[`audit.enabled`](/es/gateway/configuration-reference#audit). El registro de mensajes se
controla por separado mediante `audit.messages` y su valor predeterminado es `"off"`. Cuando
el registro está desactivado, `audit.activity.list` sigue sirviendo los registros escritos
anteriormente hasta que caduquen.

Los esquemas publicados de solicitud y resultado de `audit.list`, y de `AuditEvent`,
permanecen sin cambios y devuelven únicamente registros de ejecuciones de agentes y acciones de herramientas. Los nuevos clientes
de operador deben llamar a `audit.activity.list` cuando el Gateway lo anuncie. Los Gateways más antiguos
pueden informar de `unknown method: audit.activity.list` o, dado que
la autorización precedía a la búsqueda del método en las versiones publicadas, de `missing scope:
operator.admin` ante una solicitud con ámbito de lectura. Se debe interpretar este último como ausencia del método
solo cuando el método no se haya anunciado. A continuación, un cliente puede volver a intentar `audit.list`
únicamente cuando sus filtros no requieran compatibilidad con el tipo de mensaje, la dirección ni el
canal.

Use [`openclaw audit`](/es/cli/audit) para consultas de texto y exportaciones JSON acotadas.

## RPC del registro de tareas

Los clientes de operador inspeccionan y cancelan los registros de tareas en segundo plano del Gateway mediante
los RPC del registro de tareas (`packages/gateway-protocol/src/schema/tasks.ts`). Estos
devuelven resúmenes depurados de las tareas, no el estado del entorno de ejecución sin procesar.

- `tasks.list` requiere `operator.read`.
  - Parámetros: `status` opcional (`"queued"`, `"running"`, `"completed"`,
    `"failed"`, `"cancelled"` o `"timed_out"`) o una matriz de esos estados,
    `agentId` opcional, `sessionKey` opcional, `limit` opcional de `1` a
    `500`, y la cadena opcional `cursor`.
  - Resultado: `{ "tasks": TaskSummary[], "nextCursor"?: string }`.
- `tasks.get` requiere `operator.read`.
  - Parámetros: `{ "taskId": string }`.
  - Resultado: `{ "task": TaskSummary }`.
  - Los identificadores de tareas inexistentes devuelven el formato de error de elemento no encontrado del Gateway.
- `tasks.cancel` requiere `operator.write`.
  - Parámetros: `{ "taskId": string, "reason"?: string }`.
  - Resultado: `{ "found": boolean, "cancelled": boolean, "reason"?: string, "task"?: TaskSummary }`.
  - `found` indica si el registro contenía una tarea coincidente. `cancelled`
    indica si el entorno de ejecución aceptó o registró la cancelación.

`TaskSummary` incluye `id`, `status` y metadatos opcionales: `kind`,
`runtime`, `title`, `agentId`, `sessionKey`, `childSessionKey`, `ownerKey`,
`runId`, `taskId`, `flowId`, `parentTaskId`, `sourceId`, marcas de tiempo, progreso,
resumen final y texto de error depurado. `agentId` identifica al agente
que ejecuta la tarea; `sessionKey` y `ownerKey` conservan el contexto del solicitante y de control.

## Métodos auxiliares para operadores

- `commands.list` (`operator.read`) obtiene el inventario de comandos del entorno de ejecución para
  un agente.
  - `agentId` es opcional; omítalo para leer el espacio de trabajo del agente predeterminado.
  - `scope` controla a qué superficie apunta el `name` principal: `text` devuelve
    el token del comando de texto principal sin el `/` inicial; `native` y la
    ruta predeterminada `both` devuelven nombres nativos que tienen en cuenta al proveedor cuando están disponibles.
  - `textAliases` contiene alias exactos con barra, como `/model` y `/m`.
  - `nativeName` contiene el nombre del comando nativo que tiene en cuenta al proveedor cuando
    existe.
  - `provider` es opcional y solo afecta a la nomenclatura nativa y a la disponibilidad de comandos
    nativos de plugins.
  - `includeArgs=false` omite de la respuesta los metadatos serializados de los argumentos.
- `tools.catalog` (`operator.read`) obtiene el catálogo de herramientas del entorno de ejecución para un
  agente. La respuesta incluye herramientas agrupadas y metadatos de procedencia:
  - `source`: `core` o `plugin`
  - `pluginId`: propietario del plugin cuando `source="plugin"`
  - `optional`: indica si una herramienta del plugin es opcional
- `tools.effective` (`operator.read`) obtiene el inventario efectivo de herramientas del
  entorno de ejecución para una sesión.
  - `sessionKey` es obligatorio.
  - El Gateway deriva el contexto de confianza del entorno de ejecución a partir de la sesión en el servidor
    en lugar de aceptar un contexto de autenticación o entrega proporcionado por el llamador.
  - La respuesta es una proyección derivada por el servidor y limitada a la sesión del inventario
    activo, incluidas las herramientas del núcleo, de plugins, de canales y de servidores MCP
    ya descubiertas.
  - `tools.effective` es de solo lectura para MCP: puede proyectar un catálogo MCP
    de una sesión activa mediante la política final de herramientas, pero no crea entornos de ejecución MCP,
    conecta transportes ni emite `tools/list`. Si no existe ningún catálogo activo
    coincidente, la respuesta puede incluir un aviso como `mcp-not-yet-connected`,
    `mcp-not-yet-listed` o `mcp-stale-catalog`.
  - Las entradas de herramientas efectivas usan `source="core"`, `source="plugin"`,
    `source="channel"` o `source="mcp"`.
- `tools.invoke` (`operator.write`) invoca una herramienta disponible mediante la
  misma ruta de políticas del Gateway que `/tools/invoke`.
  - `name` es obligatorio. `args`, `sessionKey`, `agentId`, `confirm` y
    `idempotencyKey` son opcionales.
  - Si están presentes tanto `sessionKey` como `agentId`, el agente de sesión resuelto
    debe coincidir con `agentId`.
  - Los envoltorios del núcleo exclusivos del propietario, como `cron`, `gateway` y `nodes`, requieren
    identidad de propietario/administrador (`operator.admin`), aunque `tools.invoke`
    sea `operator.write`.
  - La respuesta es un contenedor orientado al SDK con `ok`, `toolName`, el campo opcional
    `output` y campos `error` tipados. Los rechazos de aprobación o de políticas devuelven
    `ok:false` en la carga útil en lugar de omitir el pipeline de políticas de herramientas
    del Gateway.
- `skills.status` (`operator.read`) obtiene el inventario visible de Skills para un
  agente.
  - `agentId` es opcional; omítalo para leer el espacio de trabajo del agente predeterminado.
  - La respuesta incluye la idoneidad, los requisitos que faltan, las comprobaciones de configuración
    y las opciones de instalación depuradas sin exponer los valores de secretos sin procesar.
- `skills.search` y `skills.detail` (`operator.read`) devuelven metadatos de
  descubrimiento de ClawHub.
- `skills.upload.begin`, `skills.upload.chunk` y `skills.upload.commit`
  (`operator.admin`) preparan un archivo privado de Skills antes de instalarlo. Esta
  es una ruta de carga administrativa independiente para clientes de confianza, no el flujo normal de
  instalación de Skills de ClawHub, y está desactivada de forma predeterminada salvo que
  `skills.install.allowUploadedArchives` esté habilitado.
  - `skills.upload.begin({ kind: "skill-archive", slug, sizeBytes, sha256?, force?, idempotencyKey? })`
    crea una carga vinculada a ese slug y valor de forzado.
  - `skills.upload.chunk({ uploadId, offset, dataBase64 })` añade bytes en
    el desplazamiento decodificado exacto.
  - `skills.upload.commit({ uploadId, sha256? })` verifica el tamaño final y
    SHA-256. La confirmación solo finaliza la carga; no instala la Skill.
  - Los archivos de Skills cargados son archivos zip que contienen una raíz `SKILL.md`. El
    nombre del directorio interno del archivo nunca selecciona el destino de instalación.
- `skills.install` (`operator.admin`) tiene tres modos:
  - Modo ClawHub: `{ source: "clawhub", slug, version?, force? }` instala una
    carpeta de Skills en el directorio `skills/` del espacio de trabajo del agente predeterminado.
  - Modo de carga: `{ source: "upload", uploadId, slug, force?, sha256?, timeoutMs? }`
    instala una carga confirmada en el directorio
    `skills/<slug>` del espacio de trabajo del agente predeterminado. El slug y el valor de forzado deben coincidir con la
    solicitud `skills.upload.begin` original. Se rechaza salvo que
    `skills.install.allowUploadedArchives` esté habilitado; el ajuste no
    afecta a las instalaciones de ClawHub.
  - Modo de instalador del Gateway: `{ name, installId, timeoutMs? }` ejecuta una acción
    `metadata.openclaw.install` declarada en el host del Gateway. Los clientes más antiguos aún pueden
    enviar `dangerouslyForceUnsafeInstall`; este campo está obsoleto,
    se acepta únicamente por compatibilidad con el protocolo y se ignora. Use
    `security.installPolicy` para las decisiones de instalación propiedad del operador.
- `skills.update` (`operator.admin`) tiene dos modos:
  - El modo ClawHub actualiza un slug registrado o todas las instalaciones registradas de ClawHub en
    el espacio de trabajo del agente predeterminado.
  - El modo de configuración modifica los valores de `skills.entries.<skillKey>`, como `enabled`,
    `apiKey` y `env`.

### Vistas de `models.list`

`models.list` acepta un parámetro `view` opcional
(`src/agents/model-catalog-visibility.ts`):

- Omitido o `"default"`: si `agents.defaults.modelPolicy.allow` está configurado, la
  respuesta es el catálogo permitido, incluidos los modelos descubiertos dinámicamente
  para las entradas `provider/*`. De lo contrario, la respuesta es el catálogo completo del
  Gateway.
- `"configured"`: comportamiento adaptado al selector. Si `agents.defaults.modelPolicy.allow` está
  configurado, sigue teniendo prioridad, incluido el descubrimiento limitado al proveedor para
  las entradas `provider/*`. Sin una lista de permitidos, la respuesta usa entradas
  `models.providers.<provider>.models` explícitas y recurre al catálogo
  completo únicamente cuando no existe ninguna fila de modelos configurada.
- `"provider-config"`: inventario `models.providers.*.models` definido por la fuente,
  independiente de las listas de permitidos del selector. Las filas incluyen capacidades públicas de los modelos y
  disponibilidad en función de las rutas, pero omiten los endpoints de los proveedores, el material de autenticación y
  la configuración de las solicitudes del entorno de ejecución.
- `"all"`: catálogo completo del Gateway, omitiendo `agents.defaults.modelPolicy.allow`. Úselo para
  interfaces de usuario de diagnóstico/descubrimiento, no para selectores de modelos normales.

## Aprobaciones de ejecución

- Cuando una solicitud de ejecución necesita aprobación, el Gateway difunde
  `exec.approval.requested`.
- Los clientes del operador la resuelven llamando a `exec.approval.resolve` (requiere
  `operator.approvals`).
- Para `host=node`, `exec.approval.request` debe incluir `systemRunPlan`
  (metadatos canónicos de `argv`/`cwd`/`rawCommand`/sesión). Se rechazan las solicitudes que no incluyan
  `systemRunPlan`.
- Tras la aprobación, las llamadas reenviadas a `node.invoke system.run` reutilizan ese
  `systemRunPlan` canónico como contexto autoritativo del comando, el directorio de trabajo y la sesión.
- Si un llamador modifica `command`, `rawCommand`, `cwd`, `agentId` o
  `sessionKey` entre la preparación y el reenvío final aprobado de `system.run`,
  el Gateway rechaza la ejecución en lugar de confiar en la carga útil modificada.

## Alternativa de entrega del agente

- Las solicitudes `agent` pueden incluir `deliver=true` para solicitar la entrega saliente.
- `bestEffortDeliver=false` (el valor predeterminado) mantiene un comportamiento estricto: los destinos de entrega
  no resueltos o solo internos devuelven `INVALID_REQUEST`.
- `bestEffortDeliver=true` permite recurrir a la ejecución solo en la sesión cuando no se puede
  resolver ninguna ruta externa de entrega (por ejemplo, sesiones internas o de chat web,
  o configuraciones multicanal ambiguas).
- Los resultados finales de `agent` pueden incluir `result.deliveryStatus` cuando se solicitó
  la entrega, utilizando los mismos estados `sent`, `suppressed`, `partial_failed` y
  `failed` documentados para
  [`openclaw agent --json --deliver`](/es/cli/agent#json-delivery-status).

## Control de versiones

- `PROTOCOL_VERSION`, `MIN_CLIENT_PROTOCOL_VERSION`,
  `MIN_NODE_PROTOCOL_VERSION` y `MIN_PROBE_PROTOCOL_VERSION` se encuentran en
  `packages/gateway-protocol/src/version.ts`.
- Los clientes envían `minProtocol` + `maxProtocol`. Los clientes del operador y de la interfaz de usuario deben
  incluir el protocolo actual en ese intervalo; los clientes y servidores actuales ejecutan
  el protocolo v4.
- Los clientes autenticados que tengan tanto `role: "node"` como `client.mode: "node"`
  pueden utilizar el protocolo de Node N-1 (actualmente v3). Las sondas ligeras de reinicio utilizan
  el mismo intervalo N-1. La autenticación de dispositivos, el emparejamiento, los ámbitos, la política de comandos y las
  aprobaciones de ejecución no cambian debido a esta ventana de compatibilidad. Las capacidades y los comandos de Node
  pertenecientes a plugins no están disponibles hasta que el Node se actualice al protocolo
  actual, porque sus superficies alojadas no forman parte del contrato N-1.
- Los esquemas y modelos se generan a partir de definiciones de TypeBox:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

### Constantes del cliente

La implementación del cliente de referencia se encuentra en `packages/gateway-client/src/`
(OpenClaw la encapsula mediante la delgada fachada `src/gateway/client.ts`). Estos
valores predeterminados son estables en el protocolo v4 y constituyen la referencia esperada para
los clientes de terceros.

| Constante                                 | Valor predeterminado                                  | Fuente                                                                                                                    |
| ----------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `PROTOCOL_VERSION`                        | `4`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_CLIENT_PROTOCOL_VERSION`             | `4`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_NODE_PROTOCOL_VERSION`               | `3`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_PROBE_PROTOCOL_VERSION`              | `3`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| Tiempo de espera de la solicitud (por RPC) | `30_000` ms                                           | `packages/gateway-client/src/client.ts` (`requestTimeoutMs`)                                                              |
| Tiempo de espera de preautenticación/desafío de conexión | `15_000` ms                                           | `packages/gateway-client/src/timeouts.ts` (la variable de entorno `OPENCLAW_HANDSHAKE_TIMEOUT_MS` puede aumentar el límite del servidor/cliente emparejado) |
| Retardo inicial de reconexión             | `1_000` ms                                            | `packages/gateway-client/src/client.ts` (`GATEWAY_RECONNECT_POLICY`)                                                      |
| Retardo máximo de reconexión              | `30_000` ms                                           | `packages/gateway-client/src/client.ts` (`GATEWAY_RECONNECT_POLICY`)                                                      |
| Límite de reintento rápido tras el cierre por token de dispositivo | `250` ms                                              | `packages/gateway-client/src/client.ts`                                                                                   |
| Periodo de gracia para la detención forzada antes de `terminate()` | `250` ms                                              | `FORCE_STOP_TERMINATE_GRACE_MS`                                                                                           |
| Tiempo de espera predeterminado de `stopAndWait()` | `1_000` ms                                            | `STOP_AND_WAIT_TIMEOUT_MS`                                                                                                |
| Intervalo de tic predeterminado (antes de `hello-ok`) | `30_000` ms                                           | `packages/gateway-client/src/client.ts`                                                                                   |
| Cierre por tiempo de espera del tic       | código `4000` cuando el silencio supera `tickIntervalMs * 2` | `packages/gateway-client/src/client.ts`                                                                                   |
| `MAX_PAYLOAD_BYTES`                       | `25 * 1024 * 1024` (25 MB)                            | `src/gateway/server-constants.ts`                                                                                         |

El servidor anuncia los valores efectivos de `policy.tickIntervalMs`,
`policy.maxPayload` y `policy.maxBufferedBytes` en `hello-ok`; los clientes
deben respetar esos valores en lugar de los valores predeterminados anteriores al protocolo de enlace.

El cliente de referencia permite que las solicitudes finitas controlen su fecha límite configurada cuando
cada solicitud pendiente tiene una. Una solicitud `expectFinal` sin un
`timeoutMs` finito, cualquier solicitud con `timeoutMs: null` o una combinación de solicitudes
finitas y sin límite mantiene activo el supervisor de tics. Si los eventos entrantes y las
respuestas permanecen en silencio más allá del umbral de tiempo de espera del tic, el cliente cierra el
socket con el código `4000`, rechaza todas las solicitudes pendientes y vuelve a conectarse. No
reproduce las solicitudes rechazadas después de volver a conectarse.

## Autenticación

- La autenticación del Gateway mediante secreto compartido usa `connect.params.auth.token` o
  `connect.params.auth.password`, según el valor configurado de
  `gateway.auth.mode` (`"none" | "token" | "password" | "trusted-proxy"`).
- Los modos que incluyen identidad, como Tailscale Serve (`gateway.auth.allowTailscale: true`)
  o `gateway.auth.mode: "trusted-proxy"` sin loopback, satisfacen la comprobación de autenticación
  de conexión mediante los encabezados de la solicitud en lugar de `connect.params.auth.*`.
- El `gateway.auth.mode: "none"` de entrada privada omite por completo la autenticación
  de conexión mediante secreto compartido; no exponga ese modo en una entrada pública o no confiable.
- Tras el emparejamiento, el Gateway emite un token de dispositivo limitado
  al rol y los ámbitos de la conexión, devuelto en `hello-ok.auth.deviceToken`. Los clientes deben
  conservarlo después de cualquier conexión correcta.
- Al volver a conectarse con ese token de dispositivo almacenado, también debe
  reutilizarse el conjunto de ámbitos aprobado y almacenado para dicho token. Esto conserva
  el acceso de lectura, sondeo y estado ya concedido y evita reducir silenciosamente
  las reconexiones a un ámbito implícito más limitado y exclusivo para administradores.
- Composición de la autenticación de conexión del lado del cliente (`selectConnectAuth` en
  `packages/gateway-client/src/client.ts`):
  - `auth.password` es independiente y siempre se reenvía cuando está establecido.
  - `auth.token` se rellena por orden de prioridad: primero el token compartido explícito,
    después un `deviceToken` explícito y, por último, un token almacenado por dispositivo
    (identificado mediante `deviceId` + `role`).
  - `auth.bootstrapToken` solo se envía cuando ninguna de las opciones anteriores ha resuelto
    `auth.token`. Un token compartido o cualquier token de dispositivo resuelto lo suprime.
  - La promoción automática de un token de dispositivo almacenado durante el reintento único
    de `AUTH_TOKEN_MISMATCH` está restringida únicamente a endpoints confiables: loopback
    o `wss://` con un `tlsFingerprint` fijado. Un `wss://` público sin fijación
    no cumple los requisitos.
- El arranque integrado mediante código de configuración devuelve el Node principal
  `hello-ok.auth.deviceToken` junto con un token de operador limitado en
  `hello-ok.auth.deviceTokens` para la transferencia móvil confiable. El token de operador
  incluye `operator.talk.secrets` para las lecturas de configuración nativa de Talk, pero
  excluye los ámbitos de modificación del emparejamiento y `operator.admin`.
- Mientras un arranque mediante código de configuración no básico espera aprobación,
  los detalles de `PAIRING_REQUIRED` incluyen `recommendedNextStep: "wait_then_retry"`,
  `retryable: true` y `pauseReconnect: false`. Siga intentando conectarse con el
  mismo token de arranque hasta que se apruebe la solicitud o el token deje de
  ser válido.
- Conserve `hello-ok.auth.deviceTokens` únicamente cuando la conexión haya usado autenticación
  de arranque en un transporte confiable, como `wss://`, o mediante emparejamiento local/loopback.
- Si un cliente proporciona un `deviceToken` explícito o un `scopes` explícito,
  el conjunto de ámbitos solicitado por ese invocador sigue siendo el autoritativo; los ámbitos almacenados
  en caché solo se reutilizan cuando el cliente vuelve a utilizar el token almacenado por dispositivo.
- Los tokens de dispositivo pueden rotarse o revocarse mediante `device.token.rotate` y
  `device.token.revoke` (requiere `operator.pairing`). Rotar o revocar un
  Node u otro rol que no sea de operador también requiere `operator.admin`.
- `device.token.rotate` devuelve metadatos de rotación. Solo devuelve el token de portador
  de reemplazo en las llamadas del mismo dispositivo que ya estén autenticadas con ese
  token de dispositivo, para que los clientes que solo usan tokens puedan conservar el reemplazo antes
  de volver a conectarse. Las rotaciones compartidas o de administrador no devuelven el token de portador.
- La emisión, rotación y revocación de tokens permanecen limitadas al conjunto de roles
  aprobado registrado en la entrada de emparejamiento de ese dispositivo; la modificación de tokens no puede ampliar
  ni dirigirse a un rol de dispositivo que nunca haya concedido la aprobación del emparejamiento.
- En las sesiones con tokens de dispositivos emparejados, la administración de dispositivos se limita
  al propio dispositivo, salvo que el invocador también tenga `operator.admin`: los invocadores sin privilegios de administrador
  solo pueden administrar el token de operador de su propia entrada de dispositivo. La administración de tokens de Node
  y de otros tokens que no sean de operador es exclusiva para administradores, incluso en el propio dispositivo del invocador.
- `device.token.rotate` y `device.token.revoke` también comparan el conjunto de ámbitos
  del token de operador de destino con los ámbitos de la sesión actual del invocador.
  Los invocadores sin privilegios de administrador no pueden rotar ni revocar un token de operador con más ámbitos que los
  que ya poseen.
- Los fallos de autenticación incluyen `error.details.code` y sugerencias de recuperación:
  - `error.details.canRetryWithDeviceToken` (booleano)
  - `error.details.recommendedNextStep`: uno de `retry_with_device_token`,
    `update_auth_configuration`, `update_auth_credentials`,
    `wait_then_retry`, `review_auth_configuration`
    (`packages/gateway-protocol/src/connect-error-details.ts`).
- Comportamiento del cliente para `AUTH_TOKEN_MISMATCH`:
  - Los clientes confiables pueden intentar un único reintento limitado con un token
    almacenado en caché por dispositivo.
  - Si ese reintento falla, detenga los bucles de reconexión automática y muestre
    indicaciones sobre la acción que debe realizar el operador.
- `AUTH_SCOPE_MISMATCH` significa que se reconoció el token de dispositivo, pero no
  cubre el rol ni los ámbitos solicitados. No lo presente como un token incorrecto; solicite
  al operador que vuelva a emparejar el dispositivo o que apruebe el contrato de ámbitos más limitado o más amplio.

## Identidad y emparejamiento de dispositivos

- Los Nodes deben incluir una identidad de dispositivo estable (`device.id`) derivada de la
  huella digital de un par de claves.
- Los Gateways emiten tokens por dispositivo y rol.
- Se requieren aprobaciones de emparejamiento para los nuevos identificadores de dispositivo, salvo que esté
  habilitada la aprobación automática local.
- La aprobación automática del emparejamiento se centra en las conexiones locales directas por loopback.
- OpenClaw también dispone de una ruta limitada de autoconexión local al backend/contenedor para
  flujos auxiliares confiables con secreto compartido.
- Las conexiones desde la misma máquina mediante tailnet o LAN siguen considerándose remotas para el emparejamiento
  y requieren aprobación.
- Normalmente, los clientes WS incluyen la identidad `device` durante `connect` (operador +
  Node). Las únicas excepciones para operadores sin dispositivo son rutas de confianza explícitas:
  - autenticación correcta de la interfaz de control del operador mediante `gateway.auth.mode: "trusted-proxy"`.
  - RPC del backend `gateway-client` mediante loopback directo en la ruta auxiliar interna
    reservada.
- Omitir la identidad del dispositivo tiene consecuencias para los ámbitos. Cuando se permite
  una conexión de operador sin dispositivo mediante una ruta de confianza explícita, OpenClaw
  sigue borrando los ámbitos declarados por la propia conexión y los deja como un conjunto vacío, salvo que esa ruta tenga
  una excepción explícita para conservarlos. Los métodos restringidos por ámbitos fallan entonces con
  `missing scope`.
- La ruta auxiliar reservada del backend `gateway-client` mediante loopback directo conserva
  los ámbitos únicamente para RPC internas del plano de control local; los identificadores de backend personalizados
  no reciben esta excepción.
- Todas las conexiones deben firmar el nonce `connect.challenge` proporcionado por el servidor.

### Diagnósticos de migración de la autenticación de dispositivos

Para los clientes heredados que todavía usan el comportamiento de firma anterior al desafío, `connect`
devuelve códigos de detalle `DEVICE_AUTH_*` en `error.details.code` con un
`error.details.reason` estable.

Fallos comunes de migración:

| Mensaje                     | details.code                     | details.reason           | Significado                                            |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | El cliente omitió `device.nonce` (o lo envió vacío).     |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | El cliente firmó con un nonce obsoleto o incorrecto.            |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | La carga útil de la firma no coincide con la carga útil v2.       |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | La marca de tiempo firmada está fuera de la desviación permitida.          |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` no coincide con la huella digital de la clave pública. |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | Falló el formato o la canonicalización de la clave pública.         |

Objetivo de la migración:

- Espere siempre a `connect.challenge`.
- Firme la carga útil v2 que incluye el nonce del servidor.
- Envíe el mismo nonce en `connect.params.device.nonce`.
- La carga útil de firma preferida es `v3`
  (`buildDeviceAuthPayloadV3` en `packages/gateway-client/src/device-auth.ts`),
  que vincula `platform` y `deviceFamily`, además de
  los campos de dispositivo, cliente, rol, ámbitos, token y nonce.
- Las firmas heredadas `v2` siguen aceptándose por compatibilidad, pero la fijación
  de los metadatos del dispositivo emparejado continúa controlando la política de comandos al volver a conectarse.

## TLS y fijación

- TLS es compatible con las conexiones WS (configuración `gateway.tls`).
- Los clientes pueden fijar opcionalmente la huella digital del certificado del Gateway mediante
  `gateway.remote.tlsFingerprint` o la opción de la CLI `--tls-fingerprint`.

## Alcance

Este protocolo expone la API completa del Gateway: estado, canales, modelos, chat,
agente, sesiones, Nodes, aprobaciones y más. La superficie exacta está definida por
los esquemas de TypeBox reexportados desde `packages/gateway-protocol/src/schema.ts`.

## Contenido relacionado

- [Crear un cliente del Gateway](https://docs.openclaw.ai/gateway/clients)
- [Integrar OpenClaw](https://docs.openclaw.ai/gateway/embedding)
- [Protocolo de puente](/es/gateway/bridge-protocol)
- [Guía operativa del Gateway](/es/gateway)
