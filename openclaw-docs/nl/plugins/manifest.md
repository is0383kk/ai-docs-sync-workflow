---
read_when:
    - Je bouwt een OpenClaw-plugin
    - Je moet een configuratieschema voor een plugin uitbrengen of validatiefouten van een plugin opsporen.
summary: Vereisten voor het Plugin-manifest en JSON-schema (strikte configuratievalidatie)
title: Pluginmanifest
x-i18n:
    generated_at: "2026-07-27T05:07:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 244e5c8265ff79b0ff6e8f4b60c9635cccc3ba66093cecab458676beb9578264
    source_path: plugins/manifest.md
    workflow: 16
---

Deze pagina behandelt het **native OpenClaw-pluginmanifest**, `openclaw.plugin.json`. Zie [Pluginbundels](/nl/plugins/bundles) voor compatibele bundelindelingen (Codex, Claude, Cursor).

Compatibele bundelindelingen gebruiken in plaats daarvan hun eigen manifestbestanden:

- Codex-bundel: `.codex-plugin/plugin.json`
- Claude-bundel: `.claude-plugin/plugin.json`, of de standaardindeling voor Claude-componenten zonder manifest
- Cursor-bundel: `.cursor-plugin/plugin.json`

OpenClaw detecteert deze indelingen automatisch, maar valideert ze niet aan de hand van het onderstaande `openclaw.plugin.json`-schema. Voor een compatibele bundel leest OpenClaw bundelmetadata, gedeclareerde skill-hoofdmappen, Claude-opdrachthoofdmappen, standaardwaarden voor Claude `settings.json`, standaardwaarden voor Claude LSP en ondersteunde hookpakketten, wanneer de indeling overeenkomt met de runtimeverwachtingen van OpenClaw.

Elke native OpenClaw-plugin **moet** `openclaw.plugin.json` in de **hoofdmap van de plugin** meeleveren. OpenClaw leest dit om de configuratie te valideren **zonder plugincode uit te voeren**. Een ontbrekend of ongeldig manifest blokkeert de configuratievalidatie en wordt als een pluginfout beschouwd.

Zie [Plugins](/nl/tools/plugin) voor de volledige handleiding van het pluginsysteem en [Capaciteitsmodel](/nl/plugins/architecture#public-capability-model) voor het native capaciteitsmodel en de huidige richtlijnen voor externe compatibiliteit.

## Wat dit bestand doet

`openclaw.plugin.json` bevat metadata die OpenClaw leest **voordat je plugincode wordt geladen**. Alles daarin moet eenvoudig genoeg zijn om te inspecteren zonder de pluginruntime op te starten.

**Gebruik het voor:**

- pluginidentiteit, configuratievalidatie en aanwijzingen voor de configuratie-UI
- metadata voor authenticatie, onboarding en installatie (alias, automatisch inschakelen, omgevingsvariabelen van providers, authenticatiekeuzes)
- activeringsaanwijzingen voor control-plane-oppervlakken
- eigenaarschap van modelreeksen in verkorte notatie
- statische momentopnamen van capaciteitseigenaarschap (`contracts`)
- gegevenskoppelingen en actiewerkwoorden voor dashboardwidgets
- statische MCP-servers die beschikbaar moeten zijn zolang de plugin is ingeschakeld
- metadata van de QA-runner die de gedeelde `openclaw qa`-host kan inspecteren
- kanaalspecifieke configuratiemetadata die in catalogus- en validatieoppervlakken worden samengevoegd

**Gebruik het niet voor:** het registreren van native runtimehooks, het declareren van toegangspunten voor plugincode of installatiemetadata voor npm. Die horen thuis in je plugincode en `package.json`.

## Minimaal voorbeeld

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## Uitgebreid voorbeeld

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "OpenRouter-providerplugin",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "modelIdNormalization": {
    "providers": {
      "openrouter": {
        "prefixWhenBare": "openrouter"
      }
    }
  },
  "providerEndpoints": [
    {
      "endpointClass": "openrouter",
      "hostSuffixes": ["openrouter.ai"]
    }
  ],
  "providerRequest": {
    "providers": {
      "openrouter": {
        "family": "openrouter"
      }
    }
  },
  "cliBackends": ["openrouter-cli"],
  "syntheticAuthRefs": ["openrouter-cli"],
  "setup": {
    "providers": [
      {
        "id": "openrouter",
        "envVars": ["OPENROUTER_API_KEY"]
      }
    ]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "OpenRouter-API-sleutel",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "OpenRouter-API-sleutel",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "API-sleutel",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## Overzicht van velden op het hoogste niveau

| Veld                                 | Verplicht | Type                         | Betekenis                                                                                                                                                                                                                                                                                      |
| ------------------------------------ | --------- | ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                   | Ja        | `string`           | Canonieke Plugin-id. Dit is de id die wordt gebruikt in `plugins.entries.<id>`.                                                                                                                                                                                                                    |
| `configSchema`                   | Ja        | `object`           | Inline JSON Schema voor de configuratie van deze Plugin.                                                                                                                                                                                                                                       |
| `requiresPlugins`                   | Nee       | `string[]`           | Plugin-id's die ook moeten zijn geïnstalleerd voordat deze Plugin effect heeft. Bij detectie blijft de Plugin laadbaar, maar wordt gewaarschuwd wanneer een vereiste Plugin ontbreekt.                                                                                                          |
| `enabledByDefault`                   | Nee       | `true`           | Markeert een gebundelde Plugin als standaard ingeschakeld. Laat dit weg of stel een andere waarde dan `true` in om de Plugin standaard uitgeschakeld te laten.                                                                                                                      |
| `enabledByDefaultOnPlatforms`                   | Nee       | `string[]`           | Markeert een gebundelde Plugin alleen op de vermelde Node.js-platforms als standaard ingeschakeld, bijvoorbeeld `["darwin"]`. Expliciete configuratie heeft nog steeds voorrang.                                                                                                           |
| `legacyPluginIds`                   | Nee       | `string[]`           | Verouderde id's die worden genormaliseerd naar deze canonieke Plugin-id.                                                                                                                                                                                                                        |
| `autoEnableWhenConfiguredProviders`                   | Nee       | `string[]`           | Provider-id's die deze Plugin automatisch moeten inschakelen wanneer ernaar wordt verwezen in authenticatie, configuratie of modelverwijzingen.                                                                                                                                                |
| `kind`                   | Nee       | `PluginKind \| PluginKind[]`           | Declareert een of meer exclusieve Plugin-typen (`"memory"`, `"context-engine"`) die door `plugins.slots.*` worden gebruikt. Een Plugin die beide posities beheert, declareert beide typen in één array.                                                                                   |
| `channels`                   | Nee       | `string[]`           | Kanaal-id's die door deze Plugin worden beheerd. Wordt gebruikt voor detectie en configuratievalidatie.                                                                                                                                                                                         |
| `providers`                   | Nee       | `string[]`           | Provider-id's die door deze Plugin worden beheerd.                                                                                                                                                                                                                                             |
| `providerCatalogEntry`                   | Nee       | `string`           | Pad naar een lichtgewicht providercatalogusmodule, relatief ten opzichte van de hoofdmap van de Plugin, voor providercatalogusmetadata binnen het manifestbereik die kunnen worden geladen zonder de volledige Plugin-runtime te activeren.                                                       |
| `modelSupport`                   | Nee       | `object`           | Door het manifest beheerde verkorte metadata voor modelfamilies, gebruikt om de Plugin vóór de runtime automatisch te laden.                                                                                                                                                                   |
| `modelCatalog`                   | Nee       | `object`           | Declaratieve modelcatalogusmetadata voor providers die door deze Plugin worden beheerd. Dit is het besturingslaagcontract voor toekomstige alleen-lezenvermeldingen, onboarding, modelkiezers, aliassen en onderdrukking zonder de Plugin-runtime te laden.                                         |
| `modelPricing`                   | Nee       | `object`           | Door de provider beheerd beleid voor het extern opzoeken van prijzen. Gebruik dit om lokale/zelfgehoste providers uit te sluiten van externe prijscatalogi of providerverwijzingen aan OpenRouter/LiteLLM-catalogus-id's te koppelen zonder provider-id's hard te coderen in de kern.                 |
| `modelIdNormalization`                   | Nee       | `object`           | Door de provider beheerde opschoning van model-id-aliassen/-voorvoegsels die moet worden uitgevoerd voordat de providerruntime wordt geladen.                                                                                                                                                   |
| `providerEndpoints`                   | Nee       | `object[]`           | Door het manifest beheerde host-/baseUrl-metadata voor providerroutes die door de kern moeten worden geclassificeerd voordat de providerruntime wordt geladen.                                                                                                                                 |
| `providerRequest`                   | Nee       | `object`           | Goedkoop te verwerken metadata over providerfamilies en aanvraagcompatibiliteit die door algemeen aanvraagbeleid wordt gebruikt voordat de providerruntime wordt geladen.                                                                                                                       |
| `secretProviderIntegrations`                   | Nee       | `Record<string, object>`           | Declaratieve voorinstellingen voor SecretRef-execproviders die door installatie- of configuratie-interfaces kunnen worden aangeboden zonder providerspecifieke integraties hard te coderen in de kern.                                                                                           |
| `cliBackends`                   | Nee       | `string[]`           | Id's van CLI-inferentiebackends die door deze Plugin worden beheerd. Wordt gebruikt voor automatische activering bij het opstarten op basis van expliciete configuratieverwijzingen.                                                                                                             |
| `syntheticAuthRefs`                   | Nee       | `string[]`           | Provider- of CLI-backendverwijzingen waarvan de door de Plugin beheerde synthetische authenticatiehook moet worden getest tijdens koude modeldetectie voordat de runtime wordt geladen.                                                                                                         |
| `nonSecretAuthMarkers`                   | Nee       | `string[]`           | Door gebundelde Plugins beheerde tijdelijke API-sleutelwaarden die niet-geheime lokale, OAuth- of omgevingsreferentiestatus vertegenwoordigen.                                                                                                                                                   |
| `commandAliases`                   | Nee       | `object[]`           | Opdrachtnamen die door deze Plugin worden beheerd en vóór het laden van de runtime Plugin-bewuste configuratie- en CLI-diagnostiek moeten opleveren.                                                                                                                                            |
| `providerUsageAuthEnvVars`                   | Nee       | `Record<string, string[]>`           | Providerreferenties die uitsluitend voor gebruik/facturering dienen. OpenClaw gebruikt deze namen voor gebruiksdetectie en het verwijderen van geheimen, maar nooit voor inferentie-authenticatie.                                                                                               |
| `providerAuthAliases`                   | Nee       | `Record<string, string>`           | Provider-id's die voor het opzoeken van authenticatie een andere provider-id moeten hergebruiken, bijvoorbeeld een programmeerprovider die de API-sleutel en authenticatieprofielen van de basisprovider deelt.                                                                                  |
| `providerAuthChoices`                   | Nee       | `object[]`           | Goedkoop te verwerken metadata voor authenticatiekeuzes voor onboardingkiezers, het bepalen van de voorkeursprovider en eenvoudige koppeling van CLI-vlaggen.                                                                                                                                  |
| `activation`                   | Nee       | `object`           | Goedkoop te verwerken metadata voor de activeringsplanner voor laden dat wordt geactiveerd door opstarten, providers, opdrachten, kanalen, routes en mogelijkheden. Alleen metadata; de Plugin-runtime blijft verantwoordelijk voor het daadwerkelijke gedrag.                                     |
| `setup`                   | Nee       | `object`           | Goedkoop te verwerken configuratie-/onboardingbeschrijvingen die detectie- en configuratie-interfaces kunnen inspecteren zonder de Plugin-runtime te laden.                                                                                                                                    |
| `qaRunners`                   | Nee       | `object[]`           | Goedkoop te verwerken QA-runnerbeschrijvingen die door de gedeelde `openclaw qa`-host worden gebruikt voordat de Plugin-runtime wordt geladen.                                                                                                                                             |
| `dashboard`                   | Nee       | `object`           | Gegevensbindingen en actiewerkwoorden voor dashboardwidgets. Elke vermelding wordt gevalideerd aan de hand van een Gateway-methode die door deze Plugin is geregistreerd met het vereiste lees- of schrijfbereik. Zie de [dashboardreferentie](#dashboard-reference).                               |
| `mcpServers`                         | Nee       | `Record<string, object>`     | Statische MCP-serverdefinities die beschikbaar worden gesteld wanneer deze plugin is ingeschakeld. Relatieve opdrachtargumenten en werkmappen worden bepaald vanaf de pluginhoofdmap. Vermeldingen van de operator in `mcp.servers` overschrijven of schakelen definities met dezelfde naam uit. Zie de [MCP-serverreferentie](#mcp-server-reference). |
| `contracts`                          | Nee       | `object`                     | Statische momentopname van het eigenaarschap van mogelijkheden voor externe authenticatiehooks, embeddings, spraak, realtime transcriptie, realtime spraak, mediabegrip, generatie van afbeeldingen/video/muziek, ophalen van webinhoud, zoeken op het web, workerproviders, extractie van document-/webinhoud en eigenaarschap van tools.                     |
| `configContracts`                    | Nee       | `object`                     | Configuratiegedrag dat eigendom is van het manifest en wordt gebruikt door generieke corehelpers: detectie van gevaarlijke vlaggen, migratiedoelen voor SecretRef en beperking van verouderde configuratiepaden. Zie de [configContracts-referentie](#configcontracts-reference).                                                                         |
| `mediaUnderstandingProviderMetadata` | Nee       | `Record<string, object>`     | Goedkope standaardinstellingen voor mediabegrip voor provider-id's die zijn gedeclareerd in `contracts.mediaUnderstandingProviders`.                                                                                                                                                                                       |
| `imageGenerationProviderMetadata`    | Nee       | `Record<string, object>`     | Goedkope authenticatiemetadata voor afbeeldingsgeneratie voor provider-id's die zijn gedeclareerd in `contracts.imageGenerationProviders`, inclusief authenticatiealiassen die eigendom zijn van de provider en controles voor basis-URL's.                                                                                                                             |
| `videoGenerationProviderMetadata`    | Nee       | `Record<string, object>`     | Goedkope authenticatiemetadata voor videogeneratie voor provider-id's die zijn gedeclareerd in `contracts.videoGenerationProviders`, inclusief authenticatiealiassen die eigendom zijn van de provider en controles voor basis-URL's.                                                                                                                             |
| `musicGenerationProviderMetadata`    | Nee       | `Record<string, object>`     | Goedkope authenticatiemetadata voor muziekgeneratie voor provider-id's die zijn gedeclareerd in `contracts.musicGenerationProviders`, inclusief authenticatiealiassen die eigendom zijn van de provider en controles voor basis-URL's.                                                                                                                             |
| `toolMetadata`                       | Nee       | `Record<string, object>`     | Goedkope beschikbaarheidsmetadata voor tools die eigendom zijn van de plugin en zijn gedeclareerd in `contracts.tools`. Gebruik deze wanneer een tool de runtime niet mag laden tenzij er bewijs uit configuratie, omgevingsvariabelen of authenticatie beschikbaar is.                                                                                                                      |
| `channelConfigs`                     | Nee       | `Record<string, object>`     | Kanaalconfiguratiemetadata die eigendom is van het manifest en vóór het laden van de runtime wordt samengevoegd in oppervlakken voor detectie en validatie.                                                                                                                                                                                     |
| `skills`                             | Nee       | `string[]`                   | Te laden Skills-mappen, relatief ten opzichte van de pluginhoofdmap.                                                                                                                                                                                                                                        |
| `name`                               | Nee       | `string`                     | Voor mensen leesbare pluginnaam.                                                                                                                                                                                                                                                                    |
| `description`                        | Nee       | `string`                     | Korte samenvatting die op pluginoppervlakken wordt weergegeven.                                                                                                                                                                                                                                                        |
| `catalog`                            | Nee       | `object`                     | Optionele presentatietips voor plugincatalogusoppervlakken. Deze metadata installeert of activeert een plugin niet en kent er geen vertrouwen aan toe.                                                                                                                                                                   |
| `icon`                               | Nee       | `string`                     | HTTPS-afbeeldings-URL voor marketplace-/cataloguskaarten. ClawHub accepteert elke geldige `https://`-URL en valt terug op het standaardpictogram van de plugin wanneer deze is weggelaten of ongeldig is.                                                                                                                             |
| `version`                            | Nee       | `string`                     | Informatieve pluginversie.                                                                                                                                                                                                                                                                  |
| `uiHints`                            | Nee       | `Record<string, object>`     | UI-labels, tijdelijke aanduidingen en gevoeligheidstips voor configuratievelden.                                                                                                                                                                                                                              |

## Naslaginformatie voor MCP-servers

`mcpServers` laat een native plugin een MCP-server leveren, inclusief een MCP App, zonder dat beheerders de statische procesdefinitie ervan hoeven te dupliceren in `openclaw.json`:

```json
{
  "mcpServers": {
    "example": {
      "transport": "stdio",
      "command": "node",
      "args": ["./mcp-server.js"]
    }
  }
}
```

OpenClaw neemt deze servers alleen op zolang de bijbehorende plugin is ingeschakeld. Relatieve paden voor `command`, `args`, `cwd` en `workingDirectory` worden herleid vanaf de hoofdmap van de plugin. De gebruikersconfiguratie blijft leidend: `mcp.servers.<name>` kan een standaardwaarde van een plugin vervangen of `enabled: false` instellen om deze weg te laten. Voor het renderen van MCP Apps en het aanroepen van servertools zijn nog steeds de normale instelling voor MCP Apps en het geldende toolbeleid vereist; het declareren van een server omzeilt geen van beide grenzen.

## Naslaginformatie voor het dashboard

`dashboard` laat een ingeschakelde plugin bestaande Gateway-RPC's beschikbaar stellen aan dashboardwidgets met de juiste rechten, zonder pluginbeleid aan de kern toe te voegen. Gegevensbindingen moeten een methode noemen die dezelfde plugin registreert met `operator.read`; actiewerkwoorden moeten een methode noemen die de plugin registreert met `operator.write`. Bij een verschil wordt de plugin tijdens de registratie geweigerd.

```json
{
  "dashboard": {
    "dataBindings": [
      {
        "id": "items.list",
        "method": "example.items.list",
        "description": "Voorbeelditems weergeven."
      }
    ],
    "actionVerbs": [
      {
        "id": "refresh",
        "method": "example.items.refresh",
        "description": "Voorbeelditems vernieuwen.",
        "paramShape": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "force": { "type": "boolean" }
          }
        }
      }
    ]
  }
}
```

De manifest-id's zijn lokaal voor de plugin. Widgetrechten gebruiken `<plugin-id>.<id>`, zoals `example.items.list` en `example.refresh`. Om de persistente naamruimte voor rechten ondubbelzinnig te houden, escapt OpenClaw `%` en `.` in het plugin-id-segment als `%25` en `%2E`; gewone plugin-id's behouden de natuurlijke vorm. `paramShape` is een optioneel JSON Schema dat wordt toegepast op het object met actieparameters voordat OpenClaw de RPC van de plugin aanroept.

## Naslaginformatie voor de catalogus

`catalog` biedt optionele weergaveaanwijzingen voor pluginbrowsers. Hosts mogen deze aanwijzingen negeren. Ze installeren of activeren de plugin nooit en veranderen het runtimegedrag of vertrouwensniveau ervan niet.

```json
{
  "catalog": {
    "featured": true,
    "order": 10
  }
}
```

| Veld       | Type      | Betekenis                                                                  |
| ---------- | --------- | -------------------------------------------------------------------------- |
| `featured` | `boolean` | Of catalogusweergaven deze plugin moeten uitlichten.                       |
| `order`    | `number`  | Oplopende weergaveaanwijzing voor samengestelde plugins; lagere waarden verschijnen eerder. |

## Naslaginformatie voor metagegevens van generatieproviders

De metagegevensvelden voor generatieproviders beschrijven statische authenticatiesignalen voor providers die zijn gedeclareerd in de bijbehorende lijst `contracts.*GenerationProviders`. OpenClaw leest deze velden voordat de providerruntime wordt geladen, zodat kerntools kunnen bepalen of een generatieprovider beschikbaar is zonder elke providerplugin te importeren.

Gebruik deze velden alleen voor eenvoudige, declaratieve feiten. Transport, aanvraagtransformaties, tokenvernieuwing, validatie van aanmeldgegevens en het daadwerkelijke generatiegedrag blijven in de pluginruntime.

```json
{
  "contracts": {
    "imageGenerationProviders": ["example-image"]
  },
  "imageGenerationProviderMetadata": {
    "example-image": {
      "aliases": ["example-image-oauth"],
      "authProviders": ["example-image"],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example-image.config",
          "overlayPath": "image",
          "mode": {
            "path": "mode",
            "default": "local",
            "allowed": ["local"]
          },
          "requiredAny": ["workflow", "workflowPath"],
          "required": ["promptNodeId"]
        }
      ],
      "authSignals": [
        {
          "provider": "example-image"
        },
        {
          "provider": "example-image-oauth",
          "providerBaseUrl": {
            "provider": "example-image",
            "defaultBaseUrl": "https://api.example.com/v1",
            "allowedBaseUrls": ["https://api.example.com/v1"]
          }
        }
      ]
    }
  }
}
```

Elke metagegevensvermelding ondersteunt:

| Veld                   | Vereist | Type       | Betekenis                                                                                                                                           |
| ---------------------- | -------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `aliases`              | Nee      | `string[]` | Aanvullende provider-id's die als statische authenticatie-aliassen voor de generatieprovider moeten gelden.                                         |
| `authProviders`        | Nee      | `string[]` | Provider-id's waarvan de geconfigureerde authenticatieprofielen als authenticatie voor deze generatieprovider moeten gelden.                       |
| `configSignals`        | Nee      | `object[]` | Eenvoudige beschikbaarheidssignalen op basis van alleen configuratie voor lokale of zelfgehoste providers die zonder authenticatieprofielen of omgevingsvariabelen kunnen worden geconfigureerd. |
| `authSignals`          | Nee      | `object[]` | Expliciete authenticatiesignalen. Wanneer deze aanwezig zijn, vervangen ze de standaardset signalen van de provider-id, `aliases` en `authProviders`. |
| `referenceAudioInputs` | Nee      | `boolean`  | Alleen voor videogeneratie. Stel in op `true` wanneer de provider referentieaudio-assets accepteert; anders verbergt `video_generate` parameters voor audioreferenties. |

Elke vermelding in `configSignals` ondersteunt:

| Veld             | Vereist | Type       | Betekenis                                                                                                                                                                                 |
| ---------------- | -------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `rootPath`       | Ja       | `string`   | Puntpad naar het configuratieobject dat eigendom is van de plugin en moet worden geïnspecteerd, bijvoorbeeld `plugins.entries.example.config`.                                                          |
| `overlayPath`    | Nee      | `string`   | Puntpad binnen de hoofdconfiguratie waarvan het object over het hoofdobject moet worden gelegd voordat het signaal wordt geëvalueerd. Gebruik dit voor mogelijkhedenpecifieke configuratie zoals `image`, `video` of `music`. |
| `overlayMapPath` | Nee      | `string`   | Puntpad binnen de hoofdconfiguratie waarvan elk object met waarden over het hoofdobject moet worden gelegd. Gebruik dit voor benoemde accounttoewijzingen zoals `accounts`, waarbij elk geconfigureerd account moet voldoen. |
| `required`       | Nee      | `string[]` | Puntpaden binnen de effectieve configuratie die geconfigureerde waarden moeten bevatten. Tekenreeksen mogen niet leeg zijn; objecten en arrays mogen niet leeg zijn.                       |
| `requiredAny`    | Nee      | `string[]` | Puntpaden binnen de effectieve configuratie waarvan er ten minste één een geconfigureerde waarde moet bevatten.                                                                           |
| `mode`           | Nee      | `object`   | Optionele tekenreeksmodusbeperking binnen de effectieve configuratie. Gebruik deze wanneer beschikbaarheid op basis van alleen configuratie slechts op één modus van toepassing is.       |

Elke beperking in `mode` ondersteunt:

| Veld         | Vereist | Type       | Betekenis                                                                          |
| ------------ | -------- | ---------- | ---------------------------------------------------------------------------------- |
| `path`       | Nee      | `string`   | Puntpad binnen de effectieve configuratie. Standaardwaarde is `mode`.  |
| `default`    | Nee      | `string`   | Moduswaarde die wordt gebruikt wanneer het pad ontbreekt in de configuratie.       |
| `allowed`    | Nee      | `string[]` | Indien aanwezig slaagt het signaal alleen wanneer de effectieve modus een van deze waarden is. |
| `disallowed` | Nee      | `string[]` | Indien aanwezig mislukt het signaal wanneer de effectieve modus een van deze waarden is. |

Elke vermelding in `authSignals` ondersteunt:

| Veld              | Vereist | Type     | Betekenis                                                                                                                                                                     |
| ----------------- | -------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | Ja       | `string` | Provider-id die moet worden gecontroleerd in geconfigureerde authenticatieprofielen.                                                                                          |
| `providerBaseUrl` | Nee      | `object` | Optionele beperking waardoor het signaal alleen meetelt wanneer de geconfigureerde provider waarnaar wordt verwezen een toegestane basis-URL gebruikt. Gebruik dit wanneer een authenticatie-alias alleen voor bepaalde API's geldig is. |

Elke beperking in `providerBaseUrl` ondersteunt:

| Veld              | Vereist | Type       | Betekenis                                                                                                                                            |
| ----------------- | -------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | Ja       | `string`   | Providerconfiguratie-id waarvan `baseUrl` moet worden gecontroleerd.                                                                        |
| `defaultBaseUrl`  | Nee      | `string`   | Basis-URL waarvan moet worden uitgegaan wanneer `baseUrl` ontbreekt in de providerconfiguratie.                                              |
| `allowedBaseUrls` | Ja       | `string[]` | Toegestane basis-URL's voor dit authenticatiesignaal. Het signaal wordt genegeerd wanneer de geconfigureerde of standaard basis-URL niet overeenkomt met een van deze genormaliseerde waarden. |

## Naslaginformatie voor toolmetagegevens

`toolMetadata` gebruikt dezelfde vormen `configSignals` en `authSignals` als de metagegevens voor generatieproviders, geïndexeerd op toolnaam. `contracts.tools` declareert het eigenaarschap. `toolMetadata` declareert eenvoudig beschikbaarheidsbewijs, zodat OpenClaw kan voorkomen dat een pluginruntime alleen wordt geïmporteerd om de toolfactory `null` te laten retourneren.

```json
{
  "setup": {
    "providers": [
      {
        "id": "example",
        "envVars": ["EXAMPLE_API_KEY"]
      }
    ]
  },
  "contracts": {
    "tools": ["example_search"]
  },
  "toolMetadata": {
    "example_search": {
      "authSignals": [
        {
          "provider": "example"
        }
      ],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example.config",
          "overlayPath": "search",
          "required": ["apiKey"]
        }
      ]
    }
  }
}
```

`toolMetadata`-vermeldingen accepteren naast de gedeelde velden `configSignals`/`authSignals` hierboven ook `optional` (markeert de tool als niet vereist voor activering van de plugin) en `replaySafe` (markeert de uitvoering van de tool als veilig om te herhalen na een onvolledige modelbeurt).

Als een tool geen `toolMetadata` heeft, behoudt OpenClaw het bestaande gedrag en laadt het de bijbehorende plugin wanneer het toolcontract overeenkomt met het beleid. Voor tools in het kritieke pad waarvan de factory afhankelijk is van authenticatie/configuratie, moeten pluginauteurs `toolMetadata` declareren in plaats van core runtime te laten importeren om dit op te vragen.

## Naslaginformatie voor providerAuthChoices

Elke `providerAuthChoices`-vermelding beschrijft één keuze voor onboarding of authenticatie. OpenClaw leest deze voordat de provider-runtime wordt geladen. Lijsten voor providerconfiguratie gebruiken deze manifestkeuzes, uit descriptors afgeleide configuratiekeuzes en metadata uit de installatiecatalogus zonder de provider-runtime te laden.

| Veld                  | Vereist  | Type                                                                  | Betekenis                                                                                                      |
| --------------------- | -------- | --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `provider`            | Ja       | `string`                                                              | Provider-id waartoe deze keuze behoort.                                                                        |
| `method`              | Ja       | `string`                                                              | Id van de authenticatiemethode waarnaar moet worden doorgestuurd.                                              |
| `choiceId`            | Ja       | `string`                                                              | Stabiele id voor de authenticatiekeuze die wordt gebruikt door onboarding- en CLI-flows.                       |
| `choiceLabel`         | Nee      | `string`                                                              | Voor gebruikers zichtbaar label. Indien weggelaten, valt OpenClaw terug op `choiceId`.                 |
| `choiceHint`          | Nee      | `string`                                                              | Korte hulptekst voor de keuzelijst.                                                                            |
| `icon`                | Nee      | HTTPS-URL                                                             | Afbeelding die naast deze keuze wordt weergegeven in ondersteunde onboardingclients.                           |
| `website`             | Nee      | HTTPS-URL                                                             | Product-, aanmeld- of installatiepagina die door ondersteunde onboardingclients wordt weergegeven.            |
| `assistantPriority`   | Nee      | `number`                                                              | Lagere waarden worden eerder gesorteerd in interactieve, door de assistent aangestuurde keuzelijsten.          |
| `assistantVisibility` | Nee      | `"visible"` \| `"manual-only"`                                        | Verbergt de keuze in assistentkeuzelijsten, maar staat handmatige selectie via de CLI nog steeds toe.          |
| `deprecatedChoiceIds` | Nee      | `string[]`                                                            | Verouderde keuze-id's die gebruikers naar deze vervangende keuze moeten doorsturen.                            |
| `groupId`             | Nee      | `string`                                                              | Optionele groeps-id om gerelateerde keuzes te groeperen.                                                       |
| `groupLabel`          | Nee      | `string`                                                              | Voor gebruikers zichtbaar label voor die groep.                                                                |
| `groupHint`           | Nee      | `string`                                                              | Korte hulptekst voor de groep.                                                                                 |
| `onboardingFeatured`  | Nee      | `boolean`                                                             | Geeft deze groep weer in de uitgelichte categorie van de interactieve onboardingkeuzelijst, vóór de vermelding "Meer...". |
| `optionKey`           | Nee      | `string`                                                              | Interne optiesleutel voor eenvoudige authenticatieflows met één vlag.                                         |
| `cliFlag`             | Nee      | `string`                                                              | Naam van de CLI-vlag, zoals `--openrouter-api-key`.                                                                |
| `cliOption`           | Nee      | `string`                                                              | Volledige vorm van de CLI-optie, zoals `--openrouter-api-key <key>`.                                                     |
| `cliDescription`      | Nee      | `string`                                                              | Beschrijving die wordt gebruikt in de CLI-help.                                                                |
| `appGuidedSecret`     | Nee      | `boolean`                                                             | Eén geplakt geheim plus de standaardinstellingen van de provider volstaat voor app-begeleide configuratie.    |
| `appGuidedDiscovery`  | Nee      | `boolean`                                                             | De overeenkomende runtime-authenticatiemethode beheert alleen-lezen lokale detectie via `appGuidedSetup`.    |
| `appGuidedAuth`       | Nee      | `"oauth"` \| `"device-code"`                                          | Interactieve, door de provider beheerde aanmelding die native configuratieclients generiek kunnen weergeven.  |
| `onboardingScopes`    | Nee      | `Array<"text-inference" \| "image-generation" \| "music-generation">` | Op welke onboardingsurfaces deze keuze moet verschijnen. Indien weggelaten, is de standaardwaarde `["text-inference"]`. |

Wanneer `appGuidedDiscovery` waar is, moet de overeenkomende authenticatiemethode van de provider
`appGuidedSetup.detect` en `appGuidedSetup.prepare` beschikbaar stellen. Detectie moet
alleen-lezen zijn: geen aanmelding, ophalen van modellen, download of schrijven van configuratie. De voorbereiding controleert
het exact geselecteerde model opnieuw en retourneert een configuratievoorstel; OpenClaw test dat
voorstel afzonderlijk live en legt het pas na succes vast.

## Naslaginformatie voor commandAliases

Gebruik `commandAliases` wanneer een plugin eigenaar is van een runtime-opdrachtnaam die gebruikers per ongeluk in `plugins.allow` kunnen opnemen of als root-CLI-opdracht kunnen proberen uit te voeren. OpenClaw gebruikt deze metadata voor diagnostiek zonder de runtimecode van de plugin te importeren.

```json
{
  "commandAliases": [
    {
      "name": "dreaming",
      "kind": "runtime-slash",
      "cliCommand": "memory"
    }
  ]
}
```

| Veld         | Vereist  | Type              | Betekenis                                                                            |
| ------------ | -------- | ----------------- | ------------------------------------------------------------------------------------ |
| `name`       | Ja       | `string`          | Opdrachtnaam die bij deze plugin hoort.                                               |
| `kind`       | Nee      | `"runtime-slash"` | Markeert de alias als een slashopdracht in een chat in plaats van een root-CLI-opdracht. |
| `cliCommand` | Nee      | `string`          | Gerelateerde root-CLI-opdracht om voor CLI-bewerkingen voor te stellen, indien aanwezig. |

## Naslaginformatie voor activation

Gebruik `activation` wanneer de plugin op efficiënte wijze kan declareren bij welke gebeurtenissen in het besturingsvlak deze in een activerings-/laadplan moet worden opgenomen.

Dit blok bevat metadata voor de planner en is geen levenscyclus-API. Het registreert geen runtimegedrag, vervangt `register(...)` niet en garandeert niet dat plugincode al is uitgevoerd. De activeringsplanner gebruikt deze velden om kandidaat-plugins te beperken voordat wordt teruggevallen op bestaande metadata over eigenaarschap in het manifest, zoals `providers`, `channels`, `commandAliases`, `setup.providers`, `contracts.tools` en hooks.

Geef de voorkeur aan de meest specifieke metadata die het eigenaarschap al beschrijft. Gebruik `providers`, `channels`, `commandAliases`, configuratiedescriptors of `contracts` wanneer deze velden de relatie uitdrukken. Gebruik `activation` voor aanvullende plannerhints die niet door die eigenaarschapsvelden kunnen worden weergegeven. Gebruik `cliBackends` op het hoogste niveau voor CLI-runtime-aliassen zoals `claude-cli`, `my-cli` of `google-gemini-cli`; `activation.onAgentHarnesses` is alleen bestemd voor ids van ingebedde agentharnassen die nog geen eigenaarschapsveld hebben.

Elke plugin moet `activation.onStartup` bewust instellen. Stel dit alleen in op `true` wanneer de plugin tijdens het opstarten van de Gateway moet worden uitgevoerd. Stel het in op `false` wanneer de plugin bij het opstarten inactief is en alleen door specifiekere triggers moet worden geladen. Het weglaten van `onStartup` zorgt er niet langer impliciet voor dat de plugin bij het opstarten wordt geladen; gebruik expliciete activeringsmetadata voor activeringstriggers bij het opstarten, voor kanalen, configuratie, agentharnassen, geheugen of andere specifiekere triggers.

```json
{
  "activation": {
    "onStartup": false,
    "onProviders": ["openai"],
    "onCommands": ["models"],
    "onChannels": ["web"],
    "onRoutes": ["gateway-webhook"],
    "onConfigPaths": ["browser"],
    "onCapabilities": ["provider", "tool"]
  }
}
```

| Veld               | Vereist | Type                                                 | Betekenis                                                                                                                                                                                    |
| ------------------ | -------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `onStartup`        | Nee      | `boolean`                                            | Expliciete activering bij het starten van de Gateway. Elke plugin moet dit instellen. `true` importeert de plugin tijdens het starten; `false` laadt deze bij het starten pas wanneer een andere overeenkomende trigger dit vereist. |
| `onProviders`      | Nee      | `string[]`                                           | Provider-id's waarvoor deze plugin in activerings-/laadplannen moet worden opgenomen.                                                                                                       |
| `onAgentHarnesses` | Nee      | `string[]`                                           | Runtime-id's van ingebedde agentharnassen waarvoor deze plugin in activerings-/laadplannen moet worden opgenomen. Gebruik `cliBackends` op het hoogste niveau voor aliassen van CLI-backends. |
| `onCommands`       | Nee      | `string[]`                                           | Opdracht-id's waarvoor deze plugin in activerings-/laadplannen moet worden opgenomen.                                                                                                       |
| `onChannels`       | Nee      | `string[]`                                           | Kanaal-id's waarvoor deze plugin in activerings-/laadplannen moet worden opgenomen.                                                                                                         |
| `onRoutes`         | Nee      | `string[]`                                           | Routetypen waarvoor deze plugin in activerings-/laadplannen moet worden opgenomen.                                                                                                          |
| `onConfigPaths`    | Nee      | `string[]`                                           | Configuratiepaden ten opzichte van de hoofdmap waarvoor deze plugin in opstart-/laadplannen moet worden opgenomen wanneer het pad aanwezig en niet expliciet uitgeschakeld is.                |
| `onCapabilities`   | Nee      | `Array<"provider" \| "channel" \| "tool" \| "hook">` | Brede capaciteitshints die worden gebruikt voor activeringsplanning door het besturingsvlak. Geef waar mogelijk de voorkeur aan specifiekere velden.                                        |

Huidige actieve gebruikers:

- De opstartplanning van de Gateway gebruikt `activation.onStartup` voor expliciete import bij het starten.
- Door opdrachten geactiveerde CLI-planning valt terug op de verouderde `commandAliases[].cliCommand` of `commandAliases[].name`.
- De opstartplanning van de agentruntime gebruikt `activation.onAgentHarnesses` voor ingebedde harnassen en `cliBackends[]` op het hoogste niveau voor aliassen van CLI-runtimes.
- Door kanalen geactiveerde installatie-/kanaalplanning valt terug op het verouderde eigenaarschap van `channels[]` wanneer expliciete activeringsmetadata voor kanalen ontbreken.
- De planning van plugins bij het starten gebruikt `activation.onConfigPaths` voor hoofdconfiguratieoppervlakken die niet kanaalspecifiek zijn, zoals het `browser`-blok van de meegeleverde browserplugin.
- Door providers geactiveerde installatie-/runtimeplanning valt terug op het verouderde eigenaarschap van `providers[]` en `cliBackends[]` op het hoogste niveau wanneer expliciete activeringsmetadata voor providers ontbreken.

Plannerdiagnostiek kan expliciete activeringshints onderscheiden van terugval op manifesteigenaarschap. `activation-command-hint` betekent bijvoorbeeld dat `activation.onCommands` overeenkwam, terwijl `manifest-command-alias` betekent dat de planner in plaats daarvan het eigenaarschap van `commandAliases` gebruikte. Deze redenlabels zijn bedoeld voor hostdiagnostiek en tests; pluginauteurs moeten de metadata blijven declareren die het eigenaarschap het beste beschrijven.

## Naslag voor qaRunners

Gebruik `qaRunners` wanneer een plugin een of meer transportrunners toevoegt onder
de gedeelde hoofdmap `openclaw qa`. Houd deze metadata goedkoop en statisch; de
pluginruntime blijft verantwoordelijk voor de daadwerkelijke CLI-registratie via een lichtgewicht
`runtime-api.ts`-oppervlak dat overeenkomende `qaRunnerCliRegistrations` exporteert. Een
optionele `adapterFactory` stelt het transport beschikbaar aan gedeelde QA-scenario's zonder
de runner van de geregistreerde opdracht te wijzigen.

```json
{
  "qaRunners": [
    {
      "commandName": "matrix",
      "description": "Voer de door Docker ondersteunde actieve QA-lane voor Matrix uit tegen een tijdelijke homeserver"
    }
  ]
}
```

| Veld          | Vereist | Type     | Betekenis                                                           |
| ------------- | -------- | -------- | ------------------------------------------------------------------- |
| `commandName` | Ja       | `string` | Subopdracht die onder `openclaw qa` is gekoppeld, bijvoorbeeld `matrix`. |
| `description` | Nee      | `string` | Terugvalhelptekst die wordt gebruikt wanneer de gedeelde host een tijdelijke opdracht nodig heeft. |

De `adapterFactory`-id moet overeenkomen met `commandName`. Exporteer geen registraties
voor opdrachten die niet in het manifest staan.

## Naslag voor setup

Gebruik `setup` wanneer installatie- en onboardingoppervlakken goedkope, door de plugin beheerde metadata nodig hebben voordat de runtime wordt geladen.

```json
{
  "setup": {
    "providers": [
      {
        "id": "openai",
        "authMethods": ["api-key"],
        "envVars": ["OPENAI_API_KEY"],
        "authEvidence": [
          {
            "type": "local-file-with-env",
            "fileEnvVar": "OPENAI_CREDENTIALS_FILE",
            "requiresAllEnv": ["OPENAI_PROJECT"],
            "credentialMarker": "openai-local-credentials",
            "source": "lokale OpenAI-referenties"
          }
        ]
      }
    ],
    "cliBackends": ["openai-cli"],
    "configMigrations": ["legacy-openai-auth"],
    "requiresRuntime": false
  }
}
```

`cliBackends` op het hoogste niveau blijft geldig en blijft CLI-inferentiebackends beschrijven. `setup.cliBackends` is het installatiespecifieke descriptoroppervlak voor besturingsvlak-/installatiestromen die uitsluitend uit metadata moeten blijven bestaan.

Wanneer aanwezig, vormen `setup.providers` en `setup.cliBackends` het voorkeursoppervlak voor descriptorgerichte opzoekacties tijdens installatiedetectie. Als de descriptor alleen de kandidaat-plugin beperkt en de installatie nog uitgebreidere runtimehooks voor de installatiefase nodig heeft, stel je `requiresRuntime: true` in en behoud je `setup-api` als terugvalpad voor uitvoering.

OpenClaw neemt `setup.providers[].envVars` op in generieke opzoekacties voor providerauthenticatie en omgevingsvariabelen. Plaats daar omgevingsmetadata voor installatie en status.

Gebruik `providerUsageAuthEnvVars` wanneer referenties op facturerings- of organisatieniveau `resolveUsageAuth` moeten activeren zonder inferentiereferenties te worden. Deze namen worden opgenomen in het blokkeren van dotenv-waarden voor werkruimten, het verwijderen van waarden uit ACP-subprocessen, het filteren van geheimen in de sandbox en het breed opschonen van geheimen. De providerruntime leest en classificeert de waarde nog steeds binnen `resolveUsageAuth`.

OpenClaw kan ook eenvoudige installatiekeuzes afleiden uit `setup.providers[].authMethods` wanneer geen installatie-item beschikbaar is of wanneer `setup.requiresRuntime: false` aangeeft dat een installatieruntime niet nodig is. Expliciete `providerAuthChoices`-items blijven de voorkeur genieten voor aangepaste labels, CLI-vlaggen, onboardingsbereik en assistentmetadata.

Stel `requiresRuntime: false` alleen in wanneer deze descriptors voldoende zijn voor het installatieoppervlak. OpenClaw behandelt een expliciete `false` als een contract dat uitsluitend uit descriptors bestaat en voert `setup-api` of `openclaw.setupEntry` niet uit voor opzoekacties tijdens de installatie. Als een plugin die uitsluitend descriptors gebruikt toch een van deze runtime-items voor installatie levert, rapporteert OpenClaw aanvullende diagnostiek en blijft het item negeren. Wanneer `requiresRuntime` is weggelaten, blijft het verouderde terugvalgedrag behouden, zodat bestaande plugins die descriptors zonder de vlag hebben toegevoegd niet defect raken.

Omdat de installatieopzoekactie door de plugin beheerde `setup-api`-code kan uitvoeren, moeten genormaliseerde waarden voor `setup.providers[].id` en `setup.cliBackends[]` uniek blijven voor alle gedetecteerde plugins. Bij dubbelzinnig eigenaarschap wordt de bewerking veilig geweigerd in plaats van een winnaar te kiezen op basis van de detectievolgorde.

Wanneer de installatieruntime wel wordt uitgevoerd, rapporteert de diagnostiek van het installatieregister descriptorafwijkingen als `setup-api` een provider of CLI-backend registreert die niet door de manifestdescriptors wordt gedeclareerd, of als een descriptor geen overeenkomende runtimeregistratie heeft. Deze diagnostiek is aanvullend en wijst verouderde plugins niet af.

### Naslag voor setup.providers

| Veld           | Vereist | Type       | Betekenis                                                                                         |
| -------------- | -------- | ---------- | ------------------------------------------------------------------------------------------------- |
| `id`           | Ja       | `string`   | Provider-id dat tijdens installatie of onboarding beschikbaar wordt gesteld. Houd genormaliseerde id's wereldwijd uniek. |
| `authMethods`  | Nee      | `string[]` | Id's van installatie-/authenticatiemethoden die deze provider ondersteunt zonder de volledige runtime te laden.          |
| `envVars`      | Nee      | `string[]` | Omgevingsvariabelen die generieke installatie-/statusoppervlakken kunnen controleren voordat de pluginruntime wordt geladen. |
| `authEvidence` | Nee      | `object[]` | Goedkope lokale controles op authenticatiebewijs voor providers die zich via niet-geheime markeringen kunnen authenticeren. |

`authEvidence` is bedoeld voor lokale, door providers beheerde referentiemarkeringen die kunnen worden geverifieerd zonder runtimecode te laden. Deze controles moeten goedkoop en lokaal blijven: geen netwerkoproepen, geen leesacties uit een sleutelhanger of geheimenbeheerder, geen shellopdrachten en geen controles via de provider-API.

Ondersteunde bewijsitems:

| Veld               | Vereist | Type       | Betekenis                                                                                                     |
| ------------------ | -------- | ---------- | ------------------------------------------------------------------------------------------------------------- |
| `type`             | Ja       | `string`   | Momenteel `local-file-with-env`.                                                                              |
| `fileEnvVar`       | Nee      | `string`   | Omgevingsvariabele met een expliciet bestandspad voor referenties.                                             |
| `fallbackPaths`    | Nee      | `string[]` | Lokale bestandspaden voor referenties die worden gecontroleerd wanneer `fileEnvVar` ontbreekt of leeg is. Ondersteunt `${HOME}` en `${APPDATA}`. |
| `requiresAnyEnv`   | Nee      | `string[]` | Ten minste één vermelde omgevingsvariabele moet niet leeg zijn voordat het bewijs geldig is.                   |
| `requiresAllEnv`   | Nee      | `string[]` | Elke vermelde omgevingsvariabele moet niet leeg zijn voordat het bewijs geldig is.                             |
| `credentialMarker` | Ja       | `string`   | Niet-geheime markering die wordt geretourneerd wanneer het bewijs aanwezig is.                                 |
| `source`           | Nee      | `string`   | Voor de gebruiker zichtbaar bronlabel voor authenticatie-/statusuitvoer.                                      |

### Installatievelden

| Veld               | Vereist | Type       | Betekenis                                                                                            |
| ------------------ | -------- | ---------- | ---------------------------------------------------------------------------------------------------- |
| `providers`        | Nee      | `object[]` | Tijdens configuratie en onboarding beschikbare descriptors voor providerconfiguratie.               |
| `cliBackends`      | Nee      | `string[]` | Backend-id's die tijdens de configuratie worden gebruikt voor descriptor-first-opzoeking. Houd genormaliseerde id's wereldwijd uniek. |
| `configMigrations` | Nee      | `string[]` | Configuratiemigratie-id's die eigendom zijn van het configuratieoppervlak van deze plugin.           |
| `requiresRuntime`  | Nee      | `boolean`  | Of voor de configuratie na het opzoeken van de descriptor nog uitvoering van `setup-api` nodig is. |

## Referentie voor uiHints

`uiHints` is een toewijzing van namen van configuratievelden naar kleine renderhints. Sleutels kunnen punten gebruiken voor geneste configuratievelden, maar geen enkel padsegment mag `__proto__`, `constructor` of `prototype` zijn; de configuratie weigert deze namen.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "API-sleutel",
      "help": "Gebruikt voor OpenRouter-verzoeken",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

Elke veldhint kan het volgende bevatten:

| Veld           | Type             | Betekenis                                                                                                         |
| -------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------- |
| `label`        | `string`         | Voor de gebruiker zichtbaar veldlabel.                                                                           |
| `help`         | `string`         | Korte helptekst.                                                                                                  |
| `tags`         | `string[]`       | Optionele UI-tags.                                                                                                |
| `advanced`     | `boolean`        | Markeert het veld als geavanceerd.                                                                                |
| `sensitive`    | `boolean`        | Markeert het veld als geheim of gevoelig.                                                                         |
| `placeholder`  | `string`         | Tijdelijke aanduiding voor formulierinvoer.                                                                       |
| `presentation` | `"phone-number"` | Alleen voor weergave bedoelde, gelokaliseerde telefoonnummeropmaak voor parseerbare internationale (`+...`)-waarden; onbewerkte waarden blijven ongewijzigd. |

## Referentie voor contracts

Gebruik `contracts` alleen voor statische metadata over eigendom van mogelijkheden die OpenClaw kan lezen zonder de runtime van de plugin te importeren.

```json
{
  "contracts": {
    "agentToolResultMiddleware": ["openclaw", "codex"],
    "trustedToolPolicies": ["workflow-budget"],
    "externalAuthProviders": ["acme-ai"],
    "embeddingProviders": ["openai-compatible"],
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "memoryEmbeddingProviders": ["local"],
    "mediaUnderstandingProviders": ["openai"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "musicGenerationProviders": ["stability-audio"],
    "documentExtractors": ["example-docs"],
    "webContentExtractors": ["firecrawl"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "workerProviders": ["example-worker"],
    "usageProviders": ["acme-ai"],
    "migrationProviders": ["hermes"],
    "gatewayMethodDispatch": ["authenticated-request"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

Elke lijst is optioneel:

| Veld                             | Type       | Betekenis                                                                                                                            |
| -------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `embeddedExtensionFactories`     | `string[]` | Fabrieks-id's voor Codex-app-serverextensies, momenteel `codex-app-server`.                                                          |
| `agentToolResultMiddleware`      | `string[]` | Runtime-id's waarvoor deze plugin middleware voor toolresultaten mag registreren.                                                    |
| `trustedToolPolicies`            | `string[]` | Lokale id's van vertrouwd beleid vóór tooluitvoering die een geïnstalleerde plugin mag registreren. Gebundelde plugins mogen beleid registreren zonder dit veld. |
| `externalAuthProviders`          | `string[]` | Provider-id's waarvan deze plugin de hook voor externe authenticatieprofielen bezit.                                                 |
| `embeddingProviders`             | `string[]` | Algemene id's van embeddingproviders die deze plugin bezit voor herbruikbaar gebruik van vectorembeddings, inclusief geheugen.       |
| `speechProviders`                | `string[]` | Id's van spraakproviders die deze plugin bezit.                                                                                      |
| `realtimeTranscriptionProviders` | `string[]` | Id's van providers voor realtime transcriptie die deze plugin bezit.                                                                |
| `realtimeVoiceProviders`         | `string[]` | Id's van providers voor realtime spraak die deze plugin bezit.                                                                      |
| `memoryEmbeddingProviders`       | `string[]` | Verouderde id's van geheugenspecifieke embeddingproviders die deze plugin bezit.                                                     |
| `mediaUnderstandingProviders`    | `string[]` | Id's van providers voor mediabegrip die deze plugin bezit.                                                                          |
| `transcriptSourceProviders`      | `string[]` | Id's van providers van transcriptbronnen die deze plugin bezit.                                                                     |
| `documentExtractors`             | `string[]` | Id's van providers voor documentextractie (bijvoorbeeld PDF) die deze plugin bezit.                                                  |
| `imageGenerationProviders`       | `string[]` | Id's van providers voor het genereren van afbeeldingen die deze plugin bezit.                                                       |
| `videoGenerationProviders`       | `string[]` | Id's van providers voor het genereren van video die deze plugin bezit.                                                              |
| `musicGenerationProviders`       | `string[]` | Id's van providers voor het genereren van muziek die deze plugin bezit.                                                             |
| `webContentExtractors`           | `string[]` | Id's van providers voor inhoudsextractie uit webpagina's die deze plugin bezit.                                                      |
| `webFetchProviders`              | `string[]` | Id's van webophaalproviders die deze plugin bezit.                                                                                   |
| `webSearchProviders`             | `string[]` | Id's van webzoekproviders die deze plugin bezit.                                                                                     |
| `workerProviders`                | `string[]` | Id's van cloudworkerproviders die deze plugin bezit voor inrichting en de door profielen ondersteunde leaselevenscyclus.             |
| `usageProviders`                 | `string[]` | Provider-id's waarvan deze plugin de hooks voor gebruiksauthenticatie en gebruikssnapshots bezit.                                    |
| `migrationProviders`             | `string[]` | Id's van importproviders die deze plugin bezit voor `openclaw migrate`.                                                              |
| `gatewayMethodDispatch`          | `string[]` | Gereserveerd recht voor geauthenticeerde HTTP-routes van plugins die Gateway-methoden binnen het proces dispatchen.                  |
| `tools`                          | `string[]` | Namen van agenttools die deze plugin bezit.                                                                                          |

`contracts.embeddedExtensionFactories` blijft behouden voor gebundelde extensiefabrieken die uitsluitend voor de Codex-app-server zijn bedoeld. Gebundelde transformaties van toolresultaten moeten in plaats daarvan `contracts.agentToolResultMiddleware` declareren en zich bij `api.registerAgentToolResultMiddleware(...)` registreren. Geïnstalleerde plugins mogen dezelfde middlewarekoppeling alleen gebruiken wanneer deze expliciet is ingeschakeld en uitsluitend voor runtimes die zij in `contracts.agentToolResultMiddleware` declareren.

Geïnstalleerde plugins die het door de host vertrouwde beleidsniveau vóór tooluitvoering nodig hebben, moeten elke geregistreerde lokale id in `contracts.trustedToolPolicies` declareren en expliciet zijn ingeschakeld. Gebundelde plugins behouden het bestaande pad voor vertrouwd beleid, maar geïnstalleerde plugins met niet-gedeclareerde beleids-id's worden vóór registratie geweigerd. Beleids-id's vallen binnen het bereik van de registrerende plugin, zodat twee plugins beide `workflow-budget` mogen declareren en registreren; één plugin mag dezelfde lokale id niet twee keer registreren.

Runtime-registraties van `api.registerTool(...)` moeten overeenkomen met `contracts.tools`. Tooldetectie gebruikt deze lijst om uitsluitend de pluginruntimes te laden die eigenaar kunnen zijn van de aangevraagde tools.

Providerplugins die `resolveExternalAuthProfiles` implementeren, moeten `contracts.externalAuthProviders` declareren; niet-gedeclareerde hooks voor externe authenticatie worden genegeerd.

Providerplugins die zowel `resolveUsageAuth` als `fetchUsageSnapshot` implementeren, moeten elke automatisch gedetecteerde provider-id in `contracts.usageProviders` declareren. Gebruiksdetectie leest dit contract voordat runtimecode wordt geladen en verifieert vervolgens beide hooks nadat alleen de gedeclareerde eigenaren zijn geladen.

Algemene embeddingproviders moeten `contracts.embeddingProviders` declareren voor elke adapter die bij `api.registerEmbeddingProvider(...)` is geregistreerd. Gebruik het algemene contract voor herbruikbare vectorgeneratie, inclusief providers die door geheugenzoekopdrachten worden gebruikt. `contracts.memoryEmbeddingProviders` is verouderde, geheugenspecifieke compatibiliteit en blijft alleen bestaan terwijl bestaande providers naar de generieke koppeling voor embeddingproviders migreren.

Workerproviders moeten elke `api.registerWorkerProvider(...)`-id in `contracts.workerProviders` declareren. Core slaat duurzame intentie op voordat `provision` wordt aangeroepen; providers valideren hun instellingen vóór externe toewijzing en herhaalde aanroepen met dezelfde bewerkings-id moeten dezelfde lease overnemen. Core slaat ook die gevalideerde momentopname van instellingen op en geeft deze met `leaseId` door aan `inspect({ leaseId, profile })` en `destroy({ leaseId, profile })`, ook nadat het benoemde profiel is gewijzigd of verwijderd. Vernietiging is idempotent, inspectie retourneert de gesloten status-unie `active` / `destroyed` / `unknown`, en materiaal van privésleutels voor SSH wordt uitsluitend via `SecretRef` verwezen. Ingerichte SSH-eindpunten moeten ook een openbare `hostKey` uit vertrouwde inrichtingsuitvoer bevatten als exact `algorithm base64`, zonder hostnaam of opmerking, zodat Core de host vóór het verbinden kan vastzetten. Providers die dynamische identiteitsreferenties aanmaken, mogen een gezaghebbende `resolveSshIdentity({ leaseId, profile, keyRef })` implementeren; providers zonder deze implementatie gebruiken de generieke geheimresolver van Core. Een gezaghebbende `unknown` maakt een actieve lokale record verweesd; na een opgeslagen vernietigingsaanvraag bevestigt deze de ontmanteling.

`contracts.gatewayMethodDispatch` accepteert momenteel `"authenticated-request"`. Het is een API-hygiënepoort voor native HTTP-routes van plugins die opzettelijk Gateway-besturingsvlakmethoden in-process aanroepen, geen sandbox tegen kwaadaardige native plugins. Gebruik dit alleen voor grondig beoordeelde gebundelde/operatoroppervlakken die al HTTP-authenticatie van de Gateway vereisen. Een geautoriseerde route blijft bereikbaar terwijl de toelating van rootwerk door de Gateway is gesloten, maar alleen wanneer deze ook `auth: "gateway"` en de routespecifieke `gatewayRuntimeScopeSurface: "trusted-operator"` declareert; gewone zusterroutes van dezelfde plugin blijven achter de toelatingsgrens. Hierdoor blijven de opschortingsstatus en hervatting bereikbaar zonder de hele plugin een omzeiling van de toelating te verlenen. Houd parsering en vormgeving van reacties begrensd buiten de aanroep; inhoudelijk of muterend werk moet via de methodedispatch van de Gateway verlopen, die de toelating en scopehandhaving beheert.

## Naslaginformatie voor configContracts

Gebruik `configContracts` voor configuratiegedrag dat eigendom is van het manifest en dat generieke kernhelpers nodig hebben zonder de pluginruntime te importeren: detectie van gevaarlijke vlaggen, migratiedoelen voor SecretRef en beperking van verouderde configuratiepaden.

```json
{
  "configContracts": {
    "compatibilityMigrationPaths": ["legacyProvider"],
    "compatibilityRuntimePaths": ["legacyProvider.webhook"],
    "dangerousFlags": [
      {
        "path": "accounts.*.allowUnverifiedSenders",
        "equals": true
      }
    ],
    "secretInputs": {
      "bundledDefaultEnabled": false,
      "paths": [
        {
          "path": "routes.*.secret",
          "expected": "string",
          "ownerKind": "route"
        }
      ]
    }
  }
}
```

| Veld                          | Vereist | Type       | Betekenis                                                                                                                                                                                                                              |
| ----------------------------- | -------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `compatibilityMigrationPaths` | Nee      | `string[]` | Configuratiepaden relatief aan de root die aangeven dat de compatibiliteitsmigraties tijdens de installatie van deze plugin mogelijk van toepassing zijn. Hiermee kunnen generieke runtimelezingen van configuratie elk installatieoppervlak van plugins overslaan wanneer de configuratie nergens naar de plugin verwijst. |
| `compatibilityRuntimePaths`   | Nee      | `string[]` | Compatibiliteitspaden relatief aan de root die deze plugin tijdens runtime kan afhandelen voordat de plugincode volledig wordt geactiveerd. Gebruik dit voor verouderde oppervlakken die verzamelingen gebundelde kandidaten moeten beperken zonder de runtime van elke compatibele plugin te importeren. |
| `dangerousFlags`              | Nee      | `object[]` | Configuratieliteralen die `openclaw doctor` als onveilig of gevaarlijk moet markeren wanneer ze zijn ingeschakeld. Zie hieronder.                                                                                                       |
| `secretInputs`                | Nee      | `object`   | Configuratiepaden onder `plugins.entries.<id>.config` voor SecretRef-migratie, audits, materialisatie bij het opstarten en optionele isolatie van runtime-eigenaren. Zie hieronder.                                                                  |

Elke `dangerousFlags`-vermelding ondersteunt:

| Veld     | Vereist | Type                                  | Betekenis                                                                                                           |
| -------- | -------- | ------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `path`   | Ja       | `string`                              | Door punten gescheiden configuratiepad relatief aan `plugins.entries.<id>.config`. Ondersteunt `*`-jokertekens voor kaart-/arraysegmenten. |
| `equals` | Ja       | `string \| number \| boolean \| null` | Exacte letterlijke waarde die deze configuratiewaarde als gevaarlijk markeert.                                      |

`secretInputs` ondersteunt:

| Veld                    | Vereist | Type       | Betekenis                                                                                                                                                                                                                                                                                                                                                  |
| ----------------------- | -------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bundledDefaultEnabled` | Nee      | `boolean`  | Overschrijf de standaardinschakeling van gebundelde plugins wanneer wordt bepaald of dit SecretRef-oppervlak actief is. Gebruik dit wanneer de plugin is gebundeld, maar het oppervlak inactief moet blijven totdat het expliciet in de configuratie wordt ingeschakeld.                                                                                       |
| `paths`                 | Ja       | `object[]` | Geheimvormige configuratiepaden, elk met `path` (door punten gescheiden, relatief aan `plugins.entries.<id>.config`, ondersteunt `*`-jokertekens), optioneel `expected` (momenteel alleen `"string"`) en optioneel `ownerKind` (momenteel alleen `"route"`). Een gedeclareerde eigenaar isoleert alleen dat exact overeenkomende pad wanneer de oplossing mislukt; de eigenaar-id is het volledige configuratiepad. |

## Naslaginformatie voor mediaUnderstandingProviderMetadata

Gebruik `mediaUnderstandingProviderMetadata` wanneer een provider voor mediabegrip standaardmodellen, prioriteit voor automatische authenticatieterugval of native documentondersteuning heeft die generieke kernhelpers nodig hebben voordat de runtime wordt geladen. Sleutels moeten ook in `contracts.mediaUnderstandingProviders` worden gedeclareerd.

```json
{
  "contracts": {
    "mediaUnderstandingProviders": ["example"]
  },
  "mediaUnderstandingProviderMetadata": {
    "example": {
      "capabilities": ["image", "audio"],
      "defaultModels": {
        "image": "example-vision-latest",
        "audio": "example-transcribe-latest"
      },
      "autoPriority": {
        "image": 40
      },
      "nativeDocumentInputs": ["pdf"],
      "documentModels": {
        "pdf": {
          "textExtraction": "example-doc-text-latest",
          "image": "example-doc-vision-latest"
        }
      }
    }
  }
}
```

Elke providervermelding kan het volgende bevatten:

| Veld                   | Type                                                             | Betekenis                                                                                                       |
| ---------------------- | ---------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `capabilities`         | `("image" \| "audio" \| "video")[]`                              | Mediamogelijkheden die door deze provider beschikbaar worden gesteld.                                          |
| `defaultModels`        | `Record<string, string>`                                         | Standaardwaarden voor de koppeling van mogelijkheden aan modellen die worden gebruikt wanneer de configuratie geen model opgeeft. |
| `autoPriority`         | `Record<string, number>`                                         | Lagere getallen worden eerder gesorteerd voor automatische, op referenties gebaseerde providerterugval.         |
| `nativeDocumentInputs` | `"pdf"[]`                                                        | Native documentinvoer die door de provider wordt ondersteund.                                                   |
| `documentModels`       | `{ pdf?: { textExtraction?: string; image?: string \| false } }` | Modeloverschrijvingen per documenttype. Stel `image: false` in om beeldgebaseerde extractie voor dat documenttype uit te schakelen. |

## Naslaginformatie voor channelConfigs

Gebruik `channelConfigs` wanneer een kanaalplugin goedkope configuratiemetadata nodig heeft voordat de runtime wordt geladen. Alleen-lezen ontdekking van kanaalinstallatie/-status kan deze metadata rechtstreeks gebruiken voor geconfigureerde externe kanalen wanneer geen installatievermelding beschikbaar is, of wanneer `setup.requiresRuntime: false` verklaart dat een installatieruntime niet nodig is.

`channelConfigs` is metadata van het pluginmanifest, geen nieuwe configuratiesectie op het hoogste niveau voor gebruikers. Gebruikers configureren kanaalinstanties nog steeds onder `channels.<channel-id>`. OpenClaw leest manifestmetadata om te bepalen welke plugin eigenaar is van dat geconfigureerde kanaal voordat de pluginruntimecode wordt uitgevoerd.

Voor een kanaalplugin beschrijven `configSchema` en `channelConfigs` verschillende paden:

- `configSchema` valideert `plugins.entries.<plugin-id>.config`
- `channelConfigs.<channel-id>.schema` valideert `channels.<channel-id>`

Niet-gebundelde plugins die `channels[]` declareren, moeten ook overeenkomende `channelConfigs`-vermeldingen declareren. Zonder deze vermeldingen kan OpenClaw de plugin nog steeds laden, maar kunnen configuratieschema's voor koude paden, installatie- en Control UI-oppervlakken de vorm van kanaaleigen opties of uitsluitend voor weergave bedoelde UI-hints niet kennen totdat de pluginruntime wordt uitgevoerd.

`channelConfigs.<channel-id>.commands.nativeCommandsAutoEnabled` en `nativeSkillsAutoEnabled` kunnen statische standaardwaarden voor `auto` declareren voor controles van opdrachtconfiguratie die worden uitgevoerd voordat de kanaalruntime wordt geladen. Gebundelde kanalen kunnen dezelfde standaardwaarden ook publiceren via `package.json#openclaw.channel.commands`, naast hun andere kanaalcatalogusmetadata waarvan het pakket eigenaar is.

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "Homeserver-URL",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Matrix-homeserververbinding",
      "commands": {
        "nativeCommandsAutoEnabled": true,
        "nativeSkillsAutoEnabled": true
      },
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

Elke kanaalvermelding kan het volgende bevatten:

| Veld          | Type                     | Betekenis                                                                                                        |
| ------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | JSON Schema voor `channels.<id>`. Vereist voor elke gedeclareerde configuratievermelding van een kanaal.        |
| `uiHints`     | `Record<string, object>` | Optionele labels, tijdelijke aanduidingen, gevoeligheid en uitsluitend voor weergave bedoelde presentatiehints voor die kanaalconfiguratiesectie. |
| `label`       | `string`                 | Kanaallabel dat wordt samengevoegd in keuze- en inspectieoppervlakken wanneer runtimemetadata nog niet gereed is. |
| `description` | `string`                 | Korte kanaalbeschrijving voor inspectie- en catalogusoppervlakken.                                               |
| `commands`    | `object`                 | Statische automatische standaardwaarden voor native opdrachten en native Skills voor configuratiecontroles vóór de runtime. |
| `preferOver`  | `string[]`               | Verouderde plugin-id's of plugin-id's met lagere prioriteit die dit kanaal in selectieoppervlakken moet overtreffen. |

### Een andere kanaalplugin vervangen

Gebruik `preferOver` wanneer jouw plugin de voorkeurseigenaar is voor een kanaal-id dat ook door een andere plugin kan worden geleverd. Veelvoorkomende gevallen zijn een hernoemde plugin-id, een zelfstandige plugin die een gebundelde plugin vervangt, of een onderhouden fork die dezelfde kanaal-id behoudt voor configuratiecompatibiliteit.

```json
{
  "id": "acme-chat",
  "channels": ["chat"],
  "channelConfigs": {
    "chat": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "webhookUrl": { "type": "string" }
        }
      },
      "preferOver": ["chat"]
    }
  }
}
```

Wanneer `channels.chat` is geconfigureerd, houdt OpenClaw rekening met zowel de kanaal-id als de voorkeurs-Plugin-id. Als de Plugin met lagere prioriteit alleen is geselecteerd omdat deze is meegeleverd of standaard is ingeschakeld, schakelt OpenClaw deze uit in de effectieve runtimeconfiguratie, zodat één Plugin eigenaar is van het kanaal en de bijbehorende tools. Expliciete selectie door de gebruiker heeft nog steeds voorrang: als de gebruiker beide Plugins expliciet inschakelt (via `plugins.allow` of een materiële `plugins.entries`-configuratie), behoudt OpenClaw die keuze en rapporteert het diagnostische gegevens over dubbele kanalen/tools in plaats van de aangevraagde Plugin-set stilzwijgend te wijzigen.

Beperk `preferOver` tot Plugin-id's die daadwerkelijk hetzelfde kanaal kunnen leveren. Het is geen algemeen prioriteitsveld en het hernoemt geen configuratiesleutels van gebruikers.

## Naslaginformatie voor modelSupport

Gebruik `modelSupport` wanneer OpenClaw je provider-Plugin moet afleiden uit verkorte model-id's zoals `gpt-5.6-sol` of `claude-sonnet-4.6` voordat de Plugin-runtime wordt geladen.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw past deze voorrangsvolgorde toe:

- expliciete `provider/model`-verwijzingen gebruiken de manifestmetagegevens van de bijbehorende `providers`
- `modelPatterns` hebben voorrang op `modelPrefixes`
- als één niet-meegeleverde Plugin en één meegeleverde Plugin beide overeenkomen, heeft de niet-meegeleverde Plugin voorrang
- resterende ambiguïteit wordt genegeerd totdat de gebruiker of configuratie een provider opgeeft

Velden:

| Veld           | Type       | Betekenis                                                                   |
| --------------- | ---------- | ------------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | Voorvoegsels die met `startsWith` worden vergeleken met verkorte model-id's.                 |
| `modelPatterns` | `string[]` | Regex-bronnen die na verwijdering van het profielsuffix worden vergeleken met verkorte model-id's. |

`modelPatterns`-vermeldingen worden gecompileerd via `compileSafeRegex`, dat patronen met geneste herhaling afwijst (bijvoorbeeld `(a+)+$`). Patronen die niet door de veiligheidscontrole komen, worden stilzwijgend overgeslagen, net als syntactisch ongeldige regex. Houd patronen eenvoudig en vermijd geneste kwantoren.

## Naslaginformatie voor modelCatalog

Gebruik `modelCatalog` wanneer OpenClaw vóór het laden van de Plugin-runtime metagegevens over providermodellen moet kennen. Dit is de bron in eigendom van het manifest voor vaste catalogusrijen, provideraliassen, onderdrukkingsregels en de detectiemodus. Runtimevernieuwing blijft de verantwoordelijkheid van de providerruntimecode, maar het manifest vertelt de core wanneer runtime vereist is.

```json
{
  "providers": ["openai"],
  "modelCatalog": {
    "providers": {
      "openai": {
        "baseUrl": "https://api.openai.com/v1",
        "api": "openai-responses",
        "models": [
          {
            "id": "gpt-5.4",
            "name": "GPT-5.4",
            "input": ["text", "image"],
            "reasoning": true,
            "contextWindow": 256000,
            "maxTokens": 128000,
            "cost": {
              "input": 1.25,
              "output": 10,
              "cacheRead": 0.125
            },
            "status": "available",
            "tags": ["default"]
          }
        ]
      }
    },
    "aliases": {
      "azure-openai-responses": {
        "provider": "openai",
        "api": "azure-openai-responses"
      }
    },
    "suppressions": [
      {
        "provider": "azure-openai-responses",
        "model": "gpt-5.3-codex-spark",
        "reason": "not available on Azure OpenAI Responses"
      }
    ],
    "discovery": {
      "openai": "static"
    }
  }
}
```

Velden op het hoogste niveau:

| Veld            | Type                                                     | Betekenis                                                                                               |
| ---------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `providers`      | `Record<string, object>`                                 | Catalogusrijen voor provider-id's die eigendom zijn van deze Plugin. Sleutels moeten ook voorkomen in `providers` op het hoogste niveau.       |
| `aliases`        | `Record<string, object>`                                 | Provideraliassen die voor catalogus- of onderdrukkingsplanning moeten verwijzen naar een provider in eigendom.              |
| `suppressions`   | `object[]`                                               | Modelrijen uit een andere bron die deze Plugin om een providerspecifieke reden onderdrukt.                  |
| `discovery`      | `Record<string, "static" \| "refreshable" \| "runtime">` | Of de providercatalogus uit manifestmetagegevens kan worden gelezen, in de cache kan worden vernieuwd of runtime vereist. |
| `runtimeAugment` | `boolean`                                                | Stel alleen in op `true` wanneer de providerruntime catalogusrijen moet toevoegen na de manifest-/configuratieplanning.       |

`aliases` neemt deel aan het opzoeken van providereigendom voor modelcatalogusplanning. Aliasdoelen moeten providers op het hoogste niveau zijn die eigendom zijn van dezelfde Plugin. Wanneer een op provider gefilterde lijst een alias gebruikt, kan OpenClaw het bijbehorende manifest lezen en API-/basis-URL-overschrijvingen van de alias toepassen zonder de providerruntime te laden. Aliassen breiden ongefilterde catalogusvermeldingen niet uit; brede lijsten geven alleen de canonieke provider-rijen van de eigenaar weer.

`suppressions` vervangt de oude providerruntime-hook `suppressBuiltInModel`. Onderdrukkingsvermeldingen worden alleen gehonoreerd wanneer de provider eigendom is van de Plugin of is gedeclareerd als een `modelCatalog.aliases`-sleutel die naar een provider in eigendom verwijst. Runtime-hooks voor onderdrukking worden niet langer aangeroepen tijdens modelresolutie.

Providervelden:

| Veld                 | Type                     | Betekenis                                                                                                                                                                                                     |
| --------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `baseUrl`             | `string`                 | Optionele standaardbasis-URL voor modellen in deze providercatalogus.                                                                                                                                                    |
| `api`                 | `ModelApi`               | Optionele standaard-API-adapter voor modellen in deze providercatalogus.                                                                                                                                                 |
| `headers`             | `Record<string, string>` | Optionele statische headers die op deze providercatalogus van toepassing zijn.                                                                                                                                                      |
| `defaultUtilityModel` | `string`                 | Optionele, door de provider aanbevolen kleine model-id voor korte interne hulptaken (titels, voortgangsbeschrijvingen). Wordt gebruikt wanneer `agents.defaults.utilityModel` niet is ingesteld en deze provider het primaire model van de agent levert. |
| `models`              | `object[]`               | Vereiste modelrijen. Rijen zonder een `id` worden genegeerd.                                                                                                                                                            |

Modelvelden:

| Veld              | Type                                                           | Betekenis                                                               |
| ------------------ | -------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `id`               | `string`                                                       | Providerlokale model-id, zonder het voorvoegsel `provider/`.                    |
| `name`             | `string`                                                       | Optionele weergavenaam.                                                      |
| `api`              | `ModelApi`                                                     | Optionele API-overschrijving per model.                                            |
| `baseUrl`          | `string`                                                       | Optionele overschrijving van de basis-URL per model.                                       |
| `headers`          | `Record<string, string>`                                       | Optionele statische headers per model.                                          |
| `input`            | `Array<"text" \| "image" \| "document">`                       | Modaliteiten die het model accepteert. Andere waarden worden stilzwijgend verwijderd.            |
| `reasoning`        | `boolean`                                                      | Of het model redeneergedrag beschikbaar stelt.                               |
| `contextWindow`    | `number`                                                       | Systeemeigen contextvenster van de provider.                                             |
| `contextTokens`    | `number`                                                       | Optionele effectieve runtimecontextlimiet wanneer deze afwijkt van `contextWindow`. |
| `maxTokens`        | `number`                                                       | Maximaal aantal uitvoertokens, indien bekend.                                           |
| `thinkingLevelMap` | `Record<string, string \| null>`                               | Optionele model-id- of parameteroverschrijvingen per denkniveau.                    |
| `cost`             | `object`                                                       | Optionele prijs in USD per miljoen tokens, inclusief optionele `tieredPricing`. |
| `compat`           | `object`                                                       | Optionele compatibiliteitsvlaggen die overeenkomen met de compatibiliteit van de OpenClaw-modelconfiguratie.  |
| `mediaInput`       | `object`                                                       | Optionele invoerconfiguratie per modaliteit, momenteel alleen voor afbeeldingen.                   |
| `status`           | `"available"` \| `"preview"` \| `"deprecated"` \| `"disabled"` | Vermeldingsstatus. Onderdruk alleen wanneer de rij helemaal niet mag worden weergegeven.          |
| `statusReason`     | `string`                                                       | Optionele reden die bij een niet-beschikbare status wordt weergegeven.                            |
| `replaces`         | `string[]`                                                     | Oudere providerlokale model-id's die door dit model worden vervangen.                       |
| `replacedBy`       | `string`                                                       | Vervangende providerlokale model-id voor verouderde rijen.                    |
| `tags`             | `string[]`                                                     | Stabiele tags die door keuzelijsten en filters worden gebruikt.                                    |

Onderdrukkingsvelden:

| Veld                       | Type       | Wat het betekent                                                                                           |
| -------------------------- | ---------- | ----------------------------------------------------------------------------------------------------------- |
| `provider`                 | `string`   | Provider-id van de upstreamrij die moet worden onderdrukt. Moet eigendom zijn van deze plugin of als een alias in eigendom zijn gedeclareerd. |
| `model`                    | `string`   | Providerlokale model-id die moet worden onderdrukt.                                                         |
| `reason`                   | `string`   | Optioneel bericht dat wordt weergegeven wanneer de onderdrukte rij rechtstreeks wordt opgevraagd.          |
| `when.baseUrlHosts`        | `string[]` | Optionele lijst met effectieve hosts van de basis-URL van de provider die vereist zijn voordat de onderdrukking wordt toegepast. |
| `when.providerConfigApiIn` | `string[]` | Optionele lijst met exacte `api`-waarden uit de providerconfiguratie die vereist zijn voordat de onderdrukking wordt toegepast. |

Plaats geen gegevens die alleen tijdens runtime beschikbaar zijn in `modelCatalog`. Gebruik `static` alleen wanneer de manifestrijen volledig genoeg zijn zodat op provider gefilterde lijst- en kiezeroppervlakken register-/runtime-ontdekking kunnen overslaan. Gebruik `refreshable` wanneer manifestrijen nuttige vermeldbare beginwaarden of aanvullingen zijn, maar een vernieuwing/cache later meer rijen kan toevoegen; vernieuwbare rijen zijn op zichzelf niet gezaghebbend. Gebruik `runtime` wanneer OpenClaw de providerruntime moet laden om de lijst te kennen.

## Naslag voor modelIdNormalization

Gebruik `modelIdNormalization` voor goedkope, door de provider beheerde opschoning van model-id's die moet plaatsvinden voordat de providerruntime wordt geladen. Hierdoor blijven aliassen zoals korte modelnamen, verouderde providerlokale id's en regels voor proxyprefixen in het manifest van de beherende plugin in plaats van in modelselectietabellen van de kern.

```json
{
  "providers": ["anthropic", "openrouter"],
  "modelIdNormalization": {
    "providers": {
      "anthropic": {
        "aliases": {
          "sonnet-4.6": "claude-sonnet-4-6"
        }
      },
      "openrouter": {
        "prefixWhenBare": "openrouter"
      }
    }
  }
}
```

Providervelden:

| Veld                                 | Type                    | Wat het betekent                                                                          |
| ------------------------------------ | ----------------------- | ------------------------------------------------------------------------------------------ |
| `aliases`                            | `Record<string,string>` | Hoofdletterongevoelige aliassen van exacte model-id's. Waarden worden teruggegeven zoals geschreven. |
| `stripPrefixes`                      | `string[]`              | Prefixen die vóór het opzoeken van aliassen moeten worden verwijderd, nuttig voor verouderde duplicatie van provider/model. |
| `prefixWhenBare`                     | `string`                | Prefix die moet worden toegevoegd wanneer de genormaliseerde model-id nog geen `/` bevat. |
| `prefixWhenBareAfterAliasStartsWith` | `object[]`              | Voorwaardelijke prefixregels voor kale id's na het opzoeken van aliassen, geïndexeerd op `modelPrefix` en `prefix`. |

## Naslag voor providerEndpoints

Gebruik `providerEndpoints` voor eindpuntclassificatie die het algemene aanvraagbeleid moet kennen voordat de providerruntime wordt geladen. De kern bepaalt nog steeds de betekenis van elke `endpointClass`; pluginmanifesten beheren de host- en basis-URL-metagegevens.

Officieel geëxternaliseerde providerplugins zijn uitgesloten van de kerndistributie, zodat
hun manifesten onzichtbaar zijn totdat ze zijn geïnstalleerd. Hun `providerEndpoints` moet
ook worden gespiegeld in `scripts/lib/official-external-provider-catalog.json`, zodat
eindpuntclassificatie zonder de plugin blijft werken; een contracttest
dwingt deze spiegeling af.

Eindpuntvelden:

| Veld                           | Type       | Wat het betekent                                                                                |
| ------------------------------ | ---------- | ------------------------------------------------------------------------------------------------ |
| `endpointClass`                | `string`   | Bekende kerneindpuntklasse, zoals `openrouter`, `moonshot-native` of `google-vertex`. |
| `hosts`                        | `string[]` | Exacte hostnamen die aan de eindpuntklasse worden gekoppeld.                                    |
| `hostSuffixes`                 | `string[]` | Hostsuffixen die aan de eindpuntklasse worden gekoppeld. Voeg `.` als prefix toe om alleen domeinsuffixen te vergelijken. |
| `baseUrls`                     | `string[]` | Exacte genormaliseerde HTTP(S)-basis-URL's die aan de eindpuntklasse worden gekoppeld.           |
| `googleVertexRegion`           | `string`   | Statische Google Vertex-regio voor exacte globale hosts.                                        |
| `googleVertexRegionHostSuffix` | `string`   | Van overeenkomende hosts te verwijderen suffix om de Google Vertex-regioprefix vrij te geven.   |

## Naslag voor providerRequest

Gebruik `providerRequest` voor goedkope metagegevens over aanvraagcompatibiliteit die het algemene aanvraagbeleid nodig heeft zonder de providerruntime te laden. Houd gedragsspecifieke herschrijving van payloads in providerruntimehooks of gedeelde helpers voor providerfamilies.

```json
{
  "providerRequest": {
    "providers": {
      "vllm": {
        "family": "vllm",
        "openAICompletions": {
          "supportsStreamingUsage": true
        }
      }
    }
  }
}
```

Providervelden:

| Veld                  | Type         | Wat het betekent                                                                       |
| --------------------- | ------------ | --------------------------------------------------------------------------------------- |
| `family`              | `string`     | Label van de providerfamilie dat wordt gebruikt voor algemene beslissingen over aanvraagcompatibiliteit en diagnostiek. |
| `compatibilityFamily` | `"moonshot"` | Optionele compatibiliteitsgroep voor providerfamilies voor gedeelde aanvraaghelpers.   |
| `openAICompletions`   | `object`     | Vlaggen voor OpenAI-compatibele voltooiingsaanvragen, momenteel `supportsStreamingUsage`. |

## Naslag voor secretProviderIntegrations

Gebruik `secretProviderIntegrations` wanneer een plugin een herbruikbare voorinstelling voor een SecretRef-exec-provider kan publiceren. OpenClaw leest deze metagegevens voordat de pluginruntime wordt geladen, slaat het plugineigendom op in `secrets.providers.<alias>.pluginIntegration` en laat de daadwerkelijke oplossing van geheimen over aan de SecretRef-runtime. Voorinstellingen worden alleen beschikbaar gesteld voor gebundelde plugins en geïnstalleerde plugins die zijn ontdekt vanuit de beheerde installatieroots voor plugins, zoals git- en ClawHub-installaties.

```json
{
  "secretProviderIntegrations": {
    "secret-store": {
      "providerAlias": "team-secrets",
      "displayName": "Team secrets",
      "source": "exec",
      "command": "${node}",
      "args": ["./bin/resolve-secrets.mjs"]
    }
  }
}
```

De mapsleutel is de integratie-id. Als `providerAlias` wordt weggelaten, gebruikt OpenClaw de integratie-id als de provideralias voor SecretRef. Provideraliassen moeten overeenkomen met het normale patroon voor SecretRef-provideraliassen, bijvoorbeeld `team-secrets` of `onepassword-work`.

Wanneer een beheerder de voorinstelling selecteert, schrijft OpenClaw een providerverwijzing zoals:

```json
{
  "secrets": {
    "providers": {
      "team-secrets": {
        "source": "exec",
        "pluginIntegration": {
          "pluginId": "acme-secrets",
          "integrationId": "secret-store"
        }
      }
    }
  }
}
```

Bij het opstarten/opnieuw laden lost OpenClaw die provider op door de huidige metagegevens van het pluginmanifest te laden, te controleren of de beherende plugin is geïnstalleerd en actief is, en de exec-opdracht uit het manifest te materialiseren. Het uitschakelen of verwijderen van de plugin trekt de provider voor actieve SecretRefs in. Beheerders die een zelfstandige exec-configuratie willen, kunnen nog steeds rechtstreeks handmatige `command`/`args`-providers schrijven.

Momenteel worden alleen `source: "exec"`-voorinstellingen ondersteund. `command` moet `${node}` zijn en `args[0]` moet een `./`-resolverscript relatief aan de pluginroot zijn. OpenClaw materialiseert dit bij het opstarten/opnieuw laden naar het huidige uitvoerbare Node-bestand en het absolute scriptpad binnen de plugin. Node-opties zoals `--require`, `--import`, `--loader`, `--env-file`, `--eval` en `--print` maken geen deel uit van het manifestcontract voor voorinstellingen. Beheerders die niet-Node-opdrachten nodig hebben, kunnen rechtstreeks zelfstandige handmatige exec-providers configureren.

OpenClaw leidt `trustedDirs` voor manifestvoorinstellingen af van de pluginroot en, voor `${node}`-voorinstellingen, van de map van het huidige uitvoerbare Node-bestand. In het manifest opgegeven `trustedDirs` worden genegeerd. Andere exec-provideropties zoals `timeoutMs`, `noOutputTimeoutMs`, `maxOutputBytes`, `jsonOnly`, `env`, `passEnv` en `allowInsecurePath` worden doorgegeven aan de normale SecretRef-configuratie voor exec-providers.

## Naslag voor modelPricing

Gebruik `modelPricing` wanneer een provider prijsstellingsgedrag op het besturingsvlak nodig heeft voordat de runtime wordt geladen. De prijsstellingscache van de Gateway leest deze metagegevens zonder providerruntimecode te importeren.

```json
{
  "providers": ["ollama", "openrouter"],
  "modelPricing": {
    "providers": {
      "ollama": {
        "external": false
      },
      "openrouter": {
        "openRouter": {
          "passthroughProviderModel": true
        },
        "liteLLM": false
      }
    }
  }
}
```

Providervelden:

| Veld         | Type              | Wat het betekent                                                                                       |
| ------------ | ----------------- | ------------------------------------------------------------------------------------------------------- |
| `external`   | `boolean`         | Stel in op `false` voor lokale/zelfgehoste providers die nooit prijsgegevens van OpenRouter of LiteLLM mogen ophalen. |
| `openRouter` | `false \| object` | Toewijzing voor het opzoeken van OpenRouter-prijzen. `false` schakelt het opzoeken via OpenRouter voor deze provider uit. |
| `liteLLM`    | `false \| object` | Toewijzing voor het opzoeken van LiteLLM-prijzen. `false` schakelt het opzoeken via LiteLLM voor deze provider uit. |

Bronvelden:

| Veld                       | Type               | Wat het betekent                                                                                                     |
| -------------------------- | ------------------ | --------------------------------------------------------------------------------------------------------------------- |
| `provider`                 | `string`           | Provider-id in de externe catalogus wanneer deze afwijkt van de OpenClaw-provider-id, bijvoorbeeld `z-ai` voor een `zai`-provider. |
| `passthroughProviderModel` | `boolean`          | Behandel model-id's met schuine strepen als geneste provider-/modelverwijzingen, nuttig voor proxyproviders zoals OpenRouter. |
| `modelIdTransforms`        | `"version-dots"[]` | Extra model-id-varianten voor de externe catalogus. `version-dots` probeert versie-id's met punten, zoals `claude-opus-4.6`. |

### OpenClaw-providerindex

De OpenClaw-providerindex bestaat uit door OpenClaw beheerde voorbeeldmetagegevens voor providers waarvan de plugins mogelijk nog niet zijn geïnstalleerd. Deze maakt geen deel uit van een pluginmanifest. Pluginmanifesten blijven de gezaghebbende bron voor geïnstalleerde plugins. De providerindex is het interne terugvalcontract dat toekomstige oppervlakken voor installeerbare providers en modelkiezers vóór installatie gebruiken wanneer een providerplugin niet is geïnstalleerd.

Volgorde van catalogusautoriteit:

1. Gebruikersconfiguratie.
2. Manifest van geïnstalleerde plugin `modelCatalog`.
3. Modelcataloguscache na expliciet vernieuwen.
4. Voorbeeldrijen van de OpenClaw-providerindex.

De Provider Index mag geen secrets, ingeschakelde status, runtime-hooks of live accountspecifieke modelgegevens bevatten. De voorbeeldcatalogi gebruiken dezelfde `modelCatalog`-provider-rijstructuur als Plugin-manifesten, maar moeten beperkt blijven tot stabiele weergavemetadata, tenzij runtime-adaptervelden zoals `api`, `baseUrl`, prijzen of compatibiliteitsvlaggen bewust worden afgestemd op het geïnstalleerde Plugin-manifest. Providers met live `/models`-detectie moeten vernieuwde rijen via het expliciete cachepad voor de modelcatalogus schrijven, in plaats van provider-API's aan te roepen tijdens normale vermeldingen of onboarding.

Provider Index-vermeldingen kunnen ook metadata voor installeerbare Plugins bevatten voor providers waarvan de Plugin uit de kern is verplaatst of anderszins nog niet is geïnstalleerd. Deze metadata volgt het patroon van de kanaalcatalogus: pakketnaam, npm-installatiespecificatie, verwachte integriteit en eenvoudige labels voor authenticatiekeuzes volstaan om een installeerbare configuratieoptie te tonen. Zodra de Plugin is geïnstalleerd, heeft het manifest daarvan voorrang en wordt de Provider Index-vermelding voor die provider genegeerd.

`openclaw doctor --fix` migreert een kleine, gesloten set verouderde manifest-capaciteitssleutels op het hoogste niveau naar `contracts.*`: `speechProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders` en `tools`. Geen van deze sleutels (of enige andere capaciteitenlijst) wordt nog als manifestveld op het hoogste niveau gelezen; bij normaal laden van manifesten worden ze alleen onder `contracts` herkend.

## Manifest versus package.json

De twee bestanden hebben verschillende functies:

| Bestand                   | Gebruik het voor                                                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Detectie, configuratievalidatie, metadata voor authenticatiekeuzes en UI-aanwijzingen die beschikbaar moeten zijn voordat Plugin-code wordt uitgevoerd                         |
| `package.json`         | npm-metadata, installatie van afhankelijkheden en het `openclaw`-blok dat wordt gebruikt voor toegangspunten, installatievoorwaarden, configuratie of catalogusmetadata |

Als je niet zeker weet waar bepaalde metadata thuishoort, gebruik je deze regel:

- als OpenClaw deze moet kennen voordat Plugin-code wordt geladen, plaats je deze in `openclaw.plugin.json`
- als deze betrekking heeft op pakkettering, toegangsbestanden of npm-installatiegedrag, plaats je deze in `package.json`

### package.json-velden die detectie beïnvloeden

Sommige Plugin-metadata van vóór de runtime staat bewust in `package.json` onder het `openclaw`-blok in plaats van in `openclaw.plugin.json`. `openclaw.bundle` en `openclaw.bundle.json` zijn geen OpenClaw-Plugin-contracten; native Plugins moeten `openclaw.plugin.json` gebruiken, samen met de ondersteunde `package.json#openclaw`-velden hieronder.

Belangrijke voorbeelden:

| Veld                                                                                      | Betekenis                                                                                                                                                                             |
| ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                                                      | Declareert toegangspunten voor native Plugins. Moet binnen de pakketmap van de Plugin blijven.                                                                                                        |
| `openclaw.runtimeExtensions`                                                               | Declareert gebouwde JavaScript-runtime-toegangspunten voor geïnstalleerde pakketten. Moet binnen de pakketmap van de Plugin blijven.                                                                      |
| `openclaw.setupEntry`                                                                      | Lichtgewicht toegangspunt uitsluitend voor configuratie, gebruikt tijdens onboarding, uitgestelde kanaalstart en alleen-lezen kanaalstatus-/SecretRef-detectie. Moet binnen de pakketmap van de Plugin blijven.      |
| `openclaw.runtimeSetupEntry`                                                               | Declareert het gebouwde JavaScript-configuratietoegangspunt voor geïnstalleerde pakketten. Vereist `setupEntry`, moet bestaan en moet binnen de pakketmap van de Plugin blijven.                              |
| `openclaw.channel`                                                                         | Eenvoudige metadata voor de kanaalcatalogus, zoals labels, documentatiepaden, aliassen en selectietekst.                                                                                                      |
| `openclaw.channel.approvalFlags`                                                           | Gesloten gedragsvlaggen voor goedkeuring die beschikbaar zijn voordat de runtime wordt geladen. `native` betekent dat het kanaal eigenaar is van de native goedkeurings-UI en afhandeling binnen dezelfde beurt.                                                |
| `openclaw.channel.commands`                                                                | Statische metadata voor automatische standaardinstellingen van native opdrachten en native Skills, gebruikt door configuratie-, audit- en opdrachtenlijstinterfaces voordat de kanaalruntime wordt geladen.                                               |
| `openclaw.channel.cliAddOptions`                                                           | `openclaw channels add`-opties die eigendom zijn van de Plugin. Elke vermelding declareert `flags`, `description`, optioneel `defaultValue` en optioneel `valueType` (`int` of `list`) voor algemene invoerconversie. |
| `openclaw.channel.configuredState`                                                         | Lichtgewicht metadata voor controle van de geconfigureerde status, waarmee de vraag "bestaat er al een configuratie die uitsluitend de omgeving gebruikt?" kan worden beantwoord zonder de volledige kanaalruntime te laden.                                              |
| `openclaw.channel.persistedAuthState`                                                      | Lichtgewicht metadata voor controle van opgeslagen authenticatie, waarmee de vraag "is er al ergens ingelogd?" kan worden beantwoord zonder de volledige kanaalruntime te laden.                                                    |
| `openclaw.install.clawhubSpec` / `openclaw.install.npmSpec` / `openclaw.install.localPath` | Aanwijzingen voor installatie en updates van gebundelde en extern gepubliceerde Plugins.                                                                                                                        |
| `openclaw.install.defaultChoice`                                                           | Voorkeursinstallatiepad wanneer meerdere installatiebronnen beschikbaar zijn.                                                                                                                       |
| `openclaw.install.minHostVersion`                                                          | Minimaal ondersteunde versie van de OpenClaw-host, met een semver-ondergrens zoals `>=2026.3.22` of `>=2026.5.1-beta.1`.                                                                                  |
| `openclaw.compat.pluginApi`                                                                | Minimaal bereik van de OpenClaw-Plugin-API dat dit pakket vereist, met een semver-ondergrens zoals `>=2026.5.27`.                                                                                      |
| `openclaw.install.expectedIntegrity`                                                       | Verwachte npm-dist-integriteitstekenreeks, zoals `sha512-...`; installatie- en updateflows verifiëren het opgehaalde artefact hiertegen.                                                                 |
| `openclaw.install.allowInvalidConfigRecovery`                                              | Staat een beperkt herstelpad toe waarbij een gebundelde Plugin opnieuw wordt geïnstalleerd wanneer de configuratie ongeldig is.                                                                                                            |
| `openclaw.install.requiredPlatformPackages`                                                | npm-pakketaliassen die aanwezig moeten worden wanneer hun platformbeperkingen in het lockbestand overeenkomen met de huidige host.                                                                                |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`                          | Maakt het mogelijk configuratieruntime-kanaalinterfaces vóór het luisteren te laden en stelt vervolgens de volledige geconfigureerde kanaal-Plugin uit tot activering nadat het luisteren is gestart.                                                      |

Manifestmetadata bepaalt welke provider-, kanaal- en configuratiekeuzes tijdens onboarding worden weergegeven voordat de runtime wordt geladen. `package.json#openclaw.install` vertelt onboarding hoe die Plugin moet worden opgehaald of ingeschakeld wanneer de gebruiker een van die keuzes selecteert. Verplaats installatieaanwijzingen niet naar `openclaw.plugin.json`.

Gebruik voor `openclaw.channel.cliAddOptions` de syntaxis van Commander voor lange opties, zoals `--initial-sync-limit <n>`. Stel `valueType: "int"` in om een niet-negatief geheel getal te parseren of `valueType: "list"` om invoer die door komma's, puntkomma's of nieuwe regels wordt gescheiden op te splitsen in tekenreeksen voordat de configuratieadapter van de Plugin deze ontvangt. Laat `valueType` weg om de geparste Commander-waarde ongewijzigd door te geven.

`openclaw.install.minHostVersion` wordt tijdens installatie en het laden van het manifestregister afgedwongen voor niet-gebundelde Plugin-bronnen. Ongeldige waarden worden geweigerd; nieuwere maar geldige waarden zorgen ervoor dat externe Plugins op oudere hosts worden overgeslagen. Er wordt aangenomen dat gebundelde bron-Plugins dezelfde versie hebben als de host-checkout.

`openclaw.install.requiredPlatformPackages` is bedoeld voor npm-pakketten die vereiste native binaire bestanden beschikbaar stellen via optionele, platformspecifieke aliassen. Vermeld voor elke ondersteunde platformalias de kale npm-pakketnaam. Tijdens de npm-installatie verifieert OpenClaw alleen de gedeclareerde alias waarvan de lockbestandbeperkingen overeenkomen met de huidige host. Als npm succes meldt maar die alias weglaat, probeert OpenClaw het eenmaal opnieuw met een nieuwe cache en draait het de installatie terug als de alias nog steeds ontbreekt.

`openclaw.compat.pluginApi` wordt tijdens pakketinstallatie afgedwongen voor niet-gebundelde Plugin-bronnen. Gebruik dit voor de ondergrens van de OpenClaw-Plugin-SDK/runtime-API waartegen het pakket is gebouwd. Dit kan strenger zijn dan `minHostVersion` wanneer een Plugin-pakket een nieuwere API nodig heeft, maar voor andere flows toch een lagere installatieaanwijzing behoudt. Officiële OpenClaw-releasesynchronisatie verhoogt bestaande officiële ondergrenzen voor Plugin-API's standaard naar de OpenClaw-releaseversie, maar releases die alleen Plugins bevatten kunnen een lagere ondergrens behouden wanneer het pakket bewust oudere hosts ondersteunt. Gebruik niet alleen de pakketversie als compatibiliteitscontract. `peerDependencies.openclaw` blijft npm-pakketmetadata; OpenClaw gebruikt het `openclaw.compat.pluginApi`-contract voor beslissingen over installatiecompatibiliteit.

Officiële metadata voor installatie op aanvraag moet `clawhubSpec` gebruiken wanneer de Plugin op ClawHub wordt gepubliceerd; onboarding behandelt dit als de externe voorkeursbron en registreert na installatie de feiten over het ClawHub-artefact. `npmSpec` blijft de compatibiliteitsterugval voor pakketten die nog niet naar ClawHub zijn verplaatst.

Exacte vastlegging van npm-versies staat al in `npmSpec`, bijvoorbeeld `"npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3"`. Officiële externe catalogusvermeldingen moeten exacte specificaties combineren met `expectedIntegrity`, zodat updateflows veilig stoppen als het opgehaalde npm-artefact niet langer overeenkomt met de vastgelegde release. Interactieve onboarding biedt voor compatibiliteit nog steeds vertrouwde npm-registerspecificaties aan, waaronder kale pakketnamen en dist-tags. Catalogusdiagnostiek kan onderscheid maken tussen exacte, zwevende, op integriteit vastgelegde, zonder integriteit vastgelegde, qua pakketnaam niet-overeenkomende en ongeldige standaardkeuzebronnen. Ook wordt gewaarschuwd wanneer `expectedIntegrity` aanwezig is, maar er geen geldige npm-bron is die ermee kan worden vastgelegd. Wanneer `expectedIntegrity` aanwezig is, dwingen installatie- en updateflows deze af; wanneer deze is weggelaten, wordt de registerresolutie zonder integriteitsvastlegging geregistreerd.

Kanaal-Plugins moeten `openclaw.setupEntry` aanbieden wanneer status-, kanaallijst- of SecretRef-scans geconfigureerde accounts moeten identificeren zonder de volledige runtime te laden. Het configuratietoegangspunt moet kanaalmetadata plus configuratie-, status- en secrets-adapters bevatten die veilig tijdens de configuratie kunnen worden gebruikt; bewaar netwerkclients, Gateway-listeners en transportruntimes in het hoofdtoegangspunt van de extensie.

Runtime-entrypointvelden overschrijven pakketgrenscontroles voor bron-entrypointvelden niet. Zo kan `openclaw.runtimeExtensions` een ontsnappend `openclaw.extensions`-pad niet laadbaar maken.

`openclaw.install.allowInvalidConfigRecovery` is bewust beperkt. Het maakt niet willekeurige defecte configuraties installeerbaar. Momenteel staat het alleen toe dat installatieflows herstellen van specifieke verouderde upgradefouten van gebundelde plugins, zoals een ontbrekend pad naar een gebundelde plugin of een verouderde `channels.<id>`-vermelding voor diezelfde gebundelde plugin. Niet-gerelateerde configuratiefouten blokkeren de installatie nog steeds en verwijzen beheerders naar `openclaw doctor --fix`.

`openclaw.channel.persistedAuthState` is pakketmetadata voor een kleine controlemodule:

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

Gebruik dit wanneer installatie-, doctor-, status- of alleen-lezen-aanwezigheidsflows een goedkope ja/nee-controle van authenticatie nodig hebben voordat de volledige kanaalplugin wordt geladen. Persistente authenticatiestatus is geen geconfigureerde kanaalstatus: gebruik deze metadata niet om plugins automatisch in te schakelen, runtime-afhankelijkheden te repareren of te bepalen of een kanaalruntime moet worden geladen. De doelexport moet een kleine functie zijn die alleen persistente status leest; leid deze niet via de volledige runtime-barrel van het kanaal.

`openclaw.channel.configuredState` ondersteunt goedkope controles op configuratie. Geef de voorkeur aan declaratieve omgevingsmetadata wanneer omgevingsvariabelen voldoende zijn:

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "env": {
          "allOf": ["TELEGRAM_BOT_TOKEN"]
        }
      }
    }
  }
}
```

Gebruik `env.allOf` wanneer elke vermelde variabele vereist is en `env.anyOf` wanneer één niet-lege variabele voldoende is. Als een kleine niet-runtimecontrole meer nodig heeft dan omgevingsmetadata, gebruik dan `specifier` plus `exportName`, zoals weergegeven voor `persistedAuthState`; wanneer `env` aanwezig is, gebruikt OpenClaw dit zonder die module te laden. Als de controle volledige configuratieresolutie of de echte kanaalruntime nodig heeft, houd die logica dan in de `config.hasConfiguredState`-hook van de plugin.

## Detectieprioriteit (dubbele plugin-id's)

OpenClaw detecteert plugins vanuit drie hoofdmappen, die in deze volgorde worden gecontroleerd: gebundelde plugins die met OpenClaw worden geleverd, de globale installatiehoofdmap (`~/.openclaw/extensions`) en de huidige werkruimtehoofdmap (`<workspace>/.openclaw/extensions`), plus eventuele expliciete `plugins.load.paths`-vermeldingen.

Als twee detecties dezelfde `id` delen, wordt alleen het manifest met de **hoogste prioriteit** behouden; duplicaten met een lagere prioriteit worden verwijderd in plaats van ernaast geladen. Prioriteit, van hoog naar laag:

1. **Door configuratie geselecteerd** — een pad dat expliciet is vastgezet in `plugins.entries.<id>`
2. **Globale installatie die overeenkomt met een bijgehouden installatierecord** — een plugin die via `openclaw plugin install`/`openclaw plugin update` is geïnstalleerd en die door de installatieregistratie van OpenClaw voor diezelfde id wordt herkend, zelfs wanneer de id ook bij een gebundelde plugin hoort
3. **Gebundeld** — plugins die met OpenClaw worden geleverd
4. **Werkruimte** — plugins die relatief aan de huidige werkruimte worden gedetecteerd
5. Elke andere gedetecteerde kandidaat

Gevolgen:

- Een geforkte of verouderde kopie van een gebundelde plugin die niet wordt bijgehouden en zich in de werkruimte of globale hoofdmap bevindt, overschaduwt de gebundelde build niet.
- Om een gebundelde plugin te overschrijven, voer je `openclaw plugin install` uit voor die id, zodat de bijgehouden globale installatie een hogere prioriteit krijgt dan de gebundelde kopie, of zet je een specifiek pad vast via `plugins.entries.<id>`, zodat dit wint op basis van door configuratie geselecteerde prioriteit.
- Het verwijderen van duplicaten wordt gelogd, zodat Doctor en diagnostiek bij het opstarten naar de verworpen kopie kunnen verwijzen.
- Door configuratie geselecteerde overschrijvingen van duplicaten worden in de diagnostiek als expliciete overschrijvingen beschreven, maar genereren nog steeds een waarschuwing, zodat verouderde forks en onbedoelde overschaduwingen zichtbaar blijven.

## Vereisten voor JSON Schema

- **Elke plugin moet een JSON Schema meeleveren**, zelfs als deze geen configuratie accepteert.
- Een leeg schema is toegestaan (bijvoorbeeld `{ "type": "object", "additionalProperties": false }`).
- Schema's worden gevalideerd wanneer configuratie wordt gelezen of geschreven, niet tijdens runtime.
- Wanneer je een gebundelde plugin uitbreidt of forkt met nieuwe configuratiesleutels, werk dan tegelijkertijd de `openclaw.plugin.json` `configSchema` van die plugin bij. Schema's van gebundelde plugins zijn strikt, dus het toevoegen van `plugins.entries.<id>.config.myNewKey` aan de gebruikersconfiguratie zonder `myNewKey` aan `configSchema.properties` toe te voegen, wordt geweigerd voordat de runtime van de plugin wordt geladen.

Voorbeeld van een schema-uitbreiding:

```json
{
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "myNewKey": {
        "type": "string"
      }
    }
  }
}
```

## Validatiegedrag

- Onbekende `channels.*`-sleutels zijn **fouten**, tenzij de kanaal-id door een pluginmanifest wordt gedeclareerd. Als dezelfde id ook voorkomt in `plugins.allow`, `plugins.entries` of `plugins.installs` (een plugin waarnaar wordt verwezen maar die momenteel niet kan worden gedetecteerd), verlaagt OpenClaw dit in plaats daarvan tot een **waarschuwing**.
- `plugins.entries.<id>`, `plugins.allow` en `plugins.deny` die naar onbekende plugin-id's verwijzen, zijn **waarschuwingen** ("verouderde configuratievermelding genegeerd"), geen fouten, zodat upgrades en verwijderde of hernoemde plugins het starten van de Gateway niet blokkeren.
- `plugins.slots.memory` die naar een onbekende plugin-id verwijst, is een **fout**, behalve voor de bekende officiële externe plugin `memory-lancedb`, waarvoor in plaats daarvan een waarschuwing wordt gegeven.
- Als een plugin is geïnstalleerd maar een defect of ontbrekend manifest of schema heeft, mislukt de validatie en rapporteert Doctor de pluginfout.
- Als er pluginconfiguratie bestaat maar de plugin **uitgeschakeld** is, wordt de configuratie behouden en verschijnt er een **waarschuwing** in Doctor en de logboeken.

Zie [Configuratiereferentie](/nl/gateway/configuration) voor het volledige `plugins.*`-schema.

## Opmerkingen

- Het manifest is **vereist voor systeemeigen OpenClaw-plugins**, inclusief laden vanuit het lokale bestandssysteem. De runtime laadt de pluginmodule nog steeds afzonderlijk; het manifest dient alleen voor detectie en validatie.
- Systeemeigen manifesten worden geparseerd met JSON5, zodat opmerkingen, afsluitende komma's en sleutels zonder aanhalingstekens worden geaccepteerd zolang de uiteindelijke waarde nog steeds een object is.
- Alleen gedocumenteerde manifestvelden worden door de manifestlader gelezen. Vermijd aangepaste sleutels op het hoogste niveau.
- `channels`, `providers`, `cliBackends` en `skills` kunnen allemaal worden weggelaten wanneer een plugin ze niet nodig heeft.
- `providerCatalogEntry` moet lichtgewicht blijven en mag geen brede runtimecode importeren; gebruik dit voor statische metadata van de providercatalogus of beperkte detectiedescriptors, niet voor uitvoering tijdens aanvragen.
- Exclusieve plugintypen worden geselecteerd via `plugins.slots.*`: `kind: "memory"` via `plugins.slots.memory` (standaard `memory-core`), `kind: "context-engine"` via `plugins.slots.contextEngine` (standaard `legacy`).
- Declareer het exclusieve plugintype in dit manifest. Runtime-entry `OpenClawPluginDefinition.kind` is verouderd en blijft alleen bestaan als compatibiliteitsfallback voor oudere plugins.
- Metadata voor omgevingsvariabelen in `setup.providers[].envVars` is alleen declaratief. Status, audit, validatie van Cron-bezorging en andere alleen-lezen-oppervlakken passen nog steeds het pluginvertrouwen en het effectieve activeringsbeleid toe voordat een omgevingsvariabele als geconfigureerd wordt beschouwd.
- Zie [Runtime-hooks voor providers](/nl/plugins/architecture-internals#provider-runtime-hooks) voor runtimewizardmetadata waarvoor providercode vereist is.
- Als je plugin afhankelijk is van systeemeigen modules, documenteer dan de buildstappen en eventuele vereisten voor de toelatingslijst van de pakketbeheerder (bijvoorbeeld pnpm `allow-build-scripts` + `pnpm rebuild <package>`).

## Gerelateerd

<CardGroup cols={3}>
  <Card title="Plugins bouwen" href="/nl/plugins/building-plugins" icon="rocket">
    Aan de slag met plugins.
  </Card>
  <Card title="Pluginarchitectuur" href="/nl/plugins/architecture" icon="diagram-project">
    Interne architectuur en capaciteitenmodel.
  </Card>
  <Card title="SDK-overzicht" href="/nl/plugins/sdk-overview" icon="book">
    Naslaginformatie voor de Plugin-SDK en imports van subpaden.
  </Card>
</CardGroup>
