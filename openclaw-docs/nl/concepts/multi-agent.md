---
read_when: You want multiple agents with separate workspaces, auth, and sessions in one Gateway process.
sidebarTitle: Multi-agent routing
status: active
summary: 'Multi-agentroutering: agentgrenzen, kanaalaccounts en koppelingen'
title: Routing voor meerdere agents
x-i18n:
    generated_at: "2026-07-27T05:43:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 46df162388205e46d5a4ea3567c8c8f7016117d2ecafe1184a35b4c95798fd80
    source_path: concepts/multi-agent.md
    workflow: 16
---

Voer meerdere _geïsoleerde_ agents uit in één Gateway-proces, elk met een eigen werkruimte, statusmap (`agentDir`) en op SQLite gebaseerde sessiegeschiedenis, plus meerdere kanaalaccounts (bijvoorbeeld twee WhatsApp-nummers). Inkomende berichten worden via **bindingen** naar de juiste agent gerouteerd.

Een **agent** omvat alles wat bij één persona hoort: werkruimtebestanden, authenticatieprofielen, modelregister en sessieopslag. Een **binding** koppelt een kanaalaccount (een Slack-werkruimte, een WhatsApp-nummer enzovoort) aan een van die agents.

## Wat is één agent

Elke agent heeft een eigen:

- **Werkruimte**: bestanden, `AGENTS.md`/`SOUL.md`/`USER.md`, lokale notities, personaregels.
- **Statusmap** (`agentDir`): authenticatieprofielen, modelregister, configuratie per agent.
- **Sessieopslag**: chatgeschiedenis en routeringsstatus in `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`.

Authenticatieprofielen zijn per agent en worden gelezen uit:

```text
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

<Note>
`sessions_history` is het veiligere pad om informatie tussen sessies op te halen: het retourneert een begrensde, geredigeerde weergave en geen onbewerkte transcriptdump. Het verwijdert handtekeningen van denkblokken, details van toolresultaatpayloads, `<relevant-memories>`-scaffolding, XML-tags voor toolaanroepen (`<tool_call>`, `<function_call>` en hun meervoudige/gedowngradede vormen) en XML voor MiniMax-toolaanroepen. Vervolgens wordt de uitvoer afgekapt en op bytegrootte begrensd.
</Note>

<Warning>
Gebruik `agentDir` nooit opnieuw voor meerdere agents — dit veroorzaakt botsingen tussen authenticatie- en sessiestatussen. Wanneer de lokale OAuth-referentie van een secundaire agent is verlopen of het vernieuwen ervan mislukt, leest OpenClaw de referentie van de standaard-/hoofdagent voor dezelfde profiel-id en neemt het het meest recente token over, zonder het vernieuwingstoken naar de opslag van de secundaire agent te kopiëren. Als je een volledig onafhankelijk OAuth-account wilt, meld je dan vanuit die agent aan. Als je referenties handmatig kopieert, kopieer dan alleen overdraagbare statische `api_key`- of `token`-profielen — OAuth-vernieuwingsmateriaal is standaard niet overdraagbaar (`copyToAgents` kan dit expliciet voor een profiel inschakelen).
</Warning>

Skills worden geladen uit de werkruimte van elke agent en uit gedeelde hoofdmappen zoals `~/.openclaw/skills`, en vervolgens gefilterd op basis van de effectieve lijst met toegestane Skills van de agent. Gebruik `agents.defaults.skills` voor een gedeelde basis en `agents.entries.*.skills` voor een vervanging per agent (expliciete vermeldingen vervangen de standaard en worden er niet mee samengevoegd). Zie [Skills: per agent versus gedeeld](/nl/tools/skills#per-agent-vs-shared-skills) en [Skills: lijsten met toegestane agents](/nl/tools/skills#agent-allowlists).

Opslag die eigendom is van een Plugin volgt de configuratie van die Plugin; door een tweede agent toe te voegen, wordt niet automatisch elke algemene Plugin-opslag opgesplitst. Configureer bijvoorbeeld [Memory Wiki-kluizen per agent](/nl/concepts/multi-agent#per-agent-memory-wiki-vaults) wanneer persona's geen gecompileerde wikikennis mogen delen.

<Note>
**Opmerking over de werkruimte:** de werkruimte van elke agent is de **standaard-cwd**, geen harde sandbox. Relatieve paden worden binnen de werkruimte omgezet, maar absolute paden kunnen andere locaties op de host bereiken, tenzij sandboxing is ingeschakeld. Zie [Sandboxing](/nl/gateway/sandboxing).
</Note>

## Paden

| Wat                              | Standaard                                                                               | Overschrijving                                                                               |
| -------------------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Configuratie                     | `~/.openclaw/openclaw.json`                                                                      | `OPENCLAW_CONFIG_PATH`                                                                           |
| Statusmap                        | `~/.openclaw`                                                                      | `OPENCLAW_STATE_DIR`                                                                           |
| Werkruimte van standaardagent    | `~/.openclaw/workspace` (of `workspace-<profile>` wanneer `OPENCLAW_PROFILE` is ingesteld)      | `agents.entries.*.workspace`, daarna `agents.defaults.workspace`, of `OPENCLAW_WORKSPACE_DIR`                          |
| Werkruimte van overige agents    | `<stateDir>/workspace-<agentId>` (of `<agents.defaults.workspace>/<agentId>` wanneer ingesteld)                            | `agents.entries.*.workspace`                                                                           |
| Agentmap                         | `~/.openclaw/agents/<agentId>/agent`                                                                      | `agents.entries.*.agentDir`                                                                           |
| Sessies en transcripten          | `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`                                                                      | —                                                                                            |
| Verouderde/gearchiveerde sessieartefacten | `~/.openclaw/agents/<agentId>/sessions`                                                             | —                                                                                            |

### Modus met één agent (standaard)

Als je niets configureert, voert OpenClaw één agent uit:

- `agentId` is standaard `main`.
- Sessies gebruiken `agent:main:<mainKey>` als sleutel (standaard is `mainKey` gelijk aan `main`).
- De werkruimte is standaard `~/.openclaw/workspace` (of `workspace-<profile>` wanneer `OPENCLAW_PROFILE` is ingesteld op iets anders dan `default`).
- De status is standaard `~/.openclaw/agents/main/agent`.

## Agenthelper

Voeg een nieuwe geïsoleerde agent toe:

```bash
openclaw agents add work
```

Vlaggen: `--workspace <dir>`, `--model <id>`, `--agent-dir <dir>`, `--bind <channel[:accountId]>` (herhaalbaar), `--non-interactive` (vereist `--workspace`).

Voeg `bindings` toe om inkomende berichten te routeren (de wizard biedt aan dit voor je te doen) en controleer vervolgens:

```bash
openclaw agents list --bindings
```

## Snel aan de slag

<Steps>
  <Step title="Maak de werkruimte van elke agent">
    ```bash
    openclaw agents add coding
    openclaw agents add social
    ```

    Elke agent krijgt een eigen werkruimte met `SOUL.md`, `AGENTS.md` en optioneel `USER.md`, plus een eigen `agentDir` en sessieopslag onder `~/.openclaw/agents/<agentId>`.

  </Step>
  <Step title="Maak kanaalaccounts">
    Maak voor elke agent één account aan op de kanalen van je voorkeur:

    - Discord: één bot per agent, schakel Message Content Intent in en kopieer elk token.
    - Telegram: één bot per agent via BotFather; kopieer elk token.
    - WhatsApp: koppel elk telefoonnummer per account.

    ```bash
    openclaw channels login --channel whatsapp --account work
    ```

    Zie de kanaalhandleidingen: [Discord](/nl/channels/discord), [Telegram](/nl/channels/telegram), [WhatsApp](/nl/channels/whatsapp).

  </Step>
  <Step title="Voeg agents, accounts en bindingen toe">
    Voeg agents toe onder `agents.entries`, kanaalaccounts onder `channels.<channel>.accounts` en verbind ze met `bindings` (zie de voorbeelden hieronder).
  </Step>
  <Step title="Herstart en controleer">
    ```bash
    openclaw gateway restart
    openclaw agents list --bindings
    openclaw channels status --probe
    ```
  </Step>
</Steps>

## Meerdere agents, meerdere persona's

Elke geconfigureerde `agentId` vormt een afzonderlijke personagrens voor de kernstatus van de agent:

- Verschillende accounts per kanaal (per `accountId`).
- Verschillende persoonlijkheden (`AGENTS.md`/`SOUL.md` per agent).
- Afzonderlijke authenticatie en sessies, waarbij toegang tussen agents alleen via expliciete functies of Plugin-configuratie wordt ingeschakeld.

Hierdoor kunnen meerdere personen één Gateway delen terwijl de kernstatus van hun agents gescheiden blijft.

## Memory Wiki-kluizen per agent

Memory Wiki gebruikt standaard één algemene kluis. Om de gecompileerde kennis van een supportagent gescheiden te houden van die van een marketingagent, stel je `plugins.entries.memory-wiki.config.vault.scope` in op `agent`:

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

Het geconfigureerde pad is de bovenliggende map. OpenClaw voegt de genormaliseerde agent-id toe, waardoor paden ontstaan zoals `~/.openclaw/wiki/support` en `~/.openclaw/wiki/marketing`. Voor CLI- en Gateway-bewerkingen binnen het bereik van een agent moet expliciet een agent worden opgegeven wanneer meerdere agents zijn geconfigureerd. Zie [Memory Wiki-kluizen per agent](/nl/plugins/memory-wiki#per-agent-vaults) voor details over bridgefiltering, migratie en vertrouwensgrenzen.

## QMD-geheugen doorzoeken tussen agents

Als je één agent de QMD-sessietranscripten van een andere agent wilt laten doorzoeken, voeg je extra verzamelingen toe onder `agents.entries.*.memory.search.qmd.extraCollections`. Gebruik `memory.search.qmd.extraCollections` wanneer elke agent dezelfde verzamelingen moet delen.

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
              extraCollections: [{ path: "notes" }], // wordt binnen de werkruimte omgezet -> verzameling met de naam "notes-main"
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

Een pad van een extra verzameling kan tussen agents worden gedeeld, maar de `name` ervan blijft expliciet wanneer het pad buiten de werkruimte van de agent ligt. Paden binnen de werkruimte blijven beperkt tot de agent, zodat elke agent een eigen verzameling voor het doorzoeken van transcripten behoudt.

## Eén WhatsApp-nummer, meerdere personen (DM-splitsing)

Routeer verschillende WhatsApp-DM's naar verschillende agents op **één** WhatsApp-account door de E.164 van de afzender (`+15551234567`) te vergelijken met `peer.kind: "direct"`. Antwoorden worden nog steeds vanaf hetzelfde WhatsApp-nummer verzonden — er is geen afzenderidentiteit per agent.

<Note>
Directe chats worden standaard samengevoegd onder de hoofdsessiesleutel van de agent, dus voor echte isolatie is één agent per persoon vereist.
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

De toegangscontrole voor DM's (koppeling/lijst met toegestane afzenders) is algemeen per WhatsApp-account, niet per agent. Koppel gedeelde groepen aan één agent of gebruik [Broadcastgroepen](/nl/channels/broadcast-groups).

## Routeringsregels

Bindingen zijn deterministisch en de specifiekste wint. Zie [Kanaalroutering](/nl/channels/channel-routing#routing-rules-how-an-agent-is-chosen) voor de volledige volgorde van niveaus (exacte peer, bovenliggende peer, jokerteken voor peers, guild+rollen, guild, team, account, kanaal, standaardagent). Enkele regels die hier het vermelden waard zijn:

- Als meerdere bindingen binnen hetzelfde niveau overeenkomen, wint de eerste in de configuratievolgorde.
- Als een binding meerdere vergelijkingsvelden instelt (bijvoorbeeld `peer` + `guildId`), moeten alle opgegeven velden overeenkomen (`AND`-semantiek).
- Een binding zonder `accountId` komt alleen overeen met het standaardaccount, niet met elk account. Gebruik `accountId: "*"` als terugvaloptie voor het hele kanaal, of `accountId: "<name>"` voor één account. Als je dezelfde binding opnieuw toevoegt met een expliciete account-id, wordt de bestaande binding die alleen voor het kanaal geldt bijgewerkt in plaats van gedupliceerd.

## Meerdere accounts/telefoonnummers

Kanalen die meerdere accounts ondersteunen (bijvoorbeeld WhatsApp) gebruiken `accountId` om elke aanmelding te identificeren. Elke `accountId` wordt naar een eigen agent gerouteerd, zodat één server meerdere telefoonnummers kan hosten zonder sessies te vermengen.

Stel `channels.<channel>.defaultAccount` in om het account te kiezen dat wordt gebruikt wanneer `accountId` is weggelaten. Als dit niet is ingesteld, valt OpenClaw terug op `default` indien aanwezig, en anders op de eerste geconfigureerde account-id (gesorteerd).

Kanalen die meerdere accounts ondersteunen: `discord`, `feishu`, `googlechat`, `imessage`, `irc`, `line`, `mattermost`, `matrix`, `nextcloud-talk`, `nostr`, `signal`, `slack`, `telegram`, `whatsapp`, `zalo`, `zalouser`.

## Concepten

- `agentId`: één „brein” (werkruimte, authenticatie per agent, sessieopslag per agent).
- `accountId`: één instantie van een kanaalaccount (bijvoorbeeld WhatsApp-account `personal` tegenover `biz`).
- `binding`: routeert inkomende berichten naar een `agentId` op basis van `(channel, accountId, peer)`, en optioneel guild-/team-id's.
- Directe chats worden samengevoegd tot `agent:<agentId>:<mainKey>` („main” per agent; zie `session.mainKey`).

## Platformvoorbeelden

<AccordionGroup>
  <Accordion title="Discord-bots per agent">
    Elk Discord-botaccount wordt aan een unieke `accountId` gekoppeld. Koppel elk account aan een agent en houd per bot een toelatingslijst bij.

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

    - Nodig elke bot uit voor de guild en schakel Message Content Intent in.
    - Tokens staan in `channels.discord.accounts.<id>.token` (het standaardaccount kan `DISCORD_BOT_TOKEN` gebruiken).

  </Accordion>
  <Accordion title="Telegram-bots per agent">
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

    - Maak met BotFather één bot per agent en kopieer elk token.
    - Tokens staan in `channels.telegram.accounts.<id>.botToken` (het standaardaccount kan `TELEGRAM_BOT_TOKEN` gebruiken).
    - Nodig voor meerdere bots in dezelfde Telegram-groep elke bot uit en vermeld de bot die moet antwoorden.
    - Schakel voor elke groepsbot de BotFather Privacy Mode uit (`/setprivacy` -> Disable) en verwijder de bot vervolgens en voeg deze opnieuw toe, zodat Telegram de instelling toepast.
    - Sta groepen toe met `channels.telegram.groups`, of gebruik `groupPolicy: "open"` alleen voor vertrouwde groepsimplementaties.
    - Plaats gebruikers-id's van afzenders in `groupAllowFrom`. Groeps- en supergroep-id's horen in `channels.telegram.groups`, niet in `groupAllowFrom`.
    - Koppel op basis van `accountId`, zodat elke bot naar zijn eigen agent routeert.

  </Accordion>
  <Accordion title="WhatsApp-nummers per agent">
    Koppel elk account voordat je de Gateway start:

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

      // Deterministische routering: de eerste overeenkomst wint (meest specifieke eerst).
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },

        // Optionele overschrijving per peer (voorbeeld: stuur een specifieke groep naar de werkagent).
        {
          agentId: "work",
          match: {
            channel: "whatsapp",
            accountId: "personal",
            peer: { kind: "group", id: "1203630...@g.us" },
          },
        },
      ],

      // Standaard uitgeschakeld: berichten tussen agents moeten expliciet worden ingeschakeld en toegestaan.
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
              // Optionele overschrijving. Standaard: ~/.openclaw/credentials/whatsapp/personal
              // authDir: "~/.openclaw/credentials/whatsapp/personal",
            },
            biz: {
              // Optionele overschrijving. Standaard: ~/.openclaw/credentials/whatsapp/biz
              // authDir: "~/.openclaw/credentials/whatsapp/biz",
            },
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## Veelvoorkomende patronen

<Tabs>
  <Tab title="WhatsApp voor dagelijks gebruik + Telegram voor diepgaand werk">
    Splits per kanaal: routeer WhatsApp naar een snelle agent voor dagelijks gebruik en Telegram naar een Opus-agent.

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

    Deze voorbeelden gebruiken `accountId: "*"`, zodat de koppelingen blijven werken als je later accounts toevoegt. Om één privébericht/groep naar Opus te routeren terwijl de rest op chat blijft, voeg je voor die peer een `match.peer`-koppeling toe — overeenkomsten met peers krijgen altijd voorrang op regels voor het hele kanaal.

  </Tab>
  <Tab title="Hetzelfde kanaal, één peer naar Opus">
    Houd WhatsApp op de snelle agent, maar routeer één privébericht naar Opus:

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

    Peer-koppelingen krijgen altijd voorrang, dus plaats ze boven de regel voor het hele kanaal.

  </Tab>
  <Tab title="Familieagent gekoppeld aan een WhatsApp-groep">
    Koppel een speciale familieagent aan één WhatsApp-groep, met vermelding als voorwaarde en een strikter toolbeleid:

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

    Lijsten met toegestane/geweigerde tools bevatten **tools**, geen Skills. Als een Skill een binair bestand moet uitvoeren, zorg er dan voor dat `exec` is toegestaan en dat het binaire bestand in de sandbox aanwezig is. Stel voor strengere toegangscontrole `agents.entries.*.groupChat.mentionPatterns` in en houd groepstoelatingslijsten ingeschakeld voor het kanaal.

  </Tab>
</Tabs>

## Sandbox- en toolconfiguratie per agent

Elke agent kan zijn eigen sandbox- en toolbeperkingen hebben:

```js
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: {
          mode: "off",  // Geen sandbox voor persoonlijke agent
        },
        // Geen toolbeperkingen - alle tools zijn beschikbaar
      },
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",     // Altijd in een sandbox
          scope: "agent",  // Eén container per agent
          docker: {
            // Optionele eenmalige configuratie na het maken van de container
            setupCommand: "apt-get update && apt-get install -y git curl",
          },
        },
        tools: {
          allow: ["read"],                    // Alleen de leestool
          deny: ["exec", "write", "edit", "apply_patch"],    // Andere weigeren
        },
      },
    ],
  },
}
```

<Note>
`setupCommand` staat onder `sandbox.docker` en wordt eenmaal uitgevoerd wanneer de container wordt gemaakt. Overschrijvingen van `sandbox.docker.*` per agent worden genegeerd wanneer het bepaalde bereik `"shared"` is.
</Note>

Dit biedt je:

- **Beveiligingsisolatie**: beperk tools voor niet-vertrouwde agents.
- **Resourcebeheer**: plaats specifieke agents in een sandbox terwijl andere op de host blijven.
- **Flexibel beleid**: verschillende machtigingen per agent.

<Note>
`tools.elevated` heeft zowel een globale toegangspoort (`tools.elevated.enabled`/`allowFrom`) als een toegangspoort per agent (`agents.entries.*.tools.elevated.enabled`/`allowFrom`). De toegangspoort per agent kan de globale alleen verder beperken — beide moeten een afzender toestaan voordat opdrachten met verhoogde bevoegdheden kunnen worden uitgevoerd. Gebruik voor groepstargeting `agents.entries.*.groupChat.mentionPatterns`, zodat @vermeldingen duidelijk aan de bedoelde agent worden gekoppeld.
</Note>

Zie [Sandbox en tools voor meerdere agents](/nl/tools/multi-agent-sandbox-tools) voor gedetailleerde voorbeelden.

## Gerelateerd

- [ACP-agenten](/nl/tools/acp-agents) — externe codeharnassen uitvoeren
- [Kanaalroutering](/nl/channels/channel-routing) — hoe berichten naar agenten worden gerouteerd
- [Aanwezigheid](/nl/concepts/presence) — aanwezigheid en beschikbaarheid van agenten
- [Sessie](/nl/concepts/session) — sessie-isolatie en routering
- [Subagenten](/nl/tools/subagents) — agentruns op de achtergrond starten
