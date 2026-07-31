---
read_when:
    - OpenClaw implementeren op Fly.io
    - Fly-volumes, secrets en configuratie voor de eerste uitvoering instellen
summary: Stapsgewijze Fly.io-implementatie voor OpenClaw met permanente opslag en HTTPS
title: Fly.io
x-i18n:
    generated_at: "2026-07-27T05:17:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d2b5119c1df8ee077f4db4f44fa92c6ae0e2bf3c355c2117e0fd39146bb49875
    source_path: install/fly.md
    workflow: 16
---

**Doel:** OpenClaw Gateway die op een [Fly.io](https://fly.io)-machine draait met permanente opslag, automatische HTTPS en toegang tot Discord/kanalen.

## Wat je nodig hebt

- [flyctl CLI](https://fly.io/docs/hands-on/install-flyctl/) geïnstalleerd
- Fly.io-account (gratis niveau werkt)
- Modelauthenticatie: API-sleutel voor de gekozen modelprovider
- Kanaalreferenties: Discord-bottoken, Telegram-token, enzovoort

## Snel aan de slag voor beginners

1. Kloon de repository en pas `fly.toml` aan
2. Maak de app en het volume aan en stel geheimen in
3. Implementeer met `fly deploy`
4. Maak via SSH verbinding om de configuratie te maken, of gebruik de Control UI

<Steps>
  <Step title="De Fly-app maken">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw

    # kies je eigen naam
    fly apps create my-openclaw

    # 1 GB is doorgaans voldoende
    fly volumes create openclaw_data --size 1 --region iad
    ```

    Kies een regio bij jou in de buurt. Veelgebruikte opties: `lhr` (Londen), `iad` (Virginia), `sjc` (San Jose).

  </Step>

  <Step title="fly.toml configureren">
    Bewerk `fly.toml` zodat deze overeenkomt met de naam en vereisten van je app. De door de repository bijgehouden `fly.toml` is de openbare sjabloon die hieronder wordt weergegeven; `deploy/fly.private.toml` is de geharde variant zonder openbaar IP-adres (zie [Privé-implementatie](#private-deployment-hardened)).

    ```toml
    app = "my-openclaw"  # naam van je app
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

    Het ingangspunt van de OpenClaw Docker-image is `tini`, dat standaard `node openclaw.mjs gateway` uitvoert. Fly `[processes]` vervangt Docker `CMD` (hier wordt `node dist/index.js gateway ...` rechtstreeks uitgevoerd, hetzelfde gecompileerde ingangspunt) zonder `ENTRYPOINT` te wijzigen, zodat het proces nog steeds onder `tini` draait.

    **Belangrijkste instellingen:**

    | Instelling                     | Waarom                                                                      |
    | ------------------------------ | --------------------------------------------------------------------------- |
    | `--bind lan`                   | Bindt aan `0.0.0.0`, zodat de proxy van Fly de Gateway kan bereiken       |
    | `--allow-unconfigured`         | Start zonder configuratiebestand (dat maak je later)                        |
    | `internal_port = 3000`         | Moet overeenkomen met `--port 3000` (of `OPENCLAW_GATEWAY_PORT`) voor de statuscontroles van Fly |
    | `memory = "2048mb"`            | 512 MB is te weinig; 2 GB wordt aanbevolen                                  |
    | `OPENCLAW_STATE_DIR = "/data"` | Slaat de status permanent op het volume op                                   |

  </Step>

  <Step title="Geheimen instellen">
    ```bash
    # vereist: gatewayauthenticatietoken voor binding buiten de loopbackinterface
    fly secrets set OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)

    # API-sleutels voor modelproviders
    fly secrets set ANTHROPIC_API_KEY=example-anthropic-key-not-real

    # optioneel: andere providers
    fly secrets set OPENAI_API_KEY=example-openai-key-not-real
    fly secrets set GOOGLE_API_KEY=...

    # kanaaltokens
    fly secrets set DISCORD_BOT_TOKEN=example-discord-bot-token
    ```

    Bindingen buiten de loopbackinterface (`--bind lan`) vereisen een geldig gatewayauthenticatiepad. Dit voorbeeld gebruikt `OPENCLAW_GATEWAY_TOKEN`, maar `gateway.auth.password` of een correct geconfigureerde implementatie buiten de loopbackinterface met een vertrouwde proxy voldoet ook aan de vereiste. Zie [Geheimenbeheer](/nl/gateway/secrets) voor het SecretRef-contract.

    Behandel deze tokens als wachtwoorden. Geef voor API-sleutels en tokens de voorkeur aan omgevingsvariabelen/`fly secrets` boven het configuratiebestand, zodat geheimen buiten `openclaw.json` blijven.

  </Step>

  <Step title="Implementeren">
    ```bash
    fly deploy
    ```

    Bij de eerste implementatie wordt de Docker-image gebouwd. Controleer na de implementatie:

    ```bash
    fly status
    fly logs
    ```

    De opstartlogboeken van de Gateway tonen `gateway ready` zodra de HTTP-/WebSocket-listener actief is. De eigen statuscontrole van Fly bewaakt `internal_port = 3000` volgens `fly.toml`; de Docker-instructie `HEALTHCHECK` van de image bevraagt daarnaast `/healthz` op de standaardpoort 18789. Die wordt hier niet gebruikt, omdat deze implementatie de Gateway overschrijft met `--port 3000`.

  </Step>

  <Step title="Configuratiebestand maken">
    Maak via SSH verbinding met de machine om een correct configuratiebestand te maken:

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

    Met `OPENCLAW_STATE_DIR=/data` is het configuratiepad `/data/openclaw.json`.

    Vervang `https://my-openclaw.fly.dev` door de werkelijke oorsprong van je Fly-app. Bij het opstarten vult de Gateway lokale oorsprongen voor de Control UI vooraf in op basis van de runtimewaarden `--bind` en `--port`, zodat de eerste start kan doorgaan voordat de configuratie bestaat. Voor browsertoegang via Fly moet de exacte HTTPS-oorsprong echter nog steeds in `gateway.controlUi.allowedOrigins` staan.

    Het Discord-token kan afkomstig zijn uit:

    - Omgevingsvariabele `DISCORD_BOT_TOKEN` (aanbevolen voor geheimen); je hoeft deze niet aan de configuratie toe te voegen, de Gateway leest deze automatisch
    - Configuratiebestand `channels.discord.token`

    Start opnieuw om de wijzigingen toe te passen:

    ```bash
    exit
    fly machine restart <machine-id>
    ```

  </Step>

  <Step title="Toegang tot de Gateway">
    ### Control UI

    ```bash
    fly open
    ```

    Of ga naar `https://my-openclaw.fly.dev/`.

    Authenticeer met het geconfigureerde gedeelde geheim: het gatewaytoken uit `OPENCLAW_GATEWAY_TOKEN`, of je wachtwoord als je bent overgeschakeld op wachtwoordauthenticatie.

    ### Logboeken

    ```bash
    fly logs              # live logboeken
    fly logs --no-tail    # recente logboeken
    ```

    ### SSH-console

    ```bash
    fly ssh console
    ```

  </Step>
</Steps>

## Problemen oplossen

### "App is not listening on expected address"

De Gateway bindt aan `127.0.0.1` in plaats van `0.0.0.0`.

**Oplossing:** voeg `--bind lan` toe aan je procesopdracht in `fly.toml`.

### Statuscontroles mislukken / verbinding geweigerd

Fly kan de Gateway niet bereiken op de geconfigureerde poort.

**Oplossing:** zorg ervoor dat `internal_port` overeenkomt met de gatewaypoort (`--port 3000` of `OPENCLAW_GATEWAY_PORT=3000`).

### OOM-/geheugenproblemen

De container wordt steeds opnieuw gestart of beëindigd. Signalen: `SIGABRT`, `v8::internal::Runtime_AllocateInYoungGeneration` of stille herstarts.

**Oplossing:** vergroot het geheugen in `fly.toml`:

```toml
[[vm]]
  memory = "2048mb"
```

Of werk een bestaande machine bij:

```bash
fly machine update <machine-id> --vm-memory 2048 -y
```

512 MB is te weinig. 1 GB werkt mogelijk, maar kan bij belasting of uitgebreide logboekregistratie een OOM veroorzaken. 2 GB wordt aanbevolen.

### Problemen met de Gateway-vergrendeling

De Gateway weigert te starten met fouten die aangeven dat deze "al actief" is nadat een container opnieuw is gestart.

De runtimevergrendelingsbestanden bevinden zich in `<tmpdir>/openclaw-<uid>/gateway.<hash>.lock`
en `gateway.state.<hash>.lock` (Linux:
`/tmp/openclaw-<uid>/gateway.*.lock`), niet op het permanente volume `/data`, dus
bij een volledige herstart van de container worden ze doorgaans samen met de rest van het
containerbestandssysteem gewist. Als een vergrendeling behouden blijft (bijvoorbeeld door een `fly machine restart`
die het containerbestandssysteem behoudt) en het opstarten blokkeert, verwijder je deze
handmatig:

```bash
fly ssh console --command "rm -f /tmp/openclaw-*/gateway.*.lock"
fly machine restart <machine-id>
```

### Configuratie wordt niet gelezen

`--allow-unconfigured` omzeilt alleen de opstartbeveiliging. Het maakt of herstelt `/data/openclaw.json` niet. Zorg er daarom voor dat je werkelijke configuratie bestaat en `"gateway": { "mode": "local" }` bevat voor een normale lokale start van de Gateway.

Controleer of de configuratie bestaat:

```bash
fly ssh console --command "cat /data/openclaw.json"
```

### Configuratie schrijven via SSH

`fly ssh console -C` ondersteunt geen shellomleiding. Zo schrijf je een configuratiebestand:

```bash
# echo + tee (via een pipe van lokaal naar extern)
echo '{"your":"config"}' | fly ssh console -C "tee /data/openclaw.json"

# of sftp
fly sftp shell
> put /local/path/config.json /data/openclaw.json
```

`fly sftp` kan mislukken als het bestand al bestaat; verwijder het eerst:

```bash
fly ssh console --command "rm /data/openclaw.json"
```

### Status blijft niet behouden

Als je na een herstart authenticatieprofielen, kanaal-/providerstatus of sessies kwijtraakt, schrijft de statusmap naar het containerbestandssysteem in plaats van naar het volume.

**Oplossing:** zorg ervoor dat `OPENCLAW_STATE_DIR=/data` is ingesteld in `fly.toml` en implementeer opnieuw.

## Bijwerken

```bash
git pull
fly deploy
fly status
fly logs
```

`git pull` + `fly deploy` is hier het beheerde pad: hiermee wordt de image opnieuw vanuit het Dockerfile gebouwd, zodat de versie van de CLI/Gateway, de basis-OS-image en eventuele wijzigingen in het Dockerfile samen worden bijgewerkt. `openclaw update` in de actieve container is niet dezelfde bewerking, omdat de image wordt geleverd als een door Docker gebouwde `dist/`-structuur zonder `.git`-checkout en zonder door npm beheerde globale installatie die kan worden gedetecteerd; zie [Bijwerken](/nl/install/updating) voor die procedure bij installaties in VM-stijl.

### De machineopdracht bijwerken

Zo wijzig je de opstartopdracht zonder een volledige herimplementatie:

```bash
fly machines list
fly machine update <machine-id> --command "node dist/index.js gateway --port 3000 --bind lan" -y

# of met meer geheugen
fly machine update <machine-id> --vm-memory 2048 --command "node dist/index.js gateway --port 3000 --bind lan" -y
```

Een latere `fly deploy` zet de machineopdracht terug naar wat in `fly.toml` staat; pas handmatige wijzigingen opnieuw toe na de herimplementatie.

## Privé-implementatie (gehard)

Fly wijst standaard openbare IP-adressen toe, waardoor je Gateway bereikbaar is via `https://your-app.fly.dev` en kan worden gevonden door internetscanners (Shodan, Censys, enzovoort).

Gebruik `deploy/fly.private.toml` voor een geharde implementatie zonder **openbaar IP-adres**: hierin ontbreekt `[http_service]`, zodat er geen openbare inkomende toegang wordt toegewezen.

### Wanneer je privé-implementatie gebruikt

- Alleen uitgaande oproepen/berichten (geen inkomende Webhooks)
- ngrok- of Tailscale-tunnels verwerken eventuele Webhook-callbacks
- Toegang tot de Gateway verloopt via SSH, proxy of WireGuard in plaats van via een browser
- De implementatie moet verborgen blijven voor internetscanners

### Installatie

```bash
fly deploy -c deploy/fly.private.toml
```

Of zet een bestaande implementatie om:

```bash
# huidige IP-adressen weergeven
fly ips list -a my-openclaw

# openbare IP-adressen vrijgeven
fly ips release <public-ipv4> -a my-openclaw
fly ips release <public-ipv6> -a my-openclaw

# overschakelen naar de privéconfiguratie, zodat toekomstige implementaties geen openbare IP-adressen opnieuw toewijzen
fly deploy -c deploy/fly.private.toml

# uitsluitend privé-IPv6 toewijzen
fly ips allocate-v6 --private -a my-openclaw
```

Hierna zou `fly ips list` alleen een IP-adres van het type `private` moeten tonen:

```text
VERSIE   IP                   TYPE             REGIO
v6       fdaa:x:x:x:x::x      privé            wereldwijd
```

### Toegang tot een privé-implementatie

**Optie 1: lokale proxy (eenvoudigst)**

```bash
fly proxy 3000:3000 -a my-openclaw
# open http://localhost:3000 in een browser
```

**Optie 2: WireGuard-VPN**

```bash
fly wireguard create
# importeer dit in een WireGuard-client en krijg vervolgens toegang via het interne IPv6-adres
# voorbeeld: http://[fdaa:x:x:x:x::x]:3000
```

**Optie 3: alleen SSH**

```bash
fly ssh console -a my-openclaw
```

### Webhooks bij een privé-implementatie

Voor webhookcallbacks (Twilio, Telnyx enzovoort) zonder openbare blootstelling:

1. **ngrok-tunnel**: voer ngrok uit in de container of als sidecar
2. **Tailscale Funnel**: stel specifieke paden beschikbaar via Tailscale
3. **Alleen uitgaand**: sommige providers (Twilio) werken voor uitgaande gesprekken zonder webhooks

Voorbeeldconfiguratie voor spraakoproepen met ngrok, onder `plugins.entries.voice-call.config`:

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

De ngrok-tunnel wordt in de container uitgevoerd en biedt een openbare webhook-URL zonder de Fly-app zelf openbaar beschikbaar te maken. Stel `webhookSecurity.allowedHosts` in op de hostnaam van de tunnel, zodat doorgestuurde hostheaders worden geaccepteerd.

### Afwegingen op het gebied van beveiliging

| Aspect             | Openbaar       | Privé         |
| ------------------ | -------------- | ------------- |
| Internetscanners   | Vindbaar       | Verborgen     |
| Directe aanvallen  | Mogelijk       | Geblokkeerd   |
| Toegang tot de bedieningsinterface | Browser | Proxy/VPN |
| Webhookbezorging   | Direct         | Via tunnel    |

## Opmerkingen

- Fly.io gebruikt de x86-architectuur; het Dockerfile is compatibel met zowel x86 als ARM.
- Gebruik `fly ssh console` voor de onboarding van WhatsApp/Telegram.
- Permanente gegevens staan op het volume bij `/data`.
- Signal vereist signal-cli (een Java-gebaseerde CLI) in de image; gebruik een aangepaste image en houd het geheugen op 2GB+.

## Kosten

Met de aanbevolen configuratie (`shared-cpu-2x`, 2GB RAM) kun je, afhankelijk van het gebruik, rekenen op ongeveer $10-15/maand; de gratis laag dekt een deel van het basisquotum. Zie [prijzen van Fly.io](https://fly.io/docs/about/pricing/) voor de actuele tarieven.

## Volgende stappen

- Stel berichtenkanalen in: [Kanalen](/nl/channels)
- Configureer de Gateway: [Gateway-configuratie](/nl/gateway/configuration)
- Houd OpenClaw up-to-date: [Bijwerken](/nl/install/updating)

## Gerelateerd

- [Installatieoverzicht](/nl/install)
- [Hetzner](/nl/install/hetzner)
- [Docker](/nl/install/docker)
- [VPS-hosting](/nl/vps)
