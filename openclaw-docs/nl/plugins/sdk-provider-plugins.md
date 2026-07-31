---
read_when:
    - Je bouwt een nieuwe modelproviderplugin
    - Je wilt een OpenAI-compatibele proxy of aangepast LLM aan OpenClaw toevoegen
    - Je moet providerauthenticatie, catalogi en runtime-hooks begrijpen
sidebarTitle: Provider plugins
summary: Stapsgewijze handleiding voor het bouwen van een modelproviderplugin voor OpenClaw
title: Providerplugins bouwen
x-i18n:
    generated_at: "2026-07-27T05:28:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9d175fafc034bd52e996d47e047df104f079f2aba66662b22e8dbdf6c21e7e0
    source_path: plugins/sdk-provider-plugins.md
    workflow: 16
---

Bouw een providerplugin om een modelprovider (LLM) aan OpenClaw toe te voegen: een modelcatalogus, authenticatie met API-sleutel en dynamische modelresolutie.

<Info>
  Nieuw met OpenClaw-plugins? Lees eerst [Aan de slag](/nl/plugins/building-plugins)
  voor de pakketstructuur en het instellen van het manifest.
</Info>

<Tip>
  Providerplugins voegen modellen toe aan de normale inferentielus van OpenClaw. Als het
  model moet worden uitgevoerd via een native agentdaemon die threads, Compaction
  of toolgebeurtenissen beheert, combineer de provider dan met een [agent-
  harness](/nl/plugins/sdk-agent-harness) in plaats van details van het daemonprotocol
  in de core te plaatsen.
</Tip>

## Stapsgewijze uitleg

<Steps>
  <Step title="Pakket en manifest">
    ### Stap 1: Pakket en manifest

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-ai",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "providers": ["acme-ai"],
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
      "id": "acme-ai",
      "name": "Acme AI",
      "description": "Acme AI-modelprovider",
      "providers": ["acme-ai"],
      "modelSupport": {
        "modelPrefixes": ["acme-"]
      },
      "setup": {
        "providers": [
          {
            "id": "acme-ai",
            "envVars": ["ACME_AI_API_KEY"]
          }
        ]
      },
      "providerAuthAliases": {
        "acme-ai-coding": "acme-ai"
      },
      "providerAuthChoices": [
        {
          "provider": "acme-ai",
          "method": "api-key",
          "choiceId": "acme-ai-api-key",
          "choiceLabel": "API-sleutel voor Acme AI",
          "groupId": "acme-ai",
          "groupLabel": "Acme AI",
          "cliFlag": "--acme-ai-api-key",
          "cliOption": "--acme-ai-api-key <key>",
          "cliDescription": "API-sleutel voor Acme AI"
        }
      ],
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    Met `setup.providers[].envVars` kan OpenClaw aanmeldgegevens detecteren zonder
    de runtime van je Plugin te laden. Voeg `providerAuthAliases` toe wanneer een providervariant
    de authenticatie van een andere provider-id moet hergebruiken. `modelSupport` is
    optioneel en laat OpenClaw je providerplugin automatisch laden op basis van verkorte
    model-id's zoals `acme-large`, voordat runtimehooks bestaan. `openclaw.compat`
    en `openclaw.build` in `package.json` zijn vereist voor publicatie op ClawHub
    (`openclaw.compat.pluginApi` en `openclaw.build.openclawVersion`
    zijn de twee vereiste velden; `minGatewayVersion` valt terug op
    `openclaw.install.minHostVersion` wanneer het wordt weggelaten).

  </Step>

  <Step title="De provider registreren">
    Een minimale tekstprovider heeft een `id`, `label`, `auth` en `catalog` nodig.
    `catalog` is de runtime-/configuratiehook die eigendom is van de provider; deze kan live
    leveranciers-API's aanroepen en retourneert `models.providers`-items.

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth";

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI-modelprovider",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          docsPath: "/providers/acme-ai",
          envVars: ["ACME_AI_API_KEY"],

          auth: [
            createProviderApiKeyAuthMethod({
              providerId: "acme-ai",
              methodId: "api-key",
              label: "API-sleutel voor Acme AI",
              hint: "API-sleutel uit je Acme AI-dashboard",
              optionKey: "acmeAiApiKey",
              flagName: "--acme-ai-api-key",
              envVar: "ACME_AI_API_KEY",
              promptMessage: "Voer je API-sleutel voor Acme AI in",
              defaultModel: "acme-ai/acme-large",
            }),
          ],

          catalog: {
            order: "simple",
            run: async (ctx) => {
              const apiKey =
                ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: {
                  baseUrl: "https://api.acme-ai.com/v1",
                  apiKey,
                  api: "openai-completions",
                  models: [
                    {
                      id: "acme-large",
                      name: "Acme Large",
                      reasoning: true,
                      input: ["text", "image"],
                      cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
                      contextWindow: 200000,
                      maxTokens: 32768,
                    },
                    {
                      id: "acme-small",
                      name: "Acme Small",
                      reasoning: false,
                      input: ["text"],
                      cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
                      contextWindow: 128000,
                      maxTokens: 8192,
                    },
                  ],
                },
              };
            },
          },
        });

        api.registerModelCatalogProvider({
          provider: "acme-ai",
          kinds: ["text"],
          liveCatalog: async (ctx) => {
            const apiKey = ctx.resolveProviderApiKey("acme-ai").apiKey;
            if (!apiKey) return null;
            return [
              {
                kind: "text",
                provider: "acme-ai",
                model: "acme-large",
                label: "Acme Large",
                source: "live",
              },
            ];
          },
        });
      },
    });
    ```

    `registerModelCatalogProvider` is het nieuwere catalogusoppervlak van het besturingsvlak
    voor de gebruikersinterface voor lijsten, hulp en selectie, met ondersteuning voor rijen van `text`, `voice`, `image_generation`,
    `video_generation` en `music_generation`. Houd aanroepen naar leverancierseindpunten
    en het toewijzen van antwoorden in de Plugin; OpenClaw beheert de gedeelde rijstructuur,
    bronlabels en de weergave van hulp.

    Dit is een werkende provider. Gebruikers kunnen nu
    `openclaw onboard --acme-ai-api-key <key>` uitvoeren en
    `acme-ai/acme-large` als model selecteren.

    ### Live modeldetectie

    Als je provider een OpenAI-compatibele `/models`-API aanbiedt, meld je de
    helper voor één provider aan voor gedeelde detectie:

    ```typescript
    catalog: {
      buildProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      buildStaticProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      liveModelDiscovery: true,
    },
    ```

    `liveModelDiscovery: true` is een openbaar Plugin SDK-contract met het volgende
    gedrag:

    | Onderdeel | Contract |
    | --- | --- |
    | Aanmeldgegevens | Detectie gebruikt de opgeloste provideraanmeldgegevens van de catalogus, met voorkeur voor `discoveryApiKey` wanneer de authenticatie deze verstrekt. Markeringen voor geheimreferenties worden nooit als tokens verzonden. Het standaardverzoek gebruikt `Authorization: Bearer <token>`; gebruik `buildRequestHeaders` voor een ander authenticatieschema van de leverancier. |
    | Eindpunt | De standaard-URL is `models` relatief ten opzichte van de effectieve `baseUrl` van de provider, inclusief een overschrijving door de beheerder wanneer `allowExplicitBaseUrl` is ingeschakeld. Gebruik `endpointPath` voor een ander relatief pad. Gebruik `endpointUrl: { url, requireBaseUrl }` alleen voor een vaste leveranciers-URL; detectie wordt overgeslagen tenzij de effectieve basis-URL nog steeds gelijk is aan `requireBaseUrl`, zodat aanmeldgegevens voor een aangepaste proxy niet naar de leverancier worden verzonden. |
    | Netwerklimieten | Ophaalbewerkingen gebruiken de SSRF-beveiliging van OpenClaw, één time-outbudget van 5 seconden voor alle paginering, een antwoordlimiet van 4 MiB per pagina en een limiet van 50 pagina's. Pagineringlinks naar een andere origin worden geweigerd; aanmeldgegevens worden verwijderd na een omleiding naar een andere origin. |
    | Cache | Geslaagde, niet-lege catalogi worden 60 seconden gecachet per provider, eindpunt en opgeloste aanmeldgegevens. Lege of onbruikbare resultaten worden niet gecachet. |
    | Filteren | Exacte live-id's behouden hun vertrouwde statische metagegevens. Nieuwe rijen worden conservatief geprojecteerd als tekst-/chatmodellen. Uitgeschakelde, gearchiveerde, verouderde, expliciet niet voor chat bestemde, embedding-, herrangschikkings-, moderatie-, spraak-, uitsluitend afbeeldings- en uitsluitend videorijen worden uitgesloten. Gebruik `readRows` alleen om rijen te selecteren uit een niet-standaard antwoordenvelop; providerspecifieke modelsemantiek hoort nog steeds thuis in een aangepaste catalogus. |
    | Fout | Live detectie is adviserend. Fouten met authenticatie, netwerk, time-out, paginering, parsering, een lege catalogus en filtering retourneren de statische seed die eigendom is van de provider, in plaats van de provider te verwijderen. |

    Geef voor een niet-Bearer- of niet-standaard lijsteindpunt opties door in plaats van
    `true`:

    ```typescript
    liveModelDiscovery: {
      endpointPath: "model-catalog",
      buildRequestHeaders: ({ apiKey, discoveryApiKey }) => ({
        "vendor-version": "2026-01-01",
        "x-api-key": discoveryApiKey ?? apiKey ?? "",
      }),
      readRows: (body) =>
        body && typeof body === "object" &&
        Array.isArray((body as { models?: unknown }).models)
          ? (body as { models: unknown[] }).models
          : [],
    },
    ```

    Gebruik `endpointUrl` niet als onvoorwaardelijke alternatieve host. De
    `requireBaseUrl`-controle ervan vormt de grens voor het isoleren van aanmeldgegevens voor providers
    waarvan de host voor de modellenlijst verschilt van de host voor inferentie.

    Als de provider aangepaste modelsemantiek nodig heeft in plaats van de conservatieve
    OpenAI-compatibele projectie, houd die projectie dan in de Plugin en gebruik
    `openclaw/plugin-sdk/provider-catalog-live-runtime` voor de gedeelde ophaallevenscyclus.
    De helper biedt beveiligde HTTP-ophaalbewerkingen, provider-authenticatieheaders,
    gestructureerde HTTP-fouten, TTL-caching en statisch terugvalgedrag zonder
    providerbeleid in de OpenClaw-core te plaatsen.

    Gebruik `buildLiveModelProviderConfig` wanneer de live-API je alleen vertelt welke
    statische catalogusrijen die eigendom zijn van de provider momenteel beschikbaar zijn:

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      buildLiveModelProviderConfig,
      type LiveModelCatalogFetchGuard,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    const STATIC_MODELS = [
      {
        id: "acme-large",
        name: "Acme Large",
        reasoning: true,
        input: ["text", "image"],
        cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
        contextWindow: 200000,
        maxTokens: 32768,
      },
      {
        id: "acme-small",
        name: "Acme Small",
        reasoning: false,
        input: ["text"],
        cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
        contextWindow: 128000,
        maxTokens: 8192,
      },
    ] as const;

    async function buildAcmeLiveProvider(params: {
      apiKey: string;
      discoveryApiKey?: string;
      fetchGuard?: LiveModelCatalogFetchGuard;
    }) {
      return await buildLiveModelProviderConfig({
        providerId: "acme-ai",
        endpoint: "https://api.acme-ai.com/v1/models",
        providerConfig: {
          baseUrl: "https://api.acme-ai.com/v1",
          api: "openai-completions",
        },
        models: STATIC_MODELS,
        apiKey: params.apiKey,
        discoveryApiKey: params.discoveryApiKey,
        fetchGuard: params.fetchGuard,
        ttlMs: 60_000,
        auditContext: "acme-ai-model-discovery",
      });
    }

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          catalog: {
            order: "simple",
            run: async (ctx) => {
              const auth = ctx.resolveProviderAuth("acme-ai");
              const apiKey =
                auth.apiKey ?? ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: await buildAcmeLiveProvider({
                  apiKey,
                  discoveryApiKey: auth.discoveryApiKey,
                }),
              };
            },
          },
          staticCatalog: {
            order: "simple",
            run: async () => ({
              provider: {
                baseUrl: "https://api.acme-ai.com/v1",
                api: "openai-completions",
                models: [...STATIC_MODELS],
              },
            }),
          },
        });
      },
    });
    ```

    Gebruik `getCachedLiveProviderModelRows` wanneer de provider-API uitgebreidere
    metadata retourneert en de plugin de rijen zelf naar OpenClaw-modeldefinities
    moet omzetten:

    ```typescript index.ts
    import {
      getCachedLiveProviderModelRows,
      LiveModelCatalogHttpError,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    async function discoverAcmeModels(apiKey: string) {
      try {
        const rows = await getCachedLiveProviderModelRows({
          providerId: "acme-ai",
          endpoint: "https://api.acme-ai.com/v1/models",
          apiKey,
          ttlMs: 60_000,
          auditContext: "acme-ai-model-discovery",
        });
        return rows
          .map((row) => projectAcmeModel(row))
          .filter((model) => model !== null);
      } catch (error) {
        if (error instanceof LiveModelCatalogHttpError) {
          return STATIC_MODELS;
        }
        throw error;
      }
    }
    ```

    `run` moet door authenticatie afgeschermd blijven en `null` retourneren wanneer er geen bruikbare aanmeldgegevens
    beschikbaar zijn. Zorg voor een offline `staticRun` of statische terugvaloptie, zodat installatie, documentatie,
    tests en selectie-interfaces niet afhankelijk zijn van live netwerktoegang. Gebruik een TTL
    die geschikt is voor de actualiteit van de modellenlijst, vermijd het pollen van het bestandssysteem tijdens verzoeken
    en geef alleen een providerspecifieke `readRows` / `readModelId` door wanneer het
    upstream-antwoord geen OpenAI-compatibele `{ data: [{ id, object }] }`-vorm
    heeft.

    Als de upstream-provider andere besturingstokens gebruikt dan OpenClaw, voeg dan een
    kleine bidirectionele teksttransformatie toe in plaats van het streampad te vervangen:

    ```typescript
    api.registerTextTransforms({
      input: [
        { from: /red basket/g, to: "blue basket" },
        { from: /paper ticket/g, to: "digital ticket" },
        { from: /left shelf/g, to: "right shelf" },
      ],
      output: [
        { from: /blue basket/g, to: "red basket" },
        { from: /digital ticket/g, to: "paper ticket" },
        { from: /right shelf/g, to: "left shelf" },
      ],
    });
    ```

    `input` herschrijft de uiteindelijke systeemprompt en de inhoud van tekstberichten vóór
    het transport. `output` herschrijft tekstincrementen van de assistent en de uiteindelijke tekst voordat
    OpenClaw zijn eigen besturingsmarkeringen verwerkt of deze via een kanaal aflevert.

    Geef voor gebundelde providers die slechts één tekstprovider met API-sleutelauthenticatie
    plus één runtime op basis van een catalogus registreren de voorkeur aan de specifiekere
    helper `defineSingleProviderPluginEntry(...)`:

    ```typescript
    import { defineSingleProviderPluginEntry } from "openclaw/plugin-sdk/provider-entry";

    export default defineSingleProviderPluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI-modelprovider",
      provider: {
        label: "Acme AI",
        docsPath: "/providers/acme-ai",
        auth: [
          {
            methodId: "api-key",
            label: "Acme AI-API-sleutel",
            hint: "API-sleutel uit je Acme AI-dashboard",
            optionKey: "acmeAiApiKey",
            flagName: "--acme-ai-api-key",
            envVar: "ACME_AI_API_KEY",
            promptMessage: "Voer je Acme AI-API-sleutel in",
            defaultModel: "acme-ai/acme-large",
          },
        ],
        catalog: {
          buildProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
          buildStaticProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
        },
      },
    });
    ```

    `buildProvider` is het live cataloguspad dat wordt gebruikt wanneer OpenClaw echte
    providerauthenticatie kan achterhalen. Het mag providerspecifieke detectie uitvoeren. Gebruik
    `buildStaticProvider` alleen voor offline rijen die veilig kunnen worden weergegeven voordat authenticatie
    is geconfigureerd; hiervoor mogen geen aanmeldgegevens of netwerkverzoeken nodig zijn.
    De `models list --all`-weergave van OpenClaw voert statische catalogi momenteel
    alleen uit voor gebundelde providerplugins, met een lege configuratie, een lege omgeving en zonder
    agent-/werkruimtepaden.

    Als je authenticatiestroom tijdens de onboarding ook `models.providers.*`, aliassen en
    het standaardmodel van de agent moet aanpassen, gebruik dan de vooraf ingestelde helpers uit
    `openclaw/plugin-sdk/provider-onboard`. De specifiekste helpers zijn
    `createDefaultModelPresetAppliers(...)`,
    `createDefaultModelsPresetAppliers(...)` en
    `createModelCatalogPresetAppliers(...)`.

    Wanneer het native eindpunt van een provider gestreamde gebruiksblokken via het
    normale `openai-completions`-transport ondersteunt, geef dan de voorkeur aan de gedeelde catalogushelpers in
    `openclaw/plugin-sdk/provider-catalog-shared` in plaats van controles
    op provider-ID's hard te coderen. `supportsNativeStreamingUsageCompat(...)` en
    `applyProviderNativeStreamingUsageCompat(...)` detecteren ondersteuning aan de hand van de
    mogelijkhedenkaart van het eindpunt, zodat native eindpunten in Moonshot-/DashScope-stijl zich nog steeds
    aanmelden, zelfs wanneer een plugin een aangepaste provider-ID gebruikt.

    De bovenstaande voorbeelden voor live detectie behandelen provider-API's in `/models`-stijl. Houd
    die detectie binnen `catalog.run`, afgeschermd op basis van bruikbare authenticatie, en zorg dat
    `staticRun` netwerkvrij blijft voor het genereren van offline catalogi.

  </Step>

  <Step title="Dynamische modelresolutie toevoegen">
    Als je provider willekeurige model-ID's accepteert (zoals een proxy of router),
    voeg dan `resolveDynamicModel` toe:

    ```typescript
    api.registerProvider({
      // ... id, label, auth, catalog from above

      resolveDynamicModel: (ctx) => ({
        id: ctx.modelId,
        name: ctx.modelId,
        provider: "acme-ai",
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        reasoning: false,
        input: ["text"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 8192,
      }),
    });
    ```

    Als voor de resolutie een netwerkoproep nodig is, gebruik dan `prepareDynamicModel` voor asynchrone
    opwarming; `resolveDynamicModel` wordt opnieuw uitgevoerd nadat deze is voltooid.

  </Step>

  <Step title="Runtime-hooks toevoegen (indien nodig)">
    De meeste providers hebben alleen `catalog` + `resolveDynamicModel` nodig. Voeg hooks
    stapsgewijs toe wanneer je provider ze nodig heeft.

    Gedeelde helperbouwers ondersteunen nu de meest voorkomende families voor replay-/toolcompatibiliteit,
    zodat plugins meestal niet elke hook afzonderlijk handmatig hoeven te koppelen:

    ```typescript
    import { buildProviderReplayFamilyHooks } from "openclaw/plugin-sdk/provider-model-shared";
    import { buildProviderStreamFamilyHooks } from "openclaw/plugin-sdk/provider-stream";
    import { buildProviderToolCompatFamilyHooks } from "openclaw/plugin-sdk/provider-tools";

    const GOOGLE_FAMILY_HOOKS = {
      ...buildProviderReplayFamilyHooks({ family: "google-gemini" }),
      ...buildProviderStreamFamilyHooks("google-thinking"),
      ...buildProviderToolCompatFamilyHooks("gemini"),
    };

    api.registerProvider({
      id: "acme-gemini-compatible",
      // ...
      ...GOOGLE_FAMILY_HOOKS,
    });
    ```

    Momenteel beschikbare replayfamilies:

    | Familie | Wat hiermee wordt gekoppeld | Gebundelde voorbeelden |
    | --- | --- | --- |
    | `openai-compatible` | Gedeeld replaybeleid in OpenAI-stijl voor OpenAI-compatibele transporten, inclusief opschoning van tool-call-ID's, correcties voor ordening met de assistent eerst en generieke validatie van Gemini-beurten waar het transport dit nodig heeft | `moonshot`, `ollama`, `xai`, `zai` |
    | `anthropic-by-model` | Claude-bewust replaybeleid dat wordt gekozen door `modelId`, zodat transporten voor Anthropic-berichten alleen Claude-specifieke opschoning van denkblokken krijgen wanneer het opgeloste model daadwerkelijk een Claude-ID is | `amazon-bedrock` |
    | `native-anthropic-by-model` | Hetzelfde Claude-per-modelbeleid als `anthropic-by-model`, plus opschoning van tool-call-ID's en behoud van native Anthropic-toolgebruik-ID's voor transporten die leverancierspecifieke native ID's moeten behouden | `anthropic-vertex`, `clawrouter` |
    | `google-gemini` | Native Gemini-replaybeleid plus opschoning van bootstrap-replay. De gedeelde familie houdt de Gemini CLI met tekstuitvoer op gelabelde redenering; de directe provider `google` overschrijft `resolveReasoningOutputMode` met `native`, omdat denkprocessen van de Gemini API als native gedachteonderdelen binnenkomen. | `google`, `google-gemini-cli` |
    | `passthrough-gemini` | Opschoning van Gemini-gedachtehandtekeningen voor Gemini-modellen die via OpenAI-compatibele proxytransporten worden uitgevoerd; schakelt geen native Gemini-replayvalidatie of bootstrap-herschrijvingen in | `openrouter`, `kilocode`, `opencode`, `opencode-go` |
    | `hybrid-anthropic-openai` | Hybride beleid voor providers die oppervlakken voor Anthropic-berichten en OpenAI-compatibele modellen in één plugin combineren; het optioneel verwijderen van alleen Claude-denkblokken blijft beperkt tot de Anthropic-zijde | `minimax` |

    Momenteel beschikbare streamfamilies:

    | Familie | Wat deze integreert | Meegeleverde voorbeelden |
    | --- | --- | --- |
    | `google-thinking` | Normalisatie van Gemini-thinkingpayloads in het gedeelde streampad | `google`, `google-gemini-cli` |
    | `kilocode-thinking` | Kilo-reasoningwrapper in het gedeelde proxystreampad, waarbij `kilo-auto/balanced` en niet-ondersteunde reasoning-id's van de proxy de geïnjecteerde thinking overslaan | `kilocode` |
    | `moonshot-thinking` | Toewijzing van native Moonshot-payloads voor binaire thinking vanuit de configuratie + het niveau `/think` | `moonshot` |
    | `minimax-fast-mode` | Herschrijving van MiniMax-modellen voor de snelle modus in het gedeelde streampad | `minimax`, `minimax-portal` |
    | `openai-responses-defaults` | Gedeelde native OpenAI/Codex Responses-wrappers: attributieheaders, `/fast`/`serviceTier`, tekstuitvoerigheid, native Codex-webzoekfunctie, vormgeving van reasoning-compatibiliteitspayloads en Responses-contextbeheer | `openai` |
    | `openrouter-thinking` | OpenRouter-reasoningwrapper voor proxyroutes, waarbij overslaan voor niet-ondersteunde modellen/`auto` centraal wordt afgehandeld | `openrouter` |
    | `tool-stream-default-on` | Standaard ingeschakelde `tool_stream`-wrapper voor providers zoals Z.AI die toolstreaming willen, tenzij dit expliciet is uitgeschakeld | `zai` |

    <Accordion title="SDK-koppelvlakken waarop de familiebouwers zijn gebaseerd">
      Elke familiebouwer is samengesteld uit openbare helpers op lager niveau die vanuit hetzelfde pakket worden geëxporteerd. Je kunt deze gebruiken wanneer een provider van het algemene patroon moet afwijken:

      - `openclaw/plugin-sdk/provider-model-shared` - `ProviderReplayFamily`, `buildProviderReplayFamilyHooks(...)` en de onbewerkte replaybouwers (`buildOpenAICompatibleReplayPolicy`, `buildAnthropicReplayPolicyForModel`, `buildGoogleGeminiReplayPolicy`, `buildHybridAnthropicOrOpenAIReplayPolicy`). Exporteert ook Gemini-replayhelpers (`sanitizeGoogleGeminiReplayHistory`, `resolveTaggedReasoningOutputMode`) en helpers voor endpoints/modellen (`resolveProviderEndpoint`, `normalizeProviderId`, `normalizeGooglePreviewModelId`).
      - `openclaw/plugin-sdk/provider-stream` - `ProviderStreamFamily`, `buildProviderStreamFamilyHooks(...)`, `composeProviderStreamWrappers(...)`, plus de gedeelde OpenAI/Codex-wrappers (`createOpenAIAttributionHeadersWrapper`, `createOpenAIFastModeWrapper`, `createOpenAIServiceTierWrapper`, `createOpenAIResponsesContextManagementWrapper`, `createCodexNativeWebSearchWrapper`), de OpenAI-compatibele DeepSeek V4-wrapper (`createDeepSeekV4OpenAICompatibleThinkingWrapper`), opschoning van thinking-prefill voor Anthropic Messages (`createAnthropicThinkingPrefillPayloadWrapper`), compatibiliteit met toolaanroepen in platte tekst (`createPlainTextToolCallCompatWrapper`) en gedeelde proxy-/providerwrappers (`createOpenRouterWrapper`, `createToolStreamWrapper`, `createMinimaxFastModeWrapper`).
      - `openclaw/plugin-sdk/provider-stream-shared` - lichtgewicht payload- en eventwrappers voor intensief gebruikte providerpaden, waaronder `createOpenAICompatibleCompletionsThinkingOffWrapper`, `createPayloadPatchStreamWrapper`, `createPlainTextToolCallCompatWrapper`, `normalizeOpenAICompatibleReasoningPayload(...)` en `setQwenChatTemplateThinking(...)`.
      - `openclaw/plugin-sdk/provider-tools` - `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks("deepseek" | "gemini" | "openai")` en onderliggende providerschemahelpers.

      Houd voor providers uit de Gemini-familie de modus voor reasoning-uitvoer afgestemd op
      het transport. Providers die rechtstreeks de Google Gemini API gebruiken, moeten `native`-
      reasoning-uitvoer gebruiken, zodat OpenClaw native thought-onderdelen verwerkt zonder
      `<think>`- / `<final>`-promptinstructies toe te voegen. Gemini CLI-achtige
      backends die alleen tekst gebruiken en een definitief JSON-/tekstantwoord parseren, kunnen het gedeelde
      getagde `google-gemini`-contract behouden.

      Sommige streamhelpers blijven bewust lokaal bij de provider. `@openclaw/anthropic-provider` houdt `wrapAnthropicProviderStream`, `resolveAnthropicBetas`, `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` en de Anthropic-wrapperbouwers op lager niveau in het eigen openbare `api.ts`- / `contract-api.ts`-koppelvlak, omdat deze de afhandeling van Claude OAuth-bèta en `context1m`-beperking coderen. De xAI-plugin houdt op vergelijkbare wijze de vormgeving van native xAI Responses in de eigen `wrapStreamFn` (`/fast`-aliassen, standaard `tool_stream`, opschoning van niet-ondersteunde strikte tools, xAI-specifieke verwijdering van reasoning-payloads).

      Hetzelfde patroon voor de pakketroot ligt ook ten grondslag aan `@openclaw/openai-provider` (providerbouwers, helpers voor standaardmodellen, realtime-providerbouwers) en `@openclaw/openrouter-provider` (providerbouwer plus helpers voor onboarding/configuratie).
    </Accordion>

    <Tabs>
      <Tab title="Tokenuitwisseling">
        Voor providers die vóór elke inferentieaanroep een tokenuitwisseling nodig hebben:

        ```typescript
        prepareRuntimeAuth: async (ctx) => {
          const exchanged = await exchangeToken(ctx.apiKey);
          return {
            apiKey: exchanged.token,
            baseUrl: exchanged.baseUrl,
            expiresAt: exchanged.expiresAt,
          };
        },
        ```
      </Tab>
      <Tab title="Aangepaste headers">
        Voor providers die aangepaste aanvraagheaders of wijzigingen in de body nodig hebben:

        ```typescript
        // wrapStreamFn retourneert een StreamFn die is afgeleid van ctx.streamFn
        wrapStreamFn: (ctx) => {
          if (!ctx.streamFn) return undefined;
          const inner = ctx.streamFn;
          return async (params) => {
            params.headers = {
              ...params.headers,
              "X-Acme-Version": "2",
            };
            return inner(params);
          };
        },
        ```
      </Tab>
      <Tab title="Native transportidentiteit">
        Voor providers die native aanvraag-/sessieheaders of metadata nodig hebben op
        generieke HTTP- of WebSocket-transporten:

        ```typescript
        resolveTransportTurnState: (ctx) => ({
          headers: {
            "x-request-id": ctx.turnId,
          },
          metadata: {
            session_id: ctx.sessionId ?? "",
            turn_id: ctx.turnId,
          },
        }),
        resolveWebSocketSessionPolicy: (ctx) => ({
          headers: {
            "x-session-id": ctx.sessionId ?? "",
          },
          degradeCooldownMs: 60_000,
        }),
        ```
      </Tab>
      <Tab title="Gebruik en facturering">
        Voor providers die gebruiks-/factureringsgegevens beschikbaar stellen:

        ```typescript
        resolveUsageAuth: async (ctx) => {
          const auth = await ctx.resolveOAuthToken();
          return auth ? { token: auth.token } : null;
        },
        fetchUsageSnapshot: async (ctx) => {
          return await fetchAcmeUsage(ctx.token, ctx.timeoutMs);
        },
        ```

        `resolveUsageAuth` heeft drie uitkomsten. Retourneer
        `{ token, accountId?, subscriptionType?, rateLimitTier? }` wanneer de
        provider een gebruiks-/factureringsreferentie heeft (de optionele velden brengen
        niet-geheime abonnementsmetadata van het opgeloste profiel over naar
        `fetchUsageSnapshot`). Retourneer
        `{ handled: true }` alleen wanneer de provider de gebruiksauthenticatie definitief heeft afgehandeld,
        maar geen bruikbaar gebruikstoken heeft, en OpenClaw de generieke
        terugval op API-sleutel/OAuth moet overslaan. Retourneer `null` of `undefined` wanneer de provider het
        verzoek niet heeft afgehandeld en OpenClaw moet doorgaan met de generieke terugval.

        Declareer de provider-id in `contracts.usageProviders`. Wanneer dat manifestcontract
        en **beide** hooks aanwezig zijn, neemt OpenClaw de provider automatisch op
        in de gebruiksverzameling zonder niet-gerelateerde providerplugins te laden.
        Er is geen update van de allowlist in de kern vereist.
        `fetchUsageSnapshot` retourneert de gedeelde providerneutrale vorm:

        - `plan`: door de provider gerapporteerd abonnement of sleutellabel
        - `windows`: opnieuw instelbare quotumvensters als gebruikte percentages
        - `billing`: getypeerde `balance`-, `spend`- of `budget`-items; `unit` kan
          een ISO-valuta of een providereenheid zoals `credits` zijn
        - `summary`: compacte providerspecifieke context die niet in deze
          gestructureerde velden past

        Houd de valutabetekenis exact. Een providertegoed is geen USD, tenzij het
        upstreamcontract dat vermeldt. Een plugin die alleen
        `fetchUsageSnapshot` implementeert, blijft beschikbaar voor expliciete/synthetische aanroepers, maar
        wordt niet automatisch gedetecteerd, omdat OpenClaw de gebruiksreferentie ervan niet kan oplossen.
      </Tab>
    </Tabs>

    <Accordion title="Algemene providerhooks">
      OpenClaw roept hooks voor model-/providerplugins ongeveer in deze volgorde aan.
      De meeste providers gebruiken er slechts 2-3. Dit is niet het volledige `ProviderPlugin`-
      contract; zie [Internals: hooks voor de providerruntime
      ](/nl/plugins/architecture-internals#provider-runtime-hooks) voor de
      volledige, momenteel actuele lijst met hooks en opmerkingen over terugval.
      Providervelden die uitsluitend voor compatibiliteit dienen en die OpenClaw niet meer aanroept, zoals
      `ProviderPlugin.capabilities` en `suppressBuiltInModel`, worden hier niet
      vermeld.

      | Hook | Wanneer te gebruiken |
      | --- | --- |
      | `catalog` | Modelcatalogus of standaardwaarden voor de basis-URL |
      | `applyConfigDefaults` | Provider-eigen globale standaardwaarden tijdens het materialiseren van de configuratie |
      | `normalizeModelId` | Opschoning van aliassen voor verouderde/previewmodel-id's vóór het opzoeken |
      | `normalizeTransport` | Opschoning van providerfamilie-`api` / `baseUrl` vóór generieke modelsamenstelling |
      | `normalizeConfig` | `models.providers.<id>`-configuratie normaliseren |
      | `applyNativeStreamingUsageCompat` | Native compatibiliteitsherschrijvingen voor streaminggebruik bij configuratieproviders |
      | `resolveConfigApiKey` | Provider-eigen authenticatieoplossing via omgevingsmarkeringen |
      | `resolveSyntheticAuth` | Synthetische authenticatie voor lokale/zelfgehoste of configuratiegestuurde omgevingen |
      | `resolveExternalAuthProfiles` | Provider-eigen externe authenticatieprofielen over elkaar leggen voor door CLI/apps beheerde referenties |
      | `shouldDeferSyntheticProfileAuth` | Synthetische tijdelijke aanduidingen voor opgeslagen profielen onder authenticatie via omgeving/configuratie verlagen |
      | `resolveDynamicModel` | Willekeurige upstreammodel-id's accepteren |
      | `prepareDynamicModel` | Asynchroon metadata ophalen vóór het oplossen |
      | `normalizeResolvedModel` | Transportherschrijvingen vóór de runner |
      | `normalizeToolSchemas` | Provider-eigen opschoning van toolschema's vóór registratie |
      | `inspectToolSchemas` | Provider-eigen diagnostiek voor toolschema's |
      | `resolveReasoningOutputMode` | Contract voor getagde versus native reasoning-uitvoer |
      | `prepareExtraParams` | Standaardaanvraagparameters |
      | `createStreamFn` | Volledig aangepast StreamFn-transport |
      | `wrapStreamFn` | Aangepaste wrappers voor headers/body in het normale streampad |
      | `resolveTransportTurnState` | Native headers/metadata per beurt |
      | `resolveWebSocketSessionPolicy` | Native WS-sessieheaders/afkoelperiode |
      | `formatApiKey` | Aangepaste runtimetokenvorm |
      | `refreshOAuth` | Aangepaste OAuth-vernieuwing |
      | `buildAuthDoctorHint` | Richtlijnen voor authenticatieherstel |
      | `matchesContextOverflowError` | Provider-eigen detectie van overschrijdingen |
      | `classifyFailoverReason` | Provider-eigen classificatie van snelheidslimieten/overbelasting |
      | `isCacheTtlEligible` | TTL-beperking voor de promptcache |
      | `buildMissingAuthMessage` | Aangepaste hint voor ontbrekende authenticatie |
      | `augmentModelCatalog` | Synthetische rijen voor voorwaartse compatibiliteit (verouderd - geef de voorkeur aan `registerModelCatalogProvider`) |
      | `resolveThinkingProfile` | Modelspecifieke `/think`-optieset |
      | `isBinaryThinking` | Compatibiliteit voor binaire thinking aan/uit (verouderd - geef de voorkeur aan `resolveThinkingProfile`) |
      | `supportsXHighThinking` | Compatibiliteit voor ondersteuning van `xhigh`-reasoning (verouderd - geef de voorkeur aan `resolveThinkingProfile`) |
      | `resolveDefaultThinkingLevel` | Compatibiliteit voor standaard `/think`-beleid (verouderd - geef de voorkeur aan `resolveThinkingProfile`) |
      | `isModernModelRef` | Modelmatching voor live-/smoketests |
      | `prepareRuntimeAuth` | Tokenuitwisseling vóór inferentie |
      | `resolveUsageAuth` | Aangepaste parsing van gebruiksreferenties |
      | `fetchUsageSnapshot` | Aangepast gebruikseindpunt |
      | `createEmbeddingProvider` | Provider-eigen embeddingadapter voor geheugen/zoeken |
      | `buildReplayPolicy` | Aangepast beleid voor transcriptreplay/Compaction |
      | `sanitizeReplayHistory` | Providerspecifieke replayherschrijvingen na generieke opschoning |
      | `validateReplayTurns` | Strikte validatie van replaybeurten vóór de ingesloten runner |
      | `onModelSelected` | Callback na selectie (bijv. telemetrie) |

      Opmerkingen over runtimeterugval:

      - `normalizeConfig` bepaalt per provider-id één verantwoordelijke plugin (eerst gebundelde providers, daarna de overeenkomende runtimeplugin) en roept alleen die hook aan - er wordt niet in andere providers gezocht. De eigen `normalizeConfig`-hook van Google normaliseert de configuratie-items `google` / `google-vertex` / `google-antigravity`; dit is geen afzonderlijke core-fallback.
      - `resolveConfigApiKey` gebruikt de providerhook wanneer die beschikbaar is. Amazon Bedrock behoudt de resolutie van AWS-omgevingsmarkeringen in zijn providerplugin; runtime-authenticatie zelf gebruikt nog steeds de standaardketen van de AWS SDK wanneer deze is geconfigureerd met `auth: "aws-sdk"`.
      - `resolveThinkingProfile(ctx)` ontvangt de geselecteerde `provider`, `modelId`, de optionele samengevoegde catalogushint `reasoning` en de optionele samengevoegde modelfeiten van `compat`. Gebruik `compat` alleen om de denkinterface/het denkprofiel van de provider te selecteren.
      - `resolveSystemPromptContribution` laat een provider cachebewuste richtlijnen voor de systeemprompt injecteren voor een modelfamilie. Geef hieraan de voorkeur boven de verouderde pluginbrede hook `before_prompt_build` wanneer het gedrag bij één provider/modelfamilie hoort en de stabiele/dynamische cachesplitsing moet behouden.

    </Accordion>

  </Step>

  <Step title="Extra mogelijkheden toevoegen (optioneel)">
    ### Stap 5: Extra mogelijkheden toevoegen

    Een providerplugin kan naast tekstinferentie embeddings, spraak, realtime transcriptie,
    realtime spraak, mediabegrip, afbeeldingsgeneratie, videogeneratie,
    webophaling en webzoekopdrachten registreren. OpenClaw classificeert dit als een
    plugin met **hybride mogelijkheden** - het aanbevolen patroon voor bedrijfsplugins
    (één plugin per leverancier). Zie
    [Intern: Eigendom van mogelijkheden](/nl/plugins/architecture#capability-ownership-model).

    Registreer elke mogelijkheid binnen `register(api)` naast je bestaande
    aanroep van `api.registerProvider(...)`. Kies alleen de tabbladen die je nodig hebt:

    <Tabs>
      <Tab title="Spraak (TTS)">
        ```typescript
        import {
          assertOkOrThrowProviderError,
          postJsonRequest,
        } from "openclaw/plugin-sdk/provider-http";

        api.registerSpeechProvider({
          id: "acme-ai",
          label: "Acme Speech",
          defaultTimeoutMs: 120_000,
          isConfigured: ({ config }) => Boolean(config.messages?.tts),
          synthesize: async (req) => {
            const { response, release } = await postJsonRequest({
              url: "https://api.example.com/v1/speech",
              headers: new Headers({ "Content-Type": "application/json" }),
              body: { text: req.text },
              timeoutMs: req.timeoutMs,
              fetchFn: fetch,
              auditContext: "acme speech",
            });
            try {
              await assertOkOrThrowProviderError(response, "Acme Speech API error");
              return {
                audioBuffer: Buffer.from(await response.arrayBuffer()),
                outputFormat: "mp3",
                fileExtension: ".mp3",
                voiceCompatible: false,
              };
            } finally {
              await release();
            }
          },
        });
        ```

        Gebruik `assertOkOrThrowProviderError(...)` voor HTTP-fouten van providers, zodat
        plugins begrensde lezingen van foutteksten, verwerking van JSON-fouten en
        achtervoegsels met aanvraag-id's delen.
      </Tab>
      <Tab title="Realtime transcriptie">
        Geef de voorkeur aan `createRealtimeTranscriptionWebSocketSession(...)` - de gedeelde
        helper verwerkt proxyvastlegging, exponentiële vertraging bij opnieuw verbinden, wegschrijven bij sluiten, gereedheids-
        handshakes, wachtrijen voor audio en diagnostiek van sluitgebeurtenissen. Je plugin
        wijst alleen upstreamgebeurtenissen toe.

        ```typescript
        api.registerRealtimeTranscriptionProvider({
          id: "acme-ai",
          label: "Acme Realtime Transcription",
          isConfigured: () => true,
          createSession: (req) => {
            const apiKey = String(req.providerConfig.apiKey ?? "");
            return createRealtimeTranscriptionWebSocketSession({
              providerId: "acme-ai",
              callbacks: req,
              url: "wss://api.example.com/v1/realtime-transcription",
              headers: { Authorization: `Bearer ${apiKey}` },
              onMessage: (event, transport) => {
                if (event.type === "session.created") {
                  transport.sendJson({ type: "session.update" });
                  transport.markReady();
                  return;
                }
                if (event.type === "transcript.final") {
                  req.onTranscript?.(event.text);
                }
              },
              sendAudio: (audio, transport) => {
                transport.sendJson({
                  type: "audio.append",
                  audio: audio.toString("base64"),
                });
              },
              onClose: (transport) => {
                transport.sendJson({ type: "audio.end" });
              },
            });
          },
        });
        ```

        Batch-STT-providers die multipart-audio via POST versturen, moeten
        `buildAudioTranscriptionFormData(...)` uit
        `openclaw/plugin-sdk/provider-http` gebruiken. De helper normaliseert bestandsnamen voor uploads,
        waaronder AAC-uploads die een bestandsnaam in M4A-stijl nodig hebben voor
        compatibele transcriptie-API's.
      </Tab>
      <Tab title="Realtime spraak">
        ```typescript
        api.registerRealtimeVoiceProvider({
          id: "acme-ai",
          label: "Acme Realtime Voice",
          capabilities: {
            transports: ["gateway-relay"],
            inputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            outputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            supportsBargeIn: true,
            handlesInputAudioBargeIn: true,
            supportsToolCalls: true,
          },
          isConfigured: ({ providerConfig }) => Boolean(providerConfig.apiKey),
          createBridge: (req) => ({
            // Stel dit alleen in als de provider meerdere toolantwoorden voor
            // één aanroep accepteert, bijvoorbeeld een onmiddellijk antwoord "bezig"
            // gevolgd door het uiteindelijke resultaat.
            supportsToolResultContinuation: false,
            connect: async () => {},
            sendAudio: () => {},
            setMediaTimestamp: () => {},
            handleBargeIn: () => {},
            submitToolResult: () => {},
            acknowledgeMark: () => {},
            close: () => {},
            isConnected: () => true,
          }),
        });
        ```

        Declareer `capabilities`, zodat `talk.catalog` geldige modi,
        transporten, audioformaten en functievlaggen beschikbaar kan stellen aan browser- en native Talk-
        clients. Implementeer `handleBargeIn` wanneer een transport kan detecteren dat een
        persoon het afspelen door de assistent onderbreekt en de provider het
        inkorten of wissen van de actieve audioreactie ondersteunt.
        `submitToolResult` kan `void` retourneren voor synchrone indiening, of een
        `Promise<void>` voor een asynchrone voltooiingsgrens die de provider-
        bridge beschikbaar kan stellen. Gateway-relaysessies wachten op die promise voordat
        ze een definitief resultaat bevestigen of de gekoppelde uitvoering wissen; wijs deze af wanneer
        de indiening mislukt.
        Stel `supportsToolResultSuppression: false` in wanneer de provider
        `options.suppressResponse` niet kan naleven. OpenClaw voorkomt dan onderdrukking voor
        interne resultaten van gedwongen raadpleging en annulering, en wijst directe
        aanvragen voor onderdrukte resultaten af in plaats van stilzwijgend een reactie te starten.
        Consumenten van `createRealtimeVoiceBridgeSession` kunnen eveneens een
        promise retourneren vanuit `onToolCall`; synchrone throws en afwijzingen worden doorgestuurd
        naar de callback `onError` van de sessie.
        Stel `handlesInputAudioBargeIn` alleen in wanneer de VAD van de provider een
        onderbreking bevestigt door `onClearAudio("barge-in")` aan te roepen. Providers die
        de vlag weglaten, gebruiken de lokale fallbackdetectie van OpenClaw voor invoeraudio.
      </Tab>
      <Tab title="Mediabegrip">
        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "acme-ai",
          capabilities: ["image", "audio"],
          describeImage: async (req) => ({ text: "Een foto van..." }),
          transcribeAudio: async (req) => ({ text: "Transcriptie..." }),
        });
        ```

        Lokale of zelfgehoste mediaproviders die bewust geen
        aanmeldgegevens vereisen, kunnen `resolveAuth` beschikbaar stellen en `kind: "none"` retourneren.
        OpenClaw handhaaft nog steeds de normale authenticatiecontrole voor providers die zich niet
        expliciet aanmelden. Bestaande providers kunnen `req.apiKey` blijven lezen;
        nieuwe providers moeten de voorkeur geven aan `req.auth`.

        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "local-audio",
          capabilities: ["audio"],
          resolveAuth: () => ({
            kind: "none",
            source: "local-audio-plugin zonder authenticatie",
          }),
          transcribeAudio: async (req) => ({ text: "Transcriptie..." }),
        });
        ```
      </Tab>
      <Tab title="Embeddings">
        ```typescript
        api.registerEmbeddingProvider({
          id: "acme-ai",
          defaultModel: "acme-embed",
          transport: "remote",
          authProviderId: "acme-ai",
          create: async ({ model }) => ({
            provider: {
              id: "acme-ai",
              model,
              dimensions: 1536,
              embed: async (input) => {
                const text = typeof input === "string" ? input : input.text;
                return fetchAcmeEmbedding(text);
              },
              embedBatch: async (inputs) =>
                Promise.all(
                  inputs.map((input) =>
                    fetchAcmeEmbedding(typeof input === "string" ? input : input.text),
                  ),
                ),
            },
          }),
        });
        ```

        Declareer dezelfde id in `contracts.embeddingProviders`. Dit is het
        algemene embeddingcontract voor herbruikbare vectorgeneratie, waaronder
        zoeken in het geheugen. `registerMemoryEmbeddingProvider(...)` is verouderde
        compatibiliteit voor bestaande geheugenspecifieke adapters.
      </Tab>
      <Tab title="Afbeeldings- en videogeneratie">
        Afbeeldings- en videomogelijkheden gebruiken een **modusbewuste** structuur. Afbeeldings-
        providers declareren verplichte mogelijkhedenblokken voor `generate` en `edit`;
        videoproviders declareren `generate`, `imageToVideo` en
        `videoToVideo`. Platte samengevoegde velden zoals `maxInputImages` /
        `maxInputVideos` / `maxDurationSeconds` volstaan niet om ondersteuning voor
        transformatiemodus of uitgeschakelde modi duidelijk aan te geven. Muziekgeneratie
        volgt hetzelfde patroon van `generate` / `edit`.

        ```typescript
        api.registerImageGenerationProvider({
          id: "acme-ai",
          label: "Acme-afbeeldingen",
          capabilities: {
            generate: { maxCount: 4, supportsSize: true },
            edit: { enabled: false },
          },
          generateImage: async (req) => ({ images: [] }),
        });

        api.registerVideoGenerationProvider({
          id: "acme-ai",
          label: "Acme-video",
          defaultTimeoutMs: 600_000,
          models: ["acme-video", "acme-image-video"],
          capabilities: {
            generate: { maxVideos: 1, maxDurationSeconds: 10, supportsResolution: true },
            imageToVideo: {
              enabled: true,
              maxVideos: 1,
              maxInputImages: 1,
              maxInputImagesByModel: { "acme/reference-to-video": 9 },
              maxDurationSeconds: 5,
            },
            videoToVideo: { enabled: false },
          },
          catalogByModel: {
            "acme-image-video": {
              modes: ["imageToVideo"],
              capabilities: {
                imageToVideo: {
                  enabled: true,
                  maxVideos: 1,
                  maxInputImages: 1,
                  resolutions: ["480P", "720P", "1080P"],
                  supportsResolution: true,
                },
                videoToVideo: { enabled: false },
              },
            },
          },
          generateVideo: async (req) => ({ videos: [] }),
        });
        ```

        `capabilities` is vereist voor beide providertypen; `edit` en de
        videotransformatieblokken (`imageToVideo`, `videoToVideo`) vereisen altijd een
        expliciete `enabled`-vlag.

        Gebruik `catalogByModel` wanneer de statische modi of mogelijkheden van een vermeld model
        afwijken van de standaardwaarden van de provider. Deze metadata houdt
        `video_generate action=list` en modelcatalogi nauwkeurig zonder
        providercode aan te roepen. Het tijdens een aanvraag opzoeken en afdwingen van mogelijkheden
        hoort nog steeds thuis in `resolveModelCapabilities` en `generateVideo`; hergebruik
        waar mogelijk dezelfde mogelijkhedenconstante voor beide paden.
      </Tab>
      <Tab title="Webpagina's ophalen en doorzoeken">
        ```typescript
        api.registerWebFetchProvider({
          id: "acme-ai-fetch",
          label: "Acme Ophalen",
          hint: "Haal pagina's op via de renderingbackend van Acme.",
          envVars: ["ACME_FETCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/fetch",
          credentialPath: "plugins.entries.acme.config.webFetch.apiKey",
          getCredentialValue: (fetchConfig) => fetchConfig?.acme?.apiKey,
          setCredentialValue: (fetchConfigTarget, value) => {
            const acme = (fetchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "Haal een pagina op via Acme Ophalen.",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });

        api.registerWebSearchProvider({
          id: "acme-ai-search",
          label: "Acme Zoeken",
          hint: "Doorzoek het web via de zoekbackend van Acme.",
          envVars: ["ACME_SEARCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/search",
          credentialPath: "plugins.entries.acme.config.webSearch.apiKey",
          getCredentialValue: (searchConfig) => searchConfig?.acme?.apiKey,
          setCredentialValue: (searchConfigTarget, value) => {
            const acme = (searchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "Doorzoek het web via Acme Zoeken.",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });
        ```

        Beide providertypen gebruiken dezelfde structuur voor het koppelen van inloggegevens:
        `hint`, `envVars`, `placeholder`, `signupUrl`, `credentialPath`,
        `getCredentialValue`, `setCredentialValue` en `createTool` zijn allemaal
        vereist.
      </Tab>
    </Tabs>

  </Step>

  <Step title="Testen">
    ### Stap 6: Testen

    ```typescript src/provider.test.ts
    import { describe, it, expect } from "vitest";
    // Exporteer je providerconfiguratieobject vanuit index.ts of een afzonderlijk bestand
    import { acmeProvider } from "./provider.js";

    describe("acme-ai-provider", () => {
      it("lost dynamische modellen op", () => {
        const model = acmeProvider.resolveDynamicModel!({
          modelId: "acme-beta-v3",
        } as any);
        expect(model.id).toBe("acme-beta-v3");
        expect(model.provider).toBe("acme-ai");
      });

      it("retourneert de catalogus wanneer een sleutel beschikbaar is", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: "test-key" }),
        } as any);
        expect(result?.provider?.models).toHaveLength(2);
      });

      it("retourneert een null-catalogus wanneer er geen sleutel is", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: undefined }),
        } as any);
        expect(result).toBeNull();
      });
    });
    ```

  </Step>
</Steps>

## Publiceren naar ClawHub

Providerplugins worden op dezelfde manier gepubliceerd als elke andere externe codeplugin:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

`clawhub skill publish <path>` is een andere opdracht voor het publiceren van een Skills-map,
niet van een pluginpakket — gebruik deze hier niet.

## Bestandsstructuur

```
<bundled-plugin-root>/acme-ai/
├── package.json              # openclaw.providers-metadata
├── openclaw.plugin.json      # Manifest met authenticatiemetadata voor de provider
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # Tests
    └── usage.ts              # Gebruikseindpunt (optioneel)
```

## Referentie voor catalogusvolgorde

`catalog.order` bepaalt wanneer je catalogus wordt samengevoegd ten opzichte van ingebouwde
providers:

| Volgorde     | Wanneer          | Gebruikssituatie                                        |
| --------- | ------------- | ----------------------------------------------- |
| `simple`  | Eerste doorgang    | Providers met alleen een API-sleutel                         |
| `profile` | Na eenvoudig  | Providers die afhankelijk zijn van authenticatieprofielen                |
| `paired`  | Na profiel | Meerdere gerelateerde vermeldingen samenstellen             |
| `late`    | Laatste doorgang     | Bestaande providers overschrijven (wint bij een conflict) |

## Volgende stappen

- [Kanaalplugins](/nl/plugins/sdk-channel-plugins) - als je plugin ook een kanaal aanbiedt
- [SDK-runtime](/nl/plugins/sdk-runtime) - `api.runtime`-helpers (TTS, zoeken, subagent)
- [SDK-overzicht](/nl/plugins/sdk-overview) - volledige referentie voor import via subpaden
- [Interne werking van plugins](/nl/plugins/architecture-internals#provider-runtime-hooks) - details over hooks en gebundelde voorbeelden

## Gerelateerd

- [Plugin-SDK instellen](/nl/plugins/sdk-setup)
- [Plugins bouwen](/nl/plugins/building-plugins)
- [Kanaalplugins bouwen](/nl/plugins/sdk-channel-plugins)
