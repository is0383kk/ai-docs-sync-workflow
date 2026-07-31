---
read_when:
    - Vorbereiten eines Fehlerberichts oder einer Supportanfrage
    - Debugging von Gateway-Abstürzen, Neustarts, Speicherdruck oder übergroßen Nutzdaten
    - Überprüfen, welche Diagnosedaten aufgezeichnet oder geschwärzt werden
summary: Teilbare Gateway-Diagnosepakete für Fehlerberichte erstellen
title: Diagnoseexport
x-i18n:
    generated_at: "2026-07-26T18:27:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 97a805fed8d51de2e63e5c6a12ce03e91701d69654882cca7795c9f3553b1c55
    source_path: gateway/diagnostics.md
    workflow: 16
---

OpenClaw kann für Fehlerberichte ein lokales Diagnose-`.zip` erstellen: bereinigter Gateway-
Status, Integritätszustand, Protokolle, Konfigurationsstruktur und aktuelle stabilitätsbezogene Ereignisse ohne Nutzdaten.

Behandeln Sie Diagnosepakete bis zur Überprüfung wie Geheimnisse. Nutzdaten und Zugangsdaten
werden standardmäßig unkenntlich gemacht, das Paket fasst jedoch weiterhin lokale Gateway-Protokolle und
den Laufzeitstatus auf Hostebene zusammen.

## Schnellstart

```bash
openclaw gateway diagnostics export
```

Gibt den Pfad der geschriebenen ZIP-Datei aus. Wählen Sie einen Ausgabepfad:

```bash
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
```

Für die Automatisierung:

```bash
openclaw gateway diagnostics export --json
```

## Chat-Befehl

Eigentümer können `/diagnostics [note]` in jeder Unterhaltung ausführen, um einen lokalen
Gateway-Export als einzelnen kopierbaren Supportbericht anzufordern:

1. Senden Sie `/diagnostics`, optional mit einer kurzen Notiz (`/diagnostics bad tool choice`).
2. OpenClaw sendet eine Vorbemerkung und fordert eine einmalige ausdrückliche Ausführungsgenehmigung an, durch die
   `openclaw gateway diagnostics export --json` ausgeführt wird. Genehmigen Sie Diagnosen nicht über
   eine Regel, die alles zulässt.
3. Nach der Genehmigung antwortet OpenClaw mit dem lokalen Paketpfad, einer Zusammenfassung
   des Manifests, Datenschutzhinweisen und relevanten Sitzungs-IDs.

In Gruppenchats kann ein Eigentümer weiterhin `/diagnostics` ausführen, OpenClaw sendet das
Exportergebnis, Genehmigungsaufforderungen und die Aufschlüsselung der Codex-Sitzungen und -Threads jedoch
privat an den Eigentümer. Die Gruppe sieht nur einen kurzen Hinweis, dass die Diagnosedaten
privat gesendet wurden. Wenn kein privater Kommunikationsweg zum Eigentümer vorhanden ist, wird der Befehl sicher abgebrochen und der
Eigentümer aufgefordert, ihn aus einer Direktnachricht auszuführen.

Wenn die aktive Sitzung die native OpenAI-Codex-Ausführungsumgebung verwendet, deckt dieselbe
Ausführungsgenehmigung auch das Hochladen von OpenAI-Feedback für die OpenClaw bekannten Codex-Threads ab.
Dieser Upload ist vom lokalen Gateway-ZIP-Paket getrennt und erfolgt nur
bei Sitzungen der Codex-Ausführungsumgebung. Die Genehmigungsaufforderung weist darauf hin, dass eine Genehmigung
auch Codex-Feedback sendet, ohne Codex-Sitzungs- oder Thread-IDs aufzulisten. Nach
der Genehmigung führt die Antwort Kanäle, OpenClaw-Sitzungs-IDs, Codex-Thread-IDs und
lokale Fortsetzungsbefehle für die an OpenAI gesendeten Threads auf. Wird die
Genehmigung abgelehnt oder ignoriert, werden der Export, das Hochladen des Codex-Feedbacks und die
Liste der Codex-IDs übersprungen.

Dadurch bleibt der Codex-Debugging-Ablauf kurz: Stellen Sie fehlerhaftes Verhalten in einem Kanal fest,
führen Sie `/diagnostics` aus, genehmigen Sie einmalig, teilen Sie den Bericht und führen Sie anschließend den ausgegebenen
Befehl `codex resume <thread-id>` lokal aus, wenn Sie den Thread
selbst untersuchen möchten. Siehe [Codex-Ausführungsumgebung](/de/plugins/codex-harness#inspect-codex-threads-locally).

## Inhalt des Exports

- `summary.md`: für Menschen lesbarer Überblick für den Support.
- `diagnostics.json`: maschinenlesbare Zusammenfassung von Konfiguration, Protokollen, Status, Integritätszustand
  und Stabilitätsdaten.
- `manifest.json`: Exportmetadaten und Dateiliste.
- Bereinigte Konfigurationsstruktur und nicht geheime Konfigurationsdetails.
- Bereinigte Protokollzusammenfassungen und aktuelle unkenntlich gemachte Protokollzeilen.
- Nach bestem Bemühen erstellte Momentaufnahmen von Gateway-Status und -Integritätszustand.
- `stability/latest.json`: neuestes persistiertes Stabilitätspaket, sofern verfügbar.

Der Export ist auch bei einem fehlerhaften Gateway nützlich: Wenn Status- oder Integritätsanfragen
fehlschlagen, werden lokale Protokolle, die Konfigurationsstruktur und das neueste Stabilitätspaket
weiterhin erfasst, sofern sie verfügbar sind.

## Datenschutzmodell

Beibehalten werden: Namen von Subsystemen, Plugin-IDs, Provider-IDs, Kanal-IDs, konfigurierte
Modi, Statuscodes, Zeitdauern, Byte-Anzahlen, Warteschlangenstatus, Speichermesswerte,
bereinigte Protokollmetadaten, unkenntlich gemachte Betriebsmeldungen, Konfigurationsstruktur und
nicht geheime Funktionseinstellungen.

Ausgelassen oder unkenntlich gemacht werden: Chattext, Prompts, Anweisungen, Webhook-Inhalte, Werkzeug-
ausgaben, Zugangsdaten, API-Schlüssel, Tokens, Cookies, geheime Werte, unverarbeitete
Anfrage-/Antwortinhalte, Konto-IDs, Nachrichten-IDs, unverarbeitete Sitzungs-IDs,
Hostnamen und lokale Benutzernamen.

Wenn eine Protokollmeldung wie Nutzereingaben, Chat-, Prompt- oder Werkzeugnutzdaten aussieht, enthält der
Export lediglich den Hinweis, dass eine Nachricht ausgelassen wurde, sowie deren Byte-Anzahl.

## Stabilitätsaufzeichnung

Der Gateway zeichnet standardmäßig einen begrenzten stabilitätsbezogenen Datenstrom ohne Nutzdaten auf, wenn
Diagnosen aktiviert sind. Er erfasst Betriebsfakten, keine Inhalte.

Derselbe Heartbeat prüft außerdem die Aktivität, wenn die Ereignisschleife oder CPU
ausgelastet erscheint, und gibt `diagnostic.liveness.warning`-Ereignisse mit Ereignisschleifenverzögerung,
Ereignisschleifenauslastung, CPU-Kern-Verhältnis, Anzahlen aktiver/wartender/eingereihter Sitzungen,
der aktuellen Start-/Laufzeitphase (sofern bekannt), aktuellen Phasenzeitspannen und
begrenzten Arbeitsbezeichnungen aus. Diese werden nur dann zu Gateway-Protokollzeilen der Stufe `warn`,
wenn Arbeit wartet oder eingereiht ist oder wenn aktive Arbeit mit einer anhaltenden Ereignisschleifenverzögerung
zusammenfällt; andernfalls werden sie mit `debug` protokolliert. Aktivitätsmessungen im Leerlauf werden weiterhin
als Diagnoseereignisse aufgezeichnet, führen für sich genommen jedoch nie zu einer Warnung.

Startphasen geben `diagnostic.phase.completed`-Ereignisse mit verstrichener Echtzeit und
CPU-Zeitmessung aus. Diagnosen für hängende eingebettete Ausführungen kennzeichnen `terminalProgressStale=true`,
wenn der letzte Bridge-Fortschritt abgeschlossen wirkte (beispielsweise ein unverarbeitetes Antwort-
element oder ein Ereignis zum Abschluss der Antwort), der Gateway die
eingebettete Ausführung jedoch weiterhin als aktiv betrachtet.

Prüfen Sie die laufende Aufzeichnung:

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --json
```

Prüfen Sie das neueste persistierte Paket nach einem fatalen Abbruch, einer Zeitüberschreitung beim Herunterfahren oder
einem Startfehler beim Neustart:

```bash
openclaw gateway stability --bundle latest
```

Erstellen Sie eine Diagnose-ZIP-Datei aus dem neuesten persistierten Paket:

```bash
openclaw gateway stability --bundle latest --export
```

Persistierte Pakete befinden sich unter `~/.openclaw/logs/stability/`, sofern Ereignisse vorhanden sind.

## Nützliche Optionen

```bash
openclaw gateway diagnostics export \
  --output openclaw-diagnostics.zip \
  --log-lines 5000 \
  --log-bytes 1000000
```

| Flag                    | Standardwert                                                                  | Beschreibung                                               |
| ----------------------- | ----------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `--output <path>`       | `$OPENCLAW_STATE_DIR/logs/support/openclaw-diagnostics-<timestamp>-<pid>.zip` | In einen bestimmten ZIP-Pfad (oder ein Verzeichnis) schreiben. |
| `--log-lines <count>`   | `5000`                                                                        | Maximale Anzahl einzuschließender bereinigter Protokollzeilen.  |
| `--log-bytes <bytes>`   | `1000000`                                                                     | Maximale Anzahl zu untersuchender Protokollbytes.               |
| `--url <url>`           | -                                                                             | Gateway-WebSocket-URL für Status-/Integritätsmomentaufnahmen.   |
| `--token <token>`       | -                                                                             | Gateway-Token für Status-/Integritätsmomentaufnahmen.           |
| `--password <password>` | -                                                                             | Gateway-Passwort für Status-/Integritätsmomentaufnahmen.        |
| `--timeout <ms>`        | `3000`                                                                        | Zeitüberschreitung für Status-/Integritätsmomentaufnahmen.      |
| `--no-stability-bundle` | aus                                                                           | Suche nach persistiertem Stabilitätspaket überspringen.         |
| `--json`                | aus                                                                           | Maschinenlesbare Exportmetadaten ausgeben.                      |

## Diagnosen deaktivieren

Diagnosen sind standardmäßig aktiviert. So deaktivieren Sie die Stabilitätsaufzeichnung und
die Erfassung von Diagnoseereignissen:

```json5
{
  diagnostics: {
    enabled: false,
  },
}
```

Das Deaktivieren von Diagnosen verringert die Detailtiefe von Fehlerberichten; die normale
Gateway-Protokollierung wird dadurch nicht beeinträchtigt.

Ereignisse bei Speicherdruck zeichnen RSS-, Heap-, Schwellenwert- und Wachstumsdaten
(`rss_threshold`, `heap_threshold`, `rss_growth`) auf, ohne einen
Dateisystemscan durchzuführen oder eine Momentaufnahme vor einem Speichermangel zu schreiben.

## Verwandte Themen

- [Integritätsprüfungen](/de/gateway/health)
- [Gateway-CLI](/de/cli/gateway#gateway-diagnostics-export)
- [Gateway-Protokoll](/de/gateway/protocol#rpc-method-families)
- [Protokollierung](/de/logging)
- [OpenTelemetry-Export](/de/gateway/opentelemetry) - separater Ablauf zum Streamen von Diagnosedaten an einen Collector
