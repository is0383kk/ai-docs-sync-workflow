---
read_when:
    - Provider-runtimehooks, kanaallevenscyclus of pakketbundels implementeren
    - Plugin-laadvolgorde of registerstatus debuggen
    - Een nieuwe Plugin-mogelijkheid of contextengine-Plugin toevoegen
summary: 'Interne details van de Plugin-architectuur: laadpijplijn, register, runtime-hooks, HTTP-routes en referentietabellen'
title: Interne werking van de Plugin-architectuur
x-i18n:
    generated_at: "2026-07-27T06:23:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 278ac23a9454ab69407c59fa197e75756fa0dc5880fcae6c3eecc15bd4733a09
    source_path: plugins/architecture-internals.md
    workflow: 16
---

Voor het openbare capaciteitsmodel, pluginvormen en eigendoms-/uitvoeringscontracten, zie [Pluginarchitectuur](/nl/plugins/architecture). Deze pagina behandelt de interne werking: laadpijplijn, register, runtimehooks, HTTP-routes van de Gateway, importpaden en schematabellen.

## Laadpijplijn

Bij het opstarten doet OpenClaw grofweg het volgende:

1. kandidaat-pluginroots ontdekken
2. native of compatibele bundelmanifesten en pakketmetadata lezen
3. onveilige kandidaten weigeren
4. pluginconfiguratie normaliseren (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. voor elke kandidaat bepalen of deze wordt ingeschakeld
6. ingeschakelde native modules laden: gebouwde gebundelde modules gebruiken een native loader;
   lokale TypeScript-broncode van derden gebruikt de Jiti-noodfallback
7. native `register(api)`-hooks aanroepen en registraties in het pluginregister verzamelen
8. het register beschikbaar stellen aan opdrachten/runtime-oppervlakken

Veiligheidscontroles worden **vóór** uitvoering tijdens runtime uitgevoerd. Discovery blokkeert een kandidaat wanneer:

- het herleide toegangspunt buiten de pluginroot valt
- het pad (of de rootmap ervan) door iedereen beschrijfbaar is
- voor niet-gebundelde plugins het eigendom van het pad niet overeenkomt met de huidige uid (of root)

Bij door iedereen beschrijfbare gebundelde mappen wordt eerst ter plaatse een `chmod`-herstelpoging uitgevoerd
(npm-/globale installaties kunnen pakketmappen met `0777` leveren), voordat de controle
opnieuw wordt uitgevoerd; eigendomscontroles worden voor de gebundelde herkomst volledig overgeslagen.

Geblokkeerde kandidaten behouden in de uitgegeven diagnose nog steeds hun plugin-id wanneer
deze bekend is (inclusief id's die zijn herleid uit een manifest in een
verder geweigerde map), zodat configuratie die naar die id verwijst een geblokkeerde
plugin ziet die is gekoppeld aan een waarschuwing over padveiligheid, in plaats van een niet-gerelateerde fout
"onbekende plugin".

### Manifest-eerst-gedrag

Het manifest is de gezaghebbende bron voor het besturingsvlak. OpenClaw gebruikt het om:

- de plugin te identificeren
- gedeclareerde kanalen/Skills/configuratieschema's of bundelcapaciteiten te ontdekken
- `plugins.entries.<id>.config` te valideren
- labels/plaatsaanduidingen van de Control UI aan te vullen
- installatie-/catalogusmetadata weer te geven
- goedkope activerings- en instellingsbeschrijvingen te behouden zonder de pluginruntime te laden

Voor native plugins is de runtimemodule het gegevensvlakgedeelte. Deze registreert
daadwerkelijk gedrag zoals hooks, tools, opdrachten of providerflows.

Optionele manifestblokken `activation` en `setup` blijven op het besturingsvlak.
Het zijn uitsluitend metadatabeschrijvingen voor activeringsplanning en het ontdekken van instellingen;
ze vervangen runtimeregistratie, `register(...)` of `setupEntry` niet.
Actieve activeringsconsumenten gebruiken aanwijzingen voor opdrachten, kanalen en providers uit het manifest om
het laden van plugins te beperken vóór bredere materialisatie van het register:

- laden via de CLI wordt beperkt tot plugins die eigenaar zijn van de opgevraagde primaire opdracht
- kanaalinstelling/pluginresolutie wordt beperkt tot plugins die eigenaar zijn van de opgevraagde
  kanaal-id
- expliciete providerinstelling/runtime-resolutie wordt beperkt tot plugins die eigenaar zijn van de
  opgevraagde provider-id
- de opstartplanning van de Gateway gebruikt `activation.onStartup` voor expliciete opstartimports;
  plugins zonder opstartmetadata worden alleen via beperktere
  activeringstriggers geladen

De activeringsplanner biedt zowel een API met alleen id's voor bestaande aanroepers als een
plan-API voor diagnostiek. Planvermeldingen melden waarom een plugin is geselecteerd,
waarbij expliciete `activation.*`-aanwijzingen worden onderscheiden van de fallback op manifesteigendom:

| Reden (uit `activation.*`-aanwijzingen)   | Reden (uit manifesteigendom)                                                             |
| ------------------------------------ | -------------------------------------------------------------------------------------------- |
| `activation-agent-harness-hint`      | —                                                                                            |
| `activation-capability-hint`         | —                                                                                            |
| `activation-channel-hint`            | `manifest-channel-owner` (`channels`)                                                        |
| `activation-command-hint`            | `manifest-command-alias` (`commandAliases`)                                                  |
| `activation-provider-hint`           | `manifest-provider-owner` (`providers`), `manifest-setup-provider-owner` (`setup.providers`) |
| `activation-route-hint`              | —                                                                                            |
| — (hooktrigger heeft geen aanwijzingsvariant) | `manifest-hook-owner` (`hooks`), `manifest-tool-contract` (`contracts.tools`)                |

Die splitsing van redenen is de compatibiliteitsgrens: bestaande pluginmetadata
blijft werken, terwijl nieuwe code brede aanwijzingen of fallbackgedrag kan detecteren
zonder de laadsemantiek tijdens runtime te wijzigen.

Runtime-preloads tijdens aanvragen die om het brede `all`-bereik vragen, leiden nog steeds
een expliciete effectieve verzameling plugin-id's af uit configuratie, opstartplanning, geconfigureerde
kanalen, slots en regels voor automatisch inschakelen
(`resolveEffectivePluginIds` in `src/plugins/effective-plugin-ids.ts`). Als die
afgeleide verzameling leeg is, houdt OpenClaw het bereik leeg in plaats van dit uit te breiden naar
elke vindbare plugin.

Het ontdekken van instellingen geeft de voorkeur aan id's waarvan de descriptor eigenaar is, zoals `setup.providers` en
`setup.cliBackends`, om kandidaat-plugins te beperken voordat wordt teruggevallen op
`setup-api` voor plugins die nog runtimehooks tijdens de instelling nodig hebben. Lijsten voor
providerinstellingen gebruiken manifest-`providerAuthChoices`, uit descriptors afgeleide instellingskeuzes
en metadata uit de installatiecatalogus zonder de providerruntime te laden. Expliciete
`setup.requiresRuntime: false` is een grenswaarde die uitsluitend voor descriptors geldt; een weggelaten
`requiresRuntime` behoudt voor compatibiliteit de fallback op de verouderde instellings-API. Als
meer dan één ontdekte plugin aanspraak maakt op dezelfde genormaliseerde instellingsprovider- of
CLI-backend-id, weigert de instellingszoekactie de ambigue eigenaar in plaats van te vertrouwen op
de ontdekkingsvolgorde. Wanneer de instellingsruntime wel wordt uitgevoerd, melden registerdiagnoses
afwijkingen tussen `setup.providers` / `setup.cliBackends` en de providers of CLI-
backends die daadwerkelijk door de instellings-API zijn geregistreerd, zonder verouderde plugins te blokkeren.

### Cachegrens van plugins

OpenClaw cachet resultaten van het ontdekken van plugins of rechtstreekse gegevens uit het manifestregister
niet achter tijdvensters op basis van de systeemtijd. Installaties, manifestbewerkingen en wijzigingen in laadpaden
moeten zichtbaar worden bij de volgende expliciete metadata-uitlezing of herbouw van een momentopname.
De parser voor manifestbestanden houdt een begrensde cache met bestandshandtekeningen bij, gesleuteld op het
geopende manifestpad plus apparaat/inode, grootte en mtime/ctime; die cache voorkomt alleen
dat ongewijzigde bytes opnieuw worden geparseerd en mag geen antwoorden over discovery, het register,
eigendom of beleid cachen.

Het veilige snelle pad voor metadata is expliciet objecteigendom, geen verborgen cache.
Hot paths bij het opstarten van de Gateway moeten de huidige `PluginMetadataSnapshot`,
de afgeleide `PluginLookUpTable` of een expliciet manifestregister door de
aanroepketen doorgeven. Configuratievalidatie, automatisch inschakelen bij het opstarten, pluginbootstrap en providerselectie
kunnen die objecten hergebruiken zolang ze de huidige configuratie en
plugininventaris vertegenwoordigen. Bij het opzoeken van instellingen worden manifestmetadata nog steeds op aanvraag
opnieuw opgebouwd, tenzij het specifieke instellingspad een expliciet manifestregister ontvangt; behoud
dit als fallback voor een koud pad in plaats van verborgen opzoekcaches toe te voegen. Wanneer de
invoer verandert, bouw je de momentopname opnieuw op en vervang je deze, in plaats van deze te wijzigen of
historische kopieën te bewaren. Weergaven van het actieve pluginregister en gebundelde
helpers voor kanaalbootstrap moeten opnieuw worden berekend vanuit het huidige
register/de huidige root. Kortlevende maps zijn binnen één aanroep prima om werk te dedupliceren of
herintreding te bewaken; ze mogen geen metadata-caches op procesniveau worden.

Voor het laden van plugins is de persistente cachelaag het laden tijdens runtime. Deze mag
loaderstatus hergebruiken wanneer code of geïnstalleerde artefacten daadwerkelijk worden geladen, zoals:

- `PluginLoaderCacheState` en compatibele actieve runtimeregisters
- Jiti-/modulecaches en loadercaches voor openbare oppervlakken die worden gebruikt om te voorkomen dat
  hetzelfde runtime-oppervlak herhaaldelijk wordt geïmporteerd
- bestandssysteemcaches voor geïnstalleerde pluginartefacten
- kortlevende maps per aanroep voor padnormalisatie of het oplossen van duplicaten

Die caches zijn implementatiedetails van het gegevensvlak. Ze mogen geen
vragen van het besturingsvlak beantwoorden, zoals "welke plugin is eigenaar van deze provider?", tenzij de
aanroeper bewust om laden tijdens runtime heeft gevraagd.

Voeg geen persistente of tijdgebonden caches toe voor:

- discoveryresultaten
- rechtstreekse manifestregisters
- manifestregisters die opnieuw zijn opgebouwd vanuit de index van geïnstalleerde plugins
- het opzoeken van providereigenaars, modelonderdrukking, providerbeleid of metadata van openbare artefacten
- elk ander van het manifest afgeleid antwoord waarbij een gewijzigd manifest, een gewijzigde geïnstalleerde index
  of een gewijzigd laadpad zichtbaar moet zijn bij de volgende metadata-uitlezing

Aanroepers die manifestmetadata opnieuw opbouwen vanuit de persistente index van geïnstalleerde plugins,
bouwen dat register op aanvraag opnieuw op. De geïnstalleerde index is duurzame status van het bronvlak;
het is geen verborgen metadatacache in het proces.

## Registermodel

Geladen plugins wijzigen niet rechtstreeks willekeurige globale variabelen van de core. Ze registreren zich in een
centraal pluginregister (`PluginRegistry` in `src/plugins/registry-types.ts`),
dat pluginrecords bijhoudt (identiteit, bron, herkomst, status, diagnostiek)
plus arrays voor elke capaciteit: tools, verouderde hooks en getypeerde hooks,
kanalen, providers, RPC-handlers van de Gateway, HTTP-routes, CLI-registrators,
achtergrondservices, opdrachten waarvan plugins eigenaar zijn en tientallen andere getypeerde providerfamilies
(spraak, embeddings, beeld-/video-/muziekgeneratie, web-
ophalen/zoeken, agentharnassen, sessieacties enzovoort).

Corefuncties lezen vervolgens uit dat register in plaats van rechtstreeks met pluginmodules
te communiceren. Hierdoor blijft het laden eenrichtingsverkeer:

- pluginmodule -> registerregistratie
- core-runtime -> registergebruik

Die scheiding is belangrijk voor de onderhoudbaarheid. Daardoor hebben de meeste core-oppervlakken slechts
één integratiepunt nodig: "het register lezen", niet "voor elke
pluginmodule een speciaal geval toevoegen".

## Callbacks voor gespreksbinding

Plugins die een gesprek binden, kunnen reageren wanneer een goedkeuring is afgehandeld.

Gebruik `api.onConversationBindingResolved(...)` om een callback te ontvangen nadat een bindingsverzoek
is goedgekeurd of afgewezen:

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // Er bestaat nu een binding voor deze plugin + dit gesprek.
        console.log(event.binding?.conversationId);
        return;
      }

      // Het verzoek is afgewezen; wis eventuele lokale status in afwachting.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

Velden van de callbackpayload:

- `status`: `"approved"` of `"denied"`
- `decision`: `"allow-once"`, `"allow-always"` of `"deny"`
- `binding`: de afgehandelde binding voor goedgekeurde verzoeken
- `request`: de oorspronkelijke samenvatting van het verzoek, ontkoppelingshint, afzender-id en
  gespreksmetadata

Deze callback dient alleen voor meldingen. Deze verandert niet wie een
gesprek mag binden en wordt uitgevoerd nadat de verwerking van de goedkeuring door de core is voltooid.

## Runtimehooks voor providers

Providerplugins hebben drie lagen:

- **Manifestmetadata** voor snel opzoeken vóór de runtime:
  `setup.providers[].envVars`, `providerAuthAliases`, `providerAuthChoices`
  en `channelConfigs`.
- **Hooks tijdens configuratie**: `catalog` plus `applyConfigDefaults`.
- **Runtimehooks**: meer dan 40 optionele hooks voor authenticatie, modelresolutie,
  streamwrapping, denkniveaus, herhalingsbeleid en gebruikseindpunten. Zie
  [Volgorde en gebruik van hooks](#hook-order-and-usage).

OpenClaw blijft eigenaar van de generieke agentlus, failover, transcriptverwerking en
toolbeleid. Deze hooks vormen het uitbreidingsoppervlak voor providerspecifiek
gedrag zonder dat een volledig aangepast inferentietransport nodig is.

Gebruik manifest `setup.providers[].envVars` wanneer de provider omgevingsvariabele-gebaseerde
inloggegevens heeft die generieke paden voor authenticatie/status/modelselectie moeten kunnen zien zonder
de Plugin-runtime te laden. Gebruik manifest `providerAuthAliases`
wanneer één provider-id de omgevingsvariabelen, authenticatieprofielen,
configuratiegebaseerde authenticatie en API-sleutelkeuze voor onboarding van een andere provider-id moet hergebruiken. Gebruik manifest
`providerAuthChoices` wanneer CLI-oppervlakken voor onboarding/authenticatiekeuze de
keuze-id, groepslabels en eenvoudige authenticatiekoppeling met één vlag van de provider moeten kennen zonder
de provider-runtime te laden. Behoud provider-runtime
`envVars` voor op operators gerichte aanwijzingen, zoals onboardinglabels of variabelen voor
het instellen van OAuth-client-id/clientgeheim.

Beschrijf omgevingsvariabele-gestuurde kanaalconfiguratie en authenticatie via de bijbehorende
`channelConfigs.<id>.schema` en configuratiedescriptors.

### Volgorde en gebruik van hooks

Voor model-/providerplugins roept OpenClaw hooks ongeveer in deze volgorde aan.
De kolom 'Wanneer gebruiken' dient als snelle beslisgids.
Provider­velden die uitsluitend voor compatibiliteit bestaan en die OpenClaw niet meer aanroept, zoals
`ProviderPlugin.capabilities` en `suppressBuiltInModel`, zijn hier bewust niet
opgenomen.

| Hook                              | Wat deze doet                                                                                                   | Wanneer te gebruiken                                                                                                                                   |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `catalog`                         | Publiceer providerconfiguratie naar `models.providers` tijdens het genereren van `models.json`                                | De provider beheert een catalogus of standaardwaarden voor de basis-URL                                                                                                  |
| `applyConfigDefaults`             | Pas algemene, door de provider beheerde configuratiestandaarden toe tijdens het materialiseren van de configuratie                                      | Standaarden zijn afhankelijk van de authenticatiemodus, omgeving of semantiek van de modelfamilie van de provider                                                                         |
| _(ingebouwde modelzoekfunctie)_         | OpenClaw probeert eerst het normale register-/cataloguspad                                                          | _(geen Plugin-hook)_                                                                                                                         |
| `normalizeModelId`                | Normaliseer verouderde aliassen of previewaliassen van model-id's vóór het opzoeken                                                     | De provider beheert het opschonen van aliassen vóór de canonieke modelresolutie                                                                                 |
| `normalizeTransport`              | Normaliseer `api` / `baseUrl` van de providerfamilie vóór de generieke modelsamenstelling                                      | De provider beheert het opschonen van het transport voor aangepaste provider-id's binnen dezelfde transportfamilie                                                          |
| `normalizeConfig`                 | Normaliseer `models.providers.<id>` vóór runtime-/providerresolutie                                           | De provider heeft configuratieopschoning nodig die bij de plugin hoort; gebundelde helpers voor de Google-familie dienen ook als vangnet voor ondersteunde Google-configuratie-items   |
| `applyNativeStreamingUsageCompat` | Pas compatibiliteitsherschrijvingen voor native streaminggebruik toe op configuratieproviders                                               | De provider heeft door het eindpunt aangestuurde correcties nodig voor metadata over native streaminggebruik                                                                          |
| `resolveConfigApiKey`             | Los authenticatie via omgevingsmarkeringen op voor configuratieproviders vóór het laden van runtime-authenticatie                                       | Providers bieden hun eigen hooks voor het oplossen van API-sleutels via omgevingsmarkeringen                                                                                |
| `resolveSyntheticAuth`            | Maak lokale/zelfgehoste of configuratiegestuurde authenticatie beschikbaar zonder platte tekst op te slaan                                   | De provider kan werken met een synthetische/lokale referentie voor aanmeldgegevens                                                                                 |
| `resolveExternalAuthProfiles`     | Leg door de provider beheerde externe authenticatieprofielen eroverheen; de standaardwaarde voor `persistence` is `runtime-only` voor aanmeldgegevens die door de CLI/app worden beheerd | De provider hergebruikt externe authenticatiegegevens zonder gekopieerde vernieuwingstokens op te slaan; declareer `contracts.externalAuthProviders` in het manifest |
| `shouldDeferSyntheticProfileAuth` | Verlaag de prioriteit van opgeslagen synthetische profielplaatsaanduidingen ten opzichte van omgevings-/configuratiegestuurde authenticatie                                      | De provider slaat synthetische plaatsaanduidingsprofielen op die geen voorrang mogen krijgen                                                                 |
| `resolveDynamicModel`             | Synchrone terugval voor door de provider beheerde model-id's die nog niet in het lokale register staan                                       | De provider accepteert willekeurige bovenliggende model-id's                                                                                                 |
| `prepareDynamicModel`             | Asynchrone opwarming, waarna `resolveDynamicModel` opnieuw wordt uitgevoerd                                                           | De provider heeft netwerkmetadata nodig voordat onbekende id's kunnen worden opgelost                                                                                  |
| `normalizeResolvedModel`          | Laatste herschrijving voordat de ingebedde runner het opgeloste model gebruikt                                               | De provider heeft transportherschrijvingen nodig, maar gebruikt nog steeds een kerntransport                                                                             |
| `normalizeToolSchemas`            | Normaliseer toolschema's voordat de ingebedde runner ze ziet                                                    | De provider heeft schemaschoning voor de transportfamilie nodig                                                                                                |
| `inspectToolSchemas`              | Maak door de provider beheerde schemadiagnostiek beschikbaar na normalisatie                                                  | De provider wil waarschuwingen voor trefwoorden zonder providerspecifieke regels aan de kern toe te voegen                                                                 |
| `resolveReasoningOutputMode`      | Selecteer het contract voor native versus getagde redeneeruitvoer                                                              | De provider heeft getagde redenerings-/einduitvoer nodig in plaats van native velden                                                                         |
| `prepareExtraParams`              | Normalisatie van aanvraagparameters vóór generieke wrappers voor streamopties                                              | De provider heeft standaardaanvraagparameters of opschoning van parameters per provider nodig                                                                           |
| `createStreamFn`                  | Vervang het normale streampad volledig door een aangepast transport                                                   | De provider heeft een aangepast wireprotocol nodig, niet alleen een wrapper                                                                                     |
| `wrapStreamFn`                    | Streamwrapper nadat generieke wrappers zijn toegepast                                                              | De provider heeft compatibiliteitswrappers voor aanvraagheaders/-body/model nodig zonder aangepast transport                                                          |
| `resolveTransportTurnState`       | Voeg native transportheaders of metadata per beurt toe                                                           | De provider wil dat generieke transporten een native beurtidentiteit van de provider verzenden                                                                       |
| `resolveWebSocketSessionPolicy`   | Voeg native WebSocket-headers of beleid voor sessieafkoeling toe                                                    | De provider wil dat generieke WS-transporten sessieheaders of terugvalbeleid afstemmen                                                               |
| `formatApiKey`                    | Formatter voor authenticatieprofielen: het opgeslagen profiel wordt de runtime-tekenreeks `apiKey`                                     | De provider slaat extra authenticatiemetadata op en heeft een aangepaste vorm van het runtimetoken nodig                                                                    |
| `refreshOAuth`                    | Overschrijving van OAuth-vernieuwing voor aangepaste vernieuwingseindpunten of beleid bij mislukte vernieuwing                                  | De provider past niet bij de gedeelde OpenClaw-vernieuwers                                                                                          |
| `buildAuthDoctorHint`             | Reparatietip die wordt toegevoegd wanneer OAuth-vernieuwing mislukt                                                                  | De provider heeft na een mislukte vernieuwing eigen richtlijnen nodig om de authenticatie te herstellen                                                                      |
| `matchesContextOverflowError`     | Door de provider beheerde matcher voor overschrijding van het contextvenster                                                                 | De provider heeft onbewerkte overloopfouten die generieke heuristieken zouden missen                                                                                |
| `classifyFailoverReason`          | Door de provider beheerde classificatie van redenen voor failover                                                                  | De provider kan onbewerkte API-/transportfouten koppelen aan snelheidsbeperking/overbelasting/enzovoort                                                                          |
| `isCacheTtlEligible`              | Beleid voor de promptcache voor proxy-/backhaulproviders                                                               | De provider heeft proxyspecifieke TTL-gating voor de cache nodig                                                                                                |
| `buildMissingAuthMessage`         | Vervanging voor het generieke herstelbericht bij ontbrekende authenticatie                                                      | De provider heeft een providerspecifieke hersteltip voor ontbrekende authenticatie nodig                                                                                 |
| `augmentModelCatalog`             | Synthetische/definitieve catalogusrijen die na ontdekking worden toegevoegd (verouderd, zie hieronder)                                  | De provider heeft synthetische rijen voor voorwaartse compatibiliteit nodig in `models list` en keuzelijsten                                                                     |
| `resolveThinkingProfile`          | Modelspecifieke instelling van het `/think`-niveau, weergavelabels en standaardwaarde                                                 | De provider biedt voor geselecteerde modellen een aangepaste denkladder of binair label                                                                 |
| `isBinaryThinking`                | Compatibiliteitshook voor het in-/uitschakelen van redeneren                                                                     | De provider biedt alleen binair in-/uitschakelen van denken                                                                                                  |
| `supportsXHighThinking`           | Compatibiliteitshook voor ondersteuning van `xhigh`-redenering                                                                   | De provider wil `xhigh` slechts voor een subset van modellen                                                                                             |
| `resolveDefaultThinkingLevel`     | Compatibiliteitshook voor het standaardniveau `/think`                                                                      | De provider beheert het standaardbeleid voor `/think` voor een modelfamilie                                                                                      |
| `isModernModelRef`                | Matcher voor moderne modellen voor liveprofielfilters en smokeselectie                                              | De provider beheert het matchen van voorkeursmodellen voor live-/smoketests                                                                                             |
| `prepareRuntimeAuth`              | Wissel een geconfigureerd aanmeldgegeven vlak vóór inferentie om voor het daadwerkelijke runtimetoken/de daadwerkelijke runtimesleutel                       | De provider heeft een tokenuitwisseling of kortlevend aanmeldgegeven voor de aanvraag nodig                                                                             |
| `resolveUsageAuth`                | Los gebruiks-/factureringsgegevens op voor `/usage` en gerelateerde statusoppervlakken                                     | De provider heeft aangepaste parsing van gebruiks-/quotatokens of andere gebruiksgegevens nodig                                                               |
| `fetchUsageSnapshot`              | Haal providerspecifieke momentopnamen van gebruik/quota op en normaliseer deze nadat de authenticatie is opgelost                             | De provider heeft een providerspecifiek gebruikseindpunt of parser voor de payload nodig                                                                           |
| `createEmbeddingProvider`         | Bouw een embeddingadapter van de provider voor geheugen/zoeken                                                     | Het gedrag voor geheugen-embeddings hoort bij de providerplugin                                                                                    |
| `buildReplayPolicy`               | Retourneer een replaybeleid dat de verwerking van transcripties voor de provider regelt                                        | De provider heeft een aangepast transcriptiebeleid nodig (bijvoorbeeld het verwijderen van denkblokken)                                                               |
| `sanitizeReplayHistory`           | Herschrijf de replaygeschiedenis na algemene opschoning van transcripties                                                        | De provider heeft providerspecifieke replayherschrijvingen nodig naast gedeelde Compaction-helpers                                                             |
| `validateReplayTurns`             | Voer de laatste validatie of hervorming van de replaybeurt uit vóór de ingebedde runner                                           | Het providertransport vereist strengere beurtvalidatie na algemene opschoning                                                                    |
| `onModelSelected`                 | Voer door de provider beheerde neveneffecten na selectie uit                                                                 | De provider heeft telemetrie of door de provider beheerde status nodig wanneer een model actief wordt                                                                  |

`normalizeModelId`, `normalizeTransport` en `normalizeConfig` controleren eerst de
overeenkomende provider-Plugin en gaan vervolgens de andere provider-Plugins
met hookondersteuning langs totdat er daadwerkelijk één de model-id of het
transport/de configuratie wijzigt. Zo blijven alias-/compatibiliteitsshims voor
providers werken zonder dat de aanroeper hoeft te weten welke gebundelde Plugin
de herschrijving beheert. Als geen providerhook een ondersteunde
configuratievermelding uit de Google-familie herschrijft, past de gebundelde
Google-configuratienormalisator die compatibiliteitsopschoning alsnog toe.

Als de provider een volledig aangepast wire-protocol of een aangepaste
requestexecutor nodig heeft, is dat een andere klasse extensie. Deze hooks zijn
bedoeld voor providergedrag dat nog steeds via de normale inferentielus van
OpenClaw wordt uitgevoerd.

`resolveUsageAuth` bepaalt of OpenClaw `fetchUsageSnapshot` moet aanroepen of
moet terugvallen op generieke referentieoplossing voor gebruiks-/statusoppervlakken.
Retourneer `{ token, accountId?, subscriptionType?, rateLimitTier? }` wanneer de provider
een gebruiksreferentie heeft (de optionele abonnementsmetadata stroomt door naar
`fetchUsageSnapshot`), retourneer
`{ handled: true }` wanneer de door de provider beheerde gebruiksauthenticatie
het verzoek heeft afgehandeld en de generieke terugval op API-sleutel/OAuth
moet onderdrukken, en retourneer `null` of `undefined`
wanneer de provider de gebruiksauthenticatie niet heeft afgehandeld.

Declareer organisatie- of factureringsreferenties in het manifest
`providerUsageAuthEnvVars`. Hierdoor kunnen generieke detectie- en
geheimenopschoningsoppervlakken ze herkennen zonder ze kandidaat te maken voor
inferentieauthenticatie.

### Providervoorbeeld

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### Ingebouwde voorbeelden

Gebundelde provider-Plugins combineren de bovenstaande hooks om aan de
catalogus-, authenticatie-, denk-, herhalings- en gebruiksvereisten van elke
leverancier te voldoen. De gezaghebbende hookset bevindt zich bij elke Plugin
onder `extensions/`; deze pagina illustreert de vormen in plaats van de
lijst te dupliceren.

<AccordionGroup>
  <Accordion title="Providers met doorgeefcatalogus">
    OpenRouter, Kilocode, Z.AI en xAI registreren `catalog` plus
    `resolveDynamicModel` / `prepareDynamicModel`, zodat ze upstream
    model-id's vóór de statische catalogus van OpenClaw kunnen aanbieden.
  </Accordion>
  <Accordion title="Providers met OAuth- en gebruikseindpunten">
    GitHub Copilot, Gemini CLI, ChatGPT Codex, MiniMax, Xiaomi en z.ai combineren
    `prepareRuntimeAuth` of `formatApiKey` met `resolveUsageAuth` +
    `fetchUsageSnapshot` om tokenuitwisseling en de integratie met
    `/usage` te beheren.
  </Accordion>
  <Accordion title="Families voor herhaling en transcriptopschoning">
    Gedeelde benoemde families (`google-gemini`, `passthrough-gemini`,
    `anthropic-by-model`, `hybrid-anthropic-openai`) laten providers via
    `buildReplayPolicy` kiezen voor transcriptbeleid, in plaats van dat elke
    Plugin de opschoning opnieuw implementeert.
  </Accordion>
  <Accordion title="Providers met alleen een catalogus">
    `byteplus`, `cloudflare-ai-gateway`, `huggingface`, `kimi-coding`, `nvidia`,
    `qianfan`, `synthetic`, `together`, `venice`, `vercel-ai-gateway` en
    `volcengine` registreren alleen `catalog` en gebruiken de gedeelde inferentielus.
  </Accordion>
  <Accordion title="Anthropic-specifieke streamhelpers">
    Betaheaders, `/fast` / `serviceTier` en `context1m` bevinden zich binnen de
    openbare `api.ts`- / `contract-api.ts`-naad van de Anthropic-Plugin
    (`wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
    `resolveAnthropicFastMode`, `resolveAnthropicServiceTier`) in plaats van in
    de generieke SDK.
  </Accordion>
</AccordionGroup>

## Runtimehelpers

Plugins hebben via `api.runtime` toegang tot geselecteerde kernhelpers. Voor TTS:

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

Opmerkingen:

- `textToSpeech` retourneert de normale TTS-uitvoerpayload van de kern voor bestands-/spraaknotitieoppervlakken.
- Gebruikt de kernconfiguratie `tts` en providerselectie.
- Retourneert een PCM-audiobuffer + samplefrequentie. Plugins moeten opnieuw samplen/coderen voor providers.
- `listVoices` is optioneel per provider. Gebruik dit voor door de leverancier beheerde stemkiezers of instelstromen.
- De kern geeft een opgeloste verzoekdeadline door aan providerhooks van `listVoices`; providerspecifieke time-outinstellingen kunnen deze overschrijven.
- Stemlijsten kunnen uitgebreidere metadata bevatten, zoals landinstelling, gender en persoonlijkheidstags voor providerbewuste keuzelijsten.
- OpenAI en ElevenLabs ondersteunen momenteel telefonie. Microsoft niet.

Plugins kunnen ook spraakproviders registreren via `api.registerSpeechProvider(...)`.

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

Opmerkingen:

- Behoud TTS-beleid, terugval en antwoordbezorging in de kern.
- Gebruik spraakproviders voor door leveranciers beheerd synthese­gedrag.
- Verouderde Microsoft-invoer voor `edge` wordt genormaliseerd naar de provider-id `microsoft`.
- Het voorkeursmodel voor eigenaarschap is bedrijfsgericht: één leveranciers-Plugin kan
  tekst-, spraak-, beeld- en toekomstige mediaproviders beheren wanneer OpenClaw
  deze capaciteitscontracten toevoegt.

Voor beeld-/audio-/videobegrip registreren Plugins één getypeerde
provider voor mediabegrip in plaats van een generieke sleutel/waarde-verzameling:

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

Opmerkingen:

- Behoud orkestratie, terugval, configuratie en kanaalbedrading in de kern.
- Behoud leveranciersgedrag in de provider-Plugin.
- Additieve uitbreiding moet getypeerd blijven: nieuwe optionele methoden, nieuwe optionele
  resultaatvelden, nieuwe optionele capaciteiten.
- Videogeneratie volgt al hetzelfde patroon:
  - de kern beheert het capaciteitscontract en de runtimehelper
  - leveranciers-Plugins registreren `api.registerVideoGenerationProvider(...)`
  - functie-/kanaal-Plugins gebruiken `api.runtime.videoGeneration.*`

Voor runtimehelpers voor mediabegrip kunnen Plugins het volgende aanroepen:

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});

const extraction = await api.runtime.mediaUnderstanding.extractStructuredWithModel({
  provider: "codex",
  model: "gpt-5.6-sol",
  input: [
    {
      type: "image",
      buffer: receiptImageBuffer,
      fileName: "receipt.png",
      mime: "image/png",
    },
    { type: "text", text: "Use the printed fields as the source of truth." },
  ],
  instructions: "Return entities and searchable tags.",
  schemaName: "example.evidence",
  jsonSchema: {
    type: "object",
    properties: {
      entities: { type: "array", items: { type: "string" } },
      tags: { type: "array", items: { type: "string" } },
    },
  },
  cfg: api.config,
});
```

Voor audiotranscriptie kunnen Plugins de runtime voor mediabegrip of de oudere
STT-alias gebruiken:

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

Opmerkingen:

- `api.runtime.mediaUnderstanding.*` is het gedeelde voorkeursoppervlak voor
  beeld-/audio-/videobegrip.
- `extractStructuredWithModel(...)` is de Plugin-gerichte naad voor begrensde,
  door providers beheerde beeldgerichte extractie. Neem ten minste één beeldinvoer op;
  tekstinvoer is aanvullende context. Product-Plugins beheren hun routes en
  schema's, terwijl OpenClaw de provider-/runtimegrens beheert.
- Gebruikt de audio­configuratie voor mediabegrip van de kern (`tools.media.audio`) en de terugvalvolgorde voor providers.
- Retourneert `{ text: undefined }` wanneer geen transcriptie-uitvoer wordt geproduceerd (bijvoorbeeld bij overgeslagen/niet-ondersteunde invoer).

Plugins kunnen ook subagentuitvoeringen op de achtergrond starten via `api.runtime.subagent`:

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  toolsAlsoAllow: ["my_plugin_progress"],
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

Opmerkingen:

- `provider` en `model` zijn optionele overschrijvingen per uitvoering, geen permanente sessiewijzigingen.
- `toolsAlsoAllow` accepteert exacte, uniek beheerde toolnamen die door de aanroepende Plugin zijn geregistreerd. Kernnamen en dubbelzinnige namen worden geweigerd. Dit is een toevoeging aan het normale profiel, maar allowlists en weigeringen van de operator blijven gezaghebbend.
- OpenClaw respecteert deze overschrijvingsvelden alleen voor vertrouwde aanroepers.
- Voor terugvaluitvoeringen die eigendom zijn van Plugins moeten operators expliciet toestemming geven met `plugins.entries.<id>.subagent.allowModelOverride: true`.
- Gebruik `plugins.entries.<id>.subagent.allowedModels` om vertrouwde Plugins te beperken tot specifieke canonieke `provider/model`-doelen, of `"*"` om elk doel expliciet toe te staan.
- Subagentuitvoeringen van niet-vertrouwde Plugins blijven werken, maar overschrijvingsverzoeken worden geweigerd in plaats van stilzwijgend terug te vallen.
- Door Plugins gemaakte subagentsessies worden gemarkeerd met de id van de makende Plugin. Terugval via `api.runtime.subagent.deleteSession(...)` mag alleen deze sessies waarvan zij eigenaar zijn verwijderen; voor het verwijderen van willekeurige sessies is nog steeds een Gateway-verzoek met beheerdersbereik vereist.

Voor zoeken op het web kunnen Plugins de gedeelde runtimehelper gebruiken in
plaats van de bedrading van de agenttool rechtstreeks te benaderen:

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw plugin runtime helpers",
    count: 5,
  },
});
```

Plugins kunnen ook providers voor zoeken op het web registreren via
`api.registerWebSearchProvider(...)`.

Opmerkingen:

- Behoud providerselectie, referentieoplossing en gedeelde verzoeksemantiek in de kern.
- Gebruik providers voor zoeken op het web voor leveranciersspecifieke zoektransporten.
- `api.runtime.webSearch.*` is het gedeelde voorkeursoppervlak voor functie-/kanaal-Plugins die zoekgedrag nodig hebben zonder afhankelijk te zijn van de agenttoolwrapper.

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "Een vriendelijke kreeftenmascotte", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`: genereer een afbeelding met de geconfigureerde keten van aanbieders voor afbeeldingsgeneratie.
- `listProviders(...)`: vermeld beschikbare aanbieders voor afbeeldingsgeneratie en hun mogelijkheden.

## HTTP-routes van de Gateway

Plugins kunnen HTTP-eindpunten beschikbaar stellen met `api.registerHttpRoute(...)`.

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

Routevelden:

- `path`: routepad onder de HTTP-server van de Gateway.
- `auth`: verplicht, `"gateway"` of `"plugin"`. Gebruik `"gateway"` om normale Gateway-authenticatie te vereisen, of `"plugin"` voor door de Plugin beheerde authenticatie/Webhook-verificatie.
- `match`: optioneel. `"exact"` (standaard) of `"prefix"`.
- `handleUpgrade`: optionele handler voor WebSocket-upgradeverzoeken op dezelfde route.
- `replaceExisting`: optioneel. Hiermee kan dezelfde Plugin zijn eigen bestaande routeregistratie vervangen.
- `handler`: retourneer `true` wanneer de route het verzoek heeft afgehandeld.

Opmerkingen:

- `api.registerHttpHandler(...)` is verwijderd en veroorzaakt een fout bij het laden van de Plugin. Gebruik in plaats daarvan `api.registerHttpRoute(...)`.
- Plugin-routes moeten `auth` expliciet declareren.
- Exacte conflicten voor `path + match` worden geweigerd, tenzij `replaceExisting: true`; een Plugin kan bovendien niet de route van een andere Plugin vervangen.
- Overlappende routes met verschillende `auth`-niveaus worden geweigerd. Houd doorvalketens voor `exact`/`prefix` uitsluitend op hetzelfde authenticatieniveau.
- Routes met `auth: "plugin"` ontvangen **niet** automatisch runtimebereiken voor operators. Ze zijn bedoeld voor door de Plugin beheerde Webhooks/handtekeningverificatie, niet voor bevoorrechte aanroepen van Gateway-helpers.
- Routes met `auth: "gateway"` worden uitgevoerd binnen een runtimescope voor Gateway-verzoeken. Het standaardoppervlak (`gatewayRuntimeScopeSurface: "write-default"`) is bewust terughoudend:
  - bearer-authenticatie met een gedeeld geheim (`gateway.auth.mode = "token"` / `"password"`) en elke authenticatiemethode zonder vertrouwde proxy krijgen één `operator.write`-scope, zelfs als de aanroeper `x-openclaw-scopes` verzendt
  - aanroepers met `trusted-proxy` zonder expliciete `x-openclaw-scopes`-header behouden ook het verouderde oppervlak met uitsluitend `operator.write`
  - aanroepers met `trusted-proxy` die wel `x-openclaw-scopes` verzenden, krijgen in plaats daarvan de gedeclareerde scopes
  - een route kan zich aanmelden voor `gatewayRuntimeScopeSurface: "trusted-operator"` om `x-openclaw-scopes` altijd te respecteren voor authenticatiemodi die een identiteit bevatten (waarbij wordt teruggevallen op de volledige standaardset CLI-scopes als de header ontbreekt)
- Gesandboxte externe Control UI-tabbladen die worden ondersteund door routes met `auth: "gateway"`, gebruiken een kortstondige, ondertekende cookietoekenning die uitsluitend door een geauthenticeerde bootstrap wordt aangemaakt; tabbladen met Plugin-authenticatie behouden hun directe iframe-pad. Vóór het koppelen voert het bovenliggende element binnen dezelfde ondoorzichtige sandbox een probe uit die eigendom is van de route, en het weigert veilig wanneer het privacybeleid van de browser de cookie blokkeert. De toekenning is gebonden aan de bezittende Plugin, de overeenkomende routehoofdmap en de huidige authenticatiegeneratie; de proceswillekeurige cookienaam voorkomt dat vertrouwde Gateways op dezelfde host elkaars cookies overschrijven, maar cookies isoleren TCP-poorten nooit. De hostnaam van de Gateway vormt daarom één grens voor inloggegevens: host geen onderling onvertrouwde services op die hostnaam, ook niet op andere poorten. Routering weigert hergebruik voor een geneste route die eigendom is van een andere Plugin. Omdat afstammelingen van de sandbox voor cookies als cross-site gelden, accepteert de toekenning uitsluitend `GET` en `HEAD` met `operator.read`; mutaties en WebSocket-upgrades blijven op expliciet door de Gateway geauthenticeerde oppervlakken. De cookie kan bewust geen CHIPS gebruiken: huidige browsers nemen een cross-site-ancestor-bit op in de partitiesleutel, waardoor geneste ondoorzichtige sandboxframes geen toegang meer zouden hebben tot assets van dezelfde route. De cookie vereist een beveiligde context en browsertoestemming voor cross-sitecookies. Daardoor zijn externe tabbladen met Gateway-authenticatie niet beschikbaar op LAN-oorsprongen met gewone HTTP of wanneer cookies van derden volledig worden geblokkeerd; gebruik HTTPS/Tailscale Serve of een door de browser vertrouwde loopback met een compatibel cookiebeleid.
- De toekenning voorkomt openbaarmaking van het bearer-token van de Gateway en onbedoeld hergebruik van routes/scopes; ze creëert geen beveiligingsgrens tussen native Plugins. Native Plugincode en de UI-inhoud die deze aanbiedt, blijven deel uitmaken van dezelfde vertrouwde Plugin-grens binnen het proces.
- Praktische regel: neem niet aan dat een Plugin-route met Gateway-authenticatie impliciet een beheerdersoppervlak is. Als je route gedrag vereist dat uitsluitend voor beheerders bestemd is, meld je dan aan voor het scopeoppervlak `trusted-operator`, vereis een authenticatiemodus die een identiteit bevat en documenteer het expliciete contract voor de `x-openclaw-scopes`-header.
- Na het matchen van de route en de authenticatie nemen gewone handlers deel aan de toelating van hoofdwerk voor de Gateway. Een voorbereide of opnieuw startende Gateway retourneert `503` voordat de handler wordt aangeroepen. De beperkte uitzondering is een door het manifest toegestane route met `auth: "gateway"` die zich tevens aanmeldt voor het routespecifieke oppervlak `trusted-operator`; deze blijft bereikbaar zodat de routering voor opschortingsbeheer niet vastloopt, terwijl gewone zusterroutes van dezelfde Plugin achter de toelatingsgrens blijven. Het eigendom van WebSocket-`handleUpgrade` gebruikt dezelfde atomaire toelatingsgrens; zodra de handler een socket accepteert, is de verdere levensduur van de socket eigendom van de Plugin en wordt deze niet door deze grens gevolgd.

## Importpaden van de Plugin-SDK

Gebruik bij het maken van nieuwe Plugins smalle SDK-subpaden in plaats van de monolithische
hoofdbarrel `openclaw/plugin-sdk`. Kernsubpaden:

| Subpad                             | Doel                                         |
| ---------------------------------- | -------------------------------------------- |
| `openclaw/plugin-sdk/plugin-entry` | Primitieven voor Plugin-registratie          |
| `openclaw/plugin-sdk/channel-core` | Helpers voor kanaaltoegang en -opbouw        |
| `openclaw/plugin-sdk/core`         | Generieke gedeelde helpers en overkoepelend contract |

Kanaalplugins kiezen uit een familie van smalle koppelvlakken — `channel-setup`,
`setup-runtime`, `setup-tools`, `channel-pairing`,
`channel-contract`, `channel-feedback`, `channel-inbound`, `channel-outbound`,
`command-auth`, `secret-input`, `webhook-ingress`,
`channel-targets` en `channel-actions`. Goedkeuringsgedrag moet worden samengebracht
in één `approvalCapability`-contract in plaats van het te mengen met niet-gerelateerde
Plugin-velden. Zie [Kanaalplugins](/nl/plugins/sdk-channel-plugins).

Runtime- en configuratiehelpers bevinden zich onder overeenkomstige, gerichte `*-runtime`-subpaden
(`approval-runtime`, `agent-runtime`, `lazy-runtime`, `directory-runtime`,
`text-runtime`, `runtime-store`, `system-event-runtime`, `heartbeat-runtime`,
`channel-activity-runtime`, enzovoort). Geef de voorkeur aan `config-contracts`,
`plugin-config-runtime`, `runtime-config-snapshot` en `config-mutation`
in plaats van de brede compatibiliteitsbarrel `config-runtime`.

<Info>
`openclaw/plugin-sdk/channel-lifecycle`, kleine façades voor kanaalhelpers,
`openclaw/plugin-sdk/config-runtime` en `openclaw/plugin-sdk/infra-runtime`
zijn verouderde compatibiliteitsshims voor oudere Plugins. Nieuwe code moet in plaats daarvan
smallere generieke primitieven importeren.
</Info>

Interne toegangspunten van de repository (per hoofdmap van een gebundeld Plugin-pakket):

- `index.js` — toegangspunt voor gebundelde Plugin
- `api.js` — barrel voor helpers/typen
- `runtime-api.js` — barrel uitsluitend voor runtime
- `setup-entry.js` — toegangspunt voor instellings-Plugin

Externe Plugins mogen uitsluitend `openclaw/plugin-sdk/*`-subpaden importeren. Importeer nooit
de `src/*` van een ander Plugin-pakket vanuit de kern of een andere Plugin.
Via façades geladen toegangspunten geven de voorkeur aan de actieve momentopname van de runtimeconfiguratie
wanneer deze bestaat, en vallen vervolgens terug op het opgeloste configuratiebestand op schijf.

Mogelijkheidsspecifieke subpaden zoals `image-generation`, `media-understanding`
en `speech` bestaan omdat gebundelde Plugins ze momenteel gebruiken. Ze zijn niet
automatisch langdurig bevroren externe contracten — raadpleeg de relevante
SDK-referentiepagina wanneer je erop vertrouwt.

## Schema's voor berichttools

Plugins moeten eigenaar zijn van kanaalspecifieke bijdragen aan het `describeMessageTool(...)`-schema
voor primitieven die geen berichten zijn, zoals reacties, leesbevestigingen en peilingen.
Gedeelde verzendpresentatie moet het generieke `MessagePresentation`-contract gebruiken
in plaats van provider-native velden voor knoppen, componenten, blokken of kaarten.
Zie [Berichtpresentatie](/nl/plugins/message-presentation) voor het contract,
de terugvalregels, providertoewijzing en de controlelijst voor Plugin-auteurs.

Plugins die kunnen verzenden, declareren via berichtmogelijkheden wat ze kunnen weergeven:

- `presentation` voor semantische presentatieblokken (`text`, `context`,
  `divider`, `chart`, `table`, `buttons`, `select`)
- `delivery-pin` voor verzoeken om vastgezette bezorging

De kern bepaalt of de presentatie native wordt weergegeven of wordt teruggebracht tot tekst.
Stel vanuit de generieke berichttool geen provider-native achterdeuren voor de UI beschikbaar.
Verouderde SDK-helpers voor oude native schema's blijven geëxporteerd voor bestaande
Plugins van derden, maar nieuwe Plugins mogen ze niet gebruiken.

## Resolutie van kanaaldoelen

Kanaalplugins moeten eigenaar zijn van kanaalspecifieke doelsemantiek. Houd de gedeelde
uitgaande host generiek en gebruik het oppervlak van de berichtenadapter voor providerregels:

- `messaging.inferTargetChatType({ to })` bepaalt of een genormaliseerd doel
  vóór het opzoeken in de directory moet worden behandeld als `direct`, `group` of `channel`.
- `messaging.targetResolver.looksLikeId(raw, normalized)` vertelt de kern of een
  invoer direct moet doorgaan naar ID-achtige resolutie in plaats van de directory te doorzoeken.
- `messaging.targetResolver.reservedLiterals` vermeldt losse woorden die
  kanaal-/sessieverwijzingen voor die provider zijn. De resolutie behoudt geconfigureerde
  directoryvermeldingen voordat gereserveerde letterlijke waarden worden geweigerd, en weigert vervolgens veilig
  wanneer de directory geen resultaat oplevert.
- `messaging.targetResolver.resolveTarget(...)` is de terugval van de Plugin wanneer
  de kern na normalisatie of nadat de directory geen resultaat oplevert een laatste resolutie door de provider nodig heeft.
- `messaging.resolveOutboundSessionRoute(...)` is eigenaar van providerspecifieke opbouw van
  sessieroutes zodra een doel is opgelost.

Aanbevolen verdeling:

- Gebruik `inferTargetChatType` voor categoriebeslissingen die vóór het
  zoeken naar peers/groepen moeten plaatsvinden.
- Gebruik `looksLikeId` voor controles van het type „behandel dit als een expliciete/native doel-ID”.
- Gebruik `resolveTarget` voor providerspecifieke normalisatieterugval, niet voor
  brede directoryzoekopdrachten.
- Bewaar provider-native ID's zoals chat-ID's, thread-ID's, JID's, handles en ruimte-ID's
  in `target`-waarden of providerspecifieke parameters, niet in generieke SDK-velden.

## Door configuratie ondersteunde directory's

Plugins die directoryvermeldingen uit configuratie afleiden, moeten die logica in de
Plugin houden en de gedeelde helpers uit
`openclaw/plugin-sdk/directory-runtime` hergebruiken.

Gebruik dit wanneer een kanaal door configuratie ondersteunde peers/groepen nodig heeft, zoals:

- door een toelatingslijst bepaalde DM-peers
- geconfigureerde toewijzingen van kanalen/groepen
- accountgebonden statische directoryterugvallen

De gedeelde helpers in `directory-runtime` verwerken uitsluitend generieke bewerkingen:

- queryfiltering
- toepassing van limieten
- helpers voor ontdubbeling/normalisatie
- opbouw van `ChannelDirectoryEntry[]`

Kanaalspecifieke accountinspectie en ID-normalisatie moeten in de
Plugin-implementatie blijven.

## Providercatalogi

Providerplugins kunnen modelcatalogi voor inferentie definiëren met
`registerProvider({ catalog: { run(...) { ... } } })`.

`catalog.run(...)` retourneert dezelfde structuur die OpenClaw schrijft naar
`models.providers`:

- `{ provider }` voor één providervermelding
- `{ providers }` voor meerdere providervermeldingen

Gebruik `catalog` wanneer de plugin providerspecifieke model-id's, standaardwaarden voor de
basis-URL of door authenticatie afgeschermde modelmetadata beheert.

`catalog.order` bepaalt wanneer de catalogus van een plugin wordt samengevoegd ten opzichte van de
ingebouwde impliciete providers van OpenClaw:

- `simple`: gewone providers die door een API-sleutel of omgevingsvariabelen worden aangestuurd
- `profile`: providers die verschijnen wanneer authenticatieprofielen bestaan
- `paired`: providers die meerdere gerelateerde providervermeldingen genereren
- `late`: laatste doorgang, na andere impliciete providers

Latere providers winnen bij een sleutelconflict, zodat plugins bewust een
ingebouwde providervermelding met dezelfde provider-id kunnen overschrijven.

Plugins kunnen ook alleen-lezen modelrijen publiceren via
`api.registerModelCatalogProvider({ provider, kinds, staticCatalog, liveCatalog
})`. Dit is het toekomstige pad voor lijst-, hulp- en keuzeschermoppervlakken en ondersteunt
rijen voor `text`, `voice`, `image_generation`, `video_generation` en `music_generation`.
Providerplugins blijven verantwoordelijk voor live endpointaanroepen, tokenuitwisseling en
het toewijzen van leveranciersreacties; de kern beheert de algemene rijvorm, bronlabels en
de opmaak van hulp voor mediatools. Providerregistraties voor mediageneratie genereren
automatisch statische catalogusrijen op basis van `defaultModel`, `models` en
`capabilities`.

Compatibiliteit:

- `discovery` werkt nog steeds als verouderde alias, maar geeft een waarschuwing over uitfasering
- als zowel `catalog` als `discovery` zijn geregistreerd, gebruikt OpenClaw `catalog`
  en geeft het een waarschuwing
- `augmentModelCatalog` is verouderd; gebundelde providers moeten
  aanvullende rijen publiceren via `registerModelCatalogProvider`

## Alleen-lezen kanaalinspectie

Als je plugin een kanaal registreert, implementeer dan bij voorkeur
`plugin.config.inspectAccount(cfg, accountId)` naast `resolveAccount(...)`.

Waarom:

- `resolveAccount(...)` is het runtimepad. Dit mag ervan uitgaan dat inloggegevens
  volledig beschikbaar zijn gemaakt en kan direct mislukken wanneer vereiste geheimen ontbreken.
- Alleen-lezen commandopaden zoals `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve` en herstelstromen voor doctor/config
  zouden geen runtime-inloggegevens beschikbaar hoeven te maken alleen om
  de configuratie te beschrijven.

Aanbevolen gedrag voor `inspectAccount(...)`:

- Retourneer alleen een beschrijvende accountstatus.
- Behoud `enabled` en `configured`.
- Neem waar relevant velden voor bron/status van inloggegevens op, zoals:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- Je hoeft geen onbewerkte tokenwaarden te retourneren alleen om alleen-lezen
  beschikbaarheid te melden. Het retourneren van `tokenStatus: "available"` (en het bijbehorende
  bronveld) is voldoende voor statusachtige commando's.
- Gebruik `configured_unavailable` wanneer inloggegevens via SecretRef zijn geconfigureerd, maar
  niet beschikbaar zijn in het huidige commandopad.

Hierdoor kunnen alleen-lezen commando's melden dat iets „geconfigureerd maar niet beschikbaar is in dit
commandopad”, in plaats van te crashen of ten onrechte te melden dat het account niet is geconfigureerd.

## Pakketbundels

Een pluginmap kan een `package.json` met `openclaw.extensions` bevatten:

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Elke vermelding wordt een plugin. Als de bundel meerdere extensies vermeldt, wordt de plugin-id
`<manifestOrPackageName>/<fileBase>` (de manifest-id heeft voorrang wanneer
deze aanwezig is; anders de niet-gescopete `package.json`-naam).

Als je plugin npm-afhankelijkheden importeert, installeer je die in die map zodat
`node_modules` beschikbaar is (`npm install` / `pnpm install`).

Beveiligingsmaatregel: elke `openclaw.extensions`-vermelding moet na het oplossen van symbolische
koppelingen binnen de pluginmap blijven. Vermeldingen die buiten de pakketmap vallen, worden
geweigerd.

Beveiligingsopmerking: `openclaw plugins install` installeert plugin-afhankelijkheden met een
projectlokale `npm install --omit=dev --ignore-scripts` (geen levenscyclusscripts,
geen ontwikkelafhankelijkheden tijdens runtime), waarbij overgenomen algemene npm-installatie-instellingen worden genegeerd.
Houd afhankelijkheidsstructuren van plugins „puur JS/TS” en vermijd pakketten waarvoor
`postinstall`-builds vereist zijn.

Optioneel: `openclaw.setupEntry` kan verwijzen naar een lichtgewicht module die alleen voor installatie dient.
Wanneer OpenClaw installatieoppervlakken nodig heeft voor een uitgeschakelde kanaalplugin, of
wanneer een kanaalplugin is ingeschakeld maar nog niet is geconfigureerd, laadt het `setupEntry`
in plaats van de volledige pluginvermelding. Dit houdt het opstarten en de installatie lichter
wanneer je hoofdpluginvermelding ook tools, hooks of andere code
uitsluitend voor runtime koppelt.

Optioneel: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
kan een kanaalplugin tijdens de opstartfase vóór het luisteren van de Gateway hetzelfde `setupEntry`-pad laten gebruiken,
zelfs wanneer het kanaal al is geconfigureerd.

Gebruik dit alleen wanneer `setupEntry` het opstartoppervlak dat beschikbaar moet zijn
voordat de Gateway begint te luisteren, volledig afdekt. In de praktijk betekent dit dat de installatievermelding
elke door het kanaal beheerde mogelijkheid moet registreren waarvan het opstarten afhankelijk is, zoals:

- de kanaalregistratie zelf
- alle HTTP-routes die beschikbaar moeten zijn voordat de Gateway begint te luisteren
- alle Gateway-methoden, tools of services die tijdens hetzelfde tijdsvenster beschikbaar moeten zijn

Als je volledige vermelding nog steeds een vereiste opstartmogelijkheid beheert, schakel
deze vlag dan niet in. Laat de plugin het standaardgedrag gebruiken en laat OpenClaw tijdens
het opstarten de volledige vermelding laden.

Gebundelde kanalen kunnen ook installatiehulpfuncties publiceren die uitsluitend het contractoppervlak bieden en die de kern
kan raadplegen voordat de volledige kanaalruntime is geladen. Het huidige installatieoppervlak
voor promotie is:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

De kern gebruikt dat oppervlak wanneer een verouderde kanaalconfiguratie voor één account moet worden
gepromoveerd naar `channels.<id>.accounts.*` zonder de volledige pluginvermelding te laden.
Matrix is het huidige gebundelde voorbeeld: het verplaatst alleen authenticatie-/bootstrap-sleutels naar een
benoemd gepromoveerd account wanneer er al benoemde accounts bestaan, en het kan een
geconfigureerde niet-canonieke sleutel voor het standaardaccount behouden in plaats van altijd
`accounts.default` te maken.

Die installatiepatchadapters houden de detectie van gebundelde contractoppervlakken lui. De importtijd
blijft kort; het promotieoppervlak wordt pas bij het eerste gebruik geladen in plaats van
bij module-import opnieuw het opstartproces van het gebundelde kanaal te starten.

Wanneer die opstartoppervlakken RPC-methoden van de Gateway bevatten, houd ze dan onder een
pluginspecifiek voorvoegsel. De beheernaamruimten van de kern (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) blijven gereserveerd en worden altijd omgezet
naar `operator.admin`, zelfs als een plugin om een beperktere scope vraagt.

Voorbeeld:

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### Metadata van de kanaalcatalogus

Kanaalplugins kunnen metadata voor installatie/detectie bekendmaken via `openclaw.channel` en
installatietips via `openclaw.install`. Hierdoor blijft de kerncatalogus vrij van gegevens.

Voorbeeld:

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (zelfgehost)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Zelfgehoste chat via Nextcloud Talk-webhookbots.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

Nuttige `openclaw.channel`-velden naast het minimale voorbeeld:

- `detailLabel`: secundair label voor uitgebreidere catalogus-/statusoppervlakken
- `docsLabel`: overschrijf de linktekst voor de documentatielink
- `preferOver`: plugin-/kanaal-id's met lagere prioriteit die deze catalogusvermelding moet overtreffen
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: tekstinstellingen voor het selectieoppervlak
- `markdownCapable`: markeert het kanaal als geschikt voor Markdown bij beslissingen over uitgaande opmaak
- `exposure.configured`: verberg het kanaal voor lijstoppervlakken met geconfigureerde kanalen wanneer ingesteld op `false`
- `exposure.setup`: verberg het kanaal voor interactieve keuzevensters voor installatie/configuratie wanneer ingesteld op `false`
- `exposure.docs`: markeer het kanaal als intern/privé voor navigatieoppervlakken in de documentatie
- `quickstartAllowFrom`: laat het kanaal deelnemen aan de standaard `allowFrom`-snelstartstroom
- `forceAccountBinding`: vereis expliciete accountkoppeling, zelfs wanneer er maar één account bestaat
- `preferSessionLookupForAnnounceTarget`: geef de voorkeur aan sessieopzoeking bij het bepalen van aankondigingsdoelen

OpenClaw kan ook **externe kanaalcatalogi** samenvoegen (bijvoorbeeld een export van een
MPM-register). Plaats een JSON-bestand op een van deze locaties:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

Of laat `OPENCLAW_PLUGIN_CATALOG_PATHS` (of `OPENCLAW_MPM_CATALOG_PATHS`) verwijzen naar
een of meer JSON-bestanden (gescheiden door komma's, puntkomma's of `PATH`). Elk bestand moet
`{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }` bevatten. De parser accepteert ook `"packages"` of `"plugins"` als verouderde aliassen voor de sleutel `"entries"`.

Gegenereerde vermeldingen in de kanaalcatalogus en de installatiecatalogus voor providers tonen
genormaliseerde feiten over de installatiebron naast het onbewerkte `openclaw.install`-blok. De
genormaliseerde feiten geven aan of de npm-specificatie een exacte versie of een zwevende
selector is, of de verwachte integriteitsmetadata aanwezig zijn en of er ook een lokaal
bronpad beschikbaar is. Wanneer de catalogus-/pakketidentiteit bekend is, waarschuwen de
genormaliseerde feiten als de geparseerde npm-pakketnaam afwijkt van die identiteit.
Ze waarschuwen ook wanneer `defaultChoice` ongeldig is of naar een bron verwijst die
niet beschikbaar is, en wanneer npm-integriteitsmetadata aanwezig zijn zonder een geldige npm-
bron. Consumenten moeten `installSource` behandelen als een aanvullend optioneel veld, zodat
handmatig gemaakte vermeldingen en catalogusshims dit niet hoeven te genereren.
Hierdoor kunnen onboarding en diagnostiek de status van het bronvlak uitleggen zonder
de pluginruntime te importeren.

Officiële externe npm-vermeldingen moeten bij voorkeur een exacte `npmSpec` plus
`expectedIntegrity` gebruiken. Kale pakketnamen en dist-tags blijven
om compatibiliteitsredenen werken, maar tonen waarschuwingen over het bronvlak zodat de catalogus kan overstappen
op vastgezette, op integriteit gecontroleerde installaties zonder bestaande plugins te breken.
Wanneer onboarding installeert vanaf een lokaal cataloguspad, registreert het een beheerde
plugin-indexvermelding met `source: "path"` en waar mogelijk een werkruimterelatieve
`sourcePath`. Het absolute operationele laadpad blijft in
`plugins.load.paths`; de installatieregistratie voorkomt dat lokale werkstationpaden dubbel worden
opgenomen in langlevende configuratie. Hierdoor blijven lokale ontwikkelinstallaties zichtbaar voor
diagnostiek van het bronvlak zonder een tweede oppervlak voor openbaarmaking van onbewerkte bestandssysteempaden
toe te voegen. De persistente SQLite-tabel `installed_plugin_index` is de gezaghebbende bron
voor installaties en kan worden vernieuwd zonder pluginruntimemodules te laden.
De `installRecords`-toewijzing ervan blijft behouden, zelfs wanneer een pluginmanifest ontbreekt of
ongeldig is; de `plugins`-payload is een opnieuw opbouwbare manifestweergave.

## Plugins voor de contextengine

Plugins voor de contextengine beheren de orkestratie van sessiecontext voor opname, samenstelling
en Compaction. Registreer ze vanuit je plugin met
`api.registerContextEngine(id, factory)` en selecteer vervolgens de actieve engine met
`plugins.slots.contextEngine`.

Gebruik dit wanneer je plugin de standaard contextpijplijn moet vervangen of
uitbreiden, in plaats van alleen geheugenzoekfuncties of hooks toe te voegen.

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("lossless-claw", (ctx) => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, sessionKey, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

De factory `ctx` stelt optionele waarden `config`, `agentDir` en `workspaceDir`
beschikbaar voor initialisatie tijdens de constructie.

De host voltooit de geregistreerde asynchrone voorbereiding van de geheugenprompt voordat
`assemble()` van een niet-verouderde engine wordt aangeroepen. `buildMemorySystemPromptAddition(...)` blijft
synchroon en leest die onveranderlijke momentopname van de run terwijl `assemble()` actief is.
Geef de aangeleverde context voor tools en citaten ongewijzigd door, zodat de momentopname
geen rungrenzen kan overschrijden.

`assemble()` kan `contextProjection` retourneren wanneer de actieve harnasomgeving een
persistente backendthread heeft. Laat dit weg voor verouderde projectie per beurt. Retourneer
`{ mode: "thread_bootstrap", epoch }` wanneer de samengestelde context eenmaal in een backendthread moet worden
geïnjecteerd en hergebruikt totdat het tijdperk verandert. Wijzig het tijdperk nadat de semantische
context van de engine verandert, bijvoorbeeld na een Compaction-pass die eigendom is van de
engine. Hosts mogen metadata van toolaanroepen, de invoervorm en geredigeerde toolresultaten
behouden in een opstartprojectie voor threads, zodat nieuwe backendthreads de toolcontinuïteit
behouden zonder onbewerkte payloads met geheimen te kopiëren.

Als jouw engine het Compaction-algoritme **niet** beheert, houd `compact()`
geïmplementeerd en delegeer het expliciet:

```ts
import {
  buildMemorySystemPromptAddition,
  delegateCompactionToRuntime,
} from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", (ctx) => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, sessionKey, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## Een nieuwe mogelijkheid toevoegen

Wanneer een plugin gedrag nodig heeft dat niet binnen de huidige API past, omzeil
het pluginsysteem dan niet via een private directe toegang. Voeg de ontbrekende mogelijkheid toe.

Aanbevolen volgorde:

1. **Definieer het kerncontract.** Bepaal welk gedeeld gedrag de kern moet beheren:
   beleid, terugvalgedrag, samenvoeging van configuratie, levenscyclus, kanaalgerichte semantiek en
   de vorm van runtimehelpers.
2. **Voeg getypeerde oppervlakken voor pluginregistratie en runtime toe.** Breid
   `OpenClawPluginApi` en/of `api.runtime` uit met het kleinst bruikbare getypeerde
   mogelijkheidsoppervlak.
3. **Koppel de kern en kanaal-/functieconsumenten.** Kanalen en functieplugins
   moeten de nieuwe mogelijkheid via de kern gebruiken, niet door rechtstreeks een
   leveranciersimplementatie te importeren.
4. **Registreer leveranciersimplementaties.** Leveranciersplugins registreren
   vervolgens hun backends voor de mogelijkheid.
5. **Voeg contractdekking toe.** Voeg tests toe zodat eigendom en registratievorm
   in de loop van de tijd expliciet blijven.

Zo blijft OpenClaw uitgesproken zonder hardgecodeerd te raken volgens het
wereldbeeld van één provider. Zie het [Kookboek voor mogelijkheden](/nl/plugins/adding-capabilities)
voor een concrete bestandschecklist en een uitgewerkt voorbeeld.

### Checklist voor mogelijkheden

Wanneer je een nieuwe mogelijkheid toevoegt, moet de implementatie doorgaans deze
oppervlakken gezamenlijk aanpassen:

- kerncontracttypen in `src/<capability>/types.ts`
- kernrunner/runtimehelper in `src/<capability>/runtime.ts`
- registratieoppervlak van de plugin-API in `src/plugins/types.ts`
- bedrading van het pluginregister in `src/plugins/registry.ts`
- beschikbaarstelling via de pluginruntime in `src/plugins/runtime/*` wanneer functie-/kanaalplugins
  deze moeten gebruiken
- vastleggings-/testhelpers in `src/test-utils/plugin-registration.ts`
- asserties voor eigendom/contracten in `src/plugins/contracts/registry.ts`
- documentatie voor operators/plugins in `docs/`

Als een van die oppervlakken ontbreekt, is dat meestal een teken dat de mogelijkheid
nog niet volledig is geïntegreerd.

### Sjabloon voor mogelijkheden

Minimaal patroon:

```ts
// core contract
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// plugin API
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// shared runtime helper for feature/channel plugins
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

Patroon voor contracttests (`src/plugins/contracts/registry.ts` stelt eigendomszoekfuncties
zoals `providerContractPluginIds` beschikbaar; tests controleren of de
`contracts.videoGenerationProviders`-lijst van een plugin overeenkomt met wat deze daadwerkelijk registreert):

```ts
expect(pluginManifest.contracts?.videoGenerationProviders).toEqual(["openai"]);
```

Zo blijft de regel eenvoudig:

- de kern beheert het mogelijkheidscontract en de orkestratie
- leveranciersplugins beheren leveranciersimplementaties
- functie-/kanaalplugins gebruiken runtimehelpers
- contracttests houden het eigendom expliciet

## Gerelateerd

- [Pluginarchitectuur](/nl/plugins/architecture) — openbaar mogelijkheidsmodel en vormen
- [Subpaden van de Plugin SDK](/nl/plugins/sdk-subpaths)
- [Configuratie van de Plugin SDK](/nl/plugins/sdk-setup)
- [Plugins bouwen](/nl/plugins/building-plugins)
