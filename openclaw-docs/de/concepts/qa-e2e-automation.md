---
doc-schema-version: 1
read_when:
    - Verstehen, wie der QA-Stack zusammenspielt
    - qa-lab, qa-channel oder einen Transportadapter erweitern
    - Repository-gestützte QA-Szenarien hinzufügen
    - Aufbau einer realitätsnäheren QA-Automatisierung rund um das Gateway-Dashboard
summary: 'Überblick über den QA-Stack: qa-lab, qa-channel, Repository-gestützte Szenarien, Live-Transport-Lanes, Transportadapter und Berichterstellung.'
title: QA-Übersicht
x-i18n:
    generated_at: "2026-07-26T18:24:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91c34a50e6197195d57228d92b19caff1785ceaa5d82d7c88a1ec0ed76abd635
    source_path: concepts/qa-e2e-automation.md
    workflow: 16
---

Der private QA-Stack testet OpenClaw auf realistische, an Channels angelehnte Weise, die
ein Unit-Test nicht leisten kann.

Komponenten:

- `extensions/qa-channel`: synthetischer Nachrichten-Channel mit Oberflächen für DMs, Channels, Threads,
  Reaktionen, Bearbeitungen und Löschungen.
- `extensions/qa-lab`: Debugger-UI, QA-Bus, Szenarioprofile und Live-
  Transportadapter zum Beobachten des Transkripts, Einspeisen eingehender Nachrichten
  und Exportieren eines Markdown-Berichts.
- `qa/`: Repository-gestützte Ausgangsartefakte für die Auftaktaufgabe und grundlegende QA-
  Szenarien.
- [Mantis](/de/concepts/mantis): Live-Verifizierung vor und nach Änderungen für Fehler, die
  reale Transporte, Browser-Screenshots, VM-Zustand und PR-Nachweise erfordern.

## Befehlsoberfläche

Jeder QA-Ablauf wird unter `pnpm openclaw qa <subcommand>` ausgeführt. Viele verfügen über `pnpm qa:*`-
Skriptaliase; beide Formen funktionieren.

| Befehl                                              | Zweck                                                                                                                                                                                                                                                               |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `qa run`                                            | Gebündelte QA-Selbstprüfung ohne `--qa-profile`; taxonomiegestützter Runner für Reifegradprofile mit `--qa-profile smoke-ci`, `--qa-profile release` oder `--qa-profile all`.                                                                                         |
| `qa suite`                                          | Repository-gestützte Szenarien im QA-Gateway-Lane ausführen. `--runner multipass` verwendet statt des Hosts eine kurzlebige Linux-VM.                                                                                                                               |
| `qa coverage`                                       | YAML-Inventar der Szenarioabdeckung ausgeben (`--json` für maschinenlesbare Ausgabe; `--match <query>` zum Ermitteln von Szenarien für ein betroffenes Verhalten; `--tools` für die Abdeckung von Runtime-Tool-Fixtures).                                      |
| `qa parity-report`                                  | Zwei `qa-suite-summary.json`-Dateien für ein Paritäts-Gate der Modellachse vergleichen oder mit `--runtime-axis --token-efficiency` Berichte zur Runtime-Parität und Token-Effizienz von Codex und OpenClaw schreiben.                                                   |
| `qa confidence-report`                              | QA-Nachweisartefakte anhand eines Manifests in einen Konfidenzbericht ohne unbekannte Elemente klassifizieren.                                                                                                                                                       |
| `qa confidence-self-test`                           | Mit Ausgangsdaten versehene Negativkontroll-Canarys schreiben, die belegen, dass das Konfidenz-Gate Abweichungen erkennt.                                                                                                                                             |
| `qa jsonl-replay`                                   | Kuratierte JSONL-Transkripte über den Replay-Harness für Runtime-Parität wiedergeben.                                                                                                                                                                                 |
| `qa character-eval`                                 | Das Charakter-QA-Szenario mit mehreren Live-Modellen ausführen und einen bewerteten Bericht erstellen. Siehe [Berichterstellung](#reporting).                                                                                                                          |
| `qa manual`                                         | Einen einmaligen Prompt im ausgewählten Provider-/Modell-Lane ausführen.                                                                                                                                                                                              |
| `qa ui`                                             | Die QA-Debugger-UI und den lokalen QA-Bus starten (Alias: `pnpm qa:lab:ui`).                                                                                                                                                                                         |
| `qa docker-build-image`                             | Das vorgefertigte QA-Docker-Image erstellen.                                                                                                                                                                                                                         |
| `qa docker-scaffold`                                | Ein Docker-Compose-Grundgerüst für das QA-Dashboard und den Gateway-Lane schreiben.                                                                                                                                                                                   |
| `qa up`                                             | Die QA-Site erstellen, den Docker-gestützten Stack starten und die URL ausgeben (Alias: `pnpm qa:lab:up`; die Variante `:fast` fügt `--use-prebuilt-image --bind-ui-dist --skip-ui-build` hinzu).                                                                  |
| `qa aimock`                                         | Nur den AIMock-Provider-Server starten.                                                                                                                                                                                                                              |
| `qa mock-openai`                                    | Nur den szenariobewussten `mock-openai`-Provider-Server starten.                                                                                                                                                                                                    |
| `qa credentials doctor` / `add` / `list` / `remove` | Den gemeinsam genutzten Convex-Anmeldedatenpool verwalten.                                                                                                                                                                                                           |
| `qa discord`                                        | Live-Transport-Lane für einen echten privaten Discord-Guild-Channel.                                                                                                                                                                                                 |
| `qa matrix`                                         | QA-Lab-Matrix-Profile für einen kurzlebigen Tuwunel-Homeserver. Siehe [Matrix-Smoke-Lanes](#matrix-smoke-lanes).                                                                                                                                                      |
| `qa slack`                                          | Live-Transport-Lane für einen echten privaten Slack-Channel.                                                                                                                                                                                                         |
| `qa telegram`                                       | Live-Transport-Lane für eine echte private Telegram-Gruppe.                                                                                                                                                                                                          |
| `qa whatsapp`                                       | Live-Transport-Lane für echte WhatsApp-Web-Konten.                                                                                                                                                                                                                    |
| `qa mantis`                                         | Runner zur Verifizierung vor und nach Änderungen bei Live-Transportfehlern, mit Nachweisen durch Discord-Statusreaktionen, Crabbox-Desktop-/Browser-Smoke und Slack-in-VNC-Smoke. Siehe [Mantis](/de/concepts/mantis) und [Mantis-Slack-Desktop-Runbook](/de/concepts/mantis-slack-desktop-runbook). |

### Profilgestütztes `qa run`

Profilgestütztes `qa run` liest die Mitgliedschaft aus `taxonomy.yaml` und leitet
die aufgelösten Szenarien anschließend über `qa suite` weiter. `--surface` und `--category` filtern
das ausgewählte Profil, anstatt separate Lanes zu definieren. Das resultierende
`qa-evidence.json` enthält eine Scorecard-Zusammenfassung des Profils mit der Anzahl ausgewählter Kategorien
und IDs fehlender Abdeckung; die einzelnen Nachweiseinträge bleiben die
maßgebliche Quelle für Tests, Abdeckungsrollen und Ergebnisse. Abdeckungs-IDs
für Taxonomiemerkmale sind exakte Nachweisziele und keine Aliase: Die primäre Szenarioabdeckung
erfüllt übereinstimmende IDs, während die sekundäre Abdeckung lediglich informativ bleibt. Jede Abdeckungs-
ID lautet exakt `taxonomy-surface.feature` und verwendet die kurze Oberflächen-ID aus
`taxonomy.yaml`. Das separate Feld `surface` eines Szenarios ist eine Bezeichnung für Ausführung und Berichterstellung
(zum Beispiel `channel` oder `runtime-tool`); es definiert nicht die Zuständigkeit
innerhalb der Taxonomie.

Kompakte Nachweise lassen `execution` pro Eintrag weg und setzen `evidenceMode: "slim"`;
`smoke-ci` verwendet standardmäßig die kompakte Form, und `--evidence-mode full` stellt vollständige Einträge wieder her:

```bash
pnpm openclaw qa run \
  --qa-profile smoke-ci \
  --category channels.conversation-routing-and-delivery \
  --provider-mode mock-openai \
  --output-dir .artifacts/qa-e2e/smoke-ci-profile-dispatch
```

Verwenden Sie `smoke-ci` für deterministische Profilnachweise mit Mock-Modell-Providern und
lokalen Crabline-Provider-Servern. Verwenden Sie `release` für Stable-/LTS-Nachweise mit
Live-Channels. Verwenden Sie `all` nur für explizite Nachweisläufe der vollständigen Taxonomie; es
wählt jede aktive Reifegradkategorie aus und kann über den `QA
Profile Evidence`-GitHub-Actions-Workflow mit `qa_profile=all` ausgeführt werden. Wenn ein
Befehl außerdem ein OpenClaw-Root-Profil benötigt, setzen Sie das Root-Profil vor den
QA-Befehl:

```bash
pnpm openclaw --profile work qa run --qa-profile smoke-ci
```

## Bedienungsablauf

Der aktuelle QA-Bedienungsablauf ist eine zweigeteilte QA-Site:

- Links: Gateway-Dashboard (Control UI) mit dem Agenten.
- Rechts: QA Lab mit dem Slack-ähnlichen Transkript und Szenarioplan.

Führen Sie sie wie folgt aus:

```bash
pnpm qa:lab:up
```

Dadurch wird die QA-Site erstellt, der Docker-gestützte Gateway-Lane gestartet und
die QA-Lab-Seite bereitgestellt, auf der ein Operator oder eine Automatisierungsschleife dem Agenten eine QA-
Mission geben, reales Channel-Verhalten beobachten und aufzeichnen kann, was funktioniert hat, fehlgeschlagen ist oder
weiterhin blockiert blieb.

Für schnellere Iterationen an der QA-Lab-UI, ohne das Docker-Image jedes Mal neu zu erstellen,
starten Sie den Stack mit einem über Bind-Mount eingebundenen QA-Lab-Bundle:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` belässt die Docker-Dienste auf einem vorgefertigten Image und
bindet `extensions/qa-lab/web/dist` per Bind-Mount in den `qa-lab`-Container ein.
`qa:lab:watch` erstellt dieses Bundle bei Änderungen neu, und der Browser lädt
automatisch neu, wenn sich der Asset-Hash von QA Lab ändert.

### Observability-Smoke-Tests

<Note>
Observability-QA ist weiterhin nur aus einem Quellcode-Checkout verfügbar. Das npm-Tarball lässt
QA Lab (und `qa-channel`) absichtlich aus, weshalb Docker-Release-Lanes für Pakete
keine `qa`-Befehle ausführen. Führen Sie diese aus einem erstellten Quellcode-Checkout aus, wenn
Sie die Diagnoseinstrumentierung ändern.
</Note>

| Alias                                   | Was ausgeführt wird                                                                                                                      |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm qa:otel:smoke`                    | Lokaler OpenTelemetry-Empfänger plus das Szenario `otel-trace-smoke` mit aktiviertem `diagnostics-otel`.                                 |
| `pnpm qa:otel:collector-smoke`          | Dieselbe Lane hinter einem echten OpenTelemetry-Collector-Docker-Container. Verwenden Sie sie bei Änderungen an der Endpunktverdrahtung oder der Collector-/OTLP-Kompatibilität. |
| `pnpm qa:prometheus:smoke`              | Das Szenario `docker-prometheus-smoke` mit aktiviertem `diagnostics-prometheus`.                                                        |
| `pnpm qa:observability:smoke`           | `qa:otel:smoke`, gefolgt von `qa:prometheus:smoke`.                                                                                      |
| `pnpm qa:observability:collector-smoke` | `qa:otel:collector-smoke`, gefolgt von `qa:prometheus:smoke`.                                                                            |

`qa:otel:smoke` startet einen lokalen OTLP/HTTP-Empfänger, führt einen minimalen Agenten-Turn
über den QA-Kanal aus und stellt anschließend sicher, dass Traces, Metriken und Protokolle exportiert werden. Dabei werden
die exportierten Protobuf-Trace-Spans dekodiert und die für das Release kritische Struktur geprüft:
`openclaw.run`, `openclaw.harness.run`, ein Modellaufruf-Span gemäß der neuesten semantischen GenAI-Konvention,
`openclaw.context.assembled` und `openclaw.message.delivery`
müssen alle vorhanden sein. Der Smoke-Test erzwingt
`OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`, daher muss der Modellaufruf-Span
den Namen `{gen_ai.operation.name} {gen_ai.request.model}` verwenden; bei erfolgreichen Turns
dürfen Modellaufrufe `StreamAbandoned` nicht exportieren; unverarbeitete Diagnose-IDs
und `openclaw.content.*`-Attribute dürfen nicht im Trace erscheinen. Der Szenario-
Prompt fordert das Modell auf, mit einer festen Markierung zu antworten und eine feste
geheime Zeichenfolge zurückzuhalten; die unverarbeiteten OTLP-Nutzdaten dürfen weder diese Werte
noch den aus der Szenario-ID abgeleiteten QA-Sitzungsschlüssel enthalten. Der Test schreibt `otel-smoke-summary.json`
neben die Artefakte der QA-Suite.

`qa:prometheus:smoke` prüft, dass nicht authentifizierte Scrapes abgelehnt werden, und
prüft anschließend, dass der authentifizierte Scrape die für das Release kritischen Metrikfamilien
ohne Prompt-Inhalt, Antwortinhalt, unverarbeitete Diagnosekennungen, Authentifizierungs-
Tokens oder lokale Pfade enthält.

### Matrix-Smoke-Lanes

Führen Sie für eine transportechte Matrix-Smoke-Lane, die keine Zugangsdaten
für einen Modell-Provider erfordert, das Release-Profil mit dem deterministischen OpenAI-Mock-Provider aus:

```bash
pnpm openclaw qa matrix --provider-mode mock-openai --profile release
```

Geben Sie für die Live-Frontier-Provider-Lane OpenAI-kompatible Zugangsdaten
explizit an:

```bash
OPENCLAW_LIVE_OPENAI_KEY="${OPENAI_API_KEY}" \
  pnpm openclaw qa matrix --provider-mode live-frontier --profile release
```

Ein einfaches `pnpm openclaw qa matrix` führt das vollständige Profil `all` aus und fährt nach
Szenariofehlern fort. Verwenden Sie `--fail-fast` für eine kürzere Feedbackschleife oder wiederholen Sie
`--scenario <id>`, um einzelne Szenarien auszuwählen; explizite Szenario-IDs haben
Vorrang vor `--profile`.

| Profil       | Szenarien | Zweck                                                                                                                                    |
| ------------ | --------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `all`        | 93        | Vollständiger Katalog (Standard).                                                                                                        |
| `release`    | 2         | Für das Release kritische Kanal-Baseline und Neuladen der Live-Zulassungsliste.                                                          |
| `fast`       | 12        | Fokussierte Abdeckung von Threads, Reaktionen, Genehmigungen, Richtlinien, Bot-Gating und verschlüsselten Antworten.                     |
| `transport`  | 50        | Threads, DM-/Raum-Routing, automatischer Beitritt, Genehmigungen, Reaktionen, Neustarts, Erwähnungs-/Zulassungslistenrichtlinien, Bearbeitungen und Reihenfolge mehrerer Akteure. |
| `media`      | 7         | Abdeckung von Bildern, generierten Bildern, Sprache, Anhängen, nicht unterstützten Medien und verschlüsselten Medien.                    |
| `e2ee-smoke` | 8         | Mindestabdeckung für verschlüsselte Antworten, Threads, Bootstrap, Wiederherstellung, Neustart, Schwärzung und Fehler.                   |
| `e2ee-deep`  | 18        | Zustandsverlust, Sicherung, Schlüsselwiederherstellung, Gerätehygiene und SAS-/QR-/DM-Verifizierung.                                     |
| `e2ee-cli`   | 9         | `openclaw matrix encryption setup`-, Wiederherstellungsschlüssel-, Mehrfachkonto-, Gateway-Roundtrip- und Selbstverifizierungsbefehle über das Harness. |

Profilzugehörigkeit und Kanalanforderungen befinden sich bei den deklarativen Matrix-
Szenarien unter `qa/scenarios/channels/`. Der Lauf wählt den Kanaltreiber aus.
Die Live-Implementierungen befinden sich unter
`extensions/qa-lab/src/live-transports/matrix/scenarios/`.

Der Adapter stellt einen temporären Tuwunel-Homeserver in Docker bereit (Standard-
Image `ghcr.io/matrix-construct/tuwunel:v1.5.1`, Servername `matrix-qa.test`,
Port `28008`), registriert temporäre Treiber-, SUT- und Beobachterbenutzer, initialisiert die
erforderlichen Räume und zeichnet die geschwärzte Anfrage-/Antwortgrenze auf. Anschließend
führt er das echte Matrix-Plugin in einem untergeordneten QA-Gateway aus, das auf diesen Transport
beschränkt ist (kein `qa-channel`), und baut die Umgebung wieder ab.

Häufig verwendete Optionen:

| Flag                     | Standard          | Zweck                                                                                |
| ------------------------ | ----------------- | ------------------------------------------------------------------------------------ |
| `--profile <profile>`    | `all`             | Wählt eines der oben aufgeführten Profile aus.                                      |
| `--scenario <id>`        | -                 | Wählt ein Szenario aus; wiederholbar.                                                |
| `--fail-fast`            | aus               | Beendet den Lauf nach der ersten fehlgeschlagenen Prüfung oder dem ersten fehlgeschlagenen Szenario. |
| `--allow-failures`       | aus               | Schreibt Artefakte, ohne bei Szenariofehlern einen fehlerhaften Exit-Code zurückzugeben. |
| `--provider-mode <mode>` | `live-frontier`   | Verwendet `mock-openai` für deterministische Verteilung oder `live-frontier` für einen Live-Provider. |
| `--model <ref>`          | Provider-Standard | Legt die primäre `provider/model`-Referenz fest.                                  |
| `--alt-model <ref>`      | Provider-Standard | Legt das alternative Modell für Szenarien fest, die zwischen Modellen wechseln.      |
| `--fast`                 | aus               | Aktiviert den schnellen Provider-Modus, sofern unterstützt.                          |
| `--output-dir <path>`    | generiert         | Wählt das Berichtsverzeichnis aus; relative Pfade werden relativ zu `--repo-root` aufgelöst. |
| `--repo-root <path>`     | aktuelles Verzeichnis | Führt den Lauf aus einem neutralen Arbeitsverzeichnis aus.                        |
| `--sut-account <id>`     | `sut`             | Wählt die Matrix-Konto-ID in der Konfiguration des untergeordneten Gateways aus.     |

Matrix-QA least keine gemeinsam genutzten Matrix-Zugangsdaten: Der Adapter erstellt
lokal temporäre Benutzer und akzeptiert daher weder `--credential-source` noch
`--credential-role`. Überschreiben Sie das Homeserver-Image mit
`OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE`; passen Sie negative Prüfungen auf ausbleibende Antworten mit
`OPENCLAW_QA_MATRIX_NO_REPLY_WINDOW_MS` an (Standard `8000`, begrenzt auf das aktive
Szenario-Zeitlimit). Der Einmalbefehl erzwingt normalerweise einen sauberen Prozessabbruch, nachdem
die Artefakte vollständig geschrieben wurden, da native Matrix-Kryptografie-Handles die Bereinigung überdauern können; setzen Sie
`OPENCLAW_QA_MATRIX_DISABLE_FORCE_EXIT=1` nur für ein direktes Test-Harness, bei dem
der Befehl stattdessen zurückkehren muss.

Jeder Lauf schreibt die üblichen QA-Lab-Artefakte in das ausgewählte Ausgabe-
verzeichnis: `qa-suite-report.md`, `qa-suite-summary.json` und
`qa-evidence.json`. Wenn die Bereinigung fehlschlägt, führen Sie den ausgegebenen
Wiederherstellungsbefehl `docker compose ... down --remove-orphans` aus. Erhöhen Sie auf langsamen Runnern
das Zeitfenster für ausbleibende Antworten; in einer schnellen CI kann ein kleineres Zeitfenster negative
Prüfungen verkürzen.

Die Szenarien decken Transportverhalten ab, das Unit-Tests nicht durchgängig
nachweisen können: Erwähnungs-Gating, Richtlinien zum Zulassen von Bots, Zulassungslisten, Antworten auf oberster Ebene und in
Threads, DM-Routing, Reaktionsverarbeitung, Unterdrückung eingehender Bearbeitungen, Deduplizierung bei der Wiedergabe
nach einem Neustart, Wiederherstellung nach einer Homeserver-Unterbrechung, Übermittlung von Genehmigungsmetadaten,
Medienverarbeitung sowie Bootstrap-, Wiederherstellungs- und Verifizierungsabläufe für Matrix E2EE. Das
E2EE-CLI-Profil führt außerdem `openclaw matrix encryption setup` und
Verifizierungsbefehle über denselben temporären Homeserver aus, bevor
Gateway-Antworten geprüft werden.

`matrix-room-block-streaming` und `subagent-thread-spawn` bleiben durch
explizite Auswahl mit `--scenario` verfügbar, gehören jedoch nicht zum standardmäßigen Profil `all`.

Die CI verwendet dieselbe Befehlsoberfläche in
`.github/workflows/qa-live-transports-convex.yml`. Geplante Läufe und Release-Läufe
führen die Release-Szenarien aus. Manuelle `matrix_profile=all`-Ausführungen verteilen
die Profile `transport`, `media`, `e2ee-smoke`, `e2ee-deep` und `e2ee-cli`;
fokussierte Ausführungen wählen `fast`, `release` oder `transport` in einem Job aus.

### Discord-Mantis-Szenarien

Discord verfügt außerdem über ausschließlich für Mantis vorgesehene Opt-in-Szenarien zur Reproduktion von Fehlern. Verwenden Sie
`--scenario discord-status-reactions-tool-only` für die explizite Zeitleiste
der Statusreaktionen oder `--scenario discord-thread-reply-filepath-attachment`,
um einen echten Discord-Thread zu erstellen und zu prüfen, dass `message.thread-reply`
einen `filePath`-Anhang beibehält. Diese Szenarien gehören nicht zur standardmäßigen
Live-Discord-Lane, da sie Vorher-/Nachher-Reproduktionsprüfungen und keine
breite Smoke-Abdeckung darstellen. Der Mantis-Workflow für Thread-Anhänge kann außerdem ein
Video eines angemeldeten Discord-Web-Zeugen hinzufügen, wenn
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_DIR` oder
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_TGZ_B64` in der QA-
Umgebung konfiguriert ist. Dieses Betrachterprofil dient ausschließlich der visuellen Aufzeichnung; die Entscheidung
über Erfolg oder Fehlschlag erfolgt weiterhin über das Discord-REST-Orakel.

Für die übrigen transportechten Smoke-Lanes:

```bash
pnpm openclaw qa discord
pnpm openclaw qa slack
pnpm openclaw qa telegram
pnpm openclaw qa whatsapp
```

Sie zielen auf einen bereits vorhandenen echten Kanal mit zwei Bots oder Konten (Treiber +
SUT). Erforderliche Umgebungsvariablen, Szenariolisten, Ausgabeartefakte und der Convex-
Zugangsdatenpool für diese vier Transporte sind in der
[QA-Referenz für Discord, Slack, Telegram und WhatsApp](#discord-slack-telegram-and-whatsapp-qa-reference)
weiter unten dokumentiert.

### Mantis-Runner für Slack Desktop und visuelle Aufgaben

Führen Sie für einen vollständigen Lauf einer Slack-Desktop-VM mit VNC-Wiederherstellung Folgendes aus:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

Dieser Befehl reserviert eine Crabbox-Desktop-/Browser-Maschine, führt den Slack-Live-
Lauf in der VM aus, öffnet Slack Web im VNC-Browser, zeichnet den Desktop auf
und kopiert `slack-qa/`, `slack-desktop-smoke.png` und
`slack-desktop-smoke.mp4` (wenn Videoaufzeichnung verfügbar ist) zurück in das
Mantis-Artefaktverzeichnis. Crabbox-Desktop-/Browser-Reservierungen stellen die
Aufzeichnungswerkzeuge und Hilfspakete für Browser/native Builds vorab bereit, sodass das Szenario
nur bei älteren Reservierungen Fallbacks installieren sollte. Mantis meldet Gesamt- und
Phasenzeiten in `mantis-slack-desktop-smoke-report.md`, damit bei langsamen Läufen erkennbar ist,
ob Zeit für das Aufwärmen der Reservierung, das Abrufen von Anmeldedaten, die Remote-Einrichtung oder
das Kopieren der Artefakte benötigt wurde. Verwenden Sie `--lease-id <cbx_...>` erneut, nachdem Sie sich
manuell über VNC bei Slack Web angemeldet haben; wiederverwendete Reservierungen halten außerdem
den pnpm-Store-Cache von Crabbox vorgewärmt. Der Standardwert `--hydrate-mode source` verifiziert aus einem
Source-Checkout und führt Installation/Build innerhalb der VM aus. Verwenden Sie `--hydrate-mode prehydrated` nur,
wenn der wiederverwendete Remote-Arbeitsbereich bereits über `node_modules` und ein gebautes `dist/`
verfügt; dieser Modus überspringt den aufwendigen Installations-/Build-Schritt und schlägt sicher fehl, wenn der
Arbeitsbereich nicht bereit ist. Mit `--gateway-setup` lässt Mantis ein persistentes
OpenClaw-Slack-Gateway innerhalb der VM auf Port `38973` laufen; ohne diese Option führt der
Befehl den normalen Bot-zu-Bot-Slack-QA-Lauf aus und wird nach der Artefakterfassung beendet.

Um die native Slack-Genehmigungsoberfläche mit Desktop-Nachweisen zu belegen, führen Sie den
Mantis-Genehmigungsprüfpunktmodus aus:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer
```

Dieser Modus schließt sich gegenseitig mit `--gateway-setup` aus. Er führt die Slack-
Genehmigungsszenarien aus, lehnt Szenario-IDs ab, die keine Genehmigung betreffen, wartet bei jedem
ausstehenden und abgeschlossenen Genehmigungsstatus, rendert die beobachtete Slack-API-Nachricht in
`approval-checkpoints/<scenario>-pending.png` und
`approval-checkpoints/<scenario>-resolved.png` und schlägt anschließend fehl, wenn ein Prüfpunkt,
Nachrichtenbeleg, eine Bestätigung oder ein gerenderter Screenshot fehlt oder
leer ist. Kalte CI-Reservierungen zeigen in
`slack-desktop-smoke.png` möglicherweise weiterhin die Slack-Anmeldung; die Bilder der Genehmigungsprüfpunkte sind der visuelle
Nachweis für diesen Lauf.

Der standardmäßige Prüfpunktlauf behält die beiden üblichen Slack-Genehmigungsszenarien bei.
Um eine der optionalen Codex-Genehmigungsrouten zu erfassen, wählen Sie sie ausdrücklich mit
`--scenario slack-codex-approval-exec-native` oder
`--scenario slack-codex-approval-plugin-native` aus; Mantis akzeptiert beide und erzeugt
dasselbe Screenshot-Paar für den ausstehenden/abgeschlossenen Status. Der Runner erweitert seine Fristen für Prüfpunkte
und Remote-Befehle für jede ausgewählte Codex-Route, damit die vollständige
Sequenz aus Genehmigung, Agent-Abschluss und Aktualisierung auf den abgeschlossenen Status beendet werden kann.

Die Checkliste für Operatoren, der GitHub-Workflow-Dispatch-Befehl, der Vertrag für Nachweiskommentare,
die Entscheidungstabelle für den Hydrate-Modus, die Interpretation der Zeitmessung und die Schritte zur
Fehlerbehandlung finden Sie im
[Mantis-Runbook für Slack Desktop](/de/concepts/mantis-slack-desktop-runbook).

Führen Sie für eine Desktop-Aufgabe im Agent-/CV-Stil Folgendes aus:

```bash
pnpm openclaw qa mantis visual-task \
  --browser-url https://example.net \
  --expect-text "Example Domain" \
  --vision-model openai/gpt-5.6-luna
```

`visual-task` reserviert eine Crabbox-Desktop-/Browser-Maschine oder verwendet sie erneut, startet
`crabbox record --while`, steuert den sichtbaren Browser über eine verschachtelte
`visual-driver`, erfasst `visual-task.png`, führt `openclaw infer image
describe` gegen den Screenshot aus, wenn `--vision-mode image-describe`
ausgewählt ist, und schreibt `visual-task.mp4`, `mantis-visual-task-summary.json`,
`mantis-visual-task-driver-result.json` und
`mantis-visual-task-report.md`. Wenn `--expect-text` gesetzt ist, fordert der Vision-
Prompt ein strukturiertes JSON-Urteil (`visible`, `evidence`, `reason`)
an und besteht nur, wenn das Modell `visible: true` mit Nachweisen meldet, die
den erwarteten Text anführen; eine `visible: false`-Antwort, die lediglich den
Zieltext zitiert, lässt die Assertion weiterhin fehlschlagen. Verwenden Sie `--vision-mode metadata` für einen
Smoke-Test ohne Modell, der Desktop, Browser, Screenshot und Video-
Infrastruktur belegt, ohne einen Provider für Bildverständnis aufzurufen. Die Aufzeichnung ist ein
erforderliches Artefakt für `visual-task`; wenn Crabbox kein nicht leeres
`visual-task.mp4` aufzeichnet, schlägt die Aufgabe selbst dann fehl, wenn der visuelle Treiber erfolgreich war. Bei
einem Fehler behält Mantis die Reservierung für VNC bei, sofern die Aufgabe nicht bereits erfolgreich war
und `--keep-lease` nicht gesetzt wurde.

### Zustandsprüfung des Anmeldedaten-Pools

Führen Sie vor der Verwendung gebündelter Live-Anmeldedaten Folgendes aus:

```bash
pnpm openclaw qa credentials doctor
```

Der Doctor prüft die Convex-Broker-Umgebung (`OPENCLAW_QA_CONVEX_SITE_URL`,
`OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX`), validiert Endpunkteinstellungen, meldet
für `OPENCLAW_QA_CONVEX_SECRET_CI` und
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` nur den Status gesetzt/fehlend und überprüft die Erreichbarkeit
der Administration/Auflistung, wenn das Maintainer-Geheimnis vorhanden ist.

## Kanonische Szenarioabdeckung

Die Stammdatei `taxonomy.yaml` definiert semantische Abdeckungs-IDs. Szenario-YAML-Dateien
unter `qa/scenarios/` ordnen jedes Szenario diesen IDs zu und verwalten die
Ausführungsmetadaten: `channel` ist die einzige Kanalanforderung, und `profiles` deklarieren
die benannte Laufzugehörigkeit. Der Kanaltreiber ist eine austauschbare Implementierungsentscheidung
auf Laufebene. TypeScript-
Runner fragen diesen Katalog ab; sie verwalten keine parallelen Szenario- oder Abdeckungsinventare.

Die statische Ausgabe von `qa coverage` meldet die Zuordnung von Taxonomie zu Szenario. Der tatsächliche
Nachweis stammt aus `qa-evidence.json`, das das ausgeführte Szenario,
die Abdeckungs-IDs, den Kanal, den tatsächlich verwendeten Treiber und das Ergebnis aufzeichnet. Kanal und Treiber sind
Berichtsdimensionen, keine zusätzlichen Vokabulare für Abdeckungs-IDs oder Achsen für die
Szenariozulässigkeit.

Führen Sie für einen Lauf in einer kurzlebigen Linux-VM, ohne Docker in den QA-Pfad einzubinden, Folgendes aus:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

Dadurch wird ein frischer Multipass-Gast gestartet, Abhängigkeiten werden installiert, OpenClaw wird
innerhalb des Gasts gebaut, `qa suite` wird ausgeführt und anschließend werden der normale QA-Bericht und
die Zusammenfassung zurück in `.artifacts/qa-e2e/...` auf dem Host kopiert. Dabei wird dasselbe
Verhalten zur Szenarioauswahl wie bei `qa suite` auf dem Host wiederverwendet.

Host- und Multipass-Suite-Läufe führen standardmäßig mehrere ausgewählte Szenarien
parallel mit isolierten Gateway-Workern aus. `qa-channel` verwendet standardmäßig
Parallelität 4, begrenzt durch die Anzahl ausgewählter Szenarien. Verwenden Sie `--concurrency
<count>`, um die Worker-Anzahl anzupassen, oder `--concurrency 1` für die serielle Ausführung.
Verwenden Sie `--pack personal-agent`, um das Benchmark-Paket für persönliche Assistenten (10
Szenarien) auszuführen. Der Paketselektor wird additiv mit wiederholten `--scenario`-Flags verwendet:
explizite Szenarien werden zuerst ausgeführt, anschließend werden die Paketszenarien in Paketreihenfolge
ausgeführt, wobei Duplikate entfernt werden. Verwenden Sie `--pack observability`, um die
Szenarien `otel-trace-smoke` und `docker-prometheus-smoke` gemeinsam auszuwählen, wenn ein
benutzerdefinierter QA-Runner bereits die Einrichtung des OpenTelemetry-Collectors bereitstellt.

Der Befehl wird mit einem Exit-Code ungleich null beendet, wenn ein Szenario fehlschlägt. Verwenden Sie `--allow-failures`,
wenn Sie Artefakte ohne fehlschlagenden Exit-Code wünschen.

Live-Läufe leiten die unterstützten QA-Authentifizierungseingaben weiter, die für den
Gast praktikabel sind: umgebungsbasierte Provider-Schlüssel, den Pfad zur QA-Live-Provider-Konfiguration und
`CODEX_HOME`, sofern vorhanden. Bewahren Sie `--output-dir` unterhalb des Repository-Stammverzeichnisses auf, damit der
Gast über den eingebundenen Arbeitsbereich zurückschreiben kann.

## QA-Referenz für Discord, Slack, Telegram und WhatsApp

Der Matrix-Adapter verwendet den oben dokumentierten kurzlebigen, Docker-gestützten Lauf.
Discord, Slack, Telegram und WhatsApp arbeiten mit bereits vorhandenen realen
Transporten, daher befindet sich ihre Referenz hier.

### Gemeinsame CLI-Flags

Diese Läufe werden über
`extensions/qa-lab/src/live-transports/shared/live-transport-cli.ts` registriert und
akzeptieren dieselben Flags:

| Flag                                  | Standardwert                                      | Beschreibung                                                                                                                                       |
| ------------------------------------- | -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--scenario <id>`                     | -                                                  | Nur dieses Szenario ausführen. Wiederholbar.                                                                                                       |
| `--output-dir <path>`                 | `<repo>/.artifacts/qa-e2e/<transport>-<timestamp>` | Ort, an den Berichte, Zusammenfassungen, Nachweise, transportspezifische Artefakte und das Ausgabelog geschrieben werden. Relative Pfade werden relativ zu `--repo-root` aufgelöst. |
| `--repo-root <path>`                  | `process.cwd()`                                    | Repository-Stammverzeichnis beim Aufruf aus einem neutralen aktuellen Arbeitsverzeichnis.                                                          |
| `--sut-account <id>`                  | `sut`                                              | Temporäre Konto-ID innerhalb der QA-Gateway-Konfiguration.                                                                                         |
| `--provider-mode <mode>`              | `live-frontier`                                    | `mock-openai`, `aimock` oder `live-frontier`.                                                                                    |
| `--model <ref>` / `--alt-model <ref>` | Provider-Standardwert                              | Primäre/alternative Modellreferenzen.                                                                                                              |
| `--fast`                              | aus                                                | Schneller Provider-Modus, sofern unterstützt.                                                                                                      |
| `--credential-source <env\|convex>`   | `env`                                              | Siehe [Convex-Anmeldedaten-Pool](#convex-credential-pool).                                                                                         |
| `--credential-role <maintainer\|ci>`  | `ci` in CI, andernfalls `maintainer`                 | Verwendete Rolle, wenn `--credential-source convex`.                                                                                                         |
| `--allow-failures`                    | aus                                                | Artefakte schreiben, ohne bei fehlgeschlagenen Szenarien einen fehlschlagenden Exit-Code zurückzugeben.                                            |

Jeder Lauf wird bei einem fehlgeschlagenen Szenario mit einem Exit-Code ungleich null beendet. `--allow-failures` schreibt
Artefakte, ohne einen fehlschlagenden Exit-Code festzulegen. Telegram akzeptiert außerdem
`--list-scenarios`, um verfügbare Szenario-IDs auszugeben und sich zu beenden; die anderen Läufe
stellen dieses Flag nicht bereit.

### Telegram-QA

```bash
pnpm openclaw qa telegram
```

Zielt auf eine echte private Telegram-Gruppe mit zwei unterschiedlichen Bots (Treiber +
SUT). Der SUT-Bot muss einen Telegram-Benutzernamen haben; die Bot-zu-Bot-Beobachtung funktioniert
am besten, wenn für beide Bots **Bot-to-Bot Communication Mode** in
`@BotFather` aktiviert ist.

Erforderliche Umgebungsvariablen bei `--credential-source env`:

- `OPENCLAW_QA_TELEGRAM_GROUP_ID` – numerische Chat-ID (Zeichenfolge).
- `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`

Das Profil `release` wählt die gepflegten Telegram-YAML-Szenarien aus; `all`
fügt optionale Belastungsprüfungen für Sitzungen, Nutzung, Antwortketten und Streaming hinzu. Explizite
`--scenario`-Werte überschreiben das Profil.

- `channel-canary`
- `channel-mention-gating`
- `telegram-help-command`
- `telegram-commands-command`
- `telegram-tools-compact-command`
- `telegram-whoami-command`
- `telegram-status-command`
- `telegram-repeated-command-authorization`
- `telegram-other-bot-command-gating`
- `telegram-context-command`
- `telegram-current-session-status-tool`
- `telegram-tool-only-usage-footer`
- `telegram-reply-chain-exact-marker`
- `telegram-stream-final-single-message`
- `telegram-long-final-reuses-preview`
- `telegram-long-final-three-chunks`

Das Profil `release` deckt immer Canary, Mention-Gating, Antworten auf native Befehle, Befehlsadressierung und Bot-zu-Bot-Gruppenantworten ab. `mock-openai`
umfasst außerdem die deterministische Prüfung der Vorschau langer finaler Antworten.
`telegram-current-session-status-tool` und
`telegram-tool-only-usage-footer` bleiben optional: Ersteres ist nur stabil,
wenn es direkt nach Canary ausgeführt wird, und Letzteres ist ein Nachweis mit echtem Telegram
für den `/usage`-Footer bei Antworten, die ausschließlich aus Tool-Aufrufen bestehen. Verwenden Sie `pnpm openclaw qa telegram
--list-scenarios --provider-mode mock-openai`, um die aktuelle
Aufteilung in Standard- und optionale Prüfungen mit Regressionsreferenzen auszugeben. Verwenden Sie `--profile all` für jedes
Live-Adapter-Szenario von Telegram.

Ausgabeartefakte:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` – Nachweiseinträge für die Prüfungen des Live-Transports,
  einschließlich der Felder für Profil, Abdeckung, Provider, Kanal, Artefakte, Ergebnis und RTT.

Telegram-Paketläufe verwenden denselben Vertrag für Telegram-Anmeldedaten. Wiederholte RTT-
Messungen sind Teil der normalen Live-Pipeline für das Telegram-Paket; die RTT-
Verteilung wird für die ausgewählte RTT-Prüfung unter `result.timing` in `qa-evidence.json`
übernommen.

```bash
OPENCLAW_QA_CREDENTIAL_SOURCE=convex \
pnpm test:docker:npm-telegram-live
```

Wenn `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` festgelegt ist, least der Live-Wrapper des Pakets
einen `kind: "telegram"`-Anmeldedatensatz, exportiert die geleasten Umgebungsvariablen für Gruppe, Treiber und SUT-
Bot in den Lauf des installierten Pakets, sendet Heartbeats für das Lease und gibt es
beim Herunterfahren frei. Der Paket-Wrapper verwendet standardmäßig 20 RTT-Prüfungen von
`channel-canary`, ein RTT-Zeitlimit von 30s und außerhalb der CI die Convex-Rolle
`maintainer`, wenn Convex ausgewählt ist. Überschreiben Sie
`OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`, `OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS`
oder `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES`, um die RTT-Messung anzupassen, ohne
einen separaten RTT-Befehl oder ein Telegram-spezifisches Zusammenfassungsformat zu erstellen.

### Discord-QA

```bash
pnpm openclaw qa discord
```

Zielt auf einen echten privaten Discord-Guild-Kanal mit zwei Bots: einen vom
Harness gesteuerten Treiber-Bot und einen SUT-Bot, der vom untergeordneten OpenClaw-Gateway
über das gebündelte Discord-Plugin gestartet wird. Überprüft die Verarbeitung von Kanal-Mentions, ob
der SUT-Bot den nativen Befehl `/help` bei Discord registriert hat, sowie
optionale Mantis-Nachweisszenarien.

Erforderliche Umgebungsvariablen bei `--credential-source env`:

- `OPENCLAW_QA_DISCORD_GUILD_ID`
- `OPENCLAW_QA_DISCORD_CHANNEL_ID`
- `OPENCLAW_QA_DISCORD_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_SUT_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_SUT_APPLICATION_ID` – muss mit der von Discord zurückgegebenen Benutzer-ID des SUT-Bots
  übereinstimmen (andernfalls schlägt die Pipeline sofort fehl).

Optional:

- `OPENCLAW_QA_DISCORD_VOICE_CHANNEL_ID` wählt den Sprach-/Bühnenkanal für
  `discord-voice-autojoin` aus; ohne diese Angabe wählt das Szenario den ersten für
  den SUT-Bot sichtbaren Sprach-/Bühnenkanal aus.

Discord-YAML-Modulszenarien (`qa/scenarios/channels/discord-*.yaml`):

- `discord-canary`
- `discord-mention-gating`
- `discord-native-help-command-registration`
- `discord-voice-autojoin` – optionales Sprachszenario. Wird eigenständig ausgeführt, aktiviert
  `channels.discord.voice.autoJoin` und überprüft, ob der aktuelle
  Discord-Sprachstatus des SUT-Bots dem Ziel-Sprach-/Bühnenkanal entspricht. Convex-Anmeldedaten für Discord
  können optional `voiceChannelId` enthalten; andernfalls ermittelt der Runner-
  Adapter den ersten für den SUT-Bot sichtbaren Sprach-/Bühnenkanal in der Guild.
- `discord-status-reactions-tool-only` – optionales Mantis-Szenario. Wird eigenständig
  ausgeführt, da es den SUT mit `messages.statusReactions.enabled=true` auf dauerhaft aktive Guild-Antworten umstellt,
  die ausschließlich aus Tool-Aufrufen bestehen, und anschließend eine REST-
  Reaktionszeitleiste sowie visuelle HTML-/PNG-Artefakte erfasst. Die Vorher-/Nachher-
  Berichte von Mantis bewahren außerdem vom Szenario bereitgestellte MP4-Artefakte als `baseline.mp4`
  und `candidate.mp4` auf.
- `discord-thread-reply-filepath-attachment` – optionales Mantis-Szenario; siehe
  [Discord-Mantis-Szenarien](#discord-mantis-scenarios).

Führen Sie das Szenario für den automatischen Beitritt zu einem Discord-Sprachkanal explizit aus:

```bash
pnpm openclaw qa discord \
  --scenario discord-voice-autojoin \
  --provider-mode mock-openai
```

Führen Sie das Mantis-Szenario für Statusreaktionen explizit aus:

```bash
pnpm openclaw qa discord \
  --scenario discord-status-reactions-tool-only \
  --provider-mode live-frontier \
  --model openai/gpt-5.6-luna \
  --alt-model openai/gpt-5.6-luna \
  --fast
```

Ausgabeartefakte:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` – Nachweiseinträge für die Prüfungen des Live-Transports.
- `discord-qa-reaction-timelines.json` und
  `discord-status-reactions-tool-only-timeline.png`, wenn das Statusreaktionsszenario
  ausgeführt wird.

### Slack-QA

```bash
pnpm openclaw qa slack
```

Zielt auf einen echten privaten Slack-Kanal mit zwei unterschiedlichen Bots: einen vom
Harness gesteuerten Treiber-Bot und einen SUT-Bot, der vom untergeordneten OpenClaw-Gateway
über das gebündelte Slack-Plugin gestartet wird.

Erforderliche Umgebungsvariablen bei `--credential-source env`:

- `OPENCLAW_QA_SLACK_CHANNEL_ID`
- `OPENCLAW_QA_SLACK_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_APP_TOKEN`

Optional:

- `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_DIR` aktiviert visuelle Freigabe-
  Prüfpunkte für Mantis. Der Adapter schreibt `<scenario>.pending.json` und
  `<scenario>.resolved.json` und wartet anschließend auf passende `.ack.json`-Dateien.
- `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_TIMEOUT_MS` überschreibt das Zeitlimit für die Bestätigung
  des Prüfpunkts. Der Standardwert ist `120000`.

Kanonische YAML-Szenarien, die über den Slack-Live-Adapter verfügbar sind:

- `thread-follow-up`
- `thread-isolation`

Slack-YAML-Modulszenarien (`qa/scenarios/channels/slack-*.yaml`):

- `slack-canary`
- `slack-mention-gating`
- `slack-allowlist-block`
- `slack-channel-disabled-warning` – optionale Prüfung mit echtem Slack, die bestätigt, dass ein
  konfigurierter deaktivierter Kanal eine strukturierte Warnung ausgibt, ohne zu antworten.
- `slack-top-level-reply-shape`
- `slack-restart-resume`
- `slack-progress-commentary-true`, `slack-progress-commentary-false`,
  `slack-progress-commentary-omitted` und
  `slack-progress-commentary-verbose-dedupe` – optionale Prüfungen mit echtem Slack für
  unabhängige Steuerungen von Kommentaren und Tool-Fortschritt, den veralteten
  Standardwert bei ausgelassenem Schlüssel und die einmalige Zustellung, wenn der dauerhafte ausführliche Fortschritt aktiviert ist.
- `slack-reaction-glyph-native` – optionales Live-Szenario für Reaktionen des Nachrichten-Tools.
  Weist den Agenten an, exakt das Symbol `✅` zu übergeben, und bestätigt, dass Slack
  `white_check_mark` für den SUT-Bot in der Zielnachricht gespeichert hat.
- `slack-chart-presentation-native` – optionales portables Diagrammszenario, das
  den nativen Block `data_visualization` und den exakten barrierefreien Text überprüft.
- `slack-table-presentation-native` – optionales portables Tabellenszenario, das
  den nativen Block `data_table`, die exakten Zeilen und den barrierefreien Text überprüft.
- `slack-table-invalid-blocks-fallback` – optionales Direkttransportszenario,
  das eine strukturell lesbare rohe Tabelle oberhalb des Limits mit 101 Datenzeilen
  zuzüglich Kopfzeile über den
  produktiven Slack-Sendepfad sendet, nachweist, dass Slack selbst `invalid_blocks` zurückgibt,
  und überprüft, dass der gespeicherte Fallback mit deaktivierter Formatierung vollständig ist und keinen
  nativen Datenblock enthält. Die Szenariodetails enthalten ausschließlich sichere Nachweise zu Fehlercode, Anzahl und
  booleschen Werten.
- `slack-approval-exec-native` – optionales natives Slack-Szenario für Exec-Freigaben.
  Fordert über das Gateway eine Exec-Freigabe an, überprüft, ob die Slack-Nachricht
  native Freigabeschaltflächen enthält, löst sie auf und überprüft die aufgelöste Slack-
  Aktualisierung.
- `slack-approval-plugin-native` – optionales natives Slack-Szenario für Plugin-Freigaben.
  Aktiviert die Weiterleitung von Exec- und Plugin-Freigaben gemeinsam, damit Plugin-
  Ereignisse nicht durch das Routing der Exec-Freigabe unterdrückt werden, und überprüft anschließend denselben
  ausstehenden/aufgelösten nativen Slack-UI-Pfad.
- `slack-codex-approval-exec-native` – optionales Codex-Guardian-Szenario für Befehlsfreigaben.
  Aktiviert das Codex-Plugin im Guardian-Modus, leitet einen von Slack stammenden
  Gateway-Agentendurchlauf über das Codex-App-Server-Harness,
  wartet auf die native Slack-Plugin-Freigabeaufforderung für
  `openclaw-codex-app-server`, löst sie auf und überprüft, ob der Codex-Durchlauf
  mit den erwarteten Markierungen für Befehlsausgabe und Assistent abgeschlossen wird.
- `slack-codex-approval-plugin-native` – optionales Codex-Guardian-Szenario für Dateifreigaben.
  Verwendet eine `apply_patch`-Anweisung außerhalb des Arbeitsbereichs, damit Codex
  die App-Server-Route für die Freigabe von Dateiänderungen ausgibt, und überprüft anschließend denselben nativen
  ausstehenden/aufgelösten Slack-Freigabepfad, die finale Assistentenmarkierung und den exakten Dateiinhalt
  vor der Bereinigung.

Die Codex-Freigabeszenarien erfordern ein `openai/*` oder `codex/*` `--model`, die
normalen Anmeldedaten für das Live-Modell sowie eine vom Codex-Plugin akzeptierte Codex-Authentifizierung oder API-Schlüssel-Authentifizierung.
Die Szenariodetails enthalten neben den redigierten Slack-Freigabemetadaten
die Codex-App-Server-Methode, den ausgewählten Codex-Modellschlüssel,
den finalen Status des Codex-Durchlaufs und die Überprüfung der Operationsmarkierung.

Ausgabeartefakte:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` – Nachweiseinträge für die Prüfungen des Live-Transports.
- `approval-checkpoints/` – nur wenn Mantis
  `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_DIR` festlegt; enthält Prüfpunkt-JSON,
  Bestätigungs-JSON und Screenshots des ausstehenden und aufgelösten Zustands.

#### Slack-Workspace einrichten

Die Pipeline benötigt zwei unterschiedliche Slack-Apps in einem Workspace sowie einen Kanal, dessen Mitglied beide
Bots sind:

- `channelId` – die `Cxxxxxxxxxx`-ID eines Kanals, in den beide Bots
  eingeladen wurden. Verwenden Sie einen dedizierten Kanal; die Pipeline veröffentlicht bei jedem Lauf Beiträge.
- `driverBotToken` – Bot-Token (`xoxb-...`) der **Treiber**-App.
- `sutBotToken` – Bot-Token (`xoxb-...`) der **SUT**-App, die eine
  von der Treiber-App getrennte Slack-App sein muss, damit ihre Bot-Benutzer-ID eindeutig ist.
- `sutAppToken` – Token auf App-Ebene (`xapp-...`) der SUT-App mit
  `connections:write`, das vom Socket Mode verwendet wird, damit die SUT-App Ereignisse empfangen kann.

Verwenden Sie vorzugsweise einen dedizierten Slack-Workspace für die QA, anstatt einen produktiven
Workspace wiederzuverwenden.

Das nachstehende SUT-Manifest beschränkt die produktive Installation des gebündelten Slack-Plugins
(`extensions/slack/src/setup-shared.ts:12`) absichtlich auf die
Berechtigungen und Ereignisse, die von der Live-Slack-QA-Suite abgedeckt werden. Informationen zur
Einrichtung des produktiven Kanals aus Benutzersicht finden Sie unter
[Schnelleinrichtung des Slack-Kanals](/de/channels/slack#quick-setup); das QA-Treiber-/SUT-
Paar ist absichtlich getrennt, da die Pipeline zwei unterschiedliche Bot-Benutzer-
IDs in einem Workspace benötigt.

**1. Treiber-App erstellen**

Rufen Sie [api.slack.com/apps](https://api.slack.com/apps) → _Create New App_ →
_From a manifest_ auf, wählen Sie den QA-Workspace aus, fügen Sie das folgende Manifest ein
und wählen Sie anschließend _Install to Workspace_:

```json
{
  "display_information": {
    "name": "OpenClaw QA Driver",
    "description": "Testtreiber-Bot für die OpenClaw-QA-Live-Pipeline von Slack"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw QA Driver",
      "always_online": true
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": ["chat:write", "channels:history", "groups:history", "users:read"]
    }
  },
  "settings": {
    "socket_mode_enabled": false
  }
}
```

Kopieren Sie das _Bot User OAuth Token_ (`xoxb-...`) – daraus wird
`driverBotToken`. Der Treiber muss lediglich Nachrichten veröffentlichen und sich selbst
identifizieren; keine Ereignisse, kein Socket Mode.

**2. SUT-App erstellen**

Wiederholen Sie _Create New App → From a manifest_ im selben Workspace. Diese QA-App
verwendet absichtlich eine eingeschränktere Version des produktiven Manifests des gebündelten Slack-Plugins
(`extensions/slack/src/setup-shared.ts:12`): Reaktions-
Scopes und -Ereignisse sind ausgelassen, da die Live-Slack-QA-Suite die
Reaktionsverarbeitung noch nicht abdeckt.

```json
{
  "display_information": {
    "name": "OpenClaw QA SUT",
    "description": "OpenClaw-QA-SUT-Connector für OpenClaw"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw QA SUT",
      "always_online": true
    },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed"
      ]
    }
  }
}
```

Nachdem Slack die App erstellt hat, führen Sie auf deren Einstellungsseite zwei Schritte aus:

- _Install to Workspace_ → kopieren Sie den _Bot User OAuth Token_ → dieser wird zu
  `sutBotToken`.
- _Basic Information → App-Level Tokens → Generate Token and Scopes_ → fügen Sie den
  Scope `connections:write` hinzu → speichern Sie → kopieren Sie den Wert `xapp-...` → dieser
  wird zu `sutAppToken`.

Überprüfen Sie, dass die beiden Bots unterschiedliche Benutzer-IDs besitzen, indem Sie `auth.test` mit jedem
Token aufrufen. Die Runtime unterscheidet Treiber und SUT anhand der Benutzer-ID; die Wiederverwendung einer App
für beide führt dazu, dass das Mention-Gating sofort fehlschlägt.

**3. Kanal erstellen**

Erstellen Sie im QA-Workspace einen Kanal (z. B. `#openclaw-qa`) und laden Sie beide
Bots aus dem Kanal heraus ein:

```text
/invite @OpenClaw QA Driver
/invite @OpenClaw QA SUT
```

Kopieren Sie die ID `Cxxxxxxxxxx` aus _channel info → About → Channel ID_ – diese
wird zu `channelId`. Ein öffentlicher Kanal funktioniert; wenn Sie einen privaten Kanal verwenden,
verfügen beide Apps bereits über `groups:history`, sodass die Verlaufsabfragen des Test-Harness
weiterhin erfolgreich sind.

**4. Anmeldedaten registrieren**

Es gibt zwei Optionen. Verwenden Sie Umgebungsvariablen für das Debugging auf einem einzelnen Rechner (setzen Sie die vier
`OPENCLAW_QA_SLACK_*`-Variablen und übergeben Sie `--credential-source env`), oder befüllen Sie
den gemeinsamen Convex-Pool, damit CI und andere Maintainer sie leasen können.

Schreiben Sie für den Convex-Pool die vier Felder in eine JSON-Datei:

```json
{
  "channelId": "Cxxxxxxxxxx",
  "driverBotToken": "xoxb-...",
  "sutBotToken": "xoxb-...",
  "sutAppToken": "xapp-..."
}
```

Wenn `OPENCLAW_QA_CONVEX_SITE_URL` und `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`
in Ihrer Shell exportiert sind, registrieren und überprüfen Sie die Anmeldedaten:

```bash
pnpm openclaw qa credentials add \
  --kind slack \
  --payload-file slack-creds.json \
  --note "QA Slack pool seed"

pnpm openclaw qa credentials list --kind slack --status all --json
```

Erwartet werden `count: 1`, `status: "active"` und kein Feld `lease`.

**5. Ende-zu-Ende-Verhalten überprüfen**

Führen Sie die Lane lokal aus, um zu bestätigen, dass beide Bots über den
Broker miteinander kommunizieren können:

```bash
pnpm openclaw qa slack \
  --credential-source convex \
  --credential-role maintainer \
  --output-dir .artifacts/qa-e2e/slack-local
```

Ein erfolgreicher Lauf ist deutlich unter 30 Sekunden abgeschlossen, und `qa-suite-report.md`
zeigt sowohl `slack-canary` als auch `slack-mention-gating` mit dem Status `pass`. Wenn die
Lane etwa 90 Sekunden lang hängt und mit `Convex credential pool exhausted
for kind "slack"` beendet wird, ist entweder der Pool leer oder jede Zeile ist geleast – `qa
credentials list --kind slack --status all --json` zeigt Ihnen, welcher Fall vorliegt.

### WhatsApp-QA

```bash
pnpm openclaw qa whatsapp
```

Zielt auf zwei dedizierte WhatsApp-Web-Konten: ein vom Test-Harness gesteuertes
Treiberkonto und ein SUT-Konto, das vom untergeordneten OpenClaw-Gateway über
das gebündelte WhatsApp-Plugin gestartet wird.

Erforderliche Umgebungsvariablen bei `--credential-source env`:

- `OPENCLAW_QA_WHATSAPP_DRIVER_PHONE_E164`
- `OPENCLAW_QA_WHATSAPP_SUT_PHONE_E164`
- `OPENCLAW_QA_WHATSAPP_DRIVER_AUTH_ARCHIVE_BASE64`
- `OPENCLAW_QA_WHATSAPP_SUT_AUTH_ARCHIVE_BASE64`

Optional:

- `OPENCLAW_QA_WHATSAPP_GROUP_JID` aktiviert Gruppenszenarien wie
  `whatsapp-mention-gating`, `whatsapp-group-pending-history-context`,
  `whatsapp-broadcast-group-fanout`, `whatsapp-group-activation-always`,
  `whatsapp-group-reply-to-bot-triggers`, Gruppenaktions-, Medien- und Umfrageszenarien
  sowie `whatsapp-group-allowlist-block`.

WhatsApp-YAML-Szenarien (`qa/scenarios/channels/whatsapp-*.yaml`):

- Grundfunktion und Gruppen-Gating: `whatsapp-canary`, `whatsapp-pairing-block`,
  `whatsapp-mention-gating`, `whatsapp-group-pending-history-context`,
  `whatsapp-group-activation-always`, `whatsapp-group-reply-to-bot-triggers`,
  `whatsapp-top-level-reply-shape`, `whatsapp-restart-resume`,
  `whatsapp-group-allowlist-block`.
- Native Befehle: `whatsapp-help-command`, `whatsapp-status-command`,
  `whatsapp-commands-command`, `whatsapp-tools-compact-command`,
  `whatsapp-whoami-command`, `whatsapp-context-command`,
  `whatsapp-native-new-command`.
- Antwort- und Endausgabeverhalten: `whatsapp-tool-only-usage-footer`,
  `whatsapp-reply-to-message`, `whatsapp-group-reply-to-message`,
  `whatsapp-reply-to-mode-batched`, `whatsapp-reply-context-isolation`,
  `whatsapp-reply-delivery-shape`, `whatsapp-stream-final-message-accounting`.
- Nachrichtenaktionen über den Benutzerpfad: `whatsapp-agent-message-action-react` beginnt
  mit einer echten Direktnachricht des Treibers, lässt das Modell das Tool `message` aufrufen und
  beobachtet die native WhatsApp-Reaktion. `whatsapp-agent-message-action-upload-file`
  verwendet dieselbe Vorgehensweise für `message(action=upload-file)` und beobachtet
  native WhatsApp-Medien. `whatsapp-group-agent-message-action-react` und
  `whatsapp-group-agent-message-action-upload-file` weisen dieselben
  benutzersichtbaren Aktionen in einer echten WhatsApp-Gruppe nach.
- Gruppen-Fan-out: `whatsapp-broadcast-group-fanout` beginnt mit einer erwähnenden
  WhatsApp-Gruppennachricht und überprüft unterschiedliche sichtbare Antworten von `main`
  und `qa-second`.
- Gruppenaktivierung: `whatsapp-group-activation-always` ändert eine echte
  Gruppensitzung in `/activation always`, weist nach, dass eine Gruppennachricht ohne Erwähnung
  den Agenten aktiviert, und stellt anschließend `/activation mention` wieder her.
  `whatsapp-group-reply-to-bot-triggers` legt eine Bot-Antwort an, sendet eine native
  zitierte Antwort darauf ohne ausdrückliche Erwähnung und überprüft, dass der Agent
  durch diesen Antwortkontext aktiviert wird.
- Eingehende Medien und strukturierte Nachrichten: `whatsapp-inbound-image-caption`,
  `whatsapp-audio-preflight`, `whatsapp-inbound-structured-messages`,
  `whatsapp-group-audio-gating`, `whatsapp-inbound-reaction-no-trigger`.
  Diese senden echte WhatsApp-Bild-, Audio-, Dokument-, Standort-, Kontakt-,
  Sticker- und Reaktionsereignisse über den Treiber.
- Direkte Gateway-Vertragsprüfungen: `whatsapp-outbound-media-matrix`,
  `whatsapp-outbound-document-preserves-filename`, `whatsapp-outbound-poll`,
  `whatsapp-outbound-send-serialization`,
  `whatsapp-group-outbound-media`, `whatsapp-group-outbound-poll`,
  `whatsapp-message-actions`, `whatsapp-reply-context-isolation`,
  `whatsapp-reply-delivery-shape`. Diese umgehen die Modellaufforderung absichtlich
  und weisen deterministische Gateway-/Kanalverträge für `send`, `poll` und
  `message.action` nach.
- Abdeckung der Zugriffssteuerung: `whatsapp-access-control-dm-open`,
  `whatsapp-access-control-dm-disabled`, `whatsapp-access-control-group-open`,
  `whatsapp-access-control-group-disabled`, `whatsapp-group-allowlist-block`.
- Native Genehmigungen: `whatsapp-approval-exec-deny-native`,
  `whatsapp-approval-exec-native`, `whatsapp-approval-exec-reaction-native`,
  `whatsapp-approval-exec-group-reaction-native`,
  `whatsapp-approval-plugin-native`.
- Statusreaktionen: `whatsapp-status-reactions`,
  `whatsapp-status-reaction-lifecycle`.

Der Katalog enthält derzeit 52 Szenarien. Die Standard-Lane `live-frontier`
wird für eine schnelle Smoke-Test-Abdeckung mit 8 Szenarien klein gehalten. Die Standard-Lane `mock-openai`
führt 39 Szenarien deterministisch über den echten WhatsApp-
Transport aus und simuliert dabei nur die Modellausgabe; Genehmigungsszenarien und einige
aufwendigere bzw. blockierende Prüfungen bleiben explizit über die Szenario-ID auswählbar.

Der WhatsApp-QA-Treiber beobachtet strukturierte Live-Ereignisse (`text`, `media`,
`location`, `reaction` und `poll`) und kann aktiv Medien, Umfragen,
Kontakte, Standorte und Sticker senden. QA Lab importiert diesen Treiber über die
Paketoberfläche `@openclaw/whatsapp/api.js`, statt auf private
WhatsApp-Runtime-Dateien zuzugreifen. Bei Gruppenbeobachtungen ist `fromJid` die Gruppen-JID,
während `participantJid` und `fromPhoneE164` den sendenden Teilnehmer identifizieren.
Nachrichteninhalte werden standardmäßig geschwärzt. Direkte Gateway-Prüfungen für Umfragen, Datei-Uploads,
Medien, Gruppenumfragen, Gruppenmedien und Antwortformen sind Transport-/API-
Vertragsprüfungen; sie gelten nicht als Nachweis dafür, dass eine Benutzereingabe den
Agenten dieselbe Aktion auswählen ließ. Der Nachweis von Aktionen über den Benutzerpfad stammt aus Szenarien
wie `whatsapp-agent-message-action-react` und
`whatsapp-group-agent-message-action-react`, bei denen der Treiber eine normale
WhatsApp-Nachricht sendet und QA Lab das daraus entstehende native WhatsApp-Artefakt beobachtet.
Die Details der WhatsApp-Szenarien enthalten die Vorgehensweise jedes Szenarios (`user-path`,
`direct-gateway` oder `native-approval`), damit Nachweise nicht mit einem
stärkeren Vertrag verwechselt werden können, als sie tatsächlich belegen.

Ausgabeartefakte:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` – Nachweiseinträge für die Live-Transportprüfungen.

### Convex-Anmeldedatenpool

Discord-, Slack-, Telegram- und WhatsApp-Lanes können Anmeldedaten aus einem
gemeinsamen Convex-Pool leasen, anstatt die oben genannten Umgebungsvariablen zu lesen. Übergeben Sie
`--credential-source convex` (oder setzen Sie `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`);
QA Lab erwirbt einen exklusiven Lease, sendet für die Dauer des
Laufs Heartbeats dafür und gibt ihn beim Herunterfahren frei. Die Pool-Arten sind `"discord"`, `"slack"`,
`"telegram"` und `"whatsapp"`.

Payload-Strukturen, die der Broker bei `admin/add` validiert:

- Discord (`kind: "discord"`): `{ guildId: string, channelId: string,
driverBotToken: string, sutBotToken: string, sutApplicationId: string }`.
- Telegram (`kind: "telegram"`): `{ groupId: string, driverToken: string,
sutToken: string }` – `groupId` muss eine numerische Chat-ID-Zeichenfolge sein.
- Echter Telegram-Benutzer (`kind: "telegram-user"`): `{ groupId: string, sutToken:
string, testerUserId: string, testerUsername: string, telegramApiId:
string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string,
tdlibArchiveBase64: string, tdlibArchiveSha256: string,
desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }` –
  ausschließlich für Mantis-Telegram-Desktop-Nachweise. Allgemeine QA-Lab-Lanes dürfen
  diese Art nicht erwerben.
- WhatsApp (`kind: "whatsapp"`): `{ driverPhoneE164: string, sutPhoneE164:
string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string,
groupJid?: string }` – Telefonnummern müssen unterschiedliche E.164-Zeichenfolgen sein.

Der Mantis-Telegram-Desktop-Nachweisworkflow hält einen exklusiven Convex-
Lease vom Typ `telegram-user` sowohl für den TDLib-CLI-Treiber als auch für den Telegram-Desktop-
Beobachter und gibt ihn nach der Veröffentlichung des Nachweises frei.

Wenn ein PR einen deterministischen visuellen Diff benötigt, kann Mantis dieselbe simulierte
Modellantwort auf `main` und auf dem PR-Head verwenden, während sich der Telegram-Formatierer oder
die Zustellungsschicht ändert. Die Aufnahmestandards sind auf PR-Kommentare abgestimmt: Standard-
Crabbox-Klasse, Desktop-Aufzeichnung mit 24 fps, Bewegungs-GIF mit 24 fps und 1920 px Vorschau-
breite. Vorher-/Nachher-Kommentare sollten ein sauberes Paket veröffentlichen, das
nur die vorgesehenen GIFs enthält.

Slack-Lanes können ebenfalls den Pool verwenden. Die Prüfungen der Slack-Payload-Struktur befinden sich derzeit
im Slack-QA-Runner statt im Broker; verwenden Sie `{ channelId: string,
driverBotToken: string, sutBotToken: string, sutAppToken: string }` mit einer
Slack-Kanal-ID wie `Cxxxxxxxxxx`. Siehe
[Slack-Workspace einrichten](#setting-up-the-slack-workspace) zur Bereitstellung von Apps
und Scopes.

Betriebliche Umgebungsvariablen und der Endpunktvertrag des Convex-Brokers sind unter
[Tests → Gemeinsame Telegram-Anmeldedaten über Convex](/de/help/testing#shared-telegram-credentials-via-convex-v1)
beschrieben (der Abschnittsname stammt aus der Zeit vor dem Mehrkanal-Pool; die Lease-Semantik gilt
für alle Arten gleichermaßen).

## Repository-gestützte Seed-Daten

Seed-Assets befinden sich in `qa/`:

- `qa/scenarios/index.yaml`
- `qa/scenarios/<theme>/*.yaml`

Sie befinden sich absichtlich in Git, damit der QA-Plan sowohl für Menschen als auch
für den Agenten sichtbar ist.

`qa-lab` bleibt ein generischer YAML-Szenario-Runner. Jede Szenario-YAML-Datei ist die
maßgebliche Quelle für einen Testlauf und sollte Folgendes definieren:

- `title` auf oberster Ebene
- `scenario`-Metadaten
- optionale Kategorie-, Funktions-, Lane- und Risikometadaten in `scenario`
- Dokumentations- und Codereferenzen in `scenario`
- optionale Plugin-Anforderungen in `scenario`
- optionaler Gateway-Konfigurations-Patch in `scenario`
- ausführbares `flow` auf oberster Ebene für Ablaufszenarien oder
  `scenario.execution.kind` / `scenario.execution.path` für Vitest- und
  Playwright-Szenarien

Die wiederverwendbare Runtime-Oberfläche, auf der `flow` basiert, bleibt generisch und
querschnittlich. YAML-Szenarien können beispielsweise transportseitige
Hilfsfunktionen mit browserseitigen Hilfsfunktionen kombinieren, die die eingebettete Control UI über
die Gateway-`browser.request`-Schnittstelle steuern, ohne einen Runner für einen Sonderfall hinzuzufügen.

Szenariodateien sollten nach Produktfunktion statt nach Ordnern des
Quellbaums gruppiert werden. Halten Sie Szenario-IDs stabil, wenn Dateien verschoben werden; verwenden Sie `docsRefs` und
`codeRefs` für die Nachverfolgbarkeit der Implementierung.

Die Basisliste sollte breit genug bleiben, um Folgendes abzudecken:

- Direktnachrichten und Kanalchats
- Thread-Verhalten
- Lebenszyklus von Nachrichtenaktionen
- Cron-Callbacks
- Speicherabruf
- Modellwechsel
- Übergabe an Subagenten
- Lesen von Repository und Dokumentation
- eine kleine Build-Aufgabe wie Lobster Invaders

## Provider-Mock-Lanes

`qa suite` verfügt über zwei lokale Provider-Mock-Lanes:

- `mock-openai` ist der szenariobewusste OpenClaw-Mock. Er bleibt die standardmäßige
  deterministische Mock-Lane für Repository-basierte QA- und Paritäts-Gates.
- `aimock` startet einen AIMock-basierten Provider-Server für experimentelle
  Protokoll-, Fixture-, Aufzeichnungs-/Wiedergabe- und Chaos-Abdeckung. Er ist additiv und
  ersetzt nicht den `mock-openai`-Szenario-Dispatcher.

Die Implementierung der Provider-Lanes befindet sich unter `extensions/qa-lab/src/providers/`.
Jeder Provider verwaltet seine Standardwerte, den Start des lokalen Servers, die Gateway-Modellkonfiguration,
die Anforderungen an die Bereitstellung von Authentifizierungsprofilen sowie die Live-/Mock-Funktionskennzeichnungen. Gemeinsamer Suite- und
Gateway-Code wird über die Provider-Registry geleitet, statt nach
Provider-Namen zu verzweigen.

## Transportadapter

`qa-lab` stellt eine generische Transportschnittstelle für YAML-QA-Szenarien bereit. `qa-channel` ist
der synthetische Standard. `crabline` startet lokale, Provider-ähnliche Server und
führt die normalen Kanal-Plugins von OpenClaw gegen sie aus. `live` ist für
echte Provider-Anmeldedaten und externe Kanäle reserviert.

Auf Architekturebene ist die Aufteilung wie folgt:

- `qa-lab` verwaltet die generische Szenarioausführung, Worker-Parallelität, das Schreiben
  von Artefakten und die Berichterstellung.
- Der Transportadapter verwaltet die Gateway-Konfiguration, Bereitschaft, Beobachtung
  ein- und ausgehender Ereignisse, Transportaktionen und den normalisierten Transportstatus.
- YAML-Szenariodateien unter `qa/scenarios/` definieren den Testlauf; `qa-lab`
  stellt die wiederverwendbare Runtime-Oberfläche für ihre Ausführung bereit.

### Einen Kanal hinzufügen

Das Hinzufügen eines Kanals zum YAML-QA-System erfordert die Kanalimplementierung
sowie ein Szenariopaket, das den Kanalvertrag abdeckt. Fügen Sie für die Smoke-CI-
Abdeckung den passenden lokalen Crabline-Provider-Server hinzu und stellen Sie ihn
über den `crabline`-Treiber bereit.

Fügen Sie keinen neuen QA-Befehl auf oberster Ebene hinzu, wenn der gemeinsame `qa-lab`-Host
den Ablauf verwalten kann.

`qa-lab` verwaltet die gemeinsamen Host-Mechanismen:

- den `openclaw qa`-Befehlsstamm
- Start und Beenden der Suite
- Worker-Parallelität
- Schreiben von Artefakten
- Berichterstellung
- Szenarioausführung
- Kompatibilitätsaliasse für ältere `qa-channel`-Szenarien

Runner-Plugins verwalten den Transportvertrag:

- wie `openclaw qa <runner>` unter dem gemeinsamen `qa`-Stamm eingebunden wird
- wie das Gateway für diesen Transport konfiguriert wird
- wie die Bereitschaft geprüft wird
- wie eingehende Ereignisse eingespeist werden
- wie ausgehende Nachrichten beobachtet werden
- wie Transkripte und der normalisierte Transportstatus bereitgestellt werden
- wie transportgestützte Aktionen ausgeführt werden
- wie transportspezifisches Zurücksetzen oder Bereinigen gehandhabt wird

Die Mindestanforderungen für die Einführung eines neuen Kanals:

1. Behalten Sie `qa-lab` als zuständige Komponente für den gemeinsamen `qa`-Stamm bei.
2. Implementieren Sie den Transport-Runner über die gemeinsame `qa-lab`-Host-Schnittstelle.
3. Belassen Sie transportspezifische Mechanismen im Runner-Plugin oder
   Kanal-Harness.
4. Binden Sie den Runner als `openclaw qa <runner>` ein, statt einen
   konkurrierenden Stammbefehl zu registrieren. Runner-Plugins sollten `qaRunners` in
   `openclaw.plugin.json` deklarieren und ein entsprechendes `qaRunnerCliRegistrations`-
   Array aus `runtime-api.ts` exportieren. Halten Sie `runtime-api.ts` schlank; die verzögerte CLI- und
   Runner-Ausführung sollte hinter separaten Einstiegspunkten verbleiben. Ein optionales
   `adapterFactory` stellt den Transport gemeinsamen Szenarien zur Verfügung, ohne
   den bestehenden Szenariokatalog des Befehls zu ändern. Partitionen desselben Kanals werden seriell
   ausgeführt, sofern die Factory nicht deklariert, dass jede Instanz isolierte Anmeldedaten oder
   kurzlebige Server, einen eigenen Gateway-Status und eigene Artefaktpfade besitzt.
5. Erstellen oder adaptieren Sie YAML-Szenarien unter den thematisch gegliederten `qa/scenarios/`-
   Verzeichnissen.
6. Verwenden Sie für neue Szenarien die generischen Szenario-Hilfsfunktionen.
7. Halten Sie bestehende Kompatibilitätsaliasse funktionsfähig, sofern im Repository nicht
   eine beabsichtigte Migration stattfindet.

Die Entscheidungsregel ist strikt:

- Wenn sich Verhalten einmalig in `qa-lab` ausdrücken lässt, legen Sie es in `qa-lab` ab.
- Wenn Verhalten von einem einzelnen Kanaltransport abhängt, belassen Sie es im entsprechenden Runner-
  Plugin oder Plugin-Harness.
- Wenn ein Szenario eine neue Funktion benötigt, die von mehr als einem Kanal genutzt werden kann,
  fügen Sie eine generische Hilfsfunktion statt einer kanalspezifischen Verzweigung in `suite.ts` hinzu.
- Wenn ein Verhalten nur für einen Transport sinnvoll ist, halten Sie das Szenario
  transportspezifisch und machen Sie dies im Szenariovertrag ausdrücklich kenntlich.

### Namen der Szenario-Hilfsfunktionen

Bevorzugte generische Hilfsfunktionen für neue Szenarien:

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

Kompatibilitätsaliasse bleiben für bestehende Szenarien verfügbar -
`waitForQaChannelReady`, `waitForOutboundMessage`, `waitForNoOutbound`,
`formatConversationTranscript`, `resetBus` -, für die Erstellung neuer Szenarien
sollten jedoch die generischen Namen verwendet werden. Die Aliasse dienen dazu, eine
Migration mit einem festen Stichtag zu vermeiden, und sind nicht das künftige Modell.

## Berichterstellung

`qa-lab` exportiert einen Markdown-Protokollbericht aus der beobachteten Bus-Zeitleiste.
Der Bericht sollte folgende Fragen beantworten:

- Was funktioniert hat
- Was fehlgeschlagen ist
- Was weiterhin blockiert war
- Welche Folgeszenarien ergänzt werden sollten

Führen Sie für das Inventar verfügbarer Szenarien – nützlich zur Einschätzung nachfolgender Arbeiten
oder zur Anbindung eines neuen Transports – `pnpm openclaw qa coverage` aus (fügen Sie `--json`
für maschinenlesbare Ausgabe hinzu). Führen Sie bei der Auswahl eines gezielten Nachweises für ein geändertes
Verhalten oder einen geänderten Dateipfad `pnpm openclaw qa coverage --match <query>` aus. Der
Übereinstimmungsbericht durchsucht Szenariometadaten, Dokumentationsreferenzen, Codereferenzen, Abdeckungs-IDs,
Plugins und Provider-Anforderungen und gibt anschließend passende `qa suite
--scenario ...`-Ziele aus.

Jeder `qa suite`-Lauf schreibt für den ausgewählten
Szenariosatz die Artefakte `qa-evidence.json`,
`qa-suite-summary.json` und `qa-suite-report.md` auf oberster Ebene. Szenarien, die `execution.kind: vitest` oder
`execution.kind: playwright` deklarieren, führen den passenden Testpfad aus und schreiben außerdem
szenariospezifische Protokolle. Szenarien, die `execution.kind: script` deklarieren, führen den
Nachweisproduzenten unter `execution.path` über `node --import tsx` aus (wobei
`${outputDir}` und `${scenarioId}` in `execution.args` erweitert werden); der
Produzent schreibt seine eigene `qa-evidence.json`, deren Einträge in
die Suite-Ausgabe importiert werden und deren Artefaktpfade relativ zu dieser
Produzenten-`qa-evidence.json` aufgelöst werden. Wenn `qa suite` über `qa run
--qa-profile` erreicht wird, enthält dieselbe `qa-evidence.json` außerdem die Zusammenfassung
der Profil-Scorecard für die ausgewählten Taxonomiekategorien.

Behandeln Sie die Abdeckungsausgabe als Hilfsmittel zur Ermittlung und nicht als Ersatz für Gates; das
ausgewählte Szenario benötigt weiterhin den richtigen Provider-Modus, Live-Transport,
Multipass, Testbox oder die richtige Release-Lane für das zu testende Verhalten. Kontext zur
Scorecard finden Sie unter [Reifegrad-Scorecard](/de/maturity/scorecard).

Führen Sie für Charakter- und Stilprüfungen dasselbe Szenario mit mehreren Live-
Modellreferenzen aus und erstellen Sie einen bewerteten Markdown-Bericht:

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.6-luna,thinking=medium,fast \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-8,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.6-sol,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-8,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

Der Befehl führt lokale untergeordnete QA-Gateway-Prozesse aus, nicht Docker. Szenarien zur
Charakterevaluierung sollten die Persona über `SOUL.md` festlegen und anschließend gewöhnliche
Benutzerinteraktionen wie Chats, Hilfe zum Arbeitsbereich und kleine Dateiaufgaben ausführen. Dem Kandidatenmodell
sollte nicht mitgeteilt werden, dass es evaluiert wird. Der Befehl bewahrt
jedes vollständige Transkript auf, zeichnet grundlegende Laufstatistiken auf und fordert anschließend die Bewertungsmodelle im
schnellen Modus mit `xhigh`-Reasoning, sofern unterstützt, dazu auf, die Läufe nach
Natürlichkeit, Stimmung und Humor zu ordnen. Verwenden Sie beim Vergleich von
Providern `--blind-judge-models`: Der Bewertungsprompt erhält weiterhin jedes Transkript und jeden Laufstatus, die
Kandidatenreferenzen werden jedoch durch neutrale Bezeichnungen wie `candidate-01` ersetzt; der
Bericht ordnet die Ranglisten nach dem Parsen wieder den tatsächlichen Referenzen zu.

Kandidatenläufe verwenden standardmäßig `high`-Thinking, mit `medium` für GPT-5.6 Luna und
`xhigh` für ältere OpenAI-Evaluierungsreferenzen, die dies unterstützen. Überschreiben Sie einen bestimmten
Kandidaten inline mit `--model provider/model,thinking=<level>`; Inline-
Optionen unterstützen außerdem `fast`, `no-fast` und `fast=<bool>`. `--thinking
<level>` legt weiterhin einen globalen Fallback fest, und die ältere `--model-thinking
<provider/model=level>`-Form bleibt aus Kompatibilitätsgründen erhalten. OpenAI-Kandidaten-
referenzen verwenden standardmäßig den schnellen Modus, sodass priorisierte Verarbeitung genutzt wird, sofern der Provider
sie unterstützt. Übergeben Sie `--fast` nur, wenn Sie den schnellen Modus für
jedes Kandidatenmodell erzwingen möchten. Die Laufzeiten von Kandidaten- und Bewertungsmodellen werden für die
Benchmark-Analyse im Bericht aufgezeichnet, die Bewertungsprompts weisen jedoch ausdrücklich an, nicht nach
Geschwindigkeit zu bewerten. Läufe von Kandidaten- und Bewertungsmodellen verwenden beide standardmäßig eine Parallelität von 16.
Verringern Sie `--concurrency` oder `--judge-concurrency`, wenn Provider-Limits oder lokale
Gateway-Auslastung einen Lauf zu störanfällig machen.

Wenn keine Kandidaten-`--model` übergeben werden, verwendet die Charakterevaluierung standardmäßig
`openai/gpt-5.6-luna`, `openai/gpt-5.2`, `openai/gpt-5`,
`anthropic/claude-opus-4-8`, `anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` und `google/gemini-3.1-pro-preview`. Wenn keine
`--judge-model` übergeben werden, verwenden die Bewertungsmodelle standardmäßig
`openai/gpt-5.6-sol,thinking=xhigh,fast` und
`anthropic/claude-opus-4-8,thinking=high`.

## Verwandte Dokumentation

- [Reifegrad-Scorecard](/de/maturity/scorecard)
- [Benchmark-Paket für persönliche Agenten](/de/concepts/personal-agent-benchmark-pack)
- [QA-Kanal](/de/channels/qa-channel)
- [Testen](/de/help/testing)
- [Dashboard](/de/web/dashboard)
