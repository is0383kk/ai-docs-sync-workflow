---
read_when:
    - Configurar un Plugin de canal (autenticación, control de acceso, varias cuentas)
    - Solución de problemas de las claves de configuración por canal
    - Auditoría de la política de mensajes directos, la política de grupos o el filtrado de menciones
summary: 'Configuración de canales: control de acceso, vinculación y claves por canal en Slack, Discord, Telegram, WhatsApp, Matrix, iMessage y más'
title: Configuración — canales
x-i18n:
    generated_at: "2026-07-26T05:39:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e346648287d275d84a9c082a3bb13edaee751d53546d8231dcf1525bf9adafc2
    source_path: gateway/config-channels.md
    workflow: 16
---

Per-channel configuration keys under `channels.*`: acceso a mensajes directos y grupos, configuraciones de varias cuentas, control por menciones y claves por canal para Slack, Discord, Telegram, WhatsApp, Matrix, iMessage y otros plugins de canal.

Para agentes, herramientas, el entorno de ejecución del Gateway y otras claves de nivel superior, consulte la [Referencia de configuración](/es/gateway/configuration-reference).

## Canales

Cada canal se inicia automáticamente cuando existe su sección de configuración (a menos que `enabled: false`). Telegram e iMessage se incluyen en el paquete principal `openclaw`. Otros canales oficiales (Discord, Slack, WhatsApp, Matrix, Microsoft Teams, IRC, Google Chat, Signal, Mattermost y más) se instalan como plugins independientes con `openclaw plugins install <spec>`; consulte [Canales](/es/channels) para ver la lista completa y las especificaciones de instalación.

### Acceso a mensajes directos y grupos

Todos los canales admiten políticas de mensajes directos y políticas de grupos:

| Política de mensajes directos | Comportamiento                                                        |
| ------------------- | --------------------------------------------------------------- |
| `pairing` (predeterminada) | Los remitentes desconocidos reciben un código de vinculación de un solo uso; el propietario debe aprobarlo |
| `allowlist`         | Solo los remitentes incluidos en `allowFrom` (o en el almacén de permitidos vinculados)             |
| `open`              | Permite todos los mensajes directos entrantes (requiere `allowFrom: ["*"]`)             |
| `disabled`          | Ignora todos los mensajes directos entrantes                                          |

| Política de grupos          | Comportamiento                                               |
| --------------------- | ------------------------------------------------------ |
| `allowlist` (predeterminada) | Solo los grupos que coincidan con la lista de permitidos configurada          |
| `open`                | Omite las listas de grupos permitidos (el control por menciones sigue aplicándose) |
| `disabled`            | Bloquea todos los mensajes de grupos o salas                          |

<Note>
`channels.defaults.groupPolicy` establece el valor predeterminado cuando no se ha definido `groupPolicy` de un proveedor.
Los códigos de vinculación caducan después de 1 hora. Las solicitudes de vinculación pendientes tienen un límite de **3 por cuenta** (con ámbito por canal e identificador de cuenta).
Si falta por completo el bloque de un proveedor (`channels.<provider>` ausente), la política de grupos del entorno de ejecución recurre a `allowlist` (cierre seguro) con una advertencia al iniciar.
</Note>

### Sustituciones del modelo por canal

Use `channels.modelByChannel` para fijar identificadores de canal o interlocutores de mensajes directos específicos a un modelo. Los valores aceptan `provider/model` o alias de modelos configurados. La asignación del canal solo se aplica cuando una sesión aún no tiene una sustitución de modelo activa (por ejemplo, una establecida mediante `/model`).

Para conversaciones de grupo o hilo, las claves son identificadores de grupo, identificadores de tema o nombres de canal específicos del canal. Para conversaciones de mensajes directos (DM), las claves son identificadores del interlocutor derivados de la identidad del remitente del canal (`nativeDirectUserId`, `origin.from`, `origin.to`, `OriginatingTo`, `From` o `SenderId`). La forma exacta de la clave depende del canal:

| Canal  | Forma de la clave de DM         | Ejemplo                                      |
| -------- | ------------------- | -------------------------------------------- |
| Discord  | identificador de usuario sin procesar         | `987654321`                                  |
| Feishu   | `feishu:ou_...`     | `feishu:ou_a8b6cab7e945387de5f253775d9b4d85` |
| Matrix   | identificador de usuario de Matrix      | `@user:matrix.org`                           |
| Slack    | `user:U...`         | `user:U12345`                                |
| Telegram | identificador de usuario sin procesar         | `123456789`                                  |
| WhatsApp | número de teléfono o JID | `15551234567`                                |

```json5
{
  channels: {
    modelByChannel: {
      discord: {
        "123456789012345678": "anthropic/claude-opus-4-6",
      },
      slack: {
        C1234567890: "openai/gpt-5.6-sol",
        "user:U12345": "openai/gpt-5.4-mini",
      },
      telegram: {
        "-1001234567890": "openai/gpt-5.4-mini",
        "-1001234567890:topic:99": "anthropic/claude-sonnet-4-6",
        "123456789": "openai/gpt-4.1",
      },
    },
  },
}
```

Las claves específicas de DM solo coinciden en conversaciones de mensajes directos; no afectan al enrutamiento de grupos o hilos.

### Valores predeterminados de los canales y Heartbeat

Use `channels.defaults` para compartir el comportamiento de la política de grupos, las menciones implícitas y Heartbeat entre proveedores:

```json5
{
  channels: {
    defaults: {
      groupPolicy: "allowlist", // abierto | lista de permitidos | deshabilitado
      contextVisibility: "all", // todo | lista de permitidos | cita de lista de permitidos
      implicitMentions: {
        replyToBot: true,
        quotedBot: true,
        threadParticipation: true,
      },
      heartbeat: {
        showOk: false,
        showAlerts: true,
        useIndicator: true,
      },
    },
  },
}
```

- `channels.defaults.groupPolicy`: política de grupos de reserva cuando no se ha definido `groupPolicy` en el nivel del proveedor.
- `channels.defaults.contextVisibility`: modo predeterminado de visibilidad del contexto adicional para todos los canales. Valores: `all` (predeterminado, incluye todo el contexto de citas, hilos e historial), `allowlist` (solo incluye el contexto de remitentes incluidos en la lista de permitidos), `allowlist_quote` (igual que la lista de permitidos, pero conserva el contexto explícito de citas o respuestas). Sustitución por canal: `channels.<channel>.contextVisibility`.
- `channels.defaults.implicitMentions`: controla qué hechos entrantes compatibles cuentan como menciones. `replyToBot`, `quotedBot` y `threadParticipation` tienen cada uno el valor predeterminado `true`, lo que conserva el comportamiento actual. Sustituya el valor por canal con `channels.<channel>.implicitMentions` o por cuenta con `channels.<channel>.accounts.<id>.implicitMentions`; cada indicador se resuelve de forma independiente siguiendo cuenta -> canal -> valores predeterminados. Los nombres son positivos: establezca un indicador en `false` para impedir que ese hecho omita el control por menciones. Las menciones explícitas nativas siempre están permitidas, y un indicador no tiene efecto cuando el canal no produce ese hecho. Consulte [Control por menciones](/es/channels/groups#mention-gating-default) para ver la matriz actual de productores. Estos ajustes no cambian los modos de respuesta o hilo salientes ni la gestión de comandos autorizados.
- `channels.defaults.heartbeat.showOk`: incluye los estados saludables de los canales en la salida de Heartbeat (valor predeterminado: `false`).
- `channels.defaults.heartbeat.showAlerts`: incluye los estados degradados o de error en la salida de Heartbeat (valor predeterminado: `true`).
- `channels.defaults.heartbeat.useIndicator`: representa la salida de Heartbeat en un estilo compacto de indicador (valor predeterminado: `true`).

### WhatsApp

WhatsApp se ejecuta mediante el canal web del Gateway (Baileys Web). Se inicia automáticamente cuando existe una sesión vinculada.

```json5
{
  web: {
    enabled: true,
  },
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // vinculación | lista de permitidos | abierto | deshabilitado
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      streaming: { chunkMode: "length" }, // longitud | nueva línea
      mediaMaxMb: 50,
      sendReadReceipts: true, // marcas azules (false en el modo de chat consigo mismo)
      groups: {
        "*": { requireMention: true },
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

- Las entradas `bindings[]` de nivel superior con `type: "acp"` configuran vinculaciones ACP persistentes para mensajes directos y grupos de WhatsApp. Use un número directo E.164 o un JID de grupo de WhatsApp en `match.peer.id`. La semántica de los campos se comparte en [Agentes ACP](/es/tools/acp-agents#persistent-channel-bindings).

<Accordion title="WhatsApp con varias cuentas">

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {},
        personal: {},
        biz: {
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

- Los comandos salientes usan de forma predeterminada la cuenta `default` si existe; de lo contrario, el primer identificador de cuenta configurado (ordenado).
- El valor opcional `channels.whatsapp.defaultAccount` sustituye esa selección de cuenta predeterminada de reserva cuando coincide con un identificador de cuenta configurado.
- El directorio de autenticación heredado de Baileys para una sola cuenta se migra mediante `openclaw doctor` a `whatsapp/default`.
- Sustituciones por cuenta: `channels.whatsapp.accounts.<id>.sendReadReceipts`, `channels.whatsapp.accounts.<id>.dmPolicy`, `channels.whatsapp.accounts.<id>.allowFrom`.

</Accordion>

### Telegram

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "your-bot-token",
      dmPolicy: "pairing",
      allowFrom: ["tg:123456789"],
      groups: {
        "*": { requireMention: true },
        "-1001234567890": {
          allowFrom: ["@admin"],
          systemPrompt: "Keep answers brief.",
          topics: {
            "99": {
              requireMention: false,
              skills: ["search"],
              systemPrompt: "Stay on topic.",
            },
          },
        },
      },
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
      historyLimit: 50,
      replyToMode: "first", // off | first | all | batched
      linkPreview: true,
      streaming: { mode: "partial" }, // off | partial | block | progress (default: partial)
      actions: { reactions: true, sendMessage: true },
      reactionNotifications: "own", // off | own | all
      mediaMaxMb: 100,
      retry: {
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
      network: {
        autoSelectFamily: true,
        dnsResultOrder: "ipv4first",
      },
      apiRoot: "https://api.telegram.org",
      trustedLocalFileRoots: ["/srv/telegram-bot-api-data"],
      proxy: "socks5://localhost:9050",
      webhookUrl: "https://example.com/telegram-webhook",
      webhookSecret: "secret",
      webhookPath: "/telegram-webhook",
    },
  },
}
```

- Token del bot: `channels.telegram.botToken` o `channels.telegram.tokenFile` (solo archivos normales; se rechazan los enlaces simbólicos), con `TELEGRAM_BOT_TOKEN` como alternativa para la cuenta predeterminada.
- `apiRoot` es únicamente la raíz de la API de bots de Telegram. Use `https://api.telegram.org` o la raíz de su servidor autoalojado o proxy, no `https://api.telegram.org/bot<TOKEN>`; `openclaw doctor --fix` elimina un sufijo final `/bot<TOKEN>` añadido por error.
- Para un servidor de API de bots autoalojado en modo `--local`, `trustedLocalFileRoots` enumera las rutas del sistema anfitrión que OpenClaw puede leer. Monte el volumen de datos del servidor en el sistema anfitrión de OpenClaw y configure su raíz de datos o el directorio por token; las rutas del contenedor bajo `/var/lib/telegram-bot-api` se asignan a esas raíces. Las demás rutas absolutas siguen rechazándose.
- El valor opcional `channels.telegram.defaultAccount` sustituye la selección de cuenta predeterminada cuando coincide con un identificador de cuenta configurado.
- En configuraciones de varias cuentas (2 o más identificadores de cuenta), establezca un valor predeterminado explícito (`channels.telegram.defaultAccount` o `channels.telegram.accounts.default`) para evitar el enrutamiento de reserva; `openclaw doctor` advierte cuando falta o no es válido.
- `configWrites: false` bloquea las escrituras de configuración iniciadas por Telegram (migraciones de identificadores de supergrupos, `/config set|unset`).
- Las entradas `bindings[]` de nivel superior con `type: "acp"` configuran vinculaciones ACP persistentes para temas de foros (use el valor canónico `chatId:topic:topicId` en `match.peer.id`). La semántica de los campos se comparte en [Agentes ACP](/es/tools/acp-agents#persistent-channel-bindings).
- Las vistas previas de transmisión de Telegram usan `sendMessage` + `editMessageText` (funciona en chats directos y grupales).
- `network.dnsResultOrder` tiene como valor predeterminado `"ipv4first"` para evitar errores frecuentes de obtención mediante IPv6.
- Política de reintentos: consulte [Política de reintentos](/es/concepts/retry).

### Discord

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",
      mediaMaxMb: 100,
      allowBots: false,
      actions: {
        reactions: true,
        stickers: true,
        polls: true,
        permissions: true,
        messages: true,
        threads: true,
        pins: true,
        search: true,
        memberInfo: true,
        roleInfo: true,
        roles: false,
        channelInfo: true,
        voiceStatus: true,
        events: true,
        moderation: false,
      },
      replyToMode: "off", // off | first | all | batched
      dmPolicy: "pairing",
      allowFrom: ["1234567890", "123456789012345678"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["openclaw-dm"] },
      guilds: {
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          ignoreOtherMentions: true,
          reactionNotifications: "own",
          users: ["987654321098765432"],
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["987654321098765432"],
              skills: ["docs"],
              systemPrompt: "Solo respuestas breves.",
            },
          },
        },
      },
      historyLimit: 20,
      textChunkLimit: 2000,
      suppressEmbeds: true,
      streaming: {
        mode: "progress", // off | partial | block | progress (valor predeterminado de Discord: progress)
        chunkMode: "length", // length | newline
        progress: {
          label: "auto",
          maxLines: 8,
          maxLineChars: 120,
          toolProgress: true,
        },
      },
      maxLinesPerMessage: 17,
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSessions: true,
        defaultSpawnContext: "fork",
      },
      voice: {
        enabled: true,
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        connectTimeoutMs: 30000,
        reconnectGraceMs: 15000,
        tts: {
          provider: "openai",
          openai: { voice: "alloy" },
        },
      },
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["987654321098765432"],
        agentFilter: ["default"],
        sessionFilter: ["discord:"],
        target: "dm", // dm | channel | both
        cleanupAfterResolve: false,
      },
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

- Token: `channels.discord.token`, con `DISCORD_BOT_TOKEN` como alternativa para la cuenta predeterminada.
- Las llamadas salientes directas que proporcionan un `token` de Discord explícito usan ese token para la llamada; la configuración de reintentos y políticas de la cuenta sigue procediendo de la cuenta seleccionada en la instantánea activa del entorno de ejecución.
- El valor opcional `channels.discord.defaultAccount` sustituye la selección de la cuenta predeterminada cuando coincide con el id de una cuenta configurada.
- Use `user:<id>` (mensaje directo) o `channel:<id>` (canal de servidor) como destinos de entrega; los identificadores numéricos sin prefijo se rechazan.
- Los slugs de los servidores se escriben en minúsculas y sustituyen los espacios por `-`; las claves de los canales usan el nombre convertido en slug (sin `#`). Es preferible usar los identificadores de los servidores.
- Los mensajes creados por bots se ignoran de forma predeterminada. `allowBots: true` los habilita; use `allowBots: "mentions"` para aceptar únicamente mensajes de bots que mencionen al bot (los mensajes propios siguen filtrándose).
- Los canales que admiten mensajes entrantes creados por bots pueden usar la [protección compartida contra bucles de bots](/es/channels/bot-loop-protection). Configure `channels.defaults.botLoopProtection` para los presupuestos de pares de referencia y, después, sustituya el canal o la cuenta únicamente cuando una superficie necesite límites diferentes.
- `channels.discord.guilds.<id>.ignoreOtherMentions` (y las sustituciones por canal) descarta los mensajes que mencionan a otro usuario o rol, pero no al bot (salvo @everyone/@here).
- `channels.discord.mentionAliases` asigna el texto saliente estable `@handle` a identificadores de usuarios de Discord antes del envío, de modo que se pueda mencionar de forma determinista a compañeros conocidos incluso cuando la caché transitoria del directorio esté vacía. Las sustituciones por cuenta se encuentran en `channels.discord.accounts.<accountId>.mentionAliases`.
- `maxLinesPerMessage` (valor predeterminado: `17`) divide los mensajes altos incluso cuando tienen menos de 2000 caracteres.
- `channels.discord.suppressEmbeds` tiene como valor predeterminado `true`, por lo que las URL salientes no se expanden en vistas previas de enlaces de Discord salvo que se deshabilite. Las cargas `embeds` explícitas siguen enviándose con normalidad; las llamadas a herramientas por mensaje pueden sustituirlo mediante `suppressEmbeds`.
- `channels.discord.threadBindings` controla el enrutamiento vinculado a hilos de Discord:
  - `enabled`: sustitución de Discord para las funciones de sesión vinculadas a hilos (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` y entrega/enrutamiento vinculados)
  - `idleHours`: sustitución de Discord para la pérdida automática del foco por inactividad, en horas (`0` la deshabilita)
  - `maxAgeHours`: sustitución de Discord para la antigüedad máxima estricta, en horas (`0` la deshabilita)
  - `spawnSessions`: interruptor para `sessions_spawn({ thread: true })` y la creación/vinculación automática de hilos al generar hilos de ACP (valor predeterminado: `true`)
  - `defaultSpawnContext`: contexto nativo de subagente para generaciones vinculadas a hilos (`"fork"` de forma predeterminada)
- Las entradas `bindings[]` de nivel superior con `type: "acp"` configuran vinculaciones persistentes de ACP para canales e hilos (use el id del canal/hilo en `match.peer.id`). La semántica de los campos se comparte en [Agentes ACP](/es/tools/acp-agents#persistent-channel-bindings).
- `channels.discord.ui.components.accentColor` establece el color de énfasis de los contenedores de componentes v2 de Discord.
- `channels.discord.agentComponents.ttlMs` controla durante cuánto tiempo permanecen registrados los callbacks enviados de componentes de Discord. Valor predeterminado: `1800000` (30 minutos); máximo: `86400000` (24 horas). Las sustituciones por cuenta se encuentran en `channels.discord.accounts.<accountId>.agentComponents.ttlMs`. Es preferible usar el TTL más corto que se ajuste al flujo de trabajo.
- `channels.discord.voice` habilita las conversaciones en canales de voz de Discord y las sustituciones opcionales de unión automática, LLM y TTS. Las configuraciones de Discord exclusivamente de texto mantienen la voz desactivada de forma predeterminada; configure `channels.discord.voice.enabled=true` para habilitarla.
- `channels.discord.voice.model` sustituye opcionalmente el modelo LLM utilizado para las respuestas de los canales de voz de Discord.
- `channels.discord.voice.daveEncryption` (valor predeterminado: `true`) y `channels.discord.voice.decryptionFailureTolerance` (valor predeterminado: `24`) se transfieren a las opciones DAVE de `@discordjs/voice`.
- `channels.discord.voice.connectTimeoutMs` controla la espera inicial de `@discordjs/voice` Ready para `/vc join` y los intentos de unión automática (valor predeterminado: `30000`).
- `channels.discord.voice.reconnectGraceMs` controla cuánto tiempo puede tardar una sesión de voz desconectada en entrar en la señalización de reconexión antes de que OpenClaw la destruya (valor predeterminado: `15000`).
- La reproducción de voz de Discord no se interrumpe cuando otro usuario empieza a hablar. Para evitar bucles de retroalimentación, OpenClaw ignora las nuevas capturas de voz mientras se reproduce TTS.
- Además, OpenClaw intenta recuperar la recepción de voz abandonando una sesión de voz y volviendo a unirse a ella tras repetidos fallos de descifrado.
- `channels.discord.streaming` es la clave canónica del modo de transmisión. El valor predeterminado de Discord es `streaming.mode: "progress"`, por lo que el progreso de las herramientas y del trabajo aparece en un único mensaje de vista previa editado; configure `streaming.mode: "off"` para deshabilitarlo. Las claves planas heredadas (`streamMode`, `chunkMode`, `blockStreaming`, `draftChunk`, `blockStreamingCoalesce`) ya no se leen durante la ejecución; ejecute `openclaw doctor --fix` para migrar la configuración persistente.
- `channels.discord.autoPresence` asigna la disponibilidad del entorno de ejecución a la presencia del bot (correcto => en línea, degradado => inactivo, agotado => no molestar) y permite sustituciones opcionales del texto de estado.
- `channels.discord.guilds.<id>.presenceEvents` dirige las llegadas de disponibilidad de personas a un canal de Discord configurado como eventos del sistema del agente. Los miembros aptos deben poder ver `channelId`; los hilos públicos heredan la visibilidad del canal principal, mientras que los hilos privados requieren además pertenecer al hilo o el permiso Manage Threads. `users` puede restringir aún más ese público. Inicializa los miembros actualmente en línea a partir de instantáneas `GUILD_CREATE` completas, dirige las transiciones observadas de desconectado a conectado y trata una primera señal posterior de conexión de un miembro no visto como una nueva disponibilidad, sin afirmar si se conectó o se unió después de la instantánea. Los servidores que superen el límite de instantáneas de 75,000 miembros de Discord requieren primero una actualización explícita de desconexión. Controles de limitación: `reconnectSuppressSeconds` (ventana de espera después de una nueva sesión del Gateway mientras se reconstruye el estado de presencia del servidor; valor predeterminado: 300; `0` la deshabilita) y `burstLimit`/`burstWindowSeconds` (límite por servidor de eventos puestos en cola correctamente; valor predeterminado: 8 eventos por ventana móvil de 60s). Las sesiones reanudadas no inician la ventana de supresión de reconexión. El tiempo de espera existente para volver a saludar a cada usuario sigue siendo de ocho horas. Requiere `channels.discord.intents.presence=true`, el Presence Intent con privilegios del Developer Portal de Discord y un Heartbeat de agente habilitado.
- `channels.discord.dangerouslyAllowNameMatching` vuelve a habilitar la coincidencia mutable de nombres/etiquetas (modo de compatibilidad de emergencia).
- `channels.discord.execApprovals`: entrega nativa de Discord para aprobaciones de ejecución y autorización de aprobadores.
  - `enabled`: `true`, `false` o `"auto"` (valor predeterminado). En modo automático, las aprobaciones de ejecución se activan cuando los aprobadores pueden resolverse a partir de `approvers` o `commands.ownerAllowFrom`.
  - `approvers`: identificadores de usuarios de Discord autorizados para aprobar solicitudes de ejecución. Si se omite, usa `commands.ownerAllowFrom` como alternativa.
  - `agentFilter`: lista opcional de identificadores de agentes permitidos. Omítala para reenviar aprobaciones de todos los agentes.
  - `sessionFilter`: patrones opcionales de claves de sesión (subcadena o expresión regular).
  - `target`: dónde enviar las solicitudes de aprobación. `"dm"` (valor predeterminado) las envía a los mensajes directos de los aprobadores, `"channel"` las envía al canal de origen y `"both"` las envía a ambos. Cuando el destino incluye `"channel"`, solo los aprobadores resueltos pueden usar los botones.
  - `cleanupAfterResolve`: cuando es `true`, elimina los mensajes directos de aprobación tras la aprobación, la denegación o el tiempo de espera agotado.

**Modos de notificación de reacciones:** `off` (ninguno), `own` (mensajes del bot, valor predeterminado), `all` (todos los mensajes), `allowlist` (de `guilds.<id>.users` en todos los mensajes).

### Google Chat

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url", // app-url | project-number
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890",
      dmPolicy: "pairing",
      allowFrom: ["users/1234567890"],
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": { allow: true, requireMention: true },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

- JSON de la cuenta de servicio: en línea (`serviceAccount`) o basado en archivos (`serviceAccountFile`).
- `serviceAccount` acepta directamente una SecretRef.
- Alternativas de entorno: `GOOGLE_CHAT_SERVICE_ACCOUNT` o `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE` (solo la cuenta predeterminada).
- Use `spaces/<spaceId>` o `users/<userId>` como destinos de entrega.
- `channels.googlechat.dangerouslyAllowNameMatching` vuelve a habilitar la coincidencia mutable de principales de correo electrónico (modo de compatibilidad de emergencia).

### Slack

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      appToken: "xapp-...",
      socketMode: {
        clientPingTimeout: 15000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
      dmPolicy: "pairing",
      allowFrom: ["U123", "U456", "*"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["G123"] },
      channels: {
        C123: { enabled: true, requireMention: true, allowBots: false },
        "#general": {
          enabled: true,
          requireMention: true,
          allowBots: false,
          users: ["U123"],
          skills: ["docs"],
          systemPrompt: "Solo respuestas breves.",
        },
      },
      historyLimit: 50,
      allowBots: false,
      reactionNotifications: "own",
      reactionAllowlist: ["U123"],
      replyToMode: "off", // off | first | all | batched
      thread: {
        historyScope: "thread", // thread | channel
        inheritParent: false,
        initialHistoryLimit: 20,
      },
      actions: {
        reactions: true,
        messages: true,
        pins: true,
        memberInfo: true,
        emojiList: true,
      },
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
      typingReaction: "hourglass_flowing_sand",
      unfurlLinks: false,
      unfurlMedia: false,
      textChunkLimit: 4000,
      streaming: {
        mode: "partial", // off | partial | block | progress
        chunkMode: "length", // length | newline
        nativeTransport: true, // usar la API de streaming nativa de Slack cuando mode=partial
      },
      mediaMaxMb: 20,
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["U123"],
        agentFilter: ["default"],
        sessionFilter: ["slack:"],
        target: "dm", // dm | channel | both
      },
    },
  },
}
```

- El **modo Socket** requiere tanto `botToken` como `appToken` (`SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` para recurrir de forma predeterminada a las variables de entorno de la cuenta).
- El **modo HTTP** requiere `botToken` además de `signingSecret` (en la raíz o por cuenta).
- La **identidad de usuario** (`identity: "user"`) publica y lee como la persona que concede la autorización. Requiere `userToken` además de `appToken` en el modo Socket, o `userToken` además de `signingSecret` en el modo HTTP. No se requiere ningún token de bot ni usuario de bot. Consulte [Identidad de usuario](/es/channels/slack#user-identity-post-as-a-real-person) para conocer los ámbitos de usuario y las suscripciones a eventos.
- `enterpriseOrgInstall: true` incorpora una cuenta a la ruta de eventos de toda la organización de Slack Enterprise Grid.
  Durante el inicio, se verifica el token del bot con `auth.test` y
  se produce un error cuando el modo configurado no coincide con la identidad de instalación de Slack.
  Los mensajes directos empresariales deben estar desactivados o usar `dmPolicy: "open"` con un
  `allowFrom: ["*"]` efectivo. Las políticas de canales y usuarios deben usar ID estables de Slack;
  los nombres mutables y los prefijos de canal no compatibles provocan un error de inicio. V1 solo gestiona
  eventos directos `message` y `app_mention` del modo Socket o HTTP con respuestas
  inmediatas; no están disponibles la retransmisión, los comandos, las interacciones, App Home, los escuchadores de eventos de reacción,
  los elementos fijados, las herramientas de acciones, las aprobaciones nativas, las vinculaciones, la entrega diferida ni
  los envíos proactivos. Las confirmaciones, la indicación de escritura y las reacciones de estado gestionadas por el escuchador
  siguen disponibles con `reactions:write`; las notificaciones de reacciones entrantes
  y las herramientas de acciones de reacción no están disponibles. Consulte
  [Instalaciones de Enterprise Grid para toda la organización](/es/channels/slack#enterprise-grid-org-wide-installs)
  para conocer el manifiesto de privilegios mínimos, el flujo de configuración y todas las restricciones.
- `socketMode` transmite los ajustes del transporte del modo Socket del SDK de Slack a la API pública del receptor Bolt. Úselo solo al investigar tiempos de espera de ping/pong o un comportamiento obsoleto del websocket. `clientPingTimeout` tiene como valor predeterminado `15000`; `serverPingTimeout` y `pingPongLoggingEnabled` solo se transmiten cuando están configurados.
- `botToken`, `appToken`, `signingSecret` y `userToken` aceptan cadenas
  de texto sin formato u objetos SecretRef.
- Las instantáneas de cuentas de Slack exponen campos de origen/estado por credencial, como
  `botTokenSource`, `botTokenStatus`, `userTokenSource`, `userTokenStatus`,
  `appTokenStatus` y, en el modo HTTP, `signingSecretStatus`.
  `configured_unavailable` significa que la cuenta está
  configurada mediante SecretRef, pero la ruta actual del comando o del entorno de ejecución no pudo
  resolver el valor secreto.
- `configWrites: false` bloquea las escrituras de configuración iniciadas por Slack.
- El valor opcional `channels.slack.defaultAccount` sustituye la selección de cuenta predeterminada cuando coincide con el ID de una cuenta configurada.
- `channels.slack.streaming.mode` es la clave canónica del modo de transmisión de Slack (valor predeterminado: `"partial"`). `channels.slack.streaming.nativeTransport` controla el transporte de streaming nativo de Slack (valor predeterminado: `true`). Los valores heredados `streamMode`, el booleano `streaming`, `chunkMode`, `blockStreaming`, `blockStreamingCoalesce` y `nativeStreaming` ya no se leen durante la ejecución; ejecute `openclaw doctor --fix` para migrar la configuración persistente a `streaming.{mode,chunkMode,block.enabled,block.coalesce,nativeTransport}`.
- `unfurlLinks` y `unfurlMedia` transmiten los booleanos `chat.postMessage` de Slack para desplegar enlaces y contenido multimedia en las respuestas del bot. `unfurlLinks` tiene como valor predeterminado `false`, por lo que los enlaces salientes del bot no se expanden en línea salvo que se habilite esta opción; `unfurlMedia` se omite salvo que esté configurado. Establezca cualquiera de los valores en `channels.slack.accounts.<accountId>` para sustituir el valor de nivel superior en una cuenta.
- Use `user:<id>` (mensaje directo) o `channel:<id>` como destinos de entrega.

**Modos de notificación de reacciones:** `off`, `own` (predeterminado), `all`, `allowlist` (de `reactionAllowlist`).

**Aislamiento de sesiones de hilos:** `thread.historyScope` es por hilo (valor predeterminado) o compartido en todo el canal. `thread.inheritParent` copia la transcripción del canal principal en los hilos nuevos. `thread.initialHistoryLimit` (valor predeterminado: `20`) limita cuántos mensajes existentes del hilo se recuperan cuando se inicia una sesión de hilo nueva; `0` desactiva la recuperación del historial del hilo.

- El streaming nativo de Slack y el estado de hilo «está escribiendo...» al estilo del asistente de Slack requieren un hilo de respuesta como destino. Los mensajes directos de nivel superior permanecen fuera de los hilos de forma predeterminada, por lo que aún pueden transmitirse mediante las vistas previas de borrador con publicación y edición de Slack, en lugar de mostrar la vista previa de streaming/estado nativa propia de los hilos.
- `typingReaction` añade una reacción temporal al mensaje entrante de Slack mientras se genera una respuesta y la elimina al finalizar. Use un código corto de emoji de Slack, como `"hourglass_flowing_sand"`.
- `channels.slack.execApprovals`: entrega del cliente de aprobaciones nativo de Slack y autorización de los aprobadores de ejecución. Usa el mismo esquema que Discord: `enabled` (`true`/`false`/`"auto"`), `approvers` (ID de usuario de Slack), `agentFilter`, `sessionFilter` y `target` (`"dm"`, `"channel"` o `"both"`). Las aprobaciones de plugins pueden usar esta ruta de cliente nativa para solicitudes originadas en Slack cuando se resuelven los aprobadores del plugin de Slack; la entrega de aprobaciones de plugins nativa de Slack también puede habilitarse mediante `approvals.plugin` para sesiones originadas en Slack o destinos de Slack. Las aprobaciones de plugins usan los aprobadores del plugin de Slack de `allowFrom` y el enrutamiento predeterminado, no los aprobadores de ejecución.

| Grupo de acciones | Valor predeterminado | Notas                              |
| ----------------- | -------------------- | ---------------------------------- |
| reactions         | habilitado           | Reaccionar + enumerar reacciones   |
| messages          | habilitado           | Leer/enviar/editar/eliminar        |
| pins              | habilitado           | Fijar/desfijar/enumerar            |
| memberInfo        | habilitado           | Información del miembro            |
| emojiList         | habilitado           | Lista de emojis personalizados     |

### Mattermost

Mattermost se instala como un plugin independiente, del mismo modo que Discord, Slack y WhatsApp:

```bash
openclaw plugins install @openclaw/mattermost
```

Consulte [npmjs.com/package/@openclaw/mattermost](https://www.npmjs.com/package/@openclaw/mattermost) para conocer las etiquetas de distribución actuales antes de fijar una versión.

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
      chatmode: "oncall", // oncall | onmessage | onchar
      oncharPrefixes: [">", "!"],
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
      commands: {
        native: true, // activación voluntaria
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // URL explícita opcional para implementaciones públicas o con proxy inverso
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      streaming: { chunkMode: "length" },
    },
  },
}
```

Modos de chat: `oncall` (responder al recibir una @mención; valor predeterminado), `onmessage` (cada mensaje), `onchar` (mensajes que comienzan con el prefijo de activación).

Cuando los comandos nativos de Mattermost están habilitados:

- `commands.callbackPath` debe ser una ruta (por ejemplo, `/api/channels/mattermost/command`), no una URL completa.
- `commands.callbackUrl` debe resolverse en el punto de conexión del Gateway de OpenClaw y ser accesible desde el servidor de Mattermost.
- Las llamadas de retorno de comandos de barra nativos se autentican con los tokens de cada comando que devuelve
  Mattermost durante el registro de los comandos de barra. Si el registro falla o no se activa
  ningún comando, OpenClaw rechaza las llamadas de retorno con
  `Unauthorized: invalid command token.`
- Para hosts de llamadas de retorno privados, de una tailnet o internos, Mattermost puede requerir
  que `ServiceSettings.AllowedUntrustedInternalConnections` incluya el host o dominio de la llamada de retorno.
  Use valores de host o dominio, no URL completas.
- `channels.mattermost.configWrites`: permitir o denegar las escrituras de configuración iniciadas por Mattermost.
- `channels.mattermost.requireMention`: requerir `@mention` antes de responder en los canales.
- `channels.mattermost.groups.<channelId>.requireMention`: sustitución por canal del requisito de mención (`"*"` como valor predeterminado).
- El valor opcional `channels.mattermost.defaultAccount` sustituye la selección de cuenta predeterminada cuando coincide con el ID de una cuenta configurada.

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // vinculación opcional de la cuenta
      dmPolicy: "pairing",
      allowFrom: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      configWrites: true,
      reactionNotifications: "own", // off | own | all | allowlist
      reactionAllowlist: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      historyLimit: 50,
    },
  },
}
```

**Modos de notificación de reacciones:** `off`, `own` (predeterminado), `all`, `allowlist` (de `reactionAllowlist`).

- `channels.signal.account`: fijar el inicio del canal a una identidad de cuenta específica de Signal.
- `channels.signal.configWrites`: permitir o denegar las escrituras de configuración iniciadas por Signal.
- El valor opcional `channels.signal.defaultAccount` sustituye la selección de cuenta predeterminada cuando coincide con el ID de una cuenta configurada.

### iMessage

OpenClaw inicia `imsg rpc` (JSON-RPC mediante la entrada/salida estándar). No se requiere ningún daemon ni puerto. Esta es la ruta preferida para las nuevas configuraciones de iMessage en OpenClaw cuando el host puede conceder permisos para la base de datos de Mensajes y Automatización.

Se eliminó la compatibilidad con BlueBubbles. `channels.bluebubbles` no es una superficie de configuración del entorno de ejecución compatible con la versión actual de OpenClaw. Migre las configuraciones antiguas a `channels.imessage`; consulte [Eliminación de BlueBubbles y la ruta imsg de iMessage](/es/announcements/bluebubbles-imessage) para ver la versión breve y [Migración desde BlueBubbles](/es/channels/imessage-from-bluebubbles) para consultar la tabla de traducción completa.

Si el Gateway no se ejecuta en el Mac con la sesión de Mensajes iniciada, mantenga `channels.imessage.enabled=true` y establezca `channels.imessage.cliPath` en un contenedor SSH que ejecute `imsg "$@"` en ese Mac. La ruta local predeterminada `imsg` solo es compatible con macOS.

Antes de depender de un contenedor SSH para los envíos de producción, verifique un `imsg send` saliente a través de ese contenedor exacto. Algunos estados de TCC de macOS asignan la automatización de Mensajes a `/usr/libexec/sshd-keygen-wrapper`, lo que puede permitir que las lecturas y las comprobaciones funcionen mientras los envíos fallan con AppleEvents `-1743`; consulte la sección de solución de problemas del contenedor SSH en [iMessage](/es/channels/imessage).

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
      remoteHost: "user@gateway-host",
      dmPolicy: "pairing",
      allowFrom: ["+15555550123", "user@example.com", "chat_id:123"],
      historyLimit: 50,
      includeAttachments: false,
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      mediaMaxMb: 16,
      service: "auto",
      sendTransport: "auto",
      region: "US",
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
      },
    },
  },
}
```

- El valor opcional `channels.imessage.defaultAccount` reemplaza la selección de cuenta predeterminada cuando coincide con el id. de una cuenta configurada.
- Requiere acceso total al disco para la base de datos de Mensajes.
- Se prefieren los destinos `chat_id:<id>`. Use `imsg chats --limit 20` para enumerar los chats.
- `cliPath` puede apuntar a un contenedor SSH; configure `remoteHost` (`host` o `user@host`) para obtener archivos adjuntos mediante SCP.
- `attachmentRoots` y `remoteAttachmentRoots` restringen las rutas de los archivos adjuntos entrantes (valor predeterminado: `/Users/*/Library/Messages/Attachments`).
- SCP usa una comprobación estricta de la clave del host, por lo que debe asegurarse de que la clave del host de retransmisión ya exista en `~/.ssh/known_hosts`.
- `channels.imessage.configWrites`: permite o deniega las escrituras de configuración iniciadas desde iMessage.
- `channels.imessage.sendTransport`: transporte de envío RPC de `imsg` preferido para las respuestas salientes normales. `auto` (valor predeterminado) usa el puente IMCore para los chats existentes cuando está en ejecución y después recurre a AppleScript; `bridge` exige la entrega mediante una API privada; `applescript` fuerza la ruta pública de automatización de Mensajes.
- `channels.imessage.actions.*`: habilita acciones de API privada que también están condicionadas por `imsg status` / `openclaw channels status --probe`.
- `channels.imessage.includeAttachments` está desactivado de forma predeterminada; establézcalo en `true` antes de esperar contenido multimedia entrante en los turnos del agente.
- La recuperación de mensajes entrantes tras reiniciar el puente/Gateway es automática (desduplicación de GUID más un límite de antigüedad para el trabajo pendiente obsoleto). Las configuraciones existentes de `channels.imessage.catchup.enabled: true` aún se admiten como perfil de compatibilidad obsoleto; `catchup` está deshabilitado de forma predeterminada.
- `channels.imessage.groups`: registro de grupos y configuración por grupo. Con `groupPolicy: "allowlist"`, configure claves `chat_id` explícitas o una entrada comodín `"*"` para que los mensajes de grupo puedan atravesar el control del registro.
- Las entradas `bindings[]` de nivel superior con `type: "acp"` pueden vincular conversaciones de iMessage a sesiones ACP persistentes. Use un identificador normalizado o un destino de chat explícito (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) en `match.peer.id`. Semántica de los campos compartidos: [Agentes ACP](/es/tools/acp-agents#persistent-channel-bindings).

<Accordion title="Ejemplo de contenedor SSH de iMessage">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix está respaldado por un Plugin y se configura en `channels.matrix`.

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
      encryption: true,
      initialSyncLimit: 20,
      defaultAccount: "ops",
      accounts: {
        ops: {
          name: "Ops",
          userId: "@ops:example.org",
          accessToken: "syt_ops_xxx",
        },
        alerts: {
          userId: "@alerts:example.org",
          password: "secret",
          proxy: "http://127.0.0.1:7891",
        },
      },
    },
  },
}
```

- La autenticación mediante token usa `accessToken`; la autenticación mediante contraseña usa `userId` + `password`.
- `channels.matrix.proxy` dirige el tráfico HTTP de Matrix a través de un proxy HTTP(S) explícito. Las cuentas con nombre pueden reemplazarlo mediante `channels.matrix.accounts.<id>.proxy`.
- `channels.matrix.network.dangerouslyAllowPrivateNetwork` permite servidores domésticos privados/internos. `proxy` y esta habilitación de red son controles independientes.
- `channels.matrix.defaultAccount` selecciona la cuenta preferida en configuraciones con varias cuentas.
- `channels.matrix.autoJoin` tiene como valor predeterminado `"off"`, por lo que las salas a las que se recibe una invitación y las nuevas invitaciones de tipo MD se ignoran hasta que se configura `autoJoin: "allowlist"` con `autoJoinAllowlist` o `autoJoin: "always"`.
- `channels.matrix.execApprovals`: entrega de aprobaciones de ejecución nativa de Matrix y autorización de los aprobadores.
  - `enabled`: `true`, `false` o `"auto"` (valor predeterminado). En el modo automático, las aprobaciones de ejecución se activan cuando los aprobadores pueden resolverse desde `approvers` o `commands.ownerAllowFrom`.
  - `approvers`: id. de usuario de Matrix (p. ej., `@owner:example.org`) con permiso para aprobar solicitudes de ejecución.
  - `agentFilter`: lista opcional de id. de agente permitidos. Omítala para reenviar las aprobaciones de todos los agentes.
  - `sessionFilter`: patrones opcionales de claves de sesión (subcadena o expresión regular).
  - `target`: dónde enviar las solicitudes de aprobación. `"dm"` (valor predeterminado), `"channel"` (sala de origen) o `"both"`.
  - Reemplazos por cuenta: `channels.matrix.accounts.<id>.execApprovals`.
- `channels.matrix.dm.sessionScope` controla cómo se agrupan los MD de Matrix en sesiones: `per-user` (valor predeterminado) los comparte según el interlocutor enrutado, mientras que `per-room` aísla cada sala de MD.
- Las comprobaciones de estado de Matrix y las consultas del directorio en tiempo real usan la misma política de proxy que el tráfico en tiempo de ejecución.
- La configuración completa de Matrix, las reglas de selección de destinos y los ejemplos de configuración se documentan en [Matrix](/es/channels/matrix).

### Microsoft Teams

Microsoft Teams está respaldado por un Plugin y se configura en `channels.msteams`.

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId, appPassword, tenantId, webhook, team/channel policies:
      // consulte /channels/msteams
    },
  },
}
```

- Rutas de claves principales tratadas aquí: `channels.msteams`, `channels.msteams.configWrites`.
- La configuración completa de Teams (credenciales, Webhook, política de MD/grupos y reemplazos por equipo/canal) se documenta en [Microsoft Teams](/es/channels/msteams).

### IRC

IRC está respaldado por un Plugin y se configura en `channels.irc`.

```json5
{
  channels: {
    irc: {
      enabled: true,
      dmPolicy: "pairing",
      configWrites: true,
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "${IRC_NICKSERV_PASSWORD}",
        register: false,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

- Rutas de claves principales tratadas aquí: `channels.irc`, `channels.irc.dmPolicy`, `channels.irc.configWrites`, `channels.irc.nickserv.*`.
- El valor opcional `channels.irc.defaultAccount` reemplaza la selección de cuenta predeterminada cuando coincide con el id. de una cuenta configurada.
- La configuración completa del canal IRC (host/puerto/TLS/canales/listas de permitidos/control de menciones) se documenta en [IRC](/es/channels/irc).

### Varias cuentas (todos los canales)

Ejecute varias cuentas por canal (cada una con su propio `accountId`):

```json5
{
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "123456:ABC...",
        },
        alerts: {
          name: "Alerts bot",
          botToken: "987654:XYZ...",
        },
      },
    },
  },
}
```

- `default` se usa cuando se omite `accountId` (CLI + enrutamiento).
- Los tokens de entorno solo se aplican a la cuenta **predeterminada**.
- La configuración base del canal se aplica a todas las cuentas, salvo que se reemplace por cuenta.
- Use `bindings[].match.accountId` para enrutar cada cuenta a un agente diferente.
- Si se añade una cuenta no predeterminada mediante `openclaw channels add` (o la incorporación del canal) mientras aún se usa una configuración de canal de nivel superior con una sola cuenta, OpenClaw promueve primero los valores de cuenta única del nivel superior y ámbito de cuenta al mapa de cuentas del canal, de modo que la cuenta original siga funcionando. La mayoría de los canales los trasladan a `channels.<channel>.accounts.default`; Matrix puede conservar en su lugar un destino existente con nombre o predeterminado que coincida.
- Las vinculaciones existentes que solo especifican el canal (sin `accountId`) siguen coincidiendo con la cuenta predeterminada; las vinculaciones con ámbito de cuenta siguen siendo opcionales.
- `openclaw doctor --fix` también repara formas mixtas trasladando los valores de cuenta única del nivel superior y ámbito de cuenta a la cuenta promovida elegida para ese canal. La mayoría de los canales usan `accounts.default`; Matrix puede conservar en su lugar un destino existente con nombre o predeterminado que coincida.

### Otros canales de Plugin

Muchos canales de Plugin se configuran como `channels.<id>` y se documentan en sus páginas de canal específicas (por ejemplo, Feishu, LINE, Nextcloud Talk, Nostr, QQ Bot, Synology Chat, Twitch y Zalo).
Consulte el índice completo de canales: [Canales](/es/channels).

### Control de menciones en chats de grupo

De forma predeterminada, los mensajes de grupo **requieren una mención** (mención en los metadatos o patrones de expresión regular seguros). Se aplica a los chats de grupo de WhatsApp, Telegram, Discord, Google Chat e iMessage.

Las respuestas visibles se controlan por separado. De forma predeterminada, las solicitudes directas normales de grupos, canales y WebChat interno se entregan automáticamente al finalizar: el texto final del asistente se publica mediante la ruta heredada de respuesta visible. Habilite `messages.visibleReplies: "message_tool"` o `messages.groupChat.visibleReplies: "message_tool"` cuando las respuestas al origen redactadas por el modelo solo deban publicarse después de que el agente invoque `message(action=send)`. Si el modelo devuelve una respuesta final sustancial sin invocar la herramienta de mensajes en un modo habilitado de solo herramientas, ese texto final permanece privado, el registro detallado del Gateway almacena los metadatos de la carga útil suprimida y OpenClaw pone en cola un único reintento de recuperación que solicita al modelo entregar la misma respuesta mediante `message(action=send)`.

La política de solo herramientas rige las respuestas del asistente al origen y los contenidos multimedia genéricos de herramientas. No suprime la salida del terminal propiedad del entorno de ejecución, como las respuestas a comandos autorizados, los avisos de finalización persistentes o los artefactos nativos del proveedor que el entorno propietario clasifica explícitamente como propiedad del host. Los artefactos propiedad del host se entregan mediante la ruta normal de distribución del canal y siguen respetando la denegación de `sendPolicy` saliente. Los turnos ambientales de `room_event` permanecen silenciosos salvo que sean comandos explícitos, incluso cuando la salida del entorno de ejecución esté marcada como propiedad del host.

Las respuestas visibles de solo herramientas requieren un modelo/entorno de ejecución que invoque herramientas de forma fiable y se recomiendan para salas ambientales compartidas en modelos de última generación, como GPT-5.6 Sol. Algunos modelos menos capaces pueden responder con texto final, pero no comprenden que la salida visible en el origen debe enviarse mediante `message(action=send)`. OpenClaw recupera de forma predeterminada el caso habitual de una respuesta final bloqueada únicamente cuando la respuesta final es sustancial, el turno de origen no fue un evento de sala, la política de envío no denegó la entrega y aún no se envió ninguna respuesta al origen. La recuperación está limitada a un único reintento; suprime la persistencia de la solicitud sintética de reintento y mantiene ese reintento fuera de la agrupación de recopilación para impedir que se combine con solicitudes en cola no relacionadas. Si el reintento también queda bloqueado o no puede ponerse en cola, OpenClaw solo entrega un diagnóstico depurado, como "He generado una respuesta, pero no he podido entregarla a este chat. Inténtelo de nuevo." El texto final privado original nunca se marca para su entrega automática al origen. Para los modelos que bloqueen respuestas repetidamente, use `"automatic"` para que el turno final del asistente sea la ruta de respuesta visible, cambie a un modelo más competente en la invocación de herramientas, consulte el resumen de la carga útil suprimida en el registro detallado del Gateway o configure `messages.groupChat.visibleReplies: "automatic"` para usar respuestas finales visibles en todas las solicitudes de grupos/canales.

Si la herramienta de mensajes no está disponible según la política de herramientas activa, OpenClaw recurre a respuestas visibles automáticas en lugar de suprimir silenciosamente la respuesta. `openclaw doctor` advierte sobre esta incompatibilidad.

Esta regla se aplica al texto final normal del agente. Los enlaces de conversación propiedad de un plugin usan la respuesta devuelta por el plugin propietario como respuesta visible para los turnos reclamados del hilo enlazado; el plugin no necesita llamar a `message(action=send)` para esas respuestas de enlace.

**Solución de problemas: una @mención en un grupo activa el indicador de escritura y luego no ocurre nada (sin error)**

Síntoma: una @mención en un grupo/canal muestra el indicador de escritura y el registro del Gateway informa de `dispatch complete (queuedFinal=false, replies=0)`, pero no llega ningún mensaje a la sala. Los MD al mismo agente reciben respuesta con normalidad.

Causa: el modo de respuesta visible del grupo/canal se resuelve como `"message_tool"`, por lo que OpenClaw ejecuta el turno, pero suprime el texto final del asistente a menos que el agente llame a `message(action=send)`. No existe ningún contrato `NO_REPLY` en este modo; si no se llama a la herramienta de mensajes, el texto final original es privado. Para los turnos de origen sustanciales, OpenClaw ahora intenta un reintento de recuperación con protección; las notas breves, el silencio explícito, los eventos de sala, los turnos denegados por la política de envío y los turnos ya entregados no se reintentan. Los turnos normales de grupos y canales usan de forma predeterminada `"automatic"`, por lo que este síntoma solo aparece cuando `messages.groupChat.visibleReplies` (o el valor global `messages.visibleReplies`) se establece explícitamente en `"message_tool"`. El valor `defaultVisibleReplies` del arnés no se aplica aquí: el resolutor de grupos/canales lo ignora; solo afecta a los chats directos/de origen (el arnés de Codex suprime de ese modo los textos finales de los chats directos).

Solución: se puede elegir un modelo con mayor capacidad para llamar a herramientas, eliminar la sustitución explícita `"message_tool"` para volver al valor predeterminado `"automatic"`, o establecer `messages.groupChat.visibleReplies: "automatic"` para forzar respuestas visibles en todas las solicitudes de grupos/canales. Un texto final sustancial que quede bloqueado ya no debería terminar como un éxito silencioso; debería recuperarse mediante un reintento de `message(action=send)` o mostrar el diagnóstico saneado del fallo de entrega. El Gateway recarga en caliente la configuración `messages` después de guardar el archivo; solo es necesario reiniciar el Gateway cuando la supervisión de archivos o la recarga de configuración estén deshabilitadas en el despliegue.

**Tipos de menciones:**

- **Menciones de metadatos**: @menciones nativas de la plataforma. Se ignoran en el modo de chat con uno mismo de WhatsApp.
- **Patrones de texto**: patrones de expresiones regulares seguros en `agents.entries.*.groupChat.mentionPatterns`. Los patrones no válidos y las repeticiones anidadas inseguras se ignoran.
- El control por menciones solo se aplica cuando es posible detectarlas (menciones nativas o al menos un patrón).

```json5
{
  messages: {
    visibleReplies: "automatic", // forzar las respuestas finales automáticas anteriores para chats directos/de origen
    groupChat: {
      historyLimit: 50,
      unmentionedInbound: "room_event", // la conversación continua de la sala sin menciones se convierte en contexto silencioso
      visibleReplies: "message_tool", // opcional; exigir message(action=send) para respuestas visibles en la sala
    },
  },
  agents: {
    list: [{ id: "main", groupChat: { mentionPatterns: ["@openclaw", "openclaw"] } }],
  },
}
```

`messages.groupChat.historyLimit` establece el valor predeterminado global. Los canales pueden sustituirlo con `channels.<channel>.historyLimit` (o por cuenta). Establezca `0` para deshabilitarlo.

`messages.groupChat.unmentionedInbound: "room_event"` envía los mensajes continuos de grupos/canales sin menciones como contexto silencioso de la sala en los canales compatibles. Los mensajes con menciones, los comandos y los mensajes directos siguen siendo solicitudes del usuario. Consulte [Eventos ambientales de sala](/es/channels/ambient-room-events) para ver ejemplos completos de Discord, Slack y Telegram.

`messages.visibleReplies` es el valor predeterminado global para eventos de origen; `messages.groupChat.visibleReplies` lo sustituye para los eventos de origen de grupos/canales. Cuando `messages.visibleReplies` no está establecido, los chats directos/de origen usan el valor predeterminado del entorno de ejecución o del arnés seleccionado, pero los turnos directos internos de WebChat usan la entrega final automática para mantener la paridad de solicitudes de Pi/Codex. Establezca `messages.visibleReplies: "message_tool"` para exigir intencionadamente `message(action=send)` a fin de producir una salida visible. Las listas de permitidos de los canales y el control por menciones siguen determinando si se procesa un evento.

#### Límites del historial de MD

```json5
{
  channels: {
    telegram: {
      dmHistoryLimit: 30,
      dms: {
        "123456789": { historyLimit: 50 },
      },
    },
  },
}
```

Resolución: sustitución por MD → valor predeterminado del proveedor → sin límite (se conserva todo).

Este resolutor lee `channels.<provider>.dmHistoryLimit` y `channels.<provider>.dms.<id>.historyLimit` para cualquier canal cuya clave de sesión siga la forma estándar `provider:direct:<id>` (o la forma heredada `provider:dm:<id>`), por lo que funciona tanto en canales incluidos como en canales de plugins, no solo en una lista fija.

#### Modo de chat con uno mismo

Incluya su propio número en `allowFrom` para habilitar el modo de chat con uno mismo (ignora las @menciones nativas y solo responde a patrones de texto):

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: { mentionPatterns: ["reisponde", "@openclaw"] },
      },
    ],
  },
}
```

### Comandos (gestión de comandos de chat)

```json5
{
  commands: {
    native: "auto", // registrar comandos nativos cuando sean compatibles
    nativeSkills: "auto", // registrar comandos nativos de Skills cuando sean compatibles
    text: true, // analizar /commands en los mensajes de chat
    bash: false, // permitir ! (alias: /bash)
    bashForegroundMs: 2000,
    config: false, // permitir /config
    mcp: false, // permitir /mcp
    plugins: false, // permitir /plugins
    debug: false, // permitir /debug
    restart: true, // permitir /restart y solicitudes externas de reinicio mediante SIGUSR1
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw", // raw | hash
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<Accordion title="Detalles de los comandos">

- Este bloque configura las superficies de comandos. Para consultar el catálogo actual de comandos integrados e incluidos, consulte [Comandos de barra diagonal](/es/tools/slash-commands).
- Esta página es una **referencia de claves de configuración**, no el catálogo completo de comandos. Los comandos propiedad de canales/plugins, como los de QQ Bot `/bot-ping` `/bot-help` `/bot-logs`, LINE `/card`, emparejamiento de dispositivos `/pair`, memoria `/dreaming`, control telefónico `/phone` y Talk `/voice`, se documentan en las páginas de sus respectivos canales/plugins y en [Comandos de barra diagonal](/es/tools/slash-commands).
- Los comandos de texto deben ser mensajes **independientes** que comiencen por `/`.
- `native: "auto"` activa los comandos nativos para Discord/Telegram y los deja desactivados para Slack.
- `nativeSkills: "auto"` activa los comandos nativos de Skills para Discord/Telegram y los deja desactivados para Slack.
- Sustitución por canal: `channels.discord.commands.native` (booleano o `"auto"`). Para Discord, `false` omite el registro y la limpieza de comandos nativos durante el inicio.
- Sustituya el registro de Skills nativas por canal con `channels.<provider>.commands.nativeSkills`.
- `channels.telegram.customCommands` añade entradas adicionales al menú del bot de Telegram.
- `bash: true` habilita `! <cmd>` para el shell del host. Requiere `tools.elevated.enabled` y que el remitente esté en `tools.elevated.allowFrom.<channel>`.
- `config: true` habilita `/config` (lee/escribe `openclaw.json`). Para los clientes `chat.send` del Gateway, las escrituras persistentes de `/config set|unset` también requieren `operator.admin`; el valor de solo lectura `/config show` continúa disponible para los clientes operadores normales con ámbito de escritura.
- `mcp: true` habilita `/mcp` para la configuración de servidores MCP administrada por OpenClaw en `mcp.servers`.
- `plugins: true` habilita `/plugins` para el descubrimiento, la instalación y los controles de habilitación/deshabilitación de plugins.
- `channels.<provider>.configWrites` controla las modificaciones de configuración por canal (valor predeterminado: true).
- En los canales con varias cuentas, `channels.<provider>.accounts.<id>.configWrites` también controla las escrituras dirigidas a esa cuenta (por ejemplo, `/allowlist --config --account <id>` o `/config set channels.<provider>.accounts.<id>...`).
- `restart: false` deshabilita `/restart` y las solicitudes externas de reinicio `SIGUSR1`. Valor predeterminado: `true`.
- `ownerAllowFrom` es la lista de propietarios permitidos explícita para los comandos exclusivos del propietario y las acciones de canal restringidas al propietario. Es independiente de `allowFrom`.
- `ownerDisplay: "hash"` genera hashes de los identificadores de propietarios en la solicitud del sistema. Establezca `ownerDisplaySecret` para controlar la generación de hashes.
- `allowFrom` se configura por proveedor. Cuando está establecido, es la **única** fuente de autorización (se ignoran las listas de permitidos y el emparejamiento de los canales, así como `useAccessGroups`).
- `useAccessGroups: false` permite que los comandos omitan las políticas de grupos de acceso cuando `allowFrom` no está establecido.
- Mapa de la documentación de comandos:
  - catálogo integrado e incluido: [Comandos de barra diagonal](/es/tools/slash-commands)
  - superficies de comandos específicas de cada canal: [Canales](/es/channels)
  - comandos de QQ Bot: [QQ Bot](/es/channels/qqbot)
  - comandos de emparejamiento: [Emparejamiento](/es/channels/pairing)
  - comando de tarjeta de LINE: [LINE](/es/channels/line)
  - Dreaming de memoria: [Dreaming](/es/concepts/dreaming)

</Accordion>

---

## Relacionado

- [Referencia de configuración](/es/gateway/configuration-reference) — claves de nivel superior
- [Configuración — agentes](/es/gateway/config-agents)
- [Descripción general de los canales](/es/channels)
