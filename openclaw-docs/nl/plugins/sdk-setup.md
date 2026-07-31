---
read_when:
    - Je voegt een installatiewizard toe aan een plugin
    - Je moet het verschil tussen setup-entry.ts en index.ts begrijpen
    - Je definieert configuratieschema's voor plugins of OpenClaw-metadata in package.json
sidebarTitle: Setup and config
summary: Installatiewizards, setup-entry.ts, configuratieschema's en package.json-metadata
title: Plugininstallatie en -configuratie
x-i18n:
    generated_at: "2026-07-27T06:01:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b07e3fa365939fa9c0885b31b7894f5e734313a7deef2297e316956063d97e45
    source_path: plugins/sdk-setup.md
    workflow: 16
---

Naslaginformatie voor Plugin-verpakking (`package.json`-metadata), manifesten (`openclaw.plugin.json`), setup-items en configuratieschema's.

<Tip>
**Op zoek naar een stapsgewijze uitleg?** De how-to-handleidingen behandelen verpakking in context: [Kanaalplugins](/nl/plugins/sdk-channel-plugins#step-1-package-and-manifest) en [Providerplugins](/nl/plugins/sdk-provider-plugins#step-1-package-and-manifest).
</Tip>

## Pakketmetadata

Je `package.json` heeft een `openclaw`-veld nodig dat het pluginsysteem vertelt wat je Plugin biedt:

<Tabs>
  <Tab title="Kanaalplugin">
    ```json
    {
      "name": "@myorg/openclaw-my-channel",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "my-channel",
          "label": "Mijn kanaal",
          "blurb": "Korte beschrijving van het kanaal."
        }
      }
    }
    ```
  </Tab>
  <Tab title="Providerplugin / ClawHub-basis">
    ```json openclaw-clawhub-package.json
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
  </Tab>
</Tabs>

<Note>
Extern publiceren op ClawHub vereist `compat` en `build`. De canonieke publicatiefragmenten staan in `docs/snippets/plugin-publish/`.
</Note>

### `openclaw`-velden

<ParamField path="extensions" type="string[]">
  Entry-pointbestanden (relatief ten opzichte van de pakketroot). Geldige bronitems voor ontwikkeling in een workspace en Git-checkout.
</ParamField>
<ParamField path="runtimeExtensions" type="string[]">
  Gebouwde JavaScript-tegenhangers voor `extensions`, waaraan de voorkeur wordt gegeven wanneer OpenClaw een geïnstalleerd npm-pakket laadt. Zie [SDK-entry-points](/nl/plugins/sdk-entrypoints) voor de oplossingsvolgorde van bron en build.
</ParamField>
<ParamField path="setupEntry" type="string">
  Lichtgewicht entry die alleen voor setup dient (optioneel).
</ParamField>
<ParamField path="runtimeSetupEntry" type="string">
  Gebouwde JavaScript-tegenhanger voor `setupEntry`. Vereist dat `setupEntry` ook is ingesteld.
</ParamField>
<ParamField path="plugin" type="object">
  `{ id, label }`-fallbackidentiteit van de Plugin, gebruikt wanneer een Plugin geen kanaal- of providermetadata heeft waaruit een id of label kan worden afgeleid.
</ParamField>
<ParamField path="channel" type="object">
  Kanaalcatalogusmetadata voor setup-, keuze-, snelstart- en statusoppervlakken.
</ParamField>
<ParamField path="install" type="object">
  Installatiehints: `npmSpec`, `localPath`, `defaultChoice`, `minHostVersion`, `expectedIntegrity`, `allowInvalidConfigRecovery`, `requiredPlatformPackages`.
</ParamField>
<ParamField path="startup" type="object">
  Vlaggen voor opstartgedrag.
</ParamField>
<ParamField path="compat" type="object">
  `pluginApi`-versiebereik dat deze Plugin ondersteunt. Vereist voor externe publicaties op ClawHub.
</ParamField>

<Note>
Provider-id's (`providers: string[]`) zijn manifestmetadata, geen pakketmetadata. Declareer ze in `openclaw.plugin.json`, niet hier — zie [Pluginmanifest](/nl/plugins/manifest).
</Note>

### `openclaw.channel`

`openclaw.channel` is goedkope pakketmetadata voor kanaaldetectie en setup-oppervlakken voordat de runtime wordt geladen.

### Setupvelden in beheer van het kanaal

Kanaalplugins moeten setupvelden eenmaal definiëren in runtimecode met `defineChannelSetupContract(...)` en de bijbehorende serialiseerbare projectie publiceren onder `openclaw.channel.setup.fields`. De runtimedefinitie leidt het invoertype af dat lokaal is voor de Plugin, parseert zowel begeleide als niet-interactieve waarden en houdt kanaalspecifieke sleutels buiten kerntypen. Met pakketmetadata kunnen `openclaw channels add <channel-id> --help` en `openclaw channels add --channel <channel-id> --help` alleen de opties van het geselecteerde kanaal vinden zonder de Plugin te laden.

```ts
import { defineChannelSetupContract } from "openclaw/plugin-sdk/channel-setup";

export const setupContract = defineChannelSetupContract({
  fields: {
    endpoint: {
      kind: "string",
      cli: { flags: "--endpoint <url>", description: "Service-eindpunt" },
    },
    transport: {
      kind: "choice",
      choices: ["native", "container"],
      cli: { flags: "--transport <kind>", description: "Transportbeheerder" },
    },
  },
  adapter: {
    applyAccountConfig: ({ cfg, input }) => ({
      ...cfg,
      channels: { ...cfg.channels, example: input },
    }),
  },
});
```

```json
{
  "openclaw": {
    "channel": {
      "id": "example",
      "setup": {
        "fields": [
          {
            "key": "endpoint",
            "kind": "string",
            "cli": { "flags": "--endpoint <url>", "description": "Service-eindpunt" }
          },
          {
            "key": "transport",
            "kind": "choice",
            "choices": ["native", "container"],
            "cli": { "flags": "--transport <kind>", "description": "Transportbeheerder" }
          }
        ]
      }
    }
  }
}
```

Ondersteunde veldtypen zijn `string`, `boolean`, `integer`, `string-list` en `choice`. Gebruik `sensitive: true` voor aanmeldgegevens. Elke veldsleutel moet gelijk zijn aan de camelCase-attribuutnaam van de lange CLI-vlag, inclusief een eventuele ontkennende vorm, zoals `apiToken` voor `--api-token`. Booleaanse velden kunnen `cli.negatedFlags` toevoegen wanneer zowel positieve als `--no-*`-vormen nodig zijn. `channel`, `account` en de accountweergave `name` blijven de gedeelde besturingsenvelop.

De uitgebrachte `setup`/`ChannelSetupInput`-adapter blijft beschikbaar voor bestaande externe plugins. Nieuwe plugins moeten `setupContract` beschikbaar stellen; OpenClaw geeft hier altijd de voorkeur aan wanneer beide aanwezig zijn.

| Veld                                   | Type       | Betekenis                                                                     |
| -------------------------------------- | ---------- | ----------------------------------------------------------------------------- |
| `id`                                   | `string`   | Canonieke kanaal-id.                                                          |
| `label`                                | `string`   | Primair kanaallabel.                                                          |
| `selectionLabel`                       | `string`   | Keuze-/setuplabel wanneer dit moet verschillen van `label`.                  |
| `detailLabel`                          | `string`   | Secundair detaillabel voor uitgebreidere kanaalcatalogi en statusoppervlakken. |
| `docsPath`                             | `string`   | Documentatiepad voor setup- en selectielinks.                                 |
| `docsLabel`                            | `string`   | Overschrijvend label voor documentatielinks wanneer dit moet verschillen van de kanaal-id. |
| `blurb`                                | `string`   | Korte beschrijving voor onboarding/catalogus.                                 |
| `order`                                | `number`   | Sorteervolgorde in kanaalcatalogi.                                             |
| `aliases`                              | `string[]` | Extra opzoekaliassen voor kanaalselectie.                                      |
| `preferOver`                           | `string[]` | Plugin-/kanaal-id's met lagere prioriteit die dit kanaal moet overtreffen.     |
| `systemImage`                          | `string`   | Optionele pictogram-/systeemafbeeldingsnaam voor kanaalcatalogi in de UI.      |
| `selectionDocsPrefix`                  | `string`   | Voorvoegseltekst vóór documentatielinks in selectieoppervlakken.               |
| `selectionDocsOmitLabel`               | `boolean`  | Toon het documentatiepad rechtstreeks in plaats van een gelabelde documentatielink in selectietekst. |
| `selectionExtras`                      | `string[]` | Extra korte tekenreeksen die aan selectietekst worden toegevoegd.              |
| `markdownCapable`                      | `boolean`  | Markeert het kanaal als geschikt voor Markdown voor beslissingen over uitgaande opmaak. |
| `exposure`                             | `object`   | Zichtbaarheidsinstellingen voor het kanaal in setup-, geconfigureerde-lijst- en documentatieoppervlakken. |
| `quickstartAllowFrom`                  | `boolean`  | Laat dit kanaal deelnemen aan de standaard snelstart-setupflow `allowFrom`.       |
| `forceAccountBinding`                  | `boolean`  | Vereis expliciete accountkoppeling, zelfs wanneer er slechts één account bestaat. |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | Geef de voorkeur aan sessieopzoeking bij het oplossen van aankondigingsdoelen voor dit kanaal. |
| `setup`                                | `object`   | Serialiseerbare setupvelden in beheer van het kanaal voor luie detectie van CLI-opties. |

Voorbeeld:

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "Mijn kanaal",
      "selectionLabel": "Mijn kanaal (zelf gehost)",
      "detailLabel": "Mijn kanaalbot",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Zelfgehoste chatintegratie op basis van Webhooks.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "Handleiding:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure` ondersteunt:

- `configured`: neem het kanaal op in geconfigureerde/statusachtige lijstweergaven
- `setup`: neem het kanaal op in interactieve setup-/configuratiekeuzelijsten
- `docs`: markeer het kanaal als openbaar zichtbaar in documentatie-/navigatieoppervlakken

### `openclaw.install`

`openclaw.install` is pakketmetadata, geen manifestmetadata.

| Veld                         | Type                                | Wat het betekent                                                                  |
| ---------------------------- | ----------------------------------- | --------------------------------------------------------------------------------- |
| `clawhubSpec`                | `string`                            | Canonieke ClawHub-specificatie voor installatie/bijwerken en install-on-demand-flows tijdens onboarding. |
| `npmSpec`                    | `string`                            | Canonieke npm-specificatie voor terugvalflows bij installatie/bijwerken.          |
| `localPath`                  | `string`                            | Lokaal ontwikkelpad of gebundeld installatiepad.                                  |
| `defaultChoice`              | `"clawhub"` \| `"npm"` \| `"local"` | Voorkeursinstallatiebron wanneer meerdere bronnen beschikbaar zijn.               |
| `minHostVersion`             | `string`                            | Minimaal ondersteunde OpenClaw-versie, `>=x.y.z` of `>=x.y.z-prerelease`.        |
| `expectedIntegrity`          | `string`                            | Verwachte npm-dist-integriteitstekenreeks, doorgaans `sha512-...`, voor vastgezette installaties. |
| `allowInvalidConfigRecovery` | `boolean`                           | Hiermee kunnen herinstallatieflows voor gebundelde plugins herstellen van specifieke fouten door verouderde configuratie. |
| `requiredPlatformPackages`   | `string[]`                          | Vereiste platformspecifieke npm-aliassen die tijdens de npm-installatie worden geverifieerd. |

<AccordionGroup>
  <Accordion title="Onboardinggedrag">
    Interactieve onboarding gebruikt `openclaw.install` voor install-on-demand-oppervlakken: als jouw plugin vóór het laden van de runtime keuzes voor providerauthenticatie of metadata voor kanaalconfiguratie/-catalogi beschikbaar stelt, kan onboarding vragen om installatie via ClawHub, npm of een lokale bron, de plugin installeren of inschakelen en daarna doorgaan met de geselecteerde flow. ClawHub-keuzes gebruiken `clawhubSpec` en hebben de voorkeur wanneer ze aanwezig zijn; npm-keuzes vereisen vertrouwde catalogusmetadata met een register-`npmSpec` (exacte versies en `expectedIntegrity` zijn optionele vastzettingen die, indien ingesteld, bij installatie/bijwerken worden afgedwongen). Bewaar „wat moet worden weergegeven” in `openclaw.plugin.json` en „hoe het moet worden geïnstalleerd” in `package.json`.
  </Accordion>
  <Accordion title="Afdwinging van minHostVersion">
    Als `minHostVersion` is ingesteld, wordt dit zowel bij installatie als bij het laden van niet-gebundelde manifestregisters afgedwongen. Oudere hosts slaan externe plugins over; ongeldige versietekenreeksen worden geweigerd. Van gebundelde bronplugins wordt aangenomen dat ze dezelfde versie hebben als de hostcheckout.
  </Accordion>
  <Accordion title="Vastgezette npm-installaties">
    Bewaar voor vastgezette npm-installaties de exacte versie in `npmSpec` en voeg de verwachte artefactintegriteit toe:

    ```json
    {
      "openclaw": {
        "install": {
          "npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3",
          "expectedIntegrity": "sha512-REPLACE_WITH_NPM_DIST_INTEGRITY",
          "defaultChoice": "npm"
        }
      }
    }
    ```

  </Accordion>
  <Accordion title="Bereik van allowInvalidConfigRecovery">
    `allowInvalidConfigRecovery` is geen algemene omzeiling voor defecte configuraties. Het is uitsluitend bedoeld voor beperkt herstel van gebundelde plugins, zodat herinstallatie/configuratie bekende restanten van upgrades kan herstellen, zoals een ontbrekend pad naar een gebundelde plugin of een verouderde `channels.<id>`-vermelding voor diezelfde plugin. Als de configuratie om andere redenen defect is, mislukt de installatie nog steeds veilig en krijgt de beheerder de instructie om `openclaw doctor --fix` uit te voeren.
  </Accordion>
</AccordionGroup>

### Uitgesteld volledig laden

Kanaalplugins kunnen kiezen voor uitgesteld laden met:

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

Wanneer dit is ingeschakeld, laadt OpenClaw tijdens de opstartfase vóór het luisteren alleen `setupEntry`, zelfs voor reeds geconfigureerde kanalen. De volledige ingang wordt geladen nadat de Gateway is begonnen met luisteren.

<Warning>
Schakel uitgesteld laden alleen in wanneer jouw `setupEntry` alles registreert wat de Gateway nodig heeft voordat deze begint met luisteren (kanaalregistratie, HTTP-routes, Gateway-methoden). Als de volledige ingang vereiste opstartmogelijkheden beheert, behoud dan het standaardgedrag.
</Warning>

Als jouw configuratie-/volledige ingang Gateway-RPC-methoden registreert, plaats deze dan onder een pluginspecifiek voorvoegsel. Gereserveerde kernbeheerdersnaamruimten (`config.*`, `exec.approvals.*`, `wizard.*`, `update.*`) blijven eigendom van de kern en worden altijd genormaliseerd naar `operator.admin`.

## Pluginmanifest

Elke native plugin moet een `openclaw.plugin.json` in de pakketroot bevatten. OpenClaw gebruikt dit om de configuratie te valideren zonder plugincode uit te voeren.

```json
{
  "id": "my-plugin",
  "name": "Mijn plugin",
  "description": "Voegt mogelijkheden van Mijn plugin toe aan OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Geheim voor Webhook-verificatie"
      }
    }
  }
}
```

Voeg voor kanaalplugins `channels` toe (en voor providerplugins `providers`):

```json
{
  "id": "my-channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

Zelfs plugins zonder configuratie moeten een schema bevatten. Een leeg schema is geldig:

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

Zie [Pluginmanifest](/nl/plugins/manifest) voor de volledige schemareferentie.

## Publiceren op ClawHub

Skills en pluginpakketten gebruiken afzonderlijke ClawHub-publicatieopdrachten. Gebruik voor pluginpakketten de pakketspecifieke opdracht:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

<Note>
`clawhub skill publish <path>` is een andere opdracht voor het publiceren van een Skills-map, niet van een pluginpakket. Zie [Publiceren op ClawHub](/nl/clawhub/publishing).
</Note>

## Configuratie-ingang

`setup-entry.ts` is een lichtgewicht alternatief voor `index.ts` dat OpenClaw laadt wanneer alleen configuratieoppervlakken nodig zijn (onboarding, configuratieherstel, inspectie van uitgeschakelde kanalen):

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

Hierdoor wordt tijdens configuratieflows geen zware runtimecode geladen (cryptografische bibliotheken, CLI-registraties, achtergrondservices).

Gebundelde werkruimtekanelen die configuratieveilige exports in nevenmodules bewaren, kunnen `defineBundledChannelSetupEntry(...)` uit `openclaw/plugin-sdk/channel-entry-contract` gebruiken in plaats van `defineSetupPluginEntry(...)`. Dat gebundelde contract ondersteunt ook een optionele `runtime`-export, zodat runtimebedrading tijdens de configuratie lichtgewicht en expliciet kan blijven.

<AccordionGroup>
  <Accordion title="Wanneer OpenClaw setupEntry gebruikt in plaats van de volledige ingang">
    - Het kanaal is uitgeschakeld, maar heeft configuratie-/onboardingoppervlakken nodig.
    - Het kanaal is ingeschakeld, maar niet geconfigureerd.
    - Uitgesteld laden is ingeschakeld (`deferConfiguredChannelFullLoadUntilAfterListen`).

  </Accordion>
  <Accordion title="Wat setupEntry moet registreren">
    - Het kanaalpluginobject (via `defineSetupPluginEntry`).
    - Alle HTTP-routes die nodig zijn voordat de Gateway luistert.
    - Alle Gateway-methoden die tijdens het opstarten nodig zijn.

    Deze Gateway-opstartmethoden moeten nog steeds gereserveerde kernbeheerdersnaamruimten zoals `config.*` of `update.*` vermijden.

  </Accordion>
  <Accordion title="Wat setupEntry NIET mag bevatten">
    - CLI-registraties.
    - Achtergrondservices.
    - Zware runtime-imports (cryptografie, SDK's).
    - Gateway-methoden die pas na het opstarten nodig zijn.

  </Accordion>
</AccordionGroup>

### Gerichte imports van configuratiehulpfuncties

Geef voor veelgebruikte paden die uitsluitend voor configuratie dienen de voorkeur aan de gerichte naden voor configuratiehulpfuncties boven de bredere `plugin-sdk/setup`-paraplu wanneer je slechts een deel van het configuratieoppervlak nodig hebt:

| Importpad                  | Gebruik dit voor                                                                          | Belangrijkste exports                                                                                                                                                                                                                                                                                                  |
| -------------------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime` | runtimehulpfuncties tijdens configuratie die beschikbaar blijven in `setupEntry` / uitgesteld opstarten van kanalen | `createSetupTranslator`, `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-tools`   | CLI-/archief-/documentatiehulpfuncties voor configuratie/installatie                     | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR`                                                                                                                                                                                                         |

Gebruik de bredere `plugin-sdk/setup`-naad wanneer je de volledige gedeelde configuratiegereedschapskist wilt, inclusief hulpfuncties voor configuratiepatches zoals `moveSingleAccountChannelSectionToDefaultAccount(...)`.

Gebruik `createSetupTranslator(...)` voor vaste tekst van de configuratiewizard. Deze gebruikt de eerste niet-lege waarde uit `OPENCLAW_LOCALE`, `LC_ALL`, `LC_MESSAGES` en `LANG`, in die volgorde, en valt vervolgens terug op Engels. Stel `OPENCLAW_LOCALE=en` in voor een expliciete Engelse overschrijving. Bewaar pluginspecifieke configuratietekst in code die eigendom is van de plugin en gebruik gedeelde catalogussleutels alleen voor algemene configuratielabels, statustekst en officiële configuratietekst voor gebundelde plugins.

De adapters voor configuratiepatches blijven bij import veilig voor veelgebruikte paden. Hun opzoekactie voor het contractoppervlak voor gebundelde promotie naar één account is lui, zodat het importeren van `plugin-sdk/setup-runtime` de detectie van gebundelde contractoppervlakken niet voortijdig laadt voordat de adapter daadwerkelijk wordt gebruikt.

### Invoervelden voor configuratie die eigendom zijn van het kanaal

`ChannelSetupInput` is een generieke envelop die wordt gedeeld door configuratieaanroepers en kanaalplugins. De permanent getypeerde velden zijn `name`, `token`, `tokenFile`,
`useEnv`, `allowFrom` en `defaultTo`. Aanvullende sleutels die eigendom zijn van de plugin kunnen nog steeds
aanwezig zijn in het runtime-invoerobject, maar het gedeelde type declareert geen
indexsignatuur. Elke plugin moet zijn eigen configuratievelden declareren en verfijnen of
ze met een schema van de plugin valideren bij de adaptergrens:

```typescript
import type { ChannelSetupAdapter, ChannelSetupInput } from "openclaw/plugin-sdk/channel-setup";

type AcmeSetupInput = ChannelSetupInput & {
  workspaceId?: string;
  webhookUrl?: string;
};

export const acmeSetupAdapter: ChannelSetupAdapter = {
  applyAccountConfig: ({ cfg, input }) => {
    const setupInput = input as AcmeSetupInput;
    return {
      ...cfg,
      channels: {
        ...cfg.channels,
        acme: {
          token: setupInput.token,
          workspaceId: setupInput.workspaceId,
          webhookUrl: setupInput.webhookUrl,
        },
      },
    };
  },
};
```

Kanaalspecifieke velden die eerder rechtstreeks op
`ChannelSetupInput` waren gedeclareerd, blijven tijdelijk getypeerd voor compatibiliteit met externe broncode.
Ze zijn verouderd. Bij een registercontrole op 2026-07-22 van 426 gepubliceerde kanaalplugins van buiten de bronstructuur
werden 21 velden zonder lezers verwijderd en 22 velden met bekende
lezers behouden. Elk behouden veld wordt verwijderd zodra geen enkele gepubliceerde plugin het nog leest;
er is geen versiegrens vereist. Nieuwe en gebundelde plugins mogen niet op deze
laag vertrouwen; declareer de velden waarvan ze eigenaar zijn lokaal.

### Kanaalgestuurde promotie van één account

Wanneer een kanaal een configuratie op het hoogste niveau voor één account opwaardeert naar `channels.<id>.accounts.*`, verplaatst het standaard gedeelde gedrag gepromoveerde waarden met accountbereik naar `accounts.default`.

Elke kanaalplugin kan die promotie uitbreiden of beperken via zijn setupadapter:

- `singleAccountKeysToMove`: extra sleutels op het hoogste niveau die naar het gepromoveerde account moeten worden verplaatst
- `namedAccountPromotionKeys`: wanneer benoemde accounts al bestaan, worden alleen deze sleutels naar het gepromoveerde account verplaatst; gedeelde beleids-/afleveringssleutels blijven op het hoofdniveau van het kanaal
- `resolveSingleAccountPromotionTarget(...)`: kies welk bestaand account gepromoveerde waarden ontvangt

De aanwezigheid van `singleAccountKeysToMove` geeft aan dat het promotiecontract volledig is. Declareer het veld ook wanneer het een lege array is om promotie van verouderde sleutels uit te schakelen. Adapters die het veld weglaten, behouden een door lezers ondersteunde promotielaag van vóór de declaratie voor reeds gepubliceerde plugins. Bij de registercontrole op 2026-07-22 werden 23 sleutels zonder gepubliceerde afhankelijken verwijderd en zes algemene sleutels plus de uitsluitend voor setup bestemde sleutel `rooms` behouden. Elke behouden sleutel wordt verwijderd zodra de gepubliceerde lezers ervan naar declaraties zijn gemigreerd; er is geen versiegrens vereist.

Declareer `openclaw.setupFeatures.configPromotion: true` in het pakketmanifest van de plugin wanneer doctor deze declaraties uit het lichtgewicht gebundelde setup-artefact moet laden. Het uitsluitend voor setup bestemde pluginoppervlak en de volledige kanaalplugin moeten dezelfde declaraties beschikbaar stellen.

Wanneer je `moveSingleAccountChannelSectionToDefaultAccount(...)` aanroept met een reeds opgeloste plugin, geef je de setupadapter ervan door als `setupSurface`. Door de aanroeper aangeleverde setupoppervlakken hebben voorrang op geladen en gebundelde opzoekmechanismen, waardoor plugins met een beperkt bereik of uitsluitend voor setup onafhankelijk blijven van globale registratie.

<Note>
Matrix is het huidige gebundelde voorbeeld. Als er precies één benoemd Matrix-account bestaat, of als `defaultAccount` naar een bestaande niet-canonieke sleutel zoals `Ops` verwijst, behoudt de promotie dat account in plaats van een nieuwe vermelding `accounts.default` te maken.
</Note>

## Configuratieschema

Pluginconfiguratie wordt gevalideerd aan de hand van het JSON Schema in je manifest. Gebruikers configureren plugins via:

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

Je plugin ontvangt deze configuratie tijdens de registratie als `api.pluginConfig`.

Gebruik voor kanaalspecifieke configuratie in plaats daarvan de kanaalconfiguratiesectie:

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### Kanaalconfiguratieschema's bouwen

Gebruik `buildChannelConfigSchema` om een Zod-schema om te zetten in de `ChannelConfigSchema`-wrapper die wordt gebruikt door configuratieartefacten waarvan de plugin eigenaar is:

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

Als je het contract al als JSON Schema of TypeBox opstelt, gebruik je de directe helper zodat OpenClaw de conversie van Zod naar JSON Schema op metadatapaden kan overslaan:

```typescript
import { Type } from "typebox";
import { buildJsonChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const configSchema = buildJsonChannelConfigSchema(
  Type.Object({
    token: Type.Optional(Type.String()),
    allowFrom: Type.Optional(Type.Array(Type.String())),
  }),
);
```

Voor plugins van derden blijft het pluginmanifest het contract voor het koude pad: neem het gegenereerde JSON Schema over in `openclaw.plugin.json#channelConfigs`, zodat configuratieschema-, setup- en UI-oppervlakken `channels.<id>` kunnen inspecteren zonder runtimecode te laden.

## Setupwizards

Kanaalplugins kunnen interactieve setupwizards voor `openclaw onboard` aanbieden. De wizard is een `ChannelSetupWizard`-object op de `ChannelPlugin`:

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

`ChannelSetupWizard` ondersteunt ook `textInputs`, `dmPolicy`, `allowFrom`, `groupAccess`, `prepare`, `finalize` en meer. Zie `src/setup-core.ts` van de Discord-plugin voor een volledig gebundeld voorbeeld.

<AccordionGroup>
  <Accordion title="Gedeelde allowFrom-prompts">
    Geef voor prompts voor DM-toelatingslijsten die alleen de standaardflow `note -> prompt -> parse -> merge -> patch` nodig hebben de voorkeur aan de gedeelde setuphelpers uit `openclaw/plugin-sdk/setup`: `createPromptParsedAllowFromForAccount(...)` en `createTopLevelChannelParsedAllowFromPrompt(...)`.
  </Accordion>
  <Accordion title="Standaardstatus voor kanaalsetup">
    Geef voor statusblokken voor kanaalsetup die alleen verschillen in labels, scores en optionele extra regels de voorkeur aan `createStandardChannelSetupStatus(...)` uit `openclaw/plugin-sdk/setup`, in plaats van in elke plugin hetzelfde `status`-object handmatig te maken.
  </Accordion>
  <Accordion title="Optioneel oppervlak voor kanaalsetup">
    Gebruik voor optionele setupoppervlakken die alleen in bepaalde contexten moeten verschijnen `createOptionalChannelSetupSurface` uit `openclaw/plugin-sdk/channel-setup`:

    ```typescript
    import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

    const setupSurface = createOptionalChannelSetupSurface({
      channel: "my-channel",
      label: "My Channel",
      npmSpec: "@myorg/openclaw-my-channel",
      docsPath: "/channels/my-channel",
    });
    // Returns { setupAdapter, setupWizard }
    ```

    `plugin-sdk/channel-setup` stelt ook de bouwers `createOptionalChannelSetupAdapter(...)` en `createOptionalChannelSetupWizard(...)` van een lager niveau beschikbaar wanneer je slechts één helft van dat optionele installatieoppervlak nodig hebt.

    De gegenereerde optionele adapter/wizard weigert bij echte configuratieschrijfbewerkingen veilig verder te gaan. Ze hergebruiken één bericht dat installatie vereist voor `validateInput`, `applyAccountConfig` en `finalize`, en voegen een documentatielink toe wanneer `docsPath` is ingesteld.

  </Accordion>
  <Accordion title="Door binaire bestanden ondersteunde setuphelpers">
    Geef voor setup-UI's die door binaire bestanden worden ondersteund de voorkeur aan de gedeelde gedelegeerde helpers, in plaats van dezelfde koppeling voor binaire bestanden/status naar elk kanaal te kopiëren:

    - `createDetectedBinaryStatus(...)` voor statusblokken die alleen verschillen in labels, hints, scores en detectie van binaire bestanden
    - `createCliPathTextInput(...)` voor tekstinvoer op basis van paden
    - `createDelegatedSetupWizardProxy(...)` wanneer `setupEntry` status-, voorbereidings- of afrondingsgedrag lui moet doorsturen naar een zwaardere volledige wizard
    - `createDelegatedTextInputShouldPrompt(...)` wanneer `setupEntry` alleen een `textInputs[*].shouldPrompt`-beslissing hoeft te delegeren

  </Accordion>
</AccordionGroup>

## Publiceren en installeren

**Externe plugins:** publiceer naar [ClawHub](/clawhub) en installeer vervolgens:

<Tabs>
  <Tab title="npm">
    ```bash
    openclaw plugins install @myorg/openclaw-my-plugin
    ```

    Kale pakketspecificaties worden tijdens de overgang bij het starten vanaf npm geïnstalleerd, tenzij de naam overeenkomt met de id van een gebundelde of officiële plugin; in dat geval gebruikt OpenClaw in plaats daarvan die lokale/officiële kopie. Gebruik `clawhub:`, `npm:`, `git:` of `npm-pack:` voor deterministische bronselectie — zie [Plugins beheren](/nl/plugins/manage-plugins).

  </Tab>
  <Tab title="Alleen ClawHub">
    ```bash
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```
  </Tab>
  <Tab title="npm-pakketspecificatie">
    Gebruik npm wanneer een pakket nog niet naar ClawHub is verplaatst, of wanneer je tijdens de migratie een
    direct npm-installatiepad nodig hebt:

    ```bash
    openclaw plugins install npm:@myorg/openclaw-my-plugin
    ```

  </Tab>
</Tabs>

**Plugins in de repository:** plaats ze onder de gebundelde pluginwerkruimtestructuur; ze worden tijdens het bouwen automatisch ontdekt.

<Info>
Voor installaties met npm als bron installeert `openclaw plugins install` het pakket in een project per plugin onder `~/.openclaw/npm/projects`, waarbij levenscyclusscripts zijn uitgeschakeld (`--ignore-scripts`). Houd afhankelijkheidsstructuren van plugins volledig in JS/TS en vermijd pakketten waarvoor `postinstall`-builds nodig zijn.
</Info>

<Note>
Bij het starten installeert Gateway geen plugin-afhankelijkheden. De installatieflows voor npm/git/ClawHub beheren het convergeren van afhankelijkheden; voor lokale plugins moeten de afhankelijkheden al zijn geïnstalleerd.
</Note>

Gebundelde pakketmetadata is expliciet en wordt bij het starten van Gateway niet afgeleid van gebouwde JavaScript. Runtime-afhankelijkheden horen thuis in het pluginpakket dat er eigenaar van is; bij het starten herstelt of spiegelt de verpakte OpenClaw nooit plugin-afhankelijkheden.

## Gerelateerd

- [Plugins bouwen](/nl/plugins/building-plugins) — stapsgewijze handleiding om aan de slag te gaan
- [Pluginmanifest](/nl/plugins/manifest) — volledige schemareferentie voor het manifest
- [SDK-ingangspunten](/nl/plugins/sdk-entrypoints) — `definePluginEntry` en `defineChannelPluginEntry`
