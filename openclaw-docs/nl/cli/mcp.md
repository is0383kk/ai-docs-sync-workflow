---
read_when:
    - Codex, Claude Code of een andere MCP-client verbinden met door OpenClaw ondersteunde kanalen
    - '`openclaw mcp serve` uitvoeren'
    - Door OpenClaw opgeslagen MCP-serverdefinities beheren
sidebarTitle: MCP
summary: Stel OpenClaw-kanaalgesprekken beschikbaar via MCP en beheer opgeslagen MCP-serverdefinities
title: MCP
x-i18n:
    generated_at: "2026-07-27T05:05:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ee6146bbc0181d10997336094d1bd693d0afb0985f1febef8e8c6b0d6e656cf9
    source_path: cli/mcp.md
    workflow: 16
---

`openclaw mcp` heeft twee taken:

- OpenClaw uitvoeren als een MCP-server met `openclaw mcp serve`
- door OpenClaw beheerde definities van uitgaande MCP-servers beheren met `list`, `show`, `status`, `doctor`, `probe`, `add`, `set`, `configure`, `tools`, `login`, `logout`, `reload` en `unset`

`serve` is OpenClaw dat als een MCP-server fungeert. Bij de andere subopdrachten fungeert OpenClaw als een clientregister voor MCP-servers die de eigen runtimes later kunnen gebruiken.

<Note>
  `list`, `show`, `set` en `unset` lezen en schrijven alleen door OpenClaw beheerde `mcp.servers`-vermeldingen in de OpenClaw-configuratie. Ze bevatten geen mcporter-servers uit `config/mcporter.json`; gebruik `mcporter list` voor dat register.
</Note>

Gebruik [`openclaw acp`](/nl/cli/acp) wanneer OpenClaw zelf een sessie met een programmeerharnas moet hosten en die runtime via ACP moet routeren.

## Kies het juiste MCP-pad

| Doel                                                                | Gebruik                                                                  | Waarom                                                                                                             |
| ------------------------------------------------------------------- | -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Een externe MCP-client OpenClaw-kanaalgesprekken laten lezen/verzenden | `openclaw mcp serve`                                                 | OpenClaw is de MCP-server en stelt door de Gateway ondersteunde gesprekken beschikbaar via stdio.                                 |
| MCP-servers van derden opslaan voor door OpenClaw beheerde agentuitvoeringen        | `openclaw mcp add`, `set`, `configure`, `tools`, `login`             | OpenClaw is het clientregister voor MCP en projecteert die servers later naar geschikte runtimes.               |
| Een opgeslagen server controleren zonder een agentbeurt uit te voeren                  | `openclaw mcp status`, `doctor`, `probe`                             | `status` en `doctor` inspecteren de configuratie; `probe` opent een live MCP-verbinding en vermeldt de mogelijkheden.               |
| MCP-configuratie vanuit een browser bewerken                                      | Control UI `/settings/mcp` (alias `/mcp`)                            | De pagina toont de inventaris, activering, OAuth-/filtersamenvattingen, opdrachttips en een editor met het bereik `mcp`.         |
| Codex app-server een afgebakende native MCP-server geven                    | `mcp.servers.<name>.codex`                                           | Het blok `codex` is alleen van invloed op de threadprojectie van Codex app-server en wordt verwijderd voordat de native configuratie wordt overgedragen. |
| Door ACP gehoste harnassessies uitvoeren                                     | [`openclaw acp`](/nl/cli/acp) en [ACP-agenten](/nl/tools/acp-agents-setup) | De ACP-brugmodus accepteert geen injectie van MCP-servers per sessie; configureer in plaats daarvan Gateway-/Plugin-bruggen.     |

<Tip>
Als je niet zeker weet welk pad je nodig hebt, begin dan met `openclaw mcp status --verbose`. Dit toont wat OpenClaw heeft opgeslagen zonder MCP-servers te starten.
</Tip>

## OpenClaw als MCP-server

Dit is het pad `openclaw mcp serve`.

### Wanneer serve te gebruiken

Gebruik `openclaw mcp serve` wanneer:

- Codex, Claude Code of een andere MCP-client rechtstreeks moet communiceren met door OpenClaw ondersteunde kanaalgesprekken
- je al een lokale of externe OpenClaw Gateway met gerouteerde sessies hebt
- je één MCP-server wilt die met alle kanaalbackends van OpenClaw werkt, in plaats van afzonderlijke bruggen per kanaal uit te voeren

Gebruik in plaats daarvan [`openclaw acp`](/nl/cli/acp) wanneer OpenClaw zelf de programmeerruntime moet hosten en de agentsessie binnen OpenClaw moet houden.

### Hoe het werkt

`openclaw mcp serve` start een stdio-MCP-server. De MCP-client is eigenaar van dat proces. Zolang de client de stdio-sessie openhoudt, maakt de brug via WebSocket verbinding met een lokale of externe OpenClaw Gateway en stelt deze gerouteerde kanaalgesprekken beschikbaar via MCP.

<Steps>
  <Step title="Client start de brug">
    De MCP-client start `openclaw mcp serve`.
  </Step>
  <Step title="Brug maakt verbinding met Gateway">
    De brug maakt via WebSocket verbinding met de OpenClaw Gateway.
  </Step>
  <Step title="Sessies worden MCP-gesprekken">
    Gerouteerde sessies worden MCP-gesprekken en hulpmiddelen voor transcripties/geschiedenis.
  </Step>
  <Step title="Livegebeurtenissen worden in de wachtrij geplaatst">
    Livegebeurtenissen worden in het geheugen in de wachtrij geplaatst zolang de brug verbonden is.
  </Step>
  <Step title="Optionele Claude-push">
    Als de Claude-kanaalmodus is ingeschakeld, kan dezelfde sessie ook Claude-specifieke pushmeldingen ontvangen.
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Belangrijk gedrag">
    - de status van de livewachtrij begint wanneer de brug verbinding maakt
    - oudere transcriptiegeschiedenis wordt gelezen met `messages_read`
    - Claude-pushmeldingen bestaan alleen zolang de MCP-sessie actief is
    - wanneer de client de verbinding verbreekt, wordt de brug afgesloten en verdwijnt de livewachtrij
    - eenmalige agentingangspunten zoals `openclaw agent` en `openclaw infer model run` beëindigen alle gebundelde MCP-runtimes die ze openen zodra het antwoord is voltooid, zodat herhaalde gescripte uitvoeringen geen onderliggende stdio-MCP-processen opstapelen
    - door OpenClaw gestarte stdio-MCP-servers (gebundeld of door de gebruiker geconfigureerd) worden bij afsluiting als procesboom beëindigd, zodat door de server gestarte onderliggende subprocessen niet blijven bestaan nadat de bovenliggende stdio-client is afgesloten
    - door een sessie te verwijderen of opnieuw in te stellen, worden de MCP-clients van die sessie via het gedeelde runtime-opruimpad verwijderd, zodat er geen achterblijvende stdio-verbindingen aan een verwijderde sessie gekoppeld blijven

  </Accordion>
</AccordionGroup>

### Kies een clientmodus

<Tabs>
  <Tab title="Algemene MCP-clients">
    Alleen standaard MCP-hulpmiddelen. Gebruik `conversations_list`, `messages_read`, `events_poll`, `events_wait`, `messages_send` en de goedkeuringshulpmiddelen.
  </Tab>
  <Tab title="Claude Code">
    Standaard MCP-hulpmiddelen plus de Claude-specifieke kanaaladapter. Schakel `--claude-channel-mode on` in of behoud de standaardwaarde `auto`.
  </Tab>
</Tabs>

<Note>
Momenteel gedraagt `auto` zich hetzelfde als `on`. Er is nog geen detectie van clientmogelijkheden.
</Note>

### Wat serve beschikbaar stelt

De brug gebruikt bestaande routeringsmetadata voor Gateway-sessies om door kanalen ondersteunde gesprekken beschikbaar te stellen. Een gesprek verschijnt wanneer OpenClaw al een sessiestatus heeft met een bekende route, zoals:

- `channel`
- metadata van ontvanger of bestemming
- optioneel `accountId`
- optioneel `threadId`

Dit biedt MCP-clients één plek om:

- recente gerouteerde gesprekken te vermelden
- recente transcriptiegeschiedenis te lezen
- op nieuwe inkomende gebeurtenissen te wachten
- een antwoord via dezelfde route terug te sturen
- goedkeuringsverzoeken te zien die binnenkomen terwijl de brug verbonden is

### Gebruik

<Tabs>
  <Tab title="Lokale Gateway">
    ```bash
    openclaw mcp serve
    ```
  </Tab>
  <Tab title="Externe Gateway (token)">
    ```bash
    openclaw mcp serve --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token
    ```
  </Tab>
  <Tab title="Externe Gateway (wachtwoord)">
    ```bash
    openclaw mcp serve --url wss://gateway-host:18789 --password-file ~/.openclaw/gateway.password
    ```
  </Tab>
  <Tab title="Uitgebreide uitvoer / Claude uit">
    ```bash
    openclaw mcp serve --verbose
    openclaw mcp serve --claude-channel-mode off
    ```
  </Tab>
</Tabs>

### Brughulpmiddelen

<AccordionGroup>
  <Accordion title="conversations_list">
    Vermeldt recente, door sessies ondersteunde gesprekken waarvoor al routeringsmetadata in de Gateway-sessiestatus aanwezig is.

    Filters: `limit` (max. 500), `search`, `channel`, `includeDerivedTitles`, `includeLastMessage`.

  </Accordion>
  <Accordion title="conversation_get">
    Retourneert één gesprek op basis van `session_key` via een rechtstreekse zoekopdracht naar een Gateway-sessie.
  </Accordion>
  <Accordion title="messages_read">
    Leest recente transcriptieberichten voor één door een sessie ondersteund gesprek. `limit` is standaard 20, maximaal 200.
  </Accordion>
  <Accordion title="attachments_fetch">
    Extraheert berichtinhoudsblokken zonder tekst uit één transcriptiebericht. Dit is een metadataweergave van transcriptie-inhoud, geen zelfstandige duurzame blobopslag voor bijlagen.
  </Accordion>
  <Accordion title="events_poll">
    Leest livegebeurtenissen die sinds een numerieke cursor in de wachtrij staan. `limit` maximaal 200.
  </Accordion>
  <Accordion title="events_wait">
    Voert een long poll uit totdat de volgende overeenkomende gebeurtenis in de wachtrij binnenkomt of een time-out verloopt (standaard 30s, maximaal 300s).

    Gebruik dit wanneer een algemene MCP-client bijna-realtimelevering nodig heeft zonder een Claude-specifiek pushprotocol.

  </Accordion>
  <Accordion title="messages_send">
    Stuurt tekst terug via dezelfde route die al voor de sessie is vastgelegd.

    Huidig gedrag:

    - vereist een bestaande gespreksroute
    - gebruikt het kanaal, de ontvanger, de account-id en de thread-id van de sessie
    - verzendt alleen tekst

  </Accordion>
  <Accordion title="permissions_list_open">
    Vermeldt openstaande goedkeuringsverzoeken voor uitvoering/Plugins die de brug heeft waargenomen sinds deze verbinding maakte met de Gateway.
  </Accordion>
  <Accordion title="permissions_respond">
    Handelt één openstaand goedkeuringsverzoek voor uitvoering/Plugins af met:

    - `allow-once`
    - `allow-always`
    - `deny`

  </Accordion>
</AccordionGroup>

### Gebeurtenismodel

De brug houdt een gebeurteniswachtrij in het geheugen bij zolang deze verbonden is.

Huidige gebeurtenistypen:

- `message`
- `exec_approval_requested`
- `exec_approval_resolved`
- `plugin_approval_requested`
- `plugin_approval_resolved`
- `claude_permission_request`

<Warning>
- de wachtrij bevat alleen livegegevens; deze begint wanneer de MCP-brug start
- `events_poll` en `events_wait` spelen oudere Gateway-geschiedenis niet zelf opnieuw af
- de duurzame achterstand moet worden gelezen met `messages_read`

</Warning>

### Claude-kanaalmeldingen

De brug kan ook Claude-specifieke kanaalmeldingen beschikbaar stellen. Dit is het OpenClaw-equivalent van een Claude Code-kanaaladapter: standaard MCP-hulpmiddelen blijven beschikbaar, maar live inkomende berichten kunnen ook binnenkomen als Claude-specifieke MCP-meldingen.

<Tabs>
  <Tab title="uit">
    `--claude-channel-mode off`: alleen standaard MCP-hulpmiddelen.
  </Tab>
  <Tab title="aan">
    `--claude-channel-mode on`: Claude-kanaalmeldingen inschakelen.
  </Tab>
  <Tab title="automatisch (standaard)">
    `--claude-channel-mode auto`: huidige standaardwaarde; hetzelfde bruggedrag als `on`.
  </Tab>
</Tabs>

Wanneer de Claude-kanaalmodus is ingeschakeld, kondigt de server experimentele Claude-mogelijkheden aan en kan deze het volgende uitzenden:

- `notifications/claude/channel`
- `notifications/claude/channel/permission`

Huidig bruggedrag:

- inkomende transcriptieberichten van `user` worden doorgestuurd als `notifications/claude/channel`
- Claude-machtigingsverzoeken die via MCP worden ontvangen, worden in het geheugen bijgehouden
- als de opdrachteigenaar in het gekoppelde gesprek later `yes <id>` of `no <id>` verzendt (`<id>` is de aanvraag-id van 5 letters, zonder `l`), zet de brug dit om in `notifications/claude/channel/permission`
- deze meldingen bestaan alleen tijdens de livesessie; als de MCP-client de verbinding verbreekt, is er geen pushdoel

Dit is bewust clientspecifiek. Algemene MCP-clients moeten de standaard pollinghulpmiddelen gebruiken.

### MCP-clientconfiguratie

Voorbeeldconfiguratie voor een stdio-client:

```json
{
  "mcpServers": {
    "openclaw": {
      "command": "openclaw",
      "args": [
        "mcp",
        "serve",
        "--url",
        "wss://gateway-host:18789",
        "--token-file",
        "/path/to/gateway.token"
      ]
    }
  }
}
```

Begin voor de meeste generieke MCP-clients met het standaardtooloppervlak en negeer de Claude-modus. Schakel de Claude-modus alleen in voor clients die de Claude-specifieke meldingsmethoden daadwerkelijk begrijpen.

### Opties

`openclaw mcp serve` ondersteunt:

<ParamField path="--url" type="string">
  WebSocket-URL van de Gateway. Standaard `gateway.remote.url` wanneer geconfigureerd.
</ParamField>
<ParamField path="--token" type="string">
  Gateway-token.
</ParamField>
<ParamField path="--token-file" type="string">
  Token uit bestand lezen.
</ParamField>
<ParamField path="--password" type="string">
  Gateway-wachtwoord.
</ParamField>
<ParamField path="--password-file" type="string">
  Wachtwoord uit bestand lezen.
</ParamField>
<ParamField path="--claude-channel-mode" type='"auto" | "on" | "off"'>
  Claude-meldingsmodus. Standaard `auto`.
</ParamField>
<ParamField path="-v, --verbose" type="boolean">
  Uitgebreide logboeken op stderr.
</ParamField>

<Tip>
Geef waar mogelijk de voorkeur aan `--token-file` of `--password-file` boven geheimen die rechtstreeks zijn opgegeven.
</Tip>

### Beveiligings- en vertrouwensgrens

De bridge verzint geen routering. Deze stelt alleen gesprekken beschikbaar die de Gateway al kan routeren.

Dat betekent:

- afzender-toelatingslijsten, koppeling en vertrouwen op kanaalniveau blijven onderdeel van de onderliggende OpenClaw-kanaalconfiguratie
- `messages_send` kan alleen antwoorden via een bestaande opgeslagen route
- de goedkeuringsstatus is alleen live/in het geheugen beschikbaar voor de huidige bridgesessie
- bridge-authenticatie moet dezelfde token- of wachtwoordcontroles van de Gateway gebruiken die je voor elke andere externe Gateway-client zou vertrouwen

Als een gesprek ontbreekt in `conversations_list`, ligt de gebruikelijke oorzaak niet bij de MCP-configuratie. Er ontbreken dan route-metagegevens in de onderliggende Gateway-sessie of deze zijn onvolledig.

### Testen

OpenClaw wordt geleverd met een deterministische Docker-smoketest voor deze bridge:

```bash
pnpm test:docker:mcp-channels
```

Deze smoketest voert één container uit: de gesprekstoestand wordt vooraf ingevuld, de Gateway wordt gestart en vervolgens wordt `openclaw mcp serve` als een stdio-subproces gestart en aangestuurd als MCP-client. De test verifieert het vinden van gesprekken, het lezen van transcripten, het lezen van metagegevens van bijlagen, het gedrag van de live-gebeurteniswachtrij en kanaal- en toestemmingsmeldingen in Claude-stijl via de echte stdio-MCP-bridge. Routering voor uitgaande verzending (`messages_send` waarbij de opgeslagen gespreksroute opnieuw wordt gebruikt) wordt afzonderlijk gedekt door unittests in `src/mcp/channel-server.test.ts`.

Dit is de snelste manier om te bewijzen dat de bridge werkt zonder een echt Telegram-, Discord- of iMessage-account aan de testrun te koppelen.

Zie [Testen](/nl/help/testing) voor een bredere testcontext.

### Probleemoplossing

<AccordionGroup>
  <Accordion title="Geen gesprekken geretourneerd">
    Dit betekent meestal dat de Gateway-sessie nog niet routeerbaar is. Controleer of in de onderliggende sessie metagegevens zijn opgeslagen voor kanaal/provider, ontvanger en optioneel account/threadroute.
  </Accordion>
  <Accordion title="events_poll of events_wait mist oudere berichten">
    Dit is te verwachten. De live-wachtrij begint wanneer de bridge verbinding maakt. Lees oudere transcriptgeschiedenis met `messages_read`.
  </Accordion>
  <Accordion title="Claude-meldingen worden niet weergegeven">
    Controleer al het volgende:

    - de client hield de stdio-MCP-sessie open
    - `--claude-channel-mode` is `on` of `auto`
    - de client begrijpt de Claude-specifieke meldingsmethoden daadwerkelijk
    - het inkomende bericht kwam binnen nadat de bridge verbinding had gemaakt

  </Accordion>
  <Accordion title="Goedkeuringen ontbreken">
    `permissions_list_open` toont alleen goedkeuringsverzoeken die zijn waargenomen terwijl de bridge verbonden was. Het is geen duurzame API voor goedkeuringsgeschiedenis.
  </Accordion>
</AccordionGroup>

## OpenClaw als MCP-clientregister

Dit is het pad voor `openclaw mcp list`, `show`, `status`, `doctor`, `probe`, `add`, `set`,
`configure`, `tools`, `login`, `logout`, `reload` en `unset`.

Deze opdrachten stellen OpenClaw niet beschikbaar via MCP. Ze beheren door OpenClaw beheerde MCP-serverdefinities onder `mcp.servers` in de OpenClaw-configuratie. Ze lezen geen mcporter-servers uit `config/mcporter.json`.

Deze opgeslagen definities zijn bedoeld voor runtimes die OpenClaw later start of configureert, zoals ingebedde OpenClaw en andere runtime-adapters. OpenClaw slaat de definities centraal op, zodat deze runtimes geen eigen dubbele lijsten met MCP-servers hoeven bij te houden.

<AccordionGroup>
  <Accordion title="Belangrijk gedrag">
    - deze opdrachten lezen of schrijven alleen de OpenClaw-configuratie
    - `status`, `list`, `show`, `doctor` zonder `--probe`, `set`, `configure`, `tools`, `logout`, `reload` en `unset` maken geen verbinding met de doel-MCP-server
    - `login` voert de MCP OAuth-netwerkflow uit voor de geconfigureerde HTTP-server en slaat de resulterende lokale inloggegevens op
    - `status --verbose` toont hints voor het opgeloste transport, authenticatie, de time-out, het filter en parallelle toolaanroepen zonder verbinding te maken
    - `doctor` controleert opgeslagen definities op lokale configuratieproblemen, zoals ontbrekende stdio-opdrachten, ongeldige werkmappen, ontbrekende TLS-bestanden, uitgeschakelde servers, letterlijke gevoelige header-/omgevingswaarden en onvolledige OAuth-autorisatie
    - `doctor --probe` voegt hetzelfde live-verbindingsbewijs toe als `probe` nadat de statische controles zijn geslaagd
    - `probe` maakt verbinding met de geselecteerde server of alle geconfigureerde servers, vermeldt tools en rapporteert mogelijkheden/diagnostiek
    - `add` bouwt een definitie op basis van vlaggen en voert vóór het opslaan een probe uit, tenzij `--no-probe` is ingesteld of eerst OAuth-autorisatie nodig is
    - runtime-adapters bepalen tijdens de uitvoering welke transportvormen ze daadwerkelijk ondersteunen
    - `enabled: false` houdt een server opgeslagen, maar sluit deze uit van detectie door de ingebedde runtime
    - `requestTimeoutMs` en `connectionTimeoutMs` stellen per server time-outs voor verzoeken en verbindingen in milliseconden in
    - `supportsParallelToolCalls: true` markeert servers die adapters gelijktijdig kunnen aanroepen
    - HTTP-servers kunnen statische headers, OAuth-aanmelding, beheer van TLS-verificatie en paden naar mTLS-certificaten/-sleutels gebruiken
    - ingebedde OpenClaw stelt geconfigureerde MCP-tools beschikbaar in de normale toolprofielen `coding` en `messaging`; `minimal` verbergt ze nog steeds en `tools.deny: ["bundle-mcp"]` schakelt ze expliciet uit
    - `toolFilter.include` en `toolFilter.exclude` per server filteren gevonden MCP-tools voordat ze OpenClaw-tools worden
    - servers die resources of prompts aankondigen, stellen ook hulpprogrammatools beschikbaar voor het vermelden/lezen van resources en het vermelden/ophalen van prompts; deze gegenereerde hulpprogrammanamen (`resources_list`, `resources_read`, `prompts_list`, `prompts_get`) gebruiken hetzelfde opname-/uitsluitingsfilter
    - dynamische wijzigingen in de MCP-toollijst maken de gecachte catalogus voor die sessie ongeldig; bij de volgende detectie/het volgende gebruik wordt deze vanaf de server vernieuwd
    - herhaalde fouten in MCP-toolaanvragen/het protocol pauzeren die server kort, zodat één defecte server niet de hele beurt verbruikt
    - sessiegebonden gebundelde MCP-runtimes worden na 10 minuten inactiviteit opgeruimd en eenmalige ingebedde uitvoeringen ruimen ze aan het einde van de uitvoering op

  </Accordion>
</AccordionGroup>

Runtime-adapters kunnen dit gedeelde register normaliseren naar de vorm die hun downstreamclient verwacht. Ingebedde OpenClaw gebruikt bijvoorbeeld OpenClaw-waarden van `transport` rechtstreeks, terwijl Claude Code en Gemini CLI-eigen waarden van `type` ontvangen, zoals `http`, `sse` of `stdio`.

Codex app-server respecteert ook een optioneel `codex`-blok op elke server. Dit zijn
OpenClaw-projectiemetagegevens die uitsluitend voor Codex app-server-threads bestemd zijn; ze wijzigen geen
ACP-sessies, generieke Codex-harnasconfiguratie of andere runtime-adapters.
Gebruik een niet-lege `codex.agents` om een server alleen naar specifieke OpenClaw-
agent-id's te projecteren. Lege, blanco of ongeldige agentlijsten worden door de configuratievalidatie
afgewezen en door het runtime-projectiepad weggelaten in plaats van
globaal te worden. Gebruik `codex.defaultToolsApprovalMode` (`auto`, `prompt` of `approve`)
om de systeemeigen `default_tools_approval_mode` van Codex uit te voeren voor een vertrouwde server.
OpenClaw verwijdert de metagegevens van `codex` voordat de systeemeigen `mcp_servers`-
configuratie aan Codex wordt doorgegeven.

### Opgeslagen MCP-serverdefinities

Opdrachten:

- `openclaw mcp list`
- `openclaw mcp show [name]`
- `openclaw mcp status [--verbose]`
- `openclaw mcp doctor [name] [--probe]`
- `openclaw mcp probe [name]`
- `openclaw mcp add <name> [flags]`
- `openclaw mcp set <name> <json>`
- `openclaw mcp configure <name> [flags]`
- `openclaw mcp tools <name> [--include csv] [--exclude csv] [--clear]`
- `openclaw mcp login <name> [--code code]`
- `openclaw mcp logout <name>`
- `openclaw mcp reload`
- `openclaw mcp unset <name>`

Opmerkingen:

- `list` sorteert servernamen.
- `show` zonder naam toont het volledige geconfigureerde MCP-serverobject.
- `status` classificeert geconfigureerde transporten zonder verbinding te maken. `--verbose` bevat opgeloste details over starten, time-outs, OAuth, filters en parallelle aanroepen, ook wanneer opgeslagen OAuth-tokens aanvullende autorisatie vereisen. Stdio-argumenten die inloggegevens bevatten, worden in tekst- en JSON-uitvoer geredigeerd.
- `doctor` voert statische controles uit zonder verbinding te maken. Voeg `--probe` toe wanneer de opdracht ook moet verifiëren dat ingeschakelde servers verbinding kunnen maken.
- `probe` maakt verbinding en rapporteert het aantal tools, ondersteuning voor resources/prompts en lijstwijzigingen, en diagnostiek.
- `add` accepteert stdio-vlaggen zoals `--command`, `--arg`, `--env` en `--cwd`, of HTTP-vlaggen zoals `--url`, `--transport`, `--header`, `--auth oauth`, TLS-, time-out- en toolselectievlaggen.
- `set` verwacht één JSON-objectwaarde op de opdrachtregel.
- `configure` werkt inschakeling, toolfilters, time-outs, OAuth, TLS en hints voor parallelle toolaanroepen bij zonder de volledige serverdefinitie te vervangen. Voeg `--probe` toe om de bijgewerkte server vóór het opslaan te verifiëren.
- `tools` werkt toolfilters per server bij. Opname-/uitsluitingsitems zijn MCP-toolnamen en eenvoudige `*`-globs.
- `login` voert de OAuth-flow uit voor HTTP-servers die zijn geconfigureerd met `auth: "oauth"`. De eerste uitvoering toont een autorisatie-URL; voer de opdracht na goedkeuring opnieuw uit met `--code`.
- `logout` wist opgeslagen OAuth-inloggegevens voor de benoemde server zonder de opgeslagen serverdefinitie te verwijderen.
- `reload` verwijdert gecachte MCP-runtimes in het proces uitsluitend voor het huidige CLI-proces. Gateway- of agentprocessen in een ander proces hebben nog steeds hun eigen herlaad- of herstartpad nodig.
- Gebruik `transport: "streamable-http"` voor Streamable HTTP MCP-servers. `openclaw mcp set` normaliseert ook CLI-eigen `type: "http"` naar dezelfde canonieke configuratievorm voor compatibiliteit.
- `unset` mislukt als de benoemde server niet bestaat.

Voorbeelden:

```bash
openclaw mcp list
openclaw mcp show context7 --json
openclaw mcp status --verbose
openclaw mcp doctor --probe
openclaw mcp probe context7 --json
openclaw mcp add memory --command npx --arg -y --arg @modelcontextprotocol/server-memory
openclaw mcp set context7 '{"command":"uvx","args":["context7-mcp"]}'
openclaw mcp tools context7 --include 'resolve-library-id,get-library-docs'
openclaw mcp set docs '{"url":"https://mcp.example.com","transport":"streamable-http"}'
openclaw mcp configure docs --timeout 20 --connect-timeout 5 --include 'search,read_*'
openclaw mcp configure docs --auth oauth --oauth-scope 'docs.read'
openclaw mcp login docs
openclaw mcp logout docs
openclaw mcp unset context7
```

### Veelgebruikte serverrecepten

Deze voorbeelden slaan alleen serverdefinities op. Voer daarna `openclaw mcp doctor --probe` uit om te bewijzen dat de server start en tools beschikbaar stelt.

<Tabs>
  <Tab title="Bestandssysteem">
    ```bash
    openclaw mcp add files \
      --command npx \
      --arg -y \
      --arg @modelcontextprotocol/server-filesystem \
      --arg "$HOME/Documents" \
      --include 'read_file,list_directory,search_files'
    openclaw mcp doctor files --probe
    ```

    Beperk bestandssysteemservers tot de kleinste directorystructuur die de agent moet kunnen lezen of bewerken.

  </Tab>
  <Tab title="Geheugen">
    ```bash
    openclaw mcp add memory \
      --command npx \
      --arg -y \
      --arg @modelcontextprotocol/server-memory
    openclaw mcp probe memory --json
    ```

    Gebruik een toolfilter als de server schrijftools beschikbaar stelt die niet toegankelijk mogen zijn voor normale agents.

  </Tab>
  <Tab title="Lokaal script">
    ```bash
    openclaw mcp add local-tools \
      --command node \
      --arg ./dist/mcp-server.js \
      --cwd /srv/openclaw-tools \
      --env API_BASE=https://internal.example
    openclaw mcp status --verbose
    ```

    `doctor` controleert of `cwd` bestaat en of de opdracht vanuit de geconfigureerde omgeving kan worden gevonden.

  </Tab>
  <Tab title="HTTP op afstand">
    ```bash
    openclaw mcp add docs \
      --url https://mcp.example.com/mcp \
      --transport streamable-http \
      --auth oauth \
      --oauth-scope docs.read \
      --timeout 20 \
      --connect-timeout 5 \
      --include 'search,read_*'
    openclaw mcp doctor docs --probe
    ```

    Gebruik OAuth wanneer de externe server dit ondersteunt. Als de server statische headers vereist, leg dan geen letterlijke bearer-tokens vast in een commit.

  </Tab>
  <Tab title="Desktop/CUA">
    ```bash
    openclaw mcp set cua-driver '{"command":"cua-driver","args":["mcp"]}'
    openclaw mcp tools cua-driver --include 'list_apps,get_window_state,click,type_text'
    openclaw mcp doctor cua-driver --probe
    ```

    Servers voor directe desktopbesturing nemen de machtigingen over van het proces dat ze starten. Gebruik beperkte toolfilters en machtigingsprompts op besturingssysteemniveau.

  </Tab>
</Tabs>

### Structuren van JSON-uitvoer

Gebruik `--json` voor scripts en dashboards. Veldensets kunnen in de loop van de tijd groeien, dus consumers moeten onbekende sleutels negeren.

<AccordionGroup>
  <Accordion title="status --json">
    ```json
    {
      "path": "/home/user/.openclaw/openclaw.json",
      "servers": [
        {
          "name": "docs",
          "configured": true,
          "enabled": true,
          "ok": true,
          "transport": "streamable-http",
          "launch": "streamable-http https://mcp.example.com/mcp",
          "auth": "oauth",
          "authStatus": {
            "hasTokens": true,
            "requiresAuthorization": false,
            "hasClientInformation": true,
            "hasCodeVerifier": false,
            "hasDiscoveryState": true,
            "hasLastAuthorizationUrl": false
          },
          "requestTimeoutMs": 20000,
          "connectionTimeoutMs": 5000,
          "toolFilter": {
            "include": ["search", "read_*"],
            "exclude": []
          },
          "supportsParallelToolCalls": true
        }
      ]
    }
    ```
  </Accordion>
  <Accordion title="doctor --json">
    ```json
    {
      "ok": true,
      "path": "/home/user/.openclaw/openclaw.json",
      "servers": [
        {
          "name": "docs",
          "ok": true,
          "issues": [
            {
              "level": "warning",
              "message": "OAuth-referenties zijn niet geautoriseerd; voer openclaw mcp login docs uit"
            }
          ]
        }
      ]
    }
    ```

    `doctor --json` wordt afgesloten met een niet-nulstatus wanneer een ingeschakelde, gecontroleerde server een probleem op `error`-niveau heeft. Problemen met `warning` en `info` worden gemeld, maar laten de opdracht op zichzelf niet mislukken.

  </Accordion>
  <Accordion title="probe --json">
    ```json
    {
      "generatedAt": "2026-05-31T09:00:00.000Z",
      "servers": {
        "docs": {
          "launch": "streamable-http https://mcp.example.com/mcp",
          "tools": 2,
          "resources": true,
          "listChanged": {
            "tools": true,
            "resources": false,
            "prompts": false
          }
        }
      },
      "tools": ["docs__read_page", "docs__search"],
      "diagnostics": []
    }
    ```

    `probe --json` opent een live MCP-clientsessie en drukt het resultaat rechtstreeks af; in tegenstelling tot `status`/`doctor` heeft de uitvoer geen `path`-veld op het hoogste niveau. De sleutels `resources` en `prompts` zijn alleen aanwezig wanneer de server die mogelijkheid daadwerkelijk aanbiedt (een server zonder prompts laat de sleutel `prompts` weg in plaats van `false` te rapporteren). Gebruik `probe` als bewijs van bereikbaarheid en mogelijkheden, niet voor statische configuratie-audits.

  </Accordion>
</AccordionGroup>

Voorbeeld van een configuratiestructuur:

```json
{
  "mcp": {
    "servers": {
      "context7": {
        "command": "uvx",
        "args": ["context7-mcp"]
      },
      "docs": {
        "url": "https://mcp.example.com",
        "transport": "streamable-http",
        "requestTimeoutMs": 20000,
        "connectionTimeoutMs": 5000,
        "supportsParallelToolCalls": true,
        "auth": "oauth",
        "oauth": {
          "scope": "docs.read"
        },
        "sslVerify": true,
        "clientCert": "/path/to/client.crt",
        "clientKey": "/path/to/client.key",
        "toolFilter": {
          "include": ["search_*"],
          "exclude": ["admin_*"]
        }
      }
    }
  }
}
```

### Stdio-transport

Start een lokaal onderliggend proces en communiceert via stdin/stdout.

| Veld                       | Beschrijving                               |
| -------------------------- | ------------------------------------------ |
| `command`         | Te starten uitvoerbaar bestand (vereist)   |
| `args`         | Array met opdrachtregelargumenten          |
| `env`         | Extra omgevingsvariabelen                  |
| `cwd` / `workingDirectory` | Werkdirectory voor het proces |

<Warning>
**Veiligheidsfilter voor de stdio-omgeving**

OpenClaw weigert omgevingssleutels voor het opstarten van interpreters, het kapen van loaders en shellinitialisatie voordat een stdio-MCP-server wordt gestart, zelfs als ze in het `env`-blok van een server staan. Hiervoor wordt hetzelfde beveiligingsbeleid voor de hostomgeving gebruikt als voor andere processen die door OpenClaw worden gestart: bekende hooks voor het opstarten van interpreters worden geblokkeerd (bijvoorbeeld `NODE_OPTIONS`, `PYTHONSTARTUP`, `PERL5OPT`, `RUBYOPT`, `BASHOPTS`, `KSH_ENV`), evenals prefixen voor het injecteren van gedeelde bibliotheken en functies (`DYLD_*`, `LD_*`, `BASH_FUNC_*`) en vergelijkbare variabelen voor runtimebesturing. Bij het opstarten worden deze stilzwijgend verwijderd en wordt een waarschuwing gelogd, zodat ze geen impliciete prelude kunnen injecteren, de interpreter kunnen vervangen, een debugger kunnen inschakelen of de dynamische linker voor het stdio-proces kunnen kapen. Dankzij een expliciete toelatingslijst blijven gewone omgevingsvariabelen voor MCP-referenties bruikbaar (`GITHUB_TOKEN`, `GH_TOKEN`, `GITLAB_TOKEN`, `NPM_TOKEN`, `NODE_AUTH_TOKEN`, `DATABASE_URL`, `MONGODB_URI`, `REDIS_URL`, `AMQP_URL`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`), samen met gewone proxy- en serverspecifieke omgevingsvariabelen (`HTTP_PROXY`, aangepaste `*_API_KEY`, enzovoort). Andere `AWS_*`-sleutels, zoals `AWS_CONFIG_FILE` en `AWS_SHARED_CREDENTIALS_FILE`, blijven geblokkeerd omdat ze naar bestanden met referenties verwijzen in plaats van rechtstreeks een referentiewaarde te bevatten.

Als je MCP-server werkelijk een van de geblokkeerde variabelen nodig heeft, stel je deze in voor het Gateway-hostproces in plaats van onder `env` van de stdio-server.
</Warning>

### SSE-/HTTP-transport

Maakt via HTTP Server-Sent Events verbinding met een externe MCP-server.

| Veld                        | Beschrijving                                                         |
| --------------------------- | -------------------------------------------------------------------- |
| `url`          | HTTP- of HTTPS-URL van de externe server (vereist)                   |
| `headers`          | Optionele sleutel-waardetoewijzing van HTTP-headers (bijvoorbeeld authenticatietokens) |
| `connectionTimeoutMs`          | Verbindingstime-out per server in ms (optioneel)                     |
| `requestTimeoutMs`          | Time-out voor MCP-verzoeken per server in milliseconden              |
| `auth: "oauth"`          | Gebruik MCP-OAuth-referenties die door `openclaw mcp login` zijn opgeslagen |
| `sslVerify`          | Stel alleen in op false voor expliciet vertrouwde privé-HTTPS-eindpunten |
| `clientCert` / `clientKey` | Paden naar het mTLS-clientcertificaat en de sleutel         |
| `supportsParallelToolCalls`          | Aanwijzing dat gelijktijdige aanroepen veilig zijn voor deze server  |

Voorbeeld:

```json
{
  "mcp": {
    "servers": {
      "remote-tools": {
        "url": "https://mcp.example.com",
        "auth": "oauth",
        "requestTimeoutMs": 20000,
        "headers": {
          "Authorization": "Bearer <token>"
        }
      }
    }
  }
}
```

Gevoelige waarden in `url` (gebruikersinformatie) en `headers` worden in logboeken en statusuitvoer geredigeerd. `openclaw mcp doctor` waarschuwt wanneer `headers`- of `env`-vermeldingen die gevoelig lijken, letterlijke waarden bevatten, zodat operators die waarden uit vastgelegde configuratie kunnen verplaatsen.

### OAuth-workflow

OAuth is bedoeld voor HTTP-MCP-servers die de MCP-OAuth-flow aanbieden. Statische `Authorization`-headers worden voor een server genegeerd zolang `auth: "oauth"` is ingeschakeld. Referenties die door `openclaw mcp login` zijn opgeslagen, werken met ingebedde MCP, CLI-runners en de lokale Codex-appserver.

Native MCP-OAuth-sessies bevinden zich in de gedeelde SQLite-database die alleen voor de eigenaar toegankelijk is op `<state-dir>/state/openclaw.sqlite` (`mcp_oauth_stores`). De rij kan toegangs- en vernieuwingstokens, geheimen voor dynamische clientregistratie, ontdekkingsmetadata en de tijdelijke PKCE-verifier bevatten. Vernieuwen, inloggen en uitloggen gebruiken dezelfde SQLite-lease, zodat parallelle OpenClaw-processen niet één vernieuwingstoken kunnen gebruiken of een uitgelogde sessie opnieuw tot leven kunnen brengen.

Upgrades vanuit de buiten gebruik gestelde `<state-dir>/mcp-oauth/*.json`-opslag worden alleen door `openclaw doctor --fix` afgehandeld. Runtimecode leest of schrijft die bestanden nooit en valt er ook nooit op terug.

Totdat referenties beschikbaar zijn, laat OpenClaw alleen die MCP-server weg uit de runtime van de agent in plaats van de agentbeurt te laten mislukken. De operator, of een agent met shelltoegang, kan vervolgens `openclaw mcp login <name>` uitvoeren en de server in een latere beurt gebruiken.

Als een server een token weigert met `insufficient_scope`, behoudt OpenClaw het aangevraagde bereik en vraagt het om `openclaw mcp login <name>` in plaats van een vernieuwing te herhalen die geen nieuw bereik kan verlenen. Die aanmelding start een nieuw autorisatieverzoek, terwijl het vorige token behouden blijft totdat vervangende referenties zijn opgeslagen.

Wanneer een externe MCP-service al wordt ondersteund door een afzonderlijk OpenClaw-authenticatieprofiel dat kan vernieuwen, kun je optioneel `oauth.authProfileId` instellen. OpenClaw vernieuwt een van beide referentiebronnen vóór de runtimeprojectie en geeft alleen het actuele toegangstoken door aan de onderliggende MCP-client.

<Steps>
  <Step title="De server opslaan">
    Voeg de server toe of werk deze bij met `auth: "oauth"` en eventuele optionele OAuth-metadata.

    ```bash
    openclaw mcp set docs '{"url":"https://mcp.example.com/mcp","transport":"streamable-http","auth":"oauth","oauth":{"scope":"docs.read"}}'
    ```

    Sla voor een bearer-token dat door een auth-profiel wordt ondersteund de profielkoppeling op:

    ```bash
    openclaw mcp set docs '{"url":"https://mcp.example.com/mcp","transport":"streamable-http","auth":"oauth","oauth":{"authProfileId":"docs:mcp"}}'
    ```

  </Step>
  <Step title="Aanmelding starten">
    Voer de aanmelding uit om het autorisatieverzoek te maken.

    ```bash
    openclaw mcp login docs
    ```

    OpenClaw toont de autorisatie-URL en slaat de tijdelijke OAuth-verificatiestatus op in gedeelde SQLite.

  </Step>
  <Step title="Voltooien met de code">
    Geef na goedkeuring in de browser de geretourneerde code door aan OpenClaw.

    ```bash
    openclaw mcp login docs --code abc123
    ```

  </Step>
  <Step title="Autorisatie controleren">
    Gebruik status of doctor om te bevestigen dat tokens aanwezig zijn en geen aanvullende autorisatie vereisen. Als status `authorization-required` meldt of doctor om aanvullende autorisatie vraagt, voer je `openclaw mcp login <name>` opnieuw uit.

    ```bash
    openclaw mcp status --verbose
    openclaw mcp doctor docs --probe
    ```

  </Step>
  <Step title="Inloggegevens wissen">
    Afmelden verwijdert opgeslagen OAuth-inloggegevens, maar behoudt de opgeslagen serverdefinitie.

    ```bash
    openclaw mcp logout docs
    ```

  </Step>
</Steps>

Als de provider tokens roteert of de autorisatiestatus vastloopt, voer je `openclaw mcp logout <name>` uit en herhaal je vervolgens `login`. `logout` kan inloggegevens voor een opgeslagen HTTP-server wissen, zelfs nadat `auth: "oauth"` uit de configuratie is verwijderd, zolang de servernaam en URL de vermelding in de opslag voor inloggegevens nog identificeren.

### Streamable HTTP-transport

`streamable-http` is een aanvullende transportoptie naast `sse` en `stdio`. Deze gebruikt HTTP-streaming voor bidirectionele communicatie met externe MCP-servers.

| Veld                        | Beschrijving                                                                           |
| --------------------------- | -------------------------------------------------------------------------------------- |
| `url`                       | HTTP- of HTTPS-URL van de externe server (vereist)                                     |
| `transport`                 | Stel in op `"streamable-http"` om dit transport te selecteren; bij weglating gebruikt OpenClaw `sse` |
| `headers`                   | Optionele sleutel-waardetoewijzing van HTTP-headers (bijvoorbeeld auth-tokens)         |
| `connectionTimeoutMs`       | Verbindingstime-out per server in ms (optioneel)                                       |
| `requestTimeoutMs`          | Time-out per MCP-verzoek per server in milliseconden                                   |
| `auth: "oauth"`             | Gebruik MCP OAuth-inloggegevens die zijn opgeslagen door `openclaw mcp login`          |
| `sslVerify`                 | Stel alleen in op false voor expliciet vertrouwde privé-HTTPS-eindpunten               |
| `clientCert` / `clientKey`  | Paden naar het mTLS-clientcertificaat en de sleutel                                    |
| `supportsParallelToolCalls` | Geeft aan dat gelijktijdige aanroepen veilig zijn voor deze server                     |

De OpenClaw-configuratie gebruikt `transport: "streamable-http"` als de canonieke spelling. CLI-eigen MCP-waarden voor `type: "http"` worden geaccepteerd wanneer ze via `openclaw mcp set` worden opgeslagen en door `openclaw doctor --fix` in bestaande configuratie worden hersteld, maar `transport` is wat ingebed OpenClaw rechtstreeks gebruikt.

Voorbeeld:

```json
{
  "mcp": {
    "servers": {
      "streaming-tools": {
        "url": "https://mcp.example.com/stream",
        "transport": "streamable-http",
        "connectionTimeoutMs": 10000,
        "requestTimeoutMs": 30000,
        "headers": {
          "Authorization": "Bearer <token>"
        }
      }
    }
  }
}
```

<Note>
Registeropdrachten starten de kanaalbridge niet. Alleen `probe` en `doctor --probe` openen een actieve MCP-clientsessie om te bewijzen dat de doelserver bereikbaar is.
</Note>

## Control UI

De Control UI in de browser bevat een speciale MCP-instellingenpagina op `/settings/mcp`; het eerdere pad `/mcp` blijft een alias. De pagina toont aantallen geconfigureerde servers, samenvattingen van ingeschakelde servers/OAuth/filters, transportrijen per server, bedieningselementen voor in- en uitschakelen, algemene CLI-opdrachten en een editor met beperkt bereik voor de configuratiesectie `mcp`.

Gebruik de pagina voor wijzigingen door operators en een snelle inventarisatie. Gebruik `openclaw mcp doctor --probe` of `openclaw mcp probe` wanneer je live bewijs van de server nodig hebt.

Operatorworkflow:

1. Open de Control UI en kies **MCP**.
2. Bekijk de overzichtskaarten voor het totale aantal, ingeschakelde, OAuth- en gefilterde servers.
3. Gebruik elke serverrij voor aanwijzingen over transport, auth, filters, time-outs en opdrachten.
4. Schakel de activering om wanneer je een definitie wilt behouden maar deze van runtimedetectie wilt uitsluiten.
5. Bewerk de configuratiesectie `mcp` met beperkt bereik voor structurele wijzigingen, zoals nieuwe servers, headers, TLS, OAuth-metadata of toolfilters.
6. Kies **Save** om alleen de configuratie op te slaan, of **Save & Publish** om deze via het configuratiepad van de Gateway toe te passen.
7. Voer `openclaw mcp doctor --probe` uit wanneer je live bewijs nodig hebt dat de bewerkte server start en tools vermeldt.

Opmerkingen:

- opdrachtfragmenten plaatsen servernamen tussen aanhalingstekens, zodat ongebruikelijke namen in een shell kopieerbaar blijven
- weergegeven URL-achtige waarden worden vóór weergave geredigeerd wanneer ze ingebedde inloggegevens bevatten
- de pagina start zelf geen MCP-transporten
- actieve runtimes kunnen `openclaw mcp reload`, publicatie van de Gateway-configuratie of een procesherstart vereisen, afhankelijk van welk proces eigenaar is van de MCP-clients

## MCP Apps

OpenClaw kan tools weergeven die de stabiele [MCP Apps-extensie](https://modelcontextprotocol.io/extensions/apps) implementeren. Apps zijn opt-in omdat hun HTML afkomstig is van de geconfigureerde MCP-server en om voor de app zichtbare tools of resources van diezelfde server kan vragen.

Schakel de hostbridge in:

```bash
openclaw config set mcp.apps.enabled true --strict-json
```

Herstart de Gateway nadat je deze instelling hebt gewijzigd. Wanneer deze is ingeschakeld, start OpenClaw een HTTP(S)-listener die uitsluitend voor de sandbox is bedoeld, op de Gateway-poort plus één (voor de standaard-Gateway, `18790`). De Control UI laadt Apps vanaf die afzonderlijke origin; de listener bedient nooit de Control UI, geauthenticeerde Gateway-routes of gebruikersgegevens.

Rechtstreekse Gateway-verbindingen hebben toegang tot beide poorten nodig. Als een reverse proxy of TLS-terminator de Control UI beschikbaar stelt, geef je Apps een eigen openbare origin en stuur je alleen die origin via een proxy door naar de sandboxlistener:

```json5
{
  mcp: {
    apps: {
      enabled: true,
      sandboxOrigin: "https://mcp-apps.example.com",
      sandboxPort: 18790,
    },
  },
}
```

De sandbox-origin moet verschillen van de origin van de Control UI. Host er geen andere geauthenticeerde of gevoelige inhoud op.

De officiële eenvoudige React-demo kan bijvoorbeeld als volgt worden geconfigureerd:

```json5
{
  mcp: {
    apps: { enabled: true },
    servers: {
      "basic-react": {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-basic-react", "--stdio"],
      },
    },
  },
}
```

Gedrag en beveiligingsgrenzen:

- OpenClaw kondigt de extensie `io.modelcontextprotocol/ui` alleen aan wanneer Apps zijn ingeschakeld.
- Alleen resources van `ui://` met exact het MIME-type `text/html;profile=mcp-app` worden weergegeven.
- UI-resources zijn beperkt tot 2 MiB, worden achter een proxy met dubbele iframe op een speciale buitenste origin geplaatst, in een ondoorzichtige binnenste App-origin geladen en beperkt door CSP die is afgeleid van de resourcemetadata.
- Tools die uitsluitend voor Apps zijn bedoeld (`_meta.ui.visibility: ["app"]`), blijven buiten de toollijsten van het model. Apps kunnen alleen voor de app zichtbare tools op hun eigen server aanroepen die ook voldoen aan het effectieve OpenClaw-toolbeleid voor de uitvoering die de weergave heeft gemaakt.
- Aan de origin gebonden App-machtigingen, zoals camera, microfoon en geolocatie, worden niet verleend zolang binnenste App-documenten ondoorzichtige origins gebruiken voor isolatie tussen Apps.
- App-HTML, volledige toolargumenten en onbewerkte resultaten bevinden zich in een begrensde weergavelease van tien minuten in het geheugen en worden niet naar schijf geschreven of naar metadata voor transcriptvoorbeelden gekopieerd. Het transcript slaat alleen een begrensde server-/tool-/resourcedescriptor op die aan de oorspronkelijke toolaanroep-ID is gekoppeld. Na een herstart van de Gateway kan de Control UI die descriptor verifiëren aan de hand van het geauthenticeerde sessietranscript en de resource `ui://` opnieuw ophalen; opnieuw samengestelde weergaven zijn alleen-lezen totdat een nieuwe uitvoering de huidige toolmachtigingen vaststelt.
- In kanaalgesprekken voegt de laatste geslaagde App-weergave in een beurt één actie in de stijl van **Open App** toe aan het uiteindelijke antwoord van de assistent. Telegram-DM's gebruiken een systeemeigen Mini App-knop; Slack en Discord geven dezelfde overdraagbare actie weer als een link. Andere kanalen behouden de oorspronkelijke antwoordtekst en voegen een begrijpelijke HTTPS-link toe.
- Startlinks voor kanalen zijn alleen beschikbaar wanneer Gateway-blootstelling via Tailscale een gepubliceerde HTTPS-origin heeft voorbereid. `gateway.tailscale.mode: "serve"` is alleen bereikbaar vanaf het tailnet; `"funnel"` is bereikbaar vanaf het openbare internet. Een extern beheerde Funnel die door `gateway.tailscale.preserveFunnel` wordt behouden, wordt eveneens als via internet bereikbaar beschouwd. Zie [Tailscale](/nl/gateway/tailscale).
- Starttickets zijn ondoorzichtig, worden alleen aangemaakt tijdens het samenstellen van het uiteindelijke kanaalantwoord en verlopen na maximaal twee minuten of wanneer de onderliggende weergavelease verloopt, afhankelijk van wat het eerst gebeurt. De URL bevat geen bearer-inloggegevens van de Gateway, sessiesleutels, weergavemetadata, App-HTML, toolinvoer of toolresultaten.
- Als er geen gepubliceerde origin of ticketcapaciteit beschikbaar is, de weergave of het ticket is verlopen, of het transport geen systeemeigen bedieningselementen kan weergeven, blijft de oorspronkelijke assistenttekst beschikbaar. De Control UI behoudt het bestaande inline App-canvas en ontvangt geen dubbele startactie.
- `openclaw security audit` waarschuwt zolang de bridge is ingeschakeld. Schakel deze uit met `openclaw config set mcp.apps.enabled false --strict-json` wanneer deze niet nodig is.

## Huidige beperkingen

Deze pagina documenteert de bridge zoals die momenteel wordt geleverd.

Huidige beperkingen:

- gespreksdetectie is afhankelijk van bestaande metadata van Gateway-sessieroutes
- geen generiek pushprotocol naast de Claude-specifieke adapter
- nog geen tools voor het bewerken van berichten of het toevoegen van reacties
- HTTP/SSE/streamable-http-transport maakt verbinding met één externe server; nog geen gemultiplexte upstream
- `permissions_list_open` bevat alleen goedkeuringen die zijn waargenomen terwijl de bridge verbonden is

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Plugins](/nl/cli/plugins)
