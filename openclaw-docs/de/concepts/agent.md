---
read_when:
    - Ändern der Agent-Laufzeit, des Workspace-Bootstraps oder des Sitzungsverhaltens
summary: Agentenlaufzeit, Workspace-Vertrag und Sitzungsinitialisierung
title: Agentenlaufzeit
x-i18n:
    generated_at: "2026-07-26T18:54:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4d3dd9c0c65e4ccd791a2a6131f1b7457c8cfee6da71502d93c355280e094390
    source_path: concepts/agent.md
    workflow: 16
---

OpenClaw wird mit einer **eingebetteten Agentenlaufzeit** ausgeliefert: einer integrierten Agentenschleife, Tool-
Anbindung und Prompt-Zusammenstellung, die sich von der Delegierung von Durchläufen an einen externen
Harness-Prozess unterscheidet. Jeder konfigurierte Agent (siehe [Multi-Agenten-Routing](/de/concepts/multi-agent)
zum Ausführen mehrerer Agenten) verfügt über einen eigenen Arbeitsbereich, Bootstrap-Dateien und
Sitzungsspeicher. Diese Seite beschreibt den Laufzeitvertrag: was der Arbeitsbereich
enthalten muss, welche Dateien injiziert werden und wie Sitzungen anhand dieses Bereichs initialisiert werden.

## Arbeitsbereich (erforderlich)

Jeder Agent verwendet ein einzelnes Arbeitsbereichsverzeichnis (`agents.defaults.workspace` oder
`agents.entries.*.workspace` je Agent) als sein **einziges** Arbeitsverzeichnis (`cwd`)
für Tools und Kontext.

Empfehlung: Verwenden Sie `openclaw setup`, um `~/.openclaw/openclaw.json` zu erstellen, falls es fehlt, und die Arbeitsbereichsdateien zu initialisieren.

Vollständiges Arbeitsbereichslayout und Sicherungsanleitung: [Agentenarbeitsbereich](/de/concepts/agent-workspace)

Wenn `agents.defaults.sandbox` aktiviert ist, können Sitzungen, die nicht die Hauptsitzung sind, dies mit
sitzungsspezifischen Arbeitsbereichen unter `agents.defaults.sandbox.workspaceRoot` überschreiben (siehe
[Gateway-Konfiguration](/de/gateway/configuration)).

## Bootstrap-Dateien (injiziert)

Im Arbeitsbereich erwartet OpenClaw die folgenden vom Benutzer bearbeitbaren Dateien:

| Datei           | Zweck                                              |
| -------------- | ---------------------------------------------------- |
| `AGENTS.md`    | Betriebsanweisungen und „Gedächtnis“                    |
| `SOUL.md`      | Persona, Grenzen, Ton                            |
| `TOOLS.md`     | Vom Benutzer gepflegte Tool-Hinweise und Konventionen           |
| `IDENTITY.md`  | Name/Stimmung/Emoji des Agenten                                |
| `USER.md`      | Benutzerprofil und bevorzugte Anrede                     |
| `HEARTBEAT.md` | Heartbeat-spezifische Anweisungen                      |
| `BOOTSTRAP.md` | Einmaliges Ritual beim ersten Start (wird nach Abschluss gelöscht) |
| `MEMORY.md`    | Stammdatei des Langzeitgedächtnisses, falls vorhanden               |

Beim ersten Durchlauf einer neuen Sitzung injiziert OpenClaw den Inhalt dieser Dateien in den Projektkontext des System-Prompts. `MEMORY.md` wird nur injiziert, wenn es im Stammverzeichnis des Arbeitsbereichs vorhanden ist.

Leere Dateien werden übersprungen. Große Dateien werden gekürzt und mit einer Markierung abgeschnitten, damit Prompts kompakt bleiben (lesen Sie die Datei, um den vollständigen Inhalt zu erhalten). Bei einer fehlenden Datei (außer `MEMORY.md`) wird stattdessen eine einzelne Markierungszeile „Datei fehlt“ injiziert; `openclaw setup` erstellt dafür eine sichere Standardvorlage.

`BOOTSTRAP.md` wird nur für einen **vollständig neuen Arbeitsbereich** erstellt (wenn keine anderen Bootstrap-Dateien vorhanden sind). Solange sie aussteht, behält OpenClaw sie im Projektkontext und ergänzt den System-Prompt um Bootstrap-Anweisungen für das anfängliche Ritual, anstatt sie in die Benutzernachricht zu kopieren. Wenn Sie sie nach Abschluss des Rituals löschen, wird sie bei späteren Neustarts nicht erneut erstellt.

Nachdem ein Arbeitsbereich erfasst wurde, speichert OpenClaw dessen Einrichtungsstatus und
Bestätigung in der gemeinsam genutzten SQLite-Datenbank unter
`~/.openclaw/state/openclaw.sqlite`. Wenn ein kürzlich bestätigter Arbeitsbereich
verschwindet oder gelöscht wird, verweigert der Start das stillschweigende erneute Anlegen von `BOOTSTRAP.md`;
stellen Sie den Arbeitsbereich wieder her oder führen Sie ein vollständiges Onboarding-Zurücksetzen durch, damit der Arbeitsbereich und sein
Datenbankstatus gemeinsam gelöscht werden.

Ältere Versionen verwendeten Arbeitsbereichs-JSON- und `.attested`-Sidecar-Dateien. Die Laufzeit
liest diese Dateien nicht. Führen Sie `openclaw doctor --fix` aus, um sie zu validieren, ihren
Status in SQLite zu importieren und jede Quelldatei zu entfernen, nachdem die importierten Zeilen überprüft wurden.

Um die Erstellung von Bootstrap-Dateien vollständig zu deaktivieren (für vorab eingerichtete Arbeitsbereiche), legen Sie Folgendes fest:

```json5
{ agents: { defaults: { skipBootstrap: true } } }
```

## Integrierte Tools

Kern-Tools (Lesen/Ausführen/Bearbeiten/Schreiben und zugehörige System-Tools) sind vorbehaltlich der Tool-Richtlinie immer verfügbar.
`apply_patch` ist für OpenAI-Modelle standardmäßig aktiviert und wird durch
`tools.exec.applyPatch` gesteuert (`enabled`, `workspaceOnly`, `allowModels`). `TOOLS.md` steuert **nicht**, welche Tools vorhanden sind; es handelt sich um
Vorgaben dazu, wie _Sie_ deren Verwendung wünschen.

## Skills

OpenClaw lädt Skills aus den folgenden Speicherorten (mit absteigender Priorität):

- Arbeitsbereich: `<workspace>/skills`
- Projekt-Agenten-Skills: `<workspace>/.agents/skills`
- Persönliche Agenten-Skills: `~/.agents/skills`
- Verwaltet/lokal: `~/.openclaw/skills`
- Gebündelt (mit der Installation ausgeliefert)
- Zusätzliche Skill-Ordner: `skills.load.extraDirs`

Skill-Stammverzeichnisse können gruppierte Ordner enthalten, beispielsweise
`<workspace>/skills/personal/foo/SKILL.md`; der Skill wird dennoch unter seinem
flachen Frontmatter-Namen bereitgestellt, beispielsweise `foo`.

Skills können durch Konfiguration/Umgebungsvariablen eingeschränkt werden (siehe `skills` unter [Gateway-Konfiguration](/de/gateway/configuration)).

## Laufzeitgrenzen

Die eingebettete Agentenlaufzeit wird von OpenClaw verwaltet: Modellerkennung, Tool-Anbindung,
Prompt-Zusammenstellung, Sitzungsverwaltung und Kanalauslieferung bilden eine gemeinsame integrierte
Laufzeitoberfläche.

## Sitzungen

Sitzungszeilen werden in der agentenspezifischen SQLite-Datenbank gespeichert:

- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`

Transkript-JSONL-Dateien können weiterhin unter
`~/.openclaw/agents/<agentId>/sessions/` als Eingaben für Legacy-Migrationen, gelöschte oder
zurückgesetzte Archive, Importe, Exporte und Support-Artefakte vorhanden sein. Der aktive Agentenverlauf wird
zusammen mit den Sitzungszeilen in SQLite gespeichert. Die Sitzungs-ID ist stabil und wird von
OpenClaw festgelegt. OpenClaw liest keine Sitzungsordner anderer Tools.

## Steuerung während des Streamings

Eingehende Prompts, die während eines laufenden Durchlaufs eintreffen, werden standardmäßig in den aktuellen Durchlauf eingespeist.
Die Steuerung erfolgt **nachdem der aktuelle Assistentendurchlauf die Ausführung seiner
Tool-Aufrufe beendet hat**, vor dem nächsten LLM-Aufruf, und überspringt nicht mehr die verbleibenden Tool-Aufrufe
der aktuellen Assistentennachricht.

`/queue steer` ist das Standardverhalten bei einem aktiven Durchlauf. `/queue followup` und
`/queue collect` lassen Nachrichten auf einen späteren Durchlauf warten, anstatt sie einzuspeisen.
`/queue interrupt` bricht stattdessen den aktiven Durchlauf ab. Informationen zum Warteschlangen- und Grenzverhalten finden Sie unter [Warteschlange](/de/concepts/queue)
und [Steuerungswarteschlange](/de/concepts/queue-steering).

Beim Block-Streaming werden abgeschlossene Assistentenblöcke gesendet, sobald sie fertiggestellt sind; es ist
**standardmäßig deaktiviert** (`agents.defaults.blockStreamingDefault: "off"`).
Passen Sie die Grenze über `agents.defaults.blockStreamingBreak` an (`text_end` gegenüber `message_end`; Standardwert: `text_end`).
Steuern Sie die weiche Blockaufteilung mit `agents.defaults.blockStreamingChunk` (Standardwert:
800-1200 Zeichen; bevorzugt Absatzumbrüche, dann Zeilenumbrüche; Sätze zuletzt).
Fassen Sie gestreamte Blöcke mit `agents.defaults.blockStreamingCoalesce` zusammen, um
Spam durch einzelne Zeilen zu reduzieren (leerlaufbasierte Zusammenführung vor dem Senden). Für Kanäle außer Telegram ist
`*.streaming.block.enabled: true` erforderlich, um Blockantworten explizit zu aktivieren (QQ Bot
streamt Blockantworten hingegen, sofern `channels.qqbot.streaming.mode` nicht `"off"` ist).
Ausführliche Tool-Zusammenfassungen werden beim Start des Tools ausgegeben (ohne Entprellung); die Control UI
streamt die Tool-Ausgabe über Agentenereignisse, sofern verfügbar.
Weitere Einzelheiten: [Streaming und Aufteilung](/de/concepts/streaming).

## Modellreferenzen

Modellreferenzen in der Konfiguration (beispielsweise `agents.defaults.model` und `agents.defaults.models`) werden analysiert, indem sie am **ersten** `/` getrennt werden.

- Verwenden Sie beim Konfigurieren von Modellen `provider/model`.
- Wenn die Modell-ID selbst `/` enthält (im OpenRouter-Stil), geben Sie das Provider-Präfix an (Beispiel: `openrouter/moonshotai/kimi-k2`).
- Wenn Sie den Provider weglassen, versucht OpenClaw zuerst einen Alias, dann eine eindeutige
  Übereinstimmung mit einem konfigurierten Provider für genau diese Modell-ID und greift erst danach
  auf den konfigurierten Standard-Provider zurück. Wenn dieser Provider das
  konfigurierte Standardmodell nicht mehr bereitstellt, greift OpenClaw auf den ersten konfigurierten
  Provider bzw. das erste konfigurierte Modell zurück, anstatt einen veralteten Standardwert eines entfernten Providers auszugeben.

## Konfiguration (minimal)

Legen Sie mindestens Folgendes fest:

- `agents.defaults.workspace`
- `channels.whatsapp.allowFrom` (dringend empfohlen)

## Verwandte Themen

- [Agentenarbeitsbereich](/de/concepts/agent-workspace)
- [Multi-Agenten-Routing](/de/concepts/multi-agent)
- [Sitzungsverwaltung](/de/concepts/session)
- [Gruppenchats](/de/channels/group-messages)
