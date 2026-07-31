---
read_when:
    - Sie möchten verstehen, was „Kontext“ in OpenClaw bedeutet
    - Sie untersuchen, warum das Modell etwas „weiß“ (oder vergessen hat)
    - Sie möchten den Kontext-Overhead reduzieren (/context, /status, /compact)
summary: 'Kontext: was das Modell sieht, wie er erstellt wird und wie er sich untersuchen lässt'
title: Kontext
x-i18n:
    generated_at: "2026-07-26T18:24:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1eb3d342a601a447487640587f746cc80a133ede338a880741f53c3e01f20ed1
    source_path: concepts/context.md
    workflow: 16
---

„Kontext“ ist **alles, was OpenClaw für einen Lauf an das Modell sendet**. Er wird durch das **Kontextfenster** des Modells (Token-Limit) begrenzt.

Einfaches mentales Modell:

- **System-Prompt** (von OpenClaw erstellt): Regeln, Tools, Skills-Liste, Zeit/Laufzeit und injizierte Workspace-Dateien.
- **Konversationsverlauf**: Ihre Nachrichten + die Nachrichten des Assistenten für diese Sitzung.
- **Tool-Aufrufe/-Ergebnisse + Anhänge**: Befehlsausgaben, gelesene Dateien, Bilder/Audio usw.

Kontext ist _nicht dasselbe_ wie „Speicher“: Speicher kann auf der Festplatte abgelegt und später erneut geladen werden; Kontext ist das, was sich im aktuellen Fenster des Modells befindet.

## Schnellstart (Kontext prüfen)

- `/status` → schnelle Ansicht „Wie voll ist mein Fenster?“ + Sitzungseinstellungen.
- `/context list` → was injiziert wird + ungefähre Größen (pro Datei + Gesamtwerte).
- `/context detail` → detailliertere Aufschlüsselung: Größen pro Datei, pro Tool-Schema und pro Skill-Eintrag, Größe des System-Prompts und Anzahl der kompaktierbaren Transkriptnachrichten.
- `/context map` → Treemap-Bild im WinDirStat-Stil mit den erfassten Kontextbeiträgen der aktuellen Sitzung.
- `/usage tokens` → Fußzeile zur Nutzung pro Antwort an normale Antworten anhängen.
- `/compact` → älteren Verlauf in einem kompakten Eintrag zusammenfassen, um Platz im Fenster freizugeben.

Siehe auch: [Slash-Befehle](/de/tools/slash-commands), [Token-Nutzung und Kosten](/de/reference/token-use), [Compaction](/de/concepts/compaction).

## Beispielausgabe

Die Werte variieren je nach Modell, Provider, Tool-Richtlinie und Inhalt Ihres Workspace.

### `/context list`

```text
🧠 Kontextaufschlüsselung
Workspace: <workspaceDir>
Bootstrap-Maximum/Datei: 12,000 Zeichen
Sandbox: Modus=non-main sandboxed=false
System-Prompt (Lauf): 38,412 Zeichen (~9,603 Token) (Projektkontext 23,901 Zeichen (~5,976 Token))

Injizierte Workspace-Dateien:
- AGENTS.md: OK | roh 1,742 Zeichen (~436 Token) | injiziert 1,742 Zeichen (~436 Token)
- SOUL.md: OK | roh 912 Zeichen (~228 Token) | injiziert 912 Zeichen (~228 Token)
- TOOLS.md: GEKÜRZT | roh 54,210 Zeichen (~13,553 Token) | injiziert 20,962 Zeichen (~5,241 Token)
- IDENTITY.md: OK | roh 211 Zeichen (~53 Token) | injiziert 211 Zeichen (~53 Token)
- USER.md: OK | roh 388 Zeichen (~97 Token) | injiziert 388 Zeichen (~97 Token)
- HEARTBEAT.md: FEHLT | roh 0 | injiziert 0
- BOOTSTRAP.md: OK | roh 0 Zeichen (~0 Token) | injiziert 0 Zeichen (~0 Token)

Skills-Liste (System-Prompt-Text): 2,184 Zeichen (~546 Token) (12 Skills)
Tools: read, edit, write, exec, process, browser, message, sessions_send, …
Tool-Liste (System-Prompt-Text): 1,032 Zeichen (~258 Token)
Tool-Schemas (JSON): 31,988 Zeichen (~7,997 Token) (werden auf den Kontext angerechnet; nicht als Text angezeigt)
Tools: (wie oben)

Sitzungs-Token (zwischengespeichert): 14,250 gesamt / ctx=32,000
```

### `/context detail`

```text
🧠 Kontextaufschlüsselung (detailliert)
…
Größte Skills (Größe des Prompt-Eintrags):
- frontend-design: 412 Zeichen (~103 Token)
- oracle: 401 Zeichen (~101 Token)
… (+10 weitere Skills)

Größte Tools (Schemagröße):
- browser: 9,812 Zeichen (~2,453 Token)
- exec: 6,240 Zeichen (~1,560 Token)
… (+N weitere Tools)
```

### `/context map`

Sendet ein Bild, das aus dem neuesten zwischengespeicherten Laufbericht und dem Sitzungstranskript generiert wird. Bevor eine normale Nachricht einen Laufbericht in der Sitzung erzeugt hat, gibt `/context map` eine Meldung über die Nichtverfügbarkeit zurück, anstatt eine Schätzung darzustellen. Die Rechteckfläche ist proportional zur Anzahl der erfassten Prompt-Zeichen:

- Konversationstranskript (Benutzernachrichten, Antworten des Assistenten, Tool-Ergebnisse, Compaction-Zusammenfassungen) sowie Laufzeitkontext pro Durchlauf und Prompt-Ergänzungen durch Hooks, die nur das Modell erreichen
- injizierte Workspace-Dateien
- Text des grundlegenden System-Prompts
- Skill-Prompt-Einträge
- JSON-Schemas der Tools

Die Konversationsgruppe wächst mit der Sitzung, daher ändert sich die Darstellung von Durchlauf zu Durchlauf; nach der Compaction wird sie zu einer Zusammenfassungskachel verdichtet.

`/context list`, `/context detail` und `/context json` können weiterhin bei Bedarf eine Schätzung prüfen, wenn kein Laufbericht zwischengespeichert ist.

## Was auf das Kontextfenster angerechnet wird

Alles, was das Modell empfängt, wird angerechnet, einschließlich:

- System-Prompt (alle Abschnitte).
- Konversationsverlauf.
- Tool-Aufrufe + Tool-Ergebnisse.
- Anhänge/Transkripte (Bilder/Audio/Dateien).
- Compaction-Zusammenfassungen und Bereinigungsartefakte.
- „Wrapper“ oder verborgene Header des Providers (nicht sichtbar, werden dennoch angerechnet).

## Wie OpenClaw den System-Prompt erstellt

Der System-Prompt gehört zu **OpenClaw** und wird bei jedem Lauf neu erstellt. Er enthält:

- Tool-Liste + kurze Beschreibungen.
- Skills-Liste (nur Metadaten; siehe unten).
- Workspace-Speicherort.
- Zeit (UTC + umgerechnete Benutzerzeit, falls konfiguriert).
- Laufzeitmetadaten (Host/Betriebssystem/Modell/Denkmodus).
- Injizierte Workspace-Bootstrap-Dateien unter **Projektkontext**.

Vollständige Aufschlüsselung: [System-Prompt](/de/concepts/system-prompt).

## Injizierte Workspace-Dateien (Projektkontext)

Standardmäßig injiziert OpenClaw einen festen Satz von Workspace-Dateien (sofern vorhanden):

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (nur beim ersten Lauf)

Große Dateien werden pro Datei mithilfe von `agents.defaults.bootstrapMaxChars` gekürzt (Standard: `20000` Zeichen). OpenClaw erzwingt außerdem mit `agents.defaults.bootstrapTotalMaxChars` eine Obergrenze für die gesamte Bootstrap-Injektion über alle Dateien hinweg (Standard: `60000` Zeichen). `/context` zeigt die Größen **roh gegenüber injiziert** sowie, ob eine Kürzung stattgefunden hat.

Bei einer Kürzung kann die Laufzeit unter „Projektkontext“ einen Warnblock in den Prompt injizieren. Konfigurieren Sie dies mit `agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always`; Standard: `always`).

## Skills: injiziert oder bei Bedarf geladen

Der System-Prompt enthält eine kompakte **Skills-Liste** (Name + Beschreibung + Speicherort). Diese Liste verursacht einen realen Mehraufwand.

Skill-Anweisungen sind standardmäßig _nicht_ enthalten. Das Modell soll die Datei `SKILL.md` des Skills `read`, **nur wenn dies erforderlich ist**.

## Tools: Es gibt zwei Kostenfaktoren

Tools wirken sich auf zwei Arten auf den Kontext aus:

1. **Text der Tool-Liste** im System-Prompt (was Sie als „Tooling“ sehen).
2. **Tool-Schemas** (JSON). Diese werden an das Modell gesendet, damit es Tools aufrufen kann. Sie werden auf den Kontext angerechnet, obwohl sie nicht als Klartext angezeigt werden.

`/context detail` schlüsselt die größten Tool-Schemas auf, damit Sie erkennen können, was den größten Anteil ausmacht.

## Befehle, Direktiven und „Inline-Kurzbefehle“

Slash-Befehle werden vom Gateway verarbeitet. Dabei gibt es verschiedene Verhaltensweisen:

- **Eigenständige Befehle**: Eine Nachricht, die nur aus `/...` besteht, wird als Befehl ausgeführt.
- **Direktiven**: `/think`, `/fast`, `/verbose`, `/trace`, `/reasoning`, `/elevated`, `/exec`, `/model`, `/queue` werden entfernt, bevor das Modell die Nachricht sieht.
  - Nachrichten, die nur Direktiven enthalten, speichern Sitzungseinstellungen dauerhaft.
  - Inline-Direktiven in einer normalen Nachricht dienen als Hinweise für die jeweilige Nachricht.
- **Inline-Kurzbefehle** (nur Absender auf der Zulassungsliste): Bestimmte `/...`-Token innerhalb einer normalen Nachricht können sofort ausgeführt werden (Beispiel: „hey /status“) und werden entfernt, bevor das Modell den verbleibenden Text sieht.

Details: [Slash-Befehle](/de/tools/slash-commands).

## Sitzungen, Compaction und Bereinigung (was erhalten bleibt)

Was über mehrere Nachrichten hinweg erhalten bleibt, hängt vom Mechanismus ab:

- **Normaler Verlauf** bleibt im Sitzungstranskript erhalten, bis er gemäß der Richtlinie kompaktiert/bereinigt wird.
- **Compaction** speichert eine Zusammenfassung im Transkript und lässt die letzten Nachrichten unverändert.
- **Bereinigung** entfernt alte Tool-Ergebnisse aus dem _speicherinternen_ Prompt, um Platz im Kontextfenster freizugeben, schreibt das Sitzungstranskript jedoch nicht neu – der vollständige Verlauf kann weiterhin auf der Festplatte eingesehen werden.

Dokumentation: [Sitzung](/de/concepts/session), [Compaction](/de/concepts/compaction), [Sitzungsbereinigung](/de/concepts/session-pruning).

Standardmäßig verwendet OpenClaw die integrierte Kontext-Engine `legacy` für die Zusammenstellung und
Compaction. Wenn Sie ein Plugin installieren, das `kind: "context-engine"` bereitstellt, und
es mit `plugins.slots.contextEngine` auswählen, delegiert OpenClaw die Kontextzusammenstellung,
`/compact` und zugehörige Lebenszyklus-Hooks für den Kontext von Subagenten an diese
Engine. `ownsCompaction: false` greift nicht automatisch auf die veraltete
Engine zurück; die aktive Engine muss `compact()` weiterhin korrekt implementieren. Unter
[Kontext-Engine](/de/concepts/context-engine) finden Sie die vollständige
austauschbare Schnittstelle, Lebenszyklus-Hooks und Konfiguration.

## Was `/context` tatsächlich meldet

`/context` bevorzugt den neuesten **während eines Laufs erstellten** Bericht zum System-Prompt, sofern verfügbar:

- `System prompt (run)` = wurde aus dem letzten eingebetteten Lauf (mit Tool-Funktionen) erfasst und im Sitzungsspeicher dauerhaft gespeichert.
- `System prompt (estimate)` = wird unmittelbar berechnet, wenn kein Laufbericht vorhanden ist (oder wenn die Ausführung über ein CLI-Backend erfolgt, das den Bericht nicht erzeugt).

In beiden Fällen werden Größen und die größten Beiträge gemeldet; der vollständige System-Prompt oder die Tool-Schemas werden **nicht** ausgegeben. Im detaillierten Modus wird das Sitzungstranskript außerdem mit demselben Prädikat für echte Konversationsnachrichten verglichen, das von Compaction verwendet wird. Dadurch lässt sich eine hohe Prompt-/Cache-Nutzung leichter von einem kompaktierbaren Konversationsverlauf unterscheiden.

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Kontext-Engine" href="/de/concepts/context-engine" icon="puzzle-piece">
    Benutzerdefinierte Kontextinjektion über Plugins.
  </Card>
  <Card title="Compaction" href="/de/concepts/compaction" icon="compress">
    Zusammenfassen langer Konversationen, damit sie innerhalb des Modellfensters bleiben.
  </Card>
  <Card title="System-Prompt" href="/de/concepts/system-prompt" icon="message-lines">
    Wie der System-Prompt erstellt wird und was er bei jedem Durchlauf injiziert.
  </Card>
  <Card title="Agentenschleife" href="/de/concepts/agent-loop" icon="arrows-rotate">
    Der vollständige Ausführungszyklus des Agenten von der eingehenden Nachricht bis zur endgültigen Antwort.
  </Card>
</CardGroup>
