---
read_when:
    - OpenClaw auf Fly.io bereitstellen
    - Fly-Volumes, Secrets und die Konfiguration für den ersten Start einrichten
summary: Schrittweise Fly.io-Bereitstellung von OpenClaw mit persistentem Speicher und HTTPS
title: Fly.io
x-i18n:
    generated_at: "2026-07-26T17:53:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d2b5119c1df8ee077f4db4f44fa92c6ae0e2bf3c355c2117e0fd39146bb49875
    source_path: install/fly.md
    workflow: 16
---

**Ziel:** OpenClaw Gateway auf einer [Fly.io](https://fly.io)-Maschine mit persistentem Speicher, automatischem HTTPS und Discord-/Kanalzugriff ausführen.

## Voraussetzungen

- [flyctl CLI](https://fly.io/docs/hands-on/install-flyctl/) installiert
- Fly.io-Konto (kostenloser Tarif ist ausreichend)
- Modellauthentifizierung: API-Schlüssel für Ihren gewählten Modell-Provider
- Kanal-Anmeldedaten: Discord-Bot-Token, Telegram-Token usw.

## Schnelleinstieg für Anfänger

1. Repository klonen, `fly.toml` anpassen
2. App und Volume erstellen, Secrets festlegen
3. Mit `fly deploy` bereitstellen
4. Per SSH anmelden, um die Konfiguration zu erstellen, oder die Control UI verwenden

<Steps>
  <Step title="Fly-App erstellen">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw

    # wählen Sie einen eigenen Namen
    fly apps create my-openclaw

    # 1 GB ist normalerweise ausreichend
    fly volumes create openclaw_data --size 1 --region iad
    ```

    Wählen Sie eine Region in Ihrer Nähe. Häufig verwendete Optionen: `lhr` (London), `iad` (Virginia), `sjc` (San Jose).

  </Step>

  <Step title="fly.toml konfigurieren">
    Bearbeiten Sie `fly.toml` entsprechend Ihrem App-Namen und Ihren Anforderungen. Die im Repository verwaltete Datei `fly.toml` ist die unten gezeigte öffentliche Vorlage; `deploy/fly.private.toml` ist die gehärtete Variante ohne öffentliche IP-Adresse (siehe [Private Bereitstellung](#private-deployment-hardened)).

    ```toml
    app = "my-openclaw"  # Ihr App-Name
    primary_region = "iad"

    [build]
      dockerfile = "Dockerfile"

    [env]
      NODE_ENV = "production"
      OPENCLAW_PREFER_PNPM = "1"
      OPENCLAW_STATE_DIR = "/data"
      NODE_OPTIONS = "--max-old-space-size=1536"

    [processes]
      app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"

    [http_service]
      internal_port = 3000
      force_https = true
      auto_stop_machines = false
      auto_start_machines = true
      min_machines_running = 1
      processes = ["app"]

    [[vm]]
      size = "shared-cpu-2x"
      memory = "2048mb"

    [mounts]
      source = "openclaw_data"
      destination = "/data"
    ```

    Der Einstiegspunkt des OpenClaw-Docker-Images ist `tini` und führt standardmäßig `node openclaw.mjs gateway` aus. Fly `[processes]` ersetzt Docker `CMD` (hier wird direkt `node dist/index.js gateway ...`, derselbe kompilierte Einstiegspunkt, ausgeführt), ohne `ENTRYPOINT` zu verändern, sodass der Prozess weiterhin unter `tini` ausgeführt wird.

    **Wichtige Einstellungen:**

    | Einstellung                    | Grund                                                                       |
    | ------------------------------ | --------------------------------------------------------------------------- |
    | `--bind lan`                   | Bindet an `0.0.0.0`, damit der Proxy von Fly das Gateway erreichen kann    |
    | `--allow-unconfigured`         | Startet ohne Konfigurationsdatei (Sie erstellen diese anschließend)         |
    | `internal_port = 3000`         | Muss für Fly-Systemdiagnosen mit `--port 3000` (oder `OPENCLAW_GATEWAY_PORT`) übereinstimmen |
    | `memory = "2048mb"`            | 512 MB sind zu wenig; 2 GB werden empfohlen                                 |
    | `OPENCLAW_STATE_DIR = "/data"` | Speichert den Zustand dauerhaft auf dem Volume                              |

  </Step>

  <Step title="Secrets festlegen">
    ```bash
    # erforderlich: Gateway-Authentifizierungstoken für Bindungen außerhalb von Loopback
    fly secrets set OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)

    # API-Schlüssel der Modell-Provider
    fly secrets set ANTHROPIC_API_KEY=example-anthropic-key-not-real

    # optional: weitere Provider
    fly secrets set OPENAI_API_KEY=example-openai-key-not-real
    fly secrets set GOOGLE_API_KEY=...

    # Kanal-Token
    fly secrets set DISCORD_BOT_TOKEN=example-discord-bot-token
    ```

    Bindungen außerhalb von Loopback (`--bind lan`) erfordern einen gültigen Gateway-Authentifizierungspfad. Dieses Beispiel verwendet `OPENCLAW_GATEWAY_TOKEN`, aber `gateway.auth.password` oder eine korrekt konfigurierte Trusted-Proxy-Bereitstellung außerhalb von Loopback erfüllen die Anforderung ebenfalls. Informationen zum SecretRef-Vertrag finden Sie unter [Secret-Verwaltung](/de/gateway/secrets).

    Behandeln Sie diese Token wie Passwörter. Verwenden Sie für API-Schlüssel und Token vorzugsweise Umgebungsvariablen/`fly secrets` statt der Konfigurationsdatei, damit Secrets nicht in `openclaw.json` gespeichert werden.

  </Step>

  <Step title="Bereitstellen">
    ```bash
    fly deploy
    ```

    Bei der ersten Bereitstellung wird das Docker-Image erstellt. Überprüfen Sie nach der Bereitstellung:

    ```bash
    fly status
    fly logs
    ```

    Beim Start protokolliert das Gateway `gateway ready`, sobald der HTTP-/WebSocket-Listener aktiv ist. Flys eigene Systemdiagnose überwacht gemäß `fly.toml` den Endpunkt `internal_port = 3000`; die Docker-Anweisung `HEALTHCHECK` des Images fragt zusätzlich `/healthz` auf dem Standardport 18789 ab, der hier nicht verwendet wird, da diese Bereitstellung für das Gateway stattdessen `--port 3000` festlegt.

  </Step>

  <Step title="Konfigurationsdatei erstellen">
    Melden Sie sich per SSH bei der Maschine an, um eine ordnungsgemäße Konfiguration zu erstellen:

    ```bash
    fly ssh console
    ```

    ```bash
    mkdir -p /data
    cat > /data/openclaw.json << 'EOF'
    {
      "agents": {
        "defaults": {
          "model": {
            "primary": "anthropic/claude-opus-4-6",
            "fallbacks": ["anthropic/claude-sonnet-4-6", "openai/gpt-5.4"]
          },
          "maxConcurrent": 4
        },
        "list": [
          {
            "id": "main",
            "default": true
          }
        ]
      },
      "auth": {
        "profiles": {
          "anthropic:default": { "mode": "token", "provider": "anthropic" },
          "openai:default": { "mode": "token", "provider": "openai" }
        }
      },
      "bindings": [
        {
          "agentId": "main",
          "match": { "channel": "discord" }
        }
      ],
      "channels": {
        "discord": {
          "enabled": true,
          "groupPolicy": "allowlist",
          "guilds": {
            "YOUR_GUILD_ID": {
              "channels": { "general": { "allow": true } },
              "requireMention": false
            }
          }
        }
      },
      "gateway": {
        "mode": "local",
        "bind": "auto",
        "controlUi": {
          "allowedOrigins": [
            "https://my-openclaw.fly.dev",
            "http://localhost:3000",
            "http://127.0.0.1:3000"
          ]
        }
      },
      "meta": {}
    }
    EOF
    ```

    Mit `OPENCLAW_STATE_DIR=/data` lautet der Konfigurationspfad `/data/openclaw.json`.

    Ersetzen Sie `https://my-openclaw.fly.dev` durch den tatsächlichen Ursprung Ihrer Fly-App. Beim Start übernimmt das Gateway lokale Ursprünge für die Control UI aus den Laufzeitwerten `--bind` und `--port`, sodass der erste Start erfolgen kann, bevor eine Konfiguration vorhanden ist. Für den Browserzugriff über Fly muss der genaue HTTPS-Ursprung dennoch in `gateway.controlUi.allowedOrigins` aufgeführt sein.

    Das Discord-Token kann aus einer der folgenden Quellen stammen:

    - Umgebungsvariable `DISCORD_BOT_TOKEN` (für Secrets empfohlen); sie muss nicht zur Konfiguration hinzugefügt werden, da das Gateway sie automatisch liest
    - Konfigurationsdatei `channels.discord.token`

    Starten Sie die Maschine neu, um die Konfiguration anzuwenden:

    ```bash
    exit
    fly machine restart <machine-id>
    ```

  </Step>

  <Step title="Auf das Gateway zugreifen">
    ### Control UI

    ```bash
    fly open
    ```

    Oder rufen Sie `https://my-openclaw.fly.dev/` auf.

    Authentifizieren Sie sich mit dem konfigurierten gemeinsamen Secret: dem Gateway-Token aus `OPENCLAW_GATEWAY_TOKEN` oder Ihrem Passwort, falls Sie zur Passwortauthentifizierung gewechselt haben.

    ### Protokolle

    ```bash
    fly logs              # Live-Protokolle
    fly logs --no-tail    # aktuelle Protokolle
    ```

    ### SSH-Konsole

    ```bash
    fly ssh console
    ```

  </Step>
</Steps>

## Fehlerbehebung

### „App lauscht nicht an der erwarteten Adresse“

Das Gateway bindet an `127.0.0.1` statt an `0.0.0.0`.

**Lösung:** Fügen Sie dem Prozessbefehl in `fly.toml` die Option `--bind lan` hinzu.

### Fehlgeschlagene Systemdiagnosen / Verbindung abgelehnt

Fly kann das Gateway am konfigurierten Port nicht erreichen.

**Lösung:** Stellen Sie sicher, dass `internal_port` mit dem Gateway-Port (`--port 3000` oder `OPENCLAW_GATEWAY_PORT=3000`) übereinstimmt.

### OOM-/Speicherprobleme

Der Container wird ständig neu gestartet oder beendet. Anzeichen: `SIGABRT`, `v8::internal::Runtime_AllocateInYoungGeneration` oder Neustarts ohne Fehlermeldung.

**Lösung:** Erhöhen Sie den Speicher in `fly.toml`:

```toml
[[vm]]
  memory = "2048mb"
```

Oder aktualisieren Sie eine vorhandene Maschine:

```bash
fly machine update <machine-id> --vm-memory 2048 -y
```

512 MB sind zu wenig. 1 GB kann ausreichen, unter Last oder bei ausführlicher Protokollierung jedoch zu einem OOM-Fehler führen. 2 GB werden empfohlen.

### Probleme mit der Gateway-Sperre

Nach einem Container-Neustart verweigert das Gateway den Start mit Fehlern, die besagen, dass es „bereits ausgeführt wird“.

Die Laufzeitsperrdateien befinden sich unter `<tmpdir>/openclaw-<uid>/gateway.<hash>.lock`
und `gateway.state.<hash>.lock` (Linux:
`/tmp/openclaw-<uid>/gateway.*.lock`), nicht auf dem persistenten Volume `/data`. Daher werden
sie bei einem vollständigen Container-Neustart normalerweise zusammen mit dem übrigen
Container-Dateisystem entfernt. Falls eine Sperre bestehen bleibt (beispielsweise bei einem `fly machine restart`,
der das Container-Dateisystem beibehält) und den Start blockiert, entfernen Sie sie
manuell:

```bash
fly ssh console --command "rm -f /tmp/openclaw-*/gateway.*.lock"
fly machine restart <machine-id>
```

### Konfiguration wird nicht gelesen

`--allow-unconfigured` umgeht lediglich die Startsperre. Es erstellt oder repariert `/data/openclaw.json` nicht. Stellen Sie daher sicher, dass Ihre tatsächliche Konfiguration vorhanden ist und für einen normalen lokalen Gateway-Start `"gateway": { "mode": "local" }` enthält.

Überprüfen Sie, ob die Konfiguration vorhanden ist:

```bash
fly ssh console --command "cat /data/openclaw.json"
```

### Konfiguration per SSH schreiben

`fly ssh console -C` unterstützt keine Shell-Umleitung. So schreiben Sie eine Konfigurationsdatei:

```bash
# echo + tee (Pipe vom lokalen zum entfernten System)
echo '{"your":"config"}' | fly ssh console -C "tee /data/openclaw.json"

# oder sftp
fly sftp shell
> put /local/path/config.json /data/openclaw.json
```

`fly sftp` kann fehlschlagen, wenn die Datei bereits vorhanden ist; löschen Sie sie zuerst:

```bash
fly ssh console --command "rm /data/openclaw.json"
```

### Zustand bleibt nicht erhalten

Wenn Authentifizierungsprofile, Kanal-/Provider-Zustand oder Sitzungen nach einem Neustart verloren gehen, wird das Zustandsverzeichnis in das Container-Dateisystem statt auf das Volume geschrieben.

**Lösung:** Stellen Sie sicher, dass `OPENCLAW_STATE_DIR=/data` in `fly.toml` festgelegt ist, und stellen Sie erneut bereit.

## Aktualisieren

```bash
git pull
fly deploy
fly status
fly logs
```

`git pull` + `fly deploy` ist hier der überwachte Pfad: Dadurch wird das Image aus dem Dockerfile neu erstellt, sodass die CLI-/Gateway-Version, das Basis-Betriebssystem-Image und alle Änderungen am Dockerfile gemeinsam aktualisiert werden. `openclaw update` innerhalb des laufenden Containers ist nicht derselbe Vorgang, da das Image als von Docker erstellter `dist/`-Baum ohne `.git`-Checkout und ohne eine von npm verwaltete globale Installation ausgeliefert wird, die dieser Vorgang erkennen könnte. Informationen zu diesem Ablauf bei VM-ähnlichen Installationen finden Sie unter [Aktualisieren](/de/install/updating).

### Maschinenbefehl aktualisieren

So ändern Sie den Startbefehl ohne vollständige erneute Bereitstellung:

```bash
fly machines list
fly machine update <machine-id> --command "node dist/index.js gateway --port 3000 --bind lan" -y

# oder mit einer Speichererhöhung
fly machine update <machine-id> --vm-memory 2048 --command "node dist/index.js gateway --port 3000 --bind lan" -y
```

Ein späterer `fly deploy` setzt den Maschinenbefehl auf den Inhalt von `fly.toml` zurück. Wenden Sie manuelle Änderungen nach einer erneuten Bereitstellung erneut an.

## Private Bereitstellung (gehärtet)

Standardmäßig weist Fly öffentliche IP-Adressen zu. Dadurch ist Ihr Gateway unter `https://your-app.fly.dev` erreichbar und kann von Internet-Scannern wie Shodan, Censys usw. gefunden werden.

Verwenden Sie `deploy/fly.private.toml` für eine gehärtete Bereitstellung **ohne öffentliche IP-Adresse**: Da `[http_service]` ausgelassen wird, wird kein öffentlicher Eingang zugewiesen.

### Wann eine private Bereitstellung verwendet werden sollte

- Nur ausgehende Aufrufe/Nachrichten (keine eingehenden Webhooks)
- ngrok- oder Tailscale-Tunnel verarbeiten alle Webhook-Rückrufe
- Der Zugriff auf das Gateway erfolgt über SSH, Proxy oder WireGuard statt über einen Browser
- Die Bereitstellung soll vor Internet-Scannern verborgen bleiben

### Einrichtung

```bash
fly deploy -c deploy/fly.private.toml
```

Oder konvertieren Sie eine vorhandene Bereitstellung:

```bash
# aktuelle IPs auflisten
fly ips list -a my-openclaw

# öffentliche IPs freigeben
fly ips release <public-ipv4> -a my-openclaw
fly ips release <public-ipv6> -a my-openclaw

# zur privaten Konfiguration wechseln, damit zukünftige Bereitstellungen keine öffentlichen IPs erneut zuweisen
fly deploy -c deploy/fly.private.toml

# ausschließlich private IPv6-Adresse zuweisen
fly ips allocate-v6 --private -a my-openclaw
```

Danach sollte `fly ips list` nur eine IP vom Typ `private` anzeigen:

```text
VERSION  IP                   TYP              REGION
v6       fdaa:x:x:x:x::x      privat           global
```

### Zugriff auf eine private Bereitstellung

**Option 1: Lokaler Proxy (am einfachsten)**

```bash
fly proxy 3000:3000 -a my-openclaw
# http://localhost:3000 in einem Browser öffnen
```

**Option 2: WireGuard-VPN**

```bash
fly wireguard create
# in einen WireGuard-Client importieren und dann über die interne IPv6-Adresse zugreifen
# Beispiel: http://[fdaa:x:x:x:x::x]:3000
```

**Option 3: Nur SSH**

```bash
fly ssh console -a my-openclaw
```

### Webhooks bei einer privaten Bereitstellung

Für Webhook-Rückrufe (Twilio, Telnyx usw.) ohne öffentliche Erreichbarkeit:

1. **ngrok-Tunnel**: ngrok im Container oder als Sidecar ausführen
2. **Tailscale Funnel**: bestimmte Pfade über Tailscale öffentlich bereitstellen
3. **Nur ausgehend**: Einige Provider (Twilio) unterstützen ausgehende Anrufe ohne Webhooks

Beispielkonfiguration für Sprachanrufe mit ngrok unter `plugins.entries.voice-call.config`:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          tunnel: { provider: "ngrok" },
          webhookSecurity: {
            allowedHosts: ["example.ngrok.app"],
          },
        },
      },
    },
  },
}
```

Der ngrok-Tunnel wird im Container ausgeführt und stellt eine öffentliche Webhook-URL bereit, ohne die Fly-App selbst öffentlich zugänglich zu machen. Setzen Sie `webhookSecurity.allowedHosts` auf den Hostnamen des Tunnels, damit weitergeleitete Host-Header akzeptiert werden.

### Sicherheitsabwägungen

| Aspekt              | Öffentlich      | Privat         |
| ------------------- | --------------- | -------------- |
| Internet-Scanner    | Auffindbar       | Verborgen      |
| Direkte Angriffe    | Möglich          | Blockiert      |
| Zugriff auf die Steuerungsoberfläche | Browser         | Proxy/VPN      |
| Webhook-Zustellung  | Direkt           | Über Tunnel    |

## Hinweise

- Fly.io verwendet die x86-Architektur; das Dockerfile ist sowohl mit x86 als auch mit ARM kompatibel.
- Verwenden Sie für das Onboarding von WhatsApp/Telegram `fly ssh console`.
- Persistente Daten befinden sich auf dem Volume unter `/data`.
- Signal benötigt signal-cli (eine Java-basierte CLI) im Image; verwenden Sie ein benutzerdefiniertes Image und mindestens 2 GB Arbeitsspeicher.

## Kosten

Mit der empfohlenen Konfiguration (`shared-cpu-2x`, 2 GB RAM) können Sie je nach Nutzung mit ungefähr 10–15 $ pro Monat rechnen; der kostenlose Tarif deckt ein gewisses Grundkontingent ab. Die aktuellen Preise finden Sie unter [Fly.io-Preise](https://fly.io/docs/about/pricing/).

## Nächste Schritte

- Messaging-Kanäle einrichten: [Kanäle](/de/channels)
- Gateway konfigurieren: [Gateway-Konfiguration](/de/gateway/configuration)
- OpenClaw auf dem neuesten Stand halten: [Aktualisierung](/de/install/updating)

## Verwandte Themen

- [Installationsübersicht](/de/install)
- [Hetzner](/de/install/hetzner)
- [Docker](/de/install/docker)
- [VPS-Hosting](/de/vps)
