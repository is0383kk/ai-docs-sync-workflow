---
read_when:
    - Bonjour-detectieproblemen op macOS/iOS oplossen
    - mDNS-servicetypen, TXT-records of de gebruikerservaring voor detectie wijzigen
summary: Bonjour-/mDNS-detectie en foutopsporing (Gateway-bakens, clients en veelvoorkomende foutmodi)
title: Bonjour-detectie
x-i18n:
    generated_at: "2026-07-27T04:58:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f43ef71b323b59362655c390a4df621c2571abbe3b2c1cd2728918c6f76d6f99
    source_path: gateway/bonjour.md
    workflow: 16
---

OpenClaw kan Bonjour (mDNS/DNS-SD) gebruiken om een actieve Gateway (WebSocket-eindpunt) te ontdekken. Browsen via multicast `local.` is een **gemaksfunctie die alleen op het LAN werkt**: de gebundelde `bonjour`-Plugin beheert LAN-advertenties, start automatisch op macOS-hosts en is opt-in op Linux, Windows en gecontaineriseerde Gateway-implementaties. Hetzelfde baken kan ook via een geconfigureerd wide-area DNS-SD-domein publiceren voor detectie tussen netwerken. Detectie werkt op basis van beste inspanning en vervangt connectiviteit via SSH of Tailnet **niet**.

## Wide-area Bonjour (unicast DNS-SD) via Tailscale

Als de Node en Gateway zich op verschillende netwerken bevinden, kan multicast-mDNS de grens niet passeren. Behoud dezelfde detectie-UX door over te schakelen op **unicast DNS-SD** ("Wide-Area Bonjour") via Tailscale:

1. Voer op de Gateway-host een DNS-server uit die via het Tailnet bereikbaar is.
2. Publiceer DNS-SD-records voor `_openclaw-gw._tcp` onder een afzonderlijke zone (voorbeeld: `openclaw.internal.`).
3. Configureer **split DNS** van Tailscale zodat je gekozen domein voor clients, waaronder iOS, via die DNS-server wordt omgezet.

`openclaw.internal.` hierboven is slechts een voorbeeld — OpenClaw ondersteunt elk detectiedomein. iOS-/Android-Nodes browsen zowel `local.` als je geconfigureerde wide-area-domein.

### Gateway-configuratie

```json5
{
  gateway: { bind: "tailnet" }, // alleen Tailnet (aanbevolen)
  discovery: { wideArea: { enabled: true, domain: "openclaw.internal" } },
}
```

`discovery.wideArea.domain` accepteert ook de omgevingsvariabele `OPENCLAW_WIDE_AREA_DOMAIN` als terugvaloptie wanneer deze niet is ingesteld.

### Eenmalige instelling van de DNS-server (Gateway-host, alleen macOS)

```bash
openclaw dns setup --apply
```

Deze opdracht werkt alleen op macOS en vereist Homebrew en een actieve Tailscale-verbinding. De opdracht installeert CoreDNS (`brew install coredns`) en configureert dit om:

- alleen op de Tailscale-interfaces van de Gateway op poort 53 te luisteren
- je gekozen domein (voorbeeld: `openclaw.internal.`) vanuit `~/.openclaw/dns/<domain>.db` aan te bieden

Voer de opdracht eerst zonder `--apply` uit om een voorbeeld van het plan te bekijken (domein, pad van het zonebestand, gedetecteerd Tailnet-IP, aanbevolen configuratie), zonder iets te installeren.

Valideer vanaf een machine die met het Tailnet is verbonden:

```bash
dns-sd -B _openclaw-gw._tcp openclaw.internal.
dig @<TAILNET_IPV4> -p 53 _openclaw-gw._tcp.openclaw.internal PTR +short
```

### DNS-instellingen van Tailscale

In de Tailscale-beheerconsole:

- Voeg een naamserver toe die naar het Tailnet-IP van de Gateway verwijst (UDP/TCP 53).
- Voeg split DNS toe zodat je detectiedomein die naamserver gebruikt.

Zodra clients Tailnet-DNS accepteren, kunnen iOS-Nodes en CLI-detectie zonder multicast door `_openclaw-gw._tcp` in je detectiedomein browsen.

### Beveiliging van de Gateway-listener

De WS-poort van de Gateway (standaard `18789`) bindt standaard aan loopback. Bind voor LAN-/Tailnet-toegang expliciet en houd authenticatie ingeschakeld. Stel voor configuraties die alleen Tailnet gebruiken `gateway.bind: "tailnet"` in `~/.openclaw/openclaw.json` in en start de Gateway (of de macOS-menubalkapp) opnieuw.

## Wat er adverteert

Alleen de Gateway adverteert `_openclaw-gw._tcp`. LAN-multicastadvertenties zijn bij inschakeling afkomstig van de gebundelde `bonjour`-Plugin; publicatie via wide-area DNS-SD blijft eigendom van de Gateway.

## Servicetypen

- `_openclaw-gw._tcp` - transportbaken van de Gateway, gebruikt door macOS-/iOS-/Android-Nodes.

## TXT-sleutels (niet-geheime hints)

| Sleutel                        | Indien aanwezig                                                                |
| ----------------------------- | ------------------------------------------------------------------------------ |
| `role=gateway`                | Altijd.                                                                        |
| `displayName=<friendly name>` | Altijd.                                                                        |
| `lanHost=<hostname>.local`    | Altijd.                                                                        |
| `gatewayPort=<port>`          | Altijd (Gateway-WS + HTTP).                                                     |
| `transport=gateway`           | Altijd.                                                                        |
| `gatewayTls=1`                | Alleen wanneer TLS is ingeschakeld.                                            |
| `gatewayTlsSha256=<sha256>`   | Alleen wanneer TLS is ingeschakeld en een vingerafdruk beschikbaar is.         |
| `gatewayDirectReachable=1`    | Alleen wanneer de Gateway rechtstreeks bereikbaar is (niet uitsluitend via een relay-/proxypad). |
| `canvasPort=<port>`           | Alleen wanneer de canvashost is ingeschakeld; momenteel hetzelfde als `gatewayPort`. |
| `tailnetDns=<magicdns>`       | Alleen in volledige mDNS-modus; optionele hint wanneer Tailnet beschikbaar is. |
| `sshPort=<port>`              | Alleen in volledige modus; weggelaten in de minimale en uitgeschakelde modus.  |
| `cliPath=<path>`              | Alleen in volledige modus; weggelaten in de minimale en uitgeschakelde modus.  |

Beveiligingsopmerkingen:

- Bonjour-/mDNS-TXT-records zijn **niet geauthenticeerd**. Clients mogen TXT niet als gezaghebbende routeringsinformatie beschouwen.
- Clients moeten routeren via het omgezette service-eindpunt (SRV + A/AAAA). Beschouw `lanHost`, `tailnetDns`, `gatewayPort` en `gatewayTlsSha256` uitsluitend als hints.
- Automatische SSH-doelbepaling moet eveneens de omgezette servicehost gebruiken, niet uitsluitend TXT-hints.
- TLS-pinning mag nooit toestaan dat een geadverteerde `gatewayTlsSha256` een eerder opgeslagen pin overschrijft.
- iOS-/Android-Nodes moeten rechtstreekse verbindingen op basis van detectie behandelen als **uitsluitend TLS** en expliciete gebruikersbevestiging vereisen voordat een vingerafdruk voor het eerst wordt vertrouwd.

## Foutopsporing op macOS

Ingebouwde hulpmiddelen:

```bash
# Door instanties browsen
dns-sd -B _openclaw-gw._tcp local.

# Eén instantie omzetten (vervang <instance>)
dns-sd -L "<instance>" _openclaw-gw._tcp local.
```

Als browsen werkt maar omzetten mislukt, is er meestal sprake van een probleem met LAN-beleid of de mDNS-resolver.

## Foutopsporing in Gateway-logboeken

De Gateway schrijft een roterend logbestand (bij het opstarten weergegeven als `gateway log file: ...`). Zoek naar regels met `bonjour:`, met name:

- `bonjour: advertise failed ...`
- `bonjour: suppressing ciao netmask assertion ...`
- `bonjour: ... name conflict resolved` / `hostname conflict resolved`

OpenClaw start elke Bonjour-service één keer en laat peilen, opnieuw proberen, oplossen van naamconflicten en opnieuw publiceren na interfacewijzigingen over aan de mDNS-responder. Dit voorkomt overlappende publicatiepogingen tijdens normale netwerkveranderingen. Herhaalde interne zelfpeilingsberichten worden onderdrukt, zodat ze het Gateway-logboek niet kunnen overspoelen.

Wanneer meerdere OpenClaw-Gateways vanaf dezelfde host adverteren, kan Bonjour achtervoegsels zoals `(2)` of `(3)` toevoegen om namen van service-instanties uniek te houden. Deze achtervoegsels zijn normale conflictoplossing en duiden niet op dubbel OCM-toezicht.

Bonjour gebruikt de systeemhostnaam voor de geadverteerde `.local`-host wanneer deze een geldig DNS-label is. Als de systeemhostnaam spaties, underscores of een ander ongeldig teken voor DNS-labels bevat, valt OpenClaw terug op `openclaw.local`. Stel `OPENCLAW_MDNS_HOSTNAME=<name>` in voordat je de Gateway start wanneer je een expliciet hostlabel nodig hebt.

## Foutopsporing op een iOS-Node

De iOS-Node gebruikt `NWBrowser` om `_openclaw-gw._tcp` te ontdekken.

Logboeken vastleggen: Settings -> Gateway -> Advanced -> **Discovery Debug Logs**, daarna Settings -> Gateway -> Advanced -> **Discovery Logs** -> reproduce -> **Copy**. Het logboek bevat statusovergangen van de browser en wijzigingen in de resultatenset.

## Wanneer Bonjour moet worden ingeschakeld

Bonjour start automatisch wanneer de Gateway op een macOS-host met een lege configuratie wordt gestart, omdat de lokale app en iOS-/Android-Nodes in de buurt doorgaans afhankelijk zijn van detectie op hetzelfde LAN.

Schakel het expliciet in wanneer automatische detectie op hetzelfde LAN nuttig is op Linux, Windows of een andere niet-macOS-host:

```bash
openclaw plugins enable bonjour
```

Wanneer Bonjour is ingeschakeld, gebruikt het `discovery.mdns.mode` om te bepalen hoeveel TXT-metadata worden gepubliceerd; dezelfde modus beheert optionele TXT-hints in wide-area DNS-SD-records. Modi:

| Modus                | Gedrag                                                                                                                                   |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `minimal` (standaard) | Alleen kern-TXT-sleutels; laat `sshPort`, `cliPath`, `tailnetDns` weg.                                                           |
| `full`              | Voegt `sshPort`, `cliPath`, `tailnetDns` toe — gebruik dit wanneer clients die hints nodig hebben.                              |
| `off`               | Onderdrukt LAN-multicast zonder de inschakeling van de Plugin te wijzigen; wide-area DNS-SD kan nog steeds publiceren wanneer `discovery.wideArea.domain` is ingesteld. |

## Wanneer Bonjour moet worden uitgeschakeld

Laat Bonjour uitgeschakeld wanneer LAN-multicastadvertenties onnodig, niet beschikbaar of schadelijk zijn — veelvoorkomende gevallen zijn niet-macOS-servers, Docker-bridgenetwerken, WSL of netwerkbeleid dat mDNS-multicast blokkeert. De Gateway blijft bereikbaar via de gepubliceerde URL, SSH, Tailnet of wide-area DNS-SD; alleen automatische LAN-detectie is onbetrouwbaar.

Gebruik de omgevingsoverschrijving voor implementatiespecifieke problemen (veilig voor Docker-images, servicebestanden, startscripts en eenmalige foutopsporing — deze verdwijnt wanneer de omgeving verdwijnt):

```bash
OPENCLAW_DISABLE_BONJOUR=1
```

Gebruik de Plugin-configuratie wanneer je de gebundelde Plugin voor LAN-detectie bewust wilt uitschakelen voor die OpenClaw-configuratie:

```bash
openclaw plugins disable bonjour
```

## Aandachtspunten voor Docker

De gebundelde Bonjour-Plugin schakelt LAN-multicastadvertenties automatisch uit in gedetecteerde containers wanneer `OPENCLAW_DISABLE_BONJOUR` niet is ingesteld. Docker-bridgenetwerken sturen mDNS-multicast (`224.0.0.251:5353`) doorgaans niet door tussen de container en het LAN, waardoor advertenties vanuit de container detectie zelden laten werken.

Aandachtspunten:

- Bonjour start automatisch op macOS-hosts en is elders opt-in. Als het uitgeschakeld blijft, wordt de Gateway niet gestopt — alleen LAN-multicastadvertenties worden overgeslagen.
- Het uitschakelen van Bonjour wijzigt `gateway.bind` niet; Docker gebruikt nog steeds standaard `OPENCLAW_GATEWAY_BIND=lan`, zodat de gepubliceerde hostpoort werkt.
- Het uitschakelen van Bonjour schakelt wide-area DNS-SD niet uit. Gebruik wide-area-detectie of Tailnet wanneer de Gateway en Node zich niet op hetzelfde LAN bevinden.
- Hergebruik van dezelfde `OPENCLAW_CONFIG_DIR` buiten Docker maakt het beleid voor automatisch uitschakelen in containers niet permanent.
- Stel `OPENCLAW_DISABLE_BONJOUR=0` alleen in voor hostnetwerken, macvlan of een ander netwerk waarvan bekend is dat mDNS-multicast wordt doorgelaten; stel dit in op `1` om uitschakeling af te dwingen.

## Problemen met uitgeschakelde Bonjour oplossen

Als een Node de Gateway na het instellen van Docker niet langer automatisch ontdekt:

1. Controleer of de Gateway in automatische, geforceerd ingeschakelde of geforceerd uitgeschakelde modus wordt uitgevoerd:

   ```bash
   docker compose config | grep OPENCLAW_DISABLE_BONJOUR
   ```

2. Controleer of de Gateway zelf via de gepubliceerde poort bereikbaar is:

   ```bash
   curl -fsS http://127.0.0.1:18789/healthz
   ```

3. Gebruik een rechtstreeks doel wanneer Bonjour is uitgeschakeld:
   - Control UI of lokale hulpmiddelen: `http://127.0.0.1:18789`
   - LAN-clients: `http://<gateway-host>:18789`
   - Clients op andere netwerken: Tailnet MagicDNS, Tailnet-IP, SSH-tunnel of wide-area DNS-SD

4. Als je de Bonjour-Plugin bewust in Docker hebt ingeschakeld en advertenties hebt afgedwongen met `OPENCLAW_DISABLE_BONJOUR=0`, test je multicast vanaf de host:

   ```bash
   dns-sd -B _openclaw-gw._tcp local.
   ```

   Als het browseresultaat leeg is of Gateway-logboeken herhaalde ciao-peilingsfouten tonen, herstel je `OPENCLAW_DISABLE_BONJOUR=1` en gebruik je een rechtstreekse route of Tailnet-route.

## Veelvoorkomende foutmodi

- **Bonjour werkt niet over verschillende netwerken heen**: gebruik Tailnet of SSH.
- **Multicast geblokkeerd**: sommige wifi-netwerken schakelen mDNS uit.
- **Advertiser blijft hangen bij detecteren/aankondigen**: hosts met geblokkeerde multicast, containerbridges, WSL of wisselende interfaces kunnen de responder in een niet-aangekondigde toestand achterlaten. De Gateway blijft beschikbaar via directe, SSH-, Tailnet- of wide-area DNS-SD-routes; schakel LAN Bonjour uit met `discovery.mdns.mode: "off"` of `OPENCLAW_DISABLE_BONJOUR=1` wanneer multicast niet beschikbaar is.
- **Docker-brugnetwerken**: Bonjour wordt automatisch uitgeschakeld in gedetecteerde containers. Stel `OPENCLAW_DISABLE_BONJOUR=0` alleen in voor een host-, macvlan- of ander mDNS-compatibel netwerk.
- **Slaapstand/wisselende interfaces**: macOS kan mDNS-resultaten tijdelijk verliezen; probeer het opnieuw.
- **Bladeren werkt, maar omzetten mislukt**: houd machinenamen eenvoudig (vermijd emoji's en leestekens) en herstart vervolgens de Gateway. De naam van de service-instantie wordt afgeleid van de hostnaam, waardoor te complexe namen sommige resolvers in de war kunnen brengen.

## Geëscapete instantienamen (`\032`)

Bonjour/DNS-SD esc aped bytes in namen van service-instanties vaak als decimale `\DDD`-reeksen (spaties worden `\032`). Dit is normaal op protocolniveau; gebruikersinterfaces moeten deze voor weergave decoderen (iOS gebruikt `BonjourEscapes.decode`).

## Inschakelen / uitschakelen / configuratie

| Instelling                                              | Effect                                                                            |
| ---------------------------------------------------- | --------------------------------------------------------------------------------- |
| `openclaw plugins enable bonjour`                    | Schakelt de meegeleverde Plugin voor LAN-detectie in op hosts waar deze niet standaard is ingeschakeld. |
| `openclaw plugins disable bonjour`                   | Schakelt LAN-multicastadvertenties uit door de meegeleverde Plugin uit te schakelen.               |
| `OPENCLAW_DISABLE_BONJOUR=1` (of `true`/`yes`/`on`)  | Schakelt LAN-multicastadvertenties uit zonder de Plugin-configuratie te wijzigen.                |
| `OPENCLAW_DISABLE_BONJOUR=0` (of `false`/`no`/`off`) | Dwingt LAN-multicastadvertenties af, ook binnen gedetecteerde containers.        |
| `discovery.mdns.mode`                                | `off` \| `minimal` (standaard) \| `full` — zie de bovenstaande modi.                         |
| `gateway.bind`                                       | Regelt de bindmodus van de Gateway in `~/.openclaw/openclaw.json`.                    |
| `OPENCLAW_SSH_PORT`                                  | Overschrijft de SSH-poort wanneer `sshPort` wordt geadverteerd (volledige modus).                  |
| `OPENCLAW_TAILNET_DNS`                               | Publiceert een MagicDNS-hint in TXT wanneer de volledige mDNS-modus is ingeschakeld.                  |
| `OPENCLAW_CLI_PATH`                                  | Overschrijft het geadverteerde CLI-pad (volledige modus).                                    |

macOS-hosts starten de meegeleverde Plugin voor LAN-detectie standaard automatisch. Wanneer de Bonjour-Plugin is ingeschakeld en `OPENCLAW_DISABLE_BONJOUR` niet is ingesteld, adverteert Bonjour op normale hosts en schakelt het zichzelf automatisch uit binnen gedetecteerde containers (Docker, Fly.io-machines en gangbare containerruntimes).

## Gerelateerde documentatie

- Detectiebeleid en transportselectie: [Detectie](/nl/gateway/discovery)
- Node-koppeling en goedkeuringen: [Gateway-koppeling](/nl/gateway/pairing)
