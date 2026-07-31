---
doc-schema-version: 1
read_when:
    - Full Release Validation ausführen oder erneut ausführen
    - Vergleich der Validierungsprofile für stabile und vollständige Releases
    - Fehlerbehebung bei Fehlern in der Release-Validierungsphase
summary: Phasen der vollständigen Release-Validierung, untergeordnete Workflows, Release-Profile, Handles für erneute Ausführungen und Nachweise
title: Vollständige Release-Validierung
x-i18n:
    generated_at: "2026-07-26T18:08:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf165d5515f4b9bb11d239382649d332d20bb8a32bd4492ae99092fb5ee2216
    source_path: reference/full-release-validation.md
    workflow: 16
---

`Full Release Validation` ist der übergreifende Rahmen für die Produktvalidierung von Releases. Die meisten Arbeiten
finden in untergeordneten Workflows statt, sodass eine fehlgeschlagene Box erneut ausgeführt werden kann, ohne den
gesamten Release neu zu starten. Führen Sie die Release-Vorbereitung aus, bevor Sie den Code-SHA fixieren; sie
aktualisiert die Gebietsschema-Ausgabe der Control UI, wenn der Hintergrund-Bot sie noch nicht
übernommen hat, und erzwingt anschließend dieselbe strikte Prüfung auf null Fallbacks, die auch von der Release-CI verwendet wird.

Fixieren Sie den produktseitig vollständigen Commit vor dem Changelog als **Code-SHA** und führen Sie dann Folgendes aus:

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

`provider` akzeptiert außerdem `anthropic` oder `minimax` für das Betriebssystem-übergreifende Onboarding und den
End-to-End-Agentendurchlauf. Das Hilfsprogramm leitet das Profil `beta` aus Alpha-/Beta-
Paketversionen ab und verwendet andernfalls `stable`. Übergeben Sie alternative Workflow-Eingaben mit
`-f key=value`; verwenden Sie `-f release_profile=full` nur für die umfassende Advisory-Prüfung.

Das Hilfsprogramm erstellt eine temporäre `release-ci/*`-Referenz, die auf genau einen vertrauenswürdigen
`origin/main`-Workflow-SHA festgelegt ist, übergibt den Ziel-SHA ausschließlich als Kandidaten-`ref`
und löscht die temporäre Referenz nach der Validierung. Jeder ausgelöste untergeordnete Workflow muss
denselben Workflow-SHA melden. Übergeben Sie
`-f reuse_evidence=false`, um eine neue Ausführung zu erzwingen, oder
`--workflow-sha <trusted-main-sha>`, um einen älteren Workflow-Commit auszuwählen, der weiterhin
vom aktuellen `origin/main` aus erreichbar ist. Der Workflow selbst erstellt oder aktualisiert niemals
Repository-Referenzen.

## Ausnahme für Extended-Stable

Die Veröffentlichung von Extended-Stable erfordert eine Ausführung, bei der sowohl Workflow als auch Ziel dem
kanonischen Branch entsprechen:

```bash
gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

Verwenden Sie weder `pnpm ci:full-release` noch `release-ci/*`. Die Veröffentlichung bindet den
Branch der Ausführung, den Head-/Ziel-SHA, das Manifest `workflowRef`, die ID und den Versuch an den kanonischen
Branch und den Release-Commit.

Portieren Sie Produktfehler zurück; nehmen Sie für Werkzeuge mit fixiertem Ziel die kleinste verhaltensbewahrende Korrektur vor;
wiederholen Sie Provider-, Genehmigungs- oder Runner-Fehler ohne
Quellcodeänderung. Jede Branch-Änderung erfordert eine vollständig neue Ausführung. Lassen Sie erforderliches
Paket-, Installer-, Update-, Kanal- oder Live-Verhalten nicht aus, nur weil das Ziel alt ist.

Wenn der Code-SHA für einen regulären Release grün ist, generieren und committen Sie ausschließlich
`CHANGELOG.md`. Dieser neue Commit ist der **Release-SHA**. Führen Sie dasselbe Hilfsprogramm für
den Release-SHA aus. Produktnachweise werden nur wiederverwendet, wenn GitHub nachweist, dass der Release-
SHA vom Code-SHA abstammt und die vollständige Menge geänderter Pfade exakt
`CHANGELOG.md` entspricht; npm-Preflight und Paket-/Installationsakzeptanz werden dennoch auf dem
Release-SHA ausgeführt.

`release_profile=stable` und `release_profile=full` führen stets den umfassenden
Live-/Docker-Dauertest aus. Übergeben Sie `run_release_soak=true`, um dieselben Dauertest-Lanes
mit dem Profil `beta` einzubeziehen. Die stabile Veröffentlichung lehnt ein Validierungsmanifest
ohne diesen Dauertest und blockierende Nachweise zur Produktleistung ab.

Package Acceptance erstellt den Kandidaten-Tarball normalerweise aus dem aufgelösten
`ref`, einschließlich vollständiger SHA-Ausführungen, die mit `pnpm ci:full-release` ausgelöst wurden. Übergeben Sie nach einer
Beta-Veröffentlichung `release_package_spec=openclaw@YYYY.M.PATCH-beta.N`, um
das veröffentlichte npm-Paket für Release-Prüfungen, Package Acceptance, Betriebssystem-übergreifende Prüfungen,
den Docker-Release-Pfad und Paket-Telegram wiederzuverwenden. Verwenden Sie `package_acceptance_package_spec`
nur, wenn Package Acceptance absichtlich ein anderes Paket nachweisen soll.
Die Live-Paket-Lane des Codex-Plugins folgt demselben Zustand: Veröffentlichte
`release_package_spec`-Werte leiten `codex_plugin_spec=npm:@openclaw/codex@<version>` ab;
SHA-/Artefakt-Ausführungen packen `extensions/codex` aus der ausgewählten Referenz; und Operatoren
können `codex_plugin_spec` direkt für Plugin-Quellen vom Typ `npm:`, `npm-pack:` oder `git:`
festlegen. Die Lane erteilt die von diesem Plugin benötigte ausdrückliche Genehmigung zur Installation der Codex CLI
und führt anschließend den Codex-CLI-Preflight sowie OpenAI-Agentendurchläufe in derselben Sitzung aus.
Ihr abschließender Durchlauf ohne Wiederholungen und mit mittlerer Denktiefe sendet sichtbaren Fortschritt mit ausgelassenem
Codex-`final`, liest zufällig erzeugte Workspace-Eingaben, schreibt deren exaktes Artefakt
und sendet einen ausdrücklichen Abschluss. Dadurch wird die Regression in v2026.7.1 erkannt, bei der das
Senden eines gewöhnlichen Fortschritts den Durchlauf beendete.

## Phasen der obersten Ebene

Für `rerun_group=all` wird zuerst ein `Check for reusable validation evidence`-Job ausgeführt.
Er sucht nach der neuesten vorherigen erfolgreichen vollständigen Validierung mit demselben Release-
Profil, derselben effektiven Dauertest-Einstellung und denselben Validierungseingaben. Wiederholungen für exakt dasselbe Ziel verwenden
`exact-target-full-validation-v1`. Ein Nachfolger, dessen vollständiges Delta exakt
`CHANGELOG.md` entspricht, verwendet `changelog-only-release-v1`; jede Produkt-Lane wird übersprungen
und der Verifizierer prüft unabhängig den GitHub-Commit-Vergleich, das unveränderliche
übergeordnete Artefakt, die untergeordneten Ausführungen und die Auslösungsprotokolle erneut. Jede andere Zieländerung erfordert
eine neue Code-SHA-Validierung. Übergeben Sie `reuse_evidence=false`, um eine neue vollständige
Ausführung zu erzwingen. Die Wiederverwendung von Nachweisen erfolgt nur aus `main` oder einer kanonischen, SHA-fixierten
`release-ci/*`-Referenz, deren Workflow-Commit weiterhin zur vertrauenswürdigen `main`-Abstammung gehört;
andere Workflow-Referenzen führen die ausgewählten Lanes neu aus.

Eine neue paketbezogene Validierung bereitet einen unveränderlichen Tarball und ein Docker-
Image-Artefakt vor, bevor Plugin Prerelease und OpenClaw Release Checks ausgelöst werden.
Beide untergeordneten Workflows prüfen vor der Verwendung denselben Paket-SHA, dieselben Artefakt-IDs und Dienst-Digests,
denselben Versuch der erzeugenden Ausführung sowie denselben Digest des Docker-Archivs. Die paketunabhängige
reine Docker-Schicht verwendet einen inhaltsadressierten GHCR-Cache; kandidatenspezifische Images
bleiben unveränderliche GitHub-Artefakte. Fokussierte Ausführungen mit einer ausdrücklich veröffentlichten
Paketspezifikation behalten stattdessen den bestehenden Paketpfad bei.

Ebenfalls für `rerun_group=all` erstellt ein `Verify Docker runtime image assets`-Job
das Docker-Ziel `runtime-assets` mit
`OPENCLAW_EXTENSIONS=diagnostics-otel,codex`. Er wird parallel zu den
anderen Phasen ausgeführt und vom übergreifenden Verifizierer erzwungen; Lanes warten vor dem
Auslösen nicht mehr auf ihn. Ein enger gefasster `rerun_group` überspringt diesen Preflight.

| Phase                   | Details                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Zielauflösung           | **Job:** `Resolve target ref`<br />**Untergeordneter Workflow:** keiner<br />**Nachweis:** Löst den Release-Branch, das Tag oder den vollständigen Commit-SHA auf und zeichnet die ausgewählten Eingaben auf.<br />**Wiederholung:** Führen Sie den übergreifenden Workflow erneut aus, wenn dies fehlschlägt.                                                                                                                                                                                                                                                                            |
| Gemeinsamer Kandidat    | **Job:** `Prepare shared release candidate`<br />**Untergeordneter Workflow:** `OpenClaw Live And E2E Checks (Reusable)`<br />**Nachweis:** Packt und validiert ein Paket für einen exakten SHA, erstellt ein funktionsfähiges Docker-Image und zeichnet unveränderliche Tupel aus Paket- und Image-Artefakten für beide paketbezogenen untergeordneten Workflows auf.<br />**Wiederholung:** Führen Sie die betroffene Paket-, Plugin-Prerelease-, Betriebssystem-übergreifende oder Live-/E2E-Gruppe erneut aus.                                                                                                                 |
| Preflight für Docker-Assets | **Job:** `Verify Docker runtime image assets`<br />**Untergeordneter Workflow:** keiner<br />**Nachweis:** Das Docker-Build-Ziel `runtime-assets` ist weiterhin erfolgreich, bevor eine andere Phase ausgelöst wird. Wird nur für `rerun_group=all` ausgeführt.<br />**Wiederholung:** Führen Sie den übergreifenden Workflow mit `rerun_group=all` erneut aus.                                                                                                                                                                                                                                         |
| Vitest und normale CI   | **Job:** `Run normal full CI`<br />**Untergeordneter Workflow:** `CI`<br />**Nachweis:** Manueller vollständiger CI-Graph für die Zielreferenz, einschließlich Linux-Node-Lanes, Shards gebündelter Plugins, Shards für Plugin- und Kanalverträge, Node-22-Kompatibilität, `check-*`, `check-additional-*`, Smoke-Tests für erstellte Artefakte, Dokumentationsprüfungen, Python-Skills, Windows, macOS, Control-UI-i18n und Android über den übergreifenden Workflow.<br />**Wiederholung:** `rerun_group=ci`.                                                                                          |
| Plugin-Prerelease       | **Job:** `Run plugin prerelease validation`<br />**Untergeordneter Workflow:** `Plugin Prerelease`<br />**Nachweis:** Release-spezifische statische Plugin-Prüfungen, agentische Plugin-Abdeckung, vollständige Plugin-Batch-Shards, Docker-Lanes für Plugin-Prereleases und ein nicht blockierendes `plugin-inspector-advisory`-Artefakt für die Kompatibilitäts-Triage.<br />**Wiederholung:** `rerun_group=plugin-prerelease`.                                                                                                                                                          |
| Release-Prüfungen       | **Job:** `Run release/live/Docker/QA validation`<br />**Untergeordneter Workflow:** `OpenClaw Release Checks`<br />**Nachweis:** Installations-Smoke-Test, Betriebssystem-übergreifende Paketprüfungen, Package Acceptance, QA-Lab-Parität, Live-Matrix und -Telegram sowie durch Gates geschützte Advisory-Lanes für Discord, WhatsApp und Slack. Stable- und Full-Profile führen außerdem umfassende Live-/E2E-Suites und Docker-Abschnitte für den Release-Pfad aus; Beta kann diese mit `run_release_soak=true` aktivieren.<br />**Wiederholung:** `rerun_group=release-checks` oder ein enger gefasster Release-Checks-Handle.              |
| Paket-Telegram          | **Job:** `Run package Telegram E2E`<br />**Untergeordneter Workflow:** `NPM Telegram Beta E2E`<br />**Nachweis:** Ein fokussierter Telegram-E2E-Test für ein veröffentlichtes Paket, wenn `release_package_spec` oder `npm_telegram_package_spec` festgelegt ist. Die vollständige Kandidatenvalidierung verwendet stattdessen den kanonischen Package-Acceptance-Telegram-E2E-Test.<br />**Wiederholung:** `rerun_group=npm-telegram` mit `release_package_spec` oder `npm_telegram_package_spec`.                                                                                                              |
| Produktleistung         | **Job:** `Run product performance evidence`<br />**Untergeordneter Workflow:** `OpenClaw Performance`<br />**Nachweis:** Leistungsdurchlauf für das Release-Profil (`profile=release`, `repeat=3`, `fail_on_regression=true`, `publish_reports=false`) für den Ziel-SHA. Die Kova-Ausgabe verbleibt in Workflow-Artefakten, und der untergeordnete Workflow muss nachweisen, dass sein Berichts-Publisher übersprungen wurde. Nur für `rerun_group=all` oder `rerun_group=performance` erforderlich (blockierend); für enger gefasste Wiederholungsgruppen nicht erforderlich.<br />**Wiederholung:** `rerun_group=performance`. |
| Übergreifender Verifizierer | **Job:** `Verify full validation`<br />**Untergeordneter Workflow:** keiner<br />**Nachweis:** Prüft die aufgezeichneten Ergebnisse der untergeordneten Ausführungen erneut und fügt Tabellen der langsamsten Jobs aus untergeordneten Workflows an.<br />**Wiederholung:** Führen Sie nur diesen Job erneut aus, nachdem ein fehlgeschlagener untergeordneter Workflow erfolgreich wiederholt wurde.                                                                                                                                                                                                                                                                 |

Der übergreifende Workflow löst die Produktleistung stets im reinen Artefaktmodus aus.
`OpenClaw Performance` erlaubt die Veröffentlichung von Berichten nur für geplante Ausführungen oder eine
manuelle Auslösung, die ausdrücklich `publish_reports=true` festlegt. Der Schutz für den reinen
Artefaktmodus muss erfolgreich abgeschlossen werden und damit nachweisen, dass der Publisher-Job übersprungen blieb.
Neue und wiederverwendete Nachweise zeichnen
`controls.performanceReportPublication=artifact-only` auf; der Verifizierer und die Auswahl für die Wiederverwendung
lehnen Nachweise ohne den entsprechenden normalisierten Nachweis des untergeordneten Leistungs-Workflows ab.

Der Verifizierer lädt das kanonische Manifest als
`full-release-validation-<run-id>-<run-attempt>` hoch. Die Nachweis-Tools validieren
dessen Artefakt-ID, Digest, erzeugenden Lauf und Versuch, bevor sie genau diese
Artefakt-ID herunterladen. Sie begrenzen die Größe der heruntergeladenen ZIP-Datei, prüfen deren Bytes anhand des REST-
`sha256:`-Digests und streamen den einzigen zulässigen, größenbeschränkten Manifesteintrag, ohne
das Archiv zu extrahieren. Ein Alias mit stabilem Namen bleibt vorübergehend für ältere
Veröffentlichungs-Consumer bestehen. Der Verifizierer bevorzugt stets das versuchsqualifizierte Artefakt;
während der Übergangsphase akzeptiert er den stabilen Namen nur für einen Manifest-v2-
Producer bei Versuch 1. Für spätere Versuche und Manifest v3 lehnt er diesen Legacy-Namen ab.

Für `ref=main` mit `rerun_group=all`, für `release/*`-Refs und für Tideclaw-
Alpha-Refs ersetzt ein neuerer übergeordneter Lauf einen älteren mit demselben Ref und derselben
Wiederholungslaufgruppe. Wenn der übergeordnete Lauf abgebrochen wird, bricht dessen Monitor alle untergeordneten
Workflows ab, die er bereits gestartet hat. Tag- und angeheftete-SHA-Validierungsläufe
brechen einander nicht ab.

## Phasen der Release-Prüfungen

`OpenClaw Release Checks` ist der größte untergeordnete Workflow. Er löst das Ziel
einmalig auf und validiert das gemeinsame Paketartefakt des übergeordneten Workflows, sofern verfügbar. Ein
direkter oder fokussierter Dispatch erstellt sein eigenes `release-package-under-test`-
Artefakt, wenn paket- oder Docker-bezogene Phasen es benötigen.

| Phase                    | Details                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Release-Ziel             | **Job:** `Resolve target ref`<br />**Zugrunde liegender Workflow:** keiner<br />**Tests:** ausgewählter Ref, optionale erwartete SHA, Profil, Wiederholungslaufgruppe und Filter für fokussierte Live-Suites.<br />**Wiederholungslauf:** `rerun_group=release-checks`.                                                                                                                                                                                                                                                                                                                                                             |
| Paketartefakt            | **Job:** `Prepare release package artifact`<br />**Zugrunde liegender Workflow:** keiner<br />**Tests:** validiert das unveränderliche Pakettupel des übergeordneten Workflows oder packt einen Kandidaten-Tarball für einen direkten/fokussierten Release-Checks-Dispatch und stellt ihn anschließend nachgelagerten paketbezogenen Prüfungen bereit.<br />**Wiederholungslauf:** die betroffene Paket-, Cross-OS- oder Live-/E2E-Gruppe.                                                                                                                                                                                                                                |
| Installations-Smoke-Test | **Job:** `Run install smoke`<br />**Zugrunde liegender Workflow:** `Install Smoke`<br />**Tests:** vollständiger Installationspfad mit Wiederverwendung des Smoke-Images aus dem Root-Dockerfile, QR-Paketinstallation, Docker-Smoke-Tests für Root und Gateway, Docker-Tests des Installers sowie Smoke-Test des Image-Providers bei globaler Bun-Installation.<br />**Wiederholungslauf:** `rerun_group=install-smoke`.                                                                                                                                                                                                                                                           |
| Betriebssystemübergreifend | **Job:** `cross_os_release_checks`<br />**Zugrunde liegender Workflow:** `OpenClaw Cross-OS Release Checks (Reusable)`<br />**Tests:** Neuinstallations- und Upgrade-Lanes unter Linux, Windows und macOS für den ausgewählten Provider und Modus unter Verwendung des Kandidaten-Tarballs sowie eines Baseline-Pakets.<br />**Wiederholungslauf:** `rerun_group=cross-os`.                                                                                                                                                                                                                                                                 |
| Repository- und Live-E2E | **Job:** `Run repo/live E2E validation`<br />**Zugrunde liegender Workflow:** `OpenClaw Live And E2E Checks (Reusable)`<br />**Tests:** Repository-E2E, Live-Cache, OpenAI-WebSocket-Streaming, native Live-Provider- und Plugin-Shards sowie Docker-gestützte Live-Modell-/Backend-/Gateway-Test-Harnesses, ausgewählt durch `release_profile`.<br />**Läufe:** `run_release_soak=true`, `release_profile=full` oder fokussiert `rerun_group=live-e2e`.<br />**Wiederholungslauf:** `rerun_group=live-e2e`, optional mit `live_suite_filter`.                                                                                |
| Docker-Release-Pfad      | **Job:** `Run Docker release-path validation`<br />**Zugrunde liegender Workflow:** `OpenClaw Live And E2E Checks (Reusable)`<br />**Tests:** Docker-Chunks des Release-Pfads anhand des gemeinsamen Paketartefakts.<br />**Läufe:** `run_release_soak=true`, `release_profile=full` oder fokussiert `rerun_group=live-e2e`.<br />**Wiederholungslauf:** `rerun_group=live-e2e`.                                                                                                                                                                                                                                     |
| Paketabnahme             | **Job:** `Run package acceptance`<br />**Zugrunde liegender Workflow:** `Package Acceptance`<br />**Tests:** Offline-Fixtures für Plugin-Pakete, Plugin-Aktualisierung, das kanonische Mock-OpenAI-Telegram-Paket-E2E und Überlebensprüfungen bei veröffentlichten Upgrades anhand desselben Tarballs. Blockierende Release-Prüfungen verwenden standardmäßig die neueste veröffentlichte Baseline; Soak-Prüfungen (`run_release_soak=true`) erweitern dies auf die letzten 4 stabilen npm-Releases sowie 3 angeheftete historische Versionen (`2026.4.23`, `2026.5.2`, `2026.4.15`) und werden anhand von Upgrade-Fixtures für gemeldete Probleme ausgeführt.<br />**Wiederholungslauf:** `rerun_group=package`. |
| Reifegrad-Scorecard      | **Job:** `Render maturity scorecard release docs`<br />**Zugrunde liegender Workflow:** `maturity-scorecard.yml`<br />**Tests:** rendert die beratenden Dokumente der Reifegrad-Scorecard anhand des Ziel-Refs. Wird nur ausgeführt, wenn `run_maturity_scorecard=true` übergeben wird.<br />**Wiederholungslauf:** `rerun_group=qa` mit `run_maturity_scorecard=true`.                                                                                                                                                                                                                                                           |
| QA-Parität               | **Job:** `Run QA Lab parity lane` und `Run QA Lab parity report`<br />**Zugrunde liegender Workflow:** direkte Jobs<br />**Tests:** agentische Paritätspakete für Kandidat und Baseline, anschließend der Paritätsbericht.<br />**Wiederholungslauf:** `rerun_group=qa-parity` oder `rerun_group=qa`.                                                                                                                                                                                                                                                                                                                         |
| QA-Laufzeitparität       | **Job:** `Verify QA Lab runtime-pair lanes`<br />**Zugrunde liegender Workflow:** direkter Job<br />**Tests:** die kanonische Core-Lane `openclaw`/`codex` (`pnpm openclaw qa suite --runtime-pair openclaw,codex --runtime-pair-lane core`) und mit `run_release_soak=true` die Soak-Lane. Hinweis: Einzelne Lane-Jobs blockieren den Verifizierer der Release-Prüfungen nicht.<br />**Wiederholungslauf:** `rerun_group=qa-parity` oder `rerun_group=qa`.                                                                                                                                                             |
| QA-Abdeckung der Laufzeit-Tools | **Job:** `Enforce QA Lab runtime tool coverage`<br />**Zugrunde liegender Workflow:** direkter Job<br />**Tests:** dynamische Tool-Abweichung zwischen `openclaw` und `codex` in der kanonischen Core-Laufzeitpaar-Lane (`pnpm openclaw qa coverage --tools`) unter Verwendung der Ausgabe dieser Lane. Blockierend: Dieser Job kann nicht durch einen Hinweisstatus überschrieben werden.<br />**Wiederholungslauf:** `rerun_group=qa-parity` oder `rerun_group=qa`.                                                                                                                                                                                                     |
| QA-Live-Matrix           | **Job:** `Run QA Live Matrix profile`<br />**Zugrunde liegender Workflow:** wiederverwendbarer Workflow `QA-Lab - All Lanes`<br />**Tests:** durch Parität bestätigte YAML-Szenarien über den gemeinsamen Matrix-Live-Adapter in der `qa-live-shared`-Umgebung.<br />**Wiederholungslauf:** `rerun_group=qa-live` oder `rerun_group=qa`; verwenden Sie `live_suite_filter=qa-live-matrix` für einen fokussierten Matrix-Wiederholungslauf.                                                                                                                                                                                                                    |
| QA-Live-Telegram         | **Job:** `Run QA Lab live Telegram lane`<br />**Zugrunde liegender Workflow:** vertrauenswürdiger `OpenClaw Release Telegram QA`-Dispatch<br />**Tests:** Live-Telegram-QA mit Convex-CI-Leases für Zugangsdaten.<br />**Wiederholungslauf:** `rerun_group=qa-live` oder `rerun_group=qa`.                                                                                                                                                                                                                                                                                                                                 |
| QA-Live-Discord          | **Job:** `Run QA Lab live Discord lane`<br />**Zugrunde liegender Workflow:** direkter beratender Job<br />**Tests:** Live-Discord-QA mit Convex-CI-Leases für Zugangsdaten, wenn `OPENCLAW_RELEASE_QA_DISCORD_LIVE_CI_ENABLED` aktiviert ist.<br />**Wiederholungslauf:** `rerun_group=qa-live` mit `live_suite_filter=qa-live-discord`.                                                                                                                                                                                                                                                                            |
| QA-Live-WhatsApp         | **Job:** `Run QA Lab live WhatsApp lane`<br />**Zugrunde liegender Workflow:** direkter beratender Job<br />**Tests:** Live-WhatsApp-QA mit Convex-CI-Leases für Zugangsdaten, wenn `OPENCLAW_RELEASE_QA_WHATSAPP_LIVE_CI_ENABLED` aktiviert ist.<br />**Wiederholungslauf:** `rerun_group=qa-live` mit `live_suite_filter=qa-live-whatsapp`.                                                                                                                                                                                                                                                                        |
| QA-Live-Slack            | **Job:** `Run QA Lab live Slack lane`<br />**Zugrunde liegender Workflow:** direkter beratender Job<br />**Tests:** Live-Slack-QA mit Convex-CI-Leases für Zugangsdaten, wenn `OPENCLAW_RELEASE_QA_SLACK_LIVE_CI_ENABLED` aktiviert ist.<br />**Wiederholungslauf:** `rerun_group=qa-live` mit `live_suite_filter=qa-live-slack`.                                                                                                                                                                                                                                                                                    |
| Release-Verifizierer     | **Job:** `Verify release checks`<br />**Zugrunde liegender Workflow:** keiner<br />**Tests:** erforderliche Release-Prüfungs-Jobs für die ausgewählte Wiederholungslaufgruppe.<br />**Wiederholungslauf:** erneut ausführen, nachdem fokussierte untergeordnete Jobs erfolgreich abgeschlossen wurden.                                                                                                                                                                                                                                                                                                                                                                                   |

## Docker-Release-Pfad-Blöcke

Die Docker-Release-Pfad-Phase führt diese Blöcke aus, wenn `live_suite_filter`
leer ist:

| Block                                                           | Abdeckung                                                                                                                                     |
| --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `core`                                                          | Smoke-Lanes für den zentralen Docker-Release-Pfad.                                                                                                        |
| `package-update-openai`                                         | Installations-/Aktualisierungsverhalten des OpenAI-Pakets, bedarfsgesteuerte Codex-Installation, durchgängige Live-Fortschrittsverfolgung des Codex-Plugins und Chat-Completions-Tool-Aufrufe. |
| `package-update-anthropic`                                      | Installations- und Aktualisierungsverhalten des Anthropic-Pakets.                                                                                               |
| `package-update-core`                                           | Provider-neutrales Paket- und Aktualisierungsverhalten.                                                                                                |
| `plugins-runtime-plugins`                                       | Plugin-Laufzeit-Lanes, die das Plugin-Verhalten ausführen.                                                                                          |
| `plugins-runtime-services`                                      | Dienstgestützte und Live-Lanes für die Plugin-Laufzeit.                                                                                                |
| `plugins-runtime-install-a` bis `plugins-runtime-install-h` | Für die parallele Release-Validierung aufgeteilte Plugin-Installations-/Laufzeit-Batches.                                                                        |
| `openwebui`                                                     | Auf einem dedizierten Runner mit großer Festplatte isolierter OpenWebUI-Kompatibilitäts-Smoke-Test, wenn angefordert.                                                      |

Verwenden Sie gezielt `docker_lanes=<lane[,lane]>` im wiederverwendbaren Live-/E2E-Workflow, wenn
nur eine Docker-Lane fehlgeschlagen ist. Die Release-Artefakte enthalten für jede Lane Befehle zur
erneuten Ausführung mit Eingaben zur Wiederverwendung von Paketartefakten und Images, sofern verfügbar.

## Release-Profile

`release_profile` steuert hauptsächlich den Umfang der Live-/Provider-Abdeckung innerhalb der Release-Prüfungen.
Es entfernt weder die normale vollständige CI noch Plugin-Prerelease, Installations-Smoke-Test, Paketabnahme
oder QA Lab. Stabile und vollständige Profile führen immer eine umfassende repo-/livebezogene
E2E- und Docker-Release-Pfad-Dauertestabdeckung aus. Das Beta-Profil kann diese mit
`run_release_soak=true` aktivieren. Die Paketabnahme stellt für jeden vollständigen Kandidaten den kanonischen
Paket-Telegram-E2E-Test bereit, sodass der übergeordnete Workflow diesen
Live-Poller nicht dupliziert.

| Profil  | Verwendungszweck                      | Enthaltene Live-/Provider-Abdeckung                                                                                                                                                                            |
| -------- | --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `beta`   | Schnellster releasekritischer Smoke-Test.   | OpenAI-/zentraler Live-Pfad, Docker-Live-Modelle für OpenAI, zentraler nativer Gateway, natives OpenAI-Gateway-Profil, natives OpenAI-Plugin und Docker-Live-Gateway für OpenAI.                                            |
| `stable` | Standardprofil für die Release-Freigabe. | `beta` plus Anthropic-Smoke-Test, Google, MiniMax, Backend, natives Live-Test-Harness, Docker-Live-CLI-Backend, Docker-ACP-Bindung, Docker-Codex-Harness, Docker-Subagent-Ankündigung und ein OpenCode-Go-Smoke-Shard. |
| `full`   | Breiter beratender Durchlauf.             | `stable` plus beratende Provider, Plugin-Live-Shards und Medien-Live-Shards.                                                                                                                               |

## Nur bei vollständigen Profilen enthaltene Ergänzungen

Diese Suites werden von `stable` übersprungen und von `full` einbezogen:

| Bereich                             | Nur bei vollständigen Profilen enthaltene Abdeckung                                                                                                          |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Docker-Live-Modelle               | OpenCode Go, OpenRouter, xAI, Z.ai und Fireworks.                                                                          |
| Docker-Live-Gateway              | Beratende Provider, aufgeteilt in die Shards DeepSeek/Fireworks, OpenCode Go/OpenRouter und xAI/Z.ai.                              |
| Native Gateway-Provider-Profile | Vollständige Anthropic-Opus- und Sonnet-/Haiku-Shards, Fireworks, DeepSeek, vollständige OpenCode-Go-Modell-Shards, OpenRouter, xAI und Z.ai. |
| Native Plugin-Live-Shards        | Plugins A–K, L–N, sonstige O–Z, Moonshot und xAI.                                                                             |
| Native Medien-Live-Shards         | Audio, Google-Musik, MiniMax-Musik und Videogruppen A–D.                                                                   |

`stable` enthält `native-live-src-gateway-profiles-anthropic-smoke` und
`native-live-src-gateway-profiles-opencode-go-smoke`; `full` verwendet stattdessen die breiteren
Anthropic- und OpenCode-Go-Modell-Shards. Gezielte erneute Ausführungen können weiterhin die
aggregierten Handles `native-live-src-gateway-profiles-anthropic` oder
`native-live-src-gateway-profiles-opencode-go` verwenden.

## Gezielte erneute Ausführungen

Verwenden Sie `rerun_group`, um die Wiederholung nicht zugehöriger Release-Umgebungen zu vermeiden:

| Handle              | Umfang                                                                                           |
| ------------------- | ----------------------------------------------------------------------------------------------- |
| `all`               | Alle Phasen der vollständigen Release-Validierung.                                                             |
| `ci`                | Nur der untergeordnete manuelle vollständige CI-Workflow.                                                                      |
| `plugin-prerelease` | Nur der untergeordnete Plugin-Prerelease-Workflow.                                                                   |
| `release-checks`    | Alle Phasen der OpenClaw-Release-Prüfungen.                                                             |
| `install-smoke`     | Vom Installations-Smoke-Test bis zu den Release-Prüfungen.                                                           |
| `cross-os`          | Betriebssystemübergreifende Release-Prüfungen.                                                                        |
| `live-e2e`          | Repo-/Live-E2E- und Docker-Release-Pfad-Validierung.                                               |
| `package`           | Paketabnahme.                                                                             |
| `qa`                | QA-Parität plus QA-Live-Lanes.                                                                   |
| `qa-parity`         | Nur QA-Paritäts-Lanes und Bericht.                                                                |
| `qa-live`           | QA-Live-Lanes für Matrix/Telegram sowie bei Aktivierung zugangsgesteuerte Lanes für Discord, WhatsApp und Slack.             |
| `npm-telegram`      | Telegram-E2E-Test für veröffentlichte Pakete; erfordert `release_package_spec` oder `npm_telegram_package_spec`. |
| `performance`       | Nur Nachweise zur Produktleistung.                                                              |

Verwenden Sie `live_suite_filter` mit `rerun_group=live-e2e`, wenn eine Live-Suite fehlgeschlagen ist.
Gültige Filter-IDs sind im wiederverwendbaren Live-/E2E-Workflow definiert, darunter
`docker-live-models`, `live-gateway-docker`,
`live-gateway-anthropic-docker`, `live-gateway-google-docker`,
`live-gateway-minimax-docker`, `live-gateway-advisory-docker`,
`live-cli-backend-docker`, `live-acp-bind-docker` und
`live-codex-harness-docker`.

Legen Sie für eine gezielte erneute Ausführung eines QA-Transports `rerun_group=qa-live` fest und verwenden Sie den
kanonischen Selektor `qa-live-matrix`, `qa-live-telegram`, `qa-live-discord`,
`qa-live-whatsapp` oder `qa-live-slack`.

Das Handle `live-gateway-advisory-docker` ist ein aggregiertes Handle zur erneuten Ausführung seiner
drei Provider-Shards und verteilt sich daher weiterhin auf alle beratenden Docker-Gateway-Jobs.

Verwenden Sie `cross_os_suite_filter` mit `rerun_group=cross-os`, wenn eine betriebssystemübergreifende Lane
fehlgeschlagen ist. Der Filter akzeptiert eine Betriebssystem-ID, eine Suite-ID oder ein Betriebssystem-/Suite-Paar,
beispielsweise `windows/packaged-upgrade`, `windows` oder `packaged-fresh`. Betriebssystemübergreifende
Zusammenfassungen enthalten phasenbezogene Zeitangaben für paketierte Upgrade-Lanes, und lang laufende
Befehle geben Heartbeat-Zeilen aus, sodass eine festhängende Aktualisierung vor dem
Job-Timeout sichtbar wird.

Fehler bei QA-Release-Prüfungen blockieren die normale Release-Validierung nur für ausgewählte
Abdeckungs-Lanes der Matrix-, Telegram- und QA-Laufzeit-Tools. QA-Parität, Laufzeitparität
und die zugangsgesteuerten Live-Lanes für Discord, WhatsApp und Slack sind beratend und
veröffentlichen Statusartefakte, ohne den Release-Prüfer zu blockieren. Tideclaw-
Alpha-Ausführungen können Release-Prüfungs-Lanes, die nicht die Paketsicherheit betreffen, weiterhin als beratend behandeln. Mit
`release_profile=beta` sind die Live-Provider-Suites `Run repo/live E2E validation`
beratend: Bereitstellungen von Drittanbieter-Modellen ändern sich während eines Releases, daher
stellt Beta deren Fehler als Warnungen dar, während stabile und vollständige Profile sie weiterhin
blockierend behandeln. Wenn
`live_suite_filter` ausdrücklich eine zugangsgesteuerte QA-Live-Lane wie Discord,
WhatsApp oder Slack anfordert, muss die entsprechende Repo-Variable `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED`
aktiviert sein; andernfalls schlägt die Eingabeerfassung fehl, statt die Lane stillschweigend zu überspringen.
Führen Sie `rerun_group=qa`, `qa-parity` oder `qa-live` erneut aus, wenn Sie
aktuelle QA-Nachweise benötigen.

## Aufzubewahrende Nachweise

Bewahren Sie die Zusammenfassung `Full Release Validation` als Index auf Release-Ebene auf. Sie verlinkt
die IDs untergeordneter Ausführungen und enthält Tabellen der langsamsten Jobs. Untersuchen Sie bei Fehlern zuerst den
untergeordneten Workflow und führen Sie anschließend das kleinste passende Handle oben erneut aus.

Dokumentieren Sie für ein reguläres Release sowohl den Code-SHA als auch den Release-SHA, die Wiederverwendungsrichtlinie
und die Menge geänderter Pfade, die erfolgreiche übergeordnete Ausführung des Code-SHA sowie die leichtgewichtige übergeordnete
Ausführung des Release-SHA. Dokumentieren Sie für Extended Stable den kanonischen Branch, den exakten Release-
SHA, die ID und den Versuch der neuen übergeordneten Ausführung, die Workflow-Referenz, jede untergeordnete Ausführung sowie jede
Kompatibilitätsreparatur des eingefrorenen Ziels oder beabsichtigte Auslassung.

Nützliche Artefakte:

- `release-package-under-test` aus `OpenClaw Release Checks`
- Docker-Release-Pfad-Artefakte unter `.artifacts/docker-tests/`
- Paketabnahme `package-under-test` und Docker-Abnahmeartefakte
- Betriebssystemübergreifende Release-Prüfungsartefakte für jedes Betriebssystem und jede Suite
- QA-Parität, Laufzeitparität und ausgewählte Artefakte für Matrix, Telegram, Discord, WhatsApp,
  oder Slack

## Workflow-Dateien

- `.github/workflows/full-release-validation.yml`
- `.github/workflows/openclaw-release-checks.yml`
- `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml`
- `.github/workflows/plugin-prerelease.yml`
- `.github/workflows/install-smoke.yml`
- `.github/workflows/install-smoke-reusable.yml`
- `.github/workflows/openclaw-cross-os-release-checks-reusable.yml`
- `.github/workflows/package-acceptance.yml`
- `.github/workflows/openclaw-performance.yml`
- `.github/workflows/npm-telegram-beta-e2e.yml`
