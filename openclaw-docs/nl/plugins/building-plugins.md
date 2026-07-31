---
doc-schema-version: 1
read_when:
    - Je wilt een nieuwe OpenClaw-plugin maken
    - Je hebt een snelstartgids voor Plugin-ontwikkeling nodig
    - Je kiest tussen documentatie voor kanalen, providers, CLI-backends, tools of hooks
sidebarTitle: Getting Started
summary: Maak binnen enkele minuten je eerste OpenClaw-plugin
title: Plugins bouwen
x-i18n:
    generated_at: "2026-07-27T05:22:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d156ea305e46d3ca311a0b2cfc42e2c4522f6f10eb70cdd5526d9e9fcd7d4ef
    source_path: plugins/building-plugins.md
    workflow: 16
---

Plugins breiden OpenClaw uit zonder de kern te wijzigen. Een plugin kan een berichtenkanaal, modelprovider, lokale CLI-backend, agenttool, hook, mediaprovider of een andere door de plugin beheerde functionaliteit toevoegen.

Je hoeft geen externe plugin aan de OpenClaw-repository toe te voegen. Publiceer het pakket op [ClawHub](/clawhub), waarna gebruikers het installeren met:

```bash
openclaw plugins install clawhub:<package-name>
```

Kale pakketspecificaties worden tijdens de overgang bij de lancering nog steeds vanaf npm geïnstalleerd. Gebruik het voorvoegsel `clawhub:` als je ClawHub-resolutie wilt.

## Vereisten

- Node 22.22.3+, Node 24.15+ of Node 25.9+, en `npm` of `pnpm`.
- TypeScript ESM-modules.
- Voor werk aan een gebundelde plugin in de repository kloon je de repository en voer je `pnpm install` uit.
  Pluginontwikkeling vanuit een broncheckout werkt alleen met pnpm, omdat OpenClaw
  gebundelde plugins ontdekt via `extensions/*`-werkruimtepakketten.

## Kies de pluginvorm

<CardGroup cols={2}>
  <Card title="Kanaalplugin" icon="messages-square" href="/nl/plugins/sdk-channel-plugins">
    Verbind OpenClaw met een berichtenplatform.
  </Card>
  <Card title="Providerplugin" icon="cpu" href="/nl/plugins/sdk-provider-plugins">
    Voeg een model-, media-, zoek-, ophaal-, spraak- of realtimeprovider toe.
  </Card>
  <Card title="CLI-backendplugin" icon="terminal" href="/nl/plugins/cli-backend-plugins">
    Voer een lokale AI-CLI uit via de modelfallback van OpenClaw.
  </Card>
  <Card title="Toolplugin" icon="wrench" href="/nl/plugins/tool-plugins">
    Registreer agenttools.
  </Card>
</CardGroup>

## Snelstart

Bouw een minimale toolplugin door één verplichte agenttool te registreren. Dit is de
kortste bruikbare pluginvorm en omvat het pakket, het manifest, het toegangspunt en
lokale verificatie.

<Steps>
  <Step title="Pakketmetadata maken">
    <CodeGroup>

```json package.json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "typebox": "1.1.39"
  },
  "peerDependencies": {
    "openclaw": ">=2026.3.24-beta.2"
  },
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2",
      "minGatewayVersion": "2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2",
      "pluginSdkVersion": "2026.3.24-beta.2"
    }
  }
}
```

```json openclaw.plugin.json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Voegt een aangepaste tool toe aan OpenClaw",
  "contracts": {
    "tools": ["my_tool"]
  },
  "activation": {
    "onStartup": true
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

    </CodeGroup>

    Gepubliceerde externe plugins moeten runtime-toegangspunten naar gebouwde JavaScript-
    bestanden laten verwijzen. Zie [SDK-toegangspunten](/nl/plugins/sdk-entrypoints) voor het volledige
    contract voor toegangspunten.

    Elke plugin heeft een manifest nodig, ook zonder configuratie. Runtimetools moeten
    voorkomen in `contracts.tools`, zodat OpenClaw het eigenaarschap kan ontdekken zonder
    elke pluginruntime voortijdig te laden. Stel `activation.onStartup`
    bewust in; dit voorbeeld wordt geladen wanneer de Gateway wordt gestart.

    Ook door de host vertrouwde pluginoppervlakken worden door het manifest afgeschermd en vereisen
    een expliciete declaratie voor geïnstalleerde plugins: `api.registerAgentToolResultMiddleware(...)`
    vereist dat elke doelruntime wordt vermeld in `contracts.agentToolResultMiddleware`,
    en `api.registerTrustedToolPolicy(...)` vereist elke beleids-id in
    `contracts.trustedToolPolicies`. Deze declaraties houden de inspectie tijdens installatie
    en runtimeregistratie op elkaar afgestemd.

    Zie [Pluginmanifest](/nl/plugins/manifest) voor elk manifestveld.

  </Step>

  <Step title="De tool registreren">
    ```typescript index.ts
    import { Type } from "typebox";
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

    export default definePluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Voegt een aangepaste tool toe aan OpenClaw",
      register(api) {
        api.registerTool({
          name: "my_tool",
          description: "Eén invoerwaarde teruggeven",
          parameters: Type.Object({ input: Type.String() }),
          outputSchema: Type.Object(
            { input: Type.String() },
            { additionalProperties: false },
          ),
          async execute(_id, params) {
            const details = { input: params.input };
            return {
              content: [{ type: "text", text: `Ontvangen: ${params.input}` }],
              details,
            };
          },
        });
      },
    });
    ```

    Gebruik `definePluginEntry` voor plugins die geen kanaalplugin zijn. Kanaalplugins gebruiken
    in plaats daarvan `defineChannelPluginEntry` uit `openclaw/plugin-sdk/core`.

  </Step>

  <Step title="De runtime testen">
    Inspecteer voor een geïnstalleerde of externe plugin de geladen runtime:

    ```bash
    openclaw plugins inspect my-plugin --runtime --json
    ```

    Als de plugin een CLI-opdracht registreert, voer je die opdracht ook uit en controleer je
    de uitvoer, bijvoorbeeld `openclaw demo-plugin ping`.

    Voor een gebundelde plugin in deze repository ontdekt OpenClaw pluginpakketten uit
    een broncheckout via de werkruimte `extensions/*`. Voer de meest gerichte
    test uit:

    ```bash
    pnpm test extensions/my-plugin/
    pnpm check
    ```

  </Step>

  <Step title="De pakketinstallatie testen">
    Voordat je een publicatieklare plugin publiceert, test je dezelfde installatievorm die gebruikers
    ontvangen. Voeg eerst een bouwstap toe, laat runtime-toegangspunten zoals
    `openclaw.extensions` verwijzen naar gebouwde JavaScript-bestanden zoals `./dist/index.js`, en zorg
    dat `npm pack` die `dist/`-uitvoer bevat. TypeScript-brontoegangspunten zijn
    alleen bedoeld voor broncheckouts en lokale ontwikkelpaden.

    Pak daarna de plugin in en installeer het tarballbestand met `npm-pack:`:

    ```bash
    npm pack --pack-destination /tmp
    openclaw plugins install npm-pack:/tmp/<plugin-package>.tgz --force
    openclaw plugins inspect my-plugin --runtime --json
    ```

    `npm-pack:` gebruikt het door OpenClaw beheerde npm-project per plugin en detecteert zo
    fouten in runtime-afhankelijkheden die tests vanuit een broncheckout kunnen verbergen. Hiermee wordt
    de pakket- en afhankelijkheidsvorm aangetoond, niet officiële vertrouwensstatus via een catalogus.
    Runtime-imports moeten in `dependencies` of `optionalDependencies` staan;
    afhankelijkheden die alleen in `devDependencies` staan, worden niet geïnstalleerd voor het
    beheerde runtimeproject.

    Gebruik geen onbewerkte archief- of padinstallatie als definitieve verificatie voor officieel of
    bevoorrecht plugingedrag. Onbewerkte bronnen zijn nuttig voor lokale foutopsporing, maar
    tonen niet hetzelfde afhankelijkheidspad aan als installaties via npm of ClawHub. Als
    je plugin afhankelijk is van de vertrouwde status van een officiële plugin, voeg je een tweede verificatie
    toe via een officiële installatie op basis van een catalogus of een gepubliceerd pakketpad dat
    officiële vertrouwensstatus vastlegt. Zie
    [Resolutie van plugin-afhankelijkheden](/nl/plugins/dependency-resolution) voor details over
    de installatieroot en het eigenaarschap van afhankelijkheden.

  </Step>

  <Step title="Publiceren">
    Valideer het pakket voordat je het publiceert:

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    ```

    Canonieke ClawHub-pakketfragmenten staan in `docs/snippets/plugin-publish/`.

  </Step>

  <Step title="Installeren">
    Installeer het gepubliceerde pakket via ClawHub:

    ```bash
    openclaw plugins install clawhub:your-org/your-plugin
    ```

  </Step>
</Steps>

<a id="registering-agent-tools"></a>

## Tools registreren

Tools kunnen verplicht of optioneel zijn. Verplichte tools zijn altijd beschikbaar wanneer de
plugin is ingeschakeld. Voor optionele tools moet de gebruiker expliciet toestemming geven voordat OpenClaw
de bijbehorende pluginruntime laadt.

Toolfabrieken ontvangen vertrouwde runtimecontext, waaronder `deliveryContext`,
`nativeChannelId` voor het actieve platformgesprek wanneer beschikbaar, en
`requesterSenderId`.

```typescript
register(api) {
  api.registerTool(
    {
      name: "workflow_tool",
      description: "Een workflow uitvoeren",
      parameters: Type.Object({ pipeline: Type.String() }),
      outputSchema: Type.Object(
        { pipeline: Type.String() },
        { additionalProperties: false },
      ),
      async execute(_id, params) {
        return {
          content: [{ type: "text", text: params.pipeline }],
          details: { pipeline: params.pipeline },
        };
      },
    },
    { optional: true },
  );
}
```

`outputSchema` is optioneel. Het beschrijft de gestructureerde `details`-waarde die wordt gebruikt door
[Codemodus](/tools/code-mode) en [Toolzoekfunctie](/nl/tools/tool-search). Catalogus-
aanroepen weigeren ongeldige schema's vóór uitvoering en valideren de uiteindelijke waarde na
toolhooks. Laat het weg voor tools zonder een stabiel JSON-resultaat. Zie
[Toolplugins](/nl/plugins/tool-plugins#output-contracts) voor het volledige contract.

Elke tool die met `api.registerTool(...)` wordt geregistreerd, moet ook in het
pluginmanifest worden gedeclareerd:

```json
{
  "contracts": {
    "tools": ["workflow_tool"]
  },
  "toolMetadata": {
    "workflow_tool": {
      "optional": true
    }
  }
}
```

Gebruikers melden zich aan met `tools.allow`:

```json5
{
  tools: { allow: ["workflow_tool"] }, // of ["my-plugin"] voor elke tool van één plugin
}
```

Optionele tools bepalen of een tool aan het model wordt aangeboden. Gebruik
[pluginmachtigingsverzoeken](/nl/plugins/plugin-permission-requests) wanneer een tool
of hook om goedkeuring moet vragen nadat het model deze heeft geselecteerd en voordat de
actie wordt uitgevoerd.

Gebruik optionele tools voor neveneffecten, ongebruikelijke binaire bestanden of functionaliteiten die
niet standaard beschikbaar mogen zijn. Toolnamen mogen niet conflicteren met namen van kerntools;
conflicten worden overgeslagen en gemeld in de plugindiagnostiek. Ongeldige
registraties worden overgeslagen en op dezelfde manier gemeld: een ontbrekende, niet-lege
`name`, een `execute` die geen functie is, of een tooldescriptor zonder een `parameters`-
object.

Toolfabrieken ontvangen een door de runtime geleverd contextobject. Gebruik `ctx.activeModel`
wanneer een tool voor de huidige beurt moet loggen, weergeven of zich moet aanpassen aan het actieve model;
dit kan `provider`, `modelId` en `modelRef` bevatten. Beschouw dit als
informatieve runtimemetadata, niet als beveiligingsgrens tegen de lokale
beheerder, geïnstalleerde plugincode of een gewijzigde OpenClaw-runtime. Gevoelige
lokale tools moeten nog steeds expliciete toestemming op plugin- of beheerdersniveau vereisen en
veilig weigeren wanneer metadata over het actieve model ontbreekt of ongeschikt is.

Het manifest declareert eigenaarschap en ontdekking; bij de uitvoering wordt nog steeds de actieve,
geregistreerde toolimplementatie aangeroepen. Houd `toolMetadata.<tool>.optional: true`
afgestemd op `api.registerTool(..., { optional: true })`, zodat OpenClaw kan voorkomen
dat die pluginruntime wordt geladen totdat de tool expliciet op de toelatingslijst staat.

## Importconventies

Importeer vanuit gerichte SDK-subpaden:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
```

Gebruik binnen je pluginpakket lokale barrelbestanden zoals `api.ts` en
`runtime-api.ts` voor interne imports. Importeer je eigen plugin niet via een
SDK-pad. Providerspecifieke helpers moeten in het providerpakket blijven, tenzij
het koppelvlak echt generiek is.

Aangepaste Gateway-RPC-methoden zijn een geavanceerd toegangspunt. Houd ze op een
pluginspecifiek voorvoegsel; beheerdersnaamruimten van de kern zoals `config.*`,
`exec.approvals.*`, `operator.admin.*`, `wizard.*` en `update.*` blijven gereserveerd
en worden omgezet naar `operator.admin`. De
`openclaw/plugin-sdk/gateway-method-runtime`-brug is gereserveerd voor HTTP-routes van plugins
die `contracts.gatewayMethodDispatch: ["authenticated-request"]` declareren.

Zie [Overzicht van de Plugin-SDK](/nl/plugins/sdk-overview) voor de volledige importkaart.

Compatibiliteitsvelden van de OpenClaw-SDK bevatten TypeScript-annotaties van het type `@deprecated`,
die editors als migratiewaarschuwingen tonen. Om ze tijdens het bouwen af te dwingen,
schakel je een typebewuste regel in, zoals
[`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/).
Oxlint is niet typebewust en kan deze annotaties daarom niet afdwingen.

## Controlelijst vóór indiening

<Check>**package.json** bevat correcte `openclaw`-metadata</Check>
<Check>Het **openclaw.plugin.json**-manifest is aanwezig en geldig</Check>
<Check>Het ingangspunt gebruikt `defineChannelPluginEntry` of `definePluginEntry`</Check>
<Check>Alle imports gebruiken specifieke `plugin-sdk/<subpath>`-paden</Check>
<Check>Interne imports gebruiken lokale modules, geen zelfimports van de SDK</Check>
<Check>Tests slagen (`pnpm test <bundled-plugin-root>/my-plugin/`)</Check>
<Check>`pnpm check` slaagt (plugins in de repository)</Check>

## Testen met bètareleases

1. Houd de releases van [openclaw/openclaw](https://github.com/openclaw/openclaw/releases) in de gaten (`Watch` > `Releases`). Bètatags zien eruit als `v2026.3.N-beta.1`. Je kunt ook [@openclaw](https://x.com/openclaw) volgen op X voor releaseaankondigingen.
2. Test je plugin met de bètatag zodra deze verschijnt. De periode vóór de stabiele release duurt doorgaans slechts enkele uren.
3. Plaats na het testen een bericht in de thread van je plugin in het Discord-kanaal `plugin-forum` ([discord.gg/clawd](https://discord.gg/clawd)), met `all good` of een beschrijving van wat niet meer werkte. Maak een thread als je er nog geen hebt.
4. Als er iets niet meer werkt, open of werk dan een issue bij met de titel `Beta blocker: <plugin-name> - <summary>` en pas het label `beta-blocker` toe. Link het issue in je thread.
5. Open een PR voor `main` met de titel `fix(<plugin-id>): beta blocker - <summary>` en link het issue zowel in de PR als in je Discord-thread. Bijdragers kunnen PR's geen labels geven, dus de titel is voor beheerders en automatisering het signaal aan de PR-zijde. Blokkerende problemen met een PR worden samengevoegd; blokkerende problemen zonder PR worden mogelijk toch uitgebracht.
6. Geen bericht betekent groen licht. Als je deze periode mist, wordt je oplossing doorgaans in de volgende cyclus opgenomen.

## Volgende stappen

<CardGroup cols={2}>
  <Card title="Kanaalplugins" icon="messages-square" href="/nl/plugins/sdk-channel-plugins">
    Bouw een plugin voor een berichtenkanaal
  </Card>
  <Card title="Providerplugins" icon="cpu" href="/nl/plugins/sdk-provider-plugins">
    Bouw een plugin voor een modelprovider
  </Card>
  <Card title="CLI-backendplugins" icon="terminal" href="/nl/plugins/cli-backend-plugins">
    Registreer een lokale AI-CLI-backend
  </Card>
  <Card title="SDK-overzicht" icon="book-open" href="/nl/plugins/sdk-overview">
    Referentie voor de importmap en registratie-API
  </Card>
  <Card title="Runtime-helpers" icon="settings" href="/nl/plugins/sdk-runtime">
    TTS, zoeken en subagent via api.runtime
  </Card>
  <Card title="Testen" icon="test-tubes" href="/nl/plugins/sdk-testing">
    Testhulpmiddelen en patronen
  </Card>
  <Card title="Pluginmanifest" icon="file-json" href="/nl/plugins/manifest">
    Volledige referentie voor het manifestschema
  </Card>
</CardGroup>

## Gerelateerd

- [Plugin-hooks](/nl/plugins/hooks)
- [Pluginarchitectuur](/nl/plugins/architecture)
