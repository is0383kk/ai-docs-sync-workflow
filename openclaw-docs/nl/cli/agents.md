---
read_when:
    - Je wilt meerdere geïsoleerde agents (werkruimten + routering + authenticatie)
summary: CLI-referentie voor `openclaw agents` (weergeven/toevoegen/verwijderen/koppelingen/koppelen/ontkoppelen/identiteit instellen)
title: Agents
x-i18n:
    generated_at: "2026-07-27T04:50:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 76a2e50462f6a52760dcb639405ed5f23857f2fa429469281e3acfa1eb61e974
    source_path: cli/agents.md
    workflow: 16
---

# `openclaw agents`

Beheer geïsoleerde agents (werkruimten + authenticatie + routering). Het uitvoeren van `openclaw agents` zonder subopdracht is gelijk aan `openclaw agents list`.

Gerelateerd:

- [Routering met meerdere agents](/nl/concepts/multi-agent)
- [Agentwerkruimte](/nl/concepts/agent-workspace)
- [Skills-configuratie](/nl/tools/skills-config): configuratie van de zichtbaarheid van Skills.

## Voorbeelden

```bash
openclaw agents list
openclaw agents list --bindings
openclaw agents add work --workspace ~/.openclaw/workspace-work
openclaw agents add work --workspace ~/.openclaw/workspace-work --bind telegram:*
openclaw agents add ops --workspace ~/.openclaw/workspace-ops --bind telegram:ops --non-interactive
openclaw agents bindings
openclaw agents bind --agent work --bind telegram:ops
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
openclaw agents set-identity --agent main --avatar avatars/openclaw.png
openclaw agents delete work
```

## Opdrachtenoverzicht

### `agents list`

Opties: `--json`, `--bindings` (neemt volledige routeringsregels op, niet alleen aantallen/samenvattingen per agent).

### `agents add [name]`

Opties: `--workspace <dir>`, `--model <id>`, `--agent-dir <dir>`, `--bind <channel[:accountId]>` (herhaalbaar), `--non-interactive`, `--json`.

- Als je een expliciete toevoegingsvlag doorgeeft, schakelt de opdracht over naar het niet-interactieve pad.
- Voor de niet-interactieve modus zijn zowel een agentnaam als `--workspace` vereist.
- `main` is gereserveerd en kan niet als nieuwe agent-id worden gebruikt.
- De interactieve modus initialiseert authenticatie door alleen overdraagbare statische aanmeldgegevens te kopiëren (`api_key` en statische `token`-profielen), tenzij aanmeldgegevens zich hiervoor afmelden met `copyToAgents: false`; OAuth-profielen met vernieuwingstokens worden niet gekopieerd, tenzij een provider zich hiervoor aanmeldt met `copyToAgents: true`. Zonder kopie blijft OAuth alleen beschikbaar via overerving bij het lezen uit de echte agentopslag van `main`. Als de geconfigureerde standaardagent niet `main` is, meld je dan afzonderlijk aan voor OAuth-profielen op de nieuwe agent.

### `agents bindings`

Opties: `--agent <id>`, `--json`.

### `agents bind`

Opties: `--agent <id>` (standaard de huidige standaardagent), `--bind <channel[:accountId]>` (herhaalbaar), `--json`.

### `agents unbind`

Opties: `--agent <id>` (standaard de huidige standaardagent), `--bind <channel[:accountId]>` (herhaalbaar), `--all`, `--json`. Accepteert `--all` of een of meer `--bind`-waarden, maar niet beide.

### `agents set-identity`

Opties: `--agent <id>`, `--workspace <dir>`, `--identity-file <path>`, `--from-identity`, `--name <name>`, `--theme <theme>`, `--emoji <emoji>`, `--avatar <value>`, `--json`. Zie [Identiteit instellen](#set-identity) hieronder.

### `agents delete <id>`

Opties: `--force`, `--json`.

- `main` kan niet worden verwijderd.
- Zonder `--force` is interactieve bevestiging vereist (mislukt in een niet-TTY-sessie; voer de opdracht opnieuw uit met `--force`).
- Mappen voor de werkruimte, agentstatus en sessietranscripten worden naar de prullenmand verplaatst en niet permanent verwijderd. Als de prullenmand niet beschikbaar is, wordt de agentconfiguratie nog steeds verwijderd en worden de paden gemeld die handmatig moeten worden opgeschoond.
- Wanneer de Gateway bereikbaar is, verloopt de verwijdering via de Gateway, zodat het opschonen van de configuratie en sessieopslag dezelfde schrijver gebruikt als het runtimeverkeer. Als de Gateway onbereikbaar is, valt de CLI terug op het offline lokale pad.
- Als de werkruimte van een andere agent hetzelfde pad gebruikt, zich binnen deze werkruimte bevindt of deze werkruimte bevat, blijft de werkruimte behouden en rapporteert `--json` `workspaceRetained`, `workspaceRetainedReason` en `workspaceSharedWith`.

## Routeringskoppelingen

Gebruik routeringskoppelingen om inkomend kanaalverkeer aan een specifieke agent te koppelen.

Als je ook per agent verschillende zichtbare Skills wilt, configureer je `agents.defaults.skills` en `agents.entries.*.skills` in `openclaw.json`. Zie [Skills-configuratie](/nl/tools/skills-config) en [Configuratiereferentie](/nl/gateway/config-agents#agentsdefaultsskills).

Koppelingen weergeven:

```bash
openclaw agents bindings
openclaw agents bindings --agent work
openclaw agents bindings --json
```

Koppelingen toevoegen:

```bash
openclaw agents bind --agent work --bind telegram:ops --bind discord:guild-a
```

Je kunt ook koppelingen toevoegen wanneer je een agent maakt:

```bash
openclaw agents add work --workspace ~/.openclaw/workspace-work --bind telegram:* --bind discord:*
```

Als je `accountId` (`--bind <channel>`) weglaat, leidt OpenClaw deze af uit configuratiehooks van de Plugin, een afgedwongen accountkoppeling of het aantal geconfigureerde accounts van het kanaal.

Als je `--agent` voor `bind` of `unbind` weglaat, richt OpenClaw zich op de huidige standaardagent.

### Indeling van `--bind`

| Indeling                     | Betekenis                                                                                          |
| ---------------------------- | -------------------------------------------------------------------------------------------------- |
| `--bind <channel>:*`         | Komt overeen met alle accounts op het kanaal.                                                      |
| `--bind <channel>:<account>` | Komt overeen met één account.                                                                      |
| `--bind <channel>`           | Komt alleen overeen met het standaardaccount, tenzij de CLI veilig een pluginspecifiek accountbereik kan bepalen. |

### Gedrag van het koppelingsbereik

- Een opgeslagen koppeling zonder `accountId` komt alleen overeen met het standaardaccount van het kanaal.
- `accountId: "*"` is de kanaalbrede terugvaloptie (alle accounts) en is minder specifiek dan een expliciete accountkoppeling.
- Als dezelfde agent al een overeenkomende kanaalkoppeling zonder `accountId` heeft en je later een koppeling maakt met een expliciete of bepaalde `accountId`, werkt OpenClaw die bestaande koppeling ter plaatse bij in plaats van een duplicaat toe te voegen.

Voorbeelden:

```bash
# overeenkomen met alle accounts op het kanaal
openclaw agents bind --agent work --bind telegram:*

# overeenkomen met een specifiek account
openclaw agents bind --agent work --bind telegram:ops

# eerste koppeling alleen op kanaalniveau
openclaw agents bind --agent work --bind telegram

# later bijwerken naar een koppeling met accountbereik
openclaw agents bind --agent work --bind telegram:alerts
```

Na de bijwerking is de routering voor die koppeling beperkt tot `telegram:alerts`. Als je ook routering voor het standaardaccount wilt, voeg je die expliciet toe (bijvoorbeeld `--bind telegram:default`).

Koppelingen verwijderen:

```bash
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents unbind --agent work --all
```

## Identiteitsbestanden

Elke agentwerkruimte kan een `IDENTITY.md` in de hoofdmap van de werkruimte bevatten:

- Voorbeeldpad: `~/.openclaw/workspace/IDENTITY.md`
- `set-identity --from-identity` leest uit de hoofdmap van de werkruimte (of uit een expliciete `--identity-file`).

Avatarpaden worden ten opzichte van de hoofdmap van de werkruimte bepaald en kunnen deze niet verlaten, zelfs niet via een symbolische koppeling.

## Identiteit instellen

`set-identity` schrijft velden naar `agents.entries.*.identity`: `name`, `theme`, `emoji`, `avatar` (pad relatief aan de werkruimte, http(s)-URL of data-URI).

- `--agent` of `--workspace` selecteert de doelagent. Als `--workspace` met meer dan één agent overeenkomt, mislukt de opdracht en wordt je gevraagd `--agent` door te geven.
- Lokale avatarafbeeldingen met een pad relatief aan de werkruimte zijn beperkt tot 2 MB. HTTP(S)-URL's en `data:`-URI's worden niet aan de lokale bestandsgroottelimiet getoetst.
- Wanneer geen expliciete identiteitsvelden zijn opgegeven, leest de opdracht identiteitsgegevens uit `IDENTITY.md`.

Laden uit `IDENTITY.md`:

```bash
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
```

Velden expliciet overschrijven:

```bash
openclaw agents set-identity --agent main --name "OpenClaw" --emoji "🦞" --avatar avatars/openclaw.png
```

Configuratievoorbeeld:

```json5
{
  agents: {
    list: [
      {
        id: "main",
        identity: {
          name: "OpenClaw",
          theme: "space lobster",
          emoji: "🦞",
          avatar: "avatars/openclaw.png",
        },
      },
    ],
  },
}
```

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Routering met meerdere agents](/nl/concepts/multi-agent)
- [Agentwerkruimte](/nl/concepts/agent-workspace)
