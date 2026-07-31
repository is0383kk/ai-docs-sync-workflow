---
read_when:
    - Je moet weten uit welk SDK-subpad je moet importeren
    - Je wilt een naslagwerk voor alle registratiemethoden op OpenClawPluginApi
    - Je zoekt een specifieke SDK-export op
sidebarTitle: Plugin SDK overview
summary: Importmap, API-referentie voor registratie en SDK-architectuur
title: Overzicht van de Plugin SDK
x-i18n:
    generated_at: "2026-07-27T05:17:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4f490aa8670c57cfc1a635fb1f5d9950fa1cabdb3d45abbc2295da796edcd52e
    source_path: plugins/sdk-overview.md
    workflow: 16
---

De plugin-SDK is het getypeerde contract tussen plugins en de kern. Deze pagina is de
referentie voor **wat je moet importeren** en **wat je kunt registreren**.

<Note>
  Deze pagina is bedoeld voor pluginauteurs die `openclaw/plugin-sdk/*` binnen
  OpenClaw gebruiken. Gebruik voor externe apps, scripts, dashboards, CI-taken en IDE-extensies
  die agents via de Gateway willen uitvoeren in plaats daarvan
  [Gateway-integraties voor externe apps](/nl/gateway/external-apps).
</Note>

<Tip>
Zoek je in plaats daarvan een praktische handleiding? Begin met [Plugins bouwen](/nl/plugins/building-plugins). Gebruik [Kanaalplugins](/nl/plugins/sdk-channel-plugins) voor kanalen, [Providerplugins](/nl/plugins/sdk-provider-plugins) voor modelproviders, [CLI-backendplugins](/nl/plugins/cli-backend-plugins) voor lokale AI-CLI-backends, [Agent-harnasplugins](/nl/plugins/sdk-agent-harness) voor native agentuitvoerders en [Plugin-hooks](/nl/plugins/hooks) voor tool- of levenscyclus-hooks.
</Tip>

## Importconventie

Importeer altijd vanuit een specifiek subpad:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Elk subpad is een kleine, zelfstandige module. Dit zorgt voor een snelle opstart en
voorkomt problemen met circulaire afhankelijkheden. Geef voor kanaalspecifieke invoer-/bouwhulpfuncties
de voorkeur aan `openclaw/plugin-sdk/channel-core`; behoud `openclaw/plugin-sdk/core` voor
het bredere overkoepelende oppervlak en gedeelde hulpfuncties zoals
`buildChannelConfigSchema`.

Publiceer voor kanaalconfiguratie het JSON Schema dat eigendom is van het kanaal via
`openclaw.plugin.json#channelConfigs`. Het subpad `plugin-sdk/channel-config-schema`
is bedoeld voor gedeelde schemaprimitieven en de generieke builder. De gebundelde
plugins van OpenClaw gebruiken `plugin-sdk/bundled-channel-config-schema` voor behouden
schema's van gebundelde kanalen. Dat gebundelde schemasubpad is geen patroon voor nieuwe
plugins.

<Warning>
  Importeer geen gemaksinterfaces met een provider- of kanaalmerk (bijvoorbeeld
  `openclaw/plugin-sdk/slack`, `.../discord`, `.../signal`, `.../whatsapp`).
  Gebundelde plugins stellen generieke SDK-subpaden samen binnen hun eigen `api.ts`- /
  `runtime-api.ts`-barrels; kerngebruikers moeten die pluginlokale
  barrels gebruiken of een nauw begrensd generiek SDK-contract toevoegen wanneer een behoefte werkelijk
  kanaaloverstijgend is.

Een kleine groep hulpinterfaces voor gebundelde plugins verschijnt nog steeds in de gegenereerde exportmap
wanneer het gebruik door de eigenaar wordt bijgehouden. Ze bestaan uitsluitend voor het onderhoud van
gebundelde plugins en zijn geen aanbevolen importpaden voor nieuwe plugins van derden.

`openclaw/plugin-sdk/discord` en `openclaw/plugin-sdk/telegram-account` worden
ook behouden als verouderde compatibiliteitsfacades voor bijgehouden gebruik door de eigenaar. Kopieer
die importpaden niet naar nieuwe plugins; gebruik in plaats daarvan geïnjecteerde runtimehulpfuncties en
generieke SDK-subpaden voor kanalen.
</Warning>

## Subpadreferentie

De plugin-SDK wordt beschikbaar gesteld als een verzameling nauw begrensde subpaden, gegroepeerd per gebied (plugin-
invoer, kanaal, provider, authenticatie, runtime, mogelijkheid, geheugen en gereserveerde
hulpfuncties voor gebundelde plugins). Zie
[Subpaden van de plugin-SDK](/nl/plugins/sdk-subpaths) voor de volledige catalogus, gegroepeerd en met links.

De inventaris van compilerinvoerpunten staat in
`scripts/lib/plugin-sdk-entrypoints.json`; getypeerde openbare exports sluiten de
interne subpaden uit die worden vermeld in
`scripts/lib/plugin-sdk-private-local-only-subpaths.json`. Productie-invoerpunten
in die lijst behouden uitsluitend JavaScript-hostruntime-exports voor afzonderlijk
gepubliceerde officiële plugins, terwijl invoerpunten die alleen voor tests dienen niet worden geëxporteerd. Voer
`pnpm plugin-sdk:surface` uit om het aantal openbare exports te controleren. Verouderde openbare
subpaden die oud genoeg zijn en niet door productiecode van gebundelde extensies worden gebruikt, worden
bijgehouden in `scripts/lib/plugin-sdk-deprecated-public-subpaths.json`; brede
verouderde barrels voor herexport worden bijgehouden in
`scripts/lib/plugin-sdk-deprecated-barrel-subpaths.json`.

## Registratie-API

De callback `register(api)` ontvangt een `OpenClawPluginApi`-object met deze
methoden:

Plugins die een extern teamchatoppervlak voor een sessie bieden, kunnen
de enige procesbrede provider registreren die wordt geëxporteerd door
`openclaw/plugin-sdk/session-discussion`. De methode `info({ sessionKey })`
meldt of een discussie niet beschikbaar is, gereed is om te worden geopend of al geopend is;
`open({ sessionKey })` maakt de discussie aan of zoekt deze op en retourneert de insluitings-
en externe URL's. Door een andere provider te registreren, wordt de huidige provider vervangen.

### Registratie van mogelijkheden

| Methode                                           | Wat deze registreert                                                                                                                         |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerProvider(...)`                      | Tekstinferentie (LLM)                                                                                                                      |
| `api.registerWorkerProvider(...)`                | Levenscyclusleases voor cloudworkers                                                                                                             |
| `api.registerModelCatalogProvider(...)`          | Modelcatalogusrijen voor het genereren van tekst en media                                                                                          |
| `api.registerAgentHarness(...)`                  | [Experimentele](/nl/plugins/sdk-agent-harness) native agentuitvoerder (Codex, Copilot)                                                         |
| `api.registerCliBackend(...)`                    | Lokale CLI-inferentiebackend                                                                                                               |
| `api.registerChannel(...)`                       | Berichtenkanaal                                                                                                                         |
| `api.registerEmbeddingProvider(...)`             | Herbruikbare provider voor vector-embeddings                                                                                                        |
| `api.registerSpeechProvider(...)`                | Tekst-naar-spraak-/STT-synthese                                                                                                            |
| `api.registerRealtimeTranscriptionProvider(...)` | Realtime streamingtranscriptie                                                                                                          |
| `api.registerRealtimeVoiceProvider(...)`         | Duplex realtime spraaksessies                                                                                                            |
| `api.registerMediaUnderstandingProvider(...)`    | Analyse van afbeeldingen/audio/video                                                                                                                |
| `api.registerTranscriptSourceProvider(...)`      | Bron voor live of geïmporteerde vergadertranscripten; vergaderplugins kunnen `createMeetingTranscriptSourceProvider` uit `plugin-sdk/transcripts` gebruiken |
| `api.registerImageGenerationProvider(...)`       | Afbeeldingen genereren                                                                                                                          |
| `api.registerMusicGenerationProvider(...)`       | Muziek genereren                                                                                                                          |
| `api.registerVideoGenerationProvider(...)`       | Video genereren                                                                                                                          |
| `api.registerWebFetchProvider(...)`              | Provider voor ophalen/scrapen van het web                                                                                                               |
| `api.registerWebSearchProvider(...)`             | Zoeken op het web                                                                                                                                |
| `api.registerCompactionProvider(...)`            | Inplugbare backend voor transcript-Compaction                                                                                                   |

Workerproviders moeten hun id ook declareren in `contracts.workerProviders`.
De kern slaat duurzame intentie op vóór `provision(profile, operationId)`. Providers valideren instellingen vóór externe toewijzing en werpen `WorkerProviderError` bij permanente afwijzing van een profiel. `provision` moet dezelfde lease overnemen wanneer de bewerkings-id wordt herhaald.
De kern slaat de gevalideerde profielinstellingen bij de lease op en levert die momentopname aan `destroy({ leaseId, profile })`, die idempotent moet zijn, en `inspect({ leaseId, profile })`, die `active`, `destroyed` of `unknown` retourneert. Hierdoor kunnen providers levenscyclusaanroepen routeren na een herstart van de Gateway of verwijdering van een benoemd profiel. SSH-eindpunten gebruiken een `SecretRef` voor `keyRef`, nooit inline sleutelmateriaal, en bevatten een `hostKey` uit vertrouwde inrichtingsuitvoer als exact `algorithm base64`, zonder hostnaam of opmerking. De kern pint `hostKey` en vertrouwt nooit een sleutel van de eerste verbinding. Een provider die een dynamische `keyRef` aanmaakt, kan `resolveSshIdentity({ leaseId, profile, keyRef })` implementeren; wanneer aanwezig is die resolver gezaghebbend, terwijl providers zonder deze resolver de geconfigureerde generieke geheimresolver gebruiken.
Providers met hernieuwbare leases kunnen ook `renew(leaseId)` implementeren.
`inspect` moet een fout werpen bij tijdelijke of onbepaalde fouten; retourneer `unknown` alleen bij gezaghebbende afwezigheid. De kern markeert een actief lokaal record als verweesd, of behandelt de afwezigheid als voltooiing van de afbraak na een opgeslagen vernietigingsverzoek.

Embeddingproviders die met `api.registerEmbeddingProvider(...)` zijn geregistreerd, moeten
ook worden vermeld in `contracts.embeddingProviders` in het pluginmanifest. Dit
is het generieke embeddingoppervlak voor herbruikbare vectorgeneratie. Geheugenzoekopdrachten
kunnen dit generieke provideroppervlak gebruiken. De oudere interface
`api.registerMemoryEmbeddingProvider(...)` en
`contracts.memoryEmbeddingProviders` is verouderde compatibiliteit terwijl
bestaande geheugenspecifieke providers migreren.

Geheugenspecifieke providers die nog steeds een runtime-`batchEmbed(...)` aanbieden, blijven het
bestaande batchcontract per bestand gebruiken, tenzij hun runtime expliciet
`sourceWideBatchEmbed: true` instelt. Met die expliciete inschakeling kan de geheugenhost chunks uit
meerdere gewijzigde geheugenbestanden en ingeschakelde bronnen in één `batchEmbed(...)`-aanroep indienen,
tot aan de batchlimieten van de host. Batchadapters die JSONL-aanvraagbestanden uploaden, moeten
providertaken splitsen vóór zowel hun maximale uploadgrootte als hun
maximale aantal aanvragen wordt bereikt. De provider moet één embedding per invoerchunk in dezelfde volgorde als
`batch.chunks` retourneren; laat de vlag weg wanneer de provider batches per bestand verwacht of
de invoervolgorde niet kan behouden binnen een grotere bronbrede taak.

### Tools en opdrachten

Gebruik [`defineToolPlugin`](/nl/plugins/tool-plugins) voor eenvoudige plugins die uitsluitend tools bevatten
met vaste toolnamen. Gebruik `api.registerTool(...)` rechtstreeks voor gemengde plugins
of volledig dynamische toolregistratie.

| Methode                                 | Wat deze registreert                                                                                                                        |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerTool(tool, opts?)`        | Agenttool (vereist of `{ optional: true }`)                                                                                            |
| `api.registerCommand(def)`             | Aangepaste opdracht (omzeilt de LLM)                                                                                                        |
| `api.registerNodeHostCommand(command)` | Opdracht afgehandeld door `openclaw node run`; optionele `agentTool`-metadata kan deze als een voor de agent zichtbare tool beschikbaar maken terwijl de Node is verbonden |

Pluginopdrachten kunnen `agentPromptGuidance` instellen wanneer de agent een korte,
door de opdracht beheerde routeringshint nodig heeft. Laat die tekst uitsluitend over de opdracht zelf gaan; voeg geen
provider- of pluginspecifiek beleid toe aan de promptbuilders van de kern.

Richtlijnitems kunnen verouderde tekenreeksen zijn, die op elk promptoppervlak van toepassing zijn, of
gestructureerde items:

```ts
agentPromptGuidance: [
  "Algemene opdrachthint.",
  { text: "Toon dit alleen in de hoofdprompt van OpenClaw.", surfaces: ["openclaw_main"] },
];
```

Gestructureerde `surfaces` kan `openclaw_main`, `codex_app_server`,
`cli_backend`, `acp_backend` of `subagent` bevatten. `pi_main` blijft een verouderde alias
voor `openclaw_main`. Laat `surfaces` weg voor opzettelijke richtlijnen voor alle oppervlakken. Geef
geen lege `surfaces`-array door; deze wordt geweigerd, zodat onbedoeld verlies van bereik
niet leidt tot globale prompttekst.

Instructies voor ontwikkelaars van de systeemeigen Codex-appserver zijn strenger dan voor andere prompt-
oppervlakken: alleen richtlijnen die expliciet zijn beperkt tot `codex_app_server`, worden naar
die lane met hogere prioriteit gepromoveerd. Verouderde tekenreeksrichtlijnen en onbegrensde gestructureerde
richtlijnen blijven voor compatibiliteit beschikbaar voor niet-Codex-promptoppervlakken.

Node-hostopdrachten worden uitgevoerd op de verbonden Node-host, niet binnen het Gateway-
proces. Als `agentTool` aanwezig is, publiceert de Node een descriptor nadat
verbinding met de Gateway is geslaagd; de Gateway stelt deze alleen beschikbaar voor agentuitvoeringen zolang die
Node verbonden is en alleen als de `command` van de descriptor deel uitmaakt van het
goedgekeurde opdrachtoppervlak van de Node. Stel `agentTool.defaultPlatforms` in om een
niet-gevaarlijke opdracht toe te voegen aan de standaardtoegestane lijst met Node-opdrachten; vereis anders
expliciete `gateway.nodes.commands.allow` of een beleid voor Node-aanroepen. `agentTool.name`
moet veilig zijn voor providers: begin met een letter, gebruik alleen letters, cijfers,
onderstrepingstekens of koppeltekens en blijf binnen 64 tekens. Door MCP ondersteunde Node-tools
kunnen `agentTool.mcp`-metadata instellen, zodat catalogus- en toolzoekoppervlakken
de identiteit van de externe MCP-server/tool kunnen tonen, maar de uitvoering verloopt nog steeds via de
geadverteerde Node-opdracht.

### Infrastructuur

| Methode                                          | Wat deze registreert                                                      |
| ----------------------------------------------- | ---------------------------------------------------------------------- |
| `api.registerHook(events, handler, opts?)`      | Gebeurtenishook                                                             |
| `api.registerHttpRoute(params)`                 | HTTP-eindpunt van de Gateway                                                  |
| `api.registerGatewayMethod(name, handler)`      | RPC-methode van de Gateway                                                     |
| `api.registerGatewayDiscoveryService(service)`  | Adverteerder voor lokale Gateway-detectie                                     |
| `api.registerCli(registrar, opts?)`             | CLI-subopdracht                                                         |
| `api.registerNodeCliFeature(registrar, opts?)`  | CLI voor Node-functies onder `openclaw nodes`                                |
| `api.registerService(service)`                  | Achtergrondservice                                                     |
| `api.registerInteractiveHandler(registration)`  | Interactieve handler                                                    |
| `api.registerAgentToolResultMiddleware(...)`    | Runtime-middleware voor toolresultaten                                         |
| `api.registerMemoryPromptSupplement(builder)`   | Aanvullende promptsectie naast het geheugen                                |
| `api.registerMemoryPromptPreparation(prepare)`  | Asynchrone voorbereiding voor een promptsectie naast het geheugen                 |
| `api.registerMemoryCorpusSupplement(adapter)`   | Aanvullend corpus voor geheugenzoekopdrachten/-lezingen                                     |
| `api.registerHostedMediaResolver(resolver)`     | Resolver voor gehoste media-URL's in browserstijl                           |
| `api.registerMcpServerConnectionResolver(...)`  | MCP-transport per aanvrager (`url`/`headers`) voor een statische servernaam |
| `api.registerTextTransforms(transforms)`        | Door de Plugin beheerde compatibiliteitsherschrijvingen van prompt-/berichttekst                |
| `api.registerConfigMigration(migrate)`          | Lichtgewicht configuratiemigratie die wordt uitgevoerd voordat de Plugin-runtime wordt geladen           |
| `api.registerMigrationProvider(provider)`       | Importfunctie voor `openclaw migrate`                                        |
| `api.registerAutoEnableProbe(probe)`            | Configuratiecontrole die deze Plugin automatisch kan inschakelen                          |
| `api.registerReload(registration)`              | Beleid voor configuratievoorvoegsels voor herstarten/hot/noop bij herladen              |
| `api.registerNodeHostCommand(command)`          | Opdrachthandler die beschikbaar is voor gekoppelde Nodes                                |
| `api.registerNodeInvokePolicy(policy)`          | Toegestaanelijst-/goedkeuringsbeleid voor door Nodes aangeroepen opdrachten                    |
| `api.registerSecurityAuditCollector(collector)` | Bevindingenverzamelaar voor `openclaw security audit`                       |

#### Webhookwerk na bevestiging

Webhookroutes die een verzoek bevestigen voordat de verwerking is voltooid, moeten
dat losgekoppelde werk naar een eigen bijgehouden toelatingsroot verplaatsen:

```typescript
import { runDetachedWebhookWork } from "openclaw/plugin-sdk/webhook-request-guards";

void runDetachedWebhookWork(() => processWebhookEvent(event)).catch((error) => {
  runtime.error?.(`webhookverzending mislukt: ${String(error)}`);
});
```

Roep `runDetachedWebhookWork(...)` synchroon aan terwijl het HTTP-verzoek nog
is toegelaten. De helper reserveert onmiddellijk een onafhankelijke root en start vervolgens de
callback in de volgende microtaak, zodat de verzoekhandler eerst de
bevestiging kan schrijven. De geretourneerde promise neemt het callbackresultaat over; aanroepers
blijven verantwoordelijk voor de afhandeling van afwijzingen. Hierdoor blijft wachtrijwerk na bevestiging geaccepteerd en
wachten afvoerbewerkingen bij herstarten of opschorten erop. Handlers die alle verwerking afwachten
voordat ze terugkeren, hebben deze helper niet nodig.

#### MCP-verbindingen met een bereik per aanvrager

Houd de **identiteit** van de MCP-server statisch (naam, toolfilter) in `mcp.servers`, het
`mcpServers`-manifestveld van een systeemeigen Plugin of een bundelmanifest. Registreer desgewenst een verbindingsresolver, zodat elke vertrouwde
berichtaanvrager een eigen transport krijgt:

```ts
api.registerMcpServerConnectionResolver({
  serverName: "user-email",
  resolve: async (ctx) => {
    // ctx.requesterSenderId wordt door de host vertrouwd; verzin hier nooit een afzenderidentiteit.
    const token = await lookupUserToken(ctx.requesterSenderId);
    if (!token) {
      return null; // laat deze server weg voor de huidige uitvoering
    }
    return {
      url: "https://mcp.example.com/email",
      headers: { Authorization: `Bearer ${token}` },
    };
  },
});
```

Contractopmerkingen:

- De resolvercontext bevat alleen een door de host vertrouwde identiteit (`requesterSenderId`,
  optioneel `agentAccountId` / `messageChannel`). Toekomstige vertrouwde velden (bijvoorbeeld
  gebruikerscontext voor Cron/subagenten) kunnen aanvullend worden toegevoegd.
- Eén Plugin beheert één servernaam: een dubbele
  `registerMcpServerConnectionResolver` voor dezelfde `serverName` vanuit een andere
  Plugin wordt geweigerd met een foutdiagnose (de eerste registratie wint), zodat
  het eigenaarschap van de verbinding nooit afhangt van de laadvolgorde van Plugins.
- Toolnamen worden afgeleid van de volledige gedeclareerde serverset, zodat gedeeltelijke resolutie
  veilige servernamen nooit tussen aanvragers of beurten wijzigt. Core controleert niet
  of verschillende eindpunten voor aanvragers identieke toolschema's aanbieden; een
  resolver moet elke aanvrager naar dezelfde logische service verwijzen, anders
  verschillen toolschema's (en de stabiliteit van de promptcache) per aanvrager.
- Uitvoeringen zonder een vertrouwde `requesterSenderId` (Cron, subagent, Heartbeat, openbare
  Gateway) materialiseren nooit servers met een bereik per aanvrager. Er is geen gedeelde
  terugvalverbinding.
- `resolve` is begrensd op 10 seconden per server; bij een time-out of fout wordt die
  server voor de uitvoering weggelaten zonder dat statische MCP mislukt.
- Opgeloste verbindingen worden maximaal elke 5 minuten per aanvrager opnieuw gevalideerd:
  bij rotatie wordt het transport opnieuw opgebouwd met nieuwe inloggegevens en een `null`-resultaat
  trekt het in (de gecachte runtime wordt zelfs midden in een sessie verwijderd). Een ingetrokken of
  geroteerd inloggegeven kan daarom nog maximaal 5 minuten in gebruik blijven.
- Opgeloste `headers` worden nooit gelogd of persistent opgeslagen; Core bewaart alleen een tijdelijke
  in het geheugen opgeslagen digest met sleutel (proceslokale HMAC) om rotatie van inloggegevens te detecteren en
  registreert opgeloste inloggegevenswaarden voor headers/URL's bij het redactieregister
  voor logboeken/foutopsporingsvastlegging.
- Servers met een bereik per aanvrager maken geen MCP App-weergaven aan: een weergave bestaat langer dan de
  door de aanvrager geverifieerde uitvoering en de grens van de Gateway-weergave heeft geen aanvrager-
  identiteit, zodat appvoorbeelden voor deze servers standaard gesloten blijven. Toolresultaten
  worden niet beïnvloed.
- Statische servers zonder resolver behouden de bestaande levenscyclus met sessiebereik.
- **Regel voor levering via het harnas:** servers met een bereik per aanvrager komen nooit in systeemeigen
  MCP-clientconfiguratie van het harnas (Codex-thread `mcp_servers`, CLI `-c mcp_servers=…` of een
  andere sessiegedeelde MCP-projectie). Harnassen leveren ze in plaats daarvan als tools met uitvoeringsbereik:
  - Ingesloten runner: MCP-runtime van de sessie + bundeltools (statisch + met bereik).
  - Codex-appserver: dynamische tools via
    `materializeRequesterScopedMcpToolsForHarnessRun` (alleen met bereik;
    statische servers blijven op de systeemeigen MCP-client van Codex).
- **Specificaties** van tools met bereik zijn sessiestabiel na de eerste succesvolle resolutie in
  die sessie, zodat harnassen met gedeelde threads (Codex) niet van thread wisselen wanneer
  afzenders veranderen. Voordat een aanvrager iets oplost, worden geen specificaties met bereik geadverteerd.
- Niet-geverifieerde aanvragers op een harnas met gedeelde threads zien nog steeds de geadverteerde
  tools met bereik; het aanroepen ervan retourneert voor die aanvrager een duidelijke fout dat de tool niet verbonden is.
  OpenClaw valt nooit terug op de inloggegevens van een andere aanvrager.

Bouwers van aanvullingen op geheugenprompts ontvangen optionele context voor `agentId`,
`agentSessionKey` en `sandboxed`. Aanroepen voor geheugencorpusaanvulling `search`
en `get` ontvangen optionele context voor `agentId` en `sandboxed`. Plugins met
door agenten beheerde opslag moeten die opslag voor elke aanroep oplossen in plaats van
tijdens de registratie één globaal pad vast te leggen. Als een agent-id vereist is maar
ontbreekt in een bewerking met meerdere agenten, sluit dan standaard af in plaats van een
willekeurige agent te kiezen.

Gebruik `registerMemoryPromptPreparation(...)` wanneer prompttekst afhankelijk is van asynchrone
Plugin-status. De callback wordt eenmaal vóór elke volledige agentprompt uitgevoerd en ontvangt
dezelfde tool-, agent-, sessie- en sandboxcontext als synchrone bouwers van geheugenprompts.
Valideer de huidige instantie van de opslageigenaar voordat je persistente status laadt en
retourneer vervolgens alleen regels voor die uitvoering. OpenClaw bevriest die regels en
geeft het onveranderlijke resultaat door aan de synchrone promptassemblage. Houd persistentie,
atomaire vervanging en verwijdering bij het verwijderen van de eigenaar binnen de beherende Plugin; poll of
lees geen bestanden vanuit een promptbouwer.

Interactieve handlers van Telegram kunnen `{ submitText }` retourneren om tekst via
het normale inkomende agentpad van Telegram te routeren nadat de handler is geslaagd. OpenClaw behoudt
de callbackknop wanneer het beleid voor inkomende berichten de tekst overslaat of de verwerking mislukt, zodat
de gebruiker het opnieuw kan proberen nadat de blokkerende voorwaarde is gewijzigd. Dit resultaatveld is
specifiek voor Telegram; andere kanalen behouden hun eigen interactieve resultaatcontracten.

### Hosthooks voor workflow-Plugins

Hosthooks zijn de SDK-seams voor Plugins die moeten deelnemen aan de levenscyclus van de host
in plaats van alleen een provider, kanaal of tool toe te voegen. Het zijn
generieke contracten; de Planmodus kan ze gebruiken, maar dat geldt ook voor goedkeuringsworkflows,
beleidscontroles voor werkruimten, achtergrondmonitors, installatiewizards en begeleidende UI-
Plugins.

| Methode                                                                               | Contract waarvoor deze verantwoordelijk is                                                                                                                                           |
| ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.session.state.registerSessionExtension(...)`                                    | Door de Plugin beheerde, JSON-compatibele sessiestatus die via Gateway-sessies wordt geprojecteerd                                                                             |
| `api.session.workflow.enqueueNextTurnInjection(...)`                                 | Duurzame precies-eenmaal-context die voor één sessie in de volgende agentbeurt wordt geïnjecteerd                                                                             |
| `api.registerTrustedToolPolicy(...)`                                                 | Door het manifest afgeschermd, vertrouwd toolbeleid vóór Plugins dat toolparameters kan blokkeren of herschrijven                                                                        |
| `api.registerToolMetadata(...)`                                                      | Weergavemetadata voor de toolcatalogus zonder de toolimplementatie te wijzigen                                                                                     |
| `api.registerCommand(...)`                                                           | Afgebakende Plugin-opdrachten; opdrachtresultaten kunnen `continueAgent: true` of `suppressReply: true` instellen; native Discord-opdrachten ondersteunen `descriptionLocalizations` |
| `api.session.controls.registerControlUiDescriptor(...)`                              | Bijdragebeschrijvingen voor de Control UI voor sessie-, tool-, uitvoerings-, instellingen- of tabbladoppervlakken                                                                      |
| `api.lifecycle.registerRuntimeLifecycle(...)`                                        | Opschooncallbacks voor door de Plugin beheerde runtimebronnen bij reset-, verwijder- en herlaadpaden                                                                          |
| `api.agent.events.registerAgentEventSubscription(...)`                               | Opgeschoonde gebeurtenisabonnementen voor workflowstatus en monitors                                                                                              |
| `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`  | Tijdelijke Plugin-status per uitvoering die aan het einde van de uitvoeringslevenscyclus wordt gewist                                                                                             |
| `api.session.workflow.registerSessionSchedulerJob(...)`                              | Opschoonmetadata voor door de Plugin beheerde plannertaken; plant geen werk en maakt geen taakrecords                                                            |
| `api.session.workflow.sendSessionAttachment(...)`                                    | Alleen gebundelde, door de host bemiddelde levering van bestandsbijlagen aan de actieve directe uitgaande sessieroute                                                            |
| `api.session.workflow.scheduleSessionTurn(...)` / `unscheduleSessionTurnsByTag(...)` | Alleen gebundelde, door Cron ondersteunde geplande sessiebeurten plus opschoning op basis van tags                                                                                    |
| `api.session.controls.registerSessionAction(...)`                                    | Getypeerde sessieacties die clients via de Gateway kunnen verzenden                                                                                             |

Een `surface: "tab"`-beschrijving voegt een zijbalktabblad toe aan de Control UI. De tabbladbeschrijvingen van actieve
Plugins worden aan dashboardclients bekendgemaakt in de hello van de Gateway
(`controlUiTabs`), zodat het tabblad alleen verschijnt wanneer de Plugin is ingeschakeld.
Gebundelde Plugins kunnen een volwaardige dashboardweergave voor hun tabblad leveren; andere
Plugins kunnen `path` instellen op een HTTP-route van de Plugin (zie
`api.registerHttpRoute(...)`) die het dashboard in een gesandboxd frame weergeeft.
`icon` is een hint voor de naam van een dashboardpictogram, `group` kiest de zijbalksectie
(`control` of `agent`), `order` bepaalt de volgorde tussen Plugin-tabbladen en `requiredScopes`
verbergt het tabblad voor verbindingen zonder die operatorbereiken:

Registreer voor een door de Gateway beveiligd extern tabblad de beschrijving `path` onder een
HTTP-route `auth: "gateway"` van dezelfde Plugin. Na de geverifieerde initialisatie krijgt de browser een
kortdurende HttpOnly-toekenning die is beperkt tot die Plugin en routehoofdmap, zodat het
gesandboxde frame kan laden zonder het bearer-token van de Gateway naar de URL
of JavaScript te kopiëren. De geverifieerde ouder vernieuwt de toekenning zolang het externe tabblad
actief is en voordat het na navigatie of hervatting van de browser wordt gekoppeld. Ook
controleert deze de toekenning vanuit dezelfde ondoorzichtige sandbox voordat het frame wordt gekoppeld, zodat
browserprivacymodi die de cookie blokkeren veilig gesloten blijven met een niet-beschikbaar paneel.
De frametoekenning accepteert alleen `GET` en `HEAD` en bevat altijd
`operator.read`; `requiredScopes` regelt de zichtbaarheid van het tabblad, maar verruimt de
cookietoekenning nooit. Mutaties blijven plaatsvinden via expliciet door de Gateway geverifieerde ouder- of
bearer-oppervlakken. Externe tabbladen vereisen HTTPS/Tailscale Serve of een
door de browser vertrouwde loopback-oorsprong; gewone HTTP op een LAN-host toont de
fout voor een beveiligde context in plaats van een paneel te koppelen dat zich niet kan verifiëren.
Volledige blokkering van cookies van derden maakt door de Gateway beveiligde tabbladen eveneens niet beschikbaar.
Zoals bij alle native Plugin-oppervlakken blijft het frame binnen de vertrouwensgrens van de geïnstalleerde
Plugin; OpenClaw behandelt geïnstalleerde Plugins niet als onderling
geïsoleerde browserbeveiligingsprincipalen.
Cookietoekenningen gebruiken de hostnaamgrens van de browser, niet de poortgrens. Plaats
geen onderling niet-vertrouwde services samen op de hostnaam van de Gateway, zelfs niet op andere
poorten.
Tabbladen die worden ondersteund door door de Plugin beheerde verificatie behouden hun directe iframe-gedrag en
vragen of vereisen deze Gateway-toekenning niet.

```typescript
api.session.controls.registerControlUiDescriptor({
  surface: "tab",
  id: "logbook",
  label: "Logboek",
  description: "Je dag als tijdlijn, opgebouwd uit schermafbeeldingen.",
  icon: "sun",
  group: "control",
  requiredScopes: ["operator.write"],
});
```

Gebruik de gegroepeerde naamruimten voor nieuwe Plugin-code:

- `api.session.state.registerSessionExtension(...)`
- `api.session.workflow.enqueueNextTurnInjection(...)`
- `api.session.workflow.registerSessionSchedulerJob(...)`
- `api.session.workflow.sendSessionAttachment(...)`
- `api.session.workflow.scheduleSessionTurn(...)`
- `api.session.workflow.unscheduleSessionTurnsByTag(...)`
- `api.session.controls.registerSessionAction(...)`
- `api.session.controls.registerControlUiDescriptor(...)`
- `api.agent.events.registerAgentEventSubscription(...)`
- `api.agent.events.emitAgentEvent(...)`
- `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`
- `api.lifecycle.registerRuntimeLifecycle(...)`

De equivalente platte methoden blijven beschikbaar als verouderde compatibiliteitsaliassen
voor bestaande Plugins. Voeg geen nieuwe Plugin-code toe die rechtstreeks
`api.registerSessionExtension`, `api.enqueueNextTurnInjection`,
`api.registerControlUiDescriptor`, `api.registerRuntimeLifecycle`,
`api.registerAgentEventSubscription`, `api.emitAgentEvent`,
`api.setRunContext`, `api.getRunContext`, `api.clearRunContext`,
`api.registerSessionSchedulerJob`, `api.registerSessionAction`,
`api.sendSessionAttachment`, `api.scheduleSessionTurn` of
`api.unscheduleSessionTurnsByTag` aanroept.

`scheduleSessionTurn(...)` is een sessiegebonden gemak boven op de Cron-planner
van de Gateway. Cron beheert de timing en maakt het achtergrondtaakrecord wanneer de
beurt wordt uitgevoerd; de Plugin SDK beperkt alleen de doelsessie, door de Plugin beheerde
naamgeving en opschoning. Gebruik `api.runtime.tasks.managedFlows` binnen de geplande
beurt wanneer het werk zelf duurzame Task Flow-status met meerdere stappen vereist.

De contracten splitsen de bevoegdheid bewust op:

- Externe Plugins kunnen eigenaar zijn van sessie-uitbreidingen, UI-beschrijvingen, opdrachten, toolmetadata, injecties voor de volgende beurt en normale hooks.
- Vertrouwd toolbeleid wordt vóór gewone `before_tool_call`-hooks uitgevoerd en wordt door de
  host vertrouwd. Gebundeld beleid wordt eerst uitgevoerd; beleid van geïnstalleerde Plugins vereist
  expliciete inschakeling plus de lokale id's ervan in
  `contracts.trustedToolPolicies` en wordt daarna uitgevoerd in de laadvolgorde van de Plugins. Beleids-id's
  zijn beperkt tot de registrerende Plugin.
- Eigendom van gereserveerde opdrachten is alleen voor gebundelde Plugins. Externe Plugins moeten hun
  eigen opdrachtnamen of aliassen gebruiken.
- `allowPromptInjection=false` schakelt hooks uit die prompts wijzigen, waaronder
  `agent_turn_prepare`, `before_prompt_build`, `heartbeat_prompt_contribution`
  en `enqueueNextTurnInjection`.

Voorbeelden van niet-Plan-consumenten:

| Plugin-archetype             | Gebruikte hooks                                                                                                                             |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Goedkeuringsworkflow            | Sessie-uitbreiding, voortzetting van opdrachten, injectie voor de volgende beurt, UI-beschrijving                                                            |
| Beleidscontrole voor budget/werkruimte | Vertrouwd toolbeleid, toolmetadata, sessieprojectie                                                                                 |
| Achtergrondmonitor voor de levenscyclus | Opschoning van de runtimelevenscyclus, abonnement op agentgebeurtenissen, eigendom/opschoning van de sessieplanner, bijdrage aan de Heartbeat-prompt, UI-beschrijving |
| Installatie- of onboardingwizard   | Sessie-uitbreiding, afgebakende opdrachten, Control UI-beschrijving                                                                              |

<Note>
  Gereserveerde naamruimten voor kernbeheer (`config.*`, `exec.approvals.*`, `wizard.*`,
  `update.*`) blijven altijd `operator.admin`, zelfs als een Plugin een
  nauwer methodebereik van de Gateway probeert toe te wijzen. Geef voor methoden die door een
  Plugin worden beheerd de voorkeur aan Plugin-specifieke voorvoegsels.
</Note>

<Accordion title="Wanneer middleware voor toolresultaten gebruiken">
  Gebundelde Plugins en expliciet ingeschakelde geïnstalleerde Plugins met overeenkomende
  manifestcontracten kunnen `api.registerAgentToolResultMiddleware(...)` gebruiken wanneer
  ze een toolresultaat na de uitvoering en voordat de runtime
  dat resultaat terugvoert naar het model moeten herschrijven. Dit is de vertrouwde, runtimeneutrale
  naad voor asynchrone uitvoerreducers zoals tokenjuice.

Plugins moeten voor elke beoogde runtime `contracts.agentToolResultMiddleware` declareren,
bijvoorbeeld `["openclaw", "codex"]`. Geïnstalleerde Plugins zonder dat
contract of zonder expliciete inschakeling kunnen deze middleware niet registreren; gebruik
normale OpenClaw-Plugin-hooks voor werk waarvoor geen timing van toolresultaten vóór het model
nodig is. Het oude registratiepad voor uitbreidingsfabrieken dat alleen voor de
ingebedde runner bestemd was, is verwijderd.
</Accordion>

### Registratie voor Gateway-detectie

Met `api.registerGatewayDiscoveryService(...)` kan een Plugin de actieve
Gateway bekendmaken via een lokaal detectietransport zoals mDNS/Bonjour. OpenClaw roept de
service aan tijdens het opstarten van de Gateway wanneer lokale detectie is ingeschakeld, geeft de
huidige Gateway-poorten en niet-geheime TXT-hintgegevens door en roept de geretourneerde
`stop`-handler aan tijdens het afsluiten van de Gateway.

```typescript
api.registerGatewayDiscoveryService({
  id: "my-discovery",
  async advertise(ctx) {
    const handle = await startMyAdvertiser({
      gatewayPort: ctx.gatewayPort,
      tls: ctx.gatewayTlsEnabled,
      displayName: ctx.machineDisplayName,
    });
    return { stop: () => handle.stop() };
  },
});
```

Plugins voor Gateway-detectie mogen bekendgemaakte TXT-waarden niet als geheimen of
verificatie behandelen. Detectie is een routeringshint; Gateway-verificatie en TLS-pinning
blijven verantwoordelijk voor vertrouwen.

### Registratiemetadata voor de CLI

`api.registerCli(registrar, opts?)` accepteert twee soorten opdrachtmetadata:

- `commands`: expliciete opdrachtnamen die eigendom zijn van de registrator
- `descriptors`: tijdens het parseren gebruikte opdrachtbeschrijvingen voor CLI-help,
  routering en luie registratie van de Plugin-CLI
- `parentPath`: optioneel pad naar de bovenliggende opdracht voor geneste opdrachtgroepen, zoals
  `["nodes"]`

Geef voor functies met gekoppelde Nodes de voorkeur aan
`api.registerNodeCliFeature(registrar, opts?)`. Dit is een kleine wrapper rond
`api.registerCli(..., { parentPath: ["nodes"] })` en maakt opdrachten zoals
`openclaw nodes canvas` expliciete Node-functies die door de Plugin worden beheerd.

Als je wilt dat een Plugin-opdracht lui geladen blijft in het normale pad van de hoofd-CLI,
geef dan `descriptors` op die elke opdrachthoofdmap op het hoogste niveau bestrijken die door die
registrator wordt aangeboden.

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "Matrix-accounts, verificatie, apparaten en profielstatus beheren",
        hasSubcommands: true,
      },
    ],
  },
);
```

Geneste opdrachten ontvangen de opgeloste bovenliggende opdracht als `program`:

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerNodesCanvasCommands } = await import("./src/cli.js");
    registerNodesCanvasCommands(program);
  },
  {
    parentPath: ["nodes"],
    descriptors: [
      {
        name: "canvas",
        description: "Canvasinhoud van een gekoppelde node vastleggen of renderen",
        hasSubcommands: true,
      },
    ],
  },
);
```

Gebruik `commands` alleen zelfstandig wanneer luie registratie van de hoofd-CLI niet nodig is.
Dat eager compatibiliteitspad blijft ondersteund, maar installeert geen
door descriptors ondersteunde tijdelijke aanduidingen voor lui laden tijdens het parsen.

### Registratie van CLI-backends

Met `api.registerCliBackend(...)` kan een plugin eigenaar zijn van de standaardconfiguratie voor een lokale
AI-CLI-backend zoals `claude-cli` of `my-cli`.

- De backend-`id` wordt het providervoorvoegsel in modelreferenties zoals `my-cli/gpt-5`.
- De backend-`config` is de gezaghebbende opdrachtadapter: het gedrag voor argv, omgeving,
  parser, sessie, afbeeldingen en betrouwbaarheid bevindt zich in de plugincode.
- Gebruikers selecteren de backend via modelreferenties of modelgebonden `agentRuntime.id`;
  `openclaw.json` herschrijft de adapter niet.
- Gebruik `normalizeConfig` wanneer geregistreerde statische velden een runtimebewuste
  normalisatiebewerking nodig hebben.
- Gebruik `resolveExecutionArgs` voor argv-herschrijvingen per aanvraag die bij
  het CLI-dialect horen, zoals het toewijzen van OpenClaw-denkniveaus aan een systeemeigen inspanningsvlag.
  De hook ontvangt `ctx.executionMode`; gebruik `"side-question"` om
  backend-eigen isolatievlaggen toe te voegen voor tijdelijke `/btw`-aanroepen. Als die vlaggen
  systeemeigen tools betrouwbaar uitschakelen voor een CLI waarop ze anders altijd actief zijn, declareer dan
  ook `sideQuestionToolMode: "disabled"`.
- Gebruik `prepareExecution` voor een backend-eigen startomgeving of tijdelijke
  authenticatie-/configuratiebruggen. De `ctx.contextTokenBudget` ervan is de effectieve tokenlimiet
  die voor de uitvoering is geselecteerd, zodat backends met systeemeigen compaction hun
  eigen drempel kunnen afstemmen zonder providerspecifieke vertakkingen in de kern. Deze ontvangt ook de
  door de kern voorbereide `ctx.env` wanneer backend-staging gebundelde MCP-instellingen moet uitbreiden.
- Backends die alle systeemeigen tools voor een specifieke uitvoering kunnen uitschakelen, mogen
  `nativeToolMode: "selectable"` declareren. Beperkte aanroepen geven een exacte
  `ctx.toolAvailability.native`-lijst plus canonieke
  `ctx.toolAvailability.openClaw`-namen door. Declareer
  `toolAvailabilityEnforcement: "execution-args"` en dwing het contract af in
  de uiteindelijke argv voor nieuw starten/hervatten, of declareer `"prepare-execution"`, dwing het af in
  het gestagede beleid en retourneer `toolAvailabilityEnforced: true`. OpenClaw schakelt
  systeemeigen tools uit voor runtimebeperkingen zoals cron-`toolsAllow` en sluit af bij fouten wanneer
  het gedeclareerde afdwingingspad onvolledig is.

Zie voor een volledige auteursgids
[CLI-backendplugins](/nl/plugins/cli-backend-plugins).

### Exclusieve slots

| Methode                                     | Wat deze registreert                                                                                                                                                                                  |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Contextengine (één tegelijk actief). Levenscycluscallbacks ontvangen `runtimeSettings` wanneer de host model-/provider-/modusdiagnostiek kan leveren; oudere strikte engines worden zonder die sleutel opnieuw geprobeerd. |
| `api.registerMemoryCapability(capability)` | Geünificeerde geheugencapaciteit                                                                                                                                                                          |

### Verouderde adapters voor geheugenembeddings

| Methode                                         | Wat deze registreert                              |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adapter voor geheugenembeddings voor de actieve plugin |

- `registerMemoryCapability` is de exclusieve API voor geheugenplugins.
- `registerMemoryCapability` kan ook `publicArtifacts.listArtifacts(...)`
  beschikbaar stellen voor door de host beheerde exports. Aanvullende plugins die deze gedeclareerde
  artefacten inventariseren, gebruiken nog steeds `listActiveMemoryPublicArtifacts(...)` uit de behouden
  `openclaw/plugin-sdk/memory-host-core`-façade totdat er een gerichte openbare API voor consumers
  bestaat; ze mogen de privé-indeling van een andere plugin niet benaderen.
- `MemoryFlushPlan.model` kan de flush-beurt vastzetten op een exacte `provider/model`-
  referentie, zoals `ollama/qwen3:8b`, zonder de actieve terugvalketen
  over te nemen.
- `registerMemoryEmbeddingProvider` is verouderd. Nieuwe embeddingproviders
  moeten `api.registerEmbeddingProvider(...)` en
  `contracts.embeddingProviders` gebruiken.
- Bestaande geheugenspecifieke providers blijven tijdens het migratievenster
  werken, maar plugininspectie rapporteert dit als compatibiliteitsschuld voor
  niet-gebundelde plugins.

### Gebeurtenissen en levenscyclus

| Methode                                       | Wat deze doet                  |
| -------------------------------------------- | ----------------------------- |
| `api.on(hookName, handler, opts?)`           | Getypeerde levenscyclushook          |
| `api.onConversationBindingResolved(handler)` | Callback voor gespreksbinding |

Zie [Pluginhooks](/nl/plugins/hooks) voor voorbeelden, veelvoorkomende hooknamen en
bewakingssemantiek.

### Beslissingssemantiek van hooks

`before_install` is een levenscyclushook voor de pluginruntime, niet het beleidsoppervlak
voor installatie door operators. Gebruik `security.installPolicy` wanneer een beslissing om toe te staan/blokkeren
zowel CLI- als Gateway-ondersteunde installatie- of updatepaden moet omvatten.

- `before_tool_call`: het retourneren van `{ block: true }` is definitief. Zodra een handler dit instelt, worden handlers met een lagere prioriteit overgeslagen.
- `before_tool_call`: het retourneren van `{ block: false }` wordt behandeld als geen beslissing (hetzelfde als het weglaten van `block`), niet als een overschrijving.
- `before_install`: het retourneren van `{ block: true }` is definitief. Zodra een handler dit instelt, worden handlers met een lagere prioriteit overgeslagen.
- `before_install`: het retourneren van `{ block: false }` wordt behandeld als geen beslissing (hetzelfde als het weglaten van `block`), niet als een overschrijving.
- `reply_dispatch`: het retourneren van `{ handled: true, ... }` is definitief. Zodra een handler de dispatch opeist, worden handlers met een lagere prioriteit en het standaardpad voor modeldispatch overgeslagen.
- `message_sending`: het retourneren van `{ cancel: true }` is definitief. Zodra een handler dit instelt, worden handlers met een lagere prioriteit overgeslagen.
- `message_sending`: het retourneren van `{ cancel: false }` wordt behandeld als geen beslissing (hetzelfde als het weglaten van `cancel`), niet als een overschrijving.
- `message_received`: gebruik het getypeerde veld `threadId` wanneer routering van inkomende threads/onderwerpen nodig is. Behoud `metadata` voor kanaalspecifieke extra's.
- `message_sending`: gebruik de getypeerde routeringsvelden `replyToId` / `threadId` voordat wordt teruggevallen op kanaalspecifieke `metadata`.
- `gateway_start`: gebruik `ctx.config`, `ctx.workspaceDir` en `ctx.getCron?.()` voor opstartstatus waarvan de Gateway eigenaar is, in plaats van te vertrouwen op interne `gateway:startup`-hooks. Cron wordt op dit moment mogelijk nog geladen.
- `cron_reconciled`: bouw een volledige externe cronprojectie opnieuw op na het opstarten of herladen van de planner. Deze omvat `reason` en de effectieve `enabled`-status, inclusief `enabled: false`, terwijl `ctx.getCron?.()` de exact afgestemde planner retourneert. Geef `ctx.abortSignal` door aan duurzaam projectiewerk; dit wordt afgebroken wanneer die momentopname van de planner wordt vervangen of de Gateway wordt gesloten.
- `cron_changed`: observeer wijzigingen in de cronlevenscyclus waarvan de Gateway eigenaar is. `scheduled`- en `removed`-gebeurtenissen zijn afstemmingshints na een commit, geen geordend deltalogboek. De `event.nextRunAtMs` van een geplande gebeurtenis ontbreekt wanneer de taak geen volgend wekmoment heeft; een verwijderde gebeurtenis bevat nog steeds de momentopname van de verwijderde taak.

Externe wekplanners moeten `cron_changed`-gebeurtenissen debouncen of samenvoegen
en vervolgens de volledige duurzame weergave opnieuw lezen vanuit de planner die het laatst door
`cron_reconciled` is vastgelegd. Neem de planner niet over uit een `cron_changed`-context: een
losgekoppelde hint van een oudere planner kan overlappen met een latere herlaadbewerking.

Gebruik `cron_reconciled` als de trigger voor een volledige momentopname voor duurzame status die is geladen bij
het opstarten van de Gateway of het vervangen van de planner. Deze wordt niet opnieuw afgespeeld bij een hot reload
van alleen een plugin. Observatiehandlers worden parallel uitgevoerd en fire-and-forget-
dispatches kunnen overlappen, dus consumers mogen niet afhankelijk zijn van de voltooiingsvolgorde van gebeurtenissen.
Behoud OpenClaw als de bron van waarheid voor controles op verschuldigde taken en uitvoering.

Zie [Veilige externe cronprojectie](/nl/plugins/hooks#safe-external-cron-projection) voor een
single-flight-adapter met duurzame vervanging, opnieuw proberen/back-off en netjes
afsluiten.

### Velden van het API-object

| Veld                    | Type                      | Beschrijving                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Plugin-ID                                                                                   |
| `api.name`               | `string`                  | Weergavenaam                                                                                |
| `api.version`            | `string?`                 | Pluginversie (optioneel)                                                                   |
| `api.description`        | `string?`                 | Pluginbeschrijving (optioneel)                                                               |
| `api.source`             | `string`                  | Bronpad van de plugin                                                                          |
| `api.rootDir`            | `string?`                 | Hoofdmap van de plugin (optioneel)                                                            |
| `api.config`             | `OpenClawConfig`          | Huidige configuratiemomentopname (actieve momentopname van de in-memory runtime indien beschikbaar)                  |
| `api.pluginConfig`       | `Record<string, unknown>` | Pluginspecifieke configuratie uit `plugins.entries.<id>.config`                                   |
| `api.runtime`            | `PluginRuntime`           | [Runtimehelpers](/nl/plugins/sdk-runtime)                                                     |
| `api.logger`             | `PluginLogger`            | Logger met beperkt bereik (`debug`, `info`, `warn`, `error`)                                            |
| `api.registrationMode`   | `PluginRegistrationMode`  | Huidige laadmodus; `"setup-runtime"` is het lichtgewicht opstart-/installatievenster vóór de volledige invoer |
| `api.resolvePath(input)` | `(string) => string`      | Pad relatief ten opzichte van de hoofdmap van de plugin oplossen                                                        |

## Conventie voor interne modules

Gebruik binnen je plugin lokale barrel-bestanden voor interne imports:

```text
my-plugin/
  api.ts            # Openbare exports voor externe consumers
  runtime-api.ts    # Runtime-exports uitsluitend voor intern gebruik
  index.ts          # Ingangspunt van de plugin
  setup-entry.ts    # Lichtgewicht ingangspunt uitsluitend voor installatie (optioneel)
```

<Warning>
  Importeer je eigen plugin nooit via `openclaw/plugin-sdk/<your-plugin>`
  vanuit productiecode. Leid interne imports via `./api.ts` of
  `./runtime-api.ts`. Het SDK-pad is uitsluitend het externe contract.
</Warning>

Openbare oppervlakken van gebundelde plugins die via facades worden geladen (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` en vergelijkbare openbare invoerbestanden) geven de voorkeur aan de
actieve momentopname van de runtimeconfiguratie wanneer OpenClaw al actief is. Als er nog geen
runtimemomentopname bestaat, vallen ze terug op het opgeloste configuratiebestand op schijf.
Facades van verpakte gebundelde plugins moeten via de plugin-
facadeladers van OpenClaw worden geladen; directe imports vanuit `dist/extensions/...` omzeilen de controles van het manifest
en de runtime-sidecar die verpakte installaties gebruiken voor code die eigendom is van plugins.

Providerplugins kunnen een beperkte, pluginlokale contract-barrel beschikbaar stellen wanneer een
helper bewust providerspecifiek is en nog niet thuishoort in een generiek SDK-
subpad. Gebundelde voorbeelden:

- **Anthropic**: openbare `api.ts`- / `contract-api.ts`-koppeling voor Claude-
  bètaheaders en `service_tier`-streamhelpers.
- **`@openclaw/openai-provider`**: `api.ts` exporteert providerbouwers,
  helpers voor standaardmodellen en realtime-providerbouwers.
- **`@openclaw/openrouter-provider`**: `api.ts` exporteert de providerbouwer
  plus helpers voor onboarding/configuratie.

<Warning>
  Productiecode van extensies moet ook imports uit `openclaw/plugin-sdk/<other-plugin>`
  vermijden. Als een helper werkelijk wordt gedeeld, verplaats deze dan naar een neutraal SDK-subpad,
  zoals `openclaw/plugin-sdk/speech`, `.../provider-model-shared` of een ander
  op mogelijkheden gericht oppervlak, in plaats van twee plugins aan elkaar te koppelen.
</Warning>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Invoerpunten" icon="door-open" href="/nl/plugins/sdk-entrypoints">
    Opties voor `definePluginEntry` en `defineChannelPluginEntry`.
  </Card>
  <Card title="Runtimehelpers" icon="gears" href="/nl/plugins/sdk-runtime">
    Volledige referentie voor de naamruimte `api.runtime`.
  </Card>
  <Card title="Installatie en configuratie" icon="sliders" href="/nl/plugins/sdk-setup">
    Verpakking, manifesten en configuratieschema's.
  </Card>
  <Card title="Testen" icon="vial" href="/nl/plugins/sdk-testing">
    Testhulpprogramma's en lintregels.
  </Card>
  <Card title="SDK-migratie" icon="arrows-turn-right" href="/nl/plugins/sdk-migration">
    Migreren vanaf verouderde oppervlakken.
  </Card>
  <Card title="Interne werking van plugins" icon="diagram-project" href="/nl/plugins/architecture">
    Diepgaande architectuur en mogelijkhedenmodel.
  </Card>
</CardGroup>
