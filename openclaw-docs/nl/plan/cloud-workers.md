---
read_when:
    - Cloudworker-provisioning, workermodus of sessieoverdracht ontwerpen of implementeren
    - Omgevingen wijzigen.*, het workerprotocol, transcriptinvoer of RPC's voor de inferentieproxy
    - De beveiligingsstatus van uitvoering door externe agents beoordelen
summary: Voer agentsessies uit op tijdelijke, via SSH bereikbare machines met door de Gateway geproxyde inferentie en live streaming in de zijbalk.
title: Plan voor cloudworkers
x-i18n:
    generated_at: "2026-07-27T06:20:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 134c3f6e486837607225d95d12a3153525b14237b362b9f9957313d9bc379dc4
    source_path: plan/cloud-workers.md
    workflow: 16
---

## Status

Voorstel, revisie 3. Niet geïmplementeerd. Richting overeengekomen in 2026-07; revisie 2 verwerkte bevindingen uit adversariële beoordeling (specifiek workerprotocol, toestandsmachines voor plaatsing/omgeving, git-bewuste inkomende synchronisatie, eenrichtingshandoff in v1, beveiligingsformulering voor gecontroleerde uitgaande verbindingen). Revisie 3 legt het eigendomsmodel voor synchronisatie vast (de worker maakt commits, de Gateway neemt ze over en publiceert ze), voegt een gewone synchronisatiemodus zonder git toe, corrigeert de uitvoering door workers naar volledig-binnen-de-box, verplaatst internetbeleid naar het moment van provisioning en herstelt agentdispatch naar mijlpaal 3.

## Probleem

OpenClaw-agentsessies voeren hun lus, tools en inferentie uit binnen het Gateway-proces op één machine. De rekenkracht wordt begrensd door die machine, langdurige taken houden haar bezet en parallel werk concurreert om haar capaciteit. Gehoste producten (Cursor-cloudagents, Claude Code op het web, Codex-cloud) lossen dit op met tijdelijke cloudsandboxes per taak, maar vereisen infrastructuur en vertrouwen van de leverancier.

Operators die al reservemachines bezitten (of deze goedkoop kunnen leasen) kunnen niet aangeven: voer deze sessie daar uit, toon haar in mijn zijbalk zoals elke andere sessie en gooi de machine daarna weg.

## Doelen

- Voer een volledige agentsessie (lus + tools) uit op een tijdelijke externe machine ("cloudworker"), terwijl de sessie in de Control UI exact zoals een lokale sessie wordt weergegeven en gestreamd.
- Geen permanente referenties op de worker (geen providerauthenticatie, geen forge-tokens) en geen directe uitgaande netwerkverbindingen; de box heeft alleen een bereikbare sshd nodig.
- Provisioning, synchroniseren, uitvoeren, verzamelen, vernietigen — volledig geautomatiseerd en provider-plugbaar (eerste provider: lease-CLI's in Crabbox-stijl).
- Verplaats actief werk bij een beurtgrens van de Gateway naar een worker zonder transcript, sessie-identiteit of (wanneer aanvraagbytes equivalent blijven) providercache-affiniteit te verliezen; haal resultaten veilig terug.
- Zowel mensen (UI) als agents (tool) kunnen werk naar een cloudworker verplaatsen.
- Ondersteun sessies die dagenlang duren; de levensduur is beleid, geen hardgecodeerde limiet.

## Geen doelen (v1)

- Geen externe codeharnassen (Claude Code, Codex CLI) op workers. Workersessies voeren alleen de ingebedde runner van OpenClaw uit. Ondersteuning voor harnassen is een opt-in voor v2, omdat harnassen hun eigen inferentie uitvoeren met hun eigen referenties.
- Geen best-of-N-/parallelle uitwaaiering van pogingen.
- Geen afhankelijkheid van VPN/tailnet. Transport verloopt uitsluitend via SSH.
- Geen nieuwe sandboxruntime. De workermachine vormt de isolatiegrens; OS-sandboxing binnen de box kan later als extra laag worden toegevoegd.
- Geen symmetrische livemigratie in v1: verplaatsing gaat van lokaal → worker; worker → lokaal vereist een gestopte sessie plus voltooide afstemming van de werkruimte. Live tweerichtingshandoff bouwt later voort op hetzelfde barrièremechanisme.
- Geen JSON-nevenstatus op de Gateway; omgevings-, plaatsings-, cursor- en toekenningsstatus worden opgeslagen in SQLite.

## Eerdere voorbeelden (wat we overnemen, wat we omkeren)

- Cursor-cloudagents: de agentlus draait in hun cloud; de VM is een doel voor tooluitvoering; een alleen-toevoegbare conversieopslag wordt naar alle clients gestreamd; warm starten via snapshot-na-installatie; zelfgehoste workers zijn workerprocessen die alleen uitgaande verbindingen maken. We nemen het model over waarbij "de gezaghebbende bron van de conversatie bij de orchestrator blijft" en het streamingmodel; we keren de plaatsing van de lus om (zie beslissing hieronder).
- Codex-cloud: runtime in twee fasen — een installatiefase met netwerktoegang, gevolgd door een offline agentfase waarin geheimen zijn verwijderd; cache van containerstatus voor snelle vervolgacties. We nemen de fasescheiding over als onze houding tegenover uitgaande verbindingen en het cache-idee voor warme v2-images.
- Claude Code op het web: VM per sessie; git-proxy die referenties isoleert (echte tokens komen nooit in de sandbox, push beperkt tot de sessiebranch); bestandssysteemsnapshot na installatie; teleport-handoff = gepushte branch + opnieuw afgespeelde geschiedenis. We nemen de isolatie van referenties en de structurering van de handoff over, maar uitgaande synchronisatie gebeurt via rsync vanaf de Gateway, zodat niet-schone working trees werken en er nergens in de buurt van de box een forge-token bestaat.
- Copilot-codeagent: standaard geweigerde uitgaande verbindingen met een allowlist voor pakketregisters. Onze standaard in stabiele toestand is strenger (helemaal geen directe uitgaande verbindingen), omdat inferentie en webzoekopdrachten via de SSH-tunnel binnenkomen — maar zie Beveiliging voor waarom dit "gecontroleerde uitgaande verbindingen" zijn en niet "geen uitgaande verbindingen".

## Architectuurbeslissing: lus op de worker, inferentie via de Gateway

Drie plaatsingen zijn overwogen:

1. Lus blijft op de Gateway, worker voert tools uit (Cursor-model). Veiligste foutdomein (transcript, inferentie, goedkeuringen en herstel na herstart blijven allemaal lokaal) en de door een beoordelaar geprefereerde eerste mijlpaal. Afgewezen als productarchitectuur: de niet-exec-tools van OpenClaw zijn bestandssysteembewerkingen binnen het proces, waardoor elke lees-, bewerkings- of grep-actie op bestanden een netwerkrondreis wordt of een grote refactor van het tooloppervlak naar grofmazige werkruimte-RPC's vereist; het runtimegedrag is intensief communicerend en wordt door latentie begrensd. We hergebruiken de essentie hiervan waar die al is gebouwd (exec-offloading naar Nodes), maar bouwen de laag voor externe tooluitvoering niet.
2. Lus en inferentie beide op de worker. Eenvoudigste foutdomein, maar modelreferenties (inclusief OAuth-profielen) moeten naar wegwerpmachines worden verstuurd, de Gateway verliest controle over beleid/routering/auditing en migratie verandert de identiteit die de provider aanroept, waardoor providercaches ongeldig worden.
3. Lus + tools op de worker, modelaanroepen geproxied via de Gateway. Gekozen. Eén rondreis per modelbeurt in plaats van per toolaanroep; tools draaien naast de code; de Gateway blijft de enige eigenaar van authenticatieprofielen, providerroutering en beleid; de worker bevat geen geheimen.

De kosten van optie 3 zijn een synchrone afhankelijkheid van de Gateway tijdens elke modelbeurt. De duurzaamheidsregels daarvan maken daarom deel uit van de beslissing en zijn geen bijzaak:

- Verlies van de Gateway halverwege een beurt laat de actieve provideraanroep mislukken. De beurt wordt als mislukt gemarkeerd en na opnieuw verbinden als een nieuwe beurt herhaald; een lopende providerstream wordt niet transparant opnieuw afgespeeld (risico op dubbele facturering/dubbele toolaanroepen).
- Elke worker↔Gateway-bewerking bevat een duurzame identiteit (zie Workerprotocol), zodat opnieuw verbinden doorgaat of gecachete eindresultaten ophaalt in plaats van bewerkingen te laten hangen.
- De Gateway is een component met capaciteitsbeheer: limieten voor gelijktijdige workers, flowcontrol en het afstoten van belasting vallen binnen de scope van v1 (zie Capaciteit).

Omdat de Gateway zowel het transcript opslaat als al het providerverkeer initieert, is de sessie locatie-onafhankelijk: het verplaatsen van de lus tussen Gateway en worker verandert niets aan de providerzijde en niets in het UI-gegevenspad. Daardoor zijn verplaatsing en terughalen goedkoop.

## Componenten

### 1. Toestandsmachine voor omgevingen + providercontract

`environments.*` in het Gateway-protocol is momenteel uitsluitend een statusprojectie. De duurzame kern is een door SQLite beheerd omgevingsrecord en een toestandsmachine, ontworpen vóór de RPC-vormen:

`requested → provisioning → bootstrapping → ready → (attached|idle) → draining → destroying → destroyed | failed | orphaned`

- Provisioning is crashbestendig: de intentieregel wordt vóór de provideraanroep opgeslagen, met een deterministische bewerkings-id, zodat de Gateway na een herstart een lopende lease kan overnemen in plaats van dubbele provisioning uit te voeren of een betaalde machine verweesd achter te laten.
- Afstemming na een herstart en een opruimer voor verweesde resources (provider `inspect` versus lokale records) zijn v1-vereisten, geen extra versteviging.

Providercontract (geïmplementeerd door een Plugin; geen providernamen of beleid in de kern):

```ts
type WorkerProvider = {
  id: string;
  provision(profile: WorkerProfile, opId: string): Promise<WorkerLease>; // → SSH-host/-poort/-gebruiker/-sleutelmateriaal
  inspect(lease: { leaseId: string; profile: WorkerProfile }): Promise<LeaseStatus>; // overnemen/gezondheid/verweesde resources opruimen
  renew?(leaseId: string): Promise<void>; // langlopende sessies versus provider-TTL's
  destroy(lease: { leaseId: string; profile: WorkerProfile }): Promise<void>; // idempotent, retourneert pas na bewijs van ontmanteling
};
```

RPC's: `environments.create`, `environments.destroy`, uitgebreide `environments.list/status` (provider, lease-id, status, ouderdom, inactieve tijd, gekoppelde sessies). Eerste providers: een wrapper voor een lease-CLI in Crabbox-vorm (productpad) en een provider voor statische SSH-hosts die als uitsluitend voor ontwikkeling wordt gemarkeerd — een worker op een gedeelde host kan niet-gerelateerde hostgegevens lezen, dus statische hosts zijn bedoeld voor functieontwikkeling, niet als standaardhouding.

### 2. Worker-bootstrap: OpenClaw op de box installeren

Geen speciaal workerartefact en geen afhankelijkheid van npm-beschikbaarheid:

- Canonieke installatie voor alle modi: een door de Gateway geproduceerde workerbundel met inhoudshash (de eigen buildoutput van de Gateway verpakt als tarball), via SSH verstuurd en op de box geïnstalleerd. Dit ondersteunt door de constructie zowel ontwikkelbuilds als nog niet uitgebrachte commits.
- `npm i -g openclaw@<exact gateway version>` is een optimalisatie wanneer de Gateway een uitgebrachte versie uitvoert; nooit `latest`.
- Bootstrap is idempotent; een warme lease met een overeenkomende bundelhash slaat de installatie over. Kale machines hebben mogelijk een toolchainfase met netwerktoegang nodig (Node-runtime) — onderdeel van de installatiefase, daarna afgesloten.
- De handshake verifieert de workerbuildhash, de set protocolfuncties en runtimecompatibiliteit. De bestaande versie-/protocolcontroles van de Gateway zijn hiervoor onvoldoende (Nodes via SSH-tunnels zijn vrijgesteld van afwijzing bij een niet-exacte versie), dus workertoelating voert een eigen controle op exact dezelfde build uit.

Workermodus (`openclaw worker`) is een ingangspunt, geen fork: verbindingsafhandeling plus de ingebedde agentrunner, met sessiepersistentie en modelaanroepen die door Gateway-RPC's worden ondersteund. Deze modus mag geen Gateway-oppervlakken starten: geen kanalen, geen automatische start van Plugins buiten de toolset van de sessie, een tijdelijke statusmap en geen lokale authenticatieprofielen.

### 3. Transport: alles via SSH

De Gateway beheert de verbinding; de worker heeft niets anders nodig dan sshd:

- De Gateway opent SSH naar de worker (referenties uit de providerlease, hostsleutel vastgezet vanuit de provisioningoutput — geen `StrictHostKeyChecking=no`) en brengt een omgekeerde tunnel tot stand die een workerlokale socket doorstuurt naar het WS-eindpunt van de Gateway.
- Besturings-/modelverkeer en werkruimteoverdracht gebruiken afzonderlijke SSH-verbindingen met hetzelfde vastgezette vertrouwensmateriaal, zodat rsync tokenstreams niet kan blokkeren door head-of-line-blocking.
- De levenscyclus van de tunnel (keepalive, opnieuw verbinden met back-off) wordt beheerd door de omgevingsruntime op de Gateway. Een korte tunnelonderbreking is onzichtbaar op sessieniveau: dankzij duurzame protocolstatus (hieronder) kan de worker zich opnieuw koppelen en doorgaan.

### 4. Workerprotocol (specifiek; niet het Node-protocol)

Adversariële beoordeling aan de hand van de huidige Node-koppelingen sloot eenvoudig hergebruik uit: wachtende Node-aanroepen zijn proceslokale promises die samen met de verbinding verdwijnen, idempotentiesleutels van Nodes worden geparseerd maar niet gededupliceerd en — doorslaggevend — een verbonden Node kan gewone Node-events uitsturen (waaronder aanvragen voor agentuitvoering), waardoor "Node-soort + capaciteitsplafond" geen beveiligingsgrens voor inkomend verkeer vormt. Workers krijgen daarom een geauthenticeerde rol `worker` met een gesloten, geversioneerde allowlist voor RPC's/events; workerverbindingen kunnen geen enkele verouderde Node-eventhandler bereiken.

Identiteit en referenties: provisioning maakt een kortlevende workerreferentie die is gebonden aan omgevings-id, workersleutel, bundelhash, de enige toegestane sessie, de toegestane RPC-set en een vervaltijd. Via SSH geverifieerde koppeling blijft van toepassing (we hebben de box geprovisioned en bezitten de sleutel), maar autorisatie komt van de aangemaakte referentie, niet van het gedeclareerde Node-oppervlak.

Duurzame bewerkingssemantiek (vorm overgenomen van de bestaande ACP-runtime en het eventlogboek daarvan — stabiele handles, serialisatie per sessie, duurzame herhaling van `(session, seq)`):

- Elke bewerking is beperkt tot `(sessionId, lifecycleRevision, runId, ownerEpoch, streamKind, seq)`.
- Eigendomsepochen schermen verouderde workers af: een vervangende worker verhoogt de epoch; late resultaten uit de oude epoch worden deterministisch geweigerd.
- Ten minste één keer afleveren met persistente ACK-cursors en gecachte eindresultaten in SQLite; deduplicatie is deterministisch. Geen garanties voor exact één keer.
- Expliciete frames voor annuleren, sluiten, hervatten en eindresultaten; op credits/vensters gebaseerde stroomregeling voor streams.
- Onderhandeling over protocolfuncties staat los van de algemene versie van het nodeprotocol.

### 5. RPC's voor de sessiebackend

Twee afzonderlijke contracten — de huidige codebase scheidt duurzame transcriptmutaties (beheerd door de sessiemanager, JSONL-boom met bovenliggende/bladstatus) van proceslokale livegebeurtenissen (streamingdelta's, levenscyclus van tools, goedkeuringen), en het workerprotocol moet die scheiding behouden:

- Duurzame transcriptcommits: de worker dient semantische toevoegingsbatches in met `runEpoch` + compare-and-swap op het basisblad; de Gateway-sessiemanager genereert invoer-id's en bovenliggende id's. De worker kan nooit vertrouwde transcriptrijen, invoer-id's, bovenliggende id's of externe sessie-id's aanleveren.
- Herhaalbare livegebeurtenissen: een getypeerde event-union met workervolgnummering, Gateway-ACK's, begrensde retentie en afscherming van late gebeurtenissen, die de bestaande fan-out van agentgebeurtenissen voedt zodat de chatweergave, toolrijen en logica voor ongelezen items/status zich identiek gedragen als bij lokale sessies.

Inferentieproxy: hergebruik de gebeurteniswoordenschat van de bestaande runtime-proxystreamclient (`src/agents/runtime/proxy.ts`), maar verplaats de vertrouwensgrens. De worker verzendt alleen de sessie-/runidentiteit, een goedgekeurde modelreferentie, context en beperkte generatieopties; de Gateway bepaalt provider, endpoint, authenticatie, headers, routering en kostenbeleid uit zijn eigen catalogus. Een door een worker aangeleverd modelobject (bijvoorbeeld een door een aanvaller beheerde `baseUrl`) wordt geweigerd. Limieten voor aanvraaggrootte, annulering, audit en herhaling van eindresultaten zijn van toepassing. Tools die op de Gateway worden uitgevoerd (websearch), worden op de Gateway uitgevoerd en retourneren resultaten via hetzelfde kanaal.

### 6. Werkruimtesynchronisatie

Het synchronisatieanker is een Gateway-lokale werkruimte met exclusief plaatsingseigendom: voor git-werkruimtes een speciale beheerde worktree (bestaande metadata van de beheerde worktree — branch, basis, eigendom van momentopnamen — vormt de basis); voor werkruimtes zonder git een doelmap die eigendom is van de Gateway. Nooit de actieve checkout van de gebruiker. Exclusief eigendom terwijl de sessie extern is geplaatst, maakt inkomende synchronisatie door de constructie conflictvrij.

Verdeling van eigendom — commit versus publiceren:

- De agent aan de workerzijde maakt normaal commits in zijn kopie (`git commit` is een lokale bewerking zonder referenties; de auteursidentiteit wordt geprojecteerd vanuit de Gateway-configuratie). Die commits zijn inerte objecten totdat de Gateway ze overneemt.
- De Gateway doet alles waarvoor vertrouwen vereist is: verifiëren dat inkomende commits voortbouwen op de vastgelegde basis, de lokale worktree fast-forwarden, pushen, PR's maken en optioneel ondertekenen/opnieuw ondertekenen — allemaal met Gateway-lokale referenties. De worker beschikt nooit over git- of forge-referenties en benadert nooit een remote.

Twee synchronisatiemodi, geselecteerd op basis van de vraag of de werkruimte een git-repository is:

- Git-modus. Uitgaand: synchroniseer de worktree met rsync (inclusief niet-gecommitte bestanden en toegestane niet-gevolgde bestanden; include/exclude zoals bij crabbox, met inachtneming van `.worktreeinclude`) via de SSH-identiteit van de tunnel, vastgelegd als een onveranderlijk basismanifest (inhoudshashes + basiscommit). Inkomend: nieuwe commits keren terug als een git-bundel of tijdelijke ref ten opzichte van de vastgelegde basis; niet-gevolgde artefacten keren terug via een expliciet manifest met controles op grootte/type/insluiting van symbolische koppelingen. Bij overname wordt de afstamming van de basis geverifieerd en wordt bij divergentie gestopt — niets overschrijft stilzwijgend een van beide zijden. Verwijderingen, hernoemingen, submodules en ontsnapping via symbolische koppelingen worden afgehandeld door de manifestregels, niet door rsync-heuristieken.
- Platte modus (geen git — bijvoorbeeld wanneer een project vanaf nul op de box wordt gebouwd). Uitgaand gebeurt met dezelfde rsync + hetzelfde basismanifest. Inkomend wordt een op manifestverschillen gebaseerde spiegeling terug naar de doelmap van de Gateway uitgevoerd, inclusief propagatie van verwijderingen. Veilig om dezelfde reden als de git-modus: exclusief eigendom betekent dat er geen gelijktijdige lokale bewerkingen bestaan die conflicten kunnen veroorzaken; het basismanifest detecteert nog steeds onverwachte lokale afwijkingen en stopt in plaats van te overschrijven.

Checkpointing beschermt dagenlange sessies tegen verlies van de lease: periodieke inkomende checkpoints (commits op een sessiebranch in de git-modus, manifestmomentopnamen in de platte modus); het interval is profielbeleid (standaard op basis van beurten).

### 7. Toestandsmachine voor plaatsing, sessies en UI

Runtimeplaatsing is een toestandsmachine in SQLite die aan de sessie is gekoppeld, geen paar losse rijvelden:

`local → requested → provisioning → syncing → starting → active(worker) → draining → reconciling → local | reclaimed | failed`

Hierin worden de omgevings-id, overgangsgeneratie, actieve eigenaarepoch, het basismanifest van de werkruimte, de hash van de workerbundel en de laatste ACK-cursors persistent opgeslagen. Toelating van een beurt claimt de plaatsing atomair voordat een van beide lussen een beurt start, zodat een lokaal bericht dat op basis van een verouderde momentopname wordt toegelaten nooit kan concurreren met een workerbeurt — op elk moment is precies één lus eigenaar van de sessie.

UI:

- Een workersessie is een gewone sessierij plus plaatsingsmetadata. Deze bevindt zich in de normale opslag, wordt weergegeven via `sessions.list` en streamt via bestaande abonnementen — de zijbalk en chat hebben geen nieuw gegevenspad nodig, alleen presentatie: een workerbadge en plaatsings-/omgevingsstatus (`provisioning / syncing / running / idle / reconciling / reclaimed`).
- UX voor aanmaken: de sessiedoelbalk (nieuw ontwerp van de sessiezijbalk) krijgt naast Gateway en node een cloudworkerbestemming. Vereist een geconfigureerd providerprofiel; de functie is onzichtbaar totdat deze is geconfigureerd.
- Agentdispatch: met een sessietool kan een agent werk aan een cloudworker overdragen zoals een mens dat doet (door een worker ondersteunde subsessie, in subagentstijl). Wordt geleverd in dezelfde mijlpaal als dispatch door mensen en wordt door dezelfde opt-in-providerconfiguratie afgeschermd. Recursie is structureel begrensd (workersessies kunnen in v1 niet zelf workers inschakelen); uitgavenbeheer gebeurt via boekhouding/audit per omgeving, niet via quotamechanismen.

## Dispatch en overdracht

v1 is bewust asymmetrisch:

- Lokaal → worker (dispatch): passeer de onderstaande migratiebarrière, richt een worker in of hergebruik er een, synchroniseer, wijzig de plaatsing; de volgende beurt wordt extern uitgevoerd.
- Worker → lokaal (terughalen): stop de sessie (laat de worker leeglopen via dezelfde barrière), voltooi de inkomende afstemming en wijzig de plaatsing naar lokaal. Geen livemigratie.
- Symmetrische liveoverdracht (een actief werkende sessie in beide richtingen verplaatsen zonder te stoppen) hergebruikt dezelfde barrière- en afstemmingsmechanismen en wordt geleverd nadat foutinjectietests de barrière hebben bewezen.

Migratiebarrière (alleen een "beurtgrens" is onvoldoende — goedkeuringen, achtergrondprocessen en transcriptsamenvoegingen na vrijgegeven vergrendelingen kunnen deze grens overschrijden):

1. Stop de toelating van nieuwe beurten (plaatsingsclaim).
2. Annuleer actieve runs of laat ze leeglopen.
3. Trek openstaande uitvoeringsgoedkeuringen en uitvoeringstoestemmingen in.
4. Laat zijwaartse transcriptschrijfbewerkingen en ACK's van livegebeurtenissen leeglopen.
5. Beëindig onderliggende workerprocessen.
6. Scherm de oude eigenaar af door de eigenaarepoch te verhogen.
7. Stem de werkruimte af (inkomend, conflictbewust).
8. Activeer de nieuwe eigenaar.

Cacheaffiniteit: omdat provideraanvragen bij beide plaatsingen afkomstig zijn van de Gateway, blijft cacheaffiniteit behouden wanneer de geserialiseerde provideraanvraag equivalent blijft — dezelfde toolvolgorde, systeeminstructies, providerwrappers en cachemetadata (die aan de Gateway-zijde blijven). Dit is een testbare eigenschap, geen aanname: tests voor byte-equivalentie tussen lokale/workerplaatsing per ondersteund providertransport maken deel uit van de mijlpaal die de workerlus introduceert.

## Beveiligingsmodel

Nauwkeurig geformuleerd: de worker heeft geen directe uitgaande netwerktoegang en geen permanente provider-/forge-referenties. Het is geen "nul uitgaand verkeer" — inferentie en door de Gateway uitgevoerde tools zijn gecontroleerde kanalen voor uitgaand verkeer (een via promptinjectie aangevallen worker kan nog steeds werkruimtebytes in modelcontext of websearch-query's plaatsen). Daarom:

- Boekhouding van gecontroleerd uitgaand verkeer: audit per omgeving en voor operators zichtbare boekhouding voor de inferentieproxy en Gateway-tools. Snelheids-/bytelimieten bestaan als protocolstroomregeling (capaciteit), niet als mechanismen voor uitgavenquota.
- Inkomend verkeer van de worker naar de Gateway is beperkt tot de gesloten allowlist van het workerprotocol; transcriptschrijfbewerkingen zijn structureel beperkt (door de Gateway gegenereerde id's, één gebonden sessie).
- Workeruitvoering heeft volledige rechten binnen de box. De box is wegwerpbaar en bevat geen referenties, dus goedkeuring per opdracht voegt wrijving toe zonder iets te beschermen; de bewaakte grens is inkomende afstemming en audit. Uitvoering loopt nooit via het goedkeuringspad voor Gateway-nodes.
- Internetbeleid is een providerbeslissing tijdens het inrichten: het omgevingsprofiel bepaalt dit bij het maken van de box (firewall/beveiligingsgroep/netwerk zonder uitgaand verkeer), eventueel met een netwerkgebonden instelfase die de provider afsluit voordat de agentfase begint. Core implementeert geen runtimeschakelaar voor het netwerk.
- Hygiëne van de box tijdens het inrichten: endpoint voor cloudmetadata geblokkeerd of aantoonbaar afwezig, geen instanceprofiel, geen overgenomen SSH-agent, geen Docker-socket, schone omgeving/homemap. SSH-hostsleutels worden vastgezet op basis van de uitvoer van de inrichting.
- Goedkeuringen en beleid voor alles aan de Gateway-zijde (push, PR, provideraanroepen) blijven op de Gateway worden uitgevoerd.

Impactgebied van een gecompromitteerde workersessie: de gesynchroniseerde kopie van de werkruimte plus wat de gecontroleerde proxykanalen toestaan — geen referenties, geen direct netwerk, geen Gateway-oppervlak buiten de allowlist.

## Capaciteit

De Gateway stuurt elke prompt en tokenstream voor N workers door, dus v1 definieert een capaciteitsmodel in plaats van dit pas in productie te ontdekken: limieten voor gelijktijdige workers per Gateway, creditvensters per stream (de huidige wachtrij voor de gebeurtenisstream is onbegrensd en de limiet van de node-socketbuffer sluit trage afnemers geforceerd af — beide zijn ongeschikt zonder aanpassingen), begrensde schijfspooling voor pieken en lastafwerping met zichtbare tegendrukstatussen in de UI. Werkruimteoverdracht blijft op een eigen SSH-kanaal.

## Levenscyclus

- Automatisch stoppen bij inactiviteit en TTL zijn beleid van het providerprofiel, geen vaste constanten. Standaardwaarden zijn ruim met expliciete keep-alive; dagenlang werk is een eersteklas scenario (provider `renew` bestaat voor leasegebaseerde backends); een sessie met een lopende beurt of recente activiteit wordt nooit teruggewonnen.
- Wanneer een worker uitvalt of wordt teruggewonnen: de plaatsing gaat naar `reclaimed`, de sessierij blijft bestaan en het volgende bericht richt een nieuwe worker in en synchroniseert opnieuw vanaf het laatste checkpoint. De conversatie gaat nooit verloren (opslag aan de Gateway-zijde); wijzigingen in de werkruimte sinds het laatste checkpoint gaan verloren en de UI meldt dit.
- Hergebruik van warme leases vanaf dag één (voor providers die dit ondersteunen); een image-momentopname na bootstrap is het snelle startpad voor v2.

## Configuratieoppervlak

Minimaal en opt-in: een providerprofielblok (provider-id, referenties/CLI-referentie, synchronisatieregels, levensduurbeleid, budgetten, optionele instelfase) plus plaatsingsselectie per sessie. Geen nieuwe omgevingsvariabelen. Niet-geconfigureerde installaties zien niets.

## Mijlpalen

De implementatie wordt aangeleverd als kleine, onafhankelijk samenvoegbare PR's; elke onderstaande mijlpaal is een reeks PR's, niet één wijziging.

1. Fundamenten: omgevingstoestandsmachine + providercontract + provider in crabbox-vorm (statische SSH als ontwikkelharnas), bootstrap van workerbundel + toelatingshandshake, SSH-tunnel + vastzetten van hostsleutel, snapshot van beheerde worktree + uitgaande synchronisatie (git- en platte modi). Opschonen van verweesde resources + overname na herstart.
2. Workerprotocol + workerlus: geverifieerde workerrol, duurzame bewerkingen/epochs/ACK-cursors, vastleggen van transcript + contracten voor live gebeurtenissen, inferentieproxy met door de Gateway opgeloste modellen, stroomregeling. Eén provider, alleen handmatige dispatch van nieuwe sessies, geen overdracht. Foutinjectietests (tunnelpartitie, herstart van Gateway, uitval van worker) blokkeren afsluiting bij fouten.
3. Dispatch + terughalen + agentdispatch: migratiebarrière, plaatsingstoestandsmachine gekoppeld aan de doelbalk van de UI, inkomende reconciliatie + controlepunten, audit per omgeving, capaciteitslimieten, agentdispatchtool (workersessies kunnen niet recursief dispatchen). Tests voor byte-equivalentie van de promptcache.
4. Symmetrische live overdracht, na foutinjectiebewijs voor mijlpaal 3.

Later: ACP-harnassen op workers als opt-in voor het hydrateren van referenties per omgeving; snelle start via snapshots/voorverwarmde images; uitwaaiering (N leases, dezelfde prompt); OS-sandboxing binnen de box; uitgebreidere vastlegging van artefacten via het artefactenschema.

## Open vragen

- Beschikbaarheid van plugins/Skills op workers: in de repository opgenomen Skills synchroniseren kosteloos met de werkruimte; door de Gateway geconfigureerde agent-Skills/plugins vereisen een expliciete beslissing over synchronisatie of uitsluiting (het tool-/pluginmanifest maakt hoe dan ook deel uit van de toelatingshandshake).
- Standaardfrequentie van controlepunten: op beurten gebaseerd versus op tijd gebaseerd voor zeer actieve chatsessies.
- Hoe omgevingsprofielen samenwerken met routering tussen meerdere agents (standaardprofielen per agent versus uitsluitend selectie per sessie).
