---
read_when:
    - Configurar la política de `tools.*`, las listas de permitidos o las funciones experimentales
    - Registrar proveedores personalizados o sobrescribir las URL base
    - Configuración de endpoints autoalojados compatibles con OpenAI
sidebarTitle: Tools and custom providers
summary: Configuración de herramientas (política, opciones experimentales, herramientas respaldadas por proveedores) y configuración personalizada del proveedor y la URL base
title: 'Configuración: herramientas y proveedores personalizados'
x-i18n:
    generated_at: "2026-07-26T04:40:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2010a2e48e8f4c8d0049e5c707bb8286e291a92312baac94301a7b5a674583c1
    source_path: gateway/config-tools.md
    workflow: 16
---

`tools.*` claves de configuración y configuración personalizada del proveedor o de la URL base. Para agentes, canales y otras claves de configuración de nivel superior, consulte la [referencia de configuración](/es/gateway/configuration-reference).

## Herramientas

### Perfiles de herramientas

`tools.profile` establece una lista base de permitidos antes de `tools.allow`/`tools.deny`:

<Note>
La incorporación local asigna de forma predeterminada `tools.profile: "coding"` a las nuevas configuraciones locales cuando no se especifica (se conservan los perfiles explícitos existentes).
</Note>

| Perfil      | Incluye                                                                                                                                                                                                                                                 |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | solo `session_status`                                                                                                                                                                                                                                   |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`, `image`, `image_generate`, `music_generate`, `video_generate`                |
| `messaging` | `group:messaging`, `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `ask_user` |
| `full`      | Sin restricciones (igual que si no se especifica)                                                                                                                                                                                                       |

`coding` y `messaging` también permiten implícitamente `bundle-mcp` (servidores MCP configurados).

### Grupos de herramientas

| Grupo              | Herramientas                                                                                                                                                                                                                                            |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash` se acepta como alias de `exec`)                                                                                                                                                                        |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                                                                                                                                                 |
| `group:sessions`   | `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `spawn_task`, `dismiss_task` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                                                                                                                                                          |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                                                                                                                                                  |
| `group:ui`         | `browser`, `screen`, `terminal`, `canvas`, `show_widget`                                                                                                                                                                                               |
| `group:automation` | `heartbeat_respond`, `cron`, `gateway`                                                                                                                                                                                                                 |
| `group:messaging`  | `message`                                                                                                                                                                                                                                              |
| `group:nodes`      | `nodes`, `computer`                                                                                                                                                                                                                                    |
| `group:agents`     | `agents_list`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`                                                                                                                                                   |
| `group:media`      | `image`, `image_generate`, `music_generate`, `video_generate`, `tts`                                                                                                                                                                                   |
| `group:openclaw`   | Todas las herramientas integradas anteriores excepto `read`/`write`/`edit`/`apply_patch`/`exec`/`process`/`canvas` (excluye las herramientas de plugins)                                                                                                                                  |
| `group:plugins`    | Herramientas propiedad de los plugins cargados, incluidos los servidores MCP configurados expuestos mediante `bundle-mcp`                                                                                                                                                           |

`spawn_task` permite que un agente de programación proponga trabajo de seguimiento confirmado sin iniciarlo. La interfaz de control muestra el título y el resumen como un elemento accionable; una TUI respaldada por el Gateway muestra una solicitud interactiva equivalente. Al aceptar cualquiera de ellos, se crea una nueva sesión de árbol de trabajo administrado y se envía allí la solicitud completa mientras continúa el turno actual. `dismiss_task` retira una sugerencia aún pendiente mediante el `task_id` efímero devuelto por `spawn_task`.

Las herramientas solo se ofrecen cuando la superficie del operador que inicia la acción puede recibir y procesar eventos de sugerencia de tareas del Gateway. Las sesiones de canales y las sesiones TUI locales o integradas no los reciben; los transportes de canales necesitan una acción de tarea tipada y portable antes de poder exponer este flujo de forma segura. Las sugerencias son locales al proceso y desaparecen cuando se reinicia el Gateway. Ambas herramientas permanecen en el perfil `coding` y en `group:sessions`, por lo que la política normal de `tools.allow` y `tools.deny` las configura automáticamente cuando la superficie las admite.

### Herramientas de MCP y plugins dentro de la política de herramientas del entorno aislado

Los servidores MCP configurados se exponen como herramientas propiedad de plugins bajo el identificador de plugin `bundle-mcp`. Los perfiles normales de herramientas pueden permitirlas, pero `tools.sandbox.tools` es una restricción adicional para las sesiones en entornos aislados. Si el modo del entorno aislado es `"all"` o `"non-main"`, incluya una de estas entradas en la lista de herramientas permitidas del entorno aislado cuando deban estar visibles las herramientas de MCP o plugins:

- `bundle-mcp` para servidores MCP administrados por OpenClaw desde `mcp.servers`
- el identificador del plugin para un plugin nativo específico
- `group:plugins` para todas las herramientas propiedad de plugins cargados
- nombres exactos de herramientas del servidor MCP o patrones globales de servidores como `outlook__send_mail` o `outlook__*` cuando solo se desea un servidor

Los patrones globales de servidores utilizan el prefijo de servidor MCP seguro para el proveedor, que no coincide necesariamente con la clave `mcp.servers` sin procesar. Los caracteres que no sean `[A-Za-z0-9_-]` se convierten en `-`, los nombres que no comienzan por una letra reciben el prefijo `mcp-`, y los prefijos largos o duplicados pueden truncarse o recibir un sufijo; por ejemplo, `mcp.servers["Outlook Graph"]` utiliza un patrón global como `outlook-graph__*`.

```json5
{
  agents: { defaults: { sandbox: { mode: "all" } } },
  mcp: {
    servers: {
      outlook: { command: "node", args: ["./outlook-mcp.js"] },
    },
  },
  tools: {
    sandbox: {
      tools: {
        alsoAllow: ["web_search", "web_fetch", "memory_search", "memory_get", "bundle-mcp"],
      },
    },
  },
}
```

Sin esa entrada en la capa del entorno aislado, el servidor MCP puede cargarse correctamente aunque sus herramientas se filtren antes de la solicitud al proveedor. Use `openclaw doctor` para detectar esta configuración en los servidores administrados por OpenClaw en `mcp.servers`. Los servidores MCP cargados desde manifiestos de plugins integrados o desde `.mcp.json` de Claude utilizan la misma restricción del entorno aislado, pero este diagnóstico todavía no enumera esas fuentes; utilice las mismas entradas de la lista de permitidos si sus herramientas desaparecen en turnos ejecutados en entornos aislados.

### `tools.codeMode`

`tools.codeMode` habilita la superficie genérica del modo de código de OpenClaw. Cuando se habilita
para una ejecución con herramientas, las herramientas normales de OpenClaw pasan a estar detrás del puente de catálogo `tools.*`
dentro del entorno aislado, y las herramientas MCP están disponibles mediante el espacio de nombres `MCP`
generado. El modelo normalmente ve `exec` y `wait`; herramientas como `computer`,
cuyos resultados estructurados no pueden atravesar el puente exclusivo de JSON, permanecen directas.

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

También se acepta la forma abreviada:

```json5
{
  tools: { codeMode: true },
}
```

Las declaraciones MCP se exponen mediante la superficie de archivos de API virtual de solo lectura en
el modo de código. El código invitado puede llamar a `API.list("mcp")` y
`API.read("mcp/<server>.d.ts")` para inspeccionar firmas de estilo TypeScript antes de
llamar a `MCP.<server>.<tool>()`. Consulte [Modo de código](/es/tools/code-mode) para conocer el
contrato de ejecución, los límites y los pasos de depuración.

### `tools.allow` / `tools.deny`

Política global para permitir o denegar herramientas (la denegación prevalece). No distingue entre mayúsculas y minúsculas, y admite comodines `*`. Se aplica incluso cuando el entorno aislado de Docker está desactivado.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

`write` y `apply_patch` son identificadores de herramientas distintos. `allow: ["write"]` también habilita `apply_patch` para los modelos compatibles, pero `deny: ["write"]` no deniega `apply_patch`. Para bloquear toda modificación de archivos, deniegue `group:fs` o enumere explícitamente cada herramienta de modificación:

```json5
{
  tools: { deny: ["write", "edit", "apply_patch"] },
}
```

<Note>
`allow` y `alsoAllow` no pueden establecerse a la vez en el mismo ámbito (`tools`, `tools.byProvider.<id>`, `agents.entries.*.tools`); la validación de la configuración lo rechaza. Combine las entradas de `alsoAllow` en `allow`, o elimine `allow` y utilice `profile` + `alsoAllow` en su lugar.
</Note>

### `tools.byProvider`

Restringe aún más las herramientas para proveedores o modelos específicos. Orden: perfil base → perfil del proveedor → permitir/denegar.

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.toolsBySender`

Restringe las herramientas para el solicitante que originó el turno actual. Se trata de una defensa en profundidad adicional al control de acceso del canal; los valores del remitente deben proceder del adaptador del canal, no del texto del mensaje. No autentica otro contenido del prompt del modelo; consulte [Controles con ámbito de solicitante y contexto del prompt](/es/gateway/security#requester-scoped-controls-and-prompt-context).

```json5
{
  tools: {
    toolsBySender: {
      "channel:discord:1234567890123": { alsoAllow: ["group:fs"] },
      "id:guest-user-id": { deny: ["group:runtime", "group:fs"] },
      "*": { deny: ["exec", "process", "write", "edit", "apply_patch"] },
    },
  },
}
```

Las claves usan prefijos explícitos: `channel:<channelId>:<senderId>`, `id:<senderId>`, `e164:<phone>`, `username:<handle>`, `name:<displayName>` o `"*"`. Los identificadores de canal son identificadores canónicos de OpenClaw; los alias como `teams` se normalizan a `msteams`. Las claves heredadas sin prefijo solo se aceptan como `id:`. El orden de coincidencia es canal+identificador, identificador, e164, nombre de usuario, nombre y, por último, comodín.

La configuración `agents.entries.*.tools.toolsBySender` por agente sustituye la coincidencia global del remitente cuando coincide, incluso con una política `{}` vacía.

### `tools.elevated`

Controla el acceso de ejecución elevado fuera del entorno aislado:

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123", "987654321098765432"],
      },
    },
  },
}
```

- La configuración por agente (`agents.entries.*.tools.elevated`) solo puede restringir más.
- `/elevated on|off|ask|full` almacena el estado por sesión; las directivas en línea se aplican a un solo mensaje.
- La ejecución `exec` elevada omite el aislamiento y usa la ruta de escape configurada (`gateway` de forma predeterminada, o `node` cuando el destino de ejecución es `node`).

### `tools.exec`

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      approvalRunningNoticeMs: 10000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      commandHighlighting: false,
      applyPatch: {
        enabled: true,
        allowModels: ["gpt-5.6-sol"],
      },
    },
  },
}
```

Los valores mostrados son los predeterminados, excepto `applyPatch.allowModels` (vacío/sin definir de forma predeterminada, lo que significa que cualquier modelo compatible puede usar `apply_patch`). `approvalRunningNoticeMs` emite un aviso de ejecución cuando una ejecución respaldada por aprobación tarda mucho; `0` lo desactiva.

### `tools.loopDetection`

Las comprobaciones de seguridad del bucle de herramientas están **desactivadas de forma predeterminada**. Establezca `enabled: true` para activar la detección. La configuración puede definirse globalmente en `tools.loopDetection` y sustituirse por agente en `agents.entries.*.tools.loopDetection`.

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
    },
  },
}
```

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // o variable de entorno BRAVE_API_KEY (proveedor Brave)
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // opcional; omítalo para la detección automática
        maxChars: 20000,
        maxCharsCap: 20000,
        maxResponseBytes: 750000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true,
        userAgent: "custom-ua",
      },
    },
  },
}
```

Los valores mostrados son los predeterminados, excepto `provider` y `userAgent`. `maxResponseBytes` se limita al intervalo 32000–10000000; `maxChars` se limita a `maxCharsCap` (aumente `maxCharsCap` para permitir respuestas más grandes).

### `tools.media`

Configura la comprensión de medios entrantes (imagen/audio/vídeo):

```json5
{
  tools: {
    media: {
      concurrency: 2,
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          capabilities: ["audio"],
        },
        { provider: "ollama", model: "gemma4:26b", capabilities: ["image"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["video"] },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-mini-transcribe" },
      image: { enabled: true, preferredModel: "ollama/gemma4:26b" },
      video: { enabled: true },
    },
  },
}
```

`tools.media.models` es la única lista de modelos configurada. Cada entrada declara las capacidades que gestiona. El selector opcional `preferredModel` acepta `provider/model`, un identificador de modelo, `provider:<id>` para entradas predeterminadas del proveedor o `cli:command`; las entradas coincidentes pasan al principio del orden de respaldo de esa capacidad. Los prompts, límites, ajustes de solicitud, ámbito, política de adjuntos y eco de transcripción de audio específicos de cada capacidad conservan los valores predeterminados para los modelos configurados y detectados automáticamente; una entrada de modelo puede sustituir los campos específicos del modelo.

<AccordionGroup>
  <Accordion title="Campos de entrada del modelo multimedia">
    **Entrada de proveedor** (`type: "provider"` u omitida):

    - `provider`: identificador del proveedor de API (`openai`, `anthropic`, `google`/`gemini`, `groq`, etc.)
    - `model`: sustitución del identificador del modelo
    - `profile` / `preferredProfile`: selección del perfil `auth-profiles.json`

    **Entrada de CLI** (`type: "cli"`):

    - `command`: ejecutable que se va a ejecutar
    - `args`: argumentos con plantilla (admite `{{AttachmentPath}}`, `{{AttachmentUrl}}`, `{{AttachmentContentType}}`, `{{AttachmentDir}}`, `{{AttachmentIndex}}`, `{{Prompt}}`, `{{MaxChars}}`, etc.; `openclaw doctor --fix` migra los marcadores de posición obsoletos `{input}` a `{{AttachmentPath}}`). Los alias anteriores `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` y `{{MediaDir}}` siguen disponibles durante su periodo de compatibilidad, pero están obsoletos.

    **Campos comunes:**

    - `capabilities`: lista que contiene uno o varios de `image`, `audio` y `video`.
    - `prompt`, `maxChars`, `maxBytes`, `timeoutSeconds`, `language`: sustituciones por entrada.
    - Las entradas coincidentes `timeoutSeconds` del modelo de imagen también se aplican cuando el agente llama a la herramienta explícita `image`. Para la comprensión de imágenes, este tiempo de espera se aplica a la propia solicitud y no se reduce por el trabajo de preparación anterior.
    - En caso de fallo, se recurre a la siguiente entrada.

    La autenticación del proveedor sigue el orden estándar: `auth-profiles.json` → variables de entorno → `models.providers.*.apiKey`.

  </Accordion>
</AccordionGroup>

### `tools.agentToAgent`

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `tools.sessions`

Controla a qué sesiones pueden dirigirse las herramientas de sesión (`sessions_list`, `sessions_history`, `sessions_send`).

Valor predeterminado: `tree` (sesión actual + sesiones generadas por ella, como subagentes, además de las sesiones de grupo observadas de forma ambiental para el mismo agente).

```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Ámbitos de visibilidad">
    - `self`: solo la clave de la sesión actual.
    - `tree`: sesión actual + sesiones generadas por la sesión actual (subagentes). Para las operaciones de lectura, también incluye las sesiones de grupo del mismo agente que la sesión actual observa mediante el conocimiento ambiental del grupo.
    - `agent`: cualquier sesión perteneciente al identificador del agente actual (puede incluir a otros usuarios si se ejecutan sesiones por remitente con el mismo identificador de agente).
    - `all`: cualquier sesión. Dirigirse a otros agentes sigue requiriendo `tools.agentToAgent`.
    - Restricción del entorno aislado: cuando la sesión actual está aislada y `agents.defaults.sandbox.sessionToolsVisibility="spawned"` (valor predeterminado), la visibilidad se fuerza a `tree`, incluso si `tools.sessions.visibility="all"`.
    - Cuando no es `all`, `sessions_list` incluye un campo compacto `visibility` que describe el modo efectivo y una advertencia de que algunas sesiones pueden omitirse fuera del ámbito actual.

  </Accordion>
</AccordionGroup>

Con el valor predeterminado `session.dmScope: "main"`, la actividad humana en un grupo hace que esa sesión de grupo del mismo agente sea visible de forma ambiental para la sesión principal del agente. En una configuración multiusuario, `"main"` también comparte una sesión de mensajes directos entre usuarios, por lo que cada usuario dirigido allí puede leer los grupos observados de forma ambiental, incluso mediante `memory_search` de la memoria de sesión. Use un `dmScope` por par para aislar los mensajes directos, o establezca `tools.sessions.visibility: "self"` para excluirse de las lecturas de sesiones observadas de forma ambiental.

### `tools.sessions_spawn`

Controla la compatibilidad con adjuntos en línea para `sessions_spawn`.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // activación voluntaria: establezca true para permitir adjuntos de archivos en línea
        maxTotalBytes: 5242880, // 5 MB en total entre todos los archivos
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 MB por archivo
        retainOnSessionKeep: false, // conservar los adjuntos cuando cleanup="keep"
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Notas sobre los adjuntos">
    - Los adjuntos requieren `enabled: true`.
    - Los adjuntos de subagentes se materializan en el espacio de trabajo secundario en `.openclaw/attachments/<uuid>/` con un `.manifest.json`.
    - Los adjuntos de ACP se limitan a imágenes y se reenvían en línea al entorno de ejecución de ACP una vez superados los mismos límites de cantidad de archivos, bytes por archivo y bytes totales.
    - El contenido de los adjuntos se censura automáticamente al conservar la transcripción.
    - Las entradas Base64 se validan mediante comprobaciones estrictas del alfabeto y el relleno, además de una protección de tamaño previa a la descodificación.
    - Los permisos de los archivos adjuntos de subagentes son `0700` para los directorios y `0600` para los archivos.
    - La limpieza de subagentes sigue la política `cleanup`: `delete` siempre elimina los adjuntos; `keep` solo los conserva cuando `retainOnSessionKeep: true`.

  </Accordion>
</AccordionGroup>

<a id="toolsexperimental"></a>

### `tools.experimental`

Indicadores experimentales de herramientas integradas. Desactivados de forma predeterminada, salvo que se aplique una regla de activación automática de GPT-5 con agencia estricta.

```json5
{
  tools: {
    experimental: {
      planTool: true, // activar update_plan experimental
    },
  },
}
```

- `planTool`: activa la herramienta estructurada `update_plan` para el seguimiento de trabajos no triviales de varios pasos.
- Valor predeterminado: `false`, salvo que `agents.defaults.embeddedAgent.executionContract` (o una sustitución por agente) se establezca en `"strict-agentic"` para una ejecución del proveedor `openai` con un identificador de modelo de la familia GPT-5 (esto también abarca las ejecuciones de OpenAI Codex CLI, ya que la autenticación y el enrutamiento de modelos de Codex se encuentran bajo el proveedor `openai`). Establezca `true` para forzar la activación de la herramienta fuera de ese ámbito, o `false` para mantenerla desactivada incluso en ejecuciones de GPT-5 con agencia estricta.
- Cuando está activada, el prompt del sistema también añade instrucciones de uso para que el modelo solo la utilice en trabajos sustanciales y mantenga como máximo un paso `in_progress`.

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        allowAgents: ["research"],
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 8,
        runTimeoutSeconds: 900,
        announceTimeoutMs: 120000,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model`: modelo predeterminado para los subagentes iniciados. Si se omite, los subagentes heredan el modelo del agente que realiza la llamada.
- `allowAgents`: lista de permitidos predeterminada de los identificadores de agentes de destino configurados para `sessions_spawn` cuando el agente solicitante no establece su propio `subagents.allowAgents` (`["*"]` = cualquier destino configurado; valor predeterminado: solo el mismo agente). Las entradas obsoletas cuya configuración de agente se haya eliminado son rechazadas por `sessions_spawn` y se omiten de `agents_list`; ejecute `openclaw doctor --fix` para eliminarlas.
- `maxConcurrent`: número máximo de ejecuciones simultáneas de subagentes. Valor predeterminado: `8`.
- `runTimeoutSeconds`: tiempo de espera (segundos) para `sessions_spawn` cuando quien realiza la llamada no proporciona su propia anulación. Valor predeterminado: `0` (sin tiempo de espera); el valor `900` mostrado anteriormente es un valor opcional habitual, no el valor predeterminado incorporado.
- `announceTimeoutMs`: tiempo de espera por llamada (milisegundos) para los intentos de entrega de anuncios de `agent` del Gateway. Valor predeterminado: `120000`. Los reintentos transitorios pueden hacer que la espera total del anuncio sea mayor que un único tiempo de espera configurado.
- `archiveAfterMinutes`: minutos que transcurren desde que finaliza una sesión de subagente hasta que se archiva automáticamente. Valor predeterminado: `60`; `0` desactiva el archivado automático.
- Política de herramientas por subagente: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## Proveedores personalizados y URL base

Los plugins de proveedores publican sus propias filas del catálogo de modelos. Añada proveedores personalizados mediante `models.providers` en la configuración o `~/.openclaw/agents/<agentId>/agent/models.json`.

Configurar un `baseUrl` de proveedor personalizado/local también constituye la decisión específica de confianza de red para las solicitudes HTTP del modelo: OpenClaw permite ese origen `scheme://host:port` exacto a través de la ruta de obtención protegida, sin añadir una opción de configuración independiente ni confiar en otros orígenes privados.

```json5
{
  models: {
    mode: "merge", // combinar (predeterminado) | reemplazar
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions", // openai-completions | openai-responses | anthropic-messages | google-generative-ai | etc.
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            contextTokens: 96000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Autenticación y precedencia de combinación">
    - Utilice `authHeader: true` + `headers` para necesidades de autenticación personalizadas.
    - Anule la raíz de configuración del agente con `OPENCLAW_AGENT_DIR`.
    - Precedencia de combinación para identificadores de proveedores coincidentes:
      - Prevalecen los valores no vacíos de `models.json` `baseUrl` del agente.
      - Los valores no vacíos de `apiKey` del agente prevalecen solo cuando ese proveedor no está administrado mediante SecretRef en el contexto actual de configuración/perfil de autenticación.
      - Los valores `apiKey` de proveedores administrados mediante SecretRef se actualizan desde los marcadores de origen (`ENV_VAR_NAME` para referencias de entorno, `secretref-managed` para referencias de archivo/ejecución) en lugar de conservar los secretos resueltos.
      - Los valores de cabecera de proveedores administrados mediante SecretRef se actualizan desde los marcadores de origen (`secretref-env:ENV_VAR_NAME` para referencias de entorno, `secretref-managed` para referencias de archivo/ejecución).
      - Los valores `apiKey`/`baseUrl` vacíos o ausentes del agente recurren a `models.providers` en la configuración.
      - Para `contextWindow`/`maxTokens` de modelos coincidentes: el valor explícito de configuración prevalece cuando está presente y es válido (un número finito positivo); de lo contrario, se utiliza el valor implícito/generado del catálogo.
      - El `contextTokens` de modelos coincidentes sigue la misma regla de prevalencia del valor explícito y, en su defecto, del implícito; utilícelo para limitar el contexto efectivo sin cambiar los metadatos nativos del modelo.
      - Los catálogos de plugins de proveedores se almacenan como fragmentos de catálogo generados y propiedad del plugin en el estado de plugins del agente.
      - Utilice `models.mode: "replace"` cuando desee que la configuración reescriba por completo `models.json` y omita la combinación de los fragmentos de catálogo propiedad de los plugins.
      - La persistencia de marcadores tiene como autoridad el origen: los marcadores se escriben a partir de la instantánea activa de la configuración de origen (antes de la resolución), no de los valores secretos resueltos durante la ejecución.

  </Accordion>
</AccordionGroup>

### Detalles de los campos del proveedor

<AccordionGroup>
  <Accordion title="Catálogo de nivel superior">
    - `models.mode`: comportamiento del catálogo de proveedores (`merge` o `replace`).
    - `models.providers`: mapa de proveedores personalizados indexado por identificador de proveedor.
      - Ediciones seguras: utilice `openclaw config set models.providers.<id> '<json>' --strict-json --merge` o `openclaw config set models.providers.<id>.models '<json-array>' --strict-json --merge` para actualizaciones aditivas. `config set` rechaza los reemplazos destructivos a menos que se proporcione `--replace`.

  </Accordion>
  <Accordion title="Conexión y autenticación del proveedor">
    - `models.providers.*.api`: adaptador de solicitudes (`openai-completions`, `openai-responses`, `openai-chatgpt-responses`, `anthropic-messages`, `google-generative-ai`, `google-vertex`, `github-copilot`, `bedrock-converse-stream`, `ollama`, `azure-openai-responses`). Para backends `/v1/chat/completions` autoalojados, como MLX, vLLM, SGLang y la mayoría de los servidores locales compatibles con OpenAI, utilice `openai-completions`. Un proveedor personalizado con `baseUrl` pero sin `api` utiliza de forma predeterminada `openai-completions`; establezca `openai-responses` solo cuando el backend admita `/v1/responses`.
    - `models.providers.*.apiKey`: credencial del proveedor (se recomienda la sustitución mediante SecretRef/entorno).
    - `models.providers.*.auth`: estrategia de autenticación (`api-key`, `token`, `oauth`, `aws-sdk`).
    - `models.providers.*.contextWindow`: ventana de contexto nativa predeterminada para los modelos de este proveedor cuando la entrada del modelo no establece `contextWindow`.
    - `models.providers.*.contextTokens`: límite de contexto efectivo predeterminado durante la ejecución para los modelos de este proveedor cuando la entrada del modelo no establece `contextTokens`.
    - `models.providers.*.maxTokens`: límite predeterminado de tokens de salida para los modelos de este proveedor cuando la entrada del modelo no establece `maxTokens`.
    - `models.providers.*.timeoutSeconds`: tiempo de espera opcional por proveedor, en segundos, para las solicitudes HTTP del modelo, incluida la gestión de conexión, cabeceras, cuerpo y cancelación total de la solicitud.
    - `models.providers.*.injectNumCtxForOpenAICompat`: para Ollama + `openai-completions`, inserta `options.num_ctx` en las solicitudes (valor predeterminado: `true`).
    - `models.providers.*.authHeader`: fuerza el transporte de credenciales en la cabecera `Authorization` cuando sea necesario.
    - `models.providers.*.baseUrl`: URL base de la API ascendente.
    - `models.providers.*.headers`: cabeceras estáticas adicionales para el enrutamiento por proxy/inquilino.

  </Accordion>
  <Accordion title="Anulaciones del transporte de solicitudes">
    `models.providers.*.request`: anulaciones del transporte para solicitudes HTTP de proveedores de modelos.

    - `request.headers`: cabeceras adicionales (combinadas con los valores predeterminados del proveedor). Los valores aceptan SecretRef.
    - `request.auth`: anulación de la estrategia de autenticación. Modos: `"provider-default"` (utilizar la autenticación incorporada del proveedor), `"authorization-bearer"` (con `token`), `"header"` (con `headerName`, `value` y `prefix` opcional).
    - `request.proxy`: anulación del proxy HTTP. Modos: `"env-proxy"` (utilizar las variables de entorno `HTTP_PROXY`/`HTTPS_PROXY`), `"explicit-proxy"` (con `url`). Ambos modos aceptan un subobjeto `tls` opcional.
    - `request.tls`: anulación de TLS para conexiones directas. Campos: `ca`, `cert`, `key`, `passphrase` (todos aceptan SecretRef), `serverName`, `insecureSkipVerify`.
    - `request.allowPrivateNetwork`: cuando es `true`, permite que las solicitudes HTTP del proveedor de modelos accedan a rangos privados, CGNAT o similares a través de la protección de obtención HTTP del proveedor. Las URL base de proveedores personalizados/locales ya confían en el origen configurado exacto, excepto los orígenes de metadatos/enlace local, que permanecen bloqueados sin una habilitación explícita. Establezca este valor en `false` para desactivar la confianza en el origen exacto. WebSocket utiliza el mismo `request` para las cabeceras/TLS, pero no esa protección SSRF de obtención. Valor predeterminado: `false`.

  </Accordion>
  <Accordion title="Entradas del catálogo de modelos">
    - `models.providers.*.models`: entradas explícitas del catálogo de modelos del proveedor.
    - `models.providers.*.models.*.input`: modalidades de entrada del modelo. Utilice `["text"]` para modelos que solo admiten texto y `["text", "image"]` para modelos nativos de imagen/visión. Los archivos adjuntos de imagen solo se insertan en los turnos del agente cuando el modelo seleccionado está marcado como compatible con imágenes.
    - `models.providers.*.models.*.contextWindow`: metadatos de la ventana de contexto nativa del modelo. Este valor anula el `contextWindow` del proveedor para ese modelo.
    - `models.providers.*.models.*.contextTokens`: límite opcional del contexto durante la ejecución. Este valor anula el `contextTokens` del proveedor; utilícelo cuando desee un presupuesto de contexto efectivo menor que el `contextWindow` nativo del modelo; `openclaw models list` muestra ambos valores cuando difieren.

    #### Declaraciones de capacidades de proveedores personalizados

    Los catálogos de proveedores son responsables de `compat` para las rutas de modelos incluidas y conocidas por el catálogo. No copie esas marcas en la configuración: OpenClaw utiliza la fila del catálogo cuando los valores configurados de `api` y `baseUrl` siguen identificando esa ruta. `openclaw doctor --fix` elimina las anulaciones heredadas coincidentes e informa de los valores divergentes para su revisión.

    Se sigue admitiendo un bloque `compat` para un proveedor realmente personalizado, un modelo personalizado o un modelo del catálogo dirigido a un endpoint diferente. Establezca únicamente las capacidades verificadas con ese endpoint:

    | Clave de ruta personalizada | Contrato de ejecución |
    | --- | --- |
    | `supportsStore` | Acepta el campo de solicitud `store` de OpenAI. |
    | `supportsPromptCacheKey` | Acepta claves de afinidad de sesión/caché de prompts de OpenAI. |
    | `supportsDeveloperRole` | Acepta mensajes `developer` en lugar de requerir `system`. |
    | `supportsReasoningEffort` | Acepta un control del esfuerzo de razonamiento. |
    | `supportsTemperature` | Acepta `temperature` para este modelo y adaptador. |
    | `supportsUsageInStreaming` | Emite metadatos de uso en las respuestas de transmisión. |
    | `supportsTools` | Admite llamadas estructuradas a herramientas/funciones. Establezca `false` para desactivar las herramientas. |
    | `supportsStrictMode` | Acepta esquemas estrictos de herramientas. |
    | `requiresStringContent` | Requiere que el contenido de los mensajes de Chat Completions sea una cadena simple. |
    | `strictMessageKeys` | Requiere que los mensajes salientes contengan únicamente claves aceptadas. |
    | `visibleReasoningDetailTypes` | Indica los tipos de bloques de detalles de razonamiento que pueden mostrarse de forma segura en las transcripciones. |
    | `supportedReasoningEfforts` | Enumera las etiquetas de razonamiento aceptadas por el endpoint. |
    | `reasoningEffortMap` | Asigna las etiquetas de pensamiento de OpenClaw a etiquetas específicas del endpoint. |
    | `maxTokensField` | Selecciona `max_tokens` o `max_completion_tokens`. |
    | `thinkingFormat` | Selecciona el dialecto de la carga útil de razonamiento del endpoint. |
    | `requiresToolResultName` | Requiere un nombre de herramienta en los mensajes de resultados de herramientas. |
    | `requiresAssistantAfterToolResult` | Requiere un mensaje del asistente después de los resultados de las herramientas. |
    | `requiresThinkingAsText` | Reproduce el razonamiento como texto en lugar de contenido estructurado. |
    | `requiresReasoningContentOnAssistantMessages` | Conserva `reasoning_content` al estilo de DeepSeek durante la reproducción. |
    | `toolSchemaProfile` | Selecciona un perfil de normalización de esquemas de herramientas definido por el proveedor. |
    | `unsupportedToolSchemaKeywords` | Elimina las palabras clave especificadas de JSON Schema que rechaza el endpoint. |
    | `toolCallArgumentsEncoding` | Selecciona la codificación de argumentos de llamadas a herramientas del endpoint. |
    | `requiresOpenAiAnthropicToolPayload` | Convierte las llamadas a herramientas con formato de OpenAI en cargas útiles de la familia Anthropic. |

  </Accordion>
  <Accordion title="Detección de Amazon Bedrock">
    - `plugins.entries.amazon-bedrock.config.discovery`: raíz de la configuración de detección automática de Bedrock.
    - `plugins.entries.amazon-bedrock.config.discovery.enabled`: activa o desactiva la detección implícita.
    - `plugins.entries.amazon-bedrock.config.discovery.region`: región de AWS para la detección.
    - `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: filtro opcional por id de proveedor para la detección dirigida.
    - `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: intervalo de sondeo para actualizar la detección.
    - `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: ventana de contexto de respaldo para los modelos detectados.
    - `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: cantidad máxima de tokens de salida de respaldo para los modelos detectados.

  </Accordion>
</AccordionGroup>

La incorporación interactiva de proveedores personalizados deduce la entrada de imágenes para patrones conocidos de ids de modelos de visión, incluidos GPT-4o/GPT-4.1/GPT-5+, las familias de razonamiento `o1`/`o3`/`o4`, Claude, Gemini, cualquier id con el sufijo `-vl` (Qwen-VL y similares) y familias con nombre como LLaVA, Pixtral, InternVL, Mllama, MiniCPM-V y GLM-4V; omite la pregunta adicional para las familias conocidas que solo admiten texto (Llama, DeepSeek, Mistral/Mixtral, Kimi/Moonshot, Codestral, Devstral, Phi, QwQ, CodeLlama e ids Qwen simples sin un sufijo vl/vision). Los ids de modelo desconocidos siguen solicitando información sobre la compatibilidad con imágenes. La incorporación no interactiva utiliza la misma deducción; se debe pasar `--custom-image-input` para forzar metadatos compatibles con imágenes o `--custom-text-input` para forzar metadatos que solo admitan texto.

### Ejemplos de proveedores

<AccordionGroup>
  <Accordion title="Cerebras (GLM 4.7 / GPT OSS)">
    El Plugin de proveedor externo oficial `cerebras` puede configurar esto mediante `openclaw onboard --auth-choice cerebras-api-key`. Utilice una configuración explícita del proveedor solo para reemplazar los valores predeterminados.

    ```json5
    {
      env: { CEREBRAS_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: {
            primary: "cerebras/zai-glm-4.7",
            fallbacks: ["cerebras/gpt-oss-120b"],
          },
          models: {
            "cerebras/zai-glm-4.7": { alias: "GLM 4.7 (Cerebras)" },
            "cerebras/gpt-oss-120b": { alias: "GPT OSS 120B (Cerebras)" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          cerebras: {
            baseUrl: "https://api.cerebras.ai/v1",
            apiKey: "${CEREBRAS_API_KEY}",
            api: "openai-completions",
            models: [
              { id: "zai-glm-4.7", name: "GLM 4.7 (Cerebras)" },
              { id: "gpt-oss-120b", name: "GPT OSS 120B (Cerebras)" },
            ],
          },
        },
      },
    }
    ```

    Utilice `cerebras/zai-glm-4.7` para Cerebras y `zai/glm-4.7` para conectarse directamente a Z.AI.

  </Accordion>
  <Accordion title="Kimi Coding">
    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: { "kimi/kimi-for-coding": { alias: "Kimi Code" } },
        },
      },
    }
    ```

    Proveedor integrado compatible con Anthropic. Acceso directo: `openclaw onboard --auth-choice kimi-code-api-key`.

  </Accordion>
  <Accordion title="Modelos locales (LM Studio)">
    Consulte [Modelos locales](/es/gateway/local-models). En resumen: ejecute un modelo local grande mediante la API Responses de LM Studio en hardware de alto rendimiento; mantenga combinados los modelos alojados para usarlos como respaldo.
  </Accordion>
  <Accordion title="MiniMax M3 (directo)">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "Minimax" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          minimax: {
            baseUrl: "https://api.minimax.io/anthropic",
            apiKey: "${MINIMAX_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.12, cacheWrite: 0 },
                contextWindow: 1000000,
                maxTokens: 131072,
              },
            ],
          },
        },
      },
    }
    ```

    Establezca `MINIMAX_API_KEY`. Accesos directos: `openclaw onboard --auth-choice minimax-global-api` o `openclaw onboard --auth-choice minimax-cn-api`. El catálogo de modelos utiliza M3 de forma predeterminada y también incluye las variantes M2.7. En la ruta de transmisión compatible con Anthropic, OpenClaw desactiva de forma predeterminada el razonamiento de MiniMax M2.x, a menos que se establezca explícitamente `thinking`; MiniMax-M3 (y M3.x) permanece de forma predeterminada en la ruta de razonamiento omitido/adaptativo del proveedor. `/fast on` o `params.fastMode: true` reescribe `MiniMax-M2.7` como `MiniMax-M2.7-highspeed`.

  </Accordion>
  <Accordion title="Moonshot AI (Kimi)">
    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: { "moonshot/kimi-k2.6": { alias: "Kimi K2.6" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
            ],
          },
        },
      },
    }
    ```

    Para el endpoint de China: `baseUrl: "https://api.moonshot.cn/v1"` o `openclaw onboard --auth-choice moonshot-api-key-cn`.

    Los endpoints nativos de Moonshot anuncian compatibilidad con el uso durante la transmisión en el transporte compartido `openai-completions`, y OpenClaw la determina a partir de las capacidades del endpoint, en lugar de basarse únicamente en el id del proveedor integrado.

  </Accordion>
  <Accordion title="OpenCode">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "opencode/claude-opus-4-6" },
          models: { "opencode/claude-opus-4-6": { alias: "Opus" } },
        },
      },
    }
    ```

    Establezca `OPENCODE_API_KEY` (o `OPENCODE_ZEN_API_KEY`). Utilice referencias `opencode/...` para el catálogo Zen o referencias `opencode-go/...` para el catálogo Go. Acceso directo: `openclaw onboard --auth-choice opencode-zen` o `openclaw onboard --auth-choice opencode-go`.

  </Accordion>
  <Accordion title="Synthetic (compatible con Anthropic)">
    ```json5
    {
      env: { SYNTHETIC_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
          models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          synthetic: {
            baseUrl: "https://api.synthetic.new/anthropic",
            apiKey: "${SYNTHETIC_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "hf:MiniMaxAI/MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```

    La URL base debe omitir `/v1` (el cliente de Anthropic lo añade). Acceso directo: `openclaw onboard --auth-choice synthetic-api-key`.

  </Accordion>
  <Accordion title="Z.AI (GLM-4.7)">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-4.7" },
          models: { "zai/glm-4.7": {} },
        },
      },
    }
    ```

    Establezca `ZAI_API_KEY`. Las referencias de modelos utilizan el id de proveedor canónico `zai/*`. Acceso directo: `openclaw onboard --auth-choice zai-api-key`.

    - Endpoint general: `https://api.z.ai/api/paas/v4`
    - Endpoint de programación: `https://api.z.ai/api/coding/paas/v4`
    - La opción de autenticación predeterminada `zai-api-key` comprueba la clave y detecta automáticamente a qué endpoint pertenece (si la detección no es concluyente, solicita una selección cuyo valor predeterminado es Global). También hay opciones de autenticación específicas de CN y Coding-Plan para realizar una selección explícita.
    - Para el endpoint general, defina un proveedor personalizado con el reemplazo de la URL base.

  </Accordion>
</AccordionGroup>

---

## Contenido relacionado

- [Configuración — agentes](/es/gateway/config-agents)
- [Configuración — canales](/es/gateway/config-channels)
- [Referencia de configuración](/es/gateway/configuration-reference) — otras claves de nivel superior
- [Herramientas y plugins](/es/tools)
