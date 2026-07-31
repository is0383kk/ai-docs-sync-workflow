---
read_when:
    - Trabajo en las funciones del canal de Discord
summary: Configuración del bot de Discord, claves de configuración, componentes, voz y solución de problemas
title: Discord
x-i18n:
    generated_at: "2026-07-26T05:01:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52a2926217f3a8dfb9398551ddacb0bc6aae6de0a164b215c55256eda9b6245e
    source_path: channels/discord.md
    workflow: 16
---

OpenClaw se conecta a Discord como bot mediante el gateway oficial de Discord. Se admiten los mensajes directos y los canales de servidor.

<CardGroup cols={3}>
  <Card title="Emparejamiento" icon="link" href="/es/channels/pairing">
    Los mensajes directos de Discord usan de forma predeterminada el modo de emparejamiento.
  </Card>
  <Card title="Comandos de barra" icon="terminal" href="/es/tools/slash-commands">
    Comportamiento de los comandos nativos y catálogo de comandos.
  </Card>
  <Card title="Solución de problemas de canales" icon="wrench" href="/es/channels/troubleshooting">
    Flujo de diagnóstico y reparación entre canales.
  </Card>
</CardGroup>

## Configuración rápida

Cree una aplicación de Discord con un bot, añada el bot a su servidor y emparéjelo con OpenClaw. Si es posible, use un servidor privado; si es necesario, [cree uno primero](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server) (**Create My Own > For me and my friends**).

<Steps>
  <Step title="Crear una aplicación y un bot de Discord">
    En el [Portal para desarrolladores de Discord](https://discord.com/developers/applications), haga clic en **New Application** y asígnele un nombre (por ejemplo, "OpenClaw").

    Abra **Bot** en la barra lateral y establezca **Username** en el nombre del agente.

  </Step>

  <Step title="Habilitar intents privilegiados">
    En la página **Bot**, en **Privileged Gateway Intents**, habilite:

    - **Message Content Intent** (obligatorio)
    - **Server Members Intent** (recomendado; obligatorio para las listas de permitidos por rol, la correspondencia entre nombres e ID y los grupos de acceso según la audiencia del canal)
    - **Presence Intent** (opcional; solo para actualizaciones de presencia)

  </Step>

  <Step title="Copiar el token del bot">
    En la página **Bot**, haga clic en **Reset Token** y copie el token.

    <Note>
    A pesar del nombre, esto genera el primer token; no se está «restableciendo» nada.
    </Note>

  </Step>

  <Step title="Generar una URL de invitación y añadir el bot al servidor">
    Abra **OAuth2** en la barra lateral. En **OAuth2 URL Generator**, habilite los ámbitos:

    - `bot`
    - `applications.commands`

    En la sección **Bot Permissions** que aparece, habilite como mínimo:

    **General Permissions**
      - View Channels

    **Text Permissions**
      - Send Messages
      - Read Message History
      - Embed Links
      - Attach Files
      - Add Reactions (opcional)

    Esta es la configuración básica para los canales de texto normales. Si el bot va a publicar en hilos, incluidos los flujos de trabajo de foros o canales multimedia que crean o continúan un hilo, habilite también **Send Messages in Threads**.

    Copie la URL generada, ábrala en un navegador, seleccione el servidor y haga clic en **Continue**. El bot debería aparecer ahora en el servidor.

  </Step>

  <Step title="Habilitar el modo de desarrollador y obtener los ID">
    En la aplicación Discord, habilite el modo de desarrollador para poder copiar los ID:

    1. **User Settings** (icono de engranaje) → **Developer** → active **Developer Mode**
       *(en dispositivos móviles: **App Settings** → **Advanced**)*
    2. Haga clic con el botón derecho en el **icono del servidor** → **Copy Server ID**
    3. Haga clic con el botón derecho en su **propio avatar** → **Copy User ID**

    Guarde el ID del servidor y el ID de usuario junto con el token del bot; necesitará los tres en el siguiente paso.

  </Step>

  <Step title="Permitir mensajes directos de miembros del servidor">
    Para que el emparejamiento funcione, Discord debe permitir que el bot le envíe mensajes directos. Haga clic con el botón derecho en el **icono del servidor** → **Privacy Settings** → active **Direct Messages**.

    Mantenga esta opción activada si utiliza mensajes directos de Discord con OpenClaw. Si solo utiliza canales del servidor, puede desactivarla después del emparejamiento.

  </Step>

  <Step title="Establecer el token del bot de forma segura (no enviarlo por chat)">
    El token del bot es un secreto. Establézcalo en el equipo que ejecuta OpenClaw antes de enviar mensajes al agente:

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
cat > discord.patch.json5 <<'JSON5'
{
  channels: {
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./discord.patch.json5 --dry-run
openclaw config patch --file ./discord.patch.json5
openclaw gateway
```

    Si OpenClaw ya se ejecuta como servicio en segundo plano, reinícielo mediante la aplicación de OpenClaw para Mac o deteniendo y reiniciando el proceso `openclaw gateway run`.
    Para instalaciones como servicio gestionado, ejecute `openclaw gateway install` desde un shell donde esté establecida `DISCORD_BOT_TOKEN`, o almacene la variable en `~/.openclaw/.env` para que el servicio pueda resolver la referencia secreta de entorno después del reinicio.
    Si Discord bloquea o limita por frecuencia la consulta inicial de la aplicación desde el host, establezca el ID de aplicación/cliente del Portal para desarrolladores para que el inicio pueda omitir esa llamada REST: `channels.discord.applicationId` para la cuenta predeterminada o `channels.discord.accounts.<accountId>.applicationId` para cada bot.

  </Step>

  <Step title="Configurar OpenClaw y realizar el emparejamiento">

    <Tabs>
      <Tab title="Pedirlo al agente">
        Chatee con el agente de OpenClaw en un canal existente (por ejemplo, Telegram) e indíqueselo. Si Discord es el primer canal, utilice en su lugar la pestaña CLI/configuración.

        > "Ya he establecido el token de mi bot de Discord en la configuración. Finaliza la configuración de Discord con el ID de usuario `<user_id>` y el ID de servidor `<server_id>`."
      </Tab>
      <Tab title="CLI / configuración">
        Configuración basada en archivos:

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: {
        source: "env",
        provider: "default",
        id: "DISCORD_BOT_TOKEN",
      },
    },
  },
}
```

        Alternativa mediante variable de entorno para la cuenta predeterminada:

```bash
DISCORD_BOT_TOKEN=...
```

        Para una configuración mediante scripts o de forma remota, escriba el mismo bloque JSON5 con `openclaw config patch --file ./discord.patch.json5 --dry-run` y vuelva a ejecutarlo sin `--dry-run`. También funcionan las cadenas de texto sin formato `token`, y se admiten valores SecretRef para `channels.discord.token` mediante proveedores de entorno, archivo y ejecución. Consulte [Gestión de secretos](/es/gateway/secrets).

        Para varios bots de Discord, mantenga el token y el ID de aplicación de cada bot en su cuenta. Las cuentas heredan un `channels.discord.applicationId` de nivel superior, así que establézcalo allí únicamente cuando todas las cuentas utilicen el mismo ID de aplicación.

```json5
{
  channels: {
    discord: {
      enabled: true,
      accounts: {
        personal: {
          token: { source: "env", provider: "default", id: "DISCORD_PERSONAL_TOKEN" },
          applicationId: "111111111111111111",
        },
        work: {
          token: { source: "env", provider: "default", id: "DISCORD_WORK_TOKEN" },
          applicationId: "222222222222222222",
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="Aprobar el primer emparejamiento por mensaje directo">
    Cuando el gateway esté en ejecución, envíe un mensaje directo al bot en Discord. Este responderá con un código de emparejamiento.

    <Tabs>
      <Tab title="Pedirlo al agente">
        Envíe el código de emparejamiento al agente en el canal existente:

        > "Aprueba este código de emparejamiento de Discord: `<CODE>`"
      </Tab>
      <Tab title="CLI">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    Los códigos de emparejamiento caducan después de 1 hora. Tras la aprobación, chatee con el agente mediante un mensaje directo de Discord.

  </Step>
</Steps>

<Note>
La resolución de tokens tiene en cuenta la cuenta. Los valores de token de la configuración tienen prioridad sobre la alternativa de entorno, y `DISCORD_BOT_TOKEN` solo se utiliza para la cuenta predeterminada.
Si dos cuentas de Discord habilitadas se resuelven con el mismo token de bot, OpenClaw inicia un solo monitor de gateway para ese token: un token procedente de la configuración tiene prioridad sobre la alternativa de entorno; de lo contrario, prevalece la primera cuenta habilitada y la cuenta duplicada se marca como deshabilitada con el motivo `duplicate bot token`.
Para llamadas salientes avanzadas (herramienta de mensajes/acciones de canal), se utiliza un `token` explícito por llamada. Esto se aplica a las acciones de envío y de tipo lectura/sondeo (leer/buscar/obtener/hilo/mensajes fijados/permisos). La configuración de políticas y reintentos de la cuenta sigue procediendo de la cuenta seleccionada en la instantánea activa del entorno de ejecución.
</Note>

## Recomendación: configurar un espacio de trabajo de servidor

Cuando los mensajes directos funcionen, puede convertir el servidor en un espacio de trabajo completo donde cada canal tenga su propia sesión de agente con su propio contexto. Se recomienda para servidores privados donde solo estén el usuario y el bot.

<Steps>
  <Step title="Añadir el servidor a la lista de permitidos de servidores">
    Esto permite que el agente responda en cualquier canal del servidor, no solo en mensajes directos.

    <Tabs>
      <Tab title="Pedirlo al agente">
        > "Añade mi ID de servidor de Discord `<server_id>` a la lista de permitidos de servidores"
      </Tab>
      <Tab title="Configuración">

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="Permitir respuestas sin @mención">
    De forma predeterminada, el agente solo responde en los canales del servidor cuando se lo @menciona. En un servidor privado, probablemente convenga que responda a todos los mensajes.

    En los canales del servidor, las respuestas normales se publican automáticamente de forma predeterminada. Para salas compartidas siempre activas, habilite `messages.groupChat.visibleReplies: "message_tool"` para que el agente pueda observar en silencio y publicar solo cuando determine que una respuesta en el canal resulta útil. Esto funciona mejor con modelos de última generación fiables en el uso de herramientas, como GPT-5.6 Sol. Los eventos ambientales de la sala permanecen en silencio salvo que la herramienta realice un envío. Consulte [Eventos ambientales de sala](/es/channels/ambient-room-events) para ver la configuración completa del modo de observación silenciosa.

    Si Discord muestra que se está escribiendo y los registros indican uso de tokens, pero no se publica ningún mensaje, compruebe si el turno se configuró como evento ambiental de sala o si se habilitaron respuestas visibles mediante la herramienta de mensajes.

    <Tabs>
      <Tab title="Pedirlo al agente">
        > "Permite que mi agente responda en este servidor sin que tengan que @mencionarlo"
      </Tab>
      <Tab title="Configuración">
        Establezca `requireMention: false` en la configuración del servidor:

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

        Para exigir envíos mediante la herramienta de mensajes en las respuestas visibles de grupos/canales, establezca `messages.groupChat.visibleReplies: "message_tool"`.

      </Tab>
    </Tabs>

  </Step>

  <Step title="Planificar la memoria en los canales del servidor">
    La memoria a largo plazo (MEMORY.md) solo se carga automáticamente en las sesiones de mensajes directos; los canales del servidor no la cargan.

    <Tabs>
      <Tab title="Pedirlo al agente">
        > "Cuando haga preguntas en canales de Discord, usa memory_search o memory_get si necesitas contexto a largo plazo de MEMORY.md."
      </Tab>
      <Tab title="Manual">
        Para disponer de contexto compartido en todos los canales, incluya instrucciones estables en `AGENTS.md` o `USER.md` (se insertan en cada sesión). Guarde las notas a largo plazo en `MEMORY.md` y acceda a ellas cuando sea necesario mediante las herramientas de memoria.
      </Tab>
    </Tabs>

  </Step>
</Steps>

Ahora cree canales y empiece a chatear. El agente ve el nombre del canal y cada canal es una sesión aislada: configure `#coding`, `#home`, `#research` o lo que mejor se adapte al flujo de trabajo.

## Modelo de ejecución

- El Gateway gestiona la conexión con Discord.
- El enrutamiento de respuestas es determinista: las respuestas a entradas de Discord vuelven a Discord.
- Los metadatos de servidores/canales de Discord se añaden al prompt del modelo como contexto no fiable, no como prefijo visible para el usuario en la respuesta. Si un modelo vuelve a copiar ese envoltorio, OpenClaw elimina los metadatos copiados de las respuestas salientes y del contexto de reproducción futuro.
- De forma predeterminada (`session.dmScope=main`), los chats directos comparten la sesión principal del agente (`agent:main:main`).
- Los canales del servidor tienen claves de sesión aisladas (`agent:<agentId>:discord:channel:<channelId>`).
- Los mensajes directos de grupo se ignoran de forma predeterminada (`channels.discord.dm.groupEnabled=false`).
- Los comandos de barra nativos se ejecutan en sesiones de comandos aisladas (`agent:<agentId>:discord:slash:<userId>`), pero siguen llevando `CommandTargetSessionKey` a la sesión de conversación enrutada.
- La entrega a Discord de anuncios de cron/heartbeat solo de texto se reduce a la respuesta final visible del asistente, que se envía una sola vez. Los elementos multimedia y las cargas útiles de componentes estructurados siguen usando varios mensajes cuando el agente emite varias cargas útiles entregables.

## Canales de foro

Los canales de foro y multimedia de Discord solo aceptan publicaciones en hilos. OpenClaw permite crearlas de dos maneras:

- Envía un mensaje al foro principal (`channel:<forumId>`) para crear automáticamente un hilo. El título del hilo es la primera línea no vacía del mensaje (truncada al límite de 100 caracteres de Discord para nombres de hilos).
- Usa `openclaw message thread create` para crear un hilo directamente. No pases `--message-id` para canales de foro.

Envía al foro principal para crear un hilo:

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "Título del tema\nCuerpo de la publicación"
```

Crea explícitamente un hilo de foro:

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "Título del tema" --message "Cuerpo de la publicación"
```

Los foros principales no aceptan componentes de Discord. Si necesitas componentes, envía el mensaje al propio hilo (`channel:<threadId>`).

## Componentes interactivos

OpenClaw admite contenedores de componentes v2 de Discord para los mensajes del agente. Usa la herramienta de mensajes con una carga `components`. Los resultados de las interacciones se enrutan de vuelta al agente como mensajes entrantes normales y siguen la configuración existente de `replyToMode` de Discord.

Bloques admitidos:

- `text`, `section`, `separator`, `actions`, `media-gallery`, `file`
- Las filas de acciones permiten hasta 5 botones o un único menú de selección
- Tipos de selección: `string`, `user`, `role`, `mentionable`, `channel`

De forma predeterminada, los componentes son de un solo uso. Establece `components.reusable=true` para permitir que los botones, las selecciones y los formularios se usen varias veces hasta que caduquen.

Para restringir quién puede hacer clic en un botón, establece `allowedUsers` en ese botón (ID de usuario de Discord, etiquetas o `*`). Los usuarios que no coincidan reciben una denegación efímera.

De forma predeterminada, las devoluciones de llamada de los componentes caducan después de 30 minutos. Establece `channels.discord.agentComponents.ttlMs` para cambiar la duración del registro de devoluciones de llamada de la cuenta predeterminada, o `channels.discord.accounts.<accountId>.agentComponents.ttlMs` para cada cuenta. El valor se expresa en milisegundos, debe ser un entero positivo y tiene un límite de `86400000` (24 horas). Los TTL más largos son adecuados para flujos de revisión o aprobación que necesitan mantener utilizables los botones, pero amplían el intervalo durante el cual un mensaje antiguo de Discord aún puede activar una acción. Usa el TTL más corto que resulte adecuado y conserva el valor predeterminado cuando las devoluciones de llamada obsoletas pudieran resultar inesperadas.

Los comandos de barra `/model` y `/models` abren un selector de modelos interactivo con listas desplegables de proveedor, modelo y entorno de ejecución compatible, además de un paso de envío. `/models add` está obsoleto y devuelve un mensaje de obsolescencia en lugar de registrar modelos desde el chat. La respuesta del selector es efímera y solo puede usarla el usuario que lo invocó. Los menús de selección de Discord tienen un límite de 25 opciones, por lo que debes añadir entradas `provider/*` a `agents.defaults.modelPolicy.allow` cuando quieras que el selector muestre modelos detectados dinámicamente solo para determinados proveedores, como `openai` o `vllm`.

Archivos adjuntos:

- Los bloques `file` deben apuntar a una referencia de archivo adjunto (`attachment://<filename>`)
- Proporciona el archivo adjunto mediante `media`/`path`/`filePath` (un solo archivo); usa `media-gallery` para varios archivos
- Usa `filename` para sustituir el nombre de carga cuando deba coincidir con la referencia del archivo adjunto

Formularios modales:

- Añade `components.modal` con hasta 5 campos
- Tipos de campo: `text`, `checkbox`, `radio`, `select`, `role-select`, `user-select`
- OpenClaw añade automáticamente un botón de activación

Ejemplo:

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "Texto alternativo opcional",
  components: {
    reusable: true,
    text: "Elige una ruta",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "Aprobar",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "Rechazar", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "Elige una opción",
          options: [
            { label: "Opción A", value: "a" },
            { label: "Opción B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "Detalles",
      triggerLabel: "Abrir formulario",
      fields: [
        { type: "text", label: "Solicitante" },
        {
          type: "select",
          label: "Prioridad",
          options: [
            { label: "Baja", value: "low" },
            { label: "Alta", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## Control de acceso y enrutamiento

<Tabs>
  <Tab title="Política de mensajes directos">
    `channels.discord.dmPolicy` controla el acceso a los mensajes directos. `channels.discord.allowFrom` es la lista de permitidos canónica para mensajes directos.

    - `pairing` (valor predeterminado)
    - `allowlist` (requiere al menos un remitente `allowFrom`)
    - `open` (requiere que `channels.discord.allowFrom` incluya `"*"`)
    - `disabled`

    Si la política de mensajes directos no está abierta, los usuarios desconocidos se bloquean (o se les solicita emparejarse en el modo `pairing`).

    Precedencia entre varias cuentas:

    - `channels.discord.accounts.default.allowFrom` solo se aplica a la cuenta `default`.
    - Para una cuenta, `allowFrom` tiene precedencia sobre el valor heredado `dm.allowFrom`.
    - Las cuentas con nombre heredan `channels.discord.allowFrom` cuando no se han establecido sus propios valores `allowFrom` ni el valor heredado `dm.allowFrom`.
    - Las cuentas con nombre no heredan `channels.discord.accounts.default.allowFrom`.

    Los valores heredados `channels.discord.dm.policy` y `channels.discord.dm.allowFrom` todavía se leen por compatibilidad. `openclaw doctor --fix` los migra a `dmPolicy` y `allowFrom` cuando puede hacerlo sin modificar el acceso.

    Formato del destino de mensajes directos para la entrega:

    - `user:<id>`
    - mención `<@id>`

    Los ID numéricos sin prefijo normalmente se resuelven como ID de canal cuando hay un canal predeterminado activo, pero los ID incluidos en la lista efectiva `allowFrom` de mensajes directos de la cuenta se tratan como destinos de mensajes directos de usuario por compatibilidad.

  </Tab>

  <Tab title="Grupos de acceso">
    Los mensajes directos de Discord y la autorización de comandos de texto pueden usar entradas dinámicas `accessGroup:<name>` en `channels.discord.allowFrom`.

    Los nombres de los grupos de acceso se comparten entre los canales de mensajes. Usa `type: "message.senders"` para un grupo estático cuyos miembros se expresen mediante la sintaxis `allowFrom` normal de cada canal, o `type: "discord.channelAudience"` cuando la audiencia `ViewChannel` actual de un canal de Discord deba definir dinámicamente la pertenencia. Comportamiento compartido de los grupos de acceso: [Grupos de acceso](/es/channels/access-groups).

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        "*": ["global-owner-id"],
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
      },
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
  },
}
```

    Un canal de texto de Discord no tiene una lista de miembros independiente. `type: "discord.channelAudience"` modela la pertenencia de la siguiente manera: el remitente del mensaje directo es miembro del servidor configurado y actualmente tiene el permiso efectivo `ViewChannel` en el canal configurado después de aplicar las sobrescrituras de roles y del canal.

    Ejemplo: permitir que cualquier persona que pueda ver `#maintainers` envíe mensajes directos al bot, mientras se mantienen cerrados para todos los demás.

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
      membership: "canViewChannel",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers"],
    },
  },
}
```

    Se pueden combinar entradas dinámicas y estáticas:

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers", "discord:123456789012345678"],
    },
  },
}
```

    Las búsquedas se cierran en caso de error. Si Discord devuelve `Missing Access`, falla la búsqueda del miembro o el canal pertenece a otro servidor, el remitente del mensaje directo se considera no autorizado.

    Activa **Server Members Intent** en el Discord Developer Portal cuando uses grupos de acceso basados en la audiencia del canal. Los mensajes directos no incluyen el estado de pertenencia al servidor, por lo que OpenClaw resuelve el miembro mediante la API REST de Discord en el momento de la autorización.

  </Tab>

  <Tab title="Política de servidores">
    La gestión de servidores se controla mediante `channels.discord.groupPolicy`:

    - `open`
    - `allowlist`
    - `disabled`

    La configuración segura de referencia cuando existe `channels.discord` es `allowlist`.

    Comportamiento de `allowlist`:

    - el servidor debe coincidir con `channels.discord.guilds` (se prefiere `id`, se acepta el slug)
    - listas de remitentes permitidos opcionales: `users` (se recomiendan ID estables) y `roles` (solo ID de roles); si se configura cualquiera de ellas, los remitentes se permiten cuando coinciden con `users` O `roles`
    - la coincidencia directa por nombre o etiqueta está desactivada de forma predeterminada; activa `channels.discord.dangerouslyAllowNameMatching: true` solo como modo de compatibilidad de emergencia
    - se admiten nombres y etiquetas para `users`, pero los ID son más seguros; `openclaw security audit` advierte cuando se usan entradas de nombre o etiqueta
    - si un servidor tiene configurado `channels`, se deniegan los canales que no figuren en la lista
    - si un servidor no tiene un bloque `channels`, se permiten todos los canales de ese servidor incluido en la lista de permitidos

    Ejemplo:

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { enabled: true },
            help: { enabled: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    `openclaw doctor --fix` migra la clave heredada `allow` de cada canal a `enabled`.

    Si solo estableces `DISCORD_BOT_TOKEN` y no creas un bloque `channels.discord`, el valor alternativo en tiempo de ejecución es `groupPolicy="allowlist"` (con una advertencia en los registros), incluso si `channels.defaults.groupPolicy` es `open`.

  </Tab>

  <Tab title="Menciones y mensajes directos grupales">
    De forma predeterminada, los mensajes de los servidores requieren una mención.

    La detección de menciones incluye:

    - mención explícita del bot
    - patrones de mención configurados (`agents.entries.*.groupChat.mentionPatterns`, con `messages.groupChat.mentionPatterns` como alternativa)
    - comportamiento implícito de respuesta al bot en los casos admitidos

    Al redactar mensajes salientes de Discord, usa la sintaxis de mención canónica: `<@USER_ID>` para usuarios, `<#CHANNEL_ID>` para canales y `<@&ROLE_ID>` para roles. No uses el formato heredado de mención por apodo `<@!USER_ID>`.

    `requireMention` se configura para cada servidor o canal (`channels.discord.guilds...`).
    `ignoreOtherMentions` descarta opcionalmente los mensajes que mencionan a otro usuario o rol, pero no al bot (excepto @everyone/@here).

    Mensajes directos grupales:

    - valor predeterminado: ignorados (`dm.groupEnabled=false`)
    - lista de permitidos opcional mediante `dm.groupChannels` (ID o slugs de canales)

  </Tab>
</Tabs>

### Enrutamiento de agentes basado en roles

Usa `bindings[].match.roles` para enrutar a los miembros de servidores de Discord hacia distintos agentes según el ID del rol. Las vinculaciones basadas en roles solo aceptan ID de roles y se evalúan después de las vinculaciones de pares o pares principales y antes de las vinculaciones exclusivas del servidor. Si una vinculación también establece otros campos de coincidencia (por ejemplo, `peer` + `guildId` + `roles`), todos los campos configurados deben coincidir.

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## Comandos nativos y autorización de comandos

- `commands.native` tiene como valor predeterminado `"auto"` y está habilitado para Discord.
- Anulación por canal: `channels.discord.commands.native`.
- `commands.native=false` omite el registro y la limpieza de comandos de barra de Discord durante el inicio. Los comandos registrados anteriormente pueden seguir visibles en Discord hasta que se eliminen de la aplicación de Discord.
- La autorización de comandos nativos utiliza las mismas listas de permitidos y políticas de Discord que el procesamiento normal de mensajes.
- Los comandos pueden seguir visibles en la interfaz de Discord para usuarios no autorizados; durante su ejecución, se aplica la autorización de OpenClaw y se responde "no autorizado".
- Configuración predeterminada de los comandos de barra: `ephemeral: true` (`channels.discord.slashCommand.ephemeral`).

Consulte [Comandos de barra](/es/tools/slash-commands) para ver el catálogo y el comportamiento de los comandos.

## Detalles de las funciones

<AccordionGroup>
  <Accordion title="Etiquetas de respuesta y respuestas nativas">
    Discord admite etiquetas de respuesta en la salida del agente:

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    Se controla mediante `channels.discord.replyToMode`:

    - `off` (valor predeterminado): no se crean hilos de respuesta implícitos; las etiquetas explícitas `[[reply_to_*]]` siguen respetándose
    - `first`: adjunta la referencia de respuesta nativa implícita al primer mensaje saliente de Discord del turno
    - `all`: la adjunta a todos los mensajes salientes
    - `batched`: la adjunta únicamente cuando el evento entrante era un lote agrupado mediante antirrebote de varios mensajes; resulta útil cuando se desean respuestas nativas principalmente para conversaciones con ráfagas ambiguas, no para cada turno de un solo mensaje

    Los identificadores de los mensajes se incluyen en el contexto y el historial para que los agentes puedan dirigirse a mensajes específicos.

  </Accordion>

  <Accordion title="Vistas previas de enlaces">
    De forma predeterminada, Discord genera elementos insertados enriquecidos para las URL. OpenClaw suprime de forma predeterminada esos elementos generados en los mensajes salientes de Discord, de modo que las URL enviadas por el agente permanecen como enlaces simples a menos que se habilite esta función:

```json5
{
  channels: {
    discord: {
      suppressEmbeds: false,
    },
  },
}
```

    Establezca `channels.discord.accounts.<id>.suppressEmbeds` para anular el valor de una cuenta. Los envíos de la herramienta de mensajes del agente también pueden incluir `suppressEmbeds: false` para un solo mensaje. Las cargas `embeds` explícitas de Discord no se suprimen mediante la configuración predeterminada de vistas previas de enlaces.

  </Accordion>

  <Accordion title="Vista previa de transmisión en directo">
    OpenClaw puede transmitir borradores de respuesta enviando un mensaje temporal y editándolo a medida que llega el texto. `channels.discord.streaming.mode` acepta `off` | `partial` | `block` | `progress` (valor predeterminado cuando no se ha establecido ninguna clave `streaming` ni la clave heredada `streamMode`). `streamMode` es un alias heredado; ejecute `openclaw doctor --fix` para reescribir la configuración persistente con la estructura anidada canónica `streaming`.

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLines: 8,
          maxLineChars: 120,
          toolProgress: false,
          commentary: false,
        },
      },
    },
  },
}
```

    - `off` deshabilita las ediciones de la vista previa de Discord.
    - `partial` edita un único mensaje de vista previa a medida que llegan los tokens.
    - `block` emite fragmentos del tamaño de un borrador; ajuste el tamaño y los puntos de corte con `streaming.preview.chunk` (`minChars`, `maxChars`, `breakPreference`), limitado a `textChunkLimit`. Cuando la transmisión por bloques está habilitada explícitamente, OpenClaw omite la transmisión de la vista previa para evitar una transmisión doble.
    - `progress` mantiene un único borrador de estado editable hasta la entrega final. De forma predeterminada, muestra una línea del preámbulo o la narración más reciente del agente, sin etiquetas generadas, separadores ni filas de herramientas.
    - Los contenidos multimedia, los errores y las respuestas finales explícitas cancelan las ediciones pendientes de la vista previa.
    - `streaming.preview.toolProgress` tiene como valor predeterminado `true` en el modo `partial`/`block`. El modo de progreso de Discord no muestra filas de herramientas de forma predeterminada; establezca `streaming.progress.toolProgress: true` para habilitarlas.
    - Establezca `streaming.progress.toolProgress: true` para añadir filas compactas de herramientas o progreso, como `🛠️ Bash: run tests` o `🔎 Web Search: for "query"`. Por compatibilidad, una configuración existente de `progress.label` o `progress.labels` conserva el valor predeterminado anterior de las filas de herramientas; establezca `toolProgress: false` para usar una etiqueta personalizada sin filas.
    - `streaming.progress.commentary` (valor predeterminado: `false`) permite incluir comentarios sin procesar del asistente en el borrador temporal de progreso. La línea de estado predeterminada del preámbulo o la narración es independiente de esta opción. Los comentarios se depuran antes de mostrarse, permanecen transitorios y no modifican la entrega de la respuesta final.
    - `streaming.progress.maxLineChars` controla el límite de la vista previa de progreso por línea. La prosa se acorta en los límites entre palabras; los detalles de comandos y rutas conservan sufijos útiles.
    - `streaming.preview.commandText` / `streaming.progress.commandText` controla los detalles de comandos y ejecuciones en las líneas compactas de progreso: `raw` (valor predeterminado) o `status` (solo la etiqueta de la herramienta).

    Para ocultar el texto sin procesar de comandos y ejecuciones sin dejar de mostrar las líneas compactas de progreso:

    ```json
    {
      "channels": {
        "discord": {
          "streaming": {
            "mode": "progress",
            "progress": {
              "toolProgress": true,
              "commandText": "status"
            }
          }
        }
      }
    }
    ```

    La transmisión de la vista previa solo admite texto; las respuestas con contenido multimedia utilizan la entrega normal.

  </Accordion>

  <Accordion title="Historial, contexto y comportamiento de los hilos">
    Contexto del historial del servidor:

    - `channels.discord.historyLimit` valor predeterminado: `20`
    - alternativa: `messages.groupChat.historyLimit`
    - `0` lo deshabilita

    Controles del historial de mensajes directos:

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    Comportamiento de los hilos:

    - Los hilos de Discord se enrutan como sesiones de canal y heredan la configuración del canal principal, salvo que se anule.
    - Las sesiones de hilo heredan la selección `/model` de nivel de sesión del canal principal como alternativa exclusiva para el modelo; las selecciones `/model` locales del hilo tienen prioridad, y el historial de transcripciones del canal principal no se copia salvo que esté habilitada su herencia.
    - `channels.discord.thread.inheritParent` (valor predeterminado: `false`) permite que los nuevos hilos automáticos se inicialicen a partir de la transcripción principal. Anulación por cuenta: `channels.discord.accounts.<id>.thread.inheritParent`.
    - Las reacciones de la herramienta de mensajes pueden resolver destinos de mensajes directos `user:<id>`.
    - `guilds.<guild>.channels.<channel>.requireMention: false` se conserva durante el mecanismo alternativo de activación de la fase de respuesta.

    Los temas de los canales se introducen como contexto **no fiable**. Las listas de permitidos limitan quién puede activar el agente, pero no constituyen un límite completo de supresión de información del contexto complementario.

  </Accordion>

  <Accordion title="Sesiones vinculadas a hilos para subagentes">
    Discord puede vincular un hilo a un destino de sesión para que los mensajes de seguimiento de ese hilo sigan enrutándose a la misma sesión, incluidas las sesiones de subagentes.

    Comandos:

    - `/focus <target>` vincula el hilo actual o uno nuevo a un destino de subagente o sesión
    - `/unfocus` elimina la vinculación del hilo actual
    - `/agents` muestra las ejecuciones activas y el estado de vinculación
    - `/session idle <duration|off>` consulta o actualiza la pérdida automática de foco por inactividad para las vinculaciones enfocadas
    - `/session max-age <duration|off>` consulta o actualiza la antigüedad máxima absoluta de las vinculaciones enfocadas

    Configuración:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
      spawnSessions: true,
      defaultSpawnContext: "fork",
    },
  },
}
```

    Notas:

    - `session.threadBindings.*` es la política canónica para Discord y Telegram.
    - `spawnSessions` controla la creación y vinculación automáticas de hilos para `sessions_spawn({ thread: true })` y las creaciones de hilos de ACP. Valor predeterminado: `true`.
    - `defaultSpawnContext` controla el contexto nativo de los subagentes para las creaciones vinculadas a hilos. Valor predeterminado: `"fork"`.
    - Las claves obsoletas `spawnSubagentSessions`/`spawnAcpSessions` se migran mediante `openclaw doctor --fix`.
    - Si las vinculaciones de hilos están deshabilitadas, `/focus` y las operaciones relacionadas no están disponibles.

    Consulte [Subagentes](/es/tools/subagents), [Agentes ACP](/es/tools/acp-agents) y [Referencia de configuración](/es/gateway/configuration-reference).

  </Accordion>

  <Accordion title="Progreso de los subagentes en el mensaje de origen">
    Establezca `channels.discord.subagentProgress: true` para mostrar la actividad secundaria en segundo plano en el mensaje de Discord que inició la ejecución principal.

```json5
{
  channels: {
    discord: {
      subagentProgress: true,
    },
  },
}
```

    Mientras haya ejecuciones secundarias activas, OpenClaw mantiene activo el indicador de escritura de Discord durante un máximo de una hora y sustituye una reacción de recuento (desde `1️⃣` hasta `🔟`) a medida que cambia el número de ejecuciones simultáneas; `🔟` también representa 10 o más. La reacción de recuento se elimina cuando finaliza la última ejecución secundaria. Una ejecución secundaria fallida, agotada por tiempo de espera o terminada deja una reacción `🔴`.

    Esta función debe habilitarse explícitamente y utiliza valores internos fijos para los tiempos y los emojis. El bot necesita el permiso **Add Reactions** para proporcionar información mediante reacciones. El valor de nivel de cuenta `channels.discord.accounts.<id>.subagentProgress` anula el valor de nivel superior.

  </Accordion>

  <Accordion title="Vinculaciones persistentes de canales ACP">
    Para espacios de trabajo ACP estables y siempre activos, configure vinculaciones ACP tipadas de nivel superior dirigidas a conversaciones de Discord.

    Ruta de configuración: `bindings[]` con `type: "acp"` y `match.channel: "discord"`.

```json5
{
  agents: {
    entries: {
      codex: {
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
    },
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
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

    Notas:

    - `/acp spawn codex --bind here` vincula el canal o hilo actual sin reemplazarlo y mantiene los mensajes futuros en la misma sesión ACP. Los mensajes del hilo heredan la vinculación del canal principal.
    - En un canal o hilo vinculado, `/new` y `/reset` restablecen la misma sesión ACP sin reemplazarla. Las vinculaciones temporales de hilos pueden anular la resolución del destino mientras estén activas.
    - `spawnSessions` controla la creación y vinculación de hilos secundarios mediante `--thread auto|here`.

    Consulte [Agentes ACP](/es/tools/acp-agents) para obtener detalles sobre el comportamiento de las vinculaciones.

  </Accordion>

  <Accordion title="Notificaciones de reacciones">
    Modo de notificación de reacciones por servidor (`guilds.<id>.reactionNotifications`):

    - `off`
    - `own` (valor predeterminado)
    - `all`
    - `allowlist` (utiliza `guilds.<id>.users`)

    Los eventos de reacción se convierten en eventos del sistema y se adjuntan a la sesión de Discord enrutada.

  </Accordion>

  <Accordion title="Eventos de presencia en línea">
    Habilite en un servidor las activaciones enrutadas del agente cuando un miembro humano pase de estar desconectado a estar en línea:

    ```json5
    {
      channels: {
        discord: {
          intents: { presence: true },
          guilds: {
            "111111111111111111": {
              presenceEvents: {
                channelId: "222222222222222222",
                users: ["333333333333333333"], // opcional; restringe aún más los usuarios del canal
                reconnectSuppressSeconds: 300, // opcional; periodo de silencio de la nueva sesión (0 lo desactiva)
                burstLimit: 8, // opcional; máximo de eventos por ventana de ráfaga
                burstWindowSeconds: 60, // opcional; ventana deslizante de detección de ráfagas
              },
            },
          },
        },
      },
    }
    ```

    `presenceEvents` requiere que el Heartbeat esté habilitado para el agente al que se dirige y el **Presence Intent** privilegiado en la página Bot de la aplicación en el Discord Developer Portal. OpenClaw obtiene los miembros actualmente conectados de cada instantánea completa de `GUILD_CREATE`, dirige las transiciones observadas de desconectado a conectado y también considera como recién disponible una primera señal posterior de conexión de un miembro no visto. Es posible que ese miembro se haya conectado o unido después de la instantánea, por lo que el evento no afirma cuál era exactamente su estado anterior. Solo son aptas las personas que pueden ver `channelId`: los canales y los hilos públicos requieren **View Channel** en el canal o en el elemento principal, mientras que los hilos privados requieren además ser miembro o tener **Manage Threads**. `users` puede restringir aún más esa audiencia. OpenClaw ignora los bots y los estados de conexión sin cambios, y mantiene un periodo de espera de ocho horas por usuario entre reinicios del Gateway. Cuando Discord establece una nueva sesión del Gateway y envía `READY`, OpenClaw suprime los eventos derivados de la presencia durante `reconnectSuppressSeconds` (valor predeterminado: 300; `0` lo desactiva) mientras se reconstruye el estado de presencia del servidor, para que los miembros detectados de nuevo no activen el agente uno por uno. Además, limita la frecuencia de los eventos puestos correctamente en cola por servidor a `burstLimit` eventos (valor predeterminado: 8) por ventana deslizante de `burstWindowSeconds` (valor predeterminado: 60) y registra una sola vez cada episodio de supresión del servidor. Una sesión reanudada no se considera una sesión nueva. Discord limita las instantáneas de los servidores con más de 75,000 miembros; en esos casos, OpenClaw requiere una actualización explícita del estado desconectado antes de saludar. El evento del sistema contiene identificadores inmutables del usuario, el servidor y el canal sin incluir nombres visibles mutables. El agente decide si saluda y cómo hacerlo.

  </Accordion>

  <Accordion title="Reacciones de confirmación">
    `ackReaction` envía un emoji de confirmación mientras OpenClaw procesa un mensaje entrante.

    Orden de resolución:

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - emoji de respaldo de la identidad del agente (`agents.entries.*.identity.emoji`; de lo contrario, "👀")

    Notas:

    - Discord acepta emojis Unicode o nombres de emojis personalizados.
    - Use `""` para desactivar la reacción en un canal o una cuenta.

    **Ámbito (`messages.ackReactionScope`):**

    Valores: `"all"` (mensajes directos + grupos, incluidos los eventos ambientales de salas), `"direct"` (solo mensajes directos), `"group-all"` (todos los mensajes de grupo excepto los eventos ambientales de salas, sin mensajes directos), `"group-mentions"` (grupos cuando se menciona al bot; **sin mensajes directos**, valor predeterminado), `"off"` / `"none"` (desactivado).

    <Note>
    El ámbito predeterminado (`"group-mentions"`) no activa reacciones de confirmación en mensajes directos ni en eventos ambientales de salas. Para obtener una reacción de confirmación en los mensajes directos entrantes de Discord y en los eventos de salas silenciosas, establezca `messages.ackReactionScope` en `"all"`.
    </Note>

  </Accordion>

  <Accordion title="Escrituras de configuración">
    Las escrituras de configuración iniciadas desde el canal están habilitadas de forma predeterminada. Esto afecta a los flujos de `/config set|unset` (cuando las funciones de comandos están habilitadas).

    Para desactivarlas:

```json5
{
  channels: {
    discord: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="Proxy del Gateway">
    Dirija el tráfico WebSocket del Gateway de Discord y las consultas REST de inicio (identificador de la aplicación + resolución de la lista de permitidos) a través de un proxy HTTP(S) con `channels.discord.proxy`.
    El uso de proxy para el WebSocket del Gateway de Discord es explícito; las conexiones WebSocket no heredan las variables de entorno de proxy del proceso del Gateway. Las consultas REST de inicio usan este proxy cuando se configura `channels.discord.proxy`.

```json5
{
  channels: {
    discord: {
      proxy: "http://proxy.example:8080",
    },
  },
}
```

    Anulación por cuenta:

```json5
{
  channels: {
    discord: {
      accounts: {
        primary: {
          proxy: "http://proxy.example:8080",
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="Compatibilidad con PluralKit">
    Habilite la resolución de PluralKit para asignar los mensajes enviados mediante proxy a la identidad del miembro del sistema:

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // opcional; necesario para sistemas privados
      },
    },
  },
}
```

    Notas:

    - las listas de permitidos pueden usar `pk:<memberId>`
    - los nombres visibles de los miembros se comparan por nombre/slug solo cuando `channels.discord.dangerouslyAllowNameMatching: true`
    - las consultas acceden a la API de PluralKit con el identificador del mensaje original
    - si la consulta falla, los mensajes enviados mediante proxy se consideran mensajes de bot y se descartan, salvo que `allowBots` permita su paso

  </Accordion>

  <Accordion title="Alias de menciones salientes">
    Use `mentionAliases` cuando los agentes necesiten menciones salientes deterministas de usuarios conocidos de Discord. Las claves son identificadores sin el `@` inicial; los valores son identificadores de usuario de Discord. Los identificadores desconocidos, `@everyone`, `@here` y las menciones dentro de fragmentos de código Markdown se dejan sin cambios.

```json5
{
  channels: {
    discord: {
      mentionAliases: {
        SupportLead: "123456789012345678",
      },
      accounts: {
        ops: {
          mentionAliases: {
            OpsLead: "234567890123456789",
          },
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="Configuración de presencia">
    Las actualizaciones de presencia se aplican cuando se establece un campo de estado o actividad, o cuando se habilita la presencia automática.

    Solo estado:

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    Actividad (el estado personalizado es el tipo de actividad predeterminado cuando se establece `activity`):

```json5
{
  channels: {
    discord: {
      activity: "Tiempo de concentración",
      activityType: 4,
    },
  },
}
```

    Transmisión:

```json5
{
  channels: {
    discord: {
      activity: "Programación en directo",
      activityType: 1,
      activityUrl: "https://twitch.tv/openclaw",
    },
  },
}
```

    Mapa de tipos de actividad:

    - 0: Jugando
    - 1: Transmitiendo (requiere `activityUrl`; `activityUrl`, a su vez, requiere `activityType: 1`)
    - 2: Escuchando
    - 3: Viendo
    - 4: Personalizada (usa el texto de la actividad como estado; el emoji es opcional)
    - 5: Compitiendo

    Presencia automática (señal de estado del entorno de ejecución):

```json5
{
  channels: {
    discord: {
      autoPresence: {
        enabled: true,
        intervalMs: 30000,
        minUpdateIntervalMs: 15000,
        exhaustedText: "tokens agotados",
      },
    },
  },
}
```

    La presencia automática asigna la disponibilidad del entorno de ejecución al estado de Discord: correcto => conectado, degradado o desconocido => inactivo, agotado o no disponible => no molestar. Valores predeterminados: `intervalMs` 30000, `minUpdateIntervalMs` 15000 (debe ser menor o igual que `intervalMs`). Sustituciones de texto opcionales:

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText` (admite el marcador de posición `{reason}`)

  </Accordion>

  <Accordion title="Aprobaciones en Discord">
    Discord admite la gestión de aprobaciones mediante botones en mensajes directos y, opcionalmente, puede publicar solicitudes de aprobación en el canal de origen.

    Ruta de configuración:

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers` (opcional; recurre a `commands.ownerAllowFrom` cuando es posible)
    - `channels.discord.execApprovals.target` (`dm` | `channel` | `both`, valor predeterminado: `dm`)
    - `agentFilter`, `sessionFilter`, `cleanupAfterResolve`

    Discord habilita automáticamente las aprobaciones nativas de ejecución cuando `enabled` no está establecido o es `"auto"`, y se puede resolver al menos un aprobador desde `execApprovals.approvers` o desde `commands.ownerAllowFrom`. Discord no deduce los aprobadores de ejecución a partir de `allowFrom` del canal, el valor heredado `dm.allowFrom` ni `defaultTo` de los mensajes directos. Establezca `enabled: false` para desactivar explícitamente Discord como cliente nativo de aprobaciones.

    Para comandos de grupo confidenciales y exclusivos del propietario, como `/diagnostics` y `/export-trajectory`, OpenClaw envía en privado las solicitudes de aprobación y los resultados finales. Primero intenta usar un mensaje directo de Discord cuando el propietario que invoca el comando tiene una ruta de propietario de Discord; de lo contrario, recurre a la primera ruta de propietario disponible de `commands.ownerAllowFrom`, como Telegram.

    Cuando `target` es `channel` o `both`, la solicitud de aprobación es visible en el canal. Solo los aprobadores resueltos pueden usar los botones; los demás usuarios reciben una denegación efímera. Las solicitudes de aprobación incluyen el texto del comando, por lo que la entrega en el canal solo debe habilitarse en canales de confianza. Si el identificador del canal no puede derivarse de la clave de sesión, OpenClaw recurre a la entrega por mensaje directo.

    Discord representa los botones de aprobación compartidos que usan otros canales de chat; el adaptador nativo de Discord añade principalmente el enrutamiento de mensajes directos a los aprobadores y la distribución a canales. Cuando esos botones están presentes, constituyen la experiencia principal de aprobación; OpenClaw solo debe incluir un comando manual `/approve` cuando el resultado de la herramienta indique que las aprobaciones por chat no están disponibles o que la aprobación manual es la única vía. Si el entorno de ejecución de aprobaciones nativas de Discord no está activo, OpenClaw mantiene visible la solicitud determinista local `/approve <id> <decision>`. Si el entorno de ejecución está activo, pero no se puede entregar una tarjeta nativa a ningún destino, OpenClaw envía un aviso alternativo en el mismo chat con el comando `/approve` exacto de la aprobación pendiente.

    La autenticación del Gateway y la resolución de aprobaciones siguen el contrato compartido del cliente del Gateway (los identificadores `plugin:` se resuelven mediante `plugin.approval.resolve`; los demás identificadores, mediante `exec.approval.resolve`). Las aprobaciones caducan después de 30 minutos de forma predeterminada.

    Consulte [Aprobaciones de ejecución](/es/tools/exec-approvals).

  </Accordion>
</AccordionGroup>

## Herramientas y controles de acciones

Las acciones de mensajes de Discord abarcan la mensajería, la administración de canales, la moderación, la presencia y los metadatos.

Ejemplos principales:

- mensajería: `sendMessage`, `readMessages`, `editMessage`, `deleteMessage`, `threadReply`
- reacciones: `react`, `reactions`, `emojiList`
- moderación: `timeout`, `kick`, `ban`
- presencia: `setPresence`

La acción `event-create` acepta un parámetro opcional `image` (URL o ruta de archivo local) para establecer la imagen de portada del evento programado.

Los controles de acciones se encuentran en `channels.discord.actions.*`.

Comportamiento predeterminado de los controles:

| Grupo de acciones                                                                                                                                                         | Valor predeterminado |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------- |
| reactions, messages, threads, pins, polls, search, memberInfo, roleInfo, channelInfo, channels, voiceStatus, events, stickers, emojiUploads, stickerUploads, permissions | habilitado           |
| roles                                                                                                                                                                    | deshabilitado        |
| moderation                                                                                                                                                               | deshabilitado        |
| presence                                                                                                                                                                 | deshabilitado        |

## Interfaz de usuario de componentes v2

OpenClaw utiliza los componentes v2 de Discord para las aprobaciones de ejecución y los marcadores entre contextos. Las acciones de mensajes de Discord también pueden aceptar `components` para interfaces de usuario personalizadas (avanzado; requiere construir una carga útil de componentes mediante la herramienta de Discord), mientras que los `embeds` heredados siguen disponibles, pero no se recomiendan.

- `channels.discord.ui.components.accentColor` establece el color de énfasis utilizado por los contenedores de componentes de Discord (hexadecimal). Por cuenta: `channels.discord.accounts.<id>.ui.components.accentColor`.
- `channels.discord.agentComponents.ttlMs` controla durante cuánto tiempo permanecen registradas las devoluciones de llamada de los componentes de Discord enviados (valor predeterminado: `1800000`; máximo: `86400000`). Por cuenta: `channels.discord.accounts.<id>.agentComponents.ttlMs`.
- `embeds` se ignoran cuando hay componentes v2 presentes.
- Las vistas previas de URL simples se suprimen de forma predeterminada. Establezca `suppressEmbeds: false` en una acción de mensaje cuando se deba expandir un único enlace saliente.

Ejemplo:

```json5
{
  channels: {
    discord: {
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
    },
  },
}
```

## Voz

Discord tiene dos superficies de voz distintas: **canales de voz** en tiempo real (conversaciones continuas) y **archivos adjuntos de mensajes de voz** (el formato de vista previa de la forma de onda). El Gateway admite ambas.

### Canales de voz

Lista de comprobación de configuración:

1. Habilite Message Content Intent en Discord Developer Portal.
2. Habilite Server Members Intent cuando se utilicen listas de usuarios o roles permitidos.
3. Invite al bot con los ámbitos `bot` y `applications.commands`.
4. Conceda Connect, Speak, Send Messages y Read Message History en el canal de voz de destino.
5. Habilite los comandos nativos (`commands.native` o `channels.discord.commands.native`).
6. Configure `channels.discord.voice`.

Utilice `/vc join|leave|status` para controlar las sesiones. El comando utiliza el agente predeterminado de la cuenta y sigue las mismas reglas de listas de permitidos y políticas de grupo que los demás comandos de Discord.

```bash
/vc join channel:<voice-channel-id>
/vc status
/vc leave
```

Para consultar los permisos efectivos del bot antes de unirse:

```bash
openclaw channels capabilities --channel discord --target channel:<voice-channel-id>
```

Ejemplo de unión automática:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        model: "openai/gpt-5.6-sol",
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        allowedChannels: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        connectTimeoutMs: 30000,
        reconnectGraceMs: 15000,
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

Notas:

- La voz de Discord es opcional para las configuraciones de solo texto; establezca `channels.discord.voice.enabled=true` (o conserve un bloque `channels.discord.voice` existente) para habilitar los comandos `/vc`, el entorno de ejecución de voz y la intención del Gateway `GuildVoiceStates`. `channels.discord.intents.voiceStates` puede anular explícitamente la suscripción a la intención; déjelo sin establecer para seguir la habilitación efectiva de la voz.
- `voice.mode` controla la ruta de conversación. El valor predeterminado es `agent-proxy`: una interfaz de voz en tiempo real gestiona los tiempos de los turnos, las interrupciones y la reproducción, delega el trabajo sustancial al agente de OpenClaw enrutado mediante `openclaw_agent_consult` y trata el resultado como una solicitud escrita de Discord de ese hablante. `stt-tts` conserva el flujo anterior de STT por lotes más TTS. `bidi` permite que el modelo en tiempo real converse directamente mientras expone `openclaw_agent_consult` para el cerebro de OpenClaw.
- `voice.agentSession` controla qué conversación de OpenClaw recibe los turnos de voz. Déjelo sin establecer para usar la sesión propia del canal de voz, o establezca `{ mode: "target", target: "channel:<text-channel-id>" }` para que el canal de voz actúe como extensión de micrófono y altavoz de una sesión existente de un canal de texto de Discord, como `#maintainers`.
- `voice.model` anula el cerebro del agente de OpenClaw para las respuestas de voz de Discord y las consultas en tiempo real. Déjelo sin establecer para heredar el modelo del agente enrutado. Es independiente de `voice.realtime.model`.
- `voice.followUsers` permite que el bot se una, se desplace y salga de la voz de Discord junto con los usuarios seleccionados. Consulte [Seguir usuarios en la voz](#follow-users-in-voice).
- `agent-proxy` enruta el habla mediante `discord-voice`, que conserva la autorización normal del propietario y de las herramientas para el hablante y la sesión de destino, pero oculta la herramienta `tts` del agente porque la voz de Discord controla la reproducción. De forma predeterminada, `agent-proxy` concede a la consulta acceso completo a herramientas equivalente al del propietario para los hablantes propietarios (`voice.realtime.toolPolicy: "owner"`) y prioriza firmemente consultar al agente de OpenClaw antes de ofrecer respuestas sustanciales (`voice.realtime.consultPolicy: "always"`). En ese modo `always` predeterminado, la capa en tiempo real no pronuncia automáticamente contenido de relleno antes de la respuesta de la consulta; captura y transcribe el habla y, a continuación, pronuncia la respuesta enrutada de OpenClaw. Si varias respuestas de consultas forzadas terminan mientras Discord aún reproduce la primera respuesta, las respuestas posteriores con el texto exacto que se debe pronunciar se ponen en cola hasta que la reproducción queda inactiva, en lugar de sustituir el habla a mitad de una frase.
- En el modo `stt-tts`, STT utiliza `tools.media.audio`; `voice.model` no afecta a la transcripción.
- En los modos en tiempo real, `voice.realtime.provider`, `voice.realtime.model` y `voice.realtime.speakerVoice` configuran la sesión de audio en tiempo real. Para OpenAI Realtime 2.1 con el cerebro Codex, utilice `voice.realtime.model: "gpt-realtime-2.1"` y `voice.model: "openai/gpt-5.6-sol"`.
- De forma predeterminada, los modos de voz en tiempo real incluyen pequeños archivos de perfil `IDENTITY.md`, `USER.md` y `SOUL.md` en las instrucciones del proveedor en tiempo real, para que los turnos directos rápidos mantengan la misma identidad, fundamentación sobre el usuario y personalidad que el agente de OpenClaw enrutado. Establezca `voice.realtime.bootstrapContextFiles` en un subconjunto para personalizarlo, o `[]` para deshabilitarlo. Solo se admiten esos archivos de perfil; `AGENTS.md` permanece en el contexto normal del agente. El contexto de perfil inyectado no sustituye a `openclaw_agent_consult` para el trabajo en el espacio de trabajo, los hechos actuales, la consulta de memoria o las acciones respaldadas por herramientas.
- En el modo en tiempo real `agent-proxy` de OpenAI, el control mediante nombre de activación se adapta de forma predeterminada a la sala: una persona puede hablar con naturalidad sin un nombre de activación, mientras que dos o más personas deben comenzar o terminar cada turno con uno. Los demás bots no cuentan como personas. Establezca `voice.realtime.requireWakeName: true` para exigir siempre un nombre de activación o `false` para no exigirlo nunca. Los nombres de activación configurados deben tener una o dos palabras. Si `voice.realtime.wakeNames` no está establecido, OpenClaw utiliza el `name` del agente enrutado junto con `OpenClaw` y, como alternativa, el identificador del agente junto con `OpenClaw`. Un control activo mediante nombre de activación deshabilita la respuesta automática del proveedor en tiempo real, enruta los turnos aceptados por la ruta de consulta del agente de OpenClaw y proporciona una breve confirmación hablada cuando se reconoce un nombre de activación inicial a partir de una transcripción parcial antes de que llegue la transcripción final. La política sigue las incorporaciones y salidas en directo sin volver a conectar la voz.
- El proveedor en tiempo real de OpenAI acepta los nombres de eventos actuales de Realtime 2 y los alias heredados compatibles con Codex para los eventos de audio de salida y transcripción, por lo que las instantáneas compatibles del proveedor pueden divergir sin perder el audio del asistente.
- `voice.realtime.bargeIn` controla si los eventos de inicio del habla de Discord interrumpen la reproducción activa en tiempo real. Si no está establecido, sigue la configuración de interrupción por audio de entrada del proveedor en tiempo real.
- `voice.realtime.minBargeInAudioEndMs` controla la duración mínima de reproducción del asistente antes de que una interrupción de OpenAI en tiempo real trunque el audio. Valor predeterminado: `250`. Establezca `0` para una interrupción inmediata en salas con poco eco, o auméntelo para configuraciones de altavoces con mucho eco.
- `voice.tts` anula `tts` solo para la reproducción de voz `stt-tts`; los modos en tiempo real utilizan `voice.realtime.speakerVoice` en su lugar. Para usar una voz de OpenAI en la reproducción de Discord, establezca `voice.tts.provider: "openai"` y elija una voz de texto a voz en `voice.tts.providers.openai.speakerVoice`. `cedar` es una buena opción de sonido masculino en el modelo TTS actual de OpenAI.
- Las anulaciones `systemPrompt` de Discord por canal se aplican a los turnos de transcripción de voz de ese canal de voz.
- Cuando OpenClaw se une a un canal de voz, la sesión del agente enrutado recibe un evento silencioso del sistema con la lista actual de participantes. Las incorporaciones y salidas posteriores de participantes actualizan esa sesión sin provocar una respuesta hablada no solicitada; los nombres para mostrar de Discord se tratan como etiquetas no fiables. Los turnos de voz autorizados también reciben una instantánea actualizada de la lista de participantes.
- Los turnos de transcripción de voz y los comandos `/vc` utilizan las entradas de Discord en `commands.ownerAllowFrom` para determinar el estado de propietario. Cuando no se configura ningún propietario de comandos de Discord, el `allowFrom` de la cuenta de Discord seleccionada (o el `dm.allowFrom` heredado) aún puede autorizar el acceso por voz sin conceder el estado de propietario. La visibilidad de las herramientas del agente sigue la política de herramientas configurada para la sesión enrutada.
- Si `voice.autoJoin` tiene varias entradas para el mismo servidor, OpenClaw se une al último canal configurado para ese servidor.
- `voice.allowedChannels` es una lista de permitidos opcional para la permanencia. Déjela sin establecer para permitir `/vc join` en cualquier canal de voz de Discord autorizado. Cuando está establecida, `/vc join`, la unión automática durante el inicio y los desplazamientos del estado de voz del bot quedan restringidos a las entradas `{ guildId, channelId }` enumeradas. Establézcala en una matriz vacía para denegar todas las uniones a la voz de Discord. Si Discord desplaza el bot fuera de la lista de permitidos, OpenClaw sale de ese canal y vuelve a unirse al destino de unión automática configurado cuando hay uno disponible.
- `voice.daveEncryption` y `voice.decryptionFailureTolerance` se transfieren a las opciones de unión de `@discordjs/voice`; los valores predeterminados del componente de nivel superior son `daveEncryption=true` y `decryptionFailureTolerance=24`.
- OpenClaw utiliza el códec `libopus-wasm` incluido para la recepción de voz de Discord y la reproducción de PCM sin procesar en tiempo real. Incluye una compilación WebAssembly fijada de libopus y no requiere complementos nativos de opus.
- `voice.connectTimeoutMs` controla la espera inicial de Ready de `@discordjs/voice` para `/vc join` y los intentos de unión automática. Valor predeterminado: `30000`.
- `voice.reconnectGraceMs` controla cuánto tiempo espera OpenClaw a que una sesión de voz desconectada comience a reconectarse antes de destruirla. Valor predeterminado: `15000`.
- En el modo `stt-tts`, la reproducción de voz no se detiene únicamente porque otro usuario empiece a hablar. Para evitar bucles de realimentación, OpenClaw ignora las nuevas capturas de voz mientras se reproduce TTS; hable después de que termine la reproducción para iniciar el siguiente turno. Los modos en tiempo real reenvían los inicios del habla como señales de interrupción al proveedor en tiempo real.
- En los modos en tiempo real, el eco de los altavoces que entra en un micrófono abierto puede parecer una interrupción y detener la reproducción. Para salas de Discord con mucho eco, establezca `voice.realtime.providers.openai.interruptResponseOnInputAudio: false` para impedir que OpenAI interrumpa automáticamente por el audio de entrada. Añada `voice.realtime.bargeIn: true` si aun así desea que los eventos de inicio del habla de Discord interrumpan la reproducción activa. El puente en tiempo real de OpenAI ignora como probable eco o ruido los truncamientos de reproducción inferiores a `voice.realtime.minBargeInAudioEndMs` y los registra como omitidos en lugar de borrar la reproducción de Discord.
- `voice.captureSilenceGraceMs` controla cuánto tiempo espera OpenClaw después de que Discord indique que un hablante ha dejado de hablar antes de finalizar ese segmento de audio para STT. Valor predeterminado: `2000`; auméntelo si Discord divide las pausas normales en transcripciones parciales entrecortadas.
- Cuando ElevenLabs es el proveedor de TTS seleccionado, la reproducción de voz de Discord utiliza TTS por streaming y comienza a partir del flujo de respuesta del proveedor. Los proveedores sin compatibilidad con streaming recurren a la ruta del archivo temporal sintetizado.
- OpenClaw supervisa los fallos de descifrado de recepción y se recupera automáticamente saliendo del canal de voz y volviendo a unirse después de varios fallos en un intervalo breve.
- Si los registros de recepción muestran repetidamente `DecryptionFailed(UnencryptedWhenPassthroughDisabled)` después de actualizar, recopile un informe de dependencias y los registros. La línea `@discordjs/voice` incluida contiene la corrección de relleno del componente de nivel superior procedente del PR #11449 de discord.js, que cerró el issue #11419 de discord.js.
- Los eventos de recepción `The operation was aborted` son esperados cuando OpenClaw finaliza un segmento capturado de un hablante; son diagnósticos detallados, no advertencias.
- Los registros detallados de voz de Discord incluyen una vista previa acotada de una línea de la transcripción STT para cada segmento de hablante aceptado, de modo que la depuración muestre tanto el lado del usuario como el de la respuesta del agente sin volcar texto de transcripción sin límites.
- En el modo `agent-proxy`, la alternativa de consulta forzada omite fragmentos de transcripción probablemente incompletos, como texto que termina en `...` o un conector final como «y», además de cierres evidentemente no procesables como «vuelvo enseguida» o «adiós». Los registros muestran `forced agent consult skipped reason=...` cuando esto evita una respuesta obsoleta en cola.

### Seguir usuarios en la voz

Utilice `voice.followUsers` cuando desee que el bot de voz de Discord permanezca con uno o varios usuarios conocidos de Discord, en lugar de unirse a un canal fijo durante el inicio o esperar a `/vc join`.

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        followUsersEnabled: true,
        followUsers: ["discord:123456789012345678"],
        allowedChannels: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
      },
    },
  },
}
```

Comportamiento:

- `followUsers` acepta ID de usuario de Discord sin procesar y valores `discord:<id>`. OpenClaw normaliza ambas formas antes de comparar los eventos de estado de voz.
- `followUsersEnabled` tiene como valor predeterminado `true` cuando se configura `followUsers`. Establézcalo en `false` para conservar la lista guardada, pero detener el seguimiento de voz automático.
- `followUsers` solo controla la permanencia en el canal de voz. No concede acceso como hablante ni autoridad de propietario; configure `commands.ownerAllowFrom` y los usuarios y roles del servidor o canal por separado.
- Cuando un usuario seguido se une a un canal de voz permitido, OpenClaw se une a ese canal. Cuando el usuario cambia de canal, OpenClaw cambia con él. Cuando el usuario seguido activo se desconecta, OpenClaw abandona el canal.
- Si hay varios usuarios seguidos en el mismo servidor y el usuario seguido activo se va, OpenClaw cambia al canal de otro usuario seguido rastreado antes de abandonar el servidor. Si varios usuarios seguidos cambian de canal a la vez, prevalece el evento de estado de voz observado más recientemente.
- `allowedChannels` continúa aplicándose. Se ignora a un usuario seguido que esté en un canal no permitido, y una sesión gestionada mediante seguimiento cambia a otro usuario seguido o abandona el canal.
- OpenClaw concilia los eventos de estado de voz omitidos al iniciarse y en intervalos acotados. La conciliación toma muestras de los servidores configurados y limita las consultas REST por ejecución, por lo que las listas `followUsers` muy grandes pueden necesitar más de un intervalo para converger.
- Si Discord o un administrador mueve el bot mientras sigue a un usuario, OpenClaw reconstruye la sesión de voz y conserva la propiedad del seguimiento cuando el destino está permitido. Si se mueve el bot fuera de `allowedChannels`, OpenClaw abandona el canal y vuelve a unirse al destino configurado cuando existe uno.
- La recuperación de recepción de DAVE puede abandonar el canal y volver a unirse a él después de varios errores de descifrado. Las sesiones gestionadas mediante seguimiento conservan la propiedad del seguimiento durante ese proceso de recuperación, por lo que una desconexión posterior del usuario seguido aún hace que se abandone el canal.

Elija entre los modos de unión:

- Use `followUsers` para configuraciones personales o de operadores en las que el bot deba estar automáticamente en el canal de voz cuando usted lo esté.
- Use `autoJoin` para bots de sala fija que deban estar presentes incluso cuando ningún usuario rastreado esté en el canal de voz.
- Use `/vc join` para uniones puntuales o salas en las que una presencia de voz automática resultaría inesperada.

Códec de voz de Discord:

- Los registros de recepción de voz muestran `discord voice: opus decoder: libopus-wasm`.
- La reproducción en tiempo real codifica PCM estéreo sin procesar de 48 kHz a Opus con el mismo paquete `libopus-wasm` incluido antes de entregar los paquetes a `@discordjs/voice`.
- La reproducción de archivos y flujos de proveedores transcodifica a PCM estéreo sin procesar de 48 kHz con ffmpeg y luego usa `libopus-wasm` para el flujo de paquetes Opus enviado a Discord.

Pipeline de STT y TTS:

- La captura PCM de Discord se convierte en un archivo WAV temporal.
- `tools.media.audio` gestiona STT, por ejemplo, `openai/gpt-4o-mini-transcribe`.
- La transcripción se envía a través de la entrada y el enrutamiento de Discord mientras el LLM de respuesta se ejecuta con una política de salida de voz que oculta la herramienta `tts` del agente y solicita que se devuelva texto, porque la voz de Discord gestiona la reproducción TTS final.
- `voice.model`, cuando se establece, sustituye únicamente el LLM de respuesta para este turno del canal de voz.
- `voice.tts` se combina sobre `tts`; los proveedores compatibles con streaming envían el audio directamente al reproductor; de lo contrario, el archivo de audio resultante se reproduce en el canal al que se ha unido.

Ejemplo de sesión de canal de voz predeterminada con proxy de agente:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        model: "openai/gpt-5.6-sol",
        followUsersEnabled: true,
        followUsers: ["123456789012345678"],
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

Sin un bloque `voice.agentSession`, cada canal de voz obtiene su propia sesión de OpenClaw enrutada. Por ejemplo, `/vc join channel:234567890123456789` se comunica con la sesión de ese canal de voz de Discord. El modelo en tiempo real es solo la interfaz de voz; las solicitudes sustanciales se entregan al agente de OpenClaw configurado. Si el modelo en tiempo real genera una transcripción final sin llamar a la herramienta de consulta, OpenClaw fuerza la consulta como mecanismo alternativo para que el comportamiento predeterminado siga siendo equivalente a hablar con el agente.

Ejemplo de STT y TTS heredado:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "stt-tts",
        model: "openai/gpt-5.4-mini",
        tts: {
          provider: "openai",
          providers: {
            openai: {
              model: "gpt-4o-mini-tts",
              speakerVoice: "cedar",
            },
          },
        },
      },
    },
  },
}
```

Ejemplo bidireccional en tiempo real:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "bidi",
        model: "openai/gpt-5.6-sol",
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
          toolPolicy: "safe-read-only",
          consultPolicy: "always",
        },
      },
    },
  },
}
```

Voz como extensión de una sesión existente de canal de Discord:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "agent-proxy",
        model: "openai/gpt-5.6-sol",
        agentSession: {
          mode: "target",
          target: "channel:123456789012345678",
        },
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
    },
  },
}
```

En el modo `agent-proxy`, el bot se une al canal de voz configurado, pero los turnos del agente de OpenClaw usan la sesión enrutada y el agente normales del canal de destino. La sesión de voz en tiempo real reproduce oralmente el resultado devuelto en el canal de voz. El agente supervisor puede seguir usando las herramientas de mensajes normales de acuerdo con su política de herramientas, incluido el envío de un mensaje independiente de Discord si esa es la acción adecuada.

Mientras hay una ejecución delegada de OpenClaw activa, las nuevas transcripciones de voz de Discord se tratan como control en directo de la ejecución antes de iniciar otro turno del agente. Frases como «estado», «cancela eso», «usa la corrección más pequeña» o «cuando termines, comprueba también las pruebas» se clasifican como entrada de estado, cancelación, orientación o seguimiento para la sesión activa. Los resultados de estado, cancelación, orientación aceptada y seguimiento se reproducen oralmente en el canal de voz para que quien llama sepa si OpenClaw gestionó la solicitud.

Formas de destino útiles:

- `target: "channel:123456789012345678"` se enruta a través de una sesión de canal de texto de Discord.
- `target: "123456789012345678"` se trata como un destino de canal.
- `target: "dm:123456789012345678"` o `target: "user:123456789012345678"` se enruta a través de esa sesión de mensajes directos.

Ejemplo de OpenAI Realtime con mucho eco:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        mode: "bidi",
        model: "openai/gpt-5.6-sol",
        realtime: {
          provider: "openai",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
          bargeIn: true,
          minBargeInAudioEndMs: 500,
          consultPolicy: "always",
          providers: {
            openai: {
              interruptResponseOnInputAudio: false,
            },
          },
        },
      },
    },
  },
}
```

Use esta configuración cuando el modelo oiga su propia reproducción de Discord a través de un micrófono abierto, pero aun así se quiera poder interrumpirlo hablando. OpenClaw impide que OpenAI interrumpa automáticamente a partir del audio de entrada sin procesar, mientras que `bargeIn: true` permite que los eventos de inicio de habla de Discord y el audio de un hablante ya activo cancelen las respuestas en tiempo real activas antes de que el siguiente turno capturado llegue a OpenAI. Las señales de interrupción muy tempranas con `audioEndMs` por debajo de `minBargeInAudioEndMs` se consideran probablemente eco o ruido y se ignoran para que el modelo no se interrumpa en el primer fotograma de reproducción.

Registros de voz esperados:

- Al unirse: `discord voice: joining ... voiceSession=... supervisorSession=... agentSessionMode=... voiceModel=... realtimeModel=...`
- Al iniciar el modo en tiempo real: `discord voice: realtime bridge starting ... autoRespond=false interruptResponse=false bargeIn=false minBargeInAudioEndMs=...`
- Al recibir audio de un hablante: `discord voice: realtime speaker turn opened ...`, `discord voice: realtime input audio started ... outputAudioMs=... outputActive=...` y `discord voice: realtime speaker turn closed ... chunks=... discordBytes=... realtimeBytes=... interruptedPlayback=...`
- Al omitir habla obsoleta: `discord voice: realtime forced agent consult skipped reason=incomplete-transcript ...` o `reason=non-actionable-closing ...`
- Al completarse la respuesta en tiempo real: `discord voice: realtime audio playback finishing reason=response.done ... audioMs=... chunks=...`
- Al detener o restablecer la reproducción: `discord voice: realtime audio playback stopped reason=... audioMs=... elapsedMs=... chunks=...`
- Al realizar una consulta en tiempo real: `discord voice: realtime consult requested ... voiceSession=... supervisorSession=... question=...`
- Al responder el agente: `discord voice: agent turn answer ...`
- Al poner en cola habla exacta: `discord voice: realtime exact speech queued ... queued=... outputAudioMs=... outputActive=...`, seguido de `discord voice: realtime exact speech dequeued reason=player-idle ...`
- Al detectar una interrupción mediante voz: `discord voice: realtime barge-in detected source=speaker-start ...` o `discord voice: realtime barge-in detected source=active-speaker-audio ...`, seguido de `discord voice: realtime barge-in requested reason=... outputAudioMs=... outputActive=...`
- Al interrumpir el modo en tiempo real: `discord voice: realtime model interrupt requested client:response.cancel reason=barge-in`, seguido de `discord voice: realtime model audio truncated client:conversation.item.truncate reason=barge-in audioEndMs=...` o `discord voice: realtime model interrupt confirmed server:response.done status=cancelled ...`
- Al ignorar eco o ruido: `discord voice: realtime model interrupt ignored client:conversation.item.truncate.skipped reason=barge-in audioEndMs=0 minAudioEndMs=250`
- Al desactivar la interrupción mediante voz: `discord voice: realtime capture ignored during playback (barge-in disabled) ...`
- Durante la reproducción inactiva: `discord voice: realtime barge-in ignored reason=... outputActive=false ... playbackChunks=0`

Para depurar audio entrecortado, lea los registros de voz en tiempo real como una cronología:

1. `realtime audio playback started` significa que Discord ha comenzado a reproducir audio del asistente. A partir de este punto, el puente empieza a contar los fragmentos de salida del asistente, los bytes PCM de Discord, los bytes en tiempo real del proveedor y la duración del audio sintetizado.
2. `realtime speaker turn opened` indica que un hablante de Discord pasa a estar activo. Si la reproducción ya está activa y `bargeIn` está habilitado, puede ir seguido de `barge-in detected source=speaker-start`.
3. `realtime input audio started` indica el primer fotograma de audio real recibido para ese turno del hablante. `outputActive=true` o un valor `outputAudioMs` distinto de cero en este punto significa que el micrófono está enviando entrada mientras la reproducción del asistente continúa activa.
4. `barge-in detected source=active-speaker-audio` significa que OpenClaw detectó audio en directo del hablante mientras la reproducción del asistente estaba activa. Esto resulta útil para distinguir una interrupción real de un evento de inicio de habla de Discord sin audio útil.
5. `barge-in requested reason=...` significa que OpenClaw solicitó al proveedor en tiempo real cancelar o truncar la respuesta activa. Incluye `outputAudioMs`, `outputActive` y `playbackChunks` para mostrar cuánto audio del asistente se había reproducido realmente antes de la interrupción.
6. `realtime audio playback stopped reason=...` es el punto de restablecimiento local de la reproducción de Discord. El motivo indica quién detuvo la reproducción: `barge-in`, `player-idle`, `provider-clear-audio`, `forced-agent-consult`, `stream-close` o `session-close`.
7. `realtime speaker turn closed` resume el turno de entrada capturado. `chunks=0` o `hasAudio=false` significa que el turno del hablante se abrió, pero ningún audio utilizable llegó al puente en tiempo real. `interruptedPlayback=true` significa que ese turno de entrada se solapó con la salida del asistente y activó la lógica de interrupción mediante voz.

Campos útiles:

- `outputAudioMs`: duración del audio del asistente generado por el proveedor en tiempo real antes de la línea del registro.
- `audioMs`: duración del audio del asistente que OpenClaw contabilizó antes de que se detuviera la reproducción.
- `elapsedMs`: tiempo de reloj transcurrido entre la apertura y el cierre del flujo de reproducción o del turno del hablante.
- `discordBytes`: bytes PCM estéreo de 48 kHz enviados a la voz de Discord o recibidos de ella.
- `realtimeBytes`: bytes PCM en el formato del proveedor enviados al proveedor en tiempo real o recibidos de él.
- `playbackChunks`: fragmentos de audio del asistente reenviados a Discord para la respuesta activa.
- `sinceLastAudioMs`: intervalo entre el último fotograma de audio capturado del hablante y el cierre de su turno.

Patrones comunes:

- La interrupción inmediata con `source=active-speaker-audio`, un valor bajo de `outputAudioMs` y el mismo usuario cerca suele indicar que el eco del altavoz está entrando en el micrófono. Aumente `voice.realtime.minBargeInAudioEndMs`, reduzca el volumen del altavoz, use auriculares o configure `voice.realtime.providers.openai.interruptResponseOnInputAudio: false`.
- `source=speaker-start` seguido de `speaker turn closed ... hasAudio=false` significa que Discord notificó el inicio de un hablante, pero ningún audio llegó a OpenClaw. Puede deberse a un evento de voz transitorio de Discord, al comportamiento de la puerta de ruido o a que un cliente activó brevemente el micrófono.
- `audio playback stopped reason=stream-close` sin una interrupción cercana ni `provider-clear-audio` significa que el flujo de reproducción local de Discord terminó inesperadamente. Compruebe los registros anteriores del proveedor y del reproductor de Discord.
- `capture ignored during playback (barge-in disabled)` significa que OpenClaw descartó intencionadamente la entrada mientras el audio del asistente estaba activo. Habilite `voice.realtime.bargeIn` si desea que la voz interrumpa la reproducción.
- `barge-in ignored ... outputActive=false` significa que Discord o el VAD del proveedor detectaron voz, pero OpenClaw no tenía ninguna reproducción activa que interrumpir. Esto no debería cortar el audio.

Las credenciales se resuelven por componente: autenticación de la ruta del LLM para `voice.model`, autenticación de STT para `tools.media.audio`, autenticación de TTS para `tts`/`voice.tts` y autenticación del proveedor en tiempo real para `voice.realtime.providers` o la configuración de autenticación habitual del proveedor.

### Mensajes de voz

Los mensajes de voz de Discord muestran una vista previa de la forma de onda y requieren audio OGG/Opus. OpenClaw genera la forma de onda automáticamente, pero necesita `ffmpeg` y `ffprobe` en el host del Gateway para inspeccionar y convertir el audio.

- Proporcione una **ruta de archivo local** (se rechazan las URL).
- Omita el contenido de texto (Discord rechaza el texto y el mensaje de voz en la misma carga útil).
- Se acepta cualquier formato de audio; OpenClaw lo convierte a OGG/Opus según sea necesario.

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## Solución de problemas

<AccordionGroup>
  <Accordion title="Se usaron intents no permitidos o el bot no ve mensajes de servidores">

    - habilite Message Content Intent
    - habilite Server Members Intent cuando dependa de la resolución de usuarios o miembros
    - reinicie el Gateway después de cambiar los intents

  </Accordion>

  <Accordion title="Los mensajes del servidor se bloquean inesperadamente">

    - verifique `groupPolicy`
    - verifique la lista de permitidos del servidor en `channels.discord.guilds`
    - si existe un mapa `channels` del servidor, solo se permiten los canales enumerados
    - verifique el comportamiento de `requireMention` y los patrones de mención

    Comprobaciones útiles:

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="La mención no es obligatoria, pero sigue bloqueado">
    Causas habituales:

    - `groupPolicy="allowlist"` sin una lista de permitidos coincidente para el servidor o canal
    - `requireMention` configurado en el lugar incorrecto (debe estar en `channels.discord.guilds` o en una entrada de canal)
    - remitente bloqueado por la lista de permitidos `users` del servidor o canal

  </Accordion>

  <Accordion title="Turnos prolongados de Discord o respuestas duplicadas">

    Registros habituales:

    - `Slow listener detected ...`
    - `stuck session: sessionKey=agent:...:discord:... state=processing ...`

    Discord no aplica un tiempo de espera propiedad del canal a los turnos del agente en cola. Los receptores de mensajes transfieren el control inmediatamente y las ejecuciones de Discord en cola conservan el orden por sesión hasta que el ciclo de vida de la sesión, herramienta o entorno de ejecución finaliza o cancela el trabajo.

  </Accordion>

  <Accordion title="Advertencias de tiempo de espera al consultar metadatos del Gateway">
    OpenClaw obtiene los metadatos `/gateway/bot` de Discord antes de conectarse. Ante fallos transitorios, utiliza como alternativa la URL predeterminada del Gateway de Discord y limita la frecuencia de los registros.

    El tiempo de espera de los metadatos es de 30 segundos de forma predeterminada. `OPENCLAW_DISCORD_GATEWAY_INFO_TIMEOUT_MS` puede sustituirlo en entornos de host poco habituales.

  </Accordion>

  <Accordion title="Reinicios por tiempo de espera de READY del Gateway">
    OpenClaw espera el evento `READY` del Gateway de Discord durante el inicio y después de las reconexiones del entorno de ejecución. Las configuraciones con varias cuentas y un inicio escalonado pueden necesitar una ventana de espera de READY inicial más larga que la predeterminada.

    El inicio espera 15 segundos y las reconexiones del entorno de ejecución esperan 30 segundos. `OPENCLAW_DISCORD_READY_TIMEOUT_MS` y `OPENCLAW_DISCORD_RUNTIME_READY_TIMEOUT_MS` siguen disponibles para entornos de host poco habituales.

  </Accordion>

  <Accordion title="Discrepancias en la auditoría de permisos">
    Las comprobaciones de permisos de `channels status --probe` solo funcionan con identificadores numéricos de canal.

    Si se usan claves de slug, la coincidencia durante la ejecución puede seguir funcionando, pero la comprobación no puede verificar completamente los permisos.

  </Accordion>

  <Accordion title="Problemas con los mensajes directos y el emparejamiento">

    - mensajes directos deshabilitados: `channels.discord.dm.enabled=false`
    - política de mensajes directos deshabilitada: `channels.discord.dmPolicy="disabled"` (heredado: `channels.discord.dm.policy`)
    - esperando la aprobación del emparejamiento en el modo `pairing`

  </Accordion>

  <Accordion title="Bucles entre bots">
    De forma predeterminada, se ignoran los mensajes creados por bots.

    Si configura `channels.discord.allowBots=true`, use reglas estrictas de menciones y listas de permitidos para evitar bucles.
    Es preferible usar `channels.discord.allowBots="mentions"` para aceptar únicamente mensajes de bots que mencionen al bot.

    OpenClaw también incluye [protección contra bucles de bots](/es/channels/bot-loop-protection) compartida. Siempre que `allowBots` permita que los mensajes creados por bots lleguen al despacho, Discord asigna el evento entrante a hechos `(account, channel, bot pair)` y la protección genérica de pares suprime el par una vez que supera el límite de eventos configurado. La protección evita bucles descontrolados entre dos bots que antes debían detenerse mediante los límites de frecuencia de Discord; no afecta a las implementaciones con un solo bot ni a las respuestas puntuales de bots que se mantengan por debajo del límite.

    Configuración predeterminada (activa cuando se establece `allowBots`):

    - `maxEventsPerWindow: 20` -- el par de bots puede intercambiar 20 mensajes dentro de la ventana deslizante
    - `windowSeconds: 60` -- duración de la ventana deslizante
    - `cooldownSeconds: 60` -- una vez superado el límite, se descartan durante un minuto todos los mensajes adicionales entre bots en cualquier dirección

    Configure una vez el valor predeterminado compartido en `channels.defaults.botLoopProtection` y, a continuación, sustitúyalo para Discord cuando un flujo de trabajo legítimo necesite más margen. El orden de precedencia es:

    - `channels.discord.accounts.<account>.botLoopProtection`
    - `channels.discord.botLoopProtection`
    - `channels.defaults.botLoopProtection`
    - valores predeterminados integrados

    Discord usa las claves genéricas `maxEventsPerWindow`, `windowSeconds` y `cooldownSeconds`.

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
        windowSeconds: 60,
        cooldownSeconds: 60,
      },
    },
    discord: {
      // Sustitución opcional para todo Discord. Los bloques de cuenta sustituyen campos
      // individuales y heredan de aquí los campos omitidos.
      botLoopProtection: {
        maxEventsPerWindow: 4,
      },
      accounts: {
        alpha: {
          // Alpha solo escucha a otros bots cuando lo mencionan.
          allowBots: "mentions",
        },
        bravo: {
          // Bravo escucha todos los mensajes de Discord creados por bots.
          allowBots: true,
          mentionAliases: {
            // Permite que Bravo escriba una mención de Alpha en Discord con el identificador de usuario configurado.
            Alpha: "ALPHA_DISCORD_USER_ID",
          },
          botLoopProtection: {
            // Permite hasta cinco mensajes por minuto antes de suprimir el par.
            maxEventsPerWindow: 5,
            windowSeconds: 60,
            cooldownSeconds: 90,
          },
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="La STT de voz descarta audio con DecryptionFailed(...)">

    - mantenga OpenClaw actualizado (`openclaw update`) para disponer de la lógica de recuperación de recepción de voz de Discord
    - confirme `channels.discord.voice.daveEncryption=true` (valor predeterminado)
    - comience con `channels.discord.voice.decryptionFailureTolerance=24` (valor predeterminado del proyecto original) y ajústelo solo si es necesario
    - revise los registros para detectar:
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - si los fallos continúan después de la reconexión automática, recopile los registros y compárelos con el historial original de recepción de DAVE en [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419) y [discord.js #11449](https://github.com/discordjs/discord.js/pull/11449)

  </Accordion>
</AccordionGroup>

## Referencia de configuración

Referencia principal: [Referencia de configuración: Discord](/es/gateway/config-channels#discord).

<Accordion title="Campos principales de Discord">

- inicio/autenticación: `enabled`, `token`, `applicationId`, `accounts.*`, `allowBots`
- política: `groupPolicy`, `dmPolicy`, `allowFrom`, `dm.*`, `guilds.*`, `guilds.*.channels.*`
- comandos: `commands.native`, `commands.useAccessGroups` (global), `configWrites`, `slashCommand.ephemeral`
- Gateway: `proxy`
- respuestas/historial: `replyToMode`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
- entrega: `textChunkLimit` (valor predeterminado: `2000`), `maxLinesPerMessage` (valor predeterminado: `17`)
- transmisión: `streaming.mode`, `streaming.chunkMode`, `streaming.preview.*`, `streaming.progress.*`, `streaming.block.*` (las claves planas heredadas `streamMode`, `draftChunk`, `blockStreaming`, `blockStreamingCoalesce` y `chunkMode` se migran a `streaming.*` mediante `openclaw doctor --fix`)
- contenido multimedia: `mediaMaxMb` (limita las cargas salientes a Discord, valor predeterminado: `100`)
- acciones: `actions.*`
- presencia: `activity`, `status`, `activityType`, `activityUrl`, `autoPresence.*`
- interfaz de usuario: `ui.components.accentColor`
- funciones: `threadBindings`, `bindings[]` en el nivel superior (`type: "acp"`), `pluralkit`, `execApprovals`, `intents`, `agentComponents.enabled`, `agentComponents.ttlMs`, `activities`, `heartbeat`, `responsePrefix`

</Accordion>

### Actividades de Discord

Configure `channels.discord.activities` para permitir que los agentes publiquen widgets HTML autónomos que se abren dentro de Discord. El bloque es opcional; cuando no está presente, OpenClaw no registra rutas de actividades, herramientas ni controladores de interacciones. Consulte [Actividades de Discord](/es/channels/discord-activities) para configurar Developer Portal, el túnel, la seguridad y la solución de problemas.

- `activities.clientSecret`: secreto del cliente OAuth2 para la aplicación de Discord; utiliza `DISCORD_CLIENT_SECRET` como alternativa
- `activities.applicationId`: identificador opcional de la aplicación de actividades; el valor predeterminado es el identificador de la aplicación del bot obtenido al iniciar el Gateway

## Seguridad y operaciones

- Trate los tokens del bot como secretos (se prefiere `DISCORD_BOT_TOKEN` en entornos supervisados).
- Conceda los permisos mínimos necesarios en Discord.
- Si el despliegue o el estado de los comandos está obsoleto, reinicie el Gateway y vuelva a comprobarlo con `openclaw channels status --probe`.

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Actividades de Discord" icon="window" href="/es/channels/discord-activities">
    Inicie widgets HTML interactivos dentro de Discord.
  </Card>
  <Card title="Emparejamiento" icon="link" href="/es/channels/pairing">
    Empareje un usuario de Discord con el Gateway.
  </Card>
  <Card title="Grupos" icon="users" href="/es/channels/groups">
    Comportamiento de los chats grupales y las listas de permitidos.
  </Card>
  <Card title="Enrutamiento de canales" icon="route" href="/es/channels/channel-routing">
    Enrute los mensajes entrantes a los agentes.
  </Card>
  <Card title="Seguridad" icon="shield" href="/es/gateway/security">
    Modelo de amenazas y refuerzo de la seguridad.
  </Card>
  <Card title="Enrutamiento multiagente" icon="sitemap" href="/es/concepts/multi-agent">
    Asigne servidores y canales a los agentes.
  </Card>
  <Card title="Comandos de barra diagonal" icon="terminal" href="/es/tools/slash-commands">
    Comportamiento nativo de los comandos.
  </Card>
</CardGroup>
