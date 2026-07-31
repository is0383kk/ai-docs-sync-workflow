---
read_when:
    - Je wilt begrijpen welke sessietools de agent heeft
    - Je wilt toegang tussen sessies of het starten van subagents configureren
    - Je wilt de status van gestarte subagents controleren
summary: Agenttools voor sessieoverstijgende status, herinnering, berichtenuitwisseling en orkestratie van subagents
title: Sessietools
x-i18n:
    generated_at: "2026-07-27T05:44:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ceaf48addc9fc57afe2f6428cda03ed8b19f4efce93b13b58b7ef493a41c62fe
    source_path: concepts/session-tool.md
    workflow: 16
---

OpenClaw biedt agents hulpmiddelen om tussen sessies te werken, de status te inspecteren en sub-agents te orkestreren.

## Beschikbare hulpmiddelen

| Hulpmiddel                 | Functie                                                                |
| -------------------- | --------------------------------------------------------------------------- |
| `sessions`           | Zichtbare sessie-instellingen aanpassen en de algemene catalogus met sessiegroepen beheren  |
| `sessions_list`      | Sessies weergeven met optionele filters (soort, label, agent, archief, voorbeeld)  |
| `sessions_search`    | Zichtbare sessietranscripten doorzoeken en overeenkomende fragmenten retourneren             |
| `sessions_history`   | Het transcript van een specifieke sessie lezen                                   |
| `sessions_send`      | Een andere sessie op dezelfde Gateway uitvoeren en optioneel wachten                 |
| `conversations_list` | Stabiele externe gespreksadressen weergeven                                 |
| `conversations_send` | Naar één exact extern gesprek verzenden zonder een lokale sessie uit te voeren     |
| `conversations_turn` | Naar één exact extern gesprek verzenden en op het bijbehorende antwoord wachten   |
| `sessions_spawn`     | Een geïsoleerde sub-agentsessie voor achtergrondwerk starten                     |
| `sessions_yield`     | De huidige beurt beëindigen en op vervolgresultaten van sub-agents wachten               |
| `subagents`          | Achtergrondwerk in deze sessieboom weergeven of annuleren                         |
| `session_status`     | Een kaart in `/status`-stijl tonen en optioneel een modeloverschrijving per sessie instellen |

Deze hulpmiddelen blijven onderworpen aan het actieve hulpmiddelenprofiel en het toestaan/weigeren-beleid. `tools.profile: "coding"` bevat de volledige set voor sessieorkestratie. `tools.profile: "messaging"` bevat zelfbediening voor sessies, ontdekking, herinnering, berichtenuitwisseling tussen sessies, hulpmiddelen voor externe gesprekken en de volledige startlevenscyclus (`sessions_spawn`, `sessions_yield` en `subagents`). De uitsluitend voor de UI bedoelde hulpmiddelen voor taaksuggesties `spawn_task` en `dismiss_task` blijven hulpmiddelen van het coderingsprofiel.

Beleid voor groepen, providers, sandboxen en afzonderlijke agents kan deze hulpmiddelen na de profielfase nog steeds verwijderen. Gebruik `/tools` vanuit de betreffende sessie om de effectieve lijst met hulpmiddelen te inspecteren.

## Sessies weergeven en lezen

`sessions_list` retourneert gerichte ontdekkingsrijen: sessiesleutel, agent, soort, kanaal, label-/titel-/voorbeeldvelden, bovenliggende en onderliggende relaties, laatste update, archief-/vastgezette status, statusversie, model, aantallen context-/totale tokens, uitvoeringsstatus en of de laatste uitvoering is afgebroken. Filter op `kinds` (matrix; geaccepteerde waarden: `main`, `group`, `cron`, `hook`, `node`, `other`), exacte `label`, exacte `agentId`, `search`-tekst of recentheid (`activeMinutes`). Actieve sessies worden standaard geretourneerd; geef `archived: true` door om in plaats daarvan gearchiveerde sessies te inspecteren. Stel `includeDerivedTitles`, `includeLastMessage` of `messageLimit` (beperkt tot 20) in wanneer je triage in postvakstijl nodig hebt: een aan het zichtbaarheidsbereik ontleende titel, een voorbeeldfragment van het laatste bericht of een begrensde reeks recente berichten in elke rij. Routering van bezorging, interne sessie-ID's, timing/instellingen per uitvoering, kostenschattingen en transcriptpaden worden bewust weggelaten; gebruik `session_status`, gesprekshulpmiddelen en `sessions_history` voor deze eigenaarspecifieke details. Afgeleide titels en voorbeelden worden alleen geproduceerd voor sessies die de aanroeper al kan zien volgens het geconfigureerde zichtbaarheidsbeleid voor sessiehulpmiddelen, zodat niet-gerelateerde sessies verborgen blijven. Wanneer de zichtbaarheid beperkt is, retourneert `sessions_list` optionele `visibility`-metagegevens die de effectieve modus tonen en waarschuwen dat resultaten mogelijk tot het bereik beperkt zijn.

`sessions_history` haalt het gesprekstranscript voor een specifieke sessie op. Standaard worden hulpmiddelresultaten uitgesloten; geef `includeTools: true` door om ze te zien. Gebruik `limit` voor het nieuwste begrensde einde. Geef `offset: 0` door wanneer je pagineringsmetagegevens nodig hebt en geef vervolgens geretourneerde `nextOffset`-waarden door om achterwaarts door oudere OpenClaw-transcriptvensters te bladeren zonder onbewerkte transcriptbestanden te lezen. Expliciete offsetpagina's voegen geen externe CLI-terugvalimports samen; gebruik de standaardweergave van het nieuwste einde (zonder `offset`) wanneer je die samengevoegde weergavegeschiedenis nodig hebt.

De geretourneerde weergave is bewust begrensd en op veiligheid gefilterd:

- assistenttekst wordt vóór het terughalen genormaliseerd:
  - denktags worden verwijderd
  - `<relevant-memories>`- / `<relevant_memories>`-structuurblokken worden verwijderd
  - XML-nettoladingblokken van hulpmiddelaanroepen in platte tekst, zoals `<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>` en `<function_calls>...</function_calls>`, worden verwijderd, inclusief afgekorte nettoladingen die nooit correct worden afgesloten
  - gedegradeerde structuur voor hulpmiddelaanroepen/-resultaten, zoals `[Tool Call: ...]`, `[Tool Result ...]` en `[Historical context ...]`, wordt verwijderd
  - gelekte modelbesturingstokens, zoals `<|assistant|>`, andere ASCII-`<|...|>`-tokens en varianten van `<｜...｜>` met volledige breedte, worden verwijderd
  - ongeldige MiniMax-XML voor hulpmiddelaanroepen, zoals `<invoke ...>` / `</minimax:tool_call>`, wordt verwijderd
- tekst die op aanmeldgegevens/tokens lijkt, wordt vóór retournering geredigeerd
- lange tekstblokken worden afgekapt
- bij zeer grote geschiedenissen kunnen oudere rijen worden weggelaten of kan een te grote rij worden vervangen door `[sessions_history omitted: message too large]`
- het hulpmiddel rapporteert samenvattingsvlaggen zoals `truncated`, `droppedMessages`, `contentTruncated`, `contentRedacted`, `bytes` en pagineringsmetagegevens

Gebruik de geretourneerde **sessiesleutel** (zoals `"main"`) met `sessions_history`, `sessions_send` en `session_status`. Deze doelhulpmiddelen kunnen ook een bekende sessie-ID omzetten, maar `sessions_list` maakt interne ID's niet zichtbaar.

Als je het exacte onbewerkte transcript nodig hebt, inspecteer dan de SQLite-transcriptrijen binnen het bereik in plaats van `sessions_history` als een ongefilterde dump te behandelen.

Gebruik [`sessions_search`](/nl/concepts/session-search) om exact in volledige tekst te zoeken in zichtbare transcripttekst van gebruikers en assistenten. De resultaten bevatten een `sessionKey` voor een vervolgaanroep van `sessions_history`; zichtbaarheidsfiltering, redigering van fragmenten en uitvoergrenzen komen overeen met de geschiedenisgrens.

## Sessie-instellingen en groepen beheren

Het aan de eigenaar voorbehouden hulpmiddel `sessions` biedt twee begrensde zelfbedieningsoppervlakken:

- `action: "patch"` wijzigt standaard de huidige sessie, of een andere zichtbare sessie die met `sessionKey` is geselecteerd. Het kan het label, het zijbalkpictogram, de vastgezette/archiefstatus, het model en het denkniveau instellen. Het biedt geen acties voor opnieuw instellen, verwijderen of Compaction.
- `group_list`, `group_set`, `group_rename` en `group_delete` beheren de algemene geordende catalogus met sessiegroepen. `group_set` vervangt de geordende namenlijst in plaats van één item aan te passen.

Een door een agent geselecteerde modelaanpassing blijft omkeerbaar totdat met die selectie een uitvoering succesvol is voltooid. Als het geselecteerde model definitief onbruikbaar is vanwege een authenticatie-, facturerings- of model-niet-gevondenfout, herstelt OpenClaw het vorige model en schrijft het een zichtbare systeemnotitie. Tijdelijke fouten met snelheidslimieten, overbelasting, time-outs, netwerken en servers maken de selectie niet ongedaan.

## Sessies versus gesprekken

Een **sessie** is lokale modelcontext. Een **gesprek** is een exact extern adres, zoals één gesprekspartner, kanaal of thread. De twee zijn gekoppeld, maar niet onderling uitwisselbaar: directe berichten kunnen één `main`-sessie delen terwijl ze afzonderlijke gespreksadressen behouden.

`conversations_list` retourneert ondoorzichtige `conversationRef`-waarden voor de actieve agent. Met een expliciete `channel` vernieuwt de Gateway ook adressen vanuit de lokale map van dat kanaal, zoals goedgekeurde Reef-gesprekspartners; gebruik `query` om een specifieke gesprekspartner buiten de huidige resultatenpagina te vinden. Ontdekking catalogiseert het adres zonder een modelcontextsessie te maken; de onderliggende sessie wordt alleen gemaakt wanneer bezorging of inkomende context dit vereist. Gespreksontdekking en -bezorging zijn uitsluitend voor de eigenaar, omdat ze de kanaalaanmeldgegevens van de Gateway gebruiken. Gebruik `conversations_send` voor verzenden zonder op het resultaat te wachten. Gebruik `conversations_turn` wanneer het externe antwoord bij de huidige modelbeurt hoort: de Gateway reserveert één transportbericht-ID, slaat vóór transport-I/O een bezorgingsbewerking en wachtrij-intentie op en retourneert het bijbehorende antwoord vanuit het hulpmiddel in plaats van een tweede lokale agentbeurt te starten. Bezorgingsbewerkingen bevinden zich buiten modeltranscripten; een vastgelegd antwoord wordt alleen als nevenartefact bewaard, terwijl het hulpmiddelresultaat eigenaar is van de modelcontext. Als de Gateway na het in de wachtrij plaatsen opnieuw wordt gestart, kan de bezorging worden hersteld, maar volgt een later antwoord de normale inkomende routering omdat de proceslokale wachter verdwenen is. Ongevraagde inkomende berichten gaan altijd verder via het normale kanaalrouteringspad.

Gebruik het gedeelde hulpmiddel `message` wanneer je al een expliciet onbewerkt kanaaldoel hebt of een kanaalspecifieke actie nodig hebt. Gespreksverwijzingen zijn beperkt tot de actieve agent en moeten via `conversations_list` worden verkregen, niet worden samengesteld uit sessiesleutels.

In de Code Mode gebruiken de gesprekshulpmiddelen hun exacte Gateway-uitvoercontracten opnieuw. Eén `exec`-cel kan adressen weergeven, een geretourneerde `conversationRef` selecteren en `conversations_send` of `conversations_turn` aanroepen; het normale hulpmiddelenbeleid en goedkeuringen blijven van toepassing op de geneste aanroepen.

## Berichten tussen sessies verzenden

`sessions_send` voert een andere sessie uit op dezelfde Gateway en wacht optioneel op het antwoord. De `sessionKey`, `label` of `agentId` ervan selecteert lokale modelcontext, geen externe bestemming. Het resulterende antwoord kan nog steeds via de bestaande bezorgingscontext van de aanvrager of het doel worden aangekondigd; dat bestaande gedrag blijft ongewijzigd. Gebruik voor exacte externe bezorging een gesprekshulpmiddel of `message` met een expliciet kanaal en doel.

- **Verzenden zonder op het resultaat te wachten:** stel `timeoutSeconds: 0` in om in de wachtrij te plaatsen en onmiddellijk terug te keren.
- **Op antwoord wachten:** stel een time-out in en ontvang het antwoord rechtstreeks.

Chat­sessies die tot een thread zijn beperkt, zoals sleutels die eindigen op `:thread:<id>`, zijn geen geldige doelen voor `sessions_send`. Gebruik voor coördinatie tussen agents de sessiesleutel van het bovenliggende kanaal, zodat via hulpmiddelen gerouteerde berichten niet in een actieve, voor mensen zichtbare thread verschijnen.

Berichten en A2A-vervolgantwoorden worden in de ontvangende prompt (`[Inter-session message ... isUser=false]`) en in de transcriptherkomst gemarkeerd als gegevens tussen sessies. De ontvangende agent moet ze behandelen als via hulpmiddelen gerouteerde gegevens, niet als een rechtstreeks door de eindgebruiker geschreven instructie.

Nadat het doel heeft geantwoord, kan OpenClaw een **terugantwoordlus** uitvoeren waarin de agents tot aan de ingebouwde limiet afwisselend berichten sturen. De doelagent kan `REPLY_SKIP` antwoorden om vroegtijdig te stoppen.

Geef `watch: true` door om de afzender ook als waarnemer van statuswijzigingen bij het doel te registreren: wanneer een andere actor later een rechtstreeks menselijk bericht naar het doel stuurt of het doel ervan wijzigt, ontvangt de afzender een systeemmelding die verwijst naar `session_status` `changesSince`. Registratie vindt plaats na succesvolle routering, richt zich op de sessie die het bericht daadwerkelijk heeft ontvangen en begint bij de huidige statusversie ervan, zodat alleen latere wijzigingen meldingen opleveren. Het resultaat rapporteert `watched: true` wanneer de registratie is geslaagd. Zie [Bewustzijn van sessiestatus](/nl/concepts/session-state).

## Status- en orkestratiehulpmiddelen

`session_status` is het lichtgewicht, met `/status` overeenkomende hulpmiddel voor de huidige of een andere zichtbare sessie. Het rapporteert gebruik, tijd, model-/runtimestatus en gekoppelde context van achtergrondtaken wanneer die aanwezig is. Net als `/status` kan het schaarse token-/cachetellers aanvullen vanuit de nieuwste invoer voor transcriptgebruik, en `model=default` wist een overschrijving per sessie. Gebruik `sessionKey="current"` voor de huidige sessie van de aanroeper; zichtbare clientlabels zoals `openclaw-tui` zijn geen sessiesleutels.

Wanneer routemetadata beschikbaar is, bevat `session_status` ook een zichtbaar `Route context`-JSON-blok en bijbehorende gestructureerde `details`-velden. Deze velden onderscheiden de sessiesleutel van de route die momenteel de live-uitvoering afhandelt:

- `origin` is waar de sessie is aangemaakt, of de provider die is afgeleid uit een voorvoegsel van een afleverbare sessiesleutel wanneer oudere status geen opgeslagen oorsprongsmetadata bevat.
- `active` is de huidige route van de live-uitvoering. Deze wordt alleen gerapporteerd voor de live of huidige sessie die nu wordt afgehandeld.
- `deliveryContext` is de permanente afleveringsroute die bij de sessie is opgeslagen en die OpenClaw opnieuw kan gebruiken voor latere aflevering, zelfs wanneer het actieve oppervlak verschilt.

## Wijzigingen in de sessiestatus

OpenClaw houdt een duurzaam signaallogboek bij van wezenlijke wijzigingen in de sessiestatus (rechtstreekse menselijke berichten aan bewaakte sessies, resultaten van onderliggende uitvoeringen, wijzigingen in doelen, Compaction). `sessions_list`-rijen en `session_status` stellen de `stateVersion` van de sessie beschikbaar, en `session_status` accepteert `changesSince: <version>` om de getypeerde gebeurtenissen na die versie te retourneren, met exacte `historyGap`-signalering wanneer de aangevraagde versie ouder is dan de bewaarde geschiedenis. Bewakers — automatisch bovenliggende spawns, expliciet `sessions_send watch: true` — ontvangen één samengevoegde melding over verouderde status wanneer een andere actor een bewaakte sessie wijzigt.

Statuswijzigingsgebeurtenissen laten herhaalde sessie-/agent-ID's weg en stellen alleen voor het model nuttige payloadvelden beschikbaar (`outcome`, `channel` of `turns`). De samenvatting van de gebeurtenis en de actor-/uitvoerings-ID's blijven beschikbaar voor reconciliatie.

Zie [Bewustzijn van sessiestatus](/nl/concepts/session-state) voor het volledige model: gebeurtenistypen, registratie van bewakers, het antispamprotocol voor meldingen, de reconciliatiestroom en de huidige limieten.

`sessions_yield` beëindigt bewust de huidige beurt, zodat het volgende bericht de vervolggebeurtenis kan zijn waarop je wacht. Gebruik dit na het starten van subagents wanneer je wilt dat voltooiingsresultaten als het volgende bericht binnenkomen in plaats van pollinglussen te bouwen.

`subagents` is de sessieboomweergave van systeemeigen subagentuitvoeringen en het gedeelde register van achtergrondtaken. `action: "list"` rapporteert actieve/recente subagents plus ACP-, CLI-/media- en Cron-taken binnen het bereik. `action: "cancel"` accepteert een geretourneerde `taskId` en kan alleen werk stoppen binnen de sessieboom waarover de aanroeper zeggenschap heeft; subagents op eindniveau kunnen geen taak van een andere sessie annuleren.

## Subagents starten

`sessions_spawn` maakt standaard een geïsoleerde sessie voor een achtergrondtaak aan. Dit is altijd niet-blokkerend; de functie retourneert onmiddellijk een `runId` en `childSessionKey`. Systeemeigen subagentuitvoeringen ontvangen de gedelegeerde taak in het eerste zichtbare `[Subagent Task]`-bericht van de onderliggende sessie, terwijl de systeemprompt alleen uitvoeringsregels en routeringscontext voor subagents bevat.

Belangrijkste opties:

- `runtime: "subagent"` (standaard) of `"acp"` voor agents van externe harnassen.
- `model`- en `thinking`-overschrijvingen voor de onderliggende sessie.
- `thread: true` om de spawn aan een chatthread te koppelen (Discord, Slack enzovoort).
- `sandbox: "require"` om sandboxing voor de onderliggende sessie af te dwingen.
- `context: "fork"` voor systeemeigen subagents wanneer de onderliggende sessie het transcript van de huidige aanvrager nodig heeft; laat dit weg of gebruik `context: "isolated"` voor een schone onderliggende sessie. `context: "fork"` is alleen geldig met `runtime: "subagent"`. Aan threads gekoppelde systeemeigen subagents gebruiken standaard `context: "fork"`, tenzij `threadBindings.defaultSpawnContext` anders aangeeft.
- `visible: true` om een permanente dashboardsessie te maken in plaats van een verborgen subagentsessie. Zichtbare spawns ondersteunen een expliciet model, een werkmap, een transcriptfork van dezelfde agent en een optionele [beheerde worktree](/nl/concepts/managed-worktrees); zie [Subagents](/nl/tools/subagents#tool-parameters) voor de exacte compatibiliteitsbeperkingen.

Standaard krijgen subagents op eindniveau geen sessietools. Wanneer `maxSpawnDepth >= 2` ontvangen subagents die op diepte 1 als orchestrator fungeren bovendien `sessions_spawn`, `subagents`, `sessions_list` en `sessions_history`, zodat ze hun eigen onderliggende agents kunnen beheren. Uitvoeringen op eindniveau krijgen nog steeds geen recursieve orchestratietools.

Na voltooiing plaatst een aankondigingsstap het resultaat in het kanaal van de aanvrager. Bij de aflevering van voltooiingen blijft routering naar een gekoppelde thread/onderwerp behouden wanneer die beschikbaar is. Als de oorsprong van de voltooiing alleen een kanaal identificeert, kan OpenClaw nog steeds de opgeslagen route van de sessie van de aanvrager (`lastChannel` / `lastTo`) hergebruiken voor rechtstreekse aflevering.

Zie [ACP-agents](/nl/tools/acp-agents) voor ACP-specifiek gedrag.

## Zichtbaarheid

Sessietools hebben een beperkt bereik om te begrenzen wat de agent kan zien:

| Niveau   | Bereik                                                      |
| ------- | ---------------------------------------------------------- |
| `self`  | Alleen de huidige sessie                                   |
| `tree`  | Huidige + gestarte sessies; leesbewerkingen omvatten bewaakte groepen van dezelfde agent |
| `agent` | Alle sessies voor deze agent                                |
| `all`   | Alle sessies (agentoverschrijdend indien geconfigureerd)                   |

De standaardwaarde is `tree`. Sessies in een sandbox worden ongeacht de configuratie beperkt tot `tree`.
Met de standaardwaarde `session.dmScope: "main"` zorgt groepsactiviteit ervoor dat bewaakte
groepssessies van dezelfde agent leesbaar zijn vanuit de hoofdsessie.

## Verder lezen

- [Sessiebeheer](/nl/concepts/session): routering, levenscyclus, onderhoud
- [Subagents](/nl/tools/subagents): levenscyclus en aflevering van onderliggende sessies
- [ACP-agents](/nl/tools/acp-agents): starten via externe harnassen
- [Multi-agent](/nl/concepts/multi-agent): multi-agentarchitectuur
- [Gateway-configuratie](/nl/gateway/configuration): configuratieopties voor sessietools

## Gerelateerd

- [Sessiebeheer](/nl/concepts/session)
- [Sessies opschonen](/nl/concepts/session-pruning)
