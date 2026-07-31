---
read_when:
    - Fehlerbehebung, warum ein Agent auf eine bestimmte Weise geantwortet hat, fehlgeschlagen ist oder Tools aufgerufen hat
    - Exportieren eines Support-Pakets für eine OpenClaw-Sitzung
    - Untersuchung von Prompt-Kontext, Tool-Aufrufen, Laufzeitfehlern oder Nutzungsmetadaten
    - Deaktivieren der Trajektorienerfassung
summary: Bereinigte Verlaufsbündel zum Debuggen einer OpenClaw-Agentensitzung exportieren
title: Trajektorienbündel
x-i18n:
    generated_at: "2026-07-26T18:53:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7fc494732b6239ad4ea58dca3920a47cb7433c680e7566855dd265c986b55e74
    source_path: tools/trajectory.md
    workflow: 16
---

Die Trajektorienerfassung ist OpenClaws Flugschreiber für einzelne Sitzungen. Sie zeichnet für jeden Agentenlauf eine
strukturierte Zeitleiste auf. Anschließend verpackt `/export-trajectory` die
aktuelle Sitzung in ein redigiertes Support-Paket, das Folgendes umfasst:

- Den Prompt, den System-Prompt und die an das Modell gesendeten Tools
- Welche Transkriptnachrichten und Tool-Aufrufe zu einer Antwort geführt haben
- Ob der Lauf wegen einer Zeitüberschreitung beendet oder abgebrochen wurde, eine Compaction erfolgte oder ein Provider-Fehler auftrat
- Welche Modelle, Plugins, Skills und Laufzeiteinstellungen aktiv waren
- Vom Provider zurückgegebene Nutzungs- und Prompt-Cache-Metadaten

Beginnen Sie für einen umfassenden Gateway-Supportbericht stattdessen mit
[`/diagnostics`](/de/gateway/diagnostics#chat-command). Damit wird das
bereinigte Gateway-Paket erfasst und bei OpenAI-Codex-Harness-Sitzungen kann nach
Genehmigung Codex-Feedback an OpenAI gesendet werden. Verwenden Sie `/export-trajectory`, wenn Sie die
detaillierte sitzungsspezifische Zeitleiste der Prompts, Tools und Transkripte benötigen.

## Schnellstart

Senden Sie in der aktiven Sitzung (Alias `/trajectory`):

```text
/export-trajectory
```

OpenClaw schreibt das Paket in den Workspace:

```text
.openclaw/trajectory-exports/openclaw-trajectory-<session>-<timestamp>/
```

Geben Sie einen relativen Namen für das Ausgabeverzeichnis an, um diesen zu überschreiben:

```text
/export-trajectory bug-1234
```

Der Name wird innerhalb von `.openclaw/trajectory-exports/` aufgelöst. Absolute Pfade und
`~`-Pfade werden abgelehnt.

Trajektorienpakete können Prompts, Modellnachrichten, Tool-Schemas, Tool-Ergebnisse,
Laufzeitereignisse und lokale Pfade enthalten. Daher durchläuft der Chatbefehl immer
die Ausführungsgenehmigung. Genehmigen Sie den Export einmalig, wenn Sie das
Paket erstellen möchten; verwenden Sie nicht „Alle zulassen“. In Gruppenchats sendet OpenClaw die
Genehmigungsaufforderung und das Exportergebnis privat an den Eigentümer, statt Details zur Trajektorie
im gemeinsamen Raum zu veröffentlichen.

Führen Sie für die lokale Prüfung oder Support-Abläufe den zugrunde liegenden CLI-Befehl
direkt aus:

```bash
openclaw sessions export-trajectory --session-key "agent:main:telegram:direct:123" --workspace .
```

Weitere Flags: `--output <path>` (Verzeichnisname innerhalb von
`.openclaw/trajectory-exports`), `--store <path>` (Überschreibung des Sitzungsspeichers),
`--agent <id>` (Agenten-ID für die Speicherauflösung), `--json` (strukturierte Ausgabe).

## Zugriff

Der Trajektorienexport ist ein Eigentümerbefehl. Der Absender muss sowohl die normalen
Autorisierungsprüfungen für Befehle als auch die Eigentümerprüfung für den Kanal bestehen.

## Was aufgezeichnet wird

Die Trajektorienerfassung ist für OpenClaw-Agentenläufe standardmäßig aktiviert.

Zu den Laufzeitereignissen gehören:

- `session.started`
- `trace.metadata`
- `context.compiled`
- `prompt.submitted`
- `model.fallback_step`, einschließlich Quellmodell, nächstem Modell, Fehlergrund/-details, Position in der Kette sowie der Angabe, ob die Kette fortgeschritten ist, erfolgreich war oder ausgeschöpft wurde
- `model.completed`
- `trace.artifacts`
- `session.ended`

Transkriptereignisse werden aus dem aktiven Sitzungszweig rekonstruiert: Benutzer-
nachrichten, Assistentennachrichten, Tool-Aufrufe, Tool-Ergebnisse, Compactions, Modell-
änderungen, Bezeichnungen und benutzerdefinierte Sitzungseinträge.

Ereignisse werden als JSON Lines mit dieser Schemamarkierung geschrieben:

```json
{
  "traceSchema": "openclaw-trajectory",
  "schemaVersion": 1
}
```

## Paketdateien

| Datei                  | Inhalt                                                                                       |
| --------------------- | ---------------------------------------------------------------------------------------------- |
| `manifest.json`       | Paketschema, Quelldateien, Ereigniszahlen und Liste der generierten Dateien                             |
| `events.jsonl`        | Geordnete Laufzeit- und Transkriptzeitleiste                                                        |
| `session-branch.json` | Redigierter aktiver Transkriptzweig und Sitzungskopf                                           |
| `metadata.json`       | OpenClaw-Version, Betriebssystem/Laufzeit, Modell, Konfigurations-Snapshot, Plugins, Skills und Prompt-Metadaten     |
| `artifacts.json`      | Endstatus, Fehler, Nutzung, Prompt-Cache, Anzahl der Compactions, Assistententext und Tool-Metadaten |
| `prompts.json`        | Übermittelte Prompts und ausgewählte Details zur Prompt-Erstellung                                         |
| `system-prompt.txt`   | Zuletzt kompilierter System-Prompt, sofern erfasst                                                   |
| `tools.json`          | An das Modell gesendete Tool-Definitionen, sofern erfasst                                              |

`manifest.json` listet die in einem bestimmten Paket vorhandenen Dateien auf; einige Dateien werden
ausgelassen, wenn in der Sitzung die entsprechenden Laufzeitdaten nicht erfasst wurden.

## Erfassungsspeicher

Laufzeit-Trajektorienereignisse werden zusammen mit der Sitzung in der agentenspezifischen SQLite-
Datenbank gespeichert. Beim Export einer Trajektorie wird ein redigiertes JSONL-Support-Paket materialisiert;
die aktive Laufzeiterfassung ist keine sitzungsnahe JSONL-Sidecar-Datei.

Ältere Dateien vom Typ `.trajectory.jsonl` und `.trajectory-path.json` können weiterhin
aus älteren Versionen oder expliziten Legacy-Dateiexporten vorhanden sein. Die Sitzungswartung behandelt
diese Dateien als Bereinigungsziele; die aktive Erfassung schreibt Datenbankzeilen.

## Erfassung deaktivieren

```bash
export OPENCLAW_TRAJECTORY=0
```

Dadurch wird die Laufzeit-Trajektorienerfassung vor dem Start von OpenClaw deaktiviert.
`/export-trajectory` kann weiterhin den Transkriptzweig exportieren, doch ausschließlich
zur Laufzeit vorhandene Daten wie kompilierter Kontext, Provider-Artefakte und Prompt-Metadaten können
fehlen.

## Flush-Zeitüberschreitung anpassen

OpenClaw schreibt Laufzeit-Trajektorienzeilen während der Agentenbereinigung endgültig weg. Die standardmäßige
Zeitüberschreitung für die Bereinigung beträgt 10,000 ms. Legen Sie bei langsamen Datenträgern oder großen Speichern
`OPENCLAW_TRAJECTORY_FLUSH_TIMEOUT_MS` vor dem Start von OpenClaw fest:

```bash
export OPENCLAW_TRAJECTORY_FLUSH_TIMEOUT_MS=30000
```

Damit wird gesteuert, wann OpenClaw eine `openclaw-trajectory-flush`-Zeitüberschreitung protokolliert und
fortfährt; die Größenlimits der Trajektorie werden dadurch nicht geändert. Um alle Schritte der
Agentenbereinigung anzupassen, die keine explizite Zeitüberschreitung übergeben, legen Sie
`OPENCLAW_AGENT_CLEANUP_TIMEOUT_MS` fest.

## Datenschutz und Limits

Trajektorienpakete sind für Support und Debugging bestimmt, nicht für die öffentliche Veröffentlichung. OpenClaw
redigiert vertrauliche Werte, bevor Exportdateien geschrieben werden:

- Anmeldedaten und bekannte geheimnisartige Nutzdatenfelder
- Bilddaten
- Lokale Zustandspfade
- Workspace-Pfade, ersetzt durch `$WORKSPACE_DIR`
- Pfade des Home-Verzeichnisses, sofern erkannt

Der Exporter begrenzt außerdem die Eingabegröße:

- Laufzeiterfassung: Die aktive Erfassung ist ein rollierendes Fenster mit einem Limit von 10 MiB, wobei die ältesten Ereignisse verworfen werden, um Platz für neue zu schaffen; der Export akzeptiert vorhandene Legacy-Laufzeit-Sidecar-Dateien bis zu 50 MiB
- Sitzungsdateien: 50 MiB
- Laufzeitereignisse pro Export: 200,000
- Exportierte Ereignisse insgesamt: 250,000
- Einzelne Laufzeitereigniszeilen werden oberhalb von 256 KiB gekürzt

Prüfen Sie Pakete, bevor Sie sie außerhalb Ihres Teams teilen. Die Redaktion erfolgt nach bestem Bemühen
und kann nicht jedes anwendungsspezifische Geheimnis erkennen.

## Fehlerbehebung

Wenn der Export keine Laufzeitereignisse enthält:

- Vergewissern Sie sich, dass OpenClaw ohne `OPENCLAW_TRAJECTORY=0` gestartet wurde
- Führen Sie eine weitere Nachricht in der Sitzung aus und exportieren Sie dann erneut
- Prüfen Sie `manifest.json` auf `runtimeEventCount`

Wenn der Befehl den Ausgabepfad ablehnt:

- Verwenden Sie einen relativen Namen wie `bug-1234`
- Übergeben Sie weder `/tmp/...` noch `~/...`
- Belassen Sie den Export innerhalb von `.openclaw/trajectory-exports/`

Wenn der Export aufgrund eines Größenfehlers fehlschlägt, hat die Sitzung oder Sidecar-Datei die
oben genannten Sicherheitslimits für den Export überschritten. Starten Sie eine neue Sitzung oder exportieren Sie eine kleinere
Reproduktion.

## Verwandte Themen

- [Diffs](/de/tools/diffs)
- [Sitzungsverwaltung](/de/concepts/session)
- [Ausführungs-Tool](/de/tools/exec)
