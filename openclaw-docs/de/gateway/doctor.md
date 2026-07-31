---
read_when:
    - Doctor-Migrationen hinzufügen oder ändern
    - Einführung inkompatibler Konfigurationsänderungen
sidebarTitle: Doctor
summary: 'Doctor-Befehl: Integritätsprüfungen, Konfigurationsmigrationen und Reparaturschritte'
title: Diagnosewerkzeug
x-i18n:
    generated_at: "2026-07-26T18:57:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f599553a2455759cd0fe56bafbc16948f7ab4d381d344b08a496bf19c9dc636
    source_path: gateway/doctor.md
    workflow: 16
---

`openclaw doctor` ist das Reparatur- und Migrationstool für OpenClaw. Es behebt veraltete Konfigurationen und Zustände, prüft den Systemzustand und stellt umsetzbare Reparaturschritte bereit.

## Schnellstart

```bash
openclaw doctor
```

### Headless- und Automatisierungsmodi

<Tabs>
  <Tab title="--yes">
    ```bash
    openclaw doctor --yes
    ```

    Standardwerte ohne Rückfragen akzeptieren (gegebenenfalls einschließlich Reparaturschritten für Neustart, Dienst und Sandbox).

  </Tab>
  <Tab title="--fix">
    ```bash
    openclaw doctor --fix
    ```

    Empfohlene Reparaturen ohne Rückfragen anwenden (`--repair` ist ein Alias).

  </Tab>
  <Tab title="--lint">
    ```bash
    openclaw doctor --lint
    openclaw doctor --lint --json
    ```

    Strukturierte Systemzustandsprüfungen für CI oder Preflight-Automatisierung ausführen. Schreibgeschützt: keine
    Rückfragen, Reparaturen, Migrationen, Neustarts oder Schreibvorgänge am Zustand.

  </Tab>
  <Tab title="--fix --force">
    ```bash
    openclaw doctor --fix --force
    ```

    Auch aggressive Reparaturen anwenden (überschreibt benutzerdefinierte Supervisor-Konfigurationen).

  </Tab>
  <Tab title="--non-interactive">
    ```bash
    openclaw doctor --non-interactive
    ```

    Ohne Rückfragen ausführen und nur sichere Migrationen anwenden (Konfigurationsnormalisierung +
    Verschieben von Zuständen auf dem Datenträger). Überspringt Neustart-, Dienst- und Sandbox-Aktionen, die eine
    manuelle Bestätigung erfordern. Migrationen veralteter Zustände werden bei Erkennung weiterhin automatisch ausgeführt.

  </Tab>
  <Tab title="--deep">
    ```bash
    openclaw doctor --deep
    ```

    Systemdienste nach zusätzlichen Gateway-Installationen durchsuchen (launchd/systemd/schtasks).

  </Tab>
</Tabs>

Um Änderungen vor dem Schreiben zu prüfen, öffnen Sie zunächst die Konfigurationsdatei:

```bash
cat ~/.openclaw/openclaw.json
```

## Schreibgeschützter Lint-Modus

`openclaw doctor --lint` ist das automatisierungsfreundliche Gegenstück zu
`openclaw doctor --fix`. Beide verwenden dieselbe Doctor-Regelregistrierung, wählen
Regeln jedoch nicht auf dieselbe Weise aus und führen sie nicht auf dieselbe Weise aus:

| Modus                    | Rückfragen | Schreibt Konfiguration/Zustand | Ausgabe                         | Verwendung für                         |
| ------------------------ | ---------- | ------------------------------ | -------------------------------- | -------------------------------------- |
| `openclaw doctor`       | ja         | nein                           | verständlicher Zustandsbericht   | manuelle Statusprüfung                 |
| `openclaw doctor --fix`       | manchmal   | ja, gemäß Reparaturrichtlinie  | verständliches Reparaturprotokoll | Anwendung genehmigter Reparaturen      |
| `openclaw doctor --lint`       | nein       | nein                           | strukturierte Befunde             | CI-, Preflight- und Review-Prüfungen   |

Der standardmäßige Lauf von `doctor --lint` verwendet das umfassende und sichere Automatisierungsprofil: Prüfungen, die
statisch und lokal sowie für CI- oder Preflight-Ausgaben nützlich sind. Opt-in-Prüfungen werden übersprungen, wenn sie
nur empfehlenden Charakter haben, von der Umgebung abhängen, von einem aktiven Dienst abhängen, den Konto-/Workspace-
Bestand betreffen oder historische Bereinigungen durchführen. Verwenden Sie `doctor --lint --all`, wenn Sie das
vollständige registrierte Lint-Audit einschließlich dieser Opt-in-Prüfungen wünschen, oder `--only <id>` für
eine gezielte Prüfung.

`doctor --fix` verwendet nicht das standardmäßige Lint-Profil und akzeptiert
`--all` nicht. Es führt den geordneten Reparaturpfad von Doctor aus: Moderne Systemzustandsprüfungen können
eine optionale `repair()`-Implementierung bereitstellen, während ältere Bereiche weiterhin ihren bisherigen
Doctor-Reparaturablauf verwenden. Einige Lint-Befunde dienen absichtlich nur der Diagnose. Daher bedeutet eine
in `--lint --all` enthaltene Prüfung nicht, dass `--fix` diesen Bereich verändert.
Der Vertrag trennt `detect()` (meldet Befunde) von `repair()` (meldet
Änderungen/Diffs/Nebenwirkungen). Dadurch bleibt ein Pfad für ein zukünftiges
`doctor --fix --dry-run` offen, ohne Lint-Prüfungen in Änderungsplaner umzuwandeln.

Einige integrierte Prüfungen sind intern standardmäßig deaktiviert, damit sie für
`--all`, `--only` und Doctor-Reparaturabläufe verfügbar bleiben, ohne Teil des standardmäßigen
`doctor --lint`-Automatisierungsprofils zu werden. Der Schweregrad wird weiterhin für jeden
Befund ausgegeben (`info`, `warning` oder `error`); die Standardauswahl ist keine
Schweregradstufe.

```bash
openclaw doctor --lint
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --json
openclaw doctor --lint --all
openclaw doctor --lint --only core/doctor/gateway-config --json
```

JSON-Ausgabefelder:

- `ok`: ob ein Befund den ausgewählten Schweregradschwellenwert erreicht hat
- `checksRun` / `checksSkipped`: Anzahlen (übersprungen aufgrund des Profils, `--only` oder `--skip`)
- `findings`: strukturierte Diagnosen mit `checkId`, `severity`, `message` und optional `path`, `line`, `column`, `ocPath`, `source`, `target`, `requirement`, `fixHint`

Exit-Codes:

| Code | Bedeutung                                                        |
| ---- | ---------------------------------------------------------------- |
| `0` | keine Befunde auf oder über dem ausgewählten Schwellenwert        |
| `1` | mindestens ein Befund hat den ausgewählten Schwellenwert erreicht |
| `2` | Befehls-/Laufzeitfehler, bevor Befunde ausgegeben werden konnten   |

Flags:

- `--severity-min info|warning|error` (Standardwert `warning`): steuert sowohl die Ausgabe als auch, was einen Exit-Code ungleich null verursacht.
- `--all`: führt jede registrierte Lint-Prüfung aus, einschließlich Opt-in-Prüfungen, die nicht in der standardmäßigen Automatisierungsgruppe enthalten sind.
- `--only <id>` (wiederholbar): nur die benannten Prüfungs-IDs ausführen; eine unbekannte ID wird als Fehlerbefund gemeldet.
- `--skip <id>` (wiederholbar): eine Prüfung ausschließen, während der restliche Lauf aktiv bleibt.
- `--json`, `--severity-min`, `--all`, `--only` und `--skip` erfordern `--lint`; einfache Läufe von `openclaw doctor` und `--fix` lehnen sie ab.

## Funktionsübersicht

<AccordionGroup>
  <Accordion title="Systemzustand, Benutzeroberfläche und Updates">
    - Optionale Preflight-Aktualisierung für Git-Installationen (nur interaktiv).
    - Prüfung der Aktualität des UI-Protokolls (erstellt die Control UI neu, wenn das Protokollschema neuer ist).
    - Systemzustandsprüfung + Aufforderung zum Neustart.
    - Nur problembezogene Hinweise zu Skills und Plugins; der fehlerfreie Bestand verbleibt in `openclaw skills check` und `openclaw plugins list`.

  </Accordion>
  <Accordion title="Konfiguration und Migrationen">
    - Konfigurationsnormalisierung für veraltete Wertstrukturen.
    - Migration der Talk-Konfiguration von veralteten flachen `talk.*`-Feldern zu `talk.provider` + `talk.providers.<provider>`.
    - Browser-Migrationsprüfungen für veraltete Chrome-Erweiterungskonfigurationen und die Bereitschaft von Chrome MCP.
    - Warnungen zu Provider-Überschreibungen für OpenCode (`models.providers.opencode` / `opencode-zen` / `opencode-go`).
    - Migration veralteter OpenAI-Codex-Provider/-Profile (`openai-codex` → `openai`) und Warnungen vor Überschattung durch veraltete `models.providers.openai-codex`.
    - Prüfung der OAuth-TLS-Voraussetzungen für OpenAI-Codex-OAuth-Profile.
    - Warnungen zur Plugin-/Tool-Zulassungsliste, wenn `plugins.allow` restriktiv ist, die Tool-Richtlinie aber weiterhin Platzhalter oder Plugin-eigene Tools anfordert.
    - Migration veralteter Zustände auf dem Datenträger (Sitzungen/Agentenverzeichnis/WhatsApp-Authentifizierung).
    - Migration veralteter Vertragsschlüssel im Plugin-Manifest (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
    - Migration des veralteten Cron-Speichers (`jobId`, `schedule.cron`, Zustellungs-/Nutzlastfelder auf oberster Ebene, Nutzlast `provider`, `notify: true`-Webhook-Fallback-Aufträge).
    - Reparatur der Codex-CLI-Laufzeitfixierung (`agentRuntime.id: "codex-cli"` → `"codex"`) in `agents.defaults`, `agents.entries.*` und `models.providers.*` (einschließlich modellspezifischer Einträge).
    - Bereinigung veralteter Plugin-Konfigurationen, wenn Plugins aktiviert sind; bei `plugins.enabled=false` bleiben veraltete Plugin-Referenzen als inaktive Eindämmungskonfiguration erhalten.

  </Accordion>
  <Accordion title="Zustand und Integrität">
    - Prüfung von Sitzungssperrdateien und Bereinigung veralteter Sperren.
    - Reparatur von Sitzungsprotokollen mit duplizierten Prompt-Umschreibungszweigen, die von betroffenen Builds der Version 2026.4.24 erstellt wurden.
    - Erkennung von Tombstones zur Neustartwiederherstellung für blockierte Hauptsitzungen und Subagenten. Doctor meldet die blockierten Sitzungen und repariert nur veraltete Abbruch-Flags, die einem vorhandenen Tombstone widersprechen; die automatische Wiederherstellung wird nicht erneut aktiviert.
    - Prüfungen von Zustandsintegrität und Berechtigungen (Sitzungen, Protokolle, Zustandsverzeichnis).
    - Prüfungen der Berechtigungen der Konfigurationsdatei (chmod 600) bei lokaler Ausführung.
    - Systemzustand der Modellauthentifizierung: prüft den OAuth-Ablauf, kann bald ablaufende Token aktualisieren und meldet Abkling-/Deaktivierungszustände von Authentifizierungsprofilen.

  </Accordion>
  <Accordion title="Gateway, Dienste und Supervisoren">
    - Reparatur des Sandbox-Images, wenn Sandboxing aktiviert ist.
    - Migration veralteter Dienste und Erkennung zusätzlicher Gateways.
    - Migration des veralteten Matrix-Kanalzustands (im Modus `--fix` / `--repair`).
    - Gateway-Laufzeitprüfungen (Dienst installiert, aber nicht aktiv; zwischengespeichertes launchd-Label).
    - Warnungen zum Kanalstatus (vom laufenden Gateway abgefragt).
    - Kanalspezifische Berechtigungsprüfungen befinden sich unter `openclaw channels capabilities`; beispielsweise werden Discord-Sprachkanalberechtigungen mit `openclaw channels capabilities --channel discord --target channel:<channel-id>` geprüft.
    - Prüfungen der WhatsApp-Reaktionsfähigkeit bei beeinträchtigtem Zustand der Gateway-Ereignisschleife, während lokale TUI-Clients noch ausgeführt werden; `--fix` beendet nur verifizierte lokale TUI-Clients.
    - Reparatur von Codex-Routen für veraltete `openai-codex/*`-Modellreferenzen in primären Modellen, Fallbacks, Bild-/Videogenerierungsmodellen, Heartbeat-/Subagenten-/Compaction-Überschreibungen, Hooks, Kanalmodellüberschreibungen und Sitzungsroutenfixierungen; `--fix` schreibt sie zu `openai/*` um, migriert `openai-codex:*`-Authentifizierungsprofile/-Reihenfolge zu `openai:*`, entfernt veraltete Laufzeitfixierungen für Sitzungen/gesamte Agenten und lässt die reparierte effektive Route bestimmen, ob Codex kompatibel ist.
    - Audit der Supervisor-Konfiguration (launchd/systemd/schtasks) mit optionaler Reparatur.
    - Bereinigung eingebetteter Proxy-Umgebungen für Gateway-Dienste, die während der Installation oder Aktualisierung Shell-Werte für `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` übernommen haben.
    - Gateway-Laufzeitprüfungen (nicht unterstützte veraltete Bun-Dienste, Pfade von Versionsmanagern).
    - Diagnose von Gateway-Portkonflikten (Standardwert `18789`).

  </Accordion>
  <Accordion title="Authentifizierung, Sicherheit und Kopplung">
    - Sicherheitswarnungen bei offenen DM-Richtlinien.
    - Gateway-Authentifizierungsprüfungen für den lokalen Token-Modus (bietet die Token-Generierung an, wenn keine Token-Quelle vorhanden ist; überschreibt keine Token-SecretRef-Konfigurationen).
    - Erkennung von Problemen bei der Gerätekopplung (ausstehende erstmalige Kopplungsanfragen, ausstehende Rollen-/Bereichserweiterungen, Abweichungen in veralteten lokalen Geräte-Token-Caches und Authentifizierungsabweichungen in Kopplungsdatensätzen).

  </Accordion>
  <Accordion title="Workspace und Shell">
    - Prüfung von systemd-Linger unter Linux.
    - Prüfung der Größe von Workspace-Bootstrap-Dateien (Warnungen bei Kürzung oder Annäherung an das Limit für Kontextdateien).
    - Bereitschaftsprüfung der Skills für den Standardagenten; meldet zulässige Skills, bei denen Binärdateien, Umgebung, Konfiguration oder Betriebssystemanforderungen fehlen, und `--fix` kann nicht verfügbare Skills in `skills.entries` deaktivieren.
    - Statusprüfung der Shell-Vervollständigung und automatische Installation/Aktualisierung.
    - Bereitschaftsprüfung des Embedding-Providers für die Speichersuche (lokales Modell, Remote-API-Schlüssel oder QMD-Binärdatei).
    - Prüfungen der Quellinstallation (Abweichung im pnpm-Workspace, fehlende UI-Assets, fehlende tsx-Binärdatei).
    - Schreibt aktualisierte Konfiguration + Assistentenmetadaten.

  </Accordion>
</AccordionGroup>

## Nachträgliche Befüllung und Zurücksetzung der Dreams-Benutzeroberfläche

  Die Dreams-Szene der Control UI enthält die Aktionen **Backfill**, **Reset** und **Clear Grounded** für den Grounded-Dreaming-Workflow. Diese verwenden RPC-Methoden im Stil des Gateway-Doctors, sind jedoch **nicht** Teil der CLI-Reparatur/Migration von `openclaw doctor`.

  | Aktion         | Funktion                                                                                                                                                          |
  | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | Backfill       | Durchsucht historische `memory/YYYY-MM-DD.md`-Dateien im aktiven Workspace, führt den Grounded-REM-Tagebuchdurchlauf aus und schreibt umkehrbare Backfill-Einträge in `DREAMS.md`. |
  | Reset          | Entfernt nur die markierten Backfill-Tagebucheinträge aus `DREAMS.md`.                                                                                     |
  | Clear Grounded | Entfernt nur bereitgestellte, ausschließlich Grounded-bezogene Kurzzeiteinträge aus der historischen Wiedergabe, für die noch kein Live-Abruf oder täglicher Support angesammelt wurde. |

  Keine dieser Aktionen bearbeitet `MEMORY.md`, führt vollständige Doctor-Migrationen aus oder stellt eigenständig Grounded-Kandidaten im Live-Speicher für die Kurzzeit-Promotion bereit. Um eine historische Grounded-Wiedergabe in den normalen Deep-Promotion-Pfad einzuspeisen, verwenden Sie stattdessen den CLI-Ablauf:

  ```bash
  openclaw memory rem-backfill --path ./memory --stage-short-term
  ```

  Dadurch werden dauerhafte Grounded-Kandidaten im Kurzzeit-Dreaming-Speicher bereitgestellt, während `DREAMS.md` die Prüfoberfläche bleibt.

  ## Detailliertes Verhalten und Begründung

  <AccordionGroup>
  <Accordion title="0. Optionale Aktualisierung (Git-Installationen)">
    Wenn es sich um einen Git-Checkout handelt und Doctor interaktiv ausgeführt wird, bietet er vor seiner Ausführung eine Aktualisierung (Abrufen/Rebase/Build) an.
  </Accordion>
  <Accordion title="1. Konfigurationsnormalisierung">
    Doctor normalisiert veraltete Wertstrukturen in das aktuelle Schema. Die aktuelle Talk-Sprachkonfiguration besteht aus `talk.provider` + `talk.providers.<provider>`, wobei sich die Echtzeit-Sprachkonfiguration unter `talk.realtime.*` befindet. Doctor überführt alte Strukturen von `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` / `talk.apiKey` in die Provider-Zuordnung und überführt veraltete Echtzeit-Selektoren auf oberster Ebene (`talk.mode`, `talk.transport`, `talk.brain`, `talk.model`, `talk.voice`) in `talk.realtime`.

    Doctor warnt außerdem, wenn `plugins.allow` nicht leer ist und die Tool-Richtlinie Platzhalter- oder Plugin-eigene Tool-Einträge verwendet. `tools.allow: ["*"]` stimmt nur mit Tools aus tatsächlich geladenen Plugins überein; die exklusive Plugin-Zulassungsliste wird dadurch nicht umgangen.

  </Accordion>
  <Accordion title="2. Migrationen veralteter Konfigurationsschlüssel">
    Wenn die Konfiguration einen veralteten Schlüssel mit einer aktiven Migration enthält, verweigern andere Befehle die Ausführung und fordern Sie auf, `openclaw doctor` auszuführen. Doctor erläutert, welche veralteten Schlüssel gefunden wurden, zeigt die angewendete Migration an und schreibt `~/.openclaw/openclaw.json` mit dem aktualisierten Schema neu. Der Gateway-Start verweigert veraltete Konfigurationsformate und fordert Sie auf, `openclaw doctor --fix` auszuführen; `openclaw.json` wird beim Start nicht neu geschrieben. Migrationen des Cron-Auftragsspeichers werden ebenfalls von `openclaw doctor --fix` verarbeitet.

    <Note>
      Doctor führt automatische Migrationen nur ungefähr zwei Monate lang
      mit, nachdem ein Schlüssel eingestellt wurde. Für ältere veraltete Schlüssel
      (beispielsweise die ursprünglichen `routing.queue`, `routing.bindings`,
      `routing.agents`/`defaultAgentId`, `routing.transcribeAudio`,
      `agent.*` auf oberster Ebene oder `identity` auf oberster
      Ebene aus der Konfigurationsstruktur vor der Multi-Agent-Unterstützung)
      gibt es keinen Migrationspfad mehr; Konfigurationen, die sie verwenden,
      schlagen nun bei der Validierung fehl, statt neu geschrieben zu werden.
      Korrigieren Sie diese Schlüssel anhand der aktuellen Konfigurationsreferenz
      manuell, bevor Doctor fortfahren kann.
    </Note>

    Aktive Migrationen:

    | Veralteter Schlüssel                                                                                    | Aktueller Schlüssel                                                                 |
    | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
    | `routing.allowFrom`                                                                              | `channels.whatsapp.allowFrom`                                                |
    | `routing.groupChat.requireMention`                                                               | `channels.whatsapp/telegram/imessage.groups."*".requireMention`             |
    | `routing.groupChat.historyLimit`                                                                 | `messages.groupChat.historyLimit`                                            |
    | `routing.groupChat.mentionPatterns`                                                              | `messages.groupChat.mentionPatterns`                                         |
    | `channels.telegram.requireMention`                                                               | `channels.telegram.groups."*".requireMention`                               |
    | `channels.webchat`, `gateway.webchat`                                                            | entfernt (WebChat wurde eingestellt)                                                 |
    | `channels.feishu.accounts.<accountId>.botName`                                                   | `channels.feishu.accounts.<accountId>.name`                                 |
    | `session.threadBindings.ttlHours`, `channels.<id>.threadBindings.ttlHours` (und kontospezifisch)      | `...threadBindings.idleHours`                                               |
    | veraltete `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey`        | `talk.provider` + `talk.providers.<provider>`                               |
    | veraltete globale Selektoren für Echtzeit-Talk (`talk.mode`/`talk.transport`/`talk.brain`/`talk.model`/`talk.voice`) | `talk.realtime`                                                              |
    | `messages.tts`                                                                                  | globales `tts`                                                              |
    | `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`)                             | `tts.providers.<provider>`                                                   |
    | `messages.tts.provider: "edge"` / `messages.tts.providers.edge`                                  | `tts.provider: "microsoft"` / `tts.providers.microsoft`                    |
    | `tools.exec.security` + `tools.exec.ask`                                                         | `tools.exec.mode`                                                            |
    | `session.idleMinutes`                                                                            | `session.reset.idleMinutes`                                                  |
    | `messages.responsePrefix` mit expliziten Kanalblöcken                                           | in `responsePrefix` des konfigurierten Kanals/Kontos kopiert; globaler Fallback für implizite/benutzerdefinierte Kanäle beibehalten |
    | `web.enabled`                                                                                    | `channels.whatsapp.enabled`                                                  |
    | `meta.lastTouchedAt`, Hook-Installationen, Cron-Speicher, gebündelte Erkennung, globaler Pfad für TTS-Einstellungen            | gemeinsamer SQLite-Zustand                                                       |
    | TTS-Sprecherfelder `voice`/`voiceName`/`voiceId`                                                 | `speakerVoice`/`speakerVoiceId`                                              |
    | `channels.<id>.tts.<provider>` / `channels.<id>.accounts.<accountId>.tts.<provider>` (alle Kanäle außer Discord)                                          | `...tts.providers.<provider>`                                                |
    | `channels.<id>.voice.tts.<provider>` / `channels.<id>.accounts.<accountId>.voice.tts.<provider>` (alle Kanäle einschließlich Discord)                          | `...voice.tts.providers.<provider>`                                          |
    | `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`)     | `plugins.entries.voice-call.config.tts.providers.<provider>`                |
    | `plugins.entries.voice-call.config.tts.provider: "edge"` / `...tts.providers.edge`                | `provider: "microsoft"` / `...tts.providers.microsoft`                      |
    | `plugins.entries.voice-call.config.provider: "log"`                                              | `"mock"`                                                                      |
    | `plugins.entries.voice-call.config.twilio.from`                                                  | `plugins.entries.voice-call.config.fromNumber`                              |
    | `plugins.entries.voice-call.config.streaming.sttProvider`                                        | `plugins.entries.voice-call.config.streaming.provider`                      |
    | `plugins.entries.voice-call.config.streaming.openaiApiKey`/`sttModel`/`silenceDurationMs`/`vadThreshold` | `plugins.entries.voice-call.config.streaming.providers.openai.*`             |
    | `models.providers.*.api: "openai"`                                                               | `"openai-completions"` (beim Gateway-Start werden außerdem Provider übersprungen, deren `api` ein zukünftiger/unbekannter Enum-Wert ist, anstatt geschlossen fehlzuschlagen) |
    | `browser.ssrfPolicy.allowPrivateNetwork`                                                         | `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`                          |
    | `browser.profiles.*.driver: "extension"`                                                         | `"existing-session"`                                                          |
    | `browser.relayBindHost`                                                                          | entfernt (veraltete Relay-Einstellung der Chrome-Erweiterung)                             |
    | `mcp.servers.*.type` (CLI-native Aliasse)                                                        | `mcp.servers.*.transport`                                                    |
    | `mcp.servers.*.disabled`                                                                         | inverses `mcp.servers.*.enabled`                                              |
    | MCP-Timeout-Aliasse `connectTimeout`/`connect_timeout`/`timeout`                                 | `connectionTimeoutMs`/`requestTimeoutMs`                                    |
    | MCP-Serverfelder in Snake Case                                                                     | MCP-Serverfelder in camelCase                                                   |
    | `tools.media.image/audio/video.models`                                                           | mit Fähigkeiten gekennzeichnetes `tools.media.models`                                        |
    | `tools.media.asyncCompletion`                                                                    | entfernt                                                                       |
    | `tools.message.allowCrossContextSend`                                                            | `tools.message.crossContext`                                                  |
    | `deepgram`-Optionen des Medienmodells                                                                   | `providerOptions.deepgram`                                                    |
    | `talk.realtime.voice`, Discord-Echtzeit-`voice`                                                 | `speakerVoice`                                                                |
    | `agents.defaults.pdfMaxBytesMb`                                                                  | `agents.defaults.pdfMaxMb`                                                    |
    | `tools.exec.timeoutSec`                                                                          | `tools.exec.timeoutSeconds`                                                   |
    | `browser.ssrfPolicy.hostnameAllowlist`                                                           | Wildcard-fähiges `browser.ssrfPolicy.allowedHostnames`                          |
    | `enableNoVnc` des Sandbox-Browsers                                                                    | `noVncEnabled`                                                                |
    | Stamm-`media`                                                                                     | `attachments`                                                                |
    | Sichtbarkeitsblöcke `heartbeat` für Kanal/Konto                                                   | `heartbeatVisibility`                                                         |
    | `channels.slack.identity`                                                                        | `channels.slack.postAs`                                                       |
    | Stamm-`audit`                                                                                     | `logging.audit`                                                               |
    | `gateway.nodes.skills.enabled`                                                                   | `gateway.nodes.allowSkills`                                                   |
    | `gateway.nodes.allowCommands`/`denyCommands`                                                    | `gateway.nodes.commands.allow`/`deny`                                         |
    | Standardwerte des Generierungsmodells                                                                       | `agents.defaults.mediaModels.{image,video,music}`                              |
    | eingestellte Optimierungsoptionen für das endgültige Layout                                                               | integriertes Standardverhalten                                                     |
    | `channels.whatsapp.messagePrefix` und veraltetes `messages.messagePrefix`                            | `channels.whatsapp.responsePrefix`                                            |
    | `channels.whatsapp.ackReaction`                                                                  | globales `messages.ackReaction` und `ackReactionScope`, sofern übersetzbar        |
    | `cron.failureDestination`                                                                        | Zielfelder in `cron.failureAlert`                                     |
    | `gateway.controlUi.chatMessageMaxWidth`, ausschließlich darstellungsbezogene `ui.prefs`-Schlüssel                       | entfernt (Textskalierung, Chatbreite und Live-Aktivität der Seitenleiste sind browserlokal) |
    | `agents.list`                                                                                    | schlüsselbasiertes `agents.entries`                                                        |
    | globales `defaultModel`                                                                         | `agents.defaults.model`                                                      |
    | `messages.messagePrefix`                                                                         | `channels.whatsapp.responsePrefix`                                            |
    | `session.maintenance.pruneDays`, `session.resetByType.dm`                                        | `session.maintenance.pruneAfter`, `session.resetByType.direct`               |
    | globales `tui`                                                                                  | entfernt (die TUI-Fußzeile verwendet den kompakten Standardwert)                            |
    | `plugins.entries.codex.config.codexDynamicToolsProfile`                                          | entfernt (der Codex-App-Server behält Codex-native Workspace-Tools stets nativ) |
    | `commands.modelsWrite`                                                                           | entfernt (`/models add` ist veraltet)                                       |
    | `agents.defaults/list[].silentReplyRewrite`, `surfaces.*.silentReplyRewrite`                     | entfernt (das exakte `NO_REPLY` wird nicht mehr in sichtbaren Fallback-Text umgeschrieben)  |
    | `agents.defaults/list[].systemPromptOverride`                                                    | entfernt (OpenClaw verwaltet den generierten System-Prompt)                        |
    | `agents.defaults/list[].embeddedPi`                                                              | `embeddedAgent`                                                              |
    | `agents.defaults/list[].sandbox.perSession`                                                      | `sandbox.scope`                                                              |
    | `agents.defaults.llm`                                                                             | entfernt (verwenden Sie `models.providers.<id>.timeoutSeconds` für Timeouts langsamer Modelle/Provider, die unterhalb der Timeout-Obergrenze für Agentenläufe bleiben) |
    | oberste Ebene `memorySearch`, `agents.defaults.memorySearch`                                         | `memory.search`                                                             |
    | `agents.entries.*.memorySearch`                                                                     | `agents.entries.*.memory.search`                                               |
    | `memorySearch.provider: "auto"`                                                                  | `"openai"`                                                                    |
    | `memorySearch.store.path` (auf jeder Ebene)                                                            | entfernt (Speicherindizes befinden sich in der jeweiligen Agentendatenbank)                       |
    | oberste Ebene `heartbeat`                                                                            | `agents.defaults.heartbeat` / `channels.defaults.heartbeat`                 |
    | `plugins.openai-codex`-Richtlinien-IDs                                                                | `plugins.openai`                                                             |
    | `tools.web.x_search.apiKey`                                                                      | `plugins.entries.xai.config.webSearch.apiKey`                               |
    | `session.maintenance.rotateBytes`, `session.parentForkMaxTokens`                                 | entfernt (veraltet)                                                        |
    | 2026.7 eingestellte Optionen zur Laufzeit- und Kanaloptimierung                                               | entfernt (integrierte Produktionsstandards gelten)                               |

    <Note>
      Die obigen `plugins.entries.voice-call.config.*`-Zeilen werden bei jedem Laden der Konfiguration vom
      Voice-Call-Plugin selbst normalisiert, nicht von `openclaw
      doctor`. Das Plugin protokolliert außerdem beim Start eine Warnung, die auf `openclaw
      doctor --fix` verweist, aber Doctor schreibt
      `openclaw.json` für diese Schlüssel derzeit nicht neu; die eigene Normalisierung des Plugins
      wendet die Änderung zur Laufzeit an.
    </Note>

    Hinweise zu Standardkonten für Kanäle mit mehreren Konten:

    - Wenn zwei oder mehr `channels.<channel>.accounts`-Einträge ohne `channels.<channel>.defaultAccount` oder `accounts.default` konfiguriert sind, warnt Doctor, dass das Fallback-Routing ein unerwartetes Konto auswählen kann.
    - Wenn `channels.<channel>.defaultAccount` auf eine unbekannte Konto-ID gesetzt ist, warnt Doctor und listet die konfigurierten Konto-IDs auf.

  </Accordion>
  <Accordion title="2b. OpenCode-Provider-Überschreibungen">
    Wenn Sie `models.providers.opencode`, `opencode-zen` oder `opencode-go` manuell hinzugefügt haben, überschreibt dies den integrierten OpenCode-Katalog aus `openclaw/plugin-sdk/llm`. Dadurch können Modelle zur falschen API gezwungen oder Kosten auf null gesetzt werden. Doctor warnt Sie, damit Sie die Überschreibung entfernen und das API-Routing sowie die Kosten pro Modell wiederherstellen können.
  </Accordion>
  <Accordion title="2c. Browsermigration und Chrome-MCP-Bereitschaft">
    Wenn Ihre Browserkonfiguration noch auf den entfernten Chrome-Erweiterungspfad verweist, normalisiert Doctor sie auf das aktuelle hostlokale Chrome-MCP-Verbindungsmodell (`browser.profiles.*.driver: "extension"` → `"existing-session"`; `browser.relayBindHost` entfernt).

    Doctor prüft außerdem den hostlokalen Chrome-MCP-Pfad, wenn Sie `defaultProfile: "user"` oder ein konfiguriertes `existing-session`-Profil verwenden:

    - prüft bei standardmäßigen Profilen mit automatischer Verbindung, ob Google Chrome auf demselben Host installiert ist
    - prüft die erkannte Chrome-Version und warnt, wenn sie älter als Chrome 144 ist
    - erinnert Sie daran, das Remote-Debugging auf der Inspektionsseite des Browsers zu aktivieren (zum Beispiel `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging` oder `edge://inspect/#remote-debugging`)

    Doctor kann die Chrome-seitige Einstellung nicht für Sie aktivieren. Hostlokales Chrome MCP erfordert weiterhin einen Chromium-basierten Browser ab Version 144 auf dem Gateway-/Node-Host, der lokal ausgeführt wird, bei dem Remote-Debugging aktiviert ist und dessen erste Zustimmungsaufforderung für die Verbindung im Browser bestätigt wurde.

    Die Bereitschaftsprüfung deckt hier nur die Voraussetzungen für lokale Verbindungen ab. Bei vorhandenen Sitzungen gelten weiterhin die aktuellen Einschränkungen der Chrome-MCP-Routen; erweiterte Routen wie `responsebody`, PDF-Export, Download-Abfang und Stapelaktionen erfordern weiterhin einen verwalteten Browser oder ein Raw-CDP-Profil. Diese Prüfung gilt nicht für Docker-, Sandbox-, Remote-Browser- oder andere Headless-Abläufe, die weiterhin Raw CDP verwenden.

  </Accordion>
  <Accordion title="2d. OAuth-TLS-Voraussetzungen">
    Wenn ein OpenAI-Codex-OAuth-Profil konfiguriert ist, prüft Doctor den OpenAI-Autorisierungsendpunkt, um sicherzustellen, dass der lokale Node-/OpenSSL-TLS-Stack die Zertifikatskette validieren kann. Wenn die Prüfung aufgrund eines Zertifikatsfehlers fehlschlägt (zum Beispiel `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, ein abgelaufenes oder selbstsigniertes Zertifikat), gibt Doctor plattformspezifische Hinweise zur Behebung aus. Unter macOS mit einem Homebrew-Node lautet die Korrektur normalerweise `brew postinstall ca-certificates`. Mit `--deep` wird die Prüfung auch ausgeführt, wenn das Gateway fehlerfrei arbeitet.
  </Accordion>
  <Accordion title="2e. Codex-OAuth-Provider-Überschreibungen">
    Wenn Sie zuvor veraltete OpenAI-Transporteinstellungen unter `models.providers.openai-codex` hinzugefügt haben, können diese den integrierten Codex-OAuth-Provider-Pfad überlagern. Doctor warnt, wenn diese alten Transporteinstellungen zusammen mit Codex OAuth vorhanden sind, damit Sie die veraltete Transportüberschreibung entfernen oder neu schreiben und das aktuelle Routingverhalten wiederherstellen können. Benutzerdefinierte Proxys und reine Header-Überschreibungen werden weiterhin unterstützt und lösen diese Warnung nicht aus; diese selbst definierten Anfragerouten kommen jedoch nicht für die implizite Codex-Auswahl infrage.
  </Accordion>
  <Accordion title="2f. Reparatur von Codex-Routen">
    Doctor prüft auf veraltete `openai-codex/*`-Modellreferenzen. Das native Routing des Codex-Harness verwendet kanonische `openai/*`-Modellreferenzen, aber das Präfix allein wählt niemals Codex aus. Wenn die Laufzeitrichtlinie nicht gesetzt oder `auto` ist, kommt nur eine exakt übereinstimmende offizielle HTTPS-Route für Platform Responses oder ChatGPT Responses ohne selbst definierte Anfrageüberschreibung infrage. Siehe [Implizite OpenAI-Agentenlaufzeit](/de/providers/openai#implicit-agent-runtime).

    Im Modus `--fix` / `--repair` schreibt Doctor betroffene Referenzen des Standardagenten und einzelner Agenten neu, einschließlich primärer Modelle, Fallbacks, Modelle zur Bild-/Videogenerierung, Heartbeat-/Subagent-/Compaction-Überschreibungen, Hooks, Kanalmodellüberschreibungen und veraltetem persistiertem Sitzungsroutenstatus:

    - `openai-codex/gpt-*` wird zu `openai/gpt-*`.
    - Die Codex-Absicht wird für reparierte Agentenmodellreferenzen in Provider-/modellbezogene `agentRuntime.id: "codex"`-Einträge verschoben.
    - Veraltete Laufzeitkonfigurationen für den gesamten Agenten und persistierte Laufzeitfixierungen von Sitzungen werden entfernt, da die Laufzeitauswahl Provider-/modellbezogen ist.
    - Bestehende Provider-/Modell-Laufzeitrichtlinien bleiben erhalten, sofern die reparierte veraltete Modellreferenz kein Codex-Routing benötigt, um den alten Authentifizierungspfad beizubehalten.
    - Bestehende Modell-Fallback-Listen bleiben erhalten, wobei ihre veralteten Einträge neu geschrieben werden; kopierte Einstellungen pro Modell werden vom veralteten Schlüssel in den kanonischen Schlüssel `openai/*` verschoben.
    - Persistierte Sitzungswerte für `modelProvider`/`providerOverride`, `model`/`modelOverride`, Fallback-Hinweise und Authentifizierungsprofilfixierungen werden in allen gefundenen Sitzungsspeichern der Agenten repariert.
    - Doctor repariert separat veraltete `agentRuntime.id: "codex-cli"`-Fixierungen (eine eigenständige veraltete Laufzeit-ID) zu `"codex"` in den Modelleinträgen `agents.defaults`, `agents.entries.*` und `models.providers.*`.
    - `/codex ...` bedeutet „eine native Codex-Konversation aus dem Chat steuern oder anbinden“.
    - `/acp ...` oder `runtime: "acp"` bedeutet „den externen ACP-/acpx-Adapter verwenden“.

  </Accordion>
  <Accordion title="2g. Bereinigung von Sitzungsrouten">
    Doctor durchsucht außerdem gefundene Sitzungsspeicher von Agenten nach veraltetem, automatisch erstelltem Routenstatus, nachdem Sie konfigurierte Modelle oder die Laufzeit von einer Plugin-eigenen Route wie Codex weg verschoben haben.

    `openclaw doctor --fix` kann automatisch erstellten veralteten Status löschen, etwa `modelOverrideSource: "auto"`-Modellfixierungen, Laufzeitmodellmetadaten, fixierte Harness-IDs, CLI-Sitzungsbindungen und automatische Authentifizierungsprofilüberschreibungen, wenn die zugehörige Route nicht mehr konfiguriert ist. Explizite benutzerdefinierte oder veraltete Sitzungsmodelloptionen werden zur manuellen Prüfung gemeldet und nicht verändert; wechseln Sie sie mit `/model ...`, `/new` oder setzen Sie die Sitzung zurück, wenn diese Route nicht mehr vorgesehen ist.

  </Accordion>
  <Accordion title="3. Migrationen veralteter Zustände (Datenträgerlayout)">
    Doctor kann ältere Datenträgerlayouts in die aktuelle Struktur migrieren:

    - Sitzungsspeicher und Transkripte: von `~/.openclaw/sessions/` nach `~/.openclaw/agents/<agentId>/sessions/`
    - Agentenverzeichnis: von `~/.openclaw/agent/` nach `~/.openclaw/agents/<agentId>/agent/`
    - WhatsApp-Authentifizierungsstatus (Baileys): vom veralteten `~/.openclaw/credentials/*.json` (außer `oauth.json`) nach `~/.openclaw/credentials/whatsapp/<accountId>/...` (Standardkonto-ID: `default`)
    - Signierte Geräteidentität: von `~/.openclaw/identity/device.json` in die `primary`-Zeile `device_identities` in `state/openclaw.sqlite`; die separate Geräteauthentifizierungsdatei bleibt unverändert

    Diese Migrationen erfolgen nach bestem Bemühen und sind idempotent; Doctor gibt Warnungen aus, wenn veraltete Ordner als Sicherungen zurückbleiben. Gateway und CLI migrieren außerdem beim Start automatisch die veralteten Sitzungen und das Agentenverzeichnis, sodass Verlauf, Authentifizierung und Modelle ohne manuellen Doctor-Lauf im agentenspezifischen Pfad landen. Die WhatsApp-Authentifizierung wird absichtlich nur über `openclaw doctor` migriert. Die Normalisierung des Talk-Providers/der Provider-Zuordnung vergleicht anhand struktureller Gleichheit, sodass Unterschiede ausschließlich in der Schlüsselreihenfolge nicht mehr wiederholt wirkungslose `doctor --fix`-Änderungen auslösen.

  </Accordion>
  <Accordion title="3a. Migrationen veralteter Plugin-Manifeste">
    Doctor durchsucht alle installierten Plugin-Manifeste nach veralteten Capability-Schlüsseln auf oberster Ebene (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders`). Wenn solche Schlüssel gefunden werden, bietet Doctor an, sie in das Objekt `contracts` zu verschieben und die Manifestdatei direkt neu zu schreiben. Diese Migration ist idempotent; wenn `contracts` bereits dieselben Werte enthält, wird der veraltete Schlüssel entfernt, ohne Daten zu duplizieren.
  </Accordion>
  <Accordion title="3b. Migrationen veralteter Cron-Speicher">
    Doctor prüft außerdem den veralteten Cron-Auftragsspeicher (`~/.openclaw/cron/jobs.json`) auf alte Auftragsstrukturen, bevor kanonische Zeilen in SQLite importiert werden.

    Aktuelle Cron-Bereinigungen umfassen:

    - `jobId` → `id`
    - `schedule.cron` → `schedule.expr`
    - Payload-Felder auf oberster Ebene (`message`, `model`, `thinking`, ...) → `payload`
    - Zustellungsfelder auf oberster Ebene (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
    - Zustellungsaliase in Payload `provider` → explizites `delivery.channel`
    - veraltete `notify: true`-Webhook-Fallback-Aufträge → explizite Webhook-Zustellung aus dem stillgelegten Raw-Wert `cron.webhook`, sofern gültig; Ankündigungsaufträge behalten ihre Chat-Zustellung und erhalten `delivery.completionDestination`. Doctor entfernt anschließend den alten Konfigurationsschlüssel. Ohne einen verwendbaren veralteten Webhook wird die wirkungslose Markierung `notify` auf oberster Ebene bei Aufträgen ohne Ziel entfernt (die vorhandene Zustellung einschließlich Ankündigungen bleibt erhalten), da die Laufzeitzustellung sie niemals liest.

    Das Gateway bereinigt beim Laden außerdem fehlerhafte Cron-Zeilen, sodass gültige Aufträge weiter ausgeführt werden. Unverarbeitete fehlerhafte Zeilen werden vor ihrer Entfernung aus `jobs.json` nach `jobs-quarantine.json` neben dem aktiven Speicher kopiert; Doctor meldet unter Quarantäne gestellte Zeilen, damit Sie sie manuell prüfen oder reparieren können.

    Beim Start normalisiert das Gateway die Laufzeitprojektion und ignoriert die Markierung `notify` auf oberster Ebene, belässt den persistierten Cron-Status jedoch zur Reparatur durch Doctor. Doctor entfernt wirkungslose Markierungen für Aufträge ohne Migrationsziel (`delivery.mode` nicht vorhanden/fehlend, ein nicht verwendbares veraltetes Webhook-Ziel oder eine vorhandene Ankündigungs-/Chat-Zustellung), ohne die vorhandene Zustellung zu verändern, sodass wiederholte `doctor --fix`-Läufe nicht mehr vor demselben Auftrag warnen.

    Unter Linux warnt Doctor außerdem, wenn die Crontab des Benutzers weiterhin das veraltete `~/.openclaw/bin/ensure-whatsapp.sh` aufruft. Dieses hostlokale Skript wird vom aktuellen OpenClaw nicht gepflegt und kann falsche `Gateway inactive`-Meldungen in `~/.openclaw/logs/whatsapp-health.log` schreiben, wenn Cron den systemd-Benutzerbus nicht erreichen kann. Entfernen Sie den veralteten Crontab-Eintrag mit `crontab -e`; verwenden Sie `openclaw channels status --probe`, `openclaw doctor` und `openclaw gateway status` für aktuelle Zustandsprüfungen.

  </Accordion>
  <Accordion title="3c. Bereinigung von Sitzungssperren">
    Doctor durchsucht alle Agent-Sitzungsverzeichnisse nach veralteten Schreibsperrdateien, die nach einer abnormal beendeten Sitzung zurückgeblieben sind. Für jede gefundene Sperrdatei meldet er: den Pfad, die PID, ob die PID noch aktiv ist, das Alter der Sperre und ob sie als veraltet gilt (inaktive PID, fehlerhafte Eigentümermetadaten, älter als 30 Minuten oder eine aktive PID, die nachweislich zu einem Prozess gehört, der nicht von OpenClaw stammt). Im Modus `--fix` / `--repair` entfernt er automatisch Sperren mit inaktiven, verwaisten, wiederverwendeten, fehlerhaft-alten oder nicht zu OpenClaw gehörenden Eigentümern. Alte Sperren, die weiterhin einem aktiven OpenClaw-Prozess gehören, werden gemeldet, aber beibehalten, damit Doctor keinen aktiven Transkript-Schreibprozess unterbricht.
  </Accordion>
  <Accordion title="3d. Reparatur von Sitzungstranskript-Zweigen">
    Doctor durchsucht die JSONL-Dateien der Agent-Sitzungen nach der duplizierten Zweigstruktur, die durch den Fehler beim Umschreiben von Prompt-Transkripten in Version 2026.4.24 entstanden ist: eine aufgegebene Benutzereingabe mit internem OpenClaw-Laufzeitkontext sowie ein aktiver Geschwisterzweig mit demselben sichtbaren Benutzer-Prompt. Im Modus `--fix` / `--repair` sichert Doctor jede betroffene Datei neben dem Original und schreibt das Transkript auf den aktiven Zweig um, sodass Gateway-Verlauf und Speicherleser keine doppelten Eingaben mehr sehen.
  </Accordion>
  <Accordion title="4. Zustandsintegritätsprüfungen (Sitzungspersistenz, Routing und Sicherheit)">
    Das Zustandsverzeichnis ist das operative Stammhirn. Wenn es verschwindet, gehen Sitzungen, Anmeldedaten, Protokolle und Konfiguration verloren, sofern keine Sicherungen an anderer Stelle vorhanden sind.

    Doctor prüft:

    - **Fehlendes Zustandsverzeichnis**: warnt vor katastrophalem Zustandsverlust, fordert zur Neuerstellung des Verzeichnisses auf und weist darauf hin, dass fehlende Daten nicht wiederhergestellt werden können.
    - **Berechtigungen des Zustandsverzeichnisses**: überprüft die Schreibbarkeit; bietet an, die Berechtigungen zu reparieren (und gibt einen Hinweis `chown` aus, wenn eine Abweichung bei Eigentümer oder Gruppe erkannt wird).
    - **Mit der Cloud synchronisiertes macOS-Zustandsverzeichnis**: warnt, wenn der Zustand unter iCloud Drive (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) oder `~/Library/CloudStorage/...` aufgelöst wird, da synchronisierte Pfade langsamere E/A sowie Sperr-/Synchronisationskonflikte verursachen können.
    - **Linux-Zustandsverzeichnis auf SD oder eMMC**: warnt, wenn der Zustand auf eine `mmcblk*`-Mount-Quelle aufgelöst wird, da zufällige E/A auf SD-/eMMC-Speichern langsamer sein kann und diese durch Schreibvorgänge für Sitzungen und Anmeldedaten schneller verschleißen können.
    - **Flüchtiges Linux-Zustandsverzeichnis**: warnt, wenn der Zustand auf `tmpfs` oder `ramfs` aufgelöst wird, da Sitzungen, Anmeldedaten, Konfiguration und SQLite-Zustand (mit WAL-/Journal-Begleitdateien) bei einem Neustart verschwinden. Docker-`overlay`-Mounts werden bewusst nicht markiert, da ihre beschreibbaren Schichten Neustarts des Hosts überdauern, solange der Container bestehen bleibt.
    - **Fehlende Sitzungsverzeichnisse**: `sessions/` und das Sitzungsspeicherverzeichnis sind erforderlich, um den Verlauf dauerhaft zu speichern und `ENOENT`-Abstürze zu vermeiden.
    - **Transkriptabweichung**: warnt, wenn bei aktuellen Sitzungseinträgen Transkriptdateien fehlen.
    - **Hauptsitzung mit „einzeiliger JSONL-Datei“**: kennzeichnet, wenn das Haupttranskript nur eine Zeile enthält (der Verlauf wächst nicht an).
    - **Mehrere Zustandsverzeichnisse**: warnt, wenn mehrere `~/.openclaw`-Ordner in verschiedenen Home-Verzeichnissen vorhanden sind oder wenn `OPENCLAW_STATE_DIR` auf einen anderen Ort verweist (der Verlauf kann zwischen Installationen aufgeteilt werden).
    - **Hinweis zum Remote-Modus**: Wenn `gateway.mode=remote`, erinnert Doctor daran, ihn auf dem Remote-Host auszuführen (dort befindet sich der Zustand).
    - **Berechtigungen der Konfigurationsdatei**: warnt, wenn `~/.openclaw/openclaw.json` für Gruppe oder alle Benutzer lesbar ist, und bietet an, die Berechtigungen auf `600` zu beschränken.

  </Accordion>
  <Accordion title="5. Zustand der Modellauthentifizierung (OAuth-Ablauf)">
    Doctor prüft OAuth-Profile im Authentifizierungsspeicher, warnt vor bald ablaufenden oder abgelaufenen Tokens und kann sie aktualisieren, wenn dies sicher möglich ist. Wenn das Anthropic-OAuth-/Token-Profil veraltet ist, schlägt er einen Anthropic-API-Schlüssel oder den Anthropic-Setup-Token-Pfad vor. Aufforderungen zur Aktualisierung erscheinen nur bei interaktiver Ausführung (TTY); `--non-interactive` überspringt Aktualisierungsversuche.

    Wenn eine OAuth-Aktualisierung dauerhaft fehlschlägt (beispielsweise `refresh_token_reused`, `invalid_grant` oder wenn ein Provider zur erneuten Anmeldung auffordert), meldet Doctor, dass eine erneute Authentifizierung erforderlich ist, und gibt den exakten auszuführenden Befehl `openclaw models auth login --provider ...` aus.

    Doctor meldet außerdem Authentifizierungsprofile, die aufgrund kurzer Abkühlzeiten (Ratenbegrenzungen/Zeitüberschreitungen/Authentifizierungsfehler) oder längerer Deaktivierungen (Abrechnungs-/Guthabenfehler) vorübergehend nicht verwendbar sind.

    Veraltete Codex-OAuth-Profile, deren Tokens sich im macOS-Schlüsselbund befinden (älteres Onboarding vor dem dateibasierten Begleitdatei-Layout), werden ausschließlich von Doctor repariert. Führen Sie `openclaw doctor --fix` einmal in einem interaktiven Terminal aus, um veraltete, im Schlüsselbund gespeicherte Tokens direkt nach `auth-profiles.json` zu migrieren; anschließend werden sie von eingebetteten Vorgängen (Telegram, Cron, Sub-Agent-Verteilung) als kanonische OpenAI-OAuth-Profile aufgelöst.

  </Accordion>
  <Accordion title="6. Modellvalidierung für Hooks">
    Wenn `hooks.gmail.model` festgelegt ist, validiert Doctor die Modellreferenz anhand des Katalogs und der Zulassungsliste und warnt, wenn sie nicht aufgelöst werden kann oder nicht zulässig ist.
  </Accordion>
  <Accordion title="7. Reparatur von Sandbox-Images">
    Wenn Sandboxing aktiviert ist, prüft Doctor Docker-Images und bietet an, sie zu erstellen oder auf veraltete Namen umzuschalten, falls das aktuelle Image fehlt.
  </Accordion>
  <Accordion title="7b. Bereinigung von Plugin-Installationen">
    Im Modus `openclaw doctor --fix` / `openclaw doctor --repair` entfernt Doctor veraltete, von OpenClaw erzeugte Bereitstellungszustände für Plugin-Abhängigkeiten: veraltete erzeugte Abhängigkeitswurzeln, alte Installations-Staging-Verzeichnisse, paketlokale Rückstände aus früherem Reparaturcode für Abhängigkeiten gebündelter Plugins sowie verwaiste oder wiederhergestellte verwaltete npm-Kopien gebündelter `@openclaw/*`-Plugins, die das aktuelle gebündelte Manifest überlagern können. Doctor verknüpft außerdem das Host-Paket `openclaw` erneut mit verwalteten npm-Plugins, die `peerDependencies.openclaw` deklarieren, damit paketlokale Laufzeitimporte wie `openclaw/plugin-sdk/*` nach Aktualisierungen oder npm-Reparaturen weiterhin aufgelöst werden.

    Doctor kann außerdem fehlende herunterladbare Plugins neu installieren, wenn die Konfiguration auf sie verweist, die lokale Plugin-Registrierung sie jedoch nicht finden kann (wesentliche `plugins.entries`, konfigurierte Kanal-/Provider-/Sucheinstellungen, konfigurierte Agent-Laufzeiten). Während Paketaktualisierungen vermeidet Doctor die Neuinstallation von Plugin-Paketen, solange das Kernpaket ausgetauscht wird; führen Sie `openclaw doctor --fix` nach der Aktualisierung erneut aus, wenn ein konfiguriertes Plugin weiterhin wiederhergestellt werden muss. Außerhalb der nachfolgend beschriebenen Ausnahme für den Start eines Container-Images führen Gateway-Start und erneutes Laden der Konfiguration keine Paketreparatur aus; Plugin-Installationen bleiben explizite Doctor-/Installations-/Aktualisierungsaufgaben.

    Der Start eines containerisierten Gateways verfügt über eine eng begrenzte Upgrade-Ausnahme: Wenn `openclaw gateway run` mit einer neuen OpenClaw-Version startet, führt es vor der Bereitschaft sichere Zustandsmigrationen und die bestehende Plugin-Konvergenz nach der Kernaktualisierung aus und zeichnet anschließend einen versionsbezogenen Prüfpunkt auf. Dieser Startdurchlauf kann veraltete Datensätze gebündelter Plugins bereinigen, lokale Plugin-Verknüpfungen reparieren, konfigurierte Plugin-Pakete neu installieren, wenn der Konvergenzpfad dies erfordert, und aktive Plugin-Nutzlasten prüfen. Wenn der Start keine sichere Reparatur durchführen kann, führen Sie dasselbe Image einmal mit `openclaw doctor --fix` und demselben eingebundenen Zustand und derselben eingebundenen Konfiguration aus, bevor Sie den Container normal neu starten.

  </Accordion>
  <Accordion title="8. Migrationen von Gateway-Diensten und Bereinigungshinweise">
    Doctor erkennt veraltete Gateway-Dienste (launchd/systemd/schtasks) und bietet an, sie zu entfernen und den OpenClaw-Dienst mit dem aktuellen Gateway-Port zu installieren. Er kann außerdem nach zusätzlichen Gateway-ähnlichen Diensten suchen und Bereinigungshinweise ausgeben. Nach Profil benannte OpenClaw-Gateway-Dienste gelten als vollwertig und werden nicht als „zusätzlich“ markiert.

    Wenn unter Linux der Gateway-Dienst auf Benutzerebene fehlt, aber ein systemweiter OpenClaw-Gateway-Dienst vorhanden ist, installiert Doctor nicht automatisch einen zweiten Dienst auf Benutzerebene. Prüfen Sie dies mit `openclaw gateway status --deep` oder `openclaw doctor --deep` und entfernen Sie anschließend das Duplikat oder legen Sie `OPENCLAW_SERVICE_REPAIR_POLICY=external` fest, wenn ein System-Supervisor den Gateway-Lebenszyklus verwaltet.

  </Accordion>
  <Accordion title="8b. Matrix-Migration beim Start">
    Wenn für ein Matrix-Kanalkonto eine ausstehende oder durchführbare Migration eines veralteten Zustands vorliegt, erstellt Doctor (im Modus `--fix` / `--repair`) eine Momentaufnahme vor der Migration und führt anschließend die bestmöglichen Migrationsschritte aus: die Migration des veralteten Matrix-Zustands und die Vorbereitung des veralteten verschlüsselten Zustands. Beide Schritte sind nicht schwerwiegend; Fehler werden protokolliert und der Start wird fortgesetzt. Im schreibgeschützten Modus (`openclaw doctor` ohne `--fix`) wird diese Prüfung vollständig übersprungen.
  </Accordion>
  <Accordion title="8c. Gerätekopplung und Authentifizierungsabweichungen">
    Doctor prüft den Zustand der Gerätekopplung im Rahmen der normalen Integritätsprüfung und meldet:

    - ausstehende erstmalige Kopplungsanfragen
    - ausstehende Rollen- oder Bereichserweiterungen für bereits gekoppelte Geräte
    - Reparaturen bei Abweichungen des öffentlichen Schlüssels, bei denen die Geräte-ID weiterhin übereinstimmt, die Geräteidentität jedoch nicht mehr mit dem genehmigten Datensatz übereinstimmt
    - gekoppelte Datensätze, denen ein aktives Token für eine genehmigte Rolle fehlt
    - gekoppelte Tokens, deren Bereiche von der genehmigten Kopplungsgrundlage abweichen
    - lokal zwischengespeicherte Gerätetoken-Einträge für den aktuellen Computer, die älter als eine Gateway-seitige Token-Rotation sind oder veraltete Bereichsmetadaten enthalten

    Doctor genehmigt Kopplungsanfragen nicht automatisch und rotiert Gerätetokens nicht automatisch. Er gibt die exakten nächsten Schritte aus:

    - ausstehende Anfragen mit `openclaw devices list` prüfen
    - die genaue Anfrage mit `openclaw devices approve <requestId>` genehmigen
    - ein neues Token mit `openclaw devices rotate --device <deviceId> --role <role>` rotieren
    - einen veralteten Datensatz mit `openclaw devices remove <deviceId>` entfernen und erneut genehmigen

    Dadurch wird die erstmalige Kopplung von ausstehenden Rollen-/Bereichserweiterungen sowie von veralteten Token-/Geräteidentitätsabweichungen unterschieden und die häufige Lücke „bereits gekoppelt, aber weiterhin Kopplung erforderlich“ geschlossen.

  </Accordion>
  <Accordion title="9. Sicherheitswarnungen">
    Doctor gibt nur dann einen Sicherheitshinweis aus, wenn er eine Warnung findet, beispielsweise einen Provider, der ohne Zulassungsliste für Direktnachrichten offen ist, oder eine gefährlich konfigurierte Richtlinie. Verwenden Sie `openclaw security audit` für das vollständige Sicherheitsinventar.
  </Accordion>
  <Accordion title="10. systemd-Linger (Linux)">
    Bei Ausführung als systemd-Benutzerdienst stellt Doctor sicher, dass Linger aktiviert ist, damit das Gateway nach der Abmeldung aktiv bleibt.
  </Accordion>
  <Accordion title="11. Workspace-Status (Skills, Plugins und TaskFlows)">
    Doctor gibt Probleme und Aktionen für den Standard-Agenten aus, nicht das Inventar des fehlerfreien Zustands:

    - **Skills**: listet zulässige, aber nicht verwendbare Skill-Namen auf; verwenden Sie `openclaw skills check` für Anforderungsdetails und vollständige Anzahlen.
    - **Plugins**: meldet nur fehlerhafte Plugin-IDs; verwenden Sie `openclaw plugins list` für das Inventar geladener, importierter, deaktivierter und gebündelter Plugins.
    - **Warnungen zur Plugin-Kompatibilität**: kennzeichnet Plugins, die Kompatibilitätsprobleme mit der aktuellen Laufzeit aufweisen.
    - **Plugin-Diagnose**: zeigt alle beim Laden von der Plugin-Registrierung ausgegebenen Warnungen oder Fehler an.
    - **TaskFlow-Wiederherstellung**: zeigt verdächtige verwaltete TaskFlows an, die manuell geprüft oder abgebrochen werden müssen.
    - **Claude CLI**: meldet nur Probleme mit Binärdatei, Authentifizierung, Profil, Workspace oder Projektverzeichnis; Details erfolgreicher Prüfungen werden ausgelassen.

  </Accordion>
  <Accordion title="11b. Größe der Bootstrap-Dateien">
    Doctor prüft, ob Workspace-Bootstrap-Dateien (beispielsweise `AGENTS.md`, `CLAUDE.md` oder andere eingefügte Kontextdateien) nahe am konfigurierten Zeichenbudget liegen oder dieses überschreiten. Er meldet für jede Datei die rohe und die eingefügte Zeichenzahl, den Kürzungsprozentsatz, die Kürzungsursache (`max/file` oder `max/total`) sowie die Gesamtzahl der eingefügten Zeichen als Anteil am Gesamtbudget. Wenn Dateien gekürzt werden oder nahe am Grenzwert liegen, gibt Doctor Tipps zur Abstimmung von `agents.defaults.bootstrapMaxChars` und `agents.defaults.bootstrapTotalMaxChars` aus.
  </Accordion>
  <Accordion title="11c. Shell-Vervollständigung">
    Doctor prüft, ob die Tab-Vervollständigung für die aktuelle Shell (zsh, bash, fish oder PowerShell) installiert ist:

    - Wenn das Shell-Profil ein langsames dynamisches Vervollständigungsmuster verwendet (`source <(openclaw completion ...)`), aktualisiert Doctor es auf die schnellere Variante mit zwischengespeicherter Datei.
    - Wenn die Vervollständigung im Profil konfiguriert ist, aber die Cache-Datei fehlt, erstellt Doctor den Cache automatisch neu.
    - Wenn überhaupt keine Vervollständigung konfiguriert ist, fordert Doctor zur Installation auf (nur im interaktiven Modus; wird mit `--non-interactive` übersprungen).

    Führen Sie `openclaw completion --write-state` aus, um den Cache manuell neu zu erstellen.

  </Accordion>
  <Accordion title="11d. Bereinigung veralteter Channel-Plugins">
    Wenn `openclaw doctor --fix` ein fehlendes Channel-Plugin entfernt, wird auch die verwaiste Channel-spezifische Konfiguration entfernt, die auf dieses Plugin verwiesen hat: `channels.<id>`-Einträge, Heartbeat-Ziele, die den Channel nannten, und `agents.*.models["<channel>/*"]`-Überschreibungen. Dies verhindert Gateway-Startschleifen, bei denen die Channel-Laufzeit nicht mehr vorhanden ist, die Konfiguration das Gateway aber weiterhin auffordert, sich daran zu binden.
  </Accordion>
  <Accordion title="12. Gateway-Authentifizierungsprüfungen (lokales Token)">
    Doctor prüft die Bereitschaft der lokalen Gateway-Token-Authentifizierung.

    - Wenn der Token-Modus ein Token benötigt und keine Token-Quelle vorhanden ist, bietet Doctor an, eines zu generieren.
    - Wenn `gateway.auth.token` von SecretRef verwaltet wird, aber nicht verfügbar ist, warnt Doctor und überschreibt es nicht mit Klartext.
    - `openclaw doctor --generate-gateway-token` erzwingt die Generierung nur, wenn keine Token-SecretRef konfiguriert ist.

  </Accordion>
  <Accordion title="12b. Schreibgeschützte SecretRef-fähige Reparaturen">
    Einige Reparaturabläufe müssen konfigurierte Anmeldedaten prüfen, ohne das Fail-Fast-Verhalten der Laufzeit abzuschwächen.

    - `openclaw doctor --fix` verwendet dasselbe schreibgeschützte SecretRef-Zusammenfassungsmodell wie Befehle der Statusfamilie für gezielte Konfigurationsreparaturen.
    - Beispiel: Die Reparatur von Telegram `allowFrom` / `groupAllowFrom` `@username` versucht, konfigurierte Bot-Anmeldedaten zu verwenden, sofern diese verfügbar sind.
    - Wenn das Telegram-Bot-Token über SecretRef konfiguriert ist, im aktuellen Befehlspfad jedoch nicht zur Verfügung steht, meldet Doctor, dass die Anmeldedaten konfiguriert, aber nicht verfügbar sind, und überspringt die automatische Auflösung, statt abzustürzen oder das Token fälschlicherweise als fehlend zu melden.

  </Accordion>
  <Accordion title="13. Gateway-Integritätsprüfung und Neustart">
    Doctor führt eine Integritätsprüfung durch und bietet einen Neustart des Gateways an, wenn es nicht ordnungsgemäß zu funktionieren scheint.
  </Accordion>
  <Accordion title="13b. Bereitschaft der Speichersuche">
    Doctor prüft, ob der konfigurierte Embedding-Provider für die Speichersuche für den Standard-Agenten bereit ist. Das Verhalten hängt vom konfigurierten Backend und Provider ab:

    - **QMD-Backend**: Prüft, ob die Binärdatei `qmd` verfügbar und startfähig ist. Falls nicht, werden Hinweise zur Behebung ausgegeben, einschließlich `npm install -g @tobilu/qmd` (oder des Bun-Äquivalents) sowie einer Option für einen manuellen Binärpfad.
    - **Expliziter lokaler Provider**: Prüft auf eine lokale Modelldatei oder eine erkannte Remote- bzw. herunterladbare Modell-URL. Falls sie fehlt, wird der Wechsel zu einem Remote-Provider vorgeschlagen.
    - **Expliziter Remote-Provider** (`openai`, `voyage` usw.): Überprüft, ob ein API-Schlüssel in der Umgebung oder im Authentifizierungsspeicher vorhanden ist. Gibt umsetzbare Hinweise zur Behebung aus, wenn er fehlt.
    - **Veralteter automatischer Provider**: Behandelt `memorySearch.provider: "auto"` als OpenAI, prüft die OpenAI-Bereitschaft und schreibt es mit `doctor --fix` in `provider: "openai"` um.

    Wenn ein zwischengespeichertes Ergebnis einer Gateway-Prüfung verfügbar ist (das Gateway war zum Zeitpunkt der Prüfung fehlerfrei), gleicht Doctor dieses Ergebnis mit der für die CLI sichtbaren Konfiguration ab und weist auf Abweichungen hin. Doctor startet im Standardpfad keine neue Embedding-Anfrage; verwenden Sie den Befehl für den detaillierten Speicherstatus, wenn Sie eine Live-Prüfung des Providers wünschen.

    Verwenden Sie `openclaw memory status --deep`, um die Embedding-Bereitschaft zur Laufzeit zu überprüfen.

  </Accordion>
  <Accordion title="14. Warnungen zum Channel-Status">
    Wenn das Gateway fehlerfrei ist, führt Doctor eine Prüfung des Channel-Status durch und meldet Warnungen mit vorgeschlagenen Korrekturen.
  </Accordion>
  <Accordion title="15. Prüfung und Reparatur der Supervisor-Konfiguration">
    Doctor prüft die installierte Supervisor-Konfiguration (launchd/systemd/schtasks) auf fehlende oder veraltete Standardwerte (beispielsweise systemd-Abhängigkeiten von network-online und die Neustartverzögerung). Wenn eine Abweichung gefunden wird, empfiehlt Doctor eine Aktualisierung und kann die Dienstdatei bzw. Aufgabe mit den aktuellen Standardwerten neu schreiben.

    Hinweise:

    - `openclaw doctor` fragt vor dem Neuschreiben der Supervisor-Konfiguration nach.
    - `openclaw doctor --yes` akzeptiert die standardmäßigen Reparaturabfragen.
    - `openclaw doctor --fix` wendet empfohlene Korrekturen ohne Abfragen an (`--repair` ist ein Alias).
    - `openclaw doctor --fix --force` überschreibt benutzerdefinierte Supervisor-Konfigurationen.
    - `OPENCLAW_SERVICE_REPAIR_POLICY=external` lässt Doctor für den Lebenszyklus des Gateway-Dienstes schreibgeschützt. Der Dienstzustand wird weiterhin gemeldet und Reparaturen außerhalb des Dienstes werden weiterhin ausgeführt, aber Installation, Start, Neustart und Bootstrap des Dienstes, das Neuschreiben der Supervisor-Konfiguration sowie die Bereinigung veralteter Dienste werden übersprungen, da ein externer Supervisor diesen Lebenszyklus verwaltet.
    - Unter Linux schreibt Doctor Befehls-/Einstiegspunkt-Metadaten nicht neu, solange die zugehörige systemd-Gateway-Unit aktiv ist. Bei der Suche nach doppelten Diensten werden außerdem inaktive, nicht veraltete zusätzliche Gateway-ähnliche Units ignoriert, damit begleitende Dienstdateien keine unnötigen Bereinigungshinweise erzeugen.
    - Wenn die Token-Authentifizierung ein Token erfordert und `gateway.auth.token` von SecretRef verwaltet wird, validiert die Installation bzw. Reparatur des Dienstes durch Doctor die SecretRef, speichert jedoch keine aufgelösten Klartext-Tokenwerte in den Umgebungsmetadaten des Supervisor-Dienstes.
    - Doctor erkennt verwaltete `.env`-/SecretRef-gestützte Dienstumgebungswerte, die ältere Installationen von LaunchAgent, systemd oder geplanten Windows-Aufgaben inline eingebettet haben, und schreibt die Dienstmetadaten so um, dass diese Werte aus der Laufzeitquelle statt aus der Supervisor-Definition geladen werden.
    - Doctor erkennt, wenn der Dienstbefehl nach Änderungen an `gateway.port` weiterhin einen alten `--port` fest vorgibt, und schreibt die Dienstmetadaten auf den aktuellen Port um.
    - Wenn die Token-Authentifizierung ein Token erfordert und die konfigurierte Token-SecretRef nicht aufgelöst werden kann, blockiert Doctor den Installations-/Reparaturpfad und gibt umsetzbare Hinweise.
    - Wenn sowohl `gateway.auth.token` als auch `gateway.auth.password` konfiguriert sind und `gateway.auth.mode` nicht festgelegt ist, blockiert Doctor die Installation/Reparatur, bis der Modus ausdrücklich festgelegt wurde.
    - Bei Linux-Benutzer-systemd-Units berücksichtigen die Prüfungen von Doctor auf Token-Abweichungen beim Vergleich der Dienst-Authentifizierungsmetadaten sowohl `Environment=`- als auch `EnvironmentFile=`-Quellen.
    - Doctor-Dienstreparaturen verweigern das Neuschreiben, Stoppen oder Neustarten eines Gateway-Dienstes durch eine ältere OpenClaw-Binärdatei, wenn die Konfiguration zuletzt von einer neueren Version geschrieben wurde. Siehe [Gateway-Fehlerbehebung](/de/gateway/troubleshooting#split-brain-installs-and-newer-config-guard).
    - Sie können ein vollständiges Neuschreiben jederzeit über `openclaw gateway install --force` erzwingen.

  </Accordion>
  <Accordion title="16. Gateway-Laufzeit- und Portdiagnose">
    Doctor untersucht die Dienstlaufzeit (PID, letzter Beendigungsstatus) und warnt, wenn der Dienst installiert ist, aber tatsächlich nicht ausgeführt wird. Außerdem prüft Doctor auf Portkonflikte am Gateway-Port (Standard: `18789`) und meldet wahrscheinliche Ursachen (Gateway wird bereits ausgeführt, SSH-Tunnel).
  </Accordion>
  <Accordion title="17. Bewährte Verfahren für die Gateway-Laufzeit">
    Doctor warnt, wenn der Gateway-Dienst unter Bun oder über einen versionsverwalteten Node-Pfad (`nvm`, `fnm`, `volta`, `asdf` usw.) ausgeführt wird. Bun kann den `node:sqlite`-Zustandsspeicher von OpenClaw nicht öffnen, daher migrieren Reparaturen veraltete Bun-Dienste zu Node. Pfade von Versionsmanagern können nach Aktualisierungen nicht mehr funktionieren, weil der Dienst Ihre Shell-Initialisierung nicht lädt. Doctor bietet an, zu einer systemweiten Node-Installation zu migrieren, sofern verfügbar (Homebrew/apt/choco).

    Neu installierte oder reparierte macOS-LaunchAgents verwenden einen kanonischen System-PATH (`/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin`), statt den PATH der interaktiven Shell zu kopieren. Dadurch bleiben von Homebrew verwaltete Systembinärdateien verfügbar, während Volta, asdf, fnm, pnpm und andere Verzeichnisse von Versionsmanagern nicht beeinflussen, welche Node-Version von untergeordneten Prozessen aufgelöst wird. Linux-Dienste behalten weiterhin explizite Umgebungsstammverzeichnisse (`NVM_DIR`, `FNM_DIR`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `BUN_INSTALL`, `PNPM_HOME`) und stabile Benutzer-Binärverzeichnisse bei; vermutete Fallback-Verzeichnisse von Versionsmanagern werden jedoch nur dann in den Dienst-PATH geschrieben, wenn diese Verzeichnisse auf dem Datenträger vorhanden sind.

  </Accordion>
  <Accordion title="18. Schreiben der Konfiguration und Assistenten-Metadaten">
    Doctor speichert alle Konfigurationsänderungen dauerhaft und versieht die Assistenten-Metadaten mit einem Zeitstempel, um den Doctor-Lauf zu protokollieren.
  </Accordion>
  <Accordion title="19. Tipps zum Arbeitsbereich (Sicherung und Speichersystem)">
    Doctor schlägt ein Speichersystem für den Arbeitsbereich vor, wenn keines vorhanden ist, und gibt einen Sicherungshinweis aus, falls der Arbeitsbereich noch nicht unter Git verwaltet wird.

    Eine vollständige Anleitung zur Struktur des Arbeitsbereichs und zur Git-Sicherung (empfohlen wird ein privates GitHub- oder GitLab-Repository) finden Sie unter [/concepts/agent-workspace](/de/concepts/agent-workspace).

  </Accordion>
</AccordionGroup>

## Verwandte Themen

- [Gateway-Betriebshandbuch](/de/gateway)
- [Gateway-Fehlerbehebung](/de/gateway/troubleshooting)
