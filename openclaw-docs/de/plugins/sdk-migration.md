---
read_when:
    - Sie sehen die Warnung OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Sie sehen die Warnung OPENCLAW_EXTENSION_API_DEPRECATED
    - Sie haben vor OpenClaw 2026.4.25 `api.registerEmbeddedExtensionFactory` verwendet.
    - Sie aktualisieren ein Plugin auf die moderne Plugin-Architektur
    - Sie pflegen ein externes OpenClaw-Plugin
sidebarTitle: Migrate to SDK
summary: Von der veralteten Abwärtskompatibilitätsschicht zum modernen Plugin-SDK migrieren
title: Plugin-SDK-Migration
x-i18n:
    generated_at: "2026-07-26T18:00:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a483f9c0f8409505fc2688872995382944e002520ceb651214dbc5ad8e3554fb
    source_path: plugins/sdk-migration.md
    workflow: 16
---

OpenClaw hat eine umfassende Abwärtskompatibilitätsschicht durch eine moderne Plugin-
Architektur ersetzt, die aus kleinen, fokussierten Imports aufgebaut ist. Wenn Ihr Plugin
vor dieser Änderung entstand, führt dieser Leitfaden es zu den aktuellen Verträgen über.

## Was sich geändert hat

Mehrere sehr weit gefasste Import-Oberflächen ermöglichten Plugins früher den Zugriff auf
fast alles über einen einzigen Einstiegspunkt:

- **`openclaw/plugin-sdk`** und **`openclaw/plugin-sdk/compat`** – exportierten
  Dutzende Hilfsfunktionen erneut, während das fokussierte SDK entwickelt wurde. Beide Wurzeln
  wurden inzwischen entfernt; importieren Sie stattdessen einen dokumentierten Unterpfad.
- **`openclaw/plugin-sdk/infra-runtime`** – ein umfassendes Barrel, das System-
  ereignisse, Heartbeat-Zustand, Zustellwarteschlangen, Fetch-/Proxy-Hilfsfunktionen, Dateihilfen,
  Genehmigungstypen und nicht zusammengehörige Dienstprogramme vermischte.
- **`openclaw/plugin-sdk/config-runtime`** – ein umfassendes Konfigurations-Barrel, das
  nur für sein späteres Kompatibilitätsfenster beibehalten wurde; direkte Hilfsfunktionen zum
  Laden und Schreiben zur Laufzeit wurden entfernt.
- **`openclaw/extension-api`** – eine entfernte Brücke, die Plugins direkten
  Zugriff auf hostseitige Hilfsfunktionen wie den eingebetteten Agent-Runner gewährte.
- **`api.registerEmbeddedExtensionFactory(...)`** – ein entfernter, ausschließlich für den
  eingebetteten Runner bestimmter Hook, der Ereignisse des eingebetteten Runners wie `tool_result` beobachtete. Verwenden Sie stattdessen Middleware für Agent-
  Werkzeugergebnisse (siehe [Erweiterungen für eingebettete Werkzeugergebnisse
  zu Middleware migrieren](#how-to-migrate)).

Das SDK-Stammverzeichnis, das Kompatibilitäts-Barrel, die Erweiterungsbrücke und die Factory
für eingebettete Erweiterungen wurden entfernt. `infra-runtime` und `config-runtime` bleiben nur für ihre
separat dokumentierten späteren Zeitfenster bestehen; neue Plugins sollten fokussierte Unterpfade verwenden.

<Warning>
  Plugins, die die entfernten Stamm-, Kompatibilitäts- oder Erweiterungsoberflächen importieren,
  werden nicht mehr geladen. Befolgen Sie vor dem Upgrade die nachstehenden Zuordnungen.
</Warning>

OpenClaw entfernt oder interpretiert dokumentiertes Plugin-Verhalten nicht in derselben
Änderung neu, die einen Ersatz einführt. Inkompatible Vertragsänderungen durchlaufen zunächst
einen Kompatibilitätsadapter, Diagnosen, Dokumentation und ein Veraltungszeitfenster. Das
gilt für SDK-Imports, Manifestfelder, Einrichtungs-APIs, Hooks und das
Registrierungsverhalten zur Laufzeit.

### Warum

- **Langsamer Start** – der Import einer Hilfsfunktion lud Dutzende nicht zusammengehöriger Module.
- **Zirkuläre Abhängigkeiten** – umfassende Re-Exporte erleichterten das
  Erzeugen von Importzyklen.
- **Unklare API-Oberfläche** – stabile Exporte ließen sich nicht von internen unterscheiden.

Jedes `openclaw/plugin-sdk/<subpath>` ist jetzt ein kleines, eigenständiges Modul mit
einem dokumentierten Vertrag.

Auch ältere Provider-Komfortschnittstellen für gebündelte Kanäle wurden entfernt –
kanalspezifische Hilfsabkürzungen waren private Annehmlichkeiten des Mono-Repos und keine
stabilen Plugin-Verträge. Verwenden Sie stattdessen schmale, generische SDK-Unterpfade. Behalten Sie
innerhalb des Arbeitsbereichs gebündelter Plugins Provider-eigene Hilfsfunktionen im jeweiligen
`api.ts` oder `runtime-api.ts` dieses Plugins:

- Anthropic behält Claude-spezifische Stream-Hilfsfunktionen in seiner eigenen `api.ts`- /
  `contract-api.ts`-Schnittstelle.
- OpenAI behält Provider-Builder, Hilfsfunktionen für Standardmodelle und Builder für Echtzeit-Provider
  in seinem eigenen `api.ts`.
- OpenRouter behält den Provider-Builder und Hilfsfunktionen für Onboarding und Konfiguration in seinem eigenen
  `api.ts`.

## Kompatibilitätsrichtlinie

Kompatibilitätsarbeiten für externe Plugins erfolgen in dieser Reihenfolge:

1. Fügen Sie den neuen Vertrag hinzu.
2. Erhalten Sie das alte Verhalten über einen Kompatibilitätsadapter.
3. Geben Sie eine Diagnose oder Warnung aus, die den alten Pfad und seinen Ersatz nennt.
4. Decken Sie beide Pfade mit Tests ab.
5. Dokumentieren Sie die Veraltung und den Migrationspfad.
6. Entfernen Sie den alten Pfad erst nach dem angekündigten Migrationszeitraum, üblicherweise in einem Major-
   Release.

Wenn ein Manifestfeld weiterhin akzeptiert wird, verwenden Sie es weiter, bis Dokumentation und
Diagnosen etwas anderes angeben. Neuer Code sollte den dokumentierten Ersatz bevorzugen;
bestehende Plugins dürfen bei gewöhnlichen Minor-Releases nicht ausfallen.

### Kompatibilität der Einrichtung veröffentlichter Kanäle

Über `2026.7.1` veröffentlichte Pakete für Slack, Discord, Signal und Microsoft Teams
importieren kanalspezifische Konfigurationsschemas aus
`openclaw/plugin-sdk/bundled-channel-config-schema`. Die veröffentlichten Pakete für Slack und
Discord importieren außerdem `createLegacyCompatChannelDmPolicy` und
`promptLegacyChannelAllowFromForAccount` aus
`openclaw/plugin-sdk/setup-runtime`.

Diese Exporte bleiben als veraltete Kompatibilitätsadapter zur Laufzeit verfügbar.
Neue und erneut veröffentlichte Plugins sollten ihre Konfigurationsschemas und Einrichtungsrichtlinien
lokal verwalten und dafür generische Primitive aus `channel-config-schema` und
`setup-runtime` verwenden. Die Kompatibilitätsexporte dürfen erst entfernt werden, wenn die
unterstützten Mindestversionen der veröffentlichten Pakete sie nicht mehr importieren.

### Kompatibilität der Eingabefelder für die Kanaleinrichtung

`ChannelSetupInput` behält jetzt dauerhaft nur noch den kanalübergreifenden Einrichtungsrahmen
typisiert. Kanalspezifische Felder bleiben in einer veralteten Kompatibilitäts-
ebene typisiert, damit vorhandene externe Plugins weiterhin kompiliert werden, während Plugin-Autoren diese
Felder in Plugin-lokale Eingabetypen für die Einrichtung verschieben.

OpenClaw veröffentlicht keine Major-Releases. Eine Registry-Prüfung vom 2026-07-22 untersuchte
426 veröffentlichte, außerhalb des Repositorys verwaltete Kanal-Plugins und entfernte 21 Felder ohne Leser.
Die 22 beibehaltenen Felder haben jeweils einen bekannten veröffentlichten Leser. Jedes weitere Feld wird
gelöscht, sobald es von keinem veröffentlichten Plugin mehr gelesen wird; die beibehaltene Menge schrumpft,
während Plugin-Autoren zu Plugin-lokalen Eingabetypen für die Einrichtung migrieren.

Dieselbe Prüfung entfernte 23 ältere, nicht deklarierte Schlüssel für die Adapter-Hochstufung ohne
veröffentlichte Abhängige. Sechs gebräuchliche Schlüssel und der nur für die Einrichtung bestimmte Schlüssel `rooms` bleiben bestehen.
Auch diese Menge schrumpft, während veröffentlichte Plugins `singleAccountKeysToMove` deklarieren.

Der gemeinsame Typ besitzt keine Indexsignatur. Plugin-eigene Schlüssel können weiterhin
in Eingabeobjekten zur Laufzeit vorhanden sein; deklarieren Sie sie in einer Plugin-lokalen Schnittmenge oder grenzen
Sie sie über das Einrichtungsschema des zuständigen Plugins ein.

| `code`                                  | `owner`   | `replacement`                                                                                    | Bedingung für die Entfernung                                           |
| --------------------------------------- | --------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| `plugin-sdk-channel-setup-input-fields` | `channel` | Bilden Sie eine Schnittmenge aus `ChannelSetupInput` und einem Plugin-lokalen Typ, der die Felder des zuständigen Kanals deklariert | Löschen Sie ein Feld, wenn die Registry-Prüfung veröffentlichter Plugins keinen Leser findet |

Die ältere Ebene zur Hochstufung nicht deklarierter Adapter folgt derselben
lesergesteuerten Richtlinie. Deklarieren Sie `singleAccountKeysToMove`, einschließlich eines leeren Arrays, wenn das
Plugin keine zusätzlichen Hochstufungsschlüssel benötigt, damit der gemeinsame Fallback Schlüssel für
Schlüssel außer Betrieb genommen werden kann.

#### Leser überprüfen

1. Blättern Sie mit jedem `nextCursor` durch `https://clawhub.ai/api/v1/packages?family=code-plugin&limit=100` und behalten Sie Pakete bei, deren `categories` `channels` enthalten.
2. Fügen Sie npm-Kandidaten aus `npm search --json --searchlimit=1000 "openclaw channel plugin"` hinzu. Fügen Sie reine Quellcode-Kandidaten aus GitHub-Codesuchen nach `openclaw/plugin-sdk/channel-setup`, `openclaw/plugin-sdk/setup` und `openclaw/plugin-sdk/core` hinzu.
3. Ermitteln Sie für jeden Kandidaten die neueste veröffentlichte Version. Führen Sie `npm pack <package>@<version> --json --pack-destination <temp-dir>` aus, entpacken Sie sie und untersuchen Sie den ausgelieferten `dist`-JavaScript-Code und die Deklarationen auf direkte oder destrukturierte Feldzugriffe. Laden Sie das ClawHub-Artefakt herunter, wenn ein Paket keine npm-Veröffentlichung besitzt.
4. Erfassen Sie Paket, Version, Feld oder Hochstufungsschlüssel und die übereinstimmende Datei. Ein Feld oder Schlüssel darf nur gelöscht werden, wenn kein veröffentlichtes Plugin-Artefakt darauf zugreift. Halten Sie die Lesernamen in den Codekommentaren neben den Listen der beibehaltenen Felder und Schlüssel mit der Prüfung synchron.

Dies ist ausschließlich ein Kompatibilitätsdatensatz für Quellcode und Typen. Er besitzt keinen Adapter zur Laufzeit und
keinen Eintrag in der Kompatibilitäts-Registry, da Eingabeobjekte für die Einrichtung und das Einrichtungs-
verhalten zur Laufzeit unverändert bleiben.

Prüfen Sie die aktuelle Migrationswarteschlange mit `pnpm plugins:boundary-report`:

| Flag                                                    | Wirkung                                                                         |
| ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `--summary` (oder `pnpm plugins:boundary-report:summary`) | Kompakte Anzahlen statt vollständiger Details.                                  |
| `--json`                                                | Maschinenlesbarer Bericht.                                                      |
| `--owner <id>`                                          | Auf ein Plugin oder einen Kompatibilitätsverantwortlichen filtern.              |
| `--fail-on-cross-owner`                                 | Bei reservierten SDK-Imports über Verantwortlichkeitsgrenzen hinweg mit einem Exit-Code ungleich null beenden. |
| `--fail-on-eligible-compat`                             | Mit einem Exit-Code ungleich null beenden, wenn das Datum `removeAfter` eines veralteten Kompatibilitätsdatensatzes überschritten wurde. |
| `--fail-on-unclassified-unused-reserved`                | Bei ungenutzten reservierten SDK-Shims mit einem Exit-Code ungleich null beenden. |

`pnpm plugins:boundary-report:ci` wird mit allen drei Fehler-Flags ausgeführt. Veraltete
Datensätze besitzen normalerweise ein ausdrückliches Datum `removeAfter` statt eines vagen „nächsten
Major-Releases“. Bei einem Datensatz, dessen Verantwortlicher noch kein Datum genehmigt hat, fehlt
`removeAfter`; er erscheint als `no-date` und kann niemals entfernt werden.
Der Bericht gruppiert veraltete Datensätze nach Datum, zählt lokale Code-/Dokumentationsreferenzen,
zeigt reservierte SDK-Imports über Verantwortlichkeitsgrenzen hinweg an und fasst die private
SDK-Brücke des Speicher-Hosts zusammen. Reservierte SDK-Unterpfade müssen eine nachverfolgte Nutzung durch den Verantwortlichen aufweisen;
ungenutzte reservierte Exporte sollten aus dem öffentlichen SDK entfernt werden.

### Veraltete Medienprojektion

Der Kompatibilitätsdatensatz `media-legacy-projection` deckt die alten parallelen
Medienfelder, Payload-Builder, Metadatenaliase für Hooks und Namen von Medienvorlagen
ab. Das genehmigte Datum `removeAfter` ist **2026-10-01** (zwei Release-Zyklen,
nachdem die Facts-First-Ersatzlösungen ausgeliefert wurden). Die Entfernung erfordert zu diesem Zeitpunkt zusätzlich eine
saubere Prüfung veröffentlichter Plugin-Artefakte; migrieren Sie vor diesem Datum.

Ersetzen Sie für den Kanaleingang die Singular-/Pluralformen `MediaPath`, `MediaUrl`,
`MediaType`, `MediaPaths`, `MediaUrls`, `MediaTypes`,
`MediaTranscribedIndexes`, `MediaWorkspaceDir` und `MediaStaged` durch geordnete
Fakten:

```ts
import { toInboundMediaFacts } from "openclaw/plugin-sdk/channel-inbound";

const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

Verwenden Sie `event.media` in den Hooks `inbound_claim` und `message_received`. Wenn entfernte
Medien nicht lokal bereitgestellt wurden, verwenden Sie `event.originalMedia` für Identität und Diagnosen
und warten Sie auf `event.media`; `event.mediaStagingPending` kennzeichnet diesen
Zustand. Lesen Sie die veralteten Singular-/Plural-Eigenschaften nicht aus
`event.metadata`.

Ersetzen Sie für CLI-Medienmodelle `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}`
und `{{MediaDir}}` durch `{{AttachmentPath}}`, `{{AttachmentUrl}}`,
`{{AttachmentContentType}}` und `{{AttachmentDir}}`. Verwenden Sie
`{{AttachmentIndex}}`, wenn die Position des Anhangs relevant ist.

Importieren Sie für die Richtlinie zum Lesen lokaler Medien `getAgentScopedMediaLocalRoots(...)` oder
`getAgentScopedMediaLocalRootsForSources(...)` aus
`openclaw/plugin-sdk/media-local-roots`. Die
`openclaw/plugin-sdk/agent-media-payload`-Fassade und ihre
`buildAgentMediaPayload(...)`-Projektion sind veraltet.

## Migration

<Steps>
  <Step title="Hilfsfunktionen zum Laden/Schreiben der Laufzeitkonfiguration migrieren">
    Gebündelte Plugins sollten `api.runtime.config.loadConfig()` und
    `api.runtime.config.writeConfigFile(...)` nicht mehr direkt aufrufen. Bevorzugen Sie die Konfiguration, die bereits
    an den aktiven Aufrufpfad übergeben wurde. Langlebige Handler, die den
    aktuellen Prozess-Snapshot benötigen, können `api.runtime.config.current()` verwenden. Langlebige
    Agent-Werkzeuge sollten `ctx.getRuntimeConfig()` innerhalb von `execute` lesen, damit ein Werkzeug,
    das vor dem Schreiben einer Konfiguration erstellt wurde, dennoch die aktualisierte Konfiguration sieht.

    Konfigurationsschreibvorgänge erfolgen über die transaktionale Hilfsfunktion mit einer ausdrücklichen
    Richtlinie für die Zeit nach dem Schreiben:

    ```typescript
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    Verwenden Sie `afterWrite: { mode: "restart", reason: "..." }`, wenn die Änderung
    einen sauberen Neustart des Gateways erfordert, und `afterWrite: { mode: "none", reason: "..." }`
    nur, wenn der Aufrufer für die Nachbereitung verantwortlich ist und den
    Neuladungsplaner bewusst unterdrückt. Mutationsergebnisse enthalten eine typisierte Zusammenfassung `followUp` für
    Tests und Protokollierung; das Gateway bleibt für die Durchführung oder
    Planung des Neustarts verantwortlich.

    `loadConfig` und `writeConfigFile` wurden aus der Plugin-
    Laufzeit entfernt. Gebündelte Plugins und Laufzeitcode des Repositorys werden durch
    `pnpm check:deprecated-api-usage` und
    `pnpm check:no-runtime-action-load-config` geschützt: Neue Verwendung in
    produktivem Plugin-Code schlägt sofort fehl, direkte Konfigurationsschreibvorgänge schlagen fehl, Gateway-Servermethoden müssen
    den Laufzeit-Snapshot der Anfrage verwenden, Laufzeit-Hilfsfunktionen für das Senden, Aktionen und Clients von Kanälen
    müssen die Konfiguration von ihrer Schnittstellengrenze erhalten, und langlebige Laufzeitmodule
    erlauben keine umgebungsbezogenen Aufrufe von `loadConfig()`.

    Neuer Plugin-Code sollte das allgemeine Barrel `openclaw/plugin-sdk/config-runtime`
    vermeiden. Verwenden Sie den spezifischen Unterpfad für die jeweilige Aufgabe:

    | Bedarf | Import |
    | --- | --- |
    | Konfigurationstypen wie `OpenClawConfig` | `openclaw/plugin-sdk/config-contracts` |
    | Konfigurationssuche am Plugin-Einstiegspunkt | `api.pluginConfig` |
    | Zusammenführen von Konfigurationen | Plugin-lokale Logik an der Konfigurationsgrenze |
    | Lesen des aktuellen Laufzeit-Snapshots | `openclaw/plugin-sdk/runtime-config-snapshot` |
    | Konfigurationsschreibvorgänge | `openclaw/plugin-sdk/config-mutation` |
    | Hilfsfunktionen für den Sitzungsspeicher | `openclaw/plugin-sdk/session-store-runtime` |
    | Markdown-Tabellenkonfiguration | `openclaw/plugin-sdk/markdown-table-runtime` |
    | Laufzeit-Hilfsfunktionen für Gruppenrichtlinien | `openclaw/plugin-sdk/runtime-group-policy` |
    | Auflösung geheimer Eingaben | `openclaw/plugin-sdk/secret-input-runtime` |
    | Modell-/Sitzungsüberschreibungen | `openclaw/plugin-sdk/model-session-runtime` |

    Gebündelte Plugins und ihre Tests werden per Scanner gegen das allgemeine
    Barrel geschützt, damit Importe und Mocks lokal auf das benötigte Verhalten beschränkt bleiben. Das
    Barrel besteht für externe Kompatibilität weiterhin, neuer Code sollte jedoch nicht
    davon abhängen.

  </Step>

  <Step title="Eingebettete Erweiterungen für Werkzeugergebnisse auf Middleware migrieren">
    Gebündelte Plugins müssen die ausschließlich für eingebettete Runner vorgesehenen
    Handler für Werkzeugergebnisse `api.registerEmbeddedExtensionFactory(...)` durch
    laufzeitneutrale Middleware ersetzen:

    ```typescript
    // OpenClaw-Laufzeitwerkzeuge und dynamische Werkzeuge der Codex-Laufzeit (das Ergebnis kann
    // transformiert werden). Ergebnisse nativer Codex-Werkzeuge werden zur Beobachtung ebenfalls weitergeleitet,
    // ihre transformierte Ausgabe erreicht das Modell jedoch nie: Der Vertrag des Codex-
    // PostToolUse-Hooks kann die Antwort eines nativen Werkzeugs nicht ersetzen.
    api.registerAgentToolResultMiddleware(async (event) => {
      return compactToolResult(event);
    }, {
      runtimes: ["openclaw", "codex"],
    });
    ```

    Aktualisieren Sie gleichzeitig das Plugin-Manifest:

    ```json
    {
      "contracts": {
        "agentToolResultMiddleware": ["openclaw", "codex"]
      }
    }
    ```

    Installierte Plugins können ebenfalls Middleware für Werkzeugergebnisse registrieren, wenn sie ausdrücklich
    aktiviert ist und jede Ziel-Laufzeit in
    `contracts.agentToolResultMiddleware` deklariert ist. Nicht deklarierte Middleware-
    Registrierungen installierter Plugins werden abgelehnt.

  </Step>

  <Step title="Native Genehmigungshandler auf Fähigkeitsfakten migrieren">
    Genehmigungsfähige Kanal-Plugins stellen natives Genehmigungsverhalten über
    `approvalCapability.nativeRuntime` sowie die gemeinsame Registry für den Laufzeitkontext
    bereit:

    - Ersetzen Sie `approvalCapability.handler.loadRuntime(...)` durch
      `approvalCapability.nativeRuntime`.
    - Verlagern Sie genehmigungsspezifische Authentifizierung/Zustellung von der veralteten Verkabelung `plugin.auth` /
      `plugin.approvals` zu `approvalCapability`.
    - `ChannelPlugin.approvals` wurde aus dem öffentlichen
      Vertrag für Kanal-Plugins entfernt; verschieben Sie Felder für Zustellung, native Funktionen und Rendering nach
      `approvalCapability`.
    - `plugin.auth` bleibt ausschließlich für Anmelde-/Abmeldeabläufe von Kanälen bestehen; der Kern
      liest dort keine Genehmigungs-Authentifizierungs-Hooks mehr.
    - Registrieren Sie kanaleigene Laufzeitobjekte (Clients, Tokens, Bolt-Apps)
      über `openclaw/plugin-sdk/channel-runtime-context`.
    - Senden Sie aus nativen Genehmigungshandlern keine Plugin-eigenen Hinweise zur Umleitung;
      der Kern ist anhand der tatsächlichen Zustellungsergebnisse für Hinweise über anderweitige Weiterleitung verantwortlich.
    - Wenn Sie `channelRuntime` an `createChannelManager(...)` übergeben, stellen Sie eine
      echte Oberfläche `createPluginRuntime().channel` bereit – partielle Stubs werden
      abgelehnt.

    Informationen zum aktuellen Aufbau der Genehmigungsfähigkeiten finden Sie unter [Kanal-Plugins](/de/plugins/sdk-channel-plugins).

  </Step>

  <Step title="Fallback-Verhalten von Windows-Wrappern prüfen">
    Wenn Ihr Plugin `openclaw/plugin-sdk/windows-spawn` verwendet, schlagen nicht aufgelöste Windows-
    Wrapper `.cmd`/`.bat` nun geschlossen fehl, sofern Sie nicht ausdrücklich
    `allowShellFallback: true` übergeben:

    ```typescript
    // Vorher
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Nachher
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Legen Sie dies nur für vertrauenswürdige Kompatibilitätsaufrufer fest, die einen
      // über die Shell vermittelten Fallback bewusst akzeptieren.
      allowShellFallback: true,
    });
    ```

    Wenn Ihr Aufrufer nicht bewusst auf den Shell-Fallback angewiesen ist, setzen Sie
    `allowShellFallback` nicht und behandeln Sie stattdessen den ausgelösten Fehler.

  </Step>

  <Step title="Veraltete Importe finden">
    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "plugin-sdk/infra-runtime" my-plugin/
    grep -r "plugin-sdk/config-runtime" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```
  </Step>

  <Step title="Durch gezielte Importe ersetzen">
    Jeder Export der alten Oberfläche ist einem bestimmten modernen Importpfad zugeordnet:

    ```typescript
    // Vorher (veraltete Abwärtskompatibilitätsschicht)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // Nachher (moderne gezielte Importe)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Verwenden Sie für hostseitige Hilfsfunktionen die injizierte Plugin-Laufzeit, statt
    direkt zu importieren:

    ```typescript
    // Vorher (veraltete extension-api-Brücke)
    import { runEmbeddedAgent } from "openclaw/extension-api";
    const result = await runEmbeddedAgent({ sessionId, prompt });

    // Nachher (injizierte Laufzeit)
    const result = await api.runtime.agent.runEmbeddedAgent({ sessionId, prompt });
    ```

    Dasselbe Muster gilt für andere veraltete Brücken-Hilfsfunktionen:

    | Alter Import | Modernes Äquivalent |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | Hilfsfunktionen für den Sitzungsspeicher | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Allgemeine infra-runtime-Importe ersetzen">
    `openclaw/plugin-sdk/infra-runtime` besteht für externe
    Kompatibilität weiterhin, neuer Code sollte jedoch die tatsächlich benötigte spezifische Oberfläche
    importieren:

    | Bedarf | Import |
    | --- | --- |
    | Hilfsfunktionen für die Systemereigniswarteschlange | `openclaw/plugin-sdk/system-event-runtime` |
    | Hilfsfunktionen für Aktivierung, Ereignisse und Sichtbarkeit von Heartbeat | `openclaw/plugin-sdk/heartbeat-runtime` |
    | Abarbeitung der Warteschlange ausstehender Zustellungen | `openclaw/plugin-sdk/delivery-queue-runtime` |
    | Telemetrie der Kanalaktivität | `openclaw/plugin-sdk/channel-activity-runtime` |
    | Speicherinterne und persistent gestützte Deduplizierungs-Caches | `openclaw/plugin-sdk/dedupe-runtime` |
    | Sichere Hilfsfunktionen für lokale Datei-/Medienpfade | `openclaw/plugin-sdk/file-access-runtime` |
    | Dispatcher-berücksichtigender Abruf | `openclaw/plugin-sdk/runtime-fetch` |
    | Hilfsfunktionen für Proxy- und geschützte Abrufe | `openclaw/plugin-sdk/fetch-runtime` |
    | Typen für SSRF-Dispatcher-Richtlinien | `openclaw/plugin-sdk/ssrf-dispatcher` |
    | Typen für Genehmigungsanfragen/-entscheidungen | `openclaw/plugin-sdk/approval-runtime` |
    | Hilfsfunktionen für Nutzdaten und Befehle von Genehmigungsantworten | `openclaw/plugin-sdk/approval-reply-runtime` |
    | Hilfsfunktionen zur Fehlerformatierung | `openclaw/plugin-sdk/error-runtime` |
    | Warten auf Transportbereitschaft | `openclaw/plugin-sdk/transport-ready-runtime` |
    | Hilfsfunktionen für sichere Tokens | `openclaw/plugin-sdk/secure-random-runtime` |
    | Begrenzte Nebenläufigkeit asynchroner Aufgaben | `openclaw/plugin-sdk/concurrency-runtime` |
    | Pflichtwertprüfungen für beweisbare Invarianten | `openclaw/plugin-sdk/expect-runtime` |
    | Numerische Typumwandlung | `openclaw/plugin-sdk/number-runtime` |
    | Prozesslokale asynchrone Sperre | `openclaw/plugin-sdk/async-lock-runtime` |
    | Dateisperren | `openclaw/plugin-sdk/file-lock` |

    Gebündelte Plugins werden per Scanner gegen `infra-runtime` geschützt, damit Repository-Code
    nicht auf das allgemeine Barrel zurückfällt.

  </Step>

  <Step title="Hilfsfunktionen für Kanalrouten migrieren">
    Neuer Code für Kanalrouten verwendet `openclaw/plugin-sdk/channel-route`. Die älteren
    Namen der Routenschlüssel bleiben als Kompatibilitätsaliase erhalten:

    | Alte Hilfsfunktion | Moderne Hilfsfunktion |
    | --- | --- |
    | `channelRouteIdentityKey(...)` | `channelRouteDedupeKey(...)` |
    | `channelRouteKey(...)` | `channelRouteCompactKey(...)` |

    Die modernen Routen-Hilfsfunktionen normalisieren `{ channel, to, accountId, threadId }`
    konsistent für native Genehmigungen, Antwortunterdrückung, Deduplizierung eingehender Nachrichten,
    Cron-Zustellung und Sitzungsrouting.

    Fügen Sie keine neuen Verwendungen von `ChannelMessagingAdapter.parseExplicitTarget` oder
    `resolveChannelRouteTargetWithParser(...)` aus
    `plugin-sdk/channel-route` hinzu – diese sind veraltet und bleiben nur für ältere
    Plugins erhalten. Neue Kanal-Plugins sollten
    `messaging.targetResolver.resolveTarget(...)` für die Normalisierung von Ziel-IDs
    und den Fallback bei fehlendem Verzeichniseintrag,
    `messaging.inferTargetChatType(...)`, wenn der Kern frühzeitig einen Peer-Typ benötigt,
    und `messaging.resolveOutboundSessionRoute(...)` für Provider-native
    Sitzungs- und Thread-Identitäten verwenden.

  </Step>

  <Step title="Erstellen und testen">
    ```bash
    pnpm build
    pnpm test my-plugin/
    ```
  </Step>
</Steps>

## Referenz für Importpfade

Die öffentliche Exportzuordnung des Pakets ist die maßgebliche Quelle für importierbare SDK-
Unterpfade. Verwenden Sie die thematischen SDK-Leitfäden, die in der [SDK-Übersicht](/de/plugins/sdk-overview)
verlinkt sind, und bevorzugen Sie den spezifischsten dokumentierten öffentlichen Unterpfad. Das Compiler-Inventar in
`scripts/lib/plugin-sdk-entrypoints.json` enthält außerdem private lokale Einträge, die
zum Erstellen gebündelter Plugins verwendet werden; ihre dortige Präsenz macht sie nicht zu öffentlichen Paketexporten.

Diese Tabelle zeigt die übliche Teilmenge für Migrationen, nicht die vollständige SDK-Oberfläche. Das
Inventar der Compiler-Einstiegspunkte befindet sich in `scripts/lib/plugin-sdk-entrypoints.json`;
Paketexporte werden aus der öffentlichen Teilmenge generiert.

Reservierte Hilfsschnittstellen für gebündelte Plugins wurden aus der öffentlichen SDK-
Exportzuordnung entfernt, mit Ausnahme ausdrücklich dokumentierter Kompatibilitätsfassaden wie dem
veralteten Shim `plugin-sdk/discord`, das für externe Plugins beibehalten wird, die weiterhin
das veröffentlichte Paket `@openclaw/discord` direkt importieren. Eigentümerspezifische
Hilfsfunktionen befinden sich innerhalb des jeweils zuständigen Plugin-Pakets; gemeinsames Hostverhalten wird
über generische SDK-Verträge wie `plugin-sdk/gateway-runtime`,
`plugin-sdk/security-runtime` und die injizierte Plugin-API bereitgestellt.

Verwenden Sie den spezifischsten Import, der zur Aufgabe passt. Wenn Sie einen Export nicht finden können,
prüfen Sie den Quellcode unter `src/plugin-sdk/` oder fragen Sie die Maintainer, welcher generische
Vertrag dafür zuständig sein sollte.

## Entfernte Kompatibilitätsoberflächen

Bei der Bereinigung im Juli 2026 wurden das Stamm-SDK und die Compat-Barrels, die Extension-API-
Brücke, die abgelaufenen SDK-Unterpfadaliase, ungenutzte SDK-Unterpfade und die öffentlichen
Exporte für ausschließlich gebündelte SDK-Module entfernt. Ausschließlich gebündelte Module bleiben ihren
Repository-Eigentümern über private lokale Build-Zuordnungen verfügbar; sie können nicht
aus dem veröffentlichten Paket importiert werden.

### Prozessglobale Veröffentlichung von API-Providern

`registerApiProvider(...)` und `unregisterApiProviders(...)` wurden aus
`openclaw/plugin-sdk/llm` entfernt. Sie veröffentlichten API-Transporte im prozessglobalen
Zustand, den lebenszyklusverwaltete Modelllaufzeiten anschließend in jede vorbereitete
Registry kopieren mussten.

Provider-Plugins sollten Textinferenz-Provider über
`api.registerProvider(...)` registrieren. Hosteigener Code und Tests, die eine
`ApiRegistry` erstellen, sollten direkt in dieser Registry registrieren, damit die Zuständigkeit für den Provider
und dessen Abbau auf die vorbereitete Laufzeit beschränkt bleiben.

### Privates Testing-Barrel

`openclaw/plugin-sdk/testing` war Repository-lokal und von ausgelieferten Paketartefakten
ausgeschlossen, daher wurde es vor seinem `removeAfter`-Datum am 2026-07-28 entfernt. Repository-
Tests verwenden spezifische Unterpfade wie `plugin-sdk/plugin-test-runtime`,
`plugin-sdk/channel-test-helpers`, `plugin-sdk/channel-target-testing`,
`plugin-sdk/test-env` und `plugin-sdk/test-fixtures`.

## Migrationsreferenz

  Diese Zuordnungen decken sowohl die im Juli 2026 entfernten Oberflächen als auch die in späteren Zeitfenstern aktiven
  Veraltungen ab. Eine Zuordnung ist eine Migrationsanleitung und kein Nachweis dafür, dass die alte
  Oberfläche weiterhin verfügbar ist; den aktuellen Status finden Sie im Kompatibilitätsregister und im
  Zeitplan für Entfernungen.

  <AccordionGroup>
  <Accordion title="Hilfsfunktionen für command-auth-Hilfe -> command-status">
    **Alt (`openclaw/plugin-sdk/command-auth`)**: `buildCommandsMessage`,
    `buildCommandsMessagePaginated`, `buildHelpMessage`.

    **Neu (`openclaw/plugin-sdk/command-status`)**: dieselben Signaturen, importiert
    aus dem enger gefassten Unterpfad. Die Kompatibilitäts-Re-Exporte von `command-auth`
    wurden entfernt.

    ```typescript
    // Vorher
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-auth";

    // Nachher
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-status";
    ```

  </Accordion>

  <Accordion title="Hilfsfunktionen für Mention-Gating -> resolveInboundMentionDecision">
    **Alt**: `resolveMentionGating(params)` und
    `resolveMentionGatingWithBypass(params)` aus
    `openclaw/plugin-sdk/channel-inbound` oder
    `openclaw/plugin-sdk/channel-mention-gating`.

    **Neu**: `resolveInboundMentionDecision({ facts, policy })` – ein Entscheidungsobjekt
    anstelle zweier getrennter Aufrufformen.

    Übernommen für Discord, iMessage, Matrix, MS Teams, QQBot, Signal,
    Telegram, WhatsApp und Zalo. Slacks eigenes `app_mention`-Ereignismodell
    verwendet diese Hilfsfunktion nicht.

  </Accordion>

  <Accordion title="Channel-Runtime-Shim und Hilfsfunktionen für Channel-Aktionen">
    `openclaw/plugin-sdk/channel-runtime` wurde entfernt. Verwenden Sie
    `openclaw/plugin-sdk/channel-runtime-context`, um Runtime-Objekte zu registrieren.

    Die nativen Hilfsfunktionen für Nachrichtenschemas in `openclaw/plugin-sdk/channel-actions`
    wurden zusammen mit den unstrukturierten „actions“-Channel-Exporten entfernt. Stellen Sie Fähigkeiten
    stattdessen über die semantische Oberfläche `presentation` bereit – Channel-Plugins
    deklarieren, was sie darstellen (Karten, Schaltflächen, Auswahlelemente), statt welche unstrukturierten
    Aktionsnamen sie akzeptieren.

  </Accordion>

  <Accordion title="Websuch-Provider-Hilfsfunktion tool() -> createTool() im Plugin">
    **Alt**: `tool()`-Factory aus `openclaw/plugin-sdk/provider-web-search`.

    **Neu**: Implementieren Sie `createTool(...)` direkt im Provider-Plugin.
    OpenClaw benötigt die SDK-Hilfsfunktion nicht mehr, um den Tool-Wrapper zu registrieren.

  </Accordion>

  <Accordion title="Channel-Umschläge im Klartext -> BodyForAgent">
    **Alt**: `api.runtime.channel.reply.formatInboundEnvelope(...)` (und das
    Feld `channelEnvelope` bei eingehenden Nachrichtenobjekten), um aus eingehenden
    Channel-Nachrichten einen flachen Prompt-Umschlag im Klartext zu erstellen.

    **Neu**: `BodyForAgent` plus strukturierte Benutzerkontextblöcke. Channel-
    Plugins fügen Routing-Metadaten (Thread, Thema, Antwortbezug, Reaktionen) als
    typisierte Felder hinzu, statt sie zu einer Prompt-Zeichenfolge zusammenzufügen. Die
    Hilfsfunktion `formatAgentEnvelope(...)` wird für synthetisch erzeugte,
    an den Assistenten gerichtete Umschläge weiterhin unterstützt, eingehende Klartextumschläge werden jedoch
    abgeschafft.

    Betroffene Bereiche: `inbound_claim`, `message_received` und jedes benutzerdefinierte
    Channel-Plugin, das den alten Umschlagtext nachverarbeitet hat.

  </Accordion>

  <Accordion title="deactivate-Hook -> gateway_stop">
    **Alt**: `api.on("deactivate", handler)`.

    **Neu**: `api.on("gateway_stop", handler)`. Derselbe Vertrag für die Bereinigung beim Herunterfahren;
    nur der Hook-Name ändert sich.

    ```typescript
    // Vorher
    api.on("deactivate", async (event, ctx) => {
      await stopPluginService(ctx);
    });

    // Nachher
    api.on("gateway_stop", async (event, ctx) => {
      await stopPluginService(ctx);
    });
    ```

    `deactivate` bleibt als veralteter Kompatibilitätsalias eingebunden, bis er
    nach dem 2026-08-16 entfernt wird.

  </Accordion>

  <Accordion title="subagent_spawning-Hook -> Thread-Bindung im Kern">
    **Alt**: `api.on("subagent_spawning", handler)`, das
    `threadBindingReady` oder `deliveryOrigin` zurückgibt.

    **Neu**: Lassen Sie den Kern `thread: true`-Subagent-Bindungen über den
    Adapter für Channel-Sitzungsbindungen vorbereiten. Verwenden Sie `api.on("subagent_spawned", handler)`
    nur zur Beobachtung nach dem Start.

    ```typescript
    // Vorher
    api.on("subagent_spawning", async () => ({
      status: "ok",
      threadBindingReady: true,
      deliveryOrigin: { channel: "discord", to: "channel:123", threadId: "456" },
    }));

    // Nachher
    api.on("subagent_spawned", async (event) => {
      await observeSubagentLaunch(event);
    });
    ```

    `subagent_spawning`, `PluginHookSubagentSpawningEvent`,
    `PluginHookSubagentSpawningResult` und
    `SubagentLifecycleHookRunner.runSubagentSpawning(...)` bleiben nur als
    veraltete Kompatibilitätsoberflächen bestehen, während externe Plugins migrieren, und werden
    nach dem 2026-08-30 entfernt.

  </Accordion>

  <Accordion title="Provider-Ermittlungstypen -> Provider-Katalogtypen">
    Vier Aliasse für Ermittlungstypen sind nun dünne Wrapper um die Typen
    der Katalogära:

    | Alter Alias                 | Neuer Typ                  |
    | ------------------------- | ------------------------- |
    | `ProviderDiscoveryOrder`  | `ProviderCatalogOrder`    |
    | `ProviderDiscoveryContext`| `ProviderCatalogContext`  |
    | `ProviderDiscoveryResult` | `ProviderCatalogResult`   |
    | `ProviderPluginDiscovery` | `ProviderPluginCatalog`   |

    Die Aliasse und die veraltete statische Sammlung `ProviderCapabilities` wurden
    entfernt. Provider-Plugins
    sollten explizite Provider-Hooks wie `buildReplayPolicy`,
    `normalizeToolSchemas` und `wrapStreamFn` statt eines statischen Objekts verwenden.

  </Accordion>

  <Accordion title="Hooks für Denk-Richtlinien -> resolveThinkingProfile">
    **Alt** (drei separate Hooks in `ProviderThinkingPolicy`):
    `isBinaryThinking(ctx)`, `supportsXHighThinking(ctx)` und
    `resolveDefaultThinkingLevel(ctx)`.

    **Neu**: ein einzelnes `resolveThinkingProfile(ctx)`, das ein
    `ProviderThinkingProfile` mit dem kanonischen `id`, optionalem `label` und einer
    nach Rang geordneten Stufenliste zurückgibt. OpenClaw stuft veraltete gespeicherte Werte automatisch
    anhand des Profilrangs herunter.

    Der Kontext enthält `provider`, `modelId`, optional zusammengeführte `reasoning`-
    sowie optional zusammengeführte Modellfakten aus `compat`. Provider-Plugins können diese
    Katalogfakten verwenden, um ein modellspezifisches Profil nur dann bereitzustellen, wenn der konfigurierte
    Anfragevertrag dies unterstützt.

    Implementieren Sie einen statt drei Hooks. Die alten Hooks wurden entfernt.

  </Accordion>

  <Accordion title="Externe Authentifizierungs-Provider -> contracts.externalAuthProviders">
    **Alt**: Implementierung externer Authentifizierungs-Hooks, ohne den Provider
    im Plugin-Manifest zu deklarieren.

    **Neu**: Deklarieren Sie `contracts.externalAuthProviders` im Plugin-Manifest
    **und** implementieren Sie `resolveExternalAuthProfiles(...)`.

    ```json
    {
      "contracts": {
        "externalAuthProviders": ["anthropic", "openai"]
      }
    }
    ```

  </Accordion>

  <Accordion title="Nachschlagen von Provider-Umgebungsvariablen -> setup.providers[].envVars">
    **Altes** Manifestfeld: `providerAuthEnvVars: { anthropic: ["ANTHROPIC_API_KEY"] }`.

    **Neu**: Spiegeln Sie dieselbe Suche nach Umgebungsvariablen in `setup.providers[].envVars`
    im Manifest. Dadurch werden Umgebungsmetadaten für Einrichtung und Status an einer Stelle zusammengeführt
    und es wird vermieden, die Plugin-Runtime nur zum Beantworten von Abfragen nach Umgebungsvariablen zu starten.

    `providerAuthEnvVars` wird nicht mehr akzeptiert.

  </Accordion>

  <Accordion title="Registrierung des Memory-Plugins -> registerMemoryCapability">
    **Alt**: drei separate Aufrufe – `api.registerMemoryPromptSection(...)`,
    `api.registerMemoryFlushPlan(...)`, `api.registerMemoryRuntime(...)`.

    **Neu**: ein Aufruf in der Memory-State-API –
    `registerMemoryCapability(pluginId, { promptBuilder, flushPlanResolver, runtime })`.

    Dieselben Slots, ein einzelner Registrierungsaufruf. Additive Hilfsfunktionen für Prompts und Korpora
    (`registerMemoryPromptSupplement`, `registerMemoryCorpusSupplement`) sind
    nicht betroffen.

  </Accordion>

  <Accordion title="API für Memory-Embedding-Provider">
    **Alt**: `api.registerMemoryEmbeddingProvider(...)` plus
    `contracts.memoryEmbeddingProviders`.

    **Neu**: `api.registerEmbeddingProvider(...)` plus
    `contracts.embeddingProviders`.

    Der generische Vertrag für Embedding-Provider ist außerhalb von Memory wiederverwendbar und stellt
    den unterstützten Pfad für neue Provider dar. Die Memory-spezifische Registrierungs-API
    bleibt als veraltete Kompatibilitätsoberfläche eingebunden, während bestehende Provider
    migrieren. Die Plugin-Prüfung meldet die Verwendung durch nicht gebündelte Plugins als
    Kompatibilitätsschuld.

  </Accordion>

  <Accordion title="Unstrukturierte Channel-Sendeergebnisse -> OutboundDeliveryResult">
    **Alt**: `{ ok, messageId, error }` über
    `ChannelSendRawResult` zurückgeben und mit
    `createRawChannelSendResultAdapter(...)` normalisieren.

    **Neu**: Geben Sie `OutboundDeliveryResult`-Felder zurück und fügen Sie den Channel mit
    `createAttachedChannelResultAdapter(...)` hinzu. Fehlgeschlagene Sendevorgänge sollten eine Ausnahme auslösen,
    statt eine Fehlerzeichenfolge zurückzugeben. Der unstrukturierte Ergebnistyp bleibt bis
    zur nächsten Hauptversion des Plugin-SDK verfügbar.

  </Accordion>

  <Accordion title="Typen für Subagent-Sitzungsnachrichten umbenannt">
    Zwei alte Typaliasse werden weiterhin aus `src/plugins/runtime/types.ts` exportiert:

    | Alt                           | Neu                             |
    | ----------------------------- | ------------------------------- |
    | `SubagentReadSessionParams`   | `SubagentGetSessionMessagesParams` |
    | `SubagentReadSessionResult`   | `SubagentGetSessionMessagesResult` |

    Die Runtime-Methode `readSession` ist zugunsten von
    `getSessionMessages` veraltet. Dieselbe Signatur; die alte Methode leitet den Aufruf an die
    neue weiter.

  </Accordion>

  <Accordion title="Entfernte APIs für Sitzungs- und Transkriptdateien">
    Die Umstellung von Sitzungen und Transkripten auf SQLite entfernt oder veraltet Plugin-seitige APIs,
    die aktive `sessions.json`-Speicher, JSONL-Transkriptpfade oder Listen
    von Sitzungsdateien offengelegt haben. Runtime-Plugins sollten Sitzungsidentitäten und SDK-Runtime-
    Hilfsfunktionen verwenden, statt aktive Dateien aufzulösen oder zu verändern.

    | Zu migrierende Oberfläche | Ersatz |
    | ----------------- | ----------- |
    | Veraltete `loadSessionStore(...)`, `updateSessionStore(...)` und `resolveSessionStoreEntry(...)` | `getSessionEntry(...)`, `listSessionEntries(...)` und Sitzungsmutationen auf Zeilenebene. |
    | Veraltetes `resolveSessionFilePath(...)` | Sitzungsidentität (`sessionKey`, `sessionId` und SDK-Runtime-Hilfsfunktionen für Ziele) sowie Gateway-Methoden, die auf der aktuellen Sitzung arbeiten. |
    | Entferntes `saveSessionStore(...)` | Gateway-eigene Sitzungs-Runtime-APIs; Plugin-Code sollte Sitzungszustand über dokumentierte Runtime-/Kontexthilfsfunktionen anfordern oder verändern, statt die aktive Speicherdatei zu schreiben. |
    | Entfernte `resolveSessionTranscriptPathInDir(...)` und `resolveAndPersistSessionFile(...)` | Sitzungsidentität und Gateway-Methoden, die auf der aktuellen Sitzung arbeiten. |
    | `readLatestAssistantTextFromSessionTranscript(...)` | Identitätsbasierte Transkriptleser, die vom aktuellen Runtime-Kontext bereitgestellt werden, oder Gateway-Verlaufs-/Sitzungsmethoden, wenn sich das Plugin außerhalb des Eigentümerpfads des Transkripts befindet. |
    | `SessionTranscriptUpdate.sessionFile` | `SessionTranscriptUpdate.target` mit `agentId`, `sessionKey` und `sessionId`. |
    | Memory-Synchronisierungseingaben wie `sessionFiles` | Vom Host bereitgestellte identitätsbasierte Transkript-/Sitzungsquellen; durchsuchen Sie für aktive Sitzungen keine aktiven JSONL-Dateien. |
    | Runtime-Optionen namens `transcriptPath` oder `sessionFile` für aktive Sitzungen | `sessionTarget`/Runtime-Zielobjekte, die eine speicherneutrale Sitzungsidentität enthalten. |

    Alte JSONL-Transkriptdateien bleiben als Import-, Archiv-, Export- und
    Support-Artefakte gültig. Sie sind nicht länger der dauerhafte Runtime-Vertrag für
    aktive Sitzungen.

    Offizielle Plugins, die mit `v2026.7.1-beta.5` veröffentlicht wurden, importierten die vier
    oben genannten veralteten Hilfsfunktionen. `openclaw/plugin-sdk/session-store-runtime` erhält
    genau diese Brücke bis zum 2026-10-12; neue Plugins müssen die Ersatzlösungen verwenden.
    `resolveStorePath(...)` bleibt eine unterstützte SDK-Hilfsfunktion und ist nicht Teil
    dieser Veraltung.

    `openclaw plugins inspect --all --runtime` meldet nicht gebündelte Plugins, deren
    Ladefehler oder Diagnosen weiterhin auf diese entfernten Datei-APIs verweisen. Der
    Beratungsdurchlauf `@openclaw/plugin-inspector` muss Version `0.3.17` oder
    neuer verwenden, damit Scans externer Pakete vor der Veröffentlichung auch Sitzungs-Hilfsfunktionen
    für den gesamten Speicher, Hilfsfunktionen für Sitzungsdateipfade, alte Transkriptdateiziele und
    Low-Level-Transkripthilfsfunktionen kennzeichnen.

  </Accordion>

  <Accordion title="runtime.tasks.flow -> runtime.tasks.managedFlows">
    **Alt**: `runtime.tasks.flow` (Singular) gab einen aktiven TaskFlow-
    Zugriff zurück.

    **Neu**: `runtime.tasks.managedFlows` behält die verwaltete TaskFlow-Mutations-
    Runtime für Plugins bei, die untergeordnete Aufgaben aus einem Ablauf erstellen, aktualisieren, abbrechen oder
    ausführen. Verwenden Sie `runtime.tasks.flows`, wenn das Plugin nur DTO-basierte
    Lesezugriffe benötigt.

    ```typescript
    // Vorher
    const flow = api.runtime.tasks.flow.fromToolContext(ctx);
    // Nachher
    const flow = api.runtime.tasks.managedFlows.fromToolContext(ctx);
    ```

    Die veralteten Aliasse wurden im Juli 2026 entfernt.

  </Accordion>

  <Accordion title="Eingebettete Erweiterungs-Factorys -> Middleware für Agent-Tool-Ergebnisse">
    Dies wird oben unter [Migration](#how-to-migrate) behandelt. Der Vollständigkeit
    halber wird es hier ebenfalls aufgeführt: Der entfernte, ausschließlich für
    eingebettete Runner bestimmte Pfad `api.registerEmbeddedExtensionFactory(...)` wird durch
    `api.registerAgentToolResultMiddleware(...)` mit einer expliziten Runtime-Liste
    in `contracts.agentToolResultMiddleware` ersetzt.
  </Accordion>

  <Accordion title="Alias OpenClawSchemaType -> OpenClawConfig">
    Der Root-SDK-Alias `OpenClawSchemaType` wurde entfernt. Verwenden Sie den
    kanonischen Namen `OpenClawConfig`.

    ```typescript
    // Vorher
    import type { OpenClawSchemaType } from "openclaw/plugin-sdk";
    // Nachher
    import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
    ```

  </Accordion>
</AccordionGroup>

<Note>
Veraltete Funktionen auf Erweiterungsebene (innerhalb der gebündelten Kanal-/Provider-Plugins
unter `extensions/`) werden in ihren eigenen Barrels `api.ts` und
`runtime-api.ts` nachverfolgt. Sie wirken sich nicht auf die Verträge von Drittanbieter-Plugins
aus und werden hier nicht aufgeführt. Wenn Sie das lokale Barrel eines gebündelten Plugins
direkt verwenden, lesen Sie vor dem Upgrade die Hinweise zu veralteten Funktionen in diesem Barrel.
</Note>

## Migration von Talk und Echtzeit-Sprachkommunikation

Code für Echtzeit-Sprachkommunikation, Telefonie, Meetings und Browser-Talk verwendet
gemeinsam einen Talk-Sitzungscontroller, der von `openclaw/plugin-sdk/realtime-voice` exportiert wird. Der
Controller verwaltet den gemeinsamen Talk-Ereignisumschlag, den Zustand des aktiven Gesprächsbeitrags,
den Erfassungszustand, den Zustand der Audioausgabe, den jüngsten Ereignisverlauf und die
Ablehnung veralteter Gesprächsbeiträge. Provider-Plugins verwalten anbieterspezifische
Echtzeitsitzungen. Browser-Meeting-Plugins verwenden `openclaw/plugin-sdk/meeting-runtime` für Sitzungs-,
Browser-, Audio-, Node-Host-, Agent-Consult- und Sprachanrufmechanismen und implementieren
anschließend `MeetingPlatformAdapter` für URL-Regeln, DOM-Skripte, die Zuordnung manueller Aktionen,
Untertitel, Erstellung und Einwahlpläne. Plattform-REST-APIs, OAuth, Artefakte, Selektoren und
Wire-Namen verbleiben im Plugin. Browser-Berechtigungspläne erhalten die angeforderte Meeting-URL,
damit jede Plattform ausschließlich ihre genau unterstützten Ursprünge freigeben kann.
Sitzungs-Runtimes müssen außerdem die plattformspezifische Live-Funktionsfähigkeit nach dem
bestätigten Verlassen des Browsers normalisieren; historische Transkriptfelder dürfen erhalten
bleiben, aber die Bereitschaft von Untertiteln und Audio darf nach dem Verlassen nicht aktiv bleiben.

Alle gebündelten Oberflächen werden mit dem gemeinsamen Controller ausgeführt: Browser-Relay,
Übergabe verwalteter Räume, Echtzeit-Sprachanrufe, Streaming-STT für Sprachanrufe, Google
Meet-Echtzeitkommunikation und natives Push-to-Talk. Der Gateway kündigt in
`hello-ok.features.events` einen Live-Talk-Ereigniskanal an: `talk.event`.

Neuer Code sollte `createTalkEventSequencer(...)` nicht direkt aufrufen, außer wenn
ein Low-Level-Adapter oder eine Test-Fixture implementiert wird. Verwenden Sie den gemeinsamen
Controller, damit auf einen Gesprächsbeitrag begrenzte Ereignisse nicht ohne Gesprächsbeitrags-ID
ausgegeben werden können, veraltete Aufrufe von `turnEnd` /
`turnCancel` keinen neueren aktiven Gesprächsbeitrag löschen können und Ereignisse im
Lebenszyklus der Audioausgabe über Telefonie, Meetings, Browser-Relay, die Übergabe verwalteter
Räume und native Talk-Clients hinweg konsistent bleiben.

Die Form der öffentlichen API:

```typescript
// Vom Gateway verwaltete Talk-Sitzungs-API.
await gateway.request("talk.session.create", {
  mode: "realtime",
  transport: "gateway-relay",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.session.appendAudio", { sessionId, audioBase64 });
await gateway.request("talk.session.cancelOutput", { sessionId, reason: "barge-in" });
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "working" },
  options: { willContinue: true },
});
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "already_delivered" },
  options: { suppressResponse: true },
});
await gateway.request("talk.session.submitToolResult", { sessionId, callId, result });
await gateway.request("talk.session.close", { sessionId });

// Vom Client verwaltete Provider-Sitzungs-API.
await gateway.request("talk.client.create", {
  mode: "realtime",
  transport: "webrtc",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.client.toolCall", { sessionKey, callId, name, args });
await gateway.request("talk.client.steer", { sessionKey, text, mode: "steer" });
```

Browsereigene WebRTC-/Provider-WebSocket-Sitzungen verwenden `talk.client.create`,
da der Browser die Provider-Aushandlung und den Medientransport verwaltet, während der
Gateway Anmeldedaten, Anweisungen und Tool-Richtlinien verwaltet. `talk.session.*` ist
die gemeinsame, vom Gateway verwaltete Oberfläche für Gateway-Relay-Echtzeitkommunikation,
Gateway-Relay-Transkription und native STT-/TTS-Sitzungen in verwalteten Räumen.

Veraltete Konfigurationen, die Echtzeitselektoren neben `talk.provider` /
`talk.providers` platzieren, sollten mit `openclaw doctor --fix` repariert werden;
die Talk-Runtime interpretiert die Sprach-/TTS-Provider-Konfiguration nicht als
Echtzeit-Provider-Konfiguration neu.

Die unterstützten Kombinationen von `talk.session.create` sind bewusst begrenzt:

| Modus           | Transport       | Logik              | Verantwortlich     | Hinweise                                                                                                                             |
| --------------- | --------------- | ------------------ | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| `realtime`      | `gateway-relay` | `agent-consult` | Gateway            | Vollduplex-Provider-Audio, das über den Gateway übertragen wird; Tool-Aufrufe werden durch das Agent-Consult-Tool geleitet.           |
| `transcription` | `gateway-relay` | `none`          | Gateway            | Nur Streaming-STT; Aufrufer senden Eingangsaudio und empfangen Transkriptereignisse.                                                  |
| `stt-tts`       | `managed-room`  | `agent-consult` | Nativer Raum/Client | Push-to-Talk- und Walkie-Talkie-artige Räume, in denen der Client Erfassung/Wiedergabe und der Gateway den Gesprächsbeitragszustand verwaltet. |
| `stt-tts`       | `managed-room`  | `direct-tools`  | Nativer Raum/Client | Ausschließlich für Administratoren bestimmter Raummodus für vertrauenswürdige Erstanbieter-Oberflächen, die Gateway-Tool-Aktionen direkt ausführen. |

Methodenzuordnung für Leser, die von den älteren Familien `talk.realtime.*` /
`talk.transcription.*` / `talk.handoff.*` migrieren (alle entfernt):

| Alt                              | Neu                                                      |
| -------------------------------- | -------------------------------------------------------- |
| `talk.realtime.session`          | `talk.client.create`                                     |
| `talk.realtime.toolCall`         | `talk.client.toolCall`                                   |
| `talk.realtime.relayAudio`       | `talk.session.appendAudio`                               |
| `talk.realtime.relayCancel`      | `talk.session.cancelOutput` oder `talk.session.cancelTurn` |
| `talk.realtime.relayToolResult`  | `talk.session.submitToolResult`                          |
| `talk.realtime.relayStop`        | `talk.session.close`                                     |
| `talk.transcription.session`     | `talk.session.create({ mode: "transcription" })`         |
| `talk.transcription.relayAudio`  | `talk.session.appendAudio`                               |
| `talk.transcription.relayCancel` | `talk.session.cancelTurn`                                |
| `talk.transcription.relayStop`   | `talk.session.close`                                     |
| `talk.handoff.create`            | `talk.session.create({ transport: "managed-room" })`     |
| `talk.handoff.join`              | `talk.session.join`                                      |
| `talk.handoff.revoke`            | `talk.session.close`                                     |

Das einheitliche Steuerungsvokabular ist ebenfalls bewusst begrenzt:

| Methode                         | Gilt für                                                | Vertrag                                                                                                                                                                                                                  |
| ------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `talk.session.appendAudio`      | `realtime/gateway-relay`, `transcription/gateway-relay` | Einen Base64-PCM-Audioblock an die Provider-Sitzung anhängen, die derselben Gateway-Verbindung gehört.                                                                                                                    |
| `talk.session.startTurn`        | `stt-tts/managed-room`                                  | Einen Benutzer-Gesprächsbeitrag in einem verwalteten Raum beginnen.                                                                                                                                                       |
| `talk.session.endTurn`          | `stt-tts/managed-room`                                  | Den aktiven Gesprächsbeitrag nach der Validierung auf einen veralteten Gesprächsbeitrag beenden.                                                                                                                          |
| `talk.session.cancelTurn`       | alle vom Gateway verwalteten Sitzungen                  | Aktive Erfassungs-, Provider-, Agent- und TTS-Arbeit für einen Gesprächsbeitrag abbrechen.                                                                                                                                |
| `talk.session.cancelOutput`     | `realtime/gateway-relay`                                | Die Audioausgabe des Assistenten stoppen, ohne den Benutzer-Gesprächsbeitrag zwangsläufig zu beenden.                                                                                                                     |
| `talk.session.submitToolResult` | `realtime/gateway-relay`                                | Einen Provider-Tool-Aufruf nach einer durch seine Bridge bereitgestellten asynchronen Fertigstellung abschließen; `options.willContinue` für eine Zwischenausgabe oder, sofern unterstützt, `options.suppressResponse` übergeben, um eine weitere Assistentenantwort zu vermeiden. |
| `talk.session.steer`            | agentengestützte Talk-Sitzungen                          | Gesprochene Steuerbefehle vom Typ `status`, `steer`, `cancel` oder `followup` an den aktiven eingebetteten Lauf senden, der aus der Talk-Sitzung aufgelöst wurde.                                                                                                 |
| `talk.session.close`            | alle einheitlichen Sitzungen                             | Relay-Sitzungen stoppen oder den Zustand verwalteter Räume widerrufen und anschließend die einheitliche Sitzungs-ID verwerfen.                                                                                           |

Führen Sie keine Provider- oder Plattformsonderfälle im Kern ein, damit dies
funktioniert. Der Kern verwaltet die Semantik von Talk-Sitzungen. Provider-Plugins
verwalten die Einrichtung anbieterspezifischer Sitzungen. Voice-Call und Google Meet
verwalten Telefonie-/Meeting-Adapter. Browser- und native Apps verwalten die
Geräteerfassungs-/Wiedergabe-UX.

## Zeitplan für die Entfernung

| Wann                                        | Was geschieht                                                                                                                              |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Jetzt**                                     | Veraltete Oberflächen, die Warnungen unterstützen, geben Laufzeitwarnungen aus; Repository-Prüfungen weisen veraltete SDK-Importe aus dem Kern und gebündelten Plugins zurück. |
| **Ausstehende Entscheidung des Verantwortlichen**                  | Datumslose Einträge bleiben veraltet und können nicht entfernt werden, bis ihr Verantwortlicher ein `removeAfter`-Datum veröffentlicht.                          |
| **Das `removeAfter`-Datum jedes Kompatibilitätseintrags** | Diese spezifische Oberfläche kann entfernt werden; nach Ablauf des Datums lässt `pnpm plugins:boundary-report --fail-on-eligible-compat` die CI fehlschlagen.    |
| **Nächste Hauptversion**                      | Datierte Oberflächen dürfen erst nach ihrem `removeAfter`-Datum entfernt werden; datumslose Einträge erfordern weiterhin die Genehmigung des Verantwortlichen und ein veröffentlichtes Datum.   |

Für die nachfolgend verbleibenden öffentlichen SDK-Unterpfade gelten registrierungsgestützte Entfernungszeiträume.
Die Zeilen vom 30. Juli wurden nach ihrer frühzeitigen, von den Maintainern genehmigten Bereinigung entfernt:
Nicht verwendete Unterpfade wurden gelöscht, frühere Kompatibilitätsaliase wurden gelöscht und
ausschließlich für gebündelte Plugins bestimmte Module wurden zu privaten lokalen Build-Zuordnungen herabgestuft.

| `removeAfter` | Stufe                               | SDK-Unterpfade                                                                                                                                                                        |
| ------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `2026-08-15`  | Frühere Kompatibilitätsveraltungen | `agent-config-primitives`, `channel-logging`, `channel-secret-runtime`, `channel-streaming`, `group-access`, `inbound-reply-dispatch`, `matrix`, `text-runtime`, `zod`              |
| `2026-09-01`  | Frühere Kompatibilitätsveraltungen | `channel-lifecycle`, `channel-message`, `channel-reply-pipeline`, `config-runtime`, `infra-runtime`                                                                                 |
| `2026-10-01`  | Legacy-Projektion für Medien            | `agent-media-payload` sowie die nicht zu Unterpfaden gehörenden `MsgContext Media*`-Felder, Builder für eingehende Medien-Payloads von Kanälen, `buildMediaPayload`, Hook-Medienaliase und `{{Media*}}`-Vorlagen |

Alle Kern-Plugins wurden bereits migriert. Externe Plugins sollten
vor der nächsten Hauptversion migriert werden. Führen Sie `pnpm plugins:boundary-report` aus, um zu sehen, welche
Kompatibilitätseinträge für die von Ihrem Plugin verwendeten Oberflächen am frühesten fällig sind.

## Warnungen vorübergehend unterdrücken

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Dies ist ein vorübergehender Ausweg, keine dauerhafte Lösung.

## Verwandte Themen

- [Erste Schritte](/de/plugins/building-plugins) - Erstellen Sie Ihr erstes Plugin
- [SDK-Übersicht](/de/plugins/sdk-overview) - vollständige Importreferenz für Unterpfade
- [Kanal-Plugins](/de/plugins/sdk-channel-plugins) - Kanal-Plugins erstellen
- [Provider-Plugins](/de/plugins/sdk-provider-plugins) - Provider-Plugins erstellen
- [Plugin-Interna](/de/plugins/architecture) - ausführlicher Einblick in die Architektur
- [Plugin-Manifest](/de/plugins/manifest) - Referenz zum Manifest-Schema
