---
read_when:
    - Sie möchten, dass Prometheus, Grafana, VictoriaMetrics oder ein anderer Scraper Metriken des OpenClaw Gateway erfasst.
    - Sie benötigen die Prometheus-Metriknamen und die Label-Richtlinie für Dashboards oder Warnmeldungen
    - Sie möchten Metriken erfassen, ohne einen OpenTelemetry-Collector auszuführen
sidebarTitle: Prometheus
summary: OpenClaw-Diagnosedaten über das Plugin diagnostics-prometheus als Prometheus-Textmetriken bereitstellen
title: Prometheus-Metriken
x-i18n:
    generated_at: "2026-07-26T18:28:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d04a46bdb401df3cdd2571b973f2a60f264862cf74da02c5a9cfa1de6ea9ffe
    source_path: gateway/prometheus.md
    workflow: 16
---

OpenClaw kann Diagnosemetriken über das offizielle
`diagnostics-prometheus`-Plugin bereitstellen. Es erfasst vertrauenswürdige Diagnosedaten sowie
intern markierte, dem Dispatcher zugeordnete Diagnoseereignisse (Signale zu Warteschlangen, Arbeitsspeicher und
Sitzungswiederherstellung) und stellt einen Prometheus-Textendpunkt unter folgender Adresse bereit:

```text
GET /api/diagnostics/prometheus
```

Der Inhaltstyp ist `text/plain; version=0.0.4; charset=utf-8`, das standardmäßige
Prometheus-Expositionsformat.

<Warning>
Die Route verwendet die Gateway-Authentifizierung (Operator-Bereich, Oberfläche für vertrauenswürdige Operatoren). Stellen Sie sie nicht als öffentlichen, nicht authentifizierten `/metrics`-Endpunkt bereit. Rufen Sie sie über denselben Authentifizierungspfad ab, den Sie für andere Operator-APIs verwenden.
</Warning>

Informationen zu Traces, Protokollen, OTLP-Push und semantischen OpenTelemetry-GenAI-Attributen finden Sie unter [OpenTelemetry-Export](/de/gateway/opentelemetry).

## Schnellstart

<Steps>
  <Step title="Plugin installieren">
    ```bash
    openclaw plugins install clawhub:@openclaw/diagnostics-prometheus
    ```
  </Step>
  <Step title="Plugin aktivieren">
    <Tabs>
      <Tab title="Konfiguration">
        ```json5
        {
          plugins: {
            allow: ["diagnostics-prometheus"],
            entries: {
              "diagnostics-prometheus": { enabled: true },
            },
          },
          diagnostics: {
            enabled: true,
          },
        }
        ```
      </Tab>
      <Tab title="CLI">
        ```bash
        openclaw plugins enable diagnostics-prometheus
        ```
      </Tab>
    </Tabs>
  </Step>
  <Step title="Gateway neu starten">
    Die HTTP-Route wird beim Start des Plugins registriert. Laden Sie das Gateway daher nach der Aktivierung neu.
  </Step>
  <Step title="Geschützte Route abrufen">
    Senden Sie dieselbe Gateway-Authentifizierung, die Ihre Operator-Clients verwenden:

    ```bash
    curl -H "Authorization: Bearer $OPENCLAW_GATEWAY_TOKEN" \
      http://127.0.0.1:18789/api/diagnostics/prometheus
    ```

  </Step>
  <Step title="Prometheus anbinden">
    ```yaml
    # prometheus.yml
    scrape_configs:
      - job_name: openclaw
        scrape_interval: 30s
        metrics_path: /api/diagnostics/prometheus
        authorization:
          credentials_file: /etc/prometheus/openclaw-gateway-token
        static_configs:
          - targets: ["openclaw-gateway:18789"]
    ```
  </Step>
</Steps>

<Note>
`diagnostics.enabled` ist standardmäßig auf `true` gesetzt; setzen Sie es nur in streng abgeschotteten Umgebungen auf `false`. Wenn es `false` ist, registriert das Plugin weiterhin die HTTP-Route, es werden jedoch keine Diagnoseereignisse an den Exporter übermittelt, sodass die Antwort leer ist.
</Note>

## Exportierte Metriken

| Metrik                                           | Typ       | Labels                                                                                    |
| ------------------------------------------------ | --------- | ----------------------------------------------------------------------------------------- |
| `openclaw_run_completed_total`                   | Zähler    | `channel`, `model`, `outcome`, `provider`, `trigger`                                      |
| `openclaw_run_duration_seconds`                  | Histogramm | `channel`, `model`, `outcome`, `provider`, `trigger`                                      |
| `openclaw_model_call_total`                      | Zähler    | `api`, `error_category`, `model`, `observation_unit`, `outcome`, `provider`, `transport`  |
| `openclaw_model_call_duration_seconds`           | Histogramm | `api`, `error_category`, `model`, `observation_unit`, `outcome`, `provider`, `transport`  |
| `openclaw_model_failover_total`                  | Zähler    | `from_model`, `from_provider`, `lane`, `reason`, `suspended`, `to_model`, `to_provider`   |
| `openclaw_model_tokens_total`                    | Zähler    | `agent`, `channel`, `model`, `provider`, `token_type`                                     |
| `openclaw_gen_ai_client_token_usage`             | Histogramm | `model`, `provider`, `token_type`                                                         |
| `openclaw_model_cost_usd_total`                  | Zähler    | `agent`, `channel`, `model`, `provider`                                                   |
| `openclaw_model_usage_duration_seconds`          | Histogramm | `agent`, `channel`, `model`, `provider`                                                   |
| `openclaw_skill_used_total`                      | Zähler    | `activation`, `agent`, `skill`, `source`                                                  |
| `openclaw_tool_execution_total`                  | Zähler    | `error_category`, `outcome`, `params_kind`, `tool`, `tool_owner`, `tool_source`           |
| `openclaw_tool_execution_duration_seconds`       | Histogramm | `error_category`, `outcome`, `params_kind`, `tool`, `tool_owner`, `tool_source`           |
| `openclaw_tool_execution_blocked_total`          | Zähler    | `denied_reason`, `params_kind`, `tool`, `tool_owner`, `tool_source`                       |
| `openclaw_harness_run_total`                     | Zähler    | `channel`, `error_category`, `harness`, `model`, `outcome`, `phase`, `plugin`, `provider` |
| `openclaw_harness_run_duration_seconds`          | Histogramm | `channel`, `error_category`, `harness`, `model`, `outcome`, `phase`, `plugin`, `provider` |
| `openclaw_webhook_received_total`                | Zähler    | `channel`, `webhook`                                                                      |
| `openclaw_webhook_error_total`                   | Zähler    | `channel`, `webhook`                                                                      |
| `openclaw_webhook_duration_seconds`              | Histogramm | `channel`, `webhook`                                                                      |
| `openclaw_message_received_total`                | Zähler    | `channel`, `source`                                                                       |
| `openclaw_message_dispatch_started_total`        | Zähler    | `channel`, `source`                                                                       |
| `openclaw_message_dispatch_completed_total`      | Zähler    | `channel`, `outcome`, `reason`, `source`                                                  |
| `openclaw_message_dispatch_duration_seconds`     | Histogramm | `channel`, `outcome`, `reason`, `source`                                                  |
| `openclaw_message_processed_total`               | Zähler    | `channel`, `outcome`, `reason`                                                            |
| `openclaw_message_processed_duration_seconds`    | Histogramm | `channel`, `outcome`, `reason`                                                            |
| `openclaw_message_delivery_started_total`        | Zähler    | `channel`, `delivery_kind`                                                                |
| `openclaw_message_delivery_total`                | Zähler    | `channel`, `delivery_kind`, `error_category`, `outcome`                                   |
| `openclaw_message_delivery_duration_seconds`     | Histogramm | `channel`, `delivery_kind`, `error_category`, `outcome`                                   |
| `openclaw_talk_event_total`                      | Zähler    | `brain`, `event_type`, `mode`, `provider`, `transport`                                    |
| `openclaw_talk_event_duration_seconds`           | Histogramm | `brain`, `event_type`, `mode`, `provider`, `transport`                                    |
| `openclaw_talk_audio_bytes`                      | Histogramm | `brain`, `event_type`, `mode`, `provider`, `transport`                                    |
| `openclaw_queue_lane_size`                       | Messwert     | `lane`                                                                                    |
| `openclaw_queue_lane_wait_seconds`               | Histogramm | `lane`                                                                                    |
| `openclaw_session_state_total`                   | Zähler    | `reason`, `state`                                                                         |
| `openclaw_session_queue_depth`                   | Messwert     | `state`                                                                                   |
| `openclaw_session_turn_created_total`            | Zähler    | `agent`, `channel`, `trigger`                                                             |
| `openclaw_session_stuck_total`                   | Zähler    | `reason`, `state`                                                                         |
| `openclaw_session_stuck_age_seconds`             | Histogramm | `reason`, `state`                                                                         |
| `openclaw_session_recovery_total`                | Zähler    | `action`, `active_work_kind`, `state`, `status`                                           |
| `openclaw_session_recovery_age_seconds`          | Histogramm | `action`, `active_work_kind`, `state`, `status`                                           |
| `openclaw_liveness_warning_total`                | Zähler    | `reason`                                                                                  |
| `openclaw_liveness_sessions`                     | Messwert     | `state`                                                                                   |
| `openclaw_liveness_event_loop_delay_p99_seconds` | Histogramm | `reason`                                                                                  |
| `openclaw_liveness_event_loop_delay_max_seconds` | Histogramm | `reason`                                                                                  |
| `openclaw_liveness_event_loop_utilization_ratio` | Histogramm | `reason`                                                                                  |
| `openclaw_liveness_cpu_core_ratio`               | Histogramm | `reason`                                                                                  |
| `openclaw_payload_large_total`                   | Zähler    | `action`, `channel`, `plugin`, `reason`, `surface`                                        |
| `openclaw_payload_large_bytes`                   | Histogramm | `action`, `channel`, `plugin`, `reason`, `surface`                                        |
| `openclaw_memory_bytes`                          | Messwert     | `kind`                                                                                    |
| `openclaw_memory_rss_bytes`                      | Histogramm | keine                                                                                      |
| `openclaw_memory_pressure_total`                 | Zähler    | `level`, `reason`                                                                         |
| `openclaw_telemetry_exporter_total`              | Zähler    | `exporter`, `reason`, `signal`, `status`                                                  |
| `openclaw_prometheus_series_dropped_total`       | Zähler    | keine                                                                                      |
| `openclaw_diagnostic_async_queue_dropped_total`  | Zähler    | `drop_class`                                                                              |
| `openclaw_diagnostic_async_queue_length`         | Messwert     | keine                                                                                      |

Für Metriken zu Modellaufrufen misst `observation_unit="request"` eine beobachtbare
Provider-Anfrage. `observation_unit="turn"` misst einen synthetischen Agentendurchlauf von Claude Code
oder der Codex CLI, der mehrere verborgene Provider-Anfragen enthalten kann.
Halten Sie diese Zeitreihen beim Vergleich der Latenz getrennt.

## Richtlinie für Labels

<AccordionGroup>
  <Accordion title="Begrenzte Labels mit niedriger Kardinalität">
    Prometheus-Labels bleiben begrenzt und weisen eine niedrige Kardinalität auf. Der Exporter gibt keine unverarbeiteten Diagnosekennungen wie `runId`, `sessionKey`, `sessionId`, `callId`, `toolCallId`, Nachrichten-IDs, Chat-IDs oder IDs von Provider-Anfragen aus.

    Labelwerte werden unkenntlich gemacht und müssen der OpenClaw-Zeichenrichtlinie für niedrige Kardinalität entsprechen. Werte, die diese Richtlinie nicht erfüllen, werden je nach Metrik durch `unknown`, `other` oder `none` ersetzt. Labels, die wie bereichsbezogene Schlüssel für Agentensitzungen aussehen, werden ebenfalls durch `unknown` ersetzt.

  </Accordion>
  <Accordion title="Zeitreihenlimit und Erfassung von Überschreitungen">
    Der Exporter begrenzt die Anzahl der im Arbeitsspeicher vorgehaltenen Zeitreihen über Zähler, Messwerte und Histogramme hinweg auf insgesamt **2048** Zeitreihen. Neue Zeitreihen, die dieses Limit überschreiten, werden verworfen, und `openclaw_prometheus_series_dropped_total` wird jedes Mal um eins erhöht.

    Überwachen Sie diesen Zähler als eindeutiges Signal dafür, dass ein vorgelagertes Attribut Werte mit hoher Kardinalität durchlässt. Der Exporter hebt das Limit niemals automatisch auf. Wenn der Zähler steigt, beheben Sie die Ursache, statt das Limit zu deaktivieren.

  </Accordion>
  <Accordion title="Was niemals in der Prometheus-Ausgabe erscheint">
    - Prompttext, Antworttext, Tool-Eingaben, Tool-Ausgaben, System-Prompts
    - Gesprächstranskripte, Audiodaten, Anruf-IDs, Raum-IDs, Übergabe-Token, Durchlauf-IDs und unverarbeitete Sitzungs-IDs
    - unverarbeitete IDs von Provider-Anfragen (gegebenenfalls nur begrenzte Hashwerte in Spans – niemals in Metriken)
    - Sitzungsschlüssel und Sitzungs-IDs
    - Hostnamen, Dateipfade, geheime Werte

  </Accordion>
</AccordionGroup>

## PromQL-Rezepte

```promql
# Token pro Minute, nach Provider aufgeschlüsselt
sum by (provider) (rate(openclaw_model_tokens_total[1m]))

# Ausgaben (USD) während der letzten Stunde, nach Modell
sum by (model) (increase(openclaw_model_cost_usd_total[1h]))

# 95. Perzentil der Dauer von Modellläufen
histogram_quantile(
  0.95,
  sum by (le, provider, model)
    (rate(openclaw_run_duration_seconds_bucket[5m]))
)

# SLO für die Wartezeit in der Warteschlange (95. Perzentil unter 2s)
histogram_quantile(
  0.95,
  sum by (le, lane) (rate(openclaw_queue_lane_wait_seconds_bucket[5m]))
) < 2

# Skill-Nutzung, nach begrenzter Quelle aufgeschlüsselt
sum by (skill, source) (increase(openclaw_skill_used_total[24h]))

# Verworfene Prometheus-Zeitreihen (Kardinalitätsalarm)
increase(openclaw_prometheus_series_dropped_total[15m]) > 0
```

<Tip>
Bevorzugen Sie `gen_ai_client_token_usage` für Provider-übergreifende Dashboards: Es folgt den semantischen OpenTelemetry-GenAI-Konventionen und stimmt mit Metriken von GenAI-Diensten überein, die nicht zu OpenClaw gehören.
</Tip>

## Wahl zwischen Prometheus- und OpenTelemetry-Export

OpenClaw unterstützt beide Schnittstellen unabhängig voneinander. Sie können eine, beide oder keine davon verwenden.

<Tabs>
  <Tab title="diagnostics-prometheus">
    - **Pull**-Modell: Prometheus ruft `/api/diagnostics/prometheus` ab.
    - Kein externer Collector erforderlich.
    - Authentifizierung über die normale Gateway-Authentifizierung.
    - Die Schnittstelle umfasst nur Metriken (keine Traces oder Protokolle).
    - Am besten für Stacks geeignet, die bereits auf Prometheus + Grafana standardisiert sind.

  </Tab>
  <Tab title="diagnostics-otel">
    - **Push**-Modell: OpenClaw sendet OTLP/HTTP an einen Collector oder ein OTLP-kompatibles Backend.
    - Die Schnittstelle umfasst Metriken, Traces und Protokolle.
    - Stellt über einen OpenTelemetry Collector (`prometheus`- oder `prometheusremotewrite`-Exporter) eine Verbindung zu Prometheus her, wenn Sie beides benötigen.
    - Den vollständigen Katalog finden Sie unter [OpenTelemetry-Export](/de/gateway/opentelemetry).

  </Tab>
</Tabs>

## Fehlerbehebung

<AccordionGroup>
  <Accordion title="Leerer Antworttext">
    - Prüfen Sie, dass `diagnostics.enabled` in der Konfiguration nicht auf `false` gesetzt ist (der Standardwert ist `true`).
    - Bestätigen Sie mit `openclaw plugins list --enabled`, dass das Plugin aktiviert und geladen ist.
    - Erzeugen Sie etwas Datenverkehr. Zähler und Histogramme geben erst nach mindestens einem Ereignis Zeilen aus.

  </Accordion>
  <Accordion title="401 / nicht autorisiert">
    Der Endpunkt erfordert den Gateway-Operator-Berechtigungsbereich (`auth: "gateway"` mit `gatewayRuntimeScopeSurface: "trusted-operator"`). Verwenden Sie dasselbe Token oder Passwort, das Prometheus für alle anderen Gateway-Operator-Routen verwendet. Es gibt keinen öffentlichen, nicht authentifizierten Modus.
  </Accordion>
  <Accordion title="`openclaw_prometheus_series_dropped_total` steigt">
    Ein neues Attribut überschreitet das Limit von **2048** Zeitreihen. Untersuchen Sie die aktuellen Metriken auf ein Label mit unerwartet hoher Kardinalität und beheben Sie die Ursache. Der Exporter verwirft absichtlich neue Zeitreihen, statt Labels stillschweigend umzuschreiben.
  </Accordion>
  <Accordion title="Prometheus zeigt nach einem Neustart veraltete Zeitreihen an">
    Das Plugin speichert seinen Zustand ausschließlich im Arbeitsspeicher. Nach einem Neustart des Gateways werden Zähler auf null zurückgesetzt, und Messwerte beginnen erneut mit ihrem nächsten gemeldeten Wert. Verwenden Sie in PromQL `rate()` und `increase()`, um Zurücksetzungen korrekt zu verarbeiten.
  </Accordion>
</AccordionGroup>

## Verwandte Themen

- [Diagnoseexport](/de/gateway/diagnostics) — lokale Diagnose-ZIP-Datei für Supportpakete
- [Systemzustand und Bereitschaft](/de/gateway/health) — `/healthz`- und `/readyz`-Prüfungen
- [Protokollierung](/de/logging) — dateibasierte Protokollierung
- [OpenTelemetry-Export](/de/gateway/opentelemetry) — OTLP-Push für Traces, Metriken und Protokolle
