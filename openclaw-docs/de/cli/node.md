---
read_when:
    - Ausführen des Headless-Node-Hosts
    - Koppeln eines Nicht-macOS-Node für system.run
summary: CLI-Referenz für `openclaw node` (Headless-Node-Host)
title: Node
x-i18n:
    generated_at: "2026-07-26T17:45:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 341539d05545ddcbf6175c34af7dca49332ba55906283b9933b9c9b1732c0e4d
    source_path: cli/node.md
    workflow: 16
---

# `openclaw node`

Führen Sie einen **Headless-Node-Host** aus, der eine Verbindung zum Gateway-WebSocket herstellt und
`system.run` / `system.which` auf diesem Rechner bereitstellt.

Unter macOS bettet die Menüleisten-App diese Node-Host-Laufzeit bereits in ihre eigene
Node-Verbindung ein und ergänzt native Mac-Funktionen. Verwenden Sie `openclaw node run` auf einem
Mac nur, wenn Sie bewusst einen Headless-Node ohne die App verwenden möchten. Werden
beide ausgeführt, entstehen zwei Node-Identitäten für denselben Rechner.

## Warum einen Node-Host verwenden?

Verwenden Sie einen Node-Host, wenn Agenten **Befehle auf anderen Rechnern** in Ihrem
Netzwerk ausführen sollen, ohne dort eine vollständige macOS-Begleit-App zu installieren.

Häufige Anwendungsfälle:

- Befehle auf entfernten Linux-/Windows-Rechnern ausführen (Build-Server, Laborrechner, NAS).
- Die Ausführung auf dem Gateway weiterhin **in einer Sandbox** isolieren, genehmigte Ausführungen jedoch an andere Hosts delegieren.
- Ein schlankes, headless Ausführungsziel für Automatisierung oder CI-Nodes bereitstellen.

Die Ausführung wird auf dem Node-Host weiterhin durch **Ausführungsgenehmigungen** und
agentenspezifische Positivlisten geschützt, sodass der Befehlszugriff begrenzt und explizit bleibt.

`openclaw node run` kann nach dem Verbindungsaufbau Plugin- oder MCP-gestützte Tools veröffentlichen.
Das Gateway vertraut standardmäßig den Deskriptoren des gekoppelten Nodes, verlangt jedoch,
dass der Befehl jedes Deskriptors innerhalb der genehmigten Befehlsoberfläche des Nodes bleibt. Der
Agent sieht jeden akzeptierten Deskriptor als normales Plugin-Tool, die Ausführung erfolgt jedoch weiterhin
über `node.invoke`. Wird die Verbindung zum Node getrennt, steht das Tool daher bei neuen
Agentenausführungen nicht mehr zur Verfügung. Gateway-Betreiber können die Veröffentlichung mit
`gateway.nodes.pluginTools.enabled: false` deaktivieren.

Fügen Sie für deklarative MCP-Tools die normale MCP-Serverstruktur unter
`nodeHost.mcp.servers` in `openclaw.json` auf dem Node-Rechner hinzu und starten Sie anschließend den
Node-Host neu. Der Node deklariert die genehmigungspflichtige Befehlsfamilie
`mcp.tools.call.v1` und veröffentlicht die aufgeführten Tools nach dem Verbindungsaufbau. Eine spätere
Änderung der Serverliste erfordert keine erneute Kopplung. Siehe
[Auf dem Node gehostete MCP-Server](/de/nodes#node-hosted-mcp-servers).

## Browser-Proxy (ohne Konfiguration)

Node-Hosts geben automatisch einen Browser-Proxy bekannt, sofern `browser.enabled` auf dem
Node nicht deaktiviert ist. Dadurch kann der Agent ohne zusätzliche Konfiguration
Browserautomatisierung auf diesem Node verwenden.

Standardmäßig stellt der Proxy die normale Browserprofiloberfläche des Nodes bereit. Wenn Sie
`nodeHost.browserProxy.allowProfiles` festlegen, wird der Proxy restriktiv:
Die Auswahl von Profilen, die nicht auf der Positivliste stehen, wird abgelehnt, und Routen zum
Erstellen oder Löschen persistenter Profile werden über den Proxy blockiert.

Deaktivieren Sie ihn bei Bedarf auf dem Node:

```json5
{
  nodeHost: {
    browserProxy: {
      enabled: false,
    },
  },
}
```

## Ausführen (Vordergrund)

```bash
openclaw node run --host <gateway-host> --port 18789
```

Optionen:

- `--host <host>`: Gateway-WebSocket-Host (Standard: `127.0.0.1`)
- `--port <port>`: Gateway-WebSocket-Port (Standard: `18789`)
- `--context-path <path>`: Kontextpfad des Gateway-WebSockets (z. B. `/openclaw-gw`). Wird an die WebSocket-URL angehängt.
- `--tls`: TLS für die Gateway-Verbindung verwenden
- `--no-tls`: Eine unverschlüsselte Gateway-Verbindung erzwingen, selbst wenn TLS in der lokalen Gateway-Konfiguration aktiviert ist
- `--tls-fingerprint <sha256>`: Erwarteter Fingerabdruck des TLS-Zertifikats (sha256)
- `--node-id <id>`: Die in der gemeinsamen SQLite-Zustandsdatenbank gespeicherte Clientinstanz-ID überschreiben (setzt die Kopplung nicht zurück)
- `--display-name <name>`: Anzeigenamen des Nodes überschreiben

## Gateway-Authentifizierung für den Node-Host

`openclaw node run` und `openclaw node install` beziehen die Gateway-Authentifizierung aus der Konfiguration bzw. aus Umgebungsvariablen (keine Flags `--token`/`--password` für Node-Befehle):

- `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD` werden zuerst geprüft.
- Danach folgt die lokale Konfiguration als Rückfalloption: `gateway.auth.token` / `gateway.auth.password`.
- Im lokalen Modus übernimmt der Node-Host absichtlich nicht `gateway.remote.token` / `gateway.remote.password`.
- Wenn `gateway.auth.token` / `gateway.auth.password` explizit über SecretRef konfiguriert ist und nicht aufgelöst werden kann, schlägt die Auflösung der Node-Authentifizierung sicher fehl (keine Verschleierung durch eine entfernte Rückfalloption).
- In `gateway.mode=remote` kommen gemäß den Prioritätsregeln für entfernte Verbindungen auch entfernte Clientfelder (`gateway.remote.token` / `gateway.remote.password`) infrage.
- Die Authentifizierungsauflösung des Node-Hosts berücksichtigt ausschließlich `OPENCLAW_GATEWAY_*`-Umgebungsvariablen.

Für einen Node, der sich mit einem unverschlüsselten `ws://`-Gateway verbindet, werden Loopback-Adressen, private
IP-Literale, `.local` und Tailnet-Hosts vom Typ `*.ts.net` akzeptiert. Legen Sie für andere
vertrauenswürdige private DNS-Namen `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` fest. Andernfalls schlägt der
Node-Start sicher fehl und fordert Sie auf, `wss://`, einen SSH-Tunnel oder
Tailscale zu verwenden. Dies ist eine Aktivierung über die Prozessumgebung und kein
`openclaw.json`-Konfigurationsschlüssel.
`openclaw node install` übernimmt die Einstellung in den überwachten Node-Dienst, wenn sie
in der Umgebung des Installationsbefehls vorhanden ist.

## Dienst (Hintergrund)

Installieren Sie einen Headless-Node-Host als Benutzerdienst (launchd unter macOS, systemd unter
Linux, Windows-Aufgabenplanung unter Windows).

```bash
openclaw node install --host <gateway-host> --port 18789
```

Optionen:

- `--host <host>`: Gateway-WebSocket-Host (Standard: `127.0.0.1`)
- `--port <port>`: Gateway-WebSocket-Port (Standard: `18789`)
- `--context-path <path>`: Kontextpfad des Gateway-WebSockets (z. B. `/openclaw-gw`). Wird an die WebSocket-URL angehängt.
- `--tls`: TLS für die Gateway-Verbindung verwenden
- `--tls-fingerprint <sha256>`: Erwarteter Fingerabdruck des TLS-Zertifikats (sha256)
- `--node-id <id>`: Die in der gemeinsamen SQLite-Zustandsdatenbank gespeicherte Clientinstanz-ID überschreiben (setzt die Kopplung nicht zurück)
- `--display-name <name>`: Anzeigenamen des Nodes überschreiben
- `--runtime <runtime>`: Dienstlaufzeit (`node`)
- `--force`: Erneut installieren/überschreiben, falls bereits installiert

Dienst verwalten:

```bash
openclaw node status
openclaw node start
openclaw node stop
openclaw node restart
openclaw node uninstall
```

Verwenden Sie `openclaw node run` für einen Node-Host im Vordergrund (kein Dienst).

Dienstbefehle akzeptieren `--json` für eine maschinenlesbare Ausgabe.

Der Node-Host wiederholt Gateway-Neustarts und netzwerkbedingte Verbindungsabbrüche innerhalb des Prozesses. Wenn das
Gateway eine endgültige Unterbrechung wegen Token-, Passwort- oder Bootstrap-Authentifizierung meldet, protokolliert der Node-Host
die Details zum Verbindungsabbruch und wird mit einem Fehlercode ungleich null beendet, sodass launchd/systemd/die Aufgabenplanung ihn
mit aktueller Konfiguration und aktuellen Anmeldedaten neu starten kann. Unterbrechungen aufgrund einer erforderlichen Kopplung verbleiben im
Vordergrundablauf, damit die ausstehende Anfrage genehmigt werden kann.

## Kopplung

Bei der ersten Verbindung wird auf dem Gateway eine ausstehende Anfrage zur Gerätekopplung (`role: node`) erstellt.

Wenn der Gateway-Host nicht interaktiv per SSH auf den Node-Host zugreifen kann (gleicher Benutzer,
vertrauenswürdiger Hostschlüssel), wird die ausstehende Anfrage automatisch genehmigt: Das Gateway
führt `openclaw node identity --json` per SSH auf dem Node-Host aus und erteilt die Genehmigung bei
exakter Übereinstimmung des Geräteschlüssels. Dies ist standardmäßig aktiviert. Unter
[SSH-verifizierte automatische Genehmigung von Geräten](/de/gateway/pairing#ssh-verified-device-auto-approval-default)
finden Sie die Voraussetzungen und Informationen zum Deaktivieren (`gateway.nodes.pairing.sshVerify: false`).

Andernfalls genehmigen Sie die Anfrage manuell über:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Prüfen Sie die lokale Node-Identität, die das Gateway verifiziert:

```bash
openclaw node identity --json
```

Der Befehl gibt die Geräte-ID und den öffentlichen Schlüssel aus der Zeile `primary` in
`state/openclaw.sqlite` aus und erstellt niemals die Datenbank oder eine neue Identität.

In streng kontrollierten Node-Netzwerken kann der Gateway-Betreiber ausdrücklich die
automatische Genehmigung der erstmaligen Node-Kopplung aus vertrauenswürdigen CIDRs aktivieren:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

Dies ist standardmäßig deaktiviert (`autoApproveCidrs` ist nicht festgelegt). Es gilt nur für eine
neue `role: node`-Kopplung ohne angeforderte Geltungsbereiche von einer Client-IP, der das
Gateway vertraut. Betreiber-/Browserclients, Control UI, WebChat sowie Aktualisierungen von Rolle,
Geltungsbereich, Metadaten oder öffentlichem Schlüssel erfordern weiterhin eine manuelle Genehmigung.

Wenn der Node die Kopplung mit geänderten Authentifizierungsdetails (Rolle/Geltungsbereiche/öffentlicher Schlüssel)
erneut versucht, wird die vorherige ausstehende Anfrage ersetzt und eine neue `requestId` erstellt.
Führen Sie vor der Genehmigung `openclaw devices list` erneut aus.

### Identitäts- und Kopplungszustand

Der Headless-Node trennt seine Clientinstanz-ID von der signierten Geräteidentität,
die das Gateway für Kopplung und Routing verwendet. Dieser Zustand befindet sich im
OpenClaw-Zustandsverzeichnis (standardmäßig `~/.openclaw` oder `$OPENCLAW_STATE_DIR`,
falls festgelegt):

| Zustand                                                  | Zweck                                                                                                                             |
| -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `state/openclaw.sqlite` (`node_host_config`)             | Clientinstanz-ID, Anzeigename und Gateway-Verbindungsmetadaten. Der Client sendet diese ID als `instanceId`.                     |
| `state/openclaw.sqlite` (`device_identities`, `primary`) | Signiertes Ed25519-Schlüsselpaar und daraus abgeleitete Geräte-ID. Bei signierten Verbindungen dient diese Geräte-ID als geroutete Node-ID und Kopplungsidentität. |
| `state/openclaw.sqlite` (`device_auth_tokens`)           | Token gekoppelter Geräte, nach kryptografischer Geräte-ID und Rolle verschlüsselt.                                                 |

`--node-id` ändert ausschließlich die Clientinstanz-ID im gemeinsamen SQLite-Zustand. Die
kryptografische Geräte-ID wird nicht geändert und die Kopplungsauthentifizierung nicht gelöscht. Auch die Migration einer veralteten
`node.json` mit `openclaw doctor --fix` setzt die Kopplung nicht zurück. So
widerrufen Sie einen Node und koppeln ihn erneut:

1. Führen Sie auf dem Gateway `openclaw nodes remove --node <id|name|ip>` aus.
2. Starten Sie auf dem Node den installierten Dienst mit `openclaw node restart` neu oder
   halten Sie ihn an und führen Sie den Vordergrundbefehl `openclaw node run` erneut aus. Dadurch wird der
   Ablauf zur Gerätekopplung gestartet. Wenn `openclaw devices list` keine Anfrage anzeigt
   und der Node `AUTH_DEVICE_TOKEN_MISMATCH` meldet, starten Sie ihn neu oder führen Sie ihn
   noch einmal aus. Der abgelehnte Versuch löscht das nun widerrufene lokale Token; beim nächsten
   Versuch kann die Kopplung angefordert werden.
3. Führen Sie auf dem Gateway `openclaw devices list` und anschließend
   `openclaw devices approve <deviceRequestId>` aus.
4. Starten Sie den Node erneut oder führen Sie ihn noch einmal aus. Ein zur Kopplung angehaltener Client wird
   nach der Genehmigung nicht automatisch fortgesetzt. Durch diese erneute Verbindung wird die separate
   Anfrage für die Befehlsoberfläche erstellt.
5. Führen Sie auf dem Gateway `openclaw nodes pending` und anschließend
   `openclaw nodes approve <nodeRequestId>` aus.

Die beiden Anfrage-IDs sind unterschiedlich. Eine anwendbare Richtlinie für vertrauenswürdige CIDRs kann
die erstmalige Gerätekopplung automatisch genehmigen. Die Genehmigung der Befehlsoberfläche bleibt
eine separate Prüfung.

Ältere OpenClaw-Versionen speicherten den Node-Host-Zustand in `node.json`, die signierte
Identität in `identity/device.json` und die gekoppelte Authentifizierung in
`identity/device-auth.json`. Halten Sie den Node-Host an und führen Sie
`openclaw doctor --fix` einmal aus. Doctor beansprucht jede veraltete Quelle, validiert sie,
importiert und überprüft die kanonische SQLite-Zeile und entfernt anschließend die alte Datei. Normale
Node-Befehle schlagen mit dieser Reparaturanweisung sicher fehl, solange eine veraltete Datei
oder ein unterbrochener Doctor-Anspruch vorhanden ist. Halten Sie `state/openclaw.sqlite` geheim;
die Datei enthält das Geräteschlüsselpaar und die Authentifizierungstoken.

## Ausführungsgenehmigungen

`system.run` wird durch lokale Ausführungsgenehmigungen geschützt:

- `$OPENCLAW_STATE_DIR/exec-approvals.json` oder
  `~/.openclaw/exec-approvals.json`, wenn die Variable nicht festgelegt ist
- [Ausführungsgenehmigungen](/de/tools/exec-approvals)
- `openclaw approvals --node <id|name|ip>` (vom Gateway aus bearbeiten)

Für die genehmigte asynchrone Node-Ausführung bereitet OpenClaw vor der Abfrage einen kanonischen
`systemRunPlan` vor. Die später genehmigte Weiterleitung `system.run` verwendet diesen gespeicherten
Plan erneut. Änderungen an Befehls-, Arbeitsverzeichnis- oder Sitzungsfeldern, nachdem die Genehmigungsanfrage
erstellt wurde, werden daher abgelehnt, statt die vom Node ausgeführte Aktion zu ändern.

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Nodes](/de/nodes)
