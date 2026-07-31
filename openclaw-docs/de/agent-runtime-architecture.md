---
summary: 'So strukturiert OpenClaw die integrierte Agent-Laufzeit: Codeaufbau, Abgrenzungen, Ressourcenmanifeste und Laufzeitauswahl.'
title: Architektur der Agent-Laufzeitumgebung
x-i18n:
    generated_at: "2026-07-26T17:38:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3e09ff21b4369a7c102db51e4458ad3ba1e86c9fe43a3a8bff72eef1713d2d51
    source_path: agent-runtime-architecture.md
    workflow: 16
---

OpenClaw besitzt die integrierte Agent-Runtime. Der Runtime-Code befindet sich unter `src/agents/`, der Modell-/Provider-Transport unter `src/llm/`, und Verträge für Plugins werden über `openclaw/plugin-sdk/*`-Barrels bereitgestellt.

## Runtime-Struktur

| Pfad                                | Zuständig für                                                                                                                                                                                                                      |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/agents/embedded-agent-runner/` | Integrierte Versuchsschleife (`run.ts`, `run/`), Modellauswahl und Provider-Normalisierung (`model*.ts`), anfragespezifische Parameter je Provider (`extra-params.*`), Compaction sowie Verknüpfung von Transkript und Sitzung.                            |
| `src/agents/sessions/`              | Sitzungspersistenz (`session-manager.ts`), Ressourcenerkennung (`package-manager.ts`, `resource-loader.ts`), sitzungsinternes Laden von `extensions`, Prompt-Vorlagen, Skills, Themes und TUI-gestützte Tool-Renderer (`tools/`). |
| `packages/agent-core/`              | Wiederverwendbarer Agent-Kern (`@openclaw/agent-core`): Agent-Schleife, Harness-Typen, Nachrichten, Compaction-Hilfsfunktionen, Prompt-Vorlagen, Skills und Verträge für die Sitzungsspeicherung.                                                           |
| `src/agents/runtime/`               | OpenClaw-Fassade, die `@openclaw/agent-core` mit der LLM-Runtime des Plugin-SDK verbindet und diese zusammen mit lokalen Proxy-Hilfsfunktionen erneut exportiert.                                                                                             |
| `src/agents/agent-tools*.ts`        | OpenClaw-eigene Tool-Definitionen, Parameterschemas, Tool-Richtlinien, Adapter vor und nach Tool-Aufrufen sowie Bearbeitungs-Tools für Host und Sandbox.                                                                                            |
| `src/agents/agent-hooks/`           | Integrierte Runtime-Hooks: Compaction-Schutzmechanismus, Compaction-Anweisungen, Kontextbereinigung.                                                                                                                                   |
| `src/agents/harness/`               | Harness-Registry, Auswahlrichtlinie und Lebenszyklus für die integrierten und durch Plugins registrierten Harnesses.                                                                                                                       |
| `src/llm/`                          | Modell-/Provider-Registry, Transport-Hilfsfunktionen und providerspezifische Stream-Implementierungen (`src/llm/providers/`).                                                                                                          |

## Grenzen

Der Kern ruft die integrierte Runtime über OpenClaw-Module und SDK-Barrels auf; es verbleiben keine externen Pakete für Agent-Frameworks. Plugins verwenden dokumentierte `openclaw/plugin-sdk/*`-Einstiegspunkte und importieren keine internen Bestandteile von `src/**`.

`@earendil-works/pi-tui` bleibt eine Drittanbieterabhängigkeit: ein Toolkit für Terminalkomponenten, das von der lokalen TUI und den Tool-Renderern für Sitzungen verwendet wird. Seine Internalisierung wäre ein separates Vendoring-Vorhaben.

## Manifeste

Ressourcenpakete deklarieren OpenClaw-Ressourcen in den Metadaten von `package.json`. Einträge sind Dateipfade oder Globs relativ zum Paketstamm:

```json
{
  "openclaw": {
    "extensions": ["extensions/index.ts"],
    "skills": ["skills/*.md"],
    "prompts": ["prompts/*.md"],
    "themes": ["themes/*.json"]
  }
}
```

Ressourcentypen, die nicht in einem Manifest aufgeführt sind, greifen auf die Erkennung der konventionellen Verzeichnisse `extensions/`, `skills/`, `prompts/` und `themes/` zurück.

## Runtime-Auswahl

- Die ID der integrierten Runtime ist `openclaw`. Der veraltete Alias `pi` wird zu `openclaw` normalisiert; `codex-app-server` wird zu `codex` normalisiert.
- Plugin-Harnesses registrieren zusätzliche Runtime-IDs (zum Beispiel `codex`).
- Die Runtime-Richtlinie ist eine modell-/providerspezifische `agentRuntime.id`-Konfiguration (der Modelleintrag hat Vorrang vor dem Providereintrag). Nicht gesetzt oder `default` wird zu `auto` aufgelöst.
- `auto` wählt ein registriertes Plugin-Harness aus, das die effektive Provider-Route unterstützt, andernfalls die integrierte OpenClaw-Runtime. Ein Provider- oder Modellpräfix allein wählt niemals ein Harness aus.
- OpenAI darf `codex` nur implizit für eine exakt übereinstimmende offizielle HTTPS-Route für Platform Responses oder ChatGPT Responses ohne selbst definierte Anfrageüberschreibung auswählen. Completions-Adapter, benutzerdefinierte Endpunkte und Routen mit selbst definiertem Anfrageverhalten verbleiben auf `openclaw`; offizielle Klartext-HTTP-Endpunkte werden abgelehnt. Siehe [Implizite Agent-Runtime von OpenAI](/de/providers/openai#implicit-agent-runtime).

## Generationen der Modell-Runtime

Beim Start des Gateway sowie bei der Veröffentlichung von Konfigurationen, Plugins oder Authentifizierungsdaten wird pro konfiguriertem Agent eine vorbereitete Modell-Runtime-Generation erstellt. Jede Generation besitzt die erkannte Authentifizierungsvorlage, die Modell-Registry und den projizierten Modellkatalog als einen atomaren Snapshot. Agent-Ausführungen erzeugen veränderliche Authentifizierungs- und Registry-Speicher aus diesem Snapshot; Pfade für Durchsuchen, Status, Cron, Doctor, TUI, PDF und Bilder lesen den veröffentlichten Katalog, statt die Dateisystemerkennung zu wiederholen.

Eigenständige eingebettete Runtimes veröffentlichen an ihrer Aktivierungsgrenze dieselbe Snapshot-Struktur. Eine fehlgeschlagene oder veraltete Generation wird niemals zusammen mit einer neueren Teilgeneration bereitgestellt; der für den Lebenszyklus zuständige Eigentümer muss zuerst einen vollständigen Ersatz veröffentlichen.

## Verwandte Themen

- [Workflow der OpenClaw-Agent-Runtime](/de/openclaw-agent-runtime)
- [Agent-Runtimes](/de/concepts/agent-runtimes)
