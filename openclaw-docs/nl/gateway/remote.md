---
read_when:
    - Externe Gateway-configuraties uitvoeren of problemen ermee oplossen
summary: Externe toegang via Gateway WS, SSH-tunnels en tailnets
title: Externe toegang
x-i18n:
    generated_at: "2026-07-27T05:01:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8f05e32fcfa16d5ddfcd684d0550c9af311914e2b4d91c95edad3490dc2e56d9
    source_path: gateway/remote.md
    workflow: 16
---

OpenClaw voert één Gateway (de master) uit op een host en verbindt elke client ermee. De Gateway beheert sessies, authenticatieprofielen, kanalen en status; al het overige is een client.

- **Operators** (jij of de macOS-app): een rechtstreekse LAN/Tailnet-WebSocket is het eenvoudigst wanneer de Gateway bereikbaar is; SSH-tunneling is de universele terugvaloptie.
- **Nodes** (iOS/Android en andere apparaten): maken verbinding met de **WebSocket** van de Gateway (LAN/tailnet of SSH-tunnel).

## Het kernidee

De WebSocket van de Gateway bindt standaard aan **loopback**, op poort `18789` (`gateway.port`). Voor extern gebruik maak je deze beschikbaar via Tailscale Serve / een vertrouwde LAN-Tailnet-binding, of stuur je de loopback-poort door via SSH.

## Topologieopties

| Opstelling                         | Waar de Gateway wordt uitgevoerd                                                                          | Meest geschikt voor                                                                                                                                         |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Altijd actieve Gateway in je tailnet | Permanente host (VPS of thuisserver), bereikbaar via Tailscale of SSH                                      | Laptops die vaak in de slaapstand staan, maar waarvoor de agent altijd actief moet zijn. Zie [exe.dev](/nl/install/exe-dev) (eenvoudige VM) of [Hetzner](/nl/install/hetzner) (productie-VPS). |
| Desktopcomputer thuis             | Desktopcomputer; laptop maakt extern verbinding via de externe modus van de macOS-app (Settings → Connection → OpenClaw runs) | De agent uitvoeren op hardware die ingeschakeld blijft. Draaiboek: [externe toegang voor macOS](/nl/platforms/mac/remote).                                      |
| Laptop                            | Laptop, veilig beschikbaar gemaakt via een SSH-tunnel of Tailscale Serve (behoud `gateway.bind: "loopback"`)       | Opstellingen met één computer. Zie [Tailscale](/nl/gateway/tailscale) en [Web](/nl/web).                                                                           |

Voor de altijd actieve opstelling en de laptopopstelling wordt aangeraden `gateway.bind: "loopback"` te behouden en **Tailscale Serve** voor de Control UI te gebruiken, of een vertrouwde LAN/Tailnet-binding met `gateway.remote.transport: "direct"`. Een SSH-tunnel is de terugvaloptie die vanaf elke computer werkt.

## Opdrachtstroom (wat waar wordt uitgevoerd)

Eén Gateway beheert status en kanalen; Nodes zijn randapparaten. Voorbeeld (een Telegram-bericht dat naar een Node-tool wordt doorgestuurd):

1. Een Telegram-bericht komt aan bij de **Gateway**.
2. De Gateway voert de **agent** uit, die beslist of een Node-tool wordt aangeroepen.
3. De Gateway roept de **Node** aan via de WebSocket van de Gateway (`node.invoke`-RPC).
4. De Node retourneert het resultaat; de Gateway antwoordt via Telegram.

Nodes voeren de Gateway-service niet uit. Per host mag slechts één Gateway worden uitgevoerd, tenzij je bewust geïsoleerde profielen gebruikt (zie [Meerdere Gateways](/nl/gateway/multiple-gateways)). De 'node mode' van de macOS-app is slechts een Node-client die via de WebSocket van de Gateway werkt.

## SSH-tunnel (CLI + tools)

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

Wanneer de tunnel actief is, bereiken `openclaw health` en `openclaw status --deep` de externe Gateway via `ws://127.0.0.1:18789`. `openclaw gateway status`, `openclaw gateway health`, `openclaw gateway probe` en `openclaw gateway call` kunnen via `--url` ook een doorgestuurde URL gebruiken.

<Note>
Vervang `18789` door je geconfigureerde `gateway.port` (of `--port` / `OPENCLAW_GATEWAY_PORT`).
</Note>

<Warning>
`--url` valt nooit terug op configuratie- of omgevingsreferenties. Geef `--token` of `--password` expliciet door; zonder deze gegevens verzendt de client geen referenties en mislukt de verbinding als de doel-Gateway authenticatie vereist.
</Warning>

## Standaardwaarden voor externe CLI-toegang

Sla een extern doel op, zodat CLI-opdrachten dit standaard gebruiken:

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://127.0.0.1:18789",
      token: "your-token",
    },
  },
}
```

Wanneer de Gateway alleen via loopback bereikbaar is, behoud je de URL `ws://127.0.0.1:18789` en open je eerst de SSH-tunnel. Bij het SSH-tunneltransport van de macOS-app komt de gedetecteerde hostnaam van de Gateway in `gateway.remote.sshTarget` (`user@host` of `user@host:port`); `gateway.remote.url` blijft de lokale tunnel-URL. Als de externe poort afwijkt van de lokale poort, stel je `gateway.remote.remotePort` in.

Hostsleutelverificatie is standaard strikt (`gateway.remote.sshHostKeyPolicy: "strict"`). Stel dit in op `"openssh"` om de verificatie in plaats daarvan aan je effectieve OpenSSH-configuratie over te laten; controleer je gebruikers- en systeeminstellingen voor SSH voordat je dit inschakelt.

Gebruik de rechtstreekse modus voor een Gateway die al via een vertrouwd LAN of Tailnet bereikbaar is:

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      transport: "direct",
      url: "ws://192.168.0.202:18789",
      token: "your-token",
    },
  },
}
```

## Prioriteitsvolgorde van referenties

De resolutie van Gateway-referenties volgt één gedeeld contract voor aanroep-, probe- en statuspaden en voor de bewaking van uitvoeringsgoedkeuringen in Discord. De Node-host gebruikt hetzelfde contract, met één uitzondering voor de lokale modus (deze negeert `gateway.remote.*`).

- Expliciete referenties (`--token`, `--password` of de `gatewayToken` van een tool) hebben altijd voorrang op aanroeppaden die expliciete authenticatie accepteren.
- Veiligheid bij URL-overschrijvingen:
  - CLI-`--url` gebruikt nooit impliciete configuratie- of omgevingsreferenties opnieuw.
  - Omgevingsvariabele `OPENCLAW_GATEWAY_URL` mag alleen omgevingsreferenties gebruiken (`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`).
- Standaardwaarden voor lokale modus:
  - token: `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token` -> `gateway.remote.token` (alleen externe terugval wanneer het lokale token niet is ingesteld)
  - wachtwoord: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.auth.password` -> `gateway.remote.password` (alleen externe terugval wanneer het lokale wachtwoord niet is ingesteld)
- Standaardwaarden voor externe modus:
  - token: `gateway.remote.token` -> `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token`
  - wachtwoord: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.remote.password` -> `gateway.auth.password`
- Uitzondering voor lokale modus van de Node-host: `gateway.remote.token` / `gateway.remote.password` worden genegeerd.
- Tokencontroles voor externe probes/status zijn standaard strikt: bij gebruik van de externe modus gebruiken ze alleen `gateway.remote.token` (geen terugval op een lokaal token).
- Omgevingsoverschrijvingen voor de Gateway gebruiken alleen `OPENCLAW_GATEWAY_*`.

## Externe toegang tot de chatinterface

WebChat heeft geen afzonderlijke HTTP-poort; de SwiftUI-chatinterface maakt rechtstreeks verbinding met de WebSocket van de Gateway.

- Stuur `18789` door via SSH (zie hierboven) en verbind clients vervolgens met `ws://127.0.0.1:18789`.
- Voor de rechtstreekse LAN/Tailnet-modus verbind je clients met de geconfigureerde privé-URL `ws://` of beveiligde URL `wss://`.
- Op macOS beheert de externe modus van de app het geselecteerde transport automatisch.

## Externe modus van de macOS-app

De menubalkapp voor macOS beheert dezelfde opstelling van begin tot eind: externe statuscontroles, WebChat en het doorsturen van Voice Wake. Draaiboek: [externe toegang voor macOS](/nl/platforms/mac/remote).

## Beveiligingsregels (extern/VPN)

Houd de Gateway **alleen via loopback bereikbaar**, tenzij je zeker weet dat een binding nodig is.

- **Loopback + SSH/Tailscale Serve** is de veiligste standaardinstelling (geen openbare blootstelling).
- Niet-versleutelde `ws://` wordt geaccepteerd voor loopback-, privé-/LAN- (RFC 1918), link-local-, CGNAT-, `.local`- en `.ts.net`-hosts. Openbare externe hosts moeten `wss://` gebruiken.
- **Niet-loopback-bindingen** (`lan`/`tailnet`/`custom`, of `auto` wanneer loopback niet beschikbaar is) moeten Gateway-authenticatie gebruiken: een token, wachtwoord of identiteitsbewuste reverse proxy met `gateway.auth.mode: "trusted-proxy"`.
- `gateway.remote.token` / `.password` zijn bronnen voor clientreferenties; ze configureren op zichzelf geen serverauthenticatie.
- Lokale aanroeppaden mogen `gateway.remote.*` alleen als terugval gebruiken wanneer `gateway.auth.*` niet is ingesteld.
- Als `gateway.auth.token` / `gateway.auth.password` expliciet via SecretRef is geconfigureerd en niet kan worden opgelost, mislukt de resolutie gesloten (zonder maskerende externe terugval).
- `gateway.remote.tlsFingerprint` legt het externe TLS-certificaat voor `wss://` vast, voor zowel operator-/besturingsverkeer als de bijbehorende Node in de rechtstreekse macOS-modus. Zonder opgeslagen vastlegging legt macOS het certificaat bij het eerste gebruik pas vast nadat de normale systeemvertrouwenscontrole is geslaagd; Gateways met een zelfondertekend certificaat of privé-CA vereisen een expliciete vingerafdruk of externe toegang via SSH.
- **Tailscale Serve** kan Control UI-/WebSocket-verkeer via identiteitsheaders authenticeren wanneer `gateway.auth.allowTailscale: true`. HTTP API-eindpunten gebruiken deze headerauthenticatie niet en volgen in plaats daarvan de normale HTTP-authenticatiemodus van de Gateway. Deze tokenloze stroom veronderstelt dat de Gateway-host wordt vertrouwd; stel dit in op `false` om overal authenticatie met een gedeeld geheim te gebruiken.
- **Trusted-proxy**-authenticatie verwacht standaard een niet-loopback, identiteitsbewuste proxy. Reverse proxies op dezelfde host via loopback vereisen expliciet `gateway.auth.trustedProxy.allowLoopback = true`.
- Behandel browserbesturing als operatortoegang: alleen via het tailnet en met bewuste Node-koppeling.

Uitgebreide informatie: [Beveiliging](/nl/gateway/security).

### macOS: permanente SSH-tunnel via LaunchAgent

Voor macOS-clients gebruikt de eenvoudigste permanente opstelling een SSH-`LocalForward`-configuratievermelding plus een LaunchAgent die de tunnel actief houdt na herstarts en crashes.

#### Stap 1: SSH-configuratie toevoegen

Bewerk `~/.ssh/config`:

```ssh
Host remote-gateway
    HostName <REMOTE_IP>
    User <REMOTE_USER>
    LocalForward 18789 127.0.0.1:18789
    IdentityFile ~/.ssh/id_rsa
```

Vervang `<REMOTE_IP>` en `<REMOTE_USER>` door je eigen waarden.

#### Stap 2: SSH-sleutel kopiëren (eenmalig)

```bash
ssh-copy-id -i ~/.ssh/id_rsa <REMOTE_USER>@<REMOTE_IP>
```

#### Stap 3: het Gateway-token configureren

```bash
openclaw config set gateway.remote.token "<your-token>"
```

Gebruik in plaats daarvan `gateway.remote.password` als de externe Gateway wachtwoordauthenticatie gebruikt. `OPENCLAW_GATEWAY_TOKEN` blijft geldig als overschrijving op shellniveau, maar voor een duurzame externe clientopstelling gebruik je `gateway.remote.token` / `gateway.remote.password`.

#### Stap 4: de LaunchAgent maken

Sla op als `~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.ssh-tunnel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/ssh</string>
        <string>-N</string>
        <string>remote-gateway</string>
    </array>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

#### Stap 5: de LaunchAgent laden

```bash
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist
```

De tunnel start automatisch bij het aanmelden, wordt na een crash opnieuw gestart en houdt de doorgestuurde poort actief.

<Note>
Als je nog een `com.openclaw.ssh-tunnel`-LaunchAgent van een oudere opstelling hebt, verwijder deze dan uit het geheugen en verwijder het bestand.
</Note>

#### Probleemoplossing

```bash
# Controleren of de tunnel actief is
ps aux | grep "ssh -N remote-gateway" | grep -v grep
lsof -i :18789

# De tunnel opnieuw starten
launchctl kickstart -k gui/$UID/ai.openclaw.ssh-tunnel

# De tunnel stoppen
launchctl bootout gui/$UID/ai.openclaw.ssh-tunnel
```

| Configuratie-item                    | Functie                                                      |
| ------------------------------------ | ------------------------------------------------------------ |
| `LocalForward 18789 127.0.0.1:18789` | Stuurt lokale poort 18789 door naar externe poort 18789       |
| `ssh -N`                             | SSH zonder externe opdrachten uit te voeren (alleen poortdoorsturing) |
| `KeepAlive`                          | Start de tunnel automatisch opnieuw als deze crasht          |
| `RunAtLoad`                          | Start de tunnel wanneer de LaunchAgent bij het aanmelden wordt geladen |

## Gerelateerd

- [Tailscale](/nl/gateway/tailscale)
- [Authenticatie](/nl/gateway/authentication)
- [Externe Gateway instellen](/nl/gateway/remote-gateway-readme)
