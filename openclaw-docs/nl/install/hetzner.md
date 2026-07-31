---
read_when:
    - Je wilt dat OpenClaw 24/7 draait op een cloud-VPS (niet op je laptop)
    - Je wilt een productieklare, permanent actieve Gateway op je eigen VPS
    - Je wilt volledige controle over persistentie, binaire bestanden en herstartgedrag
    - Je voert OpenClaw uit in Docker bij Hetzner of een vergelijkbare provider
summary: Voer OpenClaw Gateway 24/7 uit op een goedkope Hetzner-VPS (Docker), met duurzame status en ingebouwde binaire bestanden
title: Hetzner
x-i18n:
    generated_at: "2026-07-27T05:49:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ffebc0ce725fd219d13d0a556940327e70dab810b8fbee0b365c4870dc7109b
    source_path: install/hetzner.md
    workflow: 16
---

Voer een permanente OpenClaw Gateway uit op een Hetzner-VPS met Docker, met duurzame status, ingebouwde binaire bestanden en veilig herstartgedrag.

De prijzen van Hetzner veranderen; kies de kleinste Debian/Ubuntu-VPS die voldoet en schaal op als er OOM-fouten optreden.

De Gateway is toegankelijk via SSH-poortdoorsturing vanaf je laptop, of via directe openstelling van de poort als je zelf de firewall en tokens beheert.

Herinnering over het beveiligingsmodel:

- Door het bedrijf gedeelde agents zijn prima wanneer iedereen zich binnen dezelfde vertrouwensgrens bevindt en de runtime uitsluitend zakelijk wordt gebruikt.
- Houd een strikte scheiding aan: een speciale VPS/runtime en speciale accounts; geen persoonlijke Apple-/Google-/browser-/wachtwoordbeheerderprofielen op die host.
- Als gebruikers elkaar niet vertrouwen, splits ze dan op per gateway/host/OS-gebruiker.

Zie [Beveiliging](/nl/gateway/security) en [VPS-hosting](/nl/vps).

Deze handleiding gaat uit van Ubuntu of Debian op Hetzner. Stem de pakketten dienovereenkomstig af voor een andere Linux-VPS. Zie [Docker](/nl/install/docker) voor de algemene Docker-procedure.

## Wat je nodig hebt

- Hetzner-VPS met roottoegang
- SSH-toegang vanaf je laptop
- Docker en Docker Compose
- Authenticatiegegevens voor het model
- Optionele providergegevens (WhatsApp-QR-code, Telegram-bottoken, Gmail OAuth)
- Ongeveer 20 minuten

## Snelle procedure

1. Maak een Hetzner-VPS aan
2. Installeer Docker
3. Kloon de OpenClaw-repository
4. Maak permanente hostmappen
5. Configureer `.env` en `docker-compose.yml`
6. Bouw vereiste binaire bestanden in de image in
7. `docker compose up -d`
8. Controleer persistentie en toegang tot de Gateway

<Steps>
  <Step title="Maak de VPS aan">
    Maak een Ubuntu- of Debian-VPS aan bij Hetzner en maak vervolgens als root verbinding:

    ```bash
    ssh root@YOUR_VPS_IP
    ```

    Behandel de VPS als stateful infrastructuur, niet als wegwerpinfrastructuur.

  </Step>

  <Step title="Installeer Docker (op de VPS)">
    ```bash
    apt-get update
    apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sh
    ```

    Controleer:

    ```bash
    docker --version
    docker compose version
    ```

  </Step>

  <Step title="Kloon de OpenClaw-repository">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    ```

    Deze handleiding bouwt een aangepaste image, zodat alle ingebouwde binaire bestanden behouden blijven na herstarts.

  </Step>

  <Step title="Maak permanente hostmappen">
    Docker-containers zijn tijdelijk; alle langdurige status moet op de host worden opgeslagen.

    ```bash
    mkdir -p /root/.openclaw/workspace

    # Stel de eigenaar in op de containergebruiker (uid 1000):
    chown -R 1000:1000 /root/.openclaw
    ```

  </Step>

  <Step title="Configureer omgevingsvariabelen">
    Maak `.env` in de hoofdmap van de repository:

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

    Stel `OPENCLAW_GATEWAY_TOKEN` in om het stabiele gatewaytoken te beheren via
    `.env`; configureer anders `gateway.auth.token` voordat je erop vertrouwt dat clients
    na herstarts verbonden blijven. Als geen van beide is ingesteld, gebruikt OpenClaw alleen voor
    die opstart een runtime-token. Genereer een sleutelringwachtwoord voor `GOG_KEYRING_PASSWORD`:

    ```bash
    openssl rand -hex 32
    ```

    **Commit dit bestand niet.** Het bevat omgevingsvariabelen voor de container/runtime, zoals
    `OPENCLAW_GATEWAY_TOKEN`. Opgeslagen OAuth-/API-sleutelauthenticatie van providers bevindt zich in de
    gekoppelde `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`.

  </Step>

  <Step title="Docker Compose-configuratie">
    Maak of werk `docker-compose.yml` bij:

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
          # Aanbevolen: houd de Gateway op de VPS beperkt tot de loopback-interface; gebruik een SSH-tunnel voor toegang.
          # Verwijder het voorvoegsel `127.0.0.1:` om deze openbaar beschikbaar te maken en configureer de firewall dienovereenkomstig.
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

    `--allow-unconfigured` is alleen bedoeld voor eenvoudig opstarten en vervangt geen echte gatewayconfiguratie. Stel nog steeds authenticatie (`gateway.auth.token` of een wachtwoord) en een veilige bindingsmodus in voor je implementatie.

  </Step>

  <Step title="Gedeelde runtimestappen voor een Docker-VM">
    Volg de gedeelde runtimehandleiding voor de algemene Docker-hostprocedure:

    - [Bouw vereiste binaire bestanden in de image in](/nl/install/docker-vm-runtime#bake-required-binaries-into-the-image)
    - [Bouwen en starten](/nl/install/docker-vm-runtime#build-and-launch)
    - [Wat waar permanent wordt opgeslagen](/nl/install/docker-vm-runtime#what-persists-where)
    - [Updates](/nl/install/docker-vm-runtime#updates)

  </Step>

  <Step title="Hetzner-specifieke toegang">
    Open de tunnel na de gedeelde stappen voor bouwen en starten.

    **Voorwaarde:** controleer of de sshd-configuratie van je VPS TCP-doorsturing toestaat. Als je
    de SSH-configuratie hebt aangescherpt, controleer dan `/etc/ssh/sshd_config` en stel het volgende in:

    ```text
    AllowTcpForwarding local
    ```

    `local` staat lokale `ssh -L`-doorsturing vanaf je laptop toe en blokkeert
    tegelijkertijd externe doorsturing vanaf de server. Als je dit instelt op `no`, mislukt de tunnel met:
    `channel 3: open failed: administratively prohibited: open failed`

    Start na bevestiging dat TCP-doorsturing is ingeschakeld de SSH-service
    (`systemctl restart ssh`) opnieuw en voer de tunnel uit vanaf je laptop:

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP
    ```

    Open `http://127.0.0.1:18789/` en plak het geconfigureerde gedeelde geheim.
    Deze handleiding gebruikt standaard het gatewaytoken; gebruik in plaats daarvan
    je geconfigureerde wachtwoord als je bent overgeschakeld op wachtwoordauthenticatie.

  </Step>
</Steps>

De gedeelde persistentietoewijzing staat in [Docker-VM-runtime](/nl/install/docker-vm-runtime#what-persists-where).

## Infrastructure as Code (Terraform)

Voor teams die de voorkeur geven aan infrastructure-as-code-workflows, biedt een door de community onderhouden Terraform-configuratie:

- Modulaire Terraform-configuratie met extern statusbeheer
- Geautomatiseerde inrichting via cloud-init
- Implementatiescripts (bootstrap, implementatie, back-up/herstel)
- Beveiligingsversterking (firewall, UFW, uitsluitend SSH-toegang)
- SSH-tunnelconfiguratie voor toegang tot de gateway

**Repository's:**

- Infrastructuur: [openclaw-terraform-hetzner](https://github.com/andreesg/openclaw-terraform-hetzner)
- Docker-configuratie: [openclaw-docker-config](https://github.com/andreesg/openclaw-docker-config)

Deze aanpak vult de bovenstaande Docker-configuratie aan met reproduceerbare implementaties, infrastructuur onder versiebeheer en geautomatiseerd noodherstel.

<Note>
Onderhouden door de community. Raadpleeg de bovenstaande repositorylinks voor problemen of bijdragen.
</Note>

## Volgende stappen

- Stel berichtenkanalen in: [Kanalen](/nl/channels)
- Configureer de Gateway: [Gateway-configuratie](/nl/gateway/configuration)
- Houd OpenClaw up-to-date: [Bijwerken](/nl/install/updating)

## Gerelateerd

- [Installatieoverzicht](/nl/install)
- [Fly.io](/nl/install/fly)
- [Docker](/nl/install/docker)
- [VPS-hosting](/nl/vps)
