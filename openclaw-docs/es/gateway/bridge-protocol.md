---
read_when:
    - Investigación de código antiguo del cliente de Node o de registros de emparejamiento archivados
    - Auditoría de lo que solía exponer la superficie heredada de Node
summary: 'Protocolo puente histórico (nodos heredados): JSONL sobre TCP, emparejamiento, RPC con ámbito definido'
title: Protocolo de puente
x-i18n:
    generated_at: "2026-07-26T04:37:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6e8b69c59f2170439f0e7b139bf5bbdb429d7c9d8dde7b36cd64aab63939c95d
    source_path: gateway/bridge-protocol.md
    workflow: 16
---

<Warning>
El puente TCP se ha **eliminado**. Las compilaciones actuales de OpenClaw no incluyen el listener del puente y las claves de configuración `bridge.*` ya no forman parte del esquema. Esta página se conserva únicamente como referencia histórica. Utilice el [protocolo del Gateway](/es/gateway/protocol) para todos los clientes de nodos y operadores.
</Warning>

## Por qué existía

- **Límite de seguridad**: exponía una pequeña lista de permitidos en lugar de toda la superficie de la API del Gateway.
- **Emparejamiento e identidad del nodo**: el Gateway gestionaba la admisión de nodos y la vinculaba a un token por nodo.
- **Experiencia de descubrimiento**: los nodos podían descubrir gateways mediante Bonjour en la LAN o conectarse directamente a través de una tailnet.
- **WS de loopback**: el plano de control WS completo permanecía local, salvo que se estableciera un túnel mediante SSH.

## Transporte

- TCP, un objeto JSON por línea (JSONL).
- TLS opcional (`bridge.tls.enabled: true`).
- El puerto predeterminado del listener era `18790`.

Cuando TLS estaba habilitado, los registros TXT de descubrimiento incluían `bridgeTls=1` y `bridgeTlsSha256` como indicio no secreto. Los registros TXT de Bonjour/mDNS no están autenticados; los clientes no podían considerar la huella digital anunciada como una fijación autoritativa sin otra verificación fuera de banda.

## Protocolo de enlace y emparejamiento

1. El cliente envía `hello` con los metadatos del nodo y el token (si ya está emparejado).
2. Si no está emparejado, el Gateway responde con `error` (`NOT_PAIRED` / `UNAUTHORIZED`).
3. El cliente envía `pair-request`.
4. El Gateway espera la aprobación y, a continuación, envía `pair-ok` y `hello-ok`.

`hello-ok` devolvía anteriormente `serverName`; las superficies de plugins alojados ahora se anuncian mediante `pluginSurfaceUrls` en el protocolo actual del Gateway (Canvas/A2UI utiliza `pluginSurfaceUrls.canvas`).

## Tramas

Del cliente al Gateway:

- `req` / `res`: RPC de Gateway con ámbito limitado (chat, sesiones, configuración, estado, activación por voz, skills.bins).
- `event`: señales del nodo (transcripción de voz, solicitud del agente, suscripción al chat, ciclo de vida de la ejecución).

Del Gateway al cliente:

- `invoke` / `invoke-res`: comandos del nodo (`canvas.*`, `camera.*`, `screen.record`, `location.get`, `sms.send`).
- `event`: actualizaciones del chat para las sesiones suscritas.
- `ping` / `pong`: mantenimiento de conexión.

La aplicación de la lista de permitidos residía en `src/gateway/server-bridge.ts` (eliminado).

## Eventos del ciclo de vida de la ejecución

Los nodos emitían `exec.finished` para exponer la actividad `system.run` completada, que el Gateway asignaba a eventos del sistema (los nodos heredados también podían emitir `exec.started`). `exec.denied` marcaba un intento `system.run` denegado como denegación terminal sin poner en cola un evento del sistema ni activar el trabajo del agente.

Campos de la carga útil (todos opcionales salvo que se indique lo contrario):

| Campo                            | Notas                                                                                          |
| -------------------------------- | ---------------------------------------------------------------------------------------------- |
| `sessionKey`                     | Obligatorio. Sesión del agente para correlacionar eventos y, en el caso de `exec.finished`, entregar eventos del sistema. |
| `runId`                          | Id. único de ejecución para agrupar.                                                                   |
| `command`                        | Cadena del comando sin procesar o con formato.                                                               |
| `exitCode`, `timedOut`, `output` | Detalles de finalización (solo si ha finalizado).                                                            |
| `reason`                         | Motivo de la denegación (solo si se ha denegado).                                                                   |

## Uso histórico de la tailnet

- Vinculaba el puente a una IP de la tailnet: `bridge.bind: "tailnet"` en `~/.openclaw/openclaw.json` (solo histórico; `bridge.*` ya no es una configuración válida).
- Los clientes se conectaban mediante el nombre de MagicDNS o la IP de la tailnet.
- Bonjour no atraviesa redes; de lo contrario, se requería DNS-SD de área extensa o un host y puerto manuales.

## Control de versiones

El puente correspondía implícitamente a la versión 1, sin negociación de mínimos y máximos. Los clientes actuales de nodos y operadores utilizan el [protocolo del Gateway](/es/gateway/protocol) WebSocket, que sí negocia un intervalo de versiones del protocolo.

## Contenido relacionado

- [Protocolo del Gateway](/es/gateway/protocol)
- [Nodos](/es/nodes)
