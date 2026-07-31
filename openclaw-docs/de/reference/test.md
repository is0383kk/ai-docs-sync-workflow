---
read_when:
    - Tests ausführen oder beheben
summary: So führen Sie Tests lokal aus (vitest) und wann Sie Force-/Coverage-Modi verwenden sollten
title: Tests
x-i18n:
    generated_at: "2026-07-26T19:14:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 391185703e853bb523e1396eb22da4693d10d47b1644d3b2a51707d329f67dae
    source_path: reference/test.md
    workflow: 16
---

- Vollständiges Testpaket (Suites, Live, Docker): [Tests](/de/help/testing)
- Validierung von Updates und Plugin-Paketen: [Updates und Plugins testen](/de/help/testing-updates-plugins)

## Agent-Standard

Agent-Sitzungen führen einen oder wenige fokussierte Tests und kostengünstige statische Prüfungen nur dann lokal aus,
wenn die Quelle vertrauenswürdig und die bestehende Abhängigkeitsinstallation bereit ist. Führen Sie niemals
Werkzeuge eines nicht vertrauenswürdigen Repositorys lokal aus. Größere Suites, Gates für Änderungen mit
Typecheck-/Lint-Fan-out, Builds, Docker, Paket-Lanes, E2E, Live-Nachweise und
plattformübergreifende Validierungen werden remote über Crabbox ausgeführt. Aufwendige Nachweise durch vertrauenswürdige Maintainer
werden standardmäßig in der Blacksmith Testbox ausgeführt. Der konfigurierte Testbox-Workflow
stellt Anmeldedaten bereit, daher muss nicht vertrauenswürdiger Code von Mitwirkenden oder Forks stattdessen
geheimnislose Fork-CI oder eine bereinigte direkte AWS-Crabbox verwenden.

Wärmen Sie die Umgebung nicht für erwartete Arbeiten vor. Fordern Sie das Backend erst dann an, wenn
der erste aufwendige Befehl bereit ist, verwenden Sie die zurückgegebene `tbx_...`-ID für spätere aufwendige
Befehle erneut, synchronisieren Sie bei jedem Lauf den aktuellen Checkout und stoppen Sie es vor der Übergabe.

Nach der ersten erfolgreichen Wiederverwendung speichert der Wrapper die Fingerabdrücke der Basis,
der Abhängigkeiten und des Testbox-Workflows der Lease unter `.crabbox/testbox-leases/`.
Bei reinen Quelltextänderungen wird die vorgewärmte Box weiterverwendet. Eine geänderte Merge-Basis, Lockfile,
Paketmanager-Eingabe, ein geänderter Wrapper oder Testbox-Workflow führt zu einem sicheren Abbruch und erfordert eine
neue Lease. Bei jedem Lauf wird weiterhin der aktuelle Checkout synchronisiert.
`OPENCLAW_TESTBOX_ALLOW_STALE=1` dient nur der gezielten Diagnose, nicht
als Release-Nachweis.

Die folgenden lokalen Testbefehle sind für menschliche Arbeitsabläufe und begrenzte Agent-Nachweise vorgesehen.
Die Nichtverfügbarkeit eines Remote-Providers muss gemeldet werden; sie ist keine Erlaubnis,
stillschweigend ein breites lokales Gate auszuführen.

Für nicht vertrauenswürdige aufwendige Nachweise wärmen Sie die Umgebung bei Bedarf mit `--provider aws` vor. Jeder Lauf muss
`CRABBOX_ENV_ALLOW=CI` setzen, `--provider aws --no-hydrate` übergeben und
vor der Installation von Abhängigkeiten oder der Ausführung von Tests eine neue temporäre Remote-`HOME`
verwenden. Verwenden Sie eine neu vorgewärmte Lease, die ausschließlich dieser nicht vertrauenswürdigen Quelle gewidmet ist; verwenden Sie niemals
eine vertrauenswürdige oder zuvor mit Anmeldedaten ausgestattete Lease erneut. Starten Sie eine installierte vertrauenswürdige Crabbox-
Binärdatei aus einem sauberen vertrauenswürdigen `main`-Checkout und rufen Sie mit
`--fresh-pr` ausschließlich den Remote-PR ab; führen Sie niemals den Wrapper oder die Konfiguration des nicht vertrauenswürdigen Checkouts lokal aus.
Entfernen Sie `CRABBOX_AWS_INSTANCE_PROFILE` und brechen Sie sicher ab, sofern der aufgelöste Wert
`aws.instanceProfile` nicht leer ist. Verwenden Sie vor jeder Installation bzw. jedem Test vertrauenswürdige
Werkzeuge mit absoluten Pfaden, um ein IMDSv2-Token zu erzwingen, nachzuweisen, dass der IAM-Anmeldedaten-
Endpunkt 404 zurückgibt, und zu überprüfen, dass der Remote-Wert `git rev-parse HEAD` der vollständigen
geprüften SHA des PR-Heads entspricht. Binden Sie die Lease an diese SHA und stoppen bzw. wärmen Sie sie neu vor, wenn sich der Head
ändert. Laden Sie die vertrauenswürdige Datei `scripts/crabbox-untrusted-bootstrap.sh` aus einem sauberen
`main` zusammen mit `--fresh-pr` hoch; sie installiert die festgelegten Node-/pnpm-Versionen, überprüft die SHA
und die Paketmanager-Festlegung, isoliert `HOME`, installiert Abhängigkeiten und führt anschließend
den angeforderten Test aus. Wenn der Broker nicht nachweisen kann, dass keine Rolle vorhanden ist, oder kein Remote-PR existiert,
verwenden Sie geheimnislose Fork-CI. Verwenden Sie weder `hydrate-github` noch `--no-sync` oder einen
mit Anmeldedaten ausgestatteten Testbox-Workflow.
Entfernen Sie alle `CRABBOX_TAILSCALE*`-Überschreibungen, erzwingen Sie `--network public
--tailscale=false`, löschen Sie Exit-Node-/LAN-Flags und verlangen Sie, dass `crabbox inspect`
ein öffentliches Netzwerk ohne Tailscale-Status meldet, bevor Sie ein Skript hochladen.

## Reguläre lokale Reihenfolge

1. `pnpm test:changed` für Vitest-Nachweise im Änderungsumfang.
2. `pnpm test <path-or-filter>` für eine Datei, ein Verzeichnis oder ein explizites Ziel.
3. `pnpm test` nur, wenn Sie bewusst die vollständige lokale Vitest-Suite benötigen.

In einem Codex-Worktree oder verknüpften bzw. Sparse-Checkout vermeiden Agenten die direkte lokale Ausführung von
`pnpm test*` / `pnpm check*` / `pnpm crabbox:run`:

- Begrenzter fokussierter Nachweis bei bereiten Abhängigkeiten:
  `node scripts/run-vitest.mjs <path-or-filter>`.
- Prüfung von Änderungen mit vorheriger Klassifizierung: `node scripts/check-changed.mjs`; reine Dokumentations-,
  unveränderte und kleine Metadatenpläne bleiben lokal, wenn die Abhängigkeiten bereit sind,
  während aufwendige Pläne oder solche mit fehlenden Abhängigkeiten an die Testbox delegiert werden.
- Expliziter breiter Nachweis mit beibehaltener Lease: `node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox ... -- env OPENCLAW_CHECK_CHANGED_REMOTE_CHILD=1 OPENCLAW_CHANGED_LANES_RAW_SYNC=1 corepack pnpm check:changed`, sodass pnpm innerhalb der Testbox ausgeführt wird.
- Die abschließende `exitCode`-Meldung und das Zeitmessungs-JSON des Wrappers bilden das Befehlsergebnis. Ein delegierter Blacksmith-GitHub-Actions-Lauf kann nach einem erfolgreichen SSH-Befehl `cancelled` anzeigen, weil die Testbox außerhalb der Keepalive-Action gestoppt wird; prüfen Sie die Wrapper-Zusammenfassung und die Befehlsausgabe, bevor Sie dies als Fehler behandeln.
- `OPENCLAW_HEAVY_CHECK_LOCK_SCOPE=worktree <local-heavy-check command>`: Behält die Serialisierung aufwendiger Prüfungen innerhalb des aktuellen Worktrees statt im gemeinsamen Git-Verzeichnis bei, etwa für Befehle wie `pnpm check:changed` und gezielte `pnpm test ...`. Verwenden Sie dies nur auf leistungsfähigen lokalen Hosts, wenn Sie bewusst unabhängige Prüfungen über verknüpfte Worktrees hinweg ausführen.

## Kernbefehle

Läufe des Test-Wrappers enden mit einer kurzen `[test] passed|failed|skipped ... in ...`-Zusammenfassung; die eigene Dauerzeile von Vitest bleibt die Detailangabe pro Shard.

| Befehl                                            | Funktion                                                                                                                                                                                                                                                                                                                                                        |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm test`                                       | Explizite Datei-/Verzeichnisziele werden durch bereichsspezifische Vitest-Lanes geleitet. Läufe ohne Ziel dienen als Nachweis der vollständigen Suite: Feste Shard-Gruppen werden für die lokale parallele Ausführung zu Blattkonfigurationen erweitert, wobei der erwartete Shard-Fan-out vor dem Start ausgegeben wird. Die Erweiterungsgruppe wird stets in Shard-Konfigurationen pro Erweiterung aufgeteilt, statt einen einzigen riesigen Root-Projekt-Prozess zu verwenden. |
| `pnpm test:changed`                               | Kostengünstiger intelligenter Lauf geänderter Tests: präzise Ziele aus direkten Teständerungen, benachbarten `*.test.ts`-Dateien, expliziten Quellzuordnungen und dem lokalen Importgraphen. Breite Konfigurations-/Paketänderungen werden übersprungen, sofern sie keinen präzisen Tests zugeordnet werden können.                                                                                                              |
| `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` | Expliziter breiter Lauf geänderter Tests; verwenden Sie ihn, wenn Änderungen am Test-Harness, an der Konfiguration oder am Paket auf das breitere Verhalten von Vitest für geänderte Tests zurückfallen sollen.                                                                                                                                                  |
| `pnpm test:force`                                 | Gibt den konfigurierten OpenClaw-Gateway-Port frei (Standard: `18789`) und führt anschließend die vollständige Suite mit einem isolierten Gateway-Port aus, damit Servertests nicht mit einer laufenden Instanz kollidieren.                                                                                                                                    |
| `pnpm test:coverage`                              | Erstellt einen informativen V8-Coverage-Bericht für die standardmäßige Unit-Lane (`vitest.unit.config.ts`); es werden keine Coverage-Schwellenwerte erzwungen.                                                                                                                                                                                                      |
| `pnpm test:coverage:changed`                      | Nur Unit-Coverage für Dateien, die seit `origin/main` geändert wurden.                                                                                                                                                                                                                                                                                            |
| `pnpm changed:lanes`                              | Zeigt die durch den Diff gegenüber `origin/main` ausgelösten Architektur-Lanes.                                                                                                                                                                                                                                                                                  |
| `pnpm check:changed`                              | Klassifiziert die geänderten Lanes vor der Auswahl der Ausführung. Reine Dokumentations-, unveränderte und kleine Metadatenpläne bleiben lokal, wenn die Abhängigkeiten bereit sind; Pläne mit Typecheck-/Lint-Fan-out, anderen aufwendigen Lanes oder fehlenden lokalen Abhängigkeiten werden außerhalb der CI an Crabbox/Testbox delegiert. Führt Vitest nicht aus; verwenden Sie `pnpm test:changed` oder `pnpm test <target>` für Testnachweise. |

## Gemeinsamer Teststatus und Prozesshilfen

- `src/test-utils/openclaw-test-state.ts`: Verwenden Sie dies aus Vitest, wenn ein Test isolierte `HOME`, `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH`, eine Konfigurations-Fixture, einen Workspace, ein Agent-Verzeichnis oder einen Authentifizierungsprofilspeicher benötigt.
- `pnpm test:env-mutations:report`: Nicht blockierender Bericht über Tests/Harnesses, die `HOME`, `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH`, `OPENCLAW_WORKSPACE_DIR` oder zugehörige Umgebungsschlüssel direkt verändern. Verwenden Sie ihn, um Migrationskandidaten für den gemeinsamen Teststatus-Helfer zu ermitteln.
- `test/helpers/openclaw-test-instance.ts`: E2E-Tests auf Prozessebene, die einen laufenden Gateway, eine CLI-Umgebung, Protokollerfassung und Bereinigung an zentraler Stelle benötigen.
- Docker-/Bash-E2E-Lanes, die `scripts/lib/docker-e2e-image.sh` einbinden, können `docker_e2e_test_state_shell_b64 <label> <scenario>` in den Container übergeben und mit `scripts/lib/openclaw-e2e-instance.sh` dekodieren; Skripte mit mehreren Home-Verzeichnissen können `docker_e2e_test_state_function_b64` übergeben und in jedem Ablauf `openclaw_test_state_create <label> <scenario>` aufrufen. `node scripts/lib/openclaw-test-state.mjs -- create --label <name> --scenario <name> --env-file <path> --json` schreibt eine einbindbare Host-Umgebungsdatei (das `--` vor `create` verhindert, dass neuere Node-Laufzeiten `--env-file` als Node-Flag behandeln). Lanes, die einen Gateway starten, können `scripts/lib/openclaw-e2e-instance.sh` für die Auflösung des Einstiegspunkts, den Start eines OpenAI-Mocks, Vordergrund-/Hintergrundstarts, Bereitschaftsprüfungen, den Export der Statusumgebung, Protokollausgaben und die Prozessbereinigung einbinden.

## Control UI-, TUI- und Erweiterungs-Lanes

- **Gemockte E2E-Tests der Control UI:** `pnpm test:ui:e2e` führt die Vitest- und Playwright-Teststrecke aus, die die Vite Control UI startet und eine echte Chromium-Seite gegen einen gemockten Gateway-WebSocket steuert. Die Tests befinden sich in `ui/src/**/*.e2e.test.ts`; gemeinsame Mocks und Steuerungen befinden sich in `ui/src/test-helpers/control-ui-e2e.ts`. `pnpm test:e2e` schließt diese Teststrecke ein. Agent-Ausführungen verwenden standardmäßig Testbox/Crabbox, einschließlich gezielter Nachweise; verwenden Sie `node scripts/run-vitest.mjs run --config test/vitest/vitest.ui-e2e.config.ts --configLoader runner ui/src/ui/e2e/chat-flow.e2e.test.ts` nur für einen ausdrücklich festgelegten lokalen Fallback.
- **TUI-PTY-Tests:** `node scripts/run-vitest.mjs run --config test/vitest/vitest.tui-pty.config.ts` führt die schnelle PTY-Teststrecke mit einem simulierten Backend aus. `OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1` oder `pnpm tui:pty:test:watch --mode local` führt den langsameren `tui --local`-Smoke-Test aus, der nur den externen Modellendpunkt mockt. Prüfen Sie stabilen sichtbaren Text oder Fixture-Aufrufe, keine unverarbeiteten ANSI-Snapshots.
- `pnpm test:extensions` und `pnpm test extensions` führen alle Erweiterungs-/Plugin-Shards aus. Ressourcenintensive Kanal-Plugins, das Browser-Plugin und OpenAI werden als dedizierte Shards ausgeführt; andere Plugin-Gruppen bleiben gebündelt. `pnpm test extensions/<id>` führt die Teststrecke eines gebündelten Plugins aus.
- Quelldateien mit zugehörigen Tests werden zunächst diesen Tests zugeordnet, bevor auf umfassendere Verzeichnis-Globs zurückgegriffen wird. Änderungen an Hilfsfunktionen unter `src/channels/plugins/contracts/test-helpers`, `src/plugin-sdk/test-helpers` und `src/plugins/contracts` verwenden einen lokalen Importgraphen, um importierende Tests auszuführen, statt alle Shards umfassend auszuführen, wenn der Abhängigkeitspfad eindeutig ist.
- Ziele in Vertragsverzeichnissen werden auf ihre Vertragsteststrecken verteilt: `pnpm test src/channels/plugins/contracts` führt die vier Konfigurationen für Kanalverträge aus und `pnpm test src/plugins/contracts` führt die Konfiguration für Plugin-Verträge aus, da die generischen Projekte `channels`/`plugins` `contracts/**` ausschließen.
- `auto-reply` wird in drei dedizierte Konfigurationen (`core`, `top-level`, `reply`) aufgeteilt, damit das Antwort-Testsystem nicht die leichteren übergeordneten Status-/Token-/Hilfsfunktionstests dominiert.
- Ausgewählte Testdateien unter `plugin-sdk` und `commands` werden über dedizierte schlanke Teststrecken geleitet, die nur `test/setup.ts` beibehalten, während laufzeitintensive Fälle auf ihren vorhandenen Teststrecken verbleiben.
- Die grundlegende Vitest-Konfiguration verwendet standardmäßig `pool: "threads"` und `isolate: false`, wobei der gemeinsame nicht isolierte Runner in allen Repository-Konfigurationen aktiviert ist.
- `pnpm test:channels` führt `vitest.channels.config.ts` aus.

## Gateway und E2E

- Die Gateway-Integration ist optional zu aktivieren: `OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test` oder `pnpm test:gateway`.
- `pnpm test:e2e`: Repository-E2E-Aggregat = `pnpm test:e2e:gateway && pnpm test:ui:e2e`.
- `pnpm test:e2e:gateway`: Gateway-End-to-End-Smoke-Tests (WS/HTTP/Node-Kopplung mit mehreren Instanzen). Verwendet standardmäßig `threads` + `isolate: false` mit adaptiven Workern in `vitest.e2e.config.ts`; Anpassung mit `OPENCLAW_E2E_WORKERS=<n>`, ausführliche Protokolle mit `OPENCLAW_E2E_VERBOSE=1`.
- `pnpm test:live`: Live-Tests für Provider (Claude/Minimax/DeepSeek/z.ai/usw., gesteuert durch `*.live.test.ts`). Erfordert API-Schlüssel und `LIVE=1` (oder `OPENCLAW_LIVE_TEST=1`), um die Tests nicht zu überspringen; ausführliche Ausgabe mit `OPENCLAW_LIVE_TEST_QUIET=0`.

## Vollständige Docker-Suite (`pnpm test:docker:all`)

Erstellt das gemeinsam genutzte Live-Test-Image, packt OpenClaw einmal als npm-Tarball, erstellt/verwendet erneut ein minimales Node-/Git-Runner-Image sowie ein funktionales Image, das diesen Tarball in `/app` installiert, und führt anschließend Docker-Smoke-Teststrecken über einen gewichteten Scheduler aus. `scripts/package-openclaw-for-docker.mjs` ist der einzige lokale/CI-Paket-Packer und validiert den Tarball sowie `dist/postinstall-inventory.json`, bevor Docker ihn verwendet.

- Minimales Image (`OPENCLAW_DOCKER_E2E_BARE_IMAGE`): Teststrecken für Installation, Aktualisierung und Plugin-Abhängigkeiten; bindet den vorab erstellten Tarball ein, statt kopierte Repository-Quellen zu verwenden.
- Funktionales Image (`OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE`): Teststrecken für die normale Funktionalität der erstellten Anwendung.
- Definitionen der Teststrecken: `scripts/lib/docker-e2e-scenarios.mjs`. Planer: `scripts/lib/docker-e2e-plan.mjs`. Ausführungsmodul: `scripts/test-docker-all.mjs`.
- `node scripts/test-docker-all.mjs --plan-json` gibt den CI-Plan im Besitz des Schedulers aus (Teststrecken, Image-Typen, Anforderungen an Paket-/Live-Images, Zustandsszenarien, Anmeldedatenprüfungen), ohne Docker zu erstellen oder auszuführen.

Steuerungsoptionen für die Planung (Umgebungsvariablen, Standardwerte in Klammern):

| Umgebungsvariable                                                                                               | Standardwert        | Zweck                                                                                                                                                                                                                                                                                     |
| --------------------------------------------------------------------------------------------------------------- | ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_DOCKER_ALL_PARALLELISM`                                                                               | 10                  | Prozess-Slots.                                                                                                                                                                                                                                                                            |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM`                                                                          | 10                  | Provider-sensitiver Tail-Pool.                                                                                                                                                                                                                                                            |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT`                                                                                | 9                   | Obergrenze für ressourcenintensive Live-Provider-Teststrecken.                                                                                                                                                                                                                            |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT`                                                                                 | 5                   | Obergrenze für Teststrecken mit npm-Ressourcen.                                                                                                                                                                                                                                           |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT`                                                                             | 7                   | Obergrenze für Teststrecken mit Dienstressourcen.                                                                                                                                                                                                                                         |
| `OPENCLAW_DOCKER_ALL_LIVE_CLAUDE_LIMIT` / `_CODEX_LIMIT` / `_GEMINI_LIMIT` / `_DROID_LIMIT` / `_OPENCODE_LIMIT` | 4                   | Provider-spezifische Obergrenzen für ressourcenintensive Teststrecken.                                                                                                                                                                                                                     |
| `OPENCLAW_DOCKER_ALL_LIVE_OPENAI_LIMIT` / `_TELEGRAM_LIMIT`                                                     | 1                   | Engere Provider-spezifische Obergrenzen.                                                                                                                                                                                                                                                  |
| `OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` / `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT`                                         | -                   | Überschreibung für größere Hosts.                                                                                                                                                                                                                                                         |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS`                                                                          | 2000                | Verzögerung zwischen dem Start von Teststrecken; vermeidet lokale Erstellungsstürme des Docker-Daemons.                                                                                                                                                                                    |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS`                                                                           | 7,200,000 (120 min) | Fallback-Zeitüberschreitung pro Teststrecke; ausgewählte Live-/Tail-Teststrecken verwenden engere Obergrenzen.                                                                                                                                                                             |
| `OPENCLAW_DOCKER_ALL_LIVE_RETRIES`                                                                              | 1                   | Wiederholungsversuche bei vorübergehenden Live-Provider-Fehlern.                                                                                                                                                                                                                           |
| `OPENCLAW_DOCKER_ALL_DRY_RUN`                                                                                   | off                 | Gibt das Teststreckenmanifest aus, ohne Docker auszuführen.                                                                                                                                                                                                                               |
| `OPENCLAW_DOCKER_ALL_STATUS_INTERVAL_MS`                                                                        | 30000               | Intervall für die Statusausgabe aktiver Teststrecken.                                                                                                                                                                                                                                     |
| `OPENCLAW_DOCKER_ALL_TIMINGS`                                                                                   | on                  | Verwendet `.artifacts/docker-tests/lane-timings.json` erneut für die Sortierung nach längster Laufzeit zuerst; zum Deaktivieren auf `0` setzen.                                                                                                                                           |
| `OPENCLAW_DOCKER_ALL_LIVE_MODE`                                                                                 | -                   | `skip` nur für deterministische/lokale Teststrecken, `only` nur für Live-Provider-Teststrecken. Aliasse: `pnpm test:docker:local:all`, `pnpm test:docker:live:all`. Der reine Live-Modus führt die Haupt- und Tail-Live-Teststrecken in einem einzigen, nach längster Laufzeit sortierten Pool zusammen, sodass Provider-Buckets Claude-/Codex-/Gemini-Aufgaben gemeinsam bündeln. |
| `OPENCLAW_LIVE_CLI_BACKEND_SETUP_TIMEOUT_SECONDS`                                                               | 180                 | Zeitüberschreitung für die Docker-Einrichtung des CLI-Backends.                                                                                                                                                                                                                           |

Das Muster für Umgebungsvariablen zur Begrenzung von Ressourcen lautet `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT` (Ressourcenname in Großbuchstaben, nicht alphanumerische Zeichen zu `_` zusammengefasst).

Weiteres Verhalten: Der Runner führt standardmäßig einen Docker-Preflight durch, bereinigt veraltete OpenClaw-E2E-Container, teilt Caches für Provider-CLI-Tools zwischen kompatiblen Lanes und plant nach dem ersten Fehler keine neuen gepoolten Lanes mehr ein, sofern `OPENCLAW_DOCKER_ALL_FAIL_FAST=0` nicht gesetzt ist. Wenn eine Lane die effektive Gewichtungs-/Ressourcenobergrenze auf einem Host mit geringer Parallelität überschreitet, kann sie dennoch aus einem leeren Pool starten und allein ausgeführt werden, bis sie Kapazität freigibt. Pro-Lane-Protokolle, `summary.json`, `failures.json` und Phasenzeitmessungen werden unter `.artifacts/docker-tests/<run-id>/` geschrieben; verwenden Sie `pnpm test:docker:timings <summary.json>`, um langsame Lanes zu untersuchen, und `pnpm test:docker:rerun <run-id|summary.json|failures.json>`, um einfache gezielte Befehle für erneute Ausführungen auszugeben.

### Erwähnenswerte Docker-Lanes

| Befehl                                                                     | Überprüft                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm test:docker:browser-cdp-snapshot`                                     | Chromium-gestützter Quell-E2E-Container mit unverarbeitetem CDP und isoliertem Gateway; `browser doctor --deep`-CDP-Rollen-Snapshots enthalten Link-URLs, durch den Cursor als anklickbar erkannte Elemente, Iframe-Referenzen und Frame-Metadaten.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `pnpm test:docker:skill-install`                                            | Installiert den gepackten Tarball in einem unbestückten Docker-Runner mit `skills.install.allowUploadedArchives: false`, ermittelt über eine Live-ClawHub-Suche einen aktuellen Skill-Slug, installiert ihn über `openclaw skills install` und überprüft `SKILL.md`, `.clawhub/origin.json`, `.clawhub/lock.json` und `skills info --json`.                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `pnpm test:docker:live-cli-backend:claude`, `:claude:resume`, `:claude:mcp` | Gezielte Live-Prüfungen der CLI-Backends; Gemini verfügt über die entsprechenden Aliasse `:resume` und `:mcp`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `pnpm test:docker:openwebui`                                                | Dockerisiertes OpenClaw + Open WebUI: anmelden, `/api/models` prüfen und einen echten, über `/api/chat/completions` weitergeleiteten Chat ausführen. Erfordert einen verwendbaren Live-Modellschlüssel und lädt ein externes Image herunter; es wird keine mit den Unit-/E2E-Suites vergleichbare CI-Stabilität erwartet.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `pnpm test:docker:mcp-channels`                                             | Vorbereiteter Gateway-Container sowie ein Client-Container, der `openclaw mcp serve` startet: Erkennung weitergeleiteter Unterhaltungen, Lesen von Transkripten, Anhangsmetadaten, Verhalten der Live-Ereigniswarteschlange, Weiterleitung ausgehender Sendungen sowie Kanal- und Berechtigungsbenachrichtigungen im Claude-Stil über die echte stdio-Bridge (die Assertion liest unverarbeitete stdio-MCP-Frames direkt).                                                                                                                                                                                                                                                                                                                                                                                                               |
| `pnpm test:docker:upgrade-survivor`                                         | Installiert den gepackten Tarball über eine veraltete Fixture eines bestehenden Benutzers, führt ohne Live-Schlüssel für Provider/Kanäle eine Paketaktualisierung sowie Doctor nicht interaktiv aus, startet ein Loopback-Gateway und prüft, ob Agenten-/Kanalkonfiguration, Plugin-Zulassungslisten, Workspace-/Sitzungsdateien, veralteter Legacy-Zustand der Plugin-Abhängigkeiten, Start und RPC-Status erhalten bleiben.                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `pnpm test:docker:published-upgrade-survivor`                               | Installiert standardmäßig `openclaw@latest`, legt realistische Dateien eines bestehenden Benutzers an, konfiguriert über ein integriertes `openclaw config set`-Rezept, aktualisiert auf den gepackten Tarball, führt Doctor nicht interaktiv aus, schreibt `.artifacts/upgrade-survivor/summary.json` und prüft `/healthz`, `/readyz` sowie den RPC-Status. Überschreiben Sie dies mit `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC`, erweitern Sie eine Matrix mit `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` oder fügen Sie Szenario-Fixtures mit `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS=reported-issues` hinzu (enthält `configured-plugin-installs` und `stale-source-plugin-shadow`). Package Acceptance stellt diese als `published_upgrade_survivor_baseline(s)` / `_scenarios` bereit und löst Meta-Tokens wie `last-stable-4` oder `all-since-2026.4.23` auf. |
| `pnpm test:docker:update-migration`                                         | Test-Harness für das Überstehen veröffentlichter Upgrades im Szenario `plugin-deps-cleanup`, das standardmäßig bei `openclaw@2026.4.23` beginnt. Der Workflow `Update Migration` erweitert dies mit `baselines=all-since-2026.4.23`, um die Bereinigung von Abhängigkeiten konfigurierter Plugins außerhalb der vollständigen Release-CI nachzuweisen.                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `pnpm test:docker:plugins`                                                  | Installations-/Aktualisierungs-Smoke-Test für lokalen Pfad, `file:`, npm-Registry-Pakete mit hochgezogenen Abhängigkeiten, veränderliche Git-Referenzen, ClawHub-Fixtures, Marketplace-Aktualisierungen sowie Aktivierung/Inspektion des Claude-Bundles.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

## Lokales PR-Gate

Führen Sie für lokale Prüfungen zum Landen/Gaten eines PR Folgendes aus:

- `pnpm check:changed`
- `pnpm check`
- `pnpm check:test-types`
- `pnpm build`
- `pnpm test`
- `pnpm check:docs`

Wenn `pnpm test` auf einem ausgelasteten Host sporadisch fehlschlägt, führen Sie es einmal erneut aus, bevor Sie es als Regression behandeln, und isolieren Sie es anschließend mit `pnpm test <path/to/test>`. Für Hosts mit begrenztem Arbeitsspeicher:

- `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test`
- `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/openclaw-vitest-cache pnpm test:changed`

## Tools zur Testleistungsanalyse

- `pnpm test:perf:imports`: Aktiviert die Berichterstellung zu Vitest-Importdauer und -Importaufschlüsselung, während für explizite Datei-/Verzeichnisziele weiterhin bereichsspezifisches Lane-Routing verwendet wird. `pnpm test:perf:imports:changed` beschränkt dasselbe Profiling auf Dateien, die seit `origin/main` geändert wurden.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` vergleicht die Leistung des gerouteten Änderungsmodus-Pfads mit der nativen Ausführung des Root-Projekts für denselben committeten Git-Diff; `pnpm test:perf:changed:bench -- --worktree` misst die Leistung der aktuellen Worktree-Änderungsmenge, ohne sie zuvor zu committen.
- `pnpm test:perf:profile:main` schreibt ein CPU-Profil für den Vitest-Hauptthread (`.artifacts/vitest-main-profile`); `pnpm test:perf:profile:runner` schreibt CPU- und Heap-Profile für den Unit-Runner (`.artifacts/vitest-runner-profile`).
- `pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json`: Führt jede Vitest-Leaf-Konfiguration der vollständigen Suite seriell aus und schreibt gruppierte Laufzeitdaten sowie JSON-/Protokollartefakte pro Konfiguration. Berichte für vollständige Suites isolieren Dateien standardmäßig, damit beibehaltene Modulgraphen und GC-Pausen aus früheren Dateien nicht späteren Assertions zugerechnet werden; übergeben Sie `-- --no-isolate` nur, wenn Sie die Akkumulation gemeinsam genutzter Worker bewusst profilieren. Der Test Performance Agent verwendet dies als Ausgangsbasis, bevor er Korrekturen für langsame Tests versucht. `pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json` vergleicht gruppierte Berichte nach einer leistungsorientierten Änderung.
- Ausführungen vollständiger Suites, von Erweiterungen und von Include-Pattern-Shards aktualisieren lokale Zeitmessungsdaten in `.artifacts/vitest-shard-timings.json`; spätere Ausführungen vollständiger Konfigurationen verwenden diese Zeitmessungen, um langsame und schnelle Shards auszubalancieren. Include-Pattern-CI-Shards hängen den Shard-Namen an den Zeitmessungsschlüssel an, sodass die Zeitmessungen gefilterter Shards sichtbar bleiben, ohne die Zeitmessungsdaten vollständiger Konfigurationen zu ersetzen. Setzen Sie `OPENCLAW_TEST_PROJECTS_TIMINGS=0`, um das lokale Zeitmessungsartefakt zu ignorieren.

## Benchmarks

<Accordion title="Modelllatenz (scripts/bench-model.ts)">

```bash
pnpm tsx scripts/bench-model.ts --runs 10
```

Optionale Umgebungsvariablen: `MINIMAX_API_KEY`, `MINIMAX_BASE_URL`, `MINIMAX_MODEL`, `ANTHROPIC_API_KEY`. Standard-Prompt: „Antworten Sie mit einem einzigen Wort: ok. Keine Satzzeichen oder zusätzlicher Text.“

</Accordion>

<Accordion title="CLI-Start (scripts/bench-cli-startup.ts)">

```bash
pnpm test:startup:bench
pnpm test:startup:bench:smoke
pnpm test:startup:bench:save
pnpm test:startup:bench:update
pnpm test:startup:bench:check
pnpm tsx scripts/bench-cli-startup.ts --runs 12
pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --case gatewayStatus --runs 3
pnpm tsx scripts/bench-cli-startup.ts --entry openclaw.mjs --entry-secondary dist/entry.js --preset all
```

Voreinstellungen:

- `startup`: `--version`, `--help`, `health`, `health --json`, `status --json`, `status`
- `real`: `health`, `status`, `status --json`, `sessions`, `sessions --json`, `tasks --json`, `tasks list --json`, `tasks audit --json`, `agents list --json`, `gateway status`, `gateway status --json`, `gateway health --json`, `config get gateway.port`
- `all`: beide Voreinstellungen kombiniert

Die Ausgabe enthält `sampleCount`, Durchschnitt, p50, p95, Minimum/Maximum, Verteilung der Exit-Codes/Signale und den maximalen RSS-Wert pro Befehl. `--cpu-prof-dir` / `--heap-prof-dir` schreiben V8-Profile für jeden Durchlauf.

Gespeicherte Ausgabe: `pnpm test:startup:bench:smoke` schreibt `.artifacts/cli-startup-bench-smoke.json`; `pnpm test:startup:bench:save` schreibt `.artifacts/cli-startup-bench-all.json` (`runs=5 warmup=1`). Eingecheckte Fixture: `test/fixtures/cli-startup-bench.json`, aktualisiert durch `pnpm test:startup:bench:update`, verglichen durch `pnpm test:startup:bench:check`.

</Accordion>

<Accordion title="Gateway-Start (scripts/bench-gateway-startup.ts)">

Verwendet standardmäßig den gebauten CLI-Einstiegspunkt unter `dist/entry.js`; führen Sie zuerst `pnpm build` aus. Übergeben Sie `--entry scripts/run-node.mjs`, um stattdessen den Source-Runner zu messen, und halten Sie diese Ergebnisse von den Baselines des gebauten Einstiegspunkts getrennt.

```bash
pnpm test:startup:gateway -- --runs 5 --warmup 1
pnpm test:startup:gateway -- --case skipChannels --case fiftyPlugins --runs 5
node --import tsx scripts/bench-gateway-startup.ts --case default --runs 5 --output .artifacts/gateway-startup.json
```

Fall-IDs: `default`, `skipChannels` (Kanalstart übersprungen), `oneInternalHook`, `allInternalHooks`, `fiftyPlugins` (50 Manifest-Plugins), `fiftyStartupLazyPlugins` (50 beim Start verzögert geladene Manifest-Plugins).

Die Ausgabe enthält die erste Prozessausgabe, `/healthz`, `/readyz`, die Zeit des HTTP-Listen-Logs, die Zeit des Gateway-Bereitschaftslogs, CPU-Zeit, CPU-Kern-Verhältnis, maximalen RSS-Wert, Heap, Metriken der Startablaufverfolgung, Event-Loop-Verzögerung und detaillierte Metriken der Plugin-Lookup-Tabelle. Das Skript setzt `OPENCLAW_GATEWAY_STARTUP_TRACE=1` in der Umgebung des untergeordneten Gateways.

`/healthz` bezeichnet die Betriebsfähigkeit (der HTTP-Server kann antworten). `/readyz` bezeichnet die nutzbare Bereitschaft (Plugin-Sidecars beim Start, Kanäle und für die Bereitschaft kritische Arbeiten nach dem Anhängen sind abgeschlossen). Start-Hooks werden asynchron ausgelöst und sind nicht Teil der Bereitschaftsgarantie. Die Zeit des Bereitschaftslogs ist der interne Zeitstempel des Gateways; sie ist für die prozessseitige Zuordnung nützlich, ersetzt jedoch nicht die externe `/readyz`-Prüfung.

Verwenden Sie beim Vergleich von Änderungen die JSON-Ausgabe oder `--output`. Verwenden Sie `--cpu-prof-dir` nur, nachdem die Ablaufverfolgungsausgabe auf Import-, Kompilierungs- oder CPU-gebundene Arbeit hinweist, die sich allein durch Phasenzeitmessungen nicht erklären lässt.

</Accordion>

<Accordion title="Gateway-Neustart (scripts/bench-gateway-restart.ts)">

Nur macOS und Linux (verwendet SIGUSR1 für prozessinterne Neustarts; schlägt unter Windows sofort fehl). Derselbe standardmäßig gebaute Einstiegspunkt und dieselbe `--entry scripts/run-node.mjs`-Überschreibung wie beim Gateway-Start oben.

```bash
pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5
pnpm test:restart:gateway -- --case default --runs 3 --restarts 3 --warmup 1
```

Fall-IDs: `skipChannels`, `skipChannelsAcpxProbe` (ACPX-Startprüfung aktiviert), `skipChannelsNoAcpxProbe` (Prüfung deaktiviert), `default`, `fiftyPlugins`.

Die Ausgabe enthält das nächste `/healthz`, das nächste `/readyz`, Ausfallzeit, Zeitmessung der Neustartbereitschaft, CPU, RSS, Metriken der Startablaufverfolgung für den Ersatzprozess sowie Metriken der Neustartablaufverfolgung für Signalverarbeitung, das Leeren aktiver Arbeiten, Schließphasen, den nächsten Start, die Bereitschaftszeitmessung und Speicher-Snapshots. Das Skript setzt `OPENCLAW_GATEWAY_STARTUP_TRACE=1` und `OPENCLAW_GATEWAY_RESTART_TRACE=1`.

Verwenden Sie diesen Benchmark, wenn eine Änderung die Neustartsignalisierung, Schließ-Handler, den Start nach einem Neustart, das Herunterfahren von Sidecars, die Dienstübergabe oder die Bereitschaft nach einem Neustart betrifft. Beginnen Sie mit `skipChannels`, um die Gateway-Mechanik vom Kanalstart zu isolieren; verwenden Sie `default` oder Plugin-intensive Fälle erst, nachdem der eng gefasste Fall den Neustartpfad erklärt hat. Ablaufverfolgungsmetriken sind Hinweise zur Zuordnung, keine abschließenden Bewertungen — beurteilen Sie eine Neustartänderung anhand mehrerer Stichproben, des passenden Owner-Spans, des Verhaltens von `/healthz`/`/readyz` und des für Benutzer sichtbaren Neustartvertrags.

</Accordion>

## Onboarding-E2E (Docker)

Optional; nur für containerisierte Onboarding-Smoke-Tests erforderlich. Vollständiger Kaltstartablauf in einem sauberen Linux-Container:

```bash
scripts/e2e/onboard-docker.sh
```

Steuert den interaktiven Assistenten über ein Pseudo-TTY, überprüft Konfigurations-, Workspace- und Sitzungsdateien, startet anschließend das Gateway und führt `openclaw health` aus.

## QR-Import-Smoke-Test (Docker)

Stellt sicher, dass der gepflegte QR-Laufzeithelfer unter den unterstützten Docker-Node-Laufzeiten geladen wird (standardmäßig Node 24, kompatibel mit Node 22):

```bash
pnpm test:docker:qr
```

## Verwandte Themen

- [Tests](/de/help/testing)
- [Live-Tests](/de/help/testing-live)
- [Tests von Updates und Plugins](/de/help/testing-updates-plugins)
