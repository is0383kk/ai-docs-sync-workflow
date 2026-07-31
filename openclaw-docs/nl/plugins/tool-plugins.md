---
read_when:
    - Je wilt een eenvoudige OpenClaw-Plugin bouwen die alleen agenttools toevoegt
    - Je wilt defineToolPlugin gebruiken in plaats van de metagegevens van het Plugin-manifest handmatig te schrijven
    - Je moet een Plugin met alleen tools opzetten, genereren, valideren, testen of publiceren
sidebarTitle: Tool Plugins
summary: Bouw eenvoudige getypeerde agenttools met defineToolPlugin en openclaw plugins init/build/validate
title: Toolplugins
x-i18n:
    generated_at: "2026-07-27T05:17:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac23d15ba79cbdd1d8b8eab7c87007b44af16361b2866b14123e18f816bf4075
    source_path: plugins/tool-plugins.md
    workflow: 16
---

`defineToolPlugin` bouwt een Plugin die alleen door agents aanroepbare tools toevoegt: geen
kanaal, modelprovider, hook, service of setupbackend. Hiermee worden de
manifestmetadata gegenereerd die OpenClaw nodig heeft om tools te ontdekken zonder de
runtimecode van de Plugin te laden.

Begin voor plugins voor providers, kanalen, hooks, services of gemengde mogelijkheden in plaats daarvan met
[Plugins bouwen](/nl/plugins/building-plugins), [Kanaalplugins](/nl/plugins/sdk-channel-plugins)
of [Providerplugins](/nl/plugins/sdk-provider-plugins).

## Vereisten

- Node 22.22.3+, Node 24.15+ of Node 25.9+.
- TypeScript ESM-pakketuitvoer.
- `typebox` in `dependencies` (niet alleen `devDependencies` — de gegenereerde
  Plugin importeert dit tijdens runtime).
- `openclaw >=2026.5.17`, de eerste versie die
  `openclaw/plugin-sdk/tool-plugin` exporteert.
- Een pakketroot die `dist/`, `openclaw.plugin.json` en
  `package.json` bevat.

## Snelstart

```bash
openclaw plugins init stock-quotes --name "Stock Quotes"
cd stock-quotes
npm install
npm run plugin:build
npm run plugin:validate
npm test
```

`plugins init` maakt de volgende basisstructuur:

| Bestand                | Doel                                                              |
| ---------------------- | ----------------------------------------------------------------- |
| `src/index.ts`         | `defineToolPlugin`-ingang met één `echo`-tool                     |
| `src/index.test.ts`    | Metadatatest die de lijst met tools controleert                    |
| `tsconfig.json`        | NodeNext TypeScript-uitvoer naar `dist/`                           |
| `vitest.config.ts`     | Vitest-configuratie voor `src/**/*.test.ts`                        |
| `package.json`         | Scripts, runtimeafhankelijkheden, `openclaw.extensions: ["./dist/index.js"]` |
| `openclaw.plugin.json` | Gegenereerde manifestmetadata voor de eerste tool                  |

`npm run plugin:build` voert `npm run build` (tsc) uit en daarna
`openclaw plugins build --entry ./dist/index.js`. `npm run plugin:validate`
bouwt opnieuw en voert `openclaw plugins validate --entry ./dist/index.js` uit.
Bij een geslaagde validatie verschijnt:

```text
Plugin stock-quotes is geldig.
```

Opties voor `openclaw plugins init <id>`:

| Vlag                 | Standaard           | Effect                                  |
| -------------------- | ------------------ | --------------------------------------- |
| `--directory <path>` | `<id>`             | Uitvoermap                              |
| `--name <name>`      | `<id>` met hoofdletters | Weergavenaam                            |
| `--type <type>`      | `tool`             | Basistype: `tool` of `provider`    |
| `--force`            | uit                | Een bestaande uitvoermap overschrijven |

## Een tool schrijven

`defineToolPlugin` accepteert de identiteit van de Plugin, een optioneel configuratieschema en een
statische lijst met tools. Parameter- en configuratietypen worden afgeleid uit de
TypeBox-schema's.

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "Fetch stock quote snapshots.",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "Quote API key." })),
    baseUrl: Type.Optional(Type.String({ description: "Quote API base URL." })),
  }),
  tools: (tool) => [
    tool({
      name: "stock_quote",
      label: "Stock Quote",
      description: "Fetch a stock quote snapshot.",
      parameters: Type.Object({
        symbol: Type.String({ description: "Ticker symbol, for example OPEN." }),
      }),
      outputSchema: Type.Object(
        {
          symbol: Type.String(),
          configured: Type.Boolean(),
          baseUrl: Type.String(),
        },
        { additionalProperties: false },
      ),
      async execute({ symbol }, config, context) {
        context.signal?.throwIfAborted();
        return {
          symbol: symbol.toUpperCase(),
          configured: Boolean(config.apiKey),
          baseUrl: config.baseUrl ?? "https://api.example.com",
        };
      },
    }),
  ],
});
```

Toolnamen vormen de stabiele API. Kies namen die uniek, in kleine letters en
specifiek genoeg zijn om botsingen met kerntools of andere plugins te voorkomen.

## Optionele tools en factory-tools

Stel `optional: true` in wanneer gebruikers de tool expliciet aan de toelatingslijst moeten toevoegen voordat deze
naar een model wordt verzonden. `openclaw plugins build` schrijft de bijbehorende
`toolMetadata.<tool>.optional`-manifestvermelding, zodat OpenClaw kan zien dat de
tool optioneel is zonder de runtimecode van de Plugin te laden.

```typescript
tool({
  name: "workflow_run",
  description: "Run an external workflow.",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  execute: ({ goal }) => ({ queued: true, goal }),
});
```

Gebruik `factory` wanneer een tool de runtime-toolcontext nodig heeft voordat deze kan worden
gemaakt, bijvoorbeeld om zich voor een specifieke uitvoering af te melden, de sandboxstatus te inspecteren of
runtimehelpers te koppelen. De metadata blijven statisch, ook al wordt de concrete tool
tijdens runtime gebouwd.

```typescript
tool({
  name: "local_workflow",
  description: "Run a local workflow outside sandboxed sessions.",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  factory({ api, toolContext }) {
    if (toolContext.sandboxed) {
      return null;
    }
    return createLocalWorkflowTool(api);
  },
});
```

Factory's declareren nog steeds vooraf een vaste toolnaam. Gebruik `definePluginEntry`
rechtstreeks wanneer de Plugin toolnamen dynamisch berekent of tools combineert
met hooks, services, providers of opdrachten.

## Retourwaarden

`defineToolPlugin` verpakt gewone retourwaarden in de OpenClaw-indeling
voor toolresultaten:

- Retourneer een tekenreeks wanneer het model exact die tekst moet zien.
- Retourneer een JSON-compatibele waarde wanneer je wilt dat het model geformatteerde JSON ziet
  en OpenClaw de oorspronkelijke waarde in `details` bewaart.

```typescript
tool({
  name: "echo_text",
  description: "Echo input text.",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => input,
});
```

```typescript
tool({
  name: "echo_json",
  description: "Echo input as structured JSON.",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => ({ input, length: input.length }),
});
```

Gebruik een factory-tool wanneer je een aangepaste `AgentToolResult` nodig hebt of een
bestaande `api.registerTool`-implementatie wilt hergebruiken.

## Uitvoercontracten

Voeg `outputSchema` toe wanneer een tool stabiele JSON-compatibele gegevens retourneert. Dit beschrijft
de oorspronkelijke waarde die in `AgentToolResult.details` wordt opgeslagen, niet de geformatteerde tekst
in `content`:

```typescript
tool({
  name: "shipment_list",
  description: "List shipments.",
  parameters: Type.Object({
    buyer: Type.Optional(Type.String()),
  }),
  outputSchema: Type.Array(
    Type.Object(
      {
        id: Type.String(),
        buyer: Type.String(),
        paid: Type.Boolean(),
        tons: Type.Number(),
      },
      { additionalProperties: false },
    ),
  ),
  execute: ({ buyer }) => listShipments(buyer),
});
```

[Codemodus](/nl/tools/code-mode) en [Toolzoekfunctie](/nl/tools/tool-search) zetten dit
schema om in een begrensde uitvoerhint in TypeScript-stijl. Daardoor kan een model een bekend resultaat in één programma aanroepen en
transformeren, in plaats van nog een modelbeurt te besteden aan
het observeren van de vorm ervan.

OpenClaw compileert het schema voordat een catalogusaanroep wordt uitgevoerd en valideert daarna de
uiteindelijke `details`-waarde na toolhooks, voordat deze via de bridge wordt geretourneerd.
Met een ongeldig schema kan de tool niet worden uitgevoerd; als het resultaat niet overeenkomt, mislukt de voltooide
aanroep. Neem elke resultaatvariant op die geen uitzondering genereert, inclusief gestructureerde
foutvarianten, of laat het schema weg wanneer het resultaat niet stabiel is. Plaats geen geheimen
of gevoelige waarden in schemabeschrijvingen, omdat vertrouwde uitvoermetadata
zichtbaar kunnen worden voor het model.
Gebruik `{ additionalProperties: false }` op objectlagen wanneer je een volledige,
compacte uitvoerhint wilt; open of afgekorte schema's blijven beschikbaar via
`tools.describe(...)`, maar worden niet als volledige snelindexcontracten aangeboden.

Factory-tools declareren `outputSchema` op de concrete `AnyAgentTool` die ze
retourneren. De statische `tool({ factory })`-declaratie accepteert geen afzonderlijk
uitvoerschema, omdat dit van de runtimetool zou kunnen afwijken.

## Configuratie

`configSchema` is optioneel. Laat dit weg en OpenClaw past een strikt schema voor een leeg object
toe; het gegenereerde manifest bevat nog steeds `configSchema`.

```typescript
export default defineToolPlugin({
  id: "no-config-tools",
  name: "No Config Tools",
  description: "Adds tools that do not need configuration.",
  tools: () => [],
});
```

Met een `configSchema` wordt het tweede `execute`-argument daaruit getypeerd:

```typescript
const configSchema = Type.Object({
  apiKey: Type.String(),
});

export default defineToolPlugin({
  id: "configured-tools",
  name: "Configured Tools",
  description: "Adds configured tools.",
  configSchema,
  tools: (tool) => [
    tool({
      name: "configured_ping",
      description: "Check whether configuration is available.",
      parameters: Type.Object({}),
      execute: (_params, config) => ({ hasKey: config.apiKey.length > 0 }),
    }),
  ],
});
```

OpenClaw leest de Pluginconfiguratie uit de vermelding van de Plugin in de Gateway-configuratie. Codeer
geheimen niet rechtstreeks in broncode of documentatievoorbeelden; gebruik configuratie, omgevingsvariabelen
of SecretRefs volgens het beveiligingsmodel van de Plugin.

## Gegenereerde metadata

OpenClaw moet het Pluginmanifest lezen voordat de runtimecode van de Plugin wordt geïmporteerd.
`defineToolPlugin` stelt hiervoor statische metadata beschikbaar en
`openclaw plugins build` schrijft deze naar het pakket. Voer de generator opnieuw uit nadat
de Plugin-id, naam, beschrijving, het configuratieschema, de activering of toolnamen zijn
gewijzigd:

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

Gegenereerd manifest voor een Plugin met één tool:

```json
{
  "id": "stock-quotes",
  "name": "Stock Quotes",
  "description": "Fetch stock quote snapshots.",
  "version": "0.1.0",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  },
  "activation": {
    "onStartup": true
  },
  "contracts": {
    "tools": ["stock_quote"]
  }
}
```

`contracts.tools` is het belangrijke ontdekkingscontract: dit vertelt OpenClaw welke
Plugin eigenaar is van elke tool, zonder de runtime van elke geïnstalleerde Plugin te laden. Een
verouderd manifest betekent dat een tool bij de ontdekking kan ontbreken, of dat een registratiefout
aan de verkeerde Plugin wordt toegeschreven.

## Pakketmetadata

`openclaw plugins build` stemt ook `package.json` af op de geselecteerde
runtime-ingang:

```json
{
  "type": "module",
  "files": ["dist", "openclaw.plugin.json", "README.md"],
  "dependencies": {
    "typebox": "^1.1.38"
  },
  "peerDependencies": {
    "openclaw": ">=2026.5.17"
  },
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

Lever gebouwde JavaScript (`./dist/index.js`), niet een TypeScript-broningang.
Broningangen werken alleen voor werkruimte-lokale ontwikkeling.

## Valideren in CI

`plugins build --check` mislukt zonder bestanden te herschrijven wanneer gegenereerde metadata
verouderd zijn:

```bash
npm run build
openclaw plugins build --entry ./dist/index.js --check
openclaw plugins validate --entry ./dist/index.js
npm test
```

OpenClaw SDK-compatibiliteitsvelden bevatten TypeScript-annotaties voor `@deprecated`,
die editors als migratiewaarschuwingen tonen. Schakel een typebewuste regel in om ze in CI af te dwingen, zoals
[`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/).
Oxlint is niet typebewust en kan deze annotaties daarom niet afdwingen. De gegenereerde
`plugins init`-basisstructuur voegt daarom geen lintconfiguratie voor afschrijvingen toe.

`plugins validate` controleert of:

- `openclaw.plugin.json` bestaat en doorstaat de normale manifestlader.
- De huidige entry exporteert `defineToolPlugin`-metadata.
- Gegenereerde manifestvelden komen overeen met de entrymetadata.
- `contracts.tools` komt overeen met de gedeclareerde toolnamen.
- `package.json` wijst `openclaw.extensions` naar de geselecteerde runtime-entry.

## Lokaal installeren en inspecteren

Installeer vanuit een afzonderlijke OpenClaw-checkout of geïnstalleerde CLI het pakketpad:

```bash
openclaw plugins install ./stock-quotes
openclaw plugins inspect stock-quotes --runtime
```

Pak voor een rooktest van het pakket eerst het pakket in en installeer de tarball:

```bash
npm pack
openclaw plugins install npm-pack:./openclaw-plugin-stock-quotes-0.1.0.tgz
openclaw plugins inspect stock-quotes --runtime --json
```

Start of herlaad na de installatie de Gateway en vraag de agent de
tool te gebruiken. Als de tool niet zichtbaar is, inspecteer dan de Plugin-runtime en de effectieve
toolcatalogus voordat je code wijzigt (zie [Probleemoplossing](#troubleshooting)).

## Publiceren

Publiceer via ClawHub zodra het pakket gereed is. `clawhub package publish`
accepteert een bron: een lokale map, een GitHub-repository (`owner/repo[@ref]`) of een
tarball-URL.

```bash
clawhub package publish ./stock-quotes --dry-run
clawhub package publish ./stock-quotes
```

Installeer met een expliciete ClawHub-locator:

```bash
openclaw plugins install clawhub:your-org/stock-quotes
```

Losse npm-pakketspecificaties worden tijdens de overgang bij de lancering nog steeds vanuit npm geïnstalleerd, maar
ClawHub is het voorkeursplatform voor het vinden en distribueren van OpenClaw-
plugins. Zie [Publiceren op ClawHub](/nl/clawhub/publishing) voor het eigenaarsbereik en de
releasebeoordeling.

## Probleemoplossing

### `plugin entry not found: ./dist/index.js`

Het geselecteerde entrybestand bestaat niet. Voer `npm run build` uit en voer daarna
`openclaw plugins build --entry ./dist/index.js` of
`openclaw plugins validate --entry ./dist/index.js` opnieuw uit.

### `plugin entry does not expose defineToolPlugin metadata`

De entry exporteerde geen waarde die door `defineToolPlugin` is gemaakt. Controleer of de
standaardexport van de module het resultaat van `defineToolPlugin(...)` is, of geef met
`--entry` de juiste entry door.

### `openclaw.plugin.json generated metadata is stale`

Het manifest komt niet meer overeen met de entrymetadata. Voer uit:

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

Commit zowel de wijzigingen aan `openclaw.plugin.json` als aan `package.json`.

### `package.json openclaw.extensions must include ./dist/index.js`

De pakketmetadata verwijst naar een andere runtime-entry. Voer
`openclaw plugins build --entry ./dist/index.js` uit, zodat de generator de
pakketmetadata afstemt op de entry die je wilt uitbrengen.

### `Cannot find package 'typebox'`

De gebouwde Plugin importeert tijdens runtime `typebox`. Behoud dit in `dependencies`,
installeer opnieuw, bouw opnieuw en voer de validatie opnieuw uit.

### Tool verschijnt niet na installatie

Controleer het volgende in deze volgorde:

1. `openclaw plugins inspect <plugin-id> --runtime`
2. `openclaw plugins validate --root <plugin-root> --entry ./dist/index.js`
3. `openclaw.plugin.json` bevat `contracts.tools` met de verwachte toolnamen.
4. `package.json` bevat `openclaw.extensions: ["./dist/index.js"]`.
5. De Gateway is na de installatie van de Plugin opnieuw gestart of herladen.

## Zie ook

- [Plugins bouwen](/nl/plugins/building-plugins)
- [Plugin-entrypoints](/nl/plugins/sdk-entrypoints)
- [Subpaden van de Plugin-SDK](/nl/plugins/sdk-subpaths)
- [Pluginmanifest](/nl/plugins/manifest)
- [CLI voor plugins](/nl/cli/plugins)
- [Publiceren op ClawHub](/nl/clawhub/publishing)
