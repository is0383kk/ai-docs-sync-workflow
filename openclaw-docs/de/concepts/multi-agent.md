---
read_when: You want multiple agents with separate workspaces, auth, and sessions in one Gateway process.
sidebarTitle: Multi-agent routing
status: active
summary: 'Multi-Agent-Routing: Agent-Grenzen, Kanal-Konten und Bindungen'
title: Multi-Agenten-Routing
x-i18n:
    generated_at: "2026-07-26T18:24:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 46df162388205e46d5a4ea3567c8c8f7016117d2ecafe1184a35b4c95798fd80
    source_path: concepts/multi-agent.md
    workflow: 16
---

Führen Sie mehrere _isolierte_ Agenten in einem Gateway-Prozess aus, jeweils mit eigenem Workspace, Zustandsverzeichnis (`agentDir`) und SQLite-gestütztem Sitzungsverlauf sowie mehreren Kanalkonten (z. B. zwei WhatsApp-Nummern). Eingehende Nachrichten werden über **Bindungen** an den richtigen Agenten weitergeleitet.

Ein **Agent** umfasst den vollständigen Bereich einer Persona: Workspace-Dateien, Authentifizierungsprofile, Modellregistrierung und Sitzungsspeicher. Eine **Bindung** ordnet ein Kanalkonto (einen Slack-Workspace, eine WhatsApp-Nummer usw.) einem dieser Agenten zu.

## Was ist ein Agent?

Jeder Agent verfügt über einen eigenen:

- **Workspace**: Dateien, `AGENTS.md`/`SOUL.md`/`USER.md`, lokale Notizen, Persona-Regeln.
- **Zustandsverzeichnis** (`agentDir`): Authentifizierungsprofile, Modellregistrierung, agentenspezifische Konfiguration.
- **Sitzungsspeicher**: Chatverlauf und Routing-Zustand in `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`.

Authentifizierungsprofile gelten jeweils pro Agent und werden aus folgendem Pfad gelesen:

```text
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

<Note>
`sessions_history` ist der sicherere Weg zum sitzungsübergreifenden Abruf: Es gibt eine begrenzte, geschwärzte Ansicht zurück, keinen Rohdump des Transkripts. Signaturen von Denkblöcken, Details der Nutzdaten von Werkzeugergebnissen, `<relevant-memories>`-Gerüstcode, XML-Tags für Werkzeugaufrufe (`<tool_call>`, `<function_call>` sowie deren Plural- und herabgestufte Formen) und MiniMax-XML für Werkzeugaufrufe werden entfernt; anschließend wird die Ausgabe gekürzt und anhand der Byte-Größe begrenzt.
</Note>

<Warning>
Verwenden Sie `agentDir` niemals agentenübergreifend erneut — dies verursacht Kollisionen beim Authentifizierungs- und Sitzungszustand. Wenn die lokale OAuth-Anmeldeinformation eines sekundären Agenten abgelaufen ist oder ihre Aktualisierung fehlschlägt, greift OpenClaw auf die Anmeldeinformation des standardmäßigen bzw. Hauptagenten mit derselben Profil-ID zurück und übernimmt das jeweils aktuellste Token, ohne das Aktualisierungstoken in den Speicher des sekundären Agenten zu kopieren. Wenn Sie ein vollständig unabhängiges OAuth-Konto benötigen, melden Sie sich über diesen Agenten an. Wenn Sie Anmeldeinformationen manuell kopieren, kopieren Sie nur übertragbare statische `api_key`- oder `token`-Profile — OAuth-Aktualisierungsmaterial ist standardmäßig nicht übertragbar (`copyToAgents` kann dies für ein Profil ausdrücklich aktivieren).
</Warning>

Skills werden aus dem Workspace jedes Agenten sowie aus gemeinsam genutzten Stammverzeichnissen wie `~/.openclaw/skills` geladen und anschließend anhand der effektiven Skill-Zulassungsliste des Agenten gefiltert. Verwenden Sie `agents.defaults.skills` für eine gemeinsame Basis und `agents.entries.*.skills` für einen agentenspezifischen Ersatz (explizite Einträge ersetzen den Standard, sie werden nicht zusammengeführt). Siehe [Skills: agentenspezifisch und gemeinsam genutzt](/de/tools/skills#per-agent-vs-shared-skills) und [Skills: Agentenzulassungslisten](/de/tools/skills#agent-allowlists).

Der Plugin-eigene Speicher richtet sich nach der Konfiguration dieses Plugins; durch das Hinzufügen eines zweiten Agenten wird nicht automatisch jeder globale Plugin-Speicher aufgeteilt. Konfigurieren Sie beispielsweise [agentenspezifische Memory-Wiki-Vaults](/de/concepts/multi-agent#per-agent-memory-wiki-vaults), wenn Personas kein kompiliertes Wiki-Wissen gemeinsam nutzen dürfen.

<Note>
**Hinweis zum Workspace:** Der Workspace jedes Agenten ist das **standardmäßige cwd**, keine feste Sandbox. Relative Pfade werden innerhalb des Workspace aufgelöst, absolute Pfade können jedoch auf andere Speicherorte des Hosts zugreifen, sofern Sandboxing nicht aktiviert ist. Siehe [Sandboxing](/de/gateway/sandboxing).
</Note>

## Pfade

| Element                          | Standard                                                                               | Überschreibung                                                                               |
| -------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Konfiguration                    | `~/.openclaw/openclaw.json`                                                            | `OPENCLAW_CONFIG_PATH`                                                                      |
| Zustandsverzeichnis              | `~/.openclaw`                                                                          | `OPENCLAW_STATE_DIR`                                                                        |
| Workspace des Standardagenten    | `~/.openclaw/workspace` (oder `workspace-<profile>`, wenn `OPENCLAW_PROFILE` gesetzt ist)      | `agents.entries.*.workspace`, dann `agents.defaults.workspace` oder `OPENCLAW_WORKSPACE_DIR` |
| Workspace anderer Agenten        | `<stateDir>/workspace-<agentId>` (oder `<agents.defaults.workspace>/<agentId>`, wenn gesetzt) | `agents.entries.*.workspace`                                                                |
| Agentenverzeichnis               | `~/.openclaw/agents/<agentId>/agent`                                                   | `agents.entries.*.agentDir`                                                                 |
| Sitzungen und Transkripte        | `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`                             | —                                                                                           |
| Veraltete/archivierte Sitzungsartefakte | `~/.openclaw/agents/<agentId>/sessions`                                                | —                                                                                           |

### Einzelagentenmodus (Standard)

Wenn Sie nichts konfigurieren, führt OpenClaw einen Agenten aus:

- `agentId` verwendet standardmäßig `main`.
- Sitzungsschlüssel haben das Format `agent:main:<mainKey>` (der Standardwert `mainKey` ist `main`).
- Der Workspace verwendet standardmäßig `~/.openclaw/workspace` (oder `workspace-<profile>`, wenn `OPENCLAW_PROFILE` auf einen anderen Wert als `default` gesetzt ist).
- Der Zustand verwendet standardmäßig `~/.openclaw/agents/main/agent`.

## Agenten-Hilfsfunktion

Fügen Sie einen neuen isolierten Agenten hinzu:

```bash
openclaw agents add work
```

Flags: `--workspace <dir>`, `--model <id>`, `--agent-dir <dir>`, `--bind <channel[:accountId]>` (wiederholbar), `--non-interactive` (erfordert `--workspace`).

Fügen Sie `bindings` hinzu, um eingehende Nachrichten weiterzuleiten (der Assistent bietet an, dies für Sie zu erledigen), und überprüfen Sie anschließend die Konfiguration:

```bash
openclaw agents list --bindings
```

## Schnellstart

<Steps>
  <Step title="Workspace für jeden Agenten erstellen">
    ```bash
    openclaw agents add coding
    openclaw agents add social
    ```

    Jeder Agent erhält einen eigenen Workspace mit `SOUL.md`, `AGENTS.md` und optional `USER.md` sowie ein dediziertes `agentDir` und einen Sitzungsspeicher unter `~/.openclaw/agents/<agentId>`.

  </Step>
  <Step title="Kanalkonten erstellen">
    Erstellen Sie auf Ihren bevorzugten Kanälen jeweils ein Konto pro Agent:

    - Discord: ein Bot pro Agent; aktivieren Sie Message Content Intent und kopieren Sie jedes Token.
    - Telegram: ein Bot pro Agent über BotFather; kopieren Sie jedes Token.
    - WhatsApp: Verknüpfen Sie für jedes Konto die jeweilige Telefonnummer.

    ```bash
    openclaw channels login --channel whatsapp --account work
    ```

    Siehe Kanalanleitungen: [Discord](/de/channels/discord), [Telegram](/de/channels/telegram), [WhatsApp](/de/channels/whatsapp).

  </Step>
  <Step title="Agenten, Konten und Bindungen hinzufügen">
    Fügen Sie Agenten unter `agents.entries` und Kanalkonten unter `channels.<channel>.accounts` hinzu und verbinden Sie sie mit `bindings` (Beispiele weiter unten).
  </Step>
  <Step title="Neu starten und überprüfen">
    ```bash
    openclaw gateway restart
    openclaw agents list --bindings
    openclaw channels status --probe
    ```
  </Step>
</Steps>

## Mehrere Agenten, mehrere Personas

Jeder konfigurierte `agentId` bildet eine eigenständige Persona-Grenze für den zentralen Agentenzustand:

- Unterschiedliche Konten pro Kanal (je `accountId`).
- Unterschiedliche Persönlichkeiten (agentenspezifische `AGENTS.md`/`SOUL.md`).
- Getrennte Authentifizierung und Sitzungen; agentenübergreifender Zugriff wird nur durch explizite Funktionen oder die Plugin-Konfiguration aktiviert.

Dadurch können sich mehrere Personen ein Gateway teilen, während der zentrale Agentenzustand getrennt bleibt.

## Agentenspezifische Memory-Wiki-Vaults

Memory Wiki verwendet standardmäßig ein globales Vault. Um das kompilierte Wissen eines Support-Agenten vom Wissen eines Marketing-Agenten zu trennen, setzen Sie `plugins.entries.memory-wiki.config.vault.scope` auf `agent`:

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

Der konfigurierte Pfad ist das übergeordnete Verzeichnis. OpenClaw hängt die normalisierte Agenten-ID an und erzeugt Pfade wie `~/.openclaw/wiki/support` und `~/.openclaw/wiki/marketing`. Agentenspezifische CLI- und Gateway-Vorgänge erfordern einen expliziten Agenten, wenn mehrere Agenten konfiguriert sind. Weitere Informationen zur Bridge-Filterung, Migration und zu Vertrauensgrenzen finden Sie unter [Agentenspezifische Memory-Wiki-Vaults](/de/plugins/memory-wiki#per-agent-vaults).

## Agentenübergreifende QMD-Speichersuche

Damit ein Agent die QMD-Sitzungstranskripte eines anderen Agenten durchsuchen kann, fügen Sie unter `agents.entries.*.memory.search.qmd.extraCollections` zusätzliche Sammlungen hinzu. Verwenden Sie `memory.search.qmd.extraCollections`, wenn alle Agenten dieselben Sammlungen gemeinsam nutzen sollen.

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
              extraCollections: [{ path: "notes" }], // wird im Workspace aufgelöst -> Sammlung mit dem Namen "notes-main"
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

Der Pfad einer zusätzlichen Sammlung kann von mehreren Agenten gemeinsam genutzt werden, sein `name` bleibt jedoch explizit, wenn der Pfad außerhalb des Agenten-Workspace liegt. Pfade innerhalb des Workspace bleiben agentenspezifisch, sodass jeder Agent seinen eigenen Satz durchsuchbarer Transkripte behält.

## Eine WhatsApp-Nummer, mehrere Personen (DM-Aufteilung)

Leiten Sie verschiedene WhatsApp-DMs in **einem** WhatsApp-Konto an unterschiedliche Agenten weiter, indem Sie die E.164-Nummer des Absenders (`+15551234567`) mit `peer.kind: "direct"` abgleichen. Antworten werden weiterhin von derselben WhatsApp-Nummer gesendet — es gibt keine agentenspezifische Absenderidentität.

<Note>
Direktchats werden standardmäßig im Hauptsitzungsschlüssel des Agenten zusammengeführt; eine echte Isolation erfordert daher einen Agenten pro Person.
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

Die DM-Zugriffssteuerung (Kopplung/Zulassungsliste) gilt global pro WhatsApp-Konto, nicht pro Agent. Binden Sie gemeinsam genutzte Gruppen an einen Agenten oder verwenden Sie [Broadcast-Gruppen](/de/channels/broadcast-groups).

## Routing-Regeln

Bindungen sind deterministisch; die spezifischste Übereinstimmung gewinnt. Die vollständige Rangfolge (exakter Kommunikationspartner, übergeordneter Kommunikationspartner, Platzhalter für Kommunikationspartner, Guild und Rollen, Guild, Team, Konto, Kanal, Standardagent) finden Sie unter [Kanal-Routing](/de/channels/channel-routing#routing-rules-how-an-agent-is-chosen). Einige Regeln sind hier besonders hervorzuheben:

- Wenn mehrere Bindungen innerhalb derselben Rangstufe übereinstimmen, gewinnt die erste in der Konfigurationsreihenfolge.
- Wenn eine Bindung mehrere Abgleichsfelder festlegt (beispielsweise `peer` + `guildId`), müssen alle angegebenen Felder übereinstimmen (`AND`-Semantik).
- Eine Bindung ohne `accountId` stimmt nur mit dem Standardkonto überein, nicht mit jedem Konto. Verwenden Sie `accountId: "*"` für einen kanalweiten Fallback oder `accountId: "<name>"` für ein einzelnes Konto. Wenn dieselbe Bindung erneut mit einer expliziten Konto-ID hinzugefügt wird, wird die bestehende reine Kanalbindung aktualisiert, statt sie zu duplizieren.

## Mehrere Konten/Telefonnummern

Kanäle, die mehrere Konten unterstützen (z. B. WhatsApp), verwenden `accountId`, um jede Anmeldung zu identifizieren. Jedes `accountId` wird an einen eigenen Agenten weitergeleitet, sodass ein Server mehrere Telefonnummern hosten kann, ohne Sitzungen zu vermischen.

Setzen Sie `channels.<channel>.defaultAccount`, um das Konto auszuwählen, das verwendet wird, wenn `accountId` weggelassen wird. Ist dies nicht festgelegt, greift OpenClaw auf `default` zurück, sofern vorhanden, andernfalls auf die erste konfigurierte Konto-ID (sortiert).

Kanäle mit Unterstützung für mehrere Konten: `discord`, `feishu`, `googlechat`, `imessage`, `irc`, `line`, `mattermost`, `matrix`, `nextcloud-talk`, `nostr`, `signal`, `slack`, `telegram`, `whatsapp`, `zalo`, `zalouser`.

## Konzepte

- `agentId`: ein „Gehirn“ (Arbeitsbereich, agentenspezifische Authentifizierung, agentenspezifischer Sitzungsspeicher).
- `accountId`: eine Instanz eines Kanalkontos (z. B. WhatsApp-Konto `personal` gegenüber `biz`).
- `binding`: leitet eingehende Nachrichten anhand von `(channel, accountId, peer)` und optional anhand von Guild-/Team-IDs an einen `agentId` weiter.
- Direktchats werden in `agent:<agentId>:<mainKey>` zusammengeführt (agentenspezifisch „main“; siehe `session.mainKey`).

## Plattformbeispiele

<AccordionGroup>
  <Accordion title="Discord-Bots pro Agent">
    Jedes Discord-Bot-Konto wird einer eindeutigen `accountId` zugeordnet. Binden Sie jedes Konto an einen Agenten und verwalten Sie die Positivlisten pro Bot.

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

    - Laden Sie jeden Bot in die Guild ein und aktivieren Sie Message Content Intent.
    - Die Tokens befinden sich in `channels.discord.accounts.<id>.token` (das Standardkonto kann `DISCORD_BOT_TOKEN` verwenden).

  </Accordion>
  <Accordion title="Telegram-Bots pro Agent">
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

    - Erstellen Sie mit BotFather einen Bot pro Agent und kopieren Sie jedes Token.
    - Die Tokens befinden sich in `channels.telegram.accounts.<id>.botToken` (das Standardkonto kann `TELEGRAM_BOT_TOKEN` verwenden).
    - Laden Sie bei mehreren Bots in derselben Telegram-Gruppe jeden Bot ein und erwähnen Sie denjenigen, der antworten soll.
    - Deaktivieren Sie für jeden Gruppen-Bot den BotFather Privacy Mode (`/setprivacy` -> Disable). Entfernen Sie den Bot anschließend und fügen Sie ihn erneut hinzu, damit Telegram die Einstellung übernimmt.
    - Erlauben Sie Gruppen mit `channels.telegram.groups`, oder verwenden Sie `groupPolicy: "open"` nur für vertrauenswürdige Gruppenbereitstellungen.
    - Tragen Sie Benutzer-IDs von Absendern in `groupAllowFrom` ein. Gruppen- und Supergruppen-IDs gehören in `channels.telegram.groups`, nicht in `groupAllowFrom`.
    - Binden Sie anhand von `accountId`, damit jeder Bot Nachrichten an seinen eigenen Agenten weiterleitet.

  </Accordion>
  <Accordion title="WhatsApp-Nummern pro Agent">
    Verknüpfen Sie jedes Konto, bevor Sie den Gateway starten:

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

      // Deterministisches Routing: Die erste Übereinstimmung gewinnt (die spezifischste zuerst).
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },

        // Optionale Override-Regel pro Peer (Beispiel: Eine bestimmte Gruppe an den Arbeitsagenten senden).
        {
          agentId: "work",
          match: {
            channel: "whatsapp",
            accountId: "personal",
            peer: { kind: "group", id: "1203630...@g.us" },
          },
        },
      ],

      // Standardmäßig deaktiviert: Die Kommunikation zwischen Agenten muss ausdrücklich aktiviert und in die Positivliste aufgenommen werden.
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
              // Optionale Override-Regel. Standard: ~/.openclaw/credentials/whatsapp/personal
              // authDir: "~/.openclaw/credentials/whatsapp/personal",
            },
            biz: {
              // Optionale Override-Regel. Standard: ~/.openclaw/credentials/whatsapp/biz
              // authDir: "~/.openclaw/credentials/whatsapp/biz",
            },
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## Gängige Muster

<Tabs>
  <Tab title="WhatsApp für den Alltag und Telegram für intensive Arbeit">
    Nach Kanal aufteilen: Leiten Sie WhatsApp an einen schnellen Agenten für den Alltag und Telegram an einen Opus-Agenten weiter.

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

    Diese Beispiele verwenden `accountId: "*"`, damit die Bindungen weiterhin funktionieren, wenn Sie später Konten hinzufügen. Um einen einzelnen Direktchat oder eine einzelne Gruppe an Opus weiterzuleiten und den Rest beim Chat-Agenten zu belassen, fügen Sie für diesen Peer eine `match.peer`-Bindung hinzu — Peer-Übereinstimmungen haben stets Vorrang vor kanalweiten Regeln.

  </Tab>
  <Tab title="Derselbe Kanal, ein Peer an Opus">
    Belassen Sie WhatsApp beim schnellen Agenten, leiten Sie jedoch einen Direktchat an Opus weiter:

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

    Peer-Bindungen haben stets Vorrang. Platzieren Sie sie daher oberhalb der kanalweiten Regel.

  </Tab>
  <Tab title="An eine WhatsApp-Gruppe gebundener Familienagent">
    Binden Sie einen dedizierten Familienagenten an eine einzelne WhatsApp-Gruppe, mit Erwähnungsprüfung und strengeren Tool-Richtlinien:

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

    Tool-Positiv- und -Negativlisten beziehen sich auf **Tools**, nicht auf Skills. Wenn ein Skill eine Binärdatei ausführen muss, stellen Sie sicher, dass `exec` erlaubt ist und die Binärdatei in der Sandbox vorhanden ist. Legen Sie für eine strengere Prüfung `agents.entries.*.groupChat.mentionPatterns` fest und lassen Sie die Gruppen-Positivlisten für den Kanal aktiviert.

  </Tab>
</Tabs>

## Agentenspezifische Sandbox- und Tool-Konfiguration

Jeder Agent kann eigene Sandbox- und Tool-Einschränkungen haben:

```js
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: {
          mode: "off",  // Keine Sandbox für den persönlichen Agenten
        },
        // Keine Tool-Einschränkungen – alle Tools sind verfügbar
      },
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",     // Immer in einer Sandbox
          scope: "agent",  // Ein Container pro Agent
          docker: {
            // Optionale einmalige Einrichtung nach der Container-Erstellung
            setupCommand: "apt-get update && apt-get install -y git curl",
          },
        },
        tools: {
          allow: ["read"],                    // Nur das Lese-Tool
          deny: ["exec", "write", "edit", "apply_patch"],    // Andere verweigern
        },
      },
    ],
  },
}
```

<Note>
`setupCommand` befindet sich unter `sandbox.docker` und wird bei der Container-Erstellung einmal ausgeführt. Agentenspezifische `sandbox.docker.*`-Overrides werden ignoriert, wenn der aufgelöste Geltungsbereich `"shared"` ist.
</Note>

Dies bietet Ihnen:

- **Sicherheitsisolierung**: Tools für nicht vertrauenswürdige Agenten einschränken.
- **Ressourcenkontrolle**: Bestimmte Agenten in einer Sandbox ausführen, während andere auf dem Host verbleiben.
- **Flexible Richtlinien**: unterschiedliche Berechtigungen pro Agent.

<Note>
`tools.elevated` verfügt sowohl über eine globale Zugriffsprüfung (`tools.elevated.enabled`/`allowFrom`) als auch über eine agentenspezifische Zugriffsprüfung (`agents.entries.*.tools.elevated.enabled`/`allowFrom`). Die agentenspezifische Zugriffsprüfung kann die globale lediglich weiter einschränken — beide müssen einen Absender zulassen, damit Befehle mit erhöhten Berechtigungen ausgeführt werden. Verwenden Sie für die Gruppenzuordnung `agents.entries.*.groupChat.mentionPatterns`, damit @Erwähnungen eindeutig dem vorgesehenen Agenten zugeordnet werden.
</Note>

Ausführliche Beispiele finden Sie unter [Sandbox und Tools für mehrere Agenten](/de/tools/multi-agent-sandbox-tools).

## Verwandte Themen

- [ACP-Agenten](/de/tools/acp-agents) — Ausführen externer Coding-Harnesses
- [Kanal-Routing](/de/channels/channel-routing) — wie Nachrichten an Agenten weitergeleitet werden
- [Präsenz](/de/concepts/presence) — Präsenz und Verfügbarkeit von Agenten
- [Sitzung](/de/concepts/session) — Sitzungsisolierung und Routing
- [Unteragenten](/de/tools/subagents) — Starten von Agentenläufen im Hintergrund
