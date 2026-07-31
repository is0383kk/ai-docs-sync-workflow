---
read_when:
    - Necesita la semántica exacta de la configuración a nivel de campo o los valores predeterminados
    - Está validando bloques de configuración de canales, modelos, Gateway o herramientas
summary: Referencia de configuración del Gateway para las claves principales de OpenClaw, los valores predeterminados y los enlaces a referencias específicas de los subsistemas
title: Referencia de configuración
x-i18n:
    generated_at: "2026-07-26T04:38:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7135554fda444fd1b8c072af5768c53a165f7be2dcd12a7909fc7fd4bd864428
    source_path: gateway/configuration-reference.md
    workflow: 16
---

Referencia a nivel de campo para `~/.openclaw/openclaw.json`: claves, valores predeterminados y enlaces a páginas más detalladas de los subsistemas. Para obtener instrucciones de configuración orientadas a tareas, consulte [Configuración](/es/gateway/configuration). Los catálogos de comandos propiedad de canales y plugins, así como las opciones avanzadas de memoria/QMD, se encuentran en sus propias páginas, no aquí.

El formato de configuración es **JSON5** (se permiten comentarios y comas finales). Todos los campos son opcionales; OpenClaw utiliza valores predeterminados seguros cuando se omiten.

El código prevalece sobre esta página:

- `openclaw config schema` imprime el esquema JSON activo utilizado para la validación y la interfaz de control, con los metadatos de paquetes, plugins y canales integrados.
- Los agentes deben invocar la acción `gateway` de la herramienta `config.schema.lookup` para obtener un único nodo exacto del esquema delimitado por ruta antes de editar la configuración.
- `pnpm config:docs:check` / `pnpm config:docs:gen` validan el hash de referencia de este documento con respecto a la superficie actual del esquema.

Los `uiHints` del esquema también incluyen un booleano `advanced` resuelto para cada ruta.
La interfaz de control lo utiliza para mostrar primero los campos comunes y contraer los campos avanzados por
sección; la búsqueda sigue abarcando ambos niveles. Los metadatos de nivel son únicamente de presentación.
Al añadir una clave, declare su nivel en la hoja o permita que herede la declaración del
ancestro más cercano. Una ruta sin ningún ancestro declarado se considera avanzada de forma predeterminada.

Referencias específicas detalladas:

- [Referencia de configuración de memoria](/es/reference/memory-config) para `memory.search.*`, `memory.qmd.*`, `memory.citations` y la configuración de Dreaming en `plugins.entries.memory-core.config.dreaming`.
- [Comandos de barra](/es/tools/slash-commands) para el catálogo actual de comandos integrados y empaquetados.
- Páginas de los canales/plugins propietarios para las superficies de comandos específicas de cada canal.

---

## Canales

Las claves de configuración de cada canal se encuentran en [Configuración: canales](/es/gateway/config-channels): `channels.*` para Slack, Discord, Telegram, WhatsApp, Matrix, iMessage y otros canales empaquetados (autenticación, control de acceso, varias cuentas y restricción por menciones).

## Valores predeterminados de agentes, múltiples agentes, sesiones y mensajes

Consulte [Configuración: agentes](/es/gateway/config-agents) para:

- `agents.defaults.*` (espacio de trabajo, modelo, razonamiento, Heartbeat, memoria, contenido multimedia, Skills y entorno aislado)
- `multiAgent.*` (enrutamiento y vinculaciones de múltiples agentes)
- `session.*` (ciclo de vida de las sesiones, Compaction y depuración)
- `messages.*` (entrega de mensajes, TTS y renderizado de Markdown)
- `talk.*` (modo de conversación)
  - `talk.consultThinkingLevel`: anulación del nivel de razonamiento para la ejecución completa del agente de OpenClaw que respalda las consultas en tiempo real del modo de conversación de la interfaz de control
  - `talk.consultFastMode`: anulación puntual del modo rápido para las consultas en tiempo real del modo de conversación de la interfaz de control
  - `talk.speechLocale`: identificador de configuración regional BCP 47 opcional para el reconocimiento de voz del modo de conversación en Android, iOS y macOS
  - `talk.silenceTimeoutMs`: cuando no se establece, el modo de conversación conserva la ventana de pausa predeterminada de la plataforma antes de enviar la transcripción (`700 ms on macOS and Android, 900 ms on iOS`)
  - `talk.realtime.consultRouting`: mecanismo alternativo de retransmisión del Gateway para las transcripciones en tiempo real finalizadas del modo de conversación que omiten `openclaw_agent_consult`

## Herramientas y proveedores personalizados

La política de herramientas, las opciones experimentales, la configuración de herramientas respaldadas por proveedores y la configuración de
proveedores personalizados/URL base se encuentran en
[Configuración: herramientas y proveedores personalizados](/es/gateway/config-tools).

## Modelos

Las definiciones de proveedores, las listas de modelos permitidos y la configuración de proveedores personalizados se encuentran en
[Configuración: herramientas y proveedores personalizados](/es/gateway/config-tools#custom-providers-and-base-urls).
La raíz `models` también controla el comportamiento global del catálogo de modelos.

```json5
{
  models: {
    // Opcional. Valor predeterminado: true. Requiere reiniciar el Gateway cuando cambia.
    pricing: { enabled: false },
  },
}
```

- `models.mode`: comportamiento del catálogo de proveedores (`merge` o `replace`).
- `models.providers`: mapa de proveedores personalizados indexado por identificador de proveedor.
- `models.providers.*.localService`: gestor de procesos opcional bajo demanda para
  servidores de modelos locales. OpenClaw sondea el punto de conexión de estado configurado, inicia
  el `command` absoluto cuando es necesario, espera a que esté listo y, a continuación, envía la solicitud
  del modelo. Consulte [Servicios de modelos locales](/es/gateway/local-model-services).
- `models.pricing.enabled`: controla la inicialización de precios en segundo plano que
  comienza después de que los procesos auxiliares y los canales alcanzan la ruta de disponibilidad del Gateway. Cuando es `false`,
  el Gateway omite las consultas a los catálogos de precios de OpenRouter y LiteLLM; los valores
  `models.providers.*.models[].cost` configurados siguen funcionando para las estimaciones locales de costes.

## MCP

Las definiciones de servidores MCP gestionadas por OpenClaw se encuentran en `mcp.servers` y son
utilizadas por OpenClaw integrado y otros adaptadores de entorno de ejecución. Los comandos `openclaw mcp list`,
`show`, `set` y `unset` gestionan este bloque sin conectarse al
servidor de destino durante las modificaciones de configuración.

```json5
{
  mcp: {
    servers: {
      docs: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-fetch"],
      },
      remote: {
        url: "https://example.com/mcp",
        transport: "streamable-http", // streamable-http | sse
        requestTimeoutMs: 20000,
        connectionTimeoutMs: 5000,
        supportsParallelToolCalls: true,
        headers: {
          Authorization: "Bearer ${MCP_REMOTE_TOKEN}",
        },
        auth: "oauth",
        oauth: {
          scope: "docs.read",
        },
        sslVerify: true,
        clientCert: "/path/to/client.crt",
        clientKey: "/path/to/client.key",
        toolFilter: {
          include: ["search_*"],
          exclude: ["admin_*"],
        },
        // Controles opcionales de proyección del servidor de aplicaciones de Codex.
        codex: {
          agents: ["main"],
          defaultToolsApprovalMode: "approve", // auto | prompt | approve
        },
      },
    },
  },
}
```

- `mcp.servers`: definiciones con nombre de servidores MCP stdio o remotos para entornos de ejecución que
  exponen las herramientas MCP configuradas.
  Las entradas remotas utilizan `transport: "streamable-http"` o `transport: "sse"`;
  `type: "http"` es un alias nativo de la CLI que `openclaw mcp set` y
  `openclaw doctor --fix` normalizan en el campo canónico `transport`.
- `mcp.servers.<name>.enabled`: establezca `false` para conservar una definición de servidor guardada
  y excluirla al mismo tiempo de la detección MCP y la proyección de herramientas de OpenClaw integrado.
- `mcp.servers.<name>.requestTimeoutMs`: tiempo de espera de solicitudes MCP por servidor, en milisegundos.
- `mcp.servers.<name>.connectionTimeoutMs`: tiempo de espera de conexión por servidor, en milisegundos.
- `mcp.servers.<name>.supportsParallelToolCalls`: indicación opcional de simultaneidad para
  adaptadores que pueden decidir si emitir llamadas paralelas a herramientas MCP.
- `mcp.servers.<name>.auth`: establezca `"oauth"` para los servidores MCP HTTP que requieran
  OAuth. Ejecute `openclaw mcp login <name>` para almacenar los tokens en el estado de OpenClaw.
- `mcp.servers.<name>.oauth`: anulaciones opcionales del ámbito de OAuth, la URL de redirección y la URL
  de metadatos del cliente.
- `mcp.servers.<name>.sslVerify`, `clientCert`, `clientKey`: controles TLS de HTTP
  para puntos de conexión privados y TLS mutuo.
- `mcp.servers.<name>.toolFilter`: selección opcional de herramientas por servidor. `include`
  limita las herramientas MCP detectadas a los nombres coincidentes; `exclude` oculta los nombres
  coincidentes. Las entradas son nombres exactos de herramientas MCP o patrones glob simples `*`. Los servidores con
  recursos o indicaciones también generan nombres de herramientas auxiliares (`resources_list`,
  `resources_read`, `prompts_list`, `prompts_get`), y esos nombres utilizan el
  mismo filtro.
- `mcp.servers.<name>.codex`: controles opcionales de proyección del servidor de aplicaciones de Codex.
  Este bloque contiene metadatos de OpenClaw únicamente para los hilos del servidor de aplicaciones de Codex; no
  afecta a las sesiones ACP, la configuración genérica del entorno de Codex ni otros adaptadores de entorno de ejecución.
  Un valor `codex.agents` no vacío limita el servidor a los identificadores de agentes de OpenClaw enumerados.
  La validación de la configuración rechaza las listas de agentes delimitadas por ámbito que estén vacías, en blanco o no sean válidas,
  y la ruta de proyección del entorno de ejecución las omite en lugar de convertirlas en globales.
  `codex.defaultToolsApprovalMode` emite el valor nativo de Codex
  `default_tools_approval_mode` para ese servidor. OpenClaw elimina el bloque `codex`
  antes de pasar la configuración nativa `mcp_servers` a Codex. Omita el bloque para
  mantener el servidor proyectado para todos los agentes del servidor de aplicaciones de Codex con el
  comportamiento predeterminado de aprobación de MCP de Codex.
- Los entornos de ejecución MCP empaquetados y delimitados por sesión utilizan un TTL de inactividad integrado de 10 minutos.
  Las ejecuciones puntuales integradas solicitan una limpieza al finalizar la ejecución; el TTL constituye la protección de respaldo para sesiones de larga duración y futuros invocadores.
- Los cambios en `mcp.*` se aplican en caliente mediante la eliminación de los entornos de ejecución MCP de sesión almacenados en caché.
  La siguiente detección o utilización de herramientas los vuelve a crear a partir de la nueva configuración, por lo que las entradas
  `mcp.servers` eliminadas se descartan inmediatamente en lugar de esperar al TTL de inactividad.
- La detección del entorno de ejecución también respeta las notificaciones de cambios en la lista de herramientas MCP mediante el descarte
  del catálogo almacenado en caché para esa sesión. Los servidores que anuncian recursos o
  indicaciones obtienen herramientas auxiliares para enumerar/leer recursos y enumerar/obtener
  indicaciones. Los fallos repetidos en las llamadas a herramientas suspenden brevemente el servidor afectado antes
  de intentar otra llamada.

Consulte [MCP](/es/cli/mcp#openclaw-as-an-mcp-client-registry) y
[Entornos de CLI](/es/gateway/cli-backends#bundle-mcp-overlays) para conocer el comportamiento del entorno de ejecución.

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
      allowUploadedArchives: false,
    },
    workshop: {
      allowSymlinkTargetWrites: false,
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // o cadena de texto sin formato
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: lista opcional de Skills empaquetadas permitidas únicamente (no afecta a las Skills gestionadas o del espacio de trabajo).
- `load.extraDirs`: raíces adicionales de Skills compartidas (menor precedencia).
- `load.allowSymlinkTargets`: raíces de destino reales de confianza en las que pueden
  resolverse los enlaces simbólicos de Skills cuando el enlace se encuentra fuera de la raíz de origen configurada.
- `workshop.allowSymlinkTargetWrites`: permite que la aplicación de Skill Workshop escriba
  a través de destinos de enlaces simbólicos que ya son de confianza (valor predeterminado: false).
- `install.preferBrew`: cuando es true, da preferencia a los instaladores de Homebrew cuando `brew` está
  disponible antes de recurrir a otros tipos de instaladores.
- `install.nodeManager`: preferencia del instalador de Node para las especificaciones `metadata.openclaw.install`
  (`npm` | `pnpm` | `yarn` | `bun`).
- `install.allowUploadedArchives`: permite que clientes de Gateway `operator.admin` de confianza
  instalen archivos zip privados preparados mediante `skills.upload.*`
  (valor predeterminado: false). Esto solo habilita la ruta de archivos cargados; las instalaciones
  normales de ClawHub no la requieren.
- `entries.<skillKey>.enabled: false` deshabilita una Skill incluso si está empaquetada o instalada.
- `entries.<skillKey>.apiKey`: mecanismo práctico para Skills que declaran una variable de entorno principal (cadena de texto sin formato u objeto SecretRef).
- `limits.maxCandidatesPerRoot`, `limits.maxSkillsLoadedPerSource`, `limits.maxSkillsInPrompt`, `limits.maxSkillsPromptChars`, `limits.maxSkillFileBytes`: delimitan la detección de Skills y la indicación de Skills dirigida al modelo.
- La configuración de autonomía y aprobación de Skill Workshop (`workshop.autonomous.enabled`, `workshop.approvalPolicy`, `workshop.maxPending`, `workshop.maxSkillBytes`) se documenta en [Configuración de Skills](/es/tools/skills-config).

---

## Plugins

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-plugin"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- Se carga desde directorios de paquetes o bundles bajo `~/.openclaw/extensions` y `<workspace>/.openclaw/extensions`, además de los archivos o directorios indicados en `plugins.load.paths`.
- Coloque los archivos de plugins independientes en `plugins.load.paths`; las raíces de extensiones detectadas automáticamente ignoran los archivos de nivel superior `.js`, `.mjs` y `.ts`, para que los scripts auxiliares de esas raíces no bloqueen el inicio.
- La detección admite plugins nativos de OpenClaw, además de bundles compatibles de Codex y Claude, incluidos los bundles de Claude sin manifiesto con la disposición predeterminada.
- **Los cambios de configuración requieren reiniciar el Gateway.**
- `allow`: lista de permitidos opcional (solo se cargan los plugins indicados). `deny` tiene prioridad.
- `plugins.entries.<id>.apiKey`: campo práctico para la clave de API a nivel de plugin (cuando el plugin lo admite).
- `plugins.entries.<id>.env`: mapa de variables de entorno específico del plugin.
- `plugins.entries.<id>.hooks.allowPromptInjection`: cuando es `false`, el núcleo bloquea los hooks que modifican el prompt, como `before_prompt_build`. Se aplica a los hooks de plugins nativos y a los directorios de hooks proporcionados por bundles compatibles.
- `plugins.entries.<id>.hooks.allowConversationAccess`: cuando es `true`, los plugins de confianza no incluidos en el bundle pueden leer el contenido sin procesar de la conversación desde hooks tipados como `llm_input`, `llm_output`, `before_model_resolve`, `before_agent_reply`, `before_agent_run`, `before_agent_finalize` y `agent_end`.
- `plugins.entries.<id>.subagent.allowModelOverride`: confía explícitamente en este plugin para que solicite anulaciones de `provider` y `model` por ejecución en ejecuciones de subagentes en segundo plano.
- `plugins.entries.<id>.subagent.allowedModels`: lista de permitidos opcional de destinos canónicos de `provider/model` para anulaciones de subagentes de confianza. Use `"*"` solo cuando desee permitir intencionadamente cualquier modelo.
- `plugins.entries.<id>.llm.allowModelOverride`: confía explícitamente en este plugin para que solicite anulaciones de modelo para `api.runtime.llm.complete`.
- `plugins.entries.<id>.llm.allowedModels`: lista de permitidos opcional de destinos canónicos de `provider/model` para anulaciones de confianza de finalización LLM del plugin. Use `"*"` solo cuando desee permitir intencionadamente cualquier modelo.
- `plugins.entries.<id>.llm.allowAgentIdOverride`: confía explícitamente en este plugin para ejecutar `api.runtime.llm.complete` con un id de agente distinto del predeterminado.
- `plugins.entries.<id>.config`: objeto de configuración definido por el plugin (validado mediante el esquema del plugin nativo de OpenClaw cuando esté disponible).
- La configuración de cuenta y tiempo de ejecución de los plugins de canal se encuentra bajo `channels.<id>` y debe describirse mediante los metadatos `channelConfigs` del manifiesto del plugin propietario, no mediante un registro central de opciones de OpenClaw.

### Configuración del plugin del arnés de Codex

El plugin `codex` incluido en el bundle es propietario de la configuración del arnés nativo del servidor de aplicaciones de Codex bajo
`plugins.entries.codex.config`. Consulte la
[referencia del arnés de Codex](/es/plugins/codex-harness-reference) para conocer toda la superficie de configuración
y el [arnés de Codex](/es/plugins/codex-harness) para conocer el modelo de tiempo de ejecución.

`codexPlugins` se aplica solo a las sesiones que seleccionan el arnés nativo de Codex.
No habilita plugins de Codex para ejecuciones de proveedores de OpenClaw, vinculaciones de conversaciones
ACP ni ningún arnés que no sea de Codex.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
                allow_destructive_actions: false,
              },
            },
          },
        },
      },
    },
  },
}
```

- `plugins.entries.codex.config.codexPlugins.enabled`: habilita la compatibilidad nativa de Codex
  con plugins y aplicaciones para el arnés de Codex. Valor predeterminado: `false`.
- `plugins.entries.codex.config.codexPlugins.allow_all_plugins`: expone todas las
  aplicaciones actualmente accesibles conectadas a la cuenta de Codex autenticada en
  cada nuevo hilo nativo de Codex. Valor predeterminado: `false`.
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions`:
  política predeterminada de acciones destructivas para las solicitudes interactivas de aplicaciones de plugins configurados.
  Use `true` para aceptar esquemas seguros de aprobación de Codex sin solicitar confirmación, `false`
  para rechazarlos, `"auto"` para enrutar las aprobaciones requeridas por Codex mediante las
  aprobaciones de plugins de OpenClaw o `"ask"` para solicitar confirmación en cada acción
  destructiva o de escritura de un plugin sin aprobación persistente. El modo `"ask"`
  elimina las anulaciones persistentes de aprobación por herramienta de Codex para la aplicación afectada
  y selecciona al revisor humano de aprobaciones para esa aplicación antes de iniciar el hilo de Codex.
  Valor predeterminado: `true`.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.enabled`: habilita una
  entrada de plugin configurada cuando el valor global `codexPlugins.enabled` también es verdadero.
  Valor predeterminado: `true` para entradas explícitas.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.marketplaceName`:
  identidad estable del marketplace, obligatoria junto con `pluginName` para cada entrada
  resuelta. Admite `"openai-curated"` y `"workspace-directory"`. Se ignoran las entradas
  a las que les falte cualquiera de los campos de identidad.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.pluginName`: identidad estable
  del plugin de Codex, obligatoria junto con `marketplaceName`. Una entrada
  `workspace-directory` debe usar el valor `summary.id` exacto y calificado por marketplace
  que devuelve `plugin/list`, por ejemplo
  `"example-plugin@workspace-directory"`.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.allow_destructive_actions`:
  anulación de acciones destructivas por plugin. Si se omite, se usa el valor global
  `allow_destructive_actions`. El valor por plugin acepta las mismas políticas
  `true`, `false`, `"auto"` o `"ask"`.

Cada aplicación de plugin admitida que use `"ask"` enruta las solicitudes de aprobación
de esa aplicación al revisor humano. Las demás aplicaciones y las aprobaciones de hilos que no sean de aplicaciones
conservan el revisor configurado, por lo que las políticas mixtas de plugins no heredan el comportamiento de `"ask"`.

`codexPlugins.enabled` es la directiva de habilitación global. Las entradas explícitas de plugins
creadas por la migración constituyen el conjunto persistente y seleccionado de elegibilidad para instalación y reparación.
Las entradas `workspace-directory` configuradas manualmente ya deben estar
instaladas y habilitadas, y sus aplicaciones propias deben ser accesibles; OpenClaw
no las instala ni las autentica. Si Codex rechaza la solicitud explícita del catálogo del espacio de trabajo,
las entradas habilitadas del espacio de trabajo se cierran de forma segura con
`marketplace_missing`, mientras que las entradas seleccionadas del catálogo predeterminado siguen
disponibles. `plugins["*"]` no es compatible, no existe ningún interruptor `install` y
los valores locales `marketplacePath` no son campos de configuración intencionadamente porque
son específicos del host. Consulte
[plugins nativos de Codex](/es/plugins/codex-native-plugins) para conocer los requisitos de versión y
preparación del servidor de aplicaciones.

Las comprobaciones de preparación de `app/list` se almacenan en caché durante una hora y se actualizan
de forma asíncrona cuando quedan obsoletas. La configuración de aplicaciones del hilo de Codex se calcula al establecer
la sesión del arnés de Codex, no en cada turno; use `/new`, `/reset` o reinicie el Gateway
después de cambiar la configuración de plugins nativos.

`codexPlugins.allow_all_plugins` incorpora una instantánea de todas las aplicaciones de cuenta actualmente accesibles
en cada nuevo hilo nativo de Codex. No instala plugins ni aplicaciones, y
las aplicaciones inaccesibles permanecen excluidas. Las aplicaciones de cuenta usan la política global
`codexPlugins.allow_destructive_actions`. Las entradas explícitas de plugins tienen
prioridad cuando la misma aplicación está presente en ambas rutas. Si no se puede leer `app/list`,
la exposición de toda la cuenta se cierra de forma segura.

- `plugins.entries.firecrawl.config.webFetch`: configuración del proveedor de obtención web Firecrawl.
  - `apiKey`: clave de API opcional de Firecrawl para límites superiores (acepta SecretRef). Recurre a la variable de entorno `plugins.entries.firecrawl.config.webSearch.apiKey` o `FIRECRAWL_API_KEY`.
  - `baseUrl`: URL base de la API de Firecrawl (valor predeterminado: `https://api.firecrawl.dev`; las anulaciones autohospedadas deben apuntar a endpoints privados o internos).
  - `onlyMainContent`: extrae únicamente el contenido principal de las páginas (valor predeterminado: `true`).
  - `maxAgeMs`: antigüedad máxima de la caché en milisegundos (valor predeterminado: `172800000` / 2 días).
  - `timeoutSeconds`: tiempo de espera de la solicitud de extracción en segundos (valor predeterminado: `60`).
- `plugins.entries.xai.config.xSearch`: configuración de xAI X Search (búsqueda web de Grok).
  - `enabled`: habilita el proveedor X Search.
  - `model`: modelo de Grok que se usará para la búsqueda (p. ej., `"grok-4.3"`).
- `plugins.entries.memory-core.config.dreaming`: configuración del Dreaming de memoria. Consulte [Dreaming](/es/concepts/dreaming) para conocer las fases y los umbrales.
  - `enabled`: interruptor principal del Dreaming (valor predeterminado: `false`).
  - `frequency`: cadencia de Cron para cada barrido completo de Dreaming (`"0 3 * * *"` de forma predeterminada).
  - `model`: anulación opcional del modelo del subagente Dream Diary. Requiere `plugins.entries.memory-core.subagent.allowModelOverride: true`; combínela con `allowedModels` para restringir los destinos. Los errores de modelo no disponible vuelven a intentarse una vez con el modelo predeterminado de la sesión; los fallos de confianza o de la lista de permitidos no recurren silenciosamente a otro modelo.
  - La política y los umbrales de las fases son detalles de implementación (no son claves de configuración orientadas al usuario).
- La configuración completa de memoria se encuentra en la [referencia de configuración de memoria](/es/reference/memory-config):
  - `memory.search.*`
  - `agents.entries.*.memory.search.*` para anulaciones por agente
  - `memory.backend`
  - `memory.citations`
  - `memory.qmd.*`
  - `plugins.entries.memory-core.config.dreaming`
- Los plugins de bundles de Claude habilitados también pueden aportar valores predeterminados integrados de OpenClaw desde `settings.json`; OpenClaw los aplica como configuración saneada del agente, no como parches directos de la configuración de OpenClaw.
- `plugins.slots.memory`: selecciona el id del plugin de memoria activo o `"none"` para deshabilitar los plugins de memoria.
- `plugins.slots.contextEngine`: selecciona el id del plugin de motor de contexto activo; el valor predeterminado es `"legacy"`, salvo que se instale y seleccione otro motor.

Consulte [Plugins](/es/tools/plugin).

---

## Navegador

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // aceptar solo para acceso de confianza a redes privadas
      // allowPrivateNetwork: true, // alias heredado
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    tabCleanup: {
      enabled: true,
      idleMinutes: 120,
      maxTabsPerSession: 8,
      sweepMinutes: 5,
    },
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: { driver: "existing-session", attachOnly: true, color: "#00AA00" },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
    color: "#FF4500",
    // headless: false,
    // noSandbox: false,
    // extraArgs: [],
    // executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    // attachOnly: false,
  },
}
```

- `evaluateEnabled: false` deshabilita `act:evaluate` y `wait --fn`.
- `tabCleanup` controla la limpieza periódica de mejor esfuerzo de las pestañas
  rastreadas del agente principal después de un período de inactividad o cuando una sesión supera
  su límite. El rastreo solo se aplica a las pestañas creadas por la herramienta de navegador
  `action: "open"`; las pestañas abiertas por el usuario o cuya propiedad se desconoce nunca
  se adoptan. Deshabilitar `tabCleanup` no deshabilita la limpieza explícita del ciclo de vida de la sesión.
- Las aperturas locales del host con un destino CDP nativo estable y una identidad
  de navegador se almacenan en el estado SQLite compartido y siguen siendo aptas tras los
  reinicios del Gateway para `/new` y la limpieza del ciclo de vida de la sesión.
  Los destinos CDP nativos orientados a herramientas también siguen siendo aptos para la limpieza
  por inactividad y por límite tras el reinicio. Chrome MCP usa identificadores de destino locales
  al proceso, por lo que los registros de sesiones existentes en frío esperan la limpieza del
  ciclo de vida en lugar de arriesgarse a un barrido por inactividad sobre actividad posterior
  al reinicio que no puede atribuirse. OpenClaw verifica el perfil y la instancia del navegador
  antes de cerrarlos. La conexión automática de Chrome MCP, la ausencia de identidad de navegador
  `/json/version` y los destinos nativos sin resolver permanecen completamente locales al proceso,
  por lo que no se cierran automáticamente tras un reinicio. Las pestañas antiguas sin rastrear
  requieren cierre manual. Los fallos transitorios permanecen pendientes para un reintento posterior.
  Consulte [Propiedad de la limpieza de pestañas](/es/tools/browser#tab-cleanup-ownership).
- `ssrfPolicy.dangerouslyAllowPrivateNetwork` está deshabilitado cuando no se establece, por lo que la navegación del navegador permanece estricta de forma predeterminada.
- Establezca `ssrfPolicy.dangerouslyAllowPrivateNetwork: true` solo cuando se confíe intencionadamente en la navegación del navegador por redes privadas.
- En modo estricto, los puntos de conexión de perfiles CDP remotos (`profiles.*.cdpUrl`) están sujetos al mismo bloqueo de redes privadas durante las comprobaciones de accesibilidad y detección.
- `ssrfPolicy.allowPrivateNetwork` sigue siendo compatible como alias heredado.
- En modo estricto, use `ssrfPolicy.hostnameAllowlist` y `ssrfPolicy.allowedHostnames` para excepciones explícitas.
- Los perfiles remotos solo permiten adjuntarse (inicio, detención y restablecimiento deshabilitados).
- `profiles.*.cdpUrl` acepta `http://`, `https://`, `ws://` y `wss://`.
  Use HTTP(S) cuando se quiera que OpenClaw detecte `/json/version`; use WS(S)
  cuando el proveedor proporcione una URL WebSocket directa de DevTools.
- Si se puede acceder a un servicio CDP administrado externamente mediante loopback,
  establezca el valor `attachOnly: true` de ese perfil; de lo contrario, OpenClaw trata el puerto
  loopback como un perfil de navegador local administrado y puede informar de errores locales
  de propiedad del puerto.
- Los perfiles `existing-session` usan Chrome MCP en lugar de CDP y pueden adjuntarse
  en el host seleccionado o mediante un Node de navegador conectado.
- Los perfiles `existing-session` pueden establecer `userDataDir` para dirigirse
  a un perfil específico de navegador basado en Chromium, como Brave o Edge.
- Los perfiles `existing-session` pueden establecer `cdpUrl` cuando Chrome ya se está ejecutando
  detrás de un punto de conexión de detección HTTP(S) de DevTools o de un punto de conexión WS(S) directo.
  En ese modo, OpenClaw pasa el punto de conexión a Chrome MCP en lugar de usar la conexión automática;
  `userDataDir` se ignora para los argumentos de inicio de Chrome MCP.
- Los perfiles `existing-session` mantienen los límites actuales de las rutas de Chrome MCP:
  acciones basadas en instantáneas o referencias en lugar de seleccionar mediante selectores CSS,
  enlaces de carga de un solo archivo, sin anulaciones del tiempo de espera de los cuadros de diálogo,
  sin `wait --load networkidle` y sin `responsebody`, exportación a PDF, interceptación
  de descargas ni acciones por lotes.
- Los perfiles locales administrados `openclaw` asignan automáticamente `cdpPort` y `cdpUrl`;
  establezca `cdpUrl` explícitamente solo para perfiles CDP remotos o para adjuntar
  puntos de conexión de sesiones existentes.
- Los perfiles locales administrados pueden establecer `executablePath` para anular
  el valor global `browser.executablePath` de ese perfil. Use esta opción para ejecutar un perfil
  en Chrome y otro en Brave.
- Orden de detección automática: navegador predeterminado si está basado en Chromium → Chrome → Brave → Edge → Chromium → Chrome Canary.
- Tanto `browser.executablePath` como `browser.profiles.<name>.executablePath`
  aceptan `~` y `~/...` para el directorio principal del sistema operativo antes de iniciar Chromium.
  El valor `userDataDir` por perfil en los perfiles `existing-session` también expande la virgulilla.
- Servicio de control: solo loopback (puerto derivado de `gateway.port`, valor predeterminado `18791`).
- `extraArgs` agrega indicadores de inicio adicionales al arrancar Chromium localmente (por ejemplo,
  `--disable-gpu`, dimensiones de la ventana o indicadores de depuración).

---

## Interfaz de usuario

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // emoji, texto corto, URL de imagen o URI de datos
    },
    prefs: {
      theme: "claw", // claw | knot | dash | custom
      themeMode: "system", // light | dark | system
      locale: "en",
      chatShowThinking: true,
      chatShowToolCalls: true,
      chatPersistCommentary: true, // Mantiene los comentarios después de las ejecuciones en la interfaz de control; no los entrega a los canales
      chatSendShortcut: "enter", // enter | modifier-enter
      chatFollowUpMode: "steer", // steer | queue; omítalo para usar el modo de cola del servidor
      showAdvancedSettings: false, // Expande todos los grupos avanzados de Configuración
    },
  },
}
```

- `seamColor`: color de énfasis para los elementos visuales de la interfaz de usuario de la aplicación nativa (tono de la burbuja del modo de conversación, etc.).
- `assistant`: anulación de identidad de la interfaz de control. Si no se establece, usa la identidad del agente activo.
- `prefs`: preferencias del operador entre dispositivos. Esta es la ubicación canónica para que los agentes puedan
  cambiarlas mediante la puerta de aprobación y todos los clientes de la interfaz de control
  permanezcan sincronizados; los navegadores reflejan los valores en el almacenamiento local para
  un arranque instantáneo y conservan una copia local del dispositivo cuando no pueden escribir
  la configuración (ámbito de visor, sin conexión).
  `chatPersistCommentary` tiene como valor predeterminado `true`. Establecerlo en `false` mantiene
  los comentarios en directo visibles durante una ejecución, pero los elimina al finalizar e impide
  que los nuevos comentarios de Codex entren en el reflejo persistente de la transcripción. La entrega
  a canales de mensajería permanece separada y sin cambios.
  `showAdvancedSettings` tiene como valor predeterminado `false`; la búsqueda de Configuración puede abrir
  temporalmente un grupo avanzado coincidente sin cambiar esta preferencia.
  Las preferencias exclusivamente de presentación, como la escala del texto, el ancho del chat y la
  actividad en directo de la barra lateral, permanecen locales al navegador y se configuran en Configuración.
  Los clientes conectados aplican en directo los cambios del servidor: el Gateway transmite un evento
  `config.changed` que solo contiene un hash después de cada escritura persistente de la configuración,
  y los clientes actualizan su instantánea (se omite mientras un borrador de configuración local
  contiene cambios sin guardar). Los clientes que vuelven a conectarse se sincronizan al conectarse.

---

## Gateway

```json5
{
  gateway: {
    mode: "local", // local | remote
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token", // none | token | password | trusted-proxy
      token: "your-token",
      // password: "your-password", // o OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // para mode=trusted-proxy; consulte /gateway/trusted-proxy-auth
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,
      },
    },
    tailscale: {
      mode: "off", // off | serve | funnel
      resetOnExit: false,
    },
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      // root: "dist/control-ui",
      // toolTitles: false, // títulos opcionales del propósito de la IA para llamadas a herramientas (consume tokens del modelo de utilidad)
      // embedSandbox: "scripts", // strict | scripts | trusted
      // allowExternalEmbedUrls: false, // peligroso: permite URL http(s) externas absolutas para contenido incrustado
      // allowedOrigins: ["https://control.example.com"], // obligatorio para la interfaz de control fuera de loopback
      // dangerouslyAllowHostHeaderOriginFallback: false, // peligroso modo alternativo de origen mediante la cabecera Host
    },
    terminal: {
      enabled: false,
      // shell: "/bin/zsh",
    },
    remote: {
      url: "ws://127.0.0.1:18789",
      transport: "ssh", // ssh | direct
      token: "your-token",
      // password: "your-password",
    },
    trustedProxies: ["10.0.0.1"],
    // Opcional. Valor predeterminado: false.
    allowRealIpFallback: false,
    nodes: {
      pairing: {
        // Opcional. Sin establecer/deshabilitado de forma predeterminada.
        autoApproveCidrs: ["192.168.1.0/24", "fd00:1234:5678::/64"],
        // Aprobación automática verificada mediante SSH. Valor predeterminado: habilitada (true).
        // Establezca false para deshabilitar únicamente la verificación SSH; esto no afecta
        // a autoApproveCidrs arriba. Para emparejar Nodes solo manualmente, establezca false Y
        // quite autoApproveCidrs. Pase un objeto para ajustar: { user, identity,
        // timeoutMs, cidrs }.
        sshVerify: true,
      },
      commands: {
        allow: ["canvas.navigate"],
        deny: ["system.run"],
      },
    },
    tools: {
      // Denegaciones HTTP adicionales de /tools/invoke
      deny: ["browser"],
      // Elimina herramientas de la lista predeterminada de denegaciones HTTP para invocadores propietarios/administradores
      allow: ["gateway"],
    },
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000,
        },
      },
    },
  },
}
```

<Accordion title="Detalles de los campos del Gateway">

- `mode`: `local` (ejecutar el Gateway) o `remote` (conectarse a un Gateway remoto). El Gateway se niega a iniciarse a menos que `local`.
- `port`: un único puerto multiplexado para WS + HTTP. Precedencia: `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`.
- `bind`: `auto`, `loopback` (predeterminado), `lan` (`0.0.0.0`), `tailnet` (IPv4 de Tailscale cuando esté disponible; de lo contrario, bucle invertido) o `custom` (una dirección IPv4). Una dirección `tailnet` resuelta y cualquier dirección `custom` que no sea `127.0.0.1` o `0.0.0.0` requieren `127.0.0.1` en el mismo puerto para los clientes del mismo host; el inicio falla si alguno de los listeners no puede vincularse. La exposición fuera del bucle invertido sigue limitada a la interfaz seleccionada.
- **Alias de vinculación heredados**: use valores del modo de vinculación en `gateway.bind` (`auto`, `loopback`, `lan`, `tailnet`, `custom`), no alias de host (`0.0.0.0`, `127.0.0.1`, `localhost`, `::`, `::1`).
- **Nota sobre Docker**: la vinculación `loopback` predeterminada escucha en `127.0.0.1` dentro del contenedor. Con la red de puente de Docker (`-p 18789:18789`), el tráfico llega por `eth0`, por lo que no se puede acceder al Gateway. Use `--network host` o establezca `bind: "lan"` (o `bind: "custom"` con `customBindHost: "0.0.0.0"`) para escuchar en todas las interfaces.
- **Autenticación**: obligatoria de forma predeterminada. Las vinculaciones fuera del bucle invertido requieren autenticación del Gateway. En la práctica, esto implica un token o una contraseña compartidos, o un proxy inverso con reconocimiento de identidad y `gateway.auth.mode: "trusted-proxy"`. El asistente de incorporación genera un token de forma predeterminada.
- Si se configuran tanto `gateway.auth.token` como `gateway.auth.password` (incluidas las SecretRefs), establezca `gateway.auth.mode` explícitamente en `token` o `password`. El inicio y los flujos de instalación o reparación del servicio fallan cuando ambos están configurados y no se ha establecido el modo.
- `gateway.auth.mode: "none"`: modo explícito sin autenticación. Úselo solo en configuraciones locales de bucle invertido de confianza; no se ofrece intencionadamente en las indicaciones de incorporación.
- `gateway.auth.mode: "trusted-proxy"`: delega la autenticación del navegador o del usuario en un proxy inverso con reconocimiento de identidad y confía en los encabezados de identidad de `gateway.trustedProxies` (consulte [Autenticación mediante proxy de confianza](/es/gateway/trusted-proxy-auth)). De forma predeterminada, este modo espera que el proxy proceda de una dirección **fuera del bucle invertido**; los proxies inversos del mismo host mediante bucle invertido requieren `gateway.auth.trustedProxy.allowLoopback = true` explícito. Los llamadores internos del mismo host pueden usar `gateway.auth.password` como alternativa local directa; `gateway.auth.token` sigue siendo mutuamente excluyente con el modo de proxy de confianza.
- `gateway.auth.allowTailscale`: cuando `true`, los encabezados de identidad de Tailscale Serve pueden satisfacer la autenticación de la interfaz de control/WebSocket (verificados mediante `tailscale whois`). Los endpoints de la API HTTP **no** usan esa autenticación mediante encabezados de Tailscale; en su lugar, siguen el modo normal de autenticación HTTP del Gateway. Este flujo sin token presupone que el host del Gateway es de confianza. El valor predeterminado es `true` cuando `tailscale.mode = "serve"`.
- `gateway.auth.rateLimit`: limitador opcional de intentos de autenticación fallidos. Se aplica por dirección IP de cliente y por ámbito de autenticación (el secreto compartido y el token del dispositivo se registran de forma independiente). Los intentos bloqueados devuelven `429` + `Retry-After`.
  - En la ruta asíncrona de la interfaz de control de Tailscale Serve, los intentos fallidos del mismo `{scope, clientIp}` se serializan antes de escribir el fallo. Por lo tanto, los intentos incorrectos simultáneos del mismo cliente pueden activar el limitador en la segunda solicitud, en lugar de que ambos compitan y se procesen como simples discrepancias.
  - `gateway.auth.rateLimit.exemptLoopback` tiene como valor predeterminado `true`; establezca `false` cuando se quiera limitar intencionadamente también la frecuencia del tráfico de localhost (para configuraciones de prueba o despliegues de proxy estrictos).
- Los intentos de autenticación WS originados en el navegador siempre se limitan, con la exención de bucle invertido desactivada (defensa en profundidad contra ataques de fuerza bruta a localhost desde el navegador).
- En el bucle invertido, esos bloqueos originados en el navegador se aíslan por cada valor normalizado de `Origin`,
  por lo que los fallos repetidos de un origen de localhost no bloquean automáticamente
  un origen diferente.
- `tailscale.mode`: `serve` (solo tailnet, vinculación de bucle invertido) o `funnel` (público, requiere autenticación).
- `tailscale.serviceName`: nombre opcional del servicio de Tailscale para el modo Serve, como
  `svc:openclaw`. Cuando se establece, OpenClaw lo pasa a `tailscale serve
--service` para que la interfaz de control pueda exponerse mediante un servicio con nombre en lugar
  del nombre de host del dispositivo. El valor debe usar el formato de nombre de servicio `svc:<dns-label>`
  de Tailscale; el inicio informa de la URL del servicio derivada.
- `tailscale.preserveFunnel`: cuando `true` y `tailscale.mode = "serve"`, OpenClaw
  comprueba `tailscale funnel status` antes de volver a aplicar Serve durante el inicio y lo omite
  si una ruta de Funnel configurada externamente ya cubre el puerto del Gateway.
  Valor predeterminado: `false`.
- `controlUi.allowedOrigins`: lista explícita de orígenes permitidos para las conexiones WebSocket del Gateway. Es obligatoria para orígenes públicos de navegador fuera del bucle invertido. Las cargas privadas de la interfaz desde LAN/tailnet del mismo origen procedentes de hosts de bucle invertido, RFC1918/enlace local, `.local`, `.ts.net` o CGNAT de Tailscale se aceptan sin habilitar la alternativa basada en el encabezado Host.
- `controlUi.toolTitles`: habilita los títulos de propósito generados por IA para las llamadas a herramientas en el chat de la interfaz de control. Valor predeterminado: `false` (la representación de herramientas sigue siendo completamente determinista, sin llamadas al modelo en segundo plano). Cuando está habilitado, el método `chat.toolTitles` etiqueta las llamadas complejas mediante el enrutamiento estándar del modelo de utilidad: el `utilityModel` del agente (una decisión del operador que puede enviar argumentos acotados de herramientas al proveedor elegido, como cualquier tarea de utilidad) o el modelo pequeño predeterminado declarado por el proveedor de la sesión (OpenAI → `gpt-5.6-luna`, Anthropic → `claude-haiku-4-5`); además, almacena los resultados en la base de datos de estado de cada agente para que las visualizaciones repetidas nunca vuelvan a generar cargos. `utilityModel: \"\"` deshabilita los títulos, como cualquier otra tarea de utilidad; los títulos nunca recurren al modelo principal como alternativa.
- `controlUi.dangerouslyAllowHostHeaderOriginFallback`: modo peligroso que habilita la alternativa de origen basada en el encabezado Host para despliegues que dependen intencionadamente de una política de origen basada en dicho encabezado.
- `terminal.enabled`: habilita el terminal del operador con ámbito de administrador. Valor predeterminado: `false`. El terminal inicia una PTY del host en el espacio de trabajo del agente seleccionado, hereda el entorno del proceso del Gateway y se rechaza para agentes con `sandbox.mode: "all"`. Habilítelo solo en despliegues con operadores de confianza; cambiarlo reinicia el Gateway y actualiza la política de seguridad de contenido de la interfaz de control.
- `terminal.shell`: ejecutable de shell opcional. Si no se establece, OpenClaw usa `$SHELL` en Unix y `%ComSpec%` en Windows.
- `terminal.detachedSessionTimeoutSeconds`: tiempo durante el que una sesión de terminal sobrevive después de perderse su conexión (recarga de la página, suspensión del portátil), permaneciendo disponible para volver a conectarse mediante `terminal.attach` con la reproducción de su salida reciente. Valor predeterminado: `300`. Establezca `0` para finalizar las sesiones en el momento en que se pierda su conexión. Las sesiones desconectadas siguen ejecutando sus comandos, por lo que conviene reducir este valor en hosts compartidos o expuestos.
- `remote.transport`: `ssh` (predeterminado) o `direct` (ws/wss). Para `direct`, `remote.url` debe ser `wss://` en hosts públicos; el texto sin cifrar `ws://` solo se acepta para hosts de bucle invertido, LAN, enlace local, `.local`, `.ts.net` y CGNAT de Tailscale.
- `remote.remotePort`: puerto del Gateway en el host SSH remoto. El valor predeterminado es `18789`; úselo cuando el puerto del túnel local difiera del puerto del Gateway remoto.
- `remote.tlsFingerprint`: huella digital esperada del certificado SHA-256 para un Gateway `wss://` remoto. La aplicación de macOS la aplica tanto a las conexiones de operador/control como a las del nodo complementario. Sin un valor explícito, macOS registra una fijación en el primer uso solo después de que la confianza normal del sistema se valide correctamente.
- `remote.sshHostKeyPolicy`: política de claves de host del túnel SSH de macOS. `strict` es el valor predeterminado y requiere una clave que ya sea de confianza. `openssh` es una habilitación explícita de la configuración efectiva de OpenSSH para los alias administrados; revise la configuración SSH coincidente del usuario y del sistema antes de usarla. La aplicación de macOS y `configure-remote` restablecen esta política a `strict` al cambiar de destino, a menos que se vuelva a habilitar explícitamente.
- `gateway.remote.token` / `.password` son campos de credenciales del cliente remoto. No configuran por sí solos la autenticación del Gateway.
- `gateway.push.apns.relay.baseUrl`: URL HTTPS base del relay externo de APNs que se usa después de que las compilaciones de iOS respaldadas por relay publiquen registros en el Gateway. Las compilaciones públicas de la App Store usan el relay alojado de OpenClaw. Las URL de relay personalizadas deben corresponder a una ruta de compilación/despliegue de iOS deliberadamente independiente cuya URL de relay apunte a ese relay.
- `gateway.push.apns.relay.timeoutMs`: tiempo de espera de envío del Gateway al relay, en milisegundos. El valor predeterminado es `10000`.
- Los registros respaldados por relay se delegan en una identidad específica del Gateway. La aplicación de iOS emparejada obtiene `gateway.identity.get`, incluye esa identidad en el registro del relay y reenvía al Gateway una concesión de envío limitada al registro. Otro Gateway no puede reutilizar ese registro almacenado.
- `OPENCLAW_APNS_RELAY_BASE_URL` / `OPENCLAW_APNS_RELAY_TIMEOUT_MS`: anulaciones temporales mediante variables de entorno para la configuración del relay anterior.
- `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true`: vía de escape exclusiva para desarrollo destinada a URL HTTP de relay en el bucle invertido. Las URL de relay de producción deben mantenerse en HTTPS.
- `OPENCLAW_HANDSHAKE_TIMEOUT_MS`: anulación opcional mediante variable de entorno para el tiempo de espera integrado del protocolo de enlace WebSocket del Gateway previo a la autenticación.
- `channels.<provider>.healthMonitor.enabled`: opción de exclusión por canal de los reinicios del monitor de estado, manteniendo habilitado el monitor global.
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: anulación por cuenta para canales con varias cuentas. Cuando se establece, tiene precedencia sobre la anulación a nivel de canal.
- Las rutas de llamadas al Gateway local pueden usar `gateway.remote.*` como alternativa solo cuando `gateway.auth.*` no está establecido.
- Si `gateway.auth.token` / `gateway.auth.password` se configura explícitamente mediante SecretRef y no se puede resolver, la resolución falla de forma cerrada (sin que una alternativa remota oculte el fallo).
- `trustedProxies`: direcciones IP de proxies inversos que terminan TLS o insertan encabezados de cliente reenviado. Incluya únicamente proxies bajo su control. Las entradas de bucle invertido siguen siendo válidas para configuraciones de proxy o detección local en el mismo host (por ejemplo, Tailscale Serve o un proxy inverso local), pero **no** hacen que las solicitudes de bucle invertido sean aptas para `gateway.auth.mode: "trusted-proxy"`.
- `allowRealIpFallback`: cuando `true`, el Gateway acepta `X-Real-IP` si falta `X-Forwarded-For`. El valor predeterminado es `false` para aplicar un comportamiento de fallo cerrado.
- `gateway.nodes.pairing.autoApproveCidrs`: lista opcional de CIDR/IP permitidas para aprobar automáticamente el primer emparejamiento de un dispositivo de Node sin ámbitos solicitados. Está deshabilitada cuando no se establece. Esto no aprueba automáticamente el emparejamiento de operador/navegador/interfaz de control/WebChat, ni las actualizaciones de rol, ámbito, metadatos o clave pública.
- `gateway.nodes.pairing.sshVerify`: aprobación automática verificada mediante SSH para el primer emparejamiento de un dispositivo de Node (valor predeterminado: habilitada). El Gateway se conecta por SSH al host que solicita el emparejamiento (BatchMode, claves de host estrictas) y solo lo aprueba si la clave del dispositivo `openclaw node identity` coincide exactamente. Se aplica el mismo umbral de elegibilidad que para `autoApproveCidrs`; las comprobaciones se limitan a direcciones de origen privadas/CGNAT, a menos que `cidrs` las anule. Establezca `false` para deshabilitarla o `{ user, identity, timeoutMs, cidrs }` para ajustarla. Consulte [Emparejamiento de Node](/es/gateway/pairing#ssh-verified-device-auto-approval-default).
- `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny`: configuración global de permisos y denegaciones para los comandos declarados del Node después del emparejamiento y de evaluar la lista de permitidos de la plataforma. Use `commands.allow` para habilitar comandos peligrosos del Node, como `camera.snap`, `camera.clip`, `screen.record`, `health.summary`, `sms.search` y `sms.send`; `commands.deny` elimina un comando incluso si, de otro modo, un valor predeterminado de la plataforma o un permiso explícito lo incluyeran. El permiso de Salud de iOS, el permiso de SMS de Android y la autorización de comandos del Gateway son independientes. Después de que un Node cambie su lista de comandos declarados, rechace y vuelva a aprobar el emparejamiento de ese dispositivo para que el Gateway almacene la instantánea actualizada de los comandos.
- `gateway.tools.deny`: nombres de herramientas adicionales bloqueados para `POST /tools/invoke` HTTP (amplía la lista de denegación predeterminada).
- `gateway.tools.allow`: elimina nombres de herramientas de la lista de denegación HTTP predeterminada para
  los solicitantes propietarios/administradores. Esto no concede acceso de propietario/administrador a los solicitantes `operator.write`
  que portan una identidad; `cron`, `gateway` y `nodes` siguen
  sin estar disponibles para los solicitantes que no son propietarios, incluso cuando se incluyen en la lista de permitidos.

</Accordion>

### Endpoints compatibles con OpenAI

- RPC HTTP de administración: desactivado de forma predeterminada como el plugin `admin-http-rpc`. Active el plugin para registrar `POST /api/v1/admin/rpc`. Consulte [RPC HTTP de administración](/es/plugins/admin-http-rpc).
- Chat Completions: desactivado de forma predeterminada. Actívelo con `gateway.http.endpoints.chatCompletions.enabled: true`.
- API Responses: `gateway.http.endpoints.responses.enabled`.
- Refuerzo de la entrada de URL de Responses:
  - `gateway.http.endpoints.responses.maxUrlParts`
  - `gateway.http.endpoints.responses.files.urlAllowlist`
  - `gateway.http.endpoints.responses.images.urlAllowlist`
    Las listas de permitidos vacías se consideran no definidas; use `gateway.http.endpoints.responses.files.allowUrl=false`
    y/o `gateway.http.endpoints.responses.images.allowUrl=false` para desactivar la obtención de URL.
- Encabezado opcional de refuerzo de respuestas:
  - `gateway.http.securityHeaders.strictTransportSecurity` (establézcalo únicamente para orígenes HTTPS que controle; consulte [Autenticación mediante proxy de confianza](/es/gateway/trusted-proxy-auth#tls-termination-and-hsts))

### Aislamiento de múltiples instancias

Ejecute varios gateways en un mismo host con puertos y directorios de estado únicos:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json \
OPENCLAW_STATE_DIR=~/.openclaw-a \
openclaw gateway --port 19001
```

Opciones prácticas: `--dev` (usa `~/.openclaw-dev` + el puerto `19001`), `--profile <name>` (usa `~/.openclaw-<name>`).

Consulte [Varios gateways](/es/gateway/multiple-gateways).

### `gateway.tls`

```json5
{
  gateway: {
    tls: {
      enabled: false,
      autoGenerate: false,
      certPath: "/etc/openclaw/tls/server.crt",
      keyPath: "/etc/openclaw/tls/server.key",
      caPath: "/etc/openclaw/tls/ca-bundle.crt",
    },
  },
}
```

- `enabled`: habilita la terminación TLS en el receptor del gateway (HTTPS/WSS) (valor predeterminado: `false`).
- `autoGenerate`: genera automáticamente un par local de certificado y clave autofirmados cuando no se configuran archivos explícitos; solo para uso local o de desarrollo.
- `certPath`: ruta del sistema de archivos al archivo del certificado TLS.
- `keyPath`: ruta del sistema de archivos al archivo de la clave privada TLS; mantenga restringidos sus permisos.
- `caPath`: ruta opcional al paquete de CA para la verificación de clientes o cadenas de confianza personalizadas.

### `gateway.reload`

```json5
{
  gateway: {
    reload: {
      mode: "hybrid", // off | restart | hot | hybrid
      debounceMs: 500,
      deferralTimeoutMs: 300000,
    },
  },
}
```

- `mode`: controla cómo se aplican durante la ejecución los cambios de configuración.
  - `"off"`: ignora los cambios en vivo; los cambios requieren un reinicio explícito.
  - `"restart"`: reinicia siempre el proceso del gateway cuando cambia la configuración.
  - `"hot"`: aplica los cambios en el proceso sin reiniciarlo.
  - `"hybrid"` (valor predeterminado): intenta primero la recarga en caliente; si es necesario, recurre al reinicio.
- `debounceMs`: intervalo de antirrebote en ms antes de aplicar los cambios de configuración (entero no negativo; valor predeterminado: `300`).
- `deferralTimeoutMs`: tiempo máximo opcional en ms para esperar a que terminen las operaciones en curso antes de forzar un reinicio o una recarga en caliente del canal. Omítalo para usar la espera limitada predeterminada (`300000`); establezca `0` para esperar indefinidamente y registrar periódicamente advertencias de que aún hay operaciones pendientes.

---

## Entornos de trabajadores en la nube

Los trabajadores en la nube son opcionales. Si `cloudWorkers` no está presente o `profiles` está vacío, OpenClaw no acepta la creación de nuevos trabajadores. Los registros duraderos creados anteriormente siguen reconciliándose y permanecen visibles; la proyección existente de gateway/node no cambia.

Cada proveedor de trabajadores debe devolver una `hostKey` SSH desde la salida de aprovisionamiento de confianza exactamente como `algorithm base64`, sin nombre de host ni comentario. El arranque escribe esa clave en un archivo `known_hosts` aislado, usa `StrictHostKeyChecking=yes` y falla antes de abrir una conexión si el proveedor la omite. No existe una alternativa de confiar en el primer uso.

El túnel se configura bajo demanda, en lugar de formar parte del aprovisionamiento. Cuando se inicia, el gateway realiza un reenvío inverso de un socket Unix local al trabajador hacia su endpoint WebSocket de bucle local. El socket reside en un directorio remoto asignado aleatoriamente y accesible solo por su propietario; a diferencia de un puerto TCP de bucle local, otras cuentas de un trabajador multiusuario no pueden acceder a él y no puede entrar en conflicto con el puerto de otro entorno. Los mensajes de mantenimiento de conexión SSH y el retroceso de reconexión limitado solo se ejecutan mientras el propietario del túnel siga siendo el actual. Al detener el túnel, se bloquean las reconexiones antes de cerrar el proceso SSH.

El tráfico de control y la transferencia del espacio de trabajo usan conexiones SSH independientes. Ambos reutilizan la misma identidad resuelta y el archivo `known_hosts` fijado y aislado, pero la transferencia del espacio de trabajo no comparte la multiplexación de conexiones SSH con el túnel de larga duración, por lo que rsync no puede bloquear el tráfico de control.

### Perfil de Crabbox

El proveedor `crabbox` incluido aprovisiona una concesión compatible con SSH mediante la CLI local de Crabbox. El `settings.provider` interno selecciona el backend de Crabbox; es independiente del id de proveedor externo de OpenClaw.

```json5
{
  cloudWorkers: {
    profiles: {
      production: {
        provider: "crabbox",
        install: "bundle", // Predeterminado; use "npm" solo para una versión publicada del gateway.
        settings: {
          provider: "aws",
          class: "standard",
          ttl: "24h",
          idleTimeout: "60m",
          // Ruta absoluta opcional. Valor predeterminado: el archivo hermano ../crabbox/bin/crabbox y, después, PATH.
          binary: "/usr/local/bin/crabbox",
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `settings.provider` (obligatorio): backend de Crabbox que se pasa mediante `--provider`. Use un backend cuya salida de inspección incluya un endpoint SSH; `aws` selecciona el backend directo de AWS.
- `settings.class` (obligatorio): clase de máquina de Crabbox que se pasa a `--class`.
- `settings.ttl` y `settings.idleTimeout` (obligatorios): cadenas de duración Go positivas que se pasan a `--ttl` y `--idle-timeout`. Estos mecanismos de seguridad del proveedor son distintos de la política `lifetime` almacenada por OpenClaw que se muestra más adelante.
- `settings.binary`: ruta absoluta opcional al ejecutable de Crabbox. Si no se especifica, OpenClaw comprueba el checkout hermano de Crabbox, después las entradas ejecutables de `PATH` y, finalmente, invoca `crabbox`, de modo que la ausencia de la CLI siga siendo un error visible del proveedor.

Las opciones desconocidas se rechazan. Las credenciales de Crabbox y la configuración de cuenta específica del backend siguen siendo responsabilidad de Crabbox; no las incluya en `settings`. OpenClaw solo invoca la CLI local y este plugin no realiza llamadas de red al proveedor. El aprovisionamiento siempre pasa `--keep=true`; OpenClaw gestiona el ciclo de vida externo y destruye la concesión con `crabbox stop`.

<Note>
  OpenClaw resuelve la ruta `sshKey` local a la concesión de Crabbox mediante el resolvedor de secretos propiedad del proveedor y fija la `sshHostKey` autoritativa que devuelve `crabbox inspect --json`. La admisión en AWS también requiere `providerMetadata.instanceProfileAttached`. Instale Crabbox 0.38.1 o una versión posterior para este contrato de inspección cerrado.
</Note>

### Perfil de desarrollo SSH estático

```json5
{
  cloudWorkers: {
    profiles: {
      development: {
        provider: "static-ssh",
        settings: {
          host: "worker.example.test",
          port: 22,
          user: "openclaw",
          hostKey: "ssh-ed25519 <base64-public-host-key>",
          keyRef: {
            source: "env",
            provider: "default",
            id: "OPENCLAW_WORKER_SSH_KEY",
          },
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `profiles`: perfiles de trabajador con nombre e identificadores no vacíos y sin espacios en blanco al principio ni al final. Cada perfil selecciona un proveedor registrado por un plugin.
- `provider`: id no vacío del proveedor de trabajadores. Los ejemplos usan el proveedor `crabbox` incluido y el proveedor `static-ssh` de QA Lab.
- `install`: método de instalación del trabajador. `"bundle"` (valor predeterminado) transfiere un paquete con hash de contenido de la compilación instalada del gateway y admite versiones publicadas, de desarrollo y no publicadas. `"npm"` es una optimización opcional para una versión empaquetada sin modificaciones; instala `openclaw@<exact gateway version>` desde el registro público de npm y nunca instala `latest`.
- Los plugins de proveedores incluidos se seleccionan automáticamente cuando se configuran, pero las desactivaciones explícitas y `plugins.allow` siguen aplicándose. Incluya el id del proveedor (por ejemplo, `crabbox`) cuando se configure una lista de permitidos. Los plugins de proveedores externos también deben instalarse y habilitarse explícitamente.
- `settings`: JSON limitado propiedad del proveedor. El plugin seleccionado define y valida sus claves; use [objetos SecretRef](/es/gateway/secrets) para valores que contengan secretos. El proveedor SSH estático requiere `host`, `user`, `hostKey` y `keyRef`; el valor predeterminado de `port` es `22`. `hostKey` debe ser una línea de clave pública de host OpenSSH (`algorithm base64`) obtenida del host conocido o de otro canal de confianza, sin prefijo de opciones.
- `lifetime.idleTimeoutMinutes`: minutos expresados como entero positivo y almacenados para la política posterior de recuperación por inactividad.
- `lifetime.maxLifetimeMinutes`: minutos expresados como entero positivo y almacenados para la política posterior de ciclo de vida.

En el trabajador ya debe estar instalado un entorno de ejecución de Node compatible (22.22.3+, 24.15+ o 25.9+) con SQLite seguro frente al restablecimiento de WAL. El método opcional `"npm"` también requiere `npm` y acceso HTTPS saliente al registro público de npm. La configuración de cadenas de herramientas en red es una política del proveedor; el arranque informa de un error que permite tomar medidas en lugar de instalar las cadenas de herramientas por sí mismo.

Esta base instala y verifica la compilación del gateway y proporciona el ciclo de vida de inicio y detención del túnel, pero no inicia la CLI general de OpenClaw. La entrada autónoma del trabajador y el bucle se incorporarán en el próximo hito de los trabajadores en la nube.

Cada registro de entorno duradero conserva las opciones validadas de su proveedor, el método de instalación resuelto y la política de duración en una instantánea del perfil tomada en el momento de la creación. Cambiar o eliminar un perfil con nombre afecta a las nuevas creaciones; los registros existentes siguen reconciliando su ciclo de vida con esa instantánea, siempre que el plugin propietario continúe disponible.

En la primera versión de los trabajadores en la nube, los valores de duración son solo datos; la aplicación automática se incorporará con trabajos posteriores sobre el ciclo de vida. Los cambios en los perfiles requieren reiniciar el gateway.

<Warning>
  El proveedor `static-ssh` es un entorno de desarrollo de QA Lab basado en el árbol de fuentes y está excluido de las distribuciones empaquetadas. Un trabajador que se ejecute en su host compartido puede leer datos no relacionados del host, por lo que no debe usarse este proveedor como límite de aislamiento de producción.
  Su operador debe proporcionar la `hostKey` esperada; OpenClaw no obtendrá ni aceptará una clave de la primera conexión.
  Destruir su concesión solo libera el registro lógico de OpenClaw; no detiene ni limpia el host.
</Warning>

---

## Hooks

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: true,
    allowedSessionKeyPrefixes: ["hook:", "hook:gmail:"],
    allowedAgentIds: ["hooks", "main"],
    presets: ["gmail"],
    transformsDir: "~/.openclaw/hooks/transforms",
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        agentId: "hooks",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "De: {{messages[0].from}}\nAsunto: {{messages[0].subject}}\n{{messages[0].snippet}}",
        deliver: true,
        channel: "last",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

Autenticación: `Authorization: Bearer <token>` o `x-openclaw-token: <token>`.
Se rechazan los tokens de hook incluidos en la cadena de consulta.

Notas de validación y seguridad:

- `hooks.enabled=true` requiere un `hooks.token` no vacío.
- `hooks.token` debe ser distinto de la autenticación activa mediante secreto compartido del Gateway (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` o `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`); al iniciarse, se registra una advertencia de seguridad no crítica cuando se detecta su reutilización.
- `openclaw security audit` marca la reutilización de la autenticación del hook/Gateway como un hallazgo crítico, incluida la autenticación por contraseña del Gateway proporcionada solo durante la auditoría (`--auth password --password <password>`). Ejecute `openclaw doctor --fix` para rotar un `hooks.token` persistido y reutilizado; después, actualice los emisores de hooks externos para que utilicen el nuevo token del hook.
- `hooks.path` no puede ser `/`; utilice una subruta específica, como `/hooks`.
- Si `hooks.allowRequestSessionKey=true`, restrinja `hooks.allowedSessionKeyPrefixes` (por ejemplo, `["hook:"]`).
- Si una asignación o un preajuste utiliza un `sessionKey` basado en plantilla, establezca `hooks.allowedSessionKeyPrefixes` y `hooks.allowRequestSessionKey=true`. Las claves de asignación estáticas no requieren esta habilitación explícita.

**Endpoints:**

- `POST /hooks/wake` → `{ text, mode?: "now"|"next-heartbeat" }`
- `POST /hooks/agent` → `{ message, name?, agentId?, sessionKey?, wakeMode?, deliver?, channel?, to?, model?, thinking?, timeoutSeconds? }`
  - `sessionKey` de la carga útil de la solicitud solo se acepta cuando `hooks.allowRequestSessionKey=true` (valor predeterminado: `false`).
- `POST /hooks/<name>` → se resuelve mediante `hooks.mappings`
  - Los valores `sessionKey` de asignaciones renderizadas mediante plantillas se consideran proporcionados externamente y también requieren `hooks.allowRequestSessionKey=true`.

<Accordion title="Detalles de las asignaciones">

- `match.path` coincide con la subruta después de `/hooks` (p. ej., `/hooks/gmail` → `gmail`).
- `match.source` coincide con un campo de la carga útil para rutas genéricas.
- Las plantillas como `{{messages[0].subject}}` leen datos de la carga útil.
- `transform` puede apuntar a un módulo JS/TS que devuelva una acción de hook.
  - `transform.module` debe ser una ruta relativa y permanecer dentro de `hooks.transformsDir` (se rechazan las rutas absolutas y el recorrido de directorios).
  - Mantenga `hooks.transformsDir` dentro de `~/.openclaw/hooks/transforms`; se rechazan los directorios de Skills del espacio de trabajo. Si `openclaw doctor` indica que esta ruta no es válida, mueva el módulo de transformación al directorio de transformaciones de hooks o elimine `hooks.transformsDir`.
- `agentId` dirige a un agente específico; los ID desconocidos recurren al agente predeterminado.
- `allowedAgentIds`: restringe el enrutamiento efectivo de agentes, incluida la ruta del agente predeterminado cuando se omite `agentId` (`*` u omitido = permitir todos, `[]` = denegar todos).
- `defaultSessionKey`: clave de sesión fija opcional para ejecuciones de agentes mediante hooks sin un `sessionKey` explícito.
- `allowRequestSessionKey`: permite que los invocadores de `/hooks/agent` y las claves de sesión de asignaciones basadas en plantillas establezcan `sessionKey` (valor predeterminado: `false`).
- `allowedSessionKeyPrefixes`: lista de permitidos opcional de prefijos para valores `sessionKey` explícitos (solicitud + asignación), p. ej., `["hook:"]`. Pasa a ser obligatoria cuando alguna asignación o preajuste utiliza un `sessionKey` basado en plantilla.
- `deliver: true` envía la respuesta final a un canal; el valor predeterminado de `channel` es `last`.
- `model` sustituye el LLM para esta ejecución del hook (debe estar permitido si se ha establecido el catálogo de modelos).

</Accordion>

### Integración con Gmail

- El preajuste integrado de Gmail utiliza `sessionKey: "hook:gmail:{{messages[0].id}}"`.
- Esta clave por mensaje aísla el contexto de la conversación, no las herramientas ni el acceso al espacio de trabajo. Sin una asignación personalizada que establezca `agentId`, el preajuste utiliza el agente predeterminado.
- Para bandejas de entrada que no sean de confianza, dirija Gmail a un agente lector específico y restrinja dicho agente mediante la [zona de pruebas y la política de herramientas por agente](/es/tools/multi-agent-sandbox-tools). Si el lector debe notificar al agente principal, restrinja la transferencia mediante [`tools.agentToAgent`](/es/gateway/config-tools#toolsagenttoagent). Consulte [Inyección de prompts](/es/gateway/security#prompt-injection) para conocer el modelo de amenazas y el nivel de modelo recomendados.
- Si mantiene ese enrutamiento por mensaje, establezca `hooks.allowRequestSessionKey: true` y restrinja `hooks.allowedSessionKeyPrefixes` para que coincida con el espacio de nombres de Gmail, por ejemplo, `["hook:", "hook:gmail:"]`.
- Si necesita `hooks.allowRequestSessionKey: false`, sustituya el preajuste con un `sessionKey` estático en lugar del valor predeterminado basado en plantilla.

```json5
{
  hooks: {
    gmail: {
      account: "openclaw@gmail.com",
      topic: "projects/<project-id>/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,
      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

- El Gateway inicia automáticamente `gog gmail watch serve` durante el arranque cuando está configurado. Establezca `OPENCLAW_SKIP_GMAIL_WATCHER=1` para desactivarlo.
- No ejecute un `gog gmail watch serve` independiente junto con el Gateway.

---

## Host del plugin Canvas

```json5
{
  plugins: {
    entries: {
      canvas: {
        config: {
          host: {
            root: "~/.openclaw/workspace/canvas",
            liveReload: true,
            // enabled: false, // or OPENCLAW_SKIP_CANVAS_HOST=1
          },
        },
      },
    },
  },
}
```

- Sirve HTML/CSS/JS editable por agentes y A2UI mediante HTTP en el puerto del Gateway:
  - `http://<gateway-host>:<gateway.port>/__openclaw__/canvas/`
  - `http://<gateway-host>:<gateway.port>/__openclaw__/a2ui/`
- Solo local: mantenga `gateway.bind: "loopback"` (valor predeterminado).
- Enlaces que no son de bucle invertido: las rutas de Canvas requieren autenticación del Gateway (token/contraseña/proxy de confianza), como las demás superficies HTTP del Gateway.
- Las WebViews de Node normalmente no envían encabezados de autenticación; después de emparejar y conectar un nodo, el Gateway anuncia URL de capacidades con ámbito de nodo para acceder a Canvas/A2UI.
- Las URL de capacidades están vinculadas a la sesión WS activa del nodo y caducan rápidamente. No se utiliza un mecanismo alternativo basado en IP.
- Inyecta el cliente de recarga en vivo en el HTML servido.
- Crea automáticamente un `index.html` inicial cuando está vacío.
- También sirve A2UI en `/__openclaw__/a2ui/`.
- Los cambios requieren reiniciar el Gateway.
- Desactive la recarga en vivo para directorios grandes o errores de `EMFILE`.

---

## Detección

### mDNS (Bonjour)

```json5
{
  discovery: {
    mdns: {
      mode: "minimal", // minimal | full | off
    },
  },
}
```

- `minimal` (valor predeterminado): omite `cliPath` + `sshPort` de los registros TXT.
- `full`: incluye `cliPath` + `sshPort`; la difusión multicast en la LAN sigue requiriendo que el plugin `bonjour` incluido esté habilitado.
- `off`: suprime la difusión multicast en la LAN sin cambiar la habilitación del plugin.
- El plugin `bonjour` incluido se inicia automáticamente en hosts macOS y requiere habilitación explícita en Linux, Windows y despliegues del Gateway en contenedores.
- El nombre de host utiliza de forma predeterminada el nombre de host del sistema cuando este es una etiqueta DNS válida y recurre a `openclaw` en caso contrario. Sustitúyalo mediante `OPENCLAW_MDNS_HOSTNAME`.
- `OPENCLAW_DISABLE_BONJOUR=1` desactiva por completo la difusión mDNS y prevalece sobre `discovery.mdns.mode`.

### Área extensa (DNS-SD)

```json5
{
  discovery: {
    wideArea: { enabled: true },
  },
}
```

Escribe una zona DNS-SD unicast bajo `~/.openclaw/dns/`. Para la detección entre redes, combínela con un servidor DNS (se recomienda CoreDNS) y DNS dividido de Tailscale.

Configuración: `openclaw dns setup --apply`.

---

## Entorno

### `env` (variables de entorno en línea)

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

- Las variables de entorno en línea solo se aplican si falta la clave en el entorno del proceso.
- Archivos `.env`: `.env` del CWD + `~/.openclaw/.env` (ninguno sustituye las variables existentes).
- `shellEnv`: importa las claves esperadas que falten desde el perfil del shell de inicio de sesión.
- Consulte [Entorno](/es/help/environment) para conocer la precedencia completa.

### Sustitución de variables de entorno

Haga referencia a variables de entorno en cualquier cadena de configuración mediante `${VAR_NAME}`:

```json5
{
  gateway: {
    auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" },
  },
}
```

- Solo se reconocen nombres en mayúsculas: `[A-Z_][A-Z0-9_]*`.
- Las variables ausentes o vacías provocan un error al cargar la configuración.
- Use la secuencia de escape `$${VAR}` para representar literalmente `${VAR}`.
- Funciona con `$include`.

---

## Secretos

Las referencias a secretos son aditivas: los valores de texto sin formato siguen funcionando.

### `SecretRef`

Utilice una de las siguientes estructuras de objeto:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

Validación:

- Patrón de `provider`: `^[a-z][a-z0-9_-]{0,63}$`
- Patrón del ID de `source: "env"`: `^[A-Z][A-Z0-9_]{0,127}$`
- ID de `source: "file"`: puntero JSON absoluto (por ejemplo, `"/providers/openai/apiKey"`)
- Patrón del ID de `source: "exec"`: `^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$` (admite selectores `secret#json_key` al estilo de AWS)
- Los ID de `source: "exec"` no deben contener segmentos de ruta delimitados por barras `.` o `..` (por ejemplo, se rechaza `a/../b`)

### Superficie de credenciales compatible

- Matriz canónica: [Superficie de credenciales de SecretRef](/es/reference/secretref-credential-surface)
- `secrets apply` tiene como destino rutas de credenciales `openclaw.json` compatibles.
- Las referencias de `auth-profiles.json` se incluyen en la resolución durante la ejecución y en la cobertura de auditoría.

### Configuración de proveedores de secretos

```json5
{
  secrets: {
    providers: {
      default: { source: "env" }, // optional explicit env provider
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",
        timeoutMs: 5000,
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        passEnv: ["PATH", "VAULT_ADDR"],
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

Notas:

- El proveedor `file` admite `mode: "json"` y `mode: "singleValue"` (`id` debe ser `"value"` en el modo singleValue).
- Las rutas de los proveedores de archivos y ejecución producen un fallo cerrado cuando no está disponible la verificación de ACL de Windows. Establezca `allowInsecurePath: true` solo para rutas de confianza que no puedan verificarse.
- El proveedor `exec` requiere una ruta `command` absoluta y utiliza cargas útiles del protocolo en stdin/stdout.
- De forma predeterminada, se rechazan las rutas de comandos que sean enlaces simbólicos. Establezca `allowSymlinkCommand: true` para permitir rutas con enlaces simbólicos mientras se valida la ruta de destino resuelta.
- Si se configura `trustedDirs`, la comprobación del directorio de confianza se aplica a la ruta de destino resuelta.
- El entorno secundario de `exec` es mínimo de forma predeterminada; pase explícitamente las variables necesarias mediante `passEnv`.
- Las referencias a secretos se resuelven en el momento de la activación en una instantánea en memoria; posteriormente, las rutas de las solicitudes solo leen dicha instantánea.
- El filtrado de superficies activas se aplica durante la activación: las referencias sin resolver en superficies habilitadas provocan un fallo de inicio o recarga, mientras que las superficies inactivas se omiten con diagnósticos.

---

## Almacenamiento de autenticación

```json5
{
  auth: {
    profiles: {
      "anthropic:default": { provider: "anthropic", mode: "api_key" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai:personal": { provider: "openai", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:default", "anthropic:work"],
      openai: ["openai:personal"],
    },
  },
}
```

- Los perfiles por agente se almacenan en `<agentDir>/auth-profiles.json`.
- `auth-profiles.json` admite referencias a nivel de valor (`keyRef` para `api_key`, `tokenRef` para `token`) para los modos de credenciales estáticas.
- Los mapas planos heredados de `auth-profiles.json`, como `{ "provider": { "apiKey": "..." } }`, no son un formato de tiempo de ejecución; `openclaw doctor --fix` los reescribe como perfiles canónicos de clave de API de `provider:default` con una copia de seguridad `.legacy-flat.*.bak`.
- Los perfiles en modo OAuth (`auth.profiles.<id>.mode = "oauth"`) no admiten credenciales de perfiles de autenticación respaldadas por SecretRef.
- Las credenciales estáticas de tiempo de ejecución proceden de instantáneas resueltas en memoria; las entradas estáticas heredadas de `auth.json` se eliminan cuando se detectan.
- Importaciones de OAuth heredadas desde `~/.openclaw/credentials/oauth.json`.
- Consulte [OAuth](/es/concepts/oauth).
- Comportamiento de los secretos en tiempo de ejecución y herramientas de `audit/configure/apply`: [Gestión de secretos](/es/gateway/secrets).

---

## Auditoría

```json5
{
  audit: {
    enabled: true,
    messages: "off", // off | direct | all
  },
}
```

El Gateway registra eventos de auditoría **solo de metadatos** para las ejecuciones de agentes y las acciones de herramientas en la base de datos de estado compartida. Los metadatos del ciclo de vida de los mensajes requieren una activación independiente. El registro almacena la identidad, los tiempos, los nombres de las herramientas y los resultados normalizados, pero nunca los prompts, los cuerpos de los mensajes, los argumentos de las herramientas, los resultados ni el texto sin procesar de los errores. Las filas de mensajes no almacenan los identificadores sin procesar de cuentas de plataforma, conversaciones, mensajes ni destinos. Las claves de sesión de ejecuciones y herramientas siguen disponibles para la correlación y pueden contener identificadores de cuentas de plataforma o de pares. Los registros caducan después de 30 días y el registro tiene un límite de 100,000 filas. Consúltelos con
[`openclaw audit`](/es/cli/audit) o mediante la RPC del Gateway
[`audit.activity.list`](/es/gateway/protocol#audit-ledger-rpc). Consulte
[Historial de auditoría](/es/gateway/audit) para conocer el modelo de datos completo, la semántica de privacidad y los límites de cobertura.

- `enabled`: registra nuevos eventos de auditoría (valor predeterminado: `true`). El registro está activado de forma predeterminada porque un historial de auditoría que solo se active después de un incidente no puede explicar dicho incidente. Establecer `false` detiene la inserción de nuevos eventos después de reiniciar el Gateway; los registros existentes siguen siendo legibles hasta que caduquen. Al volver a activarlo, el registro se reanuda desde ese momento; el intervalo sin datos no se rellena de forma retroactiva.
- `messages`: ámbito de los metadatos de mensajes (valor predeterminado: `"off"`). `"direct"` registra únicamente las conversaciones directas conocidas. `"all"` también registra grupos, canales y tipos de conversación desconocidos. Ambos modos siguen sin incluir contenido y sustituyen los identificadores sin procesar por seudónimos con clave locales de la instalación cuando la correlación está disponible. Estos sirven para facilitar la correlación, no para anonimizar; la base de datos de estado almacena la clave de derivación, pero las exportaciones de RPC y CLI no lo hacen.

El Gateway en ejecución captura `audit.enabled` y `audit.messages` al iniciarse; reinícielo después de cambiar cualquiera de los ajustes. Actualmente, la cobertura de mensajes incluye los mensajes entrantes aceptados que llegan al despacho principal y una fila terminal por cada carga útil original de respuesta saliente lógica que alcanza la entrega duradera compartida. Las rutas locales de los plugins y de envío directo que omiten esos límites compartidos aún no están cubiertas. El escritor en segundo plano con límites funciona según el mejor esfuerzo posible; no es un archivo de cumplimiento sin pérdidas.

---

## Registro

```json5
{
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty", // pretty | compact | json
    redactSensitive: "tools", // off | tools
    redactPatterns: ["\\bTOKEN\\b\\s*[=:]\\s*([\"']?)([^\\s\"']+)\\1"],
  },
}
```

- Archivo de registro predeterminado: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`; los perfiles con nombre utilizan `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`.
- Establezca `logging.file` para usar una ruta estable.
- `consoleLevel` aumenta a `debug` cuando `--verbose`.
- `maxFileBytes`: tamaño máximo del archivo de registro activo en bytes antes de la rotación (entero positivo; valor predeterminado: `104857600` = 100 MB). OpenClaw conserva hasta cinco archivos numerados junto al archivo activo.
- `redactSensitive` / `redactPatterns`: ocultación según el mejor esfuerzo posible para la salida de consola, los registros de archivos, los registros de OTLP y el texto persistente de las transcripciones de sesión. `redactSensitive: "off"` solo desactiva esta política general de registros y transcripciones; las superficies de seguridad de la interfaz de usuario, las herramientas y los diagnósticos siguen ocultando los secretos antes de emitirlos.

---

## Diagnósticos

```json5
{
  diagnostics: {
    enabled: true,
    flags: ["telegram.*"],

    otel: {
      enabled: false,
      endpoint: "https://otel-collector.example.com:4318",
      tracesEndpoint: "https://traces.example.com/v1/traces",
      metricsEndpoint: "https://metrics.example.com/v1/metrics",
      logsEndpoint: "https://logs.example.com/v1/logs",
      protocol: "http/protobuf", // http/protobuf | grpc
      headers: { "x-tenant-id": "my-org" },
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: false,
      logsExporter: "otlp",
      sampleRate: 1.0,
      flushIntervalMs: 5000,
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },

    cacheTrace: {
      enabled: false,
      filePath: "~/.openclaw/logs/cache-trace.jsonl",
      includeMessages: true,
      includePrompt: true,
      includeSystem: true,
    },
  },
}
```

- `enabled`: conmutador principal para la salida de instrumentación (valor predeterminado: `true`).
- `flags`: matriz de cadenas de indicadores que activa la salida de registros específica (admite comodines como `"telegram.*"` o `"*"`).
- `otel.enabled`: activa el pipeline de exportación de OpenTelemetry (valor predeterminado: `false`). Para conocer la configuración completa, el catálogo de señales y el modelo de privacidad, consulte [Exportación de OpenTelemetry](/es/gateway/opentelemetry).
- `otel.endpoint`: URL del recopilador para la exportación de OTel.
- `otel.tracesEndpoint` / `otel.metricsEndpoint` / `otel.logsEndpoint`: endpoints OTLP opcionales específicos de cada señal. Cuando se establecen, reemplazan `otel.endpoint` únicamente para esa señal.
- `otel.protocol`: `"http/protobuf"` (valor predeterminado) o `"grpc"`.
- `otel.headers`: encabezados de metadatos HTTP/gRPC adicionales enviados con las solicitudes de exportación de OTel.
- `otel.serviceName`: nombre del servicio para los atributos de recursos.
- `otel.traces` / `otel.metrics` / `otel.logs`: activan la exportación de trazas, métricas o registros.
- `otel.logsExporter`: destino de exportación de registros: `"otlp"` (valor predeterminado), `"stdout"` para un objeto JSON por línea de salida estándar o `"both"`.
- `otel.sampleRate`: tasa de muestreo de trazas de `0` a `1`.
- `otel.flushIntervalMs`: intervalo periódico de vaciado de telemetría en ms.
- `otel.captureContent`: captura opcional de contenido sin procesar para los atributos de segmentos de OTEL. Está desactivada de forma predeterminada. El valor booleano `true` captura el contenido de mensajes y herramientas que no sea del sistema; la forma de objeto permite activar explícitamente `inputMessages`, `outputMessages`, `toolInputs`, `toolOutputs`, `systemPrompt` y `toolDefinitions`.
- `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`: conmutador de entorno para la forma experimental más reciente de los segmentos de inferencia de GenAI, incluidos los nombres de segmento `{gen_ai.operation.name} {gen_ai.request.model}`, el tipo de segmento `CLIENT` y `gen_ai.provider.name` en lugar del valor heredado `gen_ai.system`. De forma predeterminada, los segmentos conservan `openclaw.model.call` y `gen_ai.system` por compatibilidad; las métricas de GenAI utilizan atributos semánticos acotados.
- `OPENCLAW_OTEL_PRELOADED=1`: conmutador de entorno para hosts que ya hayan registrado un SDK global de OpenTelemetry. En ese caso, OpenClaw omite el inicio y el cierre del SDK propiedad del Plugin, pero mantiene activos los receptores de diagnóstico.
- `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`, `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` y `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`: variables de entorno de endpoints específicas de cada señal que se utilizan cuando no se establece la clave de configuración correspondiente.
- `cacheTrace.enabled`: registra instantáneas de trazas de caché para ejecuciones integradas (valor predeterminado: `false`).
- `cacheTrace.filePath`: ruta de salida para el archivo JSONL de trazas de caché (valor predeterminado: `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl`).
- `cacheTrace.includeMessages` / `includePrompt` / `includeSystem`: controlan qué se incluye en la salida de trazas de caché (todos tienen como valor predeterminado `true`).

---

## Actualización

```json5
{
  update: {
    channel: "stable", // stable | extended-stable | beta | dev
    checkOnStart: true,

    auto: {
      enabled: false,
    },
  },
}
```

- `channel`: canal de publicación: `"stable"`, `"extended-stable"`, `"beta"` o `"dev"`. El canal estable ampliado solo está disponible como paquete: los comandos en primer plano se encargan de la instalación, mientras que el Gateway puede emitir avisos de actualización de solo lectura.
- `checkOnStart`: comprueba si hay actualizaciones de npm cuando se inicia el Gateway (valor predeterminado: `true`). Las selecciones almacenadas del canal estable ampliado utilizan el mismo aviso de solo lectura y la misma programación de avisos cada 24 horas.
- `auto.enabled`: activa la actualización automática en segundo plano para las instalaciones de paquetes de los canales estable y beta (valor predeterminado: `false`). El canal estable ampliado nunca se aplica automáticamente.

---

## ACP

```json5
{
  acp: {
    enabled: true,
    dispatch: { enabled: true },
    backend: "acpx",
    fallbacks: ["acpx-secondary"],
    defaultAgent: "main",
    allowedAgents: ["main", "ops"],
    stream: {
      repeatSuppression: true,
      deliveryMode: "live", // live | final_only
    },
  },
}
```

- `enabled`: control global de la función ACP (valor predeterminado: `true`; establezca `false` para ocultar las opciones de despacho y creación de ACP).
- `dispatch.enabled`: control independiente del despacho de turnos de sesiones ACP (valor predeterminado: `true`). Establezca `false` para mantener disponibles los comandos ACP mientras se bloquea la ejecución.
- `backend`: identificador predeterminado del backend de tiempo de ejecución de ACP (debe coincidir con un Plugin de tiempo de ejecución de ACP registrado).
  Instale primero el Plugin de backend y, si se establece `plugins.allow`, incluya el identificador del Plugin de backend (por ejemplo, `acpx`) o el backend de ACP no se cargará.
- `fallbacks`: lista ordenada de identificadores de backends de ACP alternativos que se prueban cuando el backend principal falla pronto con un error aparentemente transitorio (no disponible, limitado por tasa, cuota agotada o sobrecargado) antes de producir alguna salida. Cada entrada debe coincidir con un backend de Plugin de tiempo de ejecución de ACP registrado.
- `defaultAgent`: identificador del agente de destino alternativo de ACP cuando las creaciones no especifican un destino explícito.
- `allowedAgents`: lista de identificadores de agentes permitidos para las sesiones de tiempo de ejecución de ACP; si está vacía, no hay restricciones adicionales.
- `stream.repeatSuppression`: suprime las líneas repetidas de estado o herramientas en cada turno (valor predeterminado: `true`).
- `stream.deliveryMode`: `"live"` transmite de forma incremental; `"final_only"` almacena en búfer hasta que se producen los eventos terminales del turno.
- `stream.tagVisibility`: registro de nombres de etiquetas con anulaciones booleanas de visibilidad para los eventos transmitidos.
- `runtime.installCommand`: comando de instalación opcional que se ejecuta al inicializar un entorno de tiempo de ejecución de ACP.

---

## Asistente

Comportamiento y metadatos de los flujos de configuración guiada de la CLI (`onboard`, `configure`, `doctor`):

```json5
{
  wizard: {
    accessMode: "full",
    appRecommendations: true,
    lastRunAt: "2026-01-01T00:00:00.000Z",
    lastRunVersion: "2026.1.4",
    lastRunCommit: "abc1234",
    lastRunCommand: "configure",
    lastRunMode: "local",
    securityAcknowledgedAt: "2026-01-01T00:00:00.000Z",
  },
}
```

- `wizard.accessMode`: consentimiento de detección elegido al inicio de la incorporación guiada. `"full"` (recomendado) permite que la configuración busque automáticamente aplicaciones de IA, claves y entornos de ejecución locales; `"guarded"` hace que la configuración pregunte una vez antes de realizar la búsqueda y ofrece en su lugar la configuración manual.

- `wizard.appRecommendations` tiene como valor predeterminado `true`. Establézcalo en `false` para desactivar las recomendaciones de aplicaciones instaladas durante la incorporación guiada o clásica y bloquear el acceso `device.apps` del Gateway. Los hosts Node siguen necesitando su indicador independiente de uso compartido de aplicaciones instaladas, desactivado de forma predeterminada, antes de anunciar el comando.

---

## Identidad

Consulte los campos de identidad de `agents.entries` en [Valores predeterminados del agente](/es/gateway/config-agents#agent-defaults).

---

## Puente (heredado, eliminado)

Las compilaciones actuales ya no incluyen el puente TCP. Los nodos se conectan mediante el WebSocket del Gateway. Las claves `bridge.*` ya no forman parte del esquema de configuración (la validación falla hasta que se eliminan; `openclaw doctor --fix` puede quitar las claves desconocidas).

<Accordion title="Configuración heredada del puente (referencia histórica)">

```json
{
  "bridge": {
    "enabled": true,
    "port": 18790,
    "bind": "tailnet",
    "tls": {
      "enabled": true,
      "autoGenerate": true
    }
  }
}
```

</Accordion>

---

## Cron

```json5
{
  cron: {
    enabled: true,
    webhook: "https://example.invalid/legacy", // alternativa obsoleta para tareas almacenadas con notify:true
    webhookToken: "replace-with-dedicated-token", // token de portador opcional para la autenticación del webhook saliente
    sessionRetention: "24h", // cadena de duración o false
  },
}
```

- `sessionRetention`: cuánto tiempo se conservan las sesiones completadas de ejecuciones de Cron aisladas antes de depurar las filas de sesión de SQLite. También controla la limpieza de las transcripciones archivadas de Cron eliminadas. Valor predeterminado: `24h`; establezca `false` para desactivarlo.
- El historial de ejecuciones conserva automáticamente las 2000 filas de terminal más recientes por tarea. Las filas perdidas mantienen su periodo de limpieza de 24 horas.
- `webhookToken`: token de portador utilizado para la entrega POST del webhook de Cron (`delivery.mode = "webhook"`); si se omite, no se envía ninguna cabecera de autenticación.
- `webhook`: URL de webhook alternativa heredada y obsoleta (http/https) que utiliza `openclaw doctor --fix` para migrar las tareas almacenadas que aún tienen `notify: true`; la entrega en tiempo de ejecución utiliza el valor `delivery.mode="webhook"` de cada tarea junto con `delivery.to`, o `delivery.completionDestination` cuando se conserva la entrega de anuncios.

### `cron.failureAlert`

```json5
{
  cron: {
    failureAlert: {
      enabled: false,
      after: 3,
      cooldownMs: 3600000,
      includeSkipped: false,
      mode: "announce",
      accountId: "main",
    },
  },
}
```

- `enabled`: activa las alertas de fallo para las tareas de Cron (valor predeterminado: `false`).
- `after`: número de fallos consecutivos antes de que se active una alerta (entero positivo, mín.: `1`).
- `cooldownMs`: cantidad mínima de milisegundos entre alertas repetidas para la misma tarea (entero no negativo).
- `includeSkipped`: cuenta las ejecuciones omitidas consecutivas para el umbral de alerta (valor predeterminado: `false`). Las ejecuciones omitidas se registran por separado y no afectan al aumento progresivo del intervalo tras errores de ejecución.
- `mode`: modo de entrega: `"announce"` envía mediante un mensaje de canal; `"webhook"` publica en el webhook configurado.
- `accountId`: identificador opcional de cuenta o canal para limitar el ámbito de entrega de las alertas.

### `cron.failureDestination`

```json5
{
  cron: {
    failureDestination: {
      mode: "announce",
      channel: "last",
      to: "channel:C1234567890",
      accountId: "main",
    },
  },
}
```

- Destino predeterminado para las notificaciones de fallos de Cron en todas las tareas.
- `mode`: `"announce"` o `"webhook"`; el valor predeterminado es `"announce"` cuando hay suficientes datos del destino.
- `channel`: anulación del canal para la entrega de anuncios. `"last"` reutiliza el último canal de entrega conocido.
- `to`: destino explícito del anuncio o URL del webhook. Es obligatorio para el modo webhook.
- `accountId`: anulación opcional de la cuenta para la entrega.
- El valor `delivery.failureDestination` de cada tarea anula este valor predeterminado global.
- Cuando no se establece un destino de fallo global ni específico de la tarea, las tareas que ya realizan entregas mediante `announce` recurren a ese destino principal de anuncios en caso de fallo.
- `delivery.failureDestination` solo se admite para tareas `sessionTarget="isolated"`, salvo que el valor `delivery.mode` principal de la tarea sea `"webhook"`.

Consulte [Tareas de Cron](/es/automation/cron-jobs). Las ejecuciones aisladas de Cron se registran como [tareas en segundo plano](/es/automation/tasks).

## Variables de plantilla del modelo multimedia

Marcadores de posición de plantilla expandidos en `tools.media.models[].args`:

| Variable                    | Descripción                                       |
| --------------------------- | ------------------------------------------------- |
| `{{Body}}`                  | Cuerpo completo del mensaje entrante              |
| `{{RawBody}}`               | Cuerpo sin procesar (sin envoltorios de historial/remitente) |
| `{{BodyStripped}}`          | Cuerpo sin las menciones del grupo                |
| `{{From}}`                  | Identificador del remitente                       |
| `{{To}}`                    | Identificador del destino                         |
| `{{MessageSid}}`            | Identificador del mensaje del canal               |
| `{{SessionId}}`             | UUID de la sesión actual                          |
| `{{IsNewSession}}`          | `"true"` cuando se crea una sesión nueva          |
| `{{AttachmentUrl}}`         | URL del archivo adjunto actual o referencia del proveedor |
| `{{AttachmentPath}}`        | Ruta local del archivo adjunto actual             |
| `{{AttachmentContentType}}` | Tipo de contenido MIME del archivo adjunto actual |
| `{{AttachmentDir}}`         | Directorio que contiene `AttachmentPath`          |
| `{{AttachmentIndex}}`       | Índice del hecho de origen basado en cero          |
| `{{Transcript}}`            | Transcripción de audio                            |
| `{{Prompt}}`                | Prompt multimedia resuelto para entradas de la CLI |
| `{{MaxChars}}`              | Máximo resuelto de caracteres de salida para entradas de la CLI |
| `{{ChatType}}`              | `"direct"` o `"group"`                         |
| `{{GroupSubject}}`          | Asunto del grupo (mejor aproximación)              |
| `{{GroupMembers}}`          | Vista previa de los miembros del grupo (mejor aproximación) |
| `{{SenderName}}`            | Nombre para mostrar del remitente (mejor aproximación) |
| `{{SenderE164}}`            | Número de teléfono del remitente (mejor aproximación) |
| `{{Provider}}`              | Indicación del proveedor (whatsapp, telegram, discord, etc.) |

Los nombres heredados `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` y `{{MediaDir}}`
siguen disponibles durante el periodo de compatibilidad del SDK del plugin, pero están
obsoletos. La configuración nueva debe utilizar las variables `Attachment*`.

---

## Inclusiones de configuración (`$include`)

Divida la configuración en varios archivos:

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: {
    $include: ["./clients/mueller.json5", "./clients/schmidt.json5"],
  },
}
```

**Comportamiento de la combinación:**

- Archivo único: sustituye el objeto contenedor.
- Matriz de archivos: se combinan en profundidad en orden (los posteriores anulan a los anteriores).
- Claves hermanas: se combinan después de las inclusiones (anulan los valores incluidos).
- Inclusiones anidadas: hasta 10 niveles de profundidad.
- Rutas: se resuelven en relación con el archivo que realiza la inclusión, pero deben permanecer dentro del directorio de configuración de nivel superior (`dirname` de `openclaw.json`). Las formas absolutas/`../` solo se permiten cuando siguen resolviéndose dentro de ese límite. Establezca `OPENCLAW_INCLUDE_ROOTS` (rutas absolutas) para permitir raíces adicionales fuera del directorio de configuración.
- Límites: las rutas no deben contener bytes nulos y deben tener estrictamente menos de 4096 caracteres antes y después de la resolución; cada archivo incluido está limitado a 2 MB.
- Las escrituras propiedad de OpenClaw que solo cambian una sección de nivel superior respaldada por una inclusión de archivo único se escriben en ese archivo incluido. Por ejemplo, `plugins install` actualiza `plugins: { $include: "./plugins.json5" }` en `plugins.json5` y deja `openclaw.json` intacto.
- Las inclusiones raíz, las matrices de inclusiones y las inclusiones con anulaciones hermanas son de solo lectura para las escrituras propiedad de OpenClaw; dichas escrituras fallan de forma segura en lugar de aplanar la configuración.
- Errores: mensajes claros para archivos ausentes, errores de análisis, inclusiones circulares, formato de ruta no válido y longitud excesiva.

---

## Relacionado

- [Configuración](/es/gateway/configuration)
- [Ejemplos de configuración](/es/gateway/configuration-examples)
- [Doctor](/es/gateway/doctor)
