---
read_when:
    - Configuración de integraciones de IDE basadas en ACP
    - Depuración del enrutamiento de sesiones ACP al Gateway
summary: Ejecutar el puente ACP para integraciones con IDE
title: ACP
x-i18n:
    generated_at: "2026-07-26T05:07:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: becdcfdd1cc62b206cc92e9b8248c79a2ff63cfc3779d8a124b9713e779ad33c
    source_path: cli/acp.md
    workflow: 16
---

Ejecute el puente [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) que se comunica con un Gateway de OpenClaw.

`openclaw acp` se comunica mediante ACP a través de stdio para los IDE y reenvía las solicitudes al Gateway mediante WebSocket, manteniendo las sesiones ACP asociadas a claves de sesión del Gateway. Es un puente ACP respaldado por el Gateway, no un entorno de ejecución de editor completamente nativo de ACP: se centra en el enrutamiento de sesiones, la entrega de solicitudes y las actualizaciones en streaming.

Si se desea que un cliente MCP externo se comunique directamente con las conversaciones de los canales de OpenClaw en lugar de alojar una sesión del entorno ACP, utilice [`openclaw mcp serve`](/es/cli/mcp).

## Qué no es

`openclaw acp` significa que OpenClaw actúa como servidor ACP: un IDE o cliente ACP se conecta a OpenClaw, y OpenClaw reenvía ese trabajo a una sesión del Gateway.

Esto es diferente de [Agentes ACP](/es/tools/acp-agents), donde OpenClaw ejecuta un entorno externo como Codex o Claude Code mediante `acpx`.

Regla rápida:

- el editor/cliente desea comunicarse mediante ACP con OpenClaw: utilice `openclaw acp`
- OpenClaw debe iniciar Codex/Claude/Gemini como entorno ACP: utilice `/acp spawn` y [Agentes ACP](/es/tools/acp-agents)

## Matriz de compatibilidad

| Área de ACP                                                            | Estado       | Notas                                                                                                                                                                                                                                 |
| --------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `initialize`, `newSession`, `prompt`, `cancel`                        | Implementado | Flujo principal del puente mediante stdio hacia chat/send + abort del Gateway.                                                                                                                                                         |
| `listSessions`, comandos con barra                                     | Implementado | La lista de sesiones funciona con el estado de sesiones del Gateway mediante paginación limitada por cursor y filtrado por `cwd` cuando las filas de sesión del Gateway contienen metadatos del espacio de trabajo; los comandos se anuncian mediante `available_commands_update`. |
| Metadatos de linaje de sesiones                                       | Implementado | Las listas de sesiones y las instantáneas de información de sesión incluyen el linaje principal y secundario de OpenClaw en `_meta`, de modo que los clientes ACP puedan representar gráficos de subagentes sin canales laterales privados del Gateway. |
| `resumeSession`, `closeSession`                                       | Implementado | La reanudación vuelve a vincular una sesión ACP con una sesión existente del Gateway sin reproducir el historial. El cierre cancela el trabajo activo del puente, resuelve las solicitudes pendientes como canceladas y libera el estado de sesión del puente. |
| `loadSession`                                                         | Parcial      | Vuelve a vincular la sesión ACP con una clave de sesión del Gateway y reproduce el historial del registro de eventos ACP para las sesiones creadas por el puente. Las sesiones antiguas o sin registro recurren al texto almacenado del usuario y el asistente. |
| Contenido de la solicitud (`text`, `resource` incrustados, imágenes) | Parcial      | El texto y los recursos se convierten en entrada de chat; las imágenes se convierten en archivos adjuntos del Gateway.                                                                                                                 |
| Modos de sesión                                                       | Parcial      | Se admite `session/set_mode`; el puente expone controles de sesión respaldados por el Gateway para el nivel de pensamiento, la verbosidad de las herramientas, el razonamiento, el detalle de uso y las acciones con privilegios elevados. Las superficies más amplias de modos y configuración nativas de ACP siguen fuera del alcance. |
| Streaming de pensamiento                                              | Implementado | El contenido de pensamiento del modelo se transmite como actualizaciones de sesión `agent_thought_chunk`. No se emiten planes de sesión nativos de ACP.                                                                                   |
| Actualizaciones de información y uso de la sesión                     | Parcial      | El puente emite notificaciones `session_info_update` y `usage_update` de mejor esfuerzo a partir de instantáneas almacenadas en caché de la sesión del Gateway. El uso es aproximado y solo se envía cuando los totales de tokens del Gateway están marcados como recientes. |
| Streaming de herramientas                                             | Parcial      | Los eventos `tool_call`/`tool_call_update` incluyen E/S sin procesar, contenido de texto y ubicaciones de archivos de mejor esfuerzo cuando los argumentos o resultados de las herramientas del Gateway los exponen. No se exponen terminales incrustados ni resultados más completos nativos de diferencias. |
| Aprobaciones de ejecución                                             | Parcial      | Las solicitudes de aprobación de ejecución del Gateway durante turnos activos de solicitudes ACP se retransmiten al cliente ACP mediante `session/request_permission`.                                                                          |
| Servidores MCP por sesión (`mcpServers`)                            | No compatible | El modo puente rechaza las solicitudes de servidores MCP por sesión. Configure MCP en el Gateway o el agente de OpenClaw.                                                                                                             |
| Métodos del sistema de archivos del cliente (`fs/read_text_file`, `fs/write_text_file`) | No compatible | El puente no llama a los métodos del sistema de archivos del cliente ACP.                                                                                                                                                              |
| Métodos de terminal del cliente (`terminal/*`)                      | No compatible | El puente no crea terminales del cliente ACP ni transmite identificadores de terminal mediante llamadas a herramientas.                                                                                                               |

## Limitaciones conocidas

- `loadSession` reproduce el historial completo del registro de eventos ACP solo para las sesiones creadas por el puente. Las sesiones antiguas o sin registro utilizan el respaldo de la transcripción y no reconstruyen llamadas históricas a herramientas ni avisos del sistema.
- Si varios clientes ACP comparten la misma clave de sesión del Gateway, el enrutamiento de eventos y cancelaciones es de mejor esfuerzo, en lugar de estar estrictamente aislado por cliente. Se recomienda usar las sesiones `acp-bridge:<uuid>` aisladas predeterminadas cuando se necesiten turnos locales del editor claramente separados.
- Los estados de detención del Gateway se traducen en motivos de detención de ACP, pero esa correspondencia es menos expresiva que la de un entorno de ejecución completamente nativo de ACP.
- Los controles de sesión exponen un subconjunto específico de opciones del Gateway: nivel de pensamiento, verbosidad de las herramientas, razonamiento, detalle de uso y acciones con privilegios elevados. La selección de modelos y los controles del host de ejecución no se exponen como opciones de configuración de ACP.
- `session_info_update` y `usage_update` se derivan de instantáneas de sesiones del Gateway, no de la contabilidad en tiempo real de un entorno de ejecución nativo de ACP. El uso es aproximado, no incluye datos de costes y solo se emite cuando el Gateway marca como recientes los datos del total de tokens.
- Los datos de seguimiento de las herramientas son de mejor esfuerzo: el puente muestra las rutas de archivos que aparecen en argumentos o resultados conocidos de herramientas, pero no emite terminales ACP ni diferencias estructuradas de archivos.
- La retransmisión de aprobaciones de ejecución se limita al turno activo de la solicitud ACP; se ignoran las aprobaciones de otras sesiones del Gateway.

## Uso

```bash
openclaw acp

# Gateway remoto
openclaw acp --url wss://gateway-host:18789 --token <token>

# Gateway remoto (token desde un archivo)
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# Conectarse a una clave de sesión existente
openclaw acp --session agent:main:main

# Conectarse mediante una etiqueta (debe existir previamente)
openclaw acp --session-label "support inbox"

# Restablecer la clave de sesión antes de la primera solicitud
openclaw acp --session agent:main:main --reset-session
```

## Cliente ACP (depuración)

Utilice el cliente ACP integrado para realizar una comprobación básica del puente sin un IDE. Este inicia el puente ACP y permite escribir solicitudes de forma interactiva.

```bash
openclaw acp client

# Dirigir el puente iniciado a un Gateway remoto
openclaw acp client --server-args --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# Reemplazar el comando del servidor (valor predeterminado: openclaw)
openclaw acp client --server "node" --server-args openclaw.mjs acp --url ws://127.0.0.1:19001
```

Modelo de permisos (modo de depuración del cliente):

- La aprobación automática se basa en una lista de permitidos y solo se aplica a identificadores de herramientas principales de confianza.
- La aprobación automática de `read` se limita al directorio de trabajo actual (`--cwd` cuando está configurado).
- ACP solo aprueba automáticamente categorías restringidas de solo lectura: llamadas `read` limitadas al directorio de trabajo activo, además de herramientas de búsqueda de solo lectura (`search`, `web_search`, `memory_search`). Las herramientas desconocidas o no principales, las lecturas fuera del ámbito, las herramientas con capacidad de ejecución, las herramientas del plano de control, las herramientas que realizan modificaciones y los flujos interactivos siempre requieren la aprobación explícita de la solicitud.
- El valor `toolCall.kind` proporcionado por el servidor se trata como metadatos no fiables, no como fuente de autorización.
- Esta política del puente ACP es independiente de los permisos del entorno ACPX. Si se ejecuta OpenClaw mediante el backend `acpx`, `plugins.entries.acpx.config.permissionMode=approve-all` es el interruptor de emergencia «yolo» para esa sesión del entorno.

## Pruebas rápidas del protocolo

Para la depuración a nivel de protocolo, inicie un Gateway con estado aislado y controle `openclaw acp` mediante stdio con un cliente JSON-RPC de ACP. Incluya `initialize`, `session/new`, `session/list` con un valor `cwd` absoluto, `session/resume`, `session/close`, cierre duplicado y reanudación inexistente.

La prueba debe incluir las capacidades de ciclo de vida anunciadas, una fila de sesión respaldada por el Gateway, notificaciones de actualización y el registro `sessions.list` del Gateway:

```json
{
  "initialize": {
    "protocolVersion": 1,
    "agentCapabilities": {
      "sessionCapabilities": {
        "list": {},
        "resume": {},
        "close": {}
      }
    }
  },
  "listSessions": {
    "sessions": [
      {
        "sessionId": "agent:main:acp-smoke",
        "cwd": "/path/to/workspace",
        "_meta": {
          "sessionKey": "agent:main:acp-smoke",
          "kind": "direct"
        }
      }
    ],
    "nextCursor": null
  },
  "notifications": ["session_info_update", "available_commands_update", "usage_update"],
  "gatewayLogTail": ["[gateway] ready", "[ws] ⇄ res ✓ sessions.list 305ms"]
}
```

Evite utilizar `openclaw gateway call sessions.list` como única prueba de ACP. Esa ruta de la CLI puede solicitar una ampliación del ámbito de operador con token nuevo; la corrección del puente ACP se demuestra mediante tramas de stdio de ACP junto con el registro `sessions.list` del Gateway.

## Cómo utilizarlo

Utilice ACP cuando un IDE (u otro cliente) se comunique mediante Agent Client Protocol y se desee que controle una sesión del Gateway de OpenClaw.

1. Asegúrese de que el Gateway esté en ejecución (local o remoto).
2. Configure el destino del Gateway (mediante la configuración o indicadores).
3. Configure el IDE para que ejecute `openclaw acp` mediante stdio.

Ejemplo de configuración (persistente):

```bash
openclaw config set gateway.remote.url wss://gateway-host:18789
openclaw config set gateway.remote.token <token>
```

Ejemplo de ejecución directa (sin escribir la configuración):

```bash
openclaw acp --url wss://gateway-host:18789 --token <token>
# preferible para la seguridad del proceso local
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token
```

## Selección de agentes

ACP no selecciona agentes directamente. Enruta mediante la clave de sesión del Gateway. Use claves de sesión con ámbito de agente para dirigirse a un agente específico:

```bash
openclaw acp --session agent:main:main
openclaw acp --session agent:design:main
openclaw acp --session agent:qa:bug-123
```

Cada sesión de ACP se asigna a una única clave de sesión del Gateway. Un agente puede tener muchas sesiones; ACP usa de forma predeterminada una sesión aislada `acp-bridge:<uuid>`, a menos que se sobrescriba la clave o la etiqueta.

Los `mcpServers` por sesión no son compatibles con el modo puente. Si un cliente ACP los envía durante `newSession` o `loadSession`, el puente devuelve un error claro en lugar de ignorarlos silenciosamente.

Si se desea que las sesiones respaldadas por ACPX vean las herramientas de plugins de OpenClaw o determinadas herramientas integradas, como `cron`, habilite los puentes MCP de ACPX del lado del Gateway en lugar de intentar pasar `mcpServers` por sesión. Consulte [Agentes ACP](/es/tools/acp-agents-setup#plugin-tools-mcp-bridge) y [Puente MCP de herramientas de OpenClaw](/es/tools/acp-agents-setup#openclaw-tools-mcp-bridge).

## Uso desde `acpx` (Codex, Claude y otros clientes ACP)

Si se desea que un agente de programación como Codex o Claude Code se comunique con el bot de OpenClaw mediante ACP, use `acpx` con su destino `openclaw` integrado.

Flujo habitual:

1. Ejecute el Gateway y asegúrese de que el puente ACP pueda acceder a él.
2. Dirija `acpx openclaw` a `openclaw acp`.
3. Indique la clave de sesión de OpenClaw que debe usar el agente de programación.

Ejemplos:

```bash
# Solicitud única a la sesión ACP predeterminada de OpenClaw
acpx openclaw exec "Resume el estado de la sesión activa de OpenClaw."

# Sesión persistente con nombre para turnos posteriores
acpx openclaw sessions ensure --name codex-bridge
acpx openclaw -s codex-bridge --cwd /path/to/repo \
  "Pide a mi agente de trabajo de OpenClaw contexto reciente relevante para este repositorio."
```

Si se desea que `acpx openclaw` se dirija siempre a un Gateway y una clave de sesión específicos, sobrescriba el comando del agente `openclaw` en `~/.acpx/config.json`:

```json
{
  "agents": {
    "openclaw": {
      "command": "env OPENCLAW_HIDE_BANNER=1 OPENCLAW_SUPPRESS_NOTES=1 openclaw acp --url ws://127.0.0.1:18789 --token-file ~/.openclaw/gateway.token --session agent:main:main"
    }
  }
}
```

Para un checkout local del repositorio de OpenClaw, use el punto de entrada directo de la CLI en lugar del ejecutor de desarrollo para que el flujo de ACP se mantenga limpio:

```bash
env OPENCLAW_HIDE_BANNER=1 OPENCLAW_SUPPRESS_NOTES=1 node openclaw.mjs acp ...
```

Esta es la forma más sencilla de permitir que Codex, Claude Code u otro cliente compatible con ACP obtenga información contextual de un agente de OpenClaw sin extraerla de un terminal.

## Configuración del editor Zed

Añada un agente ACP personalizado en `~/.config/zed/settings.json` (o use la interfaz de configuración de Zed):

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": ["acp"],
      "env": {}
    }
  }
}
```

Para dirigirse a un Gateway o agente específico:

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": [
        "acp",
        "--url",
        "wss://gateway-host:18789",
        "--token",
        "<token>",
        "--session",
        "agent:design:main"
      ],
      "env": {}
    }
  }
}
```

En Zed, abra el panel Agent y seleccione "OpenClaw ACP" para iniciar un hilo.

## Asignación de sesiones

De forma predeterminada, las sesiones del puente ACP obtienen una clave de sesión aislada del Gateway con el prefijo `acp-bridge:`. Estas sesiones de puente del modelo normal son sintéticas y desechables: están sujetas a la eliminación de entradas obsoletas y no se consideran superficies protegidas de conversación humana. Para reutilizar una sesión conocida, pase una clave o etiqueta de sesión:

- `--session <key>`: usa una clave de sesión específica del Gateway.
- `--session-label <label>`: resuelve una sesión existente por etiqueta.
- `--reset-session`: genera un nuevo identificador de sesión para esa clave (misma clave, nueva transcripción).

Si el cliente ACP admite metadatos, se pueden sobrescribir por sesión:

```json
{
  "_meta": {
    "sessionKey": "agent:main:main",
    "sessionLabel": "support inbox",
    "resetSession": true
  }
}
```

Obtenga más información sobre las claves de sesión en [/concepts/session](/es/concepts/session).

## Opciones

- `--url <url>`: URL WebSocket del Gateway (el valor predeterminado es `gateway.remote.url` cuando está configurado).
- `--token <token>`: token de autenticación del Gateway.
- `--token-file <path>`: lee el token de autenticación del Gateway desde un archivo.
- `--password <password>`: contraseña de autenticación del Gateway.
- `--password-file <path>`: lee la contraseña de autenticación del Gateway desde un archivo.
- `--session <key>`: clave de sesión predeterminada.
- `--session-label <label>`: etiqueta de sesión predeterminada que se resolverá.
- `--require-existing`: genera un error si la clave o etiqueta de sesión no existe.
- `--reset-session`: restablece la clave de sesión antes del primer uso.
- `--no-prefix-cwd`: no antepone el directorio de trabajo a los prompts.
- `--provenance <off|meta|meta+receipt>`: incluye metadatos de procedencia o recibos de ACP.
- `--verbose, -v`: registro detallado en stderr.

Nota de seguridad:

- `--token` y `--password` pueden ser visibles en las listas de procesos locales de algunos sistemas. Es preferible usar `--token-file`/`--password-file` o variables de entorno (`OPENCLAW_GATEWAY_TOKEN`, `OPENCLAW_GATEWAY_PASSWORD`).
- La resolución de autenticación del Gateway sigue el contrato compartido que utilizan otros clientes del Gateway:
  - modo local: entorno (`OPENCLAW_GATEWAY_*`) y después `gateway.auth.*`; recurre a `gateway.remote.*` solo cuando `gateway.auth.*` no está definido (un SecretRef local configurado pero no resuelto produce un error seguro en lugar de recurrir silenciosamente a otra opción)
  - modo remoto: `gateway.remote.*` con alternativa de entorno/configuración según las reglas de precedencia remota
  - `--url` permite sobrescrituras de forma segura y no reutiliza credenciales implícitas de configuración o entorno; pase `--token`/`--password` explícitos (o sus variantes de archivo)

### Opciones de `acp client`

- `--cwd <dir>`: directorio de trabajo de la sesión ACP.
- `--server <command>`: comando del servidor ACP (valor predeterminado: `openclaw`).
- `--server-args <args...>`: argumentos adicionales que se pasan al servidor ACP.
- `--server-verbose`: habilita el registro detallado en el servidor ACP.
- `--verbose, -v`: registro detallado del cliente.
- `openclaw acp client` establece `OPENCLAW_SHELL=acp-client` en el proceso de puente iniciado, que puede utilizarse para reglas de shell/perfil específicas del contexto.

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
- [Agentes ACP](/es/tools/acp-agents)
