---
read_when:
    - De iOS-node koppelen of opnieuw verbinden
    - De directe Apple Watch-Node inschakelen of problemen ermee oplossen
    - De iOS-app uitvoeren vanuit de broncode
    - Problemen met Gateway-detectie of canvasopdrachten oplossen
summary: 'iOS-Node-app: verbinding maken met de Gateway, koppelen, canvas en probleemoplossing'
title: iOS-app
x-i18n:
    generated_at: "2026-07-27T05:11:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2b01a63fa1e2c445f7fb35843536f7f5918e94bfe885dac19c852d7d52d86342
    source_path: platforms/ios.md
    workflow: 16
---

Beschikbaarheid: iPhone-appbuilds worden via Apple-kanalen gedistribueerd wanneer dit voor een release is ingeschakeld. Lokale ontwikkelbuilds kunnen ook vanuit de broncode worden uitgevoerd.

## Wat het doet

- Maakt via WebSocket verbinding met een Gateway (LAN of tailnet).
- Biedt Node-mogelijkheden: Canvas, schermopname, camera-opname, locatie, Talk-modus, spraakactivering en optionele gezondheidssamenvattingen.
- Ontvangt `node.invoke`-opdrachten en rapporteert statusgebeurtenissen van de Node.
- Bladert alleen-lezen door de workspace van de geselecteerde agent vanuit het Agents-oppervlak (Bestanden): navigatie door mappen, tekstvoorbeelden met syntaxisaccentuering, afbeeldingsvoorbeelden en export via het deelvenster. Geen schrijfbewerkingen; de Gateway begrenst de grootte van voorbeelden.
- Bewaart per gekoppelde Gateway een kleine, alleen-lezen offlinecache van recente chatsessies en transcripties: bij een koude start wordt de laatst bekende transcriptie onmiddellijk weergegeven en vernieuwd zodra de Gateway reageert, recente chats blijven tijdens een verbroken verbinding beschikbaar om door te bladeren en opnieuw instellen/vergeten wist de beveiligde lokale cache.
- Plaatst tekstberichten die tijdens een verbroken verbinding worden verzonden in een duurzame outbox per Gateway (maximaal 50): berichten in de wachtrij worden in de transcriptie weergegeven, bij opnieuw verbinden op volgorde verzonden met idempotente nieuwe pogingen, blijven bewaard totdat de canonieke geschiedenis de verzending bevestigt, worden vóór het tonen van een actie voor opnieuw proberen/verwijderen opnieuw geprobeerd met back-off en verlopen in plaats van te worden verzonden na 48 uur offline; opnieuw instellen/vergeten wist de wachtrij samen met de cache.
- Chat is het centrale oppervlak voor tekst en spraak. Via Chat-acties kan het volledige scherm Sessies worden geopend zonder Chat te verlaten en kan de redenering van de assistent en toolactiviteit worden getoond of verborgen. Tik op de microfoon om een concept te dicteren, open het menu om een spraakbericht op te nemen of gebruik het ingebouwde Talk-besturingselement voor realtime spraak; tijdens luisteren of spreken beweegt het Talk-besturingselement mee met het live microfoon- of afspeelniveau.
- **Settings -> OpenClaw** opent een speciale instellingenassistent voor de Gateway wanneer de operatorverbinding `operator.admin` heeft en de Gateway `openclaw.chat` ondersteunt. Het configuratiegesprek blijft gescheiden van gewone Chat, maskeert geheime antwoorden lokaal en gaat pas naar Chat nadat je op **Open Chat** tikt.
- Spreekt assistentberichten op verzoek uit: houd een bericht in Chat lang ingedrukt en kies **Listen**. De app speelt ondersteunde `tts.speak`-fragmenten van de Gateway af met de geconfigureerde TTS-provider en valt terug op spraak op het apparaat wanneer Gateway-audio niet beschikbaar of niet afspeelbaar is. Het afspelen stopt wanneer van sessie wordt gewisseld of de app naar de achtergrond gaat.

## Vereisten

- Gateway die op een ander apparaat draait (macOS, Linux of Windows via WSL2).
- Netwerkroute:
  - Hetzelfde LAN via Bonjour, **of**
  - Tailnet via unicast DNS-SD (voorbeelddomein: `openclaw.internal.`), **of**
  - Handmatig host/poort instellen (terugvaloptie).

## Snel aan de slag (koppelen en verbinden)

Bij de eerste start doorloopt de app een korte uitleg over koppelen en een
machtigingenpagina (meldingen, camera, microfoon, foto's, contacten,
agenda, herinneringen, locatie). Elke toestemming is optioneel en kan
later worden gewijzigd via **Settings** -> **Permissions** of in de iOS-app Instellingen.

1. Start een geauthenticeerde Gateway met een route die je telefoon kan bereiken. Tailscale
   Serve is de aanbevolen externe route:

```bash
openclaw gateway --port 18789 --tailscale serve
```

Gebruik voor een vertrouwde configuratie binnen hetzelfde LAN in plaats daarvan een geauthenticeerde `gateway.bind: "lan"`.
De standaardbinding aan loopback is niet bereikbaar vanaf een telefoon. Als de
Gateway nog niet is geconfigureerd, voer dan eerst `openclaw onboard` uit, zodat voor het aanmaken
van een configuratiecode een authenticatieroute met token of wachtwoord beschikbaar is.

2. Open de [Control UI](/nl/web/control-ui), selecteer **Nodes** en klik op
   **Pair mobile device** op de pagina **Devices**. Volledige toegang wordt aanbevolen
   en is standaard geselecteerd; kies alleen Limited access als je
   administratieve Gateway-besturingselementen wilt weglaten en klik vervolgens op **Create setup code**.

3. Open in de iOS-app **Settings** -> **Gateway**, scan de QR-code (of plak
   de configuratiecode) en maak verbinding.

   Als de configuratiecode zowel LAN- als Tailscale Serve-routes bevat, test de app
   deze op volgorde en slaat deze het eerste bereikbare eindpunt op.

   Gekoppelde Gateways blijven in de lijst **Gateways** staan. Het vinkje geeft
   de actieve Gateway aan; gebruik het bliksembesturingselement op een andere rij om
   tegelijkertijd de operatorsessie daarvan verbonden te houden. Als je de actieve Gateway wijzigt,
   worden andere ingeschakelde Gateways niet ontkoppeld. Alleen de actieve Gateway ontvangt
   de Node-sessie met de mogelijkheden van de iPhone, zodat opdrachten voor camera, scherm, locatie en
   andere apparaatfuncties altijd één ondubbelzinnige eigenaar hebben. iOS kan
   deze verbindingen op de voorgrond onderbreken nadat de app naar de achtergrond is gegaan.

4. De officiële app maakt automatisch verbinding. Als **Pending approval** een
   verzoek toont, controleer dan de rol en bereiken voordat je het goedkeurt.

   **Settings → Gateway** toont of de opgeslagen operatorverbinding
   **Full**- of **Limited**-toegang heeft. Configuratie met platte tekst via LAN `ws://` wordt voor de
   veiligheid van bearer-tokens automatisch beperkt. Als de toegang beperkt is, configureer dan `wss://` of
   Tailscale Serve, scan een nieuwe code voor volledige toegang vanuit de Control UI of `openclaw qr`
   en maak vervolgens opnieuw verbinding om instellingen en upgrades in te schakelen.

Voor de knop in de Control UI is een reeds gekoppelde sessie met `operator.admin` vereist.
Als terugvaloptie via de terminal selecteer je een ontdekte Gateway in de iOS-app (of schakel je
Manual Host in en voer je host/poort in), waarna je het verzoek op de Gateway-host goedkeurt:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Als de app het koppelen opnieuw probeert met gewijzigde authenticatiegegevens (rol/bereiken/openbare sleutel), wordt het vorige openstaande verzoek vervangen en wordt een nieuwe `requestId` aangemaakt. Voer `openclaw devices list` vóór goedkeuring opnieuw uit.

Optioneel: als de iOS-Node altijd verbinding maakt vanuit een strikt beheerd subnet, kun je automatische goedkeuring van een Node bij de eerste koppeling inschakelen met expliciete CIDR's of exacte IP-adressen:

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

Dit is standaard uitgeschakeld. Het geldt alleen voor een nieuwe `role: node`-koppeling zonder aangevraagde bereiken. Voor operator-/browserkoppeling en elke wijziging van rol, bereik, metadata of openbare sleutel blijft handmatige goedkeuring vereist.

5. Controleer de verbinding:

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

## Gezondheidssamenvattingen

De iOS-Node kan een optioneel, alleen-lezen HealthKit-aggregaat voor de huidige
kalenderdag retourneren. Toestemming op het iOS-apparaat en expliciete autorisatie van Gateway-opdrachten zijn
onafhankelijke vereisten. Zie [HealthKit-samenvattingen](/nl/platforms/ios-healthkit) voor
configuratie, aanroepen, payloadvelden, privacygedrag en probleemoplossing.

De bijbehorende Apple Watch blijft standaard het bestaande iPhone-relais gebruiken en
heeft geen afzonderlijke Gateway-koppeling nodig. Koppel de Watch in de Watch-app van
Apple aan de iPhone, installeer OpenClaw via **Watch app -> My Watch -> Available
Apps** en open OpenClaw vervolgens eenmaal op beide apparaten.

## Goedkeuringen van opdrachten beoordelen

Een operatorverbinding met `operator.admin`, of een gekoppelde
`operator.approvals`-verbinding waarop de Gateway expliciet is gericht, kan openstaande
uitvoeringsverzoeken op de iPhone beoordelen. De goedkeuringskaart toont het door de Gateway
opgeschoonde opdrachtvoorbeeld, de waarschuwing, hostcontext, vervaltijd en alleen de
beslissingen die bij dat verzoek worden aangeboden. De gekoppelde Apple Watch ontvangt dezelfde
voor de beoordelaar veilige prompt via het bestaande iPhone-relais en biedt de compacte
keuze om eenmalig toe te staan of te weigeren. In de directe Gateway-modus van de Watch worden geen
goedkeuringsprompts doorgegeven.

De goedkeuringsstatus wordt gedeeld met de Control UI en ondersteunde chatoppervlakken. Het
eerste vastgelegde antwoord is bepalend. iPhone en Watch halen de canonieke
terminalregistratie van de Gateway op nadat een ander oppervlak het verzoek heeft afgehandeld, na een externe
melding dat het verzoek is afgehandeld en telkens wanneer een bevestiging van de afhandeling verloren kan zijn
gegaan. Acties blijven niet beschikbaar totdat deze teruglezing bevestigt of het
verzoek nog openstaat.

Het eigendom van goedkeuringen is gekoppeld aan de geselecteerde Gateway. Door van Gateway te wisselen kan
een oude prompt niet op de vervangende verbinding worden toegepast. Gateways van vóór de
uniforme goedkeuringsmethoden vallen terug op de meegeleverde uitvoeringsspecifieke methoden;
voor bewaarde terminalstatus en uitgebreidere resultaten tussen oppervlakken is een bijgewerkte
Gateway vereist.

## Vragen van agents beantwoorden

Chat toont openstaande Gateway-vragen als systeemeigen kaarten voor operatorverbindingen
met `operator.questions` (of `operator.admin`). Kaarten ondersteunen opties met één of
meerdere selecties, optiebeschrijvingen, vrije-tekst-antwoorden via **Other** en een
aftelling tot de vervaltijd. Na opnieuw verbinden worden openstaande vragen opnieuw geladen vanuit de Gateway. Een kaart
wordt vergrendeld wanneer dit apparaat de vraag beantwoordt, een ander oppervlak deze als eerste beantwoordt of de
vraag verloopt of wordt geannuleerd.

## Optionele directe Apple Watch-Node

In de directe modus krijgt de Watch een eigen ondertekende Node-identiteit en Gateway-verbinding.
Ondersteunde Node-opdrachten blijven werken via wifi of mobiel netwerk van de Watch zolang
OpenClaw actief is, zelfs wanneer de gekoppelde iPhone niet beschikbaar is.

Vereisten:

- De iPhone is verbonden met de Gateway met het bereik `operator.admin`.
- De configuratiecode vermeldt een `wss://`-Gateway-eindpunt met een certificaat dat door
  watchOS wordt vertrouwd; de Watch pollt de bijbehorende `https://`-origin. Platte HTTP en
  zelfondertekend of alleen op vingerafdruk gebaseerd vertrouwen worden niet ondersteund. Zie [Door de Gateway beheerde
  koppeling](/nl/gateway/pairing) voor de configuratie van eindpunten. Loopback-, alleen-iPhone-
  en alleen-tailnetroutes zijn niet zelfstandig bereikbaar door de Watch.
- Voor mobiel gebruik is een Apple Watch met mobiele netwerkondersteuning en een actief abonnement vereist.
- OpenClaw is actief op de Watch. Apple staat niet toe dat gewone watchOS-apps
  algemene WebSocket-/TCP-verbindingen actief houden, daarom gebruikt de directe Node korte HTTPS-
  polls en maakt deze opnieuw verbinding wanneer de app terugkeert naar de voorgrond. Zie Apples
  [richtlijnen voor low-level netwerken in watchOS](https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchOS).

Configuratie:

1. Open op de iPhone **Settings -> Apple Watch**.
2. Tik op **Enable Direct Gateway Connection**.
3. Open OpenClaw op de Watch voordat de kortlevende configuratiecode verloopt.
4. Controleer de afzonderlijke Apple Watch-rij met `openclaw nodes status`.

De configuratiecode bevat een kortlevende bootstrapreferentie die alleen voor de Node geldt; behandel deze
als een wachtwoord totdat deze verloopt. De code bevat nooit het opgeslagen Gateway-
wachtwoord of token van de iPhone. Na het koppelen bewaart de Watch een eigen apparaattoken en
verwijdert deze de bootstrapreferentie. De directe modus omvat alleen de onderstaande opdrachten.
Chat, Talk, goedkeuringen en de bestaande `watch.*`-meldingsstroom blijven
functies van het iPhone-relais en vereisen nog steeds de gekoppelde iPhone.

Directe watchOS-Node-opdrachten:

| Oppervlak     | Opdrachten                     | Opmerkingen                                             |
| ------------- | ------------------------------ | ------------------------------------------------------- |
| Apparaat      | `device.info`, `device.status` | Watch-identiteit, batterij, temperatuur, opslag en netwerk. |
| Meldingen     | `system.notify`                | Terwijl de app actief is; vereist toestemming op de Watch. |

watchOS stelt WebKit niet beschikbaar aan apps van derden, daarom
adverteert de directe Watch-Node geen Canvas-opdrachten.

## Push via relais voor officiële builds

Officieel gedistribueerde iOS-builds gebruiken een extern pushrelais in plaats van het onbewerkte APNs-token naar de Gateway te publiceren. Officiële App Store-builds uit het openbare releasekanaal gebruiken het gehoste relais op `https://ios-push-relay.openclaw.ai`; deze basis-URL is hardgecodeerd voor App Store-distributie en leest geen enkele overschrijving.

Aangepaste relaisimplementaties vereisen een bewust afzonderlijk iOS-build-/implementatietraject waarvan de relais-URL overeenkomt met de Gateway-relais-URL. Het App Store-releasekanaal accepteert nooit een aangepaste relais-URL. Als je een aangepaste relaisbuild gebruikt, stel dan de overeenkomende Gateway-relais-URL in:

```json5
{
  gateway: {
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
        },
      },
    },
  },
}
```

Zo werkt de stroom:

- De iOS-app registreert zich bij de relay met App Attest en een StoreKit-apptransactie-JWS.
- De relay retourneert een ondoorzichtige relay-handle plus een verzendmachtiging die aan de registratie is gebonden.
- De iOS-app haalt de identiteit van de gekoppelde Gateway op (`gateway.identity.get`) en neemt deze op in de relayregistratie, zodat de door de relay ondersteunde registratie aan die specifieke Gateway wordt gedelegeerd.
- De app stuurt die door de relay ondersteunde registratie door naar de gekoppelde Gateway met `push.apns.register`.
- De Gateway gebruikt die opgeslagen relay-handle voor `push.test`, activeringen op de achtergrond en activeringssignalen.
- Als de app later verbinding maakt met een andere Gateway of met een build met een andere basis-URL voor de relay, vernieuwt deze de relayregistratie in plaats van de oude koppeling opnieuw te gebruiken.

Wat de Gateway voor dit pad **niet** nodig heeft: geen relaytoken voor de gehele implementatie en geen rechtstreekse APNs-sleutel voor officiële, door de relay ondersteunde verzendingen vanuit de App Store.

Verwachte beheerdersstroom:

1. Installeer de officiële iOS-app.
2. Optioneel: stel `gateway.push.apns.relay.baseUrl` alleen op de Gateway in wanneer je bewust een afzonderlijke aangepaste relaybuild gebruikt.
3. Koppel de app aan de Gateway en laat deze de verbinding voltooien.
4. De app publiceert `push.apns.register` zodra deze een APNs-token heeft, de beheerderssessie verbonden is en de relayregistratie slaagt.
5. Daarna kunnen `push.test`, activeringen voor opnieuw verbinden en activeringssignalen de opgeslagen, door de relay ondersteunde registratie gebruiken.

## Achtergrondbakens voor actieve status

Wanneer iOS de app activeert voor een stille push, een achtergrondvernieuwing of een gebeurtenis voor een aanzienlijke locatiewijziging, probeert de app kort opnieuw verbinding te maken met de Node en roept deze vervolgens `node.event` aan met `event: "node.presence.alive"`. De Gateway registreert dit als `lastSeenAtMs`/`lastSeenReason` in de metadata van de gekoppelde Node/het gekoppelde apparaat, maar alleen nadat de identiteit van het geauthenticeerde Node-apparaat bekend is.

De app beschouwt een activering op de achtergrond alleen als succesvol geregistreerd wanneer het antwoord van de Gateway `handled: true` bevat. Oudere Gateways kunnen `node.event` bevestigen met `{ "ok": true }`; dat antwoord is compatibel, maar geldt niet als een duurzame update van het laatst geziene tijdstip.

Opmerking over compatibiliteit:

- `OPENCLAW_APNS_RELAY_BASE_URL` werkt nog steeds als tijdelijke omgevingsvariabele-overschrijving voor de Gateway (`gateway.push.apns.relay.baseUrl` is het configuratie-eerst-pad).
- De pushmodus van de App Store-releasebuild bevat de host van de gehoste relay als vaste waarde en leest nooit een overschrijving van de relay-URL — de omgevingsvariabele `OPENCLAW_PUSH_RELAY_BASE_URL` tijdens het bouwen is alleen van invloed op lokale iOS-buildmodi en sandboxbuildmodi.

## Authenticatie- en vertrouwensstroom

De relay bestaat om twee beperkingen af te dwingen die rechtstreekse APNs op de Gateway niet kan bieden voor officiële iOS-builds:

- Alleen authentieke OpenClaw-iOS-builds die via Apple worden gedistribueerd, kunnen de gehoste relay gebruiken.
- Een Gateway kan alleen door de relay ondersteunde pushes verzenden voor iOS-apparaten die aan die specifieke Gateway zijn gekoppeld.

Stap voor stap:

1. `iOS app -> gateway`: de app wordt via de normale authenticatiestroom van de Gateway aan de Gateway gekoppeld, waardoor deze zowel een geauthenticeerde Node-sessie als een geauthenticeerde beheerderssessie krijgt. De beheerderssessie roept `gateway.identity.get` aan.
2. `iOS app -> relay`: de app roept de registratie-eindpunten van de relay via HTTPS aan met een App Attest-bewijs plus een StoreKit-apptransactie-JWS. De relay valideert de bundel-ID, het App Attest-bewijs en het Apple-distributiebewijs en vereist het officiële productie-distributiepad — hierdoor kunnen lokale Xcode-/ontwikkelbuilds de gehoste relay niet gebruiken, omdat een lokale build niet aan het officiële Apple-distributiebewijs kan voldoen.
3. `gateway identity delegation`: vóór de relayregistratie haalt de app de identiteit van de gekoppelde Gateway op uit `gateway.identity.get` en neemt deze op in de registratiepayload voor de relay. De relay retourneert een relay-handle en een aan de registratie gebonden verzendmachtiging die aan die Gateway-identiteit is gedelegeerd.
4. `gateway -> relay`: de Gateway slaat de relay-handle en verzendmachtiging uit `push.apns.register` op. Bij `push.test`, activeringen voor opnieuw verbinden en activeringssignalen ondertekent de Gateway het verzendverzoek met zijn eigen apparaatidentiteit; de relay verifieert zowel de opgeslagen verzendmachtiging als de handtekening van de Gateway aan de hand van de gedelegeerde Gateway-identiteit uit de registratie. Een andere Gateway kan die opgeslagen registratie niet opnieuw gebruiken, zelfs niet als deze op de een of andere manier de handle bemachtigt.
5. `relay -> APNs`: de relay beheert de APNs-productiereferenties en het onbewerkte APNs-token voor de officiële build. De Gateway slaat het onbewerkte APNs-token nooit op voor door de relay ondersteunde officiële builds; de relay verzendt namens de gekoppelde Gateway de uiteindelijke push naar APNs.

Waarom dit ontwerp is gemaakt: om APNs-productiereferenties buiten Gateways van gebruikers te houden, te voorkomen dat onbewerkte APNs-tokens van officiële builds op de Gateway worden opgeslagen, het gebruik van de gehoste relay alleen toe te staan voor officiële OpenClaw-iOS-builds en te voorkomen dat de ene Gateway activeringspushes verzendt naar iOS-apparaten die bij een andere Gateway horen.

Lokale/handmatige builds blijven rechtstreekse APNs gebruiken. Als je deze builds zonder de relay test, heeft de Gateway nog steeds rechtstreekse APNs-referenties nodig:

```bash
export OPENCLAW_APNS_TEAM_ID="TEAMID"
export OPENCLAW_APNS_KEY_ID="KEYID"
export OPENCLAW_APNS_PRIVATE_KEY_P8="$(cat /path/to/AuthKey_KEYID.p8)"
```

Dit zijn runtime-omgevingsvariabelen voor de Gateway-host, geen Fastlane-instellingen. `apps/ios/fastlane/.env` slaat alleen App Store Connect-authenticatie op, zoals `APP_STORE_CONNECT_KEY_ID` en `APP_STORE_CONNECT_ISSUER_ID`; hiermee wordt de rechtstreekse APNs-aflevering voor lokale iOS-builds niet geconfigureerd.

Aanbevolen opslag op de Gateway-host, in overeenstemming met andere providerreferenties onder `~/.openclaw/credentials/`:

```bash
mkdir -p ~/.openclaw/credentials/apns
chmod 700 ~/.openclaw/credentials/apns
mv /path/to/AuthKey_KEYID.p8 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
chmod 600 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
export OPENCLAW_APNS_PRIVATE_KEY_PATH="$HOME/.openclaw/credentials/apns/AuthKey_KEYID.p8"
```

Commit het bestand `.p8` niet en plaats het niet in de uitgecheckte repo.

## Detectiepaden

### Bonjour (LAN)

De iOS-app zoekt naar `_openclaw-gw._tcp` op `local.` en, indien geconfigureerd, hetzelfde detectiedomein voor wide-area DNS-SD. Gateways op hetzelfde LAN verschijnen automatisch via `local.`; voor detectie tussen netwerken kan het geconfigureerde wide-area-domein worden gebruikt zonder het bakentype te wijzigen.

### Tailnet (tussen netwerken)

Als mDNS wordt geblokkeerd, gebruik je een unicast DNS-SD-zone (kies een domein; voorbeeld: `openclaw.internal.`) en gesplitste DNS van Tailscale. Zie [Bonjour](/nl/gateway/bonjour) voor het CoreDNS-voorbeeld.

### Handmatige host/poort

Schakel in Instellingen **Handmatige host** in en voer de host en poort van de Gateway in (standaard `18789`).

## Meerdere Gateways

De app houdt een register bij van elke Gateway waaraan deze is gekoppeld, zodat je ertussen kunt wisselen zonder opnieuw te koppelen:

- **Instellingen -> Gateway** toont een lijst **Gekoppelde Gateways** waarin de actieve Gateway is gemarkeerd. Tik op een vermelding om te wisselen; de app beëindigt de huidige sessies en maakt opnieuw verbinding met de geselecteerde Gateway. Naast de verbindingsrij verschijnt een snelwisselmenu wanneer meer dan één Gateway is gekoppeld.
- Referenties, beslissingen over TLS-vertrouwen, voorkeuren per Gateway en de gecachte chatgeschiedenis worden per Gateway opgeslagen. Bij het wisselen wordt de status van Gateways nooit vermengd en volgt de pushregistratie de actieve Gateway.
- Veeg over een gekoppelde Gateway (of gebruik het contextmenu) om deze te **Vergeten**. Hiermee worden de referenties, apparaattokens, TLS-pin en gecachte chats verwijderd.
- Gedetecteerde Gateways moeten zichtbaar zijn op het netwerk om ernaar te kunnen wisselen; handmatige Gateways maken opnieuw verbinding via de opgeslagen host en poort.

## Canvas + A2UI

De iOS-Node rendert een WKWebView-canvas. Gebruik `node.invoke` om deze aan te sturen:

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.navigate --params '{"url":"http://<gateway-host>:18789/__openclaw__/canvas/"}'
```

Opmerkingen:

- De canvashost van de Gateway biedt `/__openclaw__/canvas/` en `/__openclaw__/a2ui/` aan via de HTTP-server van de Gateway (dezelfde poort als `gateway.port`, standaard `18789`).
- De iOS-Node behoudt het ingebouwde raamwerk als standaardweergave wanneer deze verbonden is. `canvas.a2ui.push` en `canvas.a2ui.reset` gebruiken de gebundelde A2UI-pagina die eigendom is van de app.
- Externe A2UI-pagina's van de Gateway kunnen op iOS alleen worden gerenderd; systeemeigen A2UI-knopacties worden alleen geaccepteerd vanaf gebundelde pagina's die eigendom zijn van de app.
- Keer terug naar het ingebouwde raamwerk met `canvas.navigate` en `{"url":""}`.

## Relatie tot Computer Use

De iOS-app is een mobiel Node-oppervlak, geen backend voor Codex Computer Use. Codex Computer Use en `cua-driver mcp` besturen een lokale macOS-desktop via MCP-tools; de iOS-app stelt iPhone-mogelijkheden beschikbaar via OpenClaw-Node-opdrachten zoals `canvas.*`, `camera.*`, `screen.*`, `location.*` en `talk.*`.

Agents kunnen de iOS-app nog steeds via OpenClaw bedienen door Node-opdrachten aan te roepen, maar deze aanroepen verlopen via het Node-protocol van de Gateway en zijn onderworpen aan de iOS-beperkingen voor de voor- en achtergrond. Gebruik [Codex Computer Use](/nl/plugins/codex-computer-use) voor lokale desktopbesturing en deze pagina voor de mogelijkheden van de iOS-Node.

### Canvas-evaluatie/-momentopname

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.eval --params '{"javaScript":"(() => { const {ctx} = window.__openclaw; ctx.clearRect(0,0,innerWidth,innerHeight); ctx.lineWidth=6; ctx.strokeStyle=\"#ff2d55\"; ctx.beginPath(); ctx.moveTo(40,40); ctx.lineTo(innerWidth-40, innerHeight-40); ctx.stroke(); return \"ok\"; })()"}'
```

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.snapshot --params '{"maxWidth":900,"format":"jpeg"}'
```

## Stemactivering + gespreksmodus

- Stemactivering en de gespreksmodus zijn beschikbaar in Instellingen.
- Realtime Talk van OpenAI gebruikt WebRTC dat door de client wordt beheerd wanneer `talk.realtime.transport` gelijk is aan `webrtc`; een expliciete configuratie van `gateway-relay` blijft door de Gateway beheerd. Zie [Gespreksmodus](/nl/nodes/talk).
- iOS-Nodes die Talk ondersteunen, adverteren de mogelijkheid `talk` en kunnen `talk.ptt.start`, `talk.ptt.stop`, `talk.ptt.cancel` en `talk.ptt.once` declareren; de Gateway staat deze push-to-talk-opdrachten standaard toe voor vertrouwde Nodes die Talk ondersteunen.
- iOS kan achtergrondaudio onderbreken; beschouw spraakfuncties als best-effort wanneer de app niet actief is.

## Veelvoorkomende fouten

- `NODE_BACKGROUND_UNAVAILABLE`: breng de iOS-app naar de voorgrond (opdrachten voor canvas, camera en scherm vereisen dit).
- `A2UI_HOST_UNAVAILABLE`: de gebundelde A2UI-pagina was niet bereikbaar in de WebView van de app; houd de app op de voorgrond op het tabblad Scherm en probeer het opnieuw.
- De koppelingsprompt verschijnt nooit: voer `openclaw devices list` uit en keur de koppeling handmatig goed.
- De Watch toont geen iPhone-status: controleer of de iPhone `watchPaired: true`
  en `watchAppInstalled: true` rapporteert in `watch.status`. Als koppeling onwaar is, koppel je de
  Watch in de Watch-app van Apple. Als installatie onwaar is, installeer je de bijbehorende app
  via **My Watch -> Available Apps**. Open na een van beide wijzigingen OpenClaw eenmaal op de
  Watch; voor onmiddellijke bereikbaarheid moeten beide apps nog steeds actief zijn,
  terwijl updates in de wachtrij later op de achtergrond kunnen aankomen.
- Opnieuw verbinden mislukt na herinstallatie: het koppelingstoken in de Sleutelhanger is gewist; koppel de Node opnieuw.

## Gerelateerde documentatie

- [Koppelen](/nl/channels/pairing)
- [Detectie](/nl/gateway/discovery)
- [Bonjour](/nl/gateway/bonjour)
