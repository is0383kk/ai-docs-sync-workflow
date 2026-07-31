---
read_when:
    - OpenClaw instellen op DigitalOcean
    - Op zoek naar een eenvoudige betaalde VPS voor OpenClaw
summary: OpenClaw hosten op een DigitalOcean Droplet
title: DigitalOcean
x-i18n:
    generated_at: "2026-07-27T05:57:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e124a59c079efda0c8e880018f2657fad784af1489ca3f98ed8ab609249e35bd
    source_path: install/digitalocean.md
    workflow: 16
---

Voer een permanente OpenClaw Gateway uit op een DigitalOcean Droplet (~$6/maand voor het Basic-abonnement van 1 GB).

DigitalOcean is een eenvoudige betaalde VPS-optie. Voor goedkopere of gratis opties:

- [Hetzner](/nl/install/hetzner) -- meer cores/RAM per dollar.
- [Oracle Cloud](/nl/install/oracle) -- Always Free ARM-niveau (maximaal 4 OCPU, 24 GB RAM), maar registratie kan lastig zijn en het werkt alleen met ARM.

## Vereisten

- DigitalOcean-account ([registreren](https://cloud.digitalocean.com/registrations/new))
- SSH-sleutelpaar (of bereidheid om wachtwoordauthenticatie te gebruiken)
- Ongeveer 20 minuten

## Installatie

<Steps>
  <Step title="Een Droplet maken">
    <Warning>
    Gebruik een schone basisimage (Ubuntu 24.04 LTS). Vermijd 1-click-images van derden uit de Marketplace, tenzij je hun opstartscripts en standaardfirewallinstellingen hebt gecontroleerd.
    </Warning>

    1. Meld je aan bij [DigitalOcean](https://cloud.digitalocean.com/).
    2. Klik op **Create > Droplets**.
    3. Kies:
       - **Region:** de regio die het dichtst bij je ligt
       - **Image:** Ubuntu 24.04 LTS
       - **Size:** Basic, Regular, 1 vCPU / 1 GB RAM / 25 GB SSD
       - **Authentication:** SSH key (aanbevolen) of password
    4. Klik op **Create Droplet** en noteer het IP-adres.

  </Step>

  <Step title="Verbinding maken en installeren">
    ```bash
    ssh root@YOUR_DROPLET_IP

    apt update && apt upgrade -y

    # Node.js 24 installeren
    curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
    apt install -y nodejs

    # OpenClaw installeren
    curl -fsSL https://openclaw.ai/install.sh | bash

    # De niet-rootgebruiker maken die eigenaar wordt van de OpenClaw-status en -services.
    adduser openclaw
    usermod -aG sudo openclaw
    loginctl enable-linger openclaw

    su - openclaw
    openclaw --version
    ```

    Gebruik de rootshell alleen voor de initiële systeemconfiguratie. Voer OpenClaw-opdrachten uit als de niet-rootgebruiker `openclaw`, zodat de status onder `/home/openclaw/.openclaw/` wordt opgeslagen en de Gateway als systemd-`--user`-service van die gebruiker wordt geïnstalleerd.

  </Step>

  <Step title="De onboarding uitvoeren">
    ```bash
    openclaw onboard --install-daemon
    ```

    De wizard begeleidt je bij modelauthenticatie, kanaalconfiguratie, het genereren van een gatewaytoken en de installatie van de daemon (systemd-gebruikersservice).

  </Step>

  <Step title="Swap toevoegen (aanbevolen voor Droplets van 1 GB)">
    ```bash
    fallocate -l 2G /swapfile
    chmod 600 /swapfile
    mkswap /swapfile
    swapon /swapfile
    echo '/swapfile none swap sw 0 0' >> /etc/fstab
    ```
  </Step>

  <Step title="De Gateway verifiëren">
    ```bash
    openclaw status
    systemctl --user status openclaw-gateway.service
    journalctl --user -u openclaw-gateway.service -f
    ```
  </Step>

  <Step title="Toegang tot de bedieningsinterface">
    De Gateway luistert standaard alleen op de loopbackinterface. Kies een van deze opties.

    **Optie A: SSH-tunnel (eenvoudigst)**

    ```bash
    # Vanaf je lokale computer
    ssh -L 18789:localhost:18789 root@YOUR_DROPLET_IP
    ```

    Open vervolgens `http://localhost:18789`.

    **Optie B: Tailscale Serve**

    ```bash
    curl -fsSL https://tailscale.com/install.sh | sudo sh
    sudo tailscale up
    openclaw config set gateway.tailscale.mode serve
    openclaw gateway restart
    ```

    Open vervolgens `https://<magicdns>/` vanaf elk apparaat op je tailnet.

    Tailscale Serve verifieert verkeer van de bedieningsinterface en WebSocket-verkeer via identiteitsheaders van het tailnet. Daarbij wordt aangenomen dat de gatewayhost zelf wordt vertrouwd. HTTP-API-eindpunten volgen desondanks nog steeds de normale authenticatiemodus van de Gateway (token/wachtwoord). Stel `gateway.auth.allowTailscale: false` in en gebruik `gateway.auth.mode: "token"` of `"password"` om expliciete gedeelde geheime referenties via Serve te vereisen.

    **Optie C: Binden aan tailnet (zonder Serve)**

    ```bash
    openclaw config set gateway.bind tailnet
    openclaw gateway restart
    ```

    Open vervolgens `http://<tailscale-ip>:18789` (token vereist).

  </Step>
</Steps>

## Persistentie en back-ups

De OpenClaw-status wordt opgeslagen onder:

- `~/.openclaw/` -- `openclaw.json`, kanaal-/providerreferenties, `auth-profiles.json` per agent en sessiegegevens.
- `~/.openclaw/workspace/` -- de agentwerkruimte (SOUL.md, geheugen, artefacten).

Deze gegevens blijven behouden wanneer de Droplet opnieuw wordt opgestart. Zo maak je een overdraagbare momentopname:

```bash
openclaw backup create
```

DigitalOcean-momentopnamen maken een back-up van de volledige Droplet; `openclaw backup create` is overdraagbaar tussen hosts.

## Tips voor 1 GB RAM

De Droplet van $6 heeft slechts 1 GB RAM. Zo blijft alles soepel werken:

- Zorg ervoor dat de bovenstaande swapstap in `/etc/fstab` staat, zodat deze na opnieuw opstarten behouden blijft.
- Geef de voorkeur aan API-gebaseerde modellen (Claude, GPT) boven lokale modellen -- lokale LLM-inferentie past niet binnen 1 GB.
- Stel `agents.defaults.model.primary` in op een kleiner model als bij grote prompts OOM-fouten optreden.
- Bewaak het systeem met `free -h` en `htop`.

## Problemen oplossen

**Gateway start niet** -- Voer `openclaw doctor --non-interactive` uit en controleer de logboeken met `journalctl --user -u openclaw-gateway.service -n 50`.

**Poort is al in gebruik** -- Voer `lsof -i :18789` uit om het proces te vinden en stop het vervolgens.

**Onvoldoende geheugen** -- Controleer met `free -h` of swap actief is. Als er nog steeds OOM-fouten optreden, schakel je over op API-gebaseerde modellen (Claude, GPT) in plaats van lokale modellen, of upgrade je naar een Droplet van 2 GB.

## Volgende stappen

- [Kanalen](/nl/channels) -- verbind Telegram, WhatsApp, Discord en meer
- [Gateway-configuratie](/nl/gateway/configuration) -- alle configuratieopties
- [Bijwerken](/nl/install/updating) -- houd OpenClaw up-to-date

## Gerelateerd

- [Installatieoverzicht](/nl/install)
- [Fly.io](/nl/install/fly)
- [Hetzner](/nl/install/hetzner)
- [VPS-hosting](/nl/vps)
