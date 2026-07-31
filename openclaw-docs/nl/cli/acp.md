---
read_when:
    - ACP-gebaseerde IDE-integraties instellen
    - ACP-sessieroutering naar de Gateway debuggen
summary: Voer de ACP-bridge uit voor IDE-integraties
title: ACP
x-i18n:
    generated_at: "2026-07-27T05:45:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: becdcfdd1cc62b206cc92e9b8248c79a2ff63cfc3779d8a124b9713e779ad33c
    source_path: cli/acp.md
    workflow: 16
---

Voer de [Agent Client Protocol (ACP)](https://agentclientprotocol.com/)-bridge uit die communiceert met een OpenClaw Gateway.

`openclaw acp` communiceert via stdio met ACP voor IDE's en stuurt prompts via WebSocket door naar de Gateway, waarbij ACP-sessies aan Gateway-sessiesleutels gekoppeld blijven. Het is een door de Gateway ondersteunde ACP-bridge, geen volledige ACP-native editorruntime: de focus ligt op sessieroutering, promptbezorging en streamingupdates.

Als je wilt dat een externe MCP-client rechtstreeks communiceert met OpenClaw-kanaalgesprekken in plaats van een ACP-harnesssessie te hosten, gebruik je in plaats daarvan [`openclaw mcp serve`](/nl/cli/mcp).

## Wat dit niet is

`openclaw acp` betekent dat OpenClaw als ACP-server fungeert: een IDE of ACP-client maakt verbinding met OpenClaw en OpenClaw stuurt dat werk door naar een Gateway-sessie.

Dit verschilt van [ACP-agents](/nl/tools/acp-agents), waarbij OpenClaw via `acpx` een externe harness zoals Codex of Claude Code uitvoert.

Vuistregel:

- editor/client wil via ACP met OpenClaw communiceren: gebruik `openclaw acp`
- OpenClaw moet Codex/Claude/Gemini als ACP-harness starten: gebruik `/acp spawn` en [ACP-agents](/nl/tools/acp-agents)

## Compatibiliteitsmatrix

| ACP-onderdeel                                                          | Status          | Opmerkingen                                                                                                                                                                                                                             |
| --------------------------------------------------------------------- | --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `initialize`, `newSession`, `prompt`, `cancel`                        | Geïmplementeerd | Kernstroom van de bridge via stdio naar Gateway-chat/send + afbreken.                                                                                                                                                                   |
| `listSessions`, slash-opdrachten                                      | Geïmplementeerd | De sessielijst werkt met de Gateway-sessiestatus, met begrensde cursorpaginering en `cwd`-filtering wanneer Gateway-sessierijen werkruimtemetadata bevatten; opdrachten worden via `available_commands_update` bekendgemaakt.             |
| Metadata over sessieafstamming                                        | Geïmplementeerd | Sessielijsten en momentopnamen met sessie-informatie bevatten de bovenliggende en onderliggende OpenClaw-afstamming in `_meta`, zodat ACP-clients subagentgrafen zonder private Gateway-zijkanalen kunnen weergeven.              |
| `resumeSession`, `closeSession`                                       | Geïmplementeerd | Hervatten koppelt een ACP-sessie opnieuw aan een bestaande Gateway-sessie zonder de geschiedenis opnieuw af te spelen. Sluiten annuleert actief bridgewerk, handelt wachtende prompts af als geannuleerd en geeft de bridgesessiestatus vrij. |
| `loadSession`                                                         | Gedeeltelijk    | Koppelt de ACP-sessie opnieuw aan een Gateway-sessiesleutel en speelt de ACP-eventlogboekgeschiedenis opnieuw af voor door de bridge gemaakte sessies. Oudere sessies of sessies zonder logboek vallen terug op opgeslagen gebruikers-/assistenttekst. |
| Promptinhoud (`text`, ingesloten `resource`, afbeeldingen)              | Gedeeltelijk    | Tekst/resources worden samengevoegd tot chatinvoer; afbeeldingen worden Gateway-bijlagen.                                                                                                                                               |
| Sessiemodi                                                           | Gedeeltelijk    | `session/set_mode` wordt ondersteund; de bridge biedt door de Gateway ondersteunde sessiebediening voor denkniveau, tooluitvoerigheid, redenering, gebruiksdetails en acties met verhoogde rechten. Bredere ACP-native modus-/configuratieoppervlakken vallen nog buiten het bereik. |
| Streaming van gedachten                                               | Geïmplementeerd | Denksequenties van het model worden gestreamd als `agent_thought_chunk`-sessie-updates. ACP-native sessieplannen worden niet verzonden.                                                                                                      |
| Sessie-informatie en gebruiksupdates                                  | Gedeeltelijk    | De bridge verzendt `session_info_update`- en naar beste vermogen `usage_update`-meldingen vanuit gecachte momentopnamen van Gateway-sessies. Het gebruik is bij benadering en wordt alleen verzonden wanneer de totale aantallen Gateway-tokens als actueel zijn gemarkeerd. |
| Streaming van tools                                                   | Gedeeltelijk    | `tool_call`-/`tool_call_update`-gebeurtenissen bevatten onbewerkte I/O, tekstinhoud en naar beste vermogen bestandslocaties wanneer argumenten/resultaten van Gateway-tools deze beschikbaar stellen. Ingesloten terminals en rijkere diff-native uitvoer worden niet beschikbaar gesteld. |
| Uitvoeringsgoedkeuringen                                              | Gedeeltelijk    | Gateway-prompts voor uitvoeringsgoedkeuring tijdens actieve ACP-promptbeurten worden met `session/request_permission` doorgestuurd naar de ACP-client.                                                                                              |
| MCP-servers per sessie (`mcpServers`)                               | Niet ondersteund | De bridgemodus weigert aanvragen voor MCP-servers per sessie. Configureer MCP in plaats daarvan op de OpenClaw Gateway of agent.                                                                                                        |
| Bestandssysteemmethoden van de client (`fs/read_text_file`, `fs/write_text_file`) | Niet ondersteund | De bridge roept geen bestandssysteemmethoden van de ACP-client aan.                                                                                                                                                                    |
| Terminalmethoden van de client (`terminal/*`)                       | Niet ondersteund | De bridge maakt geen ACP-clientterminals en streamt geen terminal-id's via toolaanroepen.                                                                                                                                               |

## Bekende beperkingen

- `loadSession` speelt de volledige ACP-eventlogboekgeschiedenis alleen opnieuw af voor door de bridge gemaakte sessies. Oudere sessies of sessies zonder logboek gebruiken een transcriptfallback en reconstrueren geen historische toolaanroepen of systeemmeldingen.
- Als meerdere ACP-clients dezelfde Gateway-sessiesleutel delen, worden gebeurtenissen en annuleringen naar beste vermogen gerouteerd in plaats van strikt per client geïsoleerd. Geef de voorkeur aan de standaard geïsoleerde `acp-bridge:<uuid>`-sessies wanneer je duidelijk gescheiden editorlokale beurten nodig hebt.
- Gateway-stopstatussen worden omgezet in ACP-stopredenen, maar die koppeling is minder expressief dan die van een volledig ACP-native runtime.
- Sessiebesturing biedt een gerichte subset van Gateway-instellingen: denkniveau, tooluitvoerigheid, redenering, gebruiksdetails en acties met verhoogde rechten. Modelselectie en bediening van de uitvoeringshost worden niet als ACP-configuratieopties aangeboden.
- `session_info_update` en `usage_update` zijn afgeleid van momentopnamen van Gateway-sessies, niet van actuele ACP-native runtimeboekhouding. Het gebruik is bij benadering, bevat geen kostengegevens en wordt alleen verzonden wanneer de Gateway de totale tokengegevens als actueel markeert.
- Meeloopgegevens van tools worden naar beste vermogen verstrekt: de bridge toont bestandspaden die in bekende toolargumenten/-resultaten voorkomen, maar verzendt geen ACP-terminals of gestructureerde bestandsdiffs.
- Het doorsturen van uitvoeringsgoedkeuringen is beperkt tot de actieve ACP-promptbeurt; goedkeuringen uit andere Gateway-sessies worden genegeerd.

## Gebruik

```bash
openclaw acp

# Externe Gateway
openclaw acp --url wss://gateway-host:18789 --token <token>

# Externe Gateway (token uit bestand)
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# Koppelen aan een bestaande sessiesleutel
openclaw acp --session agent:main:main

# Koppelen op label (moet al bestaan)
openclaw acp --session-label "support inbox"

# De sessiesleutel vóór de eerste prompt opnieuw instellen
openclaw acp --session agent:main:main --reset-session
```

## ACP-client (foutopsporing)

Gebruik de ingebouwde ACP-client om de bridge zonder IDE kort te controleren. Deze start de ACP-bridge en laat je interactief prompts typen.

```bash
openclaw acp client

# De gestarte bridge naar een externe Gateway laten verwijzen
openclaw acp client --server-args --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# De serveropdracht overschrijven (standaard: openclaw)
openclaw acp client --server "node" --server-args openclaw.mjs acp --url ws://127.0.0.1:19001
```

Machtigingsmodel (foutopsporingsmodus van client):

- Automatische goedkeuring is gebaseerd op een toelatingslijst en geldt alleen voor vertrouwde kerntool-id's.
- Automatische goedkeuring van `read` is beperkt tot de huidige werkmap (`--cwd` indien ingesteld).
- ACP keurt alleen beperkte alleen-lezenklassen automatisch goed: `read`-aanroepen binnen de actieve huidige werkmap, plus alleen-lezende zoektools (`search`, `web_search`, `memory_search`). Onbekende/niet-kern-tools, leesbewerkingen buiten het bereik, tools die opdrachten kunnen uitvoeren, besturingsvlaktools, wijzigende tools en interactieve stromen vereisen altijd expliciete goedkeuring via een prompt.
- Door de server verstrekte `toolCall.kind` wordt behandeld als niet-vertrouwde metadata, niet als autorisatiebron.
- Dit ACP-bridgebeleid staat los van ACPX-harnessmachtigingen. Als je OpenClaw via de `acpx`-backend uitvoert, is `plugins.entries.acpx.config.permissionMode=approve-all` de noodschakelaar voor de onbeperkte modus voor die harnesssessie.

## Protocol-smoketest

Start voor foutopsporing op protocolniveau een Gateway met geïsoleerde status en stuur `openclaw acp` via stdio aan met een ACP JSON-RPC-client. Test `initialize`, `session/new`, `session/list` met een absolute `cwd`, `session/resume`, `session/close`, dubbel sluiten en een ontbrekende hervatting.

Het bewijs moet de bekendgemaakte levenscyclusmogelijkheden, een door de Gateway ondersteunde sessierij, updatemeldingen en het Gateway-`sessions.list`-logboek bevatten:

```json
{
  "initialize": {
    "protocolVersion": 1,
    "agentCapabilities": {
      "sessionCapabilities": {
        "list": {},
        "resume": {},
        "close": {}
      }
    }
  },
  "listSessions": {
    "sessions": [
      {
        "sessionId": "agent:main:acp-smoke",
        "cwd": "/path/to/workspace",
        "_meta": {
          "sessionKey": "agent:main:acp-smoke",
          "kind": "direct"
        }
      }
    ],
    "nextCursor": null
  },
  "notifications": ["session_info_update", "available_commands_update", "usage_update"],
  "gatewayLogTail": ["[gateway] ready", "[ws] ⇄ res ✓ sessions.list 305ms"]
}
```

Gebruik `openclaw gateway call sessions.list` niet als het enige ACP-bewijs. Dat CLI-pad kan om een verhoging naar een operatorscope met een nieuw token vragen; de correctheid van de ACP-bridge wordt bewezen door ACP-stdioframes plus het Gateway-`sessions.list`-logboek.

## Dit gebruiken

Gebruik ACP wanneer een IDE (of andere client) Agent Client Protocol spreekt en je daarmee een OpenClaw Gateway-sessie wilt aansturen.

1. Zorg dat de Gateway actief is (lokaal of extern).
2. Configureer het Gateway-doel (configuratie of vlaggen).
3. Stel je IDE zo in dat deze `openclaw acp` via stdio uitvoert.

Voorbeeldconfiguratie (opgeslagen):

```bash
openclaw config set gateway.remote.url wss://gateway-host:18789
openclaw config set gateway.remote.token <token>
```

Voorbeeld van rechtstreeks uitvoeren (zonder configuratie weg te schrijven):

```bash
openclaw acp --url wss://gateway-host:18789 --token <token>
# aanbevolen voor de veiligheid van lokale processen
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token
```

## Agents selecteren

ACP kiest agents niet rechtstreeks. Het routeert via de Gateway-sessiesleutel. Gebruik sessiesleutels met agentbereik om een specifieke agent te benaderen:

```bash
openclaw acp --session agent:main:main
openclaw acp --session agent:design:main
openclaw acp --session agent:qa:bug-123
```

Elke ACP-sessie wordt aan één Gateway-sessiesleutel gekoppeld. Eén agent kan veel sessies hebben; ACP gebruikt standaard een geïsoleerde `acp-bridge:<uuid>`-sessie, tenzij je de sleutel of het label overschrijft.

`mcpServers` per sessie worden niet ondersteund in bridge-modus. Als een ACP-client deze tijdens `newSession` of `loadSession` verstuurt, retourneert de bridge een duidelijke fout in plaats van ze stilzwijgend te negeren.

Als je wilt dat door ACPX ondersteunde sessies toegang hebben tot OpenClaw-plugintools of geselecteerde ingebouwde tools zoals `cron`, schakel je de ACPX MCP-bridges aan de Gateway-zijde in in plaats van te proberen `mcpServers` per sessie door te geven. Zie [ACP-agents](/nl/tools/acp-agents-setup#plugin-tools-mcp-bridge) en [MCP-bridge voor OpenClaw-tools](/nl/tools/acp-agents-setup#openclaw-tools-mcp-bridge).

## Gebruik vanuit `acpx` (Codex, Claude, andere ACP-clients)

Als je wilt dat een codeeragent zoals Codex of Claude Code via ACP met je OpenClaw-bot communiceert, gebruik je `acpx` met het ingebouwde `openclaw`-doel.

Gebruikelijke werkwijze:

1. Start de Gateway en zorg dat de ACP-bridge deze kan bereiken.
2. Richt `acpx openclaw` op `openclaw acp`.
3. Stel de OpenClaw-sessiesleutel in die de codeeragent moet gebruiken.

Voorbeelden:

```bash
# Eenmalige aanvraag voor je standaard OpenClaw ACP-sessie
acpx openclaw exec "Vat de status van de actieve OpenClaw-sessie samen."

# Permanente benoemde sessie voor vervolgbeurten
acpx openclaw sessions ensure --name codex-bridge
acpx openclaw -s codex-bridge --cwd /path/to/repo \
  "Vraag mijn OpenClaw-werkagent om recente context die relevant is voor deze repository."
```

Als je wilt dat `acpx openclaw` elke keer een specifieke Gateway en sessiesleutel gebruikt, overschrijf je de agentopdracht `openclaw` in `~/.acpx/config.json`:

```json
{
  "agents": {
    "openclaw": {
      "command": "env OPENCLAW_HIDE_BANNER=1 OPENCLAW_SUPPRESS_NOTES=1 openclaw acp --url ws://127.0.0.1:18789 --token-file ~/.openclaw/gateway.token --session agent:main:main"
    }
  }
}
```

Gebruik voor een repositorylokale OpenClaw-check-out het rechtstreekse CLI-ingangspunt in plaats van de ontwikkelrunner, zodat de ACP-stream schoon blijft:

```bash
env OPENCLAW_HIDE_BANNER=1 OPENCLAW_SUPPRESS_NOTES=1 node openclaw.mjs acp ...
```

Dit is de eenvoudigste manier om Codex, Claude Code of een andere ACP-compatibele client contextuele informatie bij een OpenClaw-agent te laten ophalen zonder een terminal uit te lezen.

## Zed-editor instellen

Voeg een aangepaste ACP-agent toe in `~/.config/zed/settings.json` (of gebruik de instellingeninterface van Zed):

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": ["acp"],
      "env": {}
    }
  }
}
```

Om een specifieke Gateway of agent te benaderen:

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": [
        "acp",
        "--url",
        "wss://gateway-host:18789",
        "--token",
        "<token>",
        "--session",
        "agent:design:main"
      ],
      "env": {}
    }
  }
}
```

Open in Zed het Agent-paneel en selecteer "OpenClaw ACP" om een thread te starten.

## Sessietoewijzing

Standaard krijgen ACP-bridgesessies een geïsoleerde Gateway-sessiesleutel met het voorvoegsel `acp-bridge:`. Deze bridgesessies met normale modellen zijn synthetisch en tijdelijk: verouderde vermeldingen kunnen worden opgeschoond en ze worden niet behandeld als beschermde oppervlakken voor menselijke gesprekken. Geef een sessiesleutel of label door om een bekende sessie opnieuw te gebruiken:

- `--session <key>`: gebruik een specifieke Gateway-sessiesleutel.
- `--session-label <label>`: zoek een bestaande sessie op label.
- `--reset-session`: maak een nieuwe sessie-id voor die sleutel (dezelfde sleutel, nieuw transcript).

Als je ACP-client metadata ondersteunt, kun je dit per sessie overschrijven:

```json
{
  "_meta": {
    "sessionKey": "agent:main:main",
    "sessionLabel": "support inbox",
    "resetSession": true
  }
}
```

Lees meer over sessiesleutels op [/concepten/sessie](/nl/concepts/session).

## Opties

- `--url <url>`: Gateway-WebSocket-URL (standaard `gateway.remote.url` indien geconfigureerd).
- `--token <token>`: authenticatietoken voor de Gateway.
- `--token-file <path>`: lees het authenticatietoken voor de Gateway uit een bestand.
- `--password <password>`: authenticatiewachtwoord voor de Gateway.
- `--password-file <path>`: lees het authenticatiewachtwoord voor de Gateway uit een bestand.
- `--session <key>`: standaard sessiesleutel.
- `--session-label <label>`: standaard op te zoeken sessielabel.
- `--require-existing`: misluk als de sessiesleutel of het sessielabel niet bestaat.
- `--reset-session`: stel de sessiesleutel opnieuw in vóór het eerste gebruik.
- `--no-prefix-cwd`: voeg de werkmap niet als voorvoegsel aan prompts toe.
- `--provenance <off|meta|meta+receipt>`: neem ACP-herkomstmetadata of ontvangstbewijzen op.
- `--verbose, -v`: uitgebreide logboekregistratie naar stderr.

Beveiligingsopmerking:

- `--token` en `--password` kunnen op sommige systemen zichtbaar zijn in lokale proceslijsten. Geef de voorkeur aan `--token-file`/`--password-file` of omgevingsvariabelen (`OPENCLAW_GATEWAY_TOKEN`, `OPENCLAW_GATEWAY_PASSWORD`).
- Het bepalen van Gateway-authenticatie volgt het gedeelde contract dat andere Gateway-clients gebruiken:
  - lokale modus: omgeving (`OPENCLAW_GATEWAY_*`) en daarna `gateway.auth.*`, met terugval op `gateway.remote.*` alleen wanneer `gateway.auth.*` niet is ingesteld (een geconfigureerde maar niet opgeloste lokale SecretRef sluit bij fouten af in plaats van stilzwijgend terug te vallen)
  - externe modus: `gateway.remote.*` met terugval op omgeving/configuratie volgens de voorrangsregels voor externe modus
  - `--url` kan veilig worden overschreven en hergebruikt geen impliciete configuratie- of omgevingsreferenties; geef expliciete `--token`/`--password` door (of de bestandsvarianten)

### Opties voor `acp client`

- `--cwd <dir>`: werkmap voor de ACP-sessie.
- `--server <command>`: ACP-serveropdracht (standaard: `openclaw`).
- `--server-args <args...>`: extra argumenten die aan de ACP-server worden doorgegeven.
- `--server-verbose`: schakel uitgebreide logboekregistratie op de ACP-server in.
- `--verbose, -v`: uitgebreide clientlogboekregistratie.
- `openclaw acp client` stelt `OPENCLAW_SHELL=acp-client` in voor het gestarte bridgeproces, wat kan worden gebruikt voor contextspecifieke shell-/profielregels.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [ACP-agents](/nl/tools/acp-agents)
