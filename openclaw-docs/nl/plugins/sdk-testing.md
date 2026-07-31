---
read_when:
    - Je schrijft tests voor een plugin
    - Je hebt testhulpprogramma's uit de Plugin-SDK nodig
    - Je wilt contracttests voor gebundelde plugins begrijpen
sidebarTitle: Testing
summary: Testhulpmiddelen en -patronen voor OpenClaw-plugins
title: Plugin testen
x-i18n:
    generated_at: "2026-07-27T06:29:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9c6c050826dae3cd2c794d50b2dd95e20e6533d838161cce037742ee5fdf7e0e
    source_path: plugins/sdk-testing.md
    workflow: 16
---

Naslaginformatie voor testhulpprogramma's, patronen en lint-handhaving voor OpenClaw-
plugins.

<Tip>
  **Op zoek naar testvoorbeelden?** De instructiegidsen bevatten uitgewerkte testvoorbeelden:
  [Tests voor kanaalplugins](/nl/plugins/sdk-channel-plugins#step-6-test) en
  [Tests voor providerplugins](/nl/plugins/sdk-provider-plugins#step-6-test).
</Tip>

## Testhulpprogramma's

Deze subpaden zijn lokale broningangen van de repository voor de eigen gebundelde
plugintests van OpenClaw. Het zijn geen gepubliceerde `package.json`-exports voor plugins
van derden en ze kunnen Vitest of andere testafhankelijkheden importeren die alleen in de repository beschikbaar zijn.

```typescript
import {
  shouldAckReaction,
  removeAckReactionAfterReply,
} from "openclaw/plugin-sdk/channel-feedback";
import { installCommonResolveTargetErrorCases } from "openclaw/plugin-sdk/channel-target-testing";
import { AUTH_PROFILE_RUNTIME_CONTRACT } from "openclaw/plugin-sdk/agent-runtime-test-contracts";
import { createTestPluginApi } from "openclaw/plugin-sdk/plugin-test-api";
import { expectChannelInboundContextContract } from "openclaw/plugin-sdk/channel-contract-testing";
import { createStartAccountContext } from "openclaw/plugin-sdk/channel-test-helpers";
import { describePluginRegistrationContract } from "openclaw/plugin-sdk/plugin-test-contracts";
import { registerSingleProviderPlugin } from "openclaw/plugin-sdk/plugin-test-runtime";
import { describeOpenAIProviderRuntimeContract } from "openclaw/plugin-sdk/provider-test-contracts";
import { getProviderHttpMocks } from "openclaw/plugin-sdk/provider-http-test-mocks";
import { withEnv, withFetchPreconnect, withServer } from "openclaw/plugin-sdk/test-env";
import { isLiveTestEnabled } from "openclaw/plugin-sdk/test-live";
import { createRequestCaptureJsonFetch } from "openclaw/plugin-sdk/test-media-understanding";
import {
  bundledPluginRoot,
  createCliRuntimeCapture,
  typedCases,
} from "openclaw/plugin-sdk/test-fixtures";
import { mockNodeBuiltinModule } from "openclaw/plugin-sdk/test-node-mocks";
```

Gebruik deze gerichte subpaden voor tests van gebundelde plugins. De voormalige
`openclaw/plugin-sdk/testing`-barrel was lokaal in de repository, uitgesloten van uitgebrachte
pakketten en is verwijderd. De voormalige `openclaw/plugin-sdk/test-utils`-
alias is tegelijk verwijderd. `pnpm run lint:plugins:no-extension-test-core-imports`
(`scripts/check-no-extension-test-core-imports.ts`) houdt extensietests op
de bovenstaande gerichte testsubpaden.

### Beschikbare exports

| Export                                               | Doel                                                                                                                                  |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `createTestPluginApi`                                | Bouw een minimale mock van de Plugin-API voor unittests van directe registratie. Importeer uit `plugin-sdk/plugin-test-api`                             |
| `AUTH_PROFILE_RUNTIME_CONTRACT`                      | Gedeelde contractfixture voor authenticatieprofielen voor systeemeigen adapters van de agentruntime. Importeer uit `plugin-sdk/agent-runtime-test-contracts`            |
| `DELIVERY_NO_REPLY_RUNTIME_CONTRACT`                 | Gedeelde contractfixture voor onderdrukking van bezorging voor systeemeigen adapters van de agentruntime. Importeer uit `plugin-sdk/agent-runtime-test-contracts`    |
| `OUTCOME_FALLBACK_RUNTIME_CONTRACT`                  | Gedeelde contractfixture voor fallbackclassificatie voor systeemeigen adapters van de agentruntime. Importeer uit `plugin-sdk/agent-runtime-test-contracts` |
| `createParameterFreeTool`                            | Bouw schemafixtures voor dynamische tools voor contracttests van systeemeigen runtimes. Importeer uit `plugin-sdk/agent-runtime-test-contracts`              |
| `expectChannelInboundContextContract`                | Controleer de vorm van de inkomende kanaalcontext. Importeer uit `plugin-sdk/channel-contract-testing`                                                  |
| `installChannelOutboundPayloadContractSuite`         | Installeer contractgevallen voor uitgaande kanaalpayloads. Importeer uit `plugin-sdk/channel-contract-testing`                                       |
| `createStartAccountContext`                          | Bouw contexten voor de levenscyclus van kanaalaccounts. Importeer uit `plugin-sdk/channel-test-helpers`                                                  |
| `installChannelActionsContractSuite`                 | Installeer generieke contractgevallen voor kanaalberichtacties. Importeer uit `plugin-sdk/channel-test-helpers`                                     |
| `installChannelSetupContractSuite`                   | Installeer generieke contractgevallen voor kanaalconfiguratie. Importeer uit `plugin-sdk/channel-test-helpers`                                              |
| `installChannelStatusContractSuite`                  | Installeer generieke contractgevallen voor kanaalstatus. Importeer uit `plugin-sdk/channel-test-helpers`                                             |
| `expectDirectoryIds`                                 | Controleer kanaalmap-id's aan de hand van een functie die mappen opsomt. Importeer uit `plugin-sdk/channel-test-helpers`                               |
| `assertBundledChannelEntries`                        | Controleer of ingangspunten van gebundelde kanalen het verwachte openbare contract beschikbaar stellen. Importeer uit `plugin-sdk/channel-test-helpers`                    |
| `formatEnvelopeTimestamp`                            | Formatteer deterministische tijdstempels voor enveloppen. Importeer uit `plugin-sdk/channel-test-helpers`                                                  |
| `expectPairingReplyText`                             | Controleer de antwoordtekst voor kanaalkoppeling en extraheer de code ervan. Importeer uit `plugin-sdk/channel-test-helpers`                                    |
| `describePluginRegistrationContract`                 | Installeer contractcontroles voor Plugin-registratie. Importeer uit `plugin-sdk/plugin-test-contracts`                                              |
| `registerSingleProviderPlugin`                       | Registreer één provider-Plugin in rooktests voor de loader. Importeer uit `plugin-sdk/plugin-test-runtime`                                         |
| `registerProviderPlugin`                             | Leg alle providersoorten van één Plugin vast. Importeer uit `plugin-sdk/plugin-test-runtime`                                                 |
| `registerProviderPlugins`                            | Leg providerregistraties van meerdere plugins vast. Importeer uit `plugin-sdk/plugin-test-runtime`                                     |
| `requireRegisteredProvider`                          | Controleer of een verzameling providers een id bevat. Importeer uit `plugin-sdk/plugin-test-runtime`                                           |
| `createRuntimeEnv`                                   | Bouw een gesimuleerde runtimeomgeving voor de CLI/Plugin. Importeer uit `plugin-sdk/plugin-test-runtime`                                              |
| `createPluginRuntimeMock`                            | Bouw een gesimuleerd Plugin-runtimeoppervlak. Importeer uit `plugin-sdk/plugin-test-runtime`                                                      |
| `createPluginSetupWizardStatus`                      | Bouw helpers voor de configuratiestatus van kanaalplugins. Importeer uit `plugin-sdk/plugin-test-runtime`                                             |
| `createTestWizardPrompter`                           | Bouw een gesimuleerde promptfunctie voor de configuratiewizard. Importeer uit `plugin-sdk/plugin-test-runtime`                                                       |
| `createRuntimeTaskFlow`                              | Maak geïsoleerde TaskFlow-status voor de runtime. Importeer uit `plugin-sdk/plugin-test-runtime`                                                    |
| `runProviderCatalog`                                 | Voer een providercatalogushaak uit met testafhankelijkheden. Importeer uit `plugin-sdk/plugin-test-runtime`                                     |
| `resolveProviderWizardOptions`                       | Bepaal de keuzes van de providerconfiguratiewizard in contracttests. Importeer uit `plugin-sdk/plugin-test-runtime`                                    |
| `resolveProviderModelPickerEntries`                  | Bepaal de items van de provider-modelkiezer in contracttests. Importeer uit `plugin-sdk/plugin-test-runtime`                                    |
| `buildProviderPluginMethodChoice`                    | Bouw keuze-id's voor de providerwizard voor controles. Importeer uit `plugin-sdk/plugin-test-runtime`                                            |
| `setProviderWizardProvidersResolverForTest`          | Injecteer providers voor de providerwizard in geïsoleerde tests. Importeer uit `plugin-sdk/plugin-test-runtime`                                        |
| `describeOpenAIProviderRuntimeContract`              | Installeer runtimecontractcontroles voor providerfamilies. Importeer uit `plugin-sdk/provider-test-contracts`                                        |
| `expectPassthroughReplayPolicy`                      | Controleer of beleid voor herhaling door providers wordt doorgegeven via tools en metadata die eigendom zijn van de provider. Importeer uit `plugin-sdk/provider-test-contracts`         |
| `runRealtimeSttLiveTest`                             | Voer een live realtime test van een STT-provider uit met gedeelde audiofixtures. Importeer uit `plugin-sdk/provider-test-contracts`                       |
| `normalizeTranscriptForMatch`                        | Normaliseer live transcriptuitvoer vóór fuzzy controles. Importeer uit `plugin-sdk/provider-test-contracts`                               |
| `expectExplicitVideoGenerationCapabilities`          | Controleer of videoproviders expliciete mogelijkheden voor generatiemodi declareren. Importeer uit `plugin-sdk/provider-test-contracts`                   |
| `expectExplicitMusicGenerationCapabilities`          | Controleer of muziekproviders expliciete mogelijkheden voor genereren/bewerken declareren. Importeer uit `plugin-sdk/provider-test-contracts`                   |
| `mockSuccessfulDashscopeVideoTask`                   | Installeer een geslaagd antwoord op een DashScope-compatibele videotaak. Importeer uit `plugin-sdk/provider-test-contracts`                          |
| `getProviderHttpMocks`                               | Verkrijg toegang tot opt-in Vitest-mocks voor HTTP/authenticatie van providers. Importeer uit `plugin-sdk/provider-http-test-mocks`                                         |
| `installProviderHttpMockCleanup`                     | Stel mocks voor HTTP/authenticatie van providers na elke test opnieuw in. Importeer uit `plugin-sdk/provider-http-test-mocks`                                        |
| `installCommonResolveTargetErrorCases`               | Gedeelde testgevallen voor foutafhandeling bij het bepalen van doelen. Importeer uit `plugin-sdk/channel-target-testing`                                  |
| `shouldAckReaction`                                  | Controleer of een kanaal een bevestigingsreactie moet toevoegen. Importeer uit `plugin-sdk/channel-feedback`                                            |
| `removeAckReactionAfterReply`                        | Verwijder de bevestigingsreactie nadat het antwoord is bezorgd. Importeer uit `plugin-sdk/channel-feedback`                                                      |
| `createTestRegistry`                                 | Bouw een registerfixture voor kanaalplugins. Importeer uit `plugin-sdk/plugin-test-runtime` of `plugin-sdk/channel-test-helpers`               |
| `createEmptyPluginRegistry`                          | Bouw een lege Plugin-registerfixture. Importeer uit `plugin-sdk/plugin-test-runtime` of `plugin-sdk/channel-test-helpers`                |
| `setActivePluginRegistry`                            | Installeer een registerfixture voor Plugin-runtimetests. Importeer uit `plugin-sdk/plugin-test-runtime` of `plugin-sdk/channel-test-helpers`   |
| `createRequestCaptureJsonFetch`                      | Leg JSON-fetchverzoeken vast in tests van mediahelpers. Importeer uit `plugin-sdk/test-media-understanding`                                     |
| `isLiveTestEnabled`                                  | Scherm opt-in live providertests af. Importeer uit `plugin-sdk/test-live`                                                                      |
| `collectProviderApiKeys`                             | Zoek aanmeldgegevens voor live providertests. Importeer uit `plugin-sdk/test-live-auth`                                                    |
| `parseProviderModelMap`                              | Parseer overschrijvingen van modellen voor live muziek-/videotests. Importeer uit `plugin-sdk/test-media-generation`                                              |
| `withServer`                                         | Voer tests uit met een tijdelijke lokale HTTP-server. Importeer uit `plugin-sdk/test-env`                                                      |
| `createMockIncomingRequest`                          | Bouw een minimaal object voor inkomende HTTP-verzoeken. Importeer uit `plugin-sdk/test-env`                                                          |
| `withFetchPreconnect`                                | Voer fetchtests uit met geïnstalleerde preconnect-haken. Importeer uit `plugin-sdk/test-env`                                                       |
| `withEnv` / `withEnvAsync`                           | Pas omgevingsvariabelen tijdelijk aan. Importeer uit `plugin-sdk/test-env`                                                               |
| `createTempHomeEnv` / `withTempHome` / `withTempDir` | Maak geïsoleerde testfixtures voor het bestandssysteem. Importeer uit `plugin-sdk/test-env`                                                              |
| `createMockServerResponse`                           | Maak een minimale mock voor HTTP-serverantwoorden. Importeer uit `plugin-sdk/test-env`                                                            |
| `createProviderUsageFetch`                           | Bouw fetchfixtures voor providergebruik. Importeer uit `plugin-sdk/test-env`                                                                   |
| `useFrozenTime` / `useRealTime`                      | Bevries en herstel timers voor tijdgevoelige tests. Importeer uit `plugin-sdk/test-env`                                                    |
| `createCliRuntimeCapture`                            | Leg CLI-runtimeuitvoer vast in tests. Importeer uit `plugin-sdk/test-fixtures`                                                              |
| `importFreshModule`                                  | Importeer een ESM-module met een nieuw querytoken om de modulecache te omzeilen. Importeer uit `plugin-sdk/test-fixtures`                             |
| `bundledPluginRoot` / `bundledPluginFile`            | Bepaal fixturepaden naar de broncode of dist van gebundelde plugins. Importeer uit `plugin-sdk/test-fixtures`                                              |
| `mockNodeBuiltinModule`                              | Installeer beperkte Vitest-mocks voor ingebouwde Node-modules. Importeer uit `plugin-sdk/test-node-mocks`                                                       |
| `createSandboxTestContext`                           | Bouw testcontexten voor de sandbox. Importeer uit `plugin-sdk/test-fixtures`                                                                      |
| `writeSkill`                                         | Schrijf Skills-fixtures. Importeer uit `plugin-sdk/test-fixtures`                                                                             |
| `makeAgentAssistantMessage`                          | Bouw berichtfixtures voor agenttranscripten. Importeer uit `plugin-sdk/test-fixtures`                                                          |
| `peekSystemEvents` / `resetSystemEventsForTest`      | Inspecteer en reset fixtures voor systeemgebeurtenissen. Importeer uit `plugin-sdk/test-fixtures`                                                          |
| `sanitizeTerminalText`                               | Saniteer terminaluitvoer voor controles. Importeer uit `plugin-sdk/test-fixtures`                                                          |
| `countLines` / `hasBalancedFences`                   | Controleer de vorm van de chunking-uitvoer. Importeer uit `plugin-sdk/test-fixtures`                                                                     |
| `typedCases`                                         | Behoud letterlijke typen voor tabelgestuurde tests. Importeer uit `plugin-sdk/test-fixtures`                                                    |

Gebundelde Plugin-contractsuites gebruiken deze SDK-testsubpaden ook voor
testhelpers voor alleen-tests-registers, manifesten, openbare artefacten en runtime-fixtures.
Suites die uitsluitend voor core zijn en afhankelijk zijn van de gebundelde OpenClaw-inventaris, blijven in plaats daarvan onder
`src/plugins/contracts`.

### Typen

Gerichte testsubpaden exporteren ook typen opnieuw die nuttig zijn in testbestanden:

```typescript
import type {
  ChannelAccountSnapshot,
  ChannelGatewayContext,
} from "openclaw/plugin-sdk/channel-contract";
import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
import type { MockFn, PluginRuntime, RuntimeEnv } from "openclaw/plugin-sdk/plugin-test-runtime";
```

## Oplossing van testdoelen

Gebruik `installCommonResolveTargetErrorCases` om standaardfoutgevallen toe te voegen voor
het oplossen van kanaaldoelen:

```typescript
import { describe } from "vitest";
import { installCommonResolveTargetErrorCases } from "openclaw/plugin-sdk/channel-target-testing";

describe("oplossing van my-channel-doelen", () => {
  installCommonResolveTargetErrorCases({
    resolveTarget: ({ to, mode, allowFrom }) => {
      // De logica voor het oplossen van doelen van je kanaal
      return myChannelResolveTarget({ to, mode, allowFrom });
    },
    implicitAllowFrom: ["user1", "user2"],
  });

  // Kanaalspecifieke testgevallen toevoegen
  it("moet @username-doelen oplossen", () => {
    // ...
  });
});
```

## Testpatronen

### Registratiecontracten testen

Unittests die een handgeschreven mock van `api` doorgeven aan `register(api)`, oefenen
de acceptatiecontroles van de OpenClaw-loader niet uit. Voeg ten minste één door de loader ondersteunde
rooktest toe voor elk registratieoppervlak waarvan je Plugin afhankelijk is, met name
hooks en exclusieve mogelijkheden zoals geheugen.

De echte loader laat de Plugin-registratie mislukken wanneer vereiste metadata ontbreekt of
een Plugin een capability-API aanroept waarvan deze niet de eigenaar is. Bijvoorbeeld:
`api.registerHook(...)` vereist een hooknaam en
`api.registerMemoryCapability(...)` vereist dat het Plugin-manifest of de geëxporteerde
entry `kind: "memory"` declareert.

### Toegang tot runtimeconfiguratie testen

Geef de voorkeur aan de gedeelde mock voor de Plugin-runtime uit
`openclaw/plugin-sdk/plugin-test-runtime`. De helpers voor runtimeconfiguratie modelleren de
huidige snapshot- en mutatie-API's.

### Een kanaal-Plugin met unittests testen

```typescript
import { describe, it, expect, vi } from "vitest";

describe("my-channel-Plugin", () => {
  it("moet account uit configuratie oplossen", () => {
    const cfg = {
      channels: {
        "my-channel": {
          token: "test-token",
          allowFrom: ["user1"],
        },
      },
    };

    const account = myPlugin.setup.resolveAccount(cfg, undefined);
    expect(account.token).toBe("test-token");
  });

  it("moet account inspecteren zonder geheimen te materialiseren", () => {
    const cfg = {
      channels: {
        "my-channel": { token: "test-token" },
      },
    };

    const inspection = myPlugin.setup.inspectAccount(cfg, undefined);
    expect(inspection.configured).toBe(true);
    expect(inspection.tokenStatus).toBe("available");
    // Geen tokenwaarde blootgesteld
    expect(inspection).not.toHaveProperty("token");
  });
});
```

### Een provider-Plugin met unittests testen

```typescript
import { describe, it, expect } from "vitest";

describe("my-provider-Plugin", () => {
  it("moet dynamische modellen oplossen", () => {
    const model = myProvider.resolveDynamicModel({
      modelId: "custom-model-v2",
      // ... context
    });

    expect(model.id).toBe("custom-model-v2");
    expect(model.provider).toBe("my-provider");
    expect(model.api).toBe("openai-completions");
  });

  it("moet catalogus retourneren wanneer een API-sleutel beschikbaar is", async () => {
    const result = await myProvider.catalog.run({
      resolveProviderApiKey: () => ({ apiKey: "test-key" }),
      // ... context
    });

    expect(result?.provider?.models).toHaveLength(2);
  });
});
```

### De Plugin-runtime mocken

Voor code die `createPluginRuntimeStore` gebruikt, mock je de runtime in tests:

```typescript
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

const store = createPluginRuntimeStore<PluginRuntime>({
  pluginId: "test-plugin",
  errorMessage: "testruntime niet ingesteld",
});

// In de testconfiguratie
const mockRuntime = {
  agent: {
    resolveAgentDir: vi.fn().mockReturnValue("/tmp/agent"),
    // ... andere mocks
  },
  config: {
    current: vi.fn(() => ({}) as const),
    mutateConfigFile: vi.fn(),
    replaceConfigFile: vi.fn(),
  },
  // ... andere naamruimten
} as unknown as PluginRuntime;

store.setRuntime(mockRuntime);

// Na de tests
store.clearRuntime();
```

### Testen met stubs per instantie

Geef de voorkeur aan stubs per instantie boven prototypemutatie:

```typescript
// Aanbevolen: stub per instantie
const client = new MyChannelClient();
client.sendMessage = vi.fn().mockResolvedValue({ id: "msg-1" });

// Vermijd: prototypemutatie
// MyChannelClient.prototype.sendMessage = vi.fn();
```

## Contracttests (Plugins in de repository)

Gebundelde Plugins hebben contracttests die het eigenaarschap van registraties verifiëren:

```bash
pnpm test src/plugins/contracts/
```

Deze tests controleren:

- Welke Plugins welke providers registreren
- Welke Plugins welke spraakproviders registreren
- Correctheid van de registratievorm
- Naleving van het runtimecontract

### Afgebakende tests uitvoeren

Voor een specifieke Plugin:

```bash
pnpm test <bundled-plugin-root>/my-channel/
```

Alleen voor contracttests:

```bash
pnpm test src/plugins/contracts/shape.contract.test.ts
pnpm test src/plugins/contracts/auth-choice.contract.test.ts
pnpm test src/plugins/contracts/runtime-seams.contract.test.ts
```

## Lint-handhaving (Plugins in de repository)

`scripts/run-additional-boundary-checks.mjs` voert in CI een reeks `lint:plugins:*`
controles op importgrenzen uit; elke controle kan ook zelfstandig lokaal worden uitgevoerd:

| Opdracht                                                        | Dwingt af                                                                                     |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `pnpm run lint:plugins:no-monolithic-plugin-sdk-entry-imports` | Gebundelde Plugins mogen de monolithische hoofdbarrel `openclaw/plugin-sdk` niet importeren.              |
| `pnpm run lint:plugins:no-extension-src-imports`               | Productie-extensiebestanden mogen de `src/**`-boom van de repository niet rechtstreeks importeren (`../../src/...`).  |
| `pnpm run lint:plugins:no-extension-test-core-imports`         | Extensietestbestanden mogen geen verwijderde SDK-testaliassen of andere testhelpers die uitsluitend voor core zijn importeren. |

Externe Plugins vallen niet onder deze lintregels, maar het volgen van dezelfde
patronen wordt aanbevolen.

## Testconfiguratie

OpenClaw gebruikt Vitest 4 met informatieve V8-dekkingsrapportage. Voor Plugin-tests:

```bash
# Alle tests uitvoeren
pnpm test

# Tests voor een specifieke Plugin uitvoeren
pnpm test <bundled-plugin-root>/my-channel/src/channel.test.ts

# Uitvoeren met een filter voor een specifieke testnaam
pnpm test <bundled-plugin-root>/my-channel/ -t "resolves account"

# Uitvoeren met dekking
pnpm test:coverage
```

Als lokale uitvoeringen geheugendruk veroorzaken:

```bash
OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test
```

## Gerelateerd

- [SDK-overzicht](/nl/plugins/sdk-overview) -- importconventies
- [SDK-kanaal-Plugins](/nl/plugins/sdk-channel-plugins) -- interface voor kanaal-Plugins
- [SDK-provider-Plugins](/nl/plugins/sdk-provider-plugins) -- hooks voor provider-Plugins
- [Plugins bouwen](/nl/plugins/building-plugins) -- introductiegids
