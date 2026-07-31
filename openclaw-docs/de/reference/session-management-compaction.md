---
read_when:
    - Sie müssen Sitzungs-IDs, Transkriptereignisse oder Felder von Sitzungszeilen debuggen
    - Sie ändern das Verhalten der automatischen Compaction oder fügen Aufräumarbeiten vor der Compaction hinzu.
    - Sie möchten Speicher-Flushes oder stille System-Turns implementieren
summary: 'Detaillierter Einblick: Sitzungsspeicher und Transkripte, Lebenszyklus und Interna der (automatischen) Compaction'
title: Ausführliche Erläuterung der Sitzungsverwaltung
x-i18n:
    generated_at: "2026-07-26T19:13:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ae02d49245768831abd17e1c2e5adacfa1a36673cef2a8a7a06a5300392b104
    source_path: reference/session-management-compaction.md
    workflow: 16
---

Ein einzelner **Gateway-Prozess** verwaltet den Sitzungsstatus durchgängig. Benutzeroberflächen (macOS-App, Web-Control-UI, TUI) fragen Sitzungslisten und Tokenanzahlen beim Gateway ab. Im Remote-Modus befinden sich die Sitzungsdateien auf dem Remote-Host; das Prüfen der Dateien auf Ihrem lokalen Mac spiegelt daher nicht wider, was der Gateway verwendet.

Zunächst die Übersichtsdokumentation: [Sitzungsverwaltung](/de/concepts/session), [Compaction](/de/concepts/compaction), [Speicherübersicht](/de/concepts/memory), [Speichersuche](/de/concepts/memory-search), [Sitzungsbereinigung](/de/concepts/session-pruning), [Transkripthygiene](/de/reference/transcript-hygiene), vollständige Konfigurationsreferenz unter [Agent-Konfiguration](/de/gateway/config-agents).

## Zwei Persistenzebenen

1. **Sitzungszeilen (agentenspezifisches SQLite)** – Schlüssel-Wert-Zuordnung `sessionKey -> SessionEntry`. Veränderlicher Laufzeitstatus, der dem Gateway gehört. Erfasst Metadaten: aktuelle Sitzungs-ID, letzte Aktivität, Umschalter und Tokenzähler.
2. **Transkriptereignisse (agentenspezifisches SQLite)** – nur anhängbar und baumstrukturiert (Einträge enthalten `id` + `parentId`). Speichert die Unterhaltung, Tool-Aufrufe und Compaction-Zusammenfassungen und rekonstruiert den Modellkontext für zukünftige Interaktionen. Compaction-Prüfpunkte sind Metadaten über dem komprimierten Nachfolgetranskript – eine neue Compaction schreibt keine zweite Kopie von `.checkpoint.*.jsonl`.

Ältere Installationen können noch `sessions.json`-Dateien im Verzeichnis `sessions/`
des Agenten enthalten. Behandeln Sie diese Dateien als Legacy-Migrationseingaben für Sitzungszeilen oder als explizite
Ziele für die Offline-Wartung. Der Gateway-Start und `openclaw doctor --fix` importieren
aktive Legacy-Zeilen und den Transkriptverlauf automatisch in den agentenspezifischen
SQLite-Speicher. Führen Sie `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` aus und folgen Sie anschließend der
[Doctor-Migrationssequenz](/de/cli/doctor#session-sqlite-migration), wenn Sie explizite
Inspektions- oder Validierungsnachweise benötigen. Wenn eine Migration fehlschlägt,
nachdem Legacy-Transkriptartefakte archiviert wurden, verwenden Sie den Doctor-Wiederherstellungsmodus
aus dieser Sequenz. Die Wiederherstellung verwendet Migrationsmanifeste, stellt nur die
betroffenen archivierten Unterstützungsartefakte wieder her, bereitet auf Anforderung
einen bereinigten GitHub-Issue-Bericht vor und veranlasst die aktive Laufzeit nicht dazu,
JSONL-Dateien erneut zu lesen.

Gateway-Verlaufsleser vermeiden es, das gesamte Transkript zu materialisieren, sofern die Oberfläche keinen beliebigen historischen Zugriff benötigt. Der Verlauf der ersten Seite, der eingebettete Chatverlauf, die Wiederherstellung nach einem Neustart sowie Token-/Nutzungsprüfungen verwenden begrenzte Lesevorgänge am Ende der SQLite-Daten. Vollständige Transkriptscans erfolgen über den asynchronen Transkriptindex und werden von gleichzeitigen Lesern gemeinsam genutzt.

## Speicherorte auf dem Datenträger

Pro Agent auf dem Gateway-Host (aufgelöst über `src/config/sessions.ts`):

- Laufzeitspeicher für Sitzungszeilen: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- Laufzeit-Transkriptzeilen: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- Legacy-/Archiv-Transkriptartefakte: `~/.openclaw/agents/<agentId>/sessions/`
- Legacy-Migrationseingabe für Zeilen: `~/.openclaw/agents/<agentId>/sessions/sessions.json`

## Speicherwartung und Datenträgerkontrollen

`session.maintenance` steuert die automatische Wartung für SQLite-Sitzungszeilen, SQLite-Transkriptzeilen, Archivartefakte und Trajektorien-Sidecars:

| Schlüssel               | Standardwert          | Hinweise                                                                                    |
| ----------------------- | --------------------- | ------------------------------------------------------------------------------------------- |
| `mode`                  | `"enforce"`           | oder `"warn"` (nur Bericht, keine Änderung)                                                 |
| `pruneAfter`            | `"30d"`               | Altersgrenze für veraltete Einträge                                                         |
| `maxEntries`            | `500`                 | Obergrenze für Sitzungseinträge                                                             |
| `resetArchiveRetention` | beibehalten (keine Altersgrenze) | Altersgrenze für `*.reset.*`-/`*.deleted.*`-Transkriptarchive; eine Dauer aktiviert die Löschung |
| `maxDiskBytes`          | `10gb`                | agentenspezifisches Datenträgerbudget für Sitzungen; `false` deaktiviert es                 |
| `highWaterBytes`        | 80 % von `maxDiskBytes` | Zielwert nach der Budgetbereinigung                                                         |

Ein Zurücksetzen setzt die aktive `sessionKey -> sessionId`-Zuordnung fort, behält jedoch die vorherige SQLite-Sitzung sowie deren Transkript-, Trajektorien- und Suchzeilen bei. Dieser Verlauf bleibt unter demselben Sitzungsschlüssel durchsuchbar; gewöhnliche Eintrags- und Sitzungslisten zeigen nur die neue aktive Zuordnung. Der beibehaltene Verlauf nach dem Zurücksetzen wird durch das Datenträgerbudget begrenzt, nicht durch `resetArchiveRetention`, das nur Archivartefakte altern lässt. Eine explizite Löschung verhält sich anders: Sie schreibt und überprüft ein komprimiertes Transkriptarchiv (`*.jsonl.deleted.<timestamp>.zst`, wenn zstd verfügbar ist), bevor die Zeilen der gelöschten Sitzung entfernt werden.

Die Durchsetzung von `maxDiskBytes` verwendet physische Bytes: die agentenspezifische SQLite-Hauptdatei, ihre `-wal`-Datei und gezählte Dateien im Sitzungsverzeichnis des Agenten. Sie schätzt niemals die JSON-Größen von Zeilen und zieht keine logischen Zeilengrößen von dieser Summe ab.

Gateway-Testsitzungen für Modellläufe (Schlüssel, die `agent:*:explicit:model-run-<uuid>` entsprechen) erhalten eine separate, feste Aufbewahrungsdauer von `24h`. Diese Bereinigung ist druckgesteuert: Sie wird nur ausgeführt, wenn der Wartungs- oder Obergrenzendruck für Sitzungseinträge erreicht ist, und nur vor dem globalen Schritt zur Bereinigung bzw. Begrenzung veralteter Einträge. Andere explizite Sitzungen verwenden diese Aufbewahrungsdauer nicht.

Wenn die kombinierte physische Nutzung `maxDiskBytes` überschreitet, gibt `mode: "enforce"` zunächst Datenbankspeicher frei, für den ein Prüfpunkt gesetzt werden kann, und entfernt anschließend die ältesten beibehaltenen Archive aus Zurücksetzungen und Löschungen. Wenn die Nutzung weiterhin über `highWaterBytes` liegt, durchläuft der Vorgang historische SQLite-Sitzungen nach `sessions.updated_at`, beginnend mit der ältesten. Historisch bedeutet, dass die Sitzungs-ID weder von einem aktiven Sitzungseintrag noch von einem Routenziel oder einem zugelassenen bzw. laufenden Durchlauf referenziert wird. Für jedes Opfer schreibt die Bereinigung das komprimierte Archiv, führt fsync aus und liest es zurück, bevor eine Schreibtransaktion die Sitzungszeile und deren Transkript-, Trajektorien-, Aktiv-, Index- und FTS-Projektionen entfernt. Dies schließt Sitzungen ein, die Trajektorienereignisse, aber keine Transkriptereignisse enthalten. Die Bereinigung prüft die Routen-, Eintrags- und Zulassungsreferenzen zum Löschzeitpunkt erneut, misst die physische Nutzung nach jedem Archiv- oder Sitzungsopfer neu und beendet den Vorgang bei `highWaterBytes`.

Bestätigte Schreibvorgänge und Löschungen landen zunächst im WAL. Die Bereinigung setzt dafür einen Prüfpunkt, sodass das WAL sofort schrumpfen kann, und verwendet anschließend inkrementelles Vacuum, um freigabefähige freie Endseiten aus der Hauptdatei zurückzugeben. Seiten, die noch nicht freigegeben werden können, verbleiben in der Hauptdatei und werden daher bei der nächsten physischen Messung weiterhin mitgezählt. `mode: "warn"` meldet die aktuelle physische Überschreitung, ohne einen Prüfpunkt zu setzen, ein Archiv zu schreiben oder Zeilen zu löschen.

Wartung bei Bedarf ausführen:

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

Die Wartung behält dauerhafte externe Unterhaltungszeiger wie Gruppensitzungen und auf Threads beschränkte Chatsitzungen bei. Synthetische Laufzeiteinträge (Cron, Hooks, Heartbeat, ACP, Unteragenten) können jedoch entfernt werden, sobald sie das konfigurierte Alter, die Anzahl oder das Datenträgerbudget überschreiten. Isolierte Cron-Läufe verwenden eine separate `cron.sessionRetention`-Steuerung, unabhängig von der Aufbewahrung für Modelllauf-Testsitzungen.

Normale Gateway-Schreibvorgänge laufen über den Sitzungs-Accessor, der agentenspezifische SQLite-Änderungen über den Laufzeit-Schreibpfad serialisiert. Laufzeitcode sollte die Accessor-Hilfsfunktionen in `src/config/sessions/session-accessor.ts` bevorzugen; die Legacy-Hilfsfunktionen `sessions.json` sind Werkzeuge für Migration und Offline-Wartung. Wenn ein Gateway erreichbar ist, delegieren `openclaw sessions cleanup` und `openclaw agents delete` ohne Trockenlauf Speicheränderungen an den Gateway, damit sich die Bereinigung derselben Schreibwarteschlange anschließt; `--store <path>` ist der explizite Offline-Reparaturpfad für einen ausgewählten Legacy-Speicher und bleibt immer lokal (ebenso wie `--dry-run`). Die Bereinigung von `maxEntries` erfolgt für produktionsgroße Speicher stapelweise, sodass ein Speicher die konfigurierte Obergrenze kurzzeitig überschreiten kann, bevor die nächste Bereinigung bei Erreichen des oberen Schwellenwerts ihn auf die Zielgröße zurückführt. Lesevorgänge bereinigen oder begrenzen beim Gateway-Start niemals Einträge – dies erfolgt nur bei Schreibvorgängen oder durch `openclaw sessions cleanup --enforce`. Letzteres wendet die Obergrenze außerdem sofort an und bereinigt alte, nicht referenzierte Legacy-Transkript-, Prüfpunkt- und Trajektorienartefakte, selbst wenn kein Datenträgerbudget konfiguriert ist.

OpenClaw erstellt bei Gateway-Schreibvorgängen keine automatischen Rotationssicherungen vom Typ `sessions.json.bak.*` mehr. Das aktuelle Schema weist den Legacy-Schlüssel `session.maintenance.rotateBytes` zurück, und `openclaw doctor --fix` entfernt ihn aus älteren Konfigurationen.

Transkriptänderungen verwenden die Sitzungsschreibwarteschlange für das SQLite-Transkriptziel:

Sitzungsschreibsperren verwenden feste Produktionsstandardwerte. Die entsprechenden
Umgebungsvariablen `OPENCLAW_SESSION_WRITE_LOCK_*` bleiben für
Diagnosen auf Prozessebene und Notfallüberschreibungen verfügbar.

### Downgrade nach der SQLite-Umstellung

Stellen Sie archivierte Legacy-Transkriptartefakte wieder her, bevor Sie eine ältere
dateibasierte OpenClaw-Version ausführen:

```bash
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

Die Migration belässt Legacy-Dateien vom Typ `sessions.json` für Support und
Rollback an ihrem Platz, doch aktive Transkript-JSONL-Dateien, die in SQLite
importiert wurden, werden in `session-sqlite-import-archive/` umbenannt. Ältere dateibasierte
Laufzeiten folgen den `sessionFile`-Pfaden in `sessions.json` und benötigen daher
vor dem Start die Wiederherstellung dieser Artefakte. Die Wiederherstellung verwendet
Migrationsmanifeste, verschiebt nur erfasste archivierte Artefakte, deren ursprüngliche
Pfade fehlen, und belässt die SQLite-Datenbank für eine spätere Wiederherstellung
an ihrem Platz.

Nach der SQLite-Umstellung erstellte Sitzungen existieren ausschließlich in SQLite und
werden in einer älteren dateibasierten Laufzeit nicht angezeigt. Wenn Sie nach einem
Downgrade erneut ein Upgrade durchführen, führen Sie die Doctor-Sequenz für Inspektion
und Validierung erneut aus, damit OpenClaw die wiederhergestellten Legacy-Artefakte vor
dem Import überprüfen kann.

## Cron-Sitzungen und Laufprotokolle

Isolierte Cron-Läufe erstellen eigene Sitzungseinträge und Transkripte mit gesonderter Aufbewahrung:

- `cron.sessionRetention` (Standardwert `"24h"`) bereinigt alte isolierte Cron-Laufsitzungen aus dem Speicher; `false` deaktiviert dies.
- Der Laufverlauf behält die neuesten 2000 Abschlusszeilen pro Cron-Auftrag bei. Verlorene Zeilen behalten ihr Bereinigungsfenster von 24 Stunden.

Wenn Cron die Erstellung einer neuen isolierten Laufsitzung erzwingt, bereinigt es den vorherigen Sitzungseintrag `cron:<jobId>`, bevor es die neue Zeile schreibt: Es übernimmt sichere Einstellungen (Denk-/Schnell-/Ausführlichkeits-/Reasoning-Einstellungen, Bezeichnungen, Anzeigename) und explizit vom Benutzer ausgewählte Modell-/Authentifizierungsüberschreibungen, verwirft jedoch den umgebenden Unterhaltungskontext (Kanal-/Gruppenrouting, Sende-/Warteschlangenrichtlinie, Rechteerhöhung, Ursprung, ACP-Laufzeitbindung), sodass ein neuer isolierter Lauf keine veraltete Zustellungs- oder Laufzeitautorität von einem älteren Lauf übernehmen kann.

## Sitzungsschlüssel (`sessionKey`)

Ein `sessionKey` identifiziert, in welchem Unterhaltungsbereich Sie sich befinden (Routing + Isolation). Kanonische Regeln: [/concepts/session](/de/concepts/session).

| Muster                       | Beispiel                                                    |
| ---------------------------- | ----------------------------------------------------------- |
| Haupt-/Direktchat (pro Agent) | `agent:<agentId>:<mainKey>` (Standardwert `main`)                |
| Gruppe                       | `agent:<agentId>:<channel>:group:<id>`                      |
| Raum/Kanal (Discord/Slack)   | `agent:<agentId>:<channel>:channel:<id>` oder `...:room:<id>` |
| Cron                         | `cron:<job.id>`                                             |
| Webhook                      | `hook:<uuid>` (sofern nicht überschrieben)                           |

## Sitzungs-IDs (`sessionId`)

Jeder `sessionKey` verweist auf eine aktuelle `sessionId` (die SQLite-Transkriptidentität, welche die Unterhaltung fortsetzt). Die Entscheidungslogik befindet sich in `initSessionState()` in `src/auto-reply/reply/session.ts`.

- **Zurücksetzen** (`/new`, `/reset`) erstellt eine neue `sessionId` für diese `sessionKey`.
- **Kein automatisches Zurücksetzen** ist die Standardeinstellung. Die aktuelle `sessionId` wird fortgesetzt, während Compaction den aktiven Modellkontext begrenzt hält.
- **Tägliches Zurücksetzen** (`session.reset.mode: "daily"`) erstellt bei der nächsten Nachricht nach der konfigurierten lokalen Stundengrenze (`session.reset.atHour`, Standardwert `4`) eine neue `sessionId`.
- **Ablauf bei Inaktivität** (`session.reset.mode: "idle"` mit `session.reset.idleMinutes` oder veraltet `session.idleMinutes`) erstellt eine neue `sessionId`, wenn nach dem Inaktivitätszeitraum eine Nachricht eintrifft. Wenn sowohl das tägliche Zurücksetzen als auch der Ablauf bei Inaktivität konfiguriert sind, gilt das Ereignis, das zuerst eintritt.
- **Fortsetzen nach Wiederverbindung der Steuerungsoberfläche** bewahrt die aktuell sichtbare Sitzung für einen Sendevorgang nach einer Wiederverbindung, wenn das Gateway die passende `sessionId` von einem Client der Bedieneroberfläche empfängt. Dies ist ein einmaliges Signal; gewöhnliche veraltete Sendevorgänge erstellen weiterhin eine neue `sessionId`.
- **Systemereignisse** (Heartbeat, Cron-Aktivierungen, Ausführungsbenachrichtigungen, Gateway-Verwaltung) können die Sitzungszeile verändern, verlängern jedoch niemals die Aktualität für das tägliche Zurücksetzen oder das Zurücksetzen bei Inaktivität. Beim Wechsel durch ein Zurücksetzen werden Benachrichtigungen über Systemereignisse in der Warteschlange für die vorherige Sitzung verworfen, bevor der neue Prompt erstellt wird.
- **Richtlinie für übergeordnete Forks** verwendet beim Erstellen eines Threads oder Subagent-Forks den aktiven Branch von OpenClaw. Wenn dieser Branch zu groß ist (über einer festen internen Obergrenze, derzeit 100K Token), startet OpenClaw das untergeordnete Element mit isoliertem Kontext, statt fehlzuschlagen oder einen unbrauchbaren Verlauf zu übernehmen. Die Größenbestimmung erfolgt automatisch und ist nicht konfigurierbar; die veraltete Konfiguration `session.parentForkMaxTokens` wird durch `openclaw doctor --fix` entfernt.
- **Bediener-Forks**: `sessions.create { parentSessionKey, fork: true }` erstellt eine neue Sitzung, deren Transkript vom aktuellen Zustand der übergeordneten Sitzung abzweigt (derselbe Fork-Mechanismus wie beim Erzeugen von Subagents, einschließlich der oben genannten Größenobergrenze). Der Fork wird abgelehnt, solange in der übergeordneten Sitzung ein aktiver Lauf ausgeführt wird, übernimmt die Modellauswahl der übergeordneten Sitzung, sofern nicht ausdrücklich eine andere angegeben wird, und kennzeichnet das untergeordnete Element mit `forkedFromParent` und neuen Token-Zählern.

## Schema des Sitzungsspeichers

Der Laufzeitspeicher bewahrt `SessionEntry`-Werte in einer agentenspezifischen SQLite-Datenbank auf. Der Werttyp ist `SessionEntry` in `src/config/sessions.ts`. Wichtige Felder (nicht vollständig):

- `sessionId`: aktuelle Transkript-ID zur Adressierung von SQLite-Transkriptzeilen
- `sessionStartedAt`: Startzeitstempel der aktuellen `sessionId`; wird für die Aktualität des täglichen Zurücksetzens verwendet. Bei veralteten Zeilen kann er aus dem JSONL-Sitzungsheader abgeleitet werden.
- `lastInteractionAt`: Zeitstempel der letzten tatsächlichen Benutzer-/Kanalinteraktion; wird für die Aktualität des Zurücksetzens bei Inaktivität verwendet, damit Heartbeat-, Cron- und Ausführungsereignisse Sitzungen nicht aktiv halten. Bei veralteten Zeilen ohne dieses Feld wird auf die wiederhergestellte Sitzungsstartzeit zurückgegriffen.
- `updatedAt`: Zeitstempel der letzten Änderung der Speicherzeile, verwendet für Auflistung/Bereinigung/Verwaltung – nicht maßgeblich für die Aktualität des täglichen Zurücksetzens oder des Zurücksetzens bei Inaktivität.
- `archivedAt`: optionaler Archivierungszeitstempel. Archivierte Sitzungen bleiben mit intaktem Transkript im Speicher und werden von normalen Auflistungen aktiver Sitzungen ausgeschlossen.
- `pinnedAt`: optionaler Anheftzeitstempel. Aktive angeheftete Sitzungen werden vor nicht angehefteten Sitzungen einsortiert; beim Archivieren einer Sitzung wird ihre Anheftung aufgehoben.
- Codex-Thread-Interoperabilität: Beide Felder entsprechen der Thread-Verwaltungsstruktur von Codex – die booleschen Werte `archived`/`pinned` bei der Übertragung werden immer aus dem Zeitstempel abgeleitet und serverseitig gesetzt, entsprechend der Semantik von Codex `threads.archived_at` und der camelCase-Serialisierung. OpenClaw-Zeitstempel verwenden Millisekunden seit der Epoche, während Codex Sekunden seit der Epoche verwendet; daher konvertieren Brücken an der Plugin-Schnittstelle `codex`. Codex verfügt noch über keine API zum Anheften (nur `thread/archive`/`thread/unarchive`); der Anheftstatus verbleibt auf der OpenClaw-Seite, bis eine solche API vorhanden ist. Dann ermöglicht die übereinstimmende Struktur gebundenen Sitzungen, den Anheftstatus automatisch in beide Richtungen zu übertragen.
- Die Codex-Überwachung listet nur nicht archivierte native Threads auf. Ein Gateway-lokaler Thread mit `idle` oder `notLoaded` und unbekannter Aktivität kann über das native `thread/archive` nur archiviert werden, nachdem der Bediener ausdrücklich bestätigt hat, dass kein anderer Codex-Prozess ihn besitzt; das Plugin liest zunächst erneut den prozesslokalen Status, anschließend verschwindet der Thread aus dem Katalog. Dieser Lesevorgang kann nicht beweisen, dass nicht ein anderer App-Server-Prozess den Thread verwendet. OpenClaw lehnt die Archivierung aktiver Zeilen und Fehlerzeilen ab, und die Archivierung über gekoppelte Nodes ist nicht verfügbar, bis die Node-Brücke den vollständigen gestreamten Thread-Lebenszyklus verwalten kann. Wird die Archivierung in einem nativen Codex-Client aufgehoben, kann der Thread wieder angezeigt werden.
- `lastReadAt` / `markedUnreadAt`: serverseitig durch `sessions.patch { unread }` gesetzte Zeitstempel des Lesestatus – `unread: false` zeichnet einen Lesevorgang auf (setzt `lastReadAt`, löscht `markedUnreadAt`); `unread: true` markiert die Sitzung bis zum nächsten Lesevorgang als ungelesen. Sitzungszeilen stellen einen abgeleiteten booleschen Wert `unread` bereit: ausdrücklich als ungelesen markiert oder vor der letzten Aktivität gelesen. Sitzungen, die nie als gelesen markiert wurden, bleiben `unread: false`, damit vorhandene Installationen nach einem Upgrade nicht plötzlich hervorgehoben werden.
- `lastActivityAt`: Zeitstempel des letzten abgeschlossenen Agentenlaufs, der als lesenswerte ungelesene Aktivität zählt (Benutzer-, Kanal- und Cron-Läufe). Heartbeat- und interne Ereignisdurchläufe sowie Metadaten-Patches aktualisieren ihn nicht; `updatedAt` ist kein Aktivitätssignal.
- `sessionFile`: veraltete Markierung, die für die Kompatibilität bei Migration und Archivierung beibehalten wird; die aktive Laufzeit verwendet die SQLite-Identität
- `chatType`: `direct | group | room`
- `provider`, `subject`, `room`, `space`, `displayName`: Metadaten zur Gruppen-/Kanalbezeichnung
- Umschalter: `thinkingLevel`, `verboseLevel`, `reasoningLevel`, `elevatedLevel`, `sendPolicy` (sitzungsspezifische Überschreibung)
- Modellauswahl: `providerOverride`, `modelOverride`, `authProfileOverride`
- Token-Zähler (nach bestem Bemühen/providerabhängig): `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`
- `compactionCount`: Anzahl der abgeschlossenen automatischen Compaction-Vorgänge für diesen Sitzungsschlüssel
- `memoryFlushAt` / `memoryFlushCompactionCount`: Zeitstempel und Compaction-Anzahl der letzten Speicherleerung vor der Compaction

Das Gateway ist maßgeblich: Es kann Einträge neu schreiben oder wiederherstellen, während Sitzungen
ausgeführt werden. Migrieren Sie bei veralteten dateibasierten Installationen mit
`openclaw doctor --session-sqlite import --session-sqlite-all-agents`, statt
`sessions.json` zu bearbeiten und zu erwarten, dass die Laufzeit diese Datei weiterhin liest.

## Struktur der Transkriptereignisse

Transkripte werden vom OpenClaw-Sitzungszugriff verwaltet und dem Laufzeitcode über identitätsbasierte Hilfsfunktionen bereitgestellt. Der Ereignisstrom ist nur anhängbar:

- Erster Eintrag: Sitzungsheader – `type: "session"`, `id`, `cwd`, `timestamp`, optional `parentSession`.
- Danach: Einträge mit `id` + `parentId` (Baumstruktur).

Wichtige Eintragstypen:

- `message`: Benutzer-/Assistenten-/toolResult-Nachrichten
- `custom_message`: von einer Erweiterung eingefügte Nachricht, die _tatsächlich_ in den Modellkontext eingeht (wird in der TUI dargestellt, wenn `display: true`, und vollständig ausgeblendet, wenn `display: false`)
- `custom`: Erweiterungsstatus, der _nicht_ in den Modellkontext eingeht (zur dauerhaften Speicherung des Erweiterungsstatus über erneutes Laden hinweg)
- `compaction`: dauerhaft gespeicherte Compaction-Zusammenfassung mit `firstKeptEntryId` und `tokensBefore`
- `branch_summary`: dauerhaft gespeicherte Zusammenfassung beim Navigieren in einem Baum-Branch

OpenClaw nimmt bewusst keine „Korrekturen“ an Transkripten vor; das Gateway verwendet `SessionManager`, um sie zu lesen und zu schreiben.

## Kontextfenster im Vergleich zu erfassten Token

Zwei unterschiedliche Konzepte:

1. **Modellkontextfenster**: feste Obergrenze pro Modell (für das Modell sichtbare Token). Stammt aus dem Modellkatalog und kann über die Konfiguration überschrieben werden.
2. **Zähler des Sitzungsspeichers**: fortlaufende Statistiken, die in die Sitzungszeile geschrieben werden (verwendet für `/status` und Dashboards). `contextTokens` ist ein Laufzeitschätzwert/-berichtswert – behandeln Sie ihn nicht als strikte Garantie.

Weitere Informationen zu Grenzwerten: [/reference/token-use](/de/reference/token-use).

## Compaction: Was sie ist

Compaction fasst ältere Unterhaltungen in einem dauerhaft gespeicherten `compaction`-Eintrag im Transkript zusammen und lässt aktuelle Nachrichten unverändert. Nach der Compaction sehen zukünftige Durchläufe die Compaction-Zusammenfassung sowie die Nachrichten nach `firstKeptEntryId`. Compaction ist **dauerhaft**, anders als die Sitzungsbereinigung – siehe [/concepts/session-pruning](/de/concepts/session-pruning).

Die eingebettete OpenClaw-Compaction übernimmt standardmäßig die Thinking-Stufe der Sitzung. Legen Sie `agents.defaults.compaction.thinkingLevel` fest, um für Zusammenfassungsaufrufe eine separate Stufe zu verwenden; die Laufzeit begrenzt sie auf das jeweilige konkrete Compaction-Modell oder Fallback. Die native Compaction des Codex-App-Servers verwaltet ihre Kompaktierungsanfrage selbst und kann keine Compaction-spezifische Thinking-Überschreibung akzeptieren. Daher gibt OpenClaw eine Warnung aus und überlässt diese Einstellung Codex.

Das erneute Einfügen des Abschnitts aus AGENTS.md nach der Compaction bleibt über `agents.defaults.compaction.postCompactionSections` optional. Plugins können über `before_prompt_build` weiteren Prompt-Kontext hinzufügen.

### Chunk-Grenzen und Werkzeugzuordnung

Beim Aufteilen eines langen Transkripts in Compaction-Chunks hält OpenClaw Assistenten-Werkzeugaufrufe mit den zugehörigen `toolResult`-Einträgen zusammen:

- Wenn die Aufteilung nach Token-Anteil zwischen einem Werkzeugaufruf und dessen Ergebnis liegen würde, verschiebt OpenClaw die Grenze auf die Assistentennachricht mit dem Werkzeugaufruf, statt das Paar zu trennen.
- Wenn ein abschließender Werkzeugergebnisblock den Chunk andernfalls über die Zielgröße hinaus vergrößern würde, bewahrt OpenClaw diesen ausstehenden Werkzeugblock und lässt das nicht zusammengefasste Ende unverändert.
- Abgebrochene Werkzeugaufrufblöcke und solche mit Fehlern halten eine ausstehende Aufteilung nicht offen.

## Wann automatische Compaction erfolgt

Zwei Auslöser im eingebetteten OpenClaw-Agenten:

1. **Wiederherstellung nach Überschreitung**: Das Modell gibt einen Fehler wegen Kontextüberschreitung zurück (`request_too_large`, `context length exceeded`, `input exceeds the maximum number of tokens`, `input token count exceeds the maximum number of input tokens`, `input is too long for the model`, `ollama error: context length exceeded` und weitere providerspezifische Varianten) – Compaction durchführen und anschließend erneut versuchen. Wenn der Provider die Anzahl der versuchten Token meldet, leitet OpenClaw diese beobachtete Anzahl an die Compaction zur Wiederherstellung nach Überschreitung weiter; bestätigt der Provider die Überschreitung, stellt aber keine auswertbare Anzahl bereit, übergibt OpenClaw den Compaction-Engines und der Diagnose eine synthetische Anzahl, die das Budget minimal überschreitet. Wenn auch die Wiederherstellung nach Überschreitung fehlschlägt, zeigt OpenClaw ausdrückliche Anweisungen an und behält die aktuelle Sitzungszuordnung bei, statt unbemerkt zu einer neuen Sitzungs-ID zu wechseln – versuchen Sie die Nachricht erneut, führen Sie `/compact` aus oder führen Sie `/new` aus.
2. **Schwellenwertverwaltung**: nach einem erfolgreichen Durchlauf, wenn der aktuelle Kontext das Modellfenster abzüglich des integrierten OpenClaw-Puffers für Prompts und die nächste Modellausgabe überschreitet.

Zwei zusätzliche Schutzmechanismen werden außerhalb dieser beiden Auslöser ausgeführt:

- **Lokale Preflight-Compaction**: Legen Sie `agents.defaults.compaction.maxActiveTranscriptBytes` (Bytes oder eine Zeichenfolge wie `"20mb"`) fest, um vor dem Öffnen des nächsten Laufs eine lokale Compaction auszulösen, sobald das aktive Transkript diese Größe erreicht. Dies ist eine Größenbegrenzung für die lokalen Kosten beim erneuten Öffnen, nicht für die Roharchivierung – die normale semantische Compaction wird weiterhin ausgeführt und erfordert `truncateAfterCompaction`, damit die kompaktierte Zusammenfassung zu einem neuen Nachfolgetranskript wird.
- **Vorabprüfung während eines Turns**: Legen Sie `agents.defaults.compaction.midTurnPrecheck.enabled: true` (Standardwert `false`) fest, um eine Schutzvorrichtung für die Tool-Schleife hinzuzufügen. Nachdem ein Tool-Ergebnis angehängt wurde und vor dem nächsten Modellaufruf schätzt OpenClaw den Prompt-Druck mit derselben Preflight-Budgetlogik, die zu Beginn des Turns verwendet wird. Wenn der Kontext nicht mehr passt, führt die Schutzvorrichtung keine Inline-Compaction durch – sie löst ein strukturiertes Signal für die Vorabprüfung während des Turns aus, stoppt die aktuelle Prompt-Übermittlung und lässt die äußere Laufschleife den vorhandenen Wiederherstellungspfad verwenden (übergroße Tool-Ergebnisse kürzen, wenn dies ausreicht, oder den konfigurierten Compaction-Modus auslösen und den Versuch wiederholen). Funktioniert mit den Compaction-Modi `default` und `safeguard`, einschließlich Provider-gestützter Schutz-Compaction. Unabhängig von `maxActiveTranscriptBytes`: Die Byte-Größenbegrenzung wird ausgeführt, bevor ein Turn geöffnet wird; die Vorabprüfung während des Turns erfolgt später, nachdem neue Tool-Ergebnisse angehängt wurden.

## Compaction-Einstellungen

```json5
{
  agents: {
    defaults: {
      compaction: {
        enabled: true,
        keepRecentTokens: 20000,
      },
    },
  },
}
```

OpenClaw erzwingt eine integrierte Reserve für eingebettete Läufe und begrenzt sie anhand des aktiven Modellkontextfensters, sodass sie nicht das gesamte Prompt-Budget verbrauchen kann. Dadurch geraten lokale Modelle mit kleinem Kontext nicht bereits ab dem ersten Token in die Compaction, während genügend Spielraum für Verwaltungsaufgaben über mehrere Turns hinweg bleibt, beispielsweise für das Leeren des Speichers.

Die manuelle `/compact` berücksichtigt ein explizites `agents.defaults.compaction.keepRecentTokens` und behält den Abschnittspunkt des aktuellen Endes der Runtime bei. Ohne ein explizites Aufbewahrungsbudget ist die manuelle Compaction ein fester Prüfpunkt, und der neu aufgebaute Kontext beginnt mit der neuen Zusammenfassung.

Wenn `truncateAfterCompaction` aktiviert ist, rotiert OpenClaw das aktive Transkript nach der Compaction zu einem kompaktierten Nachfolger. Aktionen für Verzweigungs-/Wiederherstellungsprüfpunkte verwenden diesen kompaktierten Nachfolger; ältere Prüfpunktdateien vor der Compaction bleiben lesbar, solange auf sie verwiesen wird.

## Austauschbare Compaction-Provider

Plugins registrieren über `registerCompactionProvider()` in der Plugin-API einen Compaction-Provider. Wenn `agents.defaults.compaction.provider` auf die ID eines registrierten Providers gesetzt ist, delegiert die Schutzerweiterung die Zusammenfassung an diesen Provider statt an die integrierte `summarizeInStages`-Pipeline.

- `provider`: ID eines registrierten Compaction-Provider-Plugins. Für die standardmäßige LLM-Zusammenfassung nicht festlegen. Das Festlegen eines `provider` erzwingt `mode: "safeguard"`.
- Provider erhalten dieselben Compaction-Anweisungen und dieselbe Richtlinie zur Beibehaltung von Bezeichnern wie der integrierte Pfad, und die Schutzvorrichtung behält nach der Provider-Ausgabe weiterhin den Suffixkontext der letzten Turns und aufgeteilten Turns bei.
- Die integrierte Schutzzusammenfassung destilliert vorherige Zusammenfassungen zusammen mit neuen Nachrichten erneut, anstatt die vollständige vorherige Zusammenfassung unverändert beizubehalten.
- Der Schutzmodus aktiviert standardmäßig Qualitätsprüfungen für Zusammenfassungen; legen Sie `qualityGuard.enabled: false` fest, um das Verhalten zur Wiederholung bei fehlerhaft formatierter Ausgabe zu überspringen.
- Wenn der Provider fehlschlägt oder ein leeres Ergebnis zurückgibt, greift OpenClaw automatisch auf die integrierte LLM-Zusammenfassung zurück. Vom Aufrufer explizit ausgelöste Abbruch-/Zeitüberschreitungssignale werden erneut ausgelöst und nicht unterdrückt, sodass ein Abbruch immer berücksichtigt wird.

Quelle: `src/plugins/compaction-provider.ts`, `src/agents/agent-hooks/compaction-safeguard.ts`.

## Für Benutzer sichtbare Oberflächen

- `/status` in jeder Chatsitzung
- `openclaw status` (CLI)
- `openclaw sessions` / `openclaw sessions --json`
- Gateway-Protokolle (`pnpm gateway:watch` oder `openclaw logs --follow`): `embedded run auto-compaction start` + `complete`
- Ausführlicher Modus: `🧹 Auto-compaction complete` plus die Anzahl der Compactions

## Stille Verwaltungsaufgaben (`NO_REPLY`)

OpenClaw unterstützt „stille“ Turns für Hintergrundaufgaben, bei denen der Benutzer keine Zwischenausgabe sehen soll.

- Der Assistent beginnt seine Ausgabe mit dem exakten stillen Token `NO_REPLY` / `no_reply`, um „keine Antwort an den Benutzer übermitteln“ anzugeben. OpenClaw entfernt/unterdrückt diesen in der Übermittlungsschicht.
- Die Unterdrückung des exakten stillen Tokens unterscheidet nicht zwischen Groß- und Kleinschreibung: `NO_REPLY` und `no_reply` gelten beide, wenn die gesamte Nutzlast nur aus dem stillen Token besteht.
- Seit `2026.1.10` unterdrückt OpenClaw auch Entwurfs-/Eingabe-Streaming, wenn ein Teilabschnitt mit `NO_REPLY` beginnt, damit stille Vorgänge während des Turns keine Teilausgaben offenlegen.
- Dies ist nur für echte Hintergrund-Turns ohne Übermittlung vorgesehen – es ist keine Abkürzung für gewöhnliche ausführbare Benutzeranfragen.

## Speicherleerung vor der Compaction

Bevor eine automatische Compaction erfolgt, kann OpenClaw einen stillen agentischen Turn ausführen, der dauerhaften Zustand auf den Datenträger schreibt (beispielsweise `memory/YYYY-MM-DD.md` im Agent-Arbeitsbereich), damit die Compaction keinen kritischen Kontext löschen kann. OpenClaw überwacht die Kontextnutzung der Sitzung. Sobald sie einen weichen Schwellenwert unterhalb des Compaction-Schwellenwerts überschreitet, sendet OpenClaw unter Verwendung des exakten stillen Tokens `NO_REPLY` / `no_reply` eine stille Anweisung „Speicher jetzt schreiben“, sodass der Benutzer nichts sieht.

Konfiguration (`agents.defaults.compaction.memoryFlush`), vollständige Referenz unter [/gateway/config-agents](/de/gateway/config-agents#agentsdefaultscompaction):

| Schlüssel                    | Standardwert     | Hinweise                                                                                                                               |
| --------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                   | `true`           |                                                                                                                                        |
| `model`                     | nicht festgelegt | exakte Provider-/Modellüberschreibung nur für den Leerungs-Turn, beispielsweise `ollama/qwen3:8b`                                      |
| `softThresholdTokens`       | `4000`           | Abstand unterhalb des Compaction-Schwellenwerts, der eine Leerung auslöst                                                               |
| `forceFlushTranscriptBytes` | nicht festgelegt (deaktiviert) | erzwingt eine Leerung, sobald die Transkriptdatei diese Byte-Größe erreicht (oder eine Zeichenfolge wie `"2mb"`), selbst wenn die Token-Zähler veraltet sind; `0` deaktiviert |

Hinweise:

- Der integrierte Prompt und der System-Prompt enthalten einen `NO_REPLY`-Hinweis zur Unterdrückung der Übermittlung.
- Wenn `model` festgelegt ist, verwendet der Leerungs-Turn dieses Modell, ohne die Fallback-Kette der aktiven Sitzung zu übernehmen, sodass ausschließlich lokale Verwaltungsaufgaben bei einem Fehler nicht unbemerkt auf ein kostenpflichtiges Konversationsmodell zurückgreifen.
- Die Leerung wird einmal pro Compaction-Zyklus ausgeführt (in der Sitzungszeile nachverfolgt).
- Die Leerung wird nur für eingebettete OpenClaw-Sitzungen ausgeführt; CLI-Backends und Heartbeat-Turns überspringen sie.
- Die Leerung wird übersprungen, wenn der Sitzungsarbeitsbereich schreibgeschützt ist (`workspaceAccess: "ro"` oder `"none"`).
- Informationen zum Dateilayout und zu Schreibmustern des Arbeitsbereichs finden Sie unter [Speicher](/de/concepts/memory).

OpenClaw stellt in der Erweiterungs-API einen `session_before_compact`-Hook bereit, die oben beschriebene Leerungslogik befindet sich jedoch auf der Gateway-Seite (`src/auto-reply/reply/memory-flush.ts`, `src/auto-reply/reply/agent-runner-memory.ts`) und nicht in diesem Hook.

## Checkliste zur Fehlerbehebung

- **Falscher Sitzungsschlüssel?** Beginnen Sie mit [/concepts/session](/de/concepts/session) und bestätigen Sie `sessionKey` in `/status`.
- **Abweichung zwischen Speicher und Transkript?** Bestätigen Sie den Gateway-Host und den Speicherpfad aus `openclaw status`.
- **Zu viele Compactions?** Prüfen Sie das Kontextfenster des Modells (ein zu kleines Fenster erzwingt häufige Compactions) und die Aufblähung durch Tool-Ergebnisse (passen Sie die Sitzungsbereinigung an).
- **Scheint jeder Prompt bei einem kleinen lokalen Modell überzulaufen?** Vergewissern Sie sich, dass der Provider das korrekte Modellkontextfenster meldet. OpenClaw kann die effektive Reserve nur begrenzen, wenn dieses Fenster bekannt ist.
- **Werden stille Turns offengelegt?** Vergewissern Sie sich, dass die Antwort mit dem exakten stillen Token `NO_REPLY` beginnt (Groß-/Kleinschreibung wird nicht berücksichtigt) und dass Sie einen Build verwenden, der die Korrektur zur Streaming-Unterdrückung enthält (`2026.1.10`+).

## Verwandte Themen

- [Sitzungsverwaltung](/de/concepts/session)
- [Sitzungsbereinigung](/de/concepts/session-pruning)
- [Kontext-Engine](/de/concepts/context-engine)
- [Referenz zur Agent-Konfiguration](/de/gateway/config-agents)
