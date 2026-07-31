---
read_when:
    - Ejecución de entornos de programación mediante ACP
    - Configuración de sesiones ACP vinculadas a conversaciones en canales de mensajería
    - Vinculación de una conversación de un canal de mensajería a una sesión persistente de ACP
    - Solución de problemas del backend de ACP, la conexión del plugin o la entrega de finalizaciones
    - Uso de los comandos /acp desde el chat
sidebarTitle: ACP agents
summary: Ejecuta entornos externos de programación (Claude Code, Cursor, Gemini CLI, ACP explícito de Codex, ACP de OpenClaw, OpenCode) mediante el backend de ACP
title: Agentes ACP
x-i18n:
    generated_at: "2026-07-26T04:54:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fc7f32ff927c7e949be1595f6aa00ed034a51185c6a6b1e0df01a242954667d1
    source_path: tools/acp-agents.md
    workflow: 16
---

Las sesiones del [Protocolo de cliente de agentes (ACP)](https://agentclientprotocol.com/) permiten que
OpenClaw ejecute entornos externos de programación (Claude Code, Cursor, Copilot, Droid,
OpenClaw ACP, OpenCode, Gemini CLI y otros entornos ACPX compatibles)
mediante un plugin de backend ACP. Cada ejecución se registra como una
[tarea en segundo plano](/es/automation/tasks).

<Note>
**ACP es la vía para entornos externos, no la vía predeterminada de Codex.** El plugin
nativo del servidor de aplicaciones de Codex gestiona los controles `/codex ...` y el entorno de ejecución
integrado `openai/gpt-*` predeterminado para los turnos del agente; ACP gestiona los controles `/acp ...`
y las sesiones `sessions_spawn({ runtime: "acp" })`.

Para permitir que Codex o Claude Code se conecten directamente como clientes MCP externos a
conversaciones existentes de canales de OpenClaw, se debe usar
[`openclaw mcp serve`](/es/cli/mcp) en lugar de ACP.
</Note>

## ¿Qué página necesito?

| Objetivo                                                                                         | Opción                                | Notas                                                                                                                                                                                  |
| ------------------------------------------------------------------------------------------------ | ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Vincular o controlar Codex en la conversación actual                                             | `/codex bind`, `/codex threads`       | Vía nativa del servidor de aplicaciones de Codex cuando el plugin `codex` está habilitado: respuestas vinculadas al chat, reenvío de imágenes, modelo/rapidez/permisos, detención y orientación. ACP es una alternativa explícita |
| Ejecutar Claude Code, Gemini CLI, Codex ACP explícito u otro entorno externo _mediante_ OpenClaw | Esta página                           | Sesiones vinculadas al chat, `/acp spawn`, `sessions_spawn({ runtime: "acp" })`, tareas en segundo plano y controles del entorno de ejecución                                                                |
| Exponer una sesión del Gateway de OpenClaw _como_ servidor ACP para un editor o cliente          | [`openclaw acp`](/es/cli/acp)            | Modo puente: un IDE/cliente se comunica mediante ACP con OpenClaw a través de stdio/WebSocket                                                                                           |
| Reutilizar una CLI de IA local como modelo alternativo de solo texto                             | [Backends de CLI](/es/gateway/cli-backends) | No es ACP: no dispone de herramientas de OpenClaw, controles ACP ni entorno de ejecución externo                                                                                       |

## ¿Funciona sin configuración adicional?

Sí, después de instalar el plugin oficial del entorno de ejecución ACP:

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

Los repositorios de código fuente pueden usar el plugin local del espacio de trabajo `extensions/acpx` después de
`pnpm install`. Se debe ejecutar `/acp doctor` para comprobar que esté preparado.

OpenClaw solo informa a los agentes sobre la creación de procesos ACP cuando ACP es **realmente utilizable**:
ACP debe estar habilitado, el despacho no debe estar deshabilitado, la sesión actual
no debe estar bloqueada por el entorno aislado y debe haber un backend de ejecución cargado y en buen estado. Si
alguna condición no se cumple, las Skills de ACP y las instrucciones de ACP de `sessions_spawn` permanecen ocultas
para que el agente no sugiera un backend no disponible.

<AccordionGroup>
  <Accordion title="Problemas habituales de la primera ejecución">
    - Si se establece `plugins.allow`, este constituye un inventario restrictivo de plugins y **debe** incluir `acpx`; de lo contrario, el backend ACP instalado queda bloqueado intencionadamente (`/acp doctor` informa de la entrada ausente en la lista de permitidos).
    - El adaptador ACP de Codex se incluye con el plugin `acpx` y se inicia localmente cuando es posible.
    - Codex ACP se ejecuta con un `CODEX_HOME` aislado. OpenClaw copia desde la configuración de Codex del host las entradas de confianza de proyectos fiables y la configuración segura de enrutamiento de modelos/proveedores (`model`, `model_provider`, `model_reasoning_effort`, `sandbox_mode` y los campos seguros de `model_providers.<name>`); la autenticación, las notificaciones y los hooks permanecen únicamente en la configuración del host.
    - Los adaptadores de otros entornos de destino pueden descargarse bajo demanda con `npx` durante el primer uso.
    - La autenticación del proveedor ya debe existir en el host para ese entorno.
    - Si el host no tiene npm ni acceso a la red, las descargas de adaptadores durante la primera ejecución fallan hasta que las cachés se hayan preparado previamente o el adaptador se instale de otra forma.

  </Accordion>
  <Accordion title="Requisitos previos del entorno de ejecución">
    ACP inicia un proceso real de un entorno externo. OpenClaw gestiona el enrutamiento,
    el estado de las tareas en segundo plano, la entrega, las vinculaciones y las políticas; el entorno gestiona
    el inicio de sesión en su proveedor, el catálogo de modelos, el comportamiento del sistema de archivos y las herramientas nativas.

    Antes de atribuir el problema a OpenClaw, se debe comprobar lo siguiente:

    - `/acp doctor` informa de un backend habilitado y en buen estado.
    - El identificador de destino está permitido por `acp.allowedAgents` cuando se establece esa lista de permitidos.
    - El comando del entorno puede iniciarse en el host del Gateway.
    - La autenticación del proveedor está disponible para ese entorno (`claude`, `codex`, `gemini`, `opencode`, `droid`, etc.).
    - El modelo seleccionado existe para ese entorno; los identificadores de modelos no son transferibles entre entornos.
    - El `cwd` solicitado existe y es accesible; de lo contrario, se puede omitir `cwd` para que el backend use su valor predeterminado.
    - El modo de permisos se ajusta al trabajo. Las sesiones no interactivas no pueden responder a solicitudes nativas de permisos, por lo que las ejecuciones de programación con muchas operaciones de escritura o ejecución suelen necesitar un perfil de permisos ACPX que pueda continuar sin interacción.

  </Accordion>
</AccordionGroup>

Las herramientas de plugins de OpenClaw y las herramientas integradas de OpenClaw **no** se exponen de forma predeterminada a los
entornos ACP. Los puentes MCP explícitos solo deben habilitarse en
[Agentes ACP: configuración](/es/tools/acp-agents-setup) cuando el entorno deba
invocar esas herramientas directamente.

## Entornos de destino compatibles

Con el backend `acpx`, se deben usar estos identificadores como destinos `/acp spawn <id>` o
`sessions_spawn({ runtime: "acp", agentId: "<id>" })`:

| Identificador del entorno | Backend habitual                                 | Notas                                                                                                    |
| ------------------------- | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| `claude`     | Adaptador ACP de Claude Code                      | Requiere autenticación de Claude Code en el host.                                                        |
| `codex`      | Adaptador ACP de Codex                            | Alternativa ACP explícita solo cuando `/codex` nativo no está disponible o se solicita ACP.    |
| `copilot`    | Adaptador ACP de GitHub Copilot                   | Requiere autenticación de la CLI o del entorno de ejecución de Copilot.                                  |
| `cursor`     | ACP de la CLI de Cursor (`cursor-agent acp`)      | Se debe reemplazar el comando acpx si una instalación local expone un punto de entrada ACP diferente.    |
| `droid`      | CLI de Factory Droid                              | Requiere autenticación de Factory/Droid o `FACTORY_API_KEY` en el entorno.                               |
| `fast-agent` | Adaptador ACP de fast-agent-mcp                   | Se descarga bajo demanda con `uvx`.                                                         |
| `gemini`     | Adaptador ACP de Gemini CLI                       | Requiere autenticación de Gemini CLI o configuración de una clave de API.                                |
| `iflow`      | CLI de iFlow                                      | La disponibilidad del adaptador y el control del modelo dependen de la CLI instalada.                    |
| `kilocode`   | CLI de Kilo Code                                  | La disponibilidad del adaptador y el control del modelo dependen de la CLI instalada.                    |
| `kimi`       | CLI de Kimi/Moonshot                              | Requiere autenticación de Kimi/Moonshot en el host.                                                      |
| `kiro`       | CLI de Kiro                                       | La disponibilidad del adaptador y el control del modelo dependen de la CLI instalada.                    |
| `mux`        | Adaptador ACP de la CLI de Mux                    | Se descarga bajo demanda con `npx`.                                                         |
| `opencode`   | Adaptador ACP de OpenCode                         | Requiere autenticación de la CLI o del proveedor de OpenCode.                                            |
| `openclaw`   | Puente del Gateway de OpenClaw mediante `openclaw acp` | Permite que un entorno compatible con ACP se comunique con una sesión del Gateway de OpenClaw.          |
| `qoder`      | CLI de Qoder                                       | La disponibilidad del adaptador y el control del modelo dependen de la CLI instalada.                    |
| `qwen`       | Qwen Code / Qwen CLI                               | Requiere autenticación compatible con Qwen en el host.                                                   |
| `trae`       | Adaptador ACP de la CLI de Trae                    | La disponibilidad del adaptador y el control del modelo dependen de la CLI instalada.                    |

`pi` (pi-acp) también está registrado en el backend acpx, pero no es un entorno de programación
en el mismo sentido que los anteriores.

Los alias personalizados de agentes acpx pueden configurarse en el propio acpx, pero la política de OpenClaw
sigue comprobando `acp.allowedAgents` y cualquier asignación de
`agents.entries.*.runtime.acp.agent` antes del despacho.

## Guía operativa

Flujo rápido de `/acp` desde el chat:

<Steps>
  <Step title="Iniciar">
    `/acp spawn claude --bind here`,
    `/acp spawn gemini --mode persistent --thread auto` o
    `/acp spawn codex --bind here` explícito.
  </Step>
  <Step title="Trabajar">
    Se continúa en la conversación o el hilo vinculados (o se especifica explícitamente la clave
    de sesión).
  </Step>
  <Step title="Comprobar el estado">
    `/acp status`
  </Step>
  <Step title="Ajustar">
    `/acp model <provider/model>`, `/acp permissions <profile>`,
    `/acp timeout <seconds>`.
  </Step>
  <Step title="Orientar">
    Sin reemplazar el contexto: `/acp steer tighten logging and continue`.
  </Step>
  <Step title="Detener">
    `/acp cancel` (turno actual) o `/acp close` (sesión y vinculaciones).
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Detalles del ciclo de vida">
    - La creación inicia o reanuda una sesión de entorno de ejecución ACP, registra los metadatos de ACP en el almacén de sesiones de OpenClaw y puede crear una tarea en segundo plano cuando la ejecución pertenece al proceso principal.
    - Las sesiones ACP que pertenecen al proceso principal se tratan como trabajo en segundo plano incluso cuando la sesión del entorno de ejecución es persistente; la finalización y la entrega entre superficies pasan por el notificador de tareas del proceso principal, en lugar de comportarse como una sesión de chat normal orientada al usuario.
    - El mantenimiento de tareas cierra las sesiones ACP de una sola ejecución, terminales o huérfanas, que pertenecen al proceso principal. Las sesiones ACP persistentes se conservan mientras exista una vinculación de conversación activa; las sesiones persistentes obsoletas sin una vinculación activa se cierran para que no puedan reanudarse silenciosamente después de que finalice la tarea propietaria o desaparezca su registro.
    - Los mensajes de seguimiento vinculados se envían directamente a la sesión ACP hasta que la vinculación se cierra, pierde el foco, se restablece o caduca.
    - Los comandos del Gateway permanecen locales. `/acp ...`, `/status` y `/unfocus` nunca se envían como texto normal de una instrucción a un entorno ACP vinculado.
    - `cancel` cancela el turno activo cuando el backend admite cancelación; no elimina la vinculación ni los metadatos de la sesión.
    - `close` finaliza la sesión ACP desde el punto de vista de OpenClaw y elimina la vinculación. Un entorno aún puede conservar su propio historial ascendente si admite la reanudación.
    - El plugin acpx limpia los árboles de procesos contenedores y adaptadores propiedad de OpenClaw después de `close`, y elimina los procesos huérfanos ACPX obsoletos propiedad de OpenClaw durante el inicio del Gateway.
    - Los procesos de trabajo inactivos del entorno de ejecución pueden limpiarse después del periodo de inactividad integrado; los metadatos almacenados de la sesión siguen disponibles para `/acp sessions`.

  </Accordion>
  <Accordion title="Reglas de enrutamiento nativo de Codex">
    Activadores en lenguaje natural que deben dirigirse al **plugin nativo de Codex**
    cuando está habilitado:

    - "Vincula este canal de Discord con Codex."
    - "Asocia este chat con el hilo de Codex `<id>`."
    - "Muestra los hilos de Codex y, después, vincula este."

    La vinculación de conversaciones nativas de Codex es la ruta predeterminada de control del chat.
    Las herramientas dinámicas de OpenClaw siguen ejecutándose mediante OpenClaw, mientras que las
    herramientas nativas de Codex, como shell/apply-patch, se ejecutan dentro de Codex. Para los
    eventos de herramientas nativas de Codex, OpenClaw inyecta un relé de hooks nativos por turno para que los hooks de los plugins
    puedan bloquear `before_tool_call`, observar `after_tool_call` y dirigir los eventos de Codex
    `PermissionRequest` mediante las aprobaciones de OpenClaw. Los hooks de Codex `Stop`
    se retransmiten a `before_agent_finalize` de OpenClaw, donde los plugins pueden solicitar
    una pasada adicional del modelo antes de que Codex finalice su respuesta. El relé se mantiene
    deliberadamente conservador: no modifica los argumentos de las herramientas nativas de Codex
    ni reescribe los registros de los hilos de Codex. Utilice ACP explícito solo cuando necesite el
    modelo de entorno de ejecución/sesión de ACP. El límite de compatibilidad integrada con Codex se
    documenta en el
    [contrato de compatibilidad v1 del entorno de Codex](/es/plugins/codex-harness-runtime#v1-support-contract).

  </Accordion>
  <Accordion title="Guía rápida para seleccionar modelo, proveedor y entorno de ejecución">
    - Referencias de modelos heredados de Codex: ruta heredada de modelos de suscripción/OAuth de Codex reparada por doctor.
    - `openai/*`: entorno de ejecución integrado del servidor de aplicaciones nativo de Codex para los turnos del agente de OpenAI.
    - `/codex ...`: control nativo de conversaciones de Codex.
    - `/acp ...` o `runtime: "acp"`: control explícito de ACP/acpx.

  </Accordion>
  <Accordion title="Activadores en lenguaje natural para el enrutamiento de ACP">
    Activadores que deben dirigirse al entorno de ejecución ACP:

    - "Ejecuta esto como una sesión ACP de una sola ejecución de Claude Code y resume el resultado."
    - "Utiliza Gemini CLI para esta tarea en un hilo y conserva los mensajes de seguimiento en ese mismo hilo."
    - "Ejecuta Codex mediante ACP en un hilo en segundo plano."

    OpenClaw selecciona `runtime: "acp"`, resuelve el entorno `agentId`, lo vincula con
    la conversación o el hilo actual cuando se admite y dirige los mensajes de seguimiento
    a esa sesión hasta que se cierre o caduque. Codex solo sigue esta ruta cuando
    ACP/acpx se solicita explícitamente o el plugin nativo de Codex no está disponible para la
    operación solicitada.

    Para `sessions_spawn`, `runtime: "acp"` solo se anuncia cuando ACP está
    habilitado, el solicitante no está aislado y hay cargado un backend de entorno de ejecución
    ACP. `acp.dispatch.enabled=false` pausa el envío automático de hilos ACP,
    pero no oculta ni bloquea las llamadas explícitas a `sessions_spawn({ runtime: "acp" })`.
    Se dirige a identificadores de entornos ACP como `codex`, `claude`, `droid`,
    `gemini` o `opencode`. No proporcione un identificador normal de agente de configuración de OpenClaw
    procedente de `agents_list` a menos que esa entrada esté configurada explícitamente con
    `agents.entries.*.runtime.type="acp"`; en caso contrario, utilice el entorno de ejecución predeterminado
    de subagentes. Cuando un agente de OpenClaw está configurado con
    `runtime.type="acp"`, OpenClaw utiliza `runtime.acp.agent` como identificador
    del entorno subyacente.

  </Accordion>
</AccordionGroup>

## ACP frente a subagentes

Utilice ACP cuando necesite un entorno de ejecución externo. Utilice el **servidor
de aplicaciones nativo de Codex** para vincular/controlar conversaciones de Codex cuando el plugin `codex`
esté habilitado. Utilice **subagentes** cuando necesite ejecuciones delegadas nativas de OpenClaw.

| Área               | Sesión ACP                                  | Ejecución de subagente                     |
| ------------------ | ------------------------------------------- | ------------------------------------------ |
| Entorno de ejecución | Plugin de backend ACP (por ejemplo, acpx) | Entorno de ejecución nativo de subagentes de OpenClaw |
| Clave de sesión    | `agent:<agentId>:acp:<uuid>`                          | `agent:<agentId>:subagent:<uuid>`                         |
| Comandos principales | `/acp ...`                        | `/subagents ...`                         |
| Herramienta de creación | `sessions_spawn` con `runtime:"acp"` | `sessions_spawn` (entorno de ejecución predeterminado) |

Consulte también [Subagentes](/es/tools/subagents).

## Cómo ejecuta ACP Claude Code

Para Claude Code mediante ACP, la pila es:

1. Plano de control de sesiones ACP de OpenClaw.
2. Plugin oficial del entorno de ejecución `@openclaw/acpx`.
3. Adaptador ACP de Claude.
4. Mecanismos de entorno de ejecución/sesión del lado de Claude.

Claude mediante ACP es una **sesión de entorno** con controles ACP, reanudación de sesiones,
seguimiento de tareas en segundo plano y vinculación opcional de conversaciones/hilos.

Los backends de CLI son entornos de ejecución locales alternativos independientes y solo de texto; consulte
[Backends de CLI](/es/gateway/cli-backends).

Para los operadores, la regla práctica es:

- **¿Necesita `/acp spawn`, sesiones vinculables, controles del entorno de ejecución o trabajo persistente del entorno?** Utilice ACP.
- **¿Necesita una alternativa local sencilla de texto mediante la CLI sin procesar?** Utilice backends de CLI.

## Sesiones vinculadas

### Modelo mental

- **Superficie de chat**: donde las personas continúan hablando (canal de Discord, tema de Telegram, chat de iMessage).
- **Sesión ACP**: el estado duradero del entorno de ejecución de Codex/Claude/Gemini al que OpenClaw dirige los mensajes.
- **Hilo/tema secundario**: una superficie de mensajería adicional opcional creada únicamente por `--thread ...`.
- **Espacio de trabajo del entorno de ejecución**: la ubicación del sistema de archivos (`cwd`, copia de trabajo del repositorio, espacio de trabajo del backend) donde se ejecuta el entorno. Es independiente de la superficie de chat.

### Vinculaciones con la conversación actual

`/acp spawn <harness> --bind here` fija la conversación actual a la
sesión ACP creada: sin hilo secundario y en la misma superficie de chat. OpenClaw sigue
controlando el transporte, la autenticación, la seguridad y la entrega. Los mensajes de seguimiento de esa
conversación se dirigen a la misma sesión; `/new` y `/reset` restablecen la sesión
sin sustituirla; `/acp close` elimina la vinculación.

Ejemplos:

```text
/codex bind                                              # vinculación nativa de Codex; dirige aquí los mensajes futuros
/codex model gpt-5.4                                     # ajusta el hilo nativo de Codex vinculado
/codex stop                                              # controla el turno nativo activo de Codex
/acp spawn codex --bind here                             # alternativa ACP explícita para Codex
/acp spawn codex --thread auto                           # puede crear un hilo/tema secundario y vincularlo allí
/acp spawn codex --bind here --cwd /workspace/repo       # misma vinculación de chat; Codex se ejecuta en /workspace/repo
```

<AccordionGroup>
  <Accordion title="Reglas de vinculación y exclusividad">
    - `--bind here` y `--thread ...` son mutuamente excluyentes.
    - `--bind here` solo funciona en canales que anuncian la vinculación con la conversación actual; en caso contrario, OpenClaw devuelve un mensaje claro que indica la falta de compatibilidad. Las vinculaciones persisten tras reiniciar el Gateway.
    - En Discord, `spawnSessions` controla la creación de hilos secundarios para `--thread auto|here`, no para `--bind here`.
    - Si crea una sesión para otro agente ACP sin `--cwd`, OpenClaw hereda de forma predeterminada el espacio de trabajo del **agente de destino**. Las rutas heredadas que faltan (`ENOENT`/`ENOTDIR`) recurren al valor predeterminado del backend; otros errores de acceso (por ejemplo, `EACCES`) se muestran como errores de creación.
    - Los comandos de administración del Gateway permanecen locales en las conversaciones vinculadas: OpenClaw gestiona los comandos `/acp ...` incluso cuando el texto normal de seguimiento se dirige a la sesión ACP vinculada; `/status` y `/unfocus` también permanecen locales siempre que la gestión de comandos esté habilitada para esa superficie.

  </Accordion>
  <Accordion title="Sesiones vinculadas a hilos">
    Cuando las vinculaciones de hilos están habilitadas para un adaptador de canal:

    - OpenClaw vincula un hilo con una sesión ACP de destino.
    - Los mensajes de seguimiento de ese hilo se dirigen a la sesión ACP vinculada.
    - La salida de ACP se entrega de vuelta al mismo hilo.
    - La pérdida de foco, el cierre, el archivado, el tiempo de espera por inactividad o la caducidad por antigüedad máxima eliminan la vinculación.
    - `/acp close`, `/acp cancel`, `/acp status`, `/status` y `/unfocus` son comandos del Gateway, no instrucciones para el entorno ACP.

    Indicadores de funcionalidad necesarios para ACP vinculado a hilos:

    - `acp.enabled=true`
    - `acp.dispatch.enabled` está activado de forma predeterminada (establezca `false` para pausar el envío automático de hilos ACP; las llamadas explícitas a `sessions_spawn({ runtime: "acp" })` siguen funcionando).
    - Creación de sesiones de hilo habilitada en el adaptador de canal (valor predeterminado: `true`):
      - Discord/Telegram: `session.threadBindings.spawnSessions=true`

    La compatibilidad con la vinculación de hilos depende de cada adaptador. Si el adaptador de canal activo
    no admite vinculaciones de hilos, OpenClaw devuelve un mensaje claro
    que indica que no se admite o no está disponible.

  </Accordion>
  <Accordion title="Canales compatibles con hilos">
    - Cualquier adaptador de canal que exponga la capacidad de vinculación de sesiones/hilos.
    - Compatibilidad integrada actual: hilos/canales de **Discord** y temas de **Telegram** (temas de foro en grupos/supergrupos y temas de mensajes directos).
    - Los canales de plugins pueden añadir compatibilidad mediante la misma interfaz de vinculación.

  </Accordion>
</AccordionGroup>

## Vinculaciones persistentes de canales

Para flujos de trabajo no efímeros, configure vinculaciones ACP persistentes en las entradas
`bindings[]` del nivel superior.

### Modelo de vinculación

<ParamField path="bindings[].type" type='"acp"'>
  Marca una vinculación persistente de conversación ACP.
</ParamField>
<ParamField path="bindings[].match" type="object">
  Identifica la conversación de destino. Formas específicas de cada canal:

- **Canal/hilo de Discord:** `match.channel="discord"` + `match.peer.id="<channelOrThreadId>"`
- **Canal/MD de Slack:** `match.channel="slack"` + `match.peer.id="<channelId|channel:<channelId>|#<channelId>|userId|user:<userId>|slack:<userId>|<@userId>>"`. Se prefieren los identificadores estables de Slack; las vinculaciones de canales también coinciden con las respuestas dentro de los hilos de ese canal.
- **Tema de foro de Telegram:** `match.channel="telegram"` + `match.peer.id="<chatId>:topic:<topicId>"`
- **MD/grupo de WhatsApp:** `match.channel="whatsapp"` + `match.peer.id="<E.164|group JID>"`. Se deben usar números E.164 como `+15555550123` para chats directos y JID de grupos de WhatsApp como `120363424282127706@g.us` para grupos.
- **MD/grupo de iMessage:** `match.channel="imessage"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`. Se prefiere `chat_id:*` para vinculaciones estables de grupos.

</ParamField>
<ParamField path="bindings[].agentId" type="string">
  El identificador del agente de OpenClaw propietario.
</ParamField>
<ParamField path="bindings[].acp.mode" type='"persistent" | "oneshot"'>
  Anulación opcional de ACP.
</ParamField>
<ParamField path="bindings[].acp.label" type="string">
  Etiqueta opcional visible para el operador.
</ParamField>
<ParamField path="bindings[].acp.cwd" type="string">
  Directorio de trabajo opcional del entorno de ejecución.
</ParamField>
<ParamField path="bindings[].acp.backend" type="string">
  Anulación opcional del backend.
</ParamField>

### Valores predeterminados del entorno de ejecución por agente

Se debe usar `agents.entries.*.runtime` para definir los valores predeterminados de ACP una vez por agente:

- `agents.entries.*.runtime.type="acp"`
- `agents.entries.*.runtime.acp.agent` (identificador del arnés, p. ej., `codex` o `claude`)
- `agents.entries.*.runtime.acp.backend`
- `agents.entries.*.runtime.acp.mode`
- `agents.entries.*.runtime.acp.cwd`

**Precedencia de anulaciones para sesiones vinculadas de ACP:**

1. `bindings[].acp.*`
2. `agents.entries.*.runtime.acp.*`
3. Valores predeterminados globales de ACP (p. ej., `acp.backend`)

### Ejemplo

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
      {
        id: "claude",
        runtime: {
          type: "acp",
          acp: { agent: "claude", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
    {
      type: "acp",
      agentId: "claude",
      match: {
        channel: "telegram",
        accountId: "default",
        peer: { kind: "group", id: "-1001234567890:topic:42" },
      },
      acp: { cwd: "/workspace/repo-b" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "discord", accountId: "default" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "telegram", accountId: "default" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": { requireMention: false },
          },
        },
      },
    },
    telegram: {
      groups: {
        "-1001234567890": {
          topics: { "42": { requireMention: false } },
        },
      },
    },
  },
}
```

### Comportamiento

- OpenClaw garantiza que la sesión de ACP configurada exista después de la admisión específica del canal y antes de usarla.
- Los mensajes de ese canal, tema o chat se enrutan a la sesión de ACP configurada.
- Las vinculaciones de ACP configuradas controlan la ruta de su sesión. La distribución de difusión del canal no sustituye la sesión de ACP configurada para una vinculación coincidente.
- En las conversaciones vinculadas, `/new` y `/reset` restablecen en el mismo lugar la misma clave de sesión de ACP.
- Las vinculaciones temporales del entorno de ejecución (por ejemplo, las creadas por flujos de enfoque de hilos) siguen aplicándose cuando están presentes.
- Para generaciones de ACP entre agentes sin un `cwd` explícito, OpenClaw hereda el espacio de trabajo del agente de destino de la configuración del agente.
- Si faltan las rutas heredadas del espacio de trabajo, se utiliza el cwd predeterminado del backend; los errores de acceso a rutas existentes se presentan como errores de generación.

## Iniciar sesiones de ACP

Hay dos formas de iniciar una sesión de ACP:

<Tabs>
  <Tab title="Desde sessions_spawn">
    Se debe usar `runtime: "acp"` para iniciar una sesión de ACP desde un turno del agente o una
    llamada a una herramienta.

    ```json
    {
      "task": "Abrir el repositorio y resumir las pruebas que fallan",
      "runtime": "acp",
      "agentId": "codex",
      "thread": true,
      "mode": "session"
    }
    ```

    <Note>
    `runtime` tiene como valor predeterminado `subagent`, por lo que se debe establecer `runtime: "acp"` explícitamente para
    las sesiones de ACP. Si se omite `agentId`, OpenClaw usa `acp.defaultAgent`
    cuando está configurado. `mode: "session"` requiere `thread: true` para mantener una
    conversación vinculada persistente.
    </Note>

  </Tab>
  <Tab title="Desde el comando /acp">
    Se debe usar `/acp spawn` para el control explícito por parte del operador desde el chat.

    ```text
    /acp spawn codex --mode persistent --thread auto
    /acp spawn codex --mode oneshot --thread off
    /acp spawn codex --bind here
    /acp spawn codex --thread here
    ```

    Opciones principales:

    - `--mode persistent|oneshot`
    - `--bind here|off`
    - `--thread auto|here|off`
    - `--cwd <absolute-path>`
    - `--label <name>`

    Véase [Comandos de barra diagonal](/es/tools/slash-commands).

  </Tab>
</Tabs>

### Parámetros de `sessions_spawn`

<ParamField path="task" type="string" required>
  Solicitud inicial enviada a la sesión de ACP.
</ParamField>
<ParamField path="runtime" type='"acp"' required>
  Debe ser `"acp"` para las sesiones de ACP.
</ParamField>
<ParamField path="agentId" type="string">
  Identificador del arnés de destino de ACP. Si se ha establecido, se utiliza `acp.defaultAgent` como alternativa.
</ParamField>
<ParamField path="thread" type="boolean" default="false">
  Solicita el flujo de vinculación de hilos cuando sea compatible.
</ParamField>
<ParamField path="mode" type='"run" | "session"' default="run">
  `"run"` es de una sola ejecución; `"session"` es persistente. Si se especifica `thread: true` y
  se omite `mode`, OpenClaw puede utilizar de forma predeterminada el comportamiento persistente según la
  ruta del entorno de ejecución. `mode: "session"` requiere `thread: true`.
</ParamField>
<ParamField path="cwd" type="string">
  Directorio de trabajo solicitado del entorno de ejecución (validado por la política del backend o del entorno de ejecución).
  Si se omite, la generación de ACP hereda el espacio de trabajo del agente de destino cuando está configurado;
  si faltan las rutas heredadas, se utilizan los valores predeterminados del backend, mientras que los errores
  de acceso reales se devuelven.
</ParamField>
<ParamField path="label" type="string">
  Etiqueta visible para el operador que se utiliza en el texto de la sesión o del banner.
</ParamField>
<ParamField path="resumeSessionId" type="string">
  Reanuda una sesión de ACP existente en lugar de crear una nueva. El agente
  reproduce su historial de conversación mediante `session/load`. Requiere
  `runtime: "acp"`.
</ParamField>
<ParamField path="streamTo" type='"parent"'>
  `"parent"` transmite los resúmenes del progreso de la ejecución inicial de ACP a la sesión
  solicitante como eventos del sistema. OpenClaw registra el historial completo de la retransmisión en el
  estado SQLite del agente secundario y lo elimina junto con la sesión secundaria. De forma predeterminada,
  los flujos de progreso del padre muestran los comentarios del asistente y el progreso de estado de ACP, salvo que
  `streaming.progress.commentary=false`. Discord también usa de forma predeterminada el
  modo de progreso para las vistas previas del padre cuando no se configura ningún modo de transmisión. El progreso
  de estado sigue respetando `acp.stream.tagVisibility`, por lo que las etiquetas como `plan`
  permanecen ocultas salvo que se habiliten explícitamente.
</ParamField>

Las ejecuciones `sessions_spawn` de ACP usan `agents.defaults.subagents.runTimeoutSeconds`
como límite predeterminado de turnos secundarios. La herramienta no acepta anulaciones de
tiempo de espera por llamada (`runTimeoutSeconds`/`timeoutSeconds` se rechazan con un
error que indica que debe configurarse el valor predeterminado).

<ParamField path="model" type="string">
  Anulación explícita del modelo para la sesión secundaria de ACP. Las generaciones de ACP de Codex
  normalizan las referencias de OpenAI, como `openai/gpt-5.4`, a la configuración de inicio de ACP de Codex
  antes de `session/new`; las formas con barra diagonal, como `openai/gpt-5.4/high`, también establecen
  el esfuerzo de razonamiento de ACP de Codex. Cuando se omite, `sessions_spawn({ runtime: "acp" })`
  usa los valores predeterminados existentes del modelo de subagente (`agents.defaults.subagents.model` o
  `agents.entries.*.subagents.model`) cuando están configurados; de lo contrario, permite que el
  arnés de ACP use su propio modelo predeterminado. Los demás arneses deben anunciar la capacidad ACP
  `models` y ser compatibles con `session/set_model`; de lo contrario, OpenClaw/acpx falla
  claramente en lugar de utilizar silenciosamente el valor predeterminado del agente de destino.
</ParamField>
<ParamField path="thinking" type="string">
  Esfuerzo explícito de pensamiento o razonamiento. Para ACP de Codex, `minimal` se asigna a un esfuerzo
  bajo, `low`/`medium`/`high`/`xhigh` se asignan directamente y `off` omite la
  anulación del esfuerzo de razonamiento durante el inicio. Cuando se omite, las generaciones de ACP usan los valores
  predeterminados existentes de pensamiento del subagente y el valor
  `agents.defaults.models["provider/model"].params.thinking` por modelo para el modelo
  seleccionado.
</ParamField>

## Modos de vinculación y de hilo de la generación

<Tabs>
  <Tab title="--bind here|off">
    | Modo   | Comportamiento                                                               |
    | ------ | ----------------------------------------------------------------------- |
    | `here` | Vincula en el mismo lugar la conversación activa actual; falla si no hay ninguna activa. |
    | `off`  | No crea una vinculación con la conversación actual.                          |

    Notas:

    - `--bind here` es la ruta más sencilla para que un operador «haga que este canal o chat esté respaldado por Codex».
    - `--bind here` no crea un hilo secundario.
    - `--bind here` solo está disponible en canales que admiten la vinculación con la conversación actual.
    - `--bind` y `--thread` no pueden combinarse en la misma llamada a `/acp spawn`.

  </Tab>
  <Tab title="--thread auto|here|off">
    | Modo   | Comportamiento                                                                                            |
    | ------ | ------------------------------------------------------------------------------------------------- |
    | `auto` | En un hilo activo: vincula ese hilo. Fuera de un hilo: crea o vincula un hilo secundario cuando sea compatible. |
    | `here` | Requiere un hilo activo actual; falla si no se está en uno.                                                  |
    | `off`  | Sin vinculación. La sesión se inicia sin vincular.                                                                 |

    Notas:

    - En superficies de vinculación sin hilos, el comportamiento predeterminado equivale en la práctica a `off`.
    - La generación vinculada a hilos requiere que la política del canal la admita:
      - Discord/Telegram: `session.threadBindings.spawnSessions=true`
    - Se debe usar `--bind here` cuando se quiera fijar la conversación actual sin crear un hilo secundario.

  </Tab>
</Tabs>

## Modelo de entrega

Las sesiones de ACP pueden ser espacios de trabajo interactivos o trabajo en segundo plano
controlado por el padre. La ruta de entrega depende de esa modalidad.

<AccordionGroup>
  <Accordion title="Sesiones interactivas de ACP">
    Las sesiones interactivas están destinadas a mantener la conversación en una superficie de chat visible:

    - `/acp spawn ... --bind here` vincula la conversación actual a la sesión de ACP.
    - `/acp spawn ... --thread ...` vincula un hilo o tema del canal a la sesión de ACP.
    - Las vinculaciones `bindings[].type="acp"` persistentes configuradas enrutan las conversaciones coincidentes a la misma sesión de ACP.

    Los mensajes posteriores de la conversación vinculada se enrutan directamente a la sesión de ACP,
    y la salida de ACP se entrega al mismo
    canal, hilo o tema.

    Lo que OpenClaw envía al arnés:

    - Los seguimientos vinculados normales se envían como texto de prompt, además de archivos adjuntos solo cuando el arnés o el backend los admite.
    - Los comandos de administración de `/acp` y los comandos locales del Gateway se interceptan antes del envío a ACP.
    - Los eventos de finalización generados por el entorno de ejecución se materializan para cada destino. Los agentes de OpenClaw reciben el sobre de contexto de ejecución interno de OpenClaw; los arneses ACP externos reciben un prompt sin formato con el resultado secundario y la instrucción. El sobre `<<<BEGIN_OPENCLAW_INTERNAL_CONTEXT>>>` sin procesar nunca debe enviarse a arneses externos ni conservarse como texto de transcripción del usuario de ACP.
    - Las entradas de transcripción de ACP usan el texto del desencadenador visible para el usuario o el prompt de finalización sin formato. Los metadatos de eventos internos permanecen estructurados en OpenClaw siempre que sea posible y no se tratan como contenido de chat escrito por el usuario.

  </Accordion>
  <Accordion title="Sesiones ACP de ejecución única propiedad del agente principal">
    Las sesiones ACP de ejecución única iniciadas por la ejecución de otro agente son
    procesos secundarios en segundo plano, similares a los subagentes:

    - El agente principal solicita trabajo con `sessions_spawn({ runtime: "acp", mode: "run" })`.
    - El proceso secundario se ejecuta en su propia sesión de arnés ACP.
    - Los turnos del proceso secundario se ejecutan en el mismo canal en segundo plano que usan los inicios de subagentes nativos, por lo que un arnés ACP lento no bloquea el trabajo no relacionado de la sesión principal.
    - La finalización se notifica mediante la ruta de anuncio de finalización de tareas. OpenClaw convierte los metadatos internos de finalización en un prompt ACP sin formato antes de enviarlos a un arnés externo, de modo que los arneses no ven marcadores de contexto de ejecución exclusivos de OpenClaw.
    - El agente principal reformula el resultado del proceso secundario con el estilo normal del asistente cuando resulta útil una respuesta visible para el usuario.

    **No** se debe tratar esta ruta como un chat entre pares entre el agente principal y el
    proceso secundario. El proceso secundario ya dispone de un canal de finalización de vuelta al agente principal.

  </Accordion>
  <Accordion title="sessions_send y entrega A2A">
    `sessions_send` puede dirigirse a otra sesión después del inicio. Para las sesiones
    normales entre pares, OpenClaw usa una ruta de seguimiento de agente a agente (A2A)
    después de insertar el mensaje:

    - Esperar la respuesta de la sesión de destino.
    - Opcionalmente, permitir que el solicitante y el destino intercambien un número limitado de turnos de seguimiento.
    - Pedir al destino que genere un mensaje de anuncio.
    - Entregar ese anuncio al canal o hilo visible.

    Esa ruta A2A es una alternativa para los envíos entre pares en los que el remitente
    necesita un seguimiento visible. Permanece habilitada cuando una sesión no relacionada puede ver
    y enviar mensajes a un destino ACP, por ejemplo, con una configuración amplia de
    `tools.sessions.visibility`.

    OpenClaw omite el seguimiento A2A solo cuando el solicitante es el agente principal de
    su propio proceso secundario ACP de ejecución única que le pertenece. En ese caso, ejecutar A2A además
    de la finalización de la tarea puede activar al agente principal con el resultado del proceso secundario, reenviar
    la respuesta del agente principal al proceso secundario y crear un bucle de eco
    entre ambos. El resultado de `sessions_send` informa de `delivery.status="skipped"` en
    ese caso de proceso secundario propio porque la ruta de finalización ya es responsable
    del resultado.

  </Accordion>
  <Accordion title="Reanudar una sesión existente">
    Use `resumeSessionId` para continuar una sesión ACP anterior en lugar de
    comenzar de nuevo. El agente reproduce su historial de conversación mediante
    `session/load`, por lo que continúa con todo el contexto de lo ocurrido anteriormente.

    ```json
    {
      "task": "Continuar donde lo dejamos: corregir los fallos de pruebas restantes",
      "runtime": "acp",
      "agentId": "codex",
      "resumeSessionId": "<previous-session-id>"
    }
    ```

    Casos de uso habituales:

    - Transferir una sesión de Codex del portátil al teléfono: indique al agente que continúe donde se dejó.
    - Continuar mediante el agente, ahora sin interfaz, una sesión de programación iniciada interactivamente en la CLI.
    - Retomar el trabajo interrumpido por un reinicio del Gateway o un tiempo de espera por inactividad.

    Notas:

    - `resumeSessionId` solo se aplica cuando `runtime: "acp"`; el entorno de ejecución predeterminado de subagentes ignora este campo exclusivo de ACP.
    - `streamTo` solo se aplica cuando `runtime: "acp"`; el entorno de ejecución predeterminado de subagentes ignora este campo exclusivo de ACP.
    - `resumeSessionId` es un identificador de reanudación de ACP/arnés local al host, no una clave de sesión de canal de OpenClaw; OpenClaw sigue comprobando la política de inicio de ACP y la política del agente de destino antes del envío, mientras que el backend o el arnés de ACP controla la autorización para cargar ese identificador ascendente.
    - `resumeSessionId` restaura el historial de conversación de ACP ascendente; `thread` y `mode` siguen aplicándose con normalidad a la nueva sesión de OpenClaw que se está creando, por lo que `mode: "session"` sigue requiriendo `thread: true`.
    - El agente de destino debe admitir `session/load` (Codex y Claude Code lo admiten).
    - Si no se encuentra el identificador de sesión, el inicio falla con un error claro; no se recurre silenciosamente a una sesión nueva.

  </Accordion>
  <Accordion title="Prueba de humo posterior al despliegue">
    Después de desplegar un Gateway, ejecute una comprobación integral real en lugar de confiar
    en las pruebas unitarias:

    1. Verificar la versión y el commit del Gateway desplegado en el host de destino.
    2. Abrir una sesión puente ACPX temporal con un agente real.
    3. Pedir a ese agente que invoque `sessions_spawn` con `runtime: "acp"`, `agentId: "codex"`, `mode: "run"` y la tarea `Reply with exactly LIVE-ACP-SPAWN-OK`.
    4. Verificar `accepted=yes`, un `childSessionKey` real y la ausencia de errores de validación.
    5. Limpiar la sesión puente temporal.

    Mantenga la puerta de control en `mode: "run"` y omita `streamTo: "parent"`;
    las rutas `mode: "session"` vinculadas a hilos y de retransmisión de flujos son pasadas de integración
    más completas e independientes.

  </Accordion>
</AccordionGroup>

## Compatibilidad con el entorno aislado

Actualmente, las sesiones ACP se ejecutan en el entorno de ejecución del host, **no** dentro del entorno
aislado de OpenClaw.

<Warning>
**Límite de seguridad:**

- El arnés externo puede leer y escribir según sus propios permisos de la CLI y el `cwd` seleccionado.
- La política del entorno aislado de OpenClaw **no** engloba la ejecución del arnés ACP.
- OpenClaw sigue aplicando las puertas de funciones de ACP, los agentes permitidos, la propiedad de las sesiones, las vinculaciones de canales y la política de entrega del Gateway.
- Use `runtime: "subagent"` para trabajos nativos de OpenClaw sujetos al entorno aislado.

</Warning>

Limitaciones actuales:

- Si la sesión solicitante está aislada, los inicios de ACP se bloquean tanto para `sessions_spawn({ runtime: "acp" })` como para `/acp spawn`.
- `sessions_spawn` con `runtime: "acp"` no admite `sandbox: "require"`.

## Resolución del destino de sesión

La mayoría de las acciones de `/acp` aceptan un destino de sesión opcional (`session-key`,
`session-id` o `session-label`).

**Orden de resolución:**

1. El argumento de destino explícito (o `--session` para `/acp steer`)
   - prueba primero la clave
   - después, el identificador de sesión con formato UUID
   - y, por último, la etiqueta
2. La vinculación del hilo actual (si esta conversación o hilo está vinculado a una sesión ACP).
3. La sesión solicitante actual como alternativa.

Tanto las vinculaciones de la conversación actual como las del hilo participan en el paso 2.

Si no se resuelve ningún destino, OpenClaw devuelve un error claro
(`Unable to resolve session target: ...`).

## Controles de ACP

| Comando              | Función                                                    | Ejemplo                                                       |
| -------------------- | --------------------------------------------------------- | ------------------------------------------------------------- |
| `/acp spawn`         | Crea una sesión ACP; vinculación actual o de hilo opcional. | `/acp spawn codex --bind here --cwd /repo`                    |
| `/acp cancel`        | Cancela el turno en curso de la sesión de destino.         | `/acp cancel agent:codex:acp:<uuid>`                          |
| `/acp steer`         | Envía instrucciones de orientación a la sesión en ejecución. | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close`         | Cierra la sesión y desvincula los destinos de hilo.        | `/acp close`                                                  |
| `/acp status`        | Muestra el backend, modo, estado, opciones de ejecución y capacidades. | `/acp status`                                                 |
| `/acp set-mode`      | Establece el modo de ejecución de la sesión de destino.    | `/acp set-mode plan`                                          |
| `/acp set`           | Escribe una opción genérica de configuración del entorno de ejecución. | `/acp set model openai/gpt-5.4`                               |
| `/acp cwd`           | Establece una sustitución del directorio de trabajo del entorno de ejecución. | `/acp cwd /Users/user/Projects/repo`                          |
| `/acp permissions`   | Establece el perfil de la política de aprobación.          | `/acp permissions strict`                                     |
| `/acp timeout`       | Establece el tiempo de espera del entorno de ejecución (segundos). | `/acp timeout 120`                                            |
| `/acp model`         | Establece una sustitución del modelo del entorno de ejecución. | `/acp model anthropic/claude-opus-4-6`                        |
| `/acp reset-options` | Elimina las sustituciones de opciones de ejecución de la sesión. | `/acp reset-options`                                          |
| `/acp sessions`      | Enumera las sesiones ACP recientes del almacén.            | `/acp sessions`                                               |
| `/acp doctor`        | Muestra el estado del backend, sus capacidades y las correcciones aplicables. | `/acp doctor`                                                 |
| `/acp install`       | Muestra pasos deterministas de instalación y habilitación. | `/acp install`                                                |

Los controles del entorno de ejecución (`spawn`, `cancel`, `steer`, `close`, `status`, `set-mode`,
`set`, `cwd`, `permissions`, `timeout`, `model` y `reset-options`) requieren
la identidad del propietario desde canales externos y `operator.admin` desde clientes
internos del Gateway. Los remitentes autorizados que no sean propietarios aún pueden usar `sessions`,
`doctor`, `install` y `help`. Para los remitentes que no sean propietarios, `/acp sessions`
solo enumera la sesión vinculada actual o la sesión solicitante; la identidad del propietario y
los clientes `operator.admin` ven todas las sesiones recientes.

`/acp status` muestra las opciones efectivas del entorno de ejecución, además de los identificadores
de sesión del nivel del entorno de ejecución y del nivel del backend. Los errores de controles no compatibles se muestran
claramente cuando un backend carece de una capacidad. Los comandos que aceptan tokens de destino
(`session-key`, `session-id` o `session-label`) los resuelven mediante el descubrimiento de sesiones
del Gateway, incluidas las raíces `session.store` personalizadas de cada agente. `/acp sessions`
no acepta ningún token de destino.

### Asignación de opciones del entorno de ejecución

`/acp` dispone de comandos prácticos y un establecedor genérico. Operaciones equivalentes:

| Comando                      | Se asigna a                              | Notas                                                                                                                                                                                                      |
| ---------------------------- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/acp model <id>`            | clave de configuración de ejecución `model`           | Para Codex ACP, OpenClaw normaliza `openai/<model>` al id. de modelo del adaptador y asigna sufijos de razonamiento con barra, como `openai/gpt-5.4/high`, a `reasoning_effort`.                                         |
| `/acp set thinking <level>`  | opción canónica `thinking`          | OpenClaw envía el equivalente anunciado por el backend cuando está presente, con preferencia por `thinking`, seguido de `effort`, `reasoning_effort` o `thought_level`. Para Codex ACP, el adaptador asigna los valores a `reasoning_effort`. |
| `/acp permissions <profile>` | opción canónica `permissionProfile` | OpenClaw envía el equivalente anunciado por el backend cuando está presente, como `approval_policy`, `permission_profile`, `permissions` o `permission_mode`.                                                       |
| `/acp timeout <seconds>`     | opción canónica `timeoutSeconds`    | OpenClaw envía el equivalente anunciado por el backend cuando está presente, como `timeout` o `timeout_seconds`.                                                                                                     |
| `/acp cwd <path>`            | anulación del directorio de trabajo de ejecución                 | Actualización directa.                                                                                                                                                                                             |
| `/acp set <key> <value>`     | genérico                              | `key=cwd` usa la ruta de anulación del directorio de trabajo.                                                                                                                                                                      |
| `/acp reset-options`         | borra todas las anulaciones de ejecución         | -                                                                                                                                                                                                          |

## Entorno de pruebas acpx, configuración de plugins y permisos

Para conocer la configuración del entorno de pruebas acpx (alias de Claude Code, Codex y Gemini CLI),
los puentes MCP de herramientas de plugins y herramientas de OpenClaw, y los modos de permisos de ACP,
consulte [Agentes ACP: configuración](/es/tools/acp-agents-setup).

## Solución de problemas

| Síntoma                                                                                   | Causa probable                                                                                                           | Solución                                                                                                                                                                      |
| ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ACP runtime backend is not configured`                                                   | El plugin del backend falta, está deshabilitado o está bloqueado por `plugins.allow`.                                                       | Instale y habilite el plugin del backend, incluya `acpx` en `plugins.allow` cuando esté configurada esa lista de permitidos y, a continuación, ejecute `/acp doctor`.                                                 |
| `ACP is disabled by policy (acp.enabled=false)`                                           | ACP está deshabilitado globalmente.                                                                                                 | Establezca `acp.enabled=true`.                                                                                                                                                  |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)`                         | El envío automático desde mensajes de hilos normales está deshabilitado.                                                               | Establezca `acp.dispatch.enabled=true` para reanudar el enrutamiento automático de hilos; las llamadas explícitas a `sessions_spawn({ runtime: "acp" })` siguen funcionando.                                      |
| `ACP agent "<id>" is not allowed by policy`                                               | El agente no está en la lista de permitidos.                                                                                                | Use un `agentId` permitido o actualice `acp.allowedAgents`.                                                                                                                     |
| `/acp doctor` informa que el backend no está listo justo después del inicio                               | El plugin del backend falta, está deshabilitado, está bloqueado por la política de permitidos/denegados o su ejecutable configurado no está disponible.        | Instale o habilite el plugin del backend, vuelva a ejecutar `/acp doctor` e inspeccione el error de instalación o de política del backend si continúa en mal estado.                                           |
| No se encuentra el comando del entorno de pruebas                                                                 | La CLI del adaptador no está instalada, falta el plugin externo o falló la obtención de `npx` durante la primera ejecución para un adaptador que no es Codex. | Ejecute `/acp doctor`, instale o precargue el adaptador en el host del Gateway, o configure explícitamente el comando del agente acpx.                                                      |
| El entorno de pruebas indica que no se encontró el modelo                                                          | El id. de modelo es válido para otro proveedor o entorno de pruebas, pero no para este destino ACP.                                                | Use un modelo incluido en la lista de ese entorno de pruebas, configure el modelo en él u omita la anulación.                                                                            |
| Error de autenticación del proveedor en el entorno de pruebas                                                        | OpenClaw funciona correctamente, pero no se ha iniciado sesión en la CLI o el proveedor de destino.                                                     | Inicie sesión o proporcione la clave de proveedor necesaria en el entorno del host del Gateway.                                                                                             |
| `Unable to resolve session target: ...`                                                   | Token de clave, id. o etiqueta incorrecto.                                                                                                | Ejecute `/acp sessions`, copie la clave o etiqueta exacta y vuelva a intentarlo.                                                                                                                        |
| `--bind here requires running /acp spawn inside an active ... conversation`               | Se usó `--bind here` sin una conversación activa que pueda vincularse.                                                            | Vaya al chat o canal de destino y vuelva a intentarlo, o use la generación sin vinculación.                                                                                                         |
| `Conversation bindings are unavailable for <channel>.`                                    | El adaptador carece de la capacidad de vinculación ACP con la conversación actual.                                                             | Use `/acp spawn ... --thread ...` cuando sea compatible, configure `bindings[]` en el nivel superior o vaya a un canal compatible.                                                     |
| `--thread here requires running /acp spawn inside an active ... thread`                   | Se usó `--thread here` fuera del contexto de un hilo.                                                                         | Vaya al hilo de destino o use `--thread auto`/`off`.                                                                                                                      |
| `Only <user-id> can rebind this channel/conversation/thread.`                             | Otro usuario es propietario del destino de vinculación activo.                                                                           | Vuelva a vincularlo como propietario o use otra conversación u otro hilo.                                                                                                               |
| `Thread bindings are unavailable for <channel>.`                                          | El adaptador carece de la capacidad de vinculación de hilos.                                                                               | Use `--thread off` o vaya a un adaptador o canal compatible.                                                                                                                 |
| `Sandboxed sessions cannot spawn ACP sessions ...`                                        | La ejecución de ACP se realiza en el host; la sesión solicitante está en un entorno aislado.                                                              | Use `runtime="subagent"` desde sesiones en entornos aislados o ejecute la generación de ACP desde una sesión que no esté en un entorno aislado.                                                                         |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...`                   | Se solicitó `sandbox="require"` para la ejecución de ACP.                                                                         | Use `runtime="subagent"` si se requiere aislamiento, o use ACP con `sandbox="inherit"` desde una sesión que no esté en un entorno aislado.                                                      |
| `Cannot apply --model ... did not advertise model support`                                | El entorno de pruebas de destino no ofrece un cambio genérico de modelos ACP.                                                        | Use un entorno de pruebas que anuncie `models`/`session/set_model` de ACP, use referencias de modelos de Codex ACP o configure el modelo directamente en el entorno de pruebas si dispone de su propio indicador de inicio. |
| Faltan metadatos de ACP para la sesión vinculada                                                    | Los metadatos de sesión de ACP están obsoletos o se eliminaron.                                                                                    | Vuelva a crearla con `/acp spawn` y, a continuación, vuelva a vincular el hilo o a asignarle el foco.                                                                                                                    |
| `PermissionPromptUnavailableError: Permission prompt unavailable in non-interactive mode` | `permissionMode` bloquea las escrituras y la ejecución en una sesión ACP no interactiva.                                                    | Establezca `plugins.entries.acpx.config.permissionMode` en `approve-all` y reinicie el Gateway. Consulte [Configuración de permisos](/es/tools/acp-agents-setup#permission-configuration). |
| La sesión ACP falla al principio con poca salida                                                | Las solicitudes de permisos están bloqueadas por `permissionMode`/`nonInteractivePermissions`.                                        | Compruebe si aparece `AcpRuntimeError` en los registros del Gateway. Para conceder permisos completos, establezca `permissionMode=approve-all`; para una degradación gradual, establezca `nonInteractivePermissions=deny`.        |
| La sesión ACP se bloquea indefinidamente después de completar el trabajo                                     | El proceso del entorno de pruebas finalizó, pero la sesión ACP no informó de su finalización.                                                    | Actualice OpenClaw; la limpieza actual de acpx elimina los procesos obsoletos del contenedor y del adaptador que pertenecen a OpenClaw al cerrar y al iniciar el Gateway.                                             |
| El entorno de pruebas ve `<<<BEGIN_OPENCLAW_INTERNAL_CONTEXT>>>`                                      | El contenedor de eventos interno se filtró a través del límite de ACP.                                                                | Actualice OpenClaw y vuelva a ejecutar el flujo de finalización; los entornos de pruebas externos solo deben recibir solicitudes de finalización en texto sin formato.                                                          |

<Note>
`Command blocked by PreToolUse hook: Native hook relay unavailable` pertenece al
retransmisor de hooks nativo de Codex, no a ACP/acpx. En un chat de Codex vinculado, inicie una
sesión nueva con `/new` o `/reset`; si funciona una vez y vuelve a aparecer en
la siguiente llamada de herramienta nativa, reinicie el servidor de aplicaciones de Codex o el Gateway de OpenClaw
en lugar de repetir `/new`. Consulte
[Solución de problemas del entorno de pruebas de Codex](/es/plugins/codex-harness#troubleshooting).
</Note>

## Temas relacionados

- [Agentes ACP: configuración](/es/tools/acp-agents-setup)
- [Envío a agente](/es/tools/agent-send)
- [Backends de CLI](/es/gateway/cli-backends)
- [Arnés de Codex](/es/plugins/codex-harness)
- [Entorno de ejecución del arnés de Codex](/es/plugins/codex-harness-runtime)
- [Herramientas de sandbox multiagente](/es/tools/multi-agent-sandbox-tools)
- [`openclaw acp` (modo puente)](/es/cli/acp)
- [Subagentes](/es/tools/subagents)
