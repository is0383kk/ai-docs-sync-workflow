---
read_when:
    - Je wilt dat Codex Desktop- of CLI-sessies in OpenClaw verschijnen
    - Je moet een opgeslagen of inactieve lokale Codex-sessie vertakken of archiveren
    - Je stelt Codex-sessies en transcriptgeschiedenis van gekoppelde nodes beschikbaar
sidebarTitle: Codex supervision
summary: Blader door niet-gearchiveerde native Codex-sessies en gepagineerde transcripties op OpenClaw-nodes
title: Houd toezicht op Codex-sessies
x-i18n:
    generated_at: "2026-07-27T05:40:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f365e3207dff092c3dfd8f7588d60d70a16f0cce484991eb4ab3fc0bd15f8051
    source_path: plugins/codex-supervision.md
    workflow: 16
---

Codex-supervisie is een optionele mogelijkheid van de officiële `codex`-plugin. Deze
toont niet-gearchiveerde bronsessies van Codex CLI, VS Code, Atlas en ChatGPT van
de Gateway-computer en aangemelde gekoppelde computers in de normale
sessiezijbalk en het Chat-venster.

De eerste release houdt het eigenaarschap bewust beperkt:

- Een opgeslagen of inactieve lokale sessie kan een modelvergrendelde OpenClaw-Chat maken
  op basis van de begrensde, opgeslagen geschiedenis van de gebruiker en assistent. Het eerste bericht maakt een
  native snapshotvertakking en start vervolgens de volledige Codex-harnessthread met exact
  het model en de provider die Codex App Server voor die vertakking heeft geselecteerd. Bij latere
  beurten wordt het opgeslagen paar van de canonieke native thread hersteld, terwijl de
  supervisiekoppeling voorkomt dat OpenClaw een andere runtime, een ander
  model of een fallback gebruikt. Een afzonderlijk native Codex-besturingselement kan dat
  opgeslagen paar nog steeds wijzigen. Een reeds gemaakte vertakking opent de bestaande Chat.
- Van een opgeslagen sessie die door een ander Codex-proces is gevonden, is de actuele
  activiteit onbekend. Er kan een vertakking van worden gemaakt, of de sessie kan alleen worden gearchiveerd nadat de operator
  bevestigt dat geen andere Codex-client deze gebruikt.
- Een actieve bron blijft zichtbaar, maar er kan geen vertakking van worden gemaakt en deze kan niet worden gearchiveerd totdat
  de huidige beurt is voltooid. Als deze al een Chat onder supervisie heeft, blijft **Chat openen**
  beschikbaar.
- Een sessie op een gekoppelde Node stelt het opgeslagen transcript beschikbaar via begrensde,
  met cursors gepagineerde App Server-leesbewerkingen. Voortzetting op afstand
  vereist een toekomstige streaming-Node-bridge; archivering op afstand vereist daarnaast
  een lease voor runner-eigenaarschap of gelijkwaardige afscherming.
- Gearchiveerde sessies worden niet vermeld. Een opgeslagen of inactieve lokale sessie kan
  alleen worden gearchiveerd nadat de operator bevestigt dat geen andere Codex-client deze gebruikt.

## Voordat je begint

- Installeer de officiële `@openclaw/codex`-plugin op de Gateway. De OpenClaw-
  app voor macOS kan deze installeren wanneer je Codex-functies inschakelt; bij CLI-installaties kan
  `openclaw plugins install @openclaw/codex` worden uitgevoerd.
- Installeer Codex Desktop of de Codex CLI en meld je aan op elke computer waarvan je
  de sessies wilt weergeven.
- Koppel externe computers als OpenClaw-Nodes. Elke computer moet zich lokaal aanmelden;
  als supervisie alleen op de Gateway wordt ingeschakeld, wordt een andere Node niet geautoriseerd.
- Gebruik een door de eigenaar beheerde Gateway. Sessietitels, werkmappen en Git-
  branches kunnen gevoelige projectinformatie onthullen.

## Supervisie inschakelen

Begeleide `openclaw onboard` en de eerste configuratie in macOS proberen
Codex-supervisie te installeren en in te schakelen nadat een native Codex-installatie is gedetecteerd en
de geselecteerde inferentiebackend met succes is geactiveerd. Codex hoeft niet
de primaire backend te zijn. Supervisie wordt beschikbaar wanneer deze opportunistische
pluginactivering slaagt. De beschikbaarheid van App Server wordt gecontroleerd wanneer
supervisie voor het eerst verbinding maakt. Een expliciete uitschakeling van de Codex-plugin of beleidsblokkering
voorkomt opportunistische activering, en een bestaande expliciete
`supervision.enabled: false` schakelt supervisietools voor agents uit; de
operatorcatalogus blijft geregistreerd zolang de Codex-plugin actief is, tenzij
`sessionCatalog.enabled: false` deze uitschakelt. Deze afzonderlijke schakelaar laat de
Codex-provider, het harness en het supervisiebeleid voor agents ongewijzigd, maar
verwijdert ook de catalogusopdrachten voor het weergeven en lezen van gekoppelde Nodes van deze host.
Bestaande installaties kunnen dezelfde mogelijkheid handmatig inschakelen:

Schakel de `codex`-plugin en de bijbehorende supervisiemogelijkheid in `openclaw.json` in:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          supervision: {
            enabled: true,
          },
        },
      },
    },
  },
}
```

Als `plugins.allow` aanwezig is, neem dan `codex` op. Start de Gateway opnieuw nadat
je de pluginactivering hebt gewijzigd.

Zonder expliciete `appServer`-verbindingsinstellingen gebruikt supervisie een afzonderlijke
beheerde stdio-supervisieverbinding met de native Codex-homemap van de gebruiker. Het
gewone Codex-harness blijft standaard aan de agent gebonden. Hierdoor zijn native
sessies zichtbaar in beide apps zonder dat gewone OpenClaw-beurten de
native Codex-status delen. Stel `appServer.homeScope: "user"` expliciet in als het harness
die status ook moet delen. Supervisie respecteert expliciete `appServer`-verbindingsinstellingen
in plaats van deze te vervangen door de lokale standaardwaarde voor de gebruikershomemap.

Een Chat die vanuit de zijbalkgroep **Codex** is overgenomen, is geen gewone harnesssessie.
De persoonlijke supervisiekoppeling gebruikt de supervisieverbinding voor het lezen van de bron,
het maken van canonieke vertakkingen, het invoegen van geschiedenis en elke latere beurt. Met
de standaard lokale verbinding blijven de native Codex-homemap, authenticatie
en providerconfiguratie van de gebruiker behouden zonder de standaardwaarde voor andere sessies te wijzigen.
Gevolgde, overgenomen Chats nemen ook deel aan [bewustzijn van sessiestatus](/nl/concepts/session-state).

Voor de standaard lokale supervisieverbinding wordt de opslag gedeeld met native
Codex-clients. OpenClaw gaat er niet van uit dat een andere client hetzelfde actieve
App Server-proces deelt, en het eigenaarschap van de native status is procesgebonden. Daarom
wordt een thread die door de App Server voor supervisie als `notLoaded` wordt gemeld, behandeld als
**Opgeslagen / activiteit onbekend**, niet als inactief.

Pas dezelfde aanmelding toe op elke headless Node-host waarvan de sessies moeten verschijnen.
De native OpenClaw-app voor macOS leest dezelfde lokale instelling wanneer deze
de Codex-catalogus aan de gekoppelde Gateway bekendmaakt. Die gekoppelde catalogus van de native Mac ondersteunt
alleen de standaardwaarde of expliciete `appServer.transport: "stdio"` met een niet-ingestelde of
expliciete `appServer.homeScope: "user"`. `command`, `args` en `clearEnv` worden
voor dat stdio-proces gerespecteerd. Als in de Mac-configuratie `"unix"`,
`"websocket"` of `homeScope: "agent"` is geselecteerd, maakt de app de catalogusmogelijkheid
of -opdracht niet bekend, en mislukt een verouderde directe aanroep in plaats van
de Codex-homemap van de gebruiker bloot te stellen of een andere lokale stdio-App Server te starten.

Een nieuw bekendgemaakte Node-opdracht wijzigt het goedgekeurde opdrachtenoppervlak van de Node.
Keur de update goed vanaf de Gateway-host:

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

Niet-gearchiveerde Codex-sessies verschijnen ook in de hoofdzijbalk van de Control UI, gegroepeerd
op host. Selecteer er een om het opgeslagen transcript te lezen. De viewer gebruikt de nieuwste
Codex-API `thread/turns/list` met `itemsView: "full"` en laadt maximaal 20 beurten
per aanvraag; **Oudere transcriptitems laden** volgt de ondoorzichtige App Server-cursor vanaf de nieuwste pagina.
Geladen pagina's worden in chronologische volgorde weergegeven. De viewer laadt nooit een onbegrensde
`thread/read`-geschiedenis. Een pagina boven de transportveiligheidslimiet van 20 MiB
wordt veilig geweigerd in plaats van de Node- of Gateway-verbinding in gevaar te brengen.

Open de groep **Codex** in de normale sessiezijbalk. Deze bevat dezelfde sessies,
gegroepeerd op host. **Meer sessies laden** voegt de volgende pagina toe van elke host
met oudere rijen, en deze toegevoegde rijen blijven behouden tijdens de periodieke vernieuwing van de zijbalk.
Elke host verschijnt zodra de eigen native vermelding is voltooid. De zichtbare pagina
wordt opnieuw afgestemd na wijzigingen in de Node-connectiviteit, wanneer deze opnieuw focus krijgt en maximaal
elke 30 seconden; bij een gewijzigd resultaat volgt sneller een nieuwe controle. Sessies die
in Codex Desktop, de CLI of een andere native client zijn gemaakt, verschijnen daardoor zonder dat
de volledige pagina opnieuw hoeft te worden geladen. De eerste pagina volgt Codex' eigen volgorde op basis van de recentste wijziging,
waardoor een nieuw gemaakte native sessie onmiddellijk in aanmerking komt.
Elke geretourneerde zoekpagina scant een begrensd aantal native pagina's per host in plaats
van de zoekopdracht naar App Server te sturen, omdat native zoeken ook overeenkomsten kan vinden in
transcriptvoorbeelden.

De beschikbaarheid van de host en de threadstatus staan los van elkaar. **Offline** of **Niet beschikbaar**
beschrijft een hostvernieuwing; een niet-beschikbare host retourneert geen nieuwe sessierijen en
wijzigt de native status van een thread niet in `offline`. Sessierijen gebruiken Codex-
statussen zoals `idle`, `active`, `notLoaded` of fout. Een mislukte host verbergt
geen resultaten van gezonde hosts.

De waarschuwing in de zijbalk bevat de foutcode van de catalogus en de veilige onderliggende
Gateway-fout. Open **Instellingen > Automatisering > Plugins > Codex > Native sessiedetectie**
om detectie uit te schakelen zonder Codex uit te schakelen. Vergelijk voor
`NODE_LIST_FAILED` `openclaw nodes list` en **Instellingen > Apparaten**;
de gedetailleerde oorzaak identificeert de fout in de koppelingsopslag, het Node-register, de machtiging of
de Gateway-levenscyclus die moet worden hersteld.

## De operator-CLI gebruiken

De terminal-CLI biedt dezelfde niet-gearchiveerde catalogus en lokale vertakkings-
en archiveringsacties van de Gateway:

```bash
openclaw codex sessions [--search <text>] [--host <id>] [--limit <count>] [--cursor <cursor>] [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
openclaw codex continue <thread-id> [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
openclaw codex archive <thread-id> --confirm-no-other-runner [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
```

Opties voor `openclaw codex sessions`:

- `--search <text>` doorzoekt sessietitels zonder onderscheid tussen hoofdletters en kleine letters.
- `--host <id>` beperkt het antwoord tot één stabiele catalogushost, zoals
  `gateway:local` of `node:<node-id>`.
- `--limit <count>` stelt 1 tot en met 100 rijen per host in; de standaardwaarde is 50.
- `--cursor <cursor>` gaat verder met één hostpagina en vereist daarom `--host`.
- `--json` drukt het gestructureerde Gateway-antwoord af.

Alle drie de opdrachten nemen `--url`, `--token` en `--timeout <ms>` over van de
Gateway-client. Voor het weergeven van sessies geldt standaard 75.000 ms, zodat koude catalogi
van gekoppelde Nodes kunnen worden voltooid; voor voortzetten en archiveren geldt standaard 30.000 ms. Ze bieden ook de gedeelde
schakelaar `--expect-final`, die deze unaire supervisie-RPC's niet wijzigt.
Elke opdracht vereist het Gateway-bereik `operator.write`.
Standaarduitvoer van `-h, --help` is beschikbaar voor elke subopdracht.
Er is geen optie voor gearchiveerde sessies of voor het opnemen daarvan. `sessions` kan gekoppelde
hosts weergeven, maar `continue` en `archive` zijn altijd gericht op `gateway:local`; gekoppelde rijen
kunnen alleen worden weergegeven. Voor archiveren is altijd `--confirm-no-other-runner` vereist.

Deze shellopdrachten verschillen van de runtimeopdrachten `/codex` in de Chat.
`/codex threads [filter]` geeft App Server-threads weer die beschikbaar zijn voor de huidige
gespreksverbinding. `/codex sessions --host <node>` geeft hervatbare Codex-
CLI-sessiebestanden op één Node weer, niet de catalogus van de supervisievloot. `/codex
resume` en `/codex bind` koppelen het huidige gesprek in plaats van een
veilige vertakking onder supervisie te maken, en een modelvergrendelde Chat onder supervisie weigert zulke
koppelingswijzigingen. Er is geen runtimeopdracht `/codex continue` of `/codex archive`.

## Een vertakking maken vanuit een lokale sessie

Kies **Doorgaan als vertakking** bij een opgeslagen of inactieve rij van de Gateway-computer.
OpenClaw maakt een normale Chat-vermelding, spiegelt begrensde gebruikers- en assistentgeschiedenis
tot en met de laatste opgeslagen terminale beurt van de bron (voltooid, onderbroken of
mislukt), registreert een wachtende harnessvertakking en opent de Chat. De algemene modelkiezer
is vergrendeld, maar er is nog geen concreet model of concrete provider geselecteerd. De
bron wordt niet hervat en de canonieke harnessthread wordt nog niet gestart.
Wanneer je de actie herhaalt, wordt de bestaande Chat geopend in plaats van nog een
vertakking te maken.

De spiegeling behoudt het nieuwste zichtbare uiteinde dat binnen alle drie de limieten past: maximaal 200
gebruikers- of assistentberichten, in totaal 512 KiB UTF-8-tekst en 64 KiB per
bericht. Te grote berichten worden afgekapt met een markering en oudere berichten worden
weggelaten wanneer een limiet wordt bereikt. Een afbeelding of lokale afbeeldingsinvoer wordt de letterlijke
tijdelijke aanduiding `[Image attachment]`; afbeeldingsgegevens en lokale paden worden niet gekopieerd.

Stuur het eerste normale Chat-bericht om het werk te starten. De Codex-harness installeert de
echte handlers voor goedkeuring, informatieverzoeken, gebeurtenissen en aflevering. Deze gebruikt een tijdelijke
native fork op de supervisieverbinding om de bronmomentopname vast te zetten zonder
een model- of provideroverride op te geven. Codex App Server selecteert beide uit de
huidige native configuratie en retourneert de daadwerkelijke selectie. Via diezelfde
verbinding start OpenClaw de canonieke volledige harness-thread met bron `appServer`
onder de bijbehorende cwd en runtimepolicy met exact dat geretourneerde paar, injecteert het
begrensde zichtbare verloop en archiveert de tijdelijke fork. De canonieke thread
beschikt over het volledige OpenClaw-oppervlak van harnesstools. Dit is een vertakking van het zichtbare verloop, niet
een volledige native kloon van de uitvoering: bronredeneringen, toolaanroepen en toolresultaten worden
weggelaten. Deze en elke latere beurt blijft op de bewaakte Codex-verbinding
in plaats van op een andere OpenClaw-modelruntime of de gewone agent-home-harness.

De geretourneerde selectie is geen bewijs van het historische model van de bron. Als de
huidige native configuratie afwijkt van het model dat voor de laatste beurt van de bron is
vastgelegd, geeft Codex de normale waarschuwing over het modelverschil. OpenClaw gebruikt het
geretourneerde paar om de canonieke thread te starten. Codex bewaart het native
model en de provider van die canonieke thread, en bij later hervatten blijven deze behouden omdat
OpenClaw model- en provideroverrides weglaat. Als de canonieke thread via
een afzonderlijke native Codex-besturing wordt gewijzigd, accepteert OpenClaw de door Codex bewaarde
selectie. OpenClaw vervangt deze nooit door zijn buitenste model of fallbackketen.

De bewaakte, modelvergrendelde Chat kan niet worden verwijderd, van model wisselen, `/new`
of `/reset` gebruiken, de Gateway-actie voor het opnieuw instellen van de sessie aanroepen of de algemene
actie **Fork session** gebruiken. Wijzigingen via `/codex model <model>`, `/codex
bind`, `/codex resume` (waaronder een nodesessie met `--bind here`) en
`/codex detach` of `/codex unbind` worden eveneens geweigerd, omdat ze de vergrendelde
native binding zouden vervangen of wissen. De query `/codex model` en `/codex fast`,
`/codex permissions` en `/codex threads` blijven beschikbaar. Start een andere
gewone sessie als je een ander model of een nieuwe thread wilt.

Houd supervisie ingeschakeld voor deze Chat. Als supervisie wordt uitgeschakeld of de
opgeslagen verbindingsbinding niet meer beschikbaar of inconsistent wordt, wordt de beurt
veilig geweigerd in plaats van naar een gewone agent-home-sessie te gaan.

Het uitschakelen of verwijderen van de Plugin `codex` geeft dat eigendom niet vrij en
maakt de Chat niet geschikt voor een ander model. De vergrendelde Chat blijft behouden maar
niet beschikbaar; installeer of activeer dezelfde Plugin opnieuw en start de Gateway opnieuw om
deze te hervatten. Dit opzettelijke fail-closed-gedrag voorkomt dat opschoning wegens retentie of een
tijdelijke uitval van de Plugin de native binding ongemerkt verweesd achterlaat.

De agenttool `codex_threads` volgt dezelfde grens. Deze kan geen
andere fork koppelen of de gebonden native thread van de Chat archiveren. Lijsten en alleen-lezen van
metadata blijven beschikbaar. Voor het lezen van onbewerkte transcripten is `allowRawTranscripts` vereist.
Wanneer onbewerkte toegang is uitgeschakeld, weigert `codex_threads` ook zoekopdrachten in lijsten, omdat
native zoeken transcriptvoorbeelden bevat; de Control UI en operator-CLI
bieden nog steeds begrensde zoekopdrachten die alleen titels doorzoeken. Hernoemen, dearchiveren, een losgekoppelde fork en
het archiveren van een niet-gerelateerde thread zonder eigenaar vereisen
`allowWriteControls`. Geen van beide opties omzeilt de vergrendelde binding.

OpenClaw abonneert zich niet op goedkeuringsverzoeken en beantwoordt deze niet wanneer het alleen
de bronthread weergeeft of de wachtende Chat toont. Door bij de eerste beurt een afzonderlijke canonieke
harness-thread te starten, kan een ander Codex-proces eigenaar van de
bron blijven zonder concurrerende uitvoeringsschrijvers te creëren.

De oorspronkelijke bron uit de CLI, VS Code, Atlas of ChatGPT blijft zichtbaar voor native
clients en de OpenClaw-catalogus. De canonieke vertakking wordt opgeslagen als een native
Codex-thread, maar het brontype ervan is `appServer`; Codex Desktop of een andere
native client kan dat brontype filteren, waardoor niet gegarandeerd is dat de vertakking zelf
in elke native geschiedenisweergave verschijnt.

Een actieve rij die door de App Server van OpenClaw wordt gemeld, kan geen nieuwe vertakking starten. Wacht
tot de huidige beurt is voltooid en vernieuw de catalogus. Codex App Server
serialiseert wijzigingen binnen één proces, maar biedt geen exclusieve
procesoverschrijdende runner- of goedkeuringseigenaarslease.

Voor een rij **Stored / activity unknown** gebruiken de Chat-spiegel en het vastzetten van de momentopname
van de eerste beurt de toestand van Codex tot en met de laatste permanent opgeslagen terminale beurt. De bronthread
wordt niet hervat, onderbroken of gearchiveerd. Als een ander proces een
lopende beurt heeft, is het meest recente werk tijdens die uitvoering mogelijk niet aanwezig in de vertakking.

## Een lokale sessie archiveren

Kies **Archive** voor een opgeslagen of inactieve lokale Gateway-rij en bevestig vervolgens dat geen
andere Codex-client of OpenClaw-runner die thread of de daaruit voortgekomen
afstammelingen gebruikt. OpenClaw leest de proceslokale status opnieuw, gaat alleen verder bij
`idle` of `notLoaded`, roept de native archiveerbewerking van Codex aan en verwijdert de
sessie uit de lijst met niet-gearchiveerde sessies. Native Codex probeert ook de
voortgekomen afstammelingen van de thread te archiveren.

Archive is niet beschikbaar wanneer de nieuwe uitlezing meldt dat de sessie actief is of een
foutstatus heeft, wanneer deze bij een gekoppelde Node hoort of zolang een pas aangemaakte
bewaakte Chat nog een wachtende vertakking van die bron heeft. Stuur het eerste bericht van de Chat
om de canonieke vertakking ervan te materialiseren voordat je de bron archiveert.
Archive wordt ook geblokkeerd wanneer OpenClaw weet dat een actieve binding eigenaar is van
de exacte doelthread of een niet-gearchiveerde voortgekomen afstammeling. OpenClaw volgt de
experimentele Codex-query voor afstammelingen door elke pagina; een ongeldig antwoord,
mislukt verzoek, herhaalde cursor of thread, of uitputting van de veiligheidslimiet leidt tot
weigering van de archivering.

De lees-, afstammelingeninventarisatie- en archiveerverzoeken vormen samen geen voorwaardelijke
bewerking, waardoor er tussendoor nog steeds een beurt kan starten. De App Server-status wordt bovendien
niet gedeeld tussen onafhankelijke processen. De bevestiging vormt daarom de
veiligheidsgrens voor onbekende clients en die race: sluit alle andere clients af of controleer ze
op een andere manier voordat je bevestigt. Herstel een gearchiveerde thread met Codex
Desktop, de Codex-CLI of een door de eigenaar geautoriseerde native threadbeheerflow;
de thread verschijnt opnieuw na het dearchiveren.

```bash
codex unarchive <thread-id>
```

## De beperkingen van gekoppelde Nodes begrijpen

Gekoppelde Nodes bieden de geversioneerde alleen-lezenopdrachten
`codex.appServer.threads.list.v1` en
`codex.appServer.thread.turns.list.v1`. Native Node-hosts waarop de
Codex-CLI beschikbaar is, bieden ook de toegestane opdracht `codex.terminal.resume.v1`.
De Gateway ontvangt genormaliseerde
metadata en expliciet aangevraagde begrensde transcriptpagina's, nooit onbewerkte App Server-
eindpunten. Het openen van een rij in de operatorterminal voert `codex resume <thread-id>`
uit op de host die eigenaar is en stuurt de PTY van die opdracht door; dit biedt geen algemene
shell of door de Gateway opgegeven argv.

Het terminalrelais biedt niet de eigendomscontracten voor voortzetting of archivering van de harness.
Externe rijen blijven daarom zichtbaar, maar bieden geen **Continue** of
**Archive**, zelfs niet wanneer de externe thread inactief is. Gebruik Codex op die computer
via **Open in terminal**, of gebruik een toekomstige voortzettingsflow met een veilige
grens voor runnereigendom.

## Metadata en machtigingen

Catalogusrijen kunnen het volgende bevatten:

- thread- en sessie-id's
- titel en werkmap
- huidige status en actieve wachtvlaggen
- tijdstempels voor aanmaak, bijwerking en activiteit
- bron, modelprovider, versie van de Codex-CLI en Git-branch

De catalogusprojectie sluit transcriptvoorbeelden, beurten, uitvoeringspaden,
het Codex-homepad, externe Git-opslagplaatsen, commit-SHA's en onbewerkte App Server-fouten uit. Voor toegang
tot de catalogus en het lezen van transcripten in de Control UI is het Gateway-bereik `operator.write`
vereist, omdat vlootaggregatie het standaardpad `node.invoke` gebruikt, ook al
zijn beide Node-opdrachten alleen-lezen.

`supervision.allowRawTranscripts` en `supervision.allowWriteControls` regelen
autonome agenttools en zelfstandige MCP-tools. Beide zijn standaard ingesteld op `false`. Wanneer
supervisie is ingeschakeld, verwijdert `codex_threads` transcriptvoorbeelden en beurten uit
lijstresultaten en alleen-lezenresultaten voor metadata, tenzij onbewerkte transcripten zijn toegestaan; een
leesbewerking met beurten wordt veilig geweigerd. Voor elke fork, hernoeming, archivering en dearchivering
zijn schrijfrechten vereist. Deze opties beperken niet het bekijken van transcripten via de geauthenticeerde Control UI
en omzeilen geen controles op binding, host, status of bevestiging.

### Compatibiliteitstools

De officiële Plugin `codex` behoudt de vijf uitgebrachte Supervisor-toolnamen voor
bestaande agent- en zelfstandige MCP-clients:

- `codex_endpoint_probe`
- `codex_sessions_list`
- `codex_session_read`
- `codex_session_send`
- `codex_session_interrupt`

`codex_sessions_list` werkt standaard alleen met geladen items; er is geen parameter `loaded_only`.
Stel `include_stored: true` in om ook niet-gearchiveerde opgeslagen rijen uit
de toestandsdatabase van Codex te lezen. De optionele limiet `max_stored_sessions` is standaard 200
en accepteert 1 tot en met 1.000 rijen per eindpunt. Deze beperkt geladen rijen niet.
Zonder machtiging voor onbewerkte transcripten laten lijstresultaten van transcripten afgeleide namen,
voorbeelden en gedetailleerde eindpuntfouten weg.
`codex_session_read` vereist `allowRawTranscripts`; `include_turns: true`
vraagt Codex daarnaast om beurten.

`codex_session_send` en `codex_session_interrupt` vereisen
`allowWriteControls`. Verzenden accepteert `mode: "auto" | "start" | "steer"`, maar
`"start"` wordt altijd geweigerd en zowel `"auto"` als `"steer"` kunnen alleen een
leesbare actieve beurt sturen. Een inactieve thread wordt geweigerd met het advies **Codex
Sessions** te gebruiken, waar de volledige harness handlers voor goedkeuringen en tools installeert voordat
de voortzetting begint. Onderbreken vereist eveneens een actieve leesbare beurt. Deze tools
hervatten of starten geen inactieve bronthread.

`openclaw doctor --fix` verplaatst een uitgefaseerd item `codex-supervisor`, de bijbehorende eindpunt-
en machtigingsvelden en verwijzingen naar het toestaan/weigeren-beleid van de Plugin naar de officiële
Plugin `codex` zonder expliciete canonieke instellingen te overschrijven. De zelfstandige
compatibiliteitsadapter voor MCP blijft dezelfde vijf tools uit die
Plugin laden; verouderde beleidsomgevingsvariabelen zijn alleen binnen die vertrouwde
adapter van toepassing.

Zie voor elk configuratieveld voor supervisie de
[Codex-harnessreferentie](/nl/plugins/codex-harness-reference#supervision).

## Problemen oplossen

**Er verschijnen geen sessies:** controleer of `@openclaw/codex` is geïnstalleerd, zowel de
Plugin als `supervision.enabled` op true staan, de huidige lijst met toegestane Plugins
`codex` toestaat en de sessies niet zijn gearchiveerd. Start de Gateway of Node opnieuw na
het wijzigen van de activering.

**Continue is uitgeschakeld:** een niet-toegewezen rij is actief, hoort bij een gekoppelde Node,
de bijbehorende host is offline of er wacht een andere actie. Lokale opgeslagen en inactieve
Gateway-rijen bieden **Continue as branch** in plaats van een onveilige overname van de exacte thread. Een rij
die al een bewaakte Chat heeft, biedt **Open Chat**.

**Archive is uitgeschakeld:** archiveren is beschikbaar voor lokale Gateway-rijen met de status
opgeslagen/activiteit onbekend of inactief, nadat is bevestigd dat er geen andere runner is. Actieve rijen, rijen met fouten,
offline rijen, rijen van gekoppelde Nodes, rijen met een wachtende vertakking en rijen met een bekende eigenaar van de exacte binding blijven
alleen-lezen voor archivering.

**Een gearchiveerde sessie is verdwenen:** dit is te verwachten. De supervisiepagina heeft
geen weergave voor gearchiveerde items. Voer `codex unarchive <thread-id>` uit of gebruik Codex Desktop om
de sessie opnieuw weer te geven.

**De oude configuratie `codex-supervisor` bestaat nog:** voer `openclaw doctor --fix` uit. Doctor
verplaatst het uitgefaseerde Plugin-item en de bijbehorende Plugin-beleidsverwijzingen naar
`plugins.entries.codex.config.supervision` zonder expliciete Codex-
instellingen te overschrijven.

## Gerelateerd

- [Codex-harness](/nl/plugins/codex-harness)
- [Codex-harnessreferentie](/nl/plugins/codex-harness-reference)
- [Codex-harnessruntime](/nl/plugins/codex-harness-runtime)
- [Architectuur van Codex-supervisie](/specs/codex-supervision)
- [Nodes](/nl/nodes)
- [Gateway-beveiliging](/nl/gateway/security)
