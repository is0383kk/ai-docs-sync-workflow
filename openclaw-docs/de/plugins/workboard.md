---
read_when:
    - Sie möchten ein Arbeitsboard im Kanban-Stil in der Control UI
    - Sie aktivieren oder deaktivieren das gebündelte Workboard-Plugin
    - Sie möchten geplante Agentenarbeit ohne einen externen Projektmanager nachverfolgen
summary: Optionales Dashboard-Arbeitsboard für agenteneigene Karten und Sitzungsübergabe
title: Workboard-Plugin
x-i18n:
    generated_at: "2026-07-26T18:39:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ec05c990c3559015780d9cb80f3ceedd7cc79db89ccf1afd65906c8c7630331
    source_path: plugins/workboard.md
    workflow: 16
---

Das Workboard-Plugin fügt dem
[Control UI](/de/web/control-ui) ein optionales Board im Kanban-Stil hinzu: Arbeitskarten in Agentengröße, Zuweisung zu Agenten
und einen Link zurück zur Aufgabe, zum Lauf und zur Dashboard-Sitzung der Karte.

Workboard ist bewusst klein gehalten: Es verfolgt lokale Betriebsarbeit für ein
OpenClaw Gateway. Es ersetzt weder GitHub Issues, Linear, Jira noch
andere Projektmanagementsysteme für Teams.

## Aktivierung

Workboard ist enthalten, aber standardmäßig deaktiviert:

1. Öffnen Sie **Plugins** im Control UI oder verwenden Sie `/settings/plugins` relativ zum
   konfigurierten Basispfad des Control UI. Beispielsweise verwendet ein Basispfad von `/openclaw`
   den Pfad `/openclaw/settings/plugins`.
2. Suchen Sie **Workboard** und wählen Sie **Aktivieren**. Da Workboard in
   OpenClaw enthalten ist, ist keine Aktion zum **Installieren** erforderlich.
3. Wenn die Benutzeroberfläche meldet, dass ein Neustart erforderlich ist, starten Sie das Gateway neu.

Die Registerkarte „Workboard“ erscheint in der Dashboard-Navigation, nachdem die Plugin-Laufzeit geladen wurde.
Solange es deaktiviert ist, bleibt die Registerkarte in der Navigation ausgeblendet. Wenn die Route
`/workboard` direkt geöffnet wird, während das Plugin deaktiviert oder durch
`plugins.allow`/`plugins.deny` blockiert ist, wird anstelle der Kartendaten
ein Zustand angezeigt, der auf die Nichtverfügbarkeit des Plugins hinweist.

Der entsprechende CLI-Ablauf lautet:

```bash
openclaw plugins enable workboard
openclaw gateway restart
openclaw dashboard
```

## Konfiguration

Workboard besitzt keine Plugin-spezifische Konfiguration. Aktivieren oder deaktivieren Sie es über den standardmäßigen
Plugin-Eintrag:

```json5
{
  plugins: {
    entries: {
      workboard: {
        enabled: true,
        config: {},
      },
    },
  },
}
```

```bash
openclaw plugins disable workboard
openclaw gateway restart
```

## Kartenfelder

| Feld        | Werte                                                                                                         |
| ----------- | ------------------------------------------------------------------------------------------------------------- |
| `status`    | `triage`, `backlog`, `todo`, `scheduled`, `ready`, `running`, `review`, `blocked`, `done`                     |
| `priority`  | `low`, `normal`, `high`, `urgent`                                                                             |
| `labels`    | frei formulierte Zeichenfolgen                                                                                |
| `agentId`   | optional zugewiesener Agent                                                                                   |
| verknüpfte Referenzen | optionale Aufgabe, optionaler Lauf, optionale Sitzung oder Quell-URL                                          |
| `execution` | optionale Metadaten für einen von der Karte gestarteten Codex-/Claude-Lauf (Engine, Modus, Modell, Sitzung, Lauf-ID, Status) |

Karten enthalten außerdem kompakte Metadaten zu Versuchen, Kommentaren, Links, Nachweisen,
Artefakten, Automatisierungseinstellungen, Anhängen, Worker-Protokollen, Worker-Protokollstatus,
Ansprüchen, Diagnosen, Benachrichtigungen, Vorlagen-ID, Archivstatus und
Erkennung veralteter Sitzungen sowie eine Liste der letzten Ereignisse (`created`, `edited`,
`moved`, `linked`, `specified`, `decomposed`, `claimed`, `heartbeat`,
`execution_updated`, `attempt_started`, `attempt_updated`, `comment_added`,
`link_added`, `proof_added`, `artifact_added`, `attachment_added`,
`diagnostic`, `notification`, `dispatch`, `orchestration`,
`protocol_violation`, `archived`, `unarchived`, `stale`). Anhand dieser Metadaten kann ein
Operator nachvollziehen, wie sich eine Karte durch das Board bewegt hat, ohne die verknüpfte
Sitzung zu öffnen. Sie bilden lokalen Betriebskontext ab und ersetzen weder
Sitzungstranskripte noch den Verlauf eines GitHub-Issues.

Das Plugin und das Control UI verwenden denselben Workboard-Kartenvertrag. Dashboard-Aktualisierungen
bewahren daher die Herkunft und Autorität des Arbeitsbereichs, den Anspruchsstatus, Diagnoseaktionen
und Sequenznummern von Benachrichtigungen, anstatt eine kleinere, ausschließlich für die
Benutzeroberfläche bestimmte Kopie der Karte zu projizieren. Unbekannte Diagnosearten, Diagnoseschweregrade und
Benachrichtigungsarten werden ignoriert, bis beide Oberflächen sie unterstützen; sie werden niemals
in einen anderen gültigen Zustand umgeschrieben.

Das geöffnete Dashboard wird durch `plugin.workboard.changed`-Invalidierungen aktualisiert. Jedes
Ereignis enthält nur eine Store-Epoche und Revision; die Benutzeroberfläche liest anschließend die kanonischen
Karten über den normalen `operator.read`-RPC erneut ein. Mehrere Revisionen werden zu
einem einzigen nachfolgenden Lesevorgang zusammengefasst. Workboard verschiebt diesen Lesevorgang, während eine Karte gezogen,
bearbeitet oder geschrieben wird, und setzt ihn fort, nachdem die lokale Interaktion abgeschlossen ist. Bei einer
erneuten Verbindung erfolgt immer ein kanonisches Neuladen. Es gibt keine routinemäßige Abfrage aller Karten,
und **Aktualisieren** bleibt als manuelle Wiederherstellungsoption verfügbar.

Wenn mehr als ein Board vorhanden ist, enthält die Symbolleiste einen **Board**-Filter, der auf
persistierten Board-Metadaten statt nur auf den derzeit sichtbaren Karten basiert. Leere
und archivierte Boards bleiben daher auswählbar. Karten ohne explizite
Board-ID gehören zum kanonischen Board `default`. Jedes Board besitzt eine kanonische
Seite `/workboard/<boardId>`, die als Lesezeichen gespeichert, geteilt oder in der
Seitenleiste angeheftet werden kann. Die zuvor ausgelieferte Form `/workboard?board=<boardId>` bleibt als
Kompatibilitätsalias bestehen und leitet unter Beibehaltung anderer Abfrageparameter auf diese Seite weiter.
Durch Auswahl von **Alle Boards** kehren Sie zu `/workboard` zurück.

Karten werden im eigenen Gateway-Status des Plugins gespeichert und gemeinsam mit dem übrigen
OpenClaw-Status dieses Gateways verschoben (siehe [Speicherung](#storage)).

## Arbeit von einer Karte aus starten

Nicht verknüpfte Karten können die Arbeit direkt starten:

- **Codex ausführen** / **Claude ausführen** startet mit einer expliziten
  Engine einen aufgabenverfolgten Agentenlauf, sendet den Prompt der Karte und markiert die Karte als `running`. Codex-
  Läufe verwenden `openai/gpt-5.6-sol`; Claude-Läufe verwenden `anthropic/claude-sonnet-4-6`.
- **Codex öffnen** / **Claude öffnen** erstellt eine verknüpfte Dashboard-Sitzung, ohne
  den Prompt der Karte zu senden oder die Karte zu verschieben, für manuelle Arbeit, die mit dem
  Board verknüpft bleibt.

Autonome Starts verwenden den Pfad des Gateways für aufgabenverfolgte Agentenläufe (standardmäßig Agent
und Modell, sofern Codex/Claude nicht ausdrücklich ausgewählt wird); Workboard verknüpft anschließend die
resultierende Aufgabe, Lauf-ID und den Sitzungsschlüssel mit der Karte. Jede verknüpfte
Ausführung zeichnet außerdem eine Versuchszusammenfassung auf (Engine, Modus, Modell, Lauf-ID,
Zeitstempel, Status, fortlaufende Fehleranzahl), sodass wiederholte Fehler sichtbar bleiben.

Das Dashboard aktualisiert den Aufgabenstatus aus dem Aufgabenjournal des Gateways und ordnet
Aufgaben anhand der Aufgaben-ID, Lauf-ID oder des verknüpften Sitzungsschlüssels den Karten zu. Eine in der Warteschlange befindliche oder laufende
Aufgabe hält den Lebenszyklus der Karte aktiv; eine abgeschlossene, fehlgeschlagene, wegen Zeitüberschreitung beendete oder
abgebrochene Aufgabe verschiebt die Karte gemäß derselben Synchronisierungsregel wie bei verknüpften Sitzungen in Richtung `review` oder `blocked`
(siehe [Synchronisierung des Sitzungslebenszyklus](#session-lifecycle-sync)).

## Agentenwerkzeuge

| Tool                                                                                                                                             | Zweck                                                                                                                                                                                   |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `workboard_list`                                                                                                                                 | Kompakte Karten mit Anspruchs-/Diagnosestatus auflisten; optionaler Board-Filter.                                                                                                                    |
| `workboard_read`                                                                                                                                 | Eine Karte sowie begrenzten Worker-Kontext zurückgeben (Notizen, Versuche, Kommentare, Links, Nachweise, Artefakte, übergeordnete Ergebnisse, aktuelle Arbeit des zugewiesenen Bearbeiters, aktive Diagnosen).                               |
| `workboard_create`                                                                                                                               | Eine Karte mit optionalen übergeordneten Karten, Mandant, Skills, Board, Workspace-Metadaten, Idempotenzschlüssel, Laufzeitlimit und Wiederholungsbudget erstellen.                                                             |
| `workboard_link`                                                                                                                                 | Eine übergeordnete Karte mit einer untergeordneten Karte verknüpfen. Untergeordnete Karten verbleiben in `todo`, bis jede übergeordnete Karte `done` erreicht; anschließend verschiebt die Dispatch-Hochstufung sie nach `ready`.                                                     |
| `workboard_claim`                                                                                                                                | Eine Karte für den aufrufenden Agent beanspruchen; verschiebt `backlog`/`todo`/`ready` nach `running`.                                                                                                        |
| `workboard_heartbeat`                                                                                                                            | Den Anspruchs-Heartbeat während eines längeren Laufs aktualisieren.                                                                                                                                          |
| `workboard_release`                                                                                                                              | Den Anspruch nach Abschluss, Pause oder Übergabe freigeben; kann die Karte in einen Folgestatus verschieben.                                                                                                |
| `workboard_complete` / `workboard_block`                                                                                                         | Strukturierte Lebenszyklus-Tools für abschließende Zusammenfassungen, Nachweise, Artefakte und Manifeste erstellter Karten (müssen auf Karten verweisen, die mit der abgeschlossenen Karte rückverknüpft sind) oder Blockierungsgründe.                 |
| `workboard_attachment_add` / `workboard_attachment_read` / `workboard_attachment_delete`                                                         | Kleine Kartenanhänge im SQLite-Zustand des Plugins speichern, auf der Karte indizieren und im Worker-Kontext bereitstellen.                                                                                         |
| `workboard_worker_log` / `workboard_protocol_violation`                                                                                          | Worker-Protokollzeilen aufzeichnen und eine Karte blockieren, wenn ein automatisierter Worker stoppt, ohne `workboard_complete`/`workboard_block` aufzurufen.                                                           |
| `workboard_board_create` / `workboard_board_archive` / `workboard_board_delete`                                                                  | Persistierte Board-Metadaten verwalten (Anzeigename, Beschreibung, Archivstatus, Standard-Workspace).                                                                                            |
| `workboard_runs`                                                                                                                                 | Den persistierten Verlauf der Ausführungsversuche für eine Karte zurückgeben.                                                                                                                                      |
| `workboard_specify`                                                                                                                              | Eine grobe Triage-/Backlog-Karte in eine präzisierte `todo`-Karte umwandeln; zeichnet die Spezifikationszusammenfassung auf der Karte auf.                                                                                      |
| `workboard_decompose`                                                                                                                            | Eine übergeordnete Orchestrierungskarte in verknüpfte untergeordnete Karten auffächern, wobei Board-/Mandantenmetadaten übernommen werden; kann die übergeordnete Karte mit einem Manifest erstellter Karten abschließen.                                             |
| `workboard_notify_subscribe` / `workboard_notify_list` / `workboard_notify_events` / `workboard_notify_advance` / `workboard_notify_unsubscribe` | Benachrichtigungsabonnements verwalten. Ereignislesevorgänge sind wiederholungssicher; `advance` verschiebt den dauerhaften Cursor, sodass Aufrufer fortfahren können, ohne Ereignisse abgeschlossener/fehlgeschlagener/veralteter Karten zu verlieren oder doppelt zu lesen. |
| `workboard_boards` / `workboard_stats`                                                                                                           | Board-Namespaces und Warteschlangenstatistiken prüfen.                                                                                                                                                 |
| `workboard_promote` / `workboard_reassign` / `workboard_reclaim`                                                                                 | Feststeckende Arbeit wiederherstellen oder übergeben.                                                                                                                                                           |
| `workboard_comment` / `workboard_proof`                                                                                                          | Übergabenotizen hinzufügen oder Nachweis-/Artefaktreferenzen anhängen.                                                                                                                                    |
| `workboard_unblock`                                                                                                                              | Blockierte Arbeit zurück nach `todo` verschieben.                                                                                                                                                         |
| `workboard_move`                                                                                                                                 | Eine Karte in einen anderen Status verschieben; beanspruchte Karten erfordern den Agent-Anspruchsbereich des Aufrufers.                                                                                                      |
| `workboard_dispatch`                                                                                                                             | Die Hochstufung von Abhängigkeiten oder die Bereinigung veralteter Ansprüche anstoßen, ohne Worker zu starten; Worker werden über den Gateway- oder Slash-Command-Dispatch gestartet.                                                        |

Nachweisstatus sind von Workern gemeldete Ergebnisse, keine unabhängige Überprüfung. Ein Eintrag `passed`
bedeutet, dass der Worker den Erfolg seines Befehls oder seiner Prüfung meldet; Verbraucher, die ein
unabhängiges Qualitätstor benötigen, sollten den angehängten Befehl, die URL oder das Artefakt prüfen und
eine eigene Verifizierung ausführen. `workboard_proof` gibt die `proofId` des neuen Datensatzes zurück. Wenn
`workboard_complete` den endgültigen Status desselben Nachweises meldet, übergeben Sie `proofId`, damit der
ausstehende Datensatz an Ort und Stelle aufgelöst wird, ohne seine Identität oder seinen Zeitstempel zu verlieren. Ein Nachweis, der
bereits denselben endgültigen Status hat, wird unverändert wiederverwendet. Abschlussnachweise ohne
`proofId` bleiben nur anhängbar, sodass ein späterer Wiederholungsversuch den älteren Verlauf nicht allein deshalb überschreiben kann,
weil sein Befehl oder seine Notiz identisch ist.

Beanspruchte Karten lehnen Mutationen durch Agent-Tools anderer Agents ab, sofern der Aufrufer
nicht über das von `workboard_claim` zurückgegebene Anspruchstoken verfügt. Jede von einem
Agent-Tool oder Gateway-RPC-Aufruf zurückgegebene Karte schwärzt `metadata.claim.token` zu `[redacted]`
(das Token selbst wird nur einmal auf oberster Ebene und ausschließlich von `workboard_claim` zurückgegeben),
sodass Dashboard-Bediener und andere Agents den Anspruchsstatus prüfen können, ohne jemals
ein verwendbares Token zu sehen. Die Wiederherstellung erfolgt über
`workboard_promote`/`workboard_reassign`/`workboard_reclaim`, wofür das
Token nicht erforderlich ist.

## Dispatch

Der Dispatch ist Gateway-lokal: Er startet keine beliebigen Betriebssystemprozesse. Die normalen
OpenClaw-Subagent-Sitzungen sind weiterhin für die Ausführung zuständig. Ein Dispatch-Durchlauf:

1. Stuft Karten mit erfüllten Abhängigkeiten hoch.
2. Zeichnet Dispatch-Metadaten auf bereiten Karten auf.
3. Blockiert abgelaufene Ansprüche oder Läufe mit Zeitüberschreitung.
4. Markiert gemäß Board-Konfiguration Triage-Karten als Orchestrierungskandidaten.
5. Beansprucht einen kleinen Stapel bereiter Karten und startet Worker-Läufe über die
   Gateway-Subagent-Laufzeit.

Worker erhalten begrenzten Kartenkontext sowie das Anspruchstoken, das benötigt wird, um über die
Workboard-Tools einen Heartbeat zu senden und die Karte abzuschließen oder zu blockieren.

Workspace-Pfade richten sich nach den bestehenden Dateisystemberechtigungen des Aufrufers. Gateway-
Clients mit `operator.write` können konfigurierte Agent-Workspaces verwenden;
`operator.admin`-Clients können andere Host-Checkouts verwenden. Agent-Tools in einer Sandbox nutzen
den Workspace-Zugriff ihrer Sandbox, während nicht in einer Sandbox ausgeführte, auf den Workspace beschränkte Tools ihren
konfigurierten Workspace-Stamm verwenden. Workboard zeichnet diese Berechtigung auf, wenn ein Workspace
zugewiesen wird, und bildet beim Dispatch erneut die Schnittmenge mit den aktuellen Berechtigungen des Aufrufers,
sodass eine persistierte Karte den Zugriff eines späteren Aufrufers nicht erweitern kann. Bei älteren Karten mit einem
expliziten Host-Workspace, aber ohne aufgezeichnete Berechtigung muss dieser Workspace
vor einem Full-Host-Dispatch erneut gespeichert werden; Karten ohne Host-Pfad übernehmen beim
ersten Dispatch die aktuellen Berechtigungen des Aufrufers.

Workspace-gebundener Dispatch akzeptiert ein Verzeichnis oder Git-Checkout nur, wenn dessen
Repository-Stamm genau mit dem Ziel-Workspace des Agents übereinstimmt. Eine Worktree-Anforderung
wird auf dieses Verzeichnis eingegrenzt und als Verzeichnis-Workspace persistiert, sodass der
Host weder den Checkout materialisiert noch Repository-Einrichtungscode ausführt. Der
Ziel-Worker muss eine beschreibbare, nicht gemeinsam genutzte Docker-Sandbox für genau diesen
Workspace verwenden, ohne Ausführung mit erhöhten Berechtigungen, persistierte Host-/Node-Exec-Überschreibungen oder
nicht klassifizierte Plugin- und MCP-Tools. Workboard listet seine registrierten Tools auf,
anstatt einem `workboard_*`-Präfix zu vertrauen, und der Dispatch lehnt einen laufenden Docker-
Container ab, dessen Live-Mount-/Konfigurationshash veraltet ist. Der Dispatch meldet die
inkompatible Zielrichtlinie, anstatt einen weniger streng eingeschränkten Worker zu starten.
Ein Full-Host-Dispatch kann andere lokale Checkouts als Ziel verwenden und behält die normale verwaltete
Worktree-Einrichtung bei.

Workspace-Berechtigungen erzeugen kein zweites Berechtigungsmodell für den Kartenlebenszyklus.
Aufrufer, die Workboard-Karten verändern dürfen, können sie auf jeder Oberfläche manuell durch dieselben
Status verschieben; schreibgeschützter Workspace-Zugriff verhindert lediglich Worker-
Dispatches, die Schreibzugriff benötigen.

### Worker-Auswahl

Jeder Durchlauf startet standardmäßig **höchstens 3 Worker**. Bereite Karten werden nach
Priorität, dann Position und anschließend Erstellungszeit sortiert. Ein Durchlauf startet nur eine Karte pro
Eigentümer/Agent und überspringt Eigentümer, für die bereits laufende oder zu prüfende Arbeit auf dem
Board vorhanden ist. Archivierte Karten, Karten mit einem aktiven Claim und Karten, die sich nicht im Status `ready`
befinden, werden niemals für Worker-Starts ausgewählt (sie können weiterhin von der
Datenseite des Dispatch betroffen sein: Bereinigung veralteter Claims, Hochstufung von Abhängigkeiten, Timeout-
Bereinigung).

Sitzungsschlüssel sind pro Board/Karte deterministisch, sodass wiederholte Dispatches
zurück zur selben Worker-Lane geleitet werden, statt unabhängige Sitzungen zu erstellen:

- Zugewiesene Karten: `agent:<agentId>:subagent:workboard-<boardId>-<cardId>`
- Nicht zugewiesene Karten: `subagent:workboard-<boardId>-<cardId>` (Gateway ermittelt
  den konfigurierten Standard-Agenten)

Wenn ein Worker nach dem Claim einer Karte nicht gestartet werden kann, blockiert Workboard die
Karte, löscht den Claim, zeichnet den Fehler beim Start des Durchlaufs auf und fügt eine Worker-
Protokollzeile hinzu – sichtbar im Dashboard, in CLI-JSON, Agenten-Tools und der Karten-
diagnose.

### Einstiegspunkte

- Dispatch-Aktion im Dashboard
- `openclaw workboard dispatch`
- `/workboard dispatch` in einem befehlsfähigen Kanal

Alle drei verwenden die Subagent-Laufzeit des Gateway, wenn das Gateway verfügbar ist. Die
CLI verfügt über genau einen Operator-Fallback: Wenn der Gateway-Aufruf mit einem
Verbindungs-/Nichtverfügbarkeitsfehler fehlschlägt (oder bei älteren
Gateways mit einem `unknown method`-Fehler) und weder ein explizites Ziel `--url`/`--token` noch ein konfiguriertes entferntes
Gateway (`OPENCLAW_GATEWAY_URL` oder `gateway.mode: remote`) gilt, führt die CLI einen
rein datenbezogenen Dispatch für den lokalen SQLite-Status aus – sie kann Abhängigkeiten hochstufen,
veraltete Claims bereinigen und Durchläufe mit Zeitüberschreitung blockieren, aber keine Worker starten. Authentifizierungs-,
Berechtigungs- und Validierungsfehler eines erreichbaren Gateway werden nicht als
Nichtverfügbarkeit behandelt; sie werden als Befehlsfehler ausgegeben, ebenso jeder Gateway-
Fehler, wenn ein explizites Ziel `--url`/`--token` angegeben wurde.

Board-Metadaten können `autoDecompose`, `autoDecomposePerDispatch`,
`defaultAssignee` und `orchestratorProfile` festlegen. OpenClaw zeichnet diese Absicht auf und
stellt sie im Worker-Kontext bereit; die eigentliche Spezifikation/Zerlegung erfolgt weiterhin
über die normalen Workboard-Tools.

## CLI und Slash-Befehl

```bash
openclaw workboard list [--board <id>] [--status <status>] [--include-archived] [--json]
openclaw workboard create "Fix stale card lifecycle" --priority high --labels bug,workboard
openclaw workboard show <card-id> [--json]
openclaw workboard move <card-id> --status <status> [--json]
openclaw workboard dispatch [--board <id>] [--json]
```

Die Textausgabe von `list` blendet archivierte Karten standardmäßig aus (`--include-archived`
überschreibt dies); `--json` schließt archivierte Karten immer ein, entsprechend dem vollständigen Karten-
Vertrag, den vorhandene Skripte verwenden. `show` und `move` akzeptieren ein eindeutiges ID-
Präfix. `list`, `create`, `show` und `move` lesen/schreiben den lokalen Plugin-
Status immer direkt. Nur `dispatch` ruft das laufende Gateway auf, mit dem oben
beschriebenen Fallback.

Vollständige Flags, JSON-Ausgabe, Gateway-
Fallback-Verhalten, Verarbeitung von ID-Präfixen, Dispatch-Auswahlregeln und
Fehlerbehebung finden Sie unter [Workboard-CLI](/de/cli/workboard).

`/workboard list`, `/workboard show <card-id>`, `/workboard create <title>`,
`/workboard move <card-id> --status <status>` und `/workboard dispatch` entsprechen
der CLI. Auflisten und Anzeigen sind Lesevorgänge für jeden autorisierten Befehlsabsender.
Erstellen, Verschieben und Dispatch erfordern auf Chat-Oberflächen den Eigentümerstatus oder einen Gateway-
Client mit `operator.write`/`operator.admin`. Manuelle Operator-Verschiebungen verwenden dasselbe
Verhalten zum Überschreiben von Claims wie Drag-and-drop im Dashboard. Der Worktree-Zugriff
folgt weiterhin derselben oben beschriebenen Workspace-Grenze.

## Synchronisierung des Sitzungslebenszyklus

Karten können mit einer vorhandenen Dashboard-Sitzung oder einer beim
Arbeitsstart von der Karte erstellten Sitzung verknüpft werden. Verknüpfte Karten zeigen den Sitzungslebenszyklus direkt an:
laufend, veraltet, verknüpft und inaktiv, abgeschlossen, fehlgeschlagen oder nicht vorhanden. Sie können auch eine
vorhandene Sitzung auf der Registerkarte Sessions mit **Add to Workboard** erfassen; die Karte
wird mit dieser Sitzung verknüpft, verwendet die Sitzungsbezeichnung oder die letzte Benutzereingabe als Titel
und füllt die Notizen, sofern verfügbar, mit der letzten Benutzereingabe sowie der neuesten Assistentenantwort
vorab aus.

Wenn die verknüpfte Sitzung nicht mehr vorhanden ist, bleibt die Karte als Kontext verknüpft und
bietet weiterhin Startsteuerelemente, um sie in einer neuen Sitzung neu zu starten. Wenn eine aktive
verknüpfte Sitzung keine aktuellen Aktivitäten mehr meldet, markiert Workboard die Karte als
`stale` und speichert dies als Metadaten, bis der Lebenszyklus es löscht.

Solange sich eine Karte in einem aktiven Arbeitsstatus befindet, folgt Workboard der verknüpften Sitzung:

| Status der verknüpften Sitzung        | Kartenstatus |
| ------------------------------------- | ----------- |
| aktiv                                 | `running`   |
| abgeschlossen                         | `review`    |
| fehlgeschlagen, beendet, Zeitüberschreitung oder abgebrochen | `blocked`   |

**Manuelle Prüfstatus haben Vorrang.** Wenn Sie eine Karte nach `review`, `blocked` oder `done`
verschieben, wird die automatische Synchronisierung für diese Karte beendet, bis Sie sie zurück nach `todo` oder `running` verschieben.

Beim Starten einer Karte werden normale Gateway-Sitzungen verwendet; Workboard speichert nur Karten-
Metadaten und Verknüpfungen. Gesprächstranskript, Modellauswahl und Durchlauf-
Lebenszyklus verbleiben im Besitz des regulären Sitzungssystems. Verwenden Sie **Stop** auf einer aktiven
verknüpften Karte, um den aktiven Durchlauf abzubrechen – Workboard markiert diese Karte als `blocked`, sodass
sie für Folgemaßnahmen sichtbar bleibt.

Neue Karten können aus Workboard-Vorlagen gestartet werden (`bugfix`, `docs`, `release`,
`pr_review`, `plugin`). Vorlagen füllen Titel, Notizen, Bezeichnungen und Priorität vorab aus;
die Vorlagen-ID wird als Kartenmetadaten gespeichert.

## Dashboard-Arbeitsablauf

1. Öffnen Sie die Registerkarte Workboard in der Control UI.
2. Erstellen Sie eine Karte mit Titel, Notizen, Priorität, Bezeichnungen, optionalem Agenten und
   optionaler verknüpfter Sitzung – oder öffnen Sie Sessions und wählen Sie **Add to Workboard**
   für eine vorhandene Sitzung.
3. Ziehen Sie die Karte zwischen Spalten oder fokussieren Sie das kompakte Statussteuerelement und verwenden Sie
   das Menü oder ArrowLeft/ArrowRight. Während des Ziehens wird die Quellkarte abgeblendet und
   verfügbare Zielspalten erhalten eine Umrandung.
4. Starten Sie die Arbeit von der Karte aus, um eine Dashboard-Sitzung zu erstellen oder wiederzuverwenden.
5. Öffnen Sie während der Arbeit des Agenten die verknüpfte Sitzung über die Karte.
6. Lassen Sie die Lebenszyklus-Synchronisierung laufende Arbeit nach `review`/`blocked` verschieben und
   verschieben Sie die Karte nach der Annahme manuell nach `done`.

### Widgets für Sitzungs-Boards

Workboard enthält zwei native Widgets für Sitzungs-Dashboards (siehe
[Dashboards](/de/web/dashboards)). Der Agent heftet sie mit seinem `dashboard`-Tool
unter Verwendung von `content: { kind: "plugin", pluginKind, props }` an, und sie werden
als integrierte Benutzeroberfläche mit Live-Daten dargestellt – ohne Sandbox-Frame oder Funktionsfreigabe:

- `workboard:card` mit `props: { cardId }` zeigt eine Karte mit ihrem Status-
  Steuerelement, ihrer Priorität und dem zugewiesenen Agenten.
- `workboard:mini` mit optionalem `props: { boardId, limit }` zeigt Anzahlen pro Status
  sowie die wichtigsten bereiten/laufenden Karten und verlinkt auf die vollständige Board-Seite.
  Ohne `boardId` werden alle Boards zusammengefasst; mit `boardId` wird der Umfang auf dieses
  Board beschränkt (Karten, die ohne explizite Board-ID erstellt wurden, befinden sich auf `default`).

## Diagnose

Diagnosen werden aus lokalen Kartenmetadaten berechnet. Integrierte Prüfungen kennzeichnen:

| Art                         | Bedingung                                                                      |
| --------------------------- | ------------------------------------------------------------------------------ |
| `stranded_ready`            | Zugewiesene Karte mit `todo`/`backlog`/`ready`, die seit über 1 Stunde nicht aktualisiert wurde.             |
| `running_without_heartbeat` | Karte mit `running` ohne Claim-Heartbeat oder Ausführungsaktualisierung seit über 20 Minuten. |
| `blocked_too_long`          | Karte mit `blocked`, die seit über 24 Stunden nicht aktualisiert wurde.                                   |
| `repeated_failures`         | Die verfolgte Fehleranzahl der Karte erreicht 2 oder mehr.                                |
| `missing_proof`             | Karte mit `done` ohne Nachweis, Artefakte oder Anhänge.                          |
| `orphaned_session`          | Karte mit `running` und einem `sessionKey`, aber ohne `execution`-Metadaten.                |

## Berechtigungen

Gateway-RPC-Methoden befinden sich unter `workboard.*`:

| Umfang           | Methoden                                                                                                                                                                                                                                                                                                                                                                            |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operator.read`  | `cards.list`, `cards.export`, `cards.diagnostics`, Anhänge auflisten/abrufen, Benachrichtigungsereignisse lesen, `boards.list`, `cards.stats`, `cards.runs`                                                                                                                                                                                                                                       |
| `operator.write` | `cards.diagnostics.refresh`, erstellen/aktualisieren/verschieben/löschen/kommentieren/verknüpfen/Abhängigkeit verknüpfen/Nachweis/Artefakt, Anhang hinzufügen/löschen, Worker-Protokoll, Protokollverstoß, Claim/Heartbeat/freigeben/hochstufen/neu zuweisen/zurückfordern/abschließen/blockieren/Blockierung aufheben, `cards.dispatch`, `cards.bulk`, archivieren, `boards.upsert`/`archive`/`delete`, `cards.specify`/`decompose`, Benachrichtigung abonnieren/löschen/fortschreiben |

Keine RPC-Methode erfordert `operator.admin`. Browser, die mit schreibgeschütztem
Operator-Zugriff verbunden sind, können das Board prüfen, aber keine Karten ändern. Ein Admin-Umfang
erweitert die akzeptierten Workboard-Hostpfade; er ändert nicht die verfügbaren Methoden.

## Speicher

Workboard speichert dauerhafte Daten in einer Plugin-eigenen relationalen SQLite-Datenbank
im OpenClaw-Statusverzeichnis: Boards, Karten, Bezeichnungen, Lebenszyklusereignisse,
Durchlaufversuche, Kommentare, Abhängigkeitsverknüpfungen, Nachweise, Artefaktreferenzen,
Anhangsmetadaten und -blobs, Diagnosen, Benachrichtigungen, Worker-Protokolle,
Protokollstatus und Abonnements befinden sich alle in Workboard-Tabellen (nicht in
Plugin-Schlüssel-Wert-Einträgen). Ein Kartenexport erhält die Board-Darstellung,
ohne die Blob-Inhalte von Anhängen einzubetten.

Installationen, die Workboard in der Version `.28` verwendet haben, können
`openclaw doctor --fix` ausführen, um die ausgelieferten veralteten Plugin-Status-Namespaces
(`workboard.cards`, `workboard.boards`, `workboard.notify` und, falls vorhanden,
`workboard.attachments`) in die relationale Datenbank zu migrieren.

## Fehlerbehebung

**Die Registerkarte meldet, dass Workboard nicht verfügbar ist**

```bash
openclaw plugins inspect workboard --runtime --json
```

Wenn `plugins.allow` konfiguriert ist, fügen Sie `workboard` hinzu. Wenn `plugins.deny`
`workboard` enthält, entfernen Sie es, bevor Sie das Plugin aktivieren.

**Karten werden nicht gespeichert**

Stellen Sie sicher, dass die Browserverbindung über `operator.write`-Zugriff verfügt. Schreibgeschützte Operator-
Sitzungen können Karten auflisten, aber nicht erstellen, bearbeiten, verschieben oder löschen.

**Beim Starten einer Karte wird nicht die erwartete Sitzung geöffnet**

Prüfen Sie die Agenten-ID und die verknüpfte Sitzung der Karte und öffnen Sie anschließend Sessions oder Chat, um
den tatsächlichen Durchlaufstatus zu prüfen.

**Dispatch startet keinen Worker**

Stellen Sie sicher, dass mindestens eine Karte mit `ready` ohne aktiven Claim vorhanden ist:

```bash
openclaw workboard list --status ready
```

Wenn die CLI eine reine Datenverarbeitung meldet, starten Sie den Gateway oder starten Sie ihn neu und
versuchen Sie es erneut – die reine Datenverarbeitung aktualisiert den lokalen Workboard-Status, kann jedoch keine
Subagent-Worker-Läufe starten. Karten können auch übersprungen werden, wenn eine andere Karte für denselben
Verantwortlichen oder Agenten bereits ausgeführt wird oder auf eine Überprüfung wartet. Schließen Sie diese aktive
Arbeit ab, blockieren Sie sie oder geben Sie sie frei, bevor Sie weitere Aufgaben für denselben
Verantwortlichen verteilen.

## Verwandte Themen

- [Control UI](/de/web/control-ui)
- [Workboard-CLI](/de/cli/workboard)
- [Plugins](/de/tools/plugin)
- [Plugins verwalten](/de/plugins/manage-plugins)
- [Sitzungen](/de/concepts/session)
