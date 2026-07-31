---
read_when:
    - Sie möchten einen zuverlässigen Fallback, wenn API-Provider ausfallen
    - Sie führen lokale KI-CLIs aus und möchten sie wiederverwenden
    - Sie möchten die MCP-Loopback-Bridge für den Tool-Zugriff des CLI-Backends verstehen
summary: 'CLI-Backends: lokaler Fallback auf eine KI-CLI mit optionaler MCP-Tool-Bridge'
title: CLI-Backends
x-i18n:
    generated_at: "2026-07-26T17:46:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ce0427c587bf2a1e0a2ff24b5e76952eecae059e6f900af777b897b2d8d4210
    source_path: gateway/cli-backends.md
    workflow: 16
---

OpenClaw kann eine lokale KI-CLI als reine Text-Ausweichlösung ausführen, wenn API-Provider ausgefallen oder ratenbegrenzt sind oder sich fehlerhaft verhalten. Der Ansatz ist bewusst konservativ:

- OpenClaw-Tools werden nicht direkt injiziert, aber ein Backend mit `bundleMcp: true` kann Gateway-Tools über eine Loopback-MCP-Bridge empfangen.
- JSONL-Streaming für CLIs, die es unterstützen.
- Sitzungen werden unterstützt, sodass Folgeinteraktionen kohärent bleiben.
- Bilder werden durchgereicht, wenn die CLI Bildpfade akzeptiert.

Verwenden Sie dies als Sicherheitsnetz für Textantworten, die „immer funktionieren“, nicht als primären Pfad. Verwenden Sie stattdessen [ACP-Agenten](/de/tools/acp-agents) für eine vollständige Harness-Laufzeit mit ACP-Sitzungssteuerung, Hintergrundaufgaben, Thread-/Konversationsbindung und persistenten externen Programmiersitzungen; CLI-Backends sind kein ACP.

<Tip>
  Erstellen Sie ein neues Backend-Plugin? Siehe [CLI-Backend-Plugins](/de/plugins/cli-backend-plugins). Diese Seite behandelt die Konfiguration und den Betrieb eines bereits registrierten Backends.
</Tip>

## Schnellstart

Das mitgelieferte Anthropic-Plugin registriert standardmäßig ein `claude-cli`-Backend. Daher funktioniert es ohne weitere Konfiguration, sofern Claude Code installiert und angemeldet ist:

```bash
openclaw agent --agent main --message "hi" --model claude-cli/claude-sonnet-4-6
```

`main` ist die standardmäßige Agenten-ID, wenn keine explizite Agentenliste konfiguriert ist; verwenden Sie andernfalls Ihre eigene Agenten-ID.

Dem Gateway-Dienst muss die CLI in seinem `PATH` zur Verfügung stehen. Wenn eine Bereitstellung einen
nicht standardmäßigen Pfad zur ausführbaren Datei oder nicht standardmäßige Argumente benötigt, registrieren Sie diesen Adapter stattdessen in einem
[CLI-Backend-Plugin](/de/plugins/cli-backend-plugins), anstatt Startmechanismen in
`openclaw.json` abzulegen.

OpenClaw lädt automatisch das zuständige mitgelieferte Plugin, wenn die Modellauswahl oder ein
modellspezifisches `agentRuntime.id` auf dessen Backend verweist.

## Verwendung als Ausweichlösung

Fügen Sie das CLI-Backend Ihrer Ausweichliste hinzu, damit es nur ausgeführt wird, wenn die primären Modelle fehlschlagen:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["claude-cli/claude-sonnet-4-6"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "claude-cli/claude-sonnet-4-6": {},
      },
    },
  },
}
```

Konfigurierte Ausweichmodelle bleiben verwendbar, wenn der primäre Provider fehlschlägt (Authentifizierung, Ratenbegrenzungen, Zeitüberschreitungen), selbst wenn sie nicht in `agents.defaults.modelPolicy.allow` enthalten sind. Fügen Sie dieser Richtlinie nur dann ein CLI-Backend-Modell hinzu, wenn Benutzer es außerdem direkt über `/model`, eine Sitzungsüberschreibung oder `--model` auswählen können sollen. `agents.defaults.models` verwaltet lediglich modellspezifische Aliasse, Parameter und Metadaten.

## Konfiguration

Benutzer wählen über das Modell und die Laufzeitrichtlinie ein registriertes Backend aus. Behalten Sie
die kanonische Modellreferenz bei und wählen Sie die CLI-Laufzeit modellspezifisch aus:

```json5
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-5",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
      },
    },
  },
}
```

Anmeldedaten verbleiben in den OpenClaw-Authentifizierungsprofilen oder der Konfiguration des zuständigen Plugins.
Befehl, argv, Umgebung, Parsing, Sitzung, Bilder und Watchdog-Mechanismen sind
Plugin-Code, der mit `api.registerCliBackend(...)` registriert wird.

## Funktionsweise

1. Wählt anhand des Provider-Präfixes ein Backend aus (`claude-cli/...`).
2. Erstellt einen System-Prompt mit demselben OpenClaw-Prompt und Arbeitsbereichskontext.
3. Führt die CLI mit einer Sitzungs-ID aus (sofern unterstützt), damit der Verlauf konsistent bleibt. Das mitgelieferte `claude-cli`-Backend hält pro OpenClaw-Sitzung einen Claude-stdio-Prozess aktiv und sendet Folgeinteraktionen über stream-json-stdin.
4. Parst die Ausgabe (JSON oder Klartext) und gibt den endgültigen Text zurück.
5. Speichert Sitzungs-IDs pro Backend, damit Folgeinteraktionen dieselbe CLI-Sitzung wiederverwenden.

## Zeitüberschreitungen und lang laufende Arbeiten

CLI-Backends haben zwei unabhängige Begrenzungen:

- `agents.defaults.timeoutSeconds` begrenzt die gesamte Agenteninteraktion. Normale Gateway-Interaktionen übernehmen den Standardwert von 48 Stunden; `0` hebt die zeitliche Begrenzung der Interaktion auf. Eine gespeicherte Überschreibung wie `600` ersetzt diesen Standardwert.
- Der CLI-Watchdog für ausbleibende Ausgaben beendet einen Unterprozess, der keine Ausgabe erzeugt. Jedes Backend-Plugin verwaltet separate Profile für neue und fortgesetzte Sitzungen, und der Watchdog bleibt auch dann aktiv, wenn die Gesamtzeit der Interaktion unbegrenzt ist.

Entfernen Sie eine kurze Überschreibung der Gesamtzeitüberschreitung, um zum Standardwert von 48 Stunden zurückzukehren, oder legen Sie ein explizites Zeitbudget wie 12 Stunden fest:

```bash
# Zum Standardwert von 48 Stunden zurückkehren:
openclaw config unset agents.defaults.timeoutSeconds

# Oder eine explizite Begrenzung von 12 Stunden auswählen:
openclaw config set agents.defaults.timeoutSeconds 43200
```

Innerhalb einer CLI gestartete Hintergrundarbeit bleibt Teil dieses CLI-Unterprozesses. Wenn die übergeordnete Interaktion ihre Gesamtbegrenzung erreicht, beendet OpenClaw den Unterprozess und dessen CLI-interne Hintergrundaufgaben gemeinsam. Verwenden Sie für dauerhafte, lang laufende Arbeiten einen abgekoppelten OpenClaw-[Unteragenten](/de/tools/subagents) oder [ACP-Agenten](/de/tools/acp-agents); abgekoppelte Unteragenten haben standardmäßig keine Laufzeitbegrenzung.

Der Befehl `openclaw agent` hat außerdem eine eigene Anfragefrist. Sein Ausweichstandardwert von 600 Sekunden gilt für diesen Befehlsaufruf, nicht für gewöhnliche Gateway-Interaktionen; siehe [`openclaw agent`](/de/cli/agent).

### Besonderheiten der Claude-CLI

Das mitgelieferte `claude-cli`-Backend bevorzugt die native Skill-Auflösung von Claude Code. Wenn der aktuelle Skills-Snapshot mindestens einen ausgewählten Skill mit einem materialisierten Pfad enthält, übergibt OpenClaw über `--plugin-dir` ein temporäres Claude-Code-Plugin und lässt den doppelten OpenClaw-Skills-Katalog im angehängten System-Prompt weg. Ohne einen materialisierten Plugin-Skill behält OpenClaw den Prompt-Katalog als Ausweichlösung bei. Überschreibungen von Skill-Umgebungsvariablen/API-Schlüsseln gelten weiterhin für die Umgebung des untergeordneten Prozesses während der Ausführung.

Die Claude-CLI verfügt über einen eigenen nicht interaktiven Berechtigungsmodus; OpenClaw ordnet diesen der vorhandenen Ausführungsrichtlinie zu, anstatt eine Claude-spezifische Konfiguration hinzuzufügen. Für von OpenClaw verwaltete aktive Claude-Sitzungen ist die effektive Ausführungsrichtlinie maßgeblich: YOLO (`tools.exec.mode: "full"`) startet Claude normalerweise mit `--permission-mode bypassPermissions`, während eine restriktive Richtlinie es mit `--permission-mode default` startet. Als root ausgeführte Gateways verwenden ebenfalls `default`, da Claude Code den Umgehungsmodus für root ablehnt. Agentenspezifische `agents.entries.*.tools.exec`-Einstellungen überschreiben für diesen Agenten das globale `tools.exec`. Das Anthropic-Plugin normalisiert Claudes Berechtigungsflags entsprechend der effektiven Richtlinie und Hostbeschränkung.

Unter einer restriktiven Richtlinie fragt Claude OpenClaw über stdio um Erlaubnis, bevor es eines seiner nativen oder Erweiterungstools verwendet (seine eigenen Bash-, WebFetch- oder Claude-in-Chrome-Browsertools). Wenn die effektive Ausführungsabfrageeinstellung `on-miss` oder `always` ist, leitet OpenClaw jede Anfrage als interaktive Genehmigung an den Kanal der Sitzung weiter: **Einmal zulassen** erlaubt den einzelnen Aufruf, **Immer zulassen** erlaubt diesen Toolnamen für den Rest der aktiven Claude-Sitzung (nur im Arbeitsspeicher, niemals persistent gespeichert), und **Ablehnen**, eine Zeitüberschreitung oder eine nicht erreichbare Genehmigungsroute lehnen den Aufruf jeweils ab. Richtlinien, die niemals nachfragen, behalten ihr bisheriges Verhalten bei: `security: "deny"` lehnt jede Anfrage ab, und die Abfrage `off` mit weniger als vollständiger Sicherheit (Ausführungsmodus `allowlist`) lehnt ohne Nachfrage ab.

### Claude-Browsertools und Anmeldung bei 1Password

Claude Code kann über die [Claude in Chrome-Erweiterung](https://code.claude.com/docs/en/chrome) einen Chrome-Browser steuern, einschließlich des automatischen Ausfüllens von Anmeldedaten mit [1Password for Claude](/de/gateway/1password#browser-sign-in-with-1password-for-claude). Das mitgelieferte Backend aktiviert diese Funktion nicht; registrieren Sie ein [CLI-Backend-Plugin](/de/plugins/cli-backend-plugins), das `--chrome` an die Startargumente eines Backends mit `claude-stream-json`-Dialekt anhängt. OpenClaw behält ein konfiguriertes `--chrome` bei normalen Ausführungen bei und erzwingt bei Ausführungen mit einer eingeschränkten Tool-Richtlinie, beispielsweise Nebenfragen, immer `--no-chrome`. Das Chrome-Fenster, die Erweiterung und alle 1Password-Genehmigungsaufforderungen befinden sich auf dem Gateway-Host, daher muss sich jemand an diesem Rechner befinden, um die Verwendung von Anmeldedaten zu genehmigen.

Das Backend ordnet außerdem OpenClaw-`/think`-Stufen dem nativen `--effort`-Flag von Claude Code zu: `minimal`/`low` -> `low`, `medium` -> `medium` und `high`/`xhigh`/`max` werden direkt durchgereicht. Dadurch bleiben die unterstützten Fable-5-Aufwandsstufen für die abonnementgestützte Claude-CLI und API-Schlüssel-Routen identisch. `adaptive` entfernt konfigurierte `--effort`-Flags und liefert keinen Ersatz, sodass Claude Code den effektiven Aufwand aus seiner eigenen Umgebung, seinen Einstellungen und den Modellstandardwerten bestimmt. Bei anderen CLI-Backends muss das zuständige Plugin einen entsprechenden argv-Mapper deklarieren, bevor sich `/think` auf die gestartete CLI auswirkt.

Bevor OpenClaw `claude-cli` verwenden kann, muss Claude Code selbst auf demselben Host angemeldet sein:

```bash
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

Bei Docker-Installationen muss Claude Code innerhalb des persistenten Container-Home-Verzeichnisses installiert und angemeldet sein, nicht nur auf dem Host; siehe [Claude-CLI-Backend in Docker](/de/install/docker#claude-cli-backend-in-docker).

Der Gateway-Dienst muss `claude` über `PATH` auflösen können. Registrieren Sie für einen nicht standardmäßigen Pfad
ein kleines Wrapper-Backend-Plugin.

## Sitzungen

- Wenn die CLI Sitzungen unterstützt, legen Sie `sessionArgs` mit einem `{sessionId}`-Platzhalter fest (zum Beispiel `["--session-id", "{sessionId}"]`).
- Wenn die CLI einen Fortsetzungs-Unterbefehl mit anderen Flags verwendet, legen Sie `resumeArgs` (ersetzt bei der Fortsetzung `args`) und optional `resumeOutput` für Fortsetzungen ohne JSON fest.
- `sessionMode`:
  - `always`: sendet immer eine Sitzungs-ID (eine neue UUID, wenn keine gespeichert ist).
  - `existing`: sendet eine Sitzungs-ID nur, wenn bereits eine gespeichert wurde.
  - `none`: sendet niemals eine Sitzungs-ID.
- `claude-cli` verwendet standardmäßig `liveSession: "claude-stdio"`, `output: "jsonl"` und `input: "stdin"`, sodass Folgeinteraktionen den aktiven Claude-Prozess wiederverwenden, solange er läuft, auch bei benutzerdefinierten Konfigurationen ohne Transportfelder. Wenn das Gateway neu startet oder der inaktive Prozess beendet wird, setzt OpenClaw die Sitzung anhand der gespeicherten Claude-Sitzungs-ID fort. Gespeicherte Sitzungs-IDs werden vor der Fortsetzung anhand eines lesbaren Projekttranskripts überprüft; fehlt das Transkript, wird die Bindung aufgehoben (protokolliert als `reason=transcript-missing`), anstatt stillschweigend eine neue Sitzung unter `--resume` zu starten.
- Aktive Claude-Sitzungen behalten begrenzte Schutzvorkehrungen für JSONL-Ausgaben bei: 8 MiB und 20,000 rohe JSONL-Zeilen pro Interaktion.
- Gespeicherte CLI-Sitzungen stellen eine vom Provider verwaltete Kontinuität dar. Das automatische Zurücksetzen ist standardmäßig deaktiviert; `/reset` und explizite tägliche oder inaktivitätsbasierte `session.reset`-Richtlinien beenden sie weiterhin.
- Neue CLI-Sitzungen werden normalerweise nur aus der Compaction-Zusammenfassung von OpenClaw und dem Teil nach der Compaction neu initialisiert. Um kurze Sitzungen wiederherzustellen, die vor der Compaction ungültig wurden, kann ein Backend dies mit `reseedFromRawTranscriptWhenUncompacted: true` aktivieren. Die Neuinitialisierung aus dem Rohtranskript bleibt begrenzt und auf sichere Ungültigkeitsfälle beschränkt, etwa ein fehlendes CLI-Transkript, ein verwaister Toolverwendungsabschluss, Änderungen an Nachrichtenrichtlinie/System-Prompt/cwd/MCP oder ein Wiederholungsversuch nach Sitzungsablauf; Änderungen am Authentifizierungsprofil oder an der Anmeldedatenepoche initialisieren den Rohtranskriptverlauf niemals neu.

Serialisierung: `serialize: true` hält Ausführungen auf derselben Lane in der richtigen Reihenfolge (die meisten CLIs serialisieren auf einer Provider-Lane). OpenClaw verwirft außerdem die Wiederverwendung gespeicherter CLI-Sitzungen, wenn sich die ausgewählte Authentifizierungsidentität ändert, einschließlich einer geänderten Authentifizierungsprofil-ID, eines statischen API-Schlüssels, eines statischen Tokens oder der OAuth-Kontoidentität, sofern die CLI eine solche bereitstellt; allein die Rotation von OAuth-Zugriffs-/Aktualisierungstokens beendet die Sitzung nicht. Wenn eine CLI keine stabile OAuth-Konto-ID hat, überlässt OpenClaw dieser CLI die Durchsetzung ihrer eigenen Fortsetzungsberechtigungen.

## Ausweichpräambel aus claude-cli-Sitzungen

Wenn ein `claude-cli`-Versuch auf einen Nicht-CLI-Kandidaten in [`agents.defaults.model.fallbacks`](/de/concepts/model-failover) ausweicht, versieht OpenClaw den nächsten Versuch mit einem Kontextvorspann, der aus dem lokalen JSONL-Transkript von Claude Code entnommen wird (unter `~/.claude/projects/`, nach Workspace indiziert). Ohne diesen Ausgangskontext startet der Fallback-Provider ohne Kontext, da OpenClaws eigenes Sitzungstranskript für `claude-cli`-Ausführungen leer ist.

- Der Vorspann bevorzugt die neueste `/compact`-Zusammenfassung oder `compact_boundary`-Markierung und hängt dann unter Einhaltung eines Zeichenbudgets die neuesten Gesprächsbeiträge nach der Grenze an. Gesprächsbeiträge vor der Grenze werden verworfen, da sie bereits in der Zusammenfassung enthalten sind.
- Tool-Blöcke werden zu kompakten `(tool call: name)`- und `(tool result: …)`-Hinweisen zusammengeführt, um das Prompt-Budget korrekt einzuhalten; eine übergroße Zusammenfassung wird gekürzt und mit `(truncated)` gekennzeichnet.
- Fallbacks desselben Providers von `claude-cli` auf `claude-cli` verwenden Claudes eigene `--resume` und überspringen den Vorspann.
- Der Ausgangskontext verwendet die bestehende Validierung des Claude-Sitzungsdateipfads erneut, sodass keine beliebigen Pfade gelesen werden können.

## Bilder

Plugin-Autoren deklarieren die Unterstützung für Bildpfade mit `imageArg`:

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw schreibt Base64-Bilder in temporäre Dateien. Wenn `imageArg` festgelegt ist, werden diese Pfade als CLI-Argumente übergeben; andernfalls hängt OpenClaw die Dateipfade an den Prompt an (Pfadinjektion). Dies funktioniert für CLIs, die lokale Dateien automatisch anhand einfacher Pfade laden.

## Ein- und Ausgaben

- `output: "text"` (Standard) behandelt stdout als endgültige Antwort.
- `output: "json"` versucht, JSON zu parsen und Text sowie eine Sitzungs-ID zu extrahieren.
- `output: "jsonl"` parst einen JSONL-Stream und extrahiert die abschließende Agentennachricht sowie, sofern vorhanden, Sitzungskennungen.
- Bei der JSON-Ausgabe der Gemini CLI liest OpenClaw den Antworttext aus `response` und die Nutzung aus `stats`, wenn `usage` fehlt oder leer ist. Der mitgelieferte Gemini-CLI-Adapter verwendet `stream-json`.

Eingabemodi:

- `input: "arg"` (Standard) übergibt den Prompt als letztes CLI-Argument.
- `input: "stdin"` sendet den Prompt über stdin.
- Wenn der Prompt sehr lang und `maxPromptArgChars` festgelegt ist, wird stattdessen stdin verwendet.

## Plugin-eigene Standardwerte

Die Standardwerte des CLI-Backends sind Teil der Plugin-Oberfläche:

- Plugins registrieren sie mit `api.registerCliBackend(...)`.
- Die Backend-`id` wird zum Provider-Präfix in Modellreferenzen.
- Das Verhalten von Befehl, argv, Umgebung, Parser, Sitzung und Watchdog verbleibt im Plugin-Code.
- Die Backend-spezifische Normalisierung verbleibt über den optionalen `normalizeConfig`-Hook im Besitz des Plugins.

Anthropic ist für `claude-cli` und Google für `google-gemini-cli` zuständig. OpenAI-Codex-Agentenausführungen verwenden das Codex-App-Server-Harness über `openai/*`; OpenClaw registriert kein mitgeliefertes `codex-cli`-Backend mehr.

Das mitgelieferte Anthropic-Plugin registriert Folgendes für `claude-cli`:

| Schlüssel              | Wert                                                                                                                                                                                                          |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`             | `claude`                                                                                                                                                                                                      |
| `args`                | `-p --output-format stream-json --include-partial-messages --verbose --setting-sources user --allowedTools mcp__openclaw__* --disallowedTools ScheduleWakeup,CronCreate,Bash(run_in_background:true),Monitor` |
| `output`              | `jsonl`                                                                                                                                                                                                       |
| `input`               | `stdin`                                                                                                                                                                                                       |
| `modelArg`            | `--model`                                                                                                                                                                                                     |
| `sessionArgs`         | `["--session-id", "{sessionId}"]`                                                                                                                                                                             |
| `sessionMode`         | `always`                                                                                                                                                                                                      |
| `imageArg`            | `@`                                                                                                                                                                                                           |
| `imagePathScope`      | `workspace`                                                                                                                                                                                                   |
| `systemPromptFileArg` | `--append-system-prompt-file`                                                                                                                                                                                 |
| `systemPromptMode`    | `append`                                                                                                                                                                                                      |

Das mitgelieferte Google-Plugin registriert Folgendes für `google-gemini-cli`:

| Schlüssel                  | Wert                                                                                   |
| -------------------------- | -------------------------------------------------------------------------------------- |
| `command`                 | `gemini`                                                                               |
| `args`                    | `--skip-trust --approval-mode auto_edit --output-format stream-json --prompt {prompt}` |
| `resumeArgs`              | identisch, mit `--resume {sessionId}`                                               |
| `output` / `resumeOutput` | `jsonl`                                                                                |
| `jsonlDialect`            | `gemini-stream-json`                                                                   |
| `imageArg`                | `@`                                                                                    |
| `imagePathScope`          | `workspace`                                                                            |
| `modelArg`                | `--model`                                                                              |
| `sessionMode`             | `existing`                                                                             |
| `sessionIdFields`         | `["session_id", "sessionId"]`                                                          |

Voraussetzung: Die lokale Gemini CLI muss installiert und unter `PATH` als `gemini` verfügbar sein (`brew install gemini-cli` oder `npm install -g @google/gemini-cli`).

Hinweise zur Ausgabe der Gemini CLI:

- Der standardmäßige `stream-json`-Parser liest `message`-Ereignisse des Assistenten, Tool-Ereignisse, die abschließende `result`-Nutzung und schwerwiegende Gemini-Fehlerereignisse.
- Für die Nutzung wird auf `stats` zurückgegriffen, wenn `usage` fehlt oder leer ist; `stats.cached` wird in OpenClaw-`cacheRead` normalisiert, und wenn `stats.input` fehlt, werden die Eingabe-Token aus `stats.input_tokens - stats.cached` abgeleitet.

## Texttransformations-Overlays

Plugins, die kleine Kompatibilitäts-Shims für Prompts oder Nachrichten benötigen, können bidirektionale Texttransformationen deklarieren, ohne einen Provider oder ein CLI-Backend zu ersetzen:

```typescript
api.registerTextTransforms({
  input: [{ from: /red basket/g, to: "blue basket" }],
  output: [{ from: /blue basket/g, to: "red basket" }],
});
```

`input` schreibt den an die CLI übergebenen System-Prompt und Benutzer-Prompt um. `output` schreibt gestreamten Assistententext und geparsten endgültigen Text um, bevor OpenClaw seine eigenen Kontrollmarkierungen und die Kanalzustellung verarbeitet; bei Provider-gestützten Modellaufrufen stellt es außerdem Zeichenfolgenwerte innerhalb strukturierter Tool-Aufrufargumente nach der Stream-Reparatur und vor der Tool-Ausführung wieder her. Unverarbeitete Provider-JSON-Fragmente bleiben unverändert; Verbraucher sollten die strukturierte Teil-, End- oder Ergebnissnutzlast verwenden.

Legen Sie für CLIs, die Provider-spezifische JSONL-Ereignisse ausgeben, `jsonlDialect` in der Konfiguration des betreffenden Backends fest: `claude-stream-json` für Claude-Code-kompatible Streams, `gemini-stream-json` für Gemini-CLI-`stream-json`-Ereignisse.

## Zuständigkeit für native Compaction

Einige CLI-Backends führen einen Agenten aus, der sein eigenes Transkript komprimiert. Daher darf OpenClaw seinen absichernden Zusammenfasser nicht auf sie anwenden – andernfalls arbeitet dieser gegen die eigene Compaction des Backends und kann den Gesprächsbeitrag mit einem schwerwiegenden Fehler abbrechen.

`claude-cli` besitzt keinen Harness-Endpunkt (Claude Code führt die Compaction intern aus), daher deklariert es `ownsNativeCompaction: true`, und OpenClaws Compaction-Pfad gibt den Sitzungseintrag unverändert zurück. OpenClaw übergibt das effektive Kontextbudget der Ausführung über die dokumentierte [`CLAUDE_CODE_AUTO_COMPACT_WINDOW`](https://code.claude.com/docs/en/env-vars) von Claude Code, sodass die native automatische Compaction an den konfigurierten Anthropic-`contextTokens`-Grenzwerten ausgerichtet bleibt. Sitzungen mit nativem Harness wie Codex werden stattdessen weiterhin an den Compaction-Endpunkt ihres Harnesses weitergeleitet.

```typescript
api.registerCliBackend({ id: "my-cli", ownsNativeCompaction: true /* ... */ });
```

Deklarieren Sie `ownsNativeCompaction` nur für ein Backend, das tatsächlich für die Compaction zuständig ist: Es muss sein eigenes Transkript zuverlässig nahe dem Kontextfenster begrenzen und eine fortsetzbare Sitzung persistieren (z. B. `--resume` / `--session-id`); andernfalls kann eine zurückgestellte Sitzung das Budget weiterhin überschreiten.

## MCP-Overlays für Bundles

CLI-Backends erhalten OpenClaw-Tool-Aufrufe nicht direkt, ein Backend kann jedoch mit `bundleMcp: true` ein generiertes MCP-Konfigurations-Overlay aktivieren. Aktuelles mitgeliefertes Verhalten:

- `claude-cli`: generierte strikte MCP-Konfigurationsdatei.
- `google-gemini-cli`: generierte Gemini-Systemeinstellungsdatei.

Wenn Bundle-MCP aktiviert ist, führt OpenClaw folgende Schritte aus:

- Es startet einen Loopback-HTTP-MCP-Server, der Gateway-Tools für den CLI-Prozess bereitstellt und mit einer pro Ausführung vergebenen Kontextberechtigung (`OPENCLAW_MCP_TOKEN`) authentifiziert wird, die nur für den aktuellen Ausführungsversuch aktiv ist.
- Es bindet den Tool-Zugriff an den vom Gateway ausgewählten Sitzungs-, Konto- und Kanalkontext, anstatt den Headern des untergeordneten Prozesses zu vertrauen.
- Es lädt aktivierte Bundle-MCP-Server für den aktuellen Workspace und führt sie mit der bestehenden Form der MCP-Konfiguration oder -Einstellungen des Backends zusammen.
- Es schreibt die Startkonfiguration mithilfe des Backend-eigenen Integrationsmodus aus dem zuständigen Plugin um.

Eingeschränkte Ausführungen wie Cron-Jobs mit `toolsAllow` erfordern eine exakte
Backend-eigene Übersetzung. Das gebündelte `claude-cli`-Backend deaktiviert Claudes
native Tools sowie benutzerdefinierte Anpassungen auf Benutzer-, Projekt- und lokaler Ebene, einschließlich Hooks,
Plugins, Agenten, Skills und `CLAUDE.md`. Anschließend stellt es jedes zulässige
OpenClaw-Tool über den berechtigungsbeschränkten MCP-Server bereit. Dadurch verbleiben Richtlinien für Dateisystem,
Prozesse, Ausführung, Genehmigungen und Sandbox innerhalb von OpenClaw, anstatt
die Befugnisse auf Claudes native Tools oder Anpassungsprozesse auszuweiten. Dieselbe MCP-
Liste wird in Claudes generierter Konfiguration und erneut durch das Gateway bei der Auflistung und
Ausführung von Tools durchgesetzt. Vor dem Ausstellen der Berechtigung lehnt der Kern Backend-
Übersetzungen ab, die MCP-Berechtigungen außerhalb der ursprünglichen Zulassungsliste nennen.
Backends ohne exakte Übersetzung schlagen weiterhin nach dem Fail-Closed-Prinzip fehl.

Wenn keine MCP-Server aktiviert sind, fügt OpenClaw dennoch eine strikte Konfiguration ein, sobald sich ein Backend für gebündeltes MCP entscheidet, damit Hintergrundausführungen isoliert bleiben.

Sitzungsgebundene, gebündelte MCP-Laufzeitumgebungen werden zur Wiederverwendung innerhalb einer Sitzung zwischengespeichert und nach 10 Minuten Inaktivität beendet. Einmalige eingebettete Ausführungen wie Authentifizierungsprüfungen, Slug-Generierung und Active-Memory-Abruf fordern am Ende der Ausführung eine Bereinigung an, damit stdio-Kindprozesse und streamfähige HTTP-/SSE-Streams die Ausführung nicht überdauern.

Für `claude-cli` wird ein kompatibles ausgewähltes oder geordnetes OpenClaw-OAuth-/Token-Profil
an diesen untergeordneten Claude-Prozess weitergegeben. Dadurch sind agentenspezifische Profile
für den Durchlauf maßgeblich, während Claudes native Host-Anmeldung erhalten bleibt, wenn kein kompatibles
Profil vorhanden ist.

## Obergrenze für den erneuten Verlaufskontext

Wenn eine neue CLI-Sitzung mit einem vorherigen OpenClaw-Transkript initialisiert wird (beispielsweise nach einem `session_expired`-Wiederholungsversuch), wird der gerenderte `<conversation_history>`-Block begrenzt, damit erneute Initialisierungs-Prompts nicht unverhältnismäßig anwachsen. Der Standardwert beträgt 12.288 Zeichen (etwa 3.000 Token).

Claude-CLI-Backends skalieren diese Obergrenze stattdessen anhand des ermittelten Claude-Kontextfensters: Größere Kontextfenster erhalten einen größeren Ausschnitt des vorherigen Verlaufs bis zu einer festen Höchstgrenze; andere CLI-Backends behalten den konservativen Standardwert bei. Diese Obergrenze gilt nur für den Block mit dem vorherigen Verlauf im erneuten Initialisierungs-Prompt.

## Einschränkungen

- OpenClaw fügt keine Tool-Aufrufe in das CLI-Backend-Protokoll ein. Backends sehen Gateway-Tools nur, wenn sie sich für `bundleMcp: true` entscheiden.
- Streaming ist Backend-spezifisch: Einige Backends streamen JSONL, andere puffern bis zum Beenden.
- Strukturierte Ausgaben hängen vom eigenen JSON-Format der CLI ab.

## Fehlerbehebung

| Symptom               | Lösung                                                                                            |
| --------------------- | ---------------------------------------------------------------------------------------------- |
| CLI nicht gefunden         | Fügen Sie die CLI zum `PATH` des Gateway-Dienstes hinzu oder aktualisieren Sie den registrierten Befehl des zuständigen Plugins. |
| Falscher Modellname      | Aktualisieren Sie die `modelAliases`-Zuordnung des Plugins.                                                    |
| Keine Sitzungskontinuität | Überprüfen Sie `sessionArgs` und `sessionMode` des Plugins.                                            |
| Bilder werden ignoriert        | Überprüfen Sie `imageArg` des Plugins und die Unterstützung der CLI für Dateipfade.                                 |

## Verwandte Themen

- [Gateway-Betriebshandbuch](/de/gateway)
- [Lokale Modelle](/de/gateway/local-models)
