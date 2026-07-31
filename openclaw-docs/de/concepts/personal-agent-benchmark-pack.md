---
read_when:
    - Zuverlässigkeitsprüfungen für lokale persönliche Agenten ausführen
    - Erweitern des repository-gestützten QA-Szenariokatalogs
    - Überprüfung von Erinnerungen, Antworten, Memory, Schwärzung, sicherer Tool-Nachverfolgung, Aufgabenstatus, sicher teilbaren Diagnosedaten, beleggestützten Abschlussbehauptungen und Fehlerbehebung
summary: Lokale qa-channel-Szenarien für Workflow-Prüfungen datenschutzfreundlicher persönlicher Assistenten.
title: Benchmark-Paket für persönliche Agenten
x-i18n:
    generated_at: "2026-07-26T17:45:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 35da45e4b22b1044a777fa8d6bce87f9ace377950dd0af3f2419b40cfe4d9be6
    source_path: concepts/personal-agent-benchmark-pack.md
    workflow: 16
---

Das Personal Agent Benchmark Pack ist ein kleines, repositorygestütztes QA-Szenarienpaket für
lokale persönliche Assistenz-Workflows. Es ist kein generischer Modell-Benchmark und
benötigt keinen neuen Runner: Es verwendet den privaten QA-Stack ([QA-Übersicht](/de/concepts/qa-e2e-automation)),
den synthetischen [QA-Kanal](/de/channels/qa-channel) und den vorhandenen
`qa/scenarios`-YAML-Katalog.

## Szenarien

Zehn Szenarien, definiert in `qa/scenarios/personal/*.yaml`:

| Szenario-ID                                | Prüfungen                                                                                       |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| `personal-reminder-roundtrip`              | Simulierte persönliche Erinnerungen über lokale Cron-Zustellung                                |
| `personal-channel-thread-reply`            | Simuliertes Routing von Direktnachrichten und Thread-Antworten über `qa-channel`         |
| `personal-memory-preference-recall`        | Simulierter Abruf von Präferenzen aus den temporären Speicherdateien des QA-Arbeitsbereichs    |
| `personal-redaction-no-secret-leak`        | Prüfungen, dass simulierte Geheimnisse nicht ausgegeben werden                                 |
| `personal-tool-safety-followthrough`       | Sichere, lesegestützte Tool-Ausführung nach einer kurzen genehmigungsähnlichen Interaktion     |
| `personal-approval-denial-stop`            | Abbruchverhalten bei verweigerter Genehmigung einer sensiblen lokalen Leseanforderung          |
| `personal-task-followthrough-status`       | Beleggestützte Aufgabenstatusmeldung, die ausstehend, blockiert und erledigt getrennt hält      |
| `personal-share-safe-diagnostics-artifact` | Sicher teilbare Diagnoseartefakte, die nützliche Statusinformationen beibehalten und persönliche Rohinhalte auslassen |
| `personal-no-fake-progress`                | Beleggestützte Abschlussangaben, die vor dem Vorliegen lokaler Belege keinen Fortschritt vortäuschen |
| `personal-failure-recovery`                | Fehlerbehebung, die den Teilstatus meldet und Wiederholungsgrenzen klar hält                    |

Die maschinenlesbaren Metadaten des Pakets (ID-Liste, Titel, Beschreibung) befinden sich in
`extensions/qa-lab/src/scenario-packs.ts` als `QA_PERSONAL_AGENT_SCENARIO_IDS`.
Führen Sie das Paket mit `--pack personal-agent` aus:

```bash
OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa suite \
  --provider-mode mock-openai \
  --pack personal-agent \
  --concurrency 1
```

`--pack` wirkt additiv mit wiederholten `--scenario`-Flags. Explizite Szenarien werden
zuerst ausgeführt, danach werden die Paketszenarien in der Reihenfolge von `QA_PERSONAL_AGENT_SCENARIO_IDS`
ausgeführt, wobei Duplikate entfernt werden.

Das Paket ist für `qa-channel` mit `mock-openai` oder eine andere lokale QA-Provider-
Lane vorgesehen. Richten Sie es nicht auf Live-Chat-Dienste oder echte persönliche Konten.

## Datenschutzmodell

Die Szenarien verwenden ausschließlich simulierte Benutzer, simulierte Präferenzen, simulierte Geheimnisse und den
temporären QA-Gateway-Arbeitsbereich, den die Suite erstellt. Sie dürfen keine echten
OpenClaw-Benutzerspeicher, Sitzungen, Anmeldedaten, Launch Agents, globalen
Konfigurationen oder Live-Gateway-Zustände lesen oder schreiben.

Artefakte verbleiben im vorhandenen Artefaktverzeichnis der QA-Suite und werden
wie Testausgaben behandelt. Schwärzungsprüfungen verwenden simulierte Marker, sodass Fehler
sicher untersucht und in Issues gemeldet werden können.

## Paket erweitern

Fügen Sie neue `.yaml`-Fälle unter `qa/scenarios/personal/` hinzu und ergänzen Sie anschließend die Szenario-ID
in `QA_PERSONAL_AGENT_SCENARIO_IDS`. Halten Sie jeden Fall klein, lokal und deterministisch
in `mock-openai` und konzentrieren Sie ihn auf ein Verhalten persönlicher Assistenten.

Gute Kandidaten für nachfolgende Erweiterungen: Prüfungen des Exports geschwärzter Trajektorien,
Prüfungen rein lokaler Plugin-Workflows.

Fügen Sie keinen neuen Runner, kein Plugin, keine Abhängigkeit, keinen Live-Transport und keinen Modell-Judge
hinzu, solange der Szenarienkatalog nicht genügend stabile Fälle enthält, die eine solche Oberfläche rechtfertigen.
