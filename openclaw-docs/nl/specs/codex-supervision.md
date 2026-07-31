---
read_when:
    - Codex-sessieontdekking, -voortzetting of -archiefgedrag ontwerpen
    - De gebruikersinterface van de systeemeigen sessiecatalogus of Gateway-RPC's wijzigen
    - Codex-supervisie uitbreiden over gekoppelde nodes
summary: Architectuur en productgrens voor het beheren van native Codex-sessies vanuit OpenClaw.
title: Codex-toezicht
x-i18n:
    generated_at: "2026-07-27T06:10:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5e259badc8f7fdec6fa093785a1dd04394e12287ae61f00474bcd45e7b95352d
    source_path: specs/codex-supervision.md
    workflow: 16
---

# Codex-toezicht

## Doel

Met Codex-toezicht kan een OpenClaw-operator native Codex-sessies ontdekken en,
wanneer dit veilig is, een lokale branch maken via de normale OpenClaw Chat-interface.
Codex App Server blijft de eigenaar van de thread en de modellus. OpenClaw levert de
vlootcatalogus, de geauthenticeerde operatorinterface, de sessiekoppeling en de kanaalbezorging.

De functie behoort tot de officiële `codex`-plugin. Er is geen afzonderlijke
Supervisor-plugin of tweede implementatie van het Codex-protocol.

## Productgrens

De catalogus wordt geregistreerd wanneer de Codex-plugin actief is, tenzij native
sessiedetectie expliciet is uitgeschakeld met:

```text
plugins.entries.codex.config.sessionCatalog.enabled = false
```

Schakel toezichtstools voor agents in met:

```text
plugins.entries.codex.config.supervision.enabled = true
```

Het actieve initiële product is bewust beperkter dan het vlootplan voor de
lange termijn:

- Toon alleen niet-gearchiveerde Codex-threads.
- Groepeer lokale rijen en aangemelde rijen van gekoppelde nodes op stabiele hostidentiteit.
- Maak een normale, modelvergrendelde Chat-branch van een opgeslagen of inactieve Gateway-lokale
  thread, start de volledige Codex-harnessthread ervan bij de eerste beurt, of open de Chat
  die voor een eerdere branch is gemaakt.
- Archiveer een opgeslagen of inactieve Gateway-lokale thread alleen na expliciete
  bevestiging dat er geen andere runner is.
- Toon actieve lokale bronnen zonder besturingselementen voor een nieuwe branch of archivering, terwijl
  een bestaande bewaakte Chat nog steeds kan worden geopend.
- Toon de nieuwste rijen per host in de hoofdzijbalk, behoud de volledige catalogus op
  de sessiepagina en bied begrensde, met cursors gepagineerde transcriptlezingen voor
  lokale rijen en rijen van gekoppelde nodes.
- Isoleer catalogusfouten per host.

De catalogus is de niet-gearchiveerde verzameling. Een rij daarin kan nog steeds
de beurtstatus inactief, actief, `notLoaded` of fout hebben.

Toezicht voor agents blijft opt-in. Begeleide onboarding probeert dit te installeren en in te schakelen
nadat de detectie van een native Codex-installatie is geslaagd en de geselecteerde inferentiebackend
de livecontrole doorstaat, onafhankelijk van welke primaire backend de gebruiker
selecteert. Toezicht wordt alleen geactiveerd wanneer die opportunistische pluginconfiguratie
slaagt. Een expliciet uitgeschakelde plugin, beleidsblokkering of
`supervision.enabled: false` blijft bepalend voor toezichtstools, maar
schakelt de operatorsessiecatalogus niet uit. `sessionCatalog.enabled: false`
schakelt operatorontdekking en catalogusopdrachten voor gekoppelde nodes uit; de Codex-
provider en harness blijven actief.

## Eigenaarschap

De `codex`-plugin is eigenaar van al het gedrag van Codex App Server:

- endpointdetectie en verbindingslevenscyclus
- protocolinitialisatie en versiecontroles
- threads weergeven, lezen, hervatten en archiveren, en gebeurtenisafhandeling
- bruggen voor goedkeuring en gebruikersinvoer
- native threadkoppelingen aan OpenClaw-sessies
- handhaving van uitsluitend Codex-modellen en -harness na voortzetting

De Control UI en Gateway gebruiken die service waarvan de plugin eigenaar is. Ze lezen
Codex-rolloutbestanden niet rechtstreeks en implementeren geen andere App Server-client.

De standaard lokale topologie is:

```text
Codex Desktop -> privé-stdio App Server -> Codex-home van gebruiker
                                             ^
OpenClaw Codex-plugin -> App Server-verbinding voor toezicht
  (standaard beheerde stdio voor de home van de gebruiker; expliciete appServer-instellingen worden gerespecteerd)
  -> passieve broncatalogus en lezen
  -> snapshot vastzetten -> canonieke branch met appServer-bron
  -> injectie van zichtbare geschiedenis en elke latere bewaakte Chat-beurt

Gewone OpenClaw Codex-sessies -> standaard beheerde stdio voor de agent-home
  -> gewone volledige harnessthreads -> OpenClaw Chat en kanaalbezorging
```

Het inschakelen van toezicht verandert de gewone Codex-harness niet: deze blijft
standaard agentspecifiek. De afzonderlijke toezichtverbinding gebruikt standaard
beheerde stdio voor de home van de gebruiker, zodat de catalogus- en snapshotbewerkingen native
opgeslagen threads zien. Expliciete verbindingsinstellingen voor `appServer` worden gerespecteerd. Wanneer
`homeScope` niet is ingesteld, stelt de toezichtverbinding dit in op `"user"` voor stdio
of Unix en op `"agent"` voor WebSocket. Stel `appServer.homeScope: "user"`
alleen expliciet in wanneer de gewone harness ook de native Codex-
home moet delen. Een Chat die vanuit de Codex-zijbalkgroep is overgenomen, vormt de uitzondering: de privé-
toezichtkoppeling ervan houdt bronlezingen, het maken van de canonieke branch en latere
beurten op de toezichtverbinding. Livestatus en eigenaarschap blijven
proceslokaal; een thread die onbekend is bij het toezichtproces van OpenClaw is `notLoaded`,
zelfs wanneer Codex Desktop deze actief uitvoert.

Codex heeft een experimentele canonieke lokale daemon met een afzonderlijk
door het installatieprogramma beheerd bootstrapcontract. Deze functie mag die daemon niet impliciet opstarten, claimen
of veronderstellen.

## Catalogusstroom

De generieke Gateway-methode `sessions.catalog.list` stuurt door naar de `codex`-
catalogusprovider, die altijd `archived: false` aanvraagt en App Server
de standaard voor interactieve bronnen laat toepassen: `cli`, `vscode`, Atlas en ChatGPT. De methode
combineert:

1. Gateway-lokale `thread/list`-resultaten van de App Server voor toezicht,
   die standaard beheerde stdio voor de home van de gebruiker gebruikt.
2. `codex.appServer.threads.list.v1`-resultaten van elke verbonden, aangemelde node.

Transcriptselectie gebruikt lokaal `thread/turns/list` met `itemsView: "full"` of
de geversioneerde opdracht `codex.appServer.thread.turns.list.v1` op de geselecteerde
node. Elk antwoord bevat maximaal 20 opgeslagen beurten plus ondoorzichtige
voorwaartse/achterwaartse cursors. De Control UI vraagt pagina's van nieuw naar oud op, geeft elke pagina in
chronologische volgorde weer en voegt oudere pagina's vooraan toe. Deze valt nooit terug op een
onbegrensde `thread/read`. OpenClaw weigert ook elke geserialiseerde itempagina groter dan
20 MiB voordat deze het node- of Gateway-transport kan passeren.

De native implementatie voor gekoppelde macOS-nodes ondersteunt alleen een niet-ingestelde/standaard of
expliciete `appServer.transport: "stdio"` met een niet-ingestelde/standaard toezichtscope of
expliciete `appServer.homeScope: "user"`. Deze geeft de geconfigureerde `command`, `args`
en genormaliseerde `clearEnv` door aan het onderliggende proces. Met `"unix"`, `"websocket"`
of expliciete `homeScope: "agent"` adverteert deze noch de catalogusmogelijkheid
noch de opdracht; rechtstreekse aanroep wordt ook veilig geweigerd. Deze mag nooit de Codex-home van de gebruiker
blootstellen voor een agentspecifieke configuratie of lokale stdio gebruiken in plaats van een
expliciet endpoint.

De catalogusprojectie normaliseert identificatoren, titel, cwd, status, actieve wacht-
vlaggen, tijdstempels, bron, modelprovider, Codex-versie en Git-branch. Deze
retourneert geen transcriptvoorbeelden, beurten, rolloutpaden, Codex-homepaden,
externe Git-repositories, commit-SHA's, onbewerkte endpoints of onbewerkte App Server-fouten. Transcript-
antwoorden bevatten alleen de expliciet aangevraagde App Server-itempagina en de
ondoorzichtige cursors ervan.

Hostfouten blijven lokaal voor elk hostresultaat. Een offline node of niet-beschikbare
lokale App Server verwijdert gezonde hosts niet van de pagina. Connectiviteit is een
hosteigenschap, geen threadstatus: een mislukt hostresultaat bevat geen nieuwe
sessierijen en projecteert `offline` niet op native threads.

De Control UI vraagt progressieve catalogusupdates aan. Elke lokale of gekoppelde host
verschijnt wanneer de eigen App Server-vermelding is voltooid; het samengevoegde antwoord blijft
de compatibiliteits- en herstelsnapshot. De zichtbare pagina wordt opnieuw afgestemd na
connectiviteitswijzigingen, bij focus en maximaal elke 30 seconden, met een snellere ronde
na wijzigingen. Native Codex-sessies die in een andere client zijn gemaakt, worden daardoor
uiteindelijk ontdekt zonder ze in OpenClaw-opslag te importeren.

Catalogusdetectie is passief. Het weergeven of lezen van metadata mag
`thread/resume` niet aanroepen, de OpenClaw-client niet abonneren op live threadverzoeken en
geen goedkeuring beantwoorden.

Zoeken gebeurt alleen op titel en is niet hoofdlettergevoelig. Voor elke geretourneerde cataloguspagina scannen
de Gateway en de gekoppelde Mac een begrensd aantal native pagina's zonder
de zoekopdracht aan App Server door te geven, omdat native zoeken ook overeenkomsten in transcript-
voorbeelden kan vinden. Met de geretourneerde native cursor kunnen aanroepers de scan voortzetten.

## Grens van de operator-CLI

De plugin registreert drie door Gateway ondersteunde shellopdrachten:

```text
openclaw codex sessions [--search <text>] [--host <id>] [--limit <count>] [--cursor <cursor>] [--json] [gateway-options]
openclaw codex continue <thread-id> [--json] [gateway-options]
openclaw codex archive <thread-id> --confirm-no-other-runner [--json] [gateway-options]
```

`[gateway-options]` is `--url <url>`, `--token <token>`, `--timeout <ms>` en
de overgenomen `--expect-final`-schakelaar. De time-out voor sessievermelding is standaard 75,000 ms;
voor voortzetten en archiveren is deze standaard 30,000 ms;
`--expect-final` heeft geen aanvullend effect op deze unaire RPC's. Sessies zoeken
gebeurt alleen op titel en is niet hoofdlettergevoelig; elk antwoord scant een begrensde native
paginaketen en `--cursor` gaat verder met oudere resultaten. De limiet is standaard 50 per host,
accepteert 1 tot en met 100, en een cursor vereist één stabiele `--host`-
bestemming. Geen enkele opdracht accepteert
een optie voor gearchiveerd/opnemen van gearchiveerde items. Alleen `sessions` kan gekoppelde hosts als doel gebruiken;
`continue` en `archive` sturen altijd `hostId: "gateway:local"`, en voor archiveren
is de expliciete bevestigingsvlag vereist.

De shellnaamruimte is niet de `/codex`-runtimenaamruimte in Chat. In
het bijzonder geeft `/codex sessions --host <node>` Codex CLI-sessiebestanden op één
node weer, geeft `/codex threads` App Server-threads voor de huidige gespreksverbinding
weer, en wijzigt `/codex resume` of `/codex bind` de koppeling van dat gesprek.
Die opdrachten vervangen `sessions.catalog.continue` niet en er is
geen runtimeopdracht `/codex continue` of `/codex archive`.

## Lokale voortzetting

Voor een opgeslagen of inactieve Gateway-lokale rij roept de UI
`sessions.catalog.continue` aan met `catalogId: "codex"` plus de host- en thread-
id's. De plugin:

1. Hergebruikt de bestaande bewaakte Chat wanneer de bron er al een heeft.
2. Projecteert anders begrensde gebruikers- en assistentgeschiedenis tot en met de laatste
   terminale opgeslagen beurt van de bron (voltooid, onderbroken of mislukt) naar een nieuwe
   OpenClaw Chat en registreert een wachtende harnessbranch.
3. Slaat het wachtende beleid voor uitsluitend Codex-modelvergrendeling op, niet een concrete model- of
   providerselectie, plus de privé-scope van de toezichtverbinding, en
   retourneert de OpenClaw-`sessionKey`.

De geschiedenisprojectie selecteert de nieuwste staart van zichtbare gebruikers- en assistent-
berichten, met harde limieten van 200 berichten, in totaal 512 KiB UTF-8-tekst en
64 KiB per bericht. Deze vervangt invoer met afbeeldingen en lokale afbeeldingen door
`[Image attachment]`, kopieert nooit afbeeldingspayloads of paden, en laat redeneringen,
toolaanroepen en toolresultaten weg.

De UI navigeert met die sessiesleutel naar de normale Chat. Er bestaat nog geen canonieke harness-
thread. Bij de eerste normale Chat-beurt installeert de harness de echte
Codex-handlers voor goedkeuring, uitvraag, gebeurtenissen en bezorging, en vervolgens:

1. Gebruikt de toezichtverbinding om native `thread/fork` aan te roepen zonder een model-
   of provideroverride en zet de opgeslagen bronsnapshot vast. De huidige
   `ConfigManager`-status van Codex selecteert het model en de provider, en het branchantwoord
   meldt het werkelijke paar. Als het model afwijkt van het laatst geregistreerde model
   in de bron, geeft Codex de normale waarschuwing voor modelverschillen.
2. Start op diezelfde verbinding de canonieke volledige Codex-harnessthread met
   `threadSource: "appServer"`, de cwd, het beleid, de configuratie en de omgeving van OpenClaw, het
   volledige oppervlak aan OpenClaw-harnesstools, en exact het model en de provider
   die door de branch voor deze initiële start zijn geretourneerd.
3. Injecteert de begrensde zichtbare gebruikers- en assistentgeschiedenis via die
   verbinding, legt de canonieke koppeling vast zonder de toezichtscope ervan te verwijderen,
   voert de beurt uit en archiveert de tijdelijke branch.

Vóór de eerste beurt is de Chat een vergrendelde, wachtende branch met een zichtbare
geschiedenisspiegel; daarna loopt elke modelbeurt via de canonieke Codex-
harnessthread op de supervisieverbinding. De branch is geen volledige native
kloon van de rollout: bronredenering, toolaanroepen en toolresultaten worden bewust
weggelaten. Als het vastzetten van de snapshot of het maken van de canonieke thread mislukt, blijft de wachtende
branch opnieuw uitvoerbaar. Een race bij het koppelen, uitgeschakelde supervisie of een niet-beschikbare
of niet-overeenkomende supervisieverbinding zorgt dat de bewerking vóór de beurt gesloten mislukt, in plaats
van terug te vallen op de gewone agent-home-harness.

Dit garandeert selectie door Codex, niet het behoud van het historische model
van de bron. Het door de fork geretourneerde paar wordt gebruikt om de canonieke thread
te starten, en Codex bewaart het native model en de provider van die thread. Latere hervattingen
laten OpenClaw-overschrijvingen voor model en provider weg, zodat Codex het opgeslagen paar herstelt.
Als een afzonderlijke native Codex-besturing de canonieke thread wijzigt, accepteert OpenClaw
die native opgeslagen selectie. Het buitenste OpenClaw-model en de fallbackketen
vervangen deze nooit.

Modelwijzigingen, sessieverwijdering en bewerkingen voor het resetten of nieuw maken van sessies mislukken gesloten
voor de gesuperviseerde, modelvergrendelde Chat. Het wijzigen van `/codex model <model>`, `/codex
bind`, `/codex resume` (inclusief Node `--bind here`), en `/codex detach` of
`/codex unbind` mislukt eveneens gesloten, omdat deze de koppeling vervangen of wissen. De
query `/codex model` en `/codex fast`, `/codex permissions` en `/codex
threads` blijven beschikbaar. De agenttool `codex_threads` kan geen nieuwe
fork koppelen of de gekoppelde native thread archiveren. Alleen-lezen van lijsten en metadata blijft
beschikbaar; transcriptvelden vereisen `supervision.allowRawTranscripts`, terwijl
hernoemen, dearchiveren, een losgekoppelde fork en het archiveren van een niet-gerelateerde thread
`supervision.allowWriteControls` vereisen. Geen van beide opties kan de vergrendelde koppeling vervangen.
Als het OpenClaw-item werd verwijderd of gereset, zou anders de native
koppeling worden weggegooid en een generieke thread achter een op Codex lijkende sessie worden gemaakt of toegestaan.
Retentieonderhoud behoudt daarom modelvergrendelde items, zelfs wanneer ze
de normale limieten voor leeftijd, aantal of schijfbudget overschrijden. Ook bij het uitschakelen of verwijderen van de
eigenaar-Plugin blijven de vergrendeling en de markering van Plugin-eigendom behouden. De Chat blijft
niet beschikbaar en mislukt gesloten totdat dezelfde Plugin opnieuw wordt ingeschakeld; opschoning
zet deze nooit om in een gewone modelsessie.

De bron wordt door deze actie nooit hervat of gewijzigd. De tijdelijke fork zet een
snapshot vast; deze is niet de duurzame voortzettingsthread. Door bij de eerste beurt een afzonderlijke
canonieke harnessthread te starten, wordt voorkomen dat OpenClaw een
concurrerende schrijver naar de bron wordt, alleen omdat de proceslokale status een beurt
van Desktop-eigendom niet heeft gezien. De zichtbare-geschiedenisspiegel en vastgezette snapshot kunnen werk weglaten
dat in een actieve bron nog niet is voltooid. De oorspronkelijke CLI-, VS Code-,
Atlas- of ChatGPT-bron blijft beschikbaar voor zowel native catalogi als OpenClaw-catalogi.
De canonieke branch blijft een native Codex-thread in de supervisieopslag,
maar native clients kunnen het brontype `appServer` filteren, dus zichtbaarheid in Codex Desktop
is geen contract.

## Archiveringsgedrag

Voor een opgeslagen of inactieve Gateway-lokale rij vereist `sessions.catalog.archive` met
`catalogId: "codex"`
expliciet `confirmNoOtherRunner: true`, leest deze opnieuw de actuele proceslokale
status, gaat deze alleen door voor `idle` of `notLoaded`, roept native `thread/archive` aan
en retourneert deze alleen succes nadat Codex de bewerking accepteert. De rij verlaat vervolgens
de niet-gearchiveerde catalogus.

Een actieve status of foutstatus uit de nieuwe leesbewerking weigert archivering. Dat geldt ook voor een
initialiserende of wachtende gesuperviseerde branch van de bron: de eerste Chat-beurt
moet de canonieke branch concretiseren voordat de bron kan worden gearchiveerd. Een
bekende actieve eigenaar van een OpenClaw-koppeling voor het exacte doel of een niet-gearchiveerde
voortgebrachte afstammeling weigert archivering eveneens. OpenClaw pagineert de experimentele
relatie `thread/list ancestorThreadId` van Codex en mislukt gesloten bij aanvraag- of antwoordfouten,
cursor- of threadcycli en het uitputten van veiligheidslimieten. Native archivering kan
geladen werk van de bovenliggende thread en afstammelingen afsluiten, dus archivering is geen snelkoppeling
voor onderbreking. De leesbewerking, opsomming van afstammelingen en archiveringsaanroepen zijn niet atomair.
Een onafhankelijke client kan nog steeds eigenaar zijn van of werk starten op een rij die lokaal inactief of
`notLoaded` lijkt. De bevestiging dat er geen andere uitvoerder is, dekt onbekende clients en
die race af totdat Codex voorwaardelijke archivering of een procesoverschrijdende lease heeft.
Archivering via een gekoppelde Node is verboden.

Er is geen gearchiveerde weergave in de Codex-catalogus. Een thread die met
`thread/unarchive` in een ander, door de eigenaar geautoriseerd Codex-oppervlak wordt hersteld, komt opnieuw in aanmerking
voor de niet-gearchiveerde catalogus.

## Veiligheid van actieve threads

Codex serialiseert wijzigingen voor een thread tussen clients van één App Server, maar stelt
geen exclusieve procesoverschrijdende lease voor een uitvoerder of goedkeuringseigenaar beschikbaar.
Onafhankelijke stdio App Servers kunnen aan dezelfde rollout toevoegen, terwijl elke server
alleen zijn eigen status in het geheugen ziet. Goedkeuringsverzoeken kunnen ook elke abonnee
van één server bereiken, waarbij het eerste geldige antwoord het verzoek voltooit.

Daarom:

- passieve catalogusclients abonneren zich niet op goedkeuringen en wijzen deze niet automatisch af
- rijen die momenteel als actief worden gemeld, bieden noch een nieuwe branch noch Archiveren
- een niet-toegewezen bron wordt een branch met zichtbare geschiedenis waarvan de canonieke harness-
  thread de bron nooit hervat
- `notLoaded` wordt weergegeven als activiteit onbekend en kan alleen worden gearchiveerd na
  geïnformeerde bevestiging dat er geen andere uitvoerder is
- lokale archivering vereist die bevestiging plus een nieuwe leesbewerking voor `idle` of `notLoaded`,
  waarbij de protocolrace tussen lezen en archiveren wordt erkend

Onderbreking en overdracht tussen meerdere clients zijn toekomstige productbeslissingen. Ze worden niet
geïmpliceerd door het tonen van een actieve rij.

## Grens van gekoppelde Nodes

Node invoke werkt momenteel alleen via verzoek en antwoord. Het kan veilig begrensde
catalogusmetadata en pagina's met transcriptbeurten retourneren, maar kan niet de langlopende gebeurtenisstroom, goedkeuringsverzoeken,
toolaanroepen, annulering en assistentdelta's dragen die vereist zijn voor een Codex-
harnessuitvoering.

Het Node-contract ondersteunt daarom lijsten en pagina's met transcriptbeurten. Externe
rijen blijven leesbaar, maar **Doorgaan** en **Archiveren** zijn niet beschikbaar, ongeacht de inactieve status. Een
echte externe voortzetting vereist een uitvoerder aan de Node-zijde en een streamingbridge die
dezelfde invarianten voor goedkeuring en koppeling behoudt als de lokale harness.

## Machtigingen

Elke computer meldt zich lokaal aan. Het inschakelen van de Gateway geeft een andere
Node geen toestemming om de Codex-metadata ervan te lezen. De Node-capaciteit moet de normale goedkeuring
voor koppeling en opdrachtbeleid doorlopen.

Voor het weergeven van de fleet en het bekijken van transcripties wordt het Gateway-bereik `operator.write`
gebruikt, omdat hierbij gekoppelde Nodes worden aangeroepen. Lokale voortzetting en archivering zijn
geverifieerde operatoracties en blijven onderworpen aan host- en statuscontroles.

Toegang voor autonome agents en zelfstandige MCP is afzonderlijk. De meegeleverde
toolcontracten `codex_endpoint_probe`, `codex_sessions_list`, `codex_session_read`,
`codex_session_send` en `codex_session_interrupt` blijven eigendom
van de Plugin `codex`. Wanneer supervisie is ingeschakeld, vereisen onbewerkte transcriptleesbewerkingen van `codex_threads`
en van transcripties afgeleide lijstvelden ook
`supervision.allowRawTranscripts`; elke fork-, hernoem-, archiveer-
of dearchiveerbewerking van `codex_threads` vereist `supervision.allowWriteControls`. Beide beleidsregels zijn standaard
uitgeschakeld.

## Compatibiliteit

`openclaw doctor --fix` migreert de meegeleverde configuratie `plugins.entries.codex-supervisor`,
inclusief eindpunten en beleidsregels voor transcripties en schrijfbewerkingen, plus verwijzingen voor het toestaan of weigeren van Plugins naar
`plugins.entries.codex.config.supervision`. Expliciete canonieke bestemmingswaarden
winnen bij conflicten. Runtimecode gebruikt na migratie alleen de canonieke Plugin-vorm `codex`.

De officiële Plugin behoudt precies vijf Supervisor-compatibiliteitstools:
`codex_endpoint_probe`, `codex_sessions_list`, `codex_session_read`,
`codex_session_send` en `codex_session_interrupt`. De sessielijst bevat standaard alleen geladen sessies;
er is geen parameter `loaded_only`. `include_stored: true` voegt
niet-gearchiveerde rijen uit de statusdatabase toe, per eindpunt begrensd door `max_stored_sessions`
(standaard 200, geaccepteerd bereik 1 tot en met 1,000); geladen rijen worden door die
instelling niet begrensd. Van transcripties afgeleide velden en leesbewerkingen blijven afgeschermd door
`allowRawTranscripts`; verzenden en onderbreken blijven afgeschermd door `allowWriteControls`.

Compatibiliteitsverzending start of hervat nooit een inactieve thread. `mode: "start"` wordt
altijd geweigerd; `"auto"` en `"steer"` sturen alleen een leesbare actieve beurt aan.
Onderbreking vereist eveneens een actieve leesbare beurt. Voortzetting van een inactieve thread wordt
naar de native Codex-catalogus gerouteerd, zodat de volledige harness eigenaar is van goedkeuringen, tools en de koppeling.
De zelfstandige verouderde MCP-adapter haalt dezelfde tools uit de officiële
Plugin en is het enige pad dat de behouden verouderde omgevingsvariabelen voor beleid respecteert.

De catalogus-UI, Gateway-methode, Node-capaciteit en CLI-registratie van juli waren
niet uitgebracht onder de oude Plugin-id. Ze gaan rechtstreeks over naar eigendom van `codex`
zonder een tweede runtimefacade.

## Toekomstig werk

- uitvoerder aan de Node-zijde voor streaming en gebeurtenisbridge voor externe voortzetting
- expliciete leases voor uitvoerder en goedkeuringseigenaar voor gelijktijdige overdracht tussen clients
- externe archivering nadat een lease voor uitvoerdereigendom of gelijkwaardige afscherming bestaat
- onderbreking en uitgebreidere observatie van actieve sessies
- gecontroleerde overdracht tussen Codex Desktop, CLI en OpenClaw

Bladeren door archieven maakt geen deel uit van de geplande supervisiezijbalk. Native Codex-
oppervlakken blijven het herstelpad voor gearchiveerde threads.

## Acceptatietests

- Als supervisie wordt ingeschakeld, worden niet-gearchiveerde lokale sessies weergegeven.
- Gearchiveerde sessies verschijnen nooit in de catalogusrespons of UI.
- Gezonde hosts blijven zichtbaar wanneer een andere host uitvalt; een niet-beschikbare host
  retourneert geen nieuwe rijen in plaats van een offline sessiestatus te verzinnen.
- Een opgeslagen of inactieve lokale rij maakt een Chat-spiegel met een uitsluitend voor Codex geldende
  model-/runtimevergrendeling; de eerste beurt zet een tijdelijke momentopname vast en start de
  canonieke volledige harnessthread, en door Continue opnieuw te kiezen wordt de bestaande Chat geopend.
- Bij de eerste beurt worden model-/provideraanpassingen op de afsplitsing van de momentopname weggelaten en wordt
  de canonieke start vastgezet op het exacte paar dat Codex retourneert, zelfs wanneer Codex waarschuwt
  dat het huidige model afwijkt van het laatst vastgelegde model van de bron.
- In behandeling zijnde en vastgelegde supervisiekoppelingen gebruiken de supervisieverbinding voor
  toegang tot de bron, het maken van de canonieke vertakking en elke latere beurt; gewone
  Codex-sessies blijven aan de agent gekoppeld.
- Bij latere hervattingen worden OpenClaw-model-/provideraanpassingen weggelaten, blijft de
  canonieke persistente selectie van Codex behouden, worden afzonderlijke native wijzigingen aan die thread geaccepteerd
  en wordt nooit het buitenste OpenClaw-model of de fallbackketen gebruikt.
- Het uitschakelen van supervisie of het verliezen van de levenscyclus van de koppeling/verbinding leidt tot veilig falen
  in plaats van de Chat naar het gewone harnas in de thuismap van de agent te verplaatsen.
- Een Chat met supervisie en modelvergrendeling kan niet worden verwijderd zolang deze de native
  koppeling beschermt.
- De Chat spiegelt maximaal 200 berichten van gebruikers en assistenten, in totaal 512 KiB en
  64 KiB per bericht. Afbeeldingen worden tijdelijke aanduidingen; redeneringen uit de bron, toolaanroepen,
  toolresultaten, afbeeldingspayloads en lokale paden worden niet gekloond.
- De vertakkingsflow hervat de bronthre ad nooit.
- De oorspronkelijke bron blijft in aanmerking komen voor beide catalogi. De canonieke native
  vertakking gebruikt het brontype `appServer` en verschijnt niet gegarandeerd in
  Codex Desktop.
- Actieve lokale bronnen kunnen geen vertakking maken of worden gearchiveerd; een bestaande
  Chat met supervisie kan nog steeds worden geopend.
- Rijen waarvan de activiteit onbekend is, kunnen zonder bevestiging worden vertakt; voor archivering is
  expliciete bevestiging vereist dat er geen andere runner actief is.
- Een bron met een initialiserende of in behandeling zijnde vertakking met supervisie kan niet worden gearchiveerd
  totdat de eerste Chat-beurt de canonieke vertakking materialiseert.
- Een bekende actieve eigenaar van een koppeling voor het exacte doel of een niet-gearchiveerde voortgebrachte
  afstammeling blokkeert archivering; fouten bij het opsommen van afstammelingen leiden tot veilig falen, en
  expliciete bevestiging blijft verantwoordelijk voor onbekende clients en de
  raceconditie tussen statuscontrole en archivering.
- Bevestigde lokale archivering van een opgeslagen of inactieve sessie verwijdert de rij nadat de native bewerking is geslaagd.
- Rijen van gekoppelde nodes blijven zichtbaar zonder Continue of Archive.
- Passieve weergave abonneert zich nooit op threadgoedkeuringen en beantwoordt deze ook niet.
- Verouderde Supervisor-configuratie wordt gemigreerd naar de canonieke Codex-configuratiestructuur.
- De verouderde lijst wordt standaard alleen geladen, de opsomming van opgeslagen items houdt zich aan de limiet
  per endpoint en compatibiliteitsverzending start of hervat nooit een inactieve thread.
