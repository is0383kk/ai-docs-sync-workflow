---
read_when:
    - Status der Linux-Begleit-App suchen
    - Kamera, Standort oder Benachrichtigungen auf einem Linux-Node-Host aktivieren
    - Plattformabdeckung oder Beiträge planen
    - Debugging von Linux-OOM-Kills oder Exit-Code 137 auf einem VPS oder in einem Container
summary: Linux-Unterstützung + Status der Begleit-App
title: Linux-App
x-i18n:
    generated_at: "2026-07-26T17:56:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fe55d3ec63fcf8291a24126c04638f005c03c3d44ff84a26a925e931066b01cc
    source_path: platforms/linux.md
    workflow: 16
---

Der Gateway wird unter Linux vollständig unterstützt und benötigt Node. Bun kann weiterhin
als Abhängigkeits-Installer oder Runner für Paketskripte verwendet werden, OpenClaw jedoch
nicht ausführen, da es `node:sqlite` nicht bereitstellt.

## Desktop-Begleit-App

Die OpenClaw-Begleit-App für Linux ist eine Tauri-Desktop-App für einen lokalen Gateway. Sie:

- installiert die OpenClaw-CLI und die verwaltete Node-Laufzeitumgebung, wenn sie fehlen; Release-Builds installieren den stabilen Kanal automatisch, während Entwicklungs-Builds zuerst nach dem Kanal fragen
- stellt eine Verbindung zu einem fehlerfrei ausgeführten Gateway her, bevor sie Änderungen am Dienst versucht
- delegiert Installations-, Start-, Stopp- und Neustartvorgänge an den von der CLI verwalteten systemd-Benutzerdienst
- erkennt Gateways in der Nähe über Bonjour und öffnet jede Control UI in einem routenspezifischen Fenster, sodass mehrere
  Gateway-Dashboards gleichzeitig verbunden bleiben und verwendet werden können
- öffnet die vom Gateway bereitgestellte Control UI mit der aufgelösten Authentifizierungs-URL
- öffnet die Control UI nach der erstmaligen Installation im Onboarding-Modus, der
  anbietet, erkannte Erinnerungen aus Claude Code, Codex oder Hermes in den
  Agent-Arbeitsbereich zu importieren (derselbe Import ist später weiterhin unter
  Settings → Import Memory verfügbar)
- rendert Agent-gesteuerte Canvas- und gebündelte A2UI-Inhalte für einen lokal ausgeführten CLI-Node-Host
- bleibt über den Systembereich verfügbar, wenn das Fenster geschlossen wird

Stabile Releases, die aus `main` erstellt wurden, stellen `.deb`- und AppImage-Pakete als Assets im
[GitHub-Release](https://github.com/openclaw/openclaw/releases) für das Tag bereit,
benannt als `OpenClaw-<version>-amd64.deb` und `OpenClaw-<version>-amd64.AppImage`,
mit einer danebenliegenden `SHA256SUMS.linux-app.txt`-Prüfsummendatei. Laden Sie
`.deb` herunter und installieren Sie es mit `sudo apt install ./OpenClaw-<version>-amd64.deb`,
oder markieren Sie das AppImage als ausführbar und führen Sie es direkt aus. Die AppImage-Laufzeitumgebung
benötigt FUSE 2 (`sudo apt install libfuse2` oder `libfuse2t64` unter Ubuntu 24.04+);
führen Sie das AppImage ohne FUSE 2 mit `APPIMAGE_EXTRACT_AND_RUN=1` aus.

Sie können dieselben Pakete auch aus einem ausgecheckten Quellcode-Repository erstellen:

```bash
cd apps/linux/src-tauri
pnpm dlx @tauri-apps/cli@2.11.4 build --bundles deb,appimage
```

Der CI-Workflow `Linux App` lädt dieselben Pakete als
Artefakt `openclaw-linux-companion` für Pull Requests, die die App betreffen, sowie für
manuelle Ausführungen hoch. Informationen zu Linux-Build-Abhängigkeiten und
Entwicklungsbefehlen finden Sie unter `apps/linux/README.md` im Repository.

### Quick Chat

Öffnen Sie Quick Chat mit `Ctrl+Shift+Space` oder über das Taskleistenelement **Quick Chat**. Der Agent-Chip
zeigt den konfigurierten Avatar, das Emoji oder Monogramm an; wählen Sie ihn aus, um den Agent zu wechseln.
Nachrichten verwenden die Hauptsitzung des ausgewählten Agent und berücksichtigen den globalen Sitzungsbereich.
Der native Rust-Client verwaltet eine persistente Ed25519-Geräteidentität. Er verwendet das
gemeinsame Token oder Passwort aus der CLI-Übergabe nur zum Initialisieren der Kopplung, speichert anschließend das
vom Gateway ausgestellte Geräte-Token und bevorzugt es bei späteren Verbindungen. Die Identität und
das Geräte-Token befinden sich im App-Konfigurationsverzeichnis in einer Datei mit dem Modus `0600`; die WebView von Quick
Chat erhält weder Anmeldedaten noch den WebSocket.

Wenn die native Verbindung nicht verfügbar ist, zeigt Quick Chat **Gateway
unreachable — retrying** an und deaktiviert das Senden bis zur erneuten Verbindung. Ein Remote-Gerät,
das die Kopplungsphase erreicht hat, zeigt stattdessen **Approve this device in the dashboard
(Nodes)** an, einschließlich einer kurzen Geräte-ID, wenn der Gateway eine bereitstellt. Ein
Gateway, der fehlende gemeinsame Anmeldedaten benötigt, zeigt **Gateway requires a
credential — open the dashboard on the gateway host** an; in diesem Zustand wartet keine
Kopplungsanfrage auf Genehmigung. Vom Server bereitgestellte Anweisungen zur Fehlerbehebung
ersetzen diese Ausweichhinweise, wenn sie spezifischer sind.
Bei TLS-Gateways übergibt die CLI der App den SHA-256-Fingerabdruck des
Gateway-Zertifikats; der native Client bindet dieses Zertifikat fest ein und meldet **Gateway TLS
trust failed — check the certificate fingerprint** getrennt von Ausfallzeiten.
Bei Gateways, deren gemeinsames Secret über eine SecretRef konfiguriert ist, wird es bei der
CLI-Übergabe ausgelassen. Bereits gekoppelte Installationen funktionieren weiterhin mit ihrem gespeicherten Geräte-
Token, aber eine Neuinstallation kann bei Authentifizierung über ein gemeinsames Secret ohne diese
Bootstrap-Anmeldedaten keine ausstehende Kopplungsanfrage erstellen.
Das Einlösen von Einrichtungscodes und `bootstrapToken` benötigt eine eigene Produktoberfläche und bleibt
eine Folgeaufgabe; Quick Chat versucht keinen der beiden Abläufe.

Verwenden Sie unter X11 das Zahnrad in Quick Chat, um eine benutzerdefinierte Tastenkombination aufzuzeichnen oder zurückzusetzen. Der
Taskleistenschalter **Quick Chat shortcut** aktiviert oder deaktiviert sie, ohne das
einfache Taskleistenelement **Quick Chat** zu deaktivieren. Globale Tastenkombinationen sind unter Wayland nicht verfügbar, daher
werden die Tastenkombinationseinstellungen ausgeblendet und das Taskleistenelement bleibt der Einstiegspunkt.
Nach einem akzeptierten Sendevorgang bleibt Quick Chat geöffnet und streamt die
Klartextantwort des ausgewählten Agent unterhalb des Eingabefelds. Drücken Sie `Esc`, um die Leiste und ihre Antwort zu schließen;
`Ctrl+Enter` öffnet weiterhin das Dashboard.

### Canvas

Linux Canvas verwendet zwei zusammenarbeitende Prozesse. `openclaw node run` bleibt die einzige Gateway-Node-Verbindung; das gebündelte Plugin `linux-canvas` leitet `canvas.*`-Aufrufe über einen Unix-Socket nur für den jeweiligen Benutzer an die laufende Desktop-App weiter. Die App verwaltet ein bei Bedarf geöffnetes WebView-Fenster einschließlich des gebündelten A2UI-Renderers und der Aktionsbrücke zurück zum Agent.

Das Plugin ist standardmäßig aktiviert. Es kündigt Canvas nur an, wenn der Desktop-Socket unter `$XDG_RUNTIME_DIR/openclaw-canvas.sock` vorhanden ist, oder unter `/tmp/openclaw-canvas-$UID.sock`, wenn `XDG_RUNTIME_DIR` nicht verfügbar ist. Deaktivieren Sie es mit `plugins.entries.linux-canvas.enabled: false`. Auf einem Headless-Linux-Server ohne Desktop-App wird Canvas nicht angekündigt.

Linux v1 verwendet ein Canvas-Fenster. HTTP- und HTTPS-Seiten können gerendert werden, A2UI-Aktionen werden jedoch nur vom gebündelten Renderer akzeptiert.

## CLI- und SSH-Alternative

Die CLI bleibt die einfachste Option für einen Headless-Server, einen VPS oder einen Remote-Gateway:

1. Installieren Sie Node 24.15+ (empfohlen), Node 22.22.3+ (LTS) oder Node 25.9+.
2. `npm i -g openclaw@latest`
3. `openclaw onboard --install-daemon`
4. Von Ihrem Laptop aus: `ssh -N -L 18789:127.0.0.1:18789 <user>@<host>`
5. Öffnen Sie `http://127.0.0.1:18789/` und authentifizieren Sie sich mit dem konfigurierten gemeinsamen
   Secret (standardmäßig Token; Passwort, wenn `gateway.auth.mode` auf `"password"` gesetzt ist).

Vollständige Serveranleitung: [Linux-Server](/de/vps). Schrittweises VPS-Beispiel:
[exe.dev](/de/install/exe-dev).

## Node-Funktionen

Das gebündelte Linux-Node-Plugin stellt der CLI die Gerätefunktionen des Dienstes `openclaw node` bereit, ohne dass die Desktop-App erforderlich ist. Befehle werden dem Gateway nur angekündigt, wenn ihre Funktion aktiviert ist und das erforderliche lokale Werkzeug vorhanden ist.

| Funktion                                | Standard | Voraussetzung                                                        |
| --------------------------------------- | -------- | -------------------------------------------------------------------- |
| Desktop-Benachrichtigungen (`system.notify`) | Ein      | `notify-send` aus libnotify und eine Desktop-Benachrichtigungssitzung |
| Kamerafotos und -clips (`camera.*`)     | Aus      | FFmpeg, V4L2-Kamerazugriff und PulseAudio oder PipeWire für Clip-Audio |
| Standort (`location.get`)                   | Aus      | GeoClue2 und dessen `where-am-i`-Demo                          |

Konfigurieren Sie das Plugin in `openclaw.json`:

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          notify: { enabled: true },
          camera: { enabled: true },
          location: { enabled: true },
        },
      },
    },
  },
}
```

Starten Sie den Node-Dienst nach dem Ändern dieser Einstellungen neu. Die Verfügbarkeit wird einmal pro Prozess ermittelt und die Node-Ankündigung beim Neustart neu erstellt.

Der Gateway genehmigt die Befehls- und Funktionsoberfläche des Node getrennt von der Gerätekopplung. Genehmigen Sie beim ersten Start oder nach dem Aktivieren weiterer Funktionen die ausstehende Oberfläche:

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

Ein Node kann verbunden und mit einem Gerät gekoppelt sein, während seine effektiven `caps` und `commands` leer bleiben, bis diese Genehmigung abgeschlossen ist.

Kamerageräte müssen für den Dienstbenutzer lesbar sein, üblicherweise über die Gruppe `video`. Kameraclips verwenden die standardmäßige PulseAudio- oder PipeWire-Quelle, wenn `includeAudio` auf true gesetzt ist; Mikrofonaudio ist nur als diese Clip-Tonspur vorhanden, nicht als eigenständiger Befehl. Für den Standort muss der Node-Dienstbenutzer gemäß der GeoClue-Richtlinie des Hosts zugelassen sein.

`camera.snap` und `camera.clip` erfordern außerdem eine ausdrückliche Aktivierung durch den Gateway über `gateway.nodes.commands.allow`. Informationen zu Nutzlasten, Beschränkungen und Fehlern finden Sie unter [Kameraaufnahme](/de/nodes/camera) und [Standortbefehl](/de/nodes/location-command).

## Installation

- [Erste Schritte](/de/start/getting-started)
- [Installation und Aktualisierungen](/de/install/updating)
- Optional: [Bun-Paketworkflow](/de/install/bun), [Nix](/de/install/nix), [Docker](/de/install/docker)

## Gateway-Dienst (systemd)

Installieren Sie ihn mit einem der folgenden Befehle:

```bash
openclaw onboard --install-daemon
openclaw gateway install
openclaw configure   # bei Aufforderung "Gateway service" auswählen
```

Reparieren oder migrieren Sie eine vorhandene Installation:

```bash
openclaw doctor
```

`openclaw gateway install` erzeugt standardmäßig eine systemd-**Benutzer**-Unit. Die vollständige
Dienstanleitung einschließlich der Unit-Variante auf **System**ebene für gemeinsam genutzte oder
dauerhaft aktive Hosts finden Sie im [Gateway-Betriebshandbuch](/de/gateway#supervision-and-service-lifecycle).

Erstellen Sie eine Unit nur für eine benutzerdefinierte Einrichtung manuell. Minimales Beispiel einer Benutzer-Unit
(`~/.config/systemd/user/openclaw-gateway[-<profile>].service`):

```ini
[Unit]
Description=OpenClaw Gateway (profile: <profile>, v<version>)
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

Manuell erstellte Units übernehmen nicht die adaptive Heap-Größenanpassung, die `openclaw gateway install` für verwaltete Gateway-Dienste schreibt. Verwenden Sie vorzugsweise den verwalteten Installer oder legen Sie im benutzerdefinierten Supervisor ein explizites Heap-Limit fest, nachdem Sie den Spielraum für nativen Speicher berücksichtigt haben.

Aktivieren Sie die Unit:

```bash
systemctl --user enable --now openclaw-gateway[-<profile>].service
```

## Speicherdruck und OOM-Beendigungen

Unter Linux wählt der Kernel ein OOM-Opfer aus, wenn der Arbeitsspeicher eines Hosts, einer VM oder einer Container-cgroup
erschöpft ist. Der Gateway ist dafür ungeeignet, da er langlebige
Sitzungen und Kanalverbindungen verwaltet. Daher bevorzugt OpenClaw nach Möglichkeit,
vorübergehende Kindprozesse zuerst zu beenden.

Bei geeigneten Linux-Kindprozessstarts umschließt OpenClaw den Befehl mit einem kurzen
`/bin/sh`-Shim, der den eigenen `oom_score_adj`-Wert des Kindprozesses auf `1000` erhöht und anschließend
den eigentlichen Befehl mit `exec` ausführt. Dafür sind keine erhöhten Berechtigungen erforderlich: Ein Prozess darf seinen
eigenen OOM-Wert stets erhöhen.

Abgedeckte Kindprozessoberflächen:

- Vom Supervisor verwaltete Befehls-Kindprozesse
- PTY-Shell-Kindprozesse
- MCP-stdio-Server-Kindprozesse
- Von OpenClaw gestartete Browser-/Chrome-Prozesse (über die Prozesslaufzeit des Plugin-SDK)

Der Wrapper ist ausschließlich für Linux vorgesehen und wird übersprungen, wenn `/bin/sh` nicht verfügbar ist oder wenn
die Umgebung des Kindprozesses `OPENCLAW_CHILD_OOM_SCORE_ADJ` auf `0`, `false`, `no` oder
`off` setzt.

Überprüfen Sie einen Kindprozess:

```bash
cat /proc/<child-pid>/oom_score_adj
```

Der erwartete Wert für abgedeckte Kindprozesse ist `1000`; der Gateway-Prozess selbst
behält seinen normalen Wert bei (üblicherweise `0`).

`OOMPolicy=continue` der systemd-Unit hält den Gateway-Dienst aktiv, wenn
ein vorübergehender Kindprozess vom OOM-Killer ausgewählt wird, anstatt die gesamte
Unit als fehlgeschlagen zu markieren und alle Kanäle neu zu starten; der fehlgeschlagene Kindprozess beziehungsweise die fehlgeschlagene Sitzung meldet den
eigenen Fehler.

Dies ersetzt keine normale Speicheroptimierung. Wenn ein VPS oder Container wiederholt
Kindprozesse beendet, erhöhen Sie das Speicherlimit, reduzieren Sie die Parallelität oder fügen Sie strengere
Ressourcensteuerungen hinzu (systemd `MemoryMax=`, Container-Speicherlimits).

## Verwandte Themen

- [Installationsübersicht](/de/install)
- [Linux-Server](/de/vps)
- [Raspberry Pi](/de/install/raspberry-pi)
- [Gateway-Betriebshandbuch](/de/gateway)
- [Gateway-Konfiguration](/de/gateway/configuration)
