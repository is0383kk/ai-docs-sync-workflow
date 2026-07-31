---
read_when:
    - Ausführen oder Debuggen des Gateway-Prozesses
summary: Betriebshandbuch für Gateway-Dienst, Lebenszyklus und Betrieb
title: Gateway-Betriebshandbuch
x-i18n:
    generated_at: "2026-07-26T18:27:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d8b50b6041905c321887ea0f579f8d4c3b74552b2b72c37ec655e43a53dfc130
    source_path: gateway/index.md
    workflow: 16
---

Verwenden Sie diese Seite für die erstmalige Inbetriebnahme und den laufenden Betrieb des Gateway-Dienstes.

<CardGroup cols={2}>
  <Card title="Ausführliche Fehlerbehebung" icon="siren" href="/de/gateway/troubleshooting">
    Symptombasierte Diagnose mit genauen Befehlsfolgen und Log-Signaturen.
  </Card>
  <Card title="Konfiguration" icon="sliders" href="/de/gateway/configuration">
    Aufgabenorientierte Einrichtungsanleitung und vollständige Konfigurationsreferenz.
  </Card>
  <Card title="Secret-Verwaltung" icon="key-round" href="/de/gateway/secrets">
    SecretRef-Vertrag, Verhalten von Laufzeit-Snapshots sowie Migrations- und Neuladevorgänge.
  </Card>
  <Card title="Vertrag für den Secrets-Plan" icon="shield-check" href="/de/gateway/secrets-plan-contract">
    Genaue `secrets apply`-Ziel-/Pfadregeln und Verhalten von Authentifizierungsprofilen, die ausschließlich Referenzen enthalten.
  </Card>
</CardGroup>

## Lokale Inbetriebnahme in 5 Minuten

<Steps>
  <Step title="Gateway starten">

```bash
openclaw gateway --port 18789
# Debug-/Trace-Ausgaben werden nach stdio gespiegelt
openclaw gateway --port 18789 --verbose
# Listener am ausgewählten Port zwangsweise beenden, dann starten
openclaw gateway --force
```

  </Step>

  <Step title="Dienstzustand überprüfen">

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
```

Gesunder Ausgangszustand: `Runtime: running`, `Connectivity probe: ok` und eine Ihren Erwartungen entsprechende `Capability`-Zeile. Verwenden Sie `openclaw gateway status --require-rpc` als RPC-Nachweis für den Lesezugriff, nicht nur für die Erreichbarkeit.

  </Step>

  <Step title="Kanalbereitschaft validieren">

```bash
openclaw channels status --probe
```

Bei erreichbarem Gateway führt dies Live-Kanalprüfungen pro Konto und optionale Audits aus. Ist das Gateway nicht erreichbar, greift die CLI auf rein konfigurationsbasierte Kanalzusammenfassungen zurück.

  </Step>
</Steps>

<Note>
Das Neuladen der Gateway-Konfiguration überwacht den Pfad der aktiven Konfigurationsdatei, der aus den Profil-/Zustandsvorgaben oder, falls festgelegt, aus `OPENCLAW_CONFIG_PATH` aufgelöst wird. Der Standardmodus ist `gateway.reload.mode="hybrid"`. Nach dem ersten erfolgreichen Laden stellt der laufende Prozess den aktiven Konfigurations-Snapshot im Arbeitsspeicher bereit; ein erfolgreiches Neuladen ersetzt diesen Snapshot atomar.
</Note>

## Laufzeitmodell

- Ein dauerhaft aktiver Prozess für Routing, Steuerungsebene und Kanalverbindungen.
- Ein einzelner multiplexter Port für:
  - WebSocket-Steuerung/RPC
  - HTTP-APIs (`/v1/models`, `/v1/embeddings`, `/v1/chat/completions`, `/v1/responses`, `/tools/invoke`)
  - Plugin-HTTP-Routen, beispielsweise das optionale `/api/v1/admin/rpc`
  - Control UI und Hooks
- Standard-Bindungsmodus: `loopback`. Innerhalb einer erkannten Container-Umgebung ist der effektive Standard `auto` (wird für die Portweiterleitung in `0.0.0.0` aufgelöst), sofern Tailscale Serve/Funnel nicht aktiv ist; dies erzwingt stets `loopback`.
- Authentifizierung ist standardmäßig erforderlich. Konfigurationen mit gemeinsamem Secret verwenden `gateway.auth.token` / `gateway.auth.password` (oder `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`); Reverse-Proxy-Konfigurationen außerhalb von Loopback können `gateway.auth.mode: "trusted-proxy"` verwenden.

## OpenAI-kompatible Endpunkte

OpenClaws Kompatibilitätsoberfläche mit der größten Hebelwirkung:

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/chat/completions`
- `POST /v1/responses`

Warum diese Auswahl wichtig ist:

- Die meisten Integrationen mit Open WebUI, LobeChat und LibreChat prüfen zuerst `/v1/models`.
- Viele RAG- und Speicher-Pipelines erwarten `/v1/embeddings`.
- Für Agenten entwickelte Clients bevorzugen zunehmend `/v1/responses`.

`/v1/models` ist auf Agenten ausgerichtet: Es gibt für jeden konfigurierten Agenten `openclaw`, `openclaw/default` und `openclaw/<agentId>` zurück. `openclaw/default` ist der stabile Alias, der stets dem konfigurierten Standardagenten zugeordnet wird. Senden Sie `x-openclaw-model`, wenn Sie den Provider oder das Modell im Backend überschreiben möchten; andernfalls bleibt die normale Modell- und Embedding-Konfiguration des ausgewählten Agenten maßgeblich.

Alle diese Endpunkte werden über den Hauptport des Gateways ausgeführt und verwenden dieselbe vertrauenswürdige Authentifizierungsgrenze für Bediener wie die übrige Gateway-HTTP-API.

Admin-HTTP-RPC (`POST /api/v1/admin/rpc`) ist eine separate, standardmäßig deaktivierte Plugin-Route für Host-Werkzeuge, die WebSocket-RPC nicht verwenden können. Siehe [Admin-HTTP-RPC](/de/plugins/admin-http-rpc).

### Priorität von Port und Bindung

| Einstellung  | Auflösungsreihenfolge                                                 |
| ------------ | -------------------------------------------------------------------- |
| Gateway-Port | `--port` → `OPENCLAW_GATEWAY_PORT` → `gateway.port` → `18789`        |
| Bindungsmodus | CLI/Überschreibung → `gateway.bind` → `loopback` (oder `auto` in Containern) |

Installierte Gateway-Dienste speichern das aufgelöste `--port` in den Supervisor-Metadaten. Führen Sie nach einer Änderung von `gateway.port` den Befehl `openclaw doctor --fix` oder `openclaw gateway install --force` aus, damit launchd/systemd/schtasks den Prozess am neuen Port startet.

Beim Start verwendet das Gateway denselben effektiven Port und dieselbe Bindung, wenn es lokale Ursprünge der Control UI für Bindungen außerhalb von Loopback vorbelegt. Beispielsweise belegt `--bind lan --port 3000` vor der Laufzeitvalidierung `http://localhost:3000` und `http://127.0.0.1:3000` vor. Fügen Sie alle Ursprünge entfernter Browser, etwa HTTPS-Proxy-URLs, ausdrücklich zu `gateway.controlUi.allowedOrigins` hinzu.

### Hot-Reload-Modi

| `gateway.reload.mode` | Verhalten                                  |
| --------------------- | ------------------------------------------ |
| `off`                 | Kein Neuladen der Konfiguration            |
| `hot`                 | Nur Hot-Safe-Änderungen anwenden           |
| `restart`             | Bei Änderungen mit erforderlichem Neustart neu starten |
| `hybrid` (Standard)    | Wenn sicher, direkt anwenden; bei Bedarf neu starten |

## Befehlssatz für Bediener

```bash
openclaw gateway status
openclaw gateway status --deep   # ergänzt eine systemweite Dienstsuche
openclaw gateway status --json
openclaw gateway install
openclaw gateway restart
openclaw gateway stop
openclaw secrets reload
openclaw logs --follow
openclaw doctor
```

`gateway status --deep` dient der zusätzlichen Dienstsuche (LaunchDaemons/systemd-System-Units/schtasks), nicht einer tiefergehenden RPC-Zustandsprüfung.

## Mehrere Gateways (auf demselben Host)

Die meisten Installationen sollten ein Gateway pro Rechner ausführen. Ein einzelnes Gateway kann mehrere Agenten und Kanäle bereitstellen. Mehrere Gateways sind nur erforderlich, wenn Sie bewusst eine Isolierung oder einen Rettungs-Bot wünschen.

Nützliche Prüfungen:

```bash
openclaw gateway status --deep
openclaw gateway probe
```

Zu erwartendes Verhalten:

- `gateway status --deep` kann `Other gateway-like services detected (best effort)` melden und Bereinigungshinweise ausgeben, wenn noch veraltete launchd-/systemd-/schtasks-Installationen vorhanden sind.
- `gateway probe` kann vor `multiple reachable gateway identities` warnen, wenn unterschiedliche Gateways antworten oder OpenClaw nicht nachweisen kann, dass erreichbare Ziele dasselbe Gateway sind. Ein SSH-Tunnel, eine Proxy-URL oder eine konfigurierte Remote-URL zu demselben Gateway ist ein Gateway mit mehreren Transportwegen, selbst wenn sich die Transportports unterscheiden.
- Wenn dies beabsichtigt ist, isolieren Sie Ports, Konfiguration/Zustand und Workspace-Stammverzeichnisse für jedes Gateway.

Checkliste pro Instanz:

- Eindeutiges `gateway.port`
- Eindeutiges `OPENCLAW_CONFIG_PATH`
- Eindeutiges `OPENCLAW_STATE_DIR`
- Eindeutiges `agents.defaults.workspace`

Beispiel:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json OPENCLAW_STATE_DIR=~/.openclaw-a openclaw gateway --port 19001
OPENCLAW_CONFIG_PATH=~/.openclaw/b.json OPENCLAW_STATE_DIR=~/.openclaw-b openclaw gateway --port 19002
```

Ausführliche Einrichtung: [/gateway/multiple-gateways](/de/gateway/multiple-gateways).

## Remote-Zugriff

Bevorzugt: Tailscale/VPN.
Ausweichlösung: SSH-Tunnel.

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

Verbinden Sie Clients anschließend lokal mit `ws://127.0.0.1:18789`.

<Warning>
SSH-Tunnel umgehen die Gateway-Authentifizierung nicht. Bei der Authentifizierung mit gemeinsamem Secret müssen Clients auch über den Tunnel weiterhin `token`/`password` senden. Bei identitätstragenden Modi muss die Anfrage weiterhin den entsprechenden Authentifizierungspfad erfüllen.
</Warning>

Siehe: [Remote-Gateway](/de/gateway/remote), [Authentifizierung](/de/gateway/authentication), [Tailscale](/de/gateway/tailscale).

## Überwachung und Dienstlebenszyklus

Verwenden Sie überwachte Ausführungen für eine produktionsähnliche Zuverlässigkeit.

<Tabs>
  <Tab title="macOS (launchd)">

```bash
openclaw gateway install
openclaw gateway status
openclaw gateway restart
openclaw gateway stop
```

Verwenden Sie `openclaw gateway restart` für Neustarts. Verketten Sie `openclaw gateway stop` und `openclaw gateway start` nicht als Ersatz für einen Neustart.

Unter macOS verwendet `gateway stop` standardmäßig `launchctl bootout`. Dadurch wird der LaunchAgent aus der aktuellen Startsitzung entfernt, ohne eine Deaktivierung dauerhaft zu speichern. Die automatische Wiederherstellung durch KeepAlive funktioniert somit weiterhin nach unerwarteten Abstürzen, und `gateway start` aktiviert den Dienst wieder ordnungsgemäß. Um den automatischen Neustart über Systemneustarts hinweg dauerhaft zu unterdrücken, übergeben Sie `--disable`: `openclaw gateway stop --disable`.

LaunchAgent-Bezeichnungen sind `ai.openclaw.gateway` (Standard) oder `ai.openclaw.<profile>` (benanntes Profil). `openclaw doctor` prüft und behebt Abweichungen der Dienstkonfiguration.

  </Tab>

  <Tab title="Linux (systemd-Benutzerdienst)">

```bash
openclaw gateway install
systemctl --user enable --now openclaw-gateway[-<profile>].service
openclaw gateway status
```

Aktivieren Sie für den dauerhaften Betrieb nach der Abmeldung das Lingering:

```bash
sudo loginctl enable-linger $(whoami)
```

Stellen Sie auf einem Headless-Server ohne Desktop-Sitzung außerdem sicher, dass `XDG_RUNTIME_DIR` festgelegt ist (`export XDG_RUNTIME_DIR=/run/user/$(id -u)`), bevor Sie die `systemctl --user`-Befehle erneut ausführen.

Beispiel für eine manuelle Benutzer-Unit, wenn Sie einen benutzerdefinierten Installationspfad benötigen:

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target
StartLimitBurst=5
StartLimitIntervalSec=60

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
RestartPreventExitStatus=78
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
OOMPolicy=continue
KillMode=control-group

[Install]
WantedBy=default.target
```

  </Tab>

  <Tab title="Windows (nativ)">

```powershell
openclaw gateway install
openclaw gateway status --json
openclaw gateway restart
openclaw gateway stop
```

Der verwaltete native Windows-Start verwendet eine geplante Aufgabe namens `OpenClaw Gateway` (oder `OpenClaw Gateway (<profile>)` für benannte Profile). Wird das Erstellen der geplanten Aufgabe verweigert, greift OpenClaw auf ein benutzerspezifisches Startprogramm im Autostartordner zurück, das auf `gateway.cmd` im Zustandsverzeichnis verweist.

  </Tab>

  <Tab title="Linux (Systemdienst)">

Verwenden Sie für Mehrbenutzer-Hosts oder dauerhaft aktive Hosts eine System-Unit.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway[-<profile>].service
```

Verwenden Sie denselben Dienstinhalt wie für die Benutzer-Unit, installieren Sie ihn jedoch unter `/etc/systemd/system/openclaw-gateway[-<profile>].service` und passen Sie `ExecStart=` an, wenn sich Ihre ausführbare Datei `openclaw` an einem anderen Ort befindet.

Lassen Sie `openclaw doctor --fix` nicht zusätzlich einen Gateway-Dienst auf Benutzerebene für dasselbe Profil bzw. denselben Port installieren. Doctor verweigert diese automatische Installation, wenn ein OpenClaw-Gateway-Dienst auf Systemebene gefunden wird. Verwenden Sie `OPENCLAW_SERVICE_REPAIR_POLICY=external`, wenn die System-Unit für den Lebenszyklus zuständig ist.

  </Tab>
</Tabs>

Fehler aufgrund einer ungültigen Konfiguration beenden den Prozess mit Code `78`. Linux-systemd-Units verwenden `RestartPreventExitStatus=78`, um weitere Starts zu verhindern, bis die Konfiguration korrigiert wurde. launchd und die Windows-Aufgabenplanung besitzen keine entsprechende Regel zum Anhalten bei einem bestimmten Exit-Code. Daher speichert das Gateway zusätzlich den Verlauf schneller unsauberer Starts und unterdrückt nach wiederholten Startfehlern den automatischen Start von Kanal-/Provider-Konten. In diesem abgesicherten Modus startet die Steuerungsebene weiterhin zur Prüfung und Reparatur; Hot-Reloads der Konfiguration und `secrets.reload` verweigern automatische Kanalneustarts, und eine ausdrückliche `channels.start`-Anforderung durch den Bediener kann die Unterdrückung außer Kraft setzen.

## Schnellstart mit Entwicklungsprofil

```bash
openclaw --dev setup
openclaw --dev gateway --allow-unconfigured
openclaw --dev status
```

Zu den Standardwerten gehören eine isolierte Zustands-/Konfigurationsumgebung und der Gateway-Basisport `19001`.

## Protokoll-Kurzreferenz (Bedieneransicht)

- Der erste Client-Frame muss `connect` sein.
- Der Gateway gibt einen `hello-ok`-Frame mit einem `snapshot` (`presence`, `health`, `stateVersion`, `uptimeMs`) sowie `policy`-Grenzwerten (`maxPayload`, `maxBufferedBytes`, `tickIntervalMs`) zurück.
- `hello-ok.features.methods` / `events` sind eine konservative Ermittlungsliste und keine
  generierte Auflistung aller aufrufbaren Hilfsrouten.
- Anfragen: `req(method, params)` → `res(ok/payload|error)`.
- Zu den gängigen Ereignissen gehören `connect.challenge`, `agent`, `chat`,
  `session.message`, `session.operation`, `session.tool`, das optional aktivierbare
  `session.approval`, `sessions.changed`, `presence`, `tick`, `health`,
  `heartbeat`, Lebenszyklusereignisse für Kopplung/Genehmigung und `shutdown`.

Agent-Ausführungen erfolgen in zwei Phasen:

1. Sofortige Annahmebestätigung (`status:"accepted"`)
2. Abschließende Antwort nach Abschluss (`status:"ok"|"error"`), dazwischen mit gestreamten `agent`-Ereignissen.

Die vollständige Protokolldokumentation finden Sie unter [Gateway-Protokoll](/de/gateway/protocol).

## Betriebsprüfungen

### Erreichbarkeit

- Öffnen Sie eine WS-Verbindung und senden Sie `connect`.
- Erwarten Sie eine `hello-ok`-Antwort mit einer Momentaufnahme.

### Bereitschaft

```bash
openclaw gateway status
openclaw channels status --probe
openclaw health
```

### Wiederherstellung nach Lücken

Ereignisse werden nicht erneut wiedergegeben. Aktualisieren Sie bei Sequenzlücken den Zustand (`health`, `system-presence`), bevor Sie fortfahren.

## Häufige Fehlersignaturen

| Signatur                                                       | Wahrscheinliches Problem                                                          |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `refusing to bind gateway ... without auth`                    | Bindung außerhalb der Loopback-Schnittstelle ohne gültigen Gateway-Authentifizierungspfad |
| `another gateway instance is already listening` / `EADDRINUSE` | Portkonflikt                                                                      |
| `Gateway start blocked: set gateway.mode=local`                | Konfiguration ist auf den Remote-Modus eingestellt oder `gateway.mode` fehlt in einer beschädigten Konfiguration |
| `unauthorized` während des Verbindungsaufbaus                  | Nicht übereinstimmende Authentifizierung zwischen Client und Gateway              |

Vollständige Diagnoseabläufe finden Sie unter [Gateway-Fehlerbehebung](/de/gateway/troubleshooting).

## Sicherheitsgarantien

- Gateway-Protokollclients brechen sofort ab, wenn der Gateway nicht verfügbar ist (kein impliziter Fallback auf einen direkten Kanal).
- Ungültige erste Frames beziehungsweise erste Frames, die keine Verbindungsanforderung enthalten, werden abgelehnt und geschlossen.
- Beim ordnungsgemäßen Herunterfahren wird vor dem Schließen des Sockets ein `shutdown`-Ereignis ausgegeben.

## Verwandte Themen

- [Konfiguration](/de/gateway/configuration)
- [Gateway-Fehlerbehebung](/de/gateway/troubleshooting)
- [Hintergrundprozess](/de/gateway/background-process)
- [Systemzustand](/de/gateway/health)
- [Doctor](/de/gateway/doctor)
- [Authentifizierung](/de/gateway/authentication)
- [Remote-Zugriff](/de/gateway/remote)
- [Geheimnisverwaltung](/de/gateway/secrets)
