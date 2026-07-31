---
read_when:
    - Je hebt de exacte typesignatuur van defineToolPlugin, definePluginEntry of defineChannelPluginEntry nodig
    - Je wilt de registratiemodus begrijpen (volledig versus configuratie versus CLI-metadata)
    - Je zoekt opties voor het toegangspunt
sidebarTitle: Entry Points
summary: Naslaginformatie voor defineToolPlugin, definePluginEntry, defineChannelPluginEntry en defineSetupPluginEntry
title: Ingangspunten voor Plugins
x-i18n:
    generated_at: "2026-07-27T06:29:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e64fe1d65531fea8f266aa23b73064daf2ed2c5c43af8bb08ea57e347fe566f4
    source_path: plugins/sdk-entrypoints.md
    workflow: 16
---

Elke plugin exporteert een standaardentryobject. De SDK biedt een helper voor
elke entryvorm: `defineToolPlugin`, `definePluginEntry`,
`defineChannelPluginEntry`, `defineSetupPluginEntry`.

<Tip>
  **Op zoek naar een stapsgewijze uitleg?** Zie [Toolplugins](/nl/plugins/tool-plugins),
  [Kanaalplugins](/nl/plugins/sdk-channel-plugins) of
  [Providerplugins](/nl/plugins/sdk-provider-plugins) voor stapsgewijze handleidingen.
</Tip>

## Pakketentries

Geïnstalleerde plugins laten de velden `package.json` `openclaw` naar zowel bron- als
gebouwde entries verwijzen:

```json
{
  "openclaw": {
    "extensions": ["./src/index.ts"],
    "runtimeExtensions": ["./dist/index.js"],
    "setupEntry": "./src/setup-entry.ts",
    "runtimeSetupEntry": "./dist/setup-entry.js"
  }
}
```

- `extensions` en `setupEntry` zijn bronentries, die worden gebruikt voor ontwikkeling in een workspace en vanuit een
  git-checkout.
- `runtimeExtensions` en `runtimeSetupEntry` hebben de voorkeur voor geïnstalleerde
  pakketten: daarmee kunnen npm-pakketten TypeScript-compilatie tijdens runtime overslaan.
- `runtimeExtensions` moet, indien aanwezig, qua arraylengte overeenkomen met `extensions`
  (entries worden op basis van hun positie gekoppeld). `runtimeSetupEntry` vereist `setupEntry`.
- Als een `runtimeExtensions`- of `runtimeSetupEntry`-artefact is gedeclareerd maar
  ontbreekt, mislukt de installatie/detectie met een verpakkingsfout; OpenClaw valt niet
  stilzwijgend terug op de broncode. Terugvallen op de broncode (hieronder) is alleen van toepassing als er helemaal geen
  runtime-entry is gedeclareerd.
- Als een geïnstalleerd pakket alleen een TypeScript-bronentry declareert, zoekt OpenClaw
  naar een overeenkomende gebouwde `dist/*.js`-peer (of `.mjs`/`.cjs`) en gebruikt deze;
  anders valt het terug op de TypeScript-broncode.
- Alle entrypaden moeten binnen de pakketmap van de plugin blijven. Runtime-
  entries en afgeleide gebouwde JS-peers maken een ontsnappend bronpad `extensions` of
  `setupEntry` niet geldig.

## `defineToolPlugin`

**Import:** `openclaw/plugin-sdk/tool-plugin`

Voor plugins die alleen agenttools toevoegen. Houdt de broncode klein, leidt configuratie-
en toolparametertypen af uit TypeBox-schema's, verpakt gewone retourwaarden in
de OpenClaw-toolresultaatindeling en stelt statische metadata beschikbaar die
`openclaw plugins build` naar het pluginmanifest schrijft (`contracts.tools`,
`configSchema`).

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "Fetch stock quotes.",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "API key." })),
  }),
  tools: (tool) => [
    tool({
      name: "quote",
      label: "Quote",
      description: "Fetch a quote.",
      parameters: Type.Object({
        symbol: Type.String({ description: "Ticker symbol." }),
      }),
      outputSchema: Type.Object(
        {
          symbol: Type.String(),
          hasKey: Type.Boolean(),
        },
        { additionalProperties: false },
      ),
      execute: async ({ symbol }, config) => ({ symbol, hasKey: Boolean(config.apiKey) }),
    }),
  ],
});
```

- `configSchema` is optioneel; bij weglaten wordt een strikt leeg objectschema gebruikt
  (het gegenereerde manifest bevat nog steeds `configSchema`).
- `execute` retourneert een gewone tekenreeks of JSON-serialiseerbare waarde; de helper
  verpakt deze als een teksttoolresultaat waarbij `details` is ingesteld op de oorspronkelijke
  (niet naar een tekenreeks omgezette) retourwaarde.
- `outputSchema` beschrijft optioneel die oorspronkelijke `details`-waarde voor Code
  Mode en Tool Search. Catalogusaanroepen weigeren vóór uitvoering een ongeldig schema
  en valideren de uiteindelijke waarde voordat deze wordt geretourneerd.
- Voor aangepaste toolresultaten exporteert `openclaw/plugin-sdk/tool-results`
  `textResult` en `jsonResult`.
- Toolnamen zijn statisch, zodat `openclaw plugins build`
  `contracts.tools` afleidt uit de gedeclareerde tools zonder handmatig gedupliceerde namen.
- Het laden tijdens runtime blijft strikt: geïnstalleerde plugins hebben nog steeds
  `openclaw.plugin.json` en `package.json` `openclaw.extensions` nodig. OpenClaw
  voert nooit plugincode uit om ontbrekende manifestgegevens af te leiden.

## `definePluginEntry`

**Import:** `openclaw/plugin-sdk/plugin-entry`

Voor providerplugins, geavanceerde toolplugins, hookplugins en alles wat
**geen** berichtenkanaal is.

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Short summary",
  register(api) {
    api.registerProvider({/* ... */});
    api.registerTool({/* ... */});
  },
});
```

| Veld                      | Type                                                             | Vereist  | Standaard           |
| ------------------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`                      | `string`                                                         | Ja       | -                   |
| `name`                    | `string`                                                         | Ja       | -                   |
| `description`             | `string`                                                         | Ja       | -                   |
| `kind`                    | `string` (verouderd, zie hieronder)                              | Nee      | -                   |
| `configSchema`            | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | Nee      | Leeg objectschema   |
| `reload`                  | `OpenClawPluginReloadRegistration`                               | Nee      | -                   |
| `nodeHostCommands`        | `OpenClawPluginNodeHostCommand[]`                                | Nee      | -                   |
| `securityAuditCollectors` | `OpenClawPluginSecurityAuditCollector[]`                         | Nee      | -                   |
| `register`                | `(api: OpenClawPluginApi) => void`                               | Ja       | -                   |

- `id` moet overeenkomen met je `openclaw.plugin.json`-manifest.
- Externe sessiecatalogi gebruiken
  `openclaw/plugin-sdk/session-catalog` en
  `api.registerSessionCatalog({ id, label, list, read, continueSession?, archive? })`.
  De kern beheert de `sessions.catalog.*`-methoden van de Gateway; providers retourneren host-,
  sessie- en genormaliseerde transcriptprojecties zonder RPC's te registreren. Een
  lijstprovider moet de optionele callback `onHost(host)` aanroepen zodra elke host
  is afgerond; de geretourneerde hostarray blijft vereist als uiteindelijke compatibiliteits-
  momentopname.
- `kind` is verouderd: declareer in plaats daarvan een exclusief slot (`"memory"` of
  `"context-engine"`) in het veld `kind` van het `openclaw.plugin.json`-manifest.
  `kind` van de runtime-entry blijft alleen behouden als compatibiliteitsterugval voor
  oudere plugins.
- `configSchema` kan een functie zijn voor luie evaluatie. OpenClaw verwerkt en
  memoiseert het schema bij de eerste toegang, zodat kostbare schemabouwers slechts
  eenmaal worden uitgevoerd.
- Een `nodeHostCommands`-descriptor kan `isAvailable({ config, env })` definiëren.
  Als `false` wordt geretourneerd, worden die opdracht en de bijbehorende mogelijkheid weggelaten uit de Gateway-
  declaratie van de headless node. OpenClaw evalueert dit aan de hand van de node-lokale
  opstartconfiguratie; opdrachthandlers moeten bij aanroep nog steeds de beschikbaarheid
  valideren.

## `defineChannelPluginEntry`

**Import:** `openclaw/plugin-sdk/channel-core`

Verpakt `definePluginEntry` met kanaalspecifieke bedrading: het roept automatisch
`api.registerChannel({ plugin })` aan, stelt een optionele metadatakoppeling voor CLI-
hoofdhulp beschikbaar en beperkt `registerFull` op basis van de registratiemodus.

```typescript
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineChannelPluginEntry({
  id: "my-channel",
  name: "My Channel",
  description: "Short summary",
  plugin: myChannelPlugin,
  setRuntime: setMyRuntime,
  registerCliMetadata(api) {
    api.registerCli(/* ... */);
  },
  registerFull(api) {
    api.registerGatewayMethod(/* ... */);
  },
});
```

| Veld                  | Type                                                             | Vereist  | Standaard           |
| --------------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`                  | `string`                                                         | Ja       | -                   |
| `name`                | `string`                                                         | Ja       | -                   |
| `description`         | `string`                                                         | Ja       | -                   |
| `plugin`              | `ChannelPlugin`                                                  | Ja       | -                   |
| `configSchema`        | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | Nee      | Leeg objectschema   |
| `setRuntime`          | `(runtime: PluginRuntime) => void`                               | Nee      | -                   |
| `registerCliMetadata` | `(api: OpenClawPluginApi) => void`                               | Nee      | -                   |
| `registerFull`        | `(api: OpenClawPluginApi) => void`                               | Nee      | -                   |

Callbacks worden uitgevoerd per registratiemodus (volledige tabel onder
[Registratiemodus](#registration-mode)):

- `setRuntime` wordt uitgevoerd in elke modus behalve `"cli-metadata"` en
  `"tool-discovery"`. Sla hier de runtimereferentie op, doorgaans via
  `createPluginRuntimeStore`.
- `registerCliMetadata` wordt uitgevoerd voor `"cli-metadata"`, `"discovery"` en
  `"full"`. Gebruik dit als de canonieke plaats voor CLI-descriptors die eigendom zijn van het kanaal,
  zodat hoofdhulp niet-activerend blijft, detectiemomentopnamen statische
  opdrachtmetadata bevatten en normale CLI-registratie compatibel blijft met volledige
  pluginladingen.
- `registerFull` wordt alleen uitgevoerd voor `"full"` en `"tool-discovery"`. Voor
  `"tool-discovery"` wordt het uitgevoerd _in plaats van_ kanaalregistratie: OpenClaw
  slaat `registerChannel`/`setRuntime` volledig over en roept alleen
  `registerFull` aan, zodat elke provider-/toolregistratie die je kanaal nodig heeft voor
  zelfstandige tooldetectie of -uitvoering daar moet staan en niet achter de normale
  kanaalconfiguratie.
- Detectieregistratie is niet-activerend, niet importvrij: OpenClaw kan
  de vertrouwde pluginentry en kanaalpluginmodule evalueren om de
  momentopname te bouwen. Houd imports op het hoogste niveau vrij van neveneffecten en plaats sockets,
  clients, workers en services achter paden die uitsluitend via `"full"` lopen.
- Net als `definePluginEntry` kan `configSchema` een luie factory zijn; OpenClaw
  memoiseert het verwerkte schema bij de eerste toegang.

CLI-registratie:

- Gebruik `api.registerCli(..., { descriptors: [...] })` voor root-CLI-opdrachten die eigendom zijn van een plugin
  en die je lazy-loaded wilt hebben zonder dat ze uit de parseerboom van de root-CLI
  verdwijnen. Descriptornamen mogen alleen letters, cijfers, koppeltekens en
  underscores bevatten en moeten met een letter of cijfer beginnen; OpenClaw weigert andere
  vormen en verwijdert terminalbesturingsreeksen uit beschrijvingen voordat
  de hulptekst wordt weergegeven. Dek elke root van een opdracht op het hoogste niveau af die de registrar beschikbaar stelt.
  Alleen `commands` blijft het eager compatibiliteitspad gebruiken.
- Gebruik `api.registerNodeCliFeature(...)` voor functieopdrachten voor gekoppelde nodes, zodat
  ze onder `openclaw nodes` terechtkomen (gelijkwaardig aan
  `registerCli(registrar, { parentPath: ["nodes"], ... })`).
- Voeg voor andere geneste pluginopdrachten `parentPath` toe en registreer opdrachten
  op het `program`-object dat aan de registrar wordt doorgegeven; OpenClaw herleidt dit tot
  de bovenliggende opdracht voordat de plugin wordt aangeroepen.
- Registreer voor kanaalplugins CLI-descriptors vanuit `registerCliMetadata`
  en houd `registerFull` gericht op uitsluitend runtimewerk.
- Als `registerFull` ook Gateway-RPC-methoden registreert, houd deze dan onder een
  pluginspecifiek voorvoegsel. Gereserveerde beheerdersnaamruimten van de kern (`config.*`,
  `exec.approvals.*`, `wizard.*`, `update.*`) worden altijd omgezet naar
  `operator.admin`.

## `defineSetupPluginEntry`

**Importeren:** `openclaw/plugin-sdk/channel-core`

Voor het lichtgewicht bestand `setup-entry.ts`. Retourneert alleen `{ plugin }`, zonder
runtime- of CLI-koppeling.

```typescript
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineSetupPluginEntry(myChannelPlugin);
```

OpenClaw laadt dit in plaats van het volledige toegangspunt wanneer een kanaal is uitgeschakeld,
niet is geconfigureerd of wanneer uitgesteld laden is ingeschakeld. Zie
[Installatie en configuratie](/nl/plugins/sdk-setup#setup-entry) voor wanneer dit van belang is.

Combineer `defineSetupPluginEntry(...)` met de beperkte families van installatiehulpfuncties:

| Import                              | Gebruiken voor                                                                                                                                                                            |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw/plugin-sdk/setup-runtime` | Runtimeveilige installatiehulpfuncties: `createSetupTranslator`, importveilige adapters voor installatiepatches, uitvoer van opzoeknotities, `promptResolvedAllowFrom`, `splitSetupEntries`, gedelegeerde installatieproxy's |
| `openclaw/plugin-sdk/channel-setup` | Installatieoppervlakken voor optionele installaties                                                                                                                                                    |
| `openclaw/plugin-sdk/setup-tools`   | Hulpfuncties voor installatie-CLI, archieven en documentatie                                                                                                                                       |

Houd zware SDK's, CLI-registratie en langlopende runtimeservices in het
volledige toegangspunt.

Gebundelde werkruimtekanaalplugins die installatie- en runtimeoppervlakken splitsen, kunnen in plaats daarvan
`defineBundledChannelSetupEntry(...)` uit
`openclaw/plugin-sdk/channel-entry-contract` gebruiken. Hiermee kan het installatietoegangspunt
installatieveilige exports voor plugins/geheimen behouden en toch een runtime-
setter beschikbaar stellen:

```typescript
import { defineBundledChannelSetupEntry } from "openclaw/plugin-sdk/channel-entry-contract";

export default defineBundledChannelSetupEntry({
  importMetaUrl: import.meta.url,
  plugin: {
    specifier: "./channel-plugin-api.js",
    exportName: "myChannelPlugin",
  },
  runtime: {
    specifier: "./runtime-api.js",
    exportName: "setMyChannelRuntime",
  },
  registerSetupRuntime(api) {
    api.registerHttpRoute({
      path: "/my-channel/events",
      auth: "plugin",
      handler: async (req, res) => {
        /* installatieveilige route */
      },
    });
  },
});
```

Gebruik dit alleen wanneer een installatieflow werkelijk een lichtgewicht runtime-setter of
installatieveilig Gateway-oppervlak nodig heeft voordat het volledige kanaaltoegangspunt wordt geladen.
`registerSetupRuntime` wordt alleen uitgevoerd voor `"setup-runtime"`-laadbewerkingen; beperk dit
tot routes of methoden die uitsluitend voor configuratie dienen en die vóór uitgestelde
volledige activering beschikbaar moeten zijn.

## Registratiemodus

`api.registrationMode` geeft je plugin door hoe deze is geladen:

| Modus               | Wanneer                                               | Wat te registreren                                                                                                        |
| ------------------ | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `"full"`           | Normaal opstarten van de Gateway                             | Alles                                                                                                              |
| `"discovery"`      | Alleen-lezen detectie van mogelijkheden                     | Kanaalregistratie plus statische CLI-descriptors; toegangspuntcode mag worden geladen, maar sla sockets, workers, clients en services over |
| `"tool-discovery"` | Beperkte laadbewerking om tools van specifieke plugins weer te geven of uit te voeren | Alleen registratie van mogelijkheden/tools; geen kanaalactivering                                                                |
| `"setup-only"`     | Uitgeschakeld/niet-geconfigureerd kanaal                      | Alleen kanaalregistratie                                                                                               |
| `"setup-runtime"`  | Installatieflow met beschikbare runtime                  | Kanaalregistratie plus alleen de lichtgewicht runtime die nodig is voordat het volledige toegangspunt wordt geladen                               |
| `"cli-metadata"`   | Roothulp / vastleggen van CLI-metagegevens                   | Alleen CLI-descriptors                                                                                                    |

`defineChannelPluginEntry` verwerkt deze splitsing automatisch. Als je
`definePluginEntry` rechtstreeks voor een kanaal gebruikt, controleer dan zelf de modus en onthoud dat
`"tool-discovery"` de kanaalregistratie overslaat:

```typescript
register(api) {
  if (
    api.registrationMode === "cli-metadata" ||
    api.registrationMode === "discovery" ||
    api.registrationMode === "full"
  ) {
    api.registerCli(/* ... */);
    if (api.registrationMode === "cli-metadata") return;
  }

  if (api.registrationMode === "tool-discovery") {
    // Registreer alleen mogelijkhedenoppervlakken (providers/tools), geen kanaal.
    return;
  }

  api.registerChannel({ plugin: myPlugin });
  if (api.registrationMode !== "full") return;

  // Zware registraties die uitsluitend voor de runtime zijn
  api.registerService(/* ... */);
}
```

Langlopende services mogen kleine invalidatie- of levenscyclusgebeurtenissen verzenden via
hun servicecontext:

```typescript
api.registerService({
  id: "index-events",
  start(ctx) {
    ctx.gatewayEvents?.emit("changed", { revision: 1 }, { scope: "operator.read" });
  },
});
```

OpenClaw plaatst dit in de naamruimte `plugin.<plugin-id>.changed`. Gebeurtenisnamen bestaan uit één
segment in kleine letters, payloads moeten begrensde JSON zijn en het bereik moet
`operator.read`, `operator.write` of `operator.admin` zijn. De emitter bestaat alleen
gedurende de levensduur van de service en wordt ingetrokken na het stoppen of een mislukte start. Geef de voorkeur aan
versie- of invalidatiepayloads boven volledige records, zodat geautoriseerde clients
de canonieke status opnieuw lezen via de beperkte Gateway-methoden van de plugin.

De detectiemodus bouwt een niet-activerende momentopname van het register. Deze kan nog steeds
het plugintoegangspunt en het kanaalpluginobject evalueren, zodat OpenClaw
kanaalmogelijkheden en statische CLI-descriptors kan registreren. Behandel module-
evaluatie tijdens detectie als vertrouwd maar lichtgewicht: geen netwerkclients,
subprocessen, listeners, databaseverbindingen, achtergrondworkers,
het lezen van aanmeldgegevens of andere actieve runtimebijwerkingen op het hoogste niveau.

Beschouw `"setup-runtime"` als het venster waarin opstartoppervlakken die uitsluitend voor installatie dienen,
moeten bestaan zonder de volledige gebundelde kanaalruntime opnieuw te betreden. Geschikte toepassingen zijn
kanaalregistratie, installatieveilige HTTP-routes, installatieveilige Gateway-methoden
en gedelegeerde installatiehulpfuncties. Zware achtergrondservices, CLI-registrars en
initialisatie van provider-/client-SDK's horen nog steeds thuis in `"full"`.

## Pluginvormen

OpenClaw classificeert geladen plugins op basis van hun registratiegedrag:

| Vorm                 | Beschrijving                                        |
| --------------------- | -------------------------------------------------- |
| **plain-capability**  | Eén type mogelijkheid (bijv. alleen provider)           |
| **hybrid-capability** | Meerdere typen mogelijkheden (bijv. provider + spraak) |
| **hook-only**         | Alleen hooks, geen mogelijkheden                        |
| **non-capability**    | Tools/opdrachten/services, maar geen mogelijkheden        |

Gebruik `openclaw plugins inspect <id>` om de vorm van een plugin te bekijken.

## Gerelateerd

- [SDK-overzicht](/nl/plugins/sdk-overview) - registratie-API en subpadreferentie
- [Runtimehulpfuncties](/nl/plugins/sdk-runtime) - `api.runtime` en `createPluginRuntimeStore`
- [Installatie en configuratie](/nl/plugins/sdk-setup) - manifest, installatietoegangspunt, uitgesteld laden
- [Kanaalplugins](/nl/plugins/sdk-channel-plugins) - het `ChannelPlugin`-object bouwen
- [Providerplugins](/nl/plugins/sdk-provider-plugins) - providerregistratie en hooks
