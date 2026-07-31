---
read_when:
    - Ausführen des Gateways über die CLI (Entwicklung oder Server)
    - Fehlerbehebung bei Gateway-Authentifizierung, Bindungsmodi und Konnektivität
    - Gateways über Bonjour erkennen (lokal + Wide-Area-DNS-SD)
    - Integration eines externen Prozess-Supervisors für den Gateway
sidebarTitle: Gateway
summary: OpenClaw-Gateway-CLI (`openclaw gateway`) — Gateways ausführen, abfragen und erkennen
title: Gateway
x-i18n:
    generated_at: "2026-07-26T18:17:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0188d7c79571ebf8f350295775625533a83cb2eb909bcc8763e8ce81806d2214
    source_path: cli/gateway.md
    workflow: 16
---

Der Gateway ist der WebSocket-Server von OpenClaw (Kanäle, Nodes, Sitzungen, Hooks). Alle nachfolgenden Unterbefehle befinden sich unter `openclaw gateway ...`.

<CardGroup cols={3}>
  <Card title="Bonjour-Erkennung" href="/de/gateway/bonjour">
    Einrichtung von lokalem mDNS und Wide-Area-DNS-SD.
  </Card>
  <Card title="Übersicht zur Erkennung" href="/de/gateway/discovery">
    Wie OpenClaw Gateways ankündigt und findet.
  </Card>
  <Card title="Konfiguration" href="/de/gateway/configuration">
    Gateway-Konfigurationsschlüssel auf oberster Ebene.
  </Card>
</CardGroup>

## Gateway ausführen

```bash
openclaw gateway
openclaw gateway run   # äquivalente, explizite Form
```

<AccordionGroup>
  <Accordion title="Startverhalten">
    - Der Start wird verweigert, sofern `gateway.mode=local` nicht in `~/.openclaw/openclaw.json` festgelegt ist. Verwenden Sie `--allow-unconfigured` für Ad-hoc-/Entwicklungsläufe; damit wird die Prüfung umgangen, ohne die Konfiguration zu schreiben oder zu reparieren.
    - Wenn beim Start eine reparierbare ungültige Konfiguration gefunden wird, bietet ein interaktives Terminal an, `openclaw doctor --fix` auszuführen, und versucht den Start nach Zustimmung einmal erneut. Nicht interaktive Läufe reparieren niemals automatisch; stattdessen geben sie den Befehl aus. Ist die reparierte Konfiguration weiterhin ungültig, bleibt der Startvorgang angehalten.
    - `openclaw onboard --mode local` und `openclaw setup` schreiben `gateway.mode=local`. Wenn die Konfigurationsdatei vorhanden ist, aber `gateway.mode` fehlt, wird dies als beschädigte/überschriebene Konfiguration behandelt, und der Gateway weigert sich, `local` für Sie zu erraten — führen Sie das Onboarding erneut aus, legen Sie den Schlüssel manuell fest oder übergeben Sie `--allow-unconfigured`.
    - Eine Bindung über die Loopback-Schnittstelle hinaus wird ohne Authentifizierung blockiert.
    - Die `--bind`-Werte `lan`, `tailnet` und `custom` werden derzeit ausschließlich über IPv4-Pfade aufgelöst; reine IPv6-Setups mit eigenem Host benötigen einen IPv4-Sidecar oder -Proxy vor dem Gateway.
    - `SIGUSR1` löst bei entsprechender Autorisierung einen prozessinternen Neustart aus. `commands.restart` (Standard: aktiviert) steuert extern gesendete `SIGUSR1`; setzen Sie den Wert auf `false`, um manuelle Neustarts durch Betriebssystemsignale zu blockieren. Das agentenseitige Werkzeug `gateway` ist schreibgeschützt; Agenten fordern einen Neustart über das von Menschen genehmigte Delegationswerkzeug `openclaw` an.
    - `SIGINT`/`SIGTERM` beenden den Prozess, stellen jedoch keinen benutzerdefinierten Terminalzustand wieder her — wenn Sie die CLI in eine TUI oder eine Eingabe im Raw-Modus einbetten, stellen Sie das Terminal vor dem Beenden selbst wieder her.

  </Accordion>
</AccordionGroup>

### Optionen

<ParamField path="--port <port>" type="number">
  WebSocket-Port (Standard aus Konfiguration/Umgebung; üblicherweise `18789`).
</ParamField>
<ParamField path="--bind <mode>" type="string">
  Bindungsmodus: `loopback` (Standard), `lan`, `tailnet`, `auto`, `custom`.
</ParamField>
<ParamField path="--token <token>" type="string">
  Gemeinsames Token für `connect.params.auth.token`. Verwendet standardmäßig `OPENCLAW_GATEWAY_TOKEN`, sofern festgelegt.
</ParamField>
<ParamField path="--auth <mode>" type="string">
  Authentifizierungsmodus: `none`, `token`, `password`, `trusted-proxy`.
</ParamField>
<ParamField path="--password <password>" type="string">
  Passwort für `--auth password`.
</ParamField>
<ParamField path="--password-file <path>" type="string">
  Das Gateway-Passwort aus einer Datei lesen.
</ParamField>
<ParamField path="--tailscale <mode>" type="string">
  Tailscale-Freigabe: `off`, `serve`, `funnel`.
</ParamField>
<ParamField path="--tailscale-reset-on-exit" type="boolean">
  Tailscale-Serve-/Funnel-Konfiguration beim Herunterfahren zurücksetzen.
</ParamField>
<ParamField path="--allow-unconfigured" type="boolean">
  Starten, ohne `gateway.mode=local` zu erzwingen. Nur für Ad-hoc-/Entwicklungs-Bootstrapping; die Konfiguration wird weder dauerhaft gespeichert noch repariert.
</ParamField>
<ParamField path="--dev" type="boolean">
  Bei Bedarf eine Entwicklungskonfiguration und einen Arbeitsbereich erstellen (überspringt `BOOTSTRAP.md`).
</ParamField>
<ParamField path="--dev-ambient-channels" type="boolean">
  Einem Entwicklungs-Gateway erlauben, Kanäle automatisch anhand vorhandener Umgebungsvariablen zu konfigurieren. Erfordert `--dev`.
</ParamField>
<ParamField path="--reset" type="boolean">
  Entwicklungskonfiguration, Anmeldedaten, Sitzungen und Arbeitsbereich zurücksetzen. Erfordert `--dev`.
</ParamField>
<ParamField path="--force" type="boolean">
  Vor dem Start alle vorhandenen Listener am Zielport beenden. In einer nicht interaktiven Shell wird das Beenden eines verifizierten Gateway-Listeners verweigert; verwenden Sie stattdessen `--dev` oder ein isoliertes `--profile` mit einem freien Port.
</ParamField>
<ParamField path="--verbose" type="boolean">
  Ausführliche Protokollierung an stdout/stderr.
</ParamField>
<ParamField path="--cli-backend-logs" type="boolean">
  Nur CLI-Backend-Protokolle in der Konsole anzeigen (aktiviert ebenfalls stdout/stderr).
</ParamField>
<ParamField path="--ws-log <style>" type="string" default="auto">
  WebSocket-Protokollstil: `auto`, `full`, `compact`.
</ParamField>
<ParamField path="--compact" type="boolean">
  Alias für `--ws-log compact`.
</ParamField>
<ParamField path="--raw-stream" type="boolean">
  Unverarbeitete Modellstream-Ereignisse in JSONL protokollieren.
</ParamField>
<ParamField path="--raw-stream-path <path>" type="string">
  JSONL-Pfad für den unverarbeiteten Stream.
</ParamField>

`--claude-cli-logs` ist ein veralteter Alias für `--cli-backend-logs`.

Legen Sie für `--bind custom` `gateway.customBindHost` auf eine IPv4-Adresse fest. Jede andere Adresse als `127.0.0.1` oder `0.0.0.0` erfordert außerdem `127.0.0.1` am selben Port für Clients auf demselben Host; der Start schlägt fehl, wenn einer der Listener keine Bindung herstellen kann. Der Platzhalter `0.0.0.0` fügt keinen separaten erforderlichen Alias hinzu. Reine IPv6-Setups mit eigenem Host benötigen einen IPv4-Sidecar oder -Proxy vor dem Gateway.

## Gateway neu starten

```bash
openclaw gateway restart
openclaw gateway restart --safe
openclaw gateway restart --safe --skip-deferral
openclaw gateway restart --force
openclaw gateway restart --wait 30s
```

`--safe` fordert den laufenden Gateway auf, aktive Arbeiten vorab zu prüfen und einen zusammengefassten Neustart zu planen, nachdem diese Arbeiten abgeschlossen sind. Die Wartezeit ist auf 5 Minuten begrenzt; nach Ablauf des Zeitbudgets wird der Neustart erzwungen. `--safe` kann nicht mit `--force` oder `--wait` kombiniert werden.

`--skip-deferral` umgeht bei einem sicheren Neustart die Zurückstellungsprüfung für aktive Arbeiten, sodass der Gateway trotz gemeldeter Blockaden sofort neu startet. Dies erfordert `--safe` — verwenden Sie diese Option, wenn eine Zurückstellung wegen einer außer Kontrolle geratenen Aufgabe festhängt.

`--wait <duration>` überschreibt das Zeitbudget für das Leeren bei einem einfachen (nicht sicheren) Neustart. Akzeptiert Millisekunden ohne Einheit oder die Einheitensuffixe `ms`, `s`, `m`, `h`, `d` (z. B. `30s`, `5m`, `1h30m`); `--wait 0` wartet unbegrenzt. Nicht mit `--force` oder `--safe` kompatibel.

`--force` überspringt das Leeren aktiver Arbeiten und startet sofort neu. Ein einfaches `restart` (ohne Flags) behält das bestehende Neustartverhalten des Dienstmanagers bei.

<Warning>
Inline angegebenes `--password` kann in lokalen Prozesslisten sichtbar sein. Bevorzugen Sie `--password-file`, die Umgebung oder ein durch SecretRef gestütztes `gateway.auth.password`.
</Warning>

### Externe Supervisoren

Legen Sie `OPENCLAW_SUPERVISOR_MODE=external` nur fest, wenn ein anderer Prozessmanager den Lebenszyklus des Gateways verwaltet. In diesem Modus gilt:

- `openclaw gateway restart` behält das bestehende sichere, erzwungene und zeitlich begrenzte Warteverhalten bei, richtet sich jedoch an den verifizierten laufenden Gateway statt an launchd, systemd oder Task Scheduler.
- Native Vorgänge zum Installieren, Starten, Stoppen und Deinstallieren des Dienstes werden mit einem Hinweis auf die Verwendung des externen Supervisors verweigert.
- Die Selbstaktualisierung von OpenClaw wird verweigert, damit der Supervisor den Gateway stoppen, die Laufzeitumgebung ersetzen und fertigstellen sowie sie sicher neu starten kann.
- Bei einem Neustart mit einem neuen Prozess wird vor dem ordnungsgemäßen Beenden eine zeitlich begrenzte SQLite-Übergabe geschrieben. Schlägt die Persistierung fehl, führt der Gateway stattdessen einen prozessinternen Neustart durch, anstatt ohne nutzbare Übergabe zu beenden.

`OPENCLAW_SERVICE_REPAIR_POLICY=external` bleibt eine separate Reparaturrichtlinie des Doctors. Sie legt nicht die Laufzeitzuständigkeit fest; Supervisoren, die beide Verhaltensweisen benötigen, sollten beide Variablen festlegen.

Externe Supervisoren können Neustartübergaben über den verborgenen Maschinenvertrag aushandeln und verarbeiten:

```bash
openclaw gateway restart-handoff capabilities --json
openclaw gateway restart-handoff consume --expected-pid <pid> --json
```

Protokollversion `1` unterstützt den Vorgang `consume`. Bei der Verarbeitung werden die erwartete PID und die begrenzten Übergabefelder innerhalb einer einzigen sofortigen SQLite-Transaktion validiert. Eine akzeptierte Übergabe wird gelöscht, bevor der Erfolg zurückgegeben wird, sodass parallele oder wiederholte Consumer sie nicht beide akzeptieren können. Eine nicht übereinstimmende PID bleibt für den passenden Besitzer erhalten; fehlende, abgelaufene und ungültige Zeilen autorisieren keinen Neustart.

Gültige Maschinenanfragen geben JSON mit Exit-Code `0` zurück, einschließlich Ergebnissen ohne Neustart. Ungültige Argumente geben `reason: "invalid-expected-pid"` mit Exit-Code `2` zurück; Fehler des Zustandsspeichers geben `reason: "store-unavailable"` mit Exit-Code `1` zurück. Supervisoren sollten `capabilities` mit exakt der Laufzeitumgebung oder dem Launcher prüfen, die bzw. den sie verwenden werden, statt die Unterstützung aus einer OpenClaw-Versionszeichenfolge abzuleiten oder das private SQLite-Schema direkt zu lesen.

### Gateway-Profiling

- `OPENCLAW_GATEWAY_STARTUP_TRACE=1` protokolliert während des Starts die Zeitmessungen der Phasen, einschließlich der `eventLoopMax`-Verzögerung pro Phase und der Zeitmessungen für Plugin-Nachschlagetabellen (installierter Index, Manifestregistrierung, Startplanung, Arbeit an der Besitzerzuordnung).
- `OPENCLAW_GATEWAY_RESTART_TRACE=1` protokolliert auf den Neustart beschränkte `restart trace:`-Zeilen: Signalverarbeitung, Leeren aktiver Arbeiten, Herunterfahrphasen, nächster Start, Bereitschaftszeit und Speicherkennzahlen.
- `OPENCLAW_DIAGNOSTICS=timeline` mit `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=<path>` schreibt nach bestem Bemühen eine JSONL-Zeitleiste der Startdiagnose für externe QA-Testumgebungen (entspricht der Konfiguration `diagnostics.flags: ["timeline"]`; der Pfad kann weiterhin nur über die Umgebung festgelegt werden). Fügen Sie `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` hinzu, um Ereignisschleifen-Stichproben einzubeziehen.
- `pnpm build` und anschließend `pnpm test:startup:gateway -- --runs 5 --warmup 1` misst den Gateway-Start anhand des erstellten CLI-Einstiegspunkts: erste Prozessausgabe, `/healthz`, `/readyz`, Zeitmessungen der Startablaufverfolgung, Ereignisschleifenverzögerung und Zeitmessung der Plugin-Nachschlagetabelle.
- `pnpm build` und anschließend `pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5` misst den prozessinternen Neustart unter macOS oder Linux (unter Windows nicht unterstützt; der Neustart erfordert `SIGUSR1`). Verwendet `SIGUSR1`, aktiviert beide Ablaufverfolgungen im untergeordneten Prozess und zeichnet den nächsten `/healthz`, den nächsten `/readyz`, die Ausfallzeit, die Bereitschaftszeit, CPU, RSS und Metriken der Neustartablaufverfolgung auf.
- `/healthz` bezeichnet die Erreichbarkeit; `/readyz` bezeichnet die nutzbare Bereitschaft. Behandeln Sie Ablaufverfolgungszeilen und Benchmark-Ausgaben als Signal für die Zuordnung zum Verantwortlichen, nicht als vollständige Leistungsbewertung anhand einer einzelnen Zeitspanne oder Stichprobe.

## Laufenden Gateway abfragen

Alle Abfragebefehle verwenden WebSocket-RPC.

<Tabs>
  <Tab title="Ausgabemodi">
    - Standard: menschenlesbar (im TTY farbig).
    - `--json`: maschinenlesbares JSON (ohne Gestaltung/Aktivitätsanzeige).
    - `--no-color` (oder `NO_COLOR=1`): ANSI deaktivieren und das menschenlesbare Layout beibehalten.

  </Tab>
  <Tab title="Gemeinsame Optionen">
    - `--url <url>`: WebSocket-URL des Gateways.
    - `--token <token>`: Gateway-Token.
    - `--password <password>`: Gateway-Passwort.
    - `--timeout <ms>`: Zeitüberschreitung/Zeitbudget (der Standard variiert je nach Befehl; siehe die einzelnen Befehle unten).
    - `--expect-final`: auf eine „endgültige“ Antwort warten (Agentenaufrufe).

  </Tab>
</Tabs>

<Note>
Wenn Sie `--url` festlegen, greift die CLI nicht auf Anmeldedaten aus der Konfiguration oder Umgebung zurück. Übergeben Sie `--token` oder `--password` ausdrücklich. Fehlende explizite Anmeldedaten sind ein Fehler.
</Note>

### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
openclaw gateway health --port 18789
```

`/healthz` ist eine Liveness-Probe: Sie kehrt zurück, sobald der Server HTTP-Anfragen beantworten kann. `/readyz` ist strenger und bleibt rot, solange beim Start Plugin-Sidecars, Kanäle oder konfigurierte Hooks noch nicht vollständig bereit sind. Lokale oder authentifizierte ausführliche `/readyz`-Antworten enthalten einen `eventLoop`-Diagnoseblock (Verzögerung, Auslastung, CPU-Kern-Verhältnis, `degraded`-Flag).

<ParamField path="--port <port>" type="number">
  Verwendet ein lokales Loopback-Gateway an diesem Port als Ziel. Überschreibt für diesen Aufruf `OPENCLAW_GATEWAY_URL` und `OPENCLAW_GATEWAY_PORT`.
</ParamField>

### `gateway usage-cost`

Ruft Zusammenfassungen der Nutzungskosten aus Sitzungsprotokollen ab.

```bash
openclaw gateway usage-cost
openclaw gateway usage-cost --days 7
openclaw gateway usage-cost --agent work --json
openclaw gateway usage-cost --all-agents
openclaw gateway usage-cost --json
```

<ParamField path="--days <days>" type="number" default="30">
  Anzahl der einzubeziehenden Tage.
</ParamField>
<ParamField path="--agent <id>" type="string">
  Beschränkt die Zusammenfassung auf die ID eines konfigurierten Agenten.
</ParamField>
<ParamField path="--all-agents" type="boolean">
  Aggregiert über alle konfigurierten Agenten hinweg. Kann nicht mit `--agent` kombiniert werden.
</ParamField>

### `gateway stability`

Ruft den aktuellen Diagnose-Stabilitätsrekorder von einem laufenden Gateway ab.

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --bundle latest
openclaw gateway stability --bundle latest --export
openclaw gateway stability --json
```

<ParamField path="--limit <limit>" type="number" default="25">
  Maximale Anzahl einzubeziehender aktueller Ereignisse (maximal `1000`).
</ParamField>
<ParamField path="--type <type>" type="string">
  Filtert nach Diagnoseereignistyp, z. B. `payload.large` oder `diagnostic.memory.pressure`.
</ParamField>
<ParamField path="--since-seq <seq>" type="number">
  Bezieht nur Ereignisse nach einer Diagnosesequenznummer ein.
</ParamField>
<ParamField path="--bundle [path]" type="string">
  Liest ein persistiertes Stabilitätspaket, statt das laufende Gateway aufzurufen. `--bundle latest` (oder nur `--bundle`) wählt das neueste Paket im Zustandsverzeichnis aus; alternativ können Sie direkt einen JSON-Pfad zu einem Paket übergeben.
</ParamField>
<ParamField path="--export" type="boolean">
  Schreibt eine teilbare ZIP-Datei mit Supportdiagnosen, statt Stabilitätsdetails auszugeben.
</ParamField>
<ParamField path="--output <path>" type="string">
  Ausgabepfad für `--export`.
</ParamField>

<AccordionGroup>
  <Accordion title="Datenschutz und Paketverhalten">
    - Datensätze enthalten betriebliche Metadaten: Ereignisnamen, Zähler, Bytegrößen, Speicherwerte, Warteschlangen-/Sitzungsstatus, Genehmigungs-IDs, Kanal-/Plugin-Namen und redigierte Sitzungszusammenfassungen. Sie schließen Chattext, Webhook-Inhalte, Tool-Ausgaben, unverarbeitete Anfrage-/Antwortinhalte, Tokens, Cookies, geheime Werte, Hostnamen und unverarbeitete Sitzungs-IDs aus. Setzen Sie `diagnostics.enabled: false`, um den Rekorder vollständig zu deaktivieren.
    - Bei schwerwiegenden Gateway-Abbrüchen, Zeitüberschreitungen beim Herunterfahren und Startfehlern nach einem Neustart wird derselbe Diagnoseschnappschuss nach `~/.openclaw/logs/stability/openclaw-stability-*.json` geschrieben, sofern der Rekorder Ereignisse enthält. Prüfen Sie das neueste Paket mit `openclaw gateway stability --bundle latest`; `--limit`, `--type` und `--since-seq` gelten auch für die Paketausgabe.

  </Accordion>
</AccordionGroup>

### `gateway diagnostics export`

Schreibt eine lokale Diagnose-ZIP-Datei für Fehlerberichte. Informationen zum Datenschutzmodell und Paketinhalt finden Sie unter [Diagnoseexport](/de/gateway/diagnostics).

```bash
openclaw gateway diagnostics export
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
openclaw gateway diagnostics export --json
```

<ParamField path="--output <path>" type="string">
  Ausgabepfad der ZIP-Datei. Standardmäßig wird ein Supportexport im Zustandsverzeichnis erstellt.
</ParamField>
<ParamField path="--log-lines <count>" type="number" default="5000">
  Maximale Anzahl einzubeziehender bereinigter Protokollzeilen.
</ParamField>
<ParamField path="--log-bytes <bytes>" type="number" default="1000000">
  Maximale Anzahl zu prüfender Protokollbytes.
</ParamField>
<ParamField path="--url <url>" type="string">
  Gateway-WebSocket-URL für den Zustandsschnappschuss.
</ParamField>
<ParamField path="--token <token>" type="string">
  Gateway-Token für den Zustandsschnappschuss.
</ParamField>
<ParamField path="--password <password>" type="string">
  Gateway-Passwort für den Zustandsschnappschuss.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="3000">
  Zeitüberschreitung für den Status-/Zustandsschnappschuss.
</ParamField>
<ParamField path="--no-stability-bundle" type="boolean">
  Überspringt die Suche nach einem persistierten Stabilitätspaket.
</ParamField>
<ParamField path="--json" type="boolean">
  Gibt den geschriebenen Pfad, die Größe und das Manifest als JSON aus.
</ParamField>

Der Export bündelt: `manifest.json` (Dateiinventar), `summary.md` (Markdown-Zusammenfassung), `diagnostics.json` (übergeordnete Zusammenfassung von Konfiguration/Protokollen/Erkennung/Stabilität/Status/Zustand), `config/sanitized.json`, `status/gateway-status.json`, `health/gateway-health.json`, `logs/openclaw-sanitized.jsonl` und `stability/latest.json`, sofern ein Paket vorhanden ist.

Er ist für die Weitergabe vorgesehen. Er behält betriebliche Details bei, die für die Fehlerdiagnose nützlich sind – sichere Protokollfelder, Subsystemnamen, Statuscodes, Zeitdauern, konfigurierte Modi, Ports, Plugin-/Provider-IDs, nicht geheime Funktionseinstellungen und redigierte betriebliche Protokollmeldungen – und lässt Chattext, Webhook-Inhalte, Tool-Ausgaben, Anmeldedaten, Cookies, Konto-/Nachrichtenkennungen, Prompt-/Anweisungstext, Hostnamen und geheime Werte weg oder redigiert sie. Wenn eine Protokollmeldung wie Text aus einer Benutzer-, Chat- oder Tool-Nutzlast aussieht (z. B. „Benutzer sagte“, „Chattext“, „Tool-Ausgabe“, „Webhook-Inhalt“), enthält der Export nur den Hinweis, dass eine Nachricht ausgelassen wurde, sowie ihre Byteanzahl.

### `gateway status`

Zeigt den Gateway-Dienst (launchd/systemd/schtasks) sowie optional eine Verbindungs-/Authentifizierungsprobe an.

```bash
openclaw gateway status
openclaw gateway status --json
openclaw gateway status --require-rpc
```

<ParamField path="--url <url>" type="string">
  Fügt ein explizites Probeziel hinzu. Das konfigurierte entfernte Ziel und localhost werden weiterhin geprüft.
</ParamField>
<ParamField path="--token <token>" type="string">
  Token-Authentifizierung für die Probe.
</ParamField>
<ParamField path="--password <password>" type="string">
  Passwortauthentifizierung für die Probe.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  Zeitüberschreitung der Probe.
</ParamField>
<ParamField path="--no-probe" type="boolean">
  Überspringt die Verbindungsprobe (reine Dienstansicht).
</ParamField>
<ParamField path="--deep" type="boolean">
  Durchsucht auch Dienste auf Systemebene.
</ParamField>
<ParamField path="--require-rpc" type="boolean">
  Erweitert die Verbindungsprobe zu einer Leseprobe und beendet den Prozess bei einem Fehlschlag mit einem von null verschiedenen Statuscode. Kann nicht mit `--no-probe` kombiniert werden.
</ParamField>

<AccordionGroup>
  <Accordion title="Statussemantik">
    - Bleibt für Diagnosen verfügbar, selbst wenn die lokale CLI-Konfiguration fehlt oder ungültig ist.
    - Die Standardausgabe weist den Dienststatus, die WebSocket-Verbindung und die beim Handshake sichtbare Authentifizierungsfähigkeit nach – nicht Lese-, Schreib- oder Administratorvorgänge.
    - Proben verändern bei der erstmaligen Geräteauthentifizierung nichts: Sie verwenden ein vorhandenes zwischengespeichertes Geräte-Token erneut, erstellen jedoch niemals nur zur Statusprüfung eine neue CLI-Geräteidentität oder einen schreibgeschützten Kopplungsdatensatz.
    - Löst konfigurierte SecretRefs für die Probeauthentifizierung nach Möglichkeit auf. Wenn eine erforderliche SecretRef nicht aufgelöst ist, meldet `--json` bei einem Fehlschlag der Probekonnektivität/-authentifizierung `rpc.authWarning`; übergeben Sie `--token`/`--password` explizit oder korrigieren Sie die Quelle des Geheimnisses. Warnungen zu nicht aufgelöster Authentifizierung werden unterdrückt, sobald die Probe erfolgreich ist.
    - Die JSON-Ausgabe enthält `gateway.version`, wenn das laufende Gateway dies meldet; `--require-rpc` kann auf die `status.runtimeVersion`-RPC-Nutzlast zurückgreifen, wenn die Handshake-Probe keine Versionsmetadaten liefern kann.
    - Verwenden Sie `--require-rpc` in Skripten/Automatisierungen, wenn ein lauschender Dienst nicht ausreicht und auch RPC mit Leseberechtigung fehlerfrei funktionieren muss.
    - `--deep` sucht nach zusätzlichen launchd-/systemd-/schtasks-Installationen; wenn mehrere Gateway-ähnliche Dienste gefunden werden, gibt die menschenlesbare Ausgabe Bereinigungshinweise aus (üblicherweise sollte pro Rechner ein Gateway ausgeführt werden) und meldet gegebenenfalls eine kürzlich erfolgte Neustartübergabe durch den Supervisor.
    - `--deep` führt außerdem die Konfigurationsvalidierung im Plugin-fähigen Modus (`pluginValidation: "full"`) aus und zeigt Warnungen zum Plugin-Manifest an (z. B. fehlende Metadaten zur Kanalkonfiguration). Das standardmäßige `gateway status` behält den schnellen, schreibgeschützten Pfad bei, der die Plugin-Validierung überspringt.
    - Die menschenlesbare Ausgabe enthält den aufgelösten Dateipfad des Protokolls sowie die Konfigurationspfade und deren Gültigkeit für CLI und Dienst, um Abweichungen bei Profilen oder Zustandsverzeichnissen leichter zu diagnostizieren.
    - Die menschenlesbare Ausgabe enthält `Gateway heap:` mit dem angewendeten Grenzwert und seiner adaptiven Herleitung. Die JSON-Ausgabe stellt denselben Bericht als `service.gatewayHeap` bereit.

  </Accordion>
  <Accordion title="Prüfungen auf Authentifizierungsabweichungen unter Linux systemd">
    - Prüfungen auf Abweichungen bei der Dienstauthentifizierung lesen sowohl `Environment=` als auch `EnvironmentFile=` aus der Unit (einschließlich `%h`, Pfaden in Anführungszeichen, mehreren Dateien und optionalen `-`-Dateien).
    - Löst `gateway.auth.token`-SecretRefs mithilfe der zusammengeführten Laufzeitumgebung auf (zuerst die Befehlsumgebung des Dienstes, dann ersatzweise die Prozessumgebung).
    - Prüfungen auf Token-Abweichungen überspringen die Auflösung des Konfigurations-Tokens, wenn die Token-Authentifizierung tatsächlich nicht aktiv ist (`gateway.auth.mode` explizit `password`/`none`/`trusted-proxy` oder kein Modus festgelegt ist, sodass das Passwort Vorrang haben kann und kein Token-Kandidat Vorrang erhalten kann).

  </Accordion>
</AccordionGroup>

### `gateway probe`

Der Befehl zum „Debuggen von allem“. Er prüft immer:

- Ihr konfiguriertes entferntes Gateway (sofern festgelegt) und
- localhost (Loopback), **selbst wenn ein entferntes Ziel konfiguriert ist**.

Durch die Übergabe von `--url` wird dieses explizite Ziel vor den beiden anderen hinzugefügt. Die menschenlesbare Ausgabe kennzeichnet die Ziele als `URL (explicit)`, `Remote (configured)` / `Remote (configured, inactive)` und `Local loopback`.

<Note>
Wenn mehrere Probeziele erreichbar sind, werden alle ausgegeben. Ein SSH-Tunnel, eine TLS-/Proxy-URL und eine konfigurierte entfernte URL können selbst bei unterschiedlichen Transportports auf dasselbe Gateway verweisen; `multiple_gateways` ist für unterschiedliche oder hinsichtlich ihrer Identität nicht eindeutig bestimmbare erreichbare Gateways reserviert. Die Ausführung mehrerer Gateways wird für isolierte Profile unterstützt (z. B. einen Rettungs-Bot), die meisten Installationen verwenden jedoch nur ein Gateway.
</Note>

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --port 18789
```

<ParamField path="--port <port>" type="number">
  Verwendet diesen Port für das lokale Loopback-Probeziel und den entfernten Port des SSH-Tunnels. Ohne `--url` wird dadurch ausschließlich das lokale Loopback-Ziel ausgewählt statt der konfigurierten Gateway-Umgebungs-URL, des Umgebungsports oder entfernter Ziele.
</ParamField>

<AccordionGroup>
  <Accordion title="Interpretation">
    - `Reachable: yes` bedeutet, dass mindestens ein Ziel eine WebSocket-Verbindung akzeptiert hat.
    - `Capability: read-only|write-capable|admin-capable|pairing-pending|connect-only` meldet unabhängig von der Erreichbarkeit, was die Probe hinsichtlich der Authentifizierung nachweisen konnte.
    - `Read probe: ok` bedeutet, dass auch detaillierte RPC-Aufrufe mit Leseberechtigung (`health`/`status`/`system-presence`/`config.get`) erfolgreich waren.
    - `Read probe: limited - missing scope: operator.read` bedeutet, dass die Verbindung erfolgreich war, RPC mit Leseberechtigung jedoch eingeschränkt ist. Dies wird als **beeinträchtigte** Erreichbarkeit gemeldet, nicht als vollständiger Fehlschlag.
    - `Read probe: failed` nach `Connect: ok` bedeutet, dass die WebSocket-Verbindung hergestellt wurde, nachfolgende Lesediagnosen jedoch eine Zeitüberschreitung erreichten oder fehlschlugen – ebenfalls **beeinträchtigt**, nicht unerreichbar.
    - Wie `gateway status` verwendet die Probe vorhandene zwischengespeicherte Geräteauthentifizierung erneut, erstellt jedoch keine erstmalige Geräteidentität oder keinen Kopplungsstatus.
    - Der Beendigungscode ist nur dann von null verschieden, wenn keines der geprüften Ziele erreichbar ist.

  </Accordion>
  <Accordion title="JSON-Ausgabe">
    Oberste Ebene:

    - `ok`: Mindestens ein Ziel ist erreichbar.
    - `degraded`: Mindestens ein Ziel hat eine Verbindung angenommen, aber die vollständige detaillierte RPC-Diagnose nicht abgeschlossen.
    - `capability`: Beste für erreichbare Ziele festgestellte Fähigkeit (`read_only`, `write_capable`, `admin_capable`, `pairing_pending`, `connected_no_operator_scope` oder `unknown`).
    - `primaryTargetId`: Bestes Ziel, das als aktiver Gewinner behandelt werden soll, in dieser Reihenfolge: explizite URL, SSH-Tunnel, konfigurierte Gegenstelle, lokale Loopback-Schnittstelle.
    - `warnings[]`: Nach bestem Bemühen erstellte Warnungsdatensätze mit `code`, `message` und optional `targetIds`.
    - `network`: Aus der aktuellen Konfiguration und dem Hostnetzwerk abgeleitete URL-Hinweise für lokale Loopback-Schnittstellen bzw. das Tailnet.
    - `discovery.timeoutMs` / `discovery.count`: Das für diesen Prüfdurchlauf tatsächlich verwendete Ermittlungsbudget bzw. die tatsächliche Ergebnisanzahl.

    Pro Ziel (`targets[].connect`): `ok` (Erreichbarkeit und Klassifizierung als eingeschränkt), `rpcOk` (vollständiger Erfolg des Detail-RPC), `scopeLimited` (Detail-RPC wegen fehlendem Operator-Berechtigungsumfang fehlgeschlagen).

    Pro Ziel (`targets[].auth`): `role` und `scopes` werden, sofern verfügbar, in `hello-ok` gemeldet, zusammen mit der ausgegebenen Klassifizierung `capability`.

  </Accordion>
  <Accordion title="Häufige Warncodes">
    - `ssh_tunnel_failed`: Die Einrichtung des SSH-Tunnels ist fehlgeschlagen; der Befehl ist auf direkte Prüfungen ausgewichen.
    - `multiple_gateways`: Unterschiedliche Gateway-Identitäten waren erreichbar, oder OpenClaw konnte nicht nachweisen, dass die erreichbaren Ziele dasselbe Gateway sind. Ein SSH-Tunnel, eine Proxy-URL oder eine konfigurierte Remote-URL zu demselben Gateway löst dies nicht aus.
    - `auth_secretref_unresolved`: Eine konfigurierte Authentifizierungs-SecretRef konnte für ein fehlgeschlagenes Ziel nicht aufgelöst werden.
    - `probe_scope_limited`: Die WebSocket-Verbindung wurde erfolgreich hergestellt, aber die Leseprüfung war durch das fehlende `operator.read` eingeschränkt.
    - `local_tls_runtime_unavailable`: TLS ist für das lokale Gateway aktiviert, aber OpenClaw konnte den Fingerabdruck des lokalen Zertifikats nicht laden.

  </Accordion>
</AccordionGroup>

#### Remotezugriff über SSH (Funktionsgleichheit mit der Mac-App)

Der Modus „Remote over SSH“ der macOS-App verwendet eine lokale Portweiterleitung, damit ein ausschließlich über die Loopback-Schnittstelle erreichbares Remote-Gateway unter `ws://127.0.0.1:<port>` erreichbar wird.

CLI-Entsprechung:

```bash
openclaw gateway probe --ssh user@gateway-host
```

<ParamField path="--ssh <target>" type="string">
  `user@host` oder `user@host:port` (Port ist standardmäßig `22`).
</ParamField>
<ParamField path="--ssh-identity <path>" type="string">
  Identitätsdatei.
</ParamField>
<ParamField path="--ssh-auto" type="boolean">
  Wählt den ersten ermittelten Gateway-Host vom aufgelösten Ermittlungsendpunkt als SSH-Ziel aus (`local.` sowie, sofern vorhanden, die konfigurierte Weitbereichsdomäne). Reine TXT-Hinweise werden ignoriert.
</ParamField>

Konfigurationsstandardwerte (optional): `gateway.remote.sshTarget`, `gateway.remote.sshIdentity`.

### `gateway call <method>`

Niedrigstufiges RPC-Hilfsprogramm.

```bash
openclaw gateway call status
openclaw gateway call logs.tail --params '{"limit": 200}'
```

<ParamField path="--params <json>" type="string" default="{}">
  JSON-Objektzeichenfolge für Parameter.
</ParamField>
<ParamField path="--url <url>" type="string">
  WebSocket-URL des Gateways.
</ParamField>
<ParamField path="--token <token>" type="string">
  Gateway-Token.
</ParamField>
<ParamField path="--password <password>" type="string">
  Gateway-Passwort.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  Zeitüberschreitungsbudget.
</ParamField>
<ParamField path="--expect-final" type="boolean">
  Hauptsächlich für agentenartige RPCs, die vor einer abschließenden Nutzlast Zwischenereignisse streamen.
</ParamField>
<ParamField path="--json" type="boolean">
  Maschinenlesbare JSON-Ausgabe.
</ParamField>

<Note>
`--params` muss gültiges JSON sein, und jede Methode validiert ihre eigene Parameterstruktur (zusätzliche oder falsch benannte Felder werden abgelehnt).
</Note>

## Gateway-Dienst verwalten

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway uninstall
```

### Installation mit einem Wrapper

Verwenden Sie `--wrapper`, wenn der verwaltete Dienst über eine andere ausführbare Datei gestartet werden muss, beispielsweise über einen Shim für eine Geheimnisverwaltung oder ein Hilfsprogramm zur Ausführung unter einem anderen Benutzer. Der Wrapper empfängt die normalen Gateway-Argumente und ist dafür verantwortlich, schließlich `openclaw` oder Node mit diesen Argumenten per exec auszuführen.

```bash
cat > ~/.local/bin/openclaw-doppler <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
exec doppler run --project my-project --config production -- openclaw "$@"
EOF
chmod +x ~/.local/bin/openclaw-doppler

openclaw gateway install --wrapper ~/.local/bin/openclaw-doppler --force
openclaw gateway restart
```

Sie können den Wrapper auch über die Umgebung festlegen. `gateway install` prüft, ob der Pfad eine ausführbare Datei ist, schreibt den Wrapper in `ProgramArguments` des Dienstes und speichert `OPENCLAW_WRAPPER` in der Dienstumgebung für spätere erzwungene Neuinstallationen, Aktualisierungen und Reparaturen durch Doctor.

```bash
OPENCLAW_WRAPPER="$HOME/.local/bin/openclaw-doppler" openclaw gateway install --force
openclaw doctor
```

Um einen gespeicherten Wrapper zu entfernen, leeren Sie `OPENCLAW_WRAPPER` während der Neuinstallation:

```bash
OPENCLAW_WRAPPER= openclaw gateway install --force
openclaw gateway restart
```

<AccordionGroup>
  <Accordion title="Befehlsoptionen">
    - `gateway status`: `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json`
    - `gateway install`: `--port`, `--runtime <node>` (Standardwert: `node`), `--token`, `--wrapper <path>`, `--force`, `--json`
    - `gateway restart`: `--safe`, `--skip-deferral`, `--force`, `--wait <duration>`, `--json`
    - `gateway uninstall|start`: `--json`
    - `gateway stop`: `--disable`, `--force`, `--json`

  </Accordion>
  <Accordion title="Lebenszyklusverhalten">
    - `gateway start` ist idempotent: Wenn der verwaltete Dienst bereits ausgeführt wird, meldet der Befehl den laufenden Prozess und lässt ihn unverändert. Ein geladener, aber angehaltener Dienst wird wie zuvor gestartet.
    - Verwenden Sie `gateway restart`, um einen verwalteten Dienst neu zu starten. Verketten Sie nicht `gateway stop` und `gateway start` als Ersatz für einen Neustart.
    - In einer nicht interaktiven Shell erfordert `gateway stop` die Option `--force`. Interaktive Terminals behalten das bestehende Verhalten ohne Eingabeaufforderung bei. Für Automatisierung und Tests sollten Sie `gateway run --dev` oder ein isoliertes `--profile` mit einem freien Port bevorzugen.
    - Unter macOS verwendet `gateway stop` standardmäßig `launchctl bootout`. Dadurch wird der LaunchAgent aus der aktuellen Startsitzung entfernt, ohne eine Deaktivierung dauerhaft zu speichern – die automatische KeepAlive-Wiederherstellung bleibt bei zukünftigen Abstürzen aktiv, und `gateway start` aktiviert den Dienst ohne ein manuelles `launchctl enable` wieder ordnungsgemäß. Übergeben Sie `--disable`, um KeepAlive und RunAtLoad dauerhaft zu unterdrücken, sodass das Gateway erst nach dem nächsten expliziten `gateway start` erneut gestartet wird; verwenden Sie dies, wenn ein manuelles Anhalten Neustarts des Systems überdauern soll.
    - Mutationen am Gateway-Lebenszyklus hängen nach bestem Bemühen Schlüssel-Wert-Prüfdatensätze an `<state-dir>/logs/gateway-restart.log` an, darunter CLI-Start-, Stopp- und Neustartvorgänge, Anforderungen für sichere Neustarts, Supervisor-Neustarts und entkoppelte Übergaben.
    - Lebenszyklusbefehle akzeptieren `--json` für Skripting.

  </Accordion>
  <Accordion title="Heap-Dimensionierung des verwalteten Gateways">
    - `gateway install` schreibt für den verwalteten Gateway-Dienst einen ausschließlich den Heap betreffenden Wert für `NODE_OPTIONS`. Wenn Node eine Container- oder Dienstbegrenzung meldet, werden 50 % des begrenzten Arbeitsspeichers angestrebt, andernfalls 50 % des physischen Arbeitsspeichers.
    - Der nominelle Zielbereich beträgt 2048–8192 MiB, mit einer zusätzlichen Obergrenze, die 75 % für nativen Speicher freihält. Auf kleinen Hosts kann diese Obergrenze dazu führen, dass das angewendete Limit unter dem nominellen Mindestwert von 2048 MiB liegt.
    - Ein gültiges explizites `--max-old-space-size`, das bereits im installierten Dienst gespeichert ist, bleibt bei erzwungenen Neuinstallationen und Reparaturen durch Doctor erhalten. Andere `NODE_OPTIONS`-Flags werden nicht in den verwalteten Dienst übernommen.
    - Das aus der Shell-Umgebung stammende `NODE_OPTIONS` setzt diese Richtlinie nicht außer Kraft. Verwenden Sie `gateway status` oder `doctor`, um den installierten Wert zu prüfen; führen Sie `openclaw gateway install --force` aus, um ältere Dienstmetadaten neu zu erzeugen, die keine verwaltete Heap-Einstellung enthalten.
    - Die Richtlinie gilt nur für den verwalteten Gateway-Dienst. Im Vordergrund ausgeführte `gateway run`, Node-Dienste und manuell erstellte Supervisor-Einheiten behalten ihre eigene Laufzeitkonfiguration bei.

  </Accordion>
  <Accordion title="Authentifizierung und SecretRefs zum Installationszeitpunkt">
    - Wenn die Token-Authentifizierung ein Token erfordert und `gateway.auth.token` über eine SecretRef verwaltet wird, prüft `gateway install`, ob die SecretRef aufgelöst werden kann, speichert das aufgelöste Token jedoch nicht in den Umgebungsmetadaten des Dienstes.
    - Wenn die Token-Authentifizierung ein Token erfordert und die konfigurierte Token-SecretRef nicht aufgelöst werden kann, schlägt die Installation sicher fehl, statt ersatzweise Klartext zu speichern.
    - Bevorzugen Sie für die Passwortauthentifizierung auf `gateway run` `OPENCLAW_GATEWAY_PASSWORD`, `--password-file` oder ein durch SecretRef gestütztes `gateway.auth.password` gegenüber einem eingebetteten `--password`.
    - Im abgeleiteten Authentifizierungsmodus lockert das ausschließlich in der Shell gesetzte `OPENCLAW_GATEWAY_PASSWORD` die Token-Anforderungen der Installation nicht; verwenden Sie bei der Installation eines verwalteten Dienstes eine dauerhafte Konfiguration (`gateway.auth.password` oder die Konfiguration `env`).
    - Wenn sowohl `gateway.auth.token` als auch `gateway.auth.password` konfiguriert sind und `gateway.auth.mode` nicht festgelegt ist, wird die Installation blockiert, bis der Modus explizit festgelegt wurde.

  </Accordion>
</AccordionGroup>

## Gateways ermitteln (Bonjour)

`gateway discover` sucht nach Gateway-Beacons (`_openclaw-gw._tcp`).

- Multicast-DNS-SD: `local.`
- Unicast-DNS-SD (Weitbereichs-Bonjour): Wählen Sie eine Domäne aus (Beispiel: `openclaw.internal.`) und richten Sie Split-DNS sowie einen DNS-Server ein; siehe [Bonjour](/de/gateway/bonjour).

Nur Gateways mit aktivierter Bonjour-Ermittlung (Standard) kündigen den Beacon an.

TXT-Hinweise in jedem Beacon: `role` (Hinweis zur Gateway-Rolle), `transport` (Transporthinweis, z. B. `gateway`), `gatewayPort` (WebSocket-Port, normalerweise `18789`), `tailnetDns` (MagicDNS-Hostname, sofern verfügbar), `gatewayTls` / `gatewayTlsSha256` (TLS aktiviert und Zertifikatfingerabdruck). `sshPort` und `cliPath` werden nur im vollständigen Ermittlungsmodus veröffentlicht (`discovery.mdns.mode: "full"`; Standard ist `"minimal"`, wodurch sie ausgelassen werden – Clients verwenden dann standardmäßig Port `22` für SSH-Ziele).

### `gateway discover`

```bash
openclaw gateway discover
```

<ParamField path="--timeout <ms>" type="number" default="2000">
  Zeitüberschreitung pro Befehl (Durchsuchen/Auflösen).
</ParamField>
<ParamField path="--json" type="boolean">
  Maschinenlesbare Ausgabe (deaktiviert außerdem Formatierung und Aktivitätsanzeige).
</ParamField>

Beispiele:

```bash
openclaw gateway discover --timeout 4000
openclaw gateway discover --json | jq '.beacons[].wsUrl'
```

<Note>
- Durchsucht `local.` sowie die konfigurierte Weitbereichsdomäne, sofern eine aktiviert ist.
- `wsUrl` in der JSON-Ausgabe wird aus dem aufgelösten Dienstendpunkt abgeleitet, nicht aus reinen TXT-Hinweisen wie `lanHost` oder `tailnetDns`.
- `discovery.mdns.mode` steuert die Veröffentlichung von `sshPort`/`cliPath` sowohl über `local.`-mDNS als auch über Weitbereichs-DNS-SD (siehe oben).

</Note>

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Gateway-Betriebshandbuch](/de/gateway)
