---
read_when:
    - Je wilt een goedkope Linux-host die altijd actief is voor de Gateway
    - Je wilt externe toegang tot de Control UI zonder je eigen VPS te beheren
summary: Voer OpenClaw Gateway uit op exe.dev (VM + HTTPS-proxy) voor externe toegang
title: exe.dev
x-i18n:
    generated_at: "2026-07-27T05:08:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a768511d2d7e4e4ec10bcdae83684417bde05286468b0534200f8dd5ec015f7b
    source_path: install/exe-dev.md
    workflow: 16
---

**Doel:** OpenClaw Gateway draaiend op een [exe.dev](https://exe.dev)-VM, bereikbaar via `https://<vm-name>.exe.xyz`.

Deze handleiding gaat uit van de standaard **exeuntu**-image van exe.dev. Pas de pakketten dienovereenkomstig aan voor andere distributies.

## Wat je nodig hebt

- exe.dev-account
- `ssh exe.dev`-toegang tot exe.dev-VM's (optioneel, voor handmatige configuratie)

## Snel aan de slag voor beginners

1. Open [https://exe.new/openclaw](https://exe.new/openclaw)
2. Vul naar behoefte je authenticatiesleutel/-token in
3. Klik naast je VM op "Agent" en wacht tot Shelley klaar is met de inrichting
4. Open `https://<vm-name>.exe.xyz/` en authenticeer met het geconfigureerde gedeelde geheim (standaard tokenauthenticatie; wachtwoordauthenticatie werkt ook als je `gateway.auth.mode` wijzigt)
5. Keur openstaande aanvragen voor apparaatkoppeling goed met `openclaw devices approve <requestId>`

## Geautomatiseerde installatie met Shelley

Shelley, de agent van exe.dev, kan OpenClaw installeren aan de hand van een prompt:

```text
Configureer OpenClaw (https://docs.openclaw.ai/install) op deze VM. Gebruik de vlaggen voor niet-interactieve uitvoering en risicoacceptatie voor de onboarding van OpenClaw. Voeg waar nodig de verstrekte authenticatie of het token toe. Configureer nginx om verkeer van de standaardpoort 18789 door te sturen naar de hoofdlocatie in de standaard ingeschakelde siteconfiguratie en zorg dat WebSocket-ondersteuning is ingeschakeld. Koppelen gebeurt met "openclaw devices list" en "openclaw devices approve <request id>". Zorg dat het dashboard aangeeft dat de status van OpenClaw OK is. exe.dev verzorgt voor ons het doorsturen van poort 8000 naar poort 80/443 en HTTPS, dus de uiteindelijke bereikbare locatie moet <vm-name>.exe.xyz zijn, zonder poortvermelding.
```

## Handmatige installatie

<Steps>
  <Step title="De VM maken">
    Vanaf je apparaat:

    ```bash
    ssh exe.dev new
    ```

    Maak vervolgens verbinding:

    ```bash
    ssh <vm-name>.exe.xyz
    ```

    <Tip>
    Houd deze VM **stateful**. OpenClaw slaat `openclaw.json`, `auth-profiles.json` per agent, sessies en kanaal-/providerstatus op onder `~/.openclaw/`, plus de werkruimte onder `~/.openclaw/workspace/`.
    </Tip>

  </Step>

  <Step title="Vereisten installeren (op de VM)">
    ```bash
    sudo apt-get update
    sudo apt-get install -y git curl jq ca-certificates openssl
    ```
  </Step>

  <Step title="OpenClaw installeren">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```
  </Step>

  <Step title="nginx configureren als proxy naar poort 8000">
    Bewerk `/etc/nginx/sites-enabled/default`:

    ```nginx
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        listen 8000;
        listen [::]:8000;

        server_name _;

        location / {
            proxy_pass http://127.0.0.1:18789;
            proxy_http_version 1.1;

            # WebSocket-ondersteuning
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            # Standaard proxyheaders
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Time-outinstellingen voor langdurige verbindingen
            proxy_read_timeout 86400s;
            proxy_send_timeout 86400s;
        }
    }
    ```

    Overschrijf doorstuurheaders in plaats van door de client aangeleverde ketens te behouden. OpenClaw vertrouwt doorgestuurde IP-metagegevens alleen van expliciet geconfigureerde proxy's, en `X-Forwarded-For`-ketens die waarden toevoegen, worden beschouwd als een beveiligingsrisico.

  </Step>

  <Step title="OpenClaw openen en apparaten goedkeuren">
    Open `https://<vm-name>.exe.xyz/` (zie de uitvoer van de Control UI tijdens de onboarding). Als om authenticatie wordt gevraagd, plak je het geconfigureerde gedeelde geheim van de VM.

    Deze handleiding gebruikt standaard tokenauthenticatie. Haal daarom `gateway.auth.token` op met `openclaw config get gateway.auth.token`, of genereer een nieuw token met `openclaw doctor --n`. Als je de Gateway hebt overgeschakeld naar wachtwoordauthenticatie, gebruik je in plaats daarvan `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`.

    Keur apparaten goed met `openclaw devices list` en `openclaw devices approve <requestId>`. Gebruik bij twijfel Shelley vanuit je browser.

  </Step>
</Steps>

## Kanalen op afstand configureren

Gebruik voor externe hosts bij voorkeur één aanroep van `config patch` in plaats van veel SSH-aanroepen naar `config set`. Bewaar echte tokens in de VM-omgeving of `~/.openclaw/.env`, en plaats alleen SecretRefs in `openclaw.json`. Zie [Geheimen beheren](/nl/gateway/secrets) voor het volledige SecretRef-contract.

Zorg er op de VM voor dat de serviceomgeving de benodigde geheimen bevat:

```bash
cat >> ~/.openclaw/.env <<'EOF'
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...
DISCORD_BOT_TOKEN=...
OPENAI_API_KEY=sk-...
EOF
```

Maak op je lokale machine een patchbestand en stuur het via een pipe naar de VM:

```json5
// openclaw.remote.patch.json5
{
  secrets: {
    providers: {
      default: { source: "env" },
    },
  },
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      groupPolicy: "open",
      requireMention: false,
    },
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
      dmPolicy: "disabled",
      dm: { enabled: false },
      groupPolicy: "allowlist",
    },
  },
  agents: {
    defaults: {
      model: { primary: "openai/gpt-5.6-sol" },
      models: {
        "openai/gpt-5.6-sol": { params: { fastMode: true } },
      },
    },
  },
}
```

```bash
ssh <vm-name>.exe.xyz 'openclaw config patch --stdin --dry-run' < ./openclaw.remote.patch.json5
ssh <vm-name>.exe.xyz 'openclaw config patch --stdin' < ./openclaw.remote.patch.json5
ssh <vm-name>.exe.xyz 'openclaw gateway restart && openclaw health'
```

Gebruik `--replace-path` wanneer een geneste toelatingslijst exact gelijk moet worden aan de patchwaarde, bijvoorbeeld bij het vervangen van de toelatingslijst van een Discord-kanaal:

```bash
ssh <vm-name>.exe.xyz 'openclaw config patch --stdin --replace-path "channels.discord.guilds[\"123\"].channels"' < ./discord.patch.json5
```

Zie [Discord](/nl/channels/discord) en [Slack](/nl/channels/slack) voor de volledige naslaginformatie over kanaalconfiguratie.

## Externe toegang

exe.dev verzorgt de authenticatie voor externe toegang. Standaard wordt HTTP-verkeer vanaf poort 8000 doorgestuurd naar `https://<vm-name>.exe.xyz` met e-mailauthenticatie.

## Bijwerken

```bash
openclaw update
```

Zie [Bijwerken](/nl/install/updating) voor het wisselen van kanalen en handmatig herstel.

## Gerelateerd

- [Externe Gateway](/nl/gateway/remote)
- [Installatieoverzicht](/nl/install)
