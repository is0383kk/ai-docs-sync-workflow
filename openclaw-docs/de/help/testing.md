---
read_when:
    - Tests lokal oder in der CI ausführen
    - Regressionstests für Modell-/Provider-Fehler hinzufügen
    - Debugging des Gateway- und Agentenverhaltens
summary: 'Testkit: Unit-/E2E-/Live-Suiten, Docker-Runner und Testabdeckung der einzelnen Tests'
title: Testen
x-i18n:
    generated_at: "2026-07-26T18:29:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 20e0aa22bf16561334f83342abffabb387ed0b41b901773939123ecfbc0ae330
    source_path: help/testing.md
    workflow: 16
---

OpenClaw verfügt über drei Vitest-Suites (Unit/Integration, E2E, Live) sowie Docker-
Runner. Diese Seite erläutert, was die einzelnen Suites abdecken, welcher Befehl für einen
bestimmten Workflow auszuführen ist, wie Live-Tests Anmeldedaten erkennen und wie
Regressionstests für reale Provider-/Modellfehler hinzugefügt werden.

<Note>
Der **QA-Stack (qa-lab, qa-channel, Live-Transport-Lanes)** wird separat dokumentiert:

- [QA-Übersicht](/de/concepts/qa-e2e-automation) – Architektur, Befehlsoberfläche, Szenarioerstellung und Matrix-Profile.
- [Reifegrad-Scorecard](/de/maturity/scorecard) – wie QA-Nachweise aus Releases Stabilitäts- und LTS-Entscheidungen unterstützen.
- [QA-Kanal](/de/channels/qa-channel) – das synthetische Transport-Plugin für Repository-gestützte Szenarien.

Diese Seite behandelt die regulären Test-Suites und Docker-/Parallels-Runner. Die nachstehenden [QA-spezifischen Runner](#qa-specific-runners) führen die konkreten `qa`-Aufrufe auf und verweisen auf die oben genannten Referenzen.
</Note>

## Schnellstart

An den meisten Tagen:

- Vollständiger Gate-Lauf (vor dem Push erwartet): `pnpm build && pnpm check && pnpm check:test-types && pnpm test`
- Schnellerer lokaler Lauf der vollständigen Suite auf einem leistungsfähigen Rechner: `pnpm test:max`
- Direkte Vitest-Watch-Schleife: `pnpm test:watch`
- Die direkte Dateiauswahl leitet auch Plugin-/Kanalpfade weiter: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Bei der Bearbeitung eines einzelnen Fehlers sollten zuerst gezielte Läufe verwendet werden.
- Docker-gestützte QA-Site: `pnpm qa:lab:up`
- Linux-VM-gestützte QA-Lane: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

Wenn Sie Tests ändern oder zusätzliche Sicherheit benötigen:

- Informativer V8-Coverage-Bericht: `pnpm test:coverage`
- E2E-Suite: `pnpm test:e2e`

## Temporäre Testverzeichnisse

Verwenden Sie die gemeinsamen Hilfsfunktionen in `test/helpers/temp-dir.ts` für temporäre
testeigene Verzeichnisse, damit die Zuständigkeit eindeutig ist und die Bereinigung im Testlebenszyklus verbleibt:

```ts
import { afterEach } from "vitest";
import { useAutoCleanupTempDirTracker } from "../helpers/temp-dir.js";

const tempDirs = useAutoCleanupTempDirTracker(afterEach);

it("verwendet einen temporären Arbeitsbereich", () => {
  const workspace = tempDirs.make("openclaw-example-");
  // Arbeitsbereich verwenden
});
```

`useAutoCleanupTempDirTracker(afterEach)` stellt absichtlich keine manuelle
Bereinigungsmethode bereit – Vitest übernimmt die Bereinigung nach jedem Test. Ältere, systemnähere
Hilfsfunktionen (`makeTempDir`, `cleanupTempDirs`, `createTempDirTracker`) sind für
noch nicht migrierte Tests weiterhin vorhanden; vermeiden Sie deren neue Verwendung sowie neue direkte
`fs.mkdtemp*`-Aufrufe, sofern ein Test nicht ausdrücklich das unverarbeitete Verhalten temporärer
Verzeichnisse überprüft. Wenn ein direkt erstelltes temporäres Verzeichnis tatsächlich erforderlich ist, fügen Sie einen überprüfbaren Zulassungskommentar
mit Begründung hinzu:

```ts
// openclaw-temp-dir: allow überprüft das unverarbeitete fs-Bereinigungsverhalten
const workspace = fs.mkdtempSync(prefix);
```

`node scripts/report-test-temp-creations.mjs` meldet neu hinzugefügte direkte Erstellungen temporärer Verzeichnisse
und neue manuelle Verwendungen gemeinsamer Hilfsfunktionen in hinzugefügten Diff-Zeilen, ohne
bestehende Bereinigungsstile zu blockieren. Dabei wird dieselbe Testpfadklassifizierung
wie bei `scripts/changed-lanes.mjs` verwendet und die Implementierung der gemeinsamen Hilfsfunktion
selbst übersprungen. `check:changed` führt diesen Bericht für geänderte Testpfade als
reines CI-Warnsignal aus (GitHub-Warnanmerkungen, keine Fehler).

## Live- und Docker-/Parallels-Workflows

Beim Debuggen realer Provider/Modelle (erfordert echte Anmeldedaten):

- Live-Suite (Modelle sowie Gateway-Tool-/Bild-Probes): `pnpm test:live`
- Eine einzelne Live-Datei mit reduzierter Ausgabe ausführen: `pnpm test:live -- src/agents/models.profiles.live.test.ts`
- Berichte zur Laufzeitleistung: Starten Sie `OpenClaw Performance` mit
  `live_openai_candidate=true` für einen echten `openai/gpt-5.6-luna`-Agent-Turn oder
  `deep_profile=true` für Kova-CPU-/Heap-/Trace-Artefakte. Täglich geplante Läufe
  veröffentlichen Berichte für Mock-Provider-, Deep-Profile- und GPT-5.6-Luna-Lanes in
  `openclaw/clawgrit-reports` über einen separaten, Artefakte verarbeitenden Publisher-Job;
  fehlende oder ungültige Publisher-Authentifizierung führt bei geplanten und
  `profile=release`-Läufen zu einem Fehler. Manuelle Dispatches außerhalb von Releases behalten die GitHub-Artefakte
  bei und behandeln die Berichtsveröffentlichung als unverbindlich. Der Mock-Provider-Bericht enthält außerdem
  Messwerte für Gateway-Start auf Quellcodeebene, Arbeitsspeicher, Plugin-Auslastung, wiederholte
  Fake-Modell-Hello-Schleifen und CLI-Start.
- Docker-Live-Modell-Sweep: `pnpm test:docker:live-models`
  - Jedes ausgewählte Modell führt einen Text-Turn sowie eine kleine Probe nach Art eines Dateilesevorgangs aus.
    Modelle, deren Metadaten `image`-Eingaben ausweisen, führen außerdem einen kleinen Bild-Turn aus.
    Deaktivieren Sie die zusätzlichen Probes mit `OPENCLAW_LIVE_MODEL_FILE_PROBE=0` oder
    `OPENCLAW_LIVE_MODEL_IMAGE_PROBE=0`, wenn Sie Provider-Fehler isolieren.
  - CI-Abdeckung: Sowohl der tägliche `OpenClaw Scheduled Live And E2E Checks` als auch der manuelle
    `OpenClaw Release Checks` rufen den wiederverwendbaren Live-/E2E-Workflow mit
    `include_live_suites: true` auf, der nach Provider aufgeteilte Jobs für die Docker-Live-Modellmatrix
    enthält.
  - Für gezielte CI-Wiederholungsläufe starten Sie `OpenClaw Live And E2E Checks (Reusable)`
    mit `include_live_suites: true` und `live_models_only: true`.
  - Fügen Sie neue aussagekräftige Provider-Secrets zu `scripts/ci-hydrate-live-auth.sh`
    sowie zu `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml` und dessen
    geplanten bzw. Release-Aufrufern hinzu.
- Nativer Codex-Smoke-Test für gebundene Chats: `pnpm test:docker:live-codex-bind`
  - Führt eine Docker-Live-Lane über den Codex-App-Server-Pfad aus, bindet eine
    synthetische Slack-Direktnachricht mit `/codex bind`, führt `/codex fast` und
    `/codex permissions` aus und überprüft anschließend, dass eine einfache Antwort und ein Bildanhang
    über die native Plugin-Bindung statt über ACP weitergeleitet werden.
- Smoke-Test des Codex-App-Server-Harness: `pnpm test:docker:live-codex-harness`
  - Führt Gateway-Agent-Turns über das Plugin-eigene Codex-App-Server-
    Harness aus, überprüft `/codex status` und `/codex models` und
    führt standardmäßig Bild-, Cron-MCP-, Sub-Agent- und Guardian-Probes aus. Deaktivieren Sie die
    Sub-Agent-Probe mit `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=0`, wenn
    Sie andere Fehler isolieren. Deaktivieren Sie für eine gezielte Sub-Agent-Prüfung die
    anderen Probes:
    `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 pnpm test:docker:live-codex-harness`.
    Der Vorgang wird nach der Sub-Agent-Probe beendet, sofern
    `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_ONLY=0` nicht gesetzt ist.
- Smoke-Test der bedarfsgesteuerten Codex-Installation: `pnpm test:docker:codex-on-demand`
  - Installiert das paketierte OpenClaw-Tarball in Docker, führt das
    Onboarding mit einem OpenAI-API-Schlüssel aus und überprüft, dass das Codex-Plugin sowie die Abhängigkeit
    `@openai/codex` bei Bedarf in das Stammverzeichnis des verwalteten npm-Projekts
    heruntergeladen wurden.
- Live-Paket-Smoke-Test des Codex-npm-Plugins: `pnpm test:docker:live-codex-npm-plugin`
  - Installiert das vorgesehene OpenClaw-Paket und das exakte Codex-Plugin in Docker
    und verwendet anschließend einen echten OpenAI-Schlüssel für die CLI-Vorabprüfung und Turns in derselben Sitzung.
  - Der anschließende Turn mit mittlerer Denkintensität und ohne Wiederholungsversuche muss Fortschritt melden, die
    Arbeit durch zufällige Arbeitsbereichslesevorgänge und das exakte Schreiben eines Artefakts fortsetzen
    und anschließend den Abschluss melden. Ein terminaler Turn, der ausschließlich Fortschritt meldet, lässt die Lane fehlschlagen.
- Live-Smoke-Test für Plugin-Tool-Abhängigkeiten: `pnpm test:docker:live-plugin-tool`
  - Packt ein Fixture-Plugin mit einer echten `slugify`-Abhängigkeit, installiert es
    über `npm-pack:`, überprüft die Abhängigkeit unter dem Stammverzeichnis des verwalteten npm-
    Projekts und fordert anschließend ein Live-OpenAI-Modell auf, das Plugin-Tool aufzurufen und
    den verborgenen Slug zurückzugeben.
- Smoke-Test für den OpenClaw-Rettungsbefehl: `pnpm test:live:system-agent-rescue-channel`
  - Optionale zusätzliche Sicherheitsprüfung für die Rettungsbefehlsoberfläche
    von Nachrichtenkanälen. Führt `/openclaw status` aus, stellt eine persistente Modelländerung
    in die Warteschlange, antwortet mit `/openclaw yes` und überprüft den Schreibpfad für
    Audit und Konfiguration.
- Docker-Smoke-Test für den ersten OpenClaw-Start: `pnpm test:docker:system-agent-first-run`
  - Beginnt mit einem leeren OpenClaw-Zustandsverzeichnis und weist zunächst nach, dass die paketierte
    `openclaw setup`-CLI ohne Inferenz geschlossen fehlschlägt. Anschließend
    wird Fake Claude über das paketierte Aktivierungsmodul getestet und aktiviert.
    Erst danach erreicht eine unscharfe paketierte CLI-Anfrage den Planer und
    wird in eine typisierte Einrichtung aufgelöst, gefolgt von einmaligen Modell-, Agent-, Discord-Konfigurations-
    und SecretRef-Operationen. Dabei werden Konfigurations- und Audit-Einträge validiert. Dies sind
    unterstützende Nachweise für Gates und Operationen, keine Nachweise für interaktives Onboarding oder
    OpenClaw-Agenten, -Tools bzw. -Genehmigungen. Dieselbe Lane ist in QA Lab über
    `pnpm openclaw qa suite --scenario system-agent-ring-zero-setup` verfügbar.
- Moonshot-/Kimi-Kosten-Smoke-Test: Führen Sie bei gesetztem `MOONSHOT_API_KEY`
  zunächst `openclaw models list --provider moonshot --json` und anschließend einen isolierten
  `openclaw agent --local --session-id live-kimi-cost --message 'Reply exactly: KIMI_LIVE_OK' --thinking off --json`
  gegen `moonshot/kimi-k2.6` aus. Überprüfen Sie, dass der JSON-Bericht Moonshot/K2.6 ausweist und das
  Assistententranskript normalisierte `usage.cost` speichert.

<Tip>
Wenn Sie nur einen einzelnen fehlschlagenden Fall benötigen, grenzen Sie die Live-Tests vorzugsweise über die unten beschriebenen Allowlist-Umgebungsvariablen ein.
</Tip>

## QA-spezifische Runner

Diese Befehle ergänzen die Haupt-Test-Suites, wenn Sie die Realitätsnähe von QA Lab benötigen.

CI führt QA Lab in dedizierten Workflows aus. Agentische Parität ist unter
`QA-Lab - All Lanes` und der Release-Validierung eingebettet und kein eigenständiger PR-Workflow.
Für eine umfassende Validierung sollte `Full Release Validation` mit
`rerun_group=qa-parity` oder die QA-Gruppe der Release-Prüfungen verwendet werden. Stabile/standardmäßige Release-
Prüfungen halten umfassende Live-/Docker-Dauertests hinter `run_release_soak=true`; das
Profil `full` erzwingt die Ausführung der Dauertests. `QA-Lab - All Lanes` wird jede Nacht auf `main` sowie
über manuelle Dispatches ausgeführt, wobei die Mock-Paritäts-Lane, die Live-Matrix-Lane,
die von Convex verwaltete Live-Telegram-Lane und die von Convex verwaltete Live-Discord-Lane als
parallele Jobs laufen. Geplante QA- und Release-Prüfungen führen das Matrix-Release-Profil
über den gemeinsamen Live-Adapter aus. Der Standardwert der Matrix-CLI und der manuellen Workflow-Eingabe
bleibt `all`; manuelle `all`-Dispatches verteilen die Transport-, Medien- und
E2EE-Profile, während gezielte Dispatches `fast`, `release` oder
`transport` auswählen können. `OpenClaw Release Checks` führt vor der Release-Freigabe die Parität sowie das wiederverwendbare Matrix-
Live-Adapter-Profil und die Telegram-Lane aus. Release-
Transportprüfungen verwenden `mock-openai/gpt-5.6-luna`, damit sie deterministisch bleiben und
den normalen Start von Provider-Plugins vermeiden. Diese Live-Transport-Gateways
deaktivieren die Speichersuche; das Speicherverhalten bleibt durch die QA-Paritäts-Suites abgedeckt.

Vollständige Live-Medien-Shards für Releases verwenden
`ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04`, das bereits
`ffmpeg` und `ffprobe` enthält. Docker-Live-Modell-/Backend-Shards verwenden das gemeinsame
`ghcr.io/openclaw/openclaw-live-test:<sha>`-Image, das einmal pro ausgewähltem
Commit erstellt wird, und laden es anschließend mit `OPENCLAW_SKIP_DOCKER_BUILD=1` herunter, anstatt es
in jedem Shard erneut zu erstellen.

- `pnpm openclaw qa suite`
  - Führt Repository-gestützte QA-Szenarien direkt auf dem Host aus.
  - Schreibt übergeordnete Artefakte für `qa-evidence.json`, `qa-suite-summary.json` und
    `qa-suite-report.md` für die ausgewählte Szenariengruppe, einschließlich
    Auswahlen von gemischten Ablauf-, Vitest- und Playwright-Szenarien.
  - Bei Ausführung durch `pnpm openclaw qa run --qa-profile <profile>` wird
    die Scorecard des ausgewählten Taxonomieprofils in dasselbe `qa-evidence.json` eingebettet.
    `smoke-ci` schreibt kompakte Nachweise (`evidenceMode: "slim"`, kein
    `execution` pro Eintrag). `release` deckt den kuratierten Ausschnitt zur Release-Bereitschaft ab; `all`
    wählt jede aktive Reifekategorie aus und zielt auf explizite Ausführungen des Workflows
    „QA Profile Evidence“ ab, wenn ein vollständiges Scorecard-Artefakt benötigt wird.
  - Führt standardmäßig mehrere ausgewählte Szenarien parallel mit isolierten
    Gateway-Workern aus. `qa-channel` verwendet standardmäßig eine Parallelität von 4 (begrenzt durch die
    Anzahl der ausgewählten Szenarien). Verwenden Sie `--concurrency <count>`, um die Anzahl der Worker
    anzupassen, oder `--concurrency 1` für die ältere serielle Lane.
  - Wird mit einem von null verschiedenen Status beendet, wenn ein Szenario fehlschlägt. Verwenden Sie `--allow-failures` für
    Artefakte ohne fehleranzeigenden Exitcode.
  - Unterstützt die Provider-Modi `live-frontier`, `mock-openai` und `aimock`.
    `aimock` startet einen lokalen AIMock-gestützten Provider-Server für experimentelle
    Fixture- und Protokoll-Mock-Abdeckung, ohne die szenariobewusste
    `mock-openai`-Lane zu ersetzen.
- `pnpm openclaw qa coverage --match <query>`
  - Durchsucht Szenario-IDs, Titel, Oberflächen, Abdeckungs-IDs, Dokumentationsreferenzen, Code-
    Referenzen, Plugins und Provider-Anforderungen und gibt anschließend passende Suite-
    Ziele aus.
  - Verwenden Sie dies vor einem QA-Lab-Lauf, wenn das betroffene Verhalten oder der Dateipfad
    bekannt ist, nicht jedoch das kleinste Szenario. Nur als Empfehlung – wählen Sie weiterhin Mock-,
    Live-, Multipass-, Matrix- oder Transportnachweise entsprechend dem zu ändernden
    Verhalten aus.
- `pnpm test:plugins:kitchen-sink-live`
  - Führt den Live-Testparcours des OpenAI-Kitchen-Sink-Plugins über QA Lab aus.
    Installiert das externe Kitchen-Sink-Paket, überprüft das Oberflächeninventar des Plugin-SDK,
    prüft `/healthz` und `/readyz`, zeichnet Gateway-
    CPU/RSS-Nachweise auf, führt einen Live-OpenAI-Turn aus und prüft adversariale
    Diagnosen. Erfordert eine Live-OpenAI-Authentifizierung wie `OPENAI_API_KEY`. In
    bereitgestellten Testbox-Sitzungen wird automatisch das Live-Authentifizierungsprofil
    der Testbox eingebunden, wenn der `openclaw-testbox-env`-Helper vorhanden ist.
- `pnpm test:gateway:cpu-scenarios`
  - Führt den Gateway-Start-Benchmark sowie eine kleine Gruppe von Mock-QA-Lab-Szenarien
    (`channel-chat-baseline`, `memory-failure-fallback`,
    `gateway-restart-inflight-run`) aus und schreibt eine kombinierte Zusammenfassung der
    CPU-Beobachtungen unter `.artifacts/gateway-cpu-scenarios/`.
  - Markiert standardmäßig nur anhaltende Beobachtungen hoher CPU-Auslastung (`--cpu-core-warn`,
    Standardwert `0.9`; `--hot-wall-warn-ms`, Standardwert `30000`), sodass kurze Lastspitzen beim Start
    als Metriken aufgezeichnet werden, ohne wie die minutenlange
    Regression mit dauerhaft ausgelastetem Gateway zu wirken.
  - Wird mit erstellten `dist`-Artefakten ausgeführt; führen Sie zunächst einen Build aus, wenn der Checkout
    noch keine aktuellen Laufzeitausgaben enthält.
- `pnpm openclaw qa suite --runner multipass`
  - Führt dieselbe QA-Suite innerhalb einer kurzlebigen Multipass-Linux-VM aus und behält
    dabei dieselben Flags für Szenarioauswahl, Provider und Modell wie `qa suite` bei.
  - Live-Läufe leiten die für den Gast praktikablen QA-Authentifizierungseingaben weiter:
    umgebungsbasierte Provider-Schlüssel, den Pfad zur QA-Live-Provider-Konfiguration und
    `CODEX_HOME`, sofern vorhanden.
  - Ausgabeverzeichnisse müssen sich unterhalb des Repository-Stammverzeichnisses befinden, damit der Gast
    über den eingebundenen Arbeitsbereich zurückschreiben kann.
  - Schreibt den normalen QA-Bericht und die Zusammenfassung sowie Multipass-Protokolle unter
    `.artifacts/qa-e2e/...`.
- `pnpm qa:lab:up`
  - Startet die Docker-gestützte QA-Site für QA-Arbeiten nach Betreiberart.
- `pnpm test:docker:npm-onboard-channel-agent`
  - Erstellt aus dem aktuellen Checkout einen npm-Tarball, installiert ihn global in
    Docker, führt das nicht interaktive Onboarding mit einem OpenAI-API-Schlüssel aus, konfiguriert
    standardmäßig Telegram, überprüft, dass die paketierte Plugin-Laufzeit ohne
    Reparatur der Startabhängigkeiten geladen wird, führt Doctor aus und führt einen lokalen Agent-Turn
    gegen einen simulierten OpenAI-Endpunkt aus.
  - Verwenden Sie `OPENCLAW_NPM_ONBOARD_CHANNEL=discord`, um dieselbe Lane für die
    Paketinstallation mit Discord auszuführen.
- `pnpm test:docker:session-runtime-context`
  - Führt einen deterministischen Docker-Smoke-Test der erstellten App für eingebettete Laufzeitkontext-
    Transkripte aus. Überprüft, dass verborgener OpenClaw-Laufzeitkontext als
    nicht angezeigte benutzerdefinierte Nachricht bestehen bleibt, statt in den sichtbaren Benutzer-
    Turn einzufließen, legt anschließend eine betroffene fehlerhafte Sitzungs-JSONL an und überprüft,
    dass `openclaw doctor --fix` sie mit einer Sicherung auf den aktiven Branch umschreibt.
- `pnpm test:docker:npm-telegram-live`
  - Installiert einen OpenClaw-Paketkandidaten in Docker, führt das Onboarding des installierten Pakets
    aus, konfiguriert Telegram über die installierte CLI und verwendet anschließend
    die Live-Telegram-QA-Lane erneut, wobei dieses installierte Paket als Gateway
    des zu testenden Systems dient.
  - Der Wrapper bindet aus dem Checkout nur den Quellcode des `qa-lab`-Testgerüsts ein;
    das installierte Paket ist für `dist`, `openclaw/plugin-sdk` und die gebündelte
    Plugin-Laufzeit verantwortlich, sodass die Lane keine Plugins des aktuellen Checkouts in
    das zu testende Paket einmischt.
  - Verwendet standardmäßig `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@beta`; legen Sie
    `OPENCLAW_NPM_TELEGRAM_PACKAGE_TGZ=/path/to/openclaw-current.tgz` oder
    `OPENCLAW_CURRENT_PACKAGE_TGZ` fest, um statt einer Installation aus der Registry
    einen aufgelösten lokalen Tarball zu testen.
  - Gibt standardmäßig mit `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES=20` wiederholte RTT-Zeitmessungen
    in `qa-evidence.json` aus. Überschreiben Sie
    `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`,
    `OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS` oder
    `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES`, um den Lauf anzupassen.
    `OPENCLAW_NPM_TELEGRAM_RTT_CHECKS` wählt das zu untersuchende
    Telegram-QA-Szenario aus; das unterstützte RTT-Ziel ist `channel-canary`.
  - Verwendet dieselben Telegram-Umgebungsanmeldedaten oder dieselbe Convex-Anmeldedatenquelle wie
    `pnpm openclaw qa telegram`. Legen Sie für die CI-/Release-Automatisierung
    `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex` sowie
    `OPENCLAW_QA_CONVEX_SITE_URL` und ein Rollengeheimnis fest. Wenn
    `OPENCLAW_QA_CONVEX_SITE_URL` und ein Convex-Rollengeheimnis in
    der CI vorhanden sind, wählt der Docker-Wrapper Convex automatisch aus.
  - Der Wrapper validiert die Umgebungsvariablen für Telegram- oder Convex-Anmeldedaten auf dem Host,
    bevor die Docker-Build-/Installationsarbeiten beginnen. Legen Sie
    `OPENCLAW_NPM_TELEGRAM_SKIP_CREDENTIAL_PREFLIGHT=1` nur fest, wenn Sie
    bewusst die Einrichtung vor der Bereitstellung von Anmeldedaten debuggen.
  - `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci|maintainer` überschreibt nur für
    diese Lane das gemeinsam verwendete `OPENCLAW_QA_CREDENTIAL_ROLE`. Wenn Convex-
    Anmeldedaten ausgewählt sind und keine Rolle festgelegt wurde, verwendet der Wrapper in der CI `ci`
    und außerhalb der CI `maintainer`.
  - GitHub Actions stellt diese Lane als manuellen Maintainer-Workflow
    `NPM Telegram Beta E2E` bereit. Er wird bei einem Merge nicht ausgeführt. Der Workflow verwendet die
    `qa-live-shared`-Umgebung und Convex-CI-Anmeldedaten-Leases.
- GitHub Actions stellt außerdem `Package Acceptance` für parallel ausgeführte Produktnachweise
  gegen ein einzelnes Kandidatenpaket bereit. Der Workflow akzeptiert eine Git-Referenz, eine veröffentlichte npm-Spezifikation,
  eine HTTPS-Tarball-URL samt SHA-256, eine Richtlinie für vertrauenswürdige URLs oder ein Tarball-Artefakt
  aus einem anderen Lauf (`source=ref|npm|url|trusted-url|artifact`), lädt das
  normalisierte `openclaw-current.tgz` als `package-under-test` hoch und führt anschließend den
  vorhandenen Docker-E2E-Scheduler mit den Lane-Profilen `smoke`, `package`, `product`, `full`
  oder `custom` aus. Legen Sie `telegram_mode=mock-openai` oder
  `live-frontier` fest, um den Telegram-QA-Workflow mit demselben
  `package-under-test`-Artefakt auszuführen.
  - Produktnachweis für die neueste Beta:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai
```

- Der Nachweis über eine exakte Tarball-URL erfordert einen Digest und verwendet die öffentliche URL-Sicherheitsrichtlinie:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=url \
  -f package_url=https://registry.npmjs.org/openclaw/-/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

- Unternehmensinterne/private Tarball-Spiegelserver verwenden eine explizite Richtlinie für vertrauenswürdige Quellen:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

`source=trusted-url` liest `.github/package-trusted-sources.json` aus der vertrauenswürdigen Workflow-Referenz und akzeptiert weder URL-Anmeldedaten noch eine Umgehung des privaten Netzwerks per Workflow-Eingabe. Wenn die benannte Richtlinie Bearer-Authentifizierung vorsieht, konfigurieren Sie das feste Geheimnis `OPENCLAW_TRUSTED_PACKAGE_TOKEN`.

- Der Artefaktnachweis lädt ein Tarball-Artefakt aus einem anderen Actions-Lauf herunter:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=<artifact-name> \
  -f suite_profile=smoke
```

- `pnpm test:docker:plugins`
  - Paketiert und installiert den aktuellen OpenClaw-Build in Docker, startet das
    Gateway mit konfiguriertem OpenAI und aktiviert anschließend gebündelte Kanäle/Plugins durch
    Konfigurationsänderungen.
  - Überprüft, dass bei der Einrichtungserkennung nicht konfigurierte herunterladbare Plugins
    fehlen, die erste konfigurierte Doctor-Reparatur jedes fehlende
    herunterladbare Plugin explizit installiert und ein zweiter Neustart keine
    verborgene Abhängigkeitsreparatur ausführt.
  - Installiert außerdem eine bekannte ältere npm-Baseline, aktiviert Telegram vor
    der Ausführung von `openclaw update --tag <candidate>` und überprüft, dass Doctor
    nach der Aktualisierung des Kandidaten Altlasten aus früheren Plugin-Abhängigkeiten entfernt,
    ohne dass das Testgerüst nach der Installation eine Reparatur durchführt.
- `pnpm test:parallels:npm-update`
  - Führt den nativen Smoke-Test für die Aktualisierung einer Paketinstallation auf Parallels-Gästen aus.
    Jede ausgewählte Plattform installiert zunächst das angeforderte Baseline-Paket,
    führt dann im selben Gast den installierten Befehl `openclaw update` aus und
    überprüft die installierte Version, den Aktualisierungsstatus, die Gateway-Bereitschaft und
    einen lokalen Agent-Turn.
  - Verwenden Sie bei der Arbeit mit einem einzelnen Gast `--platform macos`, `--platform windows` oder `--platform linux`.
    Verwenden Sie `--json` für den Pfad des Zusammenfassungsartefakts
    und den Status pro Lane.
  - Die OpenAI-Lane verwendet standardmäßig `openai/gpt-5.6-luna` für den Nachweis des Live-Agent-Turns.
    Übergeben Sie `--model <provider/model>` oder legen Sie
    `OPENCLAW_PARALLELS_OPENAI_MODEL` fest, um ein anderes OpenAI-Modell zu validieren.
  - Umschließen Sie lange lokale Läufe mit einem Host-Timeout, damit Hänger beim Parallels-Transport
    nicht das verbleibende Testzeitfenster aufbrauchen können:

    ```bash
    timeout --foreground 150m pnpm test:parallels:npm-update -- --json
    timeout --foreground 90m pnpm test:parallels:npm-update -- --platform windows --json
    ```

  - Das Skript schreibt verschachtelte Lane-Protokolle unter
    `/tmp/openclaw-parallels-npm-update.*`. Prüfen Sie `windows-update.log`,
    `macos-update.log` oder `linux-update.log`, bevor Sie davon ausgehen, dass der äußere
    Wrapper hängt.
  - Die Windows-Aktualisierung kann auf einem kalten Gast 10 bis 15 Minuten für Doctor nach der Aktualisierung und
    die Paketaktualisierung benötigen; solange das verschachtelte npm-Debug-Protokoll fortschreitet,
    ist der Ablauf weiterhin intakt.
  - Führen Sie diesen aggregierenden Wrapper nicht parallel zu einzelnen Parallels-
    Smoke-Lanes für macOS, Windows oder Linux aus. Sie verwenden gemeinsam den VM-Zustand und können
    bei der Wiederherstellung von Snapshots, der Paketbereitstellung oder dem Gateway-Zustand des Gasts
    kollidieren.
  - Der Nachweis nach der Aktualisierung führt die normale Oberfläche der gebündelten Plugins aus, da
    Capability-Fassaden wie Sprachausgabe, Bilderzeugung und Medien-
    verständnis über gebündelte Laufzeit-APIs geladen werden, selbst wenn der Agent-
    Turn selbst nur eine einfache Textantwort prüft.

- `pnpm openclaw qa aimock`
  - Startet nur den lokalen AIMock-Provider-Server für direkte Protokoll-Smoke-
    Tests.
- `pnpm openclaw qa matrix`
  - Führt den Matrix-Live-QA-Lauf gegen einen temporären, Docker-basierten Tuwunel-
    Homeserver aus. Nur für Quellcode-Checkouts – paketierte Installationen enthalten
    `qa-lab` nicht.
  - Vollständige CLI, Profil-/Szenariokatalog, Umgebungsvariablen und Artefaktstruktur:
    [Matrix-Smoke-Läufe](/de/concepts/qa-e2e-automation#matrix-smoke-lanes).
- `pnpm openclaw qa telegram`
  - Führt den Telegram-Live-QA-Lauf gegen eine echte private Gruppe aus und verwendet dabei
    die Treiber- und SUT-Bot-Tokens aus der Umgebung.
  - Erfordert `OPENCLAW_QA_TELEGRAM_GROUP_ID`,
    `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` und
    `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`. Die Gruppen-ID muss die numerische
    Telegram-Chat-ID sein.
  - Unterstützt `--credential-source convex` für gemeinsam genutzte gepoolte Anmeldedaten.
    Verwenden Sie standardmäßig den Umgebungsmodus oder setzen Sie `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`,
    um gepoolte Leases zu aktivieren.
  - Die Standardwerte decken Canary, Mention-Gating, Befehlsadressierung, `/status`,
    erwähnte Bot-zu-Bot-Antworten und Antworten auf native Kernbefehle ab.
    Die Standardwerte von `mock-openai` decken außerdem deterministische Antwortketten- und
    Telegram-Regressionen beim Streaming der finalen Nachricht ab. Verwenden Sie `--list-scenarios`
    für optionale Prüfungen wie `session_status`.
  - Wird mit einem Exit-Code ungleich null beendet, wenn ein Szenario fehlschlägt. Verwenden Sie `--allow-failures` für
    Artefakte ohne fehleranzeigenden Exit-Code.
  - Erfordert zwei unterschiedliche Bots in derselben privaten Gruppe, wobei der SUT-Bot
    einen Telegram-Benutzernamen bereitstellt.
  - Aktivieren Sie für eine stabile Bot-zu-Bot-Beobachtung den Bot-to-Bot Communication Mode
    in `@BotFather` für beide Bots und stellen Sie sicher, dass der Treiber-Bot den
    Bot-Datenverkehr der Gruppe beobachten kann.
  - Schreibt einen Telegram-QA-Bericht, eine Zusammenfassung und `qa-evidence.json` unter
    `.artifacts/qa-e2e/...`. Antwortszenarien enthalten die RTT von der Sendeanfrage
    des Treibers bis zur beobachteten SUT-Antwort.

`Mantis Telegram Live` ist der PR-Evidenz-Wrapper für diesen Lauf. Er führt
die Kandidaten-Ref mit von Convex geleasten Telegram-Anmeldedaten aus, rendert das
redigierte QA-Berichts-/Evidenzpaket in einem Crabbox-Desktop-Browser, zeichnet MP4-
Evidenz auf, erzeugt ein bewegungsoptimiertes GIF, lädt das Artefaktpaket hoch und
veröffentlicht über die Mantis GitHub App Inline-PR-Evidenz, wenn `pr_number`
gesetzt ist. Maintainer können ihn über `Mantis Scenario`
(`scenario_id: telegram-live`) in der Actions-Benutzeroberfläche oder direkt über einen Pull-Request-Kommentar starten:

```text
@openclaw-mantis telegram
@openclaw-mantis telegram scenario=telegram-status-command
@openclaw-mantis telegram scenarios=telegram-status-command,channel-canary
```

`Mantis Telegram Desktop Proof` ist der agentische Before/After-Wrapper für
natives Telegram Desktop zur visuellen PR-Verifikation. Starten Sie ihn über die Actions-Benutzeroberfläche mit
frei formuliertem `instructions`, über `Mantis Scenario` (`scenario_id:
telegram-desktop-proof`) oder über einen PR-Kommentar:

```text
@openclaw-mantis telegram desktop proof
```

Der Mantis-Agent liest den PR, entscheidet, welches in Telegram sichtbare Verhalten
die Änderung belegt, führt den Crabbox-Telegram-Desktop-Verifikationslauf für echte Benutzer mit
Baseline- und Kandidaten-Refs aus, iteriert, bis die nativen GIFs aussagekräftig sind,
schreibt ein gepaartes `motionPreview`-Manifest und veröffentlicht dieselbe zweispaltige GIF-
Tabelle über die Mantis GitHub App, wenn `pr_number` gesetzt ist.

- `pnpm openclaw qa mantis telegram-desktop-builder`
  - Least einen Crabbox-Linux-Desktop oder verwendet ihn erneut, installiert natives Telegram
    Desktop, konfiguriert OpenClaw mit einem geleasten Telegram-SUT-Bot-Token,
    startet den Gateway und zeichnet Screenshot-/MP4-Evidenz vom
    sichtbaren VNC-Desktop auf.
  - Verwendet standardmäßig `--credential-source convex`, sodass Workflows nur das
    Convex-Broker-Secret benötigen. Verwenden Sie `--credential-source env` mit denselben
    `OPENCLAW_QA_TELEGRAM_*`-Variablen wie `pnpm openclaw qa telegram`.
  - Telegram Desktop benötigt weiterhin eine Benutzeranmeldung bzw. ein Benutzerprofil. Das Bot-Token
    konfiguriert nur OpenClaw. Verwenden Sie `--telegram-profile-archive-env <name>`
    für ein Base64-`.tgz`-Profilarchiv oder verwenden Sie `--keep-lease` und melden Sie sich
    einmal manuell über VNC an.
  - Schreibt `mantis-telegram-desktop-builder-report.md`,
    `mantis-telegram-desktop-builder-summary.json`,
    `telegram-desktop-builder.png` und `telegram-desktop-builder.mp4`
    unter das Ausgabeverzeichnis.

Live-Transport-Läufe verwenden einen gemeinsamen Standardvertrag, damit neue Transporte nicht
auseinanderdriften; die Abdeckungsmatrix der einzelnen Läufe befindet sich in der
[QA-Übersicht – Live-Transport-Abdeckung](/de/concepts/qa-e2e-automation#live-transport-coverage).
`qa-channel` ist die umfassende synthetische Suite und nicht Teil dieser Matrix.

### Gemeinsam genutzte Telegram-Anmeldedaten über Convex (v1)

Wenn `--credential-source convex` (oder `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`)
für Live-Transport-QA aktiviert ist, bezieht das QA-Lab eine exklusive Lease aus einem
Convex-basierten Pool, sendet während der Ausführung des Laufs Heartbeats für diese Lease und
gibt die Lease beim Herunterfahren frei. Der Abschnittsname stammt aus der Zeit vor der Unterstützung von Discord, Slack und
WhatsApp; der Lease-Vertrag wird von allen Arten gemeinsam genutzt.

Referenz-Projektgerüst für Convex: `qa/convex-credential-broker/`

Erforderliche Umgebungsvariablen:

- `OPENCLAW_QA_CONVEX_SITE_URL` (zum Beispiel `https://your-deployment.convex.site`)
- Ein Secret für die ausgewählte Rolle:
  - `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` für `maintainer`
  - `OPENCLAW_QA_CONVEX_SECRET_CI` für `ci`
- Auswahl der Anmeldedatenrolle:
  - CLI: `--credential-role maintainer|ci`
  - Umgebungsstandard: `OPENCLAW_QA_CREDENTIAL_ROLE` (standardmäßig `ci` in CI, andernfalls `maintainer`)

Optionale Umgebungsvariablen:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS` (Standardwert `1200000`)
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS` (Standardwert `30000`)
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS` (Standardwert `90000`)
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS` (Standardwert `15000`)
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX` (Standardwert `/qa-credentials/v1`)
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID` (optionale Trace-ID)
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` erlaubt Loopback-`http://`-Convex-URLs ausschließlich für die lokale Entwicklung.

`OPENCLAW_QA_CONVEX_SITE_URL` sollte im normalen Betrieb `https://` verwenden.

Maintainer-Adminbefehle (Pool hinzufügen/entfernen/auflisten) erfordern ausdrücklich
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`.

CLI-Hilfsbefehle für Maintainer:

```bash
pnpm openclaw qa credentials doctor
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

Verwenden Sie `doctor` vor Live-Läufen, um die Convex-Site-URL, Broker-Secrets,
den Endpunktpräfix, das HTTP-Zeitlimit und die Erreichbarkeit für Administration/Auflistung zu prüfen, ohne
Secret-Werte auszugeben. Verwenden Sie `--json` für maschinenlesbare Ausgaben in Skripten und CI-
Hilfsprogrammen.

Standard-Endpunktvertrag (`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`).
Anfragen authentifizieren sich mit einem `Authorization: Bearer <role secret>`-Header;
in den nachfolgenden Bodys ist dieser Header ausgelassen:

- `POST /acquire`
  - Anfrage: `{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - Erfolg: `{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - Ausgeschöpft/wiederholbar: `{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /payload-chunk`
  - Anfrage: `{ kind, ownerId, actorRole, credentialId, leaseToken, index }`
  - Erfolg: `{ status: "ok", index, data }`
- `POST /heartbeat`
  - Anfrage: `{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - Erfolg: `{ status: "ok" }` (oder leeres `2xx`)
- `POST /release`
  - Anfrage: `{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - Erfolg: `{ status: "ok" }` (oder leeres `2xx`)
- `POST /admin/add` (nur Maintainer-Secret)
  - Anfrage: `{ kind, actorId, payload, note?, status? }`
  - Erfolg: `{ status: "ok", credential }`
- `POST /admin/remove` (nur Maintainer-Secret)
  - Anfrage: `{ credentialId, actorId }`
  - Erfolg: `{ status: "ok", changed, credential }`
  - Schutz bei aktiver Lease: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list` (nur Maintainer-Secret)
  - Anfrage: `{ kind?, status?, includePayload?, limit? }`
  - Erfolg: `{ status: "ok", credentials, count }`

Payload-Struktur für die Telegram-Art:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` muss eine numerische Telegram-Chat-ID-Zeichenfolge sein.
- `admin/add` validiert diese Struktur für `kind: "telegram"` und lehnt fehlerhafte Payloads ab.

Payload-Struktur für die Telegram-Echtbenutzer-Art:

- `{ groupId: string, sutToken: string, testerUserId: string, testerUsername: string, telegramApiId: string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string, tdlibArchiveBase64: string, tdlibArchiveSha256: string, desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }`
- `groupId`, `testerUserId` und `telegramApiId` müssen numerische Zeichenfolgen sein.
- `tdlibArchiveSha256` und `desktopTdataArchiveSha256` müssen SHA-256-Hexadezimalzeichenfolgen sein.
- `kind: "telegram-user"` ist für den Mantis-Telegram-Desktop-Verifikationsworkflow reserviert. Generische QA-Lab-Läufe dürfen es nicht beziehen.

Vom Broker validierte Mehrkanal-Payloads:

- Discord: `{ guildId: string, channelId: string, driverBotToken: string, sutBotToken: string, sutApplicationId: string, voiceChannelId?: string }`
- WhatsApp: `{ driverPhoneE164: string, sutPhoneE164: string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string, groupJid?: string }`

Slack-Läufe können ebenfalls Leases aus dem Pool beziehen, die Slack-Payload-Validierung
befindet sich derzeit jedoch im Slack-QA-Runner und nicht im Broker. Verwenden Sie
`{ channelId: string, driverBotToken: string, sutBotToken: string, sutAppToken: string }`
für Slack-Zeilen.

### Einen Kanal zur QA hinzufügen

Die Architektur und Namen der Szenario-Hilfsfunktionen für neue Kanaladapter befinden sich in der
[QA-Übersicht – Einen Kanal hinzufügen](/de/concepts/qa-e2e-automation#adding-a-channel).
Mindestanforderungen: Implementieren Sie den Transport-Runner auf der gemeinsamen `qa-lab`-Host-
Schnittstelle, fügen Sie ein `adapterFactory` für gemeinsam genutzte Szenarien hinzu, deklarieren Sie `qaRunners` im
Plugin-Manifest, mounten Sie es als `openclaw qa <runner>` und erstellen Sie Szenarien unter
`qa/scenarios/`.

## Testsuiten (was wo ausgeführt wird)

Betrachten Sie die Suiten als „zunehmenden Realitätsgrad“ (mit zunehmender Flakiness und höheren Kosten).

### Unit-/Integrationstests (Standard)

- Befehl: `pnpm test`
- Konfiguration: Nicht zielgerichtete Ausführungen verwenden den `vitest.full-*.config.ts`-Shard-Satz und können
  Shards mit mehreren Projekten zur parallelen
  Planung in projektbezogene Konfigurationen aufteilen
- Dateien: Kern-/Unit-Testinventare unter `src/**/*.test.ts`,
  `packages/**/*.test.ts` und `test/**/*.test.ts`; UI-Unit-Tests werden im
  dedizierten `unit-ui`-Shard ausgeführt
- Umfang:
  - Reine Unit-Tests
  - Prozessinterne Integrationstests (Gateway-Authentifizierung, Routing, Tooling, Parsing, Konfiguration)
  - Deterministische Regressionstests für bekannte Fehler
- Erwartungen:
  - Wird in CI ausgeführt
  - Keine echten Schlüssel erforderlich
  - Sollte schnell und stabil sein
  - Resolver- und Loader-Tests für öffentliche Oberflächen müssen umfassendes `api.js`- und
    `runtime-api.js`-Fallback-Verhalten mit generierten kleinen Plugin-Fixtures belegen,
    nicht mit echten Quell-APIs gebündelter Plugins. Ladevorgänge echter Plugin-APIs gehören in
    Plugin-eigene Vertrags-/Integrationssuiten.

Richtlinie für native Abhängigkeiten:

- Standard-Testinstallationen überspringen optionale native Discord-Opus-Builds. Discord
  Voice verwendet das gebündelte `libopus-wasm`, und `@discordjs/opus` bleibt in
  `allowBuilds` deaktiviert, damit lokale Tests und Testbox-Läufe das native
  Add-on nicht kompilieren.
- Vergleichen Sie die Leistung von nativem Opus im `libopus-wasm`-Benchmark-Repository, nicht
  in den standardmäßigen OpenClaw-Installations-/Testschleifen. Setzen Sie `@discordjs/opus` in der
  standardmäßigen `allowBuilds` nicht auf `true`; dadurch kompilieren nicht zusammenhängende Installations-/Testschleifen
  nativen Code.

<AccordionGroup>
  <Accordion title="Projekte, Shards und bereichsspezifische Läufe">

    - Nicht zielgerichtete `pnpm test` führt dreizehn kleinere Shard-Konfigurationen (`core-unit-fast`, `core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-tooling`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) statt eines einzigen riesigen nativen Prozesses des Root-Projekts aus. Dies reduziert den maximalen RSS auf ausgelasteten Rechnern und verhindert, dass Auto-Reply-/Plugin-Arbeit unabhängigen Suites Ressourcen entzieht.
    - `pnpm test --watch` verwendet weiterhin den nativen `vitest.config.ts`-Projektgraphen des Root-Projekts, da eine Watch-Schleife mit mehreren Shards nicht praktikabel ist.
    - `pnpm test`, `pnpm test:watch` und `pnpm test:perf:imports` leiten explizite Datei-/Verzeichnisziele zuerst durch bereichsspezifische Lanes, sodass `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` nicht die vollständigen Startkosten des Root-Projekts verursacht.
    - `pnpm test:changed` erweitert geänderte Git-Pfade standardmäßig zu kostengünstigen bereichsspezifischen Lanes: direkte Teständerungen, benachbarte `*.test.ts`-Dateien, explizite Quellzuordnungen und abhängige Elemente des lokalen Importgraphen. Änderungen an Konfiguration, Einrichtung oder Paketen führen Tests nicht umfassend aus, außer Sie verwenden ausdrücklich `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed`.
    - `pnpm check:changed` ist das normale intelligente lokale Prüf-Gate für eng begrenzte Arbeiten. Es klassifiziert den Diff in Core, Core-Tests, Erweiterungen, Erweiterungstests, Apps, Dokumentation, Release-Metadaten, Live-Docker-Werkzeuge und Tooling und führt anschließend die entsprechenden Typecheck-, Lint- und Guard-Befehle aus. Es führt keine Vitest-Tests aus; rufen Sie für einen Testnachweis `pnpm test:changed` oder explizit `pnpm test <target>` auf. Versionsänderungen, die ausschließlich Release-Metadaten betreffen, führen gezielte Prüfungen von Version, Konfiguration und Root-Abhängigkeiten aus, einschließlich eines Guards, der Paketänderungen außerhalb des obersten Versionsfelds ablehnt.
    - Änderungen am Live-Docker-ACP-Harness führen gezielte Prüfungen aus: Shell-Syntax für die Live-Docker-Authentifizierungsskripte und einen Dry-Run des Live-Docker-Schedulers. Änderungen an `package.json` werden nur einbezogen, wenn der Diff auf `scripts["test:docker:live-*"]` beschränkt ist; Änderungen an Abhängigkeiten, Exporten, Versionen und anderen Paketoberflächen verwenden weiterhin die umfassenderen Guards.
    - Importarme Unit-Tests aus Agenten, Befehlen, Plugins, Auto-Reply-Hilfsfunktionen, `plugin-sdk` und ähnlichen Bereichen mit reinen Hilfsfunktionen werden durch die `unit-fast`-Lane geleitet, die `test/setup-openclaw-runtime.ts` überspringt; zustandsbehaftete bzw. laufzeitintensive Dateien verbleiben auf den vorhandenen Lanes.
    - Ausgewählte `plugin-sdk`- und `commands`-Hilfsquelldateien ordnen Ausführungen im Änderungsmodus außerdem expliziten benachbarten Tests in diesen leichtgewichtigen Lanes zu, sodass Änderungen an Hilfsfunktionen nicht erneut die vollständige rechenintensive Suite für dieses Verzeichnis ausführen.
    - `auto-reply` besitzt eigene Buckets für Core-Hilfsfunktionen auf oberster Ebene, `reply.*`-Integrationstests auf oberster Ebene und den `src/auto-reply/reply/**`-Teilbaum. Die CI teilt den Reply-Teilbaum zusätzlich in Shards für Agent-Runner, Dispatch und Befehls-/Zustandsrouting auf, damit nicht ein einzelner importintensiver Bucket den gesamten Node-Nachlauf bestimmt.
    - Die normale PR-/Main-CI überspringt bewusst den Batch-Durchlauf der gebündelten Plugins und den ausschließlich für Releases vorgesehenen `agentic-plugins`-Shard. Die vollständige Release-Validierung startet für diese Plugin-intensiven Suites bei Release-Kandidaten den separaten untergeordneten `Plugin Prerelease`-Workflow.

  </Accordion>

  <Accordion title="Abdeckung des eingebetteten Runners">

    - Wenn Sie Eingaben für die Erkennung von Nachrichtenwerkzeugen oder den Laufzeitkontext der Compaction ändern,
      müssen beide Abdeckungsebenen beibehalten werden.
    - Fügen Sie gezielte Regressionstests für reine Routing- und Normalisierungsgrenzen
      hinzu.
    - Halten Sie die Integrations-Suites des eingebetteten Runners funktionsfähig:
      `src/agents/embedded-agent-runner/compact.hooks.test.ts`,
      `src/agents/embedded-agent-runner/run.overflow-compaction.test.ts` und
      `src/agents/embedded-agent-runner/run.overflow-compaction.loop.test.ts`.
    - Diese Suites überprüfen, dass bereichsspezifische IDs und das Compaction-Verhalten weiterhin
      die echten `run.ts`- / `compact.ts`-Pfade durchlaufen; reine Hilfsfunktionstests sind
      kein ausreichender Ersatz für diese Integrationspfade.

  </Accordion>

  <Accordion title="Standardwerte für Vitest-Pool und -Isolation">

    - Die grundlegende Vitest-Konfiguration verwendet standardmäßig `threads`.
    - Die gemeinsame Vitest-Konfiguration legt `isolate: false` fest und verwendet
      den nicht isolierten Runner für die Root-Projekte sowie die E2E- und Live-Konfigurationen.
    - Die Root-UI-Lane behält ihre `jsdom`-Einrichtung und ihren Optimierer bei, wird jedoch ebenfalls
      auf dem gemeinsamen nicht isolierten Runner ausgeführt.
    - Jeder `pnpm test`-Shard übernimmt dieselben Standardwerte für `threads` + `isolate: false`
      aus der gemeinsamen Vitest-Konfiguration.
    - `scripts/run-vitest.mjs` fügt standardmäßig `--no-maglev` für untergeordnete Vitest-Node-
      Prozesse hinzu, um den V8-Kompilierungsaufwand bei großen lokalen Ausführungen zu reduzieren.
      Setzen Sie `OPENCLAW_VITEST_ENABLE_MAGLEV=1`, um einen Vergleich mit dem standardmäßigen V8-
      Verhalten durchzuführen.
    - `scripts/run-vitest.mjs` beendet explizite Vitest-Ausführungen außerhalb des Watch-Modus
      nach 5 Minuten ohne Ausgabe auf stdout oder stderr. Setzen Sie
      `OPENCLAW_VITEST_NO_OUTPUT_TIMEOUT_MS=0`, um den Watchdog für
      eine absichtlich ausgabefreie Untersuchung zu deaktivieren.

  </Accordion>

  <Accordion title="Schnelle lokale Iteration">

    - `pnpm changed:lanes` zeigt, welche Architekturlanes ein Diff auslöst.
    - Der Pre-Commit-Hook führt ausschließlich Formatierungen durch. Er nimmt formatierte Dateien
      erneut in den Staging-Bereich auf und führt weder Lint noch Typecheck oder Tests aus.
    - Führen Sie vor der Übergabe oder dem Push ausdrücklich `pnpm check:changed` aus, wenn Sie
      das intelligente lokale Prüf-Gate benötigen.
    - `pnpm test:changed` leitet standardmäßig durch kostengünstige bereichsspezifische Lanes. Verwenden Sie
      `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` nur, wenn der Agent
      entscheidet, dass eine Änderung an Harness, Konfiguration, Paket oder Vertrag tatsächlich
      eine umfassendere Vitest-Abdeckung benötigt.
    - `pnpm test:max` und `pnpm test:changed:max` behalten dasselbe Routing-
      Verhalten bei, jedoch mit einer höheren Worker-Obergrenze.
    - Die automatische Skalierung lokaler Worker ist bewusst konservativ und reduziert die Last,
      wenn der Lastdurchschnitt des Hosts bereits hoch ist, sodass mehrere gleichzeitige
      Vitest-Ausführungen standardmäßig weniger Beeinträchtigungen verursachen.
    - Die grundlegende Vitest-Konfiguration markiert die Projekte/Konfigurationsdateien als
      `forceRerunTriggers`, damit erneute Ausführungen im Änderungsmodus korrekt bleiben, wenn sich die
      Testverdrahtung ändert.
    - Die Konfiguration lässt `OPENCLAW_VITEST_FS_MODULE_CACHE` auf
      unterstützten Hosts aktiviert; setzen Sie `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path`
      für einen einzelnen expliziten Cache-Speicherort zur direkten Profilerstellung.

  </Accordion>

  <Accordion title="Performance-Debugging">

    - `pnpm test:perf:imports` aktiviert die Berichterstellung über die Dauer von Vitest-Importen sowie
      eine Aufschlüsselung der Importe.
    - `pnpm test:perf:imports:changed` beschränkt dieselbe Profilansicht auf
      Dateien, die seit `origin/main` geändert wurden.
    - Shard-Zeitdaten werden in `.artifacts/vitest-shard-timings.json` geschrieben.
      Ausführungen der gesamten Konfiguration verwenden den Konfigurationspfad als Schlüssel; CI-
      Shards mit Include-Mustern hängen den Shard-Namen an, damit gefilterte Shards
      separat verfolgt werden können.
    - Wenn ein rechenintensiver Test weiterhin den Großteil seiner Zeit mit Startimporten verbringt,
      halten Sie umfangreiche Abhängigkeiten hinter einer schmalen lokalen `*.runtime.ts`-Schnittstelle und
      mocken Sie diese Schnittstelle direkt, anstatt Laufzeithilfsfunktionen tief zu importieren,
      nur um sie über `vi.mock(...)` weiterzureichen.
    - `pnpm test:perf:changed:bench -- --ref <git-ref>` vergleicht das geroutete
      `test:changed` mit dem nativen Root-Projekt-Pfad für diesen
      committeten Diff und gibt die verstrichene Zeit sowie den maximalen RSS unter macOS aus.
    - `pnpm test:perf:changed:bench -- --worktree` benchmarked den aktuellen
      Arbeitsbaum mit nicht committeten Änderungen, indem die Liste geänderter Dateien durch
      `scripts/test-projects.mjs` und die Root-Vitest-Konfiguration geleitet wird.
    - `pnpm test:perf:profile:main` schreibt ein CPU-Profil des Hauptthreads für
      den Startaufwand sowie den Transformationsaufwand von Vitest/Vite.
    - `pnpm test:perf:profile:runner` schreibt CPU- und Heap-Profile des Runners für
      die Unit-Suite bei deaktivierter Dateiparallelität.

  </Accordion>
</AccordionGroup>

### Stabilität (Gateway)

- Befehl: `pnpm test:stability:gateway`
- Konfiguration: `test/vitest/vitest.gateway.config.ts`, `test/vitest/vitest.logging.config.ts` und `test/vitest/vitest.infra.config.ts`, jeweils auf einen Worker beschränkt
- Umfang:
  - Startet standardmäßig einen echten Loopback-Gateway mit aktivierter Diagnose
  - Leitet synthetische Gateway-Nachrichten-, Speicher- und Nutzlastaktivität mit großen Nutzlasten durch den Diagnoseereignispfad
  - Fragt `diagnostics.stability` über Gateway-WS-RPC ab
  - Deckt Persistenzhilfsfunktionen des Diagnose-Stabilitätspakets ab
  - Stellt sicher, dass der Recorder begrenzt bleibt, synthetische RSS-Stichproben das Belastungsbudget nicht überschreiten und sich die Warteschlangentiefen pro Sitzung wieder auf null leeren
- Erwartungen:
  - CI-sicher und ohne Schlüssel
  - Eng begrenzte Lane für die Nachverfolgung von Stabilitätsregressionen, kein Ersatz für die vollständige Gateway-Suite

### E2E (Repo-Aggregat)

- Befehl: `pnpm test:e2e`
- Umfang:
  - Führt die E2E-Lane für den Gateway-Smoke-Test aus
  - Führt die E2E-Lane für den Browser mit gemockter Control UI aus
- Erwartungen:
  - CI-sicher und ohne Schlüssel
  - Erfordert eine installierte Playwright-Chromium-Version

### E2E (Gateway-Smoke-Test)

- Befehl: `pnpm test:e2e:gateway`
- Konfiguration: `test/vitest/vitest.e2e.config.ts`
- Dateien: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts` und E2E-Tests gebündelter Plugins unter `extensions/`
- Laufzeitstandardwerte:
  - Verwendet Vitest `threads` mit `isolate: false`, entsprechend dem übrigen Repo.
  - Verwendet adaptive Worker (CI: bis zu 2, lokal: standardmäßig 1).
  - Wird standardmäßig im stillen Modus ausgeführt, um den Aufwand für Konsolen-E/A zu reduzieren.
- Nützliche Überschreibungen:
  - `OPENCLAW_E2E_WORKERS=<n>`, um die Worker-Anzahl zu erzwingen (auf 16 begrenzt).
  - `OPENCLAW_E2E_VERBOSE=1`, um die ausführliche Konsolenausgabe wieder zu aktivieren.
- Umfang:
  - End-to-End-Verhalten des Gateways mit mehreren Instanzen
  - WebSocket-/HTTP-Oberflächen, Node-Kopplung und aufwendigere Netzwerkkommunikation
- Erwartungen:
  - Wird in der CI ausgeführt (wenn in der Pipeline aktiviert)
  - Keine echten Schlüssel erforderlich
  - Mehr bewegliche Teile als bei Unit-Tests (kann langsamer sein)

### E2E (Control UI mit gemocktem Browser)

- Befehl: `pnpm test:ui:e2e`
- Konfiguration: `test/vitest/vitest.ui-e2e.config.ts`
- Dateien: `ui/src/**/*.e2e.test.ts`
- Umfang:
  - Startet die Vite-Control-UI
  - Steuert eine echte Chromium-Seite über Playwright
  - Ersetzt den Gateway-WebSocket durch deterministische browserinterne Mocks
- Erwartungen:
  - Wird in der CI als Teil von `pnpm test:e2e` ausgeführt
  - Kein echter Gateway und keine echten Agenten- oder Provider-Schlüssel erforderlich
  - Browserabhängigkeit muss vorhanden sein (`pnpm --dir ui exec playwright install chromium`)

### E2E: Smoke-Test des OpenShell-Backends

- Befehl: `pnpm test:e2e:openshell`
- Datei: `extensions/openshell/src/backend.e2e.test.ts`
- Umfang:
  - Verwendet einen aktiven lokalen OpenShell-Gateway erneut
  - Erstellt eine Sandbox aus einer temporären lokalen Dockerfile
  - Testet das OpenShell-Backend von OpenClaw über echtes `sandbox ssh-config` + SSH-Ausführung
  - Überprüft das kanonische Verhalten des entfernten Dateisystems über die Dateisystem-Bridge der Sandbox
- Erwartungen:
  - Nur nach ausdrücklicher Aktivierung; nicht Teil der standardmäßigen `pnpm test:e2e`-Ausführung
  - Erfordert eine lokale `openshell`-CLI sowie einen funktionierenden Docker-Daemon
  - Erfordert einen aktiven lokalen OpenShell-Gateway und dessen Konfigurationsquelle
  - Verwendet isolierte `HOME` / `XDG_CONFIG_HOME` und zerstört anschließend die Test-Sandbox
- Nützliche Überschreibungen:
  - `OPENCLAW_E2E_OPENSHELL=1`, um den Test bei der manuellen Ausführung der umfassenderen E2E-Suite zu aktivieren
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`, um eine nicht standardmäßige CLI-Binärdatei oder ein Wrapper-Skript anzugeben
  - `OPENCLAW_E2E_OPENSHELL_CONFIG_HOME=/path/to/config`, um die registrierte Gateway-Konfiguration für den isolierten Test bereitzustellen
  - `OPENCLAW_E2E_OPENSHELL_HOST_IP=172.18.0.1`, um die von der Host-Policy-Fixture verwendete Docker-Gateway-IP zu überschreiben

### Live (echte Provider + echte Modelle)

- Befehl: `pnpm test:live`
- Konfiguration: `test/vitest/vitest.live.config.ts`
- Dateien: `src/**/*.live.test.ts`, `test/**/*.live.test.ts` und Live-Tests für gebündelte Plugins unter `extensions/`
- Standard: durch `pnpm test:live` **aktiviert** (setzt `OPENCLAW_LIVE_TEST=1`)
- Umfang:
  - „Funktioniert dieser Provider/dieses Modell _heute_ tatsächlich mit echten Anmeldedaten?“
  - Erkennt Änderungen am Provider-Format, Besonderheiten bei Tool-Aufrufen, Authentifizierungsprobleme und das Verhalten bei Ratenbegrenzungen
- Erwartungen:
  - Absichtlich nicht CI-stabil (echte Netzwerke, echte Provider-Richtlinien, Kontingente, Ausfälle)
  - Verursacht Kosten / beansprucht Ratenbegrenzungen
  - Führen Sie vorzugsweise eingegrenzte Teilmengen statt „alles“ aus
- Live-Ausführungen verwenden bereits exportierte API-Schlüssel und bereitgestellte Authentifizierungsprofile.
- Standardmäßig isolieren Live-Ausführungen weiterhin `HOME` und kopieren Konfigurations-/Authentifizierungsmaterial in ein temporäres Test-Home-Verzeichnis, damit Unit-Test-Fixtures Ihr echtes `~/.openclaw` nicht verändern können.
- Setzen Sie `OPENCLAW_LIVE_USE_REAL_HOME=1` nur, wenn die Live-Tests absichtlich Ihr echtes Home-Verzeichnis verwenden sollen.
- `pnpm test:live` verwendet standardmäßig einen ruhigeren Modus: Die Fortschrittsausgabe von `[live] ...` bleibt erhalten, während Gateway-Bootstrap-Protokolle und Bonjour-Meldungen stummgeschaltet werden. Setzen Sie `OPENCLAW_LIVE_TEST_QUIET=0`, wenn Sie wieder die vollständigen Startprotokolle wünschen.
- Rotation von API-Schlüsseln (Provider-spezifisch): Setzen Sie `*_API_KEYS` im Komma-/Semikolonformat oder `*_API_KEY_1`, `*_API_KEY_2` (zum Beispiel `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) oder verwenden Sie eine Live-spezifische Überschreibung über `OPENCLAW_LIVE_*_KEY`; Tests versuchen es bei Antworten aufgrund einer Ratenbegrenzung erneut.
- Fortschritts-/Heartbeat-Ausgabe:
  - Live-Suites geben Fortschrittszeilen auf stderr aus, damit lange Provider-Aufrufe sichtbar aktiv bleiben, selbst wenn die Konsolenerfassung von Vitest keine Ausgabe zeigt.
  - `test/vitest/vitest.live.config.ts` deaktiviert das Abfangen der Konsole durch Vitest, sodass Fortschrittszeilen von Provider und Gateway während Live-Ausführungen sofort gestreamt werden.
  - Passen Sie Heartbeats für direkte Modelle mit `OPENCLAW_LIVE_HEARTBEAT_MS` an.
  - Passen Sie Gateway-/Probe-Heartbeats mit `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` an.

## Welche Suite sollte ich ausführen?

Verwenden Sie diese Entscheidungstabelle:

- Logik/Tests bearbeiten: Führen Sie `pnpm test` aus (und `pnpm test:coverage`, wenn Sie viel geändert haben)
- Gateway-Netzwerk / WS-Protokoll / Kopplung ändern: Fügen Sie `pnpm test:e2e` hinzu
- „Mein Bot ist ausgefallen“ / Provider-spezifische Fehler / Tool-Aufrufe debuggen: Führen Sie eine eingegrenzte Ausführung von `pnpm test:live` durch

## Live-Tests (mit Netzwerkzugriff)

Informationen zur Live-Modellmatrix, zu Smoke-Tests für das CLI-Backend, ACP-Smoke-Tests, dem Codex-App-Server-
Test-Harness und allen Live-Tests für Medien-Provider (Deepgram, BytePlus, ComfyUI,
Bild, Musik, Video, Medien-Harness) sowie zur Handhabung von Anmeldedaten für Live-Ausführungen

- finden Sie unter [Live-Suites testen](/de/help/testing-live). Die spezielle Checkliste zur Validierung von Updates und
  Plugins finden Sie unter
  [Updates und Plugins testen](/de/help/testing-updates-plugins).

## Docker-Runner (optionale Prüfungen, ob es „unter Linux funktioniert“)

Diese Docker-Runner sind in zwei Gruppen unterteilt:

- Live-Modell-Runner: `test:docker:live-models` und `test:docker:live-gateway` führen innerhalb des Docker-Images des Repositorys (`src/agents/models.profiles.live.test.ts` und `src/gateway/gateway-models.profiles.live.test.ts`) nur die Live-Datei aus, die ihrem Profilschlüssel entspricht, und binden dabei Ihr lokales Konfigurationsverzeichnis, Ihren Workspace und eine optionale Profil-Umgebungsdatei ein. Die entsprechenden lokalen Einstiegspunkte sind `test:live:models-profiles` und `test:live:gateway-profiles`.
- Docker-Live-Runner behalten bei Bedarf ihre eigenen praxisgerechten Obergrenzen bei:
  `test:docker:live-models` verwendet standardmäßig die kuratierte, unterstützte Auswahl mit hoher Aussagekraft, und
  `test:docker:live-gateway` verwendet standardmäßig `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` und
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Setzen Sie `OPENCLAW_LIVE_MAX_MODELS`
  oder die Gateway-Umgebungsvariablen, wenn Sie ausdrücklich eine kleinere Obergrenze oder einen größeren Scan wünschen.
- `test:docker:all` erstellt das Live-Docker-Image einmal über `test:docker:live-build`, packt OpenClaw einmal über `scripts/package-openclaw-for-docker.mjs` als npm-Tarball und erstellt/verwendet anschließend zwei `scripts/e2e/Dockerfile`-Images. Das Bare-Image ist lediglich der Node-/Git-Runner für Installations-, Update- und Plugin-Abhängigkeits-Lanes; diese Lanes binden den vorab erstellten Tarball ein. Das funktionale Image installiert denselben Tarball unter `/app` für Lanes zur Prüfung der Funktionalität der gebauten Anwendung. Die Definitionen der Docker-Lanes befinden sich in `scripts/lib/docker-e2e-scenarios.mjs`; die Planerlogik befindet sich in `scripts/lib/docker-e2e-plan.mjs`; `scripts/test-docker-all.mjs` führt den ausgewählten Plan aus. Das Aggregat verwendet einen gewichteten lokalen Scheduler: `OPENCLAW_DOCKER_ALL_PARALLELISM` steuert die Prozess-Slots, während Ressourcenobergrenzen verhindern, dass umfangreiche Live-, npm-Installations- und Mehrdienste-Lanes gleichzeitig starten. Wenn eine einzelne Lane mehr Ressourcen als die aktiven Obergrenzen benötigt, kann der Scheduler sie dennoch starten, wenn der Pool leer ist, und lässt sie anschließend allein weiterlaufen, bis wieder Kapazität verfügbar ist. Die Standardwerte sind 10 Slots, `OPENCLAW_DOCKER_ALL_LIVE_LIMIT=9`, `OPENCLAW_DOCKER_ALL_NPM_LIMIT=5` und `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT=7`; passen Sie `OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` oder `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT` (und andere `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT`-Überschreibungen) nur an, wenn der Docker-Host über mehr Kapazitätsreserven verfügt. Der Runner führt standardmäßig eine Docker-Vorabprüfung durch, entfernt veraltete OpenClaw-E2E-Container, gibt alle 30 Sekunden den Status aus, speichert die Laufzeiten erfolgreicher Lanes in `.artifacts/docker-tests/lane-timings.json` und verwendet diese Laufzeiten, um bei späteren Ausführungen längere Lanes zuerst zu starten. Verwenden Sie `OPENCLAW_DOCKER_ALL_DRY_RUN=1`, um das gewichtete Lane-Manifest auszugeben, ohne Docker zu bauen oder auszuführen, oder `node scripts/test-docker-all.mjs --plan-json`, um den CI-Plan für ausgewählte Lanes, Paket-/Image-Anforderungen und Anmeldedaten auszugeben.
- `Package Acceptance` ist das GitHub-native Paket-Gate für die Frage „Funktioniert dieser installierbare Tarball als Produkt?“. Es ermittelt ein einzelnes Kandidatenpaket aus `source=npm`, `source=ref`, `source=url`, `source=trusted-url` oder `source=artifact`, lädt es als `package-under-test` hoch und führt anschließend die wiederverwendbaren Docker-E2E-Lanes gegen genau diesen Tarball aus, statt die ausgewählte Referenz erneut zu packen. Die Profile sind nach Umfang geordnet: `smoke`, `package`, `product` und `full` (sowie `custom` für eine explizite Lane-Liste). Informationen zum Paket-/Update-/Plugin-Vertrag, zur Überlebensmatrix für veröffentlichte Upgrades, zu Release-Standardwerten und zur Fehlertriage finden Sie unter [Updates und Plugins testen](/de/help/testing-updates-plugins).
- Build- und Release-Prüfungen führen nach tsdown `scripts/check-cli-bootstrap-imports.mjs` aus. Der Guard durchläuft den statischen gebauten Graphen ab `dist/entry.js` und `dist/cli/run-main.js` und schlägt fehl, wenn dieser Bootstrap-Graph vor der Befehlsweiterleitung statisch ein externes Paket importiert (Commander, Prompt-UI, undici, Protokollierung und ähnliche startintensive Abhängigkeiten zählen alle dazu); außerdem begrenzt er den gebündelten Gateway-Ausführungs-Chunk auf 70 KB und lehnt statische Importe bekannter selten verwendeter Gateway-Pfade (`control-ui-assets`, `diagnostic-stability-bundle`, `onboard-helpers`, `process-respawn`, `restart-sentinel`, `server-close`, `server-reload-handlers`) aus diesem Chunk ab. `scripts/release-check.ts` führt separat Smoke-Tests für die gepackte CLI mit `--help`, `onboard --help`, `doctor --help`, `status --json --timeout 1`, `config schema` und `models list --provider openai` durch.
- Die Legacy-Kompatibilität von Package Acceptance ist auf `2026.4.25` begrenzt (`2026.4.25-beta.*` eingeschlossen). Bis zu diesem Grenzwert toleriert der Test-Harness ausschließlich Metadatenlücken in veröffentlichten Paketen: ausgelassene private QA-Inventareinträge, fehlendes `gateway install --wrapper`, fehlende Patch-Dateien in der aus dem Tarball abgeleiteten Git-Fixture, fehlendes persistiertes `update.channel`, Legacy-Speicherorte für Plugin-Installationsdatensätze, fehlende Persistierung von Marketplace-Installationsdatensätzen und die Migration von Konfigurationsmetadaten während `plugins update`. Bei Paketen nach `2026.4.25` führen diese Pfade strikt zu Fehlern.
- Container-Smoke-Runner: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:npm-onboard-channel-agent`, `test:docker:release-user-journey`, `test:docker:release-typed-onboarding`, `test:docker:release-media-memory`, `test:docker:release-upgrade-user-journey`, `test:docker:release-plugin-marketplace`, `test:docker:skill-install`, `test:docker:update-channel-switch`, `test:docker:upgrade-survivor`, `test:docker:published-upgrade-survivor`, `test:docker:session-runtime-context`, `test:docker:agents-delete-shared-workspace`, `test:docker:gateway-network`, `test:docker:browser-cdp-snapshot`, `test:docker:mcp-channels`, `test:docker:agent-bundle-mcp-tools`, `test:docker:cron-mcp-cleanup`, `test:docker:plugins`, `test:docker:plugin-update`, `test:docker:plugin-lifecycle-matrix` und `test:docker:config-reload` starten einen oder mehrere echte Container und überprüfen Integrationspfade auf höherer Ebene.
- Docker-/Bash-E2E-Lanes, die den gepackten OpenClaw-Tarball über `scripts/lib/openclaw-e2e-instance.sh` installieren, begrenzen `npm install` auf `OPENCLAW_E2E_NPM_INSTALL_TIMEOUT` (Standard: `600s`; setzen Sie `0`, um den Wrapper für das Debugging zu deaktivieren).

Die Docker-Runner für Live-Modelle binden außerdem nur die benötigten CLI-Authentifizierungs-Home-Verzeichnisse ein
(oder alle unterstützten, wenn die Ausführung nicht eingegrenzt ist) und kopieren sie anschließend vor der Ausführung in das
Home-Verzeichnis des Containers, damit OAuth für externe CLIs Token aktualisieren kann,
ohne den Authentifizierungsspeicher des Hosts zu verändern:

- Direkte Modelle: `pnpm test:docker:live-models` (Skript: `scripts/test-live-models-docker.sh`)
- ACP-Bind-Smoke-Test: `pnpm test:docker:live-acp-bind` (Skript: `scripts/test-live-acp-bind-docker.sh`; deckt standardmäßig Claude, Codex und Gemini ab, mit strikter Droid-/OpenCode-Abdeckung über `pnpm test:docker:live-acp-bind:droid` und `pnpm test:docker:live-acp-bind:opencode`)
- CLI-Backend-Smoke-Test: `pnpm test:docker:live-cli-backend` (Skript: `scripts/test-live-cli-backend-docker.sh`)
- Smoke-Test für den Codex-App-Server-Test-Harness: `pnpm test:docker:live-codex-harness` (Skript: `scripts/test-live-codex-harness-docker.sh`)
- Gateway + Entwicklungs-Agent: `pnpm test:docker:live-gateway` (Skript: `scripts/test-live-gateway-models-docker.sh`)
- Observability-Smoke-Tests: `pnpm qa:otel:smoke`, `pnpm qa:prometheus:smoke` und `pnpm qa:observability:smoke` sind private QA-Lanes für Quellcode-Checkouts. Sie sind absichtlich nicht Bestandteil der Paket-Docker-Release-Lanes, da der npm-Tarball QA Lab auslässt.
- Open-WebUI-Live-Smoke-Test: `pnpm test:docker:openwebui` (Skript: `scripts/e2e/openwebui-docker.sh`)
- Onboarding-Assistent (TTY, vollständiges Scaffolding): `pnpm test:docker:onboard` (Skript: `scripts/e2e/onboard-docker.sh`)
- Npm-Tarball-Onboarding-/Kanal-/Agent-Smoke-Test: `pnpm test:docker:npm-onboard-channel-agent` installiert den gepackten OpenClaw-Tarball global in Docker, konfiguriert OpenAI über ein Onboarding mit Umgebungsreferenz sowie standardmäßig Telegram, führt doctor aus und führt eine simulierte OpenAI-Agent-Interaktion aus. Verwenden Sie einen vorab erstellten Tarball mit `OPENCLAW_CURRENT_PACKAGE_TGZ=/path/to/openclaw-*.tgz` erneut, überspringen Sie den Host-Neubuild mit `OPENCLAW_NPM_ONBOARD_HOST_BUILD=0` oder wechseln Sie den Kanal mit `OPENCLAW_NPM_ONBOARD_CHANNEL=discord` oder `OPENCLAW_NPM_ONBOARD_CHANNEL=slack`.

- Smoke-Test für den Release-Benutzerablauf: `pnpm test:docker:release-user-journey` installiert das gepackte OpenClaw-Tarball global in einem sauberen Docker-Home-Verzeichnis, führt das Onboarding aus, konfiguriert einen simulierten OpenAI-Provider, führt einen Agent-Durchlauf aus, installiert/deinstalliert externe Plugins, konfiguriert ClickClack für eine lokale Test-Fixture, überprüft aus- und eingehende Nachrichten, startet den Gateway neu und führt Doctor aus.
- Smoke-Test für typisiertes Release-Onboarding: `pnpm test:docker:release-typed-onboarding` installiert das gepackte Tarball, steuert `openclaw onboard` über ein echtes TTY, konfiguriert OpenAI als Env-Ref-Provider, überprüft, dass kein unformatierter Schlüssel dauerhaft gespeichert wird, und führt einen simulierten Agent-Durchlauf aus.
- Smoke-Test für Release-Medien/-Speicher: `pnpm test:docker:release-media-memory` installiert das gepackte Tarball und überprüft das Bildverständnis anhand eines PNG-Anhangs, die Ausgabe OpenAI-kompatibler Bilderzeugung, den Abruf über die Speichersuche sowie den Erhalt des Abrufs über einen Gateway-Neustart hinweg.
- Smoke-Test für den Benutzerablauf bei Release-Upgrades: `pnpm test:docker:release-upgrade-user-journey` installiert standardmäßig die neueste veröffentlichte Baseline, die älter als das Kandidaten-Tarball ist, konfiguriert den Provider-/Plugin-/ClickClack-Zustand im veröffentlichten Paket, führt ein Upgrade auf das Kandidaten-Tarball durch und wiederholt anschließend den zentralen Agent-/Plugin-/Kanalablauf. Wenn keine ältere veröffentlichte Baseline vorhanden ist, wird die Kandidatenversion erneut verwendet. Überschreiben Sie die Baseline mit `OPENCLAW_RELEASE_UPGRADE_BASELINE_SPEC=openclaw@<version>`.
- Smoke-Test für den Release-Plugin-Marktplatz: `pnpm test:docker:release-plugin-marketplace` installiert aus einem lokalen Fixture-Marktplatz, aktualisiert das installierte Plugin, deinstalliert es und überprüft, dass die Plugin-CLI verschwindet und die Installationsmetadaten bereinigt werden.
- Smoke-Test für die Skill-Installation: `pnpm test:docker:skill-install` installiert das gepackte OpenClaw-Tarball global in Docker, deaktiviert Installationen hochgeladener Archive in der Konfiguration, ermittelt über die Suche den aktuellen Live-ClawHub-Skill-Slug, installiert ihn mit `openclaw skills install` und überprüft den installierten Skill sowie die Herkunfts-/Sperrmetadaten von `.clawhub`.
- Smoke-Test für den Wechsel des Update-Kanals: `pnpm test:docker:update-channel-switch` installiert das gepackte OpenClaw-Tarball global in Docker, wechselt von Paket `stable` zu Git `dev`, überprüft den gespeicherten Kanal und die Plugin-Funktion nach dem Update, wechselt anschließend mit `stable` zurück zum Paket und prüft den Update-Status.
- Smoke-Test für das Überleben eines Upgrades: `pnpm test:docker:upgrade-survivor` installiert das gepackte OpenClaw-Tarball über einer veränderten Fixture eines alten Benutzers mit Agents, Kanalkonfiguration, Plugin-Zulassungslisten, veraltetem Plugin-Abhängigkeitszustand sowie vorhandenen Workspace-/Sitzungsdateien. Der Test führt ein Paket-Update und Doctor nicht interaktiv ohne Live-Provider- oder Kanalschlüssel aus, startet anschließend einen Loopback-Gateway und prüft den Erhalt von Konfiguration und Zustand sowie die Zeitbudgets für Start und Status.
- Smoke-Test für das Überleben eines veröffentlichten Upgrades: `pnpm test:docker:published-upgrade-survivor` installiert standardmäßig `openclaw@latest`, legt realistische Dateien eines bestehenden Benutzers an, konfiguriert diese Baseline mit einem integrierten Befehlsrezept, validiert die resultierende Konfiguration, aktualisiert diese veröffentlichte Installation auf das Kandidaten-Tarball, führt Doctor nicht interaktiv aus, schreibt `.artifacts/upgrade-survivor/summary.json`, startet anschließend einen Loopback-Gateway und prüft konfigurierte Intentionen, Zustandserhalt, Start, `/healthz`, `/readyz` sowie RPC-Statusbudgets. Überschreiben Sie eine Baseline mit `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC`, weisen Sie den aggregierten Scheduler mit `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` an, exakte lokale Baselines wie `openclaw@2026.5.2 openclaw@2026.4.23 openclaw@2026.4.15` zu erweitern, und erweitern Sie Problem-Fixtures mit `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS`, etwa `reported-issues`; der Satz gemeldeter Probleme enthält `configured-plugin-installs` für die automatische Reparatur der Installation externer OpenClaw-Plugins. Package Acceptance stellt diese als `published_upgrade_survivor_baseline`, `published_upgrade_survivor_baselines` und `published_upgrade_survivor_scenarios` bereit, löst Meta-Baseline-Token wie `last-stable-4` oder `all-since-2026.4.23` auf, und Full Release Validation erweitert das Paket-Gate für den Release-Dauertest um `last-stable-4 2026.4.23 2026.5.2 2026.4.15` sowie `reported-issues`.
- Smoke-Test für den Sitzungslaufzeitkontext: `pnpm test:docker:session-runtime-context` überprüft die Transkriptpersistenz des verborgenen Laufzeitkontexts sowie die Doctor-Reparatur betroffener duplizierter Zweige zur Prompt-Umschreibung.
- Smoke-Test für die globale Bun-Installation: `bash scripts/e2e/bun-global-install-smoke.sh` packt den aktuellen Quellbaum, installiert ihn mit `bun install -g` in einem isolierten Home-Verzeichnis und überprüft, dass `openclaw infer image providers --json` gebündelte Bild-Provider zurückgibt, statt zu hängen. Verwenden Sie mit `OPENCLAW_BUN_GLOBAL_SMOKE_PACKAGE_TGZ=/path/to/openclaw-*.tgz` ein vorab erstelltes Tarball erneut, überspringen Sie mit `OPENCLAW_BUN_GLOBAL_SMOKE_HOST_BUILD=0` den Host-Build oder kopieren Sie mit `OPENCLAW_BUN_GLOBAL_SMOKE_DIST_IMAGE=openclaw-dockerfile-smoke:local` die Datei `dist/` aus einem erstellten Docker-Image.
- Docker-Smoke-Test für den Installer: `bash scripts/test-install-sh-docker.sh` verwendet einen gemeinsamen npm-Cache für seine Root-, Update- und Direct-npm-Container. Der Update-Smoke-Test verwendet standardmäßig npm `latest` als stabile Baseline, bevor ein Upgrade auf das Kandidaten-Tarball erfolgt. Überschreiben Sie dies lokal mit `OPENCLAW_INSTALL_SMOKE_UPDATE_BASELINE=2026.4.22` oder auf GitHub mit der Eingabe `update_baseline_version` des Install-Smoke-Workflows. Die Installer-Prüfungen ohne Root-Rechte verwenden einen isolierten npm-Cache, damit Cache-Einträge im Besitz von Root das benutzerlokale Installationsverhalten nicht verdecken. Setzen Sie `OPENCLAW_INSTALL_SMOKE_NPM_CACHE_DIR=/path/to/cache`, um den Root-/Update-/Direct-npm-Cache bei lokalen Wiederholungen erneut zu verwenden.
- Die Install-Smoke-CI überspringt das doppelte globale Direct-npm-Update mit `OPENCLAW_INSTALL_SMOKE_SKIP_NPM_GLOBAL=1`; führen Sie das Skript lokal ohne diese Umgebungsvariable aus, wenn eine direkte Abdeckung von `npm install -g` erforderlich ist.
- CLI-Smoke-Test zum Löschen eines gemeinsam genutzten Workspaces durch Agents: `pnpm test:docker:agents-delete-shared-workspace` (Skript: `scripts/e2e/agents-delete-shared-workspace-docker.sh`) erstellt standardmäßig das Image aus dem Root-Dockerfile, legt in einem isolierten Container-Home-Verzeichnis zwei Agents mit einem Workspace an, führt `agents delete --json` aus und überprüft gültiges JSON sowie das Verhalten des beibehaltenen Workspaces. Verwenden Sie das Install-Smoke-Image mit `OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_IMAGE=openclaw-dockerfile-smoke:local OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_SKIP_BUILD=1` erneut.
- Gateway-Netzwerk und Host-Lebenszyklus: `pnpm test:docker:gateway-network` (Skript: `scripts/e2e/gateway-network-docker.sh`) bewahrt den LAN-WebSocket-Smoke-Test für Authentifizierung und Integrität mit zwei Containern und verwendet anschließend Loopback-Admin-HTTP, um Prepare-Fencing, Zugriff mit beibehaltener Kontrolle, Wiederherstellung nach Fortsetzung und einen vorbereiteten Stopp/Start im selben Container nachzuweisen. Die Neustartprüfung muss abgeschlossen sein, bevor die ursprüngliche Lease abläuft, überprüft, dass der Suspendierungszustand prozesslokal ist, während die persistierte Gateway-Konfiguration und Containeridentität erhalten bleiben, und gibt maschinenlesbares JSON mit den Phasenzeiten aus.
- Smoke-Test für Browser-CDP-Snapshots: `pnpm test:docker:browser-cdp-snapshot` (Skript: `scripts/e2e/browser-cdp-snapshot-docker.sh`) erstellt das Quell-E2E-Image sowie eine Chromium-Schicht, startet Chromium mit unformatierter CDP, führt `browser doctor --deep` aus und überprüft, dass CDP-Rollen-Snapshots Link-URLs, durch den Cursor zu anklickbaren Elementen hochgestufte Objekte, Iframe-Referenzen und Frame-Metadaten abdecken.
- Regression für minimale Schlussfolgerung bei OpenAI Responses web_search: `pnpm test:docker:openai-web-search-minimal` (Skript: `scripts/e2e/openai-web-search-minimal-docker.sh`) führt einen simulierten OpenAI-Server über den Gateway aus, überprüft, dass `web_search` den Wert `reasoning.effort` von `minimal` auf `low` erhöht, erzwingt anschließend die Ablehnung durch das Provider-Schema und prüft, ob die unformatierte Detailangabe in den Gateway-Protokollen erscheint.
- MCP-Kanalbrücke (vorbereiteter Gateway + stdio-Brücke + Smoke-Test für unformatierte Claude-Benachrichtigungs-Frames): `pnpm test:docker:mcp-channels` (Skript: `scripts/e2e/mcp-channels-docker.sh`)
- MCP-Werkzeuge des OpenClaw-Bundles (echter stdio-MCP-Server + Zulassen-/Ablehnen-Smoke-Test mit eingebettetem OpenClaw-Profil): `pnpm test:docker:agent-bundle-mcp-tools` (Skript: `scripts/e2e/agent-bundle-mcp-tools-docker.sh`)
- MCP-Bereinigung für Cron/Subagent (echter Gateway + Beenden des stdio-MCP-Kindprozesses nach isolierten Cron- und einmaligen Subagent-Ausführungen): `pnpm test:docker:cron-mcp-cleanup` (Skript: `scripts/e2e/cron-mcp-cleanup-docker.sh`)
- Plugins (Installations-/Update-Smoke-Test für lokalen Pfad, `file:`, npm-Registry mit hochgezogenen Abhängigkeiten, fehlerhafte npm-Paketmetadaten, veränderliche Git-Referenzen, ClawHub-Kitchen-Sink, Marktplatz-Updates sowie Aktivierung/Inspektion des Claude-Bundles): `pnpm test:docker:plugins` (Skript: `scripts/e2e/plugins-docker.sh`)
  Setzen Sie `OPENCLAW_PLUGINS_E2E_CLAWHUB=0`, um den ClawHub-Block zu überspringen, oder überschreiben Sie das standardmäßige Kitchen-Sink-Paket-/Laufzeitpaar mit `OPENCLAW_PLUGINS_E2E_CLAWHUB_SPEC` und `OPENCLAW_PLUGINS_E2E_CLAWHUB_ID`. Ohne `OPENCLAW_CLAWHUB_URL`/`CLAWHUB_URL` verwendet der Test einen hermetischen lokalen ClawHub-Fixture-Server.
- Smoke-Test für unveränderte Plugin-Updates: `pnpm test:docker:plugin-update` (Skript: `scripts/e2e/plugin-update-unchanged-docker.sh`)
- Smoke-Test für die Plugin-Lebenszyklusmatrix: `pnpm test:docker:plugin-lifecycle-matrix` installiert das gepackte OpenClaw-Tarball in einem leeren Container, installiert ein npm-Plugin, schaltet Aktivierung/Deaktivierung um, führt über eine lokale npm-Registry Upgrades und Downgrades durch, löscht den installierten Code und überprüft anschließend, dass die Deinstallation weiterhin veralteten Zustand entfernt, während für jede Lebenszyklusphase RSS-/CPU-Metriken protokolliert werden.
- Smoke-Test für Metadaten beim Neuladen der Konfiguration: `pnpm test:docker:config-reload` (Skript: `scripts/e2e/config-reload-source-docker.sh`)
- Plugins: `pnpm test:docker:plugins` deckt Installations-/Update-Smoke-Tests für lokalen Pfad, `file:`, npm-Registry mit hochgezogenen Abhängigkeiten, veränderliche Git-Referenzen, ClawHub-Fixtures, Marktplatz-Updates sowie Aktivierung/Inspektion des Claude-Bundles ab. `pnpm test:docker:plugin-update` deckt das Verhalten bei unveränderten Updates installierter Plugins ab. `pnpm test:docker:plugin-lifecycle-matrix` deckt die ressourcenüberwachte Installation, Aktivierung, Deaktivierung, Aktualisierung, Herabstufung und Deinstallation bei fehlendem Code für npm-Plugins ab.

So können Sie das gemeinsam genutzte funktionale Image manuell vorab erstellen und wiederverwenden:

```bash
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local pnpm test:docker:e2e-build
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local OPENCLAW_SKIP_DOCKER_BUILD=1 pnpm test:docker:mcp-channels
```

Suite-spezifische Image-Überschreibungen wie `OPENCLAW_GATEWAY_NETWORK_E2E_IMAGE` haben weiterhin Vorrang, wenn sie gesetzt sind. Wenn `OPENCLAW_SKIP_DOCKER_BUILD=1` auf ein entferntes gemeinsam genutztes Image verweist, laden die Skripte es herunter, sofern es nicht bereits lokal vorhanden ist. Die QR- und Installer-Docker-Tests behalten ihre eigenen Dockerfiles, da sie das Paket-/Installationsverhalten und nicht die gemeinsam genutzte Laufzeit der erstellten Anwendung validieren.

Die Docker-Runner für Live-Modelle binden außerdem den aktuellen Checkout schreibgeschützt ein
und stellen ihn in einem temporären Arbeitsverzeichnis innerhalb des Containers bereit. Dadurch bleibt das
Laufzeit-Image schlank, während Vitest dennoch mit Ihren exakten lokalen
Quellen und Ihrer Konfiguration ausgeführt wird. Der Bereitstellungsschritt überspringt große, nur lokal vorhandene Caches und App-Build-
Ausgaben wie `.pnpm-store`, `.worktrees`, `__openclaw_vitest__` und
app-lokale `.build`- oder Gradle-Ausgabeverzeichnisse, damit Docker-Live-Ausführungen nicht
mehrere Minuten mit dem Kopieren maschinenspezifischer Artefakte verbringen. Außerdem setzen sie
`OPENCLAW_SKIP_CHANNELS=1`, damit Live-Prüfungen des Gateways keine echten
Telegram-/Discord-/usw.-Kanal-Worker innerhalb des Containers starten.
`test:docker:live-models` führt weiterhin `pnpm test:live` aus; reichen Sie daher auch
`OPENCLAW_LIVE_GATEWAY_*` durch, wenn Sie die Live-Abdeckung des Gateways in diesem Docker-Lauf
eingrenzen oder ausschließen müssen.

`test:docker:openwebui` ist ein übergeordneter Kompatibilitäts-Smoke-Test: Er startet einen
OpenClaw-Gateway-Container mit aktivierten OpenAI-kompatiblen HTTP-Endpunkten,
startet einen angehefteten Open-WebUI-Container für diesen Gateway, meldet sich über
Open WebUI an, überprüft, dass `/api/models` den Wert `openclaw/default` bereitstellt, und sendet anschließend eine
echte Chatanfrage über den `/api/chat/completions`-Proxy von Open WebUI. Setzen Sie
`OPENWEBUI_SMOKE_MODE=models` für CI-Prüfungen des Release-Pfads, die
nach der Anmeldung bei Open WebUI und der Modellerkennung beendet werden sollen, ohne auf den Abschluss
eines Live-Modells zu warten. Die erste Ausführung kann merklich langsamer sein, da Docker möglicherweise
das Open-WebUI-Image herunterladen muss und Open WebUI möglicherweise seine eigene
Kaltstart-Einrichtung abschließen muss. Dieser Lauf erwartet einen verwendbaren Live-Modellschlüssel, der über
die Prozessumgebung, bereitgestellte Authentifizierungsprofile oder einen expliziten Wert
`OPENCLAW_PROFILE_FILE` bereitgestellt wird. Erfolgreiche Ausführungen geben eine kleine JSON-Nutzlast wie
`{ "ok": true, "model": "openclaw/default", ... }` aus.

`test:docker:mcp-channels` ist absichtlich deterministisch und benötigt kein
echtes Telegram-, Discord- oder iMessage-Konto. Der Test startet einen vorbereiteten Gateway-
Container, startet einen zweiten Container, der `openclaw mcp serve` erzeugt, und
überprüft anschließend die Erkennung weitergeleiteter Unterhaltungen, das Lesen von Transkripten, Anhangs-
metadaten, das Verhalten der Live-Ereigniswarteschlange, die Weiterleitung ausgehender Sendungen sowie Claude-artige
Kanal- und Berechtigungsbenachrichtigungen über die echte stdio-MCP-Brücke. Die
Benachrichtigungsprüfung untersucht die unformatierten stdio-MCP-Frames direkt, sodass der Smoke-Test
validiert, was die Brücke tatsächlich ausgibt, und nicht nur, was ein bestimmtes Client-SDK
zufällig bereitstellt.

`test:docker:agent-bundle-mcp-tools` ist deterministisch und benötigt keinen
Live-Modellschlüssel. Es erstellt das Docker-Image des Repositorys, startet einen echten stdio-MCP-
Probe-Server im Container, stellt diesen Server über die
eingebettete MCP-Laufzeit des OpenClaw-Bundles bereit, führt das Tool aus und überprüft anschließend,
dass `coding` und `messaging` die `bundle-mcp`-Tools beibehalten, während `minimal` und
`tools.deny: ["bundle-mcp"]` sie herausfiltern.

`test:docker:cron-mcp-cleanup` ist deterministisch und benötigt keinen Live-
Modellschlüssel. Es startet einen mit Startdaten versehenen Gateway mit einem echten stdio-MCP-Probe-Server,
führt einen isolierten Cron-Durchlauf und einen einmaligen `sessions_spawn`-Kinddurchlauf aus und
überprüft anschließend, dass der MCP-Kindprozess nach jedem Durchlauf beendet wird.

Manueller ACP-Smoke-Test für Threads in natürlicher Sprache (nicht CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Bewahren Sie dieses Skript für Regressions-/Debugging-Workflows auf. Es kann erneut für die Validierung des ACP-Thread-Routings benötigt werden; löschen Sie es daher nicht.

Nützliche Umgebungsvariablen:

- `OPENCLAW_CONFIG_DIR=...` (Standard: `~/.openclaw`) wird unter `/home/node/.openclaw` eingebunden
- `OPENCLAW_WORKSPACE_DIR=...` (Standard: `~/.openclaw/workspace`) wird unter `/home/node/.openclaw/workspace` eingebunden
- `OPENCLAW_PROFILE_FILE=...` wird eingebunden und vor der Ausführung von Tests eingelesen
- `OPENCLAW_DOCKER_PROFILE_ENV_ONLY=1`, um ausschließlich aus `OPENCLAW_PROFILE_FILE` eingelesene Umgebungsvariablen zu überprüfen; dabei werden temporäre Konfigurations-/Arbeitsbereichsverzeichnisse und keine Einbindungen externer CLI-Authentifizierungen verwendet
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (Standard: `~/.cache/openclaw/docker-cli-tools`, sofern der Durchlauf nicht bereits ein CI-/verwaltetes Bind-Verzeichnis verwendet) wird für zwischengespeicherte CLI-Installationen in Docker unter `/home/node/.npm-global` eingebunden
- Externe CLI-Authentifizierungsverzeichnisse/-dateien unter `$HOME` werden unter `/host-auth...` schreibgeschützt eingebunden und anschließend vor Beginn der Tests nach `/home/node/...` kopiert
  - Standardverzeichnisse (werden verwendet, wenn der Durchlauf nicht auf bestimmte Provider beschränkt ist): `.factory`, `.gemini`, `.minimax`
  - Standarddateien: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Bei auf Provider beschränkten Durchläufen werden nur die benötigten Verzeichnisse/Dateien eingebunden, die aus `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` abgeleitet werden
  - Manuelle Überschreibung mit `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` oder einer kommagetrennten Liste wie `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`, um den Durchlauf einzuschränken
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`, um Provider im Container zu filtern
- `OPENCLAW_SKIP_DOCKER_BUILD=1`, um für erneute Durchläufe, die keinen Neuaufbau benötigen, ein vorhandenes `openclaw:local-live`-Image wiederzuverwenden
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`, um sicherzustellen, dass Anmeldedaten aus dem Profilspeicher (nicht aus der Umgebung) stammen
- `OPENCLAW_OPENWEBUI_MODEL=...`, um das vom Gateway für den Open-WebUI-Smoke-Test bereitgestellte Modell auszuwählen
- `OPENCLAW_OPENWEBUI_PROMPT=...`, um den vom Open-WebUI-Smoke-Test verwendeten Prompt zur Nonce-Prüfung zu überschreiben
- `OPENWEBUI_IMAGE=...`, um das festgelegte Open-WebUI-Image-Tag zu überschreiben

## Plausibilitätsprüfung der Dokumentation

Führen Sie nach Änderungen an der Dokumentation die Dokumentationsprüfungen aus: `pnpm check:docs`.
Führen Sie die vollständige Mintlify-Ankerüberprüfung aus, wenn Sie auch seiteninterne Überschriften prüfen müssen: `pnpm docs:check-links:anchors`.

## Offline-Regression (CI-sicher)

Dies sind Regressionen der „echten Pipeline“ ohne echte Provider:

- Tool-Aufrufe über den Gateway (OpenAI-Mock, echter Gateway + Agentenschleife): `src/gateway/gateway.test.ts` (Fall: „führt einen OpenAI-Mock-Tool-Aufruf vollständig über die Gateway-Agentenschleife aus“)
- Gateway-Assistent (WS `wizard.start`/`wizard.next`, schreibt Konfiguration + Authentifizierung erzwungen): `src/gateway/gateway.test.ts` (Fall: „führt den Assistenten über WebSocket aus und schreibt die Authentifizierungstoken-Konfiguration“)

## Zuverlässigkeitsevaluierungen für Agenten (Skills)

Es gibt bereits einige CI-sichere Tests, die sich wie „Zuverlässigkeitsevaluierungen für Agenten“ verhalten:

- Mock-Tool-Aufrufe über den echten Gateway + die Agentenschleife (`src/gateway/gateway.test.ts`).
- Vollständige Assistentenabläufe, die die Sitzungsverdrahtung und Konfigurationsauswirkungen validieren (`src/gateway/gateway.test.ts`).

Was für Skills noch fehlt (siehe [Skills](/de/tools/skills)):

- **Entscheidungsfindung:** Wählt der Agent den richtigen Skill aus (oder vermeidet irrelevante), wenn Skills im Prompt aufgeführt sind?
- **Einhaltung:** Liest der Agent vor der Verwendung `SKILL.md` und befolgt er die erforderlichen Schritte/Argumente?
- **Workflow-Verträge:** Mehrstufige Szenarien, die die Tool-Reihenfolge, die Übernahme des Sitzungsverlaufs und Sandbox-Grenzen prüfen.

Zukünftige Evaluierungen sollten zunächst deterministisch bleiben:

- Ein Szenario-Runner, der Mock-Provider verwendet, um Tool-Aufrufe und deren Reihenfolge, das Lesen von Skill-Dateien sowie die Sitzungsverdrahtung zu prüfen.
- Eine kleine Suite auf Skills ausgerichteter Szenarien (verwenden oder vermeiden, Zugriffsbeschränkung, Prompt-Injection).
- Optionale Live-Evaluierungen (explizit aktiviert, durch Umgebungsvariablen gesteuert) erst, nachdem die CI-sichere Suite vorhanden ist.

## Vertragstests (Plugin- und Kanalstruktur)

Vertragstests überprüfen, ob jedes registrierte Plugin und jeder registrierte Kanal seinem
Schnittstellenvertrag entspricht. Sie durchlaufen alle ermittelten Plugins und führen eine
Suite von Struktur- und Verhaltensprüfungen aus. Die standardmäßige `pnpm test`-Unit-Testspur
überspringt diese gemeinsamen Schnittstellen- und Smoke-Dateien absichtlich; führen Sie die Vertrags-
befehle ausdrücklich aus, wenn Sie gemeinsame Kanal- oder Provider-Oberflächen ändern.

### Befehle

- Alle Verträge: `pnpm test:contracts`
- Nur Kanalverträge: `pnpm test:contracts:channels`
- Nur Provider-Verträge: `pnpm test:contracts:plugins`

### Kanalverträge

Zu finden in `src/channels/plugins/contracts/*.contract.test.ts`. Aktuelle
Kategorien der obersten Ebene:

- **channel-catalog** – Metadaten der Kanalkatalogeinträge aus Bundle/Registry
- **plugin** (Registry-gestützt, aufgeteilt) – grundlegende Struktur der Plugin-Registrierung
- **surfaces-only** (Registry-gestützt, aufgeteilt) – Strukturprüfungen je Oberfläche für `actions`, `setup`, `status`, `outbound`, `messaging`, `threading`, `directory` und `gateway`
- **session-binding** (Registry-gestützt) – Verhalten der Sitzungsbindung
- **outbound-payload** – Struktur und Normalisierung der Nachrichtennutzlast
- **group-policy** (Fallback) – Durchsetzung der standardmäßigen Gruppenrichtlinie je Kanal
- **threading** (Registry-gestützt, aufgeteilt) – Verarbeitung von Thread-IDs
- **directory** (Registry-gestützt, aufgeteilt) – Verzeichnis-/Teilnehmerlisten-API
- **registry** und **plugins-core.\*** – interne Abläufe der Kanal-Plugin-Registry, des Loaders und der Autorisierung von Konfigurationsschreibvorgängen

Die von diesen Suites verwendeten Harness-Hilfsfunktionen zur Erfassung eingehender Weiterleitungen und für ausgehende Nutzlasten
werden intern über `src/plugin-sdk/channel-contract-testing.ts` bereitgestellt
(von npm ausgeschlossen, kein öffentlicher SDK-Unterpfad); in diesem Verzeichnis gibt es keine eigenständige
`inbound.contract.test.ts`-Datei.

### Provider-Verträge

Zu finden in `src/plugins/contracts/*.contract.test.ts`. Zu den aktuellen Kategorien
gehören:

- **shape** – Struktur des Plugin-Manifests sowie der API- und Laufzeitexporte
- **plugin-registration** (+ parallel) – Fälle der Manifestregistrierung
- **package-manifest** – Anforderungen an das Paketmanifest
- **loader** – Einrichtungs-/Bereinigungsverhalten des Plugin-Loaders
- **registry** – Inhalte und Suche der Plugin-Vertrags-Registry
- **providers** – gemeinsames Provider-Verhalten für gebündelte Provider sowie Websuch-Provider
- **auth-choice** – Metadaten zur Authentifizierungsauswahl und Einrichtungsverhalten
- **provider-catalog-deprecation** – Metadaten zu veralteten Provider-Katalogen
- **wizard.choice-resolution**, **wizard.model-picker**, **wizard.setup-options** – Verträge des Provider-Einrichtungsassistenten
- **embedding-provider**, **memory-embedding-provider**, **web-fetch-provider**, **tts** – funktionsspezifische Provider-Verträge
- **session-actions**, **session-attachments**, **session-entry-projection** – Plugin-eigene Verträge für den Sitzungsstatus
- **scheduled-turns** – Metadaten geplanter Plugin-Durchläufe und Zeitstempelgrenzen
- **host-hooks**, **run-context-lifecycle**, **runtime-import-side-effects**, **runtime-seams** – Verträge für Plugin-Host-/Laufzeitlebenszyklus und Importgrenzen
- **extension-runtime-dependencies** – Platzierung von Laufzeitabhängigkeiten für Erweiterungen

### Wann ausführen

- Nach dem Ändern von plugin-sdk-Exporten oder -Unterpfaden
- Nach dem Hinzufügen oder Ändern eines Kanal- oder Provider-Plugins
- Nach dem Refactoring der Plugin-Registrierung oder -Ermittlung

Vertragstests werden in der CI ausgeführt und benötigen keine echten API-Schlüssel.

## Regressionen hinzufügen (Leitfaden)

Wenn Sie ein im Live-Betrieb entdecktes Provider-/Modellproblem beheben:

- Fügen Sie nach Möglichkeit eine CI-sichere Regression hinzu (Mock-/Stub-Provider oder Erfassung der exakten Transformation der Anfrageform)
- Wenn das Problem grundsätzlich nur im Live-Betrieb auftritt (Ratenbegrenzungen, Authentifizierungsrichtlinien), halten Sie den Live-Test eng begrenzt und aktivieren Sie ihn explizit über Umgebungsvariablen
- Bevorzugen Sie die kleinste Ebene, die den Fehler erkennt:
  - Fehler bei der Konvertierung/Wiedergabe von Provider-Anfragen -> direkter Modelltest
  - Fehler in der Gateway-Sitzungs-/Verlaufs-/Tool-Pipeline -> Gateway-Live-Smoke-Test oder CI-sicherer Gateway-Mock-Test
- SecretRef-Schutz vor Verzeichnisdurchquerung:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` leitet aus Registry-Metadaten (`listSecretTargetRegistryEntries()`) ein Stichprobenziel je SecretRef-Klasse ab und prüft anschließend, dass Ausführungs-IDs mit Segmenten zur Verzeichnisdurchquerung abgelehnt werden.
  - Wenn Sie in `src/secrets/target-registry-data.ts` eine neue Familie von `includeInPlan`-SecretRef-Zielen hinzufügen, aktualisieren Sie `classifyTargetClass` in diesem Test. Der Test schlägt bei nicht klassifizierten Ziel-IDs absichtlich fehl, damit neue Klassen nicht unbemerkt übersprungen werden können.

## Verwandte Themen

- [Live-Tests](/de/help/testing-live)
- [Testen von Aktualisierungen und Plugins](/de/help/testing-updates-plugins)
- [CI](/de/ci)
