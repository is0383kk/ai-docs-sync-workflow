---
read_when:
    - Native OpenClaw-plugins bouwen of debuggen
    - Inzicht in het Plugin-capaciteitsmodel of de eigendomsgrenzen
    - Werken aan de laadpijplijn of het register van de Plugin
    - Runtime-hooks voor providers of kanaalplugins implementeren
sidebarTitle: Internals
summary: 'Interne werking van Plugins: capabilitymodel, eigenaarschap, contracten, laadpijplijn en runtimehelpers'
title: Interne werking van de Plugin
x-i18n:
    generated_at: "2026-07-27T05:06:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d47551b1bc2f71ce2ade3dfdd14bff8ee187616c3807f8101c1a3236e1443cc1
    source_path: plugins/architecture.md
    workflow: 16
---

Dit is de **diepgaande architectuurreferentie** voor het pluginsysteem van OpenClaw. Begin voor praktische handleidingen met een van de onderstaande gerichte pagina's.

<CardGroup cols={2}>
  <Card title="Plugins installeren en gebruiken" icon="plug" href="/nl/tools/plugin">
    Handleiding voor eindgebruikers voor het toevoegen, inschakelen en oplossen van problemen met plugins.
  </Card>
  <Card title="Plugins bouwen" icon="rocket" href="/nl/plugins/building-plugins">
    Tutorial voor je eerste plugin met het kleinste werkende manifest.
  </Card>
  <Card title="Kanaalplugins" icon="comments" href="/nl/plugins/sdk-channel-plugins">
    Bouw een plugin voor een berichtenkanaal.
  </Card>
  <Card title="Providerplugins" icon="microchip" href="/nl/plugins/sdk-provider-plugins">
    Bouw een plugin voor een modelprovider.
  </Card>
  <Card title="SDK-overzicht" icon="book" href="/nl/plugins/sdk-overview">
    Referentie voor de importmap en registratie-API.
  </Card>
</CardGroup>

## Openbaar capaciteitenmodel

Capaciteiten vormen het openbare model voor **native plugins** binnen OpenClaw. Elke native OpenClaw-plugin registreert zich voor een of meer capaciteitstypen:

| Capaciteit             | Registratiemethode                              | Voorbeeldplugins                                             |
| ---------------------- | ------------------------------------------------ | ----------------------------------------------------------- |
| Tekstinferentie         | `api.registerProvider(...)`                      | `anthropic`, `openai`                                       |
| CLI-inferentiebackend  | `api.registerCliBackend(...)`                    | `anthropic`, `openai`                                       |
| Embeddings             | `api.registerEmbeddingProvider(...)`             | Vectorplugins die eigendom zijn van providers                               |
| Spraak                 | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`                                   |
| Realtime transcriptie | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                                                    |
| Realtime spraak         | `api.registerRealtimeVoiceProvider(...)`         | `google`, `openai`                                          |
| Mediabegrip    | `api.registerMediaUnderstandingProvider(...)`    | `google`, `openai`                                          |
| Transcriptiebron     | `api.registerTranscriptSourceProvider(...)`      | `discord`, `google-meet`, `teams-meetings`, `zoom-meetings` |
| Afbeeldingen genereren       | `api.registerImageGenerationProvider(...)`       | `fal`, `google`, `openai`                                   |
| Muziek genereren       | `api.registerMusicGenerationProvider(...)`       | `fal`, `google`, `minimax`                                  |
| Video genereren       | `api.registerVideoGenerationProvider(...)`       | `fal`, `google`, `qwen`                                     |
| Webinhoud ophalen              | `api.registerWebFetchProvider(...)`              | `firecrawl`                                                 |
| Zoeken op het web             | `api.registerWebSearchProvider(...)`             | `brave`, `firecrawl`, `google`                              |
| Kanaal / berichten    | `api.registerChannel(...)`                       | `matrix`, `msteams`                                         |
| Gateway-detectie      | `api.registerGatewayDiscoveryService(...)`       | `bonjour`                                                   |

<Note>
Een plugin die nul capaciteiten registreert, maar wel hooks, tools, detectieservices of achtergrondservices levert, is een **verouderde plugin met alleen hooks**. Dat patroon wordt nog steeds volledig ondersteund.
</Note>

### Standpunt over externe compatibiliteit

Het capaciteitenmodel is in de kern geïmplementeerd en wordt momenteel gebruikt door gebundelde/native plugins, maar voor compatibiliteit met externe plugins geldt een strengere norm dan: "het is geëxporteerd, dus ligt het vast."

| Pluginsituatie                                  | Richtlijn                                                                                         |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Bestaande externe plugins                         | Houd integraties op basis van hooks werkend; dit is de compatibiliteitsbasis.                        |
| Nieuwe gebundelde/native plugins                        | Geef de voorkeur aan expliciete capaciteitsregistratie boven leverancierspecifieke interne toegang of nieuwe ontwerpen met alleen hooks. |
| Externe plugins die capaciteitsregistratie toepassen | Toegestaan, maar beschouw capaciteitsspecifieke hulpoppervlakken als in ontwikkeling, tenzij de documentatie ze als stabiel aanduidt. |

Capaciteitsregistratie is de beoogde richting. Verouderde hooks blijven tijdens de overgang voor externe plugins het veiligste pad zonder brekende wijzigingen. Niet alle geëxporteerde hulpsubpaden zijn gelijkwaardig — geef de voorkeur aan beperkte, gedocumenteerde contracten boven incidentele hulpexports.

### Pluginvormen

OpenClaw classificeert elke geladen plugin in een vorm op basis van het daadwerkelijke registratiegedrag (niet alleen statische metadata):

<AccordionGroup>
  <Accordion title="enkele capaciteit">
    Registreert precies één capaciteitstype (bijvoorbeeld een plugin met alleen een provider, zoals `arcee` of `chutes`).
  </Accordion>
  <Accordion title="hybride capaciteit">
    Registreert meerdere capaciteitstypen (bijvoorbeeld `openai` beheert tekstinferentie, spraak, mediabegrip en het genereren van afbeeldingen).
  </Accordion>
  <Accordion title="alleen hooks">
    Registreert alleen hooks (getypeerd of aangepast), zonder capaciteiten, tools, opdrachten of services.
  </Accordion>
  <Accordion title="zonder capaciteit">
    Registreert tools, opdrachten, services of routes, maar geen capaciteiten.
  </Accordion>
</AccordionGroup>

Gebruik `openclaw plugins inspect <id>` om de vorm en uitsplitsing van capaciteiten van een plugin te bekijken. Zie de [CLI-referentie](/nl/cli/plugins#inspect) voor details.

### Compatibiliteitssignalen

`openclaw doctor`, `openclaw plugins inspect <id>`, `openclaw status --all` en `openclaw plugins doctor` tonen deze compatibiliteitsmeldingen:

| Signaal                                     | Betekenis                                                                                                       |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| **configuratie geldig**                           | De configuratie wordt correct geparseerd en plugins worden gevonden                                                                        |
| **alleen hooks** (info)                       | De plugin registreert alleen hooks; dit is een ondersteund pad, maar nog niet gemigreerd naar capaciteitsregistratie                |
| **verouderde API voor geheugen-embeddings** (waarschuwing) | Een niet-gebundelde plugin gebruikt de oude geheugenspecifieke provider-API voor embeddings in plaats van `registerEmbeddingProvider` |
| **harde fout**                             | De configuratie is ongeldig of de plugin kon niet worden geladen                                                                    |

Geen van de advies- of waarschuwingssignalen maakt je plugin momenteel onbruikbaar. Deze signalen verschijnen ook in `openclaw status --all` en `openclaw plugins doctor`.

## Architectuuroverzicht

Het pluginsysteem van OpenClaw heeft vier lagen:

<Steps>
  <Step title="Manifest + detectie">
    OpenClaw vindt mogelijke plugins via geconfigureerde paden, werkruimteroots, globale pluginroots en gebundelde plugins. Detectie leest eerst native `openclaw.plugin.json`-manifesten en ondersteunde bundelmanifesten.
  </Step>
  <Step title="Inschakeling + validatie">
    De kern bepaalt of een gevonden plugin ingeschakeld, uitgeschakeld, geblokkeerd of geselecteerd is voor een exclusief slot, zoals geheugen.
  </Step>
  <Step title="Laden tijdens runtime">
    Native OpenClaw-plugins worden binnen het proces geladen en registreren capaciteiten in een centraal register. Verpakte JavaScript wordt geladen via native `require`; lokale TypeScript-broncode van derden gebruikt Jiti als noodoplossing. Compatibele bundels worden genormaliseerd tot registerrecords zonder runtimecode te importeren.
  </Step>
  <Step title="Gebruik van oppervlakken">
    De rest van OpenClaw leest het register om tools, kanalen, providerconfiguratie, hooks, HTTP-routes, CLI-opdrachten en services beschikbaar te maken.
  </Step>
</Steps>

Specifiek voor de plugin-CLI is de detectie van hoofdopdrachten opgesplitst in twee fasen:

- metadata tijdens het parseren komt uit `registerCli(..., { descriptors: [...] })`
- de echte CLI-module van de plugin kan lui geladen blijven en zich bij de eerste aanroep registreren

Zo blijft CLI-code die eigendom is van de plugin binnen de plugin, terwijl OpenClaw toch namen van hoofdopdrachten kan reserveren voordat het parseren begint.

De belangrijke ontwerpgrens:

- validatie van manifesten/configuratie moet werken op basis van **manifest-/schemametadata** zonder plugincode uit te voeren
- native capaciteitsdetectie mag vertrouwde toegangscode van plugins laden om een niet-activerende momentopname van het register te bouwen
- native runtimegedrag komt uit het `register(api)`-pad van de pluginmodule met `api.registrationMode === "full"`

Door deze scheiding kan OpenClaw configuraties valideren, ontbrekende/uitgeschakelde plugins toelichten en UI-/schemahints opbouwen voordat de volledige runtime actief is.

### Momentopname van pluginmetadata en opzoektabel

Bij het opstarten bouwt de Gateway één `PluginMetadataSnapshot` voor de huidige configuratiemomentopname. De momentopname bevat alleen metadata: de index met geïnstalleerde plugins, het manifestregister, manifestdiagnostiek, eigenaarskaarten, een normalisator voor plugin-id's en manifestrecords. De momentopname bevat geen geladen pluginmodules, provider-SDK's, pakketinhoud of runtime-exports.

Configuratievalidatie met kennis van plugins, automatisch inschakelen bij het opstarten en de pluginbootstrap van de Gateway gebruiken die momentopname in plaats van manifest-/indexmetadata onafhankelijk opnieuw op te bouwen. `PluginLookUpTable` wordt afgeleid van dezelfde momentopname en voegt het pluginplan voor het opstarten toe voor de huidige runtimeconfiguratie.

Na het opstarten bewaart de Gateway de huidige metadatamomentopname als een vervangbaar runtimeproduct. Herhaalde detectie van runtimeproviders kan die momentopname gebruiken in plaats van bij elke providercatalogusdoorloop de index met geïnstalleerde plugins en het manifestregister opnieuw op te bouwen. De momentopname wordt gewist of vervangen bij het afsluiten van de Gateway, wijzigingen in de configuratie/plugininventaris en schrijfbewerkingen aan de geïnstalleerde index; aanroepers vallen terug op het koude manifest-/indexpad wanneer er geen compatibele actuele momentopname bestaat. Compatibiliteitscontroles moeten plugindetectieroots zoals `plugins.load.paths` en de standaardwerkruimte van de agent omvatten, omdat werkruimteplugins deel uitmaken van het metadatabereik.

De momentopname en opzoektabel houden herhaalde opstartbeslissingen op het snelle pad:

- kanaaleigendom
- uitgestelde kanaalstart
- plugin-id's voor het opstarten
- eigendom van providers en CLI-backends
- eigendom van installatieproviders, opdrachtaliassen, modelcatalogusproviders en manifestcontracten
- validatie van pluginconfiguratieschema's en kanaalconfiguratieschema's
- beslissingen over automatisch inschakelen bij het opstarten

De veiligheidsgrens is het vervangen van de momentopname, niet het wijzigen ervan. Bouw de momentopname opnieuw op wanneer de configuratie, plugininventaris, installatierecords of het persistente indexbeleid wijzigen. Behandel deze niet als een breed, wijzigbaar globaal register en bewaar geen onbeperkt aantal historische momentopnamen. Het laden van plugins tijdens runtime blijft gescheiden van metadatamomentopnamen, zodat verouderde runtimestatus niet achter een metadatacache kan worden verborgen.

De cacheregel is gedocumenteerd in [Interne architectuur van plugins](/nl/plugins/architecture-internals#plugin-cache-boundary): manifest- en detectiemetadata zijn actueel, tenzij een aanroeper voor de huidige stroom een expliciete momentopname, opzoektabel of manifestregister vasthoudt. Verborgen metadatacaches en TTL's op basis van de klok maken geen deel uit van het laden van plugins. Alleen caches voor de runtimelader, modules en afhankelijkheidsartefacten mogen blijven bestaan nadat code of geïnstalleerde artefacten daadwerkelijk zijn geladen.

Sommige aanroepers op koude paden reconstrueren manifestregisters nog steeds rechtstreeks uit de persistente index van geïnstalleerde plugins, in plaats van een Gateway `PluginLookUpTable` te ontvangen. Dat pad reconstrueert het register nu op aanvraag; geef bij voorkeur de huidige opzoektabel of een expliciet manifestregister door via runtimeflows wanneer een aanroeper er al een heeft.

### Activeringsplanning

Activeringsplanning maakt deel uit van het besturingsvlak. Aanroepers kunnen vóór het laden van bredere runtimeregisters opvragen welke plugins relevant zijn voor een concrete opdracht, provider, kanaal, route, agentharnas of capability.

De planner houdt het huidige manifestgedrag compatibel:

- `activation.*`-velden zijn expliciete plannerhints
- `providers`, `channels`, `commandAliases`, `setup.providers`, `contracts.tools` en hooks blijven de terugvaloptie voor manifesteigenaarschap
- de planner-API die alleen id's gebruikt, blijft beschikbaar voor bestaande aanroepers
- de plan-API rapporteert redenlabels, zodat diagnostiek expliciete hints kan onderscheiden van terugval op eigenaarschap

<Warning>
Beschouw `activation` niet als een levenscyclushook of als vervanging voor `register(...)`. Het zijn metadata die worden gebruikt om het laden te beperken. Geef de voorkeur aan eigenaarschapsvelden wanneer die de relatie al beschrijven; gebruik `activation` alleen voor aanvullende plannerhints.
</Warning>

### Kanaalplugins en de gedeelde berichttool

Kanaalplugins hoeven voor normale chatacties geen afzonderlijke tool voor verzenden, bewerken of reageren te registreren. OpenClaw houdt één gedeelde `message`-tool in de core en kanaalplugins beheren daarachter de kanaalspecifieke detectie en uitvoering.

De huidige grens is:

- de core beheert de host van de gedeelde `message`-tool, promptbedrading, sessie-/threadadministratie en uitvoeringsdispatch
- kanaalplugins beheren detectie van acties binnen het bereik, detectie van capabilities en eventuele kanaalspecifieke schemafragmenten
- kanaalplugins beheren de providerspecifieke grammatica voor sessiegesprekken, zoals hoe gespreks-id's thread-id's coderen of van bovenliggende gesprekken worden overgenomen
- kanaalplugins voeren de uiteindelijke actie uit via hun actieadapter

Voor kanaalplugins is het SDK-oppervlak `ChannelMessageActionAdapter.describeMessageTool(...)`. Met die uniforme detectieaanroep kan een plugin zijn zichtbare acties, capabilities en schemabijdragen samen retourneren, zodat deze onderdelen niet uit elkaar gaan lopen.

Namen van berichtacties gebruiken bewust een gesloten vocabulaire dat door de core wordt beheerd, zodat elk transport elke actie kan weergeven. Plugins voegen actienamen toe via een PR voor de core; runtimeregistratie wordt bewust niet ondersteund.

Wanneer een kanaalspecifieke parameter van de berichttool een mediabron bevat, zoals een lokaal pad of externe media-URL, moet de plugin ook `mediaSourceParams` retourneren vanuit `describeMessageTool(...)`. De core gebruikt die expliciete lijst om normalisatie van sandboxpaden en hints voor toegang tot uitgaande media toe te passen zonder parameternamen die eigendom zijn van plugins hard te coderen. Gebruik daar bij voorkeur kaarten per actie en niet één vlakke lijst voor het hele kanaal, zodat een mediaparameter die alleen voor profielen geldt niet wordt genormaliseerd bij niet-gerelateerde acties zoals `send`.

De core geeft het runtimebereik door aan die detectiestap. Belangrijke velden zijn onder meer:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- vertrouwde inkomende `requesterSenderId`

Dat is belangrijk voor contextgevoelige plugins. Een kanaal kan berichtacties verbergen of beschikbaar maken op basis van het actieve account, de huidige ruimte/thread/het huidige bericht of de identiteit van de vertrouwde aanvrager, zonder kanaalspecifieke vertakkingen hard te coderen in de `message`-tool van de core.

Daarom blijven routeringswijzigingen voor ingebedde runners pluginwerk: de runner is verantwoordelijk voor het doorsturen van de huidige chat-/sessie-identiteit naar de detectiegrens van de plugin, zodat de gedeelde `message`-tool voor de huidige beurt het juiste kanaalspecifieke oppervlak beschikbaar maakt.

Voor uitvoeringshelpers die door kanalen worden beheerd, moeten kanaalplugins de uitvoeringsruntime binnen hun eigen pluginmodules houden. De core beheert niet langer de runtimes voor berichtacties van Discord, Slack, Telegram of WhatsApp onder `src/agents/tools`. We publiceren geen afzonderlijke `plugin-sdk/*-action-runtime`-subpaden en die plugins moeten hun eigen lokale runtimecode rechtstreeks uit hun eigen pluginmodules importeren.

Dezelfde grens geldt in het algemeen voor SDK-koppelingen met providernamen: de core mag geen kanaalspecifieke gemaksbarrels importeren voor Discord, Signal, Slack, WhatsApp of vergelijkbare plugins. Als de core gedrag nodig heeft, gebruikt deze de eigen `api.ts`- / `runtime-api.ts`-barrel van de gebundelde plugin of wordt de behoefte omgezet in een beperkte generieke capability in de gedeelde SDK.

Voor gebundelde plugins geldt dezelfde regel. De `runtime-api.ts` van een gebundelde plugin mag zijn eigen merkgebonden `openclaw/plugin-sdk/<plugin-id>`-facade niet opnieuw exporteren. Die merkgebonden facades blijven compatibiliteitsshimlagen voor externe plugins en oudere gebruikers, maar gebundelde plugins moeten lokale exports gebruiken plus beperkte generieke SDK-subpaden zoals `openclaw/plugin-sdk/channel-policy`, `openclaw/plugin-sdk/runtime-store` of `openclaw/plugin-sdk/webhook-ingress`. Nieuwe code mag geen SDK-facades toevoegen die specifiek zijn voor een plugin-id, tenzij de compatibiliteitsgrens voor een bestaand extern ecosysteem dit vereist.

Specifiek voor peilingen zijn er twee uitvoeringspaden:

- `outbound.sendPoll` is de gedeelde basis voor kanalen die binnen het algemene peilingsmodel passen
- `actions.handleAction("poll")` is het voorkeurspad voor kanaalspecifieke peilingssemantiek of aanvullende peilingsparameters

De core stelt het parseren van gedeelde peilingen nu uit totdat de peilingsdispatch van de plugin de actie afwijst, zodat peilingshandlers van plugins kanaalspecifieke peilingsvelden kunnen accepteren zonder eerst door de generieke peilingsparser te worden geblokkeerd.

Zie [Interne pluginarchitectuur](/nl/plugins/architecture-internals) voor de volledige opstartvolgorde.

## Eigenaarschapsmodel voor capabilities

OpenClaw behandelt een native plugin als de eigendomsgrens voor een **bedrijf** of een **functie**, niet als een verzameling niet-gerelateerde integraties.

Dat betekent:

- een bedrijfsplugin moet doorgaans alle op OpenClaw gerichte oppervlakken van dat bedrijf beheren
- een functieplugin moet doorgaans het volledige functieoppervlak beheren dat deze introduceert
- kanalen moeten gedeelde capabilities van de core gebruiken in plaats van providergedrag ad hoc opnieuw te implementeren

<AccordionGroup>
  <Accordion title="Leverancier met meerdere capabilities">
    `google` beheert tekstinferentie, CLI-backend, embeddings, spraak, realtime spraak, mediabegrip, generatie van afbeeldingen/muziek/video en zoeken op het web. `openai` beheert tekstinferentie, embeddings, spraak, realtime transcriptie, realtime spraak, mediabegrip en generatie van afbeeldingen/video. `minimax` beheert tekstinferentie plus mediabegrip, spraak, generatie van afbeeldingen/muziek/video en zoeken op het web.
  </Accordion>
  <Accordion title="Leverancier met één capability">
    `arcee` en `chutes` beheren alleen tekstinferentie; `microsoft` beheert alleen spraak. Een leveranciersplugin kan zo beperkt blijven totdat deze een groter deel van het oppervlak van die leverancier moet afdekken.
  </Accordion>
  <Accordion title="Functieplugin">
    `voice-call` beheert gesprekstransport, tools, CLI, routes en overbrugging van Twilio-mediastreams, maar gebruikt gedeelde capabilities voor spraak, realtime transcriptie en realtime spraak in plaats van leveranciersplugins rechtstreeks te importeren.
  </Accordion>
</AccordionGroup>

De beoogde eindtoestand is:

- het op OpenClaw gerichte oppervlak van een leverancier bevindt zich in één plugin, zelfs als dit tekstmodellen, spraak, afbeeldingen en video omvat
- andere leveranciers kunnen hetzelfde doen voor hun eigen oppervlak
- kanalen hoeven niet te weten welke leveranciersplugin de provider beheert; ze gebruiken het gedeelde capabilitycontract dat door de core beschikbaar wordt gesteld

Dit is het belangrijkste onderscheid:

- **plugin** = eigendomsgrens
- **capability** = corecontract dat meerdere plugins kunnen implementeren of gebruiken

Als OpenClaw dus een nieuw domein toevoegt, zoals video, is de eerste vraag niet: "welke provider moet videoverwerking hard coderen?" De eerste vraag is: "wat is het corecontract voor de videocapability?" Zodra dat contract bestaat, kunnen leveranciersplugins zich ervoor registreren en kunnen kanaal-/functieplugins het gebruiken.

Als de capability nog niet bestaat, is de juiste aanpak doorgaans:

<Steps>
  <Step title="Definieer de capability">
    Definieer de ontbrekende capability in de core.
  </Step>
  <Step title="Stel deze beschikbaar via de SDK">
    Stel deze op een getypeerde manier beschikbaar via de plugin-API/runtime.
  </Step>
  <Step title="Koppel gebruikers">
    Koppel kanalen/functies aan die capability.
  </Step>
  <Step title="Leveranciersimplementaties">
    Laat leveranciersplugins implementaties registreren.
  </Step>
</Steps>

Zo blijft eigenaarschap expliciet en wordt tegelijk voorkomen dat coregedrag afhankelijk is van één leverancier of een eenmalig pluginspecifiek codepad.

### Gelaagdheid van capabilities

Gebruik dit mentale model bij het bepalen waar code thuishoort:

<Tabs>
  <Tab title="Corelaag voor capabilities">
    Gedeelde orkestratie, beleid, terugvalopties, regels voor het samenvoegen van configuratie, leveringssemantiek en getypeerde contracten.
  </Tab>
  <Tab title="Laag voor leveranciersplugins">
    Leveranciersspecifieke API's, authenticatie, modelcatalogi, spraaksynthese, afbeeldingsgeneratie, videobackends en gebruikseindpunten.
  </Tab>
  <Tab title="Laag voor kanaal-/functieplugins">
    Integratie voor Discord/Slack/spraakgesprekken/enzovoort die corecapabilities gebruikt en deze via een oppervlak presenteert.
  </Tab>
</Tabs>

TTS volgt bijvoorbeeld deze structuur:

- de core beheert het TTS-beleid tijdens antwoorden, de terugvalvolgorde, voorkeuren en levering via kanalen
- `elevenlabs`, `google`, `microsoft` en `openai` beheren synthese-implementaties
- `voice-call` gebruikt de runtimehelper voor telefonie-TTS

Hetzelfde patroon verdient de voorkeur voor toekomstige capabilities.

### Voorbeeld van een bedrijfsplugin met meerdere capabilities

Een bedrijfsplugin moet van buitenaf samenhangend aanvoelen. Als OpenClaw gedeelde contracten heeft voor modellen, spraak, realtime transcriptie, realtime spraak, mediabegrip, afbeeldingsgeneratie, videogeneratie, webophaling en zoeken op het web, kan een leverancier al zijn oppervlakken op één plaats beheren:

```ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { exampleAiMedia } from "./exampleai-media.js";

export default definePluginEntry({
  id: "exampleai",
  name: "ExampleAI",
  description: "ExampleAI-modellen en mediacapabilities.",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // hooks voor authenticatie/modelcatalogus/runtime
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // spraakconfiguratie van de leverancier — implementeer de SpeechProviderPlugin-interface rechtstreeks
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      describeImage: (req) => exampleAiMedia.describeImage(req),
      transcribeAudio: (req) => exampleAiMedia.transcribeAudio(req),
      describeVideo: (req) => exampleAiMedia.describeVideo(req),
    });

    api.registerWebSearchProvider({
      id: "exampleai-search",
      createTool() {
        // Retourneer de webzoektool die door de leverancier wordt beheerd.
      },
    });
  },
});
```

Niet de exacte namen van de helpers zijn belangrijk. De structuur is belangrijk:

- één plugin beheert het leveranciersoppervlak
- de core blijft de capabilitycontracten beheren
- vertaling van providerverzoeken en HTTP-helpers blijven in de leveranciersplugin
- kanalen en functieplugins gebruiken `api.runtime.*`-helpers, geen leverancierscode
- contracttests kunnen controleren of de plugin de capabilities heeft geregistreerd waarvan deze beweert eigenaar te zijn

### Capabilityvoorbeeld: videobegrip

OpenClaw behandelt begrip van afbeeldingen/audio/video al als één gedeelde capability. Daar geldt hetzelfde eigenaarschapsmodel:

<Steps>
  <Step title="Core definieert het contract">
    Core definieert het contract voor mediabegrip.
  </Step>
  <Step title="Leveranciersplugins registreren zich">
    Leveranciersplugins registreren `describeImage`, `transcribeAudio` en `describeVideo` waar van toepassing.
  </Step>
  <Step title="Consumenten gebruiken het gedeelde gedrag">
    Kanalen en functieplugins gebruiken het gedeelde gedrag van core in plaats van rechtstreeks verbinding te maken met leverancierscode.
  </Step>
</Steps>

Zo worden de videoaannames van één provider niet in core verankerd. De Plugin beheert het leveranciersoppervlak; core beheert het capaciteitscontract en het fallbackgedrag.

Videogeneratie gebruikt diezelfde volgorde al: core beheert het getypeerde capaciteitscontract en de runtimehelper, en leveranciersplugins registreren daarvoor implementaties van `api.registerVideoGenerationProvider(...)`.

Een concrete checklist voor de uitrol nodig? Zie [Capaciteitenhandboek](/nl/plugins/adding-capabilities).

## Contracten en handhaving

Het API-oppervlak voor plugins is bewust getypeerd en gecentraliseerd in `OpenClawPluginApi`. Dat contract definieert de ondersteunde registratiepunten en de runtimehelpers waarop een Plugin mag vertrouwen.

Waarom dit belangrijk is:

- auteurs van plugins krijgen één stabiele interne standaard
- core kan dubbel eigenaarschap afwijzen, zoals twee plugins die dezelfde provider-id registreren
- bij het opstarten kunnen bruikbare diagnostische gegevens voor ongeldige registraties worden weergegeven
- contracttests kunnen het eigenaarschap van gebundelde plugins afdwingen en ongemerkte afwijkingen voorkomen

Er zijn twee handhavingslagen:

<AccordionGroup>
  <Accordion title="Handhaving van runtimeregistratie">
    Het pluginregister valideert registraties terwijl plugins worden geladen. Voorbeelden: dubbele provider-id's, dubbele id's van spraakproviders en ongeldige registraties leveren diagnostische gegevens voor plugins op in plaats van ongedefinieerd gedrag.
  </Accordion>
  <Accordion title="Contracttests">
    Gebundelde plugins worden tijdens testruns vastgelegd in contractregisters, zodat OpenClaw het eigenaarschap expliciet kan controleren. Dit wordt momenteel gebruikt voor modelproviders, spraakproviders, providers voor zoeken op het web en het eigenaarschap van gebundelde registraties.
  </Accordion>
</AccordionGroup>

Het praktische effect is dat OpenClaw vooraf weet welke Plugin welk oppervlak beheert. Daardoor kunnen core en kanalen naadloos samenwerken, omdat het eigenaarschap wordt gedeclareerd, getypeerd en getest in plaats van impliciet te blijven.

### Wat in een contract thuishoort

<Tabs>
  <Tab title="Goede contracten">
    - getypeerd
    - klein
    - capaciteitsspecifiek
    - beheerd door core
    - herbruikbaar door meerdere plugins
    - bruikbaar voor kanalen/functies zonder kennis van de leverancier

  </Tab>
  <Tab title="Slechte contracten">
    - leveranciersspecifiek beleid dat in core verborgen zit
    - eenmalige ontsnappingsroutes voor plugins die het register omzeilen
    - kanaalcode die rechtstreeks toegang zoekt tot een leveranciersimplementatie
    - ad-hoc-runtimeobjecten die geen deel uitmaken van `OpenClawPluginApi` of `api.runtime`

  </Tab>
</Tabs>

Kies bij twijfel een hoger abstractieniveau: definieer eerst de capaciteit en laat plugins daarop aansluiten.

## Uitvoeringsmodel

Systeemeigen OpenClaw-plugins worden **in hetzelfde proces** als de Gateway uitgevoerd. Ze draaien niet in een sandbox. Een geladen systeemeigen Plugin heeft dezelfde vertrouwensgrens op procesniveau als corecode.

<Warning>
Gevolgen van systeemeigen plugins: een Plugin kan tools, netwerkhandlers, hooks en services registreren; een fout in een Plugin kan de Gateway laten crashen of destabiliseren; en een schadelijke systeemeigen Plugin staat gelijk aan de uitvoering van willekeurige code binnen het OpenClaw-proces.
</Warning>

Compatibele bundels zijn standaard veiliger, omdat OpenClaw ze momenteel als metadata-/inhoudspakketten behandelt. In de huidige releases betekent dit voornamelijk gebundelde Skills.

Gebruik toelatingslijsten en expliciete installatie-/laadpaden voor niet-gebundelde plugins. Behandel werkruimteplugins als code voor ontwikkeling, niet als standaardinstellingen voor productie.

Houd voor gebundelde werkruimtepakketnamen de plugin-id standaard verankerd in de npm-naam: `@openclaw/<id>`, of gebruik een goedgekeurd getypeerd achtervoegsel zoals `-provider`, `-plugin`, `-speech`, `-sandbox` of `-media-understanding` wanneer het pakket bewust een beperktere pluginrol beschikbaar stelt.

<Note>
**Opmerking over vertrouwen:** `plugins.allow` vertrouwt **plugin-id's**, niet de herkomst van de bron. Een werkruimteplugin met dezelfde id als een gebundelde Plugin overschaduwt bewust het gebundelde exemplaar wanneer die werkruimteplugin is ingeschakeld/op de toelatingslijst staat. Dit is normaal en nuttig voor lokale ontwikkeling, het testen van patches en hotfixes. Het vertrouwen in gebundelde plugins wordt bepaald aan de hand van de bronmomentopname — het manifest en de code op schijf tijdens het laden — en niet aan de hand van installatiemetadata. Een beschadigd of vervangen installatierecord kan het vertrouwensoppervlak van een gebundelde Plugin niet ongemerkt uitbreiden tot buiten wat de daadwerkelijke bron claimt.
</Note>

## Exportgrens

OpenClaw exporteert capaciteiten, geen implementatiegemak.

Houd capaciteitsregistratie openbaar. Beperk exports van helpers die niet tot het contract behoren:

- helpersubpaden die specifiek zijn voor gebundelde plugins
- subpaden voor runtime-infrastructuur die niet als openbare API zijn bedoeld
- leveranciersspecifieke gemakhelpers
- helpers voor installatie/onboarding die implementatiedetails zijn

Gereserveerde helpersubpaden voor gebundelde plugins zijn uit de gegenereerde SDK-exporttoewijzing verwijderd. Houd eigenaarspecifieke helpers binnen het pakket van de beherende Plugin; promoveer alleen herbruikbaar hostgedrag naar generieke SDK-contracten zoals `plugin-sdk/gateway-runtime`, `plugin-sdk/security-runtime` en geïnjecteerde capaciteiten van de plugin-API.

## Interne werking en naslaginformatie

Zie [Interne werking van de pluginarchitectuur](/nl/plugins/architecture-internals) voor de laadpijplijn, het registermodel, runtimehooks voor providers, HTTP-routes van de Gateway, schema's voor berichttools, de oplossing van kanaaldoelen, providercatalogi, contextengineplugins en de handleiding voor het toevoegen van een nieuwe capaciteit.

## Gerelateerd

- [Plugins bouwen](/nl/plugins/building-plugins)
- [Pluginmanifest](/nl/plugins/manifest)
- [Plugin-SDK instellen](/nl/plugins/sdk-setup)
