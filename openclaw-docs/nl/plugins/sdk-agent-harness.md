---
read_when:
    - Je wijzigt de ingebouwde agentruntime of het harnessregister
    - Je registreert een agentharnas vanuit een gebundelde of vertrouwde plugin
    - Je moet begrijpen hoe de Codex-plugin zich verhoudt tot modelproviders
sidebarTitle: Agent Harness
summary: Experimentele SDK-interface voor plugins die de ingebouwde agentuitvoerder op laag niveau vervangen
title: Plugins voor agentharnassen
x-i18n:
    generated_at: "2026-07-27T06:29:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ff4e41a46ba0074fc6c8bf46da813b58d074f5e6c5c1d236d7ab78e824bdc02
    source_path: plugins/sdk-agent-harness.md
    workflow: 16
---

Een **agentharnas** is de low-level uitvoerder voor één voorbereide OpenClaw-agentbeurt. Het is geen modelprovider, geen kanaal en geen toolregister. Zie [Agentruntimes](/nl/concepts/agent-runtimes) voor het gebruikersgerichte mentale model.

Gebruik dit oppervlak alleen voor gebundelde of vertrouwde native plugins. Het contract is nog experimenteel, omdat de parametertypen bewust de huidige ingebedde runner weerspiegelen.

## Wanneer een harnas te gebruiken

Registreer een agentharnas wanneer een modelfamilie een eigen native sessieruntime heeft en het normale OpenClaw-providertransport de verkeerde abstractie is:

- een native coding-agentserver die threads en Compaction beheert
- een lokale CLI of daemon die native plan-/redeneer-/toolgebeurtenissen moet streamen
- een modelruntime die naast het OpenClaw-sessietranscript een eigen hervattings-id nodig heeft

Registreer **geen** harnas alleen om een nieuwe LLM-API toe te voegen. Bouw voor normale HTTP- of WebSocket-model-API's een [providerplugin](/nl/plugins/sdk-provider-plugins).

## Wat de kern nog steeds beheert

Voordat een harnas wordt geselecteerd, heeft OpenClaw het volgende al bepaald:

- provider en model
- runtime-authenticatiestatus, tenzij het harnas verklaart dat het de authenticatiebootstrap beheert
- denkniveau en contextbudget
- het OpenClaw-transcript-/sessiebestand
- werkruimte-, sandbox- en toolbeleid
- callbacks voor kanaalantwoorden en streaming
- beleid voor modelterugval en live modelwisseling

Een harnas voert een voorbereide poging uit; het kiest geen providers, vervangt geen kanaalbezorging en wisselt niet stilzwijgend van model.

### Door het harnas beheerde authenticatiebootstrap

Standaard bepaalt de kern de providerreferenties voordat een harnas wordt aangeroepen. Een vertrouwd harnas dat zich via zijn eigen native runtime kan authenticeren, mag
`authBootstrap: "harness"` instellen bij zijn statische `AgentHarness`-registratie. De kern slaat dan voor elke door dat harnas geclaimde poging de generieke bootstrap van providerreferenties en de fout bij ontbrekende referenties over.

De kern geeft nog steeds een compatibel, expliciet geselecteerd of geordend OpenClaw-authenticatieprofiel en de bijbehorende afgebakende opslag door wanneer die bestaan. Het harnas moet dat profiel of zijn native referenties bepalen voordat het modelaanvragen uitvoert, geheimen tot de poging beperken en bruikbare authenticatiefouten melden. Stel deze mogelijkheid niet in voor een harnas dat authenticatie slechts soms beheert.

### Geverifieerde runtimeartefacten voor de installatie

Een lokaal harnas dat inferentie kan leveren voor de eerste installatie, moet verklaren welke implementatie de probe heeft voltooid. Wanneer
`params.captureRuntimeArtifact` waar is, retourneer je een ondoorzichtig
`result.runtimeArtifact` met een stabiele id en inhoudsvingerafdruk. Registreer een overeenkomende `runtimeArtifact.validate(...)`-mogelijkheid die die binding opnieuw controleert zonder een ander harnas te laden of niet-gerelateerde plugins te scannen.

Geverifieerde OpenClaw-voortzettingen geven ook `params.expectedRuntimeArtifact` door.
Het harnas moet dit vergelijken met het exacte native proces dat het heeft verkregen en falen voordat het een native thread start of hervat als ze verschillen. Bij gewone agentbeurten worden beide velden weggelaten, zodat inhoudshashing buiten het normale hot path van aanvragen blijft. Externe/WebSocket-harnassen hebben een serverattestatiecontract nodig voordat ze kunnen deelnemen; alleen een versietekenreeks is geen artefactidentiteit.

De voorbereide poging bevat ook `params.runtimePlan`, een door OpenClaw beheerde beleidsbundel voor runtimebeslissingen die gedeeld moeten blijven tussen OpenClaw en native harnassen:

- `runtimePlan.tools.normalize(...)` en `runtimePlan.tools.logDiagnostics(...)`
  voor providerbewust beleid voor toolschema's
- `runtimePlan.transcript.resolvePolicy(...)` voor opschoning van transcripties en
  beleid voor herstel van toolaanroepen
- `runtimePlan.delivery.isSilentPayload(...)` voor gedeelde `NO_REPLY` en onderdrukking van
  mediabezorging
- `runtimePlan.outcome.classifyRunResult(...)` voor classificatie van
  modelterugval
- `runtimePlan.observability` voor opgeloste metadata van provider/model/harnas

Harnassen mogen het plan gebruiken voor beslissingen die met OpenClaw-gedrag moeten overeenkomen, maar moeten het behandelen als pogingstatus die door de host wordt beheerd: wijzig het niet en gebruik het niet om binnen een beurt van provider/model te wisselen.

### Contract voor aanvraagtransport

`supports(ctx)` ontvangt het opgeloste modeltransport in `ctx.modelProvider`.
Twee geheimevrije, door de provider beheerde feiten beschrijven de geselecteerde route:

- `runtimePolicy.compatibleIds` vermeldt de runtime-id's die de provider compatibel verklaart
  met die concrete route. Een ontbrekend beleid betekent dat de provider geen
  compatibiliteit op routeniveau heeft verklaard; het is geen toestemming om ondersteuning te veronderstellen.
- `requestTransportOverrides: "none"` betekent dat geen handmatig opgegeven overschrijving van een
  provider-/modelaanvraag hoeft te worden gereproduceerd. `"present"` betekent dat handmatig opgegeven headers, authenticatietransport, proxy-, TLS-, lokale-service- of privénetwerkgedrag of aanvraagparameters bestaan. Het feit stelt die waarden niet beschikbaar.

Retourneer `{ supported: false, reason }` wanneer het harnas het voorbereide transport niet kan reproduceren. Leid ondersteuning niet af door na de selectie de onbewerkte configuratie te lezen.
Wanneer de authenticatievoorbereiding meerdere routes voor nieuwe pogingen oplevert, moet één harnas ze allemaal ondersteunen voordat verzending plaatsvindt. Impliciete selectie gebruikt OpenClaw als geen enkele plugin de volledige set kan beheren; een expliciete of persistente pluginselectie faalt gesloten.

## Een harnas registreren

**Import:** `openclaw/plugin-sdk/agent-harness`

```typescript
import type { AgentHarness } from "openclaw/plugin-sdk/agent-harness";
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const myHarness: AgentHarness = {
  id: "my-harness",
  label: "Mijn native agentharnas",

  supports(ctx) {
    const routeSupportsHarness =
      ctx.modelProvider?.runtimePolicy?.compatibleIds.includes("my-harness") === true;
    const canReproduceRequest = ctx.modelProvider?.requestTransportOverrides !== "present";
    return ctx.provider === "my-provider" && routeSupportsHarness && canReproduceRequest
      ? { supported: true, priority: 100 }
      : { supported: false, reason: "effectieve route is niet compatibel met het harnas" };
  },

  async runAttempt(params) {
    // Start of hervat je native thread.
    // Gebruik params.prompt, params.tools, params.images, params.onPartialReply,
    // params.onAgentEvent en de andere velden van de voorbereide poging.
    return await runMyNativeTurn(params);
  },
};

export default definePluginEntry({
  id: "my-native-agent",
  name: "Mijn native agent",
  description: "Voert geselecteerde modellen uit via een native agentdaemon.",
  register(api) {
    api.registerAgentHarness(myHarness);
  },
});
```

`authBootstrap` ontbreekt bewust in dit generieke voorbeeld. Voeg
`authBootstrap: "harness"` alleen toe wanneer het harnas aan het bovenstaande contract voldoet.

### Gedelegeerde uitvoering

De eigenaar van een harnas mag `delegatedExecutionPluginIds` instellen op de id's van vertrouwde
plugins die een bestaande modelvergrendelde sessie moeten uitvoeren, zoals een spraaktransport dat een door Codex ondersteund gesprek voortzet. Dit is statische toestemming van de eigenaar, geen allowlist van de kern. Houd deze beperkt.

Gedelegeerden ontvangen alleen werktoelating en ingebedde uitvoering. OpenClaw vereist
de exact opgeslagen sessiesleutel, het opslagpad en de sessie-id; `modelSelectionLocked:
true`; en overeenkomende waarden voor `agentHarnessId` en `agentHarnessRuntimeOverride`.
De uitvoering wordt vervolgens afgebakend via de eigenaar van het harnas. Het aanmaken, patchen, resetten, verwijderen en archiveren van sessies en Gateway-mutatie blijven uitsluitend voorbehouden aan de eigenaar.

## Selectiebeleid

OpenClaw kiest een harnas nadat provider en model zijn bepaald:

1. Runtimebeleid op modelniveau heeft voorrang.
2. Runtimebeleid op providerniveau volgt daarna.
3. `auto` vraagt geregistreerde harnassen of ze de opgeloste effectieve
   route ondersteunen. Alleen provider-/modelvoorvoegsels selecteren nooit een harnas.
4. Als geen geregistreerd harnas overeenkomt, gebruikt OpenClaw zijn ingebedde runtime.

Fouten van pluginharnassen worden weergegeven als uitvoeringsfouten. In de modus `auto` is ingebedde terugval alleen van toepassing wanneer geen geregistreerd pluginharnas de opgeloste provider/het opgeloste model ondersteunt. Zodra een pluginharnas een uitvoering heeft geclaimd, speelt OpenClaw diezelfde beurt niet opnieuw af via een andere runtime, omdat dit authenticatie-/runtimesemantiek kan wijzigen of neveneffecten kan dupliceren.

Het geconfigureerde runtimebeleid blijft bepalend voor de gewenste runtime. Een persistente sessie-`agentHarnessId` behoudt het eigendom van het native transcript terwijl de route-/authenticatievoorbereiding nog in behandeling is. Geen van beide maakt een incompatibele route compatibel: zodra voorbereide feiten bestaan, moet het geselecteerde of vastgezette harnas deze ondersteunen, anders faalt de uitvoering gesloten. `/status` toont de effectieve runtime die is geselecteerd op basis van beleid, persistent eigendom en routeondersteuning.
De voorbereidingsstatus is expliciet: ontbrekende `runtimePolicy` blijft niet-gedeclareerd in plaats van te worden afgeleid uit toevallig aanwezige transportvelden.
Wanneer door het harnas beheerde authenticatie meerdere fysieke routes onopgelost laat, is het voorbereide ondersteuningsfeit de doorsnede van hun compatibele runtime-id's en meldt het aanvraagoverschrijvingen als een kandidaat die heeft. Eén niet-gedeclareerde kandidaat maakt native compatibiliteit daarom leeg; `preparedAuth.source: "harness"`
is een authenticatie-eigenaar, geen toestemming om routeondersteuning af te leiden.

Als het geselecteerde harnas onverwacht is, schakel dan `agents/harness`-debuglogboekregistratie in
en inspecteer de gestructureerde `agent harness selected`-record van de Gateway: deze bevat de geselecteerde harnas-id, selectiereden, runtime-/terugvalbeleid en, in de modus `auto`, het ondersteuningsresultaat van elke pluginkandidaat.

De gebundelde Codex-plugin registreert `codex` als zijn harnas-id. De kern behandelt dit als een gewone harnas-id van een plugin; Codex-specifieke aliassen horen thuis in de plugin- of operatorconfiguratie, niet in de gedeelde runtimeselector.

## Combinatie van provider en harnas

De meeste harnassen moeten ook een provider registreren. De provider maakt modelreferenties, authenticatiestatus, modelmetadata en `/model`-selectie zichtbaar voor de rest van OpenClaw. Het harnas claimt die provider vervolgens in `supports(...)`.

De gebundelde Codex-plugin volgt dit patroon:

- voorkeursreferenties voor gebruikersmodellen: `openai/gpt-5.6-sol`
- compatibiliteitsreferenties: verouderde `codex/gpt-*`-referenties blijven geaccepteerd, maar nieuwe
  configuraties moeten ze niet gebruiken als normale provider-/modelreferenties
- harnas-id: `codex`
- authenticatie: synthetische providerbeschikbaarheid, omdat het Codex-harnas de
  native Codex-aanmelding/-sessie beheert
- app-serveraanvraag: OpenClaw stuurt de kale model-id naar Codex en laat het
  harnas communiceren met het native app-serverprotocol

De Codex-plugin is additief. Als runtimebeleid niet is ingesteld of `auto` is, mag OpenAI Codex alleen selecteren wanneer het door de provider beheerde routecontract `codex` compatibel verklaart: een exacte officiële HTTPS-route voor Platform Responses of ChatGPT Responses zonder handmatig opgegeven aanvraagoverschrijving. Alleen het voorvoegsel `openai/*` selecteert Codex nooit. Aangepaste eindpunten, Completions-adapters en handmatig opgegeven aanvraaggedrag blijven op OpenClaw. Officiële HTTP-eindpunten met platte tekst worden geweigerd. Oudere `codex/gpt-*`-referenties blijven compatibiliteitsinvoer. Zie
[Impliciete OpenAI-agentruntime](/nl/providers/openai#implicit-agent-runtime).

Zie [Codex-harnas](/nl/plugins/codex-harness) voor operatorinstallatie, voorbeelden van modelvoorvoegsels en configuraties uitsluitend voor Codex.

De Codex-plugin handhaaft de minimale app-serverversie die is gedocumenteerd in
[Codex-harnas](/nl/plugins/codex-harness). De plugin controleert de initialisatiehandshake en blokkeert oudere servers of servers zonder versie, zodat OpenClaw alleen wordt uitgevoerd tegen het protocoloppervlak dat het heeft getest.

### Middleware voor toolresultaten

Gebundelde plugins en expliciet ingeschakelde geïnstalleerde plugins met overeenkomende manifestcontracten kunnen runtimeneutrale middleware voor toolresultaten koppelen via
`api.registerAgentToolResultMiddleware(...)` wanneer hun manifest de beoogde runtime-id's declareert in `contracts.agentToolResultMiddleware`. Deze vertrouwde naad is bedoeld voor asynchrone transformaties van toolresultaten die moeten worden uitgevoerd voordat OpenClaw of Codex tooluitvoer terugvoert naar het model.

Oudere gebundelde plugins kunnen nog steeds
`api.registerCodexAppServerExtensionFactory(...)` gebruiken voor middleware die uitsluitend voor de Codex-app-server
is bedoeld, maar nieuwe resultaattransformaties moeten de runtime-neutrale API gebruiken. De
hook `api.registerEmbeddedExtensionFactory(...)`, die uitsluitend voor de embedded runner was bedoeld, is
verwijderd; transformaties van ingebedde toolresultaten moeten runtime-neutrale middleware gebruiken.

### Classificatie van terminale uitkomsten

Native harnassen die hun eigen protocolprojectie beheren, kunnen
`classifyAgentHarnessTerminalOutcome(...)` uit
`openclaw/plugin-sdk/agent-harness-runtime` gebruiken wanneer een voltooide beurt geen
zichtbare assistenttekst heeft opgeleverd. De helper retourneert `empty`, `reasoning-only` of
`planning-only`, zodat het fallbackbeleid van OpenClaw kan bepalen of een nieuwe poging met een
ander model moet worden gedaan. `planning-only` vereist het expliciete veld `planText`
van het harnas; OpenClaw leidt dit niet af uit tekst van de assistent. De helper
laat promptfouten, lopende beurten en opzettelijk stille
antwoorden zoals `NO_REPLY` bewust ongeclassificeerd.

### Neveneffecten aan het einde van de agent

Native harnassen moeten `runAgentEndSideEffects(...)` uit
`openclaw/plugin-sdk/agent-harness-runtime` aanroepen nadat ze een poging hebben afgerond. Deze
activeert de overdraagbare hook `agent_end` en de onderzoeksregistratie van OpenClaw
zonder interactieve antwoorden te vertragen. Gebruik `awaitAgentEndSideEffects(...)` voor
lokale, niet-interactieve uitvoeringen waarbij de poging pas mag worden afgehandeld nadat die
neveneffecten zijn voltooid. Beide helpers accepteren dezelfde `{ event, ctx }`-payload als
`runAgentHarnessAgentEndHook(...)`; hun fouten wijzigen het resultaat van de voltooide
poging niet.

### Gebruikersinvoer en tooloppervlakken

Native harnassen die een gebruikersinvoerverzoek op runtimeniveau aanbieden, moeten de
gebruikersinvoerhelpers uit `openclaw/plugin-sdk/agent-harness-runtime` gebruiken om
de prompt op te maken, deze via het blokkerende antwoordpad van OpenClaw te versturen en
keuzeantwoorden of vrije invoer terug te normaliseren naar de native antwoordstructuur van de runtime. De
helper houdt de presentatie in kanalen en de TUI consistent, terwijl elk harnas zijn
eigen protocolverwerking en levenscyclus voor openstaande verzoeken behoudt.

Native harnassen die compacte, PI-achtige toolroutering nodig hebben, moeten
`createAgentHarnessToolSurfaceRuntime(...)` uit
`openclaw/plugin-sdk/agent-harness-tool-runtime` gebruiken. Deze beheert
de selectie van besturing voor toolzoekopdrachten en codemodus, compacte standaardinstellingen voor lokale modellen,
runtimecompatibele schemafiltering, verborgen catalogusuitvoering, het vullen van mappen
en het opschonen van de catalogus. Harnassen blijven verantwoordelijk voor hun SDK-specifieke
toolconversie en native uitvoeringscallback.

### Native Codex-harnasmodus

Het gebundelde harnas `codex` is de native Codex-modus voor ingebedde
OpenClaw-agentbeurten. Schakel eerst de gebundelde plugin `codex` in en neem `codex` op in
`plugins.allow` als je configuratie een beperkende toelatingslijst gebruikt. Native app-serverconfiguraties
moeten `openai/gpt-*` gebruiken; OpenAI-agentbeurten selecteren het Codex-harnas
alleen wanneer de effectieve route Codex-compatibiliteit declareert. Oudere Codex-modelreferenties
moeten worden hersteld met `openclaw doctor --fix`, en oudere
`codex/*`-modelreferenties blijven compatibiliteitsaliassen voor het native harnas.

Wanneer deze modus wordt uitgevoerd, beheert Codex de native thread-id, het hervattingsgedrag,
Compaction en de uitvoering van de app-server. OpenClaw blijft verantwoordelijk voor het chatkanaal,
de zichtbare transcriptiespiegel, het toolbeleid, goedkeuringen, medialevering en de selectie
van sessies. Gebruik provider/model `agentRuntime.id: "codex"` wanneer je moet
aantonen dat alleen het Codex-app-serverpad de uitvoering kan claimen. Expliciete plugin-
runtimes stoppen bij fouten; selectiefouten en runtimefouten van de Codex-app-server
worden niet opnieuw via een andere runtime geprobeerd.

## Striktheid van de runtime

Standaard gebruikt OpenClaw het provider/model-runtimebeleid `auto`: geregistreerde
pluginharnassen kunnen compatibele effectieve routes claimen en de ingebedde
runtime verwerkt de beurt wanneer geen enkel harnas overeenkomt. Alleen een provider/modelvoorvoegsel
selecteert nooit een harnas. Gebruik een expliciete provider/model-pluginruntime zoals
`agentRuntime.id: "codex"` wanneer het ontbreken van een harnasselectie tot een fout moet leiden
in plaats van routering via de ingebedde runtime. Expliciete selectie maakt een
incompatibele route niet compatibel. Fouten in geselecteerde pluginharnassen leiden altijd
tot een harde fout. Dit blokkeert geen expliciete provider/model-
`agentRuntime.id: "openclaw"`.

Voor uitsluitend in Codex ingebedde uitvoeringen:

```json
{
  "models": {
    "providers": {
      "openai": {
        "agentRuntime": {
          "id": "codex"
        }
      }
    }
  },
  "agents": {
    "defaults": {
      "model": "openai/gpt-5.6-sol"
    }
  }
}
```

Als je voor één canoniek model een CLI-backend wilt, plaats je de runtime bij die
modelvermelding:

```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-5",
      "models": {
        "anthropic/claude-opus-5": {
          "agentRuntime": {
            "id": "claude-cli"
          }
        }
      }
    }
  }
}
```

Overschrijvingen per agent gebruiken dezelfde modelgebonden structuur:

```json
{
  "agents": {
    "list": [
      {
        "id": "codex-only",
        "model": "openai/gpt-5.6-sol",
        "models": {
          "openai/gpt-5.6-sol": {
            "agentRuntime": { "id": "codex" }
          }
        }
      }
    ]
  }
}
```

Oudere runtimevoorbeelden voor de volledige agent, zoals dit voorbeeld, worden genegeerd:

```json
{
  "agents": {
    "defaults": {
      "agentRuntime": {
        "id": "codex"
      }
    }
  }
}
```

Met een expliciete pluginruntime mislukt een sessie vroegtijdig wanneer het gevraagde
harnas niet is geregistreerd, de opgeloste provider/het opgeloste model niet ondersteunt of
mislukt voordat neveneffecten van de beurt ontstaan. Dat is opzettelijk voor implementaties die uitsluitend
Codex gebruiken en voor livetests die moeten aantonen dat het Codex-app-serverpad
daadwerkelijk wordt gebruikt.

Deze instelling regelt alleen het ingebedde agentharnas. Ze schakelt
routering van modellen voor afbeeldingen, video, muziek, TTS, PDF of andere providerspecifieke doeleinden niet uit.

## Native sessies en transcriptiespiegel

Een harnas kan een native sessie-id, thread-id of hervattingstoken aan de daemonzijde
bewaren. Houd die koppeling expliciet verbonden met de OpenClaw-sessie en
blijf voor gebruikers zichtbare uitvoer van de assistent en tools spiegelen naar het OpenClaw-
transcript.

Het OpenClaw-transcript blijft de compatibiliteitslaag voor:

- kanaalzichtbare sessiegeschiedenis
- zoeken en indexeren van transcripten
- terugschakelen naar het ingebouwde OpenClaw-harnas tijdens een latere beurt
- algemeen gedrag voor `/new`, `/reset` en het verwijderen van sessies

Als je harnas een nevenbestand met een koppeling opslaat, implementeer dan `reset(...)`, zodat OpenClaw
dit kan wissen wanneer de bijbehorende OpenClaw-sessie opnieuw wordt ingesteld.

## Tool- en mediaresultaten

De kern stelt de OpenClaw-toollijst samen en geeft deze door aan de voorbereide
poging. Wanneer een harnas een dynamische toolaanroep uitvoert, retourneer je het toolresultaat
via de resultaatstructuur van het harnas in plaats van zelf kanaalmedia
te verzenden.

Zo blijven tekst-, afbeeldings-, video-, muziek-, TTS-, goedkeurings- en berichtentooluitvoer
op hetzelfde leveringspad als door OpenClaw ondersteunde uitvoeringen.

Stel `AgentHarnessAttemptResult.hostOwnedToolMediaUrls` alleen in voor native artefacten
die de vertrouwde harnasruntime zelf heeft gemaakt en opgeslagen. Elke vermelding moet
ook in `toolMediaUrls` voorkomen. Neem nooit media op die zijn geselecteerd door dynamische modeltools of
OpenClaw-tools. Op `message_tool_only`-routes zorgt deze beperkte herkomst ervoor dat
native runtimeartefacten de onderdrukking van bronantwoorden kunnen overleven; het normale verzendbeleid
en de toelating tot omgevingsruimten blijven van toepassing.

### Terminale tooluitkomsten

`AgentHarnessAttemptParams.observeToolTerminal` is de door de host beheerde accumulator voor terminale
uitkomsten. Een harnas dat dynamische OpenClaw-tools of native
tools uitvoert, moet deze aanroepen wanneer elke tool één terminale uitkomst bereikt, voordat het
resultaat van de poging wordt afgerond. Harnassen die geen tools uitvoeren, hoeven deze niet
aan te roepen.

Rapporteer feiten vanaf de uitvoeringsgrens:

- Geef de protocolaanroep-id door als die bestaat, samen met de canonieke toolnaam en de
  argumenten die de tool daadwerkelijk hebben bereikt na voorbereiding of herschrijving door hooks.
- Stel `executionStarted: false` in wanneer validatie, goedkeuring of een andere beveiliging
  de aanroep heeft gestopt voordat de toolimplementatie begon. Zodra dispatch mogelijk
  heeft plaatsgevonden, rapporteer je voorzichtigheidshalve `true`.
- Rapporteer `outcome: "success"` of `outcome: "failure"`. Neem de gestructureerde
  foutvelden op die vanuit de runtime beschikbaar zijn, in plaats van een fout af te leiden uit
  weergegeven tekst.
- Gebruik `nativeMutation` alleen voor native tools die geen OpenClaw-
  tooldefinitie gebruiken. Geef daar protocolgebonden mutatie- en herhalingsfeiten op; kopieer
  de mutatieclassificatie van OpenClaw niet naar het harnas.

De callback retourneert de canonieke afhandeling voor die aanroep. Neem de
`lastToolError` daarvan op in `AgentHarnessAttemptResult` en gebruik de uitvoerings-,
argument- en neveneffectfeiten ervan in de harnasprojectie in plaats van
parallelle toestand af te leiden. De host behoudt een niet-afgehandelde muterende fout na niet-gerelateerde
succesvolle tools en wist deze pas nadat de overeenkomende actie is geslaagd.

De callback blijft optioneel voor broncompatibiliteit met oudere experimentele
harnassen. Optioneel betekent niet dat een harnas dat tools uitvoert deze mag negeren:
zonder terminale rapportages kan OpenClaw de waarheidsgetrouwe status van fouten in muterende tools
niet behouden tussen latere toolaanroepen, inclusief de stille voltooiing van een Heartbeat.

### Afronding van afgehandelde tools

OpenClaw heeft mogelijk nog één definitief zichtbaar antwoord nodig nadat een harnas alle
toolaanroepen heeft voltooid, maar de native beurt zonder assistenttekst is geëindigd. Een harnas kan
zich voor dat herstel aanmelden door `finalizeSettledTurn({ attempt,
settledAttempt })` te implementeren.

De callback is een afzonderlijke mogelijkheid, geen gewone extra poging. Deze moet:

- ofwel het exacte beperkte native transcript gebruiken, ofwel een volledig applicatie-
  transcript dat tot en met de grens van het afgehandelde toolresultaat is bevroren;
- geen tools, mogelijkheden voor het verlenen van machtigingen of gebruikersinvoer, native uitvoerings-
  hooks, agents, Skills, geheugen, planning, uitbreidingen of externe bediening beschikbaar stellen;
- alleen de door de host verstrekte afrondingsprompt verzenden; en
- bij fouten stoppen als de geselecteerde transcript-/isolatiestrategie
  die beperkingen niet kan afdwingen.

OpenClaw roept de callback eenmaal aan als terminale subbewerking, buiten de
gewone poging- en herhalingslus. Een fout beëindigt de uitvoering met de
neveneffectbewuste waarschuwing voor een onvoltooide beurt; deze kan niet naar gewone
rotatie van authenticatie/profielen, modelfallback, contextherstel, voortzetting na
Compaction of door hooks aangevraagde revisiepaden gaan. Bij afronding worden ook mutatie van pluginprompts,
`before_agent_run`, LLM-invoer/-uitvoer, terminale revisie en
`agent_end`-hooks overgeslagen. Kerndiagnostiek registreert de bewerking en de fout nog steeds.

De callback retourneert `AgentHarnessSettledTurnFinalizationResult`, geen
gewoon resultaat van een poging. De openbare velden zijn beperkt tot het voltooide
assistentbericht, het gebruik van de afrondingsaanroep, metadata over transcripteigendom en
diagnostische tracering. Tool-, leverings-, media-, spawn-, levenscyclus-, herhalings-, sessie- en
fallbackstatus kunnen deze resultaatgrens niet overschrijden. Onbekende velden en
toolaanroepen van de assistent leiden tot een fout.

Een harnas dat intern zijn volledige pogingsengine hergebruikt, kan
`projectSettledTurnFinalizationAttemptResult(...)` aanroepen voordat het retourneert. De helper
weigert canoniek bewijs voor fouten, tools, levering, herhaling en levenscyclus en
projecteert vervolgens alleen het beperkte resultaat. Dit is gelaagde beveiliging na native isolatie,
geen vervanging voor het verwijderen van het native mogelijkhedenoppervlak.

Een projectiegebonden harnas moet de volledige context op
`settledAttempt.settledTurnFinalizationContext` plaatsen met
`source: "openclaw-transcript"`. Het moet de actieve vertakking vastleggen nadat de
afgehandelde beurt is gespiegeld, aantonen dat de huidige prompt en elke huidige tool-
aanroep/elk huidig toolresultaat tot en met die grens aanwezig zijn en de resulterende berichtenarray
bevriezen voordat de poging wordt geretourneerd. De afrondingsfunctie moet een ontbrekende,
niet-ondersteunde, dubbelzinnige of te grote context weigeren. Ze mag berichten niet afkappen,
eerdere geschiedenis niet verwijderen en dit applicatietranscript niet als exacte native
geschiedenis beschrijven. Harnassen die één beperkte native sessie hervatten, hebben dit
projectieveld niet nodig.

Implementeer deze callback niet door `runAttempt` aan te roepen met een best-effort-
hint `disableTools`. De eigenaar van het harnas moet de volledige native
mogelijkhedengrens afdwingen. OpenClaw biedt geen algemene fallback, omdat het
niet kan bevestigen dat een willekeurige native runtime die beperkingen heeft nageleefd.

De callback blijft optioneel voor compatibiliteit met experimentele harnesses
van derden. Wanneer de geselecteerde harness deze weglaat, behoudt OpenClaw de
bestaande fout voor een onvoltooide beurt in plaats van herhaalde neveneffecten te riskeren.

## Huidige beperkingen

- Het openbare importpad is generiek, maar sommige typealiassen voor pogingen/resultaten
  dragen voor compatibiliteit nog steeds verouderde namen.
- De installatie van harnesses van derden is experimenteel. Geef de voorkeur aan providerplugins
  totdat je een systeemeigen sessieruntime nodig hebt.
- Wisselen tussen harnesses wordt tussen beurten ondersteund. Wissel niet van harness
  midden in een beurt nadat systeemeigen tools, goedkeuringen, assistenttekst of het verzenden van
  berichten zijn gestart.

## Gerelateerd

- [SDK-overzicht](/nl/plugins/sdk-overview)
- [Runtimehelpers](/nl/plugins/sdk-runtime)
- [Providerplugins](/nl/plugins/sdk-provider-plugins)
- [Codex-harness](/nl/plugins/codex-harness)
- [Modelproviders](/nl/concepts/model-providers)
