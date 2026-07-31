---
read_when:
    - Je wilt een gecontaineriseerde Gateway met Podman in plaats van Docker
summary: Voer OpenClaw uit in een rootless Podman-container
title: Podman
x-i18n:
    generated_at: "2026-07-27T05:37:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2db1f2b0413d7b9e1b2007aaae2da9d07fa44a1b52901d4a6cbc6274e54567f1
    source_path: install/podman.md
    workflow: 16
---

Voer de OpenClaw Gateway uit in een rootless Podman-container, beheerd door je huidige niet-rootgebruiker.

Het model:

- Podman voert de Gateway-container uit.
- De `openclaw`-CLI op je host is het besturingsvlak.
- Persistente status wordt standaard op de host opgeslagen onder `~/.openclaw`.
- Voor dagelijks beheer gebruik je `openclaw --container <name> ...` in plaats van `sudo -u openclaw`, `podman exec` of een afzonderlijke servicegebruiker.

## Vereisten

- **Podman** in rootless-modus
- **OpenClaw-CLI** geïnstalleerd op de host
- **Optioneel:** `systemd --user` als je automatisch starten via Quadlet wilt beheren
- **Optioneel:** `sudo` alleen als je `loginctl enable-linger "$(whoami)"` wilt voor persistentie tijdens het opstarten op een headless host

## Snel aan de slag

<Steps>
  <Step title="Eenmalige configuratie">
    Voer vanuit de hoofdmap van de repository `./scripts/podman/setup.sh` uit.

    Hiermee wordt `openclaw:local` gebouwd in je rootless Podman-opslag (of `OPENCLAW_IMAGE` / `OPENCLAW_PODMAN_IMAGE` opgehaald indien ingesteld), wordt `~/.openclaw/openclaw.json` met `gateway.mode: "local"` aangemaakt als deze ontbreekt en wordt `~/.openclaw/.env` met een gegenereerde `OPENCLAW_GATEWAY_TOKEN` aangemaakt als deze ontbreekt.

    Optionele omgevingsvariabelen voor de build:

    | Variabele | Effect |
    | --- | --- |
    | `OPENCLAW_IMAGE` / `OPENCLAW_PODMAN_IMAGE` | Gebruik een bestaande/opgehaalde image in plaats van `openclaw:local` te bouwen |
    | `OPENCLAW_IMAGE_APT_PACKAGES` | Installeer extra apt-pakketten tijdens het bouwen van de image (accepteert ook de verouderde `OPENCLAW_DOCKER_APT_PACKAGES`) |
    | `OPENCLAW_IMAGE_PIP_PACKAGES` | Installeer extra Python-pakketten tijdens het bouwen van de image; zet versies vast en gebruik alleen pakketindexen die je vertrouwt |
    | `OPENCLAW_EXTENSIONS` | Compileer/verpak ondersteunde geselecteerde plugins en installeer hun runtime-afhankelijkheden |
    | `OPENCLAW_INSTALL_BROWSER` | Installeer Chromium en Xvfb vooraf voor browserautomatisering (stel in op `1`) |

    Voor een door Quadlet beheerde configuratie (alleen Linux + systemd-gebruikersservices):

    ```bash
    ./scripts/podman/setup.sh --quadlet
    ```

    Of stel `OPENCLAW_PODMAN_QUADLET=1` in.

  </Step>

  <Step title="De Gateway-container starten">
    ```bash
    ./scripts/run-openclaw-podman.sh launch
    ```

    Start de container met je huidige uid/gid en `--userns=keep-id` en koppelt je OpenClaw-status via een bind-mount aan de container.

  </Step>

  <Step title="Onboarding uitvoeren in de container">
    ```bash
    ./scripts/run-openclaw-podman.sh launch setup
    ```

    Open vervolgens `http://127.0.0.1:18789/` en gebruik het token uit `~/.openclaw/.env`.

    Modelauthenticatie: gebruik tijdens de configuratie door OpenClaw beheerde authenticatie (Anthropic-API-sleutels of OpenAI Codex-browser-OAuth/apparaatcodeauthenticatie voor door Codex ondersteunde OpenAI). Het Podman-startprogramma koppelt de referentiemappen van de host-CLI, zoals `~/.claude` of `~/.codex`, niet aan de configuratie- of Gateway-container. Bestaande aanmeldingen van de host-CLI zijn alleen gemakspaden op dezelfde host -- bewaar voor containerinstallaties de providerauthenticatie in de gekoppelde `~/.openclaw`-status die door de configuratie wordt beheerd.

  </Step>

  <Step title="De actieve container beheren via de host-CLI">
    ```bash
    export OPENCLAW_CONTAINER=openclaw
    ```

    Normale `openclaw`-opdrachten worden vervolgens automatisch in die container uitgevoerd:

    ```bash
    openclaw dashboard --no-open
    openclaw gateway status --deep   # omvat extra servicescan
    openclaw doctor
    openclaw channels login
    ```

    Op macOS kan Podman machine ervoor zorgen dat de browser voor de Gateway niet-lokaal lijkt. Als de Control UI na het starten fouten voor apparaatauthenticatie meldt, gebruik dan de Tailscale-richtlijnen in [Podman en Tailscale](#podman-and-tailscale).

  </Step>
</Steps>

Het handmatige startprogramma leest slechts een kleine toelatingslijst met Podman-gerelateerde sleutels uit `~/.openclaw/.env` en geeft expliciete runtime-omgevingsvariabelen door aan de container; het geeft niet het volledige omgevingsbestand door aan Podman.

<a id="podman-and-tailscale"></a>

## Podman en Tailscale

Volg voor HTTPS of externe browsertoegang de algemene Tailscale-documentatie.

Specifieke opmerkingen voor Podman:

- Houd de Podman-publicatiehost op `127.0.0.1`.
- Geef de voorkeur aan door de host beheerde `tailscale serve` boven `openclaw gateway --tailscale serve`.
- Gebruik op macOS Tailscale-toegang in plaats van geïmproviseerde lokale tunneloplossingen als de apparaatauthenticatiecontext van de lokale browser onbetrouwbaar is.

Zie [Tailscale](/nl/gateway/tailscale) en [Control UI](/nl/web/control-ui).

## Systemd (Quadlet, optioneel)

Als je `./scripts/podman/setup.sh --quadlet` hebt uitgevoerd, installeert de configuratie een Quadlet-bestand op `~/.config/containers/systemd/openclaw.container`.

| Actie | Opdracht                                    |
| ------ | ------------------------------------------ |
| Starten  | `systemctl --user start openclaw.service`  |
| Stoppen   | `systemctl --user stop openclaw.service`   |
| Status | `systemctl --user status openclaw.service` |
| Logboeken   | `journalctl --user -u openclaw.service -f` |

Na het bewerken van het Quadlet-bestand:

```bash
systemctl --user daemon-reload
systemctl --user restart openclaw.service
```

Schakel lingering in voor je huidige gebruiker voor persistentie tijdens het opstarten op SSH-/headless hosts:

```bash
sudo loginctl enable-linger "$(whoami)"
```

De gegenereerde Quadlet-service behoudt een vaste, geharde standaardvorm: `127.0.0.1` gepubliceerde poorten (`18789` Gateway, `18790` bridge), `--bind lan` in de container, `keep-id` gebruikersnaamruimte, `OPENCLAW_NO_RESPAWN=1`, `Restart=on-failure` en `TimeoutStartSec=300`. Deze leest `~/.openclaw/.env` als een runtime-`EnvironmentFile` voor waarden zoals `OPENCLAW_GATEWAY_TOKEN`, maar gebruikt niet de Podman-specifieke toelatingslijst met overrides van het handmatige startprogramma. Gebruik voor aangepaste publicatiepoorten, een aangepaste publicatiehost of andere vlaggen voor het uitvoeren van containers het handmatige startprogramma, of bewerk `~/.config/containers/systemd/openclaw.container` rechtstreeks en laad en herstart vervolgens de service.

## Configuratie, omgeving en opslag

- **Configuratiemap:** `~/.openclaw`
- **Werkruimtemap:** `~/.openclaw/workspace`
- **Tokenbestand:** `~/.openclaw/.env`
- **Starthulpprogramma:** `./scripts/run-openclaw-podman.sh`

Het startscript en Quadlet koppelen de hoststatus via bind-mounts aan de container: `OPENCLAW_CONFIG_DIR` -> `/home/node/.openclaw`, `OPENCLAW_WORKSPACE_DIR` -> `/home/node/.openclaw/workspace`. Dit zijn standaard hostmappen, geen anonieme containerstatus, zodat `openclaw.json`, `auth-profiles.json` per agent, kanaal-/providerstatus, sessies en de werkruimte behouden blijven wanneer de container wordt vervangen. De configuratie vult ook `gateway.controlUi.allowedOrigins` vooraf in voor `127.0.0.1` en `localhost` op de gepubliceerde Gateway-poort, zodat het lokale dashboard werkt met de niet-loopbackbinding van de container.

Nuttige omgevingsvariabelen voor het handmatige startprogramma (bewaar deze in `~/.openclaw/.env`; het startprogramma leest dit bestand voordat het de standaardwaarden voor de container/image definitief maakt):

| Variabele                                        | Standaard          | Effect                                 |
| ------------------------------------------ | ---------------- | -------------------------------------- |
| `OPENCLAW_PODMAN_CONTAINER`                | `openclaw`       | Containernaam                         |
| `OPENCLAW_PODMAN_IMAGE` / `OPENCLAW_IMAGE` | `openclaw:local` | Uit te voeren image                           |
| `OPENCLAW_PODMAN_GATEWAY_HOST_PORT`        | `18789`          | Hostpoort gekoppeld aan container `18789`  |
| `OPENCLAW_PODMAN_BRIDGE_HOST_PORT`         | `18790`          | Hostpoort gekoppeld aan container `18790`  |
| `OPENCLAW_PODMAN_PUBLISH_HOST`             | `127.0.0.1`      | Hostinterface voor gepubliceerde poorten     |
| `OPENCLAW_GATEWAY_BIND`                    | `lan`            | Gateway-bindingsmodus in de container |
| `OPENCLAW_PODMAN_USERNS`                   | `keep-id`        | `keep-id`, `auto` of `host`           |

Als je een niet-standaard `OPENCLAW_CONFIG_DIR` of `OPENCLAW_WORKSPACE_DIR` gebruikt, stel dan dezelfde variabelen in voor zowel `./scripts/podman/setup.sh` als latere `./scripts/run-openclaw-podman.sh launch`-opdrachten -- het repositorylokale startprogramma bewaart aangepaste padoverschrijvingen niet tussen shells.

## Images upgraden

Start de container of Quadlet-service opnieuw nadat je een nieuwe image hebt gebouwd of opgehaald.
Bij de eerste start van een nieuwe OpenClaw-versie voert de Gateway veilige reparaties aan de status en
plugins uit voordat deze meldt dat hij gereed is.

Als de Gateway afsluit in plaats van gereed te worden, voer je dezelfde image eenmaal uit met
`openclaw doctor --fix` voor dezelfde gekoppelde status/configuratie en start je de
Gateway vervolgens normaal opnieuw:

```bash
OPENCLAW_CONFIG_DIR="${OPENCLAW_CONFIG_DIR:-$HOME/.openclaw}"
OPENCLAW_WORKSPACE_DIR="${OPENCLAW_WORKSPACE_DIR:-$OPENCLAW_CONFIG_DIR/workspace}"
OPENCLAW_PODMAN_IMAGE="${OPENCLAW_PODMAN_IMAGE:-${OPENCLAW_IMAGE:-openclaw:local}}"

podman run --rm -it \
  --userns=keep-id \
  --user "$(id -u):$(id -g)" \
  -e HOME=/home/node \
  -e NPM_CONFIG_CACHE=/home/node/.openclaw/.npm \
  -v "$OPENCLAW_CONFIG_DIR:/home/node/.openclaw:rw" \
  -v "$OPENCLAW_WORKSPACE_DIR:/home/node/.openclaw/workspace:rw" \
  "$OPENCLAW_PODMAN_IMAGE" \
  openclaw doctor --fix
```

Voeg op SELinux-hosts `,Z` toe aan beide bind-mounts als Podman de toegang tot de
gekoppelde status blokkeert.

## Nuttige opdrachten

- **Containerlogboeken:** `podman logs -f openclaw`
- **Container stoppen:** `podman stop openclaw`
- **Container verwijderen:** `podman rm -f openclaw`
- **Dashboard-URL openen vanuit de host-CLI:** `openclaw dashboard --no-open`
- **Gezondheid/status via de host-CLI:** `openclaw gateway status --deep` (RPC-probe + extra servicescan)

## Problemen oplossen

- **Toegang geweigerd (EACCES) voor configuratie of werkruimte:** De container wordt standaard uitgevoerd met `--userns=keep-id` en `--user <your uid>:<your gid>`. Zorg dat de configuratie-/werkruimtepaden op de host eigendom zijn van je huidige gebruiker.
- **Starten van Gateway geblokkeerd (`gateway.mode=local` ontbreekt):** Zorg dat `~/.openclaw/openclaw.json` bestaat en `gateway.mode="local"` instelt. `scripts/podman/setup.sh` maakt dit aan als het ontbreekt.
- **Container wordt opnieuw gestart na een image-update:** Voer de eenmalige `openclaw doctor --fix`-opdracht uit in [Images upgraden](#upgrading-images) en start de Gateway vervolgens opnieuw.
- **CLI-opdrachten voor de container bereiken het verkeerde doel:** Gebruik `openclaw --container <name> ...` expliciet of exporteer `OPENCLAW_CONTAINER=<name>` in je shell.
- **`openclaw update` mislukt met `--container`:** Dit is te verwachten. Bouw de image opnieuw of haal deze op en start vervolgens de container of Quadlet-service opnieuw.
- **Quadlet-service start niet:** Voer `systemctl --user daemon-reload` uit en daarna `systemctl --user start openclaw.service`. Op headless systemen heb je mogelijk ook `sudo loginctl enable-linger "$(whoami)"` nodig.
- **SELinux blokkeert bind-mounts:** Laat het standaard mountgedrag ongewijzigd; het startprogramma voegt op Linux automatisch `:Z` toe wanneer SELinux in enforcing- of permissive-modus staat.

## Gerelateerd

- [Docker](/nl/install/docker)
- [Gateway-achtergrondproces](/nl/gateway/background-process)
- [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting)
