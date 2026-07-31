---
read_when:
    - Bonjour-detectie/-advertenties implementeren of wijzigen
    - Externe verbindingsmodi aanpassen (direct versus SSH)
    - Node-detectie en -koppeling voor externe nodes ontwerpen
summary: Node-detectie en transportmethoden (Bonjour, Tailscale, SSH) voor het vinden van de Gateway
title: Detectie en transporten
x-i18n:
    generated_at: "2026-07-27T05:50:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3a3f1a6a1212ab0bc7021e77c88de059edcb8e09eff90d3e1e59451b9b20876b
    source_path: gateway/discovery.md
    workflow: 16
---

OpenClaw kent twee gerelateerde maar verschillende discoveryproblemen:

1. **Externe bediening door de operator**: de macOS-menubalkapp die een Gateway bestuurt die elders draait.
2. **Node-koppeling**: iOS/Android (en toekomstige Nodes) die een Gateway vinden en veilig koppelen.

Alle netwerkdiscovery/-advertenties vinden plaats in de **Node Gateway**
(`openclaw gateway`); clients (Mac-app, iOS) zijn alleen afnemers.

## Termen

- **Gateway**: één langdurig actief proces dat de status beheert (sessies,
  koppeling, Node-register) en kanalen uitvoert. De meeste configuraties gebruiken er één per host;
  geïsoleerde configuraties met meerdere Gateways zijn mogelijk.
- **Gateway-WS (besturingsvlak)**: het WebSocket-eindpunt standaard op `127.0.0.1:18789`;
  bind dit aan het LAN/de tailnet via `gateway.bind`.
- **Direct WS-transport**: een Gateway-WS-eindpunt dat toegankelijk is via het LAN/de tailnet (zonder SSH).
- **SSH-transport (terugvaloptie)**: externe bediening door
  `127.0.0.1:18789` door te sturen via SSH.
- **Verouderde TCP-bridge (verwijderd)**: ouder Node-transport (zie
  [Bridge-protocol](/nl/gateway/bridge-protocol)); wordt niet langer geadverteerd voor
  discovery en maakt niet langer deel uit van huidige builds.

Protocoldetails: [Gateway-protocol](/nl/gateway/protocol),
[Bridge-protocol (verouderd)](/nl/gateway/bridge-protocol).

## Waarom zowel direct als SSH bestaan

- **Direct WS** biedt de beste gebruikerservaring op hetzelfde netwerk en binnen een tailnet: automatische
  LAN-discovery via Bonjour, koppeltokens en ACL's die door de Gateway worden beheerd,
  en geen shelltoegang vereist.
- **SSH** is de universele terugvaloptie: werkt overal waar je SSH-toegang hebt, zelfs
  tussen niet-gerelateerde netwerken, blijft werken bij multicast-/mDNS-problemen en vereist naast SSH
  geen nieuwe inkomende poort.

## Discovery-invoer

### 1) Bonjour / DNS-SD

Multicast-Bonjour werkt op basis van beste inspanning en gaat niet over netwerkgrenzen heen. OpenClaw
ondersteunt ook het doorzoeken van hetzelfde Gateway-baken via een geconfigureerd domein voor
wide-area DNS-SD, zodat discovery zowel `local.` op hetzelfde LAN als een geconfigureerd
unicast-DNS-SD-domein voor discovery tussen netwerken kan omvatten.

De **Gateway** adverteert zijn WS-eindpunt via Bonjour wanneer de gebundelde Plugin
`bonjour` is ingeschakeld; clients zoeken en tonen een lijst om een Gateway te kiezen,
en slaan vervolgens het gekozen eindpunt op.

Probleemoplossing en details over het baken: [Bonjour](/nl/gateway/bonjour).

#### Details van het servicebaken

- Servicetype: `_openclaw-gw._tcp` (baken voor Gateway-transport).
- TXT-sleutels (niet geheim):

  | Sleutel                      | Opmerkingen                                                                                                                                                     |
  | --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `role=gateway`              | Altijd aanwezig.                                                                                                                                                 |
  | `transport=gateway`         | Altijd aanwezig.                                                                                                                                                 |
  | `displayName=<name>`        | Door de operator geconfigureerde weergavenaam.                                                                                                                   |
  | `lanHost=<hostname>.local`  | Alleen LAN-mDNS-adverteerder; wordt niet geschreven door wide-area DNS-SD.                                                                                        |
  | `gatewayPort=18789`         | Gateway-WS- en HTTP-poort.                                                                                                                                       |
  | `gatewayTls=1`              | Alleen wanneer TLS is ingeschakeld.                                                                                                                              |
  | `gatewayTlsSha256=<sha256>` | Alleen wanneer TLS is ingeschakeld en een vingerafdruk beschikbaar is.                                                                                           |
  | `tailnetDns=<magicdns>`     | Optionele aanwijzing; wordt automatisch gedetecteerd wanneer Tailscale beschikbaar is.                                                                           |
  | `sshPort=<port>`            | Alleen aanwezig wanneer `discovery.mdns.mode="full"`; weggelaten (SSH gebruikt standaard `22`) in de standaardmodus `"minimal"`, zowel bij de LAN-adverteerder als bij wide-area DNS-SD. |
  | `cliPath=<path>`            | Dezelfde `discovery.mdns.mode="full"`-voorwaarde als `sshPort`; een aanwijzing voor externe installatie voor het CLI-pad.                                        |

  Een TXT-sleutel `canvasPort` is in het Plugin-discoverycontract gedefinieerd voor een
  toekomstige canvas-hostpoort, maar geen enkel huidig codepad stelt een waarde in, waardoor deze
  momenteel nooit wordt uitgezonden.

Beveiligingsopmerkingen:

- Bonjour-/mDNS-TXT-records zijn **niet geauthenticeerd**. Clients moeten TXT-waarden
  uitsluitend als aanwijzingen voor de gebruikerservaring beschouwen.
- Voor routering (host/poort) moet de voorkeur uitgaan naar het **opgeloste service-eindpunt**
  (SRV + A/AAAA) boven door TXT verstrekte `lanHost`, `tailnetDns` of `gatewayPort`.
- TLS-pinning mag nooit toestaan dat een geadverteerde `gatewayTlsSha256` een
  eerder opgeslagen pin overschrijft.
- iOS-/Android-Nodes moeten expliciete bevestiging vereisen om deze vingerafdruk te vertrouwen
  voordat een nieuwe pin voor het eerst wordt opgeslagen (verificatie buiten het kanaal),
  wanneer de gekozen route beveiligd/op TLS gebaseerd is.

Inschakelen, uitschakelen en overschrijven:

- `openclaw plugins enable bonjour` schakelt LAN-multicastadvertenties in.
- `discovery.mdns.mode` in `openclaw.json` beheert mDNS-uitzendingen:
  `"minimal"` (standaard), `"full"` (voegt `cliPath`/`sshPort` toe aan zowel het
  LAN-baken als elke wide-area DNS-SD-zone), of `"off"` (schakelt mDNS uit).
- `OPENCLAW_DISABLE_BONJOUR=1` schakelt advertenties geforceerd uit; `discovery.mdns.mode="off"`
  schakelt ze onafhankelijk uit. `OPENCLAW_DISABLE_BONJOUR=0` is een expliciete
  aanmelding die de automatische uitschakeling van de Plugin binnen een gedetecteerde container
  (Docker, containerd, Kubernetes, LXC) overschrijft; dit overschrijft
  `discovery.mdns.mode="off"` niet. De gebundelde Plugin `bonjour` start automatisch op
  macOS-hosts (`enabledByDefaultOnPlatforms: ["darwin"]`) en schakelt zichzelf automatisch uit
  binnen gedetecteerde containers; Linux, Windows en andere gecontaineriseerde
  implementaties vereisen expliciet `plugins enable bonjour`.
- `gateway.bind` in `~/.openclaw/openclaw.json` beheert de bindmodus van de Gateway.
- `OPENCLAW_SSH_PORT` overschrijft de geadverteerde SSH-poort (wordt alleen van kracht
  wanneer `discovery.mdns.mode="full"`).
- `OPENCLAW_TAILNET_DNS` publiceert een aanwijzing `tailnetDns` (MagicDNS).
- `OPENCLAW_CLI_PATH` overschrijft het geadverteerde CLI-pad.

### 2) Tailnet (tussen netwerken)

Voor Gateways op verschillende fysieke netwerken biedt Bonjour geen uitkomst. Het
aanbevolen directe doel is een Tailscale-MagicDNS-naam (bij voorkeur) of een
stabiel tailnet-IP-adres.

Als de Gateway detecteert dat deze onder Tailscale draait, publiceert deze
`tailnetDns` als optionele aanwijzing voor clients (ook in wide-area-bakens).
De macOS-app geeft voor Gateway-discovery de voorkeur aan MagicDNS-namen boven onbewerkte
Tailscale-IP-adressen. Dit blijft betrouwbaar wanneer tailnet-IP-adressen veranderen
(herstart van Nodes, nieuwe CGNAT-toewijzing), omdat MagicDNS automatisch naar het huidige IP-adres verwijst.

Voor het koppelen van mobiele Nodes versoepelen discoveryaanwijzingen nooit de transportbeveiliging op
tailnet-/openbare routes:

- iOS/Android vereist nog steeds een beveiligd pad voor de eerste verbinding via het tailnet/openbare netwerk
  (`wss://` of Tailscale Serve/Funnel).
- Een ontdekt onbewerkt tailnet-IP-adres is een routeringsaanwijzing, geen toestemming om
  externe `ws://` zonder versleuteling te gebruiken.
- Directe verbinding via een privé-LAN met `ws://` blijft ondersteund.
- Gebruik voor het eenvoudigste Tailscale-pad op mobiele Nodes Tailscale Serve, zodat
  zowel discovery als configuratie naar hetzelfde beveiligde MagicDNS-eindpunt verwijzen.

### 3) Handmatig / SSH-doel

Wanneer er geen directe route is (of direct is uitgeschakeld), kunnen clients altijd
verbinding maken via SSH door de loopbackpoort van de Gateway door te sturen. Zie
[Externe toegang](/nl/gateway/remote).

## Transportselectie (clientbeleid)

1. Gebruik een gekoppeld direct eindpunt als dit is geconfigureerd en bereikbaar is.
2. Als dat niet het geval is en discovery een Gateway vindt op `local.` of het geconfigureerde wide-area-
   domein, bied dan een keuze met één tik om deze Gateway te gebruiken en sla deze op als het
   directe eindpunt.
3. Als dat niet het geval is en een tailnet-DNS/IP is geconfigureerd, probeer dan direct verbinding te maken. Voor mobiele Nodes via
   tailnet-/openbare routes betekent direct een beveiligd eindpunt, niet externe
   `ws://` zonder versleuteling.
4. Val anders terug op SSH.

## Koppeling en authenticatie (direct transport)

De Gateway is de gezaghebbende bron voor het toelaten van Nodes/clients:

- Koppelverzoeken worden in de Gateway aangemaakt/goedgekeurd/afgewezen (zie
  [Gateway-koppeling](/nl/gateway/pairing)).
- De Gateway dwingt authenticatie af (token/sleutelpaar), evenals bereiken/ACL's (het is geen onbewerkte
  proxy naar elke methode) en frequentielimieten.

## Verantwoordelijkheden per component

- **Gateway**: adverteert discoverybakens, beheert koppelbeslissingen en host
  het WS-eindpunt.
- **macOS-app**: helpt je een Gateway te kiezen, toont koppelverzoeken en gebruikt SSH
  alleen als terugvaloptie.
- **iOS-/Android-Nodes**: doorzoeken voor het gemak Bonjour en maken verbinding met de
  gekoppelde Gateway-WS.

## Gerelateerd

- [Externe toegang](/nl/gateway/remote)
- [Tailscale](/nl/gateway/tailscale)
- [Bonjour-discovery](/nl/gateway/bonjour)
