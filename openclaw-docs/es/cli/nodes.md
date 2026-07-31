---
read_when:
    - Estás gestionando Nodes emparejados (cámaras, pantalla, lienzo)
    - Es necesario aprobar las solicitudes o invocar comandos de Node
summary: Referencia de la CLI para `openclaw nodes` (estado, emparejamiento, invocación, cámara/lienzo/pantalla/ubicación/notificaciones)
title: Nodos
x-i18n:
    generated_at: "2026-07-26T04:33:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 53003bcd3d30b0e754aa0717452700595c0cf69d9ecd6301b8a1bf320ea1838a
    source_path: cli/nodes.md
    workflow: 16
---

# `openclaw nodes`

Gestiona los nodos (dispositivos) emparejados e invoca las capacidades de los nodos.

Relacionado: [Descripción general de los nodos](/es/nodes) - [Presencia activa en el equipo](/es/nodes/presence) - [Nodos de cámara](/es/nodes/camera) - [Nodos de imágenes](/es/nodes/images)

Opciones comunes en todos los subcomandos: `--url <url>`, `--token <token>`, `--timeout <ms>` (valor predeterminado: `10000`), `--json`.

## Estado

```bash
openclaw nodes status
openclaw nodes status --connected
openclaw nodes status --last-connected 24h
openclaw nodes list
openclaw nodes describe --node <idOrNameOrIp>
```

Tanto `status` como `list` aceptan `--connected` (solo nodos conectados) y `--last-connected <duration>` (p. ej., `24h`, `7d`; solo nodos que se hayan conectado dentro de ese intervalo). `list` muestra los nodos pendientes y emparejados en tablas separadas; las filas emparejadas incluyen el tiempo transcurrido desde la conexión más reciente (Last Connect). `status` muestra una única tabla combinada con detalles de capacidad, versión y última entrada de cada nodo. Un nodo macOS conectado solo informa de la última entrada después de que se habilite **Detección de actividad en el equipo** y se conceda el permiso de Accesibilidad; la fila más reciente se marca como `active`. Consulta [Presencia activa en el equipo](/es/nodes/presence). `describe` muestra las capacidades, los permisos, la actividad y los comandos de invocación efectivos o pendientes de un nodo.

## Emparejamiento

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name <displayName>
```

Estos comandos gestionan el almacén `node.pair.*` propiedad del Gateway, que es independiente del emparejamiento de dispositivos (`openclaw devices approve`) que controla el protocolo de enlace `connect` de WS del nodo. Consulta [Nodos](/es/nodes) para conocer la relación entre ambos.

- `remove` revoca la entrada de rol emparejado del nodo. En un nodo respaldado por un dispositivo, esto revoca el rol `node` en el almacén de emparejamiento de dispositivos y desconecta sus sesiones con rol de nodo: un dispositivo con varios roles conserva su fila y solo pierde el rol `node`; la fila de un dispositivo que solo tiene el rol de nodo se elimina. También borra cualquier registro heredado coincidente de emparejamiento de nodos propiedad del Gateway.
- `pending` solo necesita el ámbito `operator.pairing`.
- `gateway.nodes.pairing.autoApproveCidrs` puede omitir el paso pendiente para el primer emparejamiento de un dispositivo `role: node` que sea explícitamente de confianza. Está desactivado de forma predeterminada y no aprueba ampliaciones de roles.
- `gateway.nodes.pairing.sshVerify` (activado de forma predeterminada) aprueba automáticamente el primer emparejamiento de un dispositivo `role: node` cuando el Gateway puede verificar la clave del dispositivo mediante SSH con el host del nodo; la primera superficie de capacidades se aprueba en el mismo paso. Consulta [Emparejamiento de nodos](/es/gateway/pairing#ssh-verified-device-auto-approval-default).
- Los requisitos de ámbito de `approve` dependen de los comandos declarados por la solicitud pendiente:
  - solicitud sin comandos: `operator.pairing`
  - comandos ordinarios de nodo: `operator.pairing` + `operator.write`
  - comandos que requieren privilegios administrativos (`system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `fs.listDir` y `system.execApprovals.get/set`): `operator.pairing` + `operator.admin`
- Ámbito de `remove`: `operator.pairing` puede eliminar filas de nodos que no sean operadores; un llamador con token de dispositivo que revoque su propio rol de nodo en un dispositivo con varios roles necesita además `operator.admin`.

## Invocación

```bash
openclaw nodes invoke --node <id> --command system.which --params '{"bins":["uname"]}'
```

Indicadores:

- `--command <command>` (obligatorio): p. ej., `canvas.eval`.
- `--params <json>`: cadena de objeto JSON (valor predeterminado: `{}`).
- `--invoke-timeout <ms>`: tiempo de espera de invocación del nodo (valor predeterminado: `15000`).
- `--idempotency-key <key>`: clave de idempotencia opcional.

`system.run` y `system.run.prepare` están bloqueados aquí; para ejecutar comandos de shell, utiliza en su lugar la herramienta `exec` con `host=node`. `system.which` se permite mediante `invoke`.

## Notificación, push, ubicación y pantalla

```bash
openclaw nodes notify --node <id> --title "Build" --body "Done" --priority timeSensitive
openclaw nodes push --node <id> --title "OpenClaw" --environment sandbox
openclaw nodes location get --node <id> --accuracy precise
openclaw nodes screen record --node <id> --duration 10s --fps 10 --out ./clip.mp4
```

- `notify` envía una notificación local a un nodo que declara `system.notify`, incluidos los nodos macOS, iOS, Android y watchOS directos. La entrega directa a watchOS requiere que OpenClaw esté activo. Requiere `--title` o `--body`. Opciones: `--sound <name>`, `--priority <passive|active|timeSensitive>`, `--delivery <system|overlay|auto>` (valor predeterminado: `system`), `--invoke-timeout <ms>` (valor predeterminado: `15000`).
- `push` envía una notificación push de prueba de APNs a un nodo iOS. Opciones: `--title <text>` (valor predeterminado: `OpenClaw`), `--body <text>`, `--environment <sandbox|production>` para sustituir el entorno de APNs detectado.
- `location get` obtiene la ubicación actual del nodo. Opciones: `--max-age <ms>` (reutiliza una posición almacenada en caché), `--accuracy <coarse|balanced|precise>`, `--location-timeout <ms>` (valor predeterminado: `10000`), `--invoke-timeout <ms>` (valor predeterminado: `20000`).
- `screen record` captura un clip breve y muestra la ruta guardada (o escribe JSON con `--json`). Opciones: `--screen <index>` (valor predeterminado: `0`), `--duration <ms|10s>` (valor predeterminado: `10000`), `--fps <fps>` (valor predeterminado: `10`), `--no-audio`, `--out <path>`, `--invoke-timeout <ms>` (valor predeterminado: `120000`).

Los comandos de Cámara y Canvas tienen su propia documentación: [Nodos de cámara](/es/nodes/camera), [Canvas](/es/platforms/mac/canvas). Canvas se implementa mediante el Plugin experimental de Canvas incluido; el núcleo conserva `openclaw nodes canvas` como punto de montaje de compatibilidad.

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
- [Nodos](/es/nodes)
