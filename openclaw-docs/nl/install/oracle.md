---
read_when:
    - OpenClaw instellen op Oracle Cloud
    - Op zoek naar gratis VPS-hosting voor OpenClaw
    - Wil je OpenClaw 24/7 op een kleine server gebruiken
summary: Host OpenClaw op de Always Free ARM-laag van Oracle Cloud
title: Oracle Cloud
x-i18n:
    generated_at: "2026-07-27T05:19:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5e1eb95b6bc8ad73e1492a03d8ebe32d89c80e58347614e6ae12d2d3d926d577
    source_path: install/oracle.md
    workflow: 16
---

Voer kosteloos een permanente OpenClaw Gateway uit op de **Always Free** ARM-laag van Oracle Cloud (maximaal 4 OCPU, 24 GB RAM en 200 GB opslag).

## Vereisten

- Oracle Cloud-account ([registreren](https://www.oracle.com/cloud/free/)) -- raadpleeg de [communityhandleiding voor registratie](https://gist.github.com/rssnyder/51e3cfedd730e7dd5f4a816143b25dbd) als je problemen ondervindt
- Tailscale-account (gratis op [tailscale.com](https://tailscale.com))
- Een SSH-sleutelpaar
- Ongeveer 30 minuten

## Installatie

<Steps>
  <Step title="Een OCI-instance maken">
    1. Meld je aan bij de [Oracle Cloud Console](https://cloud.oracle.com/).
    2. Ga naar **Compute > Instances > Create Instance**.
    3. Configureer:
       - **Naam:** `openclaw`
       - **Image:** Ubuntu 24.04 (aarch64)
       - **Shape:** `VM.Standard.A1.Flex` (Ampere ARM)
       - **OCPU's:** 2 (of maximaal 4)
       - **Geheugen:** 12 GB (of maximaal 24 GB)
       - **Opstartvolume:** 50 GB (maximaal 200 GB gratis)
       - **SSH-sleutel:** voeg je openbare sleutel toe
    4. Klik op **Create** en noteer het openbare IP-adres.

    <Tip>
    Als het maken van de instance mislukt met "Out of capacity", probeer dan een ander beschikbaarheidsdomein of probeer het later opnieuw. De capaciteit van de gratis laag is beperkt.
    </Tip>

  </Step>

  <Step title="Verbinding maken en het systeem bijwerken">
    ```bash
    ssh ubuntu@YOUR_PUBLIC_IP

    sudo apt update && sudo apt upgrade -y
    sudo apt install -y build-essential
    ```

    `build-essential` is vereist om sommige afhankelijkheden voor ARM te compileren.

  </Step>

  <Step title="Gebruiker en hostnaam configureren">
    ```bash
    sudo hostnamectl set-hostname openclaw
    sudo passwd ubuntu
    sudo loginctl enable-linger ubuntu
    ```

    Door linger in te schakelen, blijven gebruikersservices na het afmelden actief.

  </Step>

  <Step title="Tailscale installeren">
    ```bash
    curl -fsSL https://tailscale.com/install.sh | sh
    sudo tailscale up --ssh --hostname=openclaw
    ```

    Maak vanaf nu verbinding via Tailscale: `ssh ubuntu@openclaw`.

  </Step>

  <Step title="OpenClaw installeren">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    source ~/.bashrc
    ```

    Wanneer "How do you want to hatch your bot?" wordt gevraagd, selecteer je **Do this later**.

  </Step>

  <Step title="De Gateway configureren">
    Gebruik tokenauthenticatie met Tailscale Serve voor veilige externe toegang.

    ```bash
    openclaw config set gateway.bind loopback
    openclaw config set gateway.auth.mode token
    openclaw doctor --generate-gateway-token
    openclaw config set gateway.tailscale.mode serve
    openclaw config set gateway.trustedProxies '["127.0.0.1"]'

    systemctl --user restart openclaw-gateway.service
    ```

    `gateway.trustedProxies=["127.0.0.1"]` dient hier alleen voor de verwerking van doorgestuurde IP-adressen/lokale clients door de lokale Tailscale Serve-proxy. Het is **niet** `gateway.auth.mode: "trusted-proxy"`. Routes van de diffviewer behouden in deze configuratie het fail-closed-gedrag: onbewerkte `127.0.0.1`-vieweraanvragen zonder doorgestuurde proxyheaders retourneren `Diff not found`. Gebruik `mode=file` / `mode=both` voor bijlagen, of schakel externe viewers bewust in en stel `plugins.entries.diffs.config.viewerBaseUrl` in (of geef een proxy-`baseUrl` door) als je deelbare viewerlinks nodig hebt.

  </Step>

  <Step title="VCN-beveiliging aanscherpen">
    Blokkeer al het verkeer behalve Tailscale aan de netwerkrand:

    1. Ga in de OCI Console naar **Networking > Virtual Cloud Networks**.
    2. Klik op je VCN en vervolgens op **Security Lists > Default Security List**.
    3. **Verwijder** alle regels voor inkomend verkeer behalve `0.0.0.0/0 UDP 41641` (Tailscale).
    4. Behoud de standaardregels voor uitgaand verkeer (al het uitgaande verkeer toestaan).

    Hiermee worden SSH op poort 22, HTTP, HTTPS en al het overige verkeer aan de netwerkrand geblokkeerd. Vanaf dit punt kun je alleen via Tailscale verbinding maken.

  </Step>

  <Step title="Verifiëren">
    ```bash
    openclaw --version
    systemctl --user status openclaw-gateway.service
    tailscale serve status
    curl http://localhost:18789
    ```

    Open de Control UI vanaf elk apparaat in je tailnet:

    ```
    https://openclaw.<tailnet-name>.ts.net/
    ```

    Vervang `<tailnet-name>` door de naam van je tailnet (zichtbaar in `tailscale status`).

  </Step>
</Steps>

## De beveiligingsstatus verifiëren

Met het VCN afgeschermd (alleen UDP 41641 geopend) en de Gateway aan loopback gebonden, wordt openbaar verkeer aan de netwerkrand geblokkeerd en is beheerderstoegang alleen via het tailnet mogelijk. Hierdoor zijn verschillende traditionele stappen voor VPS-beveiliging overbodig:

| Traditionele stap             | Nodig?        | Waarom                                                                            |
| ----------------------------- | ------------- | --------------------------------------------------------------------------------- |
| UFW-firewall                  | Nee           | Het VCN blokkeert verkeer voordat het de instance bereikt.                        |
| fail2ban                      | Nee           | Poort 22 is in het VCN geblokkeerd; er is geen oppervlak voor brute-forceaanvallen. |
| sshd-beveiliging aanscherpen  | Nee           | Tailscale SSH gebruikt sshd niet.                                                 |
| Rootaanmelding uitschakelen   | Nee           | Tailscale verifieert aan de hand van de tailnet-identiteit, niet van systeemgebruikers. |
| Alleen SSH-sleutelauthenticatie | Nee         | Hetzelfde -- de tailnet-identiteit vervangt de SSH-sleutels van het systeem.      |
| IPv6-beveiliging aanscherpen  | Meestal niet  | Afhankelijk van de VCN-/subnetinstellingen; verifieer wat daadwerkelijk is toegewezen/blootgesteld. |

Nog steeds aanbevolen:

- `chmod 700 ~/.openclaw` om de machtigingen voor referentiebestanden te beperken.
- `openclaw security audit` voor een OpenClaw-specifieke controle van de beveiligingsstatus.
- Regelmatig `sudo apt update && sudo apt upgrade` voor patches van het besturingssysteem.
- Controleer regelmatig de apparaten in de [Tailscale-beheerconsole](https://login.tailscale.com/admin).

Snelle verificatieopdrachten:

```bash
# Bevestig dat er geen openbare poorten luisteren
sudo ss -tlnp | grep -v '127.0.0.1\|::1'

# Verifieer dat Tailscale SSH actief is
tailscale status | grep -q 'offers: ssh' && echo "Tailscale SSH actief"

# Optioneel: schakel sshd volledig uit nadat is bevestigd dat Tailscale SSH werkt
sudo systemctl disable --now ssh
```

## ARM-opmerkingen

De Always Free-laag gebruikt ARM (`aarch64`). De meeste functies van OpenClaw werken probleemloos; voor een klein aantal systeemeigen binaire bestanden zijn ARM-builds nodig:

- Node.js, Telegram, WhatsApp (Baileys): pure JavaScript, geen problemen.
- De meeste npm-pakketten met systeemeigen code: vooraf gebouwde `linux-arm64`-artefacten beschikbaar.
- Optionele CLI-hulpprogramma's (bijvoorbeeld Go-/Rust-binaire bestanden die door Skills worden geleverd): controleer vóór de installatie of er een `aarch64`- / `linux-arm64`-release beschikbaar is.

Verifieer de architectuur met `uname -m` (moet `aarch64` afdrukken). Installeer binaire bestanden zonder ARM-build vanuit de broncode of sla ze over.

## Persistentie en back-ups

De status van OpenClaw bevindt zich onder:

- `~/.openclaw/` -- `openclaw.json`, `auth-profiles.json` per agent, kanaal-/providerstatus en sessiegegevens.
- `~/.openclaw/workspace/` -- de werkruimte van de agent (SOUL.md, geheugen, artefacten).

Deze blijven behouden na opnieuw opstarten. Maak als volgt een overdraagbare momentopname:

```bash
openclaw backup create
```

## Terugvaloptie: SSH-tunnel

Als Tailscale Serve niet werkt, gebruik je vanaf je lokale computer een SSH-tunnel:

```bash
ssh -L 18789:127.0.0.1:18789 ubuntu@openclaw
```

Open vervolgens `http://localhost:18789`.

## Problemen oplossen

**Maken van instance mislukt ("Out of capacity")** -- ARM-instances in de gratis laag zijn populair. Probeer een ander beschikbaarheidsdomein of probeer het opnieuw tijdens daluren.

**Tailscale maakt geen verbinding** -- Voer `sudo tailscale up --ssh --hostname=openclaw --reset` uit om je opnieuw te verifiëren.

**Gateway start niet** -- Voer `openclaw doctor --non-interactive` uit en controleer de logboeken met `journalctl --user -u openclaw-gateway.service -n 50`.

**Problemen met ARM-binaire bestanden** -- De meeste npm-pakketten werken op ARM64. Zoek voor systeemeigen binaire bestanden naar `linux-arm64`- of `aarch64`-releases. Verifieer de architectuur met `uname -m`.

## Volgende stappen

- [Kanalen](/nl/channels) -- verbind Telegram, WhatsApp, Discord en meer
- [Gateway-configuratie](/nl/gateway/configuration) -- alle configuratieopties
- [Bijwerken](/nl/install/updating) -- houd OpenClaw up-to-date

## Gerelateerd

- [Installatieoverzicht](/nl/install)
- [GCP](/nl/install/gcp)
- [VPS-hosting](/nl/vps)
