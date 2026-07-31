---
read_when:
    - iOS-/watchOS-/Android-nodes koppelen aan een Gateway
    - Node-canvas/camera gebruiken voor agentcontext
    - Nieuwe Node-opdrachten of CLI-helpers toevoegen
summary: 'Nodes: koppelen, mogelijkheden, machtigingen en CLI-hulpprogramma''s voor canvas/camera/scherm/apparaat/meldingen/systeem'
title: Nodes
x-i18n:
    generated_at: "2026-07-27T05:51:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b4f7c80491d713777e1ba5b8f55c88bd9fa48be602b504e6ac6ba00cd12a4313
    source_path: nodes/index.md
    workflow: 16
---

Een **node** is een begeleidend apparaat (macOS/iOS/watchOS/Android/headless) dat met `role: "node"` verbinding maakt met de Gateway en via `node.invoke` een commandosurface beschikbaar stelt (bijv. `canvas.*`, `camera.*`, `device.*`, `notifications.*`, `system.*`). De meeste nodes gebruiken de Gateway-WebSocket op de operatorpoort. De optionele directe Apple Watch-node gebruikt ondertekende HTTPS-polling op diezelfde poort, omdat watchOS generieke low-level-netwerken voor gewone apps blokkeert. Protocoldetails: [Gateway-protocol](/nl/gateway/protocol).

Verouderd transport: [Bridge-protocol](/nl/gateway/bridge-protocol) (TCP JSONL; alleen historisch voor huidige nodes).

macOS kan ook in **nodemodus** worden uitgevoerd: de menubalk-app maakt als één node verbinding met de
WS-server van de Gateway (zodat `openclaw nodes …` op deze Mac werkt). De app
voegt native commando's voor Canvas, camera, scherm, meldingen en computerbesturing
toe aan dezelfde commandosurface van de node-host die door `openclaw node run` wordt gebruikt. Start geen
tweede CLI-node op die Mac; de app voert de bijbehorende CLI-node-hostruntime uit als
interne worker en blijft de enige Gateway-verbinding en node-identiteit.

Nodes zijn **randapparaten**, geen gateways: ze voeren de Gateway-service niet uit en kanaalberichten (Telegram, WhatsApp, enz.) komen op de Gateway binnen, niet op nodes.

Draaiboek voor probleemoplossing: [/nodes/troubleshooting](/nl/nodes/troubleshooting)

## Koppelen + status

Nodes gebruiken **apparaatkoppeling**. Een node presenteert tijdens het verbinden een ondertekende apparaatidentiteit; de Gateway maakt een apparaatkoppelingsverzoek voor `role: node`. Keur dit goed via de apparaten-CLI (of UI). De directe Apple Watch-configuratie gebruikt een door een beheerder aangemaakte, kortlevende configuratiecode die alleen voor nodes geldt om de vaste commandosurface met laag risico goed te keuren; latere uitbreiding van mogelijkheden vereist nog steeds normale goedkeuring.

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
```

Openstaande koppelingsverzoeken verlopen 5 minuten na de laatste nieuwe poging van het apparaat — een apparaat dat verbinding blijft maken, houdt zijn ene openstaande verzoek (en `requestId`) actief in plaats van elke paar minuten een nieuwe prompt aan te maken; zie [Nodekoppeling](/nl/gateway/pairing) voor de volledige levenscyclus van verzoek en goedkeuring. Als een node het opnieuw probeert met gewijzigde authenticatiegegevens (rol/scopes/openbare sleutel), wordt het eerdere openstaande verzoek vervangen en wordt een nieuwe `requestId` aangemaakt — clients krijgen een `device.pair.resolved`-event voor het vervangen verzoek en je moet `openclaw devices list` opnieuw uitvoeren voordat je het goedkeurt.

- `nodes status` markeert een node als **gekoppeld** wanneer de apparaatkoppelingsrol `node` bevat.
- Een verbonden native Mac kan zich aanmelden voor samengevoegde activiteit van fysieke invoer via
  **Settings -> Permissions -> Active computer detection**. Toegankelijkheid is
  ook vereist. De Gateway markeert de meest recent actieve in aanmerking komende Mac als
  `active`, geeft de agent een stabiele node-id-hint en routeert meldingen over
  nodeverbindingen daarheen vóór een vertraagde fallback. Zie
  [Aanwezigheid van actieve computer](/nl/nodes/presence) voor configuratie, privacy, timing en
  probleemoplossing.
- De apparaatkoppelingsrecord is het duurzame contract voor goedgekeurde rollen. Tokenrotatie blijft binnen dat contract; hiermee kan een gekoppelde node niet worden opgewaardeerd naar een rol die nooit door de koppelingsgoedkeuring is verleend.
- `node.pair.*` (CLI: `openclaw nodes pending/approve/reject/remove/rename`) is een afzonderlijke, door de Gateway beheerde opslag voor nodekoppelingen die de goedgekeurde commando-/mogelijkhedensurface van de node bijhoudt wanneer opnieuw verbinding wordt gemaakt. Deze bepaalt **niet** de transportauthenticatie — dat doet apparaatkoppeling.
- `openclaw nodes remove --node <id|name|ip>` verwijdert een nodekoppeling. Voor een node met een apparaat als basis trekt dit de `node`-rol van het apparaat in de opslag voor gekoppelde apparaten in en verbreekt het de noderolsessies van dat apparaat: een apparaat met meerdere rollen behoudt zijn rij en verliest alleen de `node`-rol, terwijl de rij van een apparaat dat alleen een node is, wordt verwijderd. Ook wordt elke overeenkomende vermelding uit de afzonderlijke opslag voor nodekoppelingen gewist. `operator.pairing` kan niet-operatornoderijen op andere apparaten verwijderen; een aanroeper met een apparaattoken die zijn eigen noderol op een apparaat met meerdere rollen intrekt, heeft daarnaast `operator.admin` nodig.
- Het goedkeuringsbereik volgt de gedeclareerde commando's van het openstaande verzoek:
  - verzoek zonder commando's: `operator.pairing`
  - niet-exec-nodecommando's: `operator.pairing` + `operator.write`
  - `system.run` / `system.run.prepare` / `system.which`: `operator.pairing` + `operator.admin`

## Versieverschillen en upgradevolgorde

De Gateway-WebSocket accepteert geauthenticeerde nodeclients binnen een N-1-protocolvenster.
De huidige v4-Gateway accepteert daarom v3-nodes wanneer de verbinding zowel
`role: "node"` als `client.mode: "node"` declareert. Operator- en UI-sessies moeten
nog steeds het huidige protocol gebruiken.

Voer voor gefaseerde vlootupgrades eerst een upgrade van de Gateway uit en daarna van elke node.
Een N-1-node blijft zichtbaar en beheerbaar terwijl deze wordt geüpgraded; de Gateway
registreert `legacy node protocol accepted` met een upgradeadvies. Koppeling,
apparaatverificatie, allowlists voor commando's en exec-goedkeuringen blijven van toepassing.
Mogelijkheden en commando's die eigendom zijn van een Plugin, blijven verborgen totdat de node
naar het huidige protocol is geüpgraded. Nodes die ouder zijn dan N-1 vereisen een upgrade buiten de normale verbinding om
voordat ze opnieuw verbinding kunnen maken.

Het directe watchOS-HTTPS-transport vereist de huidige protocolversie; werk
de Watch-app samen met de Gateway bij voordat je de directe modus inschakelt.

## Externe node-host (system.run)

Gebruik een **node-host** wanneer je Gateway op de ene machine wordt uitgevoerd en je commando's op een andere wilt uitvoeren. Het model communiceert nog steeds met de **Gateway**; de Gateway stuurt `exec`-aanroepen door naar de **node-host** wanneer `host=node` is geselecteerd.

| Rol          | Verantwoordelijkheid                                                |
| ------------ | ------------------------------------------------------------------- |
| Gateway-host | Ontvangt berichten, voert het model uit en routeert toolaanroepen.  |
| Node-host    | Voert `system.run`/`system.which` uit op de nodemachine. |
| Goedkeuringen | Worden op de node-host afgedwongen via `~/.openclaw/exec-approvals.json`.         |

Opmerking over goedkeuring:

- Node-uitvoeringen waarvoor goedkeuring vereist is, worden aan de exacte verzoekcontext gebonden. Het exec-pad bereidt vóór de goedkeuring een canonieke `systemRunPlan` voor; na goedkeuring stuurt de Gateway dat opgeslagen plan door, niet eventuele later door de aanroeper gewijzigde commando-/cwd-/sessievelden, en valideert de werkmap opnieuw vóór de uitvoering.
- Voor directe bestandsuitvoeringen via shell/runtime bindt OpenClaw naar beste vermogen ook één concreet lokaal bestandsoperand en weigert de uitvoering als dat bestand vóór de uitvoering verandert.
- Als OpenClaw niet exact één concreet lokaal bestand voor een interpreter-/runtimecommando kan identificeren, wordt uitvoering waarvoor goedkeuring vereist is geweigerd in plaats van volledige runtimedekking voor te wenden. Gebruik sandboxing, afzonderlijke hosts of een expliciete vertrouwde allowlist/volledige workflow voor bredere interpretersemantiek.

### Een node-host starten (voorgrond)

Op de nodemachine:

```bash
openclaw node run --host <gateway-host> --port 18789 --display-name "Build Node"
```

`node run` accepteert ook `--context-path` (Gateway-WS-contextpad), `--tls`, `--tls-fingerprint <sha256>` en `--node-id` (overschrijft de verouderde clientinstance-id; hiermee wordt de koppeling niet gereset). Geef op macOS `--share-installed-apps` door om `device.apps` te adverteren; delen is standaard uitgeschakeld. Gebruik `--no-share-installed-apps` om een eerder opgeslagen aanmelding uit te schakelen.

### Externe Gateway via SSH-tunnel (loopback-binding)

Als de Gateway aan loopback is gebonden (`gateway.bind=loopback`, standaard in lokale modus), kunnen externe node-hosts niet rechtstreeks verbinding maken. Maak een SSH-tunnel en richt de node-host op het lokale uiteinde van de tunnel.

Voorbeeld (node-host -> Gateway-host):

```bash
# Terminal A (actief houden): stuur lokale 18790 door -> gateway 127.0.0.1:18789
ssh -N -L 18790:127.0.0.1:18789 user@gateway-host

# Terminal B: exporteer het gatewaytoken en maak verbinding via de tunnel
export OPENCLAW_GATEWAY_TOKEN="<gateway-token>"
openclaw node run --host 127.0.0.1 --port 18790 --display-name "Build Node"
```

Opmerkingen:

- `openclaw node run` ondersteunt authenticatie met een token of wachtwoord.
- Omgevingsvariabelen hebben de voorkeur: `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`.
- De configuratiefallback is `gateway.auth.token` / `gateway.auth.password`.
- In lokale modus negeert de node-host opzettelijk `gateway.remote.token` / `gateway.remote.password`.
- In externe modus komen `gateway.remote.token` / `gateway.remote.password` in aanmerking volgens de externe precedentieregels.
- Als actieve lokale `gateway.auth.*`-SecretRefs zijn geconfigureerd maar niet kunnen worden opgelost, wordt node-hosta authenticatie bij twijfel geweigerd.
- De oplossing van node-hosta authenticatie respecteert alleen `OPENCLAW_GATEWAY_*`-omgevingsvariabelen.

### Een node-host starten (service)

```bash
openclaw node install --host <gateway-host> --port 18789 --display-name "Build Node"
openclaw node start
openclaw node restart
```

`node install` accepteert ook `--context-path`, `--tls`, `--tls-fingerprint`, `--node-id` (alleen de verouderde clientinstance-id), `--share-installed-apps` / `--no-share-installed-apps`, `--runtime <node>` (standaard: node) en `--force` om opnieuw te installeren. `node status`, `node stop` en `node uninstall` zijn ook beschikbaar.

### Koppelen + naam geven

Op de Gateway-host:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Als de node het opnieuw probeert met gewijzigde authenticatiegegevens, voer je `openclaw devices list` opnieuw uit en keur je de huidige `requestId` goed.

Naamgevingsopties:

- `--display-name` op `openclaw node run` / `openclaw node install` (blijft opgeslagen in de gedeelde `node_host_config`-SQLite-rij naast de clientinstance-id en metagegevens van de Gateway-verbinding).
- `openclaw nodes rename --node <id|name|ip> --name "Build Node"` (Gateway-overschrijving).

### MCP-servers die door nodes worden gehost

Configureer MCP-servers in `openclaw.json` op de nodemachine, niet op de
Gateway:

```json5
{
  nodeHost: {
    mcp: {
      servers: {
        localDocs: {
          command: "npx",
          args: ["-y", "@modelcontextprotocol/server-filesystem", "/srv/docs"],
          toolFilter: {
            include: ["read_*", "search"],
          },
        },
        internalApi: {
          url: "https://mcp.internal.example/mcp",
          transport: "streamable-http",
          headers: {
            Authorization: "Bearer ${INTERNAL_MCP_TOKEN}",
          },
        },
      },
    },
  },
}
```

De headless node-host start deze servers, vermeldt hun tools en publiceert
de descriptors nadat verbinding is gemaakt. Toolaanroepen keren via
`mcp.tools.call.v1` terug naar die node; de Gateway heeft geen overeenkomende MCP-configuratie of JS-
plugin nodig. OAuth-MCP-servers worden niet ondersteund door dit door nodes gehoste v1-pad.

Huidige node-hosts declareren de ingebouwde commandofamilie `mcp.tools.call.v1` tijdens
hun eerste koppeling, zelfs wanneer geen MCP-server is geconfigureerd. Een node die met een
oudere versie van OpenClaw is gekoppeld, kan een eenmalige upgrade van de commandosurface aanvragen nadat de
node-host is bijgewerkt. Voor het toevoegen, verwijderen of filteren van servers daarna is
opnieuw koppelen niet nodig, omdat de goedgekeurde commandofamilie ongewijzigd blijft. Start
`openclaw node run` of `openclaw node restart` opnieuw om wijzigingen in de MCP-configuratie van de node toe te passen;
de node-host bewaakt deze configuratie niet.

Gateway-operators kunnen alle voor agents zichtbare tools negeren die door gekoppelde nodes worden gepubliceerd,
waaronder MCP-tools die door nodes worden gehost, met
`gateway.nodes.pluginTools.enabled: false`. Exacte weigeringen van commando's, zoals
`gateway.nodes.commands.deny: ["mcp.tools.call.v1"]`, blokkeren ook de uitvoering.

### Skills die door nodes worden gehost

Installeer Skills in de actieve OpenClaw-skillsmap van de nodemachine,
standaard `~/.openclaw/skills`. `OPENCLAW_HOME`, `OPENCLAW_STATE_DIR` en
`OPENCLAW_CONFIG_PATH` verplaatsen dat actieve profiel. `OPENCLAW_STATE_DIR` heeft
voorrang voor Skills; anders bevindt `skills/` zich naast het pad dat door
`openclaw config file` wordt weergegeven. De headless nodehost publiceert geldige `SKILL.md`-bestanden
nadat deze verbinding heeft gemaakt, en de Gateway voegt ze alleen aan snapshots van agent-Skills toe zolang
die node verbonden blijft. De naam van elke Skills-map moet overeenkomen met het frontmatterveld
`name`, zodat de abstracte nodelocator naar één item verwijst zonder
nog een protocolveld toe te voegen.

De initiële koppeling voor de noderol keurt de publicatie van Skills goed. Voor het toevoegen, verwijderen of
wijzigen van Skills is geen nieuwe koppeling of wijziging van de Gateway-configuratie
nodig. Start `openclaw node run` of `openclaw node restart` opnieuw nadat je
Skills-bestanden van de node hebt gewijzigd; de nodehost bewaakt de Skills-map niet.

Op de node gehoste Skills-items identificeren hun node en bevatten hun uitvoeringslocatie.
Skills-bestanden, relatief verwezen paden en binaire bestanden blijven op die
node. De agent leest de geadverteerde locatie `node://.../SKILL.md` met de
normale tool `read`. `file_fetch` accepteert door de beheerder goedgekeurde absolute nodepaden,
geen locators van node-Skills; runtimes zonder de normale leestool kunnen in plaats daarvan
`cat SKILL.md` via `exec host=node node=<node-id>` uitvoeren met de geadverteerde
map `node://.../skills/<name>` als `workdir`. Bestanden en binaire bestanden waarnaar wordt verwezen,
gebruiken hetzelfde uitvoeringsdoel en dezelfde werkmap. De nodehost zet die locator om ten opzichte van
zijn actieve OpenClaw-statusmap, zodat relatieve paden op de node worden omgezet in plaats van
op de Gateway-machine. De publicerende node moet `system.run` hebben goedgekeurd,
en het uitvoeringsbeleid van de agent moet `host=node` toestaan; anders blijft de Skill
buiten de snapshot van die agent.

Stel `nodeHost.skills.enabled: false` op de node in om publicatie te stoppen. Gateway-
beheerders kunnen Skills van elke gekoppelde node negeren met
`gateway.nodes.allowSkills: false`.

### Identiteitsstatus van een headless node

De headless node bewaart drie afzonderlijke statusrecords in gedeeld SQLite:

- `~/.openclaw/state/openclaw.sqlite` (`node_host_config`): de clientinstantie-ID, weergavenaam en metagegevens van de Gateway-verbinding.
- `~/.openclaw/state/openclaw.sqlite` (`device_identities`, sleutel `primary`): het ondertekende sleutelpaar van het apparaat en de daarvan afgeleide cryptografische apparaat-ID.
- `~/.openclaw/state/openclaw.sqlite` (`device_auth_tokens`): authenticatietokens van gekoppelde apparaten, geïndexeerd op cryptografische apparaat-ID en rol.

Voor een ondertekende node gebruikt de Gateway de cryptografische apparaat-ID voor koppeling en
noderoutering. De clientinstantie-ID bestaat alleen uit verbindingsmetagegevens. Het wijzigen van
`--node-id` of migreren van een uitgefaseerde `node.json` stelt de koppeling daarom niet opnieuw in. Zie
[Identiteits- en koppelingsstatus](/nl/cli/node#identity-and-pairing-state) voor de
ondersteunde procedure voor intrekken en opnieuw koppelen en opmerkingen over upgrades.

Uitgefaseerde bestanden `identity/device.json` en `identity/device-auth.json` zijn
migratie-invoer die door Doctor wordt beheerd. Stop de nodehost en voer
`openclaw doctor --fix` uit; Doctor importeert en verifieert de rijen ervan in SQLite voordat
de oude bestanden worden verwijderd.

### De opdrachten aan de toestemmingslijst toevoegen

Uitvoeringsgoedkeuringen gelden **per nodehost**. Voeg items aan de toestemmingslijst toe vanaf de Gateway:

```bash
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/uname"
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/sw_vers"
```

Goedkeuringen bevinden zich op de nodehost in `~/.openclaw/exec-approvals.json`.

### Uitvoering op de node richten

Configureer de standaardwaarden (Gateway-configuratie):

```bash
openclaw config set tools.exec.host node
openclaw config set tools.exec.mode allowlist
openclaw config set tools.exec.node "<id-or-name>"
```

Of per sessie:

```text
/exec host=node security=allowlist node=<id-or-name>
```

Zodra dit is ingesteld, wordt elke aanroep van `exec` met `host=node` uitgevoerd op de nodehost (afhankelijk van de toestemmingslijst/goedkeuringen van de node).

`host=auto` kiest de node niet impliciet zelf, maar een expliciet verzoek per aanroep via `host=node` is toegestaan vanuit `auto`. Als je wilt dat uitvoering op de node de standaardinstelling voor de sessie is, stel dan `tools.exec.host=node` of `/exec host=node ...` expliciet in.

Gerelateerd:

- [CLI voor nodehosts](/nl/cli/node)
- [Uitvoeringstool](/nl/tools/exec)
- [Uitvoeringsgoedkeuringen](/nl/tools/exec-approvals)

### Lokale modelinferentie

Een desktop- of servernode kan modellen met chatmogelijkheden beschikbaar stellen vanaf een Ollama-server die op die node draait. Agents gebruiken de tool `node_inference` van de Ollama-plugin om geïnstalleerde modellen te detecteren en op afstand een begrensde prompt uit te voeren; de Gateway heeft geen directe netwerktoegang tot Ollama nodig. Zie [Nodelokale Ollama-inferentie](/nl/providers/ollama#node-local-inference) voor de configuratie, modelfiltering en opdrachten voor directe verificatie.

### Codex-sessies en transcripties

De officiële plugin `codex` kan niet-gearchiveerde Codex-sessies beschikbaar stellen op een
headless nodehost of native macOS-node. Catalogusregistratie is niet langer afhankelijk
van `supervision.enabled`; die optie regelt de supervisietools voor agents.
Stel `sessionCatalog.enabled: false` in de configuratie van de Codex-plugin in om de
beheerderscatalogus en catalogusopdrachten voor gekoppelde nodes uit te schakelen zonder de
provider of harness uit te schakelen.
De plugin moet nog steeds op beide computers actief zijn, en de node-instelling blijft
lokale toestemming: als je alleen de Gateway inschakelt, kan deze de Codex-status van een andere computer
niet lezen.

De node adverteert de geversioneerde alleen-lezenopdrachten
`codex.appServer.threads.list.v1` en
`codex.appServer.thread.turns.list.v1`. Een native nodehost waarop de
Codex CLI beschikbaar is, adverteert ook `codex.terminal.resume.v1`. Keur de upgrade van de nodekoppeling goed
wanneer die opdrachten voor het eerst verschijnen. De Gateway roept ze aan via het
normale nodebeleid van de plugin en isoleert storingen per host.

Rijen van gekoppelde nodes verschijnen als een groep **Codex** in de normale sessiezijbalk.
Binnen elke host worden rijen standaard gegroepeerd op projectmap; een werkmap
onder `.claude/worktrees/<name>` wordt samengevoegd met de oorspronkelijke repository, en projectgroepen
kunnen net als andere secties in de zijbalk worden ingeklapt. Gebruik het mappictogram in de cataloguskop
om de projectgroepen af te vlakken of te herstellen. Dezelfde groepering geldt voor
de catalogus met Claude-sessies.
Als je een rij selecteert, wordt standaard het normale Chat-deelvenster geopend en wordt het opgeslagen transcript
gelezen via begrensde, met een cursor gepagineerde
aanroepen van `thread/turns/list` met volledige itemprojectie. Gebruik het rijmenu, de koptekst van de viewer of de voorkeur **Codex/Claude-sessies openen in** om `codex resume <thread-id>` te starten in de beheerdersterminal op de computer die eigenaar is van de sessie. Het terminalpad van de gekoppelde node is een PTY-relay op de toestemmingslijst die door de Codex-plugin wordt beheerd, geen willekeurige uitvoering van nodeopdrachten.

De relay biedt niet de volledige contracten voor voortzetting van de OpenClaw-harness en archiefeigendom. **Doorgaan** en **Archiveren** zijn daarom niet beschikbaar voor externe rijen. Op de Gateway-computer kunnen opgeslagen en inactieve
rijen een afzonderlijke, aan een model gebonden Chat-vertakking starten. Beide kunnen alleen worden gearchiveerd
nadat de beheerder heeft bevestigd dat geen andere Codex-client deze gebruikt; de actuele
activiteit van een opgeslagen rij blijft onbekend. Actieve rijen kunnen niet worden vertakt of gearchiveerd.

Zie [Toezicht houden op Codex-sessies](/plugins/codex-supervision) voor configuratie,
paginering, lokale voortzetting en de beveiligingsgrens voor metagegevens.

### Claude-sessies en transcripties

De meegeleverde plugin `anthropic` detecteert standaard niet-gearchiveerde Claude CLI- en Claude
Desktop-sessies op de Gateway en gekoppelde nodes. Stel
`plugins.entries.anthropic.config.sessionCatalog.enabled: false` in om de
beheerderscatalogus en catalogusopdrachten voor gekoppelde nodes uit te schakelen zonder Anthropic-
modellen of de Claude CLI-backend uit te schakelen.
Een externe macOS-appnode adverteert
`anthropic.claude.sessions.list.v1` en `anthropic.claude.sessions.read.v1`
wanneer de Anthropic-plugin is ingeschakeld en `~/.claude/projects/` bestaat. Keur
de upgrade van de nodekoppeling goed wanneer die opdrachten voor het eerst verschijnen.

Een native nodehost waarop de Claude CLI beschikbaar is, adverteert ook
`anthropic.claude.terminal.resume.v1`. Geschikte CLI- en Desktop-rijen kunnen
`claude --resume <session-id>` openen in de beheerdersterminal op hun eigen host.
Hiermee wordt de native sessie overgenomen; anders dan bij overname door OpenClaw wordt
de Claude-sessie niet eerst gevorkt.

De catalogus combineert geldige projectindexrecords van Claude CLI met een begrensde
fallback voor metagegevens van niet-geïndexeerde JSONL-transcripties. Die fallback herkent
gelijktijdige, interactieve sessies die geen zijketen zijn (`cli`) en headless Agent SDK CLI-
sessies (`sdk-cli`). De lokale metagegevens van Claude Desktop leveren Desktop-titels en de archiefstatus.
Desktop-metagegevens krijgen voorrang wanneer beide bronnen naar dezelfde Claude Code-
sessie-ID verwijzen; transcripties die alleen van de CLI afkomstig zijn, blijven zichtbaar omdat de CLI geen archiefvlag
heeft. Voor het lezen van transcripties worden ondoorzichtige
byte-offsetcursors en begrensde achterwaartse bestandslezingen gebruikt, zodat bij het selecteren van een grote
sessie of het laden van een oudere pagina niet de volledige JSONL-geschiedenis in één
Gateway-respons wordt ingelezen.

De opdrachten voor weergeven en lezen zijn alleen-lezen. Ze stellen catalogusmetagegevens en transcriptie-
inhoud alleen via de generieke methoden `sessions.catalog.list` en
`sessions.catalog.read` beschikbaar aan een geauthenticeerde beheerdersverbinding met
`operator.write`. Een Gateway-lokale Claude CLI-rij kan vanuit het normale
Chat-invoerveld worden overgenomen: OpenClaw importeert begrensde zichtbare geschiedenis, hervat met
`--fork-session` bij de eerste beurt en laat het brontranscript ongewijzigd.

Een headless nodehost kan dezelfde voortzettingsflow inschakelen:

```json5
{
  nodeHost: {
    agentRuns: {
      claude: { enabled: true },
    },
  },
}
```

De node adverteert `agent.cli.claude.run.v1` alleen wanneer deze nodelokale instelling
is ingeschakeld en het uitvoerbare bestand `claude` op die node kan worden gevonden. De Gateway kan
dit niet op afstand inschakelen. De opdracht valt ook onder het bestaande beleid voor
uitvoeringsgoedkeuringen van de node. Wanneer alle drie de Claude-opdrachten worden geadverteerd en toegestaan door
het nodeopdrachtenbeleid van de Gateway, kan een Claude CLI-
rij op die node worden voortgezet: OpenClaw importeert begrensde geschiedenis, koppelt
de overgenomen sessie aan de node en de door de catalogus gemelde werkmap ervan, en
voert daar elke eenmalige beurt van `claude -p` uit. De eerste beurt gebruikt nog steeds
`--fork-session`, waardoor het brontranscript behouden blijft.

Op de node uitgevoerde beurten gebruiken de standaardwaarden van Claude op de node. In v1 ontvangen ze niet de
Gateway-loopback-MCP-configuratie of Gateway-Skills-plugin, kunnen ze niet opnieuw worden geïnitialiseerd vanuit een
Gateway-transcript en weigeren ze bijlagen en afbeeldingen. Claude Desktop-rijen en
nodes die de uitvoeringsopdracht niet adverteren, blijven alleen-lezen. De macOS-appnode
adverteert deze opdracht nog niet, dus blijven de rijen ervan alleen-lezen.

Zie [Anthropic: Claude-sessies op meerdere computers](/nl/providers/anthropic#claude-sessions-across-computers)
voor het gedrag van de Control UI en de opslagbronnen.

### OpenCode- en Pi-sessies

De meegeleverde OpenCode- en ACPX-plugins detecteren ook alleen-lezen catalogi van native sessies
op de Gateway en gekoppelde nodes. Een node adverteert
`opencode.sessions.list.v1` / `opencode.sessions.read.v1` wanneer de CLI `opencode`
is geïnstalleerd, en `acpx.pi.sessions.list.v1` / `acpx.pi.sessions.read.v1`
wanneer de sessiemap van Pi bestaat. Keur de upgrade van de nodekoppeling goed wanneer nieuwe
opdrachten voor het eerst verschijnen. Wanneer de bijbehorende CLI ook beschikbaar is, voegt de node
`opencode.terminal.resume.v1` of `acpx.pi.terminal.resume.v1` toe; via het bestaande rijmenu
en de koptekst van de viewer kan de geselecteerde sessie vervolgens opnieuw in de eigen
terminal worden geopend met `opencode --session <id>` of `pi --session <id>`.

OpenCode leest via het officiële CLI-JSON-/exportoppervlak. Pi leest zijn
gedocumenteerde JSONL-sessieopslag, waaronder project- en globale sessiemappen `settings.json`,
plus overschrijvingen via `PI_CODING_AGENT_DIR` en
`PI_CODING_AGENT_SESSION_DIR`. Beide catalogi zijn standaard ingeschakeld;
schakel ze uit in de Web UI onder **Config > Plugins**.

Bij hervatten in de terminal worden de opgeslagen werkmap van de sessie en dezelfde
duplex PTY-relay op de toestemmingslijst gebruikt als bij Codex en Claude. Hiermee wordt geen willekeurige
uitvoering van nodeopdrachten beschikbaar gesteld.

### Terminalbestandsuploads

De Control UI kan bestanden naar een geopende terminal van een gekoppelde node slepen. De systeemeigen nodehost maakt de uitsluitend voor beheerders bestemde opdracht `terminal.upload` bekend; keur de koppelingsupgrade goed wanneer deze voor het eerst verschijnt. Elk bestand is beperkt tot 16 MiB, wordt klaargezet in een persoonlijke tijdelijke map op die node en wordt zonder uitvoering als een voor de shell geciteerd pad naar de terminal teruggestuurd.

Padinvoeging ondersteunt PowerShell, `cmd.exe` en herkende POSIX-shells (`sh`, Bash, Dash, Ash, Ksh, Zsh en Fish), waaronder Git Bash op Windows. Andere shelloverschrijvingen worden geweigerd omdat hun regels voor citeren niet veilig kunnen worden afgeleid; voer de nodehost binnen WSL uit voor systeemeigen WSL-paden. `cmd.exe`-paden die `%` of `!` bevatten, worden eveneens geweigerd omdat die shell deze tekens zelfs binnen dubbele aanhalingstekens uitbreidt.

## Opdrachten aanroepen

Laag niveau (onbewerkte RPC):

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command canvas.eval --params '{"javaScript":"location.href"}'
```

`nodes invoke` blokkeert `system.run` en `system.run.prepare`; die opdrachten worden alleen uitgevoerd via de tool `exec` met `host=node` (zie hierboven). Voor de gangbare workflows om de agent een MEDIA-bijlage te geven, bestaan helpers op hoger niveau (canvas, camera, scherm en locatie, hieronder).

Langlopende streamende nodeopdrachten gebruiken aanvullende `node.invoke.progress`-
gebeurtenissen. Elke gebeurtenis bevat de aanroep-ID, een op nul gebaseerd volgnummer en een
begrensd UTF-8-tekstfragment; de Gateway ordent fragmenten voordat deze aan
de aanroeper worden geleverd. De bestaande `node.invoke.result` blijft de enige afsluitende
respons. Streamende aanroepers kunnen een inactiviteitsdeadline instellen die begint bij de
eerste voortgangsgebeurtenis en na latere voortgang opnieuw wordt ingesteld, terwijl de
afzonderlijke harde time-out van de aanroep tijdens goedkeuring en uitvoering behouden blijft. Een resultaat, harde
time-out, inactiviteitstime-out en verbroken nodeverbinding verwijderen allemaal de wachtende streamstatus.
Annulering door de aanroeper genereert `node.invoke.cancel`; de nodehost
beëindigt vervolgens de bijbehorende procesboom. Bestaande verzoek-/responsopdrachten blijven ongewijzigd.

## Opdrachtbeleid

Nodeopdrachten moeten twee controles doorstaan voordat ze kunnen worden aangeroepen:

1. De node moet de opdracht declareren in zijn geverifieerde verbindingsmetadata (`connect.commands`).
2. De op platform en goedkeuring gebaseerde toestemmingslijst van de Gateway moet de gedeclareerde opdracht bevatten.

Standaardtoestemmingslijsten per platform (vóór Plugin-standaardwaarden en overschrijvingen met `commands.allow`/`commands.deny`):

| Platform | Standaard toegestane opdrachten                                                                                                                                                                                                                                                                                           |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| iOS      | `camera.list`, `location.get`, `device.info`, `device.status`, `contacts.search`, `calendar.events`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer`, `system.notify`                                                                                                                        |
| watchOS  | `device.info`, `device.status`, `system.notify`                                                                                                                                                                                                                                                                       |
| Android  | `camera.list`, `location.get`, `notifications.list`, `notifications.actions`, `system.notify`, `device.info`, `device.status`, `device.permissions`, `device.health`, `device.apps`, `contacts.search`, `calendar.events`, `callLog.search`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer` |
| macOS    | `camera.list`, `location.get`, `device.info`, `device.status`, `device.apps`, `contacts.search`, `calendar.events`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer`, `system.notify`                                                                                                         |
| Windows  | `camera.list`, `location.get`, `device.info`, `device.status`, `system.notify`                                                                                                                                                                                                                                        |
| Linux    | `system.notify` (nodehostopdrachten zoals `system.run` vereisen goedkeuring; zie hieronder)                                                                                                                                                                                                                                  |

Deze rijen beschrijven de bovengrens van het Gateway-beleid, niet de opdrachten die door elke node-app zijn geïmplementeerd. Een opdracht kan alleen worden gebruikt wanneer de verbonden node deze ook declareert. De huidige macOS-app declareert met name niet de apparaat- en persoonsgegevensgroepen die in de beleidsrij voor macOS staan.

`canvas.*`-opdrachten (`canvas.present`, `canvas.hide`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot`, `canvas.a2ui.*`) zijn een Plugin-standaardwaarde op iOS, Android, macOS, Windows, Linux en onbekende platforms. Linux-nodes declareren ze alleen wanneer de lokale Canvas-socket van de desktop-app aanwezig is. Alle Canvas-opdrachten zijn op iOS beperkt tot de voorgrond.

`talk.ptt.start`, `talk.ptt.stop`, `talk.ptt.cancel` en `talk.ptt.once` zijn standaard toegestaan voor elke node die de mogelijkheid `talk` bekendmaakt of `talk.*`-opdrachten declareert, ongeacht het platformlabel.

Opdrachten voor desktophosts (`system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `mcp.tools.call.v1` en `screen.snapshot` op macOS/Windows/Linux) maken geen deel uit van de statische tabel met standaardwaarden per platform hierboven. Ze worden beschikbaar zodra de beheerder een koppelingsverzoek goedkeurt waarin ze worden gedeclareerd. Daarna worden ze bij opnieuw verbinden meegenomen in de goedgekeurde opdrachtenset van de node.

Gevaarlijke opdrachten of opdrachten met grote gevolgen voor de privacy vereisen nog steeds expliciete aanmelding via `gateway.nodes.commands.allow`, zelfs als een node ze declareert: `camera.snap`, `camera.clip`, `screen.record`, `computer.act`, `contacts.add`, `calendar.add`, `reminders.add`, `health.summary`, `sms.send`, `sms.search`. `gateway.nodes.commands.deny` heeft altijd voorrang op standaardwaarden en extra vermeldingen in de toestemmingslijst. Zie [HealthKit-samenvattingen](/nl/platforms/ios-healthkit) voor de toestemmingscontrole op de iPhone en [Computergebruik](/nl/nodes/computer-use) voor de aanvullende controles voor mogelijkheden, toolbeleid, activering en platformuitvoering rond desktopinvoer.

Nodeopdrachten die eigendom zijn van een Plugin kunnen een Gateway-beleid voor nodeaanroepen toevoegen. Dat beleid wordt uitgevoerd na de controle van de toestemmingslijst en voordat de opdracht naar de node wordt doorgestuurd, zodat onbewerkte `node.invoke`, CLI-helpers en speciale agenttools dezelfde Plugin-machtigingsgrens delen. Gevaarlijke Plugin-nodeopdrachten vereisen nog steeds expliciete aanmelding via `gateway.nodes.commands.allow`.

Nadat een node zijn lijst met gedeclareerde opdrachten heeft gewijzigd, wijs je de oude apparaatkoppeling af en keur je het nieuwe verzoek goed, zodat de Gateway de bijgewerkte momentopname van opdrachten opslaat.

## Configuratie (`openclaw.json`)

Nodegerelateerde instellingen staan onder `gateway.nodes` en `tools.exec`:

```json5
{
  gateway: {
    nodes: {
      // Keur een eerste nodekoppeling vanaf vertrouwde netwerken automatisch goed (CIDR-lijst).
      // Uitgeschakeld wanneer niet ingesteld. Geldt alleen voor eerste role:node-verzoeken
      // zonder aangevraagde bereiken; upgrades worden niet automatisch goedgekeurd.
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
        // Via SSH geverifieerde automatische goedkeuring (standaard: ingeschakeld). Keurt een eerste
        // nodekoppeling goed bij een exacte overeenkomst van de apparaatsleutel die via SSH wordt teruggelezen.
        sshVerify: true,
      },
      // Vertrouw agentzichtbare Plugin-tools die door gekoppelde nodes worden gepubliceerd (standaard: true).
      pluginTools: {
        enabled: true,
      },
      // Meld je aan voor gevaarlijke nodeopdrachten/opdrachten met grote gevolgen voor de privacy (camera.snap enzovoort).
      commands: {
        allow: ["camera.snap", "screen.record"],
        // Blokkeer exacte opdrachtnamen, zelfs als ze in standaardwaarden of commands.allow staan.
        deny: ["camera.clip"],
      },
    },
  },
  tools: {
    exec: {
      // Standaardhost voor uitvoering: "node" leidt alle uitvoeringsaanroepen naar een gekoppelde node.
      host: "node",
      // Beveiligingsmodus voor uitvoering op een node: sta alleen goedgekeurde/op de toestemmingslijst geplaatste opdrachten toe.
      security: "allowlist",
      // Koppel uitvoering aan een specifieke node (id of naam). Laat weg om elke node toe te staan.
      node: "build-node",
    },
  },
}
```

Gebruik exacte namen van nodeopdrachten. `commands.deny` verwijdert een opdracht, zelfs wanneer een platformstandaard of vermelding in `commands.allow` deze anders zou toestaan. Gekoppelde nodes mogen standaard agentzichtbare beschrijvingen van Plugin-tools publiceren, maar de opdracht van elke beschrijving moet nog steeds deel uitmaken van het goedgekeurde opdrachtoppervlak van de node. Stel `gateway.nodes.pluginTools.enabled: false` in om al deze beschrijvingen te negeren. Zie de [Gateway-configuratiereferentie](/nl/gateway/configuration-reference#gateway) voor details over velden voor Gateway-nodekoppeling en opdrachtbeleid.

Overschrijving van de uitvoeringsnode per agent:

```json5
{
  agents: {
    list: [
      {
        id: "main",
        tools: { exec: { node: "build-node" } },
      },
    ],
  },
}
```

## Schermafbeeldingen (canvasmomentopnamen)

Als de node de Canvas (WebView) toont, retourneert `canvas.snapshot` `{ format, base64 }`.

CLI-helper (schrijft naar een tijdelijk bestand en drukt het opgeslagen pad af):

```bash
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format png
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format jpg --max-width 1200 --quality 0.9
```

### Canvas-bediening

```bash
openclaw nodes canvas present --node <idOrNameOrIp> --target https://example.com
openclaw nodes canvas hide --node <idOrNameOrIp>
openclaw nodes canvas navigate https://example.com --node <idOrNameOrIp>
openclaw nodes canvas eval --node <idOrNameOrIp> --js "document.title"
```

Opmerkingen:

- `canvas present` accepteert URL's of lokale bestandspaden (`--target`) op nodes die lokale paden ondersteunen, plus optioneel `--x/--y/--width/--height` voor positionering. Linux Canvas accepteert HTTP(S)-URL's of de meegeleverde A2UI-renderer.
- `canvas eval` accepteert inline-JS (`--js`) of een positioneel argument.

### A2UI (Canvas)

```bash
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --text "Hello"
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --jsonl ./payload.jsonl
openclaw nodes canvas a2ui reset --node <idOrNameOrIp>
```

Opmerkingen:

- Mobiele nodes en Linux-desktopnodes gebruiken een meegeleverde A2UI-pagina die eigendom is van de app voor rendering met actieondersteuning.
- Alleen A2UI v0.8 JSONL wordt ondersteund (v0.9/createSurface wordt geweigerd).
- iOS en Android renderen externe Gateway Canvas-pagina's, maar A2UI-knopacties worden alleen verzonden vanaf de meegeleverde A2UI-pagina die eigendom is van de app. Door de Gateway gehoste HTTP/HTTPS-A2UI-pagina's zijn op die mobiele clients alleen voor rendering.
- macOS kan acties verzenden vanaf de exacte, tot een mogelijkheid beperkte Gateway A2UI-pagina die door de app is geselecteerd. Andere HTTP/HTTPS-pagina's blijven alleen voor rendering.
- Linux verzendt acties alleen vanaf de meegeleverde A2UI-pagina. Andere HTTP/HTTPS-pagina's blijven alleen voor rendering en een headless Linux-node zonder de desktop-app maakt Canvas niet bekend.

## Foto's en video's (nodecamera)

Foto's (`jpg`):

```bash
openclaw nodes camera list --node <idOrNameOrIp>
openclaw nodes camera snap --node <idOrNameOrIp>            # standaard: beide camerarichtingen (2 MEDIA-regels)
openclaw nodes camera snap --node <idOrNameOrIp> --facing front
openclaw nodes camera snap --node <idOrNameOrIp> --device-id <id> --max-width 1200 --quality 0.9 --delay-ms 2000
```

Videoclips (`mp4`):

```bash
openclaw nodes camera clip --node <idOrNameOrIp> --duration 10s
openclaw nodes camera clip --node <idOrNameOrIp> --duration 3000 --no-audio
```

Opmerkingen:

- De Node moet zich **op de voorgrond** bevinden voor `canvas.*` en `camera.*` (aanroepen op de achtergrond retourneren `NODE_BACKGROUND_UNAVAILABLE`).
- Nodes begrenzen de clipduur om de base64-payload beheersbaar te houden (zie [Camera-opname](/nl/nodes/camera) voor de exacte limieten per platform). De agenttool `nodes` begrenst de aangevraagde `durationMs` bovendien op 300000 (5 minuten) voordat de aanroep wordt doorgestuurd; de Node zelf handhaaft de strengere limiet.
- Android vraagt waar mogelijk om machtigingen voor `CAMERA`/`RECORD_AUDIO`; geweigerde machtigingen mislukken met `*_PERMISSION_REQUIRED`.

## Schermopnamen (Nodes)

Ondersteunde Nodes bieden `screen.record` (mp4). Voorbeeld:

```bash
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10 --no-audio
```

Opmerkingen:

- De beschikbaarheid van `screen.record` is afhankelijk van het Node-platform.
- De agenttool `nodes` begrenst de aangevraagde `durationMs` op 300000 (5 minuten); de Node kan een strengere limiet handhaven om de geretourneerde payload te begrenzen.
- `--no-audio` schakelt microfoonopname uit op ondersteunde platforms.
- Gebruik `--screen <index>` om een beeldscherm te selecteren wanneer meerdere schermen beschikbaar zijn (0 = primair).

## Locatie (Nodes)

Nodes bieden `location.get` wanneer Locatie is ingeschakeld in de instellingen.

CLI-helper:

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

Opmerkingen:

- Locatie is **standaard uitgeschakeld**.
- "Altijd" vereist systeemmachtiging; ophalen op de achtergrond gebeurt naar beste vermogen.
- Het antwoord bevat breedte-/lengtegraad, nauwkeurigheid (meters) en tijdstempel.
- Volledige parameter-/antwoordstructuur en foutcodes: [Locatieopdracht](/nl/nodes/location-command).

## SMS (Android-Nodes)

Android-Nodes kunnen `sms.send` en `sms.search` bieden wanneer de gebruiker de machtiging **SMS** verleent en het apparaat telefonie ondersteunt. Beide opdrachten zijn standaard gevaarlijk: de Gateway-beheerder moet ze ook toevoegen aan `gateway.nodes.commands.allow` voordat ze kunnen worden aangeroepen (zie [Opdrachtbeleid](#command-policy)).

Meld je voor alleen-lezen zoeken in SMS expliciet aan in `openclaw.json`:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["sms.search"] },
    },
  },
}
```

Voeg `sms.send` alleen afzonderlijk toe wanneer de Node ook berichten moet kunnen verzenden. Android-machtiging en Gateway-opdrachtbevoegdheid staan los van elkaar; het verlenen van de telefoonmachtiging wijzigt het Gateway-beleid niet.

Aanroep op laag niveau:

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command sms.send --params '{"to":"+15555550123","message":"Hello from OpenClaw"}'
```

Opmerkingen:

- `sms.search` kan worden gedeclareerd voordat `READ_SMS` is verleend, zodat een aanroep een machtigingsdiagnose kan retourneren; voor het lezen van berichten blijft die Android-machtiging vereist.
- Apparaten met alleen wifi en zonder telefonie maken geen melding van `sms.send`.
- Een fout `requires explicit gateway.nodes.commands.allow opt-in` betekent dat de telefoon de opdracht heeft gedeclareerd, maar de Gateway-beheerder deze niet heeft geautoriseerd.

## Opdrachten voor apparaat- en persoonsgegevens

iOS- en Android-Nodes maken standaard melding van verschillende alleen-lezen gegevensopdrachten (zie de tabel [Opdrachtbeleid](#command-policy)); Android biedt daarnaast een grotere reeks die wordt beheerd via eigen instellingen in de app. Een TypeScript-Nodehost voor macOS of een headless Mac maakt alleen melding van `device.apps` nadat de beheerder het delen van geïnstalleerde apps inschakelt met `--share-installed-apps`.

Beschikbare reeksen:

- `device.status`, `device.info` — iOS, Android, Windows.
- `device.permissions`, `device.health` — alleen Android.
- `device.apps` — Android-, macOS- en headless Mac-Nodes. Android vereist dat het delen van geïnstalleerde apps is ingeschakeld in Instellingen en retourneert standaard apps die zichtbaar zijn in de launcher. TypeScript-Nodehosts houden delen standaard uitgeschakeld en accepteren `query`, `limit` en `includeSystem`; macOS-resultaten bevatten `label`, `bundleId`, `path` en `system`.
- `notifications.list`, `notifications.actions` — alleen Android.
- `photos.latest` — iOS, Android.
- `contacts.search` — iOS, Android (standaard alleen-lezen); `contacts.add` is gevaarlijk en vereist `gateway.nodes.commands.allow`.
- `calendar.events` — iOS, Android (standaard alleen-lezen); `calendar.add` is gevaarlijk en vereist `gateway.nodes.commands.allow`.
- `reminders.list` — iOS, Android (standaard alleen-lezen); `reminders.add` is gevaarlijk en vereist `gateway.nodes.commands.allow`.
- `callLog.search` — alleen Android.
- `motion.activity`, `motion.pedometer` — iOS, Android; afhankelijk van de beschikbare sensoren.

Voorbeeldaanroepen:

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command device.status --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command device.apps --params '{"limit":10}'
openclaw nodes invoke --node <idOrNameOrIp> --command notifications.list --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command photos.latest --params '{"limit":1}'
```

## Systeemopdrachten (Nodehost / Mac-Node)

De macOS-Node biedt `system.run`, `system.which`, `system.notify` en `system.execApprovals.get/set`. De headless Nodehost biedt `system.run.prepare`, `system.run`, `system.which` en `system.execApprovals.get/set`.

Voorbeelden:

```bash
openclaw nodes notify --node <idOrNameOrIp> --title "Ping" --body "Gateway gereed"
openclaw nodes invoke --node <idOrNameOrIp> --command system.which --params '{"bins":["git"]}'
```

Opmerkingen:

- `system.run` retourneert standaarduitvoer/standaardfoutuitvoer/afsluitcode in de payload.
- Shelluitvoering verloopt nu via de tool `exec` met `host=node`; `nodes` blijft het rechtstreekse RPC-oppervlak voor expliciete Node-opdrachten.
- `nodes invoke` biedt `system.run` of `system.run.prepare` niet; die blijven uitsluitend op het exec-pad.
- Het exec-pad bereidt vóór goedkeuring een canonieke `systemRunPlan` voor. Zodra goedkeuring is verleend, stuurt de Gateway dat opgeslagen plan door, niet opdracht-/cwd-/sessievelden die later door de aanroeper zijn gewijzigd.
- `system.notify` respecteert de status van de meldingsmachtiging in de macOS-app; ondersteunt `--priority <passive|active|timeSensitive>` en `--delivery <system|overlay|auto>`.
- Niet-herkende `platform`- / `deviceFamily`-metadata van Nodes gebruiken een conservatieve standaardtoelatingslijst die `system.run` en `system.which` uitsluit. Als je deze opdrachten bewust nodig hebt voor een onbekend platform, voeg je ze expliciet toe via `gateway.nodes.commands.allow`.
- `system.run` ondersteunt `--cwd`, `--env KEY=VAL`, `--command-timeout` en `--needs-screen-recording`.
- Voor shellwrappers (`bash|sh|zsh ... -c/-lc`) worden `--env`-waarden met aanvraagbereik beperkt tot een expliciete toelatingslijst (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`).
- Bij beslissingen voor altijd toestaan in de toelatingslijstmodus worden voor bekende dispatchwrappers (`env`, `flock`, `nice`, `nohup`, `stdbuf`, `timeout`) de paden van de interne uitvoerbare bestanden opgeslagen in plaats van de wrapperpaden. Als uitpakken niet veilig is, wordt niet automatisch een vermelding in de toelatingslijst opgeslagen.
- Op Windows-Nodehosts in de toelatingslijstmodus vereisen shellwrapperuitvoeringen via `cmd.exe /c` goedkeuring (alleen een vermelding in de toelatingslijst staat de wrappervorm niet automatisch toe).
- Nodehosts negeren overschrijvingen van `PATH` in `--env` en verwijderen een grote, onderhouden verzameling opstartvariabelen voor interpreters/shells (bijvoorbeeld `NODE_OPTIONS`, `PYTHONPATH`, `BASH_ENV`, `DYLD_*`, `LD_*`) voordat een opdracht wordt uitgevoerd. Als je extra PATH-vermeldingen nodig hebt, configureer je de serviceomgeving van de Nodehost (of installeer je tools op standaardlocaties) in plaats van `PATH` door te geven via `--env`.
- In de macOS-Nodemodus wordt `system.run` beheerd door exec-goedkeuringen in de macOS-app (Settings → Exec approvals). Vragen/toelatingslijst/volledig werken hetzelfde als bij de headless Nodehost; geweigerde prompts retourneren `SYSTEM_RUN_DENIED`.
- Op de headless Nodehost wordt `system.run` beheerd door exec-goedkeuringen (`~/.openclaw/exec-approvals.json`); specifiek voor macOS vind je de omgevingsvariabelen voor routering naar de exec-host hieronder bij [Headless Nodehost](#headless-node-host-cross-platform).

## Exec aan een Node koppelen

Wanneer meerdere Nodes beschikbaar zijn, kun je exec aan een specifieke Node koppelen. Hiermee stel je de standaard-Node in voor `exec host=node` (en dit kan per agent worden overschreven).

Algemene standaardinstelling:

```bash
openclaw config set tools.exec.node "node-id-or-name"
```

Overschrijving per agent:

```bash
openclaw config get agents.entries
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
```

Hef de instelling op om elke Node toe te staan:

```bash
openclaw config unset tools.exec.node
openclaw config unset 'agents.entries.main.tools.exec.node'
```

## Machtigingenoverzicht

Nodes kunnen een `permissions`-overzicht bevatten in `node.list` / `node.describe`, met de machtigingsnaam als sleutel (bijvoorbeeld `screenRecording`, `accessibility`, `location`) en booleaanse waarden (`true` = verleend).

## Headless Nodehost (platformonafhankelijk)

OpenClaw kan een **headless Nodehost** (zonder UI) uitvoeren die verbinding maakt met de Gateway-WebSocket en `system.run` / `system.which` biedt. Dit is nuttig op Linux/Windows of om naast een server een minimale Node uit te voeren.

Start deze:

```bash
openclaw node run --host <gateway-host> --port 18789
```

Opmerkingen:

- Koppeling blijft vereist (de Gateway toont een prompt om een apparaat te koppelen).
- Metadata van de clientinstantie, ondertekende apparaatidentiteit en koppelingsauthenticatie gebruiken afzonderlijke statusrecords; zie [Headless identiteitsstatus](#headless-identity-state).
- Exec-goedkeuringen worden lokaal afgedwongen via `~/.openclaw/exec-approvals.json` (zie [Exec-goedkeuringen](/nl/tools/exec-approvals)).
- Op macOS voert de headless Nodehost `system.run` standaard lokaal uit. Stel `OPENCLAW_NODE_EXEC_HOST=app` in om `system.run` via de exec-host van de begeleidende app te routeren; voeg `OPENCLAW_NODE_EXEC_FALLBACK=0` toe om de apphost te vereisen en gesloten te falen als deze niet beschikbaar is.
- Voeg `--tls` / `--tls-fingerprint` toe wanneer de Gateway-WS TLS gebruikt.

## Mac-Nodemodus

- De macOS-menubalkapp maakt als Node verbinding met de Gateway-WS-server (zodat `openclaw nodes …` tegen deze Mac werkt).
- In de externe modus opent de app een SSH-tunnel voor de Gateway-poort en maakt verbinding met `localhost`.
