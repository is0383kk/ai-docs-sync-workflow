---
read_when:
    - Node-koppelingsgoedkeuringen implementeren zonder macOS-UI
    - CLI-flows toevoegen voor het goedkeuren van externe nodes
    - Gateway-protocol uitbreiden met Node-beheer
summary: 'Goedkeuringen voor Node-mogelijkheden: hoe Nodes na het koppelen van apparaten toegang tot opdrachten krijgen'
title: Node-koppeling
x-i18n:
    generated_at: "2026-07-27T05:34:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 25e4016657379573ddb7e9027899afd8b97b16709da6e73ed44d4016b99e715a
    source_path: gateway/pairing.md
    workflow: 16
---

Node-koppeling heeft twee lagen, die beide worden opgeslagen in de record van het gekoppelde apparaat in de
SQLite-statusdatabase van de Gateway:

- **Apparaatkoppeling** (rol `node`) schermt de `connect`-handshake af. Zie
  [Automatische goedkeuring van apparaten via vertrouwde CIDR](#trusted-cidr-device-auto-approval)
  hieronder en [Kanaalkoppeling](/nl/channels/pairing).
- **Goedkeuring van Node-mogelijkheden** (`node.pair.*`) bepaalt welke gedeclareerde
  mogelijkheden/opdrachten een verbonden Node beschikbaar mag stellen. De Gateway is de
  gezaghebbende bron; UI's (macOS-app, Control UI) zijn frontends die openstaande verzoeken
  goedkeuren of afwijzen.

De voormalige zelfstandige opslag voor Node-koppelingen (`nodes/paired.json` met een token per Node,
in januari 2026 uit het verbindingspad verwijderd) is verdwenen: Gateways voegen
eventuele resterende rijen bij het opstarten eenmalig samen met de apparaatrecords en archiveren de
verouderde bestanden met het achtervoegsel `.migrated`. Ondersteuning voor de verouderde TCP-bridge is
verwijderd.

## Hoe goedkeuring van mogelijkheden werkt

1. Een Node maakt verbinding met de Gateway-WS (apparaatkoppeling schermt deze stap af).
2. De Gateway vergelijkt het gedeclareerde oppervlak van mogelijkheden/opdrachten met het
   goedgekeurde oppervlak; nieuwe of uitgebreide oppervlakken slaan een **openstaand verzoek** op in de
   apparaatrecord en zenden `node.pair.requested` uit.
3. Je keurt het verzoek goed of wijst het af (CLI of UI).
4. Tot de goedkeuring blijven Node-opdrachten gefilterd; na goedkeuring wordt het gedeclareerde
   oppervlak beschikbaar, onderworpen aan het normale opdrachtbeleid.

Openstaande verzoeken verlopen automatisch **5 minuten na de laatste
nieuwe poging van de Node** — bij een Node die actief opnieuw verbinding maakt, blijft het ene openstaande verzoek actief
in plaats van dat per poging een nieuw verzoek (en goedkeuringsprompt) wordt gegenereerd.

## CLI-workflow (geschikt voor headless gebruik)

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes status
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name "Living Room iPad"
```

`nodes status` toont gekoppelde/verbonden Nodes en hun mogelijkheden.

## API-oppervlak (Gateway-protocol)

Gebeurtenissen:

- `node.pair.requested` - wordt uitgezonden wanneer een nieuw openstaand verzoek wordt aangemaakt.
- `node.pair.resolved` - wordt uitgezonden wanneer een verzoek wordt goedgekeurd, afgewezen of
  verloopt.

Methoden:

- `node.pair.list` - geeft openstaande en gekoppelde Nodes weer (`operator.pairing`).
- `node.pair.approve` - keurt een openstaand verzoek goed.
- `node.pair.reject` - wijst een openstaand verzoek af.
- `node.pair.remove` - verwijdert een gekoppelde Node. Hierdoor wordt de rol `node` van het apparaat
  in de opslag voor gekoppelde apparaten ingetrokken, wordt tegelijk het goedgekeurde Node-oppervlak verwijderd en
  worden de Node-rolsessies van dat apparaat ongeldig gemaakt en verbroken. Een apparaat met **gemengde rollen**
  (bijvoorbeeld een apparaat dat ook `operator` heeft) behoudt zijn rij en verliest alleen
  de rol `node`; de rij van een apparaat dat alleen een Node is, wordt verwijderd. Autorisatie:
  `operator.pairing` mag Node-rijen van niet-operators verwijderen; een aanroeper met een apparaattoken
  die zijn **eigen** Node-rol op een apparaat met gemengde rollen intrekt, heeft bovendien
  `operator.admin` nodig.
- `node.rename` - wijzigt de voor de operator zichtbare weergavenaam van een gekoppelde Node.

Verwijderd in 2026.7: `node.pair.request` en `node.pair.verify`. Openstaande
verzoeken worden tijdens Node-verbindingen door de Gateway zelf aangemaakt en het
zelfstandige token per Node waarvoor ze dienden, bestaat niet meer; Node-authenticatie gebruikt het
apparaatkoppelingstoken.

Opmerkingen:

- Nieuwe verbindingen met een ongewijzigd oppervlak hergebruiken het openstaande verzoek; herhaalde
  verzoeken vernieuwen de opgeslagen Node-metadata en de meest recente momentopname van gedeclareerde
  opdrachten op de toelatingslijst voor zichtbaarheid door de operator.
- Niveaus van operatorbereiken en controles tijdens goedkeuring worden samengevat in
  [Operatorbereiken](/nl/gateway/operator-scopes).
- `node.pair.approve` gebruikt de gedeclareerde opdrachten van het openstaande verzoek om
  aanvullende goedkeuringsbereiken af te dwingen:
  - verzoek zonder opdrachten: `operator.pairing`
  - gewoon opdrachtverzoek: `operator.pairing` + `operator.write`
  - beheerdergevoelig verzoek met `system.run`, `system.run.prepare`,
    `system.which`, `browser.proxy`, `fs.listDir` of
    `system.execApprovals.get/set`: `operator.pairing` + `operator.admin`

<Warning>
Goedkeuring van Node-koppeling registreert het vertrouwde mogelijkhedenoppervlak. Hiermee wordt het live-opdrachtoppervlak van een Node **niet** per Node vastgezet.

- Live Node-opdrachten zijn afkomstig van wat de Node bij het verbinden declareert, gefilterd door
  het globale Node-opdrachtbeleid van de Gateway (`gateway.nodes.commands.allow` en
  `gateway.nodes.commands.deny`).
- Het toelatings- en vraagbeleid van `system.run` per Node bevindt zich op de Node in
  `exec.approvals.node.*`, niet in de koppelingsrecord.

</Warning>

## Afscherming van Node-opdrachten (2026.3.31+)

<Warning>
**Incompatibele wijziging:** vanaf `2026.3.31` zijn Node-opdrachten uitgeschakeld totdat de Node-koppeling is goedgekeurd. Alleen apparaatkoppeling volstaat niet meer om gedeclareerde Node-opdrachten beschikbaar te stellen.
</Warning>

Wanneer een Node voor het eerst verbinding maakt, wordt automatisch om koppeling verzocht.
Totdat dat verzoek is goedgekeurd, worden alle openstaande Node-opdrachten van die Node
gefilterd en niet uitgevoerd. Zodra de koppeling is goedgekeurd, worden de gedeclareerde
opdrachten van de Node beschikbaar, onderworpen aan het normale opdrachtbeleid.

Dit betekent:

- Nodes die eerder uitsluitend op apparaatkoppeling vertrouwden om opdrachten beschikbaar te stellen, moeten
  nu ook de Node-koppeling voltooien.
- Opdrachten die vóór goedkeuring van de koppeling in de wachtrij zijn geplaatst, worden verwijderd en niet uitgesteld.

## Vertrouwensgrenzen voor Node-gebeurtenissen (2026.3.31+)

<Warning>
**Incompatibele wijziging:** door Nodes geïnitieerde uitvoeringen blijven nu beperkt tot een kleiner vertrouwd oppervlak.
</Warning>

Door Nodes geïnitieerde samenvattingen en gerelateerde sessiegebeurtenissen zijn beperkt tot het
bedoelde vertrouwde oppervlak. Door meldingen aangestuurde of door Nodes geactiveerde flows die
eerder afhankelijk waren van bredere toegang tot host- of sessietools, moeten mogelijk worden aangepast.
Deze beveiliging voorkomt dat Node-gebeurtenissen escaleren tot toegang tot tools op hostniveau
buiten wat de vertrouwensgrens van de Node toestaat.

Duurzame aanwezigheidsupdates van Nodes volgen dezelfde identiteitsgrens: de gebeurtenis
`node.presence.alive` wordt alleen geaccepteerd van geauthenticeerde apparaatsessies van Nodes
en werkt koppelingsmetadata alleen bij wanneer de apparaat-/Node-identiteit
al is gekoppeld. Een zelf gedeclareerde waarde `client.id` volstaat niet om
de laatst-gezien-status te schrijven.

## Via SSH geverifieerde automatische apparaatgoedkeuring (standaard)

De eerste apparaatkoppeling via `role: node` vanaf een privé-/CGNAT-adres wordt
automatisch goedgekeurd wanneer de Gateway **eigendom van de machine via SSH kan bewijzen**: deze
maakt opnieuw verbinding met de koppelingshost (`BatchMode`, `StrictHostKeyChecking=yes`),
voert daar `openclaw node identity --json` uit en keurt alleen goed wanneer de externe
apparaat-id en openbare sleutel exact overeenkomen met het openstaande verzoek. De sleutelovereenkomst
maakt dit veilig: alleen bereikbaarheid leidt nooit tot goedkeuring, waardoor medehuurders achter dezelfde NAT,
andere gebruikers op een gedeelde host en LAN-spoofing allemaal terugvallen op de normale
prompt.

Standaard ingeschakeld. Vereisten om dit te activeren:

- De gebruiker van het Gateway-proces (of `sshVerify.user`) kan niet-interactief via SSH verbinding maken met de Node-host
  (sleutels/agent; Tailscale SSH werkt ook) en de hostsleutel is
  al vertrouwd.
- `openclaw` wordt op de externe `PATH` gevonden voor niet-interactieve `sh -lc`.
- Het IP-adres dat verbinding maakt is een direct (niet via een proxy, geen loopback) privé-, ULA-,
  link-local- of CGNAT-adres, of komt overeen met `sshVerify.cidrs` wanneer dit is ingesteld.
- Dezelfde minimale geschiktheid als voor goedkeuring via vertrouwde CIDR: alleen een nieuwe Node-koppeling
  zonder bereik; upgrades, browsers, Control UI en WebChat tonen altijd een prompt.

Terwijl een controle wordt uitgevoerd, krijgt de Node-client de opdracht om pogingen te blijven doen
(`wait_then_retry`) in plaats van te pauzeren voor handmatige goedkeuring; als de controle
mislukt, valt de volgende poging terug op de normale promptflow. Mislukte doelen
krijgen een korte afkoelperiode (5 minuten na een niet-overeenkomende sleutel).

Goedgekeurde apparaten registreren `approvedVia: "ssh-verified"` en hun eerste gedeclareerde
mogelijkhedenoppervlak wordt in dezelfde stap goedgekeurd — de sleutelovereenkomst bewijst al dat
de Node wordt uitgevoerd onder het account van de operator op een machine waarvan die eigenaar is, wat
dezelfde bewering is als bij een handmatige goedkeuring van mogelijkheden. Latere uitbreidingen van het oppervlak
tonen nog steeds een prompt.

Aanscherpen of uitschakelen:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        // Volledig uitschakelen:
        sshVerify: false,
        // ...of het bereik/de controle afstemmen:
        // sshVerify: { user: "me", identity: "~/.ssh/probe", timeoutMs: 7000, cidrs: ["10.0.0.0/8"] },
      },
    },
  },
}
```

## Automatische goedkeuring (macOS-app)

De macOS-app kan proberen Node-mogelijkheidsverzoeken **stilzwijgend goed te keuren**
wanneer:

- het verzoek is gemarkeerd als `silent` (de Gateway markeert het eerste mogelijkhedenoppervlak
  als stil wanneer de apparaatkoppeling niet-interactief is goedgekeurd), en
- de app een SSH-verbinding met de Gateway-host kan verifiëren met dezelfde
  gebruiker.

Als stilzwijgende goedkeuring mislukt, valt deze terug op de normale prompt Approve/Reject.

## Automatische apparaatgoedkeuring via vertrouwde CIDR

WS-apparaatkoppeling voor `role: node` blijft standaard handmatig. Voor privé-Node-
netwerken waar de Gateway het netwerkpad al vertrouwt, kunnen operators zich aanmelden
met expliciete CIDR's of exacte IP-adressen:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

Beveiligingsgrens:

- Uitgeschakeld wanneer `gateway.nodes.pairing.autoApproveCidrs` niet is ingesteld.
- Er bestaat geen algemene modus voor automatische goedkeuring van LAN's of privénetwerken; via SSH geverifieerde
  automatische goedkeuring (hierboven) vereist een cryptografische overeenkomst van de apparaatsleutel, nooit
  alleen netwerklokaliteit.
- Alleen een nieuw `role: node`-apparaatkoppelingsverzoek zonder aangevraagde bereiken komt
  in aanmerking.
- Operator-, browser-, Control UI- en WebChat-clients blijven handmatig.
- Upgrades van rollen, bereiken, metadata en openbare sleutels blijven handmatig.
- Headerpaden van vertrouwde proxy's via loopback op dezelfde host komen niet in aanmerking, omdat dat
  pad door lokale aanroepers kan worden vervalst.

## Opschoning bij vervanging van stille koppelingen

Niet-interactieve goedkeuringen registreren hun herkomst in de rij van het gekoppelde apparaat:
goedkeuringen via lokaal beleid op dezelfde host als `silent`, Node-goedkeuringen via vertrouwde CIDR als
`trusted-cidr`, via SSH geverifieerde Node-goedkeuringen als `ssh-verified`. Clients waarvan de statusmap tijdelijk is (tijdelijke homemappen,
containers, sandboxes per uitvoering) genereren per uitvoering een nieuw sleutelpaar voor het apparaat en elke
uitvoering koppelt zich stilzwijgend opnieuw als een volledig nieuw apparaat — zonder opschoning groeit de lijst met gekoppelde apparaten
met één verouderde rij per uitvoering.

Wanneer de Gateway een **lokale** apparaatkoppeling stilzwijgend goedkeurt, trekt deze
oudere via `silent` goedgekeurde records in die bij hetzelfde clientcluster horen
(overeenkomstige `clientId`, `clientMode` en weergavenaam) en momenteel niet
verbonden zijn. Lokale clients worden op de Gateway-host zelf uitgevoerd, zodat de clustersleutel
niet met een andere machine kan overeenkomen. Tokens van ingetrokken rijen worden onmiddellijk ongeldig;
elke overeenkomende verouderde Node-koppelingsvermelding wordt gewist en een verwijderingsgebeurtenis
`node.pair.resolved` wordt uitgezonden.

Grenzen:

- Alleen records waarvan de meest recente goedkeuring lokaal op dezelfde host (`silent`) plaatsvond, komen
  in aanmerking, zowel als trigger als als doel. Koppelingen die via een vertrouwd CIDR of SSH zijn geverifieerd,
  overschrijden hosts waarbij de weergavemetadata geen machine-identiteit vormt, en worden daarom
  nooit automatisch verwijderd — gebruik hiervoor de opschoonfunctie in de Control UI of
  `openclaw nodes remove`.
- Door de eigenaar goedgekeurde koppelingen en koppelingen via een QR-/installatiecode (bootstrap) worden nooit
  automatisch verwijderd. Records die zijn goedgekeurd voordat herkomstgegevens bestonden, blijven beschermd,
  zelfs na een latere stille hergoedkeuring van dezelfde apparaat-id.
- Momenteel verbonden apparaten worden overgeslagen, zodat gelijktijdige lokale sessies met
  afzonderlijke statusmappen hun tokens behouden zolang ze actief zijn. Records die
  in de afgelopen minuut zijn goedgekeurd, worden ook overgeslagen, zodat gelijktijdige koppelingshandshakes
  elkaar niet kunnen intrekken voordat hun verbindingen zijn geregistreerd.
- De betrokken clients zijn per definitie lokaal, dus worden ze bij
  hun volgende verbinding stil opnieuw gekoppeld.

## Automatische goedkeuring bij metadata-upgrades

Wanneer een al gekoppeld apparaat opnieuw verbinding maakt met alleen wijzigingen in niet-gevoelige metadata
(bijvoorbeeld de weergavenaam of aanwijzingen over het clientplatform), behandelt OpenClaw
dit als een `metadata-upgrade`. Stille automatische goedkeuring is strikt beperkt: deze geldt alleen
voor vertrouwde lokale herverbindingen buiten de browser die al hebben bewezen over
lokale of gedeelde inloggegevens te beschikken, waaronder herverbindingen van systeemeigen apps op dezelfde host na
wijzigingen in metadata over de versie van het besturingssysteem. Browser-/Control UI-clients en externe clients
gebruiken nog steeds de expliciete flow voor hergoedkeuring. Scope-upgrades (van lezen naar
schrijven/beheerder) en wijzigingen in de openbare sleutel komen **niet** in aanmerking voor
automatische goedkeuring bij metadata-upgrades; dit blijven expliciete verzoeken om hergoedkeuring.

## Hulpmiddelen voor QR-koppeling

`/pair qr` geeft de koppelingspayload weer als gestructureerde media, zodat mobiele clients en
browserclients deze rechtstreeks kunnen scannen.

Bij het verwijderen van een apparaat worden ook alle verouderde openstaande koppelingsverzoeken voor die
apparaat-id opgeruimd, zodat `nodes pending` na een intrekking geen verweesde rijen toont.

## Lokaliteit en doorgestuurde headers

Gateway-koppeling behandelt een verbinding alleen als loopback wanneer zowel de onbewerkte socket
als eventueel bewijs van een upstreamproxy daarmee overeenstemmen. Als een verzoek via loopback binnenkomt maar
bewijs bevat in de header `Forwarded`, een `X-Forwarded-*`-header of de header `X-Real-IP`, maakt dat
bewijs uit doorgestuurde headers de aanspraak op loopbacklokaliteit ongeldig en vereist
het koppelingspad expliciete goedkeuring, in plaats van het verzoek stil te behandelen als een
verbinding vanaf dezelfde host. Zie
[Authenticatie via een vertrouwde proxy](/nl/gateway/trusted-proxy-auth) voor de overeenkomstige regel voor
operatorauthenticatie.

## Opslag (lokaal, privé)

De koppelingsstatus bevindt zich in de records van gekoppelde apparaten in de gedeelde SQLite-statusdatabase
onder de statusmap van de Gateway (standaard `~/.openclaw`):

- `~/.openclaw/state/openclaw.sqlite` (gekoppelde apparaten met apparaatauthenticatie,
  goedgekeurde Node-oppervlakken, openstaande oppervlakteverzoeken, openstaande verzoeken om apparaten te koppelen
  en bootstraptokens)

Als je `OPENCLAW_STATE_DIR` overschrijft, verhuist de database mee. Gateways
die zijn geüpgraded vanuit releases met JSON-opslagplaatsen, importeren deze bij het opstarten en laten
de archieven `devices/*.json.migrated` en `nodes/*.json.migrated` achter.

Beveiligingsopmerkingen:

- Apparaattokens zijn geheimen; behandel de statusdatabase als gevoelig.
- Voor het roteren van een apparaattoken gebruik je `openclaw devices rotate` /
  `device.token.rotate`.

## Transportgedrag

- Het transport is **statusloos**; het slaat geen lidmaatschap op.
- Als de Gateway offline is of koppeling is uitgeschakeld, kunnen Nodes niet worden gekoppeld.
- In de externe modus vindt koppeling plaats met de opslagplaats van de externe Gateway.

## Gerelateerd

- [Kanaalkoppeling](/nl/channels/pairing)
- [CLI voor Nodes](/nl/cli/nodes)
- [CLI voor apparaten](/nl/cli/devices)
