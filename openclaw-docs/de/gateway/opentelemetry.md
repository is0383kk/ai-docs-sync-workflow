---
read_when:
    - Sie möchten Metriken zur OpenClaw-Modellnutzung, zum Nachrichtenfluss oder zu Sitzungen an einen OpenTelemetry-Collector senden
    - Sie binden Traces, Metriken oder Protokolle in Grafana, Datadog, Honeycomb, New Relic, Tempo oder ein anderes OTLP-Backend ein
    - Sie benötigen die exakten Metriknamen, Span-Namen oder Attributstrukturen, um Dashboards oder Warnmeldungen zu erstellen.
summary: Exportieren Sie OpenClaw-Diagnosedaten über das diagnostics-otel-Plugin an OpenTelemetry-Collectors oder als JSONL nach stdout.
title: OpenTelemetry-Export
x-i18n:
    generated_at: "2026-07-26T17:48:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6ed37f094c6c151379d8e0aaa2633b3ebebdb08b7dcbc9403c4bdeb6e5b8cf76
    source_path: gateway/opentelemetry.md
    workflow: 16
---

OpenClaw exportiert Diagnosedaten über das offizielle `diagnostics-otel`-Plugin
mittels **OTLP/HTTP (protobuf)**. Logs können für
Container- und Sandbox-Log-Pipelines auch als JSONL in stdout geschrieben werden. Jeder Collector oder jedes Backend, der bzw. das
OTLP/HTTP akzeptiert, funktioniert ohne Codeänderungen. Informationen zu lokalen Datei-Logs finden Sie unter
[Protokollierung](/de/logging).

- **Diagnoseereignisse** sind strukturierte, prozessinterne Datensätze, die vom
  Gateway und den gebündelten Plugins für Modellläufe, Nachrichtenfluss, Sitzungen, Warteschlangen
  und exec ausgegeben werden.
- **`diagnostics-otel`** abonniert diese Ereignisse und exportiert sie als
  OpenTelemetry-**Metriken**, -**Traces** und -**Logs** über OTLP/HTTP und kann
  Log-Datensätze als JSONL nach stdout spiegeln.
- **Provider-Aufrufe** erhalten einen W3C-`traceparent`-Header aus OpenClaws
  vertrauenswürdigem Span-Kontext des Modellaufrufs, wenn der Provider-Transport benutzerdefinierte
  Header akzeptiert. Von Plugins ausgegebener Trace-Kontext wird nicht weitergegeben.
- Exporter werden nur eingebunden, wenn sowohl die Diagnoseoberfläche als auch das Plugin
  aktiviert sind, sodass die prozessinternen Kosten standardmäßig nahezu null bleiben.

## Schnellstart

```bash
openclaw plugins install clawhub:@openclaw/diagnostics-otel
```

```json5
{
  plugins: {
    allow: ["diagnostics-otel"],
    entries: {
      "diagnostics-otel": { enabled: true },
    },
  },
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      protocol: "http/protobuf",
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: true,
      sampleRate: 0.2,
      flushIntervalMs: 60000,
    },
  },
}
```

Oder aktivieren Sie das Plugin über die CLI: `openclaw plugins enable diagnostics-otel`.

<Note>
`protocol` unterstützt ausschließlich `http/protobuf`. Da `traces` und `metrics` standardmäßig aktiviert sind, bricht jeder andere Wert (einschließlich `grpc`) das gesamte diagnostics-otel-Abonnement mit einer `unsupported protocol`-Warnung ab – dadurch wird auch der stdout-Log-Export beendet. Setzen Sie `traces: false` und `metrics: false` explizit, wenn Sie nur `logsExporter: "stdout"` mit einem anderen Protokollwert als OTLP verwenden möchten.
</Note>

## Exportierte Signale

| Signal      | Enthaltene Daten                                                                                                                                                                                              |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Metriken** | Zähler/Histogramme für Token-Nutzung, Kosten, Laufdauer, Failover, Skill-Nutzung, Nachrichtenfluss, Talk-Ereignisse, Warteschlangen-Lanes, Sitzungsstatus/-wiederherstellung, Tool-Ausführung, exec, Speicher, Verfügbarkeit und Exporter-Zustand. |
| **Traces**  | Spans für Modellnutzung, Modellaufrufe, Harness-Lebenszyklus, Skill-Nutzung, Tool-Ausführung, exec, Webhook-/Nachrichtenverarbeitung, Kontextzusammenstellung und Tool-Schleifen.                                                      |
| **Logs**    | Strukturierte `logging.file`-Datensätze, die über OTLP oder als stdout-JSONL exportiert werden, wenn `diagnostics.otel.logs` aktiviert ist; Log-Inhalte werden zurückgehalten, sofern die Inhaltserfassung nicht ausdrücklich aktiviert ist.                          |

Schalten Sie `traces`, `metrics` und `logs` unabhängig voneinander um. Traces und Metriken
sind standardmäßig aktiviert, wenn `diagnostics.otel.enabled` wahr ist; Logs sind standardmäßig deaktiviert
und werden nur exportiert, wenn `diagnostics.otel.logs` ausdrücklich auf `true` gesetzt ist. Der Log-Export
verwendet standardmäßig OTLP; setzen Sie `diagnostics.otel.logsExporter` auf `stdout` für JSONL auf
stdout oder auf `both` für beides.

## Konfigurationsreferenz

```json5
{
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      tracesEndpoint: "http://otel-collector:4318/v1/traces",
      metricsEndpoint: "http://otel-collector:4318/v1/metrics",
      logsEndpoint: "http://otel-collector:4318/v1/logs",
      protocol: "http/protobuf", // grpc deaktiviert den OTLP-Export
      serviceName: "openclaw-gateway", // wenn nicht gesetzt, wird OTEL_SERVICE_NAME verwendet, danach "openclaw"
      headers: { "x-collector-token": "..." },
      traces: true,
      metrics: true,
      logs: true,
      logsExporter: "otlp", // otlp | stdout | both
      sampleRate: 0.2, // Root-Span-Sampler, 0.0..1.0
      flushIntervalMs: 60000, // Metrikexportintervall (mindestens 1000ms)
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },
  },
}
```

### Umgebungsvariablen

| Variable                                                                                                          | Zweck                                                                                                                                                                                                                                                                                                        |
| ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                                                                                     | Fallback für `diagnostics.otel.endpoint`, wenn der Konfigurationsschlüssel nicht gesetzt ist.                                                                                                                                                                                                                                         |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` / `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` / `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` | Signalspezifische Endpunkt-Fallbacks, die verwendet werden, wenn der entsprechende `diagnostics.otel.*Endpoint`-Konfigurationsschlüssel nicht gesetzt ist. Die signalspezifische Konfiguration hat Vorrang vor der signalspezifischen Umgebungsvariable, die wiederum Vorrang vor dem gemeinsamen Endpunkt hat.                                                                                                         |
| `OTEL_SERVICE_NAME`                                                                                               | Fallback für `diagnostics.otel.serviceName`, wenn der Konfigurationsschlüssel nicht gesetzt ist. Der standardmäßige Dienstname lautet `openclaw`.                                                                                                                                                                                                  |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                                                                                     | Fallback für das Übertragungsprotokoll, wenn `diagnostics.otel.protocol` nicht gesetzt ist. Nur `http/protobuf` aktiviert den Export.                                                                                                                                                                                                 |
| `OTEL_SEMCONV_STABILITY_OPT_IN`                                                                                   | Setzen Sie den Wert auf `gen_ai_latest_experimental`, um die neueste GenAI-Inferenz-Span-Struktur auszugeben: `{gen_ai.operation.name} {gen_ai.request.model}`-Span-Namen, `CLIENT`-Span-Typ und `gen_ai.provider.name` anstelle des veralteten `gen_ai.system`. GenAI-Metriken verwenden unabhängig davon immer begrenzte Attribute mit niedriger Kardinalität. |
| `OPENCLAW_OTEL_PRELOADED`                                                                                         | Setzen Sie den Wert auf `1`, wenn ein anderer Preload- oder Host-Prozess bereits das globale OpenTelemetry SDK registriert hat. Das Plugin überspringt dann seinen eigenen NodeSDK-Lebenszyklus, bindet jedoch weiterhin Diagnose-Listener ein und berücksichtigt `traces`/`metrics`/`logs`.                                                                                    |

## Datenschutz und Inhaltserfassung

Unverarbeitete Modell-/Tool-Inhalte werden standardmäßig **nicht** exportiert. Spans enthalten begrenzte
Bezeichner (Kanal, Provider, Modell, Fehlerkategorie, ausschließlich gehashte Anfrage-IDs,
Tool-Quelle, Tool-Eigentümer, Skill-Name/-Quelle) und enthalten niemals Prompt-Text,
Antworttext, Tool-Eingaben, Tool-Ausgaben, Skill-Dateipfade oder Sitzungsschlüssel.
Werte, die wie bereichsgebundene Agent-Sitzungsschlüssel aussehen (zum Beispiel beginnend mit
`agent:`), werden in Attributen mit niedriger Kardinalität durch `unknown` ersetzt. OTLP-Log-
Datensätze behalten standardmäßig Schweregrad, Logger, Codestelle, vertrauenswürdigen Trace-Kontext und
bereinigte Attribute bei; der unverarbeitete Nachrichtentext des Logs wird nur exportiert,
wenn `diagnostics.otel.captureContent` den booleschen Wert `true` hat. Granulare
`captureContent.*`-Unterschlüssel aktivieren niemals Log-Inhalte. Talk-Metriken exportieren ausschließlich
begrenzte Ereignismetadaten (Modus, Transport, Provider, Ereignistyp) – keine
Transkripte, Audio-Nutzlasten, Sitzungs-IDs, Turn-IDs, Anruf-IDs, Raum-IDs oder
Übergabe-Token.

Ausgehende Modellanfragen können einen W3C-`traceparent`-Header enthalten, der ausschließlich
aus OpenClaw-eigenem Diagnose-Trace-Kontext für den aktiven Modellaufruf generiert wird.
Vorhandene, vom Aufrufer bereitgestellte `traceparent`-Header werden ersetzt, sodass Plugins oder
benutzerdefinierte Provider-Optionen keine dienstübergreifende Trace-Abstammung vortäuschen können.

Setzen Sie `diagnostics.otel.captureContent.*` nur dann auf `true`, wenn Ihr Collector
und Ihre Aufbewahrungsrichtlinie für Prompt-, Antwort-, Tool- oder
System-Prompt-Text genehmigt sind. Jeder Unterschlüssel ist unabhängig:

- `inputMessages` – Inhalt der Benutzeraufforderung.
- `outputMessages` – Inhalt der Modellantwort.
- `toolInputs` – Nutzlasten der Tool-Argumente.
- `toolOutputs` – Nutzlasten der Tool-Ergebnisse.
- `systemPrompt` – zusammengestellter System-/Entwickler-Prompt.
- `toolDefinitions` – Namen, Beschreibungen und Schemas der Modell-Tools.

Wenn ein beliebiger Unterschlüssel aktiviert ist, erhalten Modell- und Tool-Spans begrenzte, geschwärzte
`openclaw.content.*`-Attribute ausschließlich für diese Klasse.

<Note>
Der boolesche Wert `captureContent: true` aktiviert `inputMessages`, `outputMessages`, `toolInputs`, `toolOutputs`, `toolDefinitions` und OTLP-Log-Inhalte gemeinsam, jedoch **nicht** `systemPrompt` – setzen Sie `captureContent.systemPrompt: true` ausdrücklich, wenn Sie auch den zusammengestellten System-Prompt benötigen.
</Note>

`toolInputs`/`toolOutputs`-Inhalte werden für die Tool-Ausführungen der integrierten Agent-
Runtime erfasst (`openclaw.content.tool_input` und
`gen_ai.tool.call.arguments` bei abgeschlossenen/fehlerhaften Spans;
`openclaw.content.tool_output` und `gen_ai.tool.call.result` bei abgeschlossenen
Spans). Die `openclaw.content.*`-Namen bleiben die stabilen OpenClaw-Attributnamen;
die `gen_ai.tool.call.*`-Kopien spiegeln sie für Semconv-native Viewer.
Externe Harness-Tool-Aufrufe (Codex, Claude CLI) geben
`tool.execution.*`-Spans ohne Inhaltsnutzlasten aus. Erfasste Inhalte werden über einen
vertrauenswürdigen, ausschließlich für Listener bestimmten Kanal übertragen und niemals auf dem öffentlichen Diagnoseereignis-
Bus bereitgestellt.

## Sampling und Leerung

- **Traces:** `diagnostics.otel.sampleRate` legt nur für den Root-Span einen `TraceIdRatioBasedSampler`
  fest (`0.0` verwirft alle, `1.0` behält alle bei). Wenn nicht festgelegt, wird der
  Standard des OpenTelemetry SDK verwendet (immer aktiviert).
- **Metriken:** `diagnostics.otel.flushIntervalMs` (auf ein Minimum von
  `1000` begrenzt); wenn nicht festgelegt, wird der Standard des SDK für den periodischen Export verwendet.
- **Protokolle:** OTLP-Protokolle berücksichtigen `logging.level` (Dateiprotokollstufe) und verwenden den
  Schwärzungspfad für diagnostische Protokolldatensätze, nicht die Konsolenformatierung. Installationen mit hohem
  Datenaufkommen sollten Sampling/Filterung im OTLP-Collector gegenüber lokalem
  Sampling bevorzugen. Legen Sie `diagnostics.otel.logsExporter: "stdout"` fest, wenn Ihre Plattform
  stdout/stderr bereits an einen Protokollprozessor weiterleitet und Sie keinen Collector für OTLP-Protokolle
  haben. stdout-Datensätze bestehen aus einem JSON-Objekt pro Zeile mit `ts`, `signal`,
  `service.name`, Schweregrad, Inhalt, geschwärzten Attributen und vertrauenswürdigen Trace-
  Feldern, sofern verfügbar.
- **Dateiprotokoll-Korrelation:** JSONL-Dateiprotokolle enthalten `traceId`,
  `spanId`, `parentSpanId` und `traceFlags` auf der obersten Ebene, wenn der Protokollaufruf einen gültigen
  diagnostischen Trace-Kontext enthält. Dadurch können Protokollprozessoren lokale Protokollzeilen mit
  exportierten Spans verknüpfen.
- **Anfragekorrelation:** Gateway-HTTP-Anfragen und WebSocket-Frames erstellen
  einen internen Anfrage-Trace-Bereich. Protokolle und diagnostische Ereignisse innerhalb dieses
  Bereichs übernehmen standardmäßig den Anfrage-Trace, während Spans für Agent-Ausführungen und Modellaufrufe
  als untergeordnete Elemente erstellt werden, sodass die `traceparent`-Header des Providers im
  selben Trace verbleiben.
- **Modellaufruf-Korrelation:** `openclaw.model.call`-Spans enthalten standardmäßig unbedenkliche Größenangaben zu
  Prompt-Komponenten und Token-Attribute pro Aufruf, wenn das Provider-
  Ergebnis Nutzungsdaten bereitstellt. `openclaw.model.usage` bleibt der Span für die
  Abrechnung auf Ausführungsebene für aggregierte Kosten-, Kontext- und Kanal-Dashboards und
  verbleibt im selben diagnostischen Trace, wenn die ausgebende Laufzeitumgebung über einen vertrauenswürdigen
  Trace-Kontext verfügt.

### Beobachtungseinheiten für Modellaufrufe

Jeder `openclaw.model.call`-Span kennzeichnet über
`openclaw.model_call.observation_unit`, was sein Lebenszyklus misst:

- `request` – eine beobachtbare Modell-/Provider-Anfrage. Native eingebettete Modell-
  aufrufe verwenden diese Einheit, und Exporter behandeln einen fehlenden Wert zur
  Kompatibilität mit älteren oder externen Emittern als `request`.
- `turn` – ein undurchsichtiger Agent-CLI-Durchlauf, der verborgene Modellanfragen,
  Wiederholungsversuche, Tool-Arbeit oder Hintergrundarbeit enthalten kann. Aufrufe der Claude Code CLI und des Codex-App-Servers
  verwenden diese Einheit.

Beide Einheiten bleiben Modellaufruf-Spans, damit Trace-Backends Modelleingabe,
-ausgabe, Nutzung und Hierarchie darstellen können. Anfrage-Spans verwenden die von der API abgeleitete GenAI-Operation
(`chat`, `generate_content` oder `text_completion`), während Durchlauf-Spans
`gen_ai.operation.name = invoke_agent` verwenden. Beide fließen in
`gen_ai.client.operation.duration` ein, wobei der Operationsname die Latenz direkter
Anfragen von der Latenz vollständiger Durchläufe getrennt hält. Die OTEL-Modellaufruf-
Metriken von OpenClaw enthalten außerdem `openclaw.model_call.observation_unit`; die Prometheus-
Modellaufrufmetriken stellen das entsprechende Label `observation_unit` bereit.

### Genauigkeit der Modellaufrufe der Claude Code CLI

Durchläufe der Claude Code CLI geben einen synthetischen `openclaw.model.call`-
Span auf Durchlaufebene aus. Dies sind keine Anthropic-HTTP-Anfrage-Spans. Sie verwenden `openclaw.api =
claude-code`, `openclaw.model_call.observation_unit = turn` und kennzeichnen
die Operation als `gen_ai.operation.name = invoke_agent`. Sie kennzeichnen
die CLI-Grenze von OpenClaw über
`openclaw.transport`:

- `stdio` – einmaliger lokaler Claude-Code-Prozess.
- `stdio-live` – ein Durchlauf in einer verwalteten, persistenten Claude-stdio-Sitzung.
- `paired-node-cli` – einmalige Claude-Code-Ausführung, die an eine gekoppelte
  Node delegiert wird.

Claude-CLI-Diagnosen werden nur instanziiert, während der prozessweite Diagnose-
Dispatcher aktiviert und ein interner oder vertrauenswürdiger Ereignis-Listener angefügt ist.
Wenn kein Observability-Plugin oder anderer Listener aktiv ist, überspringen Claude-CLI-Durchläufe
die synthetische Trace-Hierarchie, Inhaltspuffer und die diagnostische Abrechnung der
Stream-Bytes. Wenn die Inhaltserfassung aktiviert ist, sind Prompt- und System-Prompt-Felder
jeweils auf 128 KiB begrenzt; die Assistentenausgabe ist über höchstens 200 Hüllen hinweg auf
128 KiB begrenzt, wobei 16 KiB und ein Element für eine abschließende sichtbare
Fallback-Antwort reserviert sind. Eine Markierung zeichnet die Kürzung auf, wenn das Limit erreicht wird.

OpenClaw versieht Claude-CLI-Durchläufe mit derselben Eigentümerhierarchie, die auch von anderen
Agent-Laufzeitumgebungen verwendet wird: `openclaw.harness.run` (`openclaw.harness.id = claude-cli`)
enthält `openclaw.run`, das den Claude-`openclaw.model.call`-
Span enthält. Die Harness- und Ausführungs-Spans sind synthetische OpenClaw-Durchlaufgrenzen, keine
internen Phasen von Claude Code. Einmalige und verwaltete stdio-Durchläufe verwenden dieselbe
Hierarchie; ein tatsächlicher Wiederholungsversuch mit einer neuen Sitzung erstellt ein weiteres untergeordnetes Modellaufrufelement innerhalb
derselben OpenClaw-Ausführung.

Der Span beginnt, wenn OpenClaw den vorbereiteten CLI-Durchlauf annimmt, und endet erst,
nachdem dieser Durchlauf erfolgreich war oder fehlgeschlagen ist. Bei verwalteten Sitzungen
beendet ein vorläufiges Erfolgsergebnis den Span nicht, während Claude ergebnisspeichernde Hintergrundagenten oder
Workflows meldet; erst das endgültige Ergebnis nach dem Leeren beendet ihn. Abbruch, Zeitüberschreitung, Prozessfehler,
Ausgabe-/Analysefehler und andere Durchlauffehler beenden denselben Span mit einem Fehler.

Claude Code meldet die Nutzung pro Assistentennachricht und kann in seinem abschließenden Ergebnis auch die kumulierte
Nutzung melden. Die Antwortabrechnung von OpenClaw verwendet weiterhin die
letzte Assistentennachricht, sodass sich die bestehende Kostensemantik nicht ändert; der
Modellaufruf-Span auf Durchlaufebene verwendet die abschließende kumulierte Nutzung, sofern verfügbar,
einschließlich Cache-Lese- und Cache-Erstellungs-Token.

Für diese CLI-Spans beschreiben Byte- und Zeitfelder die beobachtbare OpenClaw-
CLI-Grenze:

- `openclaw.model_call.request_bytes` ist die UTF-8-Größe des Prompt-Werts,
  der über einmaliges stdin/argv oder die JSONL-Benutzerhülle der verwalteten stdio-Sitzung gesendet wird. Sie
  entspricht nicht der Größe der verborgenen Modellanfrage von Claude Code.
- `openclaw.model_call.response_bytes` ist die UTF-8-Größe der während des Durchlaufs
  beobachteten stdout-Ausgabe der Claude CLI. Sie entspricht nicht der Größe der Anthropic-HTTP-Antwort.
- `openclaw.model_call.time_to_first_byte_ms` ist die Zeit bis zur ersten beobachtbaren
  stdout- oder stderr-Ausgabe der Claude CLI. Sie ist nicht die Netzwerk-TTFB.

Wenn die entsprechenden granularen `captureContent`-Felder aktiviert sind, exportiert der Span
den effektiven Prompt, den OpenClaw an Claude Code sendet, den von OpenClaw angefügten System-
Prompt sowie sichtbaren Assistententext, Schlussfolgerungen und die Identität von Tool-Aufrufen über
`gen_ai.input.messages`, `gen_ai.output.messages` und
`gen_ai.system_instructions`. Tool-Argumente, undurchsichtige Denksignaturen und
Tool-Ergebnisse werden aus der Claude-Assistentenhülle ausgelassen. OpenClaw erhebt keinen
Anspruch auf Zugriff auf den privaten System-Prompt von Claude Code, die verborgene fortgesetzte oder
komprimierte Anfragenutzlast, native interne Tool-Schemas, die rohe Anthropic-HTTP-
Anfrage, interne Wiederholungsversuche, die Upstream-Anfrage-ID oder die tatsächliche Netzwerk-TTFB. Da
Claude Code seine effektiven nativen Tool-Definitionen nicht korrekt offenlegt,
füllen diese Spans `gen_ai.tool.definitions` nicht aus.

Externe Claude-Harness-Tool-Spans bleiben auch dann auf Metadaten beschränkt, wenn die Erfassung von Tool-Inhalten
aktiviert ist. Wie bei jedem Modell-Span verwenden erfasste Claude-CLI-Inhalte
den ausschließlich vertrauenswürdigen Listenern vorbehaltenen Pfad sowie die bestehenden Schwärzungs- und Größen-
grenzen des Exporters; Inhalte bleiben standardmäßig deaktiviert.

## Exportierte Metriken

### Modellnutzung

- `openclaw.tokens` (Zähler, Attribute: `openclaw.token`, `openclaw.channel`, `openclaw.provider`, `openclaw.model`, `openclaw.agent`)
- `openclaw.cost.usd` (Zähler, Attribute: `openclaw.channel`, `openclaw.provider`, `openclaw.model`)
- `openclaw.run.duration_ms` (Histogramm, Attribute: `openclaw.channel`, `openclaw.provider`, `openclaw.model`)
- `openclaw.context.tokens` (Histogramm, Attribute: `openclaw.context`, `openclaw.channel`, `openclaw.provider`, `openclaw.model`)
- `gen_ai.client.token.usage` (Histogramm, Metrik der semantischen GenAI-Konventionen, Attribute: `gen_ai.token.type` = `input`/`output`, `gen_ai.provider.name`, `gen_ai.operation.name`, `gen_ai.request.model`)
- `gen_ai.client.operation.duration` (Histogramm, Sekunden, Metrik der semantischen GenAI-Konventionen für Modellanfragen und synthetische Agent-Durchläufe; Attribute: `gen_ai.provider.name`, `gen_ai.operation.name`, `gen_ai.request.model`, optional `error.type`; Durchlaufbeobachtungen verwenden `gen_ai.operation.name = invoke_agent`)
- `openclaw.model_call.duration_ms` (Histogramm, Attribute: `openclaw.provider`, `openclaw.model`, `openclaw.api`, `openclaw.transport`, `openclaw.model_call.observation_unit`, zusätzlich `openclaw.errorCategory` und `openclaw.failureKind` bei klassifizierten Fehlern)
- `openclaw.model_call.request_bytes` (Histogramm, UTF-8-Byte-Größe der endgültigen Modellanfragenutzlast; bei der Claude Code CLI die oben beschriebene beobachtbare Prompt-Eingabe/-Hülle; kein Inhalt der rohen Nutzlast)
- `openclaw.model_call.response_bytes` (Histogramm, UTF-8-Byte-Größe gestreamter Antwortblock-Nutzlasten; hochfrequente Text-, Denk- und Tool-Aufruf-Deltas zählen nur inkrementelle `delta`-Bytes; bei der Claude Code CLI beobachtete stdout-Bytes; kein roher Antwortinhalt)
- `openclaw.model_call.time_to_first_byte_ms` (Histogramm, verstrichene Zeit vor dem ersten gestreamten Antwortereignis; bei der Claude Code CLI die erste beobachtbare CLI-Ausgabe anstelle der Netzwerk-TTFB)
- `openclaw.model.failover` (Zähler, Attribute: `openclaw.provider`, `openclaw.model`, `openclaw.failover.to_provider`, `openclaw.failover.to_model`, `openclaw.failover.reason`, `openclaw.failover.suspended`, `openclaw.lane`)
- `openclaw.skill.used` (Zähler, Attribute: `openclaw.skill.name`, `openclaw.skill.source`, `openclaw.skill.activation`, optional `openclaw.agent`, optional `openclaw.toolName`)

### Nachrichtenfluss

- `openclaw.webhook.received` (Zähler, Attribute: `openclaw.channel`, `openclaw.webhook`)
- `openclaw.webhook.error` (Zähler, Attribute: `openclaw.channel`, `openclaw.webhook`)
- `openclaw.webhook.duration_ms` (Histogramm, Attribute: `openclaw.channel`, `openclaw.webhook`)
- `openclaw.message.queued` (Zähler, Attribute: `openclaw.channel`, `openclaw.source`)
- `openclaw.message.received` (Zähler, Attribute: `openclaw.channel`, `openclaw.source`)
- `openclaw.message.dispatch.started` (Zähler, Attribute: `openclaw.channel`, `openclaw.source`)
- `openclaw.message.dispatch.completed` (Zähler, Attribute: `openclaw.channel`, `openclaw.outcome`, `openclaw.reason`, `openclaw.source`)
- `openclaw.message.dispatch.duration_ms` (Histogramm, Attribute: `openclaw.channel`, `openclaw.outcome`, `openclaw.reason`, `openclaw.source`)
- `openclaw.message.processed` (Zähler, Attribute: `openclaw.channel`, `openclaw.outcome`)
- `openclaw.message.duration_ms` (Histogramm, Attribute: `openclaw.channel`, `openclaw.outcome`)
- `openclaw.message.delivery.started` (Zähler, Attribute: `openclaw.channel`, `openclaw.delivery.kind`)
- `openclaw.message.delivery.duration_ms` (Histogramm, Attribute: `openclaw.channel`, `openclaw.delivery.kind`, `openclaw.outcome`, `openclaw.errorCategory`)

### Talk

- `openclaw.talk.event` (Zähler, Attribute: `openclaw.talk.event_type`, `openclaw.talk.mode`, `openclaw.talk.transport`, `openclaw.talk.brain`, `openclaw.talk.provider`)
- `openclaw.talk.event.duration_ms` (Histogramm, Attribute: wie bei `openclaw.talk.event`; wird ausgegeben, wenn ein Talk-Ereignis eine Dauer meldet)
- `openclaw.talk.audio.bytes` (Histogramm, Attribute: wie bei `openclaw.talk.event`; wird für Talk-Audioframe-Ereignisse ausgegeben, die eine Bytelänge melden)

### Warteschlangen und Sitzungen

- `openclaw.queue.lane.enqueue` (Zähler, Attribute: `openclaw.lane`)
- `openclaw.queue.lane.dequeue` (Zähler, Attribute: `openclaw.lane`)
- `openclaw.queue.depth` (Histogramm, Attribute: `openclaw.lane` oder `openclaw.channel=heartbeat`)
- `openclaw.queue.wait_ms` (Histogramm, Attribute: `openclaw.lane`)
- `openclaw.session.state` (Zähler, Attribute: `openclaw.state`, `openclaw.reason`)
- `openclaw.session.stuck` (Zähler, Attribute: `openclaw.state`; wird bei wiederherstellbarer veralteter Sitzungsverwaltung ausgegeben)
- `openclaw.session.stuck_age_ms` (Histogramm, Attribute: `openclaw.state`; wird bei wiederherstellbarer veralteter Sitzungsverwaltung ausgegeben)
- `openclaw.session.turn.created` (Zähler, Attribute: `openclaw.agent`, `openclaw.channel`, `openclaw.trigger`)
- `openclaw.session.recovery.requested` (Zähler, Attribute: `openclaw.state`, `openclaw.action`, `openclaw.active_work_kind`, `openclaw.reason`)
- `openclaw.session.recovery.completed` (Zähler, Attribute: `openclaw.state`, `openclaw.action`, `openclaw.status`, `openclaw.active_work_kind`, `openclaw.reason`)
- `openclaw.session.recovery.age_ms` (Histogramm, Attribute: identisch mit dem zugehörigen Wiederherstellungszähler)
- `openclaw.run.attempt` (Zähler, Attribute: `openclaw.attempt`)

### Telemetrie zur Sitzungserreichbarkeit

Eine `processing`-Sitzung nähert sich nicht dem integrierten Erreichbarkeitsschwellenwert, solange OpenClaw Fortschritte bei Antworten, Tools, Status, Blöcken oder der ACP-Laufzeit beobachtet. Tippaktivitäts-Signale zählen nicht als Fortschritt, sodass ein stummes Modell oder Harness weiterhin erkannt werden kann.

OpenClaw klassifiziert Sitzungen anhand der Arbeit, die noch beobachtet werden kann:

- `session.long_running`: Aktive eingebettete Arbeit, Modellaufrufe oder Tool-Aufrufe
  machen weiterhin Fortschritte. Eigene stumme Modellaufrufe werden vor dem integrierten Abbruchschwellenwert ebenfalls als lang laufend gemeldet, sodass langsame oder nicht streamende Modell-Provider nicht wie blockierte Gateway-Sitzungen erscheinen, solange ihr Abbruch beobachtbar ist.
- `session.stalled`: Aktive Arbeit ist vorhanden, aber der aktive Lauf hat in letzter Zeit
  keinen Fortschritt gemeldet. Eigene Modellaufrufe wechseln beim oder nach dem integrierten Abbruchschwellenwert von `session.long_running` zu
  `session.stalled`; veraltete Modell-/Tool-Aktivität ohne Besitzer
  wird nicht als harmlose lang laufende Arbeit behandelt.
  Blockierte eingebettete Läufe werden zunächst nur beobachtet und nach
  Erreichen des Abbruchschwellenwerts ohne Fortschritt abgebrochen und geleert, damit hinter der Lane eingereihte Durchläufe fortgesetzt werden können.
- `session.stuck`: Veraltete Sitzungsverwaltung ohne aktive Arbeit oder eine inaktive
  eingereihte Sitzung mit veralteter Modell-/Tool-Aktivität ohne Besitzer. Dadurch wird die
  betroffene Sitzungslane unmittelbar freigegeben, nachdem die Wiederherstellungsprüfungen bestanden wurden.

Die Wiederherstellung gibt strukturierte Ereignisse vom Typ `session.recovery.requested` und
`session.recovery.completed` aus. Der diagnostische Sitzungsstatus wird nur nach einem
verändernden Wiederherstellungsergebnis (`aborted` oder `released`) und nur dann als inaktiv markiert, wenn
dieselbe Verarbeitungsgeneration noch aktuell ist.

Nur `session.stuck` gibt den Zähler `openclaw.session.stuck`, das
Histogramm `openclaw.session.stuck_age_ms` und den Span `openclaw.session.stuck`
aus. Wiederholte `session.stuck`-Diagnosen werden mit zunehmendem Abstand ausgeführt, solange die Sitzung
unverändert bleibt. Daher sollten Dashboards bei anhaltenden Anstiegen Alarm auslösen und nicht
bei jedem Heartbeat-Takt. Informationen zur Konfigurationsoption und zu den Standardwerten finden Sie in der
[Konfigurationsreferenz](/de/gateway/configuration-reference#diagnostics).

Erreichbarkeitswarnungen geben außerdem Folgendes aus:

- `openclaw.liveness.warning` (Zähler, Attribute: `openclaw.liveness.reason`)
- `openclaw.liveness.event_loop_delay_p99_ms` (Histogramm, Attribute: `openclaw.liveness.reason`)
- `openclaw.liveness.event_loop_delay_max_ms` (Histogramm, Attribute: `openclaw.liveness.reason`)
- `openclaw.liveness.event_loop_utilization` (Histogramm, Attribute: `openclaw.liveness.reason`)
- `openclaw.liveness.cpu_core_ratio` (Histogramm, Attribute: `openclaw.liveness.reason`)

### Harness-Lebenszyklus

- `openclaw.harness.duration_ms` (Histogramm, Attribute: `openclaw.harness.id`, `openclaw.harness.plugin`, `openclaw.outcome`, bei Fehlern `openclaw.harness.phase`)

### Tool-Ausführung und Schleifenerkennung

- `openclaw.tool.execution.duration_ms` (Histogramm, Attribute: `gen_ai.tool.name`, `openclaw.toolName`, `openclaw.tool.source`, `openclaw.tool.owner`, `openclaw.tool.params.kind`, bei Fehlern zusätzlich `openclaw.errorCategory`)
- `openclaw.tool.execution.blocked` (Zähler, Attribute: `gen_ai.tool.name`, `openclaw.toolName`, `openclaw.tool.source`, `openclaw.tool.owner`, `openclaw.tool.params.kind`, `openclaw.deniedReason`)
- `openclaw.tool.loop` (Zähler, Attribute: `openclaw.toolName`, `openclaw.loop.level`, `openclaw.loop.action`, `openclaw.loop.detector`, `openclaw.loop.count`, optional `openclaw.loop.paired_tool`; wird ausgegeben, wenn eine sich wiederholende Tool-Aufrufschleife erkannt wird)

### Exec

- `openclaw.exec.duration_ms` (Histogramm, Attribute: `openclaw.exec.target`, `openclaw.exec.mode`, `openclaw.outcome`, `openclaw.failureKind`)

### Diagnoseinterna (Arbeitsspeicher, Nutzlasten, Exporter-Zustand)

- `openclaw.payload.large` (Zähler, Attribute: `openclaw.payload.surface`, `openclaw.payload.action`, `openclaw.channel`, `openclaw.plugin`, `openclaw.reason`)
- `openclaw.payload.large_bytes` (Histogramm, Attribute: identisch mit `openclaw.payload.large`)
- `openclaw.memory.rss_bytes` / `openclaw.memory.heap_used_bytes` / `openclaw.memory.heap_total_bytes` / `openclaw.memory.external_bytes` / `openclaw.memory.array_buffers_bytes` (Histogramme, keine Attribute; Stichproben des Prozessspeichers)
- `openclaw.memory.pressure` (Zähler, Attribute: `openclaw.memory.level`, `openclaw.memory.reason`)
- `openclaw.diagnostic.async_queue.dropped` (Zähler, Attribute: `openclaw.diagnostic.async_queue.drop_class`; Verwerfungen aufgrund von Rückstau in der internen Diagnosewarteschlange)
- `openclaw.telemetry.exporter.events` (Zähler, Attribute: `openclaw.exporter`, `openclaw.signal`, `openclaw.status`, optional `openclaw.reason`, optional `openclaw.errorCategory`; Selbsttelemetrie zum Lebenszyklus und zu Fehlern des Exporters)

## Exportierte Spans

- `openclaw.model.usage`
  - `openclaw.channel`, `openclaw.provider`, `openclaw.model`
  - `openclaw.tokens.*` (Eingabe/Ausgabe/Cache-Lesen/Cache-Schreiben/Gesamt)
  - `gen_ai.system` standardmäßig oder `gen_ai.provider.name`, wenn die neuesten semantischen GenAI-Konventionen aktiviert sind
  - `gen_ai.request.model`, `gen_ai.operation.name`, `gen_ai.usage.*`
- `openclaw.run`
  - `openclaw.outcome`, `openclaw.channel`, `openclaw.provider`, `openclaw.model`, `openclaw.errorCategory`
- `openclaw.model.call`
  - `gen_ai.system` standardmäßig oder `gen_ai.provider.name`, wenn die neuesten semantischen GenAI-Konventionen aktiviert sind
  - `gen_ai.request.model`, `gen_ai.operation.name`, `openclaw.provider`, `openclaw.model`, `openclaw.api`, `openclaw.transport`, `openclaw.model_call.observation_unit` (`request` oder `turn`)
  - `openclaw.errorCategory`, `error.type` und bei Fehlern optional `openclaw.failureKind`
  - `openclaw.model_call.request_bytes`, `openclaw.model_call.response_bytes`, `openclaw.model_call.time_to_first_byte_ms`
  - `openclaw.model_call.prompt.input_messages_count`, `openclaw.model_call.prompt.input_messages_chars`, `openclaw.model_call.prompt.system_prompt_chars`, `openclaw.model_call.prompt.tool_definitions_count`, `openclaw.model_call.prompt.tool_definitions_chars`, `openclaw.model_call.prompt.total_chars` (nur unbedenkliche Komponentengrößen, kein Prompt-Text)
  - `openclaw.model_call.usage.*` und `gen_ai.usage.*`, wenn das Ergebnis Nutzungsdaten für diese Anfrage oder den aggregierten Durchlauf enthält
  - Span-Ereignis `openclaw.provider.request` mit dem Attribut `openclaw.upstreamRequestIdHash` (begrenzt, hashbasiert), wenn das Ergebnis des vorgelagerten Providers eine Anfrage-ID bereitstellt; Roh-IDs werden niemals exportiert
  - Mit `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` verwenden Anfrage-Spans den neuesten GenAI-Inferenz-Span-Namen `{gen_ai.operation.name} {gen_ai.request.model}`. Durchlauf-Spans verwenden `invoke_agent`, da OpenClaw an der undurchsichtigen CLI-Grenze keinen nativen Agentennamen beansprucht. Beide verwenden die Span-Art `CLIENT` anstelle von `openclaw.model.call`.
- `openclaw.harness.run`
  - `openclaw.harness.id`, `openclaw.harness.plugin`, `openclaw.outcome`, `openclaw.provider`, `openclaw.model`, `openclaw.channel`
  - Bei Abschluss: `openclaw.harness.result_classification`, `openclaw.harness.yield_detected`, `openclaw.harness.items.started`, `openclaw.harness.items.completed`, `openclaw.harness.items.active`
  - Bei einem Fehler: `openclaw.harness.phase`, `openclaw.errorCategory`, optional `openclaw.harness.cleanup_failed`
- `openclaw.tool.execution`
  - `gen_ai.tool.name`, `gen_ai.operation.name` (`execute_tool`), `openclaw.toolName`, `openclaw.tool.source`, optional `gen_ai.tool.call.id`, `openclaw.tool.owner`, `openclaw.tool.params.*`
  - Bei Fehlern optional `openclaw.errorCategory`/`openclaw.errorCode`, bei Ablehnung durch Richtlinie oder Sandbox `openclaw.deniedReason` und `openclaw.outcome=blocked`
- `openclaw.exec`
  - `openclaw.exec.target`, `openclaw.exec.mode`, `openclaw.outcome`, `openclaw.failureKind`, `openclaw.exec.command_length`, `openclaw.exec.exit_code`, `openclaw.exec.exit_signal`, `openclaw.exec.timed_out`
- `openclaw.webhook.processed`
  - `openclaw.channel`, `openclaw.webhook`
- `openclaw.webhook.error`
  - `openclaw.channel`, `openclaw.webhook`, `openclaw.error`
- `openclaw.message.processed`
  - `openclaw.channel`, `openclaw.outcome`, `openclaw.reason`
- `openclaw.message.delivery`
  - `openclaw.channel`, `openclaw.delivery.kind`, `openclaw.outcome`, `openclaw.errorCategory`, `openclaw.delivery.result_count`
- `openclaw.session.stuck`
  - `openclaw.state`, `openclaw.ageMs`, `openclaw.queueDepth`
- `openclaw.context.assembled`
  - `openclaw.prompt.size`, `openclaw.history.size`, `openclaw.context.tokens`, `openclaw.errorCategory` (keine Inhalte von Prompt, Verlauf, Antwort oder Sitzungsschlüssel)
- `openclaw.tool.loop`
  - `openclaw.toolName`, `openclaw.loop.level`, `openclaw.loop.action`, `openclaw.loop.detector`, `openclaw.loop.count`, optional `openclaw.loop.paired_tool` (keine Schleifennachrichten, Parameter oder Tool-Ausgaben)
- `openclaw.memory.pressure`
  - `openclaw.memory.level`, `openclaw.memory.reason`, `openclaw.memory.rss_bytes`, `openclaw.memory.heap_used_bytes`, `openclaw.memory.heap_total_bytes`, `openclaw.memory.external_bytes`, `openclaw.memory.array_buffers_bytes`, optional `openclaw.memory.threshold_bytes`/`openclaw.memory.rss_growth_bytes`/`openclaw.memory.window_ms`

Wenn die Inhaltserfassung ausdrücklich aktiviert ist, können Modell- und Tool-Spans außerdem
begrenzte, geschwärzte `openclaw.content.*`-Attribute für die spezifischen
Inhaltsklassen enthalten, die aktiviert wurden.

## Katalog der Diagnoseereignisse

Die nachstehenden Ereignisse bilden die Grundlage für die oben aufgeführten Metriken und Spans oder stehen für direkte
Plugin-Abonnements zur Verfügung. `run.progress` und `run.execution_phase` sind reine
Lebenszyklussignale für direkte Abonnements; das diagnostics-otel-Plugin exportiert sie nicht als
eigenständige OTLP-Signale. Ereignisarten und `run.execution_phase.phase`-Werte sind
additiv. TypeScript-Consumer sollten Standardzweige beibehalten, anstatt davon auszugehen,
dass eine der beiden Unions dauerhaft vollständig ist.

**Modellnutzung**

- `model.usage` – Token, Kosten, Dauer, Kontext, Provider/Modell/Kanal,
  Sitzungs-IDs. `usage` dient der Provider-/Durchlaufabrechnung für Kosten und Telemetrie;
  `context.used` ist die aktuelle Prompt-/Kontext-Momentaufnahme und kann niedriger als der
  Provider-Wert `usage.total` sein, wenn zwischengespeicherte Eingaben oder Tool-Schleifenaufrufe beteiligt sind.

**Nachrichtenfluss**

- `webhook.received` / `webhook.processed` / `webhook.error`
- `message.queued` / `message.processed`
- `message.delivery.started` / `message.delivery.completed` / `message.delivery.error`

**Warteschlange und Sitzung**

- `queue.lane.enqueue` / `queue.lane.dequeue`
- `session.state` / `session.long_running` / `session.stalled` / `session.stuck`
- `run.attempt` / `run.progress`
- `run.execution_phase` (öffentliche, sitzungskorrelierte Startmeilensteine des eingebetteten Runners)
- `diagnostic.heartbeat` (aggregierte Zähler: Webhooks/Warteschlange/Sitzung)

**Harness-Lebenszyklus**

- `harness.run.started` / `harness.run.completed` / `harness.run.error` –
  Lebenszyklus pro Lauf für das Agenten-Harness. Enthält `harnessId`, optional
  `pluginId`, Provider/Modell/Kanal und Lauf-ID. Bei Abschluss werden
  `durationMs`, `outcome`, optional `resultClassification`, `yieldDetected`
  sowie `itemLifecycle`-Zähler hinzugefügt. Bei Fehlern werden `phase`
  (`prepare`/`start`/`send`/`resolve`/`cleanup`), `errorCategory` und
  optional `cleanupFailed` hinzugefügt.

**Exec**

- `exec.process.completed` – Terminalergebnis, Dauer, Ziel, Modus, Exit-
  Code und Fehlerart. Befehlstext und Arbeitsverzeichnisse sind nicht
  enthalten.
- `exec.approval.followup_suppressed` – Veraltete Genehmigungsnachverfolgung verworfen
  nach einer erneuten Sitzungsbindung. Enthält `approvalId`, `reason`
  (`session_rebound`), `phase` (`direct_delivery` oder `gateway_preflight`)
  und den Zeitstempel des Dispatchers. Sitzungsschlüssel, Routen und Befehlstext sind
  nicht enthalten.

## Ohne Exporter

Halten Sie Diagnoseereignisse für Plugins oder benutzerdefinierte Senken verfügbar, ohne
`diagnostics-otel` auszuführen:

```json5
{
  diagnostics: { enabled: true },
}
```

Verwenden Sie für gezielte Debug-Ausgaben Diagnose-Flags, ohne `logging.level`
anzuheben. Bei Flags wird die Groß-/Kleinschreibung nicht berücksichtigt und Platzhalter werden unterstützt (`telegram.*` oder
`*`):

```json5
{
  diagnostics: { flags: ["telegram.http"] },
}
```

Oder als einmalige Umgebungsüberschreibung:

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload openclaw gateway
```

Die Flag-Ausgabe wird in die Standardprotokolldatei (`logging.file`) geschrieben und weiterhin
durch `logging.redactSensitive` bereinigt. Vollständige Anleitung:
[Diagnose-Flags](/de/diagnostics/flags).

## Deaktivieren

```json5
{
  diagnostics: { otel: { enabled: false } },
}
```

Oder lassen Sie `diagnostics-otel` aus `plugins.allow` weg, oder führen Sie
`openclaw plugins disable diagnostics-otel` aus.

## Verwandte Themen

- [Protokollierung](/de/logging) – Dateiprotokolle, Konsolenausgabe, CLI-Tailing und die Registerkarte „Protokolle“ der Control UI
- [Interna der Gateway-Protokollierung](/de/gateway/logging) – WS-Protokollstile, Subsystempräfixe und Konsolenerfassung
- [Diagnose-Flags](/de/diagnostics/flags) – gezielte Debug-Protokoll-Flags
- [Diagnoseexport](/de/gateway/diagnostics) – Support-Bundle-Tool für Betreiber (getrennt vom OTEL-Export)
- [Konfigurationsreferenz](/de/gateway/configuration-reference#diagnostics) – vollständige Feldreferenz für `diagnostics.*`
