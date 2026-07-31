---
read_when:
    - Einrichten oder Debuggen der macOS-Fernsteuerung
summary: macOS-App-Ablauf zur Steuerung eines entfernten OpenClaw-Gateways
title: Fernsteuerung
x-i18n:
    generated_at: "2026-07-26T19:04:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7e558c39fa173a77bf11270a8961c14c6e2350dfc4f458da3633532513b98bf6
    source_path: platforms/mac/remote.md
    workflow: 16
---

Dieser Ablauf ermöglicht es der macOS-App, als vollständige Fernsteuerung für ein OpenClaw-Gateway zu fungieren, das auf einem anderen Host (Desktop/Server) ausgeführt wird. Die App stellt direkt eine Verbindung zu vertrauenswürdigen LAN-/Tailnet-Gateway-URLs her oder verwaltet einen SSH-Tunnel, wenn das entfernte Gateway nur über Loopback erreichbar ist. Zustandsprüfungen, die Weiterleitung von Voice Wake und Web Chat verwenden dieselbe Remote-Konfiguration aus _Settings -> General_.

## Modi

- **Lokal (dieser Mac)**: Alles wird auf dem Laptop ausgeführt; SSH ist nicht beteiligt.
- **Remote über SSH (Standard)**: OpenClaw-Befehle werden auf dem entfernten Host ausgeführt. Die App öffnet mit `-o BatchMode`, Ihrer ausgewählten Identität/Ihrem ausgewählten Schlüssel und einer lokalen Portweiterleitung eine SSH-Verbindung.
- **Direkt remote (ws/wss)**: Kein SSH-Tunnel; die App stellt direkt eine Verbindung zur Gateway-URL her (LAN, Tailscale, Tailscale Serve oder ein öffentlicher HTTPS-Reverse-Proxy).

## Remote-Transporte

- **SSH-Tunnel** (Standard): Verwendet `ssh -N -L ...`, um den Gateway-Port an localhost weiterzuleiten. Das Gateway sieht die IP-Adresse des Nodes als `127.0.0.1`, da der Tunnel über Loopback läuft.
- **Direkt (ws/wss)**: Stellt direkt eine Verbindung zur Gateway-URL her. Das Gateway sieht die tatsächliche Client-IP-Adresse.

Die App deaktiviert für ihre eigenen SSH-Prozesse das Multiplexing von SSH-Verbindungen und die Ausführung im Hintergrund nach der Authentifizierung, damit sie genau diesen Prozess überwachen und neu starten kann, selbst wenn der ausgewählte Alias `ControlMaster` oder `ForkAfterAuthentication` aktiviert.

Die Überprüfung des SSH-Hostschlüssels ist standardmäßig strikt, da Gateway-Zugangsdaten durch diesen Tunnel übertragen werden. Um stattdessen das eigene Vertrauensverhalten eines verwalteten SSH-Alias zu verwenden, legen Sie `--ssh-host-key-policy openssh` über `openclaw-mac configure-remote` fest oder setzen Sie `gateway.remote.sshHostKeyPolicy` direkt auf `"openssh"`. Prüfen Sie den Alias und jede passende `Host *`- oder Systemkonfiguration, bevor Sie sich dafür entscheiden. Wenn das SSH-Ziel geändert wird (in der App oder über `configure-remote`), wird die Richtlinie auf `strict` zurückgesetzt, sofern Sie sich nicht ausdrücklich erneut für das neue Ziel dafür entscheiden.

Im SSH-Tunnelmodus werden erkannte LAN-/Tailnet-Hostnamen als `gateway.remote.sshTarget` gespeichert. Die App behält `gateway.remote.url` am lokalen Tunnelendpunkt bei (zum Beispiel `ws://127.0.0.1:18789`), sodass CLI, Web Chat und der lokale Node-Host-Dienst denselben Loopback-Transport verwenden. Wenn die Erkennung sowohl unformatierte Tailnet-IP-Adressen als auch stabile Hostnamen zurückgibt, bevorzugt die App Tailscale-MagicDNS- oder LAN-Namen, damit Verbindungen Adressänderungen besser überstehen. Wenn sich der lokale Tunnelport vom Port des entfernten Gateways unterscheidet, setzen Sie `gateway.remote.remotePort` auf den Port des entfernten Hosts.

Die Browserautomatisierung im Remote-Modus wird vom CLI-Node-Host verwaltet, nicht vom Node der nativen macOS-App. Die App startet nach Möglichkeit den installierten Node-Host-Dienst. Um die Browsersteuerung von diesem Mac aus zu aktivieren, installieren/starten Sie ihn mit `openclaw node install ...` und `openclaw node start` (oder führen Sie `openclaw node run ...` im Vordergrund aus) und wählen Sie anschließend diesen browserfähigen Node als Ziel aus.

## Voraussetzungen auf dem entfernten Host

1. Installieren Sie Node und pnpm und erstellen/installieren Sie die OpenClaw-CLI (`pnpm install && pnpm build && pnpm link --global`).
2. Stellen Sie sicher, dass sich `openclaw` für nicht interaktive Shells im PATH befindet (erstellen Sie bei Bedarf einen symbolischen Link in `/usr/local/bin` oder `/opt/homebrew/bin`).
3. Für den SSH-Transport: Richten Sie eine schlüsselbasierte SSH-Authentifizierung ein. Tailscale-IP-Adressen werden für eine stabile Erreichbarkeit außerhalb des LAN empfohlen.

## Einrichtung der macOS-App

So konfigurieren Sie die App über SSH vor, ohne den Begrüßungsablauf zu verwenden:

```bash
openclaw-mac configure-remote \
  --ssh-target user@gateway-host \
  --local-port 18789 \
  --remote-port 18789 \
  --token "$OPENCLAW_GATEWAY_TOKEN"
```

Oder überspringen Sie SSH vollständig, wenn das Gateway bereits über ein vertrauenswürdiges LAN oder Tailnet erreichbar ist:

```bash
openclaw-mac configure-remote \
  --direct-url ws://192.168.0.202:18789 \
  --token "$OPENCLAW_GATEWAY_TOKEN"
```

`openclaw-mac connect`, `wizard` und `configure-remote` ermitteln die aktive Konfiguration in dieser Reihenfolge: `OPENCLAW_CONFIG_PATH`, dann `$OPENCLAW_STATE_DIR/openclaw.json`, dann `~/.openclaw/openclaw.json`. Beide Konfigurationsformen schreiben diese aktive Datei, markieren das Onboarding als abgeschlossen und überlassen der App beim nächsten Start die Verwaltung des ausgewählten Transports. `--local-port`/`--remote-port` verwenden standardmäßig `18789`. Weitere Flags: `--password`, `--identity <path>`, `--ssh-host-key-policy <strict|openssh>`, `--project-root <path>`, `--cli-path <path>`, `--json`. Führen Sie `openclaw-mac configure-remote --help` aus, um die vollständige Referenz anzuzeigen.

So konfigurieren Sie stattdessen über die Benutzeroberfläche:

1. Öffnen Sie _Settings -> General_.
2. Wählen Sie unter **OpenClaw runs** die Option **Remote** aus und legen Sie Folgendes fest:
   - **Transport**: **SSH tunnel** oder **Direct (ws/wss)**.
   - **SSH target**: `user@host` (optional `:port`). Wenn sich das Gateway im selben LAN befindet und über Bonjour angekündigt wird, wählen Sie es aus der Liste der erkannten Geräte aus, um dieses Feld automatisch auszufüllen.
   - **Gateway URL** (nur Direct): `wss://gateway.example.ts.net` (oder `ws://...` für lokal/LAN).
   - **Identity file** (erweitert): Pfad zu Ihrem Schlüssel.
   - **Project root** (erweitert): Pfad des entfernten Checkouts, der für Befehle verwendet wird.
   - **CLI path** (erweitert): Optionaler Pfad zu einem ausführbaren `openclaw`-Einstiegspunkt bzw. einer solchen Binärdatei (wird automatisch ausgefüllt, wenn angekündigt).
3. Klicken Sie auf **Test remote**. Ein Erfolg bedeutet, dass der entfernte `openclaw status --json` korrekt ausgeführt wurde. Fehler weisen üblicherweise auf Probleme mit PATH/CLI hin; Exit-Code 127 bedeutet, dass die CLI auf dem entfernten System nicht gefunden wurde.
4. Zustandsprüfungen und Web Chat werden nun automatisch über den ausgewählten Transport ausgeführt.

## Web Chat

- **SSH-Tunnel**: Stellt über den weitergeleiteten WebSocket-Steuerungsport (standardmäßig 18789) eine Verbindung zum Gateway her.
- **Direkt (ws/wss)**: Stellt direkt eine Verbindung zur konfigurierten Gateway-URL her.
- Es gibt keinen separaten HTTP-Server für Web Chat.

## Berechtigungen

- Der entfernte Host benötigt dieselben TCC-Genehmigungen wie der lokale Host (Automation, Accessibility, Screen Recording, Microphone, Speech Recognition, Notifications). Führen Sie das Onboarding einmal auf diesem Computer aus, um sie zu erteilen.
- Nodes geben ihren Berechtigungsstatus über `node.list` / `node.describe` bekannt, damit Agenten wissen, was verfügbar ist.

## Sicherheitshinweise

- Bevorzugen Sie Loopback-Bindungen auf dem entfernten Host und stellen Sie die Verbindung über SSH, Tailscale Serve oder eine vertrauenswürdige direkte Tailnet-/LAN-URL her.
- SSH-Tunneling erfordert standardmäßig einen bereits vertrauenswürdigen Hostschlüssel. Vertrauen Sie zuerst dem Hostschlüssel (fügen Sie ihn der konfigurierten Known-Hosts-Datei hinzu) oder setzen Sie ausdrücklich `gateway.remote.sshHostKeyPolicy: "openssh"` für einen verwalteten Alias, dessen OpenSSH-Vertrauensrichtlinie Sie akzeptieren.
- Wenn Sie das Gateway an eine Nicht-Loopback-Schnittstelle binden, verlangen Sie eine gültige Gateway-Authentifizierung: Token, Passwort oder einen identitätsorientierten Reverse-Proxy mit `gateway.auth.mode: "trusted-proxy"`.
- Direkte `wss://`-Verbindungen wenden eine Zertifikatsrichtlinie sowohl auf Operator-/Steuerungsdatenverkehr als auch auf den Mac-Begleit-Node an. Legen Sie `gateway.remote.tlsFingerprint` für einen expliziten Pin fest. Ohne einen solchen zeichnet die App erst dann einen Pin bei der ersten Verwendung auf, nachdem die normale macOS-Vertrauensprüfung erfolgreich war.
- Siehe [Sicherheit](/de/gateway/security) und [Tailscale](/de/gateway/tailscale).

## WhatsApp-Anmeldeablauf (remote)

- Führen Sie `openclaw channels login --channel whatsapp --verbose` **auf dem entfernten Host** aus. Scannen Sie den QR-Code mit WhatsApp auf Ihrem Telefon.
- Führen Sie die Anmeldung auf diesem Host erneut aus, wenn die Authentifizierung abläuft. Die Zustandsprüfung zeigt Verbindungsprobleme an.

## Fehlerbehebung

| Symptom                                          | Ursache / Behebung                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `exit 127` / nicht gefunden              | `openclaw` befindet sich für Nicht-Login-Shells nicht im PATH. Fügen Sie es zu `/etc/paths` oder Ihrer Shell-RC-Datei hinzu oder erstellen Sie einen symbolischen Link in `/usr/local/bin`/`/opt/homebrew/bin`.                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Integritätsprüfung fehlgeschlagen                 | Prüfen Sie die SSH-Erreichbarkeit und den PATH sowie, ob Baileys (WhatsApp) angemeldet ist (`openclaw status --json`).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Web Chat reagiert nicht                           | Vergewissern Sie sich, dass der Gateway auf dem Remote-Host ausgeführt wird und der weitergeleitete Port dem WS-Port des Gateways entspricht; die Benutzeroberfläche benötigt eine funktionsfähige WS-Verbindung.                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Node-IP zeigt `127.0.0.1`                  | Dies ist beim SSH-Tunnel zu erwarten. Stellen Sie **Transport** auf **Direct (ws/wss)** um, wenn der Gateway die tatsächliche Client-IP sehen soll.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Dashboard funktioniert, aber Mac-Funktionen sind offline | Die Bediener-/Steuerungsverbindung ist funktionsfähig, aber die Verbindung zum begleitenden Node ist nicht hergestellt oder dessen Befehlsoberfläche fehlt. Öffnen Sie im Menüleistenmenü den Gerätebereich und prüfen Sie, ob der Mac `paired · disconnected` ist. Direkte `wss://`-Verbindungen für Bediener und Node verwenden dieselbe konfigurierte oder gespeicherte Zertifikatsrichtlinie. Bei vertrauenswürdigen `wss://*.ts.net`-Tailscale-Serve-Endpunkten werden veraltete gespeicherte Leaf-Pins nach einer Zertifikatsrotation ersetzt und die Verbindung wird automatisch erneut versucht. Konfigurierte Pins werden nie automatisch rotiert; aktualisieren Sie `gateway.remote.tlsFingerprint` nach Prüfung des neuen Zertifikats oder wechseln Sie zu **Remote over SSH**. |
| Sprachaktivierung                                 | Aktivierungsphrasen werden im Remote-Modus automatisch weitergeleitet; es ist kein separater Weiterleiter erforderlich.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |

## Benachrichtigungstöne

Wählen Sie mit `openclaw nodes notify` für jede Benachrichtigung einen Ton aus den Skripten aus, zum Beispiel:

```bash
openclaw nodes notify --node <id> --title "Ping" --body "Remote gateway ready" --sound Glass
```

In der App gibt es keinen globalen Schalter für einen Standardton; die Aufrufer wählen für jede Anfrage einen Ton (oder keinen) aus.

## Verwandte Themen

- [macOS-App](/de/platforms/macos)
- [Remote-Zugriff](/de/gateway/remote)
