---
read_when:
    - Sie benötigen einen einsteigerfreundlichen Überblick über die Protokollierung in OpenClaw
    - Sie möchten Protokollierungsstufen, Formate oder Schwärzung konfigurieren
    - Sie führen eine Fehlerbehebung durch und müssen schnell Protokolle finden
summary: Dateiprotokolle, Konsolenausgabe, CLI-Liveanzeige und die Registerkarte „Protokolle“ der Control UI
title: Protokollierung
x-i18n:
    generated_at: "2026-07-26T18:26:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2c9303c4bc9c0797ca9c5775a281dce95229661b61d710425b2f7bec182b2e75
    source_path: logging.md
    workflow: 16
---

OpenClaw hat zwei zentrale Log-Oberflächen:

- **Datei-Logs** (JSON-Zeilen), die vom Gateway geschrieben werden.
- **Konsolenausgabe** im Terminal, in dem das Gateway ausgeführt wird.

Der Tab **Logs** der Control UI verfolgt das Gateway-Datei-Log fortlaufend. Diese Seite erläutert, wo
sich Logs befinden, wie sie gelesen werden und wie Log-Level und -Formate konfiguriert werden.

## Speicherort der Logs

Standardmäßig schreibt das Gateway pro Tag eine fortlaufende Log-Datei. Das Standardprofil
behält den bisherigen Pfad bei:

`/tmp/openclaw/openclaw-YYYY-MM-DD.log`

Benannte Profile verwenden im selben Verzeichnis einen profilbezogenen Dateinamen:

`/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`

Das Profilsegment im Dateinamen besteht aus Kleinbuchstaben und ist auf Buchstaben, Zahlen und
Bindestriche beschränkt. Einfache kleingeschriebene Namen bleiben lesbar, sodass die Kurzform `--dev`
in `openclaw-dev-YYYY-MM-DD.log` schreibt. Groß-/Kleinschreibung, Unterstriche und literale Bindestriche verwenden eine
umkehrbare Bindestrich-Escapesequenz, damit unterschiedliche Profilnamen niemals dieselbe Log-Datei verwenden.
Überlange Werte, die direkt über die Umgebung festgelegt werden, erhalten ein begrenztes Hash-Suffix,
um die Dateinamenlimits des Dateisystems einzuhalten. Ein expliziter Wert für `logging.file` überschreibt
diese Standardwerte.

Das Datum verwendet die lokale Zeitzone des Gateway-Hosts. Wenn `/tmp/openclaw` unsicher
oder nicht verfügbar ist (und unter Windows immer), verwendet OpenClaw stattdessen ein benutzerspezifisches
Verzeichnis `openclaw-<uid>` unterhalb des temporären Verzeichnisses des Betriebssystems. Datierte Log-Dateien werden
nach 24 Stunden gelöscht.

Jede Datei wird rotiert, wenn der nächste Schreibvorgang `logging.maxFileBytes`
überschreiten würde (Standard: 100 MB). OpenClaw bewahrt neben der
aktiven Datei bis zu fünf nummerierte Archive auf, beispielsweise `openclaw-YYYY-MM-DD.1.log` oder
`openclaw-dev-YYYY-MM-DD.1.log`, und schreibt in eine neue aktive Log-Datei weiter,
anstatt Diagnoseinformationen zu unterdrücken.

Sie können den Pfad in `~/.openclaw/openclaw.json` überschreiben:

```json
{
  "logging": {
    "file": "/path/to/openclaw.log"
  }
}
```

## Logs lesen

### CLI: Live-Verfolgung (empfohlen)

Verfolgen Sie die Gateway-Log-Datei über RPC:

```bash
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
```

Die Profilauswahl auf Root-Ebene löst dieselbe profilspezifische Datei auf, die vom
Gateway verwendet wird, einschließlich CLI-Fallback-Lesezugriffen, wenn lokales RPC nicht verfügbar ist.

Optionen:

| Flag                | Standard | Verhalten                                                                             |
| ------------------- | -------- | ------------------------------------------------------------------------------------- |
| `--follow`          | aus      | Verfolgung fortsetzen; stellt nach einer Trennung mit Backoff erneut eine Verbindung her |
| `--limit <n>`       | `200`    | Maximale Zeilen pro Abruf                                                             |
| `--max-bytes <n>`   | `250000` | Maximale zu lesende Byteanzahl pro Abruf                                              |
| `--interval <ms>`   | `1000`   | Abfrageintervall während der Verfolgung                                               |
| `--json`            | aus      | Zeilengetrenntes JSON (ein Ereignis pro Zeile)                                        |
| `--plain`           | aus      | Erzwingt Klartext in TTY-Sitzungen                                                    |
| `--no-color`        | —        | Deaktiviert ANSI-Farben                                                               |
| `--utc`             | aus      | Stellt Zeitstempel in UTC dar (lokale Zeit ist Standard)                              |
| `--local-time`      | aus      | Akzeptierte Kompatibilitätsschreibweise für den Standard „lokale Zeit“; darüber hinaus keine Wirkung |
| `--url` / `--token` | —        | Standardmäßige Gateway-RPC-Flags                                                      |
| `--timeout <ms>`    | `30000`  | Gateway-RPC-Zeitüberschreitung                                                        |
| `--expect-final`    | aus      | Agent-gestütztes RPC-Flag zum Warten auf die endgültige Antwort (wird hier über die gemeinsame Client-Schicht akzeptiert) |

Ausgabemodi:

- **TTY-Sitzungen**: ansprechend formatierte, farbige und strukturierte Log-Zeilen.
- **Nicht-TTY-Sitzungen**: Klartext.

Wenn Sie einen expliziten Wert für `--url` übergeben, wendet die CLI Konfigurations- oder
Umgebungs-Anmeldedaten nicht automatisch an. Geben Sie `--token` selbst an, andernfalls schlägt der Aufruf mit
`gateway url override requires explicit credentials` fehl.

Im JSON-Modus gibt die CLI mit `type` gekennzeichnete Objekte aus:

- `meta`: Stream-Metadaten (Datei, Quelle, Quelltyp, Dienst, Cursor, Größe)
- `log`: analysierter Log-Eintrag
- `notice`: Hinweise auf Kürzung/Rotation
- `raw`: nicht analysierte Log-Zeile
- `error`: Gateway-Verbindungsfehler (werden nach stderr geschrieben)

Wenn das implizite lokale Loopback-Gateway eine Kopplung anfordert, während des Verbindungsaufbaus
geschlossen wird oder eine Zeitüberschreitung eintritt, bevor `logs.tail` antwortet, greift `openclaw logs` automatisch auf die
konfigurierte Gateway-Datei-Log-Datei zurück. Explizite Ziele für `--url` verwenden
diesen Fallback nicht. `openclaw logs --follow` ist strenger: Unter Linux verwendet es, sofern verfügbar, das aktive
Benutzer-systemd-Journal des Gateways anhand der PID und versucht andernfalls mit
Backoff erneut, eine Verbindung zum aktiven Gateway herzustellen, statt eine möglicherweise veraltete, parallel abgelegte
Datei zu verfolgen.

Wenn das Gateway nicht erreichbar ist, gibt die CLI einen kurzen Hinweis zur Ausführung von Folgendem aus:

```bash
openclaw doctor
```

### Control UI (Web)

Der Tab **Logs** der Control UI verfolgt dieselbe Datei mithilfe von `logs.tail`.
Informationen zum Öffnen finden Sie unter [Control UI](/de/web/control-ui).

### Nur Kanal-Logs

Um Kanalaktivitäten (WhatsApp/Telegram usw.) zu filtern, verwenden Sie:

```bash
openclaw channels logs --channel whatsapp
```

`--channel` verwendet standardmäßig `all`; `--lines <n>` (Standard: 200) und `--json` sind ebenfalls
verfügbar.

## Log-Formate

### Datei-Logs (JSONL)

Jede Zeile in der Log-Datei ist ein JSON-Objekt. Die CLI und die Control UI analysieren diese
Einträge, um eine strukturierte Ausgabe darzustellen (Zeit, Level, Subsystem, Nachricht).

JSONL-Datensätze in Datei-Logs enthalten, sofern verfügbar, außerdem maschinenfilterbare Felder
auf oberster Ebene:

- `hostname`: Hostname des Gateways.
- `message`: vereinfachter Log-Nachrichtentext für die Volltextsuche.
- `agent_id`: aktive Agent-ID, wenn der Log-Aufruf Agent-Kontext enthält.
- `session_id`: aktive Sitzungs-ID bzw. aktiver Sitzungsschlüssel, wenn der Log-Aufruf Sitzungskontext enthält.
- `channel`: aktiver Kanal, wenn der Log-Aufruf Kanalkontext enthält.

OpenClaw bewahrt neben diesen Feldern die ursprünglichen strukturierten Log-Argumente auf,
damit vorhandene Parser, die nummerierte tslog-Argumentschlüssel lesen, weiterhin funktionieren.

Aktivitäten von Talk, Echtzeit-Sprachkommunikation und verwalteten Räumen geben begrenzte Lebenszyklus-Log-
Datensätze über dieselbe Datei-Log-Pipeline aus. Diese Datensätze enthalten Ereignistyp,
Modus, Transport, Provider sowie Größen-/Zeitmessungen, sofern verfügbar, lassen jedoch
Transkripttext, Audio-Nutzdaten, Turn-IDs, Anruf-IDs und Provider-Element-IDs aus.

### Konsolenausgabe

Konsolen-Logs sind **TTY-fähig** und für bessere Lesbarkeit formatiert:

- Subsystempräfixe (z. B. `gateway/channels/whatsapp`)
- Level-Farben (Info/Warnung/Fehler)
- Optionaler kompakter oder JSON-Modus

Die Konsolenformatierung wird durch `logging.consoleStyle` gesteuert.

### Gateway-WebSocket-Logs

`openclaw gateway` bietet außerdem WebSocket-Protokollierung für RPC-Datenverkehr:

- Normalmodus: nur relevante Ergebnisse (Fehler, Analysefehler, langsame Aufrufe)
- `--verbose`: gesamter Anfrage-/Antwortdatenverkehr
- `--ws-log auto|compact|full`: ausführlichen Darstellungsstil auswählen
- `--compact`: Alias für `--ws-log compact`

Beispiele:

```bash
openclaw gateway
openclaw gateway --verbose --ws-log compact
openclaw gateway --verbose --ws-log full
```

## Logging konfigurieren

Die gesamte Logging-Konfiguration befindet sich unter `logging` in `~/.openclaw/openclaw.json`.

```json
{
  "logging": {
    "level": "info",
    "file": "/path/to/openclaw.log",
    "consoleLevel": "info",
    "consoleStyle": "pretty",
    "redactSensitive": "tools",
    "redactPatterns": ["sk-.*"]
  }
}
```

### Log-Level

Level: `silent`, `fatal`, `error`, `warn`, `info`, `debug`, `trace`.

- `logging.level`: Level der **Datei-Logs** (JSONL) (Standard: `info`).
- `logging.consoleLevel`: Ausführlichkeitslevel der **Konsole**.

Sie können beide über die Umgebungsvariable **`OPENCLAW_LOG_LEVEL`** überschreiben (z. B. `OPENCLAW_LOG_LEVEL=debug`). Die Umgebungsvariable hat Vorrang vor der Konfigurationsdatei, sodass Sie die Ausführlichkeit für einen einzelnen Lauf erhöhen können, ohne `openclaw.json` zu bearbeiten. Sie können auch die globale CLI-Option **`--log-level <level>`** übergeben (beispielsweise `openclaw --log-level debug gateway run`), die für diesen Befehl die Umgebungsvariable überschreibt.

`--verbose` wirkt sich nur auf die Konsolenausgabe und die Ausführlichkeit des WS-Logs aus; es ändert
die Level der Datei-Logs nicht.

### Gezielte Modelltransportdiagnose

Verwenden Sie beim Debuggen von Provider-Aufrufen gezielte Umgebungs-Flags, statt
alle Logs auf `debug` zu erhöhen:

```bash
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 openclaw gateway
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools OPENCLAW_DEBUG_SSE=events openclaw gateway
```

Verfügbare Flags:

- `OPENCLAW_DEBUG_MODEL_TRANSPORT=1`: gibt Anfragestart, Fetch-Antwort, SDK-
  Header, erstes Streaming-Ereignis, Stream-Abschluss und Transportfehler auf
  dem Level `info` aus.
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=summary`: nimmt eine begrenzte Zusammenfassung der Anfrage-Nutzdaten
  in Modellanfrage-Logs auf.
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=tools`: nimmt alle Namen modellseitiger Tools in
  die Nutzdatenzusammenfassung auf.
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted`: nimmt einen redigierten, größenbegrenzten JSON-
  Snapshot der Nutzdaten auf. Nur beim Debuggen verwenden; Geheimnisse werden redigiert, aber Prompts
  und Nachrichtentext können weiterhin enthalten sein.
- `OPENCLAW_DEBUG_SSE=events`: gibt die Zeitmessung des ersten Ereignisses und des Stream-Abschlusses aus.
- `OPENCLAW_DEBUG_SSE=peek`: gibt außerdem die ersten fünf redigierten SSE-Ereignis-
  Nutzdaten aus, jeweils größenbegrenzt.
- `OPENCLAW_DEBUG_CODE_MODE=1`: gibt Diagnoseinformationen zur Modelloberfläche im Code-Modus aus,
  einschließlich der Fälle, in denen native Provider-Tools ausgeblendet werden, weil der Code-Modus die
  Tool-Oberfläche verwaltet.

Diese Flags protokollieren über das normale OpenClaw-Logging, sodass `openclaw logs --follow`
und der Tab „Logs“ der Control UI sie anzeigen. Ohne die Flags bleiben dieselben Diagnoseinformationen
auf dem Level `debug` verfügbar.

`[model-fetch]` Start- und Antwortmetadaten (Provider, API, Modell, Status,
Latenz und Anfragefelder wie Methode, URL, Zeitüberschreitung, Proxy und Richtlinie)
werden unabhängig von `OPENCLAW_DEBUG_MODEL_TRANSPORT` immer auf dem Level `info`
ausgegeben, sodass grundlegende Modelltransportdiagnosen
ohne Debug-Flags sichtbar sind.

### Trace-Korrelation

Datei-Logs sind JSONL. Wenn ein Log-Aufruf einen gültigen Diagnose-Trace-Kontext enthält,
schreibt OpenClaw die Trace-Felder als JSON-Schlüssel auf oberster Ebene (`traceId`, `spanId`,
`parentSpanId`, `traceFlags`), damit externe Log-Prozessoren die Zeile
mit OTEL-Spans und der Provider-Weitergabe von `traceparent` korrelieren können.

Gateway-HTTP-Anfragen und Gateway-WebSocket-Frames richten einen internen Anfrage-
Trace-Bereich ein. Logs und Diagnoseereignisse, die innerhalb dieses asynchronen Bereichs ausgegeben
werden, übernehmen den Anfrage-Trace, wenn sie keinen expliziten Trace-Kontext übergeben. Traces von Agent-Läufen und
Modellaufrufen werden zu untergeordneten Traces des aktiven Anfrage-Traces, sodass lokale Logs,
Diagnose-Snapshots, OTEL-Spans und vertrauenswürdige Provider-Header `traceparent`
über `traceId` miteinander verknüpft werden können, ohne rohe Anfrage- oder Modellinhalte zu protokollieren.

Talk-Lebenszyklus-Log-Datensätze werden ebenfalls an den diagnostics-otel-Log-Export weitergeleitet, wenn
der OpenTelemetry-Log-Export aktiviert ist, und verwenden dieselben begrenzten Attribute wie Datei-
Logs. Konfigurieren Sie `diagnostics.otel.logsExporter`, um OTLP, stdout-JSONL oder
beide Ziele auszuwählen.

### Größe und Zeitmessung von Modellaufrufen

Diagnoseinformationen zu Modellaufrufen erfassen begrenzte Anfrage-/Antwortmesswerte, ohne
rohe Prompt- oder Antwortinhalte zu erfassen:

- `requestPayloadBytes`: UTF-8-Byte-Größe der endgültigen Modellanfrage-Nutzlast
- `responseStreamBytes`: UTF-8-Byte-Größe des gestreamten Modellantwort-Fragments
  der Nutzlasten. Hochfrequente Text-, Denk- und Tool-Aufruf-Delta-Ereignisse zählen
  nur die inkrementellen `delta`-Bytes anstelle vollständiger `partial`-Momentaufnahmen.
- `timeToFirstByteMs`: verstrichene Zeit bis zum ersten gestreamten Antwortereignis
- `durationMs`: Gesamtdauer des Modellaufrufs

Diese Felder stehen Diagnose-Momentaufnahmen, Plugin-Hooks für Modellaufrufe und
OTEL-Spans/-Metriken für Modellaufrufe zur Verfügung, wenn der Diagnoseexport aktiviert ist.

### Konsolenstile

`logging.consoleStyle`:

- `pretty`: benutzerfreundlich, farbig und mit Zeitstempeln.
- `compact`: kompaktere Ausgabe (am besten für lange Sitzungen).
- `json`: eine JSON-Struktur pro Zeile (für Log-Verarbeitungsprogramme).

### Schwärzung

OpenClaw kann sensible Tokens schwärzen, bevor sie in der Konsolenausgabe, in Datei-Logs,
in OTLP-Log-Datensätzen, im Text persistierter Sitzungstranskripte oder in Tool-
Ereignisnutzlasten der Control UI erscheinen (Argumente beim Tool-Start, partielle/endgültige
Ergebnisnutzlasten, abgeleitete Ausführungsausgaben und Patch-Zusammenfassungen):

- Die Schwärzung sensibler Werte ist immer aktiviert.
- `logging.redactPatterns`: Liste von Regex-Zeichenfolgen, welche die Standardmenge für die Log-/Transkriptausgabe ersetzt. Bei Tool-Nutzlasten der Control UI werden benutzerdefinierte Muster zusätzlich zu den integrierten Standardmustern angewendet. Das Hinzufügen eines Musters schwächt daher niemals die Schwärzung von Werten ab, die bereits von den Standardmustern erfasst werden.

Datei-Logs und Sitzungstranskripte bleiben im JSONL-Format, übereinstimmende geheime Werte werden jedoch
maskiert, bevor die Zeile oder Nachricht auf den Datenträger geschrieben wird. Die Schwärzung erfolgt nach bestem Bemühen:
Sie gilt für texttragende Nachrichteninhalte und Log-Zeichenfolgen, nicht für jedes
Bezeichner- oder Binärnutzlastfeld.

Die integrierten Standardmuster decken gängige API-Anmeldedaten und Feldnamen für Zahlungsanmeldedaten
wie Kartennummer, CVC/CVV, gemeinsam verwendetes Zahlungstoken und Zahlungsanmeldedaten ab,
wenn sie als JSON-Felder, URL-Parameter, CLI-Flags oder Zuweisungen erscheinen.

OpenClaw schwärzt außerdem Nutzlasten an Sicherheitsgrenzen, die UI-Clients, Support-
Paketen, Diagnosebeobachtern, Genehmigungsaufforderungen oder Agenten-Tools angezeigt werden. Benutzerdefinierte
`logging.redactPatterns` können diesen Oberflächen projektspezifische Muster hinzufügen.

## Diagnose und OpenTelemetry

Diagnosen sind strukturierte, maschinenlesbare Ereignisse für Modellläufe und
Nachrichtenfluss-Telemetrie (Webhooks, Warteschlangenbildung, Sitzungsstatus). Sie ersetzen
Logs **nicht** – sie speisen Metriken, Traces und Exporter. Ereignisse werden
standardmäßig prozessintern ausgegeben (setzen Sie `diagnostics.enabled: false`, um sie zu deaktivieren);
ihr Export wird separat konfiguriert.

Zwei benachbarte Oberflächen:

- **OpenTelemetry-Export** – sendet Metriken, Traces und Logs über OTLP/HTTP an
  beliebige OpenTelemetry-kompatible Collectors oder Backends (Datadog, Grafana,
  Honeycomb, New Relic, Tempo usw.). Die vollständige Konfiguration, der Signalkatalog,
  Metrik-/Span-Namen, Umgebungsvariablen und das Datenschutzmodell befinden sich auf einer eigenen Seite:
  [OpenTelemetry-Export](/de/gateway/opentelemetry).
- **Diagnose-Flags** – gezielte Debug-Log-Flags, die zusätzliche Logs an
  `logging.file` weiterleiten, ohne `logging.level` zu erhöhen. Bei Flags wird nicht zwischen Groß- und Kleinschreibung unterschieden,
  und sie unterstützen Platzhalter (`telegram.*`, `*`). Konfigurieren Sie sie unter `diagnostics.flags`
  oder über die Umgebungsvariablen-Überschreibung `OPENCLAW_DIAGNOSTICS=...`. Vollständige Anleitung:
  [Diagnose-Flags](/de/diagnostics/flags).

Informationen zum OTLP-Export an einen Collector finden Sie unter [OpenTelemetry-Export](/de/gateway/opentelemetry).

## Tipps zur Fehlerbehebung

- **Gateway nicht erreichbar?** Führen Sie zuerst `openclaw doctor` aus.
- **Logs leer?** Prüfen Sie, ob der Gateway ausgeführt wird und in den Dateipfad
  unter `logging.file` schreibt.
- **Benötigen Sie mehr Details?** Setzen Sie `logging.level` auf `debug` oder `trace` und versuchen Sie es erneut.

## Verwandte Themen

- [OpenTelemetry-Export](/de/gateway/opentelemetry) – OTLP/HTTP-Export, Metrik-/Span-Katalog, Datenschutzmodell
- [Diagnose-Flags](/de/diagnostics/flags) – gezielte Debug-Log-Flags
- [Interna der Gateway-Protokollierung](/de/gateway/logging) – WS-Log-Stile, Subsystempräfixe und Konsolenerfassung
- [Konfigurationsreferenz](/de/gateway/configuration-reference#diagnostics) – vollständige Referenz zum Feld `diagnostics.*`
