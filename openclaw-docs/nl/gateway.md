---
read_when:
    - Het Gateway-proces uitvoeren of debuggen
summary: Runbook voor de Gateway-service, levenscyclus en beheeractiviteiten
title: Gateway-runbook
x-i18n:
    generated_at: "2026-07-27T05:51:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d8b50b6041905c321887ea0f579f8d4c3b74552b2b72c37ec655e43a53dfc130
    source_path: gateway/index.md
    workflow: 16
---

Gebruik deze pagina voor het opstarten op dag 1 en het beheer vanaf dag 2 van de Gateway-service.

<CardGroup cols={2}>
  <Card title="Diepgaande probleemoplossing" icon="siren" href="/nl/gateway/troubleshooting">
    Symptoomgerichte diagnostiek met exacte commandoreeksen en logkenmerken.
  </Card>
  <Card title="Configuratie" icon="sliders" href="/nl/gateway/configuration">
    Taakgerichte installatiehandleiding + volledige configuratiereferentie.
  </Card>
  <Card title="Beheer van geheimen" icon="key-round" href="/nl/gateway/secrets">
    SecretRef-contract, gedrag van runtime-snapshots en migratie-/herlaadbewerkingen.
  </Card>
  <Card title="Contract voor geheimenplan" icon="shield-check" href="/nl/gateway/secrets-plan-contract">
    Exacte `secrets apply`-regels voor doel/pad en gedrag van alleen-verwijzingen voor authenticatieprofielen.
  </Card>
</CardGroup>

## Lokale opstart in 5 minuten

<Steps>
  <Step title="Start de Gateway">

```bash
openclaw gateway --port 18789
# debug/trace gespiegeld naar stdio
openclaw gateway --port 18789 --verbose
# beëindig geforceerd de listener op de geselecteerde poort en start vervolgens
openclaw gateway --force
```

  </Step>

  <Step title="Controleer de servicestatus">

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
```

Gezonde uitgangssituatie: `Runtime: running`, `Connectivity probe: ok` en een `Capability`-regel die overeenkomt met wat je verwacht. Gebruik `openclaw gateway status --require-rpc` als RPC-bewijs voor leesbereik, niet alleen voor bereikbaarheid.

  </Step>

  <Step title="Valideer de gereedheid van kanalen">

```bash
openclaw channels status --probe
```

Met een bereikbare gateway voert dit live kanaalcontroles per account en optionele audits uit. Als de gateway onbereikbaar is, valt de CLI terug op kanaaloverzichten die alleen op de configuratie zijn gebaseerd.

  </Step>
</Steps>

<Note>
Het herladen van de Gateway-configuratie bewaakt het pad van het actieve configuratiebestand (afgeleid van de standaardwaarden voor profiel/status, of `OPENCLAW_CONFIG_PATH` wanneer dit is ingesteld). De standaardmodus is `gateway.reload.mode="hybrid"`. Na de eerste geslaagde laadbewerking gebruikt het actieve proces de actieve configuratiesnapshot in het geheugen; bij een geslaagde herlaadbewerking wordt die snapshot atomair vervangen.
</Note>

## Runtimemodel

- Eén permanent actief proces voor routering, het besturingsvlak en kanaalverbindingen.
- Eén gemultiplexte poort voor:
  - WebSocket-besturing/RPC
  - HTTP-API's (`/v1/models`, `/v1/embeddings`, `/v1/chat/completions`, `/v1/responses`, `/tools/invoke`)
  - HTTP-routes van Plugins, zoals de optionele `/api/v1/admin/rpc`
  - Bedieningsinterface en hooks
- Standaardbindingsmodus: `loopback`. Binnen een gedetecteerde containeromgeving is de effectieve standaardwaarde `auto` (wordt omgezet naar `0.0.0.0` voor port forwarding), tenzij Tailscale serve/funnel actief is; dat dwingt altijd `loopback` af.
- Authenticatie is standaard vereist. Configuraties met een gedeeld geheim gebruiken `gateway.auth.token` / `gateway.auth.password` (of `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`) en reverse-proxyconfiguraties buiten loopback kunnen `gateway.auth.mode: "trusted-proxy"` gebruiken.

## OpenAI-compatibele eindpunten

Het compatibiliteitsoppervlak met de grootste impact van OpenClaw:

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/chat/completions`
- `POST /v1/responses`

Waarom deze verzameling belangrijk is:

- De meeste integraties met Open WebUI, LobeChat en LibreChat controleren eerst `/v1/models`.
- Veel RAG- en geheugenpijplijnen verwachten `/v1/embeddings`.
- Clients die specifiek voor agents zijn ontworpen, geven steeds vaker de voorkeur aan `/v1/responses`.

`/v1/models` is primair voor agents: het retourneert `openclaw`, `openclaw/default` en `openclaw/<agentId>` voor elke geconfigureerde agent. `openclaw/default` is de stabiele alias die altijd naar de geconfigureerde standaardagent verwijst. Stuur `x-openclaw-model` wanneer je een andere backendprovider of een ander model wilt gebruiken; anders blijven het normale model en de embeddingconfiguratie van de geselecteerde agent leidend.

Al deze eindpunten draaien op de hoofdpoort van de Gateway en gebruiken dezelfde vertrouwde authenticatiegrens voor operators als de rest van de HTTP-API van de Gateway.

HTTP-RPC voor beheerders (`POST /api/v1/admin/rpc`) is een afzonderlijke, standaard uitgeschakelde Plugin-route voor hosthulpmiddelen die geen WebSocket-RPC kunnen gebruiken. Zie [HTTP-RPC voor beheerders](/nl/plugins/admin-http-rpc).

### Prioriteit van poort en binding

| Instelling     | Volgorde van bepaling                                                  |
| -------------- | ---------------------------------------------------------------------- |
| Gateway-poort  | `--port` → `OPENCLAW_GATEWAY_PORT` → `gateway.port` → `18789`        |
| Bindingsmodus  | CLI/overschrijving → `gateway.bind` → `loopback` (of `auto` in containers) |

Geïnstalleerde gatewayservices registreren de bepaalde `--port` in de metadata van de supervisor. Voer na het wijzigen van `gateway.port` de opdracht `openclaw doctor --fix` of `openclaw gateway install --force` uit, zodat launchd/systemd/schtasks het proces op de nieuwe poort start.

Bij het opstarten gebruikt de Gateway dezelfde effectieve poort en binding wanneer lokale oorsprongen voor de bedieningsinterface worden ingesteld voor bindingen buiten loopback. Zo stelt `--bind lan --port 3000` bijvoorbeeld `http://localhost:3000` en `http://127.0.0.1:3000` in voordat de runtimevalidatie wordt uitgevoerd. Voeg oorsprongen voor externe browsers, zoals HTTPS-proxy-URL's, expliciet toe aan `gateway.controlUi.allowedOrigins`.

### Modi voor direct herladen

| `gateway.reload.mode` | Gedrag                                     |
| --------------------- | ------------------------------------------ |
| `off`                 | Configuratie niet herladen                 |
| `hot`                 | Alleen wijzigingen toepassen die direct veilig zijn |
| `restart`             | Opnieuw starten bij wijzigingen die herladen vereisen |
| `hybrid` (standaard)  | Direct toepassen wanneer veilig, opnieuw starten wanneer vereist |

## Opdrachtenset voor operators

```bash
openclaw gateway status
openclaw gateway status --deep   # voegt een servicecontrole op systeemniveau toe
openclaw gateway status --json
openclaw gateway install
openclaw gateway restart
openclaw gateway stop
openclaw secrets reload
openclaw logs --follow
openclaw doctor
```

`gateway status --deep` is bedoeld voor aanvullende servicedetectie (LaunchDaemons/systemd-systeemeenheden/schtasks), niet voor een diepgaandere RPC-statuscontrole.

## Meerdere gateways (dezelfde host)

Voor de meeste installaties moet één gateway per machine worden uitgevoerd. Eén gateway kan meerdere agents en kanalen hosten. Je hebt alleen meerdere gateways nodig wanneer je bewust isolatie of een reddingsbot wilt.

Nuttige controles:

```bash
openclaw gateway status --deep
openclaw gateway probe
```

Wat je kunt verwachten:

- `gateway status --deep` kan `Other gateway-like services detected (best effort)` melden en opschoningstips weergeven wanneer verouderde installaties van launchd/systemd/schtasks nog aanwezig zijn.
- `gateway probe` kan waarschuwen voor `multiple reachable gateway identities` wanneer verschillende gateways antwoorden, of wanneer OpenClaw niet kan aantonen dat bereikbare doelen dezelfde gateway zijn. Een SSH-tunnel, proxy-URL of geconfigureerde externe URL naar dezelfde gateway is één gateway met meerdere transportmethoden, zelfs wanneer de transportpoorten verschillen.
- Als dit de bedoeling is, isoleer dan de poorten, configuratie/status en werkmaphoofdmappen per gateway.

Controlelijst per instantie:

- Unieke `gateway.port`
- Unieke `OPENCLAW_CONFIG_PATH`
- Unieke `OPENCLAW_STATE_DIR`
- Unieke `agents.defaults.workspace`

Voorbeeld:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json OPENCLAW_STATE_DIR=~/.openclaw-a openclaw gateway --port 19001
OPENCLAW_CONFIG_PATH=~/.openclaw/b.json OPENCLAW_STATE_DIR=~/.openclaw-b openclaw gateway --port 19002
```

Gedetailleerde configuratie: [/gateway/multiple-gateways](/nl/gateway/multiple-gateways).

## Externe toegang

Aanbevolen: Tailscale/VPN.
Alternatief: SSH-tunnel.

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

Verbind clients vervolgens lokaal met `ws://127.0.0.1:18789`.

<Warning>
SSH-tunnels omzeilen de gatewayauthenticatie niet. Voor authenticatie met een gedeeld geheim moeten clients zelfs via de tunnel nog steeds
`token`/`password` verzenden. Voor modi die een identiteit bevatten,
moet het verzoek nog steeds aan dat authenticatiepad voldoen.
</Warning>

Zie: [Externe Gateway](/nl/gateway/remote), [Authenticatie](/nl/gateway/authentication), [Tailscale](/nl/gateway/tailscale).

## Supervisie en servicelevenscyclus

Gebruik uitvoeringen onder supervisie voor productiewaardige betrouwbaarheid.

<Tabs>
  <Tab title="macOS (launchd)">

```bash
openclaw gateway install
openclaw gateway status
openclaw gateway restart
openclaw gateway stop
```

Gebruik `openclaw gateway restart` om opnieuw te starten. Koppel `openclaw gateway stop` en `openclaw gateway start` niet aan elkaar als vervanging voor opnieuw starten.

Op macOS gebruikt `gateway stop` standaard `launchctl bootout`. Hierdoor wordt de LaunchAgent uit de huidige opstartsessie verwijderd zonder deze permanent uit te schakelen, zodat automatisch herstel via KeepAlive na onverwachte crashes blijft werken en `gateway start` deze correct opnieuw inschakelt. Geef `--disable` door om automatisch opnieuw starten na een herstart permanent te onderdrukken: `openclaw gateway stop --disable`.

LaunchAgent-labels zijn `ai.openclaw.gateway` (standaard) of `ai.openclaw.<profile>` (benoemd profiel). `openclaw doctor` controleert en herstelt afwijkingen in de serviceconfiguratie.

  </Tab>

  <Tab title="Linux (systemd-gebruiker)">

```bash
openclaw gateway install
systemctl --user enable --now openclaw-gateway[-<profile>].service
openclaw gateway status
```

Schakel lingering in om de service na het afmelden actief te houden:

```bash
sudo loginctl enable-linger $(whoami)
```

Zorg er op een headless server zonder desktopsessie ook voor dat `XDG_RUNTIME_DIR` is ingesteld (`export XDG_RUNTIME_DIR=/run/user/$(id -u)`) voordat je `systemctl --user`-opdrachten opnieuw probeert.

Voorbeeld van een handmatige gebruikerseenheid wanneer je een aangepast installatiepad nodig hebt:

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target
StartLimitBurst=5
StartLimitIntervalSec=60

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
RestartPreventExitStatus=78
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
OOMPolicy=continue
KillMode=control-group

[Install]
WantedBy=default.target
```

  </Tab>

  <Tab title="Windows (native)">

```powershell
openclaw gateway install
openclaw gateway status --json
openclaw gateway restart
openclaw gateway stop
```

Voor beheerd opstarten op native Windows wordt een Scheduled Task met de naam `OpenClaw Gateway`
gebruikt (of `OpenClaw Gateway (<profile>)` voor benoemde profielen). Als het maken van een Scheduled Task
wordt geweigerd, valt OpenClaw terug op een startprogramma per gebruiker in de Startup-map
dat verwijst naar `gateway.cmd` in de statusmap.

  </Tab>

  <Tab title="Linux (systeemservice)">

Gebruik een systeemeenheid voor hosts met meerdere gebruikers of hosts die altijd actief zijn.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway[-<profile>].service
```

Gebruik dezelfde service-inhoud als voor de gebruikerseenheid, maar installeer deze onder
`/etc/systemd/system/openclaw-gateway[-<profile>].service` en pas
`ExecStart=` aan als je binaire bestand `openclaw` zich ergens anders bevindt.

Laat `openclaw doctor --fix` niet daarnaast een gatewayservice op gebruikersniveau installeren voor hetzelfde profiel/dezelfde poort. Doctor weigert die automatische installatie wanneer een OpenClaw-gatewayservice op systeemniveau wordt gevonden; gebruik `OPENCLAW_SERVICE_REPAIR_POLICY=external` wanneer de systeemeenheid eigenaar is van de levenscyclus.

  </Tab>
</Tabs>

Bij fouten door een ongeldige configuratie wordt het proces afgesloten met code `78`. Linux-systemd-eenheden gebruiken `RestartPreventExitStatus=78` om te stoppen met opnieuw starten totdat de configuratie is hersteld. launchd en Windows Task Scheduler hebben geen gelijkwaardige stopregel per afsluitcode. Daarom bewaart de Gateway ook de geschiedenis van snelle, onjuiste opstartpogingen en onderdrukt deze het automatisch starten van kanaal-/provideraccounts na herhaalde opstartfouten. In die veilige modus start het besturingsvlak nog steeds voor inspectie en herstel, weigeren directe herlaadbewerkingen van de configuratie en `secrets.reload` het automatisch opnieuw starten van kanalen, en kan een expliciet `channels.start`-verzoek van een operator de onderdrukking opheffen.

## Snel pad voor ontwikkelprofiel

```bash
openclaw --dev setup
openclaw --dev gateway --allow-unconfigured
openclaw --dev status
```

Standaardwaarden omvatten geïsoleerde status/configuratie en Gateway-basispoort `19001`.

## Beknopt protocoloverzicht (operatorperspectief)

- Het eerste clientframe moet `connect` zijn.
- De Gateway retourneert een `hello-ok`-frame met een `snapshot` (`presence`, `health`, `stateVersion`, `uptimeMs`) plus `policy`-limieten (`maxPayload`, `maxBufferedBytes`, `tickIntervalMs`).
- `hello-ok.features.methods` / `events` vormen een conservatieve lijst voor detectie, niet
  een gegenereerde dump van elke aanroepbare hulproute.
- Verzoeken: `req(method, params)` → `res(ok/payload|error)`.
- Veelvoorkomende gebeurtenissen zijn onder meer `connect.challenge`, `agent`, `chat`,
  `session.message`, `session.operation`, `session.tool`, optionele
  `session.approval`, `sessions.changed`, `presence`, `tick`, `health`,
  `heartbeat`, levenscyclusgebeurtenissen voor koppeling/goedkeuring en `shutdown`.

Agentuitvoeringen bestaan uit twee fasen:

1. Onmiddellijke bevestiging van acceptatie (`status:"accepted"`)
2. Definitief voltooiingsantwoord (`status:"ok"|"error"`), met tussendoor gestreamde `agent`-gebeurtenissen.

Zie de volledige protocoldocumentatie: [Gateway-protocol](/nl/gateway/protocol).

## Operationele controles

### Beschikbaarheid

- Open WS en verzend `connect`.
- Verwacht een `hello-ok`-antwoord met momentopname.

### Gereedheid

```bash
openclaw gateway status
openclaw channels status --probe
openclaw health
```

### Herstel na hiaten

Gebeurtenissen worden niet opnieuw afgespeeld. Vernieuw bij hiaten in de reeks de status (`health`, `system-presence`) voordat je doorgaat.

## Veelvoorkomende foutsignaturen

| Signatuur                                                      | Waarschijnlijk probleem                                                        |
| -------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `refusing to bind gateway ... without auth`                    | Binding buiten loopback zonder geldig authenticatiepad voor de Gateway          |
| `another gateway instance is already listening` / `EADDRINUSE` | Poortconflict                                                                   |
| `Gateway start blocked: set gateway.mode=local`                | Configuratie is ingesteld op externe modus, of `gateway.mode` ontbreekt in een beschadigde configuratie |
| `unauthorized` tijdens het verbinden                           | Authenticatie komt niet overeen tussen client en Gateway                        |

Gebruik [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting) voor volledige diagnosestappen.

## Veiligheidsgaranties

- Clients van het Gateway-protocol stoppen onmiddellijk met een fout wanneer de Gateway niet beschikbaar is (geen impliciete terugval naar een rechtstreeks kanaal).
- Ongeldige eerste frames of eerste frames die geen verbindingsframes zijn, worden geweigerd en gesloten.
- Bij correct afsluiten wordt vóór het sluiten van de socket een `shutdown`-gebeurtenis verzonden.

## Gerelateerd

- [Configuratie](/nl/gateway/configuration)
- [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting)
- [Achtergrondproces](/nl/gateway/background-process)
- [Status](/nl/gateway/health)
- [Doctor](/nl/gateway/doctor)
- [Authenticatie](/nl/gateway/authentication)
- [Externe toegang](/nl/gateway/remote)
- [Geheimenbeheer](/nl/gateway/secrets)
