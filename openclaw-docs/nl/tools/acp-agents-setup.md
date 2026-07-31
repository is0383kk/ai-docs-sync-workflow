---
read_when:
    - De acpx-harness voor Claude Code / Codex / Gemini CLI installeren of configureren
    - De plugin-tools- of OpenClaw-tools-MCP-bridge inschakelen
    - ACP-machtigingsmodi configureren
summary: 'ACP-agents instellen: acpx-harnasconfiguratie, Plugin-installatie, machtigingen'
title: ACP-agents — configuratie
x-i18n:
    generated_at: "2026-07-27T06:35:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ae3750092175b44252dd080717a1af176995df43c653f245f82d7e556cfd25eb
    source_path: tools/acp-agents-setup.md
    workflow: 16
---

Zie [ACP-agents](/nl/tools/acp-agents) voor het overzicht, het operator-runbook en de concepten.

Deze pagina behandelt de acpx-harnasconfiguratie, de Plugin-installatie voor de MCP-bridges en de machtigingsconfiguratie.

Gebruik deze pagina alleen wanneer je de ACP/acpx-route instelt. Gebruik voor de native Codex
app-server-runtimeconfiguratie [Codex-harnas](/nl/plugins/codex-harness). Gebruik voor
OpenAI API-sleutels of de Codex OAuth-modelproviderconfiguratie
[OpenAI](/nl/providers/openai).

Codex heeft twee OpenClaw-routes:

| Route                      | Configuratie/opdracht                                  | Installatiepagina                       |
| -------------------------- | ------------------------------------------------------ | --------------------------------------- |
| Native Codex-app-server    | `/codex ...`, `openai/gpt-*`-agentverwijzingen                | [Codex-harnas](/nl/plugins/codex-harness) |
| Expliciete Codex ACP-adapter | `/acp spawn codex`, `runtime: "acp", agentId: "codex"` | Deze pagina                             |

Geef de voorkeur aan de native route, tenzij je expliciet ACP/acpx-gedrag nodig hebt.

## Ondersteuning voor acpx-harnas (huidig)

Ingebouwde acpx-harnasaliassen (uit de vastgezette afhankelijkheid `acpx`):

| Alias        | Omvat                                                                                                           |
| ------------ | --------------------------------------------------------------------------------------------------------------- |
| `claude`     | [Claude Code](https://claude.ai/code)                                                                           |
| `codex`      | [Codex CLI](https://codex.openai.com)                                                                           |
| `copilot`    | [GitHub Copilot CLI](https://docs.github.com/copilot/how-tos/copilot-chat/use-copilot-chat-in-the-command-line) |
| `cursor`     | [Cursor CLI](https://cursor.com/docs/cli/acp) (`cursor-agent acp`)                                              |
| `droid`      | [Factory Droid](https://www.factory.ai)                                                                         |
| `fast-agent` | [fast-agent](https://fast-agent.ai)                                                                             |
| `gemini`     | [Gemini CLI](https://github.com/google/gemini-cli)                                                              |
| `iflow`      | [iFlow CLI](https://github.com/iflow-ai/iflow-cli)                                                              |
| `kilocode`   | [Kilocode](https://kilocode.ai)                                                                                 |
| `kimi`       | [Kimi CLI](https://github.com/MoonshotAI/kimi-cli)                                                              |
| `kiro`       | [Kiro CLI](https://kiro.dev)                                                                                    |
| `mux`        | [Mux](https://mux.coder.com)                                                                                    |
| `opencode`   | [OpenCode](https://opencode.ai)                                                                                 |
| `openclaw`   | OpenClaw ACP-bridge (native `openclaw acp`)                                                                     |
| `pi`         | [Pi Coding Agent](https://github.com/mariozechner/pi)                                                           |
| `qoder`      | [Qoder CLI](https://docs.qoder.com/cli/acp)                                                                     |
| `qwen`       | [Qwen Code](https://github.com/QwenLM/qwen-code)                                                                |
| `trae`       | [Trae CLI](https://docs.trae.cn/cli)                                                                            |

`factory-droid` en `factorydroid` verwijzen eveneens naar de ingebouwde `droid`-adapter.

Wanneer OpenClaw de acpx-backend gebruikt, geef je voor `agentId` de voorkeur aan deze waarden, tenzij je acpx-configuratie aangepaste agentaliasen definieert.
Als je lokale Cursor-installatie ACP nog steeds aanbiedt als `agent acp`, overschrijf dan de agentopdracht `cursor` in je acpx-configuratie in plaats van de ingebouwde standaardwaarde te wijzigen.

Bij direct gebruik van de acpx CLI kunnen ook willekeurige adapters worden aangestuurd via `--agent <command>`, maar deze onbewerkte uitweg is een functie van de acpx CLI (niet het normale OpenClaw-pad `agentId`).

Modelbesturing is afhankelijk van de mogelijkheden van de adapter. Codex ACP-modelverwijzingen worden
vóór het opstarten door OpenClaw genormaliseerd. Andere harnassen hebben ACP `models` plus
ondersteuning voor `session/set_model` nodig; als een harnas noch die ACP-mogelijkheid
noch een eigen opstartvlag voor het model aanbiedt, kan OpenClaw/acpx geen modelselectie afdwingen.

## Vereiste configuratie

Basisconfiguratie voor ACP in de kern:

```json5
{
  acp: {
    enabled: true,
    // Optioneel. Standaard true; stel in op false om ACP-dispatch te pauzeren terwijl /acp-bediening beschikbaar blijft.
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "codex",
    allowedAgents: [
      "claude",
      "codex",
      "copilot",
      "cursor",
      "droid",
      "gemini",
      "iflow",
      "kilocode",
      "kimi",
      "kiro",
      "openclaw",
      "opencode",
      "qwen",
    ],
    stream: {
      deliveryMode: "live",
    },
  },
}
```

De configuratie voor threadbinding wordt gedeeld door de ondersteunde kanaaladapters:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
      spawnSessions: true,
    },
  },
}
```

Als threadgebonden ACP-spawning niet werkt, controleer dan eerst de functievlag van de adapter:

- Discord: `session.threadBindings.spawnSessions=true`

Bindingen aan het huidige gesprek vereisen geen aanmaak van een onderliggende thread. Ze vereisen een actieve gesprekscontext en een kanaaladapter die ACP-gespreksbindingen aanbiedt.

Zie [Configuratiereferentie](/nl/gateway/configuration-reference).

## Plugin-installatie voor de acpx-backend

Verpakte installaties gebruiken de officiële runtime-Plugin `@openclaw/acpx` voor ACP.
Installeer en activeer deze voordat je ACP-harnassessies gebruikt:

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

Broncodecheck-outs kunnen na `pnpm install` ook de lokale werkruimte-Plugin gebruiken.

Begin met:

```text
/acp doctor
```

Als je `acpx` hebt uitgeschakeld, deze via `plugins.allow` / `plugins.deny` hebt geweigerd, of
terug wilt schakelen naar de verpakte Plugin, gebruik je het expliciete pakketpad:

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

Installatie vanuit de lokale werkruimte tijdens ontwikkeling:

```bash
openclaw plugins install ./path/to/local/acpx-plugin
```

Controleer daarna de status van de backend:

```text
/acp doctor
```

### Opstartcontrole van de acpx-runtime

De Plugin `acpx` sluit de ACP-runtime rechtstreeks in (geen afzonderlijk binair bestand of
versie `acpx` om te configureren). Standaard registreert deze de ingesloten backend tijdens het
opstarten van de Gateway en wacht deze vóór het gateway-signaal `ready`
op een opstartcontrole. Stel `OPENCLAW_ACPX_RUNTIME_STARTUP_PROBE=0` of
`OPENCLAW_SKIP_ACPX_RUNTIME_PROBE=1` alleen in voor scripts of omgevingen die
de opstartcontrole bewust uitgeschakeld houden. Voer `/acp doctor` uit voor een expliciete
controle op aanvraag.

Overschrijf de opdracht van een afzonderlijke ACP-agent met gestructureerde argumenten wanneer een pad
of vlagwaarde één argv-token moet blijven:

```json
{
  "plugins": {
    "entries": {
      "acpx": {
        "enabled": true,
        "config": {
          "agents": {
            "claude": {
              "command": "node",
              "args": ["/path/to/custom adapter.mjs", "--verbose"]
            }
          }
        }
      }
    }
  }
}
```

- `agents.<id>.command` is het uitvoerbare bestand of de bestaande opdrachttekenreeks voor die ACP-agent.
- `agents.<id>.args` is optioneel. Elk array-item krijgt shell-aanhalingstekens voordat OpenClaw het doorgeeft aan het huidige register voor acpx-opdrachttekenreeksen.

Zie [Plugins](/nl/tools/plugin).

### Automatisch downloaden van adapters

`acpx` downloadt ACP-adapters (bijvoorbeeld de ACP-bridges van Claude en Codex)
bij het eerste gebruik automatisch via `npx`. Je hoeft adapterpakketten niet
handmatig te installeren en er is geen afzonderlijke postinstallatiestap voor OpenClaw zelf. Als het
downloaden of spawnen van een adapter mislukt, meldt `/acp doctor` de fout.

### MCP-bridge voor Plugin-tools

Standaard stellen ACPX-sessies door OpenClaw-Plugins geregistreerde tools **niet** beschikbaar aan
het ACP-harnas.

Als je wilt dat ACP-agents zoals Codex of Claude Code geïnstalleerde
OpenClaw-Plugin-tools kunnen aanroepen, zoals het ophalen/opslaan van geheugen, activeer je de speciale bridge:

```bash
openclaw config set plugins.entries.acpx.config.pluginToolsMcpBridge true
```

Wat dit doet:

- Voegt tijdens het opstarten van de ACPX-sessie een ingebouwde MCP-server met de naam `openclaw-plugin-tools`
  toe.
- Stelt Plugin-tools beschikbaar die al zijn geregistreerd door geïnstalleerde en geactiveerde OpenClaw-
  Plugins.
- Geeft de identiteit van de actieve ACP-sessie door aan fabrieken voor Plugin-tools, zodat
  agentgebonden tools binnen de naamruimte van die agent blijven.
- Houdt de functie expliciet en standaard uitgeschakeld.

Opmerkingen over beveiliging en vertrouwen:

- Dit breidt het tooloppervlak van het ACP-harnas uit.
- ACP-agents krijgen alleen toegang tot Plugin-tools die al actief zijn in de Gateway.
- Behandel dit als dezelfde vertrouwensgrens als wanneer je die Plugins in
  OpenClaw zelf laat uitvoeren.
- Controleer geïnstalleerde Plugins voordat je dit activeert.

Aangepaste `mcpServers` blijven werken zoals voorheen. De ingebouwde bridge voor Plugin-tools is een
aanvullend optioneel gemak, geen vervanging voor generieke MCP-serverconfiguratie.

### MCP-bridge voor OpenClaw-tools

Standaard stellen ACPX-sessies ingebouwde OpenClaw-tools ook **niet** beschikbaar via
MCP. Activeer de afzonderlijke bridge voor kerntools wanneer een ACP-agent geselecteerde
ingebouwde tools nodig heeft, zoals `cron`:

```bash
openclaw config set plugins.entries.acpx.config.openClawToolsMcpBridge true
```

Wat dit doet:

- Voegt tijdens het opstarten van de ACPX-sessie een ingebouwde MCP-server met de naam `openclaw-tools`
  toe.
- Stelt geselecteerde ingebouwde OpenClaw-tools beschikbaar. De eerste server stelt `cron` beschikbaar.
- Houdt de blootstelling van kerntools expliciet en standaard uitgeschakeld.

### Configuratie van de time-out voor runtimebewerkingen

De Plugin `acpx` geeft het opstarten van de ingesloten runtime en besturingsbewerkingen standaard 120
seconden. Hierdoor krijgen tragere harnassen zoals Gemini CLI genoeg tijd
om het opstarten en initialiseren van ACP te voltooien. Overschrijf dit als je host een
andere bewerkingslimiet nodig heeft:

```bash
openclaw config set plugins.entries.acpx.config.timeoutSeconds 180
```

Runtimebeurten gebruiken de time-outs voor OpenClaw-agents/runs, waaronder `/acp timeout`.
`sessions_spawn` accepteert geen time-outoverschrijvingen per aanroep; het operatorpad
is `agents.defaults.subagents.runTimeoutSeconds`. Start de Gateway opnieuw nadat je
`timeoutSeconds` hebt gewijzigd.

### Configuratie van de agent voor statuscontroles

Wanneer `/acp doctor` of de opstartcontrole de backend controleert, test de meegeleverde Plugin `acpx`
één harnasagent. Als `acp.allowedAgents` is ingesteld, wordt standaard
de eerste toegestane agent gebruikt; anders is de standaardwaarde `codex`. Als je implementatie
een andere ACP-agent voor statuscontroles nodig heeft, stel je de controleagent expliciet in:

```bash
openclaw config set plugins.entries.acpx.config.probeAgent claude
```

Start de Gateway opnieuw nadat je deze waarde hebt gewijzigd.

## Machtigingsconfiguratie

ACP-sessies worden niet-interactief uitgevoerd — er is geen TTY om machtigingsprompts voor het schrijven van bestanden en uitvoeren van shellopdrachten goed te keuren of te weigeren. De acpx-Plugin biedt twee configuratiesleutels waarmee wordt bepaald hoe machtigingen worden afgehandeld:

Deze ACPX-harnasmachtigingen staan los van OpenClaw-uitvoeringsgoedkeuringen en van bypassvlaggen van CLI-backendleveranciers, zoals Claude CLI `--permission-mode bypassPermissions`. ACPX `approve-all` is de noodschakelaar op harnasniveau voor ACP-sessies.

Zie [Machtigingsmodi](/nl/tools/permission-modes) voor een bredere vergelijking tussen OpenClaw `tools.exec.mode`, Codex Guardian-goedkeuringen en ACPX-harnasmachtigingen.

### `permissionMode`

Bepaalt welke bewerkingen de harnasagent zonder bevestiging kan uitvoeren.

| Waarde           | Gedrag                                                        |
| ---------------- | ------------------------------------------------------------- |
| `approve-all`   | Keur alle schrijfbewerkingen naar bestanden en shellopdrachten automatisch goed. |
| `approve-reads` | Keur alleen leesbewerkingen automatisch goed; voor schrijven en uitvoeren is bevestiging vereist. |
| `deny-all`      | Weiger alle machtigingsverzoeken.                              |

### `nonInteractivePermissions`

Bepaalt wat er gebeurt wanneer een machtigingsverzoek zou worden weergegeven, maar er geen interactieve TTY beschikbaar is (wat altijd het geval is voor ACP-sessies).

| Waarde | Gedrag                                                                 |
| ------ | ---------------------------------------------------------------------- |
| `fail` | Breek de sessie af met `PermissionPromptUnavailableError`. **(standaard)** |
| `deny` | Weiger de machtiging stilzwijgend en ga door (geleidelijke degradatie). |

### Configuratie

Stel dit in via de Plugin-configuratie:

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

Start de Gateway opnieuw nadat je deze waarden hebt gewijzigd.

<Warning>
OpenClaw gebruikt standaard `permissionMode=approve-reads` en `nonInteractivePermissions=fail`. In niet-interactieve ACP-sessies kan elke schrijf- of uitvoeringsbewerking die een machtigingsverzoek activeert, mislukken met `PermissionPromptUnavailableError: Permission prompt unavailable in non-interactive mode`.

Als je machtigingen moet beperken, stel je `nonInteractivePermissions` in op `deny`, zodat sessies geleidelijk degraderen in plaats van vastlopen.
</Warning>

## Gerelateerd

- [ACP-agenten](/nl/tools/acp-agents) — overzicht, operationeel draaiboek, concepten
- [Subagenten](/nl/tools/subagents)
- [Routering met meerdere agenten](/nl/concepts/multi-agent)
