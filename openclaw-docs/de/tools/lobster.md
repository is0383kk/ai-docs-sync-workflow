---
read_when:
    - Sie möchten deterministische mehrstufige Workflows mit ausdrücklichen Genehmigungen.
    - Sie müssen einen Workflow fortsetzen, ohne frühere Schritte erneut auszuführen.
summary: Typisierte Workflow-Laufzeit für OpenClaw mit fortsetzbaren Freigabeschranken.
title: Lobster
x-i18n:
    generated_at: "2026-07-26T18:51:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 85b7900f86bfedc9d73fcc91c3d0dac37b81f7413b1e68c54dd8a797b70f79fc
    source_path: tools/lobster.md
    workflow: 16
---

Lobster führt mehrstufige Tool-Pipelines als einen einzigen deterministischen Tool-Aufruf aus, mit
expliziten Genehmigungsprüfpunkten und Fortsetzungstoken. Es befindet sich eine Ebene über
entkoppelter Hintergrundarbeit: Informationen zur Orchestrierung von Abläufen über viele entkoppelte Aufgaben hinweg
finden Sie unter [Task Flow](/de/automation/taskflow) (`openclaw tasks flow`); das Aktivitätsprotokoll für Aufgaben
finden Sie unter [Hintergrundaufgaben](/de/automation/tasks).

## Warum

Ohne Lobster erfordert ein mehrstufiger Auftrag viele Tool-Aufrufe mit Hin- und Rückübertragung, wobei das
Modell jeden Schritt orchestriert. Lobster verlagert diese Orchestrierung in eine typisierte
Laufzeitumgebung:

- **Ein Aufruf statt vieler**: Ein einzelner Lobster-Tool-Aufruf gibt ein strukturiertes
  Ergebnis für die gesamte Pipeline zurück.
- **Integrierte Genehmigungen**: Seiteneffekte (Senden, Veröffentlichen, Löschen) halten den Workflow an,
  bis sie ausdrücklich genehmigt werden.
- **Fortsetzbar**: Ein angehaltener Workflow gibt ein Token zurück; genehmigen Sie ihn und setzen Sie ihn fort, ohne
  frühere Schritte erneut auszuführen.

Lobster ist eine kleine, eingeschränkte DSL und keine universelle Skriptsprache:
Genehmigen/Fortsetzen ist ein dauerhaftes, integriertes Primitiv; Pipelines sind Daten (einfach zu
protokollieren, vergleichen, wiederholen und prüfen); die kompakte Grammatik begrenzt „kreative“ Codepfade, sodass
die Validierung realistisch bleibt; Zeitüberschreitungen, Ausgabebegrenzungen, Sandbox-Prüfungen und
Positivlisten werden von der Laufzeitumgebung durchgesetzt, nicht von jedem einzelnen Skript. Jeder Schritt kann weiterhin
eine beliebige CLI oder ein beliebiges Skript aufrufen – generieren Sie bei Bedarf `.lobster`-Dateien mit anderen Werkzeugen,
wenn Sie eine ausdrucksstärkere Autorensprache verwenden möchten.

Ohne Lobster sieht eine wiederkehrende E-Mail-Triage wie folgt aus:

```text
Benutzer: „Prüfe meine E-Mails und entwirf Antworten“
→ openclaw ruft gmail.list auf
→ LLM fasst zusammen
→ Benutzer: „Entwirf Antworten auf Nr. 2 und Nr. 5“
→ LLM erstellt Entwürfe
→ Benutzer: „Sende Nr. 2“
→ openclaw ruft gmail.send auf
(täglich wiederholen, ohne Erinnerung daran, was triagiert wurde)
```

Mit Lobster besteht derselbe Auftrag aus einem Aufruf, der zur Genehmigung anhält und anschließend fortgesetzt wird:

```json
{ "action": "run", "pipeline": "email.triage --limit 20", "timeoutMs": 30000 }
```

```json
{
  "ok": true,
  "status": "needs_approval",
  "output": [{ "summary": "5 benötigen Antworten, 2 erfordern Maßnahmen" }],
  "requiresApproval": {
    "type": "approval_request",
    "prompt": "2 Antwortentwürfe senden?",
    "items": [],
    "resumeToken": "..."
  }
}
```

## Funktionsweise

OpenClaw führt Lobster-Workflows **prozessintern** aus und verwendet dabei das gebündelte
`@clawdbot/lobster`-Paket als eingebetteten Runner. Es wird kein externer `lobster`-
Unterprozess gestartet; der Tool-Aufruf gibt direkt einen JSON-Umschlag zurück. Wenn die
Pipeline zur Genehmigung angehalten wird, enthält der Umschlag ein Fortsetzungstoken (oder eine kurze
Genehmigungs-ID), sodass sie später fortgesetzt werden kann.

## Aktivierung

Lobster ist ein **optionales** Plugin-Tool und standardmäßig nicht aktiviert. Es wird
gebündelt ausgeliefert, daher ist kein separater Installationsschritt erforderlich – erlauben Sie einfach das Tool:

```json
{
  "tools": {
    "alsoAllow": ["lobster"]
  }
}
```

Oder pro Agent:

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": {
          "alsoAllow": ["lobster"]
        }
      }
    ]
  }
}
```

<Note>
`alsoAllow` fügt `lobster` zusätzlich zum aktiven Tool-Profil hinzu, ohne
andere Kern-Tools einzuschränken. Verwenden Sie stattdessen nur `tools.allow`, wenn Sie einen restriktiven
Positivlistenmodus wünschen.
</Note>

Das Tool ist in Sandbox-Tool-Kontexten vollständig deaktiviert.

Wenn Sie die eigenständige Lobster-CLI für die Entwicklung oder externe Pipelines benötigen
(außerhalb des eingebetteten Gateway-Runners), installieren Sie sie aus dem
[Lobster-Repository](https://github.com/openclaw/lobster) und fügen Sie `lobster` zu
`PATH` hinzu.

## Muster: kleine CLI + JSON-Pipes + Genehmigungen

Erstellen Sie kleine Befehle, die JSON verarbeiten, und verketten Sie sie anschließend in einem Lobster-Aufruf.
(Die folgenden Befehlsnamen sind Beispiele – ersetzen Sie sie durch Ihre eigenen.)

```bash
inbox list --json
inbox categorize --json
inbox apply --json
```

```json
{
  "action": "run",
  "pipeline": "exec --json --shell 'inbox list --json' | exec --stdin json --shell 'inbox categorize --json' | exec --stdin json --shell 'inbox apply --json' | approve --preview-from-stdin --limit 5 --prompt 'Änderungen anwenden?'",
  "timeoutMs": 30000
}
```

Wenn die Pipeline eine Genehmigung anfordert, setzen Sie sie mit dem Token fort:

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

Beispiel: Eingabeelemente auf Tool-Aufrufe abbilden:

```bash
gog.gmail.search --query 'newer_than:1d' \
  | openclaw.invoke --tool message --action send --each --item-key message --args-json '{"provider":"telegram","to":"..."}'
```

## Reine JSON-LLM-Schritte (llm-task)

Aktivieren Sie für einen **strukturierten LLM-Schritt** innerhalb eines Workflows das optionale
Plugin-Tool `llm-task` und rufen Sie es aus Lobster auf:

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  },
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": { "alsoAllow": ["llm-task"] }
      }
    ]
  }
}
```

### Wichtige Einschränkung: eingebettetes Lobster gegenüber `openclaw.invoke`

Das gebündelte Lobster-Plugin führt Workflows **prozessintern** innerhalb des Gateway aus.
In diesem eingebetteten Modus übernimmt `openclaw.invoke` **nicht** automatisch einen
Gateway-URL-/Authentifizierungskontext für verschachtelte OpenClaw-CLI-Tool-Aufrufe.

Daher ist dieses Muster **derzeit im eingebetteten Runner nicht zuverlässig**:

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{ ... }'
```

Verwenden Sie das folgende Beispiel nur, wenn Sie die **eigenständige Lobster-CLI** in einer
Umgebung ausführen, in der `openclaw.invoke` bereits mit dem richtigen
Gateway-/Authentifizierungskontext konfiguriert ist.

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "Gib anhand der eingegebenen E-Mail die Absicht und einen Entwurf zurück.",
  "thinking": "low",
  "input": { "subject": "Hallo", "body": "Können Sie helfen?" },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

Wenn Sie derzeit das eingebettete Lobster-Plugin verwenden, bevorzugen Sie entweder:

- einen direkten Aufruf des Tools `llm-task` außerhalb von Lobster oder
- Schritte ohne `openclaw.invoke` innerhalb der Lobster-Pipeline, bis eine unterstützte
  eingebettete Brücke hinzugefügt wird.

Details und Konfigurationsoptionen finden Sie unter [LLM-Aufgabe](/de/tools/llm-task).

## Workflow-Dateien (.lobster)

Lobster kann YAML-/JSON-Workflow-Dateien mit den Feldern `name`, `args`, `steps`, `env`,
`condition` und `approval` ausführen. Legen Sie `pipeline` im Tool-Aufruf auf den Dateipfad fest.

```yaml
name: inbox-triage
args:
  tag:
    default: "family"
steps:
  - id: collect
    command: inbox list --json
  - id: categorize
    command: inbox categorize --json
    stdin: $collect.stdout
  - id: approve
    command: inbox apply --approve
    stdin: $categorize.stdout
    approval: required
  - id: execute
    command: inbox apply --execute
    stdin: $categorize.stdout
    condition: $approve.approved
```

Hinweise:

- `stdin: $step.stdout` und `stdin: $step.json` übergeben die Ausgabe eines vorherigen Schritts.
- `condition` (oder `when`) kann Schritte abhängig von `$step.approved` ausführen.

### Injizierte Umgebungsvariablen

Jede Schritt-Shell übernimmt die übergeordnete Umgebung sowie diese von Lobster injizierten
Variablen, sodass Befehle auf aufgelöste Workflow-Argumente verweisen können, ohne
Rohwerte in die Befehlszeichenfolge einzubetten:

- `LOBSTER_ARG_<NAME>` – eine pro Workflow-Argument. Der Name wird in Großbuchstaben umgewandelt, wobei jede
  Folge nicht alphanumerischer Zeichen zu `_` zusammengefasst wird, sodass das Argument `user-id` zu
  `LOBSTER_ARG_USER_ID` wird.
- `LOBSTER_ARGS_JSON` – alle aufgelösten Argumente als einzelne JSON-Zeichenfolge.

Dies ist der vollständige Satz injizierter Variablen. Es gibt **keine** schrittspezifischen Ausgabevariablen
wie `LOBSTER_STEP_<id>_STDOUT` oder `LOBSTER_STEP_<id>_JSON_<field>`; Shells
behandeln diese Namen als nicht gesetzt, sodass Standardwerte bei der Parametererweiterung den Fehler verbergen können.
Lesen Sie die Ausgabe eines vorherigen Schritts stattdessen über Schrittverweise – `$step.stdout`,
`$step.json` oder `$step.json.<field>` – in einem Wert für `stdin:`, `env:` oder `condition:`.
(`LOBSTER_STATE_DIR` ist eine separate Laufzeiteinstellung für das Zustandsverzeichnis
und kein Argument pro Ausführung.)

## Tool-Parameter

### `run`

```json
{
  "action": "run",
  "pipeline": "gog.gmail.search --query 'newer_than:1d' | email.triage",
  "cwd": "workspace",
  "timeoutMs": 30000,
  "maxStdoutBytes": 512000
}
```

Eine Workflow-Datei mit Argumenten ausführen:

```json
{
  "action": "run",
  "pipeline": "/path/to/inbox-triage.lobster",
  "argsJson": "{\"tag\":\"family\"}"
}
```

| Feld             | Standardwert | Hinweise                                                                                                     |
| ---------------- | ------------ | ------------------------------------------------------------------------------------------------------------ |
| `pipeline`       | erforderlich | Inline-Pipeline-Zeichenfolge oder ein auf `.lobster`/`.yaml`/`.yml`/`.json` endender Pfad zu einer Workflow-Datei. |
| `cwd`            | Gateway-Arbeitsverzeichnis | Relatives Arbeitsverzeichnis; muss innerhalb des Gateway-Arbeitsverzeichnisses aufgelöst werden (absolute Pfade werden abgelehnt). |
| `timeoutMs`      | `20000`     | Bricht die Ausführung ab, wenn der Wert überschritten wird.                                                  |
| `maxStdoutBytes` | `512000`    | Bricht die Ausführung ab, wenn die erfasste Standardausgabe oder Standardfehlerausgabe diese Größe überschreitet. |
| `argsJson`       | -            | JSON-Zeichenfolge mit Argumenten für eine Workflow-Datei (wird bei Inline-Pipelines ignoriert).               |

### `resume`

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

`resume` akzeptiert entweder `token` (das vollständige Fortsetzungstoken aus `requiresApproval`)
oder `approvalId` (die kurze ID aus demselben Objekt) – verwenden Sie den Wert, den die angehaltene
Ausführung zurückgegeben hat. `approve` ist erforderlich.

### Verwalteter Task-Flow-Modus

Die Übergabe von `flowControllerId` und `flowGoal` an `run` (oder `flowId` und
`flowExpectedRevision` an `resume`) führt den Aufruf über die verwaltete
[Task-Flow](/de/automation/taskflow)-API der Plugin-Laufzeitumgebung aus, anstatt
einen einfachen Umschlag zurückzugeben: OpenClaw erstellt einen dauerhaften Flow-Datensatz oder setzt ihn fort, wendet den
Lobster-Umschlag darauf an (`waiting` bei einer Genehmigung, `succeeded`/`failed` bei
Abschluss) und gibt `{ ok, envelope, flow, mutation }` zurück. Dieser Modus erfordert
eine gebundene Task-Flow-Laufzeitumgebung und ist für Plugin-/Controller-Code vorgesehen, der
einen dauerhaften Flow-Zustand über Gateway-Neustarts hinweg benötigt, nicht für die typische spontane Agent-Nutzung.

## Ausgabeumschlag

Lobster gibt einen JSON-Umschlag mit einem von drei Statuswerten zurück:

- `ok` – erfolgreich abgeschlossen
- `needs_approval` – pausiert; `requiresApproval` enthält ein `resumeToken` und eine
  kurze `approvalId`, von denen beide die Ausführung fortsetzen können
- `cancelled` – ausdrücklich abgelehnt oder abgebrochen

Das Tool stellt den Umschlag sowohl in `content` (formatiertes JSON) als auch in `details`
(Rohobjekt) bereit.

## Genehmigungen

Wenn `requiresApproval` vorhanden ist, prüfen Sie die Aufforderung und entscheiden Sie:

- `approve: true` – fortsetzen und mit Seiteneffekten fortfahren
- `approve: false` – abbrechen und den Workflow abschließen

Verwenden Sie `approve --preview-from-stdin --limit N`, um Genehmigungsanfragen eine JSON-Vorschau
ohne benutzerdefinierte jq-/Heredoc-Verknüpfung hinzuzufügen. Der Fortsetzungszustand wird als
kleine JSON-Dateien im Lobster-Zustandsverzeichnis gespeichert (standardmäßig `~/.lobster/state`,
überschreibbar mit `LOBSTER_STATE_DIR`); das Token selbst codiert nur einen
Verweis auf diesen Zustand, nicht den vollständigen Pipeline-Zustand.

## OpenProse

OpenProse lässt sich gut mit Lobster kombinieren: Verwenden Sie `/prose`, um die Vorbereitung durch mehrere Agenten
zu orchestrieren, und führen Sie anschließend eine Lobster-Pipeline für deterministische Genehmigungen aus. Wenn ein Prose-
Programm Lobster benötigt, erlauben Sie das Tool `lobster` für Unteragenten über
`tools.subagents.tools`. Siehe [OpenProse](/de/prose).

## Sicherheit

- **Nur lokal innerhalb des Prozesses** – Workflows werden innerhalb des Gateway-Prozesses ausgeführt; keine
  Netzwerkaufrufe durch das Plugin selbst.
- **Keine Secrets** – Lobster verwaltet OAuth nicht, sondern ruft OpenClaw-Tools auf, die
  dies übernehmen.
- **Sandbox-kompatibel** – deaktiviert, wenn der Tool-Kontext in einer Sandbox ausgeführt wird.
- **Gehärtet** – Zeitüberschreitungen und Ausgabelimits werden durch den eingebetteten Runner erzwungen.

## Fehlerbehebung

| Fehler                                                         | Ursache / Behebung                                                                      |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `lobster runtime timed out`                                   | Die Pipeline hat `timeoutMs` überschritten. Erhöhen Sie den Wert oder teilen Sie die Pipeline auf.                |
| `lobster stdout exceeded maxStdoutBytes` (oder `stderr`)        | Die erfasste Ausgabe hat das Limit überschritten. Erhöhen Sie `maxStdoutBytes` oder reduzieren Sie die Ausgabe.       |
| `run --args-json must be valid JSON`                          | `argsJson` (Ausführungen mit Workflow-Dateien) konnte nicht geparst werden. Korrigieren Sie die JSON-Zeichenfolge.            |
| `lobster runtime failed` (oder eine andere `runtime_error`-Meldung) | Die eingebettete Laufzeit hat eine Fehlerhülle zurückgegeben. Prüfen Sie die Gateway-Protokolle auf Details. |

## Weitere Informationen

- [Plugins](/de/tools/plugin)
- [Erstellung von Plugin-Tools](/de/plugins/building-plugins#registering-agent-tools)

## Fallstudie: Community-Workflows

Ein öffentliches Beispiel: eine „Second Brain“-CLI mit Lobster-Pipelines, die drei
Markdown-Vaults (persönlich, Partner, gemeinsam) verwalten. Die CLI gibt JSON für Statistiken,
Posteingangslisten und Prüfungen auf veraltete Inhalte aus; Lobster verkettet diese Befehle zu Workflows
wie `weekly-review`, `inbox-triage`, `memory-consolidation` und
`shared-task-sync`, jeweils mit Genehmigungsschranken. KI übernimmt, sofern verfügbar, die Beurteilung
(Kategorisierung) und greift andernfalls auf deterministische Regeln
zurück.

- Thread: [https://x.com/plattenschieber/status/2014508656335770033](https://x.com/plattenschieber/status/2014508656335770033)
- Repository: [https://github.com/bloomedai/brain-cli](https://github.com/bloomedai/brain-cli)

## Verwandte Themen

- [Automatisierung](/de/automation) – alle Automatisierungsmechanismen
- [Tool-Übersicht](/de/tools) – alle verfügbaren Agent-Tools
