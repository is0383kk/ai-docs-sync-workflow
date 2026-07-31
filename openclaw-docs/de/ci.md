---
read_when:
    - Sie müssen verstehen, warum ein CI-Job ausgeführt wurde oder nicht.
    - Sie debuggen eine fehlgeschlagene GitHub-Actions-Prüfung
    - Sie koordinieren einen Validierungslauf oder erneuten Validierungslauf für ein Release
    - Sie ändern den ClawSweeper-Dispatch oder die Weiterleitung von GitHub-Aktivitäten
summary: CI-Jobgraph, Bereichs-Gates, Release-Sammelworkflows und entsprechende lokale Befehle
title: CI-Pipeline
x-i18n:
    generated_at: "2026-07-26T17:40:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9de5b527354f3cc9eed3813e961116f3834c61bd72b29c92f762c46722815df
    source_path: ci.md
    workflow: 16
---

OpenClaw CI wird bei Pushes auf `main` ausgeführt (Markdown- und `docs/**`-Pfade werden
beim Trigger ignoriert), bei jedem Pull Request, der kein Entwurf ist, sowie bei manueller Auslösung.
Kanonische Pushes auf `main` werden einzeln ausgeführt: Die Parallelitätsgruppe `CI` lässt einen
vollständigen Integrationszyklus laufen, während GitHub nur den neuesten ausstehenden Push behält.
Neue Merges ersetzen diesen ausstehenden Lauf, anstatt bereits laufende Arbeit abzubrechen, die
eine Blacksmith-Matrix registriert hat. Bei Pull Requests werden weiterhin überholte Heads abgebrochen,
und manuelle Auslösungen verwenden isolierte Gruppen. `preflight` klassifiziert den Diff und
deaktiviert aufwendige Lanes, wenn sich nur nicht zugehörige Bereiche geändert haben. Manuelle
`workflow_dispatch`-Läufe umgehen die intelligente Eingrenzung absichtlich und fächern den
vollständigen Graphen für Release-Kandidaten und umfassende Validierungen auf. Android-Lanes bleiben
über `include_android` (oder die Eingabe `release_gate`) optional. Die ausschließlich für Releases bestimmte
Plugin-Abdeckung befindet sich im separaten
[`Plugin Prerelease`](#plugin-prerelease)-Workflow und wird nur über
[`Full Release Validation`](#full-release-validation) oder durch eine explizite manuelle
Auslösung gestartet.

## Pipeline-Übersicht

| Job                                | Zweck                                                                                                                                                                                                               | Ausführungszeitpunkt                                   |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| `preflight`                        | Geänderte Bereiche erkennen und das CI-Manifest erstellen; bei kanonischen Node-relevanten `main` den Abhängigkeits-Snapshot vor dem Auffächern aktualisieren und pflegen                                                                        | Immer bei Pushes und PRs, die keine Entwürfe sind             |
| `security-fast`                    | Erkennung privater Schlüssel, Prüfung geänderter Workflows über `zizmor` und Prüfung der produktiven Lockfile                                                                                                                             | Immer bei Pushes und PRs, die keine Entwürfe sind             |
| `pnpm-store-warmup`                | Den durch die Lockfile festgelegten Actions-Cache für Pull Requests und manuelle Läufe vorwärmen, ohne Linux-Node-Shards zu blockieren                                                                                                           | Wenn Node- oder Dokumentationsprüfungs-Lanes außerhalb von Main ausgewählt sind |
| `build-artifacts`                  | `dist/`, Control UI, Smoke-Tests der erstellten CLI, Startspeicher und eingebettete Prüfungen erstellter Artefakte erstellen                                                                                                                 | Bei Node-relevanten Änderungen                          |
| `control-ui-i18n`                  | Generierte Locale-Bundles, Metadaten und Translation Memory der Control UI überprüfen; bei automatischen Läufen informativ, bei manueller Release-CI blockierend                                                                               | Bei für die Control-UI-Internationalisierung relevanten Änderungen und manueller CI |
| `checks-fast-core`                 | Schnelle Linux-Korrektheits-Lanes: Maximalzeilen-Ratchet der Unterdrückungs-Baseline, gebündelte Komponenten + Protokoll, Bun-Launcher und die schnelle CI-Routing-Aufgabe                                                                                  | Bei Node-relevanten Änderungen                          |
| `qa-smoke-ci-profile`              | Zwei eigenständige, ausgewogene Teile der begrenzten repräsentativen automatischen QA-Smoke-Testmenge; die vollständige Taxonomieabdeckung bleibt über explizite QA-Profile verfügbar                                                         | Bei Node-relevanten Änderungen                          |
| `checks-fast-contracts-plugins-*`  | Zwei gewichtete Plugin-Vertrags-Shards                                                                                                                                                                                   | Bei Node-relevanten Änderungen                          |
| `checks-fast-contracts-channels-*` | Zwei gewichtete Kanal-Vertrags-Shards                                                                                                                                                                                  | Bei Node-relevanten Änderungen                          |
| `checks-node-*`                    | Node-Tests für geänderte Ziele bei Pull Requests; vollständige Core-Shards bei `main` sowie bei manuellen, Release- und umfassenden Fallback-Läufen                                                                                                      | Bei Node-relevanten Änderungen                          |
| `check-*`                          | In Shards aufgeteiltes Äquivalent des lokalen Main-Gates: Schutzprüfungen, Shrinkwrap, Konfigurationsmetadaten gebündelter Kanäle, Produktionstypen, Lint, Abhängigkeiten, Testtypen                                                                                   | Bei Node-relevanten Änderungen                          |
| `check-additional-*`               | Streifenweise Grenzprüfungen (einschließlich Abweichungen bei Prompt-Snapshots), Grenzen für Session-Zugriff, Transkriptleser und SQLite-Transaktionen, Lint-Gruppen für Erweiterungen, Kompilierung/Canary für Paketgrenzen sowie Architektur der Laufzeittopologie | Bei Node-relevanten Änderungen                          |
| `checks-node-compat-node22`        | Kompatibilitäts-Build und Smoke-Lane für Node 22                                                                                                                                                                            | Bei manueller CI-Auslösung für Releases                |
| `check-docs`                       | Formatierungs-, Lint- und Defektlink-Prüfungen der Dokumentation                                                                                                                                                                         | Wenn sich die Dokumentation geändert hat (PRs und manuelle Auslösung)         |
| `native-i18n`                      | Sichere Extraktion und Lokalisierung nativer Quellen bei Quell-PRs überprüfen; vollständige Parität übersetzter und plattformgenerierter Inhalte bei generierten PRs und manueller CI erzwingen                                                               | Bei für native Internationalisierung relevanten Änderungen                   |
| `skills-python`                    | Ruff + pytest für Python-basierte Skills                                                                                                                                                                                | Bei für Python-Skills relevanten Änderungen                  |
| `checks-windows`                   | Windows-spezifische Prozess-/Pfadtests sowie gemeinsame Regressionstests für Import-Spezifizierer der Laufzeit                                                                                                                                  | Bei Windows-relevanten Änderungen                       |
| `macos-node`                       | Fokussierte macOS-TypeScript-Tests: launchd, Homebrew, Laufzeitpfade, Paketierungsskripte, Prozessgruppen-Wrapper                                                                                                            | Bei macOS-relevanten Änderungen                         |
| `macos-swift`                      | Swift-Lint und Build für die macOS-App sowie Tests für die App und das gemeinsame OpenClawKit-Paket                                                                                                                         | Bei macOS-relevanten Änderungen                         |
| `ios-build`                        | Xcode-Projektgenerierung sowie Simulator-Build der iOS-App                                                                                                                                                             | Bei Änderungen an der iOS-App, dem gemeinsamen App-Kit oder Swabble    |
| `android`                          | Android-Unit-Tests für beide Varianten sowie ein Debug-APK-Build                                                                                                                                                          | Bei Android-relevanten Änderungen                       |
| `openclaw/ci-gate`                 | Abschließendes Aggregat: erfordert Preflight und Sicherheit; akzeptiert Überspringen nur für durch das Manifest deaktivierte nachgelagerte Lanes                                                                                                           | Bei jedem CI-Lauf, der kein Entwurf ist                         |
| `test-performance-agent`           | Separater Workflow: tägliche Optimierung langsamer Codex-Tests nach vertrauenswürdiger Aktivität                                                                                                                                          | Nach erfolgreicher Main-CI oder manueller Auslösung             |
| `openclaw-performance`             | Separater Workflow: tägliche/bedarfsgesteuerte Kova-Laufzeit-Leistungsberichte mit Mock-Provider-, Deep-Profile- und GPT-5.6-Live-Lanes                                                                                          | Zeitgesteuert und bei manueller Auslösung                  |

Eigenständige Periphery-Workflows erzwingen, dass für die iOS- und macOS-Apps keine ungenutzten Codebestandteile gefunden werden. Der gemeinsame OpenClawKit-Workflow scannt beide Verbraucher parallel und meldet eine Deklaration nur, wenn Periphery aus beiden Builds dieselbe Swift-USR ausgibt. Sein generierter `OpenClawProtocol/GatewayModels.swift`-Schemavertrag bleibt als generatorverwalteter Code erhalten, statt als app-lokaler ungenutzter Code behandelt zu werden.

## Fail-Fast-Reihenfolge

1. `preflight` entscheidet, welche Lanes überhaupt vorhanden sind. Die Logik von `docs-scope` und `changed-scope` besteht aus Schritten innerhalb dieses Jobs und nicht aus eigenständigen Jobs. Kanonisches `main` startet sofort, aber seine Parallelitätsgruppe lässt nur einen vollständigen Lauf zu und fasst spätere Pushes zum neuesten ausstehenden Lauf zusammen. Node-relevante Main-Pushes serialisieren hier außerdem den einzigen Schreiber auf den Abhängigkeitsdatenträger und dessen Größenverwaltung, bevor nachgelagerte Jobs den Schlüssel einbinden dürfen; Blacksmith stellt einen neuen Commit möglicherweise erst einem späteren Workflow-Lauf bereit, weshalb Verbraucher desselben Laufs den durch Marker geprüften lokalen Fallback beibehalten.
2. `security-fast`, `check-*`, `check-additional-*`, `check-docs` und `skills-python` schlagen schnell fehl, ohne auf die umfangreicheren Artefakt- und Plattformmatrix-Jobs zu warten.
3. `build-artifacts` und die Locale-Prüfungen laufen parallel zu den schnellen Linux-Lanes. Quell-PRs der Control UI und nativen Apps schließen generierte Locale-Snapshots/-Ressourcen aus; ihre serialisierten Aktualisierungs-Workflows reparieren und führen isolierte generierte PRs automatisch im Hintergrund zusammen. Die Quell-CI blockiert weiterhin bei veralteten Quellinventaren und unsicheren Lokalisierungsaufrufen. Generierte PRs, manuelle CI und Release-Vorbereitung erzwingen vollständige Parität übersetzter und plattformgenerierter Inhalte. Kanonische `release/YYYY.M.PATCH`-Branches können Reparaturen der Locale für die Release-Vorbereitung zusammen mit den übrigen generierten Release-Ausgaben enthalten.
4. Anschließend werden umfangreichere Plattform- und Laufzeit-Lanes aufgefächert: `checks-fast-core`, `checks-fast-contracts-plugins-*`, `checks-fast-contracts-channels-*`, `checks-node-*`, `checks-windows`, `macos-node`, `macos-swift`, `ios-build` und `android`.
5. `openclaw/ci-gate` wartet auf jede ausgewählte Lane. Preflight und Sicherheit müssen erfolgreich sein; nachgelagerte Jobs dürfen nur übersprungen werden, wenn das Manifest sie nicht ausgewählt hat. Eine fehlgeschlagene oder abgebrochene ausgewählte Lane lässt das Aggregat fehlschlagen.

Der Merge-Koordinator kann ein authentifiziertes, erfolgreiches `openclaw/ci-gate`
für denselben Pull-Request-Head bis zu 24 Stunden lang wiederverwenden. Dadurch muss ein
Contributor-Branch nach nicht zugehörigen Änderungen an `main` nicht neu geschrieben werden. Das wiederverwendbare Ergebnis
ersetzt nicht die separate strikte, App-eigene Test-Merge-Prüfung gegen den aktuellen Stand von `main`.
Ein späterer ausstehender oder fehlgeschlagener Wiederholungslauf löscht während des Aktualitätsfensters
kein früheres erfolgreiches Ergebnis für diesen unveränderten Head.

Das Regelsatz für den Standard-Branch erfordert den GitHub-Actions-eigenen Check `openclaw/ci-gate`. Repository-Maintainer und -Administratoren verfügen über eine auditierte Notfallumgehung, die ausschließlich für signierte direkte Fast-Forward-Landings vorgesehen ist; der Organisationsregelsatz blockiert weiterhin Löschungen und Nicht-Fast-Forward-Aktualisierungen. Normale Pull-Request-Merges sollten weiterhin das Gate verwenden, anstatt eine fehlgeschlagene CI zu umgehen. Der separate strikte App-eigene Test-Merge-Check bindet den Head weiterhin an den aktuellen `main`.

GitHub kann ersetzte Pull-Request-Jobs als `cancelled` markieren, wenn ein neuerer Head landet. Behandeln Sie dies als CI-Rauschen, sofern nicht auch der neueste Lauf für denselben PR fehlschlägt. Kanonische `main`-Läufe werden nach der Zulassung nicht abgebrochen; wenn Merge-Aktivität eintrifft, ersetzt GitHub nur den älteren ausstehenden Lauf durch den neuesten Stand. Matrix-Jobs verwenden `fail-fast: false`, und `build-artifacts` meldet eingebettete Kanal-, Core-Support-Grenz- und Gateway-Watch-Fehler direkt, anstatt kleine Prüfer-Jobs in die Warteschlange zu stellen. Der automatische CI-Parallelitätsschlüssel ist versioniert (`CI-v7-*`), damit ein GitHub-seitiger Zombie in einer alten Warteschlangengruppe neuere Main-Läufe nicht unbegrenzt blockieren kann. Manuelle vollständige Suite-Läufe verwenden `CI-manual-v1-*` und brechen laufende Läufe nicht ab. Die Speicherbegrenzung beim Start der Plugin-Liste hält auf selbst gehostetem Blacksmith Linux eine Obergrenze von 350 MiB ein und erlaubt auf von GitHub gehostetem Linux 425 MiB, dessen RSS-Ausgangswert für dieselbe gebaute CLI höher ist.

Verwenden Sie `pnpm ci:timings`, `pnpm ci:timings:recent` oder `node scripts/ci-run-timings.mjs <run-id>`, um Gesamtlaufzeit, Warteschlangenzeit, langsamste Jobs, Fehler und die `pnpm-store-warmup`-Fanout-Barriere aus GitHub Actions zusammenzufassen. Der workflowinterne Job `ci-timings-summary` ist in `ci.yml` vorhanden, derzeit jedoch deaktiviert (`if: false`); führen Sie stattdessen den Timing-Helfer lokal aus. Prüfen Sie für die Build-Zeitmessung im Job `build-artifacts` den Schritt `Build dist`: `pnpm build:ci-artifacts` gibt `[build-all] phase timings:` aus und enthält `ui:build`; der Job lädt außerdem das Artefakt `startup-memory` hoch.

## PR-Kontext und Nachweise

PRs externer Mitwirkender durchlaufen ein Gate für PR-Kontext und Nachweise aus
`.github/workflows/real-behavior-proof.yml`. Der Workflow checkt die
vertrauenswürdige Workflow-Revision (`github.workflow_sha`) aus und wertet nur den PR-Text
aus; er führt keinen Code aus dem Branch des Mitwirkenden aus.

Das Gate gilt für PR-Autoren, die weder Repository-Eigentümer noch Mitglieder,
Mitwirkende mit Zugriff oder Bots sind. Es wird bestanden, wenn der PR-Text selbst verfasste
Abschnitte `What Problem This Solves` und `Evidence` enthält. Als Nachweis können ein fokussierter
Test, ein CI-Ergebnis, ein Screenshot, eine Aufzeichnung, eine Terminalausgabe, eine Live-Beobachtung,
ein bereinigtes Protokoll oder ein Artefakt-Link dienen. Der Text beschreibt die Absicht und eine aussagekräftige Validierung;
Reviewer prüfen Code, Tests und CI, um die Korrektheit zu beurteilen.

Wenn der Check fehlschlägt, aktualisieren Sie den PR-Text, anstatt einen weiteren Code-Commit zu pushen.

## Umfang und Routing

Die Umfangslogik befindet sich in `scripts/ci-changed-scope.mjs` und wird durch Unit-Tests in `src/scripts/ci-changed-scope.test.ts` abgedeckt. Bei manueller Ausführung wird die Erkennung geänderter Bereiche übersprungen, und das Preflight-Manifest verhält sich so, als hätten sich alle Bereiche geändert.

Separate Periphery-Workflows für iOS und macOS erzwingen eine Null-Fundstellen-Richtlinie für ungenutzten Code. Sie werden jeweils nur ausgeführt, wenn ein nicht als Entwurf markierter Pull Request den jeweiligen nativen Scanbereich berührt oder wenn sie manuell ausgelöst werden.

- **Änderungen am CI-Workflow** validieren den Node-CI-Graphen, das Workflow-Linting und den Windows-Lane (`ci.yml` führt ihn aus), erzwingen jedoch nicht eigenständig native Builds für iOS, Android oder macOS; diese Plattform-Lanes bleiben auf Änderungen am jeweiligen Plattformquellcode beschränkt.
- **Workflow-Plausibilitätsprüfung** führt `actionlint`, `zizmor` für alle Workflow-YAML-Dateien, die Interpolationsprüfung für zusammengesetzte Actions und die Prüfung auf Konfliktmarkierungen aus. Der PR-bezogene Job `security-fast` führt außerdem `zizmor` für geänderte Workflow-Dateien aus, damit Workflow-Sicherheitsbefunde frühzeitig im Haupt-CI-Graphen fehlschlagen.
- **Dokumentation bei Pushes auf `main`** wird durch den eigenständigen Workflow `Docs` mit demselben ClawHub-Dokumentationsspiegel geprüft, den auch die CI verwendet, sodass gemischte Code-und-Dokumentations-Pushes nicht zusätzlich den CI-Shard `check-docs` in die Warteschlange stellen. Pull Requests und manuelle CI führen bei geänderter Dokumentation weiterhin `check-docs` aus der CI aus.
- **TUI-PTY** wird bei TUI-Änderungen im Linux-Node-Shard `checks-node-core-runtime-tui-pty` ausgeführt. Der Shard führt `test/vitest/vitest.tui-pty.config.ts` mit `OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1` aus und deckt damit sowohl den deterministischen Fixture-Lane `TuiBackend` als auch den langsameren Smoke-Test `tui --local` ab, der nur den externen Modellendpunkt simuliert.
- **Reine CI-Routing-Änderungen, die kleine Gruppe von Core-Test-Fixtures, die der schnelle Task direkt ausführt, und eng begrenzte Änderungen an Hilfsfunktionen für Plugin-Verträge** verwenden einen schnellen, ausschließlich auf Node basierenden Manifestpfad: `preflight`, `security-fast` und nur die schnellen Lanes, die von der Änderung betroffen sind – einen einzelnen CI-Routing-Task `checks-fast-core`, die beiden Plugin-Vertrags-Shards oder beide. Dieser Pfad überspringt Build-Artefakte, Node-22-Kompatibilität, Kanalverträge, vollständige Core-Shards, Shards gebündelter Plugins und zusätzliche Schutzmatrizen.
- **Windows-Node-Prüfungen** sind auf Windows-spezifische Prozess-/Pfad-Wrapper, npm-/pnpm-/UI-Runner-Hilfsfunktionen, die Paketmanagerkonfiguration und die CI-Workflow-Flächen beschränkt, die diesen Lane ausführen; nicht verwandte Quellcode-, Plugin-, Installations-Smoke- und reine Teständerungen verbleiben auf den Linux-Node-Lanes.

Die langsamsten Node-Testfamilien werden aufgeteilt oder ausbalanciert, sodass jeder Job klein bleibt, ohne übermäßig viele Runner zu reservieren:

- Plugin-Verträge und Channel-Verträge werden jeweils als zwei gewichtete, von Blacksmith unterstützte Shards mit dem standardmäßigen GitHub-Runner-Fallback ausgeführt.
- Schnelle/unterstützende Core-Unit-Lanes werden separat ausgeführt; die Core-Runtime-Infrastruktur wird in Prozess-, Shared-, Hook-, Secrets- und drei Cron-Domänen-Shards aufgeteilt.
- Auto-Reply wird mit ausgewogenen Workern ausgeführt, wobei der Reply-Unterbaum in Agent-Runner-, Befehls-, Dispatch-, Sitzungs- und State-Routing-Shards aufgeteilt wird.
- Agentische Gateway-/Server-Konfigurationen (Steuerungsebene) werden auf Chat-, Authentifizierungs-, Modell-, HTTP-/Plugin-, Runtime- und Start-Lanes aufgeteilt, statt auf erstellte Artefakte zu warten.
- Die normale CI bündelt nur isolierte Infrastruktur-Shards mit Include-Mustern in deterministische Pakete aus höchstens 64 Testdateien. Dadurch wird die Node-Matrix verkleinert, ohne nicht isolierte Befehls-/Cron-, zustandsbehaftete Agents-Core- oder Gateway-/Server-Suites zusammenzuführen. Umfangreiche feste Suites bleiben auf 8 vCPU, während die gebündelten und geringer gewichteten Lanes 4 vCPU verwenden.
- Pull Requests im kanonischen Repository verwenden den Resolver für geänderte Tests erneut für den synthetischen Diff des zusammengeführten Baums. Präzise Änderungen führen einen gezielten Node-Job aus; jede ausgewählte Testdatei erhält einen eigenen Prozess, damit die Isolation zustandsbehafteter Suites erhalten bleibt. Der Planer kombiniert gleichgeordnete Tests mit vom Importgraphen abhängigen Tests und greift auf den bestehenden kompakten Vollsuite-Plan mit 14 Jobs zurück bei Änderungen an Workspace-Paketen, Paketen/Lockfiles, gemeinsamem Harness, Split-Konfigurationen, umbenannten oder gelöschten Dateien, öffentlichen Extension-Verträgen, Tests mit spezieller Shard-Einrichtung, teilweise aufgelösten oder leeren Zielen, übergroßen Pfad- oder Zielplänen sowie Planerfehlern. Gezielte Pläne behalten stets das vollständige Boundary-Gate für erstellte Artefakte bei, da dessen Repository-Scanner nicht aus Importen abgeleitet werden können. `main`-Pushes führen dieselbe vollständige kompakte Suite aus: Ausstehende zwischenzeitliche Push-Ereignisse können zusammengefasst werden, daher muss der neueste verbleibende Lauf den vollständigen Integrationsbaum validieren und nicht nur seinen endgültigen Einzel-Push-Diff. Manuelle Ausführungen und Release-Gates behalten die vollständige benannte Matrix pro Shard bei.
- Die vollständige Node-Matrix lässt zuerst die konstant langsamen seriellen Tooling-, Auto-Reply-Befehls-Shards und den umfassenden Core-Fast-Cache-Schreiber zu. Dadurch bleibt die Obergrenze von 28 Jobs erhalten, während verhindert wird, dass Arbeiten auf dem kritischen Pfad und der Transform-Seed des nächsten Laufs in eine spätere Welle rutschen.
- Umfassende Browser-, QA-, Medien- und sonstige Plugin-Tests verwenden ihre dedizierten Vitest-Konfigurationen statt des gemeinsamen Plugin-Catch-alls. Shards mit Include-Mustern zeichnen Zeiteinträge unter Verwendung des CI-Shard-Namens auf, sodass `.artifacts/vitest-shard-timings.json` eine vollständige Konfiguration von einem gefilterten Shard unterscheiden kann.
- Linux-Node-Shard-Jobs persistieren Vitests experimentellen Dateisystem-Modul-Cache über die Upstream-Actions-Cache-API, die Blacksmith auf seinen Runnern transparent beschleunigt. Jeder CI-Shard stellt nur wieder her und entpackt den geschützten Seed in sein eigenes runnerlokales Stammverzeichnis; der Shard-Wrapper weist gleichzeitigen Vitest-Prozessen anschließend separate aktive Unterverzeichnisse zu. Nur der nicht abbrechende tägliche oder ausdrücklich ausgelöste Warmer speichert ein neues unveränderliches Archiv, sodass Pull Requests weder Transformationen veröffentlichen noch PR-spezifische Cache-Familien erzeugen können. Ein Fingerabdruck der Transformations-Eingaben verwirft inkompatible Generationen von Lockfile, Paket, tsconfig und Vitest-Konfiguration. Der geschützte Schreiber scannt und bereinigt seinen wiederhergestellten Cache auf 75 %, nachdem dieser 2 GiB überschritten hat. Vitest hasht Modul-ID, Quellinhalt, Umgebung und aufgelöste Transformationskonfiguration, sodass gewöhnliche partielle Quelländerungen unveränderte Einträge warm halten, während geänderte Module sicher einen Cache-Miss erzeugen. Grobe Wiederherstellungspräfixe überbrücken Workflow-Läufe; die normale LRU- und Inaktivitätsbereinigung des Actions-Cache begrenzt alte unveränderliche Archive.
- Vertrauenswürdige Linux-Node-Jobs binden außerdem den pnpm-Store und `node_modules` aus einer geschützten Abhängigkeitsfestplatte pro unterstützter Node-Linie ein. Paketmanifeste, Installationseinstellungen, Runner-Plattform und der exakte Node-Patch bleiben außerhalb des Festplattenschlüssels; ein exakter Fingerabdruck von Runtime und Installations-Eingaben entscheidet, ob ein Job den Baum wiederverwendet oder neu installiert und dieselbe Festplatte aktualisiert. Manifeste werden vor dem Hashing kanonisiert. Die geprüften direkten Root-Hooks behalten nur die Installations-Lifecycle-Skripte von pnpm bei, sodass Änderungen an Formatierungs- und gewöhnlichen Test-/Build-Skripten den warmen Abhängigkeitsbaum erhalten; ungeprüfte Abweichungen bei Lifecycle-Hooks schlagen sicher geschlossen fehl, bis ihre Quelleingaben in den Fingerabdruckvertrag aufgenommen wurden. Änderungen an Abhängigkeiten, Paketmanager, Hook-Quellen und Lockfile machen den Snapshot stets ungültig. Ein übereinstimmender Fingerabdruck ist notwendig, aber nicht hinreichend: Die Einrichtung prüft außerdem das Importer-Archiv und die Manifest-Prüfsummen und verifiziert anschließend die von postinstall beibehaltenen, Registry-gestützten Lockfile-Abhängigkeiten anhand der Paketmanifeste, die Node von ihren Importern auflöst. Fehlende oder veraltete Importer-Inhalte führen zu einer Neuinstallation, statt das Root-Hoisting bereitzustellen. Ein Pull Request, dessen schreibgeschützter Snapshot unbrauchbar ist, löst die Workspace-Einbindung und installiert in runnerlokalen Speicher, wodurch langsame Schreibvorgänge in einen Klon vermieden werden, den er nicht veröffentlichen kann. Kalte Installationen auf Sticky Disks deaktivieren die internen Abrufwiederholungen von pnpm und unternehmen bis zu drei begrenzte vollständige Installationsversuche aus dem fortschreitend aufgewärmten Store; ein Timeout bleibt ein Fehler. Nach einer inhaltsvalidierten Wiederherstellung oder einer Installation mit eingefrorenem Lockfile deaktiviert die Einrichtung die redundante Abhängigkeitsprüfung vor dem Lauf von pnpm: Das Repository bereinigt absichtlich Plugin-lokale `node_modules`, die pnpm andernfalls als veraltet betrachtet und durch unsichere gleichzeitige implizite Installationen während des Shard-Fan-outs repariert. Der kanonische Main-Preflight ist der einzige Schreiber und misst den Store bei jeder Aktualisierung; `pnpm store prune` wird erst ausgeführt, nachdem ausgemusterte Paketversionen ihn über 8 GiB anwachsen lassen. Die Veröffentlichung von Blacksmith-Snapshots erfolgt auch nach Abschluss eines Schreiber-Jobs asynchron, sodass der erste Lauf nach einem neuen Schlüssel oder Fingerabdruck weiterhin kalt bleiben kann; spätere inhaltsvalidierte Wiederherstellungen mit exakter Markierung dienen als Nachweis für den Rollout. Erforderliche CI-Jobs und Pull Requests erhalten kurzlebige Klone, sodass Änderungen an Abhängigkeiten keine neuen Festplatten, konkurrierenden Snapshots oder Cache-Sperren erzeugen, die Builds abbrechen können.
- Node-Shard- und Build-Artefakt-Jobs stellen außerdem den portablen On-Disk-Compile-Cache von Node über unveränderliche Actions-Caches wieder her. Unabhängige Namespaces für `test` und `build` verhindern, dass ihre Schreiber die Archive des jeweils anderen ersetzen: Der geplante Test-Warmer verwaltet den geschützten Test-Seed, während `build-artifacts` höchstens ein geschütztes Build-Archiv pro UTC-Tag aus vertrauenswürdigen `main`-Pushes veröffentlichen darf. PR- und gewöhnliche Test-Jobs lesen nur geschützte Snapshots, sodass Bytecode aus Feature-Branches niemals in den gemeinsamen Seed gelangt und PR-Verkehr keine Cache-Archive erzeugt. Dadurch wird V8-Bytecode für von Node geladene Orchestrierung, Build-Tooling und externe Abhängigkeiten über unterschiedliche Checkout-Pfade hinweg wiederverwendet, auch wenn sich nur ein Teil des Quellgraphen ändert. Vitest-Kindprozesse deaktivieren einen geerbten Compile-Cache, da Coverage innerhalb dynamischer Konfigurationen aktiviert werden kann und die V8-Coverage beim Deserialisieren von Skripten aus Bytecode an Präzision der Quellpositionen verlieren kann.
- Der Build-Artefakt-Job persistiert außerdem inhaltsbezogen fingerprintete Ausgaben des Schritts `build-all`. Die von der CI selbst erstellten Deklarationen des Plugin SDK hashen den vollständigen, dem Repository zugehörigen TypeScript-/JSON-Quellgraphen, schließen installierte und generierte Verzeichnisse aus und stellen sowohl flache Deklarationen als auch Paketbrücken wieder her, nachdem `tsdown` `dist` bereinigt hat. Dokumentations-, Workflow-, Plugin- und andere Änderungen außerhalb dieses Graphen können den Deklarations-Snapshot wiederverwenden; Quelländerungen erstellen ihn neu, bevor das Export-Gate ausgeführt wird.
- Vollständige Deklarations-Builds teilen `tsdown` in KI-, Workspace-Paket- und vereinheitlichte Gruppen auf. Jede Gruppe cached nur Deklarationen, erstellt anschließend jedoch weiterhin das Runtime-JavaScript neu, bevor diese Deklarationen wiederhergestellt werden. Änderungen am Core oder an Plugins machen daher nur den großen vereinheitlichten Graphen ungültig, während Änderungen an Workspace-Paketen konservativ jede abhängige Deklarationsgruppe ungültig machen. Öffentliche vollständige Builds verwenden im Allgemeinen einen unveränderlichen Actions-Cache; grobe Wiederherstellungsschlüssel initialisieren partielle Änderungen, gruppenspezifische Inhaltsfingerabdrücke weisen veraltete Daten zurück und GitHubs Cache-Kontingent entfernt alte Generationen. Die wöchentliche Node-22-Lane veröffentlicht stattdessen nach erfolgreichen `main`-Läufen ein 14-Tage-Artefakt und stellt auf `main` nur Artefakte wieder her, deren unveränderliche Erstelleridentität diesem Workflow zugeordnet werden kann. So wird Kontingentfluktuation vermieden, ohne PR-Code das Schreiben in einen gemeinsamen Cache zu erlauben. Private-QA-Deklarationen werden niemals in Actions-Caches persistiert, da Cache-Namespaces keine Vertraulichkeitsgrenzen darstellen.
- `check-additional-*` verteilt die ergänzende Liste der Boundary-Guards (`scripts/run-additional-boundary-checks.mjs`) auf einen promptintensiven Shard (`check-additional-boundaries-a`, der die Drift-Prüfung für Codex-Prompt-Snapshots enthält) und einen kombinierten Shard für die verbleibenden Streifen (`check-additional-boundaries-bcd`); beide führen unabhängige Guards gleichzeitig aus und geben die Laufzeiten pro Prüfung aus. Compile-/Canary-Arbeiten an Paketgrenzen bleiben zusammen, und die Runtime-Topologiearchitektur wird getrennt von der in `build-artifacts` eingebetteten Gateway-Watch-Coverage ausgeführt.
- Auf dem selbst gehosteten Build-Runner mit 32 vCPU starten Gateway Watch, Channel-Tests und der Core-Support-Boundary-Shard gemeinsam innerhalb von `build-artifacts`, nachdem `dist/` und `dist-runtime/` bereits erstellt wurden. Fallback-Läufe auf von GitHub gehosteten Runnern führen Gateway Watch weiterhin seriell aus, damit Konkurrenz um wenige Kerne dessen Bereitschaftsfrist nicht aufbraucht.

Nach der Zulassung erlaubt die kanonische Linux-CI bis zu 28 gleichzeitig ausgeführte Node-Test-Jobs und
12 für die kleineren schnellen Prüf-Lanes; Windows und Android bleiben bei zwei, weil
diese Runner-Pools kleiner sind. Kompakte Batches vollständiger Konfigurationen werden mit einem
Batch-Timeout von 120 Minuten ausgeführt, während Gruppen mit Include-Mustern dasselbe begrenzte
Job-Budget teilen.

Die Android-CI führt sowohl `testPlayDebugUnitTest` als auch `testThirdPartyDebugUnitTest` aus und erstellt anschließend die Play-Debug-APK. Die Drittanbieter-Variante verfügt weder über einen separaten Quellsatz noch über ein separates Manifest; ihre Unit-Test-Lane kompiliert die Variante weiterhin mit den BuildConfig-Flags für SMS/Anrufprotokoll, vermeidet jedoch bei jedem Android-relevanten Push einen doppelten Packaging-Job für die Debug-APK. Jede aktuelle Gradle-Aufgabe verfügt über eine geschützte Sticky Disk; PR-Jobs verwenden kurzlebige Klone, während geschützte Läufe inhaltsadressierte Gradle-Einträge direkt aktualisieren.

Schlüssel für Blacksmith-Sticky-Disks werden bewusst durch unterstützte Runtime- oder Aufgabendimensionen begrenzt, niemals durch PR-Nummer, Commit, Lauf, Branch oder Abhängigkeits-Hash. Runtime-Transformations- und Compile-Caches verwenden statt Sticky Disks den Actions-Cache, da unveränderliche Archive überprüfbare Wiederherstellungs-/Speicherergebnisse bieten und Fehler bei der Hochstufung veränderlicher Snapshots vermeiden. Fügen Sie nach einer Migration der Sticky-Schlüsselversion nur die exakten veralteten Schlüssel-, Architektur- und Regionsidentitäten zu `.github/retired-sticky-disks.json` hinzu, lösen Sie `Sticky Disk Cleanup` aus `main` mit denselben Dimensionen und derselben Bestätigung aus, überprüfen Sie die Löschung und entfernen Sie anschließend diese Einträge. Der Workflow leitet ARM-Identitäten an einen ARM-Runner weiter, weist Abweichungen zwischen Runner und Region zurück, verwendet Blacksmiths Aktion zur Löschung exakter Schlüssel und löscht niemals Docker-Builder-Caches oder Platzhalterpräfixe. Actions-Cache-Archive verwenden die normale LRU- und Inaktivitätsbereinigung.

Der `check-dependencies`-Shard führt Knip-Produktionsprüfungen für Abhängigkeiten, ungenutzte Dateien und ungenutzte Exporte aus. Der Guard für ungenutzte Dateien schlägt fehl, wenn ein PR eine neue ungeprüfte ungenutzte Datei hinzufügt oder einen veralteten Allowlist-Eintrag hinterlässt, während beabsichtigte dynamische Plugin-, generierte, Build-, Live-Test- und Paketbrücken-Oberflächen erhalten bleiben, die Knip nicht statisch auflösen kann. Der Guard für ungenutzte Exporte schließt Testunterstützungsdateien aus und schlägt bei jedem ungenutzten Produktionsexport fehl; beabsichtigte dynamische Verbraucher müssen in `config/knip.config.ts` modelliert werden. Historische Ziele führen den Export-Guard aus, wenn sie ihn bereitstellen, und behalten andernfalls ihren älteren Dead-Code-Fallback bei.

## Weiterleitung von ClawSweeper-Aktivitäten

`.github/workflows/clawsweeper-dispatch.yml` ist die zielseitige Brücke von Aktivitäten im OpenClaw-Repository zu ClawSweeper. Sie checkt keinen nicht vertrauenswürdigen Pull-Request-Code aus und führt ihn nicht aus. Der Workflow erstellt aus `CLAWSWEEPER_APP_PRIVATE_KEY` ein GitHub-App-Token und sendet anschließend kompakte `repository_dispatch`-Payloads an `openclaw/clawsweeper`.

Der Workflow hat vier Ausführungspfade:

- `clawsweeper_item` für konkrete Anfragen zur Prüfung von Issues und Pull Requests;
- `clawsweeper_comment` für explizite ClawSweeper-Befehle in Issue-Kommentaren;
- `clawsweeper_commit_review` für Prüfungsanfragen auf Commit-Ebene bei `main`-Pushes;
- `github_activity` für allgemeine GitHub-Aktivitäten, die der ClawSweeper-Agent untersuchen kann.

Der Ausführungspfad `github_activity` leitet nur normalisierte Metadaten weiter: Ereignistyp, Aktion, Akteur, Repository, Elementnummer, URL, Titel, Status sowie, sofern vorhanden, kurze Auszüge aus Kommentaren oder Reviews. Der vollständige Webhook-Body wird bewusst nicht weitergeleitet. Der empfangende Workflow in `openclaw/clawsweeper` ist `.github/workflows/github-activity.yml`; er sendet das normalisierte Ereignis an den OpenClaw-Gateway-Hook für den ClawSweeper-Agenten.

Allgemeine Aktivitäten dienen der Beobachtung und werden standardmäßig nicht zugestellt. Der ClawSweeper-Agent erhält das Discord-Ziel in seinem Prompt und sollte nur dann in `#clawsweeper` posten, wenn das Ereignis überraschend, handlungsrelevant, riskant oder betrieblich nützlich ist. Routinemäßiges Öffnen und Bearbeiten, Bot-Aktivität, dupliziertes Webhook-Rauschen und normaler Review-Verkehr sollten zu `NO_REPLY` führen.

Behandeln Sie GitHub-Titel, Kommentare, Bodys, Review-Texte, Branch-Namen und Commit-Nachrichten auf diesem gesamten Pfad als nicht vertrauenswürdige Daten. Sie dienen als Eingabe für Zusammenfassung und Triage, nicht als Anweisungen für den Workflow oder die Agenten-Runtime.

## Manuelle Ausführungen

Manuelle CI-Ausführungen verwenden denselben Job-Graphen wie die normale CI, aktivieren jedoch jeden abgegrenzten Nicht-Android-Ausführungspfad: Linux-Node-Shards, Shards für gebündelte Plugins, Plugin- und Channel-Vertrag-Shards, Node-22-Kompatibilität, `check-*`, `check-additional-*`, Smoke-Tests für erstellte Artefakte, Dokumentationsprüfungen, Python-Skills, Windows, macOS, iOS-Build sowie die Internationalisierung der Control UI und nativen App. Automatische Quell-PRs prüfen das Inventar der nativen Extraktion und die Sicherheit der Android-/Apple-Lokalisierung, ohne im selben PR übersetzte oder plattformgenerierte Ausgaben zu verlangen. Der serialisierte Workflow zur Aktualisierung der Gebietsschemata der nativen App erstellt diese Artefakte in einem isolierten PR neu und aktiviert die automatische Zusammenführung des exakten Heads, nachdem die erforderlichen Prüfungen bestanden wurden. Die vollständige native Parität bleibt für PRs mit generierten Artefakten, manuelle CI, Full Release Validation und die Release-Vorbereitung blockierend. Die Parität der Control-UI-Gebietsschemata bleibt bei automatischen PR- und `main`-Ausführungen informativ und ist bei manueller/Release-CI blockierend. Eigenständige manuelle CI-Ausführungen führen Android nur mit `include_android=true` aus (die Eingabe `release_gate` erzwingt Android ebenfalls); die vollständige Release-Überdachung aktiviert Android durch Übergabe von `include_android=true`. Statische Vorabprüfungen für Plugin-Prereleases, der ausschließlich für Releases vorgesehene `agentic-plugins`-Shard, der vollständige Batch-Durchlauf aller Erweiterungen und die Docker-Ausführungspfade für Plugin-Prereleases sind von der CI ausgeschlossen. Die Docker-Prerelease-Suite wird nur ausgeführt, wenn `Full Release Validation` den separaten Workflow `Plugin Prerelease` mit aktivierter Release-Validierungssperre auslöst.

Die PR-Prüfungen der maximalen Zeilenzahl leiten die Baseline aus dem ausgecheckten synthetischen Merge-Baum ab und prüfen dessen Head-Parent gegen den Ereignis-Head. Manuelle Ausführungen verwenden eine eindeutige Nebenläufigkeitsgruppe, damit eine vollständige Release-Candidate-Suite nicht durch einen weiteren Push oder eine weitere PR-Ausführung auf derselben Referenz abgebrochen wird. Mit der optionalen Eingabe `target_ref` kann ein vertrauenswürdiger Aufrufer diesen Graphen für einen Branch, ein Tag oder eine vollständige Commit-SHA ausführen und dabei die Workflow-Datei aus der ausgewählten Ausführungsreferenz verwenden; die Baseline für die maximale Zeilenzahl wird mit der Merge-Basis des Ziels gegenüber dem für diese Ausführung ermittelten Head des Standard-Branches verglichen. Die Eingabe `release_gate` ist eine Maintainer-Ausweichlösung mit exakter SHA für PR-CI, die aufgrund fehlender Kapazität feststeckt: Sie verlangt, dass `target_ref` eine vollständige Commit-SHA ist, die mit dem Head des ausgeführten Branches übereinstimmt, und dass `pull_request_number` den offenen PR bezeichnet, dessen Merge-Baum validiert wird.

```bash
gh workflow run ci.yml --ref release/YYYY.M.PATCH
gh workflow run ci.yml --ref main -f target_ref=<branch-or-sha> -f include_android=true
gh workflow run full-release-validation.yml --ref main -f ref=<branch-or-sha>
```

Erweiterte stabile Gateway-Ausführungen führen npm-Vorabprüfung, Full Release Validation und Plugin-
npm-Release aus `extended-stable/YYYY.M.33` aus; die Veröffentlichung des Kerns verwendet diese drei
Ausführungs-IDs sowie den Validierungsversuch. Nachweise aus `release-ci/*` sind ungültig, weil
die Veröffentlichung jede Ausführung an den kanonischen Branch und die Release-SHA bindet. Das Tag
veröffentlicht Gateway-Images und nur die `extended-stable*`-Aliasse; der Pfad überspringt
den regulären Orchestrator und dessen ClawHub-, native-App-, GitHub-Release-, Website-
und private Dist-Tag-Oberflächen. Befehle und Wiederherstellung finden Sie unter [Monatliche erweiterte stabile
Gateway-Veröffentlichung](/de/reference/RELEASING#monthly-gateway-extended-stable-publication).

## Runner

| Runner                          | Jobs                                                                                                                                                                                                                                                                              |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ubuntu-24.04`                  | `security-fast`, manuelle CI-Ausführung und Ausweichlösungen für nicht kanonische Repositorys, das QA-Smoke-Aggregat, CodeQL-Sicherheits- und Qualitätsprüfungen, Workflow-Plausibilitätsprüfung, Labeler, automatische Antwort, der eigenständige Docs-Workflow und der gesamte Install-Smoke-Workflow                                |
| `blacksmith-4vcpu-ubuntu-2404`  | `preflight`, `pnpm-store-warmup`, `native-i18n`, `checks-fast-core` außer QA-Smoke-CI, Plugin-/Channel-Vertrag-Shards, die meisten gebündelten/leichteren Linux-Node-Shards, `check-*`-Ausführungspfade außer `check-lint`, ausgewählte `check-additional-*`-Shards, `check-docs` und `skills-python` |
| `blacksmith-8vcpu-ubuntu-2404`  | Beibehaltene umfangreiche Linux-Node-Suites, grenzflächen-/erweiterungsintensive `check-additional-*`-Shards und `android`                                                                                                                                                                             |
| `blacksmith-16vcpu-ubuntu-2404` | Automatische QA-Smoke-CI-Shards, `build-artifacts` in CI und Testbox sowie `check-lint` (so CPU-empfindlich, dass 8 vCPU mehr kosteten, als sie einsparten)                                                                                                                                  |
| `blacksmith-8vcpu-windows-2025` | `checks-windows`                                                                                                                                                                                                                                                                  |
| `blacksmith-6vcpu-macos-15`     | `macos-node` auf `openclaw/openclaw`; Forks weichen auf `macos-15` aus                                                                                                                                                                                                                |
| `blacksmith-12vcpu-macos-26`    | `macos-swift` und `ios-build` auf `openclaw/openclaw`; Forks weichen auf `macos-26` aus                                                                                                                                                                                               |

## Budget für Runner-Registrierungen

Der aktuelle GitHub-Bucket für Runner-Registrierungen von OpenClaw weist in
`ghx api rate_limit` 10,000 Registrierungen selbst gehosteter Runner pro 5 Minuten aus. Prüfen Sie
`actions_runner_registration` vor jeder Optimierungsrunde erneut, da GitHub
diesen Bucket ändern kann. Das Limit wird von allen Blacksmith-Runner-Registrierungen in der
Organisation `openclaw` gemeinsam genutzt; die Installation einer weiteren Blacksmith-Instanz fügt daher
keinen neuen Bucket hinzu.

Behandeln Sie Blacksmith-Labels als knappe Ressource zur Kontrolle von Lastspitzen. Jobs, die
nur weiterleiten, benachrichtigen, zusammenfassen, Shards auswählen oder kurze CodeQL-Prüfungen ausführen, sollten
auf von GitHub gehosteten Runnern verbleiben, sofern für sie kein gemessener Blacksmith-spezifischer
Bedarf besteht. Jede neue Blacksmith-Matrix, ein größeres `max-parallel` oder ein hochfrequenter
Workflow muss seine maximale Registrierungszahl im ungünstigsten Fall ausweisen und das organisationsweite
Ziel unter etwa 60% des aktuellen Buckets halten. Beim aktuellen Bucket mit 10,000 Registrierungen
entspricht dies einem Betriebsziel von 6,000 Registrierungen, sodass Spielraum für
gleichzeitige Repositorys, Wiederholungsversuche und sich überschneidende Lastspitzen bleibt.

Der PR-Plan für geänderte Ziele reduziert die übliche Node-Testlastspitze von 14 Blacksmith-Registrierungen auf eine. PRs mit breitem Risiko behalten die kompakte Ausweichlösung mit 14 Registrierungen bei, sodass sich der ungünstigste Fall nicht verschärft.

Die CI des kanonischen Repositorys verwendet Blacksmith weiterhin als standardmäßigen Runner-Pfad für normale Push- und Pull-Request-Ausführungen. `workflow_dispatch` und Ausführungen in nicht kanonischen Repositorys verwenden von GitHub gehostete Runner; normale kanonische Ausführungen prüfen derzeit jedoch weder den Zustand der Blacksmith-Warteschlange noch weichen sie bei Nichtverfügbarkeit von Blacksmith automatisch auf von GitHub gehostete Labels aus.

## Oberflächen-Ratchets

Zwei ausschließlich reduzierbare Budgets schützen die Konfigurationsoberfläche. Bei Wachstum lassen beide die CI
fehlschlagen, bis die Budgetdatei im selben PR bewusst aktualisiert wird, und beide verlangen eine
Absenkung des Ratchets, wenn eine Bereinigung die tatsächliche Anzahl reduziert.

- `config/env-var-count-budget.txt` begrenzt die Anzahl unterschiedlicher `OPENCLAW_*`-
  Namen im Produktionsquellcode unter `src/`, `packages/` und `extensions/`
  (Tests und QA Lab ausgeschlossen). Geprüft durch `node scripts/check-env-var-count.mjs`.
  Beim Entfernen von Umgebungsvariablen: Senken Sie die Zahl im selben PR. Das Hinzufügen einer Variable ist eine
  Entscheidung über die Konfigurationsoberfläche – begründen Sie sie im PR-Body.
- `docs/.generated/config-baseline.counts.json` begrenzt je Art
  (Kern/Channel/Plugin) die Anzahl der `openclaw.json`-Schemaeinträge. Geprüft durch
  `pnpm config:docs:check`; nach jeder Schemaänderung mit `pnpm config:docs:gen`
  neu generieren.

## Lokale Entsprechungen

```bash
pnpm changed:lanes                            # den lokalen Klassifizierer für geänderte Lanes für origin/main...HEAD prüfen
pnpm check:changed                            # intelligentes lokales Prüf-Gate: geänderte Formatierung/Typprüfung/Linting/Guards nach Boundary-Lane
pnpm check                                    # schnelles lokales Gate: Produktions-tsgo + aufgeteiltes Linting + parallele schnelle Guards
pnpm check:test-types
pnpm check:timed                              # dasselbe Gate mit Zeitmessungen pro Phase
pnpm build:strict-smoke
pnpm check:architecture
pnpm test:gateway:watch-regression
OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1 node scripts/run-vitest.mjs run --config test/vitest/vitest.tui-pty.config.ts
pnpm test                                     # Vitest-Tests
pnpm test:changed                             # kostengünstige intelligente Vitest-Ziele für Änderungen
pnpm test:ui                                  # Unit-/Browser-Suite der Control UI
pnpm ui:i18n:check                            # generierte Gebietsschema-Parität der Control UI (Release-Gate)
pnpm native:i18n:baseline                     # quellcodeverwaltetes Inventar der nativen Extraktion aktualisieren
pnpm native:i18n:verify                       # Quellinventar + Lokalisierungssicherheit für Android/Apple
pnpm native:i18n:check                        # strikte Parität der Übersetzungen und plattformgenerierten Inhalte (Release-Gate)
pnpm test:channels
pnpm test:contracts:channels
pnpm check:docs                               # Dokumentformatierung + Linting + defekte Links
pnpm build                                    # Distribution erstellen, wenn CI-Artefakt-/Smoke-Prüfungen relevant sind
pnpm ios:build                                # iOS-App-Projekt generieren und erstellen
pnpm ci:timings                               # neuesten origin/main-Push-CI-Lauf zusammenfassen
pnpm ci:timings:recent                        # kürzlich erfolgreiche Main-CI-Läufe vergleichen
node scripts/ci-run-timings.mjs <run-id>      # Gesamtdauer, Warteschlangenzeit und langsamste Jobs zusammenfassen
node scripts/ci-run-timings.mjs --latest-main # Störungen durch Issues/Kommentare ignorieren und origin/main-Push-CI auswählen
node scripts/ci-run-timings.mjs --recent 10   # kürzlich erfolgreiche Main-CI-Läufe vergleichen
pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json
pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json
pnpm test:startup:memory
pnpm test:extensions:memory -- --json .artifacts/openclaw-performance/source/mock-provider/extension-memory.json
pnpm perf:kova:summary --report .artifacts/kova/reports/mock-provider/report.json --output .artifacts/kova/summary.md
```

## OpenClaw-Performance

`OpenClaw Performance` ist der Workflow für die Produkt-/Runtime-Performance. Er wird täglich auf `main` ausgeführt und kann manuell gestartet werden:

```bash
gh workflow run openclaw-performance.yml --ref main -f profile=diagnostic -f repeat=3
gh workflow run openclaw-performance.yml --ref main -f profile=smoke -f repeat=1 -f deep_profile=true -f live_openai_candidate=true
gh workflow run openclaw-performance.yml --ref main -f target_ref=v2026.5.2 -f profile=diagnostic -f repeat=3
```

Beim manuellen Start wird normalerweise die Workflow-Referenz einem Benchmark unterzogen. Setzen Sie `target_ref`, um ein Release-Tag oder einen anderen Branch mit der aktuellen Workflow-Implementierung einem Benchmark zu unterziehen. Veröffentlichte Berichtspfade und Zeiger auf die neueste Version werden nach der getesteten Referenz unterschieden, und jeder `index.md` zeichnet die getestete Referenz/SHA, die Workflow-Referenz/SHA, die Kova-Referenz, das Profil, den Lane-Authentifizierungsmodus, das Modell, die Anzahl der Wiederholungen und die Szenariofilter auf.

Der Workflow installiert OCM aus einem festgelegten Release und Kova aus `openclaw/Kova` mit der festgelegten `kova_ref`-Eingabe und führt anschließend drei Lanes aus:

- `mock-provider`: Kova-Diagnoseszenarien für eine lokal erstellte Runtime mit deterministischer, simulierter OpenAI-kompatibler Authentifizierung.
- `mock-deep-profile`: CPU-/Heap-/Trace-Profiling für Hotspots beim Start, im Gateway und bei Agent-Turns. Wird nach Zeitplan oder bei manuellem Start mit `deep_profile=true` ausgeführt.
- `live-openai-candidate`: ein echter OpenAI-`openai/gpt-5.6-luna`-Agent-Turn, der übersprungen wird, wenn `OPENAI_API_KEY` nicht verfügbar ist. Wird nach Zeitplan oder bei manuellem Start mit `live_openai_candidate=true` ausgeführt.

Die Mock-Provider-Lane führt nach dem Kova-Durchlauf außerdem OpenClaw-native Quellcode-Probes aus: Gateway-Startzeit und Arbeitsspeicher für Startfälle mit Standardeinstellungen, übersprungenem Channel, internem Hook und fünfzig Plugins; RSS beim Import gebündelter Plugins, wiederholte Mock-OpenAI-`channel-chat-baseline`-Begrüßungsschleifen, CLI-Startbefehle für das gestartete Gateway und die Smoke-Performance-Probe für den SQLite-Zustand. Wenn der zuvor veröffentlichte Mock-Provider-Quellbericht für die getestete Referenz verfügbar ist, vergleicht die Quellzusammenfassung die aktuellen RSS- und Heap-Werte mit dieser Baseline und kennzeichnet große RSS-Anstiege als `watch`. Die Markdown-Zusammenfassung der Quell-Probe befindet sich unter `source/index.md` im Berichtspaket; die JSON-Rohdaten liegen daneben.

Jede Lane lädt ihr vollständiges GitHub-Artefakt hoch, einschließlich CPU-, Heap-, Trace- und komprimierter Diagnosepakete. Ein separater Publisher-Job lädt diese Artefakte herunter und validiert sie. Anschließend erstellt er ein kurzlebiges GitHub-App-Token für ClawSweeper, das ausschließlich auf Inhalte von `openclaw/clawgrit-reports` beschränkt ist, und übergibt es ausschließlich an den Git-Push-Schritt. Er committet `report.json`, `report.md`, `index.md`, Quell-Probe-Artefakte und Paketmetadaten/-Prüfsummen unter `openclaw-performance/<tested-ref>/<run-id>-<attempt>/<lane>/`; das vollständige Diagnosearchiv verbleibt im verknüpften Actions-Artefakt. Der Publisher lehnt jede Berichtsdatei mit mehr als 50 MB ab, bevor er einen Push versucht. Der aktuelle Zeiger für die getestete Referenz ist `openclaw-performance/<tested-ref>/latest-<lane>.json`. Geplante Läufe und mit `profile=release` gestartete Läufe schlagen fehl, wenn die Erstellung des App-Tokens oder die Veröffentlichung des Berichts fehlschlägt. Bei manuellen Starts ohne Release ist die Veröffentlichung nur informativ; die GitHub-Artefakte bleiben erhalten, wenn die Authentifizierung oder Veröffentlichung fehlschlägt. Die vorherige Quell-Baseline wird anonym aus dem öffentlichen Berichts-Repository abgerufen. Ein erfolgreicher Abruf der Baseline belegt daher keine Publisher-Authentifizierung.

## Vollständige Release-Validierung

`Full Release Validation` ist der manuelle übergeordnete Workflow für „vor dem Release alles ausführen“. Er akzeptiert einen Branch, ein Tag oder einen vollständigen Commit-SHA, startet den manuellen Workflow `CI` mit diesem Ziel (einschließlich Android), startet `Plugin Prerelease` für ausschließlich releasebezogene Plugin-/Paket-/statische/Docker-Nachweise, startet `OpenClaw Performance` für den Ziel-SHA und startet `OpenClaw Release Checks` für Installations-Smoke-Tests, Paketabnahme, betriebssystemübergreifende Paketprüfungen, QA-Lab-Parität, Matrix, Telegram sowie durch Gates geschützte Discord-, WhatsApp- und Slack-Lanes (das informative Rendering der Reifegrad-Scorecard kann über `run_maturity_scorecard` aktiviert werden). Stable- und Full-Profile enthalten immer umfassende Live-/E2E- sowie Docker-Soak-Abdeckung für den Release-Pfad; beim Beta-Profil kann diese mit `run_release_soak=true` aktiviert werden. Der kanonische Telegram-E2E-Test für das Paket wird innerhalb der Paketabnahme ausgeführt, sodass ein vollständiger Kandidat keinen doppelten Live-Poller startet. Übergeben Sie nach der Veröffentlichung `release_package_spec`, um das ausgelieferte npm-Paket für Release-Prüfungen, Paketabnahme, Docker, betriebssystemübergreifende Prüfungen und Telegram wiederzuverwenden, ohne es neu zu erstellen. Verwenden Sie `npm_telegram_package_spec` nur für eine gezielte erneute Ausführung von Telegram mit dem veröffentlichten Paket. Die Live-Paket-Lane des Codex-Plugins verwendet standardmäßig denselben ausgewählten Zustand: Bei einer veröffentlichten Version leitet `release_package_spec=openclaw@<tag>` den Wert `codex_plugin_spec=npm:@openclaw/codex@<tag>` ab, während SHA-/Artefaktläufe `extensions/codex` aus der ausgewählten Referenz packen. Setzen Sie `codex_plugin_spec` explizit für benutzerdefinierte Plugin-Quellen wie `npm:`-, `npm-pack:`- oder `git:`-Spezifikationen. Der Live-Agent-Nachweis sendet sichtbare Fortschrittsmeldungen, setzt den Ablauf mit zufälligen Workspace-Lesevorgängen und dem exakten Schreiben eines Artefakts fort und sendet anschließend die Abschlussmeldung.

Unter [Vollständige Release-Validierung](/de/reference/full-release-validation) finden Sie die
Phasenmatrix, die exakten Workflow-Jobnamen, Profilunterschiede, Artefakte und
Optionen für gezielte erneute Ausführungen.

`OpenClaw Release Publish` ist der manuelle zustandsverändernde Release-Workflow. Starten Sie
reguläre Beta- und Stable-Veröffentlichungen vom vertrauenswürdigen `main`, nachdem das Release-Tag
vorhanden ist und der npm-Preflight für OpenClaw erfolgreich war (der Preflight führt
unter anderem `pnpm plugins:sync:check` aus). Das Tag wählt weiterhin den exakten
Release-Commit aus, einschließlich eines Commits auf `release/YYYY.M.PATCH`; Tideclaw-Alpha-
Veröffentlichungen verwenden weiterhin ihren jeweils passenden Alpha-Branch. Er erfordert das gespeicherte
`preflight_run_id` sowie einen erfolgreichen
`full_release_validation_run_id` und dessen exaktes
`full_release_validation_run_attempt`, startet `Plugin NPM Release` für alle
veröffentlichbaren Plugin-Pakete, startet `Plugin ClawHub Release` für denselben
Release-SHA und startet erst danach `OpenClaw NPM Release`. Die Stable-Veröffentlichung
erfordert außerdem ein exaktes `windows_node_tag`; der Workflow verifiziert das Windows-Quell-
Release und vergleicht dessen x64-/ARM64-Installationsprogramme vor jedem untergeordneten Veröffentlichungsworkflow mit der vom Kandidaten genehmigten
`windows_node_installer_digests`-Eingabe. Anschließend bewirbt
und verifiziert er dieselben festgelegten Installationsprogramm-Digests sowie den exakten Vertrag für das Begleitartefakt
und die Prüfsummen, bevor der GitHub-Release-Entwurf veröffentlicht wird.
Gezielte Reparaturen ausschließlich an Plugins verwenden `plugin_publish_scope=selected` mit einer nicht leeren
Paketliste. Ausschließlich Plugins betreffende `all-publishable`-Läufe erfordern dieselben unveränderlichen npm-
Preflight- und vollständigen Release-Validierungsnachweise wie eine Core-Veröffentlichung.

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH-beta.N \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

Verwenden Sie für einen festgelegten Commit-Nachweis auf einem schnell veränderlichen Branch den Helfer anstelle von
`gh workflow run ... --ref main -f ref=<sha>`:

```bash
pnpm ci:full-release --sha <full-sha>
```

Referenzen zum Starten von GitHub-Workflows müssen Branches oder Tags sein, keine rohen Commit-SHAs. Der
Helfer pusht einen temporären `release-ci/<sha>-...`-Branch an einem vertrauenswürdigen `main`-
Workflow-SHA, übergibt den angeforderten Ziel-SHA über die Workflow-Eingabe `ref`,
verwendet strikte Nachweise für das exakte Ziel wieder, sofern verfügbar, verifiziert, dass bei jedem untergeordneten
Workflow `headSha` dem vertrauenswürdigen Workflow-SHA entspricht, und löscht den temporären
Branch nach Abschluss des Laufs. Übergeben Sie `-f reuse_evidence=false`, um eine neue
Validierung zu erzwingen. Der übergeordnete Verifizierer schlägt außerdem fehl, wenn ein untergeordneter Workflow mit einem
anderen Workflow-SHA ausgeführt wurde.

`release_profile` steuert die an die Release-Prüfungen übergebene Live-/Provider-Breite. Die
manuellen Release-Workflows verwenden standardmäßig `stable`; verwenden Sie `full` nur, wenn Sie
bewusst die breite informative Provider-/Medienmatrix ausführen möchten. Stable- und vollständige
Release-Prüfungen führen immer die umfassenden Live-/E2E- und Docker-Soak-Tests für den Release-Pfad aus;
beim Beta-Profil kann dies mit `run_release_soak=true` aktiviert werden.

- `beta` behält die schnellsten releasekritischen OpenAI-/Core-Lanes bei.
- `stable` ergänzt den stabilen Provider-/Backend-Satz.
- `full` führt die breite informative Provider-/Medienmatrix aus.

Der übergeordnete Workflow zeichnet die IDs der gestarteten untergeordneten Läufe auf, und der abschließende Job `Verify full validation` prüft die aktuellen Ergebnisse der untergeordneten Läufe erneut und fügt für jeden untergeordneten Lauf Tabellen der langsamsten Jobs an. Wenn ein untergeordneter Workflow erneut ausgeführt wird und erfolgreich wird, führen Sie nur den übergeordneten Verifizierungsjob erneut aus, um das Gesamtergebnis und die Zeitübersicht zu aktualisieren.

Zur Wiederherstellung akzeptieren sowohl `Full Release Validation` als auch `OpenClaw Release Checks` den Wert `rerun_group`. Verwenden Sie `all` für einen Release Candidate, `ci` nur für den normalen vollständigen CI-Unterworkflow, `plugin-prerelease` nur für den Plugin-Prerelease-Unterworkflow, `performance` nur für den OpenClaw-Performance-Unterworkflow, `release-checks` für jeden Release-Unterworkflow oder eine engere Gruppe: `install-smoke`, `cross-os`, `live-e2e`, `package`, `qa`, `qa-parity`, `qa-live` oder `npm-telegram` im übergeordneten Workflow. Dadurch bleibt die erneute Ausführung einer fehlgeschlagenen Release-Box nach einer gezielten Korrektur begrenzt. Kombinieren Sie für eine einzelne fehlgeschlagene plattformübergreifende Lane `rerun_group=cross-os` mit `cross_os_suite_filter`, beispielsweise `windows/packaged-upgrade`; lange plattformübergreifende Befehle geben Heartbeat-Zeilen aus, und Zusammenfassungen paketierter Upgrades enthalten Zeitangaben pro Phase. Ausgewählte Matrix- und Telegram-QA-Lanes blockieren die normale Release-Validierung, ebenso das Abdeckungsgate für das Tool des Core-Runtime-Paars. QA-Parität, Runtime-Parität sowie die gegateten Live-Lanes für Discord, WhatsApp und Slack haben nur Hinweischarakter.

`OpenClaw Release Checks` verwendet die vertrauenswürdige Workflow-Referenz, um die ausgewählte Referenz einmalig in ein `release-package-under-test`-Tarball aufzulösen, und übergibt dieses Artefakt anschließend an die plattformübergreifenden Prüfungen und die Paketakzeptanz sowie bei ausgeführter Soak-Abdeckung an den Docker-Workflow für den Live-/E2E-Release-Pfad. Dadurch bleiben die Paketbytes über alle Release-Boxen hinweg konsistent, und derselbe Kandidat muss nicht in mehreren Unterjobs erneut gepackt werden. Für die Live-Lane des Codex-npm-Plugins übergeben die Release-Prüfungen entweder eine passende veröffentlichte Plugin-Spezifikation, die aus `release_package_spec` abgeleitet wird, die vom Operator angegebene `codex_plugin_spec` oder eine leere Eingabe, damit das Docker-Skript das Codex-Plugin des ausgewählten Checkouts packt.

Doppelte `Full Release Validation`-Ausführungen für `ref=main` und `rerun_group=all`
ersetzen den älteren übergeordneten Workflow. Der übergeordnete Monitor bricht jeden bereits
gestarteten Unterworkflow ab, wenn der übergeordnete Workflow abgebrochen wird, sodass eine neuere
Validierung von main nicht hinter einer veralteten zweistündigen Release-Prüfung warten muss.
Bei der Validierung von Release-Branches/-Tags und gezielten Gruppen für erneute Ausführungen bleibt
`cancel-in-progress: false` erhalten.

## Live- und E2E-Shards

Der Live-/E2E-Unterworkflow des Releases behält die breite native `pnpm test:live`-Abdeckung bei, führt sie jedoch über `scripts/test-live-shard.mjs` als benannte Shards statt als einzelnen seriellen Job aus:

- `native-live-src-agents` und `native-live-src-agents-zai-coding`
- `native-live-src-gateway-core`
- nach Provider gefilterte `native-live-src-gateway-profiles`-Jobs
- `native-live-src-gateway-backends`
- `native-live-src-infra`
- `native-live-test`
- `native-live-extensions-a-k`
- `native-live-extensions-l-n`
- `native-live-extensions-moonshot`
- `native-live-extensions-openai`
- `native-live-extensions-o-z-other`
- `native-live-extensions-xai`
- getrennte Medien-Shards für Audio/Video und nach Provider gefilterte Musik-Shards

Dadurch bleibt dieselbe Datei-Abdeckung erhalten, während sich Fehler langsamer Live-Provider leichter erneut ausführen und diagnostizieren lassen. Die aggregierten Shard-Namen `native-live-src-gateway`, `native-live-extensions-o-z`, `native-live-extensions-media` und `native-live-extensions-media-music` bleiben für manuelle einmalige erneute Ausführungen gültig.

Die nativen Live-Medien-Shards werden in `ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04` ausgeführt, das vom Workflow `Live Media Runner Image` erstellt wird. Dieses Image installiert `ffmpeg` und `ffprobe` vorab; Medien-Jobs prüfen vor der Einrichtung lediglich die Binärdateien. Docker-gestützte Live-Suites müssen auf normalen Blacksmith-Runnern verbleiben – Container-Jobs sind der falsche Ort, um verschachtelte Docker-Tests zu starten.

Docker-gestützte Live-Modell-/Backend-Shards verwenden pro ausgewähltem Commit ein separates gemeinsames `ghcr.io/openclaw/openclaw-live-test:<sha>-<extensions>`-Image. Der Live-Release-Workflow erstellt und veröffentlicht dieses Image einmal; anschließend werden die Shards für das Docker-Live-Modell, das nach Provider geshardete Gateway, das CLI-Backend, die ACP-Bindung und das Codex-Harness mit `OPENCLAW_SKIP_DOCKER_BUILD=1` ausgeführt. Gateway-Docker-Shards enthalten unterhalb des Workflow-Job-Timeouts explizite `timeout`-Begrenzungen auf Skriptebene, damit ein hängender Container oder Bereinigungspfad schnell fehlschlägt, statt das gesamte Budget der Release-Prüfung zu verbrauchen. Wenn diese Shards das vollständige Quell-Docker-Ziel unabhängig voneinander neu erstellen, ist die Release-Ausführung falsch konfiguriert und verschwendet Laufzeit durch doppelte Image-Builds.

## Paketakzeptanz

Verwenden Sie `Package Acceptance`, wenn die Frage lautet: „Funktioniert dieses installierbare OpenClaw-Paket als Produkt?“ Dies unterscheidet sich von der normalen CI: Die normale CI validiert den Quellbaum, während die Paketakzeptanz ein einzelnes Tarball über dasselbe Docker-E2E-Harness validiert, das Benutzer nach einer Installation oder Aktualisierung verwenden.

### Jobs

1. `resolve_package` checkt `workflow_ref` aus, löst einen Paketkandidaten auf, schreibt `.artifacts/docker-e2e-package/openclaw-current.tgz`, schreibt `.artifacts/docker-e2e-package/package-candidate.json`, lädt beide als Artefakt `package-under-test` hoch und gibt Quelle, Workflow-Referenz, Paketreferenz, Version, SHA-256 und Profil in der GitHub-Schrittzusammenfassung aus.
2. `package_integrity` lädt das Artefakt `package-under-test` herunter und erzwingt mit `scripts/check-openclaw-package-tarball.mjs` den Vertrag für öffentliche Paket-Tarballs.
3. `docker_acceptance` ruft `openclaw-live-and-e2e-checks-reusable.yml` mit dem aufgelösten Quell-SHA des Pakets (mit Rückfall auf `workflow_ref`) und `package_artifact_name=package-under-test` auf. Der wiederverwendbare Workflow lädt dieses Artefakt herunter, validiert den Tarball-Inhalt, bereitet bei Bedarf Paket-Digest-Docker-Images vor und führt die ausgewählten Docker-Lanes gegen dieses Paket aus, statt den Workflow-Checkout zu packen. Wenn ein Profil mehrere gezielte `docker_lanes` auswählt, bereitet der wiederverwendbare Workflow das Paket und die gemeinsamen Images einmal vor und verteilt diese Lanes anschließend als parallele gezielte Docker-Jobs mit eindeutigen Artefakten.
4. `package_telegram` ruft optional `NPM Telegram Beta E2E` auf. Der Job wird ausgeführt, wenn `telegram_mode` nicht `none` ist, und installiert dasselbe Artefakt `package-under-test`, wenn die Paketakzeptanz eines aufgelöst hat; bei einer eigenständigen Telegram-Ausführung kann weiterhin eine veröffentlichte npm-Spezifikation installiert werden.
5. `summary` lässt den Workflow fehlschlagen, wenn die Paketauflösung, Integrität, Docker-Akzeptanz oder die optionale Telegram-Lane fehlgeschlagen ist. Die Eingabe `advisory` stuft Akzeptanzfehler für aufrufende Workflows mit Hinweischarakter zu Warnungen herab.

### Kandidatenquellen

- `source=npm` akzeptiert ausschließlich `openclaw@extended-stable`, `openclaw@beta`, `openclaw@latest` oder eine exakte OpenClaw-Release-Version wie `openclaw@2026.4.27-beta.2`. Verwenden Sie dies für die Akzeptanz veröffentlichter Extended-Stable-, Prerelease- oder Stable-Versionen.
- `source=ref` packt einen vertrauenswürdigen `package_ref`-Branch, ein Tag oder einen vollständigen Commit-SHA. Der Resolver ruft OpenClaw-Branches/-Tags ab, prüft, ob der ausgewählte Commit aus dem Branch-Verlauf des Repositorys oder über ein Release-Tag erreichbar ist, installiert Abhängigkeiten in einem abgetrennten Worktree und packt ihn mit `scripts/package-openclaw-for-docker.mjs`.
- `source=url` lädt ein öffentliches HTTPS-`.tgz` herunter; `package_sha256` ist erforderlich. Dieser Pfad lehnt URL-Anmeldedaten, vom Standard abweichende HTTPS-Ports, private/interne/für Sonderzwecke reservierte Hostnamen oder aufgelöste IP-Adressen sowie Weiterleitungen außerhalb derselben öffentlichen Sicherheitsrichtlinie ab.
- `source=trusted-url` lädt ein HTTPS-`.tgz` aus einer benannten Richtlinie für vertrauenswürdige Quellen in `.github/package-trusted-sources.json` herunter; `package_sha256` und `trusted_source_id` sind erforderlich. Verwenden Sie dies ausschließlich für von Maintainern verwaltete Unternehmens-Mirrors oder private Paket-Repositorys, die konfigurierte Hosts, Ports, Pfadpräfixe, Weiterleitungs-Hosts oder private Netzwerkauflösung benötigen. Wenn die Richtlinie Bearer-Authentifizierung festlegt, verwendet der Workflow das feste Secret `OPENCLAW_TRUSTED_PACKAGE_TOKEN`; in URLs eingebettete Anmeldedaten werden weiterhin abgelehnt.
- `source=artifact` lädt ein `.tgz` aus `artifact_run_id` und `artifact_name` herunter; `package_sha256` ist optional, sollte jedoch für extern freigegebene Artefakte angegeben werden.

Halten Sie `workflow_ref` und `package_ref` getrennt. `workflow_ref` ist der vertrauenswürdige Workflow-/Harness-Code, der den Test ausführt. `package_ref` ist der Quell-Commit, der gepackt wird, wenn `source=ref`. Dadurch kann das aktuelle Test-Harness ältere vertrauenswürdige Quell-Commits validieren, ohne alte Workflow-Logik auszuführen.

### Suite-Profile

- `smoke` — `npm-onboard-channel-agent`, `gateway-network`, `config-reload`
- `package` — `npm-onboard-channel-agent`, `doctor-switch`, `update-channel-switch`, `skill-install`, `update-corrupt-plugin`, `upgrade-survivor`, `published-upgrade-survivor`, `root-managed-vps-upgrade`, `update-restart-auth`, `plugins-offline`, `plugin-update`
- `product` — die `package`-Gruppe mit Live-`plugins`-Abdeckung anstelle von `plugins-offline`, zusätzlich `mcp-channels`, `cron-mcp-cleanup`, `openai-web-search-minimal`, `openwebui`
- `full` — vollständige Docker-Blöcke des Release-Pfads mit OpenWebUI
- `custom` — exakt `docker_lanes`; erforderlich, wenn `suite_profile=custom`

Das Profil `package` verwendet Offline-Plugin-Abdeckung, damit die Validierung veröffentlichter Pakete nicht von der Live-Verfügbarkeit von ClawHub abhängt. Die optionale Telegram-Lane verwendet das Artefakt `package-under-test` in `NPM Telegram Beta E2E` erneut; der Pfad für veröffentlichte npm-Spezifikationen bleibt für eigenständige Ausführungen erhalten.

Die spezielle Richtlinie für Aktualisierungs- und Plugin-Tests einschließlich lokaler Befehle,
Docker-Lanes, Eingaben für die Paketakzeptanz, Release-Standardwerte und Fehleranalyse
finden Sie unter [Testen von Aktualisierungen und Plugins](/de/help/testing-updates-plugins).

Release-Prüfungen rufen die Paketakzeptanz mit `source=artifact`, dem vorbereiteten Release-Paketartefakt, `suite_profile=custom`, `docker_lanes='doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape'` und `telegram_mode=mock-openai` auf. Dadurch werden Paketmigration, Aktualisierung, Live-Installation von ClawHub-Skills, Bereinigung veralteter Plugin-Abhängigkeiten, Reparatur der Installation konfigurierter Plugins, Offline-Plugin-, Plugin-Aktualisierungs- und Telegram-Nachweise auf demselben aufgelösten Paket-Tarball ausgeführt. Setzen Sie nach der Veröffentlichung einer Beta `release_package_spec` bei der vollständigen Release-Validierung oder den OpenClaw-Release-Prüfungen, um dieselbe Matrix gegen das ausgelieferte npm-Paket auszuführen, ohne es neu zu erstellen; setzen Sie `package_acceptance_package_spec` nur, wenn die Paketakzeptanz ein anderes Paket als die übrige Release-Validierung benötigt. Plattformübergreifende Release-Prüfungen decken weiterhin betriebssystemspezifisches Onboarding sowie Installationsprogramm- und Plattformverhalten ab; die Produktvalidierung für Pakete und Aktualisierungen sollte mit der Paketakzeptanz beginnen.

Die Docker-Lane `published-upgrade-survivor` validiert pro Ausführung eine veröffentlichte Paket-Baseline im blockierenden Release-Pfad. In der Paketakzeptanz ist das aufgelöste `package-under-test`-Tarball immer der Kandidat, und `published_upgrade_survivor_baseline` wählt die veröffentlichte Rückfall-Baseline aus, standardmäßig `openclaw@latest`; Befehle zur erneuten Ausführung fehlgeschlagener Lanes behalten diese Baseline bei. Die vollständige Release-Validierung mit `run_release_soak=true` oder `release_profile=full` setzt `published_upgrade_survivor_baselines='last-stable-4 2026.4.23 2026.5.2 2026.4.15'` und `published_upgrade_survivor_scenarios=reported-issues`, um die vier neuesten stabilen npm-Releases sowie angeheftete Releases an Plugin-Kompatibilitätsgrenzen und an konkreten Problemen orientierte Fixtures für die Feishu-Konfiguration, beibehaltene Bootstrap-/Persona-Dateien, konfigurierte OpenClaw-Plugin-Installationen, Tilde-Protokollpfade und veraltete Wurzeln von Legacy-Plugin-Abhängigkeiten abzudecken. Survivor-Auswahlen für veröffentlichte Upgrades mit mehreren Baselines werden nach Baseline in separate gezielte Docker-Runner-Jobs geshardet. Der separate Workflow `Update Migration` verwendet die Docker-Lane `update-migration` mit `all-since-2026.4.23`-Baselines und `plugin-deps-cleanup`-Szenarien, wenn eine umfassende Bereinigung veröffentlichter Aktualisierungen und nicht die normale Breite der vollständigen Release-CI gefragt ist. Lokale aggregierte Ausführungen können mit `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` exakte Paketspezifikationen übergeben, mit `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC` eine einzelne Lane wie `openclaw@2026.4.15` beibehalten oder `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS` für die Szenariomatrix setzen. Die veröffentlichte Lane konfiguriert die Baseline mit einem integrierten `openclaw config set`-Befehlsrezept, zeichnet Rezeptschritte in `summary.json` auf und prüft nach dem Start des Gateways `/healthz`, `/readyz` sowie den RPC-Status. Die Lanes für eine frische paketierte Windows-Installation und eine frische Windows-Installation über das Installationsprogramm prüfen außerdem, ob ein installiertes Paket eine Browsersteuerungs-Überschreibung aus einem absoluten Windows-Rohpfad importieren kann. Der plattformübergreifende OpenAI-Smoke-Test für Agent-Turns verwendet standardmäßig `OPENCLAW_CROSS_OS_OPENAI_MODEL`, wenn dieser Wert gesetzt ist, andernfalls `openai/gpt-5.6-luna`, sodass der Installations- und Gateway-Nachweis die kostengünstigere GPT-5.6-Teststufe verwendet.

### Zeitfenster für Legacy-Kompatibilität

Package Acceptance verfügt über begrenzte Legacy-Kompatibilitätsfenster für bereits veröffentlichte Pakete. Pakete bis einschließlich `2026.4.25`, einschließlich `2026.4.25-beta.*`, dürfen den Kompatibilitätspfad verwenden:

- Bekannte private QA-Einträge in `dist/postinstall-inventory.json` dürfen auf Dateien verweisen, die im Tarball fehlen;
- `doctor-switch` darf den Persistenz-Unterfall `gateway install --wrapper` überspringen, wenn das Paket dieses Flag nicht bereitstellt;
- `update-channel-switch` darf fehlende pnpm-`patchedDependencies` aus dem vom Tarball abgeleiteten simulierten Git-Fixture entfernen und fehlende persistierte `update.channel` protokollieren;
- Plugin-Smoke-Tests dürfen Legacy-Speicherorte für Installationsdatensätze lesen oder eine fehlende Persistenz von Marketplace-Installationsdatensätzen akzeptieren;
- `plugin-update` darf die Migration von Konfigurationsmetadaten zulassen, muss aber weiterhin gewährleisten, dass der Installationsdatensatz und das Verhalten ohne Neuinstallation unverändert bleiben.

Das veröffentlichte Paket `2026.4.26` darf außerdem bei bereits ausgelieferten lokalen Build-Metadaten-Stempeldateien warnen, und Pakete bis einschließlich `2026.5.20` dürfen bei fehlendem `npm-shrinkwrap.json` warnen, statt fehlzuschlagen. Spätere Pakete müssen die modernen Verträge erfüllen; unter denselben Bedingungen schlagen sie fehl, statt zu warnen oder Schritte zu überspringen.

### Beispiele

```bash
# Das aktuelle Beta-Paket mit Abdeckung auf Produktebene validieren.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai

# Das veröffentlichte Extended-Stable-Paket mit Paketabdeckung validieren.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@extended-stable \
  -f suite_profile=package \
  -f telegram_mode=mock-openai

# Einen Release-Branch mit dem aktuellen Test-Harness packen und validieren.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=ref \
  -f package_ref=release/YYYY.M.PATCH \
  -f suite_profile=package \
  -f telegram_mode=mock-openai

# Eine Tarball-URL validieren. SHA-256 ist für source=url obligatorisch.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=url \
  -f package_url=https://example.com/openclaw-current.tgz \
  -f package_sha256=<64-char-sha256> \
  -f suite_profile=smoke

# Einen Tarball aus einer benannten vertrauenswürdigen privaten Mirror-Richtlinie validieren.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-current.tgz \
  -f package_sha256=<64-char-sha256> \
  -f suite_profile=smoke

# Einen von einem anderen Actions-Lauf hochgeladenen Tarball wiederverwenden.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=package-under-test \
  -f suite_profile=custom \
  -f docker_lanes='install-e2e plugin-update'
```

Beginnen Sie bei der Fehlerbehebung eines fehlgeschlagenen Package-Acceptance-Laufs mit der Zusammenfassung `resolve_package`, um Paketquelle, Version und SHA-256 zu bestätigen. Prüfen Sie anschließend den untergeordneten Lauf `docker_acceptance` und dessen Docker-Artefakte: `.artifacts/docker-tests/**/summary.json`, `failures.json`, Lane-Protokolle, Phasenzeiten und Befehle für erneute Läufe. Führen Sie vorzugsweise das fehlgeschlagene Paketprofil oder die exakten Docker-Lanes erneut aus, statt die vollständige Release-Validierung zu wiederholen.

## Installations-Smoke-Test

Der Workflow `Install Smoke` wird nicht mehr bei Pull Requests oder `main`-Pushes ausgeführt. Sein nächtlicher/manueller Wrapper und die Release-Validierung rufen beide den schreibgeschützten Kern `install-smoke-reusable.yml` auf, und jeder Lauf durchläuft auf von GitHub gehosteten Runnern den vollständigen Installations-Smoke-Test:

- Das Smoke-Image des Root-Dockerfiles wird einmal pro Ziel-SHA erstellt, in einem unveränderlichen Artefakt an die Workflow-Revision und den Producer-Versuch gebunden und anschließend vom CLI-Smoke-Test, vom CLI-Smoke-Test zum Löschen des gemeinsam genutzten Arbeitsbereichs durch Agents, vom Container-Gateway-Netzwerk-E2E und vom Build-Argument-Smoke-Test des gebündelten Plugins `matrix` geladen. Der Plugin-Smoke-Test überprüft die Spiegelung der Installation von Laufzeitabhängigkeiten sowie, dass das Plugin ohne Entry-Escape-Diagnosen geladen wird.
- Die QR-Paketinstallation und die Docker-Smoke-Tests für Installation/Aktualisierung (einschließlich Rocky-Linux-Installer-Lanes und einer Aktualisierungs-Lane gegen eine konfigurierbare npm-Baseline `update_baseline_version`) werden als separate Jobs ausgeführt, damit Installer-Arbeit nicht hinter den Smoke-Tests des Root-Images warten muss.

Der langsame Smoke-Test für den Image-Provider bei globaler Bun-Installation wird separat durch `run_bun_global_install_smoke` gesteuert. Er wird nach dem nächtlichen Zeitplan ausgeführt, ist standardmäßig für Workflow-Aufrufe aus Release-Prüfungen aktiviert, und bei manuellen `Install Smoke`-Dispatches kann er optional aktiviert werden. Die normale PR-CI führt für Node-relevante Änderungen weiterhin die schnelle Bun-Launcher-Regressions-Lane aus. QR- und Installer-Docker-Tests behalten ihre eigenen installationsorientierten Dockerfiles.

## Lokales Docker-E2E

`pnpm test:docker:all` erstellt ein gemeinsam genutztes Live-Test-Image vorab, packt OpenClaw einmal als npm-Tarball und erstellt zwei gemeinsam genutzte `scripts/e2e/Dockerfile`-Images:

- einen schlanken Node-/Git-Runner für Installer-, Aktualisierungs- und Plugin-Abhängigkeits-Lanes;
- ein funktionales Image, das denselben Tarball für normale Funktions-Lanes in `/app` installiert.

Die Definitionen der Docker-Lanes befinden sich in `scripts/lib/docker-e2e-scenarios.mjs`, die Planerlogik in `scripts/lib/docker-e2e-plan.mjs`, und der Runner führt nur den ausgewählten Plan aus. Der Scheduler wählt das Image pro Lane mit `OPENCLAW_DOCKER_E2E_BARE_IMAGE` und `OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE` aus und führt die Lanes anschließend mit `OPENCLAW_SKIP_DOCKER_BUILD=1` aus.

### Einstellbare Parameter

| Variable                               | Standardwert | Zweck                                                                                       |
| -------------------------------------- | ------------ | ------------------------------------------------------------------------------------------- |
| `OPENCLAW_DOCKER_ALL_PARALLELISM`      | 10      | Anzahl der Slots im Haupt-Pool für normale Lanes.                                                        |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM` | 10      | Anzahl der Slots im Provider-sensitiven Tail-Pool.                                                      |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT`       | 9       | Obergrenze für gleichzeitige Live-Lanes, damit Provider nicht drosseln.                                        |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT`        | 5       | Obergrenze für gleichzeitige npm-Installations-Lanes.                                                              |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT`    | 7       | Obergrenze für gleichzeitige Multi-Service-Lanes.                                                            |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS` | 2000    | Verzögerung zwischen Lane-Starts, um Erstellungsstürme beim Docker-Daemon zu vermeiden; für keine Verzögerung `0` festlegen.     |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS`  | 7200000 | Fallback-Zeitüberschreitung pro Lane (120 Minuten); ausgewählte Live-/Tail-Lanes verwenden engere Grenzen.           |
| `OPENCLAW_DOCKER_ALL_DRY_RUN`          | nicht gesetzt   | `1` gibt den Scheduler-Plan aus, ohne Lanes auszuführen.                                          |
| `OPENCLAW_DOCKER_ALL_LANES`            | nicht gesetzt   | Kommagetrennte Liste exakter Lanes; überspringt den Bereinigungs-Smoke-Test, damit Agents eine fehlgeschlagene Lane reproduzieren können. |

Eine Lane, die schwerer als ihre effektive Obergrenze ist, kann dennoch aus einem leeren Pool starten und läuft anschließend allein, bis sie Kapazität freigibt. Das lokale Aggregat führt Vorabprüfungen für Docker durch, entfernt veraltete OpenClaw-E2E-Container, gibt den Status aktiver Lanes aus, persistiert Lane-Laufzeiten für die Sortierung nach längster Laufzeit zuerst und plant nach dem ersten Fehler standardmäßig keine neuen gepoolten Lanes mehr ein.

### Wiederverwendbarer Live-/E2E-Workflow

Der wiederverwendbare Live-/E2E-Workflow fragt `scripts/test-docker-all.mjs --plan-json`, welches Paket, welche Image-Art, welches Live-Image, welche Lane und welche Abdeckung mit Zugangsdaten erforderlich sind. `scripts/docker-e2e.mjs` wandelt diesen Plan anschließend in GitHub-Ausgaben und Zusammenfassungen um. Er packt OpenClaw entweder über `scripts/package-openclaw-for-docker.mjs`, lädt ein Paketartefakt des aktuellen Laufs herunter oder lädt ein Paketartefakt aus `package_artifact_run_id` herunter und validiert anschließend das Tarball-Inventar. Der standardmäßige Pfad `no-push-artifact` erstellt über den Docker-Layer-Cache von Blacksmith mit dem Paket-Digest getaggte schlanke/funktionale Images, packt die exakten Image-Bytes in ein unveränderliches Workflow-Artefakt und lässt jeden Consumer dieses Artefakt überprüfen und laden. `existing-only` erfordert stattdessen explizite GHCR-Referenzen `docker_e2e_bare_image`/`docker_e2e_functional_image` und erstellt oder pusht niemals Images. Diese Registry-Abrufe verwenden pro Versuch eine begrenzte Zeitüberschreitung von 180 Sekunden, damit ein blockierter Stream schnell erneut versucht wird, statt den Großteil des kritischen Pfads der CI zu beanspruchen. Nach erfolgreicher geplanter Validierung übergibt `openclaw-scheduled-live-checks.yml` das unveränderliche Manifest der getesteten Images an den separaten Publisher mit Paketschreibzugriff; schreibgeschützte Release- und Vorab-Release-Aufrufer durchlaufen diesen Writer niemals.

### Abschnitte des Release-Pfads

Die Docker-Abdeckung für Releases führt kleinere, aufgeteilte Jobs mit `OPENCLAW_SKIP_DOCKER_BUILD=1` aus, sodass jeder Abschnitt nur die von ihm benötigte artefaktgestützte Image-Art überprüft und lädt (oder sie bei expliziter Wiederverwendung von `existing-only` abruft) und mehrere Lanes über denselben gewichteten Scheduler ausführt:

- `OPENCLAW_DOCKER_ALL_PROFILE=release-path`
- `OPENCLAW_DOCKER_ALL_CHUNK=core | package-update-openai | package-update-anthropic | package-update-core | plugins-runtime-plugins | plugins-runtime-services | plugins-runtime-install-a..h | openwebui`

Die aktuellen Docker-Abschnitte für Releases sind `core`, `package-update-openai`, `package-update-anthropic`, `package-update-core`, `plugins-runtime-plugins`, `plugins-runtime-services`, `plugins-runtime-install-a` bis `plugins-runtime-install-h` und `openwebui`. `package-update-openai` enthält die Live-Lane für das Codex-Plugin-Paket. Diese installiert das OpenClaw-Kandidatenpaket, installiert das Codex-Plugin aus `codex_plugin_spec` oder einem Tarball derselben Referenz mit ausdrücklicher Genehmigung zur Installation der Codex CLI, führt die Codex-CLI-Vorabprüfung und Agent-Turns in derselben Sitzung aus und führt anschließend einen Turn ohne Wiederholungsversuch mit mittlerem Denkaufwand aus, der Fortschritt sendet, zufällige Arbeitsbereichseingaben liest, deren exaktes Artefakt schreibt und den Abschluss meldet. `plugins-runtime-core`, `plugins-runtime` und `plugins-integrations` bleiben aggregierte Plugin-/Laufzeit-Aliasse. Der Lane-Alias `install-e2e` bleibt der aggregierte Alias für die manuelle erneute Ausführung beider Provider-Installer-Lanes.

OpenWebUI wird als eigenständiger Abschnitt `openwebui` auf einem dedizierten Blacksmith-Runner mit großer Festplatte ausgeführt, sobald Stable- oder vollständige Release-Pfad-Abdeckung ihn anfordert, selbst wenn der wiederverwendbare Workflow unterstützte Jobs an von GitHub gehostete Runner weiterleitet. Durch die Trennung des externen Image-Abrufs konkurriert das große Image in `plugins-runtime-services` nicht mit den gemeinsam genutzten Paket- und Plugin-Images; ältere aggregierte Plugin-/Laufzeit-Abschnitte enthalten OpenWebUI weiterhin für kompatible manuelle erneute Läufe. Aktualisierungs-Lanes für gebündelte Kanäle wiederholen den Vorgang bei vorübergehenden npm-Netzwerkfehlern einmal.

Jeder Abschnitt lädt `.artifacts/docker-tests/` mit Lane-Protokollen, Laufzeiten, `summary.json`, `failures.json`, Phasenzeiten, Scheduler-Plan-JSON, Tabellen langsamer Lanes und Befehlen für die erneute Ausführung einzelner Lanes hoch. Die Workflow-Eingabe `docker_lanes` führt ausgewählte Lanes mit für diesen Lauf vorbereiteten Images statt mit den Abschnittsjobs aus. Dadurch bleibt die Fehlerbehebung für fehlgeschlagene Lanes auf einen gezielten Docker-Job begrenzt; handelt es sich bei einer ausgewählten Lane um eine Live-Docker-Lane, erstellt der gezielte Job für diesen erneuten Lauf das Live-Test-Image lokal. Der Wiederholungslauf-Helfer validiert die exakt ausgewählte Ziel-SHA des Fehlerartefakts, und der manuelle Dispatch packt diese Referenz erneut, da das interne Pakettupel des wiederverwendbaren Workflows nicht Teil des Schemas `workflow_dispatch` ist. Generierte Befehle enthalten vorbereitete Image-Eingaben und `shared_image_policy=existing-only` nur dann, wenn diese Eingaben GHCR-gestützt sind; Runner-lokale Artefakt-Tags werden ausgelassen, damit ein neuer Runner sie neu erstellt. Eine explizite Zielüberschreibung entfernt wiederhergestellte GHCR-Image-Referenzen, sofern das Artefakt nicht belegt, dass sie mit der Überschreibung übereinstimmen. Vom Artefakt generierte Referenzen auf Workflow-Definitionen werden ebenfalls ausgelassen, da temporäre Full-Release-Branches gelöscht werden; der Dispatch verwendet den Standard-Branch des Repositorys, sofern der Operator ihn nicht ausdrücklich überschreibt.

```bash
pnpm test:docker:rerun <run-id>      # Docker-Artefakte herunterladen und kombinierte/je Lane gezielte Befehle für erneute Läufe ausgeben
pnpm test:docker:timings <summary>   # Zusammenfassungen langsamer Lanes und des kritischen Pfads der Phasen
```

Der geplante Live-/E2E-Workflow führt täglich die vollständige Docker-Suite des Release-Pfads aus und ruft nach erfolgreichem Abschluss den expliziten Publisher für die exakt getesteten Image-Artefakte auf.

## Plugin-Vorab-Release

`Plugin Prerelease` bietet eine aufwendigere Produkt-/Paketabdeckung und ist daher ein separater Workflow, der durch `Full Release Validation` oder explizit durch eine Bedienperson ausgelöst wird. Bei normalen Pull Requests, `main`-Pushes und eigenständigen manuellen CI-Auslösungen bleibt diese Suite deaktiviert. Er verteilt die Tests gebündelter Plugins auf acht Erweiterungs-Worker; diese Erweiterungs-Shard-Jobs führen bis zu zwei Plugin-Konfigurationsgruppen gleichzeitig aus, mit einem Vitest-Worker pro Gruppe und einem größeren Node-Heap, damit importintensive Plugin-Batches keine zusätzlichen CI-Jobs erzeugen. Der ausschließlich für Releases vorgesehene Docker-Pre-Release-Pfad (aktiviert durch die Eingabe `full_release_validation`) fasst gezielte Docker-Lanes in Vierergruppen zusammen, um nicht Dutzende Runner für Jobs mit einer Laufzeit von ein bis drei Minuten zu reservieren. Der Workflow lädt außerdem ein informatives `plugin-inspector-advisory`-Artefakt aus `@openclaw/plugin-inspector` hoch; die Ergebnisse des Inspectors dienen als Grundlage für die Triage und ändern nichts am blockierenden Plugin-Pre-Release-Gate.

## QA Lab

QA Lab verfügt außerhalb des primären intelligent eingegrenzten Workflows über dedizierte CI-Lanes. Die agentische Parität ist in die umfassenden QA- und Release-Harnesses eingebettet und kein eigenständiger PR-Workflow. Verwenden Sie `Full Release Validation` mit `rerun_group=qa-parity`, wenn die Parität im Rahmen eines umfassenden Validierungslaufs ausgeführt werden soll.

- Der Workflow `QA-Lab - All Lanes` läuft jede Nacht auf `main` sowie bei manueller Auslösung; er fächert sich in Mock-Paritätsjobs sowie Live-Jobs für Matrix, Telegram, Discord, WhatsApp und Slack auf. Live-Jobs verwenden die Umgebung `qa-live-shared`; Telegram, Discord, WhatsApp und Slack verwenden Convex-Leases, während Matrix temporäre lokale Anmeldedaten bereitstellt.

Release-Prüfungen führen Live-Transport-Lanes für Matrix und Telegram mit dem deterministischen Mock-Provider und für Mocks qualifizierten Modellen (`mock-openai/gpt-5.6-luna` und `mock-openai/gpt-5.6-luna-alt`) aus, sodass der Kanalvertrag von der Latenz der Live-Modelle und dem normalen Start der Provider-Plugins isoliert bleibt. Das Gateway für den Live-Transport deaktiviert die Speichersuche, da die QA-Parität das Speicherverhalten separat abdeckt; die Provider-Konnektivität wird durch die separaten Suites für Live-Modelle, native Provider und Docker-Provider abgedeckt.

Geplante und Release-bezogene Matrix-Gates verwenden den gemeinsam genutzten Suite-Host von QA Lab und den Live-Adapter mit den Release-Szenarien. Der CLI-Standardwert und die Eingabe des manuellen Workflows bleiben `all`; manuelle Auslösungen von `all` fächern sich in die Profile `transport`, `media`, `e2ee-smoke`, `e2ee-deep` und `e2ee-cli` auf, damit der Nachweis mit 93 Szenarien innerhalb der Zeitlimits pro Job bleibt. Gezielte manuelle Auslösungen wählen `fast`, `release` oder `transport` in einem Job aus.

`OpenClaw Release Checks` führt außerdem die releasekritischen QA-Lab-Lanes vor der Release-Freigabe aus; sein QA-Paritäts-Gate führt die Kandidaten- und Baseline-Pakete als parallele Lane-Jobs aus und lädt anschließend beide Artefakte in einen kleinen Berichtsjob herunter, der den abschließenden Paritätsvergleich durchführt.

Folgen Sie bei normalen PRs den Nachweisen aus eingegrenzter CI und eingegrenzten Prüfungen, statt die Parität als erforderlichen Status zu behandeln.

## CodeQL

Der Workflow `CodeQL` ist bewusst als eng begrenzter Sicherheitsscanner für den ersten Durchlauf konzipiert und nicht als vollständiger Scan des Repositorys. Bei täglichen und manuellen Läufen, `main`-Pushes sowie Schutzläufen für nicht als Entwurf markierte Pull Requests werden Actions-Workflow-Code und die JavaScript-/TypeScript-Bereiche mit dem höchsten Risiko mithilfe hochzuverlässiger Sicherheitsabfragen gescannt, die nach hohen/kritischen `security-severity` gefiltert sind.

Der Pull-Request-Schutz bleibt schlank: Er startet nur bei Änderungen unter `.github/actions`, `.github/codeql`, `.github/workflows`, `packages`, `scripts`, `src` oder in prozessverantwortlichen Laufzeitpfaden gebündelter Plugins und führt dieselbe Matrix hochzuverlässiger Sicherheitsabfragen wie der geplante Workflow aus. CodeQL für Android und macOS bleibt außerhalb der PR-Standardläufe.

### Sicherheitskategorien

| Kategorie                                         | Bereich                                                                                                                               |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-security-high/core-auth-secrets`         | Baseline für Authentifizierung, Geheimnisse, Sandbox, Cron und Gateway                                                               |
| `/codeql-security-high/channel-runtime-boundary`  | Implementierungsverträge der Kernkanäle sowie Kanallaufzeit des Plugins, Gateway, Plugin SDK, Geheimnisse und Audit-Berührungspunkte |
| `/codeql-security-high/network-ssrf-boundary`     | Bereiche für SSRF im Kern, IP-Parsing, Netzwerkschutz, Webabruf und SSRF-Richtlinien des Plugin SDK                                  |
| `/codeql-security-high/mcp-process-tool-boundary` | MCP-Server, Hilfsfunktionen zur Prozessausführung, ausgehende Zustellung und Gates für die Werkzeugausführung durch Agenten            |
| `/codeql-security-high/process-exec-boundary`     | Lokale Shell, Hilfsfunktionen zum Starten von Prozessen, prozessverantwortliche Laufzeiten gebündelter Plugins und Workflow-Skriptlogik |
| `/codeql-security-high/plugin-trust-boundary`     | Vertrauensbereiche für Plugin-Installation, Loader, Manifest, Registry, Paketmanagerinstallation, Laden von Quellen und Paketverträge des Plugin SDK |

### Plattformspezifische Sicherheits-Shards

- `CodeQL Android Critical Security` — geplanter Android-Sicherheits-Shard. Erstellt die Android-App manuell für CodeQL auf dem kleinsten Blacksmith-Linux-Runner, der von der Workflow-Plausibilitätsprüfung akzeptiert wird. Lädt unter `/codeql-critical-security/android` hoch.
- `CodeQL macOS Critical Security` — wöchentlicher/manueller macOS-Sicherheits-Shard. Erstellt die macOS-App manuell für CodeQL auf Blacksmith macOS, filtert Ergebnisse aus Abhängigkeits-Builds aus dem hochgeladenen SARIF heraus und lädt unter `/codeql-critical-security/macos` hoch. Bleibt außerhalb der täglichen Standardläufe, da der macOS-Build selbst bei fehlerfreiem Ergebnis die Laufzeit dominiert.

### Kategorien für kritische Qualität

`CodeQL Critical Quality` ist der entsprechende nicht sicherheitsbezogene Shard. Er führt ausschließlich nicht sicherheitsbezogene JavaScript-/TypeScript-Qualitätsabfragen mit Fehlerschweregrad über eng begrenzte, hochwertige Bereiche auf von GitHub gehosteten Linux-Runnern aus, damit Qualitätsscans das Budget für die Registrierung von Blacksmith-Runnern nicht beanspruchen. Sein Pull-Request-Schutz ist bewusst kleiner als das geplante Profil: Nicht als Entwurf markierte PRs führen aus dreizehn für PRs routbaren Shards nur die passenden Shards für die von ihnen berührten Bereiche aus — `agent-runtime-boundary`, `channel-runtime-boundary`, `config-boundary`, `core-auth-secrets`, `gateway-runtime-boundary`, `mcp-process-runtime-boundary`, `memory-runtime-boundary`, `network-runtime-boundary`, `plugin-boundary`, `plugin-sdk-package-contract`, `plugin-sdk-reply-runtime`, `provider-runtime-boundary` und `session-diagnostics-boundary`. `ui-control-plane` und `web-media-runtime-boundary` bleiben außerhalb der PR-Läufe. Änderungen an der CodeQL-Konfiguration und am Qualitäts-Workflow führen den vollständigen Satz an PR-Shards aus (die Schlüssel für den Netzwerklaufzeit-Shard richten sich nach seinen eigenen CodeQL-Konfigurationsdateien und netzwerkverantwortlichen Quellpfaden).

Die manuelle Auslösung akzeptiert:

```text
profile=all|agent-runtime-boundary|config-boundary|core-auth-secrets|channel-runtime-boundary|gateway-runtime-boundary|memory-runtime-boundary|mcp-process-runtime-boundary|network-runtime-boundary|plugin-boundary|plugin-sdk-package-contract|plugin-sdk-reply-runtime|provider-runtime-boundary|session-diagnostics-boundary
```

Die eng begrenzten Profile dienen als Lern- und Iterationsschnittstellen, um einen Qualitätsshard isoliert auszuführen.

| Kategorie                                                | Bereich                                                                                                                                                                     |
| -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-critical-quality/core-auth-secrets`            | Code für die Sicherheitsgrenzen von Authentifizierung, Geheimnissen, Sandbox, Cron und Gateway                                                                              |
| `/codeql-critical-quality/config-boundary`              | Verträge für Konfigurationsschema, Migration, Normalisierung und E/A                                                                                                        |
| `/codeql-critical-quality/gateway-runtime-boundary`     | Gateway-Protokollschemas und Verträge für Servermethoden                                                                                                                     |
| `/codeql-critical-quality/channel-runtime-boundary`     | Implementierungsverträge für Kernkanäle und gebündelte Kanal-Plugins                                                                                                        |
| `/codeql-critical-quality/agent-runtime-boundary`       | Laufzeitverträge für Befehlsausführung, Modell-/Provider-Verteilung, automatische Antwortverteilung und Warteschlangen sowie die ACP-Steuerungsebene                         |
| `/codeql-critical-quality/mcp-process-runtime-boundary` | MCP-Server und Werkzeugbrücken, Hilfsfunktionen zur Prozessüberwachung und Verträge für ausgehende Zustellung                                                                |
| `/codeql-critical-quality/memory-runtime-boundary`      | Speicher-Host-SDK, Speicherlaufzeit-Fassaden, Speicheraliase des Plugin SDK, Aktivierungslogik der Speicherlaufzeit und Speicherbefehle für Doctor                          |
| `/codeql-critical-quality/network-runtime-boundary`     | Paket für Netzwerkrichtlinien, Laufzeit für Roh-Sockets und Proxy-Erfassung, SSH-Tunnel, Gateway-Sperre, JSONL-Socket und Push-Transportbereiche                            |
| `/codeql-critical-quality/session-diagnostics-boundary` | Interna der Antwortwarteschlange, Sitzungszustellungs-Warteschlangen, Hilfsfunktionen für Bindung/Zustellung ausgehender Sitzungen, Bereiche für Diagnoseereignisse/Protokollpakete und CLI-Verträge des Sitzungs-Doctors |
| `/codeql-critical-quality/plugin-sdk-reply-runtime`     | Verteilung eingehender Antworten im Plugin SDK, Hilfsfunktionen für Antwortnutzlast/Segmentierung/Laufzeit, Kanalantwortoptionen, Zustellungswarteschlangen und Hilfsfunktionen zur Sitzungs-/Thread-Bindung |
| `/codeql-critical-quality/provider-runtime-boundary`    | Normalisierung des Modellkatalogs, Provider-Authentifizierung und -Erkennung, Registrierung der Provider-Laufzeit, Provider-Standards/Kataloge und Registrys für Web/Suche/Abruf/Einbettung |
| `/codeql-critical-quality/ui-control-plane`             | Bootstrap der Control UI, lokale Persistenz, Gateway-Steuerungsabläufe und Laufzeitverträge der Aufgabensteuerungsebene                                                     |
| `/codeql-critical-quality/web-media-runtime-boundary`   | Laufzeitverträge für Webabruf/-suche im Kern, Medien-E/A, Medienverständnis, Bilderzeugung und Medienerzeugung                                                               |
| `/codeql-critical-quality/plugin-boundary`              | Verträge für Loader, Registry, öffentliche Oberfläche und Einstiegspunkte des Plugin SDK                                                                                   |
| `/codeql-critical-quality/plugin-sdk-package-contract`  | Veröffentlichter paketinterner Quellcode des Plugin SDK und Hilfsfunktionen für Plugin-Paketverträge                                                                        |

Qualität bleibt von Sicherheit getrennt, damit Qualitätsergebnisse geplant, gemessen, deaktiviert oder erweitert werden können, ohne das Sicherheitssignal zu verschleiern. Eine CodeQL-Erweiterung für Swift, Python und gebündelte Plugins sollte erst dann als eingegrenzte oder auf Shards verteilte Folgearbeit wieder hinzugefügt werden, wenn die eng begrenzten Profile eine stabile Laufzeit und ein stabiles Signal aufweisen.

## Wartungs-Workflows

### Docs Agent

Der Workflow `Docs Agent` ist eine ereignisgesteuerte Codex-Wartungs-Lane, die bestehende Dokumentation mit kürzlich übernommenen Änderungen synchron hält. Er hat keinen rein zeitgesteuerten Lauf: Ein erfolgreicher Push-CI-Lauf auf `main`, der nicht von einem Bot stammt, kann ihn auslösen; außerdem kann er direkt manuell ausgelöst werden. Durch Workflow-Läufe ausgelöste Aufrufe werden übersprungen, wenn `main` bereits weitergelaufen ist oder innerhalb der letzten Stunde ein anderer nicht übersprungener Docs-Agent-Lauf erstellt wurde. Bei der Ausführung prüft er den Commit-Bereich vom Quell-SHA des vorherigen nicht übersprungenen Docs-Agent-Laufs bis zum aktuellen `main`, sodass ein einzelner stündlicher Lauf alle seit dem letzten Dokumentationsdurchlauf auf main angesammelten Änderungen abdecken kann.

### Testleistungs-Agent

Der Workflow `Test Performance Agent` ist eine ereignisgesteuerte Codex-Wartungslane für langsame Tests. Er hat keinen reinen Zeitplan: Ein erfolgreicher, nicht von einem Bot stammender Push-CI-Lauf auf `main` kann ihn auslösen, er wird jedoch übersprungen, wenn an diesem UTC-Tag bereits ein anderer Workflow-Run-Aufruf ausgeführt wurde oder noch läuft. Eine manuelle Auslösung umgeht diese tägliche Aktivitätssperre. Die Lane erstellt einen nach Gruppen aufgeschlüsselten Vitest-Performancebericht für die vollständige Testsuite, erlaubt Codex ausschließlich kleine, die Testabdeckung erhaltende Performancekorrekturen an Tests anstelle umfassender Refactorings, führt anschließend den Bericht für die vollständige Testsuite erneut aus und verwirft Änderungen, welche die Anzahl bestandener Tests der Baseline reduzieren. Der gruppierte Bericht erfasst für jede Konfiguration die verstrichene Zeit und den maximalen RSS unter Linux und macOS, sodass der Vorher-Nachher-Vergleich Änderungen des Testspeicherverbrauchs neben Änderungen der Dauer sichtbar macht. Wenn die Baseline fehlgeschlagene Tests enthält, darf Codex nur offensichtliche Fehler beheben, und der Bericht für die vollständige Testsuite nach dem Agentenlauf muss erfolgreich sein, bevor etwas committet wird. Wenn `main` fortgeschritten ist, bevor der Bot-Push übernommen wird, führt die Lane einen Rebase des validierten Patches durch, führt `pnpm check:changed` erneut aus und versucht den Push erneut; konfliktbehaftete veraltete Patches werden übersprungen. Sie verwendet von GitHub gehostetes Ubuntu, damit die Codex-Action dieselbe Sicherheitsstrategie zum Entziehen von sudo-Rechten wie der Dokumentationsagent beibehalten kann.

### Doppelte PRs nach dem Merge

Der Workflow `Duplicate PRs After Merge` ist ein manueller Maintainer-Workflow für die Bereinigung von Duplikaten nach der Übernahme. Standardmäßig wird ein Probelauf durchgeführt, und ausdrücklich aufgeführte PRs werden nur geschlossen, wenn `apply=true`. Vor Änderungen auf GitHub überprüft er, ob der übernommene PR gemergt wurde und ob jedes Duplikat entweder auf dasselbe Issue verweist oder sich überschneidende geänderte Codeabschnitte enthält.

```bash
gh workflow run duplicate-after-merge.yml \
  -f landed_pr=70532 \
  -f duplicate_prs='70530,70592' \
  -f apply=true
```

## Lokale Prüf-Gates und Routing von Änderungen

### Ratsche für die Anzahl der Konfigurations-Baseline

`pnpm config:docs:check` weist nicht dokumentiertes Wachstum der Konfigurationsoberfläche sowie beschädigte oder veraltete Zähler-Snapshots zurück. Wenn eine geprüfte Produktänderung absichtlich Schemapfade hinzufügt, führen Sie `pnpm config:docs:gen` aus, prüfen Sie die Änderungen der Zähler für Core, Kanäle und Plugins sowie die generierten SHA-256-Dateien und committen Sie die bewusste Erhöhung der Baseline zusammen mit Schema, Hilfe, Labels, Migration und Tests. Bearbeiten Sie die Zählerdatei nicht manuell, um die Ratsche zu umgehen.

Autoren von Konfigurationen müssen neue Blätter außerdem für die Einstellungen einer Stufe zuordnen. Fügen Sie am Blatt `advanced: false` oder
`advanced: true` hinzu oder ordnen Sie den Schlüssel einem Vorfahren unter, dessen Stufe
alle Nachfahren erben sollen. Nicht klassifizierte Wurzeln lassen den Test zur Schemaqualität
mit kopierbaren Vorlagen fehlschlagen; Pfade ohne Vorfahren werden standardmäßig als erweitert eingestuft.
Der kuratierte Snapshot häufig verwendeter Blätter macht beabsichtigte Stufenänderungen im
Review sichtbar.

Die lokale Logik für geänderte Lanes befindet sich in `scripts/changed-lanes.mjs` und wird von `scripts/check-changed.mjs` ausgeführt. Dieses lokale Prüf-Gate ist hinsichtlich der Architekturgrenzen strenger als der breite Plattformumfang der CI:

- Änderungen am Core-Produktionscode führen die Typprüfung für Core-Produktion und Core-Tests sowie Core-Linting und Schutzprüfungen aus;
- reine Änderungen an Core-Tests führen nur die Typprüfung für Core-Tests sowie Core-Linting aus;
- Änderungen am Produktionscode von Erweiterungen führen die Typprüfung für Erweiterungsproduktion und Erweiterungstests sowie Erweiterungs-Linting aus;
- reine Änderungen an Erweiterungstests führen die Typprüfung für Erweiterungstests sowie Erweiterungs-Linting aus;
- Änderungen am öffentlichen Plugin SDK oder an Plugin-Verträgen erweitern den Umfang um die Typprüfung der Erweiterungen, da Erweiterungen von diesen Core-Verträgen abhängen (Vitest-Durchläufe für Erweiterungen bleiben ausdrücklich separate Testarbeit);
- reine Versionsänderungen an Release-Metadaten führen gezielte Prüfungen von Versionen, Konfiguration und Root-Abhängigkeiten aus;
- unbekannte Änderungen an Root-Dateien oder der Konfiguration aktivieren sicherheitshalber alle Prüf-Lanes.

Das lokale Routing geänderter Tests befindet sich in `scripts/test-projects.test-support.mjs` und ist bewusst kostengünstiger als `check:changed`: Direkt bearbeitete Tests führen sich selbst aus, Quellcodeänderungen bevorzugen explizite Zuordnungen und anschließend Tests auf derselben Ebene sowie vom Importgraphen abhängige Tests. Die gemeinsame Konfiguration für die Zustellung in Gruppenräumen ist eine dieser expliziten Zuordnungen: Änderungen an der Konfiguration sichtbarer Gruppenantworten, am Zustellungsmodus für Quellantworten oder am System-Prompt des Nachrichten-Tools werden über die Core-Antworttests sowie Discord- und Slack-Zustellungsregressionen geleitet, damit eine Änderung eines gemeinsamen Standardwerts bereits vor dem ersten PR-Push fehlschlägt. Verwenden Sie `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` nur, wenn die Änderung das Testsystem so umfassend betrifft, dass die kostengünstige zugeordnete Testmenge keinen verlässlichen Ersatz darstellt.

## Testbox-Validierung

Crabbox ist der Repository-eigene Wrapper für Remote-Boxen zum Linux-Nachweis durch Maintainer. Agentensitzungen
führen einen oder wenige fokussierte Tests und kostengünstige statische Prüfungen nur dann lokal
für vertrauenswürdigen Quellcode aus, wenn die vorhandene Installation der Abhängigkeiten bereit ist. Für größere Testsuiten und
rechenintensive Arbeiten verwenden sie Crabbox, darunter Builds, Typprüfungen, aufgefächertes Linting,
Docker, Paket-Lanes, E2E, Live-Nachweise und CI-Parität. Umfangreiche Nachweise durch vertrauenswürdige Maintainer
verwenden standardmäßig `blacksmith-testbox`, und auch `.crabbox.yaml` verwendet dies nun standardmäßig. Der konfigurierte
Workflow stattet die Umgebung mit Provider- und Agentenanmeldedaten aus; nicht vertrauenswürdiger Code von Beitragenden oder
Forks muss daher stattdessen geheimnisfreie Fork-CI oder eine bereinigte direkte AWS-Crabbox verwenden.
Bereinigte AWS-Läufe setzen `CRABBOX_ENV_ALLOW=CI`, übergeben
`--no-hydrate` und verwenden ein frisches temporäres Remote-`HOME`; dadurch wird verhindert, dass die Repository-
Allowlist `OPENCLAW_*` und bestehende Authentifizierungsprofile nicht vertrauenswürdigen Code erreichen.
Sie verwenden eine neu aufgewärmte Lease, die ausschließlich diesem nicht vertrauenswürdigen Quellcode zugeordnet ist, niemals eine
vertrauenswürdige oder zuvor mit Anmeldedaten ausgestattete Lease. Starten Sie eine installierte vertrauenswürdige Crabbox-
Binärdatei aus einem sauberen vertrauenswürdigen `main`-Checkout und rufen Sie mit
`--fresh-pr` ausschließlich den Remote-PR ab; führen Sie niemals den Wrapper oder die Konfiguration des nicht vertrauenswürdigen Checkouts lokal aus.
Heben Sie die Festlegung von `CRABBOX_AWS_INSTANCE_PROFILE` auf und brechen Sie sicher ab, sofern das aufgelöste
`aws.instanceProfile` nicht leer ist. Verwenden Sie vor jeder Installation und jedem Test vertrauenswürdige
Werkzeuge mit absoluten Pfaden, um ein IMDSv2-Token zu verlangen, nachzuweisen, dass der IAM-Anmeldedaten-
Endpunkt 404 zurückgibt, und den Remote-Wert `git rev-parse HEAD` mit dem vollständigen
geprüften Head-SHA des PRs zu vergleichen. Binden Sie die Lease an diesen SHA und stoppen beziehungsweise erwärmen Sie sie bei einer Änderung des Heads erneut.
Laden Sie das vertrauenswürdige `scripts/crabbox-untrusted-bootstrap.sh` aus einem sauberen `main`
zusammen mit `--fresh-pr` hoch; es installiert festgelegte Node-/pnpm-Versionen, überprüft den SHA und
die Festlegung des Paketmanagers, isoliert `HOME`, installiert Abhängigkeiten und führt anschließend den
angeforderten Test aus.
Heben Sie alle Überschreibungen von `CRABBOX_TAILSCALE*` auf, erzwingen Sie `--network public
--tailscale=false`, entfernen Sie Exit-Node-/LAN-Flags und verlangen Sie, dass `crabbox inspect`
öffentliches Networking ohne Tailscale-Status meldet, bevor ein Skript hochgeladen wird.
Eigene AWS-/Hetzner-Kapazität bleibt außerdem die Ausweichoption bei Blacksmith-Ausfällen,
Kontingentproblemen oder ausdrücklich angeforderten Tests auf eigener Kapazität.

Agenten wärmen keine Testbox für erwartete Arbeiten vor. Fordern Sie eine Testbox erst dann an, wenn der
erste umfangreiche Befehl bereit ist, verwenden Sie die zurückgegebene `tbx_...`-ID für spätere umfangreiche
Befehle erneut, synchronisieren Sie bei jedem Lauf den aktuellen Checkout und stoppen Sie sie vor der Übergabe.

Crabbox-gestützte Blacksmith-Läufe wärmen einmalig verwendete Testboxen auf, beanspruchen und synchronisieren sie, führen Befehle aus, erstellen Berichte und bereinigen sie.
Die integrierte Plausibilitätsprüfung der Synchronisierung schlägt frühzeitig fehl, wenn
`git status --short` auf der synchronisierten Box mindestens 200 Löschungen nachverfolgter Dateien anzeigt,
wodurch das Verschwinden von Root-Dateien wie `pnpm-lock.yaml` erkannt wird. Setzen Sie bei PRs mit beabsichtigten
umfangreichen Löschungen `CRABBOX_ALLOW_MASS_DELETIONS=1` für den Remote-Befehl.

Crabbox beendet außerdem einen lokalen Aufruf der Blacksmith-CLI, der länger als fünf Minuten in der
Synchronisierungsphase verbleibt, ohne Ausgabe nach der Synchronisierung zu erzeugen. Setzen Sie
`CRABBOX_BLACKSMITH_SYNC_TIMEOUT_MS=0`, um diese Schutzprüfung zu deaktivieren, oder verwenden Sie bei ungewöhnlich großen lokalen Diffs einen höheren
Millisekundenwert.

Prüfen Sie den Wrapper vor dem ersten Lauf aus dem Repository-Root:

```bash
pnpm crabbox:run -- --help | sed -n '1,120p'
```

Der Repository-Wrapper lehnt eine veraltete Crabbox-Binärdatei ab, die den ausgewählten Provider nicht aufführt. Blacksmith-gestützte Läufe erfordern Crabbox 0.22.0 oder neuer, damit der Wrapper das aktuelle Verhalten für Synchronisierung, Warteschlange und Bereinigung der Testbox erhält. Vermeiden Sie in Codex-Worktrees oder verknüpften beziehungsweise Sparse-Checkouts das lokale Skript `pnpm crabbox:run`, da pnpm möglicherweise Abhängigkeiten abgleicht, bevor Crabbox startet; rufen Sie stattdessen den Node-Wrapper direkt auf:

```bash
node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox --timing-json --shell -- "pnpm test <path-or-filter>"
```

Wenn Sie den benachbarten Checkout verwenden, erstellen Sie die ignorierte lokale Binärdatei vor Zeitmessungen oder Nachweisarbeiten neu:

```bash
version="$(git -C ../crabbox describe --tags --always --dirty | sed 's/^v//')" \
  && go build -C ../crabbox -trimpath -ldflags "-s -w -X github.com/openclaw/crabbox/internal/cli.version=${version}" -o bin/crabbox ./cmd/crabbox
```

Der Block `blacksmith:` in `.crabbox.yaml` legt bereits die Standardwerte für Organisation, Workflow, Job und Ref fest, sodass die folgenden expliziten Flags optional sind. Gate für Änderungen:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --blacksmith-org openclaw \
  --blacksmith-workflow .github/workflows/ci-check-testbox.yml \
  --blacksmith-job check \
  --blacksmith-ref main \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm check:changed"
```

Erneute Ausführung eines fokussierten Tests auf der Testbox, wenn lokale Abhängigkeiten nicht verfügbar sind oder das
Ziel auffächert:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm test <path-or-filter>"
```

Vollständige Testsuite:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm test"
```

Lesen Sie die abschließende JSON-Zusammenfassung. Die relevanten Felder sind `provider`, `leaseId`,
`syncDelegated`, `exitCode`, `commandMs` und `totalMs`. Bei delegierten
Blacksmith-Testbox-Läufen bilden der Exit-Code des Crabbox-Wrappers und die JSON-Zusammenfassung das
Befehlsergebnis. Der verknüpfte GitHub-Actions-Lauf ist für die Ausstattung mit Anmeldedaten und das Keepalive verantwortlich; er
kann mit `cancelled` enden, wenn die Testbox extern gestoppt wird, nachdem der SSH-
Befehl bereits zurückgekehrt ist. Behandeln Sie dies als Bereinigungs-/Statusartefakt, sofern
`exitCode` des Wrappers nicht ungleich null ist oder die Befehlsausgabe keinen fehlgeschlagenen Test zeigt.
Einmalige Blacksmith-gestützte Crabbox-Läufe sollten die Testbox automatisch stoppen;
wenn ein Lauf unterbrochen wurde oder die Bereinigung unklar ist, prüfen Sie die aktiven Boxen und stoppen Sie ausschließlich
die von Ihnen erstellten Boxen:

```bash
blacksmith testbox list --all
blacksmith testbox status --id <tbx_id>
blacksmith testbox stop --id <tbx_id>
```

Verwenden Sie die Wiederverwendung nur, wenn Sie absichtlich mehrere Befehle auf derselben mit Anmeldedaten ausgestatteten Box benötigen:

```bash
node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox --id <tbx_id> --timing-json --shell -- "corepack pnpm test <path-or-filter>"
pnpm crabbox:stop -- <tbx_id>
```

Verwenden Sie die Lease erneut, nicht veralteten Quellcode. Lassen Sie `--no-sync` weg, damit jeder Lauf den
aktuellen Checkout hochlädt; verwenden Sie es nur, um einen unveränderten, bereits synchronisierten Dateibaum
absichtlich erneut auszuführen. Nicht vertrauenswürdiger Code von Beitragenden oder Forks muss bei jedem Befehl
`CRABBOX_ENV_ALLOW=CI`, `--provider aws --no-hydrate` und ein frisches
temporäres Remote-`HOME` verwenden; installieren Sie die Abhängigkeiten innerhalb dieses
bereinigten Befehls, bevor Sie Tests ausführen. Verwenden Sie ausschließlich eine neu aufgewärmte Lease erneut, die demselben
nicht vertrauenswürdigen Quellcode zugeordnet ist; niemals eine vertrauenswürdige oder zuvor mit Anmeldedaten ausgestattete Lease. Führen Sie niemals
den Wrapper oder die Konfiguration des nicht vertrauenswürdigen Checkouts lokal aus: Starten Sie die installierte
vertrauenswürdige Crabbox-Binärdatei aus einem sauberen vertrauenswürdigen `main` und übergeben Sie bei jedem
Lauf `--fresh-pr`. Lassen Sie `CRABBOX_AWS_INSTANCE_PROFILE` ungesetzt, weisen Sie ein nicht leeres aufgelöstes
Instanzprofil zurück, verlangen Sie einen vertrauenswürdigen Remote-IMDS-Nachweis ohne Rolle und überprüfen Sie den
geprüften Head-SHA vor Installation und Test. Binden Sie die Lease an diesen SHA; stoppen und
erwärmen Sie sie nach jeder Änderung des Heads erneut. Wenn kein Remote-PR vorhanden ist, verwenden Sie geheimnisfreie Fork-CI.
Wählen Sie für nicht vertrauenswürdigen Quellcode niemals `hydrate-github` oder den mit Anmeldedaten ausgestatteten Blacksmith-Workflow
aus.

Wenn Crabbox die fehlerhafte Schicht ist, Blacksmith selbst jedoch funktioniert, verwenden Sie direktes
Blacksmith nur für Diagnosen wie `list`, `status` und die Bereinigung. Beheben Sie den
Crabbox-Pfad, bevor Sie einen direkten Blacksmith-Lauf als Maintainer-Nachweis betrachten.

Wenn `blacksmith testbox list --all` und `blacksmith testbox status` funktionieren, neue
Warmups aber nach einigen Minuten weiterhin `queued` ohne IP oder URL des Actions-Laufs anzeigen,
ist von einer Belastung des Blacksmith-Providers, der Warteschlange, der Abrechnung oder der Organisationslimits auszugehen. Stoppen Sie die
von Ihnen erstellten IDs in der Warteschlange, starten Sie keine weiteren Testboxes und verlagern Sie den Nachweis auf den
unten beschriebenen Kapazitätspfad der eigenen Crabbox, während jemand das Blacksmith-Dashboard,
die Abrechnung und die Organisationslimits prüft.

Weichen Sie nur dann auf eigene Crabbox-Kapazität aus, wenn Blacksmith ausgefallen oder durch Kontingente eingeschränkt ist, die erforderliche Umgebung fehlt oder eigene Kapazität ausdrücklich das Ziel ist:

```bash
CRABBOX_CAPACITY_REGIONS=eu-west-1,eu-west-2,eu-central-1,us-east-1,us-west-2 \
  pnpm crabbox:warmup -- --provider aws --class standard --market on-demand --idle-timeout 90m
pnpm crabbox:hydrate -- --provider aws --id <cbx_id-or-slug>
pnpm crabbox:run -- --provider aws --id <cbx_id-or-slug> --timing-json --shell -- "pnpm check:changed"
pnpm crabbox:stop -- --provider aws <cbx_id-or-slug>
```

Vermeiden Sie bei hoher AWS-Auslastung `class=beast`, sofern die Aufgabe nicht tatsächlich CPU-Kapazität der 48xlarge-Klasse benötigt. Eine `beast`-Anforderung beginnt bei 192 vCPUs und überschreitet am leichtesten das regionale EC2-Spot- oder On-Demand-Standard-Kontingent. Die repo-eigene Konfiguration `.crabbox.yaml` verwendet standardmäßig `class: standard`, den On-Demand-Markt und `capacity.hints: true`, sodass vermittelte AWS-Leases die ausgewählte Region und den ausgewählten Markt, die Kontingentbelastung, den Spot-Fallback sowie Warnungen zu Klassen mit hoher Auslastung ausgeben. Verwenden Sie `fast` für umfangreichere breite Prüfungen, `large` erst, wenn standard/fast nicht ausreichen, und `beast` nur für außergewöhnliche CPU-intensive Lanes wie vollständige Testsuiten oder Docker-Matrizen für alle Plugins, explizite Release-/Blocker-Validierungen oder Performance-Profiling mit hoher Kernanzahl. Verwenden Sie `beast` nicht für `pnpm check:changed`, fokussierte Tests, reine Dokumentationsarbeit, gewöhnliches Linting/Typechecking, kleine E2E-Reproduktionen oder die Triage eines Blacksmith-Ausfalls. Verwenden Sie `--market on-demand` für die Kapazitätsdiagnose, damit Schwankungen des Spot-Markts das Signal nicht verfälschen.

`.crabbox.yaml` steuert die Standardwerte für Provider, Synchronisierung und GitHub-Actions-Hydration. Die Crabbox-Synchronisierung überträgt niemals `.git`, sodass der hydrierte Actions-Checkout seine eigenen entfernten Git-Metadaten behält, anstatt lokale Remotes und Objektspeicher der Maintainer zu synchronisieren. Die Repo-Konfiguration schließt außerdem lokale Laufzeit-/Build-Artefakte aus (wie `.artifacts` und Testberichte), die niemals übertragen werden dürfen. `.github/workflows/crabbox-hydrate.yml` steuert Checkout, Node-/pnpm-Einrichtung, den Abruf von `origin/main` und die Übergabe der nicht geheimen Umgebung für `crabbox run --id <cbx_id>`-Befehle in der eigenen Cloud.

## Verwandte Themen

- [Installationsübersicht](/de/install)
- [Entwicklungskanäle](/de/install/development-channels)
