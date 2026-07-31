---
read_when:
    - iOS-/watchOS-/Android-Nodes mit einem Gateway koppeln
    - Node-Canvas/Kamera für den Agentenkontext verwenden
    - Neue Node-Befehle oder CLI-Hilfsfunktionen hinzufügen
summary: 'Nodes: Kopplung, Funktionen, Berechtigungen und CLI-Hilfsprogramme für Canvas/Kamera/Bildschirm/Gerät/Benachrichtigungen/System'
title: Nodes
x-i18n:
    generated_at: "2026-07-26T18:32:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b4f7c80491d713777e1ba5b8f55c88bd9fa48be602b504e6ac6ba00cd12a4313
    source_path: nodes/index.md
    workflow: 16
---

Ein **Node** ist ein Begleitgerät (macOS/iOS/watchOS/Android/headless), das sich über `role: "node"` mit dem Gateway verbindet und über `node.invoke` eine Befehlsoberfläche (z. B. `canvas.*`, `camera.*`, `device.*`, `notifications.*`, `system.*`) bereitstellt. Die meisten Nodes verwenden den Gateway-WebSocket am Operator-Port. Der optionale direkte Apple-Watch-Node verwendet signierte HTTPS-Abfragen am selben Port, da watchOS gewöhnlichen Apps generische Low-Level-Netzwerkverbindungen untersagt. Protokolldetails: [Gateway-Protokoll](/de/gateway/protocol).

Veralteter Transport: [Bridge-Protokoll](/de/gateway/bridge-protocol) (TCP JSONL; für aktuelle Nodes nur von historischem Interesse).

macOS kann auch im **Node-Modus** ausgeführt werden: Die Menüleisten-App verbindet sich als ein Node mit dem
WS-Server des Gateways (sodass `openclaw nodes …` für diesen Mac funktioniert). Die App
fügt der gleichen, von `openclaw node run` verwendeten Node-Host-Befehlsoberfläche native Befehle
für Canvas, Kamera, Bildschirm, Benachrichtigungen und Computersteuerung hinzu. Starten Sie auf
diesem Mac keinen zweiten CLI-Node; die App führt die entsprechende CLI-Node-Host-Laufzeit als
internen Worker aus und bleibt die einzige Gateway-Verbindung und Node-Identität.

Nodes sind **Peripheriegeräte**, keine Gateways: Sie führen den Gateway-Dienst nicht aus, und Kanalnachrichten (Telegram, WhatsApp usw.) gehen beim Gateway ein, nicht bei Nodes.

Runbook zur Fehlerbehebung: [/nodes/troubleshooting](/de/nodes/troubleshooting)

## Kopplung und Status

Nodes verwenden die **Gerätekopplung**. Ein Node präsentiert beim Verbindungsaufbau eine signierte Geräteidentität; das Gateway erstellt eine Gerätekopplungsanfrage für `role: node`. Genehmigen Sie sie über die Geräte-CLI (oder die Benutzeroberfläche). Die direkte Einrichtung der Apple Watch verwendet einen vom Administrator ausgestellten, kurzlebigen und ausschließlich für Nodes bestimmten Einrichtungscode, um ihre feste Befehlsoberfläche mit geringem Risiko zu genehmigen; eine spätere Erweiterung der Fähigkeiten erfordert weiterhin eine reguläre Genehmigung.

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
```

Ausstehende Kopplungsanfragen laufen 5 Minuten nach dem letzten Wiederholungsversuch des Geräts ab – ein Gerät, das fortlaufend versucht, die Verbindung wiederherzustellen, hält seine eine ausstehende Anfrage (und `requestId`) aktiv, anstatt alle paar Minuten eine neue Aufforderung zu erzeugen; den vollständigen Lebenszyklus von Anfrage und Genehmigung finden Sie unter [Node-Kopplung](/de/gateway/pairing). Wenn ein Node den Versuch mit geänderten Authentifizierungsdetails (Rolle/Bereiche/öffentlicher Schlüssel) wiederholt, wird die vorherige ausstehende Anfrage ersetzt und eine neue `requestId` erstellt – Clients erhalten für die ersetzte Anfrage ein `device.pair.resolved`-Ereignis, und Sie sollten `openclaw devices list` vor der Genehmigung erneut ausführen.

- `nodes status` kennzeichnet einen Node als **gekoppelt**, wenn seine Gerätekopplungsrolle `node` umfasst.
- Ein verbundener nativer Mac kann unter
  **Settings -> Permissions -> Active computer detection** die zusammengefasste Aktivität physischer Eingaben aktivieren. Bedienungshilfen sind
  ebenfalls erforderlich. Das Gateway kennzeichnet den aktuellsten geeigneten Mac als
  `active`, gibt dem Agenten einen stabilen Hinweis auf die Node-ID und leitet Warnungen zu Node-Verbindungen
  dorthin weiter, bevor ein verzögerter Fallback erfolgt. Informationen zu Einrichtung, Datenschutz, Zeitsteuerung und
  Fehlerbehebung finden Sie unter
  [Präsenz des aktiven Computers](/de/nodes/presence).
- Der Gerätekopplungsdatensatz ist der dauerhafte Vertrag für die genehmigte Rolle. Die Token-Rotation bleibt innerhalb dieses Vertrags; sie kann einen gekoppelten Node nicht auf eine Rolle hochstufen, die durch die Kopplungsgenehmigung nie gewährt wurde.
- `node.pair.*` (CLI: `openclaw nodes pending/approve/reject/remove/rename`) ist ein separater, vom Gateway verwalteter Node-Kopplungsspeicher, der die genehmigte Befehls-/Fähigkeitsoberfläche des Nodes über erneute Verbindungen hinweg verfolgt. Er steuert **nicht** die Transportauthentifizierung – dafür ist die Gerätekopplung zuständig.
- `openclaw nodes remove --node <id|name|ip>` entfernt eine Node-Kopplung. Bei einem gerätegestützten Node widerruft dies die Rolle `node` des Geräts im Speicher gekoppelter Geräte und trennt die Sitzungen dieses Geräts mit Node-Rolle: Ein Gerät mit gemischten Rollen behält seinen Datensatz und verliert nur die Rolle `node`, während der Datensatz eines Geräts, das ausschließlich als Node dient, gelöscht wird. Außerdem wird jeder passende Eintrag aus dem separaten Node-Kopplungsspeicher entfernt. `operator.pairing` kann Nicht-Operator-Node-Datensätze auf anderen Geräten entfernen; ein Aufrufer mit Geräte-Token, der seine eigene Node-Rolle auf einem Gerät mit gemischten Rollen widerruft, benötigt zusätzlich `operator.admin`.
- Der Genehmigungsumfang richtet sich nach den in der ausstehenden Anfrage deklarierten Befehlen:
  - Anfrage ohne Befehle: `operator.pairing`
  - Node-Befehle ohne Ausführung: `operator.pairing` + `operator.write`
  - `system.run` / `system.run.prepare` / `system.which`: `operator.pairing` + `operator.admin`

## Versionsabweichung und Aktualisierungsreihenfolge

Der Gateway-WebSocket akzeptiert authentifizierte Node-Clients innerhalb eines N-1-Protokollfensters.
Das aktuelle v4-Gateway akzeptiert daher v3-Nodes, wenn die Verbindung sowohl
`role: "node"` als auch `client.mode: "node"` deklariert. Operator- und UI-Sitzungen müssen
weiterhin das aktuelle Protokoll verwenden.

Aktualisieren Sie bei gestaffelten Flottenaktualisierungen zuerst das Gateway und anschließend jeden Node.
Ein N-1-Node bleibt während seiner Aktualisierung sichtbar und verwaltbar; das Gateway
protokolliert `legacy node protocol accepted` mit einer Aktualisierungsempfehlung. Kopplung,
Geräteauthentifizierung, Befehlsfreigabelisten und Ausführungsgenehmigungen gelten weiterhin.
Plugin-eigene Fähigkeiten und Befehle bleiben verborgen, bis der Node auf
das aktuelle Protokoll aktualisiert wurde. Nodes, die älter als N-1 sind, benötigen vor
dem erneuten Verbindungsaufbau eine Out-of-Band-Aktualisierung.

Der direkte watchOS-HTTPS-Transport erfordert die aktuelle Protokollversion; aktualisieren Sie
die Watch-App zusammen mit dem Gateway, bevor Sie den direkten Modus aktivieren.

## Entfernter Node-Host (system.run)

Verwenden Sie einen **Node-Host**, wenn Ihr Gateway auf einem Rechner ausgeführt wird und Befehle auf einem anderen ausgeführt werden sollen. Das Modell kommuniziert weiterhin mit dem **Gateway**; das Gateway leitet `exec`-Aufrufe an den **Node-Host** weiter, wenn `host=node` ausgewählt ist.

| Rolle        | Verantwortlichkeit                                                   |
| ------------ | -------------------------------------------------------------------- |
| Gateway-Host | Empfängt Nachrichten, führt das Modell aus und leitet Tool-Aufrufe weiter. |
| Node-Host    | Führt `system.run`/`system.which` auf dem Node-Rechner aus. |
| Genehmigungen | Werden auf dem Node-Host über `~/.openclaw/exec-approvals.json` durchgesetzt. |

Hinweis zur Genehmigung:

- Genehmigungspflichtige Node-Ausführungen werden an den exakten Anfragekontext gebunden. Der Ausführungspfad bereitet vor der Genehmigung ein kanonisches `systemRunPlan` vor; nach der Genehmigung leitet das Gateway diesen gespeicherten Plan weiter, nicht etwa später vom Aufrufer bearbeitete Befehls-, Arbeitsverzeichnis- oder Sitzungsfelder, und validiert das Arbeitsverzeichnis vor der Ausführung erneut.
- Bei direkten Datei-Ausführungen durch Shells/Laufzeiten bindet OpenClaw nach bestem Bemühen außerdem einen konkreten lokalen Dateioperanden und verweigert die Ausführung, wenn diese Datei vor der Ausführung geändert wird.
- Wenn OpenClaw für einen Interpreter-/Laufzeitbefehl nicht genau eine konkrete lokale Datei identifizieren kann, wird die genehmigungspflichtige Ausführung verweigert, anstatt eine vollständige Laufzeitabdeckung vorzutäuschen. Verwenden Sie für umfassendere Interpreter-Semantik Sandboxing, separate Hosts oder eine explizite vertrauenswürdige Freigabeliste bzw. einen vollständigen Workflow.

### Node-Host starten (Vordergrund)

Auf dem Node-Rechner:

```bash
openclaw node run --host <gateway-host> --port 18789 --display-name "Build Node"
```

`node run` akzeptiert außerdem `--context-path` (Gateway-WS-Kontextpfad), `--tls`, `--tls-fingerprint <sha256>` und `--node-id` (überschreibt die veraltete Client-Instanz-ID; die Kopplung wird dadurch nicht zurückgesetzt). Übergeben Sie unter macOS `--share-installed-apps`, um `device.apps` anzukündigen; die Freigabe ist standardmäßig deaktiviert. Verwenden Sie `--no-share-installed-apps`, um eine zuvor gespeicherte Aktivierung zu deaktivieren.

### Entferntes Gateway über SSH-Tunnel (Loopback-Bindung)

Wenn das Gateway an Loopback gebunden ist (`gateway.bind=loopback`, Standard im lokalen Modus), können sich entfernte Node-Hosts nicht direkt verbinden. Erstellen Sie einen SSH-Tunnel und richten Sie den Node-Host auf das lokale Ende des Tunnels.

Beispiel (Node-Host -> Gateway-Host):

```bash
# Terminal A (weiterlaufen lassen): lokales 18790 an Gateway 127.0.0.1:18789 weiterleiten
ssh -N -L 18790:127.0.0.1:18789 user@gateway-host

# Terminal B: Gateway-Token exportieren und über den Tunnel verbinden
export OPENCLAW_GATEWAY_TOKEN="<gateway-token>"
openclaw node run --host 127.0.0.1 --port 18790 --display-name "Build Node"
```

Hinweise:

- `openclaw node run` unterstützt Token- oder Passwortauthentifizierung.
- Umgebungsvariablen werden bevorzugt: `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`.
- Der Konfigurations-Fallback ist `gateway.auth.token` / `gateway.auth.password`.
- Im lokalen Modus ignoriert der Node-Host absichtlich `gateway.remote.token` / `gateway.remote.password`.
- Im entfernten Modus kommen `gateway.remote.token` / `gateway.remote.password` gemäß den Prioritätsregeln für entfernte Verbindungen infrage.
- Wenn aktive lokale `gateway.auth.*`-SecretRefs konfiguriert, aber nicht aufgelöst sind, schlägt die Node-Host-Authentifizierung sicher geschlossen fehl.
- Die Auflösung der Node-Host-Authentifizierung berücksichtigt nur `OPENCLAW_GATEWAY_*`-Umgebungsvariablen.

### Node-Host starten (Dienst)

```bash
openclaw node install --host <gateway-host> --port 18789 --display-name "Build Node"
openclaw node start
openclaw node restart
```

`node install` akzeptiert außerdem `--context-path`, `--tls`, `--tls-fingerprint`, `--node-id` (nur veraltete Client-Instanz-ID), `--share-installed-apps` / `--no-share-installed-apps`, `--runtime <node>` (Standard: Node) und `--force` zur Neuinstallation. `node status`, `node stop` und `node uninstall` sind ebenfalls verfügbar.

### Koppeln und benennen

Auf dem Gateway-Host:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Wenn der Node den Versuch mit geänderten Authentifizierungsdetails wiederholt, führen Sie `openclaw devices list` erneut aus und genehmigen Sie die aktuelle `requestId`.

Benennungsoptionen:

- `--display-name` für `openclaw node run` / `openclaw node install` (wird in der gemeinsamen `node_host_config`-SQLite-Zeile zusammen mit der Client-Instanz-ID und den Metadaten der Gateway-Verbindung gespeichert).
- `openclaw nodes rename --node <id|name|ip> --name "Build Node"` (Gateway-Überschreibung).

### Auf dem Node gehostete MCP-Server

Konfigurieren Sie MCP-Server in `openclaw.json` auf dem Node-Rechner, nicht auf dem
Gateway:

```json5
{
  nodeHost: {
    mcp: {
      servers: {
        localDocs: {
          command: "npx",
          args: ["-y", "@modelcontextprotocol/server-filesystem", "/srv/docs"],
          toolFilter: {
            include: ["read_*", "search"],
          },
        },
        internalApi: {
          url: "https://mcp.internal.example/mcp",
          transport: "streamable-http",
          headers: {
            Authorization: "Bearer ${INTERNAL_MCP_TOKEN}",
          },
        },
      },
    },
  },
}
```

Der headless Node-Host startet diese Server, listet ihre Tools auf und veröffentlicht
die Deskriptoren nach dem Verbindungsaufbau. Tool-Aufrufe werden über
`mcp.tools.call.v1` an diesen Node zurückgeleitet; das Gateway benötigt weder eine entsprechende MCP-Konfiguration noch ein JS-
Plugin. OAuth-MCP-Server werden von diesem auf dem Node gehosteten v1-Pfad nicht unterstützt.

Aktuelle Node-Hosts deklarieren während ihrer ersten Kopplung die integrierte Befehlsfamilie `mcp.tools.call.v1`,
selbst wenn kein MCP-Server konfiguriert ist. Ein Node, der mit einer
älteren OpenClaw-Version gekoppelt wurde, kann nach der Aktualisierung des
Node-Hosts eine einmalige Erweiterung der Befehlsoberfläche anfordern. Das Hinzufügen, Entfernen oder Filtern von Servern danach erfordert
keine erneute Kopplung, da die genehmigte Befehlsfamilie unverändert bleibt. Starten Sie
`openclaw node run` oder `openclaw node restart` neu, um Änderungen an der Node-MCP-Konfiguration anzuwenden;
der Node-Host überwacht diese Konfiguration nicht.

Gateway-Operatoren können alle für Agenten sichtbaren Tools ignorieren, die von gekoppelten Nodes veröffentlicht werden,
einschließlich der auf Nodes gehosteten MCP-Tools, indem sie
`gateway.nodes.pluginTools.enabled: false` verwenden. Exakte Befehlsverweigerungen wie
`gateway.nodes.commands.deny: ["mcp.tools.call.v1"]` blockieren ebenfalls die Ausführung.

### Auf dem Node gehostete Skills

Installieren Sie Skills im aktiven OpenClaw-Skills-Verzeichnis des Node-Rechners,
standardmäßig `~/.openclaw/skills`. `OPENCLAW_HOME`, `OPENCLAW_STATE_DIR` und
`OPENCLAW_CONFIG_PATH` verschieben dieses aktive Profil. `OPENCLAW_STATE_DIR` hat
für Skills Vorrang; andernfalls befindet sich `skills/` neben dem von
`openclaw config file` ausgegebenen Pfad. Der Headless-Node-Host veröffentlicht nach
dem Verbindungsaufbau gültige `SKILL.md`-Dateien, und der Gateway fügt sie
nur so lange zu den Skill-Snapshots des Agenten hinzu, wie dieser Node verbunden
bleibt. Der Name jedes Skill-Verzeichnisses muss mit dem Frontmatter-Feld
`name` übereinstimmen, damit der abstrakte Node-Locator ohne ein
weiteres Protokollfeld genau einem Eintrag zugeordnet wird.

Die anfängliche Kopplung der Node-Rolle genehmigt die Veröffentlichung von
Skills. Das Hinzufügen, Entfernen oder Ändern von Skills erfordert weder eine
erneute Kopplung noch eine Änderung der Gateway-Konfiguration. Starten Sie
`openclaw node run` oder `openclaw node restart` nach Änderungen an den
Node-Skill-Dateien neu; der Node-Host überwacht das Skills-Verzeichnis nicht.

Auf dem Node gehostete Skill-Einträge identifizieren ihren Node und enthalten
ihren Ausführungsort. Skill-Dateien, referenzierte relative Pfade und
Binärdateien verbleiben auf diesem Node. Der Agent liest den angekündigten
`node://.../SKILL.md`-Speicherort mit dem normalen `read`-Tool.
`file_fetch` akzeptiert vom Betreiber genehmigte absolute Node-Pfade,
keine Node-Skill-Locators; Laufzeitumgebungen ohne das normale Lesetool können
stattdessen `cat SKILL.md` über `exec host=node node=<node-id>` ausführen und dabei das
angekündigte Verzeichnis `node://.../skills/<name>` als `workdir` verwenden.
Referenzierte Dateien und Binärdateien verwenden dasselbe Ausführungsziel und
Arbeitsverzeichnis. Der Node-Host löst diesen Locator anhand seines aktiven
OpenClaw-Zustandsverzeichnisses auf, sodass relative Pfade auf dem Node und nicht
auf dem Gateway-Rechner aufgelöst werden. Auf dem veröffentlichenden Node muss
`system.run` genehmigt sein, und die Ausführungsrichtlinie des Agenten muss
`host=node` zulassen; andernfalls wird der Skill nicht in den Snapshot
dieses Agenten aufgenommen.

Setzen Sie `nodeHost.skills.enabled: false` auf dem Node, um die Veröffentlichung zu
unterbinden. Gateway-Betreiber können Skills von allen gekoppelten Nodes mit
`gateway.nodes.allowSkills: false` ignorieren.

### Headless-Identitätszustand

Der Headless-Node verwaltet drei getrennte Zustandsdatensätze im gemeinsam
genutzten SQLite:

- `~/.openclaw/state/openclaw.sqlite` (`node_host_config`): die Clientinstanz-ID, den Anzeigenamen und die Gateway-Verbindungsmetadaten.
- `~/.openclaw/state/openclaw.sqlite` (`device_identities`, Schlüssel `primary`): das signierte Geräteschlüsselpaar und die daraus abgeleitete kryptografische Geräte-ID.
- `~/.openclaw/state/openclaw.sqlite` (`device_auth_tokens`): gekoppelte Geräte-Authentifizierungstoken, indiziert nach kryptografischer Geräte-ID und Rolle.

Für einen signierten Node verwendet der Gateway die kryptografische Geräte-ID
für die Kopplung und das Node-Routing. Die Clientinstanz-ID dient nur als
Verbindungsmetadatum. Eine Änderung von `--node-id` oder die Migration
eines außer Betrieb genommenen `node.json` setzt die Kopplung daher nicht
zurück. Informationen zum unterstützten Ablauf für Widerruf und erneute
Kopplung sowie Upgrade-Hinweise finden Sie unter
[Identitäts- und Kopplungszustand](/de/cli/node#identity-and-pairing-state).

Außer Betrieb genommene Dateien `identity/device.json` und
`identity/device-auth.json` sind Migrationseingaben unter der Zuständigkeit von Doctor.
Stoppen Sie den Node-Host und führen Sie `openclaw doctor --fix` aus; Doctor
importiert und überprüft ihre Zeilen in SQLite, bevor die alten Dateien entfernt
werden.

### Befehle zur Positivliste hinzufügen

Ausführungsgenehmigungen gelten **pro Node-Host**. Fügen Sie die
Positivlisteneinträge über den Gateway hinzu:

```bash
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/uname"
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/sw_vers"
```

Genehmigungen befinden sich auf dem Node-Host unter `~/.openclaw/exec-approvals.json`.

### Ausführung auf den Node ausrichten

Konfigurieren Sie die Standardwerte (Gateway-Konfiguration):

```bash
openclaw config set tools.exec.host node
openclaw config set tools.exec.mode allowlist
openclaw config set tools.exec.node "<id-or-name>"
```

Oder pro Sitzung:

```text
/exec host=node security=allowlist node=<id-or-name>
```

Nach der Konfiguration wird jeder `exec`-Aufruf mit
`host=node` auf dem Node-Host ausgeführt (vorbehaltlich der
Node-Positivliste und -Genehmigungen).

`host=auto` wählt den Node nicht implizit von selbst aus, aber eine
explizite `host=node`-Anforderung pro Aufruf ist aus
`auto` zulässig. Wenn die Node-Ausführung der Standard für die
Sitzung sein soll, setzen Sie `tools.exec.host=node` oder
`/exec host=node ...` explizit.

Verwandte Themen:

- [Node-Host-CLI](/de/cli/node)
- [Ausführungstool](/de/tools/exec)
- [Ausführungsgenehmigungen](/de/tools/exec-approvals)

### Lokale Modellinferenz

Ein Desktop- oder Server-Node kann chatfähige Modelle von einem auf diesem Node
ausgeführten Ollama-Server bereitstellen. Agenten verwenden das
`node_inference`-Tool des Ollama-Plugins, um installierte Modelle zu erkennen
und einen begrenzten Prompt remote auszuführen; der Gateway benötigt keinen
direkten Netzwerkzugriff auf Ollama. Einrichtung, Modellfilterung und Befehle
zur direkten Überprüfung finden Sie unter
[Node-lokale Ollama-Inferenz](/de/providers/ollama#node-local-inference).

### Codex-Sitzungen und -Transkripte

Das offizielle Plugin `codex` kann nicht archivierte Codex-Sitzungen
auf einem Headless-Node-Host oder nativen macOS-Node bereitstellen. Die
Katalogregistrierung hängt nicht mehr von `supervision.enabled` ab; diese Option
steuert die agentenseitigen Überwachungstools. Setzen Sie
`sessionCatalog.enabled: false` in der Codex-Plugin-Konfiguration, um den Betreiberkatalog und
die Katalogbefehle für gekoppelte Nodes zu deaktivieren, ohne den Provider oder
das Harness zu deaktivieren.
Das Plugin muss weiterhin auf beiden Computern aktiv sein, und die
Node-Einstellung bleibt eine lokale Einwilligung: Wenn sie nur auf dem Gateway
aktiviert wird, kann dieser den Codex-Zustand eines anderen Computers nicht
lesen.

Der Node kündigt die versionierten, schreibgeschützten Befehle
`codex.appServer.threads.list.v1` und
`codex.appServer.thread.turns.list.v1` an. Ein nativer Node-Host, auf dem die
Codex-CLI verfügbar ist, kündigt außerdem `codex.terminal.resume.v1` an. Genehmigen Sie
das Upgrade der Node-Kopplung, wenn diese Befehle erstmals erscheinen. Der
Gateway ruft sie über die normale Plugin-Node-Richtlinie auf und isoliert Fehler
nach Host.

Zeilen gekoppelter Nodes erscheinen als Gruppe **Codex** in der normalen
Sitzungsseitenleiste. Innerhalb jedes Hosts werden Zeilen standardmäßig nach
Projektordner gruppiert; ein Arbeitsverzeichnis unter `.claude/worktrees/<name>` wird
seinem Ursprungs-Repository zugeordnet, und Projektgruppen können wie andere
Bereiche der Seitenleiste eingeklappt werden. Verwenden Sie das Ordnersymbol in
der Kopfzeile des Katalogs, um die Projektgruppen aufzulösen oder
wiederherzustellen. Dieselbe Gruppierung gilt für den Katalog der
Claude-Sitzungen.
Standardmäßig öffnet die Auswahl einer Zeile den normalen Chat-Bereich und liest
das persistierte Transkript über begrenzte, cursorpaginierte
`thread/turns/list`-Aufrufe mit vollständiger Elementprojektion. Verwenden Sie das
Zeilenmenü, die Kopfzeile des Betrachters oder die Einstellung **Open Codex/Claude sessions in**, um `codex resume <thread-id>` im Betreiberterminal auf dem Computer zu starten, dem die Sitzung gehört. Der Terminalpfad des gekoppelten Nodes ist ein vom Codex-Plugin verwaltetes PTY-Relay mit Positivliste und keine beliebige Ausführung von Node-Befehlen.

Das Relay stellt nicht die vollständigen Fortsetzungs- und
Archivbesitzverträge des OpenClaw-Harness bereit. **Continue** und **Archive**
sind daher für Remote-Zeilen nicht verfügbar. Auf dem Gateway-Computer können
gespeicherte und inaktive Zeilen einen separaten, modellgebundenen Chat-Zweig
starten. Beide können erst archiviert werden, nachdem der Betreiber bestätigt
hat, dass kein anderer Codex-Client sie verwendet; die Live-Aktivität einer
gespeicherten Zeile bleibt unbekannt. Aktive Zeilen können weder verzweigt noch
archiviert werden.

Einrichtung, Paginierung, lokale Fortsetzung und die Sicherheitsgrenze für
Metadaten finden Sie unter
[Codex-Sitzungen überwachen](/de/plugins/codex-supervision).

### Claude-Sitzungen und -Transkripte

Das gebündelte Plugin `anthropic` erkennt standardmäßig nicht archivierte
Sitzungen von Claude CLI und Claude Desktop auf dem Gateway und gekoppelten
Nodes. Setzen Sie `plugins.entries.anthropic.config.sessionCatalog.enabled: false`, um den Betreiberkatalog und die
Katalogbefehle für gekoppelte Nodes zu deaktivieren, ohne Anthropic-Modelle oder
das Claude-CLI-Backend zu deaktivieren.
Ein Remote-macOS-App-Node kündigt
`anthropic.claude.sessions.list.v1` und `anthropic.claude.sessions.read.v1` an,
wenn das Anthropic-Plugin aktiviert ist und `~/.claude/projects/` vorhanden ist.
Genehmigen Sie das Upgrade der Node-Kopplung, wenn diese Befehle erstmals
erscheinen.

Ein nativer Node-Host, auf dem die Claude CLI verfügbar ist, kündigt außerdem
`anthropic.claude.terminal.resume.v1` an. Geeignete CLI- und Desktop-Zeilen können
`claude --resume <session-id>` im Betreiberterminal auf ihrem jeweiligen Host öffnen.
Dabei wird die native Sitzung übernommen; anders als bei der Übernahme durch
OpenClaw wird die Claude-Sitzung nicht zuvor verzweigt.

Der Katalog kombiniert gültige Projektindex-Datensätze der Claude CLI mit einem
begrenzten Metadaten-Fallback für nicht indizierte JSONL-Transkripte. Dieser
Fallback erkennt gleichzeitig ausgeführte interaktive Sitzungen ohne
Nebenstrang (`cli`) und Headless-Sitzungen der Agent-SDK-CLI
(`sdk-cli`). Die lokalen Metadaten von Claude Desktop liefern
Desktop-Titel und Archivstatus. Desktop-Metadaten haben Vorrang, wenn sich beide
Quellen auf dieselbe Claude-Code-Sitzungs-ID beziehen; reine CLI-Transkripte
bleiben sichtbar, da die CLI kein Archivierungskennzeichen besitzt.
Transkriptlesevorgänge verwenden undurchsichtige Byte-Offset-Cursor und
begrenztes rückwärtsgerichtetes Lesen von Dateien, sodass bei der Auswahl einer
großen Sitzung oder dem Laden einer älteren Seite nicht der gesamte
JSONL-Verlauf in eine einzige Gateway-Antwort eingelesen wird.

Die Auflistungs- und Lesebefehle sind schreibgeschützt. Sie stellen
Katalogmetadaten und Transkriptinhalte nur über die generischen Methoden
`sessions.catalog.list` und `sessions.catalog.read` für eine authentifizierte
Betreiberverbindung mit `operator.write` bereit. Eine Gateway-lokale Zeile der
Claude CLI kann aus dem normalen Chat-Eingabefeld übernommen werden: OpenClaw
importiert einen begrenzten sichtbaren Verlauf, setzt die Sitzung beim ersten
Turn mit `--fork-session` fort und lässt das Quelltranskript unverändert.

Ein Headless-Node-Host kann denselben Fortsetzungsablauf aktivieren:

```json5
{
  nodeHost: {
    agentRuns: {
      claude: { enabled: true },
    },
  },
}
```

Der Node kündigt `agent.cli.claude.run.v1` nur an, wenn diese Node-lokale Einstellung
aktiviert ist und die ausführbare Datei `claude` auf diesem Node
aufgelöst werden kann. Der Gateway kann sie nicht remote aktivieren. Der Befehl
unterliegt außerdem der bestehenden Ausführungsgenehmigungsrichtlinie des Nodes.
Wenn alle drei Claude-Befehle angekündigt und von der Node-Befehlsrichtlinie des
Gateways zugelassen werden, kann eine Zeile der Claude CLI auf diesem Node
fortgesetzt werden: OpenClaw importiert einen begrenzten Verlauf, bindet die
übernommene Sitzung an den Node und das vom Katalog gemeldete Arbeitsverzeichnis
und führt dort jeden einmaligen `claude -p`-Turn aus. Der erste Turn
verwendet weiterhin `--fork-session`, wodurch das Quelltranskript erhalten
bleibt.

Auf dem Node platzierte Turns verwenden die Claude-Standardwerte des Nodes. In
v1 erhalten sie weder die Gateway-Loopback-MCP-Konfiguration noch das
Gateway-Skills-Plugin, können nicht aus einem Gateway-Transkript neu initialisiert
werden und lehnen Anhänge und Bilder ab. Zeilen von Claude Desktop sowie Nodes,
die den Ausführungsbefehl nicht ankündigen, bleiben schreibgeschützt. Der
macOS-App-Node kündigt diesen Befehl noch nicht an, sodass seine Zeilen
schreibgeschützt bleiben.

Informationen zum Verhalten der Control UI und zu den Speicherquellen finden Sie
unter
[Anthropic: Claude-Sitzungen auf mehreren Computern](/de/providers/anthropic#claude-sessions-across-computers).

### OpenCode- und Pi-Sitzungen

Die gebündelten OpenCode- und ACPX-Plugins erkennen außerdem
schreibgeschützte native Sitzungskataloge auf dem Gateway und gekoppelten Nodes.
Ein Node kündigt `opencode.sessions.list.v1` / `opencode.sessions.read.v1` an, wenn die
`opencode`-CLI installiert ist, und
`acpx.pi.sessions.list.v1` / `acpx.pi.sessions.read.v1`, wenn das Sitzungsverzeichnis von Pi
vorhanden ist. Genehmigen Sie das Upgrade der Node-Kopplung, wenn neue Befehle
erstmals erscheinen. Wenn die entsprechende CLI ebenfalls verfügbar ist, fügt
der Node `opencode.terminal.resume.v1` oder `acpx.pi.terminal.resume.v1` hinzu; das vorhandene
Zeilenmenü und die Kopfzeile des Betrachters können die ausgewählte Sitzung dann
mit `opencode --session <id>` oder `pi --session <id>` erneut im zugehörigen Terminal
öffnen.

OpenCode liest über seine offizielle CLI-JSON-/Exportschnittstelle. Pi liest
seinen dokumentierten JSONL-Sitzungsspeicher, einschließlich projektbezogener
und globaler `settings.json`-Sitzungsverzeichnisse sowie der Überschreibungen
`PI_CODING_AGENT_DIR` und `PI_CODING_AGENT_SESSION_DIR`. Beide Kataloge sind standardmäßig
aktiviert; deaktivieren Sie sie in der Web-UI unter **Config > Plugins**.

Die Wiederaufnahme im Terminal verwendet das gespeicherte Arbeitsverzeichnis
der Sitzung und dasselbe Duplex-PTY-Relay mit Positivliste wie Codex und Claude.
Sie ermöglicht keine beliebige Ausführung von Node-Befehlen.

### Terminal-Dateiuploads

Die Control UI kann Dateien per Drag-and-drop in ein geöffnetes Terminal eines gekoppelten Node ziehen. Der native Node-Host kündigt den nur für Administratoren verfügbaren Befehl `terminal.upload` an; genehmigen Sie das Kopplungs-Upgrade, wenn es erstmals angezeigt wird. Jede Datei ist auf 16 MiB begrenzt, wird in einem privaten temporären Verzeichnis auf diesem Node bereitgestellt und als Shell-quotierter Pfad an das Terminal zurückgegeben, ohne sie auszuführen.

Das Einfügen von Pfaden unterstützt PowerShell, `cmd.exe` und erkannte POSIX-Shells (`sh`, Bash, Dash, Ash, Ksh, Zsh und Fish), einschließlich Git Bash unter Windows. Andere Shell-Überschreibungen werden abgelehnt, da deren Quotierungsregeln nicht sicher abgeleitet werden können; führen Sie den Node-Host innerhalb von WSL aus, um native WSL-Pfade zu verwenden. `cmd.exe`-Pfade, die `%` oder `!` enthalten, werden ebenfalls abgelehnt, da diese Shell jene Zeichen selbst innerhalb doppelter Anführungszeichen expandiert.

## Befehle aufrufen

Niedrige Ebene (roher RPC):

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command canvas.eval --params '{"javaScript":"location.href"}'
```

`nodes invoke` blockiert `system.run` und `system.run.prepare`; diese Befehle werden nur über das Tool `exec` mit `host=node` ausgeführt (siehe oben). Für die üblichen Abläufe, bei denen dem Agenten ein MEDIA-Anhang bereitgestellt wird, stehen Hilfsfunktionen auf höherer Ebene zur Verfügung (Canvas, Kamera, Bildschirm, Standort; siehe unten).

Lang laufende, streamende Node-Befehle verwenden additive `node.invoke.progress`-Ereignisse. Jedes Ereignis enthält die Aufruf-ID, eine nullbasierte Sequenznummer und ein
begrenztes UTF-8-Textfragment; der Gateway ordnet die Fragmente, bevor er sie an
den Aufrufer übermittelt. Die bestehende `node.invoke.result` bleibt die einzige abschließende
Antwort. Streamende Aufrufer können eine Inaktivitätsfrist festlegen, die mit dem
ersten Fortschrittsereignis beginnt und nach späteren Fortschrittsereignissen zurückgesetzt wird, während das
separate harte Zeitlimit des Aufrufs während Genehmigung und Ausführung bestehen bleibt. Ergebnis, hartes
Zeitlimit, Inaktivitätszeitlimit und Trennung des Node verwerfen jeweils ausstehenden Stream-
Status. Eine Abbrechung durch den Aufrufer gibt `node.invoke.cancel` aus; der Node-Host
beendet anschließend den zugehörigen Prozessbaum. Bestehende Anfrage-/Antwortbefehle bleiben unverändert.

## Befehlsrichtlinie

Node-Befehle müssen zwei Prüfungen bestehen, bevor sie aufgerufen werden können:

1. Der Node muss den Befehl in seinen authentifizierten Verbindungsmetadaten deklarieren (`connect.commands`).
2. Die aus Plattform und Genehmigung abgeleitete Positivliste des Gateway muss den deklarierten Befehl enthalten.

Standardmäßige Positivlisten nach Plattform (vor Plugin-Standardeinstellungen und Überschreibungen durch `commands.allow`/`commands.deny`):

| Plattform | Standardmäßig zulässige Befehle                                                                                                                                                                                                                                                                                           |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| iOS      | `camera.list`, `location.get`, `device.info`, `device.status`, `contacts.search`, `calendar.events`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer`, `system.notify`                                                                                                                        |
| watchOS  | `device.info`, `device.status`, `system.notify`                                                                                                                                                                                                                                                                       |
| Android  | `camera.list`, `location.get`, `notifications.list`, `notifications.actions`, `system.notify`, `device.info`, `device.status`, `device.permissions`, `device.health`, `device.apps`, `contacts.search`, `calendar.events`, `callLog.search`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer` |
| macOS    | `camera.list`, `location.get`, `device.info`, `device.status`, `device.apps`, `contacts.search`, `calendar.events`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer`, `system.notify`                                                                                                         |
| Windows  | `camera.list`, `location.get`, `device.info`, `device.status`, `system.notify`                                                                                                                                                                                                                                        |
| Linux    | `system.notify` (Node-Host-Befehle wie `system.run` unterliegen einer Genehmigung, siehe unten)                                                                                                                                                                                                                                  |

Diese Zeilen beschreiben die Obergrenze der Gateway-Richtlinie, nicht die von jeder Node-App implementierten Befehle. Ein Befehl kann nur verwendet werden, wenn der verbundene Node ihn ebenfalls deklariert. Insbesondere deklariert die aktuelle macOS-App die in der macOS-Richtlinienzeile aufgeführten Geräte- und personenbezogenen Datenfamilien nicht.

`canvas.*`-Befehle (`canvas.present`, `canvas.hide`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot`, `canvas.a2ui.*`) sind auf iOS, Android, macOS, Windows, Linux und unbekannten Plattformen eine Plugin-Standardeinstellung. Linux-Nodes deklarieren sie nur, wenn der lokale Canvas-Socket der Desktop-App vorhanden ist. Alle Canvas-Befehle sind unter iOS auf den Vordergrund beschränkt.

`talk.ptt.start`, `talk.ptt.stop`, `talk.ptt.cancel` und `talk.ptt.once` sind standardmäßig für jeden Node zulässig, der die Fähigkeit `talk` ankündigt oder `talk.*`-Befehle deklariert, unabhängig von der Plattformbezeichnung.

Desktop-Host-Befehle (`system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `mcp.tools.call.v1` und `screen.snapshot` unter macOS/Windows/Linux) sind nicht Bestandteil der obigen statischen Tabelle mit Plattformstandards. Sie werden verfügbar, sobald der Betreiber eine Kopplungsanfrage genehmigt, die sie deklariert. Anschließend werden sie bei erneuter Verbindung durch den genehmigten Befehlssatz des Node beibehalten.

Gefährliche oder datenschutzintensive Befehle erfordern weiterhin eine ausdrückliche Aktivierung mit `gateway.nodes.commands.allow`, selbst wenn ein Node sie deklariert: `camera.snap`, `camera.clip`, `screen.record`, `computer.act`, `contacts.add`, `calendar.add`, `reminders.add`, `health.summary`, `sms.send`, `sms.search`. `gateway.nodes.commands.deny` hat gegenüber Standardeinstellungen und zusätzlichen Positivlisteneinträgen immer Vorrang. Informationen zur Zustimmungssperre auf dem iPhone finden Sie unter [HealthKit-Zusammenfassungen](/de/platforms/ios-healthkit), Informationen zu den zusätzlichen Sperren für Fähigkeit, Tool-Richtlinie, Aktivierung und plattformspezifische Ausführung bei Desktop-Eingaben unter [Computernutzung](/de/nodes/computer-use).

Plugin-eigene Node-Befehle können eine Gateway-Richtlinie für Node-Aufrufe hinzufügen. Diese Richtlinie wird nach der Prüfung der Positivliste und vor der Weiterleitung an den Node ausgeführt, sodass rohe `node.invoke`-Aufrufe, CLI-Hilfsfunktionen und dedizierte Agenten-Tools dieselbe Plugin-Berechtigungsgrenze verwenden. Gefährliche Plugin-Node-Befehle erfordern weiterhin eine ausdrückliche Aktivierung über `gateway.nodes.commands.allow`.

Nachdem ein Node seine Liste deklarierter Befehle geändert hat, lehnen Sie die alte Gerätekopplung ab und genehmigen Sie die neue Anfrage, damit der Gateway den aktualisierten Befehlsschnappschuss speichert.

## Konfiguration (`openclaw.json`)

Node-bezogene Einstellungen befinden sich unter `gateway.nodes` und `tools.exec`:

```json5
{
  gateway: {
    nodes: {
      // Erstmalige Node-Kopplungen aus vertrauenswürdigen Netzwerken automatisch genehmigen (CIDR-Liste).
      // Deaktiviert, wenn nicht festgelegt. Gilt nur für erstmalige role:node-Anfragen
      // ohne angeforderte Geltungsbereiche; Upgrades werden nicht automatisch genehmigt.
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
        // SSH-verifizierte automatische Genehmigung (Standard: aktiviert). Genehmigt erstmalige
        // Node-Kopplungen bei exakter Übereinstimmung des über SSH zurückgelesenen Geräteschlüssels.
        sshVerify: true,
      },
      // Von gekoppelten Nodes veröffentlichte, für Agenten sichtbare Plugin-Tools als vertrauenswürdig einstufen (Standard: true).
      pluginTools: {
        enabled: true,
      },
      // Gefährliche/datenschutzintensive Node-Befehle aktivieren (camera.snap usw.).
      commands: {
        allow: ["camera.snap", "screen.record"],
        // Exakte Befehlsnamen blockieren, selbst wenn sie in den Standardeinstellungen oder commands.allow enthalten sind.
        deny: ["camera.clip"],
      },
    },
  },
  tools: {
    exec: {
      // Standardmäßiger exec-Host: "node" leitet alle exec-Aufrufe an einen gekoppelten Node weiter.
      host: "node",
      // Sicherheitsmodus für Node-exec: nur genehmigte/auf der Positivliste stehende Befehle zulassen.
      security: "allowlist",
      // exec an einen bestimmten Node binden (ID oder Name). Weglassen, um jeden Node zuzulassen.
      node: "build-node",
    },
  },
}
```

Verwenden Sie exakte Node-Befehlsnamen. `commands.deny` entfernt einen Befehl selbst dann, wenn eine Plattformstandardeinstellung oder ein `commands.allow`-Eintrag ihn andernfalls zulassen würde. Gekoppelte Nodes dürfen standardmäßig für Agenten sichtbare Plugin-Tool-Deskriptoren veröffentlichen, der Befehl jedes Deskriptors muss jedoch weiterhin zur genehmigten Befehlsoberfläche des Node gehören. Legen Sie `gateway.nodes.pluginTools.enabled: false` fest, um alle derartigen Deskriptoren zu ignorieren. Einzelheiten zu den Feldern für Gateway-Node-Kopplung und Befehlsrichtlinien finden Sie in der [Gateway-Konfigurationsreferenz](/de/gateway/configuration-reference#gateway).

Agentenspezifische Überschreibung des exec-Node:

```json5
{
  agents: {
    list: [
      {
        id: "main",
        tools: { exec: { node: "build-node" } },
      },
    ],
  },
}
```

## Screenshots (Canvas-Schnappschüsse)

Wenn der Node den Canvas (WebView) anzeigt, gibt `canvas.snapshot` `{ format, base64 }` zurück.

CLI-Hilfsfunktion (schreibt in eine temporäre Datei und gibt den gespeicherten Pfad aus):

```bash
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format png
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format jpg --max-width 1200 --quality 0.9
```

### Canvas-Steuerung

```bash
openclaw nodes canvas present --node <idOrNameOrIp> --target https://example.com
openclaw nodes canvas hide --node <idOrNameOrIp>
openclaw nodes canvas navigate https://example.com --node <idOrNameOrIp>
openclaw nodes canvas eval --node <idOrNameOrIp> --js "document.title"
```

Hinweise:

- `canvas present` akzeptiert auf Nodes, die lokale Pfade unterstützen, URLs oder lokale Dateipfade (`--target`) sowie optional `--x/--y/--width/--height` zur Positionierung. Linux Canvas akzeptiert HTTP(S)-URLs oder seinen gebündelten A2UI-Renderer.
- `canvas eval` akzeptiert eingebettetes JS (`--js`) oder ein Positionsargument.

### A2UI (Canvas)

```bash
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --text "Hello"
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --jsonl ./payload.jsonl
openclaw nodes canvas a2ui reset --node <idOrNameOrIp>
```

Hinweise:

- Mobile Nodes und Linux-Desktop-Nodes verwenden für eine aktionsfähige Darstellung eine gebündelte, der App eigene A2UI-Seite.
- Es wird nur A2UI v0.8 JSONL unterstützt (v0.9/createSurface wird abgelehnt).
- iOS und Android stellen entfernte Gateway-Canvas-Seiten dar, A2UI-Schaltflächenaktionen werden jedoch nur von der gebündelten, der App eigenen A2UI-Seite ausgelöst. Vom Gateway gehostete HTTP/HTTPS-A2UI-Seiten dienen auf diesen mobilen Clients ausschließlich der Darstellung.
- macOS kann Aktionen von genau der nach Fähigkeit begrenzten Gateway-A2UI-Seite auslösen, die von der App ausgewählt wurde. Andere HTTP/HTTPS-Seiten dienen weiterhin ausschließlich der Darstellung.
- Linux löst Aktionen nur von der gebündelten A2UI-Seite aus. Andere HTTP/HTTPS-Seiten dienen weiterhin ausschließlich der Darstellung, und ein headless Linux-Node ohne Desktop-App kündigt Canvas nicht an.

## Fotos und Videos (Node-Kamera)

Fotos (`jpg`):

```bash
openclaw nodes camera list --node <idOrNameOrIp>
openclaw nodes camera snap --node <idOrNameOrIp>            # Standard: beide Ausrichtungen (2 MEDIA-Zeilen)
openclaw nodes camera snap --node <idOrNameOrIp> --facing front
openclaw nodes camera snap --node <idOrNameOrIp> --device-id <id> --max-width 1200 --quality 0.9 --delay-ms 2000
```

Videoclips (`mp4`):

```bash
openclaw nodes camera clip --node <idOrNameOrIp> --duration 10s
openclaw nodes camera clip --node <idOrNameOrIp> --duration 3000 --no-audio
```

Hinweise:

- Der Node muss sich für `canvas.*` und `camera.*` **im Vordergrund** befinden (Aufrufe im Hintergrund geben `NODE_BACKGROUND_UNAVAILABLE` zurück).
- Nodes begrenzen die Clipdauer, damit die Base64-Nutzlast handhabbar bleibt (die genauen plattformspezifischen Grenzwerte finden Sie unter [Kameraaufnahme](/de/nodes/camera)). Das Agent-Tool `nodes` begrenzt den angeforderten Wert für `durationMs` zusätzlich auf 300000 (5 Minuten), bevor es den Aufruf weiterleitet; der Node selbst erzwingt den strengeren Grenzwert.
- Android fordert nach Möglichkeit die Berechtigungen `CAMERA`/`RECORD_AUDIO` an; verweigerte Berechtigungen führen zu `*_PERMISSION_REQUIRED`.

## Bildschirmaufnahmen (Nodes)

Unterstützte Nodes stellen `screen.record` (mp4) bereit. Beispiel:

```bash
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10 --no-audio
```

Hinweise:

- Die Verfügbarkeit von `screen.record` hängt von der Node-Plattform ab.
- Das Agent-Tool `nodes` begrenzt den angeforderten Wert für `durationMs` auf 300000 (5 Minuten); der Node kann einen strengeren Grenzwert erzwingen, um die Größe der zurückgegebenen Nutzlast zu beschränken.
- `--no-audio` deaktiviert die Mikrofonaufnahme auf unterstützten Plattformen.
- Verwenden Sie `--screen <index>`, um bei mehreren verfügbaren Bildschirmen einen Bildschirm auszuwählen (0 = primär).

## Standort (Nodes)

Nodes stellen `location.get` bereit, wenn der Standort in den Einstellungen aktiviert ist.

CLI-Hilfsbefehl:

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

Hinweise:

- Der Standort ist **standardmäßig deaktiviert**.
- „Immer“ erfordert eine Systemberechtigung; der Abruf im Hintergrund erfolgt nach bestem Bemühen.
- Die Antwort enthält Breiten-/Längengrad, Genauigkeit (Meter) und Zeitstempel.
- Vollständige Parameter-/Antwortstruktur und Fehlercodes: [Standortbefehl](/de/nodes/location-command).

## SMS (Android-Nodes)

Android-Nodes können `sms.send` und `sms.search` bereitstellen, wenn die Person die Berechtigung **SMS** erteilt und das Gerät Telefonie unterstützt. Beide Befehle sind standardmäßig gefährlich: Der Gateway-Betreiber muss sie zusätzlich zu `gateway.nodes.commands.allow` hinzufügen, bevor sie aufgerufen werden können (siehe [Befehlsrichtlinie](#command-policy)).

Aktivieren Sie für die schreibgeschützte SMS-Suche ausdrücklich `openclaw.json`:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["sms.search"] },
    },
  },
}
```

Fügen Sie `sms.send` nur dann separat hinzu, wenn der Node auch Nachrichten senden können soll. Android-Berechtigung und Gateway-Befehlsautorisierung sind voneinander unabhängig; das Erteilen der Telefonberechtigung ändert die Gateway-Richtlinie nicht.

Direkter Aufruf auf niedriger Ebene:

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command sms.send --params '{"to":"+15555550123","message":"Hallo von OpenClaw"}'
```

Hinweise:

- `sms.search` kann deklariert werden, bevor `READ_SMS` erteilt wurde, sodass ein Aufruf eine Berechtigungsdiagnose zurückgeben kann; das Lesen von Nachrichten erfordert weiterhin diese Android-Berechtigung.
- Reine WLAN-Geräte ohne Telefonie geben `sms.send` nicht an.
- Ein Fehler vom Typ `requires explicit gateway.nodes.commands.allow opt-in` bedeutet, dass das Telefon den Befehl deklariert hat, der Gateway-Betreiber ihn jedoch nicht autorisiert hat.

## Befehle für Geräte- und persönliche Daten

iOS- und Android-Nodes geben standardmäßig mehrere schreibgeschützte Datenbefehle an (siehe Tabelle zur [Befehlsrichtlinie](#command-policy)); Android stellt zusätzlich eine größere Befehlsfamilie bereit, die durch eigene Einstellungen in der App gesteuert wird. Ein TypeScript-Node-Host unter macOS oder auf einem monitorlosen Mac gibt `device.apps` erst an, nachdem der Betreiber die Freigabe installierter Apps mit `--share-installed-apps` aktiviert hat.

Verfügbare Familien:

- `device.status`, `device.info` — iOS, Android, Windows.
- `device.permissions`, `device.health` — nur Android.
- `device.apps` — Android-, macOS- und monitorlose Mac-Nodes. Unter Android muss die Freigabe installierter Apps in den Einstellungen aktiviert sein; standardmäßig werden im Launcher sichtbare Apps zurückgegeben. TypeScript-Node-Hosts lassen die Freigabe standardmäßig deaktiviert und akzeptieren `query`, `limit` und `includeSystem`; macOS-Ergebnisse enthalten `label`, `bundleId`, `path` und `system`.
- `notifications.list`, `notifications.actions` — nur Android.
- `photos.latest` — iOS, Android.
- `contacts.search` — iOS, Android (standardmäßig schreibgeschützt); `contacts.add` ist gefährlich und benötigt `gateway.nodes.commands.allow`.
- `calendar.events` — iOS, Android (standardmäßig schreibgeschützt); `calendar.add` ist gefährlich und benötigt `gateway.nodes.commands.allow`.
- `reminders.list` — iOS, Android (standardmäßig schreibgeschützt); `reminders.add` ist gefährlich und benötigt `gateway.nodes.commands.allow`.
- `callLog.search` — nur Android.
- `motion.activity`, `motion.pedometer` — iOS, Android; abhängig von den verfügbaren Sensoren.

Beispielaufrufe:

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command device.status --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command device.apps --params '{"limit":10}'
openclaw nodes invoke --node <idOrNameOrIp> --command notifications.list --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command photos.latest --params '{"limit":1}'
```

## Systembefehle (Node-Host/Mac-Node)

Der macOS-Node stellt `system.run`, `system.which`, `system.notify` und `system.execApprovals.get/set` bereit. Der monitorlose Node-Host stellt `system.run.prepare`, `system.run`, `system.which` und `system.execApprovals.get/set` bereit.

Beispiele:

```bash
openclaw nodes notify --node <idOrNameOrIp> --title "Ping" --body "Gateway bereit"
openclaw nodes invoke --node <idOrNameOrIp> --command system.which --params '{"bins":["git"]}'
```

Hinweise:

- `system.run` gibt Standardausgabe, Standardfehlerausgabe und Exit-Code in der Nutzlast zurück.
- Die Shell-Ausführung erfolgt jetzt über das Tool `exec` mit `host=node`; `nodes` bleibt die direkte RPC-Schnittstelle für explizite Node-Befehle.
- `nodes invoke` stellt `system.run` oder `system.run.prepare` nicht bereit; diese bleiben ausschließlich im Ausführungspfad.
- Der Ausführungspfad bereitet vor der Genehmigung einen kanonischen Wert für `systemRunPlan` vor. Sobald eine Genehmigung erteilt wurde, leitet das Gateway diesen gespeicherten Plan weiter und nicht etwaige später vom Aufrufer bearbeitete Befehls-, Arbeitsverzeichnis- oder Sitzungsfelder.
- `system.notify` berücksichtigt den Status der Benachrichtigungsberechtigung in der macOS-App und unterstützt `--priority <passive|active|timeSensitive>` und `--delivery <system|overlay|auto>`.
- Nicht erkannte Node-Metadaten für `platform` / `deviceFamily` verwenden eine konservative Standard-Zulassungsliste, die `system.run` und `system.which` ausschließt. Wenn Sie diese Befehle absichtlich für eine unbekannte Plattform benötigen, fügen Sie sie ausdrücklich über `gateway.nodes.commands.allow` hinzu.
- `system.run` unterstützt `--cwd`, `--env KEY=VAL`, `--command-timeout` und `--needs-screen-recording`.
- Bei Shell-Wrappern (`bash|sh|zsh ... -c/-lc`) werden anfragebezogene Werte für `--env` auf eine explizite Zulassungsliste beschränkt (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`).
- Bei Entscheidungen zur dauerhaften Zulassung im Zulassungslistenmodus speichern bekannte Dispatch-Wrapper (`env`, `flock`, `nice`, `nohup`, `stdbuf`, `timeout`) die Pfade innerer ausführbarer Dateien anstelle der Wrapper-Pfade. Wenn das Entfernen des Wrappers nicht sicher ist, wird automatisch kein Zulassungslisteneintrag gespeichert.
- Auf Windows-Node-Hosts im Zulassungslistenmodus erfordern Shell-Wrapper-Ausführungen über `cmd.exe /c` eine Genehmigung (ein Zulassungslisteneintrag allein lässt die Wrapper-Form nicht automatisch zu).
- Node-Hosts ignorieren Überschreibungen für `PATH` in `--env` und entfernen vor der Ausführung eines Befehls eine große, gepflegte Gruppe von Startvariablen für Interpreter und Shells (beispielsweise `NODE_OPTIONS`, `PYTHONPATH`, `BASH_ENV`, `DYLD_*`, `LD_*`). Wenn Sie zusätzliche PATH-Einträge benötigen, konfigurieren Sie die Dienstumgebung des Node-Hosts (oder installieren Sie Tools an Standardspeicherorten), anstatt `PATH` über `--env` zu übergeben.
- Im macOS-Node-Modus wird `system.run` durch Ausführungsgenehmigungen in der macOS-App gesteuert (Settings → Exec approvals). Nachfragen/Zulassungsliste/vollständig verhalten sich wie beim monitorlosen Node-Host; abgelehnte Aufforderungen geben `SYSTEM_RUN_DENIED` zurück.
- Auf dem monitorlosen Node-Host wird `system.run` durch Ausführungsgenehmigungen (`~/.openclaw/exec-approvals.json`) gesteuert; speziell für macOS finden Sie die Umgebungsvariablen für das Routing des Ausführungs-Hosts weiter unten unter [Monitorloser Node-Host](#headless-node-host-cross-platform).

## Bindung des Ausführungs-Nodes

Wenn mehrere Nodes verfügbar sind, können Sie die Ausführung an einen bestimmten Node binden. Dadurch wird der Standard-Node für `exec host=node` festgelegt (und kann pro Agent überschrieben werden).

Globaler Standard:

```bash
openclaw config set tools.exec.node "node-id-or-name"
```

Überschreibung pro Agent:

```bash
openclaw config get agents.entries
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
```

Entfernen Sie die Einstellung, um jeden Node zuzulassen:

```bash
openclaw config unset tools.exec.node
openclaw config unset 'agents.entries.main.tools.exec.node'
```

## Berechtigungszuordnung

Nodes können in `node.list` / `node.describe` eine Zuordnung `permissions` enthalten, deren Schlüssel Berechtigungsnamen sind (z. B. `screenRecording`, `accessibility`, `location`) und deren Werte boolesch sind (`true` = erteilt).

## Monitorloser Node-Host (plattformübergreifend)

OpenClaw kann einen **monitorlosen Node-Host** (ohne Benutzeroberfläche) ausführen, der eine Verbindung zum Gateway-WebSocket herstellt und `system.run` / `system.which` bereitstellt. Dies ist unter Linux/Windows oder zum Ausführen eines minimalen Nodes neben einem Server nützlich.

Starten Sie ihn:

```bash
openclaw node run --host <gateway-host> --port 18789
```

Hinweise:

- Die Kopplung ist weiterhin erforderlich (das Gateway zeigt eine Aufforderung zur Gerätekopplung an).
- Metadaten der Clientinstanz, signierte Geräteidentität und Kopplungsauthentifizierung verwenden separate Zustandsdatensätze; siehe [Identitätsstatus des monitorlosen Hosts](#headless-identity-state).
- Ausführungsgenehmigungen werden lokal über `~/.openclaw/exec-approvals.json` durchgesetzt (siehe [Ausführungsgenehmigungen](/de/tools/exec-approvals)).
- Unter macOS führt der monitorlose Node-Host `system.run` standardmäßig lokal aus. Legen Sie `OPENCLAW_NODE_EXEC_HOST=app` fest, um `system.run` über den Ausführungs-Host der Begleit-App zu leiten; fügen Sie `OPENCLAW_NODE_EXEC_FALLBACK=0` hinzu, um den App-Host zwingend vorauszusetzen und bei Nichtverfügbarkeit ohne Ausweichlösung abzubrechen.
- Fügen Sie `--tls` / `--tls-fingerprint` hinzu, wenn der Gateway-WebSocket TLS verwendet.

## Mac-Node-Modus

- Die macOS-Menüleisten-App stellt als Node eine Verbindung zum Gateway-WebSocket-Server her (sodass `openclaw nodes …` für diesen Mac funktioniert).
- Im Remote-Modus öffnet die App einen SSH-Tunnel für den Gateway-Port und stellt eine Verbindung zu `localhost` her.
