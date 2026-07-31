---
read_when:
    - Je wilt de Gateway uitvoeren op een Linux-server of cloud-VPS
    - Je hebt een snel overzicht van hostinghandleidingen nodig
    - Je wilt algemene Linux-servertuning voor OpenClaw
sidebarTitle: Linux Server
summary: OpenClaw uitvoeren op een Linux-server of cloud-VPS — providerkeuze, architectuur en optimalisatie
title: Linux-server
x-i18n:
    generated_at: "2026-07-27T05:56:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 634a246850ab8b854c2c799688fd368ebed3a02124baa85bf38d5ff6ef8cec64
    source_path: vps.md
    workflow: 16
---

Voer de OpenClaw Gateway uit op elke Linux-server of cloud-VPS. Deze pagina helpt je
een provider te kiezen, legt uit hoe cloudimplementaties werken en behandelt algemene Linux-
optimalisaties die overal van toepassing zijn.

## Kies een provider

<CardGroup cols={2}>
  <Card title="Azure" href="/nl/install/azure">Linux-VM</Card>
  <Card title="DigitalOcean" href="/nl/install/digitalocean">Eenvoudige betaalde VPS</Card>
  <Card title="exe.dev" href="/nl/install/exe-dev">VM met HTTPS-proxy</Card>
  <Card title="Fly.io" href="/nl/install/fly">Fly Machines</Card>
  <Card title="GCP" href="/nl/install/gcp">Compute Engine</Card>
  <Card title="Hetzner" href="/nl/install/hetzner">Docker op een Hetzner-VPS</Card>
  <Card title="Hostinger" href="/nl/install/hostinger">VPS met installatie met één klik</Card>
  <Card title="Northflank" href="/nl/install/northflank">Installatie met één klik via de browser</Card>
  <Card title="Oracle Cloud" href="/nl/install/oracle">Always Free ARM-niveau</Card>
  <Card title="Railway" href="/nl/install/railway">Installatie met één klik via de browser</Card>
  <Card title="Raspberry Pi" href="/nl/install/raspberry-pi">Zelfgehost op ARM</Card>
</CardGroup>

**AWS (EC2 / Lightsail / gratis niveau)** werkt ook goed.
Een stapsgewijze communityvideo is beschikbaar op
[x.com/techfrenAJ/status/2014934471095812547](https://x.com/techfrenAJ/status/2014934471095812547)
(communitybron -- kan onbeschikbaar worden).

## Hoe cloudconfiguraties werken

- De **Gateway draait op de VPS** en beheert de status en werkruimte.
- Je maakt vanaf je laptop of telefoon verbinding via de **bedieningsinterface** of **Tailscale/SSH**.
- Beschouw de VPS als de bron van waarheid en maak regelmatig een **back-up** van de status en werkruimte.
- Veilige standaardinstelling: houd de Gateway op loopback en open deze via een SSH-tunnel of Tailscale Serve.
  Als je deze aan `lan` of `tailnet` bindt, vereist de Gateway een gedeeld geheim
  (`gateway.auth.token` of `gateway.auth.password`), tenzij authenticatie wordt gedelegeerd aan een
  vertrouwde proxy.

Gerelateerde pagina's: [Externe toegang tot de Gateway](/nl/gateway/remote), [Platformoverzicht](/nl/platforms).

## Beveilig eerst de beheerderstoegang

Bepaal voordat je OpenClaw op een openbare VPS installeert hoe je
de server zelf wilt beheren.

- Voor beheerderstoegang uitsluitend via Tailnet: installeer eerst Tailscale, voeg de VPS toe aan je
  tailnet, verifieer een tweede SSH-sessie via het Tailscale-IP-adres of de MagicDNS-naam
  en beperk daarna openbare SSH-toegang.
- Zonder Tailscale: pas de overeenkomstige beveiligingsmaatregelen toe op je SSH-toegang voordat
  je meer services beschikbaar stelt.
- Dit staat los van toegang tot de Gateway. Je kunt OpenClaw nog steeds aan
  loopback gebonden houden en een SSH-tunnel of Tailscale Serve voor het dashboard gebruiken.

Gateway-opties die specifiek zijn voor Tailscale staan in [Tailscale](/nl/gateway/tailscale).

## Gedeelde bedrijfsagent op een VPS

Eén agent voor een team uitvoeren is een geldige configuratie wanneer elke gebruiker zich binnen
dezelfde vertrouwensgrens bevindt en de agent uitsluitend zakelijk wordt gebruikt.

- Houd deze in een speciale runtime (VPS/VM/container + afzonderlijke OS-gebruiker/accounts).
- Meld die runtime niet aan bij persoonlijke Apple-/Google-accounts of persoonlijke browser-/wachtwoordbeheerderprofielen.
- Als gebruikers elkaar als tegenstanders kunnen behandelen, splits je ze op per Gateway/host/OS-gebruiker.

Details over het beveiligingsmodel: [Beveiliging](/nl/gateway/security).

## Nodes gebruiken met een VPS

Je kunt de Gateway in de cloud houden en **nodes** koppelen op je lokale apparaten
(Mac/iOS/Android/headless). Nodes bieden lokale scherm-, camera- en canvasmogelijkheden en `system.run`-
mogelijkheden, terwijl de Gateway in de cloud blijft.

Documentatie: [Nodes](/nl/nodes), [Nodes-CLI](/nl/cli/nodes).

## Opstartoptimalisatie voor kleine VM's en ARM-hosts

Als CLI-opdrachten traag aanvoelen op VM's met weinig rekenkracht (of ARM-hosts), schakel dan de modulecompilatiecache van Node in:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

- `NODE_COMPILE_CACHE` verkort de opstarttijd van herhaalde opdrachten; bij de eerste uitvoering wordt de cache opgewarmd.
- `OPENCLAW_NO_RESPAWN=1` houdt routinematige herstarts van de Gateway binnen hetzelfde proces, waardoor extra procesoverdrachten worden vermeden en PID-tracering eenvoudig blijft op kleine hosts.
- Zie [Raspberry Pi](/nl/install/raspberry-pi) voor specifieke informatie over Raspberry Pi.

### Checklist voor systemd-optimalisatie (optioneel)

Overweeg voor VM-hosts die `systemd` gebruiken:

- Service-omgevingsvariabelen voor een stabiel opstartpad: `OPENCLAW_NO_RESPAWN=1` en
  `NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache`
- Expliciet herstartgedrag: `Restart=always`, `RestartSec=2`, `TimeoutStartSec=90`
- Schijven op SSD-basis voor status-/cachepaden om nadelen bij koude starts door willekeurige I/O te beperken.

Het standaardpad `openclaw onboard --install-daemon` installeert een systemd-gebruikers-
unit; bewerk deze met:

```bash
systemctl --user edit openclaw-gateway.service
```

```ini
[Service]
Environment=OPENCLAW_NO_RESPAWN=1
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Restart=always
RestartSec=2
TimeoutStartSec=90
```

Als je bewust een systeemunit hebt geïnstalleerd, bewerk je deze via
`sudo systemctl edit openclaw-gateway.service`.

Hoe `Restart=`-beleid geautomatiseerd herstel ondersteunt:
[systemd kan serviceherstel automatiseren](https://www.redhat.com/en/blog/systemd-automate-recovery).

Zie [Linux-geheugendruk en OOM-beëindigingen](/nl/platforms/linux#memory-pressure-and-oom-kills) voor het Linux OOM-gedrag, de selectie van onderliggende processen als slachtoffer en
`exit 137`-diagnostiek.

## Gerelateerd

- [Installatieoverzicht](/nl/install)
- [DigitalOcean](/nl/install/digitalocean)
- [Fly.io](/nl/install/fly)
- [Hetzner](/nl/install/hetzner)
