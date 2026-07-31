---
read_when: You want multiple agents with separate workspaces, auth, and sessions in one Gateway process.
sidebarTitle: Multi-agent routing
status: active
summary: 'Enrutamiento multiagente: límites de los agentes, cuentas de canales y vinculaciones'
title: Enrutamiento multiagente
x-i18n:
    generated_at: "2026-07-26T05:08:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 46df162388205e46d5a4ea3567c8c8f7016117d2ecafe1184a35b4c95798fd80
    source_path: concepts/multi-agent.md
    workflow: 16
---

Ejecute varios agentes _aislados_ en un único proceso de Gateway, cada uno con su propio espacio de trabajo, directorio de estado (`agentDir`) e historial de sesiones respaldado por SQLite, además de varias cuentas de canales (por ejemplo, dos números de WhatsApp). Los mensajes entrantes se enrutan al agente correcto mediante **vinculaciones**.

Un **agente** es el ámbito completo de cada persona: archivos del espacio de trabajo, perfiles de autenticación, registro de modelos y almacén de sesiones. Una **vinculación** asigna una cuenta de canal (un espacio de trabajo de Slack, un número de WhatsApp, etc.) a uno de esos agentes.

## Qué es un agente

Cada agente tiene sus propios elementos:

- **Espacio de trabajo**: archivos, `AGENTS.md`/`SOUL.md`/`USER.md`, notas locales y reglas de la persona.
- **Directorio de estado** (`agentDir`): perfiles de autenticación, registro de modelos y configuración por agente.
- **Almacén de sesiones**: historial de chat y estado de enrutamiento en `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`.

Los perfiles de autenticación son específicos de cada agente y se leen desde:

```text
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

<Note>
`sessions_history` es la vía más segura para recuperar información entre sesiones: devuelve una vista limitada y redactada, no un volcado sin procesar de la transcripción. Elimina las firmas de los bloques de razonamiento, los detalles de la carga útil de los resultados de herramientas, la estructura auxiliar de `<relevant-memories>`, las etiquetas XML de llamadas a herramientas (`<tool_call>`, `<function_call>` y sus formas plurales o degradadas) y el XML de llamadas a herramientas de MiniMax; después, trunca y limita la salida por tamaño en bytes.
</Note>

<Warning>
Nunca reutilice `agentDir` entre agentes, ya que provoca colisiones en el estado de autenticación y de sesión. Cuando la credencial OAuth local de un agente secundario ha caducado o no se puede actualizar, OpenClaw consulta la credencial del agente predeterminado/principal para el mismo identificador de perfil y adopta el token más reciente, sin copiar el token de actualización en el almacén del agente secundario. Si desea una cuenta OAuth completamente independiente, inicie sesión desde ese agente. Si copia credenciales manualmente, copie únicamente perfiles estáticos portátiles `api_key` o `token`; el material de actualización de OAuth no es portátil de forma predeterminada (`copyToAgents` permite habilitarlo explícitamente para un perfil).
</Warning>

Las Skills se cargan desde el espacio de trabajo de cada agente y desde raíces compartidas como `~/.openclaw/skills`, y después se filtran mediante la lista efectiva de Skills permitidas para el agente. Use `agents.defaults.skills` como base compartida y `agents.entries.*.skills` como sustitución por agente (las entradas explícitas sustituyen el valor predeterminado; no se combinan). Consulte [Skills: por agente frente a compartidas](/es/tools/skills#per-agent-vs-shared-skills) y [Skills: listas permitidas de agentes](/es/tools/skills#agent-allowlists).

El almacenamiento propiedad de un plugin sigue la configuración de ese plugin; añadir un segundo agente
no divide automáticamente todos los almacenes globales de plugins. Por ejemplo, configure
[bóvedas de Memory Wiki por agente](/es/concepts/multi-agent#per-agent-memory-wiki-vaults)
cuando las personas no deban compartir el conocimiento compilado de la wiki.

<Note>
**Nota sobre el espacio de trabajo:** el espacio de trabajo de cada agente es el **cwd predeterminado**, no un entorno aislado estricto. Las rutas relativas se resuelven dentro del espacio de trabajo, pero las rutas absolutas pueden acceder a otras ubicaciones del host, a menos que se habilite el aislamiento. Consulte [Aislamiento](/es/gateway/sandboxing).
</Note>

## Rutas

| Elemento                         | Valor predeterminado                                                                    | Sustitución                                                                                 |
| -------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Configuración                    | `~/.openclaw/openclaw.json`                                                            | `OPENCLAW_CONFIG_PATH`                                                                      |
| Directorio de estado             | `~/.openclaw`                                                                          | `OPENCLAW_STATE_DIR`                                                                        |
| Espacio de trabajo del agente predeterminado | `~/.openclaw/workspace` (o `workspace-<profile>` cuando se establece `OPENCLAW_PROFILE`)      | `agents.entries.*.workspace`, después `agents.defaults.workspace`, o `OPENCLAW_WORKSPACE_DIR` |
| Espacio de trabajo de otros agentes | `<stateDir>/workspace-<agentId>` (o `<agents.defaults.workspace>/<agentId>` cuando se establece) | `agents.entries.*.workspace`                                                                |
| Directorio del agente            | `~/.openclaw/agents/<agentId>/agent`                                                   | `agents.entries.*.agentDir`                                                                 |
| Sesiones y transcripciones       | `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`                             | —                                                                                           |
| Artefactos de sesiones heredados/archivados | `~/.openclaw/agents/<agentId>/sessions`                                                | —                                                                                           |

### Modo de agente único (predeterminado)

Si no configura nada, OpenClaw ejecuta un agente:

- `agentId` tiene como valor predeterminado `main`.
- Las sesiones usan como clave `agent:main:<mainKey>` (el valor predeterminado de `mainKey` es `main`).
- El espacio de trabajo tiene como valor predeterminado `~/.openclaw/workspace` (o `workspace-<profile>` cuando `OPENCLAW_PROFILE` se establece en un valor distinto de `default`).
- El estado tiene como valor predeterminado `~/.openclaw/agents/main/agent`.

## Asistente de agentes

Añada un nuevo agente aislado:

```bash
openclaw agents add work
```

Opciones: `--workspace <dir>`, `--model <id>`, `--agent-dir <dir>`, `--bind <channel[:accountId]>` (repetible), `--non-interactive` (requiere `--workspace`).

Añada `bindings` para enrutar los mensajes entrantes (el asistente ofrece hacerlo), y después verifique:

```bash
openclaw agents list --bindings
```

## Inicio rápido

<Steps>
  <Step title="Crear el espacio de trabajo de cada agente">
    ```bash
    openclaw agents add coding
    openclaw agents add social
    ```

    Cada agente obtiene su propio espacio de trabajo con `SOUL.md`, `AGENTS.md` y el elemento opcional `USER.md`, además de un `agentDir` dedicado y un almacén de sesiones en `~/.openclaw/agents/<agentId>`.

  </Step>
  <Step title="Crear cuentas de canales">
    Cree una cuenta por agente en los canales que prefiera:

    - Discord: un bot por agente; habilite Message Content Intent y copie cada token.
    - Telegram: un bot por agente mediante BotFather; copie cada token.
    - WhatsApp: vincule cada número de teléfono por cuenta.

    ```bash
    openclaw channels login --channel whatsapp --account work
    ```

    Consulte las guías de los canales: [Discord](/es/channels/discord), [Telegram](/es/channels/telegram), [WhatsApp](/es/channels/whatsapp).

  </Step>
  <Step title="Añadir agentes, cuentas y vinculaciones">
    Añada agentes en `agents.entries`, cuentas de canales en `channels.<channel>.accounts` y conéctelos mediante `bindings` (consulte los ejemplos siguientes).
  </Step>
  <Step title="Reiniciar y verificar">
    ```bash
    openclaw gateway restart
    openclaw agents list --bindings
    openclaw channels status --probe
    ```
  </Step>
</Steps>

## Varios agentes, varias personas

Cada `agentId` configurado constituye un límite de persona distinto para el estado principal del agente:

- Cuentas diferentes por canal (por `accountId`).
- Personalidades diferentes (`AGENTS.md`/`SOUL.md` por agente).
- Autenticación y sesiones separadas, con el acceso entre agentes habilitado únicamente mediante funciones explícitas o la configuración de plugins.

Esto permite que varias personas compartan un Gateway mientras mantienen separado el estado principal de cada agente.

## Bóvedas de Memory Wiki por agente

Memory Wiki utiliza una bóveda global de forma predeterminada. Para mantener el
conocimiento compilado de un agente de soporte separado del de un agente de marketing, establezca
`plugins.entries.memory-wiki.config.vault.scope` en `agent`:

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vault: {
            scope: "agent",
            path: "~/.openclaw/wiki",
          },
        },
      },
    },
  },
}
```

La ruta configurada es el directorio superior. OpenClaw añade el identificador
normalizado del agente, lo que genera rutas como `~/.openclaw/wiki/support` y
`~/.openclaw/wiki/marketing`. Las operaciones de la CLI y del Gateway con ámbito de agente requieren
un agente explícito cuando hay varios agentes configurados. Consulte
[Bóvedas de Memory Wiki por agente](/es/plugins/memory-wiki#per-agent-vaults) para obtener información sobre el filtrado
del puente, la migración y los límites de confianza.

## Búsqueda de memoria QMD entre agentes

Para permitir que un agente busque en las transcripciones de sesiones QMD de otro agente, añada colecciones adicionales en `agents.entries.*.memory.search.qmd.extraCollections`. Use `memory.search.qmd.extraCollections` cuando todos los agentes deban compartir las mismas colecciones.

```json5
{
  agents: {
    defaults: {
      workspace: "~/workspaces/main",
    },
    entries: {
      main: {
        workspace: "~/workspaces/main",
        memory: {
          search: {
            qmd: {
              extraCollections: [{ path: "notes" }], // se resuelve dentro del espacio de trabajo -> colección denominada "notes-main"
            },
          },
        },
      },
      family: { workspace: "~/workspaces/family" },
    },
  },
  memory: {
    backend: "qmd",
    search: {
      qmd: {
        extraCollections: [{ path: "~/agents/family/sessions", name: "family-sessions" }],
      },
    },
    qmd: { includeDefaultMemory: false },
  },
}
```

Una ruta de colección adicional puede compartirse entre agentes, pero su `name` permanece explícito cuando la ruta está fuera del espacio de trabajo del agente. Las rutas dentro del espacio de trabajo mantienen el ámbito del agente para que cada uno conserve su propio conjunto de búsqueda de transcripciones.

## Un número de WhatsApp, varias personas (división de mensajes directos)

Enrute distintos mensajes directos de WhatsApp a distintos agentes en **una** cuenta de WhatsApp mediante la coincidencia del remitente E.164 (`+15551234567`) con `peer.kind: "direct"`. Las respuestas siguen procediendo del mismo número de WhatsApp; no existe una identidad de remitente por agente.

<Note>
Los chats directos se agrupan de forma predeterminada en la clave de sesión principal del agente, por lo que el aislamiento real requiere un agente por persona.
</Note>

```json5
{
  agents: {
    list: [
      { id: "alex", workspace: "~/.openclaw/workspace-alex" },
      { id: "mia", workspace: "~/.openclaw/workspace-mia" },
    ],
  },
  bindings: [
    {
      agentId: "alex",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230001" } },
    },
    {
      agentId: "mia",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230002" } },
    },
  ],
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551230001", "+15551230002"],
    },
  },
}
```

El control de acceso a los mensajes directos (emparejamiento/lista permitida) es global para cada cuenta de WhatsApp, no para cada agente. Para grupos compartidos, vincule el grupo a un agente o use [Grupos de difusión](/es/channels/broadcast-groups).

## Reglas de enrutamiento

Las vinculaciones son deterministas y gana la más específica. Consulte [Enrutamiento de canales](/es/channels/channel-routing#routing-rules-how-an-agent-is-chosen) para ver el orden completo de niveles (par exacto, par superior, comodín de par, servidor+roles, servidor, equipo, cuenta, canal, agente predeterminado). Conviene destacar aquí algunas reglas:

- Si varias vinculaciones coinciden dentro del mismo nivel, gana la primera según el orden de la configuración.
- Si una vinculación establece varios campos de coincidencia (por ejemplo, `peer` + `guildId`), todos los campos especificados deben coincidir (semántica de `AND`).
- Una vinculación que omite `accountId` coincide únicamente con la cuenta predeterminada, no con todas las cuentas. Use `accountId: "*"` como alternativa para todo el canal o `accountId: "<name>"` para una cuenta. Añadir de nuevo la misma vinculación con un identificador de cuenta explícito actualiza la vinculación existente exclusiva del canal en lugar de duplicarla.

## Varias cuentas/números de teléfono

Los canales que admiten varias cuentas (por ejemplo, WhatsApp) usan `accountId` para identificar cada inicio de sesión. Cada `accountId` se enruta a su propio agente, por lo que un servidor puede alojar varios números de teléfono sin mezclar las sesiones.

Establezca `channels.<channel>.defaultAccount` para elegir la cuenta utilizada cuando se omite `accountId`. Si no se establece, OpenClaw recurre a `default` si está presente; de lo contrario, utiliza el primer id. de cuenta configurado (ordenado).

Canales que admiten varias cuentas: `discord`, `feishu`, `googlechat`, `imessage`, `irc`, `line`, `mattermost`, `matrix`, `nextcloud-talk`, `nostr`, `signal`, `slack`, `telegram`, `whatsapp`, `zalo`, `zalouser`.

## Conceptos

- `agentId`: un «cerebro» (espacio de trabajo, autenticación por agente y almacén de sesiones por agente).
- `accountId`: una instancia de cuenta de canal (p. ej., la cuenta de WhatsApp `personal` frente a `biz`).
- `binding`: dirige los mensajes entrantes a un `agentId` según `(channel, accountId, peer)` y, opcionalmente, los id. del gremio/equipo.
- Los chats directos se agrupan en `agent:<agentId>:<mainKey>` (la sesión «principal» por agente; consulte `session.mainKey`).

## Ejemplos de plataformas

<AccordionGroup>
  <Accordion title="Bots de Discord por agente">
    Cada cuenta de bot de Discord se asigna a un `accountId` único. Vincule cada cuenta a un agente y mantenga listas de permitidos independientes para cada bot.

    ```json5
    {
      agents: {
        list: [
          { id: "main", workspace: "~/.openclaw/workspace-main" },
          { id: "coding", workspace: "~/.openclaw/workspace-coding" },
        ],
      },
      bindings: [
        { agentId: "main", match: { channel: "discord", accountId: "default" } },
        { agentId: "coding", match: { channel: "discord", accountId: "coding" } },
      ],
      channels: {
        discord: {
          groupPolicy: "allowlist",
          accounts: {
            default: {
              token: "DISCORD_BOT_TOKEN_MAIN",
              guilds: {
                "123456789012345678": {
                  channels: {
                    "222222222222222222": { allow: true, requireMention: false },
                  },
                },
              },
            },
            coding: {
              token: "DISCORD_BOT_TOKEN_CODING",
              guilds: {
                "123456789012345678": {
                  channels: {
                    "333333333333333333": { allow: true, requireMention: false },
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

    - Invite a cada bot al gremio y habilite Message Content Intent.
    - Los tokens se encuentran en `channels.discord.accounts.<id>.token` (la cuenta predeterminada puede usar `DISCORD_BOT_TOKEN`).

  </Accordion>
  <Accordion title="Bots de Telegram por agente">
    ```json5
    {
      agents: {
        list: [
          { id: "main", workspace: "~/.openclaw/workspace-main" },
          { id: "alerts", workspace: "~/.openclaw/workspace-alerts" },
        ],
      },
      bindings: [
        { agentId: "main", match: { channel: "telegram", accountId: "default" } },
        { agentId: "alerts", match: { channel: "telegram", accountId: "alerts" } },
      ],
      channels: {
        telegram: {
          accounts: {
            default: {
              botToken: "123456:ABC...",
              dmPolicy: "pairing",
            },
            alerts: {
              botToken: "987654:XYZ...",
              dmPolicy: "allowlist",
              allowFrom: ["tg:123456789"],
            },
          },
        },
      },
    }
    ```

    - Cree un bot por agente con BotFather y copie cada token.
    - Los tokens se encuentran en `channels.telegram.accounts.<id>.botToken` (la cuenta predeterminada puede usar `TELEGRAM_BOT_TOKEN`).
    - Para usar varios bots en el mismo grupo de Telegram, invite a cada bot y mencione al que deba responder.
    - Deshabilite BotFather Privacy Mode para cada bot de grupo (`/setprivacy` -> Disable) y, a continuación, elimine y vuelva a añadir el bot para que Telegram aplique la configuración.
    - Permita grupos con `channels.telegram.groups` o use `groupPolicy: "open"` únicamente para implementaciones en grupos de confianza.
    - Incluya los id. de usuario de los remitentes en `groupAllowFrom`. Los id. de grupos y supergrupos deben incluirse en `channels.telegram.groups`, no en `groupAllowFrom`.
    - Vincule mediante `accountId` para que cada bot dirija los mensajes a su propio agente.

  </Accordion>
  <Accordion title="Números de WhatsApp por agente">
    Vincule cada cuenta antes de iniciar el Gateway:

    ```bash
    openclaw channels login --channel whatsapp --account personal
    openclaw channels login --channel whatsapp --account biz
    ```

    `~/.openclaw/openclaw.json` (JSON5):

    ```js
    {
      agents: {
        list: [
          {
            id: "home",
            default: true,
            name: "Home",
            workspace: "~/.openclaw/workspace-home",
            agentDir: "~/.openclaw/agents/home/agent",
          },
          {
            id: "work",
            name: "Work",
            workspace: "~/.openclaw/workspace-work",
            agentDir: "~/.openclaw/agents/work/agent",
          },
        ],
      },

      // Enrutamiento determinista: prevalece la primera coincidencia (primero la más específica).
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },

        // Reemplazo opcional por interlocutor (ejemplo: enviar un grupo específico al agente de trabajo).
        {
          agentId: "work",
          match: {
            channel: "whatsapp",
            accountId: "personal",
            peer: { kind: "group", id: "1203630...@g.us" },
          },
        },
      ],

      // Desactivado de forma predeterminada: la mensajería entre agentes debe habilitarse explícitamente y añadirse a la lista de permitidos.
      tools: {
        agentToAgent: {
          enabled: false,
          allow: ["home", "work"],
        },
      },

      channels: {
        whatsapp: {
          accounts: {
            personal: {
              // Reemplazo opcional. Valor predeterminado: ~/.openclaw/credentials/whatsapp/personal
              // authDir: "~/.openclaw/credentials/whatsapp/personal",
            },
            biz: {
              // Reemplazo opcional. Valor predeterminado: ~/.openclaw/credentials/whatsapp/biz
              // authDir: "~/.openclaw/credentials/whatsapp/biz",
            },
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## Patrones habituales

<Tabs>
  <Tab title="WhatsApp para el día a día y Telegram para trabajo en profundidad">
    Separe por canal: dirija WhatsApp a un agente rápido para el uso cotidiano y Telegram a un agente Opus.

    ```json5
    {
      agents: {
        list: [
          {
            id: "chat",
            name: "Everyday",
            workspace: "~/.openclaw/workspace-chat",
            model: "anthropic/claude-sonnet-4-6",
          },
          {
            id: "opus",
            name: "Deep Work",
            workspace: "~/.openclaw/workspace-opus",
            model: "anthropic/claude-opus-4-6",
          },
        ],
      },
      bindings: [
        { agentId: "chat", match: { channel: "whatsapp", accountId: "*" } },
        { agentId: "opus", match: { channel: "telegram", accountId: "*" } },
      ],
    }
    ```

    Estos ejemplos usan `accountId: "*"` para que las vinculaciones sigan funcionando si se añaden cuentas más adelante. Para dirigir un único mensaje directo/grupo a Opus y mantener el resto en el chat, añada una vinculación `match.peer` para ese interlocutor; las coincidencias de interlocutores siempre prevalecen sobre las reglas de todo el canal.

  </Tab>
  <Tab title="Mismo canal, un interlocutor dirigido a Opus">
    Mantenga WhatsApp en el agente rápido, pero dirija un mensaje directo a Opus:

    ```json5
    {
      agents: {
        list: [
          {
            id: "chat",
            name: "Everyday",
            workspace: "~/.openclaw/workspace-chat",
            model: "anthropic/claude-sonnet-4-6",
          },
          {
            id: "opus",
            name: "Deep Work",
            workspace: "~/.openclaw/workspace-opus",
            model: "anthropic/claude-opus-4-6",
          },
        ],
      },
      bindings: [
        {
          agentId: "opus",
          match: { channel: "whatsapp", accountId: "*", peer: { kind: "direct", id: "+15551234567" } },
        },
        { agentId: "chat", match: { channel: "whatsapp", accountId: "*" } },
      ],
    }
    ```

    Las vinculaciones de interlocutores siempre prevalecen, por lo que deben mantenerse por encima de la regla de todo el canal.

  </Tab>
  <Tab title="Agente familiar vinculado a un grupo de WhatsApp">
    Vincule un agente familiar dedicado a un único grupo de WhatsApp, con requisito de mención y una política de herramientas más restrictiva:

    ```json5
    {
      agents: {
        list: [
          {
            id: "family",
            name: "Family",
            workspace: "~/.openclaw/workspace-family",
            identity: { name: "Family Bot" },
            groupChat: {
              mentionPatterns: ["@family", "@familybot", "@Family Bot"],
            },
            sandbox: {
              mode: "all",
              scope: "agent",
            },
            tools: {
              allow: [
                "exec",
                "read",
                "sessions_list",
                "sessions_history",
                "sessions_send",
                "sessions_spawn",
                "session_status",
              ],
              deny: ["write", "edit", "apply_patch", "browser", "canvas", "nodes", "cron"],
            },
          },
        ],
      },
      bindings: [
        {
          agentId: "family",
          match: {
            channel: "whatsapp",
            peer: { kind: "group", id: "120363999999999999@g.us" },
          },
        },
      ],
    }
    ```

    Las listas de herramientas permitidas/denegadas son **herramientas**, no Skills. Si una habilidad necesita ejecutar un binario, asegúrese de que `exec` esté permitido y de que el binario exista en el entorno aislado. Para aplicar restricciones más estrictas, establezca `agents.entries.*.groupChat.mentionPatterns` y mantenga habilitadas las listas de grupos permitidos para el canal.

  </Tab>
</Tabs>

## Configuración del entorno aislado y de las herramientas por agente

Cada agente puede tener sus propias restricciones de entorno aislado y herramientas:

```js
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: {
          mode: "off",  // Sin entorno aislado para el agente personal
        },
        // Sin restricciones de herramientas: todas las herramientas están disponibles
      },
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",     // Siempre en un entorno aislado
          scope: "agent",  // Un contenedor por agente
          docker: {
            // Configuración inicial opcional tras crear el contenedor
            setupCommand: "apt-get update && apt-get install -y git curl",
          },
        },
        tools: {
          allow: ["read"],                    // Solo la herramienta de lectura
          deny: ["exec", "write", "edit", "apply_patch"],    // Denegar las demás
        },
      },
    ],
  },
}
```

<Note>
`setupCommand` se encuentra en `sandbox.docker` y se ejecuta una vez al crear el contenedor. Los reemplazos `sandbox.docker.*` por agente se ignoran cuando el ámbito resuelto es `"shared"`.
</Note>

Esto proporciona:

- **Aislamiento de seguridad**: restrinja las herramientas para agentes que no sean de confianza.
- **Control de recursos**: aísle agentes específicos mientras mantiene los demás en el host.
- **Políticas flexibles**: distintos permisos para cada agente.

<Note>
`tools.elevated` tiene una barrera global (`tools.elevated.enabled`/`allowFrom`) y otra por agente (`agents.entries.*.tools.elevated.enabled`/`allowFrom`). La barrera por agente solo puede restringir aún más la global: ambas deben permitir al remitente para que puedan ejecutarse comandos elevados. Para dirigirse a grupos, use `agents.entries.*.groupChat.mentionPatterns` para que las @menciones se asignen correctamente al agente previsto.
</Note>

Consulte [Entorno aislado y herramientas para varios agentes](/es/tools/multi-agent-sandbox-tools) para ver ejemplos detallados.

## Contenido relacionado

- [Agentes ACP](/es/tools/acp-agents) — ejecución de entornos externos de programación
- [Enrutamiento de canales](/es/channels/channel-routing) — cómo se enrutan los mensajes a los agentes
- [Presencia](/es/concepts/presence) — presencia y disponibilidad de los agentes
- [Sesión](/es/concepts/session) — aislamiento y enrutamiento de sesiones
- [Subagentes](/es/tools/subagents) — inicio de ejecuciones de agentes en segundo plano
