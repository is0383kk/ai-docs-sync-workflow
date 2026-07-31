---
read_when:
    - Je wilt de OpenClaw-codemodus inschakelen voor een agentrun
    - Je moet uitleggen waarom Code Mode anders is dan Codex Code Mode
    - Je beoordeelt het compacte toolcontract, de QuickJS-WASI-sandbox, de TypeScript-transformatie of de verborgen toolcatalogusbridge
    - Je voegt een interne integratie met het namespace-register voor de codemodus toe of beoordeelt deze
sidebarTitle: Code Mode
summary: Gebruik OpenClaw Code Mode om grote toolcatalogi te ontdekken, aan te roepen en te combineren in compacte JavaScript- of TypeScript-workflows
title: Codusmodus
x-i18n:
    generated_at: "2026-07-27T05:24:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a21df3bcfb11668da6dde1f7c69adcc284a28dc491c95f95097ce7f41e5c45bf
    source_path: tools/code-mode.md
    workflow: 16
---

Codemodus is een experimentele, optionele functie van de OpenClaw-agentruntime. Wanneer
deze is ingeschakeld, ziet het model niet langer elk ingeschakeld toolschema; in plaats daarvan ziet het
`exec`, `wait` en elke alleen-directe tool waarvan het gestructureerde resultaat niet via
de uitsluitend voor JSON bestemde gastbridge kan worden doorgegeven. Het model schrijft een klein JavaScript- of TypeScript-
programma dat de verborgen toolcatalogus doorzoekt, beschrijft en aanroept.

Deze pagina documenteert de OpenClaw-codemodus, niet Codex Code Mode. De twee functies
hebben dezelfde naam en dezelfde namen voor besturingstools (`exec`, `wait`), maar het zijn
afzonderlijke implementaties:

- Codex Code Mode wordt uitgevoerd binnen de Codex-codeerharness. De tool `exec` is een
  tool met vrije grammatica: het model schrijft onbewerkte JavaScript-broncode (optioneel
  voorafgegaan door een `// @exec: {...}`-pragmaregel voor uitvoeringsopties), die
  wordt uitgevoerd in de in-process V8 Code Mode-runtime van Codex.
- De OpenClaw-codemodus wordt uitgevoerd in de generieke OpenClaw-agentruntime en is
  uitgeschakeld tenzij `tools.codeMode.enabled: true` is geconfigureerd. De tool `exec`
  accepteert een JSON-`{ code, language }`-payload, die wordt uitgevoerd in een QuickJS-WASI-
  worker.

Beide zijn uitvoeringsomgevingen voor JavaScript, geen uitvoeringsomgevingen voor shellopdrachten. Beschouw ze
als onafhankelijke, verschillend geïmplementeerde functies die toevallig
gelijknamige tools `exec`/`wait` aanbieden.

## Wat het doet

- De voor het model zichtbare toollijst wordt `exec`, `wait`, plus elke alleen-directe tool
  zoals `computer` of de native-vision-lader `image`, waarvan het afbeeldingsresultaat
  de gastbridge niet kan passeren.
- `exec` evalueert door het model gegenereerde JavaScript of TypeScript in een geïsoleerde
  QuickJS-WASI-workerthread.
- Elke ingeschakelde tool die geschikt is voor de catalogus (OpenClaw-core, Plugin, MCP, client) wordt verborgen als
  zelfstandige modeltool en binnen het gastprogramma beschikbaar gemaakt via `ALL_TOOLS`
  en `tools`.
- De beschrijving van `exec` bevat een begrensde snelle index met exacte OpenClaw-/Plugin-
  catalogus-id's, compacte invoerhints en compacte gedeclareerde uitvoerhints wanneer een
  vertrouwde tool een uitvoerschema biedt. Beschrijvingen, volledige schema's,
  MCP-vermeldingen en overloopvermeldingen worden weggelaten; cataloguszoekacties aan gastzijde blijven de terugvaloptie.
- Gastcode doorzoekt de verborgen catalogus, beschrijft het schema van een tool en roept
  een tool aan via hetzelfde uitvoeringspad dat wordt gebruikt door normale agentbeurten (beleid,
  goedkeuringen, hooks en telemetrie blijven allemaal van toepassing).
- MCP-tools worden gegroepeerd onder de naamruimte `MCP`; in codemodus is dit de
  enige ondersteunde manier om ze aan te roepen.
- `wait` hervat een onderbroken uitvoering in codemodus wanneer geneste toolaanroepen nog
  in behandeling zijn.

Codemodus wijzigt alleen het voor het model bestemde orkestratieoppervlak. Het vervangt geen
tools, Plugin-tools, MCP-tools, authenticatie, goedkeuringsbeleid, kanaalgedrag
of modelselectie.

## Waarom je het gebruikt

- Kleiner promptoppervlak: providers krijgen twee besturingstools, een begrensde index van native tools
  en alleen de weinige vereiste directe tools in plaats van tientallen of honderden
  volledige toolschema's.
- Betere orkestratie: het model kan lussen, samenvoegingen, kleine transformaties,
  voorwaardelijke logica en parallelle geneste toolaanroepen binnen één codecel gebruiken.
- Minder modelrondgangen: met een gedeclareerd uitvoercontract kan het model een toolresultaat
  in één `exec` aanroepen en transformeren; onbekende uitvoer blijft eerst onbewerkt.
- Providerneutraal: werkt voor OpenClaw-, Plugin-, MCP- en clienttools zonder
  afhankelijk te zijn van providereigen code-uitvoering.
- Stopt veilig: als codemodus is ingeschakeld maar de QuickJS-WASI-runtime
  niet beschikbaar is, mislukt de uitvoering in plaats van stilzwijgend terug te vallen op brede directe
  blootstelling van tools.

Dit is vooral nuttig voor agents met een grote ingeschakelde toolcatalogus, of voor workflows waarin
het model meerdere tools moet zoeken, combineren en aanroepen voordat het antwoord geeft.

Behoud directe blootstelling van tools voor een kleine catalogus of een model dat niet betrouwbaar
korte programma's schrijft. Gebruik [Tool zoeken](/nl/tools/tool-search) wanneer je een
compacte catalogus wilt, maar de voorkeur geeft aan gestructureerde besturing voor zoeken/beschrijven/aanroepen in plaats van
de QuickJS-WASI-gast.

## Snelstart

### Codemodus inschakelen

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

Verkorte vorm:

```json5
{
  tools: {
    codeMode: true,
  },
}
```

Codemodus blijft uit wanneer `tools.codeMode` is weggelaten, `false`, of een object
zonder `enabled: true`.

Als je agents in een sandbox gebruikt met geconfigureerde MCP-servers, sta dan ook de
meegeleverde MCP-Plugin toe in het sandbox-toolbeleid, bijvoorbeeld
`tools.sandbox.tools.alsoAllow: ["bundle-mcp"]`. Zie
[Configuratie - tools en aangepaste providers](/nl/gateway/config-tools#mcp-and-plugin-tools-inside-sandbox-tool-policy).

Stel expliciete limieten in voor strengere begrenzingen:

```json5
{
  tools: {
    codeMode: {
      enabled: true,
      timeoutMs: 10000,
      memoryLimitBytes: 67108864,
      maxOutputBytes: 65536,
      maxSnapshotBytes: 10485760,
      maxPendingToolCalls: 16,
      snapshotTtlSeconds: 900,
      searchDefaultLimit: 8,
      maxSearchLimit: 50,
    },
  },
}
```

### Wat het model doet

Voor een tool met gedeclareerde uitvoer, zoals
`Array<{ id: string; paid: boolean; tons: number }>`, kan één gastprogramma
deze selecteren, aanroepen en transformeren:

```javascript
const [shipmentTool] = await tools.search("list shipments");
const shipments = await tools.callValue(shipmentTool.id, {});
return shipments.filter((shipment) => !shipment.paid && shipment.tons > 10);
```

Wanneer een regel in de snelle index eindigt op `-> ?`, is de uitvoervorm onbekend. De eerste
`exec` moet `await tools.callValue(...)` ongewijzigd retourneren. Een latere `exec` kan
de waargenomen waarde transformeren. Dit kost een extra modelbeurt, maar voorkomt dat het
model veldnamen raadt.

### Het actieve oppervlak verifiëren

Om tijdens het debuggen de vorm van de modelpayload te bevestigen, voer je de Gateway uit met
gerichte logging:

```bash
OPENCLAW_DEBUG_CODE_MODE=1 \
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 \
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools \
openclaw gateway
```

Als codemodus actief is, moeten de gelogde namen van de voor het model zichtbare tools `exec` en
`wait` zijn. Voeg voor de volledige geredigeerde providerpayload
`OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted` toe tijdens een korte debugsessie.

## Swarm gebruiken voor uitwaaiering van agents

[Swarm](/tools/swarm) voegt de globale gastvariabelen `agents.run()`, `phase()` en `log()` toe
voor het orkestreren van gelijktijdige subagents vanuit scripts in codemodus. Schakel zowel
`tools.codeMode` als `tools.swarm` in en gebruik vervolgens normale JavaScript-besturingsstromen voor
uitwaaiering, beslissingspoorten en gestructureerde verzameling. Swarm is een afzonderlijke optionele
poort; alleen Codemodus inschakelen stelt de API `agents.*` niet beschikbaar.

## Technische rondleiding

De rest van deze pagina behandelt het runtimecontract en de implementatiedetails,
voor beheerders, Plugin-auteurs die de blootstelling van tools debuggen en operators
die implementaties met een hoog risico valideren.

## Runtimestatus

|                     |                                                                                             |
| ------------------- | ------------------------------------------------------------------------------------------- |
| Runtime             | [`quickjs-wasi`](https://github.com/vercel-labs/quickjs-wasi)                               |
| Standaardstatus     | uitgeschakeld                                                                               |
| Stabiliteit         | experimenteel OpenClaw-oppervlak (Codex Code Mode is een afzonderlijk, stabiel Codex-harnessoppervlak) |
| Doeloppervlak       | generieke uitvoeringen van OpenClaw-agents                                                  |
| Beveiligingshouding | modelcode is vijandig                                                                       |
| Gebruikersbelofte   | codemodus inschakelen valt nooit stilzwijgend terug op brede directe blootstelling van tools |

## Reikwijdte

Codemodus beheert de voor het model bestemde orkestratievorm voor een voorbereide uitvoering. De modus
beheert niet de modelselectie, het kanaalgedrag, de authenticatie, het toolbeleid of de tool-
implementaties.

Binnen de reikwijdte: definities van voor het model zichtbare besturings-/directe tools, constructie van de verborgen toolcatalogus,
uitvoering van JavaScript/TypeScript door de gast, de QuickJS-WASI-worker-
runtime, hostcallbacks voor zoeken/beschrijven/aanroepen, hervatbare status voor
onderbroken gastprogramma's, limieten voor uitvoer/time-out/geheugen/openstaande aanroepen/snapshots
en projectie van telemetrie/traject voor geneste toolaanroepen.

Buiten de reikwijdte: providereigen uitvoering van externe code, semantiek voor uitvoering
van shellopdrachten, wijziging van bestaande toolautorisatie, permanente door gebruikers geschreven
scripts, toegang tot pakketbeheer/bestanden/netwerk/modules in gastcode en direct
hergebruik van interne onderdelen van Codex Code Mode.

Tools die eigendom zijn van providers, zoals externe Python-sandboxes, zijn afzonderlijke tools. Zie
[Code-uitvoering](/nl/tools/code-execution).

## Termen

- **Codemodus**: de OpenClaw-runtimemodus die cataloguscompatibele modeltools
  verbergt en `exec`, `wait` plus vereiste alleen-directe tools beschikbaar stelt.
- **Gastruntime**: de QuickJS-WASI JavaScript-VM die modelcode evalueert.
- **Hostbridge**: het beperkte JSON-compatibele callbackoppervlak van gastcode
  terug naar OpenClaw.
- **Catalogus**: de uitvoeringsgebonden lijst van effectieve tools na normale oplossing van
  toolbeleid, Plugins, MCP en clienttools.
- **Geneste toolaanroep**: een toolaanroep die vanuit gastcode via de hostbridge
  wordt uitgevoerd.
- **Snapshot**: geserialiseerde QuickJS-WASI-VM-status die wordt opgeslagen zodat `wait`
  een onderbroken uitvoering in codemodus kan voortzetten.

## Configuratie

`tools.codeMode.enabled` is de activeringspoort; het instellen van andere velden
schakelt de functie niet zelfstandig in.

| Veld                  | Standaardwaarde                 | Begrenzing                                      |
| --------------------- | ------------------------------ | ----------------------------------------------- |
| `enabled`             | `false`                        | booleaans; alleen `true` schakelt codemodus in |
| `runtime`             | `"quickjs-wasi"`               | enige ondersteunde waarde                       |
| `mode`                | `"only"`                       | stelt besturings-/directe tools beschikbaar en catalogiseert de rest |
| `languages`           | `["javascript", "typescript"]` | elke deelverzameling van de twee                |
| `timeoutMs`           | `10000`                        | `100`-`60000`                                   |
| `memoryLimitBytes`    | `67108864`                     | `1048576`-`1073741824`                          |
| `maxOutputBytes`      | `65536`                        | `1024`-`10485760`                               |
| `maxSnapshotBytes`    | `10485760`                     | `1024`-`268435456`                              |
| `maxPendingToolCalls` | `16`                           | `1`-`128`                                       |
| `snapshotTtlSeconds`  | `900`                          | `1`-`86400`                                     |
| `searchDefaultLimit`  | `8`                            | begrensd tot `maxSearchLimit`                   |
| `maxSearchLimit`      | `50`                           | `1`-`50`                                        |

Als codemodus is ingeschakeld maar QuickJS-WASI niet kan worden geladen, stopt OpenClaw veilig
voor die uitvoering; het stelt normale tools niet stilzwijgend als terugvaloptie beschikbaar.

## Activering

Codemodus wordt geëvalueerd nadat het effectieve toolbeleid bekend is en voordat het
definitieve modelverzoek wordt samengesteld:

1. Bepaal het beleid voor de agent, het model, de provider, de sandbox, het kanaal, de afzender en de run.
2. Bouw de effectieve OpenClaw-toollijst en voeg in aanmerking komende plugin-, MCP- en clienttools toe.
3. Pas het toestaan/weigeren-beleid toe.
4. Als `tools.codeMode.enabled` onwaar is, ga je door met de normale beschikbaarstelling van tools.
5. Als dit is ingeschakeld en er tools actief zijn voor de run, behoud je de vereiste tools die alleen rechtstreeks beschikbaar zijn en registreer je elke effectieve tool die voor de catalogus in aanmerking komt in de code-modecatalogus.
6. Verwijder de gecatalogiseerde tools uit de voor het model zichtbare lijst; voeg `exec` en `wait` toe naast de behouden tools die alleen rechtstreeks beschikbaar zijn.

Runs die opzettelijk geen tools hebben (onbewerkte modelaanroepen, `disableTools: true`
of een lege `tools.allow`-lijst), activeren het code-modeoppervlak niet, zelfs
wanneer `tools.codeMode.enabled: true` is geconfigureerd. Code mode en OpenClaw Tool
Search sluiten elkaar uit voor een run; als code mode wordt geactiveerd, vindt de
compaction van Tool Search niet plaats.

De code-modecatalogus is beperkt tot de run en mag geen tools uit een andere
agent, sessie, afzender of run lekken.

## Voor het model zichtbare tools

Wanneer code mode actief is, ziet het model `exec`, `wait` en alle vereiste
tools die alleen rechtstreeks beschikbaar zijn. Elke andere ingeschakelde tool wordt verborgen in de
voor het model zichtbare toollijst en geregistreerd in de code-modecatalogus.

Gebruik `exec` voor toolorkestratie, het samenvoegen van gegevens, lussen, parallelle geneste aanroepen
en gestructureerde transformaties. Gebruik `wait` alleen wanneer `exec` een hervatbaar
`waiting`-resultaat retourneert.

## `exec`

`exec` start een code-modecel en retourneert één resultaat. Invoercode wordt door het model
gegenereerd en moet als vijandig worden behandeld.

Invoer:

```typescript
type CodeModeExecInput = {
  code?: string;
  command?: string;
  language?: "javascript" | "typescript";
};
```

Regels:

- Een van `code` of `command` mag niet leeg zijn.
- `code` is het gedocumenteerde veld dat voor het model zichtbaar is.
- `command` wordt geaccepteerd als een exec-compatibele alias voor hookbeleid en
  vertrouwde herschrijvingen (de normale OpenClaw-shell-exectool gebruikt ook een `command`-
  veld); wanneer beide aanwezig zijn, moeten de waarden overeenkomen.
- `language` is standaard `"javascript"`; het schema stelt dit beschikbaar als een platte
  string-enum (`"javascript" | "typescript"`), niet als een `oneOf`/`anyOf`-union,
  omdat sommige providers die vormen weigeren.
- Als `language` `"typescript"` is, transpileert OpenClaw vóór de evaluatie.
- `exec` weigert `import`, `require`, dynamische import en moduleloaderpatronen.
- `exec` stelt de normale shellimplementatie van `exec` nooit recursief beschikbaar.
- Buitenste code-mode-`exec`-hookgebeurtenissen bevatten `toolKind: "code_mode_exec"` en
  `toolInputKind: "javascript" | "typescript"` (indien bekend), zodat beleid
  code-modecellen kan onderscheiden van shellachtige `exec`-aanroepen met dezelfde
  toolnaam.

Resultaat:

```typescript
type CodeModeResult = CodeModeCompletedResult | CodeModeWaitingResult | CodeModeFailedResult;

type CodeModeCompletedResult = {
  status: "completed";
  value: unknown;
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};

type CodeModeWaitingResult = {
  status: "waiting";
  runId: string;
  reason: "pending_tools" | "yield";
  pendingToolCalls?: CodeModePendingToolCall[];
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};

type CodeModeFailedResult = {
  status: "failed";
  error: string;
  code?: CodeModeErrorCode;
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};
```

`exec` retourneert `waiting` wanneer de guest wordt onderbroken met hervatbare status die nog
een voor het model zichtbare voortzetting vereist — een expliciete `yield_control(...)` of een
bridge-toolaanroep die niet binnen de exec-deadline is voltooid. Het resultaat
bevat een `runId` voor `wait`. Bridge-toolaanroepen — `tools.search`/`describe`/
`call` en namespace-aanroepen, waaronder MCP-namespace-aanroepen — worden automatisch afgehandeld
binnen dezelfde `exec`/`wait`-aanroep zolang ze binnen de deadline worden voltooid, zodat een
compact codeblok dat meerdere tools afwacht in één modelbeurt wordt voltooid
in plaats van voor elke await één modeltoolaanroep af te dwingen. Herstartveilige runs worden nooit
automatisch afgehandeld; hun werk in behandeling doorloopt nog steeds de controles voor veilig opnieuw afspelen.

`exec` retourneert alleen `completed` wanneer de guest-VM geen werk in behandeling heeft en de
uiteindelijke waarde JSON-compatibel is nadat de uitvoeradapter van OpenClaw is uitgevoerd.

## `wait`

`wait` zet een onderbroken code-mode-VM voort.

Invoer:

```typescript
type CodeModeWaitInput = {
  runId: string;
};
```

De uitvoer is dezelfde `CodeModeResult`-union die door `exec` wordt geretourneerd.

`wait` bestaat omdat geneste OpenClaw-tools traag, interactief of onderworpen aan
goedkeuring kunnen zijn, of gedeeltelijke updates kunnen streamen; het model hoeft niet één lange
`exec`-aanroep open te houden terwijl de host op extern werk wacht.

QuickJS-WASI-snapshot/herstel is het hervattingsmechanisme:

1. `exec` evalueert code totdat deze is voltooid, mislukt of wordt onderbroken.
2. Bij onderbreking maakt OpenClaw een snapshot van de QuickJS-VM en registreert het
   hostwerk in behandeling.
3. Wanneer het werk in behandeling is afgehandeld, herstelt `wait` de VM-snapshot en
   registreert het hostcallbacks opnieuw met stabiele namen.
4. OpenClaw levert geneste toolresultaten aan de herstelde VM en handelt
   QuickJS-taken in behandeling af.
5. `wait` retourneert `completed`, `failed` of nog een `waiting`-resultaat.

Snapshots zijn runtimestatus, geen gebruikersartefacten: ze bestaan alleen in een
kaart in het proces (zonder schrijfopdracht naar database of schijf), hebben een maximale grootte, verlopen en zijn
beperkt tot de run en sessie waardoor ze zijn gemaakt.

`wait` mislukt (als een `failed`-resultaat) wanneer:

- `runId` onbekend is of de snapshot ervan al is verlopen.
- de aanroeper zich niet binnen hetzelfde run-/sessiebereik bevindt als de onderbroken run.
- er al een `wait` wordt uitgevoerd voor die `runId`.
- het herstel van QuickJS-WASI mislukt.
- hervatten `maxOutputBytes` of `maxSnapshotBytes` zou overschrijden.

## Guest-runtime-API

```typescript
declare const ALL_TOOLS: ToolCatalogEntry[];
declare const tools: ToolCatalog;
declare const MCP: Record<string, unknown>;
declare const namespaces: Record<string, unknown>;

declare function text(value: unknown): void;
declare function json(value: unknown): void;
declare function yield_control(reason?: string): Promise<void>;
```

`ALL_TOOLS` is compacte metadata voor de tot de run beperkte catalogus; deze bevat standaard
geen volledige schema's. De voor het model zichtbare beschrijving van `exec` bevat ook een
begrensde, deterministische subset van exacte OpenClaw-/plugin-id's, compacte invoerhints
en vertrouwde gedeclareerde uitvoerhints. Beschrijvingen blijven uitgesteld, zodat
vijandige catalogusproza het model niet kan sturen. Wanneer die index een tool weglaat,
lees je `ALL_TOOLS` of roep je `tools.search(...)` aan binnen het guestprogramma.

De pijl in elke snelindexregel beschrijft de waarde van `tools.callValue(...)`.
`-> Array<{ id: string }>` is een gedeclareerde uitvoerhint; `-> ?` betekent dat de uitvoer onbekend is.
Onbekende uitvoer blijft eerst onbewerkt: retourneer de waarde ongewijzigd, bekijk deze en
filter of transformeer deze vervolgens in een latere `exec` in plaats van veldnamen te raden. Dit is ook
van toepassing wanneer het lezen van gedeclareerde uitvoer als invoer voor een laatste `-> ?`-aanroep dient: retourneer de
onbewerkte waarde van die aanroep zonder deze in de gevraagde antwoordvorm te verpakken.

```typescript
type ToolCatalogEntry = {
  id: string;
  name: string;
  label?: string;
  description: string;
  source: "openclaw" | "mcp" | "client";
  sourceName?: string;
  input: string;
  output?: string;
};
```

`input` is een begrensde TypeScript-achtige signatuur voor het gangbare geval. Gebruik
`tools.describe(...)` wanneer het exacte volledige schema nog nodig is. Externe MCP-
en clientvermeldingen gebruiken `input: "unknown"`, zodat hun niet-vertrouwde schema's
uitgesteld blijven tot `describe`. `output` is
alleen aanwezig voor een volledige compacte hint die is afgeleid van een vertrouwde OpenClaw-core
of plugin-`outputSchema`. Claims over uitvoerschema's van MCP en clients worden niet
opgenomen in deze vertrouwde catalogushint.

Plugintools gebruiken `source: "openclaw"` waarbij `sourceName` is ingesteld op de id van de eigenaarplugin;
er is geen afzonderlijke `"plugin"`-bronwaarde. `source: "mcp"` wordt
alleen gebruikt voor MCP-vermeldingen in `sourceName`/`mcp`-metadata (en wordt uitgefilterd
uit `ALL_TOOLS`/`tools.*`, zie hieronder).

Het volledige schema wordt alleen op aanvraag geladen:

```typescript
type ToolCatalogEntryWithSchema = ToolCatalogEntry & {
  parameters: unknown;
  outputSchema?: unknown;
};
```

Catalogushulpfuncties:

```typescript
type ToolCatalog = {
  search(query: string, options?: { limit?: number }): Promise<ToolCatalogEntry[]>;
  describe(id: string): Promise<ToolCatalogEntryWithSchema>;
  callValue(id: string, input?: unknown): Promise<unknown>;
  call(id: string, input?: unknown): Promise<unknown>;
  [safeToolName: string]: unknown;
};
```

Gemaksfuncties voor tools worden alleen geïnstalleerd voor ondubbelzinnige veilige namen:

```typescript
const files = await tools.search("read local file");
const fileRead = await tools.describe(files[0].id);
const content = await tools.callValue(fileRead.id, { path: "README.md" });

// Als de verborgen catalogus een ondubbelzinnige `web_search`-vermelding heeft:
const hits = await tools.web_search({ query: "OpenClaw code mode" });
```

`tools.callValue(...)` retourneert de JSON-waarde `details` van een normale tool rechtstreeks.
`tools.call(...)` behoudt de onbewerkte `{ tool, result }`-envelop voor aanroepers
die contentblokken of andere resultaatmetadata nodig hebben.

## Gedeclareerde uitvoercontracten

OpenClaw-tools kunnen `outputSchema` declareren voor de gestructureerde waarde die in
`AgentToolResult.details` wordt geplaatst. Dit is nuttig voor Code Mode en Tool Search; het is
geen provider-native toolresponsschema en verandert de rechtstreekse
beschikbaarstelling van tools niet.

Voor een tool die met `defineToolPlugin` is gemaakt, declareer je het schema naast
`parameters`:

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

const Shipment = Type.Object(
  {
    id: Type.String(),
    paid: Type.Boolean(),
    tons: Type.Number(),
  },
  { additionalProperties: false },
);

export default defineToolPlugin({
  id: "shipping",
  name: "Shipping",
  description: "Shipment tools.",
  tools: (tool) => [
    tool({
      name: "shipping_list",
      description: "List shipments.",
      parameters: Type.Object({}),
      outputSchema: Type.Array(Shipment),
      execute: async () => loadShipments(),
    }),
  ],
});
```

Voor `api.registerTool(...)` of een fabriekstool plaats je dezelfde `outputSchema`-
eigenschap op het geretourneerde `AnyAgentTool`-object.

Huidige ingebouwde contracten omvatten `agents_list`, `apply_patch`,
`conversations_list`, `conversations_send`, `conversations_turn`, `edit`,
`openclaw`, `read`, `screen`,
`sessions_history`, `sessions_list`, `sessions_search`, `sessions_send`,
`session_status`, `spawn_task`, `terminal`, `web_fetch` en `web_search`.
Exacte doorvoerbewerkingen kunnen hun bijbehorende protocolschema hergebruiken in plaats van
een contract uitsluitend voor het model te dupliceren. De gesprekstools stellen bijvoorbeeld
dezelfde Gateway-resultaatschema's beschikbaar die worden gebruikt door `conversations.list`,
`conversations.send` en `conversations.turn`; `web_fetch` beheert een toollokaal
schema waarvan de hint stabiele metagegevens, tekst, cachestatus en geneste
overloopmetagegevens beschikbaar stelt; `web_search` declareert de exacte unie van
genormaliseerde resultaten/antwoorden/fouten/onbewerkte gegevens als een volledige snelle-indexhint.
Bestandssysteemcontracten retourneren gestructureerde uitkomsten voor gelezen tekst,
afbeeldingen, afkapping en optioneel niet-gevonden; expliciete status van
bewerkingswijzigingen plus diff-/patchgegevens; en padoverzichten voor het toepassen van patches.
Wanneer de snelle index de velden declareert, kan één cel detectie en aflevering combineren
zonder een afzonderlijke inspectiebeurt:

```javascript
const listed = await tools.conversations_list({ query: "build bot" });
const target = listed.conversations.find((item) => item.label === "Build bot");
if (!target) throw new Error("conversation not found");
return await tools.conversations_send({
  conversationRef: target.conversationRef,
  message: "Build finished.",
});
```

De geneste aanroepen gebruiken nog steeds het normale toolbeleid, hooks en goedkeuringen. Als een
volledig contract exact maar te groot is voor de begrensde snelle index, blijft het
beschikbaar via `tools.describe(...)` en blijft de pijl `-> ?`.

De contractregels zijn strikt:

- Beschrijf de exacte JSON-compatibele waarde `details`, niet gerenderde
`content`-blokken of een provider-envelop.
- Neem elke niet-werpende succes- of foutvariant op. Laat `outputSchema` weg wanneer
  de tool geen stabiel gestructureerd resultaat heeft.
- Sluit objectlagen af met `{ additionalProperties: false }` voor een volledige
  snelle-indexhint. Open, te grote of anderszins gedeeltelijke schema's blijven
  beschikbaar via `tools.describe(...)`, maar maken veldgebruik in één beurt niet mogelijk.
- OpenClaw compileert het schema voordat de tool wordt uitgevoerd en valideert vervolgens
  de definitieve `details` na de normale toolhooks en voordat een catalogusaanroep
  retourneert. Met een ongeldig schema kan de tool niet worden uitgevoerd; bij een
  afwijking mislukt de uitvoering zonder de waarde af te drukken.
- Compacte hints zijn deterministisch en begrensd. `tools.describe(...)` stelt
  het volledige vertrouwde schema beschikbaar wanneer de compacte hint niet volstaat.
- Geïnstalleerde plugincode is al vertrouwde lokale code. Externe MCP- en
  clientmetagegevens blijven onvertrouwd en kunnen deze snelle-indexhints niet inschakelen.

Zie [Toolplugins](/nl/plugins/tool-plugins#output-contracts) voor informatie over
het ontwikkelen van plugins.

MCP-catalogusitems kunnen niet worden aangeroepen via `tools.callValue(...)`,
`tools.call(...)` of gemaksfuncties in codemodus; ze worden uitsluitend
beschikbaar gesteld via de gegenereerde naamruimte `MCP`. Declaratiebestanden
in TypeScript-stijl zijn beschikbaar via het alleen-lezen virtuele bestandsoppervlak
`API`, zodat agents MCP-signaturen kunnen inspecteren zonder MCP-schema's
aan de prompt toe te voegen:

```typescript
const files = await API.list("mcp");
const githubApi = await API.read("mcp/github.d.ts");

const issue = await MCP.github.createIssue({
  owner: "openclaw",
  repo: "openclaw",
  title: "Investigate gateway logs",
});

const snapshot = await MCP.chromeDevtools.takeSnapshot({ output: "markdown" });
const resource = await MCP.docs.resources.read({ uri: "memo://one" });
const prompt = await MCP.docs.prompts.get({
  name: "brief",
  arguments: { topic: "release" },
});
```

`API.read("mcp/<server>.d.ts")` retourneert compacte declaraties die zijn afgeleid van
MCP-toolmetagegevens:

```typescript
type McpToolResult = {
  content?: unknown[];
  structuredContent?: unknown;
  isError?: boolean;
  [key: string]: unknown;
};

declare namespace MCP.github {
  /** Retourneer deze API-header in TypeScript-stijl. */
  function $api(toolName?: string, options?: { schema?: boolean }): Promise<McpApiHeader>;

  /**
   * Maak een GitHub-issue.
   * @param owner Eigenaar van de repository
   * @param repo Naam van de repository
   * @param title Titel van het issue
   */
  function createIssue(input: {
    owner: string;
    repo: string;
    title: string;
    body?: string;
  }): Promise<McpToolResult>;
}
```

Declaratiebestanden zijn virtueel en worden niet onder de werkruimte- of
statusmap geschreven. Voor elke aanroep van `exec` in codemodus bouwt
OpenClaw de toolcatalogus voor de betreffende uitvoering, behoudt het de zichtbare
MCP-items, rendert het `mcp/index.d.ts` plus één `mcp/<server>.d.ts` per zichtbare
server en injecteert het die kleine alleen-lezen tabel in de QuickJS-worker.
Gastcode ziet uitsluitend het object `API`:
`API.list(prefix?)` retourneert bestandsmetagegevens en `API.read(path)`
retourneert de inhoud van de geselecteerde declaratie. Onbekende paden en
`.`- en `..`-segmenten worden geweigerd.

Hierdoor blijven grote MCP-schema's buiten de modelprompt: de agent leert via de
toolbeschrijving `exec` dat de virtuele API bestaat, leest alleen het
benodigde declaratiebestand en roept vervolgens `MCP.<server>.<tool>()` aan met één
objectargument. `MCP.<server>.$api()` blijft beschikbaar als inline terugvaloptie
voor een schemarespons van één tool binnen het programma.

De gast-runtime ziet nooit rechtstreeks hostobjecten. Invoer en uitvoer gaan
over de bridge als JSON-compatibele waarden met expliciete groottelimieten.

## Interne naamruimten

Interne naamruimten geven de codemodus een beknopte domein-API zonder meer
voor het model zichtbare tools toe te voegen. Een door de loader beheerde integratie
registreert een naamruimte zoals `Issues` of `Calendar`;
gastcode roept die naamruimte vervolgens aan binnen het QuickJS-programma, terwijl
het model nog steeds het compacte besturings-/directe oppervlak ziet.

Naamruimten zijn voorlopig intern. Er is geen openbare naamruimte-API voor de plugin-SDK:
externe pluginnaamruimten hebben een door de loader beheerd contract nodig, zodat
pluginidentiteit, geïnstalleerde manifesten, authenticatiestatus en gecachte
catalogusdescriptors niet kunnen afwijken van de plugintools waarop de naamruimte
is gebaseerd. De codemodus van de kern beheert uitsluitend de sandbox, serialisatie,
catalogusbeperking en bridgedispatch.

Gastcode kan zowel de directe globale variabele als de map `namespaces` gebruiken:

```javascript
const open = await Issues.list({ state: "open" });
const alsoOpen = await namespaces.Issues.list({ state: "open" });
return { count: open.length, alsoCount: alsoOpen.length };
```

### Levenscyclus van het register

Het naamruimteregister is proceslokaal en gebruikt de naamruimte-id als sleutel:

1. Een vertrouwde loader roept `registerCodeModeNamespaceForPlugin(pluginId, registration)` aan.
2. De codemodus maakt de verborgen `ToolSearchRuntime` voor de uitvoering
   en leest de catalogus voor die uitvoering.
3. `createCodeModeNamespaceRuntime(ctx, catalog)` behoudt alleen registraties
   waarvan alle `requiredToolNames` zichtbaar zijn en eigendom zijn van dezelfde `pluginId`.
4. Elke zichtbare naamruimte roept `createScope(ctx)` aan voor de huidige
   uitvoering en ontvangt uitvoeringscontext zoals `agentId`, `sessionKey`,
   `sessionId`, `runId`, configuratie en afbreekstatus.
5. Bereikgegevens worden geserialiseerd naar een gewone descriptor en in QuickJS
   geïnjecteerd als directe globale variabelen en `namespaces.<globalName>`.
6. Gastaanroepen worden via de workerbridge onderbroken, lossen het naamruimtepad
   op de host op, koppelen de aanroep aan een gedeclareerde catalogustool die eigendom is
   van de plugin en voeren die tool uit via `ToolSearchRuntime.callExactId`.
7. Gereedstaande bridgeaanroepen voor naamruimten worden automatisch afgehandeld binnen
   de actieve aanroep `exec`/`wait`; als er bij de time-out nog
   naamruimtewerk in behandeling is of de gast expliciet de uitvoering overdraagt, hervat
   `wait` later dezelfde naamruimte-runtime.
8. Bij terugdraaien of verwijderen van een plugin wordt
   `clearCodeModeNamespacesForPlugin(pluginId)` aangeroepen, zodat verouderde globale variabelen
   een mislukte pluginlaadbewerking niet overleven.

Naamruimteaanroepen zijn catalogustoolaanroepen: ze gebruiken dezelfde beleidshooks,
goedkeuringen, afbreekafhandeling, telemetrie, transcriptprojectie en hetzelfde
onderbrekings-/hervattingsgedrag als `tools.call(...)`.

### Registratiestructuur

Registreer naamruimten vanuit de integratie die eigenaar is van de onderliggende tools.
Houd het bereik klein en stel alleen domeinwerkwoorden beschikbaar die aan gedeclareerde
catalogustools zijn gekoppeld.

```typescript
import {
  createCodeModeNamespaceTool,
  registerCodeModeNamespaceForPlugin,
} from "../agents/code-mode-namespaces.js";

const pluginId = "github";

registerCodeModeNamespaceForPlugin(pluginId, {
  id: "github-issues",
  globalName: "Issues",
  description: "GitHub-issuehulpmiddelen voor de huidige repository.",
  requiredToolNames: ["github_list_issues", "github_update_issue"],
  prompt: "Gebruik Issues.list(params) en Issues.update(number, patch).",
  createScope: (ctx) => ({
    repository: ctx.config,
    list: createCodeModeNamespaceTool("github_list_issues", ([params]) => params ?? {}),
    update: createCodeModeNamespaceTool("github_update_issue", ([number, patch]) => ({
      number,
      patch,
    })),
  }),
});
```

`createCodeModeNamespaceTool(toolName, inputMapper)` markeert een bereiklid als een
aanroepbare naamruimtefunctie. De optionele `inputMapper` ontvangt de
gastargumenten en retourneert het invoerobject voor de onderliggende catalogustool;
zonder deze functie wordt het eerste gastargument gebruikt, of `{}`
als dit is weggelaten.

Onbewerkte hostfuncties worden geweigerd voordat gastcode wordt uitgevoerd:

```typescript
createScope: () => ({
  // Fout: dit omzeilt de levenscyclus van de catalogustool en wordt geweigerd.
  list: async () => githubClient.listIssues(),
});
```

### Eigendom en zichtbaarheid

Het eigendom van een naamruimte is gekoppeld aan `pluginId` van de
registrerende aanroeper. `requiredToolNames` is zowel een zichtbaarheidsbeperking
als een eigendomscontrole:

- elke vereiste tool moet in de uitvoeringscatalogus bestaan
- elke vereiste tool moet `sourceName === pluginId` hebben
- de naamruimte wordt verborgen wanneer een vereiste tool ontbreekt of
  eigendom is van een andere plugin
- elk aanroepbaar pad mag alleen een tool aanroepen die in
  `requiredToolNames` wordt genoemd

Dit voorkomt dat een andere plugin een naamruimte beschikbaar stelt door een tool
met dezelfde naam te registreren en houdt naamruimten in overeenstemming met het
reguliere agentbeleid: als de uitvoering de onderliggende tools niet kan zien,
kan deze ook de naamruimte niet zien.

Een GitHub-naamruimte hoort bijvoorbeeld achter een plugin te staan die eigendom is
van GitHub en die GitHub-authenticatie, REST-/GraphQL-clients, frequentielimieten,
schrijfgoedkeuringen en tests beheert. De codemodus van de kern mag geen
GitHub-specifieke API's, tokenafhandeling of providerbeleid insluiten.

### Serialisatieregels voor bereiken

`createScope(ctx)` mag een gewoon object retourneren dat JSON-compatibele
waarden, arrays, geneste objecten en aanroepmarkeringen van `createCodeModeNamespaceTool(...)`
bevat. Hostobjecten komen nooit rechtstreeks in QuickJS terecht.

De serializer weigert:

- onbewerkte functies
- cyclische objectgrafen
- onveilige padsegmenten: `__proto__`, `constructor`,
`prototype`, lege sleutels of sleutels die het interne padscheidingsteken bevatten
- `globalName`-waarden die geen JavaScript-identificatoren zijn
- `globalName`-conflicten met ingebouwde globale variabelen van de
  codemodus, zoals `tools`, `namespaces`, `text`,
  `json`, `yield_control`, `MCP`, `API`,
  `ALL_TOOLS` of `__openclaw*`

Waarden die niet naar JSON kunnen worden geserialiseerd, worden omgezet in
JSON-veilige terugvalwaarden voordat ze de bridge passeren. Binaire gegevens,
handles, sockets, clients en klasse-instanties moeten achter reguliere
catalogustools blijven.

### Prompts

De naamruimte-`description` en optionele `prompt` worden alleen aan het
voor het model zichtbare `exec`-schema toegevoegd wanneer de naamruimte voor
die uitvoering zichtbaar is. Gebruik ze om het kleinst bruikbare oppervlak aan te leren:

```typescript
{
  description: "Helpers voor de fictieproductieservice.",
  prompt:
    "Gebruik Fictions.riskAudit(), Fictions.promoteIfReady(id, status) en Fictions.unpaidOver(amount).",
}
```

Laat prompts gaan over het namespacecontract, niet over de authenticatie-instelling, implementatiegeschiedenis of niet-gerelateerd Plugingedrag.

### Opschoning

Namespaces zijn proceslokale registraties. Verwijder ze wanneer de bezittende
Plugin wordt uitgeschakeld, verwijderd of teruggedraaid:

```typescript
clearCodeModeNamespacesForPlugin(pluginId);
```

Opschoning in codemodus is eigendom van de Plugin; wis de namespaceregistraties
van de Plugin wanneer de levenscyclus ervan eindigt, in plaats van afhandelingshandles
per namespace te bewaren. Tests kunnen `clearCodeModeNamespacesForTest()` aanroepen om te voorkomen dat
registraties tussen testgevallen lekken.

### Testchecklist

Namespacewijzigingen moeten de beveiligingsgrens en het gastgedrag afdekken:

- tekst van de namespaceprompt verschijnt alleen wanneer de achterliggende tools zichtbaar zijn
- gelijknamige tools van een andere `sourceName` stellen de namespace niet beschikbaar
- onbewerkte bereikfuncties worden geweigerd
- vervalste namespace-id's en vervalste paden worden geweigerd
- aanroepbare paden kunnen niet verwijzen naar niet-gedeclareerde tools
- geneste objecten en gedeelde verwijzingen worden correct geserialiseerd
- namespaceaanroepen worden via catalogustools uitgevoerd en retourneren JSON-veilige details
- gastcode kan fouten afvangen
- opgeschorte namespaceaanroepen worden hervat via `wait`
- het terugdraaien van een Plugin wist de bijbehorende namespaceregistraties

Namespaces vullen de algemene `tools.search`/`tools.call`-catalogus aan: gebruik de
catalogus voor willekeurige ingeschakelde tools van OpenClaw, Plugins en clients; gebruik `MCP`
voor MCP-tools; gebruik andere namespaces voor gedocumenteerde domein-API's waarvan
de Plugin eigenaar is, waar beknopte code betrouwbaarder is dan herhaalde schemazoekopdrachten.

## Uitvoer-API

- `text(value)` voegt voor mensen leesbare uitvoer toe aan de array `output`.
- `json(value)` voegt na JSON-compatibele serialisatie een gestructureerd
  uitvoeritem toe.
- De uiteindelijke geretourneerde waarde van de gastcode wordt `value` in een
  `completed`-resultaat.

```typescript
type CodeModeOutput = { type: "text"; text: string } | { type: "json"; value: unknown };
```

Regels: de uitvoervolgorde komt overeen met de aanroepen van de gast; de uitvoer wordt begrensd door
`maxOutputBytes`; niet-serialiseerbare waarden worden geconverteerd naar gewone tekenreeksen of
fouten; binaire waarden worden niet ondersteund. Afbeeldingen en bestanden worden via
gewone OpenClaw-tools verzonden, niet via de codemodusbrug.

## Toolcatalogus

De verborgen catalogus bevat tools na effectieve beleidsfiltering, in deze
volgorde: kerntools van OpenClaw, meegeleverde Plugintools, externe Plugintools, MCP-
tools en vervolgens door de client geleverde tools voor de huidige uitvoering.

Catalogus-id's zijn stabiel binnen één uitvoering en waar mogelijk deterministisch
voor gelijkwaardige toolsets. Werkelijke vorm:

```text
<source>:<owner>:<tool-name>
```

waarbij `<source>` gelijk is aan `openclaw`, `mcp` of `client` (Plugintools gebruiken
`openclaw` met de Plugin-id als `<owner>`; kerntools gebruiken `openclaw:core:*`).
Voorbeelden:

```text
openclaw:core:message
openclaw:browser:browser_request
mcp:github:create_issue
client:app:select_file
```

De catalogus laat besturingstools voor de codemodus weg (`exec`, `wait`, `tool_search_code`,
`tool_search`, `tool_describe`, `tool_call`) en tools die alleen rechtstreeks beschikbaar zijn. Besturingselementen
mogen niet recursief via de catalogus worden aangeroepen; tools die alleen rechtstreeks beschikbaar zijn, blijven zichtbaar voor het model
omdat hun gestructureerde resultaten de QuickJS-brug niet kunnen passeren.

MCP-vermeldingen blijven in de catalogus met uitvoeringsbereik, zodat beleid, goedkeuringen, hooks,
telemetrie, transcriptprojectie en exacte tool-id's worden gedeeld met
normale tooluitvoering. De gastgerichte weergaven `ALL_TOOLS`, `tools.search(...)`,
`tools.describe(...)`, `tools.callValue(...)` en `tools.call(...)` laten MCP-vermeldingen weg. De
gegenereerde namespace `MCP.<server>.<tool>({ ...input })` wordt terugvertaald naar de
exacte catalogus-id en verzendt via hetzelfde uitvoerderpad.

## Interactie met Tool Search

De codemodus vervangt het OpenClaw-modeloppervlak van Tool Search voor uitvoeringen waarin deze
actief is.

Wanneer `tools.codeMode.enabled` waar is en de codemodus wordt geactiveerd:

- OpenClaw stelt `tool_search_code`, `tool_search`, `tool_describe`
  en `tool_call` niet beschikbaar als voor het model zichtbare tools.
- Hetzelfde catalogusidee verhuist naar de gast-runtime.
- De gast-runtime ontvangt compacte `ALL_TOOLS`-metadata en helpers voor zoeken/beschrijven/
  aanroepen voor niet-MCP-tools.
- MCP-aanroepen gebruiken de gegenereerde namespace `MCP` en de bijbehorende `$api()`-headers
  in plaats van `tools.call(...)`.
- Geneste aanroepen worden verzonden via hetzelfde uitvoerderpad van OpenClaw dat Tool
  Search gebruikt.

Zie [Tool Search](/nl/tools/tool-search) voor de compacte catalogusbrug van OpenClaw
die de codemodus voor actieve uitvoeringen vervangt.

## Toolnamen en conflicten

De voor het model zichtbare tool `exec` is de codemodustool. Als de normale OpenClaw-
shelltool `exec` is ingeschakeld, wordt deze voor het model verborgen en net als
elke andere tool in de catalogus opgenomen.

Binnen de gast-runtime:

- `tools.call("openclaw:core:exec", input)` kan de shell-uitvoeringstool aanroepen als
  het beleid dit toestaat.
- `tools.exec(...)` wordt alleen geïnstalleerd als de catalogusvermelding voor shell-uitvoering een
  ondubbelzinnige, veilige naam heeft.
- de codemodustool `exec` is nooit recursief beschikbaar via `tools`.

Als twee tools naar dezelfde veilige gemaksnaam worden genormaliseerd, laat OpenClaw de
gemaksfunctie weg en is `tools.call(id, input)` vereist.

## Geneste tooluitvoering

Elke geneste toolaanroep passeert de hostbrug en gaat OpenClaw opnieuw binnen,
met behoud van: actieve agent-id, sessie-id en -sleutel, afzender- en kanaalcontext,
sandboxbeleid, goedkeuringsbeleid, Plugin-`before_tool_call`-hooks, afbreek-
signaal, streamingupdates waar beschikbaar en traject-/auditgebeurtenissen.

Geneste aanroepen worden als echte toolaanroepen in het transcript geprojecteerd, zodat ondersteunings-
bundels laten zien wat er is gebeurd, waarbij de projectie de bovenliggende
codemodustoolaanroep en de geneste tool-id identificeert.

Parallelle geneste aanroepen zijn toegestaan tot `maxPendingToolCalls`.

## Levenscyclus van uitvoeringen en momentopnamen

Elke codemodusuitvoering wordt bijgehouden in een kaart binnen het proces met `runId` als sleutel (niet
opgeslagen op schijf of in een database). `exec`/`wait` retourneren een van drie resultaat-
statussen: `completed`, `waiting` of `failed`.

- Een `waiting`-resultaat bewaart de QuickJS-momentopname, wachtende brugverzoeken en
  bereikmetadata (agentuitvoerings-id, sessie-id/-sleutel) totdat `wait` deze hervat of
  deze verloopt.
- Verlopen, een verkeerde sessie, een verkeerde uitvoering en onbekende/reeds hervattende `runId`-
  waarden leveren geen afzonderlijke eindstatus op; ze worden weergegeven als een
  `failed`-resultaat (`code: "invalid_input"`) met een bericht zoals `code mode
run is unavailable or expired.` of `code mode run belongs to a different
session.`.
- De momentopname van een uitvoering wordt uit de kaart verwijderd zodra deze eindigt als
  `completed` of `failed`, of wordt verwijderd wanneer de Gateway wordt afgesloten (niets
  overleeft een herstart: dit is tijdelijke runtimestatus).
- Voor alleen-lezenwerk kan `exec` `restartSafe: true` instellen. OpenClaw weigert dan
  catalogusaanroepen en Pluginnamespaces met neveneffecten vóór uitvoering en
  markeert opgeschorte resultaten als veilig voor herhaling. Als een herstart `wait` onderbreekt,
  reconstrueert [herstel na herstart](/nl/gateway/restart-recovery) de beurt vanuit het
  transcript in plaats van de proceslokale momentopname te herstellen. De herstel-
  beurt zelf blijft beperkt tot gecontroleerde alleen-lezenkerntools en expliciet
  herhalingsveilige Plugintools.
- OpenClaw begrenst het aantal gelijktijdig opgeschorte uitvoeringen per proces (64) en
  weigert nieuwe opschortingen boven die limiet met `too many suspended code mode
runs.`.

Opslag van momentopnamen wordt begrensd door `maxSnapshotBytes` per uitvoering, de bovenstaande limiet
voor opgeschorte uitvoeringen per proces en `snapshotTtlSeconds`.

## QuickJS-WASI-runtime

OpenClaw laadt `quickjs-wasi` als directe afhankelijkheid in het pakket dat hiervan eigenaar is; het
vertrouwt niet op een transitieve kopie die voor een niet-gerelateerde afhankelijkheid is geïnstalleerd.

Verantwoordelijkheden van de runtime: de QuickJS-WASI-WebAssembly-module compileren/laden;
één geïsoleerde VM maken per codemodusuitvoering of hervatting; hostcallbacks
onder stabiele namen registreren; geheugen- en onderbrekingslimieten instellen; JavaScript evalueren; wachtende
taken afhandelen; opgeschorte VM-status vastleggen; momentopnamen voor `wait` herstellen;
VM-handles en momentopnamen na eindstatussen vrijgeven.

De runtime wordt uitgevoerd in een Node.js-workerthread, buiten de hoofd-
gebeurtenislus van OpenClaw. Een oneindige lus van een gast mag het Gateway-proces niet
voor onbepaalde tijd blokkeren; de onderbrekingshandler van de worker handhaaft de wandkloktime-out
onafhankelijk van medewerking door de gastcode.

## TypeScript

TypeScript-ondersteuning is uitsluitend een brontransformatie: geaccepteerde invoer is één
tekenreeks met TypeScript-code; uitvoer is een JavaScript-tekenreeks die door
QuickJS-WASI wordt geëvalueerd. Er is geen typecontrole, geen moduleresolutie en geen
`import`/`require`. Diagnostiek wordt geretourneerd als `failed`-resultaten.

De TypeScript-compiler wordt alleen voor TypeScript-cellen lui geladen; gewone
JavaScript-cellen en een uitgeschakelde codemodus laden deze nooit.

## Beveiligingsgrens

Modelcode is vijandig. De runtime gebruikt gelaagde beveiliging:

- voert QuickJS-WASI buiten de hoofdgebeurtenislus uit, in een workerthread
- laadt `quickjs-wasi` als directe afhankelijkheid, niet via Codex of een
  transitief pakket
- geen bestandssysteem, netwerk, subproces, module-import, omgevingsvariabelen
  of globale hostobjecten in de gast
- gebruikt geheugen- en onderbrekingslimieten van QuickJS plus een wandklok-
  time-out van het bovenliggende proces
- handhaaft limieten voor uitvoer, momentopnamen, logboeken en wachtende aanroepen
- serialiseert hostbrugwaarden via een beperkte JSON-adapter
- converteert hostfouten naar gewone gastfouten, nooit naar objecten uit het hostbereik
- verwijdert momentopnamen bij time-out, afbreken, sessie-einde of verlopen
- weigert recursieve toegang tot `exec`, `wait` en besturingstools van Tool Search
- voorkomt dat conflicten tussen gemaksnamen catalogushelpers overschaduwen

De sandbox is één beveiligingslaag; operators hebben mogelijk nog steeds verharding op
besturingssysteemniveau nodig voor implementaties met een hoog risico.

## Foutcodes

```typescript
type CodeModeErrorCode =
  | "invalid_input"
  | "runtime_unavailable"
  | "timeout"
  | "output_limit_exceeded"
  | "snapshot_limit_exceeded"
  | "internal_error";
```

`invalid_input` omvat ongeldige argumenten voor `exec`/`wait`, uitgeschakelde talen,
geweigerde moduletoegang, fouten bij TypeScript-transformatie, onbekende/verlopen/
buiten bereik vallende `runId`-waarden en te veel opgeschorte uitvoeringen. `runtime_unavailable`
omvat een QuickJS-worker die niet kan starten of met een niet-nulstatus afsluit.

Fouten die naar de gast worden geretourneerd, zijn gewone gegevens; hostinstanties van `Error`, stack-
objecten, prototypes en hostfuncties worden niet naar QuickJS overgebracht.

## Telemetrie

Het veld `telemetry` van elk resultaat rapporteert: de grootte van de verborgen catalogus en een uitsplitsing
naar bron (aantallen `openclaw`/`mcp`/`client`), cumulatieve aantallen zoek-/beschrijf-/aanroep-
bewerkingen voor de catalogus van de uitvoering en de voor het model zichtbare toolnamen (`exec`,
`wait` en behouden tools die alleen rechtstreeks beschikbaar zijn).

Telemetrie mag geen geheimen, onbewerkte omgevingswaarden of niet-geredigeerde
toolinvoer bevatten buiten het bestaande trajectbeleid van OpenClaw.

## Foutopsporing

Gebruik gerichte logging van modeltransport wanneer de codemodus zich anders gedraagt dan
een normale tooluitvoering:

```bash
OPENCLAW_DEBUG_CODE_MODE=1 \
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 \
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools \
OPENCLAW_DEBUG_SSE=events \
openclaw gateway
```

Gebruik voor het debuggen van de payloadvorm `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted`.
Hiermee wordt een begrensde, geredigeerde JSON-momentopname van het modelverzoek gelogd; gebruik dit alleen
tijdens het debuggen, omdat prompts en berichttekst nog steeds kunnen verschijnen.

Gebruik voor streamdebugging `OPENCLAW_DEBUG_SSE=peek` om de eerste vijf
geredigeerde SSE-events te loggen. De codemodus stopt ook veilig als de uiteindelijke providerpayload
niet precies één `exec`, één `wait` en uitsluitend goedgekeurde
direct-only tools bevat nadat het codemodusoppervlak is geactiveerd.

## Implementatie-indeling

- configuratiecontract: `tools.codeMode`
- catalogusbouwer: effectieve tools omzetten in compacte vermeldingen en een id-toewijzing
- adapter voor het modeloppervlak: zichtbare tools vervangen door besturings-/directe tools
- QuickJS-WASI-runtimeadapter: laden, evalueren, momentopname maken, herstellen, vrijgeven
- workersupervisor: time-out, afbreken, crashisolatie
- bridge-adapter: JSON-veilige hostcallbacks en levering van resultaten
- TypeScript-transformatieadapter
- opslag voor momentopnamen: TTL, groottelimieten, afbakening per uitvoering/sessie
- trajectieprojectie voor geneste toolaanroepen
- telemetrietellers en diagnostiek

De implementatie hergebruikt catalogus- en executorconcepten uit Tool Search, maar
gebruikt geen `node:vm`-child als sandbox.

## Validatiechecklist

De dekking van de codemodus moet aantonen dat:

- uitgeschakelde configuratie de bestaande beschikbaarstelling van tools ongewijzigd laat
- objectconfiguratie zonder `enabled: true` de codemodus uitgeschakeld laat
- ingeschakelde configuratie `exec`, `wait` en uitsluitend vereiste direct-only tools aan
  het model beschikbaar stelt wanneer tools actief zijn voor de uitvoering
- onbewerkte uitvoeringen zonder tools, `disableTools` en lege toelatingslijsten geen
  payloadhandhaving voor de codemodus activeren
- alle effectieve niet-MCP-tools die voor de catalogus in aanmerking komen in `ALL_TOOLS` verschijnen
- direct-only tools zichtbaar blijven voor het model en niet in `ALL_TOOLS` verschijnen
- geweigerde tools niet in `ALL_TOOLS` verschijnen
- `tools.search`, `tools.describe`, `tools.callValue` en `tools.call` werken voor OpenClaw-tools
- `API.list("mcp")` en `API.read("mcp/<server>.d.ts")` MCP-declaraties in TypeScript-stijl
  beschikbaar stellen zonder een bridge-/toolaanroep
- MCP-namespace `$api()` beschikbaar blijft als inline terugvaloptie voor schema's
- MCP-namespaceaanroepen werken voor zichtbare MCP-tools met één objectinvoer, terwijl
  directe MCP-catalogusvermeldingen ontbreken in `tools.*`
- besturingstools van Tool Search verborgen zijn voor zowel het modeloppervlak als de
  verborgen catalogus
- geneste aanroepen het goedkeurings- en hookgedrag behouden
- shell `exec` verborgen is voor het model, maar via de catalogus-id kan worden aangeroepen wanneer
  dit is toegestaan
- recursieve codemodus-`exec` en `wait` niet vanuit gastcode kunnen worden aangeroepen
- TypeScript-invoer wordt getransformeerd en geëvalueerd zonder TypeScript te laden op
  uitgeschakelde of uitsluitend JavaScript gebruikende paden
- `import`, `require` en toegang tot het bestandssysteem, netwerk en de omgeving mislukken
- oneindige lussen een time-out krijgen en de Gateway niet kunnen blokkeren
- fouten door geheugenlimieten de gast-VM beëindigen
- uitvoer- en momentopnamelimieten worden afgedwongen voor voltooide en onderbroken aanroepen
- `wait` een onderbroken momentopname hervat en de uiteindelijke waarde retourneert
- verlopen, afgebroken, bij een verkeerde sessie behorende en onbekende `runId`-waarden mislukken
- het opnieuw afspelen en opslaan van transcripties besturingsaanroepen van de codemodus behouden
- transcriptie en telemetrie geneste toolaanroepen duidelijk weergeven

## E2E-testplan

Voer deze uit als integratie- of end-to-endtests wanneer je de runtime wijzigt:

1. Start een Gateway met `tools.codeMode.enabled: false`.
2. Verstuur een agentbeurt met een kleine set directe tools.
3. Controleer dat de voor het model zichtbare tools ongewijzigd zijn.
4. Start opnieuw met `tools.codeMode.enabled: true`.
5. Verstuur een agentbeurt met testtools voor OpenClaw, plugins, MCP en clients.
6. Controleer dat de voor het model zichtbare lijst met tools `exec`, `wait` en uitsluitend geconfigureerde
   direct-only tools bevat.
7. Lees in `exec` `ALL_TOOLS` en controleer dat de effectieve testtools die voor de catalogus in aanmerking komen
   aanwezig zijn, terwijl direct-only tools ontbreken.
8. Roep in `exec` OpenClaw-/plugin-/clienttools aan via `tools.search`,
   `tools.describe` en `tools.callValue` (of onbewerkte `tools.call`).
9. Roep in `exec` `API.list("mcp")` en `API.read("mcp/<server>.d.ts")` aan en
   controleer dat de declaratiebestanden zichtbare MCP-tools beschrijven.
10. Roep in `exec` MCP-tools aan via `MCP.<server>.<tool>({ ...input })` en
    controleer dat directe MCP-catalogusvermeldingen ontbreken in `ALL_TOOLS` en
    `tools.*`.
11. Controleer dat geweigerde tools ontbreken en niet via een geraden id kunnen worden aangeroepen.
12. Start een geneste toolaanroep die wordt voltooid nadat `exec` `waiting` retourneert.
13. Roep `wait` aan en controleer dat de herstelde VM het toolresultaat ontvangt.
14. Controleer dat het uiteindelijke antwoord uitvoer bevat die na het herstel is geproduceerd.
15. Controleer dat time-out, afbreken en het verlopen van de momentopname de runtimestatus opschonen.
16. Exporteer het traject en controleer dat geneste aanroepen zichtbaar zijn onder de bovenliggende
    codemodusaanroep.

Voor wijzigingen op deze pagina die alleen documentatie betreffen, moet nog steeds `pnpm check:docs` worden uitgevoerd.

## Gerelateerd

- [Swarm](/tools/swarm) voor fan-out-agentorkestratie vanuit codemodusscripts
- [Tool Search](/nl/tools/tool-search)
- [Agentruntimes](/nl/concepts/agent-runtimes)
- [Exec-tool](/nl/tools/exec)
- [Code-uitvoering](/nl/tools/code-execution)
