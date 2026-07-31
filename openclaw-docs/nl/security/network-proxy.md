---
read_when:
    - Je wilt gelaagde beveiliging tegen SSRF- en DNS-rebindingaanvallen
    - Een externe forward proxy configureren voor OpenClaw-runtimeverkeer
summary: OpenClaw-runtimeverkeer via HTTP en WebSocket routeren door een door de beheerder beheerde filterproxy
title: Netwerkproxy
x-i18n:
    generated_at: "2026-07-27T05:50:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e948189d691e2cfe32e911e24071fd77157397b510d606423ef738c2565071b5
    source_path: security/network-proxy.md
    workflow: 16
---

OpenClaw kan HTTP- en WebSocket-verkeer tijdens runtime routeren via een door de beheerder beheerde forward proxy. Dit is optionele gelaagde beveiliging: centrale controle over uitgaand verkeer, sterkere SSRF-bescherming en controleerbaarheid van bestemmingen aan de netwerkgrens. Omdat de proxy de bestemming beoordeelt wanneer de verbinding tot stand wordt gebracht, na DNS-resolutie en onmiddellijk voordat de upstreamverbinding wordt geopend, verkleint dit ook het tijdsvenster waarvan een DNS-rebindingaanval afhankelijk is tussen een eerdere DNS-controle op applicatieniveau en de daadwerkelijke uitgaande verbinding. Eén proxybeleid biedt beheerders bovendien één plek om bestemmingsregels, netwerksegmentatie, snelheidslimieten of toelatingslijsten voor uitgaand verkeer af te dwingen zonder OpenClaw opnieuw te bouwen.

OpenClaw levert, downloadt, start, configureert of certificeert geen proxy. Je gebruikt de proxytechnologie die bij jouw omgeving past; OpenClaw routeert zijn eigen HTTP- en WebSocket-clients erdoorheen.

## Configuratie

```yaml
proxy:
  proxyUrl: http://127.0.0.1:3128
```

Je kunt de URL ook via de omgeving instellen:

```bash
OPENCLAW_PROXY_URL=http://127.0.0.1:3128 openclaw gateway run
```

`proxy.proxyUrl` heeft voorrang op `OPENCLAW_PROXY_URL`. Een geconfigureerde URL activeert beheerde proxyrouting; door beide URL's te verwijderen, schakel je deze uit.

| Sleutel              | Type                                 | Standaard      | Opmerkingen                                                                                                                          |
| -------------------- | ------------------------------------ | -------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `proxy.proxyUrl`     | string                               | niet ingesteld | URL van de forward proxy voor `http://` of `https://`. In de URL opgenomen referenties worden als gevoelig behandeld en in momentopnamen/logboeken geredigeerd. |
| `proxy.tls.caFile`   | string                               | niet ingesteld | CA-bundel voor het verifiëren van een `https://`-proxyeindpunt dat door een privé-CA is ondertekend.                               |
| `proxy.loopbackMode` | `gateway-only` \| `proxy` \| `block` | `gateway-only` | Regelt het omzeilingsgedrag voor loopback; zie hieronder.                                                                             |

Sla voor beheerde Gateway-services de URL op in de configuratie, zodat deze een herinstallatie overleeft, in plaats van afhankelijk te zijn van een omgevingsvariabele voor een voorgrondproces:

```bash
openclaw config set proxy.proxyUrl http://127.0.0.1:3128
openclaw gateway install --force
openclaw gateway start
```

De terugval op de omgevingsvariabele `OPENCLAW_PROXY_URL` is het meest geschikt voor voorgronduitvoeringen. Om deze met een geïnstalleerde service te gebruiken, plaats je deze in de permanente omgeving van de service (`$OPENCLAW_STATE_DIR/.env`, standaard `~/.openclaw/.env`) en installeer je de service vervolgens opnieuw, zodat launchd/systemd/Scheduled Tasks deze overneemt.

### HTTPS-proxyeindpunt met een privé-CA

```yaml
proxy:
  proxyUrl: https://proxy.corp.example:8443
  tls:
    caFile: /etc/openclaw/proxy-ca.pem
```

`proxy.tls.caFile` verifieert het eigen TLS-certificaat van het proxyeindpunt. Het is geen vertrouwensinstelling voor MITM-verkeer naar bestemmingen, geen clientcertificaat en geen vervanging voor het bestemmingsbeleid van de proxy. Gebruik `NODE_EXTRA_CA_CERTS` alleen wanneer het volledige Node-proces vanaf het opstarten een aanvullende CA moet vertrouwen (bijvoorbeeld een TLS-inspectiesysteem van een organisatie dat elk HTTPS-bestemmingscertificaat opnieuw ondertekent) — die variabele geldt voor het hele proces en moet worden ingesteld voordat Node start. OpenClaw kan deze daarom niet tijdens de uitvoering toepassen zoals bij `proxy.tls.caFile`. Geef voor het vertrouwen van HTTPS-proxyeindpunten de voorkeur aan `proxy.tls.caFile`: dit is beperkt tot beheerde proxyrouting in plaats van het hele proces.

```bash
openclaw config set proxy.proxyUrl https://proxy.corp.example:8443
openclaw config set proxy.tls.caFile /etc/openclaw/proxy-ca.pem
openclaw gateway run
```

## Hoe routering werkt

Met een geldige proxy-URL routeren beschermde runtimeprocessen (`openclaw gateway run`, `openclaw node run`, `openclaw agent --local`) normaal uitgaand HTTP- en WebSocket-verkeer via de proxy:

```text
OpenClaw-proces
  fetch, node:http, node:https, WebSocket-clients  -> proxy van beheerder -> bestemming
```

Intern installeert OpenClaw [Proxyline](https://github.com/openclaw/proxyline) als routeringsruntime op procesniveau. Deze ondersteunt `fetch`, clients op basis van undici, `node:http`/`node:https`, veelgebruikte WebSocket-clients en door helpers gemaakte `CONNECT`-tunnels. Ook vervangt deze door aanroepers geleverde Node-HTTP-agents, zodat expliciete agents (waaronder `axios`, `got`, `node-fetch` en vergelijkbare clients op basis van Node-agents) de proxy niet ongemerkt kunnen omzeilen.

Het schema van de proxy-URL beschrijft de verbinding van OpenClaw naar de proxy, niet naar de eindbestemming:

- `http://proxy.example:3128` — onbeveiligd TCP naar de proxy; OpenClaw verzendt HTTP-proxyverzoeken, waaronder `CONNECT` voor HTTPS-bestemmingen.
- `https://proxy.example:8443` — OpenClaw opent TLS naar de proxy zelf (waarbij het certificaat van de proxy wordt geverifieerd) en verzendt vervolgens HTTP-proxyverzoeken binnen die sessie.

TLS voor de bestemming staat los van TLS voor het proxyeindpunt: voor een HTTPS-bestemming vraagt OpenClaw de proxy altijd om een `CONNECT`-tunnel en start het TLS naar de bestemming via die tunnel.

Terwijl de proxy actief is, wist OpenClaw `no_proxy`/`NO_PROXY`. Deze omzeilingslijsten zijn gebaseerd op bestemmingen; als `localhost` of `127.0.0.1` daarin blijven staan, kunnen SSRF-doelen de proxy volledig omzeilen. Bij het afsluiten herstelt OpenClaw de eerdere proxyomgeving en stelt het de gecachte routeringsstatus opnieuw in.

Sommige plugins beheren een aangepast transport waarvoor eigen proxyconfiguratie nodig is, zelfs wanneer routering op procesniveau actief is. De Bot API-client van Telegram gebruikt een eigen HTTP/1-dispatcher van undici en respecteert afzonderlijk de proxyomgeving van het proces plus de terugvaloptie `OPENCLAW_PROXY_URL`.

### Loopbackmodus van de Gateway

Lokale clients van het besturingsvlak van de Gateway maken normaal verbinding met een loopback-WebSocket, zoals `ws://127.0.0.1:18789`. `proxy.loopbackMode` bepaalt of dat verkeer de beheerde proxy omzeilt:

```yaml
proxy:
  proxyUrl: http://127.0.0.1:3128
  loopbackMode: gateway-only # gateway-only, proxy, or block
```

Een geconfigureerde `proxyUrl` of `OPENCLAW_PROXY_URL` schakelt beheerde routering in. Stel
`proxy.enabled: false` alleen in als geavanceerde afmeldoptie waarmee de URL opgeslagen blijft
zonder deze te activeren.

| Modus                    | Gedrag                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway-only` (standaard) | OpenClaw registreert de actieve loopback-autoriteit van de Gateway als uitzondering voor een rechtstreekse verbinding, zodat lokaal WebSocket-verkeer van de Gateway zonder de proxy verbinding maakt. Aangepaste loopbackpoorten werken omdat de uitzondering exact op de geconfigureerde host/poort is gericht. De meegeleverde browserplugin registreert hetzelfde soort uitzondering voor de exacte lokale CDP-gereedheids- en DevTools-WebSocket-URL's van door OpenClaw gestarte beheerde browsers; de meegeleverde Ollama-provider voor geheugen-embeddings heeft een beperkter, beveiligd rechtstreeks pad voor de exacte geconfigureerde host-lokale loopbackoorsprong voor embeddings. |
| `proxy`                  | Er worden geen loopbackuitzonderingen geregistreerd; loopbackverkeer van de Gateway en Ollama loopt via de proxy. Een externe proxy moet terug kunnen routeren naar de loopbackservice op de OpenClaw-host (bijvoorbeeld via een bereikbare hostnaam, IP-adres of tunnel) — een standaard externe proxy herleidt `127.0.0.1`/`localhost` ten opzichte van zichzelf, niet ten opzichte van de OpenClaw-host.                                                                                                                                      |
| `block`                  | OpenClaw weigert loopbackverbindingen met het besturingsvlak van de Gateway en beveiligde loopbackverbindingen voor Ollama-embeddings voordat een socket wordt geopend.                                                                                                                                                                                                                                                                                                                                                                                               |

Omzeiling voor het besturingsvlak van de Gateway is beperkt tot `localhost` en letterlijke loopback-IP-URL's — gebruik `ws://127.0.0.1:18789`, `ws://[::1]:18789` of `ws://localhost:18789`. Andere hostnamen worden als normaal verkeer gerouteerd.

### Containers

Voor `openclaw --container ...`-opdrachten geeft OpenClaw `OPENCLAW_PROXY_URL` door aan de op de container gerichte onderliggende CLI wanneer deze is ingesteld. De URL moet vanuit de container bereikbaar zijn — `127.0.0.1` verwijst daar naar de container zelf, niet naar de host. OpenClaw weigert loopback-proxy-URL's voor op containers gerichte opdrachten, tenzij je `OPENCLAW_CONTAINER_ALLOW_LOOPBACK_PROXY_URL=1` instelt om die controle expliciet te negeren.

## Verwante proxytermen

- `proxy.enabled` / `proxy.proxyUrl` — uitgaande routering via een forward proxy voor runtimeverkeer. Deze pagina.
- `gateway.auth.mode: "trusted-proxy"` — binnenkomende identiteitsbewuste authenticatie via een reverse proxy voor toegang tot de Gateway. Zie [Authenticatie via vertrouwde proxy](/nl/gateway/trusted-proxy-auth).
- `openclaw proxy` — lokale debugproxy en inspectietool voor vastleggingen voor ontwikkeling en ondersteuning. Zie [openclaw proxy](/nl/cli/proxy).
- `tools.web.fetch.useTrustedEnvProxy` — opt-in voor `web_fetch` om een door de beheerder gecontroleerde HTTP(S)-omgevingsproxy DNS te laten herleiden, terwijl standaard strikte DNS-pinning en hostnaambeleid behouden blijven. Zie [Web ophalen](/nl/tools/web-fetch#trusted-env-proxy).
- Kanaal- of providerspecifieke proxy-instellingen — eigenaarspecifieke overschrijvingen voor één transport. Geef voor centrale controle over uitgaand verkeer in de hele runtime de voorkeur aan de beheerde netwerkproxy.

## De proxy valideren

Het bestemmingsbeleid van de proxy vormt de daadwerkelijke beveiligingsgrens; OpenClaw kan niet controleren of jouw proxy de juiste doelen blokkeert. Configureer deze om:

- Alleen te binden aan loopback of een vertrouwde privé-interface die uitsluitend bereikbaar is voor het OpenClaw-proces/de host/de container/het serviceaccount.
- Bestemmingen zelf te herleiden en deze na DNS-resolutie op IP-adres te blokkeren wanneer de verbinding tot stand wordt gebracht, zowel voor onbeveiligd HTTP als voor HTTPS-`CONNECT`-tunnels.
- Omzeilingen op basis van bestemmingen te weigeren voor loopback-, privé-, link-local-, metadata-, multicast-, gereserveerde en documentatiebereiken.
- Toelatingslijsten voor hostnamen te vermijden, tenzij je het DNS-resolutiepad volledig vertrouwt.
- Bestemming, beslissing, status en reden te loggen — nooit aanvraaginhoud, autorisatieheaders, cookies of andere geheimen.
- Het beleid onder versiebeheer te houden en wijzigingen als beveiligingsgevoelig te beoordelen.

Valideer vanaf dezelfde host/container/hetzelfde serviceaccount waarop OpenClaw wordt uitgevoerd:

```bash
openclaw proxy validate --proxy-url http://127.0.0.1:3128
```

Met een HTTPS-proxyeindpunt met een privé-CA:

```bash
openclaw proxy validate --proxy-url https://proxy.corp.example:8443 --proxy-ca-file /etc/openclaw/proxy-ca.pem
```

| Vlag                     | Doel                                                                 |
| ------------------------ | -------------------------------------------------------------------- |
| `--proxy-url <url>`      | Deze URL valideren in plaats van config/env op te lossen.            |
| `--proxy-ca-file <path>` | CA-bundel voor een HTTPS-proxyeindpunt.                              |
| `--allowed-url <url>`    | Bestemming die naar verwachting bereikbaar is (herhaalbaar).         |
| `--denied-url <url>`     | Bestemming die naar verwachting wordt geblokkeerd (herhaalbaar).      |
| `--apns-reachable`       | Ook controleren of de proxy een directe APNs HTTP/2-probe vanuit de sandbox kan tunnelen. |
| `--apns-authority <url>` | De APNs-authority overschrijven die met `--apns-reachable` wordt getest. |
| `--timeout-ms <ms>`      | Time-out per verzoek.                                                 |
| `--json`                 | Machineleesbare uitvoer.                                              |

Als er geen waarde voor config, omgeving of `--proxy-url` beschikbaar is, meldt de opdracht een configuratieprobleem; geef `--proxy-url` door voor een eenmalige preflight voordat je de configuratie wijzigt.

Zonder `--allowed-url`/`--denied-url` zijn de standaardcontroles: `https://example.com/` moet slagen en een tijdelijke loopback-canaryserver die de proxy niet mag bereiken, moet worden geblokkeerd. De loopbackcontrole slaagt bij een transportfout, of bij een niet-2xx-antwoord zonder het token per uitvoering van de canary; de controle mislukt bij een 2xx-antwoord zonder het token (een onverwacht succes van iets anders dan de canary) en vooral bij elk antwoord met het overeenkomende token, omdat dit bewijst dat de proxy daadwerkelijk een loopbackbestemming heeft doorgestuurd die had moeten worden geweigerd. Aangepaste `--denied-url`-doelen hebben geen dergelijk canarytoken en werken daarom fail-closed: elk HTTP-antwoord geldt als bereikbaar (mislukt) en een transportfout wordt gemeld als onbeslist in plaats van bewezen geblokkeerd, omdat OpenClaw niet kan bevestigen of je proxy een bereikbare origin heeft geweigerd of dat er iets anders is misgegaan. `--apns-reachable` verzendt opzettelijk een ongeldig providertoken, zodat een `403 InvalidProviderToken`-antwoord geldt als bewijs dat de tunnel Apple heeft bereikt. De opdracht sluit af met `1` bij elke validatiefout; inloggegevens in de proxy-URL worden zowel in tekst- als JSON-uitvoer geredigeerd.

```json
{
  "ok": true,
  "config": {
    "enabled": true,
    "proxyUrl": "http://127.0.0.1:3128/",
    "source": "override",
    "errors": []
  },
  "checks": [
    { "kind": "allowed", "url": "https://example.com/", "ok": true, "status": 200 },
    { "kind": "apns", "url": "https://api.sandbox.push.apple.com", "ok": true, "status": 403 }
  ]
}
```

Handmatige `curl`-controle (het openbare verzoek moet slagen; de loopback- en metadataverzoeken moeten door de proxy zelf worden geblokkeerd — alleen `curl` kan een weigering door de proxy niet onderscheiden van een onbereikbare origin zoals de ingebouwde canary van `openclaw proxy validate` dat kan):

```bash
curl -x http://127.0.0.1:3128 https://example.com/
curl -x http://127.0.0.1:3128 http://127.0.0.1/
curl -x http://127.0.0.1:3128 http://169.254.169.254/
```

## Aanbevolen te blokkeren bestemmingen

Beginlijst met te weigeren bestemmingen voor elke forward proxy, firewall of egressbeleid. De eigen SSRF-classificator van OpenClaw bevindt zich in `src/infra/net/ssrf.ts` en `packages/net-policy/src/ip.ts` (`BLOCKED_HOSTNAMES`, `BLOCKED_IPV4_SPECIAL_USE_RANGES`, `BLOCKED_IPV6_SPECIAL_USE_RANGES`, het benchmarkprefix uit RFC 2544 en de verwerking van ingebedde IPv4 voor NAT64/6to4/Teredo/ISATAP/IPv4-mapped-vormen) — nuttige referenties, maar OpenClaw exporteert of handhaaft deze regels niet in je externe proxy.

| Bereik of host                                                                        | Reden om te blokkeren                            |
| ------------------------------------------------------------------------------------ | ------------------------------------------------- |
| `127.0.0.0/8`, `localhost`, `localhost.localdomain`                                  | IPv4-loopback                                     |
| `::1/128`                                                                            | IPv6-loopback                                     |
| `0.0.0.0/8`, `::/128`                                                                | Niet-gespecificeerde adressen/adressen van dit netwerk |
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`                                      | Privénetwerken volgens RFC 1918                   |
| `169.254.0.0/16`, `fe80::/10`                                                        | Link-local, inclusief veelgebruikte paden voor cloudmetadata |
| `169.254.169.254`, `metadata.google.internal`                                        | Cloudmetadataservices                             |
| `100.64.0.0/10`                                                                      | Gedeelde adresruimte voor carrier-grade NAT       |
| `198.18.0.0/15`, `2001:2::/48`                                                       | Benchmarkbereiken                                 |
| `192.0.0.0/24`, `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`, `2001:db8::/32` | Bereiken voor speciaal gebruik en documentatie    |
| `224.0.0.0/4`, `ff00::/8`                                                            | Multicast                                         |
| `240.0.0.0/4`                                                                        | Gereserveerde IPv4-adressen                       |
| `fc00::/7`, `fec0::/10`                                                              | Lokale/privébereiken voor IPv6                    |
| `100::/64`, `2001:20::/28`                                                           | IPv6-bereiken voor discard en ORCHIDv2            |
| `64:ff9b::/96`, `64:ff9b:1::/48`                                                     | NAT64-prefixen met ingebedde IPv4                 |
| `2002::/16`, `2001::/32`                                                             | 6to4 en Teredo met ingebedde IPv4                 |
| `::/96`, `::ffff:0:0/96`                                                             | IPv4-compatibele en IPv4-mapped IPv6-adressen     |

Voeg eventuele aanvullende metadatahosts of gereserveerde bereiken toe die door je cloudprovider of netwerkplatform worden gedocumenteerd.

## Beperkingen

| Oppervlak                                                    | Status van beheerde proxy                                                                                                                                |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fetch`, `node:http`, `node:https`, gangbare WebSocket-clients | Worden via beheerde proxyhooks gerouteerd wanneer deze zijn geconfigureerd.                                                                               |
| Directe APNs HTTP/2                                          | Wordt via de beheerde APNs-helper `CONNECT` gerouteerd.                                                                                         |
| Loopback van het Gateway-besturingsvlak                      | Alleen direct voor de exact geconfigureerde lokale loopback-Gateway-URL.                                                                                  |
| Upstreamforwarding via de debugproxy                         | Uitgeschakeld terwijl de beheerde proxymodus actief is, tenzij dit expliciet is ingeschakeld voor lokale diagnostiek.                                    |
| IRC                                                          | Onbewerkte TCP/TLS; niet geproxied door de beheerde HTTP-proxymodus. Stel `channels.irc.enabled: false` in als je implementatie vereist dat alle egress via de forward proxy verloopt. |
| Andere clientaanroepen via onbewerkte `net`, `tls` of `http2` | Moeten vóór opname worden geclassificeerd door de bewaking voor onbewerkte sockets.                                                                       |

- Dit biedt dekking op procesniveau voor JavaScript HTTP/WebSocket-clients, geen netwerksandbox op besturingssysteemniveau.
- Onbewerkte `net`-, `tls`- en `http2`-sockets, native add-ons en niet-OpenClaw-subprocessen kunnen routering op Node-niveau omzeilen, tenzij ze proxyomgevingsvariabelen overnemen en respecteren. Afgesplitste OpenClaw-subprocess-CLI's nemen de beheerde proxy-URL en de `proxy.loopbackMode`-status over.
- Lokale WebUI's van gebruikers en lokale modelservers vallen niet onder een algemene omzeiling voor het lokale netwerk — voeg ze indien nodig toe aan de allowlist van het proxybeleid van de beheerder. De uitzondering is het bewaakte directe pad van de gebundelde Ollama-provider voor geheugenembeddings, beperkt tot de exacte host-lokale loopback-origin uit de geconfigureerde `baseUrl`; Ollama-hosts op het LAN, tailnet, privénetwerk en openbare netwerk gebruiken nog steeds de beheerde proxy.
- De directe upstreamforwarding van de lokale debugproxy (voor proxyverzoeken en `CONNECT`-tunnels) is standaard uitgeschakeld terwijl de beheerde proxymodus actief is; schakel deze alleen in voor goedgekeurde lokale diagnostiek.
- OpenClaw inspecteert, test of certificeert je proxybeleid niet. Behandel wijzigingen in het proxybeleid als beveiligingsgevoelige operationele wijzigingen.
