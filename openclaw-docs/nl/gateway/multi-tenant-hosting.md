---
doc-schema-version: 1
read_when:
    - Je host OpenClaw voor meerdere gebruikers of organisaties
    - Je moet een isolatiegrens voor tenantworkloads kiezen
summary: Host meerdere vertrouwensdomeinen voor tenants als één geïsoleerde OpenClaw Gateway-cel per tenant
title: Hosting voor meerdere tenants
x-i18n:
    generated_at: "2026-07-27T05:11:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 383d32331b45d40db6fb4ff8242dd9a3cf8898a3ccab19f0372cd06bbd83fc05
    source_path: gateway/multi-tenant-hosting.md
    workflow: 16
---

# Hosting voor meerdere tenants

Het standaardbeveiligingsmodel van OpenClaw is één vertrouwde operatorgrens per Gateway, niet de isolatie van vijandige tenants binnen één gedeelde Gateway. Het hosten van gebruikers of organisaties die geen vertrouwensgrens delen, betekent daarom dat voor elke tenant een afzonderlijke, volledige OpenClaw-instantie moet worden uitgevoerd.

`openclaw fleet` noemt elke geïsoleerde instantie een **cel**. Een cel is een volledige Gateway in een geharde container met een eigen status, referenties, werkruimte, kanaalaccounts, token en hostpoort die alleen via loopback bereikbaar is.

Fleet is **experimenteel**: de opdrachten, vlaggen en het containerprofiel kunnen tussen releases zonder uitfaseringsperiode veranderen.

Fleet is getest op Linux- en macOS-hosts. Windows-hosts zijn momenteel niet getest.

## Waarom elke tenant een cel nodig heeft

Een geverifieerde operator binnen één Gateway heeft een vertrouwde rol in het besturingsvlak. Sessie-ID's bepalen de routering; ze autoriseren niet de ene tenant ten opzichte van de andere. Agent-sandboxing kan de gevolgen van niet-vertrouwde inhoud en de uitvoering van hulpmiddelen beperken, maar maakt van één gedeelde Gateway geen autorisatiegrens tussen tenants.

Gebruik één cel per tenant, zodat elk vertrouwensdomein een afzonderlijk Gateway-proces, een afzonderlijke container, een afzonderlijke persistente statusboom en afzonderlijke Gateway-referenties heeft. Dit volgt het [Gateway-beveiligingsmodel](/nl/gateway/security): plaats onderling niet-vertrouwde gebruikers niet samen in één OpenClaw-proces of onder één OS-gebruiker.

## Architectuur

De Fleet-CLI is een lifecycle-supervisor aan de hostzijde. Deze registreert cellen in de OpenClaw-statusdatabase en vraagt een lokale Docker- of Podman-runtime om hun containers te maken, inspecteren, starten, stoppen, vervangen en verwijderen. Externe runtime-eindpunten worden niet ondersteund, omdat de bindpaden en loopback-URL's van Fleet bij de lokale host horen. Fleet fungeert niet als proxy voor berichten van tenants en voegt geen gedeeld gegevenspad op applicatieniveau tussen cellen toe.

Elke cel voert de officiële `ghcr.io/openclaw/openclaw`-image uit op een eigen, door de gebruiker gedefinieerd bridge-netwerk. Afzonderlijke bridges voorkomen rechtstreeks container-IP-verkeer tussen cellen, terwijl uitgaande NAT-toegang voor providers en kanalen behouden blijft. Uitgaand verkeer is standaard onbeperkt. Podman-cellen kunnen `--network internal` gebruiken om uitgaand verkeer te blokkeren en tegelijk de gepubliceerde loopback-Gateway-poort te behouden. Interne Docker-netwerken maken die gepubliceerde poort onbruikbaar, dus Fleet weigert deze combinatie; dwing het beleid voor uitgaand Docker-verkeer in plaats daarvan af met firewallregels op de host, zoals de `DOCKER-USER`-keten. De Gateway van de cel luistert in de container op poort `18789`, terwijl de runtime deze op de host alleen naar `127.0.0.1:<allocated-port>` publiceert. Wanneer externe toegang nodig is, kan een operator een goedgekeurde reverse proxy, SSH-tunnel of tailnet vóór dat loopback-eindpunt plaatsen.

Persistente Gateway-status is afkomstig van `<state-dir>/fleet/cells/<tenant>/` en wordt gekoppeld aan `/home/node/.openclaw`. Versleutelingssleutels voor verificatieprofielen zijn afkomstig van het afzonderlijke hostpad `<state-dir>/fleet/auth-profile-secrets/<tenant>/` en worden gekoppeld aan `/home/node/.config/openclaw`, overeenkomstig de officiële [Docker-indeling voor persistentie](/nl/install/docker#storage-and-persistence). De sleutel bevindt zich niet onder de gewone statuskoppeling. Kanaalaccounts per tenant eindigen in de cel die ze bezit; Fleet biedt geen gedeeld kanaalaccount of router voor inkomende berichten.

De officiële image gebruikt standaard de niet-rootgebruiker `node` met UID 1000. Fleet gebruikt gebruikerskoppelingen die compatibel zijn met de host, zodat persoonlijke bind mounts beschrijfbaar blijven: Podman gebruikt `keep-id`, Docker met rootrechten gebruikt de aanroepende niet-rootidentiteit en rootloze Docker koppelt container-root aan de gebruiker zonder verhoogde rechten waaronder de daemon draait. Docker en Podman passen een persoonlijke `:Z`-herlabeling toe wanneer SELinux op de host actief is. Het containerprofiel vermijdt geprivilegieerde hostfuncties en is geschikt voor rootloos gebruik, maar rootloze werking is een keuze en vereiste van de runtime op de host, niet iets wat Fleet automatisch inschakelt.

## Vertrouwensgrens

Multi-tenancy beschermt tenants tegen elkaar. De Fleet-operator en de host worden door elke tenant vertrouwd. Weerstand tegen een gecompromitteerde host is geen doelstelling.

Dit betekent dat een hostbeheerder de containerconfiguratie en -omgeving kan inspecteren, gekoppelde celgegevens kan lezen, images kan vervangen of containers kan binnengaan. Gateway-tokens en waarden die met `--env` worden doorgegeven, zijn via inspectie door Docker of Podman zichtbaar voor een beheerder. Gebruik daarom passende hostcontroles, beleid voor beheerderstoegang, monitoring, back-ups en een goedgekeurde geheimenbeheerder.

De basisconfiguratie voorkomt onbedoelde netwerkblootstelling via jokertekens en verwijdert veelgebruikte mechanismen voor privilegeverhoging in containers, maar maakt een niet-vertrouwde host niet veilig.

## Isolatieladder

Kies de grens die past bij de tenants die je host:

1. **Geharde containerbasis.** Fleet verwijdert alle Linux-capabilities, schakelt `no-new-privileges` in, past limieten toe voor PID's, geheugen, CPU en optioneel de beschrijfbare schijflaag, gebruikt afzonderlijke persistente koppelingen en netwerken per cel en publiceert uitsluitend naar de loopback van de host. Bij bridge-netwerken blijft uitgaand verkeer onbeperkt; gebruik Podman `--network internal` of firewallbeleid op de Docker-host wanneer een cel geen uitgaande verbindingen mag initiëren. Dit is het standaardprofiel voor tenants die de operator en host vertrouwen.
2. **Sterkere container- of VM-isolatie.** Configureer Docker of Podman voor workloads met een hoger risico om een sterkere OCI-isolatieruntime te gebruiken, zoals gVisor of Kata Containers, of plaats cellen in micro-VM's. Dit is runtime- of infrastructuurconfiguratie; de optie `--runtime docker|podman` van Fleet kiest de container-CLI, niet de OCI-isolatiebackend. Zie de [alternatieve containerruntimes](https://docs.docker.com/engine/daemon/alternative-runtimes/) van Docker en de [handleiding voor de Docker-VM-runtime](/nl/install/docker-vm-runtime).
3. **Afzonderlijke machines voor vijandige tenants.** Plaats vijandige tenants niet samen in één OpenClaw-proces of onder één OS-gebruiker. Gebruik afzonderlijke VM's of fysieke hosts met afzonderlijk runtimebeheer wanneer tenants niet dezelfde hostoperator vertrouwen of een sterkere administratieve grens nodig hebben.

Geen enkele trede van deze ladder verandert het vertrouwensmodel van de OpenClaw-applicatie: één Gateway blijft één vertrouwd operatordomein.

## Snel aan de slag

Maak een cel. De opdracht drukt een gegenereerd Gateway-token slechts eenmaal af, dus sla het onmiddellijk op:

```bash
openclaw fleet create acme
```

Open de gemelde `http://127.0.0.1:<port>`-URL op de Fleet-host, verifieer je met het token van die tenant en configureer providerreferenties en kanaalaccounts binnen de cel.

Controleer de containerstatus en bereikbaarheid van de Gateway:

```bash
openclaw fleet status acme
```

Voer een upgrade uit met behoud van de hostpoort, gekoppelde gegevens, het resourceprofiel, de door de gebruiker opgegeven omgeving en het Gateway-token:

```bash
openclaw fleet upgrade acme
```

Verwijder de container en registerrij, maar behoud de tenantgegevens:

```bash
openclaw fleet rm acme --force
```

Voeg `--purge-data` toe om ook de persistente tenantgegevens te verwijderen. Opschonen vereist `--force`, is onomkeerbaar en voert een insluitingscontrole op het herleide pad uit voordat iets wordt verwijderd:

```bash
openclaw fleet rm acme --purge-data --force
```

Zie de [CLI-referentie voor `openclaw fleet`](/nl/cli/fleet) voor elke opdracht en optie.

## Huidige reikwijdte

Fleet biedt de volgende mogelijkheden niet:

- Gedeelde kanaalaccounts of een gedeelde router voor inkomend verkeer
- Afgeslankte hostprocessen per tenant in plaats van volledige OpenClaw-instanties
- Externe celhosts die door één supervisor worden beheerd
- Een selfserviceportal voor tenants, een factureringsvlak of een gebruikersinterface voor gedelegeerd beheer

Deze mogelijkheden vereisen expliciete contracten voor identiteit, routering, autorisatie en foutdomeinen. Benader ze niet door één Gateway of de bijbehorende referenties tussen tenants te delen. Fleet is een lifecycle-supervisor voor één host; fleets met meerdere machines en identiteitsgestuurd beheer vereisen een afzonderlijke besturingslaag.

## Gerelateerd

- [`openclaw fleet`](/nl/cli/fleet)
- [Gateway-beveiliging](/nl/gateway/security)
- [Meerdere gateways](/nl/gateway/multiple-gateways)
- [Docker](/nl/install/docker)
- [Podman](/nl/install/podman)
