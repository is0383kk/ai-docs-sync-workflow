---
read_when:
    - Sie entwickeln ein lokales Backend-Plugin für eine KI-CLI
    - Sie möchten ein Backend für Modellreferenzen wie acme-cli/model registrieren.
    - Sie müssen eine Drittanbieter-CLI dem Text-Fallback-Runner von OpenClaw zuordnen
sidebarTitle: CLI backend plugins
summary: Erstellen Sie ein Plugin, das ein lokales AI-CLI-Backend registriert
title: CLI-Backend-Plugins erstellen
x-i18n:
    generated_at: "2026-07-26T18:36:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1923b0829b46a309e4b5a6cbbbfd3dcb76a1e14fe4106310d7a9fb37bca41d70
    source_path: plugins/cli-backend-plugins.md
    workflow: 16
---

CLI-Backend-Plugins ermöglichen OpenClaw, eine lokale KI-CLI als Backend für
Textinferenz aufzurufen. Das Backend erscheint als Provider-Präfix in Modellreferenzen:

```text
acme-cli/acme-large
```

Verwenden Sie ein CLI-Backend, wenn die vorgelagerte Integration bereits als lokaler
Befehl verfügbar ist, wenn die CLI den lokalen Anmeldestatus verwaltet oder als Ausweichlösung, wenn API-
Provider nicht verfügbar sind.

<Info>
  Wenn der vorgelagerte Dienst eine normale HTTP-Modell-API bereitstellt, schreiben Sie stattdessen ein
  [Provider-Plugin](/de/plugins/sdk-provider-plugins). Wenn die vorgelagerte
  Laufzeit vollständige Agentensitzungen, Tool-Ereignisse, Compaction oder den Status von Hintergrundaufgaben
  verwaltet, verwenden Sie ein [Agent-Harness](/de/plugins/sdk-agent-harness).
</Info>

## Verantwortungsbereich des Plugins

Ein CLI-Backend-Plugin hat drei Verträge:

| Vertrag              | Datei                  | Zweck                                                     |
| -------------------- | ---------------------- | --------------------------------------------------------- |
| Paketeinstiegspunkt  | `package.json`         | Verweist OpenClaw auf das Laufzeitmodul des Plugins       |
| Manifest-Zuständigkeit | `openclaw.plugin.json` | Deklariert die Backend-ID vor dem Laden der Laufzeit      |
| Laufzeitregistrierung | `index.ts`             | Ruft `api.registerCliBackend(...)` mit Befehlsstandardwerten auf |

Das Manifest enthält Metadaten für die Erkennung: Es führt die CLI nicht aus und registriert
kein Laufzeitverhalten. Das Laufzeitverhalten beginnt, wenn der Plugin-Einstiegspunkt
`api.registerCliBackend(...)` aufruft.

## Minimales Backend-Plugin

<Steps>
  <Step title="Paketmetadaten erstellen">
    ```json package.json
    {
      "name": "@acme/openclaw-acme-cli",
      "version": "1.0.0",
      "type": "module",
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
      },
      "dependencies": {
        "openclaw": "^2026.3.24"
      },
      "devDependencies": {
        "typescript": "^5.9.0"
      }
    }
    ```

    Veröffentlichte Pakete müssen erstellte JavaScript-Laufzeitdateien enthalten. Wenn Ihr Quell-
    Einstiegspunkt `./src/index.ts` ist, fügen Sie `openclaw.runtimeExtensions` hinzu, das auf die
    erstellte JavaScript-Entsprechung verweist. Siehe [Einstiegspunkte](/de/plugins/sdk-entrypoints).

  </Step>

  <Step title="Backend-Zuständigkeit deklarieren">
    ```json openclaw.plugin.json
    {
      "id": "acme-cli",
      "name": "Acme CLI",
      "description": "Acmes lokale KI-CLI über OpenClaw ausführen",
      "cliBackends": ["acme-cli"],
      "setup": {
        "cliBackends": ["acme-cli"],
        "requiresRuntime": false
      },
      "activation": {
        "onStartup": false
      },
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```

    `cliBackends` ist die Liste der Laufzeitzuständigkeiten; dadurch kann OpenClaw das
    Plugin automatisch laden, wenn die Modellauswahl oder `agentRuntime.id` `acme-cli` erwähnt.

    `setup.cliBackends` ist die deskriptorbasierte Einrichtungsoberfläche. Fügen Sie sie hinzu, wenn
    Modellerkennung, Onboarding oder Status das Backend erkennen sollen,
    ohne die Plugin-Laufzeit zu laden. Verwenden Sie `requiresRuntime: false` nur, wenn
    diese statischen Deskriptoren für die Einrichtung ausreichen.

  </Step>

  <Step title="Backend registrieren">
    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      CLI_FRESH_WATCHDOG_DEFAULTS,
      CLI_RESUME_WATCHDOG_DEFAULTS,
      type CliBackendPlugin,
    } from "openclaw/plugin-sdk/cli-backend";

    function buildAcmeCliBackend(): CliBackendPlugin {
      return {
        id: "acme-cli",
        liveTest: {
          defaultModelRef: "acme-cli/acme-large",
          defaultImageProbe: false,
          defaultMcpProbe: false,
          docker: {
            npmPackage: "@acme/acme-cli",
            binaryName: "acme",
          },
        },
        config: {
          command: "acme",
          args: ["chat", "--output-format", "stream-json", "--prompt", "{prompt}"],
          resumeArgs: [
            "chat",
            "--resume",
            "{sessionId}",
            "--output-format",
            "stream-json",
            "--prompt",
            "{prompt}",
          ],
          output: "jsonl",
          resumeOutput: "jsonl",
          jsonlDialect: "gemini-stream-json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            large: "acme-large-2026",
            fast: "acme-fast-2026",
          },
          sessionArgs: ["--session", "{sessionId}"],
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptFileArg: "--system-file",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          imagePathScope: "workspace",
          reliability: {
            watchdog: {
              fresh: { ...CLI_FRESH_WATCHDOG_DEFAULTS },
              resume: { ...CLI_RESUME_WATCHDOG_DEFAULTS },
            },
          },
          serialize: true,
        },
      };
    }

    export default definePluginEntry({
      id: "acme-cli",
      name: "Acme CLI",
      description: "Acmes lokale KI-CLI über OpenClaw ausführen",
      register(api) {
        api.registerCliBackend(buildAcmeCliBackend());
      },
    });
    ```

    Die Backend-ID muss mit dem Manifest-Eintrag `cliBackends` übereinstimmen. Der registrierte
    Adapter ist maßgeblicher Plugin-Code; die OpenClaw-Konfiguration wählt das Backend aus,
    schreibt dessen Befehlsvertrag jedoch nicht um.

  </Step>
</Steps>

## Konfigurationsstruktur

`CliBackendConfig` beschreibt, wie OpenClaw die CLI starten und analysieren soll. Das
ausgearbeitete Beispiel oben verwendet absichtlich dieselben Befehls-, Fortsetzungs-, JSONL-,
Modellalias-, Sitzungs-, Bild- und Watchdog-Felder wie der mitgelieferte
Adapter `google-gemini-cli`:

| Feld                                                      | Verwendung                                                                         |
| --------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `command`                                                 | Binärdateiname oder absoluter Befehlspfad                                          |
| `args`                                                    | Basis-argv für neue Ausführungen                                                   |
| `resumeArgs`                                              | Alternatives argv für fortgesetzte Sitzungen; unterstützt `{sessionId}`           |
| `output` / `resumeOutput`                                 | Parser: `json`, `jsonl` oder `text`                                               |
| `jsonlDialect`                                            | JSONL-Ereignisdialekt: `claude-stream-json` oder `gemini-stream-json`             |
| `liveSession`                                             | Modus für langlebige CLI-Prozesse (`claude-stdio`)                                 |
| `input`                                                   | Prompt-Übertragung: `arg` oder `stdin`                                          |
| `maxPromptArgChars`                                       | Maximale Prompt-Länge im Modus `arg` vor dem Ausweichen auf stdin             |
| `env` / `clearEnv`                                        | Zusätzliche zu injizierende Umgebungsvariablen oder vor dem Start zu entfernende Namen |
| `modelArg`                                                | Vor der Modell-ID verwendetes Flag                                                |
| `modelAliases`                                            | Ordnet OpenClaw-Modell-IDs CLI-nativen IDs zu                                    |
| `sessionArgs`                                             | Übergabe einer Sitzungs-ID mithilfe von `{sessionId}`                             |
| `sessionMode`                                             | `always`, `existing` oder `none`                                                  |
| `sessionIdFields`                                         | JSON-Felder, die OpenClaw aus der CLI-Ausgabe liest                               |
| `systemPromptArg` / `systemPromptFileArg`                 | Übertragung des System-Prompts                                                    |
| `systemPromptFileConfigArg` / `systemPromptFileConfigKey` | Übertragung von Konfigurationsüberschreibungen für eine System-Prompt-Datei (zum Beispiel `-c`) |
| `systemPromptMode`                                        | `append` oder `replace`                                                            |
| `systemPromptWhen`                                        | `first`, `always` oder `never`                                                    |
| `imageArg` / `imageMode`                                  | Bildpfad-Flag und Übergabe mehrerer Bilder (`repeat` oder `list`)  |
| `imagePathScope`                                          | Speicherort bereitgestellter Bilddateien vor der Übergabe: `temp` oder `workspace` |
| `serialize`                                               | Ausführungen desselben Backends geordnet halten                                   |
| `reseedFromRawTranscriptWhenUncompacted`                  | Aktiviert vor Compaction eine begrenzte Neuinitialisierung aus dem Rohtranskript für sichere Sitzungszurücksetzungen |
| `reliability.watchdog`                                    | Abstimmung des Zeitlimits ohne Ausgabe, getrennt für neue und fortgesetzte Ausführungen |

Bevorzugen Sie die kleinste statische Konfiguration, die zur CLI passt. Fügen Sie Plugin-Callbacks
nur für Verhalten hinzu, das tatsächlich zum Backend gehört.

## Erweiterte Backend-Hooks

`CliBackendPlugin` kann außerdem Folgendes definieren:

| Hook                               | Verwendung                                                                 |
| ---------------------------------- | -------------------------------------------------------------------------- |
| `normalizeConfig(config, context)` | Registrierten statischen Adapter mit Laufzeitkontext normalisieren         |
| `resolveExecutionArgs(ctx)`        | Anfragespezifische Flags wie Denkaufwand oder Isolierung von Nebenfragen hinzufügen |
| `prepareExecution(ctx)`            | Vor dem Start temporäre Authentifizierungs-, Konfigurations- oder Umgebungsbrücken erstellen |
| `transformSystemPrompt(ctx)`       | Eine abschließende CLI-spezifische Transformation des System-Prompts anwenden |
| `textTransforms`                   | Bidirektionale Ersetzungen für Prompt und Ausgabe                          |
| `defaultAuthProfileId`             | Ein bestimmtes OpenClaw-Authentifizierungsprofil bevorzugen                |
| `authEpochMode`                    | Festlegen, wie Authentifizierungsänderungen gespeicherte CLI-Sitzungen ungültig machen |
| `nativeToolMode`                   | Deklarieren, ob native Tools fehlen, immer aktiviert oder vom Host auswählbar sind |
| `toolAvailabilityEnforcement`      | Deklarieren, ob genaue Tool-Begrenzungen in argv oder bei der Ausführungsbereitstellung erzwungen werden |
| `sideQuestionToolMode`             | Deaktivierte native Tools für `/btw`-Nebenfragen deklarieren               |
| `bundleMcp` / `bundleMcpMode`      | OpenClaws Loopback-MCP-Tool-Brücke aktivieren                              |
| `ownsNativeCompaction`             | Das Backend verwaltet seine eigene Compaction – OpenClaw stellt sie zurück |
| `subscriptionAuthDispatch`         | Aktivierte eingebettete Ausführungen mit Abonnement-Anmeldedaten werden über dieses Backend ausgeführt |
| `runtimeArtifact`                  | Einen Skriptstarter an seinen vollständigen mitgelieferten Paketbaum binden |

Belassen Sie diese Hooks in der Zuständigkeit des Providers. Fügen Sie dem Kern keine
CLI-spezifischen Verzweigungen hinzu, wenn ein Backend-Hook das Verhalten ausdrücken kann.

`prepareExecution(ctx)` empfängt `ctx.contextTokenBudget`, das für den Lauf ausgewählte effektive Token-
Limit. Backends mit eigener nativer Compaction können dieses
Budget ihrem CLI-spezifischen Startvertrag zuordnen.

`runtimeArtifact` gehört dem Plugin. Es wird
nur herangezogen, wenn ein Live-Inferenzdurchlauf verifizierte Einrichtungsautorität erzeugt oder erneut validiert;
normale CLI-Läufe benötigen es nicht. Ein Backend ohne diese Deklaration kann
keine verifizierte CLI-Einrichtungsautorität erzeugen. Eine `bundled-package-tree`-Deklaration benennt
den exakten `package.json`-Eigentümer und verlangt, dass der Paket-Einstiegspunkt der
Befehl ist. OpenClaw hasht den begrenzten, vollständigen installierten Paketbaum einschließlich
verschachtelter Abhängigkeiten und bricht bei umleitenden Symlinks,
Startern außerhalb des deklarierten Pakets, Deklarationen erforderlicher externer Abhängigkeiten,
übergroßen Bäumen und unbekannten Skripten sicher ab. Deklarieren Sie dies nur, wenn dieser
Baum die vollständige Inferenzimplementierung enthält; optionale Tool-Integrationen
machen einen externen Implementierungsgraphen nicht sicher.

Wenn dasselbe Backend auch eine eigenständige native ausführbare Datei bereitstellt, führen Sie deren
kanonische Basisnamen in `nativeExecutableNames` auf. Andere native Befehle bleiben
unverifiziert.

`ctx.executionMode` ist `"agent"` für normale Durchläufe und `"side-question"` für
flüchtige `/btw`-Aufrufe. Verwenden Sie es, wenn die CLI andere Einmal-Flags benötigt,
etwa zum Deaktivieren nativer Tools, der Sitzungspersistenz oder des Fortsetzungsverhaltens für
BTW. Wenn ein Backend normalerweise `nativeToolMode: "always-on"` hat, seine
Argumentliste für Nebenfragen diese Tools jedoch zuverlässig deaktiviert, setzen Sie außerdem
`sideQuestionToolMode: "disabled"`; andernfalls bricht OpenClaw sicher ab, wenn BTW
einen CLI-Lauf ohne Tools erfordert.

Setzen Sie `nativeToolMode: "selectable"` nur, wenn das Backend jedes
backend-native Tool für einen einzelnen Lauf deaktivieren kann. Eingeschränkte Läufe erhalten einen kanonischen
Vertrag: `ctx.toolAvailability.native` ist die exakte backend-native Liste und
`ctx.toolAvailability.openClaw` ist die exakte Liste der OpenClaw-Toolnamen. Der
Host beschränkt die erzeugte MCP-Konfiguration und Berechtigung unabhängig auf diese
OpenClaw-Liste; Plugins dürfen sie im Core weder übersetzen noch Transportpräfixe hinzufügen.

Deklarieren Sie, wie das Backend diesen Vertrag durchsetzt:

- `toolAvailabilityEnforcement: "execution-args"` erfordert
  `resolveExecutionArgs`. Der Hook muss widersprüchliche Tool-Flags ersetzen, Anpassungsoberflächen
  deaktivieren, die außerhalb der ausgewählten Tools Code ausführen können, und
  durchsetzende Argumente sowohl für neue als auch für fortgesetzte Läufe zurückgeben.
- `toolAvailabilityEnforcement: "prepare-execution"` erfordert
  `prepareExecution`. Der Hook muss eine exakte Richtlinie pro Lauf bereitstellen und
  `toolAvailabilityEnforced: true` zurückgeben; eine fehlende Bestätigung führt zum sicheren Abbruch, und
  OpenClaw bereinigt die bereitgestellten Ressourcen vor dem Start.

Laufzeitbegrenzungen wie Cron-`toolsAllow` werden von
OpenClaw normalisiert und nach Gruppen erweitert, bevor dieser Vertrag erstellt wird. Native Tools werden deaktiviert, und ein
Backend ohne vollständig deklarierten Durchsetzungspfad schlägt vor der Ausführung fehl.

Plugins, die gegen `v2026.7.2-beta.1` bis `v2026.7.2-beta.3` erstellt wurden, können weiterhin
die veraltete Projektion des Transportnamens `ctx.toolAvailability.mcp` lesen und
dürfen `toolAvailabilityEnforcement` auslassen, wenn ein auswählbares Backend
`resolveExecutionArgs` implementiert. OpenClaw erkennt diesen veröffentlichten Beta-Pfad anhand der
erforderlichen `openclaw.build.openclawVersion`-Metadaten des Plugin-Pakets und
behält ihn bis einschließlich der `2026.8.x`-Linie bei. Neue und aktualisierte Plugins sollten kanonische
`ctx.toolAvailability.openClaw`-Namen verwenden und
`toolAvailabilityEnforcement: "execution-args"` ausdrücklich deklarieren; der Beta-
Kompatibilitätspfad soll nach diesem Zeitraum entfernt werden.

### `ownsNativeCompaction`: OpenClaw-Compaction deaktivieren

Wenn Ihr Backend einen Agenten ausführt, der sein **eigenes** Transkript komprimiert, setzen Sie
`ownsNativeCompaction: true`, damit OpenClaws absichernde Zusammenfassung niemals
auf dessen Sitzungen angewendet wird – der CLI-Compaction-Lebenszyklus führt keine Aktion aus und der
Durchlauf wird fortgesetzt. `claude-cli` deklariert dies, weil Claude Code
intern ohne Harness-Endpunkt komprimiert. Native Harness-Sitzungen wie Codex
werden stattdessen weiterhin an ihren Harness-Compaction-Endpunkt weitergeleitet.

**Deklarieren Sie dies nur, wenn alle folgenden Bedingungen erfüllt sind**, andernfalls
kann eine verzögerte Sitzung, die das Budget überschreitet, weiterhin über dem Budget bleiben oder veralten (OpenClaw
rettet sie nicht mehr):

- das Backend komprimiert oder begrenzt sein eigenes Transkript zuverlässig, wenn es sich seinem
  Fenster nähert;
- es speichert eine fortsetzbare Sitzung, sodass der komprimierte Zustand über mehrere Durchläufe hinweg erhalten bleibt
  (zum Beispiel `--resume` / `--session-id`);
- es handelt sich nicht um eine native Harness-Compaction-Sitzung – übereinstimmende `agentHarnessId`-
  Sitzungen werden stattdessen an den Harness-Endpunkt weitergeleitet.

## MCP-Tool-Bridge

CLI-Backends erhalten standardmäßig keine OpenClaw-Tools. Wenn die CLI eine
MCP-Konfiguration verwenden kann, aktivieren Sie dies ausdrücklich:

```typescript
return {
  id: "acme-cli",
  bundleMcp: true,
  bundleMcpMode: "codex-config-overrides",
  config: {
    command: "acme",
    args: ["chat", "--json"],
    output: "json",
  },
};
```

Unterstützte Bridge-Modi:

| Modus                     | Verwendung                                                              |
| ------------------------ | ---------------------------------------------------------------- |
| `claude-config-file`     | CLIs, die eine MCP-Konfigurationsdatei akzeptieren                              |
| `codex-config-overrides` | CLIs, die Konfigurationsüberschreibungen in der Argumentliste akzeptieren                        |
| `gemini-system-settings` | CLIs, die MCP-Einstellungen aus ihrem Systemeinstellungsverzeichnis lesen |

Aktivieren Sie die Bridge nur, wenn die CLI sie tatsächlich verwenden kann. Wenn die CLI
eine eigene integrierte Tool-Schicht hat, die nicht deaktiviert werden kann, setzen Sie `nativeToolMode:
"always-on"`, damit OpenClaw sicher abbrechen kann, wenn ein Aufrufer keine nativen
Tools verlangt. Wenn sie jedes native Tool pro Lauf deaktivieren kann, verwenden Sie `"selectable"` mit dem
oben beschriebenen `resolveExecutionArgs`-Vertrag.

## Backend auswählen

Benutzer wählen ein eigenständiges Backend über dessen Modellreferenz-Präfix aus. Ein Backend, das
ein kanonisches `modelProvider` deklariert, kann stattdessen über das
`agentRuntime.id` dieses Provider-Modells ausgewählt werden. Die Adaptermechanik verbleibt im Plugin:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "openai/gpt-5.6-sol",
        fallbacks: ["acme-cli/large"],
      },
    },
  },
}
```

Speichern Sie Anmeldedaten in OpenClaw-Authentifizierungsprofilen oder in Plugin-eigener Konfiguration. Stellen Sie sicher, dass der
registrierte Befehl im `PATH` des Gateway-Dienstes verfügbar ist; Bereitstellungen, die einen
anderen Pfad oder eine andere Argumentliste benötigen, sollten die Plugin-Registrierung ändern oder umschließen.

## Verifizierung

Fügen Sie für gebündelte Plugins einen fokussierten Test für den Builder und die Einrichtungsregistrierung
hinzu und führen Sie anschließend die gezielte Testspur des Plugins aus:

```bash
pnpm test extensions/acme-cli
```

Verifizieren Sie bei lokalen oder installierten Plugins die Erkennung und einen echten Modelllauf:

```bash
openclaw plugins inspect acme-cli --runtime --json
openclaw agent --message "antworte exakt: Backend ok" --model acme-cli/acme-large
```

Wenn das Backend Bilder oder MCP unterstützt, fügen Sie einen Live-Smoke-Test hinzu, der diese
Pfade mit der echten CLI nachweist. Verlassen Sie sich für Prompt-, Bild-,
MCP- oder Sitzungsfortsetzungsverhalten nicht auf statische Prüfung.

## Checkliste

<Check>`package.json` enthält `openclaw.extensions` und erstellte Laufzeiteinträge für veröffentlichte Pakete</Check>
<Check>`openclaw.plugin.json` deklariert `cliBackends` und bewusst gewählte `activation.onStartup`</Check>
<Check>`setup.cliBackends` ist vorhanden, wenn Einrichtung oder Modellerkennung das Backend im kalten Zustand erkennen sollen</Check>
<Check>`api.registerCliBackend(...)` verwendet dieselbe Backend-ID wie das Manifest</Check>
<Check>Das Backend-Modellpräfix oder das modellbezogene `agentRuntime.id` wählt die Registrierung aus</Check>
<Check>Einstellungen für Sitzung, System-Prompt, Bilder und Ausgabeparser entsprechen dem tatsächlichen CLI-Vertrag</Check>
<Check>Gezielte Tests und mindestens ein Live-CLI-Smoke-Test weisen den Backend-Pfad nach</Check>

## Verwandte Themen

- [CLI-Backends](/de/gateway/cli-backends) – Laufzeitauswahl und Verhalten
- [Plugins erstellen](/de/plugins/building-plugins) – Grundlagen zu Paket und Manifest
- [Überblick über das Plugin SDK](/de/plugins/sdk-overview) – Referenz zur Registrierungs-API
- [Plugin-Manifest](/de/plugins/manifest) – `cliBackends` und Einrichtungsdeskriptoren
- [Agent-Harness](/de/plugins/sdk-agent-harness) – vollständige externe Agentenlaufzeiten
