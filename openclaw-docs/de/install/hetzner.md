---
read_when:
    - Sie möchten OpenClaw rund um die Uhr auf einem Cloud-VPS ausführen (nicht auf Ihrem Laptop).
    - Sie möchten ein produktionsreifes, dauerhaft verfügbares Gateway auf Ihrem eigenen VPS.
    - Sie möchten vollständige Kontrolle über Persistenz, Binärdateien und das Neustartverhalten.
    - Sie führen OpenClaw in Docker bei Hetzner oder einem ähnlichen Provider aus
summary: OpenClaw Gateway rund um die Uhr auf einem günstigen Hetzner-VPS (Docker) mit dauerhaftem Zustand und integrierten Binärdateien ausführen
title: Hetzner
x-i18n:
    generated_at: "2026-07-26T18:30:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ffebc0ce725fd219d13d0a556940327e70dab810b8fbee0b365c4870dc7109b
    source_path: install/hetzner.md
    workflow: 16
---

Führen Sie einen persistenten OpenClaw Gateway auf einem Hetzner-VPS mit Docker aus – mit dauerhaftem Zustand, fest integrierten Binärdateien und sicherem Neustartverhalten.

Die Hetzner-Preise ändern sich; wählen Sie den kleinsten geeigneten Debian-/Ubuntu-VPS und skalieren Sie ihn bei OOM-Fehlern nach oben.

Auf den Gateway kann von Ihrem Laptop aus über SSH-Portweiterleitung oder durch direkte Portfreigabe zugegriffen werden, wenn Sie Firewall und Tokens selbst verwalten.

Hinweis zum Sicherheitsmodell:

- Gemeinsam im Unternehmen genutzte Agenten sind in Ordnung, wenn sich alle innerhalb derselben Vertrauensgrenze befinden und die Laufzeit ausschließlich geschäftlich genutzt wird.
- Sorgen Sie für eine strikte Trennung: dedizierter VPS/dedizierte Laufzeit und dedizierte Konten; keine persönlichen Apple-/Google-/Browser-/Passwortmanager-Profile auf diesem Host.
- Wenn Benutzer einander gegenüber potenziell böswillig sind, trennen Sie sie nach Gateway/Host/Betriebssystembenutzer.

Siehe [Sicherheit](/de/gateway/security) und [VPS-Hosting](/de/vps).

Diese Anleitung setzt Ubuntu oder Debian auf Hetzner voraus. Passen Sie die Pakete auf einem anderen Linux-VPS entsprechend an. Den allgemeinen Docker-Ablauf finden Sie unter [Docker](/de/install/docker).

## Voraussetzungen

- Hetzner-VPS mit Root-Zugriff
- SSH-Zugriff von Ihrem Laptop
- Docker und Docker Compose
- Anmeldedaten für die Modellauthentifizierung
- Optionale Provider-Anmeldedaten (WhatsApp-QR-Code, Telegram-Bot-Token, Gmail OAuth)
- ca. 20 Minuten

## Schnellverfahren

1. Hetzner-VPS bereitstellen
2. Docker installieren
3. OpenClaw-Repository klonen
4. Persistente Host-Verzeichnisse erstellen
5. `.env` und `docker-compose.yml` konfigurieren
6. Erforderliche Binärdateien fest in das Image integrieren
7. `docker compose up -d`
8. Persistenz und Gateway-Zugriff überprüfen

<Steps>
  <Step title="VPS bereitstellen">
    Erstellen Sie bei Hetzner einen Ubuntu- oder Debian-VPS und stellen Sie anschließend als Root eine Verbindung her:

    ```bash
    ssh root@YOUR_VPS_IP
    ```

    Behandeln Sie den VPS als zustandsbehaftete und nicht als kurzlebige Infrastruktur.

  </Step>

  <Step title="Docker installieren (auf dem VPS)">
    ```bash
    apt-get update
    apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sh
    ```

    Überprüfen Sie die Installation:

    ```bash
    docker --version
    docker compose version
    ```

  </Step>

  <Step title="OpenClaw-Repository klonen">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    ```

    Diese Anleitung erstellt ein benutzerdefiniertes Image, damit alle darin integrierten Binärdateien Neustarts überstehen.

  </Step>

  <Step title="Persistente Host-Verzeichnisse erstellen">
    Docker-Container sind kurzlebig; alle dauerhaft benötigten Zustandsdaten müssen auf dem Host gespeichert werden.

    ```bash
    mkdir -p /root/.openclaw/workspace

    # Eigentümer auf den Containerbenutzer festlegen (uid 1000):
    chown -R 1000:1000 /root/.openclaw
    ```

  </Step>

  <Step title="Umgebungsvariablen konfigurieren">
    Erstellen Sie `.env` im Stammverzeichnis des Repositorys:

    ```bash
    OPENCLAW_IMAGE=openclaw:latest
    OPENCLAW_GATEWAY_TOKEN=
    OPENCLAW_GATEWAY_BIND=lan
    OPENCLAW_GATEWAY_PORT=18789

    OPENCLAW_CONFIG_DIR=/root/.openclaw
    OPENCLAW_WORKSPACE_DIR=/root/.openclaw/workspace

    GOG_KEYRING_PASSWORD=
    XDG_CONFIG_HOME=/home/node/.openclaw
    ```

    Legen Sie `OPENCLAW_GATEWAY_TOKEN` fest, um das stabile Gateway-Token über
    `.env` zu verwalten; konfigurieren Sie andernfalls `gateway.auth.token`, bevor Sie sich bei Clients
    auf die Verfügbarkeit nach Neustarts verlassen. Wenn keines von beiden festgelegt ist, verwendet OpenClaw für
    diesen Start ein ausschließlich zur Laufzeit gültiges Token. Generieren Sie ein Schlüsselbund-Passwort für `GOG_KEYRING_PASSWORD`:

    ```bash
    openssl rand -hex 32
    ```

    **Committen Sie diese Datei nicht.** Sie enthält Umgebungsvariablen des Containers/der Laufzeit wie
    `OPENCLAW_GATEWAY_TOKEN`. Gespeicherte OAuth-/API-Schlüssel-Authentifizierungsdaten der Provider befinden sich im
    eingebundenen `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`.

  </Step>

  <Step title="Docker-Compose-Konfiguration">
    Erstellen oder aktualisieren Sie `docker-compose.yml`:

    ```yaml
    services:
      openclaw-gateway:
        image: ${OPENCLAW_IMAGE}
        build: .
        restart: unless-stopped
        env_file:
          - .env
        environment:
          - HOME=/home/node
          - NODE_ENV=production
          - TERM=xterm-256color
          - OPENCLAW_GATEWAY_BIND=${OPENCLAW_GATEWAY_BIND}
          - OPENCLAW_GATEWAY_PORT=${OPENCLAW_GATEWAY_PORT}
          - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
          - GOG_KEYRING_PASSWORD=${GOG_KEYRING_PASSWORD}
          - XDG_CONFIG_HOME=${XDG_CONFIG_HOME}
          - PATH=/home/linuxbrew/.linuxbrew/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        volumes:
          - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
          - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
        ports:
          # Empfohlen: Beschränken Sie den Gateway auf dem VPS auf die Loopback-Schnittstelle; greifen Sie über einen SSH-Tunnel darauf zu.
          # Um ihn öffentlich zugänglich zu machen, entfernen Sie das Präfix `127.0.0.1:` und konfigurieren Sie die Firewall entsprechend.
          - "127.0.0.1:${OPENCLAW_GATEWAY_PORT}:18789"
        command:
          [
            "node",
            "dist/index.js",
            "gateway",
            "--bind",
            "${OPENCLAW_GATEWAY_BIND}",
            "--port",
            "${OPENCLAW_GATEWAY_PORT}",
            "--allow-unconfigured",
          ]
    ```

    `--allow-unconfigured` dient nur zur Vereinfachung der Ersteinrichtung und ersetzt keine ordnungsgemäße Gateway-Konfiguration. Legen Sie dennoch eine Authentifizierung (`gateway.auth.token` oder Passwort) sowie einen sicheren Bindungsmodus für Ihre Bereitstellung fest.

  </Step>

  <Step title="Gemeinsame Laufzeitschritte für Docker-VMs">
    Folgen Sie der gemeinsamen Laufzeitanleitung für den allgemeinen Ablauf auf einem Docker-Host:

    - [Erforderliche Binärdateien fest in das Image integrieren](/de/install/docker-vm-runtime#bake-required-binaries-into-the-image)
    - [Erstellen und starten](/de/install/docker-vm-runtime#build-and-launch)
    - [Was wo dauerhaft gespeichert wird](/de/install/docker-vm-runtime#what-persists-where)
    - [Aktualisierungen](/de/install/docker-vm-runtime#updates)

  </Step>

  <Step title="Hetzner-spezifischer Zugriff">
    Öffnen Sie nach den gemeinsamen Schritten zum Erstellen und Starten den Tunnel.

    **Voraussetzung:** Stellen Sie sicher, dass die sshd-Konfiguration Ihres VPS TCP-Weiterleitung zulässt. Wenn Sie
    Ihre SSH-Konfiguration gehärtet haben, überprüfen Sie `/etc/ssh/sshd_config` und legen Sie Folgendes fest:

    ```text
    AllowTcpForwarding local
    ```

    `local` erlaubt lokale `ssh -L`-Weiterleitungen von Ihrem Laptop und blockiert gleichzeitig
    entfernte Weiterleitungen vom Server. Wenn Sie den Wert auf `no` setzen, schlägt der Tunnel mit folgender Meldung fehl:
    `channel 3: open failed: administratively prohibited: open failed`

    Nachdem Sie bestätigt haben, dass die TCP-Weiterleitung aktiviert ist, starten Sie den SSH-Dienst
    (`systemctl restart ssh`) neu und führen Sie den Tunnel von Ihrem Laptop aus:

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP
    ```

    Öffnen Sie `http://127.0.0.1:18789/` und fügen Sie das konfigurierte gemeinsame Geheimnis ein.
    Diese Anleitung verwendet standardmäßig das Gateway-Token; verwenden Sie stattdessen Ihr konfiguriertes Passwort,
    wenn Sie zur Passwortauthentifizierung gewechselt haben.

  </Step>
</Steps>

Die gemeinsame Persistenzübersicht finden Sie unter [Docker-VM-Laufzeit](/de/install/docker-vm-runtime#what-persists-where).

## Infrastructure as Code (Terraform)

Für Teams, die Infrastructure-as-Code-Abläufe bevorzugen, bietet eine von der Community gepflegte Terraform-Konfiguration Folgendes:

- Modulare Terraform-Konfiguration mit Remote-Zustandsverwaltung
- Automatisierte Bereitstellung über cloud-init
- Bereitstellungsskripte (Bootstrap, Bereitstellung, Sicherung/Wiederherstellung)
- Sicherheitshärtung (Firewall, UFW, ausschließlicher SSH-Zugriff)
- SSH-Tunnelkonfiguration für den Gateway-Zugriff

**Repositorys:**

- Infrastruktur: [openclaw-terraform-hetzner](https://github.com/andreesg/openclaw-terraform-hetzner)
- Docker-Konfiguration: [openclaw-docker-config](https://github.com/andreesg/openclaw-docker-config)

Dieser Ansatz ergänzt die oben beschriebene Docker-Konfiguration um reproduzierbare Bereitstellungen, versionsverwaltete Infrastruktur und automatisierte Notfallwiederherstellung.

<Note>
Von der Community gepflegt. Informationen zu Problemen oder Beiträgen finden Sie unter den obigen Repository-Links.
</Note>

## Nächste Schritte

- Messaging-Kanäle einrichten: [Kanäle](/de/channels)
- Gateway konfigurieren: [Gateway-Konfiguration](/de/gateway/configuration)
- OpenClaw aktuell halten: [Aktualisieren](/de/install/updating)

## Verwandte Themen

- [Installationsübersicht](/de/install)
- [Fly.io](/de/install/fly)
- [Docker](/de/install/docker)
- [VPS-Hosting](/de/vps)
