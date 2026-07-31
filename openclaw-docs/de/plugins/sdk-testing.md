---
read_when:
    - Sie schreiben Tests für ein Plugin
    - Sie benötigen Test-Dienstprogramme aus dem Plugin-SDK
    - Sie möchten Vertragstests für gebündelte Plugins verstehen
sidebarTitle: Testing
summary: Testwerkzeuge und -muster für OpenClaw-Plugins
title: Plugin-Tests
x-i18n:
    generated_at: "2026-07-26T19:10:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9c6c050826dae3cd2c794d50b2dd95e20e6533d838161cce037742ee5fdf7e0e
    source_path: plugins/sdk-testing.md
    workflow: 16
---

Referenz für Testhilfsprogramme, Muster und Lint-Durchsetzung für OpenClaw-
Plugins.

<Tip>
  **Suchen Sie nach Testbeispielen?** Die Anleitungen enthalten ausgearbeitete Testbeispiele:
  [Tests für Channel-Plugins](/de/plugins/sdk-channel-plugins#step-6-test) und
  [Tests für Provider-Plugins](/de/plugins/sdk-provider-plugins#step-6-test).
</Tip>

## Testhilfsprogramme

Diese Unterpfade sind repo-lokale Quell-Einstiegspunkte für die mitgelieferten
Plugin-Tests von OpenClaw. Sie sind keine veröffentlichten `package.json`-Exporte für Drittanbieter-
Plugins und können Vitest oder andere ausschließlich im Repository verfügbare Testabhängigkeiten importieren.

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

Verwenden Sie diese gezielten Unterpfade für Tests mitgelieferter Plugins. Das frühere
`openclaw/plugin-sdk/testing`-Barrel war repo-lokal, von ausgelieferten
Paketen ausgeschlossen und wurde entfernt. Der frühere Alias `openclaw/plugin-sdk/test-utils`
wurde zusammen damit entfernt. `pnpm run lint:plugins:no-extension-test-core-imports`
(`scripts/check-no-extension-test-core-imports.ts`) hält Erweiterungstests auf
den oben genannten gezielten Test-Unterpfaden.

### Verfügbare Exporte

| Export                                               | Zweck                                                                                                                                  |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `createTestPluginApi`                                | Einen minimalen Plugin-API-Mock für Unit-Tests der direkten Registrierung erstellen. Aus `plugin-sdk/plugin-test-api` importieren                             |
| `AUTH_PROFILE_RUNTIME_CONTRACT`                      | Gemeinsame Vertrags-Fixture für Authentifizierungsprofile nativer Agent-Runtime-Adapter. Aus `plugin-sdk/agent-runtime-test-contracts` importieren            |
| `DELIVERY_NO_REPLY_RUNTIME_CONTRACT`                 | Gemeinsame Vertrags-Fixture für die Zustellungsunterdrückung nativer Agent-Runtime-Adapter. Aus `plugin-sdk/agent-runtime-test-contracts` importieren    |
| `OUTCOME_FALLBACK_RUNTIME_CONTRACT`                  | Gemeinsame Vertrags-Fixture für die Fallback-Klassifizierung nativer Agent-Runtime-Adapter. Aus `plugin-sdk/agent-runtime-test-contracts` importieren |
| `createParameterFreeTool`                            | Fixtures für dynamische Tool-Schemas für Vertragstests nativer Runtimes erstellen. Aus `plugin-sdk/agent-runtime-test-contracts` importieren              |
| `expectChannelInboundContextContract`                | Form des eingehenden Channel-Kontexts prüfen. Aus `plugin-sdk/channel-contract-testing` importieren                                                  |
| `installChannelOutboundPayloadContractSuite`         | Vertragsfälle für ausgehende Channel-Nutzlasten installieren. Aus `plugin-sdk/channel-contract-testing` importieren                                       |
| `createStartAccountContext`                          | Kontexte für den Lebenszyklus von Channel-Konten erstellen. Aus `plugin-sdk/channel-test-helpers` importieren                                                  |
| `installChannelActionsContractSuite`                 | Generische Vertragsfälle für Channel-Nachrichtenaktionen installieren. Aus `plugin-sdk/channel-test-helpers` importieren                                     |
| `installChannelSetupContractSuite`                   | Generische Vertragsfälle für die Channel-Einrichtung installieren. Aus `plugin-sdk/channel-test-helpers` importieren                                              |
| `installChannelStatusContractSuite`                  | Generische Vertragsfälle für den Channel-Status installieren. Aus `plugin-sdk/channel-test-helpers` importieren                                             |
| `expectDirectoryIds`                                 | Channel-Verzeichnis-IDs aus einer Verzeichnislistenfunktion prüfen. Aus `plugin-sdk/channel-test-helpers` importieren                               |
| `assertBundledChannelEntries`                        | Prüfen, ob Einstiegspunkte gebündelter Channels den erwarteten öffentlichen Vertrag bereitstellen. Aus `plugin-sdk/channel-test-helpers` importieren                    |
| `formatEnvelopeTimestamp`                            | Deterministische Zeitstempel für Umschläge formatieren. Aus `plugin-sdk/channel-test-helpers` importieren                                                  |
| `expectPairingReplyText`                             | Antworttext der Channel-Kopplung prüfen und dessen Code extrahieren. Aus `plugin-sdk/channel-test-helpers` importieren                                    |
| `describePluginRegistrationContract`                 | Vertragsprüfungen für die Plugin-Registrierung installieren. Aus `plugin-sdk/plugin-test-contracts` importieren                                              |
| `registerSingleProviderPlugin`                       | Ein Provider-Plugin in Smoke-Tests des Loaders registrieren. Aus `plugin-sdk/plugin-test-runtime` importieren                                         |
| `registerProviderPlugin`                             | Alle Provider-Arten eines Plugins erfassen. Aus `plugin-sdk/plugin-test-runtime` importieren                                                 |
| `registerProviderPlugins`                            | Provider-Registrierungen über mehrere Plugins hinweg erfassen. Aus `plugin-sdk/plugin-test-runtime` importieren                                     |
| `requireRegisteredProvider`                          | Prüfen, ob eine Provider-Sammlung eine ID enthält. Aus `plugin-sdk/plugin-test-runtime` importieren                                           |
| `createRuntimeEnv`                                   | Eine simulierte CLI-/Plugin-Runtime-Umgebung erstellen. Aus `plugin-sdk/plugin-test-runtime` importieren                                              |
| `createPluginRuntimeMock`                            | Eine simulierte Plugin-Runtime-Oberfläche erstellen. Aus `plugin-sdk/plugin-test-runtime` importieren                                                      |
| `createPluginSetupWizardStatus`                      | Hilfsfunktionen für den Einrichtungsstatus von Channel-Plugins erstellen. Aus `plugin-sdk/plugin-test-runtime` importieren                                             |
| `createTestWizardPrompter`                           | Einen simulierten Prompter für den Einrichtungsassistenten erstellen. Aus `plugin-sdk/plugin-test-runtime` importieren                                                       |
| `createRuntimeTaskFlow`                              | Isolierten TaskFlow-Zustand der Runtime erstellen. Aus `plugin-sdk/plugin-test-runtime` importieren                                                    |
| `runProviderCatalog`                                 | Einen Provider-Katalog-Hook mit Testabhängigkeiten ausführen. Aus `plugin-sdk/plugin-test-runtime` importieren                                     |
| `resolveProviderWizardOptions`                       | Auswahlmöglichkeiten des Provider-Einrichtungsassistenten in Vertragstests auflösen. Aus `plugin-sdk/plugin-test-runtime` importieren                                    |
| `resolveProviderModelPickerEntries`                  | Einträge der Provider-Modellauswahl in Vertragstests auflösen. Aus `plugin-sdk/plugin-test-runtime` importieren                                    |
| `buildProviderPluginMethodChoice`                    | IDs für Auswahlmöglichkeiten des Provider-Assistenten für Prüfungen erstellen. Aus `plugin-sdk/plugin-test-runtime` importieren                                            |
| `setProviderWizardProvidersResolverForTest`          | Provider für den Provider-Assistenten in isolierte Tests injizieren. Aus `plugin-sdk/plugin-test-runtime` importieren                                        |
| `describeOpenAIProviderRuntimeContract`              | Runtime-Vertragsprüfungen für Provider-Familien installieren. Aus `plugin-sdk/provider-test-contracts` importieren                                        |
| `expectPassthroughReplayPolicy`                      | Prüfen, ob Provider-Wiedergaberichtlinien Provider-eigene Tools und Metadaten unverändert weiterreichen. Aus `plugin-sdk/provider-test-contracts` importieren         |
| `runRealtimeSttLiveTest`                             | Einen Live-Test eines Echtzeit-STT-Providers mit gemeinsamen Audio-Fixtures ausführen. Aus `plugin-sdk/provider-test-contracts` importieren                       |
| `normalizeTranscriptForMatch`                        | Live-Transkriptausgabe vor unscharfen Prüfungen normalisieren. Aus `plugin-sdk/provider-test-contracts` importieren                               |
| `expectExplicitVideoGenerationCapabilities`          | Prüfen, ob Video-Provider explizite Funktionen für Generierungsmodi deklarieren. Aus `plugin-sdk/provider-test-contracts` importieren                   |
| `expectExplicitMusicGenerationCapabilities`          | Prüfen, ob Musik-Provider explizite Generierungs-/Bearbeitungsfunktionen deklarieren. Aus `plugin-sdk/provider-test-contracts` importieren                   |
| `mockSuccessfulDashscopeVideoTask`                   | Eine erfolgreiche DashScope-kompatible Antwort für eine Videoaufgabe installieren. Aus `plugin-sdk/provider-test-contracts` importieren                          |
| `getProviderHttpMocks`                               | Auf explizit aktivierte Vitest-Mocks für Provider-HTTP/-Authentifizierung zugreifen. Aus `plugin-sdk/provider-http-test-mocks` importieren                                         |
| `installProviderHttpMockCleanup`                     | Provider-HTTP/-Authentifizierungs-Mocks nach jedem Test zurücksetzen. Aus `plugin-sdk/provider-http-test-mocks` importieren                                        |
| `installCommonResolveTargetErrorCases`               | Gemeinsame Testfälle für die Fehlerbehandlung bei der Zielauflösung. Aus `plugin-sdk/channel-target-testing` importieren                                  |
| `shouldAckReaction`                                  | Prüfen, ob ein Channel eine Bestätigungsreaktion hinzufügen soll. Aus `plugin-sdk/channel-feedback` importieren                                            |
| `removeAckReactionAfterReply`                        | Bestätigungsreaktion nach der Zustellung der Antwort entfernen. Aus `plugin-sdk/channel-feedback` importieren                                                      |
| `createTestRegistry`                                 | Eine Registry-Fixture für Channel-Plugins erstellen. Aus `plugin-sdk/plugin-test-runtime` oder `plugin-sdk/channel-test-helpers` importieren               |
| `createEmptyPluginRegistry`                          | Eine leere Plugin-Registry-Fixture erstellen. Aus `plugin-sdk/plugin-test-runtime` oder `plugin-sdk/channel-test-helpers` importieren                |
| `setActivePluginRegistry`                            | Eine Registry-Fixture für Plugin-Runtime-Tests installieren. Aus `plugin-sdk/plugin-test-runtime` oder `plugin-sdk/channel-test-helpers` importieren   |
| `createRequestCaptureJsonFetch`                      | JSON-Abrufanfragen in Tests von Medien-Hilfsfunktionen erfassen. Aus `plugin-sdk/test-media-understanding` importieren                                     |
| `isLiveTestEnabled`                                  | Explizit aktivierte Live-Provider-Tests absichern. Aus `plugin-sdk/test-live` importieren                                                                      |
| `collectProviderApiKeys`                             | Anmeldedaten für Live-Provider-Tests ermitteln. Aus `plugin-sdk/test-live-auth` importieren                                                    |
| `parseProviderModelMap`                              | Modellüberschreibungen für Musik-/Video-Live-Tests parsen. Aus `plugin-sdk/test-media-generation` importieren                                              |
| `withServer`                                         | Tests mit einem temporären lokalen HTTP-Server ausführen. Aus `plugin-sdk/test-env` importieren                                                      |
| `createMockIncomingRequest`                          | Ein minimales Objekt für eingehende HTTP-Anfragen erstellen. Aus `plugin-sdk/test-env` importieren                                                          |
| `withFetchPreconnect`                                | Abruf-Tests mit installierten Preconnect-Hooks ausführen. Aus `plugin-sdk/test-env` importieren                                                       |
| `withEnv` / `withEnvAsync`                           | Umgebungsvariablen vorübergehend patchen. Aus `plugin-sdk/test-env` importieren                                                               |
| `createTempHomeEnv` / `withTempHome` / `withTempDir` | Isolierte Dateisystem-Test-Fixtures erstellen. Aus `plugin-sdk/test-env` importieren                                                              |
| `createMockServerResponse`                           | Einen minimalen Mock für eine HTTP-Serverantwort erstellen. Aus `plugin-sdk/test-env` importieren                                                            |
| `createProviderUsageFetch`                           | Fixtures für den Abruf der Provider-Nutzung erstellen. Aus `plugin-sdk/test-env` importieren                                                                   |
| `useFrozenTime` / `useRealTime`                      | Timer für zeitkritische Tests einfrieren und wiederherstellen. Aus `plugin-sdk/test-env` importieren                                                    |
| `createCliRuntimeCapture`                            | CLI-Runtime-Ausgabe in Tests erfassen. Aus `plugin-sdk/test-fixtures` importieren                                                              |
| `importFreshModule`                                  | Ein ESM-Modul mit einem neuen Abfrage-Token importieren, um den Modul-Cache zu umgehen. Aus `plugin-sdk/test-fixtures` importieren                             |
| `bundledPluginRoot` / `bundledPluginFile`            | Quell- oder Distributions-Fixture-Pfade gebündelter Plugins auflösen. Aus `plugin-sdk/test-fixtures` importieren                                              |
| `mockNodeBuiltinModule`                              | Eng begrenzte Vitest-Mocks für integrierte Node-Module installieren. Aus `plugin-sdk/test-node-mocks` importieren                                                       |
| `createSandboxTestContext`                           | Sandbox-Testkontexte erstellen. Aus `plugin-sdk/test-fixtures` importieren                                                                      |
| `writeSkill`                                         | Skills-Fixtures schreiben. Aus `plugin-sdk/test-fixtures` importieren                                                                             |
| `makeAgentAssistantMessage`                          | Nachrichten-Fixtures für Agent-Transkripte erstellen. Aus `plugin-sdk/test-fixtures` importieren                                                          |
| `peekSystemEvents` / `resetSystemEventsForTest`      | Systemereignis-Fixtures prüfen und zurücksetzen. Aus `plugin-sdk/test-fixtures` importieren                                                          |
| `sanitizeTerminalText`                               | Terminalausgabe für Prüfungen bereinigen. Aus `plugin-sdk/test-fixtures` importieren                                                          |
| `countLines` / `hasBalancedFences`                   | Form der Chunking-Ausgabe sicherstellen. Aus `plugin-sdk/test-fixtures` importieren                                                                     |
| `typedCases`                                         | Literale Typen für tabellengesteuerte Tests beibehalten. Aus `plugin-sdk/test-fixtures` importieren                                                    |

Gebündelte Plugin-Vertragssuiten verwenden diese SDK-Testunterpfade außerdem für
reine Testhilfen für Registry, Manifest, öffentliche Artefakte und Runtime-Fixtures.
Nur für den Core bestimmte Suiten, die vom gebündelten OpenClaw-Inventar abhängen, verbleiben
stattdessen unter `src/plugins/contracts`.

### Typen

Spezifische Testunterpfade re-exportieren außerdem Typen, die in Testdateien nützlich sind:

```typescript
import type {
  ChannelAccountSnapshot,
  ChannelGatewayContext,
} from "openclaw/plugin-sdk/channel-contract";
import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
import type { MockFn, PluginRuntime, RuntimeEnv } from "openclaw/plugin-sdk/plugin-test-runtime";
```

## Auflösung von Testzielen

Verwenden Sie `installCommonResolveTargetErrorCases`, um Standardfehlerfälle für die
Auflösung von Kanalzielen hinzuzufügen:

```typescript
import { describe } from "vitest";
import { installCommonResolveTargetErrorCases } from "openclaw/plugin-sdk/channel-target-testing";

describe("Zielauflösung für my-channel", () => {
  installCommonResolveTargetErrorCases({
    resolveTarget: ({ to, mode, allowFrom }) => {
      // Zielauflösungslogik Ihres Kanals
      return myChannelResolveTarget({ to, mode, allowFrom });
    },
    implicitAllowFrom: ["user1", "user2"],
  });

  // Kanalspezifische Testfälle hinzufügen
  it("sollte @username-Ziele auflösen", () => {
    // ...
  });
});
```

## Testmuster

### Testen von Registrierungsverträgen

Unit-Tests, die einen manuell erstellten `api`-Mock an `register(api)` übergeben,
prüfen die Akzeptanzprüfungen des OpenClaw-Loaders nicht. Fügen Sie mindestens einen Loader-gestützten
Smoke-Test für jede Registrierungsoberfläche hinzu, von der Ihr Plugin abhängt, insbesondere
für Hooks und exklusive Fähigkeiten wie Memory.

Der tatsächliche Loader lässt die Plugin-Registrierung fehlschlagen, wenn erforderliche Metadaten fehlen oder
ein Plugin eine Capability-API aufruft, die ihm nicht gehört. Beispielsweise
erfordert `api.registerHook(...)` einen Hook-Namen, und
`api.registerMemoryCapability(...)` erfordert, dass das Plugin-Manifest oder der exportierte
Einstiegspunkt `kind: "memory"` deklariert.

### Testen des Zugriffs auf die Runtime-Konfiguration

Bevorzugen Sie den gemeinsamen Plugin-Runtime-Mock aus
`openclaw/plugin-sdk/plugin-test-runtime`. Seine Hilfsfunktionen für die Runtime-Konfiguration bilden die
aktuellen Snapshot- und Mutations-APIs ab.

### Unit-Test eines Kanal-Plugins

```typescript
import { describe, it, expect, vi } from "vitest";

describe("Plugin my-channel", () => {
  it("sollte das Konto aus der Konfiguration auflösen", () => {
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

  it("sollte das Konto prüfen, ohne Secrets zu materialisieren", () => {
    const cfg = {
      channels: {
        "my-channel": { token: "test-token" },
      },
    };

    const inspection = myPlugin.setup.inspectAccount(cfg, undefined);
    expect(inspection.configured).toBe(true);
    expect(inspection.tokenStatus).toBe("available");
    // Kein Token-Wert offengelegt
    expect(inspection).not.toHaveProperty("token");
  });
});
```

### Unit-Test eines Provider-Plugins

```typescript
import { describe, it, expect } from "vitest";

describe("Plugin my-provider", () => {
  it("sollte dynamische Modelle auflösen", () => {
    const model = myProvider.resolveDynamicModel({
      modelId: "custom-model-v2",
      // ... Kontext
    });

    expect(model.id).toBe("custom-model-v2");
    expect(model.provider).toBe("my-provider");
    expect(model.api).toBe("openai-completions");
  });

  it("sollte den Katalog zurückgeben, wenn ein API-Schlüssel verfügbar ist", async () => {
    const result = await myProvider.catalog.run({
      resolveProviderApiKey: () => ({ apiKey: "test-key" }),
      // ... Kontext
    });

    expect(result?.provider?.models).toHaveLength(2);
  });
});
```

### Mocking der Plugin-Runtime

Für Code, der `createPluginRuntimeStore` verwendet, mocken Sie die Runtime in Tests:

```typescript
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

const store = createPluginRuntimeStore<PluginRuntime>({
  pluginId: "test-plugin",
  errorMessage: "Test-Runtime nicht festgelegt",
});

// In der Testeinrichtung
const mockRuntime = {
  agent: {
    resolveAgentDir: vi.fn().mockReturnValue("/tmp/agent"),
    // ... weitere Mocks
  },
  config: {
    current: vi.fn(() => ({}) as const),
    mutateConfigFile: vi.fn(),
    replaceConfigFile: vi.fn(),
  },
  // ... weitere Namespaces
} as unknown as PluginRuntime;

store.setRuntime(mockRuntime);

// Nach den Tests
store.clearRuntime();
```

### Testen mit instanzbezogenen Stubs

Bevorzugen Sie instanzbezogene Stubs gegenüber der Mutation des Prototyps:

```typescript
// Bevorzugt: instanzbezogener Stub
const client = new MyChannelClient();
client.sendMessage = vi.fn().mockResolvedValue({ id: "msg-1" });

// Vermeiden: Mutation des Prototyps
// MyChannelClient.prototype.sendMessage = vi.fn();
```

## Vertragstests (Plugins im Repository)

Gebündelte Plugins verfügen über Vertragstests, die die Eigentümerschaft der Registrierung überprüfen:

```bash
pnpm test src/plugins/contracts/
```

Diese Tests prüfen:

- Welche Plugins welche Provider registrieren
- Welche Plugins welche Sprachanbieter registrieren
- Korrektheit der Registrierungsstruktur
- Einhaltung des Runtime-Vertrags

### Ausführen eingegrenzter Tests

Für ein bestimmtes Plugin:

```bash
pnpm test <bundled-plugin-root>/my-channel/
```

Nur für Vertragstests:

```bash
pnpm test src/plugins/contracts/shape.contract.test.ts
pnpm test src/plugins/contracts/auth-choice.contract.test.ts
pnpm test src/plugins/contracts/runtime-seams.contract.test.ts
```

## Lint-Durchsetzung (Plugins im Repository)

`scripts/run-additional-boundary-checks.mjs` führt in der CI eine Reihe von `lint:plugins:*`-
Prüfungen der Importgrenzen aus; jede davon kann auch lokal eigenständig ausgeführt werden:

| Befehl                                                        | Erzwingt                                                                                     |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `pnpm run lint:plugins:no-monolithic-plugin-sdk-entry-imports` | Gebündelte Plugins dürfen das monolithische Root-Barrel `openclaw/plugin-sdk` nicht importieren.              |
| `pnpm run lint:plugins:no-extension-src-imports`               | Extension-Produktionsdateien dürfen den Repository-Baum `src/**` nicht direkt importieren (`../../src/...`).  |
| `pnpm run lint:plugins:no-extension-test-core-imports`         | Extension-Testdateien dürfen keine entfernten SDK-Testaliase oder andere nur für den Core bestimmte Testhilfen importieren. |

Externe Plugins unterliegen diesen Lint-Regeln nicht, es wird jedoch empfohlen,
dieselben Muster zu befolgen.

## Testkonfiguration

OpenClaw verwendet Vitest 4 mit informativen V8-Coverage-Berichten. Für Plugin-Tests:

```bash
# Alle Tests ausführen
pnpm test

# Tests eines bestimmten Plugins ausführen
pnpm test <bundled-plugin-root>/my-channel/src/channel.test.ts

# Mit einem bestimmten Testnamenfilter ausführen
pnpm test <bundled-plugin-root>/my-channel/ -t "resolves account"

# Mit Coverage ausführen
pnpm test:coverage
```

Wenn lokale Ausführungen zu hoher Speicherauslastung führen:

```bash
OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test
```

## Verwandte Themen

- [SDK-Übersicht](/de/plugins/sdk-overview) -- Importkonventionen
- [SDK-Kanal-Plugins](/de/plugins/sdk-channel-plugins) -- Schnittstelle für Kanal-Plugins
- [SDK-Provider-Plugins](/de/plugins/sdk-provider-plugins) -- Hooks für Provider-Plugins
- [Plugins erstellen](/de/plugins/building-plugins) -- Leitfaden für den Einstieg
